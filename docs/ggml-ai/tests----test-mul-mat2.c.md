# `ggml\tests\test-mul-mat2.c`

```cpp
// 量化矩阵乘法

#include "ggml.h"  // 包含自定义的头文件 ggml.h

#include <float.h>  // 包含浮点数的定义
#include <stdint.h>  // 包含整数类型的定义
#include <stdio.h>  // 包含标准输入输出函数
#include <inttypes.h>  // 包含整数格式转换
#include <assert.h>  // 包含断言函数
#include <stdlib.h>  // 包含标准库函数
#include <string.h>  // 包含字符串处理函数
#include <math.h>  // 包含数学函数

#if defined(__ARM_NEON)  // 如果是 ARM 平台下的 NEON 指令集
#include "arm_neon.h"  // 包含 ARM NEON 指令集的头文件
#elif defined(__AVX__) || defined(__AVX2__)  // 如果是 AVX 或 AVX2 指令集
#include "immintrin.h"  // 包含 AVX 或 AVX2 指令集的头文件
#endif

#ifndef MIN  // 如果未定义 MIN 宏
#define MAX(a, b) ((a) > (b) ? (a) : (b))  // 定义 MAX 宏
#define MIN(a, b) ((a) < (b) ? (a) : (b))  // 定义 MIN 宏
#endif

#if defined(_MSC_VER)  // 如果是 Microsoft Visual C++ 编译器
#pragma warning(disable: 4244 4267) // 禁止 4244 和 4267 警告
#include <intrin.h>  // 包含内联汇编函数的头文件
#define __builtin_popcountll __popcnt64  // 定义内置的 popcountll 函数
#endif

const int M = 1280;  // 定义常量 M 的值为 1280
const int N = 1536;  // 定义常量 N 的值为 1536
const int K = 1280;  // 定义常量 K 的值为 1280

//const int M = 64;
//const int N = 64;
//const int K = 64;

#define QK 64  // 定义 QK 宏为 64
#define QB 4  // 定义 QB 宏为 4

//#define GGML_GQ_USE_FP16_SCALE

#if defined(GGML_GQ_USE_FP16_SCALE)  // 如果定义了 GGML_GQ_USE_FP16_SCALE 宏
#define gq_scale_t ggml_fp16_t  // 定义 gq_scale_t 类型为 ggml_fp16_t
#define GGML_FP32_TO_GQ(x) ggml_fp32_to_fp16(x)  // 定义 GGML_FP32_TO_GQ 宏
#define GGML_GQ_TO_FP32(x) ggml_fp16_to_fp32(x)  // 定义 GGML_GQ_TO_FP32 宏
#else  // 如果未定义 GGML_GQ_USE_FP16_SCALE 宏
#define gq_scale_t float  // 定义 gq_scale_t 类型为 float
#define GGML_FP32_TO_GQ(x) (x)  // 定义 GGML_FP32_TO_GQ 宏
#define GGML_GQ_TO_FP32(x) (x)  // 定义 GGML_GQ_TO_FP32 宏
#endif

#define gq_t_bits 64  // 定义 gq_t_bits 常量为 64
#define gq_quant_t uint64_t  // 定义 gq_quant_t 类型为 uint64_t

float frand(void) {  // 定义 frand 函数，返回浮点数
    return (float) rand() / (float) RAND_MAX;  // 返回随机数除以 RAND_MAX 的浮点数
}

#if defined(__AVX2__)  // 如果是 AVX2 指令集
// 水平合并 8 个 32 位整数
static inline uint32_t _mm256_hadd_epi32_gg(__m256i v) {  // 定义内联函数 _mm256_hadd_epi32_gg
    __m128i v0 = _mm256_extractf128_si256(v, 0);  // 提取 v 的低 128 位
    __m128i v1 = _mm256_extractf128_si256(v, 1);  // 提取 v 的高 128 位

    v0 = _mm_add_epi32(v0, v1);  // 低 128 位和高 128 位相加

    v1 = _mm_shuffle_epi32(v0, 0x0e);  // 交换 v0 的元素
    v0 = _mm_add_epi32(v0, v1);  // v0 和交换后的 v0 相加

    v1 = _mm_shuffle_epi32(v0, 0x01);  // 再次交换 v0 的元素
    v0 = _mm_add_epi32(v0, v1);  // v0 和交换后的 v0 相加

    return _mm_cvtsi128_si32(v0);  // 返回 v0 的第一个元素
}

// 水平合并 32 个 8 位整数
static inline int32_t _mm256_hadd_epi8_gg(__m256i v0) {  // 定义内联函数 _mm256_hadd_epi8_gg
    # 使用 AVX2 指令集中的 _mm256_set1_epi8 创建一个全为1的 256 位整数向量，并与 v0 中的每个字节相乘后相加
    __m256i v1 = _mm256_maddubs_epi16(v0, _mm256_set1_epi8(1));
    # 使用 AVX2 指令集中的 _mm256_set1_epi16 创建一个全为1的 256 位整数向量，并与 v1 中的每个元素相乘后相加
    __m256i v2 = _mm256_madd_epi16   (v1, _mm256_set1_epi16(1));
    # 使用 AVX2 指令集中的 _mm256_hadd_epi32_gg 对 v2 中的相邻元素进行水平相加
    return _mm256_hadd_epi32_gg(v2);
// 计算给定 __m256 类型的向量 v 的水平加法结果
static inline float _mm256_hadd_ps_gg(__m256 v) {
    // 将 v 的前 128 位和后 128 位相加
    const __m128 t0 = _mm_add_ps(_mm256_castps256_ps128(v), _mm256_extractf128_ps(v, 1));
    // 对 t0 进行水平加法
    const __m128 t1 = _mm_hadd_ps(t0, t0);

    // 对 t1 进行水平加法，并返回结果
    return _mm_cvtss_f32(_mm_hadd_ps(t1, t1));
}
#endif

//
// naive implementation
//

// 用于计算矩阵乘法的朴素实现
void mul_mat_f32_naive(
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

//
// method 1
//

// 计算每行的块数
static inline int quantize_1_blocks_per_row(int k) {
    return k/QK;
}

// 计算每个块的量化数
static inline int quantize_1_quants_per_block(void) {
    return QK/gq_t_bits;
}

// 计算每行的大小
static inline int quantize_1_row_size(int k) {
    const int nb = quantize_1_blocks_per_row(k);
    const int nq = quantize_1_quants_per_block();

    return nb*(2*sizeof(gq_scale_t) + nq*QB*sizeof(gq_quant_t));
}

// 对输入的浮点数进行量化
void quantize_1(const float * src, void * dst, int n, int k) {
    char * p0 = dst;

    gq_quant_t pp[QB];

    for (int j = 0; j < n; j++) {
        for (int i = 0; i < k/QK; i++) {
            float min = FLT_MAX;
            float max = -FLT_MAX;

            // 找到最小值和最大值
#ifdef __ARM_NEON
            {
                // 使用 NEON 指令集进行 SIMD 计算，初始化最小值和最大值为最大和最小的浮点数
                float32x4_t minv = vdupq_n_f32(FLT_MAX);
                float32x4_t maxv = vdupq_n_f32(-FLT_MAX);

                // 使用 NEON 指令集进行 SIMD 计算，遍历数组并更新最小值和最大值
                for (int l = 0; l < QK; l += 4) {
                    float32x4_t v = vld1q_f32(src + j*k + i*QK + l);
                    minv = vminq_f32(minv, v);
                    maxv = vmaxq_f32(maxv, v);
                }

                // 使用 NEON 指令集进行 SIMD 计算，将最小值和最大值进行归约操作
                float32x2_t minv32 = vpmin_f32(vget_low_f32(minv), vget_high_f32(minv));
                float32x2_t maxv32 = vpmax_f32(vget_low_f32(maxv), vget_high_f32(maxv));

                // 从归约结果中获取最小值和最大值
                min = MIN(vget_lane_f32(minv32, 0), vget_lane_f32(minv32, 1));
                max = MAX(vget_lane_f32(maxv32, 0), vget_lane_f32(maxv32, 1));

                //printf("SIMD min/max: %f %f\n", min, max);
            }
#else
            {
                // 如果不支持 NEON 指令集，则使用普通循环遍历数组并更新最小值和最大值
                for (int l = 0; l < QK; l++) {
                    const float v = src[j*k + i*QK + l];
                    if (v < min) min = v;
                    if (v > max) max = v;
                }

                //printf("NORM min/max: %f %f\n", min, max);
            }
#endif

            // 计算量化步长
            const float d = (max - min) / ((1 << QB) - 1);
            // 计算倒数，避免除零错误
            const float id = d ? 1.0/d : 0.0;

            // 将最小值和步长拷贝到目标地址，并更新指针位置
            memcpy(p0, &min, sizeof(float)); p0 += sizeof(float);
            memcpy(p0, &d,   sizeof(float)); p0 += sizeof(float);

            // 打印最小值、最大值、步长和倒数的值
            //printf("min/max/d/id: %f %f %f %f\n", min, max, d, id);

            // 对输入数据进行量化
            for (int s = 0; s < QK/gq_t_bits; ++s) {
                // 将 pp 数组清零
                memset(pp, 0, sizeof(pp));

                for (int l = 0; l < gq_t_bits; l++) {
                    // 从输入数据中获取值
                    const   float v = src[j*k + i*QK + s*gq_t_bits + l];
                    // 计算量化值
                    const uint8_t q = (v - min)*id;

                    for (int b = 0; b < QB; b++) {
                        // 将量化值存储到 pp 数组中
                        pp[b] |= q & (1 << b) ? (1ULL << l) : 0;
                    }
                }

                for (int b = 0; b < QB; b++) {
                    // 将量化后的数据拷贝到目标地址，并更新指针位置
                    memcpy(p0, &pp[b], sizeof(gq_quant_t)); p0 += sizeof(gq_quant_t);
                }
            }
        }
    }
}

// 矩阵乘法函数
void mul_mat_gq_1(
    const void * src0,
    const void * src1,
         float * dst,
    int m, int n, int k) {
    // 对 k 取整，确保是 gq_t_bits 的整数倍
    const int kp = k & ~(gq_t_bits - 1);

    // 定义指向输入数据的指针
    const char * restrict p0 = src0;
    const char * restrict p1 = src1;

    // 定义存储输入数据的数组
    float s0[QB + 1];
    float s1[QB + 1];

    // 定义存储量化数据的数组
    gq_quant_t m0[QB + 1];
    gq_quant_t m1[QB + 1];
    // 遍历矩阵的行
    for (int ir0 = 0; ir0 < m; ir0++) {
        // 遍历矩阵的列
        for (int ir1 = 0; ir1 < n; ir1++) {
            // 初始化浮点数求和变量
            float sumf = 0.0;

            // 计算指向矩阵 p0 和 p1 的指针
            const char * restrict pp0 = p0 + ir0*((2*sizeof(float) + (QK/gq_t_bits)*QB*sizeof(gq_quant_t))*(k/QK));
            const char * restrict pp1 = p1 + ir1*((2*sizeof(float) + (QK/gq_t_bits)*QB*sizeof(gq_quant_t))*(k/QK));

            // 遍历矩阵的列，每次处理 QK 个元素
            for (int i = 0; i < kp/QK; i++) {
                // 定义并读取 min0 和 d0
                float min0, d0;
                memcpy(&min0, pp0, sizeof(float)); pp0 += sizeof(float);
                memcpy(&d0,   pp0, sizeof(float)); pp0 += sizeof(float);

                // 定义并读取 min1 和 d1
                float min1, d1;
                memcpy(&min1, pp1, sizeof(float)); pp1 += sizeof(float);
                memcpy(&d1,   pp1, sizeof(float)); pp1 += sizeof(float);

                // 打印 min0, d0, min1, d1 的值
                //printf("min0/d0 = %f %f | min1/d1 = %f %f\n", min0, d0, min1, d1);
#if 1
                // 如果条件为真，则执行以下代码块

                // 初始化 s0 和 s1 数组的第一个元素
                s0[0] = min0;
                s1[0] = min1;

                // 循环遍历 QB 次
                for (int b = 0; b < QB; b++) {
                    // 计算并赋值 s0 和 s1 数组的后续元素
                    s0[b + 1] = d0*(1 << b);
                    s1[b + 1] = d1*(1 << b);
                }

                // 初始化 m0 和 m1 数组的第一个元素
                m0[0] = 0-1ULL;
                m1[0] = 0-1ULL;

                // 循环遍历 QK/gq_t_bits 次
                for (int s = 0; s < QK/gq_t_bits; ++s) {
                    // 循环遍历 QB 次
                    for (int b = 0; b < QB; b++) {
                        // 将 pp0 指向的 gq_quant_t 类型数据拷贝到 m0 和 m1 数组中
                        memcpy(&m0[b + 1], pp0, sizeof(gq_quant_t)); pp0 += sizeof(gq_quant_t);
                        memcpy(&m1[b + 1], pp1, sizeof(gq_quant_t)); pp1 += sizeof(gq_quant_t);
                    }

                    // 循环遍历 QB+1 次
                    for (int q0 = 0; q0 < QB + 1; q0++) {
                        // 循环遍历 QB+1 次
                        for (int q1 = 0; q1 < QB + 1; q1++) {
                            // 计算 sumf 的值
                            sumf += s0[q0]*s1[q1]*__builtin_popcountll(m0[q0] & m1[q1]);
                        }
                    }
                }
#else
#endif
            }

            // 将 sumf 赋值给 dst 数组的指定位置
            dst[ir0*n + ir1] = sumf;
        }
    }
}

//
// method 2
// n-bit quantization (2nd attempt)
//

// 返回每行需要分块的数量
static inline int quantize_2_blocks_per_row(int k) {
    return k/QK;
}

// 返回每个块需要的量化数量
static inline int quantize_2_quants_per_block(void) {
    return QK/gq_t_bits;
}

// 返回每行的大小
static inline int quantize_2_row_size(int k) {
    // 计算每行需要分块的数量
    const int nb = quantize_2_blocks_per_row(k);
    // 计算每个块需要的量化数量
    const int nq = quantize_2_quants_per_block();

    // 返回每行的大小
    return nb*(2*sizeof(gq_scale_t) + nq*QB*sizeof(gq_quant_t));
}

// 对一行数据进行量化
void quantize_2_row(const float * restrict src, void * restrict dst, int k) {
    // 断言 k 可以被 QK 整除
    assert(k % QK == 0);

    // 计算每行需要分块的数量
    const int nb = quantize_2_blocks_per_row(k);
    // 计算每个块需要的量化数量
    const int nq = quantize_2_quants_per_block();

    // 定义指向 dst 的指针，并分别指向不同类型的数据
    gq_scale_t * restrict pm = (gq_scale_t *) (dst);
    gq_scale_t * restrict pd = (gq_scale_t *) (pm + nb);
    gq_quant_t * restrict pb = (gq_quant_t *) (pd + nb);

    // 定义长度为 QB 的 gq_quant_t 类型数组
    gq_quant_t pp[QB];
    # 定义一个包含32个整数的常量数组
    static const int32_t sh[32] = {
        0,  1,  2,  3,  4,  5,  6,  7,  8,  9,  10, 11, 12, 13, 14, 15,
        16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31,
    };
    
    # 对数组进行循环遍历
    for (int i = 0; i < nb; i++) {
        # 初始化最小值为浮点数的最大值
        float min = FLT_MAX;
        # 初始化最大值为负的浮点数的最大值
        float max = -FLT_MAX;
#ifdef __ARM_NEON
        {
            // 创建一个包含四个元素的向量，每个元素的值都是 FLT_MAX
            float32x4_t minv = vdupq_n_f32(FLT_MAX);
            // 创建一个包含四个元素的向量，每个元素的值都是 -FLT_MAX
            float32x4_t maxv = vdupq_n_f32(-FLT_MAX);

            // 循环遍历 QK，每次处理四个元素
            for (int l = 0; l < QK; l += 4) {
                // 从 src 中加载四个单精度浮点数到向量 v
                float32x4_t v = vld1q_f32(src + i*QK + l);
                // 比较两个向量的对应元素，取较小值
                minv = vminq_f32(minv, v);
                // 比较两个向量的对应元素，取较大值
                maxv = vmaxq_f32(maxv, v);
            }

            // 将 minv 向量的低两个元素和高两个元素分别进行两两比较，取较小值
            float32x2_t minv32 = vpmin_f32(vget_low_f32(minv), vget_high_f32(minv));
            // 将 maxv 向量的低两个元素和高两个元素分别进行两两比较，取较大值
            float32x2_t maxv32 = vpmax_f32(vget_low_f32(maxv), vget_high_f32(maxv));

            // 从 minv32 向量中取出两个元素，分别比较取较小值
            min = MIN(vget_lane_f32(minv32, 0), vget_lane_f32(minv32, 1));
            // 从 maxv32 向量中取出两个元素，分别比较取较大值
            max = MAX(vget_lane_f32(maxv32, 0), vget_lane_f32(maxv32, 1));
        }
#else
        {
            // 遍历 QK，逐个处理
            for (int l = 0; l < QK; l++) {
                // 从 src 中取出一个单精度浮点数
                const float v = src[i*QK + l];
                // 如果 v 小于 min，则更新 min
                if (v < min) min = v;
                // 如果 v 大于 max，则更新 max
                if (v > max) max = v;
            }
        }
#endif

        // 计算 d 的值
        const float d = (max - min) / ((1 << QB) - 1);
        // 如果 d 不为 0，则计算 id 的值，否则 id 为 0
        const float id = d ? 1.0/d : 0.0;

        // 将 min 转换为 GQ 格式后存入 pm 数组
        pm[i] = GGML_FP32_TO_GQ(min);
        // 将 d 转换为 GQ 格式后存入 pd 数组
        pd[i] = GGML_FP32_TO_GQ(d);

        // 遍历 nq 次
        for (int s = 0; s < nq; ++s) {
            // 将 pp 数组清零
            memset(pp, 0, sizeof(pp));

#if 1
            // 遍历 gq_t_bits 次
            for (int l = 0; l < gq_t_bits; l++) {
                // 从 src 中取出一个单精度浮点数
                const   float v = src[i*QK + s*gq_t_bits + l];
                // 计算 q 的值
                const uint8_t q = (v - min)*id + frand();

                // 遍历 QB 次
                for (int b = 0; b < QB; b++) {
                    // 如果 q 的第 b 位为 1，则将 pp[b] 的第 l 位设为 1
                    pp[b] |= q & (1 << b) ? (1ULL << l) : 0;
                }
            }
#elif defined(__ARM_NEON)
#if 1
            {
                // 定义一个长度为 8*QB 的无符号整型数组 ppt
                uint32_t ppt[2*4*QB];

                // 创建一个包含 min 值的长度为 4 的 float32x4_t 类型向量 minv
                float32x4_t minv = vdupq_n_f32(min);
                // 创建一个包含 id 值的长度为 4 的 float32x4_t 类型向量 idv
                float32x4_t idv  = vdupq_n_f32(id);

                // 断言 gq_t_bits 除以 16 的余数为 0
                assert(gq_t_bits % 16 == 0);

                // 创建长度为 QB 的 uint32x4_t 类型数组 p0，初始化为 0
                uint32x4_t p0[QB] = { vdupq_n_u32(0) };
                // 创建长度为 QB 的 uint32x4_t 类型数组 p1，初始化为 0
                uint32x4_t p1[QB] = { vdupq_n_u32(0) };

                // 循环遍历 gq_t_bits，每次增加 16
                for (int l = 0; l < gq_t_bits; l += 16) {
                    // 从 src 数组中加载长度为 4 的 float32x4_t 类型向量到 v0
                    float32x4_t v0 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 0);
                    // 从 src 数组中加载长度为 4 的 float32x4_t 类型向量到 v1
                    float32x4_t v1 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 4);
                    // 从 src 数组中加载长度为 4 的 float32x4_t 类型向量到 v2
                    float32x4_t v2 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 8);
                    // 从 src 数组中加载长度为 4 的 float32x4_t 类型向量到 v3
                    float32x4_t v3 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 12);

                    // 对 v0 中的每个元素减去 minv 中的对应元素
                    v0 = vsubq_f32(v0, minv);
                    // 对 v1 中的每个元素减去 minv 中的对应元素
                    v1 = vsubq_f32(v1, minv);
                    // 对 v2 中的每个元素减去 minv 中的对应元素
                    v2 = vsubq_f32(v2, minv);
                    // 对 v3 中的每个元素减去 minv 中的对应元素
                    v3 = vsubq_f32(v3, minv);

                    // 对 v0 中的每个元素乘以 idv 中的对应元素
                    v0 = vmulq_f32(v0, idv);
                    // 对 v1 中的每个元素乘以 idv 中的对应元素
                    v1 = vmulq_f32(v1, idv);
                    // 对 v2 中的每个元素乘以 idv 中的对应元素
                    v2 = vmulq_f32(v2, idv);
                    // 对 v3 中的每个元素乘以 idv 中的对应元素
                    v3 = vmulq_f32(v3, idv);

#if 1
                    // 对 v0 中的每个元素加上一个随机浮点数
                    v0[0] += frand(); v0[1] += frand(); v0[2] += frand(); v0[3] += frand();
                    // 对 v1 中的每个元素加上一个随机浮点数
                    v1[0] += frand(); v1[1] += frand(); v1[2] += frand(); v1[3] += frand();
                    // 对 v2 中的每个元素加上一个随机浮点数
                    v2[0] += frand(); v2[1] += frand(); v2[2] += frand(); v2[3] += frand();
                    // 对 v3 中的每个元素加上一个随机浮点数
                    v3[0] += frand(); v3[1] += frand(); v3[2] += frand(); v3[3] += frand();
#endif

                    // 将四个单精度浮点数向量转换为四个无符号整数向量
                    uint32x4_t q0 = vcvtq_u32_f32(v0);
                    uint32x4_t q1 = vcvtq_u32_f32(v1);
                    uint32x4_t q2 = vcvtq_u32_f32(v2);
                    uint32x4_t q3 = vcvtq_u32_f32(v3);

                    // 遍历 QB 次
                    for (int b = 0; b < QB; ++b) {
                        // 创建一个包含 1<<b 的无符号整数向量
                        uint32x4_t m = vdupq_n_u32(1 << b);
                        // 创建一个包含 -b 的无符号整数向量
                        uint32x4_t r = vdupq_n_u32(-b);

                        // 如果 l 小于 32
                        if (l < 32) {
                            // 对 p0[b] 进行按位或运算
                            p0[b] = vorrq_u32(p0[b], vshlq_u32(vshlq_u32(vandq_u32(q0, m), r), vld1q_s32(sh + l + 0)));
                            p0[b] = vorrq_u32(p0[b], vshlq_u32(vshlq_u32(vandq_u32(q1, m), r), vld1q_s32(sh + l + 4)));
                            p0[b] = vorrq_u32(p0[b], vshlq_u32(vshlq_u32(vandq_u32(q2, m), r), vld1q_s32(sh + l + 8)));
                            p0[b] = vorrq_u32(p0[b], vshlq_u32(vshlq_u32(vandq_u32(q3, m), r), vld1q_s32(sh + l + 12)));
                        } else {
                            // 对 p1[b] 进行按位或运算
                            p1[b] = vorrq_u32(p1[b], vshlq_u32(vshlq_u32(vandq_u32(q0, m), r), vld1q_s32(sh + l - 32)));
                            p1[b] = vorrq_u32(p1[b], vshlq_u32(vshlq_u32(vandq_u32(q1, m), r), vld1q_s32(sh + l - 28)));
                            p1[b] = vorrq_u32(p1[b], vshlq_u32(vshlq_u32(vandq_u32(q2, m), r), vld1q_s32(sh + l - 24)));
                            p1[b] = vorrq_u32(p1[b], vshlq_u32(vshlq_u32(vandq_u32(q3, m), r), vld1q_s32(sh + l - 20)));
                        }
                    }
                }
#if QB == 4
                # 如果 QB 等于 4，则执行以下代码块
                vst1q_u32((uint32_t *) ppt + 0,  p0[0]);
                # 将 p0[0] 中的值存储到 ppt 数组的第一个位置
                vst1q_u32((uint32_t *) ppt + 4,  p1[0]);
                # 将 p1[0] 中的值存储到 ppt 数组的第五个位置
                vst1q_u32((uint32_t *) ppt + 8,  p0[1]);
                # 将 p0[1] 中的值存储到 ppt 数组的第九个位置
                vst1q_u32((uint32_t *) ppt + 12, p1[1]);
                # 将 p1[1] 中的值存储到 ppt 数组的第十三个位置
                vst1q_u32((uint32_t *) ppt + 16, p0[2]);
                # 将 p0[2] 中的值存储到 ppt 数组的第十七个位置
                vst1q_u32((uint32_t *) ppt + 20, p1[2]);
                # 将 p1[2] 中的值存储到 ppt 数组的第二十一个位置
                vst1q_u32((uint32_t *) ppt + 24, p0[3]);
                # 将 p0[3] 中的值存储到 ppt 数组的第二十五个位置
                vst1q_u32((uint32_t *) ppt + 28, p1[3]);
                # 将 p1[3] 中的值存储到 ppt 数组的第二十九个位置

                pp[0] = (ppt[0]  | ppt[1]  | ppt[2]  | ppt[3] ) | ((uint64_t) (ppt[4]  | ppt[5]  | ppt[6]  | ppt[7]) ) << 32;
                # 将 ppt 数组中的前八个元素按位或运算，并将结果存储到 pp 数组的第一个位置
                pp[1] = (ppt[8]  | ppt[9]  | ppt[10] | ppt[11]) | ((uint64_t) (ppt[12] | ppt[13] | ppt[14] | ppt[15])) << 32;
                # 将 ppt 数组中的第九到第十六个元素按位或运算，并将结果存储到 pp 数组的第二个位置
                pp[2] = (ppt[16] | ppt[17] | ppt[18] | ppt[19]) | ((uint64_t) (ppt[20] | ppt[21] | ppt[22] | ppt[23])) << 32;
                # 将 ppt 数组中的第十七到第二十四个元素按位或运算，并将结果存储到 pp 数组的第三个位置
                pp[3] = (ppt[24] | ppt[25] | ppt[26] | ppt[27]) | ((uint64_t) (ppt[28] | ppt[29] | ppt[30] | ppt[31])) << 32;
                # 将 ppt 数组中的第二十五到第三十二个元素按位或运算，并将结果存储到 pp 数组的第四个位置
#else
                # 如果 QB 不等于 4，则执行以下代码块
                for (int b = 0; b < QB; ++b) {
                    # 遍历 QB 次，执行以下代码块
                    vst1q_u32((uint32_t *) ppt + 0,  p0[b]);
                    # 将 p0[b] 中的值存储到 ppt 数组的第一个位置
                    vst1q_u32((uint32_t *) ppt + 4,  p1[b]);
                    # 将 p1[b] 中的值存储到 ppt 数组的第五个位置

                    pp[b] = (ppt[0] | ppt[1] | ppt[2] | ppt[3]) | ((uint64_t) (ppt[4] | ppt[5] | ppt[6] | ppt[7])) << 32;
                    # 将 ppt 数组中的前八个元素按位或运算，并将结果存储到 pp 数组的第 b 个位置
                }
#endif
            }
// 如果不满足条件，使用较低效的SIMD处理方式
{
    // 创建一个包含最小值的四元素向量
    float32x4_t minv = vdupq_n_f32(min);
    // 创建一个包含id值的四元素向量
    float32x4_t idv  = vdupq_n_f32(id);

    // 断言gq_t_bits的值为64
    assert(gq_t_bits == 64);
    // 创建一个长度为gq_t_bits的无符号8位整数数组
    uint8_t qq[gq_t_bits];

    // 循环遍历，每次处理16个元素
    for (int l = 0; l < gq_t_bits; l += 16) {
        // 从src指针指向的位置读取四个单精度浮点数，存储到v0中
        float32x4_t v0 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 0);
        // 从src指针指向的位置读取四个单精度浮点数，存储到v1中
        float32x4_t v1 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 4);
        // 从src指针指向的位置读取四个单精度浮点数，存储到v2中
        float32x4_t v2 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 8);
        // 从src指针指向的位置读取四个单精度浮点数，存储到v3中
        float32x4_t v3 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 12);

        // 对v0中的每个元素减去minv中对应的元素
        v0 = vsubq_f32(v0, minv);
        // 对v1中的每个元素减去minv中对应的元素
        v1 = vsubq_f32(v1, minv);
        // 对v2中的每个元素减去minv中对应的元素
        v2 = vsubq_f32(v2, minv);
        // 对v3中的每个元素减去minv中对应的元素
        v3 = vsubq_f32(v3, minv);

        // 将v0中的每个元素乘以idv中对应的元素
        v0 = vmulq_f32(v0, idv);
        // 将v1中的每个元素乘以idv中对应的元素
        v1 = vmulq_f32(v1, idv);
        // 将v2中的每个元素乘以idv中对应的元素
        v2 = vmulq_f32(v2, idv);
        // 将v3中的每个元素乘以idv中对应的元素
        v3 = vmulq_f32(v3, idv);

#if 0
        // 下面的代码被注释掉了，不会执行
#endif

                    // 将四个单精度浮点数转换为四个无符号整数
                    uint32x4_t q0 = vcvtq_u32_f32(v0);
                    uint32x4_t q1 = vcvtq_u32_f32(v1);
                    uint32x4_t q2 = vcvtq_u32_f32(v2);
                    uint32x4_t q3 = vcvtq_u32_f32(v3);

                    // 将四个无符号整数转换为八个无符号字节，并存储到指定位置
                    vst1_u8(qq + l + 0, vmovn_u16(vcombine_u16(vmovn_u32(q0), vmovn_u32(q1))));
                    vst1_u8(qq + l + 8, vmovn_u16(vcombine_u16(vmovn_u32(q2), vmovn_u32(q3))));
                }

                for (int l = 0; l < gq_t_bits; l++) {
                    for (int b = 0; b < QB; b++) {
                        const uint64_t ql = qq[l];
                        // 将 qq[l] 的第 b 位与 pp[b] 进行按位与操作，并根据结果更新 pp[b] 的值
                        pp[b] |= ((ql & (1 << b)) >> b) << l;
                    }
                }
            }
#endif
#endif
            // 将 pp 数组的内容复制到指定位置
            memcpy(pb + i*nq*QB + s*QB, pp, sizeof(pp));
        }
    }
}

// 使用 quantize_2_row 重新实现 quantize_2
void quantize_2(const float * restrict src, char * restrict dst, int n, int k) {
    // 确保 k 是 QK 的倍数
    assert(k % QK == 0);

    for (int j = 0; j < n; j++) {
        // 对每一行数据进行 quantize_2_row 操作
        quantize_2_row(src + j*k, dst, k);
        // 更新 dst 指针的位置
        dst = (char *) dst + quantize_2_row_size(k);
    }
}

// 计算向量的点积，使用 quantize_2 进行量化
void vec_dot_gq_2(const int n, float * restrict s, const void * restrict x, const void * restrict y) {
    // 计算每行数据的块数和每个块的量化数
    const int nb = quantize_2_blocks_per_row(n);
    const int nq = quantize_2_quants_per_block();

    // 将输入指针转换为特定类型的指针
    const gq_scale_t * restrict pm0 = (const gq_scale_t *) x;
    const gq_scale_t * restrict pm1 = (const gq_scale_t *) y;

    const gq_scale_t * restrict pd0 = pm0 + nb;
    const gq_scale_t * restrict pd1 = pm1 + nb;

    const gq_quant_t * restrict pb0 = (const gq_quant_t *) (pd0 + nb);
    const gq_quant_t * restrict pb1 = (const gq_quant_t *) (pd1 + nb);

    float sumf = 0.0;

#if 1
    # 循环遍历从 0 到 nb-1 的整数
    for (int i = 0; i < nb; i++) {
        # 将pm0[i]转换为float类型并赋值给m0
        const float m0 = GGML_GQ_TO_FP32(pm0[i]);
        # 将pd0[i]转换为float类型并赋值给d0
        const float d0 = GGML_GQ_TO_FP32(pd0[i]);

        # 将pm1[i]转换为float类型并赋值给m1
        const float m1 = GGML_GQ_TO_FP32(pm1[i]);
        # 将pd1[i]转换为float类型并赋值给d1
        const float d1 = GGML_GQ_TO_FP32(pd1[i]);
# 如果 QB 等于 4
        # 初始化三个整型变量，用于存储计算结果
        int isum01 = 0;
        int isum10 = 0;
        int isum11 = 0;

        # 遍历 nq 次
        for (int s = 0; s < nq; ++s) {
            # 定义指向特定位置的指针，用于访问数组中的元素
            const gq_quant_t * restrict mm0 = pb0 + i*nq*QB + s*QB;
            const gq_quant_t * restrict mm1 = pb1 + i*nq*QB + s*QB;

            # 定义一个宏，用于计算一个 64 位整数中二进制 1 的个数
#define bpcnt(x) __builtin_popcountll(x)
            # 根据 mm1 数组中的元素计算 isum01 的值
            isum01 += (1 << 0)*(bpcnt(mm1[0]));
            isum01 += (1 << 1)*(bpcnt(mm1[1]));
            isum01 += (1 << 2)*(bpcnt(mm1[2]));
            isum01 += (1 << 3)*(bpcnt(mm1[3]));

            # 根据 mm0 数组中的元素计算 isum10 的值
            isum10 += (1 << 0)*(bpcnt(mm0[0]));
            isum10 += (1 << 1)*(bpcnt(mm0[1]));
            isum10 += (1 << 2)*(bpcnt(mm0[2]));
            isum10 += (1 << 3)*(bpcnt(mm0[3]));

            # 根据 mm0 和 mm1 数组中的元素计算 isum11 的值
            isum11 += (1 << 0)*(bpcnt(mm0[0] & mm1[0]));
            isum11 += (1 << 1)*(bpcnt(mm0[0] & mm1[1]) + bpcnt(mm0[1] & mm1[0]));
            isum11 += (1 << 2)*(bpcnt(mm0[0] & mm1[2]) + bpcnt(mm0[1] & mm1[1]) + bpcnt(mm0[2] & mm1[0]));
            isum11 += (1 << 3)*(bpcnt(mm0[0] & mm1[3]) + bpcnt(mm0[1] & mm1[2]) + bpcnt(mm0[2] & mm1[1]) + bpcnt(mm0[3] & mm1[0]));
            isum11 += (1 << 4)*(bpcnt(mm0[1] & mm1[3]) + bpcnt(mm0[2] & mm1[2]) + bpcnt(mm0[3] & mm1[1]));
            isum11 += (1 << 5)*(bpcnt(mm0[2] & mm1[3]) + bpcnt(mm0[3] & mm1[2]));
            isum11 += (1 << 6)*(bpcnt(mm0[3] & mm1[3]));
#undef bpcnt
        }

        # 计算 sumf 的值
        sumf += nq*gq_t_bits*(m0*m1) + isum01*(m0*d1) + isum10*(m1*d0) + isum11*(d0*d1);
# 如果 QB 等于 3
        # 初始化三个整型变量，用于存储计算结果
        int isum01 = 0;
        int isum10 = 0;
        int isum11 = 0;

        # 遍历 nq 次
        for (int s = 0; s < nq; ++s) {
            # 定义指向特定位置的指针，用于访问数组中的元素
            const gq_quant_t * restrict mm0 = pb0 + i*nq*QB + s*QB;
            const gq_quant_t * restrict mm1 = pb1 + i*nq*QB + s*QB;

            # 根据 gq_t_bits 的值选择合适的宏，用于计算一个整数中二进制 1 的个数
#if gq_t_bits == 32
#define bpcnt(x) __builtin_popcount(x)
#else
#define bpcnt(x) __builtin_popcountll(x)
        # 如果定义了 QB，则执行以下代码块
        sumf += nq*gq_t_bits*(m0*m1) + isum01*(m0*d1) + isum10*(m1*d0) + isum11*(d0*d1);
#elif QB == 2
        # 定义并初始化 isum01、isum10、isum11 为 0
        int isum01 = 0;
        int isum10 = 0;
        int isum11 = 0;

        # 遍历 nq 次
        for (int s = 0; s < nq; ++s) {
            # 定义并初始化 mm0 和 mm1
            const gq_quant_t * restrict mm0 = pb0 + i*nq*QB + s*QB;
            const gq_quant_t * restrict mm1 = pb1 + i*nq*QB + s*QB;

            # 根据 gq_t_bits 的值选择不同的内置函数进行位计数
            # 如果 gq_t_bits 为 32，则使用 __builtin_popcount(x) 函数
            # 否则，使用 __builtin_popcountll(x) 函数
            isum01 += (1 << 0)*(bpcnt(mm1[0]));
            isum01 += (1 << 1)*(bpcnt(mm1[1]));

            isum10 += (1 << 0)*(bpcnt(mm0[0]));
            isum10 += (1 << 1)*(bpcnt(mm0[1]));

            isum11 += (1 << 0)*(bpcnt(mm0[0] & mm1[0]));
            isum11 += (1 << 1)*(bpcnt(mm0[0] & mm1[1]) + bpcnt(mm0[1] & mm1[0]));
            isum11 += (1 << 2)*(bpcnt(mm0[1] & mm1[1]));
        }

        # 计算 sumf 的值
        sumf += nq*gq_t_bits*(m0*m1) + isum01*(m0*d1) + isum10*(m1*d0) + isum11*(d0*d1);
#else
        // 定义长度为 QB+1 的浮点数数组s0和s1
        float s0[QB + 1];
        float s1[QB + 1];

        // 将m0和m1分别赋值给s0和s1的第一个元素
        s0[0] = m0;
        s1[0] = m1;

        // 循环计算s0和s1的其他元素的值
        for (int b = 0; b < QB; b++) {
            s0[b + 1] = d0*(1 << b);
            s1[b + 1] = d1*(1 << b);
        }

        // 循环计算sumf的值
        for (int s = 0; s < nq; ++s) {
            for (int q0 = 0; q0 < QB + 1; q0++) {
                const gq_quant_t mm0 = q0 ? pb0[i*nq*QB + s*QB + q0 - 1] : -1ULL;
                for (int q1 = 0; q1 < QB + 1; q1++) {
                    const gq_quant_t mm1 = q1 ? pb1[i*nq*QB + s*QB + q1 - 1] : -1ULL;
                    sumf += s0[q0]*s1[q1]*__builtin_popcountll(mm0 & mm1);
                }
            }
        }
#endif
    }
#else
#error "not implemented"
#endif

    *s = sumf;
}

// 使用vec_dot_gq_2计算两行的点积
void mul_mat_gq_2(
    const void * src0,
    const void * src1, // 转置
         float * dst,
    int m, int n, int k) {
    assert(k % QK == 0);

    // 循环计算dst的值
    for (int ir0 = 0; ir0 < m; ir0++) {
        for (int ir1 = 0; ir1 < n; ir1++) {
            vec_dot_gq_2(k, dst + ir1, src0, src1);
            src1 = (const char *) src1 + quantize_2_row_size(k);
        }
        src0 = (const char *) src0 +   quantize_2_row_size(k);
        src1 = (const char *) src1 - n*quantize_2_row_size(k);

        dst = (float *) dst + n;
    }
}

//
// method 3
// (does not work)
//

// 计算每行的块数
static inline int quantize_3_blocks_per_row(int k) {
    return k/QK;
}

// 计算每个块的量化数
static inline int quantize_3_quants_per_block(void) {
    return QK/gq_t_bits;
}

// 计算每行的大小
static inline int quantize_3_row_size(int k) {
    const int nb = quantize_3_blocks_per_row(k);
    const int nq = quantize_3_quants_per_block();

    return nb*(sizeof(gq_scale_t) + nq*QB*sizeof(gq_quant_t));
}

// 对一行进行量化
void quantize_3_row(const float * restrict src, void * restrict dst, int k) {
    assert(k % QK == 0);

    const int nb = quantize_3_blocks_per_row(k);
    const int nq = quantize_3_quants_per_block();

    gq_scale_t * restrict pd = (gq_scale_t *) (dst);
    # 定义指针 pb，指向 pd + nb 的地址，类型为 gq_quant_t
    gq_quant_t * restrict pb = (gq_quant_t *) (pd + nb);

    # 定义数组 pp，长度为 QB，存储在栈上
    gq_quant_t pp[QB];

    # 定义静态常量数组 sh，长度为 32，存储了一组固定的整数值
    static const int32_t sh[32] = {
        0,  1,  2,  3,  4,  5,  6,  7,  8,  9,  10, 11, 12, 13, 14, 15,
        16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30, 31,
    };

    # 循环遍历 nb 次，初始化 amax 为 0.0f
    for (int i = 0; i < nb; i++) {
        float amax = 0.0f; // abs max
#ifdef __ARM_NEON
        {
            // 如果支持 ARM NEON 指令集
            // 计算绝对值最大值
            float32x4_t amaxv = vdupq_n_f32(0.0f);

            // 使用 NEON 指令集计算绝对值最大值
            for (int l = 0; l < QK; l += 4) {
                float32x4_t v = vld1q_f32(src + i*QK + l);
                amaxv = vmaxq_f32(amaxv, vabsq_f32(v));
            }

            // 将结果向量的低位和高位分别求最大值
            float32x2_t amaxv32 = vpmax_f32(vget_low_f32(amaxv), vget_high_f32(amaxv));

            // 取出最大值向量中的两个值中的较大值
            amax = MAX(vget_lane_f32(amaxv32, 0), vget_lane_f32(amaxv32, 1));
        }
#else
        {
            // 如果不支持 ARM NEON 指令集
            // 使用普通循环计算绝对值最大值
            for (int l = 0; l < QK; l++) {
                const float v = src[i*QK + l];
                amax = MAX(amax, fabsf(v));
            }
        }
#endif

        // 计算量化因子
        const float d = amax / ((1 << (QB - 1)) - 1);
        const float id = d ? 1.0/d : 0.0;

        // 将浮点数转换为定点数，并存储到数组中
        pd[i] = GGML_FP32_TO_GQ(d);

        // 对每个量化值进行处理
        for (int s = 0; s < nq; ++s) {
            // 将 pp 数组清零
            memset(pp, 0, sizeof(pp));

#if 0
            // 对每个量化值进行处理
            for (int l = 0; l < gq_t_bits; l++) {
                const   float v = src[i*QK + s*gq_t_bits + l];
                const uint8_t q = v*id + frand();

                // 将量化值转换为二进制表示，并存储到 pp 数组中
                for (int b = 0; b < QB; b++) {
                    pp[b] |= q & (1 << b) ? (1ULL << l) : 0;
                }
            }
                # 如果定义了 __ARM_NEON，则执行以下代码块
                {
                    # 创建一个大小为 2*4*QB 的无符号整数数组 ppt
                    uint32_t ppt[2*4*QB];

                    # 创建一个 id 的四倍长度的浮点数数组 idv
                    float32x4_t idv  = vdupq_n_f32(id);

                    # 断言 gq_t_bits 的值为 64
                    assert(gq_t_bits == 64);

                    # 创建大小为 QB 的四个元素的无符号整数数组 p0 和 p1，每个元素都是 0
                    uint32x4_t p0[QB] = { vdupq_n_u32(0) };
                    uint32x4_t p1[QB] = { vdupq_n_u32(0) };

                    # 循环遍历 gq_t_bits，每次增加 16
                    for (int l = 0; l < gq_t_bits; l += 16) {
                        # 从 src 中加载四个浮点数到 v0
                        float32x4_t v0 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 0);
                        float32x4_t v1 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 4);
                        float32x4_t v2 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 8);
                        float32x4_t v3 = vld1q_f32(src + i*QK + s*gq_t_bits + l + 12);

                        # 将 v0、v1、v2、v3 中的每个元素与 idv 中的对应元素相乘
                        v0 = vmulq_f32(v0, idv);
                        v1 = vmulq_f32(v1, idv);
                        v2 = vmulq_f32(v2, idv);
                        v3 = vmulq_f32(v3, idv);

                        # 如果条件为真
                        # 将 v0、v1、v2、v3 中的每个元素与 frand() 的返回值相加
                        v0[0] += frand(); v0[1] += frand(); v0[2] += frand(); v0[3] += frand();
                        v1[0] += frand(); v1[1] += frand(); v1[2] += frand(); v1[3] += frand();
                        v2[0] += frand(); v2[1] += frand(); v2[2] += frand(); v2[3] += frand();
                        v3[0] += frand(); v3[1] += frand(); v3[2] += frand(); v3[3] += frand();
// 将四个单精度浮点数向量转换为四个无符号整数向量
uint32x4_t q0 = vcvtq_u32_f32(v0);
uint32x4_t q1 = vcvtq_u32_f32(v1);
uint32x4_t q2 = vcvtq_u32_f32(v2);
uint32x4_t q3 = vcvtq_u32_f32(v3);

// 遍历 QB（未定义）次
for (int b = 0; b < QB; ++b) {
    // 创建一个包含 1 左移 b 位的无符号整数向量
    uint32x4_t m = vdupq_n_u32(1 << b);
    // 创建一个包含 -b 的四个元素的有符号整数向量
    int32x4_t r = vdupq_n_s32(-b);

    // 如果 l 小于 32
    if (l < 32) {
        // 将四个向量中的每个元素与 m 进行按位与运算，然后左移 r 位，再与 sh + l + 0 处的四个元素进行按位或运算
        p0[b] = vorrq_u32(p0[b], vshlq_u32(vshlq_u32(vandq_u32(q0, m), r), vld1q_s32(sh + l + 0)));
        p0[b] = vorrq_u32(p0[b], vshlq_u32(vshlq_u32(vandq_u32(q1, m), r), vld1q_s32(sh + l + 4)));
        p0[b] = vorrq_u32(p0[b], vshlq_u32(vshlq_u32(vandq_u32(q2, m), r), vld1q_s32(sh + l + 8)));
        p0[b] = vorrq_u32(p0[b], vshlq_u32(vshlq_u32(vandq_u32(q3, m), r), vld1q_s32(sh + l + 12)));
    } else {
        // 将四个向量中的每个元素与 m 进行按位与运算，然后左移 r 位，再与 sh + l - 32 处的四个元素进行按位或运算
        p1[b] = vorrq_u32(p1[b], vshlq_u32(vshlq_u32(vandq_u32(q0, m), r), vld1q_s32(sh + l - 32)));
        p1[b] = vorrq_u32(p1[b], vshlq_u32(vshlq_u32(vandq_u32(q1, m), r), vld1q_s32(sh + l - 28)));
        p1[b] = vorrq_u32(p1[b], vshlq_u32(vshlq_u32(vandq_u32(q2, m), r), vld1q_s32(sh + l - 24)));
        p1[b] = vorrq_u32(p1[b], vshlq_u32(vshlq_u32(vandq_u32(q3, m), r), vld1q_s32(sh + l - 20)));
    }
}
#if QB == 4
    // 如果 QB 等于 4，则执行以下代码块
    vst1q_u32((uint32_t *) ppt + 0,  p0[0]);
    // 将 p0[0] 中的值存储到 ppt 数组的前 4 个位置
    vst1q_u32((uint32_t *) ppt + 4,  p1[0]);
    // 将 p1[0] 中的值存储到 ppt 数组的接下来的 4 个位置
    // ... 依此类推，将 p0 和 p1 数组中的值存储到 ppt 数组中
    pp[0] = (ppt[0]  | ppt[1]  | ppt[2]  | ppt[3] ) | ((uint64_t) (ppt[4]  | ppt[5]  | ppt[6]  | ppt[7]) ) << 32;
    // 将 ppt 数组中的值按位或运算，并将结果存储到 pp[0] 中
    // ... 依此类推，将 ppt 数组中的值按位或运算，并将结果存储到 pp 数组中
#else
    // 如果 QB 不等于 4，则执行以下代码块
    for (int q = 0; q < QB; ++q) {
        // 遍历 p0 和 p1 数组，将值存储到 ppt 数组中
        pp[q] = (ppt[0] | ppt[1] | ppt[2] | ppt[3]) | ((uint64_t) (ppt[4] | ppt[5] | ppt[6] | ppt[7])) << 32;
        // 将 ppt 数组中的值按位或运算，并将结果存储到 pp 数组中
    }
#endif
// 将 pp 数组中的值复制到 pb 数组的指定位置
memcpy(pb + i*nq*QB + s*QB, pp, sizeof(pp));
// 重写 quantize_3 函数，使用 quantize_3_row 函数
void quantize_3(const float * restrict src, char * restrict dst, int n, int k) {
    // 断言 k 能被 QK 整除
    assert(k % QK == 0);
    // 遍历 n 次，对每个 src 数组中的元素进行 quantize_3_row 操作
    for (int j = 0; j < n; j++) {
        quantize_3_row(src + j*k, dst, k);
        // 更新 dst 指针，指向下一个 quantize_3_row 的位置
        dst = (char *) dst + quantize_3_row_size(k);
    }
}
// 计算向量点积的函数
void vec_dot_gq_3(const int n, float * restrict s, const void * restrict x, const void * restrict y) {
    // 初始化 sumf 为 0.0
    float sumf = 0.0f;
    // 计算每行的块数
    const int nb = quantize_3_blocks_per_row(n);
    // 计算每个块的量化数
    const int nq = quantize_3_quants_per_block();
    // 将 x 强制转换为 gq_scale_t 类型，并赋值给 pd0 指针
    const gq_scale_t * restrict pd0 = (const gq_scale_t *) x;
    # 声明并初始化指向常量 gq_scale_t 类型的指针 pd1，指向 y 的地址
    const gq_scale_t * restrict pd1 = (const gq_scale_t *) y;

    # 声明并初始化指向常量 gq_quant_t 类型的指针 pb0，指向 pd0 + nb 的地址
    const gq_quant_t * restrict pb0 = (const gq_quant_t *) (pd0 + nb);
    # 声明并初始化指向常量 gq_quant_t 类型的指针 pb1，指向 pd1 + nb 的地址
    const gq_quant_t * restrict pb1 = (const gq_quant_t *) (pd1 + nb);
#if 1
    # 对于每个 i，计算 isum
    for (int i = 0; i < nb; i++) {
        int isum = 0;

#if QB == 4
        # 对于每个 s，计算 isum
        for (int s = 0; s < nq; ++s) {
            # 计算 m0 和 m1
            const gq_quant_t * restrict m0 = pb0 + i*nq*QB + s*QB;
            const gq_quant_t * restrict m1 = pb1 + i*nq*QB + s*QB;

            # 计算 isum
            isum += (1 << 0)*(__builtin_popcountll(m0[0] & m1[0]));
            isum += (1 << 1)*(__builtin_popcountll(m0[0] & m1[1]) + __builtin_popcountll(m0[1] & m1[0]));
            isum += (1 << 2)*(__builtin_popcountll(m0[0] & m1[2]) + __builtin_popcountll(m0[1] & m1[1]) + __builtin_popcountll(m0[2] & m1[0]));
            isum += (1 << 3)*(__builtin_popcountll(m0[0] & m1[3]) + __builtin_popcountll(m0[1] & m1[2]) + __builtin_popcountll(m0[2] & m1[1]) + __builtin_popcountll(m0[3] & m1[0]));
            isum += (1 << 4)*(__builtin_popcountll(m0[1] & m1[3]) + __builtin_popcountll(m0[2] & m1[2]) + __builtin_popcountll(m0[3] & m1[1]));
            isum += (1 << 5)*(__builtin_popcountll(m0[2] & m1[3]) + __builtin_popcountll(m0[3] & m1[2]));
            isum += (1 << 6)*(__builtin_popcountll(m0[3] & m1[3]));
        }
#else
        # 对于每个 s 和 q0，计算 isum
        for (int s = 0; s < nq; ++s) {
            for (int q0 = 0; q0 < QB; q0++) {
                const gq_quant_t mm0 = pb0[i*nq*QB + s*QB + q0];
                # 对于每个 q1，计算 isum
                for (int q1 = 0; q1 < QB; q1++) {
                    const gq_quant_t mm1 = pb1[i*nq*QB + s*QB + q1];
                    isum += (1 << (q0 + q1))*(__builtin_popcountll(mm0 & mm1));
                }
            }
        }
#endif

        # 计算 d0 和 d1
        const float d0 = GGML_GQ_TO_FP32(pd0[i]);
        const float d1 = GGML_GQ_TO_FP32(pd1[i]);

        # 计算 sumf
        sumf += d0*d1*isum;
    }
#else
#ifdef __ARM_NEON
    # 对于每个 i，计算 isum
    for (int i = 0; i < nb; i += 4) {
        int isum[4] = {0, 0, 0, 0};

        # 对于每个 k，计算 isum
        for (int k = 0; k < 4; ++k) {
            # 对于每个 s，计算 isum
            for (int s = 0; s < nq; ++s) {
                # 计算 m0 和 m1
                const gq_quant_t * restrict m0 = pb0 + (i+k)*nq*QB + s*QB;
                const gq_quant_t * restrict m1 = pb1 + (i+k)*nq*QB + s*QB;

#if QB == 4
#define bpcnt(x) __builtin_popcountll(x)
                // 定义一个宏函数，用于计算64位整数中1的个数
                // 计算每个位置上 m0 和 m1 的按位与结果中1的个数，并根据位置加权求和
# 定义并加载第一行矩阵的8个元素到uint8x8_t类型的变量m00
const uint8x8_t m00 = vld1_u8((const uint8_t *) (m0 + 0));
# 定义并加载第二行矩阵的8个元素到uint8x8_t类型的变量m01
const uint8x8_t m01 = vld1_u8((const uint8_t *) (m0 + 1));
# 定义并加载第三行矩阵的8个元素到uint8x8_t类型的变量m02
const uint8x8_t m02 = vld1_u8((const uint8_t *) (m0 + 2));
# 定义并加载第四行矩阵的8个元素到uint8x8_t类型的变量m03
const uint8x8_t m03 = vld1_u8((const uint8_t *) (m0 + 3));

# 定义并加载第一行矩阵的8个元素到uint8x8_t类型的变量m10
const uint8x8_t m10 = vld1_u8((const uint8_t *) (m1 + 0));
# 定义并加载第二行矩阵的8个元素到uint8x8_t类型的变量m11
const uint8x8_t m11 = vld1_u8((const uint8_t *) (m1 + 1));
# 定义并加载第三行矩阵的8个元素到uint8x8_t类型的变量m12
const uint8x8_t m12 = vld1_u8((const uint8_t *) (m1 + 2));
# 定义并加载第四行矩阵的8个元素到uint8x8_t类型的变量m13
const uint8x8_t m13 = vld1_u8((const uint8_t *) (m1 + 3));

# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m00m10 = vand_u8(m00, m10);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m00m11 = vand_u8(m00, m11);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m01m10 = vand_u8(m01, m10);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m00m12 = vand_u8(m00, m12);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m01m11 = vand_u8(m01, m11);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m02m10 = vand_u8(m02, m10);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m00m13 = vand_u8(m00, m13);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m01m12 = vand_u8(m01, m12);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m02m11 = vand_u8(m02, m11);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m03m10 = vand_u8(m03, m10);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m01m13 = vand_u8(m01, m13);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m02m12 = vand_u8(m02, m12);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m03m11 = vand_u8(m03, m11);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m02m13 = vand_u8(m02, m13);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m03m12 = vand_u8(m03, m12);
# 对应位置元素相与，得到新的uint8x8_t类型的变量
const uint8x8_t m03m13 = vand_u8(m03, m13);
// 宏定义，计算输入向量中每个元素的比特位为1的个数
#define bpcnt(x) vaddv_u8(vcnt_u8(x))
// 对每个元素进行计算，得到最终的累加和
isum[k] += (1ULL << 0)*(bpcnt(m00m10)) +
           (1ULL << 1)*(bpcnt(m00m11) + bpcnt(m01m10)) +
           (1ULL << 2)*(bpcnt(m00m12) + bpcnt(m01m11) + bpcnt(m02m10)) +
           (1ULL << 3)*(bpcnt(m00m13) + bpcnt(m01m12) + bpcnt(m02m11) + bpcnt(m03m10)) +
           (1ULL << 4)*(bpcnt(m01m13) + bpcnt(m02m12) + bpcnt(m03m11)) +
           (1ULL << 5)*(bpcnt(m02m13) + bpcnt(m03m12)) +
           (1ULL << 6)*(bpcnt(m03m13));
// 取消宏定义
#undef bpcnt
#else
// 使用循环计算每个元素的累加和
for (int q0 = 0; q0 < QB; q0++) {
    const gq_quant_t mm0 = m0[q0];
    for (int q1 = 0; q1 < QB; q1++) {
        const gq_quant_t mm1 = m1[q1];
        isum[k] += (1ULL << (q0 + q1))*(__builtin_popcountll(mm0 & mm1));
    }
}
#endif
// 将累加和转换为向量形式
int32x4_t isumv = vld1q_s32(isum);
// 将输入向量转换为向量形式
float32x4_t d0v = vld1q_f32(pd0 + i);
float32x4_t d1v = vld1q_f32(pd1 + i);
// 计算两个向量的点积
float32x4_t sumfv = vmulq_f32(d0v, d1v);
// 将点积结果与累加和相乘
sumfv = vmulq_f32(sumfv, vcvtq_f32_s32(isumv));
// 将向量中的元素相加得到最终结果
sumf += vaddvq_f32(sumfv);
// 结束循环
}
#else
// 报错，未实现该部分功能
#error "not implemented"
#endif

#endif
// 将最终结果赋值给输出指针
*s = sumf;
}

// 使用vec_dot_gq_3计算两行的点积
void mul_mat_gq_3(
const void * src0,
const void * src1, // 转置
     float * dst,
int m, int n, int k) {
assert(k % QK == 0);

const int nb = quantize_3_blocks_per_row(k);
const int nq = quantize_3_quants_per_block();
    # 遍历行
    for (int ir0 = 0; ir0 < m; ir0++) {
        # 遍历列
        for (int ir1 = 0; ir1 < n; ir1++) {
            # 调用函数，计算向量点积并存储到目标数组中
            vec_dot_gq_3(k, dst + ir1, src0, src1);
            # 更新 src1 指针，指向下一行的数据
            src1 = (const char *) src1 + quantize_3_row_size(k);
        }
        # 更新 src0 指针，指向下一行的数据
        src0 = (const char *) src0 +   quantize_3_row_size(k);
        # 更新 src1 指针，指向上一行的最后一个数据
        src1 = (const char *) src1 - n*quantize_3_row_size(k);

        # 更新 dst 指针，指向下一行的起始位置
        dst = (float *) dst + n;
    }
# 计算每行有多少个 4 个元素的块
static inline int quantize_4_blocks_per_row(int k) {
    return k/QK;
}

# 计算每行的大小
static inline int quantize_4_row_size(int k) {
    # 计算每行有多少个 4 个元素的块
    const int nb = quantize_4_blocks_per_row(k);

    # 返回每行的大小
    return nb*(2*sizeof(gq_scale_t) + QK/2);
}

# 对每行进行 4 位量化
void quantize_4_row(const float * restrict src, void * restrict dst, int k) {
    # 确保 k 是 QK 的倍数
    assert(k % QK == 0);
    # 确保 QB 等于 4
    assert(QB == 4);

    # 计算每行有多少个 4 个元素的块
    const int nb = quantize_4_blocks_per_row(k);

    # 为每个块的最小值、最大值和数据分配内存
    gq_scale_t * restrict pm = (gq_scale_t *) (dst);
    gq_scale_t * restrict pd = (gq_scale_t *) (pm + nb);
    uint8_t    * restrict pb = (uint8_t *)    (pd + nb);

    # 为每个块的临时数据分配内存
    uint8_t pp[QK/2];

    # 遍历每个块
    for (int i = 0; i < nb; i++) {
        # 将临时数据清零
        memset(pp, 0, sizeof(pp));

        # 初始化最小值和最大值
        float min = FLT_MAX;
        float max = -FLT_MAX;

        # 这里是一段注释掉的代码，不会被执行
#if 0
                v[0] += frand(); v[1] += frand(); v[2] += frand(); v[3] += frand();
                v[4] += frand(); v[5] += frand(); v[6] += frand(); v[7] += frand();
#ifdef

                // 将浮点数转换为无符号8位整数
                __m256i vi = _mm256_cvtps_epi32(v);

                // 提取 vi 的每个元素
                uint32_t vi_0 = _mm256_extract_epi32(vi, 0);
                uint32_t vi_1 = _mm256_extract_epi32(vi, 1);
                uint32_t vi_2 = _mm256_extract_epi32(vi, 2);
                uint32_t vi_3 = _mm256_extract_epi32(vi, 3);

                uint32_t vi_4 = _mm256_extract_epi32(vi, 4);
                uint32_t vi_5 = _mm256_extract_epi32(vi, 5);
                uint32_t vi_6 = _mm256_extract_epi32(vi, 6);
                uint32_t vi_7 = _mm256_extract_epi32(vi, 7);

                // 将整数转换为4位，2个连续的整数打包成1个字节
                pp[4*l + 0] = vi_0 | (vi_1 << 4);
                pp[4*l + 1] = vi_2 | (vi_3 << 4);
                pp[4*l + 2] = vi_4 | (vi_5 << 4);
                pp[4*l + 3] = vi_6 | (vi_7 << 4);

                //printf("vi: %7d %7d %7d %7d %7d %7d %7d %7d\n", vi_0, vi_1, vi_2, vi_3, vi_4, vi_5, vi_6, vi_7);
                //printf("v : %7.3f %7.3f %7.3f %7.3f %7.3f %7.3f %7.3f %7.3f\n", v[0], v[1], v[2], v[3], v[4], v[5], v[6], v[7]);
            }

            // 将 pp 复制到 pb
            memcpy(pb + i*QK/2, pp, sizeof(pp));
#elif defined(__ARM_NEON) && 0
        {
            // TODO
        }
#else
        {
            for (int l = 0; l < QK; l++) {
                const float v = src[i*QK + l];
                if (v < min) min = v;
                if (v > max) max = v;
            }

            const float d = (max - min) / ((1 << QB) - 1);
            const float id = d ? 1.0/d : 0.0;

            pm[i] = GGML_FP32_TO_GQ(min);
            pd[i] = GGML_FP32_TO_GQ(d);

            for (int l = 0; l < QK; l++) {
                const float v = (src[i*QK + l] - min) * id;
                const uint8_t vi = (uint8_t) (v + frand());
                pp[l/2] |= (vi & 0xf) << (4*(l & 1));
            }

            // 将 pp 复制到 pb
            memcpy(pb + i*QK/2, pp, sizeof(pp));
        }
#endif
        //printf("min %f max %f\n", min, max);
    }
}
// 重新实现 quantize_4，使用 quantize_4_row
void quantize_4(const float * restrict src, char * restrict dst, int n, int k) {
    // 确保 k 是 QK 的倍数
    assert(k % QK == 0);

    // 遍历每一列
    for (int j = 0; j < n; j++) {
        // 调用 quantize_4_row 处理当前列的数据
        quantize_4_row(src + j*k, dst, k);
        // 更新 dst 指针，指向下一列的位置
        dst = (char *) dst + quantize_4_row_size(k);
    }
}

// 计算向量点积的通用量化版本
void vec_dot_gq_4(const int n, float * restrict s, const void * restrict x, const void * restrict y) {
    // 计算每行的块数
    const int nb = quantize_4_blocks_per_row(n);

    // 强制转换输入指针类型
    const gq_scale_t * restrict pm0 = (const gq_scale_t *) x;
    const gq_scale_t * restrict pm1 = (const gq_scale_t *) y;

    const gq_scale_t * restrict pd0 = pm0 + nb;
    const gq_scale_t * restrict pd1 = pm1 + nb;

    const uint8_t * restrict pb0 = (const uint8_t *) (pd0 + nb);
    const uint8_t * restrict pb1 = (const uint8_t *) (pd1 + nb);

    float sumf = 0.0;

#if 0
    // 标量计算
    for (int i = 0; i < nb; i++) {
        // 从量化值转换为浮点数
        const float m0 = GGML_GQ_TO_FP32(pm0[i]);
        const float d0 = GGML_GQ_TO_FP32(pd0[i]);

        const float m1 = GGML_GQ_TO_FP32(pm1[i]);
        const float d1 = GGML_GQ_TO_FP32(pd1[i]);

        const uint8_t * restrict p0 = pb0 + i*QK/2;
        const uint8_t * restrict p1 = pb1 + i*QK/2;

        for (int j = 0; j < QK/2; j++) {
            const uint8_t v0 = p0[j];
            const uint8_t v1 = p1[j];

            const float f0 = d0*(v0 & 0xf) + m0;
            const float f1 = d0*(v0 >> 4)  + m0;

            const float f2 = d1*(v1 & 0xf) + m1;
            const float f3 = d1*(v1 >> 4)  + m1;

            sumf += f0*f2 + f1*f3;
        }
    }
#else
#if defined(__AVX2__)
#if QK == 64 && 0
    // 使用 AVX2 指令集进行向量化计算
    __m256 sumv0 = _mm256_setzero_ps();
    __m256 sumv1 = _mm256_setzero_ps();

    // 其他代码被省略
    // 初始化四个浮点数变量，用于存储计算结果
    float sum00 = 0.0f;
    float sum01 = 0.0f;
    float sum10 = 0.0f;
    float sum11 = 0.0f;

    // 创建一个包含 16 个 0xf 的 256 位整数常量
    const __m256i m4b = _mm256_set1_epi8(0xf);

    // 循环处理每个元素
    for (int i = 0; i < nb; i++) {
        // 将 pm0[i] 和 pd0[i] 转换为单精度浮点数
        const float m0 = GGML_GQ_TO_FP32(pm0[i]);
        const float d0 = GGML_GQ_TO_FP32(pd0[i]);

        // 将 pm1[i] 和 pd1[i] 转换为单精度浮点数
        const float m1 = GGML_GQ_TO_FP32(pm1[i]);
        const float d1 = GGML_GQ_TO_FP32(pd1[i]);

        // 计算指向 pb0 和 pb1 的指针
        const uint8_t * restrict p0 = pb0 + i*QK/2;
        const uint8_t * restrict p1 = pb1 + i*QK/2;

        // 加载 64 个字节的数据到 v0 和 v1 中
        const __m256i v0 = _mm256_loadu_si256((__m256i *) p0);
        const __m256i v1 = _mm256_loadu_si256((__m256i *) p1);

        // 对 v0 和 v1 中的数据进行按位与运算
        const __m256i v0l = _mm256_and_si256(v0, m4b);
        const __m256i v1l = _mm256_and_si256(v1, m4b);

        // 对 v0 和 v1 中的数据进行右移 4 位并进行按位与运算
        const __m256i v0h = _mm256_and_si256(_mm256_srli_epi16(v0, 4), m4b);
        const __m256i v1h = _mm256_and_si256(_mm256_srli_epi16(v1, 4), m4b);

        // 对 v0l 和 v1l 中的数据进行乘法并相加
        const __m256i pl = _mm256_maddubs_epi16(v0l, v1l);
        const __m256i ph = _mm256_maddubs_epi16(v0h, v1h);

        // 对 ph 和 pl 中的数据进行相加
        const __m256i p16 = _mm256_add_epi16(ph, pl);

        // 对 p16 中的数据进行乘法并相加
        const __m256i p = _mm256_madd_epi16(_mm256_set1_epi16(1), p16);

        // 更新 sum00、sum01、sum10 和 sum11 的值
        sum00 += m0*m1;
        sum01 += m1*d0*(_mm256_hadd_epi8_gg(_mm256_add_epi8(v0l, v0h)));
        sum10 += m0*d1*(_mm256_hadd_epi8_gg(_mm256_add_epi8(v1l, v1h)));
        sum11 += d0*d1*(_mm256_hadd_epi32_gg(p));
    }

    // 计算最终结果并赋值给 sumf
    sumf = 64.0*sum00 + sum01 + sum10 + sum11;
#elif QK == 64 && 1 // this is the best when using min + d
    # 如果 QK 等于 64 并且为真，则执行以下代码，这是在使用 min + d 时的最佳选择

    float sum00 = 0.0f;  # 初始化 sum00 为 0.0

    __m256 sum01 = _mm256_setzero_ps();  # 使用 AVX 指令集初始化 sum01 为全 0
    __m256 sum10 = _mm256_setzero_ps();  # 使用 AVX 指令集初始化 sum10 为全 0
    __m256 sum11 = _mm256_setzero_ps();  # 使用 AVX 指令集初始化 sum11 为全 0

    }

    sumf = 64.0*sum00 + _mm256_hadd_ps_gg(sum01) + _mm256_hadd_ps_gg(sum10) + _mm256_hadd_ps_gg(sum11);  # 计算 sumf 的值
#endif
#elif defined (__ARM_NEON)
    # 如果定义了 __ARM_NEON，则执行以下代码

    float sum00 = 0.0f;  # 初始化 sum00 为 0.0
    float sum01 = 0.0f;  # 初始化 sum01 为 0.0
    float sum10 = 0.0f;  # 初始化 sum10 为 0.0
    float sum11 = 0.0f;  # 初始化 sum11 为 0.0

    }

    sumf = 64.0*sum00 + sum01 + sum10 + sum11;  # 计算 sumf 的值
#endif
#endif

    *s = sumf;  # 将 sumf 的值赋给指针 s
}

// use vec_dot_gq_4 to compute the dot product of two rows
void mul_mat_gq_4(
    const void * src0,
    const void * src1, // transposed
         float * dst,
    int m, int n, int k) {
    assert(k % QK == 0);  # 断言 k 能被 QK 整除

    const int nb = quantize_4_blocks_per_row(k);  # 计算每行的块数

    for (int ir0 = 0; ir0 < m; ir0++) {  # 循环遍历 m 行
        for (int ir1 = 0; ir1 < n; ir1++) {  # 循环遍历 n 行
            vec_dot_gq_4(k, dst + ir1, src0, src1);  # 使用 vec_dot_gq_4 计算两行的点积
            src1 = (const char *) src1 + quantize_4_row_size(k);  # 更新 src1 的位置
        }
        src0 = (const char *) src0 +   quantize_4_row_size(k);  # 更新 src0 的位置
        src1 = (const char *) src1 - n*quantize_4_row_size(k);  # 更新 src1 的位置
        dst = (float *) dst + n;  # 更新 dst 的位置
    }
}

//
// method 5
// 4-bit quantization (without min, only delta)
//

static inline int quantize_5_blocks_per_row(int k) {  # 计算每行的块数
    return k/QK;
}

static inline int quantize_5_row_size(int k) {  # 计算每行的大小
    const int nb = quantize_5_blocks_per_row(k);  # 计算每行的块数

    return nb*(sizeof(gq_scale_t) + QK/2);  # 返回每行的大小
}

void quantize_5_row(const float * restrict src, void * restrict dst, int k) {  # 对每行进行 4 位量化
    assert(k % QK == 0);  # 断言 k 能被 QK 整除
    assert(QB == 4);  # 断言 QB 等于 4

    const int nb = quantize_5_blocks_per_row(k);  # 计算每行的块数

    gq_scale_t * restrict pd = (gq_scale_t *) (dst);  # 初始化指向 dst 的 gq_scale_t 类型指针
    uint8_t    * restrict pb = (uint8_t *)    (pd + nb);  # 初始化指向 dst 后面的 uint8_t 类型指针

    uint8_t pp[QK/2];  # 初始化长度为 QK/2 的 uint8_t 数组

    for (int i = 0; i < nb; i++) {  # 循环遍历每个块
        memset(pp, 0, sizeof(pp));  # 将 pp 数组清零

        float amax = 0.0f; // absolute max
#if defined(__AVX2__)
        {
            // 如果 AVX2 被定义
            assert(QK == 64);
            // 确保 QK 等于 64
            enum { QK8 = QK/8 };
            // 定义 QK8 为 QK 的八分之一

            __m256 srcv [QK8];
            // 定义一个包含 QK8 个元素的 __m256 类型数组 srcv
            __m256 asrcv[QK8];
            // 定义一个包含 QK8 个元素的 __m256 类型数组 asrcv
            __m256 amaxv[QK8];
            // 定义一个包含 QK8 个元素的 __m256 类型数组 amaxv

            for (int l = 0; l < QK8; l++) {
                // 循环遍历 QK8 次
                srcv[l]  = _mm256_loadu_ps(src + i*QK + 8*l);
                // 将 src 中的数据加载到 srcv[l] 中
            }

            for (int l = 0; l < QK8; l++) {
                // 循环遍历 QK8 次
                asrcv[l] = _mm256_and_ps(srcv[l], _mm256_castsi256_ps(_mm256_set1_epi32(0x7fffffff)));
                // 将 srcv[l] 和 0x7fffffff 进行按位与操作，结果存入 asrcv[l] 中
            }

            for (int l = 0; l < QK8/2; l++) {
                // 循环遍历 QK8/2 次
                amaxv[2*l] = _mm256_max_ps(asrcv[2*l], asrcv[2*l+1]);
                // 将 asrcv[2*l] 和 asrcv[2*l+1] 中的数据进行比较，将较大值存入 amaxv[2*l] 中
            }

            for (int l = 0; l < QK8/4; l++) {
                // 循环遍历 QK8/4 次
                amaxv[4*l] = _mm256_max_ps(amaxv[4*l], amaxv[4*l+2]);
                // 将 amaxv[4*l] 和 amaxv[4*l+2] 中的数据进行比较，将较大值存入 amaxv[4*l] 中
            }

            for (int l = 0; l < QK8/8; l++) {
                // 循环遍历 QK8/8 次
                amaxv[8*l] = _mm256_max_ps(amaxv[8*l], amaxv[8*l+4]);
                // 将 amaxv[8*l] 和 amaxv[8*l+4] 中的数据进行比较，将较大值存入 amaxv[8*l] 中
            }

            //amax = MAX(amaxv[0][0], MAX(amaxv[0][1], MAX(amaxv[0][2], MAX(amaxv[0][3], MAX(amaxv[0][4], MAX(amaxv[0][5], MAX(amaxv[0][6], amaxv[0][7])))))));

            const __m256 amaxv0_0 = _mm256_permute2f128_ps(amaxv[0], amaxv[0], 3);
            // 将 amaxv[0] 中的数据进行特定的排列操作，结果存入 amaxv0_0 中
            const __m256 amaxv0_1 = _mm256_max_ps(amaxv[0], amaxv0_0);
            // 将 amaxv[0] 和 amaxv0_0 中的数据进行比较，将较大值存入 amaxv0_1 中
            const __m256 amaxv0_2 = _mm256_permute_ps(amaxv0_1, 0x4e);
            // 将 amaxv0_1 中的数据进行特定的排列操作，结果存入 amaxv0_2 中
            const __m256 amaxv0_3 = _mm256_max_ps(amaxv0_1, amaxv0_2);
            // 将 amaxv0_1 和 amaxv0_2 中的数据进行比较，将较大值存入 amaxv0_3 中
            const __m256 amaxv0_4 = _mm256_permute_ps(amaxv0_3, 0xb1);
            // 将 amaxv0_3 中的数据进行特定的排列操作，结果存入 amaxv0_4 中
            const __m256 amaxv0_5 = _mm256_max_ps(amaxv0_3, amaxv0_4);
            // 将 amaxv0_3 和 amaxv0_4 中的数据进行比较，将较大值存入 amaxv0_5 中

            amax = _mm256_cvtss_f32(amaxv0_5);
            // 将 amaxv0_5 中的数据转换为 float 类型，存入 amax 中

            //printf("amax = %f\n", amax);

            const float d = amax / ((1 << (QB - 1)) - 1);
            // 计算 d 的值
            const float id = d ? 1.0/d : 0.0;
            // 如果 d 不为 0，则计算 id 的值为 1.0/d，否则为 0.0

            pd[i] = GGML_FP32_TO_GQ(d);
            // 将 d 转换为 GGML_FP32_TO_GQ 类型后存入 pd[i] 中

            const __m256 idv = _mm256_set1_ps(id);
            // 将 id 转换为 __m256 类型后存入 idv 中

            for (int l = 0; l < QK/8; l++) {
                // 循环遍历 QK/8 次
                __m256 v = _mm256_mul_ps(srcv[l], idv);
                // 将 srcv[l] 和 idv 中的数据进行乘法操作，结果存入 v 中
#if 0
                // 如果条件为假，则执行以下代码块
                v[0] += frand(); v[1] += frand(); v[2] += frand(); v[3] += frand();
                v[4] += frand(); v[5] += frand(); v[6] += frand(); v[7] += frand();
#endif

                // 将浮点数转换为整型
                __m256i vi = _mm256_cvtps_epi32(v);
                // 将整型向量每个元素加上常数8
                vi = _mm256_add_epi32(vi, _mm256_set1_epi32(8));

                // 提取整型向量中的每个元素
                int32_t vi_0 = _mm256_extract_epi32(vi, 0);
                int32_t vi_1 = _mm256_extract_epi32(vi, 1);
                int32_t vi_2 = _mm256_extract_epi32(vi, 2);
                int32_t vi_3 = _mm256_extract_epi32(vi, 3);

                int32_t vi_4 = _mm256_extract_epi32(vi, 4);
                int32_t vi_5 = _mm256_extract_epi32(vi, 5);
                int32_t vi_6 = _mm256_extract_epi32(vi, 6);
                int32_t vi_7 = _mm256_extract_epi32(vi, 7);

                // 将整型转换为4位，2个连续的整数打包成1个字节
                pp[4*l + 0] = vi_0 | (vi_1 << 4);
                pp[4*l + 1] = vi_2 | (vi_3 << 4);
                pp[4*l + 2] = vi_4 | (vi_5 << 4);
                pp[4*l + 3] = vi_6 | (vi_7 << 4);

                // 断言整型值在指定范围内
                assert(vi_0 >= 0 && vi_0 < 16);
                assert(vi_1 >= 0 && vi_1 < 16);
                assert(vi_2 >= 0 && vi_2 < 16);
                assert(vi_3 >= 0 && vi_3 < 16);

                assert(vi_4 >= 0 && vi_4 < 16);
                assert(vi_5 >= 0 && vi_5 < 16);
                assert(vi_6 >= 0 && vi_6 < 16);
                assert(vi_7 >= 0 && vi_7 < 16);
            }

            // 将 pp 数组的内容复制到 pb 数组的指定位置
            memcpy(pb + i*QK/2, pp, sizeof(pp));
        }
#elif defined(__ARM_NEON) && 0
        {
            // 如果条件为真，并且 ARM NEON 指令集被定义，则执行以下代码块
            // TODO
        }
#else
        {
            // 对每个元素进行遍历，找出绝对值最大的值
            for (int l = 0; l < QK; l++) {
                const float v = src[i*QK + l];
                amax = MAX(amax, fabsf(v));
            }

            // 计算最大值的绝对值除以量化因子的值
            const float d = amax / ((1 << (QB - 1)) - 1);
            // 如果 d 不为 0，则计算其倒数，否则为 0
            const float id = d ? 1.0/d : 0.0;

            // 将 d 转换为 GQ 类型，并存入 pd 数组
            pd[i] = GGML_FP32_TO_GQ(d);

            // 对每个元素进行遍历，将其乘以 id，并转换为 int8_t 类型
            for (int l = 0; l < QK; l++) {
                const float v = src[i*QK + l]*id;
                const int8_t vi = ((int8_t) (round(v))) + 8;
                // 确保 vi 大于等于 0 且小于 16
                assert(vi >= 0 && vi < 16);
                // 将 vi 的低 4 位存入 pp 数组
                pp[l/2] |= (vi & 0xf) << (4*(l & 1));
            }

            // 将 pp 数组的内容复制到 pb 数组中
            memcpy(pb + i*QK/2, pp, sizeof(pp));
        }
#endif
        //printf("min %f max %f\n", min, max);
    }
}

// 重新实现 quantize_5，使用 quantize_5_row
void quantize_5(const float * restrict src, char * restrict dst, int n, int k) {
    // 确保 k 是 QK 的整数倍
    assert(k % QK == 0);

    // 对每一行进行 quantize_5_row 操作
    for (int j = 0; j < n; j++) {
        quantize_5_row(src + j*k, dst, k);
        dst = (char *) dst + quantize_5_row_size(k);
    }
}

void vec_dot_gq_5(const int n, float * restrict s, const void * restrict x, const void * restrict y) {
    // 计算每行的块数
    const int nb = quantize_5_blocks_per_row(n);

    // 将 x 和 y 转换为 gq_scale_t 类型
    const gq_scale_t * restrict pd0 = (const gq_scale_t *) x;
    const gq_scale_t * restrict pd1 = (const gq_scale_t *) y;

    // 计算 pb0 和 pb1 的指针位置
    const uint8_t * restrict pb0 = (const uint8_t *) (pd0 + nb);
    const uint8_t * restrict pb1 = (const uint8_t *) (pd1 + nb);

    // 初始化 sumf 为 0
    float sumf = 0.0;

#if 0
    // scalar
    # 遍历 nb 次，i 从 0 到 nb-1
    for (int i = 0; i < nb; i++) {
        # 将 pd0[i] 转换为 float 类型赋值给 d0
        const float d0 = GGML_GQ_TO_FP32(pd0[i]);
        # 将 pd1[i] 转换为 float 类型赋值给 d1

        const float d1 = GGML_GQ_TO_FP32(pd1[i]);

        # 计算指向 pb0 的指针偏移量为 i*QK/2，赋值给 p0
        const uint8_t * restrict p0 = pb0 + i*QK/2;
        # 计算指向 pb1 的指针偏移量为 i*QK/2，赋值给 p1
        const uint8_t * restrict p1 = pb1 + i*QK/2;

        # 遍历 QK/2 次，j 从 0 到 QK/2-1
        for (int j = 0; j < QK/2; j++) {
            # 取出 p0[j] 赋值给 v0
            const uint8_t v0 = p0[j];
            # 取出 p1[j] 赋值给 v1

            const uint8_t v1 = p1[j];

            # 计算 f0
            const float f0 = d0*((int8_t) (v0 & 0xf) - 8);
            # 计算 f1
            const float f1 = d0*((int8_t) (v0 >> 4)  - 8);

            # 计算 f2
            const float f2 = d1*((int8_t) (v1 & 0xf) - 8);
            # 计算 f3
            const float f3 = d1*((int8_t) (v1 >> 4)  - 8);

            # 计算 sumf
            sumf += f0*f2 + f1*f3;
        }
    }
#else
#if defined(__AVX2__)
#if QK == 64 && 1
    // 初始化一个全为0的256位浮点数寄存器
    __m256 sum11 = _mm256_setzero_ps();

    // 循环处理每个数据块
    for (int i = 0; i < nb; i++) {
        // 将pd0[i]和pd1[i]转换为float类型
        const float d0 = GGML_GQ_TO_FP32(pd0[i]);
        const float d1 = GGML_GQ_TO_FP32(pd1[i]);

        // 计算p0和p1的指针位置
        const uint8_t * restrict p0 = pb0 + i*QK/2;
        const uint8_t * restrict p1 = pb1 + i*QK/2;

        // 将d0和d1转换为256位浮点数寄存器
        const __m256 d0v = _mm256_set1_ps(d0);
        const __m256 d1v = _mm256_set1_ps(d1);

        // 计算d0v和d1v的乘积
        const __m256 d0d1v = _mm256_mul_ps(d0v, d1v);

        // 创建一个全为0xf的256位整数寄存器
        const __m256i m4b = _mm256_set1_epi8(0xf);

        // 从p0和p1中加载64个8位整数到256位整数寄存器
        const __m256i v0 = _mm256_loadu_si256((__m256i *) p0);
        const __m256i v1 = _mm256_loadu_si256((__m256i *) p1);

        // 对v0和v1进行位与运算
        __m256i v0l = _mm256_and_si256(v0, m4b);
        __m256i v1l = _mm256_and_si256(v1, m4b);

        // 对v0和v1进行逻辑右移4位，并进行位与运算
        __m256i v0h = _mm256_and_si256(_mm256_srli_epi16(v0, 4), m4b);
        __m256i v1h = _mm256_and_si256(_mm256_srli_epi16(v1, 4), m4b);

        // 对v0l、v0h、v1l、v1h分别减去8
        v0l = _mm256_sub_epi8(v0l, _mm256_set1_epi8(8));
        v0h = _mm256_sub_epi8(v0h, _mm256_set1_epi8(8));
        v1l = _mm256_sub_epi8(v1l, _mm256_set1_epi8(8));
        v1h = _mm256_sub_epi8(v1h, _mm256_set1_epi8(8));

        // 对v0l、v0h取绝对值
        const __m256i v0la = _mm256_sign_epi8(v0l, v0l);
        const __m256i v0ha = _mm256_sign_epi8(v0h, v0h);

        // 对v1l、v1h取符号
        const __m256i v1ls = _mm256_sign_epi8(v1l, v0l);
        const __m256i v1hs = _mm256_sign_epi8(v1h, v0h);

        // 计算pl和ph
        const __m256i pl = _mm256_maddubs_epi16(v0la, v1ls);
        const __m256i ph = _mm256_maddubs_epi16(v0ha, v1hs);

        // 计算p16和p
        const __m256i p16 = _mm256_add_epi16(ph, pl);
        const __m256i p = _mm256_madd_epi16(_mm256_set1_epi16(1), p16);

        // 计算sum11
        sum11 = _mm256_fmadd_ps(d0d1v, _mm256_cvtepi32_ps(p), sum11);
    }

    // 对sum11进行水平加法
    sumf = _mm256_hadd_ps_gg(sum11);
#endif
#elif defined (__ARM_NEON)
    // 初始化一个float类型的变量sum11
    float sum11 = 0.0f;

    // 创建两个全为0的float32x4_t类型变量sum_0和sum_1
    //float32x4_t sum_0 = vdupq_n_f32(0.0f);
    //float32x4_t sum_1 = vdupq_n_f32(0.0f);
    // 创建一个包含8个float16类型0.0f的向量
    //float16x8_t sum_0 = vdupq_n_f16(0.0f);
    // 创建一个包含8个float16类型0.0f的向量
    //float16x8_t sum_1 = vdupq_n_f16(0.0f);

    }

    // 将sum11的值赋给sumf
    sumf = sum11;
    // 将sum_0和sum_1中的所有元素相加得到一个float32类型的值
    //sumf = vaddvq_f32(sum_0) + vaddvq_f32(sum_1);
    // 将sum_0中的所有元素相加得到一个float32类型的值
    //sumf = sum_0[0] + sum_0[1] + sum_0[2] + sum_0[3] + sum_0[4] + sum_0[5] + sum_0[6] + sum_0[7];
    // 将sum_0和sum_1中的对应元素相加
    //sum_0 = vaddq_f16(sum_0, sum_1);
    // 将sum_0中的所有元素相加得到一个float32类型的值
    //sumf = sum_0[0] + sum_0[1] + sum_0[2] + sum_0[3] + sum_0[4] + sum_0[5] + sum_0[6] + sum_0[7];
#endif
#endif

    *s = sumf;
}

// use vec_dot_gq_5 to compute the dot product of two rows
void mul_mat_gq_5(
    const void * src0,
    const void * src1, // transposed
         float * dst,
    int m, int n, int k) {
    assert(k % QK == 0);

    const int nb = quantize_5_blocks_per_row(k);

    for (int ir0 = 0; ir0 < m; ir0++) {
        for (int ir1 = 0; ir1 < n; ir1++) {
            vec_dot_gq_5(k, dst + ir1, src0, src1);
            src1 = (const char *) src1 + quantize_5_row_size(k);
        }
        src0 = (const char *) src0 +   quantize_5_row_size(k);
        src1 = (const char *) src1 - n*quantize_5_row_size(k);

        dst = (float *) dst + n;
    }
}

//
// method 6
// same as 5 but with 32 element blocks
//

static inline int quantize_6_blocks_per_row(int k) {
    return k/32;
}

static inline int quantize_6_row_size(int k) {
    const int nb = quantize_6_blocks_per_row(k);

    return nb*(sizeof(gq_scale_t) + 16);
}

void quantize_6_row(const float * restrict src, void * restrict dst, int k) {
    assert(k % 32 == 0);
    assert(QB == 4);

    const int nb = quantize_6_blocks_per_row(k);

    gq_scale_t * restrict pd = (gq_scale_t *) (dst);
    uint8_t    * restrict pb = (uint8_t *)    (pd + nb);

    uint8_t pp[16];

    for (int i = 0; i < nb; i++) {
        memset(pp, 0, sizeof(pp));

        float amax = 0.0f; // absolute max
    }
#elif defined(__ARM_NEON)
        {
            # 定义8个float32x4_t类型的变量数组，用于存储数据
            float32x4_t srcv [8];
            float32x4_t asrcv[8];
            float32x4_t amaxv[8];

            # 使用ARM NEON指令加载src数组中的数据到srcv数组中
            for (int l = 0; l < 8; l++) srcv[l]  = vld1q_f32(src + i*32 + 4*l);
            # 对asrcv数组中的数据进行绝对值处理
            for (int l = 0; l < 8; l++) asrcv[l] = vabsq_f32(srcv[l]);

            # 使用ARM NEON指令计算每两个元素中的最大值，存储到amaxv数组中
            for (int l = 0; l < 4; l++) amaxv[2*l] = vmaxq_f32(asrcv[2*l], asrcv[2*l+1]);
            for (int l = 0; l < 2; l++) amaxv[4*l] = vmaxq_f32(amaxv[4*l], amaxv[4*l+2]);
            for (int l = 0; l < 1; l++) amaxv[8*l] = vmaxq_f32(amaxv[8*l], amaxv[8*l+4]);

            # 计算amaxv数组中的最大值
            amax = MAX(
                    MAX(vgetq_lane_f32(amaxv[0], 0), vgetq_lane_f32(amaxv[0], 1)),
                    MAX(vgetq_lane_f32(amaxv[0], 2), vgetq_lane_f32(amaxv[0], 3)));

            # 计算d和id的值
            const float d = amax / ((1 << 3) - 1);
            const float id = d ? 1.0/d : 0.0;

            # 将d转换为GGML_FP32_TO_GQ类型，并存储到pd数组中
            pd[i] = GGML_FP32_TO_GQ(d);

            # 使用ARM NEON指令对srcv数组中的数据进行处理，并存储到pp数组中
            for (int l = 0; l < 8; l++) {
                const float32x4_t v = vmulq_n_f32(srcv[l], id);
                const float32x4_t vf = vaddq_f32(v, vdupq_n_f32(8.5f));
                const int32x4_t vi = vcvtq_s32_f32(vf);

                pp[2*l + 0] = vgetq_lane_s32(vi, 0) | (vgetq_lane_s32(vi, 1) << 4);
                pp[2*l + 1] = vgetq_lane_s32(vi, 2) | (vgetq_lane_s32(vi, 3) << 4);
            }

            # 将pp数组中的数据拷贝到pb数组中
            memcpy(pb + i*16, pp, sizeof(pp));
        }
#else
        // 对于每个元素进行遍历，计算绝对值的最大值
        {
            for (int l = 0; l < 32; l++) {
                const float v = src[i*32 + l];
                amax = MAX(amax, fabsf(v));
            }

            // 计算最大值的绝对值与量化因子的比值
            const float d = amax / ((1 << (QB - 1)) - 1);
            const float id = d ? 1.0/d : 0.0;

            // 将量化因子转换为 GQ 类型并存储
            pd[i] = GGML_FP32_TO_GQ(d);

            // 对每个元素进行遍历，将其乘以量化因子并转换为 int8_t 类型
            for (int l = 0; l < 32; l++) {
                const float v = src[i*32 + l]*id;
                const int8_t vi = ((int8_t) (round(v))) + 8;
                assert(vi >= 0 && vi < 16);
                pp[l/2] |= (vi & 0xf) << (4*(l & 1));
            }

            // 将转换后的数据拷贝到目标数组中
            memcpy(pb + i*16, pp, sizeof(pp));
        }
#endif
        //printf("amax = %f\n", amax);
    }
}

// 重新实现 quantize__6，使用 quantize_6_row
void quantize_6(const float * restrict src, char * restrict dst, int n, int k) {
    // 确保 k 是 32 的倍数
    assert(k % 32 == 0);

    // 对每一行进行量化
    for (int j = 0; j < n; j++) {
        quantize_6_row(src + j*k, dst, k);
        dst = (char *) dst + quantize_6_row_size(k);
    }
}

void vec_dot_gq_6(const int n, float * restrict s, const void * restrict x, const void * restrict y) {
    // 计算每行的量化块数
    const int nb = quantize_6_blocks_per_row(n);

    // 将输入转换为 GQ 类型
    const gq_scale_t * restrict pd0 = (const gq_scale_t *) x;
    const gq_scale_t * restrict pd1 = (const gq_scale_t *) y;

    // 获取量化数据的指针
    const uint8_t * restrict pb0 = (const uint8_t *) (pd0 + nb);
    const uint8_t * restrict pb1 = (const uint8_t *) (pd1 + nb);

    // 初始化求和结果
    float sumf = 0.0;

#if 0
    // scalar
    # 遍历 nb 次，i 从 0 到 nb-1
    for (int i = 0; i < nb; i++) {
        # 将 pd0[i] 转换为 float 类型赋值给 d0
        const float d0 = GGML_GQ_TO_FP32(pd0[i]);
        # 将 pd1[i] 转换为 float 类型赋值给 d1

        const float d1 = GGML_GQ_TO_FP32(pd1[i]);

        # 计算指向 pb0 的偏移量为 i*16 的地址并赋值给 p0
        const uint8_t * restrict p0 = pb0 + i*16;
        # 计算指向 pb1 的偏移量为 i*16 的地址并赋值给 p1
        const uint8_t * restrict p1 = pb1 + i*16;

        # 遍历 16 次，j 从 0 到 15
        for (int j = 0; j < 16; j++) {
            # 取出 p0[j] 赋值给 v0
            const uint8_t v0 = p0[j];
            # 取出 p1[j] 赋值给 v1

            const uint8_t v1 = p1[j];

            # 计算 f0
            const float f0 = d0*((int8_t) (v0 & 0xf) - 8);
            # 计算 f1
            const float f1 = d0*((int8_t) (v0 >> 4)  - 8);

            # 计算 f2
            const float f2 = d1*((int8_t) (v1 & 0xf) - 8);
            # 计算 f3
            const float f3 = d1*((int8_t) (v1 >> 4)  - 8);

            # 计算 sumf
            sumf += f0*f2 + f1*f3;
        }
    }
#else
#if defined(__AVX2__)
    // 如果不满足上述条件，且支持 AVX2 指令集，则执行以下代码
    // TODO
#elif defined (__ARM_NEON)
#if 0
    // 如果不满足 AVX2 条件，但支持 ARM NEON 指令集，则执行以下代码
    float sum0 = 0.0f;
    // 初始化 sum0 为 0.0

    for (int i = 0; i < nb; i++) {
        // 遍历 nb 次循环
        const float d0 = GGML_GQ_TO_FP32(pd0[i]);
        // 将 pd0[i] 转换为 float 类型并赋值给 d0
        const float d1 = GGML_GQ_TO_FP32(pd1[i]);
        // 将 pd1[i] 转换为 float 类型并赋值给 d1

        const uint8_t * restrict p0 = pb0 + i*16;
        // 定义指向 pb0 + i*16 的 uint8_t 类型指针并赋值给 p0
        const uint8_t * restrict p1 = pb1 + i*16;
        // 定义指向 pb1 + i*16 的 uint8_t 类型指针并赋值给 p1

        // ...（以下代码为一系列 NEON 指令的操作，用于向量运算）

        // scalar
        // 标量计算
        sum0 += d0*d1*vaddvq_s16(p);
        // 将 d0*d1*vaddvq_s16(p) 的结果累加到 sum0 中
    }

    sumf = sum0;
    // 将 sum0 的值赋给 sumf
#elif 1 // this is a bit faster than the above
    // 如果上述条件不满足，且为真，则执行以下代码
    float sum0 = 0.0f;
    // 初始化 sum0 为 0.0
    float sum1 = 0.0f;
    // 初始化 sum1 为 0.0

    // ...（以下代码为一系列操作）

    sumf = sum0 + sum1;
    // 将 sum0 和 sum1 的和赋给 sumf
#endif
#endif
#endif

    *s = sumf;
    // 将 sumf 的值赋给指针 s 所指向的变量

// use vec_dot_gq_6 to compute the dot product of two rows
// 使用 vec_dot_gq_6 计算两行的点积
void mul_mat_gq_6(
    const void * src0,
    // 定义指向常量的指针 src1，指向浮点数的指针 dst，以及整数 m, n, k
    const void * src1, // transposed
         float * dst,
    int m, int n, int k) {
    // 断言 k 能被 32 整除
    assert(k % 32 == 0);

    // 循环遍历 m
    for (int ir0 = 0; ir0 < m; ir0++) {
        // 循环遍历 n
        for (int ir1 = 0; ir1 < n; ir1++) {
            // 调用 vec_dot_gq_6 函数，传入参数 k, dst + ir1, src0, src1
            vec_dot_gq_6(k, dst + ir1, src0, src1);
            // 更新 src1，指向下一行的数据
            src1 = (const char *) src1 + quantize_6_row_size(k);
        }
        // 更新 src0，指向下一行的数据
        src0 = (const char *) src0 +   quantize_6_row_size(k);
        // 更新 src1，指向上一行的数据
        src1 = (const char *) src1 - n*quantize_6_row_size(k);

        // 更新 dst，指向下一行的数据
        dst = (float *) dst + n;
    }
}

int main(int argc, const char ** argv) {
    // 确保 gq_quant_t 类型的大小乘以 8 等于 gq_t_bits 的值
    assert(sizeof(gq_quant_t)*8 == gq_t_bits);
    // 初始化时间
    ggml_time_init();

    // 初始化 f16 表所需
    {
        // 初始化参数结构体
        struct ggml_init_params params = { 0, NULL, false };
        // 初始化上下文
        struct ggml_context * ctx = ggml_init(params);
        // 释放上下文
        ggml_free(ctx);
    }

    // 初始化方法
    int method = 0;
    if (argc > 1) {
        method = atoi(argv[1]);
    }

    // 分配内存
    float * src0 = malloc(sizeof(float)*M*K);
    float * src1 = malloc(sizeof(float)*N*K);
    float * dst  = malloc(sizeof(float)*M*N);

    // 分配对齐内存
    //float * src0 = (float *)aligned_alloc(32, sizeof(float)*M*K);
    //float * src1 = (float *)aligned_alloc(32, sizeof(float)*N*K);
    //float * dst  = (float *)aligned_alloc(32, sizeof(float)*M*N);

    // 初始化 src0 数组
    for (int i = 0; i < M*K; i++) {
        src0[i] = 0.8 - rand() / (float)RAND_MAX;
        /*src0[i] = rand() / (float)RAND_MAX;*/
        /*src0[i] = i % 2;*/
    }

    // 初始化 src1 数组
    for (int i = 0; i < N*K; i++) {
        src1[i] = 0.8 - rand() / (float)RAND_MAX;
        /*src1[i] = rand() / (float)RAND_MAX;*/
        /*src1[i] = i % 3;*/
    }

    // 初始化 src0_gq 和 src1_gq
    void * src0_gq = NULL;
    void * src1_gq = NULL;

    // 初始化 sizegq
    size_t sizegq = 0;
    {
        // 如果方法为1，分配src0_gq和src1_gq的内存空间，并计算sizegq的大小
        if (method == 1) {
            src0_gq = calloc(1, quantize_1_row_size(K)*M);
            src1_gq = calloc(1, quantize_1_row_size(K)*N);

            sizegq  = quantize_1_row_size(K)*M + quantize_1_row_size(K)*N;
        }

        // 如果方法为2，分配src0_gq和src1_gq的内存空间，并计算sizegq的大小
        if (method == 2) {
            src0_gq = calloc(1, quantize_2_row_size(K)*M);
            src1_gq = calloc(1, quantize_2_row_size(K)*N);

            sizegq  = quantize_2_row_size(K)*M + quantize_2_row_size(K)*N;
        }

        // 如果方法为3，分配src0_gq和src1_gq的内存空间，并计算sizegq的大小
        if (method == 3) {
            src0_gq = calloc(1, quantize_3_row_size(K)*M);
            src1_gq = calloc(1, quantize_3_row_size(K)*N);

            sizegq  = quantize_3_row_size(K)*M + quantize_3_row_size(K)*N;
        }

        // 如果方法为4，分配src0_gq和src1_gq的内存空间，并计算sizegq的大小
        if (method == 4) {
            src0_gq = calloc(1, quantize_4_row_size(K)*M);
            src1_gq = calloc(1, quantize_4_row_size(K)*N);

            sizegq  = quantize_4_row_size(K)*M + quantize_4_row_size(K)*N;
        }

        // 如果方法为5，分配src0_gq和src1_gq的内存空间，并计算sizegq的大小
        if (method == 5) {
            src0_gq = calloc(1, quantize_5_row_size(K)*M);
            src1_gq = calloc(1, quantize_5_row_size(K)*N);

            sizegq  = quantize_5_row_size(K)*M + quantize_5_row_size(K)*N;
        }

        // 如果方法为6，分配src0_gq和src1_gq的内存空间，并计算sizegq的大小
        if (method == 6) {
            src0_gq = calloc(1, quantize_6_row_size(K)*M);
            src1_gq = calloc(1, quantize_6_row_size(K)*N);

            sizegq  = quantize_6_row_size(K)*M + quantize_6_row_size(K)*N;
        }
    }

    // 计算sizef16的大小
    const size_t sizef16 = sizeof(ggml_fp16_t)*M*K + sizeof(ggml_fp16_t)*N*K;

    // 打印压缩比
    printf("compression: %f\n", (float)sizegq/sizef16);

    // 将fp32转换为gq
    // 记录开始时间
    const int64_t t_start = ggml_time_us();

    // 根据不同的方法选择不同的量化函数对输入进行量化
    if (method == 1) {
        quantize_1(src0, src0_gq, M, K);
        quantize_1(src1, src1_gq, N, K);
    }

    if (method == 2) {
        quantize_2(src0, src0_gq, M, K);
        quantize_2(src1, src1_gq, N, K);
    }

    if (method == 3) {
        quantize_3(src0, src0_gq, M, K);
        quantize_3(src1, src1_gq, N, K);
    }

    if (method == 4) {
        quantize_4(src0, src0_gq, M, K);
        quantize_4(src1, src1_gq, N, K);
    }

    if (method == 5) {
        quantize_5(src0, src0_gq, M, K);
        quantize_5(src1, src1_gq, N, K);
    }

    if (method == 6) {
        quantize_6(src0, src0_gq, M, K);
        quantize_6(src1, src1_gq, N, K);
    }

    // 记录结束时间
    const int64_t t_end = ggml_time_us();
    // 打印转换时间和方法
    printf("convert time: %f ms / method = %d\n", (t_end - t_start) / 1000.0, method);

    // 打印前16个元素的值
    for (int i = 0; i < 16; ++i) {
        printf("%f %f\n", src0[i], src1[i]);
    }

    // 设置迭代次数
    const int nIter = 1;

    // 记录开始时钟周期和开始时间
    const int64_t start = ggml_cycles();
    const int64_t start_us = ggml_time_us();

    // 初始化变量
    double iM = 1.0/M;
    double sum = 0.0f;

    // 根据不同的方法选择不同的矩阵乘法函数进行计算
    for (int i = 0; i < nIter; i++) {
        if (method == 0) {
            mul_mat_f32_naive(src0, src1, dst, M, N, K);
        }

        if (method == 1) {
            mul_mat_gq_1(src0_gq, src1_gq, dst, M, N, K);
        }

        if (method == 2) {
            mul_mat_gq_2(src0_gq, src1_gq, dst, M, N, K);
        }

        if (method == 3) {
            mul_mat_gq_3(src0_gq, src1_gq, dst, M, N, K);
        }

        if (method == 4) {
            mul_mat_gq_4(src0_gq, src1_gq, dst, M, N, K);
        }

        if (method == 5) {
            mul_mat_gq_5(src0_gq, src1_gq, dst, M, N, K);
        }

        if (method == 6) {
            mul_mat_gq_6(src0_gq, src1_gq, dst, M, N, K);
        }
    }

    // 计算矩阵乘法结果的加权和
    for (int i = 0; i < N; i++) {
        sum += dst[i]*iM;
    }
    {
        // 获取当前时间，单位为 CPU 周期
        const int64_t end = ggml_cycles();
        // 获取当前时间，单位为微秒
        const int64_t end_us = ggml_time_us();
        // 打印函数执行所花费的 CPU 周期数
        printf("%s: elapsed ticks: %" PRIu64 "\n",  __func__, end - start);
        // 打印函数执行所花费的时间，单位为微秒，并转换为毫秒
        printf("%s: elapsed us:    %d / %f ms\n",  __func__, (int)(end_us - start_us), (end_us - start_us) / 1000.0 / nIter);
    }
#if 0
    // 如果条件为假，则执行以下代码块
    // 打印 src0 数组内容
    printf("src0:\n");
    for (int i = 0; i < M; i++) {
        for (int j = 0; j < K; j++) {
            printf("%4.1f ", src0[i*K+j]);
        }
        printf("\n");
    }

    // 打印 src1 数组内容
    printf("src1:\n");
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < K; j++) {
            printf("%4.1f ", src1[i*K+j]);
        }
        printf("\n");
    }

    // 打印 dst 数组内容
    printf("dst:\n");
    for (int i = 0; i < M; i++) {
        for (int j = 0; j < N; j++) {
            printf("%4.1f ", dst[i*N+j]);
        }
        printf("\n");
    }
#endif

    // 打印 sum 变量的值
    printf("%f\n", sum);

    // 释放 src0 数组内存
    free(src0);
    // 释放 src1 数组内存
    free(src1);
    // 释放 dst 数组内存
    free(dst);

    // 如果 src0_gq 不为空，则释放其内存
    if (src0_gq) free(src0_gq);
    // 如果 src1_gq 不为空，则释放其内存
    if (src1_gq) free(src1_gq);

    // 返回 0，表示程序正常结束
    return 0;
}
```