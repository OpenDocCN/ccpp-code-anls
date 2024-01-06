# `PowerInfer\convert.py`

```
#!/usr/bin/env python3
# 指定使用 Python3 解释器

from __future__ import annotations
# 导入未来版本的特性，使得类型提示中的类型注解可以使用字符串形式的类名

import argparse
# 导入用于解析命令行参数和生成帮助信息的模块

import concurrent.futures
# 导入用于并发执行任务的模块

import dataclasses
# 导入用于创建不可变对象的模块

import enum
# 导入用于创建枚举类型的模块

import faulthandler
# 导入用于调试时追踪 Python 中的故障的模块

import functools
# 导入用于创建高阶函数的模块

import itertools
# 导入用于创建迭代器的模块

import json
# 导入用于处理 JSON 数据的模块

import math
# 导入数学计算相关的模块

import mmap
# 导入用于内存映射文件的模块

import pickle
# 导入用于序列化和反序列化 Python 对象的模块

import re
# 导入用于处理正则表达式的模块

import signal
# 导入用于处理信号的模块

import struct
# 导入用于处理字节数据的模块

import subprocess
# 导入用于创建子进程并与其进行交互的模块

import sys
# 导入用于访问 Python 解释器的一些变量和函数的模块

import time
# 导入用于处理时间的模块
# 导入 zipfile 模块，用于处理 ZIP 文件
import zipfile
# 导入 ABCMeta 和 abstractmethod，用于定义抽象基类和抽象方法
from abc import ABCMeta, abstractmethod
# 导入 ProcessPoolExecutor 和 ThreadPoolExecutor，用于并发执行任务
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor
# 导入 dataclass，用于创建数据类
from dataclasses import dataclass
# 导入 Path，用于处理文件路径
from pathlib import Path
# 导入 IO、TYPE_CHECKING、Any、Callable、Iterable、Literal、TypeVar，用于类型提示
from typing import IO, TYPE_CHECKING, Any, Callable, Iterable, Literal, TypeVar
# 导入 numpy 模块，用于科学计算
import numpy as np
# 导入 SentencePieceProcessor，用于处理文本分词
from sentencepiece import SentencePieceProcessor

# 导入 os 模块，用于与操作系统交互
import os
# 如果环境变量中没有 'NO_LOCAL_GGUF'，则将 gguf-py 目录添加到路径中
if 'NO_LOCAL_GGUF' not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))
# 导入 gguf 模块
import gguf

# 如果 TYPE_CHECKING 为真，导入 TypeAlias
if TYPE_CHECKING:
    from typing import TypeAlias

# 如果 faulthandler 模块中有 register 方法，并且 signal 模块中有 SIGUSR1 常量
# 则注册 SIGUSR1 信号
if hasattr(faulthandler, 'register') and hasattr(signal, 'SIGUSR1'):
    faulthandler.register(signal.SIGUSR1)
# 定义一个类型别名，将 'np.ndarray[Any, Any]' 赋值给 NDArray
NDArray: TypeAlias = 'np.ndarray[Any, Any]'

# 将 gguf.MODEL_ARCH.LLAMA 赋值给 ARCH
ARCH = gguf.MODEL_ARCH.LLAMA

# 将 8 赋值给 DEFAULT_CONCURRENCY
DEFAULT_CONCURRENCY = 8

# 定义一个数据类型的数据类
@dataclass(frozen=True)
class DataType:
    # 数据类型的名称
    name: str
    # 数据类型的 numpy 数据类型
    dtype: np.dtype[Any]
    # 有效的转换列表
    valid_conversions: list[str]

    # 计算元素占用的字节数
    def elements_to_bytes(self, n_elements: int) -> int:
        return n_elements * self.dtype.itemsize

# 定义一个数据类
@dataclass(frozen=True)
# 定义一个未量化的数据类型类，继承自 DataType 类
class UnquantizedDataType(DataType):
    pass

# 定义四种未量化的数据类型对象，包括数据类型名称、数据类型、有效转换类型
DT_F16  = UnquantizedDataType('F16', dtype = np.dtype(np.float16), valid_conversions = ['F32', 'Q8_0'])
DT_F32  = UnquantizedDataType('F32', dtype = np.dtype(np.float32), valid_conversions = ['F16', 'Q8_0'])
DT_I32  = UnquantizedDataType('I32', dtype = np.dtype(np.int16), valid_conversions = [])
DT_BF16 = UnquantizedDataType('BF16', dtype = np.dtype(np.uint16), valid_conversions = ['F32', 'F16', 'Q8_0'])

# 定义一个量化的数据类型类，继承自 DataType 类，并使用 dataclass 进行装饰
@dataclass(frozen=True)
class QuantizedDataType(DataType):
    block_size: int
    quantized_dtype: np.dtype[Any]
    ggml_type: gguf.GGMLQuantizationType

    # 定义 quantize 方法，用于量化数组
    def quantize(self, arr: NDArray) -> NDArray:
        raise NotImplementedError(f'Quantization for {self.name} not implemented')

    # 定义 elements_to_bytes 方法，用于将元素数量转换为字节数
    def elements_to_bytes(self, n_elements: int) -> int:
        assert n_elements % self.block_size == 0, f'Invalid number of elements {n_elements} for {self.name} with block size {self.block_size}'
        return self.quantized_dtype.itemsize * (n_elements // self.block_size)
# 使用 dataclass 装饰器创建一个不可变的 Q8_0QuantizedDataType 类，继承自 QuantizedDataType
class Q8_0QuantizedDataType(QuantizedDataType):
    # Mini Q8_0 quantization in Python!
    # 定义 quantize 方法，用于对数组进行量化
    def quantize(self, arr: NDArray) -> NDArray:
        # 断言数组大小是块大小的整数倍且不为0
        assert arr.size % self.block_size == 0 and arr.size != 0, f'Bad array size {arr.size}'
        # 断言数组类型为 np.float32
        assert arr.dtype == np.float32, f'Bad array type {arr.dtype}'
        # 计算块的数量
        n_blocks = arr.size // self.block_size
        # 将数组重塑为块
        blocks = arr.reshape((n_blocks, self.block_size))
        # 由 @Cebtenzzre 贡献的块量化的更快实现
        # 定义 quantize_blocks_q8_0 函数，用于对块进行量化
        def quantize_blocks_q8_0(blocks: NDArray) -> Iterable[tuple[Any, Any]]:
            # 计算每个块的最大值，并进行量化
            d = abs(blocks).max(axis = 1) / np.float32(127)
            with np.errstate(divide = 'ignore'):
                qs = (blocks / d[:, None]).round()
            qs[d == 0] = 0
            # 生成器，返回每个块的最大值和量化后的块
            yield from zip(d, qs)
        # 从 quantize_blocks_q8_0 生成的迭代器中创建数组，指定数量和数据类型
        return np.fromiter(quantize_blocks_q8_0(blocks), count = n_blocks, dtype = self.quantized_dtype)

# 创建一个 Q8_0QuantizedDataType 实例 DT_Q8_0
DT_Q8_0 = Q8_0QuantizedDataType('Q8_0',
    dtype = np.dtype(np.float32), valid_conversions = [],
# 定义变量 ggml_type，赋值为 GGMLQuantizationType.Q8_0，block_size 为 32，quantized_dtype 为指定的 np.dtype
ggml_type = gguf.GGMLQuantizationType.Q8_0, block_size = 32, quantized_dtype = np.dtype([('d', '<f2'), ('qs', 'i1', (32,))])

# 创建一个空的字典 NUMPY_TYPE_TO_DATA_TYPE
NUMPY_TYPE_TO_DATA_TYPE: dict[np.dtype[Any], DataType] = {}

# 遍历指定的数据类型列表，如果已经存在于 NUMPY_TYPE_TO_DATA_TYPE 中则抛出数值错误
for dt in (DT_BF16, DT_F16, DT_F32, DT_I32):
    if dt.dtype in NUMPY_TYPE_TO_DATA_TYPE:
        raise ValueError(f'Invalid duplicate data type {dt}')
    # 将数据类型添加到 NUMPY_TYPE_TO_DATA_TYPE 字典中
    NUMPY_TYPE_TO_DATA_TYPE[dt.dtype] = dt

# 创建一个字典 SAFETENSORS_DATA_TYPES，将字符串数据类型映射到指定的 DataType
SAFETENSORS_DATA_TYPES: dict[str, DataType] = {
    'BF16': DT_BF16,
    'F16': DT_F16,
    'F32': DT_F32,
    'I32': DT_I32,
}

# 待办事项：将此与 `llama_ftype` 匹配
# 待办事项：重命名为 LLAMAFileType
# 待办事项：移动到 `gguf.py`
# 定义一个枚举类型 GGMLFileType，包含了不同的文件类型
class GGMLFileType(enum.IntEnum):
    AllF32     = 0  # 表示全部为 F32 类型
    MostlyF16  = 1  # 表示大部分为 F16 类型，除了 1 维张量
    MostlyQ8_0 = 7  # 表示大部分为 Q8_0 类型，除了 1 维张量

    # 根据文件类型和张量返回数据类型
    def type_for_tensor(self, name: str, tensor: LazyTensor) -> DataType:
        # 从 GGML_FILE_TYPE_TO_DATA_TYPE 字典中获取对应的数据类型
        dt = GGML_FILE_TYPE_TO_DATA_TYPE.get(self)
        if dt is None:
            raise ValueError(self)
        # 如果张量的维度大于 1，则返回对应的数据类型，否则返回 F32 类型
        return dt if len(tensor.shape) > 1 else DT_F32

# 定义 GGMLFileType 到 DataType 的映射关系
GGML_FILE_TYPE_TO_DATA_TYPE: dict[GGMLFileType, DataType] = {
    GGMLFileType.AllF32    : DT_F32,
    GGMLFileType.MostlyF16 : DT_F16,
    GGMLFileType.MostlyQ8_0: DT_Q8_0,
}

#
# hparams loading
# 导入 dataclass 模块，用于创建数据类
@dataclass
class PredictorParams:
    # 定义稀疏阈值参数，默认为 None
    sparse_threshold: float | None = None

    # 静态方法，从 JSON 文件中加载预测器参数
    @staticmethod
    def loadPredictorJson(model: LazyModel, config_path: Path) -> PredictorParams:
        # 从配置文件中加载 JSON 数据
        config = json.load(open(config_path))
        # 返回预测器参数对象
        return PredictorParams(
            sparse_threshold = config.get("sparse_threshold"),
        )

    # 静态方法，加载模型参数
    @staticmethod
    def load(model_plus: ModelPlus) -> PredictorParams:
        # 获取配置文件路径
        config_path   = model_plus.paths[0].parent / "config.json"

        # 如果配置文件存在，则加载预测器参数
        if config_path.exists():
            params = PredictorParams.loadPredictorJson(model_plus.model, config_path)
        else:
            # 如果配置文件不存在，则...
# 创建一个名为PredictorParams的类实例
params = PredictorParams()
# 返回创建的类实例
return params

# 创建一个名为Params的数据类，包含了各种参数的定义
@dataclass
class Params:
    n_vocab:    int  # 词汇量的大小
    n_embd:     int  # 嵌入层的维度
    n_layer:    int  # 神经网络的层数
    n_ctx:      int  # 上下文的大小
    n_ff:       int  # 前馈神经网络的隐藏层维度
    n_head:     int  # 多头注意力机制的头数
    n_head_kv:  int  # 键值多头注意力机制的头数
    f_norm_eps: float  # 归一化层的epsilon值

    rope_scaling_type: gguf.RopeScalingType | None = None  # 绳索缩放类型
    f_rope_freq_base: float | None = None  # 绳索频率基数
    f_rope_scale: float | None = None  # 绳索缩放比例
    n_orig_ctx: int | None = None  # 原始上下文大小
    rope_finetuned: bool | None = None  # 绳索微调标志
    # 定义文件类型变量，可以是 GGMLFileType 类型或者 None
    ftype: GGMLFileType | None = None

    # 定义模型文件所在目录的路径，可以是 Path 类型或者 None
    path_model: Path | None = None

    # MLP 预测器参数
    predictor_params: PredictorParams = dataclasses.field(default_factory=PredictorParams)

    @staticmethod
    def guessed(model: LazyModel) -> Params:
        # 尝试使用 transformer 命名方式获取参数
        n_vocab, n_embd = model["model.embed_tokens.weight"].shape if "model.embed_tokens.weight" in model else model["tok_embeddings.weight"].shape

        # 尝试使用 transformer 命名方式获取参数
        if "model.layers.0.self_attn.q_proj.weight" in model:
            # 获取层数
            n_layer=next(i for i in itertools.count() if f"model.layers.{i}.self_attn.q_proj.weight" not in model)
        # 尝试使用 baichuan 命名方式获取参数
        elif "model.layers.0.self_attn.W_pack.weight" in model:
            # 获取层数
            n_layer=next(i for i in itertools.count() if f"model.layers.{i}.self_attn.W_pack.weight" not in model)
        else:
            # 其他情况
# 使用 itertools.count() 生成一个无限迭代器，找到第一个不在模型中的层的编号
n_layer=next(i for i in itertools.count() if f"layers.{i}.attention.wq.weight" not in model)

# 如果没有找到层的编号，抛出异常
if n_layer < 1:
    raise Exception("failed to guess 'n_layer'. This model is unknown or unsupported.\n"
                    "Suggestion: provide 'config.json' of the model in the same directory containing model files.")

# 根据经验猜测头的数量
n_head = n_embd // 128 # guessed
# 根据经验猜测多头注意力的倍数
n_mult = 256           # guessed

# 计算 Feed Forward 网络的维度
n_ff = int(2 * (4 * n_embd) / 3)
# 根据经验猜测 Feed Forward 网络的维度
n_ff = n_mult * ((n_ff + n_mult - 1) // n_mult)

# 返回模型参数
return Params(
    n_vocab    = n_vocab,
    n_embd     = n_embd,
    n_layer    = n_layer,
    n_ctx      = -1,
    n_ff       = n_ff,
    n_head     = n_head,
# 定义一个静态方法，用于从 JSON 文件中加载 HFTransformer 模型的配置参数
@staticmethod
def loadHFTransformerJson(model: LazyModel, config_path: Path) -> Params:
    # 从配置文件中加载配置参数
    config = json.load(open(config_path))

    # 初始化一些变量
    rope_scaling_type = f_rope_scale = n_orig_ctx = rope_finetuned = None
    # 获取配置中的绳子缩放参数
    rope_scaling = config.get("rope_scaling")

    # 如果配置中存在绳子缩放参数，并且有类型
    if rope_scaling is not None and (typ := rope_scaling.get("type")):
        # 获取绳子缩放因子
        rope_factor = rope_scaling.get("factor")
        f_rope_scale = rope_factor
        # 根据类型设置绳子缩放类型
        if typ == "linear":
            rope_scaling_type = gguf.RopeScalingType.LINEAR
        elif typ == "yarn":
            rope_scaling_type = gguf.RopeScalingType.YARN
            # 获取原始上下文长度和是否进行了微调
            n_orig_ctx = rope_scaling['original_max_position_embeddings']
            rope_finetuned = rope_scaling['finetuned']
        else:
            # 如果遇到未知的绳索缩放类型，则抛出未实现的错误
            raise NotImplementedError(f'Unknown rope scaling type: {typ}')

        # 根据配置文件中的参数设置最大序列长度
        if "max_sequence_length" in config:
            n_ctx = config["max_sequence_length"]
        # 如果配置文件中没有设置最大序列长度，则使用最大位置嵌入长度作为最大序列长度
        elif "max_position_embeddings" in config:
            n_ctx = config["max_position_embeddings"]
        # 如果都没有设置，则抛出异常
        else:
            raise Exception("failed to guess 'n_ctx'. This model is unknown or unsupported.\n"
                            "Suggestion: provide 'config.json' of the model in the same directory containing model files.")

        # 返回参数对象，包括词汇量、嵌入大小、层数、最大序列长度、前馈网络大小、注意力头数、键值头数、归一化参数
        return Params(
            n_vocab           = config["vocab_size"],
            n_embd            = config["hidden_size"],
            n_layer           = config["num_hidden_layers"],
            n_ctx             = n_ctx,
            n_ff              = config["intermediate_size"],
            n_head            = (n_head := config["num_attention_heads"]),
            n_head_kv         = config.get("num_key_value_heads", n_head),
            f_norm_eps        = config["rms_norm_eps"],
# 设置参数值，从配置中获取绳子频率基础值
f_rope_freq_base  = config.get("rope_theta"),
# 设置绳子缩放类型
rope_scaling_type = rope_scaling_type,
# 设置绳子缩放因子
f_rope_scale      = f_rope_scale,
# 设置原始上下文数量
n_orig_ctx        = n_orig_ctx,
# 设置绳子微调
rope_finetuned    = rope_finetuned,

# LLaMA v2 70B params.json
# 从配置文件中加载参数
@staticmethod
def loadOriginalParamsJson(model: LazyModel, config_path: Path) -> Params:
    # 读取配置文件
    config = json.load(open(config_path))

    # 用于确定 LLaMA v1 vs v2 vs CodeLlama 的 hack
    if config.get("rope_theta") == 1000000:
        # 如果绳子频率基础值为1000000，则为 CodeLlama
        n_ctx = 16384
    elif config["norm_eps"] == 1e-05:
        # 如果 norm_eps 为 1e-05，则为 LLaMA v2
        n_ctx = 4096
        else:
            # 如果不是LLaMA v1模型，则设置默认的上下文长度为2048
            n_ctx = 2048

        # 返回模型参数对象，包括词汇量大小、嵌入维度、层数、上下文长度、前馈网络大小、注意力头数、键值注意力头数、层归一化参数、Rope频率基准
        return Params(
            # 词汇量大小为配置文件中的设置，如果没有设置则取模型的词嵌入矩阵的行数
            n_vocab          = config.get("vocab_size", model["tok_embeddings.weight"].shape[0]),
            # 嵌入维度为配置文件中的维度
            n_embd           = config["dim"],
            # 层数为配置文件中的层数
            n_layer          = config["n_layers"],
            # 上下文长度为之前设置的n_ctx
            n_ctx            = n_ctx,
            # 前馈网络大小为模型第一层的前馈网络权重矩阵的行数
            n_ff             = model["layers.0.feed_forward.w1.weight"].shape[0],
            # 注意力头数为配置文件中的设置
            n_head           = (n_head := config["n_heads"]),
            # 键值注意力头数为配置文件中的设置，如果没有设置则取注意力头数
            n_head_kv        = config.get("n_kv_heads", n_head),
            # 层归一化参数为配置文件中的设置
            f_norm_eps       = config["norm_eps"],
            # Rope频率基准为配置文件中的设置，如果没有设置则为空
            f_rope_freq_base = config.get("rope_theta"),
        )

    @staticmethod
    def load(model_plus: ModelPlus) -> Params:
        # 获取模型配置文件路径和参数文件路径
        hf_config_path   = model_plus.paths[0].parent / "config.json"
        orig_config_path = model_plus.paths[0].parent / "params.json"
# 检查文件路径是否存在，如果存在则加载 HFTransformerJson 文件中的参数
if hf_config_path.exists():
    params = Params.loadHFTransformerJson(model_plus.model, hf_config_path)
# 如果 HFTransformerJson 文件不存在，但原始配置文件存在，则加载原始配置文件中的参数
elif orig_config_path.exists():
    params = Params.loadOriginalParamsJson(model_plus.model, orig_config_path)
# 如果模型格式不是 'none'，则尝试猜测参数
elif model_plus.format != 'none':
    params = Params.guessed(model_plus.model)
# 如果以上条件都不满足，则抛出数值错误
else:
    raise ValueError('Cannot guess params when model format is none')

# 设置参数中的模型路径为模型路径列表中的第一个路径的父路径
params.path_model = model_plus.paths[0].parent

# 返回参数对象
return params
# 初始化函数，接受文件名和可选的添加的标记文件名作为参数
def __init__(self, fname_tokenizer: Path, fname_added_tokens: Path | None) -> None:
    # 从文件中加载 BPE 分词器的配置
    self.bpe_tokenizer = json.loads(open(str(fname_tokenizer), encoding="utf-8").read())
    added_tokens: dict[str, int]
    # 如果存在添加的标记文件，则加载其中的标记
    if fname_added_tokens is not None:
        # FIXME: 验证这里添加的标记不能与主词汇表重叠
        added_tokens = json.load(open(fname_added_tokens, encoding="utf-8"))
    else:
        # 否则尝试在 tokenizer.json 中查找添加的标记
        tokenizer_json_file = fname_tokenizer.parent / 'tokenizer.json'
        if not tokenizer_json_file.is_file():
            added_tokens = {}
        else:
            tokenizer_json = json.load(open(tokenizer_json_file, encoding="utf-8"))
            # 从 tokenizer.json 中获取添加的标记，可能会有重复的标记
            added_tokens = dict(
                (item['content'], item['id'])
                for item in tokenizer_json.get('added_tokens', [])
                # 这里添加的标记可能与 BPE 分词器的标记重复
                if item['content'] not in self.bpe_tokenizer )

    # 获取词汇表的大小
    vocab_size: int = len(self.bpe_tokenizer)
# 创建一个期望的 ID 列表，范围从 vocab_size 到 vocab_size + len(added_tokens)
expected_ids    = list(range(vocab_size, vocab_size + len(added_tokens)))
# 获取实际的 ID 列表，并按升序排序
actual_ids      = sorted(added_tokens.values())
# 检查期望的 ID 列表和实际的 ID 列表是否一致，如果不一致则抛出异常
if expected_ids != actual_ids:
    # 计算期望的结束 ID
    expected_end_id = vocab_size + len(actual_ids) - 1
    raise Exception(f"Expected the {len(actual_ids)} added token ID(s) to be sequential in the range {vocab_size} - {expected_end_id}; got {actual_ids}")

# 按照 added_tokens 的值（ID）排序，返回一个元组列表
items = sorted(added_tokens.items(), key=lambda text_idx: text_idx[1])
# 从排序后的元组列表中提取文本，存储到 added_tokens_list 中
self.added_tokens_list    = [text for (text, idx) in items]
# 存储原始的 vocab_size
self.vocab_size_base: int = vocab_size
# 计算新的 vocab_size
self.vocab_size: int      = self.vocab_size_base + len(self.added_tokens_list)
# 存储文件名 tokenizer
self.fname_tokenizer      = fname_tokenizer
# 存储文件名 added_tokens
self.fname_added_tokens   = fname_added_tokens

# 生成 BPE tokens 的迭代器
def bpe_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
    # 获取 BPE tokenizer
    tokenizer = self.bpe_tokenizer
    # 导入 GPT2 模型的 tokenization_gpt2 模块
    from transformers.models.gpt2 import tokenization_gpt2
    # 创建一个反转的词汇表，将 ID 映射回编码的 token
    reverse_vocab = {id: encoded_tok for encoded_tok, id in tokenizer.items()}

    # 遍历 tokenizer，生成 BPE tokens 的迭代器
    for i, _ in enumerate(tokenizer):
        # 返回反转词汇表中的 token，概率为 0.0，类型为 NORMAL
        yield reverse_vocab[i], 0.0, gguf.TokenType.NORMAL
# 返回已添加的标记的迭代器，每个标记由字节、分数和标记类型组成
def added_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
    # 遍历已添加的标记列表
    for text in self.added_tokens_list:
        # 设置默认分数
        score = -1000.0
        # 返回标记的字节编码、分数和标记类型
        yield text.encode("utf-8"), score, gguf.TokenType.CONTROL

# 返回所有标记的迭代器，包括基本标记和已添加的标记
def all_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
    # 返回基于BPE的标记迭代器
    yield from self.bpe_tokens()
    # 返回已添加的标记迭代器
    yield from self.added_tokens()

# 返回对象的字符串表示形式
def __repr__(self) -> str:
    return f"<BpeVocab with {self.vocab_size_base} base tokens and {len(self.added_tokens_list)} added tokens>"

# SentencePieceVocab类的构造函数
class SentencePieceVocab:
    def __init__(self, fname_tokenizer: Path, fname_added_tokens: Path | None) -> None:
        # 使用给定的文件名初始化SentencePieceProcessor对象
        self.sentencepiece_tokenizer = SentencePieceProcessor(str(fname_tokenizer))
        added_tokens: dict[str, int]
        # 如果存在已添加标记的文件名
        if fname_added_tokens is not None:
            # 从文件中加载已添加标记的字典
            added_tokens = json.load(open(fname_added_tokens, encoding="utf-8"))
        else:
            added_tokens = {}  # 如果没有新的 token 被添加，则初始化为空字典

        vocab_size: int = self.sentencepiece_tokenizer.vocab_size()  # 获取当前 tokenizer 的词汇量大小

        new_tokens       = {id: piece for piece, id in added_tokens.items() if id >= vocab_size}  # 获取新增的 token，并且其 ID 大于当前词汇量大小
        expected_new_ids = list(range(vocab_size, vocab_size + len(new_tokens)))  # 期望的新增 token 的 ID 序列
        actual_new_ids   = sorted(new_tokens.keys())  # 实际新增 token 的 ID 序列

        if expected_new_ids != actual_new_ids:  # 检查新增 token 的 ID 是否是连续的
            raise ValueError(f"Expected new token IDs {expected_new_ids} to be sequential; got {actual_new_ids}")

        # 存储被添加到基础词汇表中的 token
        self.added_tokens_list  = [new_tokens[id] for id in actual_new_ids]
        self.vocab_size_base    = vocab_size  # 基础词汇表的大小
        self.vocab_size         = self.vocab_size_base + len(self.added_tokens_list)  # 更新后的词汇表大小
        self.fname_tokenizer    = fname_tokenizer  # 存储 tokenizer 的文件名
        self.fname_added_tokens = fname_added_tokens  # 存储新增 token 的文件名

    def sentencepiece_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
# 获取句子分词器
tokenizer = self.sentencepiece_tokenizer
# 遍历词汇表中的所有词
for i in range(tokenizer.vocab_size()):
    # 获取词汇表中对应 id 的词
    piece = tokenizer.id_to_piece(i)
    # 将词转换为 UTF-8 编码的字节流
    text: bytes = piece.encode("utf-8")
    # 获取词的得分
    score: float = tokenizer.get_score(i)

    # 初始化词的类型为普通词
    toktype = gguf.TokenType.NORMAL
    # 如果词是未知词，则将类型设置为未知词
    if tokenizer.is_unknown(i):
        toktype = gguf.TokenType.UNKNOWN
    # 如果词是控制字符，则将类型设置为控制字符
    if tokenizer.is_control(i):
        toktype = gguf.TokenType.CONTROL

    # 如果词是用户自定义的词，根据注释的链接，可能需要将类型设置为用户自定义
    # if tokenizer.is_user_defined(i): toktype = gguf.TokenType.USER_DEFINED

    # 如果词是未使用的词，则将类型设置为未使用的词
    if tokenizer.is_unused(i):
        toktype = gguf.TokenType.UNUSED
    # 如果词是字节，则将类型设置为字节
    if tokenizer.is_byte(i):
        toktype = gguf.TokenType.BYTE
# 生成器函数，返回文本、分数和标记类型的元组
def sentencepiece_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
    for text, score, toktype in self.tokenizer:
        yield text, score, toktype

# 生成器函数，返回已添加的标记的文本、分数和标记类型的元组
def added_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
    for text in self.added_tokens_list:
        score = -1000.0
        yield text.encode("utf-8"), score, gguf.TokenType.USER_DEFINED

# 生成器函数，返回所有标记的文本、分数和标记类型的元组
def all_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
    yield from self.sentencepiece_tokens()
    yield from self.added_tokens()

# 返回表示SentencePieceVocab对象的字符串
def __repr__(self) -> str:
    return f"<SentencePieceVocab with {self.vocab_size_base} base tokens and {len(self.added_tokens_list)} added tokens>"

# 定义Vocab别名类型
Vocab: TypeAlias = 'BpeVocab | SentencePieceVocab'

# 数据加载
# TODO: 重用（可能移动到gguf.py中）
# 定义一个函数，用于对权重进行排列
def permute(weights: NDArray, n_head: int, n_head_kv: int) -> NDArray:
    # 如果指定了 n_head_kv 并且 n_head 不等于 n_head_kv，则将 n_head 更新为 n_head_kv
    if n_head_kv is not None and n_head != n_head_kv:
        n_head = n_head_kv
    # 对权重进行重塑，然后交换轴，最后再次重塑，返回结果
    return (weights.reshape(n_head, 2, weights.shape[0] // n_head // 2, *weights.shape[1:])
                .swapaxes(1, 2)
                .reshape(weights.shape))

# 定义一个抽象类 Tensor
class Tensor(metaclass=ABCMeta):
    data_type: DataType

    # 定义抽象方法 astype，用于将数据类型转换为指定类型
    @abstractmethod
    def astype(self, data_type: DataType) -> Tensor: ...
    # 定义抽象方法 permute，用于对张量进行排列
    @abstractmethod
    def permute(self, n_head: int, n_head_kv: int) -> Tensor: ...
    # 定义抽象方法 permute_part，用于对部分张量进行排列
    @abstractmethod
    def permute_part(self, n_part: int, n_head: int, n_head_kv: int) -> UnquantizedTensor: ...
# 定义一个抽象方法part，接受一个整数参数n_part，返回一个未量化的张量
@abstractmethod
def part(self, n_part: int) -> UnquantizedTensor: ...

# 定义一个抽象方法to_ggml，返回一个GGMLCompatibleTensor
@abstractmethod
def to_ggml(self) -> GGMLCompatibleTensor: ...


# 将bf16_arr中的数据从bf16格式转换为fp32格式，并返回一个NDArray
def bf16_to_fp32(bf16_arr: np.ndarray[Any, np.dtype[np.uint16]]) -> NDArray:
    # 检查输入数组的数据类型是否为uint16
    assert bf16_arr.dtype == np.uint16, f"Input array should be of dtype uint16, but got {bf16_arr.dtype}"
    # 将bf16_arr中的数据转换为fp32格式
    fp32_arr = bf16_arr.astype(np.uint32) << 16
    return fp32_arr.view(np.float32)


# 定义一个未量化的张量类UnquantizedTensor，继承自Tensor类
class UnquantizedTensor(Tensor):
    # 初始化方法，接受一个NDArray参数ndarray
    def __init__(self, ndarray: NDArray) -> None:
        # 检查ndarray是否为np.ndarray类型
        assert isinstance(ndarray, np.ndarray)
        # 将ndarray赋值给实例变量ndarray
        self.ndarray = ndarray
        # 将ndarray的数据类型转换为对应的数据类型
        self.data_type = NUMPY_TYPE_TO_DATA_TYPE[ndarray.dtype]

    # 将张量转换为指定数据类型的张量
    def astype(self, data_type: DataType) -> Tensor:
        # 获取目标数据类型的dtype
        dtype = data_type.dtype
    # 如果数据类型为 DT_BF16，则将数据从bf16转换为fp32
    if self.data_type == DT_BF16:
        self.ndarray = bf16_to_fp32(self.ndarray)
    # 返回一个未量化的张量，将数据类型转换为指定的dtype
    return UnquantizedTensor(self.ndarray.astype(dtype))

    # 返回自身，表示转换为GGML格式的未量化张量
    def to_ggml(self) -> UnquantizedTensor:
        return self

    # 对张量的部分进行置换，返回一个未量化的张量
    def permute_part(self, n_part: int, n_head: int, n_head_kv: int) -> UnquantizedTensor:
        r = self.ndarray.shape[0] // 3
        return UnquantizedTensor(permute(self.ndarray[r * n_part : r * n_part + r, ...], n_head, n_head_kv))

    # 返回张量的部分，返回一个未量化的张量
    def part(self, n_part: int) -> UnquantizedTensor:
        r = self.ndarray.shape[0] // 3
        return UnquantizedTensor(self.ndarray[r * n_part : r * n_part + r, ...])

    # 对张量进行置换，返回一个未量化的张量
    def permute(self, n_head: int, n_head_kv: int) -> UnquantizedTensor:
        return UnquantizedTensor(permute(self.ndarray, n_head, n_head_kv))

# 加载未量化的懒惰张量，返回一个NDArray
def load_unquantized(lazy_tensor: LazyTensor, expected_dtype: Any = None, convert: bool = False) -> NDArray:
# 从懒加载的张量中加载数据
tensor = lazy_tensor.load()
# 断言张量是未量化的张量类型
assert isinstance(tensor, UnquantizedTensor)

# 再次检查：
# 获取张量的实际形状
actual_shape = list(tensor.ndarray.shape)
# 断言实际形状与懒加载张量的形状相同，如果不同则抛出异常
assert actual_shape == lazy_tensor.shape, (actual_shape, lazy_tensor.shape)
# 如果期望的数据类型不为空且与张量的数据类型不同
if expected_dtype is not None and expected_dtype != tensor.ndarray.dtype:
    # 如果允许转换数据类型
    if convert:
        # 将张量的数据类型转换为期望的数据类型
        tensor.ndarray = tensor.ndarray.astype(expected_dtype)
    else:
        # 否则抛出数值错误异常
        raise ValueError(f'expected this tensor to have dtype {expected_dtype}, got {tensor.ndarray.dtype}')

# 返回张量的数据
return tensor.ndarray

# 将 GGMLCompatibleTensor 设置为 UnquantizedTensor 的别名
GGMLCompatibleTensor = UnquantizedTensor

# 定义 LazyTensor 类
@dataclass
class LazyTensor:
    # 定义一个_load方法，该方法返回一个Tensor类型的对象
    _load: Callable[[], Tensor]
    # 定义shape属性，表示张量的形状
    shape: list[int]
    # 定义data_type属性，表示张量的数据类型
    data_type: DataType
    # 定义description属性，表示张量的描述信息

    # 定义load方法，返回一个Tensor对象
    def load(self) -> Tensor:
        # 调用_load方法加载数据
        ret = self._load()
        # 检查加载的数据类型是否与当前数据类型一致，如果不一致则抛出异常
        assert ret.data_type == self.data_type or (self.data_type.dtype == ret.data_type.dtype), \
                (self.data_type, ret.data_type, self.description)
        return ret

    # 定义astype方法，将LazyTensor对象转换为指定数据类型的LazyTensor对象
    def astype(self, data_type: DataType) -> LazyTensor:
        # 验证是否可以转换为指定的数据类型
        self.validate_conversion_to(data_type)

        # 定义一个内部方法load，返回转换数据类型后的Tensor对象
        def load() -> Tensor:
            return self.load().astype(data_type)
        # 返回一个新的LazyTensor对象，使用转换后的数据类型和描述信息
        return LazyTensor(load, self.shape, data_type, f'convert({data_type}) {self.description}')
    
    # 定义transposed方法，返回转置后的LazyTensor对象
    def transposed(self) -> LazyTensor:
# 定义一个方法，返回一个张量
def load() -> Tensor:
    # 调用自身的load方法加载张量
    loaded = self.load()
    # 断言加载的张量是UnquantizedTensor类型，否则抛出异常
    assert isinstance(loaded, UnquantizedTensor), f'Cannot transpose {loaded}'
    # 对加载的张量进行转置操作
    loaded.ndarray = loaded.ndarray.T
    # 返回转置后的张量
    return loaded

# 返回一个LazyTensor对象，其中包含了加载方法、形状、数据类型和描述信息
return LazyTensor(load, self.shape[::-1], self.data_type, f'transpose {self.description}')

# 验证数据类型转换是否有效
def validate_conversion_to(self, data_type: DataType) -> None:
    # 如果目标数据类型不等于当前数据类型，并且目标数据类型不在当前数据类型的有效转换列表中，则抛出异常
    if data_type != self.data_type and data_type.name not in self.data_type.valid_conversions:
        raise ValueError(f'Cannot validate conversion from {self.data_type} to {data_type}.')

# 定义一个类型别名LazyModel，表示一个字典，键为字符串，值为LazyTensor
LazyModel: TypeAlias = 'dict[str, LazyTensor]'

# 定义一个数据类ModelPlus，包含模型、路径列表和格式信息
@dataclass
class ModelPlus:
    model: LazyModel
    paths: list[Path]  # Where this was read from.
    format: Literal['ggml', 'torch', 'safetensors', 'none']
    vocab: Vocab | None  # For GGML models (which have vocab built in), the vocab.

def merge_sharded(models: list[LazyModel]) -> LazyModel:
    # Original LLaMA models have each file contain one part of each tensor.
    # Use a dict instead of a set to preserve order.
    names = {name: None for model in models for name in model}

    # 定义一个函数，用于将多个懒加载张量合并成一个
    def convert(name: str) -> LazyTensor:
        # 从每个模型中获取对应名称的懒加载张量
        lazy_tensors: list[LazyTensor] = [model[name] for model in models]
        # 如果只有一个文件包含该张量，则直接返回该张量
        if len(lazy_tensors) == 1:
            return lazy_tensors[0]
        # 如果张量是一维的，则直接返回第一个张量
        if len(lazy_tensors[0].shape) == 1:
            return lazy_tensors[0]
        # 如果张量名称满足特定条件，则返回对应的张量
        if name.startswith('tok_embeddings.') or \
           name.endswith('.attention.wo.weight') or \
           name.endswith('.feed_forward.w2.weight'):
# 如果条件成立，按列拆分
axis = 1
# 如果条件不成立，按行拆分
axis = 0
# 创建一个包含懒惰张量形状的列表
concatenated_shape = list(lazy_tensors[0].shape)
# 根据拆分轴计算拼接后的形状
concatenated_shape[axis] = sum(tensor.shape[axis] for tensor in lazy_tensors)

# 定义一个加载函数，返回未量化的张量
def load() -> UnquantizedTensor:
    # 加载未量化的张量
    ndarrays = [load_unquantized(tensor) for tensor in lazy_tensors]
    # 沿着指定轴拼接张量
    concatenated: NDArray = np.concatenate(ndarrays, axis=axis)
    # 返回未量化的张量
    return UnquantizedTensor(concatenated)
# 创建描述字符串，描述拼接后的张量
description = 'concatenated[[' + '] | ['.join(lt.description for lt in lazy_tensors) + ']]'
# 返回懒惰张量的加载函数、拼接后的形状、数据类型和描述
return LazyTensor(load, concatenated_shape, lazy_tensors[0].data_type, description)

# 返回一个字典，包含转换后的模型
return {name: convert(name) for name in names}

# 合并多个模型文件
def merge_multifile_models(models_plus: list[ModelPlus]) -> ModelPlus:
    # 获取模型文件的格式集合
    formats = set(mp.format for mp in models_plus)
    # 如果格式集合的长度不为1，抛出异常
    # assert len(formats) == 1, "different formats?"
    # 从 formats 列表中弹出最后一个元素，并赋值给 format
    format = formats.pop()
    # 从 models_plus 中的每个 mp 对象中获取 paths 属性，并将其展开成一个列表
    paths = [path for mp in models_plus for path in mp.paths]
    # 如果 models_plus 中有非 None 的 vocab，则使用第一个非 None 的 vocab
    try:
        vocab = next(mp.vocab for mp in models_plus if mp.vocab is not None)
    except StopIteration:
        vocab = None

    # 如果 models_plus 中有包含特定字符串的 model，则创建一个空的 LazyModel 字典，并将每个 mp 的 model 更新到其中
    if any("model.embed_tokens.weight" in mp.model for mp in models_plus) or \
       any("model.layers.0.fc1.weight" in mp.model for mp in models_plus):
        # Transformers models put different tensors in different files, but
        # don't split indivdual tensors between files.
        model: LazyModel = {}
        for mp in models_plus:
            model.update(mp.model)
    # 否则，将 models_plus 中的每个 model 合并成一个新的 model
    else:
        model = merge_sharded([mp.model for mp in models_plus])

    # 返回一个包含 model, paths, format, vocab 的 ModelPlus 对象
    return ModelPlus(model, paths, format, vocab)
# 对懒加载的张量进行重新排列，返回一个新的懒加载张量
def permute_lazy(lazy_tensor: LazyTensor, n_head: int, n_head_kv: int) -> LazyTensor:
    # 定义一个加载函数，返回重新排列后的张量
    def load() -> Tensor:
        return lazy_tensor.load().permute(n_head, n_head_kv)
    # 返回一个新的懒加载张量，其加载函数为上面定义的load函数
    return LazyTensor(load, lazy_tensor.shape, lazy_tensor.data_type, f'permute({n_head}, {n_head_kv}) ' + lazy_tensor.description)

# 对部分懒加载的张量进行重新排列，返回一个新的懒加载张量
def permute_part_lazy(lazy_tensor: LazyTensor, n_part: int, n_head: int, n_head_kv: int) -> LazyTensor:
    # 定义一个加载函数，返回重新排列后的张量
    def load() -> Tensor:
        return lazy_tensor.load().permute_part(n_part, n_head, n_head_kv)
    # 复制原始张量的形状，并修改第一个维度的大小
    s = lazy_tensor.shape.copy()
    s[0] = s[0] // 3
    # 返回一个新的懒加载张量，其加载函数为上面定义的load函数
    return LazyTensor(load, s, lazy_tensor.data_type, f'permute({n_head}, {n_head_kv}) ' + lazy_tensor.description)

# 对懒加载的张量进行分块，返回一个新的懒加载张量
def part_lazy(lazy_tensor: LazyTensor, n_part: int) -> LazyTensor:
    # 定义一个加载函数，返回分块后的张量
    def load() -> Tensor:
        return lazy_tensor.load().part(n_part)
    # 复制原始张量的形状，并修改第一个维度的大小
    s = lazy_tensor.shape.copy()
    s[0] = s[0] // 3
    # 返回一个新的懒加载张量，其加载函数为上面定义的load函数
    return LazyTensor(load, s, lazy_tensor.data_type, 'part ' + lazy_tensor.description)
# 定义了一个模拟 `torch.load` 功能的类，其中单个张量只在需要时才加载到内存中，而不是一次性加载所有张量。
# 在撰写本文时，PyTorch 本身无法原生实现这一功能：
# - https://github.com/pytorch/pytorch/issues/64327
# 这使我们能够在不增加内存使用量的情况下取消分片，并且方便地去除了对 PyTorch 的依赖（尽管我们仍然需要 numpy）。

# 定义了一个 LazyStorageKind 类，其中包含数据类型信息。

# 定义了一个 LazyStorage 类，其中包含加载函数、数据类型、描述等信息。
# 定义一个 LazyUnpickler 类，继承自 pickle.Unpickler
class LazyUnpickler(pickle.Unpickler):
    # 初始化方法，接受文件流、数据基本路径和 ZIP 文件对象作为参数
    def __init__(self, fp: IO[bytes], data_base_path: str, zip_file: zipfile.ZipFile):
        super().__init__(fp)
        # 设置数据基本路径和 ZIP 文件对象
        self.data_base_path = data_base_path
        self.zip_file = zip_file

    # 定义 persistent_load 方法，用于处理持久化加载
    def persistent_load(self, pid: Any) -> Any:
        # 断言 pid 的第一个元素为 'storage'
        assert pid[0] == 'storage'
        # 断言 pid 的第二个元素为 LazyStorageKind 类型
        assert isinstance(pid[1], LazyStorageKind)
        # 获取数据类型
        data_type = pid[1].data_type
        # 获取文件名的前缀
        filename_stem = pid[2]
        # 拼接完整的文件路径
        filename = f'{self.data_base_path}/{filename_stem}'
        # 获取文件信息
        info = self.zip_file.getinfo(filename)

        # 定义 load 方法，用于加载数据
        def load(offset: int, elm_count: int) -> NDArray:
            # 获取数据类型
            dtype = data_type.dtype
            # 打开 ZIP 文件中的文件
            fp = self.zip_file.open(info)
            # 定位到指定偏移量
            fp.seek(offset * dtype.itemsize)
            # 计算数据大小
            size = elm_count * dtype.itemsize
# 从文件指针中读取指定大小的数据
data = fp.read(size)
# 断言读取的数据长度与指定大小相等
assert len(data) == size
# 将读取的数据转换为指定类型的 NumPy 数组并返回
return np.frombuffer(data, dtype)

# 创建描述存储数据类型、ZIP 文件中路径和路径的描述字符串
description = f'storage data_type={data_type} path-in-zip={filename} path={self.zip_file.filename}'
# 返回一个 LazyStorage 对象，包括加载函数、类型、描述
return LazyStorage(load=load, kind=pid[1], description=description)

# 静态方法，用于重建 LazyTensor 对象
@staticmethod
def lazy_rebuild_tensor_v2(storage: Any, storage_offset: Any, size: Any, stride: Any,
                           requires_grad: Any, backward_hooks: Any, metadata: Any = None) -> LazyTensor:
    # 断言 storage 是 LazyStorage 类型
    assert isinstance(storage, LazyStorage)

    # 定义加载函数，用于加载未量化的张量
    def load() -> UnquantizedTensor:
        elm_count = stride[0] * size[0]
        return UnquantizedTensor(storage.load(storage_offset, elm_count).reshape(size))
    # 创建描述字符串，包括存储偏移和存储描述
    description = f'pickled storage_offset={storage_offset} in {storage.description}'
    # 返回一个 LazyTensor 对象，包括加载函数、大小、数据类型和描述
    return LazyTensor(load, list(size), storage.kind.data_type, description)

# 静态方法，用于根据类型和状态重建对象
@staticmethod
def rebuild_from_type_v2(func, new_type, args, state):
    # 调用指定的函数并返回结果
    return func(*args)
# 定义一个字典类型的变量CLASSES，键为元组（字符串，字符串），值为任意类型
CLASSES: dict[tuple[str, str], Any] = {
    # 使用getattr作为一种解决方案，因为mypy无法智能判断staticmethods是否具有__func__属性
    ('torch._tensor', '_rebuild_from_type_v2'): getattr(rebuild_from_type_v2, '__func__'),
    ('torch._utils', '_rebuild_tensor_v2'): getattr(lazy_rebuild_tensor_v2, '__func__'),
    ('torch', 'BFloat16Storage'): LazyStorageKind(DT_BF16),
    ('torch', 'HalfStorage'): LazyStorageKind(DT_F16),
    ('torch', 'FloatStorage'): LazyStorageKind(DT_F32),
    ('torch', 'IntStorage'): LazyStorageKind(DT_I32),
    ('torch', 'Tensor'): LazyTensor,
}

# 定义一个方法find_class，接受两个字符串类型的参数module和name，返回任意类型的值
def find_class(self, module: str, name: str) -> Any:
    # 如果module不以'torch'开头，则调用父类的find_class方法
    if not module.startswith('torch'):
        return super().find_class(module, name)
    # 否则返回CLASSES字典中对应键的值
    return self.CLASSES[(module, name)]

# 定义一个方法lazy_load_torch_file，接受一个IO类型的参数outer_fp和一个Path类型的参数path，返回ModelPlus类型的值
def lazy_load_torch_file(outer_fp: IO[bytes], path: Path) -> ModelPlus:
# 使用给定的外部文件路径创建一个 ZIP 文件对象
zf = zipfile.ZipFile(outer_fp)
# 获取 ZIP 文件中所有以 .pkl 结尾的文件路径
pickle_paths = [name for name in zf.namelist() if name.endswith('.pkl')]
# 确保只有一个以 .pkl 结尾的文件，否则抛出异常
assert len(pickle_paths) == 1, pickle_paths
# 打开第一个以 .pkl 结尾的文件
pickle_fp = zf.open(pickle_paths[0], 'r')
# 使用 LazyUnpickler 对象加载 pickle 文件中的模型
unpickler = LazyUnpickler(pickle_fp,
                          data_base_path=pickle_paths[0][:-4],
                          zip_file=zf)
model = unpickler.load()
# 将模型转换为字典形式
as_dict = dict(model.items())
# 返回包含模型字典、路径、格式和词汇表的 ModelPlus 对象
return ModelPlus(model=as_dict, paths=[path], format='torch', vocab=None)

# 从给定的文件流中延迟加载 SafeTensors 文件
def lazy_load_safetensors_file(fp: IO[bytes], path: Path) -> ModelPlus:
    # 读取头部大小并解析为 JSON 格式的字典
    header_size, = struct.unpack('<Q', fp.read(8))
    header: dict[str, dict[str, Any]] = json.loads(fp.read(header_size))
    # 使用 mmap 来避免文件偏移的竞争条件，创建内存映射
    mapped = memoryview(mmap.mmap(fp.fileno(), 0, access=mmap.ACCESS_READ))
    # 从头部大小之后的位置开始获取字节数据
    byte_buf = mapped[8 + header_size:]

    # 定义一个函数，用于将字典转换为 LazyTensor 对象
    def convert(info: dict[str, Any]) -> LazyTensor:
# 根据 info 中的数据类型获取对应的 SAFETENSORS_DATA_TYPES 中的数据类型
data_type = SAFETENSORS_DATA_TYPES[info['dtype']]
# 根据数据类型获取对应的 numpy 数据类型
numpy_dtype = data_type.dtype
# 获取数据的形状
shape: list[int] = info['shape']
# 获取数据在字节缓冲区中的起始和结束位置
begin, end = info['data_offsets']
# 确保起始位置和结束位置在合理范围内
assert 0 <= begin <= end <= len(byte_buf)
# 确保数据长度符合预期
assert end - begin == math.prod(shape) * numpy_dtype.itemsize
# 从字节缓冲区中获取数据
buf = byte_buf[begin:end]

# 定义一个加载函数，将 buf 转换为 UnquantizedTensor 类型
def load() -> UnquantizedTensor:
    return UnquantizedTensor(np.frombuffer(buf, dtype=numpy_dtype).reshape(shape))
# 生成描述信息
description = f'safetensors begin={begin} end={end} type={data_type} path={path}'
# 返回 LazyTensor 对象
return LazyTensor(load, shape, data_type, description)

# 遍历 header 中的每个元素，将其转换为 model 对象，排除 '__metadata__' 元素
model = {name: convert(info) for (name, info) in header.items() if name != '__metadata__'}
# 返回 ModelPlus 对象
return ModelPlus(model=model, paths=[path], format='safetensors', vocab=None)

# 从文件流中读取指定长度的字节数据
def must_read(fp: IO[bytes], length: int) -> bytes:
    ret = fp.read(length)
    # 如果实际读取的数据长度小于指定长度，抛出异常
    if len(ret) < length:
        raise Exception("unexpectedly reached end of file")
    return ret
    # 返回变量 ret 的值

@functools.lru_cache(maxsize=None)
# 使用 functools 模块的 lru_cache 装饰器，对函数进行缓存，maxsize=None 表示缓存大小不限制
def lazy_load_file(path: Path) -> ModelPlus:
    # 打开指定路径的文件，以二进制模式
    fp = open(path, 'rb')
    # 读取文件的前 8 个字节
    first8 = fp.read(8)
    # 将文件指针移动到文件开头
    fp.seek(0)
    if first8[:2] == b'PK':
        # 如果文件的前两个字节是 'PK'，表示是一个 ZIP 文件，即 PyTorch 格式
        return lazy_load_torch_file(fp, path)
    elif struct.unpack('<Q', first8)[0] < 16 * 1024 * 1024:
        # 如果文件的前 8 个字节表示的数字小于 16 * 1024 * 1024，可能是 safetensors 文件
        return lazy_load_safetensors_file(fp, path)
    else:
        # 否则抛出数值错误，表示未知格式
        raise ValueError(f"unknown format: {path}")


In = TypeVar('In')
Out = TypeVar('Out')
# 定义泛型变量 In 和 Out
# 定义一个并行映射函数，可以控制并发数量和最大工作线程数
def bounded_parallel_map(func: Callable[[In], Out], iterable: Iterable[In], concurrency: int, max_workers: int | None = None, use_processpool_executor: bool = False) -> Iterable[Out]:
    '''并行映射，但带有背压。如果调用者没有足够快地调用`next`，则在某个时刻停止调用`func`，而不是让结果在内存中堆积。具体来说，每个线程最多缓冲一个输出值。'''
    if concurrency < 2:
        # 如果并发数量小于2，则直接使用map函数进行映射
        yield from map(func, iterable)
        # 不会执行到这里
    iterable = iter(iterable)
    # 根据use_processpool_executor参数选择线程池或进程池执行器
    executor_class: type[ThreadPoolExecutor] | type[ProcessPoolExecutor]
    if use_processpool_executor:
        executor_class = ProcessPoolExecutor
    else:
        executor_class = ThreadPoolExecutor
    # 使用线程池或进程池执行器，根据max_workers参数设置最大工作线程数
    with executor_class(max_workers = max_workers) as executor:
        # 存储Future对象的列表
        futures: list[concurrent.futures.Future[Out]] = []
        done = False
        # 根据并发数量循环
        for _ in range(concurrency):
            try:
# 将func函数应用于iterable的下一个元素，并将结果添加到futures列表中
futures.append(executor.submit(func, next(iterable)))
# 如果StopIteration异常被捕获，将done设置为True并跳出循环
except StopIteration:
    done = True
    break

# 当futures列表不为空时，取出第一个元素的结果
while futures:
    result = futures.pop(0).result()
    # 当done为False且futures列表长度小于concurrency时，将func函数应用于iterable的下一个元素，并将结果添加到futures列表中
    while not done and len(futures) < concurrency:
        try:
            futures.append(executor.submit(func, next(iterable)))
        # 如果StopIteration异常被捕获，将done设置为True并跳出循环
        except StopIteration:
            done = True
            break
    # 产生result
    yield result

# 检查词汇表的大小是否符合要求
def check_vocab_size(params: Params, vocab: Vocab) -> None:
    # 如果params中的词汇表大小不等于vocab的词汇表大小
    if params.n_vocab != vocab.vocab_size:
        # 断言vocab是BpeVocab或SentencePieceVocab的实例
        assert isinstance(vocab, BpeVocab) or isinstance(vocab, SentencePieceVocab)
        # 如果params中的词汇表大小等于vocab的基本词汇表大小，打印警告信息
        if params.n_vocab == vocab.vocab_size_base:
            print("Ignoring added_tokens.json since model matches vocab size without it.")
# 清空已添加的特殊标记列表
vocab.added_tokens_list = []
# 将词汇表大小重置为基本词汇表大小
vocab.vocab_size = vocab.vocab_size_base
# 返回空值
return

# 构建错误信息，指出词汇表大小不匹配的问题
msg = f"Vocab size mismatch (model has {params.n_vocab}, but {vocab.fname_tokenizer}"
# 如果存在额外的标记文件，将其加入错误信息中
if vocab.fname_added_tokens is not None:
    msg += f" combined with {vocab.fname_added_tokens}"
msg += f" has {vocab.vocab_size})."
# 如果词汇表大小小于模型词汇表大小并且大于词汇表大小加上20，并且没有额外的标记文件，则说明可能缺少 added_tokens.json 文件
if vocab.vocab_size < params.n_vocab < vocab.vocab_size + 20 and vocab.fname_added_tokens is None:
    msg += f"  Most likely you are missing added_tokens.json (should be in {vocab.fname_tokenizer.parent})."
# 抛出异常，包含错误信息
raise Exception(msg)

# 定义一个输出文件类
class OutputFile:
    def __init__(self, fname_out: Path, endianess:gguf.GGUFEndian=gguf.GGUFEndian.LITTLE) -> None:
        # 创建一个 GGUFWriter 对象，用于写入输出文件
        self.gguf = gguf.GGUFWriter(fname_out, gguf.MODEL_ARCH_NAMES[ARCH], endianess=endianess)

    # 添加元数据架构
    def add_meta_arch(self, params: Params) -> None:
        name = "LLaMA"

        # TODO: 更好的逻辑来确定模型名称
# 如果上下文长度为4096，则设置名称为"LLaMA v2"
if params.n_ctx == 4096:
    name = "LLaMA v2"
# 如果模型路径不为空，则设置名称为模型路径的父目录名
elif params.path_model is not None:
    name = str(params.path_model.parent).split('/')[-1]

# 将名称添加到gguf对象中
self.gguf.add_name(name)
# 将上下文长度添加到gguf对象中
self.gguf.add_context_length(params.n_ctx)
# 将嵌入长度添加到gguf对象中
self.gguf.add_embedding_length(params.n_embd)
# 将块数量添加到gguf对象中
self.gguf.add_block_count(params.n_layer)
# 将前馈长度添加到gguf对象中
self.gguf.add_feed_forward_length(params.n_ff)
# 将绳索维度数量添加到gguf对象中
self.gguf.add_rope_dimension_count(params.n_embd // params.n_head)
# 将头数量添加到gguf对象中
self.gguf.add_head_count(params.n_head)
# 将键值头数量添加到gguf对象中
self.gguf.add_head_count_kv(params.n_head_kv)
# 将层归一化rms_eps添加到gguf对象中
self.gguf.add_layer_norm_rms_eps(params.f_norm_eps)

# 如果绳索频率基数不为空，则将其添加到gguf对象中
if params.f_rope_freq_base is not None:
    self.gguf.add_rope_freq_base(params.f_rope_freq_base)

# 如果绳索缩放类型不为空，则确保绳索缩放因子不为空
if params.rope_scaling_type:
    assert params.f_rope_scale is not None
# 添加绳索缩放类型到gguf对象
self.gguf.add_rope_scaling_type(params.rope_scaling_type)
# 添加绳索缩放因子到gguf对象
self.gguf.add_rope_scaling_factor(params.f_rope_scale)

# 如果原始上下文长度不为空，添加到gguf对象
if params.n_orig_ctx is not None:
    self.gguf.add_rope_scaling_orig_ctx_len(params.n_orig_ctx)

# 如果绳索微调不为空，添加到gguf对象
if params.rope_finetuned is not None:
    self.gguf.add_rope_scaling_finetuned(params.rope_finetuned)

# 如果文件类型不为空，添加到gguf对象
if params.ftype is not None:
    self.gguf.add_file_type(params.ftype)

# 如果预测器参数中的稀疏阈值不为空，添加到gguf对象
if params.predictor_params.sparse_threshold is not None:
    self.gguf.add_sparse_threshold(params.predictor_params.sparse_threshold)

# 添加元数据词汇到gguf对象
def add_meta_vocab(self, vocab: Vocab) -> None:
    tokens = []
    scores = []
    toktypes = []
    # 注意：`all_tokens`返回基本词汇和添加的标记
# 遍历vocab中的所有token，将text、score和toktype分别添加到对应的列表中
for text, score, toktype in vocab.all_tokens():
    tokens.append(text)
    scores.append(score)
    toktypes.append(toktype)

# 根据vocab的类型，向gguf中添加相应的tokenizer模型
if isinstance(vocab, SentencePieceVocab):
    self.gguf.add_tokenizer_model("llama")
elif isinstance(vocab, BpeVocab):
    self.gguf.add_tokenizer_model("gpt2")
else:
    raise ValueError('Unknown vocab type: Not BpeVocab or SentencePieceVocab')

# 向gguf中添加token列表、token分数和token类型
self.gguf.add_token_list(tokens)
self.gguf.add_token_scores(scores)
self.gguf.add_token_types(toktypes)

# 将特殊vocab添加到gguf中
def add_meta_special_vocab(self, svocab: gguf.SpecialVocab) -> None:
    svocab.add_to_gguf(self.gguf)

# 向gguf中添加张量信息
def add_tensor_info(self, name: str, tensor: LazyTensor) -> None:
    n_elements = int(np.prod(tensor.shape))
# 获取tensor.data_type的ggml_type属性，如果不存在则为None
raw_dtype = getattr(tensor.data_type, 'ggml_type', None)
# 获取tensor.data_type的quantized_type属性，如果不存在则为None，否则使用tensor.data_type.dtype
data_type = getattr(tensor.data_type, 'quantized_type', None) or tensor.data_type.dtype
# 计算数据类型占用的字节数
data_nbytes = tensor.data_type.elements_to_bytes(n_elements)
# 将tensor信息添加到gguf对象中
self.gguf.add_tensor_info(name, tensor.shape, data_type, data_nbytes, raw_dtype = raw_dtype)

# 将头部信息写入文件
def write_meta(self) -> None:
    self.gguf.write_header_to_file()
    self.gguf.write_kv_data_to_file()

# 将张量信息写入文件
def write_tensor_info(self) -> None:
    self.gguf.write_ti_data_to_file()

# 关闭文件
def close(self) -> None:
    self.gguf.close()

# 静态方法，仅写入词汇表信息到文件
@staticmethod
def write_vocab_only(fname_out: Path, params: Params, vocab: Vocab, svocab: gguf.SpecialVocab, endianess:gguf.GGUFEndian=gguf.GGUFEndian.LITTLE) -> None:
    # 检查词汇表大小
    check_vocab_size(params, vocab)
    # 创建输出文件对象
    of = OutputFile(fname_out, endianess=endianess)
        # 添加元数据信息
        of.add_meta_arch(params)
        of.add_meta_vocab(vocab)
        of.add_meta_special_vocab(svocab)

        # 写入元数据
        of.write_meta()

        # 关闭文件
        of.close()

    @staticmethod
    def do_item(item: tuple[str, LazyTensor]) -> tuple[DataType, NDArray]:
        # 处理每个项目，将懒惰张量加载并转换为GGML格式
        name, lazy_tensor = item
        tensor = lazy_tensor.load().to_ggml()
        return (lazy_tensor.data_type, tensor.ndarray)

    @staticmethod
    def maybe_do_quantize(item: tuple[DataType, NDArray]) -> NDArray:
        # 可能进行量化处理
        dt, arr = item
        if not isinstance(dt, QuantizedDataType):
    # 返回数组 arr
    return arr
    # 将数组 arr 量化，并返回结果
    return dt.quantize(arr)

    # 检查词汇表大小是否符合要求
    @staticmethod
    check_vocab_size(params, vocab)

    # 创建输出文件对象，并指定字节序
    of = OutputFile(fname_out, endianess=endianess)

    # 添加元数据：架构参数
    of.add_meta_arch(params)
    # 添加元数据：词汇表
    of.add_meta_vocab(vocab)
    # 添加元数据：特殊词汇表
    of.add_meta_special_vocab(svocab)

    # 添加张量信息
    for name, lazy_tensor in model.items():
        of.add_tensor_info(name, lazy_tensor)

    # 写入元数据
    of.write_meta()
    # 写入张量信息
    of.write_tensor_info()
# 定义一个函数，用于处理模型输出的文件类型
def pick_output_type(model: LazyModel, output_type_str: str | None) -> GGMLFileType:
    # 获取模型中注意力权重的数据类型
    wq_type = model[gguf.TENSOR_NAMES[gguf.MODEL_TENSOR.ATTN_Q].format(bid=0)+".weight"].data_type

# 处理模型输出的数据
# 使用并行映射函数处理模型中的每个项目，返回内部的张量数据
ndarrays_inner = bounded_parallel_map(OutputFile.do_item, model.items(), concurrency = concurrency)

# 如果文件类型是MostlyQ8_0，则使用并行映射函数处理内部的张量数据，进行量化处理
if ftype == GGMLFileType.MostlyQ8_0:
    ndarrays = bounded_parallel_map(OutputFile.maybe_do_quantize, ndarrays_inner, concurrency = concurrency, max_workers = concurrency, use_processpool_executor = True)
# 否则，使用映射函数处理内部的张量数据
else:
    ndarrays = map(OutputFile.maybe_do_quantize, ndarrays_inner)

# 记录开始时间
start = time.time()
# 遍历模型中的每个项目和对应的张量数据
for i, ((name, lazy_tensor), ndarray) in enumerate(zip(model.items(), ndarrays)):
    # 计算已经经过的时间
    elapsed = time.time() - start
    # 计算张量的大小
    size = ' x '.join(f"{dim:6d}" for dim in lazy_tensor.shape)
    # 打印输出张量的信息
    padi = len(str(len(model)))
    print(f"[{i+1:{padi}d}/{len(model)}] Writing tensor {name:38s} | size {size:16} | type {lazy_tensor.data_type.name:4} | T+{int(elapsed):4}")
    # 写入张量数据
    of.gguf.write_tensor_data(ndarray)

# 关闭输出文件
of.close()
# 如果输出类型为f32或者输出类型为空且权重类型为DT_F32，则返回AllF32
if output_type_str == "f32" or (output_type_str is None and wq_type == DT_F32):
    return GGMLFileType.AllF32
# 如果输出类型为f16或者输出类型为空且权重类型为DT_F16或DT_BF16，则返回MostlyF16
if output_type_str == "f16" or (output_type_str is None and wq_type in (DT_F16, DT_BF16)):
    return GGMLFileType.MostlyF16
# 如果输出类型为q8_0，则返回MostlyQ8_0
if output_type_str == "q8_0":
    return GGMLFileType.MostlyQ8_0

# 创建一个字典，将模型中的名称映射到对应的数据类型
name_to_type = {name: lazy_tensor.data_type for (name, lazy_tensor) in model.items()}

# 抛出异常，指出意外的类型组合
raise Exception(f"Unexpected combination of types: {name_to_type}")

# 将模型转换为指定的输出类型
def convert_to_output_type(model: LazyModel, output_type: GGMLFileType) -> LazyModel:
    return {name: tensor.astype(output_type.type_for_tensor(name, tensor))
            for (name, tensor) in model.items()}

# 将模型的名称转换为指定的参数
def convert_model_names(model: LazyModel, params: Params) -> LazyModel:
    # 创建一个张量名称映射对象
    tmap = gguf.TensorNameMap(ARCH, params.n_layer)
    # 创建一个应该跳过的模型张量集合
    should_skip: set[gguf.MODEL_TENSOR] = set(gguf.MODEL_TENSOR_SKIP.get(ARCH, []))

    # 将模型赋值给临时变量tmp
    tmp = model
# 对 HF 模型进行排列或打包一些张量，因此我们需要撤消这些操作
for i in itertools.count():
    # 如果模型中包含指定的层权重，则进行排列操作
    if f"model.layers.{i}.self_attn.q_proj.weight" in model:
        print(f"Permuting layer {i}")
        # 对指定的权重进行排列操作
        tmp[f"model.layers.{i}.self_attn.q_proj.weight"] = permute_lazy(model[f"model.layers.{i}.self_attn.q_proj.weight"], params.n_head, params.n_head)
        tmp[f"model.layers.{i}.self_attn.k_proj.weight"] = permute_lazy(model[f"model.layers.{i}.self_attn.k_proj.weight"], params.n_head, params.n_head_kv)
        #tmp[f"model.layers.{i}.self_attn.v_proj.weight"] = model[f"model.layers.{i}.self_attn.v_proj.weight"]
    # 如果模型中包含指定的层权重，则进行解包和排列操作
    elif f"model.layers.{i}.self_attn.W_pack.weight" in model:
        print(f"Unpacking and permuting layer {i}")
        # 对指定的权重进行部分解包和排列操作
        tmp[f"model.layers.{i}.self_attn.q_proj.weight"] = permute_part_lazy(model[f"model.layers.{i}.self_attn.W_pack.weight"], 0, params.n_head, params.n_head)
        tmp[f"model.layers.{i}.self_attn.k_proj.weight"] = permute_part_lazy(model[f"model.layers.{i}.self_attn.W_pack.weight"], 1, params.n_head, params.n_head_kv)
        tmp[f"model.layers.{i}.self_attn.v_proj.weight"] = part_lazy(model[f"model.layers.{i}.self_attn.W_pack.weight"], 2)
        del tmp[f"model.layers.{i}.self_attn.W_pack.weight"]
    else:
        break

# 初始化输出的 LazyModel 字典
out: LazyModel = {}
# 遍历模型中的每个张量
for name, lazy_tensor in model.items():
    # 获取张量的类型和名称
    tensor_type, name_new = tmap.get_type_and_name(name, try_suffixes = (".weight", ".bias")) or (None, None)
        # 如果新的名称为空，则抛出异常
        if name_new is None:
            raise Exception(f"Unexpected tensor name: {name}")

        # 如果张量类型在应该跳过的列表中，则打印跳过的信息并继续下一次循环
        if tensor_type in should_skip:
            print(f"skipping tensor {name_new}")
            continue

        # 打印张量的名称、新名称、数据类型和形状信息
        print(f"{name:48s} -> {name_new:40s} | {lazy_tensor.data_type.name:6s} | {lazy_tensor.shape}")
        # 将处理后的张量存入输出字典
        out[name_new] = lazy_tensor

    # 返回处理后的输出字典
    return out

# 对 LazyModel 进行后处理，转置 ffn_down 矩阵用于 Axpy 操作
def postprocess_transpose(model: LazyModel) -> LazyModel:
    """Transpose ffn_down matrices for Axpy ops."""
    # 初始化输出字典
    out: LazyModel = {}
    
    # 遍历模型中的每个张量
    for name, lazy_tensor in model.items():
        # 如果张量名称以 ".ffn_down.weight" 结尾，则将其替换为 ".ffn_down_t.weight" 并进行转置操作
        if name.endswith(".ffn_down.weight"):
            out[name.replace("ffn_down", "ffn_down_t")] = lazy_tensor.transposed()
        else:
            # 如果不是以 ".ffn_down.weight" 结尾，则不进行转置操作，直接存入输出字典
# 返回多文件模型中的第n个路径
def nth_multifile_path(path: Path, n: int) -> Path | None:
    '''给定多文件模型的任何路径（例如foo.bin.1），返回模型中的第n个路径。'''
    # 支持以下模式：
    patterns: list[tuple[str, str]] = [
        # - x.00.pth, x.01.pth, 等等
        (r'\.[0-9]{2}\.pth$', f'.{n:02}.pth'),
        # - x-00001-of-00002.bin, x-00002-of-00002.bin, 等等
        (r'-[0-9]{5}-of-(.*)$', fr'-{n:05}-of-\1'),
        # x.bin, x.bin.1, 等等
        (r'(\.[0-9]+)?$', r'\1' if n == 0 else fr'\1.{n}'),
        # x_0.pt, x_1.pt, 等等
        (r'(_[0-9]+)?\.pt$', fr'_{n}.pt'),
    ]
    # 遍历模式列表
    for regex, replacement in patterns:
# 如果路径名符合正则表达式，则将路径名中的匹配部分替换成指定的字符串
if re.search(regex, path.name):
    new_path = path.with_name(re.sub(regex, replacement, path.name))
    # 如果替换后的路径存在，则返回替换后的路径
    if new_path.exists():
        return new_path
# 如果没有匹配的路径名，则返回 None
return None

# 给定一个属于多文件模型的路径，返回该模型中所有路径的列表
def find_multifile_paths(path: Path) -> list[Path]:
    '''Given any path belonging to a multi-file model (e.g. foo.bin.1), return
    the whole list of paths in the model.
    '''
    ret: list[Path] = []
    # 使用 itertools.count() 生成无限序列，遍历多文件模型中的所有路径
    for i in itertools.count():
        nth_path = nth_multifile_path(path, i)
        # 如果获取的路径为空，则跳出循环
        if nth_path is None:
            break
        # 将获取的路径添加到结果列表中
        ret.append(nth_path)
    # 如果结果列表为空，则说明没有匹配的路径
    if not ret:
        # No matches.  This should only happen if the file was named, e.g.,
        # foo.0, and there was no file named foo.  Oh well, try to process it
        # 如果路径是一个目录，则加载模型文件
        return [path]
    return ret


def load_some_model(path: Path) -> ModelPlus:
    '''Load a model of any supported format.'''
    # 友好地接受文件或目录作为输入：
    if path.is_dir():
        # 首先检查是否是一组 safetensors 文件
        globs = ["model-00001-of-*.safetensors", "model.safetensors"]
        files = [file for glob in globs for file in path.glob(glob)]
        if not files:
            # 也尝试使用 PyTorch 的模式，但优先级较低
            globs = ["consolidated.00.pth", "pytorch_model-00001-of-*.bin", "*.pt", "pytorch_model.bin"]
            files = [file for glob in globs for file in path.glob(glob)]
        if not files:
            raise Exception(f"Can't find model in directory {path}")
        if len(files) > 1:
            raise Exception(f"Found multiple models in {path}, not sure which to pick: {files}")
# 获取文件列表中的第一个文件路径
path = files[0]

# 查找多文件路径
paths = find_multifile_paths(path)
# 创建一个空的 ModelPlus 列表
models_plus: list[ModelPlus] = []
# 遍历路径列表
for path in paths:
    # 打印加载的模型文件路径
    print(f"Loading model file {path}")
    # 惰性加载文件并将其添加到模型列表中
    models_plus.append(lazy_load_file(path))

# 合并多文件模型
model_plus = merge_multifile_models(models_plus)
# 返回合并后的模型
return model_plus

# 加载稀疏注意力的 MLP 模型
def load_mlp_model(path: Path) -> ModelPlus:
    # 确保路径是一个目录
    assert path.is_dir(), f"MLP model path {path} is not a directory"
    
    # 获取第一个模型文件路径
    first_model_path = path / "model_0.pt"
    # 确保第一个模型文件存在
    assert first_model_path.resolve(), f"MLP model path {path} does not contain model_0.pt"

    # 查找多文件路径
    model_paths = find_multifile_paths(first_model_path)
    # 创建一个空的 ModelPlus 列表
    models_plus: list[ModelPlus] = []
    # 遍历模型路径列表
    for model_path in model_paths:
        # 从模型路径中找到模型层数字
        model_layer = int(re.search(r'model_(\d+).pt', str(model_path)).group(1))
        # 打印加载的MLP模型文件路径
        print(f"Loading MLP model file {model_path}")
        # 懒加载模型文件
        mlp_model = lazy_load_file(model_path)
        # 将模型中的层数据存储到model.layers中
        mlp_model.model = {f"model.layers.{model_layer}.{name}": tensor for name, tensor in mlp_model.model.items()}
        # 将加载的模型添加到models_plus列表中
        models_plus.append(mlp_model)

    # 合并多个模型文件
    return merge_multifile_models(models_plus)


def load_vocab(path: Path, vocabtype: str | None) -> Vocab:
    # 友好地接受文件或目录作为输入路径
    # 如果是目录，可能是模型目录，tokenizer.model可能在其父目录中
    if path.is_dir():
        # 设置默认的词汇文件名为tokenizer.model
        vocab_file = "tokenizer.model"
        # 如果词汇类型是'bpe'，则使用vocab.json作为词汇文件名
        if vocabtype == 'bpe':
            vocab_file = "vocab.json"
        # 构建词汇文件的完整路径
        path2 = path / vocab_file
# 使用 `.parent` 而不是 /.. 来更好地处理符号链接的情况。
path3 = path.parent / vocab_file
# 如果 path2 存在，则将 path 赋值为 path2
if path2.exists():
    path = path2
# 如果 path3 存在，则将 path 赋值为 path3
elif path3.exists():
    path = path3
# 否则，抛出文件未找到的错误
else:
    raise FileNotFoundError(
        f"Could not find {vocab_file} in {path} or its parent; "
        "if it's in another directory, pass the directory as --vocab-dir")

# 打印加载的词汇文件路径和类型
print(f"Loading vocab file '{path}', type '{vocabtype}'")

# 创建 added_tokens_path 变量，指向 path 的父目录下的 added_tokens.json 文件
added_tokens_path = path.parent / "added_tokens.json"
# 如果 vocabtype 是 "bpe"，则返回 BpeVocab 对象
if vocabtype == "bpe":
    return BpeVocab(path, added_tokens_path if added_tokens_path.exists() else None)
# 如果 vocabtype 是 "spm"，则返回 SentencePieceVocab 对象
elif vocabtype == "spm":
    return SentencePieceVocab(path, added_tokens_path if added_tokens_path.exists() else None)
# 否则，抛出不支持的词汇类型错误
else:
    raise ValueError(f"Unsupported vocabulary type {vocabtype}")
# 根据模型路径列表和文件类型返回默认的输出文件路径
def default_outfile(model_paths: list[Path], file_type: GGMLFileType) -> Path:
    # 根据文件类型选择对应的文件名字符串
    namestr = {
        GGMLFileType.AllF32:    "f32",
        GGMLFileType.MostlyF16: "f16",
        GGMLFileType.MostlyQ8_0:"q8_0",
    }[file_type]
    # 根据选择的文件名字符串和模型路径创建输出文件路径
    ret = model_paths[0].parent / f"ggml-model-{namestr}.gguf"
    # 如果输出文件路径已经存在于模型路径中，则报错并退出程序
    if ret in model_paths:
        sys.stderr.write(
            f"Error: Default output path ({ret}) would overwrite the input. "
            "Please explicitly specify a path using --outfile.\n")
        sys.exit(1)
    # 返回输出文件路径
    return ret

# 执行模型转储操作
def do_dump_model(model_plus: ModelPlus) -> None:
    # 打印模型路径和格式信息
    print(f"model_plus.paths = {model_plus.paths!r}")
    print(f"model_plus.format = {model_plus.format!r}")
# 打印 model_plus.vocab 的值
print(f"model_plus.vocab = {model_plus.vocab!r}")
# 遍历 model_plus.model 中的每个元素，打印其名称、形状、数据类型和描述
for name, lazy_tensor in model_plus.model.items():
    print(f"{name}: shape={lazy_tensor.shape} type={lazy_tensor.data_type}; {lazy_tensor.description}")

# 主函数，接受参数列表作为输入，不返回任何结果
def main(args_in: list[str] | None = None) -> None:
    # 输出格式的选择，可以是 "f32"、"f16" 或 "q8_0"
    output_choices = ["f32", "f16"]
    # 如果系统是小端序，添加 "q8_0" 作为输出格式的选择
    if np.uint32(1) == np.uint32(1).newbyteorder("<"):
        output_choices.append("q8_0")
    # 创建参数解析器
    parser = argparse.ArgumentParser(description="Convert a LLaMa model to a GGML compatible file")
    # 添加参数选项
    parser.add_argument("--dump",        action="store_true",    help="don't convert, just show what's in the model")
    parser.add_argument("--dump-single", action="store_true",    help="don't convert, just show what's in a single model file")
    parser.add_argument("--vocab-only",  action="store_true",    help="extract only the vocab")
    parser.add_argument("--outtype",     choices=output_choices, help="output format - note: q8_0 may be very slow (default: f16 or f32 based on input)", default="f16")
    parser.add_argument("--vocab-dir",   type=Path,              help="directory containing tokenizer.model, if separate from model file")
    parser.add_argument("--outfile",     type=Path,              help="path to write to; default: based on input")
    parser.add_argument("--ctx",         type=int,               help="model training context (default: based on input)")
    parser.add_argument("--concurrency", type=int,               help=f"concurrency used for conversion (default: {DEFAULT_CONCURRENCY})", default = DEFAULT_CONCURRENCY)
    parser.add_argument("--bigendian",   action="store_true",    help="model is executed on big endian machine")
# 添加命令行参数--vocabtype，可选值为"spm"和"bpe"，默认值为"spm"
parser.add_argument("--vocabtype", choices=["spm", "bpe"], help="vocab format (default: spm)", default="spm")
# 添加命令行参数model，类型为Path，帮助信息为包含模型文件的目录，或者模型文件本身(*.pth, *.pt, *.bin, *.safetensors)
parser.add_argument("model", type=Path, help="directory containing model file, or model file itself (*.pth, *.pt, *.bin, *.safetensors)")
# 添加命令行参数mlp_model，类型为Path，帮助信息为稀疏注意力的MLP模型
parser.add_argument("mlp_model", type=Path, help="MLP model for sparse attention")

# 解析命令行参数
args = parser.parse_args(args_in)

# 尝试打开模型目录下的config.json文件，读取其中的内容
try:
    with open(args.model / "config.json", "r", encoding="utf-8") as f:
        hf_config = json.load(f)
    # 如果模型类型不是"llama"
    if model_type := hf_config.get("model_type") != "llama":
        # 调用另一个脚本来转换其他模型
        print(f"Model architecture {model_type} is not supported by this `convert.py`. Trying with `convert-hf-to-powerinfer-gguf.py`...")
        script_path = Path(__file__).resolve().parent / "convert-hf-to-powerinfer-gguf.py"
        subprocess.run(["python3", str(script_path.absolute())] + sys.argv[1:])
        return
# 如果找不到config.json文件
except FileNotFoundError:
    print("Could not find config.json under the original model directory. ", file=sys.stderr)
    sys.exit(1)

# 如果命令行参数中包含dump_single
if args.dump_single:
# 从文件中延迟加载模型
model_plus = lazy_load_file(args.model)
# 对加载的模型执行转储操作
do_dump_model(model_plus)
# 返回

# 如果不仅仅是词汇表，则加载一些模型
if not args.vocab_only:
    model_plus = load_some_model(args.model)
    # 加载参数
    params = Params.load(model_plus)
    # 加载MLP预测器模型
    mlp_predictor_plus = load_mlp_model(args.mlp_model)
    # 加载预测器参数
    params.predictor_params = PredictorParams.load(mlp_predictor_plus)
    # 合并多文件模型
    model_plus = merge_multifile_models([model_plus, mlp_predictor_plus])
# 如果只是词汇表
else:
    # 创建一个空的模型
    model_plus = ModelPlus(model = {}, paths = [args.model / 'dummy'], format = 'none', vocab = None)
    # 加载参数
    params = Params.load(model_plus)

# 如果需要转储
if args.dump:
    # 对加载的模型执行转储操作
    do_dump_model(model_plus)
    # 返回
    return
# 设置字节顺序为小端
endianess = gguf.GGUFEndian.LITTLE
# 如果参数中指定了大端字节顺序
if args.bigendian:
    # 设置字节顺序为大端
    endianess = gguf.GGUFEndian.BIG
    # 如果上下文大小为-1，则检查是否在参数中指定了上下文大小，如果没有则抛出异常
    if params.n_ctx == -1:
        if args.ctx is None:
            raise Exception("The model doesn't have a context size, and you didn't specify one with --ctx\n"
                            "Please specify one with --ctx:\n"
                            " - LLaMA v1: --ctx 2048\n"
                            " - LLaMA v2: --ctx 4096\n")
        # 将参数中指定的上下文大小赋值给n_ctx
        params.n_ctx = args.ctx

    # 如果指定了输出类型，则根据参数设置文件类型
    if args.outtype:
        params.ftype = {
            "f32": GGMLFileType.AllF32,
            "f16": GGMLFileType.MostlyF16,
            "q8_0": GGMLFileType.MostlyQ8_0,
        }[args.outtype]

    # 打印参数信息
    print(f"params = {params}")

    # 声明vocab变量为Vocab类型
    vocab: Vocab
    # 如果指定了仅使用词汇表，则执行以下操作
    if args.vocab_only:
# 如果没有指定输出文件，则抛出数值错误
if not args.outfile:
    raise ValueError("need --outfile if using --vocab-only")
# FIXME: 尝试以某种方式尊重词汇目录？
# 根据指定的词汇目录或模型加载词汇
vocab = load_vocab(args.vocab_dir or args.model, args.vocabtype)
# 创建特殊词汇对象
special_vocab = gguf.SpecialVocab(model_plus.paths[0].parent,
    load_merges = args.vocabtype == 'bpe',
    n_vocab = vocab.vocab_size)
# 将词汇写入指定的输出文件
outfile = args.outfile
OutputFile.write_vocab_only(outfile, params, vocab, special_vocab)
# 打印写入的文件名
print(f"Wrote {outfile}")
# 返回结果
return

# 如果模型中包含词汇并且没有指定词汇目录，则使用模型中的词汇
if model_plus.vocab is not None and args.vocab_dir is None:
    vocab = model_plus.vocab
# 否则，根据指定的词汇目录或模型加载词汇
else:
    vocab_dir = args.vocab_dir if args.vocab_dir else model_plus.paths[0].parent
    vocab = load_vocab(vocab_dir, args.vocabtype)
# FIXME: 尝试以某种方式尊重词汇目录？
# 创建特殊词汇对象
special_vocab = gguf.SpecialVocab(model_plus.paths[0].parent,
    load_merges = args.vocabtype == 'bpe',
```
# 以上是对给定代码的注释。
    # 初始化词汇表大小
    n_vocab = vocab.vocab_size)

    # 获取模型
    model   = model_plus.model
    # 转换模型名称
    model   = convert_model_names(model, params)
    # 后处理转置
    model   = postprocess_transpose(model)
    # 选择输出类型
    ftype   = pick_output_type(model, args.outtype)
    # 将模型转换为指定输出类型
    model   = convert_to_output_type(model, ftype)
    # 输出文件路径
    outfile = args.outfile or default_outfile(model_plus.paths, ftype)

    # 设置参数中的输出类型
    params.ftype = ftype
    # 打印输出文件名和格式
    print(f"Writing {outfile}, format {ftype}")

    # 写入所有内容到输出文件
    OutputFile.write_all(outfile, ftype, params, model, vocab, special_vocab, concurrency = args.concurrency, endianess=endianess)
    # 打印已写入的文件名
    print(f"Wrote {outfile}")

    # 后处理：写入另一个唯一的文件头，以区分原始的 GGUF 文件
    with open(outfile, "r+b") as fout:
        # 定义一个特殊的标识符
        POWERINFER_MAGIC = int.from_bytes(b"PWRI", "little")
        # 在文件开头写入特殊标识符
        fout.write(struct.pack("<I", POWERINFER_MAGIC))
# 如果当前脚本被直接执行，则调用 main() 函数。
```