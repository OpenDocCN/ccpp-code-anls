# `ggml\src\ggml-quants.h`

```
#pragma once

#include "ggml-impl.h"

// GGML internal header

#include <stdint.h>
#include <stddef.h>

#define QK4_0 32
typedef struct {
    ggml_fp16_t d;          // delta
    uint8_t qs[QK4_0 / 2];  // nibbles / quants
} block_q4_0;
static_assert(sizeof(block_q4_0) == sizeof(ggml_fp16_t) + QK4_0 / 2, "wrong q4_0 block size/padding");

#define QK4_1 32
typedef struct {
    ggml_fp16_t d;          // delta
    ggml_fp16_t m;          // min
    uint8_t qs[QK4_1 / 2];  // nibbles / quants
} block_q4_1;
static_assert(sizeof(block_q4_1) == 2 * sizeof(ggml_fp16_t) + QK4_1 / 2, "wrong q4_1 block size/padding");

#define QK5_0 32
typedef struct {
    ggml_fp16_t d;         // delta
    uint8_t qh[4];         // 5-th bit of quants
    uint8_t qs[QK5_0 / 2]; // nibbles / quants
} block_q5_0;
static_assert(sizeof(block_q5_0) == sizeof(ggml_fp16_t) + sizeof(uint32_t) + QK5_0 / 2, "wrong q5_0 block size/padding");

#define QK5_1 32
typedef struct {
    ggml_fp16_t d;         // delta
    ggml_fp16_t m;         // min
    uint8_t qh[4];         // 5-th bit of quants
    uint8_t qs[QK5_1 / 2]; // nibbles / quants
} block_q5_1;
static_assert(sizeof(block_q5_1) == 2 * sizeof(ggml_fp16_t) + sizeof(uint32_t) + QK5_1 / 2, "wrong q5_1 block size/padding");

#define QK8_0 32
typedef struct {
    ggml_fp16_t d;         // delta
    int8_t  qs[QK8_0];     // quants
} block_q8_0;
static_assert(sizeof(block_q8_0) == sizeof(ggml_fp16_t) + QK8_0, "wrong q8_0 block size/padding");

#define QK8_1 32
typedef struct {
    float d;               // delta
    float s;               // d * sum(qs[i])
    int8_t  qs[QK8_1];     // quants
} block_q8_1;
static_assert(sizeof(block_q8_1) == 2*sizeof(float) + QK8_1, "wrong q8_1 block size/padding");

//
// Super-block quantization structures
//

// Super-block size
#ifdef GGML_QKK_64
#define QK_K 64
#define K_SCALE_SIZE 4
#else
#define QK_K 256
#define K_SCALE_SIZE 12
#endif

// 2-bit quantization
// weight is represented as x = a * q + b
// 定义一个结构体，包含16个元素的数组，每个元素占用8位，用于存储权重的比例和最小值，使用4位进行量化
// 以及包含16个元素的数组，每个元素占用8位，用于存储量化值
// 以及两个ggml_fp16_t类型的变量，用于存储超级块的量化比例
typedef struct {
    uint8_t scales[QK_K/16]; // scales and mins, quantized with 4 bits
    uint8_t qs[QK_K/4];      // quants
    ggml_fp16_t d;           // super-block scale for quantized scales
    ggml_fp16_t dmin;        // super-block scale for quantized mins
} block_q2_K;
static_assert(sizeof(block_q2_K) == 2*sizeof(ggml_fp16_t) + QK_K/16 + QK_K/4, "wrong q2_K block size/padding");

// 3位量化
// 权重表示为x = a * q
// 包含16个元素的数组，每个元素占用8位，用于存储高位量化值
// 以及包含16个元素的数组，每个元素占用8位，用于存储低2位量化值
// 以及包含2个元素的数组，每个元素占用8位，用于存储量化比例
// 以及一个ggml_fp16_t类型的变量，用于存储超级块的量化比例
#ifdef GGML_QKK_64
typedef struct {
    uint8_t hmask[QK_K/8];     // quants - high bit
    uint8_t qs[QK_K/4];        // quants - low 2 bits
    uint8_t scales[2];
    ggml_fp16_t d;             // super-block scale
} block_q3_K;
static_assert(sizeof(block_q3_K) == sizeof(ggml_fp16_t) + QK_K / 4 + QK_K / 8 + 2, "wrong q3_K block size/padding");
#else
typedef struct {
    uint8_t hmask[QK_K/8];     // quants - high bit
    uint8_t qs[QK_K/4];        // quants - low 2 bits
    uint8_t scales[12];        // scales, quantized with 6 bits
    ggml_fp16_t d;             // super-block scale
} block_q3_K;
static_assert(sizeof(block_q3_K) == sizeof(ggml_fp16_t) + QK_K / 4 + QK_K / 8 + 12, "wrong q3_K block size/padding");
#endif

// 4位量化
// 包含32个元素的数组，每个元素占用8位，用于存储量化比例和最小值
// 以及包含32个元素的数组，每个元素占用8位，用于存储量化值
// 以及两个ggml_fp16_t类型的变量，用于存储超级块的量化比例
#ifdef GGML_QKK_64
typedef struct {
    ggml_fp16_t d[2];          // super-block scales/mins
    uint8_t scales[2];         // 4-bit block scales/mins
    uint8_t qs[QK_K/2];        // 4--bit quants
} block_q4_K;
static_assert(sizeof(block_q4_K) == 2*sizeof(ggml_fp16_t) + QK_K/2 + 2, "wrong q4_K block size/padding");
#else
typedef struct {
    ggml_fp16_t d;             // super-block scale for quantized scales
    ggml_fp16_t dmin;          // super-block scale for quantized mins
    // 用于存储经过6位量化的比例和最小值
    uint8_t scales[K_SCALE_SIZE]; 
    // 用于存储4位量化的值
    uint8_t qs[QK_K/2];        
} block_q4_K;
static_assert(sizeof(block_q4_K) == 2*sizeof(ggml_fp16_t) + K_SCALE_SIZE + QK_K/2, "wrong q4_K block size/padding");
#endif

// 5-bit quantization
// 8 blocks of 32 elements each
// weight is represented as x = a * q + b
// Effectively 5.5 bits per weight
#ifdef GGML_QKK_64
typedef struct {
    ggml_fp16_t d;               // super-block scale
    int8_t  scales[QK_K/16];     // 8-bit block scales
    uint8_t qh[QK_K/8];          // quants, high bit
    uint8_t qs[QK_K/2];          // quants, low 4 bits
} block_q5_K;
static_assert(sizeof(block_q5_K) == sizeof(ggml_fp16_t) + QK_K/2 + QK_K/8 + QK_K/16, "wrong q5_K block size/padding");
#else
typedef struct {
    ggml_fp16_t d;               // super-block scale for quantized scales
    ggml_fp16_t dmin;            // super-block scale for quantized mins
    uint8_t scales[K_SCALE_SIZE];   // scales and mins, quantized with 6 bits
    uint8_t qh[QK_K/8];          // quants, high bit
    uint8_t qs[QK_K/2];          // quants, low 4 bits
} block_q5_K;
static_assert(sizeof(block_q5_K) == 2*sizeof(ggml_fp16_t) + K_SCALE_SIZE + QK_K/2 + QK_K/8, "wrong q5_K block size/padding");
#endif

// 6-bit quantization
// weight is represented as x = a * q
// 16 blocks of 16 elements each
// Effectively 6.5625 bits per weight
typedef struct {
    uint8_t ql[QK_K/2];      // quants, lower 4 bits
    uint8_t qh[QK_K/4];      // quants, upper 2 bits
    int8_t  scales[QK_K/16]; // scales, quantized with 8 bits
    ggml_fp16_t d;           // super-block scale
} block_q6_K;
static_assert(sizeof(block_q6_K) == sizeof(ggml_fp16_t) + QK_K / 16 + 3*QK_K/4, "wrong q6_K block size/padding");

// This is only used for intermediate quantization and dot products
typedef struct {
    float   d;              // delta
    int8_t  qs[QK_K];       // quants
    int16_t bsums[QK_K/16]; // sum of quants in groups of 16
} block_q8_K;
// 检查 block_q8_K 结构体的大小是否正确，包括 float 类型、QK_K 和 QK_K/16*sizeof(int16_t) 的大小
static_assert(sizeof(block_q8_K) == sizeof(float) + QK_K + QK_K/16*sizeof(int16_t), "wrong q8_K block size/padding");

// 定义一系列用于将浮点数数组量化为特定类型的函数，函数名中的数字表示量化的位数
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k);
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k);
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k);
void quantize_row_q5_1_reference(const float * restrict x, block_q5_1 * restrict y, int k);
void quantize_row_q8_0_reference(const float * restrict x, block_q8_0 * restrict y, int k);
void quantize_row_q8_1_reference(const float * restrict x, block_q8_1 * restrict y, int k);

// 定义一系列用于将浮点数数组量化为特定类型的函数，函数名中的数字表示量化的位数，但返回类型为 void*
void quantize_row_q2_K_reference(const float * restrict x, block_q2_K * restrict y, int k);
void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k);
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k);
void quantize_row_q5_K_reference(const float * restrict x, block_q5_K * restrict y, int k);
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k);
void quantize_row_q8_K_reference(const float * restrict x, block_q8_K * restrict y, int k);

// 定义一系列用于将浮点数数组量化为特定类型的函数，函数名中的数字表示量化的位数，但返回类型为 void*
void quantize_row_q4_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q4_1(const float * restrict x, void * restrict y, int k);
void quantize_row_q5_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q5_1(const float * restrict x, void * restrict y, int k);
void quantize_row_q8_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q8_1(const float * restrict x, void * restrict y, int k);

// 定义一系列用于将浮点数数组量化为特定类型的函数，函数名中的数字表示量化的位数，但返回类型为 void*
void quantize_row_q2_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q3_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q4_K(const float * restrict x, void * restrict y, int k);
# 定义函数 quantize_row_q5_K，接受一个浮点数数组 x 和一个指针 y，以及一个整数 k
void quantize_row_q5_K(const float * restrict x, void * restrict y, int k);

# 定义函数 quantize_row_q6_K，接受一个浮点数数组 x 和一个指针 y，以及一个整数 k
void quantize_row_q6_K(const float * restrict x, void * restrict y, int k);

# 定义函数 quantize_row_q8_K，接受一个浮点数数组 x 和一个指针 y，以及一个整数 k
void quantize_row_q8_K(const float * restrict x, void * restrict y, int k);

# 定义函数 dequantize_row_q4_0，接受一个 block_q4_0 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q4_0(const block_q4_0 * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q4_1，接受一个 block_q4_1 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q4_1(const block_q4_1 * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q5_0，接受一个 block_q5_0 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q5_0(const block_q5_0 * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q5_1，接受一个 block_q5_1 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q5_1(const block_q5_1 * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q8_0，接受一个 block_q8_0 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q2_K，接受一个 block_q2_K 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q2_K(const block_q2_K * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q3_K，接受一个 block_q3_K 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q4_K，接受一个 block_q4_K 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q4_K(const block_q4_K * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q5_K，接受一个 block_q5_K 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q6_K，接受一个 block_q6_K 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q6_K(const block_q6_K * restrict x, float * restrict y, int k);

# 定义函数 dequantize_row_q8_K，接受一个 block_q8_K 结构体指针 x 和一个浮点数数组 y，以及一个整数 k
void dequantize_row_q8_K(const block_q8_K * restrict x, float * restrict y, int k);

# 定义函数 ggml_vec_dot_q4_0_q8_0，接受一个整数 n，一个浮点数数组指针 s，以及两个 void 类型的指针 vx 和 vy
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

# 定义函数 ggml_vec_dot_q4_1_q8_1，接受一个整数 n，一个浮点数数组指针 s，以及两个 void 类型的指针 vx 和 vy
void ggml_vec_dot_q4_1_q8_1(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

# 定义函数 ggml_vec_dot_q5_0_q8_0，接受一个整数 n，一个浮点数数组指针 s，以及两个 void 类型的指针 vx 和 vy
void ggml_vec_dot_q5_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

# 定义函数 ggml_vec_dot_q5_1_q8_1，接受一个整数 n，一个浮点数数组指针 s，以及两个 void 类型的指针 vx 和 vy
void ggml_vec_dot_q5_1_q8_1(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

# 定义函数 ggml_vec_dot_q8_0_q8_0，接受一个整数 n，一个浮点数数组指针 s，以及两个 void 类型的指针 vx 和 vy
void ggml_vec_dot_q8_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

# 定义函数 ggml_vec_dot_q2_K_q8_K，接受一个整数 n，一个浮点数数组指针 s，以及两个 void 类型的指针 vx 和 vy
void ggml_vec_dot_q2_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
# 定义了四个函数，用于计算两个向量的点积
# 参数n表示向量的长度，参数s表示计算结果，参数vx和vy分别表示两个向量的指针
void ggml_vec_dot_q3_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q4_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q5_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q6_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
```