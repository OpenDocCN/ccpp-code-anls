# `ggml\tests\test3.c`

```
#include "ggml/ggml.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>

// 判断两个浮点数是否接近
bool is_close(float a, float b, float epsilon) {
    return fabs(a - b) < epsilon;
}

// 主函数
int main(int argc, const char ** argv) {
    // 初始化参数
    struct ggml_init_params params = {
        .mem_size   = 1024*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    // 选择优化参数
    //struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_ADAM);
    struct ggml_opt_params opt_params = ggml_opt_default_params(GGML_OPT_LBFGS);

    // 设置线程数
    opt_params.n_threads = (argc > 1) ? atoi(argv[1]) : 8;

    const int NP = 1 << 12; // 定义常量 NP
    const int NF = 1 << 8;  // 定义常量 NF

    // 初始化上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建二维张量 F
    struct ggml_tensor * F = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, NF, NP);
    // 创建一维张量 l
    struct ggml_tensor * l = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, NP);

    // 正则化权重
    struct ggml_tensor * lambda = ggml_new_f32(ctx0, 1e-5f);

    srand(0); // 设置随机数种子

    // 循环初始化 l 和 F
    for (int j = 0; j < NP; j++) {
        const float ll = j < NP/2 ? 1.0f : -1.0f;
        ((float *)l->data)[j] = ll;

        for (int i = 0; i < NF; i++) {
            ((float *)F->data)[j*NF + i] = ((ll > 0 && i < NF/2 ? 1.0f : ll < 0 && i >= NF/2 ? 1.0f : 0.0f) + ((float)rand()/(float)RAND_MAX - 0.5f)*0.1f)/(0.5f*NF);
        }
    }
}
    {
        // 初始化猜测值
        struct ggml_tensor * x = ggml_set_f32(ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, NF), 0.0f);

        // 设置优化参数
        ggml_set_param(ctx0, x);

        // 计算损失函数 f = sum[(fj*x - l)^2]/n + lambda*|x^2|
        struct ggml_tensor * f =
            ggml_add(ctx0,
                    ggml_div(ctx0,
                        ggml_sum(ctx0,
                            ggml_sqr(ctx0,
                                ggml_sub(ctx0,
                                    ggml_mul_mat(ctx0, F, x),
                                    l)
                                )
                            ),
                        ggml_new_f32(ctx0, (float)NP)
                        ),
                    ggml_mul(ctx0,
                        ggml_sum(ctx0, ggml_sqr(ctx0, x)),
                        lambda)
                    );

        // 进行优化
        enum ggml_opt_result res = ggml_opt(NULL, opt_params, f);

        // 断言优化结果为成功
        GGML_ASSERT(res == GGML_OPT_OK);

        // 打印结果
        for (int i = 0; i < 16; i++) {
            printf("x[%3d] = %g\n", i, ((float *)x->data)[i]);
        }
        printf("...\n");
        for (int i = NF - 16; i < NF; i++) {
            printf("x[%3d] = %g\n", i, ((float *)x->data)[i]);
        }
        printf("\n");

        // 断言优化后的结果满足条件
        for (int i = 0; i < NF; ++i) {
            if (i < NF/2) {
                GGML_ASSERT(is_close(((float *)x->data)[i],  1.0f, 1e-2f));
            } else {
                GGML_ASSERT(is_close(((float *)x->data)[i], -1.0f, 1e-2f));
            }
        }
    }

    // 释放上下文
    ggml_free(ctx0);

    // 返回成功
    return 0;
# 闭合前面的函数定义
```