# `PowerInfer\gguf-py\gguf\constants.py`

```
# 导入必要的模块
from __future__ import annotations
import sys
from enum import Enum, IntEnum, auto
from typing import Any

# 定义常量
GGUF_MAGIC             = 0x46554747  # "GGUF"，文件标识
GGUF_VERSION           = 3  # 文件版本号
GGUF_DEFAULT_ALIGNMENT = 32  # 默认对齐值

# 元数据键值
class Keys:
    class General:
        ARCHITECTURE         = "general.architecture"  # 通用架构
        QUANTIZATION_VERSION = "general.quantization_version"  # 量化版本
        ALIGNMENT            = "general.alignment"  # 对齐
        NAME                 = "general.name"  # 名称
        AUTHOR               = "general.author"  # 作者
        URL                  = "general.url"  # URL
        DESCRIPTION          = "general.description"  # 描述
        LICENSE              = "general.license"  # 许可证
        SOURCE_URL           = "general.source.url"  # 源URL
        SOURCE_HF_REPO       = "general.source.huggingface.repository"  # 源Hugging Face仓库
        FILE_TYPE            = "general.file_type"  # 文件类型

    class LLM:
        CONTEXT_LENGTH        = "{arch}.context_length"  # 上下文长度
        EMBEDDING_LENGTH      = "{arch}.embedding_length"  # 嵌入长度
        BLOCK_COUNT           = "{arch}.block_count"  # 块数量
        FEED_FORWARD_LENGTH   = "{arch}.feed_forward_length"  # 前馈长度
        USE_PARALLEL_RESIDUAL = "{arch}.use_parallel_residual"  # 使用并行残差
        TENSOR_DATA_LAYOUT    = "{arch}.tensor_data_layout"  # 张量数据布局

    class Attention:
        HEAD_COUNT        = "{arch}.attention.head_count"  # 注意力头数量
        HEAD_COUNT_KV     = "{arch}.attention.head_count_kv"  # 注意力头数量（KV）
        MAX_ALIBI_BIAS    = "{arch}.attention.max_alibi_bias"  # 最大偏差
        CLAMP_KQV         = "{arch}.attention.clamp_kqv"  # 限制KQV
        LAYERNORM_EPS     = "{arch}.attention.layer_norm_epsilon"  # 层归一化ε
        LAYERNORM_RMS_EPS = "{arch}.attention.layer_norm_rms_epsilon"  # 层归一化RMSε
    # 定义 Rope 类，包含绳索的维度计数、频率基数、缩放类型、缩放因子、缩放原始上下文长度、缩放微调
    class Rope:
        DIMENSION_COUNT      = "{arch}.rope.dimension_count"
        FREQ_BASE            = "{arch}.rope.freq_base"
        SCALING_TYPE         = "{arch}.rope.scaling.type"
        SCALING_FACTOR       = "{arch}.rope.scaling.factor"
        SCALING_ORIG_CTX_LEN = "{arch}.rope.scaling.original_context_length"
        SCALING_FINETUNED    = "{arch}.rope.scaling.finetuned"
    
    # 定义 Tokenizer 类，包含分词器的模型、列表、token 类型、分数、合并、起始符号 ID、结束符号 ID、未知符号 ID、分隔符号 ID、填充符号 ID、添加起始符号、添加结束符号、Huggingface JSON、RWKV
    class Tokenizer:
        MODEL      = "tokenizer.ggml.model"
        LIST       = "tokenizer.ggml.tokens"
        TOKEN_TYPE = "tokenizer.ggml.token_type"
        SCORES     = "tokenizer.ggml.scores"
        MERGES     = "tokenizer.ggml.merges"
        BOS_ID     = "tokenizer.ggml.bos_token_id"
        EOS_ID     = "tokenizer.ggml.eos_token_id"
        UNK_ID     = "tokenizer.ggml.unknown_token_id"
        SEP_ID     = "tokenizer.ggml.seperator_token_id"
        PAD_ID     = "tokenizer.ggml.padding_token_id"
        ADD_BOS    = "tokenizer.ggml.add_bos_token"
        ADD_EOS    = "tokenizer.ggml.add_eos_token"
        HF_JSON    = "tokenizer.huggingface.json"
        RWKV       = "tokenizer.rwkv.world"
    
    # 定义 PowerInfer 类，包含稀疏阈值
    class PowerInfer:
        SPARSE_THRESHOLD = "powerinfer.sparse_threshold"
    
    # 定义 Split 类，包含 VRAM 容量
    class Split:
        VRAM_CAPACITY = "split.vram_capacity"
# 定义了模型架构的枚举类，用于表示不同的模型架构
class MODEL_ARCH(IntEnum):
    # 自动分配枚举值
    LLAMA     = auto()
    FALCON    = auto()
    BAICHUAN  = auto()
    GPT2      = auto()
    GPTJ      = auto()
    GPTNEOX   = auto()
    MPT       = auto()
    STARCODER = auto()
    PERSIMMON = auto()
    REFACT    = auto()
    BERT      = auto()
    BLOOM     = auto()
    STABLELM  = auto()

# 定义了模型张量的枚举类，用于表示不同的模型张量
class MODEL_TENSOR(IntEnum):
    # 自动分配枚举值
    TOKEN_EMBD      = auto()
    TOKEN_EMBD_NORM = auto()
    TOKEN_TYPES     = auto()
    POS_EMBD        = auto()
    OUTPUT          = auto()
    OUTPUT_NORM     = auto()
    # ... 其他模型张量

# 定义了模型架构名称到枚举值的映射字典
MODEL_ARCH_NAMES: dict[MODEL_ARCH, str] = {
    MODEL_ARCH.LLAMA:          "llama",
    MODEL_ARCH.FALCON:         "falcon",
    MODEL_ARCH.BAICHUAN:       "baichuan",
    # ... 其他模型架构名称映射

# 定义了模型张量名称到枚举值的映射字典
TENSOR_NAMES: dict[MODEL_TENSOR, str] = {
    MODEL_TENSOR.TOKEN_EMBD:      "token_embd",
    MODEL_TENSOR.TOKEN_EMBD_NORM: "token_embd_norm",
    MODEL_TENSOR.TOKEN_TYPES:     "token_types",
    # ... 其他模型张量名称映射
    # 模型张量输出
    MODEL_TENSOR.OUTPUT:          "output",
    # 模型张量绳索频率
    MODEL_TENSOR.ROPE_FREQS:      "rope_freqs",
    # 模型张量注意力规范化
    MODEL_TENSOR.ATTN_NORM:       "blk.{bid}.attn_norm",
    # 模型张量第二层注意力规范化
    MODEL_TENSOR.ATTN_NORM_2:     "blk.{bid}.attn_norm_2",
    # 模型张量注意力查询、键、值
    MODEL_TENSOR.ATTN_QKV:        "blk.{bid}.attn_qkv",
    MODEL_TENSOR.ATTN_Q:          "blk.{bid}.attn_q",
    MODEL_TENSOR.ATTN_K:          "blk.{bid}.attn_k",
    MODEL_TENSOR.ATTN_V:          "blk.{bid}.attn_v",
    # 模型张量注意力输出
    MODEL_TENSOR.ATTN_OUT:        "blk.{bid}.attn_output",
    # 模型张量注意力旋转嵌入
    MODEL_TENSOR.ATTN_ROT_EMBD:   "blk.{bid}.attn_rot_embd",
    # 模型张量注意力查询规范化、键规范化
    MODEL_TENSOR.ATTN_Q_NORM:     "blk.{bid}.attn_q_norm",
    MODEL_TENSOR.ATTN_K_NORM:     "blk.{bid}.attn_k_norm",
    # 模型张量前馈网络规范化、门控、下采样、上采样、下采样转置
    MODEL_TENSOR.FFN_NORM:        "blk.{bid}.ffn_norm",
    MODEL_TENSOR.FFN_GATE:        "blk.{bid}.ffn_gate",
    MODEL_TENSOR.FFN_DOWN:        "blk.{bid}.ffn_down",
    MODEL_TENSOR.FFN_UP:          "blk.{bid}.ffn_up",
    MODEL_TENSOR.FFN_DOWN_T:      "blk.{bid}.ffn_down_t",
    # 模型张量全连接层1、2
    MODEL_TENSOR.FC_1:            "blk.{bid}.fc1",
    MODEL_TENSOR.FC_2:            "blk.{bid}.fc2",
# 定义一个字典，键为MODEL_ARCH枚举类型，值为MODEL_TENSOR枚举类型的列表
MODEL_TENSORS: dict[MODEL_ARCH, list[MODEL_TENSOR]] = {
    # 对于MODEL_ARCH枚举类型LLAMA，定义其对应的MODEL_TENSOR枚举类型列表
    MODEL_ARCH.LLAMA: [
        MODEL_TENSOR.TOKEN_EMBD,
        MODEL_TENSOR.OUTPUT_NORM,
        MODEL_TENSOR.OUTPUT,
        MODEL_TENSOR.ROPE_FREQS,
        MODEL_TENSOR.ATTN_NORM,
        MODEL_TENSOR.ATTN_Q,
        MODEL_TENSOR.ATTN_K,
        MODEL_TENSOR.ATTN_V,
        MODEL_TENSOR.ATTN_OUT,
        MODEL_TENSOR.ATTN_ROT_EMBD,
        MODEL_TENSOR.FFN_NORM,
        MODEL_TENSOR.FFN_GATE,
        MODEL_TENSOR.FFN_DOWN,
        MODEL_TENSOR.FFN_UP,
        MODEL_TENSOR.FFN_DOWN_T,
        MODEL_TENSOR.FC_1,
        MODEL_TENSOR.FC_2,
    ],
    # 对于MODEL_ARCH枚举类型GPTNEOX，定义其对应的MODEL_TENSOR枚举类型列表
    MODEL_ARCH.GPTNEOX: [
        MODEL_TENSOR.TOKEN_EMBD,
        MODEL_TENSOR.OUTPUT_NORM,
        MODEL_TENSOR.OUTPUT,
        MODEL_TENSOR.ATTN_NORM,
        MODEL_TENSOR.ATTN_QKV,
        MODEL_TENSOR.ATTN_OUT,
        MODEL_TENSOR.FFN_NORM,
        MODEL_TENSOR.FFN_DOWN,
        MODEL_TENSOR.FFN_UP,
    ],
    # 对于MODEL_ARCH枚举类型FALCON，定义其对应的MODEL_TENSOR枚举类型列表
    MODEL_ARCH.FALCON: [
        MODEL_TENSOR.TOKEN_EMBD,
        MODEL_TENSOR.OUTPUT_NORM,
        MODEL_TENSOR.OUTPUT,
        MODEL_TENSOR.ATTN_NORM,
        MODEL_TENSOR.ATTN_NORM_2,
        MODEL_TENSOR.ATTN_QKV,
        MODEL_TENSOR.ATTN_OUT,
        MODEL_TENSOR.FFN_DOWN,
        MODEL_TENSOR.FFN_UP,
    ],
    # 对于MODEL_ARCH枚举类型BAICHUAN，定义其对应的MODEL_TENSOR枚举类型列表
    MODEL_ARCH.BAICHUAN: [
        MODEL_TENSOR.TOKEN_EMBD,
        MODEL_TENSOR.OUTPUT_NORM,
        MODEL_TENSOR.OUTPUT,
        MODEL_TENSOR.ROPE_FREQS,
        MODEL_TENSOR.ATTN_NORM,
        MODEL_TENSOR.ATTN_Q,
        MODEL_TENSOR.ATTN_K,
        MODEL_TENSOR.ATTN_V,
        MODEL_TENSOR.ATTN_OUT,
        MODEL_TENSOR.ATTN_ROT_EMBD,
        MODEL_TENSOR.FFN_NORM,
        MODEL_TENSOR.FFN_GATE,
        MODEL_TENSOR.FFN_DOWN,
        MODEL_TENSOR.FFN_UP,
    ],
    MODEL_ARCH.STARCODER: [  # 定义 STARCODER 模型的层次结构
        MODEL_TENSOR.TOKEN_EMBD,  # 词嵌入层
        MODEL_TENSOR.POS_EMBD,  # 位置嵌入层
        MODEL_TENSOR.OUTPUT_NORM,  # 输出层归一化
        MODEL_TENSOR.OUTPUT,  # 输出层
        MODEL_TENSOR.ATTN_NORM,  # 注意力层归一化
        MODEL_TENSOR.ATTN_QKV,  # 注意力查询、键、值
        MODEL_TENSOR.ATTN_OUT,  # 注意力输出
        MODEL_TENSOR.FFN_NORM,  # 前馈神经网络归一化
        MODEL_TENSOR.FFN_DOWN,  # 前馈神经网络下层
        MODEL_TENSOR.FFN_UP,  # 前馈神经网络上层
    ],
    MODEL_ARCH.BERT: [  # 定义 BERT 模型的层次结构
        MODEL_TENSOR.TOKEN_EMBD,  # 词嵌入层
        MODEL_TENSOR.TOKEN_TYPES,  # 词类型嵌入层
        MODEL_TENSOR.POS_EMBD,  # 位置嵌入层
        MODEL_TENSOR.OUTPUT_NORM,  # 输出层归一化
        MODEL_TENSOR.ATTN_NORM,  # 注意力层归一化
        MODEL_TENSOR.ATTN_Q,  # 注意力查询
        MODEL_TENSOR.ATTN_K,  # 注意力键
        MODEL_TENSOR.ATTN_V,  # 注意力值
        MODEL_TENSOR.ATTN_OUT,  # 注意力输出
        MODEL_TENSOR.FFN_NORM,  # 前馈神经网络归一化
        MODEL_TENSOR.FFN_DOWN,  # 前馈神经网络下层
        MODEL_TENSOR.FFN_UP,  # 前馈神经网络上层
    ],
    MODEL_ARCH.MPT: [  # 定义 MPT 模型的层次结构
        MODEL_TENSOR.TOKEN_EMBD,  # 词嵌入层
        MODEL_TENSOR.OUTPUT_NORM,  # 输出层归一化
        MODEL_TENSOR.OUTPUT,  # 输出层
        MODEL_TENSOR.ATTN_NORM,  # 注意力层归一化
        MODEL_TENSOR.ATTN_QKV,  # 注意力查询、键、值
        MODEL_TENSOR.ATTN_OUT,  # 注意力输出
        MODEL_TENSOR.FFN_NORM,  # 前馈神经网络归一化
        MODEL_TENSOR.FFN_DOWN,  # 前馈神经网络下层
        MODEL_TENSOR.FFN_UP,  # 前馈神经网络上层
    ],
    MODEL_ARCH.GPTJ: [  # 定义 GPTJ 模型的层次结构
        MODEL_TENSOR.TOKEN_EMBD,  # 词嵌入层
        MODEL_TENSOR.OUTPUT_NORM,  # 输出层归一化
        MODEL_TENSOR.OUTPUT,  # 输出层
        MODEL_TENSOR.ATTN_NORM,  # 注意力层归一化
        MODEL_TENSOR.ATTN_Q,  # 注意力查询
        MODEL_TENSOR.ATTN_K,  # 注意力键
        MODEL_TENSOR.ATTN_V,  # 注意力值
        MODEL_TENSOR.ATTN_OUT,  # 注意力输出
        MODEL_TENSOR.FFN_DOWN,  # 前馈神经网络下层
        MODEL_TENSOR.FFN_UP,  # 前馈神经网络上层
    ],
    MODEL_ARCH.PERSIMMON: [  # 定义 PERSIMMON 模型的层次结构
        MODEL_TENSOR.TOKEN_EMBD,  # 词嵌入层
        MODEL_TENSOR.OUTPUT,  # 输出层
        MODEL_TENSOR.OUTPUT_NORM,  # 输出层归一化
        MODEL_TENSOR.ATTN_NORM,  # 注意力层归一化
        MODEL_TENSOR.ATTN_QKV,  # 注意力查询、键、值
        MODEL_TENSOR.ATTN_OUT,  # 注意力输出
        MODEL_TENSOR.FFN_NORM,  # 前馈神经网络归一化
        MODEL_TENSOR.FFN_DOWN,  # 前馈神经网络下层
        MODEL_TENSOR.FFN_UP,  # 前馈神经网络上层
        MODEL_TENSOR.ATTN_Q_NORM,  # 注意力查询归一化
        MODEL_TENSOR.ATTN_K_NORM,  # 注意力键归一化
        MODEL_TENSOR.ATTN_ROT_EMBD,  # 注意力旋转嵌入层
    ],
    # 定义了不同模型架构对应的张量流程列表
    MODEL_ARCH.REFACT: [
        MODEL_TENSOR.TOKEN_EMBD,  # Token 嵌入
        MODEL_TENSOR.OUTPUT_NORM,  # 输出规范化
        MODEL_TENSOR.OUTPUT,  # 输出
        MODEL_TENSOR.ATTN_NORM,  # 注意力规范化
        MODEL_TENSOR.ATTN_Q,  # 注意力查询
        MODEL_TENSOR.ATTN_K,  # 注意力键
        MODEL_TENSOR.ATTN_V,  # 注意力值
        MODEL_TENSOR.ATTN_OUT,  # 注意力输出
        MODEL_TENSOR.FFN_NORM,  # 前馈网络规范化
        MODEL_TENSOR.FFN_GATE,  # 前馈网络门控
        MODEL_TENSOR.FFN_DOWN,  # 前馈网络下采样
        MODEL_TENSOR.FFN_UP,  # 前馈网络上采样
    ],
    MODEL_ARCH.BLOOM: [
        MODEL_TENSOR.TOKEN_EMBD,  # Token 嵌入
        MODEL_TENSOR.TOKEN_EMBD_NORM,  # Token 嵌入规范化
        MODEL_TENSOR.OUTPUT_NORM,  # 输出规范化
        MODEL_TENSOR.OUTPUT,  # 输出
        MODEL_TENSOR.ATTN_NORM,  # 注意力规范化
        MODEL_TENSOR.ATTN_QKV,  # 注意力查询键值
        MODEL_TENSOR.ATTN_OUT,  # 注意力输出
        MODEL_TENSOR.FFN_NORM,  # 前馈网络规范化
        MODEL_TENSOR.FFN_DOWN,  # 前馈网络下采样
        MODEL_TENSOR.FFN_UP,  # 前馈网络上采样
    ],
    MODEL_ARCH.STABLELM: [
        MODEL_TENSOR.TOKEN_EMBD,  # Token 嵌入
        MODEL_TENSOR.OUTPUT_NORM,  # 输出规范化
        MODEL_TENSOR.OUTPUT,  # 输出
        MODEL_TENSOR.ROPE_FREQS,  # 绳索频率
        MODEL_TENSOR.ATTN_NORM,  # 注意力规范化
        MODEL_TENSOR.ATTN_Q,  # 注意力查询
        MODEL_TENSOR.ATTN_K,  # 注意力键
        MODEL_TENSOR.ATTN_V,  # 注意力值
        MODEL_TENSOR.ATTN_OUT,  # 注意力输出
        MODEL_TENSOR.FFN_NORM,  # 前馈网络规范化
        MODEL_TENSOR.FFN_GATE,  # 前馈网络门控
        MODEL_TENSOR.FFN_DOWN,  # 前馈网络下采样
        MODEL_TENSOR.FFN_UP,  # 前馈网络上采样
    ],
    MODEL_ARCH.GPT2: [
        # TODO
    ],
    # TODO
# tensors that will not be serialized
# 定义一个字典，用于存储不会被序列化的张量
MODEL_TENSOR_SKIP: dict[MODEL_ARCH, list[MODEL_TENSOR]] = {
    MODEL_ARCH.LLAMA: [
        MODEL_TENSOR.ROPE_FREQS,
        MODEL_TENSOR.ATTN_ROT_EMBD,
    ],
    MODEL_ARCH.BAICHUAN: [
        MODEL_TENSOR.ROPE_FREQS,
        MODEL_TENSOR.ATTN_ROT_EMBD,
    ],
    MODEL_ARCH.PERSIMMON: [
        MODEL_TENSOR.ROPE_FREQS,
    ],
}

#
# types
#

# 定义 TokenType 枚举类，包括 NORMAL、UNKNOWN、CONTROL、USER_DEFINED、UNUSED、BYTE 六种类型
class TokenType(IntEnum):
    NORMAL       = 1
    UNKNOWN      = 2
    CONTROL      = 3
    USER_DEFINED = 4
    UNUSED       = 5
    BYTE         = 6

# 定义 RopeScalingType 枚举类，包括 NONE、LINEAR、YARN 三种类型
class RopeScalingType(Enum):
    NONE   = 'none'
    LINEAR = 'linear'
    YARN   = 'yarn'

# 定义 GGMLQuantizationType 枚举类，包括多种类型，如 F32、F16、Q4_0 等
class GGMLQuantizationType(IntEnum):
    F32  = 0
    F16  = 1
    Q4_0 = 2
    Q4_1 = 3
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
    I8 = 16,
    I16 = 17
    I32 = 18,

# 定义 GGUFEndian 枚举类，包括 LITTLE、BIG 两种类型
class GGUFEndian(IntEnum):
    LITTLE = 0
    BIG = 1

# 定义 GGUFValueType 枚举类，包括多种类型，如 UINT8、INT8、UINT16 等
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

    # 定义一个静态方法，根据输入的值返回对应的 GGUFValueType 类型
    @staticmethod
    def get_type(val: Any) -> GGUFValueType:
        if isinstance(val, (str, bytes, bytearray)):
            return GGUFValueType.STRING
        elif isinstance(val, list):
            return GGUFValueType.ARRAY
        elif isinstance(val, float):
            return GGUFValueType.FLOAT32
        elif isinstance(val, bool):
            return GGUFValueType.BOOL
        elif isinstance(val, int):
            return GGUFValueType.INT32
        # TODO: need help with 64-bit types in Python
        else:
            print("Unknown type:", type(val))
            sys.exit()

# 定义一个常量 QK_K，赋值为 256
# 注意：此处的 GGML_QKK_64 是一个错误的常量名
QK_K = 256
# 定义一个字典 GGML_QUANT_SIZES，存储 GGMLQuantizationType 类型对应的 (block size, type size)
GGML_QUANT_SIZES = {
    GGMLQuantizationType.F32:  (1, 4),
    GGMLQuantizationType.F16:  (1, 2),
    # 定义 GGMLQuantizationType.Q4_0 的值为元组 (32, 2 + 16)
    GGMLQuantizationType.Q4_0: (32, 2 + 16),
    # 定义 GGMLQuantizationType.Q4_1 的值为元组 (32, 2 + 2 + 16)
    GGMLQuantizationType.Q4_1: (32, 2 + 2 + 16),
    # 定义 GGMLQuantizationType.Q5_0 的值为元组 (32, 2 + 4 + 16)
    GGMLQuantizationType.Q5_0: (32, 2 + 4 + 16),
    # 定义 GGMLQuantizationType.Q5_1 的值为元组 (32, 2 + 2 + 4 + 16)
    GGMLQuantizationType.Q5_1: (32, 2 + 2 + 4 + 16),
    # 定义 GGMLQuantizationType.Q8_0 的值为元组 (32, 2 + 32)
    GGMLQuantizationType.Q8_0: (32, 2 + 32),
    # 定义 GGMLQuantizationType.Q8_1 的值为元组 (32, 4 + 4 + 32)
    GGMLQuantizationType.Q8_1: (32, 4 + 4 + 32),
    # 定义 GGMLQuantizationType.Q2_K 的值为元组 (256, 2 + 2 + QK_K // 16 + QK_K // 4)
    GGMLQuantizationType.Q2_K: (256, 2 + 2 + QK_K // 16 + QK_K // 4),
    # 定义 GGMLQuantizationType.Q3_K 的值为元组 (256, 2 + QK_K // 4 + QK_K // 8 + 12)
    GGMLQuantizationType.Q3_K: (256, 2 + QK_K // 4 + QK_K // 8 + 12),
    # 定义 GGMLQuantizationType.Q4_K 的值为元组 (256, 2 + 2 + QK_K // 2 + 12)
    GGMLQuantizationType.Q4_K: (256, 2 + 2 + QK_K // 2 + 12),
    # 定义 GGMLQuantizationType.Q5_K 的值为元组 (256, 2 + 2 + QK_K // 2 + QK_K // 8 + 12)
    GGMLQuantizationType.Q5_K: (256, 2 + 2 + QK_K // 2 + QK_K // 8 + 12),
    # 定义 GGMLQuantizationType.Q6_K 的值为元组 (256, 2 + QK_K // 2 + QK_K // 4 + QK_K // 16)
    GGMLQuantizationType.Q6_K: (256, 2 + QK_K // 2 + QK_K // 4 + QK_K // 16),
    # 定义 GGMLQuantizationType.Q8_K 的值为元组 (256, 4 + QK_K + QK_K // 8)
    GGMLQuantizationType.Q8_K: (256, 4 + QK_K + QK_K // 8),
# 为了向后兼容而创建的别名

# 通用
KEY_GENERAL_ARCHITECTURE         = Keys.General.ARCHITECTURE  # 通用架构键
KEY_GENERAL_QUANTIZATION_VERSION = Keys.General.QUANTIZATION_VERSION  # 通用量化版本键
KEY_GENERAL_ALIGNMENT            = Keys.General.ALIGNMENT  # 通用对齐键
KEY_GENERAL_NAME                 = Keys.General.NAME  # 通用名称键
KEY_GENERAL_AUTHOR               = Keys.General.AUTHOR  # 通用作者键
KEY_GENERAL_URL                  = Keys.General.URL  # 通用 URL 键
KEY_GENERAL_DESCRIPTION          = Keys.General.DESCRIPTION  # 通用描述键
KEY_GENERAL_LICENSE              = Keys.General.LICENSE  # 通用许可证键
KEY_GENERAL_SOURCE_URL           = Keys.General.SOURCE_URL  # 通用源 URL 键
KEY_GENERAL_SOURCE_HF_REPO       = Keys.General.SOURCE_HF_REPO  # 通用源 HF 仓库键
KEY_GENERAL_FILE_TYPE            = Keys.General.FILE_TYPE  # 通用文件类型键

# LLM
KEY_CONTEXT_LENGTH        = Keys.LLM.CONTEXT_LENGTH  # LLM 上下文长度键
KEY_EMBEDDING_LENGTH      = Keys.LLM.EMBEDDING_LENGTH  # LLM 嵌入长度键
KEY_BLOCK_COUNT           = Keys.LLM.BLOCK_COUNT  # LLM 块数量键
KEY_FEED_FORWARD_LENGTH   = Keys.LLM.FEED_FORWARD_LENGTH  # LLM 前馈长度键
KEY_USE_PARALLEL_RESIDUAL = Keys.LLM.USE_PARALLEL_RESIDUAL  # LLM 使用并行残差键
KEY_TENSOR_DATA_LAYOUT    = Keys.LLM.TENSOR_DATA_LAYOUT  # LLM 张量数据布局键

# 注意力
KEY_ATTENTION_HEAD_COUNT        = Keys.Attention.HEAD_COUNT  # 注意力头数量键
KEY_ATTENTION_HEAD_COUNT_KV     = Keys.Attention.HEAD_COUNT_KV  # 注意力头数量 KV 键
KEY_ATTENTION_MAX_ALIBI_BIAS    = Keys.Attention.MAX_ALIBI_BIAS  # 注意力最大偏差键
KEY_ATTENTION_CLAMP_KQV         = Keys.Attention.CLAMP_KQV  # 注意力夹紧 KQV 键
KEY_ATTENTION_LAYERNORM_EPS     = Keys.Attention.LAYERNORM_EPS  # 注意力层归一化 EPS 键
KEY_ATTENTION_LAYERNORM_RMS_EPS = Keys.Attention.LAYERNORM_RMS_EPS  # 注意力层归一化 RMS EPS 键

# RoPE
KEY_ROPE_DIMENSION_COUNT      = Keys.Rope.DIMENSION_COUNT  # RoPE 维度数量键
KEY_ROPE_FREQ_BASE            = Keys.Rope.FREQ_BASE  # RoPE 频率基准键
KEY_ROPE_SCALING_TYPE         = Keys.Rope.SCALING_TYPE  # RoPE 缩放类型键
KEY_ROPE_SCALING_FACTOR       = Keys.Rope.SCALING_FACTOR  # RoPE 缩放因子键
KEY_ROPE_SCALING_ORIG_CTX_LEN = Keys.Rope.SCALING_ORIG_CTX_LEN  # RoPE 缩放原始上下文长度键
KEY_ROPE_SCALING_FINETUNED    = Keys.Rope.SCALING_FINETUNED  # RoPE 缩放微调键

# 分词
KEY_TOKENIZER_MODEL      = Keys.Tokenizer.MODEL  # 分词器模型键
KEY_TOKENIZER_LIST       = Keys.Tokenizer.LIST  # 分词器列表键
KEY_TOKENIZER_TOKEN_TYPE = Keys.Tokenizer.TOKEN_TYPE  # 分词器标记类型键
KEY_TOKENIZER_SCORES     = Keys.Tokenizer.SCORES  # 分词器分数键
# 定义键值常量，用于访问 Tokenizer 的 MERGES 属性
KEY_TOKENIZER_MERGES     = Keys.Tokenizer.MERGES
# 定义键值常量，用于访问 Tokenizer 的 BOS_ID 属性
KEY_TOKENIZER_BOS_ID     = Keys.Tokenizer.BOS_ID
# 定义键值常量，用于访问 Tokenizer 的 EOS_ID 属性
KEY_TOKENIZER_EOS_ID     = Keys.Tokenizer.EOS_ID
# 定义键值常量，用于访问 Tokenizer 的 UNK_ID 属性
KEY_TOKENIZER_UNK_ID     = Keys.Tokenizer.UNK_ID
# 定义键值常量，用于访问 Tokenizer 的 SEP_ID 属性
KEY_TOKENIZER_SEP_ID     = Keys.Tokenizer.SEP_ID
# 定义键值常量，用于访问 Tokenizer 的 PAD_ID 属性
KEY_TOKENIZER_PAD_ID     = Keys.Tokenizer.PAD_ID
# 定义键值常量，用于访问 Tokenizer 的 HF_JSON 属性
KEY_TOKENIZER_HF_JSON    = Keys.Tokenizer.HF_JSON
# 定义键值常量，用于访问 Tokenizer 的 RWKV 属性
KEY_TOKENIZER_RWKV       = Keys.Tokenizer.RWKV
```