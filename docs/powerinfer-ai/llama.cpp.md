# `PowerInfer\llama.cpp`

```
#define LLAMA_API_INTERNAL
#include "llama.h"

#include "unicode.h"

#include "ggml.h"

#include "ggml-alloc.h"

#ifdef GGML_USE_CUBLAS
#  include "ggml-cuda.h"
#elif defined(GGML_USE_CLBLAST)
#  include "ggml-opencl.h"
#endif

#ifdef GGML_USE_METAL
#  include "ggml-metal.h"
#endif
#ifdef GGML_USE_MPI
#  include "ggml-mpi.h"
#endif
#ifndef QK_K
#  ifdef GGML_QKK_64
#    define QK_K 64
#  else
#    define QK_K 256
#  endif
#endif

#ifdef __has_include
    #if __has_include(<unistd.h>)
        #include <unistd.h>
        #if defined(_POSIX_MAPPED_FILES)
            #include <sys/mman.h>
        #endif
        #if defined(_POSIX_MEMLOCK_RANGE)
            #include <sys/resource.h>
        #endif
    #endif
#endif

#if defined(_WIN32)
    #define WIN32_LEAN_AND_MEAN
    #ifndef NOMINMAX
        #define NOMINMAX
    #endif
    #include <windows.h>
    #include <io.h>
    #include <stdio.h> // for _fseeki64
#endif

#include <algorithm>
#include <array>
#include <cassert>
#include <cinttypes>
#include <climits>
#include <cmath>
#include <cstdarg>
#include <cstddef>
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <libgen.h>
#include <forward_list>
#include <fstream>
#include <functional>
#include <initializer_list>
#include <map>
#include <memory>
#include <mutex>
#include <numeric>
#include <queue>
#include <random>
#include <regex>
#include <set>
#include <sstream>
#include <thread>
#include <unordered_map>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

#ifdef __GNUC__
#ifdef __MINGW32__
#define LLAMA_ATTRIBUTE_FORMAT(...) __attribute__((format(gnu_printf, __VA_ARGS__)))
#else
#define LLAMA_ATTRIBUTE_FORMAT(...) __attribute__((format(printf, __VA_ARGS__)))
#endif
#else
#define LLAMA_ATTRIBUTE_FORMAT(...)
#endif

#define LLAMA_MAX_NODES 4096

// 
// global variables
// 

// sparsity threshold for sparse matrix multiplication prediction
float sparse_pred_threshold = 0.;

//
// logging
//
// 定义宏，指定函数参数的格式
LLAMA_ATTRIBUTE_FORMAT(2, 3)
// 声明一个静态函数，用于内部记录日志
static void llama_log_internal        (ggml_log_level level, const char* format, ...);
// 声明一个静态函数，用于设置默认的日志回调函数
static void llama_log_callback_default(ggml_log_level level, const char * text, void * user_data);

// 定义宏，用于记录信息级别的日志
#define LLAMA_LOG_INFO(...)  llama_log_internal(GGML_LOG_LEVEL_INFO , __VA_ARGS__)
// 定义宏，用于记录警告级别的日志
#define LLAMA_LOG_WARN(...)  llama_log_internal(GGML_LOG_LEVEL_WARN , __VA_ARGS__)
// 定义宏，用于记录错误级别的日志
#define LLAMA_LOG_ERROR(...) llama_log_internal(GGML_LOG_LEVEL_ERROR, __VA_ARGS__)

//
// helpers
//

// 定义一个静态函数，用于计算 UTF-8 编码的字符长度
static size_t utf8_len(char src) {
    // 定义一个查找表，用于存储每个字节对应的字符长度
    const size_t lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3, 4 };
    // 获取字符的高位字节
    uint8_t highbits = static_cast<uint8_t>(src) >> 4;
    // 返回字符的长度
    return lookup[highbits];
}

// 定义一个静态函数，用于替换字符串中的所有指定子串
static void replace_all(std::string & s, const std::string & search, const std::string & replace) {
    std::string result;
    // 遍历字符串，查找并替换指定子串
    for (size_t pos = 0; ; pos += search.length()) {
        auto new_pos = s.find(search, pos);
        if (new_pos == std::string::npos) {
            result += s.substr(pos, s.size() - pos);
            break;
        }
        result += s.substr(pos, new_pos - pos) + replace;
        pos = new_pos;
    }
    // 将替换后的结果赋值给原字符串
    s = std::move(result);
}

// 定义一个静态函数，用于比较两个浮点数是否接近
static bool is_float_close(float a, float b, float abs_tol) {
    // 检查绝对容差是否为非负数
    if (abs_tol < 0.0) {
        throw std::invalid_argument("Tolerance must be non-negative");
    }
    // 检查是否完全相等
    if (a == b) {
        return true;
    }
    // 检查是否为无穷大
    if (std::isinf(a) || std::isinf(b)) {
        return false;
    }
    // 使用提供的绝对容差进行常规比较
    return std::fabs(b - a) <= abs_tol;
}

// 如果定义了 GGML_USE_CPU_HBM 宏，则包含 hbwmalloc.h 头文件
#ifdef GGML_USE_CPU_HBM
#include <hbwmalloc.h>
#endif

// 定义一个静态函数，用于向文件写入指定数量的零
static void zeros(std::ofstream & file, size_t n) {
    char zero = 0;
    for (size_t i = 0; i < n; ++i) {
        file.write(&zero, 1);
    }
}

// 定义一个静态函数，用于格式化字符串
LLAMA_ATTRIBUTE_FORMAT(1, 2)
static std::string format(const char * fmt, ...) {
    va_list ap;
    va_list ap2;
    va_start(ap, fmt);
    va_copy(ap2, ap);
    // 计算格式化后的字符串长度，不包括结尾的空字符
    int size = vsnprintf(NULL, 0, fmt, ap);
    // 断言确保字符串长度非负且不超过 INT_MAX
    GGML_ASSERT(size >= 0 && size < INT_MAX); // NOLINT
    // 创建一个大小为 size+1 的字符向量
    std::vector<char> buf(size + 1);
    // 将格式化后的字符串写入 buf 中
    int size2 = vsnprintf(buf.data(), size + 1, fmt, ap2);
    // 断言确保第二次格式化的字符串长度与第一次相同
    GGML_ASSERT(size2 == size);
    // 结束第二次格式化
    va_end(ap2);
    // 结束第一次格式化
    va_end(ap);
    // 返回格式化后的字符串
    return std::string(buf.data(), size);
}

//
// gguf constants (sync with gguf.py)
//

// 定义枚举类型 llm_arch
enum llm_arch {
    LLM_ARCH_LLAMA,            // llama 架构
    LLM_ARCH_FALCON,           // falcon 架构
    LLM_ARCH_BAICHUAN,         // baichuan 架构
    LLM_ARCH_GPT2,             // gpt2 架构
    LLM_ARCH_GPTJ,             // gptj 架构
    LLM_ARCH_GPTNEOX,          // gptneox 架构
    LLM_ARCH_MPT,              // mpt 架构
    LLM_ARCH_STARCODER,        // starcoder 架构
    LLM_ARCH_PERSIMMON,        // persimmon 架构
    LLM_ARCH_REFACT,           // refact 架构
    LLM_ARCH_BLOOM,            // bloom 架构
    LLM_ARCH_STABLELM,         // stablelm 架构
    LLM_ARCH_UNKNOWN,          // 未知架构
};

// 定义静态映射表，将枚举类型 llm_arch 映射到对应的字符串
static std::map<llm_arch, std::string> LLM_ARCH_NAMES = {
    { LLM_ARCH_LLAMA,           "llama"     },
    { LLM_ARCH_FALCON,          "falcon"    },
    { LLM_ARCH_GPT2,            "gpt2"      },
    { LLM_ARCH_GPTJ,            "gptj"      },
    { LLM_ARCH_GPTNEOX,         "gptneox"   },
    { LLM_ARCH_MPT,             "mpt"       },
    { LLM_ARCH_BAICHUAN,        "baichuan"  },
    { LLM_ARCH_STARCODER,       "starcoder" },
    { LLM_ARCH_PERSIMMON,       "persimmon" },
    { LLM_ARCH_REFACT,          "refact"    },
    { LLM_ARCH_BLOOM,           "bloom"     },
    { LLM_ARCH_STABLELM,        "stablelm"  },
    { LLM_ARCH_UNKNOWN,         "unknown"   },
};

// 定义枚举类型 llm_kv
enum llm_kv {
    LLM_KV_GENERAL_ARCHITECTURE,            // 通用架构
    LLM_KV_GENERAL_QUANTIZATION_VERSION,    // 量化版本
    LLM_KV_GENERAL_ALIGNMENT,               // 对齐方式
    LLM_KV_GENERAL_NAME,                    // 名称
    LLM_KV_GENERAL_AUTHOR,                  // 作者
    LLM_KV_GENERAL_URL,                     // URL
    LLM_KV_GENERAL_DESCRIPTION,             // 描述
    LLM_KV_GENERAL_LICENSE,                 // 许可证
    LLM_KV_GENERAL_SOURCE_URL,              // 源 URL
    LLM_KV_GENERAL_SOURCE_HF_REPO,          // 源 HF 仓库

    LLM_KV_CONTEXT_LENGTH,                  // 上下文长度
    LLM_KV_EMBEDDING_LENGTH,                // 嵌入长度
    LLM_KV_BLOCK_COUNT,                     // 块数量
    LLM_KV_FEED_FORWARD_LENGTH,             // 前馈长度
    LLM_KV_USE_PARALLEL_RESIDUAL,           // 使用并行残差
    LLM_KV_TENSOR_DATA_LAYOUT,              // 张量数据布局

    LLM_KV_ATTENTION_HEAD_COUNT,            // 注意力头数量
    LLM_KV_ATTENTION_HEAD_COUNT_KV,         // 注意力头数量 KV
    LLM_KV_ATTENTION_MAX_ALIBI_BIAS,        // 注意力最大偏差
    LLM_KV_ATTENTION_CLAMP_KQV,             // 注意力夹紧 KQV
    LLM_KV_ATTENTION_LAYERNORM_EPS,         // 注意力层归一化 EPS
    LLM_KV_ATTENTION_LAYERNORM_RMS_EPS,     // 注意力层归一化 RMS EPS

    LLM_KV_ROPE_DIMENSION_COUNT,            // 绳索维度数量
    LLM_KV_ROPE_FREQ_BASE,                  // 绳索频率基数
    LLM_KV_ROPE_SCALE_LINEAR,               // 绳索线性缩放
    LLM_KV_ROPE_SCALING_TYPE,               // 绳索缩放类型
    LLM_KV_ROPE_SCALING_FACTOR,             // 绳索缩放因子
    LLM_KV_ROPE_SCALING_ORIG_CTX_LEN,       // 绳索缩放原始上下文长度
    LLM_KV_ROPE_SCALING_FINETUNED,          // 绳索缩放微调

    LLM_KV_TOKENIZER_MODEL,                 // 分词器模型
    # 定义了一系列的常量，用于标识键值对的分词器列表、分词器的类型、分词器的分数、分词器的合并、分词器的起始标识符、分词器的结束标识符、分词器的未知标识符、分词器的分隔符标识符、分词器的填充标识符、分词器的 HF_JSON、分词器的 RWKV
    LLM_KV_TOKENIZER_LIST,
    LLM_KV_TOKENIZER_TOKEN_TYPE,
    LLM_KV_TOKENIZER_SCORES,
    LLM_KV_TOKENIZER_MERGES,
    LLM_KV_TOKENIZER_BOS_ID,
    LLM_KV_TOKENIZER_EOS_ID,
    LLM_KV_TOKENIZER_UNK_ID,
    LLM_KV_TOKENIZER_SEP_ID,
    LLM_KV_TOKENIZER_PAD_ID,
    LLM_KV_TOKENIZER_HF_JSON,
    LLM_KV_TOKENIZER_RWKV,

    # 定义了稀疏阈值的常量
    LLM_KV_SPARSE_THRESHOLD,

    # 定义了分割 VRAM 容量的常量
    LLM_KV_SPLIT_VRAM_CAPACITY,
// 定义静态的键值对映射，将枚举值和字符串进行关联
static std::map<llm_kv, std::string> LLM_KV_NAMES = {
    { LLM_KV_GENERAL_ARCHITECTURE,          "general.architecture"                  },
    { LLM_KV_GENERAL_QUANTIZATION_VERSION,  "general.quantization_version"          },
    { LLM_KV_GENERAL_ALIGNMENT,             "general.alignment"                     },
    { LLM_KV_GENERAL_NAME,                  "general.name"                          },
    { LLM_KV_GENERAL_AUTHOR,                "general.author"                        },
    { LLM_KV_GENERAL_URL,                   "general.url"                           },
    { LLM_KV_GENERAL_DESCRIPTION,           "general.description"                   },
    { LLM_KV_GENERAL_LICENSE,               "general.license"                       },
    { LLM_KV_GENERAL_SOURCE_URL,            "general.source.url"                    },
    { LLM_KV_GENERAL_SOURCE_HF_REPO,        "general.source.huggingface.repository" },

    { LLM_KV_CONTEXT_LENGTH,                "%s.context_length"        },
    { LLM_KV_EMBEDDING_LENGTH,              "%s.embedding_length"      },
    { LLM_KV_BLOCK_COUNT,                   "%s.block_count"           },
    { LLM_KV_FEED_FORWARD_LENGTH,           "%s.feed_forward_length"   },
    { LLM_KV_USE_PARALLEL_RESIDUAL,         "%s.use_parallel_residual" },
    { LLM_KV_TENSOR_DATA_LAYOUT,            "%s.tensor_data_layout"    },

    { LLM_KV_ATTENTION_HEAD_COUNT,          "%s.attention.head_count"             },
    { LLM_KV_ATTENTION_HEAD_COUNT_KV,       "%s.attention.head_count_kv"          },
    { LLM_KV_ATTENTION_MAX_ALIBI_BIAS,      "%s.attention.max_alibi_bias"         },
    { LLM_KV_ATTENTION_CLAMP_KQV,           "%s.attention.clamp_kqv"              },
    { LLM_KV_ATTENTION_LAYERNORM_EPS,       "%s.attention.layer_norm_epsilon"     },
    { LLM_KV_ATTENTION_LAYERNORM_RMS_EPS,   "%s.attention.layer_norm_rms_epsilon" },

    { LLM_KV_ROPE_DIMENSION_COUNT,          "%s.rope.dimension_count"                 },
};
    { LLM_KV_ROPE_FREQ_BASE,                "%s.rope.freq_base"                       },  # 定义键值对，键为LLM_KV_ROPE_FREQ_BASE，值为"%s.rope.freq_base"
    { LLM_KV_ROPE_SCALE_LINEAR,             "%s.rope.scale_linear"                    },  # 定义键值对，键为LLM_KV_ROPE_SCALE_LINEAR，值为"%s.rope.scale_linear"
    { LLM_KV_ROPE_SCALING_TYPE,             "%s.rope.scaling.type"                    },  # 定义键值对，键为LLM_KV_ROPE_SCALING_TYPE，值为"%s.rope.scaling.type"
    { LLM_KV_ROPE_SCALING_FACTOR,           "%s.rope.scaling.factor"                  },  # 定义键值对，键为LLM_KV_ROPE_SCALING_FACTOR，值为"%s.rope.scaling.factor"
    { LLM_KV_ROPE_SCALING_ORIG_CTX_LEN,     "%s.rope.scaling.original_context_length" },  # 定义键值对，键为LLM_KV_ROPE_SCALING_ORIG_CTX_LEN，值为"%s.rope.scaling.original_context_length"
    { LLM_KV_ROPE_SCALING_FINETUNED,        "%s.rope.scaling.finetuned"               },  # 定义键值对，键为LLM_KV_ROPE_SCALING_FINETUNED，值为"%s.rope.scaling.finetuned"

    { LLM_KV_TOKENIZER_MODEL,               "tokenizer.ggml.model"              },  # 定义键值对，键为LLM_KV_TOKENIZER_MODEL，值为"tokenizer.ggml.model"
    { LLM_KV_TOKENIZER_LIST,                "tokenizer.ggml.tokens"             },  # 定义键值对，键为LLM_KV_TOKENIZER_LIST，值为"tokenizer.ggml.tokens"
    { LLM_KV_TOKENIZER_TOKEN_TYPE,          "tokenizer.ggml.token_type"         },  # 定义键值对，键为LLM_KV_TOKENIZER_TOKEN_TYPE，值为"tokenizer.ggml.token_type"
    { LLM_KV_TOKENIZER_SCORES,              "tokenizer.ggml.scores"             },  # 定义键值对，键为LLM_KV_TOKENIZER_SCORES，值为"tokenizer.ggml.scores"
    { LLM_KV_TOKENIZER_MERGES,              "tokenizer.ggml.merges"             },  # 定义键值对，键为LLM_KV_TOKENIZER_MERGES，值为"tokenizer.ggml.merges"
    { LLM_KV_TOKENIZER_BOS_ID,              "tokenizer.ggml.bos_token_id"       },  # 定义键值对，键为LLM_KV_TOKENIZER_BOS_ID，值为"tokenizer.ggml.bos_token_id"
    { LLM_KV_TOKENIZER_EOS_ID,              "tokenizer.ggml.eos_token_id"       },  # 定义键值对，键为LLM_KV_TOKENIZER_EOS_ID，值为"tokenizer.ggml.eos_token_id"
    { LLM_KV_TOKENIZER_UNK_ID,              "tokenizer.ggml.unknown_token_id"   },  # 定义键值对，键为LLM_KV_TOKENIZER_UNK_ID，值为"tokenizer.ggml.unknown_token_id"
    { LLM_KV_TOKENIZER_SEP_ID,              "tokenizer.ggml.seperator_token_id" },  # 定义键值对，键为LLM_KV_TOKENIZER_SEP_ID，值为"tokenizer.ggml.seperator_token_id"
    { LLM_KV_TOKENIZER_PAD_ID,              "tokenizer.ggml.padding_token_id"   },  # 定义键值对，键为LLM_KV_TOKENIZER_PAD_ID，值为"tokenizer.ggml.padding_token_id"
    { LLM_KV_TOKENIZER_HF_JSON,             "tokenizer.huggingface.json"        },  # 定义键值对，键为LLM_KV_TOKENIZER_HF_JSON，值为"tokenizer.huggingface.json"
    { LLM_KV_TOKENIZER_RWKV,                "tokenizer.rwkv.world"              },  # 定义键值对，键为LLM_KV_TOKENIZER_RWKV，值为"tokenizer.rwkv.world"

    { LLM_KV_SPARSE_THRESHOLD,              "powerinfer.sparse_threshold" },  # 定义键值对，键为LLM_KV_SPARSE_THRESHOLD，值为"powerinfer.sparse_threshold"

    { LLM_KV_SPLIT_VRAM_CAPACITY,           "split.vram_capacity" },  # 定义键值对，键为LLM_KV_SPLIT_VRAM_CAPACITY，值为"split.vram_capacity"
// 结构体定义，用于表示LLM_KV
struct LLM_KV {
    // 构造函数，初始化arch成员变量
    LLM_KV(llm_arch arch) : arch(arch) {}

    // 成员变量，表示llm_arch类型的arch
    llm_arch arch;

    // 重载运算符()，返回格式化后的字符串
    std::string operator()(llm_kv kv) const {
        // 调用format函数，格式化LLM_KV_NAMES[kv]和LLM_ARCH_NAMES[arch]，返回格式化后的字符串
        return ::format(LLM_KV_NAMES[kv].c_str(), LLM_ARCH_NAMES[arch].c_str());
    }
};

// 枚举类型，表示llm_tensor
enum llm_tensor {
    // 列举所有的llm_tensor
    LLM_TENSOR_TOKEN_EMBD,
    LLM_TENSOR_TOKEN_EMBD_NORM,
    // ... 其他llm_tensor
    LLM_TENSOR_FFN_DOWN_T,
};

// 静态成员变量，表示LLM_TENSOR_NAMES的映射关系
static std::map<llm_arch, std::map<llm_tensor, std::string>> LLM_TENSOR_NAMES = {
    {
        LLM_ARCH_LLAMA,  # 定义LLM_ARCH_LLAMA常量
        {  # 定义LLM_ARCH_LLAMA对应的字典
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # LLM_TENSOR_TOKEN_EMBD对应"token_embd"
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # LLM_TENSOR_OUTPUT_NORM对应"output_norm"
            { LLM_TENSOR_OUTPUT,          "output" },  # LLM_TENSOR_OUTPUT对应"output"
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  # LLM_TENSOR_ROPE_FREQS对应"rope_freqs"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # LLM_TENSOR_ATTN_NORM对应"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  # LLM_TENSOR_ATTN_Q对应"blk.%d.attn_q"
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  # LLM_TENSOR_ATTN_K对应"blk.%d.attn_k"
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  # LLM_TENSOR_ATTN_V对应"blk.%d.attn_v"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # LLM_TENSOR_ATTN_OUT对应"blk.%d.attn_output"
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd" },  # LLM_TENSOR_ATTN_ROT_EMBD对应"blk.%d.attn_rot_embd"
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # LLM_TENSOR_FFN_NORM对应"blk.%d.ffn_norm"
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  # LLM_TENSOR_FFN_GATE对应"blk.%d.ffn_gate"
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # LLM_TENSOR_FFN_DOWN对应"blk.%d.ffn_down"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # LLM_TENSOR_FFN_UP对应"blk.%d.ffn_up"
            { LLM_TENSOR_FFN_DOWN_T,      "blk.%d.ffn_down_t" },  # LLM_TENSOR_FFN_DOWN_T对应"blk.%d.ffn_down_t"
            { LLM_TENSOR_MLP_PRED_FC1,    "blk.%d.fc1" },  # LLM_TENSOR_MLP_PRED_FC1对应"blk.%d.fc1"
            { LLM_TENSOR_MLP_PRED_FC2,    "blk.%d.fc2" },  # LLM_TENSOR_MLP_PRED_FC2对应"blk.%d.fc2"
        },
    },
    {
        LLM_ARCH_BAICHUAN,  # 定义架构为白船
        {  # 定义架构对应的张量名称和对应的键值
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 定义张量名称为LLM_TENSOR_TOKEN_EMBD，对应的键值为"token_embd"
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # 定义张量名称为LLM_TENSOR_OUTPUT_NORM，对应的键值为"output_norm"
            { LLM_TENSOR_OUTPUT,          "output" },  # 定义张量名称为LLM_TENSOR_OUTPUT，对应的键值为"output"
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  # 定义张量名称为LLM_TENSOR_ROPE_FREQS，对应的键值为"rope_freqs"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # 定义张量名称为LLM_TENSOR_ATTN_NORM，对应的键值为"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  # 定义张量名称为LLM_TENSOR_ATTN_Q，对应的键值为"blk.%d.attn_q"
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  # 定义张量名称为LLM_TENSOR_ATTN_K，对应的键值为"blk.%d.attn_k"
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  # 定义张量名称为LLM_TENSOR_ATTN_V，对应的键值为"blk.%d.attn_v"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # 定义张量名称为LLM_TENSOR_ATTN_OUT，对应的键值为"blk.%d.attn_output"
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd" },  # 定义张量名称为LLM_TENSOR_ATTN_ROT_EMBD，对应的键值为"blk.%d.attn_rot_embd"
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # 定义张量名称为LLM_TENSOR_FFN_NORM，对应的键值为"blk.%d.ffn_norm"
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  # 定义张量名称为LLM_TENSOR_FFN_GATE，对应的键值为"blk.%d.ffn_gate"
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # 定义张量名称为LLM_TENSOR_FFN_DOWN，对应的键值为"blk.%d.ffn_down"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # 定义张量名称为LLM_TENSOR_FFN_UP，对应的键值为"blk.%d.ffn_up"
        },
    },
    {
        LLM_ARCH_FALCON,  # 定义架构为猎鹰
        {  # 定义架构对应的张量名称和对应的键值
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 定义张量名称为LLM_TENSOR_TOKEN_EMBD，对应的键值为"token_embd"
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # 定义张量名称为LLM_TENSOR_OUTPUT_NORM，对应的键值为"output_norm"
            { LLM_TENSOR_OUTPUT,          "output" },  # 定义张量名称为LLM_TENSOR_OUTPUT，对应的键值为"output"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # 定义张量名称为LLM_TENSOR_ATTN_NORM，对应的键值为"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_NORM_2,     "blk.%d.attn_norm_2" },  # 定义张量名称为LLM_TENSOR_ATTN_NORM_2，对应的键值为"blk.%d.attn_norm_2"
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  # 定义张量名称为LLM_TENSOR_ATTN_QKV，对应的键值为"blk.%d.attn_qkv"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # 定义张量名称为LLM_TENSOR_ATTN_OUT，对应的键值为"blk.%d.attn_output"
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # 定义张量名称为LLM_TENSOR_FFN_DOWN，对应的键值为"blk.%d.ffn_down"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # 定义张量名称为LLM_TENSOR_FFN_UP，对应的键值为"blk.%d.ffn_up"
            { LLM_TENSOR_FFN_DOWN_T,      "blk.%d.ffn_down_t" },  # 定义张量名称为LLM_TENSOR_FFN_DOWN_T，对应的键值为"blk.%d.ffn_down_t"
            { LLM_TENSOR_MLP_PRED_FC1,    "blk.%d.fc1" },  # 定义张量名称为LLM_TENSOR_MLP_PRED_FC1，对应的键值为"blk.%d.fc1"
            { LLM_TENSOR_MLP_PRED_FC2,    "blk.%d.fc2" },  # 定义张量名称为LLM_TENSOR_MLP_PRED_FC2，对应的键值为"blk.%d.fc2"
        },
    },
    {
        LLM_ARCH_GPT2,  # 定义架构为GPT2
        {  # 定义架构对应的张量名称和对应的键值
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 定义张量名称为LLM_TENSOR_TOKEN_EMBD，对应的键值为"token_embd"
        },
    },
    {
        LLM_ARCH_GPTJ,  # 定义架构为GPTJ
        {  # 定义架构对应的张量名称和对应的键值
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 定义张量名称为LLM_TENSOR_TOKEN_EMBD，对应的键值为"token_embd"
        },
    },
    {
        LLM_ARCH_GPTNEOX,  # 定义架构为 GPTNEOX
        {  # 定义架构对应的张量名称和对应的键值对
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 定义张量名称和对应的键值对
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # 定义张量名称和对应的键值对
            { LLM_TENSOR_OUTPUT,          "output" },  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # 定义张量名称和对应的键值对
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # 定义张量名称和对应的键值对
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # 定义张量名称和对应的键值对
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # 定义张量名称和对应的键值对
        },
    },
    {
        LLM_ARCH_PERSIMMON,  # 定义架构为 PERSIMMON
        {  # 定义架构对应的张量名称和对应的键值对
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_OUTPUT,          "output"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_Q_NORM,     "blk.%d.attn_q_norm"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_K_NORM,     "blk.%d.attn_k_norm"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up"},  # 定义张量名称和对应的键值对
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd"},  # 定义张量名称和对应的键值对
        },
    },
    {
        LLM_ARCH_MPT,  # 定义LLM_ARCH_MPT架构
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 定义LLM_TENSOR_TOKEN_EMBD和对应的字符串
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # 定义LLM_TENSOR_OUTPUT_NORM和对应的字符串
            { LLM_TENSOR_OUTPUT,          "output" },  # 定义LLM_TENSOR_OUTPUT和对应的字符串
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # 定义LLM_TENSOR_ATTN_NORM和对应的格式化字符串
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # 定义LLM_TENSOR_FFN_NORM和对应的格式化字符串
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  # 定义LLM_TENSOR_ATTN_QKV和对应的格式化字符串
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # 定义LLM_TENSOR_ATTN_OUT和对应的格式化字符串
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # 定义LLM_TENSOR_FFN_DOWN和对应的格式化字符串
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # 定义LLM_TENSOR_FFN_UP和对应的格式化字符串
        },
    },
    {
        LLM_ARCH_STARCODER,  # 定义LLM_ARCH_STARCODER架构
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 定义LLM_TENSOR_TOKEN_EMBD和对应的字符串
            { LLM_TENSOR_POS_EMBD,        "position_embd" },  # 定义LLM_TENSOR_POS_EMBD和对应的字符串
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # 定义LLM_TENSOR_OUTPUT_NORM和对应的字符串
            { LLM_TENSOR_OUTPUT,          "output" },  # 定义LLM_TENSOR_OUTPUT和对应的字符串
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # 定义LLM_TENSOR_ATTN_NORM和对应的格式化字符串
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  # 定义LLM_TENSOR_ATTN_QKV和对应的格式化字符串
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # 定义LLM_TENSOR_ATTN_OUT和对应的格式化字符串
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # 定义LLM_TENSOR_FFN_NORM和对应的格式化字符串
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # 定义LLM_TENSOR_FFN_UP和对应的格式化字符串
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # 定义LLM_TENSOR_FFN_DOWN和对应的格式化字符串
        },
    },
    {
        LLM_ARCH_REFACT,
        {   # 定义LLM_ARCH_REFACT架构的张量名称和对应的标签
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # token_embd张量对应的标签
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # output_norm张量对应的标签
            { LLM_TENSOR_OUTPUT,          "output" },  # output张量对应的标签
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # blk.%d.attn_norm张量对应的标签
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  # blk.%d.attn_q张量对应的标签
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  # blk.%d.attn_k张量对应的标签
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  # blk.%d.attn_v张量对应的标签
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # blk.%d.attn_output张量对应的标签
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # blk.%d.ffn_norm张量对应的标签
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  # blk.%d.ffn_gate张量对应的标签
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # blk.%d.ffn_down张量对应的标签
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # blk.%d.ffn_up张量对应的标签
        },
    },
    {
        LLM_ARCH_BLOOM,
        {   # 定义LLM_ARCH_BLOOM架构的张量名称和对应的标签
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # token_embd张量对应的标签
            { LLM_TENSOR_TOKEN_EMBD_NORM, "token_embd_norm" },  # token_embd_norm张量对应的标签
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # output_norm张量对应的标签
            { LLM_TENSOR_OUTPUT,          "output" },  # output张量对应的标签
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # blk.%d.attn_norm张量对应的标签
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  # blk.%d.attn_qkv张量对应的标签
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # blk.%d.attn_output张量对应的标签
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # blk.%d.ffn_norm张量对应的标签
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # blk.%d.ffn_up张量对应的标签
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # blk.%d.ffn_down张量对应的标签
        },
    },
    {
        LLM_ARCH_STABLELM,  # 定义稳定语言模型的架构
        {  # 架构对应的张量名称和对应的键值对
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 令牌嵌入张量
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # 输出规范化张量
            { LLM_TENSOR_OUTPUT,          "output" },  # 输出张量
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  # 绳索频率张量
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # 注意力规范化张量
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  # 注意力查询张量
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  # 注意力键张量
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  # 注意力值张量
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # 注意力输出张量
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # 前馈网络规范化张量
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  # 前馈网络门控张量
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # 前馈网络下采样张量
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # 前馈网络上采样张量
        },
    },

    {
        LLM_ARCH_UNKNOWN,  # 定义未知语言模型的架构
        {  # 架构对应的张量名称和对应的键值对
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 令牌嵌入张量
        },
    },
// 从字符串转换为枚举类型 llm_arch
static llm_arch llm_arch_from_string(const std::string & name) {
    // 遍历枚举类型和对应的字符串名称
    for (const auto & kv : LLM_ARCH_NAMES) { // NOLINT
        // 如果找到对应的枚举类型，则返回该枚举类型
        if (kv.second == name) {
            return kv.first;
        }
    }
    // 如果未找到对应的枚举类型，则返回未知类型
    return LLM_ARCH_UNKNOWN;
}

// 定义枚举类型 tensor_offloading_levels
enum tensor_offloading_levels {
    TENSOR_NO_OFFLOAD,
    TENSOR_OFFLOAD_FFN,
    TENSOR_OFFLOAD_ATTN,
    TENSOR_OFFLOAD_MLP_PRED,
    TENSOR_OFFLOAD_OUTPUT,
    TENSOR_OFFLOAD_KV_CACHE,
};

// 根据 llm_tensor 获取对应的 offloading 级别
tensor_offloading_levels get_offloading_level(llm_tensor tensor) {
    // 根据不同的 llm_tensor 类型返回对应的 offloading 级别
    switch (tensor) {
        case LLM_TENSOR_TOKEN_EMBD: case LLM_TENSOR_TOKEN_EMBD_NORM: case LLM_TENSOR_POS_EMBD: 
        case LLM_TENSOR_ROPE_FREQS:
            return TENSOR_NO_OFFLOAD;
        case LLM_TENSOR_OUTPUT: case LLM_TENSOR_OUTPUT_NORM:
            return TENSOR_OFFLOAD_OUTPUT;
        case LLM_TENSOR_ATTN_Q: case LLM_TENSOR_ATTN_K: case LLM_TENSOR_ATTN_V: 
        case LLM_TENSOR_ATTN_QKV: case LLM_TENSOR_ATTN_OUT: case LLM_TENSOR_ATTN_NORM: 
        case LLM_TENSOR_ATTN_NORM_2: case LLM_TENSOR_ATTN_ROT_EMBD:
        case LLM_TENSOR_ATTN_Q_NORM: case LLM_TENSOR_ATTN_K_NORM:
            return TENSOR_OFFLOAD_ATTN;
        case LLM_TENSOR_FFN_GATE: case LLM_TENSOR_FFN_DOWN: case LLM_TENSOR_FFN_UP:
        case LLM_TENSOR_FFN_NORM: case LLM_TENSOR_FFN_DOWN_T: 
            return TENSOR_OFFLOAD_FFN;
        case LLM_TENSOR_MLP_PRED_FC1: case LLM_TENSOR_MLP_PRED_FC2:
            return TENSOR_OFFLOAD_MLP_PRED;
        default:
            // 如果未知的 llm_tensor 类型，则抛出运行时错误
            throw std::runtime_error("unknown tensor category");
    }
}

// 定义结构体 LLM_TN，用于处理 gguf 常量
// 用法：
//   const auto tn = LLM_TN(LLM_ARCH_LLAMA);
//   std::string name = tn(LLM_TENSOR_OUTPUT);                     -> "output"
//   std::string name = tn(LLM_TENSOR_TOKEN_EMBD, "bias");         -> "token_embd.bias"
//   std::string name = tn(LLM_TENSOR_ATTN_NORM, "weight", 3);     -> "blk.3.attn_norm.weight"
struct LLM_TN {
    // 构造函数，初始化 arch 属性
    LLM_TN(llm_arch arch) : arch(arch) {}

    // 枚举类型 llm_arch
    llm_arch arch;
    # 重载运算符，返回张量名称和张量的离线级别的对
    std::pair<std::string, tensor_offloading_levels> operator()(llm_tensor tensor) const {
        # 返回张量名称和张量的离线级别的对
        return std::make_pair(LLM_TENSOR_NAMES[arch].at(tensor), get_offloading_level(tensor));
    }

    # 重载运算符，返回张量名称和张量的离线级别的对，附加后缀
    std::pair<std::string, tensor_offloading_levels> operator()(llm_tensor tensor, const std::string & suffix) const {
        # 返回张量名称和张量的离线级别的对，附加后缀
        return std::make_pair(LLM_TENSOR_NAMES[arch].at(tensor) + "." + suffix, get_offloading_level(tensor));
    }

    # 重载运算符，返回张量名称和张量的离线级别的对，附加 BID
    std::pair<std::string, tensor_offloading_levels> operator()(llm_tensor tensor, int bid) const {
        # 返回张量名称和张量的离线级别的对，格式化 BID
        return std::make_pair(::format(LLM_TENSOR_NAMES[arch].at(tensor).c_str(), bid), get_offloading_level(tensor));
    }

    # 重载运算符，返回张量名称和张量的离线级别的对，附加后缀和 BID
    std::pair<std::string, tensor_offloading_levels> operator()(llm_tensor tensor, const std::string & suffix, int bid) const {
        # 返回张量名称和张量的离线级别的对，格式化 BID，附加后缀
        return std::make_pair(::format(LLM_TENSOR_NAMES[arch].at(tensor).c_str(), bid) + "." + suffix, get_offloading_level(tensor));
    }
};

//
// gguf helpers
//

// 定义宏，用于获取指定类型的键值对，并进行类型检查和异常处理
#define GGUF_GET_KEY(ctx, dst, func, type, req, key) \
do { \
    const std::string skey(key); \  // 将键名转换为字符串
    const int kid = gguf_find_key(ctx, skey.c_str()); \  // 在上下文中查找键的ID
    if (kid >= 0) { \  // 如果找到了键
        enum gguf_type ktype = gguf_get_kv_type(ctx, kid); \  // 获取键值对的类型
        if (ktype != (type)) { \  // 如果类型不匹配
            throw std::runtime_error(format("key %s has wrong type: %s", skey.c_str(), gguf_type_name(ktype))); \  // 抛出类型错误异常
        } \
        (dst) = func(ctx, kid); \  // 获取键值对的值
    } else if (req) { \  // 如果未找到键且要求必须找到
        throw std::runtime_error(format("key not found in model: %s", skey.c_str())); \  // 抛出键未找到异常
    } \
} while (0)

// 定义静态映射，将LLAMA_ROPE_SCALING_TYPES的枚举值映射到对应的字符串
static std::map<int8_t, std::string> LLAMA_ROPE_SCALING_TYPES = {
    { LLAMA_ROPE_SCALING_NONE,   "none"   },
    { LLAMA_ROPE_SCALING_LINEAR, "linear" },
    { LLAMA_ROPE_SCALING_YARN,   "yarn"   },
};

// 根据字符串获取LLAMA_ROPE_SCALING_TYPES的枚举值
static int8_t llama_rope_scaling_type_from_string(const std::string & name) {
    for (const auto & kv : LLAMA_ROPE_SCALING_TYPES) { \  // 遍历LLAMA_ROPE_SCALING_TYPES
        if (kv.second == name) { \  // 如果找到对应的字符串
            return kv.first; \  // 返回对应的枚举值
        }
    }

    return LLAMA_ROPE_SCALING_UNSPECIFIED; \  // 如果未找到对应的字符串，返回未指定的枚举值
}

//
// ggml helpers
//

// 辅助函数，用于计算图的计划并执行计算
static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads); \  // 计划图的计算

    if (plan.work_size > 0) { \  // 如果计划的工作大小大于0
        buf.resize(plan.work_size); \  // 调整缓冲区大小
        plan.work_data = buf.data(); \  // 设置计划的工作数据
    }

    ggml_graph_compute(graph, &plan); \  // 执行图的计算
}

//
// llama helpers
//

// 分配主机内存的函数，根据不同的宏定义选择不同的内存分配方式
inline void * llama_host_malloc(size_t n) {
#ifdef GGML_USE_CUBLAS
    if (ggml_cublas_loaded()) { \  // 如果加载了CUBLAS
        return ggml_cuda_host_malloc(n); \  // 使用CUDA分配内存
    } else {
        return malloc(n); \  // 否则使用标准的malloc分配内存
    }
#elif GGML_USE_METAL
    return ggml_metal_host_malloc(n); \  // 使用Metal分配内存
#elif GGML_USE_CPU_HBM
    return hbw_malloc(n); \  // 使用HBM分配内存
#else
    return malloc(n); \  // 默认使用标准的malloc分配内存
#endif
}

// 释放主机内存的函数，根据不同的宏定义选择不同的内存释放方式
inline void llama_host_free(void * ptr) {
#ifdef GGML_USE_CUBLAS
    if (ggml_cublas_loaded()) { \  // 如果加载了CUBLAS
        return ggml_cuda_host_free(ptr); \  // 使用CUDA释放内存
    } else {
        return free(ptr); \  // 否则使用标准的free释放内存
    }
#elif GGML_USE_METAL
    return ggml_metal_host_free(ptr); \  // 使用Metal释放内存
#elif GGML_USE_CPU_HBM
    // 如果使用了 CPU HBM，则调用 hbw_free 释放内存
    return hbw_free(ptr);
#else
    // 否则调用标准的 free 函数释放内存
    return free(ptr);
#endif
}

#if defined(_WIN32)
// 在 Windows 平台下，格式化系统错误码为字符串
static std::string llama_format_win_err(DWORD err) {
    LPSTR buf;
    // 调用 FormatMessageA 函数格式化错误信息
    size_t size = FormatMessageA(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
                                 NULL, err, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPSTR)&buf, 0, NULL);
    // 如果格式化失败，则返回错误信息
    if (!size) {
        return "FormatMessageA failed";
    }
    // 将格式化后的字符串转换为 std::string 类型
    std::string ret(buf, size);
    // 释放缓冲区
    LocalFree(buf);
    return ret;
}
#endif

struct llama_buffer {
    void * data = NULL;
    size_t size = 0;

    // fallback to malloc / free
    // useful in cases where CUDA can try to allocate PINNED memory
    bool fallback = false;

    void resize(size_t n) {
        // 释放当前内存
        llama_host_free(data);
        // 分配新的内存
        data = llama_host_malloc(n);
        // 如果分配失败，则使用标准的 malloc 函数
        if (!data) {
            fallback = true;
            data = malloc(n);
        } else {
            fallback = false;
        }
        // 断言内存分配成功
        GGML_ASSERT(data);
        size = n;
    }

    ~llama_buffer() {
        if (data) {
            if (fallback) { // NOLINT
                // 如果使用了标准的 malloc 函数，则调用 free 释放内存
                free(data);
            } else {
                // 否则调用 llama_host_free 释放内存
                llama_host_free(data);
            }
        }
        // 将指针置为空
        data = NULL;
    }
};

struct llama_file {
    // use FILE * so we don't have to re-open the file to mmap
    FILE * fp;
    std::string fname;
    size_t size;

    llama_file(const char * fname, const char * mode): fname(fname) {
        // 打开文件
        fp = std::fopen(fname, mode);
        // 如果打开失败，则抛出运行时错误
        if (fp == NULL) {
            throw std::runtime_error(format("failed to open %s: %s", fname, strerror(errno)));
        }
        // 定位到文件末尾，获取文件大小
        seek(0, SEEK_END);
        size = tell();
        // 定位到文件开头
        seek(0, SEEK_SET);
    }

    size_t tell() const {
#ifdef _WIN32
        __int64 ret = _ftelli64(fp);
#else
        long ret = std::ftell(fp);
#endif
        // 断言获取文件位置成功
        GGML_ASSERT(ret != -1); // this really shouldn't fail
        return (size_t) ret;
    }

    void seek(size_t offset, int whence) const {
#ifdef _WIN32
        // 如果是 Windows 系统，使用 _fseeki64 函数进行文件定位
        int ret = _fseeki64(fp, (__int64) offset, whence);
#else
        // 如果不是 Windows 系统，使用 std::fseek 函数进行文件定位
        int ret = std::fseek(fp, (long) offset, whence);
#endif
        // 断言文件定位操作是否成功
        GGML_ASSERT(ret == 0); // same
    }

    // 从文件中读取原始数据
    void read_raw(void * ptr, size_t len) const {
        // 如果长度为 0，则直接返回
        if (len == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 从文件中读取数据到指定的内存地址
        std::size_t ret = std::fread(ptr, len, 1, fp);
        // 如果发生读取错误，抛出运行时异常
        if (ferror(fp)) {
            throw std::runtime_error(format("read error: %s", strerror(errno)));
        }
        // 如果读取的数据长度不符合预期，抛出运行时异常
        if (ret != 1) {
            throw std::runtime_error(std::string("unexpectedly reached end of file"));
        }
    }

    // 从文件中读取一个 32 位无符号整数
    uint32_t read_u32() const {
        uint32_t ret;
        read_raw(&ret, sizeof(ret));
        return ret;
    }

    // 向文件中写入原始数据
    void write_raw(const void * ptr, size_t len) const {
        // 如果长度为 0，则直接返回
        if (len == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 向文件中写入数据
        size_t ret = std::fwrite(ptr, len, 1, fp);
        // 如果写入的数据长度不符合预期，抛出运行时异常
        if (ret != 1) {
            throw std::runtime_error(format("write error: %s", strerror(errno)));
        }
    }

    // 向文件中写入一个 32 位无符号整数
    void write_u32(std::uint32_t val) const {
        write_raw(&val, sizeof(val));
    }

    // 析构函数，关闭文件
    ~llama_file() {
        if (fp) {
            std::fclose(fp);
        }
    }
};

// 表示内存映射的结构体
struct llama_mmap {
    void * addr; // 内存地址
    size_t size; // 内存大小

    // 禁用拷贝构造函数
    llama_mmap(const llama_mmap &) = delete;

#ifdef _POSIX_MAPPED_FILES
    // 是否支持内存映射
    static constexpr bool SUPPORTED = true;

    // 构造函数，用于创建内存映射
    llama_mmap(struct llama_file * file, size_t prefetch = (size_t) -1 /* -1 = max value */, bool numa = false) {
        size = file->size; // 获取文件大小
        int fd = fileno(file->fp); // 获取文件描述符
        int flags = MAP_SHARED; // 内存映射标志
        // 如果在 NUMA 系统上，禁用预取/预读
        if (numa) { prefetch = 0; }
#ifdef __linux__
        // 如果需要预取，设置内存映射标志
        if (prefetch) { flags |= MAP_POPULATE; }
#endif
        // 使用 mmap 函数将文件映射到内存中
        addr = mmap(NULL, file->size, PROT_READ, flags, fd, 0);
        // 如果映射失败，抛出运行时错误
        if (addr == MAP_FAILED) {
            throw std::runtime_error(format("mmap failed: %s", strerror(errno)));
        }

        // 如果设置了预取值，通知内核预取映射的内存
        if (prefetch > 0) {
            if (posix_madvise(addr, std::min(file->size, prefetch), POSIX_MADV_WILLNEED)) {
                fprintf(stderr, "warning: posix_madvise(.., POSIX_MADV_WILLNEED) failed: %s\n",
                        strerror(errno));
            }
        }
        // 如果设置了 NUMA，通知内核不要使用预读
        if (numa) {
            if (posix_madvise(addr, file->size, POSIX_MADV_RANDOM)) {
                fprintf(stderr, "warning: posix_madvise(.., POSIX_MADV_RANDOM) failed: %s\n",
                        strerror(errno));
            }
        }
    }

    // 析构函数，释放内存映射
    ~llama_mmap() {
        munmap(addr, size);
    }
#elif defined(_WIN32)
    static constexpr bool SUPPORTED = true;
    # 创建一个名为 llama_mmap 的函数，用于映射文件到内存
    llama_mmap(struct llama_file * file, bool prefetch = true, bool numa = false) {
        # 忽略 numa 参数

        # 获取文件大小
        size = file->size;

        # 获取文件句柄
        HANDLE hFile = (HANDLE) _get_osfhandle(_fileno(file->fp));

        # 创建文件映射
        HANDLE hMapping = CreateFileMappingA(hFile, NULL, PAGE_READONLY, 0, 0, NULL);
        # 获取错误信息
        DWORD error = GetLastError();

        # 如果创建文件映射失败，则抛出运行时错误
        if (hMapping == NULL) {
            throw std::runtime_error(format("CreateFileMappingA failed: %s", llama_format_win_err(error).c_str()));
        }

        # 将文件映射到内存
        addr = MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);
        # 获取错误信息
        error = GetLastError();
        # 关闭文件映射
        CloseHandle(hMapping);

        # 如果映射失败，则抛出运行时错误
        if (addr == NULL) {
            throw std::runtime_error(format("MapViewOfFile failed: %s", llama_format_win_err(error).c_str()));
        }

        # 如果需要预取数据
        if (prefetch) {
            # 动态加载 PrefetchVirtualMemory 函数，该函数仅在 Windows 8 及以上版本可用
            BOOL (WINAPI *pPrefetchVirtualMemory) (HANDLE, ULONG_PTR, PWIN32_MEMORY_RANGE_ENTRY, ULONG);
            HMODULE hKernel32 = GetModuleHandleW(L"kernel32.dll");

            # 获取 PrefetchVirtualMemory 函数的地址
            pPrefetchVirtualMemory = reinterpret_cast<decltype(pPrefetchVirtualMemory)> (GetProcAddress(hKernel32, "PrefetchVirtualMemory"));

            # 如果成功获取到函数地址
            if (pPrefetchVirtualMemory) {
                # 预取映射的内存数据
                WIN32_MEMORY_RANGE_ENTRY range;
                range.VirtualAddress = addr;
                range.NumberOfBytes = (SIZE_T)size;
                if (!pPrefetchVirtualMemory(GetCurrentProcess(), 1, &range, 0)) {
                    # 如果预取失败，则输出警告信息
                    fprintf(stderr, "warning: PrefetchVirtualMemory failed: %s\n",
                            llama_format_win_err(GetLastError()).c_str());
                }
            }
        }
    }
    # 定义名为 llama_mmap 的函数，用于释放内存映射
    ~llama_mmap() {
        # 检查是否成功释放内存映射，如果失败则输出警告信息
        if (!UnmapViewOfFile(addr)) {
            fprintf(stderr, "warning: UnmapViewOfFile failed: %s\n",
                    llama_format_win_err(GetLastError()).c_str());
        }
    }
// 如果不支持 mmap，则设置 SUPPORTED 为 false
#else
    static constexpr bool SUPPORTED = false;

    // 构造函数 llama_mmap，接受一个 llama_file 结构体指针和两个布尔值参数
    llama_mmap(struct llama_file * file, bool prefetch = true, bool numa = false) {
        (void) file;
        (void) prefetch;
        (void) numa;

        // 抛出运行时异常，表示不支持 mmap
        throw std::runtime_error(std::string("mmap not supported"));
    }
#endif
};

// 表示使用 mlock 或 VirtualLock 锁定的内存区域；在销毁时会自动解锁
struct llama_mlock {
    void * addr = NULL; // 地址初始化为 NULL
    size_t size = 0; // 大小初始化为 0

    bool failed_already = false; // 失败标志初始化为 false

    llama_mlock() {} // 默认构造函数
    llama_mlock(const llama_mlock &) = delete; // 禁用拷贝构造函数

    // 析构函数，在销毁时自动解锁内存
    ~llama_mlock() {
        if (size) {
            raw_unlock(addr, size); // 调用 raw_unlock 函数解锁内存
        }
    }

    // 初始化函数，接受一个指针参数，用于初始化地址
    void init(void * ptr) {
        GGML_ASSERT(addr == NULL && size == 0); // NOLINT
        addr = ptr; // 初始化地址
    }

    // 扩展内存大小到目标大小
    void grow_to(size_t target_size) {
        GGML_ASSERT(addr); // 断言地址不为空
        if (failed_already) { // 如果之前失败过，则直接返回
            return;
        }
        size_t granularity = lock_granularity(); // 获取内存锁定的粒度
        target_size = (target_size + granularity - 1) & ~(granularity - 1); // 调整目标大小到粒度的整数倍
        if (target_size > size) { // 如果目标大小大于当前大小
            if (raw_lock((uint8_t *) addr + size, target_size - size)) { // 调用 raw_lock 函数锁定内存
                size = target_size; // 更新内存大小
            } else {
                failed_already = true; // 标记失败
            }
        }
    }

    // 如果支持 POSIX_MEMLOCK_RANGE，则设置 SUPPORTED 为 true
#ifdef _POSIX_MEMLOCK_RANGE
    static constexpr bool SUPPORTED = true;

    // 获取内存锁定的粒度
    static size_t lock_granularity() {
        return (size_t) sysconf(_SC_PAGESIZE);
    }

    // 根据操作系统不同，设置不同的内存锁定建议
    #ifdef __APPLE__
        #define MLOCK_SUGGESTION \
            "Try increasing the sysctl values 'vm.user_wire_limit' and 'vm.global_user_wire_limit' and/or " \
            "decreasing 'vm.global_no_user_wire_amount'.  Also try increasing RLIMIT_MLOCK (ulimit -l).\n"
    #else
        #define MLOCK_SUGGESTION \
            "Try increasing RLIMIT_MLOCK ('ulimit -l' as root).\n"
    #endif
    // 使用mlock函数锁定指定内存区域，防止被交换出去
    bool raw_lock(const void * addr, size_t size) const {
        // 如果mlock函数执行成功，则返回true
        if (!mlock(addr, size)) {
            return true;
        }

        // 获取错误信息
        char* errmsg = std::strerror(errno);
        // 检查是否建议使用其他方法
        bool suggest = (errno == ENOMEM);

        // 检查资源限制是否正常
        struct rlimit lock_limit;
        // 如果建议使用其他方法并且获取内存锁定限制成功
        if (suggest && getrlimit(RLIMIT_MEMLOCK, &lock_limit)) {
            suggest = false;
        }
        // 如果建议使用其他方法并且内存锁定限制大于当前已锁定内存大小加上要锁定的内存大小
        if (suggest && (lock_limit.rlim_max > lock_limit.rlim_cur + size)) {
            suggest = false;
        }

        // 输出警告信息
        fprintf(stderr, "warning: failed to mlock %zu-byte buffer (after previously locking %zu bytes): %s\n%s",
                size, this->size, errmsg, suggest ? MLOCK_SUGGESTION : "");
        // 返回false
        return false;
    }

    // 解锁指定内存区域
    static void raw_unlock(void * addr, size_t size) {
        // 如果munlock函数执行失败
        if (munlock(addr, size)) {
            // 输出警告信息
            fprintf(stderr, "warning: failed to munlock buffer: %s\n", std::strerror(errno));
        }
    }

    // 取消MLOCK_SUGGESTION宏定义
    #undef MLOCK_SUGGESTION
#elif defined(_WIN32)
    // 定义静态常量SUPPORTED为true
    static constexpr bool SUPPORTED = true;

    // 返回系统锁定粒度的大小
    static size_t lock_granularity() {
        // 获取系统信息
        SYSTEM_INFO si;
        GetSystemInfo(&si);
        return (size_t) si.dwPageSize;
    }

    // 在Windows平台上锁定内存区域
    bool raw_lock(void * ptr, size_t len) const {
        // 尝试次数
        for (int tries = 1; ; tries++) {
            // 尝试使用VirtualLock锁定内存区域
            if (VirtualLock(ptr, len)) {
                return true;
            }
            // 如果尝试次数为2，则输出警告信息并返回false
            if (tries == 2) {
                fprintf(stderr, "warning: failed to VirtualLock %zu-byte buffer (after previously locking %zu bytes): %s\n",
                    len, size, llama_format_win_err(GetLastError()).c_str());
                return false;
            }

            // 锁定失败，增加工作集大小并重试
            SIZE_T min_ws_size, max_ws_size;
            if (!GetProcessWorkingSetSize(GetCurrentProcess(), &min_ws_size, &max_ws_size)) {
                fprintf(stderr, "warning: GetProcessWorkingSetSize failed: %s\n",
                        llama_format_win_err(GetLastError()).c_str());
                return false;
            }
            // 根据MSDN文档，进程可以锁定的最大页面数等于其最小工作集中的页面数减去一个小的开销
            // 希望1MB的开销足够大
            size_t increment = len + 1048576;
            // 最小值必须小于等于最大值，因此需要增加两者
            min_ws_size += increment;
            max_ws_size += increment;
            if (!SetProcessWorkingSetSize(GetCurrentProcess(), min_ws_size, max_ws_size)) {
                fprintf(stderr, "warning: SetProcessWorkingSetSize failed: %s\n",
                        llama_format_win_err(GetLastError()).c_str());
                return false;
            }
        }
    }
    // 定义一个静态函数，用于释放指定内存区域
    static void raw_unlock(void * ptr, size_t len) {
        // 使用 VirtualUnlock 函数释放指定内存区域
        if (!VirtualUnlock(ptr, len)) {
            // 如果释放失败，输出警告信息
            fprintf(stderr, "warning: failed to VirtualUnlock buffer: %s\n",
                    llama_format_win_err(GetLastError()).c_str());
        }
    }
#else
    // 如果不支持，则设置为 false
    static constexpr bool SUPPORTED = false;

    // 返回锁定粒度
    static size_t lock_granularity() {
        return (size_t) 65536;
    }

    // 锁定内存区域
    bool raw_lock(const void * addr, size_t len) const {
        // 输出警告信息
        fprintf(stderr, "warning: mlock not supported on this system\n");
        return false;
    }

    // 解锁内存区域
    static void raw_unlock(const void * addr, size_t len) {}
#endif
};

// 定义一个函数指针类型
typedef void (*offload_func_t)(struct ggml_tensor * tensor);

// 空函数，不执行任何操作
static void ggml_offload_nop(struct ggml_tensor * tensor) {
    (void) tensor;
}

// 将 llama_token 转换为字符串
static std::string llama_token_to_piece(const struct llama_context * ctx, llama_token token) {
    // 创建一个长度为 8 的字符向量
    std::vector<char> result(8, 0);
    // 调用 llama_token_to_piece 函数，将结果存储在 result 中
    const int n_tokens = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
    // 如果返回值小于 0，则重新调整 result 的大小
    if (n_tokens < 0) {
        result.resize(-n_tokens);
        int check = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
        GGML_ASSERT(check == -n_tokens);
    }
    // 否则，调整 result 的大小为返回值
    else {
        result.resize(n_tokens);
    }

    return std::string(result.data(), result.size());
}

//
// 全局变量
//

// llama_state 结构体
struct llama_state {
    // 全局保存日志回调函数
    ggml_log_callback log_callback = llama_log_callback_default;
    void * log_callback_user_data = nullptr;
};

// 全局 llama_state 对象
static llama_state g_state;

// 可用的 llama 模型
enum e_model {
    MODEL_UNKNOWN,
    MODEL_1B,
    MODEL_3B,
    MODEL_7B,
    MODEL_8B,
    MODEL_13B,
    MODEL_15B,
    MODEL_30B,
    MODEL_34B,
    MODEL_40B,
    MODEL_65B,
    MODEL_70B,
};

// 定义常量
static const size_t kB = 1024;
static const size_t MB = 1024*kB;
static const size_t GB = 1024*MB;

// llama_hparams 结构体
struct llama_hparams {
    bool     vocab_only;
    uint32_t n_vocab;
    uint32_t n_ctx_train; // 模型训练时的上下文大小
    uint32_t n_embd;
    uint32_t n_head;
    uint32_t n_head_kv;
    uint32_t n_layer;
    uint32_t n_rot;
    uint32_t n_ff;

    float f_norm_eps;
    float f_norm_rms_eps;

    float    rope_freq_base_train;
    float    rope_freq_scale_train;
    // 定义一个无符号32位整数变量n_yarn_orig_ctx
    uint32_t n_yarn_orig_ctx;
    // 定义一个8位整数变量rope_scaling_type_train，并且只占用3位作为存储空间
    int8_t rope_scaling_type_train : 3;
    // 定义一个布尔类型变量rope_finetuned，只占用1位作为存储空间
    bool rope_finetuned : 1;

    // 定义两个单精度浮点数变量f_clamp_kqv和f_max_alibi_bias
    float f_clamp_kqv;
    float f_max_alibi_bias;
    
    // 如果启用了稀疏推断，则设置稀疏预测阈值为环境变量LLAMA_SPARSE_PRED_THRESHOLD的值，如果未设置则默认为"0.0"
    float sparse_pred_threshold = atof(getenv("LLAMA_SPARSE_PRED_THRESHOLD") ?: "0.0");

    // 定义一个不等于操作符的重载函数，用于比较两个llama_hparams对象是否不相等
    bool operator!=(const llama_hparams & other) const {
        // 逐个比较各个成员变量，如果有不相等的则返回true
        if (this->vocab_only  != other.vocab_only)  return true;
        if (this->n_vocab     != other.n_vocab)     return true;
        // ... 省略其他成员变量的比较
        if (!is_float_close(this->rope_freq_scale_train, other.rope_freq_scale_train, EPSILON)) return true;

        // 如果所有成员变量都相等，则返回false
        return false;
    }

    // 返回n_head/n_head_kv的结果
    uint32_t n_gqa() const {
        return n_head/n_head_kv;
    }

    // 返回n_embd/n_head的结果
    uint32_t n_embd_head() const {
        return n_embd/n_head;
    }

    // 返回n_embd/n_gqa()的结果
    uint32_t n_embd_gqa() const {
        return n_embd/n_gqa();
    }
// 结构体定义，用于存储 LLAMA 模型的参数
};

struct llama_cparams {
    uint32_t n_ctx;       // 推断期间使用的上下文大小
    uint32_t n_batch;     // 批处理大小
    uint32_t n_threads;       // 用于生成的线程数
    uint32_t n_threads_batch; // 用于批处理的线程数

    float    rope_freq_base;  // 绳索频率基础值
    float    rope_freq_scale; // 绳索频率缩放值

    uint32_t n_yarn_orig_ctx; // 原始上下文大小
    // 这些超参数在 GGUF 中未公开，因为所有现有的 YaRN 模型对它们使用相同的值。
    float yarn_ext_factor;    // YaRN 扩展因子
    float yarn_attn_factor;   // YaRN 注意力因子
    float yarn_beta_fast;     // YaRN 快速 beta
    float yarn_beta_slow;     // YaRN 慢速 beta

    bool mul_mat_q;           // 是否使用矩阵乘法
};

struct llama_layer {
    // 归一化
    struct ggml_tensor * attn_norm;
    struct ggml_tensor * attn_norm_b;
    struct ggml_tensor * attn_norm_2;
    struct ggml_tensor * attn_norm_2_b;
    struct ggml_tensor * attn_q_norm;
    struct ggml_tensor * attn_q_norm_b;
    struct ggml_tensor * attn_k_norm;
    struct ggml_tensor * attn_k_norm_b;

    // 注意力
    struct ggml_tensor * wq;
    struct ggml_tensor * wk;
    struct ggml_tensor * wv;
    struct ggml_tensor * wo;
    struct ggml_tensor * wqkv;

    // 注意力偏置
    struct ggml_tensor * bo;
    struct ggml_tensor * bqkv;

    // 归一化
    struct ggml_tensor * ffn_norm;
    struct ggml_tensor * ffn_norm_b;

    // 前馈
    struct ggml_tensor * ffn_gate; // w1
    struct ggml_tensor * ffn_down; // w2
    struct ggml_tensor * ffn_up;   // w3
    struct ggml_tensor * ffn_down_t;
    
    // 在 GPU 上切片的前馈
    struct ggml_tensor * ffn_gate_gpu;
    struct ggml_tensor * ffn_down_gpu;
    struct ggml_tensor * ffn_up_gpu;

    // 前馈偏置
    struct ggml_tensor * ffn_down_b; // b2
    struct ggml_tensor * ffn_up_b;   // b3

    // MLP 预测器权重
    struct ggml_tensor * mlp_pre_w1;
    struct ggml_tensor * mlp_pre_w2;

    // GPU 双索引
    // TODO: 需要为所有层填写此内容
    struct ggml_tensor * gpu_idx;
    struct ggml_tensor * gpu_bucket;
};

struct llama_kv_cell {
    # 定义变量pos，用于存储llama_pos类型的值，初始化为-1
    llama_pos pos   = -1;
    # 定义变量delta，用于存储llama_pos类型的值，初始化为0
    llama_pos delta = 0;

    # 创建一个空的集合，用于存储llama_seq_id类型的值
    std::set<llama_seq_id> seq_id;

    # 定义一个函数，用于检查集合中是否存在指定的llama_seq_id值，返回布尔类型
    bool has_seq_id(const llama_seq_id & id) const {
        # 使用find方法在集合中查找指定的llama_seq_id值，如果找到则返回true，否则返回false
        return seq_id.find(id) != seq_id.end();
    }
// 缓存的键值对数据的环形缓冲区
struct llama_kv_cache {
    bool has_shift = false; // 标记是否发生了移位操作

    // 注意：head 的值不仅用于优化搜索空闲的 KV 槽位，llama_decode_internal 也在使用它，
    // 所以在分配了槽位后不能随意更改它
    uint32_t head = 0; // 头部索引，用于优化搜索空闲的 KV 槽位
    uint32_t size = 0; // 大小

    // 在每次构建图之前计算
    uint32_t n = 0; // n 的值在每次构建图之前计算

    std::vector<llama_kv_cell> cells; // 存储键值对数据的单元格

    struct ggml_tensor * k = NULL; // 键的张量
    struct ggml_tensor * v = NULL; // 值的张量

    struct ggml_context * ctx = NULL; // 上下文

    llama_buffer buf; // 缓冲区

    ~llama_kv_cache() { // 析构函数
        if (ctx) { // 如果上下文存在
            ggml_free(ctx); // 释放上下文
        }

#ifdef GGML_USE_CUBLAS
        if (ggml_cublas_loaded()) { // 如果已加载 ggml_cublas
            ggml_cuda_free_data(k); // 释放键的 CUDA 数据
            ggml_cuda_free_data(v); // 释放值的 CUDA 数据
        }
#endif
    }
};

// 词汇表
struct llama_vocab {
    using id    = int32_t; // id 类型为 int32_t
    using token = std::string; // token 类型为字符串
    using ttype = llama_token_type; // ttype 类型为 llama_token_type

    struct token_data { // token_data 结构
        token text; // 文本
        float score; // 分数
        ttype type; // 类型
    };

    enum llama_vocab_type type = LLAMA_VOCAB_TYPE_SPM; // 词汇表类型为 LLAMA_VOCAB_TYPE_SPM

    std::unordered_map<token, id> token_to_id; // token 到 id 的映射
    std::vector<token_data>       id_to_token; // id 到 token_data 的映射

    std::unordered_map<token, id> special_tokens_cache; // 特殊 token 的缓存

    std::map<std::pair<std::string, std::string>, int> bpe_ranks; // BPE 排名

    // 默认的 LLaMA 特殊 token
    id special_bos_id = 1; // 开始 token 的 id
    id special_eos_id = 2; // 结束 token 的 id
    id special_unk_id = 0; // 未知 token 的 id
    id special_sep_id = -1; // 分隔 token 的 id
    id special_pad_id = -1; // 填充 token 的 id

    id linefeed_id       = 13; // 换行符的 id
    id special_prefix_id = 32007; // 特殊前缀的 id
    id special_middle_id = 32009; // 特殊中间的 id
    id special_suffix_id = 32008; // 特殊后缀的 id
    id special_eot_id    = 32010; // 特殊结束的 id
    // 查找给定 token_left 和 token_right 的 BPE 等级
    int find_bpe_rank(std::string token_left, std::string token_right) const {
        // 断言 token_left 中不包含空格和换行符
        GGML_ASSERT(token_left.find(" ") == std::string::npos);
        GGML_ASSERT(token_left.find("\n") == std::string::npos);
        // 断言 token_right 中不包含空格和换行符
        GGML_ASSERT(token_right.find(" ") == std::string::npos);
        GGML_ASSERT(token_right.find("\n") == std::string::npos);

        // 在 bpe_ranks 中查找给定 token_left 和 token_right 的组合
        auto it = bpe_ranks.find(std::make_pair(token_left, token_right));
        // 如果找不到，则返回 -1
        if (it == bpe_ranks.end()) {
            return -1;
        }

        // 返回找到的 BPE 等级
        return it->second;
    }
// 定义了一个名为 llama_gpu_split_loader 的结构体
struct llama_gpu_split_loader;

// 定义了一个名为 llama_augmentation_model_loader 的结构体
struct llama_augmentation_model_loader;

// 定义了一个名为 llama_model 的结构体
struct llama_model {
    // 模型类型，默认为 MODEL_UNKNOWN
    e_model     type  = MODEL_UNKNOWN;
    // 模型架构，默认为 LLM_ARCH_UNKNOWN
    llm_arch    arch  = LLM_ARCH_UNKNOWN;
    // 文件类型，默认为 LLAMA_FTYPE_ALL_F32
    llama_ftype ftype = LLAMA_FTYPE_ALL_F32;

    // 模型名称，默认为 "n/a"
    std::string name = "n/a";

    // 稀疏导数
    ggml_sparse_deriv sparse_deriv;

    // 超参数
    llama_hparams hparams = {};
    // 词汇表
    llama_vocab   vocab;

    // 词嵌入
    struct ggml_tensor * tok_embd;
    // 位置嵌入
    struct ggml_tensor * pos_embd;
    // 词嵌入归一化
    struct ggml_tensor * tok_norm;
    // 词嵌入归一化偏置
    struct ggml_tensor * tok_norm_b;

    // 输出归一化
    struct ggml_tensor * output_norm;
    // 输出归一化偏置
    struct ggml_tensor * output_norm_b;
    // 输出
    struct ggml_tensor * output;

    // 层列表
    std::vector<llama_layer> layers;

    // GPU 层数量
    int n_gpu_layers;

    // 上下文
    struct ggml_context * ctx = NULL;

    // 模型内存缓冲区
    llama_buffer buf;

    // 模型内存映射文件
    std::unique_ptr<llama_mmap> mapping;

    // 用于动态加载/转换模型权重的辅助模型加载器
    std::unique_ptr<struct llama_gpu_split_loader> mlp_model_loader;
    std::unique_ptr<struct llama_augmentation_model_loader> aug_model_loader;

    // 表示可能被锁定在内存中的数据的对象
    llama_mlock mlock_buf;
    llama_mlock mlock_mmap;

    // 仅用于量化统计
    std::vector<std::pair<std::string, struct ggml_tensor *>> tensors_by_name;

    // 加载时间（微秒）
    int64_t t_load_us = 0;
    // 开始时间（微秒）
    int64_t t_start_us = 0;

    // 析构函数
    ~llama_model() {
        // 如果上下文存在，则释放上下文
        if (ctx) {
            ggml_free(ctx);
        }

        // 如果使用了 CUBLAS，则释放相关资源
#ifdef GGML_USE_CUBLAS
        if (ggml_cublas_loaded()) {
            for (size_t i = 0; i < tensors_by_name.size(); ++i) {
                ggml_cuda_free_data(tensors_by_name[i].second);
            }
            ggml_cuda_free_scratch();
        }
#endif

        // 如果使用了 CLBLAST，则释放相关资源
#if defined(GGML_USE_CLBLAST)
        for (size_t i = 0; i < tensors_by_name.size(); ++i) {
            ggml_cl_free_data(tensors_by_name[i].second);
        }
#endif
    }
};

// 定义了一个名为 llama_context 的结构体
struct llama_context {
    # 定义 llma_context 类的构造函数，初始化 model、t_start_us 和 t_load_us
    llama_context(const llama_model & model) : model(model), t_start_us(model.t_start_us), t_load_us(model.t_load_us) {}
    # 定义 llma_context 类的析构函数
    ~llama_context() {
#ifdef GGML_USE_METAL
        // 如果使用 Metal，释放 Metal 上下文
        if (ctx_metal) {
            ggml_metal_free(ctx_metal);
        }
#endif
        // 释放分配器
        if (alloc) {
            ggml_allocr_free(alloc);
        }
    }

    llama_cparams cparams; // 定义 cparams 结构体

    const llama_model & model; // 定义 model 常量引用

    // 自注意力的键值缓存
    struct llama_kv_cache kv_self;

    std::mt19937 rng; // 随机数生成器

    bool has_evaluated_once = false; // 是否已经评估过一次

    int64_t t_start_us; // 开始时间
    int64_t t_load_us; // 加载时间
    int64_t t_sample_us = 0; // 采样时间，默认为0
    int64_t t_p_eval_us = 0; // prompt 评估时间，默认为0
    int64_t t_eval_us   = 0; // 评估时间，默认为0

    int32_t n_sample = 0; // 采样的标记数
    int32_t n_p_eval = 0; // prompt 评估调用的标记数（批量大小 > 1）
    int32_t n_eval   = 0; // 评估调用的标记数

    // 解码输出（二维数组：[n_tokens][n_vocab]）
    std::vector<float> logits;
    bool logits_all = false; // 是否包含所有 logits

    // 输入嵌入（一维数组：[n_embd]）
    std::vector<float> embedding;

    // 用于 `struct ggml_graph_plan.work_data` 的可重用缓冲区
    std::vector<uint8_t> work_buffer;

    // 用于评估模型的内存缓冲区
    llama_buffer buf_compute;

    llama_buffer buf_alloc;
    ggml_allocr * alloc = NULL; // 分配器初始化为空

#ifdef GGML_USE_METAL
    ggml_metal_context * ctx_metal = NULL; // Metal 上下文初始化为空
#endif

#ifdef GGML_USE_MPI
    ggml_mpi_context * ctx_mpi = NULL; // MPI 上下文初始化为空
#endif
};

//
// kv cache helpers
//

static bool llama_kv_cache_init(
        const struct llama_hparams & hparams,
             struct llama_kv_cache & cache,
                         ggml_type   wtype,
                          uint32_t   n_ctx,
                               int   n_gpu_layers) {
    const uint32_t n_embd  = hparams.n_embd_gqa(); // 获取嵌入维度
    const uint32_t n_layer = hparams.n_layer; // 获取层数

    const int64_t n_mem      = n_layer*n_ctx; // 计算内存大小
    const int64_t n_elements = n_embd*n_mem; // 计算元素个数

    cache.has_shift = false; // 初始化缓存的位移标志为 false

    cache.head = 0; // 初始化缓存头部位置为 0
    cache.size = n_ctx; // 缓存大小为 n_ctx

    cache.cells.clear(); // 清空缓存单元
    cache.cells.resize(n_ctx); // 重新设置缓存大小
    # 调整缓冲区大小，以容纳指定数量的元素和指定类型的数据，同时考虑了张量的开销
    cache.buf.resize(2u*n_elements*ggml_type_size(wtype) + 2u*ggml_tensor_overhead());
    # 将缓冲区内存清零
    memset(cache.buf.data, 0, cache.buf.size);

    # 定义初始化参数结构体
    struct ggml_init_params params;
    # 设置内存大小
    params.mem_size   = cache.buf.size;
    # 设置内存缓冲区
    params.mem_buffer = cache.buf.data;
    # 允许分配内存
    params.no_alloc   = false;

    # 初始化缓存上下文
    cache.ctx = ggml_init(params);

    # 如果初始化失败，则记录错误信息并返回 false
    if (!cache.ctx) {
        LLAMA_LOG_ERROR("%s: failed to allocate memory for kv cache\n", __func__);
        return false;
    }

    # 创建新的一维张量作为缓存的键和值
    cache.k = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);
    cache.v = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);
    # 设置键和值张量的名称
    ggml_set_name(cache.k, "cache_k");
    ggml_set_name(cache.v, "cache_v");

    # 忽略 n_gpu_layers 变量
    (void) n_gpu_layers;
#ifdef GGML_USE_CUBLAS
    // 检查是否启用了 CUBLAS
    if (ggml_cublas_loaded()) {
        // 初始化 VRAM kv 缓存大小
        size_t vram_kv_cache = 0;

        // 如果 GPU 层数大于当前层数加1
        if (n_gpu_layers > (int)n_layer + 1) {
            // 分配缓存并将 v 缓存数据传输到 GPU
            ggml_cuda_assign_buffers_no_scratch(cache.v);
            // 输出信息：将 v 缓存数据传输到 GPU
            LLAMA_LOG_INFO("%s: offloading v cache to GPU\n", __func__);
            // 计算 v 缓存数据大小
            vram_kv_cache += ggml_nbytes(cache.v);
        }
        // 如果 GPU 层数大于当前层数加2
        if (n_gpu_layers > (int)n_layer + 2) {
            // 分配缓存并将 k 缓存数据传输到 GPU
            ggml_cuda_assign_buffers_no_scratch(cache.k);
            // 输出信息：将 k 缓存数据传输到 GPU
            LLAMA_LOG_INFO("%s: offloading k cache to GPU\n", __func__);
            // 计算 k 缓存数据大小
            vram_kv_cache += ggml_nbytes(cache.k);
        }
        // 如果 VRAM kv 缓存大小大于0
        if (vram_kv_cache > 0) {
            // 输出信息：VRAM kv self 大小
            LLAMA_LOG_INFO("%s: VRAM kv self = %.2f MB\n", __func__, vram_kv_cache / 1024.0 / 1024.0);
        }
    }
#endif

    // 返回 true
    return true;
}

// 查找缓存中大小为 "n_tokens" 的空槽位
// 更新缓存头部
// 注意：成功时，重要的是缓存头部指向槽位的第一个单元格
static bool llama_kv_cache_find_slot(
           struct llama_kv_cache & cache,
        const struct llama_batch & batch) {
    // 获取缓存大小和 tokens 数量
    const uint32_t n_ctx    = cache.size;
    const uint32_t n_tokens = batch.n_tokens;

    // 如果 tokens 数量大于缓存大小
    if (n_tokens > n_ctx) {
        // 输出错误信息：tokens 数量大于缓存大小
        LLAMA_LOG_ERROR("%s: n_tokens=%d > n_ctx=%d\n", __func__, n_tokens, n_ctx);
        // 返回 false
        return false;
    }

    // 初始化测试次数
    uint32_t n_tested = 0;

    // 循环查找空槽位
    while (true) {
        // 如果头部加上 tokens 数量大于缓存大小
        if (cache.head + n_tokens > n_ctx) {
            // 更新测试次数
            n_tested += n_ctx - cache.head;
            // 重置头部为0
            cache.head = 0;
            // 继续下一轮循环
            continue;
        }

        // 初始化找到标志
        bool found = true;
        // 遍历 tokens 数量
        for (uint32_t i = 0; i < n_tokens; i++) {
            // 如果缓存中的单元格位置大于等于0
            if (cache.cells[cache.head + i].pos >= 0) {
                // 设置未找到标志，更新头部和测试次数
                found = false;
                cache.head += i + 1;
                n_tested   += i + 1;
                break;
            }
        }

        // 如果找到标志为真，跳出循环
        if (found) {
            break;
        }

        // 如果测试次数大于等于缓存大小
        if (n_tested >= n_ctx) {
            // 输出错误信息：无法找到 tokens 数量的槽位
            //LLAMA_LOG_ERROR("%s: failed to find a slot for %d tokens\n", __func__, n_tokens);
            // 返回 false
            return false;
        }
    }
    # 遍历从 0 到 n_tokens 的整数
    for (uint32_t i = 0; i < n_tokens; i++) {
        # 将 batch.pos[i] 的值赋给 cache.cells[cache.head + i].pos
        cache.cells[cache.head + i].pos = batch.pos[i];

        # 遍历从 0 到 batch.n_seq_id[i] 的整数
        for (int32_t j = 0; j < batch.n_seq_id[i]; j++) {
            # 将 batch.seq_id[i][j] 插入到 cache.cells[cache.head + i].seq_id 集合中
            cache.cells[cache.head + i].seq_id.insert(batch.seq_id[i][j]);
        }
    }

    # 返回 true
    return true;
// 返回当前使用的单元格数量
static int32_t llama_kv_cache_cell_max(const struct llama_kv_cache & cache) {
    // 从最后一个单元格开始向前遍历
    for (uint32_t i = cache.size - 1; i > 0; --i) {
        // 如果单元格的位置大于等于0且序列ID不为空，则返回当前索引加1
        if (cache.cells[i].pos >= 0 && !cache.cells[i].seq_id.empty()) {
            return i + 1;
        }
    }
    // 如果没有符合条件的单元格，则返回0
    return 0;
}

// 清空缓存中的所有单元格
static void llama_kv_cache_clear(struct llama_kv_cache & cache) {
    // 遍历所有单元格，将位置设为-1，序列ID清空
    for (int32_t i = 0; i < (int32_t) cache.size; ++i) {
        cache.cells[i].pos = -1;
        cache.cells[i].seq_id.clear();
    }
    // 将头指针设为0
    cache.head = 0;
}

// 移除指定序列ID在指定位置范围内的单元格
static void llama_kv_cache_seq_rm(
        struct llama_kv_cache & cache,
                 llama_seq_id   seq_id,
                    llama_pos   p0,
                    llama_pos   p1) {
    uint32_t new_head = cache.size;

    // 如果p0小于0，则将其设为0
    if (p0 < 0) p0 = 0;
    // 如果p1小于0，则将其设为llama_pos类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 遍历所有单元格
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果单元格的位置在指定范围内
        if (cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 如果seq_id小于0，则清空单元格的序列ID
            if (seq_id < 0) {
                cache.cells[i].seq_id.clear();
            } 
            // 如果单元格包含指定的seq_id，则从序列ID中删除该ID
            else if (cache.cells[i].has_seq_id(seq_id)) {
                cache.cells[i].seq_id.erase(seq_id);
            } 
            // 如果不包含指定的seq_id，则继续下一次循环
            else {
                continue;
            }
            // 如果单元格的序列ID为空，则将位置设为-1，并更新new_head
            if (cache.cells[i].seq_id.empty()) {
                cache.cells[i].pos = -1;
                if (new_head == cache.size) new_head = i;
            }
        }
    }

    // 如果释放了一个单元格，则将头指针设为新的位置
    if (new_head != cache.size) cache.head = new_head;
}

// 复制指定序列ID在指定位置范围内的单元格
static void llama_kv_cache_seq_cp(
        struct llama_kv_cache & cache,
                 llama_seq_id   seq_id_src,
                 llama_seq_id   seq_id_dst,
                    llama_pos   p0,
                    llama_pos   p1) {
    // 如果p0小于0，则将其设为0
    if (p0 < 0) p0 = 0;
    // 如果p1小于0，则将其设为llama_pos类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();
    // 将头指针设为0
    cache.head = 0;
}
    # 遍历缓存中的元素，从 0 到缓存大小
    for (uint32_t i = 0; i < cache.size; ++i) {
        # 检查缓存中的单元格是否具有指定的序列 ID，并且位置在 p0 和 p1 之间
        if (cache.cells[i].has_seq_id(seq_id_src) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            # 如果满足条件，则将目标序列 ID 插入到单元格的序列 ID 集合中
            cache.cells[i].seq_id.insert(seq_id_dst);
        }
    }
}

// 保持序列的键值缓存
static void llama_kv_cache_seq_keep(struct llama_kv_cache & cache, llama_seq_id seq_id) {
    // 新的头部位置初始化为缓存的大小
    uint32_t new_head = cache.size;

    // 遍历缓存中的每个单元
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果单元中不包含指定序列号
        if (!cache.cells[i].has_seq_id(seq_id)) {
            // 将单元的位置设置为-1，清空序列号
            cache.cells[i].pos = -1;
            cache.cells[i].seq_id.clear();
            // 如果新的头部位置还未被设置，则将其设置为当前位置
            if (new_head == cache.size) new_head = i;
        } else {
            // 清空单元中的序列号
            cache.cells[i].seq_id.clear();
            // 插入指定的序列号
            cache.cells[i].seq_id.insert(seq_id);
        }
    }

    // 如果释放了一个单元，将头部位置设置为释放的单元位置，以便下次搜索从该位置开始
    if (new_head != cache.size) cache.head = new_head;
}

// 移动序列的键值缓存
static void llama_kv_cache_seq_shift(
        struct llama_kv_cache & cache,
                 llama_seq_id   seq_id,
                    llama_pos   p0,
                    llama_pos   p1,
                    llama_pos   delta) {
    // 新的头部位置初始化为缓存的大小
    uint32_t new_head = cache.size;

    // 如果p0小于0，则将其设置为0
    if (p0 < 0) p0 = 0;
    // 如果p1小于0，则将其设置为llama_pos类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 遍历缓存中的每个单元
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果单元中包含指定序列号，并且位置在p0和p1之间
        if (cache.cells[i].has_seq_id(seq_id) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 标记缓存中存在移动操作
            cache.has_shift = true;
            // 更新单元的位置和增量
            cache.cells[i].pos   += delta;
            cache.cells[i].delta += delta;

            // 如果单元的位置小于0
            if (cache.cells[i].pos < 0) {
                // 将单元的位置设置为-1，清空序列号
                cache.cells[i].pos = -1;
                cache.cells[i].seq_id.clear();
                // 如果新的头部位置还未被设置，则将其设置为当前位置
                if (new_head == cache.size) new_head = i;
            }
        }
    }

    // 如果释放了一个单元，将头部位置设置为释放的单元位置，以便下次搜索从该位置开始
    // 否则，从头部位置开始下次搜索
    cache.head = new_head != cache.size ? new_head : 0;
}

//
// 模型加载和保存
//

// 枚举文件版本
enum llama_fver {
    GGUF_FILE_VERSION_V1 = 1,
    GGUF_FILE_VERSION_V2 = 2,
    GGUF_FILE_VERSION_V3 = 3,
};

// 返回文件版本的名称
static const char * llama_file_version_name(llama_fver version) {
    # 根据版本号进行切换
    switch (version) {
        # 如果版本号为 V1，则返回对应的说明
        case GGUF_FILE_VERSION_V1: return "GGUF V1 (support until nov 2023)";
        # 如果版本号为 V2，则返回对应的说明
        case GGUF_FILE_VERSION_V2: return "GGUF V2";
        # 如果版本号为 V3，则返回对应的说明
        case GGUF_FILE_VERSION_V3: return "GGUF V3 (latest)";
    }
    # 如果版本号不在已知的范围内，则返回未知
    return "unknown";
}

// 格式化张量形状的函数，接受一个整型向量作为参数
static std::string llama_format_tensor_shape(const std::vector<int64_t> & ne) {
    char buf[256];
    // 将第一个元素格式化为字符串并存储到缓冲区中
    snprintf(buf, sizeof(buf), "%5" PRId64, ne.at(0));
    // 遍历整型向量的剩余元素，格式化为字符串并追加到缓冲区中
    for (size_t i = 1; i < ne.size(); i++) {
        snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), ", %5" PRId64, ne.at(i));
    }
    // 返回格式化后的字符串
    return buf;
}

// 格式化张量形状的函数，接受一个 ggml_tensor 结构体指针作为参数
static std::string llama_format_tensor_shape(const struct ggml_tensor * t) {
    char buf[256];
    // 将第一个元素格式化为字符串并存储到缓冲区中
    snprintf(buf, sizeof(buf), "%5" PRId64, t->ne[0]);
    // 遍历 ggml_tensor 结构体的剩余元素，格式化为字符串并追加到缓冲区中
    for (int i = 1; i < GGML_MAX_DIMS; i++) {
        snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), ", %5" PRId64, t->ne[i]);
    }
    // 返回格式化后的字符串
    return buf;
}

// 定义名为 llama_model_loader 的结构体
struct llama_model_loader {
    int n_kv      = 0;  // 初始化 n_kv 为 0
    int n_tensors = 0;  // 初始化 n_tensors 为 0
    int n_created = 0;  // 初始化 n_created 为 0

    ggml_sparse_deriv sparse_deriv;  // 定义 ggml_sparse_deriv 类型的成员变量

    int64_t n_elements = 0;  // 初始化 n_elements 为 0
    size_t  n_bytes    = 0;  // 初始化 n_bytes 为 0

    bool use_mmap = false;  // 初始化 use_mmap 为 false

    llama_file  file;  // 定义 llama_file 类型的成员变量
    llama_ftype ftype;  // 定义 llama_ftype 类型的成员变量
    llama_fver  fver;   // 定义 llama_fver 类型的成员变量

    std::unique_ptr<llama_mmap> mapping;  // 定义 unique_ptr 类型的成员变量，指向 llama_mmap 类型

    struct gguf_context * ctx_gguf = NULL;  // 定义 gguf_context 结构体指针成员变量，初始化为 NULL
    struct ggml_context * ctx_meta = NULL;  // 定义 ggml_context 结构体指针成员变量，初始化为 NULL

    }

    // 定义析构函数
    ~llama_model_loader() {
        // 如果 ctx_gguf 不为空，则释放内存
        if (ctx_gguf) {
            gguf_free(ctx_gguf);
        }
        // 如果 ctx_meta 不为空，则释放内存
        if (ctx_meta) {
            ggml_free(ctx_meta);
        }
    }

    // 获取架构名称的函数
    std::string get_arch_name() const {
        const auto kv = LLM_KV(LLM_ARCH_UNKNOWN);

        std::string arch_name;
        // 通过 gguf_context 获取架构名称
        GGUF_GET_KEY(ctx_gguf, arch_name, gguf_get_val_str, GGUF_TYPE_STRING, false, kv(LLM_KV_GENERAL_ARCHITECTURE));

        return arch_name;
    }

    // 获取架构类型的函数
    enum llm_arch get_arch() const {
        const std::string arch_name = get_arch_name();

        // 通过架构名称获取架构类型
        return llm_arch_from_string(arch_name);
    }

    // 获取张量名称的函数
    const char * get_tensor_name(int i) const {
        return gguf_get_tensor_name(ctx_gguf, i);
    }

    // 获取张量元数据的函数
    struct ggml_tensor * get_tensor_meta(int i) const {
        return ggml_get_tensor(ctx_meta, get_tensor_name(i));
    }
    // 计算上下文大小和内存映射大小
    void calc_sizes(size_t & ctx_size_p, size_t & mmapped_size_p) const {
        // 初始化上下文大小和内存映射大小
        ctx_size_p     = 0;
        mmapped_size_p = 0;

        // 遍历所有张量
        for (int i = 0; i < n_tensors; i++) {
            // 获取张量的元数据
            struct ggml_tensor * meta = get_tensor_meta(i);
            // 计算上下文大小
            ctx_size_p += sizeof(struct ggml_tensor) + GGML_OBJECT_SIZE;
            // 根据是否使用内存映射，计算内存映射大小或上下文大小
            (use_mmap ? mmapped_size_p : ctx_size_p) += ggml_nbytes_pad(meta);
        }
    }

    // 为给定上下文和元数据创建张量
    struct ggml_tensor * create_tensor_for(struct ggml_context * ctx, struct ggml_tensor * meta, ggml_backend_type backend) {
        // 如果后端不是 CPU，则设置不分配内存
        if (backend != GGML_BACKEND_CPU) {
            ggml_set_no_alloc(ctx, true);
        }

        // 复制元数据创建张量
        struct ggml_tensor * tensor = ggml_dup_tensor(ctx, meta);
        // 设置张量的后端
        tensor->backend = backend; // TODO: ggml_set_backend
        // 设置张量的名称
        ggml_set_name(tensor, ggml_get_name(meta));

        // 如果后端不是 CPU，则根据是否使用内存映射设置不分配内存
        if (backend != GGML_BACKEND_CPU) {
            ggml_set_no_alloc(ctx, use_mmap);
        }

        // 增加已创建张量的计数
        n_created++;

        // 返回创建的张量
        return tensor;
    }

    // 为给定上下文、张量名称、张量形状和后端类型创建张量
    struct ggml_tensor * create_tensor(struct ggml_context * ctx, const std::pair<std::string, tensor_offloading_levels> & tn, const std::vector<int64_t> & ne, ggml_backend_type backend) {
        return create_tensor(ctx, tn.first, ne, backend);
    }
    // 创建一个张量，根据给定的名称、维度和后端类型
    struct ggml_tensor * create_tensor(struct ggml_context * ctx, const std::string &name, const std::vector<int64_t> & ne, ggml_backend_type backend) {
        // 获取当前张量
        struct ggml_tensor * cur = ggml_get_tensor(ctx_meta, name.c_str());

        // 如果当前张量为空，抛出运行时错误
        if (cur == NULL) {
            throw std::runtime_error(format("%s: tensor '%s' not found", __func__, name.c_str()));
        }

        // 如果后端类型为 GGML_BACKEND_GPU_SPLIT
        if (backend == GGML_BACKEND_GPU_SPLIT) {
            // 如果维度大小为1，抛出运行时错误
            if (ne.size() == 1) {
                throw std::runtime_error(format("%s: 1-dimensional tensor '%s' cannot be split on the GPU", __func__, name.c_str()));
            }
        }

        {
            // 初始化布尔值为 true
            bool is_ok = true;
            // 遍历维度数组
            for (size_t i = 0; i < ne.size(); ++i) {
                // 如果维度不匹配当前张量的维度
                if (ne[i] != cur->ne[i]) {
                    // 允许维度数组中的 -1 作为通配符维度
                    is_ok = ne[i] == -1;
                    break;
                }
            }
            // 如果维度不匹配，抛出运行时错误
            if (!is_ok) {
                throw std::runtime_error(
                        format("%s: tensor '%s' has wrong shape; expected %s, got %s",
                            __func__, name.c_str(),
                            llama_format_tensor_shape(ne).c_str(),
                            llama_format_tensor_shape(cur).c_str()));
            }
        }

        // 根据当前张量和后端类型创建张量
        return create_tensor_for(ctx, cur, backend);
    }

    // 检查是否已经获取所有张量
    void done_getting_tensors() const {
        // 如果创建的张量数量不等于总张量数量，抛出运行时错误
        if (n_created != n_tensors) {
            throw std::runtime_error(format("%s: wrong number of tensors; expected %d, got %d", __func__, n_tensors, n_created));
        }
    }

    // 获取文件中张量的偏移量
    size_t file_offset(const char * name) const {
        // 查找张量在文件中的索引
        const int idx = gguf_find_tensor(ctx_gguf, name);

        // 如果索引小于0，抛出运行时错误
        if (idx < 0) {
            throw std::runtime_error(format("%s: tensor '%s' not found in the file", __func__, name));
        }

        // 返回数据偏移量
        return gguf_get_data_offset(ctx_gguf) + gguf_get_tensor_offset(ctx_gguf, idx);
    }
    // 为给定的 ggml_tensor 结构加载数据
    void load_data_for(struct ggml_tensor * cur) const {
        // 获取当前 ggml_tensor 的文件偏移量
        const size_t offs = file_offset(ggml_get_name(cur));

        // 如果使用内存映射，则将数据指针指向映射地址加上偏移量处
        if (use_mmap) {
            cur->data = (uint8_t *) mapping->addr + offs;
        } else {
            // 如果不使用内存映射，则将文件指针移动到偏移量处，然后读取数据到 ggml_tensor 结构中
            file.seek(offs, SEEK_SET);
            file.read_raw(cur->data, ggml_nbytes(cur));
        }
    }
    // 加载所有数据，包括数据大小、预取大小等
    void load_all_data(struct ggml_context * ctx, llama_progress_callback progress_callback, void * progress_callback_user_data, llama_mlock * lmlock) {
        // 初始化数据大小、锁定大小、预取大小
        size_t size_data = 0;
        size_t size_lock = 0;
        size_t size_pref = 0; // prefetch

        // 遍历所有张量，计算数据大小，并根据后端类型计算预取大小
        for (int i = 0; i < gguf_get_n_tensors(ctx_gguf); i++) {
            struct ggml_tensor * cur = ggml_get_tensor(ctx, gguf_get_tensor_name(ctx_gguf, i));
            size_data += ggml_nbytes(cur);
            if (cur->backend == GGML_BACKEND_CPU) {
                size_pref += ggml_nbytes(cur);
            }
        }

        // 如果使用内存映射，则创建内存映射对象，并根据需要初始化锁定对象
        if (use_mmap) {
            mapping.reset(new llama_mmap(&file, size_pref, ggml_is_numa()));
            if (lmlock) {
                lmlock->init(mapping->addr);
            }
        }

        // 初始化已加载数据大小
        size_t done_size = 0;
        // 遍历所有张量，加载数据并根据需要调用进度回调函数
        for (int i = 0; i < gguf_get_n_tensors(ctx_gguf); i++) {
            struct ggml_tensor * cur = ggml_get_tensor(ctx, gguf_get_tensor_name(ctx_gguf, i));
            GGML_ASSERT(cur); // 未使用的张量应该已经在load_data中被捕获

            if (progress_callback) {
                progress_callback((float) done_size / size_data, progress_callback_user_data);
            }

            // 如果不使用内存映射且数据为空，则分配临时缓冲区
            if (!use_mmap && cur->data == NULL) {
                GGML_ASSERT(cur->backend != GGML_BACKEND_CPU);
                #ifdef GGML_USE_CPU_HBM
                cur->data = (uint8_t*)hbw_malloc(ggml_nbytes(cur));
                #else
                cur->data = (uint8_t*)malloc(ggml_nbytes(cur));
                #endif
            }

            // 加载数据
            load_data_for(cur);

            // 根据后端类型进行处理
            switch (cur->backend) {
                case GGML_BACKEND_CPU:
                    // 如果使用内存映射且存在锁定对象，则增加锁定大小
                    if (use_mmap && lmlock) {
                        size_lock += ggml_nbytes(cur);
                        lmlock->grow_to(size_lock);
                    }
                    break;
#ifdef GGML_USE_CUBLAS
                // 如果使用了CUBLAS，则执行以下代码块
                case GGML_BACKEND_GPU:
                case GGML_BACKEND_GPU_SPLIT:
                    // 如果后端是GPU或者GPU_SPLIT，则执行以下代码块
                    // 旧代码：
                    //ggml_cuda_transform_tensor(lt.data, lt.ggml_tensor);

                    // TODO: test if this works !!
                    // TODO: 测试这个是否有效！！
                    ggml_cuda_transform_tensor(cur->data, cur);
                    // 使用CUDA库对张量进行转换
                    if (!use_mmap) {
                        // 如果不使用内存映射，则释放当前数据
                        free(cur->data);
                    }
                    // 跳出switch语句
                    break;
#elif defined(GGML_USE_CLBLAST)
                // 如果使用了CLBLAST，则执行以下代码块
                case GGML_BACKEND_GPU:
                    // 如果后端是GPU，则执行以下代码块
                    ggml_cl_transform_tensor(cur->data, cur);
                    // 使用CLBLAST库对张量进行转换
                    if (!use_mmap) {
                        // 如果不使用内存映射，则释放当前数据
                        free(cur->data);
                    }
                    // 跳出switch语句
                    break;
#endif
                // 如果以上条件都不满足，则执行以下代码块
                default:
                    // 继续下一次循环
                    continue;
            }

            // 计算已完成的数据大小
            done_size += ggml_nbytes(cur);
        }
    }
};

//
// load LLaMA models
//

// 获取LLaMA模型架构名称
static std::string llama_model_arch_name(llm_arch arch) {
    // 在LLM_ARCH_NAMES中查找对应的架构名称
    auto it = LLM_ARCH_NAMES.find(arch);
    // 如果未找到对应的架构名称，则返回"unknown"
    if (it == LLM_ARCH_NAMES.end()) {
        return "unknown";
    }
    // 返回对应的架构名称
    return it->second;
}

// 获取LLaMA模型文件类型名称
static std::string llama_model_ftype_name(llama_ftype ftype) {
    // 如果文件类型是LLAMA_FTYPE_GUESSED，则返回去除LLAMA_FTYPE_GUESSED标志后的文件类型名称，并加上"(guessed)"后缀
    if (ftype & LLAMA_FTYPE_GUESSED) {
        return llama_model_ftype_name((enum llama_ftype) (ftype & ~LLAMA_FTYPE_GUESSED)) + " (guessed)";
    }
    # 根据文件类型进行不同的处理
    switch (ftype) {
        # 如果文件类型是LLAMA_FTYPE_ALL_F32，则返回字符串"all F32"
        case LLAMA_FTYPE_ALL_F32:     return "all F32";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_F16，则返回字符串"mostly F16"
        case LLAMA_FTYPE_MOSTLY_F16:  return "mostly F16";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q4_0，则返回字符串"mostly Q4_0"
        case LLAMA_FTYPE_MOSTLY_Q4_0: return "mostly Q4_0";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q4_1，则返回字符串"mostly Q4_1"
        case LLAMA_FTYPE_MOSTLY_Q4_1: return "mostly Q4_1";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q4_1_SOME_F16，则返回字符串"mostly Q4_1, some F16"
        case LLAMA_FTYPE_MOSTLY_Q4_1_SOME_F16:
                                      return "mostly Q4_1, some F16";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q5_0，则返回字符串"mostly Q5_0"
        case LLAMA_FTYPE_MOSTLY_Q5_0: return "mostly Q5_0";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q5_1，则返回字符串"mostly Q5_1"
        case LLAMA_FTYPE_MOSTLY_Q5_1: return "mostly Q5_1";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q8_0，则返回字符串"mostly Q8_0"
        case LLAMA_FTYPE_MOSTLY_Q8_0: return "mostly Q8_0";

        # K-quants
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q2_K，则返回字符串"mostly Q2_K"
        case LLAMA_FTYPE_MOSTLY_Q2_K:   return "mostly Q2_K";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q3_K_S，则返回字符串"mostly Q3_K - Small"
        case LLAMA_FTYPE_MOSTLY_Q3_K_S: return "mostly Q3_K - Small";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q3_K_M，则返回字符串"mostly Q3_K - Medium"
        case LLAMA_FTYPE_MOSTLY_Q3_K_M: return "mostly Q3_K - Medium";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q3_K_L，则返回字符串"mostly Q3_K - Large"
        case LLAMA_FTYPE_MOSTLY_Q3_K_L: return "mostly Q3_K - Large";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q4_K_S，则返回字符串"mostly Q4_K - Small"
        case LLAMA_FTYPE_MOSTLY_Q4_K_S: return "mostly Q4_K - Small";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q4_K_M，则返回字符串"mostly Q4_K - Medium"
        case LLAMA_FTYPE_MOSTLY_Q4_K_M: return "mostly Q4_K - Medium";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q5_K_S，则返回字符串"mostly Q5_K - Small"
        case LLAMA_FTYPE_MOSTLY_Q5_K_S: return "mostly Q5_K - Small";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q5_K_M，则返回字符串"mostly Q5_K - Medium"
        case LLAMA_FTYPE_MOSTLY_Q5_K_M: return "mostly Q5_K - Medium";
        # 如果文件类型是LLAMA_FTYPE_MOSTLY_Q6_K，则返回字符串"mostly Q6_K"
        case LLAMA_FTYPE_MOSTLY_Q6_K:   return "mostly Q6_K";

        # 如果文件类型未知，则返回字符串"unknown, may not work"
        default: return "unknown, may not work";
    }
// 根据模型类型返回对应的名称
static const char * llama_model_type_name(e_model type) {
    switch (type) {
        case MODEL_1B:  return "1B";
        case MODEL_3B:  return "3B";
        case MODEL_7B:  return "7B";
        case MODEL_8B:  return "8B";
        case MODEL_13B: return "13B";
        case MODEL_15B: return "15B";
        case MODEL_30B: return "30B";
        case MODEL_34B: return "34B";
        case MODEL_40B: return "40B";
        case MODEL_65B: return "65B";
        case MODEL_70B: return "70B";
        default:        return "?B";
    }
}

// 加载模型的架构信息
static void llm_load_arch(llama_model_loader & ml, llama_model & model) {
    model.arch = ml.get_arch();
    // 如果模型架构未知，则抛出运行时错误
    if (model.arch == LLM_ARCH_UNKNOWN) {
        throw std::runtime_error("unknown model architecture: '" + ml.get_arch_name() + "'");
    }
}

// 加载模型的超参数信息
static void llm_load_hparams(
        llama_model_loader & ml,
        llama_model & model) {
    struct gguf_context * ctx = ml.ctx_gguf;

    const auto kv = LLM_KV(model.arch);

    auto & hparams = model.hparams;

    // 获取通用的键值对
    GGUF_GET_KEY(ctx, model.name, gguf_get_val_str, GGUF_TYPE_STRING, false, kv(LLM_KV_GENERAL_NAME));

    // 获取超参数的键值对
    GGUF_GET_KEY(ctx, hparams.n_vocab,        gguf_get_arr_n,   GGUF_TYPE_ARRAY,  true, kv(LLM_KV_TOKENIZER_LIST));
    GGUF_GET_KEY(ctx, hparams.n_ctx_train,    gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_CONTEXT_LENGTH));
    GGUF_GET_KEY(ctx, hparams.n_embd,         gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_EMBEDDING_LENGTH));
    GGUF_GET_KEY(ctx, hparams.n_ff,           gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_FEED_FORWARD_LENGTH));
    GGUF_GET_KEY(ctx, hparams.n_head,         gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_ATTENTION_HEAD_COUNT));
    GGUF_GET_KEY(ctx, hparams.n_layer,        gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_BLOCK_COUNT));

    // n_head_kv 是可选的，如果不存在则默认为 n_head
    hparams.n_head_kv = hparams.n_head;
}
    // 从上下文中获取 hparams.n_head_kv 的值，并存储到 GGUF_GET_KEY 函数中
    GGUF_GET_KEY(ctx, hparams.n_head_kv, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ATTENTION_HEAD_COUNT_KV));

    // 设置 hparams.rope_finetuned 的初始值为 false
    hparams.rope_finetuned = false;
    // 从上下文中获取 hparams.rope_finetuned 的值，并存储到 GGUF_GET_KEY 函数中
    GGUF_GET_KEY(ctx, hparams.rope_finetuned, gguf_get_val_bool, GGUF_TYPE_BOOL, false, kv(LLM_KV_ROPE_SCALING_FINETUNED));

    // 设置 hparams.n_yarn_orig_ctx 的值为 hparams.n_ctx_train
    hparams.n_yarn_orig_ctx = hparams.n_ctx_train;
    // 从上下文中获取 hparams.n_yarn_orig_ctx 的值，并存储到 GGUF_GET_KEY 函数中
    GGUF_GET_KEY(ctx, hparams.n_yarn_orig_ctx, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ROPE_SCALING_ORIG_CTX_LEN));

    // 设置 hparams.rope_freq_base_train 的初始值为 10000.0f
    hparams.rope_freq_base_train = 10000.0f;
    // 从上下文中获取 hparams.rope_freq_base_train 的值，并存储到 GGUF_GET_KEY 函数中
    GGUF_GET_KEY(ctx, hparams.rope_freq_base_train, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_FREQ_BASE));

    // 设置 rope_scaling 的初始值为 "linear"
    std::string rope_scaling("linear");
    // 从上下文中获取 rope_scaling 的值，并存储到 GGUF_GET_KEY 函数中
    GGUF_GET_KEY(ctx, rope_scaling, gguf_get_val_str, GGUF_TYPE_STRING, false, kv(LLM_KV_ROPE_SCALING_TYPE));
    // 将字符串类型的 rope_scaling 转换为相应的枚举类型
    hparams.rope_scaling_type_train = llama_rope_scaling_type_from_string(rope_scaling);
    // 断言 hparams.rope_scaling_type_train 不是 LLAMA_ROPE_SCALING_UNSPECIFIED
    GGML_ASSERT(hparams.rope_scaling_type_train != LLAMA_ROPE_SCALING_UNSPECIFIED);

    // 设置 ropescale 的初始值为 0.0f
    float ropescale = 0.0f;
    // 从上下文中获取 ropescale 的值，并存储到 GGUF_GET_KEY 函数中
    GGUF_GET_KEY(ctx, ropescale, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_SCALING_FACTOR));
    // 如果 ropescale 的值为 0.0f，则尝试获取旧的键名对应的值
    if (ropescale == 0.0f) {
        GGUF_GET_KEY(ctx, ropescale, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_SCALE_LINEAR));
    }
    // 如果 ropescale 的值为 0.0f，则将 hparams.rope_freq_scale_train 的值设置为 1.0f，否则设置为 1.0f/ropescale
    hparams.rope_freq_scale_train = ropescale == 0.0f ? 1.0f : 1.0f/ropescale;

    // 对 n_rot 进行可选的合理性检查
    {
        # 计算旋转数量
        hparams.n_rot = hparams.n_embd / hparams.n_head;

        # 从上下文中获取键值对，键为LLM_KV_ROPE_DIMENSION_COUNT，值为无符号32位整数，存入hparams.n_rot
        GGUF_GET_KEY(ctx, hparams.n_rot, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ROPE_DIMENSION_COUNT));

        # 如果模型架构为LLAMA或FALCON，则进行以下判断
        if (model.arch == LLM_ARCH_LLAMA || model.arch == LLM_ARCH_FALCON) {
            # 如果n_rot不等于n_embd / n_head，则抛出运行时错误
            if (hparams.n_rot != hparams.n_embd / hparams.n_head) {
                throw std::runtime_error(format("invalid n_rot: %u, expected %u", hparams.n_rot, hparams.n_embd / hparams.n_head));
            }
        }
        // gpt-neox n_rot = rotary_pct * (n_embd / n_head)
        // gpt-j n_rot = rotary_dim
    }

    # 如果gguf_get_sparse_deriv返回true，则执行以下代码
    if (gguf_get_sparse_deriv(ctx)) {
        # 如果稀疏导数被启用，则读取稀疏阈值覆盖
        GGUF_GET_KEY(ctx, hparams.sparse_pred_threshold, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_SPARSE_THRESHOLD));
    }

    # 设置模型的ftype为ml的ftype
    model.ftype = ml.ftype;
// TODO: This should probably be in llama.h
// 声明一个静态函数 llama_tokenize_internal，接受词汇表、原始文本、是否为开头、是否为特殊字符，并返回词汇表的标识符向量
static std::vector<llama_vocab::id> llama_tokenize_internal(const llama_vocab & vocab, std::string raw_text, bool bos, bool special = false);
// 声明一个静态函数 llama_byte_to_token，接受词汇表和一个字节，返回对应的词汇标识符
static llama_token llama_byte_to_token(const llama_vocab & vocab, uint8_t ch);

// 定义一个函数 llm_load_vocab，接受模型加载器和模型对象作为参数
static void llm_load_vocab(
        llama_model_loader & ml,
        llama_model & model) {
    // 获取模型对象中的词汇表
    auto & vocab = model.vocab;

    // 获取模型加载器的 gguf 上下文
    struct gguf_context * ctx = ml.ctx_gguf;

    // 获取模型架构中的键值对
    const auto kv = LLM_KV(model.arch);

    // 在键值对中查找标记器列表的索引
    const int token_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_LIST).c_str());
    // 如果找不到，则抛出运行时错误
    if (token_idx == -1) {
        throw std::runtime_error("cannot find tokenizer vocab in model file\n");
    }

    // 初始化分数数组为空
    const float * scores = nullptr;
    // 在键值对中查找标记器分数的索引
    const int score_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_SCORES).c_str());
    // 如果找到，则获取分数数组的数据
    if (score_idx != -1) {
        scores = (const float * ) gguf_get_arr_data(ctx, score_idx);
    }

    // 初始化标记类型数组为空
    const int * toktypes = nullptr;
    // 在键值对中查找标记器标记类型的索引
    const int toktype_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_TOKEN_TYPE).c_str());
    // 如果找到，则获取标记类型数组的数据
    if (toktype_idx != -1) {
        toktypes = (const int * ) gguf_get_arr_data(ctx, toktype_idx);
    }

    // 确定词汇表类型
}
    {
        // 声明一个字符串变量用于存储分词器名称
        std::string tokenizer_name;

        // 从上下文中获取键值为 LLM_KV_TOKENIZER_MODEL 的字符串类型数值，存储到 tokenizer_name 变量中
        GGUF_GET_KEY(ctx, tokenizer_name, gguf_get_val_str, GGUF_TYPE_STRING, true, kv(LLM_KV_TOKENIZER_MODEL));

        // 如果分词器名称为 "llama"，则设置词汇表类型为 LLAMA_VOCAB_TYPE_SPM
        if (tokenizer_name == "llama") {
            vocab.type = LLAMA_VOCAB_TYPE_SPM;

            // 设置默认特殊标记
            vocab.special_bos_id = 1;
            vocab.special_eos_id = 2;
            vocab.special_unk_id = 0;
            vocab.special_sep_id = -1;
            vocab.special_pad_id = -1;
        } 
        // 如果分词器名称为 "gpt2"，则设置词汇表类型为 LLAMA_VOCAB_TYPE_BPE
        else if (tokenizer_name == "gpt2") {
            vocab.type = LLAMA_VOCAB_TYPE_BPE;

            // 读取 BPE 合并并填充 BPE 排名
            const int merges_keyidx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_MERGES).c_str());
            if (merges_keyidx == -1) {
                throw std::runtime_error("cannot find tokenizer merges in model file\n");
            }

            const int n_merges = gguf_get_arr_n(ctx, merges_keyidx);

            for (int i = 0; i < n_merges; i++) {
                const std::string word = gguf_get_arr_str(ctx, merges_keyidx, i);
                GGML_ASSERT(codepoints_from_utf8(word).size() > 0);

                std::string first;
                std::string second;

                const size_t pos = word.find(' ', 1);

                if (pos != std::string::npos) {
                    first  = word.substr(0, pos);
                    second = word.substr(pos + 1);
                }

                vocab.bpe_ranks.emplace(std::make_pair(first, second), i);
            }

            // 设置默认特殊标记
            vocab.special_bos_id = 11;
            vocab.special_eos_id = 11;
            vocab.special_unk_id = -1;
            vocab.special_sep_id = -1;
            vocab.special_pad_id = -1;
        } 
        // 如果分词器名称不是 "llama" 也不是 "gpt2"，则记录警告信息并使用默认的分词器类型 LLAMA_VOCAB_TYPE_SPM
        else {
            LLAMA_LOG_WARN("%s: unknown tokenizer: '%s'", __func__, tokenizer_name.c_str());
            LLAMA_LOG_WARN("%s: using default tokenizer: 'llama'", __func__);

            vocab.type = LLAMA_VOCAB_TYPE_SPM;
        }
    // 获取上下文中标记索引处的数组长度
    const uint32_t n_vocab = gguf_get_arr_n(ctx, token_idx);

    // 调整词汇表的大小以容纳词汇量
    vocab.id_to_token.resize(n_vocab);

    // 遍历词汇表，为每个词汇设置对应的 ID、文本、分数和类型
    for (uint32_t i = 0; i < n_vocab; i++) {
        // 从上下文中获取指定索引处的字符串
        std::string word = gguf_get_arr_str(ctx, token_idx, i);
        // 确保从 UTF-8 转换为码点后的字符串长度大于 0
        GGML_ASSERT(codepoints_from_utf8(word).size() > 0);

        // 将词汇与其 ID 关联起来
        vocab.token_to_id[word] = i;

        // 获取词汇对应的数据，并设置其文本、分数和类型
        auto & token_data = vocab.id_to_token[i];
        token_data.text  = std::move(word);
        token_data.score = scores ? scores[i] : 0.0f;
        token_data.type  = toktypes ? (llama_token_type) toktypes[i] : LLAMA_TOKEN_TYPE_NORMAL;
    }
    // 确保词汇表的大小与词汇对应的 ID 数量相等
    GGML_ASSERT(vocab.id_to_token.size() == vocab.token_to_id.size());

    // 确定换行符的标记：如果词汇表类型为 LLAMA_VOCAB_TYPE_SPM，则使用 '\n' 的标记；否则，使用 "\u010A" 的标记
    if (vocab.type == LLAMA_VOCAB_TYPE_SPM) {
        vocab.linefeed_id = llama_byte_to_token(vocab, '\n');
    } else {
        // 在模型词汇表中查找换行符的标记
        const std::vector<int> ids = llama_tokenize_internal(vocab, "\u010A", false);
        GGML_ASSERT(!ids.empty() && "model vocab missing newline token");
        vocab.linefeed_id = ids[0];
    }

    // 特殊标记
    {
        // 定义包含特殊标记类型和对应整数引用的向量
        const std::vector<std::pair<enum llm_kv, int32_t &>> special_token_types = {
            { LLM_KV_TOKENIZER_BOS_ID, vocab.special_bos_id },
            { LLM_KV_TOKENIZER_EOS_ID, vocab.special_eos_id },
            { LLM_KV_TOKENIZER_UNK_ID, vocab.special_unk_id },
            { LLM_KV_TOKENIZER_SEP_ID, vocab.special_sep_id },
            { LLM_KV_TOKENIZER_PAD_ID, vocab.special_pad_id },
        };
        // 遍历特殊标记类型向量
        for (const auto & it : special_token_types) {
            // 获取键值对应的字符串键和整数引用
            const std::string & key = kv(std::get<0>(it));
            int32_t & id = std::get<1>(it), old_id = id;
    
            // 从上下文中获取键对应的值，如果不存在则使用默认值
            GGUF_GET_KEY(ctx, id, gguf_get_val_u32, GGUF_TYPE_UINT32, false, key);
            // 必须大于等于 -1 并且小于词汇表大小。由于键是无符号的，-1 只能来自默认值，所以没有必要验证。
            if (size_t(id + 1) > vocab.id_to_token.size()) {
                // 如果特殊标记的 id 超出了词汇表大小范围，则记录警告并使用默认 id
                LLAMA_LOG_WARN("%s: bad special token: '%s' = %d, using default id %d\n",
                    __func__, key.c_str(), id, old_id);
                id = old_id;
            }
        }
    }
    
    // 构建特殊标记缓存
    }
# 打印模型的元数据信息
static void llm_load_print_meta(llama_model_loader & ml, llama_model & model) {
    # 获取模型的超参数和词汇表
    const auto & hparams = model.hparams;
    const auto & vocab   = model.vocab;

    # 获取绳索缩放类型
    const auto rope_scaling_type = LLAMA_ROPE_SCALING_TYPES.at(hparams.rope_scaling_type_train);

    # 打印模型的元数据信息
    LLAMA_LOG_INFO("%s: format           = %s\n",     __func__, llama_file_version_name(ml.fver));
    LLAMA_LOG_INFO("%s: arch             = %s\n",     __func__, LLM_ARCH_NAMES.at(model.arch).c_str());
    LLAMA_LOG_INFO("%s: vocab type       = %s\n",     __func__, vocab.type == LLAMA_VOCAB_TYPE_SPM ? "SPM" : "BPE"); // TODO: fix
    LLAMA_LOG_INFO("%s: n_vocab          = %u\n",     __func__, hparams.n_vocab);
    LLAMA_LOG_INFO("%s: n_merges         = %u\n",     __func__, (int) vocab.bpe_ranks.size());
    LLAMA_LOG_INFO("%s: n_ctx_train      = %u\n",     __func__, hparams.n_ctx_train);
    LLAMA_LOG_INFO("%s: n_embd           = %u\n",     __func__, hparams.n_embd);
    LLAMA_LOG_INFO("%s: n_head           = %u\n",     __func__, hparams.n_head);
    LLAMA_LOG_INFO("%s: n_head_kv        = %u\n",     __func__, hparams.n_head_kv);
    LLAMA_LOG_INFO("%s: n_layer          = %u\n",     __func__, hparams.n_layer);
    LLAMA_LOG_INFO("%s: n_rot            = %u\n",     __func__, hparams.n_rot); // a.k.a. n_embd_head, n_head_dim
    LLAMA_LOG_INFO("%s: n_gqa            = %u\n",     __func__, hparams.n_gqa());
    LLAMA_LOG_INFO("%s: f_norm_eps       = %.1e\n",   __func__, hparams.f_norm_eps);
    LLAMA_LOG_INFO("%s: f_norm_rms_eps   = %.1e\n",   __func__, hparams.f_norm_rms_eps);
    LLAMA_LOG_INFO("%s: f_clamp_kqv      = %.1e\n",   __func__, hparams.f_clamp_kqv);
    LLAMA_LOG_INFO("%s: f_max_alibi_bias = %.1e\n",   __func__, hparams.f_max_alibi_bias);
    LLAMA_LOG_INFO("%s: n_ff             = %u\n",     __func__, hparams.n_ff);
    LLAMA_LOG_INFO("%s: rope scaling     = %s\n",     __func__, rope_scaling_type.c_str());
}
    // 输出频率基础训练值
    LLAMA_LOG_INFO("%s: freq_base_train  = %.1f\n",   __func__, hparams.rope_freq_base_train);
    // 输出频率比例训练值
    LLAMA_LOG_INFO("%s: freq_scale_train = %g\n",     __func__, hparams.rope_freq_scale_train);
    // 输出原始上下文中的纱线数量
    LLAMA_LOG_INFO("%s: n_yarn_orig_ctx  = %u\n",     __func__, hparams.n_yarn_orig_ctx);
    // 输出绳子微调的信息
    LLAMA_LOG_INFO("%s: rope_finetuned   = %s\n",     __func__, hparams.rope_finetuned ? "yes" : "unknown");
    // 输出模型类型
    LLAMA_LOG_INFO("%s: model type       = %s\n",     __func__, llama_model_type_name(model.type));
    // 输出模型特征类型
    LLAMA_LOG_INFO("%s: model ftype      = %s\n",     __func__, llama_model_ftype_name(model.ftype).c_str());
    // 输出模型参数数量
    LLAMA_LOG_INFO("%s: model params     = %.2f B\n", __func__, ml.n_elements*1e-9);
    // 如果模型字节数小于 GB，则输出模型大小（以 MiB 和 BPW 表示）
    if (ml.n_bytes < GB) {
        LLAMA_LOG_INFO("%s: model size       = %.2f MiB (%.2f BPW) \n", __func__, ml.n_bytes/1024.0/1024.0, ml.n_bytes*8.0/ml.n_elements);
    } else {
        // 如果模型字节数大于等于 GB，则输出模型大小（以 GiB 和 BPW 表示）
        LLAMA_LOG_INFO("%s: model size       = %.2f GiB (%.2f BPW) \n", __func__, ml.n_bytes/1024.0/1024.0/1024.0, ml.n_bytes*8.0/ml.n_elements);
    }

    // 一般键值
    LLAMA_LOG_INFO("%s: general.name   = %s\n",    __func__, model.name.c_str());

    // 特殊标记
    // 如果特殊起始标记 ID 不为 -1，则输出起始标记信息
    if (vocab.special_bos_id != -1) { LLAMA_LOG_INFO( "%s: BOS token = %d '%s'\n", __func__, vocab.special_bos_id, vocab.id_to_token[vocab.special_bos_id].text.c_str() ); }
    // 如果特殊结束标记 ID 不为 -1，则输出结束标记信息
    if (vocab.special_eos_id != -1) { LLAMA_LOG_INFO( "%s: EOS token = %d '%s'\n", __func__, vocab.special_eos_id, vocab.id_to_token[vocab.special_eos_id].text.c_str() ); }
    // 如果特殊未知标记 ID 不为 -1，则输出未知标记信息
    if (vocab.special_unk_id != -1) { LLAMA_LOG_INFO( "%s: UNK token = %d '%s'\n", __func__, vocab.special_unk_id, vocab.id_to_token[vocab.special_unk_id].text.c_str() ); }
    // 如果特殊分隔标记 ID 不为 -1，则输出分隔标记信息
    if (vocab.special_sep_id != -1) { LLAMA_LOG_INFO( "%s: SEP token = %d '%s'\n", __func__, vocab.special_sep_id, vocab.id_to_token[vocab.special_sep_id].text.c_str() ); }
    # 如果特殊填充标记不等于-1，则记录特殊填充标记的信息
    if (vocab.special_pad_id != -1) { LLAMA_LOG_INFO( "%s: PAD token = %d '%s'\n", __func__, vocab.special_pad_id, vocab.id_to_token[vocab.special_pad_id].text.c_str() ); }
    # 如果换行符标记不等于-1，则记录换行符标记的信息
    if (vocab.linefeed_id    != -1) { LLAMA_LOG_INFO( "%s: LF token  = %d '%s'\n", __func__, vocab.linefeed_id,    vocab.id_to_token[vocab.linefeed_id].text.c_str() );    }

    # 输出稀疏推断的阈值信息
    LLAMA_LOG_INFO("%s: sparse_pred_threshold = %.2f\n", __func__, hparams.sparse_pred_threshold);
// 定义了一个名为 llama_gpu_split_loader 的结构体
struct llama_gpu_split_loader {
    int n_tensors = 0; // 初始化张量数量为 0
    size_t n_bytes = 0; // 初始化张量数据字节数为 0

    const std::string fname; // 声明一个常量字符串 fname
    int fver; // 声明一个整数 fver

    bool use_mmap = false; // 初始化 use_mmap 为 false，目前只支持 mmap
    std::unique_ptr<llama_mmap> mapping; // 声明一个独占指针 mapping，指向 llama_mmap 类型的对象
    struct ggml_context * ctx_meta = nullptr; // 声明一个指向 ggml_context 类型的指针 ctx_meta，初始化为 nullptr

    llama_model_loader * idx_loader; // 声明一个指向 llama_model_loader 类型的指针 idx_loader
    size_t vram_required = 0; // 初始化所需 VRAM 大小为 0

    // 构造函数，接受文件名和是否使用 mmap 作为参数
    llama_gpu_split_loader(const std::string & fname, bool use_mmap) : fname(fname), use_mmap(use_mmap) {
        GGML_ASSERT(use_mmap); // 断言确保 use_mmap 为真

        // 通过文件名和是否使用 mmap 创建一个 llama_model_loader 对象
        idx_loader = new llama_model_loader(fname, use_mmap);
        // 从 idx_loader 的上下文中获取 VRAM 所需大小
        GGUF_GET_KEY(idx_loader->ctx_gguf, vram_required, gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_NAMES[LLM_KV_SPLIT_VRAM_CAPACITY]);
        // 打印加载的 gpu_idx 和所需的 VRAM 大小
        printf("loaded gpu_idx, vram_required: %ld\n", vram_required);

        // 将 n_tensors 设置为 idx_loader 中的张量数量
        n_tensors = idx_loader->n_tensors;

        // 分配 MLP 张量的元数据/数据内存
        // TODO: 支持为张量数据分配缓冲区（当不使用 mmap 时）
        size_t per_tensor_meta_size = GGML_PAD(sizeof(struct ggml_tensor), GGML_MEM_ALIGN) + GGML_OBJECT_SIZE;
        size_t tensor_meta_size = n_tensors * per_tensor_meta_size;
        // 初始化参数结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ tensor_meta_size, // 内存大小为张量元数据大小
            /*.mem_buffer =*/ nullptr, // 内存缓冲区为空
            /*.no_alloc   =*/ true, // 不分配内存
        };
        // 初始化 ggml 上下文
        ctx_meta = ggml_init(params);
    }

    // 检查是否可以分配所需的 VRAM
    bool check_vram_allocable(size_t vram_budget) {
        return vram_budget >= vram_required; // 返回是否 VRAM 预算大于等于所需的 VRAM
    }
}
    // 将张量应用到基础模型
    int apply_tensors_to_base_model(llama_model * model) {
        // 获取模型层数
        int n_layers = model->layers.size();
        // TODO: 断言文件指针在头部结束处
        if (n_tensors != n_layers * 2) {
           // 如果张量数量与模型层数不匹配，则记录错误并返回
           LLAMA_LOG_ERROR("%s: error: the number of gpu splits does not match the layer of model\n", __func__);
            return 1;
        }
        // 记录应用 GPU 索引适配器的日志信息
        LLAMA_LOG_INFO("%s: applying gpu_idx adapter from '%s' - please wait ...\n", __func__, fname.c_str());
        // 记录 MLP 开始时间
        const int64_t t_start_mlp_us = ggml_time_us();

        // 遍历每一层模型
        for (int il = 0; il < n_layers; il++) {
            // 获取模型的当前层
            llama_layer &model_layer = model->layers[il];
            // 从索引加载器获取 GPU 索引张量和 GPU 存储桶张量
            ggml_tensor * gpu_idx = idx_loader->get_tensor_meta(il*2);
            ggml_tensor * gpu_bucket = idx_loader->get_tensor_meta(il*2+1);
            // 如果获取的张量为空，则记录错误并返回
            if (gpu_idx == nullptr || gpu_bucket == nullptr) {
                LLAMA_LOG_ERROR("%s: error: failed to load gpu index or bucket\n", __func__);
                return 1;
            }
            // 为模型层创建 CPU 索引张量和 GPU 存储桶张量
            model_layer.gpu_idx = idx_loader->create_tensor_for(ctx_meta, gpu_idx, GGML_BACKEND_CPU);
            model_layer.gpu_bucket = idx_loader->create_tensor_for(ctx_meta, gpu_bucket, GGML_BACKEND_GPU);
        }
        // 定义进度回调函数，用于加载所有数据
        llama_progress_callback cb = [](float progress, void *ctx) {
            LLAMA_LOG_INFO(".");
        };
        // 使用索引加载器加载所有数据
        idx_loader->load_all_data(ctx_meta, cb, nullptr, nullptr);

        // 计算 MLP 运行时间
        const int64_t t_mlp_us = ggml_time_us() - t_start_mlp_us;
        // 记录 MLP 运行时间
        LLAMA_LOG_INFO(" done (%.2f ms)\n", t_mlp_us / 1000.0);

        // 返回成功
        return 0;
    }
// 用于动态加载/转换羊驼模型权重的结构体
struct llama_augmentation_model_loader {
    // 辅助上下文指针，默认为空
    struct ggml_context * aux_ctx = nullptr;

    // 构造函数，接受一个羊驼模型指针作为参数
    llama_augmentation_model_loader(llama_model *model) {
        // TODO: 检查前置条件 - MLP 已加载

        // 检查要加载的增强字段
        // 1. gpu_idx;
        // 2. gpu_bucket;
        // 3. transformed ffn_down;
        // 计算辅助张量的大小
        int model_layer = model->layers.size();
        int ffn_dim = model->layers[0].ffn_up->ne[1];
        const size_t ggml_aux_tensor_size = 4 * (model_layer*ffn_dim*sizeof(float)*2+ model_layer*ffn_dim*sizeof(float) * ggml_tensor_overhead() );

        // 初始化参数结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ ggml_aux_tensor_size,
            /*.mem_buffer =*/ nullptr,
            /*.no_alloc   =*/ false,
        };
        // 初始化辅助上下文
        aux_ctx = ggml_init(params);
    }

    // 创建分段矩阵到 GPU 的函数
    ggml_tensor * create_striped_mat_to_gpu(const struct ggml_tensor *src, struct ggml_tensor * gpu_bucket) {
        if (src == NULL) {
            return NULL;
        }
        // 分配内存并将选定的权重复制到 GPU
#ifdef GGML_USE_CUBLAS
        # 如果使用了CUBLAS，则执行以下代码块
        int64_t row_len = src->ne[0];
        # 获取输入张量的第一个维度大小
        int64_t gpu_rows = gpu_bucket->ne[0];
        # 获取GPU桶的第一个维度大小
        if (gpu_rows == 0)
            # 如果GPU桶的大小为0，则返回空指针
            return NULL;
        
        ggml_set_no_alloc(aux_ctx, true);
        # 设置不分配内存标志为真
        ggml_tensor * gpu_dst = ggml_new_tensor_2d(aux_ctx, src->type, row_len, gpu_rows);
        # 在辅助上下文中创建一个2D张量，用于存储GPU计算结果
        ggml_set_backend(gpu_dst, GGML_BACKEND_GPU);
        # 设置张量的后端为GPU
        ggml_cuda_alloc_tensor(gpu_dst);
        # 在GPU上分配张量的内存

        // init two 1d views on host and device
        # 在主机和设备上初始化两个1D视图
        ggml_tensor * host_mat_row = ggml_new_tensor_1d(aux_ctx, src->type, row_len);
        # 在辅助上下文中创建一个1D张量，用于存储主机端数据
        static ggml_tensor * device_mat_row = ggml_dup_tensor(aux_ctx, host_mat_row);
        # 在辅助上下文中复制主机端张量，用于存储设备端数据
        ggml_set_backend(device_mat_row, GGML_BACKEND_GPU);
        # 设置设备端张量的后端为GPU
        ggml_cuda_alloc_tensor(device_mat_row);
        # 在GPU上分配设备端张量的内存
        *ggml_cuda_get_data_pp(device_mat_row) = *ggml_cuda_get_data_pp(gpu_dst);
        # 将设备端张量的数据指针指向GPU计算结果的数据指针

        // read raw data and copy to device depending on gpu_idx
        # 读取原始数据并根据gpu_idx复制到设备
        const enum ggml_type type = src->type;
        # 获取输入张量的数据类型
        const int ne0 = src->ne[0];
        # 获取输入张量的第一个维度大小
        const size_t row_data_size = ne0*ggml_type_size(type)/ggml_blck_size(type);
        # 计算每行数据的大小
        for (int i = 0; i < gpu_rows; i++) {
            # 遍历每个GPU行
            int32_t host_i = ((int32_t *)gpu_bucket->data)[i];
            # 获取主机端索引
            host_mat_row -> data = src -> data + host_i * row_data_size;
            # 设置主机端行数据的指针
            char ** gpu_data_pp = reinterpret_cast<char **>(ggml_cuda_get_data_pp(device_mat_row));
            # 获取设备端数据的指针
            ggml_cuda_cpy_1d(device_mat_row, host_mat_row);
            # 将主机端行数据复制到设备端
            *gpu_data_pp = *gpu_data_pp + row_data_size;
            # 更新设备端数据指针
        }
        ggml_set_no_alloc(aux_ctx, false);
        # 设置不分配内存标志为假

        return gpu_dst;
        # 返回GPU计算结果张量
#else
        return NULL;
        # 如果未使用CUBLAS，则返回空指针
#endif
    }
        // 将分片的 FFN 数据传输到 GPU
        size_t slice_ffn_mat_to_gpu(llama_layer & layer) {
            // 创建一个工作缓冲区
            std::vector<uint8_t> work_buffer;
            // 获取层的 GPU 索引和 GPU 存储桶
            ggml_tensor * gpu_idx = layer.gpu_idx;
            ggml_tensor *gpu_bucket = layer.gpu_bucket;
            // 初始化已传输字节数
            size_t offloaded_bytes = 0;

            // 将 FFN gate 数据传输到 GPU
            layer.ffn_gate_gpu = create_striped_mat_to_gpu(layer.ffn_gate, gpu_bucket);
            // 将 FFN up 数据传输到 GPU
            layer.ffn_up_gpu = create_striped_mat_to_gpu(layer.ffn_up, gpu_bucket);
            // 将 FFN down 数据传输到 GPU
            layer.ffn_down_gpu = create_striped_mat_to_gpu(layer.ffn_down_t, gpu_bucket);
            
            // 如果 FFN gate 数据已传输到 GPU，则增加已传输字节数
            if (layer.ffn_gate_gpu) {
                offloaded_bytes += ggml_nbytes(layer.ffn_gate_gpu);
            }
            // 如果 FFN up 数据已传输到 GPU，则增加已传输字节数
            if (layer.ffn_up_gpu) {
                offloaded_bytes += ggml_nbytes(layer.ffn_up_gpu);
            }
            // 如果 FFN down 数据已传输到 GPU，则增加已传输字节数
            if (layer.ffn_down_gpu) {
                offloaded_bytes += ggml_nbytes(layer.ffn_down_gpu);
            }
            // 返回已传输字节数
            return offloaded_bytes;
        }

        // 将 FFN 分片数据传输到 GPU
        size_t offload_ffn_split(llama_model * model) {
            // 输出信息日志
            LLAMA_LOG_INFO("%s: applying augmentation to model - please wait ...\n", __func__);
            // 记录开始应用增强的时间
            const int64_t t_start_aug_us = ggml_time_us();
            // 创建一个工作缓冲区
            std::vector<uint8_t> work_buffer;

            // 通过全局变量设置稀疏预测阈值
            sparse_pred_threshold = model->hparams.sparse_pred_threshold;
#if defined (GGML_USE_CUBLAS)
        // 如果定义了 GGML_USE_CUBLAS，则设置设备常量
        ggml_cuda_set_device_constants(model->hparams.sparse_pred_threshold);
#endif

        // 加载 gpu_idx 和 slice mat 到 GPU
        size_t offloaded_bytes = 0;
        for (llama_layer &model_layer : model -> layers) {
            // gpu_idx 加载
            if (model_layer.gpu_idx == NULL && model_layer.gpu_bucket == NULL) {
                // 创建并初始化 gpu_idx
                ggml_tensor * gpu_idx = ggml_new_tensor_1d(aux_ctx, GGML_TYPE_I32, model_layer.mlp_pre_w2 -> ne[1]);
                ggml_set_zero(gpu_idx);
                model_layer.gpu_idx = gpu_idx;
                // 创建并初始化 gpu_bucket
                ggml_tensor * gpu_bucket = ggml_new_tensor_1d(aux_ctx, GGML_TYPE_I32, 0);
                model_layer.gpu_bucket = gpu_bucket;
            }
            // 将 slice ffn mat 到 GPU，并计算 offloaded_bytes
            offloaded_bytes += slice_ffn_mat_to_gpu(model_layer);
            // 记录日志
            LLAMA_LOG_INFO(".");
        }

        // 记录完成日志和时间
        LLAMA_LOG_INFO(" done (%.2f ms)\n", (ggml_time_us() - t_start_aug_us) / 1000.0);
        // 返回 offloaded_bytes
        return offloaded_bytes;
    }
};

// 定义结构体 buffered_tensor_allocator
struct buffered_tensor_allocator {
    llama_model_loader &ml;
    ggml_context *ctx;
    std::map<tensor_offloading_levels, std::vector<ggml_tensor *>> alloc_queues;
    const size_t vram_budget_bytes;
    size_t vram_allocated_bytes = 0;

    // 构造函数
    buffered_tensor_allocator(llama_model_loader &ml, ggml_context *ctx, size_t vram_budget_bytes) : ctx(ctx), ml(ml), vram_budget_bytes(vram_budget_bytes) {}

    // 分配缓冲张量
    ggml_tensor * buffered_alloc(const std::string & name, const tensor_offloading_levels &level, const std::vector<int64_t> & ne) {
#if defined(GGML_USE_CUBLAS)
        // 如果定义了 GGML_USE_CUBLAS，则执行以下代码块
        if (level == TENSOR_NO_OFFLOAD || level == TENSOR_OFFLOAD_FFN) {
            // 如果级别为 TENSOR_NO_OFFLOAD 或 TENSOR_OFFLOAD_FFN，则在 CPU 上创建张量
            return ml.create_tensor(ctx, name, ne, GGML_BACKEND_CPU);
        }
        // 为 GPU 张量仅分配元数据
        bool no_alloc = ctx->no_alloc;
        ggml_set_no_alloc(ctx, true);
        ggml_tensor * meta_tensor = ml.create_tensor(ctx, name, ne, GGML_BACKEND_CPU);
        ggml_set_no_alloc(ctx, no_alloc);
        alloc_queues[level].push_back(meta_tensor);
        return meta_tensor;
#else
        // 如果未定义 GGML_USE_CUBLAS，则在 CPU 上创建张量
        return ml.create_tensor(ctx, name, ne, GGML_BACKEND_CPU);
#endif
    }

    // 对于 GPU 张量，我们需要尽可能在 VRAM 中分配它们，并原地更新张量数据。如果超出了 VRAM 预算，我们将在 CPU 内存中分配张量。
    size_t flush() {
#if defined(GGML_USE_CUBLAS)
        // 遍历离线优先级
        for (int enum_i = TENSOR_OFFLOAD_ATTN; enum_i <= TENSOR_OFFLOAD_KV_CACHE; enum_i ++) {
            tensor_offloading_levels level = static_cast<tensor_offloading_levels>(enum_i);
            for (ggml_tensor * meta_tensor : alloc_queues[level]) {
                // 获取张量数据的大小
                size_t tensor_data_size = ggml_nbytes(meta_tensor);
                if (vram_allocated_bytes + tensor_data_size > vram_budget_bytes) {
                    // 如果 VRAM 分配的字节数加上张量数据大小超出了 VRAM 预算，则返回当前 VRAM 分配的字节数
                    return vram_allocated_bytes;
                }
                // 在 VRAM 中分配
                ggml_set_backend(meta_tensor, GGML_BACKEND_GPU);
                vram_allocated_bytes += tensor_data_size;
            }
        }
#endif
        return vram_allocated_bytes;
    }
};

static bool load_gpu_split_from_split_file(llama_model & model, std::string split_path, size_t vram_budget) {
    // 从分割文件中加载 GPU 分割
    llama_gpu_split_loader loader(split_path, true);
    return loader.check_vram_allocable(vram_budget) 
        && loader.apply_tensors_to_base_model(&model) == 0;
}
// 根据给定的模型加载 GPU 分割，并且限制内存预算
static bool llm_load_gpu_split_with_budget(llama_model_loader & ml, llama_model & model, size_t vram_allocatable_bytes, bool no_cache) {
    // 获取模型文件路径
    const char * model_path = ml.file.fname.c_str();
    // 生成缓存分割路径
    std::string cached_split_path = std::string(model_path) + ".generated.gpuidx";
    // 获取模型的基本目录
    const char * model_basedir = dirname(const_cast<char *>(model_path));

    // 从先前生成的缓存中加载 GPU 分割
    if (access(cached_split_path.c_str(), F_OK) == 0 && !no_cache) {
        // 如果缓存存在且不禁用缓存，则加载 GPU 分割
        if (load_gpu_split_from_split_file(model, cached_split_path, vram_allocatable_bytes)) {
            return true;
        }
        // 如果加载失败，则记录错误信息
        LLAMA_LOG_ERROR("%s: error: failed to apply previously generated gpu split from '%s'\n", __func__, cached_split_path.c_str());
    }

    // 生成 GPU 分割
    std::string activation_path = std::string(model_basedir) + "/activation";
    // 检查激活文件是否存在
    if (access(activation_path.c_str(), F_OK) != 0) {
        // 如果激活文件不存在，则记录错误信息并返回 false
        LLAMA_LOG_ERROR("%s: error: activation files under '%s' not found\n", __func__, activation_path.c_str());
        return false;
    }

    // 计算求解器参数
    ggml_tensor * ffn_up = model.layers[0].ffn_up;
    ggml_tensor * ffn_gate = model.layers[0].ffn_gate;
    int slice_size = ffn_up->ne[1] * ggml_type_size(ffn_up->type) / ggml_blck_size(ffn_up->type);
    // 对于具有 FFN gate 的模型架构，门也被切片，否则只有上下矩阵被切片
    int vram_bytes_per_slice = slice_size * (ffn_gate ? 4.5 : 2); // TODO: why 4.5, not 3?
    int neuron_cap = floor((double)vram_allocatable_bytes / vram_bytes_per_slice) * 4;

    // 调用 powerinfer Python 模块生成指定 VRAM 大小的 GPU 分割
    LLAMA_LOG_INFO("invoking powerinfer Python module to generate gpu split for %.2f MiB of VRAM\n", vram_allocatable_bytes / 1024.0 / 1024.0);

    // 创建命令流
    std::stringstream command_ss;
    # 拼接命令字符串，调用 python3 执行 powerinfer 模块
    command_ss << "python3 -m powerinfer"
               << " --activation " << activation_path
               << " --layer " << model.hparams.n_layer
               << " --neuron " << ffn_up->ne[1]
               << " --capacity " << neuron_cap
               << " --vram-capacity " << vram_allocatable_bytes
               << " --output " << cached_split_path;
    # 如果执行命令返回非零值，或者缓存文件不存在，则输出错误信息并返回 false
    if (system(command_ss.str().c_str()) != 0 || access(cached_split_path.c_str(), F_OK) != 0) {
        LLAMA_LOG_ERROR("%s: error: failed to generate gpu split\n", __func__);
        return false;
    }
    # 载入 GPU 分割文件，并返回结果
    return load_gpu_split_from_split_file(model, cached_split_path, vram_allocatable_bytes);
}

static void llm_load_gpu_split(llama_model_loader & ml, llama_model & model, size_t vram_budget_bytes, bool no_cache, bool no_offload) {
#if defined(GGML_USE_CUBLAS)
    // 如果 VRAM 预算大于等于 512MB 并且不禁用 GPU 加速
    if (vram_budget_bytes >= 512ull * 1024 * 1024 && !no_offload) {
        // 留下 512MB 作为安全边界
        vram_budget_bytes -= 512ull * 1024 * 1024;
        // 如果无法使用给定的 VRAM 预算加载 GPU 分割模型
        if (!llm_load_gpu_split_with_budget(ml, model, vram_budget_bytes, no_cache)) {
            // 输出错误信息
            LLAMA_LOG_ERROR("%s: error: failed to generate gpu split, an empty one will be used\n", __func__);
        }
    }
#endif
    // 将 GPU 索引和分割的 FFN 应用到 GPU
    size_t ffn_offloaded_bytes = llama_model_offload_ffn_split(&model);
    // 输出信息，显示已经将多少 MiB 的 FFN 权重数据加载到 GPU
    LLAMA_LOG_INFO("%s: offloaded %.2f MiB of FFN weights to GPU\n", __func__, ffn_offloaded_bytes / 1024.0 / 1024.0);
}

static void llm_load_sparse_model_tensors(
        llama_model_loader & ml,
        llama_model & model,
        int main_gpu,
        long long vram_budget_bytes,
        bool reset_gpu_index,
        bool disable_ffn_split,
        bool use_mlock,
        llama_progress_callback progress_callback,
        void * progress_callback_user_data) {
    model.t_start_us = ggml_time_us();
    auto & ctx     = model.ctx;
    auto & hparams = model.hparams;

    size_t ctx_size;
    size_t mmapped_size;
    // 计算上下文和内存映射的大小
    ml.calc_sizes(ctx_size, mmapped_size);
    // 输出信息，显示 ggml 上下文的大小
    LLAMA_LOG_INFO("%s: ggml ctx size = %7.2f MB\n", __func__, ctx_size/1024.0/1024.0);

    // 创建 ggml 上下文
    {
        // 调整缓冲区大小
        model.buf.resize(ctx_size);
        // 如果使用 mlock，则初始化缓冲区
        if (use_mlock) {
            model.mlock_buf.init   (model.buf.data);
            model.mlock_buf.grow_to(model.buf.size);
        }

        // 初始化参数
        struct ggml_init_params params = {
            /*.mem_size   =*/ model.buf.size,
            /*.mem_buffer =*/ model.buf.data,
            /*.no_alloc   =*/ ml.use_mmap,
        };

        // 初始化 ggml 上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，则抛出运行时错误
        if (!model.ctx) {
            throw std::runtime_error(format("ggml_init() failed"));
        }
    }

    (void) main_gpu;
    # 定义枚举类型变量 llama_backend_offload，并赋值为 GGML_BACKEND_CPU
    enum ggml_backend_type llama_backend_offload = GGML_BACKEND_CPU;
    # 定义枚举类型变量 llama_backend_offload_split，并赋值为 GGML_BACKEND_CPU
    enum ggml_backend_type llama_backend_offload_split = GGML_BACKEND_CPU;
    # 定义变量 vram_capacity，并赋值为 0
    size_t vram_capacity = 0;
#ifdef GGML_USE_CUBLAS
    // 如果使用了 cuBLAS 库
    if (ggml_cublas_loaded()) {
        // 输出信息，使用 GGML_CUDA_NAME 进行 GPU 加速
        LLAMA_LOG_INFO("%s: using " GGML_CUDA_NAME " for GPU acceleration\n", __func__);
        // 设置主 GPU 设备
        ggml_cuda_set_main_device(main_gpu);

        // 设置 LLAMA 后端为 GPU
        llama_backend_offload = GGML_BACKEND_GPU;
        llama_backend_offload_split = GGML_BACKEND_GPU_SPLIT;
    }
#elif defined(GGML_USE_CLBLAST)
        // 输出信息，使用 OpenCL 进行 GPU 加速
        LLAMA_LOG_INFO("%s: using OpenCL for GPU acceleration\n", __func__);
        // 设置 LLAMA 后端为 GPU
        llama_backend_offload = GGML_BACKEND_GPU;
        llama_backend_offload_split = GGML_BACKEND_GPU;
#endif

#if defined(GGML_USE_CUBLAS)
    if (vram_budget_bytes < 0) {
        // 如果 VRAM 预算小于 0，则将其设置为剩余的 VRAM 容量
        vram_capacity = ggml_cuda_get_free_memory(main_gpu);
    } else {
        // 否则，将 VRAM 预算和当前 VRAM 可用内存的较小值作为 VRAM 容量
        vram_capacity = std::min(vram_budget_bytes, (long long) ggml_cuda_get_free_memory(main_gpu));
    }
#endif

    // 创建缓冲张量分配器
    buffered_tensor_allocator alloc(ml, ctx, vram_capacity);
    // 创建张量的 lambda 函数
    auto create_tensor = [&alloc] (
        const std::pair<std::string, tensor_offloading_levels> & tn, 
        const std::vector<int64_t> & ne) -> ggml_tensor * {
        return alloc.buffered_alloc(tn.first, tn.second, ne);
    };

    }

    // 刷新分配的 VRAM 内存
    size_t vram_allocated_bytes = alloc.flush();
    // 断言，已分配的 VRAM 内存不超过 VRAM 容量
    GGML_ASSERT_DBG(vram_allocated_bytes <= vram_capacity, "vram_allocated_bytes=%ld, vram_capacity=%ld", vram_allocated_bytes, vram_capacity);
    // 完成获取张量
    ml.done_getting_tensors();

    // 打印内存需求
    {
        // 运行推断所需的总内存
        size_t mem_required =
            ctx_size +
            mmapped_size - vram_allocated_bytes; // 在 VRAM 中的权重不在内存中

        // 输出信息，内存需求
        LLAMA_LOG_INFO("%s: mem required  = %7.2f MB\n", __func__, mem_required / 1024.0 / 1024.0);

#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST)
        // 输出信息，使用的 VRAM
        LLAMA_LOG_INFO("%s: VRAM used: %.2f MB\n", __func__, vram_allocated_bytes / 1024.0 / 1024.0);
#endif
    }

    // 填充 `tensors_by_name`
    // 遍历 ml 中的张量数量
    for (int i = 0; i < ml.n_tensors; ++i) {
        // 获取当前张量的指针
        struct ggml_tensor * cur = ggml_get_tensor(ctx, ml.get_tensor_name(i));
        // 将张量名称和指针添加到 model.tensors_by_name 中
        model.tensors_by_name.emplace_back(ggml_get_name(cur), cur);
    }

    // 加载所有数据到内存中
    ml.load_all_data(ctx, progress_callback, progress_callback_user_data, use_mlock ? &model.mlock_mmap : NULL);

    // 如果存在进度回调函数，则调用进度回调函数
    if (progress_callback) {
        progress_callback(1.0f, progress_callback_user_data);
    }

    // 将 ml.mapping 移动到 model.mapping 中
    model.mapping = std::move(ml.mapping);

    // 如果可能的话，将 FFN 段卸载到 GPU
    llm_load_gpu_split(ml, model, vram_capacity - vram_allocated_bytes, reset_gpu_index, disable_ffn_split);

    // 加载时间将在第一次评估后重新计算，因此考虑由 mmap() 推迟的页面错误
    model.t_load_us = ggml_time_us() - model.t_start_us;

    // TODO: 根据卸载结果，按类别确定 GPU 层数
    model.n_gpu_layers = -1;
}

static void llm_load_tensors(
        llama_model_loader & ml,
        llama_model & model,
        int n_gpu_layers,
        int main_gpu,
        const float * tensor_split,
        bool use_mlock,
        llama_progress_callback progress_callback,
        void * progress_callback_user_data) {
    model.t_start_us = ggml_time_us();

    auto & ctx     = model.ctx;  // 获取模型的上下文
    auto & hparams = model.hparams;  // 获取模型的超参数

    model.n_gpu_layers = n_gpu_layers;  // 设置模型的 GPU 层数

    size_t ctx_size;  // 定义上下文大小变量
    size_t mmapped_size;  // 定义内存映射大小变量

    ml.calc_sizes(ctx_size, mmapped_size);  // 计算上下文大小和内存映射大小

    LLAMA_LOG_INFO("%s: ggml ctx size = %7.2f MB\n", __func__, ctx_size/1024.0/1024.0);  // 记录上下文大小日志信息

    // 创建 ggml 上下文
    {
        model.buf.resize(ctx_size);  // 调整模型缓冲区大小
        if (use_mlock) {
            model.mlock_buf.init   (model.buf.data);  // 初始化内存锁定缓冲区
            model.mlock_buf.grow_to(model.buf.size);  // 扩展内存锁定缓冲区大小
        }

        struct ggml_init_params params = {
            /*.mem_size   =*/ model.buf.size,  // 设置内存大小
            /*.mem_buffer =*/ model.buf.data,  // 设置内存缓冲区
            /*.no_alloc   =*/ ml.use_mmap,  // 设置是否使用内存映射
        };

        model.ctx = ggml_init(params);  // 初始化 ggml 上下文
        if (!model.ctx) {
            throw std::runtime_error(format("ggml_init() failed"));  // 如果初始化失败，抛出运行时错误
        }
    }

    (void) main_gpu;  // 忽略主 GPU

    enum ggml_backend_type llama_backend_offload = GGML_BACKEND_CPU;  // 设置 LLAMA 后端卸载类型为 CPU
    enum ggml_backend_type llama_backend_offload_split = GGML_BACKEND_CPU;  // 设置 LLAMA 分割后端卸载类型为 CPU

#ifdef GGML_USE_CUBLAS
    if (ggml_cublas_loaded()) {  // 如果加载了 ggml_cublas
        LLAMA_LOG_INFO("%s: using " GGML_CUDA_NAME " for GPU acceleration\n", __func__);  // 记录使用 GGML_CUDA_NAME 进行 GPU 加速的日志信息
        ggml_cuda_set_main_device(main_gpu);  // 设置主 GPU 设备

        llama_backend_offload = GGML_BACKEND_GPU;  // 设置 LLAMA 后端卸载类型为 GPU
        llama_backend_offload_split = GGML_BACKEND_GPU_SPLIT;  // 设置 LLAMA 分割后端卸载类型为 GPU 分割
    }
#elif defined(GGML_USE_CLBLAST)
        LLAMA_LOG_INFO("%s: using OpenCL for GPU acceleration\n", __func__);  // 记录使用 OpenCL 进行 GPU 加速的日志信息
        llama_backend_offload = GGML_BACKEND_GPU;  // 设置 LLAMA 后端卸载类型为 GPU
        llama_backend_offload_split = GGML_BACKEND_GPU;  // 设置 LLAMA 分割后端卸载类型为 GPU
#endif

    // 准备权重的内存
    size_t vram_weights = 0;  // 定义权重内存大小变量
        // 定义并初始化变量，表示嵌入大小、GQA嵌入大小、层数和词汇量
        const int64_t n_embd     = hparams.n_embd;
        const int64_t n_embd_gqa = hparams.n_embd_gqa();
        const int64_t n_layer    = hparams.n_layer;
        const int64_t n_vocab    = hparams.n_vocab;

        // 根据模型架构选择不同的操作
        const auto tn = LLM_TN(model.arch);
        switch (model.arch) {
            // 对于LLAMA和REFACT架构
            case LLM_ARCH_LLAMA:
            case LLM_ARCH_REFACT:
                {
                    // 创建并初始化token嵌入张量
                    model.tok_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab}, GGML_BACKEND_CPU);

                    // 输出
                    {
                        // 定义并初始化变量，表示规范化和输出的后端类型
                        ggml_backend_type backend_norm;
                        ggml_backend_type backend_output;

                        // 如果GPU层数大于层数
                        if (n_gpu_layers > int(n_layer)) {
                            // 规范化本身不影响性能，但在VRAM上保留它可以减少数据复制
                            // 但在Windows上，除非所有内容都在GPU上，否则这对性能有害
#ifndef _WIN32
                            // 如果不是在 Windows 平台上，则将 backend_norm 设置为 llama_backend_offload
                            backend_norm = llama_backend_offload;
#else
                            // 如果是在 Windows 平台上
                            // 如果 n_gpu_layers 小于等于 n_layer + 2，则将 backend_norm 设置为 GGML_BACKEND_CPU，否则设置为 llama_backend_offload
                            backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#ifndef _WIN32
                            // 如果不是在 Windows 平台上，则将 backend_norm 设置为 llama_backend_offload
                            backend_norm = llama_backend_offload;
#else
                            // 如果是在 Windows 平台上
                            // 如果 n_gpu_layers 小于等于 n_layer + 2，则将 backend_norm 设置为 GGML_BACKEND_CPU，否则设置为 llama_backend_offload
                            backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#ifndef _WIN32
                            // 如果不是在 Windows 平台上，则将 backend_norm 设置为 llama_backend_offload
                            backend_norm = llama_backend_offload;
#else
                            // 如果是在 Windows 平台上
                            // 如果 n_gpu_layers 小于等于 n_layer + 2，则将 backend_norm 设置为 GGML_BACKEND_CPU，否则设置为 llama_backend_offload
                            backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#ifndef _WIN32
                            // 如果不是在 Windows 平台上，则将 backend_norm 设置为 llama_backend_offload
                            backend_norm = llama_backend_offload;
#else
                            // 如果是在 Windows 平台上
                            // 如果 n_gpu_layers 小于等于 n_layer + 2，则将 backend_norm 设置为 GGML_BACKEND_CPU，否则设置为 llama_backend_offload
                            backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#ifdef GGML_USE_CUBLAS
                            // 如果使用了 CUBLAS
                            if (n_gpu_layers > int(n_layer + 1)) {
                                // 输出错误信息
                                LLAMA_LOG_ERROR("%s: CUDA backend missing Persimmon CUDA ops, can offload at most %ld layers. See: https://github.com/ggerganov/llama.cpp/issues/4038\n",
                                    __func__, n_layer + 1);
                                // 抛出运行时错误
                                throw std::runtime_error("Persimmon CUDA offload failed");
                            }
#endif
                            // norm 对性能并不重要，但将其保留在 VRAM 中可以减少数据复制
                            // 在 Windows 平台上，除非所有内容都在 GPU 上，否则这样做会有害
#ifndef _WIN32
                            // 如果不是在 Windows 平台上，则将 backend_norm 设置为 llama_backend_offload
                            backend_norm = llama_backend_offload;
#else
                            // 如果是在 Windows 平台上
                            // 如果 n_gpu_layers 小于等于 n_layer + 2，则将 backend_norm 设置为 GGML_BACKEND_CPU，否则设置为 llama_backend_offload
                            backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#ifndef _WIN32
                            // 如果不是在 Windows 平台上，则将 backend_norm 设置为 llama_backend_offload
                            backend_norm = llama_backend_offload;
#else
                            // 如果是在 Windows 平台上
                            // 如果 n_gpu_layers 小于等于 n_layer + 2，则将 backend_norm 设置为 GGML_BACKEND_CPU，否则设置为 llama_backend_offload;
#ifndef _WIN32
                            // 如果不是 Windows 系统，使用 llama_backend_offload 作为后端规范
                            backend_norm = llama_backend_offload;
#else
                            // 如果是 Windows 系统，根据条件判断选择后端规范
                            backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#ifndef _WIN32
                            // 如果不是 Windows 系统，使用 llama_backend_offload 作为后端规范
                            backend_norm = llama_backend_offload;
#else
                            // 如果是 Windows 系统，根据条件判断选择后端规范
                            backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
    }

    // 完成获取张量
    ml.done_getting_tensors();

    // 打印内存需求
    {
        // 运行推断所需的总内存
        size_t mem_required =
            ctx_size +
            mmapped_size - vram_weights; // VRAM 中的权重不在内存中

        LLAMA_LOG_INFO("%s: mem required  = %7.2f MB\n", __func__, mem_required / 1024.0 / 1024.0);

#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST)
        const int n_gpu = std::min(n_gpu_layers, int(hparams.n_layer));

        LLAMA_LOG_INFO("%s: offloading %d repeating layers to GPU\n", __func__, n_gpu);
        if (n_gpu_layers > (int) hparams.n_layer) {
            LLAMA_LOG_INFO("%s: offloading non-repeating layers to GPU\n", __func__);
        }

#ifdef GGML_USE_CUBLAS
        const int max_backend_supported_layers = hparams.n_layer + 3;
        const int max_offloadable_layers       = hparams.n_layer + 3;
#elif GGML_USE_CLBLAST
        const int max_backend_supported_layers = hparams.n_layer + 1;
        const int max_offloadable_layers       = hparams.n_layer + 1;
#endif // GGML_USE_CUBLAS

        LLAMA_LOG_INFO("%s: offloaded %d/%d layers to GPU\n", __func__, std::min(n_gpu_layers, max_offloadable_layers), max_backend_supported_layers);
        LLAMA_LOG_INFO("%s: VRAM used: %.2f MB\n", __func__, vram_weights / 1024.0 / 1024.0);
#else
        (void) n_gpu_layers;
#endif // defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST)
    }

    // 填充 `tensors_by_name`
    # 遍历从 0 到 ml.n_tensors 的整数
    for (int i = 0; i < ml.n_tensors; ++i) {
        # 获取指向 ggml_tensor 结构体的指针 cur，通过 ggml_get_tensor 函数从上下文 ctx 中获取第 i 个张量的指针
        struct ggml_tensor * cur = ggml_get_tensor(ctx, ml.get_tensor_name(i));
        # 将张量的名称和指针组成的 pair 放入 model.tensors_by_name 中
        model.tensors_by_name.emplace_back(ggml_get_name(cur), cur);
    }

    # 忽略 tensor_split 的值
    (void) tensor_split;
#ifdef GGML_USE_CUBLAS
    {
        // 如果定义了 GGML_USE_CUBLAS，则设置张量分割
        ggml_cuda_set_tensor_split(tensor_split);
    }
#endif

    // 加载所有数据到上下文中，使用进度回调函数和用户数据，如果使用内存锁定，则传入内存锁定指针
    ml.load_all_data(ctx, progress_callback, progress_callback_user_data, use_mlock ? &model.mlock_mmap : NULL);

    // 如果存在进度回调函数，则调用进度回调函数，传入进度为1.0和用户数据
    if (progress_callback) {
        progress_callback(1.0f, progress_callback_user_data);
    }

    // 将 ml.mapping 移动到 model.mapping 中
    model.mapping = std::move(ml.mapping);

    // 加载时间将在第一次评估后重新计算，因此考虑由 mmap() 推迟的页面错误
    model.t_load_us = ggml_time_us() - model.t_start_us;
}

// 加载 llama 模型的函数，传入文件名、模型和参数
static bool llama_model_load(const std::string & fname, llama_model & model, const llama_model_params & params) {
    // 尝试加载 LLAMA 模型
    try {
        // 使用给定的文件名和参数创建 llama_model_loader 对象
        llama_model_loader ml(fname, params.use_mmap);

        // 如果模型的稀疏导数为 GGML_SPARSE_INFERENCE，则记录日志信息
        if (ml.sparse_deriv == GGML_SPARSE_INFERENCE) {
            LLAMA_LOG_INFO("%s: PowerInfer model loaded. Sparse inference will be used.\n", __func__);
        }

        // 设置模型的超参数中的 vocab_only 属性
        model.hparams.vocab_only = params.vocab_only;
        // 设置模型的稀疏导数属性
        model.sparse_deriv = ml.sparse_deriv;

        // 加载模型的架构
        llm_load_arch   (ml, model);
        // 加载模型的超参数
        llm_load_hparams(ml, model);
        // 加载模型的词汇表
        llm_load_vocab  (ml, model);

        // 打印模型的元信息
        llm_load_print_meta(ml, model);

        // 如果模型的词汇表大小与 id_to_token 映射的大小不匹配，则抛出运行时错误
        if (model.hparams.n_vocab != model.vocab.id_to_token.size()) {
            throw std::runtime_error("vocab size mismatch");
        }

        // 如果参数中指定了 vocab_only，则记录日志信息并返回 true
        if (params.vocab_only) {
            LLAMA_LOG_INFO("%s: vocab only - skipping tensors\n", __func__);
            return true;
        }

        // 如果模型使用稀疏推断
        if (llama_use_sparse_inference(&model)) {
            // 如果参数中指定了 n_gpu_layers，则记录警告信息并返回 false
            if (params.n_gpu_layers > 0) {
                LLAMA_LOG_WARN("%s: sparse inference ignores n_gpu_layers, you can use --vram-budget option instead\n", __func__);
                return false;
            }
            // 计算 VRAM 预算的字节数，并记录日志信息
            double vram_budget_bytes = params.vram_budget_gb * 1024.0 * 1024.0 * 1024.0;
            LLAMA_LOG_INFO("%s: sparse inference - vram budget = %.2f GB\n", __func__, vram_budget_bytes / 1024.0 / 1024.0 / 1024.0);
            // 加载稀疏模型张量
            llm_load_sparse_model_tensors(
                ml, model, params.main_gpu, vram_budget_bytes, params.reset_gpu_index, params.disable_gpu_index,
                params.use_mlock, params.progress_callback, params.progress_callback_user_data
            );
        } else {
            // 加载模型张量
            llm_load_tensors(
                ml, model, params.n_gpu_layers, params.main_gpu, params.tensor_split, params.use_mlock,
                params.progress_callback, params.progress_callback_user_data
            );
        }

    } catch (const std::exception & err) {
        // 捕获异常并记录错误日志信息，然后返回 false
        LLAMA_LOG_ERROR("error loading model: %s\n", err.what());
        return false;
    }

    // 加载模型成功，返回 true
    return true;
}

//
// llm_build
//

// 定义回调函数类型 llm_build_cb，用于处理 ggml_tensor 结构体
using llm_build_cb = std::function<void(struct ggml_tensor * cur, const char * name, int nl)>;

// 定义枚举类型 llm_rope_type，表示不同的绳索类型
enum llm_rope_type {
    LLM_ROPE,
    LLM_ROPE_NEOX,
    LLM_ROPE_GLM,
};

// 定义枚举类型 llm_ffn_op_type，表示不同的前馈神经网络操作类型
enum llm_ffn_op_type {
    LLM_FFN_SILU,
    LLM_FFN_GELU,
    LLM_FFN_RELU,
    LLM_FFN_RELU_SQR,
};

// 定义枚举类型 llm_ffn_gate_type，表示不同的前馈神经网络门类型
enum llm_ffn_gate_type {
    LLM_FFN_SEQ,
    LLM_FFN_PAR, // ffn_gate is parallel to ffn_up
};

// 定义枚举类型 llm_norm_type，表示不同的归一化类型
enum llm_norm_type {
    LLM_NORM,
    LLM_NORM_RMS,
};

// 定义静态函数 llm_build_inp_embd，用于构建输入嵌入
static struct ggml_tensor * llm_build_inp_embd(
        struct ggml_context * ctx,
        const llama_hparams & hparams,
        const llama_batch & batch,
        struct ggml_tensor * tok_embd,
        const llm_build_cb & cb) {
    const int64_t n_embd = hparams.n_embd;

    struct ggml_tensor * inpL;

    // 如果批处理中存在标记，则创建输入标记张量，并调用回调函数处理
    if (batch.token) {
        struct ggml_tensor * inp_tokens = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, batch.n_tokens);
        cb(inp_tokens, "inp_tokens", -1);

        inpL = ggml_get_rows(ctx, tok_embd, inp_tokens);
    } else {
        // 如果不存在标记，并且使用了 MPI，则抛出异常
#ifdef GGML_USE_MPI
        GGML_ASSERT(false && "not implemented");
#endif

        // 否则创建二维浮点数张量
        inpL = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, batch.n_tokens);
    }

    return inpL;
}

// 定义静态函数 llm_build_k_shift，用于构建 k-shift 操作
// Persimmon: n_rot = n_embd_head/2
// Other:     n_rot = n_embd_head
static void llm_build_k_shift(
      struct ggml_context * ctx,
      const llama_hparams & hparams,
      const llama_cparams & cparams,
      const llama_kv_cache & kv,
      struct ggml_cgraph * graph,
      llm_rope_type   type,
      int64_t   n_ctx,
      int64_t   n_rot,
      float     freq_base,
      float     freq_scale,
      const llm_build_cb & cb) {
    const int64_t n_layer     = hparams.n_layer;
    const int64_t n_head_kv   = hparams.n_head_kv;
    const int64_t n_embd_gqa  = hparams.n_embd_gqa();
    const int64_t n_embd_head = hparams.n_embd_head();
    const int32_t n_orig_ctx  = cparams.n_yarn_orig_ctx;
    const float   ext_factor  = cparams.yarn_ext_factor;
}
    # 定义注意力因子为参数中的纱线注意力因子
    const float   attn_factor = cparams.yarn_attn_factor;
    # 定义快速学习率为参数中的纱线快速学习率
    const float   beta_fast   = cparams.yarn_beta_fast;
    # 定义慢速学习率为参数中的纱线慢速学习率
    const float   beta_slow   = cparams.yarn_beta_slow;

    # 断言 n_embd_head 可以整除 n_rot
    GGML_ASSERT(n_embd_head % n_rot == 0);

    # 创建一个长度为 n_ctx 的整型张量 K_shift
    struct ggml_tensor * K_shift = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, n_ctx);
    # 调试信息，输出张量 K_shift 的信息
    cb(K_shift, "K_shift", -1);

    # 初始化绳索类型为 0
    int rope_type = 0;

    # 根据不同的类型设置不同的绳索类型
    switch (type) {
        case LLM_ROPE:      rope_type = 0; break;
        case LLM_ROPE_NEOX: rope_type = 2; break;
        case LLM_ROPE_GLM:  rope_type = 4; break;
    }

    # 遍历每一层
    for (int il = 0; il < n_layer; ++il) {
        # 创建一个临时张量，通过 ggml_rope_custom_inplace 函数进行绳索操作
        struct ggml_tensor * tmp =
            ggml_rope_custom_inplace(ctx,
                    # 从 kv.k 中创建一个 3D 视图
                    ggml_view_3d(ctx, kv.k,
                        n_rot, n_head_kv, n_ctx,
                        ggml_element_size(kv.k)*n_embd_head,
                        ggml_element_size(kv.k)*n_embd_gqa,
                        ggml_element_size(kv.k)*n_embd_gqa*n_ctx*il),
                    K_shift, n_rot, rope_type, 0, n_orig_ctx, freq_base, freq_scale,
                    ext_factor, attn_factor, beta_fast, beta_slow);
        # 调试信息，输出临时张量的信息
        cb(tmp, "K_shifted", il);
        # 在图中构建前向扩展
        ggml_build_forward_expand(graph, tmp);
    }
// 构建键值存储，返回键和值的张量指针对
static std::pair<ggml_tensor*, ggml_tensor*> llm_build_kv_store(
        struct ggml_context * ctx,
        const llama_hparams & hparams,
        const llama_kv_cache & kv,
        struct ggml_cgraph * graph,
        struct ggml_tensor * k_cur,
        struct ggml_tensor * v_cur,
        int64_t   n_ctx,
        int32_t   n_tokens,
        int32_t   kv_head,
        const llm_build_cb & cb,
        int64_t   il) {
    const int64_t n_embd_gqa = hparams.n_embd_gqa();

    // 计算转置的 [n_tokens, n_embd] V 矩阵
    struct ggml_tensor * v_cur_t = ggml_transpose(ctx, ggml_reshape_2d(ctx, v_cur, n_embd_gqa, n_tokens));
    //struct ggml_tensor * v_cur_t = ggml_transpose(ctx, v_cur); // TODO: reshape above is likely not needed
    cb(v_cur_t, "v_cur_t", il);

    // 创建 K 缓存的视图
    struct ggml_tensor * k_cache_view = ggml_view_1d(ctx, kv.k, n_tokens*n_embd_gqa,
            (ggml_element_size(kv.k)*n_embd_gqa)*(il*n_ctx + kv_head));
    cb(k_cache_view, "k_cache_view", il);

    // 创建 V 缓存的视图
    struct ggml_tensor * v_cache_view = ggml_view_2d(ctx, kv.v, n_tokens, n_embd_gqa,
            (   n_ctx)*ggml_element_size(kv.v),
            (il*n_ctx)*ggml_element_size(kv.v)*n_embd_gqa + kv_head*ggml_element_size(kv.v));
    cb(v_cache_view, "v_cache_view", il);

    // 重要：在 KV 缓存中存储 RoPE 版本的 K！
    ggml_tensor * k_cpy = ggml_cpy(ctx, k_cur,   k_cache_view);
    ggml_tensor * v_cpy = ggml_cpy(ctx, v_cur_t, v_cache_view);
    //ggml_build_forward_expand(graph, ggml_cpy(ctx, k_cur,   k_cache_view));
    //ggml_build_forward_expand(graph, ggml_cpy(ctx, v_cur_t, v_cache_view));

    return {k_cpy, v_cpy};
}
# 构建归一化层
static struct ggml_tensor * llm_build_norm(
        struct ggml_context * ctx,
         struct ggml_tensor * cur,
        const llama_hparams & hparams,
         struct ggml_tensor * mw,
         struct ggml_tensor * mb,
              llm_norm_type   type,
         const llm_build_cb & cb,
                        int   il) {
    # 根据不同的归一化类型进行处理
    switch (type) {
        case LLM_NORM:     cur = ggml_norm    (ctx, cur, hparams.f_norm_eps);     break;
        case LLM_NORM_RMS: cur = ggml_rms_norm(ctx, cur, hparams.f_norm_rms_eps); break;
    }

    # 如果存在权重或偏置项，则调用回调函数
    if (mw || mb) {
        cb(cur, "norm", il);
    }

    # 如果存在权重项，则进行乘法操作，并调用回调函数
    if (mw) {
        cur = ggml_mul(ctx, cur, mw);
        # 如果同时存在偏置项，则调用回调函数
        if (mb) {
            cb(cur, "norm_w", il);
        }
    }

    # 如果存在偏置项，则进行加法操作
    if (mb) {
        cur = ggml_add(ctx, cur, mb);
    }

    # 返回处理后的结果
    return cur;
}

# 构建前馈神经网络层
static struct ggml_tensor * llm_build_ffn(
        struct ggml_context * ctx,
         struct ggml_tensor * cur,
         struct ggml_tensor * up,
         struct ggml_tensor * up_b,
         struct ggml_tensor * gate,
         struct ggml_tensor * gate_b,
         struct ggml_tensor * down,
         struct ggml_tensor * down_b,
            llm_ffn_op_type   type_op,
          llm_ffn_gate_type   type_gate,
         const llm_build_cb & cb,
                        int   il) {
    # 对上一层和当前层进行矩阵乘法操作，并调用回调函数
    struct ggml_tensor * tmp = ggml_mul_mat(ctx, up, cur);
    cb(tmp, "ffn_up", il);

    # 如果存在上一层的偏置项，则进行加法操作，并调用回调函数
    if (up_b) {
        tmp = ggml_add(ctx, tmp, up_b);
        cb(tmp, "ffn_up_b", il);
    }

    # 如果存在门控张量，则根据不同的门控类型进行处理
    if (gate) {
        switch (type_gate) {
            case LLM_FFN_SEQ:
                {
                    # 对门控张量和临时张量进行矩阵乘法操作，并调用回调函数
                    cur = ggml_mul_mat(ctx, gate, tmp);
                    cb(cur, "ffn_gate", il);
                } break;
            case LLM_FFN_PAR:
                {
                    # 对门控张量和当前层进行矩阵乘法操作，并调用回调函数
                    cur = ggml_mul_mat(ctx, gate, cur);
                    cb(cur, "ffn_gate", il);
                } break;
        }

        # 如果存在门控张量的偏置项，则进行加法操作，并调用回调函数
        if (gate_b) {
            cur = ggml_add(ctx, cur, gate_b);
            cb(cur, "ffn_gate_b", il);
        }
    } else {
        // 如果条件不成立，将当前值设置为临时值
        cur = tmp;
    }

    switch (type_op) {
        case LLM_FFN_SILU:
            {
                // 对当前值进行 SILU 激活操作
                cur = ggml_silu(ctx, cur);
                // 调用回调函数，传递当前值、操作类型和索引值
                cb(cur, "ffn_silu", il);
            } break;
        case LLM_FFN_GELU:
            {
                // 对当前值进行 GELU 激活操作
                cur = ggml_gelu(ctx, cur);
                // 调用回调函数，传递当前值、操作类型和索引值
                cb(cur, "ffn_gelu", il);
            } break;
        case LLM_FFN_RELU:
            {
                // 对当前值进行 ReLU 激活操作
                cur = ggml_relu(ctx, cur);
                // 调用回调函数，传递当前值、操作类型和索引值
                cb(cur, "ffn_relu", il);
            } break;
        case LLM_FFN_RELU_SQR:
            {
                // 对当前值进行 ReLU 激活操作
                cur = ggml_relu(ctx, cur);
                // 调用回调函数，传递当前值、操作类型和索引值
                cb(cur, "ffn_relu", il);

                // 对当前值进行平方操作
                cur = ggml_sqr(ctx, cur);
                // 调用回调函数，传递当前值、操作类型和索引值
                cb(cur, "ffn_sqr(relu)", il);
            } break;
    }

    if (type_gate == LLM_FFN_PAR) {
        // 对当前值和临时值进行乘法操作
        cur = ggml_mul(ctx, cur, tmp);
        // 调用回调函数，传递当前值、操作类型和索引值
        cb(cur, "ffn_gate_par", il);
    }

    // 对当前值和下行值进行线性变换操作
    cur = ggml_axpy(ctx, down, cur, NULL, NULL);
    if (down_b) {
        // 如果存在下行偏置，调用回调函数，传递当前值和操作类型
        cb(cur, "ffn_down", il);
    }

    if (down_b) {
        // 如果存在下行偏置，将当前值和下行偏置相加
        cur = ggml_add(ctx, cur, down_b);
    }

    // 返回当前值
    return cur;
// 定义一个名为 llm_build_ffn_sparse 的静态函数，接受多个 ggml_tensor 类型的参数，并返回 ggml_tensor 类型的指针
static struct ggml_tensor * llm_build_ffn_sparse(
        struct ggml_context * ctx,
         struct ggml_tensor * cur,
         struct ggml_tensor * up,
         struct ggml_tensor * up_b,
         struct ggml_tensor * gate,
         struct ggml_tensor * gate_b,
         struct ggml_tensor * down,
         struct ggml_tensor * down_b,
         struct ggml_tensor * down_t,
         struct ggml_tensor * pre_w1,
         struct ggml_tensor * pre_w2,
         struct ggml_tensor * pred_inpl,
         struct ggml_tensor * gpu_index,
         struct ggml_tensor * gpu_bucket,
         struct ggml_tensor * gate_gpu,
         struct ggml_tensor * down_gpu,
         struct ggml_tensor * up_gpu,
            llm_ffn_op_type   type_op,
          llm_ffn_gate_type   type_gate,
         const llm_build_cb & cb,
                        int   il) {
    // TODO: no gpu slicing for now
    // GGML_ASSERT(gate_gpu == nullptr && down_gpu == nullptr && up_gpu == nullptr && gpu_bucket == nullptr);

    ggml_tensor *idx = nullptr; // 初始化指针 idx 为 nullptr
    ggml_tensor *idx_g = nullptr; // 初始化指针 idx_g 为 nullptr
    ggml_tensor *cur_c = nullptr; // 初始化指针 cur_c 为 nullptr
    ggml_tensor *third = nullptr; // 初始化指针 third 为 nullptr

    if (pred_inpl->backend != pre_w1->backend) { // 如果 pred_inpl 的后端不等于 pre_w1 的后端
        if (pre_w1->backend == GGML_BACKEND_CPU) { // 如果 pre_w1 的后端是 GGML_BACKEND_CPU
            pred_inpl = ggml_dup(ctx, pred_inpl); // 则复制 pred_inpl
        } else {
            // NOOP for now
        }
    }

    // 准备稀疏索引
    idx = ggml_mul_mat(ctx, pre_w1, pred_inpl); // 计算 pre_w1 与 pred_inpl 的矩阵乘法，结果赋给 idx
    cb(idx, "mlp_pre_w1", il); // 调用回调函数 cb，传递参数 idx, "mlp_pre_w1", il
    idx = ggml_relu(ctx, idx); // 对 idx 执行 ReLU 激活函数
    cb(idx, "relu_pre", il); // 调用回调函数 cb，传递参数 idx, "relu_pre", il
    idx = ggml_mul_mat(ctx, pre_w2, idx); // 计算 pre_w2 与 idx 的矩阵乘法，结果赋给 idx
    cb(idx, "mlp_pre_w2", il); // 调用回调函数 cb，传递参数 idx, "mlp_pre_w2", il

    // FFN up
    third = cur; // 将 cur 赋给 third
    struct ggml_tensor * tmp = ggml_mul_mat_idx(ctx, up, cur, idx, gpu_index); // 计算 up 与 cur、idx、gpu_index 的矩阵乘法，结果赋给 tmp
    cb(tmp, "ffn_up_sparse", il); // 调用回调函数 cb，传递参数 tmp, "ffn_up_sparse", il
#ifdef GGML_USE_CUBLAS
    struct ggml_tensor * tmp2 = ggml_mul_mat_special(ctx, up_gpu, cur, idx, gpu_bucket, up); // 使用特殊方法计算 up_gpu 与 cur、idx、gpu_bucket、up 的矩阵乘法，结果赋给 tmp2
    if (tmp2 != NULL) { // 如果 tmp2 不为空
        ggml_cuda_assign_buffers_no_alloc(tmp2); // 分配 CUDA 缓冲区给 tmp2
        cb(tmp2, "ffn_up_sparse_gpu", il); // 调用回调函数 cb，传递参数 tmp2, "ffn_up_sparse_gpu", il
    }
    # 调用 ggml_add 函数，传入参数 ctx, tmp, tmp2，并将返回值赋给 tmp
    tmp = ggml_add(ctx, tmp, tmp2);
#endif


    if (up_b) {
        // 如果存在 up_b，则将其添加到 tmp 中
        tmp = ggml_add(ctx, tmp, up_b);
        // 调用回调函数，传递 tmp 和 "ffn_up_b" 作为参数
        cb(tmp, "ffn_up_b", il);
    }

    if (gate) {
        // TODO: only support par for now
        // 断言 type_gate 等于 LLM_FFN_PAR
        GGML_ASSERT(type_gate == LLM_FFN_PAR);
        // 将 cur 赋值给 third，并将 gate 与 cur 相乘，得到新的 cur
        third = cur;
        cur = ggml_mul_mat_idx(ctx, gate, cur, idx, gpu_index);
        // 调用回调函数，传递 cur 和 "ffn_gate" 作为参数
        cb(cur, "ffn_gate", il);
#ifdef GGML_USE_CUBLAS
        // 如果使用了 CUBLAS，则进行特殊的矩阵乘法操作
        tmp2 = ggml_mul_mat_special(ctx, gate_gpu, third, idx, gpu_bucket, gate);
        if (tmp2 != NULL) {
            // 将 tmp2 分配给 CUDA 缓冲区，不进行分配
            ggml_cuda_assign_buffers_no_alloc(tmp2);
            // 调用回调函数，传递 tmp2 和 "ffn_up_sparse_gpu" 作为参数
            cb(tmp2, "ffn_up_sparse_gpu", il);
        }
        // 将 cur 与 tmp2 相加
        cur = ggml_add(ctx, cur, tmp2);
#endif

        if (gate_b) {
            // 如果存在 gate_b，则将其添加到 cur 中
            cur = ggml_add(ctx, cur, gate_b);
            // 调用回调函数，传递 cur 和 "ffn_gate_b" 作为参数
            cb(cur, "ffn_gate_b", il);
        }
    } else {
        // 如果不存在 gate，则将 cur 赋值为 tmp
        cur = tmp;
    }

    switch (type_op) {
        case LLM_FFN_RELU:
            {
                // 对 cur 进行 ReLU 激活函数操作
                cur = ggml_relu(ctx, cur);
                // 调用回调函数，传递 cur 和 "ffn_relu" 作为参数
                cb(cur, "ffn_relu", il);
            } break;
        default:
            // only support relu for now
            // 断言 type_op 等于 LLM_FFN_RELU
            GGML_ASSERT(type_op == LLM_FFN_RELU);
    }

    if (type_gate == LLM_FFN_PAR) {
        // 如果 type_gate 等于 LLM_FFN_PAR，则将 cur 与 tmp 相乘
        cur = ggml_mul(ctx, cur, tmp);
        // 调用回调函数，传递 cur 和 "ffn_gate_par" 作为参数
        cb(cur, "ffn_gate_par", il);
    }

    // 将 cur 赋值给 third
    third = cur;
#ifdef GGML_USE_CUBLAS
    // 使用 CUBLAS 进行 axpy 操作
    cur = ggml_axpy(ctx, down_gpu, cur, idx, gpu_bucket);
    if (cur != NULL) {
        // 将 cur 分配给 CUDA 缓冲区，不进行分配
        ggml_cuda_assign_buffers_no_alloc(cur);
        // 调用回调函数，传递 cur 和 "ffn_down" 作为参数
        cb(cur, "ffn_down", il);
    }
#endif
    // 对 third 和 down_t 进行 axpy 操作，结果赋值给 tmp
    tmp = ggml_axpy(ctx, down_t, third, idx, gpu_index);
    // 调用回调函数，传递 tmp 和 "ffn_down_gpu" 作为参数
    cb(tmp, "ffn_down_gpu", il);
#ifdef GGML_USE_CUBLAS
    // 如果使用了 CUBLAS，则将 cur 与 tmp 相加
    cur = ggml_add(ctx, cur, tmp);
#else
    // 否则将 tmp 赋值给 cur
    cur = tmp;
#endif

    if (down_b) {
        // 如果存在 down_b，则将其添加到 cur 中
        cur = ggml_add(ctx, cur, down_b);
        // 调用回调函数，传递 cur 和 "ffn_down" 作为参数
        cb(cur, "ffn_down", il);
    }

    // 返回最终的 cur
    return cur;
}


static ggml_tensor * k_cpy = nullptr;
static ggml_tensor * v_cpy = nullptr;

// if max_alibi_bias > 0 then apply ALiBi
// 构建 K、Q、V 矩阵
static struct ggml_tensor * llm_build_kqv(
        struct ggml_context * ctx,
        const llama_hparams & hparams,
        const llama_kv_cache & kv,
        struct ggml_tensor * wo,
        struct ggml_tensor * wo_b,
        struct ggml_tensor * q_cur,
        struct ggml_tensor * kq_scale,
        struct ggml_tensor * kq_mask,
        int64_t   n_ctx,
        int32_t   n_tokens,
        int32_t   n_kv,
        float     max_alibi_bias,
        const llm_build_cb & cb,
        int       il) {
    // 从 hparams 中获取参数
    const int64_t n_embd      = hparams.n_embd;
    const int64_t n_head      = hparams.n_head;
    const int64_t n_head_kv   = hparams.n_head_kv;
    const int64_t n_embd_head = hparams.n_embd_head();
    const int64_t n_embd_gqa  = hparams.n_embd_gqa();

    // 对当前的 Q 进行维度置换
    struct ggml_tensor * q = ggml_permute(ctx, q_cur, 0, 2, 1, 3);
    // 回调函数，传递 Q 张量和标识符
    cb(q, "q", il);

    // 从 kv 中获取 K 张量
    struct ggml_tensor * k =
        ggml_view_3d(ctx, kv.k,
                n_embd_head, n_kv, n_head_kv,
                ggml_element_size(kv.k)*n_embd_gqa,
                ggml_element_size(kv.k)*n_embd_head,
                ggml_element_size(kv.k)*n_embd_gqa*n_ctx*il);
    // 回调函数，传递 K 张量和标识符
    cb(k, "k", il);
    // 如果 k_cpy 不为空，则将 k 的第二个源设置为 k_cpy
    if (k_cpy != nullptr) {
        k->src[1] = k_cpy;
    }

    // 计算 KQ
    struct ggml_tensor * kq = ggml_mul_mat(ctx, k, q);
    // 回调函数，传递 KQ 张量和标识符
    cb(kq, "kq", il);

    // 对 KQ 进行缩放
    kq = ggml_scale(ctx, kq, kq_scale);
    // 回调函数，传递缩放后的 KQ 张量和标识符
    cb(kq, "kq_scaled", il);

    // 如果 max_alibi_bias 大于 0.0f
    if (max_alibi_bias > 0.0f) {
        // TODO: n_head or n_head_kv
        // TODO: K-shift is likely not working
        // TODO: change to ggml_add
        // 对 KQ 进行 alibi 操作
        kq = ggml_alibi(ctx, kq, /*n_past*/ 0, n_head, max_alibi_bias);
        // 回调函数，传递 alibi 后的 KQ 张量和标识符
        cb(kq, "kq_scaled_alibi", il);
    }

    // 对 KQ 进行加法操作，加上 KQ mask
    kq = ggml_add(ctx, kq, kq_mask);
    // 回调函数，传递加上 mask 后的 KQ 张量和标识符
    cb(kq, "kq_masked", il);

    // 对 KQ 进行 softmax 操作
    kq = ggml_soft_max(ctx, kq);
    // 回调函数，传递 softmax 后的 KQ 张量和标识符
    cb(kq, "kq_soft_max", il);

    // 分割缓存的 V 成 n_head 个头部
    # 创建一个指向 ggml_tensor 结构的指针 v，通过调用 ggml_view_3d 函数来初始化
    struct ggml_tensor * v =
        ggml_view_3d(ctx, kv.v,
                n_kv, n_embd_head, n_head_kv,
                ggml_element_size(kv.v)*n_ctx,
                ggml_element_size(kv.v)*n_ctx*n_embd_head,
                ggml_element_size(kv.v)*n_ctx*n_embd_gqa*il);
    # 调用回调函数 cb，传入 v 指针和字符串 "v"，用于记录日志
    cb(v, "v", il);
    # 如果 v_cpy 指针不为空，则将 v->src[1] 指向 v_cpy
    if (v_cpy != nullptr) {
        v->src[1] = v_cpy;
    }

    # 创建一个指向 ggml_tensor 结构的指针 kqv，通过调用 ggml_mul_mat 函数来初始化
    struct ggml_tensor * kqv = ggml_mul_mat(ctx, v, kq);
    # 调用回调函数 cb，传入 kqv 指针和字符串 "kqv"，用于记录日志
    cb(kqv, "kqv", il);

    # 创建一个指向 ggml_tensor 结构的指针 kqv_merged，通过调用 ggml_permute 函数来初始化
    struct ggml_tensor * kqv_merged = ggml_permute(ctx, kqv, 0, 2, 1, 3);
    # 调用回调函数 cb，传入 kqv_merged 指针和字符串 "kqv_merged"，用于记录日志
    cb(kqv_merged, "kqv_merged", il);

    # 创建一个指向 ggml_tensor 结构的指针 cur，通过调用 ggml_cont_2d 函数来初始化
    struct ggml_tensor * cur = ggml_cont_2d(ctx, kqv_merged, n_embd, n_tokens);
    # 调用回调函数 cb，传入 cur 指针和字符串 "kqv_merged_cont"，用于记录日志
    cb(cur, "kqv_merged_cont", il);

    # 将 cur 指向 ggml_mul_mat 函数的返回值，传入参数 wo 和 cur
    cur = ggml_mul_mat(ctx, wo, cur);
    # 如果 wo_b 为真，则调用回调函数 cb，传入 cur 指针和字符串 "kqv_wo"，用于记录日志
    if (wo_b) {
        cb(cur, "kqv_wo", il);
    }

    # 如果 wo_b 为真，则将 cur 指向 ggml_add 函数的返回值，传入参数 cur 和 wo_b
    if (wo_b) {
        cur = ggml_add(ctx, cur, wo_b);
    }

    # 返回指向 ggml_tensor 结构的指针 cur
    return cur;
// 定义一个名为 no_offload_cb 的常量函数指针，用于设置 ggml_tensor 的名称
const llm_build_cb no_offload_cb = [](struct ggml_tensor * cur, const char * name, int nl) {
    ggml_set_name(cur, name);
};

// 定义一个名为 llm_build_context 的结构体，包含了多个常量成员变量和一个函数指针成员变量
struct llm_build_context {
    // model, hparams, cparams, batch, kv_self 是 llama_model, llama_hparams, llama_cparams, llama_batch, llama_kv_cache 的常量引用
    const llama_model    & model;
    const llama_hparams  & hparams;
    const llama_cparams  & cparams;
    const llama_batch    & batch;
    const llama_kv_cache & kv_self;

    // n_embd, n_layer, n_ctx, n_head, n_head_kv, n_embd_head, n_embd_gqa 是整型常量
    const int64_t n_embd;
    const int64_t n_layer;
    const int64_t n_ctx;       // user-specified context size (can be different from n_ctx_train)
    const int64_t n_head;
    const int64_t n_head_kv;
    const int64_t n_embd_head;
    const int64_t n_embd_gqa;

    // freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow, norm_eps, norm_rms_eps 是浮点型常量
    const float freq_base;
    const float freq_scale;
    const float ext_factor;
    const float attn_factor;
    const float beta_fast;
    const float beta_slow;
    const float norm_eps;
    const float norm_rms_eps;

    // n_tokens, n_kv, kv_head, n_orig_ctx 是整型常量
    const int32_t n_tokens;
    const int32_t n_kv;     // size of KV cache to consider (n_kv <= n_ctx)
    const int32_t kv_head;  // index of where we store new KV data in the cache
    const int32_t n_orig_ctx;

    // do_rope_shift 是布尔型常量
    const bool do_rope_shift;

    // cb 是 llm_build_cb 类型的函数指针
    llm_build_cb cb;

    // buf_compute 是 llama_buffer 类型的引用
    llama_buffer & buf_compute;

    // ctx0 是 ggml_context 指针，默认初始化为 nullptr
    struct ggml_context * ctx0 = nullptr;

    // TODO: consider making the entire interface noexcept
    // llm_build_context 的构造函数，接受 llama_context 和 llama_batch 作为参数
    llm_build_context(
        llama_context  & lctx,
    const llama_batch  & batch,
    // 构造函数，初始化模型、超参数、上下文参数、批处理数据等成员变量
    const llm_build_cb & cb,
                  bool   worst_case) :
        model         (lctx.model),
        hparams       (model.hparams),
        cparams       (lctx.cparams),
        batch         (batch),
        kv_self       (lctx.kv_self),
        n_embd        (hparams.n_embd),
        n_layer       (hparams.n_layer),
        n_ctx         (cparams.n_ctx),
        n_head        (hparams.n_head),
        n_head_kv     (hparams.n_head_kv),
        n_embd_head   (hparams.n_embd_head()),
        n_embd_gqa    (hparams.n_embd_gqa()),
        freq_base     (cparams.rope_freq_base),
        freq_scale    (cparams.rope_freq_scale),
        ext_factor    (cparams.yarn_ext_factor),
        attn_factor   (cparams.yarn_attn_factor),
        beta_fast     (cparams.yarn_beta_fast),
        beta_slow     (cparams.yarn_beta_slow),
        norm_eps      (hparams.f_norm_eps),
        norm_rms_eps  (hparams.f_norm_rms_eps),
        n_tokens      (batch.n_tokens),
        n_kv          (worst_case ? n_ctx            : kv_self.n),
        kv_head       (worst_case ? n_ctx - n_tokens : kv_self.head),
        n_orig_ctx    (cparams.n_yarn_orig_ctx),
        do_rope_shift (worst_case || kv_self.has_shift),
        cb            (cb),
        buf_compute   (lctx.buf_compute) {
            GGML_ASSERT(!!kv_self.ctx);

            // 所有初始化应该在init()函数中完成
        }

    // 初始化函数，初始化ggml_init_params结构体并调用ggml_init函数
    void init() {
        struct ggml_init_params params = {
            /*.mem_size   =*/ buf_compute.size,
            /*.mem_buffer =*/ buf_compute.data,
            /*.no_alloc   =*/ true,
        };
        
        // 调用ggml_init函数
        ctx0 = ggml_init(params);
    }

    // 释放函数，释放ctx0指向的内存并将其置为空指针
    void free() {
        if (ctx0) {
            ggml_free(ctx0);
            ctx0 = nullptr;
        }
    }
// 结构体定义，用于表示离线计算任务的类型
enum llm_offload_func_e {
    OFFLOAD_FUNC_NOP,  // 空操作
    OFFLOAD_FUNC,  // 普通离线计算任务
    OFFLOAD_FUNC_KQ,  // KQ 离线计算任务
    OFFLOAD_FUNC_V,  // V 离线计算任务
    OFFLOAD_FUNC_NR,  // NR 离线计算任务
    OFFLOAD_FUNC_EMB,  // EMB 离线计算任务
    OFFLOAD_FUNC_OUT,  // OUT 离线计算任务
};

// TODO: 将在后端 v2 版本中移除
// 前缀树结构，用于存储离线计算任务的名称和类型
struct llm_offload_trie {
    struct node {
        ~node() {
            for (int i = 0; i < 256; ++i) {
                if (children[i]) {
                    delete children[i];
                }
            }
        }

        node * children[256] = { nullptr };  // 子节点数组，用于存储子节点指针
        llm_offload_func_e func = OFFLOAD_FUNC_NOP;  // 节点对应的离线计算任务类型，默认为 NOP
    };

    llm_offload_trie() {
        root = new node;  // 初始化根节点
    }

    llm_offload_trie(const std::unordered_map<const char *, llm_offload_func_e> & map) {
        root = new node;  // 初始化根节点

        for (const auto & kv : map) {
            add(kv.first, kv.second);  // 添加离线计算任务名称和类型到前缀树中
        }
    }

    ~llm_offload_trie() {
        delete root;  // 释放根节点及其所有子节点的内存
    }

    // 向前缀树中添加离线计算任务名称和类型
    void add(const char * name, llm_offload_func_e func) {
        node * cur = root;  // 从根节点开始

        for (int i = 0; ; ++i) {
            const uint8_t c = name[i];  // 获取名称中的字符

            if (!c) {
                break;  // 如果字符为空，则结束循环
            }

            if (!cur->children[c]) {
                cur->children[c] = new node;  // 如果当前字符对应的子节点不存在，则创建新的子节点
            }

            cur = cur->children[c];  // 移动到下一个子节点
        }

        cur->func = func;  // 将最后一个字符对应的节点设置为对应的离线计算任务类型
    }

    // 在前缀树中查找离线计算任务名称对应的类型
    llm_offload_func_e find(const char * name) const {
        const node * cur = root;  // 从根节点开始

        for (int i = 0; ; ++i) {
            const uint8_t c = name[i];  // 获取名称中的字符

            if (!c) {
                break;  // 如果字符为空，则结束循环
            }

            if (!cur->children[c]) {
                return OFFLOAD_FUNC_NOP;  // 如果当前字符对应的子节点不存在，则返回空操作类型
            }

            cur = cur->children[c];  // 移动到下一个子节点
        }

        return cur->func;  // 返回最后一个字符对应的节点的离线计算任务类型
    }

    node * root = nullptr;  // 前缀树的根节点
};

// TODO: 将在后端 v2 版本中移除
// 创建一个常量无序映射，将字符串映射到枚举类型
static const std::unordered_map<const char *, llm_offload_func_e> k_offload_map = {
  //{ "inp_tokens",                 OFFLOAD_FUNC_NR  }, // TODO: missing K-quants get_rows kernel
  //{ "inp_embd",                   OFFLOAD_FUNC_NR  }, // TODO: missing K-quants get_rows kernel
    { "pos_embd",                   OFFLOAD_FUNC_NR  },  // 将字符串"pos_embd"映射到OFFLOAD_FUNC_NR枚举类型

    { "inp_pos",                    OFFLOAD_FUNC_KQ  }, // 将字符串"inp_pos"映射到OFFLOAD_FUNC_KQ枚举类型，通常用于KQ操作（例如rope）
    { "KQ_scale",                   OFFLOAD_FUNC_KQ  }, // 将字符串"KQ_scale"映射到OFFLOAD_FUNC_KQ枚举类型
    { "KQ_mask",                    OFFLOAD_FUNC_KQ  }, // 将字符串"KQ_mask"映射到OFFLOAD_FUNC_KQ枚举类型
    { "K_shift",                    OFFLOAD_FUNC_KQ  }, // 将字符串"K_shift"映射到OFFLOAD_FUNC_KQ枚举类型
    { "K_shifted",                  OFFLOAD_FUNC_KQ  }, // 将字符串"K_shifted"映射到OFFLOAD_FUNC_KQ枚举类型

    { "inp_norm",                   OFFLOAD_FUNC_NR  }, // 将字符串"inp_norm"映射到OFFLOAD_FUNC_NR枚举类型
    { "inp_norm_w",                 OFFLOAD_FUNC_NR  }, // 将字符串"inp_norm_w"映射到OFFLOAD_FUNC_NR枚举类型
    { "inp_norm_wb",                OFFLOAD_FUNC_NR  }, // 将字符串"inp_norm_wb"映射到OFFLOAD_FUNC_NR枚举类型

    { "norm",                       OFFLOAD_FUNC     }, // 将字符串"norm"映射到OFFLOAD_FUNC枚举类型
    { "norm_w",                     OFFLOAD_FUNC     }, // 将字符串"norm_w"映射到OFFLOAD_FUNC枚举类型
    { "norm_wb",                    OFFLOAD_FUNC     }, // 将字符串"norm_wb"映射到OFFLOAD_FUNC枚举类型

    { "attn_norm",                  OFFLOAD_FUNC     }, // 将字符串"attn_norm"映射到OFFLOAD_FUNC枚举类型
    { "attn_norm_2",                OFFLOAD_FUNC     }, // 将字符串"attn_norm_2"映射到OFFLOAD_FUNC枚举类型

    { "wqkv",                       OFFLOAD_FUNC_KQ  }, // 将字符串"wqkv"映射到OFFLOAD_FUNC_KQ枚举类型
    { "bqkv",                       OFFLOAD_FUNC_KQ  }, // 将字符串"bqkv"映射到OFFLOAD_FUNC_KQ枚举类型
    { "wqkv_clamped",               OFFLOAD_FUNC_KQ  }, // 将字符串"wqkv_clamped"映射到OFFLOAD_FUNC_KQ枚举类型

    { "tmpk",                       OFFLOAD_FUNC_KQ  }, // 将字符串"tmpk"映射到OFFLOAD_FUNC_KQ枚举类型
    { "tmpq",                       OFFLOAD_FUNC_KQ  }, // 将字符串"tmpq"映射到OFFLOAD_FUNC_KQ枚举类型
    { "tmpv",                       OFFLOAD_FUNC_V   }, // 将字符串"tmpv"映射到OFFLOAD_FUNC_V枚举类型
    { "Kcur",                       OFFLOAD_FUNC_KQ  }, // 将字符串"Kcur"映射到OFFLOAD_FUNC_KQ枚举类型
    { "Qcur",                       OFFLOAD_FUNC_KQ  }, // 将字符串"Qcur"映射到OFFLOAD_FUNC_KQ枚举类型
    { "Vcur",                       OFFLOAD_FUNC_V   }, // 将字符串"Vcur"映射到OFFLOAD_FUNC_V枚举类型

    { "krot",                       OFFLOAD_FUNC_KQ  }, // 将字符串"krot"映射到OFFLOAD_FUNC_KQ枚举类型
    { "qrot",                       OFFLOAD_FUNC_KQ  }, // 将字符串"qrot"映射到OFFLOAD_FUNC_KQ枚举类型
    { "kpass",                      OFFLOAD_FUNC_KQ  }, // 将字符串"kpass"映射到OFFLOAD_FUNC_KQ枚举类型
    { "qpass",                      OFFLOAD_FUNC_KQ  }, // 将字符串"qpass"映射到OFFLOAD_FUNC_KQ枚举类型
    { "krotated",                   OFFLOAD_FUNC_KQ  }, // 将字符串"krotated"映射到OFFLOAD_FUNC_KQ枚举类型
    { "qrotated",                   OFFLOAD_FUNC_KQ  }, // 将字符串"qrotated"映射到OFFLOAD_FUNC_KQ枚举类型
    { "q",                          OFFLOAD_FUNC_KQ  },  # 将字符串"q"映射到OFFLOAD_FUNC_KQ
    { "k",                          OFFLOAD_FUNC_KQ  },  # 将字符串"k"映射到OFFLOAD_FUNC_KQ
    { "kq",                         OFFLOAD_FUNC_KQ  },  # 将字符串"kq"映射到OFFLOAD_FUNC_KQ
    { "kq_scaled",                  OFFLOAD_FUNC_KQ  },  # 将字符串"kq_scaled"映射到OFFLOAD_FUNC_KQ
    { "kq_scaled_alibi",            OFFLOAD_FUNC_KQ  },  # 将字符串"kq_scaled_alibi"映射到OFFLOAD_FUNC_KQ
    { "kq_masked",                  OFFLOAD_FUNC_KQ  },  # 将字符串"kq_masked"映射到OFFLOAD_FUNC_KQ
    { "kq_soft_max",                OFFLOAD_FUNC_V   },  # 将字符串"kq_soft_max"映射到OFFLOAD_FUNC_V
    { "v",                          OFFLOAD_FUNC_V   },  # 将字符串"v"映射到OFFLOAD_FUNC_V
    { "kqv",                        OFFLOAD_FUNC_V   },  # 将字符串"kqv"映射到OFFLOAD_FUNC_V
    { "kqv_merged",                 OFFLOAD_FUNC_V   },  # 将字符串"kqv_merged"映射到OFFLOAD_FUNC_V
    { "kqv_merged_cont",            OFFLOAD_FUNC_V   },  # 将字符串"kqv_merged_cont"映射到OFFLOAD_FUNC_V
    { "kqv_wo",                     OFFLOAD_FUNC_V   },  # 将字符串"kqv_wo"映射到OFFLOAD_FUNC_V
    { "kqv_out",                    OFFLOAD_FUNC_V   },  # 将字符串"kqv_out"映射到OFFLOAD_FUNC_V

    { "ffn_inp",                    OFFLOAD_FUNC     },  # 将字符串"ffn_inp"映射到OFFLOAD_FUNC
    { "ffn_norm",                   OFFLOAD_FUNC     },  # 将字符串"ffn_norm"映射到OFFLOAD_FUNC

    { "ffn_up",                     OFFLOAD_FUNC     },  # 将字符串"ffn_up"映射到OFFLOAD_FUNC
    { "ffn_up_b",                   OFFLOAD_FUNC     },  # 将字符串"ffn_up_b"映射到OFFLOAD_FUNC
    { "ffn_gate",                   OFFLOAD_FUNC     },  # 将字符串"ffn_gate"映射到OFFLOAD_FUNC
    { "ffn_gate_b",                 OFFLOAD_FUNC     },  # 将字符串"ffn_gate_b"映射到OFFLOAD_FUNC
    { "ffn_gate_par",               OFFLOAD_FUNC     },  # 将字符串"ffn_gate_par"映射到OFFLOAD_FUNC
    { "ffn_down",                   OFFLOAD_FUNC     },  # 将字符串"ffn_down"映射到OFFLOAD_FUNC
    { "ffn_down_b",                 OFFLOAD_FUNC     },  # 将字符串"ffn_down_b"映射到OFFLOAD_FUNC
    { "ffn_out",                    OFFLOAD_FUNC     },  # 将字符串"ffn_out"映射到OFFLOAD_FUNC

    { "ffn_silu",                   OFFLOAD_FUNC     },  # 将字符串"ffn_silu"映射到OFFLOAD_FUNC
    { "ffn_gelu",                   OFFLOAD_FUNC     },  # 将字符串"ffn_gelu"映射到OFFLOAD_FUNC
    { "ffn_relu",                   OFFLOAD_FUNC     },  # 将字符串"ffn_relu"映射到OFFLOAD_FUNC
    { "ffn_sqr(relu)",              OFFLOAD_FUNC     },  # 将字符串"ffn_sqr(relu)"映射到OFFLOAD_FUNC

    { "l_out",                      OFFLOAD_FUNC     },  # 将字符串"l_out"映射到OFFLOAD_FUNC

    { "result_norm",                OFFLOAD_FUNC_EMB },  # 将字符串"result_norm"映射到OFFLOAD_FUNC_EMB
    { "result_output",              OFFLOAD_FUNC_OUT },  # 将字符串"result_output"映射到OFFLOAD_FUNC_OUT
    };

    // 创建一个 Trie 数据结构，用于存储离线函数的映射
    static llm_offload_trie k_offload_func_trie(k_offload_map);

    // 构建图形结构
    static struct ggml_cgraph * llama_build_graph(
        llama_context & lctx,
        const llama_batch & batch) {
    // 获取模型
    const auto & model = lctx.model;

    // 检查是否应该构建最坏情况的图形（用于内存测量）
    const bool worst_case = ggml_allocr_is_measure(lctx.alloc);

    // 跟踪已经分配的输入
    bool alloc_inp_tokens   = false;
    bool alloc_inp_embd     = false;
    bool alloc_inp_pos      = false;
    bool alloc_inp_KQ_scale = false;
    bool alloc_inp_KQ_mask  = false;
    bool alloc_inp_K_shift  = false;

    // 如果使用 CUBLAS，则进行离线处理
    const bool do_offload = true;

    // 处理过的非视图张量的数量
    int n_non_view = 0;

    // 此回调允许我们对每个张量应用自定义逻辑（例如 ggml-alloc、离线处理等）
    // 将在后端 v2 中删除
    //#define LLAMA_OFFLOAD_DEBUG

    // 如果不进行离线处理，则返回
    if (!do_offload) {
        return;
    }

    // 模型层数
    const int n_layer = model.hparams.n_layer;

    // GPU 层数
    const int n_gpu_layers = model.n_gpu_layers;
    const int i_gpu_start  = n_layer - n_gpu_layers;

    // 是否应该离线处理最终规范化？如果不计算嵌入，则是
    const bool offload_emb = lctx.embedding.empty();

    // 离线函数名称映射
    static const std::unordered_map<llm_offload_func_e, std::string, std::hash<int>> k_offload_func_name = {
        { OFFLOAD_FUNC_NOP, "CPU" },
        { OFFLOAD_FUNC_OUT, "CPU" },
#ifdef GGML_USE_CUBLAS
        { OFFLOAD_FUNC,     "GPU (CUDA)" },
        { OFFLOAD_FUNC_KQ,  "GPU (CUDA) KQ" },
        { OFFLOAD_FUNC_V,   "GPU (CUDA) V" },
        { OFFLOAD_FUNC_NR,  "GPU (CUDA) NR" },
        { OFFLOAD_FUNC_EMB, "GPU (CUDA) EMB" },
        // 定义一个静态的映射表，将函数名映射到对应的处理器类型
        static const std::map<llm_offload_func_e, const char*> k_offload_func_map = {
            { OFFLOAD_FUNC,     "CPU" },
            { OFFLOAD_FUNC_KQ,  "CPU" },
            { OFFLOAD_FUNC_V,   "CPU" },
            { OFFLOAD_FUNC_NR,  "CPU" },
            { OFFLOAD_FUNC_EMB, "CPU" },
        #endif // GGML_USE_CUBLAS
        };

        // 检查全局映射表，查找当前张量对应的处理函数
        llm_offload_func_e func_e = k_offload_func_trie.find(name);

        // 如果找不到对应的处理函数
        if (func_e == OFFLOAD_FUNC_NOP) {
        #ifdef LLAMA_OFFLOAD_DEBUG
            // 如果张量没有被处理，向用户发出警告
            if (worst_case) {
                LLAMA_LOG_WARN("%s: %32s: not offloaded (ref: %s)\n", __func__,
                        cur->name, "https://github.com/ggerganov/llama.cpp/pull/3837");
            }
#endif
// 结束预处理指令

            return;
        }
        // 如果函数类型为 OFFLOAD_FUNC_NOP 或 OFFLOAD_FUNC_OUT，则跳过
        // 否则根据 n_gpu_layers 和 n_layer 判断是否需要执行相应的函数
        switch (func_e) {
            case OFFLOAD_FUNC_NOP:
            case OFFLOAD_FUNC_OUT:
                break;
            case OFFLOAD_FUNC:
                if (n_gpu_layers < n_layer) {
                    if (il < i_gpu_start) {
                        func_e = OFFLOAD_FUNC_NOP;
                    }
                }
                break;
            case OFFLOAD_FUNC_NR:
                if (n_gpu_layers <= n_layer + 0) {
                    func_e = OFFLOAD_FUNC_NOP;
                }
                break;
            case OFFLOAD_FUNC_V:
                if (n_gpu_layers <= n_layer + 1) {
                    func_e = OFFLOAD_FUNC_NOP;
                }
                break;
            case OFFLOAD_FUNC_KQ:
                if (n_gpu_layers <= n_layer + 2) {
                    func_e = OFFLOAD_FUNC_NOP;
                }
                break;
            case OFFLOAD_FUNC_EMB:
                if (!offload_emb || n_gpu_layers < n_layer) {
                    func_e = OFFLOAD_FUNC_NOP;
                }
                break;
            default: GGML_ASSERT(false);
        }

        offload_func_t func = ggml_offload_nop;

        // 为了与 Metal 等兼容，需要进行以下设置
#ifdef GGML_USE_CUBLAS
        static offload_func_t ggml_offload_gpu = ggml_cuda_assign_buffers_no_alloc;
#else
        static offload_func_t ggml_offload_gpu = ggml_offload_nop;
#endif

        // 根据函数类型选择相应的 offload 函数
        switch (func_e) {
            case OFFLOAD_FUNC_NOP:
            case OFFLOAD_FUNC_OUT: func = ggml_offload_nop; break;
            case OFFLOAD_FUNC:
            case OFFLOAD_FUNC_KQ:
            case OFFLOAD_FUNC_V:
            case OFFLOAD_FUNC_NR:
            case OFFLOAD_FUNC_EMB: func = ggml_offload_gpu; break;
            default: GGML_ASSERT(false);
        }

        // 对张量应用 offload 函数
        func(cur);
#ifdef LLAMA_OFFLOAD_DEBUG
        # 如果开启了 LLAMA_OFFLOAD_DEBUG，且 worst_case 为真，则打印函数名、当前节点名和函数类型
        if (worst_case) {
            LLAMA_LOG_INFO("%s: %32s: %s\n", __func__, cur->name, k_offload_func_name.at(func_e).c_str());
        }
#endif
    };

    # 初始化结果指针为 NULL
    struct ggml_cgraph * result = NULL;

    # 创建构建上下文对象 llm，传入 lctx、batch、cb 和 worst_case
    struct llm_build_context llm(lctx, batch, cb, worst_case);

    # 初始化构建上下文对象 llm
    llm.init();

    # 根据不同的架构类型选择不同的构建方法
    switch (model.arch) {
        case LLM_ARCH_LLAMA:
            {
                result = llm.build_llama();
                # 如果 token 数量小于 80，则使用 build_llama 方法构建结果
                # 否则使用 build_llama_dense 方法构建结果
                // if (llm.n_tokens < 80)
                //     result = llm.build_llama();
                // else 
                //     result = llm.build_llama_dense();
            } break;
        case LLM_ARCH_BAICHUAN:
            {
                result = llm.build_baichuan();
            } break;
        case LLM_ARCH_FALCON:
            {
                result = llm.build_falcon();
            } break;
        case LLM_ARCH_STARCODER:
            {
                result = llm.build_starcoder();
            } break;
        case LLM_ARCH_PERSIMMON:
            {
                result = llm.build_persimmon();
            } break;
        case LLM_ARCH_REFACT:
            {
                result = llm.build_refact();
            } break;
        case LLM_ARCH_BLOOM:
            {
                result = llm.build_bloom();
            } break;
        case LLM_ARCH_MPT:
            {
                result = llm.build_mpt();
            } break;
         case LLM_ARCH_STABLELM:
            {
                result = llm.build_stablelm();
            } break;
        default:
            # 如果架构类型不在上述枚举中，则断言失败
            GGML_ASSERT(false);
    }

    # 释放构建上下文对象 llm
    llm.free();
    # 如果是最坏情况
    if (worst_case) {
        # 初始化非视图节点总数
        int n_non_view_total = 0;

        # 遍历结果中的节点
        for (int i = 0; i < result->n_nodes; ++i) {
            # 如果节点的视图源为空指针，则非视图节点总数加一
            if (result->nodes[i]->view_src == nullptr) {
                n_non_view_total++;
            }
        }

        # 输出非视图张量处理信息
        LLAMA_LOG_INFO("%s: non-view tensors processed: %d/%d\n", __func__, n_non_view, n_non_view_total);

        # 如果非视图节点数量不等于非视图节点总数
        if (n_non_view != n_non_view_total) {
            # 输出警告信息
            LLAMA_LOG_WARN("%s: ****************************************************************\n", __func__);
            LLAMA_LOG_WARN("%s: not all non-view tensors have been processed with a callback\n",     __func__);
            LLAMA_LOG_WARN("%s: this can indicate an inefficiency in the graph implementation\n",    __func__);
            LLAMA_LOG_WARN("%s: build with LLAMA_OFFLOAD_DEBUG for more info\n",                     __func__);
            LLAMA_LOG_WARN("%s: ref: https://github.com/ggerganov/llama.cpp/pull/3837\n",            __func__);
            LLAMA_LOG_WARN("%s: ****************************************************************\n", __func__);
        }
    }

    # 返回结果
    return result;
// 解码一批标记，通过评估变换器
//
//   - lctx:      llama 上下文
//   - batch:     要评估的批次
//
// 成功时返回 0
// 警告时返回正整数
// 出错时返回负整数
//
static int llama_decode_internal(
         llama_context & lctx,
           llama_batch   batch) {
    const uint32_t n_tokens = batch.n_tokens;

    if (n_tokens == 0) {
        LLAMA_LOG_ERROR("%s: n_tokens == 0", __func__);
        return -1;
    }

    const auto & model   = lctx.model;
    const auto & hparams = model.hparams;
    const auto & cparams = lctx.cparams;

    const auto n_batch = cparams.n_batch;

    GGML_ASSERT(n_tokens <= n_batch);

    int n_threads = n_tokens == 1 ? cparams.n_threads : cparams.n_threads_batch;
    GGML_ASSERT((!batch.token && batch.embd) || (batch.token && !batch.embd)); // NOLINT

    const int64_t t_start_us = ggml_time_us();

#ifdef GGML_USE_MPI
    // TODO: 在 #3228 之后需要修复
    GGML_ASSERT(false && "not implemented");
    //ggml_mpi_eval_init(lctx.ctx_mpi, &n_tokens, &n_past, &n_threads);
#endif

    GGML_ASSERT(n_threads > 0);

    auto & kv_self = lctx.kv_self;

    GGML_ASSERT(!!kv_self.ctx);

    const int64_t n_embd  = hparams.n_embd;
    const int64_t n_vocab = hparams.n_vocab;

    // 用于更平滑的批处理 API 过渡的辅助函数
    // 在废弃 llama_eval 调用之后，这些将被移除
    std::vector<llama_pos> pos;

    std::vector<int32_t>                   n_seq_id;
    std::vector<llama_seq_id *>            seq_id_arr;
    std::vector<std::vector<llama_seq_id>> seq_id;

    if (batch.pos == nullptr) {
        pos.resize(n_tokens);
        for (uint32_t i = 0; i < n_tokens; i++) {
            pos[i] = batch.all_pos_0 + i*batch.all_pos_1;
        }

        batch.pos = pos.data();
    }
    # 如果批处理的序列 ID 为空指针
    if (batch.seq_id == nullptr) {
        # 调整 n_seq_id 的大小为 n_tokens
        n_seq_id.resize(n_tokens);
        # 调整 seq_id 的大小为 n_tokens
        seq_id.resize(n_tokens);
        # 调整 seq_id_arr 的大小为 n_tokens
        seq_id_arr.resize(n_tokens);
        # 遍历 n_tokens
        for (uint32_t i = 0; i < n_tokens; i++) {
            # 将 n_seq_id[i] 设置为 1
            n_seq_id[i] = 1;
            # 调整 seq_id[i] 的大小为 1
            seq_id[i].resize(1);
            # 将 seq_id[i][0] 设置为 batch.all_seq_id
            seq_id[i][0] = batch.all_seq_id;
            # 将 seq_id[i] 的数据指针赋值给 seq_id_arr[i]
            seq_id_arr[i] = seq_id[i].data();
        }

        # 将 n_seq_id 的数据指针赋值给 batch.n_seq_id
        batch.n_seq_id = n_seq_id.data();
        # 将 seq_id_arr 的数据指针赋值给 batch.seq_id
        batch.seq_id = seq_id_arr.data();
    }

    # 如果在 kv_self 中找不到 batch 对应的槽位
    if (!llama_kv_cache_find_slot(kv_self, batch)) {
        # 返回 1
        return 1;
    }

    # 为了避免完全利用缓存，使用启发式方法
    # 在足够的迭代之后，这种启发式方法的好处会消失
    # 如果开始对缓存进行碎片整理，这种启发式方法的好处会更加重要
    # 将 kv_self.n 设置为 cparams.n_ctx 和 llama_kv_cache_cell_max(kv_self) 中的较小值
    kv_self.n = std::min((int32_t) cparams.n_ctx, std::max(32, llama_kv_cache_cell_max(kv_self)));

    # 重置 lctx.alloc 中的分配器
    ggml_allocr_reset(lctx.alloc);

    # 使用 batch 构建图形
    ggml_cgraph * gf = llama_build_graph(lctx, batch);

    # 在 lctx.alloc 中分配图形
    ggml_allocr_alloc_graph(lctx.alloc, gf);

    # 获取结果节点和嵌入节点
    struct ggml_tensor * res        = gf->nodes[gf->n_nodes - 1];
    struct ggml_tensor * embeddings = gf->nodes[gf->n_nodes - 2];

    # 断言结果节点的名称为 "result_output"
    GGML_ASSERT(strcmp(res->name,        "result_output") == 0);
    # 断言嵌入节点的名称为 "result_norm"
    GGML_ASSERT(strcmp(embeddings->name, "result_norm")   == 0);
#ifdef GGML_USE_CUBLAS
    // 如果使用了 CUBLAS，则对每个叶子节点进行操作
    for (int i = 0; i < gf->n_leafs; i++) {
        // 获取当前叶子节点
        ggml_tensor * node = gf->leafs[i];
        // 如果节点的后端是 GPU，并且没有额外数据
        if (node->backend == GGML_BACKEND_GPU && node->extra == NULL) {
            // 计算节点数据在缓冲区中的偏移量，并分配给节点
            ggml_cuda_assign_scratch_offset(node, (char*)node->data - (char *) lctx.buf_alloc.data);
            // 将节点数据拷贝到 GPU 设备
            ggml_cuda_copy_to_device(node);
        }
    }

    // 对每个非叶子节点进行操作
    for (int i = 0; i < gf->n_nodes; i++) {
        // 获取当前非叶子节点
        ggml_tensor * node = gf->nodes[i];
        // 如果节点的后端是 GPU，并且没有额外数据
        if (node->backend == GGML_BACKEND_GPU && node->extra == NULL) {
            // 计算节点数据在缓冲区中的偏移量，并分配给节点
            ggml_cuda_assign_scratch_offset(node, (char*)node->data - (char *) lctx.buf_alloc.data);
        }
    }

    // 如果 lctx.embedding 不为空
    if (!lctx.embedding.empty()) {
        // 将 embeddings 的后端设置为 CPU
        embeddings->backend = GGML_BACKEND_CPU;
    }
    // 将 res 的后端设置为 CPU
    res->backend = GGML_BACKEND_CPU;
#endif

    // 输出图构建时间和节点、叶子节点数量
    // LLAMA_LOG_INFO("graph build time: %.3f ms (%d nodes, %d leafs)\n", (ggml_time_us() - t_start_us)/1000.0, gf->n_nodes, gf->n_leafs);

    // 对于大型提示，如果启用了 BLAS，则最好只使用一个线程
    // 否则，线程会在 BLAS 调用时自旋锁等待，降低性能
    // TODO: 这对于 Apple Silicon 非常重要，因为 CBLAS 仍然表现非常好
    //       我们仍然需要一些线程来处理所有非矩阵乘法操作，但不要太多以避免干扰 BLAS 调用。需要更好的解决方案
    if (n_tokens >= 32 && ggml_cpu_has_blas() && !ggml_cpu_has_gpublas()) {
        // 如果标记数量大于等于 32，并且具有 BLAS 但没有 GPU BLAS，则将线程数限制在 4 以内
        n_threads = std::min(4, n_threads);
    }

    // 如果所有张量都可以在 GPU 上运行，则使用多于 1 个线程是有害的
    # 检查模型的架构是否支持完全卸载
    const bool full_offload_supported =
        model.arch == LLM_ARCH_LLAMA      ||  # 检查模型的架构是否为 LLAMA
        model.arch == LLM_ARCH_BAICHUAN   ||  # 检查模型的架构是否为 BAICHUAN
        model.arch == LLM_ARCH_FALCON     ||  # 检查模型的架构是否为 FALCON
        model.arch == LLM_ARCH_REFACT     ||  # 检查模型的架构是否为 REFACT
        model.arch == LLM_ARCH_MPT        ||  # 检查模型的架构是否为 MPT
        model.arch == LLM_ARCH_STARCODER  ||  # 检查模型的架构是否为 STARCODER
        model.arch == LLM_ARCH_STABLELM;       # 检查模型的架构是否为 STABLELM
    
    # 检查模型是否完全卸载
    const bool fully_offloaded = model.n_gpu_layers >= (int) hparams.n_layer + 3;
    # 如果 CPU 支持 cuBLAS、模型支持完全卸载、并且模型已经完全卸载，则设置线程数为 8
    if (ggml_cpu_has_cublas() && full_offload_supported && fully_offloaded) {
        n_threads = 8;
    }
#if GGML_USE_MPI
    // 如果使用 MPI，则获取层数并在 MPI 上进行图计算的预处理
    const int64_t n_layer = hparams.n_layer;
    ggml_mpi_graph_compute_pre(lctx.ctx_mpi, gf, n_layer);
#endif

#ifdef GGML_USE_METAL
    // 如果使用 Metal
    if (lctx.ctx_metal) {
        // 设置 Metal 上下文的线程数，并在 Metal 上进行图计算
        ggml_metal_set_n_cb     (lctx.ctx_metal, n_threads);
        ggml_metal_graph_compute(lctx.ctx_metal, gf);
    } else {
        // 在普通环境下进行图计算
        ggml_graph_compute_helper(lctx.work_buffer, gf, n_threads);
    }
#else
    // 在普通环境下进行图计算
    ggml_graph_compute_helper(lctx.work_buffer, gf, n_threads);
#endif

#if GGML_USE_MPI
    // 如果使用 MPI，则在 MPI 上进行图计算的后处理
    ggml_mpi_graph_compute_post(lctx.ctx_mpi, gf, n_layer);
#endif

    // 更新键值环形缓冲区
    {
        // 如果键值缓冲区需要进行移位，则将其标记为不需要移位，并将所有 delta 值清零
        if (kv_self.has_shift) {
            kv_self.has_shift = false;
            for (uint32_t i = 0; i < kv_self.size; ++i) {
                kv_self.cells[i].delta = 0;
            }
        }

        // 将键值缓冲区的头指针向前移动 n_tokens 个位置
        kv_self.head += n_tokens;

        // 确保键值缓冲区的头指针指向有效的索引
        if (kv_self.head >= kv_self.size) {
            kv_self.head = 0;
        }
    }

#ifdef GGML_PERF
    // 打印每个 ggml 操作的时间信息（用于调试目的）
    // 需要定义 GGML_PERF
    ggml_graph_print(gf);
#endif

    // 以 dot 格式绘制计算图（用于调试目的）
    //if (n_past%100 == 0) {
    //    ggml_graph_dump_dot(gf, NULL, "llama.dot");
    //}

    // 提取 logits
    // TODO: 如果只需要嵌入，则不计算和提取 logits
    //       需要更新图以跳过 "result_output"
    {
        // 获取logits_out的引用
        auto & logits_out = lctx.logits;

        // 如果batch.logits存在
        if (batch.logits) {
            // 调整logits_out的大小以容纳所有的logits
            logits_out.resize(n_vocab * n_tokens);
            // 遍历每个token
            for (uint32_t i = 0; i < n_tokens; i++) {
                // 如果logits为0，则跳过
                if (batch.logits[i] == 0) {
                    continue;
                }
                // 将数据从res拷贝到logits_out中
                memcpy(logits_out.data() + (n_vocab*i), (float *) ggml_get_data(res) + (n_vocab*i), sizeof(float)*n_vocab);
            }
        } else if (lctx.logits_all) {
            // 调整logits_out的大小以容纳所有的logits
            logits_out.resize(n_vocab * n_tokens);
            // 将数据从embeddings拷贝到logits_out中
            memcpy(logits_out.data(), (float *) ggml_get_data(res), sizeof(float)*n_vocab*n_tokens);
        } else {
            // 调整logits_out的大小以容纳一个token的logits
            logits_out.resize(n_vocab);
            // 将数据从embeddings拷贝到logits_out中
            memcpy(logits_out.data(), (float *) ggml_get_data(res) + (n_vocab*(n_tokens - 1)), sizeof(float)*n_vocab);
        }
    }

    // 提取嵌入
    if (!lctx.embedding.empty()) {
        // 获取embedding_out的引用
        auto & embedding_out = lctx.embedding;

        // 调整embedding_out的大小以容纳嵌入的数据
        embedding_out.resize(n_embd);
        // 将数据从embeddings拷贝到embedding_out中
        memcpy(embedding_out.data(), (float *) ggml_get_data(embeddings) + (n_embd*(n_tokens - 1)), sizeof(float)*n_embd);
    }

    // 仅对单个token的评估进行性能测量
    if (n_tokens == 1) {
        // 计算单个token的评估时间
        lctx.t_eval_us += ggml_time_us() - t_start_us;
        // 增加单个token的评估次数
        lctx.n_eval++;
    }
    else if (n_tokens > 1) {
        // 计算多个token的评估时间
        lctx.t_p_eval_us += ggml_time_us() - t_start_us;
        // 增加多个token的评估次数
        lctx.n_p_eval += n_tokens;
    }

    // 在第一次评估时获取更准确的加载时间
    // TODO: 修复这个问题
    if (!lctx.has_evaluated_once) {
        // 计算加载时间
        lctx.t_load_us = ggml_time_us() - lctx.t_start_us;
        // 标记已经评估过一次
        lctx.has_evaluated_once = true;
    }

    // 返回0表示成功
    return 0;
}

//
// tokenizer
//

// 获取词汇的类型
static enum llama_vocab_type llama_vocab_get_type(const llama_vocab & vocab) {
    return vocab.type;
}

// 判断词汇是否为普通标记
static bool llama_is_normal_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_NORMAL;
}

// 判断词汇是否为未知标记
static bool llama_is_unknown_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_UNKNOWN;
}

// 判断词汇是否为控制标记
static bool llama_is_control_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_CONTROL;
}

// 判断词汇是否为字节标记
static bool llama_is_byte_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_BYTE;
}

// 判断词汇是否为用户定义标记
static bool llama_is_user_defined_token(const llama_vocab& vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_USER_DEFINED;
}

// 将词汇转换为字节
static uint8_t llama_token_to_byte(const llama_vocab& vocab, llama_token id) {
    GGML_ASSERT(llama_is_byte_token(vocab, id));
    const auto& token_data = vocab.id_to_token.at(id);
    switch (llama_vocab_get_type(vocab)) {
    case LLAMA_VOCAB_TYPE_SPM: {
        auto buf = token_data.text.substr(3, 2);
        return strtol(buf.c_str(), NULL, 16);
    }
    case LLAMA_VOCAB_TYPE_BPE: {
        GGML_ASSERT(false);
        return unicode_to_bytes_bpe(token_data.text);
    }
    default:
        GGML_ASSERT(false);
    }
}

// 将字节转换为词汇
static llama_token llama_byte_to_token(const llama_vocab & vocab, uint8_t ch) {
    static const char * hex = "0123456789ABCDEF";
    switch (llama_vocab_get_type(vocab)) {
    case LLAMA_VOCAB_TYPE_SPM: {
        const char buf[7] = { '<', '0', 'x', hex[ch >> 4], hex[ch & 15], '>', 0 };
        return vocab.token_to_id.at(buf);
    }
    case LLAMA_VOCAB_TYPE_BPE: {
        return vocab.token_to_id.at(bytes_to_unicode_bpe(ch));
    }
    default:
        GGML_ASSERT(false);
    }
}

// 转义空白字符
static void llama_escape_whitespace(std::string & text) {
    # 调用 replace_all 函数，将文本中的空格替换为特定的 Unicode 字符
    replace_all(text, " ", "\xe2\x96\x81");
// 静态函数，用于将字符串中的特定字符替换为空格
static void llama_unescape_whitespace(std::string & word) {
    replace_all(word, "\xe2\x96\x81", " ");
}

// 定义结构体 llm_symbol，包含索引、文本、大小等信息
struct llm_symbol {
    using index = int;
    index prev;
    index next;
    const char * text;
    size_t n;
};

// 静态断言，检查 llm_symbol 是否是平凡可复制的
static_assert(std::is_trivially_copyable<llm_symbol>::value, "llm_symbol is not trivially copyable");

// SPM tokenizer 结构体，包含比较器、队列存储、左右索引、分数和大小等信息
struct llm_bigram_spm {
    // 定义比较器，用于比较 llm_bigram_spm 结构体
    struct comparator {
        bool operator()(llm_bigram_spm & l, llm_bigram_spm & r) {
            return (l.score < r.score) || (l.score == r.score && l.left > r.left);
        }
    };
    using queue_storage = std::vector<llm_bigram_spm>;
    using queue = std::priority_queue<llm_bigram_spm, queue_storage, comparator>;
    llm_symbol::index left;
    llm_symbol::index right;
    float score;
    size_t size;
};

// SPM tokenizer 结构体，包含词汇表信息
struct llm_tokenizer_spm {
    llm_tokenizer_spm(const llama_vocab & vocab): vocab(vocab) {}

    // 重新分割函数，用于将符号重新分割成词汇表中的标识
    private:
    void resegment(llm_symbol & symbol, std::vector<llama_vocab::id> & output) {
        auto text = std::string(symbol.text, symbol.n);
        auto token = vocab.token_to_id.find(text);

        // 检查是否需要支持 is_unused
        if (token != vocab.token_to_id.end()) {
            output.push_back((*token).second);
            return;
        }

        const auto p = rev_merge.find(text);

        if (p == rev_merge.end()) {
            // 输出未形成标记的符号作为字节
            for (int j = 0; j < (int)symbol.n; ++j) {
                llama_vocab::id token_id = llama_byte_to_token(vocab, symbol.text[j]);
                output.push_back(token_id);
            }
            return;
        }

        resegment(symbols[p->second.first],  output);
        resegment(symbols[p->second.second], output);
    }
}
    // 尝试添加双字节词
    void try_add_bigram(int left, int right) {
        // 如果左边或右边的索引为-1，则返回
        if (left == -1 || right == -1) {
            return;
        }

        // 从symbols数组中获取left和right位置的文本，并拼接成字符串
        const std::string text = std::string(symbols[left].text, symbols[left].n + symbols[right].n);
        // 在词汇表中查找该文本对应的标记
        auto token = vocab.token_to_id.find(text);

        // 如果在词汇表中找不到该文本对应的标记，则返回
        if (token == vocab.token_to_id.end()) {
            return;
        }

        // 如果标记的值大于等于id_to_token数组的大小，则返回
        if (static_cast<size_t>((*token).second) >= vocab.id_to_token.size()) {
            return;
        }

        // 获取标记对应的数据
        const auto & tok_data = vocab.id_to_token[(*token).second];

        // 创建llm_bigram_spm对象，并设置其属性值
        llm_bigram_spm bigram;
        bigram.left  = left;
        bigram.right = right;
        bigram.score = tok_data.score;
        bigram.size  = text.size();

        // 将bigram对象添加到工作队列中
        work_queue.push(bigram);

        // 将文本和左右位置的pair添加到rev_merge映射中
        rev_merge[text] = std::make_pair(left, right);
    }

    // 词汇表的引用
    const llama_vocab & vocab;

    // 符号数组
    std::vector<llm_symbol> symbols;
    // bigram队列
    llm_bigram_spm::queue work_queue;

    // 存储文本和左右位置的pair的映射
    std::map<std::string, std::pair<int, int>> rev_merge;
// 结构体 llm_bigram_bpe，用于存储双字节符号的信息
struct llm_bigram_bpe {
    // 定义比较器，用于优先队列的排序
    struct comparator {
        bool operator()(const llm_bigram_bpe & l, const llm_bigram_bpe & r) const {
            return l.rank > r.rank || (l.rank == r.rank && l.left > r.left);
        }
    };

    // 定义优先队列的存储类型
    using queue_storage = std::vector<llm_bigram_bpe>;
    // 定义优先队列
    using queue = std::priority_queue<llm_bigram_bpe, queue_storage, comparator>;
    // 左侧符号的索引
    llm_symbol::index left;
    // 右侧符号的索引
    llm_symbol::index right;
    // 文本内容
    std::string text;
    // 排名
    int rank;
    // 大小
    size_t size;
};

// 结构体 llm_tokenizer_bpe，用于BPE分词
struct llm_tokenizer_bpe {
    // 构造函数，接受 llama_vocab 对象作为参数
    llm_tokenizer_bpe(const llama_vocab & vocab): vocab(vocab) {}

    // 添加新的双字节符号
    private:
    void add_new_bigram(int left, int right) {
        if (left == -1 || right == -1) {
            return;
        }

        // 获取左侧符号和右侧符号的文本内容
        std::string left_token  = std::string(symbols[left].text,  symbols[left].n);
        std::string right_token = std::string(symbols[right].text, symbols[right].n);

        int rank_found = -1;

        // 在词汇表中查找双字节符号的排名
        rank_found = vocab.find_bpe_rank(left_token, right_token);

        if (rank_found < 0) {
            return;
        }

        // 创建新的双字节符号对象
        llm_bigram_bpe bigram;

        bigram.left  = left;
        bigram.right = right;
        bigram.text  = left_token + right_token;
        bigram.size  = left_token.size() + right_token.size();
        bigram.rank  = rank_found;

        // 将新的双字节符号加入工作队列
        work_queue.push(bigram);
    }

    // llama_vocab 对象
    const llama_vocab & vocab;

    // 符号列表
    std::vector<llm_symbol> symbols;
    // 最终符号列表
    std::vector<llm_symbol> symbols_final;

    // 工作队列
    llm_bigram_bpe::queue work_queue;
};

// 枚举类型，表示片段缓冲区的变体类型
typedef enum FRAGMENT_BUFFER_VARIANT_TYPE{
    FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN,
    FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT
} FRAGMENT_BUFFER_VARIANT_TYPE;

// 结构体，表示片段缓冲区的变体
struct fragment_buffer_variant{
    // 构造函数，接受 token 作为参数
    fragment_buffer_variant(llama_vocab::id _token)
    # 定义一个名为 fragment_buffer_variant 的类，包含以下成员变量和构造函数
    class fragment_buffer_variant {
        # 构造函数，初始化 type 为 FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN，token 为 _token，raw_text 为 _dummy，offset 为 0，length 为 0
        fragment_buffer_variant(llama_vocab::id _token)
            : type(FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN),
              token(_token),
              raw_text(_dummy),
              offset(0),
              length(0){}
        # 构造函数，初始化 type 为 FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT，token 为 -1，raw_text 为 _raw_text，offset 为 _offset，length 为 _length
        fragment_buffer_variant(const std::string & _raw_text, int64_t _offset, int64_t _length)
            : type(FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT),
              token((llama_vocab::id)-1),
              raw_text(_raw_text),
              offset(_offset),
              length(_length){
                  # 断言 _offset 大于等于 0
                  GGML_ASSERT( _offset >= 0 );
                  # 断言 _length 大于等于 1
                  GGML_ASSERT( _length >= 1 );
                  # 断言 offset 加上 length 小于等于 raw_text 的长度
                  GGML_ASSERT( offset + length <= raw_text.length() );
              }
    
        # 声明成员变量 type 为常量，不可修改
        const FRAGMENT_BUFFER_VARIANT_TYPE type;
        # 声明成员变量 token 为常量，不可修改
        const llama_vocab::id token;
        # 声明成员变量 _dummy 为常量，不可修改
        const std::string _dummy;
        # 声明成员变量 raw_text 为常量引用，不可修改
        const std::string & raw_text;
        # 声明成员变量 offset 为常量，不可修改
        const uint64_t offset;
        # 声明成员变量 length 为常量，不可修改
        const uint64_t length;
    }
};
// 结束了一个代码块，可能是一个类或者函数的结束

// 定义了一个预处理器宏，用于调试
// #define PRETOKENIZERDEBUG

// 定义了一个名为tokenizer_st_partition的静态函数，接受词汇表和文本片段缓冲列表作为参数
static void tokenizer_st_partition(const llama_vocab & vocab, std::forward_list<fragment_buffer_variant> & buffer)
{
    // 遍历特殊标记列表中的每个特殊标记
    for (const auto & st: vocab.special_tokens_cache) {
        const auto & special_token = st.first;
        const auto & special_id    = st.second;

        // 遍历文本片段缓冲列表中的每个文本片段
        std::forward_list<fragment_buffer_variant>::iterator it = buffer.begin();
        while (it != buffer.end()) {
            auto & fragment = (*it);

            // 如果一个片段是文本（尚未处理）
            if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT) {
                auto * raw_text = &(fragment.raw_text);

                auto raw_text_base_offset = fragment.offset;
                auto raw_text_base_length = fragment.length;

                // 循环遍历文本
                while (true) {
                    // 在该片段中查找给定特殊标记的第一个出现位置
                    // 仅传递偏移量参数限制了“搜索区域”，但匹配坐标仍然是相对于源完整的原始文本
                    auto match = raw_text->find(special_token, raw_text_base_offset);

                    // 没有找到匹配项，停止处理该特殊标记的片段
                    if (match == std::string::npos) break;

                    // 检查匹配项是否在偏移量 <-> 长度的范围内
                    if (match + special_token.length() > raw_text_base_offset + raw_text_base_length) break;

#ifdef PRETOKENIZERDEBUG
                    // 如果定义了预处理器宏，则输出调试信息
                    fprintf(stderr, "FF: (%ld %ld %ld) '%s'\n", raw_text->length(), raw_text_base_offset, raw_text_base_length, raw_text->substr(raw_text_base_offset, raw_text_base_length).c_str());
                    // 计算匹配位置距离缓冲区起始位置的偏移量
                    auto source = std::distance(buffer.begin(), it);

                    // 如果匹配位置比基本偏移量更远，则说明匹配位置左侧有一些文本
                    if (match > raw_text_base_offset) {
                        // 左侧文本
                        const int64_t left_reminder_offset = raw_text_base_offset + 0;
                        const int64_t left_reminder_length = match - raw_text_base_offset;
                        // 在匹配位置之前插入左侧文本
                        buffer.emplace_after(it, (*raw_text), left_reminder_offset, left_reminder_length);

#ifdef PRETOKENIZERDEBUG
                        // 输出左侧文本信息
                        fprintf(stderr, "FL: (%ld %ld) '%s'\n", left_reminder_offset, left_reminder_length, raw_text->substr(left_reminder_offset, left_reminder_length).c_str());
#endif
                        it++;
                    }

                    // 插入特殊标记
                    buffer.emplace_after(it, special_id);
                    it++;

                    // 如果匹配位置加上特殊标记长度小于原始文本基本偏移量加上基本长度
                    // 则说明匹配位置右侧有一些文本
                    if (match + special_token.length() < raw_text_base_offset + raw_text_base_length) {
                        const int64_t right_reminder_offset = match + special_token.length();
                        const int64_t right_reminder_length = raw_text_base_length - ((match - raw_text_base_offset) + special_token.length());
                        // 在匹配位置之后插入右侧文本
                        buffer.emplace_after(it, (*raw_text), right_reminder_offset, right_reminder_length);

#ifdef PRETOKENIZERDEBUG
                        // 输出右侧文本信息
                        fprintf(stderr, "FR: (%ld %ld) '%s'\n", right_reminder_offset, right_reminder_length, raw_text->substr(right_reminder_offset, right_reminder_length).c_str());
#endif

                        // 增加迭代器的值
                        it++;

                        // 如果source为0，则删除buffer的第一个元素
                        if (source == 0) {
                            buffer.erase_after(buffer.before_begin());
                        } else {
                            // 否则删除指定位置的元素
                            buffer.erase_after(std::next(buffer.begin(), (source-1)));
                        }

                        // 重复右侧的操作
                        raw_text_base_offset = right_reminder_offset;
                        raw_text_base_length = right_reminder_length;

#ifdef PRETOKENIZERDEBUG
                        // 打印右侧的偏移和长度，以及对应的文本内容
                        fprintf(stderr, "RR: (%ld %ld) '%s'\n", raw_text_base_offset, raw_text_base_length, raw_text->substr(raw_text_base_offset, raw_text_base_length).c_str());
#endif
                    } else {
                        // 如果不满足条件，则执行以下操作
                        if (source == 0) {
                            buffer.erase_after(buffer.before_begin());
                        } else {
                            buffer.erase_after(std::next(buffer.begin(), (source-1)));
                        }
                        // 跳出循环
                        break;
                    }
                }
            }
            // 增加迭代器的值
            it++;
        }
    }
}

// 内部LLAMA标记化函数
static std::vector<llama_vocab::id> llama_tokenize_internal(const llama_vocab & vocab, std::string raw_text, bool bos, bool special) {
    // 创建LLAMA标记化结果的向量
    std::vector<llama_vocab::id> output;

    // OG标记化器行为：
    //
    // tokenizer.encode('', add_bos=True)  返回 [1]
    // tokenizer.encode('', add_bos=False) 返回 []

    // 如果需要添加bos，并且vocab.special_bos_id不为-1，则将其添加到输出向量中
    if (bos && vocab.special_bos_id != -1) {
        output.push_back(vocab.special_bos_id);
    }

    // 如果原始文本为空，则直接返回输出向量
    if (raw_text.empty()) {
        return output;
    }

    // 创建前向列表fragment_buffer，并将原始文本作为第一个元素插入其中
    std::forward_list<fragment_buffer_variant> fragment_buffer;
    fragment_buffer.emplace_front( raw_text, 0, raw_text.length() );

    // 如果需要特殊处理，则调用tokenizer_st_partition函数
    if (special) tokenizer_st_partition( vocab, fragment_buffer );
    # 根据词汇表类型进行不同的处理
    switch (vocab.type) {
        # 如果词汇表类型是LLAMA_VOCAB_TYPE_SPM
        case LLAMA_VOCAB_TYPE_SPM:
            {
                # 遍历片段缓冲区中的每个片段
                for (const auto & fragment: fragment_buffer)
                {
                    # 如果片段类型是原始文本
                    if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT)
                    {
                        # 在不添加前导空格的情况下，我们得不到与原始分词器相同的结果

                        # TODO: 可能可以通过修改llm_tokenizer_x以使用字符串偏移量来操作，就像pre-tokenizer一样，并将'add space prefix'作为布尔参数传递，从而完全消除这个字符串复制
                        #
                        # 创建原始文本，如果是特殊情况则为空字符串，否则为" "，再加上片段的子字符串
                        auto raw_text = (special ? "" : " ") + fragment.raw_text.substr(fragment.offset, fragment.length);
#ifdef PRETOKENIZERDEBUG
                        // 如果定义了预处理器标识符 PRETOKENIZERDEBUG，则输出调试信息
                        fprintf(stderr,"TT: (%ld %ld %ld) '%s'\n", raw_text.length(), fragment.offset, fragment.length, raw_text.c_str());
#endif
                        // 根据词汇表创建 LLAMA 分词器对象
                        llm_tokenizer_spm tokenizer(vocab);
                        // 去除原始文本中的空白字符
                        llama_escape_whitespace(raw_text);
                        // 对原始文本进行分词处理，结果存储在输出容器中
                        tokenizer.tokenize(raw_text, output);
                    }
                    else // if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN)
                    {
                        // 如果片段类型为 TOKEN，则将其添加到输出容器中
                        output.push_back(fragment.token);
                    }
                }
            } break;
        case LLAMA_VOCAB_TYPE_BPE:
            {
                for (const auto & fragment: fragment_buffer)
                {
                    if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT)
                    {
                        // 获取片段中指定偏移和长度的原始文本
                        auto raw_text = fragment.raw_text.substr(fragment.offset, fragment.length);

#ifdef PRETOKENIZERDEBUG
                        // 如果定义了预处理器标识符 PRETOKENIZERDEBUG，则输出调试信息
                        fprintf(stderr,"TT: (%ld %ld %ld) '%s'\n", raw_text.length(), fragment.offset, fragment.length, raw_text.c_str());
#endif
                        // 根据词汇表创建 LLAMA BPE 分词器对象
                        llm_tokenizer_bpe tokenizer(vocab);
                        // 对原始文本进行 BPE 分词处理，结果存储在输出容器中
                        tokenizer.tokenize(raw_text, output);
                    }
                    else // if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN)
                    {
                        // 如果片段类型为 TOKEN，则将其添加到输出容器中
                        output.push_back(fragment.token);
                    }
                }
            } break;
    }

    // 返回处理后的输出结果
    return output;
}

//
// grammar - internal
//

// 定义部分 UTF-8 结构体，用于存储部分生成的 UTF-8 序列
struct llama_partial_utf8 {
    uint32_t value;    // 目前的位值（未移位）
    int      n_remain; // 剩余的字节数；-1 表示无效序列
};

// 定义 LLAMA 语法结构体
struct llama_grammar {
    const std::vector<std::vector<llama_grammar_element>>   rules; // 规则集合
    std::vector<std::vector<const llama_grammar_element *>> stacks; // 栈集合

    // 用于存储从接受的标记中部分生成的 UTF-8 序列的缓冲区
    # 声明一个名为llama_partial_utf8的变量，类型为partial_utf8
// 结构体定义，用于存储候选的语法信息
struct llama_grammar_candidate {
    size_t               index;  // 索引
    const uint32_t     * code_points;  // 代码点指针
    llama_partial_utf8   partial_utf8;  // 部分 UTF-8
};

// 解码可能以不完整序列结尾的 UTF-8 字符串。添加终止的 0 以便用作指针。如果遇到无效序列，则返回 `llama_partial_utf8.n_remain == -1`。
static std::pair<std::vector<uint32_t>, llama_partial_utf8> decode_utf8(
        const char         * src,  // 源字符串
        llama_partial_utf8   partial_start) {  // 部分开始
    static const int      lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 2, 2, 3, 4 };  // 查找表
    const char          * pos      = src;  // 位置指针
    std::vector<uint32_t> code_points;  // 代码点向量
    uint32_t              value    = partial_start.value;  // 值
    int                   n_remain = partial_start.n_remain;  // 剩余数量

    // 继续之前的解码，如果适用
    while (*pos != 0 && n_remain > 0) {
        uint8_t next_byte = static_cast<uint8_t>(*pos);
        if ((next_byte >> 6) != 2) {
            // 无效序列，中止
            code_points.push_back(0);
            return std::make_pair(std::move(code_points), llama_partial_utf8{ 0, -1 });
        }
        value = (value << 6) + (next_byte & 0x3F);
        ++pos;
        --n_remain;
    }

    if (partial_start.n_remain > 0 && n_remain == 0) {
        code_points.push_back(value);
    }

    // 解码任何后续的 utf-8 序列，可能以不完整的序列结尾
    // 当指针指向的值不为0时，进入循环
    while (*pos != 0) {
        // 将指针指向的值转换为无符号8位整数，作为第一个字节
        uint8_t  first_byte = static_cast<uint8_t>(*pos);
        // 取第一个字节的高4位作为索引，查找对应的剩余字节数
        uint8_t  highbits   = first_byte >> 4;
        n_remain   = lookup[highbits] - 1;

        // 如果剩余字节数小于0，表示序列无效，清空code_points并返回错误信息
        if (n_remain < 0) {
            code_points.clear();
            code_points.push_back(0);
            return std::make_pair(std::move(code_points), llama_partial_utf8{ 0, n_remain });
        }

        // 计算掩码，用于提取第一个字节中的值
        uint8_t  mask       = (1 << (7 - n_remain)) - 1;
        value      = first_byte & mask;
        ++pos;
        // 当指针指向的值不为0且剩余字节数大于0时，进入循环
        while (*pos != 0 && n_remain > 0) {
            // 将值左移6位，并加上下一个字节的后6位
            value = (value << 6) + (static_cast<uint8_t>(*pos) & 0x3F);
            ++pos;
            --n_remain;
        }
        // 如果剩余字节数为0，将值添加到code_points中
        if (n_remain == 0) {
            code_points.push_back(value);
        }
    }
    // 在code_points末尾添加0
    code_points.push_back(0);

    // 返回code_points和最后一个字符的值以及剩余字节数的pair
    return std::make_pair(std::move(code_points), llama_partial_utf8{ value, n_remain });
// 返回 true，如果 pos 指向规则定义的结尾之一
static bool llama_grammar_is_end_of_sequence(const llama_grammar_element * pos) {
    switch (pos->type) {
        case LLAMA_GRETYPE_END: return true;  // NOLINT
        case LLAMA_GRETYPE_ALT: return true;  // NOLINT
        default:                return false;
    }
}

// 返回 true，如果 chr 满足 pos 处的字符范围（正常或反向范围）
// 断言 pos 指向一个字符范围元素
static std::pair<bool, const llama_grammar_element *> llama_grammar_match_char(
        const llama_grammar_element * pos,
        const uint32_t                chr) {

    bool found            = false;
    bool is_positive_char = pos->type == LLAMA_GRETYPE_CHAR;

    GGML_ASSERT(is_positive_char || pos->type == LLAMA_GRETYPE_CHAR_NOT); // NOLINT

    do {
        if (pos[1].type == LLAMA_GRETYPE_CHAR_RNG_UPPER) {
            // 包含范围，例如 [a-z]
            found = found || (pos->value <= chr && chr <= pos[1].value);
            pos += 2;
        } else {
            // 精确字符匹配，例如 [a] 或 "a"
            found = found || pos->value == chr;
            pos += 1;
        }
    } while (pos->type == LLAMA_GRETYPE_CHAR_ALT);

    return std::make_pair(found == is_positive_char, pos);
}

// 返回 true，如果给定部分 UTF-8 序列的某个延续可以满足 pos 处的字符范围（正常或反向范围）
// 断言 pos 指向一个字符范围元素
static bool llama_grammar_match_partial_char(
        const llama_grammar_element * pos,
        const llama_partial_utf8      partial_utf8) {

    bool is_positive_char = pos->type == LLAMA_GRETYPE_CHAR;
    GGML_ASSERT(is_positive_char || pos->type == LLAMA_GRETYPE_CHAR_NOT);

    uint32_t partial_value = partial_utf8.value;
    int      n_remain      = partial_utf8.n_remain;

    // 无效序列或 7 位字符跨越 2 个字节（过长）
    // 如果剩余字节数小于0或者（剩余字节数等于1且部分值小于2），则返回false
    if (n_remain < 0 || (n_remain == 1 && partial_value < 2)) {
        return false;
    }

    // 可能的UTF-8序列完成的代码点范围
    uint32_t low  = partial_value << (n_remain * 6);
    uint32_t high = low | ((1 << (n_remain * 6)) - 1);

    // 如果low等于0
    if (low == 0) {
        // 如果剩余字节数为2，则low等于1左移11位
        if (n_remain == 2) {
            low = 1 << 11;
        } 
        // 如果剩余字节数为3，则low等于1左移16位
        else if (n_remain == 3) {
            low = 1 << 16;
        }
    }

    // 循环处理
    do {
        // 如果pos[1]的类型为LLAMA_GRETYPE_CHAR_RNG_UPPER
        if (pos[1].type == LLAMA_GRETYPE_CHAR_RNG_UPPER) {
            // 包含范围，例如[a-z]
            if (pos->value <= high && low <= pos[1].value) {
                return is_positive_char;
            }
            pos += 2;
        } 
        // 如果pos[1]的类型不为LLAMA_GRETYPE_CHAR_RNG_UPPER
        else {
            // 精确字符匹配，例如[a]或"a"
            if (low <= pos->value && pos->value <= high) {
                return is_positive_char;
            }
            pos += 1;
        }
    } while (pos->type == LLAMA_GRETYPE_CHAR_ALT);

    // 返回非正字符
    return !is_positive_char;
// 将语法下推栈转换为N个可能的栈，所有栈都以字符范围（终结元素）结尾
static void llama_grammar_advance_stack(
        const std::vector<std::vector<llama_grammar_element>>   & rules,  // 语法规则
        const std::vector<const llama_grammar_element *>        & stack,  // 下推栈
        std::vector<std::vector<const llama_grammar_element *>> & new_stacks) {  // 新栈的集合

    if (stack.empty()) {  // 如果下推栈为空
        new_stacks.emplace_back(stack);  // 将空栈加入新栈的集合
        return;  // 返回
    }

    const llama_grammar_element * pos = stack.back();  // 获取栈顶元素
    # 根据 pos->type 的值进行不同的操作
    switch (pos->type) {
        # 如果 pos->type 的值为 LLAMA_GRETYPE_RULE_REF
        case LLAMA_GRETYPE_RULE_REF: {
            # 将 pos->value 转换为 size_t 类型，作为规则的 ID
            const size_t                  rule_id = static_cast<size_t>(pos->value);
            # 获取规则的起始位置
            const llama_grammar_element * subpos  = rules[rule_id].data();
            # 循环处理规则引用
            do {
                # 初始化一个不包含顶部元素（pos）的新栈
                std::vector<const llama_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
                # 如果规则引用后面还有元素，则将其加入栈中
                if (!llama_grammar_is_end_of_sequence(pos + 1)) {
                    new_stack.push_back(pos + 1);
                }
                # 如果 alternate 不为空，则将其加入栈中
                if (!llama_grammar_is_end_of_sequence(subpos)) {
                    new_stack.push_back(subpos);
                }
                # 推进栈并生成新的栈
                llama_grammar_advance_stack(rules, new_stack, new_stacks);
                # 扫描到 alternate 定义的末尾
                while (!llama_grammar_is_end_of_sequence(subpos)) {
                    subpos++;
                }
                # 如果 subpos 的类型为 LLAMA_GRETYPE_ALT，则处理下一个 alternate 定义
                if (subpos->type == LLAMA_GRETYPE_ALT) {
                    subpos++;
                } else {
                    break;
                }
            } while (true);
            break;
        }
        # 如果 pos->type 的值为 LLAMA_GRETYPE_CHAR 或 LLAMA_GRETYPE_CHAR_NOT
        case LLAMA_GRETYPE_CHAR:
        case LLAMA_GRETYPE_CHAR_NOT:
            # 将当前栈加入到新栈集合中
            new_stacks.emplace_back(stack);
            break;
        # 如果 pos->type 的值为其他类型
        default:
            # 抛出断言错误
            GGML_ASSERT(false);
    }
// 以一个文法上的可能的推入栈集合为参数，这些栈需要定位在一个字符范围上（参见`llama_grammar_advance_stack`），并且在给定位置接受字符时，产生N个可能的栈
static std::vector<std::vector<const llama_grammar_element *>> llama_grammar_accept(
        const std::vector<std::vector<llama_grammar_element>>         & rules,  // 文法规则集合
        const std::vector<std::vector<const llama_grammar_element *>> & stacks,  // 可能的推入栈集合
        const uint32_t                                                  chr) {   // 给定的字符

    std::vector<std::vector<const llama_grammar_element *>> new_stacks;  // 新的栈集合

    for (const auto & stack : stacks) {  // 遍历推入栈集合
        if (stack.empty()) {  // 如果栈为空，则跳过
            continue;
        }

        auto match = llama_grammar_match_char(stack.back(), chr);  // 匹配栈顶元素和给定字符
        if (match.first) {  // 如果匹配成功
            const llama_grammar_element * pos = match.second;  // 获取匹配的元素

            // 更新栈顶元素到下一个元素（如果有的话）
            std::vector<const llama_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
            if (!llama_grammar_is_end_of_sequence(pos)) {  // 如果不是序列的结尾
                new_stack.push_back(pos);  // 将匹配的元素推入栈
            }
            llama_grammar_advance_stack(rules, new_stack, new_stacks);  // 推进栈
        }
    }

    return new_stacks;  // 返回新的栈集合
}

// 拒绝候选项
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates(
        const std::vector<std::vector<llama_grammar_element>>         & rules,  // 文法规则集合
        const std::vector<std::vector<const llama_grammar_element *>> & stacks,  // 可能的推入栈集合
        const std::vector<llama_grammar_candidate>                    & candidates);  // 候选项集合

// 为给定的栈拒绝候选项
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates_for_stack(
        const std::vector<std::vector<llama_grammar_element>> & rules,  // 文法规则集合
        const std::vector<const llama_grammar_element *>      & stack,  // 给定的栈
        const std::vector<llama_grammar_candidate>            & candidates) {  // 候选项集合

    std::vector<llama_grammar_candidate> rejects;  // 拒绝的候选项
    # 如果栈为空，则对候选项进行处理
    if (stack.empty()) {
        # 遍历候选项，将不符合条件的项加入到拒绝列表中
        for (const auto & tok : candidates) {
            if (*tok.code_points != 0 || tok.partial_utf8.n_remain != 0) {
                rejects.push_back(tok);
            }
        }
        # 返回拒绝列表
        return rejects;
    }

    # 获取栈顶元素
    const llama_grammar_element * stack_pos = stack.back();

    # 创建下一个候选项列表
    std::vector<llama_grammar_candidate> next_candidates;
    # 遍历候选项，根据条件将其加入到下一个候选项列表或拒绝列表中
    for (const auto & tok : candidates) {
        if (*tok.code_points == 0) {
            # 如果候选项已经到达完整的代码点末尾，则根据条件判断是否加入拒绝列表
            if (tok.partial_utf8.n_remain != 0 &&
                    !llama_grammar_match_partial_char(stack_pos, tok.partial_utf8)) {
                rejects.push_back(tok);
            }
        } else if (llama_grammar_match_char(stack_pos, *tok.code_points).first) {
            next_candidates.push_back({ tok.index, tok.code_points + 1, tok.partial_utf8 });
        } else {
            rejects.push_back(tok);
        }
    }

    # 获取栈顶元素的下一个位置
    const auto * stack_pos_after = llama_grammar_match_char(stack_pos, 0).second;

    # 更新栈顶元素到下一个元素，如果有的话
    std::vector<const llama_grammar_element *> stack_after(stack.begin(), stack.end() - 1);
    if (!llama_grammar_is_end_of_sequence(stack_pos_after)) {
        stack_after.push_back(stack_pos_after);
    }
    # 创建下一个栈列表
    std::vector<std::vector<const llama_grammar_element *>> next_stacks;
    llama_grammar_advance_stack(rules, stack_after, next_stacks);

    # 获取下一个拒绝列表
    auto next_rejects = llama_grammar_reject_candidates(rules, next_stacks, next_candidates);
    # 将下一个拒绝列表中的项加入到当前拒绝列表中
    for (const auto & tok : next_rejects) {
        rejects.push_back({ tok.index, tok.code_points - 1, tok.partial_utf8 });
    }

    # 返回拒绝列表
    return rejects;
// 定义一个静态函数，用于拒绝候选项
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates(
        // 输入参数：规则、堆栈和候选项
        const std::vector<std::vector<llama_grammar_element>>         & rules,
        const std::vector<std::vector<const llama_grammar_element *>> & stacks,
        const std::vector<llama_grammar_candidate>                    & candidates) {
    // 断言堆栈不为空
    GGML_ASSERT(!stacks.empty()); // REVIEW

    // 如果候选项为空，则返回空的候选项向量
    if (candidates.empty()) {
        return std::vector<llama_grammar_candidate>();
    }

    // 对堆栈中的每个元素进行拒绝操作
    auto rejects = llama_grammar_reject_candidates_for_stack(rules, stacks.front(), candidates);

    for (size_t i = 1, size = stacks.size(); i < size; ++i) {
        rejects = llama_grammar_reject_candidates_for_stack(rules, stacks[i], rejects);
    }
    // 返回拒绝后的候选项向量
    return rejects;
}

//
// grammar - external
//

// 初始化 LLAMA 语法
struct llama_grammar * llama_grammar_init(
            // 输入参数：规则、规则数量、起始规则索引
            const llama_grammar_element ** rules,
                                 size_t    n_rules,
                                 size_t    start_rule_index) {
    const llama_grammar_element * pos;

    // 将规则定义复制到向量中
    std::vector<std::vector<llama_grammar_element>> vec_rules(n_rules);
    for (size_t i = 0; i < n_rules; i++) {
        for (pos = rules[i]; pos->type != LLAMA_GRETYPE_END; pos++) {
            vec_rules[i].push_back(*pos);
        }
        vec_rules[i].push_back({LLAMA_GRETYPE_END, 0});
    }

    // 循环遍历起始规则的备选项，构建初始堆栈
    std::vector<std::vector<const llama_grammar_element *>> stacks;
    pos = rules[start_rule_index];
    // 创建一个 do-while 循环，用于处理语法规则的解析
    do {
        // 创建一个存储语法元素指针的向量
        std::vector<const llama_grammar_element *> stack;
        // 如果当前位置不是序列的结尾，则将其添加到栈中
        if (!llama_grammar_is_end_of_sequence(pos)) {
            stack.push_back(pos);
        }
        // 使用当前栈中的元素和规则向量，推进语法解析栈
        llama_grammar_advance_stack(vec_rules, stack, stacks);
        // 循环直到当前位置为序列的结尾
        while (!llama_grammar_is_end_of_sequence(pos)) {
            // 扫描到交替定义的结尾
            pos++;
        }
        // 如果当前位置的类型是交替定义，则继续处理下一个交替定义
        if (pos->type == LLAMA_GRETYPE_ALT) {
            pos++;
        } else {
            // 否则跳出循环
            break;
        }
    } while (true);

    // 返回一个新的 llama_grammar 对象，其中包含移动后的规则向量、栈和空的映射
    return new llama_grammar{ std::move(vec_rules), std::move(stacks), {} };
// 释放 llama_grammar 结构体指针所指向的内存
void llama_grammar_free(struct llama_grammar * grammar) {
    delete grammar;
}

// 复制 llama_grammar 结构体指针所指向的内容，并返回新的指针
struct llama_grammar * llama_grammar_copy(const struct llama_grammar * grammar) {
    // 使用 grammar 的 rules、stacks 和 partial_utf8 创建新的 llama_grammar 对象
    llama_grammar * result = new llama_grammar{ grammar->rules, grammar->stacks, grammar->partial_utf8 };

    // 将 stacks 中指向旧 rules 的指针重定向到新 rules
    for (size_t is = 0; is < result->stacks.size(); is++) {
        for (size_t ie = 0; ie < result->stacks[is].size(); ie++) {
            for (size_t ir0 = 0; ir0 < grammar->rules.size(); ir0++) {
                for (size_t ir1 = 0; ir1 < grammar->rules[ir0].size(); ir1++) {
                    if (grammar->stacks[is][ie] == &grammar->rules[ir0][ir1]) {
                         result->stacks[is][ie]  =  &result->rules[ir0][ir1];
                    }
                }
            }
        }
    }

    return result;
}

// 设置随机数生成器的种子
void llama_set_rng_seed(struct llama_context * ctx, uint32_t seed) {
    // 如果种子为默认值，则使用当前时间作为种子
    if (seed == LLAMA_DEFAULT_SEED) {
        seed = time(NULL);
    }
    ctx->rng.seed(seed);
}

// 对 softmax 进行采样
void llama_sample_softmax(struct llama_context * ctx, llama_token_data_array * candidates) {
    GGML_ASSERT(candidates->size > 0);

    const int64_t t_start_sample_us = ggml_time_us();

    // 将 logits 按降序排序
    if (!candidates->sorted) {
        std::sort(candidates->data, candidates->data + candidates->size, [](const llama_token_data & a, const llama_token_data & b) {
            return a.logit > b.logit;
        });
        candidates->sorted = true;
    }

    float max_l = candidates->data[0].logit;
    float cum_sum = 0.0f;
    for (size_t i = 0; i < candidates->size; ++i) {
        float p = expf(candidates->data[i].logit - max_l);
        candidates->data[i].p = p;
        cum_sum += p;
    }
    for (size_t i = 0; i < candidates->size; ++i) {
        candidates->data[i].p /= cum_sum;
    }

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}
// 从候选项中采样出前 k 个元素
void llama_sample_top_k(struct llama_context * ctx, llama_token_data_array * candidates, int k, size_t min_keep) {
    // 记录采样开始的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 确保 k 至少为 min_keep
    k = std::max(k, (int) min_keep);
    // 确保 k 最大不超过候选项的大小
    k = std::min(k, (int) candidates->size);

    // 将分数按降序排序
    if (!candidates->sorted) {
        // 定义比较函数
        auto comp = [](const llama_token_data & a, const llama_token_data & b) {
            return a.logit > b.logit;
        };
        // 如果 k 等于候选项的大小，则对所有元素进行排序
        if (k == (int) candidates->size) {
            std::sort(candidates->data, candidates->data + candidates->size, comp);
        } else {
            // 否则对前 k 个元素进行部分排序
            std::partial_sort(candidates->data, candidates->data + k, candidates->data + candidates->size, comp);
        }
        candidates->sorted = true;
    }
    // 更新候选项的大小为 k
    candidates->size = k;

    // 如果存在上下文对象，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 从候选项中按概率采样出元素
void llama_sample_top_p(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
    // 如果 p 大于等于 1.0，则直接返回
    if (p >= 1.0f) {
        return;
    }

    // 对候选项进行 softmax 处理
    llama_sample_softmax(ctx, candidates);

    // 记录采样开始的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 计算累积概率
    float cum_sum = 0.0f;
    size_t last_idx = candidates->size;

    for (size_t i = 0; i < candidates->size; ++i) {
        cum_sum += candidates->data[i].p;

        // 检查累积概率是否大于等于 p，或者是否至少保留了 min_keep 个元素
        // 如果是，则将最后的索引设置为 i+1，表示当前迭代应该包含在集合中
        if (cum_sum >= p && i + 1 >= min_keep) {
            last_idx = i + 1;
            break;
        }
    }

    // 调整输出向量的大小，仅保留前 p 个元素
    candidates->size = last_idx;

    // 如果存在上下文对象，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

void llama_sample_min_p(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
    // 如果概率 p 小于等于 0 或者候选项为空，则直接返回
    if (p <= 0.0f || !candidates->size) {
        return;
    }

    // 对候选项进行 softmax 操作
    llama_sample_softmax(ctx, candidates);

    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 通过最大概率来缩放，获取第一个 token 的概率
    float scale = candidates->data[0].p; // scale by max prob
    size_t i = 1; // first token always matches

    // 遍历候选项，找到概率小于 p*scale 且大于等于 min_keep 的 token
    for (; i < candidates->size; ++i) {
        if (candidates->data[i].p < p * scale && i >= min_keep) {
            break; // prob too small
        }
    }

    // 调整输出向量的大小，保留匹配的 token
    candidates->size = i;

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 如果 z 大于等于 1.0 或者候选项数组的大小小于等于 2，则直接返回
    if (z >= 1.0f || candidates->size <= 2) {
        return;
    }

    // 对候选项进行 softmax 处理
    llama_sample_softmax(nullptr, candidates);
    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 计算一阶导数和二阶导数
    std::vector<float> first_derivatives(candidates->size - 1);
    std::vector<float> second_derivatives(candidates->size - 2);

    // 计算一阶导数
    for (size_t i = 0; i < first_derivatives.size(); ++i) {
        first_derivatives[i] = candidates->data[i].p - candidates->data[i + 1].p;
    }
    // 计算二阶导数
    for (size_t i = 0; i < second_derivatives.size(); ++i) {
        second_derivatives[i] = first_derivatives[i] - first_derivatives[i + 1];
    }

    // 计算二阶导数的绝对值
    for (size_t i = 0; i < second_derivatives.size(); ++i) {
        second_derivatives[i] = std::abs(second_derivatives[i]);
    }

    // 对二阶导数进行归一化处理
    {
        const float second_derivatives_sum = std::accumulate(second_derivatives.begin(), second_derivatives.end(), 0.0f);

        if (second_derivatives_sum > 1e-6f) {
            for (float & value : second_derivatives) {
                value /= second_derivatives_sum;
            }
        } else {
            for (float & value : second_derivatives) {
                value = 1.0f / second_derivatives.size();
            }
        }
    }

    // 计算累积和，找到尾部位置
    float cum_sum = 0.0f;
    size_t last_idx = candidates->size;
    for (size_t i = 0; i < second_derivatives.size(); ++i) {
        cum_sum += second_derivatives[i];

        // 检查累积和是否大于 z 或者是否至少保留了 min_keep 个标记
        if (cum_sum > z && i >= min_keep) {
            last_idx = i;
            break;
        }
    }

    // 调整输出向量的大小，仅保留尾部位置以上的标记
    candidates->size = last_idx;
    # 如果上下文对象存在
    if (ctx) {
        # 更新上下文对象中的采样时间，加上当前时间减去开始采样时间
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 典型采样函数，根据给定的概率 p 和最小保留数量 min_keep 对候选项进行采样
    void llama_sample_typical(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
        // 如果概率 p 大于等于 1.0，直接返回，不进行采样
        if (p >= 1.0f) {
            return;
        }

        // 计算候选项的 softmax 值并计算熵
        llama_sample_softmax(nullptr, candidates);

        // 计算熵
        const int64_t t_start_sample_us = ggml_time_us();
        float entropy = 0.0f;
        for (size_t i = 0; i < candidates->size; ++i) {
            entropy += -candidates->data[i].p * logf(candidates->data[i].p);
        }

        // 计算负对数概率和熵之间的绝对差异
        std::vector<float> shifted_scores;
        for (size_t i = 0; i < candidates->size; ++i) {
            float shifted_score = fabsf(-logf(candidates->data[i].p) - entropy);
            shifted_scores.push_back(shifted_score);
        }

        // 基于 shifted_scores 和它们对应的索引对候选项进行排序
        std::vector<size_t> indices(candidates->size);
        std::iota(indices.begin(), indices.end(), 0);

        std::sort(indices.begin(), indices.end(), [&](size_t a, size_t b) {
            return shifted_scores[a] < shifted_scores[b];
        });

        // 计算累积概率
        float cum_sum = 0.0f;
        size_t last_idx = indices.size();

        for (size_t i = 0; i < indices.size(); ++i) {
            size_t idx = indices[i];
            cum_sum += candidates->data[idx].p;

            // 检查累积和是否大于给定概率 p，或者是否至少保留了 min_keep 个候选项
            if (cum_sum > p && i >= min_keep - 1) {
                last_idx = i + 1;
                break;
            }
        }

        // 调整输出向量的大小，仅保留局部典型的候选项
        std::vector<llama_token_data> new_candidates;
        for (size_t i = 0; i < last_idx; ++i) {
            size_t idx = indices[i];
            new_candidates.push_back(candidates->data[idx]);
        }
    }
    // 用 new_candidates 数据替换 candidates 中的数据
    std::copy(new_candidates.begin(), new_candidates.end(), candidates->data);
    // 更新 candidates 的大小为 new_candidates 的大小
    candidates->size = new_candidates.size();

    // 如果上下文对象存在
    if (ctx) {
        // 更新上下文对象的采样时间
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
// 定义一个函数，用于对候选项进行温度采样
void llama_sample_temp(struct llama_context * ctx, llama_token_data_array * candidates_p, float temp) {
    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 遍历候选项数组，将每个候选项的logit值除以温度
    for (size_t i = 0; i < candidates_p->size; ++i) {
        candidates_p->data[i].logit /= temp;
    }

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 定义一个函数，用于对候选项进行温度采样
void llama_sample_temperature(struct llama_context * ctx, llama_token_data_array * candidates_p, float temp) {
    // 调用llama_sample_temp函数对候选项进行温度采样
    llama_sample_temp(ctx, candidates_p, temp);
}

// 定义一个函数，用于对候选项应用重复惩罚
void llama_sample_repetition_penalties(
            struct llama_context * ctx,
            llama_token_data_array * candidates,
            const llama_token * last_tokens,
            size_t   penalty_last_n,
            float   penalty_repeat,
            float   penalty_freq,
            float   penalty_present) {
    // 如果不需要重复惩罚，则直接返回
    if (penalty_last_n == 0 || (penalty_repeat == 1.0f && penalty_freq == 0.0f && penalty_present == 0.0f)) {
        return;
    }

    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 创建一个频率映射，用于统计last_tokens中每个token的出现次数
    std::unordered_map<llama_token, int> token_count;
    for (size_t i = 0; i < penalty_last_n; ++i) {
        token_count[last_tokens[i]]++;
    }

    // 对候选项应用频率和存在性惩罚
    // 遍历候选项数组中的每个元素
    for (size_t i = 0; i < candidates->size; ++i) {
        // 查找当前候选项的 id 对应的计数
        const auto token_iter = token_count.find(candidates->data[i].id);
        // 如果找不到对应的计数，则跳过当前候选项
        if (token_iter == token_count.end()) {
            continue;
        }

        // 获取当前候选项的计数
        const int count = token_iter->second;

        // 如果当前候选项的 logit 小于等于 0，则乘以惩罚值
        // 否则，除以惩罚值
        if (candidates->data[i].logit <= 0) {
            candidates->data[i].logit *= penalty_repeat;
        } else {
            candidates->data[i].logit /= penalty_repeat;
        }

        // 减去计数乘以频率惩罚值和是否存在的惩罚值
        candidates->data[i].logit -= float(count) * penalty_freq + float(count > 0) * penalty_present;
    }

    // 将候选项数组标记为未排序状态
    candidates->sorted = false;

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
// 使用 LLAMA 上下文和候选项数组以及 LLAMA 语法来生成样本
void llama_sample_grammar(struct llama_context * ctx, llama_token_data_array * candidates, const struct llama_grammar * grammar) {
    GGML_ASSERT(ctx); // 断言上下文不为空
    const int64_t t_start_sample_us = ggml_time_us(); // 记录采样开始时间

    bool allow_eos = false; // 初始化是否允许结束符的标志为假
    for (const auto & stack : grammar->stacks) { // 遍历语法中的堆栈
        if (stack.empty()) { // 如果堆栈为空
            allow_eos = true; // 设置允许结束符的标志为真
            break; // 跳出循环
        }
    }

    const llama_token eos = llama_token_eos(&ctx->model); // 获取结束符

    std::vector<std::pair<std::vector<uint32_t>, llama_partial_utf8>> candidates_decoded; // 初始化解码后的候选项数组
    std::vector<llama_grammar_candidate> candidates_grammar; // 初始化 LLAMA 语法候选项数组

    for (size_t i = 0; i < candidates->size; ++i) { // 遍历候选项数组
        const llama_token id    = candidates->data[i].id; // 获取候选项的标识
        const std::string piece = llama_token_to_piece(ctx, id); // 获取候选项的字符串表示
        if (id == eos) { // 如果候选项是结束符
            if (!allow_eos) { // 如果不允许结束符
                candidates->data[i].logit = -INFINITY; // 将候选项的 logit 设置为负无穷
            }
        } else if (piece.empty() || piece[0] == 0) { // 如果候选项为空或者第一个字符为0
            candidates->data[i].logit = -INFINITY; // 将候选项的 logit 设置为负无穷
        } else {
            candidates_decoded.push_back(decode_utf8(piece.c_str(), grammar->partial_utf8)); // 解码候选项字符串
            candidates_grammar.push_back({ i, candidates_decoded.back().first.data(), candidates_decoded.back().second }); // 将解码后的候选项加入 LLAMA 语法候选项数组
        }
    }

    const auto rejects = llama_grammar_reject_candidates(grammar->rules, grammar->stacks, candidates_grammar); // 拒绝一些候选项
    for (const auto & reject : rejects) { // 遍历被拒绝的候选项
        candidates->data[reject.index].logit = -INFINITY; // 将被拒绝的候选项的 logit 设置为负无穷
    }

    ctx->t_sample_us += ggml_time_us() - t_start_sample_us; // 更新采样时间
}

// 对数组进行 log softmax 操作
static void llama_log_softmax(float * array, size_t size) {
    float max_l = *std::max_element(array, array + size); // 获取数组中的最大值
    float sum = 0.f; // 初始化求和变量
    for (size_t i = 0; i < size; ++i) { // 遍历数组
        float p = expf(array[i] - max_l); // 计算指数
        sum += p; // 更新求和变量
        array[i] = p; // 更新数组元素
    }

    for (size_t i = 0; i < size; ++i) { // 遍历数组
        array[i] = logf(array[i] / sum); // 计算 log softmax
    }
}
void llama_sample_classifier_free_guidance(
          struct llama_context * ctx,
        llama_token_data_array * candidates,
          struct llama_context * guidance_ctx,
                         float   scale) {
    int64_t t_start_sample_us = ggml_time_us();  // 获取当前时间，单位为微秒

    GGML_ASSERT(ctx);  // 断言上下文不为空

    auto n_vocab = llama_n_vocab(llama_get_model(ctx));  // 获取词汇表大小

    GGML_ASSERT(n_vocab == (int)candidates->size);  // 断言词汇表大小与候选项数组大小相等
    GGML_ASSERT(!candidates->sorted);  // 断言候选项数组未排序

    std::vector<float> logits_base;  // 创建存储基础对数概率的向量
    logits_base.reserve(candidates->size);  // 预留候选项数组大小的空间
    for (size_t i = 0; i < candidates->size; ++i) {  // 遍历候选项数组
        logits_base.push_back(candidates->data[i].logit);  // 将候选项的对数概率添加到向量中
    }
    llama_log_softmax(logits_base.data(), candidates->size);  // 对基础对数概率进行 softmax 处理

    float* logits_guidance = llama_get_logits(guidance_ctx);  // 获取指导上下文的对数概率
    llama_log_softmax(logits_guidance, n_vocab);  // 对指导上下文的对数概率进行 softmax 处理

    for (int i = 0; i < n_vocab; ++i) {  // 遍历词汇表
        float logit_guidance = logits_guidance[i];  // 获取指导对数概率
        float logit_base = logits_base[i];  // 获取基础对数概率
        candidates->data[i].logit = scale * (logit_base - logit_guidance) + logit_guidance;  // 计算并更新候选项的对数概率
    }

    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;  // 更新上下文的采样时间
    }
}

llama_token llama_sample_token_mirostat(struct llama_context * ctx, llama_token_data_array * candidates, float tau, float eta, int m, float * mu) {
    GGML_ASSERT(ctx);  // 断言上下文不为空

    auto N = float(llama_n_vocab(llama_get_model(ctx)));  // 获取词汇表大小
    int64_t t_start_sample_us;
    t_start_sample_us = ggml_time_us();  // 获取当前时间，单位为微秒

    llama_sample_softmax(nullptr, candidates);  // 对候选项进行 softmax 处理

    // 估算使用最可能的 m 个标记来估算 s_hat
    float s_hat = 0.0;
    float sum_ti_bi = 0.0;
    float sum_ti_sq = 0.0;
    for (size_t i = 0; i < size_t(m - 1) && i < candidates->size - 1; ++i) {  // 遍历候选项数组
        float t_i = logf(float(i + 2) / float(i + 1));  // 计算 t_i
        float b_i = logf(candidates->data[i].p / candidates->data[i + 1].p);  // 计算 b_i
        sum_ti_bi += t_i * b_i;  // 更新 sum_ti_bi
        sum_ti_sq += t_i * t_i;  // 更新 sum_ti_sq
    }
    s_hat = sum_ti_bi / sum_ti_sq;  // 计算 s_hat

    // 根据估算的 s_hat 和目标惊喜值计算 k
    // 计算 epsilon_hat，即 s_hat 减去 1 的结果
    float epsilon_hat = s_hat - 1;
    // 计算 k 值，使用 powf 函数计算幂次方
    float k = powf((epsilon_hat * powf(2, *mu)) / (1 - powf(N, -epsilon_hat)), 1 / s_hat);

    // 使用 top-k 抽样方法来抽样下一个单词 X
    llama_sample_top_k(nullptr, candidates, int(k), 1);
    // 如果上下文存在，则更新抽样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 从候选集中抽样出一个 token X
    llama_token X = llama_sample_token(ctx, candidates);
    // 重新记录抽样开始时间
    t_start_sample_us = ggml_time_us();

    // 计算观察到的惊喜值与目标惊喜值之间的差值作为误差
    // 首先找到 X 在候选集中的索引
    size_t X_idx = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return candidate.id == X;
    }));
    // 计算观察到的惊喜值
    float observed_surprise = -log2f(candidates->data[X_idx].p);
    // 计算误差 e
    float e = observed_surprise - tau;

    // 使用学习率和误差来更新 mu
    *mu = *mu - eta * e;

    // 如果上下文存在，则更新抽样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 返回抽样得到的 token X
    return X;
// 从给定的候选词数组中使用 Mirostat V2 算法进行抽样，返回抽样结果
llama_token llama_sample_token_mirostat_v2(struct llama_context * ctx, llama_token_data_array * candidates, float tau, float eta, float * mu) {
    // 记录抽样开始的时间
    int64_t t_start_sample_us;
    t_start_sample_us = ggml_time_us();

    // 使用 softmax 函数对候选词进行抽样
    llama_sample_softmax(ctx, candidates);

    // 截断概率大于 mu 的候选词
    candidates->size = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return -log2f(candidate.p) > *mu;
    }));

    // 如果没有符合条件的候选词，将候选词数量设为 1
    if (candidates->size == 0) {
        candidates->size = 1;
    }

    // 如果上下文存在，更新抽样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }

    // 对剩余候选词的概率进行归一化
    llama_sample_softmax(ctx, candidates);

    // 从剩余候选词中抽样出下一个词 X
    llama_token X = llama_sample_token(ctx, candidates);
    t_start_sample_us = ggml_time_us();

    // 计算观察到的意外性与目标意外性值之间的差异作为误差
    size_t X_idx = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return candidate.id == X;
    }));
    float observed_surprise = -log2f(candidates->data[X_idx].p);
    float e = observed_surprise - tau;

    // 使用学习率和误差更新 mu
    *mu = *mu - eta * e;

    // 如果上下文存在，更新抽样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 返回抽样结果 X
    return X;
}

// 从给定的候选词数组中使用贪婪算法进行抽样，返回抽样结果
llama_token llama_sample_token_greedy(struct llama_context * ctx, llama_token_data_array * candidates) {
    // 记录抽样开始的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 找到概率最大的元素
    auto * max_iter = std::max_element(candidates->data, candidates->data + candidates->size, [](const llama_token_data & a, const llama_token_data & b) {
        return a.logit < b.logit;
    });

    // 返回概率最大的候选词
    llama_token result = max_iter->id;
    # 如果上下文对象存在
    if (ctx) {
        # 更新上下文对象中的采样时间，加上当前时间减去开始采样时间
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
        # 增加采样次数
        ctx->n_sample++;
    }
    # 返回结果
    return result;
// 返回一个 token，用于样本选择
llama_token llama_sample_token(struct llama_context * ctx, llama_token_data_array * candidates) {
    // 断言上下文不为空
    GGML_ASSERT(ctx);

    // 记录开始样本采样的时间
    const int64_t t_start_sample_us = ggml_time_us();
    // 对候选 token 进行 softmax 处理
    llama_sample_softmax(nullptr, candidates);

    // 创建一个存储概率的向量，并预留空间
    std::vector<float> probs;
    probs.reserve(candidates->size);
    // 遍历候选 token，将概率存入向量中
    for (size_t i = 0; i < candidates->size; ++i) {
        probs.push_back(candidates->data[i].p);
    }

    // 创建离散分布对象，使用概率向量
    std::discrete_distribution<> dist(probs.begin(), probs.end());
    // 获取上下文的随机数生成器
    auto & rng = ctx->rng;
    // 从分布中抽取一个索引
    int idx = dist(rng);

    // 获取抽取到的 token
    llama_token result = candidates->data[idx].id;

    // 更新样本采样时间和样本数量
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    ctx->n_sample++;
    // 返回结果 token
    return result;
}

// 接受一个 token，并更新语法栈
void llama_grammar_accept_token(struct llama_context * ctx, struct llama_grammar * grammar, llama_token token) {
    // 记录开始接受 token 的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 如果 token 是句尾符号，则检查语法栈是否为空，如果为空则返回，否则断言失败
    if (token == llama_token_eos(&ctx->model)) {
        for (const auto & stack : grammar->stacks) {
            if (stack.empty()) {
                return;
            }
        }
        GGML_ASSERT(false);
    }

    // 将 token 转换为字符串
    const std::string piece = llama_token_to_piece(ctx, token);

    // 解码字符串，获取码点序列
    const auto   decoded     = decode_utf8(piece.c_str(), grammar->partial_utf8);
    const auto & code_points = decoded.first;
    // 遍历码点序列，更新语法栈
    for (auto it = code_points.begin(), end = code_points.end() - 1; it != end; ++it) {
        grammar->stacks = llama_grammar_accept(grammar->rules, grammar->stacks, *it);
    }
    // 更新部分 UTF-8 编码
    grammar->partial_utf8 = decoded.second;
    // 断言语法栈不为空
    GGML_ASSERT(!grammar->stacks.empty());

    // 更新接受 token 的时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

//
// Beam search
//

// 定义一个 llama_beam 结构体
struct llama_beam {
    std::vector<llama_token> tokens; // 存储 token 的向量
    float p;  // 累积概率（相对于所有 beam 进行重新归一化）
    bool eob; // 初始化是否到达 beam 结尾的标志为 false，回调函数将其设置为 true。
    // 按概率对 beam 进行排序。在概率相同时，优先选择 eob 的 beam。
    # 定义小于运算符重载函数，用于比较两个 llama_beam 对象的大小关系
    bool operator<(const llama_beam & rhs) const {
        # 返回一个布尔值，表示当前对象的 p 和 eob 组成的 pair 是否小于 rhs 对象的 p 和 eob 组成的 pair
        return std::make_pair(p, eob) < std::make_pair(rhs.p, rhs.eob);
    }
    # 移除前 n 个 tokens 并丢弃它们
    void shift_tokens(const size_t n) {
        # 如果 n 不为 0
        if (n) {
            # 将 tokens 中从第 n 个元素开始到末尾的元素复制到 tokens 的开头
            std::copy(tokens.begin() + n, tokens.end(), tokens.begin());
            # 调整 tokens 的大小，移除末尾的 n 个元素
            tokens.resize(tokens.size() - n);
        }
    }
    # 返回一个 llama_beam_view 对象，表示当前对象的 tokens 数据的视图
    llama_beam_view view() const { return {tokens.data(), tokens.size(), p, eob}; }
// 用于计算 logit 相关信息的结构体
struct llama_logit_info {
    // 指向logits的指针，指向常量的指针
    const float * const logits;
    // 词汇表的大小
    const int n_vocab;
    // logits中的最大值
    const float max_l;
    // 归一化因子
    const float normalizer;
    // 内部结构体，用于计算sum_exp
    struct sum_exp {
        float max_l;
        // 重载操作符，用于计算exp(l - max_l)的和
        float operator()(float sum, float l) const { return sum + std::exp(l - max_l); }
    };
    // 构造函数，初始化logits、n_vocab、max_l、normalizer
    llama_logit_info(llama_context * ctx)
      : logits(llama_get_logits(ctx))
      , n_vocab(llama_n_vocab(llama_get_model(ctx)))
      , max_l(*std::max_element(logits, logits + n_vocab))
      , normalizer(1.0f / std::accumulate(logits, logits + n_vocab, 0.0f, sum_exp{max_l}))
      { }
    // 获取token_id对应的token_data
    llama_token_data get_token_data(const llama_token token_id) const {
        // 常量p，未被使用
        constexpr auto p = std::numeric_limits<float>::quiet_NaN();  // never used
        return {token_id, logits[token_id], p};
    }
    // 返回按logit值排名的前k个token_data
    std::vector<llama_token_data> top_k(size_t k) {
        // 以logit值为基准的小顶堆
        std::vector<llama_token_data> min_heap;  // min-heap by logit
        const llama_token k_min = std::min(static_cast<llama_token>(k), n_vocab);
        min_heap.reserve(k_min);
        for (llama_token token_id = 0 ; token_id < k_min ; ++token_id) {
            min_heap.push_back(get_token_data(token_id));
        }
        auto comp = [](const llama_token_data & a, const llama_token_data & b) { return a.logit > b.logit; };
        std::make_heap(min_heap.begin(), min_heap.end(), comp);
        for (llama_token token_id = k_min ; token_id < n_vocab ; ++token_id) {
            if (min_heap.front().logit < logits[token_id]) {
                std::pop_heap(min_heap.begin(), min_heap.end(), comp);
                min_heap.back().id = token_id;
                min_heap.back().logit = logits[token_id];
                std::push_heap(min_heap.begin(), min_heap.end(), comp);
            }
        }
        return min_heap;
    }
    // 根据logit值计算概率
    float probability_from_logit(float logit) const {
        return normalizer * std::exp(logit - max_l);
    }
};
// 定义了一个名为 llama_beam_search_data 的结构体
struct llama_beam_search_data {
    llama_context * ctx; // 指向 llama_context 类型的指针
    size_t n_beams; // 表示束搜索中束的数量
    int n_past; // 表示过去的步数
    int n_predict; // 表示预测的步数
    std::vector<llama_beam> beams; // 存储 llama_beam 对象的向量
    std::vector<llama_beam> next_beams; // 存储下一个束搜索中 llama_beam 对象的向量

    // 在每次循环迭代中重新计算的变量
    size_t common_prefix_length; // 表示公共前缀的长度

    // 用于在束状态的回调中进行通信
    std::vector<llama_beam_view> beam_views; // 存储 llama_beam_view 对象的向量

    // 构造函数，初始化结构体的成员变量
    llama_beam_search_data(llama_context * ctx, size_t n_beams, int n_past, int n_predict)
      : ctx(ctx)
      , n_beams(n_beams)
      , n_past(n_past)
      , n_predict(n_predict)
      , beam_views(n_beams) {
        beams.reserve(n_beams); // 预留空间以存储 n_beams 个元素
        next_beams.reserve(n_beams); // 预留空间以存储 n_beams 个元素
    }

    // 将束折叠成由索引指定的单一束
    void collapse_beams(const size_t beam_idx) {
        if (0u < beam_idx) {
            std::swap(beams[0], beams[beam_idx]); // 交换 beams[0] 和 beams[beam_idx] 的值
        }
        beams.resize(1); // 将 beams 的大小调整为 1
    }

    // 使用最小堆来高效地收集前 k 个元素（k=n_beams）
    // 下面的重复模式反映了堆的两个阶段：
    //  * 收集元素直到向量已满，然后在其上调用 std::make_heap()
    //  * 如果堆已满并且找到应包含的新元素，则将最小元素弹出到 back()，用新元素替换它，然后将其推入堆中
    }

    // 基于束查找公共前缀长度
    // 需要确保 beams 不为空
    size_t find_common_prefix_length() {
        size_t common_prefix_length = beams[0].tokens.size(); // 初始化 common_prefix_length 为第一个束的 tokens 大小
        for (size_t i = 1 ; i < beams.size() ; ++i) {
            common_prefix_length = std::min(common_prefix_length, beams[i].tokens.size()); // 更新 common_prefix_length 为最小的 tokens 大小
            for (size_t j = 0 ; j < common_prefix_length ; ++j) {
                if (beams[0].tokens[j] != beams[i].tokens[j]) { // 检查每个束的 tokens 是否相等
                    common_prefix_length = j; // 更新 common_prefix_length
                    break;
                }
            }
        }
        return common_prefix_length; // 返回计算得到的 common_prefix_length
    }
};
    // 构造 beams_state 以通过回调函数发送回调函数的调用者
    // 副作用：设置 common_prefix_length = find_common_prefix_length();
    llama_beams_state get_beams_state(const bool last_call) {
        // 遍历 beams 数组，将每个 beam 转换为视图并存储在 beam_views 数组中
        for (size_t i = 0 ; i < beams.size() ; ++i) {
            beam_views[i] = beams[i].view();
        }
        // 调用 find_common_prefix_length() 函数，计算公共前缀长度并赋值给 common_prefix_length
        common_prefix_length = find_common_prefix_length();
        // 返回包含 beam 视图、beam 数量、公共前缀长度和是否最后一次调用的结构体
        return {beam_views.data(), beams.size(), common_prefix_length, last_call};
    }

    // 循环：
    //  * 当 i < n_predict 且
    //  * 任何一个 beam 尚未到达结束位置（eob） 且
    //  * 最高概率的 beam（如果有并列则为多个）尚未到达句子结束位置
    //    （因为所有其他 beam 的概率只会减小）
    // 循环函数，接受回调函数和回调数据作为参数
    void loop(const llama_beam_search_callback_fn_t callback, void * const callback_data) {
        // 将一个空的 beam 添加到 beams 中，概率为 1.0，eob 为 false
        beams.push_back({{}, 1.0f, false});
        // 定义一个 lambda 函数，用于检查 beam 是否为 eob
        const auto not_eob = [](const llama_beam & beam) { return !beam.eob; };
        // 循环执行直到达到预测次数或者所有 beam 都到达 eob
        for (int i = 0 ; i < n_predict && std::any_of(beams.begin(),beams.end(),not_eob) &&
                       !beams[top_beam_index()].eob ; ++i) {
            // 调用回调函数，获取 beam 的状态并设置 common_prefix_length
            callback(callback_data, get_beams_state(false));
            // 从 beam_views 更新 beam 的值（p，eob）
            update_beams_from_beam_views();
            // 如果 common_prefix_length 不为 0，则解码并更新 n_past
            if (common_prefix_length) {
                llama_decode(ctx, llama_batch_get_one(beams[0].tokens.data(), common_prefix_length, n_past, 0));
                n_past += common_prefix_length;
            }
            // 将 next_beam 的概率置为 0，以便在后续的最小堆中排在最后
            std::for_each(next_beams.begin(), next_beams.end(), [](llama_beam & beam) { beam.p = 0.0f; });
            // 遍历 beams，移动 tokens，填充 next_beams
            for (llama_beam & beam : beams) {
                beam.shift_tokens(common_prefix_length);
                fill_next_beams_by_top_probabilities(beam);
            }
            // next_beams 成为下一次迭代的 beams，交换它们以重用内存
            beams.swap(next_beams);
            // 重新计算 beam 的概率，避免浮点数下溢
            renormalize_beam_probabilities(beams);
        }
        // 将 top_beam_index 处的 beam 进行合并
        collapse_beams(top_beam_index());
        // 再次调用回调函数，获取最终的 beam 状态
        callback(callback_data, get_beams_state(true));
    }

    // 随着 beams 的增长，累积概率会减小，重新计算概率以避免浮点数下溢
    static void renormalize_beam_probabilities(std::vector<llama_beam> & beams) {
        // 定义一个 lambda 函数，用于计算概率的总和
        const auto sum_p = [](float sum, llama_beam & beam) { return sum + beam.p; };
        // 计算概率的总和，并计算其倒数
        const float inv_sum = 1.0f / std::accumulate(beams.begin(), beams.end(), 0.0f, sum_p);
        // 将每个 beam 的概率乘以倒数，重新计算概率
        std::for_each(beams.begin(), beams.end(), [=](llama_beam & beam) { beam.p *= inv_sum; });
    }
    # 假设 beams 是非空的。使用 llama_beam::operator<() 进行排序。
    size_t top_beam_index() {
        # 返回具有最大值的元素在 beams 中的索引
        return std::max_element(beams.begin(), beams.end()) - beams.begin();
    }
    
    # 为每个可能已被回调函数更改的 beam 复制 (p,eob)
    void update_beams_from_beam_views() {
        # 遍历 beams 数组，将每个 beam 的 (p,eob) 更新为对应的 beam_views 的 (p,eob)
        for (size_t i = 0 ; i < beams.size() ; ++i) {
            beams[i].p = beam_views[i].p;
            beams[i].eob = beam_views[i].eob;
        }
    }
};

void llama_beam_search(llama_context * ctx,
                       llama_beam_search_callback_fn_t callback, void * callback_data,
                       size_t n_beams, int n_past, int n_predict) {
    assert(ctx);
    const int64_t t_start_sample_us = ggml_time_us();

    llama_beam_search_data beam_search_data(ctx, n_beams, n_past, n_predict);

    beam_search_data.loop(callback, callback_data);

    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    ctx->n_sample++;
}

//
// quantization
//

template <typename T>
struct no_init {
    T value;
    no_init() { /* do nothing */ }
};

struct quantize_state_internal {
    const llama_model                 & model;  // 声明引用类型成员变量 model
    const llama_model_quantize_params * params;  // 声明指针类型成员变量 params

    int n_attention_wv    = 0;  // 初始化整型成员变量
    int n_feed_forward_w2 = 0;  // 初始化整型成员变量
    int i_attention_wv    = 0;  // 初始化整型成员变量
    int i_feed_forward_w2 = 0;  // 初始化整型成员变量

    int n_k_quantized     = 0;  // 初始化整型成员变量
    int n_fallback        = 0;  // 初始化整型成员变量

    quantize_state_internal(const llama_model & model, const llama_model_quantize_params * params)
        : model(model)  // 初始化引用类型成员变量
        , params(params)  // 初始化指针类型成员变量
        {}

};

static void llama_convert_tensor_internal(
    struct ggml_tensor * tensor, std::vector<no_init<float>> & output, std::vector<std::thread> & workers,
    const size_t nelements, const int nthread
) {
    if (output.size() < nelements) {  // 如果输出向量大小小于元素个数
        output.resize(nelements);  // 调整输出向量大小为元素个数
    }
    float * f32_output = (float *) output.data();  // 获取输出向量的数据指针

    ggml_type_traits_t qtype;  // 声明类型特征变量
    if (ggml_is_quantized(tensor->type)) {  // 如果张量类型是量化的
        qtype = ggml_internal_get_type_traits(tensor->type);  // 获取类型特征
        if (qtype.to_float == NULL) {  // 如果类型特征中的 to_float 函数指针为空
            throw std::runtime_error(format("type %s unsupported for integer quantization: no dequantization available", ggml_type_name(tensor->type)));  // 抛出运行时错误
        }
    } else if (tensor->type != GGML_TYPE_F16) {  // 如果张量类型不是 GGML_TYPE_F16
        throw std::runtime_error(format("cannot dequantize/convert tensor type %s", ggml_type_name(tensor->type)));  // 抛出运行时错误
    }
}
    // 如果线程数小于2，根据张量类型进行相应的数据处理
    if (nthread < 2) {
        // 如果张量类型为 GGML_TYPE_F16，将 ggml_fp16_t 类型的数据转换为 float 类型的数据
        if (tensor->type == GGML_TYPE_F16) {
            ggml_fp16_to_fp32_row((ggml_fp16_t *)tensor->data, f32_output, nelements);
        } 
        // 如果张量类型为量化类型，使用相应的量化类型对象进行数据转换
        else if (ggml_is_quantized(tensor->type)) {
            qtype.to_float(tensor->data, f32_output, nelements);
        } 
        // 如果张量类型不是上述两种类型，抛出断言错误
        else {
            GGML_ASSERT(false); // unreachable
        }
        return;
    }

    // 根据张量类型确定块大小
    auto block_size = tensor->type == GGML_TYPE_F16 ? 1 : (size_t)ggml_blck_size(tensor->type);
    // 根据张量类型确定块大小所占字节数
    auto block_size_bytes = ggml_type_size(tensor->type);

    // 断言元素个数能够被块大小整除
    GGML_ASSERT(nelements % block_size == 0);
    // 计算块的个数
    auto nblocks = nelements / block_size;
    // 计算每个线程处理的块数
    auto blocks_per_thread = nblocks / nthread;
    // 计算不能被线程数整除的剩余块数
    auto spare_blocks = nblocks - (blocks_per_thread * nthread); // if blocks aren't divisible by thread count

    // 遍历每个线程，为每个线程分配相应的任务
    for (auto tnum = 0, in_buff_offs = 0, out_buff_offs = 0; tnum < nthread; tnum++) {
        // 计算当前线程处理的块数，如果是最后一个线程，加上剩余块数
        auto thr_blocks = blocks_per_thread + (tnum == nthread - 1 ? spare_blocks : 0); // num blocks for this thread
        // 计算当前线程处理的元素个数
        auto thr_elems = thr_blocks * block_size; // number of elements for this thread
        // 计算当前线程处理的输入字节数
        auto thr_block_bytes = thr_blocks * block_size_bytes; // number of input bytes for this thread

        // 定义一个 lambda 函数，根据张量类型进行相应的数据处理
        auto compute = [qtype] (ggml_type typ, uint8_t * inbuf, float * outbuf, int nels) {
            if (typ == GGML_TYPE_F16) {
                ggml_fp16_to_fp32_row((ggml_fp16_t *)inbuf, outbuf, nels);
            } else {
                qtype.to_float(inbuf, outbuf, nels);
            }
        };
        // 将当前线程的任务加入到 workers 中
        workers.emplace_back(compute, tensor->type, (uint8_t *) tensor->data + in_buff_offs, f32_output + out_buff_offs, thr_elems);
        // 更新输入数据偏移量
        in_buff_offs += thr_block_bytes;
        // 更新输出数据偏移量
        out_buff_offs += thr_elems;
    }
    // 等待所有线程完成任务
    for (auto & w : workers) { w.join(); }
    // 清空 workers
    workers.clear();
    // 获取 K 量化类型
static ggml_type get_k_quant_type(
    // 获取量化状态内部对象的引用，新类型，张量指针，浮点类型
    quantize_state_internal & qs,
    ggml_type new_type, const ggml_tensor * tensor, llama_ftype ftype
) {
    // 获取张量的名称
    const std::string name = ggml_get_name(tensor);
    // TODO: 避免硬编码张量名称 - 使用 TN_* 常量
    const llm_arch arch = qs.model.arch;
    const auto       tn = LLM_TN(arch);

    // 使用更多位的函数
    auto use_more_bits = [](int i_layer, int num_layers) -> bool {
        return i_layer < num_layers/8 || i_layer >= 7*num_layers/8 || (i_layer - num_layers/8)%3 == 2;
    };

    // 如果张量名称为输出权重
    if (name == tn(LLM_TENSOR_OUTPUT, "weight").first) {
        // 获取张量的第一个维度
        int nx = tensor->ne[0];
        // 如果架构为 FALCON 或者 nx 不能被 QK_K 整除
        if (arch == LLM_ARCH_FALCON || nx % QK_K != 0) {
            // 新类型为 GGML_TYPE_Q8_0
            new_type = GGML_TYPE_Q8_0;
        }
        // 否则如果新类型不等于 GGML_TYPE_Q8_0
        else if (new_type != GGML_TYPE_Q8_0) {
            // 新类型为 GGML_TYPE_Q6_K
            new_type = GGML_TYPE_Q6_K;
        }
    } else if (name.find("attn_v.weight") != std::string::npos) {
        // 如果文件名中包含"attn_v.weight"
        if      (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) new_type = GGML_TYPE_Q3_K;
        // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q2_K，则新类型为GGML_TYPE_Q3_K
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M) {
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q3_K_M
            // 如果qs.i_attention_wv小于2，则新类型为GGML_TYPE_Q5_K，否则为GGML_TYPE_Q4_K
            new_type = qs.i_attention_wv < 2 ? GGML_TYPE_Q5_K : GGML_TYPE_Q4_K;
        }
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q5_K;
        // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q3_K_L，则新类型为GGML_TYPE_Q5_K
        else if ((ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M || ftype == LLAMA_FTYPE_MOSTLY_Q5_K_M) &&
                use_more_bits(qs.i_attention_wv, qs.n_attention_wv)) new_type = GGML_TYPE_Q6_K;
        // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q4_K_M或LLAMA_FTYPE_MOSTLY_Q5_K_M，并且使用更多位，则新类型为GGML_TYPE_Q6_K
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S && qs.i_attention_wv < 4) new_type = GGML_TYPE_Q5_K;
        // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q4_K_S并且qs.i_attention_wv小于4，则新类型为GGML_TYPE_Q5_K
        else if (QK_K == 64 && (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S || ftype == LLAMA_FTYPE_MOSTLY_Q3_K_S) &&
                (qs.i_attention_wv < qs.n_attention_wv/8 || qs.i_attention_wv >= 7*qs.n_attention_wv/8)) new_type = GGML_TYPE_Q6_K;
        // 如果QK_K等于64，并且文件类型为LLAMA_FTYPE_MOSTLY_Q4_K_S或LLAMA_FTYPE_MOSTLY_Q3_K_S，并且qs.i_attention_wv在指定范围内，则新类型为GGML_TYPE_Q6_K
        if (qs.model.type == MODEL_70B) {
            // 如果模型类型为MODEL_70B
            // 在70B模型中，有8个头共享相同的attn_v权重。因此，attn_v.weight张量比attn_q.weight小8倍。
            // 因此，通过使用更多位对该张量进行量化，可以在几乎不增加模型大小的情况下获得量化精度的显著提升
            if (new_type == GGML_TYPE_Q3_K || new_type == GGML_TYPE_Q4_K) new_type = GGML_TYPE_Q5_K;
            // 如果新类型为GGML_TYPE_Q3_K或GGML_TYPE_Q4_K，则新类型为GGML_TYPE_Q5_K
        }
        // 递增qs.i_attention_wv
        ++qs.i_attention_wv;
    } else if (name.find("ffn_down.weight") != std::string::npos) {
        // 如果文件名包含"ffn_down.weight"
        if      (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) new_type = GGML_TYPE_Q3_K;
        // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q2_K，设置新类型为GGML_TYPE_Q3_K
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M) {
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q3_K_M
            new_type = qs.i_feed_forward_w2 < 2 ? GGML_TYPE_Q5_K
                     : arch != LLM_ARCH_FALCON || use_more_bits(qs.i_feed_forward_w2, qs.n_feed_forward_w2) ? GGML_TYPE_Q4_K
                     : GGML_TYPE_Q3_K;
        }
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) {
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q3_K_L
            new_type = arch == LLM_ARCH_FALCON ? GGML_TYPE_Q4_K : GGML_TYPE_Q5_K;
        }
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M) {
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q4_K_M
            if (arch == LLM_ARCH_FALCON) {
                new_type = qs.i_feed_forward_w2 < 2 ? GGML_TYPE_Q6_K :
                           use_more_bits(qs.i_feed_forward_w2, qs.n_feed_forward_w2) ? GGML_TYPE_Q5_K : GGML_TYPE_Q4_K;
            } else {
                if (use_more_bits(qs.i_feed_forward_w2, qs.n_feed_forward_w2)) new_type = GGML_TYPE_Q6_K;
            }
        }
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q5_K_M && use_more_bits(qs.i_feed_forward_w2, qs.n_feed_forward_w2)) new_type = GGML_TYPE_Q6_K;
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S && arch != LLM_ARCH_FALCON && qs.i_feed_forward_w2 < 4) {
            new_type = GGML_TYPE_Q5_K;
        }
        // 增加qs.i_feed_forward_w2的值
        ++qs.i_feed_forward_w2;
    } else if (name.find("attn_output.weight") != std::string::npos) {
        // 如果文件名包含"attn_output.weight"
        if (arch != LLM_ARCH_FALCON) {
            // 如果架构不是LLM_ARCH_FALCON
            if      (ftype == LLAMA_FTYPE_MOSTLY_Q2_K  ) new_type = GGML_TYPE_Q3_K;
            else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M) new_type = GGML_TYPE_Q4_K;
            else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q5_K;
        } else {
            // 如果架构是LLM_ARCH_FALCON
            if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q4_K;
        }
    }
    // 如果文件名包含"attn_qkv.weight"
    else if (name.find("attn_qkv.weight") != std::string::npos) {
        // 如果文件类型是LLAMA_FTYPE_MOSTLY_Q3_K_M或LLAMA_FTYPE_MOSTLY_Q3_K_L，则设置新类型为GGML_TYPE_Q4_K
        if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M || ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q4_K;
        // 如果文件类型是LLAMA_FTYPE_MOSTLY_Q4_K_M，则设置新类型为GGML_TYPE_Q5_K
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M) new_type = GGML_TYPE_Q5_K;
        // 如果文件类型是LLAMA_FTYPE_MOSTLY_Q5_K_M，则设置新类型为GGML_TYPE_Q6_K
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q5_K_M) new_type = GGML_TYPE_Q6_K;
    }
    // 如果文件名包含"ffn_gate.weight"或"ffn_up.weight"
    else if (name.find("ffn_gate.weight") != std::string::npos || name.find("ffn_up.weight") != std::string::npos) {
        // 如果文件类型是LLAMA_FTYPE_MOSTLY_Q2_K，则设置新类型为GGML_TYPE_Q3_K
        if (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) new_type = GGML_TYPE_Q3_K;
    }
    // 如果文件名包含"fc1.weight"或"fc2.weight"
    else if (name.find("fc1.weight") != std::string::npos || name.find("fc2.weight") != std::string::npos) {
        // 如果文件类型是LLAMA_FTYPE_MOSTLY_Q4_0，则设置新类型为GGML_TYPE_Q4_0，否则设置新类型为GGML_TYPE_Q5_0
        if (ftype == LLAMA_FTYPE_MOSTLY_Q4_0) new_type = GGML_TYPE_Q4_0;
        else new_type = GGML_TYPE_Q5_0;
    }
    // 这段代码被注释掉，用于减小Q5_K_S模型的大小
    // 相关的PPL增加完全符合大小的减小
    //else {
    //    if (ftype == LLAMA_FTYPE_MOSTLY_Q5_K_S) new_type = GGML_TYPE_Q4_K;
    //}
    // 初始化一个变量，用于标记是否需要转换不兼容的张量
    bool convert_incompatible_tensor = false;
    // 如果新类型是GGML_TYPE_Q2_K、GGML_TYPE_Q3_K、GGML_TYPE_Q4_K、GGML_TYPE_Q5_K或GGML_TYPE_Q6_K
    if (new_type == GGML_TYPE_Q2_K || new_type == GGML_TYPE_Q3_K || new_type == GGML_TYPE_Q4_K ||
        new_type == GGML_TYPE_Q5_K || new_type == GGML_TYPE_Q6_K) {
        // 获取张量的行数和列数
        int nx = tensor->ne[0];
        int ny = tensor->ne[1];
        // 如果行数不能被QK_K整除
        if (nx % QK_K != 0) {
            // 输出警告信息，标记需要转换不兼容的张量
            LLAMA_LOG_WARN("\n\n%s : tensor cols %d x %d are not divisible by %d, required for %s", __func__, nx, ny, QK_K, ggml_type_name(new_type));
            convert_incompatible_tensor = true;
        } else {
            // 否则，增加量化张量的数量
            ++qs.n_k_quantized;
        }
    }
    # 如果需要转换不兼容的张量
    if (convert_incompatible_tensor) {
        # 根据新类型进行不兼容张量的转换
        switch (new_type) {
            case GGML_TYPE_Q2_K: new_type = GGML_TYPE_Q4_0; break;
            case GGML_TYPE_Q3_K: new_type = GGML_TYPE_Q4_1; break;
            case GGML_TYPE_Q4_K: new_type = GGML_TYPE_Q5_0; break;
            case GGML_TYPE_Q5_K: new_type = GGML_TYPE_Q5_1; break;
            case GGML_TYPE_Q6_K: new_type = GGML_TYPE_Q8_0; break;
            # 如果遇到不支持的张量大小，抛出运行时错误
            default: throw std::runtime_error("\nUnsupported tensor size encountered\n");
        }
        # 记录使用回退量化的警告信息
        LLAMA_LOG_WARN(" - using fallback quantization %s\n", ggml_type_name(new_type));
        # 增加回退量化的计数
        ++qs.n_fallback;
    }

    # 返回新的类型
    return new_type;
}

static void llama_model_quantize_internal(const std::string & fname_inp, const std::string & fname_out, const llama_model_quantize_params * params) {
    ggml_type quantized_type; // 定义变量 quantized_type，用于存储量化类型
    llama_ftype ftype = params->ftype; // 从参数中获取文件类型

    switch (params->ftype) { // 根据文件类型进行不同的处理
        case LLAMA_FTYPE_MOSTLY_Q4_0: quantized_type = GGML_TYPE_Q4_0; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q4_0，则设置 quantized_type 为 GGML_TYPE_Q4_0
        case LLAMA_FTYPE_MOSTLY_Q4_1: quantized_type = GGML_TYPE_Q4_1; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q4_1，则设置 quantized_type 为 GGML_TYPE_Q4_1
        case LLAMA_FTYPE_MOSTLY_Q5_0: quantized_type = GGML_TYPE_Q5_0; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q5_0，则设置 quantized_type 为 GGML_TYPE_Q5_0
        case LLAMA_FTYPE_MOSTLY_Q5_1: quantized_type = GGML_TYPE_Q5_1; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q5_1，则设置 quantized_type 为 GGML_TYPE_Q5_1
        case LLAMA_FTYPE_MOSTLY_Q8_0: quantized_type = GGML_TYPE_Q8_0; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q8_0，则设置 quantized_type 为 GGML_TYPE_Q8_0
        case LLAMA_FTYPE_MOSTLY_F16:  quantized_type = GGML_TYPE_F16;  break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_F16，则设置 quantized_type 为 GGML_TYPE_F16
        case LLAMA_FTYPE_ALL_F32:     quantized_type = GGML_TYPE_F32;  break; // 如果文件类型为 LLAMA_FTYPE_ALL_F32，则设置 quantized_type 为 GGML_TYPE_F32

        // K-quants
        case LLAMA_FTYPE_MOSTLY_Q2_K:   quantized_type = GGML_TYPE_Q2_K; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q2_K，则设置 quantized_type 为 GGML_TYPE_Q2_K
        case LLAMA_FTYPE_MOSTLY_Q3_K_S:
        case LLAMA_FTYPE_MOSTLY_Q3_K_M:
        case LLAMA_FTYPE_MOSTLY_Q3_K_L: quantized_type = GGML_TYPE_Q3_K; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q3_K_S 或 LLAMA_FTYPE_MOSTLY_Q3_K_M 或 LLAMA_FTYPE_MOSTLY_Q3_K_L，则设置 quantized_type 为 GGML_TYPE_Q3_K
        case LLAMA_FTYPE_MOSTLY_Q4_K_S:
        case LLAMA_FTYPE_MOSTLY_Q4_K_M: quantized_type = GGML_TYPE_Q4_K; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q4_K_S 或 LLAMA_FTYPE_MOSTLY_Q4_K_M，则设置 quantized_type 为 GGML_TYPE_Q4_K
        case LLAMA_FTYPE_MOSTLY_Q5_K_S:
        case LLAMA_FTYPE_MOSTLY_Q5_K_M: quantized_type = GGML_TYPE_Q5_K; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q5_K_S 或 LLAMA_FTYPE_MOSTLY_Q5_K_M，则设置 quantized_type 为 GGML_TYPE_Q5_K
        case LLAMA_FTYPE_MOSTLY_Q6_K:   quantized_type = GGML_TYPE_Q6_K; break; // 如果文件类型为 LLAMA_FTYPE_MOSTLY_Q6_K，则设置 quantized_type 为 GGML_TYPE_Q6_K

        default: throw std::runtime_error(format("invalid output file type %d\n", ftype)); // 如果文件类型不在上述范围内，则抛出运行时错误
    }

    int nthread = params->nthread; // 从参数中获取线程数

    if (nthread <= 0) { // 如果线程数小于等于0
        nthread = std::thread::hardware_concurrency(); // 获取硬件支持的线程数
    }

    // mmap consistently increases speed Linux, and also increases speed on Windows with
    // hot cache. It may cause a slowdown on macOS, possibly related to free memory.
#if defined(__linux__) || defined(_WIN32)
    constexpr bool use_mmap = true; // 如果是 Linux 或者 Windows，则使用 mmap
#else
    constexpr bool use_mmap = false; // 否则不使用 mmap
#endif

    llama_model_loader ml(fname_inp, use_mmap); // 使用 mmap 加载模型
    // 如果使用内存映射，则重置映射对象
    if (ml.use_mmap) {
        ml.mapping.reset(new llama_mmap(&ml.file, /* prefetch */ 0, ggml_is_numa()));
    }

    // 创建 llama_model 对象
    llama_model model;
    // 加载架构参数到 model
    llm_load_arch(ml, model);
    // 加载超参数到 model
    llm_load_hparams(ml, model);

    // 创建 quantize_state_internal 对象 qs，传入 model 和 params
    struct quantize_state_internal qs(model, params);

    // 如果参数中只有复制操作，则设置 ftype 为 model.ftype
    if (params->only_copy) {
        ftype = model.ftype;
    }

    // 设置对齐大小为 GGUF_DEFAULT_ALIGNMENT
    const size_t align = GGUF_DEFAULT_ALIGNMENT;
    // 根据 ml.sparse_deriv 的值初始化 gguf_context 对象 ctx_out
    struct gguf_context * ctx_out = ml.sparse_deriv == GGML_SPARSE_INFERENCE ? gguf_init_empty_sparse() : gguf_init_empty();

    // 复制输入文件中的 KV 对
    gguf_set_kv     (ctx_out, ml.ctx_gguf);
    gguf_set_val_u32(ctx_out, "general.quantization_version", GGML_QNT_VERSION);
    gguf_set_val_u32(ctx_out, "general.file_type", ftype);

    // 遍历 ml.n_tensors 个 tensor
    for (int i = 0; i < ml.n_tensors; ++i) {
        // 获取第 i 个 tensor 的元数据
        struct ggml_tensor * meta = ml.get_tensor_meta(i);

        // 获取 tensor 的名称
        const std::string name = ggml_get_name(meta);

        // 如果名称中包含指定字符串，则增加对应的计数器
        // TODO: 避免硬编码的 tensor 名称 - 使用 TN_* 常量
        if (name.find("attn_v.weight") != std::string::npos || name.find("attn_qkv.weight") != std::string::npos) {
            ++qs.n_attention_wv;
        }
        else if (name.find("ffn_down.weight") != std::string::npos) {
            ++qs.n_feed_forward_w2;
        }
    }
    // 如果注意力权重计数不等于前馈权重计数，或者注意力权重计数不等于模型的层数，则输出警告信息
    if (qs.n_attention_wv != qs.n_feed_forward_w2 || (uint32_t)qs.n_attention_wv != model.hparams.n_layer) {
        LLAMA_LOG_WARN("%s ============ Strange model: n_attention_wv = %d, n_feed_forward_w2 = %d, hparams.n_layer = %d\n",
                __func__, qs.n_attention_wv, qs.n_feed_forward_w2, model.hparams.n_layer);
    }

    // 初始化变量和容器
    size_t total_size_org = 0;
    size_t total_size_new = 0;
    std::vector<int64_t> hist_all(1 << 4, 0);
    std::vector<std::thread> workers;
    workers.reserve(nthread);
    std::mutex mutex;
    int idx = 0;
    std::vector<no_init<uint8_t>> read_data;
    std::vector<no_init<uint8_t>> work;
    std::vector<no_init<float>> f32_conv_buf;

    // 填充原始张量，以便获得初始元数据
    // 遍历 ml 结构体中的张量数量
    for (int i = 0; i < ml.n_tensors; ++i) {
        // 获取第 i 个张量的元数据
        struct ggml_tensor * meta = ml.get_tensor_meta(i);
        // 将获取到的张量元数据添加到 ctx_out 上下文中
        gguf_add_tensor(ctx_out, meta);
    }

    // 以二进制写入模式打开文件流
    std::ofstream fout(fname_out, std::ios::binary);
    // 设置文件流在写入错误时立即抛出异常
    fout.exceptions(std::ofstream::failbit); // fail fast on write errors

    // 获取上下文中元数据的大小
    const size_t meta_size = gguf_get_meta_size(ctx_out);

    // 打印函数名和元数据大小
    LLAMA_LOG_INFO("%s: meta size = %zu bytes\n", __func__, meta_size);

    // 在文件流中创建一个大小为 meta_size 的零填充占位符
    ::zeros(fout, meta_size);

    }

    // 回到文件开头并写入更新后的元数据
    {
        fout.seekp(0);
        // 创建一个大小为 ctx_out 中元数据大小的字节数组
        std::vector<uint8_t> data(gguf_get_meta_size(ctx_out));
        // 获取上下文中的元数据并写入到 data 数组中
        gguf_get_meta_data(ctx_out, data.data());
        // 将 data 数组中的数据写入到文件流中
        fout.write((const char *) data.data(), data.size());
    }

    // 关闭文件流
    fout.close();

    // 释放上下文资源
    gguf_free(ctx_out);

    // 打印函数名和原始模型大小（以 MB 为单位）
    LLAMA_LOG_INFO("%s: model size  = %8.2f MB\n", __func__, total_size_org/1024.0/1024.0);
    // 打印函数名和量化后模型大小（以 MB 为单位）
    LLAMA_LOG_INFO("%s: quant size  = %8.2f MB\n", __func__, total_size_new/1024.0/1024.0);

    // 打印所有张量的直方图
    {
        // 计算所有直方图值的总和
        int64_t sum_all = 0;
        for (size_t i = 0; i < hist_all.size(); i++) {
            sum_all += hist_all[i];
        }

        // 如果总和大于 0，则打印直方图
        if (sum_all > 0) {
            LLAMA_LOG_INFO("%s: hist: ", __func__);
            for (size_t i = 0; i < hist_all.size(); i++) {
                // 打印每个直方图值的比例
                LLAMA_LOG_INFO("%5.3f ", hist_all[i] / float(sum_all));
            }
            LLAMA_LOG_INFO("\n");
        }
    }

    // 如果存在回退量化的张量，则打印警告信息
    if (qs.n_fallback > 0) {
        LLAMA_LOG_WARN("%s: WARNING: %d of %d tensor(s) incompatible with k-quants and required fallback quantization\n",
                __func__, qs.n_fallback, qs.n_k_quantized + qs.n_fallback);
    }
}

static int llama_apply_lora_from_file_internal(
    const struct llama_model & model, const char * path_lora, float scale, const char * path_base_model, int n_threads
) {
    // 输出信息，表示正在应用 lora 适配器
    LLAMA_LOG_INFO("%s: applying lora adapter from '%s' - please wait ...\n", __func__, path_lora);

    // 记录开始应用 lora 适配器的时间
    const int64_t t_start_lora_us = ggml_time_us();

    // 打开 lora 文件
    auto fin = std::ifstream(path_lora, std::ios::binary);
    // 如果打开失败，则输出错误信息并返回 1
    if (!fin) {
        LLAMA_LOG_ERROR("%s: failed to open '%s'\n", __func__, path_lora);
        return 1;
    }

    // 验证文件的魔数和版本号
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        uint32_t format_version;
        fin.read((char *) &format_version, sizeof(format_version));

        // 如果版本号不为 1，则输出错误信息并返回 1
        if (format_version != 1) {
            LLAMA_LOG_ERROR("%s: unsupported file version\n", __func__ );
            return 1;
        }
    }

    // 读取 lora_r 和 lora_alpha，并计算 scaling
    int32_t lora_r;
    int32_t lora_alpha;
    fin.read((char *) &lora_r, sizeof(lora_r));
    fin.read((char *) &lora_alpha, sizeof(lora_alpha));
    float scaling = scale * (float)lora_alpha / (float)lora_r;

    // 输出 lora_r、lora_alpha 和 scaling 的信息
    LLAMA_LOG_INFO("%s: r = %d, alpha = %d, scaling = %.2f\n", __func__, lora_r, lora_alpha, scaling);

    // 创建一个临时的 ggml 上下文来存储 lora 张量
    // todo: 从可能的最大张量计算大小
    std::vector<uint8_t> lora_buf(1024ull * 1024ull * 1024ull);
    struct ggml_init_params params;
    params.mem_size   = lora_buf.size();
    params.mem_buffer = lora_buf.data();
    params.no_alloc   = false;

    ggml_context * lora_ctx = ggml_init(params);
    std::unordered_map<std::string, struct ggml_tensor *> lora_tensors;

    // 创建一个模型名称到张量的映射，以加速查找
    std::unordered_map<std::string, struct ggml_tensor*> model_tensors;
    for (const auto & kv : model.tensors_by_name) {
        model_tensors.insert(kv);
    }

    // 加载基础模型
    std::unique_ptr<llama_model_loader> ml;
    ggml_context * base_ctx = NULL;
    // 创建一个存储 uint8_t 类型数据的向量 base_buf
    std::vector<uint8_t> base_buf;
    // 如果存在基础模型路径
    if (path_base_model) {
        // 输出加载基础模型的信息
        LLAMA_LOG_INFO("%s: loading base model from '%s'\n", __func__, path_base_model);
        // 通过 llama_model_loader 加载基础模型，并使用内存映射
        ml.reset(new llama_model_loader(path_base_model, /*use_mmap*/ true));

        // 计算上下文大小和内存映射大小
        size_t ctx_size;
        size_t mmapped_size;
        ml->calc_sizes(ctx_size, mmapped_size);
        // 调整 base_buf 的大小以适应上下文大小
        base_buf.resize(ctx_size);

        // 初始化基础参数
        ggml_init_params base_params;
        base_params.mem_size   = base_buf.size();
        base_params.mem_buffer = base_buf.data();
        base_params.no_alloc   = ml->use_mmap;

        // 初始化基础上下文
        base_ctx = ggml_init(base_params);

        // 如果使用内存映射，则在 llama_model_loader 中进行映射
        if (ml->use_mmap) {
            ml->mapping.reset(new llama_mmap(&ml->file, /* prefetch */ 0, ggml_is_numa()));
        }
    }

    // 读取张量并应用
    bool warned = false;
    int n_tensors = 0;

    // 创建一个存储 uint8_t 类型数据的向量 work_buffer
    std::vector<uint8_t> work_buffer;
#ifdef GGML_USE_CUBLAS
            // 如果使用了 CUBLAS，则进行以下操作
            if (dest_t->backend == GGML_BACKEND_GPU || dest_t->backend == GGML_BACKEND_GPU_SPLIT) {
                // 如果目标张量的后端是 GPU 或者 GPU_SPLIT
                if (dest_t->type != GGML_TYPE_F16) {
                    // 如果目标张量的类型不是 F16，则抛出运行时错误
                    throw std::runtime_error(format(
                        "%s: error: the simultaneous use of LoRAs and GPU acceleration is only supported for f16 models. dest_t->type: %d", __func__, dest_t->type));
                }
                // 设置 offload_func 为 ggml_cuda_assign_buffers
                offload_func = ggml_cuda_assign_buffers;
                // 设置 offload_func_force_inplace 为 ggml_cuda_assign_buffers_force_inplace
                offload_func_force_inplace = ggml_cuda_assign_buffers_force_inplace;
            }
    }

    // TODO: this should be in a destructor, it will leak on failure
    // 释放 lora_ctx 指向的内存
    ggml_free(lora_ctx);
    // 如果 base_ctx 不为空
    if (base_ctx) {
        // 释放 base_ctx 指向的内存
        ggml_free(base_ctx);
    }

    // 计算 t_lora_us，表示 LoRAs 操作的耗时
    const int64_t t_lora_us = ggml_time_us() - t_start_lora_us;
    // 记录 t_lora_us 的值，表示 LoRAs 操作完成
    LLAMA_LOG_INFO(" done (%.2f ms)\n", t_lora_us / 1000.0);

    // 返回 0，表示操作成功
    return 0;
}

//
// interface implementation
//
// 返回默认的模型参数
struct llama_model_params llama_model_default_params() {
    // 初始化 result 结构体
    struct llama_model_params result = {
        /*.n_gpu_layers                =*/ 0,
        /*.main_gpu                    =*/ 0,
        /*.vram_budget_gb              =*/ -1.0,
        /*.tensor_split                =*/ nullptr,
        /*.progress_callback           =*/ nullptr,
        /*.progress_callback_user_data =*/ nullptr,
        /*.vocab_only                  =*/ false,
        /*.use_mmap                    =*/ true,
        /*.use_mlock                   =*/ false,
    };

#ifdef GGML_USE_METAL
    // 如果使用了 METAL，则设置 n_gpu_layers 为 1
    result.n_gpu_layers = 1;
#endif

    // 返回 result 结构体
    return result;
}

// 返回默认的上下文参数
struct llama_context_params llama_context_default_params() {
    // 定义一个结构体 llama_context_params 的实例 result，并初始化其各个字段的数值
    struct llama_context_params result = {
        /*.seed                        =*/ LLAMA_DEFAULT_SEED,  // 设置种子为默认值
        /*.n_ctx                       =*/ 512,  // 设置上下文数为 512
        /*.n_batch                     =*/ 512,  // 设置批处理数为 512
        /*.n_threads                   =*/ GGML_DEFAULT_N_THREADS, // 设置线程数为默认线程数
        /*.n_threads_batch             =*/ GGML_DEFAULT_N_THREADS,  // 设置批处理线程数为默认线程数
        /*.rope_scaling_type           =*/ LLAMA_ROPE_SCALING_UNSPECIFIED,  // 设置绳索缩放类型为未指定
        /*.rope_freq_base              =*/ 0.0f,  // 设置绳索频率基数为 0.0
        /*.rope_freq_scale             =*/ 0.0f,  // 设置绳索频率缩放为 0.0
        /*.yarn_ext_factor             =*/ -1.0f,  // 设置纱线扩展因子为 -1.0
        /*.yarn_attn_factor            =*/ 1.0f,  // 设置纱线注意力因子为 1.0
        /*.yarn_beta_fast              =*/ 32.0f,  // 设置纱线快速 beta 为 32.0
        /*.yarn_beta_slow              =*/ 1.0f,  // 设置纱线慢速 beta 为 1.0
        /*.yarn_orig_ctx               =*/ 0,  // 设置纱线原始上下文为 0
        /*.mul_mat_q                   =*/ true,  // 设置是否乘以矩阵 q 为 true
        /*.f16_kv                      =*/ true,  // 设置是否使用 f16 kv 为 true
        /*.logits_all                  =*/ false,  // 设置是否记录所有 logits 为 false
        /*.embedding                   =*/ false,  // 设置是否嵌入为 false
    };

    // 返回初始化后的结构体实例 result
    return result;
// 返回默认的量化参数结构体
struct llama_model_quantize_params llama_model_quantize_default_params() {
    // 初始化默认的量化参数结构体
    struct llama_model_quantize_params result = {
        /*.nthread                     =*/ 0,  // 线程数
        /*.ftype                       =*/ LLAMA_FTYPE_MOSTLY_Q5_1,  // 文件类型
        /*.allow_requantize            =*/ false,  // 是否允许重新量化
        /*.quantize_output_tensor      =*/ true,  // 是否量化输出张量
        /*.only_copy                   =*/ false,  // 是否仅复制
        /*.pure                        =*/ false,  // 是否纯净
    };

    return result;  // 返回初始化后的结构体
}

// 返回最大设备数
int llama_max_devices(void) {
    return LLAMA_MAX_DEVICES;  // 返回最大设备数
}

// 返回是否支持内存映射
bool llama_mmap_supported(void) {
    return llama_mmap::SUPPORTED;  // 返回内存映射是否支持
}

// 返回是否支持内存锁定
bool llama_mlock_supported(void) {
    return llama_mlock::SUPPORTED;  // 返回内存锁定是否支持
}

// 初始化后端
void llama_backend_init(bool numa) {
    ggml_time_init();  // 初始化时间

    // 需要初始化 f16 表
    {
        struct ggml_init_params params = { 0, NULL, false };
        struct ggml_context * ctx = ggml_init(params);
        ggml_free(ctx);
    }

    if (numa) {
        ggml_numa_init();  // 如果需要 NUMA 初始化
    }

#ifdef GGML_USE_MPI
    ggml_mpi_backend_init();  // 如果使用 MPI，则初始化 MPI 后端
#endif
}

// 释放后端资源
void llama_backend_free(void) {
#ifdef GGML_USE_MPI
    ggml_mpi_backend_free();  // 如果使用 MPI，则释放 MPI 后端资源
#endif
}

// 返回当前时间的微秒数
int64_t llama_time_us(void) {
    return ggml_time_us();  // 返回当前时间的微秒数
}

// 从文件加载模型
struct llama_model * llama_load_model_from_file(
                             const char * path_model,
              struct llama_model_params   params) {
    ggml_time_init();  // 初始化时间

    llama_model * model = new llama_model;  // 创建模型对象

    unsigned cur_percentage = 0;  // 当前百分比
    # 如果进度回调函数为空，则设置进度回调函数和用户数据
    if (params.progress_callback == NULL) {
        params.progress_callback_user_data = &cur_percentage;
        # 设置进度回调函数的匿名函数，用于更新进度
        params.progress_callback = [](float progress, void * ctx) {
            unsigned * cur_percentage_p = (unsigned *) ctx;
            unsigned percentage = (unsigned) (100 * progress);
            # 当进度更新时，打印进度信息
            while (percentage > *cur_percentage_p) {
                *cur_percentage_p = percentage;
                LLAMA_LOG_INFO(".");
                # 当进度达到100%时，打印换行符
                if (percentage >= 100) {
                    LLAMA_LOG_INFO("\n");
                }
            }
        };
    }

    # 如果模型加载失败，则打印错误信息并返回空指针
    if (!llama_model_load(path_model, *model, params)) {
        LLAMA_LOG_ERROR("%s: failed to load model\n", __func__);
        delete model;
        return nullptr;
    }

    # 返回加载的模型
    return model;
// 释放模型内存的函数
void llama_free_model(struct llama_model * model) {
    delete model;
}

// 创建带有模型的新上下文
struct llama_context * llama_new_context_with_model(
                 struct llama_model * model,
        struct llama_context_params   params) {

    // 如果模型为空，则返回空指针
    if (!model) {
        return nullptr;
    }

    // 使用模型创建上下文对象
    llama_context * ctx = new llama_context(*model);

    // 获取模型的超参数和上下文的参数
    const auto & hparams = model->hparams;
    auto       & cparams = ctx->cparams;

    // 设置上下文参数
    cparams.n_batch          = params.n_batch;
    cparams.n_threads        = params.n_threads;
    cparams.n_threads_batch  = params.n_threads_batch;
    cparams.yarn_ext_factor  = params.yarn_ext_factor;
    cparams.yarn_attn_factor = params.yarn_attn_factor;
    cparams.yarn_beta_fast   = params.yarn_beta_fast;
    cparams.yarn_beta_slow   = params.yarn_beta_slow;
    cparams.mul_mat_q        = params.mul_mat_q;

    // 设置上下文参数的默认值
    cparams.n_ctx            = params.n_ctx           == 0    ? hparams.n_ctx_train           : params.n_ctx;
    cparams.rope_freq_base   = params.rope_freq_base  == 0.0f ? hparams.rope_freq_base_train  : params.rope_freq_base;
    cparams.rope_freq_scale  = params.rope_freq_scale == 0.0f ? hparams.rope_freq_scale_train : params.rope_freq_scale;

    // 设置原始上下文数
    cparams.n_yarn_orig_ctx  = params.yarn_orig_ctx    != 0 ? params.yarn_orig_ctx    :
                               hparams.n_yarn_orig_ctx != 0 ? hparams.n_yarn_orig_ctx :
                                                              hparams.n_ctx_train;

    // 设置绳索缩放类型
    auto rope_scaling_type = params.rope_scaling_type;
    if (rope_scaling_type == LLAMA_ROPE_SCALING_UNSPECIFIED) {
        rope_scaling_type = hparams.rope_scaling_type_train;
    }

    // 如果绳索缩放类型为无，则绳索频率缩放为1.0
    if (rope_scaling_type == LLAMA_ROPE_SCALING_NONE) {
        cparams.rope_freq_scale = 1.0f; // 如果缩放类型为无，则永远不缩放
    }

    // 如果绳索扩展因子小于0.0，则根据缩放类型设置默认值
    if (cparams.yarn_ext_factor < 0.0f) { // 负值表示“未设置”
        cparams.yarn_ext_factor = rope_scaling_type == LLAMA_ROPE_SCALING_YARN ? 1.0f : 0.0f;
    }
}
    # 如果参数中的种子值等于默认种子值，则将种子值设置为当前时间
    if (params.seed == LLAMA_DEFAULT_SEED) {
        params.seed = time(NULL);
    }

    # 记录上下文信息的日志
    LLAMA_LOG_INFO("%s: n_ctx      = %u\n",     __func__, cparams.n_ctx);
    LLAMA_LOG_INFO("%s: freq_base  = %.1f\n",   __func__, cparams.rope_freq_base);
    LLAMA_LOG_INFO("%s: freq_scale = %g\n",     __func__, cparams.rope_freq_scale);

    # 使用种子值初始化随机数生成器
    ctx->rng = std::mt19937(params.seed);
    # 将参数中的logits_all值赋给上下文对象的logits_all属性
    ctx->logits_all = params.logits_all;

    # 根据参数中的f16_kv值确定内存类型
    ggml_type memory_type = params.f16_kv ? GGML_TYPE_F16 : GGML_TYPE_F32;

    # 为上下文缓冲区预留内存空间
    // reserve memory for context buffers
    // 如果不仅仅是词汇表，则进行以下操作
    if (!hparams.vocab_only) {
        // 初始化自注意力缓存
        if (!llama_kv_cache_init(ctx->model.hparams, ctx->kv_self, memory_type, cparams.n_ctx, model->n_gpu_layers)) {
            // 如果初始化失败，则记录错误信息，释放内存，返回空指针
            LLAMA_LOG_ERROR("%s: llama_kv_cache_init() failed for self-attention cache\n", __func__);
            llama_free(ctx);
            return nullptr;
        }

        {
            // 计算自注意力缓存的大小并记录日志
            const size_t memory_size = ggml_nbytes(ctx->kv_self.k) + ggml_nbytes(ctx->kv_self.v);
            LLAMA_LOG_INFO("%s: kv self size  = %7.2f MB\n", __func__, memory_size / 1024.0 / 1024.0);
        }

        // 在推断期间调整大小
        if (params.logits_all) {
            // 如果需要记录所有的logits，则预留足够的空间
            ctx->logits.reserve(cparams.n_ctx*hparams.n_vocab);
        } else {
            // 否则只预留词汇表大小的空间
            ctx->logits.reserve(hparams.n_vocab);
        }

        if (params.embedding){
            // 如果需要嵌入，则调整嵌入的大小
            ctx->embedding.resize(hparams.n_embd);
        }

        {
            static const size_t tensor_alignment = 32;
            // 计算计算缓冲区的大小，并创建度量分配器
            ctx->buf_compute.resize(ggml_tensor_overhead()*LLAMA_MAX_NODES + ggml_graph_overhead());

            // 创建度量分配器
            ctx->alloc = ggml_allocr_new_measure(tensor_alignment);

            // 构建最坏情况下的图
            int n_tokens = (int)std::min(cparams.n_ctx, cparams.n_batch);
            int n_past = cparams.n_ctx - n_tokens;
            llama_token token = llama_token_bos(&ctx->model); // 实际上 llama_build_graph 不会使用，但在选择 token 和 embedding 输入图时需要
            ggml_cgraph * gf = llama_build_graph(*ctx, llama_batch_get_one(&token, n_tokens, n_past, 0));
        }
#ifdef GGML_USE_METAL
            // 如果使用 Metal，则执行以下操作
            if (model->n_gpu_layers > 0) {
                // 设置 Metal 日志回调函数
                ggml_metal_log_set_callback(llama_log_callback_default, NULL);

                // 初始化 Metal 上下文
                ctx->ctx_metal = ggml_metal_init(1);
                // 如果初始化失败，则记录错误信息并释放内存
                if (!ctx->ctx_metal) {
                    LLAMA_LOG_ERROR("%s: ggml_metal_init() failed\n", __func__);
                    llama_free(ctx);
                    return NULL;
                }
                // 计算图形的内存需求
                //ggml_metal_graph_find_concurrency(ctx->ctx_metal, gf, false);
                //ggml_allocr_set_parse_seq(ctx->alloc, ggml_metal_get_concur_list(ctx->ctx_metal), ggml_metal_if_optimized(ctx->ctx_metal));
            }
#endif
            // 计算图形的内存需求
            size_t alloc_size = ggml_allocr_alloc_graph(ctx->alloc, gf) + tensor_alignment;

            // 记录计算缓冲区的总大小（以 MB 为单位）
            LLAMA_LOG_INFO("%s: compute buffer total size = %.2f MB\n", __func__, (ctx->buf_compute.size + alloc_size) / 1024.0 / 1024.0);

            // 重新创建具有精确内存需求的分配器
            ggml_allocr_free(ctx->alloc);

            // 调整分配器的大小以匹配内存需求
            ctx->buf_alloc.resize(alloc_size);
            ctx->alloc = ggml_allocr_new(ctx->buf_alloc.data, ctx->buf_alloc.size, tensor_alignment);
#ifdef GGML_USE_METAL
            // 如果使用 Metal，则执行以下操作
            if (ctx->ctx_metal) {
                //ggml_allocr_set_parse_seq(ctx->alloc, ggml_metal_get_concur_list(ctx->ctx_metal), ggml_metal_if_optimized(ctx->ctx_metal));
            }
#endif
#ifdef GGML_USE_CUBLAS
            // 如果使用了CUBLAS，则设置VRAM缓冲区大小，并输出日志信息
            ggml_cuda_set_scratch_size(alloc_size);
            LLAMA_LOG_INFO("%s: VRAM scratch buffer: %.2f MB\n", __func__, alloc_size / 1024.0 / 1024.0);

            // 计算总的VRAM使用量
            auto add_tensor = [](const ggml_tensor * t, size_t & size) {
                // 如果张量的后端是GPU或GPU_SPLIT，则计算其占用的内存大小
                if (t->backend == GGML_BACKEND_GPU || t->backend == GGML_BACKEND_GPU_SPLIT) {
                    size += ggml_nbytes(t);
                }
            };
            size_t model_vram_size = 0;
            // 遍历模型中的张量，计算模型的VRAM使用量
            for (const auto & kv : model->tensors_by_name) {
                add_tensor(kv.second, model_vram_size);
            }

            size_t kv_vram_size = 0;
            // 计算键值对自身张量的VRAM使用量
            add_tensor(ctx->kv_self.k, kv_vram_size);
            add_tensor(ctx->kv_self.v, kv_vram_size);

            // 计算上下文的VRAM使用量
            size_t ctx_vram_size = alloc_size + kv_vram_size;
            // 计算总的VRAM使用量
            size_t total_vram_size = model_vram_size + ctx_vram_size;

            // 输出总的VRAM使用情况的日志信息
            LLAMA_LOG_INFO("%s: total VRAM used: %.2f MB (model: %.2f MB, context: %.2f MB)\n", __func__,
                    total_vram_size / 1024.0 / 1024.0,
                    model_vram_size / 1024.0 / 1024.0,
                    ctx_vram_size / 1024.0 / 1024.0);
#endif
        }

#ifdef GGML_USE_METAL
        if (model->n_gpu_layers > 0) {
            // 分配所有Metal资源和内存缓冲区

            void * data_ptr  = NULL;
            size_t data_size = 0;

            // 如果模型的映射存在，则使用映射的地址和大小
            if (ctx->model.mapping) {
                data_ptr  = ctx->model.mapping->addr;
                data_size = ctx->model.mapping->size;
            } else {
                // 否则，使用ggml_get_mem_buffer和ggml_get_mem_size获取内存缓冲区的地址和大小
                data_ptr  = ggml_get_mem_buffer(ctx->model.ctx);
                data_size = ggml_get_mem_size  (ctx->model.ctx);
            }

            // 获取模型上下文中张量的最大大小
            const size_t max_size = ggml_get_max_tensor_size(ctx->model.ctx);

            // 输出最大张量大小的日志信息
            LLAMA_LOG_INFO("%s: max tensor size = %8.2f MB\n", __func__, max_size/1024.0/1024.0);
#define LLAMA_METAL_CHECK_BUF(result)                            \
            // 定义宏，用于检查 Metal 缓冲区是否添加成功，如果失败则输出错误信息，释放上下文并返回空指针
            if (!(result)) {                                             \
                // 如果结果为假，输出错误信息并释放上下文，返回空指针
                LLAMA_LOG_ERROR("%s: failed to add buffer\n", __func__); \
                // 输出错误信息
                llama_free(ctx);                                         \
                // 释放上下文
                return NULL;                                             \
                // 返回空指针
            }

            // 检查并添加名为 "data" 的缓冲区
            LLAMA_METAL_CHECK_BUF(ggml_metal_add_buffer(ctx->ctx_metal, "data",  data_ptr, data_size, max_size));
            // 检查并添加名为 "kv" 的缓冲区
            LLAMA_METAL_CHECK_BUF(ggml_metal_add_buffer(ctx->ctx_metal, "kv",    ctx->kv_self.buf.data, ctx->kv_self.buf.size, 0));
            // 检查并添加名为 "alloc" 的缓冲区
            LLAMA_METAL_CHECK_BUF(ggml_metal_add_buffer(ctx->ctx_metal, "alloc", ctx->buf_alloc.data, ctx->buf_alloc.size, 0));
            // 取消宏定义 LLAMA_METAL_CHECK_BUF
#undef LLAMA_METAL_CHECK_BUF
        }
#endif
    }

#ifdef GGML_USE_MPI
    // 初始化 MPI 上下文
    ctx->ctx_mpi = ggml_mpi_init();

    // 如果 MPI 的 rank 大于 0
    if (ggml_mpi_rank(ctx->ctx_mpi) > 0) {
        // 进入一个阻塞的评估循环，使用虚拟输入，让 rank=0 驱动进程
        // TODO: 需要在 #3228 修复之后进行修复
        GGML_ASSERT(false && "not implemented");
        // 输出错误信息并退出程序
        //const std::vector<llama_token> tmp(ctx->model.hparams.n_ctx, llama_token_bos(ctx));
        //while (!llama_eval(ctx, tmp.data(), tmp.size(), 0, 0)) {};
        llama_backend_free();
        exit(1);
    }
#endif

    // 返回上下文
    return ctx;
}

// 释放上下文
void llama_free(struct llama_context * ctx) {
    delete ctx;
}

// 获取模型
const llama_model * llama_get_model(const struct llama_context * ctx) {
    return &ctx->model;
}

// 获取上下文的 n_ctx
int llama_n_ctx(const struct llama_context * ctx) {
    return ctx->cparams.n_ctx;
}

// 获取模型的词汇表类型
enum llama_vocab_type llama_vocab_type(const struct llama_model * model) {
    return model->vocab.type;
}

// 检查模型是否使用稀疏推断
bool llama_use_sparse_inference(const struct llama_model * model) {
    return model->sparse_deriv == GGML_SPARSE_INFERENCE;
}

// 获取模型的词汇表大小
int llama_n_vocab(const struct llama_model * model) {
    return model->vocab.id_to_token.size();
}

// 获取模型的训练上下文大小
int llama_n_ctx_train(const struct llama_model * model) {
    # 返回模型的超参数中的训练上下文长度
    return model->hparams.n_ctx_train;
// 返回模型的嵌入维度
int llama_n_embd(const struct llama_model * model) {
    return model->hparams.n_embd;
}

// 返回模型的绳子频率训练比例
float llama_rope_freq_scale_train(const struct llama_model * model) {
    return model->hparams.rope_freq_scale_train;
}

// 返回模型的描述信息
int llama_model_desc(const struct llama_model * model, char * buf, size_t buf_size) {
    return snprintf(buf, buf_size, "%s %s %s",
            llama_model_arch_name(model->arch).c_str(),
            llama_model_type_name(model->type),
            llama_model_ftype_name(model->ftype).c_str());
}

// 返回模型的大小
uint64_t llama_model_size(const struct llama_model * model) {
    uint64_t size = 0;
    for (const auto & it : model->tensors_by_name) {
        size += ggml_nbytes(it.second);
    }
    return size;
}

// 返回模型的参数数量
uint64_t llama_model_n_params(const struct llama_model * model) {
    uint64_t nparams = 0;
    for (const auto & it : model->tensors_by_name) {
        nparams += ggml_nelements(it.second);
    }
    return nparams;
}

// 获取模型的张量
struct ggml_tensor * llama_get_model_tensor(struct llama_model * model, const char * name) {
    return ggml_get_tensor(model->ctx, name);
}

// 对模型进行量化
int llama_model_quantize(
        const char * fname_inp,
        const char * fname_out,
        const llama_model_quantize_params * params) {
    try {
        llama_model_quantize_internal(fname_inp, fname_out, params);
        return 0;
    } catch (const std::exception & err) {
        LLAMA_LOG_ERROR("%s: failed to quantize: %s\n", __func__, err.what());
        return 1;
    }
}

// 从文件应用 LORA
int llama_apply_lora_from_file(struct llama_context * ctx, const char * path_lora, float scale, const char * path_base_model, int n_threads) {
    try {
        return llama_apply_lora_from_file_internal(ctx->model, path_lora, scale, path_base_model, n_threads);
    } catch (const std::exception & err) {
        LLAMA_LOG_ERROR("%s: failed to apply lora adapter: %s\n", __func__, err.what());
        return 1;
    }
}
// 从文件应用 LORA 模型，返回结果
int llama_model_apply_lora_from_file(const struct llama_model * model, const char * path_lora, float scale, const char * path_base_model, int n_threads) {
    try {
        // 调用内部函数应用 LORA 模型
        return llama_apply_lora_from_file_internal(*model, path_lora, scale, path_base_model, n_threads);
    } catch (const std::exception & err) {
        // 捕获异常并记录错误信息
        LLAMA_LOG_ERROR("%s: failed to apply lora adapter: %s\n", __func__, err.what());
        return 1;
    }
}

// 从文件应用 GPU 索引到模型
int llama_model_apply_gpu_idx_from_file(struct llama_model * model, const char * path_mlp, bool use_mmap) {
    // 创建 GPU 分割加载器对象
    llama_gpu_split_loader * mlp_ml = new llama_gpu_split_loader(path_mlp, use_mmap);
    // 将张量应用到基础模型，并检查是否成功
    if (mlp_ml -> apply_tensors_to_base_model(model) > 0) {
        // 记录错误信息
        LLAMA_LOG_ERROR("%s: failed to apply gpu split\n", __func__);
        return 1;
    }
    // 设置模型的 MLP 模型加载器
    model -> mlp_model_loader = std::unique_ptr<llama_gpu_split_loader>(mlp_ml);
    return 0;
}

// 卸载 FFN 分割模型
size_t llama_model_offload_ffn_split(struct llama_model * model) {
    // 创建增强模型加载器对象
    llama_augmentation_model_loader * aug_ml = new llama_augmentation_model_loader(model);    
    // 卸载 FFN 分割模型并返回卸载的字节数
    size_t offloaded_bytes = aug_ml->offload_ffn_split(model);
    return offloaded_bytes;
}

// 获取 KV 缓存中的令牌数量
int llama_get_kv_cache_token_count(const struct llama_context * ctx) {
    return ctx->kv_self.head;
}

// 清空 KV 缓存
void llama_kv_cache_clear(struct llama_context * ctx) {
    llama_kv_cache_clear(ctx->kv_self);
}

// 从 KV 缓存中删除序列
void llama_kv_cache_seq_rm(struct llama_context * ctx, llama_seq_id seq_id, llama_pos p0, llama_pos p1) {
    llama_kv_cache_seq_rm(ctx->kv_self, seq_id, p0, p1);
}

// 从 KV 缓存中复制序列
void llama_kv_cache_seq_cp(struct llama_context * ctx, llama_seq_id seq_id_src, llama_seq_id seq_id_dst, llama_pos p0, llama_pos p1) {
    // 如果源序列和目标序列相同，则直接返回
    if (seq_id_src == seq_id_dst) {
        return;
    }
    // 从 KV 缓存中复制序列
    llama_kv_cache_seq_cp(ctx->kv_self, seq_id_src, seq_id_dst, p0, p1);
}

// 从 KV 缓存中保留序列
void llama_kv_cache_seq_keep(struct llama_context * ctx, llama_seq_id seq_id) {
    llama_kv_cache_seq_keep(ctx->kv_self, seq_id);
}
// 将指定序列 ID 的键值对缓存中的数据进行平移
void llama_kv_cache_seq_shift(struct llama_context * ctx, llama_seq_id seq_id, llama_pos p0, llama_pos p1, llama_pos delta) {
    llama_kv_cache_seq_shift(ctx->kv_self, seq_id, p0, p1, delta);
}

// 返回状态的最大大小
size_t llama_get_state_size(const struct llama_context * ctx) {
    // 我们不知道随机数生成器的大小，直到实际序列化它。因此，为其序列化状态保留比足够大的内存。
    // 作为参考，std::mt19937(1337) 序列化为 6701 字节。
    const size_t s_rng_size        = sizeof(size_t);
    const size_t s_rng             = LLAMA_MAX_RNG_STATE;
    const size_t s_logits_capacity = sizeof(size_t);
    const size_t s_logits_size     = sizeof(size_t);
    const size_t s_logits          = ctx->logits.capacity() * sizeof(float);
    const size_t s_embedding_size  = sizeof(size_t);
    const size_t s_embedding       = ctx->embedding.size() * sizeof(float);
    const size_t s_kv_size         = sizeof(size_t);
    const size_t s_kv_ntok         = sizeof(int);
    const size_t s_kv              = ctx->kv_self.buf.size;

    // 计算总大小
    const size_t s_total = (
        + s_rng_size
        + s_rng
        + s_logits_capacity
        + s_logits_size
        + s_logits
        + s_embedding_size
        + s_embedding
        + s_kv_size
        + s_kv_ntok
        + s_kv
    );

    return s_total;
}

// llama_context_data
struct llama_data_context {
    // 写入数据的虚拟函数
    virtual void write(const void * src, size_t size) = 0;
    // 获取已写入数据的大小的虚拟函数
    virtual size_t get_size_written() = 0;
    // 默认析构函数
    virtual ~llama_data_context() = default;
};

// 用于数据缓冲区的上下文结构
struct llama_data_buffer_context : llama_data_context {
    uint8_t * ptr;
    size_t size_written = 0;

    // 构造函数，初始化指针
    llama_data_buffer_context(uint8_t * p) : ptr(p) {}

    // 写入数据的虚拟函数的实现
    void write(const void * src, size_t size) override {
        memcpy(ptr, src, size);
        ptr += size;
        size_written += size;
    }

    // 获取已写入数据的大小的虚拟函数的实现
    size_t get_size_written() override {
        return size_written;
    }
};

// 用于文件的上下文结构
struct llama_data_file_context : llama_data_context {
    # 声明一个指向llama_file类型的指针变量file
    llama_file * file;
    # 声明一个size_t类型的变量size_written，并初始化为0
    size_t size_written = 0;

    # 定义llama_data_file_context类的构造函数，参数为llama_file类型的指针f
    llama_data_file_context(llama_file * f) : file(f) {}

    # 重写write方法，接受void类型指针src和size_t类型的size作为参数
    void write(const void * src, size_t size) override {
        # 调用file对象的write_raw方法，将src指向的数据写入文件
        file->write_raw(src, size);
        # 更新size_written变量，累加写入的数据大小
        size_written += size;
    }

    # 重写get_size_written方法
    size_t get_size_written() override {
        # 返回size_written变量的值
        return size_written;
    }
static void llama_copy_state_data_internal(struct llama_context * ctx, llama_data_context * data_ctx) {
    // 将状态数据复制到传入的上下文中，可以是缓冲区或文件

    // 复制随机数生成器状态数据
    {
        // 创建一个字符串流对象
        std::stringstream rng_ss;
        // 将随机数生成器状态数据写入字符串流
        rng_ss << ctx->rng;

        // 获取字符串流的大小
        const size_t rng_size = rng_ss.str().size();
        // 创建一个缓冲区用于存储随机数生成器状态数据
        char rng_buf[LLAMA_MAX_RNG_STATE];

        // 将缓冲区清零
        memset(&rng_buf[0], 0, LLAMA_MAX_RNG_STATE);
        // 将字符串流中的数据复制到缓冲区
        memcpy(&rng_buf[0], rng_ss.str().data(), rng_ss.str().size());

        // 将随机数生成器状态数据的大小写入上下文
        data_ctx->write(&rng_size,   sizeof(rng_size));
        // 将随机数生成器状态数据写入上下文
        data_ctx->write(&rng_buf[0], LLAMA_MAX_RNG_STATE);
    }

    // 复制logits数据
    {
        // 获取logits的容量和大小
        const size_t logits_cap  = ctx->logits.capacity();
        const size_t logits_size = ctx->logits.size();

        // 将logits的容量和大小写入上下文
        data_ctx->write(&logits_cap,  sizeof(logits_cap));
        data_ctx->write(&logits_size, sizeof(logits_size);

        // 如果logits的大小不为0，则将logits数据写入上下文
        if (logits_size) {
            data_ctx->write(ctx->logits.data(), logits_size * sizeof(float));
        }

        // 如果大小和容量之间有间隙，则写入填充数据
        size_t padding_size = (logits_cap - logits_size) * sizeof(float);
        if (padding_size > 0) {
            // 创建一个填充数据的缓冲区，填充为0
            std::vector<uint8_t> padding(padding_size, 0);
            // 将填充数据写入上下文
            data_ctx->write(padding.data(), padding_size);
        }
    }

    // 复制嵌入数据
    {
        // 获取嵌入数据的大小
        const size_t embedding_size = ctx->embedding.size();

        // 将嵌入数据的大小写入上下文
        data_ctx->write(&embedding_size, sizeof(embedding_size);

        // 如果嵌入数据的大小不为0，则将嵌入数据写入上下文
        if (embedding_size) {
            data_ctx->write(ctx->embedding.data(), embedding_size * sizeof(float));
        }
    }

    // 复制kv缓存
}
// 从指定的源地址复制状态数据到目标地址
size_t llama_copy_state_data(struct llama_context * ctx, uint8_t * dst) {
    // 创建数据缓冲区上下文对象，传入目标地址
    llama_data_buffer_context data_ctx(dst);
    // 调用内部函数，将上下文对象中的数据复制到目标地址
    llama_copy_state_data_internal(ctx, &data_ctx);

    // 返回写入数据的大小
    return data_ctx.get_size_written();
}

// 从指定的源地址设置状态数据
size_t llama_set_state_data(struct llama_context * ctx, uint8_t * src) {
    // 初始化输入指针
    uint8_t * inp = src;

    // 设置随机数生成器状态
    {
        // 定义随机数生成器状态大小和缓冲区
        size_t rng_size;
        char   rng_buf[LLAMA_MAX_RNG_STATE];
        // 从输入指针中复制随机数生成器状态大小，并移动指针
        memcpy(&rng_size,   inp, sizeof(rng_size));    inp += sizeof(rng_size);
        // 从输入指针中复制随机数生成器状态数据，并移动指针
        memcpy(&rng_buf[0], inp, LLAMA_MAX_RNG_STATE); inp += LLAMA_MAX_RNG_STATE;
        // 创建字符串流对象，将随机数生成器状态数据写入，并读取到上下文对象中的随机数生成器
        std::stringstream rng_ss;
        rng_ss.str(std::string(&rng_buf[0], rng_size));
        rng_ss >> ctx->rng;
        // 断言字符串流操作没有失败
        GGML_ASSERT(!rng_ss.fail());
    }

    // 设置logits
    {
        // 定义logits容量和大小
        size_t logits_cap;
        size_t logits_size;
        // 从输入指针中复制logits容量，并移动指针
        memcpy(&logits_cap,  inp, sizeof(logits_cap));  inp += sizeof(logits_cap);
        // 从输入指针中复制logits大小，并移动指针
        memcpy(&logits_size, inp, sizeof(logits_size)); inp += sizeof(logits_size);
        // 断言上下文对象中的logits容量与复制的logits容量相等
        GGML_ASSERT(ctx->logits.capacity() == logits_cap);
        // 如果logits大小不为0，则调整logits大小并从输入指针中复制数据
        if (logits_size) {
            ctx->logits.resize(logits_size);
            memcpy(ctx->logits.data(), inp, logits_size * sizeof(float));
        }
        // 移动输入指针
        inp += logits_cap * sizeof(float);
    }

    // 设置embeddings
    {
        // 定义embedding大小
        size_t embedding_size;
        // 从输入指针中复制embedding大小，并移动指针
        memcpy(&embedding_size, inp, sizeof(embedding_size)); inp += sizeof(embedding_size);
        // 断言上下文对象中的embedding容量与复制的embedding大小相等
        GGML_ASSERT(ctx->embedding.capacity() == embedding_size);
        // 如果embedding大小不为0，则从输入指针中复制数据
        if (embedding_size) {
            memcpy(ctx->embedding.data(), inp, embedding_size * sizeof(float));
            inp += embedding_size * sizeof(float);
        }
    }

    // 设置kv cache

    // 计算已读取的数据大小
    const size_t nread    = inp - src;
    // 获取状态数据的最大大小
    const size_t max_size = llama_get_state_size(ctx);
    // 断言已读取的数据大小不超过最大大小
    GGML_ASSERT(nread <= max_size);

    // 返回已读取的数据大小
    return nread;
}
static bool llama_load_session_file_internal(struct llama_context * ctx, const char * path_session, llama_token * tokens_out, size_t n_token_capacity, size_t * n_token_count_out) {
    // 以只读方式打开指定路径的文件
    llama_file file(path_session, "rb");

    // 进行一些基本的检查
    {
        // 读取文件中的魔数和版本号
        const uint32_t magic   = file.read_u32();
        const uint32_t version = file.read_u32();

        // 如果魔数或版本号不匹配，则输出错误信息并返回false
        if (magic != LLAMA_SESSION_MAGIC || version != LLAMA_SESSION_VERSION) {
            LLAMA_LOG_ERROR("%s : unknown (magic, version) for session file: %08x, %08x\n", __func__, magic, version);
            return false;
        }

        // 读取文件中的会话超参数
        llama_hparams session_hparams;
        file.read_raw(&session_hparams, sizeof(llama_hparams));

        // 如果会话超参数与上下文模型的超参数不匹配，则输出提示信息并返回false
        if (session_hparams != ctx->model.hparams) {
            LLAMA_LOG_INFO("%s : model hparams didn't match from session file!\n", __func__);
            return false;
        }
    }

    // 加载提示信息
    {
        // 读取文件中的令牌数量
        const uint32_t n_token_count = file.read_u32();

        // 如果令牌数量超过了容量，则输出错误信息并返回false
        if (n_token_count > n_token_capacity) {
            LLAMA_LOG_ERROR("%s : token count in session file exceeded capacity! %u > %zu\n", __func__, n_token_count, n_token_capacity);
            return false;
        }

        // 读取文件中的令牌数据
        file.read_raw(tokens_out, sizeof(llama_token) * n_token_count);
        *n_token_count_out = n_token_count;
    }

    // 恢复上下文状态
    {
        // 计算当前状态数据的大小
        const size_t n_state_size_cur = file.size - file.tell();
        const size_t n_state_size_max = llama_get_state_size(ctx);

        // 如果当前状态数据大小超过了最大值，则输出错误信息并返回false
        if (n_state_size_cur > n_state_size_max) {
            LLAMA_LOG_ERROR("%s : the state size in session file is too big! max %zu, got %zu\n", __func__, n_state_size_max, n_state_size_cur);
            return false;
        }

        // 创建一个大小为最大状态数据大小的字节向量
        std::vector<uint8_t> state_data(n_state_size_max);
        // 读取文件中的状态数据
        file.read_raw(state_data.data(), n_state_size_cur);

        // 设置上下文的状态数据
        llama_set_state_data(ctx, state_data.data());
    }

    return true;
}
# 加载会话文件的函数，将会话文件中的数据加载到内存中
bool llama_load_session_file(struct llama_context * ctx, const char * path_session, llama_token * tokens_out, size_t n_token_capacity, size_t * n_token_count_out) {
    try {
        # 调用内部函数加载会话文件的数据
        return llama_load_session_file_internal(ctx, path_session, tokens_out, n_token_capacity, n_token_count_out);
    } catch (const std::exception & err) {
        # 捕获异常并记录错误日志
        LLAMA_LOG_ERROR("error loading session file: %s\n", err.what());
        return false;
    }
}

# 保存会话文件的函数，将内存中的数据保存到会话文件中
bool llama_save_session_file(struct llama_context * ctx, const char * path_session, const llama_token * tokens, size_t n_token_count) {
    # 创建会话文件对象
    llama_file file(path_session, "wb");

    # 写入会话文件的魔数和版本号
    file.write_u32(LLAMA_SESSION_MAGIC);
    file.write_u32(LLAMA_SESSION_VERSION);

    # 写入模型超参数
    file.write_raw(&ctx->model.hparams, sizeof(llama_hparams));

    # 保存提示信息
    file.write_u32((uint32_t) n_token_count);
    file.write_raw(tokens, sizeof(llama_token) * n_token_count);

    # 使用流保存方式保存上下文状态
    llama_data_file_context data_ctx(&file);
    llama_copy_state_data_internal(ctx, &data_ctx);

    return true;
}

# 对输入的 tokens 进行评估
int llama_eval(struct llama_context * ctx, llama_token * tokens, int32_t n_tokens, int n_past) {
    # 从缓存中移除过期的键值对
    llama_kv_cache_seq_rm(ctx->kv_self, -1, n_past, -1);

    # 解码 tokens 并返回结果
    const int ret = llama_decode_internal(*ctx, llama_batch_get_one(tokens, n_tokens, n_past, 0));
    if (ret < 0) {
        # 如果解码失败，则记录错误日志
        LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
    }

    return ret;
}

# 对输入的 embd 进行评估
int llama_eval_embd(struct llama_context * ctx, float * embd, int32_t n_tokens, int n_past) {
    # 从缓存中移除过期的键值对
    llama_kv_cache_seq_rm(ctx->kv_self, -1, n_past, -1);

    # 构建 batch 对象
    llama_batch batch = { n_tokens, nullptr, embd, nullptr, nullptr, nullptr, nullptr, n_past, 1, 0, };

    # 解码 embd 并返回结果
    const int ret = llama_decode_internal(*ctx, batch);
    # 如果返回值小于 0
    if (ret < 0) {
        # 记录错误日志，包括函数名和返回值
        LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
    }
    # 返回 ret 值
    return ret;
}

void llama_set_n_threads(struct llama_context * ctx, uint32_t n_threads, uint32_t n_threads_batch) {
    // 设置上下文中的线程数和批处理线程数
    ctx->cparams.n_threads       = n_threads;
    ctx->cparams.n_threads_batch = n_threads_batch;
}

struct llama_batch llama_batch_get_one(
             llama_token * tokens,
                 int32_t   n_tokens,
               llama_pos   pos_0,
            llama_seq_id   seq_id) {
    // 返回一个 llamba_batch 结构体
    return {
        /*n_tokens       =*/ n_tokens,  // 设置 n_tokens 字段
        /*tokens         =*/ tokens,     // 设置 tokens 字段
        /*embd           =*/ nullptr,    // 设置 embd 字段为 nullptr
        /*pos            =*/ nullptr,    // 设置 pos 字段为 nullptr
        /*n_seq_id       =*/ nullptr,    // 设置 n_seq_id 字段为 nullptr
        /*seq_id         =*/ nullptr,    // 设置 seq_id 字段为 nullptr
        /*logits         =*/ nullptr,    // 设置 logits 字段为 nullptr
        /*all_pos_0      =*/ pos_0,      // 设置 all_pos_0 字段
        /*all_pos_1      =*/ 1,          // 设置 all_pos_1 字段
        /*all_seq_id     =*/ seq_id,     // 设置 all_seq_id 字段
    };
}

struct llama_batch llama_batch_init(int32_t n_tokens, int32_t embd, int32_t n_seq_max) {
    // 初始化一个 llamba_batch 结构体
    llama_batch batch = { 0, nullptr, nullptr, nullptr, nullptr, nullptr, nullptr, 0, 0, 0, };

    if (embd) {
        // 如果 embd 不为 0，则分配内存给 embd 字段
        batch.embd = (float *) malloc(sizeof(float) * n_tokens * embd);
    } else {
        // 如果 embd 为 0，则分配内存给 token 字段
        batch.token = (llama_token *) malloc(sizeof(llama_token) * n_tokens);
    }

    // 分配内存给 pos 字段
    batch.pos      = (llama_pos *)     malloc(sizeof(llama_pos)      * n_tokens);
    // 分配内存给 n_seq_id 字段
    batch.n_seq_id = (int32_t *)       malloc(sizeof(int32_t)        * n_tokens);
    // 分配内存给 seq_id 字段
    batch.seq_id   = (llama_seq_id **) malloc(sizeof(llama_seq_id *) * n_tokens);
    for (int i = 0; i < n_tokens; ++i) {
        // 为每个 seq_id[i] 分配内存
        batch.seq_id[i] = (llama_seq_id *) malloc(sizeof(llama_seq_id) * n_seq_max);
    }
    // 分配内存给 logits 字段
    batch.logits   = (int8_t *)        malloc(sizeof(int8_t)         * n_tokens);

    return batch;  // 返回初始化后的 llamba_batch 结构体
}

void llama_batch_free(struct llama_batch batch) {
    // 释放分配的内存
    if (batch.token)    free(batch.token);
    if (batch.embd)     free(batch.embd);
    if (batch.pos)      free(batch.pos);
    if (batch.n_seq_id) free(batch.n_seq_id);
    # 如果批处理中的序列ID存在
    if (batch.seq_id) {
        # 遍历批处理中的令牌数量，释放每个序列ID的内存
        for (int i = 0; i < batch.n_tokens; ++i) {
            free(batch.seq_id[i]);
        }
        # 释放整个序列ID数组的内存
        free(batch.seq_id);
    }
    # 如果批处理中的logits存在，则释放其内存
    if (batch.logits)   free(batch.logits);
// 解码函数，将 llama_context 和 llama_batch 作为参数，返回解码结果
int llama_decode(
        struct llama_context * ctx,
          struct llama_batch   batch) {
    // 调用内部解码函数，将结果保存在 ret 中
    const int ret = llama_decode_internal(*ctx, batch);
    // 如果解码失败，记录错误信息
    if (ret < 0) {
        LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
    }
    // 返回解码结果
    return ret;
}

// 获取 logits 数组的指针
float * llama_get_logits(struct llama_context * ctx) {
    return ctx->logits.data();
}

// 获取第 i 个 logits 的指针
float * llama_get_logits_ith(struct llama_context * ctx, int32_t i) {
    return ctx->logits.data() + i*ctx->model.hparams.n_vocab;
}

// 获取 embedding 数组的指针
float * llama_get_embeddings(struct llama_context * ctx) {
    return ctx->embedding.data();
}

// 获取 token 对应的文本内容
const char * llama_token_get_text(const struct llama_model * model, llama_token token) {
    return model->vocab.id_to_token[token].text.c_str();
}

// 获取 token 对应的分数
float llama_token_get_score(const struct llama_model * model, llama_token token) {
    return model->vocab.id_to_token[token].score;
}

// 获取 token 的类型
llama_token_type llama_token_get_type(const struct llama_model * model, llama_token token) {
    return model->vocab.id_to_token[token].type;
}

// 获取特殊 token: bos
llama_token llama_token_bos(const struct llama_model * model) {
    return model->vocab.special_bos_id;
}

// 获取特殊 token: eos
llama_token llama_token_eos(const struct llama_model * model) {
    return model->vocab.special_eos_id;
}

// 获取特殊 token: nl
llama_token llama_token_nl(const struct llama_model * model) {
    return model->vocab.linefeed_id;
}

// 获取特殊 token: prefix
llama_token llama_token_prefix(const struct llama_model * model) {
    return model->vocab.special_prefix_id;
}

// 获取特殊 token: middle
llama_token llama_token_middle(const struct llama_model * model) {
    return model->vocab.special_middle_id;
}

// 获取特殊 token: suffix
llama_token llama_token_suffix(const struct llama_model * model) {
    return model->vocab.special_suffix_id;
}

// 获取特殊 token: eot
llama_token llama_token_eot(const struct llama_model * model) {
    return model->vocab.special_eot_id;
}

// 分词函数
int llama_tokenize(
    // 声明一个指向 llama_model 结构体的指针 model，一个指向字符的指针 text，一个整型变量 text_len，一个指向 llama_token 结构体的指针 tokens，一个整型变量 n_max_tokens，一个布尔变量 add_bos，一个布尔变量 special
    const struct llama_model * model,
                  const char * text,
                         int   text_len,
                 llama_token * tokens,
                         int   n_max_tokens,
                        bool   add_bos,
                        bool   special) {
    // 调用 llama_tokenize_internal 函数对输入的文本进行分词处理，返回结果存储在 res 中
    auto res = llama_tokenize_internal(model->vocab, std::string(text, text_len), add_bos, special);

    // 如果 n_max_tokens 小于 res 的大小，则输出错误信息并返回负数 res 的大小
    if (n_max_tokens < (int) res.size()) {
        // LLAMA_LOG_ERROR("%s: too many tokens\n", __func__);
        return -((int) res.size());
    }

    // 将 res 中的分词结果复制到 tokens 数组中
    for (size_t i = 0; i < res.size(); i++) {
        tokens[i] = res[i];
    }

    // 返回 res 的大小作为处理后的分词结果的数量
    return res.size();
}

// 解码文本，将 UTF-8 编码的文本解码为字节流
static std::string llama_decode_text(const std::string & text) {
    // 存储解码后的文本
    std::string decoded_text;
    // 将 UTF-8 编码的文本转换为 Unicode 序列
    auto unicode_sequences = codepoints_from_utf8(text);
    // 遍历 Unicode 序列，将每个 Unicode 序列转换为字节流并添加到解码后的文本中
    for (auto& unicode_sequence : unicode_sequences) {
        decoded_text += unicode_to_bytes_bpe(codepoint_to_utf8(unicode_sequence));
    }

    return decoded_text;
}

// 将 llama_token 转换为 piece，并写入到 buf 中，不包括空终止符
int llama_token_to_piece(const struct llama_model * model, llama_token token, char * buf, int length) {
    // 返回 0 表示成功
    return 0;
}

// 获取 llama_context 的时间统计信息
struct llama_timings llama_get_timings(struct llama_context * ctx) {
    // 初始化时间统计结果
    struct llama_timings result = {
        /*.t_start_ms  =*/ 1e-3 * ctx->t_start_us,
        /*.t_end_ms    =*/ 1.00 * ggml_time_ms(),
        /*.t_load_ms   =*/ 1e-3 * ctx->t_load_us,
        /*.t_sample_ms =*/ 1e-3 * ctx->t_sample_us,
        /*.t_p_eval_ms =*/ 1e-3 * ctx->t_p_eval_us,
        /*.t_eval_ms   =*/ 1e-3 * ctx->t_eval_us,

        /*.n_sample =*/ std::max(1, ctx->n_sample),
        /*.n_p_eval =*/ std::max(1, ctx->n_p_eval),
        /*.n_eval   =*/ std::max(1, ctx->n_eval),
    };

    return result;
}

// 打印 llama_context 的时间统计信息
void llama_print_timings(struct llama_context * ctx) {
    // 获取时间统计信息
    const llama_timings timings = llama_get_timings(ctx);

    // 打印加载时间
    LLAMA_LOG_INFO("\n");
    LLAMA_LOG_INFO("%s:        load time = %10.2f ms\n", __func__, timings.t_load_ms);
    // 打印采样时间
    LLAMA_LOG_INFO("%s:      sample time = %10.2f ms / %5d runs   (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, timings.t_sample_ms, timings.n_sample, timings.t_sample_ms / timings.n_sample, 1e3 / timings.t_sample_ms * timings.n_sample);
    // 打印提示评估时间
    LLAMA_LOG_INFO("%s: prompt eval time = %10.2f ms / %5d tokens (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, timings.t_p_eval_ms, timings.n_p_eval, timings.t_p_eval_ms / timings.n_p_eval, 1e3 / timings.t_p_eval_ms * timings.n_p_eval);
}
    # 记录信息日志，包括函数名、评估时间、评估次数、每个标记的平均时间、每秒处理的标记数
    LLAMA_LOG_INFO("%s:        eval time = %10.2f ms / %5d runs   (%8.2f ms per token, %8.2f tokens per second)\n",
                __func__, timings.t_eval_ms, timings.n_eval, timings.t_eval_ms / timings.n_eval, 1e3 / timings.t_eval_ms * timings.n_eval);
    # 记录信息日志，包括函数名、总时间
    LLAMA_LOG_INFO("%s:       total time = %10.2f ms\n", __func__, (timings.t_end_ms - timings.t_start_ms));
# 重置 llama_context 结构体中的计时信息
void llama_reset_timings(struct llama_context * ctx) {
    # 记录重置计时的起始时间
    ctx->t_start_us = ggml_time_us();
    # 重置采样时间和采样次数
    ctx->t_sample_us = ctx->n_sample = 0;
    # 重置评估时间和评估次数
    ctx->t_eval_us   = ctx->n_eval   = 0;
    # 重置部分评估时间和部分评估次数
    ctx->t_p_eval_us = ctx->n_p_eval = 0;
}

# 打印系统信息并返回字符串指针
const char * llama_print_system_info(void) {
    # 静态字符串对象
    static std::string s;

    # 清空字符串
    s  = "";
    # 拼接字符串，记录 AVX 是否可用
    s += "AVX = "         + std::to_string(ggml_cpu_has_avx())         + " | ";
    # 拼接字符串，记录 AVX2 是否可用
    s += "AVX2 = "        + std::to_string(ggml_cpu_has_avx2())        + " | ";
    # 拼接字符串，记录 AVX512 是否可用
    s += "AVX512 = "      + std::to_string(ggml_cpu_has_avx512())      + " | ";
    # 拼接字符串，记录 AVX512_VBMI 是否可用
    s += "AVX512_VBMI = " + std::to_string(ggml_cpu_has_avx512_vbmi()) + " | ";
    # 拼接字符串，记录 AVX512_VNNI 是否可用
    s += "AVX512_VNNI = " + std::to_string(ggml_cpu_has_avx512_vnni()) + " | ";
    # 拼接字符串，记录 FMA 是否可用
    s += "FMA = "         + std::to_string(ggml_cpu_has_fma())         + " | ";
    # 拼接字符串，记录 NEON 是否可用
    s += "NEON = "        + std::to_string(ggml_cpu_has_neon())        + " | ";
    # 拼接字符串，记录 ARM_FMA 是否可用
    s += "ARM_FMA = "     + std::to_string(ggml_cpu_has_arm_fma())     + " | ";
    # 拼接字符串，记录 F16C 是否可用
    s += "F16C = "        + std::to_string(ggml_cpu_has_f16c())        + " | ";
    # 拼接字符串，记录 FP16_VA 是否可用
    s += "FP16_VA = "     + std::to_string(ggml_cpu_has_fp16_va())     + " | ";
    # 拼接字符串，记录 WASM_SIMD 是否可用
    s += "WASM_SIMD = "   + std::to_string(ggml_cpu_has_wasm_simd())   + " | ";
    # 拼接字符串，记录 BLAS 是否可用
    s += "BLAS = "        + std::to_string(ggml_cpu_has_blas())        + " | ";
    # 拼接字符串，记录 SSE3 是否可用
    s += "SSE3 = "        + std::to_string(ggml_cpu_has_sse3())        + " | ";
    # 拼接字符串，记录 SSSE3 是否可用
    s += "SSSE3 = "       + std::to_string(ggml_cpu_has_ssse3())       + " | ";
    # 拼接字符串，记录 VSX 是否可用
    s += "VSX = "         + std::to_string(ggml_cpu_has_vsx())         + " | ";

    # 返回字符串指针
    return s.c_str();
}

# 将 llama_context 结构体中的计时信息以 YAML 格式写入文件流
void llama_dump_timing_info_yaml(FILE * stream, const llama_context * ctx) {
    # 写入换行符
    fprintf(stream, "\n");
    # 写入分隔符
    fprintf(stream, "###########\n");
    # 写入标题
    fprintf(stream, "# Timings #\n");
    # 写入分隔符
    fprintf(stream, "###########\n");
    # 写入换行符
    fprintf(stream, "\n");

    # 写入评估时间的平均值
    fprintf(stream, "mst_eval: %.2f  # ms / token during generation\n",
            1.0e-3 * ctx->t_eval_us / ctx->n_eval);
    # 输出 prompt 处理期间每个 token 的平均时间（毫秒/个）
    fprintf(stream, "mst_p_eval: %.2f  # ms / token during prompt processing\n",
            1.0e-3 * ctx->t_p_eval_us / ctx->n_p_eval);
    # 输出抽样期间每个 token 的平均时间（毫秒/个）
    fprintf(stream, "mst_sample: %.2f  # ms / token during sampling\n",
            1.0e-3 * ctx->t_sample_us / ctx->n_sample);
    # 输出生成的 token 数量（不包括第一个） 
    fprintf(stream, "n_eval: %d  # number of tokens generated (excluding the first one)\n", ctx->n_eval);
    # 输出在开始阶段以批处理方式处理的 token 数量
    fprintf(stream, "n_p_eval: %d  # number of tokens processed in batches at the beginning\n", ctx->n_p_eval);
    # 输出抽样的 token 数量
    fprintf(stream, "n_sample: %d  # number of sampled tokens\n", ctx->n_sample);
    # 输出生成 token 总共花费的微秒数
    fprintf(stream, "t_eval_us: %" PRId64 "  # total microseconds spent generating tokens\n", ctx->t_eval_us);
    # 输出加载模型总共花费的微秒数
    fprintf(stream, "t_load_us: %" PRId64 "  # total microseconds spent loading the model\n", ctx->t_load_us);
    # 输出 prompt 处理总共花费的微秒数
    fprintf(stream, "t_p_eval_us: %" PRId64 "  # total microseconds spent prompt processing\n", ctx->t_p_eval_us);
    # 输出抽样总共花费的微秒数
    fprintf(stream, "t_sample_us: %" PRId64 "  # total microseconds spent sampling\n", ctx->t_sample_us);
    # 输出生成 token 的速度（每秒生成的 token 数）
    fprintf(stream, "ts_eval: %.2f  # tokens / second during generation\n",
            1.0e6 * ctx->n_eval / ctx->t_eval_us);
    # 输出 prompt 处理的速度（每秒处理的 token 数）
    fprintf(stream, "ts_p_eval: %.2f  # tokens / second during prompt processing\n",
            1.0e6 * ctx->n_p_eval / ctx->t_p_eval_us);
    # 输出抽样的速度（每秒抽样的 token 数）
    fprintf(stream, "ts_sample: %.2f  # tokens / second during sampling\n",
            1.0e6 * ctx->n_sample / ctx->t_sample_us);
// 返回模型上下文中的张量名称和张量指针的映射
const std::vector<std::pair<std::string, struct ggml_tensor *>> & llama_internal_get_tensor_map(
    struct llama_context * ctx
) {
    return ctx->model.tensors_by_name;
}

// 设置日志回调函数和用户数据
void llama_log_set(ggml_log_callback log_callback, void * user_data) {
    // 如果 log_callback 为空，则使用默认的日志回调函数
    g_state.log_callback = log_callback ? log_callback : llama_log_callback_default;
    g_state.log_callback_user_data = user_data;
}

// 内部使用的日志函数，带可变参数列表
static void llama_log_internal_v(ggml_log_level level, const char * format, va_list args) {
    va_list args_copy;
    va_copy(args_copy, args);
    char buffer[128];
    // 格式化日志消息
    int len = vsnprintf(buffer, 128, format, args);
    // 如果消息长度小于 128，则直接调用日志回调函数
    if (len < 128) {
        g_state.log_callback(level, buffer, g_state.log_callback_user_data);
    } else {
        // 如果消息长度大于等于 128，则动态分配内存来存储消息，并调用日志回调函数
        char* buffer2 = new char[len+1];
        vsnprintf(buffer2, len+1, format, args_copy);
        buffer2[len] = 0;
        g_state.log_callback(level, buffer2, g_state.log_callback_user_data);
        delete[] buffer2;
    }
    va_end(args_copy);
}

// 内部使用的日志函数，带固定参数列表
static void llama_log_internal(ggml_log_level level, const char * format, ...) {
    va_list args;
    va_start(args, format);
    llama_log_internal_v(level, format, args);
    va_end(args);
}

// 默认的日志回调函数，将日志消息输出到标准错误流
static void llama_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);
    fflush(stderr);
}
```