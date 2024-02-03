# `whisper.cpp\whisper.cpp`

```cpp
// 包含 whisper.h 头文件
#include "whisper.h"

// 如果定义了 WHISPER_USE_COREML，则包含 whisper-encoder.h 头文件
#ifdef WHISPER_USE_COREML
#include "coreml/whisper-encoder.h"
#endif

// 如果定义了 GGML_USE_METAL，则包含 ggml-metal.h 头文件
#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif

// 如果定义了 GGML_USE_CUBLAS，则包含 ggml-cuda.h 头文件
#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif

// 如果定义了 WHISPER_USE_OPENVINO，则包含 whisper-openvino-encoder.h 头文件
#ifdef WHISPER_USE_OPENVINO
#include "openvino/whisper-openvino-encoder.h"
#endif

// 包含 ggml.h 头文件
#include "ggml.h"
// 包含 ggml-alloc.h 头文件
#include "ggml-alloc.h"
// 包含 ggml-backend.h 头文件
#include "ggml-backend.h"

// 包含一些标准库头文件
#include <atomic>
#include <algorithm>
#include <cassert>
#define _USE_MATH_DEFINES
#include <cmath>
#include <cstdio>
#include <cstdarg>
#include <cstring>
#include <fstream>
#include <map>
#include <set>
#include <string>
#include <thread>
#include <vector>
#include <regex>
#include <random>
#include <functional>

// 如果编译器是 MSC，则禁用警告 4244 和 4267
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 如果定义了 GGML_BIG_ENDIAN，则包含 bit 头文件
#if defined(GGML_BIG_ENDIAN)
#include <bit>

// 定义模板函数 byteswap，用于字节交换
template<typename T>
static T byteswap(T value) {
    return std::byteswap(value);
}

// 特化模板函数 byteswap，用于交换 float 类型字节
template<>
float byteswap(float value) {
    return std::bit_cast<float>(byteswap(std::bit_cast<std::uint32_t>(value)));
}

// 定义模板函数 byteswap_tensor_data，用于对 tensor 数据进行字节交换
template<typename T>
static void byteswap_tensor_data(ggml_tensor * tensor) {
    T * datum = reinterpret_cast<T *>(tensor->data);
    for (int i = 0; i < ggml_nelements(tensor); i++) {
        datum[i] = byteswap(datum[i]);
    }
}

// 定义函数 byteswap_tensor，根据 tensor 类型选择对应的字节交换函数
static void byteswap_tensor(ggml_tensor * tensor) {
    switch (tensor->type) {
        case GGML_TYPE_I16: {
            byteswap_tensor_data<int16_t>(tensor);
            break;
        }
        case GGML_TYPE_F16: {
            byteswap_tensor_data<ggml_fp16_t>(tensor);
            break;
        }
        case GGML_TYPE_I32: {
            byteswap_tensor_data<int32_t>(tensor);
            break;
        }
        case GGML_TYPE_F32: {
            byteswap_tensor_data<float>(tensor);
            break;
        }
        default: { // GML_TYPE_I8
            break;
        }
    }
}

// 宏定义 BYTESWAP_VALUE，用于对值进行字节交换
#define BYTESWAP_VALUE(d) d = byteswap(d)
// 宏定义 BYTESWAP_FILTERS，用于字节交换过滤器
#define BYTESWAP_FILTERS(f)            \
    // 使用 do-while 循环，目的是为了能够使用 break 语句来跳出循环
    do {                              \
        // 遍历 f.data 中的每个元素，并对其进行字节交换操作
        for (auto & datum : f.data) { \
            // 调用 byteswap 函数对当前元素进行字节交换
            datum = byteswap(datum);  \
        }                             \
    } while (0)
// 定义宏 BYTESWAP_TENSOR，用于对张量进行字节交换
#define BYTESWAP_TENSOR(t)       \
    do {                         \
        byteswap_tensor(t); \
    } while (0)
// 如果不需要字节交换，则定义空的宏 BYTESWAP_VALUE
#else
#define BYTESWAP_VALUE(d) do {} while (0)
// 如果不需要字节交换，则定义空的宏 BYTESWAP_FILTERS
#define BYTESWAP_FILTERS(f) do {} while (0)
// 如果不需要字节交换，则定义空的宏 BYTESWAP_TENSOR
#define BYTESWAP_TENSOR(t) do {} while (0)
#endif

// 根据不同的编译器定义不同的宏 WHISPER_ATTRIBUTE_FORMAT
#ifdef __GNUC__
#ifdef __MINGW32__
// 如果是 MinGW 编译器，则定义 WHISPER_ATTRIBUTE_FORMAT 为 gnu_printf
#define WHISPER_ATTRIBUTE_FORMAT(...) __attribute__((format(gnu_printf, __VA_ARGS__)))
#else
// 如果不是 MinGW 编译器，则定义 WHISPER_ATTRIBUTE_FORMAT 为 printf
#define WHISPER_ATTRIBUTE_FORMAT(...) __attribute__((format(printf, __VA_ARGS__)))
#endif
#else
// 如果不是 GNU 编译器，则定义空的宏 WHISPER_ATTRIBUTE_FORMAT
#define WHISPER_ATTRIBUTE_FORMAT(...)
#endif

//
// logging
//

// 定义日志输出函数 whisper_log_internal，带有格式化参数
WHISPER_ATTRIBUTE_FORMAT(2, 3)
static void whisper_log_internal        (ggml_log_level level, const char * format, ...);
// 定义默认的日志回调函数 whisper_log_callback_default
static void whisper_log_callback_default(ggml_log_level level, const char * text, void * user_data);

// 定义宏 WHISPER_LOG_ERROR，用于输出错误日志
#define WHISPER_LOG_ERROR(...) whisper_log_internal(GGML_LOG_LEVEL_ERROR, __VA_ARGS__)
// 定义宏 WHISPER_LOG_WARN，用于输出警告日志
#define WHISPER_LOG_WARN(...)  whisper_log_internal(GGML_LOG_LEVEL_WARN , __VA_ARGS__)
// 定义宏 WHISPER_LOG_INFO，用于输出信息日志
#define WHISPER_LOG_INFO(...)  whisper_log_internal(GGML_LOG_LEVEL_INFO , __VA_ARGS__)

// 如果定义了 WHISPER_DEBUG，则定义宏 WHISPER_LOG_DEBUG 用于输出调试日志
// 否则定义为空的宏 WHISPER_LOG_DEBUG
#if defined(WHISPER_DEBUG)
#define WHISPER_LOG_DEBUG(...) whisper_log_internal(GGML_LOG_LEVEL_DEBUG, __VA_ARGS__)
#else
#define WHISPER_LOG_DEBUG(...)
#endif

// 定义宏 WHISPER_ASSERT，用于断言检查，如果条件不成立则输出错误日志并终止程序
#define WHISPER_ASSERT(x) \
    do { \
        if (!(x)) { \
            WHISPER_LOG_ERROR("WHISPER_ASSERT: %s:%d: %s\n", __FILE__, __LINE__, #x); \
            abort(); \
        } \
    } while (0)

// 定义最大解码器数量和最大节点数量的常量
//#define WHISPER_USE_FLASH_ATTN
//#define WHISPER_USE_FLASH_FF
#define WHISPER_MAX_DECODERS 8
#define WHISPER_MAX_NODES 4096

//
// ggml helpers
//

// 定义 ggml_graph_compute_helper 函数，用于计算图形计划
static bool ggml_graph_compute_helper(
          struct ggml_cgraph * graph,
        std::vector<uint8_t> & buf,
                         int   n_threads,
      whisper_abort_callback   abort_callback,
                        void * abort_callback_data) {
    // 根据图形和线程数计算计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);
    # 设置计划的中止回调函数
    plan.abort_callback = abort_callback;
    # 设置计划的中止回调数据
    plan.abort_callback_data = abort_callback_data;

    # 如果计划的工作大小大于0
    if (plan.work_size > 0) {
        # 调整缓冲区大小为计划的工作大小
        buf.resize(plan.work_size);
        # 将缓冲区的数据指针赋给计划的工作数据
        plan.work_data = buf.data();
    }

    # 调用 ggml_graph_compute 函数执行计划的图计算
    return ggml_graph_compute(graph, &plan);
// 静态函数，用于辅助计算图形，根据后端和线程数进行计算
static bool ggml_graph_compute_helper(
       struct ggml_backend * backend,
        struct ggml_cgraph * graph,
                       int   n_threads) {
    // 如果后端是 CPU，则设置线程数
    if (ggml_backend_is_cpu(backend)) {
        ggml_backend_cpu_set_n_threads(backend, n_threads);
    }
    // 如果启用 Metal 后端，则设置计算块数
#ifdef GGML_USE_METAL
    if (ggml_backend_is_metal(backend)) {
        ggml_backend_metal_set_n_cb(backend, n_threads);
    }
#endif
    // 调用后端计算图形的函数
    return ggml_backend_graph_compute(backend, graph);
}

// 为不具有维度 0 可被 "pad" 整除的张量提供更快的矩阵乘法
// 思路是将原始矩阵乘法表示为两个矩阵乘法的和
// X_0 @ Y_0 + X_1 @ Y_1
// 这里 X_0 和 Y_0 是 X 和 Y 的视图，其维度 0 可被 "pad" 整除
// X_1 和 Y_1 是剩余的视图，X_1 和 Y_1 最终是可以用更通用的内核处理的小矩阵
static struct ggml_tensor * ggml_mul_mat_pad(struct ggml_context * ctx, struct ggml_tensor * x, struct ggml_tensor * y, int pad = 32) {
    // 仅当维度 0 至少比填充大 8 倍时才使用填充
    // 否则我们不会从优化中获得太多好处
    const int n_pad_req = 8;

    // 如果 x 的维度 0 可以被 "pad" 整除，或者 x 的维度 0 除以 "pad" 小于 n_pad_req
    if (x->ne[0] % pad == 0 || x->ne[0] / pad < n_pad_req) {
        // 直接进行矩阵乘法
        return ggml_mul_mat(ctx, x, y);
    }

    // 创建 X_0 和 X_1 视图
    struct ggml_tensor * x_0 = ggml_view_3d(ctx, x, (x->ne[0]/pad)*pad, x->ne[1], x->ne[2], x->nb[1], x->nb[2], 0);
    struct ggml_tensor * x_1 = ggml_view_3d(ctx, x,  x->ne[0]%pad,      x->ne[1], x->ne[2], x->nb[1], x->nb[2], x_0->ne[0]*x_0->nb[0]);

    // 创建 Y_0 和 Y_1 视图
    struct ggml_tensor * y_0 = ggml_view_3d(ctx, y, (y->ne[0]/pad)*pad, y->ne[1], y->ne[2], y->nb[1], y->nb[2], 0);
    struct ggml_tensor * y_1 = ggml_view_3d(ctx, y,  y->ne[0]%pad,      y->ne[1], y->ne[2], y->nb[1], y->nb[2], y_0->ne[0]*y_0->nb[0]);

    // 返回两个矩阵乘法的和
    return ggml_add(ctx,
            ggml_mul_mat(ctx, x_0, y_0),
            ggml_mul_mat(ctx, x_1, y_1));
}
// 如果其他平台可以从这个优化中受益，请检查
// CUDA 目前存在问题 - 似乎 ggml_mul_mat 未正确处理视图
#if defined(GGML_USE_METAL)
#define ggml_mul_mat ggml_mul_mat_pad
#endif

// 可用的 Whisper 模型
enum e_model {
    MODEL_UNKNOWN,
    MODEL_TINY,
    MODEL_BASE,
    MODEL_SMALL,
    MODEL_MEDIUM,
    MODEL_LARGE,
};

// 模型名称映射
static const std::map<e_model, std::string> g_model_name = {
    { MODEL_UNKNOWN,  "unknown"  },
    { MODEL_TINY,     "tiny"     },
    { MODEL_BASE,     "base"     },
    { MODEL_SMALL,    "small"    },
    { MODEL_MEDIUM,   "medium"   },
    { MODEL_LARGE,    "large"    },
};

// 语言映射
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
    { "no",  { 29,  "norwegian",      } },  // 包含语言代码和对应的编号、语言名称的元组
    { "th",  { 30,  "thai",           } },  // 包含语言代码和对应的编号、语言名称的元组
    { "ur",  { 31,  "urdu",           } },  // 包含语言代码和对应的编号、语言名称的元组
    { "hr",  { 32,  "croatian",       } },  // 包含语言代码和对应的编号、语言名称的元组
    { "bg",  { 33,  "bulgarian",      } },  // 包含语言代码和对应的编号、语言名称的元组
    { "lt",  { 34,  "lithuanian",     } },  // 包含语言代码和对应的编号、语言名称的元组
    { "la",  { 35,  "latin",          } },  // 包含语言代码和对应的编号、语言名称的元组
    { "mi",  { 36,  "maori",          } },  // 包含语言代码和对应的编号、语言名称的元组
    { "ml",  { 37,  "malayalam",      } },  // 包含语言代码和对应的编号、语言名称的元组
    { "cy",  { 38,  "welsh",          } },  // 包含语言代码和对应的编号、语言名称的元组
    { "sk",  { 39,  "slovak",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "te",  { 40,  "telugu",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "fa",  { 41,  "persian",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "lv",  { 42,  "latvian",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "bn",  { 43,  "bengali",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "sr",  { 44,  "serbian",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "az",  { 45,  "azerbaijani",    } },  // 包含语言代码和对应的编号、语言名称的元组
    { "sl",  { 46,  "slovenian",      } },  // 包含语言代码和对应的编号、语言名称的元组
    { "kn",  { 47,  "kannada",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "et",  { 48,  "estonian",       } },  // 包含语言代码和对应的编号、语言名称的元组
    { "mk",  { 49,  "macedonian",     } },  // 包含语言代码和对应的编号、语言名称的元组
    { "br",  { 50,  "breton",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "eu",  { 51,  "basque",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "is",  { 52,  "icelandic",      } },  // 包含语言代码和对应的编号、语言名称的元组
    { "hy",  { 53,  "armenian",       } },  // 包含语言代码和对应的编号、语言名称的元组
    { "ne",  { 54,  "nepali",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "mn",  { 55,  "mongolian",      } },  // 包含语言代码和对应的编号、语言名称的元组
    { "bs",  { 56,  "bosnian",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "kk",  { 57,  "kazakh",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "sq",  { 58,  "albanian",       } },  // 包含语言代码和对应的编号、语言名称的元组
    { "sw",  { 59,  "swahili",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "gl",  { 60,  "galician",       } },  // 包含语言代码和对应的编号、语言名称的元组
    { "mr",  { 61,  "marathi",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "pa",  { 62,  "punjabi",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "si",  { 63,  "sinhala",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "km",  { 64,  "khmer",          } },  // 包含语言代码和对应的编号、语言名称的元组
    { "sn",  { 65,  "shona",          } },  // 包含语言代码和对应的编号、语言名称的元组
    { "yo",  { 66,  "yoruba",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "so",  { 67,  "somali",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "af",  { 68,  "afrikaans",      } },  // 包含语言代码和对应的编号、语言名称的元组
    { "oc",  { 69,  "occitan",        } },  // 包含语言代码和对应的编号、语言名称的元组
    { "ka",  { 70,  "georgian",       } },  // 包含语言代码和对应的编号、语言名称的元组
    { "be",  { 71,  "belarusian",     } },  // 包含语言代码和对应的编号、语言名称的元组
    { "tg",  { 72,  "tajik",          } },  // 包含语言代码和对应的编号、语言名称的元组
    { "sd",  { 73,  "sindhi",         } },  // 包含语言代码和对应的编号、语言名称的元组
    { "gu",  { 74,  "gujarati",       } },  // 包含语言代码和对应的编号、语言名称的元组
    { "am",  { 75,  "amharic",        } },  // 键值对，键为"am"，值为包含编号75和字符串"amharic"的元组
    { "yi",  { 76,  "yiddish",        } },  // 键值对，键为"yi"，值为包含编号76和字符串"yiddish"的元组
    { "lo",  { 77,  "lao",            } },  // 键值对，键为"lo"，值为包含编号77和字符串"lao"的元组
    { "uz",  { 78,  "uzbek",          } },  // 键值对，键为"uz"，值为包含编号78和字符串"uzbek"的元组
    { "fo",  { 79,  "faroese",        } },  // 键值对，键为"fo"，值为包含编号79和字符串"faroese"的元组
    { "ht",  { 80,  "haitian creole", } },  // 键值对，键为"ht"，值为包含编号80和字符串"haitian creole"的元组
    { "ps",  { 81,  "pashto",         } },  // 键值对，键为"ps"，值为包含编号81和字符串"pashto"的元组
    { "tk",  { 82,  "turkmen",        } },  // 键值对，键为"tk"，值为包含编号82和字符串"turkmen"的元组
    { "nn",  { 83,  "nynorsk",        } },  // 键值对，键为"nn"，值为包含编号83和字符串"nynorsk"的元组
    { "mt",  { 84,  "maltese",        } },  // 键值对，键为"mt"，值为包含编号84和字符串"maltese"的元组
    { "sa",  { 85,  "sanskrit",       } },  // 键值对，键为"sa"，值为包含编号85和字符串"sanskrit"的元组
    { "lb",  { 86,  "luxembourgish",  } },  // 键值对，键为"lb"，值为包含编号86和字符串"luxembourgish"的元组
    { "my",  { 87,  "myanmar",        } },  // 键值对，键为"my"，值为包含编号87和字符串"myanmar"的元组
    { "bo",  { 88,  "tibetan",        } },  // 键值对，键为"bo"，值为包含编号88和字符串"tibetan"的元组
    { "tl",  { 89,  "tagalog",        } },  // 键值对，键为"tl"，值为包含编号89和字符串"tagalog"的元组
    { "mg",  { 90,  "malagasy",       } },  // 键值对，键为"mg"，值为包含编号90和字符串"malagasy"的元组
    { "as",  { 91,  "assamese",       } },  // 键值对，键为"as"，值为包含编号91和字符串"assamese"的元组
    { "tt",  { 92,  "tatar",          } },  // 键值对，键为"tt"，值为包含编号92和字符串"tatar"的元组
    { "haw", { 93,  "hawaiian",       } },  // 键值对，键为"haw"，值为包含编号93和字符串"hawaiian"的元组
    { "ln",  { 94,  "lingala",        } },  // 键值对，键为"ln"，值为包含编号94和字符串"lingala"的元组
    { "ha",  { 95,  "hausa",          } },  // 键值对，键为"ha"，值为包含编号95和字符串"hausa"的元组
    { "ba",  { 96,  "bashkir",        } },  // 键值对，键为"ba"，值为包含编号96和字符串"bashkir"的元组
    { "jw",  { 97,  "javanese",       } },  // 键值对，键为"jw"，值为包含编号97和字符串"javanese"的元组
    { "su",  { 98,  "sundanese",      } },  // 键值对，键为"su"，值为包含编号98和字符串"sundanese"的元组
    { "yue", { 99,  "cantonese",      } },  // 键值对，键为"yue"，值为包含编号99和字符串"cantonese"的元组
};

// 定义结构体 whisper_mel，包含 n_len、n_len_org、n_mel 和 data 成员变量
struct whisper_mel {
    int n_len;
    int n_len_org;
    int n_mel;

    std::vector<float> data;
};

// 定义结构体 whisper_filters，包含 n_mel、n_fft 和 data 成员变量
struct whisper_filters {
    int32_t n_mel;
    int32_t n_fft;

    std::vector<float> data;
};

// 定义结构体 whisper_vocab，包含 n_vocab、token_to_id、id_to_token 和各种特殊 token 的成员变量
struct whisper_vocab {
    using id    = int32_t;
    using token = std::string;

    int n_vocab = 51864;

    std::map<token, id> token_to_id;
    std::map<id, token> id_to_token;

    // 定义各种特殊 token 的 id
    id token_eot        = 50256;
    id token_sot        = 50257;
    id token_translate  = 50357;
    id token_transcribe = 50358;
    id token_solm       = 50359;
    id token_prev       = 50360;
    id token_nosp       = 50361;
    id token_not        = 50362;
    id token_beg        = 50363;

    // 判断是否为多语言模型
    bool is_multilingual() const {
        return n_vocab >= 51865;
    }

    // 返回语言数量
    int num_languages() const {
        return n_vocab - 51765 - (is_multilingual() ? 1 : 0);
    }
};

// 定义结构体 whisper_segment，包含 t0、t1、text、tokens 和 speaker_turn_next 成员变量
struct whisper_segment {
    int64_t t0;
    int64_t t1;

    std::string text;

    std::vector<whisper_token_data> tokens;

    bool speaker_turn_next;
};

// 定义结构体 whisper_batch，包含 n_tokens、token、pos、n_seq_id、seq_id、logits 成员变量
struct whisper_batch {
    int32_t n_tokens;

    whisper_token  *  token;
    whisper_pos    *  pos;
    int32_t        *  n_seq_id;
    whisper_seq_id ** seq_id;   // null terminated
    int8_t         *  logits;
};

// 初始化 whisper_batch 结构体
static struct whisper_batch whisper_batch_init(int32_t n_tokens, int32_t n_seq_max) {
    whisper_batch batch = { 0, nullptr, nullptr, nullptr, nullptr, nullptr, };

    // 分配内存给 batch 的成员变量
    batch.token    = (whisper_token *  ) malloc(sizeof(whisper_token)    * (n_tokens));
    batch.pos      = (whisper_pos *)     malloc(sizeof(whisper_pos)      * (n_tokens));
    batch.n_seq_id = (int32_t *)         malloc(sizeof(int32_t)          * (n_tokens));
    # 为 batch 结构体中的 seq_id 成员分配内存空间，大小为 n_tokens + 1 个 whisper_seq_id 指针
    batch.seq_id   = (whisper_seq_id **) malloc(sizeof(whisper_seq_id *) * (n_tokens + 1));
    # 遍历 n_tokens 次，为每个 seq_id[i] 分配内存空间，大小为 n_seq_max 个 whisper_seq_id
    for (int i = 0; i < n_tokens; ++i) {
        batch.seq_id[i] = (whisper_seq_id *) malloc(sizeof(whisper_seq_id)   * n_seq_max);
    }
    # 为 batch 结构体中的 seq_id 的最后一个元素赋值为 nullptr
    batch.seq_id[n_tokens] = nullptr;
    # 为 batch 结构体中的 logits 成员分配内存空间，大小为 n_tokens 个 int8_t
    batch.logits   = (int8_t *)          malloc(sizeof(int8_t)           * n_tokens);

    # 返回分配好内存空间的 batch 结构体
    return batch;
// 释放 whisper_batch 结构体中动态分配的内存
static void whisper_batch_free(struct whisper_batch batch) {
    // 如果 batch.token 不为空，则释放其内存
    if (batch.token)    free(batch.token);
    // 如果 batch.pos 不为空，则释放其内存
    if (batch.pos)      free(batch.pos);
    // 如果 batch.n_seq_id 不为空，则释放其内存
    if (batch.n_seq_id) free(batch.n_seq_id);
    // 如果 batch.seq_id 不为空，则释放其内存
    if (batch.seq_id) {
        // 遍历 batch.seq_id 数组，释放每个元素的内存
        for (int i = 0; batch.seq_id[i]; ++i) {
            free(batch.seq_id[i]);
        }
        // 释放 batch.seq_id 数组的内存
        free(batch.seq_id);
    }
    // 如果 batch.logits 不为空，则释放其内存
    if (batch.logits)   free(batch.logits);
}

// 准备 whisper_batch 结构体的数据，用于旧版本的处理
static void whisper_batch_prep_legacy(whisper_batch & batch, const whisper_token * tokens, int n_tokens, int n_past, int seq_id) {
    // 设置 batch.n_tokens 为 n_tokens
    batch.n_tokens = n_tokens;
    // 遍历 n_tokens 次
    for (int i = 0; i < n_tokens; ++i) {
        // 如果 tokens 不为空，则将 tokens[i] 赋值给 batch.token[i]
        if (tokens) {
            batch.token[i] = tokens[i];
        }
        // 设置 batch.pos[i] 为 n_past + i
        batch.pos     [i]    = n_past + i;
        // 设置 batch.n_seq_id[i] 为 1
        batch.n_seq_id[i]    = 1;
        // 设置 batch.seq_id[i][0] 为 seq_id
        batch.seq_id  [i][0] = seq_id;
        // 设置 batch.logits[i] 为 0
        batch.logits  [i]    = 0;
    }
    // 设置 batch.logits[n_tokens - 1] 为 1
    batch.logits[n_tokens - 1] = 1;
}

// 使用自定义的 pair 结构体替换 std::pair（原因：std::pair 速度较慢）
template<typename A, typename B>
struct whisper_pair {
    A first;
    B second;

    // 定义一个接受两个参数的构造函数
    whisper_pair(const A& a, const B& b) : first(a), second(b) {}
    // 定义一个不接受参数的构造函数
    whisper_pair() : first(A()), second(B()) {}
};

// whisper 使用的 ggml_allocr 的包装器
struct whisper_allocr {
    ggml_allocr * alloc = nullptr;

    std::vector<uint8_t> meta;

    ggml_backend_buffer_t buffer;
};

// 获取 whisper_allocr 结构体的大小
static size_t whisper_allocr_size(struct whisper_allocr & allocr) {
    return allocr.meta.size() + ggml_allocr_max_size(allocr.alloc);
}

// 测量图的内存使用情况并准备 allocr 的内部数据缓冲区
static void whisper_allocr_graph_init(struct whisper_allocr & allocr, ggml_backend_t backend, std::function<struct ggml_cgraph *()> && get_graph) {
    auto & alloc = allocr.alloc;
    auto & meta  = allocr.meta;

    // 从 backend 创建一个新的 allocr 对象
    alloc = ggml_allocr_new_measure_from_backend(backend);
}
    # 调整 meta 的大小，以容纳 ggml_tensor_overhead()*WHISPER_MAX_NODES + ggml_graph_overhead() 的空间
    meta.resize(ggml_tensor_overhead()*WHISPER_MAX_NODES + ggml_graph_overhead());

    # 使用 allocr_alloc_graph 函数为当前图形分配内存
    ggml_allocr_alloc_graph(alloc, get_graph());
// 重新分配内存图的内存，根据后端类型
static void whisper_allocr_graph_realloc(struct whisper_allocr & allocr, ggml_backend_t backend) {
    // 如果分配器为空，则返回
    if (allocr.alloc == nullptr) {
        // 如果我们使用像 CoreML 或 OpenVINO 这样的外部编码器，这里可能为空
        return;
    }

    // 获取分配器和缓冲区的引用
    auto & alloc  = allocr.alloc;
    auto & buffer = allocr.buffer;

    // 获取分配器的最大大小
    size_t size = ggml_allocr_max_size(alloc);

    // 释放分配器
    ggml_allocr_free(alloc);

    // 为缓冲区分配新的内存
    buffer = ggml_backend_alloc_buffer(backend, size);
    // 根据新的缓冲区创建新的分配器
    alloc = ggml_allocr_new_from_buffer(buffer);
}

// 释放分配器
static void whisper_allocr_free(struct whisper_allocr & allocr) {
    // 如果分配器存在
    if (allocr.alloc) {
        // 释放分配器
        ggml_allocr_free(allocr.alloc);
        // 释放缓冲区
        ggml_backend_buffer_free(allocr.buffer);
        // 将分配器指针设为 nullptr
        allocr.alloc = nullptr;
    }
}

// 中等大小的超参数
// hparams: {
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
// 默认的超参数（Whisper tiny）
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
}
    // 定义指向 MLP Layer Normalization 权重和偏置的指针
    struct ggml_tensor * mlp_ln_w;
    struct ggml_tensor * mlp_ln_b;

    // 定义指向第一个 MLP 层权重和偏置的指针
    struct ggml_tensor * mlp_0_w;
    struct ggml_tensor * mlp_0_b;

    // 定义指向第二个 MLP 层权重和偏置的指针
    struct ggml_tensor * mlp_1_w;
    struct ggml_tensor * mlp_1_b;
// token decoding layer 结构体，用于存储解码器的各个层的参数
struct whisper_layer_decoder {
    // decoder.blocks.*.attn_ln
    // 注意力层的 Layer Normalization 参数
    struct ggml_tensor * attn_ln_0_w;
    struct ggml_tensor * attn_ln_0_b;

    // decoder.blocks.*.attn.out
    // 注意力层的输出参数
    struct ggml_tensor * attn_ln_1_w;
    struct ggml_tensor * attn_ln_1_b;

    // decoder.blocks.*.attn.query
    // 注意力层的查询参数
    struct ggml_tensor * attn_q_w;
    struct ggml_tensor * attn_q_b;

    // decoder.blocks.*.attn.key
    // 注意力层的键参数
    struct ggml_tensor * attn_k_w;

    // decoder.blocks.*.attn.value
    // 注意力层的值参数
    struct ggml_tensor * attn_v_w;
    struct ggml_tensor * attn_v_b;

    // decoder.blocks.*.cross_attn_ln
    // 交叉注意力层的 Layer Normalization 参数
    struct ggml_tensor * cross_attn_ln_0_w;
    struct ggml_tensor * cross_attn_ln_0_b;

    // decoder.blocks.*.cross_attn.out
    // 交叉注意力层的输出参数
    struct ggml_tensor * cross_attn_ln_1_w;
    struct ggml_tensor * cross_attn_ln_1_b;

    // decoder.blocks.*.cross_attn.query
    // 交叉注意力层的查询参数
    struct ggml_tensor * cross_attn_q_w;
    struct ggml_tensor * cross_attn_q_b;

    // decoder.blocks.*.cross_attn.key
    // 交叉注意力层的键参数
    struct ggml_tensor * cross_attn_k_w;

    // decoder.blocks.*.cross_attn.value
    // 交叉注意力层的值参数
    struct ggml_tensor * cross_attn_v_w;
    struct ggml_tensor * cross_attn_v_b;

    // decoder.blocks.*.mlp_ln
    // 多层感知机层的 Layer Normalization 参数
    struct ggml_tensor * mlp_ln_w;
    struct ggml_tensor * mlp_ln_b;

    // decoder.blocks.*.mlp.0
    // 多层感知机层的第一个线性变换参数
    struct ggml_tensor * mlp_0_w;
    struct ggml_tensor * mlp_0_b;

    // decoder.blocks.*.mlp.2
    // 多层感知机层的第二个线性变换参数
    struct ggml_tensor * mlp_1_w;
    struct ggml_tensor * mlp_1_b;
};

// whisper_kv_cell 结构体，用于存储键值对的信息
struct whisper_kv_cell {
    // 键值对的位置信息
    whisper_pos pos = -1;

    // 键值对的序列 ID 集合
    std::set<whisper_seq_id> seq_id;

    // 检查是否存在特定序列 ID 的键值对
    bool has_seq_id(const whisper_seq_id & id) const {
        return seq_id.find(id) != seq_id.end();
    }
};

// whisper_kv_cache 结构体，用于存储键值缓存的信息
struct whisper_kv_cache {
    // 缓存头部位置
    uint32_t head = 0;
    // 缓存大小
    uint32_t size = 0;

    // 在每次构建图之前计算的值
    uint32_t n = 0;

    // 键值对单元的向量
    std::vector<whisper_kv_cell> cells;

    // 键和值的张量
    struct ggml_tensor * k;
    struct ggml_tensor * v;

    // 上下文
    struct ggml_context * ctx;

    // 后端缓冲区
    ggml_backend_buffer_t buffer;
};

// whisper_model 结构体
struct whisper_model {
    // 初始化模型类型为未知
    e_model type = MODEL_UNKNOWN;

    // 初始化超参数和滤波器
    whisper_hparams hparams;
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

    // 初始化解码器的 token 嵌入
    struct ggml_tensor * d_te;

    // 初始化解码器的 Layer Normalization 参数
    struct ggml_tensor * d_ln_w;
    struct ggml_tensor * d_ln_b;

    // 初始化编码器的层列表
    std::vector<whisper_layer_encoder> layers_encoder;
    
    // 初始化解码器的层列表
    std::vector<whisper_layer_decoder> layers_decoder;

    // 创建包含模型张量元信息的 ggml 上下文
    struct ggml_context * ctx;

    // 模型后端数据是只读的，可以在处理器之间共享
    std::vector<struct ggml_backend_buffer *> buffers;

    // 张量数量
    int n_loaded;
    
    // 存储张量名称到张量指针的映射
    std::map<std::string, struct ggml_tensor *> tensors;
// 结构体定义，用于存储部分生成的 UTF-8 序列
struct whisper_partial_utf8 {
    uint32_t value;    // 当前位值（未移位）
    int      n_remain; // 剩余字节数；-1 表示无效序列
};

// 结构体定义，用于存储语法规则
struct whisper_grammar {
    /*const*/ std::vector<std::vector<whisper_grammar_element>> rules; // 存储语法规则
    std::vector<std::vector<const whisper_grammar_element *>>   stacks; // 存储语法规则栈

    // 用于存储从接受的标记中部分生成的 UTF-8 序列的缓冲区
    whisper_partial_utf8 partial_utf8;
};

// 结构体定义，用于存储语法规则候选项
struct whisper_grammar_candidate {
    whisper_token          id;             // 标记 ID
    const uint32_t       * code_points;    // 代码点
    whisper_partial_utf8   partial_utf8;   // 部分生成的 UTF-8 序列
};

// 结构体定义，用于存储标记序列
struct whisper_sequence {
    std::vector<whisper_token_data> tokens; // 标记数组

    // 当前迭代中累积的转录（用于截断标记数组）
    int result_len;

    double sum_logprobs_all; // 所有标记的对数概率之和
    double sum_logprobs;     // 标记的对数概率之和（前 result_len 个标记）
    double avg_logprobs;     // 标记的平均对数概率
    double entropy;          // 标记的熵
    double score;            // 可能性排名分数
};

// 标签：WHISPER_DECODER_INIT
// 结构体定义，用于存储解码器状态
struct whisper_decoder {
    // 当前生成的标记序列
    whisper_sequence sequence;

    // 生成的标记序列的语法解析状态
    whisper_grammar  grammar;

    int i_batch;    // 当前批次中标记的索引
    int seek_delta; // 基于解码的时间戳标记找到的窗口偏移量

    bool failed;    // 当前段落是否解码失败？
    bool completed; // 解码器是否完成当前段落？
    bool has_ts;    // 是否已经为当前段落采样了非起始时间戳标记？

    // 上一次 whisper_decode 后的新标记概率、logits 和对数概率（一维数组：[n_vocab]）
    std::vector<float> probs;
    std::vector<float> logits;
    std::vector<float> logprobs;
}
    // 用于避免内存分配的工作容器
    std::vector<whisper_pair<double, whisper_vocab::id>> logits_id;

    // 用于在 t > 0.0 时进行采样的可变的伪随机数生成器
    mutable std::mt19937 rng;
// 结构体定义，用于存储 Whisper 模型的状态信息
struct whisper_state {
    int64_t t_sample_us = 0; // 采样时间（微秒）
    int64_t t_encode_us = 0; // 编码时间（微秒）
    int64_t t_decode_us = 0; // 解码时间（微秒）
    int64_t t_batchd_us = 0; // 批处理时间（微秒）
    int64_t t_prompt_us = 0; // 提示时间（微秒）
    int64_t t_mel_us = 0; // MEL 时间（微秒）

    int32_t n_sample = 0; // 采样的标记数
    int32_t n_encode = 0; // 编码器调用次数
    int32_t n_decode = 0; // 解码器调用次数，n_tokens == 1（文本生成）
    int32_t n_batchd = 0; // 解码器调用次数，n_tokens < 16（批处理解码）
    int32_t n_prompt = 0; // 解码器调用次数，n_tokens > 1（提示编码）
    int32_t n_fail_p = 0; // logprob 阈值失败次数
    int32_t n_fail_h = 0; // 熵阈值失败次数

    // 所有解码器共享的统一自注意力 KV 缓存
    whisper_kv_cache kv_self;

    // 解码器之间共享的交叉注意力 KV 缓存
    whisper_kv_cache kv_cross;

    whisper_mel mel; // MEL

    whisper_batch batch; // 批处理

    whisper_decoder decoders[WHISPER_MAX_DECODERS]; // 解码器数组

    ggml_backend_t backend = nullptr; // 后端

    // ggml-alloc:
    // - 将中间张量的元信息存储在 `meta` 缓冲区中
    // - 将实际张量数据存储在 `data` 缓冲区中
    whisper_allocr alloc_conv;
    whisper_allocr alloc_encode;
    whisper_allocr alloc_cross;
    whisper_allocr alloc_decode;

    // 编码器的结果
    struct ggml_tensor * embd_conv = nullptr;
    struct ggml_tensor * embd_enc  = nullptr;

    // GPU 卸载的辅助工具
    std::vector<float> inp_mel;
    std::vector<float> inp_mask;

    // 解码输出（二维数组：[n_tokens][n_vocab]）
    std::vector<float> logits;

    std::vector<whisper_segment> result_all; // 所有结果
    std::vector<whisper_token>   prompt_past; // 过去的提示

    int lang_id = 0; // 默认为英语

    std::string path_model; // 由 whisper_init_from_file_with_params() 填充

#ifdef WHISPER_USE_COREML
    whisper_coreml_context * ctx_coreml = nullptr; // CoreML 上下文
#endif

#ifdef WHISPER_USE_OPENVINO
    // 创建一个指向 OpenVINO 上下文的指针，并初始化为 nullptr
    whisper_openvino_context * ctx_openvino = nullptr;
#endif

    // [EXPERIMENTAL] token-level timestamps data
    int64_t t_beg  = 0; // 初始化 t_beg 变量为 0
    int64_t t_last = 0; // 初始化 t_last 变量为 0

    whisper_token tid_last; // 声明一个 whisper_token 类型的变量 tid_last

    std::vector<float> energy; // 声明一个存储 PCM 信号能量的向量 energy

    // [EXPERIMENTAL] speed-up techniques
    int32_t exp_n_audio_ctx = 0; // 初始化 exp_n_audio_ctx 变量为 0，用于加速技术

};

struct whisper_context {
    int64_t t_load_us  = 0; // 初始化 t_load_us 变量为 0
    int64_t t_start_us = 0; // 初始化 t_start_us 变量为 0

    ggml_type wtype = ggml_type::GGML_TYPE_F16; // 初始化 wtype 变量为 GGML_TYPE_F16，表示权重类型为 FP16
    ggml_type itype = ggml_type::GGML_TYPE_F16; // 初始化 itype 变量为 GGML_TYPE_F16，表示中间类型为 FP16

    whisper_context_params params; // 声明一个 whisper_context_params 类型的变量 params

    whisper_model model; // 声明一个 whisper_model 类型的变量 model
    whisper_vocab vocab; // 声明一个 whisper_vocab 类型的变量 vocab

    whisper_state * state = nullptr; // 初始化 state 变量为空指针

    ggml_backend_t backend = nullptr; // 初始化 backend 变量为空指针

    std::string path_model; // 声明一个存储模型路径的字符串变量 path_model

};

struct whisper_global {
    // We save the log callback globally
    ggml_log_callback log_callback = whisper_log_callback_default; // 初始化 log_callback 变量为 whisper_log_callback_default
    void * log_callback_user_data = nullptr; // 初始化 log_callback_user_data 变量为空指针
};

static whisper_global g_state; // 声明一个 whisper_global 类型的变量 g_state

template<typename T>
static void read_safe(whisper_model_loader * loader, T & dest) {
    loader->read(loader->context, &dest, sizeof(T)); // 从 loader 中读取数据到 dest 变量中
    BYTESWAP_VALUE(dest); // 对 dest 变量进行字节交换
}

static bool kv_cache_init(
        const struct whisper_hparams & hparams,
             struct whisper_kv_cache & cache,
                      ggml_backend_t   backend,
                           ggml_type   wtype,
                                 int   n_ctx) {
    const int64_t n_text_state = hparams.n_text_state; // 初始化 n_text_state 变量为 hparams 中的 n_text_state
    const int64_t n_text_layer = hparams.n_text_layer; // 初始化 n_text_layer 变量为 hparams 中的 n_text_layer

    const int64_t n_mem      = n_text_layer*n_ctx; // 计算 n_mem 变量的值
    const int64_t n_elements = n_text_state*n_mem; // 计算 n_elements 变量的值

    struct ggml_init_params params = {
        /*.mem_size   =*/ 2*ggml_tensor_overhead(), // 初始化 params 中的 mem_size 变量
        /*.mem_buffer =*/ nullptr, // 初始化 params 中的 mem_buffer 变量为空指针
        /*.no_alloc   =*/ true, // 初始化 params 中的 no_alloc 变量为 true
    };

    cache.head = 0; // 初始化 cache 中的 head 变量为 0
    cache.size = n_ctx; // 初始化 cache 中的 size 变量为 n_ctx

    cache.cells.clear(); // 清空 cache 中的 cells 向量
    cache.cells.resize(n_ctx); // 调整 cache 中的 cells 向量大小为 n_ctx

    cache.ctx = ggml_init(params); // 使用 ggml_init 函数初始化 cache 中的 ctx 变量
    // 如果缓存上下文为空，则输出错误信息并返回 false
    if (!cache.ctx) {
        WHISPER_LOG_ERROR("%s: failed to allocate memory for kv cache\n", __func__);
        return false;
    }

    // 为缓存的键值对分配内存空间
    cache.k = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);
    cache.v = ggml_new_tensor_1d(cache.ctx, wtype, n_elements);

    // 计算缓存键值对占用的内存字节数
    const size_t mem_bytes = ggml_nbytes(cache.k) + ggml_nbytes(cache.v);

    // 在后端分配缓存的内存空间
    cache.buffer = ggml_backend_alloc_buffer(backend, mem_bytes);

    // 将张量分配到后端缓存中
    {
        ggml_allocr * alloc = ggml_allocr_new_from_buffer(cache.buffer);

        ggml_allocr_alloc(alloc, cache.k);
        ggml_allocr_alloc(alloc, cache.v);

        ggml_allocr_free(alloc);
    }

    // 返回 true 表示成功
    return true;
// 释放缓存中的资源，包括上下文、缓冲区，并将上下文指针置为空
static void kv_cache_free(struct whisper_kv_cache & cache) {
    // 如果上下文存在，则释放上下文
    if (cache.ctx) {
        ggml_free(cache.ctx);
        // 释放缓冲区
        ggml_backend_buffer_free(cache.buffer);
        // 将上下文指针置为空
        cache.ctx = nullptr;
    }
}

// 在缓存中查找插槽，用于存储批处理数据
static bool whisper_kv_cache_find_slot(
           struct whisper_kv_cache & cache,
        const struct whisper_batch & batch) {
    // 获取缓存大小和批处理数据中的令牌数量
    const uint32_t n_ctx    = cache.size;
    const uint32_t n_tokens = batch.n_tokens;

    // 如果批处理数据中的令牌数量大于缓存大小，则记录错误并返回 false
    if (n_tokens > n_ctx) {
        WHISPER_LOG_ERROR("%s: n_tokens=%d > n_ctx=%d\n", __func__, n_tokens, n_ctx);
        return false;
    }

    uint32_t n_tested = 0;

    // 循环查找可用的插槽
    while (true) {
        // 如果当前插槽不足以容纳批处理数据中的令牌数量，则移动到下一个插槽
        if (cache.head + n_tokens > n_ctx) {
            n_tested += n_ctx - cache.head;
            cache.head = 0;
            continue;
        }

        bool found = true;
        // 检查当前插槽是否可用
        for (uint32_t i = 0; i < n_tokens; i++) {
            if (cache.cells[cache.head + i].pos >= 0) {
                found = false;
                cache.head += i + 1;
                n_tested   += i + 1;
                break;
            }
        }

        // 如果找到可用插槽，则跳出循环
        if (found) {
            break;
        }

        // 如果已经测试了所有插槽仍未找到可用插槽，则返回 false
        if (n_tested >= n_ctx) {
            //WHISPER_LOG_ERROR("%s: failed to find a slot for %d tokens\n", __func__, n_tokens);
            return false;
        }
    }

    // 将批处理数据存储到缓存中的插槽中
    for (uint32_t i = 0; i < n_tokens; i++) {
        cache.cells[cache.head + i].pos = batch.pos[i];

        for (int32_t j = 0; j < batch.n_seq_id[i]; j++) {
            cache.cells[cache.head + i].seq_id.insert(batch.seq_id[i][j]);
        }
    }

    return true;
}

// 查找当前正在使用的单元格数量
static int32_t whisper_kv_cache_cell_max(const struct whisper_kv_cache & cache) {
    // 从后往前遍历缓存，找到第一个正在使用的单元格并返回其索引加一
    for (uint32_t i = cache.size - 1; i > 0; --i) {
        if (cache.cells[i].pos >= 0 && !cache.cells[i].seq_id.empty()) {
            return i + 1;
        }
    }

    return 1;
}

// 清空缓存中的数据
static void whisper_kv_cache_clear(struct whisper_kv_cache & cache) {
    // 遍历缓存中的所有元素
    for (int32_t i = 0; i < (int32_t) cache.size; ++i) {
        // 将当前元素的位置设为-1
        cache.cells[i].pos = -1;
        // 清空当前元素的序列ID
        cache.cells[i].seq_id.clear();
    }
    // 将缓存的头部位置设为0
    cache.head = 0;
// 从缓存中删除指定序列 ID 在指定位置范围内的数据
static void whisper_kv_cache_seq_rm(
        struct whisper_kv_cache & cache,
                 whisper_seq_id   seq_id,
                    whisper_pos   p0,
                    whisper_pos   p1) {
    // 初始化新的头指针为缓存大小
    uint32_t new_head = cache.size;

    // 如果 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    // 如果 p1 小于 0，则将其设置为 whisper_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<whisper_pos>::max();

    // 遍历缓存中的每个元素
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果当前元素的位置在 p0 和 p1 之间
        if (cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 如果 seq_id 小于 0，则清空当前元素的序列 ID
            if (seq_id < 0) {
                cache.cells[i].seq_id.clear();
            } else if (cache.cells[i].has_seq_id(seq_id)) {
                // 如果当前元素包含指定的序列 ID，则删除该序列 ID
                cache.cells[i].seq_id.erase(seq_id);
            } else {
                // 否则继续下一个元素
                continue;
            }
            // 如果当前元素的序列 ID 为空，则将其位置设置为 -1
            if (cache.cells[i].seq_id.empty()) {
                cache.cells[i].pos = -1;
                // 如果新的头指针还未更新，则更新为当前元素的索引
                if (new_head == cache.size) new_head = i;
            }
        }
    }

    // 如果释放了一个槽位，则将头指针设置为该槽位，以便从该位置开始搜索
    if (new_head != cache.size) cache.head = new_head;
}

// 将指定序列 ID 的数据复制到另一个序列 ID
static void whisper_kv_cache_seq_cp(
        struct whisper_kv_cache & cache,
                 whisper_seq_id   seq_id_src,
                 whisper_seq_id   seq_id_dst,
                    whisper_pos   p0,
                    whisper_pos   p1) {
    // 如果 p0 小于 0，则将其设置为 0
    if (p0 < 0) p0 = 0;
    // 如果 p1 小于 0，则将其设置为 whisper_pos 类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<whisper_pos>::max();

    // 将头指针设置为 0
    cache.head = 0;

    // 遍历缓存中的每个元素
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果当前元素包含指定的序列 ID 并且位置在 p0 和 p1 之间
        if (cache.cells[i].has_seq_id(seq_id_src) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            // 将指定序列 ID 插入到当前元素的序列 ID 中
            cache.cells[i].seq_id.insert(seq_id_dst);
        }
    }
}

// 初始化 Whisper 后端
static ggml_backend_t whisper_backend_init(const whisper_context_params & params) {
    // 初始化 GPU 后端为 NULL
    ggml_backend_t backend_gpu = NULL;

    // 初始化后端
#ifdef GGML_USE_CUBLAS
    # 如果参数中指定使用 GPU，并且 CUDA 库已加载
    if (params.use_gpu && ggml_cublas_loaded()) {
        # 打印信息，表示正在使用 CUDA 后端
        WHISPER_LOG_INFO("%s: using CUDA backend\n", __func__);
        # 初始化 CUDA 后端
        backend_gpu = ggml_backend_cuda_init(0);
        # 如果初始化失败
        if (!backend_gpu) {
            # 打印错误信息，表示 CUDA 后端初始化失败
            WHISPER_LOG_ERROR("%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#ifdef GGML_USE_METAL
    // 如果使用 GPU，并且 Metal 可用，则初始化 Metal 后端
    if (params.use_gpu) {
        // 输出日志信息，指示使用 Metal 后端
        WHISPER_LOG_INFO("%s: using Metal backend\n", __func__);
        // 设置 Metal 后端日志回调函数
        ggml_backend_metal_log_set_callback(g_state.log_callback, g_state.log_callback_user_data);
        // 初始化 Metal 后端
        backend_gpu = ggml_backend_metal_init();
        // 检查 Metal 后端初始化是否成功
        if (!backend_gpu) {
            // 输出错误日志信息，指示 Metal 后端初始化失败
            WHISPER_LOG_ERROR("%s: ggml_backend_metal_init() failed\n", __func__);
        } else if (!ggml_backend_metal_supports_family(backend_gpu, 7)) {
            // 检查 Metal GPU 是否支持 family 7，如果不支持则回退到 CPU
            WHISPER_LOG_ERROR("%s: Metal GPU does not support family 7 - falling back to CPU\n", __func__);
            // 释放 Metal 后端资源
            ggml_backend_free(backend_gpu);
            backend_gpu = NULL;
        }
    }
#endif

    // 如果存在 GPU 后端，则返回 GPU 后端
    if (backend_gpu) {
        return backend_gpu;
    }
    // 否则初始化 CPU 后端并返回
    return ggml_backend_cpu_init();
}

// 从 ggml 文件加载模型
//
// 文件格式:
//
//   - hparams
//   - 预先计算的 mel 滤波器
//   - 词汇表
//   - 权重
//
// 详细信息请参考 convert-pt-to-ggml.py 脚本
//
static bool whisper_model_load(struct whisper_model_loader * loader, whisper_context & wctx) {
    // 输出日志信息，指示加载模型
    WHISPER_LOG_INFO("%s: loading model\n", __func__);

    // 获取加载模型的起始时间
    const int64_t t_start_us = ggml_time_us();

    // 将起始时间保存到 wctx 中
    wctx.t_start_us = t_start_us;

    // 获取模型和词汇表的引用
    auto & model = wctx.model;
    auto & vocab = wctx.vocab;

    // 验证魔数
    {
        uint32_t magic;
        // 从 loader 中安全读取魔数
        read_safe(loader, magic);
        // 如果魔数不匹配 GGML_FILE_MAGIC，则输出错误日志信息
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

        // 从 loader 中读取 mel 滤波器的参数
        read_safe(loader, filters.n_mel);
        read_safe(loader, filters.n_fft);

        // 调整 filters.data 的大小
        filters.data.resize(filters.n_mel * filters.n_fft);
        // 从 loader 中读取 filters.data 的数据
        loader->read(loader->context, filters.data.data(), filters.data.size() * sizeof(float));
        // 对 filters 进行字节交换
        BYTESWAP_FILTERS(filters);
    }

    // 加载词汇表
    }

    // 获取 wctx 的 wtype
    const ggml_type wtype = wctx.wtype;
    // 定义变量 vtype，根据条件 wctx.wtype == GGML_TYPE_F32 判断赋值为 GGML_TYPE_F32 或 GGML_TYPE_F16，表示转换类型

    // 创建 ggml 上下文
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;

        // 获取音频层和文本层的数量
        const int n_audio_layer = hparams.n_audio_layer;
        const int n_text_layer  = hparams.n_text_layer;

        // 计算需要的张量数量
        const size_t n_tensors = 10 /* input */ + 15 + 15*n_audio_layer + 24*n_text_layer;

        // 初始化 ggml_init_params 结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ n_tensors*ggml_tensor_overhead(),
            /*.mem_buffer =*/ nullptr,
            /*.no_alloc   =*/ true,
        };

        // 初始化 ggml 上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，记录错误信息并返回 false
        if (!model.ctx) {
            WHISPER_LOG_ERROR("%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 准备权重的张量
    }

    // 初始化 whisper 后端
    wctx.backend = whisper_backend_init(wctx.params);

    // 一些设备对单个内存缓冲区的最大大小有限制
    // 例如，iPhone 对每个缓冲区限制为 1GB
    // 为了解决这个问题，我们将分配多个较小大小的缓冲区，并将模型权重的张量分割在它们之间
    //
    // map_t2b 将张量名称映射到缓冲区索引
    // 遍历张量时，当当前缓冲区已满时，我们将分配新的缓冲区
    //
    // 最后，我们为每个缓冲区创建一个单独的分配器，并用它来分配张量
    // 我们保持分配器活动，直到所有张量加载完毕

    // 断言模型的缓冲区为空
    GGML_ASSERT(model.buffers.empty());

    // 创建映射张量名称到缓冲区索引的 map
    std::map<std::string, int> map_t2b;
    {
        // 初始化主缓冲区大小和当前缓冲区大小
        size_t size_main = 0;
        size_t size_cur  = 0;
    
        // 定义常量 GB 为 1GB 的大小
        static const size_t GB = 1024ull*1024ull*1024ull;
    
        // 遍历模型中的张量
        for (const auto & t : model.tensors) {
            // 计算当前张量的大小，包括额外开销
            const size_t cur = ggml_nbytes(t.second) + ggml_tensor_overhead();
    
            // 如果将张量添加到当前缓冲区会超过限制，则需要分配新的缓冲区
            if (size_cur + cur > GB) {
                // 断言当前缓冲区大小大于0，确保张量不会过大无法放入单个缓冲区
                GGML_ASSERT(size_cur > 0 && "A tensor is too large to fit in a single buffer");
    
                // 在模型的缓冲区列表中添加新的缓冲区
                model.buffers.emplace_back(ggml_backend_alloc_buffer(wctx.backend, size_cur));
    
                // 更新当前缓冲区大小为当前张量大小
                size_cur = cur;
            }
    
            // 将张量名称映射到缓冲区索引
            map_t2b[t.first] = model.buffers.size();
    
            // 更新当前缓冲区大小和主缓冲区大小
            size_cur  += cur;
            size_main += cur;
        }
    
        // 如果当前缓冲区大小大于0，则分配最后一个缓冲区
        if (size_cur > 0) {
            model.buffers.emplace_back(ggml_backend_alloc_buffer(wctx.backend, size_cur));
        }
    
        // 断言模型的缓冲区列表大小大于0
        GGML_ASSERT(model.buffers.size() > 0);
    
        // 输出日志信息，显示总大小和缓冲区数量
        WHISPER_LOG_INFO("%s: %8s total size = %8.2f MB (%d buffers)\n", __func__, ggml_backend_name(wctx.backend), size_main / 1e6, (int) model.buffers.size());
    }
    
    // 创建一个存储指向缓冲区的指针的向量
    std::vector<ggml_allocr *> allocs(model.buffers.size());
    for (size_t i = 0; i < allocs.size(); ++i) {
        // 从缓冲区创建新的分配器
        allocs[i] = ggml_allocr_new_from_buffer(model.buffers[i]);
    }
    
    // 在后端缓冲区中分配张量
    {
        for (const auto & t : model.tensors) {
            ggml_allocr_alloc(allocs[map_t2b[t.first]], t.second);
        }
    }
    
    // 加载权重
#ifdef GGML_USE_METAL
                || ggml_backend_is_metal(backend)
#endif
                )) {
                // 如果定义了 GGML_USE_METAL 宏或者使用 Metal 后端，则直接读取到张量中
                loader->read(loader->context, tensor->data, ggml_nbytes(tensor));
                BYTESWAP_TENSOR(tensor);
            } else {
                // 否则先读取到临时缓冲区，然后再复制到设备内存中
                read_buf.resize(ggml_nbytes(tensor));

                loader->read(loader->context, read_buf.data(), read_buf.size());

                ggml_backend_tensor_set(tensor, read_buf.data(), 0, ggml_nbytes(tensor));
            }

            //printf("%48s - [%5d, %5d, %5d], type = %6s, %6.2f MB\n", name.data(), ne[0], ne[1], ne[2], ggml_type_name((ggml_type) ttype), ggml_nbytes(tensor)/1e6);
            total_size += ggml_nbytes(tensor);
            model.n_loaded++;
        }

        WHISPER_LOG_INFO("%s: model size    = %7.2f MB\n", __func__, total_size/1e6);

        if (model.n_loaded == 0) {
            WHISPER_LOG_WARN("%s: WARN no tensors loaded from model file - assuming empty model for testing\n", __func__);
        } else if (model.n_loaded != (int) model.tensors.size()) {
            WHISPER_LOG_ERROR("%s: ERROR not all tensors loaded from model file - expected %zu, got %d\n", __func__, model.tensors.size(), model.n_loaded);
            return false;
        }
    }

    for (auto & alloc : allocs) {
        ggml_allocr_free(alloc);
    }

    wctx.t_load_us = ggml_time_us() - t_start_us;

    return true;
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

    // 返回是否使用 CoreML 或 OpenVINO 进行编码
    return use_coreml || use_openvino;
}
// 构建卷积神经网络的计算图
static struct ggml_cgraph * whisper_build_graph_conv(
        whisper_context & wctx,
          whisper_state & wstate,
              const int   mel_offset) {
    // 获取模型、输入 mel 数据和超参数
    const auto & model   = wctx.model;
    const auto & mel_inp = wstate.mel;
    const auto & hparams = model.hparams;

    // 计算上下文大小和音频状态大小
    const int n_ctx   = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;
    const int n_state = hparams.n_audio_state; GGML_UNUSED(n_state);

    // 获取 mel 频谱的大小
    const int n_mels = hparams.n_mels;

    // 初始化参数结构体
    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_conv.meta.size(),
        /*.mem_buffer =*/ wstate.alloc_conv.meta.data(),
        /*.no_alloc   =*/ true,
    };

    // 初始化上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建新的计算图
    ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 获取分配器
    ggml_allocr * alloc = wstate.alloc_conv.alloc;

    // 创建 mel 张量
    struct ggml_tensor * mel = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, 2*n_ctx, n_mels);
    ggml_allocr_alloc(alloc, mel);

    // 断言 mel 张量类型为 GGML_TYPE_F32
    assert(mel->type == GGML_TYPE_F32);
    // 如果分配器不是测量模式
    if (!ggml_allocr_is_measure(alloc)) {
        // 断言输入 mel 数据的 mel 频谱大小与模型中的一致
        assert(mel_inp.n_mel == n_mels);

        // 调整输入 mel 数据的大小
        wstate.inp_mel.resize(ggml_nelements(mel));

        float * dst = wstate.inp_mel.data();
        // 将目标数组初始化为 0
        memset(dst, 0, ggml_nbytes(mel));

        // 计算 mel 数据的起始和结束索引
        const int i0 = std::min(mel_offset,           mel_inp.n_len);
        const int i1 = std::min(mel_offset + 2*n_ctx, mel_inp.n_len);

        // 将 mel 数据复制到目标数组中
        for (int j = 0; j < mel_inp.n_mel; ++j) {
            for (int i = i0; i < i1; ++i) {
                dst[j*2*n_ctx + (i - i0)] = mel_inp.data[j*mel_inp.n_len + i];
            }
        }

        // 设置 mel 张量的数据
        ggml_backend_tensor_set(mel, wstate.inp_mel.data(), 0, ggml_nelements(mel)*sizeof(float));
    }

    // 初始化当前张量为 nullptr
    struct ggml_tensor * cur = nullptr;
}
    // 如果外部编码失败，则执行以下操作
    if (!whisper_encode_external(wstate)) {
        // 执行卷积操作并应用 GELU 激活函数
        {
            // 使用 ctx0 执行一维卷积操作，传入模型的第一个卷积层权重和 mel 数据，步长为 1，填充为 1
            cur = ggml_conv_1d_ph(ctx0, model.e_conv_1_w, mel, 1, 1);
            // 添加第一个卷积层的偏置
            cur = ggml_add(ctx0, cur, model.e_conv_1_b);

            // 应用 GELU 激活函数
            cur = ggml_gelu(ctx0, cur);

            // 使用 ctx0 执行一维卷积操作，传入模型的第二个卷积层权重和上一步的结果 cur，步长为 2，填充为 1
            cur = ggml_conv_1d_ph(ctx0, model.e_conv_2_w, cur, 2, 1);
            // 添加第二个卷积层的偏置
            cur = ggml_add(ctx0, cur, model.e_conv_2_b);

            // 应用 GELU 激活函数
            cur = ggml_gelu(ctx0, cur);
        }

        // 设置当前节点的名称为 "embd_conv"
        ggml_set_name(cur, "embd_conv");
        // 将当前节点 cur 赋值给 wstate 结构体的 embd_conv 字段
        wstate.embd_conv = cur;
    } else {
#ifdef WHISPER_USE_COREML
        // 如果使用 CoreML
        cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx);
        // 创建一个新的二维浮点型张量
        ggml_allocr_alloc(alloc, cur);
        // 分配张量内存空间

        if (!ggml_allocr_is_measure(alloc)) {
            // 如果不是测量模式
            whisper_coreml_encode(wstate.ctx_coreml, mel->ne[0], mel->ne[1], (float *) mel->data, (float *) cur->data);
            // 使用 CoreML 对数据进行编码
        }
#endif
#ifdef WHISPER_USE_OPENVINO
        // 如果使用 OpenVINO
        cur = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx);
        // 创建一个新的二维浮点型张量
        ggml_allocr_alloc(alloc, cur);
        // 分配张量内存空间

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
    // 构建前向传播

    ggml_free(ctx0);
    // 释放上下文内存空间

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
    // 创建一个新的张量，用于查看 wstate.embd_conv 的内容

    const float KQscale = 1.0f/sqrtf(float(n_state)/n_head);
    // 计算 KQscale 的值
    // ===================================================================
    // NOTE: experimenting with partial evaluation of the encoder (ignore)
    //static int iter = -1;
    //const int n_iter = 1500/n_ctx;

    //iter = (iter + 1) % n_iter;

    //if (iter == 0) {
    //    memset(model.memory_cross_k->data, 0, ggml_nbytes(model.memory_cross_k));
    //    memset(model.memory_cross_v->data, 0, ggml_nbytes(model.memory_cross_v));
    //}

    // 初始化迭代次数为0
    static int iter = 0;

    // 计算 e_pe 的步长
    const size_t e_pe_stride = model.e_pe->ne[0]*ggml_element_size(model.e_pe);
    // 计算 e_pe 的偏移量
    const size_t e_pe_offset = model.e_pe->ne[0]*ggml_element_size(model.e_pe)*n_ctx*iter;

    // 创建一个视图，用于处理 e_pe 和 cur 张量
    struct ggml_tensor * e_pe = ggml_view_2d(ctx0, model.e_pe, model.e_pe->ne[0], n_ctx, e_pe_stride, e_pe_offset);
    // 将 e_pe 和 cur 张量相加，并将结果赋给 cur
    cur = ggml_add(ctx0, e_pe, ggml_cont(ctx0, ggml_transpose(ctx0, cur)));

    // ===================================================================

    // original:
    //cur = ggml_add(ctx0, model.e_pe, ggml_transpose(ctx0, cur));

    // 将 cur 张量赋给 inpL
    struct ggml_tensor * inpL = cur;
    // 遍历每一层编码器
    for (int il = 0; il < n_layer; ++il) {
        // 获取当前层的引用
        const auto & layer = model.layers_encoder[il];

        // norm
        {
            // 对输入进行归一化处理
            cur = ggml_norm(ctx0, inpL, hparams.eps);

            // 对归一化后的结果进行线性变换
            cur = ggml_add(ctx0,
                    ggml_mul(ctx0, cur, layer.attn_ln_0_w),
                    layer.attn_ln_0_b);
        }

        // self-attention
        {
            // 计算 Qcur = layer.attn_q_w * cur
            struct ggml_tensor * Qcur = ggml_mul_mat(ctx0,
                    layer.attn_q_w,
                    cur);

            // 添加偏置项 layer.attn_q_b
            Qcur = ggml_add(ctx0, Qcur, layer.attn_q_b);

            // 计算 Key，没有偏置项
            struct ggml_tensor * Kcur = ggml_mul_mat(ctx0,
                    layer.attn_k_w,
                    cur);

            // 计算 Value
            struct ggml_tensor * Vcur = ggml_mul_mat(ctx0,
                    layer.attn_v_w,
                    cur);

            // 添加偏置项 layer.attn_v_b
            Vcur = ggml_add(ctx0, Vcur, layer.attn_v_b);

            // ------
#ifdef WHISPER_USE_FLASH_ATTN
            // 创建 Q 张量，将 Qcur 张量按照指定维度顺序复制并排列
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // 创建 K 张量，将 Kcur 张量按照指定维度顺序复制并排列
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Kcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // 创建 V 张量，将 Vcur 张量按照指定维度顺序复制、重塑并排列
            struct ggml_tensor * V =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                Vcur,
                                n_state/n_head, n_head, n_ctx),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, wctx.itype, n_ctx, n_state/n_head, n_head));

            // 创建 KQV 张量，进行闪电注意力计算
            struct ggml_tensor * KQV = ggml_flash_attn(ctx0, Q, K, V, false);
#else
            // 如果不满足条件，则执行以下代码块

            // 对当前的 Q 张量进行维度置换操作，得到新的张量 Q
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Qcur,
                            ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // 对当前的 K 张量进行维度置换操作，得到新的张量 K
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        ggml_cpy(ctx0,
                            Kcur,
                            ggml_new_tensor_3d(ctx0, wctx.itype, n_state/n_head, n_head, n_ctx)),
                        0, 2, 1, 3);

            // 计算 K 与 Q 的矩阵乘法，得到新的张量 KQ
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // 对 KQ 张量进行缩放操作，得到新的张量 KQ_scaled
            struct ggml_tensor * KQ_scaled = ggml_scale(ctx0, KQ, KQscale);

            // 对 KQ_scaled 张量进行 Softmax 操作，得到新的张量 KQ_soft_max
            struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_scaled);

            // 对当前的 V 张量进行维度置换和重塑操作，得到新的张量 V
            struct ggml_tensor * V =
                ggml_cpy(ctx0,
                        ggml_permute(ctx0,
                            ggml_reshape_3d(ctx0,
                                Vcur,
                                n_state/n_head, n_head, n_ctx),
                            1, 2, 0, 3),
                        ggml_new_tensor_3d(ctx0, wctx.itype, n_ctx, n_state/n_head, n_head)
                        );

            // 计算 V 与 KQ_soft_max 的矩阵乘法，得到新的张量 KQV
            struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);
        # 结束条件判断
        # 如果满足条件，则执行以下代码块
#endif

            # 对输入张量 KQV 进行维度置换操作，生成新的张量 KQV_merged
            struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

            # 复制张量 KQV_merged，生成新的张量 cur
            cur = ggml_cpy(ctx0,
                    KQV_merged,
                    ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_state, n_ctx));
        }

        # 注意力机制中的投影操作
        {
            # 矩阵乘法操作，将 cur 与 layer.attn_ln_1_w 相乘
            cur = ggml_mul_mat(ctx0,
                    layer.attn_ln_1_w,
                    cur);

            # 加法操作，将 cur 与 layer.attn_ln_1_b 相加
            cur = ggml_add(ctx0, cur, layer.attn_ln_1_b);
        }

        # 将输入张量 inpL 加到当前张量 cur 上
        cur = ggml_add(ctx0, cur, inpL);

        # 将当前张量 cur 赋值给 inpFF
        struct ggml_tensor * inpFF = cur;

        # 前馈神经网络
        {
            # 归一化操作
            {
                # 对 inpFF 进行归一化操作，使用参数 hparams.eps
                cur = ggml_norm(ctx0, inpFF, hparams.eps);

                # 矩阵乘法和加法操作，cur = mlp_ln_w*cur + mlp_ln_b
                cur = ggml_add(ctx0,
                        ggml_mul(ctx0, cur, layer.mlp_ln_w),
                        layer.mlp_ln_b);
            }

            # 根据宏定义选择不同的前馈网络实现
#ifdef WHISPER_USE_FLASH_FF
            # 使用快闪前馈网络进行计算
            cur = ggml_flash_ff(ctx0,
                    ggml_cpy(ctx0, cur, ggml_new_tensor_2d(ctx0, wstate.itype, n_state, n_ctx)),
                    layer.mlp_0_w, layer.mlp_0_b, layer.mlp_1_w, layer.mlp_1_b);
#else
            # 全连接层操作
            cur = ggml_mul_mat(ctx0,
                    layer.mlp_0_w,
                    cur);

            cur = ggml_add(ctx0, cur, layer.mlp_0_b);

            # GELU 激活函数操作
            cur = ggml_gelu(ctx0, cur);

            # 投影操作
            cur = ggml_mul_mat(ctx0,
                    layer.mlp_1_w,
                    cur);

            cur = ggml_add(ctx0, cur, layer.mlp_1_b);
#endif
        }

        # 将当前张量 cur 加到输入张量 inpFF 上
        inpL = ggml_add(ctx0, cur, inpFF);
    }

    # 将当前张量 cur 赋值给 inpL
    cur = inpL;

    # 归一化操作
    {
        # 对当前张量 cur 进行归一化操作，使用参数 hparams.eps
        cur = ggml_norm(ctx0, cur, hparams.eps);

        # 矩阵乘法和加法操作，cur = ln_f_g*cur + ln_f_b
        cur = ggml_add(ctx0,
                ggml_mul(ctx0, cur, model.e_ln_w),
                model.e_ln_b);
    }

    # 构建前向传播扩展
    ggml_build_forward_expand(gf, cur);

    # 更新 wstate 中的 embd_enc 字段
    wstate.embd_enc = cur;

    # 打印图结构
    #ggml_graph_print(gf);
    ////////////////////////////////////////////////////////////////////////////

    // 打印内存使用情况，包括函数名、使用的内存量（单位为 MB）
    //printf("%s: used_mem = %f MB, %f MB, %f MB %f MB %f MB\n", __func__,
    //        ggml_used_mem(ctx0)/1e6,
    //        wstate.get_buf_max_mem(0)/1e6,
    //        wstate.get_buf_max_mem(1)/1e6,
    //        wstate.get_buf_max_mem(2)/1e6,
    //        wstate.get_buf_max_mem(3)/1e6);

    // 释放上下文 ctx0 所占用的内存
    ggml_free(ctx0);

    // 返回 gf 变量
    return gf;
// 预先计算交叉注意力内存
static struct ggml_cgraph * whisper_build_graph_cross(
        whisper_context & wctx,
        whisper_state & wstate) {
    // 获取模型和超参数
    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    // 获取上下文大小、状态大小和头数
    const int n_ctx   = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx;
    const int n_state = hparams.n_audio_state;
    const int n_head  = hparams.n_audio_head;

    // 初始化参数结构体
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

    // 计算 Kscale 值
    const float  Kscale = pow(float(n_state) / n_head, -0.25);
}
    // 遍历模型中的文本层
    for (int il = 0; il < model.hparams.n_text_layer; ++il) {
        // 获取当前文本层
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

        // 添加交叉注意力的偏置
        Vcross = ggml_add(ctx0,
                    Vcross,
                    layer.cross_attn_v_b);

        // 转置 V 值并重塑为二维张量
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

    // 打印图形
    //ggml_graph_print(gf);

    // 释放上下文
    ggml_free(ctx0);

    // 返回图形
    return gf;
// 评估给定状态下的编码器
//
// 给定音频录制（更具体地说，其对数梅尔频谱图），运行编码器的前向传递部分，返回编码后的特征
//
//   - wctx:      模型
//   - wstate:     编码器的状态
//   - n_threads:  要使用的线程数
//   - mel_offset: 梅尔频谱图中的偏移量（即音频偏移）
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
        // 获取分配器
        auto & alloc = wstate.alloc_conv.alloc;

        // 重置分配器
        ggml_allocr_reset(alloc);

        // 构建卷积图
        ggml_cgraph * gf = whisper_build_graph_conv(wctx, wstate, mel_offset);

        // 分配卷积图
        ggml_allocr_alloc_graph(alloc, gf);

        // 如果未外部编码，则计算卷积图
        if (!whisper_encode_external(wstate)) {
            if (!ggml_graph_compute_helper(wstate.backend, gf, n_threads)) {
                return false;
            }
        }
    }

    // encoder
    if (!whisper_encode_external(wstate)) {
        // 获取分配器
        auto & alloc = wstate.alloc_encode.alloc;

        // 重置分配器
        ggml_allocr_reset(alloc);

        // 构建编码器图
        ggml_cgraph * gf = whisper_build_graph_encoder(wctx, wstate);

        // 分配编码器图
        ggml_allocr_alloc_graph(alloc, gf);

        // 计算编码器图
        if (!ggml_graph_compute_helper(wstate.backend, gf, n_threads)) {
            return false;
        }
    }

    // cross
    {
        // 获取分配器
        auto & alloc = wstate.alloc_cross.alloc;

        // 重置分配器
        ggml_allocr_reset(alloc);

        // 构建交叉图
        ggml_cgraph * gf = whisper_build_graph_cross(wctx, wstate);

        // 分配交叉图
        ggml_allocr_alloc_graph(alloc, gf);

        // 计算交叉图
        if (!ggml_graph_compute_helper(wstate.backend, gf, n_threads)) {
            return false;
        }
    }

    // 更新编码时间和编码次数
    wstate.t_encode_us += ggml_time_us() - t_start_us;
    wstate.n_encode++;
}
    # 如果存在中止回调函数并且中止回调函数返回真，则返回假；否则返回真
    return !(abort_callback && abort_callback(abort_callback_data));
// 构建解码器的计算图
static struct ggml_cgraph * whisper_build_graph_decoder(
         whisper_context & wctx,  // Whisper 上下文
         whisper_state   & wstate, // Whisper 状态
     const whisper_batch & batch) { // Whisper 批处理数据

    const auto & model   = wctx.model; // 获取 Whisper 模型
    const auto & hparams = model.hparams; // 获取模型超参数

    auto & kv_self = wstate.kv_self; // 获取状态中的自身键值对

    WHISPER_ASSERT(!!kv_self.ctx); // 断言自身键值对的上下文存在

    ggml_allocr * alloc = wstate.alloc_decode.alloc; // 获取解码器的分配器

    const int n_ctx   = kv_self.size; // 自身键值对的大小
    const int n_state = hparams.n_text_state; // 文本状态的大小
    const int n_head  = hparams.n_text_head; // 文本头的大小
    const int n_layer = hparams.n_text_layer; // 文本层的大小

    const int n_tokens    = batch.n_tokens; // 批处理数据中的标记数量
    const int n_audio_ctx = wstate.exp_n_audio_ctx > 0 ? wstate.exp_n_audio_ctx : hparams.n_audio_ctx; // 音频上下文的大小

    const int32_t n_kv     = ggml_allocr_is_measure(alloc) ? n_ctx            : kv_self.n; // 键值对的数量
    const int32_t kv_head  = ggml_allocr_is_measure(alloc) ? n_ctx - n_tokens : kv_self.head; // 键值对头的数量

    //WHISPER_LOG_DEBUG("%s: n_past = %d, n_tokens = %d, n_audio_ctx = %d, n_ctx = %d\n", __func__, n_past, n_tokens, n_audio_ctx, n_ctx);

    // 初始化参数
    struct ggml_init_params params = {
        /*.mem_size   =*/ wstate.alloc_decode.meta.size(), // 内存大小
        /*.mem_buffer =*/ wstate.alloc_decode.meta.data(), // 内存缓冲区
        /*.no_alloc   =*/ true, // 不分配内存
    };

    // 初始化上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建计算图
    ggml_cgraph * gf = ggml_new_graph_custom(ctx0, WHISPER_MAX_NODES, false);

    // 创建嵌入张量
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
    ggml_allocr_alloc(alloc, embd);

    // 如果不是度量模式，则设置嵌入张量的值
    if (!ggml_allocr_is_measure(alloc)) {
        ggml_backend_tensor_set(embd, batch.token, 0, n_tokens*ggml_element_size(embd));
    }

    // 创建位置张量
    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);
    ggml_allocr_alloc(alloc, position);

    // 如果不是度量模式，则设置位置张量的值
    if (!ggml_allocr_is_measure(alloc)) {
        for (int i = 0; i < n_tokens; ++i) {
            const int32_t val = batch.pos[i];
            ggml_backend_tensor_set(position, &val, i*sizeof(int32_t), sizeof(int32_t));
        }
    }
}
    // 计算 KQscale 的值，用于后续计算
    const float KQscale = pow(float(n_state)/n_head, -0.25);

    // 创建一个 3D 的浮点数类型的张量 KQ_mask
    struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
    // 分配内存给 KQ_mask
    ggml_allocr_alloc(alloc, KQ_mask);

    // 如果内存不是度量，则执行以下操作
    if (!ggml_allocr_is_measure(alloc)) {
        // 调整输入掩码的大小
        wstate.inp_mask.resize(n_kv*n_tokens);

        // 获取输入掩码的数据指针
        float * data = wstate.inp_mask.data();
        // 将输入掩码数据初始化为 0
        memset(data, 0, ggml_nbytes(KQ_mask));

        // 遍历输入掩码的维度
        for (int h = 0; h < 1; ++h) {
            for (int j = 0; j < n_tokens; ++j) {
                const whisper_pos    pos    = batch.pos[j];
                const whisper_seq_id seq_id = batch.seq_id[j][0];

                for (int i = 0; i < n_kv; ++i) {
                    // 如果不满足条件，则将数据设置为负无穷
                    if (!kv_self.cells[i].has_seq_id(seq_id) || kv_self.cells[i].pos > pos) {
                        data[h*(n_kv*n_tokens) + j*n_kv + i] = -INFINITY;
                    }
                }
            }
        }

        // 将输入掩码数据设置到 KQ_mask 中
        ggml_backend_tensor_set(KQ_mask, wstate.inp_mask.data(), 0, ggml_nelements(KQ_mask)*sizeof(float));
    }

    // 对当前张量进行编码和位置编码
    struct ggml_tensor * cur =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.d_te, embd),
                ggml_get_rows(ctx0, model.d_pe, position));

    // 将当前张量设置为输入张量
    struct ggml_tensor * inpL = cur;

    }

    // 将当前张量设置为 cur
    cur = inpL;

    // 归一化
    {
        // 对当前张量进行归一化
        cur = ggml_norm(ctx0, cur, hparams.eps);

        // 对当前张量进行加法和乘法操作
        cur = ggml_add(ctx0,
                ggml_mul(ctx0,
                    cur,
                    model.d_ln_w),
                model.d_ln_b);
    }

    // 仅为最后一个标记计算对数
    // 将此行注释掉以计算所有 n_tokens 的对数
    // 可能在未来有用
    //cur = ggml_view_2d(ctx0, cur, cur->ne[0], 1, cur->nb[1], (cur->ne[1] - 1)*cur->nb[1]);

    // 计算 logits
    struct ggml_tensor * logits = ggml_mul_mat(ctx0, model.d_te, cur);

    // 构建前向扩展
    ggml_build_forward_expand(gf, logits);

    // 释放上下文
    ggml_free(ctx0);

    // 返回 gf
    return gf;
// 结束大括号，表示函数 whisper_decode_internal 的结束
}

// 评估解码器
//
// 给定文本提示 + 音频特征 -> 计算下一个标记的对数
//
//   - model:      模型
//   - n_threads:  使用的线程数
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
    // 记录开始时间
    const int64_t t_start_us = ggml_time_us();

    // 获取模型和超参数
    const auto & model   = wctx.model;
    const auto & hparams = model.hparams;

    // 获取词汇表大小和提示中的标记数
    const int n_vocab  = hparams.n_vocab;
    const int n_tokens = batch.n_tokens;

    // 获取输出 logits 的引用
    auto & logits_out = wstate.logits;

    struct ggml_tensor * logits;

    // 查找批次的 KV 槽
    {
        auto & kv_self = wstate.kv_self;

        // 如果找不到批次的 KV 槽，则返回 false
        if (!whisper_kv_cache_find_slot(kv_self, batch)) {
            return false;
        }

        // 更新 KV 槽的大小
        kv_self.n = whisper_kv_cache_cell_max(kv_self);
        //kv_self.n = std::min((int32_t) hparams.n_text_ctx, std::max(32, whisper_kv_cache_cell_max(kv_self)));
        //printf("n_tokens = %5d, kv_self.head = %5d, kv_self.n = %5d, seq_id = %5d\n", batch.n_tokens, kv_self.head, kv_self.n, batch.seq_id[0][0]);
    }

    // 解码器
    {
        auto & alloc = wstate.alloc_decode.alloc;

        // 重置分配器
        ggml_allocr_reset(alloc);

        // 构建解码器图
        ggml_cgraph * gf = whisper_build_graph_decoder(wctx, wstate, batch);

        // 分配图
        ggml_allocr_alloc_graph(alloc, gf);

        // 获取 logits 节点
        logits = gf->nodes[gf->n_nodes - 1];

        // 使用指定线程数计算图
        if (!ggml_graph_compute_helper(wstate.backend, gf, n_threads)) {
            return false;
        }
    }

    // 调整 logits_out 的大小
    logits_out.resize(n_tokens*n_vocab);
    // 遍历 n_tokens 次
    for (int i = 0; i < n_tokens; i++) {
        // 如果 batch.logits[i] 等于 0，则跳过当前循环
        if (batch.logits[i] == 0) {
            continue;
        }
        // 从 logits 中获取数据，存储到 logits_out 中的指定位置
        ggml_backend_tensor_get(logits, logits_out.data() + (n_vocab*i), sizeof(float)*(n_vocab*i), sizeof(float)*n_vocab);
    }

    // 如果 batch 中的 n_tokens 大于 1
    if (batch.n_tokens > 1) {
        // 打印内存使用情况
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

    // 返回是否需要中止回调函数的结果
    return !(abort_callback && abort_callback(abort_callback_data));
// 闭合之前的函数定义
}

// 将整数时间戳转换为时间戳字符串，可选择是否使用逗号分隔符
static std::string to_timestamp(int64_t t, bool comma = false) {
    // 将时间戳乘以10，得到毫秒级时间戳
    int64_t msec = t * 10;
    // 计算小时部分
    int64_t hr = msec / (1000 * 60 * 60);
    msec = msec - hr * (1000 * 60 * 60);
    // 计算分钟部分
    int64_t min = msec / (1000 * 60);
    msec = msec - min * (1000 * 60);
    // 计算秒部分
    int64_t sec = msec / 1000;
    msec = msec - sec * 1000;

    // 格式化时间戳字符串
    char buf[32];
    snprintf(buf, sizeof(buf), "%02d:%02d:%02d%s%03d", (int) hr, (int) min, (int) sec, comma ? "," : ".", (int) msec);

    return std::string(buf);
}

// 定义正弦和余弦值的预计算表
#define SIN_COS_N_COUNT WHISPER_N_FFT
static float sin_vals[SIN_COS_N_COUNT];
static float cos_vals[SIN_COS_N_COUNT];

// 在FFT中，我们经常使用相同值的正弦和余弦运算。
// 我们可以使用预计算的值来加速这个过程。
static void fill_sin_cos_table() {
    static bool is_filled = false;
    if (is_filled) return;
    for (int i = 0; i < SIN_COS_N_COUNT; i++) {
        double theta = (2*M_PI*i)/SIN_COS_N_COUNT;
        sin_vals[i] = sinf(theta);
        cos_vals[i] = cosf(theta);
    }
    is_filled = true;
}

// 简单的离散傅立叶变换
// 输入为实数值
// 输出为复数值
static void dft(const std::vector<float> & in, std::vector<float> & out) {
    int N = in.size();

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

// Cooley-Tukey快速傅立叶变换
// 简易实现 - 使用更好的实现方式
// 输入为实数值
// 输出为复数值
static void fft(const std::vector<float> & in, std::vector<float> & out) {
    out.resize(in.size()*2);

    int N = in.size();
    // 如果输入大小为1，则直接将输入复制到输出，并在输出的下一个位置填充0
    if (N == 1) {
        out[0] = in[0];
        out[1] = 0;
        return;
    }

    // 如果输入大小为奇数，则直接进行离散傅立叶变换
    if (N%2 == 1) {
        dft(in, out);
        return;
    }

    // 创建存储偶数索引和奇数索引元素的向量
    std::vector<float> even;
    std::vector<float> odd;

    // 预留空间以提高性能
    even.reserve(N/2);
    odd.reserve(N/2);

    // 将输入数据分别存储到偶数索引和奇数索引的向量中
    for (int i = 0; i < N; i++) {
        if (i % 2 == 0) {
            even.push_back(in[i]);
        } else {
            odd.push_back(in[i]);
        }
    }

    // 创建存储偶数索引和奇数索引元素的傅立叶变换结果的向量
    std::vector<float> even_fft;
    std::vector<float> odd_fft;

    // 对偶数索引和奇数索引的数据进行傅立叶变换
    fft(even, even_fft);
    fft(odd, odd_fft);

    // 计算正弦和余弦值的步长
    const int sin_cos_step = SIN_COS_N_COUNT / N;
    // 对每个频率进行傅立叶变换计算
    for (int k = 0; k < N/2; k++) {
        int idx = k * sin_cos_step; // t = 2*M_PI*k/N
        float re = cos_vals[idx]; // cos(t)
        float im = -sin_vals[idx]; // sin(t)

        float re_odd = odd_fft[2*k + 0];
        float im_odd = odd_fft[2*k + 1];

        // 计算傅立叶变换结果的实部和虚部
        out[2*k + 0] = even_fft[2*k + 0] + re*re_odd - im*im_odd;
        out[2*k + 1] = even_fft[2*k + 1] + re*im_odd + im*re_odd;

        out[2*(k + N/2) + 0] = even_fft[2*k + 0] - re*re_odd + im*im_odd;
        out[2*(k + N/2) + 1] = even_fft[2*k + 1] - re*im_odd - im*re_odd;
    }
// 静态函数，生成汉宁窗口
static bool hann_window(int length, bool periodic, std::vector<float> & output) {
    // 如果输出向量大小小于指定长度，调整大小
    if (output.size() < static_cast<size_t>(length)) {
        output.resize(length);
    }
    // 初始化偏移量为-1
    int offset = -1;
    // 如果是周期性的，将偏移量设为0
    if (periodic) {
        offset = 0;
    }
    // 生成汉宁窗口
    for (int i = 0; i < length; i++) {
        output[i] = 0.5*(1.0 - cosf((2.0*M_PI*i)/(length + offset)));
    }

    return true;
}

// 静态函数，处理梅尔频谱的工作线程
static void log_mel_spectrogram_worker_thread(int ith, const std::vector<float> & hann, const std::vector<float> & samples,
                                              int n_samples, int frame_size, int frame_step, int n_threads,
                                              const whisper_filters & filters, whisper_mel & mel) {
    // 初始化FFT输入向量和输出向量
    std::vector<float> fft_in(frame_size, 0.0);
    std::vector<float> fft_out(2 * frame_step);
    // 确保n_fft == 1 + (WHISPER_N_FFT / 2)，表示从bin_0到bin_nyquist
    int n_fft = 1 + (frame_size / 2);
    int i = ith;

    // 仅当fft_in不全为零时计算FFT
    // 循环处理每个样本的帧
    for (; i < std::min(n_samples / frame_step + 1, mel.n_len); i += n_threads) {
        // 计算当前帧的偏移量
        const int offset = i * frame_step;

        // 应用汉宁窗口（比原始数据乘以汉宁窗口快约10%）
        for (int j = 0; j < std::min(frame_size, n_samples - offset); j++) {
            // 对当前帧数据应用汉宁窗口
            fft_in[j] = hann[j] * samples[offset + j];
        }
        // 如果当前帧数据不足一个帧的长度，用零填充
        if (n_samples - offset < frame_size) {
            std::fill(fft_in.begin() + (n_samples - offset), fft_in.end(), 0.0);
        }

        // 进行快速傅里叶变换（FFT）
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

            // 对结果取对数并确保不小于 1e-10
            sum = log10(std::max(sum, 1e-10));

            // 将结果存入梅尔频谱数据中
            mel.data[j * mel.n_len + i] = sum;
        }
    }

    // 否则 fft_out 全为零
    double sum = log10(1e-10);
    // 处理剩余的帧
    for (; i < mel.n_len; i += n_threads) {
        for (int j = 0; j < mel.n_mel; j++) {
            // 将结果存入梅尔频谱数据中
            mel.data[j * mel.n_len + i] = sum;
        }
    }
// 结束函数定义
}

// 引用链接：https://github.com/openai/whisper/blob/main/whisper/audio.py#L110-L157
// 计算对数梅尔频谱图
static bool log_mel_spectrogram(
              whisper_state & wstate,  // Whisper 状态
              const float * samples,   // 输入音频样本
              const int   n_samples,   // 音频样本数量
              const int   /*sample_rate*/,  // 采样率
              const int   frame_size,  // 帧大小
              const int   frame_step,  // 帧步长
              const int   n_mel,       // 梅尔频道数量
              const int   n_threads,   // 线程数量
              const whisper_filters & filters,  // 滤波器
              const bool   debug,      // 调试模式
              whisper_mel & mel) {     // 梅尔频谱图

    const int64_t t_start_us = ggml_time_us();  // 记录开始时间

    // 汉宁窗口（使用 cosf 消除差异）
    // 引用链接：https://pytorch.org/docs/stable/generated/torch.hann_window.html
    // 引用链接：https://github.com/openai/whisper/blob/main/whisper/audio.py#L147
    std::vector<float> hann;  // 汉宁窗口向量
    hann_window(frame_size, true, hann);  // 创建汉宁窗口

    // 计算填充长度
    int64_t stage_1_pad = WHISPER_SAMPLE_RATE * 30;  // 阶段1填充
    int64_t stage_2_pad = frame_size / 2;  // 阶段2填充

    // 初始化向量并从 C 数组复制数据
    std::vector<float> samples_padded;  // 填充后的音频样本向量
    samples_padded.resize(n_samples + stage_1_pad + stage_2_pad * 2);  // 调整大小
    std::copy(samples, samples + n_samples, samples_padded.begin() + stage_2_pad);  // 复制数据

    // 在音频末尾填充30秒的零（480,000个样本）+ 在音频末尾反射填充200个样本
    std::fill(samples_padded.begin() + n_samples + stage_2_pad, samples_padded.begin() + n_samples + stage_1_pad + 2 * stage_2_pad, 0);  // 填充零
    // 在音频开头反射填充200个样本
    std::reverse_copy(samples + 1, samples + 1 + stage_2_pad, samples_padded.begin());  // 反射填充

    mel.n_mel     = n_mel;  // 设置梅尔频道数量
    // 引用链接：https://github.com/pytorch/pytorch/blob/main/aten/src/ATen/native/SpectralOps.cpp#L936
    // 计算帧数 + 移除最后一帧
    mel.n_len     = (samples_padded.size() - frame_size) / frame_step;  // 计算帧数
    // 计算半填充样本长度以确保兼容性
    // 计算原始长度
    mel.n_len_org = 1 + (n_samples + stage_2_pad - frame_size) / frame_step;
    // 调整数据大小
    mel.data.resize(mel.n_mel * mel.n_len);

    // 创建线程池
    {
        std::vector<std::thread> workers(n_threads - 1);
        // 启动并发线程
        for (int iw = 0; iw < n_threads - 1; ++iw) {
            workers[iw] = std::thread(
                    log_mel_spectrogram_worker_thread, iw + 1, std::cref(hann), samples_padded,
                    n_samples + stage_2_pad, frame_size, frame_step, n_threads,
                    std::cref(filters), std::ref(mel));
        }

        // 主线程执行
        log_mel_spectrogram_worker_thread(0, hann, samples_padded, n_samples + stage_2_pad, frame_size, frame_step, n_threads, filters, mel);

        // 等待并发线程结束
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

    // 输出 log_mel_spectrogram 到文件
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
}

// 将文本拆分为标记
//
// 参考链接: https://github.com/openai/gpt-2/blob/a74da5d99abaaba920de8131d64da2862a8f213b/src/encoder.py#L53
//
// 正则表达式 (Python):
// r"""'s|'t|'re|'ve|'m|'ll|'d| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+"""
//
// 正则表达式 (C++):
// R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)"
//
static std::vector<whisper_vocab::id> tokenize(const whisper_vocab & vocab, const std::string & text) {
    std::vector<std::string> words;

    // 首先将文本拆分为单词
    {
        std::string str = text;
        std::string pat = R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)";

        std::regex re(pat);
        std::smatch m;

        while (std::regex_search(str, m, re)) {
            for (auto x : m) {
                words.push_back(x);
            }
            str = m.suffix();
        }
    }

    // 找到形成单词的最长标记:
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
// 将 .bin 替换为 -encoder.mlmodelc
static std::string whisper_get_coreml_path_encoder(std::string path_bin) {
    auto pos = path_bin.rfind('.');
    // 如果在路径中找到了 "-" 符号，则截取该符号之前的部分作为新的路径
    if (pos != std::string::npos) {
        path_bin = path_bin.substr(0, pos);
    }

    // 在路径中查找最后一个 "-" 符号的位置
    pos = path_bin.rfind('-');
    // 如果找到了 "-" 符号
    if (pos != std::string::npos) {
        // 截取从 "-" 符号开始到结尾的子字符串
        auto sub = path_bin.substr(pos);
        // 如果子字符串长度为 5，且第二个字符为 'q'，第四个字符为 '_'
        if (sub.size() == 5 && sub[1] == 'q' && sub[3] == '_') {
            // 截取 "-" 符号之前的部分作为新的路径
            path_bin = path_bin.substr(0, pos);
        }
    }

    // 在路径末尾添加 "-encoder.mlmodelc"
    path_bin += "-encoder.mlmodelc";

    // 返回处理后的路径
    return path_bin;
#ifdef WHISPER_USE_OPENVINO
// 如果定义了 WHISPER_USE_OPENVINO 宏，则编译以下代码块

// 根据给定的路径获取对应的编码器文件路径，将 .bin 替换为 -encoder-openvino.xml
static std::string whisper_openvino_get_path_encoder(std::string path_bin) {
    // 查找路径中最后一个 . 的位置
    auto pos = path_bin.rfind('.');
    // 如果找到了 . 的位置
    if (pos != std::string::npos) {
        // 截取 . 之前的部分作为新的路径
        path_bin = path_bin.substr(0, pos);
    }

    // 在路径末尾添加 -encoder-openvino.xml
    path_bin += "-encoder-openvino.xml";

    // 返回新的路径
    return path_bin;
}

// 根据给定的路径获取对应的缓存文件路径，将 .bin 替换为 -encoder-openvino-cache
static std::string whisper_openvino_get_path_cache(std::string path_bin) {
    // 查找路径中最后一个 . 的位置
    auto pos = path_bin.rfind('.');
    // 如果找到了 . 的位置
    if (pos != std::string::npos) {
        // 截取 . 之前的部分作为新的路径
        path_bin = path_bin.substr(0, pos);
    }

    // 在路径末尾添加 -encoder-openvino-cache
    path_bin += "-encoder-openvino-cache";

    // 返回新的路径
    return path_bin;
}
#endif

// 初始化 whisper_state 结构体
struct whisper_state * whisper_init_state(whisper_context * ctx) {
    // 填充正弦余弦表
    fill_sin_cos_table();

    // 创建 whisper_state 结构体对象
    whisper_state * state = new whisper_state;

    // 初始化 whisper_state 结构体中的 backend 成员
    state->backend = whisper_backend_init(ctx->params);

    // 在这一点上，我们还不知道将使用多少个解码器，因此我们过度分配 3 倍的 ctx
    // 理论上，可能会有一种情况不够用，但实际上应该总是足够的
    const int factor = 3;

    // 初始化 self-attention 缓存
    if (!kv_cache_init(ctx->model.hparams, state->kv_self, ctx->backend, ctx->itype, factor*ctx->model.hparams.n_text_ctx)) {
        WHISPER_LOG_ERROR("%s: kv_cache_init() failed for self-attention cache\n", __func__);
        delete state;
        return nullptr;
    }

    {
        // 计算 self-attention 缓存的内存大小
        const size_t memory_size = ggml_nbytes(state->kv_self.k) + ggml_nbytes(state->kv_self.v);
        WHISPER_LOG_INFO("%s: kv self size  = %7.2f MB\n", __func__, memory_size / 1e6);
    }

    // 初始化 cross-attention 缓存
    if (!kv_cache_init(ctx->model.hparams, state->kv_cross, ctx->backend, ctx->itype, ctx->model.hparams.n_audio_ctx)) {
        WHISPER_LOG_ERROR("%s: kv_cache_init() failed for cross-attention cache\n", __func__);
        delete state;
        return nullptr;
    }

    {
        // 计算 cross-attention 缓存的内存大小
        const size_t memory_size = ggml_nbytes(state->kv_cross.k) + ggml_nbytes(state->kv_cross.v);
        WHISPER_LOG_INFO("%s: kv cross size = %7.2f MB\n", __func__, memory_size / 1e6);
    }
}
#ifdef WHISPER_USE_COREML
    // 如果定义了 WHISPER_USE_COREML 宏，则执行以下代码块
    const auto path_coreml = whisper_get_coreml_path_encoder(ctx->path_model);

    // 输出加载 Core ML 模型的信息
    WHISPER_LOG_INFO("%s: loading Core ML model from '%s'\n", __func__, path_coreml.c_str());
    WHISPER_LOG_INFO("%s: first run on a device may take a while ...\n", __func__);

    // 初始化 Core ML 模型
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
        // 输出 Core ML 模型加载成功的信息
        WHISPER_LOG_INFO("%s: Core ML model loaded\n", __func__);
    }
#endif

    // 预留 logits 的空间
    state->logits.reserve(ctx->vocab.n_vocab * ctx->model.hparams.n_text_ctx);

    // 初始化批处理对象
    state->batch = whisper_batch_init(ctx->model.hparams.n_text_ctx, WHISPER_MAX_DECODERS);

    // 初始化第一个解码器的 tokens 空间
    // TAGS: WHISPER_DECODER_INIT
    state->decoders[0].sequence.tokens.reserve(ctx->model.hparams.n_text_ctx);

    // 预留第一个解码器的 probs、logits、logprobs、logits_id 的空间
    state->decoders[0].probs.reserve    (ctx->vocab.n_vocab);
    state->decoders[0].logits.reserve   (ctx->vocab.n_vocab);
    state->decoders[0].logprobs.reserve (ctx->vocab.n_vocab);
    state->decoders[0].logits_id.reserve(ctx->model.hparams.n_vocab);

    // 初始化第一个解码器的随机数生成器
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
        // 初始化交叉分配器的图形
        whisper_allocr_graph_init(state->alloc_cross, ctx->backend,
                [&]() {
                    // 返回交叉构建的图形
                    return whisper_build_graph_cross(*ctx, *state);
                });

        // 记录日志信息，显示计算缓冲区大小（交叉）
        WHISPER_LOG_INFO("%s: compute buffer (cross)  = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_cross) / 1e6);
    }

    // 解码器分配器
    {
        // 初始化解码器分配器的图形
        whisper_allocr_graph_init(state->alloc_decode, ctx->backend,
                [&]() {
                    const auto & hparams = ctx->model.hparams;

                    // TODO: 确保这是最坏情况
                    const int n_tokens = hparams.n_text_ctx;
                    const int n_past   = 0;

                    // 准备传统批次
                    whisper_batch_prep_legacy(state->batch, nullptr, n_tokens, n_past, 0);

                    // 返回解码器构建的图形
                    return whisper_build_graph_decoder(*ctx, *state, state->batch);
                });

        // 记录日志信息，显示计算缓冲区大小（解码）
        WHISPER_LOG_INFO("%s: compute buffer (decode) = %7.2f MB\n", __func__, whisper_allocr_size(state->alloc_decode) / 1e6);
    }

    // 重新分配卷积分配器的图形
    whisper_allocr_graph_realloc(state->alloc_conv,   ctx->backend);
    // 重新分配编码分配器的图形
    whisper_allocr_graph_realloc(state->alloc_encode, ctx->backend);
    // 重新分配交叉分配器的图形
    whisper_allocr_graph_realloc(state->alloc_cross,  ctx->backend);
    // 重新分配解码器分配器的图形
    whisper_allocr_graph_realloc(state->alloc_decode, ctx->backend);

    // 返回状态
    return state;
// 初始化 OpenVINO 编码器的上下文
int whisper_ctx_init_openvino_encoder(
        struct whisper_context * ctx,
                    const char * model_path,
                    const char * device,
                    const char * cache_dir) {
#ifndef WHISPER_USE_OPENVINO
    // 如果未启用 OpenVINO，则返回 1
    (void)(ctx);
    (void)(model_path);
    (void)(device);
    (void)(cache_dir);

    return 1;
#else
    // 如果 model_path 为空且上下文中没有设置 model_path，则返回错误
    if (!model_path && ctx->path_model.empty()) {
        WHISPER_LOG_ERROR("%s: model_path is nullptr, and ctx has no model_path set.\n", __func__);
        return 1;
    }

    std::string path_encoder;
    if (!model_path) {
        // 如果 model_path 未设置，则尝试在与 ggml-<model>.bin 模型相同的目录中查找
        path_encoder = whisper_openvino_get_path_encoder(ctx->path_model);
    } else {
        path_encoder = model_path;
    }

    std::string path_cache;
    if (!cache_dir) {
        // 如果 cache_dir 未设置，则设置为与 ggml-<model>.bin 相邻的目录
        path_cache = whisper_openvino_get_path_cache(ctx->path_model);
    } else {
        path_cache = cache_dir;
    }

    WHISPER_LOG_INFO("%s: loading OpenVINO model from '%s'\n", __func__, path_encoder.c_str());
    WHISPER_LOG_INFO("%s: first run on a device may take a while ...\n", __func__);

    // 初始化 OpenVINO 编码器
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

// 返回默认的上下文参数
struct whisper_context_params whisper_context_default_params() {
    struct whisper_context_params result = {
        /*.use_gpu    =*/ true,
    };
    return result;
}

// 从文件初始化上下文，使用指定参数，不包含状态
struct whisper_context * whisper_init_from_file_with_params_no_state(const char * path_model, struct whisper_context_params params) {
    // 使用 WHISPER_LOG_INFO 打印加载模型的信息，包括函数名和模型路径
    WHISPER_LOG_INFO("%s: loading model from '%s'\n", __func__, path_model);

    // 打开二进制文件流，读取模型数据
    auto fin = std::ifstream(path_model, std::ios::binary);
    // 如果文件流打开失败，记录错误信息并返回空指针
    if (!fin) {
        WHISPER_LOG_ERROR("%s: failed to open '%s'\n", __func__, path_model);
        return nullptr;
    }

    // 创建一个模型加载器对象
    whisper_model_loader loader = {};

    // 将文件流指针赋值给加载器对象的上下文
    loader.context = &fin;

    // 定义读取数据的函数，从文件流中读取指定大小的数据
    loader.read = [](void * ctx, void * output, size_t read_size) {
        std::ifstream * fin = (std::ifstream*)ctx;
        fin->read((char *)output, read_size);
        return read_size;
    };

    // 定义判断文件流是否结束的函数
    loader.eof = [](void * ctx) {
        std::ifstream * fin = (std::ifstream*)ctx;
        return fin->eof();
    };

    // 定义关闭文件流的函数
    loader.close = [](void * ctx) {
        std::ifstream * fin = (std::ifstream*)ctx;
        fin->close();
    };

    // 使用加载器对象和参数初始化模型上下文
    auto ctx = whisper_init_with_params_no_state(&loader, params);

    // 如果成功初始化模型上下文，设置模型路径
    if (ctx) {
        ctx->path_model = path_model;
    }

    // 返回模型上下文
    return ctx;
}

// 从内存缓冲区初始化 whisper_context 结构体，不包含状态信息
struct whisper_context * whisper_init_from_buffer_with_params_no_state(void * buffer, size_t buffer_size, struct whisper_context_params params) {
    // 定义内部缓冲区结构体
    struct buf_context {
        uint8_t* buffer;
        size_t size;
        size_t current_offset;
    };

    // 初始化内部缓冲区结构体
    buf_context ctx = { reinterpret_cast<uint8_t*>(buffer), buffer_size, 0 };

    // 打印日志信息
    WHISPER_LOG_INFO("%s: loading model from buffer\n", __func__);

    // 初始化 whisper_model_loader 结构体
    whisper_model_loader loader = {};

    // 设置 loader 的上下文为内部缓冲区结构体
    loader.context = &ctx;

    // 定义读取函数
    loader.read = [](void * ctx, void * output, size_t read_size) {
        buf_context * buf = reinterpret_cast<buf_context *>(ctx);

        // 计算需要拷贝的数据大小
        size_t size_to_copy = buf->current_offset + read_size < buf->size ? read_size : buf->size - buf->current_offset;

        // 拷贝数据到输出
        memcpy(output, buf->buffer + buf->current_offset, size_to_copy);
        buf->current_offset += size_to_copy;

        return size_to_copy;
    };

    // 定义判断是否到达文件末尾的函数
    loader.eof = [](void * ctx) {
        buf_context * buf = reinterpret_cast<buf_context *>(ctx);

        return buf->current_offset >= buf->size;
    };

    // 定义关闭函数
    loader.close = [](void * /*ctx*/) { };

    // 使用 loader 和参数初始化 whisper_context 结构体
    return whisper_init_with_params_no_state(&loader, params);
}

// 从文件初始化 whisper_context 结构体，不包含状态信息
struct whisper_context * whisper_init_with_params_no_state(struct whisper_model_loader * loader, struct whisper_context_params params) {
    // 初始化时间
    ggml_time_init();

    // 创建 whisper_context 结构体
    whisper_context * ctx = new whisper_context;
    ctx->params = params;

    // 如果加载模型失败，则释放资源并返回空指针
    if (!whisper_model_load(loader, *ctx)) {
        loader->close(loader->context);
        WHISPER_LOG_ERROR("%s: failed to load model\n", __func__);
        delete ctx;
        return nullptr;
    }

    // 关闭 loader
    loader->close(loader->context);

    return ctx;
}

// 从文件初始化 whisper_context 结构体，包含状态信息
struct whisper_context * whisper_init_from_file_with_params(const char * path_model, struct whisper_context_params params) {
    // 从文件初始化 whisper_context 结构体，不包含状态信息
    whisper_context * ctx = whisper_init_from_file_with_params_no_state(path_model, params);
    if (!ctx) {
        return nullptr;
    }

    // 初始化状态信息
    ctx->state = whisper_init_state(ctx);
    // 如果上下文状态为空，则释放上下文内存并返回空指针
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }

    // 返回上下文指针
    return ctx;
// 从缓冲区和参数初始化 whisper_context 结构体，返回指向该结构体的指针
struct whisper_context * whisper_init_from_buffer_with_params(void * buffer, size_t buffer_size, struct whisper_context_params params) {
    // 调用 whisper_init_from_buffer_with_params_no_state 函数初始化 whisper_context 结构体，不包含状态信息
    whisper_context * ctx = whisper_init_from_buffer_with_params_no_state(buffer, buffer_size, params);
    // 如果初始化失败，返回空指针
    if (!ctx) {
        return nullptr;
    }

    // 初始化 whisper_context 结构体的状态信息
    ctx->state = whisper_init_state(ctx);
    // 如果状态信息初始化失败，释放 whisper_context 结构体内存，返回空指针
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }

    // 返回初始化后的 whisper_context 结构体指针
    return ctx;
}

// 从加载器和参数初始化 whisper_context 结构体，返回指向该结构体的指针
struct whisper_context * whisper_init_with_params(struct whisper_model_loader * loader, struct whisper_context_params params) {
    // 调用 whisper_init_with_params_no_state 函数初始化 whisper_context 结构体，不包含状态信息
    whisper_context * ctx = whisper_init_with_params_no_state(loader, params);
    // 如果初始化失败，返回空指针
    if (!ctx) {
        return nullptr;
    }

    // 初始化 whisper_context 结构体的状态信息
    ctx->state = whisper_init_state(ctx);
    // 如果状态信息初始化失败，释放 whisper_context 结构体内存，返回空指针
    if (!ctx->state) {
        whisper_free(ctx);
        return nullptr;
    }

    // 返回初始化后的 whisper_context 结构体指针
    return ctx;
}

// 从文件路径初始化 whisper_context 结构体，返回指向该结构体的指针
struct whisper_context * whisper_init_from_file(const char * path_model) {
    // 调用 whisper_init_from_file_with_params 函数，使用默认参数初始化 whisper_context 结构体
    return whisper_init_from_file_with_params(path_model, whisper_context_default_params());
}

// 从缓冲区初始化 whisper_context 结构体，返回指向该结构体的指针
struct whisper_context * whisper_init_from_buffer(void * buffer, size_t buffer_size) {
    // 调用 whisper_init_from_buffer_with_params 函数，使用默认参数初始化 whisper_context 结构体
    return whisper_init_from_buffer_with_params(buffer, buffer_size, whisper_context_default_params());
}

// 从加载器初始化 whisper_context 结构体，返回指向该结构体的指针
struct whisper_context * whisper_init(struct whisper_model_loader * loader) {
    // 调用 whisper_init_with_params 函数，使用默认参数初始化 whisper_context 结构体
    return whisper_init_with_params(loader, whisper_context_default_params());
}

// 从文件路径初始化 whisper_context 结构体，不包含状态信息，返回指向该结构体的指针
struct whisper_context * whisper_init_from_file_no_state(const char * path_model) {
    // 调用 whisper_init_from_file_with_params_no_state 函数，使用默认参数初始化 whisper_context 结构体，不包含状态信息
    return whisper_init_from_file_with_params_no_state(path_model, whisper_context_default_params());
}

// 从缓冲区初始化 whisper_context 结构体，不包含状态信息，返回指向该结构体的指针
struct whisper_context * whisper_init_from_buffer_no_state(void * buffer, size_t buffer_size) {
    // 调用 whisper_init_from_buffer_with_params_no_state 函数，使用默认参数初始化 whisper_context 结构体，不包含状态信息
    return whisper_init_from_buffer_with_params_no_state(buffer, buffer_size, whisper_context_default_params());
}

// 从加载器初始化 whisper_context 结构体，不包含状态信息，返回指向该结构体的指针
struct whisper_context * whisper_init_no_state(struct whisper_model_loader * loader) {
    // 调用 whisper_init_with_params_no_state 函数，使用默认参数初始化 whisper_context 结构体，不包含状态信息
    return whisper_init_with_params_no_state(loader, whisper_context_default_params());
}

// 释放 whisper_state 结构体的内存
void whisper_free_state(struct whisper_state * state)
    # 如果状态存在（非空），则释放状态中的自身键值缓存和交叉键值缓存
    if (state) {
        kv_cache_free(state->kv_self);
        kv_cache_free(state->kv_cross);
#ifdef WHISPER_USE_COREML
        // 如果使用 CoreML，释放 CoreML 上下文
        if (state->ctx_coreml != nullptr) {
            whisper_coreml_free(state->ctx_coreml);
            state->ctx_coreml = nullptr;
        }
#endif

#ifdef WHISPER_USE_OPENVINO
        // 如果使用 OpenVINO，释放 OpenVINO 上下文
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

        // 删除状态对象
        delete state;
    }
}

// 释放 Whisper 上下文
void whisper_free(struct whisper_context * ctx) {
    if (ctx) {
        // 释放模型上下文
        if (ctx->model.ctx) {
            ggml_free(ctx->model.ctx);
        }

        // 释放模型缓冲区
        for (auto & buffer : ctx->model.buffers) {
            if (buffer) {
                ggml_backend_buffer_free(buffer);
            }
        }

        // 释放状态对象
        whisper_free_state(ctx->state);

        // 释放后端对象
        ggml_backend_free(ctx->backend);

        // 删除 Whisper 上下文
        delete ctx;
    }
}

// 释放 Whisper 上下文参数
void whisper_free_context_params(struct whisper_context_params * params) {
    if (params) {
        delete params;
    }
}

// 释放 Whisper 完整参数
void whisper_free_params(struct whisper_full_params * params) {
    if (params) {
        delete params;
    }
}

// 将 PCM 转换为 Mel 频谱，使用给定状态对象
int whisper_pcm_to_mel_with_state(struct whisper_context * ctx, struct whisper_state * state, const float * samples, int n_samples, int n_threads) {
    // 计算 Mel 频谱
    if (!log_mel_spectrogram(*state, samples, n_samples, WHISPER_SAMPLE_RATE, WHISPER_N_FFT, WHISPER_HOP_LENGTH, ctx->model.filters.n_mel, n_threads, ctx->model.filters, false, state->mel)) {
        // 如果计算失败，记录错误信息并返回错误码
        WHISPER_LOG_ERROR("%s: failed to compute mel spectrogram\n", __func__);
        return -1;
    }

    return 0;
}

// 将 PCM 转换为 Mel 频谱
int whisper_pcm_to_mel(struct whisper_context * ctx, const float * samples, int n_samples, int n_threads) {
    // 使用给定状态对象进行 PCM 到 Mel 频谱转换
    return whisper_pcm_to_mel_with_state(ctx, ctx->state, samples, n_samples, n_threads);
// 结束 whisper_pcm_to_mel 函数定义

// 使用相位估计器对音频进行加速处理，速度提高为原来的两倍（不使用相位锁定的相位估计器效果不佳）
int whisper_pcm_to_mel_phase_vocoder_with_state(struct whisper_context * ctx, struct whisper_state * state, const float * samples, int n_samples, int n_threads) {
    // 如果无法计算梅尔频谱，则输出错误信息并返回-1
    if (!log_mel_spectrogram(*state, samples, n_samples, WHISPER_SAMPLE_RATE, 2 * WHISPER_N_FFT, 2 * WHISPER_HOP_LENGTH, ctx->model.filters.n_mel, n_threads, ctx->model.filters, false, state->mel)) {
        WHISPER_LOG_ERROR("%s: failed to compute mel spectrogram\n", __func__);
        return -1;
    }

    // 返回成功
    return 0;
}

// 使用相位估计器对音频进行加速处理，速度提高为原来的两倍（不使用相位锁定的相位估计器效果不佳）
int whisper_pcm_to_mel_phase_vocoder(struct whisper_context * ctx, const float * samples, int n_samples, int n_threads) {
    // 调用 whisper_pcm_to_mel_phase_vocoder_with_state 函数，传入上下文、状态、音频样本和线程数
    return whisper_pcm_to_mel_phase_vocoder_with_state(ctx, ctx->state, samples, n_samples, n_threads);
}

// 使用 WSOLA 对音频进行加速处理，速度提高为原来的两倍
// 待实现

// 使用 HPTSM 对音频进行加速处理，速度提高为原来的两倍
// 待实现

// 使用 PV（带相位锁定）对音频进行加速处理，速度提高为原来的两倍
// 待实现

// 设置状态中的梅尔频谱数据
int whisper_set_mel_with_state(
        struct whisper_context * ctx,
          struct whisper_state * state,
                   const float * data,
                           int   n_len,
                           int   n_mel) {
    // 如果梅尔频带数量与上下文中的不匹配，则输出错误信息并返回-1
    if (n_mel != ctx->model.filters.n_mel) {
        WHISPER_LOG_ERROR("%s: invalid number of mel bands: %d (expected %d)\n", __func__, n_mel, ctx->model.filters.n_mel);
        return -1;
    }

    // 设置状态中的梅尔频谱数据
    state->mel.n_len     = n_len;
    state->mel.n_len_org = n_len;
    state->mel.n_mel     = n_mel;

    // 调整状态中的梅尔频谱数据大小，并拷贝数据
    state->mel.data.resize(n_len*n_mel);
    memcpy(state->mel.data.data(), data, n_len*n_mel*sizeof(float));

    // 返回成功
    return 0;
}
// 设置 MEL 参数并调用 whisper_set_mel_with_state 函数
int whisper_set_mel(
        struct whisper_context * ctx,
        const float * data,
        int n_len,
        int n_mel) {
    return whisper_set_mel_with_state(ctx, ctx->state, data, n_len, n_mel);
}

// 使用给定的状态结构体进行编码
int whisper_encode_with_state(struct whisper_context * ctx, struct whisper_state * state, int offset, int n_threads) {
    // 如果编码失败，则记录错误信息并返回 -1
    if (!whisper_encode_internal(*ctx, *state, offset, n_threads, nullptr, nullptr)) {
        WHISPER_LOG_ERROR("%s: failed to eval\n", __func__);
        return -1;
    }

    return 0;
}

// 对当前状态进行编码
int whisper_encode(struct whisper_context * ctx, int offset, int n_threads) {
    // 如果编码失败，则记录错误信息并返回 -1
    if (!whisper_encode_internal(*ctx, *ctx->state, offset, n_threads, nullptr, nullptr)) {
        WHISPER_LOG_ERROR("%s: failed to eval\n", __func__);
        return -1;
    }

    return 0;
}

// 使用给定的状态结构体进行解码
int whisper_decode_with_state(struct whisper_context * ctx, struct whisper_state * state, const whisper_token * tokens, int n_tokens, int n_past, int n_threads) {
    // 准备批处理数据
    whisper_batch_prep_legacy(state->batch, tokens, n_tokens, n_past, 0);

    // 从缓存中移除键值对序列
    whisper_kv_cache_seq_rm(state->kv_self, 0, n_past, -1);

    // 如果解码失败，则记录错误信息并返回 1
    if (!whisper_decode_internal(*ctx, *state, state->batch, n_threads, nullptr, nullptr)) {
        WHISPER_LOG_ERROR("%s: failed to eval\n", __func__);
        return 1;
    }

    return 0;
}

// 对当前状态进行解码
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
    // 调用 tokenize 函数对文本进行标记化
    const auto res = tokenize(ctx->vocab, text);

    // 如果结果标记数超过最大标记数，则记录错误信息并返回 -1
    if (n_max_tokens < (int) res.size()) {
        WHISPER_LOG_ERROR("%s: too many resulting tokens: %d (max %d)\n", __func__, (int) res.size(), n_max_tokens);
        return -1;
    }
}
    # 遍历 res 列表中的元素，将每个元素赋值给 tokens 列表对应位置的元素
    for (int i = 0; i < (int) res.size(); i++) {
        tokens[i] = res[i];
    }

    # 返回 res 列表的大小
    return res.size();
}

// 返回语言映射表中最大的语言ID
int whisper_lang_max_id() {
    // 初始化最大ID为0
    auto max_id = 0;
    // 遍历语言映射表，更新最大ID
    for (const auto & kv : g_lang) {
        max_id = std::max(max_id, kv.second.first);
    }

    return max_id;
}

// 根据语言名称获取对应的语言ID
int whisper_lang_id(const char * lang) {
    // 如果语言映射表中不存在该语言，则遍历映射表查找对应语言ID
    if (!g_lang.count(lang)) {
        for (const auto & kv : g_lang) {
            if (kv.second.second == lang) {
                return kv.second.first;
            }
        }

        // 输出错误日志并返回-1
        WHISPER_LOG_ERROR("%s: unknown language '%s'\n", __func__, lang);
        return -1;
    }
    // 返回对应语言的ID
    return g_lang.at(lang).first;
}

// 根据语言ID获取对应的语言名称
const char * whisper_lang_str(int id) {
    for (const auto & kv : g_lang) {
        if (kv.second.first == id) {
            return kv.first.c_str();
        }
    }

    // 输出错误日志并返回空指针
    WHISPER_LOG_ERROR("%s: unknown language id %d\n", __func__, id);
    return nullptr;
}

// 根据语言ID获取对应的完整语言名称
const char * whisper_lang_str_full(int id) {
   for (const auto & kv : g_lang) {
        if (kv.second.first == id) {
            return kv.second.second.c_str();
        }
    }

    // 输出错误日志并返回空指针
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

    // 如果偏移量小于0，输出错误日志并返回-1
    if (seek < 0) {
        WHISPER_LOG_ERROR("%s: offset %dms is before the start of the audio\n", __func__, offset_ms);
        return -1;
    }

    // 如果偏移量大于等于音频结束时间，输出错误日志并返回-2
    if (seek >= state->mel.n_len_org) {
        WHISPER_LOG_ERROR("%s: offset %dms is past the end of the audio (%dms)\n", __func__, offset_ms, state->mel.n_len_org*10);
        return -2;
    }

    // 运行编码器
    if (whisper_encode_with_state(ctx, state, seek, n_threads) != 0) {
        WHISPER_LOG_ERROR("%s: failed to encode\n", __func__);
        return -6;
    }

    // 初始化提示符
    const std::vector<whisper_token> prompt = { whisper_token_sot(ctx) };
    // 如果使用给定的上下文、状态、提示数据和大小进行解码时出错，则记录错误信息并返回错误代码
    if (whisper_decode_with_state(ctx, state, prompt.data(), prompt.size(), 0, n_threads) != 0) {
        WHISPER_LOG_ERROR("%s: failed to decode\n", __func__);
        return -7;
    }

    // 获取状态对象中第一个解码器的logits_id，并清空其中的内容
    auto & logits_id = state->decoders[0].logits_id;
    logits_id.clear();

    // 遍历全局语言映射表g_lang，将每个语言对应的logits值和语言ID添加到logits_id中
    for (const auto & kv : g_lang) {
        const auto token_lang = whisper_token_lang(ctx, kv.second.first);
        logits_id.emplace_back(state->logits[token_lang], kv.second.first);
    }

    // 按照logits值降序排序logits_id中的元素
    {
        using pair_type = std::remove_reference<decltype(logits_id)>::type::value_type;
        std::sort(logits_id.begin(), logits_id.end(), [](const pair_type & a, const pair_type & b) {
            return a.first > b.first;
        });
    }

    // 对logits_id中的logits值进行softmax处理
    {
        const auto max = logits_id[0].first;

        double sum = 0.0f;
        for (auto & kv : logits_id) {
            kv.first = exp(kv.first - max);
            sum += kv.first;
        }

        for (auto & kv : logits_id) {
            kv.first /= sum;
        }
    }

    // 将softmax处理后的概率值存储到lang_probs中，并打印每个语言的概率值
    {
        for (const auto & prob : logits_id) {
            if (lang_probs) {
                lang_probs[prob.second] = prob.first;
            }

            //printf("%s: lang %2d (%3s): %f\n", __func__, prob.second, whisper_lang_str(prob.second), prob.first);
        }
    }

    // 返回概率值最大的语言ID
    return logits_id[0].second;
# 返回 whisper_lang_auto_detect_with_state 函数的结果
int whisper_lang_auto_detect(
        struct whisper_context * ctx,
                           int   offset_ms,
                           int   n_threads,
                         float * lang_probs) {
    return whisper_lang_auto_detect_with_state(ctx, ctx->state, offset_ms, n_threads, lang_probs);
}

# 返回模型的词汇量
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

# 返回模型的梅尔频谱数量
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
# 从 whisper_state 结构体中获取 mel.n_len_org 的值并返回
int whisper_n_len_from_state(struct whisper_state * state) {
    return state->mel.n_len_org;
}

# 从 whisper_context 结构体中获取 state->mel.n_len_org 的值并返回
int whisper_n_len(struct whisper_context * ctx) {
    return ctx->state->mel.n_len_org;
}

# 从 whisper_context 结构体中获取 vocab.n_vocab 的值并返回
int whisper_n_vocab(struct whisper_context * ctx) {
    return ctx->vocab.n_vocab;
}

# 从 whisper_context 结构体中获取 model.hparams.n_text_ctx 的值并返回
int whisper_n_text_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_text_ctx;
}

# 从 whisper_context 结构体中获取 model.hparams.n_audio_ctx 的值并返回
int whisper_n_audio_ctx(struct whisper_context * ctx) {
    return ctx->model.hparams.n_audio_ctx;
}

# 从 whisper_context 结构体中获取 vocab.is_multilingual() 的值，如果为真返回 1，否则返回 0
int whisper_is_multilingual(struct whisper_context * ctx) {
    return ctx->vocab.is_multilingual() ? 1 : 0;
}

# 从 whisper_context 结构体中获取 state->logits.data() 的值并返回
float * whisper_get_logits(struct whisper_context * ctx) {
    return ctx->state->logits.data();
}

# 从 whisper_state 结构体中获取 logits.data() 的值并返回
float * whisper_get_logits_from_state(struct whisper_state * state) {
    return state->logits.data();
}

# 根据 whisper_token 和 whisper_context 结构体中的 vocab.id_to_token 获取对应的字符串并返回
const char * whisper_token_to_str(struct whisper_context * ctx, whisper_token token) {
    return ctx->vocab.id_to_token.at(token).c_str();
}

# 返回 whisper_context 结构体中的 vocab.token_eot 的值
whisper_token whisper_token_eot(struct whisper_context * ctx) {
    return ctx->vocab.token_eot;
}

# 返回 whisper_context 结构体中的 vocab.token_sot 的值
whisper_token whisper_token_sot(struct whisper_context * ctx) {
    return ctx->vocab.token_sot;
}

# 返回 whisper_context 结构体中的 vocab.token_solm 的值
whisper_token whisper_token_solm(struct whisper_context * ctx) {
    return ctx->vocab.token_solm;
}

# 返回 whisper_context 结构体中的 vocab.token_prev 的值
whisper_token whisper_token_prev(struct whisper_context * ctx) {
    return ctx->vocab.token_prev;
}

# 返回 whisper_context 结构体中的 vocab.token_nosp 的值
whisper_token whisper_token_nosp(struct whisper_context * ctx) {
    return ctx->vocab.token_nosp;
}

# 返回 whisper_context 结构体中的 vocab.token_not 的值
whisper_token whisper_token_not(struct whisper_context * ctx) {
    return ctx->vocab.token_not;
}

# 返回 whisper_context 结构体中的 vocab.token_beg 的值
whisper_token whisper_token_beg(struct whisper_context * ctx) {
    return ctx->vocab.token_beg;
}

# 根据 lang_id 返回对应的语言 token
whisper_token whisper_token_lang(struct whisper_context * ctx, int lang_id) {
    return whisper_token_sot(ctx) + 1 + lang_id;
}

# 返回 whisper_context 结构体中的 vocab.token_translate 的值
whisper_token whisper_token_translate(struct whisper_context * ctx) {
    return ctx->vocab.token_translate;
}

# 返回 whisper_context 结构体中的 vocab.token_transcribe 的值
whisper_token whisper_token_transcribe(struct whisper_context * ctx) {
    return ctx->vocab.token_transcribe;
}
// 打印程序运行时间信息
void whisper_print_timings(struct whisper_context * ctx) {
    // 获取当前时间
    const int64_t t_end_us = ggml_time_us();

    // 打印空行
    WHISPER_LOG_INFO("\n");
    // 打印加载时间信息
    WHISPER_LOG_INFO("%s:     load time = %8.2f ms\n", __func__, ctx->t_load_us / 1000.0f);
    // 如果状态不为空
    if (ctx->state != nullptr) {

        // 获取样本数、编码次数、解码次数、批处理次数、提示次数
        const int32_t n_sample = std::max(1, ctx->state->n_sample);
        const int32_t n_encode = std::max(1, ctx->state->n_encode);
        const int32_t n_decode = std::max(1, ctx->state->n_decode);
        const int32_t n_batchd = std::max(1, ctx->state->n_batchd);
        const int32_t n_prompt = std::max(1, ctx->state->n_prompt);

        // 打印回退次数信息
        WHISPER_LOG_INFO("%s:     fallbacks = %3d p / %3d h\n", __func__, ctx->state->n_fail_p, ctx->state->n_fail_h);
        // 打印 mel 时间信息
        WHISPER_LOG_INFO("%s:      mel time = %8.2f ms\n", __func__, ctx->state->t_mel_us / 1000.0f);
        // 打印样本时间信息
        WHISPER_LOG_INFO("%s:   sample time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_sample_us, n_sample, 1e-3f * ctx->state->t_sample_us / n_sample);
        // 打印编码时间信息
        WHISPER_LOG_INFO("%s:   encode time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_encode_us, n_encode, 1e-3f * ctx->state->t_encode_us / n_encode);
        // 打印解码时间信息
        WHISPER_LOG_INFO("%s:   decode time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_decode_us, n_decode, 1e-3f * ctx->state->t_decode_us / n_decode);
        // 打印批处理时间信息
        WHISPER_LOG_INFO("%s:   batchd time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_batchd_us, n_batchd, 1e-3f * ctx->state->t_batchd_us / n_batchd);
        // 打印提示时间信息
        WHISPER_LOG_INFO("%s:   prompt time = %8.2f ms / %5d runs (%8.2f ms per run)\n", __func__, 1e-3f * ctx->state->t_prompt_us, n_prompt, 1e-3f * ctx->state->t_prompt_us / n_prompt);
    }
    // 打印总时间信息
    WHISPER_LOG_INFO("%s:    total time = %8.2f ms\n", __func__, (t_end_us - ctx->t_start_us)/1000.0f);
}

// 重置计时器
void whisper_reset_timings(struct whisper_context * ctx) {
    // 设置起始时间
    ctx->t_start_us = ggml_time_us();
}
    // 如果上下文状态不为空
    if (ctx->state != nullptr) {
        // 重置各时间统计字段为0
        ctx->state->t_mel_us = 0;
        ctx->state->t_sample_us = 0;
        ctx->state->t_encode_us = 0;
        ctx->state->t_decode_us = 0;
        ctx->state->t_batchd_us = 0;
        ctx->state->t_prompt_us = 0;
        // 重置各计数字段为0
        ctx->state->n_sample = 0;
        ctx->state->n_encode = 0;
        ctx->state->n_decode = 0;
        ctx->state->n_batchd = 0;
        ctx->state->n_prompt = 0;
    }
// 检查是否定义了 WHISPER_USE_COREML 宏，如果定义了返回1，否则返回0
static int whisper_has_coreml(void) {
#ifdef WHISPER_USE_COREML
    return 1;
#else
    return 0;
#endif
}

// 检查是否定义了 WHISPER_USE_OPENVINO 宏，如果定义了返回1，否则返回0
static int whisper_has_openvino(void) {
#ifdef WHISPER_USE_OPENVINO
    return 1;
#else
    return 0;
#endif
}

// 打印系统信息，包括各种 CPU 特性和库支持情况
const char * whisper_print_system_info(void) {
    static std::string s;

    // 初始化字符串 s
    s  = "";
    // 拼接 AVX 特性信息
    s += "AVX = "       + std::to_string(ggml_cpu_has_avx())       + " | ";
    // 拼接 AVX2 特性信息
    s += "AVX2 = "      + std::to_string(ggml_cpu_has_avx2())      + " | ";
    // 拼接 AVX512 特性信息
    s += "AVX512 = "    + std::to_string(ggml_cpu_has_avx512())    + " | ";
    // 拼接 FMA 特性信息
    s += "FMA = "       + std::to_string(ggml_cpu_has_fma())       + " | ";
    // 拼接 NEON 特性信息
    s += "NEON = "      + std::to_string(ggml_cpu_has_neon())      + " | ";
    // 拼接 ARM_FMA 特性信息
    s += "ARM_FMA = "   + std::to_string(ggml_cpu_has_arm_fma())   + " | ";
    // 拼接 METAL 特性信息
    s += "METAL = "     + std::to_string(ggml_cpu_has_metal())     + " | ";
    // 拼接 F16C 特性信息
    s += "F16C = "      + std::to_string(ggml_cpu_has_f16c())      + " | ";
    // 拼接 FP16_VA 特性信息
    s += "FP16_VA = "   + std::to_string(ggml_cpu_has_fp16_va())   + " | ";
    // 拼接 WASM_SIMD 特性信息
    s += "WASM_SIMD = " + std::to_string(ggml_cpu_has_wasm_simd()) + " | ";
    // 拼接 BLAS 特性信息
    s += "BLAS = "      + std::to_string(ggml_cpu_has_blas())      + " | ";
    // 拼接 SSE3 特性信息
    s += "SSE3 = "      + std::to_string(ggml_cpu_has_sse3())      + " | ";
    // 拼接 SSSE3 特性信息
    s += "SSSE3 = "     + std::to_string(ggml_cpu_has_ssse3())     + " | ";
    // 拼接 VSX 特性信息
    s += "VSX = "       + std::to_string(ggml_cpu_has_vsx())       + " | ";
    // 拼接 CUDA 特性信息
    s += "CUDA = "      + std::to_string(ggml_cpu_has_cublas())    + " | ";
    // 拼接 COREML 支持信息
    s += "COREML = "    + std::to_string(whisper_has_coreml())     + " | ";
    // 拼接 OPENVINO 支持信息
    s += "OPENVINO = "  + std::to_string(whisper_has_openvino())   + " | ";

    // 返回字符串 s 的 C 风格指针
    return s.c_str();
}

//////////////////////////////////
// Grammar - ported from llama.cpp
//////////////////////////////////

// 解码可能以不完整序列结尾的 UTF-8 字符串。添加终止符0以用作指针。如果遇到无效序列，则返回 `whisper_partial_utf8.n_remain == -1`。
// 解码 UTF-8 编码的字符串，返回解码后的 Unicode 码点数组和未完成的部分
std::pair<std::vector<uint32_t>, whisper_partial_utf8> decode_utf8(
        const char         * src,
        whisper_partial_utf8   partial_start) {
    // UTF-8 编码长度查找表
    static const int      lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 2, 2, 3, 4 };
    // 当前处理位置
    const char          * pos      = src;
    // 存储解码后的 Unicode 码点
    std::vector<uint32_t> code_points;
    // 存储部分解码的值和剩余字节数
    uint32_t              value    = partial_start.value;
    int                   n_remain = partial_start.n_remain;

    // 继续之前的解码，如果适用
    while (*pos != 0 && n_remain > 0) {
        // 获取下一个字节
        uint8_t next_byte = static_cast<uint8_t>(*pos);
        // 检查是否为有效的 UTF-8 序列
        if ((next_byte >> 6) != 2) {
            // 无效序列，中止解码
            code_points.push_back(0);
            return std::make_pair(std::move(code_points), whisper_partial_utf8{ 0, -1 });
        }
        // 更新值和剩余字节数
        value = (value << 6) + (next_byte & 0x3F);
        ++pos;
        --n_remain;
    }

    // 处理部分解码的情况
    if (partial_start.n_remain > 0 && n_remain == 0) {
        code_points.push_back(value);
    }

    // 解码后续的 UTF-8 序列，可能以不完整序列结束
    while (*pos != 0) {
        // 获取第一个字节和高位
        uint8_t  first_byte = static_cast<uint8_t>(*pos);
        uint8_t  highbits   = first_byte >> 4;
        // 计算剩余字节数
        n_remain   = lookup[highbits] - 1;

        if (n_remain < 0) {
            // 无效序列，中止解码
            code_points.clear();
            code_points.push_back(0);
            return std::make_pair(std::move(code_points), whisper_partial_utf8{ 0, n_remain });
        }

        // 掩码和更新值
        uint8_t  mask       = (1 << (7 - n_remain)) - 1;
        value      = first_byte & mask;
        ++pos;
        // 处理剩余字节
        while (*pos != 0 && n_remain > 0) {
            value = (value << 6) + (static_cast<uint8_t>(*pos) & 0x3F);
            ++pos;
            --n_remain;
        }
        // 如果剩余字节数为 0，则添加到码点数组中
        if (n_remain == 0) {
            code_points.push_back(value);
        }
    }
    // 添加结束标志
    code_points.push_back(0);

    return std::make_pair(std::move(code_points), whisper_partial_utf8{ value, n_remain });
}
// 返回 true 当 pos 指向规则定义的末尾时
static bool whisper_grammar_is_end_of_sequence(const whisper_grammar_element * pos) {
    switch (pos->type) {
        case WHISPER_GRETYPE_END: return true;  // NOLINT
        case WHISPER_GRETYPE_ALT: return true;  // NOLINT
        default:                return false;
    }
}

// 返回 true 当 chr 满足 pos 处的字符范围时（正向或反向范围）
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

// 返回 true 当给定部分 UTF-8 序列的某个延续可能满足 pos 处的字符范围时（正向或反向范围）
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

    do {
        // 如果下一个位置的类型为 WHISPER_GRETYPE_CHAR_RNG_UPPER
        if (pos[1].type == WHISPER_GRETYPE_CHAR_RNG_UPPER) {
            // 包含范围，例如 [a-z]
            if (pos->value <= high && low <= pos[1].value) {
                return is_positive_char;
            }
            pos += 2;
        } 
        // 如果下一个位置的类型不是 WHISPER_GRETYPE_CHAR_RNG_UPPER
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
// 将一个语法下推栈转换为 N 个可能的栈，所有栈都以一个字符范围（终结符元素）结尾
static void whisper_grammar_advance_stack(
        const std::vector<std::vector<whisper_grammar_element>>   & rules,  // 语法规则的向量
        const std::vector<const whisper_grammar_element *>        & stack,  // 下推栈的向量
        std::vector<std::vector<const whisper_grammar_element *>> & new_stacks) {  // 新栈的向量

    // 如果下推栈为空，则将其添加到新栈中并返回
    if (stack.empty()) {
        new_stacks.push_back(stack);
        return;
    }

    // 获取下推栈中最后一个元素的指针
    const whisper_grammar_element * pos = stack.back();
    switch (pos->type) {
        case WHISPER_GRETYPE_RULE_REF: {
            // 如果当前位置是规则引用类型
            const size_t                  rule_id = static_cast<size_t>(pos->value);
            // 获取规则引用的规则 ID
            const whisper_grammar_element * subpos  = rules[rule_id].data();
            // 获取规则引用对应的规则元素数组的指针
            do {
                // 初始化一个不包含顶部元素（pos）的新栈
                std::vector<const whisper_grammar_element *> new_stack(stack.begin(), stack.end() - 1);
                if (!whisper_grammar_is_end_of_sequence(pos + 1)) {
                    // 如果规则引用后面还有元素，将其添加到栈中
                    new_stack.push_back(pos + 1);
                }
                if (!whisper_grammar_is_end_of_sequence(subpos)) {
                    // 如果替代规则非空，将其添加到栈中
                    new_stack.push_back(subpos);
                }
                // 推进栈并生成新的栈
                whisper_grammar_advance_stack(rules, new_stack, new_stacks);
                while (!whisper_grammar_is_end_of_sequence(subpos)) {
                    // 扫描到替代定义的末尾
                    subpos++;
                }
                if (subpos->type == WHISPER_GRETYPE_ALT) {
                    // 还有该规则的另一个替代定义需要处理
                    subpos++;
                } else {
                    break;
                }
            } while (true);
            break;
        }
        case WHISPER_GRETYPE_CHAR:
        case WHISPER_GRETYPE_CHAR_NOT:
            // 字符类型或非字符类型，将当前栈添加到新栈中
            new_stacks.push_back(stack);
            break;
        default:
            // 替代结束（WHISPER_GRETYPE_END，WHISPER_GRETYPE_ALT）或字符范围中间（WHISPER_GRETYPE_CHAR_ALT，WHISPER_GRETYPE_CHAR_RNG_UPPER）；
            // 栈不应该留在这些位置
            WHISPER_ASSERT(false);
    }
// 为给定语法上的一组可能的下推栈（必须位于字符范围上，参见`whisper_grammar_advance_stack`），
// 如果给定字符在这些位置被接受，则生成 N 个可能的栈
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
            // 推进栈
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
    // 创建一个存储被拒绝的候选项的向量
    std::vector<whisper_grammar_candidate> rejects;

    // 如果堆栈为空
    if (stack.empty()) {
        // 遍历候选项
        for (auto tok : candidates) {
            // 如果候选项的代码点不为0或者部分 UTF-8 字符的剩余数量不为0
            if (*tok.code_points != 0 || tok.partial_utf8.n_remain != 0) {
                // 将该候选项添加到被拒绝的向量中
                rejects.push_back(tok);
            }
        }
        // 返回被拒绝的向量
        return rejects;
    }

    // 获取堆栈顶部的元素
    const whisper_grammar_element * stack_pos = stack.back();

    // 创建一个存储下一个候选项的向量
    std::vector<whisper_grammar_candidate> next_candidates;
    // 遍历候选项
    for (auto tok : candidates) {
        // 如果候选项的代码点为0
        if (*tok.code_points == 0) {
            // 如果候选项以无法满足语法位置的部分序列结束，则将其添加到被拒绝的向量中
            if (tok.partial_utf8.n_remain != 0 && !whisper_grammar_match_partial_char(stack_pos, tok.partial_utf8)) {
                rejects.push_back(tok);
            }
        } else if (whisper_grammar_match_char(stack_pos, *tok.code_points).first) {
            // 如果候选项的代码点匹配当前堆栈位置的字符，则将其添加到下一个候选项的向量中
            next_candidates.push_back({ tok.id, tok.code_points + 1, tok.partial_utf8 });
        } else {
            // 否则将候选项添加到被拒绝的向量中
            rejects.push_back(tok);
        }
    }

    // 获取堆栈顶部元素的下一个位置
    const auto * stack_pos_after = whisper_grammar_match_char(stack_pos, 0).second;

    // 更新堆栈顶部元素为下一个元素（如果有的话）
    std::vector<const whisper_grammar_element *> stack_after(stack.begin(), stack.end() - 1);
    if (!whisper_grammar_is_end_of_sequence(stack_pos_after)) {
        stack_after.push_back(stack_pos_after);
    }
    // 创建存储下一个堆栈的向量
    std::vector<std::vector<const whisper_grammar_element *>> next_stacks;
    // 推进堆栈到下一个状态
    whisper_grammar_advance_stack(rules, stack_after, next_stacks);

    // 获取下一个被拒绝的候选项
    auto next_rejects = whisper_grammar_reject_candidates(rules, next_stacks, next_candidates);
    // 将下一个被拒绝的候选项添加到被拒绝的向量中
    for (auto tok : next_rejects) {
        rejects.push_back({ tok.id, tok.code_points - 1, tok.partial_utf8 });
    }

    // 返回被拒绝的向量
    return rejects;
// 拒绝候选项，根据规则、堆栈和候选项，返回拒绝的候选项列表
static std::vector<whisper_grammar_candidate> whisper_grammar_reject_candidates(
        const std::vector<std::vector<whisper_grammar_element>>         & rules,
        const std::vector<std::vector<const whisper_grammar_element *>> & stacks,
        const std::vector<whisper_grammar_candidate>                    & candidates) {
    // 如果候选项为空或堆栈为空，则返回空的候选项列表
    if (candidates.empty() || stacks.empty()) {
        return std::vector<whisper_grammar_candidate>();
    }

    // 根据规则、堆栈的第一个元素和候选项，获取拒绝的候选项列表
    auto rejects = whisper_grammar_reject_candidates_for_stack(rules, stacks.front(), candidates);

    // 遍历堆栈中的每个元素，获取拒绝的候选项列表
    for (size_t i = 1, size = stacks.size(); i < size; ++i) {
        rejects = whisper_grammar_reject_candidates_for_stack(rules, stacks[i], rejects);
    }
    // 返回拒绝的候选项列表
    return rejects;
}

// 初始化 Whisper 语法，包括规则、规则数量和起始规则索引
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

    // 循环遍历起始规则的备选项，构建初始堆栈
    std::vector<std::vector<const whisper_grammar_element *>> stacks;
    pos = rules[i_start_rule];
    // 使用 do-while 循环处理语法规则的解析
    do {
        // 创建一个存储语法规则元素指针的栈
        std::vector<const whisper_grammar_element *> stack;
        // 如果当前位置不是序列的结尾
        if (!whisper_grammar_is_end_of_sequence(pos)) {
            // 如果有备选项，将其添加到栈中
            stack.push_back(pos);
        }
        // 推进栈的处理
        whisper_grammar_advance_stack(vec_rules, stack, stacks);
        // 扫描到备选项定义的末尾
        while (!whisper_grammar_is_end_of_sequence(pos)) {
            pos++;
        }
        // 如果当前位置的类型是备选项
        if (pos->type == WHISPER_GRETYPE_ALT) {
            // 还有这个规则的另一个备选项需要处理
            pos++;
        } else {
            // 否则跳出循环
            break;
        }
    } while (true);

    // 返回解析后的结果，包括语法规则、栈和空的字典
    return { std::move(vec_rules), std::move(stacks), {} };
// 静态函数，用于在给定语法和参数的情况下抑制无效语法
static void whisper_suppress_invalid_grammar(
             whisper_context  & ctx,  // 上下文对象的引用
    const whisper_full_params & params,  // 参数对象的常量引用
           std::vector<float> & logits,  // 浮点数向量的引用
    const     whisper_grammar & grammar) {  // 语法对象的常量引用

    // 如果语法规则或语法堆栈为空，则直接返回
    if (grammar.rules.empty() || grammar.stacks.empty()) {
        return;
    }

    // 创建一个表示结束符的令牌
    const whisper_token eot = whisper_token_eot(&ctx);

    // 创建存储解码后候选项的向量和语法候选项的向量
    std::vector<std::pair<std::vector<uint32_t>, whisper_partial_utf8>> candidates_decoded;
    std::vector<whisper_grammar_candidate> candidates_grammar;

    // 遍历所有令牌，将非空文本解码为 UTF-8，并存储到候选项向量中
    for (whisper_token id = 0; id < eot; ++id) {
        const std::string & text = ctx.vocab.id_to_token[id];
        if (!text.empty()) {
            candidates_decoded.push_back(decode_utf8(text.c_str(), grammar.partial_utf8));
            candidates_grammar.push_back({ id, candidates_decoded.back().first.data(), candidates_decoded.back().second });
        }
    }

    // 拒绝不符合语法规则的候选项，并对相应的 logits 进行惩罚
    const auto rejects = whisper_grammar_reject_candidates(grammar.rules, grammar.stacks, candidates_grammar);

    for (const auto & reject : rejects) {
        logits[reject.id] -= params.grammar_penalty;
    }

    // 当语法允许继续时，对结束符令牌进行惩罚
    //if (!allow_eot) {
    //    logits[eot] -= params.grammar_penalty;
    //}
    //fprintf(stderr, "Allowed: (%zu tokens)\n", size - rejects.size());
}

// 静态函数，用于在给定上下文、语法和令牌的情况下接受令牌
static void whisper_grammar_accept_token(whisper_context & ctx, whisper_grammar & grammar, whisper_token token) {
    // 如果语法规则或语法堆栈为空，则直接返回
    if (grammar.rules.empty() || grammar.stacks.empty()) {
        return;
    }

    // 获取令牌对应的文本
    //fprintf(stderr, "Accept: '%s'\n", ctx.vocab.id_to_token[token].c_str());

    const std::string & text = ctx.vocab.id_to_token[token];

    // 如果文本以 "[_" 开头，则跳过该令牌
    if (text.rfind("[_", 0) == 0) {
        // fprintf(stderr, " (skipped)\n");
        return;
    }
}
    // 打印换行符到标准错误流
    // 注意在解码后的字符串中添加终止符0
    // 解码给定的 UTF-8 文本，使用语法的部分 UTF-8
    const auto   decoded     = decode_utf8(text.c_str(), grammar.partial_utf8);
    // 获取解码后的代码点
    const auto & code_points = decoded.first;
    // 遍历代码点，不包括最后一个
    for (auto it = code_points.begin(), end = code_points.end() - 1; it != end; ++it) {
        // 使用语法规则接受代码点，更新语法栈
        grammar.stacks = whisper_grammar_accept(grammar.rules, grammar.stacks, *it);
    }
    // 更新部分 UTF-8 字符串
    grammar.partial_utf8 = decoded.second;
// 结束语法
//////////////

////////////////////////////////////////////////////////////////////////////

// 根据默认参数创建 whisper_context_params 结构体的指针
struct whisper_context_params * whisper_context_default_params_by_ref() {
    // 获取默认参数
    struct whisper_context_params params = whisper_context_default_params();

    // 创建新的 whisper_context_params 结构体指针
    struct whisper_context_params* result = new whisper_context_params();
    // 将默认参数赋值给新创建的结构体指针
    *result = params;
    // 返回新创建的结构体指针
    return result;
}

// 根据默认参数和采样策略创建 whisper_full_params 结构体的指针
struct whisper_full_params * whisper_full_default_params_by_ref(enum whisper_sampling_strategy strategy) {
    // 获取默认参数
    struct whisper_full_params params = whisper_full_default_params(strategy);

    // 创建新的 whisper_full_params 结构体指针
    struct whisper_full_params* result = new whisper_full_params();
    // 将默认参数赋值给新创建的结构体指针
    *result = params;
    // 返回新创建的结构体指针
    return result;
}

// 根据采样策略设置默认参数
struct whisper_full_params whisper_full_default_params(enum whisper_sampling_strategy strategy) {
    // 创建结果结构体
    };

    // 根据不同的采样策略设置参数
    switch (strategy) {
        case WHISPER_SAMPLING_GREEDY:
            {
                // 设置贪婪策略参数
                result.greedy = {
                    /*.best_of   =*/ 5,
                };
            } break;
        case WHISPER_SAMPLING_BEAM_SEARCH:
            {
                // 设置波束搜索策略参数
                result.beam_search = {
                    /*.beam_size =*/ 5,

                    /*.patience  =*/ -1.0f,
                };
            } break;
    }

    // 返回设置好的参数
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

// 检查是否应该在单词上拆分
static inline bool should_split_on_word(const char * txt, bool split_on_word) {
    // 如果不需要在单词上拆分，则返回 true
    if (!split_on_word) return true;

    // 如果文本以空格开头，则返回 true
    return txt[0] == ' ';
}

// 将最后一个段落包装到最大长度的字符中
// 返回新段落的数量
// 将当前段落包装成指定长度的片段，并根据是否需要在单词间拆分进行处理
static int whisper_wrap_segment(struct whisper_context & ctx, struct whisper_state & state, int max_len, bool split_on_word) {
    // 获取当前段落
    auto segment = state.result_all.back();

    // 初始化返回值和累计长度
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
        // 获取当前标记的长度
        const int cur = strlen(txt);

        // 如果累计长度加上当前标记长度超过最大长度，并且不是第一个标记，并且需要在单词间拆分
        if (acc + cur > max_len && i > 0 && should_split_on_word(txt, split_on_word)) {
            // 将已处理的文本赋值给当前段落
            state.result_all.back().text = std::move(text);
            state.result_all.back().t1 = token.t0;
            state.result_all.back().tokens.resize(i);
            state.result_all.back().speaker_turn_next = false;

            // 创建新的段落
            state.result_all.push_back({});
            state.result_all.back().t0 = token.t0;
            state.result_all.back().t1 = segment.t1;

            // 将剩余的标记添加到新的段落中
            state.result_all.back().tokens.insert(
                state.result_all.back().tokens.end(),
                    segment.tokens.begin() + i,
                    segment.tokens.end());

            state.result_all.back().speaker_turn_next = segment.speaker_turn_next;

            // 重置累计长度和文本字符串
            acc = 0;
            text = "";

            // 更新当前段落为新创建的段落
            segment = state.result_all.back();
            i = -1;

            // 增加返回值
            res++;
        } else {
            // 更新累计长度和文本字符串
            acc += cur;
            text += txt;
        }
    }

    // 将剩余的文本赋值给当前段落
    state.result_all.back().text = std::move(text);

    // 返回处理的段落数量
    return res;
}

// 非语音标记列表
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
    const auto & vocab      = ctx.vocab; // 获取上下文的词汇表
    const auto & tokens_cur = decoder.sequence.tokens; // 获取解码器当前序列的tokens

    const bool is_initial = tokens_cur.size() == 0; // 检查是否为初始状态
    const int  n_logits   = vocab.id_to_token.size(); // 获取logits的数量

    WHISPER_ASSERT(n_logits == ctx.vocab.n_vocab); // 断言logits数量与词汇表大小相同

    // 提取最后一个token的logits
    // 我们将进行更改，因此不希望直接使用ctx.logits缓冲区
    auto & probs    = decoder.probs;
    auto & logits   = decoder.logits;
    auto & logprobs = decoder.logprobs;
    {
        logits.resize(n_logits); // 调整logits的大小
        memcpy(logits.data(), state.logits.data() + decoder.i_batch*n_logits, n_logits*sizeof(float)); // 复制logits数据

        if (temperature > 0.0f) {
            for (int i = 0; i < n_logits; i++) {
                logits[i] /= temperature; // 根据温度调整logits值
            }
        }

        //稍后将填充
        probs.resize(n_logits); // 调整probs的大小
        logprobs.resize(n_logits); // 调整logprobs的大小
    }

    // 在这里应用logit过滤器
    // 参考: https://github.com/openai/whisper/blob/0b1ba3d46ebf7fe6f953acfd8cad62a4f851b49f/whisper/decoding.py#L480-L493
    }

    // 计算probs
    {
        for (int i = 0; i < n_logits; ++i) {
            if (logits[i] == -INFINITY) {
                probs[i] = 0.0f;
            } else {
                probs[i] = expf(logprobs[i]); // 计算probs值
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

        // 对pairs进行排序，按照概率从大到小排序
        std::sort(pairs.begin(), pairs.end(), [](const std::pair<float, int>& a, const std::pair<float, int>& b) {
            return a.first > b.first;
        });

        // 打印前10个排序后的概率和相关信息
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

    // "And", "and", " And", " and"
    //printf("logits[\"and\"]  = %f\n", logits[vocab.token_to_id.at("and")]);
    //printf("logits[\"And\"]  = %f\n", logits[vocab.token_to_id.at("And")]);
    //printf("logits[\" and\"] = %f\n", logits[vocab.token_to_id.at(" and")]);
    //printf("logits[\" And\"] = %f\n", logits[vocab.token_to_id.at(" And")]);
    //printf("logits[\" so\"]  = %f\n", logits[vocab.token_to_id.at(" so")]);

    //printf("logprobs[\"and\"]  = %f\n", logprobs[vocab.token_to_id.at("and")]);
    //printf("logprobs[\"And\"]  = %f\n", logprobs[vocab.token_to_id.at("And")]);
    //printf("logprobs[\" and\"] = %f\n", logprobs[vocab.token_to_id.at(" and")]);
    //printf("logprobs[\" And\"] = %f\n", logprobs[vocab.token_to_id.at(" And")]);
    //printf("logprobs[\" so\"]  = %f\n", logprobs[vocab.token_to_id.at(" so")]);

    //printf("probs[\"and\"]  = %f\n", probs[vocab.token_to_id.at("and")]);
    //printf("probs[\"And\"]  = %f\n", probs[vocab.token_to_id.at("And")]);
    // 打印 probs 字典中键为 " and" 对应的值
    //printf("probs[\" and\"] = %f\n", probs[vocab.token_to_id.at(" and")]);
    // 打印 probs 字典中键为 " And" 对应的值
    //printf("probs[\" And\"] = %f\n", probs[vocab.token_to_id.at(" And")]);
    // 打印 probs 字典中键为 " so" 对应的值
    //printf("probs[\" so\"]  = %f\n", probs[vocab.token_to_id.at(" so")]);
// 结束 C++ 预处理指令
#endif
}

// 生成一个样本 token
static whisper_token_data whisper_sample_token(
            whisper_context & ctx,
      const whisper_decoder & decoder,
                       bool   best) {
    // 初始化结果结构体
    whisper_token_data result = {
        0, 0, 0.0f, 0.0f, 0.0f, 0.0f, -1, -1, 0.0f,
    };

    // 获取词汇表
    const auto & vocab = ctx.vocab;

    // 获取概率和对数概率
    const auto & probs    = decoder.probs;
    const auto & logprobs = decoder.logprobs;

    // 获取词汇表大小
    const int n_logits = vocab.n_vocab;

    {
        // 初始化总概率和最大概率
        double sum_ts = 0.0;
        double max_ts = 0.0;

        // 遍历词汇表中的 token
        for (int i = vocab.token_beg; i < n_logits; i++) {
            // 如果概率为负无穷，则跳过
            if (probs[i] == -INFINITY) {
                continue;
            }

            // 计算总概率和更新最大概率及对应的 token id
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

    // 如果选择最佳 token
    if (best) {
        // 遍历所有 token，找到概率最大的 token
        for (int i = 0; i < n_logits; ++i) {
            if (result.p < probs[i]) {
                result.id   = i;
                result.p    = probs[i];
                result.plog = logprobs[i];
            }
        }
    } else {
        // 使用概率生成离散分布
        std::discrete_distribution<> dist(probs.begin(), probs.end());

        // 从分布中采样一个 token
        result.id   = dist(decoder.rng);
        result.p    = probs[result.id];
        result.plog = logprobs[result.id];
    }

    // 如果选择的 token 在词汇表中
    if (result.id >= vocab.token_beg) {
        result.tid = result.id;
        result.pt  = result.p;
    }

    // 返回生成的 token
    return result;
}

// 生成前 k 个 token
static std::vector<whisper_token_data> whisper_sample_token_topk(
            whisper_context & ctx,
            whisper_decoder & decoder,
                        int   k) {
    // 获取词汇表
    const auto & vocab = ctx.vocab;

    // 获取概率、logits 和对数概率
    const auto & probs    = decoder.probs;
    const auto & logits   = decoder.logits;
    const auto & logprobs = decoder.logprobs;

    // 获取词汇表大小
    const int n_logits = vocab.n_vocab;

    // 初始化 logits_id
    auto & logits_id = decoder.logits_id;

    logits_id.resize(n_logits);
}
    // 遍历 logits 数组，将每个元素的值和索引组成 pair 存入 logits_id 数组
    for (int i = 0; i < n_logits; ++i) {
        logits_id[i].first = logits[i];
        logits_id[i].second = i;
    }

    {
        // 定义 pair_type 类型为 logits_id 的 value_type
        using pair_type = std::remove_reference<decltype(logits_id)>::type::value_type;
        // 对 logits_id 数组进行部分排序，保留前 k 个元素，按照第一个元素（logits 值）降序排列
        std::partial_sort(
                logits_id.begin(),
                logits_id.begin() + k, logits_id.end(),
                [](const pair_type & a, const pair_type & b) {
            return a.first > b.first;
        });
    }

    // 创建一个空的 whisper_token_data 类型的 vector，并预留 k 个元素的空间
    std::vector<whisper_token_data> result;
    result.reserve(k);

    // 初始化变量 tid、pt、ptsum
    whisper_token tid = vocab.token_beg;
    float pt    = 0.0;
    float ptsum = 0.0;

    {
        // 初始化 sum_ts 和 max_ts 为 0
        double sum_ts = 0.0;
        double max_ts = 0.0;

        // 遍历 probs 数组，计算 sum_ts 和 max_ts，并更新 tid
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

        // 计算 pt 和 ptsum
        pt    = max_ts/(sum_ts + 1e-10);
        ptsum = sum_ts;
    }

    // 创建一个离散分布对象 dist，以 probs 数组为参数
    std::discrete_distribution<> dist(probs.begin(), probs.end());

    // 遍历 k 次，根据 dist 随机选择一个 id，并将相关信息存入 result 中
    for (int i = 0; i < k; ++i) {
        const auto id = dist(decoder.rng);
        //printf("XXX %d %d %f %f %f %f\n", id, tid, probs[id], logprobs[id], pt, ptsum);

        // 将选定的 id、tid、相关概率信息存入 result
        result.push_back({ id, tid, probs[id], logprobs[id], pt, ptsum, -1, -1, 0.0f, });

        // 如果 result[i].id 大于等于 vocab.token_beg，则更新 tid 和 pt
        if (result[i].id >= vocab.token_beg) {
            result[i].tid = result[i].id;
            result[i].pt  = result[i].p;
        }
    }

    // 返回结果 vector
    return result;
}

// 定义一个静态函数，用于计算序列的得分
// 参数包括参数结构体和序列对象
static void whisper_sequence_score(
        const struct whisper_full_params & params,
                        whisper_sequence & sequence) {
    // 如果序列结果长度为0，则直接返回
    if (sequence.result_len == 0) {
        return;
    }

    // 初始化结果值为0
    double result = 0.0f;

    // 遍历序列结果中的每个token，累加其概率对数值
    for (int i = 0; i < sequence.result_len; ++i) {
        result += sequence.tokens[i].plog;
    }

    // 计算序列的总对数概率和平均对数概率
    sequence.sum_logprobs = result;
    sequence.avg_logprobs = result/sequence.result_len;

    // 初始化惩罚值为序列结果长度
    double penalty = sequence.result_len;

    // 如果长度惩罚大于0，则计算惩罚值
    if (params.length_penalty > 0.0f) {
        penalty = pow((5.0 + penalty)/6.0, params.length_penalty);
    }

    // 计算序列的得分
    sequence.score = result/penalty;

    // 计算最后32个token序列的熵
    {
        const int n = 32;

        int cnt = 0;
        double entropy = 0.0f;

        // 统计最后n个token的出现次数
        std::map<whisper_token, int> token_counts;
        for (int i = std::max(0, sequence.result_len - n); i < sequence.result_len; ++i) {
            token_counts[sequence.tokens[i].id]++;
            cnt++;
        }

        // 计算熵值
        for (const auto & kv : token_counts) {
            const auto p = kv.second/(double)cnt;
            entropy -= p*log(p);

            //WHISPER_LOG_DEBUG("entropy: %d %f %f, count %d\n", kv.first, p, log(p), kv.second);
        }

        // 将计算得到的熵值赋给序列对象
        sequence.entropy = entropy;
    }
}

// 使用状态进行完整的whisper计算
int whisper_full_with_state(
        struct whisper_context * ctx,
          struct whisper_state * state,
    struct whisper_full_params   params,
                   const float * samples,
                           int   n_samples) {
    // 清空旧的结果
    auto & result_all = state->result_all;

    result_all.clear();
    // 如果样本数量大于0
    if (n_samples > 0) {
        // 计算对数梅尔频谱
        if (params.speed_up) {
            // 如果需要加速，则输出错误信息并返回-1
            WHISPER_LOG_ERROR("%s: failed to compute log mel spectrogram\n", __func__);
            return -1;
        } else {
            // 否则调用函数计算对数梅尔频谱
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

        // 自动检测语言
        const auto lang_id = whisper_lang_auto_detect_with_state(ctx, state, 0, params.n_threads, probs.data());
        if (lang_id < 0) {
            // 如果自动检测失败，则输出错误信息并返回-3
            WHISPER_LOG_ERROR("%s: failed to auto-detect language\n", __func__);
            return -3;
        }
        // 更新语言ID和语言参数
        state->lang_id = lang_id;
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
        // 初始化时间戳相关参数
        state->t_beg    = 0;
        state->t_last   = 0;
        state->tid_last = 0;
        // 如果样本数量大于0，则计算信号能量
        if (n_samples > 0) {
            state->energy = get_signal_energy(samples, n_samples, 32);
        }
    }

    // 计算起始和结束帧索引
    const int seek_start = params.offset_ms/10;
    const int seek_end = params.duration_ms == 0 ? whisper_n_len_from_state(state) : seek_start + params.duration_ms/10;

    // 如果频谱长度小于1.0秒（100帧），则返回
    // 基本上不处理小于1.0秒的内容
    // 参考问题＃39：https://github.com/ggerganov/whisper.cpp/issues/39
    // 如果输入的时间段太短，则返回错误
    if (seek_end < seek_start + (params.speed_up ? 50 : 100)) {
        WHISPER_LOG_DEBUG("%s: input is too short - %d ms < 1000 ms\n", __func__, (seek_end - seek_start)*10);
        return 0;
    }

    // 一组要使用的温度值
    // [ t0, t0 + delta, t0 + 2*delta, ..., < 1.0f + 1e-6f ]
    std::vector<float> temperatures;
    if (params.temperature_inc > 0.0f) {
        for (float t = params.temperature; t < 1.0f + 1e-6f; t += params.temperature_inc) {
            temperatures.push_back(t);
        }
    } else {
        temperatures.push_back(params.temperature);
    }

    // 初始化解码器
    int n_decoders = 1;

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

    n_decoders = std::max(1, n_decoders);

    // 如果请求的解码器数量超过最大值，则返回错误
    if (n_decoders > WHISPER_MAX_DECODERS) {
        WHISPER_LOG_ERROR("%s: too many decoders requested (%d), max = %d\n", __func__, n_decoders, WHISPER_MAX_DECODERS);
        return -4;
    }

    // 初始化解码器
    // TAGS: WHISPER_DECODER_INIT
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
    // 如果不使用上下文，则清空
    if (params.no_context) {
        prompt_past.clear();
    }

    // 准备提示
    {
        // 创建一个存储提示标记的向量
        std::vector<whisper_token> prompt_tokens;

        // 如果没有指定提示标记但有初始提示文本
        if (!params.prompt_tokens && params.initial_prompt) {
            // 调整向量大小为1024
            prompt_tokens.resize(1024);
            // 使用whisper_tokenize函数将初始提示文本转换为标记，并更新向量大小
            prompt_tokens.resize(whisper_tokenize(ctx, params.initial_prompt, prompt_tokens.data(), prompt_tokens.size()));
            // 更新params中的提示标记和标记数量
            params.prompt_tokens   = prompt_tokens.data();
            params.prompt_n_tokens = prompt_tokens.size();
        }

        // 将提示标记添加到提示过去的标记之前
        if (params.prompt_tokens && params.prompt_n_tokens > 0) {
            // 从指针中解析标记
            for (int i = 0; i < params.prompt_n_tokens; i++) {
                prompt_past.push_back(params.prompt_tokens[i]);
            }
            // 将标记旋转，使提示过去的标记在前
            std::rotate(prompt_past.begin(), prompt_past.end() - params.prompt_n_tokens, prompt_past.end());
        }
    }

    // 覆盖audio_ctx，最大允许值为hparams.n_audio_ctx
    if (params.audio_ctx > whisper_n_audio_ctx(ctx)) {
        // 如果audio_ctx大于最大允许值，记录错误并返回-5
        WHISPER_LOG_ERROR("%s: audio_ctx is larger than the maximum allowed (%d > %d)\n", __func__, params.audio_ctx, whisper_n_audio_ctx(ctx));
        return -5;
    }
    // 更新状态中的exp_n_audio_ctx
    state->exp_n_audio_ctx = params.audio_ctx;

    // 这些标记确定将执行的任务
    std::vector<whisper_token> prompt_init = { whisper_token_sot(ctx), };

    // 如果是多语言环境
    if (whisper_is_multilingual(ctx)) {
        // 获取语言ID并更新状态中的lang_id
        const int lang_id = whisper_lang_id(params.language);
        state->lang_id = lang_id;
        // 添加语言标记和翻译或转录标记到prompt_init中
        prompt_init.push_back(whisper_token_lang(ctx, lang_id));
        if (params.translate) {
            prompt_init.push_back(whisper_token_translate(ctx));
        } else {
            prompt_init.push_back(whisper_token_transcribe(ctx));
        }
    }

    // 精简模型需要"no_timestamps"标记
    {
        // 检查是否使用了蒸馏模型，并且未禁用时间戳
        const bool is_distil = ctx->model.hparams.n_text_layer == 2;
        if (is_distil && !params.no_timestamps) {
            // 如果使用了蒸馏模型且未禁用时间戳，则输出警告信息并强制禁用时间戳
            WHISPER_LOG_WARN("%s: using distilled model - forcing no_timestamps\n", __func__);
            params.no_timestamps = true;
        }
    }
    
    // 如果禁用了时间戳，则将一个特殊的令牌添加到初始化提示中
    if (params.no_timestamps) {
        prompt_init.push_back(whisper_token_not(ctx));
    }
    
    // 初始化 seek 变量为 seek_start
    int seek = seek_start;
    
    // 创建一个空的令牌向量 prompt，并预留空间以容纳文本上下文中的令牌数量
    std::vector<whisper_token> prompt;
    prompt.reserve(whisper_n_text_ctx(ctx));
    
    // 定义一个结构体 beam_candidate，用于存储候选项的信息
    struct beam_candidate {
        int decoder_idx;
        int seek_delta;
        bool has_ts;
        whisper_sequence sequence;
        whisper_grammar grammar;
    };
    
    // 创建一个二维向量 bc_per_dec，用于存储每个解码器的 beam_candidate
    std::vector<std::vector<beam_candidate>> bc_per_dec(n_decoders);
    
    // 创建一个向量 beam_candidates，用于存储所有的 beam_candidate
    std::vector<beam_candidate> beam_candidates;
    
    // 主循环开始
#ifdef WHISPER_DEBUG
                        {
                            // 如果定义了 WHISPER_DEBUG 宏，则输出调试信息
                            const auto tt = token.pt > 0.10 ? ctx->vocab.id_to_token.at(token.tid) : "[?]";
                            // 使用 WHISPER_LOG_DEBUG 宏输出调试信息
                            WHISPER_LOG_DEBUG("%s: id = %3d, decoder = %d, token = %6d, p = %6.3f, ts = %10s, %6.3f, result_len = %4d '%s'\n",
                                    __func__, i, j, token.id, token.p, tt.c_str(), token.pt, result_len, ctx->vocab.id_to_token.at(token.id).c_str());
                        }
    }

    // 返回值为 0
    return 0;
}

// 对外接口函数，调用 whisper_full_with_state 函数
int whisper_full(
        struct whisper_context * ctx,
    struct whisper_full_params   params,
                   const float * samples,
                           int   n_samples) {
    return whisper_full_with_state(ctx, ctx->state, params, samples, n_samples);
}

// 并行处理函数，根据给定的处理器数量进行并行处理
int whisper_full_parallel(
        struct whisper_context * ctx,
        struct whisper_full_params params,
        const float * samples,
        int n_samples,
        int n_processors) {
    // 如果只有一个处理器，则直接调用 whisper_full 函数
    if (n_processors == 1) {
        return whisper_full(ctx, params, samples, n_samples);
    }
    int ret = 0;

    // 为每个线程准备独立的状态
    std::vector<whisper_state*> states;

    // 计算偏移样本数和每个处理器的样本数
    const int offset_samples = (WHISPER_SAMPLE_RATE*params.offset_ms)/1000;
    const int n_samples_per_processor = (n_samples - offset_samples)/n_processors;

    // 调用线程将处理第一个块，其他线程将处理剩余的块

    // 创建线程对象数组
    std::vector<std::thread> workers(n_processors - 1);
    // 遍历创建每个线程的状态
    for (int i = 0; i < n_processors - 1; ++i) {
        // 为每个线程创建一个新的状态
        states.push_back(whisper_init_state(ctx));

        // 计算每个线程处理的起始样本数
        const int start_samples = offset_samples + (i + 1)*n_samples_per_processor;
        // 计算当前线程处理的样本数
        const int n_samples_cur = (i == n_processors - 2) ? n_samples - start_samples : n_samples_per_processor;

        // 复制参数到当前线程的参数
        auto params_cur = params;

        // 设置当前线程参数的偏移时间为0，关闭进度和实时打印
        params_cur.offset_ms = 0;
        params_cur.print_progress = false;
        params_cur.print_realtime = false;

        // 设置回调函数为nullptr
        params_cur.new_segment_callback = nullptr;
        params_cur.new_segment_callback_user_data = nullptr;

        params_cur.progress_callback = nullptr;
        params_cur.progress_callback_user_data = nullptr;

        // 创建线程并执行处理函数
        workers[i] = std::thread(whisper_full_with_state, ctx, states[i], std::move(params_cur), samples + start_samples, n_samples_cur);
    }

    {
        // 复制参数到当前线程的参数
        auto params_cur = params;

        // 禁用实时打印，因为只有第一个块会显示
        params_cur.print_realtime = false;

        // 运行第一个转换，使用默认状态，但只针对第一个块
        ret = whisper_full_with_state(ctx, ctx->state, std::move(params_cur), samples, offset_samples + n_samples_per_processor);
    }

    // 等待所有线程执行完毕
    for (int i = 0; i < n_processors - 1; ++i) {
        workers[i].join();
    }

    // 计算时间偏移
    const int64_t offset_t = (int64_t) params.offset_ms/10.0;

    // 将所有状态的结果合并到result_state->result_all中
    // 遍历处理器，从第一个到倒数第二个
    for (int i = 0; i < n_processors - 1; ++i) {
        // 获取第i个处理器的结果集
        auto& results_i = states[i]->result_all;

        // 遍历第i个处理器的结果集
        for (auto& result : results_i) {
            // 校正段时间戳，考虑偏移量
            result.t0 += 100 * ((i + 1) * n_samples_per_processor) / WHISPER_SAMPLE_RATE + offset_t;
            result.t1 += 100 * ((i + 1) * n_samples_per_processor) / WHISPER_SAMPLE_RATE + offset_t;

            // 确保段不重叠
            if (!ctx->state->result_all.empty()) {
                result.t0 = std::max(result.t0, ctx->state->result_all.back().t1);
            }

            // 将结果添加到上下文状态的结果集中
            ctx->state->result_all.push_back(std::move(result));

            // 对每个段调用新段回调函数
            if (params.new_segment_callback) {
                params.new_segment_callback(ctx, ctx->state, 1, params.new_segment_callback_user_data);
            }
        }

        // 更新上下文状态的时间统计信息
        ctx->state->t_mel_us += states[i]->t_mel_us;
        ctx->state->t_sample_us += states[i]->t_sample_us;
        ctx->state->t_encode_us += states[i]->t_encode_us;
        ctx->state->t_decode_us += states[i]->t_decode_us;
        ctx->state->t_batchd_us += states[i]->t_batchd_us;
        ctx->state->t_prompt_us += states[i]->t_prompt_us;

        // 更新上下文状态的样本数量统计信息
        ctx->state->n_sample += states[i]->n_sample;
        ctx->state->n_encode += states[i]->n_encode;
        ctx->state->n_decode += states[i]->n_decode;
        ctx->state->n_batchd += states[i]->n_batchd;
        ctx->state->n_prompt += states[i]->n_prompt;

        // 释放第i个处理器的状态
        whisper_free_state(states[i]);
    }

    // 计算时间统计信息的平均值
    ctx->state->t_mel_us    /= n_processors;
    ctx->state->t_sample_us /= n_processors;
    ctx->state->t_encode_us /= n_processors;
    ctx->state->t_decode_us /= n_processors;

    // 打印关于音频边界的信息
    WHISPER_LOG_WARN("\n");
    WHISPER_LOG_WARN("%s: the audio has been split into %d chunks at the following times:\n", __func__, n_processors);
    # 循环遍历处理器数量减一次，从0到n_processors-2
    for (int i = 0; i < n_processors - 1; ++i) {
        # 输出警告日志，显示函数名、分割范围、时间戳
        WHISPER_LOG_WARN("%s: split %d - %s\n", __func__, (i + 1), to_timestamp(100*((i + 1)*n_samples_per_processor)/WHISPER_SAMPLE_RATE + offset_t).c_str());
    }
    # 输出警告日志，提示在这些边界附近可能会降低转录质量
    WHISPER_LOG_WARN("%s: the transcription quality may be degraded near these boundaries\n", __func__);

    # 返回结果
    return ret;
// 从状态结构体中获取所有结果的段落数量
int whisper_full_n_segments_from_state(struct whisper_state * state) {
    return state->result_all.size();
}

// 获取上下文结构体中所有结果的段落数量
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

// 从状态结构体中获取指定段落的起始时间
int64_t whisper_full_get_segment_t0_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].t0;
}

// 获取上下文结构体中指定段落的起始时间
int64_t whisper_full_get_segment_t0(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].t0;
}

// 从状态结构体中获取指定段落的结束时间
int64_t whisper_full_get_segment_t1_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].t1;
}

// 获取上下文结构体中指定段落的结束时间
int64_t whisper_full_get_segment_t1(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].t1;
}

// 从状态结构体中获取指定段落的说话者转换标志
bool whisper_full_get_segment_speaker_turn_next_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].speaker_turn_next;
}

// 获取上下文结构体中指定段落的说话者转换标志
bool whisper_full_get_segment_speaker_turn_next(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].speaker_turn_next;
}

// 从状态结构体中获取指定段落的文本内容
const char * whisper_full_get_segment_text_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].text.c_str();
}

// 获取上下文结构体中指定段落的文本内容
const char * whisper_full_get_segment_text(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].text.c_str();
}

// 从状态结构体中获取指定段落的标记数量
int whisper_full_n_tokens_from_state(struct whisper_state * state, int i_segment) {
    return state->result_all[i_segment].tokens.size();
}

// 获取上下文结构体中指定段落的标记数量
int whisper_full_n_tokens(struct whisper_context * ctx, int i_segment) {
    return ctx->state->result_all[i_segment].tokens.size();
}
// 从给定的上下文和状态中获取指定段落和标记的文本内容
const char * whisper_full_get_token_text_from_state(struct whisper_context * ctx, struct whisper_state * state, int i_segment, int i_token) {
    return ctx->vocab.id_to_token[state->result_all[i_segment].tokens[i_token].id].c_str();
}

// 从给定的上下文中获取指定段落和标记的文本内容
const char* whisper_full_get_token_text(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->vocab.id_to_token[ctx->state->result_all[i_segment].tokens[i_token].id].c_str();
}

// 从给定的状态中获取指定段落和标记的标记 ID
whisper_token whisper_full_get_token_id_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token].id;
}

// 从给定的上下文中获取指定段落和标记的标记 ID
whisper_token whisper_full_get_token_id(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token].id;
}

// 从给定的状态中获取指定段落和标记的标记数据
struct whisper_token_data whisper_full_get_token_data_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token];
}

// 从给定的上下文中获取指定段落和标记的标记数据
struct whisper_token_data whisper_full_get_token_data(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token];
}

// 从给定的状态中获取指定段落和标记的概率值
float whisper_full_get_token_p_from_state(struct whisper_state * state, int i_segment, int i_token) {
    return state->result_all[i_segment].tokens[i_token].p;
}

// 从给定的上下文中获取指定段落和标记的概率值
float whisper_full_get_token_p(struct whisper_context * ctx, int i_segment, int i_token) {
    return ctx->state->result_all[i_segment].tokens[i_token].p;
}

// =================================================================================================

// 临时接口，用于暴露 ggml 接口
// 在将来当 ggml 成为一个独立库时将被移除

// 执行 memcpy 的基准测试，输出结果到标准错误流
WHISPER_API int whisper_bench_memcpy(int n_threads) {
    fputs(whisper_bench_memcpy_str(n_threads), stderr);
    return 0;
}

// 返回执行 memcpy 基准测试的结果字符串
WHISPER_API const char * whisper_bench_memcpy_str(int n_threads) {
    static std::string s;
    s = "";
    char strbuf[256];

    // 初始化 ggml 时间
    ggml_time_init();
}
    size_t n    = 20; // 定义变量 n，赋值为 20
    size_t arr  = n_threads > 0 ? 1024llu : n_threads; // 如果 n_threads 大于 0，则 arr 赋值为 1024，否则赋值为 n_threads

    // 1GB array
    const size_t size = arr*1e6; // 定义常量 size，赋值为 arr 乘以 1e6

    double sum  = 0.0; // 定义变量 sum，赋值为 0.0

    // heat-up
    {
        char * src = (char *) malloc(size); // 分配大小为 size 的内存给 src
        char * dst = (char *) malloc(size); // 分配大小为 size 的内存给 dst

        for (size_t i = 0; i < size; i++) src[i] = i; // 初始化 src 数组

        memcpy(dst, src, size); // 复制 src 到 dst，用于热身

        double tsum = 0.0; // 定义变量 tsum，赋值为 0.0

        for (size_t i = 0; i < n; i++) { // 循环 n 次
            const int64_t t0 = ggml_time_us(); // 记录起始时间

            memcpy(dst, src, size); // 复制 src 到 dst

            const int64_t t1 = ggml_time_us(); // 记录结束时间

            tsum += (t1 - t0)*1e-6; // 计算时间差并累加到 tsum

            src[rand() % size] = rand() % 256; // 随机修改 src 数组的值
        }

        snprintf(strbuf, sizeof(strbuf), "memcpy: %7.2f GB/s (heat-up)\n", (double) (n*size)/(tsum*1e9)); // 格式化输出字符串到 strbuf
        s += strbuf; // 将字符串添加到 s 变量中

        // needed to prevent the compiler from optimizing the memcpy away
        {
            for (size_t i = 0; i < size; i++) sum += dst[i]; // 遍历 dst 数组并累加到 sum
        }

        free(src); // 释放 src 内存
        free(dst); // 释放 dst 内存
    }

    // single-thread
    {
        char * src = (char *) malloc(size); // 分配大小为 size 的内存给 src
        char * dst = (char *) malloc(size); // 分配大小为 size 的内存给 dst

        for (size_t i = 0; i < size; i++) src[i] = i; // 初始化 src 数组

        memcpy(dst, src, size); // 复制 src 到 dst，用于热身

        double tsum = 0.0; // 定义变量 tsum，赋值为 0.0

        for (size_t i = 0; i < n; i++) { // 循环 n 次
            const int64_t t0 = ggml_time_us(); // 记录起始时间

            memcpy(dst, src, size); // 复制 src 到 dst

            const int64_t t1 = ggml_time_us(); // 记录结束时间

            tsum += (t1 - t0)*1e-6; // 计算时间差并累加到 tsum

            src[rand() % size] = rand() % 256; // 随机修改 src 数组的值
        }

        snprintf(strbuf, sizeof(strbuf), "memcpy: %7.2f GB/s ( 1 thread)\n", (double) (n*size)/(tsum*1e9)); // 格式化输出字符串到 strbuf
        s += strbuf; // 将字符串添加到 s 变量中

        // needed to prevent the compiler from optimizing the memcpy away
        {
            for (size_t i = 0; i < size; i++) sum += dst[i]; // 遍历 dst 数组并累加到 sum
        }

        free(src); // 释放 src 内存
        free(dst); // 释放 dst 内存
    }

    // multi-thread
    // 循环创建多个线程，每个线程执行一定的操作
    for (int32_t k = 1; k <= n_threads; k++) {
        // 分配内存空间用于存储源数据和目标数据
        char * src = (char *) malloc(size);
        char * dst = (char *) malloc(size);

        // 初始化源数据
        for (size_t i = 0; i < size; i++) src[i] = i;

        // 预热，将源数据拷贝到目标数据
        memcpy(dst, src, size); // heat-up

        // 初始化时间总和
        double tsum = 0.0;

        // 定义 lambda 函数 helper，用于执行具体的操作
        auto helper = [&](int th) {
            // 计算每个线程处理的数据范围
            const int64_t i0 = (th + 0)*size/k;
            const int64_t i1 = (th + 1)*size/k;

            // 循环执行操作
            for (size_t i = 0; i < n; i++) {
                // 在目标数据中复制部分源数据
                memcpy(dst + i0, src + i0, i1 - i0);

                // 在源数据中随机修改一个值
                src[i0 + rand() % (i1 - i0)] = rand() % 256;
            };
        };

        // 记录开始时间
        const int64_t t0 = ggml_time_us();

        // 创建多个线程并执行 helper 函数
        std::vector<std::thread> threads(k - 1);
        for (int32_t th = 0; th < k - 1; ++th) {
            threads[th] = std::thread(helper, th);
        }

        // 执行最后一个线程的操作
        helper(k - 1);

        // 等待所有线程执行完毕
        for (int32_t th = 0; th < k - 1; ++th) {
            threads[th].join();
        }

        // 记录结束时间
        const int64_t t1 = ggml_time_us();

        // 计算时间差并更新时间总和
        tsum += (t1 - t0)*1e-6;

        // 计算并记录每秒传输的数据量
        snprintf(strbuf, sizeof(strbuf), "memcpy: %7.2f GB/s (%2d thread)\n", (double) (n*size)/(tsum*1e9), k);
        s += strbuf;

        // 防止编译器优化掉 memcpy 操作
        {
            // 计算目标数据的和
            for (size_t i = 0; i < size; i++) sum += dst[i];
        }

        // 释放内存空间
        free(src);
        free(dst);
    }

    // 记录目标数据的和
    snprintf(strbuf, sizeof(strbuf), "sum:    %f\n", sum);
    s += strbuf;

    // 返回结果字符串
    return s.c_str();
}

// 定义一个名为 whisper_bench_ggml_mul_mat 的函数，参数为线程数，返回整型值
WHISPER_API int whisper_bench_ggml_mul_mat(int n_threads) {
    // 将 whisper_bench_ggml_mul_mat_str 返回的字符串输出到标准错误流
    fputs(whisper_bench_ggml_mul_mat_str(n_threads), stderr);
    // 返回整数值 0
    return 0;
}

// 定义一个名为 whisper_bench_ggml_mul_mat_str 的函数，参数为线程数，返回常量字符指针
WHISPER_API const char * whisper_bench_ggml_mul_mat_str(int n_threads) {
    // 静态字符串对象 s
    static std::string s;
    // 将 s 置空
    s = "";
    // 字符数组 strbuf 大小为 256
    char strbuf[256];

    // 初始化 ggml 时间
    ggml_time_init();

    // 最大值 n_max 为 128
    const int n_max = 128;

    // 大小为 64, 128, 256, 512, 1024, 2048, 4096 的向量 sizes
    const std::vector<size_t> sizes = {
        64, 128, 256, 512, 1024, 2048, 4096,
    };

    // 最大大小 N_max 为 sizes 中最后一个元素
    const size_t N_max = sizes.back();

    // 创建大小为 3*N_max*N_max*sizeof(float) + 3*ggml_tensor_overhead() + ggml_graph_overhead() 的字节向量 buf
    std::vector<uint8_t> buf(3llu*N_max*N_max*sizeof(float) + 3*ggml_tensor_overhead() + ggml_graph_overhead());
    // 创建空的工作向量
    std::vector<uint8_t> work;

    // 在缓冲区中放入一堆随机数据
    for (size_t i = 0; i < buf.size(); i++) buf[i] = i;

    }

    // 返回 s 的 C 风格字符串
    return s.c_str();
}

// =================================================================================================

// =================================================================================================

//
// 下面是实验性内容
//
// 不确定这些内容是否应该是库的一部分，因为结果的质量不能保证。除非找到稳健的算法实现，否则可能会被移除

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

// 一个高于文本发音时间的成本函数/启发式，可以改进
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

    # 返回计算后的结果
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
        // 在半窗口范围内计算绝对值的和
        for (int j = -hw; j <= hw; j++) {
            // 确保索引在有效范围内
            if (i + j >= 0 && i + j < n_samples) {
                sum += fabs(signal[i + j]);
            }
        }
        // 计算平均值并存储在结果向量中
        result[i] = sum/(2*hw + 1);
    }

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

    // 获取信号样本数量
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

    // 如果令牌数量为1，则设置起始和结束时间戳并返回
    if (n == 1) {
        tokens[0].t0 = t0;
        tokens[0].t1 = t1;

        return;
    }

    // 获取状态中的时间戳信息
    auto & t_beg    = state.t_beg;
    auto & t_last   = state.t_last;
    auto & tid_last = state.tid_last;
}
    // 遍历 tokens 数组，处理每个 token
    for (int j = 0; j < n; ++j) {
        // 获取当前 token 的引用
        auto & token = tokens[j];

        // 处理第一个 token
        if (j == 0) {
            // 如果是第一个 token，根据条件设置 t0 和 t1
            if (token.id == whisper_token_beg(&ctx)) {
                tokens[j    ].t0 = t0;
                tokens[j    ].t1 = t0;
                tokens[j + 1].t0 = t0;

                t_beg    = t0;
                t_last   = t0;
                tid_last = whisper_token_beg(&ctx);
            } else {
                tokens[j    ].t0 = t_last;
            }
        }

        // 计算当前 token 的时间戳
        const int64_t tt = t_beg + 2*(token.tid - whisper_token_beg(&ctx));

        // 更新 token 的属性值
        tokens[j].id    = token.id;
        tokens[j].tid   = token.tid;
        tokens[j].p     = token.p;
        tokens[j].pt    = token.pt;
        tokens[j].ptsum = token.ptsum;

        // 计算当前 token 的声音长度
        tokens[j].vlen = voice_length(whisper_token_to_str(&ctx, token.id));

        // 根据条件更新时间戳
        if (token.pt > thold_pt && token.ptsum > thold_ptsum && token.tid > tid_last && tt <= t1) {
            if (j > 0) {
                tokens[j - 1].t1 = tt;
            }
            tokens[j].t0 = tt;
            tid_last = token.tid;
        }
    }

    // 设置最后两个 token 的时间戳
    tokens[n - 2].t1 = t1;
    tokens[n - 1].t0 = t1;
    tokens[n - 1].t1 = t1;

    // 更新 t_last 为 t1
    t_last = t1;

    // 查找具有未知时间戳的 token 区间
    // 根据 token 的声音长度，按比例分割区间并填充时间戳
    {
        // 初始化指针 p0 和 p1
        int p0 = 0;
        int p1 = 0;

        // 循环直到结束
        while (true) {
            // 移动 p1 直到找到第一个非负数的元素
            while (p1 < n && tokens[p1].t1 < 0) {
                p1++;
            }

            // 如果 p1 超出范围，则将其减小到 n-1
            if (p1 >= n) {
                p1--;
            }

            // 打印调试信息
            //printf("p0=%d p1=%d t0=%lld t1=%lld\n", p0, p1, tokens[p0].t0, tokens[p1].t1);

            // 如果 p1 大于 p0，则计算 p0 到 p1 之间的总长度
            if (p1 > p0) {
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

    // 修正（以防万一）
    for (int j = 0; j < n - 1; j++) {
        // 如果当前元素的 t1 为负数，则将下一个元素的 t0 设置为当前元素的 t1
        if (tokens[j].t1 < 0) {
            tokens[j + 1].t0 = tokens[j].t1;
        }

        // 如果当前元素的 t1 大于前一个元素的 t0，则调整当前元素的时间范围
        if (j > 0) {
            if (tokens[j - 1].t1 > tokens[j].t0) {
                tokens[j].t0 = tokens[j - 1].t1;
                tokens[j].t1 = std::max(tokens[j].t0, tokens[j].t1);
            }
        }
    }

    // VAD
    // 根据语音活动扩展或收缩标记

    // fixed token expand (optional)
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

    // debug info
    //for (int j = 0; j < n; ++j) {
    //    const auto & token = tokens[j];
    //}
    // 如果 token 的概率大于阈值并且概率总和大于0.01，则将 token 转换为字符串，否则返回"[?]"
    // 打印函数名、转换后的 token、概率、概率阈值、概率总和、token 长度、token 起始位置、token 结束位置、转换后的 token
    // 如果 tokens[j].id 大于等于结束标记，则跳过当前循环
// 设置日志回调函数和用户数据
void whisper_log_set(ggml_log_callback log_callback, void * user_data) {
    // 如果传入的日志回调函数不为空，则使用传入的函数，否则使用默认的回调函数
    g_state.log_callback = log_callback ? log_callback : whisper_log_callback_default;
    // 设置日志回调函数的用户数据
    g_state.log_callback_user_data = user_data;
}

// 内部日志函数，支持格式化输出
GGML_ATTRIBUTE_FORMAT(2, 3)
static void whisper_log_internal(ggml_log_level level, const char * format, ...) {
    va_list args;
    va_start(args, format);
    // 创建一个缓冲区，用于存储格式化后的日志信息
    char buffer[1024];
    // 格式化日志信息
    int len = vsnprintf(buffer, 1024, format, args);
    // 如果格式化后的长度小于缓冲区大小，则直接调用日志回调函数输出日志
    if (len < 1024) {
        g_state.log_callback(level, buffer, g_state.log_callback_user_data);
    } else {
        // 如果格式化后的长度大于等于缓冲区大小，则动态分配一个更大的缓冲区
        char* buffer2 = new char[len+1];
        // 重新格式化日志信息
        vsnprintf(buffer2, len+1, format, args);
        buffer2[len] = 0;
        // 调用日志回调函数输出日志
        g_state.log_callback(level, buffer2, g_state.log_callback_user_data);
        // 释放动态分配的缓冲区
        delete[] buffer2;
    }
    va_end(args);
}

// 默认的日志回调函数，将日志信息输出到标准错误流
static void whisper_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    // 忽略日志级别和用户数据
    (void) level;
    (void) user_data;
    // 将日志信息输出到标准错误流
    fputs(text, stderr);
    // 刷新标准错误流
    fflush(stderr);
}
```