# `whisper.cpp\examples\talk-llama\llama.cpp`

```cpp
#define LLAMA_API_INTERNAL
#include "llama.h"

#include "unicode.h"

#include "ggml.h"
#include "ggml-alloc.h"
#include "ggml-backend.h"

#ifdef GGML_USE_CUBLAS
#  include "ggml-cuda.h"
#elif defined(GGML_USE_CLBLAST)
#  include "ggml-opencl.h"
#elif defined(GGML_USE_VULKAN)
#  include "ggml-vulkan.h"
#elif defined(GGML_USE_SYCL)
#  include "ggml-sycl.h"
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
            #include <fcntl.h>
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
#endif

#include <algorithm>
#include <array>
#include <cassert>
#include <cfloat>
#include <cinttypes>
#include <climits>
#include <cmath>
#include <cstdarg>
#include <cstddef>
#include <cstdint>
#include <cstdio>
#include <cstring>
#include <ctime>
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
#include <type_traits>
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

#define LLAMA_MAX_NODES   8192
// 定义 LLAMA_MAX_EXPERTS 常量为 8

//
// logging
//

// 声明一个静态函数 llama_log_internal，用于记录日志信息
LLAMA_ATTRIBUTE_FORMAT(2, 3)
static void llama_log_internal        (ggml_log_level level, const char* format, ...);
// 声明一个静态函数 llama_log_callback_default，用于记录默认的日志回调函数
static void llama_log_callback_default(ggml_log_level level, const char * text, void * user_data);

// 定义宏 LLAMA_LOG_INFO，用于记录信息级别的日志
#define LLAMA_LOG_INFO(...)  llama_log_internal(GGML_LOG_LEVEL_INFO , __VA_ARGS__)
// 定义宏 LLAMA_LOG_WARN，用于记录警告级别的日志
#define LLAMA_LOG_WARN(...)  llama_log_internal(GGML_LOG_LEVEL_WARN , __VA_ARGS__)
// 定义宏 LLAMA_LOG_ERROR，用于记录错误级别的日志
#define LLAMA_LOG_ERROR(...) llama_log_internal(GGML_LOG_LEVEL_ERROR, __VA_ARGS__)

//
// helpers
//

// 定义静态函数 utf8_len，用于计算 UTF-8 编码字符的长度
static size_t utf8_len(char src) {
    // 定义一个数组 lookup，用于存储不同高位字节对应的字符长度
    const size_t lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3, 4 };
    // 计算字符的高位字节
    uint8_t highbits = static_cast<uint8_t>(src) >> 4;
    // 返回字符的长度
    return lookup[highbits];
}

// 定义静态函数 replace_all，用于替换字符串中的所有指定子串
static void replace_all(std::string & s, const std::string & search, const std::string & replace) {
    // 定义一个新的字符串 result
    std::string result;
    // 遍历原字符串 s，查找并替换指定子串
    for (size_t pos = 0; ; pos += search.length()) {
        auto new_pos = s.find(search, pos);
        if (new_pos == std::string::npos) {
            result += s.substr(pos, s.size() - pos);
            break;
        }
        result += s.substr(pos, new_pos - pos) + replace;
        pos = new_pos;
    }
    // 将替换后的结果赋值给原字符串 s
    s = std::move(result);
}

// 定义静态函数 is_float_close，用于比较两个浮点数是否接近
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

    // 使用提供的绝对容差进行比较
    return std::fabs(b - a) <= abs_tol;
}

// 定义静态函数 zeros，用于向文件写入指定数量的零字节
static void zeros(std::ofstream & file, size_t n) {
    // 定义一个零字节
    char zero = 0;
    // 向文件写入 n 个零字节
    for (size_t i = 0; i < n; ++i) {
        file.write(&zero, 1);
    }
}

// 定义宏 LLAMA_ATTRIBUTE_FORMAT，用于格式化字符串
LLAMA_ATTRIBUTE_FORMAT(1, 2)
// 定义函数 format，用于格式化字符串
static std::string format(const char * fmt, ...) {
    // 定义变长参数列表
    va_list ap;
    va_list ap2;
    va_start(ap, fmt);
    va_copy(ap2, ap);
    // 使用 vsnprintf 计算格式化字符串 fmt 和可变参数 ap 所需的缓冲区大小
    int size = vsnprintf(NULL, 0, fmt, ap);
    // 断言 size 大于等于 0 且小于 INT_MAX
    GGML_ASSERT(size >= 0 && size < INT_MAX); // NOLINT
    // 创建一个大小为 size+1 的字符向量 buf
    std::vector<char> buf(size + 1);
    // 使用 vsnprintf 将格式化字符串 fmt 和可变参数 ap2 写入 buf 中
    int size2 = vsnprintf(buf.data(), size + 1, fmt, ap2);
    // 断言 size2 等于 size
    GGML_ASSERT(size2 == size);
    // 结束可变参数 ap2
    va_end(ap2);
    // 结束可变参数 ap
    va_end(ap);
    // 返回 buf 中的字符串，长度为 size
    return std::string(buf.data(), size);
// gguf 常量（与 gguf.py 同步）

// 定义枚举类型 llm_arch，表示不同的架构
enum llm_arch {
    LLM_ARCH_LLAMA,         // llama 架构
    LLM_ARCH_FALCON,        // falcon 架构
    LLM_ARCH_BAICHUAN,      // baichuan 架构
    LLM_ARCH_GPT2,          // gpt2 架构
    LLM_ARCH_GPTJ,          // gptj 架构
    LLM_ARCH_GPTNEOX,       // gptneox 架构
    LLM_ARCH_MPT,           // mpt 架构
    LLM_ARCH_STARCODER,     // starcoder 架构
    LLM_ARCH_PERSIMMON,     // persimmon 架构
    LLM_ARCH_REFACT,        // refact 架构
    LLM_ARCH_BLOOM,         // bloom 架构
    LLM_ARCH_STABLELM,      // stablelm 架构
    LLM_ARCH_QWEN,          // qwen 架构
    LLM_ARCH_QWEN2,         // qwen2 架构
    LLM_ARCH_PHI2,          // phi2 架构
    LLM_ARCH_PLAMO,         // plamo 架构
    LLM_ARCH_CODESHELL,     // codeshell 架构
    LLM_ARCH_ORION,         // orion 架构
    LLM_ARCH_UNKNOWN,       // 未知架构
};

// 定义静态映射表 LLM_ARCH_NAMES，将架构枚举值映射到对应的字符串
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
    { LLM_ARCH_QWEN,            "qwen"      },
    { LLM_ARCH_QWEN2,           "qwen2"     },
    { LLM_ARCH_PHI2,            "phi2"      },
    { LLM_ARCH_PLAMO,           "plamo"     },
    { LLM_ARCH_CODESHELL,       "codeshell" },
    { LLM_ARCH_ORION,           "orion"     },
};

// 定义枚举类型 llm_kv，表示不同的键值
enum llm_kv {
    LLM_KV_GENERAL_ARCHITECTURE,            // 通用架构
    LLM_KV_GENERAL_QUANTIZATION_VERSION,    // 通用量化版本
    LLM_KV_GENERAL_ALIGNMENT,               // 通用对齐
    LLM_KV_GENERAL_NAME,                    // 通用名称
    LLM_KV_GENERAL_AUTHOR,                  // 通用作者
    LLM_KV_GENERAL_URL,                     // 通用 URL
    LLM_KV_GENERAL_DESCRIPTION,             // 通用描述
    LLM_KV_GENERAL_LICENSE,                 // 通用许可证
    LLM_KV_GENERAL_SOURCE_URL,              // 通用源 URL
    LLM_KV_GENERAL_SOURCE_HF_REPO,          // 通用源 HF 仓库

    LLM_KV_CONTEXT_LENGTH,                  // 上下文长度
    LLM_KV_EMBEDDING_LENGTH,                // 嵌入长度
    LLM_KV_BLOCK_COUNT,                     // 块数量
    LLM_KV_FEED_FORWARD_LENGTH,             // 前馈长度
    LLM_KV_USE_PARALLEL_RESIDUAL,           // 使用并行残差
    LLM_KV_TENSOR_DATA_LAYOUT,              // 张量数据布局
    LLM_KV_EXPERT_COUNT,                    // 专家数量
    LLM_KV_EXPERT_USED_COUNT,               // 使用的专家数量

    LLM_KV_ATTENTION_HEAD_COUNT,            // 注意力头数量
    LLM_KV_ATTENTION_HEAD_COUNT_KV,         // 注意力头数量 KV
    LLM_KV_ATTENTION_MAX_ALIBI_BIAS,  # 定义注意力机制的最大偏差
    LLM_KV_ATTENTION_CLAMP_KQV,  # 定义注意力机制的 KQV 值的截断
    LLM_KV_ATTENTION_KEY_LENGTH,  # 定义注意力机制的键长度
    LLM_KV_ATTENTION_VALUE_LENGTH,  # 定义注意力机制的值长度
    LLM_KV_ATTENTION_LAYERNORM_EPS,  # 定义注意力机制的 LayerNorm epsilon
    LLM_KV_ATTENTION_LAYERNORM_RMS_EPS,  # 定义注意力机制的 LayerNorm RMS epsilon

    LLM_KV_ROPE_DIMENSION_COUNT,  # 定义绳索的维度计数
    LLM_KV_ROPE_FREQ_BASE,  # 定义绳索的基础频率
    LLM_KV_ROPE_SCALE_LINEAR,  # 定义绳索的线性缩放
    LLM_KV_ROPE_SCALING_TYPE,  # 定义绳索的缩放类型
    LLM_KV_ROPE_SCALING_FACTOR,  # 定义绳索的缩放因子
    LLM_KV_ROPE_SCALING_ORIG_CTX_LEN,  # 定义绳索的原始上下文长度
    LLM_KV_ROPE_SCALING_FINETUNED,  # 定义绳索的微调

    LLM_KV_TOKENIZER_MODEL,  # 定义分词器的模型
    LLM_KV_TOKENIZER_LIST,  # 定义分词器的列表
    LLM_KV_TOKENIZER_TOKEN_TYPE,  # 定义分词器的标记类型
    LLM_KV_TOKENIZER_SCORES,  # 定义分词器的分数
    LLM_KV_TOKENIZER_MERGES,  # 定义分词器的合并
    LLM_KV_TOKENIZER_BOS_ID,  # 定义分词器的 BOS ID
    LLM_KV_TOKENIZER_EOS_ID,  # 定义分词器的 EOS ID
    LLM_KV_TOKENIZER_UNK_ID,  # 定义分词器的 UNK ID
    LLM_KV_TOKENIZER_SEP_ID,  # 定义分词器的 SEP ID
    LLM_KV_TOKENIZER_PAD_ID,  # 定义分词器的 PAD ID
    LLM_KV_TOKENIZER_ADD_BOS,  # 定义分词器是否添加 BOS
    LLM_KV_TOKENIZER_ADD_EOS,  # 定义分词器是否添加 EOS
    LLM_KV_TOKENIZER_HF_JSON,  # 定义分词器的 HF JSON
    LLM_KV_TOKENIZER_RWKV,  # 定义分词器的 RWKV
// 定义一个静态的键值对映射，将枚举值和对应的字符串名称进行关联
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
    { LLM_KV_EXPERT_COUNT,                  "%s.expert_count"          },
    { LLM_KV_EXPERT_USED_COUNT,             "%s.expert_used_count"     },

    { LLM_KV_ATTENTION_HEAD_COUNT,          "%s.attention.head_count"             },
    { LLM_KV_ATTENTION_HEAD_COUNT_KV,       "%s.attention.head_count_kv"          },
    { LLM_KV_ATTENTION_MAX_ALIBI_BIAS,      "%s.attention.max_alibi_bias"         },
    { LLM_KV_ATTENTION_CLAMP_KQV,           "%s.attention.clamp_kqv"              },
    { LLM_KV_ATTENTION_KEY_LENGTH,          "%s.attention.key_length"             },
};
    { LLM_KV_ATTENTION_VALUE_LENGTH,        "%s.attention.value_length"           }, 
    # 存储注意力机制数值长度的键值对

    { LLM_KV_ATTENTION_LAYERNORM_EPS,       "%s.attention.layer_norm_epsilon"     }, 
    # 存储注意力机制层归一化 epsilon 的键值对

    { LLM_KV_ATTENTION_LAYERNORM_RMS_EPS,   "%s.attention.layer_norm_rms_epsilon" }, 
    # 存储注意力机制层归一化 RMS epsilon 的键值对

    { LLM_KV_ROPE_DIMENSION_COUNT,          "%s.rope.dimension_count"                 }, 
    # 存储绳索维度计数的键值对

    { LLM_KV_ROPE_FREQ_BASE,                "%s.rope.freq_base"                       }, 
    # 存储绳索频率基数的键值对

    { LLM_KV_ROPE_SCALE_LINEAR,             "%s.rope.scale_linear"                    }, 
    # 存储绳索线性缩放的键值对

    { LLM_KV_ROPE_SCALING_TYPE,             "%s.rope.scaling.type"                    }, 
    # 存储绳索缩放类型的键值对

    { LLM_KV_ROPE_SCALING_FACTOR,           "%s.rope.scaling.factor"                  }, 
    # 存储绳索缩放因子的键值对

    { LLM_KV_ROPE_SCALING_ORIG_CTX_LEN,     "%s.rope.scaling.original_context_length" }, 
    # 存储绳索缩放原始上下文长度的键值对

    { LLM_KV_ROPE_SCALING_FINETUNED,        "%s.rope.scaling.finetuned"               }, 
    # 存储绳索缩放微调的键值对

    { LLM_KV_TOKENIZER_MODEL,               "tokenizer.ggml.model"              }, 
    # 存储分词器模型的键值对

    { LLM_KV_TOKENIZER_LIST,                "tokenizer.ggml.tokens"             }, 
    # 存储分词器标记列表的键值对

    { LLM_KV_TOKENIZER_TOKEN_TYPE,          "tokenizer.ggml.token_type"         }, 
    # 存储分词器标记类型的键值对

    { LLM_KV_TOKENIZER_SCORES,              "tokenizer.ggml.scores"             }, 
    # 存储分词器分数的键值对

    { LLM_KV_TOKENIZER_MERGES,              "tokenizer.ggml.merges"             }, 
    # 存储分词器合并的键值对

    { LLM_KV_TOKENIZER_BOS_ID,              "tokenizer.ggml.bos_token_id"       }, 
    # 存储分词器起始标记 ID 的键值对

    { LLM_KV_TOKENIZER_EOS_ID,              "tokenizer.ggml.eos_token_id"       }, 
    # 存储分词器结束标记 ID 的键值对

    { LLM_KV_TOKENIZER_UNK_ID,              "tokenizer.ggml.unknown_token_id"   }, 
    # 存储分词器未知标记 ID 的键值对

    { LLM_KV_TOKENIZER_SEP_ID,              "tokenizer.ggml.seperator_token_id" }, 
    # 存储分词器分隔符标记 ID 的键值对

    { LLM_KV_TOKENIZER_PAD_ID,              "tokenizer.ggml.padding_token_id"   }, 
    # 存储分词器填充标记 ID 的键值对

    { LLM_KV_TOKENIZER_ADD_BOS,             "tokenizer.ggml.add_bos_token"      }, 
    # 存储分词器是否添加起始标记的键值对

    { LLM_KV_TOKENIZER_ADD_EOS,             "tokenizer.ggml.add_eos_token"      }, 
    # 存储分词器是否添加结束标记的键值对

    { LLM_KV_TOKENIZER_HF_JSON,             "tokenizer.huggingface.json"        }, 
    # 存储 Hugging Face JSON 的键值对
    # 定义一个键值对，键为LLM_KV_TOKENIZER_RWKV，值为"tokenizer.rwkv.world"
    { LLM_KV_TOKENIZER_RWKV, "tokenizer.rwkv.world" },
// 定义一个结构体 LLM_KV，包含一个枚举类型 llm_arch 的成员变量 arch
struct LLM_KV {
    LLM_KV(llm_arch arch) : arch(arch) {}

    llm_arch arch;

    // 重载 () 运算符，返回格式化后的字符串
    std::string operator()(llm_kv kv) const {
        return ::format(LLM_KV_NAMES[kv].c_str(), LLM_ARCH_NAMES[arch].c_str());
    }
};

// 定义一个枚举类型 llm_tensor，列出不同的 tensor 类型
enum llm_tensor {
    LLM_TENSOR_TOKEN_EMBD,
    LLM_TENSOR_TOKEN_EMBD_NORM,
    LLM_TENSOR_POS_EMBD,
    LLM_TENSOR_OUTPUT,
    LLM_TENSOR_OUTPUT_NORM,
    LLM_TENSOR_ROPE_FREQS,
    LLM_TENSOR_ATTN_Q,
    LLM_TENSOR_ATTN_K,
    LLM_TENSOR_ATTN_V,
    LLM_TENSOR_ATTN_QKV,
    LLM_TENSOR_ATTN_OUT,
    LLM_TENSOR_ATTN_NORM,
    LLM_TENSOR_ATTN_NORM_2,
    LLM_TENSOR_ATTN_ROT_EMBD,
    LLM_TENSOR_FFN_GATE_INP,
    LLM_TENSOR_FFN_NORM,
    LLM_TENSOR_FFN_GATE,
    LLM_TENSOR_FFN_DOWN,
    LLM_TENSOR_FFN_UP,
    LLM_TENSOR_FFN_ACT,
    LLM_TENSOR_FFN_DOWN_EXP,
    LLM_TENSOR_FFN_GATE_EXP,
    LLM_TENSOR_FFN_UP_EXP,
    LLM_TENSOR_ATTN_Q_NORM,
    LLM_TENSOR_ATTN_K_NORM,
};

// 定义一个静态的嵌套 map，将不同的架构和 tensor 类型映射到对应的字符串
static std::map<llm_arch, std::map<llm_tensor, std::string>> LLM_TENSOR_NAMES = {
    {
        LLM_ARCH_LLAMA,  // 定义LLM_ARCH_LLAMA常量
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_TENSOR_TOKEN_EMBD常量和对应的字符串
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_TENSOR_OUTPUT_NORM常量和对应的字符串
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_TENSOR_OUTPUT常量和对应的字符串
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  // 定义LLM_TENSOR_ROPE_FREQS常量和对应的字符串
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_TENSOR_ATTN_NORM常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  // 定义LLM_TENSOR_ATTN_Q常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  // 定义LLM_TENSOR_ATTN_K常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  // 定义LLM_TENSOR_ATTN_V常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_TENSOR_ATTN_OUT常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd" },  // 定义LLM_TENSOR_ATTN_ROT_EMBD常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_FFN_GATE_INP,    "blk.%d.ffn_gate_inp" },  // 定义LLM_TENSOR_FFN_GATE_INP常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义LLM_TENSOR_FFN_NORM常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  // 定义LLM_TENSOR_FFN_GATE常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_TENSOR_FFN_DOWN常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_TENSOR_FFN_UP常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_FFN_GATE_EXP,    "blk.%d.ffn_gate.%d" },  // 定义LLM_TENSOR_FFN_GATE_EXP常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_FFN_DOWN_EXP,    "blk.%d.ffn_down.%d" },  // 定义LLM_TENSOR_FFN_DOWN_EXP常量和对应的字符串，其中%d表示占位符
            { LLM_TENSOR_FFN_UP_EXP,      "blk.%d.ffn_up.%d" },  // 定义LLM_TENSOR_FFN_UP_EXP常量和对应的字符串，其中%d表示占位符
        },
    },
    {
        LLM_ARCH_BAICHUAN,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_TOKEN_EMBD张量对应的名称为"token_embd"
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_OUTPUT_NORM张量对应的名称为"output_norm"
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_OUTPUT张量对应的名称为"output"
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_ROPE_FREQS张量对应的名称为"rope_freqs"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_ATTN_NORM张量对应的名称为"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_ATTN_Q张量对应的名称为"blk.%d.attn_q"
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_ATTN_K张量对应的名称为"blk.%d.attn_k"
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_ATTN_V张量对应的名称为"blk.%d.attn_v"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_ATTN_OUT张量对应的名称为"blk.%d.attn_output"
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_ATTN_ROT_EMBD张量对应的名称为"blk.%d.attn_rot_embd"
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_FFN_NORM张量对应的名称为"blk.%d.ffn_norm"
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_FFN_GATE张量对应的名称为"blk.%d.ffn_gate"
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_FFN_DOWN张量对应的名称为"blk.%d.ffn_down"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_ARCH_BAICHUAN架构下的LLM_TENSOR_FFN_UP张量对应的名称为"blk.%d.ffn_up"
        },
    },
    {
        LLM_ARCH_FALCON,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_TOKEN_EMBD张量对应的名称为"token_embd"
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_OUTPUT_NORM张量对应的名称为"output_norm"
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_OUTPUT张量对应的名称为"output"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_ATTN_NORM张量对应的名称为"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_NORM_2,     "blk.%d.attn_norm_2" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_ATTN_NORM_2张量对应的名称为"blk.%d.attn_norm_2"
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_ATTN_QKV张量对应的名称为"blk.%d.attn_qkv"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_ATTN_OUT张量对应的名称为"blk.%d.attn_output"
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_FFN_DOWN张量对应的名称为"blk.%d.ffn_down"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_ARCH_FALCON架构下的LLM_TENSOR_FFN_UP张量对应的名称为"blk.%d.ffn_up"
        },
    },
    {
        # 定义LLM_ARCH_GPT2架构对应的张量名称和文件名
        LLM_ARCH_GPT2,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },
            { LLM_TENSOR_POS_EMBD,        "position_embd" },
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },
            { LLM_TENSOR_OUTPUT,          "output" },
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
        },
    },
    {
        # 定义LLM_ARCH_GPTJ架构对应的张量名称和文件名
        LLM_ARCH_GPTJ,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },
        },
    },
    {
        # 定义LLM_ARCH_GPTNEOX架构对应的张量名称和文件名
        LLM_ARCH_GPTNEOX,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },
            { LLM_TENSOR_OUTPUT,          "output" },
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },
        },
    },
    {
        LLM_ARCH_PERSIMMON,
        {   # 定义 Persimmon 架构的张量名称映射
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd"},  # token embedding 张量
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm"},  # 输出层归一化张量
            { LLM_TENSOR_OUTPUT,          "output"},  # 输出张量
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm"},  # 注意力层归一化张量
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv"},  # 注意力层 QKV 张量
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output"},  # 注意力层输出张量
            { LLM_TENSOR_ATTN_Q_NORM,     "blk.%d.attn_q_norm"},  # 注意力层 Q 归一化张量
            { LLM_TENSOR_ATTN_K_NORM,     "blk.%d.attn_k_norm"},  # 注意力层 K 归一化张量
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm"},  # Feed Forward 层归一化张量
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down"},  # Feed Forward 层下采样张量
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up"},  # Feed Forward 层上采样张量
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd"},  # 注意力层旋转 embedding 张量
        },
    },
    {
        LLM_ARCH_MPT,
        {   # 定义 MPT 架构的张量名称映射
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # token embedding 张量
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # 输出层归一化张量
            { LLM_TENSOR_OUTPUT,          "output" },  # 输出张量
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # 注意力层归一化张量
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # Feed Forward 层归一化张量
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  # 注意力层 QKV 张量
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # 注意力层输出张量
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # Feed Forward 层下采样张量
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # Feed Forward 层上采样张量
            { LLM_TENSOR_FFN_ACT,         "blk.%d.ffn.act" },  # Feed Forward 层激活张量
        },
    },
    {
        LLM_ARCH_STARCODER,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_ARCH_STARCODER架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_POS_EMBD,        "position_embd" },
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },
            { LLM_TENSOR_OUTPUT,          "output" },
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
        },
    },
    {
        LLM_ARCH_REFACT,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_ARCH_REFACT架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },
            { LLM_TENSOR_OUTPUT,          "output" },
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },
        },
    },
    {
        LLM_ARCH_BLOOM,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_TOKEN_EMBD对应的名称为"token_embd"
            { LLM_TENSOR_TOKEN_EMBD_NORM, "token_embd_norm" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_TOKEN_EMBD_NORM对应的名称为"token_embd_norm"
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_OUTPUT_NORM对应的名称为"output_norm"
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_OUTPUT对应的名称为"output"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_ATTN_NORM对应的名称为"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_ATTN_QKV对应的名称为"blk.%d.attn_qkv"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_ATTN_OUT对应的名称为"blk.%d.attn_output"
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_FFN_NORM对应的名称为"blk.%d.ffn_norm"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_FFN_UP对应的名称为"blk.%d.ffn_up"
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_ARCH_BLOOM架构中的LLM_TENSOR_FFN_DOWN对应的名称为"blk.%d.ffn_down"
        },
    },
    {
        LLM_ARCH_STABLELM,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_TOKEN_EMBD对应的名称为"token_embd"
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_OUTPUT_NORM对应的名称为"output_norm"
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_OUTPUT对应的名称为"output"
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_ROPE_FREQS对应的名称为"rope_freqs"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_ATTN_NORM对应的名称为"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_ATTN_Q对应的名称为"blk.%d.attn_q"
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_ATTN_K对应的名称为"blk.%d.attn_k"
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_ATTN_V对应的名称为"blk.%d.attn_v"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_ATTN_OUT对应的名称为"blk.%d.attn_output"
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_FFN_NORM对应的名称为"blk.%d.ffn_norm"
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_FFN_GATE对应的名称为"blk.%d.ffn_gate"
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_FFN_DOWN对应的名称为"blk.%d.ffn_down"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_ARCH_STABLELM架构中的LLM_TENSOR_FFN_UP对应的名称为"blk.%d.ffn_up"
        },
    },
    {
        LLM_ARCH_QWEN,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_ARCH_QWEN架构下的张量名称和对应的字符串表示
        },
    },
    {
        LLM_ARCH_QWEN2,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_ARCH_QWEN2架构下的张量名称和对应的字符串表示
        },
    },
    {
        # 定义LLM_ARCH_PHI2架构对应的张量名称和对应的字符串表示
        LLM_ARCH_PHI2,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },
            { LLM_TENSOR_OUTPUT,          "output" },
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },
        },
    },
    {
        # 定义LLM_ARCH_PLAMO架构对应的张量名称和对应的字符串表示
        LLM_ARCH_PLAMO,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },
            { LLM_TENSOR_OUTPUT,          "output" },
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd" },
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },
        },
    },
    {
        LLM_ARCH_CODESHELL,  // 定义LLM_ARCH_CODESHELL架构
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_TENSOR_TOKEN_EMBD张量和对应的名称
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_TENSOR_OUTPUT_NORM张量和对应的名称
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_TENSOR_OUTPUT张量和对应的名称
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  // 定义LLM_TENSOR_ROPE_FREQS张量和对应的名称
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_TENSOR_ATTN_NORM张量和对应的名称
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  // 定义LLM_TENSOR_ATTN_Q张量和对应的名称
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  // 定义LLM_TENSOR_ATTN_K张量和对应的名称
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  // 定义LLM_TENSOR_ATTN_V张量和对应的名称
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  // 定义LLM_TENSOR_ATTN_QKV张量和对应的名称
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_TENSOR_ATTN_OUT张量和对应的名称
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd" },  // 定义LLM_TENSOR_ATTN_ROT_EMBD张量和对应的名称
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义LLM_TENSOR_FFN_NORM张量和对应的名称
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  // 定义LLM_TENSOR_FFN_GATE张量和对应的名称
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_TENSOR_FFN_DOWN张量和对应的名称
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_TENSOR_FFN_UP张量和对应的名称
        },
    },
    {
        LLM_ARCH_ORION,  // 定义LLM_ARCH_ORION架构
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义LLM_TENSOR_TOKEN_EMBD张量和对应的名称
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义LLM_TENSOR_OUTPUT_NORM张量和对应的名称
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义LLM_TENSOR_OUTPUT张量和对应的名称
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  // 定义LLM_TENSOR_ROPE_FREQS张量和对应的名称
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义LLM_TENSOR_ATTN_NORM张量和对应的名称
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  // 定义LLM_TENSOR_ATTN_Q张量和对应的名称
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  // 定义LLM_TENSOR_ATTN_K张量和对应的名称
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  // 定义LLM_TENSOR_ATTN_V张量和对应的名称
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义LLM_TENSOR_ATTN_OUT张量和对应的名称
            { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd" },  // 定义LLM_TENSOR_ATTN_ROT_EMBD张量和对应的名称
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义LLM_TENSOR_FFN_NORM张量和对应的名称
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },  // 定义LLM_TENSOR_FFN_GATE张量和对应的名称
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义LLM_TENSOR_FFN_DOWN张量和对应的名称
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义LLM_TENSOR_FFN_UP张量和对应的名称
        },
    },
    {
        # 定义LLM_ARCH_UNKNOWN对应的数据结构
        LLM_ARCH_UNKNOWN,
        {
            # 定义LLM_ARCH_UNKNOWN中LLM_TENSOR_TOKEN_EMBD对应的数据结构
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },
        },
    },
// 结构体 llm_arch 的枚举类型，根据字符串返回对应的枚举值
static llm_arch llm_arch_from_string(const std::string & name) {
    // 遍历枚举类型和字符串的映射
    for (const auto & kv : LLM_ARCH_NAMES) { // NOLINT
        // 如果找到对应的字符串，返回对应的枚举值
        if (kv.second == name) {
            return kv.first;
        }
    }

    // 如果未找到对应的字符串，返回未知枚举值
    return LLM_ARCH_UNKNOWN;
}

// 用于处理 gguf 常量的辅助函数
// 用法：
//
//   const auto tn = LLM_TN(LLM_ARCH_LLAMA);
//
//   std::string name = tn(LLM_TENSOR_OUTPUT);                     -> "output"
//   std::string name = tn(LLM_TENSOR_TOKEN_EMBD, "bias");         -> "token_embd.bias"
//   std::string name = tn(LLM_TENSOR_ATTN_NORM, "weight", 3);     -> "blk.3.attn_norm.weight"
//
struct LLM_TN {
    // 构造函数，初始化架构
    LLM_TN(llm_arch arch) : arch(arch) {}

    // 架构类型
    llm_arch arch;

    // 重载运算符，返回对应张量的名称
    std::string operator()(llm_tensor tensor) const {
        return LLM_TENSOR_NAMES[arch].at(tensor);
    }

    // 重载运算符，返回对应张量的名称加后缀
    std::string operator()(llm_tensor tensor, const std::string & suffix) const {
        return LLM_TENSOR_NAMES[arch].at(tensor) + "." + suffix;
    }

    // 重载运算符，返回对应张量的名称加编号
    std::string operator()(llm_tensor tensor, int bid) const {
        return ::format(LLM_TENSOR_NAMES[arch].at(tensor).c_str(), bid);
    }

    // 重载运算符，返回对应张量的名称加后缀和编号
    std::string operator()(llm_tensor tensor, const std::string & suffix, int bid) const {
        return ::format(LLM_TENSOR_NAMES[arch].at(tensor).c_str(), bid) + "." + suffix;
    }

    // 重载运算符，返回对应张量的名称加后缀、编号和 xid
    std::string operator()(llm_tensor tensor, const std::string & suffix, int bid, int xid) const {
        return ::format(LLM_TENSOR_NAMES[arch].at(tensor).c_str(), bid, xid) + "." + suffix;
    }
};

//
// gguf 辅助函数
//

// 映射不同的绳索缩放类型到字符串
static std::map<int8_t, std::string> LLAMA_ROPE_SCALING_TYPES = {
    { LLAMA_ROPE_SCALING_NONE,   "none"   },
    { LLAMA_ROPE_SCALING_LINEAR, "linear" },
    { LLAMA_ROPE_SCALING_YARN,   "yarn"   },
};

// 根据字符串返回对应的绳索缩放类型
static int8_t llama_rope_scaling_type_from_string(const std::string & name) {
    // 遍历绳索缩放类型和字符串的映射
    for (const auto & kv : LLAMA_ROPE_SCALING_TYPES) {
        // 如果找到对应的字符串，返回对应的绳索缩放类型
        if (kv.second == name) {
            return kv.first;
        }
    }

    // 如果未找到对应的字符串，返回未指定的绳索缩放类型
    return LLAMA_ROPE_SCALING_UNSPECIFIED;
}
// 将 GGUF 数据转换为字符串表示形式
static std::string gguf_data_to_str(enum gguf_type type, const void * data, int i) {
    // 根据不同的数据类型进行转换并返回对应的字符串表示形式
    switch (type) {
        case GGUF_TYPE_UINT8:   return std::to_string(((const uint8_t  *)data)[i]);
        case GGUF_TYPE_INT8:    return std::to_string(((const int8_t   *)data)[i]);
        case GGUF_TYPE_UINT16:  return std::to_string(((const uint16_t *)data)[i]);
        case GGUF_TYPE_INT16:   return std::to_string(((const int16_t  *)data)[i]);
        case GGUF_TYPE_UINT32:  return std::to_string(((const uint32_t *)data)[i]);
        case GGUF_TYPE_INT32:   return std::to_string(((const int32_t  *)data)[i]);
        case GGUF_TYPE_UINT64:  return std::to_string(((const uint64_t *)data)[i]);
        case GGUF_TYPE_INT64:   return std::to_string(((const int64_t  *)data)[i]);
        case GGUF_TYPE_FLOAT32: return std::to_string(((const float    *)data)[i]);
        case GGUF_TYPE_FLOAT64: return std::to_string(((const double   *)data)[i]);
        case GGUF_TYPE_BOOL:    return ((const bool *)data)[i] ? "true" : "false";
        // 处理未知类型的情况
        default:                return format("unknown type %d", type);
    }
}

// 将 GGUF 键值对转换为字符串表示形式
static std::string gguf_kv_to_str(const struct gguf_context * ctx_gguf, int i) {
    // 获取键值对的数据类型
    const enum gguf_type type = gguf_get_kv_type(ctx_gguf, i);
    # 根据不同的类型进行不同的处理
    switch (type) {
        # 如果类型为字符串，则返回对应的字符串值
        case GGUF_TYPE_STRING:
            return gguf_get_val_str(ctx_gguf, i);
        # 如果类型为数组
        case GGUF_TYPE_ARRAY:
            {
                # 获取数组元素的类型
                const enum gguf_type arr_type = gguf_get_arr_type(ctx_gguf, i);
                # 获取数组元素的数量
                int arr_n = gguf_get_arr_n(ctx_gguf, i);
                # 获取数组元素的数据
                const void * data = gguf_get_arr_data(ctx_gguf, i);
                # 创建一个字符串流对象
                std::stringstream ss;
                # 添加左括号到字符串流中
                ss << "[";
                # 遍历数组元素
                for (int j = 0; j < arr_n; j++) {
                    # 如果数组元素类型为字符串
                    if (arr_type == GGUF_TYPE_STRING) {
                        # 获取字符串值
                        std::string val = gguf_get_arr_str(ctx_gguf, i, j);
                        # 转义引号
                        replace_all(val, "\\", "\\\\");
                        replace_all(val, "\"", "\\\"");
                        # 将字符串值添加到字符串流中
                        ss << '"' << val << '"';
                    } else if (arr_type == GGUF_TYPE_ARRAY) {
                        # 如果数组元素类型为数组，则添加占位符
                        ss << "???";
                    } else {
                        # 否则将数组元素转换为字符串并添加到字符串流中
                        ss << gguf_data_to_str(arr_type, data, j);
                    }
                    # 如果不是最后一个数组元素，则添加逗号和空格
                    if (j < arr_n - 1) {
                        ss << ", ";
                    }
                }
                # 添加右括号到字符串流中
                ss << "]";
                # 返回字符串流中的内容
                return ss.str();
            }
        # 默认情况下，将数据转换为字符串并返回
        default:
            return gguf_data_to_str(type, gguf_get_val_data(ctx_gguf, i), 0);
    }
}

//
// ggml helpers
//

// 计算图形的辅助函数，根据给定的图形和线程数计算结果
static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    // 根据图形和线程数创建计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划中有工作量
    if (plan.work_size > 0) {
        // 调整缓冲区大小
        buf.resize(plan.work_size);
        // 将工作数据指针指向缓冲区数据
        plan.work_data = buf.data();
    }

    // 执行图形计算
    ggml_graph_compute(graph, &plan);
}

//
// llama helpers
//

// 如果是 Windows 平台，格式化错误信息
#if defined(_WIN32)
static std::string llama_format_win_err(DWORD err) {
    LPSTR buf;
    // 格式化错误信息
    size_t size = FormatMessageA(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
                                 NULL, err, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPSTR)&buf, 0, NULL);
    // 如果格式化失败
    if (!size) {
        return "FormatMessageA failed";
    }
    // 返回格式化后的错误信息
    std::string ret(buf, size);
    LocalFree(buf);
    return ret;
}
#endif

// 用于延迟初始化的结构体模板
template <typename T>
struct no_init {
    T value;
    no_init() { /* do nothing */ }
};

// llama 文件结构体
struct llama_file {
    // 使用 FILE * 类型，以便在 mmap 时不必重新打开文件
    FILE * fp;
    size_t size;

    // 构造函数，打开文件并获取文件大小
    llama_file(const char * fname, const char * mode) {
        fp = std::fopen(fname, mode);
        // 如果打开文件失败
        if (fp == NULL) {
            throw std::runtime_error(format("failed to open %s: %s", fname, strerror(errno)));
        }
        // 定位到文件末尾获取文件大小
        seek(0, SEEK_END);
        size = tell();
        // 定位回文件开头
        seek(0, SEEK_SET);
    }

    // 获取当前文件指针位置
    size_t tell() const {
#ifdef _WIN32
        __int64 ret = _ftelli64(fp);
#else
        long ret = std::ftell(fp);
#endif
        GGML_ASSERT(ret != -1); // 这不应该失败
        return (size_t) ret;
    }

    // 移动文件指针到指定位置
    void seek(size_t offset, int whence) const {
#ifdef _WIN32
        int ret = _fseeki64(fp, (__int64) offset, whence);
#else
        int ret = std::fseek(fp, (long) offset, whence);
#endif
        GGML_ASSERT(ret == 0); // 同上
    }
    // 读取指定长度的原始数据到指针所指向的内存中
    void read_raw(void * ptr, size_t len) const {
        // 如果长度为0，则直接返回
        if (len == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 从文件指针中读取指定长度的数据到指定内存中
        std::size_t ret = std::fread(ptr, len, 1, fp);
        // 如果读取出错，则抛出运行时错误
        if (ferror(fp)) {
            throw std::runtime_error(format("read error: %s", strerror(errno)));
        }
        // 如果读取的数据长度不符合预期，则抛出运行时错误
        if (ret != 1) {
            throw std::runtime_error("unexpectedly reached end of file");
        }
    }

    // 读取一个32位无符号整数
    uint32_t read_u32() const {
        uint32_t ret;
        // 调用read_raw函数读取数据到ret中
        read_raw(&ret, sizeof(ret));
        return ret;
    }

    // 将指定长度的原始数据写入文件
    void write_raw(const void * ptr, size_t len) const {
        // 如果长度为0，则直接返回
        if (len == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 将指定长度的数据写入文件
        size_t ret = std::fwrite(ptr, len, 1, fp);
        // 如果写入数据长度不符合预期，则抛出运行时错误
        if (ret != 1) {
            throw std::runtime_error(format("write error: %s", strerror(errno)));
        }
    }

    // 将一个32位无符号整数写入文件
    void write_u32(std::uint32_t val) const {
        // 调用write_raw函数将val写入文件
        write_raw(&val, sizeof(val));
    }

    // 析构函数，关闭文件指针
    ~llama_file() {
        if (fp) {
            std::fclose(fp);
        }
    }
};

// 定义结构体 llama_mmap
struct llama_mmap {
    // 内存映射的地址和大小
    void * addr;
    size_t size;

    // 禁用拷贝构造函数
    llama_mmap(const llama_mmap &) = delete;

#ifdef _POSIX_MAPPED_FILES
    // 是否支持内存映射
    static constexpr bool SUPPORTED = true;

    // 映射的片段列表 (起始偏移量, 结束偏移量)
    std::vector<std::pair<size_t, size_t>> mapped_fragments;

    // 构造函数，根据文件和参数创建内存映射
    llama_mmap(struct llama_file * file, size_t prefetch = (size_t) -1 /* -1 = max value */, bool numa = false) {
        // 设置内存映射的大小为文件大小
        size = file->size;
        // 获取文件描述符
        int fd = fileno(file->fp);
        // 设置映射标志为共享映射
        int flags = MAP_SHARED;
        // 在 NUMA 系统上，预取/预读会影响性能
        if (numa) { prefetch = 0; }
#ifdef __linux__
        // 建议内核按顺序读取文件 (增加预读)
        if (posix_fadvise(fd, 0, 0, POSIX_FADV_SEQUENTIAL)) {
            LLAMA_LOG_WARN("warning: posix_fadvise(.., POSIX_FADV_SEQUENTIAL) failed: %s\n",
                    strerror(errno));
        }
        // 如果需要预取，则设置标志为 MAP_POPULATE
        if (prefetch) { flags |= MAP_POPULATE; }
#endif
        // 创建内存映射
        addr = mmap(NULL, file->size, PROT_READ, flags, fd, 0);
        // 检查映射是否成功
        if (addr == MAP_FAILED) { // NOLINT
            throw std::runtime_error(format("mmap failed: %s", strerror(errno)));
        }

        // 如果需要预取
        if (prefetch > 0) {
            // 建议内核预加载映射的内存
            if (posix_madvise(addr, std::min(file->size, prefetch), POSIX_MADV_WILLNEED)) {
                LLAMA_LOG_WARN("warning: posix_madvise(.., POSIX_MADV_WILLNEED) failed: %s\n",
                        strerror(errno));
            }
        }
        // 如果在 NUMA 系统上
        if (numa) {
            // 建议内核不要使用预读
            // (因为下一页可能不属于同一节点)
            if (posix_madvise(addr, file->size, POSIX_MADV_RANDOM)) {
                LLAMA_LOG_WARN("warning: posix_madvise(.., POSIX_MADV_RANDOM) failed: %s\n",
                        strerror(errno));
            }
        }

        // 初始化映射片段列表
        mapped_fragments.emplace_back(0, file->size);
    }
    // 将指定范围[first, last)对齐到页边界
    static void align_range(size_t * first, size_t * last, size_t page_size) {
        // 计算 first 到下一页的偏移量
        size_t offset_in_page = *first & (page_size - 1);
        size_t offset_to_page = offset_in_page == 0 ? 0 : page_size - offset_in_page;
        *first += offset_to_page;

        // 将 last 对齐到上一页
        *last = *last & ~(page_size - 1);

        // 如果 last 小于等于 first，则将 last 设为 first
        if (*last <= *first) {
            *last = *first;
        }
    }

    // 部分取消映射范围在[first, last)的文件
    ~llama_mmap() {
        // 遍历已映射的片段
        for (const auto & frag : mapped_fragments) {
            // 使用 munmap 取消映射指定地址范围的内存
            if (munmap((char *) addr + frag.first, frag.second - frag.first)) {
                // 如果取消映射失败，则输出警告信息
                LLAMA_LOG_WARN("warning: munmap failed: %s\n", strerror(errno));
            }
        }
    }
#elif defined(_WIN32)
    // 如果是在 Windows 平台下编译，则支持为 true
    static constexpr bool SUPPORTED = true;

    // 在 Windows 平台下使用内存映射，设置预取大小和 NUMA 参数
    llama_mmap(struct llama_file * file, size_t prefetch = (size_t) -1, bool numa = false) {
        // 忽略 NUMA 参数
        GGML_UNUSED(numa);

        // 获取文件大小
        size = file->size;

        // 获取文件句柄
        HANDLE hFile = (HANDLE) _get_osfhandle(_fileno(file->fp));

        // 创建文件映射
        HANDLE hMapping = CreateFileMappingA(hFile, NULL, PAGE_READONLY, 0, 0, NULL);

        // 检查文件映射是否成功
        if (hMapping == NULL) {
            DWORD error = GetLastError();
            throw std::runtime_error(format("CreateFileMappingA failed: %s", llama_format_win_err(error).c_str()));
        }

        // 映射文件到内存
        addr = MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);
        DWORD error = GetLastError();
        // 关闭文件映射句柄
        CloseHandle(hMapping);

        // 检查文件映射是否成功
        if (addr == NULL) {
            throw std::runtime_error(format("MapViewOfFile failed: %s", llama_format_win_err(error).c_str()));
        }

        // 如果设置了预取大小
        if (prefetch > 0) {
            // 如果 Windows 版本大于等于 8
#if _WIN32_WINNT >= 0x602
            // 动态加载 PrefetchVirtualMemory 函数，该函数仅在 Windows 8 及以上版本存在
            BOOL (WINAPI *pPrefetchVirtualMemory) (HANDLE, ULONG_PTR, PWIN32_MEMORY_RANGE_ENTRY, ULONG);
            HMODULE hKernel32 = GetModuleHandleW(L"kernel32.dll");

            // 可能在 Windows 8 之前的系统上失败
            pPrefetchVirtualMemory = reinterpret_cast<decltype(pPrefetchVirtualMemory)> (GetProcAddress(hKernel32, "PrefetchVirtualMemory"));

            if (pPrefetchVirtualMemory) {
                // 建议内核预加载映射的内存
                WIN32_MEMORY_RANGE_ENTRY range;
                range.VirtualAddress = addr;
                range.NumberOfBytes = (SIZE_T) std::min(size, prefetch);
                if (!pPrefetchVirtualMemory(GetCurrentProcess(), 1, &range, 0)) {
                    LLAMA_LOG_WARN("warning: PrefetchVirtualMemory failed: %s\n",
                            llama_format_win_err(GetLastError()).c_str());
                }
            }
#ifdef _POSIX_MEMLOCK_RANGE
    // 定义静态常量 SUPPORTED 为 true，表示支持 POSIX 内存锁定范围
    static constexpr bool SUPPORTED = true;
#else
    // 定义静态常量 SUPPORTED 为 false，表示不支持 POSIX 内存锁定范围
    static constexpr bool SUPPORTED = false;
#endif

// 表示使用 mmap 映射文件的类
struct llama_mmap {
    // 构造函数，根据文件和参数创建 mmap 对象
    llama_mmap(struct llama_file * file, size_t prefetch = -1, bool numa = false) {
        // 未使用的参数
        GGML_UNUSED(file);
        GGML_UNUSED(prefetch);
        GGML_UNUSED(numa);

        // 抛出异常，表示不支持 mmap
        throw std::runtime_error("mmap not supported");
    }

    // 解除映射指定范围的片段
    void unmap_fragment(size_t first, size_t last) {
        // 未支持的操作
        GGML_UNUSED(first);
        GGML_UNUSED(last);

        // 抛出异常，表示不支持 mmap
        throw std::runtime_error("mmap not supported");
    }
};

// 表示一些内存区域被使用 mlock 或 VirtualLock 锁定的类；在销毁时会自动解锁
struct llama_mlock {
    void * addr = NULL;
    size_t size = 0;

    bool failed_already = false;

    // 默认构造函数
    llama_mlock() {}
    // 禁用拷贝构造函数
    llama_mlock(const llama_mlock &) = delete;

    // 析构函数，如果有大小，则解锁内存
    ~llama_mlock() {
        if (size) {
            raw_unlock(addr, size);
        }
    }

    // 初始化内存地址
    void init(void * ptr) {
        // 断言地址和大小为 0
        GGML_ASSERT(addr == NULL && size == 0); // NOLINT
        addr = ptr;
    }

    // 扩展内存大小到目标大小
    void grow_to(size_t target_size) {
        GGML_ASSERT(addr);
        if (failed_already) {
            return;
        }
        size_t granularity = lock_granularity();
        target_size = (target_size + granularity - 1) & ~(granularity - 1);
        if (target_size > size) {
            if (raw_lock((uint8_t *) addr + size, target_size - size)) {
                size = target_size;
            } else {
                failed_already = true;
            }
        }
    }
};
    // 返回系统页面大小
    static size_t lock_granularity() {
        return (size_t) sysconf(_SC_PAGESIZE);
    }

    #ifdef __APPLE__
        // 定义针对苹果系统的内存锁定建议
        #define MLOCK_SUGGESTION \
            "Try increasing the sysctl values 'vm.user_wire_limit' and 'vm.global_user_wire_limit' and/or " \
            "decreasing 'vm.global_no_user_wire_amount'.  Also try increasing RLIMIT_MLOCK (ulimit -l).\n"
    #else
        // 定义针对非苹果系统的内存锁定建议
        #define MLOCK_SUGGESTION \
            "Try increasing RLIMIT_MLOCK ('ulimit -l' as root).\n"
    #endif

    // 锁定内存区域
    bool raw_lock(const void * addr, size_t size) const {
        // 尝试锁定内存区域，如果成功返回 true
        if (!mlock(addr, size)) {
            return true;
        }

        // 获取错误信息
        char* errmsg = std::strerror(errno);
        // 检查是否建议增加内存限制
        bool suggest = (errno == ENOMEM);

        // 检查资源限制是否正常
        struct rlimit lock_limit;
        if (suggest && getrlimit(RLIMIT_MEMLOCK, &lock_limit)) {
            suggest = false;
        }
        if (suggest && (lock_limit.rlim_max > lock_limit.rlim_cur + size)) {
            suggest = false;
        }

        // 输出警告信息
        LLAMA_LOG_WARN("warning: failed to mlock %zu-byte buffer (after previously locking %zu bytes): %s\n%s",
                size, this->size, errmsg, suggest ? MLOCK_SUGGESTION : "");
        return false;
    }

    // 取消锁定内存区域
    #undef MLOCK_SUGGESTION
    static void raw_unlock(void * addr, size_t size) {
        // 尝试解锁内存区域，如果失败输出警告信息
        if (munlock(addr, size)) {
            LLAMA_LOG_WARN("warning: failed to munlock buffer: %s\n", std::strerror(errno));
        }
    }
#elif defined(_WIN32)
    // 如果是在 Windows 平台下编译，则支持该功能
    static constexpr bool SUPPORTED = true;

    // 获取系统信息，返回页面大小
    static size_t lock_granularity() {
        SYSTEM_INFO si;
        GetSystemInfo(&si);
        return (size_t) si.dwPageSize;
    }

    // 在 Windows 平台下使用 VirtualLock 锁定内存区域
    bool raw_lock(void * ptr, size_t len) const {
        for (int tries = 1; ; tries++) {
            // 尝试使用 VirtualLock 锁定内存区域
            if (VirtualLock(ptr, len)) {
                return true;
            }
            // 如果第一次尝试失败，进行第二次尝试
            if (tries == 2) {
                // 输出警告信息，并返回失败
                LLAMA_LOG_WARN("warning: failed to VirtualLock %zu-byte buffer (after previously locking %zu bytes): %s\n",
                    len, size, llama_format_win_err(GetLastError()).c_str());
                return false;
            }

            // 获取当前进程的工作集大小
            SIZE_T min_ws_size, max_ws_size;
            if (!GetProcessWorkingSetSize(GetCurrentProcess(), &min_ws_size, &max_ws_size)) {
                // 输出警告信息，并返回失败
                LLAMA_LOG_WARN("warning: GetProcessWorkingSetSize failed: %s\n",
                        llama_format_win_err(GetLastError()).c_str());
                return false;
            }
            // 增加工作集大小，然后再次尝试
            size_t increment = len + 1048576;
            min_ws_size += increment;
            max_ws_size += increment;
            if (!SetProcessWorkingSetSize(GetCurrentProcess(), min_ws_size, max_ws_size)) {
                // 输出警告信息，并返回失败
                LLAMA_LOG_WARN("warning: SetProcessWorkingSetSize failed: %s\n",
                        llama_format_win_err(GetLastError()).c_str());
                return false;
            }
        }
    }
    // 定义一个静态函数，用于解锁指定内存区域
    static void raw_unlock(void * ptr, size_t len) {
        // 如果无法解锁指定内存区域，则输出警告信息
        if (!VirtualUnlock(ptr, len)) {
            // 输出警告信息，包含失败原因
            LLAMA_LOG_WARN("warning: failed to VirtualUnlock buffer: %s\n",
                    llama_format_win_err(GetLastError()).c_str());
        }
    }
#else
    // 如果不满足条件，则设置 SUPPORTED 为 false
    static constexpr bool SUPPORTED = false;

    // 返回锁定粒度为 65536
    static size_t lock_granularity() {
        return (size_t) 65536;
    }

    // 锁定指定地址和长度的内存，如果不支持则输出警告信息并返回 false
    bool raw_lock(const void * addr, size_t len) const {
        LLAMA_LOG_WARN("warning: mlock not supported on this system\n");
        return false;
    }

    // 解锁指定地址和长度的内存
    static void raw_unlock(const void * addr, size_t len) {}
#endif
};

// 将 llama_token 转换为字符串
static std::string llama_token_to_piece(const struct llama_context * ctx, llama_token token) {
    // 初始化一个长度为 8 的字符向量
    std::vector<char> result(8, 0);
    // 将 llama_token 转换为字符串
    const int n_tokens = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
    // 如果转换失败，则重新调整向量大小并再次尝试转换
    if (n_tokens < 0) {
        result.resize(-n_tokens);
        int check = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
        GGML_ASSERT(check == -n_tokens);
    }
    else {
        result.resize(n_tokens);
    }

    return std::string(result.data(), result.size());
}

// 返回 CPU 默认的缓冲区类型
static ggml_backend_buffer_type_t llama_default_buffer_type_cpu(bool host_buffer) {
    ggml_backend_buffer_type_t buft = nullptr;

#if defined(GGML_USE_CUBLAS)
    // 如果使用 CUBLAS，则根据 host_buffer 决定返回的缓冲区类型
    if (host_buffer) {
        buft = ggml_backend_cuda_host_buffer_type();
    }
#elif defined(GGML_USE_SYCL)
    // 如果使用 SYCL，则返回 SYCL 的主机缓冲区类型
    buft = ggml_backend_sycl_host_buffer_type();
#elif defined(GGML_USE_CPU_HBM)
    // 如果使用 CPU_HBM，则返回 CPU_HBM 的缓冲区类型
    buft = ggml_backend_cpu_hbm_buffer_type();
#elif defined(GGML_USE_VULKAN)
    // 如果使用 VULKAN，则根据 host_buffer 决定返回的缓冲区类型
    if (host_buffer) {
        buft = ggml_backend_vk_host_buffer_type();
    }
#endif

    // 如果没有匹配到特定的缓冲区类型，则返回 CPU 的缓冲区类型
    if (buft == nullptr) {
        buft = ggml_backend_cpu_buffer_type();
    }
    return buft;

    GGML_UNUSED(host_buffer);
}

// 返回卸载默认的缓冲区类型
static ggml_backend_buffer_type_t llama_default_buffer_type_offload(int gpu) {
    ggml_backend_buffer_type_t buft = nullptr;

#ifdef GGML_USE_METAL
    // 如果使用 METAL，则返回 METAL 的缓冲区类型
    buft = ggml_backend_metal_buffer_type();
#elif defined(GGML_USE_CUBLAS)
    // 如果使用 CUBLAS，则返回指定 GPU 的 CUDA 缓冲区类型
    buft = ggml_backend_cuda_buffer_type(gpu);
#elif defined(GGML_USE_VULKAN)
    // 如果使用 VULKAN，则返回 VULKAN 的缓冲区类型
    buft = ggml_backend_vk_buffer_type();
// 如果定义了 GGML_USE_SYCL，则使用 SYCL 后端的缓冲区类型
    buft = ggml_backend_sycl_buffer_type(gpu);
// 如果定义了 GGML_USE_CLBLAST，则使用 CLBLAST 后端的缓冲区类型
    buft = ggml_backend_opencl_buffer_type();

// 如果缓冲区类型为空指针，则使用 CPU 默认缓冲区类型
    if (buft == nullptr) {
        buft = llama_default_buffer_type_cpu(true);
    }
    // 返回缓冲区类型
    return buft;

    GGML_UNUSED(gpu);
}

// 根据是否有多个 GPU，以及张量分割信息，选择默认的缓冲区类型
static ggml_backend_buffer_type_t llama_default_buffer_type_split(int fallback_gpu, const float * tensor_split) {
    ggml_backend_buffer_type_t buft = nullptr;

// 如果定义了 GGML_USE_CUBLAS 并且有多个 CUDA 设备，则使用 CUDA 分割缓冲区类型
    if (ggml_backend_cuda_get_device_count() > 1) {
        buft = ggml_backend_cuda_split_buffer_type(tensor_split);
    }

// 如果缓冲区类型为空指针，则使用默认的离线缓冲区类型
    if (buft == nullptr) {
        buft = llama_default_buffer_type_offload(fallback_gpu);
    }
    // 返回缓冲区类型
    return buft;

    GGML_UNUSED(tensor_split);
}

//
// 全局变量
//

// 定义 llama_state 结构体
struct llama_state {
    // 构造函数
    llama_state() {
// 如果定义了 GGML_USE_METAL，则设置 Metal 后端的日志回调函数
        ggml_backend_metal_log_set_callback(log_callback, log_callback_user_data);
    }

    // 全局保存日志回调函数
    ggml_log_callback log_callback = llama_log_callback_default;
    void * log_callback_user_data = nullptr;
};

// 全局 llama_state 实例
static llama_state g_state;

// 可用的 llama 模型
enum e_model {
    MODEL_UNKNOWN,
    MODEL_0_5B,
    MODEL_1B,
    MODEL_3B,
    MODEL_4B,
    MODEL_7B,
    MODEL_8B,
    MODEL_13B,
    MODEL_14B,
    MODEL_15B,
    MODEL_30B,
    MODEL_34B,
    MODEL_40B,
    MODEL_65B,
    MODEL_70B,
    MODEL_SMALL,
    MODEL_MEDIUM,
    MODEL_LARGE,
    MODEL_XL,
};

// 定义常量
static const size_t kiB = 1024;
static const size_t MiB = 1024*kiB;
static const size_t GiB = 1024*MiB;

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
    uint32_t n_embd_head_k; // 键的维度（d_k）。假设查询的维度相同，但有 n_head 个查询头，只有 n_head_kv 个键值头
    // 值的维度（d_v）或者 n_embd_head 的维度
    uint32_t n_embd_head_v;
    // 前馈神经网络的隐藏层维度
    uint32_t n_ff;
    // 专家数量，默认为 0
    uint32_t n_expert = 0;
    // 已使用的专家数量，默认为 0

    uint32_t n_expert_used = 0;

    // 归一化的 epsilon 值
    float f_norm_eps;
    // RMS 归一化的 epsilon 值

    float f_norm_rms_eps;

    // 训练时的绳索频率基础值
    float    rope_freq_base_train;
    // 训练时的绳索频率缩放值
    float    rope_freq_scale_train;
    // 原始上下文的纱线数量
    uint32_t n_yarn_orig_ctx;
    // 训练时的绳索缩放类型
    int8_t   rope_scaling_type_train : 3;
    // 是否对绳索进行微调

    bool     rope_finetuned : 1;

    // 用于限制 k, q, v 的值
    float f_clamp_kqv;
    // 最大的 alibi 偏差值

    float f_max_alibi_bias;
    # 重载不等于运算符，比较两个 llama_hparams 对象是否不相等
    bool operator!=(const llama_hparams & other) const {
        # 检查各个参数是否相等，若有不相等的情况则返回 true
        if (this->vocab_only    != other.vocab_only)    return true;
        if (this->n_vocab       != other.n_vocab)       return true;
        if (this->n_ctx_train   != other.n_ctx_train)   return true;
        if (this->n_embd        != other.n_embd)        return true;
        if (this->n_head        != other.n_head)        return true;
        if (this->n_head_kv     != other.n_head_kv)     return true;
        if (this->n_layer       != other.n_layer)       return true;
        if (this->n_rot         != other.n_rot)         return true;
        if (this->n_embd_head_k != other.n_embd_head_k) return true;
        if (this->n_embd_head_v != other.n_embd_head_v) return true;
        if (this->n_ff          != other.n_ff)          return true;
        if (this->n_expert      != other.n_expert)      return true;
        if (this->n_expert_used != other.n_expert_used) return true;

        if (this->rope_finetuned  != other.rope_finetuned)  return true;
        if (this->n_yarn_orig_ctx != other.n_yarn_orig_ctx) return true;

        # 定义浮点数比较的精度
        const float EPSILON = 1e-9f;

        # 检查浮点数是否在精度范围内相等，若不相等则返回 true
        if (!is_float_close(this->f_norm_eps,            other.f_norm_eps,            EPSILON)) return true;
        if (!is_float_close(this->f_norm_rms_eps,        other.f_norm_rms_eps,        EPSILON)) return true;
        if (!is_float_close(this->rope_freq_base_train,  other.rope_freq_base_train,  EPSILON)) return true;
        if (!is_float_close(this->rope_freq_scale_train, other.rope_freq_scale_train, EPSILON)) return true;

        # 若所有参数相等，则返回 false
        return false;
    }

    # 返回 n_head 除以 n_head_kv 的结果
    uint32_t n_gqa() const {
        return n_head/n_head_kv;
    }

    # 返回 key embeddings 在所有 k-v heads 中的维度
    uint32_t n_embd_k_gqa() const { // dimension of key embeddings across all k-v heads
        return n_embd_head_k * n_head_kv;
    }

    # 返回 value embeddings 在所有 k-v heads 中的维度
    uint32_t n_embd_v_gqa() const { // dimension of value embeddings across all k-v heads
        return n_embd_head_v * n_head_kv;
    }
// 结构体定义，用于存储 LLAMA 模型的参数
struct llama_cparams {
    uint32_t n_ctx;       // 推理过程中使用的上下文大小
    uint32_t n_batch;
    uint32_t n_threads;       // 用于生成的线程数
    uint32_t n_threads_batch; // 用于批处理的线程数

    float    rope_freq_base;
    float    rope_freq_scale;

    uint32_t n_yarn_orig_ctx;
    // 这些超参数在 GGUF 中没有暴露，因为所有现有的 YaRN 模型对它们使用相同的值。
    float yarn_ext_factor;
    float yarn_attn_factor;
    float yarn_beta_fast;
    float yarn_beta_slow;

    bool mul_mat_q;
    bool offload_kqv;

    ggml_backend_sched_eval_callback cb_eval;
    void * cb_eval_user_data;
};

// 结构体定义，用于存储 LLAMA 模型的层信息
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
    struct ggml_tensor * bq;
    struct ggml_tensor * bk;
    struct ggml_tensor * bv;
    struct ggml_tensor * bo;
    struct ggml_tensor * bqkv;

    // 归一化
    struct ggml_tensor * ffn_norm;
    struct ggml_tensor * ffn_norm_b;

    // 前馈神经网络
    struct ggml_tensor * ffn_gate; // w1
    struct ggml_tensor * ffn_down; // w2
    struct ggml_tensor * ffn_up;   // w3

    // 前馈神经网络 MoE
    struct ggml_tensor * ffn_gate_inp;
    struct ggml_tensor * ffn_gate_exp[LLAMA_MAX_EXPERTS];
    struct ggml_tensor * ffn_down_exp[LLAMA_MAX_EXPERTS];
    struct ggml_tensor * ffn_up_exp  [LLAMA_MAX_EXPERTS];

    // 前馈神经网络偏置
    struct ggml_tensor * ffn_down_b; // b2
    struct ggml_tensor * ffn_up_b;   // b3
    struct ggml_tensor * ffn_act;
};
// 定义了一个名为 llama_kv_cell 的结构体，用于表示 LLAMA 键值对的单元格
struct llama_kv_cell {
    llama_pos pos   = -1; // 单元格的位置，默认为 -1
    llama_pos delta = 0; // 单元格的增量，默认为 0

    std::set<llama_seq_id> seq_id; // 存储序列 ID 的集合

    // 检查是否存在特定的序列 ID
    bool has_seq_id(const llama_seq_id & id) const {
        return seq_id.find(id) != seq_id.end();
    }
};

// 定义了一个名为 llama_kv_cache 的结构体，用于表示缓存的 KV 数据的环形缓冲区
struct llama_kv_cache {
    bool has_shift = false; // 是否有移位操作

    // 注意：head 的值不仅用于优化搜索空闲的 KV 插槽，llama_decode_internal 也使用它，
    // 因此在分配插槽后不能随意更改它
    uint32_t head = 0; // 头部位置
    uint32_t size = 0; // 大小
    uint32_t used = 0; // 使用的单元格数量（至少有一个序列 ID）

    // 每次构建图之前计算的值
    uint32_t n = 0;

    std::vector<llama_kv_cell> cells; // KV 单元格数组

    std::vector<struct ggml_tensor *> k_l; // 每层的键
    std::vector<struct ggml_tensor *> v_l; // 每层的值

    std::vector<struct ggml_context *> ctxs; // 上下文数组
    std::vector<ggml_backend_buffer_t> bufs; // 后端缓冲区数组

    // 计算所有缓冲区的总大小
    size_t total_size() const {
        size_t size = 0;
        for (ggml_backend_buffer_t buf : bufs) {
            size += ggml_backend_buffer_get_size(buf);
        }
        return size;
    }

    // 析构函数，释放上下文和后端缓冲区
    ~llama_kv_cache() {
        for (struct ggml_context * ctx : ctxs) {
            ggml_free(ctx);
        }
        for (ggml_backend_buffer_t buf : bufs) {
            ggml_backend_buffer_free(buf);
        }
    }
};

// 定义了一个名为 llama_vocab 的结构体，用于表示 LLAMA 词汇表
struct llama_vocab {
    using id    = int32_t; // ID 类型为 int32_t
    using token = std::string; // 标记类型为字符串
    using ttype = llama_token_type; // 标记类型为 LLAMA 标记类型

    // 标记数据结构
    struct token_data {
        token text; // 文本
        float score; // 分数
        ttype type; // 类型
    };

    enum llama_vocab_type type = LLAMA_VOCAB_TYPE_SPM; // LLAMA 词汇表类型

    std::unordered_map<token, id> token_to_id; // 标记到 ID 的映射
    std::vector<token_data>       id_to_token; // ID 到标记数据的映射

    std::unordered_map<token, id> special_tokens_cache; // 特殊标记缓存

    std::map<std::pair<std::string, std::string>, int> bpe_ranks; // BPE 排名映射

    // 默认的 LLAMA 特殊标记
    id special_bos_id = 1; // 开始标记 ID
    id special_eos_id = 2; // 结束标记 ID
    id special_unk_id = 0; // 未知标记 ID
    id special_sep_id = -1; // 分隔标记 ID
    id special_pad_id = -1; // 填充标记 ID
}
    // 特殊标记，用于指示是否在词的开头添加特殊标记
    int special_add_bos = -1; // -1 未知，1 添加，0 不添加
    // 特殊标记，用于指示是否在词的结尾添加特殊标记
    int special_add_eos = -1; // -1 未知，1 添加，0 不添加

    // 换行符的标识
    id linefeed_id       = 13;
    // 特殊前缀的标识
    id special_prefix_id = 32007;
    // 特殊中间的标识
    id special_middle_id = 32009;
    // 特殊后缀的标识
    id special_suffix_id = 32008;
    // 特殊结束的标识
    id special_eot_id    = 32010;

    // 查找两个词的 BPE 排名
    int find_bpe_rank(const std::string & token_left, const std::string & token_right) const {
        // 断言左侧词中不包含空格和换行符
        GGML_ASSERT(token_left.find(' ') == std::string::npos);
        GGML_ASSERT(token_left.find('\n') == std::string::npos);
        // 断言右侧词中不包含空格和换行符
        GGML_ASSERT(token_right.find(' ') == std::string::npos);
        GGML_ASSERT(token_right.find('\n') == std::string::npos);

        // 在 BPE 排名映射中查找给定的左右词对
        auto it = bpe_ranks.find(std::make_pair(token_left, token_right));
        // 如果未找到，则返回 -1
        if (it == bpe_ranks.end()) {
            return -1;
        }

        // 返回找到的 BPE 排名
        return it->second;
    }
// 结构体定义，用于存储 LLAMA 模型的相关信息
struct llama_model {
    // 模型类型，默认为未知
    e_model     type  = MODEL_UNKNOWN;
    // 模型架构，默认为未知
    llm_arch    arch  = LLM_ARCH_UNKNOWN;
    // 模型数据类型，默认为所有数据类型为单精度浮点数
    llama_ftype ftype = LLAMA_FTYPE_ALL_F32;

    // 模型名称，默认为 "n/a"
    std::string name = "n/a";

    // 模型超参数
    llama_hparams hparams = {};
    // 模型词汇表
    llama_vocab   vocab;

    // 模型中的各种张量
    struct ggml_tensor * tok_embd;
    struct ggml_tensor * pos_embd;
    struct ggml_tensor * tok_norm;
    struct ggml_tensor * tok_norm_b;
    struct ggml_tensor * output_norm;
    struct ggml_tensor * output_norm_b;
    struct ggml_tensor * output;
    struct ggml_tensor * output_b;

    // 模型中的层
    std::vector<llama_layer> layers;

    // 模型分割模式
    llama_split_mode split_mode;
    // 主 GPU
    int main_gpu;
    // GPU 层数
    int n_gpu_layers;

    // gguf 元数据
    std::unordered_map<std::string, std::string> gguf_kv;

    // 层到缓冲类型的映射
    struct layer_buft {
        layer_buft() : buft_matrix(nullptr), buft(nullptr) {}
        layer_buft(ggml_backend_buffer_type_t matrix) : buft_matrix(matrix), buft(matrix) {}
        layer_buft(ggml_backend_buffer_type_t matrix, ggml_backend_buffer_type_t other) : buft_matrix(matrix), buft(other) {}

        ggml_backend_buffer_type_t buft_matrix; // 仅用于矩阵 - 用于分割缓冲区和仅支持矩阵乘法的后端
        ggml_backend_buffer_type_t buft;        // 其他类型
    };

    // 输入缓冲类型
    layer_buft buft_input;
    // 输出缓冲类型
    layer_buft buft_output;
    // 层缓冲类型
    std::vector<layer_buft> buft_layer;

    // 存储模型张量元数据的上下文
    std::vector<struct ggml_context *> ctxs;

    // 存储模型张量数据的内存缓冲区
    std::vector<ggml_backend_buffer_t> bufs;

    // 模型内存映射文件
    std::unique_ptr<llama_mmap> mapping;

    // 表示可能被锁定在内存中的数据对象
    std::vector<std::unique_ptr<llama_mlock>> mlock_bufs;
    llama_mlock mlock_mmap;

    // 仅用于量化统计
    std::vector<std::pair<std::string, struct ggml_tensor *>> tensors_by_name;

    // 加载时间
    int64_t t_load_us = 0;
    // 启动时间
    int64_t t_start_us = 0;
};
    # 析构函数，用于释放内存资源
    ~llama_model() {
        # 遍历上下文列表，释放每个上下文的内存资源
        for (struct ggml_context * ctx : ctxs) {
            ggml_free(ctx);
        }
        # 遍历缓冲区列表，释放每个缓冲区的内存资源
        for (ggml_backend_buffer_t buf : bufs) {
            ggml_backend_buffer_free(buf);
        }
    }
// 结构体 llama_context，包含了模型相关的上下文信息
struct llama_context {
    // 构造函数，初始化 llama_context 对象
    llama_context(const llama_model & model) : model(model), t_start_us(model.t_start_us), t_load_us(model.t_load_us) {}
    // 析构函数，释放资源
    ~llama_context() {
        // 释放调度器资源
        ggml_backend_sched_free(sched);

        // 释放所有后端资源
        for (ggml_backend_t backend : backends) {
            ggml_backend_free(backend);
        }

        // 释放输入缓冲区资源
        ggml_backend_buffer_free(buf_input);
        ggml_free(ctx_input);
    }

    // 参数配置
    llama_cparams cparams;

    // 后端列表
    std::vector<ggml_backend_t> backends;
#ifdef GGML_USE_METAL
    ggml_backend_t backend_metal = nullptr;
#endif
    ggml_backend_t backend_cpu = nullptr;

    // 模型引用
    const llama_model & model;

    // 自注意力机制的键值缓存
    struct llama_kv_cache kv_self;

    // 随机数生成器
    std::mt19937 rng;

    // 是否已经评估过一次
    bool has_evaluated_once = false;

    // 时间统计
    int64_t t_start_us;
    int64_t t_load_us;
    int64_t t_sample_us = 0;
    int64_t t_p_eval_us = 0;
    int64_t t_eval_us   = 0;

    // 样本数量
    int32_t n_sample = 0; // number of tokens sampled
    int32_t n_p_eval = 0; // number of tokens in eval calls for the prompt (with batch size > 1)
    int32_t n_eval   = 0; // number of eval calls

    // 解码输出
    std::vector<float> logits;
#ifndef NDEBUG
    // 调试模式下的标志数组
    std::vector<bool>  logits_valid;
#endif
    bool logits_all = false;

    // 输入嵌入
    std::vector<float> embedding;

    // 用于评估模型的内存缓冲区
    std::vector<uint8_t> buf_compute_meta;
    ggml_backend_sched_t sched = nullptr;
    // 输入张量的分配器
    ggml_tallocr * alloc = nullptr;

    // 输入张量
    ggml_backend_buffer_t buf_input = nullptr;
    ggml_context * ctx_input = nullptr;
    struct ggml_tensor * inp_tokens;    // I32 [n_batch]
    struct ggml_tensor * inp_embd;      // F32 [n_embd, n_batch]
    struct ggml_tensor * inp_pos;       // I32 [n_batch]
    struct ggml_tensor * inp_KQ_mask;   // F32 [n_ctx, n_batch]
};
    // 定义一个指向 ggml_tensor 结构体的指针，表示输入的 K_shift 张量，数据类型为 I32，长度为 n_ctx
    struct ggml_tensor * inp_K_shift;   // I32 [n_ctx]
#ifdef GGML_USE_MPI
    // 如果定义了 GGML_USE_MPI，则创建 MPI 上下文对象
    ggml_mpi_context * ctx_mpi = NULL;
#endif
};

//
// kv cache helpers
//

// 初始化键值缓存
static bool llama_kv_cache_init(
             struct llama_kv_cache & cache, // 键值缓存对象
                 const llama_model & model, // 模型对象
                         ggml_type   ktype, // 键的数据类型
                         ggml_type   vtype, // 值的数据类型
                          uint32_t   n_ctx, // 上下文数量
                              bool   offload) { // 是否进行数据迁移
    const struct llama_hparams & hparams = model.hparams;

    const uint32_t n_embd_k_gqa = hparams.n_embd_k_gqa();
    const uint32_t n_embd_v_gqa = hparams.n_embd_v_gqa();
    const int64_t  n_layer      = hparams.n_layer;

    // 初始化缓存属性
    cache.has_shift = false;
    cache.head = 0;
    cache.size = n_ctx;
    cache.used = 0;

    // 清空缓存单元格并设置大小
    cache.cells.clear();
    cache.cells.resize(n_ctx);

#ifdef GGML_USE_CLBLAST
    // 如果定义了 GGML_USE_CLBLAST，则禁用数据迁移
    offload = false;
#endif

    // 统计使用的缓冲区类型
    std::map<ggml_backend_buffer_type_t, int> buft_layer_count;
    if (offload) {
        for (int64_t i = 0; i < n_layer; ++i) {
            buft_layer_count[model.buft_layer[i].buft]++;
        }
    } else {
        buft_layer_count[llama_default_buffer_type_cpu(true)] = n_layer;
    }

    // 为每种缓冲区类型创建上下文
    std::map<ggml_backend_buffer_type_t, ggml_context *> ctx_map;
    for (auto & it : buft_layer_count) {
        int n_layers = it.second;
        struct ggml_init_params params = {
            /*.mem_size   =*/ 2u*n_layers*ggml_tensor_overhead(),
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
        };
        ggml_context * ctx = ggml_init(params);
        if (!ctx) {
            LLAMA_LOG_ERROR("%s: failed to allocate context for kv cache\n", __func__);
            return false;
        }
        ctx_map[it.first] = ctx;
        cache.ctxs.push_back(ctx);
    }

    // 预留空间以存储键和值
    cache.k_l.reserve(n_layer);
    cache.v_l.reserve(n_layer);
    // 遍历每个层次，为每个层次创建上下文和张量
    for (int i = 0; i < (int) n_layer; i++) {
        // 如果开启了 offload，则使用 ctx_map 中对应的上下文，否则使用缓存中的第一个上下文
        struct ggml_context * ctx = offload ? ctx_map.at(model.buft_layer[i].buft) : cache.ctxs.front();
        // 创建一个一维张量 k
        ggml_tensor * k = ggml_new_tensor_1d(ctx, ktype, n_embd_k_gqa*n_ctx);
        // 创建一个一维张量 v
        ggml_tensor * v = ggml_new_tensor_1d(ctx, vtype, n_embd_v_gqa*n_ctx);
        // 为张量 k 和 v 设置名称
        ggml_format_name(k, "cache_k_l%d", i);
        ggml_format_name(v, "cache_v_l%d", i);
        // 将张量 k 和 v 添加到缓存的 k_l 和 v_l 中
        cache.k_l.push_back(k);
        cache.v_l.push_back(v);
    }

    // 为每个上下文分配张量并初始化缓冲区，以避免填充中出现 NaN
    for (auto it : ctx_map) {
        // 获取上下文对应的缓冲区类型和上下文
        ggml_backend_buffer_type_t buft = it.first;
        ggml_context * ctx = it.second;
        // 从缓冲区类型中为上下文分配张量
        ggml_backend_buffer_t buf = ggml_backend_alloc_ctx_tensors_from_buft(ctx, buft);
        // 如果分配失败，则记录错误信息并返回 false
        if (!buf) {
            LLAMA_LOG_ERROR("%s: failed to allocate buffer for kv cache\n", __func__);
            return false;
        }
        // 清空缓冲区
        ggml_backend_buffer_clear(buf, 0);
        // 记录缓冲区的大小信息
        LLAMA_LOG_INFO("%s: %10s KV buffer size = %8.2f MiB\n", __func__, ggml_backend_buffer_name(buf), ggml_backend_buffer_get_size(buf)/1024.0/1024.0);
        // 将缓冲区添加到缓存的 bufs 中
        cache.bufs.push_back(buf);
    }

    // 返回 true 表示成功
    return true;
// 寻找缓存中大小为 "n_tokens" 的空槽位
// 更新缓存头指针
// 注意：成功时，重要的是缓存头指针指向槽位的第一个单元格
static bool llama_kv_cache_find_slot(
           struct llama_kv_cache & cache,
        const struct llama_batch & batch) {
    // 缓存的上下文大小
    const uint32_t n_ctx    = cache.size;
    // 批处理的令牌数量
    const uint32_t n_tokens = batch.n_tokens;

    if (n_tokens > n_ctx) {
        // 如果令牌数量大于上下文大小，记录错误并返回 false
        LLAMA_LOG_ERROR("%s: n_tokens=%d > n_ctx=%d\n", __func__, n_tokens, n_ctx);
        return false;
    }

    uint32_t n_tested = 0;

    while (true) {
        if (cache.head + n_tokens > n_ctx) {
            n_tested += n_ctx - cache.head;
            cache.head = 0;
            continue;
        }

        bool found = true;
        for (uint32_t i = 0; i < n_tokens; i++) {
            if (cache.cells[cache.head + i].pos >= 0) {
                found = false;
                cache.head += i + 1;
                n_tested   += i + 1;
                break;
            }
        }

        if (found) {
            break;
        }

        if (n_tested >= n_ctx) {
            //LLAMA_LOG_ERROR("%s: failed to find a slot for %d tokens\n", __func__, n_tokens);
            return false;
        }
    }

    for (uint32_t i = 0; i < n_tokens; i++) {
        cache.cells[cache.head + i].pos = batch.pos[i];

        for (int32_t j = 0; j < batch.n_seq_id[i]; j++) {
            cache.cells[cache.head + i].seq_id.insert(batch.seq_id[i][j]);
        }
    }

    cache.used += n_tokens;

    return true;
}

// 查找当前正在使用的单元格数量
static int32_t llama_kv_cache_cell_max(const struct llama_kv_cache & cache) {
    for (uint32_t i = cache.size - 1; i > 0; --i) {
        if (cache.cells[i].pos >= 0 && !cache.cells[i].seq_id.empty()) {
            return i + 1;
        }
    }

    return 0;
}

static void llama_kv_cache_clear(struct llama_kv_cache & cache) {
    // 遍历缓存中的所有元素
    for (int32_t i = 0; i < (int32_t) cache.size; ++i) {
        // 将当前元素的位置设为-1
        cache.cells[i].pos = -1;
        // 清空当前元素的序列ID
        cache.cells[i].seq_id.clear();
    }
    // 将缓存的头部位置设为0
    cache.head = 0;
    // 将缓存已使用的大小设为0
    cache.used = 0;
}

// 从缓存中删除指定序列 ID 的数据
static void llama_kv_cache_seq_rm(
        struct llama_kv_cache & cache,
                 llama_seq_id   seq_id,
                    llama_pos   p0,
                    llama_pos   p1) {
    // 初始化新的头指针为缓存大小
    uint32_t new_head = cache.size;

    // 如果 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    // 如果 p1 小于 0，则将其设置为 llama_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 遍历缓存中的每个单元
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果单元的位置在 p0 和 p1 之间，并且序列 ID 小于 0
        if (cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            if (seq_id < 0) {
                // 清空单元的序列 ID
                cache.cells[i].seq_id.clear();
            } else if (cache.cells[i].has_seq_id(seq_id)) {
                // 如果单元包含指定的序列 ID，则删除该序列 ID
                cache.cells[i].seq_id.erase(seq_id);
            } else {
                // 否则继续下一个单元
                continue;
            }
            // 如果单元的序列 ID 为空
            if (cache.cells[i].seq_id.empty()) {
                // 如果单元的位置大于等于 0，则减少已使用单元的计数
                if (cache.cells[i].pos >= 0) cache.used--;

                // 将单元的位置设置为 -1
                cache.cells[i].pos = -1;
                // 如果新的头指针还未更新，则更新为当前单元的索引
                if (new_head == cache.size) new_head = i;
            }
        }
    }

    // 如果释放了一个单元，则将头指针设置为该单元的索引，以便搜索可以从该位置开始
    if (new_head != cache.size && new_head < cache.head) cache.head = new_head;
}

// 将指定序列 ID 的数据复制到另一个序列 ID
static void llama_kv_cache_seq_cp(
        struct llama_kv_cache & cache,
                 llama_seq_id   seq_id_src,
                 llama_seq_id   seq_id_dst,
                    llama_pos   p0,
                    llama_pos   p1) {
    // 如果 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    // 如果 p1 小于 0，则将其设置为 llama_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 将头指针设置为 0
    cache.head = 0;

    // 遍历缓存中的每个单元
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果单元包含源序列 ID，并且位置在 p0 和 p1 之间
        if (cache.cells[i].has_seq_id(seq_id_src) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 将目标序列 ID 插入到单元的序列 ID 中
            cache.cells[i].seq_id.insert(seq_id_dst);
        }
    }
}

// 保留指定序列 ID 的数据，其他数据将被删除
static void llama_kv_cache_seq_keep(struct llama_kv_cache & cache, llama_seq_id seq_id) {
    // 初始化新的头指针为缓存大小
    uint32_t new_head = cache.size;
    // 遍历缓存中的所有单元格
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果单元格中不包含指定的序列 ID
        if (!cache.cells[i].has_seq_id(seq_id)) {
            // 如果单元格中的位置大于等于 0，则缓存使用量减一
            if (cache.cells[i].pos >= 0) cache.used--;
            // 将单元格中的位置设为 -1
            cache.cells[i].pos = -1;
            // 清空单元格中的序列 ID
            cache.cells[i].seq_id.clear();
            // 如果 new_head 等于 cache.size，则将 new_head 设为当前遍历的单元格索引
            if (new_head == cache.size) new_head = i;
        } else {
            // 清空单元格中的序列 ID
            cache.cells[i].seq_id.clear();
            // 将指定的序列 ID 插入到单元格中
            cache.cells[i].seq_id.insert(seq_id);
        }
    }

    // 如果释放了一个槽位，则将头指针设置为该槽位，以便从该位置开始搜索
    if (new_head != cache.size && new_head < cache.head) cache.head = new_head;
// 向前移动缓存中的键值对，根据给定的序列 ID、位置范围和增量
static void llama_kv_cache_seq_shift(
        struct llama_kv_cache & cache,
                 llama_seq_id   seq_id,
                    llama_pos   p0,
                    llama_pos   p1,
                    llama_pos   delta) {
    // 初始化新的头部索引为缓存大小
    uint32_t new_head = cache.size;

    // 如果位置 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    // 如果位置 p1 小于 0，则将其设置为 llama_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 遍历缓存中的键值对
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果当前键值对具有指定的序列 ID，并且位置在指定范围内
        if (cache.cells[i].has_seq_id(seq_id) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 标记缓存已经发生了移动
            cache.has_shift = true;
            // 更新键值对的位置和增量
            cache.cells[i].pos   += delta;
            cache.cells[i].delta += delta;

            // 如果更新后的位置小于 0
            if (cache.cells[i].pos < 0) {
                // 如果当前键值对的序列 ID 不为空，则减少已使用的键值对数量
                if (!cache.cells[i].seq_id.empty()) cache.used--;
                // 将当前键值对的位置设置为 -1，并清空序列 ID
                cache.cells[i].pos = -1;
                cache.cells[i].seq_id.clear();
                // 如果新的头部索引还未更新，则将其设置为当前键值对的索引
                if (new_head == cache.size) new_head = i;
            }
        }
    }

    // 如果释放了一个槽位，则将头部索引设置为该槽位，以便下次搜索从该位置开始
    // 否则从头部开始搜索
    cache.head = new_head != cache.size ? new_head : 0;
}

// 根据给定的序列 ID、位置范围和除数，将缓存中的键值对位置进行除法操作
static void llama_kv_cache_seq_div(
        struct llama_kv_cache & cache,
                 llama_seq_id   seq_id,
                    llama_pos   p0,
                    llama_pos   p1,
                          int   d) {
    // 如果位置 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    // 如果位置 p1 小于 0，则将其设置为 llama_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 遍历缓存中的键值对
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果当前键值对具有指定的序列 ID，并且位置在指定范围内
        if (cache.cells[i].has_seq_id(seq_id) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 标记缓存已经发生了移动
            cache.has_shift = true;

            // 将当前键值对的原位置保存为 p_old，然后进行除法操作并更新增量
            {
                llama_pos p_old = cache.cells[i].pos;
                cache.cells[i].pos   /= d;
                cache.cells[i].delta += cache.cells[i].pos - p_old;
            }
        }
    }
}

//
// 模型加载和保存
//

// 定义文件版本枚举类型
enum llama_fver {
    GGUF_FILE_VERSION_V1 = 1,
    GGUF_FILE_VERSION_V2 = 2,
    GGUF_FILE_VERSION_V3 = 3,
};
// 根据给定的 llama_fver 枚举值返回对应的版本名称字符串
static const char * llama_file_version_name(llama_fver version) {
    switch (version) {
        case GGUF_FILE_VERSION_V1: return "GGUF V1 (support until nov 2023)";
        case GGUF_FILE_VERSION_V2: return "GGUF V2";
        case GGUF_FILE_VERSION_V3: return "GGUF V3 (latest)";
    }

    return "unknown";
}

// 格式化输出给定的 tensor 形状，返回格式化后的字符串
static std::string llama_format_tensor_shape(const std::vector<int64_t> & ne) {
    char buf[256];
    // 将第一个元素格式化输出到缓冲区
    snprintf(buf, sizeof(buf), "%5" PRId64, ne.at(0));
    // 遍历剩余元素，依次格式化输出到缓冲区
    for (size_t i = 1; i < ne.size(); i++) {
        snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), ", %5" PRId64, ne.at(i));
    }
    return buf;
}

// 格式化输出给定的 tensor 结构体的形状，返回格式化后的字符串
static std::string llama_format_tensor_shape(const struct ggml_tensor * t) {
    char buf[256];
    // 将第一个元素格式化输出到缓冲区
    snprintf(buf, sizeof(buf), "%5" PRId64, t->ne[0]);
    // 遍历剩余元素，依次格式化输出到缓冲区
    for (int i = 1; i < GGML_MAX_DIMS; i++) {
        snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), ", %5" PRId64, t->ne[i]);
    }
    return buf;
}

// GGUFMeta 命名空间下的模板类 GKV_Base_Type，用于定义不同类型的键值对基础类型
namespace GGUFMeta {
    // 模板类 GKV_Base_Type 的具体实现
    template <typename T, gguf_type gt_, T (*gfun)(const gguf_context *, const int)>
    struct GKV_Base_Type {
        static constexpr gguf_type gt = gt_;

        // 获取键值对的值的静态方法
        static T getter(const gguf_context * ctx, const int kid) {
            return gfun(ctx, kid);
        }
    };

    // 模板类 GKV_Base 的具体实现，继承自 GKV_Base_Type，定义不同类型的键值对基础类型
    template<typename T> struct GKV_Base;

    // 定义不同类型的键值对基础类型，包括 bool、uint8_t、uint16_t、uint32_t、uint64_t、int8_t
    template<> struct GKV_Base<bool        >: GKV_Base_Type<bool,         GGUF_TYPE_BOOL,    gguf_get_val_bool> {};
    template<> struct GKV_Base<uint8_t     >: GKV_Base_Type<uint8_t,      GGUF_TYPE_UINT8,   gguf_get_val_u8  > {};
    template<> struct GKV_Base<uint16_t    >: GKV_Base_Type<uint16_t,     GGUF_TYPE_UINT16,  gguf_get_val_u16 > {};
    template<> struct GKV_Base<uint32_t    >: GKV_Base_Type<uint32_t,     GGUF_TYPE_UINT32,  gguf_get_val_u32 > {};
    template<> struct GKV_Base<uint64_t    >: GKV_Base_Type<uint64_t,     GGUF_TYPE_UINT64,  gguf_get_val_u64 > {};
    template<> struct GKV_Base<int8_t      >: GKV_Base_Type<int8_t,       GGUF_TYPE_INT8,    gguf_get_val_i8  > {};
}
    // 定义模板结构体 GKV_Base，用于处理不同类型的数据
    template<> struct GKV_Base<int16_t     >: GKV_Base_Type<int16_t,      GGUF_TYPE_INT16,   gguf_get_val_i16 > {};
    // 定义模板结构体 GKV_Base，用于处理 int32_t 类型的数据
    template<> struct GKV_Base<int32_t     >: GKV_Base_Type<int32_t,      GGUF_TYPE_INT32,   gguf_get_val_i32 > {};
    // 定义模板结构体 GKV_Base，用于处理 int64_t 类型的数据
    template<> struct GKV_Base<int64_t     >: GKV_Base_Type<int64_t,      GGUF_TYPE_INT64,   gguf_get_val_i64 > {};
    // 定义模板结构体 GKV_Base，用于处理 float 类型的数据
    template<> struct GKV_Base<float       >: GKV_Base_Type<float,        GGUF_TYPE_FLOAT32, gguf_get_val_f32 > {};
    // 定义模板结构体 GKV_Base，用于处理 double 类型的数据
    template<> struct GKV_Base<double      >: GKV_Base_Type<double,       GGUF_TYPE_FLOAT64, gguf_get_val_f64 > {};
    // 定义模板结构体 GKV_Base，用于处理 const char* 类型的数据
    template<> struct GKV_Base<const char *>: GKV_Base_Type<const char *, GGUF_TYPE_STRING,  gguf_get_val_str > {};

    // 定义模板结构体 GKV_Base，用于处理 std::string 类型的数据
    template<> struct GKV_Base<std::string> {
        // 设置 gguf_type 为 GGUF_TYPE_STRING
        static constexpr gguf_type gt = GGUF_TYPE_STRING;

        // 定义 getter 函数，用于获取字符串数据
        static std::string getter(const gguf_context * ctx, const int kid) {
            return gguf_get_val_str(ctx, kid);
        }
    };

    // 定义结构体 ArrayInfo，用于存储数组信息
    struct ArrayInfo{
        const gguf_type gt; // 存储数组类型
        const size_t length; // 存储数组长度
        const void * data; // 存储数组数据
    };

    // 定义模板结构体 GKV_Base，用于处理 ArrayInfo 类型的数据
    template<> struct GKV_Base<ArrayInfo> {
        public:
        // 设置 gguf_type 为 GGUF_TYPE_ARRAY
        static constexpr gguf_type gt = GGUF_TYPE_ARRAY;
        // 定义 getter 函数，用于获取数组信息
        static ArrayInfo getter(const gguf_context *ctx, const int k) {
            return ArrayInfo {
                gguf_get_arr_type(ctx, k), // 获取数组类型
                size_t(gguf_get_arr_n(ctx, k)), // 获取数组长度
                gguf_get_arr_data(ctx, k), // 获取数组数据
            };
        }
    };

    // 定义模板结构体 GKV_Base，用于处理任意类型的数据
    template<typename T>
    };
}

// 定义 llama_model_loader 结构体，用于加载模型
struct llama_model_loader {
    // 初始化键值对数量、张量数量、已创建数量
    int n_kv      = 0;
    int n_tensors = 0;
    int n_created = 0;

    // 初始化元素数量、字节数
    int64_t n_elements = 0;
    size_t  n_bytes    = 0;

    // 是否使用内存映射
    bool use_mmap = false;

    // llama_file、llama_ftype、llama_fver 对象
    llama_file  file;
    llama_ftype ftype;
    llama_fver  fver;

    // 唯一指针 mapping，键值对覆盖的无序映射
    std::unique_ptr<llama_mmap> mapping;
    std::unordered_map<std::string, struct llama_model_kv_override> kv_overrides;

    // gguf_context、ggml_context 对象
    struct gguf_context * ctx_gguf = NULL;
    struct ggml_context * ctx_meta = NULL;

    // 架构名称、LLM_KV 对象
    std::string arch_name;
    LLM_KV      llm_kv    = LLM_KV(LLM_ARCH_UNKNOWN);

    }

    // 析构函数
    ~llama_model_loader() {
        // 释放 gguf_context
        if (ctx_gguf) {
            gguf_free(ctx_gguf);
        }
        // 释放 ggml_context
        if (ctx_meta) {
            ggml_free(ctx_meta);
        }
    }

    // 模板函数，获取数组长度
    template<typename T>
    typename std::enable_if<std::is_integral<T>::value, bool>::type
    get_arr_n(const std::string & key, T & result, const bool required = true) {
        // 查找键在 gguf_context 中的索引
        const int kid = gguf_find_key(ctx_gguf, key.c_str());

        // 如果键不存在且为必需，则抛出异常
        if (kid < 0) {
            if (required) {
                throw std::runtime_error(format("key not found in model: %s", key.c_str()));
            }
            return false;
        }

        // 获取数组信息
        struct GGUFMeta::ArrayInfo arr_info =
            GGUFMeta::GKV<GGUFMeta::ArrayInfo>::get_kv(ctx_gguf, kid);

        // 将数组长度赋值给 result
        result = arr_info.length;
        return true;
    }

    // 模板函数，获取数组长度
    template<typename T>
    typename std::enable_if<std::is_integral<T>::value, bool>::type
    get_arr_n(const enum llm_kv kid, T & result, const bool required = true) {
        return get_arr_n(llm_kv(kid), result, required);
    }

    // 模板函数
    // 根据给定的键获取对应的值，并将结果存储在 result 中，如果 required 为 true，则必须找到对应的键，否则抛出异常
    bool get_key(const std::string & key, T & result, const bool required = true) {
        // 在 kv_overrides 中查找给定的键
        auto it = kv_overrides.find(key);

        // 如果找到对应的键，则获取其值，否则设置为 nullptr
        const struct llama_model_kv_override * override =
            it != kv_overrides.end() ? &it->second : nullptr;

        // 使用 GGUFMeta::GKV 类的 set 方法设置结果值，并返回是否找到对应的键
        const bool found = GGUFMeta::GKV<T>::set(ctx_gguf, key, result, override);

        // 如果 required 为 true 且未找到对应的键，则抛出异常
        if (required && !found) {
            throw std::runtime_error(format("key not found in model: %s", key.c_str()));
        }

        // 返回是否找到对应的键
        return found;
    }

    // 根据枚举类型 llm_kv 获取对应键的值
    template<typename T>
    bool get_key(const enum llm_kv kid, T & result, const bool required = true) {
        // 调用 get_key 方法，将枚举类型转换为字符串类型
        return get_key(llm_kv(kid), result, required);
    }

    // 获取架构名称
    std::string get_arch_name() const {
        return arch_name;
    }

    // 获取架构类型
    enum llm_arch get_arch() const {
        return llm_kv.arch;
    }

    // 根据索引获取张量名称
    const char * get_tensor_name(int i) const {
        return gguf_get_tensor_name(ctx_gguf, i);
    }

    // 根据张量名称获取张量元数据
    struct ggml_tensor * get_tensor_meta(const char * name) const {
        return ggml_get_tensor(ctx_meta, name);
    }

    // 根据索引获取张量元数据
    struct ggml_tensor * get_tensor_meta(int i) const {
        // 调用 get_tensor_name 方法获取张量名称，再根据名称获取张量元数据
        return get_tensor_meta(get_tensor_name(i));
    }

    // 为给定的张量元数据创建张量
    struct ggml_tensor * create_tensor_for(struct ggml_context * ctx, struct ggml_tensor * meta) {
        // 复制给定的张量元数据，设置名称并增加计数
        struct ggml_tensor * tensor = ggml_dup_tensor(ctx, meta);
        ggml_set_name(tensor, ggml_get_name(meta));

        n_created++;

        // 返回创建的张量
        return tensor;
    }
    // 创建一个张量，根据给定的名称、形状和是否必需参数
    struct ggml_tensor * create_tensor(struct ggml_context * ctx, const std::string & name, const std::vector<int64_t> & ne, bool required = true) {
        // 获取当前张量
        struct ggml_tensor * cur = ggml_get_tensor(ctx_meta, name.c_str());

        // 如果当前张量为空
        if (cur == NULL) {
            // 如果不是必需的，则返回空指针
            if (!required) {
                return NULL;
            }
            // 抛出运行时错误，指示张量未找到
            throw std::runtime_error(format("%s: tensor '%s' not found", __func__, name.c_str()));
        }

        {
            // 检查张量的形状是否正确
            bool is_ok = true;
            for (size_t i = 0; i < ne.size(); ++i) {
                if (ne[i] != cur->ne[i]) {
                    is_ok = false;
                    break;
                }
            }
            // 如果形状不正确，则抛出运行时错误
            if (!is_ok) {
                throw std::runtime_error(
                        format("%s: tensor '%s' has wrong shape; expected %s, got %s",
                            __func__, name.c_str(),
                            llama_format_tensor_shape(ne).c_str(),
                            llama_format_tensor_shape(cur).c_str()));
            }
        }

        // 为当前张量创建张量
        return create_tensor_for(ctx, cur);
    }

    // 检查是否已获取所有张量
    void done_getting_tensors() const {
        // 如果创建的张量数量与总张量数量不匹配，则抛出运行时错误
        if (n_created != n_tensors) {
            throw std::runtime_error(format("%s: wrong number of tensors; expected %d, got %d", __func__, n_tensors, n_created));
        }
    }

    // 获取文件中张量的偏移量
    size_t file_offset(const char * name) const {
        // 查找张量在文件中的索引
        const int idx = gguf_find_tensor(ctx_gguf, name);

        // 如果索引小于0，则抛出运行时错误，指示张量未在文件中找到
        if (idx < 0) {
            throw std::runtime_error(format("%s: tensor '%s' not found in the file", __func__, name));
        }

        // 返回数据偏移量和张量偏移量之和
        return gguf_get_data_offset(ctx_gguf) + gguf_get_tensor_offset(ctx_gguf, idx);
    }
    // 初始化映射，可以选择是否预取数据，lmlock 用于内存锁定
    void init_mapping(bool prefetch = true, llama_mlock * lmlock = nullptr) {
        // 如果使用 mmap，则预取整个文件 - 所有数据都是必需的
        if (use_mmap) {
            mapping.reset(new llama_mmap(&file, prefetch ? -1 : 0, ggml_is_numa()));
        }

        // 计算所有张量的总大小，用于进度报告
        for (int i = 0; i < gguf_get_n_tensors(ctx_gguf); i++) {
            struct ggml_tensor * cur = ggml_get_tensor(ctx_meta, gguf_get_tensor_name(ctx_gguf, i));
            size_data += ggml_nbytes(cur);
        }

        if (use_mmap && mapping) {
            if (lmlock) {
                lmlock->init(mapping->addr);
            }
            mmap_used_first = mapping->size;
        }
    }

    // 获取映射范围
    void get_mapping_range(size_t * first, size_t * last, ggml_context * ctx) const {
        GGML_ASSERT(mapping);

        *first = mapping->size;
        *last  = 0;
        for (ggml_tensor * tensor = ggml_get_first_tensor(ctx); tensor; tensor = ggml_get_next_tensor(ctx, tensor)) {
            const size_t offs = file_offset(ggml_get_name(tensor));
            *first = std::min(*first, offs);
            *last  = std::max(*last,  offs + ggml_nbytes(tensor));
        }
    }

    // 为了向后兼容，不支持 ggml-backend
    void load_data_for(struct ggml_tensor * cur) const {
        const size_t offs = file_offset(ggml_get_name(cur));

        if (use_mmap && mapping) {
            if (cur->data == nullptr) {
                cur->data = (uint8_t *)mapping->addr + offs;
            } else {
                memcpy(cur->data, (uint8_t *)mapping->addr + offs, ggml_nbytes(cur));
            }
        } else {
            GGML_ASSERT(cur->data != nullptr);
            file.seek(offs, SEEK_SET);
            file.read_raw(cur->data, ggml_nbytes(cur));
        }
    }

    // 已完成的大小
    size_t size_done = 0;
    // 数据总大小
    size_t size_data = 0;
    // 使用的 mmap 起始位置
    size_t mmap_used_first = -1;
    // 使用的 mmap 结束位置
    size_t mmap_used_last  = 0;
    // 如果被 progress_callback 取消，则返回 false
    }
};

//
// load LLaMA models
//

// 根据架构类型返回对应的架构名称
static std::string llama_model_arch_name(llm_arch arch) {
    // 在架构名称映射表中查找对应的架构名称
    auto it = LLM_ARCH_NAMES.find(arch);
    // 如果未找到对应的架构名称，则返回"unknown"
    if (it == LLM_ARCH_NAMES.end()) {
        return "unknown";
    }
    // 返回找到的架构名称
    return it->second;
}

// 根据数据类型返回对应的数据类型名称
static std::string llama_model_ftype_name(llama_ftype ftype) {
    // 如果数据类型包含LLAMA_FTYPE_GUESSED标志
    if (ftype & LLAMA_FTYPE_GUESSED) {
        // 返回去除LLAMA_FTYPE_GUESSED标志后的数据类型名称，并加上"(guessed)"后缀
        return llama_model_ftype_name((enum llama_ftype) (ftype & ~LLAMA_FTYPE_GUESSED)) + " (guessed)";
    }

    // 根据数据类型返回对应的数据类型名称
    switch (ftype) {
        case LLAMA_FTYPE_ALL_F32:     return "all F32";
        case LLAMA_FTYPE_MOSTLY_F16:  return "F16";
        case LLAMA_FTYPE_MOSTLY_Q4_0: return "Q4_0";
        case LLAMA_FTYPE_MOSTLY_Q4_1: return "Q4_1";
        case LLAMA_FTYPE_MOSTLY_Q4_1_SOME_F16:
                                      return "Q4_1, some F16";
        case LLAMA_FTYPE_MOSTLY_Q5_0: return "Q5_0";
        case LLAMA_FTYPE_MOSTLY_Q5_1: return "Q5_1";
        case LLAMA_FTYPE_MOSTLY_Q8_0: return "Q8_0";

        // K-quants
        case LLAMA_FTYPE_MOSTLY_Q2_K:   return "Q2_K - Medium";
        case LLAMA_FTYPE_MOSTLY_Q2_K_S: return "Q2_K - Small";
        case LLAMA_FTYPE_MOSTLY_Q3_K_S: return "Q3_K - Small";
        case LLAMA_FTYPE_MOSTLY_Q3_K_M: return "Q3_K - Medium";
        case LLAMA_FTYPE_MOSTLY_Q3_K_L: return "Q3_K - Large";
        case LLAMA_FTYPE_MOSTLY_Q4_K_S: return "Q4_K - Small";
        case LLAMA_FTYPE_MOSTLY_Q4_K_M: return "Q4_K - Medium";
        case LLAMA_FTYPE_MOSTLY_Q5_K_S: return "Q5_K - Small";
        case LLAMA_FTYPE_MOSTLY_Q5_K_M: return "Q5_K - Medium";
        case LLAMA_FTYPE_MOSTLY_Q6_K:   return "Q6_K";
        case LLAMA_FTYPE_MOSTLY_IQ2_XXS:return "IQ2_XSS - 2.0625 bpw";
        case LLAMA_FTYPE_MOSTLY_IQ2_XS: return "IQ2_XS - 2.3125 bpw";
        case LLAMA_FTYPE_MOSTLY_Q3_K_XS:return "Q3_K - Extra small";

        // 默认情况下返回"unknown, may not work"
        default: return "unknown, may not work";
    }
}

// 根据模型类型返回对应的模型类型名称
static const char * llama_model_type_name(e_model type) {
    # 根据不同的类型返回对应的型号字符串
    switch (type) {
        case MODEL_1B:     return "1B";
        case MODEL_3B:     return "3B";
        case MODEL_7B:     return "7B";
        case MODEL_8B:     return "8B";
        case MODEL_13B:    return "13B";
        case MODEL_14B:    return "14B";
        case MODEL_15B:    return "15B";
        case MODEL_30B:    return "30B";
        case MODEL_34B:    return "34B";
        case MODEL_40B:    return "40B";
        case MODEL_65B:    return "65B";
        case MODEL_70B:    return "70B";
        case MODEL_SMALL:  return "0.1B";
        case MODEL_MEDIUM: return "0.4B";
        case MODEL_LARGE:  return "0.8B";
        case MODEL_XL:     return "1.5B";
        # 默认情况下返回未知型号
        default:           return "?B";
    }
// 加载模型的架构信息
static void llm_load_arch(llama_model_loader & ml, llama_model & model) {
    // 获取模型的架构信息
    model.arch = ml.get_arch();
    // 如果架构信息未知，则抛出运行时错误
    if (model.arch == LLM_ARCH_UNKNOWN) {
        throw std::runtime_error("unknown model architecture: '" + ml.get_arch_name() + "'");
    }
}

// 加载模型的超参数信息
static void llm_load_hparams(
        llama_model_loader & ml,
        llama_model & model) {
    auto & hparams = model.hparams;
    const gguf_context * ctx = ml.ctx_gguf;

    // 将元数据转换为字符串
    for (int i = 0; i < gguf_get_n_kv(ctx); i++) {
        // 获取键值对的类型
        enum gguf_type type = gguf_get_kv_type(ctx, i);
        // 如果类型为数组，则跳过
        if (type == GGUF_TYPE_ARRAY) {
            continue;
        }
        // 获取键名和对应的字符串值
        const char * name = gguf_get_key(ctx, i);
        const std::string value = gguf_kv_to_str(ctx, i);
        // 将键值对添加到模型的键值对映射中
        model.gguf_kv.emplace(name, value);
    }

    // 获取通用键值对
    ml.get_key(LLM_KV_GENERAL_NAME, model.name, false);

    // 获取超参数键值对
    ml.get_arr_n(LLM_KV_TOKENIZER_LIST,       hparams.n_vocab);
    ml.get_key  (LLM_KV_CONTEXT_LENGTH,       hparams.n_ctx_train);
    ml.get_key  (LLM_KV_EMBEDDING_LENGTH,     hparams.n_embd);
    ml.get_key  (LLM_KV_FEED_FORWARD_LENGTH,  hparams.n_ff);
    ml.get_key  (LLM_KV_ATTENTION_HEAD_COUNT, hparams.n_head);
    ml.get_key  (LLM_KV_BLOCK_COUNT,          hparams.n_layer);
    ml.get_key  (LLM_KV_EXPERT_COUNT,         hparams.n_expert,      false);
    ml.get_key  (LLM_KV_EXPERT_USED_COUNT,    hparams.n_expert_used, false);

    // 断言超参数的专家数量和使用的专家数量不超过最大值
    GGML_ASSERT(hparams.n_expert <= LLAMA_MAX_EXPERTS);
    GGML_ASSERT(hparams.n_expert_used <= hparams.n_expert);
    if (hparams.n_expert > 0) {
        GGML_ASSERT(hparams.n_expert_used > 0);
    } else {
        GGML_ASSERT(hparams.n_expert_used == 0);
    }

    // n_head_kv 是可选的，如果不存在则默认为 n_head
    hparams.n_head_kv = hparams.n_head;
    ml.get_key(LLM_KV_ATTENTION_HEAD_COUNT_KV, hparams.n_head_kv, false);

    bool rope_finetuned = false;
    // 获取绳索微调标志，如果不存在则默认为 false
    ml.get_key(LLM_KV_ROPE_SCALING_FINETUNED, rope_finetuned, false);
}
    // 设置 rope_finetuned 参数为给定的 rope_finetuned 值
    hparams.rope_finetuned = rope_finetuned;

    // 将 n_yarn_orig_ctx 参数设置为 n_ctx_train 参数的值
    hparams.n_yarn_orig_ctx = hparams.n_ctx_train;
    // 从 ml 中获取 LLM_KV_ROPE_SCALING_ORIG_CTX_LEN 对应的值，并设置给 n_yarn_orig_ctx 参数
    ml.get_key(LLM_KV_ROPE_SCALING_ORIG_CTX_LEN, hparams.n_yarn_orig_ctx, false);

    // 设置 rope_freq_base_train 参数为 10000.0f
    hparams.rope_freq_base_train = 10000.0f;
    // 从 ml 中获取 LLM_KV_ROPE_FREQ_BASE 对应的值，并设置给 rope_freq_base_train 参数
    ml.get_key(LLM_KV_ROPE_FREQ_BASE, hparams.rope_freq_base_train, false);

    // 设置 rope_scaling_type_train 参数为从字符串转换而来的 rope_scaling 值
    std::string rope_scaling("linear");
    ml.get_key(LLM_KV_ROPE_SCALING_TYPE, rope_scaling, false);
    hparams.rope_scaling_type_train = llama_rope_scaling_type_from_string(rope_scaling);
    // 断言 rope_scaling_type_train 参数不为 LLAMA_ROPE_SCALING_UNSPECIFIED
    GGML_ASSERT(hparams.rope_scaling_type_train != LLAMA_ROPE_SCALING_UNSPECIFIED);

    // 设置 ropescale 参数为 0.0f
    float ropescale = 0.0f;
    // 如果从 ml 中获取 LLM_KV_ROPE_SCALING_FACTOR 对应的值失败，则尝试获取旧的键名 LLM_KV_ROPE_SCALE_LINEAR 对应的值
    if (!ml.get_key(LLM_KV_ROPE_SCALING_FACTOR, ropescale, false)) {
        ml.get_key(LLM_KV_ROPE_SCALE_LINEAR, ropescale, false);
    }
    // 设置 rope_freq_scale_train 参数为 ropescale 的倒数，如果 ropescale 为 0 则设置为 1
    hparams.rope_freq_scale_train = ropescale == 0.0f ? 1.0f : 1.0f/ropescale;

    // 对 n_rot 参数进行检查
    {
        // 计算 n_rot 参数的值
        hparams.n_rot = hparams.n_embd / hparams.n_head;
        // 从 ml 中获取 LLM_KV_ROPE_DIMENSION_COUNT 对应的值，并设置给 n_rot 参数
        ml.get_key(LLM_KV_ROPE_DIMENSION_COUNT, hparams.n_rot, false);

        // 如果模型架构为 LLM_ARCH_LLAMA 或 LLM_ARCH_FALCON，则进行进一步检查
        if (model.arch == LLM_ARCH_LLAMA || model.arch == LLM_ARCH_FALCON) {
            // 如果 n_rot 不等于 n_embd / n_head，则抛出异常
            if (hparams.n_rot != hparams.n_embd / hparams.n_head) {
                throw std::runtime_error(format("invalid n_rot: %u, expected %u", hparams.n_rot, hparams.n_embd / hparams.n_head));
            }
        }
        // 对于不同的模型，n_rot 的计算方式有所不同
        // gpt-neox: n_rot = rotary_pct * (n_embd / n_head)
        // gpt-j: n_rot = rotary_dim
    }

    // 设置 n_embd_head_k 参数为 n_embd / n_head
    hparams.n_embd_head_k = hparams.n_embd / hparams.n_head;
    // 从 ml 中获取 LLM_KV_ATTENTION_KEY_LENGTH 对应的值，并设置给 n_embd_head_k 参数
    ml.get_key(LLM_KV_ATTENTION_KEY_LENGTH, hparams.n_embd_head_k, false);

    // 设置 n_embd_head_v 参数为 n_embd / n_head
    hparams.n_embd_head_v = hparams.n_embd / hparams.n_head;
    // 从 ml 中获取 LLM_KV_ATTENTION_VALUE_LENGTH 对应的值，并设置给 n_embd_head_v 参数
    ml.get_key(LLM_KV_ATTENTION_VALUE_LENGTH, hparams.n_embd_head_v, false);

    // 设置模型的 ftype 参数为 ml 的 ftype 参数
    model.ftype = ml.ftype;
// TODO: 这个函数可能应该在 llama.h 文件中
// 声明一个静态函数 llama_tokenize_internal，接受 llama_vocab 对象、原始文本、是否为开头、是否为特殊字符作为参数，返回一个 llama_vocab::id 类型的向量
static std::vector<llama_vocab::id> llama_tokenize_internal(const llama_vocab & vocab, std::string raw_text, bool bos, bool special = false);
// 声明一个静态函数 llama_byte_to_token，接受 llama_vocab 对象和一个 uint8_t 类型的字符作为参数，返回一个 llama_token 类型的对象
static llama_token llama_byte_to_token(const llama_vocab & vocab, uint8_t ch);

// 定义一个函数 llm_load_vocab，接受一个 llama_model_loader 对象和一个 llama_model 对象作为参数
static void llm_load_vocab(
        llama_model_loader & ml,
        llama_model & model) {
    // 获取 model 对象中的 vocab 对象的引用
    auto & vocab = model.vocab;

    // 获取 ml 对象中的 gguf_context 对象的引用
    struct gguf_context * ctx = ml.ctx_gguf;

    // 获取 model.arch 对应的 kv 函数
    const auto kv = LLM_KV(model.arch);

    // 在 gguf_context 中查找 LLM_KV_TOKENIZER_LIST 对应的键的索引
    const int token_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_LIST).c_str());
    // 如果找不到，则抛出运行时错误
    if (token_idx == -1) {
        throw std::runtime_error("cannot find tokenizer vocab in model file\n");
    }

    // 初始化 scores 指针为空
    const float * scores = nullptr;
    // 在 gguf_context 中查找 LLM_KV_TOKENIZER_SCORES 对应的键的索引
    const int score_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_SCORES).c_str());
    // 如果找到，则将 scores 指向对应数据的内存地址
    if (score_idx != -1) {
        scores = (const float * ) gguf_get_arr_data(ctx, score_idx);
    }

    // 初始化 toktypes 指针为空
    const int * toktypes = nullptr;
    // 在 gguf_context 中查找 LLM_KV_TOKENIZER_TOKEN_TYPE 对应的键的索引
    const int toktype_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_TOKEN_TYPE).c_str());
    // 如果找到，则将 toktypes 指向对应数据的内存地址
    if (toktype_idx != -1) {
        toktypes = (const int * ) gguf_get_arr_data(ctx, toktype_idx);
    }

    // 确定词汇表类型
    {
        // 声明一个字符串变量用于存储分词器名称
        std::string tokenizer_name;

        // 从模型中获取分词器名称
        ml.get_key(LLM_KV_TOKENIZER_MODEL, tokenizer_name);

        // 判断分词器名称，如果是 "llama"，则设置词汇表类型为 LLAMA_VOCAB_TYPE_SPM
        if (tokenizer_name == "llama") {
            vocab.type = LLAMA_VOCAB_TYPE_SPM;

            // 设置默认特殊标记
            vocab.special_bos_id = 1;
            vocab.special_eos_id = 2;
            vocab.special_unk_id = 0;
            vocab.special_sep_id = -1;
            vocab.special_pad_id = -1;
        } else if (tokenizer_name == "gpt2") {
            // 如果分词器名称是 "gpt2"，则设置词汇表类型为 LLAMA_VOCAB_TYPE_BPE

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
        } else {
            // 如果分词器名称不是 "llama" 也不是 "gpt2"，输出警告信息并使用默认分词器 "llama"
            LLAMA_LOG_WARN("%s: unknown tokenizer: '%s'", __func__, tokenizer_name.c_str());
            LLAMA_LOG_WARN("%s: using default tokenizer: 'llama'", __func__);

            vocab.type = LLAMA_VOCAB_TYPE_SPM;
        }
    }
    // 获取词汇表的大小
    const uint32_t n_vocab = gguf_get_arr_n(ctx, token_idx);

    // 调整 id_to_token 的大小以容纳词汇表的大小
    vocab.id_to_token.resize(n_vocab);

    // 遍历词汇表中的每个单词
    for (uint32_t i = 0; i < n_vocab; i++) {
        // 获取单词字符串
        std::string word = gguf_get_arr_str(ctx, token_idx, i);
        // 确保单词字符串转换为 UTF-8 后长度大于 0
        GGML_ASSERT(codepoints_from_utf8(word).size() > 0);

        // 将单词与对应的索引存入 token_to_id 映射中
        vocab.token_to_id[word] = i;

        // 获取当前单词的 token_data 引用
        auto & token_data = vocab.id_to_token[i];
        // 设置 token_data 的文本为移动后的单词字符串
        token_data.text  = std::move(word);
        // 设置 token_data 的分数为 scores[i] 或默认值 0.0f
        token_data.score = scores ? scores[i] : 0.0f;
        // 设置 token_data 的类型为 toktypes[i] 或默认类型 LLAMA_TOKEN_TYPE_NORMAL
        token_data.type  = toktypes ? (llama_token_type) toktypes[i] : LLAMA_TOKEN_TYPE_NORMAL;
    }
    // 确保 id_to_token 和 token_to_id 的大小相等
    GGML_ASSERT(vocab.id_to_token.size() == vocab.token_to_id.size());

    // 确定换行符的 token ID
    if (vocab.type == LLAMA_VOCAB_TYPE_SPM) {
        // 如果词汇表类型为 SPM，则将换行符 '\n' 转换为 token ID
        vocab.linefeed_id = llama_byte_to_token(vocab, '\n');
    } else {
        // 如果词汇表类型不为 SPM，则通过 llama_tokenize_internal 函数获取换行符的 token ID
        const std::vector<int> ids = llama_tokenize_internal(vocab, "\u010A", false);
        // 确保 ids 不为空，即模型词汇表中包含换行符
        GGML_ASSERT(!ids.empty() && "model vocab missing newline token");
        // 将第一个 token ID 设置为换行符的 token ID
        vocab.linefeed_id = ids[0];
    }

    // 处理特殊 token
    {
        // 定义包含特殊标记类型和对应整数引用的向量
        const std::vector<std::pair<enum llm_kv, int32_t &>> special_token_types = {
            { LLM_KV_TOKENIZER_BOS_ID, vocab.special_bos_id },
            { LLM_KV_TOKENIZER_EOS_ID, vocab.special_eos_id },
            { LLM_KV_TOKENIZER_UNK_ID, vocab.special_unk_id },
            { LLM_KV_TOKENIZER_SEP_ID, vocab.special_sep_id },
            { LLM_KV_TOKENIZER_PAD_ID, vocab.special_pad_id },
        };
        // 遍历特殊标记类型和整数引用的向量
        for (const auto & it : special_token_types) {
            // 获取特殊标记类型对应的键
            const std::string & key = kv(std::get<0>(it));
            // 获取整数引用
            int32_t & id = std::get<1>(it);
    
            // 定义新的 ID 变量
            uint32_t new_id;
            // 如果无法获取特殊标记类型对应的新 ID，则继续下一次循环
            if (!ml.get_key(std::get<0>(it), new_id, false)) {
                continue;
            }
            // 如果新 ID 大于等于词汇表的大小，则记录警告信息
            if (new_id >= vocab.id_to_token.size()) {
                LLAMA_LOG_WARN("%s: bad special token: '%s' = %ud, using default id %d\n",
                    __func__, key.c_str(), new_id, id);
            } else {
                // 否则更新整数引用为新 ID
                id = new_id;
            }
        }
    
        // 处理是否添加 BOS 和 EOS 标记
        {
            // 临时变量用于存储布尔值
            bool temp = true;
    
            // 如果能够获取是否添加 BOS 标记的键，则更新词汇表的特殊添加 BOS 标记值
            if (ml.get_key(LLM_KV_TOKENIZER_ADD_BOS, temp, false)) {
                vocab.special_add_bos = int(temp);
            }
            // 如果能够获取是否添加 EOS 标记的键，则更新词汇表的特殊添加 EOS 标记值
            if (ml.get_key(LLM_KV_TOKENIZER_ADD_EOS, temp, false)) {
                vocab.special_add_eos = int(temp);
            }
        }
    }
    
    // 构建特殊标记缓存
// 加载模型元数据并打印信息
static void llm_load_print_meta(llama_model_loader & ml, llama_model & model) {
    // 获取模型超参数和词汇表
    const auto & hparams = model.hparams;
    const auto & vocab   = model.vocab;

    // 获取绳索缩放类型
    const auto rope_scaling_type = LLAMA_ROPE_SCALING_TYPES.at(hparams.rope_scaling_type_train);

    // 打印模型元数据信息
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
    LLAMA_LOG_INFO("%s: n_rot            = %u\n",     __func__, hparams.n_rot);
    LLAMA_LOG_INFO("%s: n_embd_head_k    = %u\n",     __func__, hparams.n_embd_head_k);
    LLAMA_LOG_INFO("%s: n_embd_head_v    = %u\n",     __func__, hparams.n_embd_head_v);
    LLAMA_LOG_INFO("%s: n_gqa            = %u\n",     __func__, hparams.n_gqa());
    LLAMA_LOG_INFO("%s: n_embd_k_gqa     = %u\n",     __func__, hparams.n_embd_k_gqa());
    LLAMA_LOG_INFO("%s: n_embd_v_gqa     = %u\n",     __func__, hparams.n_embd_v_gqa());
    LLAMA_LOG_INFO("%s: f_norm_eps       = %.1e\n",   __func__, hparams.f_norm_eps);
    LLAMA_LOG_INFO("%s: f_norm_rms_eps   = %.1e\n",   __func__, hparams.f_norm_rms_eps);
    LLAMA_LOG_INFO("%s: f_clamp_kqv      = %.1e\n",   __func__, hparams.f_clamp_kqv);
}
    // 输出函数最大偏差值
    LLAMA_LOG_INFO("%s: f_max_alibi_bias = %.1e\n",   __func__, hparams.f_max_alibi_bias);
    // 输出前馈神经网络的数量
    LLAMA_LOG_INFO("%s: n_ff             = %u\n",     __func__, hparams.n_ff);
    // 输出专家数量
    LLAMA_LOG_INFO("%s: n_expert         = %u\n",     __func__, hparams.n_expert);
    // 输出已使用的专家数量
    LLAMA_LOG_INFO("%s: n_expert_used    = %u\n",     __func__, hparams.n_expert_used);
    // 输出绳子缩放类型
    LLAMA_LOG_INFO("%s: rope scaling     = %s\n",     __func__, rope_scaling_type.c_str());
    // 输出绳子基础训练频率
    LLAMA_LOG_INFO("%s: freq_base_train  = %.1f\n",   __func__, hparams.rope_freq_base_train);
    // 输出绳子训练频率缩放
    LLAMA_LOG_INFO("%s: freq_scale_train = %g\n",     __func__, hparams.rope_freq_scale_train);
    // 输出原始上下文纱线数量
    LLAMA_LOG_INFO("%s: n_yarn_orig_ctx  = %u\n",     __func__, hparams.n_yarn_orig_ctx);
    // 输出是否微调绳子
    LLAMA_LOG_INFO("%s: rope_finetuned   = %s\n",     __func__, hparams.rope_finetuned ? "yes" : "unknown");
    // 输出模型类型
    LLAMA_LOG_INFO("%s: model type       = %s\n",     __func__, llama_model_type_name(model.type));
    // 输出模型特征类型
    LLAMA_LOG_INFO("%s: model ftype      = %s\n",     __func__, llama_model_ftype_name(model.ftype).c_str());
    // 根据模型元素数量输出模型参数大小
    if (ml.n_elements >= 1e12) {
        LLAMA_LOG_INFO("%s: model params     = %.2f T\n", __func__, ml.n_elements*1e-12);
    } else if (ml.n_elements >= 1e9) {
        LLAMA_LOG_INFO("%s: model params     = %.2f B\n", __func__, ml.n_elements*1e-9);
    } else if (ml.n_elements >= 1e6) {
        LLAMA_LOG_INFO("%s: model params     = %.2f M\n", __func__, ml.n_elements*1e-6);
    } else {
        LLAMA_LOG_INFO("%s: model params     = %.2f K\n", __func__, ml.n_elements*1e-3);
    }
    // 根据模型字节数输出模型大小
    if (ml.n_bytes < GiB) {
        LLAMA_LOG_INFO("%s: model size       = %.2f MiB (%.2f BPW) \n", __func__, ml.n_bytes/1024.0/1024.0,        ml.n_bytes*8.0/ml.n_elements);
    } else {
        LLAMA_LOG_INFO("%s: model size       = %.2f GiB (%.2f BPW) \n", __func__, ml.n_bytes/1024.0/1024.0/1024.0, ml.n_bytes*8.0/ml.n_elements);
    }

    // 输出通用键值对的名称
    LLAMA_LOG_INFO("%s: general.name     = %s\n",    __func__, model.name.c_str());

    // 输出特殊标记
    // 如果特殊的 BOS token 存在，则打印其信息
    if (vocab.special_bos_id != -1) { LLAMA_LOG_INFO( "%s: BOS token        = %d '%s'\n", __func__, vocab.special_bos_id, vocab.id_to_token[vocab.special_bos_id].text.c_str() ); }
    // 如果特殊的 EOS token 存在，则打印其信息
    if (vocab.special_eos_id != -1) { LLAMA_LOG_INFO( "%s: EOS token        = %d '%s'\n", __func__, vocab.special_eos_id, vocab.id_to_token[vocab.special_eos_id].text.c_str() ); }
    // 如果特殊的 UNK token 存在，则打印其信息
    if (vocab.special_unk_id != -1) { LLAMA_LOG_INFO( "%s: UNK token        = %d '%s'\n", __func__, vocab.special_unk_id, vocab.id_to_token[vocab.special_unk_id].text.c_str() ); }
    // 如果特殊的 SEP token 存在，则打印其信息
    if (vocab.special_sep_id != -1) { LLAMA_LOG_INFO( "%s: SEP token        = %d '%s'\n", __func__, vocab.special_sep_id, vocab.id_to_token[vocab.special_sep_id].text.c_str() ); }
    // 如果特殊的 PAD token 存在，则打印其信息
    if (vocab.special_pad_id != -1) { LLAMA_LOG_INFO( "%s: PAD token        = %d '%s'\n", __func__, vocab.special_pad_id, vocab.id_to_token[vocab.special_pad_id].text.c_str() ); }
    // 如果特殊的 LF token 存在，则打印其信息
    if (vocab.linefeed_id    != -1) { LLAMA_LOG_INFO( "%s: LF token         = %d '%s'\n", __func__, vocab.linefeed_id,    vocab.id_to_token[vocab.linefeed_id].text.c_str() );    }
// 返回 false 如果被 progress_callback 取消
static bool llm_load_tensors(
        llama_model_loader & ml, // llama_model_loader 对象的引用
        llama_model & model, // llama_model 对象的引用
        int n_gpu_layers, // GPU 层数
        enum llama_split_mode split_mode, // 分割模式
        int main_gpu, // 主 GPU
        const float * tensor_split, // 张量分割
        bool use_mlock, // 是否使用 mlock
        llama_progress_callback progress_callback, // 进度回调函数
        void * progress_callback_user_data) { // 进度回调函数的用户数据

    model.t_start_us = ggml_time_us(); // 记录开始时间

    auto & hparams = model.hparams; // 获取模型的超参数

    model.split_mode   = split_mode; // 设置分割模式
    model.main_gpu     = main_gpu; // 设置主 GPU
    model.n_gpu_layers = n_gpu_layers; // 设置 GPU 层数

    const int64_t n_layer     = hparams.n_layer; // 获取层数
    const int64_t i_gpu_start = std::max((int64_t) hparams.n_layer - n_gpu_layers, (int64_t) 0); // 计算 GPU 起始层

    // 没有将输入层卸载到 GPU 上的好处，所以始终保留在 CPU 上
    model.buft_input = llama_default_buffer_type_cpu(true); // 设置输入层的缓冲类型为 CPU

    model.buft_layer.resize(n_layer); // 调整层的缓冲类型大小

    // 分配 CPU 层
    for (int64_t i = 0; i < i_gpu_start; ++i) {
        model.buft_layer[i] = llama_default_buffer_type_cpu(true); // 设置 CPU 层的缓冲类型为 CPU
    }

#ifdef GGML_USE_CUBLAS
    // 如果分割模式为 LLAMA_SPLIT_LAYER
    if (split_mode == LLAMA_SPLIT_LAYER) {
        // 计算分割点
        int device_count = ggml_backend_cuda_get_device_count();
        // 检查是否所有的分割点都为零或者为空
        bool all_zero = tensor_split == nullptr || std::all_of(tensor_split, tensor_split + device_count, [](float x) { return x == 0.0f; });
        float splits[GGML_CUDA_MAX_DEVICES];
        // 如果所有分割点为零，则使用默认的分割方式，根据空闲内存
        if (all_zero) {
            for (int i = 0; i < device_count; ++i) {
                size_t total;
                size_t free;
                ggml_backend_cuda_get_device_memory(i, &total, &free);
                splits[i] = free;
            }
        } else {
            // 否则使用给定的分割点
            std::copy(tensor_split, tensor_split + device_count, splits);
        }

        // 计算分割点的总和并归一化以获取实际的分割点
        float split_sum = 0.0f;
        for (int i = 0; i < device_count; ++i) {
            split_sum += splits[i];
            splits[i] = split_sum;
        }
        for (int i = 0; i < device_count; ++i) {
            splits[i] /= split_sum;
        }

        // 根据分割点将重复的层分配到设备上
        int act_gpu_layers = std::min(n_gpu_layers, (int)n_layer + 1);
        for (int64_t i = i_gpu_start; i < n_layer; ++i) {
            int layer_gpu = std::upper_bound(splits, splits + device_count, float(i - i_gpu_start)/act_gpu_layers) - splits;
            model.buft_layer[i] = llama_default_buffer_type_offload(layer_gpu);
        }
        // 分配输出层
        if (n_gpu_layers > n_layer) {
            int layer_gpu = std::upper_bound(splits, splits + device_count, float(act_gpu_layers - 1)/act_gpu_layers) - splits;
            model.buft_output = llama_default_buffer_type_offload(layer_gpu);
        } else {
            model.buft_output = llama_default_buffer_type_cpu(true);
        }
    } else
#endif
    {
        // 定义一个用于分割的缓冲区类型变量
        ggml_backend_buffer_type_t split_buft;
        // 如果分割模式为 LLAMA_SPLIT_ROW
        if (split_mode == LLAMA_SPLIT_ROW) {
            // 使用默认的分割缓冲区类型创建 split_buft
            split_buft = llama_default_buffer_type_split(main_gpu, tensor_split);
        } else {
            // 否则，对于不支持的后端，使用 LLAMA_SPLIT_NONE 或 LLAMA_SPLIT_LAYER
            split_buft = llama_default_buffer_type_offload(main_gpu);
        }
        // 为重复的层分配缓冲区类型
        for (int64_t i = i_gpu_start; i < n_layer; ++i) {
            model.buft_layer[i] = {
                split_buft,
                llama_default_buffer_type_offload(main_gpu)
            };
        }
        // 为输出层分配缓冲区类型
        if (n_gpu_layers > n_layer) {
            model.buft_output = {
                split_buft,
                llama_default_buffer_type_offload(main_gpu)
            };
        } else {
            // 如果 GPU 层数小于总层数，使用 CPU 缓冲区类型
            model.buft_output = llama_default_buffer_type_cpu(true);
        }
    }

    // 统计使用的缓冲区类型
    std::map<ggml_backend_buffer_type_t, int> buft_layer_count;
    buft_layer_count[model.buft_input.buft]++;
    buft_layer_count[model.buft_input.buft_matrix]++;
    buft_layer_count[model.buft_output.buft]++;
    buft_layer_count[model.buft_output.buft_matrix]++;
    for (int64_t i = 0; i < n_layer; ++i) {
        buft_layer_count[model.buft_layer[i].buft]++;
        buft_layer_count[model.buft_layer[i].buft_matrix]++;
    }

    // 为每种缓冲区类型创建一个上下文
    size_t ctx_size = ggml_tensor_overhead() * ml.n_tensors;
    std::map<ggml_backend_buffer_type_t, ggml_context *> ctx_map;
    for (auto & it : buft_layer_count) {
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
        };
        // 初始化上下文
        ggml_context * ctx = ggml_init(params);
        if (!ctx) {
            throw std::runtime_error(format("failed to create context"));
        }
        ctx_map[it.first] = ctx;
        model.ctxs.push_back(ctx);
    }
    // 记录日志信息，输出函数名和上下文大小（单位为 MiB）
    LLAMA_LOG_INFO("%s: ggml ctx size = %7.2f MiB\n", __func__, model.ctxs.size()*ctx_size/1024.0/1024.0);

    // 创建用于权重的张量
    }

    // 完成获取张量的操作
    ml.done_getting_tensors();

    // 初始化映射，如果使用内存锁定，则传入 model.mlock_mmap
    ml.init_mapping(true, use_mlock ? &model.mlock_mmap : nullptr);

    // 创建后端缓冲区
    std::vector<std::pair<ggml_context *, ggml_backend_buffer_t>> ctx_bufs;

    // 遍历上下文映射
    for (auto & it : ctx_map) {
        ggml_backend_buffer_type_t buft = it.first;
        ggml_context * ctx = it.second;
        ggml_backend_buffer_t buf = nullptr;

        // 只有包含模型张量的 mmap 区域被映射到后端缓冲区
        // 这对于 Apple Silicon 上的 Metal 很重要：如果整个模型可以映射到 Metal 缓冲区，那么我们可以只使用 Metal 处理所有层
        // 当模型大小超过 Metal 缓冲区大小但不超过 RAM 大小时，这允许使用部分卸载
        if (ml.use_mmap && buft == llama_default_buffer_type_cpu(true)) {
            // 获取映射范围
            size_t first, last;
            ml.get_mapping_range(&first, &last, ctx);
            // 从指针创建 CPU 缓冲区
            buf = ggml_backend_cpu_buffer_from_ptr((char *) ml.mapping->addr + first, last - first);
        }
#ifdef GGML_USE_METAL
        // 如果使用 Metal 并且开启内存映射
        else if (ml.use_mmap && buft == ggml_backend_metal_buffer_type()) {
            // 获取张量的最大尺寸
            const size_t max_size = ggml_get_max_tensor_size(ctx);
            // 获取映射范围的起始和结束位置
            size_t first, last;
            ml.get_mapping_range(&first, &last, ctx);
            // 从 Metal 缓冲区中创建指向指定位置的缓冲区
            buf = ggml_backend_metal_buffer_from_ptr((char *) ml.mapping->addr + first, last - first, max_size);
        }
#endif
        // 如果不满足上述条件
        else {
            // 从指定类型的缓冲区中为上下文分配张量
            buf = ggml_backend_alloc_ctx_tensors_from_buft(ctx, buft);
            // 如果成功分配缓冲区并且开启内存锁定并且缓冲区是主机缓冲区
            if (buf != nullptr && use_mlock && ggml_backend_buffer_is_host(buf)) {
                // 将缓冲区添加到模型的内存锁定缓冲区列表中
                model.mlock_bufs.emplace_back(new llama_mlock);
                auto & mlock_buf = model.mlock_bufs.back();
                // 初始化内存锁定缓冲区
                mlock_buf->init   (ggml_backend_buffer_get_base(buf));
                // 调整内存锁定缓冲区的大小
                mlock_buf->grow_to(ggml_backend_buffer_get_size(buf));
            }
        }
        // 如果缓冲区为空
        if (buf == nullptr) {
            // 抛出运行时错误
            throw std::runtime_error("failed to allocate buffer");
        }
        // 表示该缓冲区包含权重
        // 用于 ggml_backend_sched 改进操作调度 -> 使用权重的操作优先调度到包含权重的后端
        ggml_backend_buffer_set_usage(buf, GGML_BACKEND_BUFFER_USAGE_WEIGHTS);
        // 将缓冲区添加到模型的缓冲区列表中
        model.bufs.push_back(buf);
        // 将上下文和缓冲区添加到上下文缓冲区列表中
        ctx_bufs.emplace_back(ctx, buf);
    }

    // 打印内存需求
    {
        // 确定要将多少个重复层次的模型层次放到 GPU 上
        const int n_gpu = std::min(n_gpu_layers, int(hparams.n_layer));
    
        // 输出信息，显示将多少个重复层次的模型层次放到 GPU 上
        LLAMA_LOG_INFO("%s: offloading %d repeating layers to GPU\n", __func__, n_gpu);
        // 如果要放到 GPU 上的层数大于模型总层数，则输出信息
        if (n_gpu_layers > (int) hparams.n_layer) {
            LLAMA_LOG_INFO("%s: offloading non-repeating layers to GPU\n", __func__);
        }
    
        // 计算支持的最大后端层数和可卸载的最大层数
        const int max_backend_supported_layers = hparams.n_layer + 1;
        const int max_offloadable_layers       = hparams.n_layer + 1;
    
        // 输出信息，显示放到 GPU 上的层数和支持的最大后端层数
        LLAMA_LOG_INFO("%s: offloaded %d/%d layers to GPU\n", __func__, std::min(n_gpu_layers, max_offloadable_layers), max_backend_supported_layers);
    
        // 遍历模型缓冲区，输出每个缓冲区的大小
        for (ggml_backend_buffer_t buf : model.bufs) {
            LLAMA_LOG_INFO("%s: %10s buffer size = %8.2f MiB\n", __func__, ggml_backend_buffer_name(buf), ggml_backend_buffer_get_size(buf) / 1024.0 / 1024.0);
        }
    }
    
    // 填充 tensors_by_name
    for (ggml_context * ctx : model.ctxs) {
        for (auto * cur = ggml_get_first_tensor(ctx); cur != NULL; cur = ggml_get_next_tensor(ctx, cur)) {
            model.tensors_by_name.emplace_back(ggml_get_name(cur), cur);
        }
    }
    
    // 加载张量数据
    for (auto & it : ctx_bufs) {
        ggml_context * ctx = it.first;
        ggml_backend_buffer_t buf = it.second;
        // 如果加载数据失败，则返回 false
        if (!ml.load_all_data(ctx, progress_callback, progress_callback_user_data, buf, use_mlock ? &model.mlock_mmap : NULL)) {
            return false;
        }
    }
    
    // 将 mapping 移动到模型中
    model.mapping = std::move(ml.mapping);
    
    // 加载时间将在第一次评估后重新计算，因此考虑由 mmap() 推迟的页面错误
    model.t_load_us = ggml_time_us() - model.t_start_us;
    // 返回 true
    return true;
// 返回值说明：成功返回0，错误返回-1，通过llama_progress_callback取消返回-2
static int llama_model_load(const std::string & fname, llama_model & model, const llama_model_params & params) {
    try {
        // 使用给定的文件名、参数和键值对覆盖加载llama模型
        llama_model_loader ml(fname, params.use_mmap, params.kv_overrides);

        // 设置模型的超参数中的vocab_only属性
        model.hparams.vocab_only = params.vocab_only;

        // 加载模型的架构
        llm_load_arch   (ml, model);
        // 加载模型的超参数
        llm_load_hparams(ml, model);
        // 加载模型的词汇表
        llm_load_vocab  (ml, model);

        // 打印加载的元数据
        llm_load_print_meta(ml, model);

        // 如果词汇表大小与模型中的id_to_token映射大小不匹配，则抛出异常
        if (model.hparams.n_vocab != model.vocab.id_to_token.size()) {
            throw std::runtime_error("vocab size mismatch");
        }

        // 如果参数中设置了vocab_only，则跳过加载张量
        if (params.vocab_only) {
            LLAMA_LOG_INFO("%s: vocab only - skipping tensors\n", __func__);
            return 0;
        }

        // 加载张量数据
        if (!llm_load_tensors(
            ml, model, params.n_gpu_layers, params.split_mode,  params.main_gpu, params.tensor_split, params.use_mlock,
            params.progress_callback, params.progress_callback_user_data
        )) {
            return -2;
        }
    } catch (const std::exception & err) {
        // 捕获异常并记录错误信息
        LLAMA_LOG_ERROR("%s: error loading model: %s\n", __func__, err.what());
        return -1;
    }

    return 0;
}

// 构建输入嵌入张量
static struct ggml_tensor * llm_build_inp_embd(
        struct ggml_context * ctx,
        const llama_hparams & hparams,
        const llama_batch & batch,
        struct ggml_tensor * tok_embd,
        struct ggml_tensor * inp_tokens,
        struct ggml_tensor * inp_embd,
        const llm_build_cb & cb) {
    // 定义一个常量 n_embd，其值为 hparams 结构体中的 n_embd 成员变量
    const int64_t n_embd = hparams.n_embd;

    // 声明一个指向 ggml_tensor 结构体的指针 inpL
    struct ggml_tensor * inpL;

    // 如果 batch 结构体中的 token 成员为真
    if (batch.token) {
        // 通过 ggml_view_1d 函数创建一个一维视图 inp_tokens_v，指向 inp_tokens 数组，长度为 batch.n_tokens，起始索引为 0
        struct ggml_tensor * inp_tokens_v = ggml_view_1d(ctx, inp_tokens, batch.n_tokens, 0);
        // 调用回调函数 cb，传入参数 inp_tokens, "inp_tokens", -1
        cb(inp_tokens, "inp_tokens", -1);

        // 通过 ggml_get_rows 函数获取 tok_embd 中 inp_tokens_v 对应的行，赋值给 inpL
        inpL = ggml_get_rows(ctx, tok_embd, inp_tokens_v);
    } else {
#ifdef GGML_USE_MPI
        // 如果定义了 GGML_USE_MPI，则抛出错误信息"not implemented"
        GGML_ASSERT(false && "not implemented");
#endif

        // 从输入嵌入中创建一个 2D 视图，用于输入层
        inpL = ggml_view_2d(ctx, inp_embd, n_embd, batch.n_tokens, inp_embd->nb[1], 0);
    }

    // 返回输入层
    return inpL;
}

// Persimmon: n_rot = n_embd_head_k/2
// Other:     n_rot = n_embd_head_k
// 构建 K 移位
static void llm_build_k_shift(
      struct ggml_context * ctx,
      const llama_hparams & hparams,
      const llama_cparams & cparams,
     const llama_kv_cache & kv,
       struct ggml_cgraph * graph,
       struct ggml_tensor * K_shift,
            llm_rope_type   type,
                  int64_t   n_ctx,
                  float     freq_base,
                  float     freq_scale,
       const llm_build_cb & cb) {
    // 获取模型参数
    const int64_t n_layer       = hparams.n_layer;
    const int64_t n_head_kv     = hparams.n_head_kv;
    const int64_t n_embd_head_k = hparams.n_embd_head_k;
    const int64_t n_embd_k_gqa  = hparams.n_embd_k_gqa();
    const int32_t n_rot         = hparams.n_rot;
    const int32_t n_orig_ctx    = cparams.n_yarn_orig_ctx;
    const float   ext_factor    = cparams.yarn_ext_factor;
    const float   attn_factor   = cparams.yarn_attn_factor;
    const float   beta_fast     = cparams.yarn_beta_fast;
    const float   beta_slow     = cparams.yarn_beta_slow;

    // 初始化绳索类型为 0
    int rope_type = 0;

    // 根据类型设置绳索类型
    switch (type) {
        case LLM_ROPE:      rope_type = 0; break;
        case LLM_ROPE_NEOX: rope_type = 2; break;
        case LLM_ROPE_GLM:  rope_type = 4; break;
    }
    // 遍历每一层神经网络
    for (int il = 0; il < n_layer; ++il) {
        // 旋转前 n_rot 维度的数据
        struct ggml_tensor * tmp =
            ggml_rope_custom_inplace(ctx,
                    // 创建一个 3D 视图
                    ggml_view_3d(ctx, kv.k_l[il],
                        n_embd_head_k, n_head_kv, n_ctx,
                        // 计算每行的大小
                        ggml_row_size(kv.k_l[il]->type, n_embd_head_k),
                        ggml_row_size(kv.k_l[il]->type, n_embd_k_gqa),
                        0),
                    K_shift, n_rot, rope_type, 0, n_orig_ctx, freq_base, freq_scale,
                    ext_factor, attn_factor, beta_fast, beta_slow);
        // 回调函数，处理 K_shifted 数据
        cb(tmp, "K_shifted", il);
        // 构建前向扩展
        ggml_build_forward_expand(graph, tmp);
    }
// 构建键值存储，用于存储模型中的键值对数据
static void llm_build_kv_store(
        struct ggml_context * ctx, // 上下文对象
        const llama_hparams & hparams, // 模型超参数
        const llama_kv_cache & kv, // 键值缓存对象
        struct ggml_cgraph * graph, // 计算图对象
        struct ggml_tensor * k_cur, // 当前键张量
        struct ggml_tensor * v_cur, // 当前值张量
        int64_t   n_ctx, // 上下文大小
        int32_t   n_tokens, // 令牌数量
        int32_t   kv_head, // 键值头数量
        const llm_build_cb & cb, // 回调函数
        int64_t   il) { // il参数

    const int64_t n_embd_k_gqa = hparams.n_embd_k_gqa(); // 获取键的嵌入维度
    const int64_t n_embd_v_gqa = hparams.n_embd_v_gqa(); // 获取值的嵌入维度

    // 计算转置后的 [n_tokens, n_embd] V 矩阵
    struct ggml_tensor * v_cur_t = ggml_transpose(ctx, ggml_reshape_2d(ctx, v_cur, n_embd_v_gqa, n_tokens));
    //struct ggml_tensor * v_cur_t = ggml_transpose(ctx, v_cur); // TODO: reshape above is likely not needed
    cb(v_cur_t, "v_cur_t", il); // 调用回调函数

    // 创建键缓存视图
    struct ggml_tensor * k_cache_view = ggml_view_1d(ctx, kv.k_l[il], n_tokens*n_embd_k_gqa,
            (ggml_row_size(kv.k_l[il]->type, n_embd_k_gqa))*kv_head);
    cb(k_cache_view, "k_cache_view", il); // 调用回调函数

    // 创建值缓存视图
    struct ggml_tensor * v_cache_view = ggml_view_2d(ctx, kv.v_l[il], n_tokens, n_embd_v_gqa,
            (  n_ctx)*ggml_element_size(kv.v_l[il]),
            (kv_head)*ggml_element_size(kv.v_l[il]));
    cb(v_cache_view, "v_cache_view", il); // 调用回调函数

    // 重要：在键值缓存中存储 RoPE 版本的 K
    ggml_build_forward_expand(graph, ggml_cpy(ctx, k_cur,   k_cache_view));
    ggml_build_forward_expand(graph, ggml_cpy(ctx, v_cur_t, v_cache_view));
}

// 构建规范化层
static struct ggml_tensor * llm_build_norm(
        struct ggml_context * ctx, // 上下文对象
        struct ggml_tensor * cur, // 当前张量
        const llama_hparams & hparams, // 模型超参数
        struct ggml_tensor * mw, // 权重张量
        struct ggml_tensor * mb, // 偏置张量
        llm_norm_type   type, // 规范化类型
        const llm_build_cb & cb, // 回调函数
        int   il) { // il参数
    # 根据不同的类型进行不同的操作
    switch (type) {
        # 如果类型为 LLM_NORM，则调用 ggml_norm 函数进行归一化处理
        case LLM_NORM:     cur = ggml_norm    (ctx, cur, hparams.f_norm_eps);     break;
        # 如果类型为 LLM_NORM_RMS，则调用 ggml_rms_norm 函数进行 RMS 归一化处理
        case LLM_NORM_RMS: cur = ggml_rms_norm(ctx, cur, hparams.f_norm_rms_eps); break;
    }

    # 如果 mw 或者 mb 不为空，则调用 cb 函数进行回调
    if (mw || mb) {
        cb(cur, "norm", il);
    }

    # 如果 mw 不为空，则调用 ggml_mul 函数进行矩阵乘法操作
    if (mw) {
        cur = ggml_mul(ctx, cur, mw);
        # 如果 mb 不为空，则调用 cb 函数进行回调
        if (mb) {
            cb(cur, "norm_w", il);
        }
    }

    # 如果 mb 不为空，则调用 ggml_add 函数进行矩阵加法操作
    if (mb) {
        cur = ggml_add(ctx, cur, mb);
    }

    # 返回处理后的结果
    return cur;
# 构建前馈神经网络（Feedforward Neural Network）的层
static struct ggml_tensor * llm_build_ffn(
        struct ggml_context * ctx,  # 上下文对象
         struct ggml_tensor * cur,  # 当前层的张量
         struct ggml_tensor * up,   # 上一层的张量
         struct ggml_tensor * up_b,  # 上一层的偏置张量
         struct ggml_tensor * gate,  # 门控张量
         struct ggml_tensor * gate_b,  # 门控偏置张量
         struct ggml_tensor * down,  # 下一层的张量
         struct ggml_tensor * down_b,  # 下一层的偏置张量
         struct ggml_tensor * act_scales,  # 激活尺度张量
            llm_ffn_op_type   type_op,  # 前馈神经网络操作类型
          llm_ffn_gate_type   type_gate,  # 门控类型
         const llm_build_cb & cb,  # 构建回调函数
                        int   il) {  # 层索引
    # 计算上一层和当前层的乘积
    struct ggml_tensor * tmp = ggml_mul_mat(ctx, up, cur);
    # 调用回调函数处理乘积结果
    cb(tmp, "ffn_up", il);

    # 如果存在上一层的偏置张量
    if (up_b) {
        # 将上一层的偏置张量加到乘积结果中
        tmp = ggml_add(ctx, tmp, up_b);
        # 调用回调函数处理加和结果
        cb(tmp, "ffn_up_b", il);
    }

    # 如果存在门控张量
    if (gate) {
        # 根据门控类型进行不同的操作
        switch (type_gate) {
            case LLM_FFN_SEQ:
                {
                    # 计算门控张量和乘积结果的乘积
                    cur = ggml_mul_mat(ctx, gate, tmp);
                    # 调用回调函数处理门控结果
                    cb(cur, "ffn_gate", il);
                } break;
            case LLM_FFN_PAR:
                {
                    # 计算门控张量和当前层的乘积
                    cur = ggml_mul_mat(ctx, gate, cur);
                    # 调用回调函数处理门控结果
                    cb(cur, "ffn_gate", il);
                } break;
        }

        # 如果存在门控偏置张量
        if (gate_b) {
            # 将门控偏置张量加到门控结果中
            cur = ggml_add(ctx, cur, gate_b);
            # 调用回调函数处理门控偏置结果
            cb(cur, "ffn_gate_b", il);
        }
    } else {
        # 如果不存在门控张量，则当前层结果为乘积结果
        cur = tmp;
    }
}
    # 根据不同的操作类型进行不同的处理
    switch (type_op) {
        # 如果是 SILU 操作类型
        case LLM_FFN_SILU:
            {
                # 对当前数据进行 SILU 操作
                cur = ggml_silu(ctx, cur);
                # 回调函数，传入当前数据、操作名称和索引
                cb(cur, "ffn_silu", il);
            } break;
        # 如果是 GELU 操作类型
        case LLM_FFN_GELU:
            {
                # 对当前数据进行 GELU 操作
                cur = ggml_gelu(ctx, cur);
                # 回调函数，传入当前数据、操作名称和索引
                cb(cur, "ffn_gelu", il);
                # 如果激活尺度不为空
                if (act_scales != NULL) {
                    # 对当前数据除以激活尺度
                    cur = ggml_div(ctx, cur, act_scales);
                    # 回调函数，传入当前数据、操作名称和索引
                    cb(cur, "ffn_act", il);
                }
            } break;
        # 如果是 RELU 操作类型
        case LLM_FFN_RELU:
            {
                # 对当前数据进行 RELU 操作
                cur = ggml_relu(ctx, cur);
                # 回调函数，传入当前数据、操作名称和索引
                cb(cur, "ffn_relu", il);
            } break;
        # 如果是 RELU_SQR 操作类型
        case LLM_FFN_RELU_SQR:
            {
                # 对当前数据进行 RELU 操作
                cur = ggml_relu(ctx, cur);
                # 回调函数，传入当前数据、操作名称和索引
                cb(cur, "ffn_relu", il);

                # 对当前数据进行平方操作
                cur = ggml_sqr(ctx, cur);
                # 回调函数，传入当前数据、操作名称和索引
                cb(cur, "ffn_sqr(relu)", il);
            } break;
    }

    # 如果门类型是 PAR
    if (type_gate == LLM_FFN_PAR) {
        # 对当前数据和临时数据进行乘法操作
        cur = ggml_mul(ctx, cur, tmp);
        # 回调函数，传入当前数据、操作名称和索引
        cb(cur, "ffn_gate_par", il);
    }

    # 对当前数据和下行数据进行矩阵乘法操作
    cur = ggml_mul_mat(ctx, down, cur);
    # 如果下行数据存在
    if (down_b) {
        # 回调函数，传入当前数据、操作名称和索引
        cb(cur, "ffn_down", il);
    }

    # 如果下行数据存在
    if (down_b) {
        # 对当前数据和下行偏置数据进行加法操作
        cur = ggml_add(ctx, cur, down_b);
    }

    # 返回当前数据
    return cur;
// 如果 max_alibi_bias 大于 0，则应用 ALiBi
static struct ggml_tensor * llm_build_kqv(
        struct ggml_context * ctx,
        const llama_model & model,
        const llama_hparams & hparams,
        const llama_kv_cache & kv,
        struct ggml_cgraph * graph,
        struct ggml_tensor * wo,
        struct ggml_tensor * wo_b,
        struct ggml_tensor * q_cur,
        struct ggml_tensor * kq_mask,
        int64_t   n_ctx,
        int32_t   n_tokens,
        int32_t   n_kv,
        float     max_alibi_bias,
        float     kq_scale,
        const llm_build_cb & cb,
        int       il) {
    // 获取模型参数
    const int64_t n_head        = hparams.n_head;
    const int64_t n_head_kv     = hparams.n_head_kv;
    const int64_t n_embd_head_k = hparams.n_embd_head_k;
    const int64_t n_embd_k_gqa  = hparams.n_embd_k_gqa();
    const int64_t n_embd_head_v = hparams.n_embd_head_v;

    // 对当前查询张量 q 进行维度置换
    struct ggml_tensor * q = ggml_permute(ctx, q_cur, 0, 2, 1, 3);
    // 回调函数处理查询张量 q
    cb(q, "q", il);

    // 获取键张量 k
    struct ggml_tensor * k =
        ggml_view_3d(ctx, kv.k_l[il],
                n_embd_head_k, n_kv, n_head_kv,
                ggml_row_size(kv.k_l[il]->type, n_embd_k_gqa),
                ggml_row_size(kv.k_l[il]->type, n_embd_head_k),
                0);
    // 回调函数处理键张量 k
    cb(k, "k", il);

    // 计算键-值张量 kq
    struct ggml_tensor * kq = ggml_mul_mat(ctx, k, q);
    // 回调函数处理键-值张量 kq
    cb(kq, "kq", il);

    // 如果模型架构为 LLM_ARCH_PHI2，则需要使用 F32 精度执行 KQ 乘法，否则会得到 NaN
    if (model.arch == LLM_ARCH_PHI2) {
        // 设置键-值张量 kq 的精度为 F32
        ggml_mul_mat_set_prec(kq, GGML_PREC_F32);
    }
}
    // 如果最大的 alibi 偏差大于 0.0f
    if (max_alibi_bias > 0.0f) {
        // 临时分支，直到弄清楚如何通过 ggml_add 处理 ggml_alibi
        kq = ggml_scale(ctx, kq, kq_scale);
        cb(kq, "kq_scaled", il);

        // 如果最大的 alibi 偏差大于 0.0f
        if (max_alibi_bias > 0.0f) {
            // 待办事项：n_head 或 n_head_kv
            // 待办事项：K-shift 可能不起作用
            // 待办事项：改为 ggml_add
            kq = ggml_alibi(ctx, kq, /*n_past*/ 0, n_head, max_alibi_bias);
            cb(kq, "kq_scaled_alibi", il);
        }

        kq = ggml_add(ctx, kq, kq_mask);
        cb(kq, "kq_masked", il);

        kq = ggml_soft_max(ctx, kq);
        cb(kq, "kq_soft_max", il);
    } else {
        kq = ggml_soft_max_ext(ctx, kq, kq_mask, kq_scale);
        cb(kq, "kq_soft_max_ext", il);
    }

    // 将缓存的 v 拆分成 n_head 个头
    struct ggml_tensor * v =
        ggml_view_3d(ctx, kv.v_l[il],
                n_kv, n_embd_head_v, n_head_kv,
                ggml_element_size(kv.v_l[il])*n_ctx,
                ggml_element_size(kv.v_l[il])*n_ctx*n_embd_head_v,
                0);
    cb(v, "v", il);

    struct ggml_tensor * kqv = ggml_mul_mat(ctx, v, kq);
    cb(kqv, "kqv", il);

    struct ggml_tensor * kqv_merged = ggml_permute(ctx, kqv, 0, 2, 1, 3);
    cb(kqv_merged, "kqv_merged", il);

    struct ggml_tensor * cur = ggml_cont_2d(ctx, kqv_merged, n_embd_head_k*n_head, n_tokens);
    cb(cur, "kqv_merged_cont", il);

    ggml_build_forward_expand(graph, cur);

    cur = ggml_mul_mat(ctx, wo, cur);
    // 如果 wo_b 存在
    if (wo_b) {
        cb(cur, "kqv_wo", il);
    }

    // 如果 wo_b 存在
    if (wo_b) {
        cur = ggml_add(ctx, cur, wo_b);
    }

    // 返回当前结果
    return cur;
// 构建键值对操作，将键值对缓存中的数据存储到图中的张量中
static struct ggml_tensor * llm_build_kv(
        struct ggml_context * ctx,
        const llama_model & model,
        const llama_hparams & hparams,
        const llama_kv_cache & kv,
        struct ggml_cgraph * graph,
        struct ggml_tensor * wo,
        struct ggml_tensor * wo_b,
        struct ggml_tensor * k_cur,
        struct ggml_tensor * v_cur,
        struct ggml_tensor * q_cur,
        struct ggml_tensor * kq_mask,
        int64_t   n_ctx,
        int32_t   n_tokens,
        int32_t   kv_head,
        int32_t   n_kv,
        float     max_alibi_bias,
        float     kq_scale,
        const llm_build_cb & cb,
        int       il) {

    // 将 q_cur、k_cur、v_cur 三个节点一起添加到图中，以避免它们被重新排序
    // 通过这样做，减少图中的分割次数
    ggml_build_forward_expand(graph, q_cur);
    ggml_build_forward_expand(graph, k_cur);
    ggml_build_forward_expand(graph, v_cur);

    // 构建键值对存储操作，将键值对缓存中的数据存储到图中的张量中
    llm_build_kv_store(ctx, hparams, kv, graph, k_cur, v_cur, n_ctx, n_tokens, kv_head, cb, il);

    struct ggml_tensor * cur;
    // 构建 kqv 操作，生成当前张量 cur
    cur = llm_build_kqv(ctx, model, hparams, kv, graph,
            wo, wo_b,
            q_cur, kq_mask, n_ctx, n_tokens, n_kv, max_alibi_bias, kq_scale, cb, il);
    // 调用回调函数 cb，传递当前张量 cur 和标识 "kqv_out"
    cb(cur, "kqv_out", il);

    // 返回当前张量 cur
    return cur;
}

// 定义 llm_build_context 结构体，包含模型、上下文、超参数、控制参数、批次、自身键值对缓存等信息
struct llm_build_context {
    const llama_model    & model;
    const llama_context  & lctx;
    const llama_hparams  & hparams;
    const llama_cparams  & cparams;
    const llama_batch    & batch;
    const llama_kv_cache & kv_self;

    const int64_t n_embd;
    const int64_t n_layer;
    const int64_t n_ctx;       // 用户指定的上下文大小（可以与 n_ctx_train 不同）
    const int64_t n_head;
    const int64_t n_head_kv;
    const int64_t n_embd_head_k;
    const int64_t n_embd_k_gqa;
    const int64_t n_embd_head_v;
    const int64_t n_embd_v_gqa;
    const int64_t n_expert;
    // 声明一个常量整数变量 n_expert_used

    // 声明一系列常量浮点数变量，用于存储模型训练中的参数
    const float freq_base;
    const float freq_scale;
    const float ext_factor;
    const float attn_factor;
    const float beta_fast;
    const float beta_slow;
    const float norm_eps;
    const float norm_rms_eps;

    // 声明一系列常量整数变量，用于存储模型训练中的参数
    const int32_t n_tokens;
    const int32_t n_kv;     // 考虑的 KV 缓存大小（n_kv <= n_ctx）
    const int32_t kv_head;  // 在缓存中存储新的 KV 数据的索引
    const int32_t n_orig_ctx;

    // 声明一个常量布尔变量，用于标识是否进行绳索移位
    const bool do_rope_shift;

    // 声明一个常量引用类型变量 cb，用于存储 llm_build_cb 类型的回调函数

    // 声明一个引用类型变量 buf_compute_meta，用于存储 uint8_t 类型的向量

    // 声明一个指向 ggml_context 结构体的指针 ctx0，初始值为 nullptr

    // TODO: 考虑将整个接口设置为 noexcept
    // 构造函数 llm_build_context，接受 llama_context 和 llama_batch 作为参数
    // 构造函数，初始化模型参数和上下文参数
    const llm_build_cb & cb,
                  bool   worst_case) :
        model            (lctx.model),  // 初始化模型
        lctx             (lctx),         // 初始化上下文
        hparams          (model.hparams),  // 初始化模型超参数
        cparams          (lctx.cparams),   // 初始化上下文参数
        batch            (batch),          // 初始化批处理数据
        kv_self          (lctx.kv_self),   // 初始化自身键值对
        n_embd           (hparams.n_embd),  // 初始化嵌入维度
        n_layer          (hparams.n_layer),  // 初始化层数
        n_ctx            (cparams.n_ctx),    // 初始化上下文长度
        n_head           (hparams.n_head),   // 初始化头数
        n_head_kv        (hparams.n_head_kv),  // 初始化键值头数
        n_embd_head_k    (hparams.n_embd_head_k),  // 初始化键头嵌入维度
        n_embd_k_gqa     (hparams.n_embd_k_gqa()),  // 初始化 GQA 键嵌入维度
        n_embd_head_v    (hparams.n_embd_head_v),  // 初始化值头嵌入维度
        n_embd_v_gqa     (hparams.n_embd_v_gqa()),  // 初始化 GQA 值嵌入维度
        n_expert         (hparams.n_expert),  // 初始化专家数
        n_expert_used    (hparams.n_expert_used),  // 初始化使用的专家数
        freq_base        (cparams.rope_freq_base),  // 初始化频率基数
        freq_scale       (cparams.rope_freq_scale),  // 初始化频率缩放
        ext_factor       (cparams.yarn_ext_factor),  // 初始化扩展因子
        attn_factor      (cparams.yarn_attn_factor),  // 初始化注意力因子
        beta_fast        (cparams.yarn_beta_fast),  // 初始化快速学习率
        beta_slow        (cparams.yarn_beta_slow),  // 初始化慢速学习率
        norm_eps         (hparams.f_norm_eps),  // 初始化归一化 epsilon
        norm_rms_eps     (hparams.f_norm_rms_eps),  // 初始化 RMS 归一化 epsilon
        n_tokens         (batch.n_tokens),  // 初始化令牌数
        n_kv             (worst_case ? n_ctx            : kv_self.n),  // 初始化键值对数
        kv_head          (worst_case ? n_ctx - n_tokens : kv_self.head),  // 初始化键值头
        n_orig_ctx       (cparams.n_yarn_orig_ctx),  // 初始化原始上下文长度
        do_rope_shift    (worst_case || kv_self.has_shift),  // 初始化是否进行绳索移位
        cb               (cb),  // 初始化回调函数
        buf_compute_meta (lctx.buf_compute_meta) {  // 初始化计算元数据缓冲区
            // 所有初始化应在 init() 中完成
        }

    // 初始化函数，设置 GGML 初始化参数并调用 ggml_init 函数
    void init() {
        struct ggml_init_params params = {
            /*.mem_size   =*/ buf_compute_meta.size(),  // 设置内存大小
            /*.mem_buffer =*/ buf_compute_meta.data(),  // 设置内存缓冲区
            /*.no_alloc   =*/ true,  // 设置不分配内存
        };

        // 调用 ggml_init 函数进行初始化
        ctx0 = ggml_init(params);
    }
    // 定义一个名为 free 的方法，用于释放资源
    void free() {
        // 检查 ctx0 是否为空
        if (ctx0) {
            // 释放 ctx0 指向的内存
            ggml_free(ctx0);
            // 将 ctx0 指针设置为 nullptr
            ctx0 = nullptr;
        }
    }
// 结构体 ggml_cgraph 的指针 result 初始化为 NULL
static struct ggml_cgraph * llama_build_graph(
         llama_context & lctx,
     const llama_batch & batch) {
    // 获取 lctx 中的 model
    const auto & model = lctx.model;

    // 检查是否应该构建最坏情况图（用于内存测量）
    const bool worst_case = ggml_tallocr_is_measure(lctx.alloc);

    // 定义回调函数 cb，允许对每个张量应用自定义逻辑（例如 ggml-alloc、卸载等）
    llm_build_cb cb = [&](struct ggml_tensor * cur, const char * name, int il) {
        // 如果 il 大于等于 0
        if (il >= 0) {
            // 格式化张量名称
            ggml_format_name(cur, "%s-%d", name, il);
        } else {
            // 设置张量名称
            ggml_set_name(cur, name);
        }

        // 如果不使用离线 kqv
        if (!lctx.cparams.offload_kqv) {
            // 如果名称为 "kqv_merged_cont"
            if (strcmp(name, "kqv_merged_cont") == 0) {
                // 将 KV 存储和注意力输出之间的所有节点在 CPU 上运行
                ggml_backend_sched_set_node_backend(lctx.sched, cur, lctx.backend_cpu);
            }
        }
    };

    // 定义 llm_build_context 结构体 llm，传入 lctx、batch、cb 和 worst_case
    struct llm_build_context llm(lctx, batch, cb, worst_case);

    //
    // 设置输入数据
    //
    // 检查是否不是测量分配
    if (!ggml_tallocr_is_measure(lctx.alloc)) {
        // 如果批次中有 token
        if (batch.token) {
            // 获取 token 的数量
            const int64_t n_tokens = batch.n_tokens;
            // 设置输入 tokens 的数据
            ggml_backend_tensor_set(lctx.inp_tokens, batch.token, 0, n_tokens*ggml_element_size(lctx.inp_tokens));
        }

        // 如果批次中有 embd
        if (batch.embd) {
            // 获取 embd 的数量和 token 的数量
            const int64_t n_embd   = llm.n_embd;
            const int64_t n_tokens = batch.n_tokens;
            // 设置输入 embd 的数据
            ggml_backend_tensor_set(lctx.inp_embd, batch.embd, 0, n_tokens*n_embd*ggml_element_size(lctx.inp_embd));
        }

        // 如果批次中有 pos
        if (batch.pos) {
            // 获取 token 的数量
            const int64_t n_tokens = batch.n_tokens;
            // 设置输入 pos 的数据
            ggml_backend_tensor_set(lctx.inp_pos, batch.pos, 0, n_tokens*ggml_element_size(lctx.inp_pos));
        }

        {
            // 获取 kv 的数量和 token 的数量
            const int64_t n_kv     = llm.n_kv;
            const int64_t n_tokens = batch.n_tokens;
            // 断言输入 KQ_mask 的缓冲区在主机上
            GGML_ASSERT(ggml_backend_buffer_is_host(lctx.inp_KQ_mask->buffer));
            float * data = (float *) lctx.inp_KQ_mask->data;

            // 遍历计算数据
            for (int h = 0; h < 1; ++h) {
                for (int j = 0; j < n_tokens; ++j) {
                    const llama_pos    pos    = batch.pos[j];
                    const llama_seq_id seq_id = batch.seq_id[j][0];

                    for (int i = 0; i < n_kv; ++i) {
                        float f;
                        // 如果 kv_self 中的单元格没有 seq_id 或者 pos 大于当前 pos
                        if (!lctx.kv_self.cells[i].has_seq_id(seq_id) || lctx.kv_self.cells[i].pos > pos) {
                            f = -INFINITY;
                        } else {
                            f = 0;
                        }
                        data[h*(n_kv*n_tokens) + j*n_kv + i] = f;
                    }
                }
            }
        }

        // 如果需要进行绳索移位
        if (llm.do_rope_shift) {
            // 获取上下文的数量
            const int64_t n_ctx = llm.n_ctx;
            // 断言输入 K_shift 的缓冲区在主机上
            GGML_ASSERT(ggml_backend_buffer_is_host(lctx.inp_K_shift->buffer));
            int32_t * data = (int32_t *) lctx.inp_K_shift->data;

            // 遍历计算数据
            for (int i = 0; i < n_ctx; ++i) {
                data[i] = lctx.kv_self.cells[i].delta;
            }
        }
    }
    }

    // 初始化LLM
    llm.init();

    // 根据不同的架构选择不同的构建方法
    switch (model.arch) {
        case LLM_ARCH_LLAMA:
            {
                result = llm.build_llama();
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
        case LLM_ARCH_QWEN:
            {
                result = llm.build_qwen();
            } break;
        case LLM_ARCH_QWEN2:
            {
                result = llm.build_qwen2();
            } break;
        case LLM_ARCH_PHI2:
            {
                result = llm.build_phi2();
            } break;
        case LLM_ARCH_PLAMO:
            {
                result = llm.build_plamo();
            } break;
        case LLM_ARCH_GPT2:
            {
                result = llm.build_gpt2();
            } break;
        case LLM_ARCH_CODESHELL:
            {
                result = llm.build_codeshell();
            } break;
        case LLM_ARCH_ORION:
            {
                result = llm.build_orion();
            } break;
        default:
            // 如果架构不匹配，断言失败
            GGML_ASSERT(false);
    }

    // 释放LLM资源
    llm.free();

    // 返回结果
    return result;
// 解码一批令牌通过评估变压器
//
//   - lctx:      llama 上下文
//   - batch:     要评估的批次
//
// 在成功时返回 0
// 在警告时返回正整数
// 在错误时返回负整数
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
    // TODO: needs fix after #3228
    GGML_ASSERT(false && "not implemented");
    //ggml_mpi_eval_init(lctx.ctx_mpi, &n_tokens, &n_past, &n_threads);
#endif

    GGML_ASSERT(n_threads > 0);

    auto & kv_self = lctx.kv_self;

    const int64_t n_embd  = hparams.n_embd;
    const int64_t n_vocab = hparams.n_vocab;

    // 用于更平滑的批处理 API 过渡的辅助函数
    // 在弃用 llama_eval 调用之后，这些将被移除
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
}
    // 如果批处理的序列 ID 为空指针，则进行以下操作
    if (batch.seq_id == nullptr) {
        // 调整 n_seq_id 的大小为 n_tokens
        n_seq_id.resize(n_tokens);
        // 调整 seq_id 的大小为 n_tokens
        seq_id.resize(n_tokens);
        // 调整 seq_id_arr 的大小为 n_tokens
        seq_id_arr.resize(n_tokens);
        // 遍历 n_tokens，初始化 n_seq_id、seq_id 和 seq_id_arr
        for (uint32_t i = 0; i < n_tokens; i++) {
            n_seq_id[i] = 1;
            seq_id[i].resize(1);
            seq_id[i][0] = batch.all_seq_id;
            seq_id_arr[i] = seq_id[i].data();
        }

        // 将 n_seq_id 的数据指针赋给 batch.n_seq_id
        batch.n_seq_id = n_seq_id.data();
        // 将 seq_id_arr 的数据指针赋给 batch.seq_id
        batch.seq_id = seq_id_arr.data();
    }

    // 如果当前头部之前有足够的未使用单元格 ->
    //   最好从缓存的开头开始搜索，希望填充它
    if (kv_self.head > kv_self.used + 2*n_tokens) {
        // 将头部位置重置为 0
        kv_self.head = 0;
    }

    // 如果在缓存中找不到插槽，则返回 1
    if (!llama_kv_cache_find_slot(kv_self, batch)) {
        return 1;
    }

    // 一种启发式方法，避免在缓存尚未被利用时进行完整的搜索
    // 经过足够的迭代后，这种启发式方法的好处会消失
    // 如果开始对缓存进行碎片整理，则这种启发式方法的好处将更为重要
    // 计算 kv_self.n 的值，取 cparams.n_ctx 和 llama_kv_cache_cell_max(kv_self) 的最小值，且大于等于 32
    kv_self.n = std::min((int32_t) cparams.n_ctx, std::max(32, GGML_PAD(llama_kv_cache_cell_max(kv_self), 32)));
    //kv_self.n = llama_kv_cache_cell_max(kv_self);

    // 重置后端调度器
    ggml_backend_sched_reset(lctx.sched);
    // 设置评估回调函数和用户数据
    ggml_backend_sched_set_eval_callback(lctx.sched, lctx.cparams.cb_eval, lctx.cparams.cb_eval_user_data);

    // 构建图形
    ggml_cgraph * gf = llama_build_graph(lctx, batch);

    // 输出始终是图中的最后一个张量
    struct ggml_tensor * res = gf->nodes[gf->n_nodes - 1];
    GGML_ASSERT(strcmp(res->name, "result_output") == 0);

    // 嵌入可能是倒数第二个张量，也可能是倒数第三个张量
    struct ggml_tensor * embeddings = gf->nodes[gf->n_nodes - 2];
    if (strcmp(embeddings->name, "result_norm") != 0) {
        embeddings = gf->nodes[gf->n_nodes - 3];
        GGML_ASSERT(strcmp(embeddings->name, "result_norm") == 0);
    }
    // 输出日志信息，记录图构建时间，节点数量和叶子节点数量
    // 格式化输出构建时间，单位为毫秒
    // ggml_time_us() - t_start_us 表示构建时间的微秒数，除以1000转换为毫秒
    // gf->n_nodes 表示节点数量，gf->n_leafs 表示叶子节点数量
    // LLAMA_LOG_INFO 是一个日志输出函数
    // 该行代码用于记录图构建时间和节点信息
    // 通常用于性能分析和调试
    // 该行代码可能会输出到日志文件或控制台

    // 对于大型提示，如果启用了 BLAS，最好只使用一个线程
    // 否则，线程会在 BLAS 调用时自旋锁等待，降低性能
    // TODO: 这对于 Apple Silicon 非常重要，因为 CBLAS 仍然表现非常好
    //       我们仍然需要一些线程来处理所有非矩阵乘法操作，但不要太多以避免干扰 BLAS 调用。需要一个更好的解决方案
    // 如果 tokens 数量大于等于 32，且 CPU 支持 BLAS，且不支持 GPU BLAS
    // 则将线程数量限制在 4 和当前线程数量中的最小值
    // 用于优化性能，避免线程在 BLAS 调用时等待，降低性能

    // 检查是否完全离线加载模型
    // 如果 CPU 支持 CUBLAS 或 Vulkan，并且模型已完全离线加载
    // 则将线程数量设置为 1
    // 用于优化性能，避免干扰 BLAS 调用
#ifdef GGML_USE_MPI
    // 如果定义了 GGML_USE_MPI，则执行以下代码块
    const int64_t n_layer = hparams.n_layer;
    // 获取层数
    ggml_mpi_graph_compute_pre(lctx.ctx_mpi, gf, n_layer);
    // 使用 MPI 进行图计算的预处理
#endif

#ifdef GGML_USE_METAL
    // 如果定义了 GGML_USE_METAL，则执行以下代码块
    if (ggml_backend_is_metal(lctx.backend_metal)) {
        // 检查当前后端是否为 Metal
        ggml_backend_metal_set_n_cb(lctx.backend_metal, n_threads);
        // 设置 Metal 后端的回调函数数量
    }
#endif

    if (lctx.backend_cpu != nullptr) {
        // 如果 CPU 后端不为空
        ggml_backend_cpu_set_n_threads(lctx.backend_cpu, n_threads);
        // 设置 CPU 后端的线程数量
    }
    ggml_backend_sched_graph_compute(lctx.sched, gf);
    // 调度图计算任务

    // fprintf(stderr, "splits: %d\n", ggml_backend_sched_get_n_splits(lctx.sched));
    // 输出调度任务的分割数量到标准错误流

#ifdef GGML_USE_MPI
    // 如果定义了 GGML_USE_MPI，则执行以下代码块
    ggml_mpi_graph_compute_post(lctx.ctx_mpi, gf, n_layer);
    // 使用 MPI 进行图计算的后处理
#endif

    // update the kv ring buffer
    {
        // 更新键值环形缓冲区
        if (kv_self.has_shift) {
            // 如果键值缓冲区已经移位
            kv_self.has_shift = false;
            // 将移位标志设为假
            for (uint32_t i = 0; i < kv_self.size; ++i) {
                // 遍历缓冲区中的每个元素
                kv_self.cells[i].delta = 0;
                // 将每个元素的增量设为0
            }
        }

        kv_self.head += n_tokens;
        // 更新缓冲区头部位置

        // Ensure kv cache head points to a valid index.
        if (kv_self.head >= kv_self.size) {
            // 确保缓冲区头部指向有效索引
            kv_self.head = 0;
            // 如果超出缓冲区大小，则将头部位置重置为0
        }
    }

#ifdef GGML_PERF
    // 如果定义了 GGML_PERF，则执行以下代码块
    // 打印每个 GGML 操作的时间信息（用于调试目的）
    ggml_graph_print(gf);
#endif

    // plot the computation graph in dot format (for debugging purposes)
    //if (n_past%100 == 0) {
    //    ggml_graph_dump_dot(gf, NULL, "llama.dot");
    //}

    // extract logits
    // TODO: do not compute and extract logits if only embeddings are needed
    //       need to update the graphs to skip "result_output"
    {
        // 提取逻辑回归结果
        auto & logits_out = lctx.logits;

#ifndef NDEBUG
        // 如果未定义 NDEBUG，则执行以下代码块
        auto & logits_valid = lctx.logits_valid;
        // 获取有效的逻辑回归结果
        logits_valid.clear();
        // 清空有效的逻辑回归结果
        logits_valid.resize(n_tokens);
        // 调整有效的逻辑回归结果大小

        logits_out.clear();
        // 清空逻辑回归结果
#endif
// 结束预处理指令

// 获取结果节点的后端
ggml_backend_t res_backend = ggml_backend_sched_get_node_backend(lctx.sched, res);
GGML_ASSERT(res_backend != nullptr);
// 如果存在logits，则处理logits数据
if (batch.logits) {
    logits_out.resize(n_vocab * n_tokens);
    for (uint32_t i = 0; i < n_tokens; i++) {
        // 如果logits[i]为0，则跳过
        if (batch.logits[i] == 0) {
            continue;
        }
        // 异步获取结果节点的数据
        ggml_backend_tensor_get_async(res_backend, res, logits_out.data() + (n_vocab*i), (n_vocab*i)*sizeof(float), n_vocab*sizeof(float));
#ifndef NDEBUG
        logits_valid[i] = true;
#endif
    }
} else if (lctx.logits_all) {
    logits_out.resize(n_vocab * n_tokens);
    // 异步获取结果节点的数据
    ggml_backend_tensor_get_async(res_backend, res, logits_out.data(), 0, n_vocab*n_tokens*sizeof(float));
#ifndef NDEBUG
    std::fill(logits_valid.begin(), logits_valid.end(), true);
#endif
} else {
    logits_out.resize(n_vocab);
    // 异步获取结果节点的数据
    ggml_backend_tensor_get_async(res_backend, res, logits_out.data(), (n_vocab*(n_tokens - 1))*sizeof(float), n_vocab*sizeof(float));
#ifndef NDEBUG
    logits_valid[0] = true;
#endif
}
// 同步结果节点的后端
ggml_backend_synchronize(res_backend);

// 提取嵌入
if (!lctx.embedding.empty()) {
    auto & embedding_out = lctx.embedding;

    embedding_out.resize(n_embd);
    ggml_backend_t embeddings_backend = ggml_backend_sched_get_node_backend(lctx.sched, embeddings);
    // 异步获取嵌入节点的数据
    ggml_backend_tensor_get_async(embeddings_backend, embeddings, embedding_out.data(), (n_embd*(n_tokens - 1))*sizeof(float), n_embd*sizeof(float));
    // 同步嵌入节点的后端
    ggml_backend_synchronize(embeddings_backend);
}

// 仅针对单个标记评估性能
if (n_tokens == 1) {
    lctx.t_eval_us += ggml_time_us() - t_start_us;
    lctx.n_eval++;
}
else if (n_tokens > 1) {
    lctx.t_p_eval_us += ggml_time_us() - t_start_us;
    lctx.n_p_eval += n_tokens;
}
    // 如果尚未评估过一次，则获取更准确的加载时间
    // TODO: 修复这个问题
    if (!lctx.has_evaluated_once) {
        // 计算加载时间
        lctx.t_load_us = ggml_time_us() - lctx.t_start_us;
        // 标记已经评估过一次
        lctx.has_evaluated_once = true;
    }

    // 返回值为0
    return 0;
}

//
// tokenizer
//

// 获取词汇表中指定词汇的类型
static enum llama_vocab_type llama_vocab_get_type(const llama_vocab & vocab) {
    return vocab.type;
}

// 判断指定词汇是否为普通标记
static bool llama_is_normal_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_NORMAL;
}

// 判断指定词汇是否为未知标记
static bool llama_is_unknown_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_UNKNOWN;
}

// 判断指定词汇是否为控制标记
static bool llama_is_control_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_CONTROL;
}

// 判断指定词汇是否为字节标记
static bool llama_is_byte_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_BYTE;
}

// 判断指定词汇是否为用户定义标记
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

// 转义文本中的空白字符
static void llama_escape_whitespace(std::string & text) {
    # 调用 replace_all 函数，将文本中的空格替换为特定的 Unicode 字符"\xe2\x96\x81"
    replace_all(text, " ", "\xe2\x96\x81");
// 用于将字符串中的特殊字符替换为空格
static void llama_unescape_whitespace(std::string & word) {
    replace_all(word, "\xe2\x96\x81", " ");
}

// 定义了一个结构体 llm_symbol，包含索引、前一个索引、后一个索引、文本和大小
struct llm_symbol {
    using index = int;
    index prev;
    index next;
    const char * text;
    size_t n;
};

// 静态断言，检查 llm_symbol 是否是平凡可复制的
static_assert(std::is_trivially_copyable<llm_symbol>::value, "llm_symbol is not trivially copyable");

// SPM 分词器
// 原始实现：https://github.com/ggerganov/llama.cpp/commit/074bea2eb1f1349a0118239c4152914aecaa1be4
struct llm_bigram_spm {
    // 定义比较器，用于优先队列排序
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

// 定义了一个结构体 llm_tokenizer_spm，包含词汇表
struct llm_tokenizer_spm {
    llm_tokenizer_spm(const llama_vocab & vocab): vocab(vocab) {}

    // 重新分词函数，将符号重新分词为输出向量
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
    // 尝试添加一个双字节词
    void try_add_bigram(int left, int right) {
        // 如果左边或右边的索引为-1，则返回
        if (left == -1 || right == -1) {
            return;
        }

        // 将左右两个符号的文本合并成一个字符串
        const std::string text = std::string(symbols[left].text, symbols[left].n + symbols[right].n);
        // 查找该字符串在词汇表中的索引
        auto token = vocab.token_to_id.find(text);

        // 如果字符串不在词汇表中，则返回
        if (token == vocab.token_to_id.end()) {
            return;
        }

        // 如果字符串在词汇表中的索引超出范围，则返回
        if (static_cast<size_t>((*token).second) >= vocab.id_to_token.size()) {
            return;
        }

        // 获取词汇表中该字符串对应的数据
        const auto & tok_data = vocab.id_to_token[(*token).second];

        // 创建一个llm_bigram_spm对象，并设置其属性
        llm_bigram_spm bigram;
        bigram.left  = left;
        bigram.right = right;
        bigram.score = tok_data.score;
        bigram.size  = text.size();

        // 将bigram对象添加到工作队列中
        work_queue.push(bigram);

        // 将文本和左右索引的对应关系存储在rev_merge中
        rev_merge[text] = std::make_pair(left, right);
    }

    // 词汇表对象的引用
    const llama_vocab & vocab;

    // 符号向量
    std::vector<llm_symbol> symbols;
    // llm_bigram_spm对象队列
    llm_bigram_spm::queue work_queue;

    // 存储文本和左右索引的对应关系
    std::map<std::string, std::pair<int, int>> rev_merge;
};

// BPE tokenizer
// adapted from https://github.com/cmp-nct/ggllm.cpp [MIT License]
// tried to simplify unicode stuff, so most likely does not work 100% correctly!

// TODO: there are a lot of common parts between spm and bpe tokenizers, should be refactored and reused

// 定义结构体 llm_bigram_bpe，用于存储 bigram BPE 相关信息
struct llm_bigram_bpe {
    // 定义比较器，用于优先队列排序
    struct comparator {
        bool operator()(const llm_bigram_bpe & l, const llm_bigram_bpe & r) const {
            return l.rank > r.rank || (l.rank == r.rank && l.left > r.left);
        }
    };

    // 定义优先队列存储结构
    using queue_storage = std::vector<llm_bigram_bpe>;
    using queue = std::priority_queue<llm_bigram_bpe, queue_storage, comparator>;
    // 左侧符号索引
    llm_symbol::index left;
    // 右侧符号索引
    llm_symbol::index right;
    // 文本内容
    std::string text;
    // 排名
    int rank;
    // 大小
    size_t size;
};

// 定义结构体 llm_tokenizer_bpe，用于 BPE 分词
struct llm_tokenizer_bpe {
    // 构造函数，初始化词汇表
    llm_tokenizer_bpe(const llama_vocab & vocab): vocab(vocab) {}

    // 添加新的 bigram
private:
    void add_new_bigram(int left, int right) {
        if (left == -1 || right == -1) {
            return;
        }

        // 获取左侧和右侧符号的文本内容
        std::string left_token  = std::string(symbols[left].text,  symbols[left].n);
        std::string right_token = std::string(symbols[right].text, symbols[right].n);

        int rank_found = -1;

        // 查找 bigram 的排名
        rank_found = vocab.find_bpe_rank(left_token, right_token);

        if (rank_found < 0) {
            return;
        }

        // 创建新的 bigram 对象
        llm_bigram_bpe bigram;

        bigram.left  = left;
        bigram.right = right;
        bigram.text  = left_token + right_token;
        bigram.size  = left_token.size() + right_token.size();
        bigram.rank  = rank_found;

        // 将 bigram 加入工作队列
        work_queue.push(bigram);
    }

    // 词汇表
    const llama_vocab & vocab;

    // 符号列表
    std::vector<llm_symbol> symbols;
    std::vector<llm_symbol> symbols_final;

    // 工作队列
    llm_bigram_bpe::queue work_queue;
};

// 定义枚举类型 FRAGMENT_BUFFER_VARIANT_TYPE，表示片段缓冲区变体类型
typedef enum FRAGMENT_BUFFER_VARIANT_TYPE{
    FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN,
    FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT
} FRAGMENT_BUFFER_VARIANT_TYPE;

// 定义结构体 fragment_buffer_variant，表示片段缓冲区变体
struct fragment_buffer_variant{
    fragment_buffer_variant(llama_vocab::id _token)
    // 构造函数，初始化成员变量为 FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN 类型
    // _token 为传入的 token 值
    // _dummy 为默认值
    // offset 和 length 初始化为 0
    fragment_buffer_variant(const std::string & _raw_text, int64_t _offset, int64_t _length)
    :
        type(FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT), // 初始化类型为 FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT
        token((llama_vocab::id)-1), // 初始化 token 为 -1
        raw_text(_raw_text), // 初始化 raw_text 为传入的 _raw_text
        offset(_offset), // 初始化 offset 为传入的 _offset
        length(_length){ // 初始化 length 为传入的 _length
            // 断言 offset 大于等于 0
            GGML_ASSERT( _offset >= 0 );
            // 断言 length 大于等于 1
            GGML_ASSERT( _length >= 1 );
            // 断言 offset + length 小于等于 raw_text 的长度
            GGML_ASSERT( offset + length <= raw_text.length() );
        }

    // 常量成员变量，表示 fragment_buffer_variant 的类型
    const FRAGMENT_BUFFER_VARIANT_TYPE type;
    // 常量成员变量，表示 token 的值
    const llama_vocab::id token;
    // 常量成员变量，表示一个默认的字符串
    const std::string _dummy;
    // 常量引用成员变量，表示原始文本
    const std::string & raw_text;
    // 常量成员变量，表示偏移量
    const uint64_t offset;
    // 常量成员变量，表示长度
    const uint64_t length;
// 结束当前的代码块
};

// 定义预处理器调试宏
// #define PRETOKENIZERDEBUG

// 分词器状态分区函数，根据词汇表和文本片段缓冲区进行处理
static void tokenizer_st_partition(const llama_vocab & vocab, std::forward_list<fragment_buffer_variant> & buffer)
{
    // 遍历每个特殊标记
    for (const auto & st: vocab.special_tokens_cache) {
        const auto & special_token = st.first;
        const auto & special_id    = st.second;

        // 遍历每个文本片段
        std::forward_list<fragment_buffer_variant>::iterator it = buffer.begin();
        while (it != buffer.end()) {
            auto & fragment = (*it);

            // 如果片段是文本（尚未处理）
            if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT) {
                auto * raw_text = &(fragment.raw_text);

                auto raw_text_base_offset = fragment.offset;
                auto raw_text_base_length = fragment.length;

                // 循环遍历文本
                while (true) {
                    // 在该片段中查找给定特殊标记的第一个出现位置
                    // 仅传递偏移量参数限制了“搜索区域”，但匹配坐标仍相对于源完整的原始文本
                    auto match = raw_text->find(special_token, raw_text_base_offset);

                    // 未找到任何匹配项，停止处理该片段的给定特殊标记
                    if (match == std::string::npos) break;

                    // 检查匹配项是否在偏移量 <-> 长度的范围内
                    if (match + special_token.length() > raw_text_base_offset + raw_text_base_length) break;

#ifdef PRETOKENIZERDEBUG
                    // 如果定义了预处理器调试宏，则输出调试信息
                    LLAMA_LOG_WARN("FF: (%ld %ld %ld) '%s'\n", raw_text->length(), raw_text_base_offset, raw_text_base_length, raw_text->substr(raw_text_base_offset, raw_text_base_length).c_str());
#ifdef
                    // 计算匹配位置相对于缓冲区起始位置的偏移量
                    auto source = std::distance(buffer.begin(), it);

                    // 如果匹配位置比基本偏移量更靠后，则说明匹配位置左侧有一些文本
                    if (match > raw_text_base_offset) {
                        // 左侧文本
                        const int64_t left_reminder_offset = raw_text_base_offset + 0;
                        const int64_t left_reminder_length = match - raw_text_base_offset;
                        // 在匹配位置之前插入左侧文本
                        buffer.emplace_after(it, (*raw_text), left_reminder_offset, left_reminder_length);

#ifdef PRETOKENIZERDEBUG
                        // 输出左侧文本信息
                        LLAMA_LOG_WARN("FL: (%ld %ld) '%s'\n", left_reminder_offset, left_reminder_length, raw_text->substr(left_reminder_offset, left_reminder_length).c_str());
#endif
                        it++;
                    }

                    // 插入特殊标记
                    buffer.emplace_after(it, special_id);
                    it++;

                    // 如果匹配位置加上特殊标记长度小于原始文本长度，则说明匹配位置右侧有一些文本
                    if (match + special_token.length() < raw_text_base_offset + raw_text_base_length) {
                        const int64_t right_reminder_offset = match + special_token.length();
                        const int64_t right_reminder_length = raw_text_base_length - ((match - raw_text_base_offset) + special_token.length());
                        // 在匹配位置之后插入右侧文本
                        buffer.emplace_after(it, (*raw_text), right_reminder_offset, right_reminder_length);

#ifdef PRETOKENIZERDEBUG
                        // 输出右侧文本信息
                        LLAMA_LOG_WARN("FR: (%ld %ld) '%s'\n", right_reminder_offset, right_reminder_length, raw_text->substr(right_reminder_offset, right_reminder_length).c_str());
#else

                        // 移动迭代器到下一个位置
                        it++;

                        // 如果 source 为 0，则删除 buffer 中第一个元素
                        if (source == 0) {
                            buffer.erase_after(buffer.before_begin());
                        } else {
                            // 否则，删除 buffer 中 source-1 位置的元素
                            buffer.erase_after(std::next(buffer.begin(), (source-1)));
                        }

                        // 重复右侧的操作
                        raw_text_base_offset = right_reminder_offset;
                        raw_text_base_length = right_reminder_length;

#ifdef PRETOKENIZERDEBUG
                        // 输出调试信息
                        LLAMA_LOG_WARN("RR: (%ld %ld) '%s'\n", raw_text_base_offset, raw_text_base_length, raw_text->substr(raw_text_base_offset, raw_text_base_length).c_str());
#endif
                    } else {
                        if (source == 0) {
                            // 如果 source 为 0，则删除 buffer 中第一个元素
                            buffer.erase_after(buffer.before_begin());
                        } else {
                            // 否则，删除 buffer 中 source-1 位置的元素
                            buffer.erase_after(std::next(buffer.begin(), (source-1)));
                        }
                        // 跳出循环
                        break;
                    }
                }
            }
            // 移动迭代器到下一个位置
            it++;
        }
    }
}

// 内部 LLAMA 分词函数
static std::vector<llama_vocab::id> llama_tokenize_internal(const llama_vocab & vocab, std::string raw_text, bool bos, bool special) {
    // 初始化输出向量
    std::vector<llama_vocab::id> output;

    // OG 分词器行为：
    //
    // tokenizer.encode('', add_bos=True)  返回 [1]
    // tokenizer.encode('', add_bos=False) 返回 []

    // 如果需要在开头添加 bos，并且 vocab 中有特殊的 bos_id，则将其添加到输出向量中
    if (bos && vocab.special_bos_id != -1) {
        output.push_back(vocab.special_bos_id);
    }

    // 如果原始文本为空，则直接返回输出向量
    if (raw_text.empty()) {
        return output;
    }

    // 初始化片段缓冲区
    std::forward_list<fragment_buffer_variant> fragment_buffer;
    fragment_buffer.emplace_front( raw_text, 0, raw_text.length() );

    // 如果需要特殊处理，则调用 tokenizer_st_partition 函数
    if (special) tokenizer_st_partition( vocab, fragment_buffer );
    # 根据词汇表类型进行不同的处理
    switch (vocab.type) {
        # 如果词汇表类型是 SPM
        case LLAMA_VOCAB_TYPE_SPM:
            {
                # 遍历片段缓冲区中的每个片段
                for (const auto & fragment: fragment_buffer)
                {
                    # 如果片段类型是原始文本
                    if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT)
                    {
                        # 在不添加前导空格的情况下，我们无法获得与原始分词器相同的结果

                        # TODO: 可能可以通过修改 llm_tokenizer_x 使其像预分词器一样使用字符串偏移量
                        # 并将 'add space prefix' 作为布尔参数传递，从而完全消除此字符串复制
                        #
                        # 提取片段的原始文本，根据偏移量和长度进行截取
                        auto raw_text = fragment.raw_text.substr(fragment.offset, fragment.length);
                        # 如果当前片段是缓冲区中的第一个片段
                        if (&fragment == &fragment_buffer.front()) {
                            raw_text = " " + raw_text; // 如果第一个标记不是特殊标记，则在前面加上空格
                        }
#ifdef PRETOKENIZERDEBUG
                        // 如果定义了预处理令牌调试宏，则记录调试信息
                        LLAMA_LOG_WARN("TT: (%ld %ld %ld) '%s'\n", raw_text.length(), fragment.offset, fragment.length, raw_text.c_str());
#endif
                        // 使用给定词汇表创建 SPM 分词器
                        llm_tokenizer_spm tokenizer(vocab);
                        // 转义原始文本中的空白字符
                        llama_escape_whitespace(raw_text);
                        // 对原始文本进行分词处理，结果存储在输出向量中
                        tokenizer.tokenize(raw_text, output);
                    }
                    else // if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN)
                    {
                        // 如果片段类型为 TOKEN，则将其添加到输出向量中
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
                        // 提取原始文本片段
                        auto raw_text = fragment.raw_text.substr(fragment.offset, fragment.length);

#ifdef PRETOKENIZERDEBUG
                        // 如果定义了预处理令牌调试宏，则记录调试信息
                        LLAMA_LOG_WARN("TT: (%ld %ld %ld) '%s'\n", raw_text.length(), fragment.offset, fragment.length, raw_text.c_str());
#endif
                        // 使用给定词汇表创建 BPE 分词器
                        llm_tokenizer_bpe tokenizer(vocab);
                        // 对原始文本进行分词处理，结果存储在输出向量中
                        tokenizer.tokenize(raw_text, output);
                    }
                    else // if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN)
                    {
                        // 如果片段类型为 TOKEN，则将其添加到输出向量中
                        output.push_back(fragment.token);
                    }
                }
            } break;
    }

    // 返回处理后的输出向量
    return output;
}

//
// grammar - internal
//

// 部分生成的 UTF-8 序列的结构体
struct llama_partial_utf8 {
    uint32_t value;    // 到目前为止的位值（未移位）
    int      n_remain; // 剩余字节数；-1 表示无效序列
};

// 语法规则结构体
struct llama_grammar {
    const std::vector<std::vector<llama_grammar_element>>   rules; // 规则向量
    std::vector<std::vector<const llama_grammar_element *>> stacks; // 栈向量

    // 用于存储从接受的令牌部分生成的 UTF-8 序列的缓冲区
    # 定义一个名为llama_partial_utf8的变量，类型为partial_utf8
    llama_partial_utf8                                      partial_utf8;
// 结构体定义，用于存储候选的语法解析结果
struct llama_grammar_candidate {
    size_t               index; // 索引
    const uint32_t     * code_points; // 指向代码点的指针
    llama_partial_utf8   partial_utf8; // 部分 UTF-8 编码
};

// 解码可能以不完整序列结尾的 UTF-8 字符串。添加终止符0以用作指针。如果遇到无效序列，则返回`llama_partial_utf8.n_remain == -1`。
static std::pair<std::vector<uint32_t>, llama_partial_utf8> decode_utf8(
        const std::string & src, // 源字符串
        llama_partial_utf8   partial_start) { // 部分 UTF-8 编码的起始值
    static const int      lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 2, 2, 3, 4 }; // UTF-8 编码长度查找表
    const char          * pos      = src.c_str(); // 源字符串的指针
    std::vector<uint32_t> code_points; // 存储解码后的代码点
    // 英文字符串通常具有相同数量的代码点和字节。`+ 1` 用于终止符0。
    code_points.reserve(src.size() + 1); // 预留存储空间
    uint32_t              value    = partial_start.value; // 部分 UTF-8 编码的值
    int                   n_remain = partial_start.n_remain; // 剩余字节数

    // 继续之前的解码，如果适用
    while (*pos != 0 && n_remain > 0) {
        uint8_t next_byte = static_cast<uint8_t>(*pos); // 下一个字节
        if ((next_byte >> 6) != 2) {
            // 无效序列，中止
            code_points.push_back(0); // 添加0表示无效序列
            return std::make_pair(std::move(code_points), llama_partial_utf8{ 0, -1 }); // 返回无效序列
        }
        value = (value << 6) + (next_byte & 0x3F); // 计算代码点值
        ++pos;
        --n_remain;
    }

    if (partial_start.n_remain > 0 && n_remain == 0) {
        code_points.push_back(value); // 添加代码点值
    }

    // 解码任何后续的 UTF-8 序列，可能以不完整序列结尾
    // 当指针指向的值不为0时，进入循环
    while (*pos != 0) {
        // 将指针指向的值转换为无符号8位整数，作为第一个字节
        uint8_t  first_byte = static_cast<uint8_t>(*pos);
        // 获取第一个字节的高4位，作为索引查找剩余字节数
        uint8_t  highbits   = first_byte >> 4;
        // 计算剩余字节数
        n_remain   = lookup[highbits] - 1;

        // 如果剩余字节数小于0，表示序列无效，清空code_points并返回错误信息
        if (n_remain < 0) {
            code_points.clear();
            code_points.push_back(0);
            return std::make_pair(std::move(code_points), llama_partial_utf8{ 0, n_remain });
        }

        // 计算掩码，用于提取值
        uint8_t  mask       = (1 << (7 - n_remain)) - 1;
        // 提取值
        value      = first_byte & mask;
        ++pos;
        // 循环读取剩余字节，拼接成完整的值
        while (*pos != 0 && n_remain > 0) {
            value = (value << 6) + (static_cast<uint8_t>(*pos) & 0x3F);
            ++pos;
            --n_remain;
        }
        // 如果剩余字节数为0，表示值完整，加入code_points
        if (n_remain == 0) {
            code_points.push_back(value);
        }
    }
    // 在code_points末尾添加0，表示结束
    code_points.push_back(0);

    // 返回解析结果和剩余部分
    return std::make_pair(std::move(code_points), llama_partial_utf8{ value, n_remain });
// 返回 true 当 pos 指向规则定义的结尾时
static bool llama_grammar_is_end_of_sequence(const llama_grammar_element * pos) {
    switch (pos->type) {
        case LLAMA_GRETYPE_END: return true;  // NOLINT
        case LLAMA_GRETYPE_ALT: return true;  // NOLINT
        default:                return false;
    }
}

// 返回 true 当 chr 满足 pos 处的字符范围（正向或反向范围）
// 断言 pos 指向字符范围元素
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

// 返回 true 当给定部分 UTF-8 序列的某个延续可以满足 pos 处的字符范围（正向或反向范围）
// 断言 pos 指向字符范围元素
static bool llama_grammar_match_partial_char(
        const llama_grammar_element * pos,
        const llama_partial_utf8      partial_utf8) {

    bool is_positive_char = pos->type == LLAMA_GRETYPE_CHAR;
    GGML_ASSERT(is_positive_char || pos->type == LLAMA_GRETYPE_CHAR_NOT);

    uint32_t partial_value = partial_utf8.value;
    int      n_remain      = partial_utf8.n_remain;

    // 无效序列或 7 位字符跨越 2 个字节（过长）
    // 如果剩余的字节数小于0或者等于1且部分值小于2，则返回false
    if (n_remain < 0 || (n_remain == 1 && partial_value < 2)) {
        return false;
    }

    // 可能的 UTF-8 序列可以完成的代码点范围
    uint32_t low  = partial_value << (n_remain * 6);
    uint32_t high = low | ((1 << (n_remain * 6)) - 1);

    // 如果 low 等于0
    if (low == 0) {
        // 如果剩余的字节数为2
        if (n_remain == 2) {
            low = 1 << 11;
        } 
        // 如果剩余的字节数为3
        else if (n_remain == 3) {
            low = 1 << 16;
        }
    }

    // 循环处理
    do {
        // 如果下一个位置的类型为 inclusive range，例如 [a-z]
        if (pos[1].type == LLAMA_GRETYPE_CHAR_RNG_UPPER) {
            // 如果当前位置的值在范围内且下一个位置的值在范围内，则返回 is_positive_char
            if (pos->value <= high && low <= pos[1].value) {
                return is_positive_char;
            }
            pos += 2;
        } 
        // 如果下一个位置的类型为 exact char match，例如 [a] 或 "a"
        else {
            // 如果当前位置的值在范围内，则返回 is_positive_char
            if (low <= pos->value && pos->value <= high) {
                return is_positive_char;
            }
            pos += 1;
        }
    } while (pos->type == LLAMA_GRETYPE_CHAR_ALT);

    // 返回非 is_positive_char
    return !is_positive_char;
// 将一个语法下推栈转换为 N 个可能的栈，所有栈都以一个字符范围（终结符元素）结尾
static void llama_grammar_advance_stack(
        const std::vector<std::vector<llama_grammar_element>>   & rules,  // 语法规则的向量
        const std::vector<const llama_grammar_element *>        & stack,  // 下推栈的向量
        std::vector<std::vector<const llama_grammar_element *>> & new_stacks) {  // 新栈的向量

    // 如果下推栈为空
    if (stack.empty()) {
        // 将空栈加入到新栈中
        new_stacks.emplace_back(stack);
        return;
    }

    // 获取栈顶元素的指针
    const llama_grammar_element * pos = stack.back();
    // 根据 pos 指针指向的类型进行不同的处理
    switch (pos->type) {
        // 如果是规则引用类型
        case LLAMA_GRETYPE_RULE_REF: {
            // 获取规则引用的规则 ID
            const size_t rule_id = static_cast<size_t>(pos->value);
            // 获取规则引用对应的规则元素数组的指针
            const llama_grammar_element * subpos = rules[rule_id].data();
            // 循环处理规则引用对应的规则元素
            do {
                // 初始化一个不包含栈顶元素（pos）的新栈
                std::vector<const llama_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
                // 如果规则引用后面还有元素，则将其加入新栈
                if (!llama_grammar_is_end_of_sequence(pos + 1)) {
                    new_stack.push_back(pos + 1);
                }
                // 如果当前规则引用对应的规则元素不为空，则将其加入新栈
                if (!llama_grammar_is_end_of_sequence(subpos)) {
                    new_stack.push_back(subpos);
                }
                // 推进栈并生成新的栈
                llama_grammar_advance_stack(rules, new_stack, new_stacks);
                // 扫描到当前规则引用对应的规则元素的末尾
                while (!llama_grammar_is_end_of_sequence(subpos)) {
                    subpos++;
                }
                // 如果当前规则引用对应的规则元素类型为 LLAMA_GRETYPE_ALT，则继续处理下一个备选定义
                if (subpos->type == LLAMA_GRETYPE_ALT) {
                    subpos++;
                } else {
                    break;
                }
            } while (true);
            break;
        }
        // 如果是字符类型或者非字符类型
        case LLAMA_GRETYPE_CHAR:
        case LLAMA_GRETYPE_CHAR_NOT:
            // 将当前栈加入新栈集合
            new_stacks.emplace_back(stack);
            break;
        // 默认情况下，出现了不应该出现的情况，触发断言
        default:
            // 结束备选定义（LLAMA_GRETYPE_END, LLAMA_GRETYPE_ALT）或者在字符范围中间（LLAMA_GRETYPE_CHAR_ALT, LLAMA_GRETYPE_CHAR_RNG_UPPER）；栈不应该停留在这些情况下
            GGML_ASSERT(false);
    }
// 接受一个文法上可能的下推栈集合，这些栈需要位于一个字符范围上（参见`llama_grammar_advance_stack`），并在这些位置上接受给定字符时生成N个可能的栈
static std::vector<std::vector<const llama_grammar_element *>> llama_grammar_accept(
        const std::vector<std::vector<llama_grammar_element>>         & rules,  // 文法规则集合
        const std::vector<std::vector<const llama_grammar_element *>> & stacks, // 下推栈集合
        const uint32_t                                                  chr) {  // 给定字符

    std::vector<std::vector<const llama_grammar_element *>> new_stacks;  // 存储新的下推栈集合

    for (const auto & stack : stacks) {  // 遍历每个下推栈
        if (stack.empty()) {  // 如果栈为空，则跳过
            continue;
        }

        auto match = llama_grammar_match_char(stack.back(), chr);  // 匹配栈顶元素和给定字符
        if (match.first) {  // 如果匹配成功
            const llama_grammar_element * pos = match.second;  // 获取匹配的元素

            // 更新栈顶元素到下一个元素，如果有的话
            std::vector<const llama_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
            if (!llama_grammar_is_end_of_sequence(pos)) {  // 如果不是序列的结尾
                new_stack.push_back(pos);
            }
            llama_grammar_advance_stack(rules, new_stack, new_stacks);  // 推进栈
        }
    }

    return new_stacks;  // 返回新的下推栈集合
}

// 拒绝候选项
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates(
        const std::vector<std::vector<llama_grammar_element>>         & rules,  // 文法规则集合
        const std::vector<std::vector<const llama_grammar_element *>> & stacks, // 下推栈集合
        const std::vector<llama_grammar_candidate>                    & candidates);  // 候选项集合

// 为给定栈拒绝候选项
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates_for_stack(
        const std::vector<std::vector<llama_grammar_element>> & rules,  // 文法规则集合
        const std::vector<const llama_grammar_element *>      & stack,  // 下推栈
        const std::vector<llama_grammar_candidate>            & candidates) {  // 候选项集合

    std::vector<llama_grammar_candidate> rejects;  // 存储拒绝的候选项
    // 如果栈为空，则遍历候选项，将不符合条件的候选项加入到拒绝列表中
    if (stack.empty()) {
        for (const auto & tok : candidates) {
            if (*tok.code_points != 0 || tok.partial_utf8.n_remain != 0) {
                rejects.push_back(tok);
            }
        }
        return rejects;
    }

    // 获取栈顶元素
    const llama_grammar_element * stack_pos = stack.back();

    // 创建下一轮候选项列表
    std::vector<llama_grammar_candidate> next_candidates;
    for (const auto & tok : candidates) {
        if (*tok.code_points == 0) {
            // 如果候选项已经到达完整的字符序列末尾，且以无法满足语法位置的部分序列结尾，则加入到拒绝列表中
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

    // 获取栈顶元素匹配空字符后的位置
    const auto * stack_pos_after = llama_grammar_match_char(stack_pos, 0).second;

    // 更新栈顶元素为下一个元素（如果有的话）
    std::vector<const llama_grammar_element *> stack_after(stack.begin(), stack.end() - 1);
    if (!llama_grammar_is_end_of_sequence(stack_pos_after)) {
        stack_after.push_back(stack_pos_after);
    }
    // 获取下一轮栈的可能情况
    std::vector<std::vector<const llama_grammar_element *>> next_stacks;
    llama_grammar_advance_stack(rules, stack_after, next_stacks);

    // 对下一轮候选项进行拒绝处理
    auto next_rejects = llama_grammar_reject_candidates(rules, next_stacks, next_candidates);
    for (const auto & tok : next_rejects) {
        rejects.push_back({ tok.index, tok.code_points - 1, tok.partial_utf8 });
    }

    return rejects;
// 静态函数，用于拒绝候选项，根据规则、堆栈和候选项来筛选出不符合条件的候选项
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates(
        const std::vector<std::vector<llama_grammar_element>>         & rules,
        const std::vector<std::vector<const llama_grammar_element *>> & stacks,
        const std::vector<llama_grammar_candidate>                    & candidates) {
    // 断言堆栈不为空
    GGML_ASSERT(!stacks.empty()); // REVIEW

    // 如果候选项为空，则返回空的候选项向量
    if (candidates.empty()) {
        return std::vector<llama_grammar_candidate>();
    }

    // 调用 llama_grammar_reject_candidates_for_stack 函数，对第一个堆栈的候选项进行筛选
    auto rejects = llama_grammar_reject_candidates_for_stack(rules, stacks.front(), candidates);

    // 遍历堆栈，对每个堆栈的候选项进行筛选
    for (size_t i = 1, size = stacks.size(); i < size; ++i) {
        rejects = llama_grammar_reject_candidates_for_stack(rules, stacks[i], rejects);
    }
    // 返回筛选后的候选项
    return rejects;
}

//
// grammar - external
//

// 初始化 llama_grammar 结构体
struct llama_grammar * llama_grammar_init(
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

    // 循环遍历起始规则的替代规则，构建初始堆栈
    std::vector<std::vector<const llama_grammar_element *>> stacks;
    pos = rules[start_rule_index];
    // 使用 do-while 循环处理语法规则的不同选择
    do {
        // 创建一个存储语法元素指针的向量
        std::vector<const llama_grammar_element *> stack;
        // 如果当前位置不是序列的结尾
        if (!llama_grammar_is_end_of_sequence(pos)) {
            // 如果有备选项，将其添加到栈中
            stack.push_back(pos);
        }
        // 推进栈
        llama_grammar_advance_stack(vec_rules, stack, stacks);
        // 扫描到备选定义的末尾
        while (!llama_grammar_is_end_of_sequence(pos)) {
            pos++;
        }
        // 如果当前位置的类型是备选项
        if (pos->type == LLAMA_GRETYPE_ALT) {
            // 还有这个规则的另一个备选定义需要处理
            pos++;
        } else {
            // 否则跳出循环
            break;
        }
    } while (true);

    // 返回一个新的 llama_grammar 对象，包含移动后的规则向量、栈向量和空的映射
    return new llama_grammar{ std::move(vec_rules), std::move(stacks), {} };
// 释放 llama_grammar 结构体指针所指向的内存
void llama_grammar_free(struct llama_grammar * grammar) {
    delete grammar;
}

// 复制 llama_grammar 结构体指针指向的内容，并返回新的 llama_grammar 结构体指针
struct llama_grammar * llama_grammar_copy(const struct llama_grammar * grammar) {
    // 使用原始 grammar 的规则、堆栈和 partial_utf8 创建新的 llama_grammar 结构体指针
    llama_grammar * result = new llama_grammar{ grammar->rules, grammar->stacks, grammar->partial_utf8 };

    // 将堆栈中指向原始规则的指针重定向到新的规则
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

//
// 采样
//

// 设置 llama_context 结构体指针中的随机数种子
void llama_set_rng_seed(struct llama_context * ctx, uint32_t seed) {
    // 如果种子为默认值，则使用当前时间作为种子
    if (seed == LLAMA_DEFAULT_SEED) {
        seed = time(NULL);
    }
    ctx->rng.seed(seed);
}

// 对 softmax 函数进行采样
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

    // 更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}
// 从候选项中采样出前 k 个元素，保留至少 min_keep 个元素
void llama_sample_top_k(struct llama_context * ctx, llama_token_data_array * candidates, int32_t k, size_t min_keep) {
    // TODO: 将桶排序移到单独的函数中，以便 top_p/tail_free/typical/softmax first 具有相同的速度
    // if (k >= (int32_t)candidates->size) {
    //     return;
    // }

    // 记录采样开始时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 确保 k 至少为 min_keep
    k = std::max(k, (int) min_keep);
    // 确保 k 不超过候选项的大小
    k = std::min(k, (int) candidates->size);

    // 按照分数降序排序
    }
    candidates->size = k;

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 从候选项中采样出概率前 p 的元素，保留至少 min_keep 个元素
void llama_sample_top_p(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
    // 如果 p 大于等于 1.0，则直接返回
    if (p >= 1.0f) {
        return;
    }

    // 对候选项进行 softmax 处理
    llama_sample_softmax(ctx, candidates);

    // 记录采样开始时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 计算累积概率
    float cum_sum = 0.0f;
    size_t last_idx = candidates->size;

    for (size_t i = 0; i < candidates->size; ++i) {
        cum_sum += candidates->data[i].p;

        // 检查累积概率是否大于等于 p，或者是否至少保留了 min_keep 个元素
        // 如果是，则将最后一个索引设置为 i+1，表示当前迭代应包含在集合中
        if (cum_sum >= p && i + 1 >= min_keep) {
            last_idx = i + 1;
            break;
        }
    }

    // 调整输出向量的大小，仅保留前 p 的元素
    candidates->size = last_idx;

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 从候选项中采样出概率至少为 p 的元素，保留至少 min_keep 个元素
void llama_sample_min_p(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
    // 如果 p 小于等于 0.0，或者候选项为空，则直接返回
    if (p <= 0.0f || !candidates->size) {
        return;
    }

    // 记录采样开始时间
    const int64_t t_start_sample_us = ggml_time_us();

    bool min_p_applied = false;

    // 如果候选项未排序，则首先尝试未排序的实现
}
    // 如果候选项未排序
    if (!candidates->sorted) {
        // 创建一个新的向量用于存储筛选后的候选项
        std::vector<llama_token_data> filtered_tokens;

        // 初始化最大 logit 值为负无穷
        float max_logit = -FLT_MAX;
        // 遍历所有候选项，找到最大的 logit 值
        for (size_t i = 0; i < candidates->size; ++i) {
            max_logit = std::max(max_logit, candidates->data[i].logit);
        }
        // 计算最小的 logit 值，用于筛选 p_i >= p * p_max 的候选项
        const float min_logit = max_logit + logf(p);

        // 遍历所有候选项，将满足条件的候选项加入到筛选后的向量中
        for (size_t i = 0; i < candidates->size; ++i) {
            if (candidates->data[i].logit >= min_logit) {
                filtered_tokens.push_back(candidates->data[i]);
            }
        }

        // 如果筛选后的候选项数量大于等于最小保留数量，则操作成功
        if (filtered_tokens.size() >= min_keep) {
            // 将筛选后的候选项拷贝回原始候选项数组中
            memcpy(candidates->data, filtered_tokens.data(), filtered_tokens.size()*sizeof(llama_token_data));
            candidates->size = filtered_tokens.size();
            min_p_applied = true;
        }
    }

    // 如果候选项已排序或者未排序实现失败，则使用此实现
    if (!min_p_applied) {
        // 如果候选项未排序，则按 logit 值降序排序
        if (!candidates->sorted) {
            std::sort(candidates->data, candidates->data + candidates->size, [](const llama_token_data & a, const llama_token_data & b) {
                return a.logit > b.logit;
            });
            candidates->sorted = true;
        }

        // 计算最小的 logit 值，用于筛选 p_i >= p * p_max 的候选项
        const float min_logit = candidates->data[0].logit + logf(p);
        size_t i = 1; // 第一个 token 总是匹配的

        // 遍历候选项，找到第一个不满足条件的 token
        for (; i < candidates->size; ++i) {
            if (candidates->data[i].logit < min_logit && i >= min_keep) {
                break; // 概率太小
            }
        }

        // 调整输出向量的大小，仅保留匹配的 token
        candidates->size = i;
    }

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 释放尾部样本，根据给定的 z 值和最小保留数量来筛选候选项
    void llama_sample_tail_free(struct llama_context * ctx, llama_token_data_array * candidates, float z, size_t min_keep) {
        // 如果 z 大于等于 1.0 或者候选项数量小于等于 2，则直接返回
        if (z >= 1.0f || candidates->size <= 2) {
            return;
        }

        // 对候选项进行 softmax 处理
        llama_sample_softmax(nullptr, candidates);
        // 记录开始采样的时间戳
        const int64_t t_start_sample_us = ggml_time_us();

        // 计算一阶和二阶导数
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

        // 归一化二阶导数
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
    }
    # 检查指针 ctx 是否存在，如果存在则执行下面的操作
    if (ctx) {
        # 更新 ctx 结构体中 t_sample_us 字段的数值，增加当前时间与 t_start_sample_us 之间的时间差
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
// 定义一个函数，用于从候选项中典型地抽样一部分，保留概率大于p的token，同时保留至少min_keep个token
void llama_sample_typical(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
    // 如果概率p大于等于1.0，直接返回
    if (p >= 1.0f) {
        return;
    }

    // 计算logits的softmax并计算熵
    llama_sample_softmax(nullptr, candidates);

    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 初始化熵值
    float entropy = 0.0f;
    for (size_t i = 0; i < candidates->size; ++i) {
        // 计算熵值
        entropy += -candidates->data[i].p * logf(candidates->data[i].p);
    }

    // 计算每个候选项的负对数概率和熵值之间的绝对差
    std::vector<float> shifted_scores;
    for (size_t i = 0; i < candidates->size; ++i) {
        float shifted_score = fabsf(-logf(candidates->data[i].p) - entropy);
        shifted_scores.push_back(shifted_score);
    }

    // 根据shifted_scores和对应的索引对token进行排序
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

        // 检查累积和是否大于typical或者是否至少保留了min_keep个token
        if (cum_sum > p && i >= min_keep - 1) {
            last_idx = i + 1;
            break;
        }
    }

    // 调整输出向量的大小，仅保留局部typical的token
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
    // 将 candidates 标记为未排序状态
    candidates->sorted = false;

    // 如果上下文对象存在
    if (ctx) {
        // 更新上下文对象中的采样时间
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 计算样本熵
void llama_sample_entropy(struct llama_context * ctx, llama_token_data_array * candidates_p, float min_temp, float max_temp, float exponent_val) {
    // 记录开始样本时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 如果只有一个（或零个）候选项，则无需执行任何操作
    if(candidates_p->size <= 1) {
        return;
    }

    // 计算最大可能熵
    float max_entropy = -logf(1.0f / candidates_p->size);

    // 对 softmax 概率进行计算
    llama_sample_softmax(nullptr, candidates_p);

    // 计算 softmax 概率的熵
    float entropy = 0.0f;
    for (size_t i = 0; i < candidates_p->size; ++i) {
        float prob = candidates_p->data[i].p;
        if (prob > 0.0f) { // 确保不计算 log(0)
            entropy -= prob * logf(prob);
        }
    }

    // 标准化熵（这里 max_entropy 不可能为 0，因为上面已经检查了 candidates_p->size != 1）
    float normalized_entropy = entropy / max_entropy;

    // 使用幂函数将标准化熵映射到所需温度范围
    float dyn_temp = min_temp + (max_temp - min_temp) * powf(normalized_entropy, exponent_val);

#ifdef DEBUG
    LLAMA_LOG_INFO("Your text maxtemp value is: %f\n", max_temp);
    LLAMA_LOG_INFO("Entropy: %f\n", entropy);
    LLAMA_LOG_INFO("Max Possible Entropy: %f\n", max_entropy);
    LLAMA_LOG_INFO("Normalized Entropy: %f\n", normalized_entropy);
    LLAMA_LOG_INFO("Exponent: %f\n", exponent_val);
    LLAMA_LOG_INFO("Dynamic Temperature (dyn_temp): %f\n", dyn_temp);
#endif

    // 应用动态计算的温度缩放
    for (size_t i = 0; i < candidates_p->size; ++i) {
        candidates_p->data[i].logit /= dyn_temp;
    }

    // 在使用动态温度缩放后重新计算 softmax 概率
    double max_l_double = candidates_p->data[0].logit;
    double cum_sum_double = 0.0;
    // 遍历候选项数组中的每个元素
    for (size_t i = 0; i < candidates_p->size; ++i) {
        // 计算概率的指数值，用于缩放概率
        double p = exp(candidates_p->data[i].logit - max_l_double);
        // 将缩放后的概率存储在候选项数组中
        candidates_p->data[i].p = p; // Store the scaled probability
        // 累加缩放后的概率，用于后续概率归一化
        cum_sum_double += p;
    }
    // 再次遍历候选项数组中的每个元素
    for (size_t i = 0; i < candidates_p->size; ++i) {
        // 将缩放后的概率除以累加值，实现概率的归一化
        candidates_p->data[i].p /= cum_sum_double; // Re-normalize the probabilities
    }
#ifdef DEBUG
    // 如果处于调试模式，打印动态温度缩放后的前25个概率
    LLAMA_LOG_INFO("\nUpdated Top 25 Probabilities After Dynamic Temperature Scaling (in percentages):\n");
    // 遍历前25个候选项，打印每个候选项的概率
    for (size_t i = 0; i < 25 && i < candidates_p->size; ++i) {
        LLAMA_LOG_INFO("Token %zu: %f%%\n", i + 1, candidates_p->data[i].p * 100.0f);
    }
#endif

    // 如果上下文存在，更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 根据温度对候选项进行采样
void llama_sample_temp(struct llama_context * ctx, llama_token_data_array * candidates_p, float temp) {
    const int64_t t_start_sample_us = ggml_time_us();

    // 根据温度对每个候选项的logit值进行调整
    for (size_t i = 0; i < candidates_p->size; ++i) {
        candidates_p->data[i].logit /= temp;
    }

    // 如果上下文存在，更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 对候选项进行温度采样
void llama_sample_temperature(struct llama_context * ctx, llama_token_data_array * candidates_p, float temp) {
    llama_sample_temp(ctx, candidates_p, temp);
}

// 对候选项应用重复惩罚
void llama_sample_repetition_penalties(
            struct llama_context * ctx,
          llama_token_data_array * candidates,
               const llama_token * last_tokens,
                          size_t   penalty_last_n,
                           float   penalty_repeat,
                           float   penalty_freq,
                           float   penalty_present) {
    // 如果不需要重复惩罚，直接返回
    if (penalty_last_n == 0 || (penalty_repeat == 1.0f && penalty_freq == 0.0f && penalty_present == 0.0f)) {
        return;
    }

    const int64_t t_start_sample_us = ggml_time_us();

    // 创建一个频率映射，统计last_tokens中每个token的出现次数
    std::unordered_map<llama_token, int> token_count;
    for (size_t i = 0; i < penalty_last_n; ++i) {
        token_count[last_tokens[i]]++;
    }

    // 对候选项应用频率和存在性惩罚
    // 遍历候选项数组中的每个元素
    for (size_t i = 0; i < candidates->size; ++i) {
        // 查找当前候选项的 ID 在 token_count 中的计数
        const auto token_iter = token_count.find(candidates->data[i].id);
        // 如果在 token_count 中找不到当前候选项的 ID，则跳过当前循环
        if (token_iter == token_count.end()) {
            continue;
        }

        // 获取当前候选项的计数
        const int count = token_iter->second;

        // 如果当前候选项的 logit 值小于等于 0，则乘以 penalty_repeat；否则除以 penalty_repeat
        if (candidates->data[i].logit <= 0) {
            candidates->data[i].logit *= penalty_repeat;
        } else {
            candidates->data[i].logit /= penalty_repeat;
        }

        // 更新当前候选项的 logit 值，根据计数和惩罚值进行调整
        candidates->data[i].logit -= float(count) * penalty_freq + float(count > 0) * penalty_present;
    }

    // 将候选项数组标记为未排序状态
    candidates->sorted = false;

    // 如果上下文对象存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
// 定义函数 llama_sample_grammar，用于从给定的候选项中生成样本语法
void llama_sample_grammar(struct llama_context * ctx, llama_token_data_array * candidates, const struct llama_grammar * grammar) {
    // 断言上下文不为空
    GGML_ASSERT(ctx);
    // 获取当前时间戳，单位为微秒
    const int64_t t_start_sample_us = ggml_time_us();

    // 初始化变量 allow_eos 为 false
    bool allow_eos = false;
    // 遍历语法中的栈，如果栈为空，则将 allow_eos 设置为 true 并跳出循环
    for (const auto & stack : grammar->stacks) {
        if (stack.empty()) {
            allow_eos = true;
            break;
        }
    }

    // 获取结束符 token
    const llama_token eos = llama_token_eos(&ctx->model);

    // 初始化存储解码后候选项的容器
    std::vector<std::pair<std::vector<uint32_t>, llama_partial_utf8>> candidates_decoded;
    candidates_decoded.reserve(candidates->size);
    // 初始化存储语法候选项的容器
    std::vector<llama_grammar_candidate> candidates_grammar;
    candidates_grammar.reserve(candidates->size);

    // 遍历候选项数组
    for (size_t i = 0; i < candidates->size; ++i) {
        // 获取候选项的 token id 和对应的字符串
        const llama_token id    = candidates->data[i].id;
        const std::string piece = llama_token_to_piece(ctx, id);
        // 如果 token id 为结束符
        if (id == eos) {
            // 如果不允许结束符，则将该候选项的 logit 设置为负无穷
            if (!allow_eos) {
                candidates->data[i].logit = -INFINITY;
            }
        } else if (piece.empty() || piece[0] == 0) {
            // 如果字符串为空或第一个字符为 0，则将该候选项的 logit 设置为负无穷
            candidates->data[i].logit = -INFINITY;
        } else {
            // 解码字符串为 UTF-8，并存储到相应容器中
            candidates_decoded.push_back(decode_utf8(piece, grammar->partial_utf8));
            candidates_grammar.push_back({ i, candidates_decoded.back().first.data(), candidates_decoded.back().second });
        }
    }

    // 拒绝不符合语法规则的候选项
    const auto rejects = llama_grammar_reject_candidates(grammar->rules, grammar->stacks, candidates_grammar);
    for (const auto & reject : rejects) {
        candidates->data[reject.index].logit = -INFINITY;
    }

    // 更新采样时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

// 定义静态函数 llama_log_softmax，用于对数组进行 log softmax 操作
static void llama_log_softmax(float * array, size_t size) {
    // 获取数组中的最大值
    float max_l = *std::max_element(array, array + size);
    float sum = 0.f;
    // 计算 softmax 分母
    for (size_t i = 0; i < size; ++i) {
        float p = expf(array[i] - max_l);
        sum += p;
        array[i] = p;
    }

    // 计算 log softmax
    for (size_t i = 0; i < size; ++i) {
        array[i] = logf(array[i] / sum);
    }
}
    }

这是一个代码块的结束标记，表示一个函数或者一个循环的结束。
// 应用指导信息到样本中，更新logits数组中的值
void llama_sample_apply_guidance(
          struct llama_context * ctx,
                         float * logits,
                         float * logits_guidance,
                         float   scale) {
    // 断言上下文不为空
    GGML_ASSERT(ctx);

    // 记录开始采样时间
    const auto t_start_sample_us = ggml_time_us();
    // 获取词汇表大小
    const auto n_vocab = llama_n_vocab(llama_get_model(ctx));

    // 对logits数组进行log_softmax操作
    llama_log_softmax(logits, n_vocab);
    // 对logits_guidance数组进行log_softmax操作
    llama_log_softmax(logits_guidance, n_vocab);

    // 遍历词汇表，更新logits数组中的值
    for (int i = 0; i < n_vocab; ++i) {
              auto & l = logits[i];
        const auto & g = logits_guidance[i];

        l = scale * (l - g) + g;
    }

    // 更新上下文中的采样时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

// 释放指导信息后的分类器样本
void llama_sample_classifier_free_guidance(
          struct llama_context * ctx,
        llama_token_data_array * candidates,
          struct llama_context * guidance_ctx,
                         float   scale) {
    // 断言上下文不为空
    GGML_ASSERT(ctx);
    int64_t t_start_sample_us;

    // 记录开始采样时间
    t_start_sample_us = ggml_time_us();
    // 获取词汇表大小
    const size_t n_vocab = llama_n_vocab(llama_get_model(ctx));

    // 断言词汇表大小与候选数组大小相等
    GGML_ASSERT(n_vocab == candidates->size);
    // 断言候选数组未排序
    GGML_ASSERT(!candidates->sorted);

    // 创建logits_base数组，存储候选数组中的logit值
    std::vector<float> logits_base(n_vocab);
    for (size_t i = 0; i < n_vocab; ++i) {
        logits_base[i] = candidates->data[i].logit;
    }

    // 获取指导信息上下文中的logits数组
    float * logits_guidance = llama_get_logits(guidance_ctx);

    // 更新上下文中的采样时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    // 应用指导信息到分类器样本中
    llama_sample_apply_guidance(ctx, logits_base.data(), logits_guidance, scale);
    // 重新记录开始采样时间
    t_start_sample_us = ggml_time_us();

    // 将更新后的logits_base数组中的值写回候选数组
    for (size_t i = 0; i < n_vocab; ++i) {
        candidates->data[i].logit = logits_base[i];
    }

    // 更新上下文中的采样时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

// 对token进行采样，返回采样后的token
llama_token llama_sample_token_mirostat(struct llama_context * ctx, llama_token_data_array * candidates, float tau, float eta, int32_t m, float * mu) {
    // 断言上下文不为空
    GGML_ASSERT(ctx);

    // 获取词汇表大小并转换为浮点数
    auto N = float(llama_n_vocab(llama_get_model(ctx)));
    int64_t t_start_sample_us;
    // 记录开始采样时间
    t_start_sample_us = ggml_time_us();
    // 调用 llama_sample_softmax 函数，传入空指针和候选项
    llama_sample_softmax(nullptr, candidates);

    // 通过最可能的 m 个标记估计 s_hat
    float s_hat = 0.0;
    float sum_ti_bi = 0.0;
    float sum_ti_sq = 0.0;
    for (size_t i = 0; i < size_t(m - 1) && i < candidates->size - 1; ++i) {
        // 计算 t_i
        float t_i = logf(float(i + 2) / float(i + 1));
        // 计算 b_i
        float b_i = logf(candidates->data[i].p / candidates->data[i + 1].p);
        sum_ti_bi += t_i * b_i;
        sum_ti_sq += t_i * t_i;
    }
    s_hat = sum_ti_bi / sum_ti_sq;

    // 根据估计的 s_hat 和目标惊喜值计算 k
    float epsilon_hat = s_hat - 1;
    float k = powf((epsilon_hat * powf(2, *mu)) / (1 - powf(N, -epsilon_hat)), 1 / s_hat);

    // 使用 top-k 抽样方法抽样下一个词 X
    llama_sample_top_k(nullptr, candidates, int(k), 1);
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 从候选项中抽样一个标记 X
    llama_token X = llama_sample_token(ctx, candidates);
    t_start_sample_us = ggml_time_us();

    // 计算观察到的惊喜和目标惊喜值之间的误差
    size_t X_idx = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return candidate.id == X;
    }));
    float observed_surprise = -log2f(candidates->data[X_idx].p);
    float e = observed_surprise - tau;

    // 使用学习率和误差更新 mu
    *mu = *mu - eta * e;

    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 返回抽样的标记 X
    return X;
// 从给定的候选词数组中使用 Mirostat V2 算法进行抽样，返回抽样得到的词的 token
llama_token llama_sample_token_mirostat_v2(struct llama_context * ctx, llama_token_data_array * candidates, float tau, float eta, float * mu) {
    // 记录抽样开始的时间戳
    int64_t t_start_sample_us;
    t_start_sample_us = ggml_time_us();

    // 使用 softmax 函数对候选词进行抽样
    llama_sample_softmax(ctx, candidates);

    // 截断那些超过 mu 阈值的词
    candidates->size = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return -log2f(candidate.p) > *mu;
    }));

    // 如果候选词为空，则将其大小设置为 1
    if (candidates->size == 0) {
        candidates->size = 1;
    }

    // 如果上下文对象存在，则更新抽样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }

    // 对剩余候选词的概率进行归一化
    llama_sample_softmax(ctx, candidates);

    // 从剩余候选词中抽样出下一个词 X
    llama_token X = llama_sample_token(ctx, candidates);
    t_start_sample_us = ggml_time_us();

    // 计算误差，即观察到的惊讶值与目标惊讶值之间的差异
    size_t X_idx = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return candidate.id == X;
    }));
    float observed_surprise = -log2f(candidates->data[X_idx].p);
    float e = observed_surprise - tau;

    // 使用学习率和误差更新 mu
    *mu = *mu - eta * e;

    // 如果上下文对象存在，则更新抽样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 返回抽样得到的词的 token
    return X;
}

// 从给定的候选词数组中使用贪婪算法进行抽样，返回抽样得到的词的 token
llama_token llama_sample_token_greedy(struct llama_context * ctx, llama_token_data_array * candidates) {
    // 记录抽样开始的时间戳
    const int64_t t_start_sample_us = ggml_time_us();

    // 找到 logit 值最大的元素
    auto * max_iter = std::max_element(candidates->data, candidates->data + candidates->size, [](const llama_token_data & a, const llama_token_data & b) {
        return a.logit < b.logit;
    });

    // 返回 logit 值最大的元素的 token
    llama_token result = max_iter->id;
}
    # 如果上下文对象存在
    if (ctx) {
        # 更新上下文对象中的采样时间，累加当前时间减去开始采样时间
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
        # 增加采样次数
        ctx->n_sample++;
    }
    # 返回结果
    return result;
// 返回一个 token，用于语言模型的采样
llama_token llama_sample_token(struct llama_context * ctx, llama_token_data_array * candidates) {
    // 断言上下文不为空
    GGML_ASSERT(ctx);

    // 记录采样开始时间
    const int64_t t_start_sample_us = ggml_time_us();
    // 对候选 token 进行 softmax 处理
    llama_sample_softmax(nullptr, candidates);

    // 创建一个存储概率的向量
    std::vector<float> probs;
    probs.reserve(candidates->size);
    // 遍历候选 token，将概率添加到向量中
    for (size_t i = 0; i < candidates->size; ++i) {
        probs.push_back(candidates->data[i].p);
    }

    // 创建一个离散分布对象
    std::discrete_distribution<> dist(probs.begin(), probs.end());
    // 获取上下文中的随机数生成器
    auto & rng = ctx->rng;
    // 从分布中采样一个索引
    int idx = dist(rng);

    // 获取采样到的 token
    llama_token result = candidates->data[idx].id;

    // 更新采样时间和采样次数
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    ctx->n_sample++;
    // 返回结果 token
    return result;
}

// 接受一个 token 并更新语法栈
void llama_grammar_accept_token(struct llama_context * ctx, struct llama_grammar * grammar, llama_token token) {
    // 记录接受 token 开始时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 如果 token 是句子结束符，则检查语法栈是否为空，若为空则返回，否则断言失败
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

    // 解码字符串并获取码点
    const auto decoded = decode_utf8(piece, grammar->partial_utf8);
    const auto & code_points = decoded.first;
    // 遍历码点并更新语法栈
    for (auto it = code_points.begin(), end = code_points.end() - 1; it != end; ++it) {
        grammar->stacks = llama_grammar_accept(grammar->rules, grammar->stacks, *it);
    }
    // 更新部分 UTF-8 字符串
    grammar->partial_utf8 = decoded.second;
    // 断言语法栈不为空
    GGML_ASSERT(!grammar->stacks.empty());

    // 更新接受 token 的时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

//
// Beam search
//

// 定义一个 beam 结构体
struct llama_beam {
    std::vector<llama_token> tokens; // 存储 token 的向量
    float p;  // 累积 beam 概率（相对于所有 beam 进行重新归一化）
    bool eob; // 初始化 end-of-beam 为 false。回调函数将其设置为 true。
    // 按概率对 beam 进行排序。在概率相同时，优先选择 eob 的 beam。
    // 定义小于运算符，比较两个 llama_beam 对象的 p 和 eob 成员变量
    bool operator<(const llama_beam & rhs) const {
        // 使用 std::make_pair 创建一对 pair 对象，比较两个 pair 对象的大小
        return std::make_pair(p, eob) < std::make_pair(rhs.p, rhs.eob);
    }
    
    // 移除前 n 个令牌并丢弃它们
    void shift_tokens(const size_t n) {
        // 如果 n 不为 0
        if (n) {
            // 将 tokens 中从第 n 个元素开始到末尾的元素复制到 tokens 的开头
            std::copy(tokens.begin() + n, tokens.end(), tokens.begin());
            // 调整 tokens 的大小，移除 n 个元素
            tokens.resize(tokens.size() - n);
        }
    }
    
    // 返回一个 llama_beam_view 对象，包含 tokens 数组的数据指针、大小、p 和 eob 成员变量的值
    llama_beam_view view() const { return {tokens.data(), tokens.size(), p, eob}; }
// 用于计算与 logit 相关信息的结构体
struct llama_logit_info {
    // 指向 logits 数组的指针
    const float * const logits;
    // 词汇表大小
    const int n_vocab;
    // logits 数组中的最大值
    const float max_l;
    // 归一化因子
    const float normalizer;
    // 内部结构体，用于计算 sum(exp(l - max_l))
    struct sum_exp {
        // 最大值 max_l
        float max_l;
        // 重载 () 运算符，计算 sum + std::exp(l - max_l)
        float operator()(float sum, float l) const { return sum + std::exp(l - max_l); }
    };
    // 构造函数，初始化结构体成员
    llama_logit_info(llama_context * ctx)
      : logits(llama_get_logits(ctx))
      , n_vocab(llama_n_vocab(llama_get_model(ctx)))
      , max_l(*std::max_element(logits, logits + n_vocab))
      , normalizer(1.0f / std::accumulate(logits, logits + n_vocab, 0.0f, sum_exp{max_l}))
      { }
    // 获取 token_id 对应的 token_data
    llama_token_data get_token_data(const llama_token token_id) const {
        // 未使用的常量 p
        constexpr auto p = std::numeric_limits<float>::quiet_NaN();  // never used
        return {token_id, logits[token_id], p};
    }
    // 返回 logit 最高的前 k 个 token_data
    std::vector<llama_token_data> top_k(size_t k) {
        // 以 logit 为关键字的小顶堆
        std::vector<llama_token_data> min_heap;  // min-heap by logit
        const llama_token k_min = std::min(static_cast<llama_token>(k), n_vocab);
        min_heap.reserve(k_min);
        // 初始化小顶堆
        for (llama_token token_id = 0 ; token_id < k_min ; ++token_id) {
            min_heap.push_back(get_token_data(token_id));
        }
        auto comp = [](const llama_token_data & a, const llama_token_data & b) { return a.logit > b.logit; };
        std::make_heap(min_heap.begin(), min_heap.end(), comp);
        // 更新小顶堆
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
    // 根据 logit 计算概率
    float probability_from_logit(float logit) const {
        return normalizer * std::exp(logit - max_l);
    }
};
// 定义结构体 llama_beam_search_data，包含了多个成员变量和方法
struct llama_beam_search_data {
    llama_context * ctx; // 指向 llama_context 类型的指针
    size_t n_beams; // 用于存储束搜索的数量
    int n_past; // 用于存储过去的数量
    int n_predict; // 用于存储预测的数量
    std::vector<llama_beam> beams; // 存储 llama_beam 对象的向量
    std::vector<llama_beam> next_beams; // 存储下一个束搜索的 llama_beam 对象的向量

    // 在每次循环迭代中重新计算的变量
    size_t common_prefix_length; // 存储公共前缀的长度

    // 用于在回调函数中传递和接收束搜索状态的变量
    std::vector<llama_beam_view> beam_views; // 存储 llama_beam_view 对象的向量

    // 构造函数，初始化成员变量
    llama_beam_search_data(llama_context * ctx, size_t n_beams, int n_past, int n_predict)
      : ctx(ctx)
      , n_beams(n_beams)
      , n_past(n_past)
      , n_predict(n_predict)
      , beam_views(n_beams) {
        beams.reserve(n_beams); // 预留空间以存储 n_beams 个元素
        next_beams.reserve(n_beams); // 预留空间以存储 n_beams 个元素
    }

    // 将束搜索合并为给定索引的单个束搜索
    void collapse_beams(const size_t beam_idx) {
        if (0u < beam_idx) {
            std::swap(beams[0], beams[beam_idx]); // 交换 beams[0] 和 beams[beam_idx] 的值
        }
        beams.resize(1); // 将 beams 的大小调整为 1
    }

    // 使用最小堆有效地收集前 k 个元素（k=n_beams）
    // 下面的重复模式反映了堆的两个阶段：
    //  * 收集元素直到向量已满，然后在其上调用 std::make_heap()
    //  * 如果堆已满且找到应包含的新元素，则将最小元素弹出到 back()，用新元素替换它，然后将其推入堆中
    }

    // 基于束搜索查找公共前缀长度
    // 要求 beams 不为空
    size_t find_common_prefix_length() {
        size_t common_prefix_length = beams[0].tokens.size(); // 初始化 common_prefix_length 为第一个束搜索的 tokens 大小
        for (size_t i = 1 ; i < beams.size() ; ++i) {
            common_prefix_length = std::min(common_prefix_length, beams[i].tokens.size()); // 更新 common_prefix_length 为最小的 tokens 大小
            for (size_t j = 0 ; j < common_prefix_length ; ++j) {
                if (beams[0].tokens[j] != beams[i].tokens[j]) {
                    common_prefix_length = j; // 如果发现不同的 token，则更新 common_prefix_length
                    break;
                }
            }
        }
        return common_prefix_length; // 返回计算得到的公共前缀长度
    }
}
    // 构造 beams_state 以通过回调函数发送回调函数的调用者
    // 副作用：设置 common_prefix_length = find_common_prefix_length();
    llama_beams_state get_beams_state(const bool last_call) {
        // 遍历 beams 数组，获取每个 beam 的视图并存储在 beam_views 数组中
        for (size_t i = 0 ; i < beams.size() ; ++i) {
            beam_views[i] = beams[i].view();
        }
        // 调用 find_common_prefix_length() 函数获取公共前缀长度
        common_prefix_length = find_common_prefix_length();
        // 返回包含 beam 视图、beam 数量、公共前缀长度和是否为最后一次调用的结构体
        return {beam_views.data(), beams.size(), common_prefix_length, last_call};
    }

    // 循环：
    //  * 当 i < n_predict 且
    //  * 任何一个 beam 尚未到达结束（eob）且
    //  * 最高概率的 beam（可能有多个并列）尚未到达句子结尾
    //    （因为所有其他 beam 的概率只会减小）
    // 循环执行 beam search 过程，直到达到预测次数或者所有 beam 都结束或者顶部 beam 结束
    void loop(const llama_beam_search_callback_fn_t callback, void * const callback_data) {
        // 初始化一个空的 beam，概率为 1.0，结束标志为 false
        beams.push_back({{}, 1.0f, false});
        // 定义一个 lambda 函数，用于判断 beam 是否结束
        const auto not_eob = [](const llama_beam & beam) { return !beam.eob; };
        // 循环执行 beam search 过程
        for (int i = 0 ; i < n_predict && std::any_of(beams.begin(),beams.end(),not_eob) &&
                       !beams[top_beam_index()].eob ; ++i) {
            // 调用回调函数，更新共同前缀长度
            callback(callback_data, get_beams_state(false));
            // 更新 beam 的值（概率和结束标志）
            update_beams_from_beam_views();
            // 如果存在共同前缀长度
            if (common_prefix_length) {
                // 解码共同前缀长度的 token
                llama_decode(ctx, llama_batch_get_one(beams[0].tokens.data(), common_prefix_length, n_past, 0));
                n_past += common_prefix_length;
            }
            // 将下一个 beam 的概率置为 0，以便后续放入最小堆
            std::for_each(next_beams.begin(), next_beams.end(), [](llama_beam & beam) { beam.p = 0.0f; });
            // 遍历当前 beams，更新下一个 beams 的概率
            for (llama_beam & beam : beams) {
                beam.shift_tokens(common_prefix_length);
                fill_next_beams_by_top_probabilities(beam);
            }
            // 将 next_beams 替换为当前 beams，以便重复使用内存
            beams.swap(next_beams);
            // 重新计算 beam 的概率，避免浮点数下溢
            renormalize_beam_probabilities(beams);
        }
        // 合并 beams，获取最终结果
        collapse_beams(top_beam_index());
        callback(callback_data, get_beams_state(true));
    }

    // 随着 beams 增多，累积概率会减小，需要重新归一化
    static void renormalize_beam_probabilities(std::vector<llama_beam> & beams) {
        // 定义 lambda 函数，用于计算概率之和
        const auto sum_p = [](float sum, llama_beam & beam) { return sum + beam.p; };
        // 计算概率之和的倒数
        const float inv_sum = 1.0f / std::accumulate(beams.begin(), beams.end(), 0.0f, sum_p);
        // 重新计算每个 beam 的概率
        std::for_each(beams.begin(), beams.end(), [=](llama_beam & beam) { beam.p *= inv_sum; });
    }
    // 假设 beams 非空。使用 llama_beam::operator<() 进行排序。
    size_t top_beam_index() {
        // 返回具有最大值的元素在 beams 中的索引
        return std::max_element(beams.begin(), beams.end()) - beams.begin();
    }
    
    // 为每个可能被回调函数更改的 beam 复制 (p,eob)
    void update_beams_from_beam_views() {
        // 遍历 beams 数组
        for (size_t i = 0 ; i < beams.size() ; ++i) {
            // 将 beam_views 中的 p 和 eob 复制到对应的 beams 中
            beams[i].p = beam_views[i].p;
            beams[i].eob = beam_views[i].eob;
        }
    }
};

// 调用 LLAMA 梁搜索算法，传入上下文、回调函数、回调数据、梁数量、过去步数、预测步数
void llama_beam_search(llama_context * ctx,
                       llama_beam_search_callback_fn_t callback, void * callback_data,
                       size_t n_beams, int n_past, int n_predict) {
    // 断言上下文不为空
    assert(ctx);
    // 记录开始采样时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 创建 LLAMA 梁搜索数据对象
    llama_beam_search_data beam_search_data(ctx, n_beams, n_past, n_predict);

    // 调用梁搜索数据对象的循环方法，传入回调函数和回调数据
    beam_search_data.loop(callback, callback_data);

    // 更新上下文的采样时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    // 增加采样次数
    ctx->n_sample++;
}

//
// 量化
//

// 定义内部量化状态结构体
struct quantize_state_internal {
    const llama_model                 & model;
    const llama_model_quantize_params * params;

    int n_attention_wv    = 0;
    int n_ffn_down        = 0;
    int n_ffn_gate        = 0;
    int n_ffn_up          = 0;
    int i_attention_wv    = 0;
    int i_ffn_down        = 0;
    int i_ffn_gate        = 0;
    int i_ffn_up          = 0;

    int n_k_quantized     = 0;
    int n_fallback        = 0;

    bool has_imatrix      = false;

    // 构造函数，初始化模型和参数
    quantize_state_internal(const llama_model & model, const llama_model_quantize_params * params)
        : model(model)
        , params(params)
        {}
};

// 将 tensor 转换为 float 类型的输出
static void llama_convert_tensor_internal(
    struct ggml_tensor * tensor, std::vector<no_init<float>> & output, std::vector<std::thread> & workers,
    const size_t nelements, const int nthread
) {
    // 如果输出向量大小小于元素数量，调整输出向量大小
    if (output.size() < nelements) {
        output.resize(nelements);
    }
    // 将输出向量转换为 float 类型指针
    float * f32_output = (float *) output.data();

    // 获取 tensor 类型的量化信息
    ggml_type_traits_t qtype;
    if (ggml_is_quantized(tensor->type)) {
        qtype = ggml_internal_get_type_traits(tensor->type);
        // 如果没有浮点数转换函数，抛出运行时错误
        if (qtype.to_float == NULL) {
            throw std::runtime_error(format("type %s unsupported for integer quantization: no dequantization available", ggml_type_name(tensor->type)));
        }
    } else if (tensor->type != GGML_TYPE_F16) {
        // 如果 tensor 类型不是 GGML_TYPE_F16，抛出运行时错误
        throw std::runtime_error(format("cannot dequantize/convert tensor type %s", ggml_type_name(tensor->type)));
    }
    // 如果线程数小于2，则直接处理数据，转换为 float 类型
    if (nthread < 2) {
        // 如果数据类型为 GGML_TYPE_F16，则将数据转换为 float 类型
        if (tensor->type == GGML_TYPE_F16) {
            ggml_fp16_to_fp32_row((ggml_fp16_t *)tensor->data, f32_output, nelements);
        } 
        // 如果数据类型为量化类型，则调用相应的函数将数据转换为 float 类型
        else if (ggml_is_quantized(tensor->type)) {
            qtype.to_float(tensor->data, f32_output, nelements);
        } 
        // 如果数据类型不是上述两种类型，则抛出异常
        else {
            GGML_ASSERT(false); // unreachable
        }
        return;
    }

    // 计算每个数据块的大小
    size_t block_size = tensor->type == GGML_TYPE_F16 ? 1 : (size_t)ggml_blck_size(tensor->type);
    // 计算每个数据块的字节数
    size_t block_size_bytes = ggml_type_size(tensor->type);

    // 断言数据元素个数能够被块大小整除
    GGML_ASSERT(nelements % block_size == 0);
    // 计算数据块的数量
    size_t nblocks = nelements / block_size;
    // 计算每个线程处理的数据块数量
    size_t blocks_per_thread = nblocks / nthread;
    // 计算无法整除线程数的剩余数据块数量
    size_t spare_blocks = nblocks - (blocks_per_thread * nthread); // if blocks aren't divisible by thread count

    // 初始化输入缓冲区偏移量和输出缓冲区偏移量
    size_t in_buff_offs = 0;
    size_t out_buff_offs = 0;

    // 遍历每个线程
    for (int tnum = 0; tnum < nthread; tnum++) {
        // 计算当前线程处理的数据块数量
        size_t thr_blocks = blocks_per_thread + (tnum == nthread - 1 ? spare_blocks : 0); // num blocks for this thread
        // 计算当前线程处理的元素数量
        size_t thr_elems = thr_blocks * block_size; // number of elements for this thread
        // 计算当前线程处理的字节数
        size_t thr_block_bytes = thr_blocks * block_size_bytes; // number of input bytes for this thread

        // 定义 lambda 函数 compute，根据数据类型将输入数据转换为 float 类型
        auto compute = [qtype] (ggml_type typ, uint8_t * inbuf, float * outbuf, int nels) {
            if (typ == GGML_TYPE_F16) {
                ggml_fp16_to_fp32_row((ggml_fp16_t *)inbuf, outbuf, nels);
            } else {
                qtype.to_float(inbuf, outbuf, nels);
            }
        };
        // 将 compute 函数添加到线程池中，并传入相应的参数
        workers.emplace_back(compute, tensor->type, (uint8_t *) tensor->data + in_buff_offs, f32_output + out_buff_offs, thr_elems);
        // 更新输入缓冲区偏移量和输出缓冲区偏移量
        in_buff_offs += thr_block_bytes;
        out_buff_offs += thr_elems;
    }
    // 等待所有线程执行完毕
    for (auto & w : workers) { w.join(); }
    // 清空线程池
    workers.clear();
// 获取 K 量化类型的函数，根据当前量化状态、新类型、张量和浮点类型来确定
static ggml_type get_k_quant_type(quantize_state_internal & qs, ggml_type new_type, const ggml_tensor * tensor, llama_ftype ftype) {
    // 获取张量的名称
    const std::string name = ggml_get_name(tensor);

    // TODO: 避免硬编码张量名称 - 使用 TN_* 常量
    const llm_arch arch = qs.model.arch;
    const auto       tn = LLM_TN(arch);

    // 定义一个 lambda 函数，用于判断是否使用更多位
    auto use_more_bits = [](int i_layer, int num_layers) -> bool {
        return i_layer < num_layers/8 || i_layer >= 7*num_layers/8 || (i_layer - num_layers/8)%3 == 2;
    };
    // 获取专家数量
    const int n_expert = std::max(1, (int)qs.model.hparams.n_expert);
    // 定义一个 lambda 函数，用于获取层信息
    auto layer_info = [n_expert] (int i_layer, int n_layer, const char * name) {
        if (n_expert > 1) {
            // 在 Mixtral-8x7B 的 FFN 中，“专家”并不是连续的，而是偶尔随机分布在模型中。
            // 因此，简单地将 i_ffn_down 除以 n_expert 并不能得到当前层，我们需要解析张量名称。
            n_layer /= n_expert;
            if (sscanf(name, "blk.%d.", &i_layer) != 1) {
                throw std::runtime_error(format("Failed to determine layer for tensor %s", name));
            }
            if (i_layer < 0 || i_layer >= n_layer) {
                throw std::runtime_error(format("Bad layer %d for tensor %s. Must be in [0, %d)", i_layer, name, n_layer));
            }
        }
        return std::make_pair(i_layer, n_layer);
    };

    // 如果张量名称为输出权重张量
    if (name == tn(LLM_TENSOR_OUTPUT, "weight")) {
        int nx = tensor->ne[0];
        // 如果架构为 LLM_ARCH_FALCON 或者 nx 不能被 QK_K 整除
        if (arch == LLM_ARCH_FALCON || nx % QK_K != 0) {
            new_type = GGML_TYPE_Q8_0;
        }
        // 如果浮点类型为 LLAMA_FTYPE_MOSTLY_IQ2_XXS 或 LLAMA_FTYPE_MOSTLY_IQ2_XS
        else if (ftype == LLAMA_FTYPE_MOSTLY_IQ2_XXS || ftype == LLAMA_FTYPE_MOSTLY_IQ2_XS) {
            new_type = GGML_TYPE_Q5_K;
        }
        // 如果新类型不为 GGML_TYPE_Q8_0
        else if (new_type != GGML_TYPE_Q8_0) {
            new_type = GGML_TYPE_Q6_K;
        }
    // 如果文件类型为LLAMA_FTYPE_MOSTLY_IQ2_XXS或LLAMA_FTYPE_MOSTLY_IQ2_XS
    } else if (ftype == LLAMA_FTYPE_MOSTLY_IQ2_XXS || ftype == LLAMA_FTYPE_MOSTLY_IQ2_XS) {
        // 如果文件名包含"attn_v.weight"
        if (name.find("attn_v.weight") != std::string::npos) {
            // 如果模型参数n_gqa()大于等于4或者n_expert大于等于4，则设置新类型为GGML_TYPE_Q4_K，否则为GGML_TYPE_Q2_K
            if (qs.model.hparams.n_gqa() >= 4 || qs.model.hparams.n_expert >= 4) new_type = GGML_TYPE_Q4_K;
            else new_type = GGML_TYPE_Q2_K;
            // 递增i_attention_wv
            ++qs.i_attention_wv;
        }
        // 如果文件名包含"ffn_down"
        else if (name.find("ffn_down") != std::string::npos) {
            // 如果i_ffn_down小于n_ffn_down的八分之一，则设置新类型为GGML_TYPE_Q2_K
            if (qs.i_ffn_down < qs.n_ffn_down/8) new_type = GGML_TYPE_Q2_K;
            // 递增i_ffn_down
            ++qs.i_ffn_down;
        }
        // 如果文件名为"token_embd.weight"，则设置新类型为GGML_TYPE_Q2_K
        else if (name == "token_embd.weight") new_type = GGML_TYPE_Q2_K;
    } else if (name.find("attn_v.weight") != std::string::npos) {
        // 如果文件名中包含"attn_v.weight"
        if      (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) {
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q2_K
            // 根据条件设置新的类型
            new_type = qs.model.hparams.n_gqa() >= 4 ? GGML_TYPE_Q4_K : GGML_TYPE_Q3_K;
        }
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q2_K_S && qs.model.hparams.n_gqa() >= 4) {
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q2_K_S 并且 n_gqa() 大于等于4
            // 设置新的类型为GGML_TYPE_Q4_K
            new_type = GGML_TYPE_Q4_K;
        }
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M) {
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q3_K_M
            // 根据条件设置新的类型
            new_type = qs.i_attention_wv < 2 ? GGML_TYPE_Q5_K : GGML_TYPE_Q4_K;
        }
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) 
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q3_K_L
            // 设置新的类型为GGML_TYPE_Q5_K
            new_type = GGML_TYPE_Q5_K;
        else if ((ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M || ftype == LLAMA_FTYPE_MOSTLY_Q5_K_M) &&
                use_more_bits(qs.i_attention_wv, qs.n_attention_wv)) 
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q4_K_M 或 LLAMA_FTYPE_MOSTLY_Q5_K_M 并且使用更多位
            // 设置新的类型为GGML_TYPE_Q6_K
            new_type = GGML_TYPE_Q6_K;
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S && qs.i_attention_wv < 4) 
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q4_K_S 并且 i_attention_wv 小于4
            // 设置新的类型为GGML_TYPE_Q5_K
            new_type = GGML_TYPE_Q5_K;
        else if (QK_K == 64 && (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S || ftype == LLAMA_FTYPE_MOSTLY_Q3_K_S) &&
                (qs.i_attention_wv < qs.n_attention_wv/8 || qs.i_attention_wv >= 7*qs.n_attention_wv/8)) 
            // 如果 QK_K 等于64 并且文件类型为LLAMA_FTYPE_MOSTLY_Q4_K_S 或 LLAMA_FTYPE_MOSTLY_Q3_K_S
            // 并且 i_attention_wv 在特定范围内
            // 设置新的类型为GGML_TYPE_Q6_K
            new_type = GGML_TYPE_Q6_K;
        if (qs.model.type == MODEL_70B) {
            // 如果模型类型为MODEL_70B
            // 在70B模型中，有8个头共享相同的attn_v权重。因此，attn_v.weight张量比attn_q.weight小8倍。
            // 因此，通过使用更多位对该张量进行量化，可以在几乎不增加模型大小的情况下获得量化精度的显著提升
            if (new_type == GGML_TYPE_Q3_K || new_type == GGML_TYPE_Q4_K) 
                // 如果新类型为GGML_TYPE_Q3_K 或 GGML_TYPE_Q4_K
                // 设置新的类型为GGML_TYPE_Q5_K
                new_type = GGML_TYPE_Q5_K;
        }
        if (qs.model.hparams.n_expert == 8) {
            // 如果模型参数中的专家数量为8
            // 将其提升到Q8_0会交换大约128MB
            // TODO: 探索更好的策略
            new_type = GGML_TYPE_Q8_0;
        }
        // 增加i_attention_wv的值
        ++qs.i_attention_wv;
    // 如果文件名包含"attn_k.weight"
    } else if (name.find("attn_k.weight") != std::string::npos) {
        // 如果模型的专家数量为8
        if (qs.model.hparams.n_expert == 8) {
            // 对于8专家模型，将其提升到Q8_0只会交换大约128MB
            // TODO: 探索更好的策略
            new_type = GGML_TYPE_Q8_0;
        }
        // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q3_K_XS
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_XS) {
            new_type = GGML_TYPE_Q2_K;
        }
    // 如果文件名包含"attn_output.weight"
    } else if (name.find("attn_output.weight") != std::string::npos) {
        // 如果架构不是LLM_ARCH_FALCON
        if (arch != LLM_ARCH_FALCON) {
            // 如果模型的专家数量为8
            if (qs.model.hparams.n_expert == 8) {
                // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q2_K或LLAMA_FTYPE_MOSTLY_Q3_K_XS或...
                if (ftype == LLAMA_FTYPE_MOSTLY_Q2_K   || ftype == LLAMA_FTYPE_MOSTLY_Q3_K_XS ||
                    ftype == LLAMA_FTYPE_MOSTLY_Q3_K_S || ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M ||
                    ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S || ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M) {
                    new_type = GGML_TYPE_Q5_K;
                }
            } else {
                // 根据文件类型设置新类型
                if      (ftype == LLAMA_FTYPE_MOSTLY_Q2_K  ) new_type = GGML_TYPE_Q3_K;
                else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M) new_type = GGML_TYPE_Q4_K;
                else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q5_K;
            }
        } else {
            // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q3_K_L
            if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q4_K;
        }
    }
    // 如果文件名包含"attn_qkv.weight"
    else if (name.find("attn_qkv.weight") != std::string::npos) {
        // 根据文件类型设置新类型
        if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M || ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q4_K;
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M) new_type = GGML_TYPE_Q5_K;
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q5_K_M) new_type = GGML_TYPE_Q6_K;
    }
    // 如果名称中包含"ffn_gate"，则执行以下操作
    else if (name.find("ffn_gate") != std::string::npos) {
        // 获取层信息
        auto info = layer_info(qs.i_ffn_gate, qs.n_ffn_gate, name.c_str());
        int i_layer = info.first, n_layer = info.second;
        // 如果条件成立，设置新类型为GGML_TYPE_Q2_K
        if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_XS && !use_more_bits(i_layer, n_layer)) {
            new_type = GGML_TYPE_Q2_K;
        }
        // 递增qs.i_ffn_gate
        ++qs.i_ffn_gate;
    }
    // 如果名称中包含"ffn_up"，则执行以下操作
    else if (name.find("ffn_up") != std::string::npos) {
        // 获取层信息
        auto info = layer_info(qs.i_ffn_up, qs.n_ffn_up, name.c_str());
        int i_layer = info.first, n_layer = info.second;
        // 如果条件成立，设置新类型为GGML_TYPE_Q2_K
        if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_XS && !use_more_bits(i_layer, n_layer)) {
            new_type = GGML_TYPE_Q2_K;
        }
        // 递增qs.i_ffn_up
        ++qs.i_ffn_up;
    }
    //    if (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) new_type = GGML_TYPE_Q3_K;
    //}
    // IK: let's remove this, else Q2_K is almost the same as Q3_K_S
    //else if (name.find("ffn_gate") != std::string::npos || name.find("ffn_up") != std::string::npos) {
    //    if (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) new_type = GGML_TYPE_Q3_K;
    //}
    // This can be used to reduce the size of the Q5_K_S model.
    // The associated PPL increase is fully in line with the size reduction
    //else {
    //    if (ftype == LLAMA_FTYPE_MOSTLY_Q5_K_S) new_type = GGML_TYPE_Q4_K;
    //}
    // 检查是否需要转换不兼容的张量
    bool convert_incompatible_tensor = false;
    // 如果新类型符合以下条件，则执行以下操作
    if (new_type == GGML_TYPE_Q2_K || new_type == GGML_TYPE_Q3_K || new_type == GGML_TYPE_Q4_K ||
        new_type == GGML_TYPE_Q5_K || new_type == GGML_TYPE_Q6_K ||
        new_type == GGML_TYPE_IQ2_XS || new_type == GGML_TYPE_IQ2_XXS) {
        // 获取张量的行数和列数
        int nx = tensor->ne[0];
        int ny = tensor->ne[1];
        // 如果列数不能被QK_K整除，则输出警告信息并设置convert_incompatible_tensor为true
        if (nx % QK_K != 0) {
            LLAMA_LOG_WARN("\n\n%s : tensor cols %d x %d are not divisible by %d, required for %s", __func__, nx, ny, QK_K, ggml_type_name(new_type));
            convert_incompatible_tensor = true;
        } else {
            // 否则递增qs.n_k_quantized
            ++qs.n_k_quantized;
        }
    }
    // 如果需要转换不兼容的张量类型
    if (convert_incompatible_tensor) {
        // 根据新的类型进行不同的处理
        switch (new_type) {
            // 对于不同的类型，进行不同的转换
            case GGML_TYPE_IQ2_XXS:
            case GGML_TYPE_IQ2_XS:
            case GGML_TYPE_Q2_K: new_type = GGML_TYPE_Q4_0; break;
            case GGML_TYPE_Q3_K: new_type = GGML_TYPE_Q4_1; break;
            case GGML_TYPE_Q4_K: new_type = GGML_TYPE_Q5_0; break;
            case GGML_TYPE_Q5_K: new_type = GGML_TYPE_Q5_1; break;
            case GGML_TYPE_Q6_K: new_type = GGML_TYPE_Q8_0; break;
            // 对于未知的类型，抛出运行时错误
            default: throw std::runtime_error("\nUnsupported tensor size encountered\n");
        }
        // 记录使用回退量化的信息
        LLAMA_LOG_WARN(" - using fallback quantization %s\n", ggml_type_name(new_type));
        // 增加回退量化的计数
        ++qs.n_fallback;
    }

    // 返回新的类型
    return new_type;
// 内部函数，用于量化模型
static void llama_model_quantize_internal(const std::string & fname_inp, const std::string & fname_out, const llama_model_quantize_params * params) {
    // 定义量化后的数据类型
    ggml_type quantized_type;
    // 获取参数中的数据类型
    llama_ftype ftype = params->ftype;

    // 根据不同的数据类型选择对应的量化类型
    switch (params->ftype) {
        case LLAMA_FTYPE_MOSTLY_Q4_0: quantized_type = GGML_TYPE_Q4_0; break;
        case LLAMA_FTYPE_MOSTLY_Q4_1: quantized_type = GGML_TYPE_Q4_1; break;
        case LLAMA_FTYPE_MOSTLY_Q5_0: quantized_type = GGML_TYPE_Q5_0; break;
        case LLAMA_FTYPE_MOSTLY_Q5_1: quantized_type = GGML_TYPE_Q5_1; break;
        case LLAMA_FTYPE_MOSTLY_Q8_0: quantized_type = GGML_TYPE_Q8_0; break;
        case LLAMA_FTYPE_MOSTLY_F16:  quantized_type = GGML_TYPE_F16;  break;
        case LLAMA_FTYPE_ALL_F32:     quantized_type = GGML_TYPE_F32;  break;

        // K-quants
        case LLAMA_FTYPE_MOSTLY_Q2_K_S:
        case LLAMA_FTYPE_MOSTLY_Q2_K:   quantized_type = GGML_TYPE_Q2_K; break;
        case LLAMA_FTYPE_MOSTLY_Q3_K_XS:
        case LLAMA_FTYPE_MOSTLY_Q3_K_S:
        case LLAMA_FTYPE_MOSTLY_Q3_K_M:
        case LLAMA_FTYPE_MOSTLY_Q3_K_L: quantized_type = GGML_TYPE_Q3_K; break;
        case LLAMA_FTYPE_MOSTLY_Q4_K_S:
        case LLAMA_FTYPE_MOSTLY_Q4_K_M: quantized_type = GGML_TYPE_Q4_K; break;
        case LLAMA_FTYPE_MOSTLY_Q5_K_S:
        case LLAMA_FTYPE_MOSTLY_Q5_K_M: quantized_type = GGML_TYPE_Q5_K; break;
        case LLAMA_FTYPE_MOSTLY_Q6_K:   quantized_type = GGML_TYPE_Q6_K; break;
        case LLAMA_FTYPE_MOSTLY_IQ2_XXS:quantized_type = GGML_TYPE_IQ2_XXS; break;
        case LLAMA_FTYPE_MOSTLY_IQ2_XS :quantized_type = GGML_TYPE_IQ2_XS;  break;

        // 默认情况下抛出异常
        default: throw std::runtime_error(format("invalid output file type %d\n", ftype));
    }

    // 获取线程数
    int nthread = params->nthread;

    // 如果线程数小于等于0，则设置为硬件支持的线程数
    if (nthread <= 0) {
        nthread = std::thread::hardware_concurrency();
    }

    // 在 Linux 上 mmap 一致地增加速度，在 Windows 上也会增加速度
    // 热缓存。这可能会导致 macOS 减速，可能与空闲内存有关。
#if defined(__linux__) || defined(_WIN32)
    // 如果是在 Linux 或者 Windows 系统下，使用内存映射
    constexpr bool use_mmap = true;
#else
    // 否则不使用内存映射
    constexpr bool use_mmap = false;
#endif

    // 使用 llama_model_loader 类加载模型文件
    llama_model_loader ml(fname_inp, use_mmap, NULL);
    // 初始化模型映射，不进行预取？
    ml.init_mapping(false);

    // 创建 llama_model 对象
    llama_model model;
    // 加载模型架构
    llm_load_arch(ml, model);
    // 加载模型超参数
    llm_load_hparams(ml, model);

    // 创建 quantize_state_internal 结构体对象
    struct quantize_state_internal qs(model, params);

    // 如果只进行拷贝操作
    if (params->only_copy) {
        ftype = model.ftype;
    }
    // 初始化 imatrix_data 为空指针
    const std::unordered_map<std::string, std::vector<float>> * imatrix_data = nullptr;
    // 如果存在 imatrix 数据
    if (params->imatrix) {
        // 将 imatrix 数据转换为 unordered_map 类型
        imatrix_data = static_cast<const std::unordered_map<std::string, std::vector<float>*>(params->imatrix);
        // 如果 imatrix_data 存在
        if (imatrix_data) {
            // 输出日志信息
            LLAMA_LOG_INFO("================================ Have weights data with %d entries\n",int(imatrix_data->size()));
            // 设置 qs 的 has_imatrix 属性为 true
            qs.has_imatrix = true;
        }
    }

    // 设置对齐大小为 GGUF_DEFAULT_ALIGNMENT
    const size_t align = GGUF_DEFAULT_ALIGNMENT;
    // 初始化 gguf_context 结构体对象
    struct gguf_context * ctx_out = gguf_init_empty();

    // 复制输入文件中的 KV 对
    gguf_set_kv     (ctx_out, ml.ctx_gguf);
    // 设置 "general.quantization_version" 键值对
    gguf_set_val_u32(ctx_out, "general.quantization_version", GGML_QNT_VERSION);
    // 设置 "general.file_type" 键值对
    gguf_set_val_u32(ctx_out, "general.file_type", ftype);

    // 遍历模型中的张量
    for (int i = 0; i < ml.n_tensors; ++i) {
        // 获取第 i 个张量的元数据
        struct ggml_tensor * meta = ml.get_tensor_meta(i);

        // 获取张量的名称
        const std::string name = ggml_get_name(meta);

        // TODO: 避免硬编码张量名称 - 使用 TN_* 常量
        // 如果张量名称包含 "attn_v.weight" 或 "attn_qkv.weight"
        if (name.find("attn_v.weight") != std::string::npos || name.find("attn_qkv.weight") != std::string::npos) {
            // 增加注意力权重张量计数
            ++qs.n_attention_wv;
        }
        // 如果张量名称包含 "ffn_down"
        else if (name.find("ffn_down") != std::string::npos) {
            // 增加下采样前馈网络张量计数
            ++qs.n_ffn_down;
        }
        // 如果张量名称包含 "ffn_gate"
        else if (name.find("ffn_gate") != std::string::npos) {
            // 增加门控前馈网络张量计数
            ++qs.n_ffn_gate;
        }
        // 如果张量名称包含 "ffn_up"
        else if (name.find("ffn_up") != std::string::npos) {
            // 增加上采样前馈网络张量计数
            ++qs.n_ffn_up;
        }
    }
    // 检查模型参数是否符合预期，如果不符合则输出警告信息
    if (qs.n_attention_wv != qs.n_ffn_down || (uint32_t)qs.n_attention_wv != model.hparams.n_layer) {
        LLAMA_LOG_WARN("%s ============ Strange model: n_attention_wv = %d, n_ffn_down = %d, hparams.n_layer = %d\n",
                __func__, qs.n_attention_wv, qs.n_ffn_down, model.hparams.n_layer);
    }

    // 初始化变量用于记录原始数据大小和新数据大小
    size_t total_size_org = 0;
    size_t total_size_new = 0;
    // 初始化一个大小为 16 的整型数组，用于记录数据直方图
    std::vector<int64_t> hist_all(1 << 4, 0);

    // 初始化线程数组和互斥锁
    std::vector<std::thread> workers;
    workers.reserve(nthread);
    std::mutex mutex;

    // 初始化索引变量
    int idx = 0;

    // 初始化三个未初始化的向量，用于存储读取数据、工作数据和浮点数卷积缓冲区
    std::vector<no_init<uint8_t>> read_data;
    std::vector<no_init<uint8_t>> work;
    std::vector<no_init<float>> f32_conv_buf;

    // 遍历模型的张量，获取张量的元数据并添加到输出上下文中
    for (int i = 0; i < ml.n_tensors; ++i) {
        struct ggml_tensor * meta = ml.get_tensor_meta(i);
        gguf_add_tensor(ctx_out, meta);
    }

    // 打开二进制文件输出流，并设置写入错误时立即抛出异常
    std::ofstream fout(fname_out, std::ios::binary);
    fout.exceptions(std::ofstream::failbit); // fail fast on write errors

    // 获取输出上下文的元数据大小
    const size_t meta_size = gguf_get_meta_size(ctx_out);

    // 输出元数据大小信息
    LLAMA_LOG_INFO("%s: meta size = %zu bytes\n", __func__, meta_size);

    // 在文件中写入元数据大小的占位符
    ::zeros(fout, meta_size);

    }

    // 将文件指针移动到文件开头，并写入更新后的元数据
    {
        fout.seekp(0);
        std::vector<uint8_t> data(gguf_get_meta_size(ctx_out));
        gguf_get_meta_data(ctx_out, data.data());
        fout.write((const char *) data.data(), data.size());
    }

    // 关闭文件输出流
    fout.close();

    // 释放输出上下文
    gguf_free(ctx_out);

    // 输出原始数据大小和新数据大小的信息
    LLAMA_LOG_INFO("%s: model size  = %8.2f MB\n", __func__, total_size_org/1024.0/1024.0);
    LLAMA_LOG_INFO("%s: quant size  = %8.2f MB\n", __func__, total_size_new/1024.0/1024.0);

    // 打印所有张量的直方图
    {
        // 初始化一个变量 sum_all 用于存储所有元素的总和
        int64_t sum_all = 0;
        // 遍历 hist_all 数组，计算所有元素的总和
        for (size_t i = 0; i < hist_all.size(); i++) {
            sum_all += hist_all[i];
        }
    
        // 如果总和大于0，则输出日志信息
        if (sum_all > 0) {
            // 输出函数名和提示信息
            LLAMA_LOG_INFO("%s: hist: ", __func__);
            // 遍历 hist_all 数组，输出每个元素除以总和的比例
            for (size_t i = 0; i < hist_all.size(); i++) {
                LLAMA_LOG_INFO("%5.3f ", hist_all[i] / float(sum_all));
            }
            // 输出换行符
            LLAMA_LOG_INFO("\n");
        }
    }
    
    // 如果存在需要回退的情况，则输出警告信息
    if (qs.n_fallback > 0) {
        LLAMA_LOG_WARN("%s: WARNING: %d of %d tensor(s) incompatible with k-quants and required fallback quantization\n",
                __func__, qs.n_fallback, qs.n_k_quantized + qs.n_fallback);
    }
// 静态函数，应用来自文件的 LORA 适配器，接受模型、LORA 文件路径、缩放比例、基础模型路径和线程数作为参数
static int llama_apply_lora_from_file_internal(
    const struct llama_model & model, const char * path_lora, float scale, const char * path_base_model, int n_threads
) {
    // 打印信息，应用 LORA 适配器从指定路径的文件中 - 请稍候...
    LLAMA_LOG_INFO("%s: applying lora adapter from '%s' - please wait ...\n", __func__, path_lora);

    // 记录开始应用 LORA 适配器的时间
    const int64_t t_start_lora_us = ggml_time_us();

    // 打开 LORA 文件
    llama_file fin(path_lora, "rb");

    // 验证文件的魔数和版本
    {
        // 读取文件的魔数
        uint32_t magic = fin.read_u32();
        // 如果魔数不是预期的值，打印错误信息并返回 1
        if (magic != LLAMA_FILE_MAGIC_GGLA) {
            LLAMA_LOG_ERROR("%s: bad file magic\n", __func__);
            return 1;
        }

        // 读取文件的格式版本
        uint32_t format_version = fin.read_u32();
        // 如果版本不受支持，打印错误信息并返回 1
        if (format_version != 1) {
            LLAMA_LOG_ERROR("%s: unsupported file version\n", __func__ );
            return 1;
        }
    }

    // 读取 LORA 参数
    int32_t lora_r = fin.read_u32();
    int32_t lora_alpha = fin.read_u32();
    // 计算缩放值
    float scaling = scale * (float)lora_alpha / (float)lora_r;

    // 打印 LORA 参数信息
    LLAMA_LOG_INFO("%s: r = %d, alpha = %d, scaling = %.2f\n", __func__, lora_r, lora_alpha, scaling);

    // 加载基础模型
    std::unique_ptr<llama_model_loader> ml;
    if (path_base_model) {
        // 打印信息，从指定路径加载基础模型
        LLAMA_LOG_INFO("%s: loading base model from '%s'\n", __func__, path_base_model);
        // 使用内存映射加载基础模型
        ml.reset(new llama_model_loader(path_base_model, /*use_mmap*/ true, /*kv_overrides*/ nullptr));
        // 初始化映射，不进行预取
        ml->init_mapping(/*prefetch*/ false);
    }

    // 定义张量元数据结构
    struct tensor_meta {
        std::string name;
        ggml_type type;
        int32_t ne[2];
        size_t offset;
    };
    // 创建张量名到张量元数据的映射
    std::map<std::string, tensor_meta> tensor_meta_map;

    // 加载所有张量元数据
    // 进入循环，读取文件内容直到文件末尾
    while (true) {
        // 检查是否已经到达文件末尾
        if (fin.tell() == fin.size) {
            // 文件末尾，跳出循环
            break;
        }

        // 读取张量的维度、名称长度和数据类型
        int32_t n_dims;
        int32_t name_len;
        int32_t ftype;

        fin.read_raw(&n_dims, sizeof(n_dims));
        fin.read_raw(&name_len, sizeof(name_len));
        fin.read_raw(&ftype, sizeof(ftype));

        // 检查张量维度是否为1或2
        if (n_dims != 1 && n_dims != 2) {
            // 输出错误信息并返回
            LLAMA_LOG_ERROR("%s: unsupported tensor dimension %d\n", __func__, n_dims);
            return 1;
        }

        // 读取张量的维度信息
        int32_t ne[2] = { 1, 1 };
        for (int i = 0; i < n_dims; ++i) {
            fin.read_raw(&ne[i], sizeof(ne[i]));
        }

        // 读取张量的名称
        std::string name;
        {
            // 确保名称长度不超过最大长度
            GGML_ASSERT(name_len < GGML_MAX_NAME);
            char buf[GGML_MAX_NAME];
            fin.read_raw(buf, name_len);
            name = std::string(buf, name_len);
        }

        // 检查名称是否以".loraA"或".loraB"结尾
        std::string lora_suffix;
        if (name.length() > 6) {
            lora_suffix = name.substr(name.length() - 6);
        }
        if (lora_suffix != ".loraA" && lora_suffix != ".loraB") {
            // 输出错误信息并返回
            LLAMA_LOG_ERROR("%s: error: '%s' is not a lora tensor\n", __func__, name.c_str());
            return 1;
        }

        // 确定张量的数据类型
        ggml_type wtype;
        switch (ftype) {
            case 0: wtype = GGML_TYPE_F32;  break;
            case 1: wtype = GGML_TYPE_F16;  break;
            default:
                    {
                        // 输出错误信息并返回
                        LLAMA_LOG_ERROR("%s: invalid tensor data type '%d'\n", __func__, ftype);
                        return false;
                    }
        }

        // 计算数据偏移量并对齐到32字节边界
        size_t offset = fin.tell();
        offset = (offset + 31) & -32;

        // 跳过张量数据
        fin.seek(offset + ggml_row_size(wtype, ne[0]) * ne[1], SEEK_SET);

        // 将张量的元数据添加到映射中
        tensor_meta_map.emplace(name, tensor_meta{ name, wtype, { ne[0], ne[1] }, offset });
    }

    // 初始化变量
    bool warned = false;
    int n_tensors = 0;

    // 应用
    // 初始化 CPU 后端
    ggml_backend_t backend_cpu = ggml_backend_cpu_init();
    // 如果初始化失败，记录错误信息并返回错误代码
    if (backend_cpu == nullptr) {
        LLAMA_LOG_ERROR("%s: error: failed to initialize cpu backend\n", __func__);
        return 1;
    }
    // 设置 CPU 后端的线程数
    ggml_backend_cpu_set_n_threads(backend_cpu, n_threads);

    // 创建一个存储未初始化的 uint8_t 类型的向量
    std::vector<no_init<uint8_t>> read_buf;
#if 0
        // 如果条件为假，则执行以下代码块
        // TODO: 使用调度程序，以便在 CPU 和 GPU 之间减少数据拷贝
        //ggml_backend_sched_t sched = ggml_backend_sched_new(backends.data(), backends.size(), GGML_DEFAULT_GRAPH_SIZE);

        // 调度计算
        ggml_build_forward_expand(gf, build_graph());
        ggml_backend_sched_init_measure(sched, gf);

        // 重新创建图，因为之前的图已被测量销毁
        ggml_graph_clear(gf);
        ggml_build_forward_expand(gf, build_graph());
        ggml_backend_sched_graph_compute(sched, gf);
        ggml_backend_sched_free(sched);
#endif

        // 释放 LoRa 缓冲区
        ggml_backend_buffer_free(lora_buf);
        // 释放图缓冲区
        ggml_backend_buffer_free(graph_buf);
        // 释放 LoRa 上下文
        ggml_free(lora_ctx);

        // 增加张量计数
        n_tensors++;
        // 每处理 4 个张量打印一个点
        if (n_tensors % 4 == 0) {
            LLAMA_LOG_INFO(".");
        }
    }

    // 释放 CPU 后端
    ggml_backend_free(backend_cpu);

    // 计算 LoRa 时间
    const int64_t t_lora_us = ggml_time_us() - t_start_lora_us;
    // 打印完成信息及时间
    LLAMA_LOG_INFO(" done (%.2f ms)\n", t_lora_us / 1000.0);

    // 返回成功
    return 0;
}

//
// 接口实现
//
// 返回默认模型参数
struct llama_model_params llama_model_default_params() {
    struct llama_model_params result = {
        /*.n_gpu_layers                =*/ 0,
        /*.split_mode                  =*/ LLAMA_SPLIT_LAYER,
        /*.main_gpu                    =*/ 0,
        /*.tensor_split                =*/ nullptr,
        /*.progress_callback           =*/ nullptr,
        /*.progress_callback_user_data =*/ nullptr,
        /*.kv_overrides                =*/ nullptr,
        /*.vocab_only                  =*/ false,
        /*.use_mmap                    =*/ true,
        /*.use_mlock                   =*/ false,
    };

#ifdef GGML_USE_METAL
    // 注意：通常我们有大量的 VRAM，因此默认情况下将所有层都卸载到 GPU
    result.n_gpu_layers = 999;
#endif

    return result;
}

// 返回默认上下文参数
struct llama_context_params llama_context_default_params() {
    // 定义结构体 llama_context_params，并初始化其成员变量
    struct llama_context_params result = {
        /*.seed                        =*/ LLAMA_DEFAULT_SEED, // 设置默认种子值
        /*.n_ctx                       =*/ 512, // 设置上下文大小
        /*.n_batch                     =*/ 512, // 设置批处理大小
        /*.n_threads                   =*/ GGML_DEFAULT_N_THREADS, // 设置线程数，默认值为 GGML_DEFAULT_N_THREADS
        /*.n_threads_batch             =*/ GGML_DEFAULT_N_THREADS, // 设置批处理线程数，默认值为 GGML_DEFAULT_N_THREADS
        /*.rope_scaling_type           =*/ LLAMA_ROPE_SCALING_UNSPECIFIED, // 设置绳索缩放类型
        /*.rope_freq_base              =*/ 0.0f, // 设置绳索频率基数
        /*.rope_freq_scale             =*/ 0.0f, // 设置绳索频率比例
        /*.yarn_ext_factor             =*/ -1.0f, // 设置纱线扩展因子
        /*.yarn_attn_factor            =*/ 1.0f, // 设置纱线注意力因子
        /*.yarn_beta_fast              =*/ 32.0f, // 设置纱线快速 beta 值
        /*.yarn_beta_slow              =*/ 1.0f, // 设置纱线慢速 beta 值
        /*.yarn_orig_ctx               =*/ 0, // 设置原始上下文
        /*.cb_eval                     =*/ nullptr, // 设置回调函数指针为 nullptr
        /*.cb_eval_user_data           =*/ nullptr, // 设置回调函数用户数据指针为 nullptr
        /*.type_k                      =*/ GGML_TYPE_F16, // 设置键的数据类型为 GGML_TYPE_F16
        /*.type_v                      =*/ GGML_TYPE_F16, // 设置值的数据类型为 GGML_TYPE_F16
        /*.mul_mat_q                   =*/ true, // 设置是否乘以矩阵 Q
        /*.logits_all                  =*/ false, // 设置是否输出所有 logits
        /*.embedding                   =*/ false, // 设置是否嵌入
        /*.offload_kqv                 =*/ true, // 设置是否卸载 kqv
    };

    // 返回初始化后的结构体 result
    return result;
// 返回默认的量化参数结构体
struct llama_model_quantize_params llama_model_quantize_default_params() {
    // 初始化默认的量化参数结构体
    struct llama_model_quantize_params result = {
        /*.nthread                     =*/ 0,  // 线程数
        /*.ftype                       =*/ LLAMA_FTYPE_MOSTLY_Q5_1,  // 数据类型
        /*.allow_requantize            =*/ false,  // 是否允许重新量化
        /*.quantize_output_tensor      =*/ true,  // 是否量化输出张量
        /*.only_copy                   =*/ false,  // 是否仅复制
        /*.pure                        =*/ false,  // 是否纯净
        /*.imatrix                     =*/ nullptr,  // 输入矩阵
    };

    return result;
}

// 返回最大设备数
int32_t llama_max_devices(void) {
    return LLAMA_MAX_DEVICES;
}

// 返回是否支持内存映射
bool llama_mmap_supported(void) {
    return llama_mmap::SUPPORTED;
}

// 返回是否支持内存锁定
bool llama_mlock_supported(void) {
    return llama_mlock::SUPPORTED;
}

// 初始化后端
void llama_backend_init(bool numa) {
    // 初始化时间
    ggml_time_init();

    // 需要初始化 f16 表
    {
        struct ggml_init_params params = { 0, NULL, false };
        struct ggml_context * ctx = ggml_init(params);
        ggml_free(ctx);
    }

    // 如果需要 NUMA 支持
    if (numa) {
        ggml_numa_init();
    }

#ifdef GGML_USE_MPI
    ggml_mpi_backend_init();
#endif
}

// 释放后端资源
void llama_backend_free(void) {
#ifdef GGML_USE_MPI
    ggml_mpi_backend_free();
#endif
    ggml_quantize_free();
}

// 返回当前时间的微秒数
int64_t llama_time_us(void) {
    return ggml_time_us();
}

// 从文件加载模型
struct llama_model * llama_load_model_from_file(
                             const char * path_model,
              struct llama_model_params   params) {
    // 初始化时间
    ggml_time_init();

    // 创建新的模型对象
    llama_model * model = new llama_model;

    unsigned cur_percentage = 0;
}
    // 如果进度回调函数为空，则设置进度回调函数为匿名函数
    if (params.progress_callback == NULL) {
        // 将当前进度指针作为用户数据传递给进度回调函数
        params.progress_callback_user_data = &cur_percentage;
        params.progress_callback = [](float progress, void * ctx) {
            // 将上下文转换为 unsigned 指针
            unsigned * cur_percentage_p = (unsigned *) ctx;
            // 计算当前进度百分比
            unsigned percentage = (unsigned) (100 * progress);
            // 当百分比大于当前进度时，更新当前进度并输出日志
            while (percentage > *cur_percentage_p) {
                *cur_percentage_p = percentage;
                LLAMA_LOG_INFO(".");
                // 如果进度达到100%，输出换行符
                if (percentage >= 100) {
                    LLAMA_LOG_INFO("\n");
                }
            }
            return true;
        };
    }

    // 载入模型文件到模型对象中
    int status = llama_model_load(path_model, *model, params);
    // 断言状态小于等于0
    GGML_ASSERT(status <= 0);
    // 处理载入模型的状态
    if (status < 0) {
        // 如果载入失败，根据状态输出相应日志
        if (status == -1) {
            LLAMA_LOG_ERROR("%s: failed to load model\n", __func__);
        } else if (status == -2) {
            LLAMA_LOG_INFO("%s: cancelled model load\n", __func__);
        }
        // 释放模型对象内存并返回空指针
        delete model;
        return nullptr;
    }

    // 返回载入成功的模型对象
    return model;
// 释放模型内存的函数
void llama_free_model(struct llama_model * model) {
    // 删除模型指针
    delete model;
}

// 创建带有模型的新上下文
struct llama_context * llama_new_context_with_model(
                 struct llama_model * model,
        struct llama_context_params   params) {

    // 如果模型为空指针，则返回空指针
    if (!model) {
        return nullptr;
    }

    // 使用模型创建上下文对象
    llama_context * ctx = new llama_context(*model);

    // 获取模型的超参数和上下文的参数引用
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
    cparams.offload_kqv      = params.offload_kqv;

    // 设置上下文参数的默认值
    cparams.n_ctx            = params.n_ctx           == 0    ? hparams.n_ctx_train           : params.n_ctx;
    cparams.rope_freq_base   = params.rope_freq_base  == 0.0f ? hparams.rope_freq_base_train  : params.rope_freq_base;
    cparams.rope_freq_scale  = params.rope_freq_scale == 0.0f ? hparams.rope_freq_scale_train : params.rope_freq_scale;

    // 设置原始上下文数
    cparams.n_yarn_orig_ctx  = params.yarn_orig_ctx    != 0 ? params.yarn_orig_ctx    :
                               hparams.n_yarn_orig_ctx != 0 ? hparams.n_yarn_orig_ctx :
                                                              hparams.n_ctx_train;

    // 设置回调函数和用户数据
    cparams.cb_eval           = params.cb_eval;
    cparams.cb_eval_user_data = params.cb_eval_user_data;

    // 设置绳子缩放类型
    auto rope_scaling_type = params.rope_scaling_type;
    if (rope_scaling_type == LLAMA_ROPE_SCALING_UNSPECIFIED) {
        rope_scaling_type = hparams.rope_scaling_type_train;
    }

    // 如果绳子缩放类型为无，则绳子频率缩放为1.0
    if (rope_scaling_type == LLAMA_ROPE_SCALING_NONE) {
        cparams.rope_freq_scale = 1.0f; // 如果缩放类型为无，则永远不缩放
    }
}
    // 如果yarn_ext_factor小于0，表示未设置，根据rope_scaling_type设置yarn_ext_factor为1.0或0.0
    if (cparams.yarn_ext_factor < 0.0f) { // negative indicates 'not set'
        cparams.yarn_ext_factor = rope_scaling_type == LLAMA_ROPE_SCALING_YARN ? 1.0f : 0.0f;
    }

    // 如果seed为默认值LLAMA_DEFAULT_SEED，则将seed设置为当前时间
    if (params.seed == LLAMA_DEFAULT_SEED) {
        params.seed = time(NULL);
    }

    // 输出日志信息，显示n_ctx、freq_base和freq_scale的值
    LLAMA_LOG_INFO("%s: n_ctx      = %u\n",     __func__, cparams.n_ctx);
    LLAMA_LOG_INFO("%s: freq_base  = %.1f\n",   __func__, cparams.rope_freq_base);
    LLAMA_LOG_INFO("%s: freq_scale = %g\n",     __func__, cparams.rope_freq_scale);

    // 使用seed初始化随机数生成器rng，设置logits_all的值
    ctx->rng = std::mt19937(params.seed);
    ctx->logits_all = params.logits_all;

    // 将params中的type_k和type_v赋值给常量type_k和type_v
    const ggml_type type_k = params.type_k;
    const ggml_type type_v = params.type_v;

    // 断言hparams.n_embd_head_k和hparams.n_embd_head_v能被type_k和type_v的块大小整除
    GGML_ASSERT(hparams.n_embd_head_k % ggml_blck_size(type_k) == 0);
    GGML_ASSERT(hparams.n_embd_head_v % ggml_blck_size(type_v) == 0);

    // 如果不是仅初始化词汇表，则初始化后端
    if (!hparams.vocab_only) {
        // initialize backends
#ifdef GGML_USE_METAL
        // 如果定义了 GGML_USE_METAL，并且模型中有 GPU 层，则初始化 Metal 后端
        if (model->n_gpu_layers > 0) {
            ctx->backend_metal = ggml_backend_metal_init();
            // 如果 Metal 后端初始化失败，则记录错误信息并释放内存，返回空指针
            if (ctx->backend_metal == nullptr) {
                LLAMA_LOG_ERROR("%s: failed to initialize Metal backend\n", __func__);
                llama_free(ctx);
                return nullptr;
            }
            // 将 Metal 后端添加到上下文的后端列表中
            ctx->backends.push_back(ctx->backend_metal);
        }
#elif defined(GGML_USE_CUBLAS)
        // 如果定义了 GGML_USE_CUBLAS，并且模型中有 GPU 层
        if (model->n_gpu_layers > 0) {
            // 当 split_mode 为 LLAMA_SPLIT_NONE 或 LLAMA_SPLIT_ROW 时，只使用主 GPU 后端
            if (model->split_mode == LLAMA_SPLIT_NONE || model->split_mode == LLAMA_SPLIT_ROW) {
                // 初始化 CUDA 主 GPU 后端
                ggml_backend_t backend = ggml_backend_cuda_init(model->main_gpu);
                // 如果 CUDA 主 GPU 后端初始化失败，则记录错误信息并释放内存，返回空指针
                if (backend == nullptr) {
                    LLAMA_LOG_ERROR("%s: failed to initialize CUDA%d backend\n", __func__, model->main_gpu);
                    llama_free(ctx);
                    return nullptr;
                }
                // 将 CUDA 主 GPU 后端添加到上下文的后端列表中
                ctx->backends.push_back(backend);
            } else {
                // LLAMA_SPLIT_LAYER 需要为每个 GPU 初始化一个后端
                for (int device = 0; device < ggml_backend_cuda_get_device_count(); ++device) {
                    // 初始化 CUDA 后端
                    ggml_backend_t backend = ggml_backend_cuda_init(device);
                    // 如果 CUDA 后端初始化失败，则记录错误信息并释放内存，返回空指针
                    if (backend == nullptr) {
                        LLAMA_LOG_ERROR("%s: failed to initialize CUDA%d backend\n", __func__, device);
                        llama_free(ctx);
                        return nullptr;
                    }
                    // 将 CUDA 后端添加到上下文的后端列表中
                    ctx->backends.push_back(backend);
                }
            }
        }
#elif defined(GGML_USE_VULKAN)
        // 如果定义了使用 Vulkan，则执行以下代码块
        if (model->n_gpu_layers > 0) {
            // 如果模型的 GPU 层数大于 0
            // 初始化 Vulkan 后端
            ggml_backend_t backend = ggml_backend_vk_init();
            // 如果初始化失败
            if (backend == nullptr) {
                // 输出错误信息
                LLAMA_LOG_ERROR("%s: failed to initialize Vulkan backend\n", __func__);
                // 释放上下文内存
                llama_free(ctx);
                // 返回空指针
                return nullptr;
            }
            // 将 Vulkan 后端添加到上下文的后端列表中
            ctx->backends.push_back(backend);
        }
#elif defined(GGML_USE_SYCL)
        // 如果定义了使用 SYCL，则执行以下代码块
        if (model->n_gpu_layers > 0) {
            // 如果模型的 GPU 层数大于 0
            // 初始化 SYCL 后端
            ggml_backend_t backend = ggml_backend_sycl_init(model->main_gpu);
            // 如果初始化失败
            if (backend == nullptr) {
                // 输出错误信息
                LLAMA_LOG_ERROR("%s: failed to initialize SYCL%d backend\n", __func__, model->main_gpu);
                // 释放上下文内存
                llama_free(ctx);
                // 返回空指针
                return nullptr;
            }
            // 将 SYCL 后端添加到上下文的后端列表中
            ctx->backends.push_back(backend);
        }
    }

#ifdef GGML_USE_MPI
    // 如果定义了使用 MPI
    // 初始化 MPI 上下文
    ctx->ctx_mpi = ggml_mpi_init();

    // 如果 MPI 的 rank 大于 0
    if (ggml_mpi_rank(ctx->ctx_mpi) > 0) {
        // 进入一个阻塞的评估循环，使用虚拟输入，让 rank=0 驱动进程
        // TODO: 在 #3228 之后需要修复
        GGML_ASSERT(false && "not implemented");
        //const std::vector<llama_token> tmp(ctx->model.hparams.n_ctx, llama_token_bos(ctx));
        //while (!llama_eval(ctx, tmp.data(), tmp.size(), 0, 0)) {};
        // 释放后端资源
        llama_backend_free();
        // 退出程序
        exit(1);
    }
#endif

    // 返回上下文
    return ctx;
}

// 释放上下文内存
void llama_free(struct llama_context * ctx) {
    delete ctx;
}

// 获取模型
const llama_model * llama_get_model(const struct llama_context * ctx) {
    return &ctx->model;
}

// 获取上下文的 n_ctx
uint32_t llama_n_ctx(const struct llama_context * ctx) {
    return ctx->cparams.n_ctx;
}

// 获取上下文的 n_batch
uint32_t llama_n_batch(const struct llama_context * ctx) {
    return ctx->cparams.n_batch;
}

// 获取模型的词汇表类型
enum llama_vocab_type llama_vocab_type(const struct llama_model * model) {
    return model->vocab.type;
}

// 获取模型的词汇表大小
int32_t llama_n_vocab(const struct llama_model * model) {
    return model->vocab.id_to_token.size();
}

// 获取模型的训练上下文大小
int32_t llama_n_ctx_train(const struct llama_model * model) {
    # 返回模型的训练上下文长度
    return model->hparams.n_ctx_train;
}

// 返回模型中嵌入的维度
int32_t llama_n_embd(const struct llama_model * model) {
    return model->hparams.n_embd;
}

// 返回模型中绳子频率比例（训练）
float llama_rope_freq_scale_train(const struct llama_model * model) {
    return model->hparams.rope_freq_scale_train;
}

// 根据键值获取模型元数据的字符串值
int32_t llama_model_meta_val_str(const struct llama_model * model, const char * key, char * buf, size_t buf_size) {
    const auto & it = model->gguf_kv.find(key);
    if (it == model->gguf_kv.end()) {
        if (buf_size > 0) {
            buf[0] = '\0';
        }
        return -1;
    }
    return snprintf(buf, buf_size, "%s", it->second.c_str());
}

// 返回模型元数据的键值对数量
int32_t llama_model_meta_count(const struct llama_model * model) {
    return (int)model->gguf_kv.size();
}

// 根据索引获取模型元数据的键值
int32_t llama_model_meta_key_by_index(const struct llama_model * model, int i, char * buf, size_t buf_size) {
    if (i < 0 || i >= (int)model->gguf_kv.size()) {
        if (buf_size > 0) {
            buf[0] = '\0';
        }
        return -1;
    }
    auto it = model->gguf_kv.begin();
    std::advance(it, i);
    return snprintf(buf, buf_size, "%s", it->first.c_str());
}

// 根据索引获取模型元数据的值
int32_t llama_model_meta_val_str_by_index(const struct llama_model * model, int32_t i, char * buf, size_t buf_size) {
    if (i < 0 || i >= (int)model->gguf_kv.size()) {
        if (buf_size > 0) {
            buf[0] = '\0';
        }
        return -1;
    }
    auto it = model->gguf_kv.begin();
    std::advance(it, i);
    return snprintf(buf, buf_size, "%s", it->second.c_str());
}

// 返回模型描述信息
int32_t llama_model_desc(const struct llama_model * model, char * buf, size_t buf_size) {
    return snprintf(buf, buf_size, "%s %s %s",
            llama_model_arch_name(model->arch).c_str(),
            llama_model_type_name(model->type),
            llama_model_ftype_name(model->ftype).c_str());
}

// 返回模型占用的内存大小
uint64_t llama_model_size(const struct llama_model * model) {
    uint64_t size = 0;
    for (const auto & it : model->tensors_by_name) {
        size += ggml_nbytes(it.second);
    }
    return size;
}
// 计算给定模型的参数数量
uint64_t llama_model_n_params(const struct llama_model * model) {
    // 初始化参数数量为0
    uint64_t nparams = 0;
    // 遍历模型中的张量，计算参数数量
    for (const auto & it : model->tensors_by_name) {
        nparams += ggml_nelements(it.second);
    }
    // 返回参数数量
    return nparams;
}

// 根据名称获取模型中的张量
struct ggml_tensor * llama_get_model_tensor(struct llama_model * model, const char * name) {
    // 在模型的张量中查找指定名称的张量
    auto it = std::find_if(model->tensors_by_name.begin(), model->tensors_by_name.end(),
            [name](const std::pair<std::string, struct ggml_tensor *> & it) {
                return it.first == name;
            });
    // 如果未找到指定名称的张量，则返回空指针
    if (it == model->tensors_by_name.end()) {
        return nullptr;
    }
    // 返回找到的张量
    return it->second;
}

// 对模型进行量化
uint32_t llama_model_quantize(
        const char * fname_inp,
        const char * fname_out,
        const llama_model_quantize_params * params) {
    // 尝试对模型进行内部量化
    try {
        llama_model_quantize_internal(fname_inp, fname_out, params);
        return 0;
    } catch (const std::exception & err) {
        // 捕获异常并记录错误日志
        LLAMA_LOG_ERROR("%s: failed to quantize: %s\n", __func__, err.what());
        return 1;
    }
}

// 从文件应用 LORA
int32_t llama_apply_lora_from_file(struct llama_context * ctx, const char * path_lora, float scale, const char * path_base_model, int32_t n_threads) {
    // 尝试从文件应用 LORA 适配器
    try {
        return llama_apply_lora_from_file_internal(ctx->model, path_lora, scale, path_base_model, n_threads);
    } catch (const std::exception & err) {
        // 捕获异常并记录错误日志
        LLAMA_LOG_ERROR("%s: failed to apply lora adapter: %s\n", __func__, err.what());
        return 1;
    }
}

// 对模型从文件应用 LORA
int32_t llama_model_apply_lora_from_file(const struct llama_model * model, const char * path_lora, float scale, const char * path_base_model, int32_t n_threads) {
    // 尝试对模型从文件应用 LORA 适配器
    try {
        return llama_apply_lora_from_file_internal(*model, path_lora, scale, path_base_model, n_threads);
    } catch (const std::exception & err) {
        // 捕获异常并记录错误日志
        LLAMA_LOG_ERROR("%s: failed to apply lora adapter: %s\n", __func__, err.what());
        return 1;
    }
}
// 初始化 llama_kv_cache_view 结构体
struct llama_kv_cache_view llama_kv_cache_view_init(const struct llama_context * ctx, int32_t n_max_seq) {
    // 初始化 result 结构体
    struct llama_kv_cache_view result = {
        /*.n_cells            = */ 0,  // 设置 n_cells 初始值为 0
        /*.n_max_seq          = */ n_max_seq,  // 设置 n_max_seq 为传入的参数值
        /*.token_count        = */ 0,  // 设置 token_count 初始值为 0
        /*.used_cells         = */ llama_get_kv_cache_used_cells(ctx),  // 获取当前使用的 cells 数量
        /*.max_contiguous     = */ 0,  // 设置 max_contiguous 初始值为 0
        /*.max_contiguous_idx = */ -1,  // 设置 max_contiguous_idx 初始值为 -1
        /*.cells              = */ nullptr,  // 设置 cells 初始值为 nullptr
        /*.cells_sequences    = */ nullptr,  // 设置 cells_sequences 初始值为 nullptr
    };
    return result;  // 返回初始化后的 result 结构体
}

// 释放 llama_kv_cache_view 结构体的内存
void llama_kv_cache_view_free(struct llama_kv_cache_view * view) {
    if (view->cells != nullptr) {  // 检查 cells 是否为 nullptr
        free(view->cells);  // 释放 cells 内存
        view->cells = nullptr;  // 将 cells 指针置为 nullptr
    }
    if (view->cells_sequences != nullptr) {  // 检查 cells_sequences 是否为 nullptr
        free(view->cells_sequences);  // 释放 cells_sequences 内存
        view->cells_sequences = nullptr;  // 将 cells_sequences 指针置为 nullptr
    }
}

// 更新 llama_kv_cache_view 结构体
void llama_kv_cache_view_update(const struct llama_context * ctx, struct llama_kv_cache_view * view) {
    // 如果 n_cells 小于 kv_self.size 或 cells 为 nullptr
    if (uint32_t(view->n_cells) < ctx->kv_self.size || view->cells == nullptr) {
        view->n_cells = int32_t(ctx->kv_self.size);  // 更新 n_cells 为 kv_self.size
        // 重新分配 cells 内存
        void * p = realloc(view->cells, sizeof(struct llama_kv_cache_view_cell) * view->n_cells);
        GGML_ASSERT(p != nullptr && "Failed to alloc kv_cache_view cells");  // 断言内存分配是否成功
        view->cells = (struct llama_kv_cache_view_cell *)p;  // 更新 cells 指针
        // 重新分配 cells_sequences 内存
        p = realloc(view->cells_sequences, sizeof(llama_seq_id) * view->n_max_seq * view->n_cells);
        GGML_ASSERT(p != nullptr && "Failed to alloc kv_cache_view cells sequences");  // 断言内存分配是否成功
        view->cells_sequences = (llama_seq_id *)p;  // 更新 cells_sequences 指针
    }

    // 获取 kv_self.cells
    const std::vector<llama_kv_cell> & kv_cells = ctx->kv_self.cells;
    // 初始化指向 cells 和 cells_sequences 的指针
    llama_kv_cache_view_cell * c_curr = view->cells;
    llama_seq_id * cs_curr = view->cells_sequences;
    int32_t used_cells = 0;  // 初始化 used_cells 为 0
    int32_t token_count = 0;  // 初始化 token_count 为 0
    int32_t curr_contig_idx = -1;  // 初始化 curr_contig_idx 为 -1
    uint32_t max_contig = 0;  // 初始化 max_contig 为 0
    int32_t max_contig_idx = -1;  // 初始化 max_contig_idx 为 -1
}
    // 遍历 kv_self 中的每个元素
    for (int32_t i = 0; i < int32_t(ctx->kv_self.size); i++, c_curr++, cs_curr += view->n_max_seq) {
        // 获取当前元素的序列 ID 数量
        const size_t curr_size = kv_cells[i].seq_id.size();
        // 更新 token_count
        token_count += curr_size;
        // 设置当前元素的位置
        c_curr->pos = kv_cells[i].pos + kv_cells[i].delta;

        // 如果当前元素的序列 ID 数量大于 0
        if (curr_size > 0) {
            // 如果当前连续序列的索引有效且当前元素索引与当前连续序列索引的差值大于最大连续序列长度
            if (curr_contig_idx >= 0 && uint32_t(i - curr_contig_idx) > max_contig) {
                // 更新最大连续序列长度和索引
                max_contig = i - curr_contig_idx;
                max_contig_idx = curr_contig_idx;
            }
            // 重置当前连续序列索引
            curr_contig_idx = -1;
        } else if (curr_contig_idx < 0) {
            // 如果当前连续序列索引无效，则设置为当前元素索引
            curr_contig_idx = i;
        }

        // 遍历当前元素的序列 ID
        int seq_idx = 0;
        for (const llama_seq_id it : kv_cells[i].seq_id) {
            // 如果序列 ID 超过最大序列数，则跳出循环
            if (seq_idx >= view->n_max_seq) {
                break;
            }
            // 将序列 ID 存入 cs_curr
            cs_curr[seq_idx] = it;
            seq_idx++;
        }
        // 如果序列 ID 数量不为 0，则更新 used_cells
        if (seq_idx != 0) {
            used_cells++;
        }
        // 将剩余位置填充为 -1
        for (; seq_idx < view->n_max_seq; seq_idx++) {
            cs_curr[seq_idx] = -1;
        }
    }
    // 如果当前连续序列索引有效且 kv_cells 中剩余元素数量大于最大连续序列长度
    if (curr_contig_idx >= 0 && kv_cells.size() - curr_contig_idx > max_contig) {
        // 更新最大连续序列长度和索引
        max_contig_idx = curr_contig_idx;
        max_contig = kv_cells.size() - curr_contig_idx;
    }
    // 更新 view 中的相关属性
    view->max_contiguous = max_contig;
    view->max_contiguous_idx = max_contig_idx;
    view->token_count = token_count;
    view->used_cells = used_cells;
    // 如果 used_cells 与 kv_self.used 不一致，则输出错误日志
    if (uint32_t(used_cells) != ctx->kv_self.used) {
        LLAMA_LOG_ERROR("%s: used cells mismatch. kv_cache says %d but we calculated %d\n",
            __func__, ctx->kv_self.used, used_cells);
    }
// 获取键值缓存中的令牌数量
int32_t llama_get_kv_cache_token_count(const struct llama_context * ctx) {
    // 初始化结果为0
    int result = 0;

    // 遍历键值缓存中的所有单元格，累加每个单元格中序列 ID 的大小
    for (uint32_t i = 0; i < ctx->kv_self.size; i++) {
        result += ctx->kv_self.cells[i].seq_id.size();
    }

    // 返回结果
    return result;
}

// 获取键值缓存中已使用的单元格数量
int32_t llama_get_kv_cache_used_cells(const struct llama_context * ctx) {
    return ctx->kv_self.used;
}

// 清空键值缓存
void llama_kv_cache_clear(struct llama_context * ctx) {
    // 调用键值缓存中的清空函数
    llama_kv_cache_clear(ctx->kv_self);
}

// 从键值缓存中移除指定序列 ID 的数据
void llama_kv_cache_seq_rm(struct llama_context * ctx, llama_seq_id seq_id, llama_pos p0, llama_pos p1) {
    // 调用键值缓存中的移除函数
    llama_kv_cache_seq_rm(ctx->kv_self, seq_id, p0, p1);
}

// 将一个序列 ID 的数据复制到另一个序列 ID
void llama_kv_cache_seq_cp(struct llama_context * ctx, llama_seq_id seq_id_src, llama_seq_id seq_id_dst, llama_pos p0, llama_pos p1) {
    // 如果源序列 ID 和目标序列 ID 相同，则直接返回
    if (seq_id_src == seq_id_dst) {
        return;
    }
    // 调用键值缓存中的复制函数
    llama_kv_cache_seq_cp(ctx->kv_self, seq_id_src, seq_id_dst, p0, p1);
}

// 保留指定序列 ID 的数据，清除其他数据
void llama_kv_cache_seq_keep(struct llama_context * ctx, llama_seq_id seq_id) {
    // 调用键值缓存中的保留函数
    llama_kv_cache_seq_keep(ctx->kv_self, seq_id);
}

// 将指定序列 ID 的数据在指定范围内移动指定偏移量
void llama_kv_cache_seq_shift(struct llama_context * ctx, llama_seq_id seq_id, llama_pos p0, llama_pos p1, llama_pos delta) {
    // 如果偏移量为0，则直接返回
    if (delta == 0) {
        return;
    }

    // 调用键值缓存中的移动函数
    llama_kv_cache_seq_shift(ctx->kv_self, seq_id, p0, p1, delta);
}

// 将指定序列 ID 的数据在指定范围内分割为指定份数
void llama_kv_cache_seq_div(struct llama_context * ctx, llama_seq_id seq_id, llama_pos p0, llama_pos p1, int d) {
    // 如果份数为1，则直接返回
    if (d == 1) {
        return;
    }

    // 调用键值缓存中的分割函数
    llama_kv_cache_seq_div(ctx->kv_self, seq_id, p0, p1, d);
}

// 返回状态的最大大小
size_t llama_get_state_size(const struct llama_context * ctx) {
    // 为序列化状态的 RNG 预留足够的内存空间
    // 参考：std::mt19937(1337) 序列化后占用 6701 字节
    const size_t s_rng_size        = sizeof(size_t);
    const size_t s_rng             = LLAMA_MAX_RNG_STATE;
    const size_t s_logits_size     = sizeof(size_t);
}
    // 计算 logits 的内存占用大小，假设为最坏情况，即所有元素都被序列化
    const size_t s_logits          = ctx->logits.capacity() * sizeof(float);
    // 计算 embedding_size 的内存占用大小
    const size_t s_embedding_size  = sizeof(size_t);
    // 计算 embedding 的内存占用大小
    const size_t s_embedding       = ctx->embedding.size() * sizeof(float);
    // 计算 kv_size 的内存占用大小
    const size_t s_kv_size         = sizeof(size_t);
    // 计算 kv_ntok 的内存占用大小
    const size_t s_kv_ntok         = sizeof(int);
    // 计算 kv 的内存占用大小
    const size_t s_kv              = ctx->kv_self.total_size();

    // 计算总内存占用大小
    const size_t s_total = (
        + s_rng_size
        + s_rng
        + s_logits_size
        + s_logits
        + s_embedding_size
        + s_embedding
        + s_kv_size
        + s_kv_ntok
        + s_kv
    );

    // 返回总内存占用大小
    return s_total;
// 结构体 llama_data_context 用于定义数据上下文的接口
struct llama_data_context {
    // 纯虚函数，用于写入数据
    virtual void write(const void * src, size_t size) = 0;
    // 纯虚函数，用于获取已写入数据的大小
    virtual size_t get_size_written() = 0;
    // 默认析构函数
    virtual ~llama_data_context() = default;
};

// 结构体 llama_data_buffer_context 继承自 llama_data_context，用于在缓冲区中写入数据
struct llama_data_buffer_context : llama_data_context {
    // 指向缓冲区的指针
    uint8_t * ptr;
    // 已写入数据的大小
    size_t size_written = 0;

    // 构造函数，初始化指针
    llama_data_buffer_context(uint8_t * p) : ptr(p) {}

    // 实现写入数据的方法
    void write(const void * src, size_t size) override {
        // 将数据从 src 复制到缓冲区中
        memcpy(ptr, src, size);
        // 更新指针和已写入数据的大小
        ptr += size;
        size_written += size;
    }

    // 实现获取已写入数据大小的方法
    size_t get_size_written() override {
        return size_written;
    }
};

// 结构体 llama_data_file_context 继承自 llama_data_context，用于在文件中写入数据
struct llama_data_file_context : llama_data_context {
    // 指向文件的指针
    llama_file * file;
    // 已写入数据的大小
    size_t size_written = 0;

    // 构造函数，初始化文件指针
    llama_data_file_context(llama_file * f) : file(f) {}

    // 实现写入数据的方法
    void write(const void * src, size_t size) override {
        // 调用文件对象的写入方法写入数据
        file->write_raw(src, size);
        // 更新已写入数据的大小
        size_written += size;
    }

    // 实现获取已写入数据大小的方法
    size_t get_size_written() override {
        return size_written;
    }
};

/** 将状态数据复制到缓冲区或文件，取决于传入的上下文
 *
 * 文件上下文：
 * llama_file file("/path", "wb");
 * llama_data_file_context data_ctx(&file);
 * llama_copy_state_data(ctx, &data_ctx);
 *
 * 缓冲区上下文：
 * std::vector<uint8_t> buf(max_size, 0);
 * llama_data_buffer_context data_ctx(&buf.data());
 * llama_copy_state_data(ctx, &data_ctx);
 *
*/
static void llama_copy_state_data_internal(struct llama_context * ctx, llama_data_context * data_ctx) {
    // 复制随机数生成器状态数据
    {
        // 创建字符串流对象，将随机数生成器状态数据写入其中
        std::ostringstream rng_ss;
        rng_ss << ctx->rng;

        // 获取字符串流中的字符串和大小
        const std::string & rng_str = rng_ss.str();
        const size_t        rng_size = rng_str.size();

        // 断言随机数生成器状态数据大小不超过最大值
        GGML_ASSERT(rng_size <= LLAMA_MAX_RNG_STATE);

        // 写入随机数生成器状态数据大小和数据内容
        data_ctx->write(&rng_size,      sizeof(rng_size));
        data_ctx->write(rng_str.data(), rng_size);
    }

    // 复制逻辑数据
    {
        // 获取logits的大小
        const size_t logits_size = ctx->logits.size();
    
        // 将logits的大小写入数据流
        data_ctx->write(&logits_size, sizeof(logits_size));
    
        // 如果logits的大小不为0，则将logits数据写入数据流
        if (logits_size) {
            data_ctx->write(ctx->logits.data(), logits_size * sizeof(float));
        }
    }
    
    // 复制嵌入
    {
        // 获取嵌入的大小
        const size_t embedding_size = ctx->embedding.size();
    
        // 将嵌入的大小写入数据流
        data_ctx->write(&embedding_size, sizeof(embedding_size));
    
        // 如果嵌入的大小不为0，则将嵌入数据写入数据流
        if (embedding_size) {
            data_ctx->write(ctx->embedding.data(), embedding_size * sizeof(float));
        }
    }
    
    // 复制kv缓存
    }
}

// 从指定源地址复制状态数据到目标地址
size_t llama_copy_state_data(struct llama_context * ctx, uint8_t * dst) {
    // 创建数据缓冲区上下文对象
    llama_data_buffer_context data_ctx(dst);
    // 调用内部函数复制状态数据
    llama_copy_state_data_internal(ctx, &data_ctx);

    // 返回写入数据的大小
    return data_ctx.get_size_written();
}

// 设置状态数据，从指定源地址读取
size_t llama_set_state_data(struct llama_context * ctx, uint8_t * src) {
    // 指向输入源地址的指针
    uint8_t * inp = src;

    // 设置随机数生成器(rng)
    {
        size_t rng_size;
        // 从输入源地址中复制随机数生成器大小
        memcpy(&rng_size, inp, sizeof(rng_size)); inp += sizeof(rng_size);

        // 断言随机数生成器大小不超过最大值
        GGML_ASSERT(rng_size <= LLAMA_MAX_RNG_STATE);

        // 从输入源地址中读取随机数生成器数据
        std::string rng_str((char *)inp, rng_size); inp += rng_size;

        // 创建字符串输入流对象
        std::istringstream rng_ss(rng_str);
        // 从输入流中读取数据到随机数生成器对象
        rng_ss >> ctx->rng;

        // 断言读取操作没有失败
        GGML_ASSERT(!rng_ss.fail());
    }

    // 设置logits
    {
        size_t logits_size;
        // 从输入源地址中复制logits大小
        memcpy(&logits_size, inp, sizeof(logits_size)); inp += sizeof(logits_size);

        // 断言logits容量大于等于大小
        GGML_ASSERT(ctx->logits.capacity() >= logits_size);

        if (logits_size) {
            // 调整logits大小
            ctx->logits.resize(logits_size);
            // 从输入源地址中复制logits数据
            memcpy(ctx->logits.data(), inp, logits_size * sizeof(float));
            inp += logits_size * sizeof(float);
        }
    }

    // 设置embeddings
    {
        size_t embedding_size;
        // 从输入源地址中复制embeddings大小
        memcpy(&embedding_size, inp, sizeof(embedding_size)); inp += sizeof(embedding_size);

        // 断言embeddings容量等于大小
        GGML_ASSERT(ctx->embedding.capacity() == embedding_size);

        if (embedding_size) {
            // 从输入源地址中复制embeddings数据
            memcpy(ctx->embedding.data(), inp, embedding_size * sizeof(float));
            inp += embedding_size * sizeof(float);
        }
    }

    // 设置kv缓存
    }

    // 计算读取的字节数
    const size_t nread    = inp - src;
    // 获取状态数据的最大大小
    const size_t max_size = llama_get_state_size(ctx);

    // 断言读取的字节数不超过最大大小
    GGML_ASSERT(nread <= max_size);

    // 返回读取的字节数
    return nread;
}

// 内部函数，加载会话文件
static bool llama_load_session_file_internal(struct llama_context * ctx, const char * path_session, llama_token * tokens_out, size_t n_token_capacity, size_t * n_token_count_out) {
    // 打开会话文件
    llama_file file(path_session, "rb");

    // 进行一些基本的检查
    {
        // 读取文件中的魔数和版本号
        const uint32_t magic   = file.read_u32();
        const uint32_t version = file.read_u32();

        // 检查魔数和版本号是否匹配
        if (magic != LLAMA_SESSION_MAGIC || version != LLAMA_SESSION_VERSION) {
            // 输出错误信息并返回 false
            LLAMA_LOG_ERROR("%s : unknown (magic, version) for session file: %08x, %08x\n", __func__, magic, version);
            return false;
        }

        // 读取会话超参数
        llama_hparams session_hparams;
        file.read_raw(&session_hparams, sizeof(llama_hparams));

        // 检查会话超参数是否与模型超参数匹配
        if (session_hparams != ctx->model.hparams) {
            // 输出信息并返回 false
            LLAMA_LOG_INFO("%s : model hparams didn't match from session file!\n", __func__);
            return false;
        }
    }

    // 加载提示信息
    {
        // 读取令牌数量
        const uint32_t n_token_count = file.read_u32();

        // 检查令牌数量是否超出容量
        if (n_token_count > n_token_capacity) {
            // 输出错误信息并返回 false
            LLAMA_LOG_ERROR("%s : token count in session file exceeded capacity! %u > %zu\n", __func__, n_token_count, n_token_capacity);
            return false;
        }

        // 读取令牌数据并更新令牌数量
        file.read_raw(tokens_out, sizeof(llama_token) * n_token_count);
        *n_token_count_out = n_token_count;
    }

    // 恢复上下文状态
    {
        // 计算当前状态数据大小和最大状态数据大小
        const size_t n_state_size_cur = file.size - file.tell();
        const size_t n_state_size_max = llama_get_state_size(ctx);

        // 检查状态数据大小是否超出最大值
        if (n_state_size_cur > n_state_size_max) {
            // 输出错误信息并返回 false
            LLAMA_LOG_ERROR("%s : the state size in session file is too big! max %zu, got %zu\n", __func__, n_state_size_max, n_state_size_cur);
            return false;
        }

        // 读取状态数据并设置到上下文中
        std::vector<uint8_t> state_data(n_state_size_max);
        file.read_raw(state_data.data(), n_state_size_cur);

        llama_set_state_data(ctx, state_data.data());
    }

    // 返回 true 表示成功
    return true;
}
// 结束 llama_load_session_file 函数定义
bool llama_load_session_file(struct llama_context * ctx, const char * path_session, llama_token * tokens_out, size_t n_token_capacity, size_t * n_token_count_out) {
    try {
        // 调用内部函数 llama_load_session_file_internal 加载会话文件
        return llama_load_session_file_internal(ctx, path_session, tokens_out, n_token_capacity, n_token_count_out);
    } catch (const std::exception & err) {
        // 捕获异常并记录错误日志
        LLAMA_LOG_ERROR("error loading session file: %s\n", err.what());
        return false;
    }
}

// 开始 llama_save_session_file 函数定义
bool llama_save_session_file(struct llama_context * ctx, const char * path_session, const llama_token * tokens, size_t n_token_count) {
    // 创建 llama_file 对象，以写入二进制文件
    llama_file file(path_session, "wb");

    // 写入会话文件的魔数和版本号
    file.write_u32(LLAMA_SESSION_MAGIC);
    file.write_u32(LLAMA_SESSION_VERSION);

    // 写入模型超参数
    file.write_raw(&ctx->model.hparams, sizeof(llama_hparams));

    // 写入提示信息的数量和内容
    file.write_u32((uint32_t) n_token_count);
    file.write_raw(tokens, sizeof(llama_token) * n_token_count);

    // 使用流保存方式保存上下文状态
    llama_data_file_context data_ctx(&file);
    llama_copy_state_data_internal(ctx, &data_ctx);

    return true;
}

// 开始 llama_eval 函数定义
int llama_eval(
        struct llama_context * ctx,
                 llama_token * tokens,
                     int32_t   n_tokens,
                     int32_t   n_past) {
    // 从键值缓存中移除过去的键值对
    llama_kv_cache_seq_rm(ctx->kv_self, -1, n_past, -1);

    // 解码 tokens，并返回结果
    const int ret = llama_decode_internal(*ctx, llama_batch_get_one(tokens, n_tokens, n_past, 0));
    if (ret < 0) {
        // 如果解码失败，记录错误日志
        LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
    }

    return ret;
}

// 开始 llama_eval_embd 函数定义
int llama_eval_embd(
            struct llama_context * ctx,
                           float * embd,
                         int32_t   n_tokens,
                         int32_t   n_past) {
    // 从键值缓存中移除过去的键值对
    llama_kv_cache_seq_rm(ctx->kv_self, -1, n_past, -1);

    // 创建包含嵌入向量的批次对象
    llama_batch batch = { n_tokens, nullptr, embd, nullptr, nullptr, nullptr, nullptr, n_past, 1, 0, };

    // 解码 tokens，并返回结果
    const int ret = llama_decode_internal(*ctx, batch);
    # 如果返回值小于0，表示解码失败
    if (ret < 0) {
        # 记录错误日志，包含函数名和返回值
        LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
    }

    # 返回解码结果
    return ret;
// 设置 LLAMA 上下文中的线程数和批处理线程数
void llama_set_n_threads(struct llama_context * ctx, uint32_t n_threads, uint32_t n_threads_batch) {
    // 设置上下文参数中的线程数
    ctx->cparams.n_threads       = n_threads;
    // 设置上下文参数中的批处理线程数
    ctx->cparams.n_threads_batch = n_threads_batch;
}

// 获取一个 LLAMA 批次
struct llama_batch llama_batch_get_one(
             llama_token * tokens,
                 int32_t   n_tokens,
               llama_pos   pos_0,
            llama_seq_id   seq_id) {
    // 返回一个包含指定信息的批次结构体
    return {
        /*n_tokens       =*/ n_tokens,
        /*tokens         =*/ tokens,
        /*embd           =*/ nullptr,
        /*pos            =*/ nullptr,
        /*n_seq_id       =*/ nullptr,
        /*seq_id         =*/ nullptr,
        /*logits         =*/ nullptr,
        /*all_pos_0      =*/ pos_0,
        /*all_pos_1      =*/ 1,
        /*all_seq_id     =*/ seq_id,
    };
}

// 初始化 LLAMA 批次
struct llama_batch llama_batch_init(int32_t n_tokens, int32_t embd, int32_t n_seq_max) {
    // 创建一个初始值为零的批次结构体
    llama_batch batch = { 0, nullptr, nullptr, nullptr, nullptr, nullptr, nullptr, 0, 0, 0, };

    // 根据是否有嵌入向量，分配内存
    if (embd) {
        batch.embd = (float *) malloc(sizeof(float) * n_tokens * embd);
    } else {
        batch.token = (llama_token *) malloc(sizeof(llama_token) * n_tokens);
    }

    // 分配其他内存空间
    batch.pos      = (llama_pos *)     malloc(sizeof(llama_pos)      * n_tokens);
    batch.n_seq_id = (int32_t *)       malloc(sizeof(int32_t)        * n_tokens);
    batch.seq_id   = (llama_seq_id **) malloc(sizeof(llama_seq_id *) * n_tokens);
    for (int i = 0; i < n_tokens; ++i) {
        batch.seq_id[i] = (llama_seq_id *) malloc(sizeof(llama_seq_id) * n_seq_max);
    }
    batch.logits   = (int8_t *)        malloc(sizeof(int8_t)         * n_tokens);

    return batch;
}

// 释放 LLAMA 批次占用的内存
void llama_batch_free(struct llama_batch batch) {
    if (batch.token)    free(batch.token);
    if (batch.embd)     free(batch.embd);
    if (batch.pos)      free(batch.pos);
    if (batch.n_seq_id) free(batch.n_seq_id);
    # 检查批处理中的序列ID是否存在
    if (batch.seq_id) {
        # 遍历批处理中的序列ID，并释放内存
        for (int i = 0; i < batch.n_tokens; ++i) {
            free(batch.seq_id[i]);
        }
        # 释放存储序列ID的内存
        free(batch.seq_id);
    }
    # 检查批处理中的logits是否存在，并释放内存
    if (batch.logits)   free(batch.logits);
}

// 解码函数，调用 llama_decode_internal 函数进行解码操作，并返回结果
int32_t llama_decode(
        struct llama_context * ctx,
          struct llama_batch   batch) {
    const int ret = llama_decode_internal(*ctx, batch);
    // 如果解码失败，记录错误信息
    if (ret < 0) {
        LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
    }

    return ret;
}

// 获取模型的 logits 数组指针
float * llama_get_logits(struct llama_context * ctx) {
    return ctx->logits.data();
}

// 获取模型的第 i 个 logits 数组指针
float * llama_get_logits_ith(struct llama_context * ctx, int32_t i) {
    // 确保第 i 个 logits 有效
    assert(ctx->logits_valid.at(i));
    return ctx->logits.data() + i*ctx->model.hparams.n_vocab;
}

// 获取模型的 embedding 数组指针
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

// 获取特殊 token BOS 的 ID
llama_token llama_token_bos(const struct llama_model * model) {
    return model->vocab.special_bos_id;
}

// 获取特殊 token EOS 的 ID
llama_token llama_token_eos(const struct llama_model * model) {
    return model->vocab.special_eos_id;
}

// 获取特殊 token NL 的 ID
llama_token llama_token_nl(const struct llama_model * model) {
    return model->vocab.linefeed_id;
}

// 获取特殊 token ADD_BOS 的 ID
int32_t llama_add_bos_token(const struct llama_model * model) {
    return model->vocab.special_add_bos;
}

// 获取特殊 token ADD_EOS 的 ID
int32_t llama_add_eos_token(const struct llama_model * model) {
    return model->vocab.special_add_eos;
}

// 获取特殊 token PREFIX 的 ID
llama_token llama_token_prefix(const struct llama_model * model) {
    return model->vocab.special_prefix_id;
}

// 获取特殊 token MIDDLE 的 ID
llama_token llama_token_middle(const struct llama_model * model) {
    return model->vocab.special_middle_id;
}

// 获取特殊 token SUFFIX 的 ID
llama_token llama_token_suffix(const struct llama_model * model) {
    return model->vocab.special_suffix_id;
}
// 返回特殊结束标记的 LLAMA 令牌
llama_token llama_token_eot(const struct llama_model * model) {
    return model->vocab.special_eot_id;
}

// 对文本进行 LLAMA 令牌化
int32_t llama_tokenize(
    const struct llama_model * model,
                  const char * text,
                     int32_t   text_len,
                 llama_token * tokens,
                     int32_t   n_max_tokens,
                        bool   add_bos,
                        bool   special) {
    // 调用内部 LLAMA 令牌化函数
    auto res = llama_tokenize_internal(model->vocab, std::string(text, text_len), add_bos, special);

    // 如果令牌数量超过最大限制，则返回错误
    if (n_max_tokens < (int) res.size()) {
        // LLAMA_LOG_ERROR("%s: too many tokens\n", __func__);
        return -((int) res.size());
    }

    // 将令牌复制到输出数组中
    for (size_t i = 0; i < res.size(); i++) {
        tokens[i] = res[i];
    }

    return res.size();
}

// 解码文本
static std::string llama_decode_text(const std::string & text) {
    std::string decoded_text;
    // 将 UTF-8 编码的文本转换为 Unicode 序列
    auto unicode_sequences = codepoints_from_utf8(text);
    // 将 Unicode 序列转换为字节对应的 BPE 编码
    for (auto& unicode_sequence : unicode_sequences) {
        decoded_text += unicode_to_bytes_bpe(codepoint_to_utf8(unicode_sequence));
    }

    return decoded_text;
}

// 将 LLAMA 令牌转换为片段
// 不会向缓冲区写入空终止符
int32_t llama_token_to_piece(const struct llama_model * model, llama_token token, char * buf, int32_t length) {
    // 返回 0
    return 0;
}

// 获取 LLAMA 上下文的时间统计信息
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

// 打印 LLAMA 上下文的时间统计信息
void llama_print_timings(struct llama_context * ctx) {
    const llama_timings timings = llama_get_timings(ctx);
}
    # 输出一个空行
    LLAMA_LOG_INFO("\n");
    # 输出加载时间信息
    LLAMA_LOG_INFO("%s:        load time = %10.2f ms\n", __func__, timings.t_load_ms);
    # 输出采样时间信息
    LLAMA_LOG_INFO("%s:      sample time = %10.2f ms / %5d runs   (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, timings.t_sample_ms, timings.n_sample, timings.t_sample_ms / timings.n_sample, 1e3 / timings.t_sample_ms * timings.n_sample);
    # 输出提示评估时间信息
    LLAMA_LOG_INFO("%s: prompt eval time = %10.2f ms / %5d tokens (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, timings.t_p_eval_ms, timings.n_p_eval, timings.t_p_eval_ms / timings.n_p_eval, 1e3 / timings.t_p_eval_ms * timings.n_p_eval);
    # 输出评估时间信息
    LLAMA_LOG_INFO("%s:        eval time = %10.2f ms / %5d runs   (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, timings.t_eval_ms, timings.n_eval, timings.t_eval_ms / timings.n_eval, 1e3 / timings.t_eval_ms * timings.n_eval);
    # 输出总时间信息
    LLAMA_LOG_INFO("%s:       total time = %10.2f ms / %5d tokens\n", __func__, (timings.t_end_ms - timings.t_start_ms), (timings.n_p_eval + timings.n_eval));
}

// 重置 LLAMA 上下文中的计时器
void llama_reset_timings(struct llama_context * ctx) {
    // 记录开始时间
    ctx->t_start_us = ggml_time_us();
    // 初始化采样时间和次数
    ctx->t_sample_us = ctx->n_sample = 0;
    // 初始化评估时间和次数
    ctx->t_eval_us   = ctx->n_eval   = 0;
    // 初始化部分评估时间和次数
    ctx->t_p_eval_us = ctx->n_p_eval = 0;
}

// 打印系统信息并返回
const char * llama_print_system_info(void) {
    // 静态字符串对象
    static std::string s;

    // 清空字符串
    s  = "";
    // 拼接 AVX 信息
    s += "AVX = "         + std::to_string(ggml_cpu_has_avx())         + " | ";
    // 拼接 AVX_VNNI 信息
    s += "AVX_VNNI = "    + std::to_string(ggml_cpu_has_avx_vnni())    + " | ";
    // 拼接 AVX2 信息
    s += "AVX2 = "        + std::to_string(ggml_cpu_has_avx2())        + " | ";
    // 拼接 AVX512 信息
    s += "AVX512 = "      + std::to_string(ggml_cpu_has_avx512())      + " | ";
    // 拼接 AVX512_VBMI 信息
    s += "AVX512_VBMI = " + std::to_string(ggml_cpu_has_avx512_vbmi()) + " | ";
    // 拼接 AVX512_VNNI 信息
    s += "AVX512_VNNI = " + std::to_string(ggml_cpu_has_avx512_vnni()) + " | ";
    // 拼接 FMA 信息
    s += "FMA = "         + std::to_string(ggml_cpu_has_fma())         + " | ";
    // 拼接 NEON 信息
    s += "NEON = "        + std::to_string(ggml_cpu_has_neon())        + " | ";
    // 拼接 ARM_FMA 信息
    s += "ARM_FMA = "     + std::to_string(ggml_cpu_has_arm_fma())     + " | ";
    // 拼接 F16C 信息
    s += "F16C = "        + std::to_string(ggml_cpu_has_f16c())        + " | ";
    // 拼接 FP16_VA 信息
    s += "FP16_VA = "     + std::to_string(ggml_cpu_has_fp16_va())     + " | ";
    // 拼接 WASM_SIMD 信息
    s += "WASM_SIMD = "   + std::to_string(ggml_cpu_has_wasm_simd())   + " | ";
    // 拼接 BLAS 信息
    s += "BLAS = "        + std::to_string(ggml_cpu_has_blas())        + " | ";
    // 拼接 SSE3 信息
    s += "SSE3 = "        + std::to_string(ggml_cpu_has_sse3())        + " | ";
    // 拼接 SSSE3 信息
    s += "SSSE3 = "       + std::to_string(ggml_cpu_has_ssse3())       + " | ";
    // 拼接 VSX 信息
    s += "VSX = "         + std::to_string(ggml_cpu_has_vsx())         + " | ";

    // 返回字符串的 C 风格指针
    return s.c_str();
}

// 将 LLAMA 上下文中的计时信息以 YAML 格式输出到文件流中
void llama_dump_timing_info_yaml(FILE * stream, const llama_context * ctx) {
    // 输出换行符
    fprintf(stream, "\n");
    // 输出分隔线
    fprintf(stream, "###########\n");
    // 输出标题
    fprintf(stream, "# Timings #\n");
    // 输出分隔线
    fprintf(stream, "###########\n");
    // 输出换行符
    fprintf(stream, "\n");

    // 输出评估时间信息
    fprintf(stream, "mst_eval: %.2f  # ms / token during generation\n",
            1.0e-3 * ctx->t_eval_us / ctx->n_eval);
}
    # 输出每个 token 处理时的平均时间，单位为毫秒
    fprintf(stream, "mst_p_eval: %.2f  # ms / token during prompt processing\n",
            1.0e-3 * ctx->t_p_eval_us / ctx->n_p_eval);
    # 输出每个 token 采样时的平均时间，单位为毫秒
    fprintf(stream, "mst_sample: %.2f  # ms / token during sampling\n",
            1.0e-3 * ctx->t_sample_us / ctx->n_sample);
    # 输出生成的 token 数量（不包括第一个 token）
    fprintf(stream, "n_eval: %d  # number of tokens generated (excluding the first one)\n", ctx->n_eval);
    # 输出在开始时以批处理方式处理的 token 数量
    fprintf(stream, "n_p_eval: %d  # number of tokens processed in batches at the beginning\n", ctx->n_p_eval);
    # 输出采样的 token 数量
    fprintf(stream, "n_sample: %d  # number of sampled tokens\n", ctx->n_sample);
    # 输出生成 token 总共花费的微秒数
    fprintf(stream, "t_eval_us: %" PRId64 "  # total microseconds spent generating tokens\n", ctx->t_eval_us);
    # 输出加载模型总共花费的微秒数
    fprintf(stream, "t_load_us: %" PRId64 "  # total microseconds spent loading the model\n", ctx->t_load_us);
    # 输出处理提示时总共花费的微秒数
    fprintf(stream, "t_p_eval_us: %" PRId64 "  # total microseconds spent prompt processing\n", ctx->t_p_eval_us);
    # 输出采样总共花费的微秒数
    fprintf(stream, "t_sample_us: %" PRId64 "  # total microseconds spent sampling\n", ctx->t_sample_us);
    # 输出生成 token 的速度，单位为每秒生成的 token 数量
    fprintf(stream, "ts_eval: %.2f  # tokens / second during generation\n",
            1.0e6 * ctx->n_eval / ctx->t_eval_us);
    # 输出处理提示时的速度，单位为每秒处理的 token 数量
    fprintf(stream, "ts_p_eval: %.2f  # tokens / second during prompt processing\n",
            1.0e6 * ctx->n_p_eval / ctx->t_p_eval_us);
    # 输出采样的速度，单位为每秒采样的 token 数量
    fprintf(stream, "ts_sample: %.2f  # tokens / second during sampling\n",
            1.0e6 * ctx->n_sample / ctx->t_sample_us);
// 返回模型上下文中的张量名称到张量指针的映射，仅供内部测试使用
const std::vector<std::pair<std::string, struct ggml_tensor *>> & llama_internal_get_tensor_map(
    struct llama_context * ctx
) {
    return ctx->model.tensors_by_name;
}

// 设置日志回调函数和用户数据
void llama_log_set(ggml_log_callback log_callback, void * user_data) {
    // 如果传入的日志回调函数为空，则使用默认的日志回调函数
    g_state.log_callback = log_callback ? log_callback : llama_log_callback_default;
    g_state.log_callback_user_data = user_data;
    // 如果使用 Metal 后端，则设置 Metal 日志回调函数
#ifdef GGML_USE_METAL
    ggml_backend_metal_log_set_callback(g_state.log_callback, g_state.log_callback_user_data);
#endif
}

// 内部函数，处理带有可变参数列表的日志记录
static void llama_log_internal_v(ggml_log_level level, const char * format, va_list args) {
    va_list args_copy;
    va_copy(args_copy, args);
    char buffer[128];
    // 格式化日志消息
    int len = vsnprintf(buffer, 128, format, args);
    // 如果消息长度小于缓冲区大小，则调用日志回调函数
    if (len < 128) {
        g_state.log_callback(level, buffer, g_state.log_callback_user_data);
    } else {
        char* buffer2 = new char[len+1];
        vsnprintf(buffer2, len+1, format, args_copy);
        buffer2[len] = 0;
        g_state.log_callback(level, buffer2, g_state.log_callback_user_data);
        delete[] buffer2;
    }
    va_end(args_copy);
}

// 内部函数，处理不带可变参数列表的日志记录
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