# `ggml\tests\test-vec2.c`

```
#include <stdint.h>  // 包含标准整数类型的头文件
#include <stdio.h>   // 包含标准输入输出的头文件
#include <assert.h>  // 包含断言的头文件
#include <stdlib.h>  // 包含标准库函数的头文件
#include <time.h>    // 包含时间处理函数的头文件

#include <sys/time.h>  // 包含系统时间处理函数的头文件

#include <arm_neon.h>  // 包含 ARM NEON 指令集的头文件

const int N = 1 << 12;  // 定义常量 N，值为 2 的 12 次方
const int M = 1 << 12;  // 定义常量 M，值为 2 的 12 次方

//
// naive implementation
//

void mul_mat_vec_f32_0(
    const float * restrict src0,  // 限定 src0 指针为只读
    const float * restrict src1,  // 限定 src1 指针为只读
    float * dst,                   // 目标数组指针
    int nrows,                     // 行数
    int ncols) {                   // 列数
    for (int i = 0; i < nrows; i++) {  // 遍历行
        float sum = 0.0f;               // 初始化求和变量
        for (int j = 0; j < ncols; j++) {  // 遍历列
            sum += src0[i*ncols + j]*src1[j];  // 矩阵向量乘法
        }
        dst[i] = sum;  // 将求和结果存入目标数组
    }
}

void mul_mat_vec_f16_0(
    const __fp16 * src0,  // 源矩阵指针
    const __fp16 * src1,  // 源向量指针
           float * dst,   // 目标数组指针
    int nrows,            // 行数
    int ncols) {          // 列数

    const int n64 = ncols & ~63;  // 计算 ncols 与 63 的按位与结果

    }
}

void mul_mat_vec_f16_1(
    const __fp16 * src0,  // 源矩阵指针
    const __fp16 * src1,  // 源向量指针
           float * dst,   // 目标数组指针
    int nrows,            // 行数
    int ncols) {          // 列数

    const int n32 = ncols & ~31;  // 计算 ncols 与 31 的按位与结果
}
    # 遍历行数
    for (int r = 0; r < nrows; r++) {
        # 初始化行求和结果
        float sumf = 0.0;

        # 初始化四个长度为8的float16x8_t类型向量，每个元素都是0.0
        float16x8_t sum0 = vdupq_n_f16(0.0f);
        float16x8_t sum1 = vdupq_n_f16(0.0f);
        float16x8_t sum2 = vdupq_n_f16(0.0f);
        float16x8_t sum3 = vdupq_n_f16(0.0f);

        # 初始化8个float16x8_t类型向量，用于存储待处理的数据
        float16x8_t x0, x1, x2, x3;
        float16x8_t y0, y1, y2, y3;

        # 限定指针p0指向的内存区域只能由p0指针访问，提高内存访问效率
        const __fp16 * restrict p0 = src0 + r*ncols;

        # 遍历n32，每次处理32个元素
        for (int i = 0; i < n32; i += 32) {
            # 从p0指针指向的内存区域加载16个fp16类型的数据到x0, x1, x2, x3
            x0 = vld1q_f16(p0 + i + 0 );
            x1 = vld1q_f16(p0 + i + 8 );
            x2 = vld1q_f16(p0 + i + 16);
            x3 = vld1q_f16(p0 + i + 24);

            # 从src1指针指向的内存区域加载16个fp16类型的数据到y0, y1, y2, y3
            y0 = vld1q_f16(src1 + i + 0 );
            y1 = vld1q_f16(src1 + i + 8 );
            y2 = vld1q_f16(src1 + i + 16);
            y3 = vld1q_f16(src1 + i + 24);

            # 使用FMA指令进行向量运算，将结果存储到sum0, sum1, sum2, sum3中
            sum0 = vfmaq_f16(sum0, x0, y0);
            sum1 = vfmaq_f16(sum1, x1, y1);
            sum2 = vfmaq_f16(sum2, x2, y2);
            sum3 = vfmaq_f16(sum3, x3, y3);
        }

        # 将sum0和sum1相加，sum2和sum3相加，再将结果相加，得到sum0
        sum0 = vaddq_f16(sum0, sum1);
        sum2 = vaddq_f16(sum2, sum3);
        sum0 = vaddq_f16(sum0, sum2);

        # 将sum0转换为两个float32x4_t类型向量
        float32x4_t sum0f32 = vcvt_f32_f16(vget_low_f16(sum0));
        float32x4_t sum1f32 = vcvt_f32_f16(vget_high_f16(sum0));

        # 将sum0f32和sum1f32相加，得到sum0f32
        sum0f32 = vaddq_f32(sum0f32, sum1f32);

        # 将sum0f32中的元素相加，得到sumf32
        float32x2_t sumf32 = vadd_f32(vget_low_f32(sum0f32), vget_high_f32(sum0f32));
        # 将sumf32中的两个元素相加，得到sumf
        sumf = vget_lane_f32(sumf32, 0) + vget_lane_f32(sumf32, 1);

        # 遍历剩余的元素，将其与src1中对应位置的元素相乘并累加到sumf中
        for (int j = n32; j < n32; j++) {
            sumf += src0[r*ncols + j]*src1[j];
        }

        # 将sumf存储到dst中
        dst[r] = sumf;
    }
}

// 获取当前时间的微秒数
uint64_t get_time_us(void) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    return tv.tv_sec * 1000000 + tv.tv_usec;
}

// 主函数
int main(int argc, const char ** argv) {
    // 分配内存空间
    float * src0 = malloc(sizeof(float)*N*M);
    float * src1 = malloc(sizeof(float)*M);
    float * dst  = malloc(sizeof(float)*N);

    // 使用内存对齐分配内存空间
    //float * src0 = (float *)(aligned_alloc(64, sizeof(float)*N*M));
    //float * src1 = (float *)(aligned_alloc(64, sizeof(float)*M));
    //float * dst  = (float *)(aligned_alloc(64, sizeof(float)*N));

    // 为 src0 和 src1 赋随机值
    for (int i = 0; i < N*M; i++) {
        src0[i] = rand() / (float)RAND_MAX;
    }

    for (int i = 0; i < M; i++) {
        src1[i] = rand() / (float)RAND_MAX;
    }

    // 将 src0 和 src1 转换为 __fp16 类型
    __fp16 * src0_fp16 = (__fp16 *)(malloc(sizeof(__fp16)*N*M));
    __fp16 * src1_fp16 = (__fp16 *)(malloc(sizeof(__fp16)*M));

    {
        // 记录开始时间
        const uint64_t t_start = get_time_us();

        // 将 src0 转换为 __fp16 类型
        for (int i = 0; i < N*M; i++) {
            src0_fp16[i] = src0[i];
            //printf("%f %f\n", src0[i], src0_fp16[i]);
            //assert(!isnan(src0_fp16[i]));
        }

        // 将 src1 转换为 __fp16 类型
        for (int i = 0; i < M; i++) {
            src1_fp16[i] = src1[i];
        }

        // 计算转换时间
        const uint64_t t_end = get_time_us();
        printf("convert time: %f ms\n", (t_end - t_start) / 1000.0);
    }

    // 打印转换后的值
    for (int i = 0; i < 16; ++i) {
        printf("%f %f\n", src0[i], src0_fp16[i]);
    }

    // 选择计算方法
    int method = 0;
    if (argc > 1) {
        method = atoi(argv[1]);
    }

    const int nIter = 1000;

    // 记录开始时间
    const clock_t start = clock();
    const uint64_t start_us = get_time_us();

    double iM = 1.0/M;
    double sum = 0.0f;
    // 循环执行计算方法
    for (int i = 0; i < nIter; i++) {
        if (method == 0) {
            mul_mat_vec_f32_0(src0, src1, dst, N, M);
        }

        if (method == 1) {
            mul_mat_vec_f16_0(src0_fp16, src1_fp16, dst, N, M);
        }

        if (method == 2) {
            mul_mat_vec_f16_1(src0_fp16, src1_fp16, dst, N, M);
        }
    }
    // 遍历数组，计算累加和
    for (int i = 0; i < N; i++) {
        sum += dst[i]*iM;
    }

    // 计算程序结束时的时钟时间和微秒时间
    {
        const clock_t end = clock();
        const uint64_t end_us = get_time_us();
        // 打印函数名和经过的时钟周期数
        printf("%s: elapsed ticks: %ld\n",  __func__, end - start);
        // 打印函数名和经过的微秒数，以及平均每次迭代的毫秒数
        printf("%s: elapsed us:    %llu / %f ms\n",  __func__, end_us - start_us, (end_us - start_us) / 1000.0 / nIter);
    }

    // 打印累加和
    printf("%f\n", sum);

    // 释放动态分配的内存
    free(src0);
    free(src1);
    free(dst);

    free(src0_fp16);
    free(src1_fp16);

    // 返回 0 表示程序正常结束
    return 0;
# 闭合前面的函数定义
```