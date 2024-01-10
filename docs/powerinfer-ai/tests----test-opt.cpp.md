# `PowerInfer\tests\test-opt.cpp`

```
#include "ggml.h"

#include <cmath>
#include <cstdio>
#include <cstdlib>
#include <cassert>

#define MAX_NARGS 2

#if defined(__GNUC__)
#pragma GCC diagnostic ignored "-Wdouble-promotion"
#endif

//
// logging
//
#define GGML_DEBUG 0
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

#define GGML_PRINT(...) printf(__VA_ARGS__)


static float frand(void) {
    // 生成一个随机浮点数
    return (float)rand()/(float)RAND_MAX;
}

static struct ggml_tensor * get_random_tensor(
    struct ggml_context * ctx0, int ndims, int64_t ne[], float fmin, float fmax
) {
    // 创建一个新的张量对象
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
    }

    # 返回赋值后的结果数组
    return result;
int main(void) {
    // 初始化参数结构体，设置内存大小为1GB，内存缓冲区为空，不禁用分配
    struct ggml_init_params params = {
        /* .mem_size   = */ 1024*1024*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ false,
    };

    // 初始化上下文
    struct ggml_context * ctx = ggml_init(params);

    // 定义三个不同大小的张量
    int64_t ne1[4] = {4, 128, 1, 1};
    int64_t ne2[4] = {4, 256, 1, 1};
    int64_t ne3[4] = {128, 256, 1, 1};

    // 获取随机张量a和b，并设置为参数
    struct ggml_tensor * a = get_random_tensor(ctx, 2, ne1, -1, +1);
    struct ggml_tensor * b = get_random_tensor(ctx, 2, ne2, -1, +1);
    ggml_set_param(ctx, a);
    ggml_set_param(ctx, b);

    // 获取随机张量c
    struct ggml_tensor * c = get_random_tensor(ctx, 2, ne3, -1, +1);

    // 计算ab = a * b
    struct ggml_tensor * ab = ggml_mul_mat(ctx, a, b);
    // 计算d = c - ab
    struct ggml_tensor * d  = ggml_sub(ctx, c, ab);
    // 计算e = (d^2)的和
    struct ggml_tensor * e  = ggml_sum(ctx, ggml_sqr(ctx, d));

    // 创建新的计算图
    struct ggml_cgraph * ge = ggml_new_graph_custom(ctx, GGML_DEFAULT_GRAPH_SIZE, true);
    // 构建前向传播
    ggml_build_forward_expand(ge, e);
    // 重置计算图
    ggml_graph_reset(ge);

    // 使用上下文计算计算图
    ggml_graph_compute_with_ctx(ctx, ge, /*n_threads*/ 1);

    // 获取张量e的第一个元素值
    const float fe = ggml_get_f32_1d(e, 0);
    // 打印结果
    printf("%s: e = %.4f\n", __func__, fe);

    // 设置优化参数
    struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_ADAM);

    // 优化张量e
    ggml_opt(ctx, opt_params, e);

    // 重置计算图
    ggml_graph_reset(ge);

    // 使用上下文计算计算图
    ggml_graph_compute_with_ctx(ctx, ge, /*n_threads*/ 1);

    // 获取优化后的张量e的第一个元素值
    const float fe_opt = ggml_get_f32_1d(e, 0);
    // 打印结果
    printf("%s: original  e = %.4f\n", __func__, fe);
    printf("%s: optimized e = %.4f\n", __func__, fe_opt);

    // 判断优化后的值是否小于等于原始值
    const bool success = (fe_opt <= fe);
    assert(success);

    // 释放上下文
    ggml_free(ctx);
    // 返回成功或失败
    return success ? 0 : -1;
}
// 定义一个包含4个元素的int64_t数组ne3，分别为128, 256, 1, 1
// main函数原始输出e = 68371.1328
// main函数优化后输出e = 7854.4502

// 定义一个包含4个元素的int64_t数组ne1，分别为32, 128, 1, 1
// 定义一个包含4个元素的int64_t数组ne2，分别为32, 256, 1, 1
// 定义一个包含4个元素的int64_t数组ne3，分别为128, 256, 1, 1
// main函数原始输出e = 126061.1953
// main函数优化后输出e = 5451.0166

// 定义一个包含4个元素的int64_t数组ne1，分别为4, 1024, 1, 1
// 定义一个包含4个元素的int64_t数组ne2，分别为4, 2048, 1, 1
// 定义一个包含4个元素的int64_t数组ne3，分别为1024, 2048, 1, 1
// main函数原始输出e = 1620817.8750
// main函数优化后输出e = 698387.6875

// 另一次在M1上运行
// 定义一个包含4个元素的int64_t数组ne1，分别为4, 1024, 1, 1
// 定义一个包含4个元素的int64_t数组ne2，分别为4, 2048, 1, 1
// 定义一个包含4个元素的int64_t数组ne3，分别为1024, 2048, 1, 1
// main函数原始输出e = 1629595.6250
// main函数优化后输出e = 698169.1250

// 定义一个包含4个元素的int64_t数组ne1，分别为32, 1024, 1, 1
// 定义一个包含4个元素的int64_t数组ne2，分别为32, 2048, 1, 1
// 定义一个包含4个元素的int64_t数组ne3，分别为1024, 2048, 1, 1
// main函数原始输出e = 8146770.5000
// main函数优化后输出e = 651119.1250
```