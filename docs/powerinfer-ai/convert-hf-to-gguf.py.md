# `PowerInfer\convert-hf-to-gguf.py`

```cpp
#!/usr/bin/env python3

from __future__ import annotations  # 导入未来版本的注解特性

import argparse  # 导入命令行参数解析模块
import contextlib  # 导入上下文管理模块
import json  # 导入 JSON 模块
import os  # 导入操作系统相关模块
import re  # 导入正则表达式模块
import sys  # 导入系统相关模块
from enum import IntEnum  # 从枚举模块中导入 IntEnum 类
from pathlib import Path  # 从路径模块中导入 Path 类
from typing import TYPE_CHECKING, Any, ContextManager, Iterator, cast  # 导入类型提示相关模块

import numpy as np  # 导入 NumPy 模块
import torch  # 导入 PyTorch 模块

if TYPE_CHECKING:  # 如果是类型检查
    from torch import Tensor  # 从 PyTorch 模块中导入 Tensor 类型

if 'NO_LOCAL_GGUF' not in os.environ:  # 如果环境变量中没有 NO_LOCAL_GGUF
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))  # 将 gguf-py 目录添加到系统路径中
import gguf  # 导入 gguf 模块


###### MODEL DEFINITIONS ######

class SentencePieceTokenTypes(IntEnum):  # 定义 SentencePieceTokenTypes 枚举类，继承自 IntEnum
    NORMAL = 1  # 枚举值 1
    UNKNOWN = 2  # 枚举值 2
    CONTROL = 3  # 枚举值 3
    USER_DEFINED = 4  # 枚举值 4
    UNUSED = 5  # 枚举值 5
    BYTE = 6  # 枚举值 6


class Model:  # 定义 Model 类
    def __init__(self, dir_model: Path, ftype: int, fname_out: Path, is_big_endian: bool):  # 初始化方法，接受 dir_model、ftype、fname_out 和 is_big_endian 参数
        self.dir_model = dir_model  # 设置 dir_model 属性
        self.ftype = ftype  # 设置 ftype 属性
        self.fname_out = fname_out  # 设置 fname_out 属性
        self.is_big_endian = is_big_endian  # 设置 is_big_endian 属性
        self.endianess = gguf.GGUFEndian.BIG if is_big_endian else gguf.GGUFEndian.LITTLE  # 根据 is_big_endian 设置 endianess 属性
        self.is_safetensors = self._is_model_safetensors()  # 调用 _is_model_safetensors 方法，设置 is_safetensors 属性
        self.num_parts = Model.count_model_parts(self.dir_model, ".safetensors" if self.is_safetensors else ".bin")  # 调用 count_model_parts 方法，设置 num_parts 属性
        self.part_names = self._get_part_names()  # 调用 _get_part_names 方法，设置 part_names 属性
        self.hparams = Model.load_hparams(self.dir_model)  # 调用 load_hparams 方法，设置 hparams 属性
        self.model_arch = self._get_model_architecture()  # 调用 _get_model_architecture 方法，设置 model_arch 属性
        self.gguf_writer = gguf.GGUFWriter(fname_out, gguf.MODEL_ARCH_NAMES[self.model_arch], endianess=self.endianess)  # 创建 GGUFWriter 对象，设置 gguf_writer 属性

    def set_vocab(self):  # 定义 set_vocab 方法
        self._set_vocab_gpt2()  # 调用 _set_vocab_gpt2 方法
    # 返回一个迭代器，迭代器的元素是元组，包含字符串和张量
    def get_tensors(self) -> Iterator[tuple[str, Tensor]]:
        # 遍历模型部分的名称
        for part_name in self.part_names:
            # 打印加载模型部分的信息
            print(f"gguf: loading model part '{part_name}'")
            # 声明上下文管理器
            ctx: ContextManager[Any]
            # 如果使用安全张量，则从safetensors模块中导入safe_open函数，并使用其返回的上下文管理器
            if self.is_safetensors:
                from safetensors import safe_open
                ctx = cast(ContextManager[Any], safe_open(self.dir_model / part_name, framework="pt", device="cpu"))
            # 否则，使用torch.load加载模型部分，并指定map_location为"cpu"
            else:
                ctx = contextlib.nullcontext(torch.load(self.dir_model / part_name, map_location="cpu"))

            # 进入上下文管理器，加载模型部分的张量数据
            with ctx as model_part:
                # 遍历模型部分的键
                for name in model_part.keys():
                    # 如果使用安全张量，则通过get_tensor方法获取张量数据，否则直接通过索引获取张量数据
                    data = model_part.get_tensor(name) if self.is_safetensors else model_part[name]
                    # 返回模型部分的名称和张量数据
                    yield name, data

    # 设置gguf参数
    def set_gguf_parameters(self):
        # 向gguf_writer添加模型目录的名称
        self.gguf_writer.add_name(self.dir_model.name)
        # 向gguf_writer添加块的数量，优先使用"hparams"中的"n_layers"，其次使用"num_hidden_layers"，最后使用"n_layer"
        self.gguf_writer.add_block_count(self.hparams.get(
            "n_layers", self.hparams.get("num_hidden_layers", self.hparams.get("n_layer")),
        ))
        # 如果"hparams"中存在"max_position_embeddings"，则向gguf_writer添加上下文长度
        if (n_ctx := self.hparams.get("max_position_embeddings")) is not None:
            self.gguf_writer.add_context_length(n_ctx)
        # 如果"hparams"中存在"hidden_size"，则向gguf_writer添加嵌入长度
        if (n_embd := self.hparams.get("hidden_size")) is not None:
            self.gguf_writer.add_embedding_length(n_embd)
        # 如果"hparams"中存在"intermediate_size"，则向gguf_writer添加前馈长度
        if (n_ff := self.hparams.get("intermediate_size")) is not None:
            self.gguf_writer.add_feed_forward_length(n_ff)
        # 如果"hparams"中存在"num_attention_head"，则向gguf_writer添加头的数量
        if (n_head := self.hparams.get("num_attention_head")) is not None:
            self.gguf_writer.add_head_count(n_head)
        # 向gguf_writer添加并行残差的使用情况，默认为True
        self.gguf_writer.add_parallel_residual(self.hparams.get("use_parallel_residual", True))
    # 定义一个方法用于写入张量数据
    def write_tensors(self):
        # 获取模型的层数，如果没有指定，则使用默认值
        block_count = self.hparams.get("n_layers", self.hparams.get("num_hidden_layers", self.hparams.get("n_layer")))
        # 获取张量名称映射
        tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)
        # 遍历获取的张量数据
        for name, data_torch in self.get_tensors():
            # 如果张量名称以指定的后缀结尾，则跳过
            if name.endswith((".attention.masked_bias", ".attention.bias", ".attention.rotary_emb.inv_freq")):
                continue

            # 保存原始数据类型
            old_dtype = data_torch.dtype

            # 将不支持的数据类型转换为float32
            if data_torch.dtype not in (torch.float16, torch.float32):
                data_torch = data_torch.to(torch.float32)

            # 将张量数据转换为numpy数组
            data = data_torch.squeeze().numpy()

            # 映射张量名称
            new_name = tensor_map.get_name(name, try_suffixes=(".weight", ".bias"))
            if new_name is None:
                # 如果无法映射张量名称，则打印错误信息并退出程序
                print(f"Can not map tensor {name!r}")
                sys.exit()

            # 获取数据维度和数据类型
            n_dims = len(data.shape)
            data_dtype = data.dtype

            # 如果需要float32类型的数据，则将float16转换为float32
            if self.ftype == 0 and data_dtype == np.float16:
                data = data.astype(np.float32)

            # TODO: 为什么不能直接使用这些float16？应该没有理由将float16存储为float32
            if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
                # 如果需要float16类型的数据，并且数据维度为1，则将float16转换为float32
                data = data.astype(np.float32)

            # 如果需要float16类型的数据，并且数据类型为float32，数据名称以".weight"结尾，并且数据维度为2，则将float32转换为float16
            if self.ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
                data = data.astype(np.float16)

            # 打印转换后的张量名称、数据维度和数据类型
            print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")

            # 将转换后的张量数据添加到gguf_writer中
            self.gguf_writer.add_tensor(new_name, data)
    # 写入模型数据，包括张量数据、头部数据、键值数据和张量数据
    def write(self):
        self.write_tensors()  # 写入张量数据
        self.gguf_writer.write_header_to_file()  # 将头部数据写入文件
        self.gguf_writer.write_kv_data_to_file()  # 将键值数据写入文件
        self.gguf_writer.write_tensors_to_file()  # 将张量数据写入文件
        self.gguf_writer.close()  # 关闭写入器

    # 写入词汇表数据，包括头部数据和键值数据
    def write_vocab(self):
        self.gguf_writer.write_header_to_file()  # 将头部数据写入文件
        self.gguf_writer.write_kv_data_to_file()  # 将键值数据写入文件
        self.gguf_writer.close()  # 关闭写入器

    # 统计模型部件的数量
    @staticmethod
    def count_model_parts(dir_model: Path, prefix: str) -> int:
        num_parts = 0
        for filename in os.listdir(dir_model):
            if filename.endswith(prefix):  # 如果文件名以指定前缀结尾
                num_parts += 1  # 数量加一
        return num_parts  # 返回部件数量

    # 加载超参数配置
    @staticmethod
    def load_hparams(dir_model):
        with open(dir_model / "config.json", "r", encoding="utf-8") as f:  # 打开配置文件
            return json.load(f)  # 加载并返回 JSON 格式的配置数据

    # 根据模型架构返回对应的模型类
    @staticmethod
    def from_model_architecture(model_architecture):
        # 根据模型架构返回对应的模型类
        # ...

    # 检查模型是否包含安全张量数据
    def _is_model_safetensors(self) -> bool:
        return Model.count_model_parts(self.dir_model, ".safetensors") > 0  # 判断是否存在安全张量数据，返回布尔值
    # 获取部分名称的私有方法
    def _get_part_names(self):
        # 如果是安全张量
        if self.is_safetensors:
            # 如果只有一个 .safetensors 文件
            if self.num_parts == 1:
                return ("model.safetensors",)
            # 返回一系列 .safetensors 文件名
            return (f"model-{n:05}-of-{self.num_parts:05}.safetensors" for n in range(1, self.num_parts + 1))
        
        # 如果不是安全张量
        if self.num_parts == 1:
            return ("pytorch_model.bin",)  # 如果只有一个 .bin 文件
        # 返回一系列 .bin 文件名
        return (f"pytorch_model-{n:05}-of-{self.num_parts:05}.bin" for n in range(1, self.num_parts + 1))

    # 获取模型架构的私有方法
    def _get_model_architecture(self) -> gguf.MODEL_ARCH:
        # 获取模型架构
        arch = self.hparams["architectures"][0]
        # 根据不同的架构返回对应的枚举值
        if arch == "GPTNeoXForCausalLM":
            return gguf.MODEL_ARCH.GPTNEOX
        if arch == "BloomForCausalLM":
            return gguf.MODEL_ARCH.BLOOM
        if arch == "MPTForCausalLM":
            return gguf.MODEL_ARCH.MPT
        if arch in ("BaichuanForCausalLM", "BaiChuanForCausalLM"):
            return gguf.MODEL_ARCH.BAICHUAN
        if arch == "FalconForCausalLM":
            return gguf.MODEL_ARCH.FALCON
        if arch == "GPTBigCodeForCausalLM":
            return gguf.MODEL_ARCH.STARCODER
        if arch == "GPTRefactForCausalLM":
            return gguf.MODEL_ARCH.REFACT
        if arch == "PersimmonForCausalLM":
            return gguf.MODEL_ARCH.PERSIMMON
        if arch in ("StableLMEpochForCausalLM", "LlavaStableLMEpochForCausalLM"):
            return gguf.MODEL_ARCH.STABLELM
        
        # 如果架构不在已知的枚举值中，则抛出未实现的错误
        raise NotImplementedError(f'Architecture "{arch}" not supported!')
    # 设置 GPT2 的词汇表
    def _set_vocab_gpt2(self):
        # 获取模型目录和超参数
        dir_model = self.dir_model
        hparams = self.hparams
        # 初始化 tokens 和 toktypes 列表
        tokens: list[bytearray] = []
        toktypes: list[int] = []

        # 导入 AutoTokenizer 类
        from transformers import AutoTokenizer  # type: ignore[attr-defined]
        # 从预训练模型目录创建 tokenizer 对象
        tokenizer = AutoTokenizer.from_pretrained(dir_model)
        # 获取词汇表大小，如果未指定则使用 tokenizer 的词汇表大小
        vocab_size = hparams.get("vocab_size", len(tokenizer.vocab))
        # 断言 tokenizer 的词汇表中的最大值小于指定的词汇表大小
        assert max(tokenizer.vocab.values()) < vocab_size

        # 创建反向词汇表，将编码的词转换为 ID
        reverse_vocab = {id_: encoded_tok for encoded_tok, id_ in tokenizer.vocab.items()}
        # 获取 tokenizer 的添加词汇表
        added_vocab = tokenizer.get_added_vocab()

        # 遍历词汇表大小
        for i in range(vocab_size):
            # 如果 ID 不在反向词汇表中
            if i not in reverse_vocab:
                # 创建并添加用户定义的填充标记
                pad_token = f"[PAD{i}]".encode('utf-8')
                tokens.append(bytearray(pad_token))
                toktypes.append(gguf.TokenType.USER_DEFINED)
            # 如果反向词汇表中的词在添加词汇表中
            elif reverse_vocab[i] in added_vocab:
                # 添加反向词汇表中的词
                tokens.append(reverse_vocab[i])
                # 根据添加的特殊标记类型添加对应的类型
                if tokenizer.added_tokens_decoder[i].special:
                    toktypes.append(gguf.TokenType.CONTROL)
                else:
                    toktypes.append(gguf.TokenType.USER_DEFINED)
            # 如果反向词汇表中的词不在添加词汇表中
            else:
                # 添加反向词汇表中的词
                tokens.append(reverse_vocab[i])
                # 添加普通词汇类型
                toktypes.append(gguf.TokenType.NORMAL)

        # 将 tokenizer 模型添加到 gguf_writer
        self.gguf_writer.add_tokenizer_model("gpt2")
        # 将 tokens 列表添加到 gguf_writer
        self.gguf_writer.add_token_list(tokens)
        # 将 toktypes 列表添加到 gguf_writer
        self.gguf_writer.add_token_types(toktypes)

        # 创建特殊词汇表对象
        special_vocab = gguf.SpecialVocab(dir_model, load_merges=True)
        # 将特殊词汇表添加到 gguf_writer
        special_vocab.add_to_gguf(self.gguf_writer)
# 定义 GPTNeoXModel 类，继承自 Model 类
class GPTNeoXModel(Model):
    # 设置 GGUF 参数
    def set_gguf_parameters(self):
        # 获取隐藏层的数量
        block_count = self.hparams["num_hidden_layers"]

        # 添加模型名称到 GGUF 写入器
        self.gguf_writer.add_name(self.dir_model.name)
        # 添加最大位置嵌入长度到 GGUF 写入器
        self.gguf_writer.add_context_length(self.hparams["max_position_embeddings"])
        # 添加隐藏层大小到 GGUF 写入器
        self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])
        # 添加块数量到 GGUF 写入器
        self.gguf_writer.add_block_count(block_count)
        # 添加前馈网络长度到 GGUF 写入器
        self.gguf_writer.add_feed_forward_length(self.hparams["intermediate_size"])
        # 添加绳索维度数量到 GGUF 写入器
        self.gguf_writer.add_rope_dimension_count(
            int(self.hparams["rotary_pct"] * (self.hparams["hidden_size"] // self.hparams["num_attention_heads"])),
        )
        # 添加注意力头数量到 GGUF 写入器
        self.gguf_writer.add_head_count(self.hparams["num_attention_heads"])
        # 添加并行残差连接到 GGUF 写入器
        self.gguf_writer.add_parallel_residual(self.hparams.get("use_parallel_residual", True))
        # 添加层归一化 epsilon 到 GGUF 写入器
        self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_eps"])


# 定义 BloomModel 类，继承自 Model 类
class BloomModel(Model):
    # 设置 GGUF 参数
    def set_gguf_parameters(self):
        # 添加模型名称到 GGUF 写入器
        self.gguf_writer.add_name("Bloom")
        # 获取隐藏层大小或嵌入维度
        n_embed = self.hparams.get("hidden_size", self.hparams.get("n_embed"))
        # 获取注意力头数量
        n_head = self.hparams.get("n_head", self.hparams.get("num_attention_heads"))
        # 添加上下文长度到 GGUF 写入器
        self.gguf_writer.add_context_length(self.hparams.get("seq_length", n_embed))
        # 添加嵌入长度到 GGUF 写入器
        self.gguf_writer.add_embedding_length(n_embed)
        # 添加前馈网络长度到 GGUF 写入器
        self.gguf_writer.add_feed_forward_length(4 * n_embed)
        # 添加块数量到 GGUF 写入器
        self.gguf_writer.add_block_count(self.hparams["n_layer"])
        # 添加注意力头数量到 GGUF 写入器
        self.gguf_writer.add_head_count(n_head)
        # 添加键值注意力头数量到 GGUF 写入器
        self.gguf_writer.add_head_count_kv(n_head)
        # 添加层归一化 epsilon 到 GGUF 写入器
        self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_epsilon"])
        # 添加文件类型到 GGUF 写入器
        self.gguf_writer.add_file_type(self.ftype)

# 定义 MPTModel 类，继承自 Model 类
class MPTModel(Model):
    # 设置 GGUF 参数
    # 获取层的数量
    block_count = self.hparams["n_layers"]
    # 添加模型名称到 GGUF 写入器
    self.gguf_writer.add_name(self.dir_model.name)
    # 添加上下文长度到 GGUF 写入器
    self.gguf_writer.add_context_length(self.hparams["max_seq_len"])
    # 添加嵌入长度到 GGUF 写入器
    self.gguf_writer.add_embedding_length(self.hparams["d_model"])
    # 添加块数量到 GGUF 写入器
    self.gguf_writer.add_block_count(block_count)
    # 添加前馈长度到 GGUF 写入器
    self.gguf_writer.add_feed_forward_length(4 * self.hparams["d_model"])
    # 添加头的数量到 GGUF 写入器
    self.gguf_writer.add_head_count(self.hparams["n_heads"])
    # 如果存在 kv_n_heads，则添加 kv_n_heads 到 GGUF 写入器
    if kv_n_heads := self.hparams["attn_config"].get("kv_n_heads"):
        self.gguf_writer.add_head_count_kv(kv_n_heads)
    # 添加层归一化的 epsilon 到 GGUF 写入器
    self.gguf_writer.add_layer_norm_eps(1e-5)
    # 如果存在 clip_qkv，则添加 clip_qkv 到 GGUF 写入器
    if self.hparams["attn_config"]["clip_qkv"] is not None:
        self.gguf_writer.add_clamp_kqv(self.hparams["attn_config"]["clip_qkv"])
    # 添加最大 alibi 偏差到 GGUF 写入器
    self.gguf_writer.add_max_alibi_bias(self.hparams["attn_config"]["alibi_bias_max"])
class BaichuanModel(Model):
    # 设置词汇表
    def set_vocab(self):
        self._set_vocab_sentencepiece()

    # 设置 GGUF 参数
    def set_gguf_parameters(self):
        # 获取隐藏层的数量
        block_count = self.hparams["num_hidden_layers"]
        # 获取注意力头的数量
        head_count = self.hparams["num_attention_heads"]
        # 获取键值头的数量
        head_count_kv = self.hparams.get("num_key_value_heads", head_count)
        # 获取 HF 仓库的名称或路径
        hf_repo = self.hparams.get("_name_or_path", "")

        # 上下文长度初始化为 0
        ctx_length = 0
        # 根据参数中的不同字段设置上下文长度
        if "max_sequence_length" in self.hparams:
            ctx_length = self.hparams["max_sequence_length"]
        elif "max_position_embeddings" in self.hparams:
            ctx_length = self.hparams["max_position_embeddings"]
        elif "model_max_length" in self.hparams:
            ctx_length = self.hparams["model_max_length"]
        else:
            # 如果找不到上下文长度参数，则打印错误信息并退出程序
            print("gguf: can not find ctx length parameter.")
            sys.exit()

        # 添加模型名称到 GGUF 写入器
        self.gguf_writer.add_name(self.dir_model.name)
        # 添加 HF 仓库到 GGUF 写入器
        self.gguf_writer.add_source_hf_repo(hf_repo)
        # 添加张量数据布局到 GGUF 写入器
        self.gguf_writer.add_tensor_data_layout("Meta AI original pth")
        # 添加上下文长度到 GGUF 写入器
        self.gguf_writer.add_context_length(ctx_length)
        # 添加嵌入长度到 GGUF 写入器
        self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])
        # 添加块数量到 GGUF 写入器
        self.gguf_writer.add_block_count(block_count)
        # 添加前馈长度到 GGUF 写入器
        self.gguf_writer.add_feed_forward_length(self.hparams["intermediate_size"])
        # 添加绳索维度数量到 GGUF 写入器
        self.gguf_writer.add_rope_dimension_count(self.hparams["hidden_size"] // self.hparams["num_attention_heads"])
        # 添加注意力头数量到 GGUF 写入器
        self.gguf_writer.add_head_count(head_count)
        # 添加键值头数量到 GGUF 写入器
        self.gguf_writer.add_head_count_kv(head_count_kv)
        # 添加层归一化 RMS 衰减参数到 GGUF 写入器
        self.gguf_writer.add_layer_norm_rms_eps(self.hparams["rms_norm_eps"])

        # 如果参数中包含绳索缩放信息，并且包含线性缩放因子
        if self.hparams.get("rope_scaling") is not None and "factor" in self.hparams["rope_scaling"]:
            # 如果绳索缩放类型为线性，则添加到 GGUF 写入器
            if self.hparams["rope_scaling"].get("type") == "linear":
                self.gguf_writer.add_rope_scaling_type(gguf.RopeScalingType.LINEAR)
                # 添加线性缩放因子到 GGUF 写入器
                self.gguf_writer.add_rope_scaling_factor(self.hparams["rope_scaling"]["factor"])
    # 反转哈希分桶的排列，将权重张量进行重新排列
    def _reverse_hf_permute(self, weights: Tensor, n_head: int, n_kv_head: int | None = None) -> Tensor:
        # 如果指定了键值头的数量，并且头的数量不等于键值头的数量，则将头的数量除以键值头的数量
        if n_kv_head is not None and n_head != n_kv_head:
            n_head //= n_kv_head
    
        # 将权重张量进行形状重塑，然后交换轴，最后再次重塑
        return (
            weights.reshape(n_head, 2, weights.shape[0] // n_head // 2, *weights.shape[1:])
            .swapaxes(1, 2)
            .reshape(weights.shape)
        )
    
    # 反转哈希分桶的排列的一部分，将权重张量的一部分进行重新排列
    def _reverse_hf_permute_part(
        self, weights: Tensor, n_part: int, n_head: int, n_head_kv: int | None = None,
    ) -> Tensor:
        # 计算每部分的大小
        r = weights.shape[0] // 3
        # 调用_reverse_hf_permute函数，对权重张量的一部分进行重新排列
        return self._reverse_hf_permute(weights[r * n_part:r * n_part + r, ...], n_head, n_head_kv)
    
    # 反转哈希分桶的一部分，将权重张量的一部分返回
    def _reverse_hf_part(self, weights: Tensor, n_part: int) -> Tensor:
        # 计算每部分的大小
        r = weights.shape[0] // 3
        # 返回权重张量的一部分
        return weights[r * n_part:r * n_part + r, ...]
# 定义 FalconModel 类，继承自 Model 类
class FalconModel(Model):
    # 设置 GGUF 参数
    def set_gguf_parameters(self):
        # 获取隐藏层的数量
        block_count = self.hparams.get("num_hidden_layers")
        # 如果隐藏层数量为空，则使用旧的名称 n_layer
        if block_count is None:
            block_count = self.hparams["n_layer"]  # old name

        # 获取注意力头的数量
        n_head = self.hparams.get("num_attention_heads")
        # 如果注意力头数量为空，则使用旧的名称 n_head
        if n_head is None:
            n_head = self.hparams["n_head"]  # old name

        # 获取键值头的数量
        n_head_kv = self.hparams.get("num_kv_heads")
        # 如果键值头数量为空，则使用旧的名称 n_head_kv，如果仍为空，则默认为 1
        if n_head_kv is None:
            n_head_kv = self.hparams.get("n_head_kv", 1)  # old name

        # 添加名称为 "Falcon" 的 GGUF 参数
        self.gguf_writer.add_name("Falcon")
        # 添加上下文长度为 2048，不在 config.json 中
        self.gguf_writer.add_context_length(2048)
        # 添加张量数据布局为 "jploski"，用于 qkv 张量转换
        self.gguf_writer.add_tensor_data_layout("jploski")
        # 添加嵌入长度为隐藏层大小
        self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])
        # 添加前馈长度为隐藏层大小的 4 倍
        self.gguf_writer.add_feed_forward_length(4 * self.hparams["hidden_size"])
        # 添加块数量
        self.gguf_writer.add_block_count(block_count)
        # 添加注意力头数量
        self.gguf_writer.add_head_count(n_head)
        # 添加键值头数量
        self.gguf_writer.add_head_count_kv(n_head_kv)
        # 添加层归一化的 epsilon 值
        self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_epsilon"])
        # 添加文件类型
        self.gguf_writer.add_file_type(self.ftype)

# 定义 StarCoderModel 类，继承自 Model 类
class StarCoderModel(Model):
    # 设置 GGUF 参数
    def set_gguf_parameters(self):
        # 获取隐藏层的数量
        block_count = self.hparams["n_layer"]

        # 添加名称为 "StarCoder" 的 GGUF 参数
        self.gguf_writer.add_name("StarCoder")
        # 添加上下文长度为 n_positions
        self.gguf_writer.add_context_length(self.hparams["n_positions"])
        # 添加嵌入长度为 n_embd
        self.gguf_writer.add_embedding_length(self.hparams["n_embd"])
        # 添加前馈长度为 n_embd 的 4 倍
        self.gguf_writer.add_feed_forward_length(4 * self.hparams["n_embd"])
        # 添加块数量
        self.gguf_writer.add_block_count(block_count)
        # 添加注意力头数量
        self.gguf_writer.add_head_count(self.hparams["n_head"])
        # 添加键值头数量为 1
        self.gguf_writer.add_head_count_kv(1)
        # 添加层归一化的 epsilon 值
        self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_epsilon"])
        # 添加文件类型
        self.gguf_writer.add_file_type(self.ftype)

# 定义 RefactModel 类，继承自 Model 类
class RefactModel(Model):
    # 此处省略了具体的方法实现
    # 设置 GGUF 参数
    hidden_dim = self.hparams["n_embd"]  # 从超参数中获取隐藏层维度
    inner_dim = 4 * hidden_dim  # 内部维度是隐藏层维度的四倍
    hidden_dim = int(2 * inner_dim / 3)  # 重新计算隐藏层维度
    multiple_of = 256  # 设置倍数
    ff_dim = multiple_of * ((hidden_dim + multiple_of - 1) // multiple_of)  # 计算前馈网络维度

    block_count = self.hparams["n_layer"]  # 从超参数中获取块数量

    self.gguf_writer.add_name("Refact")  # 添加名称为 "Refact" 的信息到 GGUF 写入器
    self.gguf_writer.add_context_length(self.hparams["n_positions"])  # 添加上下文长度到 GGUF 写入器
    self.gguf_writer.add_embedding_length(self.hparams["n_embd"])  # 添加嵌入长度到 GGUF 写入器

    self.gguf_writer.add_feed_forward_length(ff_dim)  # 添加前馈网络长度到 GGUF 写入器
    self.gguf_writer.add_block_count(block_count)  # 添加块数量到 GGUF 写入器
    self.gguf_writer.add_head_count(self.hparams["n_head"])  # 添加头数量到 GGUF 写入器
    self.gguf_writer.add_head_count_kv(1)  # 添加键值头数量到 GGUF 写入器
    self.gguf_writer.add_layer_norm_rms_eps(self.hparams["layer_norm_epsilon"])  # 添加层归一化 RMS Epsilon 到 GGUF 写入器
    self.gguf_writer.add_file_type(self.ftype)  # 添加文件类型到 GGUF 写入器
class PersimmonModel(Model):
    # 设置 GGUF 参数
    def set_gguf_parameters(self):
        # 获取块的数量，如果没有指定则使用隐藏层的数量
        block_count = self.hparams.get("num_layers", self.hparams.get("num_hidden_layers"))
        # 获取注意力头的数量
        head_count = self.hparams["num_attention_heads"]
        # 复制注意力头的数量
        head_count_kv = head_count
        # 获取隐藏层的大小
        hidden_size = self.hparams["hidden_size"]

        # 添加名称到 GGUF 写入器
        self.gguf_writer.add_name('persimmon-8b-chat')
        # 添加嵌入长度到 GGUF 写入器
        self.gguf_writer.add_embedding_length(hidden_size)
        # 添加块的数量到 GGUF 写入器
        self.gguf_writer.add_block_count(block_count)
        # 添加前馈网络长度到 GGUF 写入器
        self.gguf_writer.add_feed_forward_length(self.hparams["intermediate_size"])
        # 添加绳索维度数量到 GGUF 写入器
        self.gguf_writer.add_rope_dimension_count(hidden_size // head_count)
        # 添加注意力头的数量到 GGUF 写入器
        self.gguf_writer.add_head_count(head_count)
        # 添加键值注意力头的数量到 GGUF 写入器
        self.gguf_writer.add_head_count_kv(head_count_kv)
        # 添加绳索频率基数到 GGUF 写入器
        self.gguf_writer.add_rope_freq_base(self.hparams["rope_theta"])
        # 添加层归一化 epsilon 到 GGUF 写入器
        self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_eps"])
        # 添加层归一化 RMS epsilon 到 GGUF 写入器
        self.gguf_writer.add_layer_norm_rms_eps(self.hparams["rms_norm_eps"])

    # 设置词汇表
    def set_vocab(self):
        # 设置 SentencePiece 词汇表
        self._set_vocab_sentencepiece()
        # 添加起始标记的 token id 到 GGUF 写入器
        # self.gguf_writer.add_bos_token_id(71013)
        # 添加结束标记的 token id 到 GGUF 写入器
        # self.gguf_writer.add_eos_token_id(71013)
    # 定义一个方法用于写入张量数据
    def write_tensors(self):
        # 获取块的数量，优先使用参数中的"num_layers"，如果不存在则使用"num_hidden_layers"
        block_count = self.hparams.get("num_layers", self.hparams.get("num_hidden_layers"))
        # 获取张量名称映射
        tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)

        # 遍历获取的张量数据
        for name, data_torch in self.get_tensors():
            # 如果张量名称以".self_attention.rotary_emb.inv_freq"结尾，则跳过
            if name.endswith(".self_attention.rotary_emb.inv_freq"):
                continue
            # 保存原始数据类型
            old_dtype = data_torch.dtype
            # 将张量数据转换为float32类型，压缩维度并转换为numpy数组
            data = data_torch.to(torch.float32).squeeze().numpy()
            # 获取新的张量名称
            new_name = tensor_map.get_name(name, try_suffixes=(".weight", ".bias"))
            # 如果找不到新的张量名称，则打印错误信息并退出程序
            if new_name is None:
                print(f"Can not map tensor {name!r}")
                sys.exit()
            # 获取数据的维度数量，并打印数据类型转换信息
            n_dims = len(data.shape)
            print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")
            # 将新的张量名称和数据添加到gguf_writer中
            self.gguf_writer.add_tensor(new_name, data)
# 定义一个名为 StableLMModel 的类，继承自 Model 类
class StableLMModel(Model):
    # 定义一个名为 set_gguf_parameters 的方法
    def set_gguf_parameters(self):
        # 获取超参数
        hparams = self.hparams
        # 获取隐藏层的数量
        block_count = hparams["num_hidden_layers"]

        # 将模型名称添加到 gguf_writer 中
        self.gguf_writer.add_name(dir_model.name)
        # 将最大位置嵌入长度添加到 gguf_writer 中
        self.gguf_writer.add_context_length(hparams["max_position_embeddings"])
        # 将隐藏层大小添加到 gguf_writer 中
        self.gguf_writer.add_embedding_length(hparams["hidden_size"])
        # 将块数量添加到 gguf_writer 中
        self.gguf_writer.add_block_count(block_count)
        # 将前馈神经网络长度添加到 gguf_writer 中
        self.gguf_writer.add_feed_forward_length(hparams["intermediate_size"])
        # 将绳索维度数量添加到 gguf_writer 中
        self.gguf_writer.add_rope_dimension_count(int(hparams["rope_pct"]*(hparams["hidden_size"] // hparams["num_attention_heads"])))
        # 将注意力头数量添加到 gguf_writer 中
        self.gguf_writer.add_head_count(hparams["num_attention_heads"])
        # 将是否使用并行残差添加到 gguf_writer 中
        self.gguf_writer.add_parallel_residual(hparams["use_parallel_residual"] if "use_parallel_residual" in hparams else True)
        # 将层归一化 epsilon 添加到 gguf_writer 中
        self.gguf_writer.add_layer_norm_eps(1e-5)

# 定义一个名为 parse_args 的方法，返回类型为 argparse.Namespace
def parse_args() -> argparse.Namespace:
    # 创建一个参数解析器对象
    parser = argparse.ArgumentParser(description="Convert a huggingface model to a GGML compatible file")
    # 添加一个参数，用于提取仅词汇表
    parser.add_argument(
        "--vocab-only", action="store_true",
        help="extract only the vocab",
    )
    # 添加一个参数，用于指定输出文件路径
    parser.add_argument(
        "--outfile", type=Path,
        help="path to write to; default: based on input",
    )
    # 添加一个参数，用于指定输出格式
    parser.add_argument(
        "--outtype", type=str, choices=["f32", "f16"], default="f16",
        help="output format - use f32 for float32, f16 for float16",
    )
    # 添加一个参数，用于指示模型是否在大端机器上执行
    parser.add_argument("--bigendian", action="store_true", help="model is executed on big endian machine")
    # 添加一个参数，用于指定模型目录
    parser.add_argument(
        "model", type=Path,
        help="directory containing model file",
    )

    # 解析并返回参数
    return parser.parse_args()

# 解析命令行参数
args = parse_args()

# 获取模型目录
dir_model = args.model
# 如果模型目录不存在，则打印错误信息并退出程序
if not dir_model.is_dir():
    print(f'Error: {args.model} is not a directory', file=sys.stderr)
    sys.exit(1)

# 定义一个字典，用于将输出格式映射为 GGMLQuantizationType 枚举类型
ftype_map = {
    "f32": gguf.GGMLQuantizationType.F32,
    "f16": gguf.GGMLQuantizationType.F16,
}

# 如果指定了输出文件路径
if args.outfile is not None:
    # 从参数中获取输出文件名
    fname_out = args.outfile
# 如果条件不成立，则将输出文件名设置为与模型相同的目录
else:
    fname_out = dir_model / f'ggml-model-{args.outtype}.gguf'

# 打印加载模型的信息
print(f"Loading model: {dir_model.name}")

# 从模型目录加载超参数
hparams = Model.load_hparams(dir_model)

# 根据超参数中的模型架构创建模型类
model_class = Model.from_model_architecture(hparams["architectures"][0])

# 根据模型类创建模型实例
model_instance = model_class(dir_model, ftype_map[args.outtype], fname_out, args.bigendian)

# 设置模型参数
print("Set model parameters")
model_instance.set_gguf_parameters()

# 设置模型分词器
print("Set model tokenizer")
model_instance.set_vocab()

# 如果只需要导出词汇表
if args.vocab_only:
    print(f"Exporting model vocab to '{fname_out}'")
    model_instance.write_vocab()
# 否则
else:
    print(f"Exporting model to '{fname_out}'")
    model_instance.write()

# 打印成功导出模型的信息
print(f"Model successfully exported to '{fname_out}'")
```