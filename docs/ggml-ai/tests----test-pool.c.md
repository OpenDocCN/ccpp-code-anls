# `ggml\tests\test-pool.c`

```cpp
#include "ggml/ggml.h"

#include <string.h>
#include <stdio.h>
#include <stdlib.h>

// 创建并返回一个 ggml 上下文结构体指针
struct ggml_context* make_ctx(void) {
    // 初始化参数结构体
    struct ggml_init_params params = {
        .mem_size = 2 * 1024 * 1024,
    };

    // 调用 ggml_init 函数，传入初始化参数，返回 ggml 上下文结构体指针
    return ggml_init(params);
}

// 主函数
int main(int argc, const char** argv) {

    // 创建一个包含 1024 个 float 类型元素的数组
    float buf_f32[1024];
    for (int i = 0; i < 1024; ++i) {
        buf_f32[i] = (float)(i + 1);
    }

    // avg pool 1d
    {
        // 创建 ggml 上下文结构体指针
        struct ggml_context * ctx = make_ctx();
        // 创建一个 2 维的 float 类型张量
        struct ggml_tensor * t = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
        // 将 buf_f32 数组的数据复制到张量 t 的数据中
        memcpy(t->data, buf_f32, ggml_nbytes(t));

        // 对张量 t 进行 1 维平均池化操作
        struct ggml_tensor * t_pooled = ggml_pool_1d(ctx, t, GGML_OP_POOL_AVG, 3, 3, 0);
        // 断言池化后的张量 t_pooled 的维度符合预期
        GGML_ASSERT(t_pooled->ne[0] == 3);
        GGML_ASSERT(t_pooled->ne[1] == 2);
        GGML_ASSERT(t_pooled->ne[2] == 1);

        // 创建一个计算图
        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        // 构建前向传播计算图
        ggml_build_forward_expand(graph, t_pooled);

        // 使用上下文计算计算图
        ggml_graph_compute_with_ctx(ctx, graph, 4);

        // 获取池化后张量的数据
        const float * output = ggml_get_data_f32(t_pooled);

        // 断言输出数据符合预期
        GGML_ASSERT(output[0] == 2);
        GGML_ASSERT(output[1] == 5);
        GGML_ASSERT(output[2] == 8);
        GGML_ASSERT(output[3] == 12);
        GGML_ASSERT(output[4] == 15);
        GGML_ASSERT(output[5] == 18);

        // 释放上下文
        ggml_free(ctx);
    }

    // max pool 1d
    {
        // 创建上下文对象
        struct ggml_context * ctx = make_ctx();
        // 创建一个包含10行2列的浮点型二维张量
        struct ggml_tensor * t = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 10, 2);
        // 将 buf_f32 中的数据复制到张量 t 的数据中
        memcpy(t->data, buf_f32, ggml_nbytes(t));

        // 对 t 进行一维最大池化操作，池化窗口大小为3，步长为3，填充为0
        struct ggml_tensor * t_pooled = ggml_pool_1d(ctx, t, GGML_OP_POOL_MAX, 3, 3, 0);
        // 断言池化后张量的维度是否符合预期
        GGML_ASSERT(t_pooled->ne[0] == 3);
        GGML_ASSERT(t_pooled->ne[1] == 2);
        GGML_ASSERT(t_pooled->ne[2] == 1);

        // 创建计算图对象
        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        // 构建前向扩展计算图
        ggml_build_forward_expand(graph, t_pooled);

        // 使用上下文对象计算计算图，迭代4次
        ggml_graph_compute_with_ctx(ctx, graph, 4);

        // 获取池化后张量的数据，并进行断言判断
        const float * output = ggml_get_data_f32(t_pooled);
        GGML_ASSERT(output[0] == 3);
        GGML_ASSERT(output[1] == 6);
        GGML_ASSERT(output[2] == 9);
        GGML_ASSERT(output[3] == 13);
        GGML_ASSERT(output[4] == 16);
        GGML_ASSERT(output[5] == 19);

        // 释放上下文对象及其相关资源
        ggml_free(ctx);
    }

    // avg pool 2d
    {
        // 创建上下文对象
        struct ggml_context * ctx = make_ctx();
        // 创建一个三维的浮点型张量对象
        struct ggml_tensor * t = ggml_new_tensor_3d(ctx, GGML_TYPE_F32, 10, 10, 2);
        // 将 buf_f32 中的数据复制到张量对象的数据中
        memcpy(t->data, buf_f32, ggml_nbytes(t));

        // 对张量对象进行二维平均池化操作
        struct ggml_tensor * t_pooled = ggml_pool_2d(ctx, t, GGML_OP_POOL_AVG, 3, 4, 3, 4, 0, 0);
        // 断言池化后的张量的维度是否符合预期
        GGML_ASSERT(t_pooled->ne[0] == 3);
        GGML_ASSERT(t_pooled->ne[1] == 2);
        GGML_ASSERT(t_pooled->ne[2] == 2);
        GGML_ASSERT(t_pooled->ne[3] == 1);

        // 创建计算图对象
        struct ggml_cgraph * graph = ggml_new_graph(ctx);
        // 构建前向扩展计算图
        ggml_build_forward_expand(graph, t_pooled);

        // 使用上下文对象进行计算图的计算
        ggml_graph_compute_with_ctx(ctx, graph, 4);

        // 获取池化后张量的数据
        const float * output = ggml_get_data_f32(t_pooled);
        // 断言池化后的张量的部分数据是否符合预期
        GGML_ASSERT(output[0] == 17);
        GGML_ASSERT(output[1] == 20);
        GGML_ASSERT(output[2] == 23);
        GGML_ASSERT(output[3] == 57);
        GGML_ASSERT(output[4] == 60);
        GGML_ASSERT(output[5] == 63);
        GGML_ASSERT(output[6] == 117);
        GGML_ASSERT(output[7] == 120);
        GGML_ASSERT(output[8] == 123);
        GGML_ASSERT(output[9] == 157);
        GGML_ASSERT(output[10] == 160);
        GGML_ASSERT(output[11] == 163);

        // 释放上下文对象及其关联的资源
        ggml_free(ctx);
    }

    // max pool 2d
    # 创建一个 ggml_context 结构体指针，并初始化
    struct ggml_context * ctx = make_ctx();
    # 创建一个 3 维的 ggml_tensor 结构体指针，并初始化为 float 类型，大小为 10x10x2
    struct ggml_tensor * t = ggml_new_tensor_3d(ctx, GGML_TYPE_F32, 10, 10, 2);
    # 将 buf_f32 中的数据拷贝到 t->data 中
    memcpy(t->data, buf_f32, ggml_nbytes(t));

    # 对 t 进行 2D 池化操作，得到 t_pooled
    struct ggml_tensor * t_pooled = ggml_pool_2d(ctx, t, GGML_OP_POOL_MAX, 3, 4, 3, 4, 0, 0);
    # 检查 t_pooled 的维度是否符合预期
    GGML_ASSERT(t_pooled->ne[0] == 3);
    GGML_ASSERT(t_pooled->ne[1] == 2);
    GGML_ASSERT(t_pooled->ne[2] == 2);
    GGML_ASSERT(t_pooled->ne[3] == 1);

    # 创建一个新的计算图
    struct ggml_cgraph * graph = ggml_new_graph(ctx);
    # 构建前向传播计算图
    ggml_build_forward_expand(graph, t_pooled);

    # 使用给定的上下文和计算图进行计算
    ggml_graph_compute_with_ctx(ctx, graph, 4);

    # 获取 t_pooled 的数据，并进行断言检查
    const float * output = ggml_get_data_f32(t_pooled);
    GGML_ASSERT(output[0] == 33);
    GGML_ASSERT(output[1] == 36);
    GGML_ASSERT(output[2] == 39);
    GGML_ASSERT(output[3] == 73);
    GGML_ASSERT(output[4] == 76);
    GGML_ASSERT(output[5] == 79);
    GGML_ASSERT(output[6] == 133);
    GGML_ASSERT(output[7] == 136);
    GGML_ASSERT(output[8] == 139);
    GGML_ASSERT(output[9] == 173);
    GGML_ASSERT(output[10] == 176);
    GGML_ASSERT(output[11] == 179);

    # 释放上下文所占用的内存
    ggml_free(ctx);

    # 返回 0，表示程序执行成功
    return 0;
# 闭合前面的函数定义
```