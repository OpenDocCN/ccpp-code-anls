# `ggml\tests\test-mul-mat1.c`

```
#include <stdint.h>
#include <stdio.h>
#include <assert.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#include <sys/time.h>

#include <arm_neon.h>

#include <Accelerate/Accelerate.h>

// 定义矩阵的维度
const int M = 1280;
const int N = 1536;
const int K = 1280;

// 获取当前时间的微秒数
uint64_t get_time_us(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000000 + tv.tv_usec;
}

//
// naive implementation
//

// 实现矩阵乘法（单精度浮点数）
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
            dst[i*n + j] = sum;
        }
    }
}

// 实现矩阵乘法（半精度浮点数）
void mul_mat_f16_0(
    const __fp16 * src0,
    const __fp16 * src1,
           float * dst,
    int m, int n, int k) {
    const int k32 = k & ~31;
    // 遍历 m 行
    for (int i = 0; i < m; i++) {
        // 遍历 n 列
        for (int j = 0; j < n; j++) {
            // 初始化 sumf 为 0
            float sumf = 0.0;

            // 初始化四个 float16x8_t 类型的变量为 0
            float16x8_t sum0 = vdupq_n_f16(0.0f);
            float16x8_t sum1 = vdupq_n_f16(0.0f);
            float16x8_t sum2 = vdupq_n_f16(0.0f);
            float16x8_t sum3 = vdupq_n_f16(0.0f);

            // 初始化八个 float16x8_t 类型的变量
            float16x8_t x0, x1, x2, x3;
            float16x8_t y0, y1, y2, y3;

            // 限定指针 p0 和 p1 的访问范围
            const __fp16 * restrict p0 = src0 + i*k;
            const __fp16 * restrict p1 = src1 + j*k;

            // 遍历 k32，每次增加 32
            for (int l = 0; l < k32; l += 32) {
                // 从 p0 和 p1 中加载数据到 x0, x1, x2, x3, y0, y1, y2, y3
                x0 = vld1q_f16(p0 + l + 0 );
                x1 = vld1q_f16(p0 + l + 8 );
                x2 = vld1q_f16(p0 + l + 16);
                x3 = vld1q_f16(p0 + l + 24);

                y0 = vld1q_f16(p1 + l + 0 );
                y1 = vld1q_f16(p1 + l + 8 );
                y2 = vld1q_f16(p1 + l + 16);
                y3 = vld1q_f16(p1 + l + 24);

                // 使用 vfmaq_f16 函数进行乘加操作
                sum0 = vfmaq_f16(sum0, x0, y0);
                sum1 = vfmaq_f16(sum1, x1, y1);
                sum2 = vfmaq_f16(sum2, x2, y2);
                sum3 = vfmaq_f16(sum3, x3, y3);
            }

            // 将 sum0..sum3 合并到 sum0
            sum0 = vaddq_f16(sum0, sum1);
            sum2 = vaddq_f16(sum2, sum3);
            sum0 = vaddq_f16(sum0, sum2);

            // 将 sum0 转换为两个 float32x4_t 类型
            float32x4_t sum0f32 = vcvt_f32_f16(vget_low_f16(sum0));
            float32x4_t sum1f32 = vcvt_f32_f16(vget_high_f16(sum0));

            // 将 sum0f32 和 sum1f32 合并到 sumf32
            sum0f32 = vaddq_f32(sum0f32, sum1f32);

            // 将 sumf32 中的值相加得到 sumf
            float32x2_t sumf32 = vadd_f32(vget_low_f32(sum0f32), vget_high_f32(sum0f32));
            sumf = vget_lane_f32(sumf32, 0) + vget_lane_f32(sumf32, 1);

            // 遍历 k32 到 k32，每次增加 1
            for (int l = k32; l < k32; l++) {
                // 计算 p0[l]*p1[l] 并加到 sumf
                sumf += p0[l]*p1[l];
            }

            // 将 sumf 存入 dst 数组中
            dst[i*n + j] = sumf;
        }
    }
}

// 使用块大小为32进行阻塞
void mul_mat_f16_1(
    const __fp16 * src0,
    const __fp16 * src1,
           float * dst,
    int m, int n, int k) {

    const int k32 = k & ~31; // 将k向下取整到最接近的32的倍数
    const int bs  = 32; // 定义块大小为32

    memset(dst, 0, m*n*sizeof(float); // 将dst数组初始化为0

    }

}

void mul_mat_f8_0(
    const uint8_t * src0,
    const uint8_t * src1,
           float * dst,
    int m, int n, int k) {
    const int k32 = k & ~31; // 将k向下取整到最接近的32的倍数

    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            float sumf = 0.0; // 初始化sumf为0

            const uint8_t * restrict p0 = src0 + i*k; // 定义指向src0的指针p0
            const uint8_t * restrict p1 = src1 + j*k; // 定义指向src1的指针p1

            for (int l = 0; l < k32; l += 32) { // 循环，每次增加32
                uint8x16_t x0 = vld1q_u8(p0 + l + 0 ); // 从p0加载16个8位整数到x0
                uint8x16_t x1 = vld1q_u8(p0 + l + 16); // 从p0加载16个8位整数到x1

                uint8x16_t y0 = vld1q_u8(p1 + l + 0 ); // 从p1加载16个8位整数到y0
                uint8x16_t y1 = vld1q_u8(p1 + l + 16); // 从p1加载16个8位整数到y1

                x0 = vmulq_u8(x0, y0); // 逐元素相乘
                x1 = vmulq_u8(x1, y1); // 逐元素相乘

                sumf += vaddvq_u8(x0) + vaddvq_u8(x1); // 将x0和x1中的元素相加，并将结果加到sumf中
            }

            dst[i*n + j] = sumf; // 将sumf赋值给dst数组的对应位置
        }
    }
}

int main(int argc, const char ** argv) {
    float * src0 = malloc(sizeof(float)*M*K); // 为src0分配内存
    float * src1 = malloc(sizeof(float)*N*K); // 为src1分配内存
    float * dst  = malloc(sizeof(float)*M*N); // 为dst分配内存

    for (int i = 0; i < M*K; i++) {
        src0[i] = rand() / (float)RAND_MAX; // 生成随机数并赋值给src0数组
    }

    for (int i = 0; i < N*K; i++) {
        src1[i] = rand() / (float)RAND_MAX; // 生成随机数并赋值给src1数组
    }

    // 将src0和src1转换为__fp16
    __fp16 * src0_fp16 = (__fp16 *)(malloc(sizeof(__fp16)*M*K)); // 为src0_fp16分配内存
    __fp16 * src1_fp16 = (__fp16 *)(malloc(sizeof(__fp16)*N*K)); // 为src1_fp16分配内存

    uint8_t * src0_fp8 = (uint8_t *)(malloc(sizeof(__fp16)*M*K)); // 为src0_fp8分配内存
    uint8_t * src1_fp8 = (uint8_t *)(malloc(sizeof(__fp16)*N*K)); // 为src1_fp8分配内存
    // 获取当前时间的微秒数
    const uint64_t t_start = get_time_us();

    // 将src0数组中的数据复制到src0_fp16数组中
    for (int i = 0; i < M*K; i++) {
        src0_fp16[i] = src0[i];
        // 打印src0和src0_fp16数组中的数据
        // 确保src0_fp16数组中的数据不是NaN
        //printf("%f %f\n", src0[i], src0_fp16[i]);
        //assert(!isnan(src0_fp16[i]));
    }

    // 将src1数组中的数据复制到src1_fp16数组中
    for (int i = 0; i < N*K; i++) {
        src1_fp16[i] = src1[i];
    }

    // 获取循环结束时的当前时间的微秒数
    const uint64_t t_end = get_time_us();
    // 打印转换时间
    printf("convert time: %f ms\n", (t_end - t_start) / 1000.0);

    // 打印src0和src0_fp16数组中的前16个数据
    for (int i = 0; i < 16; ++i) {
        printf("%f %f\n", src0[i], src0_fp16[i]);
    }

    // 从命令行参数中获取method值
    int method = 0;
    if (argc > 1) {
        method = atoi(argv[1]);
    }

    // 设置循环迭代次数
    const int nIter = 1;

    // 获取开始时的时钟时间
    const clock_t start = clock();
    // 获取开始时的当前时间的微秒数
    const uint64_t start_us = get_time_us();

    // 计算M的倒数
    double iM = 1.0/M;
    double sum = 0.0f;
    // 循环nIter次
    for (int i = 0; i < nIter; i++) {
        // 根据method值选择不同的矩阵乘法函数进行计算
        if (method == 0) {
            mul_mat_f32_0(src0, src1, dst, M, N, K);
        }

        if (method == 1) {
            mul_mat_f16_0(src0_fp16, src1_fp16, dst, M, N, K);
        }

        if (method == 2) {
            mul_mat_f16_1(src0_fp16, src1_fp16, dst, M, N, K);
        }

        if (method == 3) {
            mul_mat_f8_0(src0_fp8, src1_fp8, dst, M, N, K);
        }

        if (method == 4) {
            // 使用加速框架中的BLAS sgemm函数进行矩阵乘法计算
            cblas_sgemm(CblasRowMajor, CblasNoTrans, CblasTrans, M, N, K, 1.0f, src0, K, src1, K, 0.0f, dst, N);
        }
    }

    // 计算dst数组中元素乘以iM的和
    for (int i = 0; i < N; i++) {
        sum += dst[i]*iM;
    }

    // 获取结束时的时钟时间
    const clock_t end = clock();
    // 获取结束时的当前时间的微秒数
    const uint64_t end_us = get_time_us();
    // 打印函数执行所花费的时钟滴答数
    printf("%s: elapsed ticks: %ld\n",  __func__, end - start);
    // 打印函数执行所花费的微秒数
    printf("%s: elapsed us:    %llu / %f ms\n",  __func__, end_us - start_us, (end_us - start_us) / 1000.0 / nIter);

    // 打印sum的值
    printf("%f\n", sum);

    // 释放内存
    free(src0);
    free(src1);
    free(dst);

    free(src0_fp16);
    free(src1_fp16);

    // 返回0表示程序执行成功
    return 0;
# 闭合前面的代码块
```