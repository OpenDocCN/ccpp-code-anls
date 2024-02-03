# `chatglm.cpp\chatglm_cpp\convert.py`

```
"""
Convert Hugging Face ChatGLM/ChatGLM2 models to GGML format
"""
# 导入必要的库
import argparse
import platform
import struct
import sys
from enum import Enum
from pathlib import Path
from typing import BinaryIO, Optional

import torch
import torch.nn.functional as F
from tabulate import tabulate
from tqdm import tqdm
from transformers import AutoConfig, AutoModel, AutoModelForCausalLM, AutoTokenizer

# 定义一些常量
GGML_QK8_0 = 32
GGML_QK4_0 = 32
GGML_QK4_1 = 32
GGML_QK5_0 = 32
GGML_QK5_1 = 32

GGML_MEM_ALIGN = 16

# 如果操作系统是 macOS，则模拟 cpm_kernels 模块
if platform.system() == "Darwin":
    sys.modules["cpm_kernels"] = object()  # type: ignore

# 定义 GGMLType 枚举类
class GGMLType(Enum):
    F32 = 0
    F16 = 1
    Q4_0 = 2
    Q4_1 = 3
    Q5_0 = 6
    Q5_1 = 7
    Q8_0 = 8

# 定义 ModelType 枚举类
class ModelType(Enum):
    CHATGLM = 1
    CHATGLM2 = 2
    CHATGLM3 = 3
    BAICHUAN7B = 1024
    BAICHUAN13B = 1025
    INTERNLM = 1280

# 定义量化函数 quantize_q8_0
def quantize_q8_0(tensor: torch.Tensor) -> torch.Tensor:
    # 确保张量的形状可以被 GGML_QK8_0 整除
    assert tensor.shape[1] % GGML_QK8_0 == 0
    tensor = tensor.view(-1, GGML_QK8_0)
    # 计算缩放因子
    scale = tensor.abs().max(dim=-1, keepdim=True).values / ((1 << 7) - 1)
    # 对张量进行量化
    tensor = (tensor / scale).round().clamp(min=-128, max=127).char()
    # 将缩放因子添加到每个块中
    tensor = torch.cat((scale.half().view(torch.int8), tensor), dim=-1)
    return tensor

# 定义量化函数 quantize_q4_0
def quantize_q4_0(tensor: torch.Tensor) -> torch.Tensor:
    # 确保张量的形状可以被 GGML_QK4_0 整除
    assert tensor.shape[1] % GGML_QK4_0 == 0
    tensor = tensor.view(-1, GGML_QK4_0)
    # 计算最大值的索引和值
    abs_max_indices = tensor.abs().max(dim=-1, keepdim=True).indices
    max_values = torch.take_along_dim(tensor, abs_max_indices, dim=-1)
    scale = max_values / -8
    # 对张量进行量化
    tensor = (tensor / scale + 8).round().clamp(min=0, max=15).char()
    # 将两个 int4 权重压缩成一个 int8
    tensor = tensor[:, :16] | (tensor[:, 16:] << 4)
    # 将缩放因子添加到每个块中
    # 将scale张量转换为半精度浮点数，然后转换为8位整数，并与tensor张量在最后一个维度上拼接
    tensor = torch.cat((scale.half().view(torch.int8), tensor), dim=-1)
    # 返回拼接后的张量
    return tensor
def quantize_q4_1(tensor: torch.Tensor) -> torch.Tensor:
    # 将输入张量量化为 Q4.1 格式，对应 ggml.c 中的 ggml_quantize_q4_1 函数
    assert tensor.shape[1] % GGML_QK4_1 == 0
    # 将张量形状重新调整为 (-1, GGML_QK4_1)
    tensor = tensor.view(-1, GGML_QK4_1)
    # 计算每行的最小值并保持维度，得到最小值张量
    min_vals = tensor.min(dim=-1, keepdim=True).values
    # 计算每行的最大值并保持维度，得到最大值张量
    max_vals = tensor.max(dim=-1, keepdim=True).values
    # 计算缩放比例
    scale = (max_vals - min_vals) / ((1 << 4) - 1)
    # 对张量进行量化，取整、截断到 [0, 15] 范围，并转换为 char 类型
    tensor = ((tensor - min_vals) / scale).round().clamp(min=0, max=15).char()
    # 将两个 int4 权重压缩为一个 int8
    tensor = tensor[:, :16] | (tensor[:, 16:] << 4)
    # 在每个块中添加缩放和最小值
    tensor = torch.cat((scale.half().view(torch.int8), min_vals.half().view(torch.int8), tensor), dim=-1)
    return tensor


def quantize_q5_0(tensor: torch.Tensor) -> torch.Tensor:
    # 将输入张量量化为 Q5.0 格式，对应 ggml.c 中的 ggml_quantize_q5_0 函数
    assert tensor.shape[1] % GGML_QK5_0 == 0
    # 将张量形状重新调整为 (-1, GGML_QK5_0)
    tensor = tensor.view(-1, GGML_QK5_0)
    # 计算每行绝对值最大值的索引
    abs_max_indices = tensor.abs().max(dim=-1, keepdim=True).indices
    # 根据索引获取最大值
    max_values = torch.take_along_dim(tensor, abs_max_indices, dim=-1)
    # 计算缩放比例
    scale = max_values / -16
    # 对张量进行量化，取整、截断到 [0, 31] 范围，并转换为 char 类型
    tensor = (tensor / scale + 16).round().clamp(min=0, max=31).char()
    # 将两个 int5 权重压缩为一个 int8
    qs = (tensor[:, :16] & 0x0F) | (tensor[:, 16:] << 4)
    qh = torch.zeros(tensor.shape[:-1], dtype=torch.int32)
    for i in range(32):
        qh |= ((tensor[:, i] & 0x10) >> 4).int() << i
    # 在每个块中添加缩放
    tensor = torch.cat((scale.half().view(torch.int8), qh[..., None].view(torch.int8), qs), dim=-1)
    return tensor


def quantize_q5_1(tensor: torch.Tensor) -> torch.Tensor:
    # 将输入张量量化为 Q5.1 格式，对应 ggml.c 中的 ggml_quantize_q5_1 函数
    assert tensor.shape[1] % GGML_QK5_1 == 0
    # 将张量形状重新调整为 (-1, GGML_QK5_1)
    tensor = tensor.view(-1, GGML_QK5_1)
    # 计算每行的最小值并保持维度，得到最小值张量
    min_vals = tensor.min(dim=-1, keepdim=True).values
    # 计算每行的最大值并保持维度，得到最大值张量
    max_vals = tensor.max(dim=-1, keepdim=True).values
    # 计算缩放比例
    scale = (max_vals - min_vals) / ((1 << 5) - 1)
    # 对张量进行量化，取整、截断到 [0, 31] 范围，并转换为 char 类型
    tensor = ((tensor - min_vals) / scale).round().clamp(min=0, max=31).char()
    # 将两个 int5 权重压缩为一个 int8
    qs = (tensor[:, :16] & 0x0F) | (tensor[:, 16:] << 4)
    # 创建一个全零张量，形状为去掉最后一个维度的 tensor 的形状，数据类型为 int32
    qh = torch.zeros(tensor.shape[:-1], dtype=torch.int32)
    # 遍历 32 次
    for i in range(32):
        # 对每一列的数据进行位运算，将结果按位或到 qh 中
        qh |= ((tensor[:, i] & 0x10) >> 4).int() << i

    # 在每个块中添加 scale 和 min 值
    # 将 scale 和 min_vals 转换为 int8 类型后，与 qh 和 qs 拼接在一起，沿着最后一个维度拼接
    tensor = torch.cat(
        (scale.half().view(torch.int8), min_vals.half().view(torch.int8), qh[..., None].view(torch.int8), qs), dim=-1
    )
    # 返回拼接后的张量
    return tensor
# 将张量数据写入文件
def dump_tensor(f, name: str, tensor: torch.Tensor, ggml_type: GGMLType):
    # 断言张量数据类型为 torch.float32
    assert tensor.dtype == torch.float32

    # 写入张量名称长度和名称
    f.write(struct.pack("i", len(name.encode())))
    f.write(name.encode())

    # 写入张量形状、数据类型和 GGML 类型
    f.write(struct.pack("i" * (2 + tensor.ndim), tensor.ndim, *tensor.shape, ggml_type.value)

    # 根据 GGML 类型对张量数据进行处理
    if ggml_type == GGMLType.F32:
        tensor = tensor.float()
    elif ggml_type == GGMLType.F16:
        tensor = tensor.half()
    elif ggml_type == GGMLType.Q8_0:
        tensor = quantize_q8_0(tensor)
    elif ggml_type == GGMLType.Q4_0:
        tensor = quantize_q4_0(tensor)
    elif ggml_type == GGMLType.Q4_1:
        tensor = quantize_q4_1(tensor)
    elif ggml_type == GGMLType.Q5_0:
        tensor = quantize_q5_0(tensor)
    elif ggml_type == GGMLType.Q5_1:
        tensor = quantize_q5_1(tensor)
    else:
        raise NotImplementedError(f"Cannot dump tensor of dtype {tensor.dtype}")

    # 对齐地址
    aligned_pos = (f.tell() + (GGML_MEM_ALIGN - 1)) // GGML_MEM_ALIGN * GGML_MEM_ALIGN
    f.seek(aligned_pos)
    # 将张量数据转换为 numpy 数组，并写入文件

# 准备存储状态字典的张量信息
def dump_state_dict(f, weight_names, state_dict, quantization_bit, ggml_type):
    tensor_info = []
    # 遍历权重名称列表，显示处理模型状态的进度条
    for name in tqdm(weight_names, desc="Processing model states"):
        # 获取当前权重张量
        tensor = state_dict[name]
        # 如果张量维度为2
        if tensor.ndim == 2:
            # 2维权重：如果需要，应该对其进行量化

            # 步骤1：将其反量化为float32
            if tensor.dtype == torch.int8:
                # 确保量化位数为4或8
                assert quantization_bit in [4, 8]
                # 获取通道-wise的缩放因子
                scale = state_dict[f"{name}_scale"].float()

                if quantization_bit == 4:
                    # 将int4权重转换为int8
                    low_bits = ((tensor << 4) & 0xF0) >> 4
                    high_bits = (tensor & 0xF0) >> 4
                    tensor = torch.stack((high_bits, low_bits), dim=-1).view(tensor.shape[0], -1)
                tensor = tensor * scale[:, None]
            else:
                tensor = tensor.float()

            # 步骤2：将其量化为ggml格式
            tensor_ggml_type = ggml_type
        else:
            # 1维权重：将其转换为float32
            assert tensor.ndim == 1
            tensor = tensor.float()
            tensor_ggml_type = GGMLType.F32

        # 将张量和其ggml类型转储到文件中
        dump_tensor(f, name, tensor, tensor_ggml_type)
        # 记录张量信息（名称、形状、数据类型）
        tensor_info.append((name, tensor.shape, tensor_ggml_type.name))

    # 打印张量信息表格
    print(tabulate(tensor_info, headers=["name", "shape", "dtype"], tablefmt="psql"))
# 定义一个基础转换器类
class BaseConverter:
    # 定义一个类方法，用于转换模型数据
    @classmethod
    def convert(cls, f, model, tokenizer, ggml_type):
        # 写入"ggml"作为文件头标识
        f.write(b"ggml")  # magic
        # 将模型类型和版本号打包写入文件
        f.write(struct.pack("ii", cls.MODEL_TYPE.value, 1))  # model type & version
        # 调用子类的方法，将模型配置信息写入文件
        cls.dump_config(f, model.config, ggml_type)
        # 调用子类的方法，将分词器信息写入文件
        cls.dump_tokenizer(f, tokenizer)
        # 调用子类的方法，将模型数据写入文件
        cls.dump_model(f, model, ggml_type)


# 定义一个特定类型的转换器类，继承自基础转换器类
class ChatGLMConverter(BaseConverter):
    # 定义模型类型为CHATGLM
    MODEL_TYPE = ModelType.CHATGLM

    # 定义一个静态方法，用于将模型配置信息写入文件
    @staticmethod
    def dump_config(f, config, ggml_type):
        # 断言配置中的二维位置编码应为True
        assert config.position_encoding_2d, "unimplemented: position_encoding_2d should be True"
        # 断言内部隐藏层大小应为隐藏层大小的4倍
        assert (
            config.inner_hidden_size == 4 * config.hidden_size
        ), "unimplemented: inner_hidden_size should be 4 times hidden_size"
        # 将配置信息打包写入文件
        config_values = [
            ggml_type.value,
            config.vocab_size,
            config.hidden_size,
            config.num_attention_heads,
            config.num_layers,
            config.inner_hidden_size,
            config.max_sequence_length,
            config.bos_token_id if config.bos_token_id is not None else -1,
            config.eos_token_id if config.eos_token_id is not None else -1,
            config.pad_token_id if config.pad_token_id is not None else -1,
            config.sep_token_id if config.sep_token_id is not None else -1,
        ]
        f.write(struct.pack("i" * len(config_values), *config_values))

    # 定义一个静态方法，用于将分词器信息写入文件
    @staticmethod
    def dump_tokenizer(f, tokenizer):
        # 获取序列化的模型协议
        serialized_model_proto = tokenizer.sp_tokenizer.text_tokenizer.sp.serialized_model_proto()
        # 将序列化的模型协议长度写入文件
        f.write(struct.pack("i", len(serialized_model_proto)))
        # 将序列化的模型协议写入文件
        f.write(serialized_model_proto)
    # 将模型的权重信息保存到文件中
    def dump_model(f, model, ggml_type):
        # 断言 lm_head 权重必须与输入嵌入层权重相同，否则抛出异常
        assert torch.allclose(
            model.state_dict()["transformer.word_embeddings.weight"], model.state_dict()["lm_head.weight"]
        ), "unimplemented: lm_head weight must be tied to input embedding"

        # 初始化权重名称列表，包括输入嵌入层权重和各个层的权重
        weight_names = ["transformer.word_embeddings.weight"]
        for i in range(model.config.num_layers):
            weight_names += [
                f"transformer.layers.{i}.input_layernorm.weight",
                f"transformer.layers.{i}.input_layernorm.bias",
                f"transformer.layers.{i}.attention.query_key_value.weight",
                f"transformer.layers.{i}.attention.query_key_value.bias",
                f"transformer.layers.{i}.attention.dense.weight",
                f"transformer.layers.{i}.attention.dense.bias",
                f"transformer.layers.{i}.post_attention_layernorm.weight",
                f"transformer.layers.{i}.post_attention_layernorm.bias",
                f"transformer.layers.{i}.mlp.dense_h_to_4h.weight",
                f"transformer.layers.{i}.mlp.dense_h_to_4h.bias",
                f"transformer.layers.{i}.mlp.dense_4h_to_h.weight",
                f"transformer.layers.{i}.mlp.dense_4h_to_h.bias",
            ]
        # 添加最终层的权重名称
        weight_names += [
            "transformer.final_layernorm.weight",
            "transformer.final_layernorm.bias",
        ]
        # 调用 dump_state_dict 函数保存权重信息到文件中
        dump_state_dict(f, weight_names, model.state_dict(), model.config.quantization_bit, ggml_type)
# 定义一个 ChatGLM2Converter 类，继承自 BaseConverter 类
class ChatGLM2Converter(BaseConverter):
    # 设置模型类型为 CHATGLM2
    MODEL_TYPE = ModelType.CHATGLM2

    # 静态方法，用于将配置信息写入文件
    @staticmethod
    def dump_config(f, config, ggml_type):
        # 断言配置中 add_bias_linear 必须为 False
        assert config.add_bias_linear is False, "unimplemented: add_bias_linear must be false"
        # 断言配置中 add_qkv_bias 必须为 True
        assert config.add_qkv_bias is True, "unimplemented: add_qkv_bias must be true"
        # 断言配置中 apply_residual_connection_post_layernorm 必须为 False
        assert (
            config.apply_residual_connection_post_layernorm is False
        ), "unimplemented: apply_residual_connection_post_layernorm must be false"
        # 断言配置中 kv_channels 乘以 num_attention_heads 必须等于 hidden_size
        assert (
            config.kv_channels * config.num_attention_heads == config.hidden_size
        ), "unimplemented: invalid kv_channels"
        # 断言配置中 multi_query_attention 必须为 True
        assert config.multi_query_attention is True, "unimplemented: multi_query_attention must be true"
        # 断言配置中 original_rope 必须为 True
        assert config.original_rope is True, "unimplemented: original_rope must be true"
        # 断言配置中 post_layer_norm 必须为 True
        assert config.post_layer_norm is True, "unimplemented: post_layer_norm must be true"
        # 断言配置中 rmsnorm 必须为 True
        assert config.rmsnorm is True, "unimplemented: rmsnorm must be true"

        # 将配置信息按照指定顺序写入文件
        config_values = [
            ggml_type.value,
            config.padded_vocab_size,
            config.hidden_size,
            config.num_attention_heads,
            config.num_layers,
            config.ffn_hidden_size,
            config.seq_length,
            config.bos_token_id if config.bos_token_id is not None else -1,
            config.eos_token_id if config.eos_token_id is not None else -1,
            config.pad_token_id if config.pad_token_id is not None else -1,
            config.sep_token_id if config.sep_token_id is not None else -1,
            config.multi_query_group_num,
        ]
        f.write(struct.pack("i" * len(config_values), *config_values)

    # 静态方法，用于将分词器信息写入文件
    @staticmethod
    def dump_tokenizer(f, tokenizer):
        # 获取序列化的模型协议
        serialized_model_proto = tokenizer.tokenizer.sp_model.serialized_model_proto()
        # 将序列化的模型协议长度写入文件
        f.write(struct.pack("i", len(serialized_model_proto)))
        # 将序列化的模型协议写入文件

    @staticmethod
    # 将模型的权重信息保存到文件中
    def dump_model(f, model, ggml_type):
        # 初始化权重名称列表，包含模型中需要保存的权重名称
        weight_names = ["transformer.embedding.word_embeddings.weight"]
        # 遍历模型的每一层，获取每一层的权重名称并添加到权重名称列表中
        for i in range(model.config.num_layers):
            weight_names += [
                f"transformer.encoder.layers.{i}.input_layernorm.weight",
                f"transformer.encoder.layers.{i}.self_attention.query_key_value.weight",
                f"transformer.encoder.layers.{i}.self_attention.query_key_value.bias",
                f"transformer.encoder.layers.{i}.self_attention.dense.weight",
                f"transformer.encoder.layers.{i}.post_attention_layernorm.weight",
                f"transformer.encoder.layers.{i}.mlp.dense_h_to_4h.weight",
                f"transformer.encoder.layers.{i}.mlp.dense_4h_to_h.weight",
            ]
        # 添加最后一层的权重名称到权重名称列表中
        weight_names += [
            "transformer.encoder.final_layernorm.weight",
            "transformer.output_layer.weight",
        ]
        # 调用函数将模型的状态字典保存到文件中
        dump_state_dict(f, weight_names, model.state_dict(), model.config.quantization_bit, ggml_type)
# ChatGLM3Converter 类继承自 ChatGLM2Converter 类
class ChatGLM3Converter(ChatGLM2Converter):
    # 设置 MODEL_TYPE 属性为 ModelType.CHATGLM3
    MODEL_TYPE = ModelType.CHATGLM3

# BaichuanConverter 类继承自 BaseConverter 类
class BaichuanConverter(BaseConverter):
    # 将配置信息写入文件
    @staticmethod
    def dump_config(f, config, ggml_type):
        # 断言 hidden_act 必须为 "silu"，否则抛出异常
        assert config.hidden_act == "silu", "unimplemented: hidden_act must be silu"

        # 将配置信息按照指定格式写入文件
        config_values = [
            ggml_type.value,
            config.vocab_size,
            config.hidden_size,
            config.num_attention_heads,
            config.num_hidden_layers,
            config.intermediate_size,
            config.model_max_length,
            config.bos_token_id if config.bos_token_id is not None else -1,
            config.eos_token_id if config.eos_token_id is not None else -1,
            config.pad_token_id if config.pad_token_id is not None else -1,
            config.sep_token_id if config.sep_token_id is not None else -1,
        ]
        f.write(struct.pack("i" * len(config_values), *config_values)

    # 将分词器信息写入文件
    @staticmethod
    def dump_tokenizer(f, tokenizer):
        # 获取序列化的模型协议
        serialized_model_proto = tokenizer.sp_model.serialized_model_proto()
        # 将序列化的模型协议的长度写入文件
        f.write(struct.pack("i", len(serialized_model_proto)))
        # 将序列化的模型协议写入文件

    @staticmethod
    # 定义一个函数，用于将模型的权重信息保存到文件中
    def dump_model(f, model, ggml_type):
        # 初始化权重名称列表，包括模型的嵌入权重
        weight_names = ["model.embed_tokens.weight"]
        # 遍历模型的隐藏层，添加每一层的权重名称
        for i in range(model.config.num_hidden_layers):
            weight_names += [
                f"model.layers.{i}.input_layernorm.weight",
                f"model.layers.{i}.self_attn.W_pack.weight",
                f"model.layers.{i}.self_attn.o_proj.weight",
                f"model.layers.{i}.post_attention_layernorm.weight",
                f"model.layers.{i}.mlp.gate_proj.weight",
                f"model.layers.{i}.mlp.down_proj.weight",
                f"model.layers.{i}.mlp.up_proj.weight",
            ]
        # 添加额外的权重名称，包括模型的归一化权重和语言模型头的权重
        weight_names += [
            "model.norm.weight",
            "lm_head.weight",
        ]

        # 如果模型的词汇大小为125696，则对lm_head的权重进行归一化处理
        if model.config.vocab_size == 125696:
            # 对lm_head的权重进行归一化处理
            model.lm_head.weight.data = F.normalize(model.lm_head.weight.data)

        # 调用dump_state_dict函数，将权重信息保存到文件中
        dump_state_dict(f, weight_names, model.state_dict(), quantization_bit=None, ggml_type=ggml_type)
# 定义 Baichuan7BConverter 类，继承自 BaichuanConverter 类
class Baichuan7BConverter(BaichuanConverter):
    # 设置 MODEL_TYPE 属性为 Baichuan7B
    MODEL_TYPE = ModelType.BAICHUAN7B

# 定义 Baichuan13BConverter 类，继承自 BaichuanConverter 类
class Baichuan13BConverter(BaichuanConverter):
    # 设置 MODEL_TYPE 属性为 Baichuan13B
    MODEL_TYPE = ModelType.BAICHUAN13B

# 定义 InternLMConverter 类，继承自 BaseConverter 类
class InternLMConverter(BaseConverter):
    # 设置 MODEL_TYPE 属性为 INTERNLM
    MODEL_TYPE = ModelType.INTERNLM

    # 定义 dump_config 静态方法，用于将配置信息写入文件
    @staticmethod
    def dump_config(f, config, ggml_type):
        # 断言 hidden_act 属性为 "silu"，否则抛出异常
        assert config.hidden_act == "silu", "unimplemented: hidden_act must be silu"

        # 将配置信息转换为列表
        config_values = [
            ggml_type.value,
            config.vocab_size,
            config.hidden_size,
            config.num_attention_heads,
            config.num_hidden_layers,
            config.intermediate_size,
            config.max_position_embeddings,
            config.bos_token_id if config.bos_token_id is not None else -1,
            config.eos_token_id if config.eos_token_id is not None else -1,
            config.pad_token_id if config.pad_token_id is not None else -1,
            config.sep_token_id if config.sep_token_id is not None else -1,
        ]

        # 将配置信息写入文件
        f.write(struct.pack("i" * len(config_values), *config_values))

    # 定义 dump_tokenizer 静态方法，用于将分词器信息写入文件
    @staticmethod
    def dump_tokenizer(f, tokenizer):
        # 获取序列化的模型协议
        serialized_model_proto = tokenizer.sp_model.serialized_model_proto()
        # 将序列化的模型协议长度写入文件
        f.write(struct.pack("i", len(serialized_model_proto)))
        # 将序列化的模型协议写入文件
        f.write(serialized_model_proto)

    # 定义 convert 方法，用于转换模型
    @staticmethod
    def convert(f: BinaryIO, model_name_or_path: str, lora_model_name_or_path: Optional[str] = None, dtype: str = "q4_0"):
        # 根据 dtype 获取 GGMLType 枚举类型
        ggml_type = GGMLType[dtype.upper()]

        # 根据模型名称或路径加载分词器
        tokenizer = AutoTokenizer.from_pretrained(model_name_or_path, trust_remote_code=True)

        # 根据模型名称或路径加载配置信息
        config = AutoConfig.from_pretrained(model_name_or_path, trust_remote_code=True)
        # 根据配置信息中的 auto_map 判断自动模型类
        if "AutoModel" in config.auto_map:
            auto_model_class = AutoModel
        elif "AutoModelForCausalLM" in config.auto_map:
            auto_model_class = AutoModelForCausalLM
        else:
            # 抛出异常，无法找到自动模型类
            raise RuntimeError(f"Cannot find auto model class to load {model_name_or_path}")
    # 从预训练模型名称或路径加载自动模型类，设置信任远程代码和低CPU内存使用
    model = auto_model_class.from_pretrained(model_name_or_path, trust_remote_code=True, low_cpu_mem_usage=True)

    # 如果存在另一个模型名称或路径
    if lora_model_name_or_path is not None:
        # 导入 PeftModel 类
        from peft import PeftModel

        # 从预训练模型和另一个模型名称或路径加载 PeftModel
        model = PeftModel.from_pretrained(model, lora_model_name_or_path)
        # 合并并卸载模型
        model = model.merge_and_unload()

    # 如果模型配置的模型类型是 "chatglm"
    if model.config.model_type == "chatglm":
        # 如果模型配置中存在 "multi_query_attention" 属性
        if hasattr(model.config, "multi_query_attention"):
            # 如果 transformers_version 为 "4.27.1"，则使用 ChatGLM2Converter 转换器
            if model.config.transformers_version == "4.27.1":
                ChatGLM2Converter.convert(f, model, tokenizer, ggml_type)
            else:
                # 否则使用 ChatGLM3Converter 转换器
                ChatGLM3Converter.convert(f, model, tokenizer, ggml_type)
        else:
            # 如果不存在 "multi_query_attention" 属性，则使用 ChatGLMConverter 转换器
            ChatGLMConverter.convert(f, model, tokenizer, ggml_type)
    # 如果模型配置的模型类型是 "baichuan"
    elif model.config.model_type == "baichuan":
        # 如果隐藏层大小为 5120，则使用 Baichuan13BConverter 转换器
        if model.config.hidden_size == 5120:
            Baichuan13BConverter.convert(f, model, tokenizer, ggml_type)
        else:
            # 否则使用 Baichuan7BConverter 转换器
            Baichuan7BConverter.convert(f, model, tokenizer, ggml_type)
    # 如果模型配置的模型类型是 "internlm"
    elif model.config.model_type == "internlm":
        # 使用 InternLMConverter 转换器
        InternLMConverter.convert(f, model, tokenizer, ggml_type)
    else:
        # 抛出运行时错误，指示未知的模型类型
        raise RuntimeError(f"Unknown model type {model.config.model_type}")
# 主函数，用于解析命令行参数并执行相应的操作
def main():
    # 创建参数解析器对象，指定程序名称
    parser = argparse.ArgumentParser("chatglm-convert")
    # 添加参数：模型名称或路径，默认为"THUDM/chatglm-6b"，字符串类型，用于AutoModel.from_pretrained
    parser.add_argument(
        "-i",
        "--model_name_or_path",
        default="THUDM/chatglm-6b",
        type=str,
        help="Model name or path used in AutoModel.from_pretrained",
    )
    # 添加参数：Lora模型名称或路径，默认为None，字符串类型，用于PeftModel.from_pretrained
    parser.add_argument(
        "-l",
        "--lora_model_name_or_path",
        default=None,
        type=str,
        help="Lora model name or path used in PeftModel.from_pretrained",
    )
    # 添加参数：保存生成的GGML模型的路径，默认为"chatglm-ggml.bin"，Path类型
    parser.add_argument(
        "-o", "--save_path", default="chatglm-ggml.bin", type=Path, help="Path to save the generated GGML model"
    )
    # 添加参数：GGML模型的量化类型，默认为"q4_0"，字符串类型，可选值为["f32", "f16", "q8_0", "q4_0", "q4_1", "q5_0", "q5_1"]
    parser.add_argument(
        "-t",
        "--type",
        default="q4_0",
        type=str,
        choices=["f32", "f16", "q8_0", "q4_0", "q4_1", "q5_0", "q5_1"],
        help="GGML model quantization type",
    )
    # 解析命令行参数
    args = parser.parse_args()

    # 打开保存路径的文件对象，以二进制写入模式
    with open(args.save_path, "wb") as f:
        # 调用convert函数，将模型转换为GGML格式并写入文件中，指定数据类型为args.type
        convert(f, args.model_name_or_path, dtype=args.type)

    # 打印提示信息，显示GGML模型保存的路径
    print(f"GGML model saved to {args.save_path}")


if __name__ == "__main__":
    # 执行主函数
    main()
```