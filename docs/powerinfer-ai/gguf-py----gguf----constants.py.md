# `PowerInfer\gguf-py\gguf\constants.py`

```
# 导入必要的模块和类
from __future__ import annotations  # 导入未来版本的注解特性
import sys  # 导入sys模块
from enum import Enum, IntEnum, auto  # 导入枚举相关的类和函数
from typing import Any  # 导入类型提示相关的类和函数

#
# 常量
#

GGUF_MAGIC             = 0x46554747  # 定义GGUF_MAGIC常量，值为0x46554747，即"GGUF"的ASCII码
GGUF_VERSION           = 3  # 定义GGUF_VERSION常量，值为3
GGUF_DEFAULT_ALIGNMENT = 32  # 定义GGUF_DEFAULT_ALIGNMENT常量，值为32

#
# 元数据键
#

# 定义一个类Keys，用于存储元数据的键
class Keys:
# 定义一个 General 类，包含一些通用的属性常量
class General:
    # 架构
    ARCHITECTURE         = "general.architecture"
    # 量化版本
    QUANTIZATION_VERSION = "general.quantization_version"
    # 对齐
    ALIGNMENT            = "general.alignment"
    # 名称
    NAME                 = "general.name"
    # 作者
    AUTHOR               = "general.author"
    # URL
    URL                  = "general.url"
    # 描述
    DESCRIPTION          = "general.description"
    # 许可证
    LICENSE              = "general.license"
    # 源 URL
    SOURCE_URL           = "general.source.url"
    # 源 Huggingface 仓库
    SOURCE_HF_REPO       = "general.source.huggingface.repository"
    # 文件类型
    FILE_TYPE            = "general.file_type"

# 定义一个 LLM 类，包含一些语言模型相关的属性常量
class LLM:
    # 上下文长度
    CONTEXT_LENGTH        = "{arch}.context_length"
    # 嵌入长度
    EMBEDDING_LENGTH      = "{arch}.embedding_length"
    # 块数量
    BLOCK_COUNT           = "{arch}.block_count"
    # 前馈长度
    FEED_FORWARD_LENGTH   = "{arch}.feed_forward_length"
    # 使用并行残差
    USE_PARALLEL_RESIDUAL = "{arch}.use_parallel_residual"
    # 张量数据布局
    TENSOR_DATA_LAYOUT    = "{arch}.tensor_data_layout"
# 定义了 Attention 类，包含了一些与注意力机制相关的参数
class Attention:
    # 注意力头的数量
    HEAD_COUNT = "{arch}.attention.head_count"
    # 键值对注意力头的数量
    HEAD_COUNT_KV = "{arch}.attention.head_count_kv"
    # 最大的偏差
    MAX_ALIBI_BIAS = "{arch}.attention.max_alibi_bias"
    # 限制键、值、查询的范围
    CLAMP_KQV = "{arch}.attention.clamp_kqv"
    # 层归一化的 epsilon
    LAYERNORM_EPS = "{arch}.attention.layer_norm_epsilon"
    # 层归一化的 RMS epsilon
    LAYERNORM_RMS_EPS = "{arch}.attention.layer_norm_rms_epsilon"

# 定义了 Rope 类，包含了一些与绳索相关的参数
class Rope:
    # 维度数量
    DIMENSION_COUNT = "{arch}.rope.dimension_count"
    # 频率基数
    FREQ_BASE = "{arch}.rope.freq_base"
    # 缩放类型
    SCALING_TYPE = "{arch}.rope.scaling.type"
    # 缩放因子
    SCALING_FACTOR = "{arch}.rope.scaling.factor"
    # 缩放原始上下文长度
    SCALING_ORIG_CTX_LEN = "{arch}.rope.scaling.original_context_length"
    # 调整过的缩放
    SCALING_FINETUNED = "{arch}.rope.scaling.finetuned"

# 定义了 Tokenizer 类，包含了一些与分词器相关的参数
class Tokenizer:
    # 模型
    MODEL = "tokenizer.ggml.model"
    # 分词器的列表
    LIST = "tokenizer.ggml.tokens"
# 定义不同类型的标记器的键值
TOKEN_TYPE = "tokenizer.ggml.token_type"
# 定义分数的键值
SCORES     = "tokenizer.ggml.scores"
# 定义合并的键值
MERGES     = "tokenizer.ggml.merges"
# 定义开始符号的键值
BOS_ID     = "tokenizer.ggml.bos_token_id"
# 定义结束符号的键值
EOS_ID     = "tokenizer.ggml.eos_token_id"
# 定义未知符号的键值
UNK_ID     = "tokenizer.ggml.unknown_token_id"
# 定义分隔符的键值
SEP_ID     = "tokenizer.ggml.seperator_token_id"
# 定义填充符号的键值
PAD_ID     = "tokenizer.ggml.padding_token_id"
# 定义是否添加开始符号的键值
ADD_BOS    = "tokenizer.ggml.add_bos_token"
# 定义是否添加结束符号的键值
ADD_EOS    = "tokenizer.ggml.add_eos_token"
# 定义 Huggingface JSON 的键值
HF_JSON    = "tokenizer.huggingface.json"
# 定义 RWKV 的键值
RWKV       = "tokenizer.rwkv.world"

# 定义 PowerInfer 类的稀疏阈值的键值
class PowerInfer:
    SPARSE_THRESHOLD = "powerinfer.sparse_threshold"

# 定义 Split 类的 VRAM 容量的键值
class Split:
    VRAM_CAPACITY = "split.vram_capacity"
# 定义了一个枚举类，用于存储模型张量名称的推荐映射
# 枚举类中包含了各个模型的名称，每个名称都对应一个自增的整数值
# 这些整数值可以用于存储模型张量名称的映射关系
# 定义了一个枚举类 MODEL_TENSOR，列举了不同类型的张量模型
TOKEN_EMBD      = auto()  # 令牌嵌入
TOKEN_EMBD_NORM = auto()  # 规范化的令牌嵌入
TOKEN_TYPES     = auto()  # 令牌类型
POS_EMBD        = auto()  # 位置嵌入
OUTPUT          = auto()  # 输出
OUTPUT_NORM     = auto()  # 规范化的输出
ROPE_FREQS      = auto()  # ROPE 频率
ATTN_Q          = auto()  # 注意力查询
ATTN_K          = auto()  # 注意力键
ATTN_V          = auto()  # 注意力值
ATTN_QKV        = auto()  # 注意力查询、键、值
ATTN_OUT        = auto()  # 注意力输出
ATTN_NORM       = auto()  # 规范化的注意力
ATTN_NORM_2     = auto()  # 第二个规范化的注意力
ATTN_ROT_EMBD   = auto()  # 注意力旋转嵌入
FFN_GATE        = auto()  # 前馈网络门控
FFN_DOWN        = auto()  # 前馈网络下采样
FFN_UP          = auto()  # 前馈网络上采样
# 定义枚举类型的自动编号
FFN_NORM        = auto()  # 自动编号
ATTN_Q_NORM     = auto()  # 自动编号
ATTN_K_NORM     = auto()  # 自动编号
FFN_DOWN_T      = auto()  # 自动编号
FC_1            = auto()  # 自动编号
FC_2            = auto()  # 自动编号

# 定义模型架构名称和枚举类型的对应关系
MODEL_ARCH_NAMES: dict[MODEL_ARCH, str] = {
    MODEL_ARCH.LLAMA:          "llama",      # 模型架构LLAMA对应的名称
    MODEL_ARCH.FALCON:         "falcon",     # 模型架构FALCON对应的名称
    MODEL_ARCH.BAICHUAN:       "baichuan",   # 模型架构BAICHUAN对应的名称
    MODEL_ARCH.GPT2:           "gpt2",       # 模型架构GPT2对应的名称
    MODEL_ARCH.GPTJ:           "gptj",       # 模型架构GPTJ对应的名称
    MODEL_ARCH.GPTNEOX:        "gptneox",    # 模型架构GPTNEOX对应的名称
    MODEL_ARCH.MPT:            "mpt",        # 模型架构MPT对应的名称
    MODEL_ARCH.STARCODER:      "starcoder",  # 模型架构STARCODER对应的名称
    MODEL_ARCH.PERSIMMON:      "persimmon",  # 模型架构PERSIMMON对应的名称
    MODEL_ARCH.REFACT:         "refact",     # 模型架构REFACT对应的名称
}
# 定义模型架构的常量映射
MODEL_ARCH = {
    MODEL_ARCH.BERT:           "bert",  # BERT模型架构
    MODEL_ARCH.BLOOM:          "bloom",  # Bloom模型架构
    MODEL_ARCH.STABLELM:       "stablelm",  # StableLM模型架构
}

# 定义模型张量的常量映射
TENSOR_NAMES: dict[MODEL_TENSOR, str] = {
    MODEL_TENSOR.TOKEN_EMBD:      "token_embd",  # token embedding张量
    MODEL_TENSOR.TOKEN_EMBD_NORM: "token_embd_norm",  # 规范化的token embedding张量
    MODEL_TENSOR.TOKEN_TYPES:     "token_types",  # token类型张量
    MODEL_TENSOR.POS_EMBD:        "position_embd",  # 位置embedding张量
    MODEL_TENSOR.OUTPUT_NORM:     "output_norm",  # 输出规范化张量
    MODEL_TENSOR.OUTPUT:          "output",  # 输出张量
    MODEL_TENSOR.ROPE_FREQS:      "rope_freqs",  # ROPE频率张量
    MODEL_TENSOR.ATTN_NORM:       "blk.{bid}.attn_norm",  # 注意力规范化张量
    MODEL_TENSOR.ATTN_NORM_2:     "blk.{bid}.attn_norm_2",  # 第二个注意力规范化张量
    MODEL_TENSOR.ATTN_QKV:        "blk.{bid}.attn_qkv",  # 注意力的QKV张量
    MODEL_TENSOR.ATTN_Q:          "blk.{bid}.attn_q",  # 注意力的Q张量
    MODEL_TENSOR.ATTN_K:          "blk.{bid}.attn_k",  # 注意力的K张量
    MODEL_TENSOR.ATTN_V:          "blk.{bid}.attn_v",  # 注意力的V张量
    MODEL_TENSOR.ATTN_OUT:        "blk.{bid}.attn_output",  # 注意力的输出张量
}
# 定义了一个字典，键为MODEL_TENSOR中的常量，值为对应的字符串模板
MODEL_TENSOR.ATTN_ROT_EMBD:   "blk.{bid}.attn_rot_embd",
MODEL_TENSOR.ATTN_Q_NORM:     "blk.{bid}.attn_q_norm",
MODEL_TENSOR.ATTN_K_NORM:     "blk.{bid}.attn_k_norm",
MODEL_TENSOR.FFN_NORM:        "blk.{bid}.ffn_norm",
MODEL_TENSOR.FFN_GATE:        "blk.{bid}.ffn_gate",
MODEL_TENSOR.FFN_DOWN:        "blk.{bid}.ffn_down",
MODEL_TENSOR.FFN_UP:          "blk.{bid}.ffn_up",
MODEL_TENSOR.FFN_DOWN_T:      "blk.{bid}.ffn_down_t",
MODEL_TENSOR.FC_1:            "blk.{bid}.fc1",
MODEL_TENSOR.FC_2:            "blk.{bid}.fc2",
}

# 定义了一个字典，键为MODEL_ARCH中的常量，值为对应的MODEL_TENSOR常量列表
MODEL_TENSORS: dict[MODEL_ARCH, list[MODEL_TENSOR]] = {
    MODEL_ARCH.LLAMA: [
        MODEL_TENSOR.TOKEN_EMBD,
        MODEL_TENSOR.OUTPUT_NORM,
        MODEL_TENSOR.OUTPUT,
        MODEL_TENSOR.ROPE_FREQS,
        MODEL_TENSOR.ATTN_NORM,
        MODEL_TENSOR.ATTN_Q,
# 定义了不同模型架构下的不同张量（tensor）名称
MODEL_TENSOR.ATTN_K,  # 注意力机制中的K张量
MODEL_TENSOR.ATTN_V,  # 注意力机制中的V张量
MODEL_TENSOR.ATTN_OUT,  # 注意力机制的输出张量
MODEL_TENSOR.ATTN_ROT_EMBD,  # 注意力机制的旋转嵌入张量
MODEL_TENSOR.FFN_NORM,  # 前馈神经网络的归一化张量
MODEL_TENSOR.FFN_GATE,  # 前馈神经网络的门控张量
MODEL_TENSOR.FFN_DOWN,  # 前馈神经网络的下采样张量
MODEL_TENSOR.FFN_UP,  # 前馈神经网络的上采样张量
MODEL_TENSOR.FFN_DOWN_T,  # 前馈神经网络的下采样转置张量
MODEL_TENSOR.FC_1,  # 全连接层1的张量
MODEL_TENSOR.FC_2,  # 全连接层2的张量
MODEL_ARCH.GPTNEOX: [  # GPTNEOX模型架构下的张量名称列表
    MODEL_TENSOR.TOKEN_EMBD,  # 令牌嵌入张量
    MODEL_TENSOR.OUTPUT_NORM,  # 输出的归一化张量
    MODEL_TENSOR.OUTPUT,  # 输出张量
    MODEL_TENSOR.ATTN_NORM,  # 注意力机制的归一化张量
    MODEL_TENSOR.ATTN_QKV,  # 注意力机制的查询、键、值张量
    MODEL_TENSOR.ATTN_OUT,  # 注意力机制的输出张量
    MODEL_TENSOR.FFN_NORM,  # 前馈神经网络的归一化张量
# 定义了不同模型架构对应的张量列表
MODEL_ARCH.BERT: [
    MODEL_TENSOR.TOKEN_EMBD,  # 词嵌入张量
    MODEL_TENSOR.OUTPUT_NORM,  # 输出规范化张量
    MODEL_TENSOR.OUTPUT,  # 输出张量
    MODEL_TENSOR.ATTN_NORM,  # 注意力规范化张量
    MODEL_TENSOR.ATTN_QKV,  # 注意力查询、键、值张量
    MODEL_TENSOR.ATTN_OUT,  # 注意力输出张量
    MODEL_TENSOR.FFN_DOWN,  # 前馈神经网络下行张量
    MODEL_TENSOR.FFN_UP,  # 前馈神经网络上行张量
],
MODEL_ARCH.FALCON: [
    MODEL_TENSOR.TOKEN_EMBD,  # 词嵌入张量
    MODEL_TENSOR.OUTPUT_NORM,  # 输出规范化张量
    MODEL_TENSOR.OUTPUT,  # 输出张量
    MODEL_TENSOR.ATTN_NORM,  # 注意力规范化张量
    MODEL_TENSOR.ATTN_NORM_2,  # 第二个注意力规范化张量
    MODEL_TENSOR.ATTN_QKV,  # 注意力查询、键、值张量
    MODEL_TENSOR.ATTN_OUT,  # 注意力输出张量
    MODEL_TENSOR.FFN_DOWN,  # 前馈神经网络下行张量
    MODEL_TENSOR.FFN_UP,  # 前馈神经网络上行张量
],
MODEL_ARCH.BAICHUAN: [
    MODEL_TENSOR.TOKEN_EMBD,  # 词嵌入张量
    MODEL_TENSOR.OUTPUT_NORM,  # 输出规范化张量
    MODEL_TENSOR.OUTPUT,  # 输出张量
    MODEL_TENSOR.ROPE_FREQS,  # 绳索频率张量
    MODEL_TENSOR.ATTN_NORM,  # 注意力规范化张量
],
# 定义了一个包含模型张量的列表，包括注意力机制的查询、键、值、输出等张量
MODEL_TENSOR.ATTN_Q,
MODEL_TENSOR.ATTN_K,
MODEL_TENSOR.ATTN_V,
MODEL_TENSOR.ATTN_OUT,
MODEL_TENSOR.ATTN_ROT_EMBD,
MODEL_TENSOR.FFN_NORM,
MODEL_TENSOR.FFN_GATE,
MODEL_TENSOR.FFN_DOWN,
MODEL_TENSOR.FFN_UP,

# 定义了另一个包含模型张量的列表，包括词嵌入、位置嵌入、输出规范化、输出等张量
MODEL_ARCH.STARCODER: [
    MODEL_TENSOR.TOKEN_EMBD,
    MODEL_TENSOR.POS_EMBD,
    MODEL_TENSOR.OUTPUT_NORM,
    MODEL_TENSOR.OUTPUT,
    MODEL_TENSOR.ATTN_NORM,
    MODEL_TENSOR.ATTN_QKV,
    MODEL_TENSOR.ATTN_OUT,
    MODEL_TENSOR.FFN_NORM,
    MODEL_TENSOR.FFN_DOWN,
# 定义了不同模型架构对应的张量列表
MODEL_ARCH.BERT: [
    # BERT 模型对应的张量列表
    MODEL_TENSOR.TOKEN_EMBD,  # Token embedding
    MODEL_TENSOR.TOKEN_TYPES,  # Token types
    MODEL_TENSOR.POS_EMBD,  # Positional embedding
    MODEL_TENSOR.OUTPUT_NORM,  # Output normalization
    MODEL_TENSOR.ATTN_NORM,  # Attention normalization
    MODEL_TENSOR.ATTN_Q,  # Attention query
    MODEL_TENSOR.ATTN_K,  # Attention key
    MODEL_TENSOR.ATTN_V,  # Attention value
    MODEL_TENSOR.ATTN_OUT,  # Attention output
    MODEL_TENSOR.FFN_NORM,  # Feed-forward network normalization
    MODEL_TENSOR.FFN_DOWN,  # Feed-forward network down
    MODEL_TENSOR.FFN_UP,  # Feed-forward network up
],
MODEL_ARCH.MPT: [
    # MPT 模型对应的张量列表
    MODEL_TENSOR.TOKEN_EMBD,  # Token embedding
    MODEL_TENSOR.OUTPUT_NORM,  # Output normalization
    MODEL_TENSOR.OUTPUT,  # Output
],
# 定义了不同模型架构下的不同张量类型
MODEL_TENSOR.ATTN_NORM, MODEL_TENSOR.ATTN_QKV, MODEL_TENSOR.ATTN_OUT, MODEL_TENSOR.FFN_NORM, MODEL_TENSOR.FFN_DOWN, MODEL_TENSOR.FFN_UP
# GPTJ 模型架构下的张量类型
MODEL_ARCH.GPTJ: [
    MODEL_TENSOR.TOKEN_EMBD, MODEL_TENSOR.OUTPUT_NORM, MODEL_TENSOR.OUTPUT, MODEL_TENSOR.ATTN_NORM, MODEL_TENSOR.ATTN_Q, MODEL_TENSOR.ATTN_K, MODEL_TENSOR.ATTN_V, MODEL_TENSOR.ATTN_OUT, MODEL_TENSOR.FFN_DOWN, MODEL_TENSOR.FFN_UP
],
# PERSIMMON 模型架构下的张量类型
MODEL_ARCH.PERSIMMON: [
# 定义了一系列模型张量的常量，包括TOKEN_EMBD, OUTPUT, OUTPUT_NORM等等
MODEL_TENSOR = {
    MODEL_ARCH.ORIG: [
        MODEL_TENSOR.TOKEN_EMBD,  # 模型张量中的token嵌入
        MODEL_TENSOR.OUTPUT,  # 模型张量中的输出
        MODEL_TENSOR.OUTPUT_NORM,  # 模型张量中的输出规范化
        MODEL_TENSOR.ATTN_NORM,  # 模型张量中的注意力规范化
        MODEL_TENSOR.ATTN_QKV,  # 模型张量中的注意力查询键值
        MODEL_TENSOR.ATTN_OUT,  # 模型张量中的注意力输出
        MODEL_TENSOR.FFN_NORM,  # 模型张量中的前馈网络规范化
        MODEL_TENSOR.FFN_DOWN,  # 模型张量中的前馈网络下采样
        MODEL_TENSOR.FFN_UP,  # 模型张量中的前馈网络上采样
        MODEL_TENSOR.ATTN_Q_NORM,  # 模型张量中的注意力查询规范化
        MODEL_TENSOR.ATTN_K_NORM,  # 模型张量中的注意力键规范化
        MODEL_TENSOR.ATTN_ROT_EMBD,  # 模型张量中的注意力旋转嵌入
    ],
    MODEL_ARCH.REFACT: [
        MODEL_TENSOR.TOKEN_EMBD,  # 重构后的模型张量中的token嵌入
        MODEL_TENSOR.OUTPUT_NORM,  # 重构后的模型张量中的输出规范化
        MODEL_TENSOR.OUTPUT,  # 重构后的模型张量中的输出
        MODEL_TENSOR.ATTN_NORM,  # 重构后的模型张量中的注意力规范化
        MODEL_TENSOR.ATTN_Q,  # 重构后的模型张量中的注意力查询
        MODEL_TENSOR.ATTN_K,  # 重构后的模型张量中的注意力键
    ]
}
# 定义了两种模型架构的常量，包括ORIG和REFACT
MODEL_ARCH = {
    'ORIG': 'original',  # 原始模型架构
    'REFACT': 'refactored'  # 重构后的模型架构
}
# 定义了不同模型架构下的张量列表
MODEL_TENSOR.ATTN_V, MODEL_TENSOR.ATTN_OUT, MODEL_TENSOR.FFN_NORM, MODEL_TENSOR.FFN_GATE, MODEL_TENSOR.FFN_DOWN, MODEL_TENSOR.FFN_UP
# 定义了不同模型架构下的张量列表
MODEL_ARCH.BLOOM: [
    MODEL_TENSOR.TOKEN_EMBD, MODEL_TENSOR.TOKEN_EMBD_NORM, MODEL_TENSOR.OUTPUT_NORM, MODEL_TENSOR.OUTPUT, MODEL_TENSOR.ATTN_NORM, MODEL_TENSOR.ATTN_QKV, MODEL_TENSOR.ATTN_OUT, MODEL_TENSOR.FFN_NORM, MODEL_TENSOR.FFN_DOWN, MODEL_TENSOR.FFN_UP
# 定义了不同模型架构下的张量列表
MODEL_ARCH.STABLELM: [
# 定义了一系列模型张量的常量，包括TOKEN_EMBD, OUTPUT_NORM, OUTPUT等等
# 这些常量可能代表了模型的不同部分或者特征
# 在不同的模型架构下，可能会有不同的张量常量
# 每个模型架构下可能还有一些待完成的TODO项
# 定义一个字典，用于存储不会被序列化的张量
MODEL_TENSOR_SKIP: dict[MODEL_ARCH, list[MODEL_TENSOR]] = {
    # 对于LLAMA模型，不会被序列化的张量有ROPE_FREQS和ATTN_ROT_EMBD
    MODEL_ARCH.LLAMA: [
        MODEL_TENSOR.ROPE_FREQS,
        MODEL_TENSOR.ATTN_ROT_EMBD,
    ],
    # 对于BAICHUAN模型，不会被序列化的张量有ROPE_FREQS和ATTN_ROT_EMBD
    MODEL_ARCH.BAICHUAN: [
        MODEL_TENSOR.ROPE_FREQS,
        MODEL_TENSOR.ATTN_ROT_EMBD,
    ],
    # 对于PERSIMMON模型，不会被序列化的张量有ROPE_FREQS
    MODEL_ARCH.PERSIMMON: [
        MODEL_TENSOR.ROPE_FREQS,
    ],
}

#
# types
#
# 定义一个枚举类 TokenType，包含 NORMAL、UNKNOWN、CONTROL、USER_DEFINED、UNUSED、BYTE 六种类型
class TokenType(IntEnum):
    NORMAL       = 1
    UNKNOWN      = 2
    CONTROL      = 3
    USER_DEFINED = 4
    UNUSED       = 5
    BYTE         = 6

# 定义一个枚举类 RopeScalingType，包含 NONE、LINEAR、YARN 三种类型，类型为字符串
class RopeScalingType(Enum):
    NONE   = 'none'
    LINEAR = 'linear'
    YARN   = 'yarn'

# 定义一个枚举类 GGMLQuantizationType，包含 F32、F16、Q4_0、Q4_1 四种类型
class GGMLQuantizationType(IntEnum):
    F32  = 0
    F16  = 1
    Q4_0 = 2
    Q4_1 = 3
# 定义一系列变量，用于存储不同的数值
Q5_0 = 6
Q5_1 = 7
Q8_0 = 8
Q8_1 = 9
Q2_K = 10
Q3_K = 11
Q4_K = 12
Q5_K = 13
Q6_K = 14
Q8_K = 15
I8 = 16,  # 定义一个8位整数变量
I16 = 17  # 定义一个16位整数变量
I32 = 18  # 定义一个32位整数变量

# 定义一个枚举类型，表示字节序
class GGUFEndian(IntEnum):
    LITTLE = 0  # 小端字节序
    BIG = 1  # 大端字节序
# 定义一个枚举类 GGUFValueType，包含不同数值类型的枚举值
class GGUFValueType(IntEnum):
    UINT8   = 0
    INT8    = 1
    UINT16  = 2
    INT16   = 3
    UINT32  = 4
    INT32   = 5
    FLOAT32 = 6
    BOOL    = 7
    STRING  = 8
    ARRAY   = 9
    UINT64  = 10
    INT64   = 11
    FLOAT64 = 12

    # 静态方法，根据输入的值返回对应的 GGUFValueType 枚举值
    @staticmethod
    def get_type(val: Any) -> GGUFValueType:
        # 如果输入值是字符串、字节串或字节数组，则返回字符串类型的枚举值
        if isinstance(val, (str, bytes, bytearray)):
            return GGUFValueType.STRING
        # 如果输入值是列表，则返回数组类型的枚举值
        elif isinstance(val, list):
# 如果值是数组类型，则返回数组类型枚举值
        return GGUFValueType.ARRAY
    # 如果值是浮点数类型，则返回浮点数类型枚举值
        elif isinstance(val, float):
            return GGUFValueType.FLOAT32
    # 如果值是布尔类型，则返回布尔类型枚举值
        elif isinstance(val, bool):
            return GGUFValueType.BOOL
    # 如果值是整数类型，则返回整数类型枚举值
        elif isinstance(val, int):
            return GGUFValueType.INT32
    # 如果值是其他类型，则打印未知类型并退出程序
        else:
            print("Unknown type:", type(val))
            sys.exit()


# 注意：不支持GGML_QKK_64
QK_K = 256
# 这里的项目是（块大小，类型大小）
GGML_QUANT_SIZES = {
    GGMLQuantizationType.F32:  (1, 4),
    GGMLQuantizationType.F16:  (1, 2),
    GGMLQuantizationType.Q4_0: (32, 2 + 16),
```
# 定义了一个常量QK_K，并赋值为256
# 定义了一个字典GGML_QUANT_SIZES，包含了不同GGMLQuantizationType对应的块大小和类型大小
# 定义了不同的量化类型对应的参数元组，每个元组包含了不同的数值
GGMLQuantizationType.Q4_1: (32, 2 + 2 + 16),
GGMLQuantizationType.Q5_0: (32, 2 + 4 + 16),
GGMLQuantizationType.Q5_1: (32, 2 + 2 + 4 + 16),
GGMLQuantizationType.Q8_0: (32, 2 + 32),
GGMLQuantizationType.Q8_1: (32, 4 + 4 + 32),
GGMLQuantizationType.Q2_K: (256, 2 + 2 + QK_K // 16 + QK_K // 4),
GGMLQuantizationType.Q3_K: (256, 2 + QK_K // 4 + QK_K // 8 + 12),
GGMLQuantizationType.Q4_K: (256, 2 + 2 + QK_K // 2 + 12),
GGMLQuantizationType.Q5_K: (256, 2 + 2 + QK_K // 2 + QK_K // 8 + 12),
GGMLQuantizationType.Q6_K: (256, 2 + QK_K // 2 + QK_K // 4 + QK_K // 16),
GGMLQuantizationType.Q8_K: (256, 4 + QK_K + QK_K // 8),

# 为了向后兼容，定义了一些通用的键的别名
# 通用
KEY_GENERAL_ARCHITECTURE         = Keys.General.ARCHITECTURE
KEY_GENERAL_QUANTIZATION_VERSION = Keys.General.QUANTIZATION_VERSION
KEY_GENERAL_ALIGNMENT            = Keys.General.ALIGNMENT
# 定义通用键的名称
KEY_GENERAL_NAME                 = Keys.General.NAME
# 定义通用键的作者
KEY_GENERAL_AUTHOR               = Keys.General.AUTHOR
# 定义通用键的 URL
KEY_GENERAL_URL                  = Keys.General.URL
# 定义通用键的描述
KEY_GENERAL_DESCRIPTION          = Keys.General.DESCRIPTION
# 定义通用键的许可证
KEY_GENERAL_LICENSE              = Keys.General.LICENSE
# 定义通用键的源 URL
KEY_GENERAL_SOURCE_URL           = Keys.General.SOURCE_URL
# 定义通用键的源 HF 仓库
KEY_GENERAL_SOURCE_HF_REPO       = Keys.General.SOURCE_HF_REPO
# 定义通用键的文件类型
KEY_GENERAL_FILE_TYPE            = Keys.General.FILE_TYPE

# LLM
# 定义 LLM 的上下文长度
KEY_CONTEXT_LENGTH        = Keys.LLM.CONTEXT_LENGTH
# 定义 LLM 的嵌入长度
KEY_EMBEDDING_LENGTH      = Keys.LLM.EMBEDDING_LENGTH
# 定义 LLM 的块数量
KEY_BLOCK_COUNT           = Keys.LLM.BLOCK_COUNT
# 定义 LLM 的前馈长度
KEY_FEED_FORWARD_LENGTH   = Keys.LLM.FEED_FORWARD_LENGTH
# 定义 LLM 是否使用并行残差
KEY_USE_PARALLEL_RESIDUAL = Keys.LLM.USE_PARALLEL_RESIDUAL
# 定义 LLM 的张量数据布局
KEY_TENSOR_DATA_LAYOUT    = Keys.LLM.TENSOR_DATA_LAYOUT

# attention
# 定义注意力机制的头数量
KEY_ATTENTION_HEAD_COUNT        = Keys.Attention.HEAD_COUNT
# 定义注意力机制的键值头数量
KEY_ATTENTION_HEAD_COUNT_KV     = Keys.Attention.HEAD_COUNT_KV
# 定义注意力机制的最大偏差键
KEY_ATTENTION_MAX_ALIBI_BIAS = Keys.Attention.MAX_ALIBI_BIAS
# 定义注意力机制的KQV值的限制键
KEY_ATTENTION_CLAMP_KQV = Keys.Attention.CLAMP_KQV
# 定义注意力机制的LayerNorm的epsilon值键
KEY_ATTENTION_LAYERNORM_EPS = Keys.Attention.LAYERNORM_EPS
# 定义注意力机制的LayerNorm的RMS epsilon值键
KEY_ATTENTION_LAYERNORM_RMS_EPS = Keys.Attention.LAYERNORM_RMS_EPS

# RoPE
# 定义RoPE的维度计数键
KEY_ROPE_DIMENSION_COUNT = Keys.Rope.DIMENSION_COUNT
# 定义RoPE的基础频率键
KEY_ROPE_FREQ_BASE = Keys.Rope.FREQ_BASE
# 定义RoPE的缩放类型键
KEY_ROPE_SCALING_TYPE = Keys.Rope.SCALING_TYPE
# 定义RoPE的缩放因子键
KEY_ROPE_SCALING_FACTOR = Keys.Rope.SCALING_FACTOR
# 定义RoPE的原始上下文长度键
KEY_ROPE_SCALING_ORIG_CTX_LEN = Keys.Rope.SCALING_ORIG_CTX_LEN
# 定义RoPE的微调缩放键
KEY_ROPE_SCALING_FINETUNED = Keys.Rope.SCALING_FINETUNED

# tokenization
# 定义分词器的模型键
KEY_TOKENIZER_MODEL = Keys.Tokenizer.MODEL
# 定义分词器的列表键
KEY_TOKENIZER_LIST = Keys.Tokenizer.LIST
# 定义分词器的token类型键
KEY_TOKENIZER_TOKEN_TYPE = Keys.Tokenizer.TOKEN_TYPE
# 定义分词器的分数键
KEY_TOKENIZER_SCORES = Keys.Tokenizer.SCORES
# 定义分词器的合并键
KEY_TOKENIZER_MERGES = Keys.Tokenizer.MERGES
# 定义分词器的BOS ID键
KEY_TOKENIZER_BOS_ID = Keys.Tokenizer.BOS_ID
# 定义常量，表示分词器的结束符号的 ID
KEY_TOKENIZER_EOS_ID     = Keys.Tokenizer.EOS_ID
# 定义常量，表示分词器的未知符号的 ID
KEY_TOKENIZER_UNK_ID     = Keys.Tokenizer.UNK_ID
# 定义常量，表示分词器的分隔符号的 ID
KEY_TOKENIZER_SEP_ID     = Keys.Tokenizer.SEP_ID
# 定义常量，表示分词器的填充符号的 ID
KEY_TOKENIZER_PAD_ID     = Keys.Tokenizer.PAD_ID
# 定义常量，表示分词器的 HF_JSON
KEY_TOKENIZER_HF_JSON    = Keys.Tokenizer.HF_JSON
# 定义常量，表示分词器的 RWKV
KEY_TOKENIZER_RWKV       = Keys.Tokenizer.RWKV
```