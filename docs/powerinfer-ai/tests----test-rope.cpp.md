# `PowerInfer\tests\test-rope.cpp`

```
// 包含 ggml.h 头文件
#include "ggml.h"

// 包含数学、输入输出、标准库、断言和向量头文件
#include <cmath>
#include <cstdio>
#include <cstdlib>
#include <cassert>
#include <vector>

// 如果是 MSC 编译器，禁止警告 4244 和 4267
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 如果是 GCC 编译器，忽略 -Wdouble-promotion 警告
#if defined(__GNUC__)
#pragma GCC diagnostic ignored "-Wdouble-promotion"
#endif

// 定义最大参数个数为 3
#define MAX_NARGS 3

// 取消 MIN 和 MAX 宏定义，重新定义 MIN 和 MAX 宏
#undef MIN
#undef MAX
#define MIN(a, b) ((a) < (b) ? (a) : (b))
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 定义 GGML_SILU_FP16 宏

// 根据 GGML_DEBUG 宏定义，定义不同级别的打印日志宏
#if (GGML_DEBUG >= 1)
#define GGML_PRINT_DEBUG(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG(...)
#endif

#if (GGML_DEBUG >= 5)
#define GGML_PRINT_DEBUG_5(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG_5(...)
#endif

#if (GGML_DEBUG >= 10)
#define GGML_PRINT_DEBUG_10(...) printf(__VA_ARGS__)
#else
#define GGML_PRINT_DEBUG_10(...)
#endif

// 定义 GGML_PRINT 宏，用于打印日志
#define GGML_PRINT(...) printf(__VA_ARGS__)

// 定义 frand 函数，返回 0 到 1 之间的随机浮点数
static float frand(void) {
    return (float)rand()/(float)RAND_MAX;
}

// 定义 irand 函数，返回 0 到 n-1 之间的随机整数
static int irand(int n) {
    if (n == 0) return 0;
    return rand()%n;
}

// 定义 get_random_dims 函数，生成随机维度
static void get_random_dims(int64_t * dims, int ndims) {
    dims[0] = dims[1] = dims[2] = dims[3] = 1;

    for (int i = 0; i < ndims; i++) {
        dims[i] = 1 + irand(4);
    }
}

// 定义 get_random_tensor_f32 函数，返回随机的 float 类型张量
static struct ggml_tensor * get_random_tensor_f32(
        struct ggml_context * ctx0,
        int ndims,
        const int64_t ne[],
        float fmin,
        float fmax) {
    // 创建一个新的 float 类型张量
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F32, ndims, ne);
    # 根据维度的不同，使用不同的循环方式对结果数组进行赋值
    switch (ndims) {
        # 当维度为1时，使用单层循环对结果数组进行赋值
        case 1:
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)result->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
        # 当维度为2时，使用嵌套循环对结果数组进行赋值
        case 2:
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((float *)result->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
            break;
        # 当维度为3时，使用三层嵌套循环对结果数组进行赋值
        case 3:
            for (int i2 = 0; i2 < ne[2]; i2++) {
                for (int i1 = 0; i1 < ne[1]; i1++) {
                    for (int i0 = 0; i0 < ne[0]; i0++) {
                        ((float *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                    }
                }
            }
            break;
        # 当维度为4时，使用四层嵌套循环对结果数组进行赋值
        case 4:
            for (int i3 = 0; i3 < ne[3]; i3++) {
                for (int i2 = 0; i2 < ne[2]; i2++) {
                    for (int i1 = 0; i1 < ne[1]; i1++) {
                        for (int i0 = 0; i0 < ne[0]; i0++) {
                            ((float *)result->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                        }
                    }
                }
            }
            break;
        # 默认情况下，维度不在1到4之间，触发断言错误
        default:
            assert(false);
    };

    # 返回赋值后的结果数组
    return result;
}

static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    // 根据图和线程数计算执行计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划中有工作量
    if (plan.work_size > 0) {
        // 调整缓冲区大小
        buf.resize(plan.work_size);
        // 设置计划中的工作数据
        plan.work_data = buf.data();
    }

    // 执行图的计算
    ggml_graph_compute(graph, &plan);
}

int main(int /*argc*/, const char ** /*argv*/) {
    // 初始化参数
    struct ggml_init_params params = {
        /* .mem_size   = */ 128*1024*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ false,
    };

    // 创建工作缓冲区
    std::vector<uint8_t> work_buffer;

    // 初始化上下文
    struct ggml_context * ctx0 = ggml_init(params);

    struct ggml_tensor * x;

    // rope f32
    }

    // 释放上下文
    ggml_free(ctx0);

    // 返回成功
    return 0;
}
```