# `PowerInfer\convert.py`

```
#!/usr/bin/env python3
# 指定脚本的解释器为 Python 3

from __future__ import annotations
# 导入未来版本的注解特性

import argparse
import concurrent.futures
import dataclasses
import enum
import faulthandler
import functools
import itertools
import json
import math
import mmap
import pickle
import re
import signal
import struct
import subprocess
import sys
import time
import zipfile
# 导入所需的标准库和第三方库

from abc import ABCMeta, abstractmethod
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor
from dataclasses import dataclass
from pathlib import Path
from typing import IO, TYPE_CHECKING, Any, Callable, Iterable, Literal, TypeVar
# 导入需要的模块和类型

import numpy as np
# 导入 NumPy 库

from sentencepiece import SentencePieceProcessor
# 导入 SentencePieceProcessor 类

import os
# 导入 os 模块
if 'NO_LOCAL_GGUF' not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))
# 如果环境变量中没有设置 NO_LOCAL_GGUF，则将 gguf-py 目录添加到模块搜索路径中
import gguf
# 导入 gguf 模块

if TYPE_CHECKING:
    from typing import TypeAlias
# 如果是类型检查阶段，则导入 TypeAlias 类型

if hasattr(faulthandler, 'register') and hasattr(signal, 'SIGUSR1'):
    faulthandler.register(signal.SIGUSR1)
# 如果 faulthandler 模块中有 register 方法，并且 signal 模块中有 SIGUSR1 常量，则注册 SIGUSR1 信号

NDArray: TypeAlias = 'np.ndarray[Any, Any]'
# 定义 TypeAlias 类型别名为 'np.ndarray[Any, Any]'

ARCH = gguf.MODEL_ARCH.LLAMA
# 设置 ARCH 变量为 gguf 模块中的 MODEL_ARCH.LLAMA 常量

DEFAULT_CONCURRENCY = 8
# 设置 DEFAULT_CONCURRENCY 变量为 8

#
# data types
#

@dataclass(frozen=True)
# 使用 dataclass 装饰器，创建一个不可变的数据类
class DataType:
    name: str
    dtype: np.dtype[Any]
    valid_conversions: list[str]

    def elements_to_bytes(self, n_elements: int) -> int:
        return n_elements * self.dtype.itemsize
# 定义 DataType 类，包含名称、数据类型、有效转换列表和将元素转换为字节的方法

@dataclass(frozen=True)
# 使用 dataclass 装饰器，创建一个不可变的数据类
class UnquantizedDataType(DataType):
    pass
# 定义 UnquantizedDataType 类，继承自 DataType 类

DT_F16  = UnquantizedDataType('F16', dtype = np.dtype(np.float16), valid_conversions = ['F32', 'Q8_0'])
# 创建 F16 数据类型对象
DT_F32  = UnquantizedDataType('F32', dtype = np.dtype(np.float32), valid_conversions = ['F16', 'Q8_0'])
# 创建 F32 数据类型对象
DT_I32  = UnquantizedDataType('I32', dtype = np.dtype(np.int16), valid_conversions = [])
# 创建 I32 数据类型对象
DT_BF16 = UnquantizedDataType('BF16', dtype = np.dtype(np.uint16), valid_conversions = ['F32', 'F16', 'Q8_0'])
# 创建 BF16 数据类型对象

@dataclass(frozen=True)
# 使用 dataclass 装饰器，创建一个不可变的数据类
class QuantizedDataType(DataType):
    block_size: int
    quantized_dtype: np.dtype[Any]
    ggml_type: gguf.GGMLQuantizationType
# 定义 QuantizedDataType 类，继承自 DataType 类，包含块大小、量化数据类型和 GGML 量化类型
    # 对数组进行量化处理，返回量化后的数组
    def quantize(self, arr: NDArray) -> NDArray:
        # 抛出未实现错误，提示该量化方法未实现
        raise NotImplementedError(f'Quantization for {self.name} not implemented')
    
    # 将元素数量转换为字节数，返回转换后的字节数
    def elements_to_bytes(self, n_elements: int) -> int:
        # 断言元素数量能够整除块大小，否则抛出错误
        assert n_elements % self.block_size == 0, f'Invalid number of elements {n_elements} for {self.name} with block size {self.block_size}'
        # 返回量化后的数据类型的字节数乘以块的数量
        return self.quantized_dtype.itemsize * (n_elements // self.block_size)
# 定义一个名为 Q8_0QuantizedDataType 的类，它是 QuantizedDataType 的子类，并且是不可变的
@dataclass(frozen=True)
class Q8_0QuantizedDataType(QuantizedDataType):
    # 在 Python 中进行 Mini Q8_0 量化！
    def quantize(self, arr: NDArray) -> NDArray:
        # 断言数组大小能够整除块大小且不为零
        assert arr.size % self.block_size == 0 and arr.size != 0, f'Bad array size {arr.size}'
        # 断言数组的数据类型为 np.float32
        assert arr.dtype == np.float32, f'Bad array type {arr.dtype}'
        # 计算块的数量
        n_blocks = arr.size // self.block_size
        # 将数组重塑为块
        blocks = arr.reshape((n_blocks, self.block_size))
        # 由 @Cebtenzzre 贡献的块量化的更快实现
        def quantize_blocks_q8_0(blocks: NDArray) -> Iterable[tuple[Any, Any]]:
            # 计算每个块的最大值，并进行块内除法
            d = abs(blocks).max(axis = 1) / np.float32(127)
            with np.errstate(divide = 'ignore'):
                qs = (blocks / d[:, None]).round()
            qs[d == 0] = 0
            yield from zip(d, qs)
        # 从块量化生成器中创建数组
        return np.fromiter(quantize_blocks_q8_0(blocks), count = n_blocks, dtype = self.quantized_dtype)

# 创建一个名为 DT_Q8_0 的 Q8_0QuantizedDataType 实例
DT_Q8_0 = Q8_0QuantizedDataType('Q8_0',
    dtype = np.dtype(np.float32), valid_conversions = [],
    ggml_type = gguf.GGMLQuantizationType.Q8_0, block_size = 32,
    quantized_dtype = np.dtype([('d', '<f2'), ('qs', 'i1', (32,))]))

# 跳过量化类型，因为它们可能也映射到 np.float32
NUMPY_TYPE_TO_DATA_TYPE: dict[np.dtype[Any], DataType] = {}
for dt in (DT_BF16, DT_F16, DT_F32, DT_I32):
    if dt.dtype in NUMPY_TYPE_TO_DATA_TYPE:
        raise ValueError(f'Invalid duplicate data type {dt}')
    NUMPY_TYPE_TO_DATA_TYPE[dt.dtype] = dt

# 创建一个名为 SAFETENSORS_DATA_TYPES 的字典，将字符串映射到 DataType
SAFETENSORS_DATA_TYPES: dict[str, DataType] = {
    'BF16': DT_BF16,
    'F16': DT_F16,
    'F32': DT_F32,
    'I32': DT_I32,
}

# TODO: 与 `llama_ftype` 匹配
# TODO: 重命名为 LLAMAFileType
# TODO: 移动到 `gguf.py`
# 创建一个名为 GGMLFileType 的枚举类
class GGMLFileType(enum.IntEnum):
    AllF32     = 0
    MostlyF16  = 1  # 除了 1 维张量
    MostlyQ8_0 = 7  # 除了 1 维张量
    # 定义一个方法，用于确定张量的数据类型
    def type_for_tensor(self, name: str, tensor: LazyTensor) -> DataType:
        # 从 GGML_FILE_TYPE_TO_DATA_TYPE 字典中获取数据类型
        dt = GGML_FILE_TYPE_TO_DATA_TYPE.get(self)
        # 如果获取不到数据类型，则抛出数值错误
        if dt is None:
            raise ValueError(self)
        # 如果张量的维度大于1，则返回获取到的数据类型，否则返回 F32 数据类型
        # 1D tensors are always F32.
        return dt if len(tensor.shape) > 1 else DT_F32
# 定义一个字典，将 GGML 文件类型映射到数据类型
GGML_FILE_TYPE_TO_DATA_TYPE: dict[GGMLFileType, DataType] = {
    GGMLFileType.AllF32    : DT_F32,
    GGMLFileType.MostlyF16 : DT_F16,
    GGMLFileType.MostlyQ8_0: DT_Q8_0,
}

#
# hparams loading
#

# 定义一个数据类，用于存储预测器参数
@dataclass
class PredictorParams:
    sparse_threshold: float | None = None

    # 从 JSON 文件中加载预测器参数
    @staticmethod
    def loadPredictorJson(model: LazyModel, config_path: Path) -> PredictorParams:
        config = json.load(open(config_path))
        return PredictorParams(
            sparse_threshold = config.get("sparse_threshold"),
        )

    # 加载预测器参数
    @staticmethod
    def load(model_plus: ModelPlus) -> PredictorParams:
        config_path   = model_plus.paths[0].parent / "config.json"

        # 如果配置文件存在，则加载预测器参数
        if config_path.exists():
            params = PredictorParams.loadPredictorJson(model_plus.model, config_path)
        # 否则，使用默认参数
        else:
            params = PredictorParams()

        return params

# 定义一个数据类，用于存储模型参数
@dataclass
class Params:
    n_vocab:    int
    n_embd:     int
    n_layer:    int
    n_ctx:      int
    n_ff:       int
    n_head:     int
    n_head_kv:  int
    f_norm_eps: float

    rope_scaling_type: gguf.RopeScalingType | None = None
    f_rope_freq_base: float | None = None
    f_rope_scale: float | None = None
    n_orig_ctx: int | None = None
    rope_finetuned: bool | None = None

    ftype: GGMLFileType | None = None

    # 模型文件所在目录的路径
    path_model: Path | None = None

    # MLP 预测器参数
    predictor_params: PredictorParams = dataclasses.field(default_factory=PredictorParams)

    @staticmethod
    # 定义一个函数，用于猜测模型的参数
    def guessed(model: LazyModel) -> Params:
        # 尝试使用transformer的命名方式来获取词汇量和嵌入维度
        n_vocab, n_embd = model["model.embed_tokens.weight"].shape if "model.embed_tokens.weight" in model else model["tok_embeddings.weight"].shape

        # 尝试使用transformer的命名方式来获取层数
        if "model.layers.0.self_attn.q_proj.weight" in model:
            n_layer=next(i for i in itertools.count() if f"model.layers.{i}.self_attn.q_proj.weight" not in model)
        elif "model.layers.0.self_attn.W_pack.weight" in model:   # 接下来尝试使用baichuan的命名方式
            n_layer=next(i for i in itertools.count() if f"model.layers.{i}.self_attn.W_pack.weight" not in model)
        else:
            n_layer=next(i for i in itertools.count() if f"layers.{i}.attention.wq.weight" not in model)

        # 如果无法猜测出层数，则抛出异常
        if n_layer < 1:
            raise Exception("failed to guess 'n_layer'. This model is unknown or unsupported.\n"
                            "Suggestion: provide 'config.json' of the model in the same directory containing model files.")

        # 猜测头数和多头注意力的维度
        n_head = n_embd // 128 # guessed
        n_mult = 256           # guessed

        # TODO: 验证这一部分代码
        # 计算前馈神经网络的维度
        n_ff = int(2 * (4 * n_embd) / 3)
        n_ff = n_mult * ((n_ff + n_mult - 1) // n_mult)

        # 返回参数对象
        return Params(
            n_vocab    = n_vocab,
            n_embd     = n_embd,
            n_layer    = n_layer,
            n_ctx      = -1,
            n_ff       = n_ff,
            n_head     = n_head,
            n_head_kv  = n_head,
            f_norm_eps = 1e-5,
        )

    @staticmethod
    # LLaMA v2 70B params.json
    # {"dim": 8192, "multiple_of": 4096, "ffn_dim_multiplier": 1.3, "n_heads": 64, "n_kv_heads": 8, "n_layers": 80, "norm_eps": 1e-05, "vocab_size": -1}
    @staticmethod
    # 从给定的模型和配置文件路径加载原始参数的 JSON 数据，并返回 Params 对象
    def loadOriginalParamsJson(model: LazyModel, config_path: Path) -> Params:
        # 读取配置文件的 JSON 数据
        config = json.load(open(config_path))

        # hack to determine LLaMA v1 vs v2 vs CodeLlama
        # 判断配置文件中的特定参数值，以确定是 LLaMA v1 还是 v2 还是 CodeLlama
        if config.get("rope_theta") == 1000000:
            # CodeLlama
            n_ctx = 16384
        elif config["norm_eps"] == 1e-05:
            # LLaMA v2
            n_ctx = 4096
        else:
            # LLaMA v1
            n_ctx = 2048

        # 返回 Params 对象，包含从配置文件中读取的参数
        return Params(
            n_vocab          = config.get("vocab_size", model["tok_embeddings.weight"].shape[0]),
            n_embd           = config["dim"],
            n_layer          = config["n_layers"],
            n_ctx            = n_ctx,
            n_ff             = model["layers.0.feed_forward.w1.weight"].shape[0],
            n_head           = (n_head := config["n_heads"]),
            n_head_kv        = config.get("n_kv_heads", n_head),
            f_norm_eps       = config["norm_eps"],
            f_rope_freq_base = config.get("rope_theta"),
        )

    # 从给定的 ModelPlus 对象加载 Params 对象
    @staticmethod
    def load(model_plus: ModelPlus) -> Params:
        # 构建 HF 配置文件路径和原始配置文件路径
        hf_config_path   = model_plus.paths[0].parent / "config.json"
        orig_config_path = model_plus.paths[0].parent / "params.json"

        # 根据不同情况加载 Params 对象
        if hf_config_path.exists():
            # 如果 HF 配置文件存在，则使用 HFTransformerJson 加载 Params 对象
            params = Params.loadHFTransformerJson(model_plus.model, hf_config_path)
        elif orig_config_path.exists():
            # 如果原始配置文件存在，则使用 OriginalParamsJson 加载 Params 对象
            params = Params.loadOriginalParamsJson(model_plus.model, orig_config_path)
        elif model_plus.format != 'none':
            # 如果模型格式不为 'none'，则使用 guessed 方法猜测 Params 对象
            params = Params.guessed(model_plus.model)
        else:
            # 如果以上条件都不满足，则抛出数值错误
            raise ValueError('Cannot guess params when model format is none')

        # 设置 Params 对象的模型路径
        params.path_model = model_plus.paths[0].parent

        # 返回加载的 Params 对象
        return params
# 定义 BpeVocab 类
class BpeVocab:
    # 初始化方法，接受分词器文件名和额外添加的标记文件名作为参数
    def __init__(self, fname_tokenizer: Path, fname_added_tokens: Path | None) -> None:
        # 从分词器文件中加载 BPE 分词器的配置
        self.bpe_tokenizer = json.loads(open(str(fname_tokenizer), encoding="utf-8").read())
        added_tokens: dict[str, int]
        # 如果存在额外添加的标记文件
        if fname_added_tokens is not None:
            # 加载额外添加的标记
            added_tokens = json.load(open(fname_added_tokens, encoding="utf-8"))
        else:
            # 尝试从 tokenizer.json 文件中查找额外添加的标记
            tokenizer_json_file = fname_tokenizer.parent / 'tokenizer.json'
            # 如果文件不存在，则设置额外添加的标记为空字典
            if not tokenizer_json_file.is_file():
                added_tokens = {}
            else:
                # 从 tokenizer.json 文件中加载额外添加的标记
                tokenizer_json = json.load(open(tokenizer_json_file, encoding="utf-8"))
                # 从加载的标记中筛选出不在 BPE 分词器中的额外添加的标记
                added_tokens = dict(
                    (item['content'], item['id'])
                    for item in tokenizer_json.get('added_tokens', [])
                    if item['content'] not in self.bpe_tokenizer
                )

        # 计算词汇表的大小
        vocab_size: int = len(self.bpe_tokenizer)
        # 预期的额外标记 ID 序列
        expected_ids    = list(range(vocab_size, vocab_size + len(added_tokens)))
        # 实际的额外标记 ID 序列
        actual_ids      = sorted(added_tokens.values())
        # 检查预期的额外标记 ID 序列和实际的额外标记 ID 序列是否一致
        if expected_ids != actual_ids:
            expected_end_id = vocab_size + len(actual_ids) - 1
            # 如果不一致，则抛出异常
            raise Exception(f"Expected the {len(actual_ids)} added token ID(s) to be sequential in the range {vocab_size} - {expected_end_id}; got {actual_ids}")

        # 对额外标记按照 ID 进行排序，得到额外标记列表
        items = sorted(added_tokens.items(), key=lambda text_idx: text_idx[1])
        self.added_tokens_list    = [text for (text, idx) in items]
        self.vocab_size_base: int = vocab_size
        self.vocab_size: int      = self.vocab_size_base + len(self.added_tokens_list)
        self.fname_tokenizer      = fname_tokenizer
        self.fname_added_tokens   = fname_added_tokens
    # 返回一个可迭代的元组，包含字节、分数和标记类型
    def bpe_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
        # 获取BPE分词器
        tokenizer = self.bpe_tokenizer
        # 导入GPT2模型的tokenization_gpt2
        from transformers.models.gpt2 import tokenization_gpt2
        # 创建一个反向词汇表，将编码后的标记映射回原始标记
        reverse_vocab = {id: encoded_tok for encoded_tok, id in tokenizer.items()}
    
        # 遍历BPE分词器
        for i, _ in enumerate(tokenizer):
            # 生成反向词汇表中的标记、分数和标记类型为NORMAL的元组
            yield reverse_vocab[i], 0.0, gguf.TokenType.NORMAL
    
    # 返回一个可迭代的元组，包含字节、分数和标记类型
    def added_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
        # 遍历已添加的标记列表
        for text in self.added_tokens_list:
            # 设置分数为-1000.0
            score = -1000.0
            # 生成文本编码后的字节、分数和标记类型为CONTROL的元组
            yield text.encode("utf-8"), score, gguf.TokenType.CONTROL
    
    # 返回一个可迭代的元组，包含字节、分数和标记类型
    def all_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
        # 从bpe_tokens()生成器中获取元组
        yield from self.bpe_tokens()
        # 从added_tokens()生成器中获取元组
        yield from self.added_tokens()
    
    # 返回描述BpeVocab对象的字符串
    def __repr__(self) -> str:
        return f"<BpeVocab with {self.vocab_size_base} base tokens and {len(self.added_tokens_list)} added tokens>"
class SentencePieceVocab:
    # 初始化函数，接受分词器文件名和额外标记文件名作为参数
    def __init__(self, fname_tokenizer: Path, fname_added_tokens: Path | None) -> None:
        # 使用分词器文件名创建 SentencePieceProcessor 对象
        self.sentencepiece_tokenizer = SentencePieceProcessor(str(fname_tokenizer))
        # 声明一个空的字典 added_tokens
        added_tokens: dict[str, int]
        # 如果存在额外标记文件名，则从中加载标记到 added_tokens 字典中
        if fname_added_tokens is not None:
            added_tokens = json.load(open(fname_added_tokens, encoding="utf-8"))
        else:
            # 否则将 added_tokens 初始化为空字典
            added_tokens = {}

        # 获取分词器的词汇量大小
        vocab_size: int = self.sentencepiece_tokenizer.vocab_size()

        # 从 added_tokens 中筛选出新添加的标记，并将其存储在 new_tokens 中
        new_tokens       = {id: piece for piece, id in added_tokens.items() if id >= vocab_size}
        # 生成期望的新标记 ID 列表
        expected_new_ids = list(range(vocab_size, vocab_size + len(new_tokens)))
        # 获取实际的新标记 ID 列表
        actual_new_ids   = sorted(new_tokens.keys())

        # 如果期望的新标记 ID 列表与实际的新标记 ID 列表不一致，则抛出数值错误
        if expected_new_ids != actual_new_ids:
            raise ValueError(f"Expected new token IDs {expected_new_ids} to be sequential; got {actual_new_ids}")

        # 存储被添加到基础词汇表中的标记
        self.added_tokens_list  = [new_tokens[id] for id in actual_new_ids]
        # 存储基础词汇表的大小
        self.vocab_size_base    = vocab_size
        # 存储总词汇表的大小
        self.vocab_size         = self.vocab_size_base + len(self.added_tokens_list)
        # 存储分词器文件名
        self.fname_tokenizer    = fname_tokenizer
        # 存储额外标记文件名
        self.fname_added_tokens = fname_added_tokens
    # 生成 sentencepiece_tokens 方法，返回一个元组的可迭代对象，包含字节、分数和标记类型
    def sentencepiece_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
        # 获取 sentencepiece_tokenizer 对象
        tokenizer = self.sentencepiece_tokenizer
        # 遍历词汇表大小的范围
        for i in range(tokenizer.vocab_size()):
            # 获取词汇表中 id 对应的词
            piece = tokenizer.id_to_piece(i)
            # 将词转换为字节
            text: bytes = piece.encode("utf-8")
            # 获取词的分数
            score: float = tokenizer.get_score(i)

            # 初始化标记类型为 NORMAL
            toktype = gguf.TokenType.NORMAL
            # 如果词是未知的，则标记类型为 UNKNOWN
            if tokenizer.is_unknown(i):
                toktype = gguf.TokenType.UNKNOWN
            # 如果词是控制字符，则标记类型为 CONTROL
            if tokenizer.is_control(i):
                toktype = gguf.TokenType.CONTROL

            # 如果词是未使用的，则标记类型为 UNUSED
            if tokenizer.is_unused(i):
                toktype = gguf.TokenType.UNUSED
            # 如果词是字节，则标记类型为 BYTE
            if tokenizer.is_byte(i):
                toktype = gguf.TokenType.BYTE

            # 返回字节、分数和标记类型的元组
            yield text, score, toktype

    # 生成 added_tokens 方法，返回一个元组的可迭代对象，包含字节、分数和标记类型
    def added_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
        # 遍历添加的词列表
        for text in self.added_tokens_list:
            # 设置分数为 -1000.0
            score = -1000.0
            # 返回字节、分数和标记类型为 USER_DEFINED 的元组
            yield text.encode("utf-8"), score, gguf.TokenType.USER_DEFINED

    # 生成 all_tokens 方法，返回一个元组的可迭代对象，包含字节、分数和标记类型
    def all_tokens(self) -> Iterable[tuple[bytes, float, gguf.TokenType]]:
        # 返回 sentencepiece_tokens 方法的结果
        yield from self.sentencepiece_tokens()
        # 返回 added_tokens 方法的结果
        yield from self.added_tokens()

    # 生成 __repr__ 方法，返回描述对象的字符串
    def __repr__(self) -> str:
        # 返回包含基本词汇大小和添加词汇数量的描述字符串
        return f"<SentencePieceVocab with {self.vocab_size_base} base tokens and {len(self.added_tokens_list)} added tokens>"
# 定义类型别名 Vocab，可以是 'BpeVocab' 或 'SentencePieceVocab'
Vocab: TypeAlias = 'BpeVocab | SentencePieceVocab'

#
# 数据加载
# TODO: 重用（可能移动到 gguf.py 中）
#

# 定义函数 permute，对权重进行重新排列
def permute(weights: NDArray, n_head: int, n_head_kv: int) -> NDArray:
    # 如果 n_head_kv 不为空且 n_head 不等于 n_head_kv，则将 n_head 设置为 n_head_kv
    if n_head_kv is not None and n_head != n_head_kv:
        n_head = n_head_kv
    # 重新排列权重数组
    return (weights.reshape(n_head, 2, weights.shape[0] // n_head // 2, *weights.shape[1:])
                .swapaxes(1, 2)
                .reshape(weights.shape))

# 定义抽象类 Tensor
class Tensor(metaclass=ABCMeta):
    data_type: DataType

    @abstractmethod
    def astype(self, data_type: DataType) -> Tensor: ...
    @abstractmethod
    def permute(self, n_head: int, n_head_kv: int) -> Tensor: ...
    @abstractmethod
    def permute_part(self, n_part: int, n_head: int, n_head_kv: int) -> UnquantizedTensor: ...
    @abstractmethod
    def part(self, n_part: int) -> UnquantizedTensor: ...
    @abstractmethod
    def to_ggml(self) -> GGMLCompatibleTensor: ...

# 定义函数 bf16_to_fp32，将 bf16 数组转换为 fp32 数组
def bf16_to_fp32(bf16_arr: np.ndarray[Any, np.dtype[np.uint16]]) -> NDArray:
    # 断言输入数组的数据类型为 uint16
    assert bf16_arr.dtype == np.uint16, f"Input array should be of dtype uint16, but got {bf16_arr.dtype}"
    # 将 bf16 数组转换为 fp32 数组
    fp32_arr = bf16_arr.astype(np.uint32) << 16
    return fp32_arr.view(np.float32)

# 定义类 UnquantizedTensor，继承自 Tensor
class UnquantizedTensor(Tensor):
    # 初始化方法
    def __init__(self, ndarray: NDArray) -> None:
        # 断言输入参数为 np.ndarray 类型
        assert isinstance(ndarray, np.ndarray)
        self.ndarray = ndarray
        self.data_type = NUMPY_TYPE_TO_DATA_TYPE[ndarray.dtype]

    # 实现父类的抽象方法 astype
    def astype(self, data_type: DataType) -> Tensor:
        dtype = data_type.dtype
        # 如果数据类型为 DT_BF16，则将数组转换为 fp32 类型
        if self.data_type == DT_BF16:
            self.ndarray = bf16_to_fp32(self.ndarray)
        return UnquantizedTensor(self.ndarray.astype(dtype))

    # 实现父类的抽象方法 to_ggml
    def to_ggml(self) -> UnquantizedTensor:
        return self
    # 对输入的张量进行分块置换操作，返回置换后的张量
    def permute_part(self, n_part: int, n_head: int, n_head_kv: int) -> UnquantizedTensor:
        # 计算每个部分的大小
        r = self.ndarray.shape[0] // 3
        # 对指定部分进行分块置换操作，并返回置换后的张量
        return UnquantizedTensor(permute(self.ndarray[r * n_part : r * n_part + r, ...], n_head, n_head_kv))

    # 对输入的张量进行分块操作，返回指定部分的张量
    def part(self, n_part: int) -> UnquantizedTensor:
        # 计算每个部分的大小
        r = self.ndarray.shape[0] // 3
        # 返回指定部分的张量
        return UnquantizedTensor(self.ndarray[r * n_part : r * n_part + r, ...])

    # 对输入的张量进行置换操作，返回置换后的张量
    def permute(self, n_head: int, n_head_kv: int) -> UnquantizedTensor:
        # 对整个张量进行置换操作，并返回置换后的张量
        return UnquantizedTensor(permute(self.ndarray, n_head, n_head_kv))
# 加载未量化的 LazyTensor，返回 NDArray
def load_unquantized(lazy_tensor: LazyTensor, expected_dtype: Any = None, convert: bool = False) -> NDArray:
    # 加载 LazyTensor 中的数据
    tensor = lazy_tensor.load()
    # 确保加载的数据是 UnquantizedTensor 类型
    assert isinstance(tensor, UnquantizedTensor)

    # 再次检查数据形状是否一致
    actual_shape = list(tensor.ndarray.shape)
    assert actual_shape == lazy_tensor.shape, (actual_shape, lazy_tensor.shape)
    # 如果指定了期望的数据类型，并且加载的数据类型不一致
    if expected_dtype is not None and expected_dtype != tensor.ndarray.dtype:
        # 如果允许转换数据类型，则进行转换
        if convert:
            tensor.ndarray = tensor.ndarray.astype(expected_dtype)
        # 否则抛出数值错误
        else:
            raise ValueError(f'expected this tensor to have dtype {expected_dtype}, got {tensor.ndarray.dtype}')

    # 返回加载的数据
    return tensor.ndarray


# GGMLCompatibleTensor 类型与 UnquantizedTensor 类型相同
GGMLCompatibleTensor = UnquantizedTensor


# 定义 LazyTensor 类
@dataclass
class LazyTensor:
    _load: Callable[[], Tensor]
    shape: list[int]
    data_type: DataType
    description: str

    # 加载 LazyTensor 中的数据
    def load(self) -> Tensor:
        ret = self._load()
        # 确保加载的数据类型与 LazyTensor 的数据类型一致，或者可以转换
        assert ret.data_type == self.data_type or (self.data_type.dtype == ret.data_type.dtype), \
                (self.data_type, ret.data_type, self.description)
        return ret

    # 将 LazyTensor 转换为指定的数据类型
    def astype(self, data_type: DataType) -> LazyTensor:
        self.validate_conversion_to(data_type)

        # 定义加载函数，将加载的数据转换为指定的数据类型
        def load() -> Tensor:
            return self.load().astype(data_type)
        return LazyTensor(load, self.shape, data_type, f'convert({data_type}) {self.description}')
    
    # 对 LazyTensor 进行转置操作
    def transposed(self) -> LazyTensor:
        # 定义加载函数，对加载的数据进行转置操作
        def load() -> Tensor:
            loaded = self.load()
            assert isinstance(loaded, UnquantizedTensor), f'Cannot transpose {loaded}'
            loaded.ndarray = loaded.ndarray.T
            return loaded
        return LazyTensor(load, self.shape[::-1], self.data_type, f'transpose {self.description}')
    # 定义一个方法，用于验证数据类型转换是否有效
    def validate_conversion_to(self, data_type: DataType) -> None:
        # 如果目标数据类型不等于当前数据类型，并且目标数据类型的名称不在当前数据类型的有效转换列表中
        if data_type != self.data_type and data_type.name not in self.data_type.valid_conversions:
            # 抛出数值错误，提示无法验证从当前数据类型到目标数据类型的转换
            raise ValueError(f'Cannot validate conversion from {self.data_type} to {data_type}.')
# 定义类型别名 LazyModel，表示一个字典，键为字符串，值为 LazyTensor
LazyModel: TypeAlias = 'dict[str, LazyTensor]'

# 定义数据类 ModelPlus，包含 model（LazyModel）、paths（文件路径列表）、format（字符串字面值，表示模型格式）、vocab（Vocab 或 None）
@dataclass
class ModelPlus:
    model: LazyModel
    paths: list[Path]  # Where this was read from.
    format: Literal['ggml', 'torch', 'safetensors', 'none']
    vocab: Vocab | None  # For GGML models (which have vocab built in), the vocab.

# 定义函数 merge_sharded，接受一个 LazyModel 列表，返回一个 LazyModel
def merge_sharded(models: list[LazyModel]) -> LazyModel:
    # Original LLaMA models have each file contain one part of each tensor.
    # Use a dict instead of a set to preserve order.
    # 原始的 LLaMA 模型中，每个文件包含每个张量的一部分。使用字典而不是集合来保持顺序。
    names = {name: None for model in models for name in model}

    # 定义内部函数 convert，接受一个字符串参数 name，返回一个 LazyTensor
    def convert(name: str) -> LazyTensor:
        # 创建一个 LazyTensor 列表，包含每个模型中对应名称的 LazyTensor
        lazy_tensors: list[LazyTensor] = [model[name] for model in models]
        if len(lazy_tensors) == 1:
            # only one file; don't go through this procedure since there might
            # be quantized tensors
            # 只有一个文件；不要经过这个过程，因为可能有量化的张量
            return lazy_tensors[0]
        if len(lazy_tensors[0].shape) == 1:
            # the tensor is just duplicated in every file
            # 张量在每个文件中都是重复的
            return lazy_tensors[0]
        if name.startswith('tok_embeddings.') or \
           name.endswith('.attention.wo.weight') or \
           name.endswith('.feed_forward.w2.weight'):
            # split by columns
            # 按列拆分
            axis = 1
        else:
            # split by rows
            # 按行拆分
            axis = 0
        concatenated_shape = list(lazy_tensors[0].shape)
        concatenated_shape[axis] = sum(tensor.shape[axis] for tensor in lazy_tensors)

        # 定义内部函数 load，返回一个 UnquantizedTensor
        def load() -> UnquantizedTensor:
            # 创建一个包含每个 LazyTensor 数据的 NDArray 列表
            ndarrays = [load_unquantized(tensor) for tensor in lazy_tensors]
            # 沿指定轴连接数组
            concatenated: NDArray = np.concatenate(ndarrays, axis=axis)
            return UnquantizedTensor(concatenated)
        description = 'concatenated[[' + '] | ['.join(lt.description for lt in lazy_tensors) + ']]'
        # 返回一个 LazyTensor，包含 load 函数、拼接后的形状、数据类型和描述
        return LazyTensor(load, concatenated_shape, lazy_tensors[0].data_type, description)
    # 返回一个字典，包含每个名称和对应的转换后的 LazyTensor
    return {name: convert(name) for name in names}

# 定义函数 merge_multifile_models，接受一个 ModelPlus 列表，返回一个 ModelPlus
def merge_multifile_models(models_plus: list[ModelPlus]) -> ModelPlus:
    # 创建一个包含所有models_plus中format属性的集合
    formats = set(mp.format for mp in models_plus)
    # 确保所有的format属性都相同，否则抛出异常
    # assert len(formats) == 1, "different formats?"
    # 从集合中取出唯一的format值
    format = formats.pop()
    # 从所有models_plus中取出所有的路径，组成一个列表
    paths = [path for mp in models_plus for path in mp.paths]
    # 如果存在非None的vocab，则使用第一个非None的vocab
    try:
        vocab = next(mp.vocab for mp in models_plus if mp.vocab is not None)
    # 如果不存在非None的vocab，则将vocab设为None
    except StopIteration:
        vocab = None
    
    # 如果models_plus中存在"model.embed_tokens.weight"或"model.layers.0.fc1.weight"，则执行以下操作
    if any("model.embed_tokens.weight" in mp.model for mp in models_plus) or \
       any("model.layers.0.fc1.weight" in mp.model for mp in models_plus):
        # 初始化一个空的LazyModel字典
        model: LazyModel = {}
        # 遍历models_plus，将每个mp.model中的内容更新到model中
        for mp in models_plus:
            model.update(mp.model)
    # 如果不存在上述条件，则执行以下操作
    else:
        # 将所有models_plus中的model合并成一个model
        model = merge_sharded([mp.model for mp in models_plus])
    
    # 返回一个包含model, paths, format, vocab的ModelPlus对象
    return ModelPlus(model, paths, format, vocab)
# 定义一个函数，用于延迟加载 LazyTensor，并对其进行重新排列，返回一个 LazyTensor
def permute_lazy(lazy_tensor: LazyTensor, n_head: int, n_head_kv: int) -> LazyTensor:
    # 定义一个内部函数，用于加载 LazyTensor 并对其进行重新排列
    def load() -> Tensor:
        return lazy_tensor.load().permute(n_head, n_head_kv)
    # 返回一个新的 LazyTensor，其中包含加载函数、形状、数据类型和描述信息
    return LazyTensor(load, lazy_tensor.shape, lazy_tensor.data_type, f'permute({n_head}, {n_head_kv}) ' + lazy_tensor.description)

# 定义一个函数，用于延迟加载 LazyTensor 的部分数据，并对其进行重新排列，返回一个 LazyTensor
def permute_part_lazy(lazy_tensor: LazyTensor, n_part: int, n_head: int, n_head_kv: int) -> LazyTensor:
    # 定义一个内部函数，用于加载 LazyTensor 的部分数据并对其进行重新排列
    def load() -> Tensor:
        return lazy_tensor.load().permute_part(n_part, n_head, n_head_kv)
    # 复制 LazyTensor 的形状，并修改第一个维度的大小
    s = lazy_tensor.shape.copy()
    s[0] = s[0] // 3
    # 返回一个新的 LazyTensor，其中包含加载函数、修改后的形状、数据类型和描述信息
    return LazyTensor(load, s, lazy_tensor.data_type, f'permute({n_head}, {n_head_kv}) ' + lazy_tensor.description)

# 定义一个函数，用于延迟加载 LazyTensor 的部分数据，返回一个 LazyTensor
def part_lazy(lazy_tensor: LazyTensor, n_part: int) -> LazyTensor:
    # 定义一个内部函数，用于加载 LazyTensor 的部分数据
    def load() -> Tensor:
        return lazy_tensor.load().part(n_part)
    # 复制 LazyTensor 的形状，并修改第一个维度的大小
    s = lazy_tensor.shape.copy()
    s[0] = s[0] // 3
    # 返回一个新的 LazyTensor，其中包含加载函数、修改后的形状、数据类型和描述信息
    return LazyTensor(load, s, lazy_tensor.data_type, 'part ' + lazy_tensor.description)

# 定义一个数据类，表示延迟加载的存储类型
@dataclass
class LazyStorageKind:
    data_type: DataType

# 定义一个数据类，表示延迟加载的存储
@dataclass
class LazyStorage:
    load: Callable[[int, int], NDArray]
    kind: LazyStorageKind
    description: str

# 定义一个类，用于反序列化延迟加载的对象
class LazyUnpickler(pickle.Unpickler):
    def __init__(self, fp: IO[bytes], data_base_path: str, zip_file: zipfile.ZipFile):
        super().__init__(fp)
        self.data_base_path = data_base_path
        self.zip_file = zip_file
    # 从持久化标识符中加载数据
    def persistent_load(self, pid: Any) -> Any:
        # 断言标识符的第一个元素为'storage'
        assert pid[0] == 'storage'
        # 断言标识符的第二个元素为LazyStorageKind类型
        assert isinstance(pid[1], LazyStorageKind)
        # 获取数据类型
        data_type = pid[1].data_type
        # 获取文件名的基本部分
        filename_stem = pid[2]
        # 构建完整的文件路径
        filename = f'{self.data_base_path}/{filename_stem}'
        # 获取文件信息
        info = self.zip_file.getinfo(filename)

        # 定义加载函数，用于从ZIP文件中加载数据
        def load(offset: int, elm_count: int) -> NDArray:
            # 获取数据类型
            dtype = data_type.dtype
            # 打开ZIP文件
            fp = self.zip_file.open(info)
            # 定位到指定偏移量
            fp.seek(offset * dtype.itemsize)
            # 计算需要读取的数据大小
            size = elm_count * dtype.itemsize
            # 读取数据
            data = fp.read(size)
            # 断言读取的数据大小与期望的大小相同
            assert len(data) == size
            # 将读取的数据转换为NumPy数组
            return np.frombuffer(data, dtype)

        # 构建描述信息
        description = f'storage data_type={data_type} path-in-zip={filename} path={self.zip_file.filename}'
        # 返回LazyStorage对象
        return LazyStorage(load=load, kind=pid[1], description=description)

    # 从LazyStorage对象重建LazyTensor对象
    @staticmethod
    def lazy_rebuild_tensor_v2(storage: Any, storage_offset: Any, size: Any, stride: Any,
                               requires_grad: Any, backward_hooks: Any, metadata: Any = None) -> LazyTensor:
        # 断言storage参数为LazyStorage类型
        assert isinstance(storage, LazyStorage)

        # 定义加载函数，用于从LazyStorage中加载数据并重建为UnquantizedTensor
        def load() -> UnquantizedTensor:
            elm_count = stride[0] * size[0]
            return UnquantizedTensor(storage.load(storage_offset, elm_count).reshape(size))
        # 构建描述信息
        description = f'pickled storage_offset={storage_offset} in {storage.description}'
        # 返回LazyTensor对象
        return LazyTensor(load, list(size), storage.kind.data_type, description)

    # 从类型和状态重建对象
    @staticmethod
    def rebuild_from_type_v2(func, new_type, args, state):
        # 调用指定的函数并返回结果
        return func(*args)
    # 定义一个字典，键为元组（字符串，字符串），值为任意类型
    CLASSES: dict[tuple[str, str], Any] = {
        # 使用getattr作为一种解决方案，因为mypy无法智能判断staticmethod是否有__func__属性
        ('torch._tensor', '_rebuild_from_type_v2'): getattr(rebuild_from_type_v2, '__func__'),
        ('torch._utils', '_rebuild_tensor_v2'): getattr(lazy_rebuild_tensor_v2, '__func__'),
        ('torch', 'BFloat16Storage'): LazyStorageKind(DT_BF16),
        ('torch', 'HalfStorage'): LazyStorageKind(DT_F16),
        ('torch', 'FloatStorage'): LazyStorageKind(DT_F32),
        ('torch', 'IntStorage'): LazyStorageKind(DT_I32),
        ('torch', 'Tensor'): LazyTensor,
    }
    
    # 定义一个方法，用于查找类
    def find_class(self, module: str, name: str) -> Any:
        # 如果module不是以'torch'开头，则调用父类的find_class方法
        if not module.startswith('torch'):
            return super().find_class(module, name)
        # 返回CLASSES字典中对应键的值
        return self.CLASSES[(module, name)]
# 从外部文件流中延迟加载 Torch 文件，返回模型和路径的组合对象
def lazy_load_torch_file(outer_fp: IO[bytes], path: Path) -> ModelPlus:
    # 使用外部文件流创建 ZIP 文件对象
    zf = zipfile.ZipFile(outer_fp)
    # 获取 ZIP 文件中以 .pkl 结尾的文件路径列表
    pickle_paths = [name for name in zf.namelist() if name.endswith('.pkl')]
    # 确保只有一个 .pkl 文件
    assert len(pickle_paths) == 1, pickle_paths
    # 打开 .pkl 文件，创建 pickle 文件对象
    pickle_fp = zf.open(pickle_paths[0], 'r')
    # 创建 LazyUnpickler 对象，用于延迟加载模型
    unpickler = LazyUnpickler(pickle_fp,
                              data_base_path=pickle_paths[0][:-4],
                              zip_file=zf)
    # 加载模型
    model = unpickler.load()
    # 将模型转换为字典形式
    as_dict = dict(model.items())
    # 返回模型和路径的组合对象
    return ModelPlus(model=as_dict, paths=[path], format='torch', vocab=None)


# 从文件流中延迟加载 SafeTensors 文件，返回模型和路径的组合对象
def lazy_load_safetensors_file(fp: IO[bytes], path: Path) -> ModelPlus:
    # 读取头部大小
    header_size, = struct.unpack('<Q', fp.read(8))
    # 解析头部 JSON 数据
    header: dict[str, dict[str, Any]] = json.loads(fp.read(header_size))
    # 使用 mmap 来避免文件偏移的竞争条件
    mapped = memoryview(mmap.mmap(fp.fileno(), 0, access=mmap.ACCESS_READ))
    # 从文件流中获取数据
    byte_buf = mapped[8 + header_size:]

    # 将信息转换为 LazyTensor 对象
    def convert(info: dict[str, Any]) -> LazyTensor:
        # 获取数据类型和形状
        data_type = SAFETENSORS_DATA_TYPES[info['dtype']]
        numpy_dtype = data_type.dtype
        shape: list[int] = info['shape']
        begin, end = info['data_offsets']
        # 确保数据偏移和长度正确
        assert 0 <= begin <= end <= len(byte_buf)
        assert end - begin == math.prod(shape) * numpy_dtype.itemsize
        buf = byte_buf[begin:end]

        # 加载数据的函数
        def load() -> UnquantizedTensor:
            return UnquantizedTensor(np.frombuffer(buf, dtype=numpy_dtype).reshape(shape))
        # 创建 LazyTensor 对象
        description = f'safetensors begin={begin} end={end} type={data_type} path={path}'
        return LazyTensor(load, shape, data_type, description)
    # 将信息转换为模型字典
    model = {name: convert(info) for (name, info) in header.items() if name != '__metadata__'}
    # 返回模型和路径的组合对象
    return ModelPlus(model=model, paths=[path], format='safetensors', vocab=None)


# 从文件流中必须读取指定长度的字节数据
def must_read(fp: IO[bytes], length: int) -> bytes:
    # 读取指定长度的字节数据
    ret = fp.read(length)
    # 如果读取的数据长度小于指定长度，抛出异常
    if len(ret) < length:
        raise Exception("unexpectedly reached end of file")
    return ret
# 使用functools.lru_cache装饰器，对lazy_load_file函数进行缓存，maxsize=None表示不限制缓存大小
@functools.lru_cache(maxsize=None)
# 定义lazy_load_file函数，接受Path类型参数，返回ModelPlus类型
def lazy_load_file(path: Path) -> ModelPlus:
    # 以二进制只读方式打开文件
    fp = open(path, 'rb')
    # 读取文件的前8个字节
    first8 = fp.read(8)
    # 将文件指针移动到文件开头
    fp.seek(0)
    # 如果文件的前两个字节是'PK'，表示是zip文件，即PyTorch格式，调用lazy_load_torch_file函数加载文件
    if first8[:2] == b'PK':
        return lazy_load_torch_file(fp, path)
    # 如果文件的前8个字节表示的数字小于16*1024*1024，可能是safetensors文件，调用lazy_load_safetensors_file函数加载文件
    elif struct.unpack('<Q', first8)[0] < 16 * 1024 * 1024:
        return lazy_load_safetensors_file(fp, path)
    # 否则抛出数值错误，表示未知格式
    else:
        raise ValueError(f"unknown format: {path}")


# 定义泛型In和Out
In = TypeVar('In')
Out = TypeVar('Out')

# 定义bounded_parallel_map函数，接受func、iterable、concurrency、max_workers和use_processpool_executor等参数，返回Iterable[Out]类型
def bounded_parallel_map(func: Callable[[In], Out], iterable: Iterable[In], concurrency: int, max_workers: int | None = None, use_processpool_executor: bool = False) -> Iterable[Out]:
    '''Parallel map, but with backpressure.  If the caller doesn't call `next`
    fast enough, this will stop calling `func` at some point rather than
    letting results pile up in memory.  Specifically, there is a max of one
    output value buffered per thread.'''
    # 如果并发数小于2，使用map函数逐个调用func，并通过yield from返回结果
    if concurrency < 2:
        yield from map(func, iterable)
        # Not reached.
    # 将iterable转换为迭代器
    iterable = iter(iterable)
    # 定义executor_class变量，根据use_processpool_executor的值选择ThreadPoolExecutor或ProcessPoolExecutor
    executor_class: type[ThreadPoolExecutor] | type[ProcessPoolExecutor]
    if use_processpool_executor:
        executor_class = ProcessPoolExecutor
    else:
        executor_class = ThreadPoolExecutor
    # 使用executor_class创建线程池或进程池，max_workers参数指定最大工作线程数
    with executor_class(max_workers = max_workers) as executor:
        # 定义futures列表，用于存储Future对象
        futures: list[concurrent.futures.Future[Out]] = []
        # 标记任务是否完成
        done = False
        # 根据并发数循环提交任务
        for _ in range(concurrency):
            try:
                futures.append(executor.submit(func, next(iterable)))
            except StopIteration:
                done = True
                break

        # 循环处理已完成的任务
        while futures:
            # 获取已完成任务的结果
            result = futures.pop(0).result()
            # 当任务未完成且futures列表长度小于并发数时，继续提交任务
            while not done and len(futures) < concurrency:
                try:
                    futures.append(executor.submit(func, next(iterable)))
                except StopIteration:
                    done = True
                    break
            # 通过yield返回结果
            yield result
# 检查词汇表大小是否匹配，如果不匹配则抛出异常
def check_vocab_size(params: Params, vocab: Vocab) -> None:
    # 如果参数中的词汇表大小不等于实际词汇表大小
    if params.n_vocab != vocab.vocab_size:
        # 断言词汇表是 BpeVocab 或 SentencePieceVocab 类型
        assert isinstance(vocab, BpeVocab) or isinstance(vocab, SentencePieceVocab)
        # 如果参数中的词汇表大小等于基本词汇表大小
        if params.n_vocab == vocab.vocab_size_base:
            # 打印提示信息并重置词汇表的一些属性
            print("Ignoring added_tokens.json since model matches vocab size without it.")
            vocab.added_tokens_list = []
            vocab.vocab_size = vocab.vocab_size_base
            return
        # 构造异常信息
        msg = f"Vocab size mismatch (model has {params.n_vocab}, but {vocab.fname_tokenizer}"
        # 如果存在额外的 tokens 文件名，则添加到异常信息中
        if vocab.fname_added_tokens is not None:
            msg += f" combined with {vocab.fname_added_tokens}"
        msg += f" has {vocab.vocab_size})."
        # 如果词汇表大小在一定范围内，并且没有额外的 tokens 文件，则添加提示信息到异常信息中
        if vocab.vocab_size < params.n_vocab < vocab.vocab_size + 20 and vocab.fname_added_tokens is None:
            msg += f"  Most likely you are missing added_tokens.json (should be in {vocab.fname_tokenizer.parent})."
        # 抛出异常
        raise Exception(msg)


# 定义 OutputFile 类
class OutputFile:
    # 初始化方法
    def __init__(self, fname_out: Path, endianess:gguf.GGUFEndian=gguf.GGUFEndian.LITTLE) -> None:
        # 创建 GGUFWriter 对象并赋值给实例属性 gguf
        self.gguf = gguf.GGUFWriter(fname_out, gguf.MODEL_ARCH_NAMES[ARCH], endianess=endianess)
    # 添加元数据架构信息
    def add_meta_arch(self, params: Params) -> None:
        # 设置默认模型名称为 "LLaMA"
        name = "LLaMA"

        # TODO: 更好的逻辑来确定模型名称
        # 如果上下文长度为 4096，则模型名称为 "LLaMA v2"
        if params.n_ctx == 4096:
            name = "LLaMA v2"
        # 如果模型路径不为空，则使用路径中的最后一个目录名作为模型名称
        elif params.path_model is not None:
            name = str(params.path_model.parent).split('/')[-1]

        # 添加模型名称到元数据中
        self.gguf.add_name(name)
        # 添加上下文长度到元数据中
        self.gguf.add_context_length(params.n_ctx)
        # 添加嵌入长度到元数据中
        self.gguf.add_embedding_length(params.n_embd)
        # 添加块数量到元数据中
        self.gguf.add_block_count(params.n_layer)
        # 添加前馈长度到元数据中
        self.gguf.add_feed_forward_length(params.n_ff)
        # 添加绳子维度数量到元数据中
        self.gguf.add_rope_dimension_count(params.n_embd // params.n_head)
        # 添加头数量到元数据中
        self.gguf.add_head_count(params.n_head)
        # 添加键值头数量到元数据中
        self.gguf.add_head_count_kv(params.n_head_kv)
        # 添加层归一化 RMS 误差到元数据中
        self.gguf.add_layer_norm_rms_eps(params.f_norm_eps)

        # 如果绳子频率基数不为空，则添加到元数据中
        if params.f_rope_freq_base is not None:
            self.gguf.add_rope_freq_base(params.f_rope_freq_base)

        # 如果绳子缩放类型不为空，则添加到元数据中，并确保绳子缩放因子不为空
        if params.rope_scaling_type:
            assert params.f_rope_scale is not None
            self.gguf.add_rope_scaling_type(params.rope_scaling_type)
            self.gguf.add_rope_scaling_factor(params.f_rope_scale)

        # 如果原始上下文长度不为空，则添加到元数据中
        if params.n_orig_ctx is not None:
            self.gguf.add_rope_scaling_orig_ctx_len(params.n_orig_ctx)

        # 如果绳子微调不为空，则添加到元数据中
        if params.rope_finetuned is not None:
            self.gguf.add_rope_scaling_finetuned(params.rope_finetuned)

        # 如果文件类型不为空，则添加到元数据中
        if params.ftype is not None:
            self.gguf.add_file_type(params.ftype)

        # 如果预测器参数中的稀疏阈值不为空，则添加到元数据中
        if params.predictor_params.sparse_threshold is not None:
            self.gguf.add_sparse_threshold(params.predictor_params.sparse_threshold)
    # 向元数据中添加词汇表，包括词汇、分数和词汇类型
    def add_meta_vocab(self, vocab: Vocab) -> None:
        tokens = []  # 存储词汇
        scores = []  # 存储分数
        toktypes = []  # 存储词汇类型
        # 注意：`all_tokens` 返回基本词汇和添加的词汇
        for text, score, toktype in vocab.all_tokens():
            tokens.append(text)  # 添加词汇
            scores.append(score)  # 添加分数
            toktypes.append(toktype)  # 添加词汇类型

        if isinstance(vocab, SentencePieceVocab):  # 如果词汇类型是 SentencePieceVocab
            self.gguf.add_tokenizer_model("llama")  # 向 gguf 中添加分词器模型 llama
        elif isinstance(vocab, BpeVocab):  # 如果词汇类型是 BpeVocab
            self.gguf.add_tokenizer_model("gpt2")  # 向 gguf 中添加分词器模型 gpt2
        else:
            raise ValueError('Unknown vocab type: Not BpeVocab or SentencePieceVocab')  # 抛出值错误，未知的词汇类型：不是 BpeVocab 或 SentencePieceVocab
        self.gguf.add_token_list(tokens)  # 向 gguf 中添加词汇列表
        self.gguf.add_token_scores(scores)  # 向 gguf 中添加词汇分数
        self.gguf.add_token_types(toktypes)  # 向 gguf 中添加词汇类型

    # 向元数据中添加特殊词汇表
    def add_meta_special_vocab(self, svocab: gguf.SpecialVocab) -> None:
        svocab.add_to_gguf(self.gguf)  # 将特殊词汇表添加到 gguf 中

    # 向元数据中添加张量信息
    def add_tensor_info(self, name: str, tensor: LazyTensor) -> None:
        n_elements = int(np.prod(tensor.shape))  # 计算张量元素个数
        raw_dtype = getattr(tensor.data_type, 'ggml_type', None)  # 获取原始数据类型
        data_type = getattr(tensor.data_type, 'quantized_type', None) or tensor.data_type.dtype  # 获取数据类型
        data_nbytes = tensor.data_type.elements_to_bytes(n_elements)  # 计算数据字节数
        self.gguf.add_tensor_info(name, tensor.shape, data_type, data_nbytes, raw_dtype = raw_dtype)  # 向 gguf 中添加张量信息

    # 将元数据头部写入文件
    def write_meta(self) -> None:
        self.gguf.write_header_to_file()  # 将 gguf 的头部写入文件
        self.gguf.write_kv_data_to_file()  # 将 gguf 的键值数据写入文件

    # 将张量信息写入文件
    def write_tensor_info(self) -> None:
        self.gguf.write_ti_data_to_file()  # 将 gguf 的张量信息写入文件

    # 关闭 gguf
    def close(self) -> None:
        self.gguf.close()  # 关闭 gguf

    @staticmethod
    # 写入仅包含词汇表的文件
    def write_vocab_only(fname_out: Path, params: Params, vocab: Vocab, svocab: gguf.SpecialVocab, endianess:gguf.GGUFEndian=gguf.GGUFEndian.LITTLE) -> None:
        # 检查词汇表大小是否符合要求
        check_vocab_size(params, vocab)

        # 创建输出文件对象
        of = OutputFile(fname_out, endianess=endianess)

        # 添加元数据：架构信息
        of.add_meta_arch(params)
        # 添加元数据：词汇表信息
        of.add_meta_vocab(vocab)
        # 添加元数据：特殊词汇表信息
        of.add_meta_special_vocab(svocab)

        # 写入元数据
        of.write_meta()

        # 关闭输出文件对象
        of.close()

    # 处理单个项目
    @staticmethod
    def do_item(item: tuple[str, LazyTensor]) -> tuple[DataType, NDArray]:
        # 解包项目元组
        name, lazy_tensor = item
        # 加载懒惰张量并转换为 GGML 格式的张量
        tensor = lazy_tensor.load().to_ggml()
        # 返回数据类型和张量数组的元组
        return (lazy_tensor.data_type, tensor.ndarray)

    # 可能进行量化处理
    @staticmethod
    def maybe_do_quantize(item: tuple[DataType, NDArray]) -> NDArray:
        # 解包数据类型和数组元组
        dt, arr = item
        # 如果数据类型不是量化数据类型，则直接返回数组
        if not isinstance(dt, QuantizedDataType):
            return arr
        # 对数组进行量化处理并返回
        return dt.quantize(arr)

    @staticmethod
    # 定义一个函数，将模型数据写入指定文件
    def write_all(fname_out: Path, ftype: GGMLFileType, params: Params, model: LazyModel, vocab: Vocab, svocab: gguf.SpecialVocab, concurrency: int = DEFAULT_CONCURRENCY, endianess: gguf.GGUFEndian = gguf.GGUFEndian.LITTLE) -> None:
        # 检查词汇表大小是否符合要求
        check_vocab_size(params, vocab)

        # 创建一个输出文件对象
        of = OutputFile(fname_out, endianess=endianess)

        # 添加元数据：架构参数、词汇表、特殊词汇表
        of.add_meta_arch(params)
        of.add_meta_vocab(vocab)
        of.add_meta_special_vocab(svocab)

        # 添加张量信息
        for name, lazy_tensor in model.items():
            of.add_tensor_info(name, lazy_tensor)

        # 写入元数据
        of.write_meta()
        of.write_tensor_info()

        # 处理张量数据
        ndarrays_inner = bounded_parallel_map(OutputFile.do_item, model.items(), concurrency = concurrency)
        if ftype == GGMLFileType.MostlyQ8_0:
            # 如果文件类型为MostlyQ8_0，则使用并行处理
            ndarrays = bounded_parallel_map(OutputFile.maybe_do_quantize, ndarrays_inner, concurrency = concurrency, max_workers = concurrency, use_processpool_executor = True)
        else:
            # 否则，使用普通的处理方式
            ndarrays = map(OutputFile.maybe_do_quantize, ndarrays_inner)

        # 记录开始时间
        start = time.time()
        # 遍历模型数据，写入张量数据
        for i, ((name, lazy_tensor), ndarray) in enumerate(zip(model.items(), ndarrays)):
            elapsed = time.time() - start
            size = ' x '.join(f"{dim:6d}" for dim in lazy_tensor.shape)
            padi = len(str(len(model)))
            # 打印写入进度信息
            print(f"[{i+1:{padi}d}/{len(model)}] Writing tensor {name:38s} | size {size:16} | type {lazy_tensor.data_type.name:4} | T+{int(elapsed):4}")
            of.gguf.write_tensor_data(ndarray)

        # 关闭输出文件
        of.close()
# 根据给定的模型和输出类型字符串，返回对应的 GGML 文件类型
def pick_output_type(model: LazyModel, output_type_str: str | None) -> GGMLFileType:
    # 获取模型中注意力权重的数据类型
    wq_type = model[gguf.TENSOR_NAMES[gguf.MODEL_TENSOR.ATTN_Q].format(bid=0)+".weight"].data_type

    # 如果输出类型为 "f32" 或者为 None 且注意力权重数据类型为 DT_F32，则返回 GGMLFileType.AllF32
    if output_type_str == "f32" or (output_type_str is None and wq_type == DT_F32):
        return GGMLFileType.AllF32
    # 如果输出类型为 "f16" 或者为 None 且注意力权重数据类型为 DT_F16 或 DT_BF16，则返回 GGMLFileType.MostlyF16
    if output_type_str == "f16" or (output_type_str is None and wq_type in (DT_F16, DT_BF16)):
        return GGMLFileType.MostlyF16
    # 如果输出类型为 "q8_0"，则返回 GGMLFileType.MostlyQ8_0
    if output_type_str == "q8_0":
        return GGMLFileType.MostlyQ8_0

    # 创建一个字典，将模型中每个张量的名称映射到其数据类型
    name_to_type = {name: lazy_tensor.data_type for (name, lazy_tensor) in model.items()}

    # 抛出异常，指示出现了意外的类型组合
    raise Exception(f"Unexpected combination of types: {name_to_type}")

# 将模型转换为指定的输出类型
def convert_to_output_type(model: LazyModel, output_type: GGMLFileType) -> LazyModel:
    # 使用推断的类型将模型中的每个张量转换为指定的输出类型
    return {name: tensor.astype(output_type.type_for_tensor(name, tensor))
            for (name, tensor) in model.items()}

# 转换模型中的名称
def convert_model_names(model: LazyModel, params: Params) -> LazyModel:
    # 创建一个张量名称映射对象，根据模型架构和层数
    tmap = gguf.TensorNameMap(ARCH, params.n_layer)
    # 创建一个应该跳过的模型张量集合，根据模型架构
    should_skip: set[gguf.MODEL_TENSOR] = set(gguf.MODEL_TENSOR_SKIP.get(ARCH, []))

    # 临时保存模型
    tmp = model

    # HF 模型对一些张量进行排列或打包，因此需要撤消这些操作
    # 使用 itertools.count() 创建一个无限迭代器，用于循环遍历
    for i in itertools.count():
        # 检查模型中是否存在指定的层权重
        if f"model.layers.{i}.self_attn.q_proj.weight" in model:
            # 如果存在，打印信息
            print(f"Permuting layer {i}")
            # 对权重进行排列
            tmp[f"model.layers.{i}.self_attn.q_proj.weight"] = permute_lazy(model[f"model.layers.{i}.self_attn.q_proj.weight"], params.n_head, params.n_head)
            tmp[f"model.layers.{i}.self_attn.k_proj.weight"] = permute_lazy(model[f"model.layers.{i}.self_attn.k_proj.weight"], params.n_head, params.n_head_kv)
           #tmp[f"model.layers.{i}.self_attn.v_proj.weight"] =              model[f"model.layers.{i}.self_attn.v_proj.weight"]
        # 如果不存在指定的层权重，检查是否存在其他指定的权重
        elif f"model.layers.{i}.self_attn.W_pack.weight" in model:
            # 如果存在，打印信息
            print(f"Unpacking and permuting layer {i}")
            # 对权重进行部分排列
            tmp[f"model.layers.{i}.self_attn.q_proj.weight"] = permute_part_lazy(model[f"model.layers.{i}.self_attn.W_pack.weight"], 0, params.n_head, params.n_head)
            tmp[f"model.layers.{i}.self_attn.k_proj.weight"] = permute_part_lazy(model[f"model.layers.{i}.self_attn.W_pack.weight"], 1, params.n_head, params.n_head_kv)
            tmp[f"model.layers.{i}.self_attn.v_proj.weight"] = part_lazy        (model[f"model.layers.{i}.self_attn.W_pack.weight"], 2)
            del tmp[f"model.layers.{i}.self_attn.W_pack.weight"]
        else:
            # 如果都不存在，跳出循环
            break

    # 创建一个空的 LazyModel 对象
    out: LazyModel = {}
    # 遍历模型中的每个元素
    for name, lazy_tensor in model.items():
        # 获取张量类型和名称
        tensor_type, name_new = tmap.get_type_and_name(name, try_suffixes = (".weight", ".bias")) or (None, None)
        # 如果名称为空，抛出异常
        if name_new is None:
            raise Exception(f"Unexpected tensor name: {name}")

        # 如果张量类型在应该跳过的列表中，打印信息并继续下一次循环
        if tensor_type in should_skip:
            print(f"skipping tensor {name_new}")
            continue

        # 打印张量的名称、新名称、数据类型和形状
        print(f"{name:48s} -> {name_new:40s} | {lazy_tensor.data_type.name:6s} | {lazy_tensor.shape}")
        # 将张量添加到输出对象中
        out[name_new] = lazy_tensor

    # 返回输出对象
    return out
# 转置 LazyModel 中的 ffn_down 矩阵，用于 Axpy 操作
def postprocess_transpose(model: LazyModel) -> LazyModel:
    # 创建一个空的 LazyModel 对象
    out: LazyModel = {}
    
    # 遍历 LazyModel 中的每个键值对
    for name, lazy_tensor in model.items():
        # 如果键以 ".ffn_down.weight" 结尾，则将键名中的 "ffn_down" 替换为 "ffn_down_t"，并将对应的值进行转置
        if name.endswith(".ffn_down.weight"):
            out[name.replace("ffn_down", "ffn_down_t")] = lazy_tensor.transposed()
        # 否则直接将键值对添加到新的 LazyModel 对象中
        else:
            out[name] = lazy_tensor
    
    # 返回转置后的 LazyModel 对象
    return out

# 给定多文件模型的任意路径和一个整数 n，返回模型中的第 n 个路径
def nth_multifile_path(path: Path, n: int) -> Path | None:
    '''Given any path belonging to a multi-file model (e.g. foo.bin.1), return
    the nth path in the model.
    '''
    # 支持以下模式：
    patterns: list[tuple[str, str]] = [
        # - x.00.pth, x.01.pth, 等
        (r'\.[0-9]{2}\.pth$', f'.{n:02}.pth'),
        # - x-00001-of-00002.bin, x-00002-of-00002.bin, 等
        (r'-[0-9]{5}-of-(.*)$', fr'-{n:05}-of-\1'),
        # x.bin, x.bin.1, 等
        (r'(\.[0-9]+)?$', r'\1' if n == 0 else fr'\1.{n}'),
        # x_0.pt, x_1.pt, 等
        (r'(_[0-9]+)?\.pt$', fr'_{n}.pt'),
    ]
    # 遍历模式列表
    for regex, replacement in patterns:
        # 如果路径名匹配正则表达式
        if re.search(regex, path.name):
            # 构建新的路径并检查其是否存在，存在则返回该路径
            new_path = path.with_name(re.sub(regex, replacement, path.name))
            if new_path.exists():
                return new_path
    # 如果没有匹配的路径，则返回 None
    return None

# 给定任意属于多文件模型的路径，返回模型中的所有路径列表
def find_multifile_paths(path: Path) -> list[Path]:
    '''Given any path belonging to a multi-file model (e.g. foo.bin.1), return
    the whole list of paths in the model.
    '''
    # 创建一个空的路径列表
    ret: list[Path] = []
    # 使用 itertools.count() 生成无限序列，遍历模型中的所有路径
    for i in itertools.count():
        # 获取第 i 个路径
        nth_path = nth_multifile_path(path, i)
        # 如果路径不存在，则跳出循环
        if nth_path is None:
            break
        # 将路径添加到路径列表中
        ret.append(nth_path)
    # 如果路径列表为空，则返回包含原始路径的列表；否则返回路径列表
    if not ret:
        return [path]
    return ret

# 加载任何支持的格式的模型
def load_some_model(path: Path) -> ModelPlus:
    '''Load a model of any supported format.'''
    # Be extra-friendly and accept either a file or a directory: 
    # 检查路径是文件还是目录，接受两种形式的输入

    if path.is_dir():
        # Check if it's a set of safetensors files first
        # 检查是否是一组 safetensors 文件
        globs = ["model-00001-of-*.safetensors", "model.safetensors"]
        # 使用 glob 模块匹配文件名
        files = [file for glob in globs for file in path.glob(glob)]
        if not files:
            # Try the PyTorch patterns too, with lower priority
            # 如果没有找到 safetensors 文件，则尝试使用 PyTorch 的模式匹配
            globs = ["consolidated.00.pth", "pytorch_model-00001-of-*.bin", "*.pt", "pytorch_model.bin"]
            files = [file for glob in globs for file in path.glob(glob)]
        if not files:
            # 如果仍然没有找到文件，则抛出异常
            raise Exception(f"Can't find model in directory {path}")
        if len(files) > 1:
            # 如果找到多个文件，则抛出异常
            raise Exception(f"Found multiple models in {path}, not sure which to pick: {files}")
        # 将路径设置为找到的文件的第一个
        path = files[0]

    # 查找多文件路径
    paths = find_multifile_paths(path)
    # 创建一个空列表用于存储 ModelPlus 对象
    models_plus: list[ModelPlus] = []
    # 遍历路径列表
    for path in paths:
        # 打印加载的模型文件路径
        print(f"Loading model file {path}")
        # 懒加载文件并将其添加到 models_plus 列表中
        models_plus.append(lazy_load_file(path))

    # 合并多文件模型
    model_plus = merge_multifile_models(models_plus)
    # 返回合并后的模型
    return model_plus
def load_mlp_model(path: Path) -> ModelPlus:
    '''Load MLP models for sparse attention from directory.'''
    # 确保路径是一个目录
    assert path.is_dir(), f"MLP model path {path} is not a directory"
    
    # 获取第一个模型文件的路径
    first_model_path = path / "model_0.pt"
    # 确保第一个模型文件存在
    assert first_model_path.resolve(), f"MLP model path {path} does not contain model_0.pt"

    # 查找所有模型文件的路径
    model_paths = find_multifile_paths(first_model_path)
    models_plus: list[ModelPlus] = []
    for model_path in model_paths:
        # 从模型文件路径中提取模型层编号
        model_layer = int(re.search(r'model_(\d+).pt', str(model_path)).group(1))
        print(f"Loading MLP model file {model_path}")
        # 懒加载模型文件
        mlp_model = lazy_load_file(model_path)
        # 重命名模型中的层
        mlp_model.model = {f"model.layers.{model_layer}.{name}": tensor for name, tensor in mlp_model.model.items()}
        # 将模型添加到列表中
        models_plus.append(mlp_model)

    # 合并多个模型文件的模型
    return merge_multifile_models(models_plus)


def load_vocab(path: Path, vocabtype: str | None) -> Vocab:
    # Be extra-friendly and accept either a file or a directory.  Also, if it's
    # a directory, it might be the model directory, and tokenizer.model might
    # be in the parent of that.
    # 如果路径是一个目录，则可能是模型目录，词汇文件可能在其父目录中
    if path.is_dir():
        vocab_file = "tokenizer.model"
        if vocabtype == 'bpe':
            vocab_file = "vocab.json"
        path2 = path / vocab_file
        # 使用 `.parent` 而不是 /.. 更好地处理符号链接的情况
        path3 = path.parent / vocab_file
        # 如果 path2 存在，则使用 path2，否则使用 path3
        if path2.exists():
            path = path2
        elif path3.exists():
            path = path3
        else:
            # 如果在路径或其父目录中找不到词汇文件，则引发 FileNotFoundError
            raise FileNotFoundError(
                f"Could not find {vocab_file} in {path} or its parent; "
                "if it's in another directory, pass the directory as --vocab-dir")

    # 打印加载的词汇文件路径和类型
    print(f"Loading vocab file '{path}', type '{vocabtype}'")

    # 获取添加的标记文件路径
    added_tokens_path = path.parent / "added_tokens.json"
    # 如果词汇类型是 "bpe"，则返回 BpeVocab 对象
    if vocabtype == "bpe":
        return BpeVocab(path, added_tokens_path if added_tokens_path.exists() else None)
    # 如果词汇表类型是 "spm"，则返回一个 SentencePieceVocab 对象
    elif vocabtype == "spm":
        # 如果存在添加的标记路径，则使用该路径创建 SentencePieceVocab 对象，否则传入 None
        return SentencePieceVocab(path, added_tokens_path if added_tokens_path.exists() else None)
    # 如果词汇表类型不是 "spm"，则抛出数值错误，提示不支持的词汇表类型
    else:
        raise ValueError(f"Unsupported vocabulary type {vocabtype}")
# 根据模型路径列表和文件类型返回默认输出文件路径
def default_outfile(model_paths: list[Path], file_type: GGMLFileType) -> Path:
    # 根据文件类型选择对应的文件名字符串
    namestr = {
        GGMLFileType.AllF32:    "f32",
        GGMLFileType.MostlyF16: "f16",
        GGMLFileType.MostlyQ8_0:"q8_0",
    }[file_type]
    # 根据文件名字符串构建默认输出文件路径
    ret = model_paths[0].parent / f"ggml-model-{namestr}.gguf"
    # 如果默认输出路径已经存在于模型路径中，则报错并退出程序
    if ret in model_paths:
        sys.stderr.write(
            f"Error: Default output path ({ret}) would overwrite the input. "
            "Please explicitly specify a path using --outfile.\n")
        sys.exit(1)
    # 返回默认输出文件路径
    return ret


# 执行模型转储操作
def do_dump_model(model_plus: ModelPlus) -> None:
    # 打印模型路径
    print(f"model_plus.paths = {model_plus.paths!r}")
    # 打印模型格式
    print(f"model_plus.format = {model_plus.format!r}")
    # 打印模型词汇表
    print(f"model_plus.vocab = {model_plus.vocab!r}")
    # 遍历模型中的每个名称和懒惰张量，并打印相关信息
    for name, lazy_tensor in model_plus.model.items():
        print(f"{name}: shape={lazy_tensor.shape} type={lazy_tensor.data_type}; {lazy_tensor.description}")


# 主函数
def main(args_in: list[str] | None = None) -> None:
    # 输出格式选项
    output_choices = ["f32", "f16"]
    # 如果系统是小端序，则支持 q8_0 输出格式
    if np.uint32(1) == np.uint32(1).newbyteorder("<"):
        output_choices.append("q8_0")
    # 创建命令行参数解析器
    parser = argparse.ArgumentParser(description="Convert a LLaMa model to a GGML compatible file")
    # 添加命令行参数：是否只转储模型内容
    parser.add_argument("--dump",        action="store_true",    help="don't convert, just show what's in the model")
    # 添加命令行参数：是否只转储单个模型文件内容
    parser.add_argument("--dump-single", action="store_true",    help="don't convert, just show what's in a single model file")
    # 添加命令行参数：是否只提取词汇表
    parser.add_argument("--vocab-only",  action="store_true",    help="extract only the vocab")
    # 添加命令行参数：输出格式选择
    parser.add_argument("--outtype",     choices=output_choices, help="output format - note: q8_0 may be very slow (default: f16 or f32 based on input)", default="f16")
    # 添加命令行参数：词汇表所在目录
    parser.add_argument("--vocab-dir",   type=Path,              help="directory containing tokenizer.model, if separate from model file")
    # 添加一个名为“--outfile”的命令行参数，类型为Path，用于指定输出文件的路径；默认值为基于输入文件的路径
    parser.add_argument("--outfile",     type=Path,              help="path to write to; default: based on input")
    # 添加一个名为“--ctx”的命令行参数，类型为int，用于指定模型训练上下文（默认值：基于输入）
    parser.add_argument("--ctx",         type=int,               help="model training context (default: based on input)")
    # 添加一个名为“--concurrency”的命令行参数，类型为int，用于指定转换时使用的并发数（默认值：DEFAULT_CONCURRENCY）
    parser.add_argument("--concurrency", type=int,               help=f"concurrency used for conversion (default: {DEFAULT_CONCURRENCY})", default = DEFAULT_CONCURRENCY)
    # 添加一个名为“--bigendian”的命令行参数，action为“store_true”，用于指定模型在大端机器上执行
    parser.add_argument("--bigendian",   action="store_true",    help="model is executed on big endian machine")
    # 添加一个名为“--vocabtype”的命令行参数，choices为["spm", "bpe"]，用于指定词汇表格式（默认值：spm）
    parser.add_argument("--vocabtype",   choices=["spm", "bpe"], help="vocab format (default: spm)", default="spm")
    # 添加一个位置参数“model”，类型为Path，用于指定包含模型文件的目录，或者模型文件本身（*.pth, *.pt, *.bin, *.safetensors）
    parser.add_argument("model",         type=Path,              help="directory containing model file, or model file itself (*.pth, *.pt, *.bin, *.safetensors)")
    # 添加一个位置参数“mlp_model”，类型为Path，用于指定稀疏注意力的MLP模型
    parser.add_argument("mlp_model",     type=Path,              help="MLP model for sparse attention")

    # 解析命令行参数
    args = parser.parse_args(args_in)

    # 尝试打开模型目录下的“config.json”文件，读取其中的配置信息
    try:
        with open(args.model / "config.json", "r", encoding="utf-8") as f:
            hf_config = json.load(f)
        # 如果配置中的模型类型不是“llama”
        if model_type := hf_config.get("model_type") != "llama":
            # 调用另一个脚本来转换其他类型的模型
            print(f"Model architecture {model_type} is not supported by this `convert.py`. Trying with `convert-hf-to-powerinfer-gguf.py`...")
            script_path = Path(__file__).resolve().parent / "convert-hf-to-powerinfer-gguf.py"
            subprocess.run(["python3", str(script_path.absolute())] + sys.argv[1:])
            return
    # 如果找不到“config.json”文件
    except FileNotFoundError:
        print("Could not find config.json under the original model directory. ", file=sys.stderr)
        sys.exit(1)

    # 如果设置了args.dump_single
    if args.dump_single:
        # 惰性加载模型文件
        model_plus = lazy_load_file(args.model)
        # 执行dump_model函数
        do_dump_model(model_plus)
        return
    # 如果不仅仅是词汇表，则加载一些模型
    if not args.vocab_only:
        model_plus = load_some_model(args.model)
        # 加载模型参数
        params = Params.load(model_plus)
        # 加载MLP预测器模型
        mlp_predictor_plus = load_mlp_model(args.mlp_model)
        # 设置预测器参数
        params.predictor_params = PredictorParams.load(mlp_predictor_plus)
        # 合并多文件模型
        model_plus = merge_multifile_models([model_plus, mlp_predictor_plus])
    else:
        # 如果仅仅是词汇表，则创建一个空的模型
        model_plus = ModelPlus(model = {}, paths = [args.model / 'dummy'], format = 'none', vocab = None)
        # 加载模型参数
        params = Params.load(model_plus)

    # 如果需要转储模型，则执行转储操作并返回
    if args.dump:
        do_dump_model(model_plus)
        return
    # 设置字节序
    endianess = gguf.GGUFEndian.LITTLE
    if args.bigendian:
        endianess = gguf.GGUFEndian.BIG

    # 如果模型参数中的上下文大小为-1，则根据参数或默认值设置上下文大小
    if params.n_ctx == -1:
        if args.ctx is None:
            raise Exception("The model doesn't have a context size, and you didn't specify one with --ctx\n"
                            "Please specify one with --ctx:\n"
                            " - LLaMA v1: --ctx 2048\n"
                            " - LLaMA v2: --ctx 4096\n")
        params.n_ctx = args.ctx

    # 如果指定了输出类型，则根据参数设置文件类型
    if args.outtype:
        params.ftype = {
            "f32": GGMLFileType.AllF32,
            "f16": GGMLFileType.MostlyF16,
            "q8_0": GGMLFileType.MostlyQ8_0,
        }[args.outtype]

    # 打印模型参数
    print(f"params = {params}")

    # 如果仅仅是词汇表，则加载词汇表并写入输出文件
    vocab: Vocab
    if args.vocab_only:
        if not args.outfile:
            raise ValueError("need --outfile if using --vocab-only")
        # FIXME: Try to respect vocab_dir somehow?
        vocab = load_vocab(args.vocab_dir or args.model, args.vocabtype)
        special_vocab = gguf.SpecialVocab(model_plus.paths[0].parent,
            load_merges = args.vocabtype == 'bpe',
            n_vocab = vocab.vocab_size)
        outfile = args.outfile
        OutputFile.write_vocab_only(outfile, params, vocab, special_vocab)
        print(f"Wrote {outfile}")
        return

    # 如果模型中包含词汇表并且未指定词汇表目录，则使用模型中的词汇表
    if model_plus.vocab is not None and args.vocab_dir is None:
        vocab = model_plus.vocab
    # 如果条件不成立，执行以下代码块
    else:
        # 如果参数中有指定词汇表目录，则使用指定的目录，否则使用模型路径的父目录
        vocab_dir = args.vocab_dir if args.vocab_dir else model_plus.paths[0].parent
        # 根据指定的词汇表目录和词汇表类型加载词汇表
        vocab = load_vocab(vocab_dir, args.vocabtype)
    # FIXME: 尝试以某种方式尊重vocab_dir？
    # 创建一个特殊词汇表对象，根据模型路径的父目录和加载的词汇表类型
    special_vocab = gguf.SpecialVocab(model_plus.paths[0].parent,
        load_merges = args.vocabtype == 'bpe',
        n_vocab = vocab.vocab_size)
    # 获取模型对象
    model   = model_plus.model
    # 转换模型名称
    model   = convert_model_names(model, params)
    # 对模型进行后处理转置
    model   = postprocess_transpose(model)
    # 选择输出类型
    ftype   = pick_output_type(model, args.outtype)
    # 将模型转换为指定的输出类型
    model   = convert_to_output_type(model, ftype)
    # 如果没有指定输出文件，则使用默认的输出文件名
    outfile = args.outfile or default_outfile(model_plus.paths, ftype)
    # 设置参数中的输出类型
    params.ftype = ftype
    # 打印输出文件名和格式
    print(f"Writing {outfile}, format {ftype}")
    # 将模型、词汇表、特殊词汇表等信息写入输出文件
    OutputFile.write_all(outfile, ftype, params, model, vocab, special_vocab, concurrency = args.concurrency, endianess=endianess)
    # 打印已写入的文件名
    print(f"Wrote {outfile}")
    # 后处理：写入另一个唯一的文件头，以区分原始的GGUF文件
    with open(outfile, "r+b") as fout:
        # 定义一个魔术数值
        POWERINFER_MAGIC = int.from_bytes(b"PWRI", "little")
        # 将魔术数值以小端字节序写入文件头部
        fout.write(struct.pack("<I", POWERINFER_MAGIC))
# 如果当前模块被直接执行，则调用 main() 函数
if __name__ == '__main__':
    main()
```