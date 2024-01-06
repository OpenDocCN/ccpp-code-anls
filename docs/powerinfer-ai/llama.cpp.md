# `PowerInfer\llama.cpp`

```
# 定义 LLAMA_API_INTERNAL 宏
#define LLAMA_API_INTERNAL
# 包含 llama.h 头文件
#include "llama.h"
# 包含 unicode.h 头文件
#include "unicode.h"
# 包含 ggml.h 头文件
#include "ggml.h"
# 包含 ggml-alloc.h 头文件
#include "ggml-alloc.h"
# 如果定义了 GGML_USE_CUBLAS，则包含 ggml-cuda.h 头文件；否则，如果定义了 GGML_USE_CLBLAST，则包含 ggml-opencl.h 头文件
#ifdef GGML_USE_CUBLAS
#  include "ggml-cuda.h"
#elif defined(GGML_USE_CLBLAST)
#  include "ggml-opencl.h"
# 如果定义了 GGML_USE_METAL，则包含 ggml-metal.h 头文件
#ifdef GGML_USE_METAL
#  include "ggml-metal.h"
# 如果定义了 GGML_USE_MPI，则包含 ggml-mpi.h 头文件
#ifdef GGML_USE_MPI
#  include "ggml-mpi.h"
#ifndef QK_K
#  ifdef GGML_QKK_64
#    define QK_K 64
#  else
#    define QK_K 256
#  endif
#endif
```
- 如果未定义 QK_K，则根据 GGML_QKK_64 的定义来确定 QK_K 的值，如果未定义 GGML_QKK_64，则将 QK_K 定义为 256。

```
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
```
- 如果编译器支持 __has_include，且包含 <unistd.h>，则包含 <unistd.h> 头文件，并根据定义的宏来决定是否包含 <sys/mman.h> 和 <sys/resource.h> 头文件。
#if defined(_WIN32)
    #define WIN32_LEAN_AND_MEAN  // 定义 WIN32_LEAN_AND_MEAN 宏
    #ifndef NOMINMAX
        #define NOMINMAX  // 如果未定义 NOMINMAX 宏，则定义 NOMINMAX 宏
    #endif
    #include <windows.h>  // 包含 Windows 平台相关的头文件
    #include <io.h>  // 包含输入输出相关的头文件
    #include <stdio.h> // for _fseeki64  // 包含标准输入输出相关的头文件，用于 _fseeki64 函数
#endif

#include <algorithm>  // 包含算法相关的头文件
#include <array>  // 包含数组相关的头文件
#include <cassert>  // 包含断言相关的头文件
#include <cinttypes>  // 包含整数类型相关的头文件
#include <climits>  // 包含整数类型的最值相关的头文件
#include <cmath>  // 包含数学相关的头文件
#include <cstdarg>  // 包含可变参数相关的头文件
#include <cstddef>  // 包含标准定义相关的头文件
#include <cstdint>  // 包含整数类型相关的头文件
#include <cstdio> // C标准输入输出库
#include <cstring> // C字符串操作库
#include <ctime> // C时间库
#include <libgen.h> // POSIX标准中的函数原型
#include <forward_list> // 单向链表容器
#include <fstream> // 文件输入输出流库
#include <functional> // 函数对象库
#include <initializer_list> // 列表初始化库
#include <map> // 映射容器库
#include <memory> // 智能指针库
#include <mutex> // 互斥量库
#include <numeric> // 数值算法库
#include <queue> // 队列容器库
#include <random> // 随机数库
#include <regex> // 正则表达式库
#include <set> // 集合容器库
#include <sstream> // 字符串流库
#include <thread> // 线程库
#include <unordered_map> // 无序映射容器库
// 如果是在 Microsoft Visual C++ 编译器下，禁止警告 4244 和 4267，这两个警告表示可能会丢失数据
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 如果是在 GNU 编译器下
#ifdef __GNUC__
// 如果是在 MinGW32 下，使用 GNU printf 格式
#ifdef __MINGW32__
#define LLAMA_ATTRIBUTE_FORMAT(...) __attribute__((format(gnu_printf, __VA_ARGS__)))
// 否则，使用普通的 printf 格式
#else
#define LLAMA_ATTRIBUTE_FORMAT(...) __attribute__((format(printf, __VA_ARGS__)))
#endif
// 如果不是在 GNU 编译器下，不做任何处理
#else
#define LLAMA_ATTRIBUTE_FORMAT(...)
#endif

// 定义 LLAMA_MAX_NODES 为 4096
#define LLAMA_MAX_NODES 4096

// 
// 全局变量
// 
// 设置稀疏矩阵乘法预测的稀疏阈值
float sparse_pred_threshold = 0.;

//
// 日志记录
//

// 定义一个内部的日志记录函数，参数为日志级别、格式字符串和可变参数
static void llama_log_internal        (ggml_log_level level, const char* format, ...);
// 定义一个默认的日志回调函数，参数为日志级别、日志文本和用户数据
static void llama_log_callback_default(ggml_log_level level, const char * text, void * user_data);

// 定义宏，用于记录信息级别的日志
#define LLAMA_LOG_INFO(...)  llama_log_internal(GGML_LOG_LEVEL_INFO , __VA_ARGS__)
// 定义宏，用于记录警告级别的日志
#define LLAMA_LOG_WARN(...)  llama_log_internal(GGML_LOG_LEVEL_WARN , __VA_ARGS__)
// 定义宏，用于记录错误级别的日志
#define LLAMA_LOG_ERROR(...) llama_log_internal(GGML_LOG_LEVEL_ERROR, __VA_ARGS__)

//
// 辅助函数
//

// 计算UTF-8编码字符串的长度
static size_t utf8_len(char src) {
    // 定义一个包含16个元素的无符号整数数组lookup
    const size_t lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3, 4 };
    // 将src的值转换为无符号8位整数，然后右移4位，得到高4位的值
    uint8_t highbits = static_cast<uint8_t>(src) >> 4;
    // 返回lookup数组中highbits对应位置的值
    return lookup[highbits];
}

// 替换字符串s中所有的search子串为replace子串
static void replace_all(std::string & s, const std::string & search, const std::string & replace) {
    // 定义一个空字符串result
    std::string result;
    // 从位置0开始循环查找search子串
    for (size_t pos = 0; ; pos += search.length()) {
        // 在字符串s中从pos位置开始查找search子串
        auto new_pos = s.find(search, pos);
        // 如果找不到search子串，则将pos到末尾的子串加入result，结束循环
        if (new_pos == std::string::npos) {
            result += s.substr(pos, s.size() - pos);
            break;
        }
        // 将pos到new_pos之间的子串加入result，并加入replace子串
        result += s.substr(pos, new_pos - pos) + replace;
        // 更新pos为new_pos的位置
        pos = new_pos;
    }
    // 将result移动给字符串s
    s = std::move(result);
}

// 判断两个浮点数a和b是否在绝对误差范围内接近
static bool is_float_close(float a, float b, float abs_tol) {
// 检查绝对容差是否为非负数
if (abs_tol < 0.0) {
    throw std::invalid_argument("Tolerance must be non-negative");
}

// 精确相等性检查
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

#ifdef GGML_USE_CPU_HBM
```

// 这段代码是一个函数，用于比较两个数的大小，并根据给定的绝对容差进行比较。首先检查绝对容差是否为非负数，然后进行精确相等性检查，接着检查是否有无穷大的情况，最后使用提供的绝对容差进行常规比较。
// 包含 hbwmalloc.h 头文件
#include <hbwmalloc.h>
#endif

// 将文件流中的前 n 个字节置零
static void zeros(std::ofstream & file, size_t n) {
    char zero = 0;
    for (size_t i = 0; i < n; ++i) {
        file.write(&zero, 1);
    }
}

// 格式化字符串
LLAMA_ATTRIBUTE_FORMAT(1, 2)
static std::string format(const char * fmt, ...) {
    va_list ap;
    va_list ap2;
    va_start(ap, fmt);
    va_copy(ap2, ap);
    // 获取格式化后的字符串长度
    int size = vsnprintf(NULL, 0, fmt, ap);
    // 断言字符串长度大于等于 0 且小于 INT_MAX
    GGML_ASSERT(size >= 0 && size < INT_MAX); // NOLINT
    // 创建一个字符向量，大小为格式化后的字符串长度加 1
    std::vector<char> buf(size + 1);
    // 将格式化后的字符串写入字符向量
    int size2 = vsnprintf(buf.data(), size + 1, fmt, ap2);
    // 确保 size2 等于 size，如果不等则触发断言
    GGML_ASSERT(size2 == size);
    // 结束可变参数列表的使用
    va_end(ap2);
    // 结束可变参数列表的使用
    va_end(ap);
    // 从 buf 中创建一个包含 size 大小的字符串并返回
    return std::string(buf.data(), size);
}

//
// gguf 常量（与 gguf.py 同步）
//

// 定义枚举类型 llm_arch，包含多个枚举值
enum llm_arch {
    LLM_ARCH_LLAMA,
    LLM_ARCH_FALCON,
    LLM_ARCH_BAICHUAN,
    LLM_ARCH_GPT2,
    LLM_ARCH_GPTJ,
    LLM_ARCH_GPTNEOX,
    LLM_ARCH_MPT,
    LLM_ARCH_STARCODER,
    LLM_ARCH_PERSIMMON,
// 定义枚举类型，表示不同的架构类型
LLM_ARCH_REFACT,
LLM_ARCH_BLOOM,
LLM_ARCH_STABLELM,
LLM_ARCH_UNKNOWN,
};

// 创建架构类型到字符串名称的映射
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
# 定义枚举类型LLM_ARCH_UNKNOWN，对应的值为"unknown"
{ LLM_ARCH_UNKNOWN,         "unknown"   },

# 定义枚举类型LLM_KV_GENERAL_ARCHITECTURE，对应的值为0
LLM_KV_GENERAL_ARCHITECTURE,
# 定义枚举类型LLM_KV_GENERAL_QUANTIZATION_VERSION，对应的值为1
LLM_KV_GENERAL_QUANTIZATION_VERSION,
# 定义枚举类型LLM_KV_GENERAL_ALIGNMENT，对应的值为2
LLM_KV_GENERAL_ALIGNMENT,
# 定义枚举类型LLM_KV_GENERAL_NAME，对应的值为3
LLM_KV_GENERAL_NAME,
# 定义枚举类型LLM_KV_GENERAL_AUTHOR，对应的值为4
LLM_KV_GENERAL_AUTHOR,
# 定义枚举类型LLM_KV_GENERAL_URL，对应的值为5
LLM_KV_GENERAL_URL,
# 定义枚举类型LLM_KV_GENERAL_DESCRIPTION，对应的值为6
LLM_KV_GENERAL_DESCRIPTION,
# 定义枚举类型LLM_KV_GENERAL_LICENSE，对应的值为7
LLM_KV_GENERAL_LICENSE,
# 定义枚举类型LLM_KV_GENERAL_SOURCE_URL，对应的值为8
LLM_KV_GENERAL_SOURCE_URL,
# 定义枚举类型LLM_KV_GENERAL_SOURCE_HF_REPO，对应的值为9
LLM_KV_GENERAL_SOURCE_HF_REPO,

# 定义枚举类型LLM_KV_CONTEXT_LENGTH，对应的值为10
LLM_KV_CONTEXT_LENGTH,
# 定义枚举类型LLM_KV_EMBEDDING_LENGTH，对应的值为11
LLM_KV_EMBEDDING_LENGTH,
# 定义枚举类型LLM_KV_BLOCK_COUNT，对应的值为12
LLM_KV_BLOCK_COUNT,
# 定义枚举类型LLM_KV_FEED_FORWARD_LENGTH，对应的值为13
LLM_KV_FEED_FORWARD_LENGTH,
# 定义枚举类型LLM_KV_USE_PARALLEL_RESIDUAL，对应的值为14
LLM_KV_USE_PARALLEL_RESIDUAL,
    # 定义LLM_KV_TENSOR_DATA_LAYOUT

    # 定义LLM_KV_ATTENTION_HEAD_COUNT
    # 定义LLM_KV_ATTENTION_HEAD_COUNT_KV
    # 定义LLM_KV_ATTENTION_MAX_ALIBI_BIAS
    # 定义LLM_KV_ATTENTION_CLAMP_KQV
    # 定义LLM_KV_ATTENTION_LAYERNORM_EPS
    # 定义LLM_KV_ATTENTION_LAYERNORM_RMS_EPS

    # 定义LLM_KV_ROPE_DIMENSION_COUNT
    # 定义LLM_KV_ROPE_FREQ_BASE
    # 定义LLM_KV_ROPE_SCALE_LINEAR
    # 定义LLM_KV_ROPE_SCALING_TYPE
    # 定义LLM_KV_ROPE_SCALING_FACTOR
    # 定义LLM_KV_ROPE_SCALING_ORIG_CTX_LEN
    # 定义LLM_KV_ROPE_SCALING_FINETUNED

    # 定义LLM_KV_TOKENIZER_MODEL
    # 定义LLM_KV_TOKENIZER_LIST
    # 定义LLM_KV_TOKENIZER_TOKEN_TYPE
    LLM_KV_TOKENIZER_SCORES,  // 定义一个枚举值，表示tokenizer的scores
    LLM_KV_TOKENIZER_MERGES,  // 定义一个枚举值，表示tokenizer的merges
    LLM_KV_TOKENIZER_BOS_ID,  // 定义一个枚举值，表示tokenizer的bos_id
    LLM_KV_TOKENIZER_EOS_ID,  // 定义一个枚举值，表示tokenizer的eos_id
    LLM_KV_TOKENIZER_UNK_ID,  // 定义一个枚举值，表示tokenizer的unk_id
    LLM_KV_TOKENIZER_SEP_ID,  // 定义一个枚举值，表示tokenizer的sep_id
    LLM_KV_TOKENIZER_PAD_ID,  // 定义一个枚举值，表示tokenizer的pad_id
    LLM_KV_TOKENIZER_HF_JSON,  // 定义一个枚举值，表示tokenizer的hf_json
    LLM_KV_TOKENIZER_RWKV,     // 定义一个枚举值，表示tokenizer的rwkv

    LLM_KV_SPARSE_THRESHOLD,   // 定义一个枚举值，表示sparse的threshold

    LLM_KV_SPLIT_VRAM_CAPACITY,  // 定义一个枚举值，表示split的vram_capacity
};

static std::map<llm_kv, std::string> LLM_KV_NAMES = {
    { LLM_KV_GENERAL_ARCHITECTURE,          "general.architecture"                  },  // 将枚举值和对应的字符串映射关系存入map中
    { LLM_KV_GENERAL_QUANTIZATION_VERSION,  "general.quantization_version"          },
    { LLM_KV_GENERAL_ALIGNMENT,             "general.alignment"                     },
    { LLM_KV_GENERAL_NAME,                  "general.name"                          },
    { LLM_KV_GENERAL_AUTHOR,                "general.author"                        },  // 将 general.author 存储到 LLM_KV_GENERAL_AUTHOR
    { LLM_KV_GENERAL_URL,                   "general.url"                           },  // 将 general.url 存储到 LLM_KV_GENERAL_URL
    { LLM_KV_GENERAL_DESCRIPTION,           "general.description"                   },  // 将 general.description 存储到 LLM_KV_GENERAL_DESCRIPTION
    { LLM_KV_GENERAL_LICENSE,               "general.license"                       },  // 将 general.license 存储到 LLM_KV_GENERAL_LICENSE
    { LLM_KV_GENERAL_SOURCE_URL,            "general.source.url"                    },  // 将 general.source.url 存储到 LLM_KV_GENERAL_SOURCE_URL
    { LLM_KV_GENERAL_SOURCE_HF_REPO,        "general.source.huggingface.repository" },  // 将 general.source.huggingface.repository 存储到 LLM_KV_GENERAL_SOURCE_HF_REPO

    { LLM_KV_CONTEXT_LENGTH,                "%s.context_length"        },  // 将 %s.context_length 存储到 LLM_KV_CONTEXT_LENGTH
    { LLM_KV_EMBEDDING_LENGTH,              "%s.embedding_length"      },  // 将 %s.embedding_length 存储到 LLM_KV_EMBEDDING_LENGTH
    { LLM_KV_BLOCK_COUNT,                   "%s.block_count"           },  // 将 %s.block_count 存储到 LLM_KV_BLOCK_COUNT
    { LLM_KV_FEED_FORWARD_LENGTH,           "%s.feed_forward_length"   },  // 将 %s.feed_forward_length 存储到 LLM_KV_FEED_FORWARD_LENGTH
    { LLM_KV_USE_PARALLEL_RESIDUAL,         "%s.use_parallel_residual" },  // 将 %s.use_parallel_residual 存储到 LLM_KV_USE_PARALLEL_RESIDUAL
    { LLM_KV_TENSOR_DATA_LAYOUT,            "%s.tensor_data_layout"    },  // 将 %s.tensor_data_layout 存储到 LLM_KV_TENSOR_DATA_LAYOUT

    { LLM_KV_ATTENTION_HEAD_COUNT,          "%s.attention.head_count"             },  // 将 %s.attention.head_count 存储到 LLM_KV_ATTENTION_HEAD_COUNT
    { LLM_KV_ATTENTION_HEAD_COUNT_KV,       "%s.attention.head_count_kv"          },  // 将 %s.attention.head_count_kv 存储到 LLM_KV_ATTENTION_HEAD_COUNT_KV
    { LLM_KV_ATTENTION_MAX_ALIBI_BIAS,      "%s.attention.max_alibi_bias"         },  // 将 %s.attention.max_alibi_bias 存储到 LLM_KV_ATTENTION_MAX_ALIBI_BIAS
    { LLM_KV_ATTENTION_CLAMP_KQV,           "%s.attention.clamp_kqv"              },  // 将 %s.attention.clamp_kqv 存储到 LLM_KV_ATTENTION_CLAMP_KQV
    { LLM_KV_ATTENTION_LAYERNORM_EPS,       "%s.attention.layer_norm_epsilon"     },  // 将 %s.attention.layer_norm_epsilon 存储到 LLM_KV_ATTENTION_LAYERNORM_EPS
    { LLM_KV_ATTENTION_LAYERNORM_RMS_EPS,   "%s.attention.layer_norm_rms_epsilon" },  // 将 %s.attention.layer_norm_rms_epsilon 存储到 LLM_KV_ATTENTION_LAYERNORM_RMS_EPS
    { LLM_KV_ROPE_DIMENSION_COUNT,          "%s.rope.dimension_count"                 },  // 定义键值对，表示绳子维度数量
    { LLM_KV_ROPE_FREQ_BASE,                "%s.rope.freq_base"                       },  // 定义键值对，表示绳子频率基数
    { LLM_KV_ROPE_SCALE_LINEAR,             "%s.rope.scale_linear"                    },  // 定义键值对，表示绳子线性缩放
    { LLM_KV_ROPE_SCALING_TYPE,             "%s.rope.scaling.type"                    },  // 定义键值对，表示绳子缩放类型
    { LLM_KV_ROPE_SCALING_FACTOR,           "%s.rope.scaling.factor"                  },  // 定义键值对，表示绳子缩放因子
    { LLM_KV_ROPE_SCALING_ORIG_CTX_LEN,     "%s.rope.scaling.original_context_length" },  // 定义键值对，表示绳子原始上下文长度
    { LLM_KV_ROPE_SCALING_FINETUNED,        "%s.rope.scaling.finetuned"               },  // 定义键值对，表示绳子微调缩放

    { LLM_KV_TOKENIZER_MODEL,               "tokenizer.ggml.model"              },  // 定义键值对，表示分词器模型
    { LLM_KV_TOKENIZER_LIST,                "tokenizer.ggml.tokens"             },  // 定义键值对，表示分词器标记列表
    { LLM_KV_TOKENIZER_TOKEN_TYPE,          "tokenizer.ggml.token_type"         },  // 定义键值对，表示分词器标记类型
    { LLM_KV_TOKENIZER_SCORES,              "tokenizer.ggml.scores"             },  // 定义键值对，表示分词器分数
    { LLM_KV_TOKENIZER_MERGES,              "tokenizer.ggml.merges"             },  // 定义键值对，表示分词器合并
    { LLM_KV_TOKENIZER_BOS_ID,              "tokenizer.ggml.bos_token_id"       },  // 定义键值对，表示分词器起始标记 ID
    { LLM_KV_TOKENIZER_EOS_ID,              "tokenizer.ggml.eos_token_id"       },  // 定义键值对，表示分词器结束标记 ID
    { LLM_KV_TOKENIZER_UNK_ID,              "tokenizer.ggml.unknown_token_id"   },  // 定义键值对，表示分词器未知标记 ID
    { LLM_KV_TOKENIZER_SEP_ID,              "tokenizer.ggml.seperator_token_id" },  // 定义键值对，表示分词器分隔符标记 ID
    { LLM_KV_TOKENIZER_PAD_ID,              "tokenizer.ggml.padding_token_id"   },  // 定义键值对，表示分词器填充标记 ID
    { LLM_KV_TOKENIZER_HF_JSON,             "tokenizer.huggingface.json"        },  // 定义键值对，表示 Hugging Face 分词器的 JSON 数据
    { LLM_KV_TOKENIZER_RWKV,                "tokenizer.rwkv.world"              }, 
    // 定义一个键值对，LLM_KV_TOKENIZER_RWKV作为键，"tokenizer.rwkv.world"作为值

    { LLM_KV_SPARSE_THRESHOLD,              "powerinfer.sparse_threshold" },
    // 定义一个键值对，LLM_KV_SPARSE_THRESHOLD作为键，"powerinfer.sparse_threshold"作为值

    { LLM_KV_SPLIT_VRAM_CAPACITY,           "split.vram_capacity" },
    // 定义一个键值对，LLM_KV_SPLIT_VRAM_CAPACITY作为键，"split.vram_capacity"作为值
};

struct LLM_KV {
    LLM_KV(llm_arch arch) : arch(arch) {}
    // 定义一个结构体LLM_KV，包含一个构造函数，用于初始化arch成员变量

    llm_arch arch;
    // 定义一个llm_arch类型的成员变量arch

    std::string operator()(llm_kv kv) const {
        return ::format(LLM_KV_NAMES[kv].c_str(), LLM_ARCH_NAMES[arch].c_str());
    }
    // 重载()运算符，根据传入的llm_kv类型的参数kv，返回LLM_KV_NAMES和LLM_ARCH_NAMES数组中对应位置的字符串格式化结果
};

enum llm_tensor {
    LLM_TENSOR_TOKEN_EMBD,
    LLM_TENSOR_TOKEN_EMBD_NORM,
    // 定义一个枚举类型llm_tensor，包含LLM_TENSOR_TOKEN_EMBD和LLM_TENSOR_TOKEN_EMBD_NORM两个枚举值
    LLM_TENSOR_POS_EMBD,  # 定义LLM模型中的位置嵌入张量
    LLM_TENSOR_OUTPUT,  # 定义LLM模型中的输出张量
    LLM_TENSOR_OUTPUT_NORM,  # 定义LLM模型中的输出规范化张量
    LLM_TENSOR_ROPE_FREQS,  # 定义LLM模型中的ROPE频率张量
    LLM_TENSOR_ATTN_Q,  # 定义LLM模型中的注意力查询张量
    LLM_TENSOR_ATTN_K,  # 定义LLM模型中的注意力键张量
    LLM_TENSOR_ATTN_V,  # 定义LLM模型中的注意力值张量
    LLM_TENSOR_ATTN_QKV,  # 定义LLM模型中的注意力查询-键-值张量
    LLM_TENSOR_ATTN_OUT,  # 定义LLM模型中的注意力输出张量
    LLM_TENSOR_ATTN_NORM,  # 定义LLM模型中的注意力规范化张量
    LLM_TENSOR_ATTN_NORM_2,  # 定义LLM模型中的注意力规范化2张量
    LLM_TENSOR_ATTN_ROT_EMBD,  # 定义LLM模型中的注意力旋转嵌入张量
    LLM_TENSOR_FFN_GATE,  # 定义LLM模型中的FFN门控张量
    LLM_TENSOR_FFN_DOWN,  # 定义LLM模型中的FFN下采样张量
    LLM_TENSOR_FFN_UP,  # 定义LLM模型中的FFN上采样张量
    LLM_TENSOR_FFN_NORM,  # 定义LLM模型中的FFN规范化张量
    LLM_TENSOR_ATTN_Q_NORM,  # 定义LLM模型中的注意力查询规范化张量
    LLM_TENSOR_ATTN_K_NORM,  # 定义LLM模型中的注意力键规范化张量
    LLM_TENSOR_MLP_PRED_FC1,  # 定义LLM模型中的MLP预测全连接层1张量
    LLM_TENSOR_MLP_PRED_FC2,  # 定义LLM模型中的MLP预测全连接层2张量
// 定义枚举类型 LLM_TENSOR，表示不同的张量类型
enum LLM_TENSOR {
    LLM_TENSOR_TOKEN_EMBD,
    LLM_TENSOR_OUTPUT_NORM,
    LLM_TENSOR_OUTPUT,
    LLM_TENSOR_ROPE_FREQS,
    LLM_TENSOR_ATTN_NORM,
    LLM_TENSOR_ATTN_Q,
    LLM_TENSOR_ATTN_K,
    LLM_TENSOR_ATTN_V,
    LLM_TENSOR_ATTN_OUT,
    LLM_TENSOR_ATTN_ROT_EMBD,
    LLM_TENSOR_FFN_NORM,
    LLM_TENSOR_FFN_GATE,
    LLM_TENSOR_FFN_DOWN_T,
};

// 定义嵌套的 map 结构 LLM_TENSOR_NAMES，用于存储不同架构下不同张量的名称
static std::map<llm_arch, std::map<llm_tensor, std::string>> LLM_TENSOR_NAMES = {
    {
        LLM_ARCH_LLAMA, // 第一层 map 的键为 LLM_ARCH_LLAMA
        {
            // 第二层 map 存储了不同张量类型对应的名称
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
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
        }
    }
};
    # 定义了一个包含元组的列表，每个元组包含了一个整数和一个字符串，用于表示张量类型和对应的文件名
    { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # 表示前馈神经网络向上的张量类型和文件名模板
    { LLM_TENSOR_FFN_DOWN_T,      "blk.%d.ffn_down_t" },  # 表示前馈神经网络向下的张量类型和文件名模板
    { LLM_TENSOR_MLP_PRED_FC1,    "blk.%d.fc1" },  # 表示多层感知机预测第一层的张量类型和文件名模板
    { LLM_TENSOR_MLP_PRED_FC2,    "blk.%d.fc2" },  # 表示多层感知机预测第二层的张量类型和文件名模板
    # ...
    # 定义了另一个包含元组的列表，每个元组包含了一个整数和一个字符串，用于表示张量类型和对应的文件名
    { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # 表示令牌嵌入的张量类型和文件名
    { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # 表示输出规范化的张量类型和文件名
    { LLM_TENSOR_OUTPUT,          "output" },  # 表示输出的张量类型和文件名
    { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },  # 表示绳索频率的张量类型和文件名
    { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # 表示注意力规范化的张量类型和文件名模板
    { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },  # 表示注意力查询的张量类型和文件名模板
    { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },  # 表示注意力键的张量类型和文件名模板
    { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },  # 表示注意力值的张量类型和文件名模板
    { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # 表示注意力输出的张量类型和文件名模板
    { LLM_TENSOR_ATTN_ROT_EMBD,   "blk.%d.attn_rot_embd" },  # 表示注意力旋转嵌入的张量类型和文件名模板
    { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # 表示前馈神经网络规范化的张量类型和文件名模板
    // 定义LLM_ARCH_TRANSFORMER类型的张量名称和对应的文件名格式
    {
        LLM_ARCH_TRANSFORMER,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 词嵌入张量
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 输出规范化张量
            { LLM_TENSOR_OUTPUT,          "output" },  // 输出张量
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 注意力规范化张量
            { LLM_TENSOR_ATTN_NORM_2,     "blk.%d.attn_norm_2" },  // 第二个注意力规范化张量
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  // 注意力查询、键、值张量
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 注意力输出张量
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 下采样前馈神经网络张量
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 上采样前馈神经网络张量
        },
    },
    // 定义LLM_ARCH_FALCON类型的张量名称和对应的文件名格式
    {
        LLM_ARCH_FALCON,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 词嵌入张量
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 输出规范化张量
            { LLM_TENSOR_OUTPUT,          "output" },  // 输出张量
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 注意力规范化张量
            { LLM_TENSOR_ATTN_NORM_2,     "blk.%d.attn_norm_2" },  // 第二个注意力规范化张量
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  // 注意力查询、键、值张量
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 注意力输出张量
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 下采样前馈神经网络张量
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 上采样前馈神经网络张量
            { LLM_TENSOR_FFN_DOWN_T,      "blk.%d.ffn_down_t" },  // 下采样前馈神经网络张量
            { LLM_TENSOR_MLP_PRED_FC1,    "blk.%d.fc1" },  // MLP预测全连接层1张量
            { LLM_TENSOR_MLP_PRED_FC2,    "blk.%d.fc2" },  // MLP预测全连接层2张量
        },
    },
    },
    {
        LLM_ARCH_GPT2,  // 定义 GPT2 架构
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义 token_embd 张量
        },
    },
    {
        LLM_ARCH_GPTJ,  // 定义 GPTJ 架构
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义 token_embd 张量
        },
    },
    {
        LLM_ARCH_GPTNEOX,  // 定义 GPTNEOX 架构
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义 token_embd 张量
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义 output_norm 张量
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义 output 张量
    # 定义一个包含张量类型和对应文件名格式的映射关系的数组
    {
        LLM_ARCH_APPLE,
        {
            # 定义张量类型为LLM_TENSOR_ATTN_NORM，对应的文件名格式为"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },
            # 定义张量类型为LLM_TENSOR_ATTN_QKV，对应的文件名格式为"blk.%d.attn_qkv"
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },
            # 定义张量类型为LLM_TENSOR_ATTN_OUT，对应的文件名格式为"blk.%d.attn_output"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            # 定义张量类型为LLM_TENSOR_FFN_NORM，对应的文件名格式为"blk.%d.ffn_norm"
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },
            # 定义张量类型为LLM_TENSOR_FFN_DOWN，对应的文件名格式为"blk.%d.ffn_down"
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
            # 定义张量类型为LLM_TENSOR_FFN_UP，对应的文件名格式为"blk.%d.ffn_up"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },
        },
    },
    # 定义另一个架构LLM_ARCH_PERSIMMON的张量类型和对应文件名格式的映射关系
    {
        LLM_ARCH_PERSIMMON,
        {
            # 定义张量类型为LLM_TENSOR_TOKEN_EMBD，对应的文件名格式为"token_embd"
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd"},
            # 定义张量类型为LLM_TENSOR_OUTPUT_NORM，对应的文件名格式为"output_norm"
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm"},
            # 定义张量类型为LLM_TENSOR_OUTPUT，对应的文件名格式为"output"
            { LLM_TENSOR_OUTPUT,          "output"},
            # 定义张量类型为LLM_TENSOR_ATTN_NORM，对应的文件名格式为"blk.%d.attn_norm"
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm"},
            # 定义张量类型为LLM_TENSOR_ATTN_QKV，对应的文件名格式为"blk.%d.attn_qkv"
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv"},
            # 定义张量类型为LLM_TENSOR_ATTN_OUT，对应的文件名格式为"blk.%d.attn_output"
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output"},
            # 定义张量类型为LLM_TENSOR_ATTN_Q_NORM，对应的文件名格式为"blk.%d.attn_q_norm"
            { LLM_TENSOR_ATTN_Q_NORM,     "blk.%d.attn_q_norm"},
            # 定义张量类型为LLM_TENSOR_ATTN_K_NORM，对应的文件名格式为"blk.%d.attn_k_norm"
            { LLM_TENSOR_ATTN_K_NORM,     "blk.%d.attn_k_norm"},
            # 定义张量类型为LLM_TENSOR_FFN_NORM，对应的文件名格式为"blk.%d.ffn_norm"
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm"},
    {
        LLM_ARCH_MPT,  // 定义了LLM_ARCH_MPT的架构
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  // 定义了LLM_TENSOR_TOKEN_EMBD和对应的文件名
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  // 定义了LLM_TENSOR_OUTPUT_NORM和对应的文件名
            { LLM_TENSOR_OUTPUT,          "output" },  // 定义了LLM_TENSOR_OUTPUT和对应的文件名
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  // 定义了LLM_TENSOR_ATTN_NORM和对应的文件名
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  // 定义了LLM_TENSOR_FFN_NORM和对应的文件名
            { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  // 定义了LLM_TENSOR_ATTN_QKV和对应的文件名
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  // 定义了LLM_TENSOR_ATTN_OUT和对应的文件名
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  // 定义了LLM_TENSOR_FFN_DOWN和对应的文件名
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  // 定义了LLM_TENSOR_FFN_UP和对应的文件名
        },
    },
    # 定义LLM_ARCH_STARCODER架构的张量名称和对应的键值
    LLM_ARCH_STARCODER,
    {
        { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # token_embd张量对应的键值
        { LLM_TENSOR_POS_EMBD,        "position_embd" },  # position_embd张量对应的键值
        { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # output_norm张量对应的键值
        { LLM_TENSOR_OUTPUT,          "output" },  # output张量对应的键值
        { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },  # blk.%d.attn_norm张量对应的键值
        { LLM_TENSOR_ATTN_QKV,        "blk.%d.attn_qkv" },  # blk.%d.attn_qkv张量对应的键值
        { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },  # blk.%d.attn_output张量对应的键值
        { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },  # blk.%d.ffn_norm张量对应的键值
        { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },  # blk.%d.ffn_up张量对应的键值
        { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },  # blk.%d.ffn_down张量对应的键值
    },
    # 定义LLM_ARCH_REFACT架构的张量名称和对应的键值
    {
        LLM_ARCH_REFACT,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },  # token_embd张量对应的键值
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },  # output_norm张量对应的键值
            { LLM_TENSOR_OUTPUT,          "output" },  # output张量对应的键值
# 定义了一个包含元组的列表，每个元组包含了一个枚举值和对应的字符串
# LLM_TENSOR_ATTN_NORM 对应 "blk.%d.attn_norm"
# LLM_TENSOR_ATTN_Q 对应 "blk.%d.attn_q"
# LLM_TENSOR_ATTN_K 对应 "blk.%d.attn_k"
# LLM_TENSOR_ATTN_V 对应 "blk.%d.attn_v"
# LLM_TENSOR_ATTN_OUT 对应 "blk.%d.attn_output"
# LLM_TENSOR_FFN_NORM 对应 "blk.%d.ffn_norm"
# LLM_TENSOR_FFN_GATE 对应 "blk.%d.ffn_gate"
# LLM_TENSOR_FFN_DOWN 对应 "blk.%d.ffn_down"
# LLM_TENSOR_FFN_UP 对应 "blk.%d.ffn_up"
# LLM_ARCH_BLOOM 对应的是另一个包含元组的列表
# 每个元组包含了一个枚举值和对应的字符串
# LLM_TENSOR_TOKEN_EMBD 对应 "token_embd"
# LLM_TENSOR_TOKEN_EMBD_NORM 对应 "token_embd_norm"
# LLM_TENSOR_OUTPUT_NORM 对应 "output_norm"
# LLM_TENSOR_OUTPUT 对应 "output"
# LLM_TENSOR_ATTN_NORM 对应 "blk.%d.attn_norm"
# LLM_TENSOR_ATTN_QKV 对应 "blk.%d.attn_qkv"
    // 定义了一个包含键值对的数组，键是LLM_TENSOR枚举值，值是对应的字符串
    {
        LLM_ARCH_TRANSFORMER,
        {
            // 定义了LLM_TENSOR_ATTN_OUT对应的字符串
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            // 定义了LLM_TENSOR_FFN_NORM对应的字符串
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },
            // 定义了LLM_TENSOR_FFN_UP对应的字符串
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },
            // 定义了LLM_TENSOR_FFN_DOWN对应的字符串
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" },
        },
    },
    {
        LLM_ARCH_STABLELM,
        {
            // 定义了LLM_TENSOR_TOKEN_EMBD对应的字符串
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },
            // 定义了LLM_TENSOR_OUTPUT_NORM对应的字符串
            { LLM_TENSOR_OUTPUT_NORM,     "output_norm" },
            // 定义了LLM_TENSOR_OUTPUT对应的字符串
            { LLM_TENSOR_OUTPUT,          "output" },
            // 定义了LLM_TENSOR_ROPE_FREQS对应的字符串
            { LLM_TENSOR_ROPE_FREQS,      "rope_freqs" },
            // 定义了LLM_TENSOR_ATTN_NORM对应的字符串
            { LLM_TENSOR_ATTN_NORM,       "blk.%d.attn_norm" },
            // 定义了LLM_TENSOR_ATTN_Q对应的字符串
            { LLM_TENSOR_ATTN_Q,          "blk.%d.attn_q" },
            // 定义了LLM_TENSOR_ATTN_K对应的字符串
            { LLM_TENSOR_ATTN_K,          "blk.%d.attn_k" },
            // 定义了LLM_TENSOR_ATTN_V对应的字符串
            { LLM_TENSOR_ATTN_V,          "blk.%d.attn_v" },
            // 定义了LLM_TENSOR_ATTN_OUT对应的字符串
            { LLM_TENSOR_ATTN_OUT,        "blk.%d.attn_output" },
            // 定义了LLM_TENSOR_FFN_NORM对应的字符串
            { LLM_TENSOR_FFN_NORM,        "blk.%d.ffn_norm" },
            // 定义了LLM_TENSOR_FFN_GATE对应的字符串
            { LLM_TENSOR_FFN_GATE,        "blk.%d.ffn_gate" },
// 定义了一个名为LLM_ARCH_NAMES的静态映射表，将LLM_TENSOR枚举值映射到对应的字符串
static const std::map<llm_tensor, const char *> LLM_ARCH_NAMES[] = {
    // LLM_ARCH_TRANSFORMER下的映射关系
    {
        LLM_ARCH_TRANSFORMER,
        {
            { LLM_TENSOR_FFN_DOWN,        "blk.%d.ffn_down" }, // 将LLM_TENSOR_FFN_DOWN映射为"blk.%d.ffn_down"
            { LLM_TENSOR_FFN_UP,          "blk.%d.ffn_up" },   // 将LLM_TENSOR_FFN_UP映射为"blk.%d.ffn_up"
        },
    },
    // LLM_ARCH_UNKNOWN下的映射关系
    {
        LLM_ARCH_UNKNOWN,
        {
            { LLM_TENSOR_TOKEN_EMBD,      "token_embd" },      // 将LLM_TENSOR_TOKEN_EMBD映射为"token_embd"
        },
    },
};

// 根据给定的字符串name，返回对应的llm_arch枚举值
static llm_arch llm_arch_from_string(const std::string & name) {
    // 遍历LLM_ARCH_NAMES映射表
    for (const auto & kv : LLM_ARCH_NAMES) { // NOLINT
        // 如果找到对应的字符串name，则返回对应的llm_arch枚举值
        if (kv.second == name) {
            return kv.first;
        }
    }
    // 返回未知的架构类型
    return LLM_ARCH_UNKNOWN;
}

// 定义张量的离线级别枚举
enum tensor_offloading_levels {
    TENSOR_NO_OFFLOAD, // 无离线
    TENSOR_OFFLOAD_FFN, // 前馈神经网络离线
    TENSOR_OFFLOAD_ATTN, // 注意力离线
    TENSOR_OFFLOAD_MLP_PRED, // 多层感知机预测离线
    TENSOR_OFFLOAD_OUTPUT, // 输出离线
    TENSOR_OFFLOAD_KV_CACHE, // 键值缓存离线
};

// 获取张量的离线级别
tensor_offloading_levels get_offloading_level(llm_tensor tensor) {
    switch (tensor) {
        case LLM_TENSOR_TOKEN_EMBD: case LLM_TENSOR_TOKEN_EMBD_NORM: case LLM_TENSOR_POS_EMBD: 
        case LLM_TENSOR_ROPE_FREQS:
            return TENSOR_NO_OFFLOAD; // 返回无离线
        case LLM_TENSOR_OUTPUT: case LLM_TENSOR_OUTPUT_NORM:
            return TENSOR_OFFLOAD_OUTPUT; // 返回输出离线
        case LLM_TENSOR_ATTN_Q: case LLM_TENSOR_ATTN_K: case LLM_TENSOR_ATTN_V: 
```

// 根据不同的输入值返回对应的张量处理类型
switch (input_value) {
    // 如果输入值为以下几种情况之一，则返回张量处理类型为 TENSOR_OFFLOAD_ATTN
    case LLM_TENSOR_ATTN_QKV: case LLM_TENSOR_ATTN_OUT: case LLM_TENSOR_ATTN_NORM: 
    case LLM_TENSOR_ATTN_NORM_2: case LLM_TENSOR_ATTN_ROT_EMBD:
    case LLM_TENSOR_ATTN_Q_NORM: case LLM_TENSOR_ATTN_K_NORM:
        return TENSOR_OFFLOAD_ATTN;
    // 如果输入值为以下几种情况之一，则返回张量处理类型为 TENSOR_OFFLOAD_FFN
    case LLM_TENSOR_FFN_GATE: case LLM_TENSOR_FFN_DOWN: case LLM_TENSOR_FFN_UP:
    case LLM_TENSOR_FFN_NORM: case LLM_TENSOR_FFN_DOWN_T: 
        return TENSOR_OFFLOAD_FFN;
    // 如果输入值为以下几种情况之一，则返回张量处理类型为 TENSOR_OFFLOAD_MLP_PRED
    case LLM_TENSOR_MLP_PRED_FC1: case LLM_TENSOR_MLP_PRED_FC2:
        return TENSOR_OFFLOAD_MLP_PRED;
    // 如果输入值不属于以上任何一种情况，则抛出运行时错误，提示未知的张量类别
    default:
        throw std::runtime_error("unknown tensor category");
}

// 辅助函数，用于处理 gguf 常量
// 用法：
// 
//   const auto tn = LLM_TN(LLM_ARCH_LLAMA);
// 定义了一个结构体 LLM_TN，用于处理不同类型的张量命名和级别
struct LLM_TN {
    // 构造函数，初始化结构体的 arch 属性
    LLM_TN(llm_arch arch) : arch(arch) {}

    // 属性，表示架构类型
    llm_arch arch;

    // 重载运算符，返回张量名称和级别的 pair 对象
    std::pair<std::string, tensor_offloading_levels> operator()(llm_tensor tensor) const {
        return std::make_pair(LLM_TENSOR_NAMES[arch].at(tensor), get_offloading_level(tensor));
    }

    // 重载运算符，返回张量名称加上后缀和级别的 pair 对象
    std::pair<std::string, tensor_offloading_levels> operator()(llm_tensor tensor, const std::string & suffix) const {
        return std::make_pair(LLM_TENSOR_NAMES[arch].at(tensor) + "." + suffix, get_offloading_level(tensor));
    }

    // 重载运算符，返回张量名称加上编号和级别的 pair 对象
    std::pair<std::string, tensor_offloading_levels> operator()(llm_tensor tensor, int bid) const {
        return std::make_pair(::format(LLM_TENSOR_NAMES[arch].at(tensor).c_str(), bid), get_offloading_level(tensor));
    }
}
// 重载运算符，返回一个键值对，键是字符串，值是枚举类型 tensor_offloading_levels
std::pair<std::string, tensor_offloading_levels> operator()(llm_tensor tensor, const std::string & suffix, int bid) const {
    // 使用 format 函数格式化字符串，得到键名，然后与后缀拼接，再获取对应的 offloading level
    return std::make_pair(::format(LLM_TENSOR_NAMES[arch].at(tensor).c_str(), bid) + "." + suffix, get_offloading_level(tensor));
}

// GGUF 辅助函数
#define GGUF_GET_KEY(ctx, dst, func, type, req, key) \
do { \
    // 将 key 转换为字符串
    const std::string skey(key); \
    // 在上下文中查找键的索引
    const int kid = gguf_find_key(ctx, skey.c_str()); \
    // 如果找到了键
    if (kid >= 0) { \
        // 获取键值的类型
        enum gguf_type ktype = gguf_get_kv_type(ctx, kid); \
        // 如果类型不匹配
        if (ktype != (type)) { \
            // 抛出运行时错误，指出键的类型错误
            throw std::runtime_error(format("key %s has wrong type: %s", skey.c_str(), gguf_type_name(ktype))); \
        } \
        // 使用给定的函数获取键值
        (dst) = func(ctx, kid); \
```
// 如果请求存在，抛出运行时错误，指明模型中未找到对应的键
} else if (req) { \
    throw std::runtime_error(format("key not found in model: %s", skey.c_str())); \
} \
} while (0)

// 定义静态映射，将整数和字符串对应起来
static std::map<int8_t, std::string> LLAMA_ROPE_SCALING_TYPES = {
    { LLAMA_ROPE_SCALING_NONE,   "none"   },
    { LLAMA_ROPE_SCALING_LINEAR, "linear" },
    { LLAMA_ROPE_SCALING_YARN,   "yarn"   },
};

// 根据字符串返回对应的整数值
static int8_t llama_rope_scaling_type_from_string(const std::string & name) {
    // 遍历静态映射，查找与输入字符串对应的整数值
    for (const auto & kv : LLAMA_ROPE_SCALING_TYPES) {
        if (kv.second == name) {
            return kv.first;
        }
    }
    // 如果未找到对应的整数值，则返回未指定的值
    return LLAMA_ROPE_SCALING_UNSPECIFIED;
}
// ggml helpers

// 计算图形的辅助函数，将计算结果存储在buf中，使用给定的图形和线程数
static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    // 使用给定的图形和线程数创建计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划中有工作量
    if (plan.work_size > 0) {
        // 调整buf的大小以容纳工作数据
        buf.resize(plan.work_size);
        // 将工作数据指针指向buf的数据
        plan.work_data = buf.data();
    }

    // 执行图形计算
    ggml_graph_compute(graph, &plan);
}

// llama helpers
// 根据宏定义选择不同的内存分配方式
inline void * llama_host_malloc(size_t n) {
    // 如果使用CUBLAS并且CUBLAS已加载，则使用CUDA主机内存分配
    if (ggml_cublas_loaded()) {
        return ggml_cuda_host_malloc(n);
    } else {
        // 否则使用标准的malloc进行内存分配
        return malloc(n);
    }
    // 如果使用Metal，则使用Metal主机内存分配
    return ggml_metal_host_malloc(n);
    // 如果使用CPU HBM，则使用HBM内存分配
    return hbw_malloc(n);
    // 如果没有定义特定的内存分配方式，则使用标准的malloc进行内存分配
    return malloc(n);
}

// 根据宏定义选择不同的内存释放方式
inline void llama_host_free(void * ptr) {
    // 如果使用CUBLAS并且CUBLAS已加载，则使用CUDA主机内存释放
    if (ggml_cublas_loaded()) {
        return ggml_cuda_host_free(ptr);
    } else {
        // 如果不满足上述条件，释放指针内存
        return free(ptr);
    }
#elif GGML_USE_METAL
    // 如果使用 Metal，调用 ggml_metal_host_free 函数释放指针内存
    return ggml_metal_host_free(ptr);
#elif GGML_USE_CPU_HBM
    // 如果使用 CPU HBM，调用 hbw_free 函数释放指针内存
    return hbw_free(ptr);
#else
    // 如果以上条件都不满足，调用标准库函数 free 释放指针内存
    return free(ptr);
#endif
}

#if defined(_WIN32)
// 定义一个静态函数，用于将 Windows 错误码转换为字符串
static std::string llama_format_win_err(DWORD err) {
    LPSTR buf;
    // 调用 Windows API 函数 FormatMessageA 将错误码转换为字符串
    size_t size = FormatMessageA(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
                                 NULL, err, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPSTR)&buf, 0, NULL);
    // 如果转换失败，返回错误信息
    if (!size) {
        return "FormatMessageA failed";
    }
    // 使用 buf 中的数据和大小创建一个字符串对象
    std::string ret(buf, size);
    // 释放 buf 所指向的内存
    LocalFree(buf);
    // 返回创建的字符串对象
    return ret;
}
#endif

struct llama_buffer {
    // 数据指针初始化为空
    void * data = NULL;
    // 大小初始化为 0
    size_t size = 0;

    // 如果 CUDA 尝试分配固定内存，可以回退到使用 malloc / free
    bool fallback = false;

    // 重新分配内存大小
    void resize(size_t n) {
        // 释放当前数据指针所指向的内存
        llama_host_free(data);

        // 分配新的内存空间
        data = llama_host_malloc(n);
        // 如果分配失败，则将 fallback 标记为 true
        if (!data) {
            fallback = true;
    // 分配 n 个字节的内存空间
    data = malloc(n);
    // 如果分配失败，则将 fallback 设为 false
    } else {
        fallback = false;
    }

    // 断言 data 不为空，如果为空则触发断言错误
    GGML_ASSERT(data);
    // 将 size 设为 n
    size = n;
    }

    // 析构函数，用于释放内存
    ~llama_buffer() {
        // 如果 data 不为空
        if (data) {
            // 如果 fallback 为 true，则使用 free 函数释放内存
            if (fallback) { // NOLINT
                free(data);
            } else {
                // 否则使用 llama_host_free 函数释放内存
                llama_host_free(data);
            }
        }
        // 将 data 设为 NULL
        data = NULL;
    }
// 结构体定义，用于表示文件信息
struct llama_file {
    // 使用 FILE * 类型的指针，以便于使用 mmap 函数
    FILE * fp;
    // 文件名
    std::string fname;
    // 文件大小
    size_t size;

    // 构造函数，打开文件并获取文件大小
    llama_file(const char * fname, const char * mode): fname(fname) {
        // 打开文件
        fp = std::fopen(fname, mode);
        // 如果打开失败，抛出运行时错误
        if (fp == NULL) {
            throw std::runtime_error(format("failed to open %s: %s", fname, strerror(errno)));
        }
        // 定位到文件末尾，获取文件大小
        seek(0, SEEK_END);
        size = tell();
        // 定位回文件开头
        seek(0, SEEK_SET);
    }

    // 获取当前文件指针位置
    size_t tell() const {
#ifdef _WIN32
    // 获取当前文件指针位置
    __int64 ret = _ftelli64(fp);
#else
    long ret = std::ftell(fp);
#endif
    // 断言文件指针位置不为-1，表示获取成功
    GGML_ASSERT(ret != -1); // this really shouldn't fail
    // 返回文件指针位置
    return (size_t) ret;
}

// 设置文件指针位置
void seek(size_t offset, int whence) const {
#ifdef _WIN32
    int ret = _fseeki64(fp, (__int64) offset, whence);
#else
    int ret = std::fseek(fp, (long) offset, whence);
#endif
    // 断言设置文件指针位置成功
    GGML_ASSERT(ret == 0); // same
}

// 读取原始数据
void read_raw(void * ptr, size_t len) const {
    // 如果长度为0，直接返回
    if (len == 0) {
        return;
        }
        // 重置错误码
        errno = 0;
        // 从文件中读取数据到指定的内存块中
        std::size_t ret = std::fread(ptr, len, 1, fp);
        // 检查文件流是否出错
        if (ferror(fp)) {
            // 如果出错，抛出运行时错误，包含错误信息
            throw std::runtime_error(format("read error: %s", strerror(errno)));
        }
        // 检查读取的数据长度是否符合预期
        if (ret != 1) {
            // 如果不符合预期，抛出运行时错误，指示文件已经到达结尾
            throw std::runtime_error(std::string("unexpectedly reached end of file"));
        }
    }

    // 读取 32 位无符号整数
    uint32_t read_u32() const {
        uint32_t ret;
        // 读取原始数据
        read_raw(&ret, sizeof(ret));
        return ret;
    }

    // 写入原始数据到文件
    void write_raw(const void * ptr, size_t len) const {
        // 如果长度为 0，直接返回
        if (len == 0) {
            return;
        }
        // 重置错误码
        errno = 0;
        // 将数据写入文件
        size_t ret = std::fwrite(ptr, len, 1, fp);
        // 检查写入是否成功，如果不成功则抛出运行时错误
        if (ret != 1) {
            throw std::runtime_error(format("write error: %s", strerror(errno)));
        }
    }

    // 写入无符号32位整数
    void write_u32(std::uint32_t val) const {
        write_raw(&val, sizeof(val));
    }

    // 析构函数，关闭文件指针
    ~llama_file() {
        if (fp) {
            std::fclose(fp);
        }
    }
};

// llama_mmap 结构体定义
struct llama_mmap {
    // 声明指针变量 addr 和 size_t 类型的变量 size
    void * addr;
    size_t size;

    // 禁用拷贝构造函数
    llama_mmap(const llama_mmap &) = delete;

    // 如果系统支持映射文件，则设置 SUPPORTED 为 true
    #ifdef _POSIX_MAPPED_FILES
    static constexpr bool SUPPORTED = true;

    // 构造函数，接受 llama_file 结构体指针和预取大小作为参数
    llama_mmap(struct llama_file * file, size_t prefetch = (size_t) -1 /* -1 = max value */, bool numa = false) {
        // 设置 size 为文件大小
        size = file->size;
        // 获取文件描述符
        int fd = fileno(file->fp);
        // 设置映射标志为 MAP_SHARED
        int flags = MAP_SHARED;
        // 如果在 NUMA 系统上，预取会影响性能，则将预取设置为 0
        if (numa) { prefetch = 0; }
        // 在 Linux 系统上，如果预取大小不为 0，则设置映射标志为 MAP_POPULATE
        #ifdef __linux__
        if (prefetch) { flags |= MAP_POPULATE; }
        #endif
        // 调用 mmap 函数，将文件映射到内存中
        addr = mmap(NULL, file->size, PROT_READ, flags, fd, 0);
        // 如果映射失败，则抛出运行时异常
        if (addr == MAP_FAILED) {
            throw std::runtime_error(format("mmap failed: %s", strerror(errno)));
        }

        if (prefetch > 0) {
            // 如果预取值大于0，向内核建议预加载映射的内存
            if (posix_madvise(addr, std::min(file->size, prefetch), POSIX_MADV_WILLNEED)) {
                fprintf(stderr, "warning: posix_madvise(.., POSIX_MADV_WILLNEED) failed: %s\n",
                        strerror(errno));
            }
        }
        if (numa) {
            // 向内核建议不要使用预读取
            // （因为下一个页面可能不属于同一个节点）
            if (posix_madvise(addr, file->size, POSIX_MADV_RANDOM)) {
                fprintf(stderr, "warning: posix_madvise(.., POSIX_MADV_RANDOM) failed: %s\n",
                        strerror(errno));
            }
        }
    }

    ~llama_mmap() {
```

        // 如果是在 Linux 平台下
        munmap(addr, size);
    }
// 如果是在 Windows 平台下
#elif defined(_WIN32)
    // 定义一个常量表达式，表示在 Windows 平台下支持 mmap
    static constexpr bool SUPPORTED = true;

    // 在 Windows 平台下使用 mmap 函数
    llama_mmap(struct llama_file * file, bool prefetch = true, bool numa = false) {
        // 忽略 numa 参数

        // 获取文件大小
        size = file->size;

        // 获取文件句柄
        HANDLE hFile = (HANDLE) _get_osfhandle(_fileno(file->fp));

        // 创建文件映射
        HANDLE hMapping = CreateFileMappingA(hFile, NULL, PAGE_READONLY, 0, 0, NULL);
        // 获取错误信息
        DWORD error = GetLastError();

        // 如果创建文件映射失败，抛出运行时错误
        if (hMapping == NULL) {
            throw std::runtime_error(format("CreateFileMappingA failed: %s", llama_format_win_err(error).c_str()));
        }

        // 将文件映射到进程的地址空间
        addr = MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);
        # 获取最近一次发生的错误代码
        error = GetLastError();
        # 关闭内存映射句柄
        CloseHandle(hMapping);

        # 如果映射地址为空，抛出运行时错误
        if (addr == NULL) {
            throw std::runtime_error(format("MapViewOfFile failed: %s", llama_format_win_err(error).c_str()));
        }

        # 如果需要预取数据
        if (prefetch) {
            # 动态加载 PrefetchVirtualMemory 函数，该函数仅在 Windows 8 及以上版本中存在
            BOOL (WINAPI *pPrefetchVirtualMemory) (HANDLE, ULONG_PTR, PWIN32_MEMORY_RANGE_ENTRY, ULONG);
            HMODULE hKernel32 = GetModuleHandleW(L"kernel32.dll");

            # 尝试获取 PrefetchVirtualMemory 函数的地址，可能在 Windows 8 以下的系统中失败
            pPrefetchVirtualMemory = reinterpret_cast<decltype(pPrefetchVirtualMemory)> (GetProcAddress(hKernel32, "PrefetchVirtualMemory"));

            # 如果成功获取到 PrefetchVirtualMemory 函数的地址
            if (pPrefetchVirtualMemory) {
                # 告知内核预加载映射的内存
                WIN32_MEMORY_RANGE_ENTRY range;
                range.VirtualAddress = addr;
                range.NumberOfBytes = (SIZE_T)size;
// 如果操作系统支持预取虚拟内存，则进行预取操作
if (!pPrefetchVirtualMemory(GetCurrentProcess(), 1, &range, 0)) {
    // 如果预取虚拟内存操作失败，输出警告信息
    fprintf(stderr, "warning: PrefetchVirtualMemory failed: %s\n",
            llama_format_win_err(GetLastError()).c_str());
}
// 如果操作系统不支持预取虚拟内存，则不进行任何操作
}

// 析构函数，用于释放内存映射
~llama_mmap() {
    // 如果解除内存映射失败，输出警告信息
    if (!UnmapViewOfFile(addr)) {
        fprintf(stderr, "warning: UnmapViewOfFile failed: %s\n",
                llama_format_win_err(GetLastError()).c_str());
    }
}
#else
// 如果操作系统不支持内存映射，则设置SUPPORTED为false
static constexpr bool SUPPORTED = false;

// 构造函数，用于创建内存映射
llama_mmap(struct llama_file * file, bool prefetch = true, bool numa = false) {
    // 忽略参数file和prefetch
    (void) file;
    (void) prefetch;
// 声明一个名为numa的void指针，但未使用
(void) numa;

// 抛出一个运行时错误，说明mmap不被支持
throw std::runtime_error(std::string("mmap not supported"));
}

// 表示使用mlock或VirtualLock锁定的内存区域；在销毁时将自动解锁
struct llama_mlock {
    void * addr = NULL; // 初始化地址为NULL
    size_t size = 0; // 初始化大小为0

    bool failed_already = false; // 初始化失败标志为false

    llama_mlock() {} // 默认构造函数
    llama_mlock(const llama_mlock &) = delete; // 禁用拷贝构造函数

    ~llama_mlock() { // 析构函数
        if (size) { // 如果大小不为0
    // 解锁指定地址和大小的内存块
    raw_unlock(addr, size);
}

void init(void * ptr) {
    // 确保地址和大小都为0
    GGML_ASSERT(addr == NULL && size == 0); // NOLINT
    // 初始化地址
    addr = ptr;
}

void grow_to(size_t target_size) {
    // 确保地址不为空
    GGML_ASSERT(addr);
    // 如果已经失败，则直接返回
    if (failed_already) {
        return;
    }
    // 获取内存锁定的粒度
    size_t granularity = lock_granularity();
    // 将目标大小调整为粒度的整数倍
    target_size = (target_size + granularity - 1) & ~(granularity - 1);
    // 如果目标大小大于当前大小
    if (target_size > size) {
        // 尝试锁定内存块
        if (raw_lock((uint8_t *) addr + size, target_size - size)) {
            size = target_size;
        } else {
// 如果已经失败过，则设置 failed_already 为 true
failed_already = true;
// 结束 if 语句块
}
// 结束 while 语句块
}
// 如果定义了 _POSIX_MEMLOCK_RANGE，则设置 SUPPORTED 为 true
static constexpr bool SUPPORTED = true;
// 返回内存锁定的粒度
static size_t lock_granularity() {
    return (size_t) sysconf(_SC_PAGESIZE);
}
// 如果是苹果系统，则设置 MLOCK_SUGGESTION 为相应建议，否则设置为另一种建议
#ifdef __APPLE__
    #define MLOCK_SUGGESTION \
        "Try increasing the sysctl values 'vm.user_wire_limit' and 'vm.global_user_wire_limit' and/or " \
        "decreasing 'vm.global_no_user_wire_amount'.  Also try increasing RLIMIT_MLOCK (ulimit -l).\n"
#else
    #define MLOCK_SUGGESTION \
        "Try increasing RLIMIT_MLOCK ('ulimit -l' as root).\n"
#endif
    // 使用mlock函数锁定指定地址的内存区域，防止被交换出去
    bool raw_lock(const void * addr, size_t size) const {
        // 如果mlock函数执行成功，则返回true
        if (!mlock(addr, size)) {
            return true;
        }

        // 获取错误信息
        char* errmsg = std::strerror(errno);
        // 检查是否建议使用其他方法，比如增加内存限制
        bool suggest = (errno == ENOMEM);

        // 检查资源限制是否正常
        struct rlimit lock_limit;
        // 如果建议使用其他方法，并且获取内存锁定的资源限制成功
        if (suggest && getrlimit(RLIMIT_MEMLOCK, &lock_limit)) {
            suggest = false;
        }
        // 如果建议使用其他方法，并且内存锁定的资源限制大于当前已锁定的内存大小加上要锁定的内存大小
        if (suggest && (lock_limit.rlim_max > lock_limit.rlim_cur + size)) {
            suggest = false;
        }

        // 输出警告信息，包括锁定的内存大小、之前已锁定的内存大小、错误信息和建议
        fprintf(stderr, "warning: failed to mlock %zu-byte buffer (after previously locking %zu bytes): %s\n%s",
                size, this->size, errmsg, suggest ? MLOCK_SUGGESTION : "");
    # 返回 false，表示未支持原始锁定操作
    return false;
    # 取消定义 MLOCK_SUGGESTION

    # 如果是 Windows 系统
    # 定义支持原始锁定操作
    static constexpr bool SUPPORTED = true;

    # 获取系统信息，返回页面大小作为锁定粒度
    static size_t lock_granularity() {
        SYSTEM_INFO si;
        GetSystemInfo(&si);
        return (size_t) si.dwPageSize;
    }

    # 对指定内存区域进行原始锁定
    bool raw_lock(void * ptr, size_t len) const {
// 设置循环次数变量，无限循环直到条件满足
for (int tries = 1; ; tries++) {
    // 尝试将指针指向的内存锁定，如果成功则返回true
    if (VirtualLock(ptr, len)) {
        return true;
    }
    // 如果尝试次数为2，则输出警告信息并返回false
    if (tries == 2) {
        fprintf(stderr, "warning: failed to VirtualLock %zu-byte buffer (after previously locking %zu bytes): %s\n",
            len, size, llama_format_win_err(GetLastError()).c_str());
        return false;
    }

    // 锁定失败，但这只是第一次尝试；增加工作集大小并重试
    SIZE_T min_ws_size, max_ws_size;
    // 获取当前进程的工作集大小
    if (!GetProcessWorkingSetSize(GetCurrentProcess(), &min_ws_size, &max_ws_size)) {
        fprintf(stderr, "warning: GetProcessWorkingSetSize failed: %s\n",
                llama_format_win_err(GetLastError()).c_str());
        return false;
    }
    // 根据MSDN文档："进程可以锁定的最大页面数等于其最小工作集中的页面数减去
    // 为了避免小的开销，增加一定的内存空间
    // 希望增加 1MB 的内存空间足够了
    size_t increment = len + 1048576;
    // 最小值必须小于等于最大值，因此需要同时增加两者的值
    min_ws_size += increment;
    max_ws_size += increment;
    // 如果设置进程的工作集大小失败，则输出警告信息并返回 false
    if (!SetProcessWorkingSetSize(GetCurrentProcess(), min_ws_size, max_ws_size)) {
        fprintf(stderr, "warning: SetProcessWorkingSetSize failed: %s\n",
                llama_format_win_err(GetLastError()).c_str());
        return false;
    }
}

// 释放已锁定的内存空间
static void raw_unlock(void * ptr, size_t len) {
    // 如果释放已锁定的内存空间失败，则输出警告信息
    if (!VirtualUnlock(ptr, len)) {
        fprintf(stderr, "warning: failed to VirtualUnlock buffer: %s\n",
                llama_format_win_err(GetLastError()).c_str());
    }
}
// 如果不满足条件，则设置SUPPORTED为false
static constexpr bool SUPPORTED = false;

// 返回锁定粒度
static size_t lock_granularity() {
    return (size_t) 65536;
}

// 锁定指定地址和长度的内存区域
bool raw_lock(const void * addr, size_t len) const {
    // 输出警告信息，表示在该系统上不支持mlock操作
    fprintf(stderr, "warning: mlock not supported on this system\n");
    return false;
}

// 解锁指定地址和长度的内存区域
static void raw_unlock(const void * addr, size_t len) {}
#endif
};

// 定义一个指向ggml_tensor结构体的函数指针类型
typedef void (*offload_func_t)(struct ggml_tensor * tensor);

// 定义一个空的函数，用于卸载ggml_tensor结构体
static void ggml_offload_nop(struct ggml_tensor * tensor) {
    (void) tensor;
}
// 将 llama_token 转换为对应的字符串片段
static std::string llama_token_to_piece(const struct llama_context * ctx, llama_token token) {
    // 创建一个包含8个元素的字符向量，并初始化为0
    std::vector<char> result(8, 0);
    // 调用 llama_token_to_piece 函数将 token 转换为字符串片段，并将结果存储在 result 中
    const int n_tokens = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
    // 如果 n_tokens 小于0，说明结果不符合预期，重新调整 result 的大小并再次调用 llama_token_to_piece 函数
    if (n_tokens < 0) {
        result.resize(-n_tokens);
        int check = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
        // 断言 check 等于 -n_tokens
        GGML_ASSERT(check == -n_tokens);
    }
    // 如果 n_tokens 大于等于0，说明结果符合预期，调整 result 的大小
    else {
        result.resize(n_tokens);
    }

    // 将 result 转换为字符串并返回
    return std::string(result.data(), result.size());
}

//
// 全局变量
//
// 定义了一个名为llama_state的结构体
struct llama_state {
    // 保存日志回调函数的全局变量
    ggml_log_callback log_callback = llama_log_callback_default;
    // 保存日志回调函数的用户数据的全局变量
    void * log_callback_user_data = nullptr;
};

// 创建了一个名为g_state的静态llama_state对象
static llama_state g_state;

// 可用的llama模型枚举
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
// 定义模型的三种不同版本
MODEL_40B,
MODEL_65B,
MODEL_70B,
};

// 定义常量，表示字节、千字节和兆字节
static const size_t kB = 1024;
static const size_t MB = 1024*kB;
static const size_t GB = 1024*MB;

// 定义 llama_hparams 结构体，包含模型的超参数信息
struct llama_hparams {
    bool     vocab_only; // 是否只包含词汇表
    uint32_t n_vocab; // 词汇表大小
    uint32_t n_ctx_train; // 模型训练时的上下文大小
    uint32_t n_embd; // 嵌入维度
    uint32_t n_head; // 多头注意力的头数
    uint32_t n_head_kv; // 多头注意力中键值的头数
    uint32_t n_layer; // 层数
    uint32_t n_rot; // 旋转数
    uint32_t n_ff; // 前馈网络的隐藏层大小
    // 定义两个浮点型变量
    float f_norm_eps;
    float f_norm_rms_eps;

    // 定义两个浮点型变量和一个无符号整型变量，以及一个有符号整型变量和一个布尔型变量
    float    rope_freq_base_train;
    float    rope_freq_scale_train;
    uint32_t n_yarn_orig_ctx;
    int8_t   rope_scaling_type_train : 3;
    bool     rope_finetuned : 1;

    // 定义两个浮点型变量
    float f_clamp_kqv;
    float f_max_alibi_bias;
    
    // 如果启用了稀疏推断，则设置稀疏预测阈值为环境变量LLAMA_SPARSE_PRED_THRESHOLD的值，如果没有设置则默认为0.0
    float sparse_pred_threshold = atof(getenv("LLAMA_SPARSE_PRED_THRESHOLD") ?: "0.0");

    // 重载不等于运算符，比较当前对象和另一个llama_hparams对象的成员变量是否不相等
    bool operator!=(const llama_hparams & other) const {
        // 如果成员变量vocab_only不相等，则返回true
        if (this->vocab_only  != other.vocab_only)  return true;
        // 如果成员变量n_vocab不相等，则返回true
        if (this->n_vocab     != other.n_vocab)     return true;
        // 如果成员变量n_ctx_train不相等，则返回true
        if (this->n_ctx_train != other.n_ctx_train) return true;
        // 如果成员变量n_embd不相等，则返回true
        if (this->n_embd      != other.n_embd)      return true;
        // 检查对象的属性是否相等，如果不相等则返回 true
        if (this->n_head      != other.n_head)      return true;
        if (this->n_head_kv   != other.n_head_kv)   return true;
        if (this->n_layer     != other.n_layer)     return true;
        if (this->n_rot       != other.n_rot)       return true;
        if (this->n_ff        != other.n_ff)        return true;
        if (this->rope_finetuned  != other.rope_finetuned)  return true;
        if (this->n_yarn_orig_ctx != other.n_yarn_orig_ctx) return true;

        // 定义浮点数比较的精度
        const float EPSILON = 1e-9;

        // 检查浮点数是否在给定精度范围内相等，如果不相等则返回 true
        if (!is_float_close(this->f_norm_eps,            other.f_norm_eps,            EPSILON)) return true;
        if (!is_float_close(this->f_norm_rms_eps,        other.f_norm_rms_eps,        EPSILON)) return true;
        if (!is_float_close(this->rope_freq_base_train,  other.rope_freq_base_train,  EPSILON)) return true;
        if (!is_float_close(this->rope_freq_scale_train, other.rope_freq_scale_train, EPSILON)) return true;

        // 如果以上条件都不满足，则返回 false
        return false;
    }

    // 返回 n_head 除以 n_head_kv 的结果
    uint32_t n_gqa() const {
        return n_head/n_head_kv;
    }

    // 返回嵌入维度除以头数的结果
    uint32_t n_embd_head() const {
        return n_embd/n_head;
    }

    // 返回嵌入维度除以 GQA 数的结果
    uint32_t n_embd_gqa() const {
        return n_embd/n_gqa();
    }
};

// LLAMA 模型的超参数结构体
struct llama_cparams {
    uint32_t n_ctx;       // 推理过程中使用的上下文大小
    uint32_t n_batch;     // 批处理大小
    uint32_t n_threads;       // 用于生成的线程数
    uint32_t n_threads_batch; // 用于批处理的线程数

    float    rope_freq_base;  // 绳索频率基数
    float    rope_freq_scale; // 绳索频率比例
    uint32_t n_yarn_orig_ctx;
    // 原始 YaRN 上下文的数量

    // 这些超参数在 GGUF 中没有暴露，因为所有现有的 YaRN 模型都使用相同的值。
    float yarn_ext_factor;
    // YaRN 扩展因子
    float yarn_attn_factor;
    // YaRN 注意力因子
    float yarn_beta_fast;
    // YaRN 快速学习率
    float yarn_beta_slow;
    // YaRN 慢速学习率

    bool mul_mat_q;
    // 是否对 Q 矩阵进行乘法运算

};

struct llama_layer {
    // 归一化
    struct ggml_tensor * attn_norm;
    // 注意力归一化
    struct ggml_tensor * attn_norm_b;
    // 注意力归一化偏置
    struct ggml_tensor * attn_norm_2;
    // 注意力归一化2
    struct ggml_tensor * attn_norm_2_b;
    // 注意力归一化2偏置
    struct ggml_tensor * attn_q_norm;
    // 注意力 Q 归一化
    struct ggml_tensor * attn_q_norm_b;
    // 注意力 Q 归一化偏置
    struct ggml_tensor * attn_k_norm;
    // 注意力 K 归一化
```

    // 定义指向 ggml_tensor 结构体的指针 attn_k_norm_b

    // 定义指向 ggml_tensor 结构体的指针，用于存储注意力机制中的查询向量
    struct ggml_tensor * wq;
    // 定义指向 ggml_tensor 结构体的指针，用于存储注意力机制中的键向量
    struct ggml_tensor * wk;
    // 定义指向 ggml_tensor 结构体的指针，用于存储注意力机制中的值向量
    struct ggml_tensor * wv;
    // 定义指向 ggml_tensor 结构体的指针，用于存储注意力机制中的输出向量
    struct ggml_tensor * wo;
    // 定义指向 ggml_tensor 结构体的指针，用于存储注意力机制中的查询、键、值向量的组合
    struct ggml_tensor * wqkv;

    // 定义指向 ggml_tensor 结构体的指针，用于存储注意力机制中的偏置项
    struct ggml_tensor * bo;
    // 定义指向 ggml_tensor 结构体的指针，用于存储注意力机制中的查询、键、值向量的组合的偏置项
    struct ggml_tensor * bqkv;

    // 定义指向 ggml_tensor 结构体的指针，用于存储归一化后的注意力机制输出
    struct ggml_tensor * ffn_norm;
    // 定义指向 ggml_tensor 结构体的指针，用于存储归一化后的注意力机制输出的偏置项
    struct ggml_tensor * ffn_norm_b;

    // 定义指向 ggml_tensor 结构体的指针，用于存储前馈神经网络中的门控单元
    struct ggml_tensor * ffn_gate; // w1
    // 定义指向 ggml_tensor 结构体的指针，用于存储前馈神经网络中的下采样权重
    struct ggml_tensor * ffn_down; // w2
    struct ggml_tensor * ffn_up;   // 定义指向 ggml_tensor 结构体的指针变量，用于表示上行传输的张量
    struct ggml_tensor * ffn_down_t;  // 定义指向 ggml_tensor 结构体的指针变量，用于表示下行传输的张量
    
    // 在 GPU 上切片的 ff
    struct ggml_tensor * ffn_gate_gpu;  // 定义指向 ggml_tensor 结构体的指针变量，用于表示在 GPU 上切片的 ff
    struct ggml_tensor * ffn_down_gpu;  // 定义指向 ggml_tensor 结构体的指针变量，用于表示在 GPU 上切片的下行 ff
    struct ggml_tensor * ffn_up_gpu;  // 定义指向 ggml_tensor 结构体的指针变量，用于表示在 GPU 上切片的上行 ff

    // ff 偏置
    struct ggml_tensor * ffn_down_b; // 定义指向 ggml_tensor 结构体的指针变量，用于表示 ff 的下行偏置
    struct ggml_tensor * ffn_up_b;   // 定义指向 ggml_tensor 结构体的指针变量，用于表示 ff 的上行偏置

    // mlp 预测器权重
    struct ggml_tensor * mlp_pre_w1;  // 定义指向 ggml_tensor 结构体的指针变量，用于表示 mlp 预测器的权重1
    struct ggml_tensor * mlp_pre_w2;  // 定义指向 ggml_tensor 结构体的指针变量，用于表示 mlp 预测器的权重2

    // GPU 双索引
    // TODO: 需要为所有层填充这个
    struct ggml_tensor * gpu_idx;  // 定义指向 ggml_tensor 结构体的指针变量，用于表示 GPU 的双索引
    struct ggml_tensor * gpu_bucket;  // 定义指向 ggml_tensor 结构体的指针变量，用于表示 GPU 的桶
// 结构体定义，用于存储键值对的位置、偏移量和序列号集合
struct llama_kv_cell {
    llama_pos pos   = -1; // 键值对的位置，默认为-1
    llama_pos delta = 0; // 键值对的偏移量，默认为0

    std::set<llama_seq_id> seq_id; // 存储键值对的序列号集合

    // 检查序列号集合中是否存在特定的序列号
    bool has_seq_id(const llama_seq_id & id) const {
        return seq_id.find(id) != seq_id.end();
    }
};

// 用于缓存键值对数据的环形缓冲区
struct llama_kv_cache {
    bool has_shift = false; // 标记是否发生了移位操作

    // 注意：head的值不仅用于优化搜索空闲的KV槽位，llama_decode_internal也使用它，
    // 因此在分配了槽位后不能随意更改它
    // 定义一个32位无符号整数变量head，并初始化为0
    uint32_t head = 0;
    // 定义一个32位无符号整数变量size，并初始化为0
    uint32_t size = 0;

    // 在每次构建图之前计算的变量n，初始化为0
    uint32_t n = 0;

    // 定义一个llama_kv_cell类型的向量cells
    std::vector<llama_kv_cell> cells;

    // 定义一个指向ggml_tensor结构体的指针k，并初始化为NULL
    struct ggml_tensor * k = NULL;
    // 定义一个指向ggml_tensor结构体的指针v，并初始化为NULL
    struct ggml_tensor * v = NULL;

    // 定义一个指向ggml_context结构体的指针ctx，并初始化为NULL
    struct ggml_context * ctx = NULL;

    // 定义一个llama_buffer类型的变量buf
    llama_buffer buf;

    // 析构函数，用于释放内存
    ~llama_kv_cache() {
        // 如果ctx指针不为空，则调用ggml_free函数释放内存
        if (ctx) {
            ggml_free(ctx);
        }
    }
#ifdef GGML_USE_CUBLAS
        // 如果定义了 GGML_USE_CUBLAS 宏
        if (ggml_cublas_loaded()) {
            // 如果 CUDA BLAS 已加载
            ggml_cuda_free_data(k);
            // 释放 k 的 CUDA 数据
            ggml_cuda_free_data(v);
            // 释放 v 的 CUDA 数据
        }
#endif
    }
};

struct llama_vocab {
    // 定义 llama_vocab 结构体
    using id    = int32_t;
    // 使用 int32_t 类型定义 id
    using token = std::string;
    // 使用 std::string 类型定义 token
    using ttype = llama_token_type;
    // 使用 llama_token_type 类型定义 ttype

    struct token_data {
        // 定义 token_data 结构体
        token text;
        // 定义 text 字段，类型为 token
        float score;
        // 定义 score 字段，类型为 float
        ttype type;
        // 定义 type 字段，类型为 ttype
    };
    // 定义枚举类型 llama_vocab_type，并赋值为 LLAMA_VOCAB_TYPE_SPM
    enum llama_vocab_type type = LLAMA_VOCAB_TYPE_SPM;

    // 创建一个从 token 到 id 的无序映射
    std::unordered_map<token, id> token_to_id;
    // 创建一个存储 token_data 的向量
    std::vector<token_data>       id_to_token;

    // 创建一个从 token 到 id 的无序映射，用于缓存特殊 token
    std::unordered_map<token, id> special_tokens_cache;

    // 创建一个从包含两个字符串的 pair 到整数的映射，用于存储 BPE（字节对编码）的排名
    std::map<std::pair<std::string, std::string>, int> bpe_ranks;

    // 默认的 LLaMA 特殊 token
    id special_bos_id = 1;  // 开始 token 的 id
    id special_eos_id = 2;  // 结束 token 的 id
    id special_unk_id = 0;  // 未知 token 的 id
    id special_sep_id = -1; // 分隔 token 的 id
    id special_pad_id = -1; // 填充 token 的 id

    id linefeed_id       = 13;    // 换行符的 id
    id special_prefix_id = 32007; // 特殊前缀的 id
    id special_middle_id = 32009; // 特殊中缀的 id
    id special_suffix_id = 32008; // 特殊后缀的 id
// 定义一个整型变量 special_eot_id 并赋值为 32010
id special_eot_id = 32010;

// 定义一个函数 find_bpe_rank，用于查找 BPE（字节对编码）的排名
int find_bpe_rank(std::string token_left, std::string token_right) const {
    // 断言 token_left 中不包含空格和换行符
    GGML_ASSERT(token_left.find(" ") == std::string::npos);
    GGML_ASSERT(token_left.find("\n") == std::string::npos);
    // 断言 token_right 中不包含空格和换行符
    GGML_ASSERT(token_right.find(" ") == std::string::npos);
    GGML_ASSERT(token_right.find("\n") == std::string::npos);

    // 在 bpe_ranks 中查找 token_left 和 token_right 组成的键值对
    auto it = bpe_ranks.find(std::make_pair(token_left, token_right));
    // 如果找不到，则返回 -1
    if (it == bpe_ranks.end()) {
        return -1;
    }

    // 返回找到的排名
    return it->second;
}

// 定义结构体 llama_gpu_split_loader
struct llama_gpu_split_loader;

// 定义结构体 llama_augmentation_model_loader
struct llama_augmentation_model_loader;
# 定义了一个名为 llama_model 的结构体，用于存储模型相关的信息
struct llama_model {
    # 模型类型，默认为 MODEL_UNKNOWN
    e_model     type  = MODEL_UNKNOWN;
    # 模型架构，默认为 LLM_ARCH_UNKNOWN
    llm_arch    arch  = LLM_ARCH_UNKNOWN;
    # 模型文件类型，默认为 LLAMA_FTYPE_ALL_F32
    llama_ftype ftype = LLAMA_FTYPE_ALL_F32;

    # 模型名称，默认为 "n/a"
    std::string name = "n/a";

    # 存储稀疏导数相关信息的结构体
    ggml_sparse_deriv sparse_deriv;

    # 模型超参数，默认为空
    llama_hparams hparams = {};
    # 模型词汇表
    llama_vocab   vocab;

    # 指向 token embedding 的指针
    struct ggml_tensor * tok_embd;
    # 指向 position embedding 的指针
    struct ggml_tensor * pos_embd;
    # 指向 token normalization 的指针
    struct ggml_tensor * tok_norm;
    # 指向 token normalization 偏置的指针
    struct ggml_tensor * tok_norm_b;

    # 指向输出 normalization 的指针
    struct ggml_tensor * output_norm;
    # 指向输出 normalization 偏置的指针
    struct ggml_tensor * output_norm_b;
    # 指向输出的指针
    struct ggml_tensor * output;
}
// 创建一个存储 llama_layer 对象的向量
std::vector<llama_layer> layers;

// 存储 GPU 层的数量
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
    // 创建一个名为mlock_mmap的llama_mlock对象
    llama_mlock mlock_mmap;

    // 用于存储名称和对应的ggml_tensor指针的向量，仅用于quantize-stats
    std::vector<std::pair<std::string, struct ggml_tensor *>> tensors_by_name;

    // 用于存储加载时间的微秒数
    int64_t t_load_us = 0;
    // 用于存储开始时间的微秒数
    int64_t t_start_us = 0;

    // 析构函数，用于释放内存
    ~llama_model() {
        // 如果上下文存在，则释放上下文
        if (ctx) {
            ggml_free(ctx);
        }

        // 如果使用了CUBLAS，则释放相关资源
#ifdef GGML_USE_CUBLAS
        // 遍历tensors_by_name向量，释放对应的CUDA内存
        if (ggml_cublas_loaded()) {
            for (size_t i = 0; i < tensors_by_name.size(); ++i) {
                ggml_cuda_free_data(tensors_by_name[i].second);
            }
            // 释放CUDA的临时内存
            ggml_cuda_free_scratch();
        }
// 结束预处理指令
#endif

// 如果定义了 GGML_USE_CLBLAST，则释放张量数据
#if defined(GGML_USE_CLBLAST)
        for (size_t i = 0; i < tensors_by_name.size(); ++i) {
            ggml_cl_free_data(tensors_by_name[i].second);
        }
#endif
    }
};

// 定义 llama_context 结构体，初始化 t_start_us 和 t_load_us
struct llama_context {
    llama_context(const llama_model & model) : model(model), t_start_us(model.t_start_us), t_load_us(model.t_load_us) {}
    // 析构函数
    ~llama_context() {
#ifdef GGML_USE_METAL
        // 如果定义了 GGML_USE_METAL 并且 ctx_metal 存在，则释放 ctx_metal
        if (ctx_metal) {
            ggml_metal_free(ctx_metal);
        }
#endif
        // 如果 alloc 存在，则释放 alloc
        if (alloc) {
            ggml_allocr_free(alloc);
    }
    }

    // 定义名为 cparams 的 llama_cparams 结构体
    llama_cparams cparams;

    // 定义名为 model 的常量 llama_model 的引用
    const llama_model & model;

    // 用于自注意力的键值对缓存
    struct llama_kv_cache kv_self;

    // 用于生成随机数的 Mersenne Twister 伪随机数生成器
    std::mt19937 rng;

    // 标记是否已经评估过至少一次
    bool has_evaluated_once = false;

    // 记录开始时间的微秒数
    int64_t t_start_us;

    // 记录加载时间的微秒数
    int64_t t_load_us;

    // 记录采样时间的微秒数，默认为 0
    int64_t t_sample_us = 0;

    // 记录预测评估时间的微秒数，默认为 0
    int64_t t_p_eval_us = 0;

    // 记录评估时间的微秒数，默认为 0
    int64_t t_eval_us   = 0;
    int32_t n_sample = 0; // 初始化为0，用于记录采样的标记数量
    int32_t n_p_eval = 0; // 初始化为0，用于记录在提示的评估调用中的标记数量（批量大小大于1）
    int32_t n_eval   = 0; // 初始化为0，用于记录评估调用的数量

    // 解码输出（2维数组：[n_tokens][n_vocab]）
    std::vector<float> logits; // 用于存储解码输出的概率分布
    bool logits_all = false; // 初始化为false，用于标记是否已经获取了所有的logits

    // 输入嵌入（1维数组：[n_embd]）
    std::vector<float> embedding; // 用于存储输入的嵌入向量

    // 用于`struct ggml_graph_plan.work_data`的可重用缓冲区
    std::vector<uint8_t> work_buffer; // 用于存储工作数据的缓冲区

    // 用于评估模型的内存缓冲区
    llama_buffer buf_compute; // 用于存储计算的缓冲区

    llama_buffer buf_alloc; // 用于分配内存的缓冲区
    ggml_allocr * alloc = NULL; // 初始化为NULL，用于分配内存的指针
#ifdef GGML_USE_METAL
    // 如果定义了 GGML_USE_METAL 宏，则创建 Metal 上下文对象
    ggml_metal_context * ctx_metal = NULL;
#endif

#ifdef GGML_USE_MPI
    // 如果定义了 GGML_USE_MPI 宏，则创建 MPI 上下文对象
    ggml_mpi_context * ctx_mpi = NULL;
#endif
};

//
// kv cache helpers
//

static bool llama_kv_cache_init(
        // 初始化键值缓存
        const struct llama_hparams & hparams,
             struct llama_kv_cache & cache,
                         ggml_type   wtype,
                          uint32_t   n_ctx,
                               int   n_gpu_layers) {
    // 获取参数中的 n_embd 值
    const uint32_t n_embd  = hparams.n_embd_gqa();
// 定义一个无符号32位整数变量n_layer，其值为hparams.n_layer
const uint32_t n_layer = hparams.n_layer;

// 定义一个64位整数变量n_mem，其值为n_layer乘以n_ctx
const int64_t n_mem      = n_layer*n_ctx;

// 定义一个64位整数变量n_elements，其值为n_embd乘以n_mem
const int64_t n_elements = n_embd*n_mem;

// 将cache.has_shift设置为false
cache.has_shift = false;

// 将cache.head设置为0
cache.head = 0;

// 将cache.size设置为n_ctx
cache.size = n_ctx;

// 清空cache.cells，并将其大小调整为n_ctx
cache.cells.clear();
cache.cells.resize(n_ctx);

// 调整cache.buf的大小为2倍n_elements乘以wtype的字节大小再加上2倍ggml_tensor_overhead()的大小
cache.buf.resize(2u*n_elements*ggml_type_size(wtype) + 2u*ggml_tensor_overhead());

// 将cache.buf的数据全部设置为0
memset(cache.buf.data, 0, cache.buf.size);

// 定义一个ggml_init_params结构体变量params
struct ggml_init_params params;

// 将params的mem_size设置为cache.buf的大小
params.mem_size   = cache.buf.size;

// 将params的mem_buffer设置为cache.buf的数据
params.mem_buffer = cache.buf.data;

// 将params的no_alloc设置为false
params.no_alloc   = false;
    # 初始化缓存上下文
    cache.ctx = ggml_init(params);

    # 如果缓存上下文初始化失败，记录错误信息并返回false
    if (!cache.ctx) {
        LLAMA_LOG_ERROR("%s: failed to allocate memory for kv cache\n", __func__);
        return false;
    }

    # 创建缓存的键和值的张量，并设置它们的名称
    cache.k = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);
    cache.v = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);
    ggml_set_name(cache.k, "cache_k");
    ggml_set_name(cache.v, "cache_v");

    # 忽略n_gpu_layers

    # 如果使用CUBLAS库，计算显存中缓存的大小
    if (ggml_cublas_loaded()) {
        size_t vram_kv_cache = 0;

        # 如果GPU层数大于层号加1，执行以下操作
        if (n_gpu_layers > (int)n_layer + 1) {
// 在 GPU 上分配缓冲区，不使用临时缓冲区
ggml_cuda_assign_buffers_no_scratch(cache.v);
// 打印信息，将 v 缓存卸载到 GPU
LLAMA_LOG_INFO("%s: offloading v cache to GPU\n", __func__);
// 计算 v 缓存占用的 VRAM 大小并累加
vram_kv_cache += ggml_nbytes(cache.v);

// 如果 GPU 层数大于当前层数加 2
if (n_gpu_layers > (int)n_layer + 2) {
    // 在 GPU 上分配缓冲区，不使用临时缓冲区
    ggml_cuda_assign_buffers_no_scratch(cache.k);
    // 打印信息，将 k 缓存卸载到 GPU
    LLAMA_LOG_INFO("%s: offloading k cache to GPU\n", __func__);
    // 计算 k 缓存占用的 VRAM 大小并累加
    vram_kv_cache += ggml_nbytes(cache.k);
}

// 如果 VRAM kv 缓存大小大于 0
if (vram_kv_cache > 0) {
    // 打印信息，显示 VRAM kv 自身大小
    LLAMA_LOG_INFO("%s: VRAM kv self = %.2f MB\n", __func__, vram_kv_cache / 1024.0 / 1024.0);
}

// 返回 true
return true;
// 注意：成功时，cache.head 指向槽的第一个单元格非常重要。
static bool llama_kv_cache_find_slot(
           struct llama_kv_cache & cache,
        const struct llama_batch & batch) {
    // 获取缓存大小和批处理中的标记数
    const uint32_t n_ctx    = cache.size;
    const uint32_t n_tokens = batch.n_tokens;

    // 如果批处理中的标记数大于缓存大小，记录错误并返回false
    if (n_tokens > n_ctx) {
        LLAMA_LOG_ERROR("%s: n_tokens=%d > n_ctx=%d\n", __func__, n_tokens, n_ctx);
        return false;
    }

    // 初始化已测试标记数
    uint32_t n_tested = 0;

    // 循环查找可用槽
    while (true) {
        // 如果当前槽不足以容纳批处理中的标记数，更新已测试标记数，重置当前槽的位置，继续查找
        if (cache.head + n_tokens > n_ctx) {
            n_tested += n_ctx - cache.head;
            cache.head = 0;
            continue;
        }

        // 初始化布尔变量 found 为 true
        bool found = true;
        // 遍历缓存中的 tokens
        for (uint32_t i = 0; i < n_tokens; i++) {
            // 如果缓存中的单元格位置大于等于 0
            if (cache.cells[cache.head + i].pos >= 0) {
                // 将 found 设为 false
                found = false;
                // 更新缓存头部位置
                cache.head += i + 1;
                // 更新测试数量
                n_tested   += i + 1;
                // 退出循环
                break;
            }
        }

        // 如果 found 为 true，则跳出循环
        if (found) {
            break;
        }

        // 如果测试数量大于等于上下文数量
        if (n_tested >= n_ctx) {
            // 输出错误日志并返回 false
            //LLAMA_LOG_ERROR("%s: failed to find a slot for %d tokens\n", __func__, n_tokens);
            return false;
        }
    }

    // 遍历每个 token
    for (uint32_t i = 0; i < n_tokens; i++) {
        // 将 batch 中的位置信息赋值给 cache 中对应的单元格
        cache.cells[cache.head + i].pos = batch.pos[i];

        // 遍历 batch 中的每个序列 ID
        for (int32_t j = 0; j < batch.n_seq_id[i]; j++) {
            // 将 batch 中的序列 ID 插入到 cache 中对应单元格的序列 ID 集合中
            cache.cells[cache.head + i].seq_id.insert(batch.seq_id[i][j]);
        }
    }

    // 返回 true，表示操作成功
    return true;
}

// 查找当前使用的单元格数量
static int32_t llama_kv_cache_cell_max(const struct llama_kv_cache & cache) {
    // 从 cache 的最后一个单元格开始向前遍历
    for (uint32_t i = cache.size - 1; i > 0; --i) {
        // 如果单元格的位置大于等于 0 并且序列 ID 集合不为空
        if (cache.cells[i].pos >= 0 && !cache.cells[i].seq_id.empty()) {
            // 返回当前单元格的索引加 1，表示当前使用的单元格数量
            return i + 1;
        }
    }
    // 返回值为0
    return 0;
}

// 清空缓存中的数据
static void llama_kv_cache_clear(struct llama_kv_cache & cache) {
    // 遍历缓存中的元素
    for (int32_t i = 0; i < (int32_t) cache.size; ++i) {
        // 将缓存中的位置设为-1
        cache.cells[i].pos = -1;
        // 清空缓存中的序列ID
        cache.cells[i].seq_id.clear();
    }
    // 将缓存头部位置设为0
    cache.head = 0;
}

// 从缓存中删除指定序列ID的数据
static void llama_kv_cache_seq_rm(
        struct llama_kv_cache & cache,
                 llama_seq_id   seq_id,
                    llama_pos   p0,
                    llama_pos   p1) {
    // 新的头部位置为缓存的大小
    uint32_t new_head = cache.size;

    // 如果p0小于0，则将其设为0
    if (p0 < 0) p0 = 0;
    // 如果 p1 小于 0，则将其赋值为 llama_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 遍历缓存中的元素
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果缓存中的位置在 p0 和 p1 之间
        if (cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 如果 seq_id 小于 0，则清空缓存中的序列 ID
            if (seq_id < 0) {
                cache.cells[i].seq_id.clear();
            } 
            // 如果缓存中的元素包含指定的序列 ID，则删除该序列 ID
            else if (cache.cells[i].has_seq_id(seq_id)) {
                cache.cells[i].seq_id.erase(seq_id);
            } 
            // 如果不包含指定的序列 ID，则继续下一次循环
            else {
                continue;
            }
            // 如果缓存中的序列 ID 为空，则将位置设为 -1，并更新 new_head
            if (cache.cells[i].seq_id.empty()) {
                cache.cells[i].pos = -1;
                if (new_head == cache.size) new_head = i;
            }
        }
    }

    // 如果释放了一个缓存槽位，则将头指针指向该槽位，以便下次搜索可以从该位置开始
    if (new_head != cache.size) cache.head = new_head;
// 将序列缓存中的数据从一个位置复制到另一个位置
static void llama_kv_cache_seq_cp(
        struct llama_kv_cache & cache,  // 序列缓存对象的引用
                 llama_seq_id   seq_id_src,  // 源序列 ID
                 llama_seq_id   seq_id_dst,  // 目标序列 ID
                    llama_pos   p0,  // 起始位置
                    llama_pos   p1) {  // 结束位置
    // 如果起始位置小于0，则将其设置为0
    if (p0 < 0) p0 = 0;
    // 如果结束位置小于0，则将其设置为最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 将缓存的头部位置设置为0
    cache.head = 0;

    // 遍历缓存中的数据
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果数据的序列 ID 与源序列 ID 相同，并且位置在指定范围内
        if (cache.cells[i].has_seq_id(seq_id_src) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 将数据的序列 ID 设置为目标序列 ID
            cache.cells[i].seq_id.insert(seq_id_dst);
        }
    }
}
// 保持序列号的键值缓存，更新缓存中的序列号
static void llama_kv_cache_seq_keep(struct llama_kv_cache & cache, llama_seq_id seq_id) {
    // 获取新的头部位置
    uint32_t new_head = cache.size;

    // 遍历缓存中的所有元素
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果缓存中的元素不包含给定的序列号
        if (!cache.cells[i].has_seq_id(seq_id)) {
            // 将该元素的位置设置为-1，清空序列号
            cache.cells[i].pos = -1;
            cache.cells[i].seq_id.clear();
            // 如果是第一个空闲位置，更新新的头部位置
            if (new_head == cache.size) new_head = i;
        } else {
            // 如果缓存中的元素包含给定的序列号，清空原有的序列号，插入新的序列号
            cache.cells[i].seq_id.clear();
            cache.cells[i].seq_id.insert(seq_id);
        }
    }

    // 如果释放了一个位置，将头部位置设置为新的空闲位置，以便下次搜索可以从这里开始
    if (new_head != cache.size) cache.head = new_head;
}

static void llama_kv_cache_seq_shift(
        struct llama_kv_cache & cache,
// 更新缓存中的位置信息
void update_cache(uint32_t llama_seq_id, llama_pos seq_id, llama_pos p0, llama_pos p1, llama_pos delta) {
    // 记录新的缓存大小
    uint32_t new_head = cache.size;

    // 如果 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    // 如果 p1 小于 0，则将其设置为 llama_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<llama_pos>::max();

    // 遍历缓存中的元素
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果缓存中的元素具有指定的序列 ID，并且位置在 p0 和 p1 之间
        if (cache.cells[i].has_seq_id(seq_id) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 标记缓存已经发生了位置变化
            cache.has_shift = true;
            // 更新缓存中元素的位置和增量
            cache.cells[i].pos   += delta;
            cache.cells[i].delta += delta;

            // 如果更新后的位置小于 0
            if (cache.cells[i].pos < 0) {
                // 将位置设置为 -1，清空序列 ID
                cache.cells[i].pos = -1;
                cache.cells[i].seq_id.clear();
                // 如果新的头部位置还未更新，则更新为当前位置
                if (new_head == cache.size) new_head = i;
            }
    }
    // 如果我们释放了一个槽位，将头指针设置为该槽位，以便从那里开始搜索。
    // 否则，我们从头开始搜索下一个槽位。
    cache.head = new_head != cache.size ? new_head : 0;
}

//
// 模型加载和保存
//

// 定义枚举类型 llama_fver
enum llama_fver {
    GGUF_FILE_VERSION_V1 = 1,  // 文件版本1
    GGUF_FILE_VERSION_V2 = 2,  // 文件版本2
    GGUF_FILE_VERSION_V3 = 3,  // 文件版本3
};

// 根据文件版本返回版本名称
static const char * llama_file_version_name(llama_fver version) {
    // 根据不同的版本号返回对应的版本名称
    switch (version) {
// 根据文件版本号返回对应的说明文字
switch (file_version) {
    case GGUF_FILE_VERSION_V1: return "GGUF V1 (support until nov 2023)";
    case GGUF_FILE_VERSION_V2: return "GGUF V2";
    case GGUF_FILE_VERSION_V3: return "GGUF V3 (latest)";
}

// 如果版本号不在已知范围内，则返回未知
return "unknown";
}

// 格式化张量形状为字符串
static std::string llama_format_tensor_shape(const std::vector<int64_t> & ne) {
    char buf[256];
    // 将第一个元素格式化为字符串
    snprintf(buf, sizeof(buf), "%5" PRId64, ne.at(0));
    // 遍历剩余元素，将其格式化为字符串并拼接到 buf 中
    for (size_t i = 1; i < ne.size(); i++) {
        snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), ", %5" PRId64, ne.at(i));
    }
    // 返回格式化后的字符串
    return buf;
}

// 格式化张量形状为字符串
static std::string llama_format_tensor_shape(const struct ggml_tensor * t) {
    char buf[256];
    // 将张量的第一个元素格式化为字符串
    snprintf(buf, sizeof(buf), "%5" PRId64, t->ne[0]);
```

# 循环遍历从1到GGML_MAX_DIMS的整数，将t->ne[i]的值格式化为字符串，追加到buf后面
for (int i = 1; i < GGML_MAX_DIMS; i++) {
    snprintf(buf + strlen(buf), sizeof(buf) - strlen(buf), ", %5" PRId64, t->ne[i]);
}
# 返回格式化后的字符串buf
return buf;
}

# 定义一个名为llama_model_loader的结构体
struct llama_model_loader {
    # 初始化n_kv为0
    int n_kv      = 0;
    # 初始化n_tensors为0
    int n_tensors = 0;
    # 初始化n_created为0
    int n_created = 0;

    # 定义ggml_sparse_deriv类型的sparse_deriv变量
    ggml_sparse_deriv sparse_deriv;

    # 初始化n_elements为0
    int64_t n_elements = 0;
    # 初始化n_bytes为0
    size_t  n_bytes    = 0;

    # 初始化use_mmap为false
    bool use_mmap = false;

    # 定义llama_file类型的file变量
    llama_file  file;
    # 定义llama_ftype类型的ftype变量
    llama_ftype ftype;
}
    // 定义名为 llama_fver 的结构体变量
    llama_fver  fver;

    // 创建一个指向 llama_mmap 对象的智能指针
    std::unique_ptr<llama_mmap> mapping;

    // 定义指向 gguf_context 结构体和 ggml_context 结构体的指针，并初始化为 NULL
    struct gguf_context * ctx_gguf = NULL;
    struct ggml_context * ctx_meta = NULL;

    // 定义 llama_model_loader 构造函数，接受文件名和是否使用内存映射作为参数
    llama_model_loader(const std::string & fname, bool use_mmap) : file(fname.c_str(), "rb") {
        // 初始化 gguf_init_params 结构体
        struct gguf_init_params params = {
            /*.no_alloc = */ true,  // 设置 no_alloc 字段为 true
            /*.ctx      = */ &ctx_meta,  // 设置 ctx 字段为 ctx_meta 的地址
        };

        // 从文件中初始化 gguf_context 结构体，并将结果赋值给 ctx_gguf
        ctx_gguf = gguf_init_from_file(fname.c_str(), params);
        // 如果初始化失败，抛出运行时错误
        if (!ctx_gguf) {
            throw std::runtime_error(format("%s: failed to load model from %s\n", __func__, fname.c_str()));
        }

        // 获取 ctx_gguf 中的键值对数量和张量数量
        n_kv      = gguf_get_n_kv(ctx_gguf);
        n_tensors = gguf_get_n_tensors(ctx_gguf);
        // 从上下文中获取稀疏导数
        sparse_deriv = gguf_get_sparse_deriv(ctx_gguf);
        // 从上下文中获取文件版本
        fver = (enum llama_fver ) gguf_get_version(ctx_gguf);

        // 遍历每个张量
        for (int i = 0; i < n_tensors; i++) {
            // 获取张量的名称
            const char * name = gguf_get_tensor_name(ctx_gguf, i);
            // 根据名称从元数据上下文中获取张量
            struct ggml_tensor * t = ggml_get_tensor(ctx_meta, name);
            // 计算张量的元素数量并累加
            n_elements += ggml_nelements(t);
            // 计算张量的字节数并累加
            n_bytes    += ggml_nbytes(t);
        }

        // 打印加载的元数据信息，包括键值对数量、张量数量、文件名和版本信息
        LLAMA_LOG_INFO("%s: loaded meta data with %d key-value pairs and %d tensors from %s (version %s)\n",
                __func__, n_kv, n_tensors, fname.c_str(), llama_file_version_name(fver));

        // 根据每种量化的张量数量确定文件类型，并打印元数据
        // TODO: make optional
        {
            // 创建一个映射，用于存储每种类型的张量数量
            std::map<enum ggml_type, uint32_t> n_type;

            // 初始化最大类型的张量数量为0
            uint32_t n_type_max = 0;
            // 初始化最大类型为GGML_TYPE_F32
            enum ggml_type type_max = GGML_TYPE_F32;
// 遍历 n_tensors 个张量
for (int i = 0; i < n_tensors; i++) {
    // 获取第 i 个张量的名称
    const char * name = gguf_get_tensor_name(ctx_gguf, i);
    // 获取第 i 个张量的元数据
    struct ggml_tensor * meta = ggml_get_tensor(ctx_meta, name);

    // 统计不同类型张量的数量
    n_type[meta->type]++;

    // 更新出现次数最多的张量类型及其数量
    if (n_type_max < n_type[meta->type]) {
        n_type_max = n_type[meta->type];
        type_max   = meta->type;
    }

    // 打印张量信息
    LLAMA_LOG_INFO("%s: - tensor %4d: %32s %-8s [ %s ]\n", __func__, i, name, ggml_type_name(meta->type), llama_format_tensor_shape(meta).c_str());
}

// 根据出现次数最多的张量类型进行不同的操作
switch (type_max) {
    case GGML_TYPE_F32:  ftype = LLAMA_FTYPE_ALL_F32;       break;
    case GGML_TYPE_F16:  ftype = LLAMA_FTYPE_MOSTLY_F16;    break;
    case GGML_TYPE_Q4_0: ftype = LLAMA_FTYPE_MOSTLY_Q4_0;   break;
    case GGML_TYPE_Q4_1: ftype = LLAMA_FTYPE_MOSTLY_Q4_1;   break;
}
// 根据不同的GGML类型设置对应的LLAMA文件类型
switch (ggml_type) {
    case GGML_TYPE_Q5_0: ftype = LLAMA_FTYPE_MOSTLY_Q5_0;   break;  // 如果是GGML_TYPE_Q5_0，则设置为LLAMA_FTYPE_MOSTLY_Q5_0
    case GGML_TYPE_Q5_1: ftype = LLAMA_FTYPE_MOSTLY_Q5_1;   break;  // 如果是GGML_TYPE_Q5_1，则设置为LLAMA_FTYPE_MOSTLY_Q5_1
    case GGML_TYPE_Q8_0: ftype = LLAMA_FTYPE_MOSTLY_Q8_0;   break;  // 如果是GGML_TYPE_Q8_0，则设置为LLAMA_FTYPE_MOSTLY_Q8_0
    case GGML_TYPE_Q2_K: ftype = LLAMA_FTYPE_MOSTLY_Q2_K;   break;  // 如果是GGML_TYPE_Q2_K，则设置为LLAMA_FTYPE_MOSTLY_Q2_K
    case GGML_TYPE_Q3_K: ftype = LLAMA_FTYPE_MOSTLY_Q3_K_M; break;  // 如果是GGML_TYPE_Q3_K，则设置为LLAMA_FTYPE_MOSTLY_Q3_K_M
    case GGML_TYPE_Q4_K: ftype = LLAMA_FTYPE_MOSTLY_Q4_K_M; break;  // 如果是GGML_TYPE_Q4_K，则设置为LLAMA_FTYPE_MOSTLY_Q4_K_M
    case GGML_TYPE_Q5_K: ftype = LLAMA_FTYPE_MOSTLY_Q5_K_M; break;  // 如果是GGML_TYPE_Q5_K，则设置为LLAMA_FTYPE_MOSTLY_Q5_K_M
    case GGML_TYPE_Q6_K: ftype = LLAMA_FTYPE_MOSTLY_Q6_K;   break;  // 如果是GGML_TYPE_Q6_K，则设置为LLAMA_FTYPE_MOSTLY_Q6_K
    default:  // 如果是其他类型
         {
             LLAMA_LOG_WARN("%s: unknown type %s\n", __func__, ggml_type_name(type_max));  // 记录警告日志，表示未知类型
             ftype = LLAMA_FTYPE_ALL_F32;  // 设置为LLAMA_FTYPE_ALL_F32
         } break;
}

// 标记为“猜测”文件类型的方式
ftype = (llama_ftype) (ftype | LLAMA_FTYPE_GUESSED);  // 使用位运算将ftype标记为“猜测”文件类型

// 查找指定键的值
const int kid = gguf_find_key(ctx_gguf, "general.file_type");
            // 如果 kid 大于等于 0，则获取对应的类型值
            if (kid >= 0) {
                ftype = (llama_ftype) gguf_get_val_u32(ctx_gguf, kid);
            }
        }

        // 遍历 n_kv，获取每个键值对的名称和类型，然后打印出来
        for (int i = 0; i < n_kv; i++) {
            const char * name         = gguf_get_key(ctx_gguf, i);
            const enum gguf_type type = gguf_get_kv_type(ctx_gguf, i);

            LLAMA_LOG_INFO("%s: - kv %3d: %42s %-8s\n", __func__, i, name, gguf_type_name(type));
        }

        // 打印类型计数
        for (auto & kv : n_type) {
            // 如果计数为 0，则跳过
            if (kv.second == 0) {
                continue;
            }

            // 打印类型和对应的张量数量
            LLAMA_LOG_INFO("%s: - type %4s: %4d tensors\n", __func__, ggml_type_name(kv.first), kv.second);
        }
        }

        // 检查是否支持 mmap，如果不支持则记录警告并禁用 mmap
        if (!llama_mmap::SUPPORTED) {
            LLAMA_LOG_WARN("%s: mmap is not supported on this platform\n", __func__);
            use_mmap = false;
        }

        // 设置类成员变量 use_mmap 为检查结果
        this->use_mmap = use_mmap;
    }

    // 析构函数，释放内存映射和元数据
    ~llama_model_loader() {
        if (ctx_gguf) {
            gguf_free(ctx_gguf);
        }
        if (ctx_meta) {
            ggml_free(ctx_meta);
        }
    }

    // 返回架构名称的函数
    std::string get_arch_name() const {
// 创建一个LLM_KV对象，用于存储LLM架构信息
const auto kv = LLM_KV(LLM_ARCH_UNKNOWN);

// 声明一个存储架构名称的字符串变量
std::string arch_name;

// 从ctx_gguf中获取架构名称，并存储到arch_name变量中
GGUF_GET_KEY(ctx_gguf, arch_name, gguf_get_val_str, GGUF_TYPE_STRING, false, kv(LLM_KV_GENERAL_ARCHITECTURE));

// 返回获取到的架构名称
return arch_name;
}

// 获取架构信息并转换为枚举类型
enum llm_arch get_arch() const {
    // 调用get_arch_name()函数获取架构名称
    const std::string arch_name = get_arch_name();

    // 将架构名称转换为枚举类型并返回
    return llm_arch_from_string(arch_name);
}

// 获取第i个张量的名称
const char * get_tensor_name(int i) const {
    // 从ctx_gguf中获取第i个张量的名称并返回
    return gguf_get_tensor_name(ctx_gguf, i);
}

// 获取第i个张量的元数据
struct ggml_tensor * get_tensor_meta(int i) const {
    // 从ctx_meta中获取第i个张量的元数据并返回
    return ggml_get_tensor(ctx_meta, get_tensor_name(i));
}
    // 计算上下文大小和内存映射大小
    void calc_sizes(size_t & ctx_size_p, size_t & mmapped_size_p) const {
        ctx_size_p     = 0;  // 初始化上下文大小为0
        mmapped_size_p = 0;  // 初始化内存映射大小为0

        for (int i = 0; i < n_tensors; i++) {
            // 获取第i个张量的元数据
            struct ggml_tensor * meta = get_tensor_meta(i);
            // 计算上下文大小
            ctx_size_p += sizeof(struct ggml_tensor) + GGML_OBJECT_SIZE;
            // 如果使用内存映射，则将内存映射大小增加，否则增加上下文大小
            (use_mmap ? mmapped_size_p : ctx_size_p) += ggml_nbytes_pad(meta);
        }
    }

    // 为给定上下文和元数据创建张量
    struct ggml_tensor * create_tensor_for(struct ggml_context * ctx, struct ggml_tensor * meta, ggml_backend_type backend) {
        // 如果后端不是CPU，则设置不分配内存
        if (backend != GGML_BACKEND_CPU) {
            ggml_set_no_alloc(ctx, true);
        }

        // 复制元数据创建张量
        struct ggml_tensor * tensor = ggml_dup_tensor(ctx, meta);
        tensor->backend = backend; // 设置张量的后端类型
        // TODO: ggml_set_backend
// 设置张量的名称为元数据中的名称
ggml_set_name(tensor, ggml_get_name(meta));

// 如果后端不是 CPU，则设置不使用内存映射
if (backend != GGML_BACKEND_CPU) {
    ggml_set_no_alloc(ctx, use_mmap);
}

// 增加创建张量的计数
n_created++;

// 返回创建的张量
return tensor;
}

// 根据张量名称、张量维度和后端类型创建张量
struct ggml_tensor * create_tensor(struct ggml_context * ctx, const std::pair<std::string, tensor_offloading_levels> & tn, const std::vector<int64_t> & ne, ggml_backend_type backend) {
    return create_tensor(ctx, tn.first, ne, backend);
}

// 根据张量名称、张量维度和后端类型创建张量
struct ggml_tensor * create_tensor(struct ggml_context * ctx, const std::string &name, const std::vector<int64_t> & ne, ggml_backend_type backend) {
    // 获取上下文中指定名称的张量
    struct ggml_tensor * cur = ggml_get_tensor(ctx_meta, name.c_str());

    // 如果张量不存在，则抛出运行时错误
    if (cur == NULL) {
        throw std::runtime_error(format("%s: tensor '%s' not found", __func__, name.c_str()));
        }

        // 如果后端是 GGML_BACKEND_GPU_SPLIT
        if (backend == GGML_BACKEND_GPU_SPLIT) {
            // 如果张量的维度为1，则抛出异常
            if (ne.size() == 1) {
                throw std::runtime_error(format("%s: 1-dimensional tensor '%s' cannot be split on the GPU", __func__, name.c_str()));
            }
        }

        {
            // 检查张量的形状是否符合预期
            bool is_ok = true;
            for (size_t i = 0; i < ne.size(); ++i) {
                if (ne[i] != cur->ne[i]) {
                    // 允许 ne 中的 -1 作为通配符维度
                    is_ok = ne[i] == -1;
                    break;
                }
            }
            // 如果形状不符合预期，则抛出异常
            if (!is_ok) {
                throw std::runtime_error(
                        format("%s: tensor '%s' has wrong shape; expected %s, got %s",
    // 调用函数__func__，传入name.c_str()、ne和cur作为参数，格式化输出信息
    llama_log_info(format("Getting tensor '%s' with shape %s (current shape %s)",
                            __func__, name.c_str(),
                            llama_format_tensor_shape(ne).c_str(),
                            llama_format_tensor_shape(cur).c_str()));
    }

    // 返回根据当前上下文、当前形状和后端创建的张量
    return create_tensor_for(ctx, cur, backend);
}

// 完成获取张量的操作
void done_getting_tensors() const {
    // 如果创建的张量数量不等于预期的张量数量，抛出运行时错误
    if (n_created != n_tensors) {
        throw std::runtime_error(format("%s: wrong number of tensors; expected %d, got %d", __func__, n_tensors, n_created));
    }
}

// 返回文件中指定名称的偏移量
size_t file_offset(const char * name) const {
    // 在上下文中查找张量的索引
    const int idx = gguf_find_tensor(ctx_gguf, name);

    // 如果索引小于0，抛出运行时错误，表示文件中未找到指定名称的张量
    if (idx < 0) {
        throw std::runtime_error(format("%s: tensor '%s' not found in the file", __func__, name));
    }

    // 返回指定索引的张量数据在文件中的偏移量
    return gguf_get_data_offset(ctx_gguf) + gguf_get_tensor_offset(ctx_gguf, idx);
}

// 加载指定张量的数据
void load_data_for(struct ggml_tensor * cur) const {
    // 获取张量数据在文件中的偏移量
    const size_t offs = file_offset(ggml_get_name(cur));

    // 如果使用内存映射，则将数据指针指向对应偏移量处
    if (use_mmap) {
        cur->data = (uint8_t *) mapping->addr + offs;
    } else {
        // 否则，定位文件指针到对应偏移量处，读取数据到张量中
        file.seek(offs, SEEK_SET);
        file.read_raw(cur->data, ggml_nbytes(cur));
    }
}

// 加载所有数据
void load_all_data(struct ggml_context * ctx, llama_progress_callback progress_callback, void * progress_callback_user_data, llama_mlock * lmlock) {
    // 初始化数据大小和锁大小
    size_t size_data = 0;
    size_t size_lock = 0;
    size_t size_pref = 0; // prefetch
// 遍历 gguf 中的张量，计算数据大小
for (int i = 0; i < gguf_get_n_tensors(ctx_gguf); i++) {
    // 获取当前张量
    struct ggml_tensor * cur = ggml_get_tensor(ctx, gguf_get_tensor_name(ctx_gguf, i));
    // 计算数据大小
    size_data += ggml_nbytes(cur);
    // 如果当前张量的后端是 CPU，则增加预取数据大小
    if (cur->backend == GGML_BACKEND_CPU) {
        size_pref += ggml_nbytes(cur);
    }
}

// 如果使用内存映射
if (use_mmap) {
    // 创建内存映射对象
    mapping.reset(new llama_mmap(&file, size_pref, ggml_is_numa()));
    // 如果需要锁定内存，则初始化锁定
    if (lmlock) {
        lmlock->init(mapping->addr);
    }
}

// 初始化已处理数据大小
size_t done_size = 0;
// 再次遍历 gguf 中的张量
for (int i = 0; i < gguf_get_n_tensors(ctx_gguf); i++) {
    // 获取当前张量
    struct ggml_tensor * cur = ggml_get_tensor(ctx, gguf_get_tensor_name(ctx_gguf, i));
    // 断言当前张量不为空，因为未使用的张量应该已经在 load_data 中被捕获
    GGML_ASSERT(cur); // unused tensors should have been caught by load_data already
}
            // 如果存在进度回调函数，则调用回调函数，传入当前进度和用户数据
            if (progress_callback) {
                progress_callback((float) done_size / size_data, progress_callback_user_data);
            }

            // 如果不使用内存映射且当前数据为空，则分配临时缓冲区
            if (!use_mmap && cur->data == NULL) {
                // 确保当前数据不在 CPU 后端
                GGML_ASSERT(cur->backend != GGML_BACKEND_CPU);
                #ifdef GGML_USE_CPU_HBM
                // 如果支持 CPU HBM，则使用 hbw_malloc 分配内存
                cur->data = (uint8_t*)hbw_malloc(ggml_nbytes(cur));
                #else
                // 否则使用 malloc 分配内存
                cur->data = (uint8_t*)malloc(ggml_nbytes(cur));
                #endif
            }

            // 加载当前数据
            load_data_for(cur);

            // 根据后端类型进行不同处理
            switch (cur->backend) {
                case GGML_BACKEND_CPU:
                    // 如果使用内存映射且 lmlock 为真，则执行以下操作
                    if (use_mmap && lmlock) {
                    // 计算当前数据的大小并加到锁定的大小上
                    size_lock += ggml_nbytes(cur);
                    // 调整锁定的大小
                    lmlock->grow_to(size_lock);
                    // 跳出当前的 switch 语句
                    break;
#ifdef GGML_USE_CUBLAS
                case GGML_BACKEND_GPU:
                case GGML_BACKEND_GPU_SPLIT:
                    // 旧代码：
                    //ggml_cuda_transform_tensor(lt.data, lt.ggml_tensor);

                    // TODO: 测试这段代码是否有效！！
                    // 使用 CUDA 转换张量
                    ggml_cuda_transform_tensor(cur->data, cur);
                    // 如果不使用内存映射，则释放当前数据
                    if (!use_mmap) {
                        free(cur->data);
                    }
                    // 跳出当前的 switch 语句
                    break;
#elif defined(GGML_USE_CLBLAST)
                case GGML_BACKEND_GPU:
                    // 使用 OpenCL 转换张量
                    ggml_cl_transform_tensor(cur->data, cur);
                    // 如果不使用内存映射，则释放当前数据
                    if (!use_mmap) {
                    free(cur->data);
                    // 释放当前节点的数据内存
                }
                break;
#ifndef
                default:
                    // 默认情况下继续循环
                    continue;
            }

            done_size += ggml_nbytes(cur);
            // 更新已处理数据的大小
        }
    }
};

//
// load LLaMA models
//

static std::string llama_model_arch_name(llm_arch arch) {
    auto it = LLM_ARCH_NAMES.find(arch);
    // 在LLM_ARCH_NAMES中查找给定的arch
    if (it == LLM_ARCH_NAMES.end()) {
        // 如果未找到，执行以下操作
        return "unknown";
    }
    return it->second;
}

static std::string llama_model_ftype_name(llama_ftype ftype) {
    // 如果 ftype 包含 LLAMA_FTYPE_GUESSED 标志位
    if (ftype & LLAMA_FTYPE_GUESSED) {
        // 返回去除 LLAMA_FTYPE_GUESSED 标志位后的 ftype 对应的名称，并加上 "(guessed)" 后缀
        return llama_model_ftype_name((enum llama_ftype) (ftype & ~LLAMA_FTYPE_GUESSED)) + " (guessed)";
    }

    // 根据 ftype 的值进行不同的处理
    switch (ftype) {
        case LLAMA_FTYPE_ALL_F32:     return "all F32";
        case LLAMA_FTYPE_MOSTLY_F16:  return "mostly F16";
        case LLAMA_FTYPE_MOSTLY_Q4_0: return "mostly Q4_0";
        case LLAMA_FTYPE_MOSTLY_Q4_1: return "mostly Q4_1";
        case LLAMA_FTYPE_MOSTLY_Q4_1_SOME_F16:
                                      return "mostly Q4_1, some F16";
        case LLAMA_FTYPE_MOSTLY_Q5_0: return "mostly Q5_0";
        case LLAMA_FTYPE_MOSTLY_Q5_1: return "mostly Q5_1";
        case LLAMA_FTYPE_MOSTLY_Q8_0: return "mostly Q8_0";
// 根据不同的LLAMA_FTYPE_MOSTLY_Q*类型返回对应的描述字符串
// 返回值为描述字符串
static const char * llama_file_type_name(e_file_type type) {
    switch (type) {
        case LLAMA_FTYPE_MOSTLY_Q2_K:   return "mostly Q2_K";
        case LLAMA_FTYPE_MOSTLY_Q3_K_S: return "mostly Q3_K - Small";
        case LLAMA_FTYPE_MOSTLY_Q3_K_M: return "mostly Q3_K - Medium";
        case LLAMA_FTYPE_MOSTLY_Q3_K_L: return "mostly Q3_K - Large";
        case LLAMA_FTYPE_MOSTLY_Q4_K_S: return "mostly Q4_K - Small";
        case LLAMA_FTYPE_MOSTLY_Q4_K_M: return "mostly Q4_K - Medium";
        case LLAMA_FTYPE_MOSTLY_Q5_K_S: return "mostly Q5_K - Small";
        case LLAMA_FTYPE_MOSTLY_Q5_K_M: return "mostly Q5_K - Medium";
        case LLAMA_FTYPE_MOSTLY_Q6_K:   return "mostly Q6_K";
        // 默认情况下返回未知类型的描述字符串
        default: return "unknown, may not work";
    }
}

// 根据不同的MODEL_*类型返回对应的描述字符串
// 返回值为描述字符串
static const char * llama_model_type_name(e_model type) {
    switch (type) {
        case MODEL_1B:  return "1B";
        case MODEL_3B:  return "3B";
        // 其他情况下返回未知类型的描述字符串
        default: return "unknown";
    }
}
// 根据不同的模型类型返回对应的字符串
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

// 加载模型的架构信息
static void llm_load_arch(llama_model_loader & ml, llama_model & model) {
    // 获取模型的架构信息
    model.arch = ml.get_arch();
    // 如果模型的架构信息未知，则抛出运行时错误
    if (model.arch == LLM_ARCH_UNKNOWN) {
        throw std::runtime_error("unknown model architecture: '" + ml.get_arch_name() + "'");
    }
}
// 加载模型超参数
static void llm_load_hparams(
        llama_model_loader & ml,  // llama_model_loader 对象引用
        llama_model & model) {    // llama_model 对象引用
    struct gguf_context * ctx = ml.ctx_gguf;  // 获取 gguf 上下文

    const auto kv = LLM_KV(model.arch);  // 获取模型架构的键值对

    auto & hparams = model.hparams;  // 获取模型超参数

    // 获取通用键值对
    GGUF_GET_KEY(ctx, model.name, gguf_get_val_str, GGUF_TYPE_STRING, false, kv(LLM_KV_GENERAL_NAME));

    // 获取超参数键值对
    GGUF_GET_KEY(ctx, hparams.n_vocab,        gguf_get_arr_n,   GGUF_TYPE_ARRAY,  true, kv(LLM_KV_TOKENIZER_LIST));
    GGUF_GET_KEY(ctx, hparams.n_ctx_train,    gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_CONTEXT_LENGTH));
    GGUF_GET_KEY(ctx, hparams.n_embd,         gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_EMBEDDING_LENGTH));
    GGUF_GET_KEY(ctx, hparams.n_ff,           gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_FEED_FORWARD_LENGTH));
    GGUF_GET_KEY(ctx, hparams.n_head,         gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_ATTENTION_HEAD_COUNT));
    GGUF_GET_KEY(ctx, hparams.n_layer,        gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_BLOCK_COUNT));
}
    // 将 n_head_kv 设置为 n_head 的值，如果没有指定则默认为 n_head
    hparams.n_head_kv = hparams.n_head;
    // 从上下文中获取键值对，将其赋值给 hparams.n_head_kv，键值对的类型为 uint32，如果不存在则使用默认值
    GGUF_GET_KEY(ctx, hparams.n_head_kv, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ATTENTION_HEAD_COUNT_KV));

    // 将 rope_finetuned 设置为 false
    hparams.rope_finetuned = false;
    // 从上下文中获取键值对，将其赋值给 hparams.rope_finetuned，键值对的类型为 bool，如果不存在则使用默认值
    GGUF_GET_KEY(ctx, hparams.rope_finetuned, gguf_get_val_bool, GGUF_TYPE_BOOL, false,
                 kv(LLM_KV_ROPE_SCALING_FINETUNED));

    // 将 n_yarn_orig_ctx 设置为 n_ctx_train 的值
    hparams.n_yarn_orig_ctx = hparams.n_ctx_train;
    // 从上下文中获取键值对，将其赋值给 hparams.n_yarn_orig_ctx，键值对的类型为 uint32，如果不存在则使用默认值
    GGUF_GET_KEY(ctx, hparams.n_yarn_orig_ctx, gguf_get_val_u32, GGUF_TYPE_UINT32, false,
                 kv(LLM_KV_ROPE_SCALING_ORIG_CTX_LEN));

    // 将 rope_freq_base_train 设置为 10000.0f
    hparams.rope_freq_base_train = 10000.0f;
    // 从上下文中获取键值对，将其赋值给 hparams.rope_freq_base_train，键值对的类型为 float32，如果不存在则使用默认值
    GGUF_GET_KEY(ctx, hparams.rope_freq_base_train, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_FREQ_BASE));

    // 将 rope_scaling 设置为 "linear"
    std::string rope_scaling("linear");
    // 从上下文中获取键值对，将其赋值给 rope_scaling，键值对的类型为 string，如果不存在则使用默认值
    GGUF_GET_KEY(ctx, rope_scaling, gguf_get_val_str, GGUF_TYPE_STRING, false, kv(LLM_KV_ROPE_SCALING_TYPE));
    // 将字符串类型的 rope_scaling 转换为相应的枚举类型，并赋值给 hparams.rope_scaling_type_train
    hparams.rope_scaling_type_train = llama_rope_scaling_type_from_string(rope_scaling);
    // 断言 hparams.rope_scaling_type_train 不为 LLAMA_ROPE_SCALING_UNSPECIFIED
    GGML_ASSERT(hparams.rope_scaling_type_train != LLAMA_ROPE_SCALING_UNSPECIFIED);
// rope_freq_scale (inverse of the kv) is optional
// 定义并初始化变量ropescale，用于存储绳索频率的缩放因子，默认值为0.0
float ropescale = 0.0f;
// 从上下文中获取键值为LLM_KV_ROPE_SCALING_FACTOR的值，如果不存在则不做任何操作
GGUF_GET_KEY(ctx, ropescale, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_SCALING_FACTOR));
// 如果ropescale的值为0.0f，则尝试获取旧的键名LLM_KV_ROPE_SCALE_LINEAR的值
if (ropescale == 0.0f) {
    GGUF_GET_KEY(ctx, ropescale, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_SCALE_LINEAR));
}
// 根据ropescale的值设置hparams.rope_freq_scale_train的值
hparams.rope_freq_scale_train = ropescale == 0.0f ? 1.0f : 1.0f/ropescale;

// sanity check for n_rot (optional)
// 对n_rot进行健全性检查，n_rot的值为n_embd除以n_head
{
    hparams.n_rot = hparams.n_embd / hparams.n_head;
    // 从上下文中获取键值为LLM_KV_ROPE_DIMENSION_COUNT的值，如果不存在则不做任何操作
    GGUF_GET_KEY(ctx, hparams.n_rot, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ROPE_DIMENSION_COUNT));
    // 如果模型的架构为LLM_ARCH_LLAMA或LLM_ARCH_FALCON，并且n_rot的值不等于n_embd除以n_head，则抛出运行时错误
    if (model.arch == LLM_ARCH_LLAMA || model.arch == LLM_ARCH_FALCON) {
        if (hparams.n_rot != hparams.n_embd / hparams.n_head) {
            throw std::runtime_error(format("invalid n_rot: %u, expected %u", hparams.n_rot, hparams.n_embd / hparams.n_head));
        }
    }
}
    // 计算旋转矩阵的维度，根据旋转百分比和嵌入维度以及头数计算
    // 在gpt-j模型中，旋转矩阵的维度等于旋转维度
    }

    if (gguf_get_sparse_deriv(ctx)) {
        // 如果启用了稀疏导数，读取稀疏阈值覆盖值
        GGUF_GET_KEY(ctx, hparams.sparse_pred_threshold, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_SPARSE_THRESHOLD));
    }

    // 架构特定的键值对
    switch (model.arch) {
        case LLM_ARCH_LLAMA:
            {
                // 获取层归一化的RMS误差阈值
                GGUF_GET_KEY(ctx, hparams.f_norm_rms_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS));

                switch (hparams.n_layer) {
                    case 26: model.type = e_model::MODEL_3B; break;
                    case 32: model.type = e_model::MODEL_7B; break;
                    case 40: model.type = e_model::MODEL_13B; break;
                    case 48: model.type = e_model::MODEL_34B; break;
// 根据不同的条件设置模型类型
switch (arch) {
    case LLM_ARCH_GPT3:
        {
            // 根据参数设置模型类型
            GGUF_GET_KEY(ctx, hparams.f_norm_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_EPS));
            
            // 根据不同的层数设置不同的模型类型
            switch (hparams.n_layer) {
                case 60: model.type = e_model::MODEL_30B; break;
                case 80: model.type = hparams.n_head == hparams.n_head_kv ? e_model::MODEL_65B : e_model::MODEL_70B; break;
                default: model.type = e_model::MODEL_UNKNOWN;
            }
        } break;
    case LLM_ARCH_FALCON:
        {
            // 根据参数设置模型类型
            GGUF_GET_KEY(ctx, hparams.f_norm_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_EPS);
            
            // 根据不同的层数设置不同的模型类型
            switch (hparams.n_layer) {
                case 32: model.type = e_model::MODEL_7B; break;
                case 60: model.type = e_model::MODEL_40B; break;
                default: model.type = e_model::MODEL_UNKNOWN;
            }
        } break;
    case LLM_ARCH_BAICHUAN:
        {
            // 根据参数设置模型类型
            GGUF_GET_KEY(ctx, hparams.f_norm_rms_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS));
            
            // 根据不同的层数设置不同的模型类型
            switch (hparams.n_layer) {
                case 32: model.type = e_model::MODEL_7B; break;
// 根据不同的架构类型设置模型类型
case LLM_ARCH_STARCODER: 
    // 获取参数中的注意力层归一化系数
    GGUF_GET_KEY(ctx, hparams.f_norm_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_EPS));
    // 根据层的数量设置模型类型
    switch (hparams.n_layer) {
        case 24: model.type = e_model::MODEL_1B; break;
        case 36: model.type = e_model::MODEL_3B; break;
        case 42: model.type = e_model::MODEL_7B; break;
        case 40: model.type = e_model::MODEL_15B; break;
        default: model.type = e_model::MODEL_UNKNOWN;
    }
    break;
case LLM_ARCH_PERSIMMON:
    // 获取参数中的注意力层归一化系数
    GGUF_GET_KEY(ctx, hparams.f_norm_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_EPS));
    // 根据层的数量设置模型类型
    switch (hparams.n_layer) {
        case 36: model.type = e_model::MODEL_8B; break;
// 根据不同的架构类型设置模型类型
switch (arch_type) {
    case LLM_ARCH_DEFAULT:
        {
            // 根据参数设置模型类型
            GGUF_GET_KEY(ctx, hparams.f_norm_rms_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS));
            // 根据层数设置模型类型
            switch (hparams.n_layer) {
                case 32: model.type = e_model::MODEL_1B; break;
                default: model.type = e_model::MODEL_UNKNOWN;
            }
        } break;
    case LLM_ARCH_REFACT:
        {
            // 根据参数设置模型类型
            GGUF_GET_KEY(ctx, hparams.f_norm_rms_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS));
            // 根据层数设置模型类型
            switch (hparams.n_layer) {
                case 32: model.type = e_model::MODEL_1B; break;
                default: model.type = e_model::MODEL_UNKNOWN;
            }
        } break;
    case LLM_ARCH_BLOOM:
        {
            // 根据参数设置模型类型
            GGUF_GET_KEY(ctx, hparams.f_norm_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_EPS));
            // 根据层数和嵌入维度设置模型类型
            switch (hparams.n_layer) {
                case 24: model.type = e_model::MODEL_1B; break;
                case 30:
                    switch (hparams.n_embd) {
                        case 2560: model.type = e_model::MODEL_3B; break;
// 根据不同的情况设置模型类型
switch (hparams.n_layer) {
    // 当层数为 32 时，设置模型类型为 MODEL_7B
    case 32: model.type = e_model::MODEL_7B; break;
    // 当层数为 48 时，设置模型类型为 MODEL_30B
    case 48: model.type = e_model::MODEL_30B; break;
    // 其他情况下，设置模型类型为 MODEL_UNKNOWN
    default: model.type = e_model::MODEL_UNKNOWN;
}
// 从上下文中获取参数的值，并存储到指定的变量中
GGUF_GET_KEY(ctx, hparams.f_norm_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, kv(LLM_KV_ATTENTION_LAYERNORM_EPS));

// 根据不同的层数设置模型的类型
switch (hparams.n_layer) {
    case 32: model.type = e_model::MODEL_3B; break;
    default: model.type = e_model::MODEL_UNKNOWN;
}

// 设置模型的特征类型
model.ftype = ml.ftype;
}

// TODO: 这个可能应该放在llama.h中
// 内部调用，根据词汇表将原始文本进行分词处理
static std::vector<llama_vocab::id> llama_tokenize_internal(const llama_vocab & vocab, std::string raw_text, bool bos, bool special = false);
// 将字节转换为标记
static llama_token llama_byte_to_token(const llama_vocab & vocab, uint8_t ch);

// 加载词汇表
static void llm_load_vocab(
        llama_model_loader & ml,
// 传入 llama_model 和 model 对象
void some_function(llama_model & model) {
    // 获取 model 对象中的 vocab
    auto & vocab = model.vocab;

    // 获取 ml 对象中的 ctx_gguf
    struct gguf_context * ctx = ml.ctx_gguf;

    // 获取 model.arch 对应的键值
    const auto kv = LLM_KV(model.arch);

    // 在 ctx 中查找 token_idx 对应的键值
    const int token_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_LIST).c_str());
    // 如果找不到，抛出异常
    if (token_idx == -1) {
        throw std::runtime_error("cannot find tokenizer vocab in model file\n");
    }

    // 初始化 scores 为 nullptr
    const float * scores = nullptr;
    // 在 ctx 中查找 score_idx 对应的键值
    const int score_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_SCORES).c_str());
    // 如果找到，将 scores 指向对应的数组数据
    if (score_idx != -1) {
        scores = (const float * ) gguf_get_arr_data(ctx, score_idx);
    }

    // 初始化 toktypes 为 nullptr
    const int * toktypes = nullptr;
    // 在 ctx 中查找 toktype_idx 对应的键值
    const int toktype_idx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_TOKEN_TYPE).c_str());
```

    // 如果 toktype_idx 不等于 -1，则获取 toktype 数据
    if (toktype_idx != -1) {
        toktypes = (const int * ) gguf_get_arr_data(ctx, toktype_idx);
    }

    // 确定词汇表类型
    {
        // 声明一个字符串变量 tokenizer_name
        std::string tokenizer_name;

        // 从上下文中获取键为 LLM_KV_TOKENIZER_MODEL 的值，并存储到 tokenizer_name 中
        GGUF_GET_KEY(ctx, tokenizer_name, gguf_get_val_str, GGUF_TYPE_STRING, true, kv(LLM_KV_TOKENIZER_MODEL));

        // 如果 tokenizer_name 为 "llama"，则设置词汇表类型为 LLAMA_VOCAB_TYPE_SPM，并设置默认特殊标记的值
        if (tokenizer_name == "llama") {
            vocab.type = LLAMA_VOCAB_TYPE_SPM;
            vocab.special_bos_id = 1;
            vocab.special_eos_id = 2;
            vocab.special_unk_id = 0;
            vocab.special_sep_id = -1;
            vocab.special_pad_id = -1;
        } 
        // 如果 tokenizer_name 为 "gpt2"，则...
        else if (tokenizer_name == "gpt2") {
// 设置vocab的类型为LLAMA_VOCAB_TYPE_BPE
vocab.type = LLAMA_VOCAB_TYPE_BPE;

// 读取bpe merges并填充bpe ranks
// 查找tokenizer merges在模型文件中的索引
const int merges_keyidx = gguf_find_key(ctx, kv(LLM_KV_TOKENIZER_MERGES).c_str());
// 如果找不到tokenizer merges，则抛出运行时错误
if (merges_keyidx == -1) {
    throw std::runtime_error("cannot find tokenizer merges in model file\n");
}

// 获取merges的数量
const int n_merges = gguf_get_arr_n(ctx, merges_keyidx);

// 遍历merges
for (int i = 0; i < n_merges; i++) {
    // 获取merge中的单词
    const std::string word = gguf_get_arr_str(ctx, merges_keyidx, i);
    // 确保单词转换成codepoints后长度大于0
    GGML_ASSERT(codepoints_from_utf8(word).size() > 0);

    // 定义两个字符串用于存储merge的两个部分
    std::string first;
    std::string second;

    // 查找空格的位置，分割merge为两部分
    const size_t pos = word.find(' ', 1);

    // 如果找到空格
    if (pos != std::string::npos) {
            // 用于分割词语的位置
            first  = word.substr(0, pos);
            // 分割后的词语
            second = word.substr(pos + 1);
        }

        // 将分割后的词语对应的索引加入到词汇表的BPE排名中
        vocab.bpe_ranks.emplace(std::make_pair(first, second), i);
    }

    // 默认特殊标记
    vocab.special_bos_id = 11;
    vocab.special_eos_id = 11;
    vocab.special_unk_id = -1;
    vocab.special_sep_id = -1;
    vocab.special_pad_id = -1;
} else {
    // 如果未知的分词器，则记录警告信息
    LLAMA_LOG_WARN("%s: unknown tokenizer: '%s'", __func__, tokenizer_name.c_str());
    LLAMA_LOG_WARN("%s: using default tokenizer: 'llama'", __func__);

    // 使用默认的分词器类型
    vocab.type = LLAMA_VOCAB_TYPE_SPM;
}
    // 获取词汇表中词汇的数量
    const uint32_t n_vocab = gguf_get_arr_n(ctx, token_idx);

    // 调整 id_to_token 的大小以容纳词汇数量
    vocab.id_to_token.resize(n_vocab);

    // 遍历词汇表中的词汇
    for (uint32_t i = 0; i < n_vocab; i++) {
        // 从上下文中获取词汇表中的词汇，并转换为字符串
        std::string word = gguf_get_arr_str(ctx, token_idx, i);
        // 确保词汇转换为 UTF-8 编码后有内容
        GGML_ASSERT(codepoints_from_utf8(word).size() > 0);

        // 将词汇和对应的索引存入 token_to_id 中
        vocab.token_to_id[word] = i;

        // 获取 id_to_token 中的 token_data 引用
        auto & token_data = vocab.id_to_token[i];
        // 将词汇移动到 token_data 的文本字段中
        token_data.text  = std::move(word);
        // 如果存在分数数组，则将分数存入 token_data 的 score 字段中，否则为 0.0f
        token_data.score = scores ? scores[i] : 0.0f;
        // 如果存在类型数组，则将类型存入 token_data 的 type 字段中，否则为 LLAMA_TOKEN_TYPE_NORMAL
        token_data.type  = toktypes ? (llama_token_type) toktypes[i] : LLAMA_TOKEN_TYPE_NORMAL;
    }
    // 确保 id_to_token 和 token_to_id 的大小相等
    GGML_ASSERT(vocab.id_to_token.size() == vocab.token_to_id.size());

    // 确定换行符的词汇：如果词汇表类型为 LLAMA_VOCAB_TYPE_SPM
    if (vocab.type == LLAMA_VOCAB_TYPE_SPM) {
// 如果换行符在词汇表中已经存在，则将其转换为对应的标记ID
vocab.linefeed_id = llama_byte_to_token(vocab, '\n');
} else {
    // 如果换行符在词汇表中不存在，则将其添加到词汇表中，并获取对应的标记ID
    const std::vector<int> ids = llama_tokenize_internal(vocab, "\u010A", false);
    GGML_ASSERT(!ids.empty() && "model vocab missing newline token");
    vocab.linefeed_id = ids[0];
}

// 特殊标记
{
    // 定义包含特殊标记类型和对应标记ID的向量
    const std::vector<std::pair<enum llm_kv, int32_t &>> special_token_types = {
        { LLM_KV_TOKENIZER_BOS_ID, vocab.special_bos_id },
        { LLM_KV_TOKENIZER_EOS_ID, vocab.special_eos_id },
        { LLM_KV_TOKENIZER_UNK_ID, vocab.special_unk_id },
        { LLM_KV_TOKENIZER_SEP_ID, vocab.special_sep_id },
        { LLM_KV_TOKENIZER_PAD_ID, vocab.special_pad_id },
    };
    // 遍历特殊标记类型和对应标记ID的向量
    for (const auto & it : special_token_types) {
        // 获取特殊标记类型对应的字符串键
        const std::string & key = kv(std::get<0>(it));
        // 获取特殊标记ID的引用
        int32_t & id = std::get<1>(it), old_id = id;
    // 从上下文中获取特殊标记的键值对应的值
    GGUF_GET_KEY(ctx, id, gguf_get_val_u32, GGUF_TYPE_UINT32, false, key);
    // id 必须大于等于 -1 并且小于词汇表大小。由于 id 是无符号的，-1 只能来自默认值，所以没有必要验证。
    if (size_t(id + 1) > vocab.id_to_token.size()) {
        // 如果 id 超出了词汇表大小范围，则记录警告信息，并使用旧的 id
        LLAMA_LOG_WARN("%s: bad special token: '%s' = %d, using default id %d\n",
            __func__, key.c_str(), id, old_id);
        id = old_id;
    }
}

// 构建特殊标记缓存
{
    // TODO: 目前尚不清楚特殊标记是否保证是确定类型的，是否总是在 'added_tokens.json' 等文件中正确标记。
    // 假设是，因为特殊标记不是面向最终用户的，它们被设计为无法被分词器匹配，因此词汇表中无法被分词器匹配的标记就是特殊标记。
    // 从测试来看，这似乎与特殊标记是一一对应的。
}
        // 初始化特殊标记计数器
        uint32_t special_tokens_count_by_type = 0;
        uint32_t special_tokens_count_from_verification = 0;

        // 初始化特殊标记定义不匹配标志
        bool special_tokens_definition_mismatch = false;

        // 遍历词汇表中的每个标记
        for (const auto & t : vocab.token_to_id) {
            const auto & token = t.first;
            const auto & id    = t.second;

            // 在迭代过程中计算词汇表中非正常标记的数量
            if (vocab.id_to_token[id].type != LLAMA_TOKEN_TYPE_NORMAL) {
                special_tokens_count_by_type++;
            }

            // 跳过单个字符标记
            # 检查 token 的长度是否大于1
            if (token.length() > 1) {
                # 初始化一个变量用于标记是否可以被分词
                bool is_tokenizable = false;

                # 将 token 字符串表示按照所有可能的方式分割成两部分，并检查两部分是否都可以匹配到有效的 token
                for (unsigned i = 1; i < token.length();) {
                    # 获取左半部分和右半部分的子字符串
                    const auto left  = token.substr(0, i);
                    const auto right = token.substr(i);

                    # 检查是否没有在 utf 序列的中间分割
                    auto utf = utf8_len(left.at(left.length() - 1));

                    if (utf == 1) {
                        # 如果左半部分和右半部分都可以在词汇表中找到对应的 id，则标记为可以被分词，并跳出循环
                        if (vocab.token_to_id.find(left)  != vocab.token_to_id.end() &&
                            vocab.token_to_id.find(right) != vocab.token_to_id.end() ) {
                            is_tokenizable = true;
                            break;
                        }
                        i++;
                    } else {
                    // 跳过多字节 utf 序列的剩余部分
                    i += utf - 1;
                }
            }

            if (!is_tokenizable) {
                // 一些标记是多字节的，但它们是等效文本长度为1的 utf 序列
                // 在这里重新过滤它们会更快，因为现在候选者要少得多

                // 计算标记字符串表示的总 "utf" 长度
                size_t utf8_str_len = 0;
                for (unsigned i = 0; i < token.length();) {
                    utf8_str_len++;
                    i += utf8_len(token.at(i));
                }

                // 跳过那些只有一个字符的标记
                if (utf8_str_len > 1) {
                    // 此时剩下的只有特殊标记
                    vocab.special_tokens_cache[token] = id;
// 逐个计算手动找到的特殊标记的数量
special_tokens_count_from_verification++;

// 如果这个手动找到的特殊标记没有被标记为特殊标记，标记为不匹配
if (vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_NORMAL) {
    special_tokens_definition_mismatch = true;
}
```

这段代码的作用是对特殊标记进行计数，并检查是否有不匹配的情况。如果有不匹配的情况，则记录下来并输出警告信息。
// 打印特殊标记定义检查的信息
LLAMA_LOG_INFO("%s: special tokens definition check successful ( %u/%zu ).\n",
    __func__,
    special_tokens_count_from_verification, vocab.id_to_token.size()
);

// 打印模型加载的元信息
static void llm_load_print_meta(llama_model_loader & ml, llama_model & model) {
    // 获取模型的超参数和词汇表
    const auto & hparams = model.hparams;
    const auto & vocab   = model.vocab;

    // 获取绳索缩放类型
    const auto rope_scaling_type = LLAMA_ROPE_SCALING_TYPES.at(hparams.rope_scaling_type_train);

    // 打印模型的格式、架构、词汇表类型、词汇表大小和合并次数
    LLAMA_LOG_INFO("%s: format           = %s\n",     __func__, llama_file_version_name(ml.fver));
    LLAMA_LOG_INFO("%s: arch             = %s\n",     __func__, LLM_ARCH_NAMES.at(model.arch).c_str());
    LLAMA_LOG_INFO("%s: vocab type       = %s\n",     __func__, vocab.type == LLAMA_VOCAB_TYPE_SPM ? "SPM" : "BPE"); // TODO: fix
    LLAMA_LOG_INFO("%s: n_vocab          = %u\n",     __func__, hparams.n_vocab);
    LLAMA_LOG_INFO("%s: n_merges         = %u\n",     __func__, (int) vocab.bpe_ranks.size());
}
# 输出日志信息，显示训练上下文的数量
LLAMA_LOG_INFO("%s: n_ctx_train      = %u\n",     __func__, hparams.n_ctx_train);
# 输出日志信息，显示嵌入的维度
LLAMA_LOG_INFO("%s: n_embd           = %u\n",     __func__, hparams.n_embd);
# 输出日志信息，显示头的数量
LLAMA_LOG_INFO("%s: n_head           = %u\n",     __func__, hparams.n_head);
# 输出日志信息，显示键值头的数量
LLAMA_LOG_INFO("%s: n_head_kv        = %u\n",     __func__, hparams.n_head_kv);
# 输出日志信息，显示层数
LLAMA_LOG_INFO("%s: n_layer          = %u\n",     __func__, hparams.n_layer);
# 输出日志信息，显示旋转的数量
LLAMA_LOG_INFO("%s: n_rot            = %u\n",     __func__, hparams.n_rot); // a.k.a. n_embd_head, n_head_dim
# 输出日志信息，显示全局查询的数量
LLAMA_LOG_INFO("%s: n_gqa            = %u\n",     __func__, hparams.n_gqa());
# 输出日志信息，显示归一化的 epsilon 值
LLAMA_LOG_INFO("%s: f_norm_eps       = %.1e\n",   __func__, hparams.f_norm_eps);
# 输出日志信息，显示 RMS 归一化的 epsilon 值
LLAMA_LOG_INFO("%s: f_norm_rms_eps   = %.1e\n",   __func__, hparams.f_norm_rms_eps);
# 输出日志信息，显示键值查询的 clamp 值
LLAMA_LOG_INFO("%s: f_clamp_kqv      = %.1e\n",   __func__, hparams.f_clamp_kqv);
# 输出日志信息，显示最大的偏差值
LLAMA_LOG_INFO("%s: f_max_alibi_bias = %.1e\n",   __func__, hparams.f_max_alibi_bias);
# 输出日志信息，显示前馈网络的数量
LLAMA_LOG_INFO("%s: n_ff             = %u\n",     __func__, hparams.n_ff);
# 输出日志信息，显示绳子的缩放类型
LLAMA_LOG_INFO("%s: rope scaling     = %s\n",     __func__, rope_scaling_type.c_str());
# 输出日志信息，显示绳子的基础训练频率
LLAMA_LOG_INFO("%s: freq_base_train  = %.1f\n",   __func__, hparams.rope_freq_base_train);
# 输出日志信息，显示绳子的训练频率缩放
LLAMA_LOG_INFO("%s: freq_scale_train = %g\n",     __func__, hparams.rope_freq_scale_train);
# 输出日志信息，显示原始上下文的数量
LLAMA_LOG_INFO("%s: n_yarn_orig_ctx  = %u\n",     __func__, hparams.n_yarn_orig_ctx);
# 输出日志信息，显示绳子是否微调过
LLAMA_LOG_INFO("%s: rope_finetuned   = %s\n",     __func__, hparams.rope_finetuned ? "yes" : "unknown");
# 输出日志信息，显示模型类型
LLAMA_LOG_INFO("%s: model type       = %s\n",     __func__, llama_model_type_name(model.type));
# 输出日志信息，显示模型的特征类型
LLAMA_LOG_INFO("%s: model ftype      = %s\n",     __func__, llama_model_ftype_name(model.ftype).c_str());
# 输出日志信息，显示模型参数的数量
LLAMA_LOG_INFO("%s: model params     = %.2f B\n", __func__, ml.n_elements*1e-9);
// 如果模型大小小于1GB，则打印模型大小（以MiB为单位）和每个元素的位数
if (ml.n_bytes < GB) {
    LLAMA_LOG_INFO("%s: model size       = %.2f MiB (%.2f BPW) \n", __func__, ml.n_bytes/1024.0/1024.0, ml.n_bytes*8.0/ml.n_elements);
} 
// 如果模型大小大于等于1GB，则打印模型大小（以GiB为单位）和每个元素的位数
else {
    LLAMA_LOG_INFO("%s: model size       = %.2f GiB (%.2f BPW) \n", __func__, ml.n_bytes/1024.0/1024.0/1024.0, ml.n_bytes*8.0/ml.n_elements);
}

// 打印通用键值对的名称
LLAMA_LOG_INFO("%s: general.name   = %s\n",    __func__, model.name.c_str());

// 如果特殊起始标记的ID不为-1，则打印起始标记的ID和文本
if (vocab.special_bos_id != -1) { LLAMA_LOG_INFO( "%s: BOS token = %d '%s'\n", __func__, vocab.special_bos_id, vocab.id_to_token[vocab.special_bos_id].text.c_str() ); }
// 如果特殊结束标记的ID不为-1，则打印结束标记的ID和文本
if (vocab.special_eos_id != -1) { LLAMA_LOG_INFO( "%s: EOS token = %d '%s'\n", __func__, vocab.special_eos_id, vocab.id_to_token[vocab.special_eos_id].text.c_str() ); }
// 如果特殊未知标记的ID不为-1，则打印未知标记的ID和文本
if (vocab.special_unk_id != -1) { LLAMA_LOG_INFO( "%s: UNK token = %d '%s'\n", __func__, vocab.special_unk_id, vocab.id_to_token[vocab.special_unk_id].text.c_str() ); }
// 如果特殊分隔标记的ID不为-1，则打印分隔标记的ID和文本
if (vocab.special_sep_id != -1) { LLAMA_LOG_INFO( "%s: SEP token = %d '%s'\n", __func__, vocab.special_sep_id, vocab.id_to_token[vocab.special_sep_id].text.c_str() ); }
// 如果特殊填充标记的ID不为-1，则打印填充标记的ID和文本
if (vocab.special_pad_id != -1) { LLAMA_LOG_INFO( "%s: PAD token = %d '%s'\n", __func__, vocab.special_pad_id, vocab.id_to_token[vocab.special_pad_id].text.c_str() ); }
// 如果换行符的ID不为-1，则打印换行符的ID和文本
if (vocab.linefeed_id    != -1) { LLAMA_LOG_INFO( "%s: LF token  = %d '%s'\n", __func__, vocab.linefeed_id,    vocab.id_to_token[vocab.linefeed_id].text.c_str() );    }

// 打印稀疏推断的阈值
LLAMA_LOG_INFO("%s: sparse_pred_threshold = %.2f\n", __func__, hparams.sparse_pred_threshold);
// 定义了一个名为 llama_gpu_split_loader 的结构体
struct llama_gpu_split_loader {
    // 初始化了两个变量，分别表示张量数量和张量数据的字节数
    int n_tensors = 0;
    size_t n_bytes = 0; // tensor data bytes

    // 声明了一个名为 fname 的常量字符串
    const std::string fname;
    // 声明了一个名为 fver 的整数变量
    int fver;

    // 声明了一个名为 use_mmap 的布尔变量，并初始化为 false
    bool use_mmap = false; // only supports mmap yet
    // 声明了一个名为 mapping 的独特指针，指向 llama_mmap 类型的对象
    std::unique_ptr<llama_mmap> mapping;
    // 声明了一个指向 ggml_context 类型的指针，并初始化为 nullptr
    struct ggml_context * ctx_meta = nullptr;

    // 声明了一个名为 idx_loader 的指针，指向 llama_model_loader 类型的对象
    llama_model_loader * idx_loader;
    // 声明了一个名为 vram_required 的变量，表示所需的显存大小
    size_t vram_required = 0;

    // 定义了一个名为 llama_gpu_split_loader 的构造函数，接受文件名和是否使用 mmap 作为参数
    llama_gpu_split_loader(const std::string & fname, bool use_mmap) : fname(fname), use_mmap(use_mmap) {
        // 断言 use_mmap 为真
        GGML_ASSERT(use_mmap);
        // 为 idx_loader 分配了一个新的 llama_model_loader 对象
        idx_loader = new llama_model_loader(fname, use_mmap);
    }
```
// 使用 GGUF 获取指定索引的键值对，并将值存储到 vram_required 中
GGUF_GET_KEY(idx_loader->ctx_gguf, vram_required, gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_NAMES[LLM_KV_SPLIT_VRAM_CAPACITY]);
// 打印加载的 gpu_idx 和 vram_required 的数值
printf("loaded gpu_idx, vram_required: %ld\n", vram_required);

// 获取 idx_loader 中的 n_tensors 值
n_tensors = idx_loader->n_tensors;

// 为 MLP 张量分配内存数据/数据
// TODO: 支持为张量数据分配缓冲区（当不使用 mmap 时）
// 计算每个张量元数据的大小
size_t per_tensor_meta_size = GGML_PAD(sizeof(struct ggml_tensor), GGML_MEM_ALIGN) + GGML_OBJECT_SIZE;
// 计算所有张量元数据的总大小
size_t tensor_meta_size = n_tensors * per_tensor_meta_size;
// 初始化参数结构体
struct ggml_init_params params = {
    /*.mem_size   =*/ tensor_meta_size,  // 内存大小为张量元数据的总大小
    /*.mem_buffer =*/ nullptr,           // 内存缓冲区为空
    /*.no_alloc   =*/ true,              // 不进行分配内存
};
// 初始化上下文元数据
ctx_meta = ggml_init(params);

// 检查是否可以分配指定大小的 VRAM
bool check_vram_allocable(size_t vram_budget) {
    return vram_budget >= vram_required;  // 返回是否可以分配指定大小的 VRAM
}
    // 将张量应用到基础模型
    int apply_tensors_to_base_model(llama_model * model) {
        // 获取模型层数
        int n_layers = model->layers.size();
        // TODO: 断言文件指针在头部结束处
        if (n_tensors != n_layers * 2) {
           // 如果张量数量不等于层数乘以2，则输出错误信息并返回1
           LLAMA_LOG_ERROR("%s: error: the number of gpu splits does not match the layer of model\n", __func__);
            return 1;
        }
        // 输出信息，应用gpu_idx适配器，请等待...
        LLAMA_LOG_INFO("%s: applying gpu_idx adapter from '%s' - please wait ...\n", __func__, fname.c_str());
        // 获取开始时间
        const int64_t t_start_mlp_us = ggml_time_us();

        // 遍历每一层
        for (int il = 0; il < n_layers; il++) {
            // 获取模型的当前层
            llama_layer &model_layer = model->layers[il];
            // 获取当前层的gpu_idx张量和gpu_bucket张量
            ggml_tensor * gpu_idx = idx_loader->get_tensor_meta(il*2);
            ggml_tensor * gpu_bucket = idx_loader->get_tensor_meta(il*2+1);
            // 如果获取失败，则输出错误信息并返回1
            if (gpu_idx == nullptr || gpu_bucket == nullptr) {
                LLAMA_LOG_ERROR("%s: error: failed to load gpu index or bucket\n", __func__);
                return 1;
            }
            // 为模型层创建gpu_idx张量
            model_layer.gpu_idx = idx_loader->create_tensor_for(ctx_meta, gpu_idx, GGML_BACKEND_CPU);
// 设置模型层的 GPU 存储桶
model_layer.gpu_bucket = idx_loader->create_tensor_for(ctx_meta, gpu_bucket, GGML_BACKEND_GPU);
// 定义进度回调函数，输出进度信息
llama_progress_callback cb = [](float progress, void *ctx) {
    LLAMA_LOG_INFO(".");
};
// 加载所有数据，并传入进度回调函数
idx_loader->load_all_data(ctx_meta, cb, nullptr, nullptr);
// 计算加载模型所花费的时间
const int64_t t_mlp_us = ggml_time_us() - t_start_mlp_us;
// 输出加载模型所花费的时间
LLAMA_LOG_INFO(" done (%.2f ms)\n", t_mlp_us / 1000.0);
// 返回加载结果
return 0;
}

// 用于动态加载/转换 llama 模型权重的结构
struct llama_augmentation_model_loader {
    // 辅助上下文指针
    struct ggml_context * aux_ctx = nullptr;

    // 构造函数，接受一个 llama_model 指针作为参数
    llama_augmentation_model_loader(llama_model *model) {
        // TODO: 检查前置条件 - MLP 已加载
        // 检查需要加载的增强字段
        // 1. gpu_idx;
        // 2. gpu_bucket;
        // 3. transformed ffn_down;
        // 计算 ggml_aux_tensor_size 的值
        const int64_t ggml_aux_tensor_size = 4 * (100 * 100 + 5120*40*4 * ggml_tensor_overhead() + (int64_t)13824*5120*40*4);
        
        // 获取模型层数
        int model_layer = model->layers.size();
        // 获取模型第一层的 ffn_up 的维度
        int ffn_dim = model->layers[0].ffn_up->ne[1];
        // 计算 ggml_aux_tensor_size 的值
        const size_t ggml_aux_tensor_size = 4 * (model_layer*ffn_dim*sizeof(float)*2+ model_layer*ffn_dim*sizeof(float) * ggml_tensor_overhead() );

        // 初始化 ggml_init_params 结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ ggml_aux_tensor_size,
            /*.mem_buffer =*/ nullptr,
            /*.no_alloc   =*/ false,
        };
        // 调用 ggml_init 函数进行初始化
        aux_ctx = ggml_init(params);
    }

    // 创建将带状矩阵传输到 GPU 的函数
    ggml_tensor * create_striped_mat_to_gpu(const struct ggml_tensor *src, struct ggml_tensor * gpu_bucket) {
        // 检查源矩阵是否为空
        if (src == NULL) {
        // 如果条件不满足，返回空指针
        return NULL;
        // 分配内存并将选定的权重复制到 GPU
#ifdef GGML_USE_CUBLAS
        // 获取源张量的行长度和 GPU 桶的行数
        int64_t row_len = src->ne[0];
        int64_t gpu_rows = gpu_bucket->ne[0];
        // 如果 GPU 行数为 0，则返回空指针
        if (gpu_rows == 0)
            return NULL;
        
        // 设置不分配辅助上下文
        ggml_set_no_alloc(aux_ctx, true);
        // 在 GPU 上创建新的二维张量
        ggml_tensor * gpu_dst = ggml_new_tensor_2d(aux_ctx, src->type, row_len, gpu_rows);
        // 设置张量的后端为 GPU
        ggml_set_backend(gpu_dst, GGML_BACKEND_GPU);
        // 在 GPU 上分配张量
        ggml_cuda_alloc_tensor(gpu_dst);

        // 在主机和设备上初始化两个一维视图
        ggml_tensor * host_mat_row = ggml_new_tensor_1d(aux_ctx, src->type, row_len);
        // 在设备上创建静态一维张量
        static ggml_tensor * device_mat_row = ggml_dup_tensor(aux_ctx, host_mat_row);
        // 设置张量的后端为 GPU
        ggml_set_backend(device_mat_row, GGML_BACKEND_GPU);
        // 在 GPU 上分配张量
        ggml_cuda_alloc_tensor(device_mat_row);
        // 将设备一维张量的数据指针指向 GPU 目标张量的数据指针
        *ggml_cuda_get_data_pp(device_mat_row) = *ggml_cuda_get_data_pp(gpu_dst);
// 根据不同的gpu_idx，读取原始数据并将其复制到设备上
const enum ggml_type type = src->type; // 获取源数据的类型
const int ne0 = src->ne[0]; // 获取源数据的ne[0]值
const size_t row_data_size = ne0*ggml_type_size(type)/ggml_blck_size(type); // 计算每行数据的大小
for (int i = 0; i < gpu_rows; i++) { // 遍历gpu_rows次
    int32_t host_i = ((int32_t *)gpu_bucket->data)[i]; // 获取host_i的值
    host_mat_row -> data = src -> data + host_i * row_data_size; // 计算host_mat_row的数据位置
    char ** gpu_data_pp = reinterpret_cast<char **>(ggml_cuda_get_data_pp(device_mat_row)); // 获取gpu_data_pp的值
    ggml_cuda_cpy_1d(device_mat_row, host_mat_row); // 将host_mat_row的数据复制到device_mat_row
    *gpu_data_pp = *gpu_data_pp + row_data_size; // 更新gpu_data_pp的值
}
ggml_set_no_alloc(aux_ctx, false); // 设置aux_ctx的no_alloc为false

return gpu_dst; // 返回gpu_dst
#else
return NULL; // 如果不满足条件，返回NULL
#endif
}
// 将层的 ffnn 网络参数数据切片并传输到 GPU
size_t slice_ffn_mat_to_gpu(llama_layer & layer) {
    // 创建一个用于临时存储数据的缓冲区
    std::vector<uint8_t> work_buffer;
    // 获取 GPU 索引和 GPU 存储桶
    ggml_tensor * gpu_idx = layer.gpu_idx;
    ggml_tensor *gpu_bucket = layer.gpu_bucket;
    // 初始化已传输的字节数
    size_t offloaded_bytes = 0;

    // 将 ffnn 网络参数数据传输到 GPU
    layer.ffn_gate_gpu = create_striped_mat_to_gpu(layer.ffn_gate, gpu_bucket);
    layer.ffn_up_gpu = create_striped_mat_to_gpu(layer.ffn_up, gpu_bucket);
    layer.ffn_down_gpu = create_striped_mat_to_gpu(layer.ffn_down_t, gpu_bucket);
    
    // 如果 ffnn 网络参数数据成功传输到 GPU，则增加已传输的字节数
    if (layer.ffn_gate_gpu) {
        offloaded_bytes += ggml_nbytes(layer.ffn_gate_gpu);
    }
    if (layer.ffn_up_gpu) {
        offloaded_bytes += ggml_nbytes(layer.ffn_up_gpu);
    }
    if (layer.ffn_down_gpu) {
        offloaded_bytes += ggml_nbytes(layer.ffn_down_gpu);
    }
        // 返回 offloaded_bytes 变量的值
        return offloaded_bytes;
    }

    // 将模型分片并加载到 GPU
    size_t offload_ffn_split(llama_model * model) {
        // 打印信息
        LLAMA_LOG_INFO("%s: applying augmentation to model - please wait ...\n", __func__);
        // 获取开始时间
        const int64_t t_start_aug_us = ggml_time_us();
        // 创建一个存储工作数据的缓冲区
        std::vector<uint8_t> work_buffer;

        // 通过全局变量设置稀疏阈值
        sparse_pred_threshold = model->hparams.sparse_pred_threshold;
        // 如果使用了CUBLAS，则设置设备常量
#if defined (GGML_USE_CUBLAS)
        ggml_cuda_set_device_constants(model->hparams.sparse_pred_threshold);
#endif

        // 加载gpu_idx和slice mat到GPU
        size_t offloaded_bytes = 0;
        // 遍历模型的每一层
        for (llama_layer &model_layer : model -> layers) {
            // 加载gpu_idx
            if (model_layer.gpu_idx == NULL && model_layer.gpu_bucket == NULL) {
                // 创建一个新的1维张量
                ggml_tensor * gpu_idx = ggml_new_tensor_1d(aux_ctx, GGML_TYPE_I32, model_layer.mlp_pre_w2 -> ne[1]);
// 将 GPU 索引设置为零
ggml_set_zero(gpu_idx);
// 设置模型层的 GPU 索引
model_layer.gpu_idx = gpu_idx;
// 创建一个新的一维整型张量，用于 GPU 存储
ggml_tensor * gpu_bucket = ggml_new_tensor_1d(aux_ctx, GGML_TYPE_I32, 0);
// 将新创建的 GPU 张量赋值给模型层的 GPU 存储桶
model_layer.gpu_bucket = gpu_bucket;

// 将模型层的权重矩阵切片传输到 GPU 上，并返回传输的字节数
offloaded_bytes += slice_ffn_mat_to_gpu(model_layer);
// 记录日志信息
LLAMA_LOG_INFO(".");

// 输出日志信息，显示完成并计算所花费的时间
LLAMA_LOG_INFO(" done (%.2f ms)\n", (ggml_time_us() - t_start_aug_us) / 1000.0);
// 返回已传输的字节数
return offloaded_bytes;
}

// 定义一个结构体 buffered_tensor_allocator
struct buffered_tensor_allocator {
    // 引用模型加载器和上下文
    llama_model_loader &ml;
    ggml_context *ctx;
    // 存储不同级别的张量分配队列
    std::map<tensor_offloading_levels, std::vector<ggml_tensor *>> alloc_queues;
    // VRAM 预算字节数
    const size_t vram_budget_bytes;
    // 已分配的 VRAM 字节数
    size_t vram_allocated_bytes = 0;
// 创建一个 buffered_tensor_allocator 类的构造函数，接受 llama_model_loader 和 ggml_context 对象以及 vram_budget_bytes 参数
buffered_tensor_allocator(llama_model_loader &ml, ggml_context *ctx, size_t vram_budget_bytes) : ctx(ctx), ml(ml), vram_budget_bytes(vram_budget_bytes) {}

// 分配一个缓冲区张量，接受名称、张量的迁移级别和张量的形状作为参数
ggml_tensor * buffered_alloc(const std::string & name, const tensor_offloading_levels &level, const std::vector<int64_t> & ne) {
    // 如果使用了 GGML_USE_CUBLAS 宏定义
#if defined(GGML_USE_CUBLAS)
        // 如果张量的迁移级别是 TENSOR_NO_OFFLOAD 或 TENSOR_OFFLOAD_FFN，则在 CPU 上创建张量
        if (level == TENSOR_NO_OFFLOAD || level == TENSOR_OFFLOAD_FFN) {
            return ml.create_tensor(ctx, name, ne, GGML_BACKEND_CPU);
        }
        // 为 GPU 张量仅分配元数据
        bool no_alloc = ctx->no_alloc;
        ggml_set_no_alloc(ctx, true);
        ggml_tensor * meta_tensor = ml.create_tensor(ctx, name, ne, GGML_BACKEND_CPU);
        ggml_set_no_alloc(ctx, no_alloc);
        alloc_queues[level].push_back(meta_tensor);
        return meta_tensor;
    // 如果没有定义 GGML_USE_CUBLAS 宏
#else
        // 在 CPU 上创建张量
        return ml.create_tensor(ctx, name, ne, GGML_BACKEND_CPU);
#endif
    }
    // 对于 GPU 张量，我们需要尽可能在 VRAM 中分配它们，并原地更新张量数据。如果超出了 VRAM 预算，我们就在 CPU 内存中分配张量。
    size_t flush() {
#if defined(GGML_USE_CUBLAS)
        // 遍历离线优先级
        for (int enum_i = TENSOR_OFFLOAD_ATTN; enum_i <= TENSOR_OFFLOAD_KV_CACHE; enum_i ++) {
            tensor_offloading_levels level = static_cast<tensor_offloading_levels>(enum_i);
            for (ggml_tensor * meta_tensor : alloc_queues[level]) {
                // 获取张量数据大小
                size_t tensor_data_size = ggml_nbytes(meta_tensor);
                // 如果 VRAM 已分配的字节数加上张量数据大小超出了 VRAM 预算，就返回已分配的 VRAM 字节数
                if (vram_allocated_bytes + tensor_data_size > vram_budget_bytes) {
                    return vram_allocated_bytes;
                }
                // 在 VRAM 中分配内存
                ggml_set_backend(meta_tensor, GGML_BACKEND_GPU);
                vram_allocated_bytes += tensor_data_size;
            }
        }
#endif
        // 返回已分配的 VRAM 字节数
        return vram_allocated_bytes;
// 从分割文件中加载 GPU 分割数据到模型中
static bool load_gpu_split_from_split_file(llama_model & model, std::string split_path, size_t vram_budget) {
    // 创建 GPU 分割加载器对象
    llama_gpu_split_loader loader(split_path, true);
    // 检查是否可以分配指定大小的 VRAM，并将张量应用到基础模型中
    return loader.check_vram_allocable(vram_budget) 
        && loader.apply_tensors_to_base_model(&model) == 0;
}

// 根据预算加载 GPU 分割数据
static bool llm_load_gpu_split_with_budget(llama_model_loader & ml, llama_model & model, size_t vram_allocatable_bytes, bool no_cache) {
    // 获取模型文件路径
    const char * model_path = ml.file.fname.c_str();
    // 生成缓存分割文件路径
    std::string cached_split_path = std::string(model_path) + ".generated.gpuidx";
    // 获取模型文件所在目录
    const char * model_basedir = dirname(const_cast<char *>(model_path));

    // 从先前生成的缓存中加载 GPU 分割数据
    if (access(cached_split_path.c_str(), F_OK) == 0 && !no_cache) {
        // 如果成功加载 GPU 分割数据，则返回 true
        if (load_gpu_split_from_split_file(model, cached_split_path, vram_allocatable_bytes)) {
            return true;
        }
        // 如果加载失败，则记录错误日志
        LLAMA_LOG_ERROR("%s: error: failed to apply previously generated gpu split from '%s'\n", __func__, cached_split_path.c_str());
    }

    // 生成 GPU 分割
    // 创建激活路径字符串
    std::string activation_path = std::string(model_basedir) + "/activation";
    // 检查激活文件是否存在
    if (access(activation_path.c_str(), F_OK) != 0) {
        // 如果激活文件不存在，记录错误信息并返回 false
        LLAMA_LOG_ERROR("%s: error: activation files under '%s' not found\n", __func__, activation_path.c_str());
        return false;
    }

    // 计算求解器参数
    ggml_tensor * ffn_up = model.layers[0].ffn_up;
    ggml_tensor * ffn_gate = model.layers[0].ffn_gate;
    // 计算切片大小
    int slice_size = ffn_up->ne[1] * ggml_type_size(ffn_up->type) / ggml_blck_size(ffn_up->type);
    // 对于具有 FFN gate 的模型架构，门也被切片，否则只有上下矩阵被切片
    int vram_bytes_per_slice = slice_size * (ffn_gate ? 4.5 : 2); // TODO: why 4.5, not 3?
    // 计算神经元容量
    int neuron_cap = floor((double)vram_allocatable_bytes / vram_bytes_per_slice) * 4;

    // 记录信息，调用 powerinfer Python 模块生成 GPU 分割
    LLAMA_LOG_INFO("invoking powerinfer Python module to generate gpu split for %.2f MiB of VRAM\n", vram_allocatable_bytes / 1024.0 / 1024.0);

    // 创建命令流
    std::stringstream command_ss;
    // 构建命令字符串，调用 powerinfer 模块进行计算
    command_ss << "python3 -m powerinfer"
               << " --activation " << activation_path
               << " --layer " << model.hparams.n_layer
               << " --neuron " << ffn_up->ne[1]
               << " --capacity " << neuron_cap
               << " --vram-capacity " << vram_allocatable_bytes
               << " --output " << cached_split_path;
    // 如果调用命令失败或者生成的缓存文件不存在，则记录错误并返回 false
    if (system(command_ss.str().c_str()) != 0 || access(cached_split_path.c_str(), F_OK) != 0) {
        LLAMA_LOG_ERROR("%s: error: failed to generate gpu split\n", __func__);
        return false;
    }
    // 调用成功后，加载 GPU 分割文件
    return load_gpu_split_from_split_file(model, cached_split_path, vram_allocatable_bytes);
}

// 加载 GPU 分割文件的静态函数
static void llm_load_gpu_split(llama_model_loader & ml, llama_model & model, size_t vram_budget_bytes, bool no_cache, bool no_offload) {
    // 如果 VRAM 预算大于等于 512MB 并且不禁用离线计算
    if (vram_budget_bytes >= 512ull * 1024 * 1024 && !no_offload) {
        // 减去 512MB 作为安全边界
        vram_budget_bytes -= 512ull * 1024 * 1024;
        // 如果加载 GPU 分割文件失败
        if (!llm_load_gpu_split_with_budget(ml, model, vram_budget_bytes, no_cache)) {
// 记录错误日志，指示生成 GPU 分割失败，将使用空的分割
LLAMA_LOG_ERROR("%s: error: failed to generate gpu split, an empty one will be used\n", __func__);
}

// 如果定义了条件编译符号，则执行以下代码
#ifdef
// 将 FFN 权重数据分配到 GPU 上
size_t ffn_offloaded_bytes = llama_model_offload_ffn_split(&model);
// 记录信息日志，指示成功将多少 MiB 的 FFN 权重数据分配到 GPU 上
LLAMA_LOG_INFO("%s: offloaded %.2f MiB of FFN weights to GPU\n", __func__, ffn_offloaded_bytes / 1024.0 / 1024.0);
}

// 加载稀疏模型张量数据
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
    // 记录加载模型张量数据的起始时间
    model.t_start_us = ggml_time_us();
    // 获取模型的上下文和超参数
    auto & ctx     = model.ctx;
    auto & hparams = model.hparams;

    // 计算上下文和内存映射的大小
    size_t ctx_size;
    size_t mmapped_size;
    ml.calc_sizes(ctx_size, mmapped_size);
    // 打印上下文大小的日志信息
    LLAMA_LOG_INFO("%s: ggml ctx size = %7.2f MB\n", __func__, ctx_size/1024.0/1024.0);

    // 创建 ggml 上下文
    {
        // 调整模型缓冲区的大小
        model.buf.resize(ctx_size);
        // 如果使用 mlock，则初始化和扩展 mlock 缓冲区
        if (use_mlock) {
            model.mlock_buf.init   (model.buf.data);
            model.mlock_buf.grow_to(model.buf.size);
        }

        // 初始化 ggml 的参数
        struct ggml_init_params params = {
            /*.mem_size   =*/ model.buf.size,
            /*.mem_buffer =*/ model.buf.data,
            /*.no_alloc   =*/ ml.use_mmap,
    };

    // 初始化模型上下文
    model.ctx = ggml_init(params);
    // 如果模型上下文初始化失败，抛出运行时错误
    if (!model.ctx) {
        throw std::runtime_error(format("ggml_init() failed"));
    }
}

// 忽略 main_gpu

// 设置 LLAMA 后端为 CPU
enum ggml_backend_type llama_backend_offload = GGML_BACKEND_CPU;
// 设置 LLAMA 分割后端为 CPU
enum ggml_backend_type llama_backend_offload_split = GGML_BACKEND_CPU;
// 设置 VRAM 容量为 0
size_t vram_capacity = 0;

#ifdef GGML_USE_CUBLAS
// 如果加载了 ggml_cublas 库
if (ggml_cublas_loaded()) {
    // 输出信息，使用 GGML_CUDA_NAME 进行 GPU 加速
    LLAMA_LOG_INFO("%s: using " GGML_CUDA_NAME " for GPU acceleration\n", __func__);
    // 设置主 GPU 设备
    ggml_cuda_set_main_device(main_gpu);

    // 设置 LLAMA 后端为 GPU
    llama_backend_offload = GGML_BACKEND_GPU;
// 如果定义了 GGML_BACKEND_GPU_SPLIT，则将 llama_backend_offload_split 设置为 GGML_BACKEND_GPU_SPLIT
llama_backend_offload_split = GGML_BACKEND_GPU_SPLIT;
}

// 如果定义了 GGML_USE_CLBLAST，则打印信息并将 llama_backend_offload 和 llama_backend_offload_split 设置为 GGML_BACKEND_GPU
#elif defined(GGML_USE_CLBLAST)
    LLAMA_LOG_INFO("%s: using OpenCL for GPU acceleration\n", __func__);
    llama_backend_offload = GGML_BACKEND_GPU;
    llama_backend_offload_split = GGML_BACKEND_GPU;
#endif

// 如果定义了 GGML_USE_CUBLAS
#if defined(GGML_USE_CUBLAS)
    // 如果 vram_budget_bytes 小于 0，则将 vram_capacity 设置为主 GPU 的剩余内存
    if (vram_budget_bytes < 0) {
        // 让其等于 VRAM 的剩余空间
        vram_capacity = ggml_cuda_get_free_memory(main_gpu);
    } else {
        // 否则将 vram_capacity 设置为 vram_budget_bytes 和主 GPU 的剩余内存中较小的一个
        vram_capacity = std::min(vram_budget_bytes, (long long) ggml_cuda_get_free_memory(main_gpu));
    }
#endif

// 使用 vram_capacity 创建一个缓冲张量分配器
buffered_tensor_allocator alloc(ml, ctx, vram_capacity);
// 创建一个 lambda 函数 create_tensor，使用 alloc 作为参数
auto create_tensor = [&alloc] (
    const std::pair<std::string, tensor_offloading_levels> & tn, 
// 定义一个lambda函数，接受ne作为参数，返回一个ggml_tensor指针
const std::vector<int64_t> & ne) -> ggml_tensor * {
    // 调用alloc对象的buffered_alloc方法，传入tn.first, tn.second, ne作为参数，返回结果
    return alloc.buffered_alloc(tn.first, tn.second, ne);
};

{
    // 获取hparams对象的n_embd属性值，赋给n_embd变量
    const int64_t n_embd     = hparams.n_embd;
    // 调用hparams对象的n_embd_gqa方法，将返回值赋给n_embd_gqa变量
    const int64_t n_embd_gqa = hparams.n_embd_gqa();
    // 获取hparams对象的n_layer属性值，赋给n_layer变量
    const int64_t n_layer    = hparams.n_layer;
    // 获取hparams对象的n_vocab属性值，赋给n_vocab变量
    const int64_t n_vocab    = hparams.n_vocab;

    // 调用LLM_TN函数，传入model.arch作为参数，返回结果赋给tn变量
    const auto tn = LLM_TN(model.arch);
    // 根据model.arch的值进行switch分支
    switch (model.arch) {
        // 如果model.arch的值是LLM_ARCH_LLAMA或LLM_ARCH_REFACT
        case LLM_ARCH_LLAMA:
        case LLM_ARCH_REFACT:
            {
                // 创建model.tok_embd张量，调用create_tensor方法，传入tn(LLM_TENSOR_TOKEN_EMBD, "weight")和{n_embd, n_vocab}作为参数
                model.tok_embd = create_tensor(tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab});

                // output
                {
                    // 创建model.output_norm张量，调用create_tensor方法，传入tn(LLM_TENSOR_OUTPUT_NORM, "weight")和{n_embd}作为参数
                    model.output_norm = create_tensor(tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd});
# 设置模型的输出张量
model.output = create_tensor(tn(LLM_TENSOR_OUTPUT, "weight"), {n_embd, n_vocab});

# 获取全连接层的数量
const uint32_t n_ff = hparams.n_ff;

# 调整模型的层数
model.layers.resize(n_layer);

# 遍历每一层
for (uint32_t i = 0; i < n_layer; ++i) {
    auto & layer = model.layers[i];

    # 创建注意力层的归一化张量
    layer.attn_norm = create_tensor(tn(LLM_TENSOR_ATTN_NORM, "weight", i), {n_embd});

    # 创建注意力层的查询张量
    layer.wq = create_tensor(tn(LLM_TENSOR_ATTN_Q, "weight", i), {n_embd, n_embd});

    # 创建注意力层的键张量
    layer.wk = create_tensor(tn(LLM_TENSOR_ATTN_K, "weight", i), {n_embd, n_embd_gqa});

    # 创建注意力层的值张量
    layer.wv = create_tensor(tn(LLM_TENSOR_ATTN_V, "weight", i), {n_embd, n_embd_gqa});

    # 创建注意力层的输出张量
    layer.wo = create_tensor(tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd});

    # 创建全连接层的归一化张量
    layer.ffn_norm = create_tensor(tn(LLM_TENSOR_FFN_NORM, "weight", i), {n_embd});

    # 创建全连接层的门控张量
    layer.ffn_gate = create_tensor(tn(LLM_TENSOR_FFN_GATE, "weight", i), {n_embd, n_ff});

    # 创建全连接层的下采样张量
    layer.ffn_down_t = create_tensor(tn(LLM_TENSOR_FFN_DOWN_T, "weight", i), {n_embd, n_ff});
}
// 创建一个名为layer.mlp_pre_w1的张量，用于存储MLP预测层的权重，形状为{n_embd, GGML_NE_WILDCARD}
layer.mlp_pre_w1 = create_tensor(tn(LLM_TENSOR_MLP_PRED_FC1, "weight", i), {n_embd, GGML_NE_WILDCARD});
// 创建一个名为layer.mlp_pre_w2的张量，用于存储MLP预测层的权重，形状为{GGML_NE_WILDCARD, n_ff}
layer.mlp_pre_w2 = create_tensor(tn(LLM_TENSOR_MLP_PRED_FC2, "weight", i), {GGML_NE_WILDCARD, n_ff});
// 创建一个名为layer.ffn_up的张量，用于存储FFN层的权重，形状为{n_embd, n_ff}
layer.ffn_up   = create_tensor(tn(LLM_TENSOR_FFN_UP,   "weight", i), {n_embd,   n_ff});

// 当LLM_ARCH为FALCON时
case LLM_ARCH_FALCON:
{
    // 创建一个名为model.tok_embd的张量，用于存储token嵌入的权重，形状为{n_embd, n_vocab}
    model.tok_embd = create_tensor(tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab});

    // 创建输出相关的张量
    {
        // 创建一个名为model.output_norm的张量，用于存储输出的归一化权重，形状为{n_embd}
        model.output_norm   = create_tensor(tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd});
        // 创建一个名为model.output_norm_b的张量，用于存储输出的归一化偏置，形状为{n_embd}
        model.output_norm_b = create_tensor(tn(LLM_TENSOR_OUTPUT_NORM, "bias"),   {n_embd});
        // 创建一个名为model.output的张量，用于存储输出的权重，形状为{n_embd, n_vocab}
        model.output        = create_tensor(tn(LLM_TENSOR_OUTPUT,      "weight"), {n_embd, n_vocab});
    }

    // 获取hparams中的n_ff值
    const uint32_t n_ff = hparams.n_ff;

    // 调整模型的层数为n_layer
    model.layers.resize(n_layer);
}
// 遍历每个层，i 从 0 到 n_layer-1
for (uint32_t i = 0; i < n_layer; ++i) {
    // 获取当前层的引用
    auto & layer = model.layers[i];

    // 创建注意力归一化的权重和偏置张量
    layer.attn_norm   = create_tensor(tn(LLM_TENSOR_ATTN_NORM,   "weight", i), {n_embd});
    layer.attn_norm_b = create_tensor(tn(LLM_TENSOR_ATTN_NORM,   "bias", i),   {n_embd});

    // 如果存在指定名称的张量，则创建第二个注意力归一化的权重和偏置张量
    if (gguf_find_tensor(ml.ctx_gguf, tn(LLM_TENSOR_ATTN_NORM_2, "weight", i).first.c_str()) >= 0) {
        layer.attn_norm_2   = create_tensor(tn(LLM_TENSOR_ATTN_NORM_2, "weight", i), {n_embd});
        layer.attn_norm_2_b = create_tensor(tn(LLM_TENSOR_ATTN_NORM_2, "bias", i),   {n_embd});
    }

    // 创建注意力权重张量、输出权重张量、下游前馈网络张量、预测网络第一层权重张量、预测网络第二层权重张量、上游前馈网络张量
    layer.wqkv = create_tensor(tn(LLM_TENSOR_ATTN_QKV, "weight", i), {n_embd, n_embd + 2*n_embd_gqa});
    layer.wo   = create_tensor(tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd});
    layer.ffn_down_t = create_tensor(tn(LLM_TENSOR_FFN_DOWN_T, "weight", i), {n_embd, n_ff});
    layer.mlp_pre_w1 = create_tensor(tn(LLM_TENSOR_MLP_PRED_FC1, "weight", i), {n_embd, GGML_NE_WILDCARD});
    layer.mlp_pre_w2 = create_tensor(tn(LLM_TENSOR_MLP_PRED_FC2, "weight", i), {GGML_NE_WILDCARD, n_ff});
    layer.ffn_up   = create_tensor(tn(LLM_TENSOR_FFN_UP,   "weight", i), {n_embd,   n_ff});
}
    // 如果架构未知，则抛出运行时错误
    throw std::runtime_error("unknown architecture");
    }

    // 获取已分配的 VRAM 字节数
    size_t vram_allocated_bytes = alloc.flush();
    // 断言已分配的 VRAM 字节数不超过 VRAM 容量
    GGML_ASSERT_DBG(vram_allocated_bytes <= vram_capacity, "vram_allocated_bytes=%ld, vram_capacity=%ld", vram_allocated_bytes, vram_capacity);
    // 完成获取张量
    ml.done_getting_tensors();

    // 打印内存需求
    {
        // 运行推理所需的总内存
        size_t mem_required =
            ctx_size +
            mmapped_size - vram_allocated_bytes; // VRAM 中的权重不在内存中

        // 打印函数名和所需内存量（以 MB 为单位）
        LLAMA_LOG_INFO("%s: mem required  = %7.2f MB\n", __func__, mem_required / 1024.0 / 1024.0);

        // 如果使用了 CUBLAS 或 CLBLAST，则打印已使用的 VRAM（以 MB 为单位）
#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST)
        LLAMA_LOG_INFO("%s: VRAM used: %.2f MB\n", __func__, vram_allocated_bytes / 1024.0 / 1024.0);
#endif
    // populate `tensors_by_name`
    // 遍历所有张量，将张量名和张量对象存入model.tensors_by_name中

    for (int i = 0; i < ml.n_tensors; ++i) {
        // 获取当前张量对象
        struct ggml_tensor * cur = ggml_get_tensor(ctx, ml.get_tensor_name(i));
        // 将张量名和张量对象存入model.tensors_by_name中
        model.tensors_by_name.emplace_back(ggml_get_name(cur), cur);
    }

    // 加载所有数据
    ml.load_all_data(ctx, progress_callback, progress_callback_user_data, use_mlock ? &model.mlock_mmap : NULL);

    // 如果有进度回调函数，则调用进度回调函数
    if (progress_callback) {
        progress_callback(1.0f, progress_callback_user_data);
    }

    // 将ml.mapping移动到model.mapping中
    model.mapping = std::move(ml.mapping);

    // 如果可能的话，将FFN段加载到GPU上
    llm_load_gpu_split(ml, model, vram_capacity - vram_allocated_bytes, reset_gpu_index, disable_ffn_split);

    // 加载时间将在第一次评估后重新计算
// 计算延迟加载时间，即从开始加载到完成加载所经过的时间
model.t_load_us = ggml_time_us() - model.t_start_us;

model.n_gpu_layers = -1; // TODO: 根据离线结果，根据类别确定GPU层数？
}

// 加载张量数据
static void llm_load_tensors(
        llama_model_loader & ml,
        llama_model & model,
        int n_gpu_layers,
        int main_gpu,
        const float * tensor_split,
        bool use_mlock,
        llama_progress_callback progress_callback,
        void * progress_callback_user_data) {
    // 记录加载开始时间
    model.t_start_us = ggml_time_us();

    // 获取模型上下文和超参数
    auto & ctx     = model.ctx;
    auto & hparams = model.hparams;
    // 设置模型的 GPU 层数
    model.n_gpu_layers = n_gpu_layers;

    // 声明上下文大小和内存映射大小
    size_t ctx_size;
    size_t mmapped_size;

    // 计算上下文大小和内存映射大小
    ml.calc_sizes(ctx_size, mmapped_size);

    // 输出上下文大小的日志信息
    LLAMA_LOG_INFO("%s: ggml ctx size = %7.2f MB\n", __func__, ctx_size/1024.0/1024.0);

    // 创建 ggml 上下文
    {
        // 调整模型缓冲区大小
        model.buf.resize(ctx_size);
        // 如果使用 mlock，则初始化和扩展 mlock 缓冲区
        if (use_mlock) {
            model.mlock_buf.init   (model.buf.data);
            model.mlock_buf.grow_to(model.buf.size);
        }

        // 设置 ggml 初始化参数
        struct ggml_init_params params = {
            /*.mem_size   =*/ model.buf.size,
            /*.mem_buffer =*/ model.buf.data,
    /* 设置参数，使用内存映射 */
    params = (struct ggml_params) {
        .no_alloc   = ml.use_mmap,
    };

    /* 初始化模型上下文 */
    model.ctx = ggml_init(params);
    /* 如果模型上下文初始化失败，抛出运行时错误 */
    if (!model.ctx) {
        throw std::runtime_error(format("ggml_init() failed"));
    }

    /* 设置主 GPU 设备 */
    (void) main_gpu;

    /* 设置 LLAMA 后端为 CPU */
    enum ggml_backend_type llama_backend_offload = GGML_BACKEND_CPU;
    enum ggml_backend_type llama_backend_offload_split = GGML_BACKEND_CPU;

    /* 如果使用 CUBLAS，设置 LLAMA 后端为 GPU，并打印信息 */
    #ifdef GGML_USE_CUBLAS
    if (ggml_cublas_loaded()) {
        LLAMA_LOG_INFO("%s: using " GGML_CUDA_NAME " for GPU acceleration\n", __func__);
        ggml_cuda_set_main_device(main_gpu);
        llama_backend_offload = GGML_BACKEND_GPU;
    }
    // 如果定义了 GGML_USE_CUDA，则使用 CUDA 进行 GPU 加速
    llama_backend_offload = GGML_BACKEND_GPU;
    // 如果定义了 GGML_USE_CUDA，则使用 CUDA 进行 GPU 加速
    llama_backend_offload_split = GGML_BACKEND_GPU_SPLIT;
#elif defined(GGML_USE_CLBLAST)
    // 如果定义了 GGML_USE_CLBLAST，则使用 OpenCL 进行 GPU 加速
    LLAMA_LOG_INFO("%s: using OpenCL for GPU acceleration\n", __func__);
    llama_backend_offload = GGML_BACKEND_GPU;
    llama_backend_offload_split = GGML_BACKEND_GPU;
#endif

    // 为权重准备内存
    size_t vram_weights = 0;
    {
        // 获取模型参数
        const int64_t n_embd     = hparams.n_embd;
        const int64_t n_embd_gqa = hparams.n_embd_gqa();
        const int64_t n_layer    = hparams.n_layer;
        const int64_t n_vocab    = hparams.n_vocab;

        // 根据模型架构选择内存分配策略
        const auto tn = LLM_TN(model.arch);
        switch (model.arch) {
            case LLM_ARCH_LLAMA:
            case LLM_ARCH_REFACT:
// 创建一个名为tok_embd的模型张量，使用ml.create_tensor函数，传入上下文ctx、张量名称和大小参数{n_embd, n_vocab}，使用GGML_BACKEND_CPU作为后端
model.tok_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab}, GGML_BACKEND_CPU);

// 输出
{
    // 声明变量backend_norm和backend_output
    ggml_backend_type backend_norm;
    ggml_backend_type backend_output;

    // 如果GPU层数大于层数
    if (n_gpu_layers > int(n_layer)) {
        // norm在性能上不重要，但在VRAM中保留它可以减少数据复制
        // 但在Windows上，除非所有内容都在GPU上，否则这对性能有害
#ifndef _WIN32
        backend_norm = llama_backend_offload;
#else
        backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#endif // _WIN32

        // 使用llama_backend_offload_split作为输出后端
        backend_output = llama_backend_offload_split;
    } else {
        // 使用GGML_BACKEND_CPU作为norm的后端
        backend_norm   = GGML_BACKEND_CPU;
# 设置后端输出为 CPU
backend_output = GGML_BACKEND_CPU;

# 创建模型的输出归一化张量
model.output_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd}, backend_norm);
# 创建模型的输出张量
model.output = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT, "weight"), {n_embd, n_vocab}, backend_output);

# 如果归一化的后端是 GPU，则计算 VRAM 权重
if (backend_norm == GGML_BACKEND_GPU) {
    vram_weights += ggml_nbytes(model.output_norm);
}
# 如果输出的后端是 GPU 分割，则计算 VRAM 权重
if (backend_output == GGML_BACKEND_GPU_SPLIT) {
    vram_weights += ggml_nbytes(model.output);
}

# 设置前馈神经网络的层数
const uint32_t n_ff = hparams.n_ff;

# 计算 GPU 起始索引
const int i_gpu_start = n_layer - n_gpu_layers;

# 调整模型的层数
model.layers.resize(n_layer);
# 循环遍历每个层
for (uint32_t i = 0; i < n_layer; ++i) {
    # 根据索引判断当前层的计算后端类型，如果索引小于 GPU 开始索引，则使用 CPU 计算后端，否则使用 llama_backend_offload
    const ggml_backend_type backend = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload; // NOLINT
    # 根据索引判断当前层的计算后端类型，如果索引小于 GPU 开始索引，则使用 CPU 计算后端，否则使用 llama_backend_offload_split
    const ggml_backend_type backend_split = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload_split; // NOLINT

    # 获取当前层对象的引用
    auto & layer = model.layers[i];

    # 创建注意力归一化张量
    layer.attn_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM, "weight", i), {n_embd}, backend);

    # 创建注意力查询张量
    layer.wq = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_Q,   "weight", i), {n_embd, n_embd},     backend_split);
    # 创建注意力键张量
    layer.wk = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_K,   "weight", i), {n_embd, n_embd_gqa}, backend_split);
    # 创建注意力值张量
    layer.wv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_V,   "weight", i), {n_embd, n_embd_gqa}, backend_split);
    # 创建注意力输出张量
    layer.wo = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd},     backend_split);

    # 创建前馈网络归一化张量
    layer.ffn_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "weight", i), {n_embd}, backend);

    # 创建前馈网络门控张量
    layer.ffn_gate = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_GATE, "weight", i), {n_embd,   n_ff}, backend_split);
    # 创建前馈网络下采样张量
    layer.ffn_down = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "weight", i), {  n_ff, n_embd}, backend_split);
    # 创建前馈网络上采样张量
    layer.ffn_up   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,   "weight", i), {n_embd,   n_ff}, backend_split);

    # 如果当前层的计算后端类型为 GPU，则执行以下操作
    if (backend == GGML_BACKEND_GPU) {
# 计算各层权重在 VRAM 中所占用的字节数并累加
vram_weights +=
    ggml_nbytes(layer.attn_norm) + ggml_nbytes(layer.wq)       + ggml_nbytes(layer.wk)       +
    ggml_nbytes(layer.wv)        + ggml_nbytes(layer.wo)       + ggml_nbytes(layer.ffn_norm) +
    ggml_nbytes(layer.ffn_gate)  + ggml_nbytes(layer.ffn_down) + ggml_nbytes(layer.ffn_up);
# 结束当前循环
}
# 结束当前循环
}
# 结束当前循环
} break;
# 根据不同的架构类型进行不同的操作
case LLM_ARCH_BAICHUAN:
    # 创建 token embedding 张量
    model.tok_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab}, GGML_BACKEND_CPU);
    {
        # 定义变量
        ggml_backend_type backend_norm;
        ggml_backend_type backend_output;
        # 如果 GPU 层数大于层数
        if (n_gpu_layers > int(n_layer)) {
            # 在不同平台下选择不同的后端类型
            # 在 Windows 平台下，norm 不是性能相关的，但在 VRAM 中保留它可以减少数据复制
            # 但在 Windows 平台下，除非所有内容都在 GPU 上，否则这样做会有害
#ifndef _WIN32
            backend_norm = llama_backend_offload;
#else
// 如果 GPU 层数小于等于层数加2，则使用 CPU 后端，否则使用 offload 后端
backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;

// 设置输出后端为 offload 分割
backend_output = llama_backend_offload_split;
} else {
    // 如果不满足条件，使用 CPU 后端
    backend_norm   = GGML_BACKEND_CPU;
    // 输出也使用 CPU 后端
    backend_output = GGML_BACKEND_CPU;
}

// 创建输出归一化张量
model.output_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd},          backend_norm);
// 创建输出张量
model.output      = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT,      "weight"), {n_embd, n_vocab}, backend_output);

// 如果归一化后端为 GPU，则计算 VRAM 占用
if (backend_norm == GGML_BACKEND_GPU) {
    vram_weights += ggml_nbytes(model.output_norm);
}
// 如果输出后端为 GPU 分割，则计算 VRAM 占用
if (backend_output == GGML_BACKEND_GPU_SPLIT) {
    vram_weights += ggml_nbytes(model.output);
}
// 定义并初始化变量 n_ff，其值为 hparams.n_ff
const uint32_t n_ff = hparams.n_ff;

// 计算 GPU 起始层的索引
const int i_gpu_start = n_layer - n_gpu_layers;

// 调整模型的层数
model.layers.resize(n_layer);

// 遍历每一层
for (uint32_t i = 0; i < n_layer; ++i) {
    // 根据当前层的索引确定使用的后端类型
    const ggml_backend_type backend = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload; // NOLINT
    const ggml_backend_type backend_split = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload_split; // NOLINT

    // 获取当前层的引用
    auto & layer = model.layers[i];

    // 创建注意力层的归一化张量
    layer.attn_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM, "weight", i), {n_embd}, backend);

    // 创建注意力层的查询张量
    layer.wq = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_Q,   "weight", i), {n_embd, n_embd},     backend_split);
    // 创建注意力层的键张量
    layer.wk = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_K,   "weight", i), {n_embd, n_embd_gqa}, backend_split);
    // 创建注意力层的值张量
    layer.wv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_V,   "weight", i), {n_embd, n_embd_gqa}, backend_split);
    // 创建注意力层的输出张量
    layer.wo = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd},     backend_split);

    // 创建前馈神经网络层的归一化张量
    layer.ffn_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "weight", i), {n_embd}, backend);
}
# 创建一个名为ffn_gate的张量，用于存储前馈神经网络的门控权重
layer.ffn_gate = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_GATE, "weight", i), {n_embd, n_ff}, backend_split);
# 创建一个名为ffn_down的张量，用于存储前馈神经网络的下层权重
layer.ffn_down = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "weight", i), {n_ff, n_embd}, backend_split);
# 创建一个名为ffn_up的张量，用于存储前馈神经网络的上层权重
layer.ffn_up = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP, "weight", i), {n_embd, n_ff}, backend_split);

# 如果使用GPU后端，则计算各个张量占用的显存空间并累加
if (backend == GGML_BACKEND_GPU) {
    vram_weights +=
        ggml_nbytes(layer.attn_norm) + ggml_nbytes(layer.wq) + ggml_nbytes(layer.wk) +
        ggml_nbytes(layer.wv) + ggml_nbytes(layer.wo) + ggml_nbytes(layer.ffn_norm) +
        ggml_nbytes(layer.ffn_gate) + ggml_nbytes(layer.ffn_down) + ggml_nbytes(layer.ffn_up);
}
# 结束当前的switch语句块
} break;
# 如果架构为LLM_ARCH_FALCON，则执行以下代码
case LLM_ARCH_FALCON:
{
    # 目前仅支持CPU，待后续实现GPU支持
    // TODO: CPU-only for now

    # 创建一个名为tok_embd的张量，用于存储token的嵌入权重
    model.tok_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab}, GGML_BACKEND_CPU);

    # 输出
// 定义两个变量，用于存储后端类型
ggml_backend_type backend_norm;
ggml_backend_type backend_output;

// 如果 GPU 层数大于层数
if (n_gpu_layers > int(n_layer)) {
    // norm 对性能影响不大，但在 VRAM 中保留它可以减少数据复制
    // 但在 Windows 上，除非所有内容都在 GPU 上，否则这对性能有害
    #ifndef _WIN32
    backend_norm = llama_backend_offload;
    #else
    backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
    #endif // _WIN32

    // 输出后端为 offload_split
    backend_output = llama_backend_offload_split;
} else {
    // norm 和 output 的后端都为 CPU
    backend_norm   = GGML_BACKEND_CPU;
    backend_output = GGML_BACKEND_CPU;
}

// 创建一个张量，用于存储输出的 norm，指定后端类型
model.output_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd}, backend_norm);
# 创建模型的输出层偏置张量，指定张量名称、形状和后端类型
model.output_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "bias"), {n_embd}, backend_norm);
# 创建模型的输出层权重张量，指定张量名称、形状和后端类型
model.output = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT, "weight"), {n_embd, n_vocab}, backend_output);

# 如果使用 GPU 后端，计算模型输出层权重和偏置张量所占用的显存
if (backend_norm == GGML_BACKEND_GPU):
    vram_weights += ggml_nbytes(model.output_norm);
    vram_weights += ggml_nbytes(model.output_norm_b);
# 如果使用 GPU 分割后端，计算模型输出层权重张量所占用的显存
if (backend_output == GGML_BACKEND_GPU_SPLIT):
    vram_weights += ggml_nbytes(model.output);

# 设置前馈神经网络层数
const uint32_t n_ff = hparams.n_ff;

# 计算 GPU 起始索引
const int i_gpu_start = n_layer - n_gpu_layers;

# 调整模型层列表大小
model.layers.resize(n_layer);

# 遍历每一层，根据索引确定使用的后端类型
for (uint32_t i = 0; i < n_layer; ++i):
    const ggml_backend_type backend = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload; // NOLINT
// 根据索引 i 判断使用的后端类型，小于 i_gpu_start 的情况下使用 CPU 后端，否则使用 llama_backend_offload_split 后端
const ggml_backend_type backend_split = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload_split; // NOLINT

// 获取模型的第 i 层
auto & layer = model.layers[i];

// 创建注意力归一化的权重张量和偏置张量
layer.attn_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM,   "weight", i), {n_embd}, backend);
layer.attn_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM,   "bias", i),   {n_embd}, backend);

// 如果存在第二个注意力归一化的权重张量，创建它和对应的偏置张量
if (gguf_find_tensor(ml.ctx_gguf, tn(LLM_TENSOR_ATTN_NORM_2, "weight", i).first.c_str()) >= 0) {
    layer.attn_norm_2   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM_2, "weight", i), {n_embd}, backend);
    layer.attn_norm_2_b = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM_2, "bias", i),   {n_embd}, backend);

    // 如果使用 GPU 后端，计算第二个注意力归一化的权重张量和偏置张量所占用的显存
    if (backend == GGML_BACKEND_GPU) {
        vram_weights += ggml_nbytes(layer.attn_norm_2);
        vram_weights += ggml_nbytes(layer.attn_norm_2_b);
    }
}

// 创建查询、键、值张量和输出张量
layer.wqkv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_QKV, "weight", i), {n_embd, n_embd + 2*n_embd_gqa}, backend_split);
layer.wo   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd},                backend_split);
# 创建下层前馈神经网络的权重张量
layer.ffn_down = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "weight", i), {  n_ff, n_embd}, backend_split);
# 创建上层前馈神经网络的权重张量
layer.ffn_up   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,   "weight", i), {n_embd,   n_ff}, backend_split);

# 如果使用 GPU 后端，计算权重张量占用的显存空间
if (backend == GGML_BACKEND_GPU) {
    vram_weights +=
        ggml_nbytes(layer.attn_norm) + ggml_nbytes(layer.attn_norm_b) +
        ggml_nbytes(layer.wqkv)      + ggml_nbytes(layer.wo)          +
        ggml_nbytes(layer.ffn_down)  + ggml_nbytes(layer.ffn_up);
}

# 根据不同的模型架构创建不同的张量
switch (arch) {
    case LLM_ARCH_TRANSFORMER:
        # 创建 token embedding 张量
        model.tok_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab}, GGML_BACKEND_CPU);
        # 创建 position embedding 张量
        model.pos_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_POS_EMBD, "weight"),   {n_embd, hparams.n_ctx_train}, GGML_BACKEND_CPU);

        # 输出
        {
            # 定义用于归一化和输出的后端类型
            ggml_backend_type backend_norm;
            ggml_backend_type backend_output;
// 如果 GPU 层数大于层数，则进行以下操作
if (n_gpu_layers > int(n_layer)) {
    // norm 对性能影响不大，但在 VRAM 中保留它可以减少数据复制
    // 在 Windows 上，除非所有内容都在 GPU 上，否则这样做会有害
    #ifndef _WIN32
        backend_norm = llama_backend_offload;
    #else
        backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
    #endif // _WIN32

    // 设置输出的后端为 offload_split
    backend_output = llama_backend_offload_split;
} else {
    // 如果 GPU 层数小于等于层数，则将后端设置为 CPU
    backend_norm   = GGML_BACKEND_CPU;
    backend_output = GGML_BACKEND_CPU;
}

// 创建输出层的权重张量，使用指定的后端
model.output_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd},          backend_norm);
// 创建输出层的偏置张量，使用指定的后端
model.output_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "bias"),   {n_embd},          backend_norm);
// 创建输出层的权重张量，使用指定的后端
model.output        = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT,      "weight"), {n_embd, n_vocab}, backend_output);
// 如果后端是 GPU，则计算模型输出的规范化和偏置的显存占用
if (backend_norm == GGML_BACKEND_GPU) {
    vram_weights += ggml_nbytes(model.output_norm);
    vram_weights += ggml_nbytes(model.output_norm_b);
}
// 如果后端是 GPU 分离，则计算模型输出的显存占用
if (backend_output == GGML_BACKEND_GPU_SPLIT) {
    vram_weights += ggml_nbytes(model.output);
}

// 获取前馈神经网络的层数
const uint32_t n_ff = hparams.n_ff;

// 计算 GPU 开始的索引
const int i_gpu_start = n_layer - n_gpu_layers;

// 调整模型的层数
model.layers.resize(n_layer);

// 遍历模型的每一层
for (uint32_t i = 0; i < n_layer; ++i) {
    // 根据层的索引确定后端类型，如果索引小于 GPU 开始的索引，则使用 CPU 后端，否则使用 llama_backend_offload
    const ggml_backend_type backend       = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload; // NOLINT
    // 根据层的索引确定后端类型，如果索引小于 GPU 开始的索引，则使用 CPU 后端，否则使用 llama_backend_offload_split
    const ggml_backend_type backend_split = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload_split; // NOLINT

    // 获取当前层的引用
    auto & layer = model.layers[i];
}
# 创建注意力层的权重和偏置张量
layer.attn_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM,   "weight", i), {n_embd}, backend);
layer.attn_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM,   "bias", i),   {n_embd}, backend);

# 创建注意力层的查询、键、值权重和偏置张量
layer.wqkv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_QKV, "weight", i), {n_embd, n_embd + 2*n_embd_gqa}, backend_split);
layer.bqkv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_QKV, "bias", i),   {n_embd + 2*n_embd_gqa},         backend);

# 创建注意力层的输出权重和偏置张量
layer.wo   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd},   backend_split);
layer.bo   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "bias", i),   {n_embd},           backend);

# 创建前馈神经网络层的归一化权重和偏置张量
layer.ffn_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "weight", i), {n_embd}, backend);
layer.ffn_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "bias", i),   {n_embd}, backend);

# 创建前馈神经网络层的下降权重和偏置张量
layer.ffn_down   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "weight", i), {n_ff, n_embd}, backend_split);
layer.ffn_down_b = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "bias", i),   {n_embd},       backend);

# 创建前馈神经网络层的上升权重和偏置张量
layer.ffn_up   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,   "weight", i), {n_embd, n_ff}, backend_split);
layer.ffn_up_b = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,   "bias", i),           {n_ff}, backend);

# 如果是 GPU 后端，则执行以下操作
if (backend == GGML_BACKEND_GPU) {
# 计算各个层的权重并累加到vram_weights中
vram_weights +=
    # 计算注意力归一化层的字节数和偏置的字节数
    ggml_nbytes(layer.attn_norm) + ggml_nbytes(layer.attn_norm_b) +
    # 计算wqkv层的字节数和偏置的字节数
    ggml_nbytes(layer.wqkv)      + ggml_nbytes(layer.bqkv)        +
    # 计算wo层的字节数和偏置的字节数
    ggml_nbytes(layer.wo)        + ggml_nbytes(layer.bo)          +
    # 计算ffn_norm层的字节数和偏置的字节数
    ggml_nbytes(layer.ffn_norm)  + ggml_nbytes(layer.ffn_norm_b)  +
    # 计算ffn_down层的字节数和偏置的字节数
    ggml_nbytes(layer.ffn_down)  + ggml_nbytes(layer.ffn_down_b)  +
    # 计算ffn_up层的字节数和偏置的字节数
    ggml_nbytes(layer.ffn_up)    + ggml_nbytes(layer.ffn_up_b);
# 结束当前的循环
}
# 结束当前的switch语句
} break;
# 当前架构为LLM_ARCH_PERSIMMON时的处理
case LLM_ARCH_PERSIMMON:
{
    # 创建名为"weight"的张量，形状为{n_embd, n_vocab}，在CPU上使用ml.create_tensor函数
    model.tok_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD, "weight"),  {n_embd, n_vocab}, GGML_BACKEND_CPU);

    {
        # 定义变量backend_norm和backend_output
        ggml_backend_type backend_norm;
        ggml_backend_type backend_output;

        # 如果n_gpu_layers大于n_layer，则执行以下代码块
        if (n_gpu_layers > int(n_layer)) {
            # 如果GGML_USE_CUBLAS宏被定义，则执行以下代码块
// 如果 GPU 层数大于层数加一，则抛出运行时错误，提示 CUDA 后端缺少 Persimmon CUDA 操作，最多可以卸载 n_layer + 1 层
if (n_gpu_layers > int(n_layer + 1)) {
    LLAMA_LOG_ERROR("%s: CUDA backend missing Persimmon CUDA ops, can offload at most %ld layers. See: https://github.com/ggerganov/llama.cpp/issues/4038\n",
        __func__, n_layer + 1);
    throw std::runtime_error("Persimmon CUDA offload failed");
}
#endif

// norm 对性能并不重要，但在 VRAM 中保留它可以减少数据复制
// 在 Windows 上，除非所有内容都在 GPU 上，否则这对性能有害
#ifndef _WIN32
backend_norm = llama_backend_offload;
#else
backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#endif // _WIN32

// 设置输出后端为 offload_split
backend_output = llama_backend_offload_split;
} else {
    // 如果不使用 GPU，则将 norm 和输出后端都设置为 CPU
    backend_norm   = GGML_BACKEND_CPU;
    backend_output = GGML_BACKEND_CPU;
}
# 创建模型的输出归一化权重张量
model.output_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd}, backend_norm)
# 创建模型的输出归一化偏置张量
model.output_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "bias"), {n_embd}, backend_norm)
# 创建模型的输出权重张量
model.output = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT, "weight"), {n_embd, n_vocab}, backend_output)

# 如果归一化操作在 GPU 上执行，则计算占用显存的权重大小
if (backend_norm == GGML_BACKEND_GPU):
    vram_weights += ggml_nbytes(model.output_norm)
    vram_weights += ggml_nbytes(model.output_norm_b)

# 如果输出操作在 GPU 上执行，则计算占用显存的权重大小
if (backend_output == GGML_BACKEND_GPU_SPLIT):
    vram_weights += ggml_nbytes(model.output)

# 设置前馈神经网络层数
const uint32_t n_ff = hparams.n_ff
# 计算 GPU 开始的索引
const int i_gpu_start = n_layer - n_gpu_layers
# 调整模型层数
model.layers.resize(n_layer)

# 遍历每一层神经网络
for (uint32_t i = 0; i < n_layer; ++i):
    # 根据索引确定当前层的计算后端类型
    const ggml_backend_type backend = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload
    const ggml_backend_type backend_split = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload_split
    # 获取当前层的引用
    auto & layer = model.layers[i]
# 为 layer 对象创建不同类型的张量，并初始化它们的名称、形状和后端类型
layer.attn_norm     = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM,   "weight", i), {n_embd}, backend);
layer.attn_norm_b   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM,   "bias",   i), {n_embd}, backend);
layer.wqkv          = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_QKV,    "weight", i), {n_embd, n_embd + 2*n_embd_gqa}, backend_split);
layer.bqkv          = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_QKV,    "bias",   i), {n_embd + 2*n_embd_gqa},         backend);
layer.wo            = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT,    "weight", i), {n_embd, n_embd},   backend_split);
layer.bo            = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT,    "bias",   i), {n_embd},           backend);
layer.ffn_down      = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN,    "weight", i), {n_ff, n_embd}, backend_split);
layer.ffn_down_b    = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN,    "bias",   i), {n_embd},       backend);
layer.ffn_up        = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,      "weight", i), {n_embd,   n_ff}, backend_split);
layer.ffn_up_b      = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,      "bias",   i), {n_ff},           backend);
layer.ffn_norm      = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM,    "weight", i), {n_embd}, backend);
layer.ffn_norm_b    = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM,    "bias",   i), {n_embd}, backend);
layer.attn_q_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_Q_NORM, "weight", i), {64}, backend);
layer.attn_q_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_Q_NORM, "bias",   i), {64}, backend);
layer.attn_k_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_K_NORM, "weight", i), {64}, backend);
layer.attn_k_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_K_NORM, "bias",   i), {64}, backend);
// TODO: CPU-only for now
// 目前仅支持 CPU

// 创建 token embedding 张量，指定大小和后端为 CPU
model.tok_embd   = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD,      "weight"), {n_embd, n_vocab}, GGML_BACKEND_CPU);
// 创建 token embedding 归一化张量，指定大小和后端为 CPU
model.tok_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD_NORM, "weight"), {n_embd},          GGML_BACKEND_CPU);
// 创建 token embedding 归一化偏置张量，指定大小和后端为 CPU
model.tok_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD_NORM, "bias"),   {n_embd},          GGML_BACKEND_CPU);

// 输出
{
    ggml_backend_type backend_norm;
    ggml_backend_type backend_output;

    if (n_gpu_layers > int(n_layer)) {
        // 在 GPU 层数大于层级时
        // 归一化在性能上并不重要，但在 VRAM 中保留它可以减少数据复制
        // 但在 Windows 上，除非所有内容都在 GPU 上，否则这对性能有害
#ifndef _WIN32
        backend_norm = llama_backend_offload;
#else
        backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#endif // _WIN32
# 如果使用 llama_backend_offload_split，则将 backend_output 设置为 llama_backend_offload_split
# 否则将 backend_norm 和 backend_output 都设置为 GGML_BACKEND_CPU
if (use_offload) {
    backend_output = llama_backend_offload_split;
} else {
    backend_norm   = GGML_BACKEND_CPU;
    backend_output = GGML_BACKEND_CPU;
}

# 创建模型的输出归一化权重张量，指定张量的名称、形状和计算后端
model.output_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd},          backend_norm);
# 创建模型的输出归一化偏置张量，指定张量的名称、形状和计算后端
model.output_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "bias"),   {n_embd},          backend_norm);
# 创建模型的输出权重张量，指定张量的名称、形状和计算后端
model.output        = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT,      "weight"), {n_embd, n_vocab}, backend_output);

# 如果使用 GPU 计算后端，则计算模型输出归一化权重和偏置张量所占用的显存
if (backend_norm == GGML_BACKEND_GPU) {
    vram_weights += ggml_nbytes(model.output_norm);
    vram_weights += ggml_nbytes(model.output_norm_b);
}
# 如果使用 GPU_SPLIT 计算后端，则计算模型输出权重张量所占用的显存
if (backend_output == GGML_BACKEND_GPU_SPLIT) {
    vram_weights += ggml_nbytes(model.output);
}

# 获取前馈神经网络的神经元数量
const uint32_t n_ff = hparams.n_ff;
// 定义 GPU 开始的层索引
const int i_gpu_start = n_layer - n_gpu_layers;

// 调整模型的层数
model.layers.resize(n_layer);

// 遍历每一层
for (uint32_t i = 0; i < n_layer; ++i) {
    // 根据层索引确定使用的后端类型
    const ggml_backend_type backend       = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload; // NOLINT
    const ggml_backend_type backend_split = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload_split; // NOLINT

    // 获取当前层
    auto & layer = model.layers[i];

    // 创建注意力规范化的权重和偏置张量
    layer.attn_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM,   "weight", i), {n_embd}, backend);
    layer.attn_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM,   "bias", i),   {n_embd}, backend);

    // 创建注意力查询键值的权重和偏置张量
    layer.wqkv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_QKV, "weight", i), {n_embd, n_embd + 2*n_embd_gqa}, backend_split);
    layer.bqkv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_QKV, "bias", i),   {n_embd + 2*n_embd_gqa},         backend);

    // 创建注意力输出的权重和偏置张量
    layer.wo   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd},                backend_split);
    layer.bo   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "bias", i),   {n_embd},                        backend);
}
# 创建 layer.ffn_norm 张量，用于存储神经网络层的权重，大小为{n_embd}，使用指定的后端
layer.ffn_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "weight", i), {n_embd}, backend);
# 创建 layer.ffn_norm_b 张量，用于存储神经网络层的偏置，大小为{n_embd}，使用指定的后端
layer.ffn_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "bias", i),   {n_embd}, backend);

# 创建 layer.ffn_down 张量，用于存储神经网络层的权重，大小为{n_ff, n_embd}，使用指定的后端
layer.ffn_down   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "weight", i), {n_ff, n_embd}, backend_split);
# 创建 layer.ffn_down_b 张量，用于存储神经网络层的偏置，大小为{n_embd}，使用指定的后端
layer.ffn_down_b = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "bias", i),   {n_embd},       backend);

# 创建 layer.ffn_up 张量，用于存储神经网络层的权重，大小为{n_embd, n_ff}，使用指定的后端
layer.ffn_up   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,   "weight", i), {n_embd,   n_ff}, backend_split);
# 创建 layer.ffn_up_b 张量，用于存储神经网络层的偏置，大小为{n_ff}，使用指定的后端
layer.ffn_up_b = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,   "bias", i),   {n_ff},           backend);

# 如果使用的后端是 GPU
if (backend == GGML_BACKEND_GPU) {
    # 计算占用显存的权重和偏置的总字节数
    vram_weights +=
        ggml_nbytes(layer.attn_norm) + ggml_nbytes(layer.attn_norm_b) +
        ggml_nbytes(layer.wqkv)      + ggml_nbytes(layer.bqkv)        +
        ggml_nbytes(layer.wo)        + ggml_nbytes(layer.bo)          +
        ggml_nbytes(layer.ffn_norm)  + ggml_nbytes(layer.ffn_norm_b)  +
        ggml_nbytes(layer.ffn_up)    + ggml_nbytes(layer.ffn_up_b)    +
        ggml_nbytes(layer.ffn_down)  + ggml_nbytes(layer.ffn_down_b);
}
// 根据架构类型选择不同的操作
case LLM_ARCH_MPT:
    {
        // 创建一个名为 "tok_embd" 的张量，表示 token embedding，大小为 (n_embd, n_vocab)，使用 CPU 后端
        model.tok_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab}, GGML_BACKEND_CPU);

        // 输出
        {
            ggml_backend_type backend_norm;  // 定义变量 backend_norm，表示规范化的后端类型
            ggml_backend_type backend_output;  // 定义变量 backend_output，表示输出的后端类型

            if (n_gpu_layers > int(n_layer)) {
                // 如果 GPU 层数大于层数，则选择不同的后端类型
                // 规范化对性能不重要，但在 VRAM 中保留它可以减少数据复制
                // 在 Windows 上，除非所有内容都在 GPU 上，否则这对性能有害
#ifndef _WIN32
                backend_norm = llama_backend_offload;  // 如果不是 Windows，使用 llama_backend_offload 后端
#else
                backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;  // 如果是 Windows，根据条件选择不同的后端类型
#endif // _WIN32

                backend_output = llama_backend_offload_split;  // 输出使用 llama_backend_offload_split 后端
            } else {
# 设置神经网络的输入和输出的计算后端为 CPU
backend_norm   = GGML_BACKEND_CPU;
backend_output = GGML_BACKEND_CPU;

# 如果计算后端为 GPU，则将 VRAM 中的权重数据大小加入到 vram_weights 中
if (backend_norm == GGML_BACKEND_GPU) {
    vram_weights += ggml_nbytes(model.output_norm);
}
# 如果输出计算后端为 GPU_SPLIT，则将 VRAM 中的权重数据大小加入到 vram_weights 中
if (backend_output == GGML_BACKEND_GPU_SPLIT) {
    vram_weights += ggml_nbytes(model.output);
}

# 设置模型的输出规范化张量和输出张量
model.output_norm   = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd},          backend_norm);
model.output        = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT,      "weight"), {n_embd, n_vocab}, backend_output);

# 调整神经网络的层数
model.layers.resize(n_layer);
// 遍历每一层神经网络
for (uint32_t i = 0; i < n_layer; ++i) {
    // 根据条件选择使用 CPU 还是 GPU 运行
    const ggml_backend_type backend = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload; // NOLINT
    const ggml_backend_type backend_split = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload_split; // NOLINT

    // 获取当前层的引用
    auto & layer = model.layers[i];

    // 创建注意力规范化张量
    layer.attn_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM, "weight", i), {n_embd}, backend);
    // 创建注意力权重张量
    layer.wqkv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_QKV, "weight", i), {n_embd, n_embd + 2*n_embd_gqa}, backend_split);
    // 创建注意力输出张量
    layer.wo   = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd}, backend_split);

    // 创建前馈网络规范化张量
    layer.ffn_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "weight", i), {n_embd}, backend);

    // 创建前馈网络下层权重张量
    layer.ffn_down = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "weight", i), {  n_ff, n_embd}, backend_split);
    // 创建前馈网络上层权重张量
    layer.ffn_up   = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,   "weight", i), {n_embd,   n_ff}, backend_split);

    // 如果使用 GPU 运行，计算 VRAM 占用
    if (backend == GGML_BACKEND_GPU) {
        vram_weights +=
            ggml_nbytes(layer.attn_norm) +
            ggml_nbytes(layer.wqkv)      +
// 计算各个层的字节大小并相加，用于确定模型的总字节大小
ggml_nbytes(layer.wo) + ggml_nbytes(layer.ffn_norm) + ggml_nbytes(layer.ffn_down) + ggml_nbytes(layer.ffn_up);

// 根据不同的模型架构进行处理
switch (arch) {
    case LLM_ARCH_GPTLM:
        // 对于 GPTLM 模型架构
        {
            // 创建 token embedding 张量
            model.tok_embd = ml.create_tensor(ctx, tn(LLM_TENSOR_TOKEN_EMBD, "weight"), {n_embd, n_vocab}, GGML_BACKEND_CPU);

            // 输出
            {
                // 定义 norm 和 output 的后端类型
                ggml_backend_type backend_norm;
                ggml_backend_type backend_output;

                // 如果 GPU 层数大于层数
                if (n_gpu_layers > int(n_layer)) {
                    // 在 VRAM 上保留 norm 可以减少数据复制
                    // 在 Windows 上，除非所有内容都在 GPU 上，否则这样做会有害
#ifndef _WIN32
// 如果定义了宏 _WIN32，则将 backend_norm 设置为 llama_backend_offload，否则根据条件判断设置为 GGML_BACKEND_CPU 或 llama_backend_offload
#ifdef _WIN32
    backend_norm = llama_backend_offload;
#else
    backend_norm = n_gpu_layers <= (int) n_layer + 2 ? GGML_BACKEND_CPU : llama_backend_offload;
#endif // _WIN32

// 设置 backend_output 为 llama_backend_offload_split
backend_output = llama_backend_offload_split;
} else {
    // 如果条件不满足，则将 backend_norm 和 backend_output 设置为 GGML_BACKEND_CPU
    backend_norm   = GGML_BACKEND_CPU;
    backend_output = GGML_BACKEND_CPU;
}

// 使用 ml.create_tensor 创建 model.output_norm_b、model.output_norm 和 model.output 张量，并指定相应的大小和后端类型
model.output_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "bias"), {n_embd},          backend_norm);
model.output_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT_NORM, "weight"), {n_embd},          backend_norm);
model.output      = ml.create_tensor(ctx, tn(LLM_TENSOR_OUTPUT,      "weight"), {n_embd, n_vocab}, backend_output);

// 如果 backend_norm 为 GGML_BACKEND_GPU，则将 vram_weights 增加 model.output_norm 占用的内存大小
if (backend_norm == GGML_BACKEND_GPU) {
    vram_weights += ggml_nbytes(model.output_norm);
}
// 如果 backend_output 为 GGML_BACKEND_GPU_SPLIT，则将 vram_weights 增加 model.output 占用的内存大小
if (backend_output == GGML_BACKEND_GPU_SPLIT) {
    vram_weights += ggml_nbytes(model.output);
}
                    }
                    }

                    // 定义神经网络的前馈层的神经元数量
                    const uint32_t n_ff = hparams.n_ff;

                    // 计算 GPU 起始层的索引
                    const int i_gpu_start = n_layer - n_gpu_layers;

                    // 调整神经网络层的数量
                    model.layers.resize(n_layer);

                    // 遍历每一层神经网络
                    for (uint32_t i = 0; i < n_layer; ++i) {
                        /*
                        llama_model_loader: - tensor    4:         blk.0.attn_output.weight f16      [  2560,  2560,     1,     1 ]
                        */
                        // 根据当前层的索引确定使用的计算后端类型
                        const ggml_backend_type backend = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload; // NOLINT
                        const ggml_backend_type backend_split = int(i) < i_gpu_start ? GGML_BACKEND_CPU : llama_backend_offload_split; // NOLINT

                        // 获取当前层的引用
                        auto & layer = model.layers[i];

                        // 创建当前层的注意力规范化权重张量
                        layer.attn_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM, "weight", i), {n_embd}, backend);
                        // 创建当前层的注意力规范化偏置张量
                        layer.attn_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_NORM, "bias", i), {n_embd}, backend);
# 为当前层创建注意力机制的查询权重张量
layer.wq = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_Q,   "weight", i), {n_embd, n_embd},     backend_split);
# 为当前层创建注意力机制的键权重张量
layer.wk = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_K,   "weight", i), {n_embd, n_embd_gqa}, backend_split);
# 为当前层创建注意力机制的值权重张量
layer.wv = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_V,   "weight", i), {n_embd, n_embd_gqa}, backend_split);
# 为当前层创建注意力机制的输出权重张量
layer.wo = ml.create_tensor(ctx, tn(LLM_TENSOR_ATTN_OUT, "weight", i), {n_embd, n_embd},     backend_split);

# 为当前层创建前馈神经网络的归一化张量
layer.ffn_norm = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "weight", i), {n_embd}, backend);
# 为当前层创建前馈神经网络的归一化偏置张量
layer.ffn_norm_b = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_NORM, "bias", i), {n_embd}, backend);

# 为当前层创建前馈神经网络的门控权重张量
layer.ffn_gate = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_GATE, "weight", i), {n_embd,   n_ff}, backend_split);
# 为当前层创建前馈神经网络的下采样权重张量
layer.ffn_down = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_DOWN, "weight", i), {  n_ff, n_embd}, backend_split);
# 为当前层创建前馈神经网络的上采样权重张量
layer.ffn_up = ml.create_tensor(ctx, tn(LLM_TENSOR_FFN_UP,   "weight", i), {n_embd,   n_ff}, backend_split);

# 如果使用 GPU 后端，则计算当前层所占用的显存权重大小并累加到 vram_weights 中
if (backend == GGML_BACKEND_GPU) {
    vram_weights +=
        ggml_nbytes(layer.attn_norm) + ggml_nbytes(layer.wq)       + ggml_nbytes(layer.wk)       +
        ggml_nbytes(layer.wv)        + ggml_nbytes(layer.wo)       + ggml_nbytes(layer.ffn_norm) +
        ggml_nbytes(layer.ffn_gate)  + ggml_nbytes(layer.ffn_down) + ggml_nbytes(layer.ffn_up);
}
    } break;
    // 结束 switch 语句块

    default:
        // 如果没有匹配的 case，抛出未知架构的异常
        throw std::runtime_error("unknown architecture");
    }
    // 结束 switch 语句块

ml.done_getting_tensors();
// 标记模型加载完成

// 打印内存需求
{
    // 运行推理所需的总内存
    size_t mem_required =
        ctx_size +
        mmapped_size - vram_weights; // VRAM 中的权重不在内存中

    LLAMA_LOG_INFO("%s: mem required  = %7.2f MB\n", __func__, mem_required / 1024.0 / 1024.0);

#if defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST)
    // GPU 数量为 GPU 层和模型层数的最小值
    const int n_gpu = std::min(n_gpu_layers, int(hparams.n_layer));
// 记录日志，输出正在将重复层次的数量 offloading 到 GPU
LLAMA_LOG_INFO("%s: offloading %d repeating layers to GPU\n", __func__, n_gpu);
// 如果重复层次的数量大于模型层次的数量，记录日志，输出正在将非重复层次 offloading 到 GPU
if (n_gpu_layers > (int) hparams.n_layer) {
    LLAMA_LOG_INFO("%s: offloading non-repeating layers to GPU\n", __func__);
}

#ifdef GGML_USE_CUBLAS
    // 如果使用 CUBLAS，设置最大支持的后端层次数量和最大可 offload 的层次数量
    const int max_backend_supported_layers = hparams.n_layer + 3;
    const int max_offloadable_layers       = hparams.n_layer + 3;
#elif GGML_USE_CLBLAST
    // 如果使用 CLBLAST，设置最大支持的后端层次数量和最大可 offload 的层次数量
    const int max_backend_supported_layers = hparams.n_layer + 1;
    const int max_offloadable_layers       = hparams.n_layer + 1;
#endif // GGML_USE_CUBLAS

// 记录日志，输出已经 offloaded 到 GPU 的层次数量和最大支持的后端层次数量
LLAMA_LOG_INFO("%s: offloaded %d/%d layers to GPU\n", __func__, std::min(n_gpu_layers, max_offloadable_layers), max_backend_supported_layers);
// 记录日志，输出 VRAM 使用量
LLAMA_LOG_INFO("%s: VRAM used: %.2f MB\n", __func__, vram_weights / 1024.0 / 1024.0);
#else
    // 如果未定义 GGML_USE_CUBLAS 或 GGML_USE_CLBLAST，忽略 n_gpu_layers
    (void) n_gpu_layers;
#endif // defined(GGML_USE_CUBLAS) || defined(GGML_USE_CLBLAST)
}
// 遍历模型中的张量，获取张量的名称和指针，存储到model.tensors_by_name中
for (int i = 0; i < ml.n_tensors; ++i) {
    // 获取当前张量的指针
    struct ggml_tensor * cur = ggml_get_tensor(ctx, ml.get_tensor_name(i));
    // 将张量的名称和指针存储到model.tensors_by_name中
    model.tensors_by_name.emplace_back(ggml_get_name(cur), cur);
}

// 设置tensor_split
(void) tensor_split;
#ifdef GGML_USE_CUBLAS
{
    // 如果使用CUBLAS，则设置tensor_split
    ggml_cuda_set_tensor_split(tensor_split);
}
#endif

// 加载所有数据到模型中
ml.load_all_data(ctx, progress_callback, progress_callback_user_data, use_mlock ? &model.mlock_mmap : NULL);

// 如果有进度回调函数，则调用进度回调函数
if (progress_callback) {
    progress_callback(1.0f, progress_callback_user_data);
}
    // 将 ml.mapping 移动到 model.mapping，避免数据的复制
    model.mapping = std::move(ml.mapping);

    // 在第一次评估后重新计算加载时间，考虑由 mmap() 延迟引起的页面错误
    model.t_load_us = ggml_time_us() - model.t_start_us;
}

static bool llama_model_load(const std::string & fname, llama_model & model, const llama_model_params & params) {
    try {
        // 使用文件名和参数创建 llama_model_loader 对象 ml
        llama_model_loader ml(fname, params.use_mmap);

        // 如果 ml.sparse_deriv 为 GGML_SPARSE_INFERENCE，则打印相应信息
        if (ml.sparse_deriv == GGML_SPARSE_INFERENCE) {
            LLAMA_LOG_INFO("%s: PowerInfer model loaded. Sparse inference will be used.\n", __func__);
        }

        // 设置 model 的一些参数
        model.hparams.vocab_only = params.vocab_only;
        model.sparse_deriv = ml.sparse_deriv;

        // 调用 llm_load_arch 函数加载模型的架构信息
        llm_load_arch   (ml, model);
        // 调用 llm_load_hparams 函数加载模型的超参数信息
        llm_load_hparams(ml, model);
// 加载词汇表到模型中
llm_load_vocab(ml, model);

// 加载打印元数据到模型中
llm_load_print_meta(ml, model);

// 如果模型的词汇表大小与词汇表中的标记数量不匹配，则抛出运行时错误
if (model.hparams.n_vocab != model.vocab.id_to_token.size()) {
    throw std::runtime_error("vocab size mismatch");
}

// 如果参数中只包含词汇表，则跳过张量处理并返回 true
if (params.vocab_only) {
    LLAMA_LOG_INFO("%s: vocab only - skipping tensors\n", __func__);
    return true;
}

// 如果模型使用稀疏推断
if (llama_use_sparse_inference(&model)) {
    // 如果参数中指定的 GPU 层数大于 0，则警告稀疏推断会忽略 n_gpu_layers，建议使用 --vram-budget 选项
    if (params.n_gpu_layers > 0) {
        LLAMA_LOG_WARN("%s: sparse inference ignores n_gpu_layers, you can use --vram-budget option instead\n", __func__);
        return false;
    }
    // 计算 VRAM 预算的字节数，并打印稀疏推断的 VRAM 预算
    double vram_budget_bytes = params.vram_budget_gb * 1024.0 * 1024.0 * 1024.0;
    LLAMA_LOG_INFO("%s: sparse inference - vram budget = %.2f GB\n", __func__, vram_budget_bytes / 1024.0 / 1024.0 / 1024.0);
// 如果模型是稀疏的，则加载稀疏模型张量
llm_load_sparse_model_tensors(
    ml, model, params.main_gpu, vram_budget_bytes, params.reset_gpu_index, params.disable_gpu_index,
    params.use_mlock, params.progress_callback, params.progress_callback_user_data
);
// 如果模型不是稀疏的，则加载普通模型张量
} else {
    llm_load_tensors(
        ml, model, params.n_gpu_layers, params.main_gpu, params.tensor_split, params.use_mlock,
        params.progress_callback, params.progress_callback_user_data
    );
}

// 捕获异常并记录错误信息
} catch (const std::exception & err) {
    LLAMA_LOG_ERROR("error loading model: %s\n", err.what());
    return false;
}

// 加载模型成功，返回 true
return true;
}
// 定义一个名为 llm_build_cb 的函数类型，用于接受一个 ggml_tensor 结构体指针、一个字符串和一个整数作为参数并返回 void
using llm_build_cb = std::function<void(struct ggml_tensor * cur, const char * name, int nl)>;

// 定义一个枚举类型 llm_rope_type，包括 LLM_ROPE、LLM_ROPE_NEOX 和 LLM_ROPE_GLM 三个枚举值
enum llm_rope_type {
    LLM_ROPE,
    LLM_ROPE_NEOX,
    LLM_ROPE_GLM,
};

// 定义一个枚举类型 llm_ffn_op_type，包括 LLM_FFN_SILU、LLM_FFN_GELU、LLM_FFN_RELU 和 LLM_FFN_RELU_SQR 四个枚举值
enum llm_ffn_op_type {
    LLM_FFN_SILU,
    LLM_FFN_GELU,
    LLM_FFN_RELU,
    LLM_FFN_RELU_SQR,
};

// 定义一个枚举类型 llm_ffn_gate_type，包括 LLM_FFN_SEQ 枚举值
enum llm_ffn_gate_type {
    LLM_FFN_SEQ,
// 定义枚举类型，表示LLM的FFN门是与FFN上门并行的
LLM_FFN_PAR,

// 定义枚举类型，表示LLM的规范化类型
enum llm_norm_type {
    LLM_NORM, // 普通规范化
    LLM_NORM_RMS, // 均方根规范化
};

// 创建LLM输入嵌入的函数
static struct ggml_tensor * llm_build_inp_embd(
        struct ggml_context * ctx, // 上下文对象
        const llama_hparams & hparams, // LLM的超参数
        const llama_batch & batch, // LLM的批处理数据
        struct ggml_tensor * tok_embd, // token嵌入的张量
        const llm_build_cb & cb) { // 构建回调函数

    const int64_t n_embd = hparams.n_embd; // 获取嵌入维度

    struct ggml_tensor * inpL; // 输入张量

    // 如果批处理数据中有token
    if (batch.token) {
        // 创建输入token的张量
        struct ggml_tensor * inp_tokens = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, batch.n_tokens);
// 调用函数 cb，传入参数 inp_tokens 和字符串 "inp_tokens"，-1 作为标志
cb(inp_tokens, "inp_tokens", -1);

// 如果条件成立，执行以下代码块
inpL = ggml_get_rows(ctx, tok_embd, inp_tokens);
// 否则，执行以下代码块
#ifdef GGML_USE_MPI
    GGML_ASSERT(false && "not implemented");
#endif
// 根据条件选择不同的操作，创建 2 维张量 inpL
inpL = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, batch.n_tokens);

// 返回 inpL
return inpL;
}

// Persimmon: n_rot = n_embd_head/2
// Other:     n_rot = n_embd_head
// 根据不同的情况，设置 n_rot 的值
static void llm_build_k_shift(
      struct ggml_context * ctx,
      const llama_hparams & hparams,
      const llama_cparams & cparams,
// 定义一个函数，接受一系列参数，包括键值缓存、图形结构、绳索类型、上下文数量、旋转数量、频率基数、频率比例和构建回调函数
void build_model(const llama_kv_cache & kv,
                 struct ggml_cgraph * graph,
                 llm_rope_type   type,
                 int64_t   n_ctx,
                 int64_t   n_rot,
                 float     freq_base,
                 float     freq_scale,
                 const llm_build_cb & cb) {
    // 从超参数中获取层数、头部键值数量、GQA嵌入数量和头部嵌入数量
    const int64_t n_layer     = hparams.n_layer;
    const int64_t n_head_kv   = hparams.n_head_kv;
    const int64_t n_embd_gqa  = hparams.n_embd_gqa();
    const int64_t n_embd_head = hparams.n_embd_head();
    // 从上下文参数中获取原始上下文数量、扩展因子、注意力因子、快速学习率和慢速学习率
    const int32_t n_orig_ctx  = cparams.n_yarn_orig_ctx;
    const float   ext_factor  = cparams.yarn_ext_factor;
    const float   attn_factor = cparams.yarn_attn_factor;
    const float   beta_fast   = cparams.yarn_beta_fast;
    const float   beta_slow   = cparams.yarn_beta_slow;

    // 断言头部嵌入数量能够被旋转数量整除
    GGML_ASSERT(n_embd_head % n_rot == 0);
}
    // 创建一个指向 ggml_tensor 结构体的指针 K_shift，该结构体表示一个一维张量，数据类型为 GGML_TYPE_I32，长度为 n_ctx
    struct ggml_tensor * K_shift = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, n_ctx);
    // 调用回调函数 cb，传入 K_shift 指针和字符串 "K_shift"，以及参数 -1
    cb(K_shift, "K_shift", -1);

    // 初始化变量 rope_type 为 0
    int rope_type = 0;

    // 根据 type 的值进行不同的赋值操作
    switch (type) {
        case LLM_ROPE:      rope_type = 0; break;
        case LLM_ROPE_NEOX: rope_type = 2; break;
        case LLM_ROPE_GLM:  rope_type = 4; break;
    }

    // 循环遍历 n_layer 次
    for (int il = 0; il < n_layer; ++il) {
        // 创建一个指向 ggml_tensor 结构体的指针 tmp，表示一个张量
        // 调用 ggml_rope_custom_inplace 函数进行一些操作，返回结果赋值给 tmp
        // 该函数的具体操作需要根据函数定义和上下文来解释
        struct ggml_tensor * tmp =
            ggml_rope_custom_inplace(ctx,
                    ggml_view_3d(ctx, kv.k,
                        n_rot, n_head_kv, n_ctx,
                        ggml_element_size(kv.k)*n_embd_head,
                        ggml_element_size(kv.k)*n_embd_gqa,
                        ggml_element_size(kv.k)*n_embd_gqa*n_ctx*il),
// 定义变量 K_shift, n_rot, rope_type, 0, n_orig_ctx, freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow
// 并传入函数参数中
K_shift, n_rot, rope_type, 0, n_orig_ctx, freq_base, freq_scale,
ext_factor, attn_factor, beta_fast, beta_slow);
// 调用函数 cb，传入参数 "K_shifted", il
cb(tmp, "K_shifted", il);
// 调用函数 ggml_build_forward_expand，传入参数 graph, tmp
ggml_build_forward_expand(graph, tmp);
// 定义函数 llm_build_kv_store，传入参数 ctx, hparams, kv, graph, k_cur, v_cur, n_ctx, n_tokens, kv_head, cb, il
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
// 定义变量 n_embd_gqa，并赋值为 hparams.n_embd_gqa()
const int64_t n_embd_gqa = hparams.n_embd_gqa();
// 计算转置的 [n_tokens, n_embd] V 矩阵
struct ggml_tensor * v_cur_t = ggml_transpose(ctx, ggml_reshape_2d(ctx, v_cur, n_embd_gqa, n_tokens));
// 创建一个指向 v_cur_t 的视图
cb(v_cur_t, "v_cur_t", il);

// 创建一个指向 k 的视图，用于缓存
struct ggml_tensor * k_cache_view = ggml_view_1d(ctx, kv.k, n_tokens*n_embd_gqa,
        (ggml_element_size(kv.k)*n_embd_gqa)*(il*n_ctx + kv_head));
cb(k_cache_view, "k_cache_view", il);

// 创建一个指向 v 的视图，用于缓存
struct ggml_tensor * v_cache_view = ggml_view_2d(ctx, kv.v, n_tokens, n_embd_gqa,
        (   n_ctx)*ggml_element_size(kv.v),
        (il*n_ctx)*ggml_element_size(kv.v)*n_embd_gqa + kv_head*ggml_element_size(kv.v));
cb(v_cache_view, "v_cache_view", il);

// 重要：将 RoPE-ed 版本的 K 存储在 KV 缓存中
ggml_tensor * k_cpy = ggml_cpy(ctx, k_cur,   k_cache_view);
ggml_tensor * v_cpy = ggml_cpy(ctx, v_cur_t, v_cache_view);
// 创建并存储 K 的 RoPE-ed 版本
// 创建并存储 V 的 RoPE-ed 版本
// 返回一个包含 k_cpy 和 v_cpy 的字典
return {k_cpy, v_cpy};
}

// 构建归一化操作
static struct ggml_tensor * llm_build_norm(
        struct ggml_context * ctx, // 上下文
         struct ggml_tensor * cur, // 当前张量
        const llama_hparams & hparams, // 参数
         struct ggml_tensor * mw, // mw 张量
         struct ggml_tensor * mb, // mb 张量
              llm_norm_type   type, // 归一化类型
         const llm_build_cb & cb, // 构建回调函数
                        int   il) { // il 参数
    switch (type) {
        case LLM_NORM:     cur = ggml_norm    (ctx, cur, hparams.f_norm_eps);     break; // 根据类型选择不同的归一化操作
        case LLM_NORM_RMS: cur = ggml_rms_norm(ctx, cur, hparams.f_norm_rms_eps); break; // 根据类型选择不同的 RMS 归一化操作
    }

    if (mw || mb) {
        cb(cur, "norm", il); // 如果 mw 或 mb 存在，则调用回调函数
    }
    # 如果存在mw，则使用ggml_mul函数计算cur和mw的乘积，并将结果赋给cur
    if (mw) {
        cur = ggml_mul(ctx, cur, mw);
        # 如果存在mb，则调用cb函数，将cur、"norm_w"和il作为参数传入
        if (mb) {
            cb(cur, "norm_w", il);
        }
    }
    # 如果存在mb，则使用ggml_add函数计算cur和mb的和，并将结果赋给cur
    if (mb) {
        cur = ggml_add(ctx, cur, mb);
    }
    # 返回cur
    return cur;
}
# 定义一个静态的ggml_tensor结构体指针类型的函数llm_build_ffn，接受ctx、cur和up作为参数
static struct ggml_tensor * llm_build_ffn(
        struct ggml_context * ctx,
        struct ggml_tensor * cur,
        struct ggml_tensor * up,
// 定义一个函数，接受多个参数，包括指向 ggml_tensor 结构体的指针和其他类型参数
void some_function(struct ggml_tensor * up,
         struct ggml_tensor * up_b,
         struct ggml_tensor * gate,
         struct ggml_tensor * gate_b,
         struct ggml_tensor * down,
         struct ggml_tensor * down_b,
         llm_ffn_op_type   type_op,
         llm_ffn_gate_type   type_gate,
         const llm_build_cb & cb,
         int   il) {
    // 调用 ggml_mul_mat 函数，传入上面两个指针参数，并将结果赋给 tmp
    struct ggml_tensor * tmp = ggml_mul_mat(ctx, up, cur);
    // 调用回调函数 cb，传入 tmp 指针和字符串 "ffn_up"，以及整数参数 il
    cb(tmp, "ffn_up", il);

    // 如果 up_b 存在
    if (up_b) {
        // 调用 ggml_add 函数，传入 ctx, tmp, up_b，并将结果赋给 tmp
        tmp = ggml_add(ctx, tmp, up_b);
        // 调用回调函数 cb，传入 tmp 指针和字符串 "ffn_up_b"，以及整数参数 il
        cb(tmp, "ffn_up_b", il);
    }

    // 如果 gate 存在
    if (gate) {
        // 根据 type_gate 的值进行不同的操作
        switch (type_gate) {
            // 如果 type_gate 的值为 LLM_FFN_SEQ
            case LLM_FFN_SEQ:
        {
            // 如果操作类型为 LLM_FFN_MUL
            cur = ggml_mul_mat(ctx, gate, tmp);
            // 调用回调函数，传入当前值、操作类型和索引
            cb(cur, "ffn_gate", il);
        } break;
    case LLM_FFN_PAR:
        {
            // 如果操作类型为 LLM_FFN_PAR
            cur = ggml_mul_mat(ctx, gate, cur);
            // 调用回调函数，传入当前值、操作类型和索引
            cb(cur, "ffn_gate", il);
        } break;
    }

    // 如果 gate_b 存在
    if (gate_b) {
        // 将当前值与 gate_b 相加
        cur = ggml_add(ctx, cur, gate_b);
        // 调用回调函数，传入当前值、操作类型和索引
        cb(cur, "ffn_gate_b", il);
    }
} else {
    // 如果 gate_b 不存在，则将当前值设为 tmp
    cur = tmp;
}

// 根据操作类型进行不同的处理
switch (type_op) {
# 根据不同的情况调用不同的函数，并将结果传递给回调函数
case LLM_FFN_SILU:
    # 调用 ggml_silu 函数处理当前情况
    cur = ggml_silu(ctx, cur);
    # 调用回调函数，传递当前结果、"ffn_silu" 和 il
    cb(cur, "ffn_silu", il);
    break;
case LLM_FFN_GELU:
    # 调用 ggml_gelu 函数处理当前情况
    cur = ggml_gelu(ctx, cur);
    # 调用回调函数，传递当前结果、"ffn_gelu" 和 il
    cb(cur, "ffn_gelu", il);
    break;
case LLM_FFN_RELU:
    # 调用 ggml_relu 函数处理当前情况
    cur = ggml_relu(ctx, cur);
    # 调用回调函数，传递当前结果、"ffn_relu" 和 il
    cb(cur, "ffn_relu", il);
    break;
case LLM_FFN_RELU_SQR:
    # 调用 ggml_relu 函数处理当前情况
    cur = ggml_relu(ctx, cur);
    # 调用回调函数，传递当前结果、"ffn_relu" 和 il
    cb(cur, "ffn_relu", il);
# 对当前值进行平方操作
cur = ggml_sqr(ctx, cur);
# 调用回调函数，传递当前值、操作名称和层级
cb(cur, "ffn_sqr(relu)", il);
# 结束当前的 switch 语句

# 如果门的类型是 LLM_FFN_PAR
if (type_gate == LLM_FFN_PAR) {
    # 对当前值和临时值进行乘法操作
    cur = ggml_mul(ctx, cur, tmp);
    # 调用回调函数，传递当前值、操作名称和层级
    cb(cur, "ffn_gate_par", il);
}

# 对当前值和下降值进行线性组合操作
cur = ggml_axpy(ctx, down, cur, NULL, NULL);
# 如果存在下降值的回调函数
if (down_b) {
    # 调用回调函数，传递当前值、操作名称和层级
    cb(cur, "ffn_down", il);
}

# 如果存在下降值
if (down_b) {
    # 对当前值和下降偏置进行加法操作
    cur = ggml_add(ctx, cur, down_b);
}
// 返回当前张量
return cur;
}

// 构建稀疏前馈神经网络
static struct ggml_tensor * llm_build_ffn_sparse(
        struct ggml_context * ctx,  // 上下文
         struct ggml_tensor * cur,  // 当前张量
         struct ggml_tensor * up,  // 上层张量
         struct ggml_tensor * up_b,  // 上层偏置张量
         struct ggml_tensor * gate,  // 门控张量
         struct ggml_tensor * gate_b,  // 门控偏置张量
         struct ggml_tensor * down,  // 下层张量
         struct ggml_tensor * down_b,  // 下层偏置张量
         struct ggml_tensor * down_t,  // 下层目标张量
         struct ggml_tensor * pre_w1,  // 预处理权重1张量
         struct ggml_tensor * pre_w2,  // 预处理权重2张量
         struct ggml_tensor * pred_inpl,  // 预测输入张量
         struct ggml_tensor * gpu_index,  // GPU索引张量
         struct ggml_tensor * gpu_bucket,  // GPU桶张量
         struct ggml_tensor * gate_gpu,  // 门控GPU张量
         struct ggml_tensor * down_gpu,  // 下层GPU张量
// 定义一个函数，接受多个参数，包括指向结构体的指针、枚举类型、回调函数、整数
void some_function(struct ggml_tensor * down_gpu,
                   struct ggml_tensor * gate_gpu,
                   struct ggml_tensor * up_gpu,
                   llm_ffn_op_type   type_op,
                   llm_ffn_gate_type   type_gate,
                   const llm_build_cb & cb,
                   int   il) {
    // TODO: no gpu slicing for now
    // 暂时不支持 GPU 切片
    // GGML_ASSERT(gate_gpu == nullptr && down_gpu == nullptr && up_gpu == nullptr && gpu_bucket == nullptr);

    // 定义并初始化指向 ggml_tensor 结构体的指针
    ggml_tensor *idx = nullptr;
    ggml_tensor *idx_g = nullptr;
    ggml_tensor *cur_c = nullptr;
    ggml_tensor *third = nullptr;

    // 如果输入数据的后端与 pre_w1 的后端不一致
    if (pred_inpl->backend != pre_w1->backend) {
        // 如果 pre_w1 的后端是 CPU
        if (pre_w1->backend == GGML_BACKEND_CPU) {
            // 复制输入数据
            pred_inpl = ggml_dup(ctx, pred_inpl);
        } else {
            // 否则，暂时不做任何操作
            // NOOP for now
        }
    }
}
    // 准备稀疏索引
    idx = ggml_mul_mat(ctx, pre_w1, pred_inpl);
    // 没有离线处理
    cb(idx, "mlp_pre_w1", il);
    // 对稀疏索引进行 ReLU 激活函数处理
    idx = ggml_relu(ctx, idx);
    cb(idx, "relu_pre", il);
    // 使用稀疏索引进行矩阵乘法运算
    idx = ggml_mul_mat(ctx, pre_w2, idx);
    cb(idx, "mlp_pre_w2", il);

    // FFN 上行
    third = cur;
    // 使用稀疏索引进行矩阵乘法运算
    struct ggml_tensor * tmp = ggml_mul_mat_idx(ctx, up, cur, idx, gpu_index);
    cb(tmp, "ffn_up_sparse", il);
#ifdef GGML_USE_CUBLAS
    // 使用特殊的矩阵乘法运算（针对 GPU）
    struct ggml_tensor * tmp2 = ggml_mul_mat_special(ctx, up_gpu, cur, idx, gpu_bucket, up);
    if (tmp2 != NULL) {
        ggml_cuda_assign_buffers_no_alloc(tmp2);
        cb(tmp2, "ffn_up_sparse_gpu", il);
    }
    // 调用 ggml_add 函数，将 tmp2 添加到 tmp 中
    tmp = ggml_add(ctx, tmp, tmp2);
#endif

    // 如果 up_b 存在
    if (up_b) {
        // 调用 ggml_add 函数，将 up_b 添加到 tmp 中
        tmp = ggml_add(ctx, tmp, up_b);
        // 调用 cb 函数，传递 tmp, "ffn_up_b", il 参数
        cb(tmp, "ffn_up_b", il);
    }

    // 如果 gate 存在
    if (gate) {
        // TODO: 目前只支持 par
        GGML_ASSERT(type_gate == LLM_FFN_PAR);
        // 将 cur 赋值给 third
        third = cur;
        // 调用 ggml_mul_mat_idx 函数，计算 gate 与 cur 之间的乘积
        cur = ggml_mul_mat_idx(ctx, gate, cur, idx, gpu_index);
        // 调用 cb 函数，传递 cur, "ffn_gate", il 参数
        cb(cur, "ffn_gate", il);
#ifdef GGML_USE_CUBLAS
        // 调用 ggml_mul_mat_special 函数，计算 gate_gpu 与 third 之间的特殊乘积
        tmp2 = ggml_mul_mat_special(ctx, gate_gpu, third, idx, gpu_bucket, gate);
        // 如果 tmp2 不为空
        if (tmp2 != NULL) {
            // 调用 ggml_cuda_assign_buffers_no_alloc 函数
            ggml_cuda_assign_buffers_no_alloc(tmp2);
        // 调用 cb 函数，传入 tmp2, "ffn_up_sparse_gpu", il 作为参数
        cb(tmp2, "ffn_up_sparse_gpu", il);
        // 如果 gate_b 存在，则将其加入到 cur 中，并调用 cb 函数，传入 cur, "ffn_gate_b", il 作为参数
        if (gate_b) {
            cur = ggml_add(ctx, cur, gate_b);
            cb(cur, "ffn_gate_b", il);
        }
        // 如果 type_op 为 LLM_FFN_RELU，则对 cur 进行 relu 操作，并调用 cb 函数，传入 cur, "ffn_relu", il 作为参数
        switch (type_op) {
            case LLM_FFN_RELU:
                cur = ggml_relu(ctx, cur);
                cb(cur, "ffn_relu", il);
                break;
            default:
                // 其他情况下不做任何操作
        }
    // 如果操作类型不是 LLM_FFN_RELU，则抛出断言错误
    GGML_ASSERT(type_op == LLM_FFN_RELU);

    // 如果门类型是 LLM_FFN_PAR，则进行一系列操作，并调用回调函数
    if (type_gate == LLM_FFN_PAR) {
        cur = ggml_mul(ctx, cur, tmp);
        cb(cur, "ffn_gate_par", il);
    }

    // 将当前值赋给第三个变量
    third = cur;

    // 如果使用了 CUBLAS，则进行一系列操作，并调用回调函数
#ifdef GGML_USE_CUBLAS
    cur = ggml_axpy(ctx, down_gpu, cur, idx, gpu_bucket);
    if (cur != NULL) {
        ggml_cuda_assign_buffers_no_alloc(cur);
        cb(cur, "ffn_down", il);
    }
#endif

    // 进行一系列操作，并调用回调函数
    tmp = ggml_axpy(ctx, down_t, third, idx, gpu_index);
    cb(tmp, "ffn_down_gpu", il);

    // 如果使用了 CUBLAS，则进行一系列操作
#ifdef GGML_USE_CUBLAS
    // 如果条件成立，将 cur 更新为 ggml_add 函数的返回值，否则更新为 tmp
    cur = ggml_add(ctx, cur, tmp);
#else
    // 否则，将 cur 更新为 tmp
    cur = tmp;
#endif

    // 如果 down_b 存在，则将 cur 更新为 ggml_add 函数的返回值，并调用 cb 函数
    if (down_b) {
        cur = ggml_add(ctx, cur, down_b);
        cb(cur, "ffn_down", il);
    }

    // 返回 cur
    return cur;
}

// 声明并初始化静态指针 k_cpy 和 v_cpy
static ggml_tensor * k_cpy = nullptr;
static ggml_tensor * v_cpy = nullptr;

// 如果 max_alibi_bias 大于 0，则应用 ALiBi
// 构建 k、q、v 张量
static struct ggml_tensor * llm_build_kqv(
        struct ggml_context * ctx,
// 定义函数，接受一系列参数
void some_function(
    // llama_hparams 类型的引用参数
    const llama_hparams & hparams,
    // llama_kv_cache 类型的引用参数
    const llama_kv_cache & kv,
    // ggml_tensor 类型指针参数
    struct ggml_tensor * wo,
    // ggml_tensor 类型指针参数
    struct ggml_tensor * wo_b,
    // ggml_tensor 类型指针参数
    struct ggml_tensor * q_cur,
    // ggml_tensor 类型指针参数
    struct ggml_tensor * kq_scale,
    // ggml_tensor 类型指针参数
    struct ggml_tensor * kq_mask,
    // int64_t 类型参数
    int64_t n_ctx,
    // int32_t 类型参数
    int32_t n_tokens,
    // int32_t 类型参数
    int32_t n_kv,
    // float 类型参数
    float max_alibi_bias,
    // llm_build_cb 类型的引用参数
    const llm_build_cb & cb,
    // int 类型参数
    int il) {
    // 定义并初始化一系列变量
    const int64_t n_embd      = hparams.n_embd;
    const int64_t n_head      = hparams.n_head;
    const int64_t n_head_kv   = hparams.n_head_kv;
    const int64_t n_embd_head = hparams.n_embd_head();
    const int64_t n_embd_gqa  = hparams.n_embd_gqa();

    // 调用 ggml_permute 函数，对 q_cur 进行排列操作，返回结果赋值给 q
    struct ggml_tensor * q = ggml_permute(ctx, q_cur, 0, 2, 1, 3);
}
    # 调用函数cb，传入参数q, "q", il
    cb(q, "q", il);

    # 创建一个3D视图的张量k，传入参数ctx, kv.k, n_embd_head, n_kv, n_head_kv, 以及三个不同的内存偏移量
    struct ggml_tensor * k =
        ggml_view_3d(ctx, kv.k,
                n_embd_head, n_kv, n_head_kv,
                ggml_element_size(kv.k)*n_embd_gqa,
                ggml_element_size(kv.k)*n_embd_head,
                ggml_element_size(kv.k)*n_embd_gqa*n_ctx*il);
    # 调用函数cb，传入参数k, "k", il
    cb(k, "k", il);
    
    # 如果k_cpy不为空指针，则将k的src[1]指向k_cpy
    if (k_cpy != nullptr) {
        k->src[1] = k_cpy;
    }

    # 创建一个新的张量kq，通过将k和q相乘得到
    struct ggml_tensor * kq = ggml_mul_mat(ctx, k, q);
    # 调用函数cb，传入参数kq, "kq", il
    cb(kq, "kq", il);

    # 对kq进行缩放，传入参数kq_scale
    kq = ggml_scale(ctx, kq, kq_scale);
    # 调用函数cb，传入参数kq, "kq_scaled", il
    cb(kq, "kq_scaled", il);

    # 如果max_alibi_bias大于0.0f，则执行以下代码
    if (max_alibi_bias > 0.0f) {
    // TODO: n_head or n_head_kv - 待办事项：确定是使用 n_head 还是 n_head_kv
    // TODO: K-shift is likely not working - 待办事项：K-shift 可能不起作用
    // TODO: change to ggml_add - 待办事项：改为使用 ggml_add
    kq = ggml_alibi(ctx, kq, /*n_past*/ 0, n_head, max_alibi_bias); // 使用 ggml_alibi 函数对 kq 进行处理
    cb(kq, "kq_scaled_alibi", il); // 回调函数，记录 kq_scaled_alibi 的值

    kq = ggml_add(ctx, kq, kq_mask); // 使用 ggml_add 函数对 kq 进行处理
    cb(kq, "kq_masked", il); // 回调函数，记录 kq_masked 的值

    kq = ggml_soft_max(ctx, kq); // 使用 ggml_soft_max 函数对 kq 进行处理
    cb(kq, "kq_soft_max", il); // 回调函数，记录 kq_soft_max 的值

    // split cached v into n_head heads
    // 将缓存的 v 分割成 n_head 个头部
    struct ggml_tensor * v =
        ggml_view_3d(ctx, kv.v,
                n_kv, n_embd_head, n_head_kv,
                ggml_element_size(kv.v)*n_ctx,
                ggml_element_size(kv.v)*n_ctx*n_embd_head,
                ggml_element_size(kv.v)*n_ctx*n_embd_gqa*il);
    # 调用回调函数cb，传入参数v, "v", il
    cb(v, "v", il);
    # 如果v_cpy不为空指针，则将v的src[1]指向v_cpy
    if (v_cpy != nullptr) {
        v->src[1] = v_cpy;
    }

    # 使用ggml_mul_mat函数计算v和kq的乘积，结果存储在kqv中
    struct ggml_tensor * kqv = ggml_mul_mat(ctx, v, kq);
    # 调用回调函数cb，传入参数kqv, "kqv", il
    cb(kqv, "kqv", il);

    # 对kqv进行维度置换，结果存储在kqv_merged中
    struct ggml_tensor * kqv_merged = ggml_permute(ctx, kqv, 0, 2, 1, 3);
    # 调用回调函数cb，传入参数kqv_merged, "kqv_merged", il
    cb(kqv_merged, "kqv_merged", il);

    # 对kqv_merged进行2D连续化，结果存储在cur中
    struct ggml_tensor * cur = ggml_cont_2d(ctx, kqv_merged, n_embd, n_tokens);
    # 调用回调函数cb，传入参数cur, "kqv_merged_cont", il
    cb(cur, "kqv_merged_cont", il);

    # 使用ggml_mul_mat函数计算wo和cur的乘积，结果存储在cur中
    cur = ggml_mul_mat(ctx, wo, cur);
    # 如果wo_b为真，则调用回调函数cb，传入参数cur, "kqv_wo", il
    if (wo_b) {
        cb(cur, "kqv_wo", il);
    }

    # 如果wo_b为真，则执行以下代码
    if (wo_b) {
// 调用 ggml_add 函数，将 wo_b 添加到 ctx 中的 cur 中
cur = ggml_add(ctx, cur, wo_b);

// 返回 cur
return cur;
}

// 定义一个名为 no_offload_cb 的常量，其类型为 llm_build_cb，用于构建 ggml_tensor
const llm_build_cb no_offload_cb = [](struct ggml_tensor * cur, const char * name, int nl) {
    // 为 cur 设置名称
    ggml_set_name(cur, name);
};

// 定义一个名为 llm_build_context 的结构体，包含 model、hparams、cparams、batch、kv_self、n_embd、n_layer、n_ctx 等成员变量
struct llm_build_context {
    const llama_model    & model;
    const llama_hparams  & hparams;
    const llama_cparams  & cparams;
    const llama_batch    & batch;
    const llama_kv_cache & kv_self;

    const int64_t n_embd;
    const int64_t n_layer;
    const int64_t n_ctx;       // 用户指定的上下文大小（可以与 n_ctx_train 不同）
    // 定义头部数量
    const int64_t n_head;
    // 定义头部键值对数量
    const int64_t n_head_kv;
    // 定义头部嵌入数量
    const int64_t n_embd_head;
    // 定义全局查询嵌入数量
    const int64_t n_embd_gqa;

    // 定义频率基数
    const float freq_base;
    // 定义频率缩放
    const float freq_scale;
    // 定义扩展因子
    const float ext_factor;
    // 定义注意力因子
    const float attn_factor;
    // 定义快速学习率
    const float beta_fast;
    // 定义慢速学习率
    const float beta_slow;
    // 定义归一化 epsilon
    const float norm_eps;
    // 定义归一化 RMS epsilon
    const float norm_rms_eps;

    // 定义标记数量
    const int32_t n_tokens;
    // 定义键值缓存大小（考虑的 KV 缓存大小）（n_kv <= n_ctx）
    const int32_t n_kv;
    // 定义键值缓存中存储新 KV 数据的索引
    const int32_t kv_head;
    // 定义原始上下文数量
    const int32_t n_orig_ctx;

    // 定义是否进行绳索移位
    const bool do_rope_shift;
// 定义了一个名为cb的llm_build_cb类型的变量
llm_build_cb cb;

// 定义了一个名为buf_compute的llama_buffer类型的引用
llama_buffer & buf_compute;

// 定义了一个指向ggml_context类型的指针ctx0，并初始化为nullptr
struct ggml_context * ctx0 = nullptr;

// TODO: 考虑使整个接口成为noexcept
// llm_build_context构造函数，接受llama_context、llama_batch、llm_build_cb和bool类型的参数
llm_build_context(
    llama_context  & lctx,  // 引用类型参数lctx
    const llama_batch  & batch,  // 常量引用类型参数batch
    const llm_build_cb & cb,  // 常量引用类型参数cb
    bool   worst_case) :  // bool类型参数worst_case
    model         (lctx.model),  // 初始化model成员变量为lctx.model
    hparams       (model.hparams),  // 初始化hparams成员变量为model.hparams
    cparams       (lctx.cparams),  // 初始化cparams成员变量为lctx.cparams
    batch         (batch),  // 初始化batch成员变量为batch
    kv_self       (lctx.kv_self),  // 初始化kv_self成员变量为lctx.kv_self
    n_embd        (hparams.n_embd),  // 初始化n_embd成员变量为hparams.n_embd
    n_layer       (hparams.n_layer),  // 初始化n_layer成员变量为hparams.n_layer
        n_ctx         (cparams.n_ctx),  # 设置变量 n_ctx 为 cparams.n_ctx 的值
        n_head        (hparams.n_head),  # 设置变量 n_head 为 hparams.n_head 的值
        n_head_kv     (hparams.n_head_kv),  # 设置变量 n_head_kv 为 hparams.n_head_kv 的值
        n_embd_head   (hparams.n_embd_head()),  # 设置变量 n_embd_head 为 hparams.n_embd_head() 的返回值
        n_embd_gqa    (hparams.n_embd_gqa()),  # 设置变量 n_embd_gqa 为 hparams.n_embd_gqa() 的返回值
        freq_base     (cparams.rope_freq_base),  # 设置变量 freq_base 为 cparams.rope_freq_base 的值
        freq_scale    (cparams.rope_freq_scale),  # 设置变量 freq_scale 为 cparams.rope_freq_scale 的值
        ext_factor    (cparams.yarn_ext_factor),  # 设置变量 ext_factor 为 cparams.yarn_ext_factor 的值
        attn_factor   (cparams.yarn_attn_factor),  # 设置变量 attn_factor 为 cparams.yarn_attn_factor 的值
        beta_fast     (cparams.yarn_beta_fast),  # 设置变量 beta_fast 为 cparams.yarn_beta_fast 的值
        beta_slow     (cparams.yarn_beta_slow),  # 设置变量 beta_slow 为 cparams.yarn_beta_slow 的值
        norm_eps      (hparams.f_norm_eps),  # 设置变量 norm_eps 为 hparams.f_norm_eps 的值
        norm_rms_eps  (hparams.f_norm_rms_eps),  # 设置变量 norm_rms_eps 为 hparams.f_norm_rms_eps 的值
        n_tokens      (batch.n_tokens),  # 设置变量 n_tokens 为 batch.n_tokens 的值
        n_kv          (worst_case ? n_ctx            : kv_self.n),  # 根据条件设置变量 n_kv 的值
        kv_head       (worst_case ? n_ctx - n_tokens : kv_self.head),  # 根据条件设置变量 kv_head 的值
        n_orig_ctx    (cparams.n_yarn_orig_ctx),  # 设置变量 n_orig_ctx 为 cparams.n_yarn_orig_ctx 的值
        do_rope_shift (worst_case || kv_self.has_shift),  # 根据条件设置变量 do_rope_shift 的值
        cb            (cb),  # 设置变量 cb 为 cb 的值
        buf_compute   (lctx.buf_compute) {  # 设置变量 buf_compute 为 lctx.buf_compute 的值
// 断言确保 kv_self.ctx 不为空
GGML_ASSERT(!!kv_self.ctx);

// 所有的初始化应该在 init() 函数中完成
}

// 初始化函数，使用 ggml_init_params 结构体初始化参数
void init() {
    // 初始化参数结构体
    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_compute.size,  // 内存大小
        /*.mem_buffer =*/ buf_compute.data,  // 内存缓冲区
        /*.no_alloc   =*/ true,              // 不进行分配
    };
    
    // 使用初始化参数初始化上下文
    ctx0 = ggml_init(params);
}

// 释放函数，释放上下文占用的资源
void free() {
    // 如果上下文不为空
    if (ctx0) {
        // 释放上下文
        ggml_free(ctx0);
        // 将上下文指针置为空
        ctx0 = nullptr;
    // 结构体 ggml_cgraph 的指针 gf，用于构建一个自定义的图形
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);

    // 断言，确保 n_embd_head 等于 hparams.n_rot
    GGML_ASSERT(n_embd_head == hparams.n_rot);

    // 定义两个指向 ggml_tensor 结构体的指针 cur 和 inpL
    struct ggml_tensor * cur;
    struct ggml_tensor * inpL;

    // 调用 llm_build_inp_embd 函数，构建输入嵌入 inpL
    inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
    // 回调函数 cb，传入 inpL 和字符串 "inp_embd"，-1 为默认值
    cb(inpL, "inp_embd", -1);

    // 定义一个指向 ggml_tensor 结构体的指针 inp_pos，包含位置信息
    struct ggml_tensor * inp_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
    // 回调函数 cb，传入 inp_pos 和字符串 "inp_pos"，-1 为默认值
    cb(inp_pos, "inp_pos", -1);

    // 定义一个指向 ggml_tensor 结构体的指针 KQ_scale，用于缩放
    struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        // 调用函数cb，对KQ_scale进行缩放操作，并命名为"KQ_scale"，-1表示默认参数

        // 创建一个3维的浮点型张量KQ_mask，用于1个头的掩码，将会广播到所有的头
        struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
        // 调用函数cb，对KQ_mask进行操作，并命名为"KQ_mask"，-1表示默认参数

        // 如果需要进行绳索移位，则进行K-cache的移位操作
        if (do_rope_shift) {
            llm_build_k_shift(ctx0, hparams, cparams, kv_self, gf, LLM_ROPE, n_ctx, n_embd_head, freq_base, freq_scale, cb);
        }

        for (int il = 0; il < n_layer; ++il) {
            // 将inpL赋值给inpSA
            struct ggml_tensor * inpSA = inpL;

            // 构建归一化操作
            cur = llm_build_norm(ctx0, inpL, hparams,
                    model.layers[il].attn_norm, NULL,
                    LLM_NORM_RMS, cb, il);
            // 调用函数cb，对cur进行操作，并命名为"attn_norm"，il表示当前层的索引
            // 自注意力机制
            {
                // 计算 Q 和 K 并对它们进行 RoPE 处理
                struct ggml_tensor * Qcur = ggml_mul_mat(ctx0, model.layers[il].wq, cur);
                cb(Qcur, "Qcur", il);  // 回调函数，用于调试输出

                struct ggml_tensor * Kcur = ggml_mul_mat(ctx0, model.layers[il].wk, cur);
                cb(Kcur, "Kcur", il);  // 回调函数，用于调试输出

                struct ggml_tensor * Vcur = ggml_mul_mat(ctx0, model.layers[il].wv, cur);
                cb(Vcur, "Vcur", il);  // 回调函数，用于调试输出

                Qcur = ggml_rope_custom(
                    ctx0, ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head,    n_tokens), inp_pos,
                    n_embd_head, 0, 0, n_orig_ctx, freq_base, freq_scale,
                    ext_factor, attn_factor, beta_fast, beta_slow
                );
                cb(Qcur, "Qcur", il);  // 回调函数，用于调试输出

                Kcur = ggml_rope_custom(
// 使用 ggml_reshape_3d 函数对输入进行重塑，得到 ctx0
ctx0 = ggml_reshape_3d(ctx0, Kcur, n_embd_head, n_head_kv, n_tokens, inp_pos, n_embd_head, 0, 0, n_orig_ctx, freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow);
// 调用回调函数 cb，传递 Kcur 和 "Kcur" 字符串
cb(Kcur, "Kcur", il);

// 调用 llm_build_kv_store 函数，得到 k_cpy 和 v_cpy
std::tie(k_cpy, v_cpy) = llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

// 调用 llm_build_kqv 函数，得到 cur
cur = llm_build_kqv(ctx0, hparams, kv_self, model.layers[il].wo, NULL, Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, -1.0f, cb, il);
// 调用回调函数 cb，传递 cur 和 "kqv_out" 字符串
cb(cur, "kqv_out", il);

// 使用 ggml_add 函数将 cur 和 inpSA 相加，得到 ffn_inp
struct ggml_tensor * ffn_inp = ggml_add(ctx0, cur, inpSA);
// 调用回调函数 cb，传递 ffn_inp 和 "ffn_inp" 字符串
cb(ffn_inp, "ffn_inp", il);

// 调用 llm_build_norm 函数，得到 cur
cur = llm_build_norm(ctx0, ffn_inp, hparams, 
# 调用 llm_build_ffn_sparse 函数，构建稀疏的前馈神经网络
# 参数包括上一层的输出，上一层的归一化，使用的归一化方法，回调函数，以及层的索引
cur = llm_build_ffn_sparse(ctx0, cur,
    model.layers[il].ffn_up,   NULL,
    model.layers[il].ffn_gate, NULL,
    model.layers[il].ffn_down, NULL,
    model.layers[il].ffn_down_t,
    model.layers[il].mlp_pre_w1,
    model.layers[il].mlp_pre_w2,
    ffn_inp, # 目前，llama的预测使用与前馈神经网络相同的输入
    model.layers[il].gpu_idx, 
    model.layers[il].gpu_bucket, model.layers[il].ffn_gate_gpu, model.layers[il].ffn_down_gpu, model.layers[il].ffn_up_gpu, # TODO: 目前禁用 GPU 卸载
    LLM_FFN_RELU, LLM_FFN_PAR, no_offload_cb, il);

# 如果条件为真，则执行以下代码块
if (1) {
    # 调用 llm_build_ffn_sparse 函数，构建稀疏的前馈神经网络
    # 参数包括上一层的输出，上一层的归一化，使用的归一化方法，回调函数，以及层的索引
    cur = llm_build_ffn_sparse(ctx0, cur,
        model.layers[il].ffn_up,   NULL,
        model.layers[il].ffn_gate, NULL,
        model.layers[il].ffn_down, NULL,
        model.layers[il].ffn_down_t,
        model.layers[il].mlp_pre_w1,
        model.layers[il].mlp_pre_w2,
        ffn_inp, # 目前，llama的预测使用与前馈神经网络相同的输入
        model.layers[il].gpu_idx, 
        model.layers[il].gpu_bucket, model.layers[il].ffn_gate_gpu, model.layers[il].ffn_down_gpu, model.layers[il].ffn_up_gpu, # TODO: 目前禁用 GPU 卸载
        LLM_FFN_RELU, LLM_FFN_PAR, no_offload_cb, il);
} else {
    # 否则，回退到密集型前馈神经网络
    cur = llm_build_ffn(ctx0, cur,
        model.layers[il].ffn_up,   NULL,
// 对每一层的前馈神经网络进行构建
for (int il = 0; il < model.num_layers; il++) {
    // 如果存在前馈神经网络的门控单元，则构建门控单元
    if (model.layers[il].ffn_gate != NULL) {
        ggml_add(ctx0, cur, model.layers[il].ffn_gate, NULL,
                model.layers[il].ffn_down_t, NULL,
                LLM_FFN_RELU, LLM_FFN_PAR, cb, il);
    }
    // 构建前馈神经网络的输入
    cur = ggml_add(ctx0, cur, ffn_inp);
    // 回调函数，通知当前层的输出
    cb(cur, "l_out", il);
    // 为下一层构建输入
    inpL = cur;
}

// 重置当前节点
cur = inpL;

// 构建归一化层
cur = llm_build_norm(ctx0, cur, hparams,
        model.output_norm, NULL,
        LLM_NORM_RMS, cb, -1);
// 回调函数，通知结果的归一化
cb(cur, "result_norm", -1);
// 计算 lm_head
cur = ggml_mul_mat(ctx0, model.output, cur);
// 将结果输出到回调函数中
cb(cur, "result_output", -1);

// 构建前向传播扩展
ggml_build_forward_expand(gf, cur);

// 返回构建好的图
return gf;
}

// 构建稠密的 LLAMA 图
struct ggml_cgraph * build_llama_dense() {
    // 创建一个自定义大小的图
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);

    // 断言 n_embd_head 等于 hparams.n_rot
    GGML_ASSERT(n_embd_head == hparams.n_rot);

    // 声明变量
    struct ggml_tensor * cur;
    struct ggml_tensor * inpL;

    // 构建输入嵌入
    inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
    // 将输入嵌入输出到回调函数中
    cb(inpL, "inp_embd", -1);
// 创建一个包含位置信息的一维张量，长度为n_tokens
struct ggml_tensor * inp_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
cb(inp_pos, "inp_pos", -1); // 将张量注册到回调函数中

// 创建一个包含KQ_scale的一维张量，长度为1
struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
cb(KQ_scale, "KQ_scale", -1); // 将张量注册到回调函数中

// 创建一个包含KQ_mask的三维张量，维度分别为n_kv, n_tokens, 1
struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
cb(KQ_mask, "KQ_mask", -1); // 将张量注册到回调函数中

// 如果需要，对整个K-cache进行移位
if (do_rope_shift) {
    llm_build_k_shift(ctx0, hparams, cparams, kv_self, gf, LLM_ROPE, n_ctx, n_embd_head, freq_base, freq_scale, cb);
}

// 对每一层进行迭代
for (int il = 0; il < n_layer; ++il) {
    struct ggml_tensor * inpSA = inpL; // 创建一个指向inpL的张量指针
// norm
// 使用 llm_build_norm 函数构建规范化后的输出
cur = llm_build_norm(ctx0, inpL, hparams,
        model.layers[il].attn_norm, NULL,
        LLM_NORM_RMS, cb, il);
// 调用回调函数，传递当前输出和层索引
cb(cur, "attn_norm", il);

// self-attention
{
    // 计算 Q 和 K，并对它们进行 RoPE
    // 使用 ggml_mul_mat 函数计算 Qcur
    struct ggml_tensor * Qcur = ggml_mul_mat(ctx0, model.layers[il].wq, cur);
    // 调用回调函数，传递当前 Qcur 和层索引
    cb(Qcur, "Qcur", il);

    // 使用 ggml_mul_mat 函数计算 Kcur
    struct ggml_tensor * Kcur = ggml_mul_mat(ctx0, model.layers[il].wk, cur);
    // 调用回调函数，传递当前 Kcur 和层索引
    cb(Kcur, "Kcur", il);

    // 使用 ggml_mul_mat 函数计算 Vcur
    struct ggml_tensor * Vcur = ggml_mul_mat(ctx0, model.layers[il].wv, cur);
    // 调用回调函数，传递当前 Vcur 和层索引
    cb(Vcur, "Vcur", il);

    // 对 Qcur 进行 RoPE
    Qcur = ggml_rope_custom(
# 调用 ggml_reshape_3d 函数对 ctx0 和 Qcur 进行重塑，返回结果赋值给 ctx0
# 参数包括 ctx0, Qcur, n_embd_head, n_head, n_tokens 等
ctx0, ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head, n_tokens), inp_pos,
n_embd_head, 0, 0, n_orig_ctx, freq_base, freq_scale,
ext_factor, attn_factor, beta_fast, beta_slow
);
# 调用 cb 函数，将 Qcur 和 "Qcur" 作为参数传递，记录日志
cb(Qcur, "Qcur", il);

# 调用 ggml_rope_custom 函数对 ctx0 和 Kcur 进行自定义操作，返回结果赋值给 Kcur
# 参数包括 ctx0, Kcur, n_embd_head, n_head_kv, n_tokens 等
Kcur = ggml_rope_custom(
ctx0, ggml_reshape_3d(ctx0, Kcur, n_embd_head, n_head_kv, n_tokens), inp_pos,
n_embd_head, 0, 0, n_orig_ctx, freq_base, freq_scale,
ext_factor, attn_factor, beta_fast, beta_slow
);
# 调用 cb 函数，将 Kcur 和 "Kcur" 作为参数传递，记录日志
cb(Kcur, "Kcur", il);

# 调用 llm_build_kv_store 函数构建键值存储，返回结果赋值给 k_cpy 和 v_cpy
# 参数包括 ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head 等
std::tie(k_cpy, v_cpy) = llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

# 调用 llm_build_kqv 函数构建键值查询，返回结果赋值给 cur
# 参数包括 ctx0, hparams, kv_self, model.layers[il].wo, NULL, Qcur, KQ_scale, KQ_mask 等
cur = llm_build_kqv(ctx0, hparams, kv_self,
model.layers[il].wo, NULL,
Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, -1.0f, cb, il);
# 调用 cb 函数，将 cur 和 "kqv_out" 作为参数传递，记录日志
cb(cur, "kqv_out", il);
// 创建一个指向 ggml_tensor 结构的指针，指向 ggml_add 函数返回的结果
struct ggml_tensor * ffn_inp = ggml_add(ctx0, cur, inpSA);
// 调用回调函数 cb，传递 ffn_inp 指针和字符串 "ffn_inp"，以及整数 il
cb(ffn_inp, "ffn_inp", il);

// 前馈网络
{
    // 使用 llm_build_norm 函数构建正则化层，传递上下文 ctx0、输入 ffn_inp、超参数 hparams、
    // 模型层的正则化参数 model.layers[il].ffn_norm、NULL、LLM_NORM_RMS、回调函数 cb 和整数 il
    cur = llm_build_norm(ctx0, ffn_inp, hparams,
            model.layers[il].ffn_norm, NULL,
            LLM_NORM_RMS, cb, il);
    // 调用 no_offload_cb 函数，传递 cur 指针、字符串 "ffn_norm" 和整数 il
    no_offload_cb(cur, "ffn_norm", il);

    // 如果条件为真
    if (0) {
        // 使用 llm_build_ffn_sparse 函数构建稀疏前馈网络，传递上下文 ctx0、cur、
        // model.layers[il].ffn_up、NULL、model.layers[il].ffn_gate、NULL、
        // model.layers[il].ffn_down、NULL、model.layers[il].ffn_down_t、
        // model.layers[il].mlp_pre_w1、model.layers[il].mlp_pre_w2、ffn_inp
        // 目前，llama 的预测使用与前馈网络相同的输入 ffn_inp
// 对模型的每一层进行处理
for (int il = 0; il < model.num_layers; il++) {
    // 如果当前层支持 GPU 加速
    if (model.layers[il].gpu_idx >= 0) {
        // 使用 GPU 加速构建前馈神经网络
        cur = llm_build_ffn_gpu(ctx0, cur,
            model.layers[il].ffn_up_gpu, model.layers[il].ffn_gate_gpu,
            model.layers[il].ffn_down_gpu, // TODO: 目前禁用 GPU 卸载
            LLM_FFN_RELU, LLM_FFN_PAR, no_offload_cb, il);
    } else {
        // 回退到密集层
        cur = llm_build_ffn(ctx0, cur,
            model.layers[il].ffn_up,   NULL,
            model.layers[il].ffn_gate, NULL,
            model.layers[il].ffn_down_t, NULL,
            LLM_FFN_RELU, LLM_FFN_PAR, cb, il);
    }
    // 回调函数处理当前层的输出
    cb(cur, "ffn_out", il);
}

// 添加全局汇聚层
cur = ggml_add(ctx0, cur, ffn_inp);
// 回调函数处理最终输出
cb(cur, "l_out", il);

// 为下一层准备输入
inpL = cur;
// 将输入指针赋值给 cur
cur = inpL;

// 使用 llm_build_norm 函数构建规范化模型，并将结果赋值给 cur
cur = llm_build_norm(ctx0, cur, hparams, model.output_norm, NULL, LLM_NORM_RMS, cb, -1);
// 调用回调函数 cb，传递 cur 指针和字符串 "result_norm"，-1
cb(cur, "result_norm", -1);

// 使用 ggml_mul_mat 函数将 model.output 和 cur 进行矩阵乘法，并将结果赋值给 cur
cur = ggml_mul_mat(ctx0, model.output, cur);
// 调用回调函数 cb，传递 cur 指针和字符串 "result_output"，-1
cb(cur, "result_output", -1);

// 使用 ggml_build_forward_expand 函数构建前向传播扩展
ggml_build_forward_expand(gf, cur);

// 返回构建的图形对象 gf
return gf;
}

// 构建百川图形对象
struct ggml_cgraph * build_baichuan() {
    // 创建一个自定义大小的图形对象 gf
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);
// 声明指向 ggml_tensor 结构体的指针变量 cur 和 inpL
struct ggml_tensor * cur;
struct ggml_tensor * inpL;

// 调用 llm_build_inp_embd 函数构建输入嵌入 inpL
inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
// 调用回调函数 cb，传递 inpL 和字符串 "inp_embd"，-1 为标识符
cb(inpL, "inp_embd", -1);

// 声明指向 ggml_tensor 结构体的指针变量 inp_pos，包含位置信息
struct ggml_tensor * inp_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
// 调用回调函数 cb，传递 inp_pos 和字符串 "inp_pos"，-1 为标识符
cb(inp_pos, "inp_pos", -1);

// 声明指向 ggml_tensor 结构体的指针变量 KQ_scale，用于缩放 KQ
struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
// 调用回调函数 cb，传递 KQ_scale 和字符串 "KQ_scale"，-1 为标识符
cb(KQ_scale, "KQ_scale", -1);

// 声明指向 ggml_tensor 结构体的指针变量 KQ_mask，用于掩码
struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
// 调用回调函数 cb，传递 KQ_mask 和字符串 "KQ_mask"，-1 为标识符
cb(KQ_mask, "KQ_mask", -1);

// 如果需要，对整个 K-cache 进行移位
if (do_rope_shift) {
// 调用 llm_build_k_shift 函数，传入参数，构建 K-shift 操作
llm_build_k_shift(ctx0, hparams, cparams, kv_self, gf, LLM_ROPE, n_ctx, n_embd_head, freq_base, freq_scale, cb);

// 遍历每一层
for (int il = 0; il < n_layer; ++il) {
    // 将输入赋值给 inpSA
    struct ggml_tensor * inpSA = inpL;

    // 构建归一化操作
    cur = llm_build_norm(ctx0, inpL, hparams, model.layers[il].attn_norm, NULL, LLM_NORM_RMS, cb, il);
    cb(cur, "attn_norm", il);

    // self-attention
    {
        // 计算 Qcur，将 model.layers[il].wq 与 cur 相乘
        struct ggml_tensor * Qcur = ggml_mul_mat(ctx0, model.layers[il].wq, cur);
        cb(Qcur, "Qcur", il);

        // 计算 Kcur，将 model.layers[il].wk 与 cur 相乘
        struct ggml_tensor * Kcur = ggml_mul_mat(ctx0, model.layers[il].wk, cur);
        cb(Kcur, "Kcur", il);

        // 计算 Vcur，将 model.layers[il].wv 与 cur 相乘
        struct ggml_tensor * Vcur = ggml_mul_mat(ctx0, model.layers[il].wv, cur);
# 调用函数cb，传入参数Vcur, "Vcur", il
cb(Vcur, "Vcur", il);

# 根据model.type的取值进行不同的处理
switch (model.type) {
    # 当model.type为MODEL_7B时，进行以下操作
    case MODEL_7B:
        # 调用ggml_rope_custom函数，传入参数ctx0, ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head, n_tokens), inp_pos,
        # n_embd_head, 0, 0, n_orig_ctx, freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow
        Qcur = ggml_rope_custom(
            ctx0, ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head, n_tokens), inp_pos,
            n_embd_head, 0, 0, n_orig_ctx, freq_base, freq_scale,
            ext_factor, attn_factor, beta_fast, beta_slow
        );
        # 调用ggml_rope_custom函数，传入参数ctx0, ggml_reshape_3d(ctx0, Kcur, n_embd_head, n_head_kv, n_tokens), inp_pos,
        # n_embd_head, 0, 0, n_orig_ctx, freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow
        Kcur = ggml_rope_custom(
            ctx0, ggml_reshape_3d(ctx0, Kcur, n_embd_head, n_head_kv, n_tokens), inp_pos,
            n_embd_head, 0, 0, n_orig_ctx, freq_base, freq_scale,
            ext_factor, attn_factor, beta_fast, beta_slow
        );
        break;
    # 当model.type为MODEL_13B时，进行以下操作
    case MODEL_13B:
        # 调用ggml_reshape_3d函数，传入参数ctx0, Qcur, n_embd/n_head, n_head, n_tokens，并将结果赋值给Qcur
        Qcur = ggml_reshape_3d(ctx0, Qcur, n_embd/n_head, n_head, n_tokens);
        # 调用ggml_reshape_3d函数，传入参数ctx0, Kcur, n_embd/n_head, n_head, n_tokens，并将结果赋值给Kcur
        Kcur = ggml_reshape_3d(ctx0, Kcur, n_embd/n_head, n_head, n_tokens);
        break;
    # 其他情况
    default:
// 断言，如果条件为假，则终止程序
GGML_ASSERT(false);
// 调用回调函数，传入当前的查询向量和键向量，以及当前的层索引
cb(Qcur, "Qcur", il);
cb(Kcur, "Kcur", il);

// 构建键值存储
llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

// 对于13B模型应用ALiBi，设置最大的ALiBi偏差
const float max_alibi_bias = model.type == MODEL_13B ? 8.0f : -1.0f;

// 构建键-查询-值
cur = llm_build_kqv(ctx0, hparams, kv_self,
        model.layers[il].wo, NULL,
        Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, max_alibi_bias, cb, il);
cb(cur, "kqv_out", il);

// 创建输入张量，将当前的输出张量和输入自注意力张量相加
struct ggml_tensor * ffn_inp = ggml_add(ctx0, cur, inpSA);
cb(ffn_inp, "ffn_inp", il);

// 前馈神经网络
// 使用 llm_build_norm 函数构建规范化层，返回当前层的指针
cur = llm_build_norm(ctx0, ffn_inp, hparams,
                    model.layers[il].ffn_norm, NULL,
                    LLM_NORM_RMS, cb, il);
// 调用回调函数，传递当前层指针和层名称
cb(cur, "ffn_norm", il);

// 使用 llm_build_ffn 函数构建前馈神经网络层，返回当前层的指针
cur = llm_build_ffn(ctx0, cur,
                    model.layers[il].ffn_up,   NULL,
                    model.layers[il].ffn_gate, NULL,
                    model.layers[il].ffn_down, NULL,
                    LLM_FFN_SILU, LLM_FFN_PAR, cb, il);
// 调用回调函数，传递当前层指针和层名称
cb(cur, "ffn_out", il);

// 将当前层和输入层添加到 GGML 模型中
cur = ggml_add(ctx0, cur, ffn_inp);
// 调用回调函数，传递当前层指针和层名称
cb(cur, "l_out", il);

// 为下一层准备输入
inpL = cur;
// 将输入指针赋值给 cur
cur = inpL;

// 使用 llm_build_norm 函数构建正则化模型，并更新 cur 指针
cur = llm_build_norm(ctx0, cur, hparams, model.output_norm, NULL, LLM_NORM_RMS, cb, -1);
// 调用回调函数 cb，传递 cur 指针和字符串 "result_norm"，-1
cb(cur, "result_norm", -1);

// 使用 ggml_mul_mat 函数将模型输出和 cur 进行矩阵乘法，并更新 cur 指针
cur = ggml_mul_mat(ctx0, model.output, cur);
// 调用回调函数 cb，传递 cur 指针和字符串 "result_output"，-1
cb(cur, "result_output", -1);

// 使用 ggml_build_forward_expand 函数构建前向传播扩展
ggml_build_forward_expand(gf, cur);

// 返回构建好的图形
return gf;
}

// 构建 falcon 模型
struct ggml_cgraph * build_falcon() {
    // 创建一个自定义大小的图形
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);
// 定义指向 ggml_tensor 结构体的指针变量 cur 和 inpL
struct ggml_tensor * cur;
struct ggml_tensor * inpL;

// 调用 llm_build_inp_embd 函数构建输入嵌入 inpL
inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
// 调用回调函数 cb，传递 inpL 和字符串 "inp_embd"，-1 为标识符
cb(inpL, "inp_embd", -1);

// 创建一个包含位置信息的 ggml_tensor 结构体 inp_pos
struct ggml_tensor * inp_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
// 调用回调函数 cb，传递 inp_pos 和字符串 "inp_pos"，-1 为标识符
cb(inp_pos, "inp_pos", -1);

// 创建一个包含 KQ_scale 的 ggml_tensor 结构体 KQ_scale
struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
// 调用回调函数 cb，传递 KQ_scale 和字符串 "KQ_scale"，-1 为标识符
cb(KQ_scale, "KQ_scale", -1);

// 创建一个包含 KQ_mask 的 ggml_tensor 结构体，包含三维数组
struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
// 调用回调函数 cb，传递 KQ_mask 和字符串 "KQ_mask"，-1 为标识符
cb(KQ_mask, "KQ_mask", -1);

// 如果需要，对整个 K-cache 进行移位
if (do_rope_shift) {
// 调用函数 llm_build_k_shift，传入参数 ctx0, hparams, cparams, kv_self, gf, LLM_ROPE_NEOX, n_ctx, n_embd_head, freq_base, freq_scale, cb
llm_build_k_shift(ctx0, hparams, cparams, kv_self, gf, LLM_ROPE_NEOX, n_ctx, n_embd_head, freq_base, freq_scale, cb);
// 结束调用

// 循环遍历 n_layer 次
for (int il = 0; il < n_layer; ++il) {
    // 声明并初始化指向 ggml_tensor 结构的指针 attn_norm
    struct ggml_tensor * attn_norm;
    // 调用函数 llm_build_norm，传入参数 ctx0, inpL, hparams, model.layers[il].attn_norm, model.layers[il].attn_norm_b, LLM_NORM, cb, il
    attn_norm = llm_build_norm(ctx0, inpL, hparams, model.layers[il].attn_norm, model.layers[il].attn_norm_b, LLM_NORM, cb, il);
    // 结束调用

    // 注释：调用 cb 函数，传入参数 attn_norm, "attn_norm", il
    // cb(attn_norm, "attn_norm", il);

    // self-attention
    {
        // 如果 model.layers[il].attn_norm_2 存在
        if (model.layers[il].attn_norm_2) {
            // Falcon-40B
            // 调用函数 llm_build_norm，传入参数 ctx0, inpL, hparams, model.layers[il].attn_norm_2, model.layers[il].attn_norm_2_b
            cur = llm_build_norm(ctx0, inpL, hparams, model.layers[il].attn_norm_2, model.layers[il].attn_norm_2_b);
// 对输入进行归一化处理
LLM_NORM, cb, il);
cb(cur, "attn_norm_2", il);

// 如果不需要归一化，则直接使用原始输入
} else {
    cur = attn_norm;
}

// 使用注意力机制计算新的查询向量
cur = ggml_mul_mat(ctx0, model.layers[il].wqkv, cur);
cb(cur, "wqkv", il);

// 根据计算得到的查询向量，分别获取查询、键、值张量
struct ggml_tensor * Qcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd, n_tokens, cur->nb[1], 0*sizeof(float)*(n_embd)));
struct ggml_tensor * Kcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd_gqa, n_tokens, cur->nb[1], 1*sizeof(float)*(n_embd)));
struct ggml_tensor * Vcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd_gqa, n_tokens, cur->nb[1], 1*sizeof(float)*(n_embd + n_embd_gqa)));

// 输出查询、键、值张量
cb(Qcur, "Qcur", il);
cb(Kcur, "Kcur", il);
cb(Vcur, "Vcur", il);

// 将查询、键张量重塑为三维张量
Qcur = ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head, n_tokens);
Kcur = ggml_reshape_3d(ctx0, Kcur, n_embd_head, n_head_kv, n_tokens);
// 使用 neox 模式的模式 = 2
Qcur = ggml_rope_custom(
    ctx0, Qcur, inp_pos, n_embd_head, 2, 0, n_orig_ctx,
    freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow
);
// 回调函数，将 Qcur 传递给回调函数，标识为 "Qcur"，并传递当前层索引 il
cb(Qcur, "Qcur", il);

Kcur = ggml_rope_custom(
    ctx0, Kcur, inp_pos, n_embd_head, 2, 0, n_orig_ctx,
    freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow
);
// 回调函数，将 Kcur 传递给回调函数，标识为 "Kcur"，并传递当前层索引 il
cb(Kcur, "Kcur", il);

// 调用 llm_build_kv_store 函数，获取 k_cpy 和 v_cpy
std::tie(k_cpy, v_cpy) = llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

// 调用 llm_build_kqv 函数，获取 cur
cur = llm_build_kqv(ctx0, hparams, kv_self,
        model.layers[il].wo, NULL,
        Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, -1.0f, cb, il);
// 回调函数，将 cur 传递给回调函数，标识为 "kqv_out"，并传递当前层索引 il
cb(cur, "kqv_out", il);
// 将当前指针赋值给 ffn_inp
struct ggml_tensor * ffn_inp = cur;

// 前馈
attn_norm->src[3] = ffn_inp;
// cur->ne[1] 是输入长度。在提示阶段我们使用稠密的前馈神经网络以获得更好的性能
if (llama_use_sparse_inference(&model)) {
    // 如果使用稀疏推断，则构建稀疏的前馈神经网络
    cur = llm_build_ffn_sparse(ctx0, attn_norm,
        model.layers[il].ffn_up,   NULL,
        NULL, NULL,
        model.layers[il].ffn_down, NULL,
        model.layers[il].ffn_down_t,
        model.layers[il].mlp_pre_w1,
        model.layers[il].mlp_pre_w2,
        inpL,
        model.layers[il].gpu_idx, 
        model.layers[il].gpu_bucket, model.layers[il].ffn_gate_gpu, model.layers[il].ffn_down_gpu, model.layers[il].ffn_up_gpu, // TODO: 目前禁用 GPU 卸载
        LLM_FFN_RELU, LLM_FFN_SEQ, no_offload_cb, il);
} else {
    // 否则构建常规的前馈神经网络
    cur = llm_build_ffn(ctx0, attn_norm, // !! 使用注意力规范，而不是结果
// 对模型的每一层进行前向传播计算
for (int il = 0; il < model.num_layers; il++) {
    // 计算上层的前向传播结果
    ggml_ffn_forward(model.layers[il].ffn_up, NULL, NULL, model.layers[il].ffn_down_t, NULL, LLM_FFN_RELU, LLM_FFN_SEQ, cb, il);
    // 回调函数，通知前向传播计算完成
    cb(cur, "ffn_out", il);
}

// 将输入层的输出添加到当前层的输入中
cur = ggml_add(ctx0, cur, ffn_inp);
// 回调函数，通知当前层的输出
cb(cur, "l_out", il);

// 将输入层的输出添加到当前层的输入中
cur = ggml_add(ctx0, cur, inpL);
// 回调函数，通知当前层的输出
cb(cur, "l_out", il);

// 为下一层准备输入
inpL = cur;

// 将当前层的输出作为下一层的输入
cur = inpL;

// 进行归一化处理
// norm
# 调用 llm_build_norm 函数，对模型的输出进行归一化处理，并返回处理后的结果
cur = llm_build_norm(ctx0, cur, hparams, model.output_norm, model.output_norm_b, LLM_NORM, cb, -1);
# 调用回调函数，将归一化处理后的结果传递给回调函数，并标记为 "result_norm"
cb(cur, "result_norm", -1);

# 调用 ggml_mul_mat 函数，对模型的输出和归一化处理后的结果进行矩阵相乘操作
cur = ggml_mul_mat(ctx0, model.output, cur);
# 调用回调函数，将矩阵相乘后的结果传递给回调函数，并标记为 "result_output"
cb(cur, "result_output", -1);

# 调用 ggml_build_forward_expand 函数，对当前结果进行前向扩展操作
ggml_build_forward_expand(gf, cur);

# 返回构建好的图形对象
return gf;
}

# 定义一个名为 build_starcoder 的函数，用于构建星际编码器
struct ggml_cgraph * build_starcoder() {
    # 创建一个自定义大小的图形对象
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);
// 定义指向 ggml_tensor 结构体的指针 cur, pos, inpL
struct ggml_tensor * cur;
struct ggml_tensor * pos;
struct ggml_tensor * inpL;

// 调用 llm_build_inp_embd 函数构建输入嵌入 inpL
inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
// 调用回调函数 cb，传递 inpL 和字符串 "inp_embd"，-1 为标识符
cb(inpL, "inp_embd", -1);

// 创建一个包含位置信息的 ggml_tensor 结构体 inp_pos
struct ggml_tensor * inp_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
// 调用回调函数 cb，传递 inp_pos 和字符串 "inp_pos"，-1 为标识符
cb(inp_pos, "inp_pos", -1);

// 创建一个包含 KQ_scale 的 ggml_tensor 结构体
struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
// 调用回调函数 cb，传递 KQ_scale 和字符串 "KQ_scale"，-1 为标识符
cb(KQ_scale, "KQ_scale", -1);

// 创建一个包含 KQ_mask 的 ggml_tensor 结构体，包含三维数组
struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
// 调用回调函数 cb，传递 KQ_mask 和字符串 "KQ_mask"，-1 为标识符
cb(KQ_mask, "KQ_mask", -1);
        # 通过调用 ggml_get_rows 函数获取位置嵌入的值
        pos = ggml_get_rows(ctx0, model.pos_embd, inp_pos);
        # 调用回调函数，传递位置嵌入的值和标识符
        cb(pos, "pos_embd", -1);

        # 将位置嵌入的值添加到输入值中
        inpL = ggml_add(ctx0, inpL, pos);
        # 调用回调函数，传递输入值和标识符
        cb(inpL, "inpL", -1);

        # 遍历每一层
        for (int il = 0; il < n_layer; ++il) {
            # 构建归一化层
            cur = llm_build_norm(ctx0, inpL, hparams,
                    model.layers[il].attn_norm,
                    model.layers[il].attn_norm_b,
                    LLM_NORM, cb, il);
            # 调用回调函数，传递当前值和标识符
            cb(cur, "attn_norm", il);

            // 自注意力
            {
                # 通过矩阵乘法计算当前值和权重矩阵 wqkv 的乘积
                cur = ggml_mul_mat(ctx0, model.layers[il].wqkv, cur);
                # 调用回调函数，传递当前值和标识符
                cb(cur, "wqkv", il);

                # 将当前值和偏置项 bqkv 相加
                cur = ggml_add(ctx0, cur, model.layers[il].bqkv);
                # 调用回调函数，传递当前值和标识符
                cb(cur, "bqkv", il);
# 创建指向当前位置的指针 Qcur，指向一个二维视图，包含 n_embd 行和 n_tokens 列的数据，偏移量为 0*sizeof(float)*(n_embd)
struct ggml_tensor * Qcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd, n_tokens, cur->nb[1], 0*sizeof(float)*(n_embd)));
# 创建指向当前位置的指针 Kcur，指向一个二维视图，包含 n_embd_gqa 行和 n_tokens 列的数据，偏移量为 1*sizeof(float)*(n_embd)
struct ggml_tensor * Kcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd_gqa, n_tokens, cur->nb[1], 1*sizeof(float)*(n_embd)));
# 创建指向当前位置的指针 Vcur，指向一个二维视图，包含 n_embd_gqa 行和 n_tokens 列的数据，偏移量为 1*sizeof(float)*(n_embd + n_embd_gqa)
struct ggml_tensor * Vcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd_gqa, n_tokens, cur->nb[1], 1*sizeof(float)*(n_embd + n_embd_gqa)));

# 调用回调函数 cb，传入 Qcur 指针和字符串 "Qcur"，以及 il 参数
cb(Qcur, "Qcur", il);
# 调用回调函数 cb，传入 Kcur 指针和字符串 "Kcur"，以及 il 参数
cb(Kcur, "Kcur", il);
# 调用回调函数 cb，传入 Vcur 指针和字符串 "Vcur"，以及 il 参数
cb(Vcur, "Vcur", il);

# 重新调整 Qcur 指针指向的数据，将其转换为 n_embd_head 行、n_head 列、n_tokens 深度的三维数据
Qcur = ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head, n_tokens);

# 构建键值存储，传入参数 ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il
llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

# 构建键值查询向量，传入参数 ctx0, hparams, kv_self, model.layers[il].wo, model.layers[il].bo, Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, -1.0f, cb, il
cur = llm_build_kqv(ctx0, hparams, kv_self, model.layers[il].wo, model.layers[il].bo, Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, -1.0f, cb, il);
# 调用回调函数 cb，传入 cur 指针和字符串 "kqv_out"，以及 il 参数
cb(cur, "kqv_out", il);
// 创建一个指向 ggml_tensor 结构的指针，指向一个由 ggml_add 函数创建的张量
struct ggml_tensor * ffn_inp = ggml_add(ctx0, cur, inpL);
// 调用回调函数 cb，传递 ffn_inp 指针和字符串 "ffn_inp"，以及整数 il
cb(ffn_inp, "ffn_inp", il);

// 构建 FF 网络
{
    // 构建归一化层，返回新的张量指针
    cur = llm_build_norm(ctx0, ffn_inp, hparams,
            model.layers[il].ffn_norm,
            model.layers[il].ffn_norm_b,
            LLM_NORM, cb, il);
    // 调用回调函数 cb，传递 cur 指针和字符串 "ffn_norm"，以及整数 il
    cb(cur, "ffn_norm", il);

    // 构建前馈神经网络层，返回新的张量指针
    cur = llm_build_ffn(ctx0, cur,
            model.layers[il].ffn_up,   model.layers[il].ffn_up_b,
            NULL,                      NULL,
            model.layers[il].ffn_down, model.layers[il].ffn_down_b,
            LLM_FFN_GELU, LLM_FFN_SEQ, cb, il);
    // 调用回调函数 cb，传递 cur 指针和字符串 "ffn_out"，以及整数 il
    cb(cur, "ffn_out", il);
}

// 更新输入层指针，指向 ffn_inp
inpL = ggml_add(ctx0, cur, ffn_inp);
// 调用 cb 函数，处理输入层数据，并将结果存储到 "l_out" 中
cb(inpL, "l_out", il);

// 使用输入层数据构建正则化模型
cur = llm_build_norm(ctx0, inpL, hparams, model.output_norm, model.output_norm_b, LLM_NORM, cb, -1);
// 将正则化模型的结果存储到 "result_norm" 中
cb(cur, "result_norm", -1);

// 使用模型输出数据和正则化模型的结果进行矩阵乘法运算
cur = ggml_mul_mat(ctx0, model.output, cur);
// 将矩阵乘法运算的结果存储到 "result_output" 中
cb(cur, "result_output", -1);

// 构建前向传播扩展
ggml_build_forward_expand(gf, cur);

// 返回构建好的图形
return gf;
}

// 构建 persimmon 图形
struct ggml_cgraph * build_persimmon() {
    // 创建一个自定义大小的新图形
    struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);
        // 定义旋转数量为 n_embd_head 除以 2
        const int64_t n_rot = n_embd_head / 2;

        // 定义指向 ggml_tensor 结构体的指针 cur 和 inpL
        struct ggml_tensor * cur;
        struct ggml_tensor * inpL;

        // 调用 llm_build_inp_embd 函数构建输入嵌入 inpL
        inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
        // 调用回调函数 cb，传递 inpL 和字符串 "imp_embd"，-1 为标识符
        cb(inpL, "imp_embd", -1);

        // 创建一个一维的 ggml_tensor 结构体指针 inp_pos，类型为 GGML_TYPE_I32，大小为 n_tokens
        struct ggml_tensor * inp_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
        // 调用回调函数 cb，传递 inp_pos 和字符串 "inp_pos"，-1 为标识符
        cb(inp_pos, "inp_pos", -1);

        // 创建一个一维的 ggml_tensor 结构体指针 KQ_scale，类型为 GGML_TYPE_F32，大小为 1
        // 用于存储 KQ 的缩放比例
        struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        // 调用回调函数 cb，传递 KQ_scale 和字符串 "KQ_scale"，-1 为标识符
        cb(KQ_scale, "KQ_scale", -1);

        // 创建一个三维的 ggml_tensor 结构体指针 KQ_mask，类型为 GGML_TYPE_F32，大小为 n_kv * n_tokens * 1
        // 用于存储 KQ 的掩码
        struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
        // 调用回调函数 cb，传递 KQ_mask 和字符串 "KQ_mask"，-1 为标识符
        cb(KQ_mask, "KQ_mask", -1);

        // 如果需要进行绳索位移操作
        if (do_rope_shift) {
            // 调用 llm_build_k_shift 函数构建 K 位移
            llm_build_k_shift(ctx0, hparams, cparams, kv_self, gf, LLM_ROPE_NEOX, n_ctx, n_embd_head, freq_base, freq_scale, cb);
        }
        }

        // 遍历每一层神经网络
        for (int il = 0; il < n_layer; ++il) {
            // 保存输入层的指针，用于残差连接
            struct ggml_tensor * residual = inpL;

            // 构建归一化层
            cur = llm_build_norm(ctx0, inpL, hparams,
                    model.layers[il].attn_norm,
                    model.layers[il].attn_norm_b,
                    LLM_NORM, cb, il);
            cb(cur, "attn_norm", il);

            // 自注意力机制
            {
                // 计算 QKV 矩阵与输入的乘积
                cur = ggml_mul_mat(ctx0, model.layers[il].wqkv, cur);
                cb(cur, "wqkv", il);

                // 加上偏置
                cur = ggml_add(ctx0, cur, model.layers[il].bqkv);
                cb(cur, "bqkv", il);

                // 分割 QKV
# 确保头部键值对的数量与头部数量相等
GGML_ASSERT(n_head_kv == n_head);

# 重塑张量为4维，形状为(n_embd_head, 3, n_head, n_tokens)，并命名为tmpqkv
struct ggml_tensor * tmpqkv = ggml_reshape_4d(ctx0, cur, n_embd_head, 3, n_head, n_tokens);
cb(tmpqkv, "tmpqkv", il);

# 对tmpqkv进行维度置换，将维度0和3交换，维度1和2交换，并命名为tmpqkv_perm
struct ggml_tensor * tmpqkv_perm = ggml_cont(ctx0, ggml_permute(ctx0, tmpqkv, 0, 3, 1, 2));
cb(tmpqkv_perm, "tmpqkv", il);

# 从tmpqkv_perm中提取出tmpq张量，并命名为tmpq
struct ggml_tensor * tmpq = ggml_view_3d(
        ctx0, tmpqkv_perm, n_embd_head, n_head, n_tokens,
        ggml_element_size(tmpqkv_perm) * n_embd_head,
        ggml_element_size(tmpqkv_perm) * n_embd_head * n_head,
        0
        );
cb(tmpq, "tmpq", il);

# 从tmpqkv_perm中提取出tmpk张量，并命名为tmpk
struct ggml_tensor * tmpk = ggml_view_3d(
        ctx0, tmpqkv_perm, n_embd_head, n_head, n_tokens,
        ggml_element_size(tmpqkv_perm) * n_embd_head,
        ggml_element_size(tmpqkv_perm) * n_embd_head * n_head,
// 计算临时变量的大小
ggml_element_size(tmpqkv_perm) * n_embd_head * n_head * n_tokens
);

// 调用回调函数cb，传递临时变量tmpk和字符串"tmpk"，以及层索引il
cb(tmpk, "tmpk", il);

// 构建Q的Layer Norm
tmpq = llm_build_norm(ctx0, tmpq, hparams,
        model.layers[il].attn_q_norm,
        model.layers[il].attn_q_norm_b,
        LLM_NORM, cb, il);
// 调用回调函数cb，传递临时变量tmpq和字符串"tmpq"，以及层索引il
cb(tmpq, "tmpq", il);

// 构建K的Layer Norm
tmpk = llm_build_norm(ctx0, tmpk, hparams,
        model.layers[il].attn_k_norm,
        model.layers[il].attn_k_norm_b,
        LLM_NORM, cb, il);
// 调用回调函数cb，传递临时变量tmpk和字符串"tmpk"，以及层索引il
cb(tmpk, "tmpk", il);

// RoPE第一个n_rot的q/k，传递另一半，并连接。
struct ggml_tensor * qrot = ggml_view_3d(
        ctx0, tmpq, n_rot, n_head, n_tokens,
// 计算并分配内存给 qrot，表示旋转后的查询向量
struct ggml_tensor * qrot = ggml_view_3d(
                        ctx0, tmpq, n_rot, n_head, n_tokens,
                        ggml_element_size(tmpq) * n_embd_head,
                        ggml_element_size(tmpq) * n_embd_head * n_head,
                        0
                        );
// 将 qrot 添加到回调函数中，用于调试和监控
cb(qrot, "qrot", il);

// 计算并分配内存给 krot，表示旋转后的键向量
struct ggml_tensor * krot = ggml_view_3d(
                        ctx0, tmpk, n_rot, n_head, n_tokens,
                        ggml_element_size(tmpk) * n_embd_head,
                        ggml_element_size(tmpk) * n_embd_head * n_head,
                        0
                        );
// 将 krot 添加到回调函数中，用于调试和监控
cb(krot, "krot", il);

// 获取 tmpq 的后半部分，例如 tmpq[n_rot:, :, :]
struct ggml_tensor * qpass = ggml_view_3d(
                        ctx0, tmpq, n_rot, n_head, n_tokens,
                        ggml_element_size(tmpq) * n_embd_head,
                        ggml_element_size(tmpq) * n_embd_head * n_head,
                        ggml_element_size(tmpq) * n_rot
                );
                // 调用 cb 函数，将 qpass 的值和名称传递给回调函数，il 为回调函数的参数
                cb(qpass, "qpass", il);

                // 创建一个 3 维的视图，表示 kpass，参数为 ctx0, tmpk, n_rot, n_head, n_tokens
                // 以及三个表示内存大小的参数
                struct ggml_tensor * kpass = ggml_view_3d(
                        ctx0, tmpk, n_rot, n_head, n_tokens,
                        ggml_element_size(tmpk) * n_embd_head,
                        ggml_element_size(tmpk) * n_embd_head * n_head,
                        ggml_element_size(tmpk) * n_rot
                        );
                // 调用 cb 函数，将 kpass 的值和名称传递给回调函数，il 为回调函数的参数
                cb(kpass, "kpass", il);

                // 创建一个自定义的绳索，表示 qrotated，参数为 ctx0, qrot, inp_pos, n_rot, 2, 0, n_orig_ctx
                // 以及一系列参数
                struct ggml_tensor * qrotated = ggml_rope_custom(
                    ctx0, qrot, inp_pos, n_rot, 2, 0, n_orig_ctx,
                    freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow
                );
                // 调用 cb 函数，将 qrotated 的值和名称传递给回调函数，il 为回调函数的参数
                cb(qrotated, "qrotated", il);

                // 创建一个自定义的绳索，表示 krotated，参数为 ctx0, krot, inp_pos, n_rot, 2, 0, n_orig_ctx
                // 以及一系列参数
                struct ggml_tensor * krotated = ggml_rope_custom(
                    ctx0, krot, inp_pos, n_rot, 2, 0, n_orig_ctx,
                    freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow
                );
                );
                // 调用回调函数，将 krotated 传递给回调函数，同时记录日志
                cb(krotated, "krotated", il);

                // ggml 目前只支持在 dim=2 上进行连接，因此我们需要对 qrot、qpass、concat 进行排列，然后再排列回来。
                // 对 qrotated 进行排列操作
                qrotated = ggml_cont(ctx0, ggml_permute(ctx0, qrotated, 2, 1, 0, 3));
                // 调用回调函数，将 qrotated 传递给回调函数，同时记录日志
                cb(qrotated, "qrotated", il);

                // 对 krotated 进行排列操作
                krotated = ggml_cont(ctx0, ggml_permute(ctx0, krotated, 2, 1, 0, 3));
                // 调用回调函数，将 krotated 传递给回调函数，同时记录日志
                cb(krotated, "krotated", il);

                // 对 qpass 进行排列操作
                qpass = ggml_cont(ctx0, ggml_permute(ctx0, qpass, 2, 1, 0, 3));
                // 调用回调函数，将 qpass 传递给回调函数，同时记录日志
                cb(qpass, "qpass", il);

                // 对 kpass 进行排列操作
                kpass = ggml_cont(ctx0, ggml_permute(ctx0, kpass, 2, 1, 0, 3));
                // 调用回调函数，将 kpass 传递给回调函数，同时记录日志
                cb(kpass, "kpass", il);

                // 创建一个新的张量 Qcur，通过连接 qrotated 和 qpass 得到
                struct ggml_tensor * Qcur = ggml_concat(ctx0, qrotated, qpass);
                // 调用回调函数，将 Qcur 传递给回调函数，同时记录日志
                cb(Qcur, "Qcur", il);
# 创建一个新的张量 Kcur，通过连接 krotated 和 kpass 得到
struct ggml_tensor * Kcur = ggml_concat(ctx0, krotated, kpass);
# 调用回调函数 cb，传递 Kcur 张量和字符串 "Kcur"，以及整数 il
cb(Kcur, "Kcur", il);

# 创建一个新的张量 Q，通过对 Qcur 进行维度置换得到
struct ggml_tensor * Q = ggml_cont(ctx0, ggml_permute(ctx0, Qcur, 2, 1, 0, 3));
# 调用回调函数 cb，传递 Q 张量和字符串 "Q"，以及整数 il
cb(Q, "Q", il);

# 对 Kcur 进行维度置换，得到新的 Kcur
Kcur = ggml_cont(ctx0, ggml_permute(ctx0, Kcur, 2, 1, 0, 3));
# 调用回调函数 cb，传递 Kcur 张量和字符串 "Kcur"，以及整数 il
cb(Kcur, "Kcur", il);

# 创建一个新的张量 Vcur，通过对 tmpqkv_perm 进行视图操作得到
struct ggml_tensor * Vcur = ggml_view_3d(
        ctx0, tmpqkv_perm, n_embd_head, n_head, n_tokens,
        ggml_element_size(tmpqkv_perm) * n_embd_head,
        ggml_element_size(tmpqkv_perm) * n_embd_head * n_head,
        ggml_element_size(tmpqkv_perm) * n_embd_head * n_head * n_tokens * 2
        );
# 调用回调函数 cb，传递 Vcur 张量和字符串 "Vcur"，以及整数 il
cb(Vcur, "Vcur", il);

# 构建键值存储，传递参数 ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il
llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

# 待办事项：未经测试，可能存在问题
// TODO: not tested, could be broken
// 使用上下文、超参数、自身键值对、输出权重、输出偏置、查询、查询缩放、查询掩码等参数构建键值查询
cur = llm_build_kqv(ctx0, hparams, kv_self,
                    model.layers[il].wo, model.layers[il].bo,
                    Q, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, -1.0f, cb, il);
// 调用回调函数，传递当前结果和标识符
cb(cur, "kqv_out", il);

// 创建残差连接输入张量
struct ggml_tensor * ffn_inp = ggml_add(ctx0, residual, cur);
// 调用回调函数，传递当前结果和标识符
cb(ffn_inp, "ffn_inp", il);

// 前馈神经网络
{
    // 构建归一化层
    cur = llm_build_norm(ctx0, ffn_inp, hparams,
                        model.layers[il].ffn_norm,
                        model.layers[il].ffn_norm_b,
                        LLM_NORM, cb, il);
    // 调用回调函数，传递当前结果和标识符
    cb(cur, "ffn_norm", il);

    // 构建前馈神经网络层
    cur = llm_build_ffn(ctx0, cur,
                        model.layers[il].ffn_up,   model.layers[il].ffn_up_b,
                        NULL,                      NULL,
# 对模型的每一层进行操作
for il in range(model.num_layers):
    # 对当前层的下采样进行操作
    ggml_add(ctx0, cur, model.layers[il].ffn_down, model.layers[il].ffn_down_b, LLM_FFN_RELU_SQR, LLM_FFN_SEQ, cb, il);
    cb(cur, "ffn_out", il);

# 将当前结果与输入进行操作
cur = ggml_add(ctx0, cur, ffn_inp);
cb(cur, "l_out", il);

# 更新输入层
inpL = cur;

# 对输入层进行归一化操作
cur = llm_build_norm(ctx0, cur, hparams, model.output_norm, model.output_norm_b, LLM_NORM, cb, -1);
cb(cur, "result_norm", -1);

# 对模型输出进行矩阵乘法操作
cur = ggml_mul_mat(ctx0, model.output, cur);
        # 调用 cb 函数，传入 cur, "result_output", -1 作为参数
        cb(cur, "result_output", -1);

        # 调用 ggml_build_forward_expand 函数，传入 gf, cur 作为参数
        ggml_build_forward_expand(gf, cur);

        # 返回 gf
        return gf;
    }

    # 构建重构
    struct ggml_cgraph * build_refact() {
        # 创建一个自定义大小为 LLAMA_MAX_NODES 的新图形对象 gf
        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);

        # 声明变量 cur 和 inpL
        struct ggml_tensor * cur;
        struct ggml_tensor * inpL;

        # 使用 llm_build_inp_embd 函数构建 inpL
        inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
        # 调用 cb 函数，传入 inpL, "inp_embd", -1 作为参数
        cb(inpL, "inp_embd", -1);

        # 创建一个名为 KQ_scale 的新一维张量对象
        struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        # 调用 cb 函数，传入 KQ_scale, "KQ_scale", -1 作为参数
        cb(KQ_scale, "KQ_scale", -1);
// 创建一个3维的浮点类型的新张量KQ_mask，用于1个头的掩码，它将被广播到所有的头
struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
cb(KQ_mask, "KQ_mask", -1);  // 将KQ_mask张量传递给回调函数cb，并命名为"KQ_mask"，传递-1作为标识符

for (int il = 0; il < n_layer; ++il) {  // 对每一层进行循环
    struct ggml_tensor * inpSA = inpL;  // 将inpL张量赋值给inpSA

    // 构建归一化层，使用inpL张量、hparams参数、模型层的attn_norm属性，LLM_NORM_RMS模式，回调函数cb和il作为参数
    cur = llm_build_norm(ctx0, inpL, hparams, model.layers[il].attn_norm, NULL, LLM_NORM_RMS, cb, il);
    cb(cur, "attn_norm", il);  // 将cur张量传递给回调函数cb，并命名为"attn_norm"，传递il作为标识符

    // 自注意力
    {
        // 计算Qcur张量，使用model.layers[il].wq和cur张量
        struct ggml_tensor * Qcur = ggml_mul_mat(ctx0, model.layers[il].wq, cur);
        cb(Qcur, "Qcur", il);  // 将Qcur张量传递给回调函数cb，并命名为"Qcur"，传递il作为标识符

        // 计算Kcur张量，使用model.layers[il].wk和cur张量
        struct ggml_tensor * Kcur = ggml_mul_mat(ctx0, model.layers[il].wk, cur);
        cb(Kcur, "Kcur", il);  // 将Kcur张量传递给回调函数cb，并命名为"Kcur"，传递il作为标识符
# 使用 ggml_mul_mat 函数计算当前层的权重矩阵和输入数据的乘积，得到 Vcur 张量
struct ggml_tensor * Vcur = ggml_mul_mat(ctx0, model.layers[il].wv, cur);
# 调用回调函数 cb，传入 Vcur 张量和字符串 "Vcur"，用于记录当前操作
cb(Vcur, "Vcur", il);

# 使用 ggml_reshape_3d 函数对 Kcur 张量进行维度重塑
Kcur = ggml_reshape_3d(ctx0, Kcur, n_embd_head, n_head_kv, n_tokens);
# 调用回调函数 cb，传入 Kcur 张量和字符串 "Kcur"，用于记录当前操作
cb(Kcur, "Kcur", il);

# 使用 ggml_reshape_3d 函数对 Qcur 张量进行维度重塑
Qcur = ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head, n_tokens);
# 调用回调函数 cb，传入 Qcur 张量和字符串 "Qcur"，用于记录当前操作
cb(Qcur, "Qcur", il);

# 调用 llm_build_kv_store 函数构建键值存储
llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

# 调用 llm_build_kqv 函数构建键值查询向量
cur = llm_build_kqv(ctx0, hparams, kv_self, model.layers[il].wo, NULL, Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, 8.0f, cb, il);
# 调用回调函数 cb，传入 cur 张量和字符串 "kqv_out"，用于记录当前操作
cb(cur, "kqv_out", il);

# 使用 ggml_add 函数对 cur 和 inpSA 张量进行相加，得到 ffn_inp 张量
struct ggml_tensor * ffn_inp = ggml_add(ctx0, cur, inpSA);
# 调用回调函数 cb，传入 ffn_inp 张量和字符串 "ffn_inp"，用于记录当前操作
cb(ffn_inp, "ffn_inp", il);
// 前馈网络
{
    // 构建前馈网络的归一化层
    cur = llm_build_norm(ctx0, ffn_inp, hparams,
            model.layers[il].ffn_norm, NULL,
            LLM_NORM_RMS, cb, il);
    cb(cur, "ffn_norm", il);

    // 构建前馈网络的全连接层
    cur = llm_build_ffn(ctx0, cur,
            model.layers[il].ffn_up,   NULL,
            model.layers[il].ffn_gate, NULL,
            model.layers[il].ffn_down, NULL,
            LLM_FFN_SILU, LLM_FFN_PAR, cb, il);
    cb(cur, "ffn_out", il);
}

// 将前馈网络的输出与输入相加
cur = ggml_add(ctx0, cur, ffn_inp);
cb(cur, "l_out", il);

// 为下一层准备输入
inpL = cur;
        }

        // 将输入层数据赋值给当前指针
        cur = inpL;

        // 构建归一化层
        cur = llm_build_norm(ctx0, cur, hparams,
                model.output_norm, NULL,
                LLM_NORM_RMS, cb, -1);
        // 将归一化层数据传递给回调函数
        cb(cur, "result_norm", -1);

        // lm_head
        // 将模型输出与归一化层数据相乘
        cur = ggml_mul_mat(ctx0, model.output, cur);
        // 将结果数据传递给回调函数
        cb(cur, "result_output", -1);

        // 构建前向传播扩展
        ggml_build_forward_expand(gf, cur);

        // 返回构建好的计算图
        return gf;
    }

    // 构建布隆过滤器
    struct ggml_cgraph * build_bloom() {
        // 创建一个自定义大小的计算图
        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);
// 定义指向 ggml_tensor 结构体的指针 cur 和 inpL
struct ggml_tensor * cur;
struct ggml_tensor * inpL;

// 调用 llm_build_inp_embd 函数构建输入嵌入 inpL
inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
// 调用回调函数 cb，传递 inpL 和字符串 "inp_embd"，-1 为标识符
cb(inpL, "inp_embd", -1);

// 创建一个名为 KQ_scale 的一维浮点类型的 ggml_tensor 结构体
struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
// 调用回调函数 cb，传递 KQ_scale 和字符串 "KQ_scale"，-1 为标识符
cb(KQ_scale, "KQ_scale", -1);

// 创建一个名为 KQ_mask 的三维浮点类型的 ggml_tensor 结构体
// 用于表示 1 个头的掩码，将被广播到所有头
struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
// 调用回调函数 cb，传递 KQ_mask 和字符串 "KQ_mask"，-1 为标识符
cb(KQ_mask, "KQ_mask", -1);

// 调用 llm_build_norm 函数构建输入规范化 inpL
inpL = llm_build_norm(ctx0, inpL, hparams, model.tok_norm, model.tok_norm_b, LLM_NORM, cb, -1);
// 调用回调函数 cb，传递 inpL 和字符串 "inp_norm"，-1 为标识符
cb(inpL, "inp_norm", -1);
// 遍历每一层神经网络
for (int il = 0; il < n_layer; ++il) {
    // 构建当前层的归一化模块
    cur = llm_build_norm(ctx0, inpL, hparams,
            model.layers[il].attn_norm,
            model.layers[il].attn_norm_b,
            LLM_NORM, cb, il);
    // 回调函数，处理当前层的归一化模块
    cb(cur, "attn_norm", il);

    // self-attention
    {
        // 计算当前层的 QKV 矩阵乘以输入张量
        cur = ggml_mul_mat(ctx0, model.layers[il].wqkv, cur);
        // 回调函数，处理当前层的 wqkv
        cb(cur, "wqkv", il);

        // 将当前层的偏置加到结果张量上
        cur = ggml_add(ctx0, cur, model.layers[il].bqkv);
        // 回调函数，处理当前层的 bqkv
        cb(cur, "bqkv", il);

        // 将当前张量切片为 Q、K、V 三个张量
        struct ggml_tensor * Qcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd,     n_tokens, cur->nb[1], 0*sizeof(float)*(n_embd)));
        struct ggml_tensor * Kcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd_gqa, n_tokens, cur->nb[1], 1*sizeof(float)*(n_embd)));
        struct ggml_tensor * Vcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd_gqa, n_tokens, cur->nb[1], 1*sizeof(float)*(n_embd + n_embd_gqa)));
// 调用函数cb，将Qcur的值和名称"Qcur"传递给回调函数，il为参数
cb(Qcur, "Qcur", il);
// 调用函数cb，将Kcur的值和名称"Kcur"传递给回调函数，il为参数
cb(Kcur, "Kcur", il);
// 调用函数cb，将Vcur的值和名称"Vcur"传递给回调函数，il为参数
cb(Vcur, "Vcur", il);

// 重新调整Qcur的维度为3维
Qcur = ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head, n_tokens);

// 构建键值存储
llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

// 构建KQV
cur = llm_build_kqv(ctx0, hparams, kv_self,
        model.layers[il].wo, model.layers[il].bo,
        Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, 8.0f, cb, il);
// 调用函数cb，将cur的值和名称"kqv_out"传递给回调函数，il为参数
cb(cur, "kqv_out", il);

// 添加输入
struct ggml_tensor * ffn_inp = ggml_add(ctx0, cur, inpL);
// 调用函数cb，将ffn_inp的值和名称"ffn_inp"传递给回调函数，il为参数
cb(ffn_inp, "ffn_inp", il);

// 执行FF
# 使用 llm_build_norm 函数构建规范化层，返回规范化后的结果
cur = llm_build_norm(ctx0, ffn_inp, hparams,
                    model.layers[il].ffn_norm,
                    model.layers[il].ffn_norm_b,
                    LLM_NORM, cb, il);
# 调用回调函数，记录构建的层信息
cb(cur, "ffn_norm", il);

# 使用 llm_build_ffn 函数构建前馈神经网络层，返回前馈神经网络的输出
cur = llm_build_ffn(ctx0, cur,
                    model.layers[il].ffn_up,   model.layers[il].ffn_up_b,
                    NULL,                      NULL,
                    model.layers[il].ffn_down, model.layers[il].ffn_down_b,
                    LLM_FFN_GELU, LLM_FFN_SEQ, cb, il);
# 调用回调函数，记录构建的层信息
cb(cur, "ffn_out", il);

# 将当前构建的层添加到输入层列表中
inpL = ggml_add(ctx0, cur, ffn_inp);
# 调用回调函数，记录构建的层信息
cb(inpL, "l_out", il);

# 使用 llm_build_norm 函数构建规范化层，返回规范化后的结果
cur = llm_build_norm(ctx0, inpL, hparams,
                    model.output_norm,
```
这段代码主要是构建神经网络模型的层，包括规范化层和前馈神经网络层。通过不同的函数构建不同的层，并通过回调函数记录构建的层信息。
        model.output_norm_b,  // 获取模型的输出归一化偏置
        LLM_NORM, cb, -1);     // 对模型的输出进行归一化处理，并将结果传递给回调函数

        cb(cur, "result_norm", -1);  // 将归一化处理后的结果传递给回调函数，标记为"result_norm"

        cur = ggml_mul_mat(ctx0, model.output, cur);  // 使用矩阵乘法计算模型的输出
        cb(cur, "result_output", -1);  // 将输出结果传递给回调函数，标记为"result_output"

        ggml_build_forward_expand(gf, cur);  // 构建前向传播扩展

        return gf;  // 返回构建好的图

    }

    struct ggml_cgraph * build_mpt() {  // 构建 MPT 图
        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, LLAMA_MAX_NODES, false);  // 创建一个自定义的图

        struct ggml_tensor * cur;  // 当前张量
        struct ggml_tensor * inpL;  // 输入张量

        inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);  // 构建输入的嵌入张量
        cb(inpL, "inp_embd", -1);  // 将输入的嵌入张量传递给回调函数，标记为"inp_embd"
// 创建一个包含一个浮点数的一维张量 KQ_scale
struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
// 调用回调函数 cb，传入 KQ_scale 张量和字符串 "KQ_scale"，-1 为参数
cb(KQ_scale, "KQ_scale", -1);

// 创建一个包含 n_kv 行、n_tokens 列、1 个深度的三维张量 KQ_mask，用于表示 1 个头的掩码，将会被广播到所有头
struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
// 调用回调函数 cb，传入 KQ_mask 张量和字符串 "KQ_mask"，-1 为参数
cb(KQ_mask, "KQ_mask", -1);

// 遍历 n_layer 次，每次创建一个 attn_norm 张量
for (int il = 0; il < n_layer; ++il) {
    struct ggml_tensor * attn_norm;

    // 调用 llm_build_norm 函数构建一个归一化张量 attn_norm
    attn_norm = llm_build_norm(ctx0, inpL, hparams,
            model.layers[il].attn_norm,
            NULL,
            LLM_NORM, cb, il);
    // 调用回调函数 cb，传入 attn_norm 张量和字符串 "attn_norm"，il 为参数
    cb(attn_norm, "attn_norm", il);

    // self-attention
    {
# 将指针 cur 指向 attn_norm
cur = attn_norm;

# 使用矩阵相乘计算 ctx0 和 model.layers[il].wqkv 的乘积，结果存储到 cur 中
cur = ggml_mul_mat(ctx0, model.layers[il].wqkv, cur);
cb(cur, "wqkv", il);  # 回调函数，将结果 cur 和字符串 "wqkv" 传递给回调函数 cb

# 如果参数 hparams.f_clamp_kqv 大于 0.0f，则对 cur 进行范围限制，结果存储到 cur 中
if (hparams.f_clamp_kqv > 0.0f) {
    cur = ggml_clamp(ctx0, cur, -hparams.f_clamp_kqv, hparams.f_clamp_kqv);
    cb(cur, "wqkv_clamped", il);  # 回调函数，将结果 cur 和字符串 "wqkv_clamped" 传递给回调函数 cb
}

# 根据指定的参数创建 Qcur、Kcur、Vcur 三个结构体指针
struct ggml_tensor * Qcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd, n_tokens, cur->nb[1], 0*sizeof(float)*(n_embd)));
struct ggml_tensor * Kcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd_gqa, n_tokens, cur->nb[1], 1*sizeof(float)*(n_embd)));
struct ggml_tensor * Vcur = ggml_cont(ctx0, ggml_view_2d(ctx0, cur, n_embd_gqa, n_tokens, cur->nb[1], 1*sizeof(float)*(n_embd + n_embd_gqa)));

# 回调函数，将 Qcur、Kcur、Vcur 三个指针和对应的字符串传递给回调函数 cb
cb(Qcur, "Qcur", il);
cb(Kcur, "Kcur", il);
cb(Vcur, "Vcur", il);

# 重新整形 Qcur，将其变为三维张量，结果存储到 Qcur 中
Qcur = ggml_reshape_3d(ctx0, Qcur, n_embd_head, n_head, n_tokens);
// 调用函数 llm_build_kv_store，构建键值存储
llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

// 调用函数 llm_build_kqv，构建键值查询
cur = llm_build_kqv(ctx0, hparams, kv_self,
        model.layers[il].wo, NULL,
        Qcur, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, hparams.f_max_alibi_bias, cb, il);
// 回调函数，传递键值查询结果
cb(cur, "kqv_out", il);

// 添加输入
struct ggml_tensor * ffn_inp = ggml_add(ctx0, cur, inpL);
// 回调函数，传递输入结果
cb(ffn_inp, "ffn_inp", il);

// 前馈传播
{
    // 调用函数 llm_build_norm，构建归一化层
    cur = llm_build_norm(ctx0, ffn_inp, hparams,
            model.layers[il].ffn_norm,
            NULL,
            LLM_NORM, cb, il);
    // 回调函数，传递归一化层结果
    cb(cur, "ffn_norm", il);
}
# 从上一层到当前层的前馈神经网络构建
cur = llm_build_ffn(ctx0, cur,
                    model.layers[il].ffn_up,   NULL,
                    NULL,                      NULL,
                    model.layers[il].ffn_down, NULL,
                    LLM_FFN_GELU, LLM_FFN_SEQ, cb, il);
# 回调函数，处理前馈神经网络的输出
cb(cur, "ffn_out", il);

# 将前馈神经网络的输出添加到全局门控记忆单元中
cur = ggml_add(ctx0, cur, ffn_inp);
# 回调函数，处理全局门控记忆单元的输出
cb(cur, "l_out", il);

# 为下一层构建输入
inpL = cur;

# 将当前层的输入设置为上一层的输出
cur = inpL;

# 构建当前层的归一化层
cur = llm_build_norm(ctx0, cur, hparams,
                    model.output_norm,
                    NULL,
    // 创建一个新的计算图
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 定义当前张量和输入张量
    struct ggml_tensor * cur;
    struct ggml_tensor * inpL;

    // 构建输入的嵌入层张量
    inpL = llm_build_inp_embd(ctx0, hparams, batch, model.tok_embd, cb);
    // 回调函数，记录输入嵌入层张量
    cb(inpL, "inp_embd", -1);
// 创建一个包含位置信息的一维整型张量，长度为n_tokens
struct ggml_tensor * inp_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
// 调用回调函数cb，传入inp_pos张量和名称“inp_pos”，-1表示无特定标识符
cb(inp_pos, "inp_pos", -1);

// 创建一个包含KQ_scale的一维浮点型张量，长度为1
struct ggml_tensor * KQ_scale = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
// 调用回调函数cb，传入KQ_scale张量和名称“KQ_scale”，-1表示无特定标识符
cb(KQ_scale, "KQ_scale", -1);

// 创建一个包含KQ_mask的三维浮点型张量，维度分别为n_kv, n_tokens, 1
// 用于1个头的掩码，将被广播到所有头部
struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
// 调用回调函数cb，传入KQ_mask张量和名称“KQ_mask”，-1表示无特定标识符
cb(KQ_mask, "KQ_mask", -1);

// 如果需要对整个K缓存进行移位
if (do_rope_shift) {
    // 调用llm_build_k_shift函数，对K缓存进行移位
    llm_build_k_shift(ctx0, hparams, cparams, kv_self, gf, LLM_ROPE_NEOX, n_ctx, hparams.n_rot, freq_base, freq_scale, cb);
}

// 对每一层进行循环处理
for (int il = 0; il < n_layer; ++il) {
    // 将inpL赋值给inpSA
    struct ggml_tensor * inpSA = inpL;
// norm
// 调用 llm_build_norm 函数构建规范化层，传入参数 ctx0, inpL, hparams, model.layers[il].attn_norm, model.layers[il].attn_norm_b, LLM_NORM, cb, il
cur = llm_build_norm(ctx0, inpL, hparams, model.layers[il].attn_norm, model.layers[il].attn_norm_b, LLM_NORM, cb, il);
// 调用回调函数 cb，传入参数 cur, "attn_norm", il
cb(cur, "attn_norm", il);

// self-attention
{
    // compute Q and K and RoPE them
    // 计算 Q 并 RoPE 它们
    struct ggml_tensor * tmpq = ggml_mul_mat(ctx0, model.layers[il].wq, cur);
    // 调用回调函数 cb，传入参数 tmpq, "tmpq", il
    cb(tmpq, "tmpq", il);

    struct ggml_tensor * tmpk = ggml_mul_mat(ctx0, model.layers[il].wk, cur);
    // 调用回调函数 cb，传入参数 tmpk, "tmpk", il
    cb(tmpk, "tmpk", il);

    struct ggml_tensor * Vcur = ggml_mul_mat(ctx0, model.layers[il].wv, cur);
    // 调用回调函数 cb，传入参数 Vcur, "Vcur", il
    cb(Vcur, "Vcur", il);

    // RoPE the first n_rot of q/k, pass the other half, and concat.
    // RoPE q/k 的前 n_rot 部分，传递另一半，并连接起来
}
// 创建一个指向旋转后的查询张量的指针，并将其传递给回调函数
struct ggml_tensor * qrot = ggml_cont(ctx0, ggml_view_3d(
                        ctx0, tmpq, hparams.n_rot, n_head, n_tokens,
                        ggml_element_size(tmpq) * n_embd_head,
                        ggml_element_size(tmpq) * n_embd_head * n_head,
                        0
                        ));
cb(qrot, "qrot", il);

// 创建一个指向旋转后的键张量的指针，并将其传递给回调函数
struct ggml_tensor * krot = ggml_cont(ctx0, ggml_view_3d(
                        ctx0, tmpk, hparams.n_rot, n_head, n_tokens,
                        ggml_element_size(tmpk) * n_embd_head,
                        ggml_element_size(tmpk) * n_embd_head * n_head_kv,
                        0
                        ));
cb(krot, "krot", il);

// 获取tmpq的后半部分，例如tmpq[n_rot:, :, :]
struct ggml_tensor * qpass = ggml_view_3d(
                        ctx0, tmpq, (n_embd_head - hparams.n_rot), n_head, n_tokens,
                        ggml_element_size(tmpq) * n_embd_head,
# 计算临时变量 tmpq 的元素大小乘以 n_embd_head 和 n_head，得到结果
# 计算临时变量 tmpq 的元素大小乘以 hparams.n_rot，得到结果
# 创建一个名为 qpass 的结构体 ggml_tensor，表示 q 的传递
# 创建一个名为 kpass 的结构体 ggml_tensor，表示 k 的传递
# 创建一个名为 qrotated 的结构体 ggml_tensor，表示 q 的旋转
# 创建一个名为 krotated 的结构体 ggml_tensor，表示 k 的旋转
                // 调用 ggml_cont 函数对输入的张量进行连续化操作
                ctx0, krot, inp_pos, hparams.n_rot, 2, 0, n_orig_ctx,
                freq_base, freq_scale, ext_factor, attn_factor, beta_fast, beta_slow
                );
                cb(krotated, "krotated", il);

                // ggml 目前仅支持在 dim=2 上进行张量的拼接
                // 因此我们需要对 qrot, qpass, concat 进行排列，然后再排列回来
                qrotated = ggml_cont(ctx0, ggml_permute(ctx0, qrotated, 2, 1, 0, 3));
                cb(qrotated, "qrotated", il);

                // 对 krotated 进行连续化操作
                krotated = ggml_cont(ctx0, ggml_permute(ctx0, krotated, 2, 1, 0, 3));
                cb(krotated, "krotated", il);

                // 对 qpass 进行连续化操作
                qpass = ggml_cont(ctx0, ggml_permute(ctx0, qpass, 2, 1, 0, 3));
                cb(qpass, "qpass", il);

                // 对 kpass 进行连续化操作
                kpass = ggml_cont(ctx0, ggml_permute(ctx0, kpass, 2, 1, 0, 3));
                cb(kpass, "kpass", il);

                // 创建一个新的张量 Qcur，通过对 qrotated 和 qpass 进行拼接
                struct ggml_tensor * Qcur = ggml_concat(ctx0, qrotated, qpass);
                # 调用回调函数，记录当前状态 Qcur
                cb(Qcur, "Qcur", il);

                # 将 krotated 和 kpass 进行拼接，得到 Kcur
                struct ggml_tensor * Kcur = ggml_concat(ctx0, krotated, kpass);
                # 调用回调函数，记录当前状态 Kcur
                cb(Kcur, "Kcur", il);

                # 对 Qcur 进行维度变换和重排列，得到 Q
                struct ggml_tensor * Q = ggml_cont(ctx0, ggml_permute(ctx0, Qcur, 2, 1, 0, 3));
                # 调用回调函数，记录当前状态 Q
                cb(Q, "Q", il);

                # 对 Kcur 进行维度变换和重排列，得到 Kcur
                Kcur = ggml_cont(ctx0, ggml_permute(ctx0, Kcur, 2, 1, 0, 3));
                # 调用回调函数，记录当前状态 Kcur
                cb(Kcur, "Kcur", il);

                # 构建键值存储结构
                llm_build_kv_store(ctx0, hparams, kv_self, gf, Kcur, Vcur, n_ctx, n_tokens, kv_head, cb, il);

                # 构建键值查询结构
                cur = llm_build_kqv(ctx0, hparams, kv_self,
                        model.layers[il].wo, NULL,
                        Q, KQ_scale, KQ_mask, n_ctx, n_tokens, n_kv, -1.0f, cb, il);
                # 调用回调函数，记录当前状态 kqv_out
                cb(cur, "kqv_out", il);
            }

            # 将当前状态 cur 和 inpSA 进行加法操作，得到 ffn_inp
            struct ggml_tensor * ffn_inp = ggml_add(ctx0, cur, inpSA);
// 调用 cb 函数，传入 ffn_inp、"ffn_inp" 和 il 作为参数
cb(ffn_inp, "ffn_inp", il);

// 构建前馈神经网络
{
    // 使用 llm_build_norm 函数构建归一化层
    cur = llm_build_norm(ctx0, ffn_inp, hparams,
            model.layers[il].ffn_norm,
            model.layers[il].ffn_norm_b,
            LLM_NORM, cb, il);
    // 调用 cb 函数，传入 cur、"ffn_norm" 和 il 作为参数
    cb(cur, "ffn_norm", il);

    // 使用 llm_build_ffn 函数构建前馈神经网络
    cur = llm_build_ffn(ctx0, cur,
            model.layers[il].ffn_up,   NULL,
            model.layers[il].ffn_gate, NULL,
            model.layers[il].ffn_down, NULL,
            LLM_FFN_SILU, LLM_FFN_PAR, cb, il);
    // 调用 cb 函数，传入 cur、"ffn_out" 和 il 作为参数
    cb(cur, "ffn_out", il);
}

// 将 cur 和 ffn_inp 相加，并将结果赋给 cur
cur = ggml_add(ctx0, cur, ffn_inp);
// 调用 cb 函数，传入 cur、"l_out" 和 il 作为参数
cb(cur, "l_out", il);
// 将当前层的输出作为下一层的输入
inpL = cur;
```

```
// 将当前层的输出作为下一层的输入
cur = inpL;
```

```
// 使用模型参数和输入数据构建正则化层
cur = llm_build_norm(ctx0, cur, hparams,
                model.output_norm,
                model.output_norm_b,
                LLM_NORM, cb, -1);
cb(cur, "result_norm", -1);
```

```
// 使用 lm_head 对当前层的输出进行处理
cur = ggml_mul_mat(ctx0, model.output, cur);
cb(cur, "result_output", -1);
```

```
// 使用前向扩展构建前向传播图
ggml_build_forward_expand(gf, cur);
```

```
// 返回前向传播图
return gf;
    }
};

//
// tensor offloading helpers
//
// TODO: will be removed with backend v2
// 定义枚举类型，用于表示张量卸载函数的类型
enum llm_offload_func_e {
    OFFLOAD_FUNC_NOP, // 空操作
    OFFLOAD_FUNC, // 卸载函数
    OFFLOAD_FUNC_KQ, // 卸载函数 KQ
    OFFLOAD_FUNC_V, // 卸载函数 V
    OFFLOAD_FUNC_NR, // 卸载函数 NR
    OFFLOAD_FUNC_EMB, // 卸载函数 EMB
    OFFLOAD_FUNC_OUT, // 卸载函数 OUT
};

// TODO: will be removed with backend v2
// 定义结构体，用于表示张量卸载的前缀树
struct llm_offload_trie {
    // 定义一个节点结构体，用于构建字典树
    struct node {
        // 析构函数，用于释放节点的子节点
        ~node() {
            // 遍历子节点数组，如果子节点存在则释放
            for (int i = 0; i < 256; ++i) {
                if (children[i]) {
                    delete children[i];
                }
            }
        }

        // 子节点数组，用于存储子节点的指针
        node * children[256] = { nullptr };
        // 用于存储节点的函数类型
        llm_offload_func_e func = OFFLOAD_FUNC_NOP;
    };

    // 构造函数，初始化字典树的根节点
    llm_offload_trie() {
        root = new node;
    }

    // 带参数的构造函数，根据给定的映射构建字典树
    llm_offload_trie(const std::unordered_map<const char *, llm_offload_func_e> & map) {
        // 初始化字典树的根节点
        root = new node;
        // 遍历 map 中的每个键值对
        for (const auto & kv : map) {
            // 调用 add 方法，将键值对中的键和值作为参数传入
            add(kv.first, kv.second);
        }
    }

    // 析构函数，用于释放内存
    ~llm_offload_trie() {
        // 删除根节点，释放内存
        delete root;
    }

    // 添加方法，用于向字典树中添加节点
    void add(const char * name, llm_offload_func_e func) {
        // 从根节点开始遍历
        node * cur = root;

        // 遍历字符串 name 中的每个字符
        for (int i = 0; ; ++i) {
            const uint8_t c = name[i];

            // 如果遍历到字符串末尾，跳出循环
            if (!c) {
                break;
            }

            // 如果当前节点的子节点中没有字符 c 对应的节点
            if (!cur->children[c]) {
            // 如果当前节点的子节点中没有对应字符的节点，创建一个新的节点
            if (!cur->children[c]) {
                cur->children[c] = new node;
            }

            // 移动当前节点到对应字符的节点
            cur = cur->children[c];
        }

        // 将函数指针赋值给当前节点
        cur->func = func;
    }

    // 根据名称查找函数指针
    llm_offload_func_e find(const char * name) const {
        // 从根节点开始查找
        const node * cur = root;

        // 遍历名称中的字符
        for (int i = 0; ; ++i) {
            const uint8_t c = name[i];

            // 如果遍历到字符串末尾，结束循环
            if (!c) {
                break;
            }

            // 如果当前节点的子节点中没有对应字符的节点，结束循环
            if (!cur->children[c]) {
// 如果没有找到匹配的函数，则返回 OFFLOAD_FUNC_NOP
return OFFLOAD_FUNC_NOP;
}

// 遍历前缀树，查找匹配的函数
cur = cur->children[c];
}

// 返回匹配函数的指针
return cur->func;
}

// 前缀树的根节点
node * root = nullptr;
};

// TODO: 将在后端 v2 版本中移除
// 定义字符串到枚举值的映射关系
static const std::unordered_map<const char *, llm_offload_func_e> k_offload_map = {
  //{ "inp_tokens",                 OFFLOAD_FUNC_NR  }, // TODO: missing K-quants get_rows kernel
  //{ "inp_embd",                   OFFLOAD_FUNC_NR  }, // TODO: missing K-quants get_rows kernel
  // 将 "pos_embd" 映射为 OFFLOAD_FUNC_NR
  { "pos_embd",                   OFFLOAD_FUNC_NR  },

  // 将 "inp_pos" 映射为 OFFLOAD_FUNC_KQ
  { "inp_pos",                    OFFLOAD_FUNC_KQ  }, // this is often used for KQ ops (e.g. rope)
  // 将 "KQ_scale" 映射为 OFFLOAD_FUNC_KQ
  { "KQ_scale",                   OFFLOAD_FUNC_KQ  },
    { "KQ_mask",                    OFFLOAD_FUNC_KQ  },  // 将 "KQ_mask" 映射到 OFFLOAD_FUNC_KQ
    { "K_shift",                    OFFLOAD_FUNC_KQ  },  // 将 "K_shift" 映射到 OFFLOAD_FUNC_KQ
    { "K_shifted",                  OFFLOAD_FUNC_KQ  },  // 将 "K_shifted" 映射到 OFFLOAD_FUNC_KQ

    { "inp_norm",                   OFFLOAD_FUNC_NR  },  // 将 "inp_norm" 映射到 OFFLOAD_FUNC_NR
    { "inp_norm_w",                 OFFLOAD_FUNC_NR  },  // 将 "inp_norm_w" 映射到 OFFLOAD_FUNC_NR
    { "inp_norm_wb",                OFFLOAD_FUNC_NR  },  // 将 "inp_norm_wb" 映射到 OFFLOAD_FUNC_NR

    { "norm",                       OFFLOAD_FUNC     },  // 将 "norm" 映射到 OFFLOAD_FUNC
    { "norm_w",                     OFFLOAD_FUNC     },  // 将 "norm_w" 映射到 OFFLOAD_FUNC
    { "norm_wb",                    OFFLOAD_FUNC     },  // 将 "norm_wb" 映射到 OFFLOAD_FUNC

    { "attn_norm",                  OFFLOAD_FUNC     },  // 将 "attn_norm" 映射到 OFFLOAD_FUNC
    { "attn_norm_2",                OFFLOAD_FUNC     },  // 将 "attn_norm_2" 映射到 OFFLOAD_FUNC

    { "wqkv",                       OFFLOAD_FUNC_KQ  },  // 将 "wqkv" 映射到 OFFLOAD_FUNC_KQ
    { "bqkv",                       OFFLOAD_FUNC_KQ  },  // 将 "bqkv" 映射到 OFFLOAD_FUNC_KQ
    { "wqkv_clamped",               OFFLOAD_FUNC_KQ  },  // 将 "wqkv_clamped" 映射到 OFFLOAD_FUNC_KQ

    { "tmpk",                       OFFLOAD_FUNC_KQ  },  // 将 "tmpk" 映射到 OFFLOAD_FUNC_KQ
    { "tmpq",                       OFFLOAD_FUNC_KQ  },  // 将 "tmpq" 映射到 OFFLOAD_FUNC_KQ
    { "tmpv",                       OFFLOAD_FUNC_V   },  // 将 "tmpv" 映射到 OFFLOAD_FUNC_V
    { "Kcur",                       OFFLOAD_FUNC_KQ  },  // 将 "Kcur" 映射到 OFFLOAD_FUNC_KQ
    { "Qcur",                       OFFLOAD_FUNC_KQ  },  // 将 "Qcur" 映射到 OFFLOAD_FUNC_KQ
    { "Vcur",                       OFFLOAD_FUNC_V   },  // 将 "Vcur" 映射到 OFFLOAD_FUNC_V

    { "krot",                       OFFLOAD_FUNC_KQ  },  // 将 "krot" 映射到 OFFLOAD_FUNC_KQ
    { "qrot",                       OFFLOAD_FUNC_KQ  },  // 将 "qrot" 映射到 OFFLOAD_FUNC_KQ
    { "kpass",                      OFFLOAD_FUNC_KQ  },  // 将 "kpass" 映射到 OFFLOAD_FUNC_KQ
    { "qpass",                      OFFLOAD_FUNC_KQ  },  // 将 "qpass" 映射到 OFFLOAD_FUNC_KQ
    { "krotated",                   OFFLOAD_FUNC_KQ  },  // 将 "krotated" 映射到 OFFLOAD_FUNC_KQ
    { "qrotated",                   OFFLOAD_FUNC_KQ  },  // 将 "qrotated" 映射到 OFFLOAD_FUNC_KQ

    { "q",                          OFFLOAD_FUNC_KQ  },  // 将 "q" 映射到 OFFLOAD_FUNC_KQ
    { "k",                          OFFLOAD_FUNC_KQ  },  // 将 "k" 映射到 OFFLOAD_FUNC_KQ
    { "kq",                         OFFLOAD_FUNC_KQ  },  // 将 "kq" 映射到 OFFLOAD_FUNC_KQ
    { "kq_scaled",                  OFFLOAD_FUNC_KQ  },  // 将 "kq_scaled" 映射到 OFFLOAD_FUNC_KQ
    { "kq_scaled_alibi",            OFFLOAD_FUNC_KQ  },  // 将 "kq_scaled_alibi" 映射到 OFFLOAD_FUNC_KQ
    { "kq_masked",                  OFFLOAD_FUNC_KQ  },  // 将 "kq_masked" 映射到 OFFLOAD_FUNC_KQ
    { "kq_soft_max",                OFFLOAD_FUNC_V   },  // 将 "kq_soft_max" 映射到 OFFLOAD_FUNC_V
    { "v",                          OFFLOAD_FUNC_V   },  // 将字符串 "v" 映射到 OFFLOAD_FUNC_V
    { "kqv",                        OFFLOAD_FUNC_V   },  // 将字符串 "kqv" 映射到 OFFLOAD_FUNC_V
    { "kqv_merged",                 OFFLOAD_FUNC_V   },  // 将字符串 "kqv_merged" 映射到 OFFLOAD_FUNC_V
    { "kqv_merged_cont",            OFFLOAD_FUNC_V   },  // 将字符串 "kqv_merged_cont" 映射到 OFFLOAD_FUNC_V
    { "kqv_wo",                     OFFLOAD_FUNC_V   },  // 将字符串 "kqv_wo" 映射到 OFFLOAD_FUNC_V
    { "kqv_out",                    OFFLOAD_FUNC_V   },  // 将字符串 "kqv_out" 映射到 OFFLOAD_FUNC_V

    { "ffn_inp",                    OFFLOAD_FUNC     },  // 将字符串 "ffn_inp" 映射到 OFFLOAD_FUNC
    { "ffn_norm",                   OFFLOAD_FUNC     },  // 将字符串 "ffn_norm" 映射到 OFFLOAD_FUNC

    { "ffn_up",                     OFFLOAD_FUNC     },  // 将字符串 "ffn_up" 映射到 OFFLOAD_FUNC
    { "ffn_up_b",                   OFFLOAD_FUNC     },  // 将字符串 "ffn_up_b" 映射到 OFFLOAD_FUNC
    { "ffn_gate",                   OFFLOAD_FUNC     },  // 将字符串 "ffn_gate" 映射到 OFFLOAD_FUNC
    { "ffn_gate_b",                 OFFLOAD_FUNC     },  // 将字符串 "ffn_gate_b" 映射到 OFFLOAD_FUNC
    { "ffn_gate_par",               OFFLOAD_FUNC     },  // 将字符串 "ffn_gate_par" 映射到 OFFLOAD_FUNC
    { "ffn_down",                   OFFLOAD_FUNC     },  // 将字符串 "ffn_down" 映射到 OFFLOAD_FUNC
    { "ffn_down_b",                 OFFLOAD_FUNC     },  // 将字符串 "ffn_down_b" 映射到 OFFLOAD_FUNC
    { "ffn_out",                    OFFLOAD_FUNC     },  // 将字符串 "ffn_out" 映射到 OFFLOAD_FUNC

    { "ffn_silu",                   OFFLOAD_FUNC     },  // 将字符串 "ffn_silu" 映射到 OFFLOAD_FUNC
    { "ffn_gelu",                   OFFLOAD_FUNC     },  // 将 "ffn_gelu" 映射为 OFFLOAD_FUNC
    { "ffn_relu",                   OFFLOAD_FUNC     },  // 将 "ffn_relu" 映射为 OFFLOAD_FUNC
    { "ffn_sqr(relu)",              OFFLOAD_FUNC     },  // 将 "ffn_sqr(relu)" 映射为 OFFLOAD_FUNC

    { "l_out",                      OFFLOAD_FUNC     },  // 将 "l_out" 映射为 OFFLOAD_FUNC

    { "result_norm",                OFFLOAD_FUNC_EMB },  // 将 "result_norm" 映射为 OFFLOAD_FUNC_EMB
    { "result_output",              OFFLOAD_FUNC_OUT },  // 将 "result_output" 映射为 OFFLOAD_FUNC_OUT
};

static llm_offload_trie k_offload_func_trie(k_offload_map);  // 使用 k_offload_map 初始化 llm_offload_trie

static struct ggml_cgraph * llama_build_graph(
         llama_context & lctx,
     const llama_batch & batch) {
    const auto & model = lctx.model;

    // 检查是否应该构建最坏情况的图（用于内存测量）
    const bool worst_case = ggml_allocr_is_measure(lctx.alloc);  // 检查 lctx.alloc 是否需要进行内存测量
    // 跟踪已经分配的输入
    bool alloc_inp_tokens   = false; // 是否已经分配了输入 tokens
    bool alloc_inp_embd     = false; // 是否已经分配了输入 embd
    bool alloc_inp_pos      = false; // 是否已经分配了输入 pos
    bool alloc_inp_KQ_scale = false; // 是否已经分配了输入 KQ_scale
    bool alloc_inp_KQ_mask  = false; // 是否已经分配了输入 KQ_mask
    bool alloc_inp_K_shift  = false; // 是否已经分配了输入 K_shift

#ifdef GGML_USE_CUBLAS
    const bool do_offload = true; // 如果使用了 CUBLAS，则进行 offload
#else
    const bool do_offload = true; // TODO: 在重构完成后设置为 false
#endif

    int n_non_view = 0; // 已经被回调处理的非视图张量的数量

    // 此回调允许我们对每个张量应用自定义逻辑（例如 ggml-alloc、offloading 等）
    // TODO: 将在后端 v2 中删除
    llm_build_cb cb = [&](struct ggml_tensor * cur, const char * name, int il) {
        if (il >= 0) {
        // 如果 il 大于 0，则使用格式化的名称设置当前节点的名称
        ggml_format_name(cur, "%s-%d", name, il);
        // 否则，直接设置当前节点的名称
        } else {
            ggml_set_name(cur, name);
        }

        //
        // 分配输入张量并设置输入数据
        //
        // TODO: 将在后端 v2 中移除

        // 如果 alloc_inp_tokens 为假且名称与 "inp_tokens" 相同
        if (!alloc_inp_tokens && strcmp(name, "inp_tokens") == 0) {
            // 分配当前节点的内存
            ggml_allocr_alloc(lctx.alloc, cur);

            // 如果 lctx.alloc 不是度量，并且 batch.token 存在
            if (!ggml_allocr_is_measure(lctx.alloc) && batch.token) {
                // 获取当前节点的元素数量
                const int64_t n_tokens = cur->ne[0];

                // 将 batch.token 的数据复制到当前节点的数据中
                memcpy(cur->data, batch.token, n_tokens*ggml_element_size(cur));
            }

            // 设置 alloc_inp_tokens 为真
            alloc_inp_tokens = true;
        }

        // 如果输入嵌入未分配并且名称与“inp_embd”相同
        if (!alloc_inp_embd && strcmp(name, "inp_embd") == 0) {
            // 在当前上下文中分配输入嵌入
            ggml_allocr_alloc(lctx.alloc, cur);

            // 如果当前上下文不是度量，并且批处理中存在嵌入
            if (!ggml_allocr_is_measure(lctx.alloc) && batch.embd) {
                // 获取嵌入的数量和标记的数量
                const int64_t n_embd   = cur->ne[0];
                const int64_t n_tokens = cur->ne[1];

                // 将批处理中的嵌入数据复制到当前数据中
                memcpy(cur->data, batch.embd, n_tokens*n_embd*ggml_element_size(cur));
            }

            // 标记输入嵌入已分配
            alloc_inp_embd = true;
        }

        // 如果输入位置未分配并且名称与“inp_pos”相同
        if (!alloc_inp_pos && strcmp(name, "inp_pos") == 0) {
            // 在当前上下文中分配输入位置
            ggml_allocr_alloc(lctx.alloc, cur);

            // 如果当前上下文不是度量，并且批处理中存在位置
            if (!ggml_allocr_is_measure(lctx.alloc) && batch.pos) {
                // 获取标记的数量
                const int64_t n_tokens = cur->ne[0];
// 将指针 cur->data 强制转换为 int32_t 类型的指针，并赋值给 data
int32_t * data = (int32_t *) cur->data;

// 遍历 n_tokens 次，将 batch.pos 中的值赋给 data 数组
for (int i = 0; i < n_tokens; ++i) {
    data[i] = batch.pos[i];
}

// 设置 alloc_inp_pos 为 true
alloc_inp_pos = true;

// 如果 alloc_inp_KQ_scale 为 false，并且 name 与 "KQ_scale" 相同
if (!alloc_inp_KQ_scale && strcmp(name, "KQ_scale") == 0) {
    // 在 lctx.alloc 上分配内存
    ggml_allocr_alloc(lctx.alloc, cur);

    // 如果 lctx.alloc 不是度量
    if (!ggml_allocr_is_measure(lctx.alloc)) {
        // 获取模型参数中的 n_embd_head，并计算 1.0f 除以 n_embd_head 的平方根，赋值给 cur
        const int64_t n_embd_head = model.hparams.n_embd_head();
        ggml_set_f32(cur, 1.0f/sqrtf(float(n_embd_head)));
    }

    // 设置 alloc_inp_KQ_scale 为 true
    alloc_inp_KQ_scale = true;
}
        }

        // 如果 alloc_inp_KQ_mask 未分配并且文件名为 "KQ_mask"，则执行以下操作
        if (!alloc_inp_KQ_mask && strcmp(name, "KQ_mask") == 0) {
            // 分配内存
            ggml_allocr_alloc(lctx.alloc, cur);

            // 如果不是度量，则执行以下操作
            if (!ggml_allocr_is_measure(lctx.alloc)) {
                // 获取键值对数量和标记数量
                const int64_t n_kv     = cur->ne[0];
                const int64_t n_tokens = cur->ne[1];

                // 将数据转换为浮点数类型，并初始化为0
                float * data = (float *) cur->data;
                memset(data, 0, ggml_nbytes(cur));

                // 循环遍历标记数量
                for (int h = 0; h < 1; ++h) {
                    for (int j = 0; j < n_tokens; ++j) {
                        // 获取批处理中的位置和序列 ID
                        const llama_pos    pos    = batch.pos[j];
                        const llama_seq_id seq_id = batch.seq_id[j][0];

                        // 循环遍历键值对数量
                        for (int i = 0; i < n_kv; ++i) {
                            // 如果键值对中不包含序列 ID 或者键值对的位置大于当前位置，则执行以下操作
                            if (!lctx.kv_self.cells[i].has_seq_id(seq_id) || lctx.kv_self.cells[i].pos > pos) {
                                // 将数据设置为负无穷
                                data[h*(n_kv*n_tokens) + j*n_kv + i] = -INFINITY;
        // 如果输入 K_shift 未分配并且名称与 "K_shift" 相同
        if (!alloc_inp_K_shift && strcmp(name, "K_shift") == 0) {
            // 分配输入 K_shift
            ggml_allocr_alloc(lctx.alloc, cur);

            // 如果分配的不是测量
            if (!ggml_allocr_is_measure(lctx.alloc)) {
                // 获取上下文的数量
                const int64_t n_ctx = cur->ne[0];

                // 将数据转换为 int32_t 类型
                int32_t * data = (int32_t *) cur->data;

                // 遍历上下文数量，将数据赋值为 lctx.kv_self.cells[i].delta
                for (int i = 0; i < n_ctx; ++i) {
                    data[i] = lctx.kv_self.cells[i].delta;
                }
        }

        // 设置 alloc_inp_K_shift 为 true
        alloc_inp_K_shift = true;
    }

    // 如果当前节点是视图张量，则不再进行后续处理
    if (cur->view_src != nullptr) {
        return;
    }

    // 如果当前节点的操作不是 GGML_OP_NONE，则增加非视图节点计数
    if (cur->op != GGML_OP_NONE) {
        n_non_view++;
    }

    //
    // offload layers
    //
    // TODO: 将会在后端 v2 中移除

//#define LLAMA_OFFLOAD_DEBUG
```

// 如果不需要进行 offload，则直接返回
if (!do_offload) {
    return;
}

// 获取模型的层数和 GPU 层的数量
const int n_layer = model.hparams.n_layer;
const int n_gpu_layers = model.n_gpu_layers;
const int i_gpu_start  = n_layer - n_gpu_layers;

// 如果需要 offload，且当前上下文的嵌入为空，则需要 offload 最终的规范化
const bool offload_emb = lctx.embedding.empty();

// 定义一个静态的哈希表，用于存储 offload 函数类型和对应的名称
static const std::unordered_map<llm_offload_func_e, std::string, std::hash<int>> k_offload_func_name = {
    { OFFLOAD_FUNC_NOP, "CPU" },
    { OFFLOAD_FUNC_OUT, "CPU" },
#ifdef GGML_USE_CUBLAS
    { OFFLOAD_FUNC,     "GPU (CUDA)" },
    { OFFLOAD_FUNC_KQ,  "GPU (CUDA) KQ" },
    { OFFLOAD_FUNC_V,   "GPU (CUDA) V" },
// 定义一个包含 offload 函数和对应描述的静态映射表
static const std::map<llm_offload_func_e, const char*> offloadFuncMap = {
    { OFFLOAD_FUNC_NR,  "GPU (CUDA) NR" },
    { OFFLOAD_FUNC_EMB, "GPU (CUDA) EMB" },
#else
    { OFFLOAD_FUNC,     "CPU" },
    { OFFLOAD_FUNC_KQ,  "CPU" },
    { OFFLOAD_FUNC_V,   "CPU" },
    { OFFLOAD_FUNC_NR,  "CPU" },
    { OFFLOAD_FUNC_EMB, "CPU" },
#endif // GGML_USE_CUBLAS
};

// 检查全局映射表，确定该张量使用的 offload 函数
llm_offload_func_e func_e = k_offload_func_trie.find(name);

// 如果 func_e 为 OFFLOAD_FUNC_NOP，则表示该张量未被 offload，输出警告信息
#ifdef LLAMA_OFFLOAD_DEBUG
if (worst_case) {
    LLAMA_LOG_WARN("%s: %32s: not offloaded (ref: %s)\n", __func__,
            cur->name, "https://github.com/ggerganov/llama.cpp/pull/3837");
        }
#endif

        // 如果条件不满足，则返回
        return;
    }

    // 计算层数的数量，并且遵循提供的 n_gpu_layers
    switch (func_e) {
        case OFFLOAD_FUNC_NOP:
        case OFFLOAD_FUNC_OUT:
            break;
        case OFFLOAD_FUNC:
            // 如果 n_gpu_layers 小于 n_layer，则将 func_e 设置为 OFFLOAD_FUNC_NOP
            if (n_gpu_layers < n_layer) {
                if (il < i_gpu_start) {
                    func_e = OFFLOAD_FUNC_NOP;
                }
            }
            break;
        case OFFLOAD_FUNC_NR:
            // 如果 n_gpu_layers 小于等于 n_layer + 0，则执行以下操作
# 设置 func_e 为 OFFLOAD_FUNC_NOP
func_e = OFFLOAD_FUNC_NOP;
# 结束当前的 switch 语句
break;
# 当 OFFLOAD_FUNC_V 时
case OFFLOAD_FUNC_V:
    # 如果 GPU 层数小于等于当前层数加一
    if (n_gpu_layers <= n_layer + 1) {
        # 设置 func_e 为 OFFLOAD_FUNC_NOP
        func_e = OFFLOAD_FUNC_NOP;
    }
    # 结束当前的 switch 语句
    break;
# 当 OFFLOAD_FUNC_KQ 时
case OFFLOAD_FUNC_KQ:
    # 如果 GPU 层数小于等于当前层数加二
    if (n_gpu_layers <= n_layer + 2) {
        # 设置 func_e 为 OFFLOAD_FUNC_NOP
        func_e = OFFLOAD_FUNC_NOP;
    }
    # 结束当前的 switch 语句
    break;
# 当 OFFLOAD_FUNC_EMB 时
case OFFLOAD_FUNC_EMB:
    # 如果没有 offload_emb 或者 GPU 层数小于当前层数
    if (!offload_emb || n_gpu_layers < n_layer) {
        # 设置 func_e 为 OFFLOAD_FUNC_NOP
        func_e = OFFLOAD_FUNC_NOP;
    }
    # 结束当前的 switch 语句
    break;
# 默认情况下
default: GGML_ASSERT(false);
// 定义一个名为func的offload_func_t类型的变量，并初始化为ggml_offload_nop
offload_func_t func = ggml_offload_nop;

// 为了与Metal等兼容，需要使用CUBLAS时定义ggml_offload_gpu为ggml_cuda_assign_buffers_no_alloc，否则定义为ggml_offload_nop
#ifdef GGML_USE_CUBLAS
    static offload_func_t ggml_offload_gpu = ggml_cuda_assign_buffers_no_alloc;
#else
    static offload_func_t ggml_offload_gpu = ggml_offload_nop;
#endif

// 根据func_e的值进行不同的处理
switch (func_e) {
    // 如果func_e的值为OFFLOAD_FUNC_NOP或OFFLOAD_FUNC_OUT，则将func设置为ggml_offload_nop
    case OFFLOAD_FUNC_NOP:
    case OFFLOAD_FUNC_OUT: func = ggml_offload_nop; break;
    // 如果func_e的值为OFFLOAD_FUNC、OFFLOAD_FUNC_KQ、OFFLOAD_FUNC_V、OFFLOAD_FUNC_NR或OFFLOAD_FUNC_EMB，则将func设置为ggml_offload_gpu
    case OFFLOAD_FUNC:
    case OFFLOAD_FUNC_KQ:
    case OFFLOAD_FUNC_V:
    case OFFLOAD_FUNC_NR:
    case OFFLOAD_FUNC_EMB: func = ggml_offload_gpu; break;
    // 如果func_e的值不在上述情况中，则触发断言错误
    default: GGML_ASSERT(false);
}
// 对张量应用 offload 函数
func(cur);

#ifdef LLAMA_OFFLOAD_DEBUG
// 如果是最坏情况，记录 offload 函数的信息
if (worst_case) {
    LLAMA_LOG_INFO("%s: %32s: %s\n", __func__, cur->name, k_offload_func_name.at(func_e).c_str());
}
#endif
};

// 初始化结果指针为 NULL
struct ggml_cgraph * result = NULL;

// 创建构建上下文对象 llm，传入 lctx、batch、cb 和 worst_case
struct llm_build_context llm(lctx, batch, cb, worst_case);

// 初始化 llm 对象
llm.init();

// 根据模型的架构类型进行不同的处理
switch (model.arch) {
    case LLM_ARCH_LLAMA:
        {
```
// 调用 llm 对象的 build_llama 方法，返回结果
result = llm.build_llama();
// 如果 llm 对象的 n_tokens 小于 80，则调用 build_llama 方法，否则调用 build_llama_dense 方法
// if (llm.n_tokens < 80)
//     result = llm.build_llama();
// else 
//     result = llm.build_llama_dense();
// 根据不同的架构类型，调用对应的 build 方法
case LLM_ARCH_BAICHUAN:
    result = llm.build_baichuan();
    break;
case LLM_ARCH_FALCON:
    result = llm.build_falcon();
    break;
case LLM_ARCH_STARCODER:
    result = llm.build_starcoder();
    break;
case LLM_ARCH_PERSIMMON:
// 根据不同的架构类型调用不同的构建函数，并将结果赋值给result变量
switch (arch_type) {
    case LLM_ARCH_PERSIMMON:
        // 调用build_persimmon函数
        result = llm.build_persimmon();
        break;
    case LLM_ARCH_REFACT:
        // 调用build_refact函数
        result = llm.build_refact();
        break;
    case LLM_ARCH_BLOOM:
        // 调用build_bloom函数
        result = llm.build_bloom();
        break;
    case LLM_ARCH_MPT:
        // 调用build_mpt函数
        result = llm.build_mpt();
        break;
    case LLM_ARCH_STABLELM:
        // 调用build_stablelm函数
        result = llm.build_stablelm();
        break;
    default:
        // 如果架构类型不匹配，则断言失败
        GGML_ASSERT(false);
}
    // 释放内存
    llm.free();

    // 如果是最坏情况
    if (worst_case) {
        // 统计非视图张量的总数
        int n_non_view_total = 0;

        // 遍历结果中的节点，统计非视图张量的数量
        for (int i = 0; i < result->n_nodes; ++i) {
            if (result->nodes[i]->view_src == nullptr) {
                n_non_view_total++;
            }
        }

        // 输出非视图张量的处理情况
        LLAMA_LOG_INFO("%s: non-view tensors processed: %d/%d\n", __func__, n_non_view, n_non_view_total);

        // 如果非视图张量的数量不等于总数，输出警告信息
        if (n_non_view != n_non_view_total) {
            LLAMA_LOG_WARN("%s: ****************************************************************\n", __func__);
            LLAMA_LOG_WARN("%s: not all non-view tensors have been processed with a callback\n",     __func__);
            LLAMA_LOG_WARN("%s: this can indicate an inefficiency in the graph implementation\n",    __func__);
            LLAMA_LOG_WARN("%s: build with LLAMA_OFFLOAD_DEBUG for more info\n",                     __func__);
// 记录警告信息，包括函数名和相关链接
LLAMA_LOG_WARN("%s: ref: https://github.com/ggerganov/llama.cpp/pull/3837\n", __func__);
// 记录警告信息，包括函数名
LLAMA_LOG_WARN("%s: ****************************************************************\n", __func__);
}

// 返回结果
return result;
}

// 通过评估变换器来解码一批标记
//
//   - lctx:      llama 上下文
//   - batch:     要评估的批次
//
// 成功返回 0
// 警告返回正整数
// 错误返回负整数
//
static int llama_decode_internal(
         llama_context & lctx,
           llama_batch   batch) {
    // 定义一个无符号32位整数变量n_tokens，其值为batch中的token数量
    const uint32_t n_tokens = batch.n_tokens;

    // 如果token数量为0，则记录错误日志并返回-1
    if (n_tokens == 0) {
        LLAMA_LOG_ERROR("%s: n_tokens == 0", __func__);
        return -1;
    }

    // 获取模型、超参数和上下文参数的引用
    const auto & model   = lctx.model;
    const auto & hparams = model.hparams;
    const auto & cparams = lctx.cparams;

    // 获取批次的数量
    const auto n_batch = cparams.n_batch;

    // 断言token数量不超过批次数量
    GGML_ASSERT(n_tokens <= n_batch);

    // 根据token数量确定使用的线程数
    int n_threads = n_tokens == 1 ? cparams.n_threads : cparams.n_threads_batch;
    // 断言要么batch中有token而没有embd，要么有embd而没有token
    GGML_ASSERT((!batch.token && batch.embd) || (batch.token && !batch.embd)); // NOLINT

    // 获取当前时间的微秒数
    const int64_t t_start_us = ggml_time_us();
#ifdef GGML_USE_MPI
    // 如果定义了 GGML_USE_MPI 宏，则执行以下代码
    // TODO: 在 #3228 问题修复后需要修复这里的代码
    // 抛出断言错误，表示该部分代码尚未实现
    GGML_ASSERT(false && "not implemented");
    //ggml_mpi_eval_init(lctx.ctx_mpi, &n_tokens, &n_past, &n_threads);
#endif

    // 断言确保线程数大于0
    GGML_ASSERT(n_threads > 0);

    // 获取自身键值对
    auto & kv_self = lctx.kv_self;

    // 断言确保自身键值对的上下文存在
    GGML_ASSERT(!!kv_self.ctx);

    // 获取嵌入维度和词汇量
    const int64_t n_embd  = hparams.n_embd;
    const int64_t n_vocab = hparams.n_vocab;

    // 用于平滑批处理 API 过渡的辅助函数
    // 在 llama_eval 调用被弃用后，这些将被移除
    std::vector<llama_pos> pos;

    // 序列 ID 的数量
    std::vector<int32_t> n_seq_id;
    # 声明一个存储指向 llama_seq_id 对象的指针的向量
    std::vector<llama_seq_id *> seq_id_arr;
    # 声明一个存储向量的向量，每个向量存储 llama_seq_id 对象
    std::vector<std::vector<llama_seq_id>> seq_id;

    # 如果 batch 的 pos 指针为空
    if (batch.pos == nullptr) {
        # 调整 pos 的大小为 n_tokens
        pos.resize(n_tokens);
        # 为每个位置分配内存
        for (uint32_t i = 0; i < n_tokens; i++) {
            pos[i] = batch.all_pos_0 + i*batch.all_pos_1;
        }

        # 将 pos 的数据指针赋值给 batch 的 pos
        batch.pos = pos.data();
    }

    # 如果 batch 的 seq_id 指针为空
    if (batch.seq_id == nullptr) {
        # 调整 n_seq_id 的大小为 n_tokens
        n_seq_id.resize(n_tokens);
        # 调整 seq_id 的大小为 n_tokens
        seq_id.resize(n_tokens);
        # 调整 seq_id_arr 的大小为 n_tokens
        seq_id_arr.resize(n_tokens);
        # 为每个位置分配内存
        for (uint32_t i = 0; i < n_tokens; i++) {
            n_seq_id[i] = 1;
            # 调整每个位置的 seq_id 大小为 1
            seq_id[i].resize(1);
            # 将 batch 的 all_seq_id 赋值给每个位置的 seq_id
            seq_id[i][0] = batch.all_seq_id;
// 将 seq_id[i] 的数据存储到 seq_id_arr[i] 中
seq_id_arr[i] = seq_id[i].data();

// 设置 batch 结构体中的 n_seq_id 字段为 n_seq_id 的数据
batch.n_seq_id = n_seq_id.data();

// 设置 batch 结构体中的 seq_id 字段为 seq_id_arr 的数据
batch.seq_id = seq_id_arr.data();

// 如果在 kv_self 中找不到 batch 对应的槽位，则返回 1
if (!llama_kv_cache_find_slot(kv_self, batch)) {
    return 1;
}

// 一种启发式方法，用于避免在缓存尚未被使用时进行完整的访问
// 经过足够的迭代次数后，这种启发式方法的好处会消失
// 如果我们开始对缓存进行碎片整理，那么这种启发式方法的好处将变得更加重要
// 设置 kv_self.n 的值，取 cparams.n_ctx 和 llama_kv_cache_cell_max(kv_self) 的最大值，且不小于 32
kv_self.n = std::min((int32_t) cparams.n_ctx, std::max(32, llama_kv_cache_cell_max(kv_self)));

// 重置 lctx.alloc 的内存分配器
ggml_allocr_reset(lctx.alloc);
# 使用 llama_build_graph 函数构建计算图
ggml_cgraph * gf = llama_build_graph(lctx, batch);

# 在 lctx.alloc 上分配 gf 的内存
ggml_allocr_alloc_graph(lctx.alloc, gf);

# 获取计算图中最后一个节点的指针
struct ggml_tensor * res        = gf->nodes[gf->n_nodes - 1];
# 获取计算图中倒数第二个节点的指针
struct ggml_tensor * embeddings = gf->nodes[gf->n_nodes - 2];

# 断言最后一个节点的名称为 "result_output"
GGML_ASSERT(strcmp(res->name,        "result_output") == 0);
# 断言倒数第二个节点的名称为 "result_norm"
GGML_ASSERT(strcmp(embeddings->name, "result_norm")   == 0);

# 如果使用了 CUBLAS，则将 leaf 节点中的数据拷贝到 GPU 上
#ifdef GGML_USE_CUBLAS
    for (int i = 0; i < gf->n_leafs; i++) {
        ggml_tensor * node = gf->leafs[i];
        # 如果节点的后端是 GPU 并且额外数据为空，则分配 GPU 上的内存并拷贝数据
        if (node->backend == GGML_BACKEND_GPU && node->extra == NULL) {
            ggml_cuda_assign_scratch_offset(node, (char*)node->data - (char *) lctx.buf_alloc.data);
            ggml_cuda_copy_to_device(node);
        }
    }
#endif
// 遍历图中的节点，对于 GPU 后端且没有额外数据的节点，计算并分配临时偏移量
for (int i = 0; i < gf->n_nodes; i++) {
    ggml_tensor * node = gf->nodes[i];
    if (node->backend == GGML_BACKEND_GPU && node->extra == NULL) {
        ggml_cuda_assign_scratch_offset(node, (char*)node->data - (char *) lctx.buf_alloc.data);
    }
}

// 如果嵌入不为空，强制将输出放在 CPU 上
if (!lctx.embedding.empty()) {
    embeddings->backend = GGML_BACKEND_CPU;
}
// 将结果强制放在 CPU 上
res->backend = GGML_BACKEND_CPU;
#endif

// 输出图构建时间和节点、叶子节点数量的日志信息
// (ggml_time_us() - t_start_us)/1000.0 用于计算图构建时间
// gf->n_nodes 表示节点数量，gf->n_leafs 表示叶子节点数量
// 用于大型提示，如果启用了 BLAS，最好只使用一个线程
// 否则，线程会在 BLAS 调用时自旋锁等待，降低性能
// TODO: 这对于 Apple Silicon 非常重要，因为 CBLAS 仍然表现非常好
// 如果操作数的数量大于等于32，并且CPU支持BLAS但不支持GPU BLAS，则需要一些线程来处理所有非矩阵乘法操作，但不能太多以避免干扰BLAS调用。需要一个更好的解决方案

// 如果所有张量都可以在GPU上运行，那么使用多于1个线程是有害的。
const bool full_offload_supported = 
    model.arch == LLM_ARCH_LLAMA      ||
    model.arch == LLM_ARCH_BAICHUAN   ||
    model.arch == LLM_ARCH_FALCON     ||
    model.arch == LLM_ARCH_REFACT     ||
    model.arch == LLM_ARCH_MPT        ||
    model.arch == LLM_ARCH_STARCODER  ||
    model.arch == LLM_ARCH_STABLELM;

const bool fully_offloaded = model.n_gpu_layers >= (int) hparams.n_layer + 3;
// 如果CPU支持cuBLAS，并且完全卸载支持，并且已完全卸载，则使用8个线程
if (ggml_cpu_has_cublas() && full_offload_supported && fully_offloaded) {
    n_threads = 8;
}
#if GGML_USE_MPI
    // 如果使用 MPI，则获取层数并进行 MPI 图计算的预处理
    const int64_t n_layer = hparams.n_layer;
    ggml_mpi_graph_compute_pre(lctx.ctx_mpi, gf, n_layer);
#endif

#ifdef GGML_USE_METAL
    // 如果使用 Metal
    if (lctx.ctx_metal) {
        // 设置 Metal 的线程数，并进行 Metal 图计算
        ggml_metal_set_n_cb     (lctx.ctx_metal, n_threads);
        ggml_metal_graph_compute(lctx.ctx_metal, gf);
    } else {
        // 否则使用普通的图计算辅助函数
        ggml_graph_compute_helper(lctx.work_buffer, gf, n_threads);
    }
#else
    // 如果不使用 Metal，则使用普通的图计算辅助函数
    ggml_graph_compute_helper(lctx.work_buffer, gf, n_threads);
#endif

#if GGML_USE_MPI
    // 如果使用 MPI，则进行 MPI 图计算的后处理
    ggml_mpi_graph_compute_post(lctx.ctx_mpi, gf, n_layer);
#endif
// 更新 kv 环形缓冲区
{
    // 如果 kv_self 中有移位操作，将 has_shift 置为 false，并将所有单元格的 delta 置为 0
    if (kv_self.has_shift) {
        kv_self.has_shift = false;
        for (uint32_t i = 0; i < kv_self.size; ++i) {
            kv_self.cells[i].delta = 0;
        }
    }

    // 将 kv_self.head 增加 n_tokens
    kv_self.head += n_tokens;

    // 确保 kv 缓存头指向一个有效的索引
    if (kv_self.head >= kv_self.size) {
        kv_self.head = 0;
    }
}

#ifdef GGML_PERF
    // 打印每个 ggml 操作的时间信息（用于调试目的）
    // 如果定义了 GGML_PERF，则打印计算图
    ggml_graph_print(gf);
#endif

    // 以 dot 格式绘制计算图（用于调试目的）
    //if (n_past%100 == 0) {
    //    ggml_graph_dump_dot(gf, NULL, "llama.dot");
    //}

    // 提取逻辑值
    // TODO: 如果只需要嵌入，不要计算和提取逻辑值
    //       需要更新图表以跳过 "result_output"
    {
        auto & logits_out = lctx.logits;

        // 如果批次中有逻辑值
        if (batch.logits) {
            // 调整逻辑值输出的大小
            logits_out.resize(n_vocab * n_tokens);
            // 遍历标记
            for (uint32_t i = 0; i < n_tokens; i++) {
                // 如果批次中的逻辑值为 0，则跳过
                if (batch.logits[i] == 0) {
                    continue;
    // 如果需要将模型输出的logits复制到logits_out中
    if (lctx.logits) {
        // 如果需要将所有logits复制到logits_out中
        if (lctx.logits_all) {
            // 调整logits_out的大小以容纳所有logits
            logits_out.resize(n_vocab * n_tokens);
            // 将模型输出的logits复制到logits_out中
            memcpy(logits_out.data(), (float *) ggml_get_data(res), sizeof(float)*n_vocab*n_tokens);
        } 
        // 如果只需要将最后一个时间步的logits复制到logits_out中
        else {
            // 调整logits_out的大小以容纳最后一个时间步的logits
            logits_out.resize(n_vocab);
            // 将模型输出的最后一个时间步的logits复制到logits_out中
            memcpy(logits_out.data(), (float *) ggml_get_data(res) + (n_vocab*(n_tokens - 1)), sizeof(float)*n_vocab);
        }
    }

    // 如果需要提取嵌入
    if (!lctx.embedding.empty()) {
        // 获取嵌入的引用
        auto & embedding_out = lctx.embedding;
        // 调整嵌入的大小以容纳嵌入向量
        embedding_out.resize(n_embd);
        // 将模型输出的嵌入向量复制到embedding_out中
        memcpy(embedding_out.data(), (float *) ggml_get_data(embeddings) + (n_embd*(n_tokens - 1)), sizeof(float)*n_embd);
    }
// 只针对单个标记的评估进行性能测量
if (n_tokens == 1) {
    // 计算单个标记的评估时间并累加
    lctx.t_eval_us += ggml_time_us() - t_start_us;
    // 单个标记的评估次数加一
    lctx.n_eval++;
}
// 如果标记数量大于1
else if (n_tokens > 1) {
    // 计算多个标记的评估时间并累加
    lctx.t_p_eval_us += ggml_time_us() - t_start_us;
    // 多个标记的评估次数加上标记数量
    lctx.n_p_eval += n_tokens;
}

// 在第一次评估时获取更准确的加载时间
// TODO: 修复这个问题
if (!lctx.has_evaluated_once) {
    // 计算加载时间并赋值给 lctx.t_load_us
    lctx.t_load_us = ggml_time_us() - lctx.t_start_us;
    // 将 has_evaluated_once 标记为 true
    lctx.has_evaluated_once = true;
}

// 返回 0
return 0;
// 获取词汇表中词汇的类型
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
// 检查给定的标记是否为字节类型的标记
static bool llama_is_byte_token(const llama_vocab & vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_BYTE;
}

// 检查给定的标记是否为用户定义的标记
static bool llama_is_user_defined_token(const llama_vocab& vocab, llama_token id) {
    return vocab.id_to_token[id].type == LLAMA_TOKEN_TYPE_USER_DEFINED;
}

// 将标记转换为字节类型的数据
static uint8_t llama_token_to_byte(const llama_vocab& vocab, llama_token id) {
    GGML_ASSERT(llama_is_byte_token(vocab, id));
    // 获取标记对应的数据
    const auto& token_data = vocab.id_to_token.at(id);
    // 根据词汇表的类型进行不同的处理
    switch (llama_vocab_get_type(vocab)) {
    // 如果词汇表类型为 SPM
    case LLAMA_VOCAB_TYPE_SPM: {
        // 从标记数据中提取字节数据并转换为十六进制
        auto buf = token_data.text.substr(3, 2);
        return strtol(buf.c_str(), NULL, 16);
    }
    // 如果词汇表类型为 BPE
    case LLAMA_VOCAB_TYPE_BPE: {
        // 抛出断言错误，不支持将 BPE 标记转换为字节数据
        GGML_ASSERT(false);
        return unicode_to_bytes_bpe(token_data.text);
    }
// 默认情况下，如果不匹配任何情况，触发断言错误
default:
    GGML_ASSERT(false);
}

// 将字节转换为标记
static llama_token llama_byte_to_token(const llama_vocab & vocab, uint8_t ch) {
    // 定义十六进制字符
    static const char * hex = "0123456789ABCDEF";
    // 根据词汇表类型进行切换
    switch (llama_vocab_get_type(vocab)) {
    // 如果词汇表类型为 SPM
    case LLAMA_VOCAB_TYPE_SPM: {
        // 创建包含十六进制表示的字符数组
        const char buf[7] = { '<', '0', 'x', hex[ch >> 4], hex[ch & 15], '>', 0 };
        // 返回对应的标记
        return vocab.token_to_id.at(buf);
    }
    // 如果词汇表类型为 BPE
    case LLAMA_VOCAB_TYPE_BPE: {
        // 将字节转换为 Unicode，并返回对应的标记
        return vocab.token_to_id.at(bytes_to_unicode_bpe(ch));
    }
    // 默认情况下，触发断言错误
    default:
        GGML_ASSERT(false);
    }
}
// 用于将字符串中的空格替换为特定的字符串
static void llama_escape_whitespace(std::string & text) {
    replace_all(text, " ", "\xe2\x96\x81");
}

// 用于将特定的字符串替换为空格
static void llama_unescape_whitespace(std::string & word) {
    replace_all(word, "\xe2\x96\x81", " ");
}

// 定义了一个结构体 llm_symbol，包含了索引、前一个索引、后一个索引、文本和大小
struct llm_symbol {
    using index = int;
    index prev;
    index next;
    const char * text;
    size_t n;
};

// 静态断言，检查 llm_symbol 是否是平凡可复制的
static_assert(std::is_trivially_copyable<llm_symbol>::value, "llm_symbol is not trivially copyable");

// SPM tokenizer
// 原始实现：
// 定义了一个结构体 llm_bigram_spm，包括了比较器和优先队列的定义
struct llm_bigram_spm {
    // 定义了一个比较器，用于比较两个 llm_bigram_spm 结构体的大小关系
    struct comparator {
        bool operator()(llm_bigram_spm & l, llm_bigram_spm & r) {
            return (l.score < r.score) || (l.score == r.score && l.left > r.left);
        }
    };
    // 定义了优先队列的存储类型和优先队列的类型
    using queue_storage = std::vector<llm_bigram_spm>;
    using queue = std::priority_queue<llm_bigram_spm, queue_storage, comparator>;
    // 定义了左右符号的索引、得分、大小
    llm_symbol::index left;
    llm_symbol::index right;
    float score;
    size_t size;
};

// 定义了一个结构体 llm_tokenizer_spm，包括了词汇表的定义
struct llm_tokenizer_spm {
    // 构造函数，接受一个 llama_vocab 类型的词汇表作为参数
    llm_tokenizer_spm(const llama_vocab & vocab): vocab(vocab) {}

    // 定义了一个函数 tokenize，接受一个字符串和一个输出向量作为参数
    void tokenize(const std::string & text, std::vector<llama_vocab::id> & output) {
// 将字符串拆分成 UTF-8 字符
int index = 0; // 初始化索引
size_t offs = 0; // 初始化偏移量
while (offs < text.size()) { // 循环直到偏移量小于字符串长度
    llm_symbol sym; // 创建符号对象
    size_t len = utf8_len(text[offs]); // 获取 UTF-8 字符的长度
    sym.text = text.c_str() + offs; // 设置符号对象的文本内容
    sym.n = std::min(len, text.size() - offs); // 设置符号对象的长度
    offs += sym.n; // 更新偏移量
    sym.prev = index - 1; // 设置符号对象的前一个索引
    sym.next = offs == text.size() ? -1 : index + 1; // 设置符号对象的下一个索引
    index++; // 更新索引
    symbols.emplace_back(sym); // 将符号对象添加到符号列表中
}

// 使用所有可能的两个字符标记初始化工作队列
for (size_t i = 1; i < symbols.size(); ++i) { // 遍历符号列表
    try_add_bigram(i - 1, i); // 尝试添加两个字符的标记
}
// 只要工作队列不为空，就不断替换最高频率的符号对。
while (!work_queue.empty()) {
    // 获取工作队列中的最高频率的符号对
    auto bigram = work_queue.top();
    // 弹出队列中的最高频率的符号对
    work_queue.pop();

    // 获取左右符号的引用
    auto & left_sym = symbols[bigram.left];
    auto & right_sym = symbols[bigram.right];

    // 如果其中一个符号已经被合并，则跳过
    if (left_sym.n == 0 || right_sym.n == 0 ||
        left_sym.n + right_sym.n != bigram.size) {
        continue;
    }

    // 将右符号合并到左符号中
    left_sym.n += right_sym.n;
    right_sym.n = 0;

    // 记录合并后的左符号和大小
    //LLAMA_LOG_INFO("left = '%*s' size = %zu\n", (int) left_sym.n, left_sym.text, bigram.size);
}
// 从链表中移除右侧符号
left_sym.next = right_sym.next;
// 如果右侧符号的下一个符号存在，则更新其前一个符号的值
if (right_sym.next >= 0) {
    symbols[right_sym.next].prev = bigram.left;
}

// 寻找更多的替换
try_add_bigram(left_sym.prev, bigram.left);
try_add_bigram(bigram.left, left_sym.next);
}

// 遍历符号数组，对每个符号重新分割
for (int i = 0; i != -1; i = symbols[i].next) {
    auto & symbol = symbols[i];
    resegment(symbol, output);
}
}

// 重新分割符号的私有方法
private:
void resegment(llm_symbol & symbol, std::vector<llama_vocab::id> & output) {
    // 将符号的文本转换为字符串
    auto text = std::string(symbol.text, symbol.n);
        // 在词汇表中查找文本对应的标记
        auto token = vocab.token_to_id.find(text);

        // 是否需要支持 is_unused？
        if (token != vocab.token_to_id.end()) {
            // 如果找到了对应的标记，将其添加到输出中并返回
            output.push_back((*token).second);
            return;
        }

        // 在反向合并表中查找文本
        const auto p = rev_merge.find(text);

        // 如果在反向合并表中找不到文本
        if (p == rev_merge.end()) {
            // 输出未形成标记的任何符号作为字节
            for (int j = 0; j < (int)symbol.n; ++j) {
                // 将符号转换为标记并添加到输出中
                llama_vocab::id token_id = llama_byte_to_token(vocab, symbol.text[j]);
                output.push_back(token_id);
            }
            return;
        }

        // 重新分割符号并输出
        resegment(symbols[p->second.first],  output);
    // 重新分割符号序列，并将结果输出
    resegment(symbols[p->second.second], output);
    }

    // 尝试添加二元组
    void try_add_bigram(int left, int right) {
        // 如果左边或右边的索引为-1，则返回
        if (left == -1 || right == -1) {
            return;
        }

        // 将左右符号的文本合并成一个字符串
        const std::string text = std::string(symbols[left].text, symbols[left].n + symbols[right].n);
        // 查找合并后的字符串在词汇表中的索引
        auto token = vocab.token_to_id.find(text);

        // 如果在词汇表中找不到合并后的字符串，则返回
        if (token == vocab.token_to_id.end()) {
            return;
        }

        // 如果在词汇表中找到了合并后的字符串，但索引超出了词汇表的大小，则返回
        if (static_cast<size_t>((*token).second) >= vocab.id_to_token.size()) {
            return;
        }

        // 获取合并后字符串在词汇表中的数据
        const auto & tok_data = vocab.id_to_token[(*token).second];
// 创建一个llm_bigram_spm对象
llm_bigram_spm bigram;
// 设置bigram对象的左右值、分数和大小
bigram.left  = left;
bigram.right = right;
bigram.score = tok_data.score;
bigram.size  = text.size();

// 将bigram对象加入工作队列
work_queue.push(bigram);

// 是否需要支持is_unused？
// 将文本和其左右值作为键值对加入到rev_merge映射中
rev_merge[text] = std::make_pair(left, right);

// llama_vocab对象的引用
const llama_vocab & vocab;

// 存储llm_symbol对象的向量
std::vector<llm_symbol> symbols;

// 存储llm_bigram_spm对象的队列
llm_bigram_spm::queue work_queue;

// 存储字符串和其左右值的映射
std::map<std::string, std::pair<int, int>> rev_merge;
// 定义了一个结构体 llm_bigram_bpe，用于存储双字节的BPE（Byte Pair Encoding）信息
// 定义了一个比较器，用于比较两个 llm_bigram_bpe 对象的大小关系
// 使用了 std::priority_queue 定义了一个优先队列，用于存储 llm_bigram_bpe 对象
// 定义了结构体的成员变量 left 和 right，用于存储双字节的索引
// 定义了结构体的成员变量 text，用于存储文本信息
// 定义了结构体的成员变量 rank，用于存储双字节的排名
// 定义一个结构体，用于存储大小信息
struct llm_tokenizer_bpe {
    size_t size;
};

// 定义一个 BPE 分词器，使用给定的词汇表
struct llm_tokenizer_bpe {
    llm_tokenizer_bpe(const llama_vocab & vocab): vocab(vocab) {}

    // 对输入的文本进行分词处理，将结果存储在输出向量中
    void tokenize(const std::string & text, std::vector<llama_vocab::id> & output) {
        // 初始化前一个索引为-1
        int final_prev_index = -1;
        // 对文本进行 BPE 预处理，得到词语集合
        auto word_collection = bpe_gpt2_preprocess(text);

        // 清空符号最终集合
        symbols_final.clear();

        // 遍历词语集合
        for (auto & word : word_collection) {
            // 初始化工作队列
            work_queue = llm_bigram_bpe::queue();
            // 清空符号集合
            symbols.clear();

            // 初始化索引和偏移量
            int index = 0;
            size_t offset = 0;

            // 循环处理词语中的每个字符
            while (offset < word.size()) {
// 定义llm_symbol类型的变量sym
llm_symbol sym;
// 计算字符长度，取word.size() - offset和utf8_len(word[offset])的最小值
size_t char_len = std::min(word.size() - offset, (size_t) ::utf8_len(word[offset]));
// 设置sym的文本为word.c_str() + offset
sym.text = word.c_str() + offset;
// 设置sym的n为char_len
sym.n = char_len;
// 更新offset
offset += sym.n;
// 设置sym的prev为index - 1
sym.prev = index - 1;
// 如果offset等于word.size()，设置sym的next为-1，否则设置为index + 1
sym.next = offset == word.size() ? -1 : index + 1;
// 更新index
index++;
// 将sym添加到symbols向量中
symbols.emplace_back(sym);
// 遍历symbols向量，对每个元素调用add_new_bigram函数
for (size_t i = 1; i < symbols.size(); ++i) {
    add_new_bigram(i - 1, i);
}

// 构建token
while (!work_queue.empty()) {
    // 从work_queue中取出一个bigram
    auto bigram = work_queue.top();
    work_queue.pop();
    // 获取bigram左侧的符号
    auto & left_symbol = symbols[bigram.left];
// 获取右侧符号的引用
auto & right_symbol = symbols[bigram.right];

// 如果左侧符号或右侧符号的长度为0，则跳过
if (left_symbol.n == 0 || right_symbol.n == 0) {
    continue;
}

// 将左侧符号和右侧符号转换为字符串
std::string left_token = std::string(left_symbol.text, left_symbol.n);
std::string right_token = std::string(right_symbol.text, right_symbol.n);

// 如果左侧符号和右侧符号拼接起来不等于bigram的文本，则跳过
if (left_token + right_token != bigram.text) {
    continue;  // 如果过时，则跳过此bigram
}

// 将右侧符号合并到左侧符号中
left_symbol.n += right_symbol.n;
right_symbol.n = 0;

// 从链中移除右侧符号
left_symbol.next = right_symbol.next;
if (right_symbol.next >= 0) {
    symbols[right_symbol.next].prev = bigram.left;
}
// 将当前符号的左侧添加为新的双字母组合
add_new_bigram(left_symbol.prev, bigram.left);

// 将当前符号的右侧添加为新的双字母组合
add_new_bigram(bigram.left, left_symbol.next);

// 将已完成的标记添加到最终列表中，保持正确的顺序
for (auto & sym : symbols) {
    if (sym.n > 0) {
        // 设置前一个和后一个标记的正确顺序
        sym.prev = final_prev_index;
        sym.next = -1;
        // 如果存在前一个标记，则更新其后一个标记的索引
        if (final_prev_index != -1) {
            symbols_final[final_prev_index].next = symbols_final.size();
        }
        // 将标记添加到最终列表中
        symbols_final.emplace_back(sym);
        final_prev_index = symbols_final.size() - 1;
    }
}

// 将 symbols_final 赋值给 symbols
symbols = symbols_final;
// 检查符号集合是否为空
if (!symbols.empty()) {
    // 遍历符号集合
    for (int i = 0; i != -1; i = symbols[i].next) {
        // 获取当前符号
        auto & symbol = symbols[i];
        // 如果符号长度为0，则跳过
        if (symbol.n == 0) {
            continue;
        }

        // 将符号转换为字符串
        const std::string str = std::string(symbol.text, symbol.n);
        // 在词汇表中查找符号对应的标记
        const auto token = vocab.token_to_id.find(str);

        // 如果符号不在词汇表中
        if (token == vocab.token_to_id.end()) {
            // 遍历符号的每个字节
            for (auto j = str.begin(); j != str.end(); ++j) {
                // 将字节转换为字符串
                std::string byte_str(1, *j);
                // 在词汇表中查找字节对应的标记
                auto token_multibyte = vocab.token_to_id.find(byte_str);
                // 如果字节不在词汇表中，抛出运行时错误
                if (token_multibyte == vocab.token_to_id.end()) {
                    throw std::runtime_error("ERROR: byte not found in vocab");
                }
                // 将字节对应的标记添加到输出中
                output.push_back((*token_multibyte).second);
            }
        }
    }
}
                } else {
                    // 如果条件不满足，将当前token的值添加到输出向量中
                    output.push_back((*token).second);
                }
            }
        }
    }

private:
    // 添加新的双字母组合
    void add_new_bigram(int left, int right) {
        // 如果左边或右边的索引为-1，直接返回
        if (left == -1 || right == -1) {
            return;
        }

        // 将左边和右边的符号转换为字符串
        std::string left_token  = std::string(symbols[left].text,  symbols[left].n);
        std::string right_token = std::string(symbols[right].text, symbols[right].n);

        // 初始化rank_found为-1
        int rank_found = -1;

        // 在词汇表中查找双字母组合的排名
        rank_found = vocab.find_bpe_rank(left_token, right_token);
        // 如果未找到排名，则返回
        if (rank_found < 0) {
            return;
        }

        // 创建 llm_bigram_bpe 对象
        llm_bigram_bpe bigram;

        // 设置 bigram 对象的左右文本、文本内容、大小和排名
        bigram.left  = left;
        bigram.right = right;
        bigram.text  = left_token + right_token;
        bigram.size  = left_token.size() + right_token.size();
        bigram.rank  = rank_found;

        // 将 bigram 对象加入工作队列
        work_queue.push(bigram);
    }

    // 对文本进行 BPE 编码预处理，返回编码后的单词列表
    std::vector<std::string> bpe_gpt2_preprocess(const std::string & text) {
        // 创建存储 BPE 单词和编码后单词的向量
        std::vector<std::string> bpe_words;
        std::vector<std::string> bpe_encoded_words;

        // 初始化 token 字符串
        std::string token = "";
// 定义布尔变量，用于标记是否正在收集数字、字母、特殊字符、空白字符
bool collecting_numeric = false;
bool collecting_letter = false;
bool collecting_special = false;
bool collecting_whitespace_lookahead = false;
bool collecting = false;

// 创建存储 UTF-8 文本的向量，并预留空间
std::vector<std::string> text_utf;
text_utf.reserve(text.size());
bpe_words.reserve(text.size());
bpe_encoded_words.reserve(text.size());

// 将 UTF-8 文本转换为码点，并存储在 text_utf 中
auto cps = codepoints_from_utf8(text);
for (size_t i = 0; i < cps.size(); ++i)
    text_utf.emplace_back(codepoint_to_utf8(cps[i]));

// 遍历 text_utf 中的每个字符
for (int i = 0; i < (int)text_utf.size(); i++) {
    // 获取当前字符的 UTF-8 表示
    const std::string & utf_char = text_utf[i];
    // 初始化分割条件为 false，剩余字节数为 text_utf 的大小减去当前位置
    bool split_condition = false;
    int bytes_remain = text_utf.size() - i;
            // 前向和后向查找
            const std::string & utf_char_next = (i + 1 < (int)text_utf.size()) ? text_utf[i + 1] : ""; // 获取下一个字符
            const std::string & utf_char_next_next = (i + 2 < (int)text_utf.size()) ? text_utf[i + 2] : ""; // 获取下下一个字符

            // 处理缩略词
            if (!split_condition && bytes_remain >= 2) { // 如果不是分割条件并且剩余字节数大于等于2
                // 's|'t|'m|'d
                if (utf_char == "\'" && (utf_char_next == "s" || utf_char_next == "t" || utf_char_next == "m" || utf_char_next == "d")) { // 如果当前字符是'并且下一个字符是s、t、m、d中的一个
                    split_condition = true; // 设置分割条件为真
                }
                if (split_condition) { // 如果满足分割条件
                    if (token.size()) { // 如果token不为空
                        bpe_words.emplace_back(token); // 将之前的内容作为token推入bpe_words
                    }
                    token = utf_char + utf_char_next; // 将当前字符和下一个字符作为token
                    bpe_words.emplace_back(token); // 将token推入bpe_words
                    token = ""; // 重置token为空
                    i++; // i自增1
                    continue; // 继续下一次循环
                }
            }
            // 如果不是分割条件，并且剩余字节数大于等于3
            if (!split_condition && bytes_remain >= 3) {
                // 're|'ve|'ll
                // 如果当前字符是单引号，并且下一个字符是'r'、'v'、'l'，并且再下一个字符是'e'、'e'、'l'
                if (utf_char == "\'" && (
                    (utf_char_next == "r" && utf_char_next_next == "e") ||
                    (utf_char_next == "v" && utf_char_next_next == "e") ||
                    (utf_char_next == "l" && utf_char_next_next == "l"))
                    ) {
                    split_condition = true; // 设置分割条件为真
                }
                if (split_condition) {
                    // 当前标记 + 下一个标记可以被定义
                    if (token.size()) {
                        bpe_words.emplace_back(token); // 将之前的内容作为标记推入
                    }
                    token = utf_char + utf_char_next + utf_char_next_next; // 构造缩略词
                    bpe_words.emplace_back(token); // 缩略词
                    token = "";
                    i += 2;
                    continue;
# 如果不是在分割条件并且不在收集状态下
if (!split_condition && !collecting) {
    # 如果当前字符是字母，或者（当前token为空且当前字符是空格且下一个字符是字母）
    if (codepoint_type(utf_char) == CODEPOINT_TYPE_LETTER || (!token.size() && utf_char == " " && codepoint_type(utf_char_next) == CODEPOINT_TYPE_LETTER)) {
        # 开始收集字母
        collecting_letter = true;
        collecting = true;
    }
    # 如果当前字符是数字，或者（当前token为空且当前字符是空格且下一个字符是数字）
    else if (codepoint_type(utf_char) == CODEPOINT_TYPE_DIGIT || (!token.size() && utf_char == " " && codepoint_type(utf_char_next) == CODEPOINT_TYPE_DIGIT)) {
        # 开始收集数字
        collecting_numeric = true;
        collecting = true;
    }
    # 如果当前字符不是字母、数字、空格，或者
    else if (
        ((codepoint_type(utf_char) != CODEPOINT_TYPE_LETTER && codepoint_type(utf_char) != CODEPOINT_TYPE_DIGIT) && (codepoint_type(utf_char) != CODEPOINT_TYPE_WHITESPACE)) ||
        ) {
        # 开始收集特殊字符
        collecting_special = true;
        collecting = true;
    }
    # 如果当前字符是空格且下一个字符也是空格
    else if (codepoint_type(utf_char) == CODEPOINT_TYPE_WHITESPACE && codepoint_type(utf_char_next) == CODEPOINT_TYPE_WHITESPACE) {
        # 开始收集空格
        collecting_whitespace_lookahead = true;
# 如果正在收集字符，则设置收集标志为真
collecting = true;
# 否则，如果当前字符是空白字符，则设置分割条件为真
else if (codepoint_type(utf_char) == CODEPOINT_TYPE_WHITESPACE) {
    split_condition = true;
}
# 否则，如果没有分割条件且正在收集字符
else if (!split_condition && collecting) {
    # 如果正在收集字母且当前字符不是字母，则设置分割条件为真
    if (collecting_letter && codepoint_type(utf_char) != CODEPOINT_TYPE_LETTER) {
        split_condition = true;
    }
    # 否则，如果正在收集数字且当前字符不是数字，则设置分割条件为真
    else if (collecting_numeric && codepoint_type(utf_char) != CODEPOINT_TYPE_DIGIT) {
        split_condition = true;
    }
    # 否则，设置分割条件为真
    split_condition = true;
}
# 否则，如果正在收集空白字符的前瞻且下一个字符是字母或数字，则设置分割条件为真
else if (collecting_whitespace_lookahead && (codepoint_type(utf_char_next) == CODEPOINT_TYPE_LETTER || codepoint_type(utf_char_next) == CODEPOINT_TYPE_DIGIT)) {
    split_condition = true;
}
            // 如果下一个 UTF-8 字符为空，则满足分割条件，将当前字符加入 token
            if (utf_char_next == "") {
                split_condition = true; // 最终条件
                token += utf_char;
            }

            // 如果满足分割条件
            if (split_condition) {
                // 如果 token 非空，将其加入 bpe_words 中
                if (token.size()) {
                    bpe_words.emplace_back(token);
                }
                // 重置 token，并将各种收集标志位设为 false
                token = utf_char;
                collecting = false;
                collecting_letter = false;
                collecting_numeric = false;
                collecting_special = false;
                collecting_whitespace_lookahead = false;
            }
            // 如果不满足分割条件，将当前字符加入 token
            else {
                token += utf_char;
            }
// 遍历 bpe_words 中的每个单词
for (std::string & word : bpe_words) {
    // 初始化编码后的 token
    std::string encoded_token = "";
    // 遍历单词中的每个字符
    for (char & c : word) {
        // 将字符转换为 Unicode BPE 编码，并添加到编码后的 token 中
        encoded_token += bytes_to_unicode_bpe(c);
    }
    // 将编码后的 token 添加到 bpe_encoded_words 中
    bpe_encoded_words.emplace_back(encoded_token);
}

// 返回编码后的单词列表
return bpe_encoded_words;
}

// 声明一个 llama_vocab 类型的引用变量 vocab

// 声明两个 llm_symbol 类型的向量 symbols 和 symbols_final

// 声明一个 llm_bigram_bpe 类型的队列 work_queue
# 定义枚举类型 FRAGMENT_BUFFER_VARIANT_TYPE，包括 TOKEN 和 RAW_TEXT 两种类型
typedef enum FRAGMENT_BUFFER_VARIANT_TYPE{
    FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN,  # 表示 TOKEN 类型
    FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT  # 表示 RAW_TEXT 类型
} FRAGMENT_BUFFER_VARIANT_TYPE;

# 定义结构体 fragment_buffer_variant，包括两个构造函数
struct fragment_buffer_variant{
    # 构造函数，接受一个 llama_vocab::id 类型的参数 _token
    fragment_buffer_variant(llama_vocab::id _token)
    :
        type(FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN),  # 设置类型为 TOKEN
        token(_token),  # 设置 token 值为传入的参数
        raw_text(_dummy),  # 设置 raw_text 值为默认值
        offset(0),  # 设置 offset 值为 0
        length(0){}  # 设置 length 值为 0

    # 构造函数，接受一个 std::string 类型的参数 _raw_text 和两个 int64_t 类型的参数 _offset 和 _length
    fragment_buffer_variant(const std::string & _raw_text, int64_t _offset, int64_t _length)
    :
        type(FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT),  # 设置类型为 RAW_TEXT
        token((llama_vocab::id)-1),  # 设置 token 值为 -1
        raw_text(_raw_text),  # 设置 raw_text 值为传入的参数
        offset(_offset),  # 设置 offset 值为传入的参数
        length(_length){  # 设置 length 值为传入的参数
// 确保偏移量大于等于0
GGML_ASSERT( _offset >= 0 );
// 确保长度大于等于1
GGML_ASSERT( _length >= 1 );
// 确保偏移量加长度不超过原始文本的长度
GGML_ASSERT( offset + length <= raw_text.length() );

// 定义片段缓冲区变体类型
const FRAGMENT_BUFFER_VARIANT_TYPE type;
// 特殊词汇的标识符
const llama_vocab::id token;
// 用于占位的虚拟字符串
const std::string _dummy;
// 原始文本的引用
const std::string & raw_text;
// 片段的偏移量
const uint64_t offset;
// 片段的长度
const uint64_t length;
};

// 静态函数，用于对标记进行分割
static void tokenizer_st_partition(const llama_vocab & vocab, std::forward_list<fragment_buffer_variant> & buffer)
{
    // 对于每个特殊标记
    for (const auto & st: vocab.special_tokens_cache) {
        // 获取特殊标记的引用
        const auto & special_token = st.first;
        // 获取特殊 ID
        const auto & special_id    = st.second;

        // 遍历文本片段
        std::forward_list<fragment_buffer_variant>::iterator it = buffer.begin();
        while (it != buffer.end()) {
            auto & fragment = (*it);

            // 如果片段是文本（尚未处理）
            if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT) {
                auto * raw_text = &(fragment.raw_text);

                // 获取文本基础偏移量和长度
                auto raw_text_base_offset = fragment.offset;
                auto raw_text_base_length = fragment.length;

                // 循环遍历文本
                while (true) {
                    // 在该片段中查找给定特殊标记的第一个出现位置
                    // 仅传递偏移量参数限制了“搜索区域”，但匹配坐标仍然是相对于源完整的原始文本
                    auto match = raw_text->find(special_token, raw_text_base_offset);
// 如果没有找到特殊标记的匹配项，就停止处理该片段
if (match == std::string::npos) break;

// 检查匹配项是否在偏移量和长度范围内
if (match + special_token.length() > raw_text_base_offset + raw_text_base_length) break;

#ifdef PRETOKENIZERDEBUG
// 输出调试信息
fprintf(stderr, "FF: (%ld %ld %ld) '%s'\n", raw_text->length(), raw_text_base_offset, raw_text_base_length, raw_text->substr(raw_text_base_offset, raw_text_base_length).c_str());
#endif
// 计算匹配项在原始文本中的位置
auto source = std::distance(buffer.begin(), it);

// 如果匹配项在基本偏移之后
// 则说明它左侧有一些文本
if (match > raw_text_base_offset) {
    // 左侧文本
    const int64_t left_reminder_offset = raw_text_base_offset + 0;
    const int64_t left_reminder_length = match - raw_text_base_offset;
    // 在匹配项之前插入左侧文本
    buffer.emplace_after(it, (*raw_text), left_reminder_offset, left_reminder_length);
#ifdef PRETOKENIZERDEBUG
                        // 如果定义了预处理器宏 PRETOKENIZERDEBUG，则输出左侧剩余文本的偏移量、长度和内容
                        fprintf(stderr, "FL: (%ld %ld) '%s'\n", left_reminder_offset, left_reminder_length, raw_text->substr(left_reminder_offset, left_reminder_length).c_str());
#endif
                        // 移动迭代器到下一个位置
                        it++;
                    }

                    // 插入特殊标记
                    buffer.emplace_after(it, special_id);
                    // 移动迭代器到下一个位置
                    it++;

                    // 处理右侧剩余文本
                    if (match + special_token.length() < raw_text_base_offset + raw_text_base_length) {
                        const int64_t right_reminder_offset = match + special_token.length();
                        const int64_t right_reminder_length = raw_text_base_length - ((match - raw_text_base_offset) + special_token.length());
                        // 在迭代器指向的位置后插入右侧剩余文本
                        buffer.emplace_after(it, (*raw_text), right_reminder_offset, right_reminder_length);

#ifdef PRETOKENIZERDEBUG
                        // 如果定义了预处理器宏 PRETOKENIZERDEBUG，则输出右侧剩余文本的偏移量、长度和内容
                        fprintf(stderr, "FR: (%ld %ld) '%s'\n", right_reminder_offset, right_reminder_length, raw_text->substr(right_reminder_offset, right_reminder_length).c_str());
#endif
                        // 增加迭代器的值
                        it++;

                        // 如果来源为0，删除第一个元素
                        if (source == 0) {
                            buffer.erase_after(buffer.before_begin());
                        } else {
                            // 否则，删除指定位置的元素
                            buffer.erase_after(std::next(buffer.begin(), (source-1)));
                        }

                        // 重复上述步骤处理右侧的情况
                        raw_text_base_offset = right_reminder_offset;
                        raw_text_base_length = right_reminder_length;

#ifdef PRETOKENIZERDEBUG
                        // 输出调试信息
                        fprintf(stderr, "RR: (%ld %ld) '%s'\n", raw_text_base_offset, raw_text_base_length, raw_text->substr(raw_text_base_offset, raw_text_base_length).c_str());
#endif
                    } else {
                        // 如果来源为0，删除第一个元素
                        if (source == 0) {
                            buffer.erase_after(buffer.before_begin());
                        } else {
                            // 否则，删除指定位置的元素
                            buffer.erase_after(std::next(buffer.begin(), (source-1)));
// 定义一个静态函数 llama_tokenize_internal，接受一个 llama_vocab 对象和两个布尔值参数，返回一个 llama_vocab::id 类型的向量
static std::vector<llama_vocab::id> llama_tokenize_internal(const llama_vocab & vocab, std::string raw_text, bool bos, bool special) {
    // 定义一个名为 output 的向量，用于存储结果
    std::vector<llama_vocab::id> output;

    // OG tokenizer behavior:
    //
    // tokenizer.encode('', add_bos=True)  returns [1]
    // tokenizer.encode('', add_bos=False) returns []
    // 如果需要在开头添加特殊标记，并且 vocab 中有特殊标记的 id
    if (bos && vocab.special_bos_id != -1) {
        // 将特殊标记的 id 添加到 output 中
        output.push_back(vocab.special_bos_id);
    }

    // 如果原始文本为空，则直接返回输出
    if (raw_text.empty()) {
        return output;
    }

    // 创建一个前向列表，用于存储片段缓冲区的变体
    std::forward_list<fragment_buffer_variant> fragment_buffer;
    // 在片段缓冲区的前面插入一个元素，该元素包含原始文本的信息
    fragment_buffer.emplace_front( raw_text, 0, raw_text.length() );

    // 如果特殊情况为真，则使用特殊的分词器对词汇表进行分区
    if (special) tokenizer_st_partition( vocab, fragment_buffer );

    // 根据词汇表的类型进行不同的处理
    switch (vocab.type) {
        case LLAMA_VOCAB_TYPE_SPM:
            {
                // 遍历片段缓冲区中的每个片段
                for (const auto & fragment: fragment_buffer)
                {
                    // 如果片段类型为原始文本
                    if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT)
                    {
                        // 在不添加前导空格的情况下，我们无法获得与原始分词器相同的结果
// TODO: 可能可以通过修改 llm_tokenizer_x 以使用字符串偏移量来完全消除这个字符串复制
// 并像 pretokenizer 一样传递 'add space prefix' 作为布尔参数
//
// 如果特殊标志为真，则将原始文本设置为空字符串，否则为" "，再加上从片段中提取的文本
auto raw_text = (special ? "" : " ") + fragment.raw_text.substr(fragment.offset, fragment.length);

#ifdef PRETOKENIZERDEBUG
// 打印调试信息，输出原始文本的长度、偏移量、长度和内容
fprintf(stderr,"TT: (%ld %ld %ld) '%s'\n", raw_text.length(), fragment.offset, fragment.length, raw_text.c_str());
#endif
// 使用词汇表创建 llm_tokenizer_spm 对象
llm_tokenizer_spm tokenizer(vocab);
// 转义原始文本中的空白字符
llama_escape_whitespace(raw_text);
// 对原始文本进行分词，并将结果存储到输出容器中
tokenizer.tokenize(raw_text, output);
// 如果片段类型为 FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN
} else {
    // 将片段中的标记添加到输出容器中
    output.push_back(fragment.token);
}
// 遍历 fragment_buffer 中的每个 fragment
{
    for (const auto & fragment: fragment_buffer)
    {
        // 如果 fragment 的类型是原始文本
        if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_RAW_TEXT)
        {
            // 获取原始文本的子串
            auto raw_text = fragment.raw_text.substr(fragment.offset, fragment.length);

            // 如果定义了预处理调试宏，则输出原始文本信息
#ifdef PRETOKENIZERDEBUG
            fprintf(stderr,"TT: (%ld %ld %ld) '%s'\n", raw_text.length(), fragment.offset, fragment.length, raw_text.c_str());
#endif
            // 使用词汇表创建 BPE 分词器
            llm_tokenizer_bpe tokenizer(vocab);
            // 对原始文本进行分词处理，结果存储在 output 中
            tokenizer.tokenize(raw_text, output);
        }
        // 如果 fragment 的类型是标记
        else // if (fragment.type == FRAGMENT_BUFFER_VARIANT_TYPE_TOKEN)
        {
            // 将标记添加到输出中
            output.push_back(fragment.token);
        }
    }
} break;
```
// 返回output变量的值
    return output;
}

//
// grammar - internal
//

// 定义llama_partial_utf8结构体，用于存储部分生成的UTF-8序列
struct llama_partial_utf8 {
    uint32_t value;    // 存储到目前为止的位值（未移位）
    int      n_remain; // 剩余的字节数；-1表示无效序列
};

// 定义llama_grammar结构体，包含规则、堆栈和部分生成的UTF-8序列的缓冲区
struct llama_grammar {
    const std::vector<std::vector<llama_grammar_element>>   rules; // 规则集合
    std::vector<std::vector<const llama_grammar_element *>> stacks; // 堆栈集合

    llama_partial_utf8                                      partial_utf8; // 部分生成的UTF-8序列的缓冲区
};
// 定义了一个结构体，用于存储候选的语法解析结果
struct llama_grammar_candidate {
    size_t               index;  // 索引
    const uint32_t     * code_points;  // 指向存储编码点的指针
    llama_partial_utf8   partial_utf8;  // 存储部分 UTF-8 编码的结构体
};

// 解码一个可能以不完整序列结尾的 UTF-8 字符串。添加一个终止的 0 以便作为指针使用。如果遇到无效序列，则返回 `llama_partial_utf8.n_remain == -1`。
static std::pair<std::vector<uint32_t>, llama_partial_utf8> decode_utf8(
        const char         * src,  // 源字符串
        llama_partial_utf8   partial_start) {  // 部分 UTF-8 编码的起始位置
    static const int      lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 2, 2, 3, 4 };  // UTF-8 编码长度的查找表
    const char          * pos      = src;  // 指向当前位置的指针
    std::vector<uint32_t> code_points;  // 存储解码后的编码点
    uint32_t              value    = partial_start.value;  // 部分编码点的值
    int                   n_remain = partial_start.n_remain;  // 剩余字节数

    // 继续之前的解码，如果适用
    while (*pos != 0 && n_remain > 0) {  // 当前位置不为终止符且剩余字节数大于 0 时循环
// 读取下一个字节，并转换为无符号8位整数
uint8_t next_byte = static_cast<uint8_t>(*pos);
// 检查是否为无效的 UTF-8 序列，如果是则中止
if ((next_byte >> 6) != 2) {
    code_points.push_back(0);
    return std::make_pair(std::move(code_points), llama_partial_utf8{ 0, -1 });
}
// 将值左移6位，并加上下一个字节的低6位
value = (value << 6) + (next_byte & 0x3F);
// 移动到下一个位置
++pos;
// 剩余字节数减一
--n_remain;
}

// 如果存在部分开始且剩余字节数为0，则将值添加到 code_points 中
if (partial_start.n_remain > 0 && n_remain == 0) {
    code_points.push_back(value);
}

// 解码任何后续的 UTF-8 序列，可能以不完整的序列结束
while (*pos != 0) {
    // 读取下一个字节，并获取高4位
    uint8_t  first_byte = static_cast<uint8_t>(*pos);
    uint8_t  highbits   = first_byte >> 4;
    // 根据高4位查找剩余字节数
    n_remain   = lookup[highbits] - 1;
        if (n_remain < 0) {
            // 如果剩余字节数小于0，表示序列无效，清空code_points并返回空结果
            code_points.clear();
            code_points.push_back(0);
            return std::make_pair(std::move(code_points), llama_partial_utf8{ 0, n_remain });
        }

        uint8_t  mask       = (1 << (7 - n_remain)) - 1;
                 value      = first_byte & mask;
        ++pos;
        // 循环处理多字节序列
        while (*pos != 0 && n_remain > 0) {
            value = (value << 6) + (static_cast<uint8_t>(*pos) & 0x3F);
            ++pos;
            --n_remain;
        }
        // 如果剩余字节数为0，表示序列有效，将结果添加到code_points中
        if (n_remain == 0) {
            code_points.push_back(value);
        }
    }
```

// 将值0添加到code_points的末尾
code_points.push_back(0);

// 返回一个包含code_points和llama_partial_utf8的pair对象
return std::make_pair(std::move(code_points), llama_partial_utf8{ value, n_remain });
}

// 如果pos指向规则的定义结尾之一，则返回true
static bool llama_grammar_is_end_of_sequence(const llama_grammar_element * pos) {
    switch (pos->type) {
        case LLAMA_GRETYPE_END: return true;  // NOLINT
        case LLAMA_GRETYPE_ALT: return true;  // NOLINT
        default:                return false;
    }
}

// 如果chr满足pos处的字符范围（正常或反向范围），则返回true
// 断言pos指向一个字符范围元素
static std::pair<bool, const llama_grammar_element *> llama_grammar_match_char(
        const llama_grammar_element * pos,
        const uint32_t                chr) {
    # 声明并初始化一个布尔变量，用于表示是否找到匹配
    bool found            = false;
    # 声明并初始化一个布尔变量，用于表示是否为正字符类型
    bool is_positive_char = pos->type == LLAMA_GRETYPE_CHAR;

    # 断言，如果不是正字符类型，则为负字符类型
    GGML_ASSERT(is_positive_char || pos->type == LLAMA_GRETYPE_CHAR_NOT); // NOLINT

    # 循环，处理字符匹配规则
    do {
        # 如果下一个字符是一个字符范围的上限
        if (pos[1].type == LLAMA_GRETYPE_CHAR_RNG_UPPER) {
            # 包含范围，例如 [a-z]
            found = found || (pos->value <= chr && chr <= pos[1].value);
            pos += 2;
        } else {
            # 精确字符匹配，例如 [a] 或 "a"
            found = found || pos->value == chr;
            pos += 1;
        }
    } while (pos->type == LLAMA_GRETYPE_CHAR_ALT);

    # 返回一个布尔值对，表示是否找到匹配和处理后的位置
    return std::make_pair(found == is_positive_char, pos);
}
// 返回 true，如果给定部分 UTF-8 序列的某个延续可以满足位置 pos 处的字符范围（正常或反向范围）
// 断言 pos 指向一个字符范围元素
static bool llama_grammar_match_partial_char(
        const llama_grammar_element * pos,
        const llama_partial_utf8      partial_utf8) {

    // 判断是否为正向字符范围，如果不是则为反向字符范围
    bool is_positive_char = pos->type == LLAMA_GRETYPE_CHAR;
    GGML_ASSERT(is_positive_char || pos->type == LLAMA_GRETYPE_CHAR_NOT);

    // 获取部分 UTF-8 序列的值和剩余字节数
    uint32_t partial_value = partial_utf8.value;
    int      n_remain      = partial_utf8.n_remain;

    // 如果序列无效或者 7 位字符跨越 2 个字节（过长）
    if (n_remain < 0 || (n_remain == 1 && partial_value < 2)) {
        return false;
    }

    // 可能的代码点范围，该部分 UTF-8 序列可以完成的
    uint32_t low  = partial_value << (n_remain * 6);
    // 计算高位，将低位与剩余位数的掩码相或
    uint32_t high = low | ((1 << (n_remain * 6)) - 1);

    // 如果低位为0
    if (low == 0) {
        // 如果剩余位数为2，将低位设置为1左移11位
        if (n_remain == 2) {
            low = 1 << 11;
        } 
        // 如果剩余位数为3，将低位设置为1左移16位
        else if (n_remain == 3) {
            low = 1 << 16;
        }
    }

    // 循环
    do {
        // 如果类型为LLAMA_GRETYPE_CHAR_RNG_UPPER
        if (pos[1].type == LLAMA_GRETYPE_CHAR_RNG_UPPER) {
            // 包含范围，例如[a-z]
            if (pos->value <= high && low <= pos[1].value) {
                return is_positive_char;
            }
            pos += 2;
        } 
        // 否则
        else {
            // 精确字符匹配，例如[a]或"a"
            if (low <= pos->value && pos->value <= high) {
// 返回一个布尔值，表示当前字符是否为正字符
return is_positive_char;
// 如果当前字符不是正字符，则继续向后移动一个位置
pos += 1;
// 循环直到遇到字符范围的替代元素
} while (pos->type == LLAMA_GRETYPE_CHAR_ALT);

// 如果当前字符不是正字符，则返回否定字符
return !is_positive_char;
}

// 将语法推入栈转换为N个可能的栈，所有栈都以字符范围（终结元素）结尾
static void llama_grammar_advance_stack(
        const std::vector<std::vector<llama_grammar_element>>   & rules,
        const std::vector<const llama_grammar_element *>        & stack,
        std::vector<std::vector<const llama_grammar_element *>> & new_stacks) {

    // 如果栈为空，则将其添加到新栈中并返回
    if (stack.empty()) {
        new_stacks.emplace_back(stack);
        return;
    }

    // 获取栈顶元素
    const llama_grammar_element * pos = stack.back();

    // 根据栈顶元素的类型进行不同的处理
    switch (pos->type) {
        case LLAMA_GRETYPE_RULE_REF: {
            // 获取规则的 ID
            const size_t                  rule_id = static_cast<size_t>(pos->value);
            // 获取规则对应的元素
            const llama_grammar_element * subpos  = rules[rule_id].data();
            do {
                // 初始化一个不包含栈顶元素的新栈
                std::vector<const llama_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
                // 如果规则引用后面还有元素，则将其加入新栈
                if (!llama_grammar_is_end_of_sequence(pos + 1)) {
                    new_stack.push_back(pos + 1);
                }
                // 如果替代规则非空，则将其加入新栈
                if (!llama_grammar_is_end_of_sequence(subpos)) {
                    new_stack.push_back(subpos);
                }
                // 推进栈
                llama_grammar_advance_stack(rules, new_stack, new_stacks);
                while (!llama_grammar_is_end_of_sequence(subpos)) {
                    // 当子位置不是序列的结尾时，继续扫描到交替定义的结尾
                    subpos++;
                }
                if (subpos->type == LLAMA_GRETYPE_ALT) {
                    // 如果子位置的类型是交替定义，表示还有另一个交替定义需要处理，移动到下一个位置
                    subpos++;
                } else {
                    // 否则跳出循环
                    break;
                }
            } while (true);
            // 结束当前规则的处理
            break;
        }
        case LLAMA_GRETYPE_CHAR:
        case LLAMA_GRETYPE_CHAR_NOT:
            // 对于字符类型和非字符类型，将当前栈添加到新的栈中
            new_stacks.emplace_back(stack);
            break;
        default:
            // 对于交替的结束（LLAMA_GRETYPE_END, LLAMA_GRETYPE_ALT）或者字符范围的中间（LLAMA_GRETYPE_CHAR_ALT, LLAMA_GRETYPE_CHAR_RNG_UPPER），
            // 栈不应该留在这个位置
// 断言条件为假，如果条件为真则终止程序并输出错误信息
GGML_ASSERT(false);
}

// llama_grammar_accept函数接受一组可能的下推栈，这些栈需要位于字符范围上（参见`llama_grammar_advance_stack`），并在给定字符被接受时产生N个可能的栈
static std::vector<std::vector<const llama_grammar_element *>> llama_grammar_accept(
        const std::vector<std::vector<llama_grammar_element>>         & rules,  // 语法规则
        const std::vector<std::vector<const llama_grammar_element *>> & stacks, // 下推栈
        const uint32_t                                                  chr) {   // 字符

    std::vector<std::vector<const llama_grammar_element *>> new_stacks;  // 新的下推栈

    for (const auto & stack : stacks) {  // 遍历下推栈
        if (stack.empty()) {  // 如果栈为空则跳过
            continue;
        }
// 使用 llama_grammar_match_char 函数匹配栈顶元素和输入字符，返回匹配结果
auto match = llama_grammar_match_char(stack.back(), chr);
if (match.first) {
    // 如果匹配成功，获取匹配的元素
    const llama_grammar_element * pos = match.second;

    // 更新栈顶元素到下一个元素，如果有的话
    std::vector<const llama_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
    if (!llama_grammar_is_end_of_sequence(pos)) {
        new_stack.push_back(pos);
    }
    // 调用 llama_grammar_advance_stack 函数更新栈
    llama_grammar_advance_stack(rules, new_stack, new_stacks);
}

// 返回更新后的栈
return new_stacks;
}

// 定义 llama_grammar_reject_candidates 函数，接受规则和栈作为参数
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates(
        const std::vector<std::vector<llama_grammar_element>>         & rules,
        const std::vector<std::vector<const llama_grammar_element *>> & stacks,
// llama_grammar_reject_candidates_for_stack函数用于拒绝候选项，返回拒绝的候选项列表
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates_for_stack(
        // rules参数为规则的向量，stack参数为栈的向量，candidates参数为候选项的向量
        const std::vector<std::vector<llama_grammar_element>> & rules,
        const std::vector<const llama_grammar_element *>      & stack,
        const std::vector<llama_grammar_candidate>            & candidates) {

    // 创建一个空的拒绝候选项的向量
    std::vector<llama_grammar_candidate> rejects;

    // 如果栈为空
    if (stack.empty()) {
        // 遍历候选项，如果候选项的code_points不为0或者partial_utf8的n_remain不为0，则加入到拒绝的向量中
        for (const auto & tok : candidates) {
            if (*tok.code_points != 0 || tok.partial_utf8.n_remain != 0) {
                rejects.push_back(tok);
            }
        }
        // 返回拒绝的向量
        return rejects;
    }

    // 获取栈顶元素的指针
    const llama_grammar_element * stack_pos = stack.back();
// 创建一个存储 llama_grammar_candidate 对象的向量
std::vector<llama_grammar_candidate> next_candidates;
// 遍历候选对象
for (const auto & tok : candidates) {
    // 如果候选对象的 code_points 指向的值为 0
    if (*tok.code_points == 0) {
        // 如果候选对象的 partial_utf8 的剩余字符数不为 0，并且无法满足语法位置要求，则拒绝该候选对象
        if (tok.partial_utf8.n_remain != 0 &&
                !llama_grammar_match_partial_char(stack_pos, tok.partial_utf8)) {
            rejects.push_back(tok);
        }
    } else if (llama_grammar_match_char(stack_pos, *tok.code_points).first) {
        // 如果候选对象的 code_points 指向的值匹配语法要求，则将其添加到下一个候选对象的向量中
        next_candidates.push_back({ tok.index, tok.code_points + 1, tok.partial_utf8 });
    } else {
        // 否则拒绝该候选对象
        rejects.push_back(tok);
    }
}

// 获取匹配字符后的栈位置
const auto * stack_pos_after = llama_grammar_match_char(stack_pos, 0).second;

// 更新栈顶元素到下一个元素（如果有的话）
std::vector<const llama_grammar_element *> stack_after(stack.begin(), stack.end() - 1);
// 如果栈顶不是语法序列的结尾，则将栈顶位置添加到 stack_after 中
if (!llama_grammar_is_end_of_sequence(stack_pos_after)) {
    stack_after.push_back(stack_pos_after);
}

// 创建一个存储下一个可能栈的向量
std::vector<std::vector<const llama_grammar_element *>> next_stacks;

// 使用规则和更新后的栈，获取下一个可能的栈
llama_grammar_advance_stack(rules, stack_after, next_stacks);

// 获取下一个可能栈的拒绝候选项
auto next_rejects = llama_grammar_reject_candidates(rules, next_stacks, next_candidates);

// 将下一个可能栈的拒绝候选项添加到 rejects 中
for (const auto & tok : next_rejects) {
    rejects.push_back({ tok.index, tok.code_points - 1, tok.partial_utf8 });
}

// 返回 rejects
return rejects;
}

// 静态函数，用于获取拒绝候选项
static std::vector<llama_grammar_candidate> llama_grammar_reject_candidates(
        const std::vector<std::vector<llama_grammar_element>>         & rules,
        const std::vector<std::vector<const llama_grammar_element *>> & stacks,
        const std::vector<llama_grammar_candidate>                    & candidates) {
    // 断言栈不为空
    GGML_ASSERT(!stacks.empty()); // REVIEW
// 如果候选人列表为空，则返回一个空的llama_grammar_candidate向量
if (candidates.empty()) {
    return std::vector<llama_grammar_candidate>();
}

// 使用stacks的第一个元素和候选人列表来拒绝候选人
auto rejects = llama_grammar_reject_candidates_for_stack(rules, stacks.front(), candidates);

// 遍历stacks中的每个元素，拒绝候选人并更新rejects
for (size_t i = 1, size = stacks.size(); i < size; ++i) {
    rejects = llama_grammar_reject_candidates_for_stack(rules, stacks[i], rejects);
}
// 返回最终的rejects
return rejects;
}

//
// grammar - external
//

// 初始化llama_grammar结构
struct llama_grammar * llama_grammar_init(
            const llama_grammar_element ** rules,
                                 size_t    n_rules,
                                 size_t    start_rule_index) {
    // 声明指向 llama_grammar_element 类型的指针 pos
    const llama_grammar_element * pos;

    // 将规则定义复制到向量中
    std::vector<std::vector<llama_grammar_element>> vec_rules(n_rules);
    for (size_t i = 0; i < n_rules; i++) {
        for (pos = rules[i]; pos->type != LLAMA_GRETYPE_END; pos++) {
            vec_rules[i].push_back(*pos);
        }
        vec_rules[i].push_back({LLAMA_GRETYPE_END, 0});
    }

    // 循环遍历起始规则的替代项，构建初始堆栈
    std::vector<std::vector<const llama_grammar_element *>> stacks;
    pos = rules[start_rule_index];
    do {
        std::vector<const llama_grammar_element *> stack;
        if (!llama_grammar_is_end_of_sequence(pos)) {
            // 如果替代项不为空，将其添加到堆栈中
            stack.push_back(pos);
        }
// 调用函数 llama_grammar_advance_stack，传入参数 vec_rules, stack, stacks
llama_grammar_advance_stack(vec_rules, stack, stacks);
// 当位置 pos 不是序列的结尾时，执行循环
while (!llama_grammar_is_end_of_sequence(pos)) {
    // 扫描到交替定义的结尾
    pos++;
}
// 如果位置 pos 的类型是 LLAMA_GRETYPE_ALT
if (pos->type == LLAMA_GRETYPE_ALT) {
    // 存在该规则的另一个交替定义需要处理，移动到下一个位置
    pos++;
} else {
    // 否则跳出循环
    break;
}
// 无限循环，直到条件不满足跳出
} while (true);

// 返回一个新的 llama_grammar 对象，使用移动语义转移 vec_rules, stacks 的所有权
return new llama_grammar{ std::move(vec_rules), std::move(stacks), {} };
}

// 释放 llama_grammar 结构体指针所指向的内存
void llama_grammar_free(struct llama_grammar * grammar) {
    delete grammar;
}
// 复制 llama_grammar 结构体的内容并返回一个新的 llama_grammar 指针
struct llama_grammar * llama_grammar_copy(const struct llama_grammar * grammar) {
    // 创建一个新的 llama_grammar 结构体，并使用原始 grammar 的规则、堆栈和部分 UTF-8 标志进行初始化
    llama_grammar * result = new llama_grammar{ grammar->rules, grammar->stacks, grammar->partial_utf8 };

    // 将堆栈中的元素重定向指向新的规则
    for (size_t is = 0; is < result->stacks.size(); is++) {
        for (size_t ie = 0; ie < result->stacks[is].size(); ie++) {
            for (size_t ir0 = 0; ir0 < grammar->rules.size(); ir0++) {
                for (size_t ir1 = 0; ir1 < grammar->rules[ir0].size(); ir1++) {
                    // 如果原始堆栈中的元素指向原始规则，则将新堆栈中的元素指向新规则
                    if (grammar->stacks[is][ie] == &grammar->rules[ir0][ir1]) {
                         result->stacks[is][ie]  =  &result->rules[ir0][ir1];
                    }
                }
            }
        }
    }

    // 返回复制后的 llama_grammar 结构体指针
    return result;
}
// 设置随机数种子
void llama_set_rng_seed(struct llama_context * ctx, uint32_t seed) {
    // 如果种子值为默认值，则使用当前时间作为种子
    if (seed == LLAMA_DEFAULT_SEED) {
        seed = time(NULL);
    }
    // 使用种子值设置随机数生成器的种子
    ctx->rng.seed(seed);
}

// 对候选项进行 softmax 抽样
void llama_sample_softmax(struct llama_context * ctx, llama_token_data_array * candidates) {
    // 断言候选项数组的大小大于 0
    GGML_ASSERT(candidates->size > 0);

    // 记录开始抽样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 对 logit 按降序排序
    if (!candidates->sorted) {
        std::sort(candidates->data, candidates->data + candidates->size, [](const llama_token_data & a, const llama_token_data & b) {
            return a.logit > b.logit;
        });
    }
    // 将候选项按照概率排序
    candidates->sorted = true;

    // 计算最大概率值
    float max_l = candidates->data[0].logit;
    // 计算概率的累加和
    float cum_sum = 0.0f;
    for (size_t i = 0; i < candidates->size; ++i) {
        // 计算概率
        float p = expf(candidates->data[i].logit - max_l);
        candidates->data[i].p = p;
        // 累加概率
        cum_sum += p;
    }
    // 归一化概率
    for (size_t i = 0; i < candidates->size; ++i) {
        candidates->data[i].p /= cum_sum;
    }

    // 更新上下文中的采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 从候选项中采样出前 k 个，并保留至少 min_keep 个
void llama_sample_top_k(struct llama_context * ctx, llama_token_data_array * candidates, int k, size_t min_keep) {
    // 获取当前时间的微秒数
    const int64_t t_start_sample_us = ggml_time_us();

    // 确保 k 的值不小于 min_keep
    k = std::max(k, (int) min_keep);
    // 确保 k 的值不大于 candidates->size
    k = std::min(k, (int) candidates->size);

    // 将分数按降序排序
    if (!candidates->sorted) {
        // 定义比较函数，按照 logit 字段进行比较
        auto comp = [](const llama_token_data & a, const llama_token_data & b) {
            return a.logit > b.logit;
        };
        // 如果 k 等于 candidates->size，则对整个数组进行排序
        if (k == (int) candidates->size) {
            std::sort(candidates->data, candidates->data + candidates->size, comp);
        } else {
            // 否则对前 k 个元素进行部分排序
            std::partial_sort(candidates->data, candidates->data + k, candidates->data + candidates->size, comp);
        }
        candidates->sorted = true;
    }
    // 更新 candidates->size 的值为 k
    candidates->size = k;

    // 如果上下文存在，则执行以下操作
    if (ctx) {
// 更新采样时间
ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

// 从候选项中按照概率进行采样
void llama_sample_top_p(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
    // 如果概率大于等于1，直接返回
    if (p >= 1.0f) {
        return;
    }

    // 对候选项进行 softmax 采样
    llama_sample_softmax(ctx, candidates);

    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 计算累积概率
    float cum_sum = 0.0f;
    size_t last_idx = candidates->size;

    // 遍历候选项，累加概率
    for (size_t i = 0; i < candidates->size; ++i) {
        cum_sum += candidates->data[i].p;
// 检查累积和是否至少为p，或者我们至少保留了min_keep个标记
// 我们将最后的索引设置为i+1，表示当前迭代应该包含在集合中
if (cum_sum >= p && i + 1 >= min_keep) {
    last_idx = i + 1;
    break;
}

// 调整输出向量的大小，仅保留前p个标记
candidates->size = last_idx;

// 如果上下文存在，则更新采样时间
if (ctx) {
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}
}

void llama_sample_min_p(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
    // 如果p小于等于0.0或者候选集为空，则直接返回
    if (p <= 0.0f || !candidates->size) {
        return;
    }
    // 调用 llama_sample_softmax 函数，对候选项进行 softmax 处理
    llama_sample_softmax(ctx, candidates);

    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 通过第一个候选项的概率值来缩放，以便进行比较
    float scale = candidates->data[0].p; // scale by max prob
    // 初始化索引 i 为 1，因为第一个 token 总是匹配的
    size_t i = 1; // first token always matches

    // 遍历候选项，找到概率值小于阈值的 token，并且保留至少 min_keep 个 token
    for (; i < candidates->size; ++i) {
        if (candidates->data[i].p < p * scale && i >= min_keep) {
            break; // prob too small
        }
    }

    // 调整输出向量的大小，仅保留匹配的 token
    candidates->size = i;

    // 如果上下文对象存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
// 释放样本尾部内存，参数包括上下文指针、候选数据数组、z 值和最小保留大小
void llama_sample_tail_free(struct llama_context * ctx, llama_token_data_array * candidates, float z, size_t min_keep) {
    // 如果 z 值大于等于 1.0 或者候选数据数组大小小于等于 2，则直接返回
    if (z >= 1.0f || candidates->size <= 2) {
        return;
    }

    // 对候选数据数组进行 softmax 处理
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
}
// 计算二阶导数的绝对值
for (size_t i = 0; i < second_derivatives.size(); ++i) {
    second_derivatives[i] = std::abs(second_derivatives[i]);
}

// 对二阶导数进行归一化处理
{
    // 计算二阶导数的总和
    const float second_derivatives_sum = std::accumulate(second_derivatives.begin(), second_derivatives.end(), 0.0f);

    // 如果二阶导数的总和大于1e-6f，则对每个值进行归一化处理
    if (second_derivatives_sum > 1e-6f) {
        for (float & value : second_derivatives) {
            value /= second_derivatives_sum;
        }
    } 
    // 如果二阶导数的总和小于等于1e-6f，则将每个值设置为1.0f除以二阶导数的大小
    else {
        for (float & value : second_derivatives) {
            value = 1.0f / second_derivatives.size();
        }
    }
}
# 初始化累积和为0
float cum_sum = 0.0f;
# 初始化最后一个索引为候选项的大小
size_t last_idx = candidates->size;
# 遍历二阶导数的大小
for (size_t i = 0; i < second_derivatives.size(); ++i) {
    # 累积和加上当前二阶导数的值
    cum_sum += second_derivatives[i];

    # 检查累积和是否大于z，或者是否至少保留了min_keep个标记
    if (cum_sum > z && i >= min_keep) {
        # 更新最后一个索引为当前索引i
        last_idx = i;
        # 跳出循环
        break;
    }
}

# 调整输出向量的大小，仅保留尾部位置以上的标记
candidates->size = last_idx;

# 如果上下文存在，则更新时间采样
if (ctx) {
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}
// 使用典型采样算法从候选项中选择一个token
void llama_sample_typical(struct llama_context * ctx, llama_token_data_array * candidates, float p, size_t min_keep) {
    // 参考实现：https://github.com/huggingface/transformers/compare/main...cimeister:typical-sampling:typical-pr
    // 如果概率大于等于1，直接返回
    if (p >= 1.0f) {
        return;
    }

    // 计算logits的softmax并计算熵
    llama_sample_softmax(nullptr, candidates);

    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 计算熵
    float entropy = 0.0f;
    for (size_t i = 0; i < candidates->size; ++i) {
        entropy += -candidates->data[i].p * logf(candidates->data[i].p);
    }

    // 计算负对数概率和熵之间的绝对差值
    std::vector<float> shifted_scores;
    // 遍历候选项数组，计算每个候选项的偏移得分，并将其加入到偏移得分数组中
    for (size_t i = 0; i < candidates->size; ++i) {
        float shifted_score = fabsf(-logf(candidates->data[i].p) - entropy);
        shifted_scores.push_back(shifted_score);
    }

    // 根据偏移得分和对应的索引对候选项进行排序
    std::vector<size_t> indices(candidates->size);
    // 生成索引序列
    std::iota(indices.begin(), indices.end(), 0);

    // 使用偏移得分对索引进行排序
    std::sort(indices.begin(), indices.end(), [&](size_t a, size_t b) {
        return shifted_scores[a] < shifted_scores[b];
    });

    // 计算累积概率
    float cum_sum = 0.0f;
    size_t last_idx = indices.size();

    // 遍历排序后的索引数组，计算累积概率
    for (size_t i = 0; i < indices.size(); ++i) {
        size_t idx = indices[i];
        cum_sum += candidates->data[idx].p;
    }
    // 检查累积和是否大于典型值，或者是否至少保留了 min_keep 个标记
    if (cum_sum > p && i >= min_keep - 1) {
        // 如果满足条件，记录最后一个索引并跳出循环
        last_idx = i + 1;
        break;
    }
}

// 调整输出向量的大小，仅保留局部典型的标记
std::vector<llama_token_data> new_candidates;
for (size_t i = 0; i < last_idx; ++i) {
    size_t idx = indices[i];
    new_candidates.push_back(candidates->data[idx]);
}

// 用 new_candidates 数据替换 candidates 中的数据
std::copy(new_candidates.begin(), new_candidates.end(), candidates->data);
candidates->size = new_candidates.size();

// 如果存在上下文，则执行以下操作
if (ctx) {
// 更新采样时间
ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

// 对候选数据进行采样
void llama_sample_temp(struct llama_context * ctx, llama_token_data_array * candidates_p, float temp) {
    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 对候选数据进行温度调整
    for (size_t i = 0; i < candidates_p->size; ++i) {
        candidates_p->data[i].logit /= temp;
    }

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

// 对候选数据进行温度采样
void llama_sample_temperature(struct llama_context * ctx, llama_token_data_array * candidates_p, float temp) {
    // 调用采样函数
    llama_sample_temp(ctx, candidates_p, temp);
}
// 计算重复惩罚
void llama_sample_repetition_penalties(
            struct llama_context * ctx,  // LLAMA 上下文结构体指针
          llama_token_data_array * candidates,  // 候选 token 数据数组指针
               const llama_token * last_tokens,  // 最后的 token 数组指针
                          size_t   penalty_last_n,  // 最后的 n 个 token 的惩罚值
                           float   penalty_repeat,  // 重复惩罚值
                           float   penalty_freq,  // 频率惩罚值
                           float   penalty_present) {  // 存在惩罚值
    // 如果最后的 n 个 token 的惩罚值为 0，或者重复惩罚、频率惩罚、存在惩罚都为 0，则直接返回
    if (penalty_last_n == 0 || (penalty_repeat == 1.0f && penalty_freq == 0.0f && penalty_present == 0.0f)) {
        return;
    }

    // 获取当前时间的微秒数
    const int64_t t_start_sample_us = ggml_time_us();

    // 创建一个频率映射，用于计算 last_tokens 中每个 token 的出现次数
    std::unordered_map<llama_token, int> token_count;
    for (size_t i = 0; i < penalty_last_n; ++i) {
        token_count[last_tokens[i]]++;
    }
    // 对候选项应用频率和存在性惩罚
    for (size_t i = 0; i < candidates->size; ++i) {
        // 查找候选项在token_count中的出现次数
        const auto token_iter = token_count.find(candidates->data[i].id);
        // 如果候选项在token_count中不存在，则跳过
        if (token_iter == token_count.end()) {
            continue;
        }

        // 获取候选项在token_count中的出现次数
        const int count = token_iter->second;

        // 如果候选项的logit值小于等于0，则乘以惩罚值，否则除以惩罚值
        if (candidates->data[i].logit <= 0) {
            candidates->data[i].logit *= penalty_repeat;
        } else {
            candidates->data[i].logit /= penalty_repeat;
        }

        // 减去频率惩罚和存在性惩罚
        candidates->data[i].logit -= float(count) * penalty_freq + float(count > 0) * penalty_present;
    }
    // 设置候选项排序标志为假
    candidates->sorted = false;

    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

void llama_sample_grammar(struct llama_context * ctx, llama_token_data_array * candidates, const struct llama_grammar * grammar) {
    // 断言上下文存在
    GGML_ASSERT(ctx);
    // 获取采样开始时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 初始化是否允许结束符的标志为假
    bool allow_eos = false;
    // 遍历语法中的堆栈，如果为空则允许结束符
    for (const auto & stack : grammar->stacks) {
        if (stack.empty()) {
            allow_eos = true;
            break;
        }
    }

    // 获取结束符
    const llama_token eos = llama_token_eos(&ctx->model);
```

// 声明两个存储候选项解码结果的容器
std::vector<std::pair<std::vector<uint32_t>, llama_partial_utf8>> candidates_decoded;
std::vector<llama_grammar_candidate> candidates_grammar;

// 遍历候选项数组
for (size_t i = 0; i < candidates->size; ++i) {
    // 获取候选项的标识符
    const llama_token id = candidates->data[i].id;
    // 将标识符转换为字符串
    const std::string piece = llama_token_to_piece(ctx, id);
    // 如果标识符为结束符并且不允许结束符，则将该候选项的logit设为负无穷
    if (id == eos) {
        if (!allow_eos) {
            candidates->data[i].logit = -INFINITY;
        }
    } 
    // 如果字符串为空或者第一个字符为0，则将该候选项的logit设为负无穷
    else if (piece.empty() || piece[0] == 0) {
        candidates->data[i].logit = -INFINITY;
    } 
    // 否则，对字符串进行UTF-8解码，并将解码结果存入相应的容器中
    else {
        candidates_decoded.push_back(decode_utf8(piece.c_str(), grammar->partial_utf8));
        candidates_grammar.push_back({ i, candidates_decoded.back().first.data(), candidates_decoded.back().second });
    }
}

// 使用语法规则和候选项的解码结果进行语法拒绝操作
const auto rejects = llama_grammar_reject_candidates(grammar->rules, grammar->stacks, candidates_grammar);
    // 对于拒绝的候选项，将其logit值设为负无穷
    for (const auto & reject : rejects) {
        candidates->data[reject.index].logit = -INFINITY;
    }

    // 更新上下文中的采样时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

// 计算数组的log_softmax
static void llama_log_softmax(float * array, size_t size) {
    // 找到数组中的最大值
    float max_l = *std::max_element(array, array + size);
    float sum = 0.f;
    // 计算softmax值
    for (size_t i = 0; i < size; ++i) {
        float p = expf(array[i] - max_l);
        sum += p;
        array[i] = p;
    }

    // 计算log_softmax值
    for (size_t i = 0; i < size; ++i) {
        array[i] = logf(array[i] / sum);
    }
}
// 释放指导上下文中的样本分类器
void llama_sample_classifier_free_guidance(
          struct llama_context * ctx,  // 指向 llama_context 结构的指针
        llama_token_data_array * candidates,  // 指向候选数据数组的指针
          struct llama_context * guidance_ctx,  // 指向指导上下文的指针
                         float   scale) {  // 缩放因子
    int64_t t_start_sample_us = ggml_time_us();  // 获取当前时间的微秒数

    GGML_ASSERT(ctx);  // 断言 ctx 不为空

    auto n_vocab = llama_n_vocab(llama_get_model(ctx));  // 获取模型的词汇量

    GGML_ASSERT(n_vocab == (int)candidates->size);  // 断言词汇量与候选数据数组大小相等
    GGML_ASSERT(!candidates->sorted);  // 断言候选数据数组未排序

    std::vector<float> logits_base;  // 创建存储浮点数的向量
    logits_base.reserve(candidates->size);  // 预留候选数据数组大小的空间
    for (size_t i = 0; i < candidates->size; ++i) {  // 遍历候选数据数组
        logits_base.push_back(candidates->data[i].logit);  // 将候选数据的 logit 添加到向量中
    }
    # 对基础logits进行log_softmax操作
    llama_log_softmax(logits_base.data(), candidates->size);

    # 获取guidance上下文的logits并进行log_softmax操作
    float* logits_guidance = llama_get_logits(guidance_ctx);
    llama_log_softmax(logits_guidance, n_vocab);

    # 遍历词汇表，计算每个词的logit_guidance和logit_base，然后根据公式计算新的logit并存储在candidates中
    for (int i = 0; i < n_vocab; ++i) {
        float logit_guidance = logits_guidance[i];
        float logit_base = logits_base[i];
        candidates->data[i].logit = scale * (logit_base - logit_guidance) + logit_guidance;
    }

    # 如果上下文存在，则更新上下文的采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
}

# 从mirostat中采样一个token
llama_token llama_sample_token_mirostat(struct llama_context * ctx, llama_token_data_array * candidates, float tau, float eta, int m, float * mu) {
    # 断言上下文存在
    GGML_ASSERT(ctx);

    # 获取模型的词汇表大小
    auto N = float(llama_n_vocab(llama_get_model(ctx)));
    // 声明并初始化 t_start_sample_us 变量，用于存储当前时间的微秒数
    int64_t t_start_sample_us;
    t_start_sample_us = ggml_time_us();

    // 调用 llama_sample_softmax 函数，对 candidates 进行 softmax 操作
    llama_sample_softmax(nullptr, candidates);

    // 用最可能的 m 个 tokens 估计 s_hat
    float s_hat = 0.0;
    float sum_ti_bi = 0.0;
    float sum_ti_sq = 0.0;
    for (size_t i = 0; i < size_t(m - 1) && i < candidates->size - 1; ++i) {
        // 计算 t_i 和 b_i
        float t_i = logf(float(i + 2) / float(i + 1));
        float b_i = logf(candidates->data[i].p / candidates->data[i + 1].p);
        // 更新 sum_ti_bi 和 sum_ti_sq
        sum_ti_bi += t_i * b_i;
        sum_ti_sq += t_i * t_i;
    }
    // 计算 s_hat
    s_hat = sum_ti_bi / sum_ti_sq;

    // 根据估计的 s_hat 和目标惊讶值计算 k
    float epsilon_hat = s_hat - 1;
    float k = powf((epsilon_hat * powf(2, *mu)) / (1 - powf(N, -epsilon_hat)), 1 / s_hat);
    // 使用 top-k 抽样方法从候选集中抽取下一个单词 X
    llama_sample_top_k(nullptr, candidates, int(k), 1);
    // 如果上下文存在，则更新抽样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 从候选集中抽取一个 token X
    llama_token X = llama_sample_token(ctx, candidates);
    // 重置抽样时间
    t_start_sample_us = ggml_time_us();

    // 计算误差，即观察到的惊喜值与目标惊喜值之间的差异
    // 找到 X 在候选集中的索引
    size_t X_idx = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return candidate.id == X;
    }));
    // 计算观察到的惊喜值
    float observed_surprise = -log2f(candidates->data[X_idx].p);
    // 计算误差 e
    float e = observed_surprise - tau;

    // 使用学习率和误差更新 mu
    *mu = *mu - eta * e;

    // 如果上下文存在，则继续执行
    if (ctx) {
    // 记录开始采样的时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    // 返回采样结果
    return X;
}

// 从候选词中进行采样，返回采样结果的 token
llama_token llama_sample_token_mirostat_v2(struct llama_context * ctx, llama_token_data_array * candidates, float tau, float eta, float * mu) {
    int64_t t_start_sample_us;
    // 记录开始采样的时间
    t_start_sample_us = ggml_time_us();

    // 对候选词进行 softmax 采样
    llama_sample_softmax(ctx, candidates);

    // 截断概率大于 mu 的候选词
    candidates->size = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return -log2f(candidate.p) > *mu;
    }));

    // 如果没有符合条件的候选词，将候选词数量设为1
    if (candidates->size == 0) {
        candidates->size = 1;
    }
    // 如果上下文对象存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }

    // 对剩余单词的概率进行归一化处理
    llama_sample_softmax(ctx, candidates);

    // 从剩余单词中随机抽样出下一个单词 X
    llama_token X = llama_sample_token(ctx, candidates);
    t_start_sample_us = ggml_time_us();

    // 计算观察到的惊喜和目标惊喜值之间的差异作为误差
    size_t X_idx = std::distance(candidates->data, std::find_if(candidates->data, candidates->data + candidates->size, [&](const llama_token_data & candidate) {
        return candidate.id == X;
    }));
    float observed_surprise = -log2f(candidates->data[X_idx].p);
    float e = observed_surprise - tau;

    // 使用学习率和误差更新 mu
    *mu = *mu - eta * e;
    // 如果上下文存在，则更新采样时间
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    }
    // 返回最大元素
    return X;
}

// 采用贪婪算法对候选项进行抽样
llama_token llama_sample_token_greedy(struct llama_context * ctx, llama_token_data_array * candidates) {
    // 记录开始采样的时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 查找最大元素
    auto * max_iter = std::max_element(candidates->data, candidates->data + candidates->size, [](const llama_token_data & a, const llama_token_data & b) {
        return a.logit < b.logit;
    });

    // 获取最大元素的标识
    llama_token result = max_iter->id;
    // 如果上下文存在，则更新采样时间并增加采样次数
    if (ctx) {
        ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
        ctx->n_sample++;
    }
    // 返回结果
    return result;
}

// 从候选项中抽样一个 token
llama_token llama_sample_token(struct llama_context * ctx, llama_token_data_array * candidates) {
    // 断言上下文不为空
    GGML_ASSERT(ctx);

    // 记录抽样开始时间
    const int64_t t_start_sample_us = ggml_time_us();
    // 对候选项进行 softmax 处理
    llama_sample_softmax(nullptr, candidates);

    // 创建概率数组
    std::vector<float> probs;
    probs.reserve(candidates->size);
    // 遍历候选项，将概率值加入数组
    for (size_t i = 0; i < candidates->size; ++i) {
        probs.push_back(candidates->data[i].p);
    }

    // 创建离散分布对象
    std::discrete_distribution<> dist(probs.begin(), probs.end());
    // 获取上下文中的随机数生成器
    auto & rng = ctx->rng;
    // 从分布中抽样一个索引
    int idx = dist(rng);

    // 获取抽样到的 token
    llama_token result = candidates->data[idx].id;
    // 更新采样时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    // 增加样本计数
    ctx->n_sample++;
    // 返回结果
    return result;
}

void llama_grammar_accept_token(struct llama_context * ctx, struct llama_grammar * grammar, llama_token token) {
    // 记录开始采样时间
    const int64_t t_start_sample_us = ggml_time_us();

    // 如果 token 是句子结束符
    if (token == llama_token_eos(&ctx->model)) {
        // 遍历语法中的栈
        for (const auto & stack : grammar->stacks) {
            // 如果栈为空，返回
            if (stack.empty()) {
                return;
            }
        }
        // 断言错误
        GGML_ASSERT(false);
    }

    // 将 token 转换为字符串
    const std::string piece = llama_token_to_piece(ctx, token);
    // 注意解码后的字符串中是否包含终止的 0
    const auto   decoded     = decode_utf8(piece.c_str(), grammar->partial_utf8);
    // 获取解码后的码点
    const auto & code_points = decoded.first;
    // 遍历码点，处理除最后一个之外的所有码点
    for (auto it = code_points.begin(), end = code_points.end() - 1; it != end; ++it) {
        // 使用码点更新语法栈
        grammar->stacks = llama_grammar_accept(grammar->rules, grammar->stacks, *it);
    }
    // 更新部分 UTF-8 编码
    grammar->partial_utf8 = decoded.second;
    // 断言语法栈不为空
    GGML_ASSERT(!grammar->stacks.empty());

    // 更新采样时间
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
}

//
// Beam search
//

// 定义 Llama Beam 结构
struct llama_beam {
    // 存储令牌的向量
    std::vector<llama_token> tokens;
    // 累积梁概率（相对于所有梁进行重新归一化）
    float p;
    // 初始化是否到达梁的末尾为假。回调函数将其设置为真。
    bool eob;
// 按概率对光束进行排序。在出现并列的情况下，优先选择在 eob 处的光束。
bool operator<(const llama_beam & rhs) const {
    return std::make_pair(p, eob) < std::make_pair(rhs.p, rhs.eob);
}

// 移除前 n 个标记并丢弃它们。
void shift_tokens(const size_t n) {
    if (n) {
        std::copy(tokens.begin() + n, tokens.end(), tokens.begin());
        tokens.resize(tokens.size() - n);
    }
}

// 返回光束的视图。
llama_beam_view view() const { return {tokens.data(), tokens.size(), p, eob}; }

// 用于计算与 logit 相关的信息的结构体。
struct llama_logit_info {
    const float * const logits;  // logit 数组的指针
    const int n_vocab;  // 词汇表的大小
    const float max_l;  // 最大的 logit 值
    const float normalizer;  // 规范化因子
}
    // 定义一个结构体，用于计算指数和
    struct sum_exp {
        float max_l; // 最大的指数
        // 重载 () 运算符，用于计算指数和
        float operator()(float sum, float l) const { return sum + std::exp(l - max_l); }
    };
    // 构造函数，初始化成员变量
    llama_logit_info(llama_context * ctx)
      : logits(llama_get_logits(ctx)) // 获取logits
      , n_vocab(llama_n_vocab(llama_get_model(ctx))) // 获取词汇表大小
      , max_l(*std::max_element(logits, logits + n_vocab)) // 计算logits中的最大值
      , normalizer(1.0f / std::accumulate(logits, logits + n_vocab, 0.0f, sum_exp{max_l})) // 计算归一化因子
      { }
    // 获取token的数据
    llama_token_data get_token_data(const llama_token token_id) const {
        constexpr auto p = std::numeric_limits<float>::quiet_NaN();  // 永远不会被使用
        return {token_id, logits[token_id], p}; // 返回token的数据
    }
    // 返回logit值最高的前k个token_data
    std::vector<llama_token_data> top_k(size_t k) {
        std::vector<llama_token_data> min_heap;  // 通过logit值构建的最小堆
        const llama_token k_min = std::min(static_cast<llama_token>(k), n_vocab); // 取k和词汇表大小的最小值
        min_heap.reserve(k_min); // 预留空间
        for (llama_token token_id = 0 ; token_id < k_min ; ++token_id) { // 遍历前k个token
        // 将 token_id 对应的 token 数据加入最小堆中
        min_heap.push_back(get_token_data(token_id));
        // 定义 lambda 表达式 comp 用于比较 llama_token_data 结构体的 logit 字段
        auto comp = [](const llama_token_data & a, const llama_token_data & b) { return a.logit > b.logit; };
        // 使用 comp 函数对象创建最小堆
        std::make_heap(min_heap.begin(), min_heap.end(), comp);
        // 遍历 token_id，更新最小堆中的数据
        for (llama_token token_id = k_min ; token_id < n_vocab ; ++token_id) {
            // 如果最小堆的堆顶元素的 logit 小于当前 token_id 对应的 logit，则更新堆顶元素
            if (min_heap.front().logit < logits[token_id]) {
                // 弹出堆顶元素
                std::pop_heap(min_heap.begin(), min_heap.end(), comp);
                // 更新堆顶元素的 id 和 logit
                min_heap.back().id = token_id;
                min_heap.back().logit = logits[token_id];
                // 将更新后的元素重新加入最小堆
                std::push_heap(min_heap.begin(), min_heap.end(), comp);
            }
        }
        // 返回更新后的最小堆
        return min_heap;
    }
    // 根据 logit 计算概率
    float probability_from_logit(float logit) const {
        return normalizer * std::exp(logit - max_l);
    }
};

// 定义结构体 llama_beam_search_data
struct llama_beam_search_data {
    // 声明一个指向 llama_context 类型的指针 ctx
    llama_context * ctx;
    // 声明一个 size_t 类型的变量 n_beams
    size_t n_beams;
    // 声明一个 int 类型的变量 n_past
    int n_past;
    // 声明一个 int 类型的变量 n_predict
    int n_predict;
    // 声明一个存储 llama_beam 对象的向量 beams
    std::vector<llama_beam> beams;
    // 声明一个存储 llama_beam 对象的向量 next_beams
    std::vector<llama_beam> next_beams;

    // 在每次循环迭代时重新计算的变量，表示公共前缀的长度
    size_t common_prefix_length;

    // 用于在回调函数中传递和接收 beams 状态的信息
    std::vector<llama_beam_view> beam_views;

    // llama_beam_search_data 的构造函数，初始化成员变量
    llama_beam_search_data(llama_context * ctx, size_t n_beams, int n_past, int n_predict)
      : ctx(ctx)
      , n_beams(n_beams)
      , n_past(n_past)
      , n_predict(n_predict)
      , beam_views(n_beams) {
        // 预留空间以容纳 n_beams 个元素
        beams.reserve(n_beams);
    // 为下一个beam预留空间，大小为n_beams
    next_beams.reserve(n_beams);
    }

    // 将beams折叠成由索引指定的单个beam
    void collapse_beams(const size_t beam_idx) {
        if (0u < beam_idx) {
            std::swap(beams[0], beams[beam_idx]);
        }
        beams.resize(1);
    }

    // 使用最小堆来高效地收集前k个元素（k=n_beams）
    // 下面的重复模式反映了堆的两个阶段：
    //  * 收集元素直到向量已满，然后在其上调用std::make_heap()
    //  * 如果堆已满并且找到应包含的新元素，则将最小元素弹出到back()，用新元素替换它，然后将其推入堆中。
    void fill_next_beams_by_top_probabilities(llama_beam & beam) {
        // 最小堆使用大于比较器
        const auto comp = [](const llama_beam & a, const llama_beam & b) { return a.p > b.p; };
        if (beam.eob) {
// 如果 next_beams 的大小小于 n_beams，则将当前的 beam 添加到 next_beams 中
if (next_beams.size() < n_beams) {
    next_beams.push_back(std::move(beam));
    // 如果 next_beams 的大小等于 n_beams，则对其进行堆排序
    if (next_beams.size() == n_beams) {
        std::make_heap(next_beams.begin(), next_beams.end(), comp);
    }
} 
// 如果 next_beams 的大小等于 n_beams 且当前 beam 的概率大于 next_beams 中最小概率的 beam，则替换最小概率的 beam
else if (next_beams.front().p < beam.p) {
    std::pop_heap(next_beams.begin(), next_beams.end(), comp); // 弹出堆顶元素
    next_beams.back() = std::move(beam); // 将当前 beam 移动到堆尾
    std::push_heap(next_beams.begin(), next_beams.end(), comp); // 将堆尾元素调整到合适位置
} 
// 如果当前 beam 不在句子末尾，则使用下一个 top_k 的 token 进行分支
else {
    if (!beam.tokens.empty()) {
        llama_decode(ctx, llama_batch_get_one(beam.tokens.data(), beam.tokens.size(), n_past, 0)); // 使用 beam 的 tokens 进行解码
    }
    llama_logit_info logit_info(ctx);
    std::vector<llama_token_data> next_tokens = logit_info.top_k(n_beams); // 获取下一个 top_k 的 token
    size_t i=0;
    // 如果 next_beams 的大小小于 n_beams，则继续添加 token
    if (next_beams.size() < n_beams) {
# 当下一个 beam 的数量小于指定数量时，进行循环
for (; next_beams.size() < n_beams ; ++i) {
    # 创建下一个 beam，并将下一个 token 的 id 添加到 tokens 中
    llama_beam next_beam = beam;
    next_beam.tokens.push_back(next_tokens[i].id);
    # 计算下一个 beam 的概率，并将其添加到 next_beam 的概率中
    next_beam.p *= logit_info.probability_from_logit(next_tokens[i].logit);
    # 将 next_beam 添加到 next_beams 中
    next_beams.push_back(std::move(next_beam));
}
# 对 next_beams 进行堆排序
std::make_heap(next_beams.begin(), next_beams.end(), comp);
# 如果下一个 beam 的概率为 0，则进行循环
} else {
    for (; next_beams.front().p == 0.0f ; ++i) {
        # 弹出堆顶元素
        std::pop_heap(next_beams.begin(), next_beams.end(), comp);
        # 将当前 beam 赋值给 next_beams 的最后一个元素
        next_beams.back() = beam;
        # 将下一个 token 的 id 添加到 tokens 中
        next_beams.back().tokens.push_back(next_tokens[i].id);
        # 计算下一个 beam 的概率，并将其添加到 next_beams 的概率中
        next_beams.back().p *= logit_info.probability_from_logit(next_tokens[i].logit);
        # 将 next_beams 进行堆排序
        std::push_heap(next_beams.begin(), next_beams.end(), comp);
    }
}
# 当 i 小于指定数量时，进行循环
for (; i < n_beams ; ++i) {
    # 计算下一个 beam 的概率
    const float next_p = beam.p * logit_info.probability_from_logit(next_tokens[i].logit);
    # 如果 next_beams 的堆顶元素的概率小于 next_p，则进行操作
    if (next_beams.front().p < next_p) {
        # 弹出堆顶元素
        std::pop_heap(next_beams.begin(), next_beams.end(), comp);
                    // 将当前的 beam 添加到下一个 beams 中
                    next_beams.back() = beam;
                    // 将下一个 token 的 id 添加到当前 beam 的 tokens 中
                    next_beams.back().tokens.push_back(next_tokens[i].id);
                    // 更新当前 beam 的概率值
                    next_beams.back().p = next_p;
                    // 使用 comp 函数重新构建最大堆
                    std::push_heap(next_beams.begin(), next_beams.end(), comp);
                }
            }
        }
    }

    // 基于 beams 找到公共前缀长度
    // 要求 beams 不为空
    size_t find_common_prefix_length() {
        // 初始化公共前缀长度为第一个 beam 的 tokens 长度
        size_t common_prefix_length = beams[0].tokens.size();
        // 遍历所有 beam，找到最小的 tokens 长度作为公共前缀长度
        for (size_t i = 1 ; i < beams.size() ; ++i) {
            common_prefix_length = std::min(common_prefix_length, beams[i].tokens.size());
            // 检查每个 beam 的 tokens 是否相同，找到最长的公共前缀长度
            for (size_t j = 0 ; j < common_prefix_length ; ++j) {
                if (beams[0].tokens[j] != beams[i].tokens[j]) {
                    common_prefix_length = j;
                    break;
                }
    }
    }
    // 返回公共前缀长度
    return common_prefix_length;
}

// 构建 beams_state 以通过回调函数发送回调函数
// 副作用：设置 common_prefix_length = find_common_prefix_length();
llama_beams_state get_beams_state(const bool last_call) {
    // 遍历 beams 数组，获取每个 beam 的视图
    for (size_t i = 0 ; i < beams.size() ; ++i) {
        beam_views[i] = beams[i].view();
    }
    // 获取公共前缀长度
    common_prefix_length = find_common_prefix_length();
    // 返回 beams_state 对象
    return {beam_views.data(), beams.size(), common_prefix_length, last_call};
}

// 循环：
//  * 当 i < n_predict 时，并且
//  * 任何一个 beam 还没有到达结束位置，并且
//  * 最高概率的 beam（如果有并列则为多个）还没有到达句子的结束位置
//    （因为所有其他 beam 的概率只会减小）
// 循环执行 beam search，直到达到预测次数或者所有 beam 都结束
void loop(const llama_beam_search_callback_fn_t callback, void * const callback_data) {
    // 初始化一个空的 beam，概率为 1.0，结束标志为 false
    beams.push_back({{}, 1.0f, false});
    // 定义一个 lambda 函数，用于判断 beam 是否结束
    const auto not_eob = [](const llama_beam & beam) { return !beam.eob; };
    // 循环执行 beam search
    for (int i = 0 ; i < n_predict && std::any_of(beams.begin(),beams.end(),not_eob) &&
                       !beams[top_beam_index()].eob ; ++i) {
        // 调用回调函数，获取 beam 状态
        callback(callback_data, get_beams_state(false));
        // 更新 beam 的值
        update_beams_from_beam_views();
        // 如果有公共前缀长度，解码并更新 n_past
        if (common_prefix_length) {
            llama_decode(ctx, llama_batch_get_one(beams[0].tokens.data(), common_prefix_length, n_past, 0));
            n_past += common_prefix_length;
        }
        // 将下一个 beam 的概率置为 0，以便放置在最小堆的最后
        std::for_each(next_beams.begin(), next_beams.end(), [](llama_beam & beam) { beam.p = 0.0f; });
        // 根据当前 beam 的概率填充下一个 beam
        for (llama_beam & beam : beams) {
            beam.shift_tokens(common_prefix_length);
            fill_next_beams_by_top_probabilities(beam);
        }
        // 下一个 beam 成为下一次迭代的 beam，交换它们以重用内存
        beams.swap(next_beams);
        // 重新计算 beam 的概率
        renormalize_beam_probabilities(beams);
    }
}
    }
    // 折叠顶部梁
    collapse_beams(top_beam_index());
    // 调用回调函数，传入回调数据和获取梁状态
    callback(callback_data, get_beams_state(true));
}

// 随着梁的增长，累积概率会减小。
// 重新归一化它们，以避免浮点下溢。
static void renormalize_beam_probabilities(std::vector<llama_beam> & beams) {
    // 计算概率之和
    const auto sum_p = [](float sum, llama_beam & beam) { return sum + beam.p; };
    // 计算概率之和的倒数
    const float inv_sum = 1.0f / std::accumulate(beams.begin(), beams.end(), 0.0f, sum_p);
    // 对每个梁的概率进行归一化
    std::for_each(beams.begin(), beams.end(), [=](llama_beam & beam) { beam.p *= inv_sum; });
}

// 假设梁不为空。使用llama_beam::operator<()进行排序。
size_t top_beam_index() {
    // 返回概率最大的梁的索引
    return std::max_element(beams.begin(), beams.end()) - beams.begin();
}

// 从梁视图更新梁的副本（p，eob）
void update_beams_from_beam_views() {
// 遍历 beams 容器中的元素，将 beam_views 容器中对应位置的数据赋值给 beams 容器中的元素
for (size_t i = 0 ; i < beams.size() ; ++i) {
    beams[i].p = beam_views[i].p; // 将 beam_views 容器中第 i 个元素的 p 值赋给 beams 容器中第 i 个元素的 p 值
    beams[i].eob = beam_views[i].eob; // 将 beam_views 容器中第 i 个元素的 eob 值赋给 beams 容器中第 i 个元素的 eob 值
}

// 定义 llama_beam_search 函数，接受回调函数、回调数据、束数量、过去数量和预测数量作为参数
void llama_beam_search(llama_context * ctx,
                       llama_beam_search_callback_fn_t callback, void * callback_data,
                       size_t n_beams, int n_past, int n_predict) {
    assert(ctx); // 断言 ctx 不为空

    // 获取当前时间戳
    const int64_t t_start_sample_us = ggml_time_us();

    // 创建 llama_beam_search_data 对象，传入上下文、束数量、过去数量和预测数量
    llama_beam_search_data beam_search_data(ctx, n_beams, n_past, n_predict);

    // 调用 beam_search_data 对象的 loop 方法，传入回调函数和回调数据
    beam_search_data.loop(callback, callback_data);

    // 更新上下文的采样时间和采样次数
    ctx->t_sample_us += ggml_time_us() - t_start_sample_us;
    ctx->n_sample++;
}
// 定义一个模板结构体，用于表示未初始化的值
template <typename T>
struct no_init {
    T value;
    no_init() { /* do nothing */ } // 无参构造函数，不执行任何操作
};

// 定义一个内部量化状态结构体
struct quantize_state_internal {
    const llama_model                 & model; // 引用类型成员变量，指向 llama_model 对象
    const llama_model_quantize_params * params; // 指针类型成员变量，指向 llama_model_quantize_params 对象

    int n_attention_wv    = 0; // 初始化整型成员变量 n_attention_wv 为 0
    int n_feed_forward_w2 = 0; // 初始化整型成员变量 n_feed_forward_w2 为 0
    int i_attention_wv    = 0; // 初始化整型成员变量 i_attention_wv 为 0
    int i_feed_forward_w2 = 0; // 初始化整型成员变量 i_feed_forward_w2 为 0
    # 初始化变量 n_k_quantized 和 n_fallback
    int n_k_quantized     = 0;
    int n_fallback        = 0;

    # 定义 quantize_state_internal 函数，接受一个 llama_model 对象和一个 llama_model_quantize_params 指针作为参数
    quantize_state_internal(const llama_model & model, const llama_model_quantize_params * params)
        : model(model)
        , params(params)
        {}
};

# 定义静态函数 llama_convert_tensor_internal，接受一个 ggml_tensor 指针，一个存储 float 类型数据的向量，一个存储线程对象的向量，元素数量和线程数量作为参数
static void llama_convert_tensor_internal(
    struct ggml_tensor * tensor, std::vector<no_init<float>> & output, std::vector<std::thread> & workers,
    const size_t nelements, const int nthread
) {
    # 如果输出向量的大小小于元素数量，调整输出向量的大小为元素数量
    if (output.size() < nelements) {
        output.resize(nelements);
    }
    # 将输出向量的数据类型转换为 float 指针
    float * f32_output = (float *) output.data();

    # 定义 ggml_type_traits_t 类型的变量 qtype
    ggml_type_traits_t qtype;
    # 如果输入的 tensor 类型是量化类型
    if (ggml_is_quantized(tensor->type)) {
# 获取张量的类型特征
qtype = ggml_internal_get_type_traits(tensor->type);
# 如果没有浮点数转换函数，则抛出运行时错误
if (qtype.to_float == NULL) {
    throw std::runtime_error(format("type %s unsupported for integer quantization: no dequantization available", ggml_type_name(tensor->type)));
} 
# 如果张量类型不是 GGML_TYPE_F16，则抛出运行时错误
else if (tensor->type != GGML_TYPE_F16) {
    throw std::runtime_error(format("cannot dequantize/convert tensor type %s", ggml_type_name(tensor->type)));
}

# 如果线程数小于 2
if (nthread < 2) {
    # 如果张量类型是 GGML_TYPE_F16，则调用 ggml_fp16_to_fp32_row 函数
    if (tensor->type == GGML_TYPE_F16) {
        ggml_fp16_to_fp32_row((ggml_fp16_t *)tensor->data, f32_output, nelements);
    } 
    # 如果张量类型是量化类型，则调用对应的转换函数
    else if (ggml_is_quantized(tensor->type)) {
        qtype.to_float(tensor->data, f32_output, nelements);
    } 
    # 否则抛出错误
    else {
        GGML_ASSERT(false); // unreachable
    }
    return;
}

# 计算块大小，如果张量类型是 GGML_TYPE_F16，则块大小为 1，否则为对应类型的块大小
auto block_size = tensor->type == GGML_TYPE_F16 ? 1 : (size_t)ggml_blck_size(tensor->type);
    // 计算每个数据块的字节数
    auto block_size_bytes = ggml_type_size(tensor->type);

    // 断言数据元素个数能够被块大小整除
    GGML_ASSERT(nelements % block_size == 0);
    // 计算数据块的个数
    auto nblocks = nelements / block_size;
    // 计算每个线程处理的数据块个数
    auto blocks_per_thread = nblocks / nthread;
    // 计算不能被线程数整除的剩余数据块个数
    auto spare_blocks = nblocks - (blocks_per_thread * nthread); // if blocks aren't divisible by thread count

    // 遍历每个线程
    for (auto tnum = 0, in_buff_offs = 0, out_buff_offs = 0; tnum < nthread; tnum++) {
        // 计算当前线程处理的数据块个数
        auto thr_blocks = blocks_per_thread + (tnum == nthread - 1 ? spare_blocks : 0); // num blocks for this thread
        // 计算当前线程处理的元素个数
        auto thr_elems = thr_blocks * block_size; // number of elements for this thread
        // 计算当前线程处理的输入数据字节数
        auto thr_block_bytes = thr_blocks * block_size_bytes; // number of input bytes for this thread

        // 定义一个计算函数，根据数据类型进行不同的计算
        auto compute = [qtype] (ggml_type typ, uint8_t * inbuf, float * outbuf, int nels) {
            if (typ == GGML_TYPE_F16) {
                ggml_fp16_to_fp32_row((ggml_fp16_t *)inbuf, outbuf, nels);
            } else {
                qtype.to_float(inbuf, outbuf, nels);
            }
        };
        // 将计算函数添加到线程池中
        workers.emplace_back(compute, tensor->type, (uint8_t *) tensor->data + in_buff_offs, f32_output + out_buff_offs, thr_elems);
        // 增加输入缓冲区偏移量
        in_buff_offs += thr_block_bytes;
        // 增加输出缓冲区偏移量
        out_buff_offs += thr_elems;
    }
    // 等待所有线程完成
    for (auto & w : workers) { w.join(); }
    // 清空线程容器
    workers.clear();
}

// 获取量化类型
static ggml_type get_k_quant_type(
    quantize_state_internal & qs,
    ggml_type new_type, const ggml_tensor * tensor, llama_ftype ftype
) {
    // 获取张量的名称
    const std::string name = ggml_get_name(tensor);
    // TODO: 避免硬编码张量名称 - 使用 TN_* 常量
    // 获取架构类型
    const llm_arch arch = qs.model.arch;
    // 获取张量名称
    const auto       tn = LLM_TN(arch);

    // 判断是否使用更多位
    auto use_more_bits = [](int i_layer, int num_layers) -> bool {
        return i_layer < num_layers/8 || i_layer >= 7*num_layers/8 || (i_layer - num_layers/8)%3 == 2;
    };
# 如果名称为LLM_TENSOR_OUTPUT的权重张量
if (name == tn(LLM_TENSOR_OUTPUT, "weight").first) {
    # 获取张量的第一个维度大小
    int nx = tensor->ne[0];
    # 如果架构为LLM_ARCH_FALCON或者nx不能被QK_K整除
    if (arch == LLM_ARCH_FALCON || nx % QK_K != 0) {
        # 设置新类型为GGML_TYPE_Q8_0
        new_type = GGML_TYPE_Q8_0;
    }
    # 如果新类型不等于GGML_TYPE_Q8_0
    else if (new_type != GGML_TYPE_Q8_0) {
        # 设置新类型为GGML_TYPE_Q6_K
        new_type = GGML_TYPE_Q6_K;
    }
} 
# 如果名称包含"attn_v.weight"
else if (name.find("attn_v.weight") != std::string::npos) {
    # 根据不同的ftype设置新类型
    if      (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) new_type = GGML_TYPE_Q3_K;
    else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M) {
        # 根据条件设置新类型为GGML_TYPE_Q5_K或GGML_TYPE_Q4_K
        new_type = qs.i_attention_wv < 2 ? GGML_TYPE_Q5_K : GGML_TYPE_Q4_K;
    }
    else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q5_K;
    else if ((ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M || ftype == LLAMA_FTYPE_MOSTLY_Q5_K_M) &&
            use_more_bits(qs.i_attention_wv, qs.n_attention_wv)) new_type = GGML_TYPE_Q6_K;
    else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S && qs.i_attention_wv < 4) new_type = GGML_TYPE_Q5_K;
    else if (QK_K == 64 && (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S || ftype == LLAMA_FTYPE_MOSTLY_Q3_K_S) &&
            (qs.i_attention_wv < qs.n_attention_wv/8 || qs.i_attention_wv >= 7*qs.n_attention_wv/8)) new_type = GGML_TYPE_Q6_K;
    # 如果模型类型为MODEL_70B
    if (qs.model.type == MODEL_70B) {
// 在70B模型中，我们有8个头共享相同的attn_v权重。因此，attn_v.weight张量比attn_q.weight小8倍。因此，我们可以通过更多的比特量化这个张量，几乎可以忽略不计地增加模型大小，从而获得量化精度的显著提升。
if (new_type == GGML_TYPE_Q3_K || new_type == GGML_TYPE_Q4_K) new_type = GGML_TYPE_Q5_K;

// 如果名称中包含"ffn_down.weight"，则执行以下操作
} else if (name.find("ffn_down.weight") != std::string::npos) {
    // 如果ftype为LLAMA_FTYPE_MOSTLY_Q2_K，则将new_type设置为GGML_TYPE_Q3_K
    if (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) new_type = GGML_TYPE_Q3_K;
    // 如果ftype为LLAMA_FTYPE_MOSTLY_Q3_K_M，则根据条件设置new_type
    else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M) {
        new_type = qs.i_feed_forward_w2 < 2 ? GGML_TYPE_Q5_K
                 : arch != LLM_ARCH_FALCON || use_more_bits(qs.i_feed_forward_w2, qs.n_feed_forward_w2) ? GGML_TYPE_Q4_K
                 : GGML_TYPE_Q3_K;
    }
    // 如果ftype为LLAMA_FTYPE_MOSTLY_Q3_K_L，则根据条件设置new_type
    else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) {
        new_type = arch == LLM_ARCH_FALCON ? GGML_TYPE_Q4_K : GGML_TYPE_Q5_K;
    }
    // 如果ftype为LLAMA_FTYPE_MOSTLY_Q4_K_M，则根据条件设置new_type
    else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M) {
        if (arch == LLM_ARCH_FALCON) {
            new_type = qs.i_feed_forward_w2 < 2 ? GGML_TYPE_Q6_K :
                       use_more_bits(qs.i_feed_forward_w2, qs.n_feed_forward_w2) ? GGML_TYPE_Q5_K : GGML_TYPE_Q4_K;
    } else {
        // 如果条件不满足，且满足使用更多位数的条件，则设置新类型为 GGML_TYPE_Q6_K
        if (use_more_bits(qs.i_feed_forward_w2, qs.n_feed_forward_w2)) new_type = GGML_TYPE_Q6_K;
    }
}
else if (ftype == LLAMA_FTYPE_MOSTLY_Q5_K_M && use_more_bits(qs.i_feed_forward_w2, qs.n_feed_forward_w2)) new_type = GGML_TYPE_Q6_K;
else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_S && arch != LLM_ARCH_FALCON && qs.i_feed_forward_w2 < 4) {
    // 如果条件满足，则设置新类型为 GGML_TYPE_Q5_K
    new_type = GGML_TYPE_Q5_K;
}
// 递增 qs.i_feed_forward_w2
++qs.i_feed_forward_w2;
} else if (name.find("attn_output.weight") != std::string::npos) {
    if (arch != LLM_ARCH_FALCON) {
        // 根据不同的条件设置不同的新类型
        if      (ftype == LLAMA_FTYPE_MOSTLY_Q2_K  ) new_type = GGML_TYPE_Q3_K;
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M) new_type = GGML_TYPE_Q4_K;
        else if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q5_K;
    } else {
        // 如果条件满足，则设置新类型为 GGML_TYPE_Q4_K
        if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q4_K;
    }
}
else if (name.find("attn_qkv.weight") != std::string::npos) {
    // 根据不同的条件设置不同的新类型
    if (ftype == LLAMA_FTYPE_MOSTLY_Q3_K_M || ftype == LLAMA_FTYPE_MOSTLY_Q3_K_L) new_type = GGML_TYPE_Q4_K;
    // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q4_K_M，则将新类型设置为GGML_TYPE_Q5_K
    else if (ftype == LLAMA_FTYPE_MOSTLY_Q4_K_M) new_type = GGML_TYPE_Q5_K;
    // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q5_K_M，则将新类型设置为GGML_TYPE_Q6_K
    else if (ftype == LLAMA_FTYPE_MOSTLY_Q5_K_M) new_type = GGML_TYPE_Q6_K;
    // 如果文件名包含"ffn_gate.weight"或"ffn_up.weight"，且文件类型为LLAMA_FTYPE_MOSTLY_Q2_K，则将新类型设置为GGML_TYPE_Q3_K
    else if (name.find("ffn_gate.weight") != std::string::npos || name.find("ffn_up.weight") != std::string::npos) {
        if (ftype == LLAMA_FTYPE_MOSTLY_Q2_K) new_type = GGML_TYPE_Q3_K;
    }
    // 如果文件名包含"fc1.weight"或"fc2.weight"，根据文件类型设置新类型为GGML_TYPE_Q4_0或GGML_TYPE_Q5_0
    else if (name.find("fc1.weight") != std::string::npos || name.find("fc2.weight") != std::string::npos) {
        if (ftype == LLAMA_FTYPE_MOSTLY_Q4_0) new_type = GGML_TYPE_Q4_0;
        else new_type = GGML_TYPE_Q5_0;
    }
    // 如果文件类型为LLAMA_FTYPE_MOSTLY_Q5_K_S，则将新类型设置为GGML_TYPE_Q4_K
    // 以下注释部分是被注释掉的代码，不会被执行
    // This can be used to reduce the size of the Q5_K_S model.
    // The associated PPL increase is fully in line with the size reduction
    //else {
    //    if (ftype == LLAMA_FTYPE_MOSTLY_Q5_K_S) new_type = GGML_TYPE_Q4_K;
    //}
    // 初始化一个布尔变量convert_incompatible_tensor为false
    bool convert_incompatible_tensor = false;
    // 如果新类型为GGML_TYPE_Q2_K、GGML_TYPE_Q3_K、GGML_TYPE_Q4_K、GGML_TYPE_Q5_K或GGML_TYPE_Q6_K，则执行以下操作
    if (new_type == GGML_TYPE_Q2_K || new_type == GGML_TYPE_Q3_K || new_type == GGML_TYPE_Q4_K ||
        new_type == GGML_TYPE_Q5_K || new_type == GGML_TYPE_Q6_K) {
        // 获取tensor的第一个和第二个维度的大小
        int nx = tensor->ne[0];
        int ny = tensor->ne[1];
# 如果nx除以QK_K的余数不为0，则输出警告信息并将convert_incompatible_tensor设置为true
if (nx % QK_K != 0) {
    LLAMA_LOG_WARN("\n\n%s : tensor cols %d x %d are not divisible by %d, required for %s", __func__, nx, ny, QK_K, ggml_type_name(new_type));
    convert_incompatible_tensor = true;
} else {
    # 如果能够整除，则增加n_k_quantized计数
    ++qs.n_k_quantized;
}

# 如果convert_incompatible_tensor为true，则根据new_type进行不兼容类型的转换
if (convert_incompatible_tensor) {
    switch (new_type) {
        case GGML_TYPE_Q2_K: new_type = GGML_TYPE_Q4_0; break;
        case GGML_TYPE_Q3_K: new_type = GGML_TYPE_Q4_1; break;
        case GGML_TYPE_Q4_K: new_type = GGML_TYPE_Q5_0; break;
        case GGML_TYPE_Q5_K: new_type = GGML_TYPE_Q5_1; break;
        case GGML_TYPE_Q6_K: new_type = GGML_TYPE_Q8_0; break;
        default: throw std::runtime_error("\nUnsupported tensor size encountered\n");
    }
    # 输出警告信息并增加n_fallback计数
    LLAMA_LOG_WARN(" - using fallback quantization %s\n", ggml_type_name(new_type));
    ++qs.n_fallback;
}
    // 返回新的类型
    return new_type;
}

// 内部量化模型函数，将输入文件名和输出文件名以及量化参数传入
static void llama_model_quantize_internal(const std::string & fname_inp, const std::string & fname_out, const llama_model_quantize_params * params) {
    // 定义量化后的类型
    ggml_type quantized_type;
    // 获取参数中的数据类型
    llama_ftype ftype = params->ftype;

    // 根据不同的数据类型进行量化
    switch (params->ftype) {
        case LLAMA_FTYPE_MOSTLY_Q4_0: quantized_type = GGML_TYPE_Q4_0; break;
        case LLAMA_FTYPE_MOSTLY_Q4_1: quantized_type = GGML_TYPE_Q4_1; break;
        case LLAMA_FTYPE_MOSTLY_Q5_0: quantized_type = GGML_TYPE_Q5_0; break;
        case LLAMA_FTYPE_MOSTLY_Q5_1: quantized_type = GGML_TYPE_Q5_1; break;
        case LLAMA_FTYPE_MOSTLY_Q8_0: quantized_type = GGML_TYPE_Q8_0; break;
        case LLAMA_FTYPE_MOSTLY_F16:  quantized_type = GGML_TYPE_F16;  break;
        case LLAMA_FTYPE_ALL_F32:     quantized_type = GGML_TYPE_F32;  break;

        // K-quants
        case LLAMA_FTYPE_MOSTLY_Q2_K:   quantized_type = GGML_TYPE_Q2_K; break;
        case LLAMA_FTYPE_MOSTLY_Q3_K_S:
        case LLAMA_FTYPE_MOSTLY_Q3_K_M:
// 根据不同的文件类型设置相应的量化类型
case LLAMA_FTYPE_MOSTLY_Q3_K_L: quantized_type = GGML_TYPE_Q3_K; break;
case LLAMA_FTYPE_MOSTLY_Q4_K_S:
case LLAMA_FTYPE_MOSTLY_Q4_K_M: quantized_type = GGML_TYPE_Q4_K; break;
case LLAMA_FTYPE_MOSTLY_Q5_K_S:
case LLAMA_FTYPE_MOSTLY_Q5_K_M: quantized_type = GGML_TYPE_Q5_K; break;
case LLAMA_FTYPE_MOSTLY_Q6_K:   quantized_type = GGML_TYPE_Q6_K; break;

// 如果线程数小于等于0，则设置为系统可用线程数
int nthread = params->nthread;
if (nthread <= 0) {
    nthread = std::thread::hardware_concurrency();
}

// 根据操作系统类型确定是否使用 mmap 来增加速度
// 在 Linux 上使用 mmap 一直会增加速度，在 Windows 上热缓存时也会增加速度
// 在 macOS 上可能会减慢速度，可能与空闲内存有关
#if defined(__linux__) || defined(_WIN32)
constexpr bool use_mmap = true;
// 如果条件为假，则 use_mmap 常量为 false
#else
    constexpr bool use_mmap = false;
#endif

    // 使用给定的文件名创建 llama_model_loader 对象
    llama_model_loader ml(fname_inp, use_mmap);
    
    // 如果使用内存映射，则创建内存映射对象
    if (ml.use_mmap) {
        ml.mapping.reset(new llama_mmap(&ml.file, /* prefetch */ 0, ggml_is_numa()));
    }

    // 创建 llama_model 对象
    llama_model model;
    
    // 加载架构信息到 model 对象
    llm_load_arch(ml, model);
    
    // 加载超参数信息到 model 对象
    llm_load_hparams(ml, model);

    // 创建 quantize_state_internal 对象
    struct quantize_state_internal qs(model, params);

    // 如果只需要复制，则设置 ftype 为 model.ftype
    if (params->only_copy) {
        ftype = model.ftype;
    }

    // 设置默认对齐大小
    const size_t align = GGUF_DEFAULT_ALIGNMENT;
// 根据 ml.sparse_deriv 的值选择初始化稀疏或非稀疏的 gguf_context 结构体
struct gguf_context * ctx_out = ml.sparse_deriv == GGML_SPARSE_INFERENCE ? gguf_init_empty_sparse() : gguf_init_empty();

// 将输入文件中的键值对复制到 ctx_out 中
gguf_set_kv     (ctx_out, ml.ctx_gguf);
// 设置 general.quantization_version 的值为 GGML_QNT_VERSION
gguf_set_val_u32(ctx_out, "general.quantization_version", GGML_QNT_VERSION);
// 设置 general.file_type 的值为 ftype
gguf_set_val_u32(ctx_out, "general.file_type", ftype);

// 遍历 ml.n_tensors 个张量
for (int i = 0; i < ml.n_tensors; ++i) {
    // 获取第 i 个张量的元数据
    struct ggml_tensor * meta = ml.get_tensor_meta(i);

    // 获取张量的名称
    const std::string name = ggml_get_name(meta);

    // 如果张量名称中包含 "attn_v.weight" 或 "attn_qkv.weight"，则增加 qs.n_attention_wv 的值
    // 否则，如果张量名称中包含 "ffn_down.weight"，则增加 qs.n_feed_forward_w2 的值
    // TODO: 避免硬编码张量名称 - 使用 TN_* 常量
    if (name.find("attn_v.weight") != std::string::npos || name.find("attn_qkv.weight") != std::string::npos) {
        ++qs.n_attention_wv;
    }
    else if (name.find("ffn_down.weight") != std::string::npos) {
        ++qs.n_feed_forward_w2;
    }
}
    // 检查条件，如果不满足则输出警告信息
    if (qs.n_attention_wv != qs.n_feed_forward_w2 || (uint32_t)qs.n_attention_wv != model.hparams.n_layer) {
        LLAMA_LOG_WARN("%s ============ Strange model: n_attention_wv = %d, n_feed_forward_w2 = %d, hparams.n_layer = %d\n",
                __func__, qs.n_attention_wv, qs.n_feed_forward_w2, model.hparams.n_layer);
    }

    // 初始化变量
    size_t total_size_org = 0;
    size_t total_size_new = 0;
    // 创建一个大小为 16 的整型数组，初始化为 0
    std::vector<int64_t> hist_all(1 << 4, 0);

    // 创建线程数组，预留 nthread 个位置
    std::vector<std::thread> workers;
    workers.reserve(nthread);
    // 创建互斥锁
    std::mutex mutex;

    // 初始化变量
    int idx = 0;

    // 创建三个未初始化的字节型数组
    std::vector<no_init<uint8_t>> read_data;
    std::vector<no_init<uint8_t>> work;
    std::vector<no_init<float>> f32_conv_buf;

    // 填充原始张量，以获取初始元数据
    // 遍历 ml 中的张量数量
    for (int i = 0; i < ml.n_tensors; ++i) {
        // 获取第 i 个张量的元数据
        struct ggml_tensor * meta = ml.get_tensor_meta(i);
        // 将张量的元数据添加到输出上下文中
        gguf_add_tensor(ctx_out, meta);
    }

    // 以二进制模式打开输出文件
    std::ofstream fout(fname_out, std::ios::binary);
    // 设置在写入错误时立即抛出异常
    fout.exceptions(std::ofstream::failbit); // fail fast on write errors

    // 获取输出上下文中元数据的大小
    const size_t meta_size = gguf_get_meta_size(ctx_out);

    // 记录元数据的大小
    LLAMA_LOG_INFO("%s: meta size = %zu bytes\n", __func__, meta_size);

    // 在输出文件中创建元数据的占位符
    ::zeros(fout, meta_size);

    // 遍历 ml 中的张量数量
    for (int i = 0; i < ml.n_tensors; ++i) {
        // 获取第 i 个张量的元数据
        struct ggml_tensor * tensor = ml.get_tensor_meta(i);
        // 获取张量的名称
        const std::string name = ggml_get_name(tensor);
        // 如果不使用内存映射，则执行以下操作
        if (!ml.use_mmap) {
            // 如果读取的数据大小小于张量的总字节数，则调整读取数据的大小
            if (read_data.size() < ggml_nbytes(tensor)) {
                read_data.resize(ggml_nbytes(tensor));
            }
            // 将张量的数据指针指向读取的数据
            tensor->data = read_data.data();
        }
        // 为张量加载数据
        ml.load_data_for(tensor);

        // 输出日志信息，包括张量的索引、名称、形状、类型等信息
        LLAMA_LOG_INFO("[%4d/%4d] %36s - [%s], type = %6s, ",
               ++idx, ml.n_tensors,
               ggml_get_name(tensor),
               llama_format_tensor_shape(tensor).c_str(),
               ggml_type_name(tensor->type));

        // 这里原本是一个正则表达式，但是 <regex> 的编译时间开销非常大
        // 检查张量名称是否以 'weight' 结尾，用于量化
        bool quantize = name.rfind("weight") == name.size() - 6; // ends with 'weight'?

        // 只对二维张量进行量化
        quantize &= (tensor->n_dims == 2);
        // 如果参数允许量化输出张量，或者张量名称不是 "output.weight"，则进行量化
        quantize &= params->quantize_output_tensor || name != "output.weight";
        # 按位与操作，将 quantize 变量更新为其与 params->only_copy 的逻辑非
        quantize &= !params->only_copy;

        # 声明枚举类型 new_type，以及指向数据的指针 new_data 和数据大小 new_size
        enum ggml_type new_type;
        void * new_data;
        size_t new_size;

        # 如果需要进行量化
        if (quantize) {
            # 将 new_type 更新为 quantized_type
            new_type = quantized_type;
            # 如果不是纯量化，则调用 get_k_quant_type 函数获取新的量化类型
            if (!params->pure) {
                new_type = get_k_quant_type(qs, new_type, tensor, ftype);
            }

            # 如果决定量化到与张量已经存在的类型相同，则不需要进行量化
            quantize = tensor->type != new_type;
        }
        # 如果不需要量化
        if (!quantize) {
            # 将 new_type 更新为张量的类型
            new_type = tensor->type;
            # 将 new_data 更新为张量的数据
            new_data = tensor->data;
            # 将 new_size 更新为张量数据的大小
            new_size = ggml_nbytes(tensor);
// 输出日志，显示 tensor 的大小（以 MB 为单位）
LLAMA_LOG_INFO("size = %8.3f MB\n", ggml_nbytes(tensor)/1024.0/1024.0);

// 如果 tensor 的类型为 GGML_TYPE_F32
if (tensor->type == GGML_TYPE_F32) {
    // 将 tensor 的数据转换为 float 类型的指针
    f32_data = (float *) tensor->data;
} 
// 如果 tensor 的类型为量化类型，并且不允许重新量化
else if (ggml_is_quantized(tensor->type) && !params->allow_requantize) {
    // 抛出运行时错误，显示禁用从特定类型重新量化的信息
    throw std::runtime_error(format("requantizing from type %s is disabled", ggml_type_name(tensor->type)));
} 
// 如果不满足上述条件
else {
    // 调用 llama_convert_tensor_internal 函数将 tensor 转换为 float 类型的数据，并存储在 f32_conv_buf 中
    llama_convert_tensor_internal(tensor, f32_conv_buf, workers, nelements, nthread);
    // 将 f32_conv_buf 转换为 float 类型的指针
    f32_data = (float *) f32_conv_buf.data();
}

// 输出日志，显示量化到新类型的信息
LLAMA_LOG_INFO("quantizing to %s .. ", ggml_type_name(new_type));
// 刷新标准输出缓冲区
fflush(stdout);

// 如果 work 的大小小于 nelements * 4
if (work.size() < nelements * 4) {
    // 调整 work 的大小为 nelements * 4，作为大小的上限
    work.resize(nelements * 4); // upper bound on size
}
            }
            // 从work对象中获取数据
            new_data = work.data();
            // 创建一个包含16个int64_t类型元素的静态数组
            std::array<int64_t, 1 << 4> hist_cur = {};

            // 定义一个常量chunk_size，并赋值为32*512
            static const int chunk_size = 32 * 512;
            // 计算需要处理的块数
            const int nchunk = (nelements + chunk_size - 1)/chunk_size;
            // 计算使用的线程数
            const int nthread_use = nthread > 1 ? std::max(1, std::min(nthread, nchunk)) : 1;
            // 如果使用的线程数小于2
            if (nthread_use < 2) {
                // 对数据进行量化处理
                new_size = ggml_quantize_chunk(new_type, f32_data, new_data, 0, nelements, hist_cur.data());
            } else {
                // 初始化计数器和新数据大小
                size_t counter = 0;
                new_size = 0;
                // 定义一个lambda表达式compute
                auto compute = [&mutex, &counter, &hist_cur, &new_size, new_type, f32_data, new_data, nelements]() {
                    // 创建一个包含16个int64_t类型元素的局部数组
                    std::array<int64_t, 1 << 4> local_hist = {};
                    // 初始化局部大小
                    size_t local_size = 0;
                    // 循环处理数据
                    while (true) {
                        // 加锁
                        std::unique_lock<std::mutex> lock(mutex);
                        // 获取当前处理的起始位置
                        size_t first = counter; counter += chunk_size;
                        // 如果起始位置超出数据范围
                        if (first >= nelements) {
                            // 如果局部大小大于0
                            if (local_size > 0) {
# 遍历 local_hist 列表，将其值加到 hist_cur 列表对应位置上
for (int j=0; j<int(local_hist.size()); ++j) {
    hist_cur[j] += local_hist[j];
}
# 将 local_size 的值加到 new_size 上
new_size += local_size;

# 退出循环
break;

# 释放锁
lock.unlock();

# 计算最后一个元素的位置
size_t last = std::min(nelements, first + chunk_size);

# 将 local_hist 列表中的数据进行量化处理，将结果存储到 new_data 中
local_size += ggml_quantize_chunk(new_type, f32_data, new_data, first, last - first, local_hist.data());

# 创建多个线程执行 compute 函数
for (int it = 0; it < nthread_use - 1; ++it) {
    workers.emplace_back(compute);
}

# 执行 compute 函数
compute();

# 等待所有线程执行完毕
for (auto & w : workers) { w.join(); }

# 清空 workers 列表
workers.clear();
// 输出日志，显示原始大小和新大小，并输出直方图
LLAMA_LOG_INFO("size = %8.2f MB -> %8.2f MB | hist: ", ggml_nbytes(tensor)/1024.0/1024.0, new_size/1024.0/1024.0);
// 初始化变量 tot_count
int64_t tot_count = 0;
// 遍历 hist_cur 数组，更新 hist_all 数组和计算 tot_count
for (size_t i = 0; i < hist_cur.size(); i++) {
    hist_all[i] += hist_cur[i];
    tot_count += hist_cur[i];
}
// 如果 tot_count 大于 0，则输出 hist_cur 数组的每个元素占总元素数的比例
if (tot_count > 0) {
    for (size_t i = 0; i < hist_cur.size(); i++) {
        LLAMA_LOG_INFO("%5.3f ", hist_cur[i] / float(nelements));
    }
}
// 输出换行符
LLAMA_LOG_INFO("\n");
// 更新总原始大小和总新大小
total_size_org += ggml_nbytes(tensor);
total_size_new += new_size;
// 更新 gguf 元数据
gguf_set_tensor_type(ctx_out, name.c_str(), new_type);
gguf_set_tensor_data(ctx_out, name.c_str(), new_data, new_size);
    // 写入张量数据和填充
    fout.write((const char *) new_data, new_size);
    // 在文件末尾添加零填充，使文件大小达到指定的对齐大小
    zeros(fout, GGML_PAD(new_size, align) - new_size);
    }

    // 回到文件开头并写入更新的元数据
    {
        fout.seekp(0);
        // 获取更新后的元数据大小，并将元数据写入文件
        std::vector<uint8_t> data(gguf_get_meta_size(ctx_out));
        gguf_get_meta_data(ctx_out, data.data());
        fout.write((const char *) data.data(), data.size());
    }

    // 关闭文件
    fout.close();

    // 释放上下文
    gguf_free(ctx_out);

    // 记录原始模型大小和量化后的模型大小
    LLAMA_LOG_INFO("%s: model size  = %8.2f MB\n", __func__, total_size_org/1024.0/1024.0);
    LLAMA_LOG_INFO("%s: quant size  = %8.2f MB\n", __func__, total_size_new/1024.0/1024.0);
// 打印所有张量的直方图
{
    int64_t sum_all = 0; // 初始化所有张量值的总和为0
    for (size_t i = 0; i < hist_all.size(); i++) { // 遍历所有张量的直方图
        sum_all += hist_all[i]; // 计算所有张量值的总和
    }

    if (sum_all > 0) { // 如果总和大于0
        LLAMA_LOG_INFO("%s: hist: ", __func__); // 打印函数名称和直方图信息
        for (size_t i = 0; i < hist_all.size(); i++) { // 再次遍历所有张量的直方图
            LLAMA_LOG_INFO("%5.3f ", hist_all[i] / float(sum_all)); // 打印每个张量值在总和中的比例
        }
        LLAMA_LOG_INFO("\n"); // 打印换行符
    }
}

if (qs.n_fallback > 0) { // 如果需要回退量化的张量数量大于0
    LLAMA_LOG_WARN("%s: WARNING: %d of %d tensor(s) incompatible with k-quants and required fallback quantization\n",
            __func__, qs.n_fallback, qs.n_k_quantized + qs.n_fallback); // 打印警告信息，显示需要回退量化的张量数量和总共需要量化的张量数量
}
    }
}

static int llama_apply_lora_from_file_internal(
    const struct llama_model & model, const char * path_lora, float scale, const char * path_base_model, int n_threads
) {
    // 打印信息，应用来自指定路径的 lora 适配器 - 请稍候...
    LLAMA_LOG_INFO("%s: applying lora adapter from '%s' - please wait ...\n", __func__, path_lora);

    // 获取开始应用 lora 适配器的时间
    const int64_t t_start_lora_us = ggml_time_us();

    // 打开指定路径的文件流
    auto fin = std::ifstream(path_lora, std::ios::binary);
    // 如果文件流打开失败，记录错误信息并返回 1
    if (!fin) {
        LLAMA_LOG_ERROR("%s: failed to open '%s'\n", __func__, path_lora);
        return 1;
    }

    // 验证文件的魔数和版本
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
    // 读取格式版本号
    uint32_t format_version;
    fin.read((char *) &format_version, sizeof(format_version));

    // 检查格式版本号是否为1，如果不是则输出错误信息并返回1
    if (format_version != 1) {
        LLAMA_LOG_ERROR("%s: unsupported file version\n", __func__ );
        return 1;
    }

    // 读取 lora_r 和 lora_alpha 的数值
    int32_t lora_r;
    int32_t lora_alpha;
    fin.read((char *) &lora_r, sizeof(lora_r));
    fin.read((char *) &lora_alpha, sizeof(lora_alpha));

    // 计算 scaling 值
    float scaling = scale * (float)lora_alpha / (float)lora_r;

    // 输出 r、alpha 和 scaling 的信息
    LLAMA_LOG_INFO("%s: r = %d, alpha = %d, scaling = %.2f\n", __func__, lora_r, lora_alpha, scaling);

    // 创建一个临时的 ggml 上下文来存储 lora 张量
    // todo: 从可能的最大张量计算大小
    std::vector<uint8_t> lora_buf(1024ull * 1024ull * 1024ull);
    // 定义结构体 ggml_init_params，用于初始化参数
    struct ggml_init_params params;
    // 设置内存大小为 lora_buf 的大小
    params.mem_size   = lora_buf.size();
    // 设置内存缓冲区为 lora_buf 的数据
    params.mem_buffer = lora_buf.data();
    // 设置为允许分配内存
    params.no_alloc   = false;

    // 使用初始化参数初始化 ggml 上下文
    ggml_context * lora_ctx = ggml_init(params);
    // 创建一个存储名称到张量的无序映射
    std::unordered_map<std::string, struct ggml_tensor *> lora_tensors;

    // 创建一个模型名称到张量的映射，以加速查找
    std::unordered_map<std::string, struct ggml_tensor*> model_tensors;
    // 遍历模型的张量，插入到模型张量映射中
    for (const auto & kv : model.tensors_by_name) {
        model_tensors.insert(kv);
    }

    // 加载基础模型
    std::unique_ptr<llama_model_loader> ml;
    ggml_context * base_ctx = NULL;
    std::vector<uint8_t> base_buf;
    // 如果存在基础模型路径
    if (path_base_model) {
        // 输出加载基础模型的信息
        LLAMA_LOG_INFO("%s: loading base model from '%s'\n", __func__, path_base_model);
// 重置指向llama_model_loader对象的智能指针，使用mmap进行内存映射
ml.reset(new llama_model_loader(path_base_model, /*use_mmap*/ true));

// 计算上下文大小和内存映射大小
size_t ctx_size;
size_t mmapped_size;
ml->calc_sizes(ctx_size, mmapped_size);
base_buf.resize(ctx_size);

// 初始化基础参数
ggml_init_params base_params;
base_params.mem_size   = base_buf.size();
base_params.mem_buffer = base_buf.data();
base_params.no_alloc   = ml->use_mmap;

// 初始化基础上下文
base_ctx = ggml_init(base_params);

// 如果使用mmap，则在llama_model_loader中进行内存映射
// 可能这应该在llama_model_loader中进行
if (ml->use_mmap) {
    ml->mapping.reset(new llama_mmap(&ml->file, /* prefetch */ 0, ggml_is_numa()));
}
    // 读取张量并应用
    bool warned = false; // 初始化一个布尔变量warned，用于标记是否已经发出警告
    int n_tensors = 0; // 初始化一个整型变量n_tensors，用于记录张量的数量

    std::vector<uint8_t> work_buffer; // 创建一个存储uint8_t类型数据的向量work_buffer

    while (true) { // 进入无限循环
        int32_t n_dims; // 定义一个int32_t类型变量n_dims，用于存储张量的维度
        int32_t length; // 定义一个int32_t类型变量length，用于存储张量的长度
        int32_t ftype; // 定义一个int32_t类型变量ftype，用于存储张量的类型

        fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims)); // 从输入流fin中读取n_dims的值
        fin.read(reinterpret_cast<char *>(&length), sizeof(length)); // 从输入流fin中读取length的值
        fin.read(reinterpret_cast<char *>(&ftype),  sizeof(ftype)); // 从输入流fin中读取ftype的值
        if (fin.eof()) { // 如果已经到达文件末尾
            break; // 退出循环
        }

        int32_t ne[2] = { 1, 1 }; // 创建一个长度为2的int32_t数组ne，并初始化为{1, 1}
        for (int i = 0; i < n_dims; ++i) { // 进入循环，遍历张量的维度
        // 从文件流中读取数据到ne[i]中，使用reinterpret_cast进行类型转换
        fin.read(reinterpret_cast<char *>(&ne[i]), sizeof(ne[i]));
        // 读取文件名
        std::string name;
        {
            // 创建一个缓冲区，读取文件名数据到缓冲区中
            char buf[1024];
            fin.read(buf, length);
            // 将缓冲区中的数据转换为字符串赋值给name
            name = std::string(buf, length);
        }

        // 检查文件名是否以.lora结尾，并获取张量的类型
        const std::string lora_suffix = ".lora";
        // 在文件名中查找.lora的位置
        size_t pos = name.rfind(lora_suffix);
        // 如果未找到.lora后缀，则输出错误信息并返回1
        if (pos == std::string::npos) {
            LLAMA_LOG_ERROR("%s: error: '%s' is not a lora tensor\n", __func__, name.c_str());
            return 1;
        }

        // 获取.lora后缀之后的字符串作为lora_type
        std::string lora_type = name.substr(pos + lora_suffix.length());
        // 将文件名赋值给base_name
        std::string base_name = name;
        // 从字符串base_name中删除pos位置之后的所有字符
        base_name.erase(pos);
        
        // 如果model_tensors中找不到base_name，则输出错误信息并返回1
        if (model_tensors.find(base_name) == model_tensors.end()) {
            LLAMA_LOG_ERROR("%s: unknown tensor '%s' in lora adapter\n", __func__, name.data());
            return 1;
        }

        // 根据ftype的值选择对应的ggml类型
        ggml_type wtype;
        switch (ftype) {
            case 0: wtype = GGML_TYPE_F32;  break;
            case 1: wtype = GGML_TYPE_F16;  break;
            default:
                    {
                        // 如果ftype不是0或1，则输出错误信息并返回false
                        LLAMA_LOG_ERROR("%s: invalid tensor data type '%d'\n",
                                __func__, ftype);
                        return false;
                    }
        }
// 声明一个指向 ggml_tensor 结构体的指针 lora_tensor
ggml_tensor * lora_tensor;
// 如果数据维度为2，则创建一个2维的新的 ggml_tensor
if (n_dims == 2) {
    lora_tensor = ggml_new_tensor_2d(lora_ctx, wtype, ne[0], ne[1]);
}
// 如果数据维度不为2，则记录错误信息并返回1
else {
    LLAMA_LOG_ERROR("%s: unsupported tensor dimension %d\n", __func__, n_dims);
    return 1;
}
// 给 lora_tensor 设置名称为 "lora_tensor"
ggml_set_name(lora_tensor, "lora_tensor");

// 加载张量数据
// 获取当前文件读取位置
size_t offset = fin.tellg();
// 获取 lora_tensor 数据的大小
size_t tensor_data_size = ggml_nbytes(lora_tensor);
// 将 offset 向上取整到最近的32的倍数
offset = (offset + 31) & -32;
// 设置文件读取位置为 offset
fin.seekg(offset);
// 从文件中读取 tensor_data_size 大小的数据到 lora_tensor->data 中
fin.read((char*)lora_tensor->data, tensor_data_size);

// 将 lora_tensor 存入 lora_tensors 中，使用 name 作为键
lora_tensors[name] = lora_tensor;

// 检查是否同时有 A 和 B 张量，并应用
# 如果在 lora_tensors 中找到了 base_name + ".loraA" 和 base_name + ".loraB" 两个键
if (lora_tensors.find(base_name + ".loraA") != lora_tensors.end() &&
    lora_tensors.find(base_name + ".loraB") != lora_tensors.end()) {
    # 获取 model_tensors 中 base_name 对应的 ggml_tensor 对象
    ggml_tensor * dest_t = model_tensors[base_name];

    # 定义 offload_func 和 offload_func_force_inplace 两个函数指针，初始值为 ggml_offload_nop
    offload_func_t offload_func               = ggml_offload_nop;
    offload_func_t offload_func_force_inplace = ggml_offload_nop;

    # 如果定义了 GGML_USE_CUBLAS
    # 并且 dest_t 的 backend 是 GGML_BACKEND_GPU 或 GGML_BACKEND_GPU_SPLIT
    if (dest_t->backend == GGML_BACKEND_GPU || dest_t->backend == GGML_BACKEND_GPU_SPLIT) {
        # 如果 dest_t 的 type 不是 GGML_TYPE_F16，则抛出运行时错误
        if (dest_t->type != GGML_TYPE_F16) {
            throw std::runtime_error(format(
                "%s: error: the simultaneous use of LoRAs and GPU acceleration is only supported for f16 models. dest_t->type: %d", __func__, dest_t->type));
        }
        # 将 offload_func 和 offload_func_force_inplace 分别指向 ggml_cuda_assign_buffers 和 ggml_cuda_assign_buffers_force_inplace
        offload_func = ggml_cuda_assign_buffers;
        offload_func_force_inplace = ggml_cuda_assign_buffers_force_inplace;
    }

    # 定义指针 base_t
    ggml_tensor * base_t;
}
            // 如果存在模型加载器
            if (ml) {
                // 获取模型加载器的上下文
                struct gguf_context * ctx_gguf = ml->ctx_gguf;

                // 从基础模型加载数据
                if (gguf_find_tensor(ctx_gguf, base_name.c_str()) < 0) {
                    // 如果找不到基础模型中的张量，记录错误并返回
                    LLAMA_LOG_ERROR("%s: error: tensor '%s' not found in base model\n", __func__, base_name.c_str());
                    return 1;
                }

                // 创建基础张量，但是这部分代码可能没有经过测试，可能不起作用
                base_t = ml->create_tensor(base_ctx, base_name, { (uint32_t)dest_t->ne[0], (uint32_t)dest_t->ne[1] }, GGML_BACKEND_CPU);
                // 为基础张量加载数据
                ml->load_data_for(base_t);
            } else {
                // 如果没有模型加载器，则基础张量就是目标张量
                base_t = dest_t;
            }

            // 如果基础张量是量化的
            if (ggml_is_quantized(base_t->type)) {
                // 如果还没有警告过
                if (!warned) {
                    // 记录警告信息，说明使用量化模型可能会导致较差的质量
                    LLAMA_LOG_WARN("%s: warning: using a lora adapter with a quantized model may result in poor quality, "
// 如果使用的是 f16 或 f32 的基础模型，则发出警告
if (base_t->type != GGML_TYPE_F16 && base_t->type != GGML_TYPE_F32) {
    LLAMA_LOG_WARN("%s: 使用 f16 或 f32 的基础模型\n", __func__);
    warned = true;
}

// 获取 loraA 张量并设置其类型为 GGML_TYPE_F32
ggml_tensor * loraA = lora_tensors[base_name + ".loraA"];
GGML_ASSERT(loraA->type == GGML_TYPE_F32);
ggml_set_name(loraA, "loraA");

// 获取 loraB 张量并设置其类型为 GGML_TYPE_F32
ggml_tensor * loraB = lora_tensors[base_name + ".loraB"];
GGML_ASSERT(loraB->type == GGML_TYPE_F32);
ggml_set_name(loraB, "loraB");

// 检查基础模型和 loraA、loraB 张量的维度是否兼容，如果不兼容则返回错误
if (base_t->ne[0] != loraA->ne[1] || base_t->ne[1] != loraB->ne[1]) {
    LLAMA_LOG_ERROR("%s: 不兼容的张量维度 (%" PRId64 " and %" PRId64 ");"
                    " 你确定这个适配器适用于这个模型吗？\n", __func__, base_t->ne[0], loraA->ne[1]);
    return 1;
}

// 计算 w = w + BA*s
# 计算矩阵 loraA 和 loraB 的乘积，返回结果 BA
ggml_tensor * BA = ggml_mul_mat(lora_ctx, loraA, loraB);
# 将 BA 数据传输到指定位置
offload_func(BA);
# 给 BA 设置名称为 "BA"
ggml_set_name(BA, "BA");

# 如果 scaling 不等于 1.0f，则进行下面的操作
if (scaling != 1.0f) {
    # 创建一个新的浮点数张量 scale_tensor，值为 scaling
    ggml_tensor * scale_tensor = ggml_new_f32(lora_ctx, scaling);
    # 给 scale_tensor 设置名称为 "scale_tensor"
    ggml_set_name(scale_tensor, "scale_tensor");

    # 对 BA 进行就地缩放操作，使用 scale_tensor 进行缩放
    BA = ggml_scale_inplace(lora_ctx, BA, scale_tensor);
    # 将缩放后的 BA 数据传输到指定位置
    offload_func(BA);
    # 给缩放后的 BA 设置名称为 "BA_scaled"
    ggml_set_name(BA, "BA_scaled");
}

# 创建一个新的张量 r
ggml_tensor * r;
# 如果 base_t 等于 dest_t，则进行下面的操作
if (base_t == dest_t) {
    # 对 dest_t 进行就地加法操作，使用 BA 进行加法
    r = ggml_add_inplace(lora_ctx, dest_t, BA);
    # 强制将 r 数据传输到指定位置
    offload_func_force_inplace(r);
    # 给就地加法后的 r 设置名称为 "r_add_inplace"
    ggml_set_name(r, "r_add_inplace");
}
# 如果 base_t 不等于 dest_t，则进行下面的操作
else {
// 调用 ggml_add 函数，将结果保存在 r 中
r = ggml_add(lora_ctx, base_t, BA);
// 调用 offload_func 函数，处理 r
offload_func(r);
// 设置 r 的名称为 "r_add"
ggml_set_name(r, "r_add");

// 调用 ggml_cpy 函数，将结果保存在 r 中
r = ggml_cpy(lora_ctx, r, dest_t);
// 调用 offload_func 函数，处理 r
offload_func(r);
// 设置 r 的名称为 "r_cpy"
ggml_set_name(r, "r_cpy");

// 创建一个新的图形对象 gf
struct ggml_cgraph * gf = ggml_new_graph(lora_ctx);
// 构建前向扩展
ggml_build_forward_expand(gf, r);

// 使用 n_threads 线程计算图形对象 gf
ggml_graph_compute_helper(work_buffer, gf, n_threads);

// 不再需要这些张量，重置上下文以节省内存
ggml_free(lora_ctx);
// 重新初始化上下文
lora_ctx = ggml_init(params);
// 清空张量列表
lora_tensors.clear();

// 张量数量加一
n_tensors++;
    // 如果张量数量能被4整除，打印一个点
    if (n_tensors % 4 == 0) {
        LLAMA_LOG_INFO(".");
    }
}

// TODO: 这应该在析构函数中，否则在失败时会造成内存泄漏
ggml_free(lora_ctx);
// 如果base_ctx存在，释放其内存
if (base_ctx) {
    ggml_free(base_ctx);
}

// 计算LoRa操作所花费的时间
const int64_t t_lora_us = ggml_time_us() - t_start_lora_us;
// 打印LoRa操作所花费的时间
LLAMA_LOG_INFO(" done (%.2f ms)\n", t_lora_us / 1000.0);

// 返回0表示成功
return 0;
}

//
// 接口实现
// 定义一个名为 llama_model_params 的结构体，并初始化默认参数
struct llama_model_params llama_model_default_params() {
    // 初始化结构体 result，并设置默认参数值
    struct llama_model_params result = {
        /*.n_gpu_layers                =*/ 0,  // GPU 层数，默认为 0
        /*.main_gpu                    =*/ 0,  // 主 GPU 编号，默认为 0
        /*.vram_budget_gb              =*/ -1.0,  // VRAM 预算，单位为 GB，默认为 -1.0
        /*.tensor_split                =*/ nullptr,  // 张量分割，默认为空指针
        /*.progress_callback           =*/ nullptr,  // 进度回调函数，默认为空指针
        /*.progress_callback_user_data =*/ nullptr,  // 进度回调函数的用户数据，默认为空指针
        /*.vocab_only                  =*/ false,  // 仅使用词汇表，默认为 false
        /*.use_mmap                    =*/ true,  // 使用内存映射，默认为 true
        /*.use_mlock                   =*/ false,  // 使用内存锁定，默认为 false
    };

    // 如果定义了 GGML_USE_METAL，则设置 n_gpu_layers 为 1
#ifdef GGML_USE_METAL
    result.n_gpu_layers = 1;
#endif

    // 返回初始化后的结构体 result
    return result;
}
// 返回默认的 Llama 上下文参数
struct llama_context_params llama_context_default_params() {
    // 初始化 Llama 上下文参数结构体
    struct llama_context_params result = {
        /*.seed                        =*/ LLAMA_DEFAULT_SEED,  // 设置默认种子
        /*.n_ctx                       =*/ 512,  // 设置默认上下文数
        /*.n_batch                     =*/ 512,  // 设置默认批处理数
        /*.n_threads                   =*/ GGML_DEFAULT_N_THREADS, // 设置默认线程数
        /*.n_threads_batch             =*/ GGML_DEFAULT_N_THREADS,  // 设置默认批处理线程数
        /*.rope_scaling_type           =*/ LLAMA_ROPE_SCALING_UNSPECIFIED,  // 设置默认绳索缩放类型
        /*.rope_freq_base              =*/ 0.0f,  // 设置默认绳索频率基数
        /*.rope_freq_scale             =*/ 0.0f,  // 设置默认绳索频率比例
        /*.yarn_ext_factor             =*/ -1.0f,  // 设置默认纱线扩展因子
        /*.yarn_attn_factor            =*/ 1.0f,  // 设置默认纱线注意力因子
        /*.yarn_beta_fast              =*/ 32.0f,  // 设置默认纱线快速 beta
        /*.yarn_beta_slow              =*/ 1.0f,  // 设置默认纱线慢速 beta
        /*.yarn_orig_ctx               =*/ 0,  // 设置默认纱线原始上下文
        /*.mul_mat_q                   =*/ true,  // 设置默认矩阵乘法 q
        /*.f16_kv                      =*/ true,  // 设置默认 f16 kv
        /*.logits_all                  =*/ false,  // 设置默认所有对数
        /*.embedding                   =*/ false,  // 设置默认嵌入
// 返回默认的量化参数结构体
struct llama_model_quantize_params llama_model_quantize_default_params() {
    // 初始化默认的量化参数结构体
    struct llama_model_quantize_params result = {
        /*.nthread                     =*/ 0,  // 线程数，默认为0
        /*.ftype                       =*/ LLAMA_FTYPE_MOSTLY_Q5_1,  // 文件类型，默认为LLAMA_FTYPE_MOSTLY_Q5_1
        /*.allow_requantize            =*/ false,  // 是否允许重新量化，默认为false
        /*.quantize_output_tensor      =*/ true,  // 是否量化输出张量，默认为true
        /*.only_copy                   =*/ false,  // 是否仅复制，默认为false
        /*.pure                        =*/ false,  // 是否纯净，默认为false
    };

    return result;  // 返回初始化后的结构体
}

// 返回最大设备数
int llama_max_devices(void) {
    return LLAMA_MAX_DEVICES;  // 返回预定义的最大设备数
}
}

// 检查是否支持内存映射
bool llama_mmap_supported(void) {
    return llama_mmap::SUPPORTED;
}

// 检查是否支持内存锁定
bool llama_mlock_supported(void) {
    return llama_mlock::SUPPORTED;
}

// 初始化 Llama 后端
void llama_backend_init(bool numa) {
    // 初始化 ggml 时间
    ggml_time_init();

    // 初始化 f16 表格所需
    {
        // 初始化参数
        struct ggml_init_params params = { 0, NULL, false };
        // 初始化 ggml 上下文
        struct ggml_context * ctx = ggml_init(params);
        // 释放 ggml 上下文
        ggml_free(ctx);
    }
// 如果 numa 存在，则初始化 numa
if (numa) {
    ggml_numa_init();
}

// 如果定义了 GGML_USE_MPI，则初始化 MPI 后端
#ifdef GGML_USE_MPI
    ggml_mpi_backend_init();
#endif
}

// 释放 llama 后端资源
void llama_backend_free(void) {
#ifdef GGML_USE_MPI
    ggml_mpi_backend_free();
#endif
}

// 返回当前时间的微秒数
int64_t llama_time_us(void) {
    return ggml_time_us();
}

// 从文件加载模型到 llama_model 结构体
struct llama_model * llama_load_model_from_file(
// 初始化时间
ggml_time_init();

// 创建一个 llama_model 对象
llama_model * model = new llama_model;

// 当进度回调函数为空时，设置默认的进度回调函数
unsigned cur_percentage = 0;
if (params.progress_callback == NULL) {
    // 设置进度回调函数的用户数据为当前进度的指针
    params.progress_callback_user_data = &cur_percentage;
    // 设置进度回调函数为一个 lambda 表达式，用于更新进度并输出日志信息
    params.progress_callback = [](float progress, void * ctx) {
        unsigned * cur_percentage_p = (unsigned *) ctx;
        unsigned percentage = (unsigned) (100 * progress);
        while (percentage > *cur_percentage_p) {
            *cur_percentage_p = percentage;
            LLAMA_LOG_INFO(".");
            if (percentage >= 100) {
                LLAMA_LOG_INFO("\n");
            }
        }
    };
    }

    // 如果模型加载失败，则记录错误信息并删除模型，然后返回空指针
    if (!llama_model_load(path_model, *model, params)) {
        LLAMA_LOG_ERROR("%s: failed to load model\n", __func__);
        delete model;
        return nullptr;
    }

    // 返回加载成功的模型
    return model;
}

// 释放模型内存
void llama_free_model(struct llama_model * model) {
    delete model;
}

// 创建带有模型的新上下文
struct llama_context * llama_new_context_with_model(
                 struct llama_model * model,
        struct llama_context_params   params) {

    // 如果模型为空，则返回空指针
    if (!model) {
    // 如果模型为空指针，则返回空指针
    return nullptr;
    // 创建一个新的llama_context对象，使用给定的模型作为参数
    llama_context * ctx = new llama_context(*model);
    // 获取模型的超参数
    const auto & hparams = model->hparams;
    // 获取上下文参数
    auto       & cparams = ctx->cparams;
    // 设置上下文参数的值为给定参数的值
    cparams.n_batch          = params.n_batch;
    cparams.n_threads        = params.n_threads;
    cparams.n_threads_batch  = params.n_threads_batch;
    cparams.yarn_ext_factor  = params.yarn_ext_factor;
    cparams.yarn_attn_factor = params.yarn_attn_factor;
    cparams.yarn_beta_fast   = params.yarn_beta_fast;
    cparams.yarn_beta_slow   = params.yarn_beta_slow;
    cparams.mul_mat_q        = params.mul_mat_q;
    // 如果给定参数的上下文大小为0，则使用模型的训练上下文大小，否则使用给定参数的上下文大小
    cparams.n_ctx            = params.n_ctx           == 0    ? hparams.n_ctx_train           : params.n_ctx;
    // 如果给定参数的绳索基础频率为0.0，则使用模型的训练绳索基础频率，否则使用给定参数的绳索基础频率
    cparams.rope_freq_base   = params.rope_freq_base  == 0.0f ? hparams.rope_freq_base_train  : params.rope_freq_base;
    // 如果给定参数的绳索频率缩放为0.0，则使用模型的训练绳索频率缩放，否则使用给定参数的绳索频率缩放
    cparams.rope_freq_scale  = params.rope_freq_scale == 0.0f ? hparams.rope_freq_scale_train : params.rope_freq_scale;
# 设置 cparams.n_yarn_orig_ctx 的值，如果 params.yarn_orig_ctx 不为 0，则使用它，否则使用 hparams.n_yarn_orig_ctx，如果都为 0，则使用 hparams.n_ctx_train
cparams.n_yarn_orig_ctx  = params.yarn_orig_ctx    != 0 ? params.yarn_orig_ctx    :
                           hparams.n_yarn_orig_ctx != 0 ? hparams.n_yarn_orig_ctx :
                                                          hparams.n_ctx_train;

# 设置 rope_scaling_type 的值，如果为 LLAMA_ROPE_SCALING_UNSPECIFIED，则使用 hparams.rope_scaling_type_train
auto rope_scaling_type = params.rope_scaling_type;
if (rope_scaling_type == LLAMA_ROPE_SCALING_UNSPECIFIED) {
    rope_scaling_type = hparams.rope_scaling_type_train;
}

# 如果 rope_scaling_type 为 LLAMA_ROPE_SCALING_NONE，则将 cparams.rope_freq_scale 设置为 1.0f，表示不进行频率缩放
if (rope_scaling_type == LLAMA_ROPE_SCALING_NONE) {
    cparams.rope_freq_scale = 1.0f; // never scale if scaling type is none
}

# 如果 cparams.yarn_ext_factor 小于 0.0f，则将其设置为 1.0f 或 0.0f，取决于 rope_scaling_type 是否为 LLAMA_ROPE_SCALING_YARN
if (cparams.yarn_ext_factor < 0.0f) { // negative indicates 'not set'
    cparams.yarn_ext_factor = rope_scaling_type == LLAMA_ROPE_SCALING_YARN ? 1.0f : 0.0f;
}

# 如果 params.seed 为 LLAMA_DEFAULT_SEED，则将其设置为当前时间的时间戳
if (params.seed == LLAMA_DEFAULT_SEED) {
    params.seed = time(NULL);
}
    }

    // 输出上下文信息
    LLAMA_LOG_INFO("%s: n_ctx      = %u\n",     __func__, cparams.n_ctx);
    // 输出频率基数信息
    LLAMA_LOG_INFO("%s: freq_base  = %.1f\n",   __func__, cparams.rope_freq_base);
    // 输出频率缩放信息
    LLAMA_LOG_INFO("%s: freq_scale = %g\n",     __func__, cparams.rope_freq_scale);

    // 使用参数中的种子初始化随机数生成器
    ctx->rng = std::mt19937(params.seed);
    // 将参数中的所有对数存储到上下文中
    ctx->logits_all = params.logits_all;

    // 根据参数中的 f16_kv 决定内存类型
    ggml_type memory_type = params.f16_kv ? GGML_TYPE_F16 : GGML_TYPE_F32;

    // 为上下文缓冲区保留内存空间
    if (!hparams.vocab_only) {
        // 如果不仅仅是词汇表，则初始化自注意力缓存
        if (!llama_kv_cache_init(ctx->model.hparams, ctx->kv_self, memory_type, cparams.n_ctx, model->n_gpu_layers)) {
            // 如果初始化失败，则输出错误信息，释放内存并返回空指针
            LLAMA_LOG_ERROR("%s: llama_kv_cache_init() failed for self-attention cache\n", __func__);
            llama_free(ctx);
            return nullptr;
        }

        {
        // 计算内存大小，包括键和值的大小
        const size_t memory_size = ggml_nbytes(ctx->kv_self.k) + ggml_nbytes(ctx->kv_self.v);
        // 打印键值对的内存大小
        LLAMA_LOG_INFO("%s: kv self size  = %7.2f MB\n", __func__, memory_size / 1024.0 / 1024.0);
    }

    // 在推断期间调整大小
    if (params.logits_all) {
        // 如果需要存储所有logits，则预留足够的空间
        ctx->logits.reserve(cparams.n_ctx*hparams.n_vocab);
    } else {
        // 否则只预留vocab大小的空间
        ctx->logits.reserve(hparams.n_vocab);
    }

    if (params.embedding){
        // 如果需要embedding，则调整embedding的大小
        ctx->embedding.resize(hparams.n_embd);
    }

    {
        // 张量对齐的大小
        static const size_t tensor_alignment = 32;
        // 计算缓冲区的大小，用于存储张量和图结构，同时分配器缓冲区用于存储张量数据
        ctx->buf_compute.resize(ggml_tensor_overhead()*LLAMA_MAX_NODES + ggml_graph_overhead());
// 创建度量分配器
ctx->alloc = ggml_allocr_new_measure(tensor_alignment);

// 构建最坏情况下的图形
int n_tokens = (int)std::min(cparams.n_ctx, cparams.n_batch);
int n_past = cparams.n_ctx - n_tokens;
llama_token token = llama_token_bos(&ctx->model); // 实际上 llama_build_graph 不使用，但 llama_build_graph 需要选择 token 和嵌入输入图形
ggml_cgraph * gf = llama_build_graph(*ctx, llama_batch_get_one(&token, n_tokens, n_past, 0));

#ifdef GGML_USE_METAL
if (model->n_gpu_layers > 0) {
    // 设置 Metal 日志回调函数
    ggml_metal_log_set_callback(llama_log_callback_default, NULL);

    // 初始化 Metal 上下文
    ctx->ctx_metal = ggml_metal_init(1);
    if (!ctx->ctx_metal) {
        // Metal 初始化失败时输出错误信息
        LLAMA_LOG_ERROR("%s: ggml_metal_init() failed\n", __func__);
        // 释放上下文并返回空指针
        llama_free(ctx);
        return NULL;
    }
    // 查找 Metal 图形并发性
    // ggml_metal_graph_find_concurrency(ctx->ctx_metal, gf, false);
            }
#endif
            // 测量图形的内存需求
            size_t alloc_size = ggml_allocr_alloc_graph(ctx->alloc, gf) + tensor_alignment;

            // 记录计算缓冲区的总大小
            LLAMA_LOG_INFO("%s: compute buffer total size = %.2f MB\n", __func__, (ctx->buf_compute.size + alloc_size) / 1024.0 / 1024.0);

            // 根据准确的内存需求重新创建分配器
            ggml_allocr_free(ctx->alloc);

            // 调整分配器的缓冲区大小
            ctx->buf_alloc.resize(alloc_size);
            // 创建新的分配器
            ctx->alloc = ggml_allocr_new(ctx->buf_alloc.data, ctx->buf_alloc.size, tensor_alignment);
#ifdef GGML_USE_METAL
            if (ctx->ctx_metal) {
                //ggml_allocr_set_parse_seq(ctx->alloc, ggml_metal_get_concur_list(ctx->ctx_metal), ggml_metal_if_optimized(ctx->ctx_metal));
            }
#endif
#ifdef GGML_USE_CUBLAS
            // 设置CUDA的临时内存大小
            ggml_cuda_set_scratch_size(alloc_size);
// 记录日志信息，输出 VRAM 临时缓冲区的大小
LLAMA_LOG_INFO("%s: VRAM scratch buffer: %.2f MB\n", __func__, alloc_size / 1024.0 / 1024.0);

// 计算总的 VRAM 使用量
auto add_tensor = [](const ggml_tensor * t, size_t & size) {
    // 如果张量的后端是 GPU 或者 GPU 分割，则计算其占用的内存大小并加到总大小中
    if (t->backend == GGML_BACKEND_GPU || t->backend == GGML_BACKEND_GPU_SPLIT) {
        size += ggml_nbytes(t);
    }
};
size_t model_vram_size = 0;
// 遍历模型中的张量，计算其占用的内存大小并加到总大小中
for (const auto & kv : model->tensors_by_name) {
    add_tensor(kv.second, model_vram_size);
}

size_t kv_vram_size = 0;
// 计算键值对张量占用的内存大小并加到总大小中
add_tensor(ctx->kv_self.k, kv_vram_size);
add_tensor(ctx->kv_self.v, kv_vram_size);

// 计算上下文占用的内存大小，包括分配的大小和键值对张量的大小
size_t ctx_vram_size = alloc_size + kv_vram_size;
// 计算总的 VRAM 使用量，包括模型占用的内存和上下文占用的内存
size_t total_vram_size = model_vram_size + ctx_vram_size;
// 输出日志信息，显示总的 VRAM 使用情况，包括模型使用的 VRAM 和上下文使用的 VRAM
LLAMA_LOG_INFO("%s: total VRAM used: %.2f MB (model: %.2f MB, context: %.2f MB)\n", __func__,
                total_vram_size / 1024.0 / 1024.0,
                model_vram_size / 1024.0 / 1024.0,
                ctx_vram_size / 1024.0 / 1024.0);
// 如果使用 Metal
#ifdef GGML_USE_METAL
        if (model->n_gpu_layers > 0) {
            // 分配所有 Metal 资源和内存缓冲区
            void * data_ptr  = NULL;
            size_t data_size = 0;
            // 如果模型的映射存在，则使用映射的地址和大小
            if (ctx->model.mapping) {
                data_ptr  = ctx->model.mapping->addr;
                data_size = ctx->model.mapping->size;
            } else {
                // 否则，使用 ggml_get_mem_buffer 和 ggml_get_mem_size 获取内存缓冲区的地址和大小
                data_ptr  = ggml_get_mem_buffer(ctx->model.ctx);
                data_size = ggml_get_mem_size  (ctx->model.ctx);
// 定义最大尺寸为上下文模型的最大张量尺寸
const size_t max_size = ggml_get_max_tensor_size(ctx->model.ctx);

// 打印最大张量尺寸的日志信息
LLAMA_LOG_INFO("%s: max tensor size = %8.2f MB\n", __func__, max_size/1024.0/1024.0);

// 定义宏LLAMA_METAL_CHECK_BUF，用于检查Metal上下文中是否成功添加缓冲区
#define LLAMA_METAL_CHECK_BUF(result)                            
            if (!(result)) {                                             
                // 如果添加缓冲区失败，则打印错误日志信息，释放上下文并返回空指针
                LLAMA_LOG_ERROR("%s: failed to add buffer\n", __func__); 
                llama_free(ctx);                                         
                return NULL;                                             
            }

// 使用LLAMA_METAL_CHECK_BUF宏检查并添加"data"缓冲区
LLAMA_METAL_CHECK_BUF(ggml_metal_add_buffer(ctx->ctx_metal, "data",  data_ptr, data_size, max_size));
// 使用LLAMA_METAL_CHECK_BUF宏检查并添加"kv"缓冲区
LLAMA_METAL_CHECK_BUF(ggml_metal_add_buffer(ctx->ctx_metal, "kv",    ctx->kv_self.buf.data, ctx->kv_self.buf.size, 0));
// 使用LLAMA_METAL_CHECK_BUF宏检查并添加"alloc"缓冲区
LLAMA_METAL_CHECK_BUF(ggml_metal_add_buffer(ctx->ctx_metal, "alloc", ctx->buf_alloc.data, ctx->buf_alloc.size, 0));
// 取消LLAMA_METAL_CHECK_BUF宏的定义
#undef LLAMA_METAL_CHECK_BUF
#ifdef GGML_USE_MPI
    // 如果使用了 MPI，则初始化 MPI 上下文
    ctx->ctx_mpi = ggml_mpi_init();

    // 如果当前进程的 MPI 排名大于 0
    if (ggml_mpi_rank(ctx->ctx_mpi) > 0) {
        // 进入一个阻塞的评估循环，使用虚拟输入，让排名为 0 的进程驱动整个过程
        // TODO: 在 #3228 之后需要修复
        GGML_ASSERT(false && "not implemented");
        //const std::vector<llama_token> tmp(ctx->model.hparams.n_ctx, llama_token_bos(ctx));
        //while (!llama_eval(ctx, tmp.data(), tmp.size(), 0, 0)) {};
        // 释放 LLAMA 后端资源
        llama_backend_free();
        // 退出程序
        exit(1);
    }
#endif

    // 释放 LLAMA 上下文资源
    return ctx;
}

// 释放 LLAMA 上下文资源
void llama_free(struct llama_context * ctx) {
    delete ctx;
```

// 返回指向上下文中模型的指针
const llama_model * llama_get_model(const struct llama_context * ctx) {
    return &ctx->model;
}

// 返回上下文中的上下文数量
int llama_n_ctx(const struct llama_context * ctx) {
    return ctx->cparams.n_ctx;
}

// 返回模型的词汇表类型
enum llama_vocab_type llama_vocab_type(const struct llama_model * model) {
    return model->vocab.type;
}

// 返回模型是否使用稀疏推断
bool llama_use_sparse_inference(const struct llama_model * model) {
    return model->sparse_deriv == GGML_SPARSE_INFERENCE;
}

// 返回模型的词汇表大小
int llama_n_vocab(const struct llama_model * model) {
    return model->vocab.id_to_token.size();
}
// 返回模型的上下文训练数量
int llama_n_ctx_train(const struct llama_model * model) {
    return model->hparams.n_ctx_train;
}

// 返回模型的嵌入维度
int llama_n_embd(const struct llama_model * model) {
    return model->hparams.n_embd;
}

// 返回模型的绳索频率比例（用于训练）
float llama_rope_freq_scale_train(const struct llama_model * model) {
    return model->hparams.rope_freq_scale_train;
}

// 将模型的架构名称、类型名称和特征类型名称格式化为字符串，存储到buf中
int llama_model_desc(const struct llama_model * model, char * buf, size_t buf_size) {
    return snprintf(buf, buf_size, "%s %s %s",
            llama_model_arch_name(model->arch).c_str(),
            llama_model_type_name(model->type),
            llama_model_ftype_name(model->ftype).c_str());
}
# 计算LLAMA模型的大小，遍历模型中的张量并累加它们的字节数
uint64_t llama_model_size(const struct llama_model * model) {
    uint64_t size = 0;
    for (const auto & it : model->tensors_by_name) {
        size += ggml_nbytes(it.second);
    }
    return size;
}

# 计算LLAMA模型的参数数量，遍历模型中的张量并累加它们的元素数量
uint64_t llama_model_n_params(const struct llama_model * model) {
    uint64_t nparams = 0;
    for (const auto & it : model->tensors_by_name) {
        nparams += ggml_nelements(it.second);
    }
    return nparams;
}

# 获取LLAMA模型中指定名称的张量
struct ggml_tensor * llama_get_model_tensor(struct llama_model * model, const char * name) {
    return ggml_get_tensor(model->ctx, name);
}
# 对输入的模型进行量化处理，将结果保存到指定文件中
int llama_model_quantize(
        const char * fname_inp,  # 输入模型文件名
        const char * fname_out,  # 输出模型文件名
        const llama_model_quantize_params * params) {  # 量化参数
    try:
        llama_model_quantize_internal(fname_inp, fname_out, params)  # 调用内部函数进行模型量化处理
        return 0  # 处理成功，返回0
    catch (const std::exception & err):  # 捕获异常
        LLAMA_LOG_ERROR("%s: failed to quantize: %s\n", __func__, err.what())  # 记录错误日志
        return 1  # 处理失败，返回1
    }
}

# 从文件中加载 LORA 适配器并应用到指定的上下文中
int llama_apply_lora_from_file(struct llama_context * ctx, const char * path_lora, float scale, const char * path_base_model, int n_threads) {
    try:
        return llama_apply_lora_from_file_internal(ctx->model, path_lora, scale, path_base_model, n_threads)  # 调用内部函数加载并应用 LORA 适配器
    catch (const std::exception & err):  # 捕获异常
        LLAMA_LOG_ERROR("%s: failed to apply lora adapter: %s\n", __func__, err.what())  # 记录错误日志
        return 1  # 处理失败，返回1
    }
// 从文件中应用 LORA 模型
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

// 从文件中应用 GPU 索引模型
int llama_model_apply_gpu_idx_from_file(struct llama_model * model, const char * path_mlp, bool use_mmap) {
    // 创建 GPU 分割加载器对象
    llama_gpu_split_loader * mlp_ml = new llama_gpu_split_loader(path_mlp, use_mmap);
    // 如果应用张量到基础模型失败，则记录错误信息并返回
    if (mlp_ml -> apply_tensors_to_base_model(model) > 0) {
        LLAMA_LOG_ERROR("%s: failed to apply gpu split\n", __func__);
        return 1;
    }
    // 将加载器对象设置为模型的唯一指针
    model -> mlp_model_loader = std::unique_ptr<llama_gpu_split_loader>(mlp_ml);
    return 0;
}
// 将模型的前馈神经网络拆分并卸载到LLAMA模型中
size_t llama_model_offload_ffn_split(struct llama_model * model) {
    // 创建LLAMA增强模型加载器对象
    llama_augmentation_model_loader * aug_ml = new llama_augmentation_model_loader(model);    
    // 将前馈神经网络拆分并卸载到LLAMA模型中，返回卸载的字节数
    size_t offloaded_bytes = aug_ml->offload_ffn_split(model);
    // 返回卸载的字节数
    return offloaded_bytes;
}

// 获取LLAMA上下文中键值缓存的令牌数量
int llama_get_kv_cache_token_count(const struct llama_context * ctx) {
    // 返回键值缓存中的令牌数量
    return ctx->kv_self.head;
}

// 清空LLAMA键值缓存
void llama_kv_cache_clear(struct llama_context * ctx) {
    // 清空LLAMA上下文中的键值缓存
    llama_kv_cache_clear(ctx->kv_self);
}

// 从LLAMA键值缓存中按序删除数据
void llama_kv_cache_seq_rm(struct llama_context * ctx, llama_seq_id seq_id, llama_pos p0, llama_pos p1) {
    // 从LLAMA上下文中的键值缓存中按序删除数据
    llama_kv_cache_seq_rm(ctx->kv_self, seq_id, p0, p1);
}
// 将一个序列的键值对缓存复制到另一个序列的键值对缓存中
void llama_kv_cache_seq_cp(struct llama_context * ctx, llama_seq_id seq_id_src, llama_seq_id seq_id_dst, llama_pos p0, llama_pos p1) {
    // 如果源序列和目标序列相同，则直接返回，不进行复制操作
    if (seq_id_src == seq_id_dst) {
        return;
    }
    // 调用函数将源序列的键值对缓存复制到目标序列的键值对缓存中
    llama_kv_cache_seq_cp(ctx->kv_self, seq_id_src, seq_id_dst, p0, p1);
}

// 保持一个序列的键值对缓存
void llama_kv_cache_seq_keep(struct llama_context * ctx, llama_seq_id seq_id) {
    // 调用函数保持一个序列的键值对缓存
    llama_kv_cache_seq_keep(ctx->kv_self, seq_id);
}

// 移动一个序列的键值对缓存中的数据
void llama_kv_cache_seq_shift(struct llama_context * ctx, llama_seq_id seq_id, llama_pos p0, llama_pos p1, llama_pos delta) {
    // 调用函数移动一个序列的键值对缓存中的数据
    llama_kv_cache_seq_shift(ctx->kv_self, seq_id, p0, p1, delta);
}

// 返回状态的最大大小
size_t llama_get_state_size(const struct llama_context * ctx) {
    // 我们不知道随机数生成器的大小，直到我们实际对其进行序列化。因此，为其序列化状态保留比足够大的内存。
    // 作为参考，std::mt19937(1337) 序列化为 6701 字节。
    const size_t s_rng_size        = sizeof(size_t);
    // 定义并初始化各个变量的大小
    const size_t s_rng             = LLAMA_MAX_RNG_STATE; // 定义随机数生成器状态的大小
    const size_t s_logits_capacity = sizeof(size_t); // 定义logits容量的大小
    const size_t s_logits_size     = sizeof(size_t); // 定义logits大小的大小
    const size_t s_logits          = ctx->logits.capacity() * sizeof(float); // 定义logits数据的大小
    const size_t s_embedding_size  = sizeof(size_t); // 定义embedding大小的大小
    const size_t s_embedding       = ctx->embedding.size() * sizeof(float); // 定义embedding数据的大小
    const size_t s_kv_size         = sizeof(size_t); // 定义kv大小的大小
    const size_t s_kv_ntok         = sizeof(int); // 定义kv ntok的大小
    const size_t s_kv              = ctx->kv_self.buf.size; // 定义kv数据的大小

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
// 函数定义，计算字符串中 's' 的个数
int count_s(const char *str) {
    int s_total = 0; // 初始化 's' 的总数为 0
    while (*str) { // 遍历字符串
        if (*str == 's') { // 如果当前字符是 's'
            s_total++; // 's' 的总数加一
        }
        str++; // 指针移动到下一个字符
    }
    return s_total; // 返回 's' 的总数
}

// 结构体定义，用于处理 llama 数据的上下文
struct llama_data_context {
    virtual void write(const void * src, size_t size) = 0; // 纯虚函数，用于写入数据
    virtual size_t get_size_written() = 0; // 纯虚函数，用于获取已写入数据的大小
    virtual ~llama_data_context() = default; // 虚析构函数，默认实现
};

// 结构体定义，继承自 llama_data_context，用于处理 llama 数据的缓冲区上下文
struct llama_data_buffer_context : llama_data_context {
    uint8_t * ptr; // 指向数据缓冲区的指针
    size_t size_written = 0; // 已写入数据的大小，默认为 0

    llama_data_buffer_context(uint8_t * p) : ptr(p) {} // 构造函数，初始化指针

    void write(const void * src, size_t size) override { // 实现父类纯虚函数，用于写入数据
    // 将源地址的数据复制到目标地址，复制大小为size
    memcpy(ptr, src, size);
    // 指针移动到下一个位置
    ptr += size;
    // 记录已写入的数据大小
    size_written += size;
}

// 返回已写入的数据大小
size_t get_size_written() override {
    return size_written;
}

// 定义llama_data_file_context结构体，继承自llama_data_context
struct llama_data_file_context : llama_data_context {
    // 文件指针
    llama_file * file;
    // 已写入的数据大小
    size_t size_written = 0;

    // 构造函数，初始化文件指针
    llama_data_file_context(llama_file * f) : file(f) {}

    // 写入数据到文件
    void write(const void * src, size_t size) override {
        // 调用文件对象的write_raw方法写入数据
        file->write_raw(src, size);
        // 更新已写入的数据大小
        size_written += size;
    }
// 重写函数，返回已写入的大小
size_t get_size_written() override {
    return size_written;
}

/** 将状态数据复制到传入的上下文中的缓冲区或文件中，取决于传入的上下文类型
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
    // 复制随机数生成器状态
    {
        // 创建一个字符串流对象rng_ss，并将ctx->rng的内容写入其中
        std::stringstream rng_ss;
        rng_ss << ctx->rng;

        // 获取字符串流的大小
        const size_t rng_size = rng_ss.str().size();
        // 创建一个字符数组rng_buf，用于存储随机数生成器状态
        char rng_buf[LLAMA_MAX_RNG_STATE];

        // 将rng_buf数组清零
        memset(&rng_buf[0], 0, LLAMA_MAX_RNG_STATE);
        // 将rng_ss中的内容复制到rng_buf数组中
        memcpy(&rng_buf[0], rng_ss.str().data(), rng_ss.str().size());

        // 将rng_size写入数据上下文
        data_ctx->write(&rng_size,   sizeof(rng_size));
        // 将rng_buf数组写入数据上下文
        data_ctx->write(&rng_buf[0], LLAMA_MAX_RNG_STATE);
    }

    // 复制logits
    {
        // 获取logits的容量和大小
        const size_t logits_cap  = ctx->logits.capacity();
        const size_t logits_size = ctx->logits.size();
        // 将logits_cap的值写入数据流
        data_ctx->write(&logits_cap,  sizeof(logits_cap));
        // 将logits_size的值写入数据流
        data_ctx->write(&logits_size, sizeof(logits_size);

        // 如果logits_size不为0，将ctx->logits中的数据写入数据流
        if (logits_size) {
            data_ctx->write(ctx->logits.data(), logits_size * sizeof(float));
        }

        // 如果容量和大小之间有间隙，写入填充数据
        size_t padding_size = (logits_cap - logits_size) * sizeof(float);
        if (padding_size > 0) {
            // 创建一个填充了零的缓冲区
            std::vector<uint8_t> padding(padding_size, 0);
            data_ctx->write(padding.data(), padding_size);
        }
    }

    // 复制嵌入
    {
        // 获取嵌入的大小
        const size_t embedding_size = ctx->embedding.size();

        // 将嵌入的大小写入数据流
        data_ctx->write(&embedding_size, sizeof(embedding_size));
        // 如果嵌入大小不为零，则将嵌入数据写入上下文
        if (embedding_size) {
            data_ctx->write(ctx->embedding.data(), embedding_size * sizeof(float));
        }
    }

    // 复制 kv 缓存
    {
        // 获取上下文中的 kv_self、hparams 和 cparams
        const auto & kv_self = ctx->kv_self;
        const auto & hparams = ctx->model.hparams;
        const auto & cparams = ctx->cparams;

        // 获取层数、嵌入大小和上下文大小
        const auto   n_layer = hparams.n_layer;
        const auto   n_embd  = hparams.n_embd_gqa();
        const auto   n_ctx   = cparams.n_ctx;

        // 获取 kv 缓存的大小、头部位置和大小
        const size_t   kv_buf_size = kv_self.buf.size;
        const uint32_t kv_head     = kv_self.head;
        const uint32_t kv_size     = kv_self.size;
        // 将kv_buf_size的值写入数据流
        data_ctx->write(&kv_buf_size, sizeof(kv_buf_size));
        // 将kv_head的值写入数据流
        data_ctx->write(&kv_head,     sizeof(kv_head));
        // 将kv_size的值写入数据流
        data_ctx->write(&kv_size,     sizeof(kv_size));

        // 如果kv_buf_size不为0，则执行以下操作
        if (kv_buf_size) {
            // 计算元素大小
            const size_t elt_size = ggml_element_size(kv_self.k);
            // 初始化一个上下文cpy_ctx
            ggml_context * cpy_ctx = ggml_init({ 6*ggml_tensor_overhead() + ggml_graph_overhead(), NULL, /* no_alloc */ true });
            // 创建一个新的图gf
            ggml_cgraph * gf = ggml_new_graph(cpy_ctx);

            // 创建一个3维张量kout3d
            ggml_tensor * kout3d = ggml_new_tensor_3d(cpy_ctx, kv_self.k->type, n_embd, kv_head, n_layer);
            // 初始化一个存储kout3d数据的向量
            std::vector<uint8_t> kout3d_data(ggml_nbytes(kout3d), 0);
            // 将kout3d的数据指针指向kout3d_data的数据
            kout3d->data = kout3d_data.data();

            // 创建一个3维张量vout3d
            ggml_tensor * vout3d = ggml_new_tensor_3d(cpy_ctx, kv_self.v->type, kv_head, n_embd, n_layer);
            // 初始化一个存储vout3d数据的向量
            std::vector<uint8_t> vout3d_data(ggml_nbytes(vout3d), 0);
            // 将vout3d的数据指针指向vout3d_data的数据
            vout3d->data = vout3d_data.data();

            // 创建一个3维张量k3d，将kv_self.k的数据视图转换为3维
            ggml_tensor * k3d = ggml_view_3d(cpy_ctx, kv_self.k,
                n_embd, kv_head, n_layer,
// 创建一个3D视图，将kv_self.v的数据复制到k3d中
ggml_tensor * k3d = ggml_view_3d(cpy_ctx, kv_self.v, kv_head, n_embd, n_layer, elt_size*n_embd, elt_size*n_embd*n_ctx, 0);

// 创建一个3D视图，将kv_self.v的数据复制到v3d中
ggml_tensor * v3d = ggml_view_3d(cpy_ctx, kv_self.v, kv_head, n_embd, n_layer, elt_size*n_ctx, elt_size*n_ctx*n_embd, 0);

// 构建前向扩展，将k3d的数据复制到kout3d中
ggml_build_forward_expand(gf, ggml_cpy(cpy_ctx, k3d, kout3d));

// 构建前向扩展，将v3d的数据复制到vout3d中
ggml_build_forward_expand(gf, ggml_cpy(cpy_ctx, v3d, vout3d));

// 计算图的计算辅助函数
ggml_graph_compute_helper(ctx->work_buffer, gf, /*n_threads*/ 1);

// 释放cpy_ctx
ggml_free(cpy_ctx);

// 数据现在在kout3d_data和vout3d_data缓冲区中
// 将它们写入文件
data_ctx->write(kout3d_data.data(), kout3d_data.size());
data_ctx->write(vout3d_data.data(), vout3d_data.size());

// 循环遍历kv_self.cells中的元素
for (uint32_t i = 0; i < kv_size; ++i) {
    const auto & cell = kv_self.cells[i];
// 将cell的位置信息和序列ID的大小写入数据上下文
const llama_pos pos = cell.pos;
const size_t seq_id_size = cell.seq_id.size();

data_ctx->write(&pos, sizeof(pos));
data_ctx->write(&seq_id_size, sizeof(seq_id_size));

// 遍历序列ID，将每个序列ID写入数据上下文
for (auto seq_id : cell.seq_id) {
    data_ctx->write(&seq_id, sizeof(seq_id));
}
// 结束循环
}
}
// 将状态数据复制到目标内存中
size_t llama_copy_state_data(struct llama_context * ctx, uint8_t * dst) {
    llama_data_buffer_context data_ctx(dst);
    llama_copy_state_data_internal(ctx, &data_ctx);

    // 返回写入数据的大小
    return data_ctx.get_size_written();
}
// 设置从指定源地址读取的状态
size_t llama_set_state_data(struct llama_context * ctx, uint8_t * src) {
    uint8_t * inp = src; // 设置输入指针为源地址

    // 设置随机数生成器状态
    {
        size_t rng_size; // 随机数生成器状态大小
        char   rng_buf[LLAMA_MAX_RNG_STATE]; // 随机数生成器状态缓冲区

        // 从输入中复制随机数生成器状态大小，并移动输入指针
        memcpy(&rng_size,   inp, sizeof(rng_size));    inp += sizeof(rng_size);
        // 从输入中复制随机数生成器状态数据，并移动输入指针
        memcpy(&rng_buf[0], inp, LLAMA_MAX_RNG_STATE); inp += LLAMA_MAX_RNG_STATE;

        // 创建字符串流并将随机数生成器状态数据写入其中
        std::stringstream rng_ss;
        rng_ss.str(std::string(&rng_buf[0], rng_size));
        // 从字符串流中读取随机数生成器状态并存储到上下文中
        rng_ss >> ctx->rng;

        // 断言随机数生成器状态读取是否成功
        GGML_ASSERT(!rng_ss.fail());
    }
    // 设置logits（逻辑回归模型的输出）
    {
        // 定义logits的容量和大小
        size_t logits_cap;
        size_t logits_size;

        // 从输入中复制logits的容量和大小
        memcpy(&logits_cap,  inp, sizeof(logits_cap));  inp += sizeof(logits_cap);
        memcpy(&logits_size, inp, sizeof(logits_size)); inp += sizeof(logits_size);

        // 检查上下文中logits的容量是否与复制的logits_cap相等
        GGML_ASSERT(ctx->logits.capacity() == logits_cap);

        // 如果logits_size不为0，则调整上下文中logits的大小，并从输入中复制logits数据
        if (logits_size) {
            ctx->logits.resize(logits_size);
            memcpy(ctx->logits.data(), inp, logits_size * sizeof(float));
        }

        // 将输入指针移动到logits数据之后
        inp += logits_cap * sizeof(float);
    }

    // 设置embeddings（嵌入层）
    {
```
        // 定义变量 embedding_size，用于存储嵌入大小
        size_t embedding_size;

        // 从输入中复制嵌入大小的值到 embedding_size 变量中，并将输入指针移动到下一个位置
        memcpy(&embedding_size, inp, sizeof(embedding_size)); inp += sizeof(embedding_size);

        // 断言嵌入的容量与 embedding_size 相等
        GGML_ASSERT(ctx->embedding.capacity() == embedding_size);

        // 如果 embedding_size 不为 0，则从输入中复制嵌入数据到 ctx->embedding 中，并将输入指针移动到下一个位置
        if (embedding_size) {
            memcpy(ctx->embedding.data(), inp, embedding_size * sizeof(float));
            inp += embedding_size * sizeof(float);
        }
    }

    // 设置 kv 缓存
    {
        // 获取上下文中的 kv_self、hparams 和 cparams
        const auto & kv_self = ctx->kv_self;
        const auto & hparams = ctx->model.hparams;
        const auto & cparams = ctx->cparams;

        // 获取层数和嵌入维度
        const int    n_layer = hparams.n_layer;
        const int    n_embd  = hparams.n_embd_gqa();
        # 定义常量 n_ctx，表示上下文的大小
        const int    n_ctx   = cparams.n_ctx;

        # 定义变量 kv_buf_size，kv_head，kv_size，表示键值对缓冲区的大小，头部信息和大小
        size_t   kv_buf_size;
        uint32_t kv_head;
        uint32_t kv_size;

        # 从输入中复制键值对缓冲区大小，头部信息和大小，并将指针向后移动
        memcpy(&kv_buf_size, inp, sizeof(kv_buf_size)); inp += sizeof(kv_buf_size);
        memcpy(&kv_head,     inp, sizeof(kv_head));     inp += sizeof(kv_head);
        memcpy(&kv_size,     inp, sizeof(kv_size));     inp += sizeof(kv_size);

        # 如果键值对缓冲区大小不为0，则进行以下操作
        if (kv_buf_size) {
            # 断言键值对自身缓冲区的大小与 kv_buf_size 相等
            GGML_ASSERT(kv_self.buf.size == kv_buf_size);

            # 计算每个元素的大小
            const size_t elt_size = ggml_element_size(kv_self.k);

            # 初始化一个上下文，创建一个新的图
            ggml_context * cpy_ctx = ggml_init({ 6*ggml_tensor_overhead() + ggml_graph_overhead(), NULL, /* no_alloc */ true });
            ggml_cgraph * gf = ggml_new_graph(cpy_ctx);

            # 创建一个新的三维张量，表示输入的键值对信息
            ggml_tensor * kin3d = ggml_new_tensor_3d(cpy_ctx, kv_self.k->type, n_embd, kv_head, n_layer);
            kin3d->data = (void *) inp;
# 将输入字节大小增加到输入指针
inp += ggml_nbytes(kin3d);

# 创建一个新的三维张量vin3d，并将输入数据指针赋给它
ggml_tensor * vin3d = ggml_new_tensor_3d(cpy_ctx, kv_self.v->type, kv_head, n_embd, n_layer);
vin3d->data = (void *) inp;
inp += ggml_nbytes(vin3d);

# 创建一个三维张量k3d，通过对kv_self.k进行视图操作得到
ggml_tensor * k3d = ggml_view_3d(cpy_ctx, kv_self.k,
    n_embd, kv_head, n_layer,
    elt_size*n_embd, elt_size*n_embd*n_ctx, 0);

# 创建一个三维张量v3d，通过对kv_self.v进行视图操作得到
ggml_tensor * v3d = ggml_view_3d(cpy_ctx, kv_self.v,
    kv_head, n_embd, n_layer,
    elt_size*n_ctx, elt_size*n_ctx*n_embd, 0);

# 构建前向扩展，将k3d和vin3d的副本传递给ggml_build_forward_expand函数
ggml_build_forward_expand(gf, ggml_cpy(cpy_ctx, kin3d, k3d));
ggml_build_forward_expand(gf, ggml_cpy(cpy_ctx, vin3d, v3d));

# 使用图计算辅助函数计算图的前向传播
ggml_graph_compute_helper(ctx->work_buffer, gf, /*n_threads*/ 1);

# 释放cpy_ctx所占用的内存
ggml_free(cpy_ctx);
// 设置ctx->kv_self.head为kv_head，ctx->kv_self.size为kv_size
ctx->kv_self.head = kv_head;
ctx->kv_self.size = kv_size;

// 调整ctx->kv_self.cells的大小为kv_size
ctx->kv_self.cells.resize(kv_size);

// 遍历kv_size次
for (uint32_t i = 0; i < kv_size; ++i) {
    llama_pos pos;
    size_t    seq_id_size;

    // 从inp中复制sizeof(pos)大小的数据到pos，然后移动inp指针
    memcpy(&pos,         inp, sizeof(pos));         inp += sizeof(pos);
    // 从inp中复制sizeof(seq_id_size)大小的数据到seq_id_size，然后移动inp指针
    memcpy(&seq_id_size, inp, sizeof(seq_id_size)); inp += sizeof(seq_id_size);

    // 设置ctx->kv_self.cells[i].pos为pos
    ctx->kv_self.cells[i].pos = pos;

    llama_seq_id seq_id;

    // 遍历seq_id_size次
    for (size_t j = 0; j < seq_id_size; ++j) {
        // 从inp中复制sizeof(seq_id)大小的数据到seq_id，然后移动inp指针
        memcpy(&seq_id, inp, sizeof(seq_id)); inp += sizeof(seq_id);
        // 将seq_id插入到ctx->kv_self.cells[i].seq_id中
        ctx->kv_self.cells[i].seq_id.insert(seq_id);
    }
}
// 定义一个静态函数，用于内部加载会话文件到 llama_context 结构体中
static bool llama_load_session_file_internal(struct llama_context * ctx, const char * path_session, llama_token * tokens_out, size_t n_token_capacity, size_t * n_token_count_out) {
    // 以只读方式打开会话文件
    llama_file file(path_session, "rb");

    // 进行健全性检查
    {
        // 读取文件中的魔数（magic number）
        const uint32_t magic   = file.read_u32();
        // 读取文件中的版本号
        const uint32_t version = file.read_u32();
        // 检查魔数和版本号是否匹配，如果不匹配则输出错误信息并返回false
        if (magic != LLAMA_SESSION_MAGIC || version != LLAMA_SESSION_VERSION) {
            LLAMA_LOG_ERROR("%s : unknown (magic, version) for session file: %08x, %08x\n", __func__, magic, version);
            return false;
        }

        // 读取会话参数
        llama_hparams session_hparams;
        file.read_raw(&session_hparams, sizeof(llama_hparams));

        // 检查会话参数是否与模型参数匹配，如果不匹配则输出提示信息并返回false
        if (session_hparams != ctx->model.hparams) {
            LLAMA_LOG_INFO("%s : model hparams didn't match from session file!\n", __func__);
            return false;
        }
    }

    // 加载提示信息
    {
        // 读取提示信息的标记数量
        const uint32_t n_token_count = file.read_u32();

        // 如果提示信息的标记数量超过了容量，则输出错误信息
        if (n_token_count > n_token_capacity) {
            LLAMA_LOG_ERROR("%s : token count in session file exceeded capacity! %u > %zu\n", __func__, n_token_count, n_token_capacity);
// 如果文件读取失败，返回 false
return false;
}

// 从文件中读取原始数据到 tokens_out，每个 token 占用 sizeof(llama_token) 的大小，更新 n_token_count_out 的值
file.read_raw(tokens_out, sizeof(llama_token) * n_token_count);
*n_token_count_out = n_token_count;
}

// 恢复上下文状态
{
    // 计算当前状态大小和最大状态大小
    const size_t n_state_size_cur = file.size - file.tell();
    const size_t n_state_size_max = llama_get_state_size(ctx);

    // 如果当前状态大小超过最大状态大小，记录错误并返回 false
    if (n_state_size_cur > n_state_size_max) {
        LLAMA_LOG_ERROR("%s : the state size in session file is too big! max %zu, got %zu\n", __func__, n_state_size_max, n_state_size_cur);
        return false;
    }

    // 读取当前状态数据到 state_data 中
    std::vector<uint8_t> state_data(n_state_size_max);
    file.read_raw(state_data.data(), n_state_size_cur);
// 将状态数据设置到LLAMA上下文中
llama_set_state_data(ctx, state_data.data());
}

// 从文件中加载会话数据到LLAMA上下文中
bool llama_load_session_file(struct llama_context * ctx, const char * path_session, llama_token * tokens_out, size_t n_token_capacity, size_t * n_token_count_out) {
    try {
        // 调用内部函数加载会话文件数据
        return llama_load_session_file_internal(ctx, path_session, tokens_out, n_token_capacity, n_token_count_out);
    } catch (const std::exception & err) {
        // 捕获异常并记录错误日志
        LLAMA_LOG_ERROR("error loading session file: %s\n", err.what());
        return false;
    }
}

// 将LLAMA上下文中的会话数据保存到文件中
bool llama_save_session_file(struct llama_context * ctx, const char * path_session, const llama_token * tokens, size_t n_token_count) {
    // 创建一个以二进制写入模式打开的文件对象
    llama_file file(path_session, "wb");

    // 写入会话魔数和版本号到文件中
    file.write_u32(LLAMA_SESSION_MAGIC);
    file.write_u32(LLAMA_SESSION_VERSION);
    // 将模型超参数写入文件
    file.write_raw(&ctx->model.hparams, sizeof(llama_hparams));

    // 保存提示信息的长度，并将提示信息写入文件
    file.write_u32((uint32_t) n_token_count);
    file.write_raw(tokens, sizeof(llama_token) * n_token_count);

    // 使用流保存方式保存上下文状态
    llama_data_file_context data_ctx(&file);
    llama_copy_state_data_internal(ctx, &data_ctx);

    // 返回操作成功
    return true;
}

// 对上下文进行评估
int llama_eval(
        struct llama_context * ctx,
                 llama_token * tokens,
                     int32_t   n_tokens,
                         int   n_past) {
    // 从缓存中移除指定范围的键值对
    llama_kv_cache_seq_rm(ctx->kv_self, -1, n_past, -1);
// 调用 llama_batch_get_one 函数获取 tokens 中第 n_tokens 个元素对应的数据，并解码
const int ret = llama_decode_internal(*ctx, llama_batch_get_one(tokens, n_tokens, n_past, 0));
// 如果解码失败，记录错误日志
if (ret < 0) {
    LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
}

// 返回解码结果
return ret;
}

// 计算 tokens 对应的嵌入向量
int llama_eval_embd(
            struct llama_context * ctx,
                           float * embd,
                         int32_t   n_tokens,
                             int   n_past) {
    // 从 kv_self 中移除指定范围的键值对
    llama_kv_cache_seq_rm(ctx->kv_self, -1, n_past, -1);

    // 初始化 batch 结构体
    llama_batch batch = { n_tokens, nullptr, embd, nullptr, nullptr, nullptr, nullptr, n_past, 1, 0, };

    // 调用 llama_decode_internal 函数解码 batch 中的数据
    const int ret = llama_decode_internal(*ctx, batch);
    // 如果解码失败，记录错误日志
    if (ret < 0) {
// 记录错误日志，包括函数名和返回值
LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
}

// 设置LLAMA上下文中的线程数和批处理线程数
void llama_set_n_threads(struct llama_context * ctx, uint32_t n_threads, uint32_t n_threads_batch) {
    ctx->cparams.n_threads       = n_threads;
    ctx->cparams.n_threads_batch = n_threads_batch;
}

// 获取一个LLAMA批处理对象
struct llama_batch llama_batch_get_one(
             llama_token * tokens,
                 int32_t   n_tokens,
               llama_pos   pos_0,
            llama_seq_id   seq_id) {
    return {
        /*n_tokens       =*/ n_tokens,  // 设置n_tokens字段
        /*tokens         =*/ tokens,    // 设置tokens字段
        /*embd           =*/ nullptr,   // 设置embd字段为空指针
        /*pos            =*/ nullptr,  // 初始化 pos 为 nullptr
        /*n_seq_id       =*/ nullptr,  // 初始化 n_seq_id 为 nullptr
        /*seq_id         =*/ nullptr,  // 初始化 seq_id 为 nullptr
        /*logits         =*/ nullptr,  // 初始化 logits 为 nullptr
        /*all_pos_0      =*/ pos_0,    // 初始化 all_pos_0 为 pos_0
        /*all_pos_1      =*/ 1,         // 初始化 all_pos_1 为 1
        /*all_seq_id     =*/ seq_id,    // 初始化 all_seq_id 为 seq_id
    };
}

struct llama_batch llama_batch_init(int32_t n_tokens, int32_t embd, int32_t n_seq_max) {
    llama_batch batch = { 0, nullptr, nullptr, nullptr, nullptr, nullptr, nullptr, 0, 0, 0, };  // 初始化 batch 结构体

    if (embd) {
        batch.embd = (float *) malloc(sizeof(float) * n_tokens * embd);  // 如果 embd 不为 0，则分配内存给 batch.embd
    } else {
        batch.token = (llama_token *) malloc(sizeof(llama_token) * n_tokens);  // 如果 embd 为 0，则分配内存给 batch.token
    }

    batch.pos      = (llama_pos *)     malloc(sizeof(llama_pos)      * n_tokens);  // 分配内存给 batch.pos
    # 为 batch.n_seq_id 分配内存，大小为 int32_t 类型的 n_tokens 倍
    batch.n_seq_id = (int32_t *)       malloc(sizeof(int32_t)        * n_tokens);
    # 为 batch.seq_id 分配内存，大小为 llama_seq_id* 类型的 n_tokens 倍
    batch.seq_id   = (llama_seq_id **) malloc(sizeof(llama_seq_id *) * n_tokens);
    # 遍历 n_tokens，为每个 batch.seq_id[i] 分配内存，大小为 llama_seq_id 类型的 n_seq_max 倍
    for (int i = 0; i < n_tokens; ++i) {
        batch.seq_id[i] = (llama_seq_id *) malloc(sizeof(llama_seq_id) * n_seq_max);
    }
    # 为 batch.logits 分配内存，大小为 int8_t 类型的 n_tokens 倍
    batch.logits   = (int8_t *)        malloc(sizeof(int8_t)         * n_tokens);

    # 返回分配好内存的 batch 结构体
    return batch;
}

# 释放 batch 结构体中的内存
void llama_batch_free(struct llama_batch batch) {
    # 释放 batch.token 指向的内存
    if (batch.token)    free(batch.token);
    # 释放 batch.embd 指向的内存
    if (batch.embd)     free(batch.embd);
    # 释放 batch.pos 指向的内存
    if (batch.pos)      free(batch.pos);
    # 释放 batch.n_seq_id 指向的内存
    if (batch.n_seq_id) free(batch.n_seq_id);
    # 释放 batch.seq_id 指向的内存
    if (batch.seq_id) {
        # 遍历 batch.seq_id，释放每个 batch.seq_id[i] 指向的内存
        for (int i = 0; i < batch.n_tokens; ++i) {
            free(batch.seq_id[i]);
        }
        # 释放 batch.seq_id 指向的内存
        free(batch.seq_id);
    }
    // 如果 batch.logits 存在，则释放其内存空间
    if (batch.logits)   free(batch.logits);
}

// 解码函数，使用 llama_context 和 llama_batch 作为参数
int llama_decode(
        struct llama_context * ctx,
          struct llama_batch   batch) {
    // 调用内部解码函数 llama_decode_internal，返回结果保存在 ret 中
    const int ret = llama_decode_internal(*ctx, batch);
    // 如果 ret 小于 0，记录错误日志
    if (ret < 0) {
        LLAMA_LOG_ERROR("%s: failed to decode, ret = %d\n", __func__, ret);
    }
    // 返回 ret
    return ret;
}

// 获取 logits 数据的函数，使用 llama_context 作为参数
float * llama_get_logits(struct llama_context * ctx) {
    // 返回 ctx 中 logits 的数据指针
    return ctx->logits.data();
}

// 获取第 i 个 logits 数据的函数，使用 llama_context 和 int32_t 作为参数
float * llama_get_logits_ith(struct llama_context * ctx, int32_t i) {
// 返回指向模型输出的指定位置的指针
return ctx->logits.data() + i*ctx->model.hparams.n_vocab;
}

// 返回指向模型嵌入的指针
float * llama_get_embeddings(struct llama_context * ctx) {
    return ctx->embedding.data();
}

// 返回标记对应的文本内容
const char * llama_token_get_text(const struct llama_model * model, llama_token token) {
    return model->vocab.id_to_token[token].text.c_str();
}

// 返回标记对应的分数
float llama_token_get_score(const struct llama_model * model, llama_token token) {
    return model->vocab.id_to_token[token].score;
}

// 返回标记对应的类型
llama_token_type llama_token_get_type(const struct llama_model * model, llama_token token) {
    return model->vocab.id_to_token[token].type;
}

// 返回表示句子开头的标记
llama_token llama_token_bos(const struct llama_model * model) {
# 返回模型的特殊起始符号的标识符
llama_token llama_token_bos(const struct llama_model * model) {
    return model->vocab.special_bos_id;
}

# 返回模型的特殊结束符号的标识符
llama_token llama_token_eos(const struct llama_model * model) {
    return model->vocab.special_eos_id;
}

# 返回模型的特殊换行符号的标识符
llama_token llama_token_nl(const struct llama_model * model) {
    return model->vocab.linefeed_id;
}

# 返回模型的特殊前缀符号的标识符
llama_token llama_token_prefix(const struct llama_model * model) {
    return model->vocab.special_prefix_id;
}

# 返回模型的特殊中间符号的标识符
llama_token llama_token_middle(const struct llama_model * model) {
    return model->vocab.special_middle_id;
}

# 返回模型的特殊后缀符号的标识符
llama_token llama_token_suffix(const struct llama_model * model) {
// 返回模型中特殊后缀的标识符
return model->vocab.special_suffix_id;

// 返回模型中特殊结束标记的标识符
llama_token llama_token_eot(const struct llama_model * model) {
    return model->vocab.special_eot_id;
}

// 对输入文本进行标记化处理
int llama_tokenize(
    const struct llama_model * model, // 输入的模型
                  const char * text,   // 输入的文本
                         int   text_len, // 输入文本的长度
                 llama_token * tokens,  // 存储标记结果的数组
                         int   n_max_tokens, // 最大标记数量
                        bool   add_bos,  // 是否添加开始标记
                        bool   special) { // 是否处理特殊标记
    // 调用内部标记化处理函数
    auto res = llama_tokenize_internal(model->vocab, std::string(text, text_len), add_bos, special);

    // 如果标记数量超过最大限制，则返回错误
    if (n_max_tokens < (int) res.size()) {
        // 输出错误信息
        // LLAMA_LOG_ERROR("%s: too many tokens\n", __func__);
        return -((int) res.size());
    }

    // 遍历结果数组，将每个元素赋值给tokens数组
    for (size_t i = 0; i < res.size(); i++) {
        tokens[i] = res[i];
    }

    // 返回结果数组的大小
    return res.size();
}

// 对输入的文本进行 llama 解码
static std::string llama_decode_text(const std::string & text) {
    // 初始化解码后的文本
    std::string decoded_text;
    // 将输入文本转换为 Unicode 序列
    auto unicode_sequences = codepoints_from_utf8(text);
    // 遍历 Unicode 序列，将每个 Unicode 转换为字节对应的 BPE 编码
    for (auto& unicode_sequence : unicode_sequences) {
        decoded_text += unicode_to_bytes_bpe(codepoint_to_utf8(unicode_sequence));
    }

    // 返回解码后的文本
    return decoded_text;
}

// 不向缓冲区写入空终止符
// 将 llama_token 转换为对应的文本片段
int llama_token_to_piece(const struct llama_model * model, llama_token token, char * buf, int length) {
    // 检查 token 是否在合法范围内
    if (0 <= token && token < llama_n_vocab(model)) {
        // 根据词汇表类型进行不同的处理
        switch (llama_vocab_get_type(model->vocab)) {
        case LLAMA_VOCAB_TYPE_SPM: {
            // 如果 token 是正常的词汇表项
            if (llama_is_normal_token(model->vocab, token)) {
                // 获取 token 对应的文本
                std::string result = model->vocab.id_to_token[token].text;
                // 去除文本中的空白字符
                llama_unescape_whitespace(result);
                // 如果目标缓冲区长度不足，返回负数
                if (length < (int) result.length()) {
                    return -result.length();
                }
                // 将文本复制到目标缓冲区
                memcpy(buf, result.c_str(), result.length());
                return result.length();
            } 
            // 如果 token 是未知的词汇表项
            else if (llama_is_unknown_token(model->vocab, token)) { // NOLINT
                // 如果目标缓冲区长度不足，返回负数
                if (length < 3) {
                    return -3;
                }
                // 将未知词汇的占位符复制到目标缓冲区
                memcpy(buf, "\xe2\x96\x85", 3);
                return 3;
            } 
            // 如果 token 是控制字符
            else if (llama_is_control_token(model->vocab, token)) {
                // 空语句，不做任何操作
                ;
            } else if (llama_is_byte_token(model->vocab, token)) {
                // 如果 token 是字节类型的，且长度小于1，则返回-1
                if (length < 1) {
                    return -1;
                }
                // 将 token 转换为字节，并存入 buf 中，返回1
                buf[0] = llama_token_to_byte(model->vocab, token);
                return 1;
            } else {
                // TODO: 目前我们接受所有不支持的 token 类型，将它们像 CONTROL token 一样屏蔽
                // GGML_ASSERT(false);
            }
            break;
        }
        case LLAMA_VOCAB_TYPE_BPE: {
            if (llama_is_normal_token(model->vocab, token)) {
                // 如果 token 是正常类型的，将其转换为字符串，并解码
                std::string result = model->vocab.id_to_token[token].text;
                result = llama_decode_text(result);
                // 如果长度小于结果字符串的长度，返回负的结果字符串长度
                if (length < (int) result.length()) {
                    return -result.length();
                }
                // 将 result 字符串的内容复制到 buf 缓冲区中
                memcpy(buf, result.c_str(), result.length());
                // 返回 result 字符串的长度
                return result.length();
            } else if (llama_is_control_token(model->vocab, token)) {
                // 如果 token 是控制标记，则不做任何操作
                ;
            } else {
                // TODO: 目前我们接受所有不支持的标记类型，像控制标记一样将它们抑制
                // GGML_ASSERT(false);
            }
            // 跳出 switch 语句
            break;
        }
        // 默认情况下
        default:
            // 抛出断言错误
            GGML_ASSERT(false);
        }
    }
    // 返回 0
    return 0;
}

// 获取 llama_context 结构体中的时间信息
struct llama_timings llama_get_timings(struct llama_context * ctx) {
    // 创建 llama_timings 结构体 result
    struct llama_timings result = {
        /*.t_start_ms  =*/ 1e-3 * ctx->t_start_us,  // 将上下文中的微秒级起始时间转换为毫秒级
        /*.t_end_ms    =*/ 1.00 * ggml_time_ms(),   // 获取当前时间并转换为毫秒级作为结束时间
        /*.t_load_ms   =*/ 1e-3 * ctx->t_load_us,   // 将上下文中的微秒级加载时间转换为毫秒级
        /*.t_sample_ms =*/ 1e-3 * ctx->t_sample_us, // 将上下文中的微秒级采样时间转换为毫秒级
        /*.t_p_eval_ms =*/ 1e-3 * ctx->t_p_eval_us, // 将上下文中的微秒级参数评估时间转换为毫秒级
        /*.t_eval_ms   =*/ 1e-3 * ctx->t_eval_us,   // 将上下文中的微秒级评估时间转换为毫秒级

        /*.n_sample =*/ std::max(1, ctx->n_sample),  // 获取上下文中的采样数量，如果小于1则取1
        /*.n_p_eval =*/ std::max(1, ctx->n_p_eval),  // 获取上下文中的参数评估数量，如果小于1则取1
        /*.n_eval   =*/ std::max(1, ctx->n_eval),    // 获取上下文中的评估数量，如果小于1则取1
    };

    return result;
}

void llama_print_timings(struct llama_context * ctx) {
    const llama_timings timings = llama_get_timings(ctx);  // 获取上下文中的时间信息

    LLAMA_LOG_INFO("\n");
    LLAMA_LOG_INFO("%s:        load time = %10.2f ms\n", __func__, timings.t_load_ms);  // 打印加载时间
    # 记录日志信息，包括函数名、采样时间、采样次数、每个标记的平均时间、每秒处理的标记数
    LLAMA_LOG_INFO("%s:      sample time = %10.2f ms / %5d runs   (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, timings.t_sample_ms, timings.n_sample, timings.t_sample_ms / timings.n_sample, 1e3 / timings.t_sample_ms * timings.n_sample);
    # 记录日志信息，包括函数名、提示评估时间、提示评估标记数、每个标记的平均时间、每秒处理的标记数
    LLAMA_LOG_INFO("%s: prompt eval time = %10.2f ms / %5d tokens (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, timings.t_p_eval_ms, timings.n_p_eval, timings.t_p_eval_ms / timings.n_p_eval, 1e3 / timings.t_p_eval_ms * timings.n_p_eval);
    # 记录日志信息，包括函数名、评估时间、评估次数、每次评估的平均时间、每秒处理的标记数
    LLAMA_LOG_INFO("%s:        eval time = %10.2f ms / %5d runs   (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, timings.t_eval_ms, timings.n_eval, timings.t_eval_ms / timings.n_eval, 1e3 / timings.t_eval_ms * timings.n_eval);
    # 记录日志信息，包括函数名、总时间
    LLAMA_LOG_INFO("%s:       total time = %10.2f ms\n", __func__, (timings.t_end_ms - timings.t_start_ms));
}

# 重置计时器
void llama_reset_timings(struct llama_context * ctx) {
    # 记录开始时间
    ctx->t_start_us = ggml_time_us();
    # 重置采样时间和次数
    ctx->t_sample_us = ctx->n_sample = 0;
    # 重置评估时间和次数
    ctx->t_eval_us   = ctx->n_eval   = 0;
    # 重置提示评估时间和次数
    ctx->t_p_eval_us = ctx->n_p_eval = 0;
}

# 打印系统信息
const char * llama_print_system_info(void) {
    # 静态字符串变量
    static std::string s;

    # 清空字符串
    s  = "";
# 将各种 CPU 特性的信息转换为字符串，并拼接到 s 变量中
s += "AVX = "         + std::to_string(ggml_cpu_has_avx())         + " | ";
s += "AVX2 = "        + std::to_string(ggml_cpu_has_avx2())        + " | ";
s += "AVX512 = "      + std::to_string(ggml_cpu_has_avx512())      + " | ";
s += "AVX512_VBMI = " + std::to_string(ggml_cpu_has_avx512_vbmi()) + " | ";
s += "AVX512_VNNI = " + std::to_string(ggml_cpu_has_avx512_vnni()) + " | ";
s += "FMA = "         + std::to_string(ggml_cpu_has_fma())         + " | ";
s += "NEON = "        + std::to_string(ggml_cpu_has_neon())        + " | ";
s += "ARM_FMA = "     + std::to_string(ggml_cpu_has_arm_fma())     + " | ";
s += "F16C = "        + std::to_string(ggml_cpu_has_f16c())        + " | ";
s += "FP16_VA = "     + std::to_string(ggml_cpu_has_fp16_va())     + " | ";
s += "WASM_SIMD = "   + std::to_string(ggml_cpu_has_wasm_simd())   + " | ";
s += "BLAS = "        + std::to_string(ggml_cpu_has_blas())        + " | ";
s += "SSE3 = "        + std::to_string(ggml_cpu_has_sse3())        + " | ";
s += "SSSE3 = "       + std::to_string(ggml_cpu_has_ssse3())       + " | ";
s += "VSX = "         + std::to_string(ggml_cpu_has_vsx())         + " | ";

# 返回拼接后的字符串的 C 风格字符串
return s.c_str()
    # 打印空行
    fprintf(stream, "\n");
    # 打印分隔符和标题
    fprintf(stream, "###########\n");
    fprintf(stream, "# Timings #\n");
    fprintf(stream, "###########\n");
    fprintf(stream, "\n");

    # 打印生成过程中每个标记的平均毫秒数
    fprintf(stream, "mst_eval: %.2f  # ms / token during generation\n",
            1.0e-3 * ctx->t_eval_us / ctx->n_eval);
    # 打印处理提示过程中每个标记的平均毫秒数
    fprintf(stream, "mst_p_eval: %.2f  # ms / token during prompt processing\n",
            1.0e-3 * ctx->t_p_eval_us / ctx->n_p_eval);
    # 打印抽样过程中每个标记的平均毫秒数
    fprintf(stream, "mst_sample: %.2f  # ms / token during sampling\n",
            1.0e-3 * ctx->t_sample_us / ctx->n_sample);
    # 打印生成的标记数量（不包括第一个）
    fprintf(stream, "n_eval: %d  # number of tokens generated (excluding the first one)\n", ctx->n_eval);
    # 打印在开始时以批处理方式处理的标记数量
    fprintf(stream, "n_p_eval: %d  # number of tokens processed in batches at the beginning\n", ctx->n_p_eval);
    # 打印抽样的标记数量
    fprintf(stream, "n_sample: %d  # number of sampled tokens\n", ctx->n_sample);
    # 打印生成标记所花费的总微秒数
    fprintf(stream, "t_eval_us: %" PRId64 "  # total microseconds spent generating tokens\n", ctx->t_eval_us);
    # 打印加载模型所花费的总微秒数
    fprintf(stream, "t_load_us: %" PRId64 "  # total microseconds spent loading the model\n", ctx->t_load_us);
    # 打印处理提示所花费的总微秒数
    fprintf(stream, "t_p_eval_us: %" PRId64 "  # total microseconds spent prompt processing\n", ctx->t_p_eval_us);
    # 打印抽样所花费的总微秒数
    fprintf(stream, "t_sample_us: %" PRId64 "  # total microseconds spent sampling\n", ctx->t_sample_us);
    # 打印生成标记的速度（每秒生成的标记数量）
    fprintf(stream, "ts_eval: %.2f  # tokens / second during generation\n",
// 输出提示信息，显示每秒处理的标记数
fprintf(stream, "ts_eval: %.2f  # tokens / second during evaluation\n",
        1.0e6 * ctx->n_eval / ctx->t_eval_us);
// 输出提示信息，显示每秒处理的标记数
fprintf(stream, "ts_p_eval: %.2f  # tokens / second during prompt processing\n",
        1.0e6 * ctx->n_p_eval / ctx->t_p_eval_us);
// 输出提示信息，显示每秒处理的标记数
fprintf(stream, "ts_sample: %.2f  # tokens / second during sampling\n",
        1.0e6 * ctx->n_sample / ctx->t_sample_us);
}

// 用于内部测试使用，返回模型中张量的名称和结构体指针的向量
const std::vector<std::pair<std::string, struct ggml_tensor *>> & llama_internal_get_tensor_map(
    struct llama_context * ctx
) {
    return ctx->model.tensors_by_name;
}

// 设置日志回调函数和用户数据
void llama_log_set(ggml_log_callback log_callback, void * user_data) {
    g_state.log_callback = log_callback ? log_callback : llama_log_callback_default;
    g_state.log_callback_user_data = user_data;
}

// 内部使用的日志输出函数
static void llama_log_internal_v(ggml_log_level level, const char * format, va_list args) {
// 创建一个与args相同的args_copy变量
va_list args_copy;
// 复制args的内容到args_copy
va_copy(args_copy, args);
// 创建一个大小为128的字符数组buffer
char buffer[128];
// 使用format和args的内容将字符串格式化为buffer，最多128个字符
int len = vsnprintf(buffer, 128, format, args);
// 如果格式化后的字符串长度小于128
if (len < 128) {
    // 调用全局状态中的log_callback函数，将level、buffer和log_callback_user_data作为参数
    g_state.log_callback(level, buffer, g_state.log_callback_user_data);
} else {
    // 创建一个大小为len+1的字符数组buffer2
    char* buffer2 = new char[len+1];
    // 使用format和args_copy的内容将字符串格式化为buffer2，最多len+1个字符
    vsnprintf(buffer2, len+1, format, args_copy);
    // 将buffer2的第len个字符设为0
    buffer2[len] = 0;
    // 调用全局状态中的log_callback函数，将level、buffer2和log_callback_user_data作为参数
    g_state.log_callback(level, buffer2, g_state.log_callback_user_data);
    // 释放buffer2的内存
    delete[] buffer2;
}
// 结束args_copy的使用
va_end(args_copy);
}

// 定义一个静态函数llama_log_internal，接受level和format作为参数
static void llama_log_internal(ggml_log_level level, const char * format, ...) {
    // 创建一个va_list类型的args变量
    va_list args;
    // 初始化args，使其指向format之后的参数
    va_start(args, format);
    // 调用llama_log_internal_v函数，传入level、format和args作为参数
    llama_log_internal_v(level, format, args);
}
// 结束可变参数列表的使用
va_end(args);
}

// 默认的日志回调函数，将日志输出到标准错误流
static void llama_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    // 忽略日志级别和用户数据
    (void) level;
    (void) user_data;
    // 将日志文本输出到标准错误流
    fputs(text, stderr);
    // 刷新标准错误流
    fflush(stderr);
}
```