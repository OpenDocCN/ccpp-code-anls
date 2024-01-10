# `ggml\tests\test2.c`

```
#define _CRT_SECURE_NO_DEPRECATE // Disables ridiculous "unsafe" warnigns on Windows
#include "ggml/ggml.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义一个函数，用于判断两个浮点数是否接近
bool is_close(float a, float b, float epsilon) {
    return fabs(a - b) < epsilon;
}

// 主函数
int main(int argc, const char ** argv) {
    // 初始化参数结构体
    struct ggml_init_params params = {
        .mem_size   = 128*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    // 初始化优化参数结构体，使用 LBFGS 优化算法
    struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_LBFGS);

    // 设置线程数，默认为 8
    int nthreads = 8;
    const char *env = getenv("GGML_NTHREADS");
    if (env != NULL) {
        nthreads = atoi(env);
    }
    if (argc > 1) {
        nthreads = atoi(argv[1]);
    }
    opt_params.n_threads = nthreads;
    printf("test2: n_threads:%d\n", opt_params.n_threads);

    // 定义输入和输出数据
    const float xi[] = {  1.0f,  2.0f,  3.0f,  4.0f,  5.0f , 6.0f,  7.0f,  8.0f,  9.0f,  10.0f, };
          float yi[] = { 15.0f, 25.0f, 35.0f, 45.0f, 55.0f, 65.0f, 75.0f, 85.0f, 95.0f, 105.0f, };

    const int n = sizeof(xi)/sizeof(xi[0]);

    // 初始化 GGML 上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建输入和输出张量
    struct ggml_tensor * x = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, n);
    struct ggml_tensor * y = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, n);

    // 将数据填充到输入和输出张量中
    for (int i = 0; i < n; i++) {
        ((float *) x->data)[i] = xi[i];
        ((float *) y->data)[i] = yi[i];
    }
    {
        // 创建两个新的浮点数张量 t0 和 t1，初始值为 0.0f
        struct ggml_tensor * t0 = ggml_new_f32(ctx0, 0.0f);
        struct ggml_tensor * t1 = ggml_new_f32(ctx0, 0.0f);
    
        // 初始化自动微分参数
        ggml_set_param(ctx0, t0);
        ggml_set_param(ctx0, t1);
    
        // 计算损失函数 f = sum_i[(t0 + t1*x_i - y_i)^2]/(2n)
        struct ggml_tensor * f =
            ggml_div(ctx0,
                    ggml_sum(ctx0,
                        ggml_sqr(ctx0,
                            ggml_sub(ctx0,
                                ggml_add(ctx0,
                                    ggml_mul(ctx0, x, ggml_repeat(ctx0, t1, x)),
                                    ggml_repeat(ctx0, t0, x)),
                                y)
                            )
                        ),
                    ggml_new_f32(ctx0, 2.0f*n));
    
        // 使用梯度下降法优化损失函数
        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);
    
        // 打印 t0 和 t1 的值
        printf("t0 = %f\n", ggml_get_f32_1d(t0, 0));
        printf("t1 = %f\n", ggml_get_f32_1d(t1, 0));
    
        // 断言优化结果为 GGML_OPT_OK
        GGML_ASSERT(res == GGML_OPT_OK);
    
        // 断言 t0 和 t1 的值接近预期值，误差范围为 1e-3f
        GGML_ASSERT(is_close(ggml_get_f32_1d(t0, 0),  5.0f, 1e-3f));
        GGML_ASSERT(is_close(ggml_get_f32_1d(t1, 0), 10.0f, 1e-3f));
    }
    {
        // 创建一个新的浮点型张量 t0，值为 -1.0f
        struct ggml_tensor * t0 = ggml_new_f32(ctx0, -1.0f);
        // 创建一个新的浮点型张量 t1，值为 9.0f
        struct ggml_tensor * t1 = ggml_new_f32(ctx0,  9.0f);

        // 将 t0 设置为参数
        ggml_set_param(ctx0, t0);
        // 将 t1 设置为参数
        ggml_set_param(ctx0, t1);

        // 计算损失函数 f = 0.5*sum_i[abs(t0 + t1*x_i - y_i)]/n
        struct ggml_tensor * f =
            ggml_mul(ctx0,
                    ggml_new_f32(ctx0, 1.0/(2*n)),
                    ggml_sum(ctx0,
                        ggml_abs(ctx0,
                            ggml_sub(ctx0,
                                ggml_add(ctx0,
                                    ggml_mul(ctx0, x, ggml_repeat(ctx0, t1, x)),
                                    ggml_repeat(ctx0, t0, x)),
                                y)
                            )
                        )
                    );


        // 使用优化器优化损失函数
        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        // 断言优化结果为 GGML_OPT_OK
        GGML_ASSERT(res == GGML_OPT_OK);
        // 断言 t0 的值接近 5.0f，误差为 1e-2f
        GGML_ASSERT(is_close(ggml_get_f32_1d(t0, 0),  5.0f, 1e-2f));
        // 断言 t1 的值接近 10.0f，误差为 1e-2f
        GGML_ASSERT(is_close(ggml_get_f32_1d(t1, 0), 10.0f, 1e-2f));
    }

    {
        // 创建一个新的浮点型张量 t0，值为 5.0f
        struct ggml_tensor * t0 = ggml_new_f32(ctx0,  5.0f);
        // 创建一个新的浮点型张量 t1，值为 -4.0f
        struct ggml_tensor * t1 = ggml_new_f32(ctx0, -4.0f);

        // 将 t0 设置为参数
        ggml_set_param(ctx0, t0);
        // 将 t1 设置为参数
        ggml_set_param(ctx0, t1);

        // 计算损失函数 f = t0^2 + t1^2
        struct ggml_tensor * f =
            ggml_add(ctx0,
                    ggml_sqr(ctx0, t0),
                    ggml_sqr(ctx0, t1)
                    );

        // 使用优化器优化损失函数
        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        // 断言优化结果为 GGML_OPT_OK
        GGML_ASSERT(res == GGML_OPT_OK);
        // 断言 f 的值接近 0.0f，误差为 1e-3f
        GGML_ASSERT(is_close(ggml_get_f32_1d(f,  0), 0.0f, 1e-3f));
        // 断言 t0 的值接近 0.0f，误差为 1e-3f
        GGML_ASSERT(is_close(ggml_get_f32_1d(t0, 0), 0.0f, 1e-3f));
        // 断言 t1 的值接近 0.0f，误差为 1e-3f
        GGML_ASSERT(is_close(ggml_get_f32_1d(t1, 0), 0.0f, 1e-3f));
    }

    /////////////////////////////////////////
    {
        // 创建一个新的浮点型张量 t0，值为 -7.0f
        struct ggml_tensor * t0 = ggml_new_f32(ctx0, -7.0f);
        // 创建一个新的浮点型张量 t1，值为 8.0f
        struct ggml_tensor * t1 = ggml_new_f32(ctx0,  8.0f);

        // 设置 t0 为优化参数
        ggml_set_param(ctx0, t0);
        // 设置 t1 为优化参数
        ggml_set_param(ctx0, t1);

        // 计算目标函数 f = (t0 + 2*t1 - 7)^2 + (2*t0 + t1 - 5)^2
        struct ggml_tensor * f =
            ggml_add(ctx0,
                    // 计算 (t0 + 2*t1 - 7) 的平方
                    ggml_sqr(ctx0,
                        // 计算 t0 + 2*t1
                        ggml_sub(ctx0,
                            // 计算 t0 + 2*t1
                            ggml_add(ctx0,
                                t0,
                                ggml_mul(ctx0, t1, ggml_new_f32(ctx0, 2.0f))),
                            ggml_new_f32(ctx0, 7.0f)
                            )
                        ),
                    // 计算 (2*t0 + t1 - 5) 的平方
                    ggml_sqr(ctx0,
                        // 计算 2*t0 + t1
                        ggml_sub(ctx0,
                            // 计算 2*t0 + t1
                            ggml_add(ctx0,
                                ggml_mul(ctx0, t0, ggml_new_f32(ctx0, 2.0f)),
                                t1),
                            ggml_new_f32(ctx0, 5.0f)
                            )
                        )
                    );

        // 对目标函数进行优化
        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        // 断言优化结果为 GGML_OPT_OK
        GGML_ASSERT(res == GGML_OPT_OK);
        // 断言优化后的 f 的第一个元素接近于 0.0f，误差为 1e-3f
        GGML_ASSERT(is_close(ggml_get_f32_1d(f,  0), 0.0f, 1e-3f));
        // 断言优化后的 t0 的第一个元素接近于 1.0f，误差为 1e-3f
        GGML_ASSERT(is_close(ggml_get_f32_1d(t0, 0), 1.0f, 1e-3f));
        // 断言优化后的 t1 的第一个元素接近于 3.0f，误差为 1e-3f
        GGML_ASSERT(is_close(ggml_get_f32_1d(t1, 0), 3.0f, 1e-3f));
    }

    // 释放上下文 ctx0
    ggml_free(ctx0);

    // 返回 0
    return 0;
}
# 闭合前面的函数定义
```