# `PowerInfer\convert-hf-to-powerinfer-gguf.py`

```
#!/usr/bin/env python3
# 指定使用 Python3 解释器

from __future__ import annotations
# 导入未来版本的特性，用于支持类型注解

from abc import ABC, abstractmethod
# 导入抽象基类和抽象方法装饰器

import argparse
# 导入用于解析命令行参数的模块

import contextlib
# 导入上下文管理器的模块

import json
# 导入用于处理 JSON 数据的模块

import os
# 导入用于与操作系统交互的模块

import re
# 导入用于处理正则表达式的模块

import struct
# 导入用于处理字节数据的模块

import sys
# 导入用于与 Python 解释器交互的模块

from enum import IntEnum
# 导入用于创建枚举类型的模块

from pathlib import Path
# 导入用于处理文件路径的模块

from typing import TYPE_CHECKING, Any, ContextManager, Iterator, Optional, cast
# 导入用于类型注解的模块

import numpy as np
# 导入用于处理数值数据的模块

import torch
import torch.nn as tnn
# 导入用于构建神经网络的模块
# 导入 dataclass 模块中的 dataclass 类
from dataclasses import dataclass

# 如果 TYPE_CHECKING 为真，则从 torch 模块中导入 Tensor 类
if TYPE_CHECKING:
    from torch import Tensor

# 如果环境变量中没有 "NO_LOCAL_GGUF"，则将 gguf-py 目录添加到系统路径中
if "NO_LOCAL_GGUF" not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / "gguf-py"))
import gguf

# 定义 SentencePieceTokenTypes 枚举类
class SentencePieceTokenTypes(IntEnum):
    NORMAL = 1
    UNKNOWN = 2
    CONTROL = 3
    USER_DEFINED = 4
    UNUSED = 5
    BYTE = 6
# 定义一个名为 ReluMLP 的类，继承自 tnn.Module
class ReluMLP(tnn.Module):
    # 初始化方法，接受输入维度、隐藏层维度和输出维度作为参数
    def __init__(self, input_dim: int, hidden_dim: int, output_dim: int):
        # 调用父类的初始化方法
        super(ReluMLP, self).__init__()
        # 创建一个全连接层，输入维度为 input_dim，输出维度为 hidden_dim，不使用偏置
        self.fc1 = tnn.Linear(input_dim, hidden_dim, bias=False)
        # 创建一个 ReLU 激活函数
        self.relu = tnn.ReLU()
        # 创建一个全连接层，输入维度为 hidden_dim，输出维度为 output_dim，不使用偏置
        self.fc2 = tnn.Linear(hidden_dim, output_dim, bias=False)

    # 前向传播方法，接受输入 x，对输入进行前向传播计算并返回结果
    def forward(self, x):
        # 将输入 x 传入第一个全连接层
        x = self.fc1(x)
        # 将第一个全连接层的输出传入 ReLU 激活函数
        x = self.relu(x)
        # 将 ReLU 激活函数的输出传入第二个全连接层
        x = self.fc2(x)
        # 返回最终结果
        return x

    # 从文件中加载模型的静态方法，接受模型文件路径作为参数
    @staticmethod
    def from_file(model_file: Path):
        # 使用 torch.load 方法加载模型文件，指定在 CPU 上加载
        model = torch.load(model_file, map_location="cpu")
        # 获取隐藏层和输入层的维度
        hidden_size, input_size = model.get("fc1.weight").shape
        # 获取输出层和隐藏层的维度
        output_size, _ = model.get("fc2.weight").shape
# 创建一个具有输入大小、隐藏层大小和输出大小的多层感知器神经网络
mlp = ReluMLP(input_size, hidden_size, output_size)
# 加载预训练模型的状态字典
mlp.load_state_dict(model)
# 返回加载的多层感知器神经网络模型
return mlp

# 模型类的抽象基类，用于模型转换
class Model(ABC):
    def __init__(
        self,
        dir_model: Path,
        dir_mlp_pred: Path,
        ftype: int,
        fname_out: Path,
        is_big_endian: bool,
    ):
        # 模型目录
        self.dir_model = dir_model
        # 多层感知器预测目录
        self.dir_mlp_pred = dir_mlp_pred
        # 文件类型
        self.ftype = ftype
        # 输出文件名
        self.fname_out = fname_out
# 设置是否为大端序
self.is_big_endian = is_big_endian
# 根据大端序设置字节顺序
self.endianess = (
    gguf.GGUFEndian.BIG if is_big_endian else gguf.GGUFEndian.LITTLE
)
# 判断模型是否为安全张量
self.is_safetensors = self._is_model_safetensors()
# 统计模型部件数量
self.num_parts = Model.count_model_parts(
    self.dir_model, ".safetensors" if self.is_safetensors else ".bin"
)
# 获取模型部件名称
self.part_names = self._get_part_names()
# 加载模型超参数
self.hparams = Model.load_hparams(self.dir_model)
# 获取模型架构
self.model_arch = self._get_model_architecture()
# 创建 GGUFWriter 对象
self.gguf_writer = gguf.GGUFWriter(
    fname_out, gguf.MODEL_ARCH_NAMES[self.model_arch], endianess=self.endianess, use_temp_file = False
)

# 设置词汇表
def set_vocab(self):
    self._set_vocab_gpt2()

# 获取张量
def get_tensors(self) -> Iterator[tuple[str, Tensor]]:
    # 遍历获取 MLP 部件层名称
    for model_layer, part_name in self._get_mlp_part_layer_names():
# 打印加载 MLP 部分的信息
print(f"gguf: loading mlp part '{part_name}'")
# 从文件中加载 ReluMLP 模型
mlp_model = ReluMLP.from_file(self.dir_mlp_pred / part_name)
# 遍历模型的参数和对应的数据
for name, data in mlp_model.state_dict().items():
    # 生成模型层和参数名的字符串，并返回参数数据
    yield f"blk.{model_layer}.{name}", data

# 遍历模型的各个部分
for part_name in self.part_names:
    # 打印加载模型部分的信息
    print(f"gguf: loading model part '{part_name}'")
    # 定义上下文管理器
    ctx: ContextManager[Any]
    # 如果使用安全张量
    if self.is_safetensors:
        # 从安全张量库中导入 safe_open 函数
        from safetensors import safe_open
        # 使用 safe_open 函数打开模型文件，并指定框架和设备
        ctx = cast(
            ContextManager[Any],
            safe_open(self.dir_model / part_name, framework="pt", device="cpu"),
        )
    # 如果不使用安全张量
    else:
        # 使用 contextlib.nullcontext 函数创建上下文管理器，加载模型文件并指定设备
        ctx = contextlib.nullcontext(
            torch.load(self.dir_model / part_name, map_location="cpu")
        )
# 使用上下文管理器打开模型部分
with ctx as model_part:
    # 遍历模型部分的键（名称）
    for name in model_part.keys():
        # 获取模型部分的张量数据，如果是安全张量则使用 get_tensor 方法，否则直接使用索引获取
        data = (
            model_part.get_tensor(name)
            if self.is_safetensors
            else model_part[name]
        )
        # 生成器，返回每个名称和对应的数据
        yield name, data

# 抽象方法，设置 GGUF 参数
@abstractmethod
def set_gguf_parameters(self):
    pass
    # 添加模型目录名称到 GGUF 写入器
    # self.gguf_writer.add_name(self.dir_model.name)
    # 添加块数量到 GGUF 写入器
    # self.gguf_writer.add_block_count(
    #     self.hparams.get(
    #         "n_layers",
    #         self.hparams.get("num_hidden_layers", self.hparams.get("n_layer")),
    #     )
    # )
    # 如果存在 max_position_embeddings 参数，则添加到 GGUF 写入器
    # if (n_ctx := self.hparams.get("max_position_embeddings")) is not None:
# 添加上下文长度到gguf_writer
# 如果hidden_size参数存在，则添加嵌入长度到gguf_writer
# 如果intermediate_size参数存在，则添加前馈长度到gguf_writer
# 如果num_attention_head参数存在，则添加注意力头数到gguf_writer
# 添加并行残差连接到gguf_writer，如果use_parallel_residual参数不存在，则默认为True

# 定义抽象方法write_tensors
# 写入张量数据
# 将头部信息写入文件
# 将键值数据写入文件
# 将张量数据写入文件
# 关闭 gguf_writer 对象
self.gguf_writer.close()

# 写入词汇表数据，包括写入文件头和键值数据，然后关闭 gguf_writer 对象
def write_vocab(self):
    self.gguf_writer.write_header_to_file()
    self.gguf_writer.write_kv_data_to_file()
    self.gguf_writer.close()

# 统计指定目录下以指定前缀开头的文件数量
@staticmethod
def count_model_parts(dir_model: Path, prefix: str) -> int:
    num_parts = 0
    for filename in os.listdir(dir_model):
        if filename.endswith(prefix):
            num_parts += 1
    return num_parts

# 加载模型超参数配置
@staticmethod
def load_hparams(dir_model):
    with open(dir_model / "config.json", "r", encoding="utf-8") as f:
        return json.load(f)
    # 从模型架构中创建模型对象
    @staticmethod
    def from_model_architecture(model_architecture):
        # 如果模型架构是"FalconForCausalLM"或"RWForCausalLM"，返回FalconModel
        if model_architecture in ("FalconForCausalLM", "RWForCausalLM"):
            return FalconModel
        # 如果模型架构是"LlamaForCausalLM"，返回LlamaModel
        if model_architecture == "LlamaForCausalLM":
            return LlamaModel

        # 如果模型架构不在支持的架构列表中，抛出NotImplementedError
        raise NotImplementedError(f'Architecture "{model_architecture}" not supported!')

    # 检查模型是否使用了safetensors
    def _is_model_safetensors(self) -> bool:
        return Model.count_model_parts(self.dir_model, ".safetensors") > 0

    # 获取MLP部分层的名称
    def _get_mlp_part_layer_names(self):
        """Returns a generator of (index, name) for MLP predictors of each model layer"""
        # 计算MLP部分的数量
        n_mlp_parts = Model.count_model_parts(self.dir_mlp_pred, ".pt")
        # 返回一个生成器，包含每个模型层的索引和名称
        return ((n, f"model_{n}.pt") for n in range(n_mlp_parts))

    # 获取部分名称
    def _get_part_names(self):
        # 如果模型使用了safetensors
        if self.is_safetensors:
# 如果只有一个 .safetensors 文件，则返回一个元组，包含文件名"model.safetensors"
# 如果有多个 .safetensors 文件，则返回一个生成器表达式，生成文件名"model-00001-of-00001.safetensors"到"model-xxxxx-of-xxxxx.safetensors"的文件名
if self.num_parts == 1:  
    return ("model.safetensors",)
return (
    f"model-{n:05}-of-{self.num_parts:05}.safetensors"
    for n in range(1, self.num_parts + 1)
)

# 如果只有一个 .bin 文件，则返回一个元组，包含文件名"pytorch_model.bin"
# 如果有多个 .bin 文件，则返回一个生成器表达式，生成文件名"pytorch_model-00001-of-00001.bin"到"pytorch_model-xxxxx-of-xxxxx.bin"的文件名
if self.num_parts == 1:  
    return ("pytorch_model.bin",)
return (
    f"pytorch_model-{n:05}-of-{self.num_parts:05}.bin"
    for n in range(1, self.num_parts + 1)
)

# 获取模型架构信息
def _get_model_architecture(self) -> gguf.MODEL_ARCH:
    # 获取模型架构列表中的第一个架构
    arch = self.hparams["architectures"][0]
    # 如果架构是"FalconForCausalLM"，则返回FALCON架构
    if arch == "FalconForCausalLM":
        return gguf.MODEL_ARCH.FALCON
    # 如果架构是"RWForCausalLM"或"LlamaForCausalLM"，则返回LLAMA架构
    if arch == "RWForCausalLM" or arch == "LlamaForCausalLM":
        return gguf.MODEL_ARCH.LLAMA
# 如果架构不受支持，则引发未实现的错误
        raise NotImplementedError(f'Architecture "{arch}" not supported!')

    # 将张量键转换为架构特定的键
    def _translate_tensor_key(
        self, key: str, try_suffixes=(".weight", ".bias")
    ) -> Optional[str]:
        # 获取模型层数
        block_count = self.hparams.get(
            "n_layers",
            self.hparams.get("num_hidden_layers", self.hparams.get("n_layer")),
        )
        # 获取张量名称映射
        tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)
        # 获取架构特定的张量键
        arch_tensor_key = tensor_map.get_name(key, try_suffixes=try_suffixes)
        if arch_tensor_key is not None:
            return arch_tensor_key
        # 检查和处理 ReluMLP 层
        mlp_match = re.match(r"^blk\.\d+\.fc\d\.weight$", key)
        if mlp_match:
            return mlp_match.group(0)
        return None
# 设置 GPT2 模型的词汇表
def _set_vocab_gpt2(self):
    # 获取模型目录和超参数
    dir_model = self.dir_model
    hparams = self.hparams
    # 初始化 tokens 和 toktypes 列表
    tokens: list[bytearray] = []
    toktypes: list[int] = []

    # 导入 AutoTokenizer 类
    from transformers import AutoTokenizer  # type: ignore[attr-defined]

    # 从预训练模型目录加载 tokenizer
    tokenizer = AutoTokenizer.from_pretrained(dir_model)
    # 获取词汇表大小，如果未指定则使用 tokenizer 的词汇表大小
    vocab_size = hparams.get("vocab_size", len(tokenizer.vocab))
    # 确保 tokenizer 的词汇表中的最大值小于指定的词汇表大小
    assert max(tokenizer.vocab.values()) < vocab_size

    # 创建反转的词汇表，将编码的词转换为 ID
    reverse_vocab = {
        id_: encoded_tok for encoded_tok, id_ in tokenizer.vocab.items()
    }
    # 获取添加的词汇表
    added_vocab = tokenizer.get_added_vocab()

    # 遍历词汇表大小
    for i in range(vocab_size):
        # 如果当前 ID 不在反转的词汇表中
        if i not in reverse_vocab:
            # 创建一个填充标记的字节数组
            pad_token = f"[PAD{i}]".encode("utf-8")
# 如果词汇表中的词在添加的词汇中，则将其添加到tokens列表中，并根据特殊性添加到toktypes列表中
                tokens.append(bytearray(pad_token))
                toktypes.append(gguf.TokenType.USER_DEFINED)
            # 如果词汇表中的词在已添加的词汇中，则将其添加到tokens列表中，并根据特殊性添加到toktypes列表中
            elif reverse_vocab[i] in added_vocab:
                tokens.append(reverse_vocab[i])
                if tokenizer.added_tokens_decoder[i].special:
                    toktypes.append(gguf.TokenType.CONTROL)
                else:
                    toktypes.append(gguf.TokenType.USER_DEFINED)
            # 如果词汇表中的词不在已添加的词汇中，则将其添加到tokens列表中，并将其类型设置为NORMAL
            else:
                tokens.append(reverse_vocab[i])
                toktypes.append(gguf.TokenType.NORMAL)

        # 将tokenizer模型添加到gguf_writer中
        self.gguf_writer.add_tokenizer_model("gpt2")
        # 将tokens列表添加到gguf_writer中
        self.gguf_writer.add_token_list(tokens)
        # 将toktypes列表添加到gguf_writer中
        self.gguf_writer.add_token_types(toktypes)

        # 从dir_model中加载特殊词汇，并将其添加到gguf_writer中
        special_vocab = gguf.SpecialVocab(dir_model, load_merges=True)
        special_vocab.add_to_gguf(self.gguf_writer)

    # 设置vocab_sentencepiece
    def _set_vocab_sentencepiece(self):
# 导入 SentencePieceProcessor 类
from sentencepiece import SentencePieceProcessor

# 设置 tokenizer_path 变量为模型文件的路径
tokenizer_path = self.dir_model / "tokenizer.model"

# 初始化空列表 tokens, scores, toktypes
tokens: list[bytes] = []
scores: list[float] = []
toktypes: list[int] = []

# 如果模型文件不存在，则打印错误信息并退出程序
if not tokenizer_path.is_file():
    print(f"Error: Missing {tokenizer_path}", file=sys.stderr)
    sys.exit(1)

# 创建 SentencePieceProcessor 对象并加载模型文件
tokenizer = SentencePieceProcessor(str(tokenizer_path))

# 获取词汇表大小
vocab_size = self.hparams.get("vocab_size", tokenizer.vocab_size())

# 遍历词汇表中的 token_id
for token_id in range(vocab_size):
    # 根据 token_id 获取对应的词片段
    piece = tokenizer.id_to_piece(token_id)
    # 将词片段编码为 utf-8 格式的字节流
    text = piece.encode("utf-8")
    # 获取词片段的分数
    score = tokenizer.get_score(token_id)
# 设置默认的 token 类型为 NORMAL
toktype = SentencePieceTokenTypes.NORMAL
# 如果 token 是未知的，则设置 token 类型为 UNKNOWN
if tokenizer.is_unknown(token_id):
    toktype = SentencePieceTokenTypes.UNKNOWN
# 如果 token 是控制字符，则设置 token 类型为 CONTROL
elif tokenizer.is_control(token_id):
    toktype = SentencePieceTokenTypes.CONTROL
# 如果 token 是未使用的，则设置 token 类型为 UNUSED
elif tokenizer.is_unused(token_id):
    toktype = SentencePieceTokenTypes.UNUSED
# 如果 token 是字节，则设置 token 类型为 BYTE
elif tokenizer.is_byte(token_id):
    toktype = SentencePieceTokenTypes.BYTE

# 将 token 添加到 tokens 列表中
tokens.append(text)
# 将分数添加到 scores 列表中
scores.append(score)
# 将 token 类型添加到 toktypes 列表中
toktypes.append(toktype)

# 检查是否存在 added_tokens.json 文件
added_tokens_file = self.dir_model / "added_tokens.json"
if added_tokens_file.is_file():
    # 如果文件存在，则打开并加载其中的 JSON 数据
    with open(added_tokens_file, "r", encoding="utf-8") as f:
        added_tokens_json = json.load(f)

        # 遍历 added_tokens_json 中的键
        for key in added_tokens_json:
# 将 key 编码为 UTF-8 格式并添加到 tokens 列表中
tokens.append(key.encode("utf-8"))
# 将 -1000.0 添加到 scores 列表中
scores.append(-1000.0)
# 将 SentencePieceTokenTypes.USER_DEFINED 添加到 toktypes 列表中
toktypes.append(SentencePieceTokenTypes.USER_DEFINED)

# 向 gguf_writer 中添加 tokenizer 模型 "llama"
self.gguf_writer.add_tokenizer_model("llama")
# 向 gguf_writer 中添加 token 列表
self.gguf_writer.add_token_list(tokens)
# 向 gguf_writer 中添加 token 分数列表
self.gguf_writer.add_token_scores(scores)
# 向 gguf_writer 中添加 token 类型列表
self.gguf_writer.add_token_types(toktypes)

# 创建一个特殊的词汇对象 special_vocab
special_vocab = gguf.SpecialVocab(self.dir_model, n_vocab=len(tokens))
# 将特殊词汇对象添加到 gguf_writer 中
special_vocab.add_to_gguf(self.gguf_writer)

# 定义 LlamaModel 类，继承自 Model 类
class LlamaModel(Model):
    # 设置词汇表
    def set_vocab(self):
        self._set_vocab_sentencepiece()

    # 设置 gguf 参数
    def set_gguf_parameters(self, params: PredictorParams):
        # 向 gguf_writer 中添加名称 "Llama"
        self.gguf_writer.add_name("Llama")
        # 向 gguf_writer 中添加上下文长度 2048，该参数不在 config.json 中
        self.gguf_writer.add_context_length(2048)
# 添加嵌入长度到gguf_writer中
self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])

# 添加块数量到gguf_writer中
self.gguf_writer.add_block_count(self.hparams["num_hidden_layers"])

# 添加前馈长度到gguf_writer中
self.gguf_writer.add_feed_forward_length(self.hparams["intermediate_size"])

# 添加绳索维度数量到gguf_writer中
self.gguf_writer.add_rope_dimension_count(
    self.hparams["hidden_size"] // self.hparams["num_attention_heads"]
)

# 添加注意力头数量到gguf_writer中
self.gguf_writer.add_head_count(self.hparams["num_attention_heads"])

# 添加键值头数量到gguf_writer中
self.gguf_writer.add_head_count_kv(self.hparams["num_key_value_heads"])

# 添加层归一化rms_eps到gguf_writer中
self.gguf_writer.add_layer_norm_rms_eps(self.hparams["rms_norm_eps"])

# 添加绳索频率基数到gguf_writer中
self.gguf_writer.add_rope_freq_base(self.hparams["rope_theta"])

# 添加文件类型到gguf_writer中
self.gguf_writer.add_file_type(self.ftype)

# 如果params中有稀疏阈值，则添加到gguf_writer中
if params.sparse_threshold is not None:
    self.gguf_writer.add_sparse_threshold(params.sparse_threshold)

# 写入张量数据
def write_tensors(self):
    for name, data_torch in self.get_tensors():
        # 如果名称以指定字符串结尾，则不需要处理
        if name.endswith(
# 遍历文件名列表
for name in (
                    ".attention.masked_bias",
                    ".attention.bias",
                    ".attention.rotary_emb.inv_freq",
                ):
                # 如果文件名符合特定条件，跳过本次循环
                continue

            # 保存原始数据类型
            old_dtype = data_torch.dtype

            # 将不支持的数据类型转换为float32
            if data_torch.dtype not in (torch.float16, torch.float32):
                data_torch = data_torch.to(torch.float32)

            # 压缩数据并转换为numpy数组
            data = data_torch.squeeze().numpy()

            # 映射张量名称
            new_name = self._translate_tensor_key(name)
            # 如果无法映射张量名称，打印错误信息并退出程序
            if new_name is None:
                print(f"Can not map tensor {name!r}")
                sys.exit()
            # 如果是 FFN Down 层的权重矩阵，需要对其进行转置以支持 PowerInfer 中的 Axpy 操作
            if "ffn_down" in new_name:
                # 将名称中的 "ffn_down" 替换为 "ffn_down_t"
                new_name = new_name.replace("ffn_down", "ffn_down_t")
                # 对数据进行转置
                data = data.T

            # 获取数据的维度
            n_dims = len(data.shape)
            # 获取数据的数据类型
            data_dtype = data.dtype

            # 如果需要 f32 类型的数据，将所有 float16 类型的数据转换为 float32
            if self.ftype == 0 and data_dtype == np.float16:
                data = data.astype(np.float32)

            # TODO: 为什么不能直接使用 float16？存储为 float32 的原因是什么？
            # 如果需要 f16 类型的数据，并且数据类型为 float16 且维度为 1，则将数据转换为 float32
            if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
                data = data.astype(np.float32)

            # 如果需要 f16 类型的数据，并且数据类型为 float32 且维度为 2，则将数据转换为 float16
            if (
# 检查条件：self.ftype 等于 1，data_dtype 等于 np.float32，name 以 ".weight" 结尾，n_dims 等于 2
if (
    self.ftype == 1
    and data_dtype == np.float32
    and name.endswith(".weight")
    and n_dims == 2
):
    # 如果条件成立，将数据类型转换为 np.float16
    data = data.astype(np.float16)

# 打印新名称、n_dims 值以及旧数据类型到新数据类型的转换信息
print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")

# 将新名称和数据添加到 gguf_writer 中
self.gguf_writer.add_tensor(new_name, data)


# FalconModel 类的方法，用于设置 gguf 参数
class FalconModel(Model):
    def set_gguf_parameters(self, params: PredictorParams):
        # 获取隐藏层的数量
        block_count = self.hparams.get("num_hidden_layers")
        # 如果 block_count 为 None，则使用旧名称 n_layer
        if block_count is None:
            block_count = self.hparams["n_layer"]  # old name

        # 获取注意力头的数量
        n_head = self.hparams.get("num_attention_heads")
        # 如果 n_head 为 None，则...
# 从模型参数中获取 n_head 的值，如果没有则使用旧的参数名
n_head = self.hparams["n_head"]  # old name

# 从模型参数中获取 num_kv_heads 的值，如果没有则使用旧的参数名 n_head_kv
n_head_kv = self.hparams.get("num_kv_heads")
if n_head_kv is None:
    n_head_kv = self.hparams.get("n_head_kv", 1)  # old name

# 添加名称到 gguf_writer
self.gguf_writer.add_name("Falcon")
# 添加上下文长度到 gguf_writer，值为 2048
self.gguf_writer.add_context_length(2048)  # not in config.json
# 添加张量数据布局到 gguf_writer，使用 "jploski" 的转换方式
self.gguf_writer.add_tensor_data_layout("jploski")  # qkv tensor transform
# 添加嵌入长度到 gguf_writer，值为模型参数中的 hidden_size
self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])
# 添加前馈长度到 gguf_writer，值为模型参数中 hidden_size 的 4 倍
self.gguf_writer.add_feed_forward_length(4 * self.hparams["hidden_size"])
# 添加块数量到 gguf_writer
self.gguf_writer.add_block_count(block_count)
# 添加头数量到 gguf_writer，值为 n_head
self.gguf_writer.add_head_count(n_head)
# 添加键值头数量到 gguf_writer，值为 n_head_kv
self.gguf_writer.add_head_count_kv(n_head_kv)
# 添加层归一化 epsilon 到 gguf_writer，值为模型参数中的 layer_norm_epsilon
self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_epsilon"])
# 添加文件类型到 gguf_writer，值为 ftype
self.gguf_writer.add_file_type(self.ftype)

# 如果模型参数中有稀疏阈值，则添加到 gguf_writer
if params.sparse_threshold is not None:
    self.gguf_writer.add_sparse_threshold(params.sparse_threshold)
# 定义一个方法用于写入张量数据
def write_tensors(self):
    # 获取注意力头的数量，如果没有指定则使用旧的参数名
    n_head = self.hparams.get("num_attention_heads")
    if n_head is None:
        n_head = self.hparams["n_head"]  # old name

    # 获取键值头的数量，如果没有指定则使用旧的参数名
    n_head_kv = self.hparams.get("num_kv_heads")
    if n_head_kv is None:
        n_head_kv = self.hparams.get("n_head_kv", 1)  # old name

    # 计算每个注意力头的维度
    head_dim = self.hparams["hidden_size"] // n_head

    # 遍历获取的张量数据
    for name, data_torch in self.get_tensors():
        # 保存原始数据类型
        old_dtype = data_torch.dtype

        # 将不支持的数据类型转换为float32
        if data_torch.dtype not in (torch.float16, torch.float32):
            data_torch = data_torch.to(torch.float32)

        # QKV张量转换
        # 原始的查询-键-值张量包含n_head_kv个"kv组"
# 如果名称中包含 "query_key_value"，则进行以下操作
# 重新排列数据，将 n_head 个查询权重，紧接着是 n_head_kv 个键权重，然后是 n_head_kv 个值权重，以连续的方式排列
# 参考链接：https://github.com/jploski/ggml/blob/falcon40b/examples/falcon/convert-hf-to-ggml.py

if "query_key_value" in name:
    # 将数据按照指定的维度重新排列
    qkv = data_torch.view(
        n_head_kv, n_head // n_head_kv + 2, head_dim, head_dim * n_head
    )
    # 提取查询权重
    q = qkv[:, :-2].reshape(n_head * head_dim, head_dim * n_head)
    # 提取键权重
    k = qkv[:, [-2]].reshape(n_head_kv * head_dim, head_dim * n_head)
    # 提取值权重
    v = qkv[:, [-1]].reshape(n_head_kv * head_dim, head_dim * n_head)
    # 将提取的权重拼接成一个新的数据张量
    data_torch = torch.cat((q, k, v)).reshape_as(data_torch)

# 将数据张量转换为 numpy 数组
data = data_torch.squeeze().numpy()

# 映射张量名称
# 根据给定的名称转换张量的键
new_name = self._translate_tensor_key(name)
# 如果转换后的名称为None，则打印错误信息并退出程序
if new_name is None:
    print(f"Can not map tensor {name!r}")
    sys.exit()

# 对于FFN Down层的权重矩阵，需要进行转置以支持PowerInfer中的Axpy操作，因此在运行时不需要再进行转置
if "ffn_down" in new_name:
    new_name = new_name.replace("ffn_down", "ffn_down_t")
    data = data.T

# 获取数据的维度和数据类型
n_dims = len(data.shape)
data_dtype = data.dtype

# 如果需要的数据类型是f32，并且数据类型为float16，则将数据类型转换为float32
if self.ftype == 0 and data_dtype == np.float16:
    data = data.astype(np.float32)

# TODO: 为什么不能直接使用这些float16？没有理由将float16存储为float32
if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
# 将数据类型转换为 np.float32
data = data.astype(np.float32)

# 如果需要转换为 np.float16，并且原数据类型为 np.float32，并且变量名以".weight"结尾，并且数据维度为2，则将数据类型转换为 np.float16
if (
    self.ftype == 1
    and data_dtype == np.float32
    and name.endswith(".weight")
    and n_dims == 2
):
    data = data.astype(np.float16)

# 打印新变量名、数据维度和数据类型的变化
print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")

# 将新变量名和数据添加到 gguf_writer 中
self.gguf_writer.add_tensor(new_name, data)

# 定义了一个名为 PredictorParams 的数据类，其中包含了一个名为 sparse_threshold 的可选参数，默认值为 None
@dataclass
class PredictorParams:
    sparse_threshold: float | None = None
# 静态方法，从指定路径加载预测器的配置文件，并返回预测器参数对象
@staticmethod
def loadPredictorJson(config_path: Path) -> PredictorParams:
    # 从配置文件路径加载 JSON 数据
    config = json.load(open(config_path))
    # 根据加载的 JSON 数据创建预测器参数对象
    return PredictorParams(
        sparse_threshold = config.get("sparse_threshold"),
    )

# 静态方法，从模型实例加载预测器参数
@staticmethod
def load(model_instance: Model) -> PredictorParams:
    # 获取配置文件路径
    config_path   = model_instance.dir_mlp_pred  / "config.json"

    # 如果配置文件存在，则加载预测器参数
    if config_path.exists():
        params = PredictorParams.loadPredictorJson(config_path)
    # 如果配置文件不存在，则创建一个空的预测器参数对象
    else:
        params = PredictorParams()

    # 返回加载或创建的预测器参数对象
    return params

###### CONVERSION LOGIC ######
# 解析命令行参数，返回参数的命名空间
def parse_args() -> argparse.Namespace:
    # 创建参数解析器，设置程序描述
    parser = argparse.ArgumentParser(
        description="Convert a huggingface model to a GGML compatible file"
    )
    # 添加参数：是否只提取词汇表
    parser.add_argument(
        "--vocab-only",
        action="store_true",
        help="extract only the vocab",
    )
    # 添加参数：输出文件路径
    parser.add_argument(
        "--outfile",
        type=Path,
        help="path to write to; default: based on input",
    )
    # 添加参数：输出类型，可选值为"f32"和"f16"
    parser.add_argument(
        "--outtype",
        type=str,
        choices=["f32", "f16"],
        # ...
    )
# 设置输出格式的默认值为f16，可以使用f32来表示float32，f16来表示float16
default="f16",
help="output format - use f32 for float32, f16 for float16",

# 添加一个参数，表示模型在大端机器上执行
parser.add_argument(
    "--bigendian",
    action="store_true",
    help="model is executed on big endian machine",
)

# 添加一个参数，表示模型文件所在的目录
parser.add_argument(
    "model",
    type=Path,
    help="directory containing model file",
)

# 添加一个参数，表示MLP预测器所在的目录
parser.add_argument(
    "mlp_predictors",
    type=Path,
    help="directory containing MLP predictors for model",
)

# 解析命令行参数并返回
return parser.parse_args()
# 解析命令行参数
args = parse_args()

# 获取模型目录和MLP预测器目录
dir_model = args.model
dir_mlp_pred = args.mlp_predictors

# 如果模型目录不是一个目录，则打印错误信息并退出程序
if not dir_model.is_dir():
    print(f"Error: {args.model} is not a directory", file=sys.stderr)
    sys.exit(1)

# 如果MLP预测器目录不是一个目录，则打印错误信息并退出程序
if not dir_mlp_pred.is_dir():
    print(f"Error: {args.mlp_predictors} is not a directory", file=sys.stderr)
    sys.exit(1)

# 定义文件类型映射关系
ftype_map = {
    "f32": gguf.GGMLQuantizationType.F32,
    "f16": gguf.GGMLQuantizationType.F16,
}

# 如果指定了输出文件，则将其赋值给fname_out变量
if args.outfile is not None:
    fname_out = args.outfile
# 如果条件不满足，则在与模型相同的目录中输出默认文件名
else:
    fname_out = dir_model / f"ggml-model-{args.outtype}.gguf"

# 打印加载模型的信息
print(f"Loading model: {dir_model.name}")

# 从模型目录加载超参数
hparams = Model.load_hparams(dir_model)

# 从模型架构创建模型类
model_class = Model.from_model_architecture(hparams["architectures"][0])

# 创建模型实例
model_instance = model_class(
    dir_model, dir_mlp_pred, ftype_map[args.outtype], fname_out, args.bigendian
)

# 设置模型参数
print("Set model parameters")
params = PredictorParams.load(model_instance)
model_instance.set_gguf_parameters(params)

# 设置模型分词器
print("Set model tokenizer")
model_instance.set_vocab()
# 如果参数中指定了只导出词汇表，则打印导出模型词汇表的信息，并调用模型实例的写词汇表方法
if args.vocab_only:
    print(f"Exporting model vocab to '{fname_out}'")
    model_instance.write_vocab()
# 如果没有指定只导出词汇表，则打印导出模型的信息，并调用模型实例的写方法
else:
    print(f"Exporting model to '{fname_out}'")
    model_instance.write()

# 后处理：向导出的文件中写入另一个唯一的文件头，以区别于原始的 GGUF 文件
# 以读写方式打开导出的文件
with open(fname_out, "r+b") as fout:
    # 定义一个魔术数，使用小端字节序转换为整数
    POWERINFER_MAGIC = int.from_bytes(b"PWRI", "little")
    # 向文件中写入使用小端字节序编码的魔术数
    fout.write(struct.pack("<I", POWERINFER_MAGIC))

# 打印成功导出模型的信息
print(f"Model successfully exported to '{fname_out}'")
```