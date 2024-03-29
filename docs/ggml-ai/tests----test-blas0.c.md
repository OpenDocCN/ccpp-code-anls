# `ggml\tests\test-blas0.c`

```cpp
#include "ggml.h"

#include <stdint.h>
#include <stdio.h>
#include <assert.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <math.h>

#include <sys/time.h>

#include <arm_neon.h>

#include <Accelerate/Accelerate.h>

// 获取当前时间的微秒数
uint64_t get_time_us(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000000 + tv.tv_usec;
}

//
// naive implementation
//

// 矩阵乘法的朴素实现
void mul_mat_f32_0(
    const float * restrict src0, // M x K
    const float * restrict src1, // N x K (transposed)
    float * dst,
    int m, int n, int k) {
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            float sum = 0;
            for (int l = 0; l < k; l++) {
                sum += src0[i*k + l] * src1[j*k + l];
            }
            dst[j*m + i] = sum;
        }
    }
}

// 主函数
int main(int argc, const char ** argv) {
    if (argc < 4) {
        printf("Usage: %s M N K\n", argv[0]);
        return 1;
    }

    const int n_threads = 1;

    int M = atoi(argv[1]);
    int N = atoi(argv[2]);
    int K = atoi(argv[3]);

    srand(time(NULL));

    if (M == 0) M = rand() % 1000 + 1;
    if (N == 0) N = rand() % 1000 + 1;
    if (K == 0) K = rand() % 1000 + 1;

    printf("M = %d, N = %d, K = %d\n", M, N, K);

    float * src0 = malloc(sizeof(float)*M*K);
    float * src1 = malloc(sizeof(float)*N*K);
    float * dst0 = malloc(sizeof(float)*M*N); // naive
    float * dst1 = malloc(sizeof(float)*M*N); // blas

    // 初始化参数
    struct ggml_init_params params = {
        .mem_size   = 2048ul*1024*1024,
        .mem_buffer = NULL,
        .no_alloc   = false,
    };

    // 初始化上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建浮点型2D张量
    struct ggml_tensor * s0_f32 = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, K, M);
    struct ggml_tensor * s1_f32 = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, K, N);

    // 创建半精度浮点型2D张量
    struct ggml_tensor * s0_f16 = ggml_new_tensor_2d(ctx0, GGML_TYPE_F16, K, M);
    struct ggml_tensor * s1_f16 = ggml_new_tensor_2d(ctx0, GGML_TYPE_F16, K, N);
    // 遍历二维数组 src0，填充随机数值
    for (int j = 0; j < M; j++) {
        for (int i = 0; i < K; i++) {
            src0[j*K + i] = 1e-3*(rand() % 1000);
        }
    }

    // 遍历二维数组 src1，填充随机数值
    for (int j = 0; j < N; j++) {
        for (int i = 0; i < K; i++) {
            src1[j*K + i] = 1e-3*(rand() % 1000);
        }
    }

    // 将 src0 复制到 s0_f32 和 s0_f16
    {
        float       * p_f32 = s0_f32->data;
        ggml_fp16_t * p_f16 = s0_f16->data;
        for (int i = 0; i < M; i++) {
            for (int j = 0; j < K; j++) {
                p_f32[i*K + j] = src0[i*K + j];
                p_f16[i*K + j] = ggml_fp32_to_fp16(src0[i*K + j]);
            }
        }
    }

    // 将 src1 复制到 s1_f32 和 s1_f16
    {
        float       * p_f32 = s1_f32->data;
        ggml_fp16_t * p_f16 = s1_f16->data;
        for (int i = 0; i < N; i++) {
            for (int j = 0; j < K; j++) {
                p_f32[i*K + j] = src1[i*K + j];
                p_f16[i*K + j] = ggml_fp32_to_fp16(src1[i*K + j]);
            }
        }
    }

    // 记录开始时间
    const clock_t start = clock();
    const uint64_t start_us = get_time_us();

    // 计算矩阵乘法 dst0 = src0 * src1
    double iM = 1.0/M;
    mul_mat_f32_0(src0, src1, dst0, M, N, K);

    // 使用加速框架中的 BLAS sgemm 函数进行矩阵乘法计算
    cblas_sgemm(CblasRowMajor, CblasNoTrans, CblasTrans, N, M, K, 1.0f, src1, K, src0, K, 0.0f, dst1, M);

    // 初始化 dst2 和 dst3
    struct ggml_tensor * dst2 = NULL;
    struct ggml_tensor * dst3 = NULL;

    // 使用 ggml_mul_mat 函数计算 s0_f32 和 s1_f32 的矩阵乘法，并进行计算图的前向传播
    {
        dst2 = ggml_mul_mat(ctx0, s0_f32, s1_f32);

        struct ggml_cgraph * gf = ggml_new_graph(ctx0);
        ggml_build_forward_expand(gf, dst2);
        ggml_graph_compute_with_ctx(ctx0, gf, n_threads);
    }

    // 使用 ggml_mul_mat 函数计算 s0_f16 和 s1_f32 的矩阵乘法，并进行计算图的前向传播
    {
        dst3 = ggml_mul_mat(ctx0, s0_f16, s1_f32);

        struct ggml_cgraph * gf = ggml_new_graph(ctx0);
        ggml_build_forward_expand(gf, dst3);
        ggml_graph_compute_with_ctx(ctx0, gf, n_threads);
    }

    // 初始化布尔值变量用于记录计算结果是否正确
    bool ok_blas = true;
    bool ok_ggml_f32 = true;
    bool ok_ggml_f16 = true;

    // 检查 BLAS 计算结果
    // 遍历 dst0 和 dst1 数组，检查它们的差值是否超过阈值，如果超过则输出对应的索引和数值，并将 ok_blas 设为 false
    for (int i = 0; i < M*N; i++) {
        if (fabs(dst0[i] - dst1[i])/fabs(dst0[i]) > 0.0001) {
            printf("dst0[%d] = %f, dst1[%d] = %f\n", i, dst0[i], i, dst1[i]);
            ok_blas = false;
        }
    }

    // 检查 ggml (f32)
    {
        // 获取 dst2 数组的数据指针
        float * p = dst2->data;
        // 遍历 dst0 和 dst2 数组，检查它们的差值是否超过阈值，如果超过则输出对应的索引和数值，并将 ok_ggml_f32 设为 false
        for (int i = 0; i < M*N; i++) {
            if (fabs(dst0[i] - p[i])/fabs(dst0[i]) > 0.0001) {
                printf("dst0[%d] = %f, dst2[%d] = %f\n", i, dst0[i], i, p[i]);
                ok_ggml_f32 = false;
            }
        }
    }

    // 检查 ggml (f16)
    {
        // 获取 dst3 数组的数据指针
        float * p = dst3->data;
        // 遍历 dst0 和 dst3 数组，检查它们的差值是否超过阈值，如果超过则输出对应的索引和数值，并将 ok_ggml_f16 设为 false
        for (int i = 0; i < M*N; i++) {
            if (fabs(dst0[i] - p[i])/fabs(dst0[i]) > 0.01) {
                printf("dst0[%d] = %f, dst3[%d] = %f\n", i, dst0[i], i, p[i]);
                ok_ggml_f16 = false;
            }
        }
    }

    {
        // 计算程序运行时间并输出
        const clock_t end = clock();
        const uint64_t end_us = get_time_us();
        printf("%s: elapsed ticks: %ld\n",  __func__, end - start);
    }
#if 0
    // print src0
    printf("src0:\n");
    for (int i = 0; i < M; i++) {
        for (int j = 0; j < K; j++) {
            printf("%4.1f ", src0[i*K+j]);
        }
        printf("\n");
    }

    // print src1
    printf("src1:\n");
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < K; j++) {
            printf("%4.1f ", src1[i*K+j]);
        }
        printf("\n");
    }

    printf("\n");
    printf("dst0 (naive):\n");
    for (int j = 0; j < N; j++) {
        for (int i = 0; i < M; i++) {
            printf("%4.1f ", dst0[j*M+i]);
        }
        printf("\n");
    }

    printf("\n");
    printf("dst1 (BLAS):\n");
    for (int j = 0; j < N; j++) {
        for (int i = 0; i < M; i++) {
            printf("%4.1f ", dst1[j*M+i]);
        }
        printf("\n");
    }

    printf("\n");
    printf("dst2 (ggml f32):\n");
    for (int j = 0; j < N; j++) {
        for (int i = 0; i < M; i++) {
            printf("%4.1f ", ((float *)dst2->data)[j*M+i]);
        }
        printf("\n");
    }

    printf("\n");
    printf("dst3 (ggml f16):\n");
    for (int j = 0; j < N; j++) {
        for (int i = 0; i < M; i++) {
            printf("%4.1f ", ((float *)dst3->data)[j*M+i]);
        }
        printf("\n");
    }

    printf("\n");
#endif

    // 释放内存
    free(src0);
    free(src1);
    free(dst0);
    free(dst1);

    // 释放 ggml 上下文
    ggml_free(ctx0);

    // 打印 BLAS 运行结果
    printf("ok_blas = %d\n", ok_blas);
    if (!ok_blas) {
        printf("ERROR: BLAS failed\n");
    }

    // 打印 ggml f32 运行结果
    printf("ok_ggml_f32 = %d\n", ok_ggml_f32);
    if (!ok_ggml_f32) {
        printf("ERROR: ggml failed\n");
    }

    // 打印 ggml f16 运行结果
    printf("ok_ggml_f16 = %d\n", ok_ggml_f16);
    if (!ok_ggml_f16) {
        printf("ERROR: ggml failed\n");
    }

    // 返回运行结果
    return (ok_blas && ok_ggml_f32 && ok_ggml_f16) ? 0 : 1;
}
```