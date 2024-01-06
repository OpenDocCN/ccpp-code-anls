# `PowerInfer\tests\test-opt.cpp`

```
// 包含 ggml.h 头文件
#include "ggml.h"

// 包含数学函数头文件
#include <cmath>
// 包含标准输入输出头文件
#include <cstdio>
// 包含标准库头文件
#include <cstdlib>
// 包含断言头文件
#include <cassert>

// 定义最大参数个数为 2
#define MAX_NARGS 2

// 如果使用的是 GNU 编译器，忽略双精度提升的警告
#if defined(__GNUC__)
#pragma GCC diagnostic ignored "-Wdouble-promotion"
#endif

//
// logging
//

// 定义 GGML_DEBUG 为 0
#define GGML_DEBUG 0
// 如果 GGML_DEBUG 大于等于 1，则打印调试信息
#if (GGML_DEBUG >= 1)
#define GGML_PRINT_DEBUG(...) printf(__VA_ARGS__)
// 否则，不打印调试信息
#else
// 定义宏 GGML_PRINT_DEBUG，用于打印调试信息，但不包括参数
#endif

#if (GGML_DEBUG >= 5)
// 如果 GGML_DEBUG 大于等于 5，则定义宏 GGML_PRINT_DEBUG_5，用于打印调试信息，包括参数
#define GGML_PRINT_DEBUG_5(...) printf(__VA_ARGS__)
#else
// 否则定义为空
#define GGML_PRINT_DEBUG_5(...)
#endif

#if (GGML_DEBUG >= 10)
// 如果 GGML_DEBUG 大于等于 10，则定义宏 GGML_PRINT_DEBUG_10，用于打印调试信息，包括参数
#define GGML_PRINT_DEBUG_10(...) printf(__VA_ARGS__)
#else
// 否则定义为空
#define GGML_PRINT_DEBUG_10(...)
#endif

// 定义宏 GGML_PRINT，用于打印信息，包括参数
#define GGML_PRINT(...) printf(__VA_ARGS__)

// 定义静态函数 frand，返回一个 0 到 1 之间的随机浮点数
static float frand(void) {
    return (float)rand()/(float)RAND_MAX;
// 获取一个随机数张量
static struct ggml_tensor * get_random_tensor(
    struct ggml_context * ctx0, int ndims, int64_t ne[], float fmin, float fmax
) {
    // 创建一个新的张量对象
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F32, ndims, ne);

    // 根据张量的维度进行不同的处理
    switch (ndims) {
        // 当维度为1时
        case 1:
            // 遍历张量的第一维度，给每个元素赋予一个随机数
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)result->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
        // 当维度为2时
        case 2:
            // 遍历张量的第二维度和第一维度，给每个元素赋予一个随机数
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((float *)result->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
            break;
# 根据不同的情况，使用嵌套循环为结果数组赋值
case 3:
    # 对于三维情况，使用三重循环为结果数组赋值
    for (int i2 = 0; i2 < ne[2]; i2++) {
        for (int i1 = 0; i1 < ne[1]; i1++) {
            for (int i0 = 0; i0 < ne[0]; i0++) {
                # 根据随机数生成的方式为结果数组赋值
                ((float *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
            }
        }
    }
    break;
case 4:
    # 对于四维情况，使用四重循环为结果数组赋值
    for (int i3 = 0; i3 < ne[3]; i3++) {
        for (int i2 = 0; i2 < ne[2]; i2++) {
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    # 根据随机数生成的方式为结果数组赋值
                    ((float *)result->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
        }
    }
    break;
    // 默认情况下，如果程序执行到这里，会触发断言错误
    default:
        assert(false);
}

// 主函数
int main(void) {
    // 初始化参数结构体
    struct ggml_init_params params = {
        /* .mem_size   = */ 1024*1024*1024,  // 内存大小为 1GB
        /* .mem_buffer = */ NULL,            // 内存缓冲区为空
        /* .no_alloc   = */ false,           // 不禁止分配内存
    };

    // 初始化 ggml 上下文
    struct ggml_context * ctx = ggml_init(params);

    // 定义三个长度为 4 的整型数组
    int64_t ne1[4] = {4, 128, 1, 1};
    int64_t ne2[4] = {4, 256, 1, 1};
    int64_t ne3[4] = {128, 256, 1, 1};
# 创建一个随机的张量a，包含2个元素，范围在-1到+1之间
struct ggml_tensor * a = get_random_tensor(ctx, 2, ne1, -1, +1);
# 创建一个随机的张量b，包含2个元素，范围在-1到+1之间
struct ggml_tensor * b = get_random_tensor(ctx, 2, ne2, -1, +1);
# 设置a为参数
ggml_set_param(ctx, a);
# 设置b为参数
ggml_set_param(ctx, b);

# 创建一个随机的张量c，包含2个元素，范围在-1到+1之间
struct ggml_tensor * c = get_random_tensor(ctx, 2, ne3, -1, +1);

# 计算a和b的矩阵乘积
struct ggml_tensor * ab = ggml_mul_mat(ctx, a, b);
# 计算c和ab的差
struct ggml_tensor * d  = ggml_sub(ctx, c, ab);
# 计算d的平方和
struct ggml_tensor * e  = ggml_sum(ctx, ggml_sqr(ctx, d));

# 创建一个新的计算图ge
struct ggml_cgraph * ge = ggml_new_graph_custom(ctx, GGML_DEFAULT_GRAPH_SIZE, true);
# 构建前向传播
ggml_build_forward_expand(ge, e);
# 重置计算图
ggml_graph_reset(ge);

# 使用上下文ctx计算计算图ge，使用1个线程
ggml_graph_compute_with_ctx(ctx, ge, /*n_threads*/ 1);

# 获取张量e的第一个元素的值
const float fe = ggml_get_f32_1d(e, 0);
# 打印函数名和张量e的值
printf("%s: e = %.4f\n", __func__, fe);
// 使用默认参数创建一个优化器参数结构体
struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_ADAM);

// 使用上面创建的优化器参数结构体进行优化
ggml_opt(ctx, opt_params, e);

// 重置图形引擎
ggml_graph_reset(ge);

// 使用上下文和图形引擎计算，使用一个线程
ggml_graph_compute_with_ctx(ctx, ge, /*n_threads*/ 1);

// 从 e 中获取一个浮点数，并打印原始 e 的值
const float fe_opt = ggml_get_f32_1d(e, 0);
printf("%s: original  e = %.4f\n", __func__, fe);

// 打印优化后的 e 的值
printf("%s: optimized e = %.4f\n", __func__, fe_opt);

// 检查优化后的 e 是否小于等于原始 e
const bool success = (fe_opt <= fe);
assert(success);

// 释放上下文
ggml_free(ctx);

// 如果成功则返回 0，否则返回 -1
return success ? 0 : -1;
}
// 下面是注释掉的代码，不会被执行
// int64_t ne1[4] = {4, 128, 1, 1};
// int64_t ne2[4] = {4, 256, 1, 1};;
// 定义一个长度为4的int64_t类型数组ne3，并初始化为{128, 256, 1, 1}
// main函数：原始值 e = 25890.9375
// main函数：优化后值 e = 10094.7031

// 定义一个长度为4的int64_t类型数组ne1，并初始化为{8, 128, 1, 1}
// 定义一个长度为4的int64_t类型数组ne2，并初始化为{8, 256, 1, 1}
// 定义一个长度为4的int64_t类型数组ne3，并初始化为{128, 256, 1, 1}
// main函数：原始值 e = 39429.5078
// main函数：优化后值 e = 9275.8936

// 定义一个长度为4的int64_t类型数组ne1，并初始化为{16, 128, 1, 1}
// 定义一个长度为4的int64_t类型数组ne2，并初始化为{16, 256, 1, 1}
// 定义一个长度为4的int64_t类型数组ne3，并初始化为{128, 256, 1, 1}
// main函数：原始值 e = 68371.1328
// main函数：优化后值 e = 7854.4502

// 定义一个长度为4的int64_t类型数组ne1，并初始化为{32, 128, 1, 1}
// 定义一个长度为4的int64_t类型数组ne2，并初始化为{32, 256, 1, 1}
// 定义一个长度为4的int64_t类型数组ne3，并初始化为{128, 256, 1, 1}
// main: original  e = 126061.1953
// main: optimized e = 5451.0166

// 定义三个长度为4的整型数组，分别表示不同参数下的性能测试
// int64_t ne1[4] = {4, 1024, 1, 1};
// int64_t ne2[4] = {4, 2048, 1, 1};;
// int64_t ne3[4] = {1024, 2048, 1, 1};
// main: original  e = 1620817.8750
// main: optimized e = 698387.6875

// 另一次在 M1 上运行
// int64_t ne1[4] = {4, 1024, 1, 1};
// int64_t ne2[4] = {4, 2048, 1, 1};;
// int64_t ne3[4] = {1024, 2048, 1, 1};
// main: original  e = 1629595.6250
// main: optimized e = 698169.1250

// 重新定义三个长度为4的整型数组，分别表示不同参数下的性能测试
// int64_t ne1[4] = {32, 1024, 1, 1};
// int64_t ne2[4] = {32, 2048, 1, 1};;
// int64_t ne3[4] = {1024, 2048, 1, 1};
// main: original  e = 8146770.5000
// 这是一个注释，用于说明下面的代码是主要的优化部分，e的值为651119.1250
```