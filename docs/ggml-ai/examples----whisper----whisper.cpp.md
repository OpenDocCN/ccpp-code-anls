# `ggml\examples\whisper\whisper.cpp`

```
#include "whisper.h"

#ifdef WHISPER_USE_COREML
#include "coreml/whisper-encoder.h"
#endif

#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif

#ifdef WHISPER_USE_OPENVINO
#include "openvino/whisper-openvino-encoder.h"
#endif

#include "ggml.h"
#include "ggml-alloc.h"
#include "ggml-backend.h"

#include <atomic>  // 原子操作库
#include <algorithm>  // 算法库
#include <cassert>  // 断言库
#define _USE_MATH_DEFINES
#include <cmath>  // 数学库
#include <cstdio>  // C标准输入输出库
#include <cstdarg>  // 可变参数库
#include <cstring>  // 字符串库
#include <fstream>  // 文件流库
#include <map>  // 映射库
#include <set>  // 集合库
#include <string>  // 字符串库
#include <thread>  // 线程库
#include <vector>  // 向量库
#include <regex>  // 正则表达式库
#include <random>  // 随机数库
#include <functional>  // 函数对象库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

#if defined(GGML_BIG_ENDIAN)
#include <bit>  // 位操作库

template<typename T>
static T byteswap(T value) {
    return std::byteswap(value);  // 字节交换函数
}

template<>
float byteswap(float value) {
    return std::bit_cast<float>(byteswap(std::bit_cast<std::uint32_t>(value)));  // 浮点数字节交换函数
}

template<typename T>
static void byteswap_tensor_data(ggml_tensor * tensor) {
    T * datum = reinterpret_cast<T *>(tensor->data);  // 将数据指针转换为特定类型指针
    for (int i = 0; i < ggml_nelements(tensor); i++) {
        datum[i] = byteswap(datum[i]);  // 对数据进行字节交换
    }
}

static void byteswap_tensor(ggml_tensor * tensor) {
    switch (tensor->type) {
        case GGML_TYPE_I16: {
            byteswap_tensor_data<int16_t>(tensor);  // 对I16类型数据进行字节交换
            break;
        }
        case GGML_TYPE_F16: {
            byteswap_tensor_data<ggml_fp16_t>(tensor);  // 对F16类型数据进行字节交换
            break;
        }
        case GGML_TYPE_I32: {
            byteswap_tensor_data<int32_t>(tensor);  // 对I32类型数据进行字节交换
            break;
        }
        case GGML_TYPE_F32: {
            byteswap_tensor_data<float>(tensor);  // 对F32类型数据进行字节交换
            break;
        }
        default: { // GML_TYPE_I8
            break;
        }
    }
}

#define BYTESWAP_VALUE(d) d = byteswap(d)  // 宏定义，对值进行字节交换
#define BYTESWAP_FILTERS(f)            \  // 宏定义，对滤波器进行字节交换
    # 使用 do-while 循环，目的是为了在循环体内使用 break 语句时能够跳出整个循环
    do {                              \
        # 遍历 f.data 中的每个元素
        for (auto & datum : f.data) { \
            # 对每个元素进行字节交换操作
            datum = byteswap(datum);  \
        }                             \
    } while (0)
// 定义一个宏，用于对张量进行字节交换
#define BYTESWAP_TENSOR(t)       \
    do {                         \
        byteswap_tensor(t); \
    } while (0)
#else
// 如果不需要字节交换，则定义空的宏
#define BYTESWAP_VALUE(d) do {} while (0)
#define BYTESWAP_FILTERS(f) do {} while (0)
#define BYTESWAP_TENSOR(t) do {} while (0)
#endif

#ifdef __GNUC__
#ifdef __MINGW32__
// 根据不同的编译器定义不同的格式化属性
#define WHISPER_ATTRIBUTE_FORMAT(...) __attribute__((format(gnu_printf, __VA_ARGS__)))
#else
#define WHISPER_ATTRIBUTE_FORMAT(...) __attribute__((format(printf, __VA_ARGS__)))
#endif
#else
#define WHISPER_ATTRIBUTE_FORMAT(...)
#endif

//
// logging
//

// 定义日志输出函数，支持格式化输出
WHISPER_ATTRIBUTE_FORMAT(2, 3)
static void whisper_log_internal        (ggml_log_level level, const char * format, ...);
static void whisper_log_callback_default(ggml_log_level level, const char * text, void * user_data);

// 定义输出不同级别日志的宏
#define WHISPER_LOG_ERROR(...) whisper_log_internal(GGML_LOG_LEVEL_ERROR, __VA_ARGS__)
#define WHISPER_LOG_WARN(...)  whisper_log_internal(GGML_LOG_LEVEL_WARN , __VA_ARGS__)
#define WHISPER_LOG_INFO(...)  whisper_log_internal(GGML_LOG_LEVEL_INFO , __VA_ARGS__)

// 定义是否启用调试日志的宏
#define WHISPER_DEBUG

#if defined(WHISPER_DEBUG)
// 如果启用调试日志，则定义输出调试日志的宏
#define WHISPER_LOG_DEBUG(...) whisper_log_internal(GGML_LOG_LEVEL_DEBUG, __VA_ARGS__)
#else
#define WHISPER_LOG_DEBUG(...)
#endif

// 定义断言宏，如果条件不成立则输出错误日志并终止程序
#define WHISPER_ASSERT(x) \
    do { \
        if (!(x)) { \
            WHISPER_LOG_ERROR("WHISPER_ASSERT: %s:%d: %s\n", __FILE__, __LINE__, #x); \
            abort(); \
        } \
    } while (0)

//#define WHISPER_USE_FLASH_ATTN
//#define WHISPER_USE_FLASH_FF
// 定义最大解码器数量和最大节点数量
#define WHISPER_MAX_DECODERS 8
#define WHISPER_MAX_NODES 4096

//
// ggml helpers
//

// 定义一个辅助函数，用于计算图的执行计划
static bool ggml_graph_compute_helper(
          struct ggml_cgraph * graph,
        std::vector<uint8_t> & buf,
                         int   n_threads,
      whisper_abort_callback   abort_callback,
                        void * abort_callback_data) {
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);
    # 设置计划的中止回调函数
    plan.abort_callback = abort_callback;
    # 设置计划的中止回调数据
    plan.abort_callback_data = abort_callback_data;

    # 如果计划的工作大小大于0
    if (plan.work_size > 0) {
        # 调整缓冲区大小为工作大小
        buf.resize(plan.work_size);
        # 将计划的工作数据设置为缓冲区的数据
        plan.work_data = buf.data();
    }

    # 调用 ggml_graph_compute 函数执行计划的图计算
    return ggml_graph_compute(graph, &plan);
// 辅助函数，用于计算图形的操作
static bool ggml_graph_compute_helper(
       struct ggml_backend * backend,
        struct ggml_cgraph * graph,
                       int   n_threads) {
    // 如果后端是 CPU，则设置线程数
    if (ggml_backend_is_cpu(backend)) {
        ggml_backend_cpu_set_n_threads(backend, n_threads);
    }
    // 如果使用 Metal 后端，则设置计算块数
#ifdef GGML_USE_METAL
    if (ggml_backend_is_metal(backend)) {
        ggml_backend_metal_set_n_cb(backend, n_threads);
    }
#endif
    // 调用后端的图形计算函数
    return ggml_backend_graph_compute(backend, graph);
}

// 用于对没有维度 0 可被 "pad" 整除的张量进行更快的矩阵乘法运算
// 思路是将原始矩阵乘法：
//
//   Z = X @ Y
//
// 表示为两个矩阵乘法的和：
//
//   Z = (X_0 @ Y_0) + (X_1 @ Y_1)
//
// 这里 X_0 和 Y_0 是 X 和 Y 的视图，其维度 0 可被 "pad" 整除
// 而 X_1 和 Y_1 是剩余的视图。X_1 和 Y_1 最终会变成可以用更通用的内核处理的小矩阵
//
static struct ggml_tensor * ggml_mul_mat_pad(struct ggml_context * ctx, struct ggml_tensor * x, struct ggml_tensor * y, int pad = 32) {
    // 只有当维度 0 至少比填充值大 8 倍时才使用填充
    // 否则我们不会从优化中获得太多好处
    const int n_pad_req = 8;

    if (x->ne[0] % pad == 0 || x->ne[0] / pad < n_pad_req) {
        return ggml_mul_mat(ctx, x, y);
    }

    // 创建 X_0 和 X_1 的视图
    struct ggml_tensor * x_0 = ggml_view_3d(ctx, x, (x->ne[0]/pad)*pad, x->ne[1], x->ne[2], x->nb[1], x->nb[2], 0);
    struct ggml_tensor * x_1 = ggml_view_3d(ctx, x,  x->ne[0]%pad,      x->ne[1], x->ne[2], x->nb[1], x->nb[2], x_0->ne[0]*x_0->nb[0]);

    // 创建 Y_0 和 Y_1 的视图
    struct ggml_tensor * y_0 = ggml_view_3d(ctx, y, (y->ne[0]/pad)*pad, y->ne[1], y->ne[2], y->nb[1], y->nb[2], 0);
    struct ggml_tensor * y_1 = ggml_view_3d(ctx, y,  y->ne[0]%pad,      y->ne[1], y->ne[2], y->nb[1], y->nb[2], y_0->ne[0]*y_0->nb[0]);

    // 返回两个矩阵乘法的和
    return ggml_add(ctx,
            ggml_mul_mat(ctx, x_0, y_0),
            ggml_mul_mat(ctx, x_1, y_1));
}
// 检查其他平台是否可以从这个优化中受益
// CUDA 目前存在问题 - 似乎 ggml_mul_mat 不能正确处理视图
#if defined(GGML_USE_METAL)
#define ggml_mul_mat ggml_mul_mat_pad
#endif

// 可用的 whisper 模型
enum e_model {
    MODEL_UNKNOWN,
    MODEL_TINY,
    MODEL_BASE,
    MODEL_SMALL,
    MODEL_MEDIUM,
    MODEL_LARGE,
};

// 模型名称与枚举值的映射
static const std::map<e_model, std::string> g_model_name = {
    { MODEL_UNKNOWN,  "unknown"  },
    { MODEL_TINY,     "tiny"     },
    { MODEL_BASE,     "base"     },
    { MODEL_SMALL,    "small"    },
    { MODEL_MEDIUM,   "medium"   },
    { MODEL_LARGE,    "large"    },
};

// 语言代码与其对应的编号和名称的映射
static const std::map<std::string, std::pair<int, std::string>> g_lang = {
    { "en",  { 0,  "english",         } },
    { "zh",  { 1,  "chinese",         } },
    { "de",  { 2,  "german",          } },
    { "es",  { 3,  "spanish",         } },
    { "ru",  { 4,  "russian",         } },
    { "ko",  { 5,  "korean",          } },
    { "fr",  { 6,  "french",          } },
    { "ja",  { 7,  "japanese",        } },
    { "pt",  { 8,  "portuguese",      } },
    { "tr",  { 9,  "turkish",         } },
    { "pl",  { 10, "polish",          } },
    { "ca",  { 11,  "catalan",        } },
    { "nl",  { 12,  "dutch",          } },
    { "ar",  { 13,  "arabic",         } },
    { "sv",  { 14,  "swedish",        } },
    { "it",  { 15,  "italian",        } },
    { "id",  { 16,  "indonesian",     } },
    { "hi",  { 17,  "hindi",          } },
    { "fi",  { 18,  "finnish",        } },
    { "vi",  { 19,  "vietnamese",     } },
    { "he",  { 20,  "hebrew",         } },
    { "uk",  { 21,  "ukrainian",      } },
    { "el",  { 22,  "greek",          } },
    { "ms",  { 23,  "malay",          } },
    { "cs",  { 24,  "czech",          } },
    { "ro",  { 25,  "romanian",       } },
    { "da",  { 26,  "danish",         } },
    { "hu",  { 27,  "hungarian",      } },
    { "ta",  { 28,  "tamil",          } },
    { "no",  { 29,  "norwegian",      } },  # 包含键值对，键为"no"，值为包含29和"norwegian"的元组
    { "th",  { 30,  "thai",           } },  # 包含键值对，键为"th"，值为包含30和"thai"的元组
    { "ur",  { 31,  "urdu",           } },  # 包含键值对，键为"ur"，值为包含31和"urdu"的元组
    { "hr",  { 32,  "croatian",       } },  # 包含键值对，键为"hr"，值为包含32和"croatian"的元组
    { "bg",  { 33,  "bulgarian",      } },  # 包含键值对，键为"bg"，值为包含33和"bulgarian"的元组
    { "lt",  { 34,  "lithuanian",     } },  # 包含键值对，键为"lt"，值为包含34和"lithuanian"的元组
    { "la",  { 35,  "latin",          } },  # 包含键值对，键为"la"，值为包含35和"latin"的元组
    { "mi",  { 36,  "maori",          } },  # 包含键值对，键为"mi"，值为包含36和"maori"的元组
    { "ml",  { 37,  "malayalam",      } },  # 包含键值对，键为"ml"，值为包含37和"malayalam"的元组
    { "cy",  { 38,  "welsh",          } },  # 包含键值对，键为"cy"，值为包含38和"welsh"的元组
    { "sk",  { 39,  "slovak",         } },  # 包含键值对，键为"sk"，值为包含39和"slovak"的元组
    { "te",  { 40,  "telugu",         } },  # 包含键值对，键为"te"，值为包含40和"telugu"的元组
    { "fa",  { 41,  "persian",        } },  # 包含键值对，键为"fa"，值为包含41和"persian"的元组
    { "lv",  { 42,  "latvian",        } },  # 包含键值对，键为"lv"，值为包含42和"latvian"的元组
    { "bn",  { 43,  "bengali",        } },  # 包含键值对，键为"bn"，值为包含43和"bengali"的元组
    { "sr",  { 44,  "serbian",        } },  # 包含键值对，键为"sr"，值为包含44和"serbian"的元组
    { "az",  { 45,  "azerbaijani",    } },  # 包含键值对，键为"az"，值为包含45和"azerbaijani"的元组
    { "sl",  { 46,  "slovenian",      } },  # 包含键值对，键为"sl"，值为包含46和"slovenian"的元组
    { "kn",  { 47,  "kannada",        } },  # 包含键值对，键为"kn"，值为包含47和"kannada"的元组
    { "et",  { 48,  "estonian",       } },  # 包含键值对，键为"et"，值为包含48和"estonian"的元组
    { "mk",  { 49,  "macedonian",     } },  # 包含键值对，键为"mk"，值为包含49和"macedonian"的元组
    { "br",  { 50,  "breton",         } },  # 包含键值对，键为"br"，值为包含50和"breton"的元组
    { "eu",  { 51,  "basque",         } },  # 包含键值对，键为"eu"，值为包含51和"basque"的元组
    { "is",  { 52,  "icelandic",      } },  # 包含键值对，键为"is"，值为包含52和"icelandic"的元组
    { "hy",  { 53,  "armenian",       } },  # 包含键值对，键为"hy"，值为包含53和"armenian"的元组
    { "ne",  { 54,  "nepali",         } },  # 包含键值对，键为"ne"，值为包含54和"nepali"的元组
    { "mn",  { 55,  "mongolian",      } },  # 包含键值对，键为"mn"，值为包含55和"mongolian"的元组
    { "bs",  { 56,  "bosnian",        } },  # 包含键值对，键为"bs"，值为包含56和"bosnian"的元组
    { "kk",  { 57,  "kazakh",         } },  # 包含键值对，键为"kk"，值为包含57和"kazakh"的元组
    { "sq",  { 58,  "albanian",       } },  # 包含键值对，键为"sq"，值为包含58和"albanian"的元组
    { "sw",  { 59,  "swahili",        } },  # 包含键值对，键为"sw"，值为包含59和"swahili"的元组
    { "gl",  { 60,  "galician",       } },  # 包含键值对，键为"gl"，值为包含60和"galician"的元组
    { "mr",  { 61,  "marathi",        } },  # 包含键值对，键为"mr"，值为包含61和"marathi"的元组
    { "pa",  { 62,  "punjabi",        } },  # 包含键值对，键为"pa"，值为包含62和"punjabi"的元组
    { "si",  { 63,  "sinhala",        } },  # 包含键值对，键为"si"，值为包含63和"sinhala"的元组
    { "km",  { 64,  "khmer",          } },  # 包含键值对，键为"km"，值为包含64和"khmer"的元组
    { "sn",  { 65,  "shona",          } },  # 包含键值对，键为"sn"，值为包含65和"shona"的元组
    { "yo",  { 66,  "yoruba",         } },  # 包含键值对，键为"yo"，值为包含66和"yoruba"的元组
    { "so",  { 67,  "somali",         } },  # 包含键值对，键为"so"，值为包含67和"somali"的元组
    { "af",  { 68,  "afrikaans",      } },  # 包含键值对，键为"af"，值为包含68和"afrikaans"的元组
    { "oc",  { 69,  "occitan",        } },  # 包含键值对，键为"oc"，值为包含69和"occitan"的元组
    { "ka",  { 70,  "georgian",       } },  # 包含键值对，键为"ka"，值为包含70和"georgian"的元组
    { "be",  { 71,  "belarusian",     } },  # 包含键值对，键为"be"，值为包含71和"belarusian"的元组
    { "tg",  { 72,  "tajik",          } },  # 包含键值对，键为"tg"，值为包含72和"tajik"的元组
    { "sd",  { 73,  "sindhi",         } },  # 包含键值对，键为"sd"，值为包含73和"sindhi"的元组
    { "gu",  { 74,  "gujarati",       } },  # 包含键值对，键为"gu"，值为包含74和"gujarati"的元组
    # 创建一个包含语言代码和对应语言信息的字典
    { "am",  { 75,  "amharic",        } },
    { "yi",  { 76,  "yiddish",        } },
    { "lo",  { 77,  "lao",            } },
    { "uz",  { 78,  "uzbek",          } },
    { "fo",  { 79,  "faroese",        } },
    { "ht",  { 80,  "haitian creole", } },
    { "ps",  { 81,  "pashto",         } },
    { "tk",  { 82,  "turkmen",        } },
    { "nn",  { 83,  "nynorsk",        } },
    { "mt",  { 84,  "maltese",        } },
    { "sa",  { 85,  "sanskrit",       } },
    { "lb",  { 86,  "luxembourgish",  } },
    { "my",  { 87,  "myanmar",        } },
    { "bo",  { 88,  "tibetan",        } },
    { "tl",  { 89,  "tagalog",        } },
    { "mg",  { 90,  "malagasy",       } },
    { "as",  { 91,  "assamese",       } },
    { "tt",  { 92,  "tatar",          } },
    { "haw", { 93,  "hawaiian",       } },
    { "ln",  { 94,  "lingala",        } },
    { "ha",  { 95,  "hausa",          } },
    { "ba",  { 96,  "bashkir",        } },
    { "jw",  { 97,  "javanese",       } },
    { "su",  { 98,  "sundanese",      } },
    { "yue", { 99,  "cantonese",      } },
// 定义了一个名为 whisper_mel 的结构体，包含 n_len, n_len_org, n_mel 三个整型变量和一个存储浮点数的向量 data
struct whisper_mel {
    int n_len;
    int n_len_org;
    int n_mel;

    std::vector<float> data;
};

// 定义了一个名为 whisper_filters 的结构体，包含 n_mel, n_fft 两个整型变量和一个存储浮点数的向量 data
struct whisper_filters {
    int32_t n_mel;
    int32_t n_fft;

    std::vector<float> data;
};

// 定义了一个名为 whisper_vocab 的结构体
struct whisper_vocab {
    using id    = int32_t;
    using token = std::string;

    int n_vocab = 51864; // 初始化 n_vocab 为 51864

    std::map<token, id> token_to_id; // 存储 token 到 id 的映射
    std::map<id, token> id_to_token; // 存储 id 到 token 的映射

    // 下面是一系列特殊的 token id 的定义
    id token_eot        = 50256;
    id token_sot        = 50257;
    id token_translate  = 50357;
    id token_transcribe = 50358;
    id token_solm       = 50359;
    id token_prev       = 50360;
    id token_nosp       = 50361;
    id token_not        = 50362;
    id token_beg        = 50363;

    // 返回 n_vocab 是否大于等于 51865，用于判断是否为多语言模型
    bool is_multilingual() const {
        return n_vocab >= 51865;
    }

    // 返回语言数量，n_vocab 减去基础数量 51765 再减去是否为多语言模型的标志位
    int num_languages() const {
        return n_vocab - 51765 - (is_multilingual() ? 1 : 0);
    }
};

// 定义了一个名为 whisper_segment 的结构体，包含 t0, t1 两个整型变量，一个存储字符串的 text，一个存储 whisper_token_data 结构体的向量 tokens，一个布尔值 speaker_turn_next
struct whisper_segment {
    int64_t t0;
    int64_t t1;

    std::string text;

    std::vector<whisper_token_data> tokens;

    bool speaker_turn_next;
};

// 定义了一个名为 whisper_batch 的结构体，包含 n_tokens 整型变量，指向 whisper_token 类型的指针 token，指向 whisper_pos 类型的指针 pos，指向整型数组的指针 n_seq_id，指向指针数组的指针 seq_id，指向 int8_t 类型的指针 logits
struct whisper_batch {
    int32_t n_tokens;

    whisper_token  *  token;
    whisper_pos    *  pos;
    int32_t        *  n_seq_id;
    whisper_seq_id ** seq_id;   // null terminated
    int8_t         *  logits;
};

// 定义了一个静态函数 whisper_batch_init，接受 n_tokens 和 n_seq_max 两个整型参数，返回一个初始化后的 whisper_batch 结构体
static struct whisper_batch whisper_batch_init(int32_t n_tokens, int32_t n_seq_max) {
    whisper_batch batch = { 0, nullptr, nullptr, nullptr, nullptr, nullptr, }; // 初始化 batch 结构体的各个成员为空

    // 为 batch 结构体的各个成员分配内存空间
    batch.token    = (whisper_token *  ) malloc(sizeof(whisper_token)    * (n_tokens));
    batch.pos      = (whisper_pos *)     malloc(sizeof(whisper_pos)      * (n_tokens));
    batch.n_seq_id = (int32_t *)         malloc(sizeof(int32_t)          * (n_tokens));
    # 为 batch.seq_id 分配内存空间，大小为 n_tokens + 1 个 whisper_seq_id 指针
    batch.seq_id   = (whisper_seq_id **) malloc(sizeof(whisper_seq_id *) * (n_tokens + 1));
    # 遍历 n_tokens，为每个 batch.seq_id[i] 分配内存空间，大小为 n_seq_max 个 whisper_seq_id
    for (int i = 0; i < n_tokens; ++i) {
        batch.seq_id[i] = (whisper_seq_id *) malloc(sizeof(whisper_seq_id)   * n_seq_max);
    }
    # 将 batch.seq_id[n_tokens] 指向空指针
    batch.seq_id[n_tokens] = nullptr;
    # 为 batch.logits 分配内存空间，大小为 n_tokens 个 int8_t
    batch.logits   = (int8_t *)          malloc(sizeof(int8_t)           * n_tokens);

    # 返回 batch 对象
    return batch;
// 释放批处理对象中的内存资源
static void whisper_batch_free(struct whisper_batch batch) {
    // 如果批处理对象中的 token 不为空，则释放其内存
    if (batch.token)    free(batch.token);
    // 如果批处理对象中的 pos 不为空，则释放其内存
    if (batch.pos)      free(batch.pos);
    // 如果批处理对象中的 n_seq_id 不为空，则释放其内存
    if (batch.n_seq_id) free(batch.n_seq_id);
    // 如果批处理对象中的 seq_id 不为空，则释放其内存
    if (batch.seq_id) {
        // 遍历 seq_id 数组，释放每个元素的内存
        for (int i = 0; batch.seq_id[i]; ++i) {
            free(batch.seq_id[i]);
        }
        // 释放 seq_id 数组的内存
        free(batch.seq_id);
    }
    // 如果批处理对象中的 logits 不为空，则释放其内存
    if (batch.logits)   free(batch.logits);
}

// 准备批处理对象的传统方法
static void whisper_batch_prep_legacy(whisper_batch & batch, const whisper_token * tokens, int n_tokens, int n_past, int seq_id) {
    // 设置批处理对象的 n_tokens 属性
    batch.n_tokens = n_tokens;
    // 遍历 n_tokens，为批处理对象的各个属性赋值
    for (int i = 0; i < n_tokens; ++i) {
        // 如果 tokens 不为空，则将其值赋给 batch.token[i]
        if (tokens) {
            batch.token[i] = tokens[i];
        }
        // 设置 batch.pos[i] 的值为 n_past + i
        batch.pos     [i]    = n_past + i;
        // 设置 batch.n_seq_id[i] 的值为 1
        batch.n_seq_id[i]    = 1;
        // 设置 batch.seq_id[i][0] 的值为 seq_id
        batch.seq_id  [i][0] = seq_id;
        // 设置 batch.logits[i] 的值为 0
        batch.logits  [i]    = 0;
    }
    // 设置 batch.logits[n_tokens - 1] 的值为 1
    batch.logits[n_tokens - 1] = 1;
}

// 使用自定义的 pair 结构替换 std::pair（原因：std::pair 很慢）
template<typename A, typename B>
struct whisper_pair {
    A first;  // 第一个元素
    B second; // 第二个元素

    // 定义一个接受两个参数的构造函数
    whisper_pair(const A& a, const B& b) : first(a), second(b) {}
    // 定义一个不接受参数的构造函数
    whisper_pair() : first(A()), second(B()) {}
};

// whisper 使用的 ggml_allocr 包装器
struct whisper_allocr {
    ggml_allocr * alloc = nullptr;  // 分配器对象指针

    std::vector<uint8_t> meta;  // 元数据

    ggml_backend_buffer_t buffer;  // 缓冲区对象
};

// 获取 whisper_allocr 结构体的大小
static size_t whisper_allocr_size(struct whisper_allocr & allocr) {
    return allocr.meta.size() + ggml_allocr_max_size(allocr.alloc);
}

// 测量图的内存使用情况并准备 allocr 的内部数据缓冲区
static void whisper_allocr_graph_init(struct whisper_allocr & allocr, ggml_backend_t backend, std::function<struct ggml_cgraph *()> && get_graph) {
    auto & alloc = allocr.alloc;  // 分配器对象的引用
    auto & meta  = allocr.meta;   // 元数据的引用

    // 使用后端创建一个新的测量分配器
    alloc = ggml_allocr_new_measure_from_backend(backend);
}
    # 调整 meta 的大小，以容纳 ggml_tensor_overhead()*WHISPER_MAX_NODES + ggml_graph_overhead() 的空间
    meta.resize(ggml_tensor_overhead()*WHISPER_MAX_NODES + ggml_graph_overhead());

    # 使用 alloc 分配器为当前图形分配空间
    ggml_allocr_alloc_graph(alloc, get_graph());
// 重新分配内存分配器的图形，根据后端类型重新分配内存
static void whisper_allocr_graph_realloc(struct whisper_allocr & allocr, ggml_backend_t backend) {
    // 如果分配器的分配指针为空，则返回，这可能是因为我们使用了像 CoreML 或 OpenVINO 这样的外部编码器
    if (allocr.alloc == nullptr) {
        return;
    }

    // 获取分配器的分配和缓冲区引用
    auto & alloc  = allocr.alloc;
    auto & buffer = allocr.buffer;

    // 获取分配器的最大大小
    size_t size = ggml_allocr_max_size(alloc);

    // 释放分配器的内存
    ggml_allocr_free(alloc);

    // 使用后端类型和大小分配新的缓冲区
    buffer = ggml_backend_alloc_buffer(backend, size);
    // 从新的缓冲区创建新的分配器
    alloc = ggml_allocr_new_from_buffer(buffer);
}

// 释放内存分配器
static void whisper_allocr_free(struct whisper_allocr & allocr) {
    // 如果分配器的分配指针不为空
    if (allocr.alloc) {
        // 释放分配器的内存
        ggml_allocr_free(allocr.alloc);
        // 释放分配器的缓冲区
        ggml_backend_buffer_free(allocr.buffer);
        // 将分配指针设置为空
        allocr.alloc = nullptr;
    }
}

// 媒介
// 超参数: {
// 'n_mels': 80,
// 'n_vocab': 51864,
// 'n_audio_ctx': 1500,
// 'n_audio_state': 1024,
// 'n_audio_head': 16,
// 'n_audio_layer': 24,
// 'n_text_ctx': 448,
// 'n_text_state': 1024,
// 'n_text_head': 16,
// 'n_text_layer': 24
// }
//
// 默认超参数（Whisper tiny）
struct whisper_hparams {
    int32_t n_vocab       = 51864;
    int32_t n_audio_ctx   = 1500;
    int32_t n_audio_state = 384;
    int32_t n_audio_head  = 6;
    int32_t n_audio_layer = 4;
    int32_t n_text_ctx    = 448;
    int32_t n_text_state  = 384;
    int32_t n_text_head   = 6;
    int32_t n_text_layer  = 4;
    int32_t n_mels        = 80;
    int32_t ftype         = 1;
    float   eps           = 1e-5f;
};

// 音频编码层
struct whisper_layer_encoder {
    // 编码器.blocks.*.attn_ln
    struct ggml_tensor * attn_ln_0_w;
    struct ggml_tensor * attn_ln_0_b;

    // 编码器.blocks.*.attn.out
    struct ggml_tensor * attn_ln_1_w;
    struct ggml_tensor * attn_ln_1_b;

    // 编码器.blocks.*.attn.query
    struct ggml_tensor * attn_q_w;
    struct ggml_tensor * attn_q_b;

    // 编码器.blocks.*.attn.key
    struct ggml_tensor * attn_k_w;

    // 编码器.blocks.*.attn.value
    struct ggml_tensor * attn_v_w;
    struct ggml_tensor * attn_v_b;
    // 定义指向 MLP Layer Normalization 权重和偏置的指针
    struct ggml_tensor * mlp_ln_w;
    struct ggml_tensor * mlp_ln_b;
    
    // 定义指向第一个 MLP 层的权重和偏置的指针
    struct ggml_tensor * mlp_0_w;
    struct ggml_tensor * mlp_0_b;
    
    // 定义指向第二个 MLP 层的权重和偏置的指针
    struct ggml_tensor * mlp_1_w;
    struct ggml_tensor * mlp_1_b;
// token decoding layer
// 定义了一个结构体，用于存储解码器的各个部分的权重和偏置

struct whisper_layer_decoder {
    // decoder.blocks.*.attn_ln
    // 解码器中注意力层的权重和偏置
    struct ggml_tensor * attn_ln_0_w;
    struct ggml_tensor * attn_ln_0_b;

    // decoder.blocks.*.attn.out
    // 解码器中注意力输出层的权重和偏置
    struct ggml_tensor * attn_ln_1_w;
    struct ggml_tensor * attn_ln_1_b;

    // decoder.blocks.*.attn.query
    // 解码器中注意力查询的权重和偏置
    struct ggml_tensor * attn_q_w;
    struct ggml_tensor * attn_q_b;

    // decoder.blocks.*.attn.key
    // 解码器中注意力键的权重

    struct ggml_tensor * attn_k_w;

    // decoder.blocks.*.attn.value
    // 解码器中注意力值的权重和偏置
    struct ggml_tensor * attn_v_w;
    struct ggml_tensor * attn_v_b;

    // decoder.blocks.*.cross_attn_ln
    // 解码器中交叉注意力层的权重和偏置
    struct ggml_tensor * cross_attn_ln_0_w;
    struct ggml_tensor * cross_attn_ln_0_b;

    // decoder.blocks.*.cross_attn.out
    // 解码器中交叉注意力输出层的权重和偏置
    struct ggml_tensor * cross_attn_ln_1_w;
    struct ggml_tensor * cross_attn_ln_1_b;

    // decoder.blocks.*.cross_attn.query
    // 解码器中交叉注意力查询的权重和偏置
    struct ggml_tensor * cross_attn_q_w;
    struct ggml_tensor * cross_attn_q_b;

    // decoder.blocks.*.cross_attn.key
    // 解码器中交叉注意力键的权重
    struct ggml_tensor * cross_attn_k_w;

    // decoder.blocks.*.cross_attn.value
    // 解码器中交叉注意力值的权重和偏置
    struct ggml_tensor * cross_attn_v_w;
    struct ggml_tensor * cross_attn_v_b;

    // decoder.blocks.*.mlp_ln
    // 解码器中多层感知机层的权重和偏置
    struct ggml_tensor * mlp_ln_w;
    struct ggml_tensor * mlp_ln_b;

    // decoder.blocks.*.mlp.0
    // 解码器中多层感知机第一层的权重和偏置
    struct ggml_tensor * mlp_0_w;
    struct ggml_tensor * mlp_0_b;

    // decoder.blocks.*.mlp.2
    // 解码器中多层感知机第二层的权重和偏置
    struct ggml_tensor * mlp_1_w;
    struct ggml_tensor * mlp_1_b;
};

// 定义了一个名为 whisper_kv_cell 的结构体
struct whisper_kv_cell {
    // 初始化 pos 为 -1
    whisper_pos pos = -1;

    // 定义了一个名为 seq_id 的集合
    std::set<whisper_seq_id> seq_id;

    // 检查是否存在指定的 seq_id
    bool has_seq_id(const whisper_seq_id & id) const {
        return seq_id.find(id) != seq_id.end();
    }
};

// 定义了一个名为 whisper_kv_cache 的结构体
struct whisper_kv_cache {
    // 初始化 head 和 size 为 0
    uint32_t head = 0;
    uint32_t size = 0;

    // 在每次构建图之前计算
    uint32_t n = 0;

    // 定义了一个名为 cells 的 whisper_kv_cell 结构体数组
    std::vector<whisper_kv_cell> cells;

    // 定义了 k 和 v 的 ggml_tensor 指针
    struct ggml_tensor * k;
    struct ggml_tensor * v;

    // 定义了一个 ggml_context 指针
    struct ggml_context * ctx;

    // 定义了一个 ggml_backend_buffer_t 类型的 buffer
    ggml_backend_buffer_t buffer;
};

// 定义了一个名为 whisper_model 的结构体
struct whisper_model {
    // 初始化模型类型为未知
    e_model type = MODEL_UNKNOWN;

    // 初始化模型的超参数
    whisper_hparams hparams;
    // 初始化模型的滤波器
    whisper_filters filters;

    // 初始化编码器的位置嵌入
    struct ggml_tensor * e_pe;

    // 初始化编码器的第一个卷积层权重和偏置
    struct ggml_tensor * e_conv_1_w;
    struct ggml_tensor * e_conv_1_b;

    // 初始化编码器的第二个卷积层权重和偏置
    struct ggml_tensor * e_conv_2_w;
    struct ggml_tensor * e_conv_2_b;

    // 初始化编码器的 Layer Normalization 参数
    struct ggml_tensor * e_ln_w;
    struct ggml_tensor * e_ln_b;

    // 初始化解码器的位置嵌入
    struct ggml_tensor * d_pe;

    // 初始化解码器的标记嵌入
    struct ggml_tensor * d_te;

    // 初始化解码器的 Layer Normalization 参数
    struct ggml_tensor * d_ln_w;
    struct ggml_tensor * d_ln_b;

    // 初始化编码器的层列表
    std::vector<whisper_layer_encoder> layers_encoder;
    // 初始化解码器的层列表
    std::vector<whisper_layer_decoder> layers_decoder;

    // 初始化包含模型张量的上下文
    struct ggml_context * ctx;

    // 模型后端数据是只读的，可以在处理器之间共享
    struct ggml_backend_buffer * buffer;

    // 张量数量
    int n_loaded;
    // 存储模型张量的映射
    std::map<std::string, struct ggml_tensor *> tensors;
// 结构体定义，用于部分 UTF-8 编码
struct whisper_partial_utf8 {
    uint32_t value;    // 已经存在的位值（未移位）
    int      n_remain; // 剩余的字节数；-1 表示无效序列
};

// 结构体定义，用于语法规则
struct whisper_grammar {
    /*const*/ std::vector<std::vector<whisper_grammar_element>> rules; // 语法规则的二维向量
    std::vector<std::vector<const whisper_grammar_element *>>   stacks; // 语法规则元素的指针的二维向量

    // 用于部分生成的 UTF-8 序列的缓冲区，来自已接受的标记
    whisper_partial_utf8 partial_utf8; // 部分 UTF-8 编码
};

// 结构体定义，用于语法规则的候选项
struct whisper_grammar_candidate {
    whisper_token          id; // 标记 ID
    const uint32_t       * code_points; // 代码点指针
    whisper_partial_utf8   partial_utf8; // 部分 UTF-8 编码
};

// 结构体定义，用于序列
struct whisper_sequence {
    std::vector<whisper_token_data> tokens; // 标记数据的向量

    // 当前迭代中累积的转录（用于截断标记数组）
    int result_len; // 结果长度

    double sum_logprobs_all; // 所有标记的对数概率之和
    double sum_logprobs;     // 标记的对数概率之和（前 result_len 个标记）
    double avg_logprobs;     // 标记的平均对数概率
    double entropy;          // 标记的熵
    double score;            // 可能性排名分数
};

// 标签：WHISPER_DECODER_INIT
// 结构体定义，用于解码器
struct whisper_decoder {
    // 当前生成的标记序列
    whisper_sequence sequence; // 序列

    // 生成的标记序列的语法解析状态
    whisper_grammar  grammar; // 语法

    int i_batch;    // 当前批次中标记的索引
    int seek_delta; // 基于解码时间戳标记找到的窗口移动

    bool failed;    // 当前段落是否解码失败？
    bool completed; // 解码器是否完成了当前段落？
    bool has_ts;    // 是否已经为当前段落抽样了非起始时间戳标记？

    // 上一次 whisper_decode 后的新标记概率、logits 和对数概率（一维数组：[n_vocab]）
    std::vector<float> probs;   // 概率
    std::vector<float> logits;  // logits
    std::vector<float> logprobs; // 对数概率
};
    // 创建一个工作容器，用于避免内存分配
    std::vector<whisper_pair<double, whisper_vocab::id>> logits_id;

    // 可变的伪随机数生成器，用于 t > 0.0 时的抽样
    mutable std::mt19937 rng;
// 结构体定义，用于存储whisper模型的状态信息
struct whisper_state {
    int64_t t_sample_us = 0; // 采样时间（微秒）
    int64_t t_encode_us = 0; // 编码时间（微秒）
    int64_t t_decode_us = 0; // 解码时间（微秒）
    int64_t t_batchd_us = 0; // 批处理时间（微秒）
    int64_t t_prompt_us = 0; // 提示时间（微秒）
    int64_t t_mel_us = 0; // mel时间（微秒）

    int32_t n_sample = 0; // 采样的标记数
    int32_t n_encode = 0; // 编码器调用次数
    int32_t n_decode = 0; // 具有n_tokens == 1的解码器调用次数（文本生成）
    int32_t n_batchd = 0; // 具有n_tokens < 16的解码器调用次数（批处理解码）
    int32_t n_prompt = 0; // 具有n_tokens > 1的解码器调用次数（提示编码）
    int32_t n_fail_p = 0; // logprob阈值失败次数
    int32_t n_fail_h = 0; // 熵阈值失败次数

    // 所有解码器的统一自注意KV缓存
    whisper_kv_cache kv_self;

    // 解码器的交叉注意KV缓存
    // 所有解码器共享
    whisper_kv_cache kv_cross;

    whisper_mel mel; // mel对象

    whisper_batch batch; // batch对象

    whisper_decoder decoders[WHISPER_MAX_DECODERS]; // 解码器数组

    ggml_backend_t backend = nullptr; // ggml后端对象

    // ggml-alloc:
    // - 将中间张量的元信息存储到`meta`缓冲区中
    // - 将实际张量数据存储到`data`缓冲区中
    whisper_allocr alloc_conv;
    whisper_allocr alloc_encode;
    whisper_allocr alloc_cross;
    whisper_allocr alloc_decode;

    // 编码器的结果
    struct ggml_tensor * embd_conv = nullptr;
    struct ggml_tensor * embd_enc  = nullptr;

    // GPU卸载的辅助对象
    std::vector<float> inp_mel; // 输入mel数据
    std::vector<float> inp_mask; // 输入mask数据

    // 解码器输出（二维数组：[n_tokens][n_vocab]）
    std::vector<float> logits;

    std::vector<whisper_segment> result_all; // 所有结果
    std::vector<whisper_token>   prompt_past; // 过去的提示

    int lang_id = 0; // 默认为英语

    std::string path_model; // 由whisper_init_from_file_with_params()填充的模型路径

#ifdef WHISPER_USE_COREML
    whisper_coreml_context * ctx_coreml = nullptr; // coreml上下文对象
#endif

#ifdef WHISPER_USE_OPENVINO
    # 创建一个指向whisper_openvino_context类型的指针，并初始化为nullptr
    whisper_openvino_context * ctx_openvino = nullptr;
#endif

    // [EXPERIMENTAL] token-level timestamps data
    int64_t t_beg  = 0;  // 初始化 t_beg 为 0
    int64_t t_last = 0;  // 初始化 t_last 为 0

    whisper_token tid_last;  // 声明一个 whisper_token 类型的变量 tid_last

    std::vector<float> energy; // PCM 信号能量的向量

    // [EXPERIMENTAL] speed-up techniques
    int32_t exp_n_audio_ctx = 0; // 0 - use default  // 初始化 exp_n_audio_ctx 为 0，表示使用默认值
};

struct whisper_context {
    int64_t t_load_us  = 0;  // 初始化 t_load_us 为 0
    int64_t t_start_us = 0;  // 初始化 t_start_us 为 0

    ggml_type wtype = ggml_type::GGML_TYPE_F16; // weight type (FP32 / FP16 / QX)  // 初始化 wtype 为 GGML_TYPE_F16
    ggml_type itype = ggml_type::GGML_TYPE_F16; // intermediate type (FP32 or FP16)  // 初始化 itype 为 GGML_TYPE_F16

    whisper_context_params params;  // 声明一个 whisper_context_params 类型的变量 params

    whisper_model model;  // 声明一个 whisper_model 类型的变量 model
    whisper_vocab vocab;  // 声明一个 whisper_vocab 类型的变量 vocab

    whisper_state * state = nullptr;  // 声明一个指向 whisper_state 类型的指针 state，初始化为 nullptr

    ggml_backend_t backend = nullptr;  // 声明一个 ggml_backend_t 类型的变量 backend，初始化为 nullptr

    std::string path_model; // populated by whisper_init_from_file_with_params()  // 声明一个字符串变量 path_model
};

struct whisper_global {
    // We save the log callback globally
    ggml_log_callback log_callback = whisper_log_callback_default;  // 初始化 log_callback 为 whisper_log_callback_default
    void * log_callback_user_data = nullptr;  // 声明一个指向 void 类型的指针 log_callback_user_data，初始化为 nullptr
};

static whisper_global g_state;  // 声明一个静态的 whisper_global 类型的变量 g_state

template<typename T>
static void read_safe(whisper_model_loader * loader, T & dest) {
    loader->read(loader->context, &dest, sizeof(T));  // 从 loader 中读取数据到 dest
    BYTESWAP_VALUE(dest);  // 对 dest 进行字节交换
}

static bool kv_cache_init(
        const struct whisper_hparams & hparams,
             struct whisper_kv_cache & cache,
                      ggml_backend_t   backend,
                           ggml_type   wtype,
                                 int   n_ctx) {
    const int64_t n_text_state = hparams.n_text_state;  // 获取 hparams 中的 n_text_state
    const int64_t n_text_layer = hparams.n_text_layer;  // 获取 hparams 中的 n_text_layer

    const int64_t n_mem      = n_text_layer*n_ctx;  // 计算 n_mem
    const int64_t n_elements = n_text_state*n_mem;  // 计算 n_elements

    struct ggml_init_params params = {
        /*.mem_size   =*/ 2*ggml_tensor_overhead(),  // 初始化 params 中的 mem_size
        /*.mem_buffer =*/ nullptr,  // 初始化 params 中的 mem_buffer
        /*.no_alloc   =*/ true,  // 初始化 params 中的 no_alloc 为 true
    };

    cache.head = 0;  // 初始化 cache 中的 head 为 0
    cache.size = n_ctx;  // 初始化 cache 中的 size 为 n_ctx

    cache.cells.clear();  // 清空 cache 中的 cells
    cache.cells.resize(n_ctx);  // 重新设置 cache 中的 cells 大小为 n_ctx

    cache.ctx = ggml_init(params);  // 使用 params 初始化 cache 中的 ctx
    // 如果缓存上下文为空，则输出错误信息并返回false
    if (!cache.ctx) {
        WHISPER_LOG_ERROR("%s: failed to allocate memory for kv cache\n", __func__);
        return false;
    }

    // 为缓存的键值对分配一维张量
    cache.k = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);
    cache.v = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);

    // 计算键值对张量所需的内存字节数
    const size_t mem_bytes = ggml_nbytes(cache.k) + ggml_nbytes(cache.v);

    // 在后端分配缓存所需的内存空间
    cache.buffer = ggml_backend_alloc_buffer(backend, mem_bytes);

    // 将张量分配到后端缓存中
    {
        ggml_allocr * alloc = ggml_allocr_new_from_buffer(cache.buffer);

        ggml_allocr_alloc(alloc, cache.k);
        ggml_allocr_alloc(alloc, cache.v);

        ggml_allocr_free(alloc);
    }

    // 返回true表示缓存分配成功
    return true;
# 释放缓存中的资源
static void kv_cache_free(struct whisper_kv_cache & cache) {
    # 如果缓存上下文存在，则释放内存
    if (cache.ctx) {
        ggml_free(cache.ctx);
        # 释放缓存中的后端缓冲区
        ggml_backend_buffer_free(cache.buffer);
        # 将缓存上下文置为空指针
        cache.ctx = nullptr;
    }
}

# 在缓存中查找插槽
static bool whisper_kv_cache_find_slot(
           struct whisper_kv_cache & cache,
        const struct whisper_batch & batch) {
    # 获取缓存大小和批次中的令牌数量
    const uint32_t n_ctx    = cache.size;
    const uint32_t n_tokens = batch.n_tokens;

    # 如果批次中的令牌数量大于缓存大小，则记录错误并返回false
    if (n_tokens > n_ctx) {
        WHISPER_LOG_ERROR("%s: n_tokens=%d > n_ctx=%d\n", __func__, n_tokens, n_ctx);
        return false;
    }

    # 初始化已测试的插槽数量
    uint32_t n_tested = 0;

    # 循环查找可用的插槽
    while (true) {
        # 如果当前插槽不足以容纳批次中的令牌数量，则继续查找
        if (cache.head + n_tokens > n_ctx) {
            n_tested += n_ctx - cache.head;
            cache.head = 0;
            continue;
        }

        # 初始化找到标志
        bool found = true;
        # 遍历批次中的令牌，检查是否有已被占用的插槽
        for (uint32_t i = 0; i < n_tokens; i++) {
            if (cache.cells[cache.head + i].pos >= 0) {
                found = false;
                cache.head += i + 1;
                n_tested   += i + 1;
                break;
            }
        }

        # 如果找到可用的插槽，则跳出循环
        if (found) {
            break;
        }

        # 如果已测试的插槽数量超过了缓存大小，则记录错误并返回false
        if (n_tested >= n_ctx) {
            //WHISPER_LOG_ERROR("%s: failed to find a slot for %d tokens\n", __func__, n_tokens);
            return false;
        }
    }

    # 将批次中的令牌数据写入缓存中的插槽
    for (uint32_t i = 0; i < n_tokens; i++) {
        cache.cells[cache.head + i].pos = batch.pos[i];

        for (int32_t j = 0; j < batch.n_seq_id[i]; j++) {
            cache.cells[cache.head + i].seq_id.insert(batch.seq_id[i][j]);
        }
    }

    return true;
}

# 查找当前已使用的插槽数量
static int32_t whisper_kv_cache_cell_max(const struct whisper_kv_cache & cache) {
    # 从缓存末尾开始遍历，找到第一个被占用的插槽并返回其索引+1
    for (uint32_t i = cache.size - 1; i > 0; --i) {
        if (cache.cells[i].pos >= 0 && !cache.cells[i].seq_id.empty()) {
            return i + 1;
        }
    }

    # 如果没有被占用的插槽，则返回1
    return 1;
}

# 清空缓存
static void whisper_kv_cache_clear(struct whisper_kv_cache & cache) {
    # 遍历缓存中的元素
    for (int32_t i = 0; i < (int32_t) cache.size; ++i) {
        # 将缓存中第i个元素的位置设为-1
        cache.cells[i].pos = -1;
        # 清空缓存中第i个元素的序列ID
        cache.cells[i].seq_id.clear();
    }
    # 将缓存的头部位置设为0
    cache.head = 0;
# 从缓存中删除指定序列 ID 在指定位置范围内的数据
static void whisper_kv_cache_seq_rm(
        struct whisper_kv_cache & cache,
                 whisper_seq_id   seq_id,
                    whisper_pos   p0,
                    whisper_pos   p1) {
    # 初始化新的头指针为缓存的大小
    uint32_t new_head = cache.size;

    # 如果 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    # 如果 p1 小于 0，则将其设置为 whisper_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<whisper_pos>::max();

    # 遍历缓存中的数据
    for (uint32_t i = 0; i < cache.size; ++i) {
        # 如果数据的位置在 p0 和 p1 之间
        if (cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            # 如果序列 ID 小于 0，则清空数据的序列 ID
            if (seq_id < 0) {
                cache.cells[i].seq_id.clear();
            } 
            # 如果数据包含指定的序列 ID，则从数据的序列 ID 中删除该 ID
            else if (cache.cells[i].has_seq_id(seq_id)) {
                cache.cells[i].seq_id.erase(seq_id);
            } 
            # 如果数据不包含指定的序列 ID，则继续下一次循环
            else {
                continue;
            }
            # 如果数据的序列 ID 为空，则将数据的位置设置为 -1
            if (cache.cells[i].seq_id.empty()) {
                cache.cells[i].pos = -1;
                # 如果新的头指针还未更新，则将其设置为当前数据的索引
                if (new_head == cache.size) new_head = i;
            }
        }
    }

    # 如果释放了一个数据槽位，则将头指针设置为该位置，以便搜索可以从那里开始
    if (new_head != cache.size) cache.head = new_head;
}

# 将指定序列 ID 在指定位置范围内的数据复制到另一个序列 ID
static void whisper_kv_cache_seq_cp(
        struct whisper_kv_cache & cache,
                 whisper_seq_id   seq_id_src,
                 whisper_seq_id   seq_id_dst,
                    whisper_pos   p0,
                    whisper_pos   p1) {
    # 如果 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    # 如果 p1 小于 0，则将其设置为 whisper_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<whisper_pos>::max();

    # 将头指针设置为 0
    cache.head = 0;

    # 遍历缓存中的数据
    for (uint32_t i = 0; i < cache.size; ++i) {
        # 如果数据包含指定的序列 ID 并且位置在 p0 和 p1 之间，则将目标序列 ID 插入到数据的序列 ID 中
        if (cache.cells[i].has_seq_id(seq_id_src) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            cache.cells[i].seq_id.insert(seq_id_dst);
        }
    }
}

# 初始化 Whisper 后端
static ggml_backend_t whisper_backend_init(const whisper_context_params & params) {
    # 初始化 GPU 后端为 NULL
    ggml_backend_t backend_gpu = NULL;

    # 初始化后端
#ifdef GGML_USE_CUBLAS
    # 如果参数中指定使用 GPU，并且 CUDA 库已加载
    if (params.use_gpu && ggml_cublas_loaded()) {
        # 输出信息日志，指示正在使用 CUDA 后端
        WHISPER_LOG_INFO("%s: using CUDA backend\n", __func__);
        # 初始化 CUDA 后端
        backend_gpu = ggml_backend_cuda_init(0);
        # 如果初始化失败，输出错误日志
        if (!backend_gpu) {
            WHISPER_LOG_ERROR("%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#ifdef GGML_USE_METAL
    // 如果使用 GPU，则使用 Metal 后端
    if (params.use_gpu) {
        WHISPER_LOG_INFO("%s: using Metal backend\n", __func__);
        // 设置 Metal 后端的日志回调函数
        ggml_metal_log_set_callback(whisper_log_callback_default, nullptr);
        // 初始化 Metal 后端
        backend_gpu = ggml_backend_metal_init();
        // 如果初始化失败，则记录错误日志
        if (!backend_gpu) {
            WHISPER_LOG_ERROR("%s: ggml_backend_metal_init() failed\n", __func__);
        } else if (!ggml_backend_metal_supports_family(backend_gpu, 7)) {
            // 如果 Metal GPU 不支持 family 7，则记录错误日志并回退到 CPU
            WHISPER_LOG_ERROR("%s: Metal GPU does not support family 7 - falling back to CPU\n", __func__);
            ggml_backend_free(backend_gpu);
            backend_gpu = NULL;
        }
    }
#endif

    // 如果存在 GPU 后端，则返回 GPU 后端
    if (backend_gpu) {
        return backend_gpu;
    }
    // 否则返回 CPU 后端
    return ggml_backend_cpu_init();
}

// 从 ggml 文件中加载模型
//
// 文件格式:
//
//   - hparams
//   - 预先计算的 mel 滤波器
//   - 词汇表
//   - 权重
//
// 详细信息请参见 convert-pt-to-ggml.py 脚本
//
static bool whisper_model_load(struct whisper_model_loader * loader, whisper_context & wctx) {
    WHISPER_LOG_INFO("%s: loading model\n", __func__);

    const int64_t t_start_us = ggml_time_us();

    wctx.t_start_us = t_start_us;

    auto & model = wctx.model;
    auto & vocab = wctx.vocab;

    // 验证魔数
    {
        uint32_t magic;
        read_safe(loader, magic);
        // 如果魔数不匹配，则记录错误日志并返回 false
        if (magic != GGML_FILE_MAGIC) {
            WHISPER_LOG_ERROR("%s: invalid model data (bad magic)\n", __func__);
            return false;
        }
    }

    // 加载 hparams
    }

    // 加载 mel 滤波器
    {
        auto & filters = wctx.model.filters;

        read_safe(loader, filters.n_mel);
        read_safe(loader, filters.n_fft);

        // 调整滤波器数据的大小，并读取数据
        filters.data.resize(filters.n_mel * filters.n_fft);
        loader->read(loader->context, filters.data.data(), filters.data.size() * sizeof(float));
        BYTESWAP_FILTERS(filters);
    }

    // 加载词汇表
    }

    // 设置 wtype 为 wctx 的 wtype
    const ggml_type wtype = wctx.wtype;
    // 定义变量vtype，如果wctx.wtype等于GGML_TYPE_F32，则vtype等于GGML_TYPE_F32，否则等于GGML_TYPE_F16
    const ggml_type vtype = wctx.wtype == GGML_TYPE_F32 ? GGML_TYPE_F32 : GGML_TYPE_F16; // conv type

    // 创建ggml上下文
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;

        // 获取音频层和文本层的数量
        const int n_audio_layer = hparams.n_audio_layer;
        const int n_text_layer  = hparams.n_text_layer;

        // 计算需要分配的张量数量
        const size_t n_tensors = 10 /* input */ + 15 + 15*n_audio_layer + 24*n_text_layer;

        // 初始化ggml_init_params结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ n_tensors*ggml_tensor_overhead(),
            /*.mem_buffer =*/ nullptr,
            /*.no_alloc   =*/ true,
        };

        // 初始化ggml上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，则记录错误信息并返回false
        if (!model.ctx) {
            WHISPER_LOG_ERROR("%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 准备权重的张量
    }

    // 初始化whisper后端
    wctx.backend = whisper_backend_init(wctx.params);

    {
        // 计算主要缓冲区的大小
        size_t size_main = 0;

        // 遍历模型的张量，计算总大小
        for (const auto & t : model.tensors) {
            size_main += ggml_nbytes(t.second) + ggml_tensor_overhead();
        }

        // 在后端分配缓冲区
        model.buffer = ggml_backend_alloc_buffer(wctx.backend, size_main);

        // 记录缓冲区大小信息
        WHISPER_LOG_INFO("%s: %8s buffer size = %8.2f MB\n", __func__, ggml_backend_name(wctx.backend), size_main / 1e6);
    }

    // 从缓冲区创建分配器
    ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer);

    // 在后端缓冲区中分配张量
    {
        for (const auto & t : model.tensors) {
            ggml_allocr_alloc(alloc, t.second);
        }
    }

    // 加载权重
#ifdef GGML_USE_METAL
                || ggml_backend_is_metal(backend)
#endif
                )) {
                // 如果定义了 GGML_USE_METAL 或者当前后端是 Metal，则执行以下代码块
                // 对于 CPU 和 Metal 后端，我们可以直接读取到张量中
                loader->read(loader->context, tensor->data, ggml_nbytes(tensor));
                BYTESWAP_TENSOR(tensor);
            } else {
                // 否则，先读入一个临时缓冲区，然后再复制到设备内存中
                read_buf.resize(ggml_nbytes(tensor));

                loader->read(loader->context, read_buf.data(), read_buf.size());

                ggml_backend_tensor_set(tensor, read_buf.data(), 0, ggml_nbytes(tensor));
            }

            //printf("%48s - [%5d, %5d, %5d], type = %6s, %6.2f MB\n", name.data(), ne[0], ne[1], ne[2], ggml_type_name((ggml_type) ttype), ggml_nbytes(tensor)/1e6);
            // 打印张量的名称、维度、类型和大小
            total_size += ggml_nbytes(tensor);
            model.n_loaded++;
        }

        WHISPER_LOG_INFO("%s: model size    = %7.2f MB\n", __func__, total_size/1e6);
        // 记录模型的大小信息

        if (model.n_loaded == 0) {
            WHISPER_LOG_WARN("%s: WARN no tensors loaded from model file - assuming empty model for testing\n", __func__);
            // 如果没有从模型文件中加载张量，则记录警告信息
        } else if (model.n_loaded != (int) model.tensors.size()) {
            WHISPER_LOG_ERROR("%s: ERROR not all tensors loaded from model file - expected %zu, got %d\n", __func__, model.tensors.size(), model.n_loaded);
            return false;
            // 如果加载的张量数量与模型文件中的张量数量不一致，则记录错误信息并返回 false
        }
    }

    ggml_allocr_free(alloc);
    // 释放分配的内存

    wctx.t_load_us = ggml_time_us() - t_start_us;
    // 记录加载模型所花费的时间

    return true;
    // 返回 true 表示加载模型成功

}

static bool whisper_encode_external(const whisper_state & wstate) {
    GGML_UNUSED(wstate);

#ifndef WHISPER_USE_COREML
    const bool use_coreml = false;
#else
    const bool use_coreml = wstate.ctx_coreml != nullptr;
#endif

#ifndef WHISPER_USE_OPENVINO
    const bool use_openvino = false;
#else
    const bool use_openvino = wstate.ctx_openvino != nullptr;
#endif

    return use_coreml || use_openvino;
    // 返回是否使用 CoreML 或 OpenVINO 的标志
}
// 构建卷积神经网络的计算图
static struct ggml_cgraph * whisper_build_graph_conv(
        whisper_context & wctx,
        whisper_state & wstate,
        const int   mel_offset) {
    // 获取模型、mel输入和超参数的引用
    const auto & model   = wctx.model;
    const auto & mel_inp = wstate.mel;
    const auto & hparams = model.hparams;

    // 计算上下文大小和音频状态大小
    const int n_ctx   = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;
    const int n_state = hparams.n_audio_state; GGML_UNUSED(n_state);

    // 获取mel频谱的维度
    const int n_mels = hparams.n_mels;

    // 初始化参数结构体
    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_conv.meta.size(),
        /*.mem_buffer =*/ wstate.alloc_conv.meta.data(),
        /*.no_alloc   =*/ true,
    };

    // 初始化上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建计算图
    ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 获取分配器
    ggml_allocr * alloc = wstate.alloc_conv.alloc;

    // 创建mel张量
    struct ggml_tensor * mel = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, 2*n_ctx, n_mels);
    ggml_allocr_alloc(alloc, mel);

    // 断言mel张量的类型为GGML_TYPE_F32
    assert(mel->type == GGML_TYPE_F32);
    // 如果分配器不是测量模式
    if (!ggml_allocr_is_measure(alloc)) {
        // 断言输入mel的频谱维度与n_mels相同
        assert(mel_inp.n_mel == n_mels);

        // 调整输入mel的大小
        wstate.inp_mel.resize(ggml_nelements(mel));

        float * dst = wstate.inp_mel.data();
        // 将输入mel数据初始化为0
        memset(dst, 0, ggml_nbytes(mel));

        // 计算i0和i1的值
        const int i0 = std::min(mel_offset,           mel_inp.n_len);
        const int i1 = std::min(mel_offset + 2*n_ctx, mel_inp.n_len);

        // 将mel输入数据复制到输入mel中
        for (int j = 0; j < mel_inp.n_mel; ++j) {
            for (int i = i0; i < i1; ++i) {
                dst[j*2*n_ctx + (i - i0)] = mel_inp.data[j*mel_inp.n_len + i];
            }
        }

        // 设置mel张量的数据
        ggml_backend_tensor_set(mel, wstate.inp_mel.data(), 0, ggml_nelements(mel)*sizeof(float));
    }

    struct ggml_tensor * cur = nullptr;
}
    # 如果外部编码失败，则执行以下操作
    if (!whisper_encode_external(wstate)) {
        # 执行卷积操作并使用GELU激活函数
        {
            # 使用1维卷积操作
            cur = ggml_conv_1d_ph(ctx0, model.e_conv_1_w, mel, 1, 1);
            # 添加偏置
            cur = ggml_add(ctx0, cur, model.e_conv_1_b);

            # 使用GELU激活函数
            cur = ggml_gelu(ctx0, cur);

            # 使用1维卷积操作
            cur = ggml_conv_1d_ph(ctx0, model.e_conv_2_w, cur, 2, 1);
            # 添加偏置
            cur = ggml_add(ctx0, cur, model.e_conv_2_b);

            # 使用GELU激活函数
            cur = ggml_gelu(ctx0, cur);
        }

        # 设置当前操作的名称为"embd_conv"
        ggml_set_name(cur, "embd_conv");
        # 将当前操作赋值给wstate.embd_conv
        wstate.embd_conv = cur;
    } else {
#ifdef WHISPER_USE_COREML
        // 如果使用 CoreML
        cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx);
        // 创建一个新的二维浮点类型的张量
        ggml_allocr_alloc(alloc, cur);
        // 分配张量的内存空间

        if (!ggml_allocr_is_measure(alloc)) {
            // 如果不是测量模式
            whisper_coreml_encode(wstate.ctx_coreml, mel->ne[0], mel->ne[1], (float *) mel->data, (float *) cur->data);
            // 使用 CoreML 对数据进行编码
        }
#endif
#ifdef WHISPER_USE_OPENVINO
        // 如果使用 OpenVINO
        cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx);
        // 创建一个新的二维浮点类型的张量
        ggml_allocr_alloc(alloc, cur);
        // 分配张量的内存空间

        if (!ggml_allocr_is_measure(alloc)) {
            // 如果不是测量模式
            whisper_openvino_encode(wstate.ctx_openvino, mel, cur);
            // 使用 OpenVINO 对数据进行编码
        }
#endif

        ggml_set_name(cur, "embd_enc");
        // 设置张量的名称为 "embd_enc"
        wstate.embd_enc = cur;
        // 将当前张量赋值给 wstate.embd_enc
    }

    ggml_build_forward_expand(gf, cur);
    // 构建前向扩展

    ggml_free(ctx0);
    // 释放上下文

    return gf;
    // 返回图形
}

static struct ggml_cgraph * whisper_build_graph_encoder(
        whisper_context & wctx,
          whisper_state & wstate) {
    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    const int n_ctx   = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;
    const int n_state = hparams.n_audio_state;
    const int n_head  = hparams.n_audio_head;
    const int n_layer = hparams.n_audio_layer;

    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_encode.meta.size(),
        /*.mem_buffer =*/ wstate.alloc_encode.meta.data(),
        /*.no_alloc   =*/ true,
    };
    // 初始化参数结构体

    struct ggml_context * ctx0 = ggml_init(params);
    // 初始化上下文

    ggml_cgraph * gf = ggml_new_graph_custom(ctx0, WHISPER_MAX_NODES, false);
    // 创建自定义图形

    //ggml_allocr * alloc = wstate.alloc_encode.alloc;

    //struct ggml_tensor * cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_ctx, n_state);
    //ggml_allocr_alloc(alloc, cur);

    //if (!ggml_allocr_is_measure(alloc)) {
    //    ggml_backend_tensor_copy(wstate.embd_conv, cur);
    //}
    struct ggml_tensor * cur = ggml_view_tensor(ctx0, wstate.embd_conv);
    // 创建一个新的张量，用于查看 wstate.embd_conv 的张量

    const float KQscale = 1.0f/sqrtf(float(n_state)/n_head);
    // 计算 KQscale
    // ===================================================================
    // 注意：对编码器进行部分求值的实验（忽略）
    //static int iter = -1;
    //const int n_iter = 1500/n_ctx;

    //iter = (iter + 1) % n_iter;

    //if (iter == 0) {
    //    memset(model.memory_cross_k->data, 0, ggml_nbytes(model.memory_cross_k));
    //    memset(model.memory_cross_v->data, 0, ggml_nbytes(model.memory_cross_v));
    //}

    // 将迭代次数初始化为0
    static int iter = 0;

    // 计算偏移量和步长
    const size_t e_pe_stride = model.e_pe->ne[0]*ggml_element_size(model.e_pe);
    const size_t e_pe_offset = model.e_pe->ne[0]*ggml_element_size(model.e_pe)*n_ctx*iter;

    // 创建一个视图，用于对输入张量进行操作
    struct ggml_tensor * e_pe = ggml_view_2d(ctx0, model.e_pe, model.e_pe->ne[0], n_ctx, e_pe_stride, e_pe_offset);
    // 将当前张量与 e_pe 相加，并将结果赋给 cur
    cur = ggml_add(ctx0, e_pe, ggml_cont(ctx0, ggml_transpose(ctx0, cur)));

    // ===================================================================

    // 原始代码：
    //cur = ggml_add(ctx0, model.e_pe, ggml_transpose(ctx0, cur));

    // 将 cur 赋给 inpL
    struct ggml_tensor * inpL = cur;
    // 遍历每一层的编码器
    for (int il = 0; il < n_layer; ++il) {
        // 获取当前层的引用
        const auto & layer = model.layers_encoder[il];

        // norm
        {
            // 对输入进行归一化处理
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // 对归一化后的输入进行线性变换和偏置处理
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0, cur, layer.attn_ln_0_w),
                    layer.attn_ln_0_b);
        }

        // self-attention
        {
            // 计算当前输入的查询向量
            struct ggml_tensor * Qcur = ggml_mul_mat(ctx0,
                    layer.attn_q_w,
                    cur);

            // 对查询向量进行偏置处理
            Qcur = ggml_add(ctx0, Qcur, layer.attn_q_b);

            // 计算当前输入的键向量，注意：键向量没有偏置
            struct ggml_tensor * Kcur = ggml_mul_mat(ctx0,
                    layer.attn_k_w,
                    cur);

            // 计算当前输入的值向量
            struct ggml_tensor * Vcur = ggml_mul_mat(ctx0,
                    layer.attn_v_w,
                    cur);

            // 对值向量进行线性变换和偏置处理
            Vcur = ggml_add(ctx0, Vcur, layer.attn_v_b);

            // ------
#ifdef WHISPER_USE_FLASH_ATTN
            // 根据当前的 Q 张量创建一个新的张量，维度顺序为 (n_state/n_head, n_head, n_ctx)
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // 根据当前的 K 张量创建一个新的张量，维度顺序为 (n_state/n_head, n_head, n_ctx)
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Kcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // 根据当前的 V 张量创建一个新的张量，维度顺序为 (n_ctx, n_state/n_head, n_head)
            struct ggml_tensor * V =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                Vcur,
                                n_state/n_head, n_head, n_ctx),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, wctx.itype, n_ctx, n_state/n_head, n_head));

            // 使用 Q、K、V 张量进行 Flash Attention 操作，得到 KQV 张量
            struct ggml_tensor * KQV = ggml_flash_attn(ctx0, Q, K, V, false);
#else
            // 创建一个新的张量 Q，通过对当前张量 Qcur 进行复制和排列
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // 创建一个新的张量 K，通过对当前张量 Kcur 进行复制和排列
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Kcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // 计算 K 和 Q 的矩阵乘法
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // 对 KQ 进行缩放
            struct ggml_tensor * KQ_scaled = ggml_scale(ctx0, KQ, KQscale);

            // 对缩放后的 KQ 进行 Softmax 操作
            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_scaled);

            // 创建一个新的张量 V，通过对当前张量 Vcur 进行复制、重塑和排列
            struct ggml_tensor * V =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                Vcur,
                                n_state/n_head, n_head, n_ctx),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, wctx.itype, n_ctx, n_state/n_head, n_head)
                        );

            // 计算 V 和 KQ_soft_max 的矩阵乘法
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);
        # 对输入张量 KQV 进行维度置换操作，得到 KQV_merged 张量
        struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

        # 复制 KQV_merged 张量，得到 cur 张量
        cur = ggml_cpy(ctx0,
                KQV_merged,
                ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx));
    }

    // projection
    {
        # 使用 layer.attn_ln_1_w 对 cur 进行矩阵乘法
        cur = ggml_mul_mat(ctx0,
                layer.attn_ln_1_w,
                cur);

        # 对 cur 进行加法操作，加上 layer.attn_ln_1_b 张量
        cur = ggml_add(ctx0, cur, layer.attn_ln_1_b);
    }

    # 加上输入张量 inpL
    cur = ggml_add(ctx0, cur, inpL);

    # 将 cur 赋值给 inpFF
    struct ggml_tensor * inpFF = cur;

    // feed-forward network
    {
        // norm
        {
            # 对 inpFF 进行归一化操作
            cur = ggml_norm(ctx0, inpFF, hparams.eps);

            # 对 cur 进行线性变换和加法操作
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0, cur, layer.mlp_ln_w),
                    layer.mlp_ln_b);
        }

#ifdef WHISPER_USE_FLASH_FF
        # 使用 ggml_flash_ff 函数进行前馈传播
        cur = ggml_flash_ff(ctx0,
                ggml_cpy(ctx0, cur, ggml_new_tensor_2d(ctx0, wstate.itype, n_state, n_ctx)),
                layer.mlp_0_w, layer.mlp_0_b, layer.mlp_1_w, layer.mlp_1_b);
#else
        // fully connected
        # 对 cur 进行矩阵乘法
        cur = ggml_mul_mat(ctx0,
                layer.mlp_0_w,
                cur);

        # 对 cur 进行加法操作，加上 layer.mlp_0_b 张量
        cur = ggml_add(ctx0, cur, layer.mlp_0_b);

        // GELU activation
        # 对 cur 进行 GELU 激活函数操作
        cur = ggml_gelu(ctx0, cur);

        // projection
        # 对 cur 进行矩阵乘法
        cur = ggml_mul_mat(ctx0,
                layer.mlp_1_w,
                cur);

        # 对 cur 进行加法操作，加上 layer.mlp_1_b 张量
        cur = ggml_add(ctx0, cur, layer.mlp_1_b);
#endif
    }

    # 将 cur 和 inpFF 进行加法操作
    inpL = ggml_add(ctx0, cur, inpFF);
}

# 将 inpL 赋值给 cur
cur = inpL;

// norm
{
    # 对 cur 进行归一化操作
    cur = ggml_norm(ctx0, cur, hparams.eps);

    # 对 cur 进行线性变换和加法操作
    cur = ggml_add(ctx0,
            ggml_mul(ctx0, cur, model.e_ln_w),
            model.e_ln_b);
}

# 构建前向传播扩展
ggml_build_forward_expand(gf, cur);

# 将 cur 赋值给 wstate.embd_enc
wstate.embd_enc = cur;

# 打印图形结构
#ggml_graph_print(gf);
    // 释放内存
    ggml_free(ctx0);
    // 返回结果
    return gf;
// 预先计算交叉注意力内存
static struct ggml_cgraph * whisper_build_graph_cross(
        whisper_context & wctx,
        whisper_state & wstate) {
    // 获取模型和超参数
    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    // 获取上下文大小、状态大小和头部数量
    const int n_ctx   = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;
    const int n_state = hparams.n_audio_state;
    const int n_head  = hparams.n_audio_head;

    // 初始化参数
    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_cross.meta.size(),
        /*.mem_buffer =*/ wstate.alloc_cross.meta.data(),
        /*.no_alloc   =*/ true,
    };

    // 初始化上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建新的计算图
    ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 创建当前张量视图
    struct ggml_tensor * cur = ggml_view_tensor(ctx0, wstate.embd_enc);

    // 计算 Kscale
    const float  Kscale = pow(float(n_state) / n_head, -0.25);
    // 遍历模型中的文本层
    for (int il = 0; il < model.hparams.n_text_layer; ++il) {
        // 获取当前文本层的引用
        auto & layer = model.layers_decoder[il];

        // 计算交叉注意力的 K 值
        struct ggml_tensor* Kcross = ggml_mul_mat(ctx0,
                layer.cross_attn_k_w,
                cur);

        // 对 K 值进行缩放
        Kcross = ggml_scale(ctx0, Kcross, Kscale);

        // 计算交叉注意力的 V 值
        struct ggml_tensor* Vcross = ggml_mul_mat(ctx0,
                layer.cross_attn_v_w,
                cur);

        // 对 V 值进行偏置
        Vcross = ggml_add(ctx0,
                    Vcross,
                    layer.cross_attn_v_b);

        // 对 V 值进行转置和重塑
        Vcross = ggml_transpose(ctx0, ggml_reshape_2d(ctx0, Vcross, n_state, n_ctx));

        // 获取 K 值的视图
        struct ggml_tensor * k = ggml_view_1d(ctx0, wstate.kv_cross.k,
                n_state*n_ctx,
                (ggml_element_size(wstate.kv_cross.k)*n_state)*(il*n_ctx));

        // 获取 V 值的视图
        struct ggml_tensor * v = ggml_view_2d(ctx0, wstate.kv_cross.v, n_ctx, n_state,
                (   n_ctx)*ggml_element_size(wstate.kv_cross.v),
                (il*n_ctx)*ggml_element_size(wstate.kv_cross.v)*n_state);

        // 构建前向传播的扩展
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcross, k));
        ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcross, v));
    }

    // 释放上下文
    ggml_free(ctx0);

    // 返回图形
    return gf;
// 评估给定状态下的编码器
//
// 给定音频录制（更具体地说，它的对数梅尔频谱图），运行编码器部分的前向传递，返回编码后的特征
//
//   - wctx:      模型
//   - wstate:     编码器的状态
//   - n_threads:  要使用的线程数
//   - mel_offset: 梅尔频谱图中的偏移（即音频偏移）
//
static bool whisper_encode_internal(
        whisper_context & wctx,
          whisper_state & wstate,
              const int   mel_offset,
              const int   n_threads,
 whisper_abort_callback   abort_callback,
                   void * abort_callback_data) {
    const int64_t t_start_us = ggml_time_us();

    // conv
    {
        auto & alloc = wstate.alloc_conv.alloc;

        ggml_allocr_reset(alloc);

        ggml_cgraph * gf = whisper_build_graph_conv(wctx, wstate, mel_offset);

        ggml_allocr_alloc_graph(alloc, gf);

        if (!whisper_encode_external(wstate)) {
            if (!ggml_graph_compute_helper(wstate.backend, gf, n_threads)) {
                return false;
            }
        }
    }

    // encoder
    if (!whisper_encode_external(wstate)) {
        auto & alloc = wstate.alloc_encode.alloc;

        ggml_allocr_reset(alloc);

        ggml_cgraph * gf = whisper_build_graph_encoder(wctx, wstate);

        ggml_allocr_alloc_graph(alloc, gf);

        if (!ggml_graph_compute_helper(wstate.backend, gf, n_threads)) {
            return false;
        }
    }

    // cross
    {
        auto & alloc = wstate.alloc_cross.alloc;

        ggml_allocr_reset(alloc);

        ggml_cgraph * gf = whisper_build_graph_cross(wctx, wstate);

        ggml_allocr_alloc_graph(alloc, gf);

        if (!ggml_graph_compute_helper(wstate.backend, gf, n_threads)) {
            return false;
        }
    }

    wstate.t_encode_us += ggml_time_us() - t_start_us;
    wstate.n_encode++;
}
    # 返回一个布尔值，表示是否需要中止操作
    return !(abort_callback && abort_callback(abort_callback_data));
    # 构建解码器的计算图
    static struct ggml_cgraph * whisper_build_graph_decoder(
         whisper_context & wctx,  # 传入 whisper_context 对象的引用
         whisper_state   & wstate,  # 传入 whisper_state 对象的引用
     const whisper_batch & batch) {  # 传入 whisper_batch 对象的常量引用
    const auto & model   = wctx.model;  # 获取 wctx 对象中的 model 属性
    const auto & hparams = model.hparams;  # 获取 model 对象中的 hparams 属性

    auto & kv_self = wstate.kv_self;  # 获取 wstate 对象中的 kv_self 属性

    WHISPER_ASSERT(!!kv_self.ctx);  # 断言 kv_self.ctx 不为空

    ggml_allocr * alloc = wstate.alloc_decode.alloc;  # 获取 wstate.alloc_decode.alloc 属性

    const int n_ctx   = kv_self.size;  # 获取 kv_self.size 属性
    const int n_state = hparams.n_text_state;  # 获取 hparams.n_text_state 属性
    const int n_head  = hparams.n_text_head;  # 获取 hparams.n_text_head 属性
    const int n_layer = hparams.n_text_layer;  # 获取 hparams.n_text_layer 属性

    const int n_tokens    = batch.n_tokens;  # 获取 batch.n_tokens 属性
    const int n_audio_ctx = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;  # 根据条件获取 n_audio_ctx 属性

    const int32_t n_kv     = ggml_allocr_is_measure(alloc) ? n_ctx            : kv_self.n;  # 根据条件获取 n_kv 属性
    const int32_t kv_head  = ggml_allocr_is_measure(alloc) ? n_ctx - n_tokens : kv_self.head;  # 根据条件获取 kv_head 属性

    # 初始化参数结构体
    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_decode.meta.size(),  # 获取 wstate.alloc_decode.meta 的大小
        /*.mem_buffer =*/ wstate.alloc_decode.meta.data(),  # 获取 wstate.alloc_decode.meta 的数据
        /*.no_alloc   =*/ true,  # 设置 no_alloc 为 true
    };

    # 初始化上下文对象
    struct ggml_context * ctx0 = ggml_init(params);

    # 创建计算图对象
    ggml_cgraph * gf = ggml_new_graph_custom(ctx0, WHISPER_MAX_NODES, false);

    # 创建并分配内存给 embd 张量
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
    ggml_allocr_alloc(alloc, embd);

    # 如果不是测量模式，则设置 embd 张量的值
    if (!ggml_allocr_is_measure(alloc)) {
        ggml_backend_tensor_set(embd, batch.token, 0, n_tokens*ggml_element_size(embd));
    }

    # 创建并分配内存给 position 张量
    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
    ggml_allocr_alloc(alloc, position);

    # 如果不是测量模式，则设置 position 张量的值
    if (!ggml_allocr_is_measure(alloc)) {
        for (int i = 0; i < n_tokens; ++i) {
            const int32_t val = batch.pos[i];
            ggml_backend_tensor_set(position, &val, i*sizeof(int32_t), sizeof(int32_t));
        }
    }
    # 计算 KQscale 的值，用于后续计算
    const float KQscale = pow(float(n_state)/n_head, -0.25);

    # 创建一个 3 维的浮点数类型的张量 KQ_mask
    struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
    # 分配内存给 KQ_mask
    ggml_allocr_alloc(alloc, KQ_mask);

    # 如果分配器不是度量器
    if (!ggml_allocr_is_measure(alloc)) {
        # 调整输入掩码的大小
        wstate.inp_mask.resize(n_kv*n_tokens);

        # 获取输入掩码的数据指针，并将其初始化为 0
        float * data = wstate.inp_mask.data();
        memset(data, 0, ggml_nbytes(KQ_mask));

        # 遍历输入掩码的维度
        for (int h = 0; h < 1; ++h) {
            for (int j = 0; j < n_tokens; ++j) {
                # 获取批处理中的位置和序列 ID
                const whisper_pos    pos    = batch.pos[j];
                const whisper_seq_id seq_id = batch.seq_id[j][0];

                # 遍历输入掩码的维度
                for (int i = 0; i < n_kv; ++i) {
                    # 如果 kv_self 中的单元格不包含序列 ID 或者位置大于当前位置，则将输入掩码的值设置为负无穷
                    if (!kv_self.cells[i].has_seq_id(seq_id) || kv_self.cells[i].pos > pos) {
                        data[h*(n_kv*n_tokens) + j*n_kv + i] = -INFINITY;
                    }
                }
            }
        }

        # 将输入掩码的数据拷贝到 KQ_mask 中
        ggml_backend_tensor_set(KQ_mask, wstate.inp_mask.data(), 0, ggml_nelements(KQ_mask)*sizeof(float));
    }

    # 计算当前张量 cur
    struct ggml_tensor * cur =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.d_te, embd),
                ggml_get_rows(ctx0, model.d_pe, position));

    # 将当前张量 cur 赋值给 inpL
    struct ggml_tensor * inpL = cur;

    }

    # 将当前张量 cur 赋值给 inpL
    cur = inpL;

    # 归一化
    {
        # 对当前张量 cur 进行归一化
        cur = ggml_norm(ctx0, cur, hparams.eps);

        # 对当前张量 cur 进行加法和乘法操作
        cur = ggml_add(ctx0,
                ggml_mul(ctx0,
                    cur,
                    model.d_ln_w),
                model.d_ln_b);
    }

    # 仅为最后一个标记计算对数
    # 将此行注释掉以计算所有 n_tokens 的对数
    # 可能在将来会有用
    #cur = ggml_view_2d(ctx0, cur, cur->ne[0], 1, cur->nb[1], (cur->ne[1] - 1)*cur->nb[1]);

    # 计算 logits
    struct ggml_tensor * logits = ggml_mul_mat(ctx0, model.d_te, cur);

    # 构建前向扩展
    ggml_build_forward_expand(gf, logits);

    # 释放上下文
    ggml_free(ctx0);

    # 返回 gf
    return gf;
// 评估解码器
//
// 给定文本提示 + 音频特征 -> 计算下一个标记的对数
//
//   - model:      模型
//   - n_threads:  要使用的线程数
//   - tokens:     文本提示
//   - n_tokens:   提示中的标记数
//   - n_past:     用于前缀提示的过去标记数
//
static bool whisper_decode_internal(
        whisper_context & wctx,
          whisper_state & wstate,
    const whisper_batch & batch,
              const int   n_threads,
 whisper_abort_callback   abort_callback,
                   void * abort_callback_data) {
    const int64_t t_start_us = ggml_time_us();

    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    const int n_vocab  = hparams.n_vocab;
    const int n_tokens = batch.n_tokens;

    auto & logits_out = wstate.logits;

    struct ggml_tensor * logits;

    // 为批次找到 KV 槽
    {
        auto & kv_self = wstate.kv_self;

        if (!whisper_kv_cache_find_slot(kv_self, batch)) {
            return false;
        }

        kv_self.n = whisper_kv_cache_cell_max(kv_self);
        //kv_self.n = std::min((int32_t) hparams.n_text_ctx, std::max(32, whisper_kv_cache_cell_max(kv_self)));
        //printf("n_tokens = %5d, kv_self.head = %5d, kv_self.n = %5d, seq_id = %5d\n", batch.n_tokens, kv_self.head, kv_self.n, batch.seq_id[0][0]);
    }

    // 解码器
    {
        auto & alloc = wstate.alloc_decode.alloc;

        ggml_allocr_reset(alloc);

        ggml_cgraph * gf = whisper_build_graph_decoder(wctx, wstate, batch);

        ggml_allocr_alloc_graph(alloc, gf);

        logits = gf->nodes[gf->n_nodes - 1];

        if (!ggml_graph_compute_helper(wstate.backend, gf, n_threads)) {
            return false;
        }
    }

    logits_out.resize(n_tokens*n_vocab);
    // 遍历 n_tokens 次循环
    for (int i = 0; i < n_tokens; i++) {
        // 如果 batch.logits[i] 等于 0，则跳过当前循环
        if (batch.logits[i] == 0) {
            continue;
        }
        // 从 logits 中获取数据，并存储到 logits_out 中的指定位置
        ggml_backend_tensor_get(logits, logits_out.data() + (n_vocab*i), sizeof(float)*(n_vocab*i), sizeof(float)*n_vocab);
    }

    // 如果 batch 中的 n_tokens 大于 1
    if (batch.n_tokens > 1) {
        // 打印函数名和内存使用情况
        //printf("%s: used_mem = %f MB, %f MB, %f MB %f MB %f MB\n", __func__,
        //        ggml_used_mem(ctx0)/1e6,
        //        wstate.get_buf_max_mem(0)/1e6,
        //        wstate.get_buf_max_mem(1)/1e6,
        //        wstate.get_buf_max_mem(2)/1e6,
        //        wstate.get_buf_max_mem(3)/1e6);
    }

    // 如果 batch 中的 n_tokens 等于 1
    if (batch.n_tokens == 1) {
        // 更新解码时间和解码次数
        wstate.t_decode_us += ggml_time_us() - t_start_us;
        wstate.n_decode++;
    } else if (batch.n_tokens < 16) {
        // 更新批处理时间和批处理次数
        wstate.t_batchd_us += ggml_time_us() - t_start_us;
        wstate.n_batchd += n_tokens;
    } else {
        // 更新提示时间和提示次数
        wstate.t_prompt_us += ggml_time_us() - t_start_us;
        wstate.n_prompt += n_tokens;
    }

    // 返回是否中止回调函数的结果
    return !(abort_callback && abort_callback(abort_callback_data));
// 以毫秒为单位的时间戳转换为格式化的时间字符串，可选择是否使用逗号作为小数点分隔符
static std::string to_timestamp(int64_t t, bool comma = false) {
    // 将时间戳转换为毫秒
    int64_t msec = t * 10;
    // 计算小时
    int64_t hr = msec / (1000 * 60 * 60);
    msec = msec - hr * (1000 * 60 * 60);
    // 计算分钟
    int64_t min = msec / (1000 * 60);
    msec = msec - min * (1000 * 60);
    // 计算秒
    int64_t sec = msec / 1000;
    msec = msec - sec * 1000;

    // 格式化时间字符串
    char buf[32];
    snprintf(buf, sizeof(buf), "%02d:%02d:%02d%s%03d", (int) hr, (int) min, (int) sec, comma ? "," : ".", (int) msec);

    return std::string(buf);
}

// 预先计算正弦和余弦值，以加速傅立叶变换过程
static void fill_sin_cos_table() {
    static bool is_filled = false;
    if (is_filled) return;
    // 计算正弦和余弦值
    for (int i = 0; i < SIN_COS_N_COUNT; i++) {
        double theta = (2*M_PI*i)/SIN_COS_N_COUNT;
        sin_vals[i] = sinf(theta);
        cos_vals[i] = cosf(theta);
    }
    is_filled = true;
}

// 简单的离散傅立叶变换，输入为实数，输出为复数
static void dft(const std::vector<float> & in, std::vector<float> & out) {
    int N = in.size();

    // 为输出分配空间
    out.resize(N*2);
    const int sin_cos_step = SIN_COS_N_COUNT / N;

    for (int k = 0; k < N; k++) {
        float re = 0;
        float im = 0;

        for (int n = 0; n < N; n++) {
            int idx = (k * n * sin_cos_step) % (SIN_COS_N_COUNT); // t = 2*M_PI*k*n/N
            re += in[n]*cos_vals[idx]; // cos(t)
            im -= in[n]*sin_vals[idx]; // sin(t)
        }

        out[k*2 + 0] = re;
        out[k*2 + 1] = im;
    }
}

// Cooley-Tukey 快速傅立叶变换，输入为实数，输出为复数
static void fft(const std::vector<float> & in, std::vector<float> & out) {
    // 为输出分配空间
    out.resize(in.size()*2);

    int N = in.size();
    # 如果输入长度为1，直接将输入复制到输出，然后返回
    if (N == 1) {
        out[0] = in[0];
        out[1] = 0;
        return;
    }

    # 如果输入长度为奇数，直接进行离散傅立叶变换，然后返回
    if (N%2 == 1) {
        dft(in, out);
        return;
    }

    # 创建存储偶数索引元素的向量
    std::vector<float> even;
    # 创建存储奇数索引元素的向量
    std::vector<float> odd;

    # 预留存储空间
    even.reserve(N/2);
    odd.reserve(N/2);

    # 将输入数组中的偶数索引元素存入even向量，奇数索引元素存入odd向量
    for (int i = 0; i < N; i++) {
        if (i % 2 == 0) {
            even.push_back(in[i]);
        } else {
            odd.push_back(in[i]);
        }
    }

    # 创建存储偶数索引元素FFT结果的向量
    std::vector<float> even_fft;
    # 创建存储奇数索引元素FFT结果的向量
    std::vector<float> odd_fft;

    # 对偶数索引元素进行FFT
    fft(even, even_fft);
    # 对奇数索引元素进行FFT
    fft(odd, odd_fft);

    # 计算sin和cos值的步长
    const int sin_cos_step = SIN_COS_N_COUNT / N;
    # 对N/2个频率点进行计算
    for (int k = 0; k < N/2; k++) {
        # 计算sin和cos值的索引
        int idx = k * sin_cos_step; // t = 2*M_PI*k/N
        # 获取cos(t)值
        float re = cos_vals[idx]; // cos(t)
        # 获取-sin(t)值
        float im = -sin_vals[idx]; // sin(t)

        # 获取奇数索引元素FFT结果的实部和虚部
        float re_odd = odd_fft[2*k + 0];
        float im_odd = odd_fft[2*k + 1];

        # 计算输出的实部和虚部
        out[2*k + 0] = even_fft[2*k + 0] + re*re_odd - im*im_odd;
        out[2*k + 1] = even_fft[2*k + 1] + re*im_odd + im*re_odd;

        out[2*(k + N/2) + 0] = even_fft[2*k + 0] - re*re_odd + im*im_odd;
        out[2*(k + N/2) + 1] = even_fft[2*k + 1] - re*im_odd - im*re_odd;
    }
    // 静态函数，使用汉宁窗口对输入的数据进行加窗处理
    static bool hann_window(int length, bool periodic, std::vector<float> & output) {
        // 如果输出向量的大小小于指定长度，调整输出向量的大小为指定长度
        if (output.size() < static_cast<size_t>(length)) {
            output.resize(length);
        }
        // 初始化偏移量为-1
        int offset = -1;
        // 如果周期性为真，将偏移量设置为0
        if (periodic) {
            offset = 0;
        }
        // 使用汉宁窗口函数对输入数据进行加窗处理
        for (int i = 0; i < length; i++) {
            output[i] = 0.5*(1.0 - cosf((2.0*M_PI*i)/(length + offset)));
        }

        return true;
    }

    // 静态函数，用于在工作线程中计算对数梅尔频谱
    static void log_mel_spectrogram_worker_thread(int ith, const std::vector<float> & hann, const std::vector<float> & samples,
                                                  int n_samples, int frame_size, int frame_step, int n_threads,
                                                  const whisper_filters & filters, whisper_mel & mel) {
        // 初始化FFT输入向量，大小为帧大小，初始值为0.0
        std::vector<float> fft_in(frame_size, 0.0);
        // 初始化FFT输出向量，大小为2倍帧步长
        std::vector<float> fft_out(2 * frame_step);
        // 确保n_fft == 1 + (WHISPER_N_FFT / 2)，表示FFT的长度
        int n_fft = 1 + (frame_size / 2);
        // 初始化工作线程的索引
        int i = ith;

        // 仅当fft_in不全为零时计算FFT
    // 循环遍历每个帧的数据，计算梅尔频谱
    for (; i < std::min(n_samples / frame_step + 1, mel.n_len); i += n_threads) {
        // 计算偏移量
        const int offset = i * frame_step;

        // 应用汉宁窗口（速度更快约10%）
        for (int j = 0; j < std::min(frame_size, n_samples - offset); j++) {
            fft_in[j] = hann[j] * samples[offset + j];
        }
        // 填充剩余部分为零
        if (n_samples - offset < frame_size) {
            std::fill(fft_in.begin() + (n_samples - offset), fft_in.end(), 0.0);
        }

        // 进行 FFT
        fft(fft_in, fft_out);

        // 计算复数的模的平方
        for (int j = 0; j < frame_size; j++) {
            fft_out[j] = (fft_out[2 * j + 0] * fft_out[2 * j + 0] + fft_out[2 * j + 1] * fft_out[2 * j + 1]);
        }

        // 计算梅尔频谱
        for (int j = 0; j < mel.n_mel; j++) {
            double sum = 0.0;

            // 展开循环（由 GH 用户 @lunixbochs 建议）
            int k = 0;
            for (k = 0; k < n_fft - 3; k += 4) {
                sum +=
                        fft_out[k + 0] * filters.data[j * n_fft + k + 0] +
                        fft_out[k + 1] * filters.data[j * n_fft + k + 1] +
                        fft_out[k + 2] * filters.data[j * n_fft + k + 2] +
                        fft_out[k + 3] * filters.data[j * n_fft + k + 3];
            }

            // 处理 n_fft 的余数
            for (; k < n_fft; k++) {
                sum += fft_out[k] * filters.data[j * n_fft + k];
            }

            sum = log10(std::max(sum, 1e-10));

            mel.data[j * mel.n_len + i] = sum;
        }
    }

    // 否则 fft_out 全部为零
    double sum = log10(1e-10);
    // 填充剩余的梅尔频谱数据为 sum
    for (; i < mel.n_len; i += n_threads) {
        for (int j = 0; j < mel.n_mel; j++) {
            mel.data[j * mel.n_len + i] = sum;
        }
    }
    // 引用：https://github.com/openai/whisper/blob/main/whisper/audio.py#L110-L157
    // 计算对数梅尔频谱
    static bool log_mel_spectrogram(
                  whisper_state & wstate,  // Whisper 状态
                  const float * samples,    // 输入音频样本
                  const int   n_samples,    // 音频样本数量
                  const int   /*sample_rate*/,  // 采样率（未使用）
                  const int   frame_size,   // 帧大小
                  const int   frame_step,   // 帧步长
                  const int   n_mel,        // 梅尔滤波器数量
                  const int   n_threads,    // 线程数量
                  const whisper_filters & filters,  // Whisper 滤波器
                  const bool   debug,        // 调试模式
                  whisper_mel & mel) {       // 输出的梅尔频谱
    const int64_t t_start_us = ggml_time_us();  // 记录开始时间

    // 汉宁窗口（使用 cosf 来消除差异）
    // 引用：https://pytorch.org/docs/stable/generated/torch.hann_window.html
    // 引用：https://github.com/openai/whisper/blob/main/whisper/audio.py#L147
    std::vector<float> hann;  // 创建存储汉宁窗口的向量
    hann_window(frame_size, true, hann);  // 生成汉宁窗口

    // 计算填充长度
    int64_t stage_1_pad = WHISPER_SAMPLE_RATE * 30;  // 第一阶段填充长度
    int64_t stage_2_pad = frame_size / 2;  // 第二阶段填充长度

    // 初始化一个向量，并从 C 数组中复制数据到其中
    std::vector<float> samples_padded;  // 创建存储填充后音频样本的向量
    samples_padded.resize(n_samples + stage_1_pad + stage_2_pad * 2);  // 调整向量大小
    std::copy(samples, samples + n_samples, samples_padded.begin() + stage_2_pad);  // 复制音频样本数据到向量中

    // 在音频末尾填充 30 秒的零（480,000 个样本）+ 在音频末尾反射填充 200 个样本
    std::fill(samples_padded.begin() + n_samples + stage_2_pad, samples_padded.begin() + n_samples + stage_1_pad + 2 * stage_2_pad, 0);  // 在向量中填充零值

    // 在音频开头反射填充 200 个样本
    std::reverse_copy(samples + 1, samples + 1 + stage_2_pad, samples_padded.begin());  // 在向量开头进行反射填充

    mel.n_mel     = n_mel;  // 设置梅尔频谱的梅尔滤波器数量
    // 引用：https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/native/SpectralOps.cpp#L936
    // 计算帧的数量 + 移除最后一帧
    mel.n_len     = (samples_padded.size() - frame_size) / frame_step;  // 计算帧的数量
    // 计算半填充的样本长度以确保兼容性
    // 计算原始长度
    mel.n_len_org = 1 + (n_samples + stage_2_pad - frame_size) / frame_step;
    // 调整数据大小
    mel.data.resize(mel.n_mel * mel.n_len);

    // 创建线程池
    {
        std::vector<std::thread> workers(n_threads - 1);
        // 启动并行线程
        for (int iw = 0; iw < n_threads - 1; ++iw) {
            workers[iw] = std::thread(
                    log_mel_spectrogram_worker_thread, iw + 1, std::cref(hann), samples_padded,
                    n_samples + stage_2_pad, frame_size, frame_step, n_threads,
                    std::cref(filters), std::ref(mel));
        }

        // 主线程执行
        log_mel_spectrogram_worker_thread(0, hann, samples_padded, n_samples + stage_2_pad, frame_size, frame_step, n_threads, filters, mel);

        // 等待所有线程结束
        for (int iw = 0; iw < n_threads - 1; ++iw) {
            workers[iw].join();
        }
    }

    // 截断和归一化
    double mmax = -1e20;
    for (int i = 0; i < mel.n_mel*mel.n_len; i++) {
        if (mel.data[i] > mmax) {
            mmax = mel.data[i];
        }
    }

    mmax -= 8.0;

    for (int i = 0; i < mel.n_mel*mel.n_len; i++) {
        if (mel.data[i] < mmax) {
            mel.data[i] = mmax;
        }

        mel.data[i] = (mel.data[i] + 4.0)/4.0;
    }

    // 更新时间
    wstate.t_mel_us += ggml_time_us() - t_start_us;

    // 输出 log_mel_spectrogram
    if (debug) {
        std::ofstream outFile("log_mel_spectrogram.json");
        outFile << "[";
        for (uint64_t i = 0; i < mel.data.size() - 1; i++) {
            outFile << mel.data[i] << ", ";
        }
        outFile << mel.data[mel.data.size() - 1] << "]";
        outFile.close();
    }

    // 返回 true
    return true;
// 结束大括号，表示函数或代码块的结束
}

// 将文本分割成标记
//
// 参考链接：https://github.com/openai/gpt-2/blob/a74da5d99abaaba920de8131d64da2862a8f213b/src/encoder.py#L53
//
// 正则表达式（Python）：
// r"""'s|'t|'re|'ve|'m|'ll|'d| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+"""
//
// 正则表达式（C++）：
// R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)"
//
static std::vector<whisper_vocab::id> tokenize(const whisper_vocab & vocab, const std::string & text) {
    // 创建一个字符串向量用于存储单词
    std::vector<std::string> words;

    // 首先将文本分割成单词
    {
        // 创建一个字符串变量来存储文本
        std::string str = text;
        // 创建一个正则表达式模式
        std::string pat = R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)";
        // 编译正则表达式
        std::regex re(pat);
        std::smatch m;

        // 使用正则表达式来匹配文本
        while (std::regex_search(str, m, re)) {
            for (auto x : m) {
                words.push_back(x);
            }
            str = m.suffix();
        }
    }

    // 找到形成单词的最长标记：
    std::vector<whisper_vocab::id> tokens;
    for (const auto & word : words) {
        if (word.empty()) continue;

        int i = 0;
        int n = word.size();
        while (i < n) {
            int j = n;
            bool found = false;
            while (j > i) {
                auto sub = word.substr(i, j-i);
                auto it = vocab.token_to_id.find(sub);
                if (it != vocab.token_to_id.end()) {
                    tokens.push_back(it->second);
                    i = j;
                    found = true;
                    break;
                }
                --j;
            }
            if (!found) {
                WHISPER_LOG_ERROR("unknown token\n");
                ++i;
            }
        }
    }

    return tokens;
}

//
// 接口实现
//

#ifdef WHISPER_USE_COREML
// 用-encoder.mlmodelc替换.bin
static std::string whisper_get_coreml_path_encoder(std::string path_bin) {
    // 找到最后一个'.'的位置
    // 如果在路径中找到了子字符串，则执行以下操作
    if (pos != std::string::npos) {
        // 从路径中截取子字符串并赋值给 path_bin
        path_bin = path_bin.substr(0, pos);
    }

    // 查找路径中最后一个 "-" 的位置
    pos = path_bin.rfind('-');
    // 如果找到了 "-"，则执行以下操作
    if (pos != std::string::npos) {
        // 从 "-" 开始的子字符串赋值给 sub
        auto sub = path_bin.substr(pos);
        // 如果子字符串长度为 5，且第二个字符为 'q'，第四个字符为 '_'，则执行以下操作
        if (sub.size() == 5 && sub[1] == 'q' && sub[3] == '_') {
            // 从路径中截取子字符串并赋值给 path_bin
            path_bin = path_bin.substr(0, pos);
        }
    }

    // 在路径末尾添加 "-encoder.mlmodelc"
    path_bin += "-encoder.mlmodelc";

    // 返回处理后的路径
    return path_bin;
#ifdef WHISPER_USE_OPENVINO
// 如果定义了 WHISPER_USE_OPENVINO，则执行以下代码

// 根据输入的路径，替换文件名后缀为“-encoder-openvino.xml”
static std::string whisper_openvino_get_path_encoder(std::string path_bin) {
    // 寻找路径中最后一个“.”的位置
    auto pos = path_bin.rfind('.');
    // 如果找到了“.”，则执行以下代码
    if (pos != std::string::npos) {
        // 截取路径中“.”之前的部分
        path_bin = path_bin.substr(0, pos);
    }
    // 在路径末尾添加“-encoder-openvino.xml”
    path_bin += "-encoder-openvino.xml";
    // 返回修改后的路径
    return path_bin;
}

// 根据输入的路径，替换文件名后缀为“-encoder-openvino-cache”
static std::string whisper_openvino_get_path_cache(std::string path_bin) {
    // 寻找路径中最后一个“.”的位置
    auto pos = path_bin.rfind('.');
    // 如果找到了“.”，则执行以下代码
    if (pos != std::string::npos) {
        // 截取路径中“.”之前的部分
        path_bin = path_bin.substr(0, pos);
    }
    // 在路径末尾添加“-encoder-openvino-cache”
    path_bin += "-encoder-openvino-cache";
    // 返回修改后的路径
    return path_bin;
}
#endif

// 初始化 whisper_state 结构体
struct whisper_state * whisper_init_state(whisper_context * ctx) {
    // 填充正弦和余弦表
    fill_sin_cos_table();

    // 创建 whisper_state 结构体的实例
    whisper_state * state = new whisper_state;

    // 初始化 whisper_state 结构体中的 backend 成员
    state->backend = whisper_backend_init(ctx->params);

    // 在这一点上，我们还不知道将使用多少解码器，因此我们过度分配 3 倍的 ctx
    // 理论上，可能会有一种情况不够用，但实际上应该总是足够的
    const int factor = 3;

    // 如果 kv_cache_init() 失败，则输出错误信息并释放内存，返回空指针
    if (!kv_cache_init(ctx->model.hparams, state->kv_self, ctx->backend, ctx->itype, factor*ctx->model.hparams.n_text_ctx)) {
        WHISPER_LOG_ERROR("%s: kv_cache_init() failed for self-attention cache\n", __func__);
        delete state;
        return nullptr;
    }

    {
        // 计算 kv_self.k 和 kv_self.v 的内存大小，并输出信息
        const size_t memory_size = ggml_nbytes(state->kv_self.k) + ggml_nbytes(state->kv_self.v);
        WHISPER_LOG_INFO("%s: kv self size  = %7.2f MB\n", __func__, memory_size / 1e6);
    }

    // 如果 kv_cache_init() 失败，则输出错误信息并释放内存，返回空指针
    if (!kv_cache_init(ctx->model.hparams, state->kv_cross, ctx->backend, ctx->itype, ctx->model.hparams.n_audio_ctx)) {
        WHISPER_LOG_ERROR("%s: kv_cache_init() failed for cross-attention cache\n", __func__);
        delete state;
        return nullptr;
    }

    {
        // 计算 kv_cross.k 和 kv_cross.v 的内存大小，并输出信息
        const size_t memory_size = ggml_nbytes(state->kv_cross.k) + ggml_nbytes(state->kv_cross.v);
        WHISPER_LOG_INFO("%s: kv cross size = %7.2f MB\n", __func__, memory_size / 1e6);
    }
}
#ifdef WHISPER_USE_COREML
    // 如果启用了 CoreML，则获取 CoreML 模型的路径
    const auto path_coreml = whisper_get_coreml_path_encoder(ctx->path_model);

    // 输出加载 Core ML 模型的信息
    WHISPER_LOG_INFO("%s: loading Core ML model from '%s'\n", __func__, path_coreml.c_str());
    WHISPER_LOG_INFO("%s: first run on a device may take a while ...\n", __func__);

    // 初始化 CoreML 上下文
    state->ctx_coreml = whisper_coreml_init(path_coreml.c_str());
    if (!state->ctx_coreml) {
        // 如果初始化失败，则输出错误信息
        WHISPER_LOG_ERROR("%s: failed to load Core ML model from '%s'\n", __func__, path_coreml.c_str());
#ifndef WHISPER_COREML_ALLOW_FALLBACK
        // 如果不允许回退，则删除状态并返回空指针
        delete state;
        return nullptr;
#endif
    } else {
        // 如果初始化成功，则输出 Core ML 模型加载信息
        WHISPER_LOG_INFO("%s: Core ML model loaded\n", __func__);
    }
#endif

    // 预留 logits 的空间
    state->logits.reserve(ctx->vocab.n_vocab * ctx->model.hparams.n_text_ctx);

    // 初始化批处理
    state->batch = whisper_batch_init(ctx->model.hparams.n_text_ctx, WHISPER_MAX_DECODERS);

    // 初始化解码器的 tokens 空间
    state->decoders[0].sequence.tokens.reserve(ctx->model.hparams.n_text_ctx);

    // 预留解码器的概率、logits、logprobs 和 logits_id 的空间
    state->decoders[0].probs.reserve    (ctx->vocab.n_vocab);
    state->decoders[0].logits.reserve   (ctx->vocab.n_vocab);
    state->decoders[0].logprobs.reserve (ctx->vocab.n_vocab);
    state->decoders[0].logits_id.reserve(ctx->model.hparams.n_vocab);

    // 初始化解码器的随机数生成器
    state->decoders[0].rng = std::mt19937(0);

    // 初始化卷积分配器
    {
        whisper_allocr_graph_init(state->alloc_conv, ctx->backend,
                [&]() {
                    return whisper_build_graph_conv(*ctx, *state, 0);
                });

        // 输出卷积计算缓冲区的大小信息
        WHISPER_LOG_INFO("%s: compute buffer (conv)   = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_conv) / 1e6);
    }

    // 初始化编码器分配器
    if (!whisper_encode_external(*state)) {
        whisper_allocr_graph_init(state->alloc_encode, ctx->backend,
                [&]() {
                    return whisper_build_graph_encoder(*ctx, *state);
                });

        // 输出编码计算缓冲区的大小信息
        WHISPER_LOG_INFO("%s: compute buffer (encode) = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_encode) / 1e6);
    }
    // 交叉分配器
    {
        // 初始化交叉分配器图形，使用上下文的后端
        whisper_allocr_graph_init(state->alloc_cross, ctx->backend,
                [&]() {
                    // 返回交叉构建图形的结果
                    return whisper_build_graph_cross(*ctx, *state);
                });

        // 记录计算缓冲区（交叉）的大小
        WHISPER_LOG_INFO("%s: compute buffer (cross)  = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_cross) / 1e6);
    }

    // 解码器分配器
    {
        // 初始化解码器分配器图形，使用上下文的后端
        whisper_allocr_graph_init(state->alloc_decode, ctx->backend,
                [&]() {
                    const auto & hparams = ctx->model.hparams;

                    // TODO: 确保这是最坏情况
                    const int n_tokens = hparams.n_text_ctx;
                    const int n_past   = 0;

                    // 准备批次数据
                    whisper_batch_prep_legacy(state->batch, nullptr, n_tokens, n_past, 0);

                    // 返回解码器构建图形的结果
                    return whisper_build_graph_decoder(*ctx, *state, state->batch);
                });

        // 记录计算缓冲区（解码）的大小
        WHISPER_LOG_INFO("%s: compute buffer (decode) = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_decode) / 1e6);
    }

    // 重新分配卷积分配器图形
    whisper_allocr_graph_realloc(state->alloc_conv,   ctx->backend);
    // 重新分配编码分配器图形
    whisper_allocr_graph_realloc(state->alloc_encode, ctx->backend);
    // 重新分配交叉分配器图形
    whisper_allocr_graph_realloc(state->alloc_cross,  ctx->backend);
    // 重新分配解码器分配器图形
    whisper_allocr_graph_realloc(state->alloc_decode, ctx->backend);

    // 返回状态
    return state;
}


int whisper_ctx_init_openvino_encoder(
        struct whisper_context * ctx,
                    const char * model_path,
                    const char * device,
                    const char * cache_dir) {
#ifndef WHISPER_USE_OPENVINO
    (void)(ctx);
    (void)(model_path);
    (void)(device);
    (void)(cache_dir);

    return 1;
#else
    if (!model_path && ctx->path_model.empty()) {
        WHISPER_LOG_ERROR("%s: model_path is nullptr, and ctx has no model_path set.\n", __func__);
        return 1;
    }

    std::string path_encoder;
    if (!model_path) {
        // 如果 model_path 未设置，则尝试在与 ggml-<model>.bin 模型相同的目录中找到它
        path_encoder = whisper_openvino_get_path_encoder(ctx->path_model);
    } else {
        path_encoder = model_path;
    }

    std::string path_cache;
    if (!cache_dir) {
        // 如果 cache_dir 未设置，则将其设置为与 ggml-<model>.bin 相邻的目录
        path_cache = whisper_openvino_get_path_cache(ctx->path_model);
    } else {
        path_cache = cache_dir;
    }

    WHISPER_LOG_INFO("%s: loading OpenVINO model from '%s'\n", __func__, path_encoder.c_str());
    WHISPER_LOG_INFO("%s: first run on a device may take a while ...\n", __func__);

    ctx->state->ctx_openvino = whisper_openvino_init(path_encoder.c_str(), device, path_cache.c_str());
    if (!ctx->state->ctx_openvino) {
        WHISPER_LOG_ERROR("%s: failed to init OpenVINO encoder from '%s'\n", __func__, path_encoder.c_str());
        return 1;
    } else {
        WHISPER_LOG_INFO("%s: OpenVINO model loaded\n", __func__);
    }

    return 0;
#endif
}


struct whisper_context_params whisper_context_default_params() {
    struct whisper_context_params result = {
        /*.use_gpu    =*/ true,
    };
    return result;
}


struct whisper_context * whisper_init_from_file_with_params_no_state(const char * path_model, struct whisper_context_params params) {
    # 使用 WHISPER_LOG_INFO 记录加载模型的信息，包括函数名和模型路径
    WHISPER_LOG_INFO("%s: loading model from '%s'\n", __func__, path_model);

    # 打开模型文件，以二进制模式读取
    auto fin = std::ifstream(path_model, std::ios::binary);
    # 如果文件打开失败，记录错误信息并返回空指针
    if (!fin) {
        WHISPER_LOG_ERROR("%s: failed to open '%s'\n", __func__, path_model);
        return nullptr;
    }

    # 初始化模型加载器
    whisper_model_loader loader = {};

    # 将文件流指针赋值给加载器的上下文
    loader.context = &fin;

    # 定义读取函数，从文件流中读取指定大小的数据
    loader.read = [](void * ctx, void * output, size_t read_size) {
        std::ifstream * fin = (std::ifstream*)ctx;
        fin->read((char *)output, read_size);
        return read_size;
    };

    # 定义判断文件流是否结束的函数
    loader.eof = [](void * ctx) {
        std::ifstream * fin = (std::ifstream*)ctx;
        return fin->eof();
    };

    # 定义关闭文件流的函数
    loader.close = [](void * ctx) {
        std::ifstream * fin = (std::ifstream*)ctx;
        fin->close();
    };

    # 使用加载器和参数初始化上下文
    auto ctx = whisper_init_with_params_no_state(&loader, params);

    # 如果上下文成功初始化，设置模型路径
    if (ctx) {
        ctx->path_model = path_model;
    }

    # 返回上下文
    return ctx;
// 从内存缓冲区初始化 whisper 上下文，不包含状态信息
struct whisper_context * whisper_init_from_buffer_with_params_no_state(void * buffer, size_t buffer_size, struct whisper_context_params params) {
    // 定义内部缓冲区上下文结构体
    struct buf_context {
        uint8_t* buffer;
        size_t size;
        size_t current_offset;
    };

    // 初始化内部缓冲区上下文对象
    buf_context ctx = { reinterpret_cast<uint8_t*>(buffer), buffer_size, 0 };

    // 记录日志信息
    WHISPER_LOG_INFO("%s: loading model from buffer\n", __func__);

    // 初始化模型加载器
    whisper_model_loader loader = {};

    // 设置模型加载器的上下文
    loader.context = &ctx;

    // 定义读取函数
    loader.read = [](void * ctx, void * output, size_t read_size) {
        buf_context * buf = reinterpret_cast<buf_context *>(ctx);

        size_t size_to_copy = buf->current_offset + read_size < buf->size ? read_size : buf->size - buf->current_offset;

        // 将数据从缓冲区复制到输出
        memcpy(output, buf->buffer + buf->current_offset, size_to_copy);
        buf->current_offset += size_to_copy;

        return size_to_copy;
    };

    // 定义结束标志函数
    loader.eof = [](void * ctx) {
        buf_context * buf = reinterpret_cast<buf_context *>(ctx);

        // 判断是否已经读取完整个缓冲区
        return buf->current_offset >= buf->size;
    };

    // 定义关闭函数
    loader.close = [](void * /*ctx*/) { };

    // 使用指定的加载器和参数初始化 whisper 上下文
    return whisper_init_with_params_no_state(&loader, params);
}

// 从文件初始化 whisper 上下文，不包含状态信息
struct whisper_context * whisper_init_with_params_no_state(struct whisper_model_loader * loader, struct whisper_context_params params) {
    // 初始化时间
    ggml_time_init();

    // 创建 whisper 上下文对象
    whisper_context * ctx = new whisper_context;
    ctx->params = params;

    // 如果模型加载失败，则关闭加载器，记录错误日志，释放上下文对象并返回空指针
    if (!whisper_model_load(loader, *ctx)) {
        loader->close(loader->context);
        WHISPER_LOG_ERROR("%s: failed to load model\n", __func__);
        delete ctx;
        return nullptr;
    }

    // 关闭加载器
    loader->close(loader->context);

    // 返回 whisper 上下文对象
    return ctx;
}

// 从文件初始化 whisper 上下文，包含状态信息
struct whisper_context * whisper_init_from_file_with_params(const char * path_model, struct whisper_context_params params) {
    // 从文件初始化 whisper 上下文，不包含状态信息
    whisper_context * ctx = whisper_init_from_file_with_params_no_state(path_model, params);
    // 如果初始化失败，则返回空指针
    if (!ctx) {
        return nullptr;
    }

    // 初始化 whisper 上下文的状态信息
    ctx->state = whisper_init_state(ctx);
}
    # 如果上下文状态为空，则释放上下文并返回空指针
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }
    
    # 返回上下文
    return ctx;
# 从给定的缓冲区和参数初始化 whisper 上下文
struct whisper_context * whisper_init_from_buffer_with_params(void * buffer, size_t buffer_size, struct whisper_context_params params) {
    # 调用 whisper_init_from_buffer_with_params_no_state 函数初始化 whisper 上下文
    whisper_context * ctx = whisper_init_from_buffer_with_params_no_state(buffer, buffer_size, params);
    # 如果初始化失败，则返回空指针
    if (!ctx) {
        return nullptr;
    }

    # 初始化 whisper 上下文的状态
    ctx->state = whisper_init_state(ctx);
    # 如果状态初始化失败，则释放上下文并返回空指针
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }

    # 返回初始化后的 whisper 上下文
    return ctx;
}

# 根据给定的加载器和参数初始化 whisper 上下文
struct whisper_context * whisper_init_with_params(struct whisper_model_loader * loader, struct whisper_context_params params) {
    # 调用 whisper_init_with_params_no_state 函数初始化 whisper 上下文
    whisper_context * ctx = whisper_init_with_params_no_state(loader, params);
    # 如果初始化失败，则返回空指针
    if (!ctx) {
        return nullptr;
    }

    # 初始化 whisper 上下文的状态
    ctx->state = whisper_init_state(ctx);
    # 如果状态初始化失败，则释放上下文并返回空指针
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }

    # 返回初始化后的 whisper 上下文
    return ctx;
}

# 从文件初始化 whisper 上下文
struct whisper_context * whisper_init_from_file(const char * path_model) {
    # 调用 whisper_init_from_file_with_params 函数，使用默认参数初始化 whisper 上下文
    return whisper_init_from_file_with_params(path_model, whisper_context_default_params());
}

# 从缓冲区初始化 whisper 上下文
struct whisper_context * whisper_init_from_buffer(void * buffer, size_t buffer_size) {
    # 调用 whisper_init_from_buffer_with_params 函数，使用默认参数初始化 whisper 上下文
    return whisper_init_from_buffer_with_params(buffer, buffer_size, whisper_context_default_params());
}

# 使用加载器初始化 whisper 上下文
struct whisper_context * whisper_init(struct whisper_model_loader * loader) {
    # 调用 whisper_init_with_params 函数，使用默认参数初始化 whisper 上下文
    return whisper_init_with_params(loader, whisper_context_default_params());
}

# 从文件初始化 whisper 上下文，不包含状态
struct whisper_context * whisper_init_from_file_no_state(const char * path_model) {
    # 调用 whisper_init_from_file_with_params_no_state 函数，使用默认参数初始化 whisper 上下文
    return whisper_init_from_file_with_params_no_state(path_model, whisper_context_default_params());
}

# 从缓冲区初始化 whisper 上下文，不包含状态
struct whisper_context * whisper_init_from_buffer_no_state(void * buffer, size_t buffer_size) {
    # 调用 whisper_init_from_buffer_with_params_no_state 函数，使用默认参数初始化 whisper 上下文
    return whisper_init_from_buffer_with_params_no_state(buffer, buffer_size, whisper_context_default_params());
}

# 使用加载器初始化 whisper 上下文，不包含状态
struct whisper_context * whisper_init_no_state(struct whisper_model_loader * loader) {
    # 调用 whisper_init_with_params_no_state 函数，使用默认参数初始化 whisper 上下文
    return whisper_init_with_params_no_state(loader, whisper_context_default_params());
}

# 释放 whisper 状态
void whisper_free_state(struct whisper_state * state)
    # 如果状态存在
    if (state) {
        # 释放状态中的自身键值缓存
        kv_cache_free(state->kv_self);
        # 释放状态中的交叉键值缓存
        kv_cache_free(state->kv_cross);
#ifdef WHISPER_USE_COREML
        // 如果使用了 CoreML，则释放 CoreML 上下文并将指针置为空
        if (state->ctx_coreml != nullptr) {
            whisper_coreml_free(state->ctx_coreml);
            state->ctx_coreml = nullptr;
        }
#endif

#ifdef WHISPER_USE_OPENVINO
        // 如果使用了 OpenVINO，则释放 OpenVINO 上下文并将指针置为空
        if (state->ctx_openvino != nullptr) {
            whisper_openvino_free(state->ctx_openvino);
            state->ctx_openvino = nullptr;
        }
#endif

        // 释放批处理对象
        whisper_batch_free(state->batch);

        // 释放卷积分配器
        whisper_allocr_free(state->alloc_conv);
        // 释放编码分配器
        whisper_allocr_free(state->alloc_encode);
        // 释放交叉分配器
        whisper_allocr_free(state->alloc_cross);
        // 释放解码分配器
        whisper_allocr_free(state->alloc_decode);

        // 释放后端对象
        ggml_backend_free(state->backend);

        // 释放状态对象
        delete state;
    }
}

void whisper_free(struct whisper_context * ctx) {
    if (ctx) {
        if (ctx->model.ctx) {
            // 释放模型上下文
            ggml_free(ctx->model.ctx);
        }

        if (ctx->model.buffer) {
            // 释放模型缓冲区
            ggml_backend_buffer_free(ctx->model.buffer);
        }

        // 释放状态对象
        whisper_free_state(ctx->state);

        // 释放后端对象
        ggml_backend_free(ctx->backend);

        // 释放上下文对象
        delete ctx;
    }
}

void whisper_free_context_params(struct whisper_context_params * params) {
    if (params) {
        // 释放上下文参数对象
        delete params;
    }
}

void whisper_free_params(struct whisper_full_params * params) {
    if (params) {
        // 释放参数对象
        delete params;
    }
}

int whisper_pcm_to_mel_with_state(struct whisper_context * ctx, struct whisper_state * state, const float * samples, int n_samples, int n_threads) {
    // 将 PCM 转换为 Mel 频谱
    if (!log_mel_spectrogram(*state, samples, n_samples, WHISPER_SAMPLE_RATE, WHISPER_N_FFT, WHISPER_HOP_LENGTH, ctx->model.filters.n_mel, n_threads, ctx->model.filters, false, state->mel)) {
        // 如果转换失败，则记录错误信息并返回 -1
        WHISPER_LOG_ERROR("%s: failed to compute mel spectrogram\n", __func__);
        return -1;
    }

    return 0;
}

int whisper_pcm_to_mel(struct whisper_context * ctx, const float * samples, int n_samples, int n_threads) {
    // 调用带状态的 PCM 转换为 Mel 频谱的函数
    return whisper_pcm_to_mel_with_state(ctx, ctx->state, samples, n_samples, n_threads);
}
// 使用相位估计器将音频加速两倍后转换为梅尔频谱图，同时应用状态
int whisper_pcm_to_mel_phase_vocoder_with_state(struct whisper_context * ctx, struct whisper_state * state, const float * samples, int n_samples, int n_threads) {
    // 如果无法计算梅尔频谱图，则记录错误并返回-1
    if (!log_mel_spectrogram(*state, samples, n_samples, WHISPER_SAMPLE_RATE, 2 * WHISPER_N_FFT, 2 * WHISPER_HOP_LENGTH, ctx->model.filters.n_mel, n_threads, ctx->model.filters, false, state->mel)) {
        WHISPER_LOG_ERROR("%s: failed to compute mel spectrogram\n", __func__);
        return -1;
    }
    // 返回0表示成功
    return 0;
}

// 使用相位估计器将音频加速两倍后转换为梅尔频谱图
int whisper_pcm_to_mel_phase_vocoder(struct whisper_context * ctx, const float * samples, int n_samples, int n_threads) {
    // 调用带状态的函数，传入上下文、状态、音频样本和线程数
    return whisper_pcm_to_mel_phase_vocoder_with_state(ctx, ctx->state, samples, n_samples, n_threads);
}

// 与whisper_pcm_to_mel相同，但应用WSOLA加速音频两倍
// 待办事项

// 与whisper_pcm_to_mel相同，但应用HPTSM加速音频两倍
// 待办事项

// 与whisper_pcm_to_mel相同，但应用PV（带相位锁定）加速音频两倍
// 待办事项

// 使用状态设置梅尔频谱图
int whisper_set_mel_with_state(
        struct whisper_context * ctx,
          struct whisper_state * state,
                   const float * data,
                           int   n_len,
                           int   n_mel) {
    // 如果梅尔频带数量不符合预期，则记录错误并返回-1
    if (n_mel != ctx->model.filters.n_mel) {
        WHISPER_LOG_ERROR("%s: invalid number of mel bands: %d (expected %d)\n", __func__, n_mel, ctx->model.filters.n_mel);
        return -1;
    }
    // 设置状态中的梅尔频谱图参数和数据
    state->mel.n_len     = n_len;
    state->mel.n_len_org = n_len;
    state->mel.n_mel     = n_mel;
    state->mel.data.resize(n_len*n_mel);
    memcpy(state->mel.data.data(), data, n_len*n_mel*sizeof(float));
    // 返回0表示成功
    return 0;
}
// 设置 MEL 参数，调用 whisper_set_mel_with_state 函数
int whisper_set_mel(
        struct whisper_context * ctx,
        const float * data,
        int n_len,
        int n_mel) {
    return whisper_set_mel_with_state(ctx, ctx->state, data, n_len, n_mel);
}

// 使用给定的状态进行编码，调用 whisper_encode_internal 函数
int whisper_encode_with_state(struct whisper_context * ctx, struct whisper_state * state, int offset, int n_threads) {
    // 如果编码失败，则记录错误信息并返回 -1
    if (!whisper_encode_internal(*ctx, *state, offset, n_threads, nullptr, nullptr)) {
        WHISPER_LOG_ERROR("%s: failed to eval\n", __func__);
        return -1;
    }

    return 0;
}

// 对当前状态进行编码，调用 whisper_encode_internal 函数
int whisper_encode(struct whisper_context * ctx, int offset, int n_threads) {
    // 如果编码失败，则记录错误信息并返回 -1
    if (!whisper_encode_internal(*ctx, *ctx->state, offset, n_threads, nullptr, nullptr)) {
        WHISPER_LOG_ERROR("%s: failed to eval\n", __func__);
        return -1;
    }

    return 0;
}

// 使用给定的状态进行解码，调用 whisper_decode_internal 函数
int whisper_decode_with_state(struct whisper_context * ctx, struct whisper_state * state, const whisper_token * tokens, int n_tokens, int n_past, int n_threads) {
    // 准备批处理
    whisper_batch_prep_legacy(state->batch, tokens, n_tokens, n_past, 0);

    // 从键值缓存中移除序列
    whisper_kv_cache_seq_rm(state->kv_self, 0, n_past, -1);

    // 如果解码失败，则记录错误信息并返回 1
    if (!whisper_decode_internal(*ctx, *state, state->batch, n_threads, nullptr, nullptr)) {
        WHISPER_LOG_ERROR("%s: failed to eval\n", __func__);
        return 1;
    }

    return 0;
}

// 对当前状态进行解码，调用 whisper_decode_with_state 函数
int whisper_decode(struct whisper_context * ctx, const whisper_token * tokens, int n_tokens, int n_past, int n_threads) {
    // 如果状态为空，则记录错误信息并返回 -1
    if (ctx->state == nullptr) {
        WHISPER_LOG_ERROR("%s: ERROR state was not loaded.\n", __func__);
        return -1;
    }

    return whisper_decode_with_state(ctx, ctx->state, tokens, n_tokens, n_past, n_threads);
}

// 对文本进行标记化
int whisper_tokenize(struct whisper_context * ctx, const char * text, whisper_token * tokens, int n_max_tokens) {
    // 调用 tokenize 函数进行文本标记化
    const auto res = tokenize(ctx->vocab, text);

    // 如果结果数量超过最大标记数量，则记录错误信息并返回 -1
    if (n_max_tokens < (int) res.size()) {
        WHISPER_LOG_ERROR("%s: too many resulting tokens: %d (max %d)\n", __func__, (int) res.size(), n_max_tokens);
        return -1;
    }
    # 遍历 res 列表中的元素
    for (int i = 0; i < (int) res.size(); i++) {
        # 将 res 列表中的元素赋值给 tokens 列表中对应的位置
        tokens[i] = res[i];
    }
    
    # 返回 res 列表的大小
    return res.size();
// 返回语言映射表中最大的语言ID
int whisper_lang_max_id() {
    // 初始化最大ID为0
    auto max_id = 0;
    // 遍历语言映射表，更新最大ID
    for (const auto & kv : g_lang) {
        max_id = std::max(max_id, kv.second.first);
    }
    // 返回最大ID
    return max_id;
}

// 根据语言名称获取对应的语言ID
int whisper_lang_id(const char * lang) {
    // 如果语言映射表中不存在该语言，则遍历映射表查找对应的语言ID
    if (!g_lang.count(lang)) {
        for (const auto & kv : g_lang) {
            if (kv.second.second == lang) {
                return kv.second.first;
            }
        }
        // 输出错误信息并返回-1
        WHISPER_LOG_ERROR("%s: unknown language '%s'\n", __func__, lang);
        return -1;
    }
    // 返回对应语言的ID
    return g_lang.at(lang).first;
}

// 根据语言ID获取对应的语言名称
const char * whisper_lang_str(int id) {
    // 遍历语言映射表，根据ID查找对应的语言名称
    for (const auto & kv : g_lang) {
        if (kv.second.first == id) {
            return kv.first.c_str();
        }
    }
    // 输出错误信息并返回空指针
    WHISPER_LOG_ERROR("%s: unknown language id %d\n", __func__, id);
    return nullptr;
}

// 根据语言ID获取对应的完整语言名称
const char * whisper_lang_str_full(int id) {
    // 遍历语言映射表，根据ID查找对应的完整语言名称
    for (const auto & kv : g_lang) {
        if (kv.second.first == id) {
            return kv.second.second.c_str();
        }
    }
    // 输出错误信息并返回空指针
    WHISPER_LOG_ERROR("%s: unknown language id %d\n", __func__, id);
    return nullptr;
}

// 根据状态自动检测语言
int whisper_lang_auto_detect_with_state(
        struct whisper_context * ctx,
        struct whisper_state * state,
        int   offset_ms,
        int   n_threads,
        float * lang_probs) {
    // 计算偏移量
    const int seek = offset_ms/10;

    // 如果偏移量小于0，输出错误信息并返回-1
    if (seek < 0) {
        WHISPER_LOG_ERROR("%s: offset %dms is before the start of the audio\n", __func__, offset_ms);
        return -1;
    }

    // 如果偏移量大于等于原始音频长度，输出错误信息并返回-2
    if (seek >= state->mel.n_len_org) {
        WHISPER_LOG_ERROR("%s: offset %dms is past the end of the audio (%dms)\n", __func__, offset_ms, state->mel.n_len_org*10);
        return -2;
    }

    // 运行编码器
    if (whisper_encode_with_state(ctx, state, seek, n_threads) != 0) {
        WHISPER_LOG_ERROR("%s: failed to encode\n", __func__);
        return -6;
    }

    // 创建提示标记向量
    const std::vector<whisper_token> prompt = { whisper_token_sot(ctx) };
}
    // 使用whisper_decode_with_state函数解码prompt数据，如果返回值不为0，则输出错误信息并返回-7
    if (whisper_decode_with_state(ctx, state, prompt.data(), prompt.size(), 0, n_threads) != 0) {
        WHISPER_LOG_ERROR("%s: failed to decode\n", __func__);
        return -7;
    }

    // 获取state对象中第一个解码器的logits_id，并清空其中的内容
    auto & logits_id = state->decoders[0].logits_id;
    logits_id.clear();

    // 遍历g_lang中的键值对，计算logits_id并存储
    for (const auto & kv : g_lang) {
        const auto token_lang = whisper_token_lang(ctx, kv.second.first);
        logits_id.emplace_back(state->logits[token_lang], kv.second.first);
    }

    // 对logits_id进行降序排序
    {
        using pair_type = std::remove_reference<decltype(logits_id)>::type::value_type;
        std::sort(logits_id.begin(), logits_id.end(), [](const pair_type & a, const pair_type & b) {
            return a.first > b.first;
        });
    }

    // 对logits_id进行softmax处理
    {
        // 获取logits_id中第一个元素的值
        const auto max = logits_id[0].first;

        // 计算softmax的分母
        double sum = 0.0f;
        for (auto & kv : logits_id) {
            kv.first = exp(kv.first - max);
            sum += kv.first;
        }

        // 计算softmax的分子
        for (auto & kv : logits_id) {
            kv.first /= sum;
        }
    }

    // 将logits_id中的概率值存储到lang_probs中，并打印信息
    {
        for (const auto & prob : logits_id) {
            if (lang_probs) {
                lang_probs[prob.second] = prob.first;
            }

            //printf("%s: lang %2d (%3s): %f\n", __func__, prob.second, whisper_lang_str(prob.second), prob.first);
        }
    }

    // 返回logits_id中第一个元素的第二个值
    return logits_id[0].second;
# 返回自动检测语言的结果
int whisper_lang_auto_detect(
        struct whisper_context * ctx,
                           int   offset_ms,
                           int   n_threads,
                         float * lang_probs) {
    # 调用带有状态参数的自动检测语言函数
    return whisper_lang_auto_detect_with_state(ctx, ctx->state, offset_ms, n_threads, lang_probs);
}

# 返回模型的词汇量大小
int whisper_model_n_vocab(struct whisper_context * ctx) {
    return ctx->model.hparams.n_vocab;
}

# 返回模型的音频上下文大小
int whisper_model_n_audio_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_ctx;
}

# 返回模型的音频状态大小
int whisper_model_n_audio_state(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_state;
}

# 返回模型的音频头部大小
int whisper_model_n_audio_head(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_head;
}

# 返回模型的音频层数
int whisper_model_n_audio_layer(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_layer;
}

# 返回模型的文本上下文大小
int whisper_model_n_text_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_ctx;
}

# 返回模型的文本状态大小
int whisper_model_n_text_state(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_state;
}

# 返回模型的文本头部大小
int whisper_model_n_text_head(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_head;
}

# 返回模型的文本层数
int whisper_model_n_text_layer(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_layer;
}

# 返回模型的梅尔频率数量
int whisper_model_n_mels(struct whisper_context * ctx) {
    return ctx->model.hparams.n_mels;
}

# 返回模型的特征类型
int whisper_model_ftype(struct whisper_context * ctx) {
    return ctx->model.hparams.ftype;
}

# 返回模型的类型
int whisper_model_type(struct whisper_context * ctx) {
    return ctx->model.type;
}

# 返回可读的模型类型
const char *whisper_model_type_readable(struct whisper_context * ctx) {
    switch (ctx->model.type) {
    case e_model::MODEL_TINY:
        return "tiny";
    case e_model::MODEL_BASE:
        return "base";
    case e_model::MODEL_SMALL:
        return "small";
    case e_model::MODEL_MEDIUM:
        return "medium";
    case e_model::MODEL_LARGE:
        return "large";
    default:
        return "unknown";
    }
}
// 从状态结构体中获取原始长度参数
int whisper_n_len_from_state(struct whisper_state * state) {
    return state->mel.n_len_org;
}

// 从上下文结构体中获取原始长度参数
int whisper_n_len(struct whisper_context * ctx) {
    return ctx->state->mel.n_len_org;
}

// 从上下文结构体中获取词汇表大小
int whisper_n_vocab(struct whisper_context * ctx) {
    return ctx->vocab.n_vocab;
}

// 从上下文结构体中获取文本上下文大小
int whisper_n_text_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_ctx;
}

// 从上下文结构体中获取音频上下文大小
int whisper_n_audio_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_ctx;
}

// 判断上下文结构体中的词汇表是否为多语言
int whisper_is_multilingual(struct whisper_context * ctx) {
    return ctx->vocab.is_multilingual() ? 1 : 0;
}

// 从上下文结构体中获取logits数据指针
float * whisper_get_logits(struct whisper_context * ctx) {
    return ctx->state->logits.data();
}

// 从状态结构体中获取logits数据指针
float * whisper_get_logits_from_state(struct whisper_state * state) {
    return state->logits.data();
}

// 将词汇表中的token转换为字符串
const char * whisper_token_to_str(struct whisper_context * ctx, whisper_token token) {
    return ctx->vocab.id_to_token.at(token).c_str();
}

// 获取结束token
whisper_token whisper_token_eot(struct whisper_context * ctx) {
    return ctx->vocab.token_eot;
}

// 获取开始token
whisper_token whisper_token_sot(struct whisper_context * ctx) {
    return ctx->vocab.token_sot;
}

// 获取语言开始token
whisper_token whisper_token_solm(struct whisper_context * ctx) {
    return ctx->vocab.token_solm;
}

// 获取前一个token
whisper_token whisper_token_prev(struct whisper_context * ctx) {
    return ctx->vocab.token_prev;
}

// 获取空格token
whisper_token whisper_token_nosp(struct whisper_context * ctx) {
    return ctx->vocab.token_nosp;
}

// 获取不是token
whisper_token whisper_token_not(struct whisper_context * ctx) {
    return ctx->vocab.token_not;
}

// 获取开始token
whisper_token whisper_token_beg(struct whisper_context * ctx) {
    return ctx->vocab.token_beg;
}

// 获取特定语言的token
whisper_token whisper_token_lang(struct whisper_context * ctx, int lang_id) {
    return whisper_token_sot(ctx) + 1 + lang_id;
}

// 获取翻译token
whisper_token whisper_token_translate(struct whisper_context * ctx) {
    return ctx->vocab.token_translate;
}

// 获取转录token
whisper_token whisper_token_transcribe(struct whisper_context * ctx) {
    return ctx->vocab.token_transcribe;
}
void whisper_print_timings(struct whisper_context * ctx) {
    // 获取当前时间作为结束时间
    const int64_t t_end_us = ggml_time_us();

    // 输出空行
    WHISPER_LOG_INFO("\n");
    // 输出加载时间
    WHISPER_LOG_INFO("%s:     load time = %8.2f ms\n", __func__, ctx->t_load_us / 1000.0f);
    // 如果状态不为空
    if (ctx->state != nullptr) {
        // 获取最大值
        const int32_t n_sample = std::max(1, ctx->state->n_sample);
        const int32_t n_encode = std::max(1, ctx->state->n_encode);
        const int32_t n_decode = std::max(1, ctx->state->n_decode);
        const int32_t n_batchd = std::max(1, ctx->state->n_batchd);
        const int32_t n_prompt = std::max(1, ctx->state->n_prompt);

        // 输出失败次数
        WHISPER_LOG_INFO("%s:     fallbacks = %3d p / %3d h\n", __func__, ctx->state->n_fail_p, ctx->state->n_fail_h);
        // 输出 mel 时间
        WHISPER_LOG_INFO("%s:      mel time = %8.2f ms\n", __func__, ctx->state->t_mel_us / 1000.0f);
        // 输出 sample 时间
        WHISPER_LOG_INFO("%s:   sample time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_sample_us, n_sample, 1e-3f * ctx->state->t_sample_us / n_sample);
        // 输出 encode 时间
        WHISPER_LOG_INFO("%s:   encode time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_encode_us, n_encode, 1e-3f * ctx->state->t_encode_us / n_encode);
        // 输出 decode 时间
        WHISPER_LOG_INFO("%s:   decode time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_decode_us, n_decode, 1e-3f * ctx->state->t_decode_us / n_decode);
        // 输出 batchd 时间
        WHISPER_LOG_INFO("%s:   batchd time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_batchd_us, n_batchd, 1e-3f * ctx->state->t_batchd_us / n_batchd);
        // 输出 prompt 时间
        WHISPER_LOG_INFO("%s:   prompt time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_prompt_us, n_prompt, 1e-3f * ctx->state->t_prompt_us / n_prompt);
    }
    // 输出总时间
    WHISPER_LOG_INFO("%s:    total time = %8.2f ms\n", __func__, (t_end_us - ctx->t_start_us)/1000.0f);
}

void whisper_reset_timings(struct whisper_context * ctx) {
    // 获取当前时间作为开始时间
    ctx->t_start_us = ggml_time_us();
}
    # 如果上下文状态不为空，则执行以下操作
    if (ctx->state != nullptr) {
        # 将上下文状态中的各时间统计字段初始化为0
        ctx->state->t_mel_us = 0;
        ctx->state->t_sample_us = 0;
        ctx->state->t_encode_us = 0;
        ctx->state->t_decode_us = 0;
        ctx->state->t_batchd_us = 0;
        ctx->state->t_prompt_us = 0;
        # 将上下文状态中的各计数字段初始化为0
        ctx->state->n_sample = 0;
        ctx->state->n_encode = 0;
        ctx->state->n_decode = 0;
        ctx->state->n_batchd = 0;
        ctx->state->n_prompt = 0;
    }
// 检查是否定义了 WHISPER_USE_COREML，如果定义了则返回1，否则返回0
static int whisper_has_coreml(void) {
#ifdef WHISPER_USE_COREML
    return 1;
#else
    return 0;
#endif
}

// 检查是否定义了 WHISPER_USE_OPENVINO，如果定义了则返回1，否则返回0
static int whisper_has_openvino(void) {
#ifdef WHISPER_USE_OPENVINO
    return 1;
#else
    return 0;
#endif
}

// 打印系统信息
const char * whisper_print_system_info(void) {
    // 静态字符串对象
    static std::string s;

    // 清空字符串
    s  = "";
    // 拼接字符串，包括各种 CPU 功能的信息
    s += "AVX = "       + std::to_string(ggml_cpu_has_avx())       + " | ";
    s += "AVX2 = "      + std::to_string(ggml_cpu_has_avx2())      + " | ";
    s += "AVX512 = "    + std::to_string(ggml_cpu_has_avx512())    + " | ";
    s += "FMA = "       + std::to_string(ggml_cpu_has_fma())       + " | ";
    s += "NEON = "      + std::to_string(ggml_cpu_has_neon())      + " | ";
    s += "ARM_FMA = "   + std::to_string(ggml_cpu_has_arm_fma())   + " | ";
    s += "METAL = "     + std::to_string(ggml_cpu_has_metal())     + " | ";
    s += "F16C = "      + std::to_string(ggml_cpu_has_f16c())      + " | ";
    s += "FP16_VA = "   + std::to_string(ggml_cpu_has_fp16_va())   + " | ";
    s += "WASM_SIMD = " + std::to_string(ggml_cpu_has_wasm_simd()) + " | ";
    s += "BLAS = "      + std::to_string(ggml_cpu_has_blas())      + " | ";
    s += "SSE3 = "      + std::to_string(ggml_cpu_has_sse3())      + " | ";
    s += "SSSE3 = "     + std::to_string(ggml_cpu_has_ssse3())     + " | ";
    s += "VSX = "       + std::to_string(ggml_cpu_has_vsx())       + " | ";
    s += "CUDA = "      + std::to_string(ggml_cpu_has_cublas())    + " | ";
    s += "COREML = "    + std::to_string(whisper_has_coreml())     + " | ";
    s += "OPENVINO = "  + std::to_string(whisper_has_openvino())   + " | ";

    // 返回字符串的 C 风格指针
    return s.c_str();
}

// 从 llama.cpp 移植的语法
// 解码可能以不完整序列结尾的 UTF-8 字符串。添加终止符0以便用作指针。如果遇到无效序列，则返回 `whisper_partial_utf8.n_remain == -1`。
// 解码 UTF-8 编码的字符串，返回解码后的 Unicode 码点数组和可能存在的部分字符
std::pair<std::vector<uint32_t>, whisper_partial_utf8> decode_utf8(
        const char         * src,
        whisper_partial_utf8   partial_start) {
    // UTF-8 编码长度查找表
    static const int      lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 2, 2, 3, 4 };
    // 源字符串位置指针
    const char          * pos      = src;
    // 存储 Unicode 码点的数组
    std::vector<uint32_t> code_points;
    // 部分字符的值
    uint32_t              value    = partial_start.value;
    // 部分字符剩余字节数
    int                   n_remain = partial_start.n_remain;

    // 继续之前的解码，如果适用
    while (*pos != 0 && n_remain > 0) {
        // 获取下一个字节
        uint8_t next_byte = static_cast<uint8_t>(*pos);
        // 检查是否为无效序列，如果是则中止
        if ((next_byte >> 6) != 2) {
            code_points.push_back(0);
            return std::make_pair(std::move(code_points), whisper_partial_utf8{ 0, -1 });
        }
        // 计算 Unicode 码点值
        value = (value << 6) + (next_byte & 0x3F);
        ++pos;
        --n_remain;
    }

    // 如果部分字符剩余字节数大于 0 且当前字节已经结束
    if (partial_start.n_remain > 0 && n_remain == 0) {
        code_points.push_back(value);
    }

    // 解码任何后续的 UTF-8 序列，可能以不完整的序列结束
    while (*pos != 0) {
        // 获取第一个字节
        uint8_t  first_byte = static_cast<uint8_t>(*pos);
        // 获取高位字节
        uint8_t  highbits   = first_byte >> 4;
        // 计算剩余字节数
        n_remain   = lookup[highbits] - 1;

        // 如果剩余字节数小于 0，表示无效序列，中止
        if (n_remain < 0) {
            code_points.clear();
            code_points.push_back(0);
            return std::make_pair(std::move(code_points), whisper_partial_utf8{ 0, n_remain });
        }

        // 计算 Unicode 码点值
        uint8_t  mask       = (1 << (7 - n_remain)) - 1;
        value      = first_byte & mask;
        ++pos;
        // 继续计算 Unicode 码点值
        while (*pos != 0 && n_remain > 0) {
            value = (value << 6) + (static_cast<uint8_t>(*pos) & 0x3F);
            ++pos;
            --n_remain;
        }
        // 如果剩余字节数为 0，表示完整的 Unicode 码点
        if (n_remain == 0) {
            code_points.push_back(value);
        }
    }
    // 添加结束标志
    code_points.push_back(0);

    return std::make_pair(std::move(code_points), whisper_partial_utf8{ value, n_remain });
}
// 返回 true，如果 pos 指向规则定义的结束位置
static bool whisper_grammar_is_end_of_sequence(const whisper_grammar_element * pos) {
    switch (pos->type) {
        case WHISPER_GRETYPE_END: return true;  // NOLINT
        case WHISPER_GRETYPE_ALT: return true;  // NOLINT
        default:                return false;
    }
}

// 返回 true，如果 chr 满足 pos 处的字符范围（正常或反向范围）
// 断言 pos 指向字符范围元素
static std::pair<bool, const whisper_grammar_element *> whisper_grammar_match_char(
        const whisper_grammar_element * pos,
        const uint32_t                chr) {

    bool found            = false;
    bool is_positive_char = pos->type == WHISPER_GRETYPE_CHAR;

    WHISPER_ASSERT(is_positive_char || pos->type == WHISPER_GRETYPE_CHAR_NOT); // NOLINT

    do {
        if (pos[1].type == WHISPER_GRETYPE_CHAR_RNG_UPPER) {
            // 包含范围，例如 [a-z]
            found = found || (pos->value <= chr && chr <= pos[1].value);
            pos += 2;
        } else {
            // 精确字符匹配，例如 [a] 或 "a"
            found = found || pos->value == chr;
            pos += 1;
        }
    } while (pos->type == WHISPER_GRETYPE_CHAR_ALT);

    return std::make_pair(found == is_positive_char, pos);
}

// 返回 true，如果给定部分 UTF-8 序列的某个延续可以满足 pos 处的字符范围（正常或反向范围）
// 断言 pos 指向字符范围元素
static bool whisper_grammar_match_partial_char(
        const whisper_grammar_element * pos,
        const whisper_partial_utf8      partial_utf8) {

    bool is_positive_char = pos->type == WHISPER_GRETYPE_CHAR;
    WHISPER_ASSERT(is_positive_char || pos->type == WHISPER_GRETYPE_CHAR_NOT);

    uint32_t partial_value = partial_utf8.value;
    int      n_remain      = partial_utf8.n_remain;

    // 无效序列或 7 位字符跨越 2 个字节（过长）
}
    // 如果剩余字节数小于0或者（剩余字节数等于1且部分值小于2），则返回false
    if (n_remain < 0 || (n_remain == 1 && partial_value < 2)) {
        return false;
    }

    // 可能的 UTF-8 序列完成的代码点范围
    uint32_t low  = partial_value << (n_remain * 6);
    uint32_t high = low | ((1 << (n_remain * 6)) - 1);

    // 如果 low 等于0
    if (low == 0) {
        // 如果剩余字节数为2，则 low 等于1左移11位
        if (n_remain == 2) {
            low = 1 << 11;
        } 
        // 如果剩余字节数为3，则 low 等于1左移16位
        else if (n_remain == 3) {
            low = 1 << 16;
        }
    }

    // 循环处理
    do {
        // 如果下一个位置的类型为 WHISPER_GRETYPE_CHAR_RNG_UPPER
        if (pos[1].type == WHISPER_GRETYPE_CHAR_RNG_UPPER) {
            // 包含范围，例如 [a-z]
            if (pos->value <= high && low <= pos[1].value) {
                return is_positive_char;
            }
            pos += 2;
        } 
        // 如果下一个位置的类型不为 WHISPER_GRETYPE_CHAR_RNG_UPPER
        else {
            // 精确字符匹配，例如 [a] 或 "a"
            if (low <= pos->value && pos->value <= high) {
                return is_positive_char;
            }
            pos += 1;
        }
    } while (pos->type == WHISPER_GRETYPE_CHAR_ALT);

    // 返回非正字符
    return !is_positive_char;
// 将一个语法下推栈转换为N个可能的栈，所有栈都以字符范围（终结元素）结尾
static void whisper_grammar_advance_stack(
        // 语法规则的向量，包含多个规则
        const std::vector<std::vector<whisper_grammar_element>>   & rules,
        // 当前的语法下推栈
        const std::vector<const whisper_grammar_element *>        & stack,
        // 新的语法下推栈的向量
        std::vector<std::vector<const whisper_grammar_element *>> & new_stacks) {

    // 如果当前栈为空，则将其加入到新栈中并返回
    if (stack.empty()) {
        new_stacks.push_back(stack);
        return;
    }

    // 获取栈顶元素的指针
    const whisper_grammar_element * pos = stack.back();
    # 根据 pos->type 的不同情况进行不同的处理
    switch (pos->type) {
        # 如果 pos->type 是 WHISPER_GRETYPE_RULE_REF
        case WHISPER_GRETYPE_RULE_REF: {
            # 将 pos->value 转换为 size_t 类型，作为 rule_id
            const size_t                  rule_id = static_cast<size_t>(pos->value);
            # 获取 rules[rule_id] 的数据，并赋值给 subpos
            const whisper_grammar_element * subpos  = rules[rule_id].data();
            # 循环处理 rule ref
            do {
                # 初始化一个不包含顶部元素（pos）的新栈
                std::vector<const whisper_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
                # 如果 rule ref 后面还有元素，将其加入新栈
                if (!whisper_grammar_is_end_of_sequence(pos + 1)) {
                    new_stack.push_back(pos + 1);
                }
                # 如果 alternate 不为空，将其加入新栈
                if (!whisper_grammar_is_end_of_sequence(subpos)) {
                    new_stack.push_back(subpos);
                }
                # 推进栈并生成新的栈
                whisper_grammar_advance_stack(rules, new_stack, new_stacks);
                # 扫描到 alternate 定义的末尾
                while (!whisper_grammar_is_end_of_sequence(subpos)) {
                    subpos++;
                }
                # 如果 subpos 的类型是 WHISPER_GRETYPE_ALT，表示还有另一个 alternate 定义需要处理
                if (subpos->type == WHISPER_GRETYPE_ALT) {
                    subpos++;
                } else {
                    break;
                }
            } while (true);
            break;
        }
        # 如果 pos->type 是 WHISPER_GRETYPE_CHAR 或 WHISPER_GRETYPE_CHAR_NOT
        case WHISPER_GRETYPE_CHAR:
        case WHISPER_GRETYPE_CHAR_NOT:
            # 将当前栈加入到新栈集合中
            new_stacks.push_back(stack);
            break;
        # 其他情况
        default:
            # 抛出断言错误
            WHISPER_ASSERT(false);
    }
// 接受一个文法上可能的下推栈集合，这些栈需要位于一个字符范围内（参见 `whisper_grammar_advance_stack`），
// 并在这些位置上如果给定字符被接受，则产生 N 个可能的栈
static std::vector<std::vector<const whisper_grammar_element *>> whisper_grammar_accept(
        const std::vector<std::vector<whisper_grammar_element>>         & rules,
        const std::vector<std::vector<const whisper_grammar_element *>> & stacks,
        const uint32_t                                                  chr) {

    // 创建一个新的栈集合
    std::vector<std::vector<const whisper_grammar_element *>> new_stacks;

    // 遍历每个栈
    for (const auto & stack : stacks) {
        // 如果栈为空，则继续下一个栈
        if (stack.empty()) {
            continue;
        }

        // 检查栈顶元素是否匹配给定字符
        auto match = whisper_grammar_match_char(stack.back(), chr);
        if (match.first) {
            const whisper_grammar_element * pos = match.second;

            // 更新栈顶元素到下一个元素（如果有的话）
            std::vector<const whisper_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
            if (!whisper_grammar_is_end_of_sequence(pos)) {
                new_stack.push_back(pos);
            }
            // 调用函数以更新栈集合
            whisper_grammar_advance_stack(rules, new_stack, new_stacks);
        }
    }

    return new_stacks;
}

// 声明函数 whisper_grammar_reject_candidates
static std::vector<whisper_grammar_candidate> whisper_grammar_reject_candidates(
        const std::vector<std::vector<whisper_grammar_element>>         & rules,
        const std::vector<std::vector<const whisper_grammar_element *>> & stacks,
        const std::vector<whisper_grammar_candidate>                    & candidates);

// 声明函数 whisper_grammar_reject_candidates_for_stack
static std::vector<whisper_grammar_candidate> whisper_grammar_reject_candidates_for_stack(
        const std::vector<std::vector<whisper_grammar_element>> & rules,
        const std::vector<const whisper_grammar_element *>      & stack,
        const std::vector<whisper_grammar_candidate>            & candidates) {
    # 创建一个名为rejects的whisper_grammar_candidate类型的向量
    std::vector<whisper_grammar_candidate> rejects;

    # 如果stack为空
    if (stack.empty()) {
        # 遍历candidates中的每个tok
        for (auto tok : candidates) {
            # 如果tok的code_points不为0，或者tok的partial_utf8.n_remain不为0
            if (*tok.code_points != 0 || tok.partial_utf8.n_remain != 0) {
                # 将tok添加到rejects中
                rejects.push_back(tok);
            }
        }
        # 返回rejects
        return rejects;
    }

    # 获取stack的最后一个元素
    const whisper_grammar_element * stack_pos = stack.back();

    # 创建一个名为next_candidates的whisper_grammar_candidate类型的向量
    std::vector<whisper_grammar_candidate> next_candidates;
    # 遍历candidates中的每个tok
    for (auto tok : candidates) {
        # 如果tok的code_points为0
        if (*tok.code_points == 0) {
            # 如果tok的partial_utf8.n_remain不为0，并且无法满足语法中的位置
            if (tok.partial_utf8.n_remain != 0 && !whisper_grammar_match_partial_char(stack_pos, tok.partial_utf8)) {
                # 将tok添加到rejects中
                rejects.push_back(tok);
            }
        } else if (whisper_grammar_match_char(stack_pos, *tok.code_points).first) {
            # 将tok的id、code_points+1和partial_utf8添加到next_candidates中
            next_candidates.push_back({ tok.id, tok.code_points + 1, tok.partial_utf8 });
        } else {
            # 将tok添加到rejects中
            rejects.push_back(tok);
        }
    }

    # 获取stack_pos之后的元素
    const auto * stack_pos_after = whisper_grammar_match_char(stack_pos, 0).second;

    # 更新stack的顶部元素为下一个元素（如果有的话）
    std::vector<const whisper_grammar_element *> stack_after(stack.begin(), stack.end() - 1);
    if (!whisper_grammar_is_end_of_sequence(stack_pos_after)) {
        stack_after.push_back(stack_pos_after);
    }
    # 创建名为next_stacks的whisper_grammar_element类型的向量的向量
    std::vector<std::vector<const whisper_grammar_element *>> next_stacks;
    # 将rules、stack_after和next_stacks传递给whisper_grammar_advance_stack函数
    whisper_grammar_advance_stack(rules, stack_after, next_stacks);

    # 获取whisper_grammar_reject_candidates函数的返回值，存储在next_rejects中
    auto next_rejects = whisper_grammar_reject_candidates(rules, next_stacks, next_candidates);
    # 遍历next_rejects中的每个tok
    for (auto tok : next_rejects) {
        # 将tok的id、code_points-1和partial_utf8添加到rejects中
        rejects.push_back({ tok.id, tok.code_points - 1, tok.partial_utf8 });
    }

    # 返回rejects
    return rejects;
}

static std::vector<whisper_grammar_candidate> whisper_grammar_reject_candidates(
        const std::vector<std::vector<whisper_grammar_element>>         & rules,
        const std::vector<std::vector<const whisper_grammar_element *>> & stacks,
        const std::vector<whisper_grammar_candidate>                    & candidates) {
    // 如果候选项为空或者堆栈为空，返回空的候选项向量
    if (candidates.empty() || stacks.empty()) {
        return std::vector<whisper_grammar_candidate>();
    }

    // 为第一个堆栈中的候选项拒绝不合格的候选项
    auto rejects = whisper_grammar_reject_candidates_for_stack(rules, stacks.front(), candidates);

    // 遍历每个堆栈，为其中的候选项拒绝不合格的候选项
    for (size_t i = 1, size = stacks.size(); i < size; ++i) {
        rejects = whisper_grammar_reject_candidates_for_stack(rules, stacks[i], rejects);
    }
    // 返回拒绝的候选项
    return rejects;
}

static struct whisper_grammar whisper_grammar_init(
            const whisper_grammar_element ** rules,
                                 size_t      n_rules,
                                 size_t      i_start_rule) {
    const whisper_grammar_element * pos;

    // 将规则定义复制到向量中
    std::vector<std::vector<whisper_grammar_element>> vec_rules(n_rules);
    for (size_t i = 0; i < n_rules; i++) {
        for (pos = rules[i]; pos->type != WHISPER_GRETYPE_END; pos++) {
            vec_rules[i].push_back(*pos);
        }
        vec_rules[i].push_back({WHISPER_GRETYPE_END, 0});
    }

    // 循环遍历起始规则的替代项，构建初始堆栈
    std::vector<std::vector<const whisper_grammar_element *>> stacks;
    pos = rules[i_start_rule];
    // 创建一个do-while循环，用于处理语法规则的解析
    do {
        // 创建一个存储指针的向量
        std::vector<const whisper_grammar_element *> stack;
        // 如果当前位置不是序列的结尾，则将其添加到栈中
        if (!whisper_grammar_is_end_of_sequence(pos)) {
            stack.push_back(pos);
        }
        // 推进栈，处理语法规则
        whisper_grammar_advance_stack(vec_rules, stack, stacks);
        // 循环直到当前位置为序列的结尾
        while (!whisper_grammar_is_end_of_sequence(pos)) {
            // 扫描到交替定义的结尾
            pos++;
        }
        // 如果当前位置的类型是交替类型
        if (pos->type == WHISPER_GRETYPE_ALT) {
            // 存在该规则的另一个交替定义需要处理
            pos++;
        } else {
            // 否则跳出循环
            break;
        }
    } while (true);

    // 返回处理后的结果，包括移动的规则向量、移动的栈和空的对象
    return { std::move(vec_rules), std::move(stacks), {} };
}
// 静态函数，用于抑制无效语法
static void whisper_suppress_invalid_grammar(
             whisper_context  & ctx,  // 上下文对象的引用
    const whisper_full_params & params,  // 参数对象的常量引用
           std::vector<float> & logits,  // 浮点数向量的引用
    const     whisper_grammar & grammar) {  // 语法对象的常量引用

    // 如果语法规则为空或语法栈为空，则直接返回
    if (grammar.rules.empty() || grammar.stacks.empty()) {
        return;
    }

    // 创建一个结束符号对象
    const whisper_token eot = whisper_token_eot(&ctx);

    // 创建两个空向量，用于存储解码后的候选项和语法候选项
    std::vector<std::pair<std::vector<uint32_t>, whisper_partial_utf8>> candidates_decoded;
    std::vector<whisper_grammar_candidate>                              candidates_grammar;

    // 遍历结束符号之前的所有标记
    for (whisper_token id = 0; id < eot; ++id) {
        // 获取标记对应的文本
        const std::string & text = ctx.vocab.id_to_token[id];
        // 如果文本不为空，则解码文本并存储到候选项中
        if (!text.empty()) {
            candidates_decoded.push_back(decode_utf8(text.c_str(), grammar.partial_utf8));
            candidates_grammar.push_back({ id, candidates_decoded.back().first.data(), candidates_decoded.back().second });
        }
    }

    // 使用语法规则和栈以及候选项，拒绝一些候选项
    const auto rejects = whisper_grammar_reject_candidates(grammar.rules, grammar.stacks, candidates_grammar);

    // 对于每一个被拒绝的候选项，减去语法惩罚值
    for (const auto & reject : rejects) {
        logits[reject.id] -= params.grammar_penalty;
    }

    // 当语法允许继续时，对结束符号进行惩罚
    //if (!allow_eot) {
    //    logits[eot] -= params.grammar_penalty;
    //}
    //fprintf(stderr, "Allowed: (%zu tokens)\n", size - rejects.size());
}

// 静态函数，用于接受标记并更新语法
static void whisper_grammar_accept_token(whisper_context & ctx, whisper_grammar & grammar, whisper_token token) {
    // 如果语法规则为空或语法栈为空，则直接返回
    if (grammar.rules.empty() || grammar.stacks.empty()) {
        return;
    }

    //fprintf(stderr, "Accept: '%s'\n", ctx.vocab.id_to_token[token].c_str());

    // 获取标记对应的文本
    const std::string & text = ctx.vocab.id_to_token[token];

    // 如果文本以"[_"开头，则跳过
    if (text.rfind("[_", 0) == 0) {
        // fprintf(stderr, " (skipped)\n");
        return;
    }
}
    // 打印换行符到标准错误流
    // 注意：解码后的字符串需要以终止符0结尾
    const auto   decoded     = decode_utf8(text.c_str(), grammar.partial_utf8);
    // 获取解码后的码点
    const auto & code_points = decoded.first;
    // 遍历码点，处理除最后一个之外的所有码点
    for (auto it = code_points.begin(), end = code_points.end() - 1; it != end; ++it) {
        // 使用语法规则接受码点，更新语法栈
        grammar.stacks = whisper_grammar_accept(grammar.rules, grammar.stacks, *it);
    }
    // 更新部分 UTF-8 编码
    grammar.partial_utf8 = decoded.second;
// 结束语法
}

// 默认参数结构体的引用
struct whisper_context_params * whisper_context_default_params_by_ref() {
    // 获取默认的上下文参数
    struct whisper_context_params params = whisper_context_default_params();

    // 创建一个新的上下文参数结构体指针
    struct whisper_context_params* result = new whisper_context_params();
    // 将默认参数赋值给新创建的结构体指针
    *result = params;
    // 返回新创建的结构体指针
    return result;
}

// 默认完整参数结构体的引用
struct whisper_full_params * whisper_full_default_params_by_ref(enum whisper_sampling_strategy strategy) {
    // 获取默认的完整参数
    struct whisper_full_params params = whisper_full_default_params(strategy);

    // 创建一个新的完整参数结构体指针
    struct whisper_full_params* result = new whisper_full_params();
    // 将默认参数赋值给新创建的结构体指针
    *result = params;
    // 返回新创建的结构体指针
    return result;
}

// 获取默认的完整参数
struct whisper_full_params whisper_full_default_params(enum whisper_sampling_strategy strategy) {
    // 创建一个默认参数结构体
    struct whisper_full_params result = {};

    // 根据策略进行不同的设置
    switch (strategy) {
        case WHISPER_SAMPLING_GREEDY:
            {
                // 设置贪婪策略的参数
                result.greedy = {
                    /*.best_of   =*/ 5,
                };
            } break;
        case WHISPER_SAMPLING_BEAM_SEARCH:
            {
                // 设置波束搜索策略的参数
                result.beam_search = {
                    /*.beam_size =*/ 5,
                    /*.patience  =*/ -1.0f,
                };
            } break;
    }

    // 返回设置好的参数结构体
    return result;
}

// 前向声明
static std::vector<float> get_signal_energy(const float * signal, int n_samples, int n_samples_per_half_window);
static void whisper_exp_compute_token_level_timestamps(
        struct whisper_context & ctx,
        struct whisper_state & state,
        int   i_segment,
        float   thold_pt,
        float   thold_ptsum);

// 内联函数，根据是否在单词上进行分割
static inline bool should_split_on_word(const char * txt, bool split_on_word) {
    // 如果不需要在单词上进行分割，则返回true
    if (!split_on_word) return true;

    // 否则根据第一个字符是否为空格来决定是否进行分割
    return txt[0] == ' ';
}

// 将最后一个段落包装到最大长度的字符
// 返回新段落数量
// 对给定的段落进行包装，确保每个段落的长度不超过最大长度，并且可以选择是否在单词边界处分割
static int whisper_wrap_segment(struct whisper_context & ctx, struct whisper_state & state, int max_len, bool split_on_word) {
    // 获取最后一个结果段落
    auto segment = state.result_all.back();

    // 初始化返回值和累积长度
    int res = 1;
    int acc = 0;

    // 初始化文本字符串
    std::string text;

    // 遍历段落中的每个标记
    for (int i = 0; i < (int) segment.tokens.size(); i++) {
        // 获取当前标记
        const auto & token = segment.tokens[i];
        // 如果标记的 ID 大于等于结束标记的 ID，则跳过
        if (token.id >= whisper_token_eot(&ctx)) {
            continue;
        }

        // 将标记转换为字符串
        const auto txt = whisper_token_to_str(&ctx, token.id);
        const int cur = strlen(txt);

        // 如果累积长度加上当前标记长度超过最大长度，并且不是第一个标记，并且应该在单词边界处分割
        if (acc + cur > max_len && i > 0 && should_split_on_word(txt, split_on_word)) {
            // 将文本字符串赋值给当前结果段落的文本
            state.result_all.back().text = std::move(text);
            state.result_all.back().t1 = token.t0;
            state.result_all.back().tokens.resize(i);
            state.result_all.back().speaker_turn_next = false;

            // 创建新的结果段落
            state.result_all.push_back({});
            state.result_all.back().t0 = token.t0;
            state.result_all.back().t1 = segment.t1;

            // 将标记[i, end]添加到新的段落中
            state.result_all.back().tokens.insert(
                state.result_all.back().tokens.end(),
                    segment.tokens.begin() + i,
                    segment.tokens.end());

            state.result_all.back().speaker_turn_next = segment.speaker_turn_next;

            // 重置累积长度和文本字符串
            acc = 0;
            text = "";

            // 更新当前段落为新创建的段落
            segment = state.result_all.back();
            i = -1;

            // 增加返回值
            res++;
        } else {
            // 更新累积长度和文本字符串
            acc += cur;
            text += txt;
        }
    }

    // 将文本字符串赋值给当前结果段落的文本
    state.result_all.back().text = std::move(text);

    // 返回结果
    return res;
}

// 非语音标记的列表
static const std::vector<std::string> non_speech_tokens = {
    "\"", "#", "(", ")", "*", "+", "/", ":", ";", "<", "=", ">", "@", "[", "\\", "]", "^",
    "_", "`", "{", "|", "}", "~", "「", "」", "『", "』", "<<", ">>", "<<<", ">>>", "--",
    "---", "-(", "-[", "('", "(\"", "((", "))", "(((", ")))", "[[", "]]", "{{", "}}", "♪♪",
    "♪♪♪","♩", "♪", "♫", "♬", "♭", "♮", "♯"
};
// 处理选择的解码器的logits
// - 应用logit过滤器
// - 计算logprobs和probs
// TODO: 优化
static void whisper_process_logits(
              struct whisper_context & ctx,
               struct whisper_state  & state,
              struct whisper_decoder & decoder,
    const struct whisper_full_params   params,
                               float   temperature) {
    const auto & vocab      = ctx.vocab;  // 获取词汇表
    const auto & tokens_cur = decoder.sequence.tokens;  // 获取当前解码器的tokens

    const bool is_initial = tokens_cur.size() == 0;  // 检查是否为初始状态
    const int  n_logits   = vocab.id_to_token.size();  // 获取logits的数量

    WHISPER_ASSERT(n_logits == ctx.vocab.n_vocab);  // 断言logits数量与词汇表数量相等

    // 提取最后一个token的logits
    // 我们将进行改变，因此不希望直接使用ctx.logits缓冲区
    auto & probs    = decoder.probs;  // 概率
    auto & logits   = decoder.logits;  // logits
    auto & logprobs = decoder.logprobs;  // logprobs
    {
        logits.resize(n_logits);  // 调整logits的大小
        memcpy(logits.data(), state.logits.data() + decoder.i_batch*n_logits, n_logits*sizeof(float));  // 复制logits数据

        if (temperature > 0.0f) {  // 如果温度大于0
            for (int i = 0; i < n_logits; i++) {  // 遍历logits
                logits[i] /= temperature;  // 应用温度
            }
        }

        //稍后将被填充
        probs.resize(n_logits);  // 调整概率的大小
        logprobs.resize(n_logits);  // 调整logprobs的大小
    }

    // 在这里应用logit过滤器
    // 参考: https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L480-L493
    }

    // 计算概率
    {
        for (int i = 0; i < n_logits; ++i) {  // 遍历logits
            if (logits[i] == -INFINITY) {  // 如果logits为负无穷
                probs[i] = 0.0f;  // 概率为0
            } else {
                probs[i] = expf(logprobs[i]);  // 计算概率
            }
        }
    }

#if 0
    // 打印前100个logits - token字符串：logit
    //for (int i = 0; i < 10; i++) {
    //    const auto token   = vocab.id_to_token.at(i);
    //    const auto prob    = probs[i];
    //    const auto logit   = logits[i];
    // print sorted
    {
        // 创建一个存储概率和索引对的向量
        std::vector<std::pair<float, int>> pairs;

        // 遍历概率数组，将概率和索引组成的对添加到pairs中
        for (int i = 0; i < n_logits; ++i) {
            pairs.push_back(std::make_pair(probs[i], i));
        }

        // 对pairs按照概率值进行排序
        std::sort(pairs.begin(), pairs.end(), [](const std::pair<float, int>& a, const std::pair<float, int>& b) {
            return a.first > b.first;
        });

        // 打印排序后的前10个概率和对应的token、logit、logprob
        for (int i = 0; i < 10; i++) {
            const auto token   = vocab.id_to_token.at(pairs[i].second);
            const auto prob    = pairs[i].first;
            const auto logit   = logits[pairs[i].second];
            const auto logprob = logprobs[pairs[i].second];
            printf("%16s : id=%6d prob=%9.5f logit=%9.5f logprob=%9.5f '%s'\n", token.c_str(), pairs[i].second, prob, logit, logprob, token.c_str());
        }

        // 打印分隔线
        printf("----------------\n");
    }
    // 打印 probs 字典中 " and" 对应的值
    // 打印 probs 字典中 " And" 对应的值
    // 打印 probs 字典中 " so" 对应的值
#endif
}

static whisper_token_data whisper_sample_token(
            whisper_context & ctx,
      const whisper_decoder & decoder,
                       bool   best) {
    whisper_token_data result = {
        0, 0, 0.0f, 0.0f, 0.0f, 0.0f, -1, -1, 0.0f,
    };

    const auto & vocab = ctx.vocab;

    const auto & probs    = decoder.probs;
    const auto & logprobs = decoder.logprobs;

    const int n_logits = vocab.n_vocab;

    {
        // 计算总概率和最大概率
        double sum_ts = 0.0;
        double max_ts = 0.0;

        for (int i = vocab.token_beg; i < n_logits; i++) {
            // 如果概率为负无穷，则跳过
            if (probs[i] == -INFINITY) {
                continue;
            }

            sum_ts += probs[i];
            if (max_ts < probs[i]) {
                max_ts = probs[i];
                result.tid = i;
            }
        }

        // 计算最大概率占总概率的比例和总概率
        result.pt    = max_ts/(sum_ts + 1e-10);
        result.ptsum = sum_ts;
    }

    if (best) {
        // 如果选择最佳结果
        for (int i = 0; i < n_logits; ++i) {
            // 更新结果的标识、概率和对数概率
            if (result.p < probs[i]) {
                result.id   = i;
                result.p    = probs[i];
                result.plog = logprobs[i];
            }
        }
    } else {
        // 如果不选择最佳结果，使用离散分布进行采样
        std::discrete_distribution<> dist(probs.begin(), probs.end());

        result.id   = dist(decoder.rng);
        result.p    = probs[result.id];
        result.plog = logprobs[result.id];
    }

    // 如果结果标识大于等于词汇表的起始标识，则更新结果的标识和最大概率占总概率的比例
    if (result.id >= vocab.token_beg) {
        result.tid = result.id;
        result.pt  = result.p;
    }

    return result;
}

static std::vector<whisper_token_data> whisper_sample_token_topk(
            whisper_context & ctx,
            whisper_decoder & decoder,
                        int   k) {
    const auto & vocab = ctx.vocab;

    const auto & probs    = decoder.probs;
    const auto & logits   = decoder.logits;
    const auto & logprobs = decoder.logprobs;

    const int n_logits = vocab.n_vocab;

    auto & logits_id = decoder.logits_id;

    // 调整 logits_id 的大小为词汇表大小
    logits_id.resize(n_logits);
    # 遍历 logits 数组，将每个元素和其索引组成 pair 存入 logits_id 数组
    for (int i = 0; i < n_logits; ++i) {
        logits_id[i].first = logits[i];
        logits_id[i].second = i;
    }

    {
        # 定义 pair_type 类型为 logits_id 的 value_type
        using pair_type = std::remove_reference<decltype(logits_id)>::type::value_type;
        # 对 logits_id 数组进行部分排序，只保留前 k 个元素，按照第一个元素（logits 值）降序排列
        std::partial_sort(
                logits_id.begin(),
                logits_id.begin() + k, logits_id.end(),
                [](const pair_type & a, const pair_type & b) {
            return a.first > b.first;
        });
    }

    # 创建一个空的 whisper_token_data 数组，预留 k 个元素的空间
    std::vector<whisper_token_data> result;
    result.reserve(k);

    # 初始化 whisper_token 和两个浮点数变量
    whisper_token tid = vocab.token_beg;
    float pt    = 0.0;
    float ptsum = 0.0;

    {
        # 初始化两个浮点数变量，用于计算概率
        double sum_ts = 0.0;
        double max_ts = 0.0;

        # 遍历 probs 数组，计算 sum_ts 和 max_ts，并更新 tid
        for (int i = vocab.token_beg; i < n_logits; i++) {
            if (probs[i] == -INFINITY) {
                continue;
            }

            sum_ts += probs[i];
            if (max_ts < probs[i]) {
                max_ts = probs[i];
                tid = i;
            }
        }

        # 计算 pt 和 ptsum
        pt    = max_ts/(sum_ts + 1e-10);
        ptsum = sum_ts;
    }

    # 创建一个离散分布对象 dist，以 probs 数组为参数
    std::discrete_distribution<> dist(probs.begin(), probs.end());

    # 遍历 k 次，根据 dist 随机选择一个 id，并将相关信息存入 result 数组
    for (int i = 0; i < k; ++i) {
        const auto id = dist(decoder.rng);
        # 将相关信息存入 result 数组
        result.push_back({ id, tid, probs[id], logprobs[id], pt, ptsum, -1, -1, 0.0f, });

        # 如果 result[i].id 大于等于 vocab.token_beg，则更新 tid 和 pt
        if (result[i].id >= vocab.token_beg) {
            result[i].tid = result[i].id;
            result[i].pt  = result[i].p;
        }
    }

    # 返回 result 数组
    return result;
// 定义一个静态函数，用于计算序列的得分和熵
static void whisper_sequence_score(
        const struct whisper_full_params & params,
                        whisper_sequence & sequence) {
    // 如果序列的结果长度为0，则直接返回
    if (sequence.result_len == 0) {
        return;
    }

    // 初始化结果值为0
    double result = 0.0f;

    // 遍历序列中的结果，计算结果的对数概率之和
    for (int i = 0; i < sequence.result_len; ++i) {
        result += sequence.tokens[i].plog;
    }

    // 将对数概率之和赋值给序列的对数概率之和和平均对数概率
    sequence.sum_logprobs = result;
    sequence.avg_logprobs = result/sequence.result_len;

    // 初始化惩罚值为结果长度
    double penalty = sequence.result_len;

    // 如果长度惩罚大于0，则计算惩罚值
    if (params.length_penalty > 0.0f) {
        penalty = pow((5.0 + penalty)/6.0, params.length_penalty);
    }

    // 将结果值除以惩罚值作为序列的得分
    sequence.score = result/penalty;

    // 计算序列最后32个标记的熵
    {
        // 定义常量n为32
        const int n = 32;

        // 初始化计数和熵的值
        int cnt = 0;
        double entropy = 0.0f;

        // 创建标记和出现次数的映射
        std::map<whisper_token, int> token_counts;
        // 遍历最后n个标记，统计每个标记的出现次数
        for (int i = std::max(0, sequence.result_len - n); i < sequence.result_len; ++i) {
            token_counts[sequence.tokens[i].id]++;
            cnt++;
        }

        // 计算每个标记的概率和熵
        for (const auto & kv : token_counts) {
            const auto p = kv.second/(double)cnt;
            entropy -= p*log(p);

            //WHISPER_LOG_DEBUG("entropy: %d %f %f, count %d\n", kv.first, p, log(p), kv.second);
        }

        // 将计算得到的熵赋值给序列的熵
        sequence.entropy = entropy;
    }
}

// 使用状态进行完整的whisper推理
int whisper_full_with_state(
        struct whisper_context * ctx,
          struct whisper_state * state,
    struct whisper_full_params   params,
                   const float * samples,
                           int   n_samples) {
    // 清空旧的结果
    auto & result_all = state->result_all;

    result_all.clear();
    // 如果样本数大于0
    if (n_samples > 0) {
        // 计算对数梅尔频谱
        if (params.speed_up) {
            // 如果需要加速，则输出错误信息并返回-1
            WHISPER_LOG_ERROR("%s: failed to compute log mel spectrogram\n", __func__);
            return -1;
        } else {
            // 否则，使用whisper_pcm_to_mel_with_state函数计算对数梅尔频谱
            if (whisper_pcm_to_mel_with_state(ctx, state, samples, n_samples, params.n_threads) != 0) {
                // 如果计算失败，则输出错误信息并返回-2
                WHISPER_LOG_ERROR("%s: failed to compute log mel spectrogram\n", __func__);
                return -2;
            }
        }
    }

    // 如果语言未指定，则自动检测语言
    if (params.language == nullptr || strlen(params.language) == 0 || strcmp(params.language, "auto") == 0 || params.detect_language) {
        // 初始化概率向量
        std::vector<float> probs(whisper_lang_max_id() + 1, 0.0f);

        // 自动检测语言并返回语言ID
        const auto lang_id = whisper_lang_auto_detect_with_state(ctx, state, 0, params.n_threads, probs.data());
        if (lang_id < 0) {
            // 如果自动检测失败，则输出错误信息并返回-3
            WHISPER_LOG_ERROR("%s: failed to auto-detect language\n", __func__);
            return -3;
        }
        // 将检测到的语言ID存储到状态中
        state->lang_id = lang_id;
        // 将参数中的语言指针指向检测到的语言字符串
        params.language = whisper_lang_str(lang_id);

        // 输出自动检测到的语言信息
        WHISPER_LOG_INFO("%s: auto-detected language: %s (p = %f)\n", __func__, params.language, probs[whisper_lang_id(params.language)]);
        // 如果需要检测语言，则返回0
        if (params.detect_language) {
            return 0;
        }
    }

    // 如果需要输出时间戳
    if (params.token_timestamps) {
        // 重置时间戳相关状态
        state->t_beg    = 0;
        state->t_last   = 0;
        state->tid_last = 0;
        // 如果样本数大于0，则计算信号能量
        if (n_samples > 0) {
            state->energy = get_signal_energy(samples, n_samples, 32);
        }
    }

    // 计算起始帧和结束帧
    const int seek_start = params.offset_ms/10;
    const int seek_end = params.duration_ms == 0 ? whisper_n_len_from_state(state) : seek_start + params.duration_ms/10;

    // 如果频谱长度小于1.0秒（100帧），则返回
    // 基本上不处理小于1.0秒的任何内容
    // 参见问题＃39：https://github.com/ggerganov/whisper.cpp/issues/39
    // 如果结束位置减去开始位置小于 (params.speed_up ? 50 : 100) 毫秒，则输出调试信息并返回 0
    if (seek_end < seek_start + (params.speed_up ? 50 : 100)) {
        WHISPER_LOG_DEBUG("%s: input is too short - %d ms < 1000 ms\n", __func__, (seek_end - seek_start)*10);
        return 0;
    }

    // 一组要使用的温度值
    // [ t0, t0 + delta, t0 + 2*delta, ..., < 1.0f + 1e-6f ]
    std::vector<float> temperatures;
    // 如果温度增量大于 0.0f，则根据温度增量循环生成一组温度值
    if (params.temperature_inc > 0.0f) {
        for (float t = params.temperature; t < 1.0f + 1e-6f; t += params.temperature_inc) {
            temperatures.push_back(t);
        }
    } else {
        // 否则只使用一个温度值
        temperatures.push_back(params.temperature);
    }

    // 初始化解码器数量
    int n_decoders = 1;

    // 根据策略设置解码器数量
    switch (params.strategy) {
        case WHISPER_SAMPLING_GREEDY:
            {
                n_decoders = params.greedy.best_of;
            } break;
        case WHISPER_SAMPLING_BEAM_SEARCH:
            {
                n_decoders = std::max(params.greedy.best_of, params.beam_search.beam_size);
            } break;
    };

    // 确保解码器数量不小于 1
    n_decoders = std::max(1, n_decoders);

    // 如果解码器数量超过最大值，则输出错误信息并返回 -4
    if (n_decoders > WHISPER_MAX_DECODERS) {
        WHISPER_LOG_ERROR("%s: too many decoders requested (%d), max = %d\n", __func__, n_decoders, WHISPER_MAX_DECODERS);
        return -4;
    }

    // 标记：WHISPER_DECODER_INIT
    // 初始化解码器
    for (int j = 1; j < n_decoders; j++) {
        auto & decoder = state->decoders[j];

        decoder.sequence.tokens.reserve(state->decoders[0].sequence.tokens.capacity());

        decoder.probs.resize   (ctx->vocab.n_vocab);
        decoder.logits.resize  (ctx->vocab.n_vocab);
        decoder.logprobs.resize(ctx->vocab.n_vocab);
        decoder.logits_id.reserve(ctx->model.hparams.n_vocab);

        decoder.rng = std::mt19937(0);
    }

    // 到目前为止累积的文本上下文
    auto & prompt_past = state->prompt_past;
    // 如果禁用上下文，则清空上下文
    if (params.no_context) {
        prompt_past.clear();
    }

    // 准备提示
    {
        // 创建一个名为 prompt_tokens 的空向量
        std::vector<whisper_token> prompt_tokens;

        // 如果参数中没有 prompt_tokens 且存在 initial_prompt，则执行以下操作
        if (!params.prompt_tokens && params.initial_prompt) {
            // 调整 prompt_tokens 的大小为 1024
            prompt_tokens.resize(1024);
            // 调整 prompt_tokens 的大小为实际 token 数量
            prompt_tokens.resize(whisper_tokenize(ctx, params.initial_prompt, prompt_tokens.data(), prompt_tokens.size()));
            // 将 prompt_tokens 的指针赋值给 params.prompt_tokens
            params.prompt_tokens   = prompt_tokens.data();
            // 将 prompt_tokens 的大小赋值给 params.prompt_n_tokens
            params.prompt_n_tokens = prompt_tokens.size();
        }

        // 如果存在 prompt_tokens 且 prompt_n_tokens 大于 0，则执行以下操作
        if (params.prompt_tokens && params.prompt_n_tokens > 0) {
            // 从指针中解析 tokens
            for (int i = 0; i < params.prompt_n_tokens; i++) {
                // 将 params.prompt_tokens[i] 添加到 prompt_past 的末尾
                prompt_past.push_back(params.prompt_tokens[i]);
            }
            // 将 prompt_past 中的元素循环左移 params.prompt_n_tokens 个位置
            std::rotate(prompt_past.begin(), prompt_past.end() - params.prompt_n_tokens, prompt_past.end());
        }
    }

    // 覆盖 audio_ctx，最大允许值为 hparams.n_audio_ctx
    if (params.audio_ctx > whisper_n_audio_ctx(ctx)) {
        // 如果 params.audio_ctx 大于最大允许值，则记录错误信息并返回 -5
        WHISPER_LOG_ERROR("%s: audio_ctx is larger than the maximum allowed (%d > %d)\n", __func__, params.audio_ctx, whisper_n_audio_ctx(ctx));
        return -5;
    }
    // 将参数中的 audio_ctx 赋值给 state->exp_n_audio_ctx
    state->exp_n_audio_ctx = params.audio_ctx;

    // 这些 tokens 决定将执行的任务
    // 创建一个名为 prompt_init 的向量，包含一个起始 token
    std::vector<whisper_token> prompt_init = { whisper_token_sot(ctx), };

    // 如果是多语言环境，则执行以下操作
    if (whisper_is_multilingual(ctx)) {
        // 获取语言的 ID，并赋值给 state->lang_id
        const int lang_id = whisper_lang_id(params.language);
        state->lang_id = lang_id;
        // 将语言 token 添加到 prompt_init 中
        prompt_init.push_back(whisper_token_lang(ctx, lang_id));
        // 如果需要翻译，则添加翻译 token；否则添加转录 token
        if (params.translate) {
            prompt_init.push_back(whisper_token_translate(ctx));
        } else {
            prompt_init.push_back(whisper_token_transcribe(ctx));
        }
    }

    // 精简模型需要 "no_timestamps" token
    // 检查模型是否为蒸馏模型，并且不禁用时间戳
    const bool is_distil = ctx->model.hparams.n_text_layer == 2;
    if (is_distil && !params.no_timestamps) {
        // 如果是蒸馏模型且不禁用时间戳，则记录警告日志，并强制禁用时间戳
        WHISPER_LOG_WARN("%s: using distilled model - forcing no_timestamps\n", __func__);
        params.no_timestamps = true;
    }

    // 如果禁用了时间戳，则将初始化的提示信息添加到 prompt_init 中
    if (params.no_timestamps) {
        prompt_init.push_back(whisper_token_not(ctx));
    }

    // 初始化 seek 变量为 seek_start
    int seek = seek_start;

    // 初始化 prompt 变量为 whisper_token 类型的向量，预留空间为文本上下文的大小
    std::vector<whisper_token> prompt;
    prompt.reserve(whisper_n_text_ctx(ctx));

    // 定义 beam_candidate 结构体，包含 decoder_idx、seek_delta、has_ts、sequence 和 grammar 属性
    struct beam_candidate {
        int decoder_idx;
        int seek_delta;
        bool has_ts;
        whisper_sequence sequence;
        whisper_grammar grammar;
    };

    // 初始化 bc_per_dec 变量为包含 n_decoders 个空向量的向量
    std::vector<std::vector<beam_candidate>> bc_per_dec(n_decoders);

    // 初始化 beam_candidates 变量为 beam_candidate 类型的向量
    std::vector<beam_candidate> beam_candidates;

    // 主循环
#ifdef WHISPER_DEBUG
                        {
                            // 如果定义了 WHISPER_DEBUG 宏，则输出调试信息
                            const auto tt = token.pt > 0.10 ? ctx->vocab.id_to_token.at(token.tid) : "[?]";
                            // 使用 WHISPER_LOG_DEBUG 宏输出调试信息
                            WHISPER_LOG_DEBUG("%s: id = %3d, decoder = %d, token = %6d, p = %6.3f, ts = %10s, %6.3f, result_len = %4d '%s'\n",
                                    __func__, i, j, token.id, token.p, tt.c_str(), token.pt, result_len, ctx->vocab.id_to_token.at(token.id).c_str());
                        }
    }

    return 0;
}

int whisper_full(
        struct whisper_context * ctx,
    struct whisper_full_params   params,
                   const float * samples,
                           int   n_samples) {
    return whisper_full_with_state(ctx, ctx->state, params, samples, n_samples);
}

int whisper_full_parallel(
        struct whisper_context * ctx,
        struct whisper_full_params params,
        const float * samples,
        int n_samples,
        int n_processors) {
    if (n_processors == 1) {
        return whisper_full(ctx, params, samples, n_samples);
    }
    int ret = 0;

    // 为每个线程准备单独的状态
    std::vector<whisper_state*> states;

    // 计算偏移样本数和每个处理器的样本数
    const int offset_samples = (WHISPER_SAMPLE_RATE*params.offset_ms)/1000;
    const int n_samples_per_processor = (n_samples - offset_samples)/n_processors;

    // 调用线程将处理第一块样本，其它线程将处理剩余的块

    // 创建线程数组
    std::vector<std::thread> workers(n_processors - 1);
    // 对每个处理器创建一个新的状态
    for (int i = 0; i < n_processors - 1; ++i) {
        states.push_back(whisper_init_state(ctx));

        // 计算每个处理器开始处理的样本数
        const int start_samples = offset_samples + (i + 1)*n_samples_per_processor;
        // 计算每个处理器处理的样本数
        const int n_samples_cur = (i == n_processors - 2) ? n_samples - start_samples : n_samples_per_processor;

        // 复制参数到当前参数
        auto params_cur = params;

        // 重置当前参数的偏移时间和打印进度和实时打印
        params_cur.offset_ms = 0;
        params_cur.print_progress = false;
        params_cur.print_realtime = false;

        // 重置当前参数的新段回调和进度回调
        params_cur.new_segment_callback = nullptr;
        params_cur.new_segment_callback_user_data = nullptr;
        params_cur.progress_callback = nullptr;
        params_cur.progress_callback_user_data = nullptr;

        // 使用当前状态和参数创建一个新的线程
        workers[i] = std::thread(whisper_full_with_state, ctx, states[i], std::move(params_cur), samples + start_samples, n_samples_cur);
    }

    {
        // 复制参数到当前参数
        auto params_cur = params;

        // 禁用实时打印
        params_cur.print_realtime = false;

        // 运行第一个转换，使用默认状态，但只针对第一个块
        ret = whisper_full_with_state(ctx, ctx->state, std::move(params_cur), samples, offset_samples + n_samples_per_processor);
    }

    // 等待所有线程结束
    for (int i = 0; i < n_processors - 1; ++i) {
        workers[i].join();
    }

    // 计算偏移时间
    const int64_t offset_t = (int64_t) params.offset_ms/10.0;

    // 将所有其他状态的结果合并到result_state->result_all中
    // 遍历处理器，从第一个到倒数第二个
    for (int i = 0; i < n_processors - 1; ++i) {
        // 获取第 i 个处理器的结果引用
        auto& results_i = states[i]->result_all;

        // 遍历第 i 个处理器的结果
        for (auto& result : results_i) {
            // 校正段时间戳，考虑偏移量
            result.t0 += 100 * ((i + 1) * n_samples_per_processor) / WHISPER_SAMPLE_RATE + offset_t;
            result.t1 += 100 * ((i + 1) * n_samples_per_processor) / WHISPER_SAMPLE_RATE + offset_t;

            // 确保段不重叠
            if (!ctx->state->result_all.empty()) {
                result.t0 = std::max(result.t0, ctx->state->result_all.back().t1);
            }

            // 将结果移动到上下文状态的结果中
            ctx->state->result_all.push_back(std::move(result));

            // 对每个段调用新段回调函数
            if (params.new_segment_callback) {
                params.new_segment_callback(ctx, ctx->state, 1, params.new_segment_callback_user_data);
            }
        }

        // 更新上下文状态的 mel 时间
        ctx->state->t_mel_us += states[i]->t_mel_us;

        // 更新上下文状态的 sample 时间
        ctx->state->t_sample_us += states[i]->t_sample_us;
        // 更新上下文状态的 encode 时间
        ctx->state->t_encode_us += states[i]->t_encode_us;
        // 更新上下文状态的 decode 时间
        ctx->state->t_decode_us += states[i]->t_decode_us;
        // 更新上下文状态的 batchd 时间
        ctx->state->t_batchd_us += states[i]->t_batchd_us;
        // 更新上下文状态的 prompt 时间
        ctx->state->t_prompt_us += states[i]->t_prompt_us;

        // 更新上下文状态的样本数
        ctx->state->n_sample += states[i]->n_sample;
        // 更新上下文状态的编码数
        ctx->state->n_encode += states[i]->n_encode;
        // 更新上下文状态的解码数
        ctx->state->n_decode += states[i]->n_decode;
        // 更新上下文状态的批处理数
        ctx->state->n_batchd += states[i]->n_batchd;
        // 更新上下文状态的提示数
        ctx->state->n_prompt += states[i]->n_prompt;

        // 释放处理器状态
        whisper_free_state(states[i]);
    }

    // 计算时间的平均值
    ctx->state->t_mel_us    /= n_processors;
    ctx->state->t_sample_us /= n_processors;
    ctx->state->t_encode_us /= n_processors;
    ctx->state->t_decode_us /= n_processors;

    // 打印关于音频边界的信息
    WHISPER_LOG_WARN("\n");
    WHISPER_LOG_WARN("%s: the audio has been split into %d chunks at the following times:\n", __func__, n_processors);
    # 循环遍历处理器数量减一次
    for (int i = 0; i < n_processors - 1; ++i) {
        # 输出警告日志，记录分割信息和时间戳
        WHISPER_LOG_WARN("%s: split %d - %s\n", __func__, (i + 1), to_timestamp(100*((i + 1)*n_samples_per_processor)/WHISPER_SAMPLE_RATE + offset_t).c_str());
    }
    # 输出警告日志，提醒可能在边界附近降低转录质量
    WHISPER_LOG_WARN("%s: the transcription quality may be degraded near these boundaries\n", __func__);

    # 返回结果
    return ret;
// 从状态结构体中获取所有结果段的数量
int whisper_full_n_segments_from_state(struct whisper_state * state) {
    return state->result_all.size();
}

// 获取上下文结构体中所有结果段的数量
int whisper_full_n_segments(struct whisper_context * ctx) {
    return ctx->state->result_all.size();
}

// 从状态结构体中获取语言ID
int whisper_full_lang_id_from_state(struct whisper_state * state) {
    return state->lang_id;
}

// 获取上下文结构体中的语言ID
int whisper_full_lang_id(struct whisper_context * ctx) {
    return ctx->state->lang_id;
}

// 从状态结构体中获取指定结果段的起始时间
int64_t whisper_full_get_segment_t0_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].t0;
}

// 获取上下文结构体中指定结果段的起始时间
int64_t whisper_full_get_segment_t0(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].t0;
}

// 从状态结构体中获取指定结果段的结束时间
int64_t whisper_full_get_segment_t1_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].t1;
}

// 获取上下文结构体中指定结果段的结束时间
int64_t whisper_full_get_segment_t1(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].t1;
}

// 从状态结构体中获取指定结果段的说话者转换标志
bool whisper_full_get_segment_speaker_turn_next_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].speaker_turn_next;
}

// 获取上下文结构体中指定结果段的说话者转换标志
bool whisper_full_get_segment_speaker_turn_next(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].speaker_turn_next;
}

// 从状态结构体中获取指定结果段的文本内容
const char * whisper_full_get_segment_text_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].text.c_str();
}

// 获取上下文结构体中指定结果段的文本内容
const char * whisper_full_get_segment_text(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].text.c_str();
}

// 从状态结构体中获取指定结果段的标记数量
int whisper_full_n_tokens_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].tokens.size();
}

// 获取上下文结构体中指定结果段的标记数量
int whisper_full_n_tokens(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].tokens.size();
}
// 从给定状态中获取指定段落和标记的文本内容
const char * whisper_full_get_token_text_from_state(struct whisper_context * ctx, struct whisper_state * state, int i_segment, int i_token) {
    return ctx->vocab.id_to_token[state->result_all[i_segment].tokens[i_token].id].c_str();
}

// 获取指定上下文中指定段落和标记的文本内容
const char* whisper_full_get_token_text(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->vocab.id_to_token[ctx->state->result_all[i_segment].tokens[i_token].id].c_str();
}

// 从给定状态中获取指定段落和标记的标记 ID
whisper_token whisper_full_get_token_id_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token].id;
}

// 获取指定上下文中指定段落和标记的标记 ID
whisper_token whisper_full_get_token_id(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token].id;
}

// 从给定状态中获取指定段落和标记的标记数据
struct whisper_token_data whisper_full_get_token_data_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token];
}

// 获取指定上下文中指定段落和标记的标记数据
struct whisper_token_data whisper_full_get_token_data(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token];
}

// 从给定状态中获取指定段落和标记的概率值
float whisper_full_get_token_p_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token].p;
}

// 获取指定上下文中指定段落和标记的概率值
float whisper_full_get_token_p(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token].p;
}

// 临时接口，用于暴露 ggml 接口，将来 ggml 成为独立库时将被移除
WHISPER_API int whisper_bench_memcpy(int n_threads) {
    // 将 whisper_bench_memcpy_str 返回的字符串输出到标准错误流
    fputs(whisper_bench_memcpy_str(n_threads), stderr);
    return 0;
}

// 返回描述 memcpy 性能的字符串
WHISPER_API const char * whisper_bench_memcpy_str(int n_threads) {
    // 静态字符串对象，用于存储描述 memcpy 性能的字符串
    static std::string s;
    // 清空字符串内容
    s = "";
    // 字符串缓冲区
    char strbuf[256];

    // 初始化 ggml 时间
    ggml_time_init();
    // 定义一个大小为20的变量n
    size_t n    = 20;
    // 如果n_threads大于0，则arr为1024，否则为n_threads，用于避免编译器优化
    size_t arr  = n_threads > 0 ? 1024llu : n_threads; 

    // 定义一个大小为1GB的数组
    const size_t size = arr*1e6;

    // 定义一个double类型的变量sum，并初始化为0.0
    double sum  = 0.0;

    // 预热
    {
        // 分配大小为size的内存空间，并将其转换为char指针src
        char * src = (char *) malloc(size);
        // 分配大小为size的内存空间，并将其转换为char指针dst
        char * dst = (char *) malloc(size);

        // 将src数组中的元素赋值为i
        for (size_t i = 0; i < size; i++) src[i] = i;

        // 将src数组的内容复制到dst数组，用于预热
        memcpy(dst, src, size); // heat-up

        // 定义一个double类型的变量tsum，并初始化为0.0
        double tsum = 0.0;

        // 循环n次
        for (size_t i = 0; i < n; i++) {
            // 获取当前时间戳t0
            const int64_t t0 = ggml_time_us();

            // 将src数组的内容复制到dst数组
            memcpy(dst, src, size);

            // 获取当前时间戳t1
            const int64_t t1 = ggml_time_us();

            // 计算时间差并累加到tsum中
            tsum += (t1 - t0)*1e-6;

            // 修改src数组中随机位置的元素值
            src[rand() % size] = rand() % 256;
        }

        // 将结果格式化为字符串并存储到strbuf中
        snprintf(strbuf, sizeof(strbuf), "memcpy: %7.2f GB/s (heat-up)\n", (double) (n*size)/(tsum*1e9));
        // 将结果字符串追加到s中
        s += strbuf;

        // 防止编译器优化掉memcpy操作
        {
            // 循环遍历dst数组，将每个元素的值累加到sum中
            for (size_t i = 0; i < size; i++) sum += dst[i];
        }

        // 释放src和dst数组的内存空间
        free(src);
        free(dst);
    }

    // 单线程
    {
        // 分配大小为size的内存空间，并将其转换为char指针src
        char * src = (char *) malloc(size);
        // 分配大小为size的内存空间，并将其转换为char指针dst
        char * dst = (char *) malloc(size);

        // 将src数组中的元素赋值为i
        for (size_t i = 0; i < size; i++) src[i] = i;

        // 将src数组的内容复制到dst数组，用于预热
        memcpy(dst, src, size); // heat-up

        // 定义一个double类型的变量tsum，并初始化为0.0
        double tsum = 0.0;

        // 循环n次
        for (size_t i = 0; i < n; i++) {
            // 获取当前时间戳t0
            const int64_t t0 = ggml_time_us();

            // 将src数组的内容复制到dst数组
            memcpy(dst, src, size);

            // 获取当前时间戳t1
            const int64_t t1 = ggml_time_us();

            // 计算时间差并累加到tsum中
            tsum += (t1 - t0)*1e-6;

            // 修改src数组中随机位置的元素值
            src[rand() % size] = rand() % 256;
        }

        // 将结果格式化为字符串并存储到strbuf中
        snprintf(strbuf, sizeof(strbuf), "memcpy: %7.2f GB/s ( 1 thread)\n", (double) (n*size)/(tsum*1e9));
        // 将结果字符串追加到s中
        s += strbuf;

        // 防止编译器优化掉memcpy操作
        {
            // 循环遍历dst数组，将每个元素的值累加到sum中
            for (size_t i = 0; i < size; i++) sum += dst[i];
        }

        // 释放src和dst数组的内存空间
        free(src);
        free(dst);
    }

    // 多线程
    // 循环创建多个线程，每个线程分配内存并进行数据拷贝操作
    for (int32_t k = 1; k <= n_threads; k++) {
        // 为源数据和目标数据分配内存空间
        char * src = (char *) malloc(size);
        char * dst = (char *) malloc(size);

        // 初始化源数据
        for (size_t i = 0; i < size; i++) src[i] = i;

        // 对目标数据进行一次拷贝操作，用于预热
        memcpy(dst, src, size); // heat-up

        // 初始化时间总和
        double tsum = 0.0;

        // 定义一个 lambda 函数，用于在多线程中执行数据拷贝和修改操作
        auto helper = [&](int th) {
            // 根据线程数和当前线程编号计算数据拷贝的起始和结束位置
            const int64_t i0 = (th + 0)*size/k;
            const int64_t i1 = (th + 1)*size/k;

            // 循环执行数据拷贝和修改操作
            for (size_t i = 0; i < n; i++) {
                memcpy(dst + i0, src + i0, i1 - i0);

                // 在源数据中随机修改一个位置的数值
                src[i0 + rand() % (i1 - i0)] = rand() % 256;
            };
        };

        // 记录开始时间
        const int64_t t0 = ggml_time_us();

        // 创建多个线程，并将 helper 函数作为线程执行的函数
        std::vector<std::thread> threads(k - 1);
        for (int32_t th = 0; th < k - 1; ++th) {
            threads[th] = std::thread(helper, th);
        }

        // 在当前线程中执行 helper 函数
        helper(k - 1);

        // 等待其他线程执行完毕
        for (int32_t th = 0; th < k - 1; ++th) {
            threads[th].join();
        }

        // 记录结束时间
        const int64_t t1 = ggml_time_us();

        // 计算时间差，并累加到 tsum 中
        tsum += (t1 - t0)*1e-6;

        // 将结果格式化为字符串，添加到 s 中
        snprintf(strbuf, sizeof(strbuf), "memcpy: %7.2f GB/s (%2d thread)\n", (double) (n*size)/(tsum*1e9), k);
        s += strbuf;

        // 防止编译器优化掉 memcpy 操作，计算目标数据的和
        {
            for (size_t i = 0; i < size; i++) sum += dst[i];
        }

        // 释放源数据和目标数据的内存空间
        free(src);
        free(dst);
    }

    // 将目标数据的和格式化为字符串，添加到 s 中
    snprintf(strbuf, sizeof(strbuf), "sum:    %f\n", sum);
    s += strbuf;

    // 返回结果字符串的 C 风格指针
    return s.c_str();
}

// 定义一个名为 whisper_bench_ggml_mul_mat 的函数，参数为线程数，返回类型为整型
WHISPER_API int whisper_bench_ggml_mul_mat(int n_threads) {
    // 将 whisper_bench_ggml_mul_mat_str 函数返回的字符串输出到标准错误流
    fputs(whisper_bench_ggml_mul_mat_str(n_threads), stderr);
    // 返回整数 0
    return 0;
}

// 定义一个名为 whisper_bench_ggml_mul_mat_str 的函数，参数为线程数，返回类型为常量字符指针
WHISPER_API const char * whisper_bench_ggml_mul_mat_str(int n_threads) {
    // 定义一个静态的字符串对象 s
    static std::string s;
    // 将字符串 s 置空
    s = "";
    // 定义一个字符数组 strbuf，大小为 256
    char strbuf[256];

    // 初始化 ggml 时间
    ggml_time_init();

    // 定义常量整数 n_max 为 128
    const int n_max = 128;

    // 定义一个大小为 64, 128, 256, 512, 1024, 2048, 4096 的大小为 size_t 的向量 sizes
    const std::vector<size_t> sizes = {
        64, 128, 256, 512, 1024, 2048, 4096,
    };

    // 定义常量大小为 N_max 的大小为 sizes 的最后一个元素
    const size_t N_max = sizes.back();

    // 定义一个大小为 3*N_max*N_max*sizeof(float) + 3*ggml_tensor_overhead() + ggml_graph_overhead() 的无符号字符型向量 buf
    std::vector<uint8_t> buf(3llu*N_max*N_max*sizeof(float) + 3*ggml_tensor_overhead() + ggml_graph_overhead());
    // 定义一个大小为 work 的无符号字符型向量
    std::vector<uint8_t> work;

    // 在缓冲区中放入一堆随机数据
    for (size_t i = 0; i < buf.size(); i++) buf[i] = i;

    }

    // 返回字符串 s 的 C 风格字符串
    return s.c_str();
}

// =================================================================================================

// =================================================================================================

//
// Experimental stuff below
//
// Not sure if these should be part of the library at all, because the quality of the results is not
// guaranteed. Might get removed at some point unless a robust algorithm implementation is found
//

// =================================================================================================

//
// token-level timestamps
//

// 将时间戳转换为样本数
static int timestamp_to_sample(int64_t t, int n_samples) {
    return std::max(0, std::min((int) n_samples - 1, (int) ((t*WHISPER_SAMPLE_RATE)/100)));
}

// 将样本数转换为时间戳
static int64_t sample_to_timestamp(int i_sample) {
    return (100ll*i_sample)/WHISPER_SAMPLE_RATE;
}

// 一个高效的代价函数/启发式，对于需要更长时间发音的文本具有较高的值
// 显然，可以改进
static float voice_length(const std::string & text) {
    float res = 0.0f;
    # 遍历文本中的每个字符
    for (char c : text) {
        # 如果字符是空格，则增加0.01到结果中
        if (c == ' ') {
            res += 0.01f;
        } 
        # 如果字符是逗号，则增加2.00到结果中
        else if (c == ',') {
            res += 2.00f;
        } 
        # 如果字符是句号、感叹号或问号，则增加3.00到结果中
        else if (c == '.') {
            res += 3.00f;
        } else if (c == '!') {
            res += 3.00f;
        } else if (c == '?') {
            res += 3.00f;
        } 
        # 如果字符是数字，则增加3.00到结果中
        else if (c >= '0' && c <= '9') {
            res += 3.00f;
        } 
        # 其他情况下，增加1.00到结果中
        else {
            res += 1.00f;
        }
    }

    # 返回最终结果
    return res;
// 计算信号能量的平均值
static std::vector<float> get_signal_energy(const float * signal, int n_samples, int n_samples_per_half_window) {
    // 定义半窗口大小
    const int hw = n_samples_per_half_window;

    // 创建存储结果的向量
    std::vector<float> result(n_samples);

    // 遍历信号样本
    for (int i = 0; i < n_samples; i++) {
        // 初始化总和
        float sum = 0;
        // 遍历半窗口内的样本
        for (int j = -hw; j <= hw; j++) {
            // 确保索引在有效范围内，计算绝对值并累加
            if (i + j >= 0 && i + j < n_samples) {
                sum += fabs(signal[i + j]);
            }
        }
        // 计算平均值并存储到结果向量中
        result[i] = sum/(2*hw + 1);
    }

    // 返回结果向量
    return result;
}

// 计算令牌级别的时间戳
static void whisper_exp_compute_token_level_timestamps(
        struct whisper_context & ctx,
          struct whisper_state & state,
                           int   i_segment,
                         float   thold_pt,
                         float   thold_ptsum) {
    // 获取当前段的信息
    auto & segment = state.result_all[i_segment];
    auto & tokens  = segment.tokens;

    // 获取信号样本数
    const int n_samples = state.energy.size();

    // 如果没有信号数据，则记录错误并返回
    if (n_samples == 0) {
        WHISPER_LOG_ERROR("%s: no signal data available\n", __func__);
        return;
    }

    // 获取段的起始和结束时间戳
    const int64_t t0 = segment.t0;
    const int64_t t1 = segment.t1;

    // 获取令牌数量
    const int n = tokens.size();

    // 如果令牌数量为0，则直接返回
    if (n == 0) {
        return;
    }

    // 如果令牌数量为1，则直接设置起始和结束时间戳并返回
    if (n == 1) {
        tokens[0].t0 = t0;
        tokens[0].t1 = t1;

        return;
    }

    // 获取状态中的时间戳和上一个令牌的信息
    auto & t_beg    = state.t_beg;
    auto & t_last   = state.t_last;
    auto & tid_last = state.tid_last;
}
    // 遍历 tokens 数组，处理每个 token
    for (int j = 0; j < n; ++j) {
        // 获取当前 token 的引用
        auto & token = tokens[j];

        // 如果是第一个 token
        if (j == 0) {
            // 如果当前 token 的 id 等于 whisper_token_beg(&ctx) 返回的 id
            if (token.id == whisper_token_beg(&ctx)) {
                // 设置当前 token 的 t0 和 t1
                tokens[j    ].t0 = t0;
                tokens[j    ].t1 = t0;
                tokens[j + 1].t0 = t0;

                // 更新 t_beg、t_last 和 tid_last
                t_beg    = t0;
                t_last   = t0;
                tid_last = whisper_token_beg(&ctx);
            } else {
                // 设置当前 token 的 t0 为 t_last
                tokens[j    ].t0 = t_last;
            }
        }

        // 计算当前 token 的时间戳
        const int64_t tt = t_beg + 2*(token.tid - whisper_token_beg(&ctx));

        // 更新当前 token 的属性
        tokens[j].id    = token.id;
        tokens[j].tid   = token.tid;
        tokens[j].p     = token.p;
        tokens[j].pt    = token.pt;
        tokens[j].ptsum = token.ptsum;

        // 计算当前 token 的声音长度
        tokens[j].vlen = voice_length(whisper_token_to_str(&ctx, token.id));

        // 如果当前 token 满足条件
        if (token.pt > thold_pt && token.ptsum > thold_ptsum && token.tid > tid_last && tt <= t1) {
            // 如果不是第一个 token，更新前一个 token 的 t1
            if (j > 0) {
                tokens[j - 1].t1 = tt;
            }
            // 更新当前 token 的 t0 和 tid_last
            tokens[j].t0 = tt;
            tid_last = token.tid;
        }
    }

    // 更新倒数第二个 token 的 t1，倒数第一个 token 的 t0 和 t1
    tokens[n - 2].t1 = t1;
    tokens[n - 1].t0 = t1;
    tokens[n - 1].t1 = t1;

    // 更新 t_last

    t_last = t1;

    // 查找具有未知时间戳的 token 区间
    // 根据 token 的声音长度，按比例分割区间并填充时间戳
    {
        // 初始化指针 p0 和 p1
        int p0 = 0;
        int p1 = 0;

        // 循环直到条件不满足
        while (true) {
            // 循环直到 p1 超出范围或者 tokens[p1].t1 小于 0
            while (p1 < n && tokens[p1].t1 < 0) {
                p1++;
            }

            // 如果 p1 超出范围，则将其减一
            if (p1 >= n) {
                p1--;
            }

            // 打印调试信息
            //printf("p0=%d p1=%d t0=%lld t1=%lld\n", p0, p1, tokens[p0].t0, tokens[p1].t1);

            // 如果 p1 大于 p0
            if (p1 > p0) {
                // 计算 p0 到 p1 之间的 vlen 总和
                double psum = 0.0;
                for (int j = p0; j <= p1; j++) {
                    psum += tokens[j].vlen;
                }

                // 打印调试信息
                //printf("analyzing %d - %d, psum = %f\n", p0, p1, psum);

                // 计算时间间隔 dt
                const double dt = tokens[p1].t1 - tokens[p0].t0;

                // 根据声音长度比例分割时间
                for (int j = p0 + 1; j <= p1; j++) {
                    const double ct = tokens[j - 1].t0 + dt*tokens[j - 1].vlen/psum;

                    tokens[j - 1].t1 = ct;
                    tokens[j    ].t0 = ct;
                }
            }

            // 移动指针
            p1++;
            p0 = p1;
            // 如果 p1 超出范围，则跳出循环
            if (p1 >= n) {
                break;
            }
        }
    }

    // 修正时间戳（以防万一）
    for (int j = 0; j < n - 1; j++) {
        if (tokens[j].t1 < 0) {
            tokens[j + 1].t0 = tokens[j].t1;
        }

        if (j > 0) {
            if (tokens[j - 1].t1 > tokens[j].t0) {
                tokens[j].t0 = tokens[j - 1].t1;
                tokens[j].t1 = std::max(tokens[j].t0, tokens[j].t1);
            }
        }
    }

    // VAD
    // 根据语音活动情况扩展或收缩 tokens

    // 固定的 token 扩展（可选）
    //{
    //    const int t_expand = 0;

    //    for (int j = 0; j < n; j++) {
    //        if (j > 0) {
    //            tokens[j].t0 = std::max(0, (int) (tokens[j].t0 - t_expand));
    //        }
    //        if (j < n - 1) {
    //            tokens[j].t1 = tokens[j].t1 + t_expand;
    //        }
    //    }
    //}

    // 调试信息
    //for (int j = 0; j < n; ++j) {
    //    const auto & token = tokens[j];
    // 根据条件判断是否将 token 的字符串表示赋值给 tt，如果条件成立则赋值，否则赋值为"[?]"
    // 使用 printf 函数输出函数名、tt、token 的各项属性值和字符串表示
    // 如果 tokens[j].id 大于等于 ctx 中结束标记的 id，则跳过本次循环
// 设置日志回调函数和用户数据
void whisper_log_set(ggml_log_callback log_callback, void * user_data) {
    // 如果 log_callback 不为空，则使用传入的回调函数，否则使用默认的回调函数
    g_state.log_callback = log_callback ? log_callback : whisper_log_callback_default;
    // 设置日志回调函数的用户数据
    g_state.log_callback_user_data = user_data;
}

// 内部日志记录函数，支持格式化输出
GGML_ATTRIBUTE_FORMAT(2, 3)
static void whisper_log_internal(ggml_log_level level, const char * format, ...) {
    va_list args;
    va_start(args, format);
    // 创建一个大小为 1024 的字符数组作为缓冲区
    char buffer[1024];
    // 使用可变参数列表和格式化字符串将内容写入缓冲区
    int len = vsnprintf(buffer, 1024, format, args);
    // 如果内容长度小于 1024，则调用日志回调函数
    if (len < 1024) {
        g_state.log_callback(level, buffer, g_state.log_callback_user_data);
    } else {
        // 如果内容长度大于等于 1024，则动态分配一个新的字符数组作为缓冲区
        char* buffer2 = new char[len+1];
        // 使用可变参数列表和格式化字符串将内容写入新的缓冲区
        vsnprintf(buffer2, len+1, format, args);
        buffer2[len] = 0;
        // 调用日志回调函数
        g_state.log_callback(level, buffer2, g_state.log_callback_user_data);
        // 释放动态分配的缓冲区
        delete[] buffer2;
    }
    va_end(args);
}

// 默认的日志回调函数，将日志输出到标准错误流
static void whisper_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    // 忽略日志级别和用户数据
    (void) level;
    (void) user_data;
    // 将日志文本输出到标准错误流
    fputs(text, stderr);
    // 刷新标准错误流
    fflush(stderr);
}
```