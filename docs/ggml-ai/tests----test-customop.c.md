# `ggml\tests\test-customop.c`

```
#include "ggml/ggml.h"  // 包含 ggml 库的头文件
#include <string.h>  // 包含字符串处理函数的头文件
#include <stdio.h>  // 包含标准输入输出函数的头文件
#include <stdlib.h>  // 包含标准库函数的头文件
#include <assert.h>  // 包含断言函数的头文件

#if defined(_WIN32)
#include <windows.h>  // 如果是 Windows 系统，包含 Windows API 的头文件
typedef volatile LONG atomic_int;  // 定义原子整型类型
static LONG atomic_fetch_add(atomic_int * ptr, LONG inc) {  // 定义原子加法操作函数
    return InterlockedExchangeAdd(ptr, inc);
}
#else
#include <stdatomic.h>  // 如果不是 Windows 系统，包含 C11 原子操作的头文件
#endif

#define MIN(a, b) ((a) < (b) ? (a) : (b))  // 定义取最小值的宏
#define MAX(a, b) ((a) > (b) ? (a) : (b))  // 定义取最大值的宏

struct ggml_context * make_ctx(void) {  // 定义创建上下文的函数
    struct ggml_init_params params = {  // 初始化 ggml_init_params 结构体
        /*.mem_size   =*/ 1 * 1024 * 1024,  // 设置内存大小为 1MB
        /*.mem_buffer =*/ NULL,  // 内存缓冲区为空
        /*.no_alloc   =*/ false,  // 不禁用内存分配
    };

    return ggml_init(params);  // 调用 ggml_init 函数初始化上下文
}

char g_userdata[] = "ggml";  // 定义全局用户数据数组并初始化为 "ggml"
atomic_int g_custom1_count = 0;  // 定义原子整型变量并初始化为 0
atomic_int g_custom2_count = 0;  // 定义原子整型变量并初始化为 0
atomic_int g_custom3_count = 0;  // 定义原子整型变量并初始化为 0

void custom1(struct ggml_tensor * dst , const struct ggml_tensor * a, int ith, int nth, void * userdata) {  // 定义自定义函数 custom1
    // 检查用户数据是否正确
    assert(userdata == NULL);  // 断言用户数据为空
    assert(ggml_are_same_shape(dst, a));  // 断言目标张量和输入张量形状相同

    atomic_fetch_add(&g_custom1_count, 1);  // 原子操作，自定义函数1计数加1

    const float * a_data = ggml_get_data_f32(a);  // 获取输入张量的浮点数据
    float * dst_data = ggml_get_data_f32(dst);  // 获取目标张量的浮点数据

    // 假设张量是连续的
    assert(ggml_is_contiguous(dst));  // 断言目标张量是连续的
    assert(ggml_is_contiguous(a));  // 断言输入张量是连续的

    // 按元素并行处理
    const int ne = (int)ggml_nelements(dst);  // 获取目标张量的元素个数
    const int dr = (ne + nth - 1) / nth;  // 计算每个线程处理的元素个数
    const int ie0 = dr * ith;  // 计算当前线程处理的起始索引
    const int ie1 = MIN(ie0 + dr, ne);  // 计算当前线程处理的结束索引

    for (int i = ie0; i < ie1; ++i) {  // 遍历处理元素
        dst_data[i] = a_data[i] * 2;  // 目标张量的元素值等于输入张量的元素值乘以2
    }
}

void custom2(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, int ith, int nth, void * userdata) {  // 定义自定义函数 custom2
    // 检查用户数据是否正确
    assert(userdata == g_userdata);  // 断言用户数据等于全局用户数据数组
    assert(strcmp(userdata, "ggml") == 0);  // 断言用户数据等于 "ggml"
    assert(ggml_are_same_shape(dst, a));  // 断言目标张量和输入张量a的形状相同
    assert(ggml_are_same_shape(dst, b));  // 断言目标张量和输入张量b的形状相同

    atomic_fetch_add(&g_custom2_count, 1);  // 原子操作，自定义函数2计数加1

    const float * a_data = ggml_get_data_f32(a);  // 获取输入张量a的浮点数据
    const float * b_data = ggml_get_data_f32(b);  // 获取输入张量b的浮点数据
    // 获取目标数据的浮点型指针
    float * dst_data = ggml_get_data_f32(dst);

    // 按行并行化处理
    const int nr = (int)ggml_nrows(dst);
    // 每个线程处理的行数
    const int dr = (nr + nth - 1) / nth;
    // 该线程处理的行范围
    const int ir0 = dr * ith;
    const int ir1 = MIN(ir0 + dr, nr);

    // 列数
    const int nc = (int)dst->ne[0];

    // 这里假设张量是连续的
    assert(ggml_is_contiguous(dst));
    assert(ggml_is_contiguous(a));
    assert(ggml_is_contiguous(b));

    for (int ir = ir0; ir < ir1; ++ir) {
        for (int ic = 0; ic < nc; ++ic) {
            const int i = ir * nc + ic;
            dst_data[i] = a_data[i] + b_data[i];
        }
    }
void custom3(struct ggml_tensor * dst , const struct ggml_tensor * a, const struct ggml_tensor * b, const struct ggml_tensor * c, int ith, int nth, void * userdata) {
    // 检查 userdata 是否正确
    assert(userdata == g_userdata);
    // 检查 userdata 是否为 "ggml"
    assert(strcmp(userdata, "ggml") == 0);
    // 检查 dst 和 a 的形状是否相同
    assert(ggml_are_same_shape(dst, a));
    // 检查 dst 和 b 的形状是否相同
    assert(ggml_are_same_shape(dst, b));
    // 检查 dst 和 c 的形状是否相同
    assert(ggml_are_same_shape(dst, c));

    // 原子操作，将 g_custom3_count 加一
    atomic_fetch_add(&g_custom3_count, 1);

    // 获取 a, b, c, dst 的数据指针
    const float * a_data = ggml_get_data_f32(a);
    const float * b_data = ggml_get_data_f32(b);
    const float * c_data = ggml_get_data_f32(c);
    float * dst_data = ggml_get_data_f32(dst);

    // 不并行化，确保 ith 为 0
    assert(ith == 0);

    // 元素的数量
    const int ne = (int)ggml_nelements(dst);

    // 假设张量是连续的，进行断言检查
    assert(ggml_is_contiguous(dst));
    assert(ggml_is_contiguous(a));
    assert(ggml_is_contiguous(b));
    assert(ggml_is_contiguous(c));

    // 循环遍历每个元素，计算 dst_data[i] = a_data[i] + b_data[i] + c_data[i]
    for (int i = 0; i < ne; ++i) {
        dst_data[i] = a_data[i] + b_data[i] + c_data[i];
    }
}

int main(int argc, const char** argv) {

    // 创建三个长度为 1024 的 float 数组，并初始化
    float buf1_f32[1024];
    for (int i = 0; i < 1024; ++i) {
        buf1_f32[i] = (float)(i + 1);
    }
    float buf2_f32[1024];
    for (int i = 0; i < 1024; ++i) {
        buf2_f32[i] = (float)(i + 1) * 2;
    }
    float buf3_f32[1024];
    for (int i = 0; i < 1024; ++i) {
        buf3_f32[i] = (float)(i + 1) * 3;
    }

    // map_custom1
    // 2 个任务，没有 userdata，按元素并行化
    {
        // 创建一个上下文对象
        struct ggml_context * ctx = make_ctx();
        // 创建一个包含10行2列的浮点型张量对象
        struct ggml_tensor * t = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
        // 将 buf1_f32 的数据复制到张量对象 t 的数据中
        memcpy(t->data, buf1_f32, ggml_nbytes(t));

        // 对张量 t 进行自定义操作 custom1，返回一个新的张量对象 m1
        struct ggml_tensor * m1 = ggml_map_custom1(ctx, t, custom1, 2, NULL);

        // 创建一个计算图对象
        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        // 将 m1 添加到计算图中
        ggml_build_forward_expand(graph, m1);

        // 使用上下文对象 ctx 运行计算图，最多并行 4 个任务
        ggml_graph_compute_with_ctx(ctx, graph, 4);

        // 获取张量 m1 的数据
        const float * output = ggml_get_data_f32(m1);

        // 遍历张量 m1 的每个元素，检查是否满足条件
        for (int i = 0; i < ggml_nelements(m1); ++i) {
            assert(output[i] == buf1_f32[i] * 2);
        }
        // 断言自定义操作 custom1 的调用次数为 2
        assert(g_custom1_count == 2);

        // 释放上下文对象 ctx
        ggml_free(ctx);
    }

    // map_custom2
    // 最大任务数为 4，用户数据为 g_userdata，按行并行化
    {
        // 创建一个上下文对象
        struct ggml_context * ctx = make_ctx();
        // 创建一个包含10行2列的浮点型张量对象 t1
        struct ggml_tensor * t1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
        // 将 buf1_f32 的数据复制到张量对象 t1 的数据中
        memcpy(t1->data, buf1_f32, ggml_nbytes(t1));
        // 创建一个包含10行2列的浮点型张量对象 t2
        struct ggml_tensor * t2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
        // 将 buf2_f32 的数据复制到张量对象 t2 的数据中
        memcpy(t2->data, buf2_f32, ggml_nbytes(t2));

        // 对张量 t1 和 t2 进行自定义操作 custom2，返回一个新的张量对象 m2
        struct ggml_tensor * m2 = ggml_map_custom2(ctx, t1, t2, custom2, GGML_N_TASKS_MAX, g_userdata);

        // 创建一个计算图对象
        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        // 将 m2 添加到计算图中
        ggml_build_forward_expand(graph, m2);

        // 使用上下文对象 ctx 运行计算图，最多并行 4 个任务
        ggml_graph_compute_with_ctx(ctx, graph, 4);

        // 获取张量 m2 的数据
        const float * output = ggml_get_data_f32(m2);

        // 遍历张量 m2 的每个元素，检查是否满足条件
        for (int i = 0; i < ggml_nelements(m2); ++i) {
            assert(output[i] == buf1_f32[i] + buf2_f32[i]);
        }

        // 断言自定义操作 custom2 的调用次数为 4
        assert(g_custom2_count == 4);

        // 释放上下文对象 ctx
        ggml_free(ctx);
    }

    // map_custom3
    // 1 个任务，用户数据，不并行化
    # 创建一个 ggml 上下文结构体
    struct ggml_context * ctx = make_ctx();
    # 创建一个包含 10 行 2 列的 float 类型的二维张量 t1
    struct ggml_tensor * t1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
    # 将 buf1_f32 的数据拷贝到 t1 的数据中
    memcpy(t1->data, buf1_f32, ggml_nbytes(t1));
    # 创建一个包含 10 行 2 列的 float 类型的二维张量 t2
    struct ggml_tensor * t2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
    # 将 buf2_f32 的数据拷贝到 t2 的数据中
    memcpy(t2->data, buf2_f32, ggml_nbytes(t2));
    # 创建一个包含 10 行 2 列的 float 类型的二维张量 t3
    struct ggml_tensor * t3 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
    # 将 buf3_f32 的数据拷贝到 t3 的数据中
    memcpy(t3->data, buf3_f32, ggml_nbytes(t3));

    # 使用自定义函数 custom3 对 t1, t2, t3 进行映射操作，得到结果张量 m3
    struct ggml_tensor * m3 = ggml_map_custom3(ctx, t1, t2, t3, custom3, 1, g_userdata);

    # 创建一个新的计算图对象 graph
    struct ggml_cgraph * graph = ggml_new_graph(ctx);
    # 构建前向扩展计算图，将 m3 加入到计算图中
    ggml_build_forward_expand(graph, m3);

    # 使用上下文 ctx 进行计算图的计算，使用 4 个线程
    ggml_graph_compute_with_ctx(ctx, graph, 4);

    # 获取 m3 的数据，并赋值给 output
    const float * output = ggml_get_data_f32(m3);

    # 遍历输出结果，对每个元素进行断言判断
    for (int i = 0; i < ggml_nelements(m3); ++i) {
        assert(output[i] == buf1_f32[i] + buf2_f32[i] + buf3_f32[i]);
    }

    # 断言自定义函数 custom3 被调用一次
    assert(g_custom3_count == 1);

    # 释放上下文 ctx
    ggml_free(ctx);
    
    # 返回 0，表示执行成功
    return 0;
# 闭合前面的函数定义
```