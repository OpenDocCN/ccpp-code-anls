# `ggml\tests\test1.c`

```
#include "ggml/ggml.h"

#include <stdio.h>
#include <stdlib.h>

int main(int argc, const char ** argv) {
    const int n_threads = 2;

    // 初始化参数结构体，设置内存大小为128MB，内存缓冲区为空，允许分配内存
    struct ggml_init_params params = {
        .mem_size   = 128*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    // 初始化 GGML 上下文
    struct ggml_context * ctx0 = ggml_init(params);

    {
        // 创建一个一维的浮点型张量 x
        struct ggml_tensor * x = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);

        // 设置参数为 x
        ggml_set_param(ctx0, x);

        // 创建一维的浮点型张量 a
        struct ggml_tensor * a = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        // 计算 x 与 x 的乘积，得到张量 b
        struct ggml_tensor * b = ggml_mul(ctx0, x, x);
        // 计算 b 与 a 的乘积，得到张量 f
        struct ggml_tensor * f = ggml_mul(ctx0, b, a);

        // 打印对象信息
        ggml_print_objects(ctx0);

        // 创建一个自定义大小的前向计算图 gf
        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        // 构建前向计算图
        ggml_build_forward_expand(gf, f);
        // 复制前向计算图得到后向计算图 gb
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        // 构建后向计算图
        ggml_build_backward_expand(ctx0, gf, gb, false);

        // 设置张量 x 的值为 2.0
        ggml_set_f32(x, 2.0f);
        // 设置张量 a 的值为 3.0
        ggml_set_f32(a, 3.0f);

        // 重置前向计算图
        ggml_graph_reset(gf);
        // 设置张量 f 的梯度为 1.0
        ggml_set_f32(f->grad, 1.0f);

        // 使用上下文计算后向计算图
        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        // 打印 f 和 df/dx 的值
        printf("f     = %f\n", ggml_get_f32_1d(f, 0));
        printf("df/dx = %f\n", ggml_get_f32_1d(x->grad, 0));

        // 断言 f 和 df/dx 的值
        GGML_ASSERT(ggml_get_f32_1d(f, 0)       == 12.0f);
        GGML_ASSERT(ggml_get_f32_1d(x->grad, 0) == 12.0f);

        // 设置张量 x 的值为 3.0
        ggml_set_f32(x, 3.0f);

        // 重置前向计算图
        ggml_graph_reset(gf);
        // 设置张量 f 的梯度为 1.0
        ggml_set_f32(f->grad, 1.0f);

        // 使用上下文计算后向计算图
        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        // 打印 f 和 df/dx 的值
        printf("f     = %f\n", ggml_get_f32_1d(f, 0));
        printf("df/dx = %f\n", ggml_get_f32_1d(x->grad, 0));

        // 断言 f 和 df/dx 的值
        GGML_ASSERT(ggml_get_f32_1d(f, 0)       == 27.0f);
        GGML_ASSERT(ggml_get_f32_1d(x->grad, 0) == 18.0f);

        // 将前向计算图 gf 和后向计算图 gb 导出为 DOT 文件
        ggml_graph_dump_dot(gf, NULL, "test1-1-forward.dot");
        ggml_graph_dump_dot(gb, gf,   "test1-1-backward.dot");
    }

    ///////////////////////////////////////////////////////////////
}
    {
        // 创建三个一维张量 x1, x2, x3，数据类型为 GGML_TYPE_F32，长度为 1
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * x3 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
    
        // 设置张量 x1, x2, x3 的值分别为 3.0f, 1.0f, 0.0f
        ggml_set_f32(x1, 3.0f);
        ggml_set_f32(x2, 1.0f);
        ggml_set_f32(x3, 0.0f);
    
        // 将张量 x1, x2 添加到计算图中
        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);
    
        // 计算 y = x1 * x1 + x1 * x2
        struct ggml_tensor * y = ggml_add(ctx0, ggml_mul(ctx0, x1, x1), ggml_mul(ctx0, x1, x2));
    
        // 创建一个自定义大小的计算图 gf
        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        // 构建前向传播计算图
        ggml_build_forward_expand(gf, y);
        // 复制计算图 gf 到 gb
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        // 构建反向传播计算图
        ggml_build_backward_expand(ctx0, gf, gb, false);
    
        // 重置计算图 gf
        ggml_graph_reset(gf);
        // 设置张量 y 的梯度为 1.0f
        ggml_set_f32(y->grad, 1.0f);
    
        // 使用上下文 ctx0 计算计算图 gb
        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);
    
        // 打印张量 y 的值
        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        // 打印张量 x1 的梯度
        printf("df/dx1 = %f\n", ggml_get_f32_1d(x1->grad, 0));
        // 打印张量 x2 的梯度
        printf("df/dx2 = %f\n", ggml_get_f32_1d(x2->grad, 0));
    
        // 断言张量 y 的值为 12.0f
        GGML_ASSERT(ggml_get_f32_1d(y, 0)        == 12.0f);
        // 断言张量 x1 的梯度为 7.0f
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 7.0f);
        // 断言张量 x2 的梯度为 3.0f
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 3.0f);
    
        // 创建张量 g1, g2 分别指向 x1, x2 的梯度
        struct ggml_tensor * g1 = x1->grad;
        struct ggml_tensor * g2 = x2->grad;
    
        // 复制计算图 gb 到 gbb
        struct ggml_cgraph * gbb = ggml_graph_dup(ctx0, gb);
        // 构建反向传播计算图
        ggml_build_backward_expand(ctx0, gb, gbb, true);
    
        // 重置计算图 gb
        ggml_graph_reset(gb);
        // 设置张量 g1, g2 的梯度为 1.0f
        ggml_set_f32(g1->grad, 1.0f);
        ggml_set_f32(g2->grad, 1.0f);
    
        // 使用上下文 ctx0 计算计算图 gbb
        ggml_graph_compute_with_ctx(ctx0, gbb, n_threads);
    
        // 打印 H * [1, 1] 的结果
        printf("H * [1, 1] = [ %f %f ]\n", ggml_get_f32_1d(x1->grad, 0), ggml_get_f32_1d(x2->grad, 0));
    
        // 断言张量 x1 的梯度为 3.0f
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 3.0f);
        // 断言张量 x2 的梯度为 1.0f
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 1.0f);
    
        // 将计算图 gf 导出为 test1-2-forward.dot 文件
        ggml_graph_dump_dot(gf, NULL, "test1-2-forward.dot");
        // 将计算图 gb 导出为 test1-2-backward.dot 文件
        ggml_graph_dump_dot(gb, gf,   "test1-2-backward.dot");
    }
    ///////////////////////////////////////////////////////////////
    // 创建两个一维张量 x1 和 x2，数据类型为 GGML_TYPE_F32，长度为 1
    {
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 1);

        // 将 x1 和 x2 设置为参数
        ggml_set_param(ctx0, x1);
        ggml_set_param(ctx0, x2);

        // 计算 y = (x1 * x1 + x1 * x2) * x1
        struct ggml_tensor * y = ggml_mul(ctx0, ggml_add(ctx0, ggml_mul(ctx0, x1, x1), ggml_mul(ctx0, x1, x2)), x1);

        // 创建一个自定义大小的计算图 gf，并构建前向传播
        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        ggml_build_forward_expand(gf, y);

        // 复制 gf 为 gb，并构建反向传播
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        ggml_build_backward_expand(ctx0, gf, gb, false);

        // 设置 x1 和 x2 的值
        ggml_set_f32(x1, 3.0f);
        ggml_set_f32(x2, 4.0f);

        // 重置 gf，并设置 y 的梯度为 1.0
        ggml_graph_reset(gf);
        ggml_set_f32(y->grad, 1.0f);

        // 使用上下文 ctx0 计算 gb 的值，使用 n_threads 个线程
        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        // 打印 y 的值
        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        // 打印 df/dx1 的值
        printf("df/dx1 = %f\n", ggml_get_f32_1d(x1->grad, 0));
        // 打印 df/dx2 的值
        printf("df/dx2 = %f\n", ggml_get_f32_1d(x2->grad, 0));

        // 断言 y 的值为 63.0f
        GGML_ASSERT(ggml_get_f32_1d(y, 0)        == 63.0f);
        // 断言 df/dx1 的值为 51.0f
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 51.0f);
        // 断言 df/dx2 的值为 9.0f
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 9.0f);

        // 将 gf 的计算图以 DOT 格式输出到文件 "test1-3-forward.dot"
        ggml_graph_dump_dot(gf, NULL, "test1-3-forward.dot");
        // 将 gb 的计算图以 DOT 格式输出到文件 "test1-3-backward.dot"
        ggml_graph_dump_dot(gb, gf,   "test1-3-backward.dot");
    }
    ///////////////////////////////////////////////////////////////
    {
        // 创建一个包含3个元素的一维浮点型张量 x1
        struct ggml_tensor * x1 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);
        // 创建一个包含3个元素的一维浮点型张量 x2
        struct ggml_tensor * x2 = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, 3);

        // 将 x1 设置为参数
        ggml_set_param(ctx0, x1);
        // 将 x2 设置为参数
        ggml_set_param(ctx0, x2);

        // 创建一个新的张量 y，其值为 x1 和 x2 的乘积的和
        struct ggml_tensor * y = ggml_sum(ctx0, ggml_mul(ctx0, x1, x2));

        // 创建一个自定义大小的前向计算图
        struct ggml_cgraph * gf = ggml_new_graph_custom(ctx0, GGML_DEFAULT_GRAPH_SIZE, true);
        // 构建前向计算图
        ggml_build_forward_expand(gf, y);
        // 复制前向计算图为后向计算图
        struct ggml_cgraph * gb = ggml_graph_dup(ctx0, gf);
        // 构建后向计算图
        ggml_build_backward_expand(ctx0, gf, gb, false);

        // 设置张量 x1 的值为 3.0
        ggml_set_f32(x1, 3.0f);
        // 设置张量 x2 的值为 5.0
        ggml_set_f32(x2, 5.0f);

        // 重置前向计算图
        ggml_graph_reset(gf);
        // 设置张量 y 的梯度为 1.0
        ggml_set_f32(y->grad, 1.0f);

        // 使用上下文计算后向计算图
        ggml_graph_compute_with_ctx(ctx0, gb, n_threads);

        // 打印张量 y 的值
        printf("y      = %f\n", ggml_get_f32_1d(y, 0));
        // 打印张量 x1 的梯度
        printf("df/dx1 = %f %f %f\n",
                ggml_get_f32_1d(x1->grad, 0),
                ggml_get_f32_1d(x1->grad, 1),
                ggml_get_f32_1d(x1->grad, 2));
        // 打印张量 x2 的梯度
        printf("df/dx2 = %f %f %f\n",
                ggml_get_f32_1d(x2->grad, 0),
                ggml_get_f32_1d(x2->grad, 1),
                ggml_get_f32_1d(x2->grad, 2));

        // 断言张量 y 的值为 45.0
        GGML_ASSERT(ggml_get_f32_1d(y, 0)        == 45.0f);
        // 断言张量 x1 的梯度的值
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 0) == 5.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 0) == 3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 1) == 5.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 1) == 3.0f);
        GGML_ASSERT(ggml_get_f32_1d(x1->grad, 2) == 5.0f);
        GGML_ASSERT(ggml_get_f32_1d(x2->grad, 2) == 3.0f);

        // 将前向计算图以 DOT 格式导出为文件
        ggml_graph_dump_dot(gf, NULL, "test1-5-forward.dot");
        // 将后向计算图以 DOT 格式导出为文件
        ggml_graph_dump_dot(gb, gf,   "test1-5-backward.dot");
    }

    ///////////////////////////////////////////////////////////////

    }

    ///////////////////////////////////////////////////////////////

    }

    ///////////////////////////////////////////////////////////////

    }

    // 释放上下文
    ggml_free(ctx0);

    // 返回 0
    return 0;
# 闭合前面的函数定义
```