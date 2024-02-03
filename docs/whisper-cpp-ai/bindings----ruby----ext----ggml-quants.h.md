# `whisper.cpp\bindings\ruby\ext\ggml-quants.h`

```cpp
#pragma once

#include "ggml-impl.h"

// GGML internal header

#include <stdint.h>
#include <stddef.h>

// 定义 QK4_0 常量为 32
#define QK4_0 32
// 定义 block_q4_0 结构体，包含 delta 和 nibbles/quants 数组
typedef struct {
    ggml_fp16_t d;          // delta
    uint8_t qs[QK4_0 / 2];  // nibbles / quants
} block_q4_0;
// 静态断言，检查 block_q4_0 结构体大小是否正确
static_assert(sizeof(block_q4_0) == sizeof(ggml_fp16_t) + QK4_0 / 2, "wrong q4_0 block size/padding");

// 定义 QK4_1 常量为 32
#define QK4_1 32
// 定义 block_q4_1 结构体，包含 delta、min 和 nibbles/quants 数组
typedef struct {
    ggml_fp16_t d;          // delta
    ggml_fp16_t m;          // min
    uint8_t qs[QK4_1 / 2];  // nibbles / quants
} block_q4_1;
// 静态断言，检查 block_q4_1 结构体大小是否正确
static_assert(sizeof(block_q4_1) == 2 * sizeof(ggml_fp16_t) + QK4_1 / 2, "wrong q4_1 block size/padding");

// 定义 QK5_0 常量为 32
#define QK5_0 32
// 定义 block_q5_0 结构体，包含 delta、5-th bit of quants 和 nibbles/quants 数组
typedef struct {
    ggml_fp16_t d;         // delta
    uint8_t qh[4];         // 5-th bit of quants
    uint8_t qs[QK5_0 / 2]; // nibbles / quants
} block_q5_0;
// 静态断言，检查 block_q5_0 结构体大小是否正确
static_assert(sizeof(block_q5_0) == sizeof(ggml_fp16_t) + sizeof(uint32_t) + QK5_0 / 2, "wrong q5_0 block size/padding");

// 定义 QK5_1 常量为 32
#define QK5_1 32
// 定义 block_q5_1 结构体，包含 delta、min、5-th bit of quants 和 nibbles/quants 数组
typedef struct {
    ggml_fp16_t d;         // delta
    ggml_fp16_t m;         // min
    uint8_t qh[4];         // 5-th bit of quants
    uint8_t qs[QK5_1 / 2]; // nibbles / quants
} block_q5_1;
// 静态断言，检查 block_q5_1 结构体大小是否正确
static_assert(sizeof(block_q5_1) == 2 * sizeof(ggml_fp16_t) + sizeof(uint32_t) + QK5_1 / 2, "wrong q5_1 block size/padding");

// 定义 QK8_0 常量为 32
#define QK8_0 32
// 定义 block_q8_0 结构体，包含 delta 和 quants 数组
typedef struct {
    ggml_fp16_t d;         // delta
    int8_t  qs[QK8_0];     // quants
} block_q8_0;
// 静态断言，检查 block_q8_0 结构体大小是否正确
static_assert(sizeof(block_q8_0) == sizeof(ggml_fp16_t) + QK8_0, "wrong q8_0 block size/padding");

// 定义 QK8_1 常量为 32
#define QK8_1 32
// 定义 block_q8_1 结构体，包含 delta、d * sum(qs[i]) 和 quants 数组
typedef struct {
    float d;               // delta
    float s;               // d * sum(qs[i])
    int8_t  qs[QK8_1];     // quants
} block_q8_1;
// 静态断言，检查 block_q8_1 结构体大小是否正确
static_assert(sizeof(block_q8_1) == 2*sizeof(float) + QK8_1, "wrong q8_1 block size/padding");

//
// Super-block quantization structures
//

// 根据 GGML_QKK_64 宏定义确定超级块大小和 K_SCALE_SIZE
#ifdef GGML_QKK_64
#define QK_K 64
#define K_SCALE_SIZE 4
#else
#define QK_K 256
#define K_SCALE_SIZE 12
#endif

// 2-bit quantization
// weight is represented as x = a * q + b
// 定义一个结构体 block_q2_K，包含了用于量化的数据和缩放因子
// 每个 block 包含 16 个元素，共有 16 个 block
// 每个权重元素使用 2.5625 位进行量化
typedef struct {
    uint8_t scales[QK_K/16]; // 使用 4 位进行量化的缩放因子和最小值
    uint8_t qs[QK_K/4];      // 量化值
    ggml_fp16_t d;           // 用于量化缩放因子的超级块尺度
    ggml_fp16_t dmin;        // 用于量化最小值的超级块尺度
} block_q2_K;
static_assert(sizeof(block_q2_K) == 2*sizeof(ggml_fp16_t) + QK_K/16 + QK_K/4, "wrong q2_K block size/padding");

// 3 位量化
// 权重表示为 x = a * q
// 每个 block 包含 16 个元素，共有 16 个 block
// 每个权重元素使用 3.4375 位进行量化
#ifdef GGML_QKK_64
typedef struct {
    uint8_t hmask[QK_K/8];     // 量化值 - 高位
    uint8_t qs[QK_K/4];        // 量化值 - 低 2 位
    uint8_t scales[2];
    ggml_fp16_t d;             // 超级块尺度
} block_q3_K;
static_assert(sizeof(block_q3_K) == sizeof(ggml_fp16_t) + QK_K / 4 + QK_K / 8 + 2, "wrong q3_K block size/padding");
#else
typedef struct {
    uint8_t hmask[QK_K/8];     // 量化值 - 高位
    uint8_t qs[QK_K/4];        // 量化值 - 低 2 位
    uint8_t scales[12];        // 使用 6 位进行量化的缩放因子
    ggml_fp16_t d;             // 超级块尺度
} block_q3_K;
static_assert(sizeof(block_q3_K) == sizeof(ggml_fp16_t) + QK_K / 4 + QK_K / 8 + 12, "wrong q3_K block size/padding");
#endif

// 4 位量化
// 每个 block 包含 32 个元素，共有 8 个 block
// 权重表示为 x = a * q + b
// 每个权重元素使用 4.5 位进行量化
#ifdef GGML_QKK_64
typedef struct {
    ggml_fp16_t d[2];          // 超级块尺度/最小值
    uint8_t scales[2];         // 使用 4 位进行量化的缩放因子/最小值
    uint8_t qs[QK_K/2];        // 使用 4 位进行量化的量化值
} block_q4_K;
static_assert(sizeof(block_q4_K) == 2*sizeof(ggml_fp16_t) + QK_K/2 + 2, "wrong q4_K block size/padding");
#else
typedef struct {
    ggml_fp16_t d;             // 用于量化缩放因子的超级块尺度
    ggml_fp16_t dmin;          // 用于量化最小值的超级块尺度
    // 定义一个长度为 K_SCALE_SIZE 的 uint8_t 类型数组，用于存储经过量化为 6 位的比例和最小值
    uint8_t scales[K_SCALE_SIZE]; 
    // 定义一个长度为 QK_K/2 的 uint8_t 类型数组，用于存储经过 4 位量化的值
    uint8_t qs[QK_K/2];        
#ifdef GGML_QKK_64
// 如果定义了 GGML_QKK_64 宏，则使用 5 位量化
typedef struct {
    ggml_fp16_t d;               // 超级块的缩放因子
    int8_t  scales[QK_K/16];     // 8 位块的缩放因子
    uint8_t qh[QK_K/8];          // 量化值的高 1 位
    uint8_t qs[QK_K/2];          // 量化值的低 4 位
} block_q5_K;
// 检查结构体 block_q5_K 的大小是否正确
static_assert(sizeof(block_q5_K) == sizeof(ggml_fp16_t) + QK_K/2 + QK_K/8 + QK_K/16, "wrong q5_K block size/padding");
#else
// 如果未定义 GGML_QKK_64 宏，则使用 6 位量化
typedef struct {
    ggml_fp16_t d;               // 用于量化缩放因子的超级块缩放因子
    ggml_fp16_t dmin;            // 用于量化最小值的超级块缩放因子
    uint8_t scales[K_SCALE_SIZE];   // 使用 6 位量化的缩放因子和最小值
    uint8_t qh[QK_K/8];          // 量化值的高 1 位
    uint8_t qs[QK_K/2];          // 量化值的低 4 位
} block_q5_K;
// 检查结构体 block_q5_K 的大小是否正确
static_assert(sizeof(block_q5_K) == 2*sizeof(ggml_fp16_t) + K_SCALE_SIZE + QK_K/2 + QK_K/8, "wrong q5_K block size/padding");
#endif

// 使用 6 位量化
// 权重表示为 x = a * q
// 每个包含 16 个元素的 16 个块
// 每个权重有效位数为 6.5625
typedef struct {
    uint8_t ql[QK_K/2];      // 量化值的低 4 位
    uint8_t qh[QK_K/4];      // 量化值的高 2 位
    int8_t  scales[QK_K/16]; // 缩放因子，使用 8 位量化
    ggml_fp16_t d;           // 超级块的缩放因子
} block_q6_K;
// 检查结构体 block_q6_K 的大小是否正确
static_assert(sizeof(block_q6_K) == sizeof(ggml_fp16_t) + QK_K / 16 + 3*QK_K/4, "wrong q6_K block size/padding");

// 仅用于中间量化和点积
typedef struct {
    float   d;              // 增量
    int8_t  qs[QK_K];       // 量化值
    int16_t bsums[QK_K/16]; // 每组 16 个量化值的总和
} block_q8_K;
// 确保 block_q8_K 结构体的大小与指定的大小相匹配，用于检查内存布局是否正确
static_assert(sizeof(block_q8_K) == sizeof(float) + QK_K + QK_K/16*sizeof(int16_t), "wrong q8_K block size/padding");

// 定义一系列用于将浮点数数组量化为特定类型的数据结构的函数声明
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k);
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k);
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k);
void quantize_row_q5_1_reference(const float * restrict x, block_q5_1 * restrict y, int k);
void quantize_row_q8_0_reference(const float * restrict x, block_q8_0 * restrict y, int k);
void quantize_row_q8_1_reference(const float * restrict x, block_q8_1 * restrict y, int k);

void quantize_row_q2_K_reference(const float * restrict x, block_q2_K * restrict y, int k);
void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k);
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k);
void quantize_row_q5_K_reference(const float * restrict x, block_q5_K * restrict y, int k);
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k);
void quantize_row_q8_K_reference(const float * restrict x, block_q8_K * restrict y, int k);

// 定义一系列用于将浮点数数组量化为特定类型的数据结构的函数声明，但返回类型为 void *
void quantize_row_q4_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q4_1(const float * restrict x, void * restrict y, int k);
void quantize_row_q5_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q5_1(const float * restrict x, void * restrict y, int k);
void quantize_row_q8_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q8_1(const float * restrict x, void * restrict y, int k);

void quantize_row_q2_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q3_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q4_K(const float * restrict x, void * restrict y, int k);
// 定义函数 quantize_row_q5_K，将输入数组 x 中的数据量化后存储到 y 中，数据量化级别为 K
void quantize_row_q5_K(const float * restrict x, void * restrict y, int k);
// 定义函数 quantize_row_q6_K，将输入数组 x 中的数据量化后存储到 y 中，数据量化级别为 K
void quantize_row_q6_K(const float * restrict x, void * restrict y, int k);
// 定义函数 quantize_row_q8_K，将输入数组 x 中的数据量化后存储到 y 中，数据量化级别为 K
void quantize_row_q8_K(const float * restrict x, void * restrict y, int k);

// 反量化
// 定义函数 dequantize_row_q4_0，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 0
void dequantize_row_q4_0(const block_q4_0 * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q4_1，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 1
void dequantize_row_q4_1(const block_q4_1 * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q5_0，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 0
void dequantize_row_q5_0(const block_q5_0 * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q5_1，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 1
void dequantize_row_q5_1(const block_q5_1 * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q8_0，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 0
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k);
//void dequantize_row_q8_1(const block_q8_1 * restrict x, float * restrict y, int k);

// 定义函数 dequantize_row_q2_K，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 K
void dequantize_row_q2_K(const block_q2_K * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q3_K，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 K
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q4_K，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 K
void dequantize_row_q4_K(const block_q4_K * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q5_K，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 K
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q6_K，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 K
void dequantize_row_q6_K(const block_q6_K * restrict x, float * restrict y, int k);
// 定义函数 dequantize_row_q8_K，将输入结构体 x 中的数据反量化后存储到数组 y 中，数据量化级别为 K
void dequantize_row_q8_K(const block_q8_K * restrict x, float * restrict y, int k);

// 点积
// 定义函数 ggml_vec_dot_q4_0_q8_0，计算两个向量的点积并存储到数组 s 中，向量数据量化级别分别为 0 和 0
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
// 定义函数 ggml_vec_dot_q4_1_q8_1，计算两个向量的点积并存储到数组 s 中，向量数据量化级别分别为 1 和 1
void ggml_vec_dot_q4_1_q8_1(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
// 定义函数 ggml_vec_dot_q5_0_q8_0，计算两个向量的点积并存储到数组 s 中，向量数据量化级别分别为 0 和 0
void ggml_vec_dot_q5_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
// 定义函数 ggml_vec_dot_q5_1_q8_1，计算两个向量的点积并存储到数组 s 中，向量数据量化级别分别为 1 和 1
void ggml_vec_dot_q5_1_q8_1(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
// 定义函数 ggml_vec_dot_q8_0_q8_0，计算两个向量的点积并存储到数组 s 中，向量数据量化级别分别为 0 和 0
void ggml_vec_dot_q8_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 定义函数 ggml_vec_dot_q2_K_q8_K，计算两个向量的点积并存储到数组 s 中，向量数据量化级别分别为 K 和 K
void ggml_vec_dot_q2_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
# 定义四个函数，用于计算两个向量的点积并将结果存储在指定的数组中
void ggml_vec_dot_q3_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q4_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q5_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q6_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
```