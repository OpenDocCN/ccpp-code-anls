# `PowerInfer\convert-hf-to-gguf.py`

```
#!/usr/bin/env python3
# 指定使用 Python3 解释器

from __future__ import annotations
# 导入未来版本的特性，用于支持类型注解

import argparse
# 导入用于解析命令行参数的模块
import contextlib
# 导入用于创建上下文管理器的模块
import json
# 导入用于处理 JSON 数据的模块
import os
# 导入用于与操作系统交互的模块
import re
# 导入用于处理正则表达式的模块
import sys
# 导入用于与 Python 解释器交互的模块
from enum import IntEnum
# 从枚举模块中导入 IntEnum 类
from pathlib import Path
# 从路径模块中导入 Path 类
from typing import TYPE_CHECKING, Any, ContextManager, Iterator, cast
# 导入类型提示相关的模块和类

import numpy as np
# 导入 NumPy 库
import torch
# 导入 PyTorch 库

if TYPE_CHECKING:
    from torch import Tensor
# 如果是类型检查阶段，则从 PyTorch 中导入 Tensor 类型
# 检查环境变量中是否存在'NO_LOCAL_GGUF'，如果不存在则将'gguf-py'路径插入到系统路径中
if 'NO_LOCAL_GGUF' not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))
# 导入gguf模块

###### 模型定义 ######

# 定义一个枚举类，包含了句子片段的不同类型
class SentencePieceTokenTypes(IntEnum):
    NORMAL = 1
    UNKNOWN = 2
    CONTROL = 3
    USER_DEFINED = 4
    UNUSED = 5
    BYTE = 6

# 定义一个模型类
class Model:
    # 初始化方法，接受模型目录、文件类型、输出文件名和大小端标志作为参数
    def __init__(self, dir_model: Path, ftype: int, fname_out: Path, is_big_endian: bool):
        # 将参数赋值给实例变量
        self.dir_model = dir_model
        self.ftype = ftype
# 设置输出文件名
self.fname_out = fname_out
# 设置是否为大端序
self.is_big_endian = is_big_endian
# 根据大端序设置相应的字节序
self.endianess = gguf.GGUFEndian.BIG if is_big_endian else gguf.GGUFEndian.LITTLE
# 判断模型是否为安全张量模型
self.is_safetensors = self._is_model_safetensors()
# 统计模型部分的数量
self.num_parts = Model.count_model_parts(self.dir_model, ".safetensors" if self.is_safetensors else ".bin")
# 获取模型部分的名称
self.part_names = self._get_part_names()
# 加载模型的超参数
self.hparams = Model.load_hparams(self.dir_model)
# 获取模型的架构
self.model_arch = self._get_model_architecture()
# 创建 GGUFWriter 对象
self.gguf_writer = gguf.GGUFWriter(fname_out, gguf.MODEL_ARCH_NAMES[self.model_arch], endianess=self.endianess)

# 设置词汇表
def set_vocab(self):
    self._set_vocab_gpt2()

# 获取张量
def get_tensors(self) -> Iterator[tuple[str, Tensor]]:
    # 遍历模型部分的名称
    for part_name in self.part_names:
        # 打印加载模型部分的信息
        print(f"gguf: loading model part '{part_name}'")
        # 定义上下文管理器
        ctx: ContextManager[Any]
        # 如果是安全张量模型，则使用 safe_open 方法打开模型部分
        if self.is_safetensors:
            from safetensors import safe_open
            ctx = cast(ContextManager[Any], safe_open(self.dir_model / part_name, framework="pt", device="cpu"))
            else:
                # 如果不是安全张量，则使用torch.load加载模型部分，并指定在CPU上加载
                ctx = contextlib.nullcontext(torch.load(self.dir_model / part_name, map_location="cpu"))

            # 使用上下文管理器加载模型部分
            with ctx as model_part:
                # 遍历模型部分的所有键
                for name in model_part.keys():
                    # 如果是安全张量，则使用get_tensor方法获取数据，否则直接通过键获取数据
                    data = model_part.get_tensor(name) if self.is_safetensors else model_part[name]
                    # 生成器返回键和数据
                    yield name, data

    # 设置GGUF参数
    def set_gguf_parameters(self):
        # 添加模型目录的名称到GGUF写入器
        self.gguf_writer.add_name(self.dir_model.name)
        # 添加块的数量到GGUF写入器
        self.gguf_writer.add_block_count(self.hparams.get(
            "n_layers", self.hparams.get("num_hidden_layers", self.hparams.get("n_layer")),
        ))
        # 如果存在最大位置嵌入长度，则添加到GGUF写入器
        if (n_ctx := self.hparams.get("max_position_embeddings")) is not None:
            self.gguf_writer.add_context_length(n_ctx)
        # 如果存在隐藏层大小，则添加到GGUF写入器
        if (n_embd := self.hparams.get("hidden_size")) is not None:
            self.gguf_writer.add_embedding_length(n_embd)
        # 如果存在中间层大小，则添加到GGUF写入器
        if (n_ff := self.hparams.get("intermediate_size")) is not None:
            self.gguf_writer.add_feed_forward_length(n_ff)
        # 如果存在注意力头数量，则添加到GGUF写入器
        if (n_head := self.hparams.get("num_attention_head")) is not None:
# 添加头部计数
self.gguf_writer.add_head_count(n_head)
# 添加并行残差连接
self.gguf_writer.add_parallel_residual(self.hparams.get("use_parallel_residual", True))

# 写入张量
# 获取块数量
block_count = self.hparams.get("n_layers", self.hparams.get("num_hidden_layers", self.hparams.get("n_layer")))
# 获取张量名称映射
tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)
# 遍历获取的张量
for name, data_torch in self.get_tensors():
    # 如果张量名称以指定后缀结尾，则跳过
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
# 根据给定的名称和后缀尝试获取新的名称
new_name = tensor_map.get_name(name, try_suffixes=(".weight", ".bias"))
# 如果获取不到新的名称，打印错误信息并退出程序
if new_name is None:
    print(f"Can not map tensor {name!r}")
    sys.exit()

# 获取数据的维度
n_dims = len(data.shape)
# 获取数据的数据类型
data_dtype = data.dtype

# 如果期望的数据类型是 f32，并且数据类型是 float16，则将数据类型转换为 float32
if self.ftype == 0 and data_dtype == np.float16:
    data = data.astype(np.float32)

# TODO: 为什么不能直接使用这些 float16？应该没有理由将 float16 存储为 float32
# 如果期望的数据类型是 f16，并且数据类型是 float16，并且数据维度是 1，则将数据类型转换为 float32
if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
    data = data.astype(np.float32)

# 如果期望的数据类型是 f16，并且数据类型是 float32，并且名称以 ".weight" 结尾，并且数据维度是 2，则将数据类型转换为 float16
if self.ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
    data = data.astype(np.float16)
            # 打印新名称、维度数和数据类型转换前后的信息
            print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")

            # 将数据添加到 GGUF 写入器中
            self.gguf_writer.add_tensor(new_name, data)

    # 写入所有数据
    def write(self):
        self.write_tensors()
        self.gguf_writer.write_header_to_file()
        self.gguf_writer.write_kv_data_to_file()
        self.gguf_writer.write_tensors_to_file()
        self.gguf_writer.close()

    # 写入词汇表数据
    def write_vocab(self):
        self.gguf_writer.write_header_to_file()
        self.gguf_writer.write_kv_data_to_file()
        self.gguf_writer.close()

    # 统计模型部件的数量
    @staticmethod
    def count_model_parts(dir_model: Path, prefix: str) -> int:
        num_parts = 0
        # 遍历模型目录下的文件
        for filename in os.listdir(dir_model):
    # 如果文件名以指定前缀结尾，则计数器加一
    if filename.endswith(prefix):
        num_parts += 1
    # 返回计数器的值
    return num_parts

    # 从指定模型目录加载超参数
    @staticmethod
    def load_hparams(dir_model):
        # 打开指定模型目录下的config.json文件，以utf-8编码读取内容，并返回解析后的JSON对象
        with open(dir_model / "config.json", "r", encoding="utf-8") as f:
            return json.load(f)

    # 根据模型架构名称返回对应的模型类
    @staticmethod
    def from_model_architecture(model_architecture):
        # 如果模型架构为"GPTNeoXForCausalLM"，则返回GPTNeoXModel类
        if model_architecture == "GPTNeoXForCausalLM":
            return GPTNeoXModel
        # 如果模型架构为"BloomForCausalLM"，则返回BloomModel类
        if model_architecture == "BloomForCausalLM":
            return BloomModel
        # 如果模型架构为"MPTForCausalLM"，则返回MPTModel类
        if model_architecture == "MPTForCausalLM":
            return MPTModel
        # 如果模型架构为"BaichuanForCausalLM"或"BaiChuanForCausalLM"，则返回BaichuanModel类
        if model_architecture in ("BaichuanForCausalLM", "BaiChuanForCausalLM"):
            return BaichuanModel
        # 如果模型架构是"FalconForCausalLM"或"RWForCausalLM"，则返回FalconModel
        if model_architecture in ("FalconForCausalLM", "RWForCausalLM"):
            return FalconModel
        # 如果模型架构是"GPTBigCodeForCausalLM"，则返回StarCoderModel
        if model_architecture == "GPTBigCodeForCausalLM":
            return StarCoderModel
        # 如果模型架构是"GPTRefactForCausalLM"，则返回RefactModel
        if model_architecture == "GPTRefactForCausalLM":
            return RefactModel
        # 如果模型架构是"PersimmonForCausalLM"，则返回PersimmonModel
        if model_architecture == "PersimmonForCausalLM":
            return PersimmonModel
        # 如果模型架构是"StableLMEpochForCausalLM"或"LlavaStableLMEpochForCausalLM"，则返回StableLMModel
        if model_architecture in ("StableLMEpochForCausalLM", "LlavaStableLMEpochForCausalLM"):
            return StableLMModel
        # 如果以上条件都不满足，则返回Model
        return Model

    # 检查模型是否有.safetensors文件
    def _is_model_safetensors(self) -> bool:
        return Model.count_model_parts(self.dir_model, ".safetensors") > 0

    # 获取模型部件的名称
    def _get_part_names(self):
        # 如果模型有.safetensors文件
        if self.is_safetensors:
            # 如果模型部件只有一个.safetensors文件
            if self.num_parts == 1:
                return ("model.safetensors",)
            # 如果模型部件有多个.safetensors文件
            return (f"model-{n:05}-of-{self.num_parts:05}.safetensors" for n in range(1, self.num_parts + 1))
        # 如果只有一个 .bin 文件，则返回一个元组，元组中只包含一个元素 "pytorch_model.bin"
        if self.num_parts == 1:  
            return ("pytorch_model.bin",)
        # 如果有多个 .bin 文件，则返回一个生成器，生成器会生成多个文件名，格式为 "pytorch_model-00001-of-00005.bin" 等
        return (f"pytorch_model-{n:05}-of-{self.num_parts:05}.bin" for n in range(1, self.num_parts + 1))

    # 获取模型架构
    def _get_model_architecture(self) -> gguf.MODEL_ARCH:
        # 获取模型的架构
        arch = self.hparams["architectures"][0]
        # 根据不同的架构名称返回对应的枚举值
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
            # 如果架构名称为 "GPTRefactForCausalLM"，则返回对应的枚举值
# 根据模型架构名称返回对应的架构类型
def get_model_arch(arch):
    # 如果架构是 "GPT2LMHeadModel"，返回对应的架构类型
    if arch == "GPT2LMHeadModel":
        return gguf.MODEL_ARCH.GPT2
    # 如果架构是 "RefactForCausalLM"，返回对应的架构类型
    if arch == "RefactForCausalLM":
        return gguf.MODEL_ARCH.REFACT
    # 如果架构是 "PersimmonForCausalLM"，返回对应的架构类型
    if arch == "PersimmonForCausalLM":
        return gguf.MODEL_ARCH.PERSIMMON
    # 如果架构是 "StableLMEpochForCausalLM" 或 "LlavaStableLMEpochForCausalLM"，返回对应的架构类型
    if arch in ("StableLMEpochForCausalLM", "LlavaStableLMEpochForCausalLM"):
        return gguf.MODEL_ARCH.STABLELM

    # 如果架构不在以上列出的架构类型中，抛出 NotImplementedError 异常
    raise NotImplementedError(f'Architecture "{arch}" not supported!')

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
    # 根据模型目录创建 tokenizer 对象
    tokenizer = AutoTokenizer.from_pretrained(dir_model)
    # 获取词汇表大小，如果超参数中有指定则使用超参数中的值，否则使用 tokenizer.vocab 的大小
    vocab_size = hparams.get("vocab_size", len(tokenizer.vocab))
    # 断言 tokenizer.vocab 中的最大值小于词汇表大小
    assert max(tokenizer.vocab.values()) < vocab_size
    # 创建反向词汇表，将编码后的词汇映射到对应的 id
    reverse_vocab = {id_: encoded_tok for encoded_tok, id_ in tokenizer.vocab.items()}
# 获取tokenizer中添加的词汇表
added_vocab = tokenizer.get_added_vocab()

# 遍历vocab_size范围内的值
for i in range(vocab_size):
    # 如果i不在reverse_vocab中
    if i not in reverse_vocab:
        # 创建一个包含i的[PAD{i}]的字节流，并添加到tokens列表中
        pad_token = f"[PAD{i}]".encode('utf-8')
        tokens.append(bytearray(pad_token))
        # 将USER_DEFINED类型添加到toktypes列表中
        toktypes.append(gguf.TokenType.USER_DEFINED)
    # 如果reverse_vocab[i]在added_vocab中
    elif reverse_vocab[i] in added_vocab:
        # 将reverse_vocab[i]添加到tokens列表中
        tokens.append(reverse_vocab[i])
        # 如果tokenizer.added_tokens_decoder[i].special为True，将CONTROL类型添加到toktypes列表中，否则添加USER_DEFINED类型
        if tokenizer.added_tokens_decoder[i].special:
            toktypes.append(gguf.TokenType.CONTROL)
        else:
            toktypes.append(gguf.TokenType.USER_DEFINED)
    # 如果reverse_vocab[i]不在added_vocab中
    else:
        # 将reverse_vocab[i]添加到tokens列表中
        tokens.append(reverse_vocab[i])
        # 将NORMAL类型添加到toktypes列表中
        toktypes.append(gguf.TokenType.NORMAL)

# 将"gpt2"模型添加到gguf_writer中
self.gguf_writer.add_tokenizer_model("gpt2")
# 将tokens列表添加到gguf_writer中
self.gguf_writer.add_token_list(tokens)
# 将toktypes列表添加到gguf_writer中
self.gguf_writer.add_token_types(toktypes)
# 创建一个特殊词汇对象，从指定目录加载合并的词汇
special_vocab = gguf.SpecialVocab(dir_model, load_merges=True)
# 将特殊词汇添加到 gguf_writer 中
special_vocab.add_to_gguf(self.gguf_writer)

# 导入 SentencePieceProcessor 类
from sentencepiece import SentencePieceProcessor

# 设置 tokenizer_path 变量为模型目录下的 tokenizer.model 文件路径
tokenizer_path = self.dir_model / 'tokenizer.model'

# 初始化空列表 tokens 用于存储字节串
tokens: list[bytes] = []
# 初始化空列表 scores 用于存储浮点数
scores: list[float] = []
# 初始化空列表 toktypes 用于存储整数
toktypes: list[int] = []

# 如果 tokenizer_path 文件不存在
if not tokenizer_path.is_file():
    # 打印错误信息并退出程序
    print(f'Error: Missing {tokenizer_path}', file=sys.stderr)
    sys.exit(1)

# 创建一个 SentencePieceProcessor 对象，加载 tokenizer.model 文件
tokenizer = SentencePieceProcessor(str(tokenizer_path))
# 获取 vocab_size 参数，如果没有指定则使用 tokenizer 的词汇大小
vocab_size = self.hparams.get('vocab_size', tokenizer.vocab_size())
# 遍历词汇表中的每个 token ID
for token_id in range(vocab_size):
    # 根据 token ID 获取对应的 token
    piece = tokenizer.id_to_piece(token_id)
    # 将 token 转换为 UTF-8 编码的字节流
    text = piece.encode("utf-8")
    # 获取 token 的分数
    score = tokenizer.get_score(token_id)

    # 初始化 token 类型为普通类型
    toktype = SentencePieceTokenTypes.NORMAL
    # 根据 token ID 判断其类型，并赋值给 toktype
    if tokenizer.is_unknown(token_id):
        toktype = SentencePieceTokenTypes.UNKNOWN
    elif tokenizer.is_control(token_id):
        toktype = SentencePieceTokenTypes.CONTROL
    elif tokenizer.is_unused(token_id):
        toktype = SentencePieceTokenTypes.UNUSED
    elif tokenizer.is_byte(token_id):
        toktype = SentencePieceTokenTypes.BYTE

    # 将处理后的 token、分数和类型添加到对应的列表中
    tokens.append(text)
    scores.append(score)
    toktypes.append(toktype)

# 设置添加的 tokens 文件路径
added_tokens_file = self.dir_model / 'added_tokens.json'
# 检查是否存在 added_tokens_file 文件
if added_tokens_file.is_file():
    # 如果文件存在，以 utf-8 编码方式打开文件
    with open(added_tokens_file, "r", encoding="utf-8") as f:
        # 从文件中加载 JSON 数据
        added_tokens_json = json.load(f)

        # 遍历 JSON 数据中的键
        for key in added_tokens_json:
            # 将键编码为 utf-8 格式并添加到 tokens 列表中
            tokens.append(key.encode("utf-8"))
            # 添加一个固定的分数到 scores 列表中
            scores.append(-1000.0)
            # 添加用户定义的类型到 toktypes 列表中
            toktypes.append(SentencePieceTokenTypes.USER_DEFINED)

# 向 gguf_writer 添加 tokenizer 模型
self.gguf_writer.add_tokenizer_model("llama")
# 向 gguf_writer 添加 token 列表
self.gguf_writer.add_token_list(tokens)
# 向 gguf_writer 添加 token 分数
self.gguf_writer.add_token_scores(scores)
# 向 gguf_writer 添加 token 类型
self.gguf_writer.add_token_types(toktypes)

# 创建一个特殊的词汇对象
special_vocab = gguf.SpecialVocab(self.dir_model, n_vocab=len(tokens))
# 将特殊词汇添加到 gguf_writer 中
special_vocab.add_to_gguf(self.gguf_writer)

# 定义 GPTNeoXModel 类的 set_gguf_parameters 方法
class GPTNeoXModel(Model):
    def set_gguf_parameters(self):
# 设置块的数量为模型超参数中的隐藏层数量
block_count = self.hparams["num_hidden_layers"]

# 添加模型名称到GGUF写入器
self.gguf_writer.add_name(self.dir_model.name)

# 添加上下文长度到GGUF写入器
self.gguf_writer.add_context_length(self.hparams["max_position_embeddings"])

# 添加嵌入长度到GGUF写入器
self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])

# 添加块数量到GGUF写入器
self.gguf_writer.add_block_count(block_count)

# 添加前馈网络长度到GGUF写入器
self.gguf_writer.add_feed_forward_length(self.hparams["intermediate_size"])

# 添加绳索维度数量到GGUF写入器
self.gguf_writer.add_rope_dimension_count(
    int(self.hparams["rotary_pct"] * (self.hparams["hidden_size"] // self.hparams["num_attention_heads"])),
)

# 添加注意力头数量到GGUF写入器
self.gguf_writer.add_head_count(self.hparams["num_attention_heads"])

# 添加并行残差到GGUF写入器
self.gguf_writer.add_parallel_residual(self.hparams.get("use_parallel_residual", True))

# 添加层归一化epsilon到GGUF写入器
self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_eps"])

# 设置GGUF参数为Bloom模型
def set_gguf_parameters(self):
    self.gguf_writer.add_name("Bloom")
    n_embed = self.hparams.get("hidden_size", self.hparams.get("n_embed"))
    n_head = self.hparams.get("n_head", self.hparams.get("num_attention_heads"))
# 设置上下文长度
self.gguf_writer.add_context_length(self.hparams.get("seq_length", n_embed))
# 设置嵌入长度
self.gguf_writer.add_embedding_length(n_embed)
# 设置前馈长度
self.gguf_writer.add_feed_forward_length(4 * n_embed)
# 设置块数量
self.gguf_writer.add_block_count(self.hparams["n_layer"])
# 设置头数量
self.gguf_writer.add_head_count(n_head)
# 设置键值头数量
self.gguf_writer.add_head_count_kv(n_head)
# 设置层归一化 epsilon
self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_epsilon"])
# 设置文件类型
self.gguf_writer.add_file_type(self.ftype)

# 写入张量
block_count = self.hparams["n_layer"]
# 获取张量
tensors = dict(self.get_tensors())
# 获取张量名称映射
tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)
# 是否有语言模型头
has_lm_head = True
# 获取头数量
n_head = self.hparams.get("n_head", self.hparams.get("num_attention_heads"))
# 获取嵌入长度
n_embed = self.hparams.get("hidden_size", self.hparams.get("n_embed"))

# 遍历张量
for name, data_torch in tensors.items():
    # 如果张量中不存在语言模型头权重和输出权重，则将 has_lm_head 设置为 False
    if "lm_head.weight" not in tensors.keys() and "output.weight" not in tensors.keys():
        has_lm_head = False
# 使用正则表达式将字符串中的'transformer.'替换为空字符串
name = re.sub(r'transformer\.', '', name)

# 保存原始数据类型
old_dtype = data_torch.dtype

# 如果数据类型不是torch.float16或torch.float32，则将数据类型转换为torch.float32
if data_torch.dtype not in (torch.float16, torch.float32):
    data_torch = data_torch.to(torch.float32)

# 压缩数据并转换为numpy数组
data = data_torch.squeeze().numpy()

# 如果name符合指定的正则表达式，则进行以下操作
if re.match(r"h\.\d+\.self_attention\.query_key_value\.weight", name):
    # 将数据重塑为指定形状
    qkv_weights = data.reshape((n_head, 3, n_embed // n_head, n_embed))
    # 将数据连接起来
    data = np.concatenate(
        (
            qkv_weights[:, 0, :, :].reshape((-1, n_embed)),
            qkv_weights[:, 1, :, :].reshape((-1, n_embed)),
                qkv_weights[:, 2, :, :].reshape((-1, n_embed)),
            ),
            axis=0,
        )
        # 重新格式化注意力机制中的权重
        print("re-format attention.linear_qkv.weight")
    elif re.match(r"h\.\d+\.self_attention\.query_key_value\.bias", name):
        # 重新格式化注意力机制中的偏置
        qkv_bias = data.reshape((n_head, 3, n_embed // n_head))
        data = np.concatenate(
            (
                qkv_bias[:, 0, :].reshape((n_embed,)),
                qkv_bias[:, 1, :].reshape((n_embed,)),
                qkv_bias[:, 2, :].reshape((n_embed,)),
            ),
            axis=0,
        )
        print("re-format attention.linear_qkv.bias")

    # 映射张量名称
    new_name = tensor_map.get_name(name, try_suffixes=(".weight", ".bias"))
    if new_name is None:
# 打印无法映射的张量名称，并退出程序
print(f"Can not map tensor {name!r}")
sys.exit()

# 获取数据的维度和数据类型
n_dims = len(data.shape)
data_dtype = data.dtype

# 如果需要的数据类型是 f32，并且数据类型是 float16，则将数据类型转换为 float32
if self.ftype == 0 and data_dtype == np.float16:
    data = data.astype(np.float32)

# TODO: 为什么不能直接使用这些 float16？应该没有理由将 float16 存储为 float32
if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
    data = data.astype(np.float32)

# 如果需要的数据类型是 f16，并且数据类型是 float32，并且张量名称以 ".weight" 结尾，并且维度为 2，则将数据类型转换为 float16
if self.ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
    data = data.astype(np.float16)

# 打印转换后的张量名称、形状和数据类型
print(f"=> {new_name}, shape = {data.shape}, {old_dtype} --> {data.dtype}")
# 将张量添加到 GGUF 写入器中
self.gguf_writer.add_tensor(new_name, data)

# 如果模型没有语言模型头，并且名称为 "word_embeddings.weight"，则将张量添加到 GGUF 写入器中，并打印相关信息
if not has_lm_head and name == "word_embeddings.weight":
    self.gguf_writer.add_tensor("output.weight", data)
    print(name, f"=> output.weight, shape = {data.shape}, {old_dtype} --> {data.dtype}")

# 设置 GGUF 参数
class MPTModel(Model):
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
    # 添加头数量到 GGUF 写入器
    self.gguf_writer.add_head_count(self.hparams["n_heads"])
    # 如果存在 kv_n_heads，则添加 kv 头数量到 GGUF 写入器
    if kv_n_heads := self.hparams["attn_config"].get("kv_n_heads"):
        self.gguf_writer.add_head_count_kv(kv_n_heads)
    # 添加层归一化 epsilon 到 GGUF 写入器
    self.gguf_writer.add_layer_norm_eps(1e-5)
    # 如果存在注意力配置的 clip_qkv，则添加到 GGUF 写入器
    if self.hparams["attn_config"]["clip_qkv"] is not None:
# 添加 clamp_kqv 参数到 gguf_writer 中
self.gguf_writer.add_clamp_kqv(self.hparams["attn_config"]["clip_qkv"])
# 添加 max_alibi_bias 参数到 gguf_writer 中
self.gguf_writer.add_max_alibi_bias(self.hparams["attn_config"]["alibi_bias_max"])

# 写入张量数据
block_count = self.hparams.get("n_layers", self.hparams.get("num_hidden_layers"))
# 获取模型架构和块数量的张量名称映射
tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)
# 遍历获取的张量数据
for name, data_torch in self.get_tensors():
    # 如果张量名称以指定后缀结尾，则跳过
    if name.endswith((".attention.masked_bias", ".attention.bias", ".attention.rotary_emb.inv_freq")):
        continue

    # 保存原始数据类型
    old_dtype = data_torch.dtype

    # 将不支持的数据类型转换为 float32
    if data_torch.dtype not in (torch.float16, torch.float32):
        data_torch = data_torch.to(torch.float32)

    # 将张量数据转换为 numpy 数组
    data = data_torch.squeeze().numpy()

    # 映射张量名称
# 根据给定的名称和后缀尝试获取新的名称
new_name = tensor_map.get_name(name, try_suffixes=(".weight", ".bias"))
# 如果获取不到新的名称，则打印错误信息并退出程序
if new_name is None:
    print(f"Can not map tensor {name!r}")
    sys.exit()

# 获取数据的维度
n_dims = len(data.shape)
# 获取数据的数据类型
data_dtype = data.dtype

# 如果期望的数据类型是 float32，并且数据的数据类型是 float16，则将数据转换为 float32
if self.ftype == 0 and data_dtype == np.float16:
    data = data.astype(np.float32)

# TODO: 为什么不能直接使用 float16？存储为 float32 的原因是什么？
# 如果期望的数据类型是 float16，并且数据的数据类型是 float16，并且数据的维度是1，则将数据转换为 float32
if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
    data = data.astype(np.float32)

# 如果期望的数据类型是 float16，并且数据的数据类型是 float32，并且名称以 ".weight" 结尾，并且数据的维度是2，则将数据转换为 float16
if self.ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
    data = data.astype(np.float16)
# 打印新名称、维度数和数据类型转换信息
print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")

# 将数据添加到 GGUF 写入器中
self.gguf_writer.add_tensor(new_name, data)

# 注意：MPT 输出与原始模型中的 wte 相关联；
# 为了在 llama.cpp 中更容易实现，它在 GGUF 中被复制了 :/
if new_name == "token_embd.weight":
    # 如果新名称是 "token_embd.weight"，则将数据添加到 GGUF 写入器中的 "output.weight"
    self.gguf_writer.add_tensor("output.weight", data)

# BaichuanModel 类的方法，设置词汇表
def set_vocab(self):
    self._set_vocab_sentencepiece()

# BaichuanModel 类的方法，设置 GGUF 参数
def set_gguf_parameters(self):
    # 获取隐藏层的数量
    block_count = self.hparams["num_hidden_layers"]
    # 获取注意力头的数量
    head_count = self.hparams["num_attention_heads"]
    # 获取键值头的数量，如果未指定则与注意力头数量相同
    head_count_kv = self.hparams.get("num_key_value_heads", head_count)
    # 获取 hf_repo 参数，如果未指定则为空字符串
    hf_repo = self.hparams.get("_name_or_path", "")
# 初始化上下文长度为0
ctx_length = 0
# 如果参数中包含最大序列长度，则将上下文长度设置为最大序列长度
if "max_sequence_length" in self.hparams:
    ctx_length = self.hparams["max_sequence_length"]
# 如果参数中包含最大位置嵌入长度，则将上下文长度设置为最大位置嵌入长度
elif "max_position_embeddings" in self.hparams:
    ctx_length = self.hparams["max_position_embeddings"]
# 如果参数中包含模型最大长度，则将上下文长度设置为模型最大长度
elif "model_max_length" in self.hparams:
    ctx_length = self.hparams["model_max_length"]
# 如果以上条件都不满足，则打印错误信息并退出程序
else:
    print("gguf: can not find ctx length parameter.")
    sys.exit()

# 添加模型名称到gguf_writer
self.gguf_writer.add_name(self.dir_model.name)
# 添加源hf_repo到gguf_writer
self.gguf_writer.add_source_hf_repo(hf_repo)
# 添加张量数据布局到gguf_writer
self.gguf_writer.add_tensor_data_layout("Meta AI original pth")
# 添加上下文长度到gguf_writer
self.gguf_writer.add_context_length(ctx_length)
# 添加嵌入长度到gguf_writer
self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])
# 添加块数量到gguf_writer
self.gguf_writer.add_block_count(block_count)
# 添加前馈长度到gguf_writer
self.gguf_writer.add_feed_forward_length(self.hparams["intermediate_size"])
# 添加绳索维度数量到gguf_writer
self.gguf_writer.add_rope_dimension_count(self.hparams["hidden_size"] // self.hparams["num_attention_heads"])
# 添加头数量到gguf_writer
self.gguf_writer.add_head_count(head_count)
# 将头部计数键值对添加到gguf_writer中
self.gguf_writer.add_head_count_kv(head_count_kv)

# 将层归一化的rms_eps值添加到gguf_writer中
self.gguf_writer.add_layer_norm_rms_eps(self.hparams["rms_norm_eps"])

# 如果hparams中存在rope_scaling并且包含"factor"键
if self.hparams.get("rope_scaling") is not None and "factor" in self.hparams["rope_scaling"]:
    # 如果rope_scaling中的"type"为"linear"
    if self.hparams["rope_scaling"].get("type") == "linear":
        # 将RopeScalingType.LINEAR添加到gguf_writer中
        self.gguf_writer.add_rope_scaling_type(gguf.RopeScalingType.LINEAR)
        # 将rope_scaling中的"factor"值添加到gguf_writer中
        self.gguf_writer.add_rope_scaling_factor(self.hparams["rope_scaling"]["factor"])

# 从生成器对象中收集张量
model_kv = dict(self.get_tensors())
# 获取隐藏层的数量
block_count = self.hparams["num_hidden_layers"]
# 获取注意力头的数量
head_count = self.hparams["num_attention_heads"]
# 获取张量名称映射
tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)
# 获取键值头的数量
head_count_kv = self.hparams.get("num_key_value_heads", head_count)

# 遍历隐藏层的数量
for i in range(block_count):
    # 如果存在指定的权重张量
    if (w := model_kv.get(f"model.layers.{i}.self_attn.W_pack.weight")) is not None:
        # 打印信息
        print(f"Unpacking and permuting layer {i}")
        # 将指定的权重张量添加到model_kv中
        model_kv[f"model.layers.{i}.self_attn.q_proj.weight"] = \
# 对模型参数进行处理
# 对权重进行逆变换
self._reverse_hf_permute_part(w, 0, head_count, head_count)
# 将逆变换后的权重存入模型参数字典中
model_kv[f"model.layers.{i}.self_attn.k_proj.weight"] = \
    self._reverse_hf_permute_part(w, 1, head_count, head_count_kv)
model_kv[f"model.layers.{i}.self_attn.v_proj.weight"] = \
    self._reverse_hf_part(w, 2)
# 删除指定的模型参数
del model_kv[f"model.layers.{i}.self_attn.W_pack.weight"]

# 遍历模型参数字典
for name, data_torch in model_kv.items():
    # 如果参数名以".rotary_emb.inv_freq"结尾，则跳过
    if name.endswith(".rotary_emb.inv_freq"):
        continue

    # 保存原始数据类型
    old_dtype = data_torch.dtype

    # 将不支持的数据类型转换为float32
    if data_torch.dtype not in (torch.float16, torch.float32):
        data_torch = data_torch.to(torch.float32)

    # 将数据转换为numpy数组
    data = data_torch.squeeze().numpy()
# 根据映射表映射张量名称
new_name = tensor_map.get_name(name, try_suffixes=(".weight", ".bias"))
# 如果找不到新的名称，打印错误信息并退出程序
if new_name is None:
    print(f"Can not map tensor {name!r}")
    sys.exit()

# 获取数据的维度和数据类型
n_dims = len(data.shape)
data_dtype = data.dtype

# 如果需要的数据类型是 f32，将所有 float16 类型的数据转换为 float32
if self.ftype == 0 and data_dtype == np.float16:
    data = data.astype(np.float32)

# TODO: 为什么不能直接使用 float16？存储为 float32 的原因是什么？
if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
    data = data.astype(np.float32)

# 如果需要的数据类型是 f16，将所有二维权重张量的 float32 数据转换为 float16
if self.ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
    data = data.astype(np.float16)
# 打印名称、新名称、维度数和数据类型变化信息
print(f"{name} -> {new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")
# 将数据添加到gguf_writer中
self.gguf_writer.add_tensor(new_name, data)

# 反转HF排列，将权重张量按照指定的头数和键值头数进行重排
def _reverse_hf_permute(self, weights: Tensor, n_head: int, n_kv_head: int | None = None) -> Tensor:
    # 如果指定了键值头数且头数不相等，则重新计算头数
    if n_kv_head is not None and n_head != n_kv_head:
        n_head //= n_kv_head
    # 重排权重张量
    return (
        weights.reshape(n_head, 2, weights.shape[0] // n_head // 2, *weights.shape[1:])
        .swapaxes(1, 2)
        .reshape(weights.shape)
    )

# 反转HF排列的部分，将权重张量的部分按照指定的部分数、头数和键值头数进行重排
def _reverse_hf_permute_part(
    self, weights: Tensor, n_part: int, n_head: int, n_head_kv: int | None = None,
) -> Tensor:
    # 计算部分的大小
    r = weights.shape[0] // 3
    # 调用_reverse_hf_permute函数进行部分的重排
    return self._reverse_hf_permute(weights[r * n_part:r * n_part + r, ...], n_head, n_head_kv)
# 定义一个私有方法，用于反转权重张量的部分内容
# 参数 weights: 权重张量， n_part: 部分数量
# 返回值：反转后的权重张量
def _reverse_hf_part(self, weights: Tensor, n_part: int) -> Tensor:
    # 计算每部分的大小
    r = weights.shape[0] // 3
    # 返回指定部分的权重张量
    return weights[r * n_part:r * n_part + r, ...]

# FalconModel 类，继承自 Model 类
class FalconModel(Model):
    # 设置 GGUF 参数
    def set_gguf_parameters(self):
        # 获取隐藏层的数量
        block_count = self.hparams.get("num_hidden_layers")
        # 如果未指定，则使用旧名称的参数
        if block_count is None:
            block_count = self.hparams["n_layer"]  # old name

        # 获取注意力头的数量
        n_head = self.hparams.get("num_attention_heads")
        # 如果未指定，则使用旧名称的参数
        if n_head is None:
            n_head = self.hparams["n_head"]  # old name

        # 获取键值头的数量
        n_head_kv = self.hparams.get("num_kv_heads")
        # 如果未指定，则使用旧名称的参数，如果仍未指定，则默认为 1
        if n_head_kv is None:
            n_head_kv = self.hparams.get("n_head_kv", 1)  # old name

        # 向 GGUF 写入器添加名称 "Falcon"
        self.gguf_writer.add_name("Falcon")
# 设置上下文长度为2048
self.gguf_writer.add_context_length(2048)  # not in config.json
# 设置张量数据布局为"jploski"
self.gguf_writer.add_tensor_data_layout("jploski")  # qkv tensor transform
# 设置嵌入长度为self.hparams["hidden_size"]
self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])
# 设置前馈长度为4 * self.hparams["hidden_size"]
self.gguf_writer.add_feed_forward_length(4 * self.hparams["hidden_size"])
# 设置块数量为block_count
self.gguf_writer.add_block_count(block_count)
# 设置头数量为n_head
self.gguf_writer.add_head_count(n_head)
# 设置键值头数量为n_head_kv
self.gguf_writer.add_head_count_kv(n_head_kv)
# 设置层归一化的epsilon值为self.hparams["layer_norm_epsilon"]
self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_epsilon"])
# 设置文件类型为self.ftype
self.gguf_writer.add_file_type(self.ftype)

# 写入张量数据
block_count = self.hparams.get("num_hidden_layers")
if block_count is None:
    block_count = self.hparams["n_layer"]  # old name

n_head = self.hparams.get("num_attention_heads")
if n_head is None:
    n_head = self.hparams["n_head"]  # old name

n_head_kv = self.hparams.get("num_kv_heads")
        # 如果 n_head_kv 为 None，则从 hparams 中获取 n_head_kv 的值，使用旧名称
        if n_head_kv is None:
            n_head_kv = self.hparams.get("n_head_kv", 1)  # old name

        # 计算每个头的维度
        head_dim = self.hparams["hidden_size"] // n_head
        # 获取模型架构中张量的名称映射
        tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)

        # 遍历获取的张量
        for name, data_torch in self.get_tensors():
            # 保存原始数据类型
            old_dtype = data_torch.dtype

            # 将不支持的数据类型转换为 float32
            if data_torch.dtype not in (torch.float16, torch.float32):
                data_torch = data_torch.to(torch.float32)

            # QKV 张量转换
            # 原始的 query_key_value 张量包含 n_head_kv "kv 组"，
            # 每个组包括 n_head/n_head_kv 个查询权重，后跟一个键和一个值权重（由 kv 组中的所有查询头共享）。
            # 这种布局在 GGML 中很难处理。
            # 因此在这里重新排列它们，使得我们有 n_head 个查询权重，后跟 n_head_kv 个键权重，后跟 n_head_kv 个值权重
            # 检查是否包含 "query_key_value" 的名称
            if "query_key_value" in name:
                # 将数据转换为 PyTorch 张量，并按特定方式进行视图重塑
                qkv = data_torch.view(n_head_kv, n_head // n_head_kv + 2, head_dim, head_dim * n_head)
                q = qkv[:, :-2].reshape(n_head * head_dim, head_dim * n_head)
                k = qkv[:, [-2]].reshape(n_head_kv * head_dim, head_dim * n_head)
                v = qkv[:, [-1]].reshape(n_head_kv * head_dim, head_dim * n_head)
                # 将 q、k、v 合并为一个张量，并按原始数据的形状进行重塑
                data_torch = torch.cat((q, k, v)).reshape_as(data_torch)

            # 将 PyTorch 张量转换为 NumPy 数组
            data = data_torch.squeeze().numpy()

            # 映射张量名称
            new_name = tensor_map.get_name(name, try_suffixes=(".weight", ".bias"))
            # 如果无法映射张量名称，则打印错误信息并退出程序
            if new_name is None:
                print(f"Can not map tensor {name!r}")
                sys.exit()

            # 获取数据的维度和数据类型
            n_dims = len(data.shape)
            data_dtype = data.dtype
# 如果需要使用float32类型的数据，将所有float16类型的数据转换为float32
if self.ftype == 0 and data_dtype == np.float16:
    data = data.astype(np.float32)

# TODO: 为什么不能直接使用这些float16数据？应该没有理由将float16存储为float32
if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
    data = data.astype(np.float32)

# 如果需要使用float16类型的数据，将所有float32类型的二维权重张量转换为float16
if self.ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
    data = data.astype(np.float16)

# 打印数据的新名称、维度和数据类型转换前后的变化
print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")

# 将数据添加到gguf_writer中
self.gguf_writer.add_tensor(new_name, data)


class StarCoderModel(Model):
    # 设置gguf参数的方法
    def set_gguf_parameters(self):
# 设置块的数量为模型超参数中的层数
block_count = self.hparams["n_layer"]

# 添加模型名称到 GGUF 写入器
self.gguf_writer.add_name("StarCoder")

# 添加上下文长度到 GGUF 写入器
self.gguf_writer.add_context_length(self.hparams["n_positions"])

# 添加嵌入长度到 GGUF 写入器
self.gguf_writer.add_embedding_length(self.hparams["n_embd"])

# 添加前馈网络长度到 GGUF 写入器
self.gguf_writer.add_feed_forward_length(4 * self.hparams["n_embd"])

# 添加块数量到 GGUF 写入器
self.gguf_writer.add_block_count(block_count)

# 添加头数量到 GGUF 写入器
self.gguf_writer.add_head_count(self.hparams["n_head"])

# 添加键值头数量到 GGUF 写入器
self.gguf_writer.add_head_count_kv(1)

# 添加层归一化 epsilon 到 GGUF 写入器
self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_epsilon"])

# 添加文件类型到 GGUF 写入器
self.gguf_writer.add_file_type(self.ftype)

# 设置 GGUF 参数的方法
class RefactModel(Model):
    # 设置隐藏维度为模型超参数中的嵌入长度
    hidden_dim = self.hparams["n_embd"]
    
    # 设置内部维度为 4 倍的隐藏维度
    inner_dim = 4 * hidden_dim
    
    # 重新设置隐藏维度为内部维度的 2/3
    hidden_dim = int(2 * inner_dim / 3)
    
    # 设置倍数为 256
    multiple_of = 256
    
    # 计算前馈网络维度
    ff_dim = multiple_of * ((hidden_dim + multiple_of - 1) // multiple_of)
        # 获取层的数量
        block_count = self.hparams["n_layer"]

        # 添加“Refact”名称到记录器
        self.gguf_writer.add_name("Refact")
        # 添加上下文长度到记录器，从配置文件中获取
        self.gguf_writer.add_context_length(self.hparams["n_positions"])
        # 添加嵌入长度到记录器，从配置文件中获取
        self.gguf_writer.add_embedding_length(self.hparams["n_embd"])

        # 添加前馈神经网络长度到记录器
        self.gguf_writer.add_feed_forward_length(ff_dim)
        # 添加块的数量到记录器
        self.gguf_writer.add_block_count(block_count)
        # 添加头的数量到记录器，从配置文件中获取
        self.gguf_writer.add_head_count(self.hparams["n_head"])
        # 添加键值头的数量到记录器
        self.gguf_writer.add_head_count_kv(1)
        # 添加层归一化的 RMS 误差到记录器，从配置文件中获取
        self.gguf_writer.add_layer_norm_rms_eps(self.hparams["layer_norm_epsilon"])
        # 添加文件类型到记录器
        self.gguf_writer.add_file_type(self.ftype)

    def write_tensors(self):
        # 获取隐藏层维度
        hidden_dim = self.hparams["n_embd"]
        # 计算内部维度
        inner_dim = 4 * hidden_dim
        # 重新计算隐藏层维度
        hidden_dim = int(2 * inner_dim / 3)
        # 设置为 256 的倍数
        multiple_of = 256
# 计算满足条件的最小整数倍数
ff_dim = multiple_of * ((hidden_dim + multiple_of - 1) // multiple_of)
# 获取头数
n_head = self.hparams["n_head"]
# 键值头数
n_head_kv = 1
# 头维度
head_dim = self.hparams["n_embd"] // n_head
# 块数量
block_count = self.hparams["n_layer"]

# 获取模型架构的张量名称映射
tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)

# 获取张量字典
tensors = dict(self.get_tensors())
# 遍历块数量
for i in range(block_count):
    # 如果存在指定键的张量，则进行处理
    if (w := tensors.get(f"transformer.h.{i}.attn.kv.weight")) is not None:
        # 更新指定键的值
        tensors[f"model.layers.{i}.self_attn.k_proj.weight"] = w[:n_head_kv * head_dim]
        tensors[f"model.layers.{i}.self_attn.v_proj.weight"] = w[n_head_kv * head_dim:]
        # 删除指定键的值
        del tensors[f"transformer.h.{i}.attn.kv.weight"]
    if (w := tensors.get(f"transformer.h.{i}.attn.q.weight")) is not None:
        tensors[f"model.layers.{i}.self_attn.q_proj.weight"] = w
        del tensors[f"transformer.h.{i}.attn.q.weight"]
    if (w := tensors.get(f"transformer.h.{i}.mlp.gate_up_proj.weight")) is not None:
        tensors[f"model.layers.{i}.mlp.gate_proj.weight"] = w[:ff_dim]
        tensors[f"model.layers.{i}.mlp.up_proj.weight"] = w[ff_dim:]
# 删除指定名称的张量
del tensors[f"transformer.h.{i}.mlp.gate_up_proj.weight"]

# 遍历张量字典，获取张量名称和数据
for name, data_torch in tensors.items():
    # 保存原始数据类型
    old_dtype = data_torch.dtype

    # 将不支持的数据类型转换为float32
    if data_torch.dtype not in (torch.float16, torch.float32):
        data_torch = data_torch.to(torch.float32)

    # 将张量数据转换为numpy数组
    data = data_torch.squeeze().numpy()

    # 映射张量名称
    new_name = tensor_map.get_name(name, try_suffixes=(".weight",))
    if new_name is None:
        print(f"Can not map tensor {name!r}")
        sys.exit()

    # 获取数据的维度和数据类型
    n_dims = len(data.shape)
    data_dtype = data.dtype
            # 如果需要 float32 类型的数据，将所有 float16 类型的数据转换为 float32
            if self.ftype == 0 and data_dtype == np.float16:
                data = data.astype(np.float32)

            # TODO: 为什么不能直接使用这些 float16 类型的数据？应该没有理由将 float16 存储为 float32
            if self.ftype == 1 and data_dtype == np.float16 and n_dims == 1:
                data = data.astype(np.float32)

            # 如果需要 float16 类型的数据，将所有 float32 类型的二维权重张量转换为 float16
            if self.ftype == 1 and data_dtype == np.float32 and name.endswith(".weight") and n_dims == 2:
                data = data.astype(np.float16)

            # 打印转换后的数据信息
            print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")

            # 将转换后的数据添加到 gguf_writer 中
            self.gguf_writer.add_tensor(new_name, data)


class PersimmonModel(Model):
    # 设置 gguf 参数
    def set_gguf_parameters(self):
        # 获取层的数量，如果没有指定，则使用默认值
        block_count = self.hparams.get("num_layers", self.hparams.get("num_hidden_layers"))
# 获取注意力头的数量
head_count = self.hparams["num_attention_heads"]
# 将注意力头的数量赋值给键值注意力头的数量
head_count_kv = head_count
# 获取隐藏层的大小
hidden_size = self.hparams["hidden_size"]

# 向gguf_writer中添加名称
self.gguf_writer.add_name('persimmon-8b-chat')
# 向gguf_writer中添加嵌入长度
self.gguf_writer.add_embedding_length(hidden_size)
# 向gguf_writer中添加块数量
self.gguf_writer.add_block_count(block_count)
# 向gguf_writer中添加前馈网络长度
self.gguf_writer.add_feed_forward_length(self.hparams["intermediate_size"])
# 向gguf_writer中添加绳子维度数量
self.gguf_writer.add_rope_dimension_count(hidden_size // head_count)
# 向gguf_writer中添加注意力头数量
self.gguf_writer.add_head_count(head_count)
# 向gguf_writer中添加键值注意力头数量
self.gguf_writer.add_head_count_kv(head_count_kv)
# 向gguf_writer中添加绳子频率基数
self.gguf_writer.add_rope_freq_base(self.hparams["rope_theta"])
# 向gguf_writer中添加层归一化epsilon
self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_eps"])
# 向gguf_writer中添加层归一化RMS epsilon
self.gguf_writer.add_layer_norm_rms_eps(self.hparams["rms_norm_eps"])

# 设置词汇表
def set_vocab(self):
    # 调用_set_vocab_sentencepiece方法
    self._set_vocab_sentencepiece()
    # 向gguf_writer中添加起始标记ID
    # self.gguf_writer.add_bos_token_id(71013)
    # 向gguf_writer中添加结束标记ID
    # self.gguf_writer.add_eos_token_id(71013)
# 定义一个方法用于写入张量数据
def write_tensors(self):
    # 获取模型的层数，如果没有指定，则使用隐藏层的数量
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
        # 将张量数据转换为float32类型，并转换为numpy数组
        data = data_torch.to(torch.float32).squeeze().numpy()
        # 获取新的张量名称
        new_name = tensor_map.get_name(name, try_suffixes=(".weight", ".bias"))
        # 如果找不到新的张量名称，则打印错误信息并退出程序
        if new_name is None:
            print(f"Can not map tensor {name!r}")
            sys.exit()
        # 获取数据的维度，并打印数据类型转换信息
        n_dims = len(data.shape)
        print(f"{new_name}, n_dims = {n_dims}, {old_dtype} --> {data.dtype}")
        # 将新的张量名称和数据添加到gguf_writer中
        self.gguf_writer.add_tensor(new_name, data)


class StableLMModel(Model):
# 设置 GGUF 参数
def set_gguf_parameters(self):
    # 获取超参数
    hparams = self.hparams
    # 获取隐藏层的数量
    block_count = hparams["num_hidden_layers"]

    # 添加模型名称
    self.gguf_writer.add_name(dir_model.name)
    # 添加上下文长度
    self.gguf_writer.add_context_length(hparams["max_position_embeddings"])
    # 添加嵌入长度
    self.gguf_writer.add_embedding_length(hparams["hidden_size"])
    # 添加块数量
    self.gguf_writer.add_block_count(block_count)
    # 添加前馈长度
    self.gguf_writer.add_feed_forward_length(hparams["intermediate_size"])
    # 添加绳子维度数量
    self.gguf_writer.add_rope_dimension_count(int(hparams["rope_pct"]*(hparams["hidden_size"] // hparams["num_attention_heads"])))
    # 添加头数量
    self.gguf_writer.add_head_count(hparams["num_attention_heads"])
    # 添加并行残差
    self.gguf_writer.add_parallel_residual(hparams["use_parallel_residual"] if "use_parallel_residual" in hparams else True)
    # 添加层归一化 epsilon
    self.gguf_writer.add_layer_norm_eps(1e-5)

# 解析参数
def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(description="Convert a huggingface model to a GGML compatible file")
    parser.add_argument(
        "--vocab-only", action="store_true",
# 添加命令行参数，用于提取词汇表
parser.add_argument(
    help="extract only the vocab",
)

# 添加命令行参数，用于指定输出文件路径，默认为基于输入文件的路径
parser.add_argument(
    "--outfile", type=Path,
    help="path to write to; default: based on input",
)

# 添加命令行参数，用于指定输出格式，可选值为"f32"和"f16"，默认为"f16"
parser.add_argument(
    "--outtype", type=str, choices=["f32", "f16"], default="f16",
    help="output format - use f32 for float32, f16 for float16",
)

# 添加命令行参数，用于指示模型在大端机器上执行
parser.add_argument("--bigendian", action="store_true", help="model is executed on big endian machine")

# 添加命令行参数，用于指定模型文件所在的目录
parser.add_argument(
    "model", type=Path,
    help="directory containing model file",
)

# 解析命令行参数并返回结果
return parser.parse_args()

# 解析命令行参数并存储在args变量中
args = parse_args()
# 获取模型目录
dir_model = args.model
# 如果模型目录不存在，则打印错误信息并退出程序
if not dir_model.is_dir():
    print(f'Error: {args.model} is not a directory', file=sys.stderr)
    sys.exit(1)

# 定义文件类型映射关系
ftype_map = {
    "f32": gguf.GGMLQuantizationType.F32,
    "f16": gguf.GGMLQuantizationType.F16,
}

# 如果指定了输出文件，则使用指定的文件名，否则默认在模型目录下生成输出文件
if args.outfile is not None:
    fname_out = args.outfile
else:
    fname_out = dir_model / f'ggml-model-{args.outtype}.gguf'

# 打印加载模型的信息
print(f"Loading model: {dir_model.name}")

# 加载模型的超参数
hparams = Model.load_hparams(dir_model)
# 根据模型架构创建模型类实例
model_class = Model.from_model_architecture(hparams["architectures"][0])
# 根据模型类实例化模型对象
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
# 否则导出整个模型
else:
    print(f"Exporting model to '{fname_out}'")
    model_instance.write()

# 打印导出成功的消息
print(f"Model successfully exported to '{fname_out}'")
```