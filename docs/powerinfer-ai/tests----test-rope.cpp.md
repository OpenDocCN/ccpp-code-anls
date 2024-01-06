# `PowerInfer\tests\test-rope.cpp`

```
#include "ggml.h"
// 包含自定义的头文件 ggml.h

#include <cmath>
// 包含数学函数库

#include <cstdio>
// 包含输入输出函数库

#include <cstdlib>
// 包含标准库函数库

#include <cassert>
// 包含断言函数库

#include <vector>
// 包含向量容器类模板

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif
// 如果是在 MSC 编译器下，禁止警告 4244 和 4267，可能会丢失数据

#if defined(__GNUC__)
#pragma GCC diagnostic ignored "-Wdouble-promotion"
#endif
// 如果是在 GNU 编译器下，忽略 -Wdouble-promotion 警告

#define MAX_NARGS 3
// 定义宏 MAX_NARGS 为 3

#undef MIN
// 取消之前定义的宏 MIN
#undef MAX
// 取消之前定义的宏 MAX
// 定义一个宏，用于返回两个数中的较小值
#define MIN(a, b) ((a) < (b) ? (a) : (b))
// 定义一个宏，用于返回两个数中的较大值
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 定义一个宏，用于指定 GGML 使用 FP16 数据类型
#define GGML_SILU_FP16

//
// logging
//

// 如果 GGML_DEBUG 大于等于 1，则定义一个宏，用于打印调试信息
#if (GGML_DEBUG >= 1)
#define GGML_PRINT_DEBUG(...) printf(__VA_ARGS__)
// 否则，定义为空的宏
#else
#define GGML_PRINT_DEBUG(...)
#endif

// 如果 GGML_DEBUG 大于等于 5，则定义一个宏，用于打印详细的调试信息
#if (GGML_DEBUG >= 5)
#define GGML_PRINT_DEBUG_5(...) printf(__VA_ARGS__)
// 否则，定义为空的宏
#else
#define GGML_PRINT_DEBUG_5(...)
#endif
#if (GGML_DEBUG >= 10)
// 如果 GGML_DEBUG 大于等于 10，则定义 GGML_PRINT_DEBUG_10 宏为 printf 函数
#define GGML_PRINT_DEBUG_10(...) printf(__VA_ARGS__)
#else
// 否则定义 GGML_PRINT_DEBUG_10 宏为空
#define GGML_PRINT_DEBUG_10(...)
#endif

// 定义 GGML_PRINT 宏为 printf 函数
#define GGML_PRINT(...) printf(__VA_ARGS__)

// 生成一个 0 到 1 之间的随机浮点数
static float frand(void) {
    return (float)rand()/(float)RAND_MAX;
}

// 生成一个 0 到 n-1 之间的随机整数
static int irand(int n) {
    if (n == 0) return 0;
    return rand()%n;
}

// 将维度数组的前四个元素设置为 1
static void get_random_dims(int64_t * dims, int ndims) {
    dims[0] = dims[1] = dims[2] = dims[3] = 1;
```
// 遍历ndims维度，为每个维度生成一个随机数作为该维度的大小
for (int i = 0; i < ndims; i++) {
    dims[i] = 1 + irand(4);
}

// 生成一个随机的float类型的tensor
static struct ggml_tensor * get_random_tensor_f32(
        struct ggml_context * ctx0,
        int ndims,
        const int64_t ne[],
        float fmin,
        float fmax) {
    // 创建一个新的tensor对象
    struct ggml_tensor * result = ggml_new_tensor(ctx0, GGML_TYPE_F32, ndims, ne);

    // 根据不同维度的情况，生成随机的float数值填充到tensor中
    switch (ndims) {
        case 1:
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)result->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
# 当 case 为 2 时，使用嵌套循环遍历二维数组，对每个元素赋值为随机数乘以范围加上最小值
case 2:
    for (int i1 = 0; i1 < ne[1]; i1++) {
        for (int i0 = 0; i0 < ne[0]; i0++) {
            ((float *)result->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
        }
    }
    break;
# 当 case 为 3 时，使用嵌套循环遍历三维数组，对每个元素赋值为随机数乘以范围加上最小值
case 3:
    for (int i2 = 0; i2 < ne[2]; i2++) {
        for (int i1 = 0; i1 < ne[1]; i1++) {
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)result->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
            }
        }
    }
    break;
# 当 case 为 4 时，使用嵌套循环遍历四维数组，对每个元素赋值为随机数乘以范围加上最小值
case 4:
    for (int i3 = 0; i3 < ne[3]; i3++) {
        for (int i2 = 0; i2 < ne[2]; i2++) {
            for (int i1 = 0; i1 < ne[1]; i1++) {
// 循环遍历ne[0]次，i0从0到ne[0]-1
for (int i0 = 0; i0 < ne[0]; i0++) {
    // 将随机生成的浮点数乘以(fmax - fmin)，再加上fmin，赋值给result->data中的指定位置
    ((float *)result->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
}
// 结束i0循环

// 结束i1循环

// 结束i2循环

// 结束i3循环

// 结束switch语句

// 返回result

// 根据图形计划和线程数计算结果
static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    // 根据图形和线程数创建计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划的工作大小大于0
    if (plan.work_size > 0) {
        // 调整buf的大小为计划的工作大小
        buf.resize(plan.work_size);
        // 将计划的工作数据指针指向buf的数据
        plan.work_data = buf.data();
    }

    ggml_graph_compute(graph, &plan);
}

int main(int /*argc*/, const char ** /*argv*/) {
    // 初始化参数结构体，设置内存大小为128MB，内存缓冲区为空，允许分配内存
    struct ggml_init_params params = {
        /* .mem_size   = */ 128*1024*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ false,
    };

    // 创建一个存储uint8_t类型数据的向量
    std::vector<uint8_t> work_buffer;

    // 初始化ggml上下文，传入初始化参数
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建一个ggml_tensor结构体指针x

    // 使用f32类型的rope进行循环，循环次数为3次
    for (int m = 0; m < 3; ++m) {
// 定义一个常量，表示张量的维度
const int ndims = 4;

// 定义四个常量，表示每个维度的大小
const int64_t n_rot = 128;
const int64_t ne[4] = { 2*n_rot, 32, 73, 1 };

// 定义两个常量，表示过去的时间步
const int n_past_0 = 100;
const int n_past_2 = 33;

// 创建三个一维整型张量
struct ggml_tensor * p0 = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, ne[2]);
struct ggml_tensor * p1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, ne[2]);
struct ggml_tensor * p2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, ne[2]);

// 为每个张量赋值
for (int i = 0; i < ne[2]; ++i) {
    ((int32_t *) p0->data)[i] = n_past_0 + i;
    ((int32_t *) p1->data)[i] = n_past_2 - n_past_0;
    ((int32_t *) p2->data)[i] = n_past_2 + i;
}

// 根据条件设置不同的模式
const int mode = m == 0 ? 0 : m == 1 ? 2 : 4;
// 生成一个随机的浮点数张量，范围在-1.0到1.0之间
x = get_random_tensor_f32(ctx0, ndims, ne, -1.0f, 1.0f);

// 使用 ggml_rope 函数创建一个新的张量 r0，对 x 进行操作
// 参数 p0, n_rot, mode, 1024 分别表示...
struct ggml_tensor * r0 = ggml_rope(ctx0, x,  p0, n_rot, mode, 1024);

// 使用 ggml_rope 函数创建一个新的张量 r1，对 r0 进行操作
// 参数 p1, n_rot, mode, 1024 分别表示...
// "context swap" 表示忘记 n_past_0 - n_past_2 个标记
struct ggml_tensor * r1 = ggml_rope(ctx0, r0, p1, n_rot, mode, 1024);

// 使用 ggml_rope 函数创建一个新的张量 r2，对 x 进行操作
// 参数 p2, n_rot, mode, 1024 分别表示...
struct ggml_tensor * r2 = ggml_rope(ctx0, x,  p2, n_rot, mode, 1024);

// 创建一个新的计算图
ggml_cgraph * gf = ggml_new_graph(ctx0);

// 在计算图中构建 r0 的前向扩展
ggml_build_forward_expand(gf, r0);
// 在计算图中构建 r1 的前向扩展
ggml_build_forward_expand(gf, r1);
// 在计算图中构建 r2 的前向扩展
ggml_build_forward_expand(gf, r2);

// 使用计算图计算结果
ggml_graph_compute_helper(work_buffer, gf, 4);

// 检查 r1 和 r2 是否相同
        {
            // 初始化三个变量，用于存储计算结果
            double sum0 = 0.0f;
            double sum1 = 0.0f;
            double diff = 0.0f;

            // 将指针 r1->data 强制转换为 float 类型指针，并赋值给 r1_data
            const float * r1_data = (float *) r1->data;
            // 将指针 r2->data 强制转换为 float 类型指针，并赋值给 r2_data
            const float * r2_data = (float *) r2->data;

            // 获取 r1 中元素的数量
            const int n_elements = ggml_nelements(r1);

            // 遍历 r1 和 r2 中的元素进行计算
            for (int i = 0; i < n_elements; ++i) {
                // 计算 r1_data 中元素的绝对值并累加到 sum0
                sum0 += fabs(r1_data[i]);
                // 计算 r2_data 中元素的绝对值并累加到 sum1
                sum1 += fabs(r2_data[i]);
                // 计算 r1_data 和 r2_data 中对应元素的差的绝对值并累加到 diff
                diff += fabs(r1_data[i] - r2_data[i]);
                // 如果 r1_data 和 r2_data 中对应元素的差的绝对值大于 0.0001f，则输出相关信息
                //if (fabs(r1_data[i] - r2_data[i]) > 0.0001f) {
                //    printf("%d: %f %f\n", i, r1_data[i], r2_data[i]);
                //    printf("diff: %f\n", fabs(r1_data[i] - r2_data[i]));
                //}
            }
            //for (int i = 4096; i < 4096 + 128; ++i) {
            //    printf("%f %f\n", r1_data[i], r2_data[i]);
            //}

            // 打印模式的值
            printf("mode: %d\n", mode);
            // 打印 sum0 的值
            printf("sum0: %f\n", sum0);
            // 打印 sum1 的值
            printf("sum1: %f\n", sum1);
            // 打印 diff 的值
            printf("diff: %f\n", diff);
            // 打印相对误差（diff 除以 sum0）的值
            printf("rel err: %f\n", diff / sum0);
            // 打印相对误差（diff 除以 sum1）的值
            printf("rel err: %f\n", diff / sum1);

            // 断言相对误差（diff 除以 sum0）小于 0.0001
            GGML_ASSERT(diff / sum0 < 0.0001f);
            // 断言相对误差（diff 除以 sum1）小于 0.0001
            GGML_ASSERT(diff / sum1 < 0.0001f);
        }
    }

    // 释放内存
    ggml_free(ctx0);

    // 返回 0 表示成功
    return 0;
}
抱歉，我无法为您提供代码注释，因为我无法识别您想要解释的代码。 但是，如果您有任何编程问题，我会很乐意帮助您解决问题。
```