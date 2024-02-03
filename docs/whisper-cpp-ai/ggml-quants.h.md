# `whisper.cpp\ggml-quants.h`

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
// 每个权重元素使用 2.625 位进行量化
typedef struct {
    uint8_t scales[QK_K/16]; // 用 4 位进行量化的缩放因子和最小值
    uint8_t qs[QK_K/4];      // 量化值
    ggml_fp16_t d;           // 用于量化缩放因子的超级块缩放
    ggml_fp16_t dmin;        // 用于量化最小值的超级块缩放
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
    ggml_fp16_t d;             // 超级块缩放
} block_q3_K;
static_assert(sizeof(block_q3_K) == sizeof(ggml_fp16_t) + QK_K / 4 + QK_K / 8 + 2, "wrong q3_K block size/padding");
#else
typedef struct {
    uint8_t hmask[QK_K/8];     // 量化值 - 高位
    uint8_t qs[QK_K/4];        // 量化值 - 低 2 位
    uint8_t scales[12];        // 用 6 位进行量化的缩放因子
    ggml_fp16_t d;             // 超级块缩放
} block_q3_K;
static_assert(sizeof(block_q3_K) == sizeof(ggml_fp16_t) + QK_K / 4 + QK_K / 8 + 12, "wrong q3_K block size/padding");
#endif

// 4 位量化
// 每个 block 包含 32 个元素，共有 8 个 block
// 权重表示为 x = a * q + b
// 每个权重元素使用 4.5 位进行量化
#ifdef GGML_QKK_64
typedef struct {
    ggml_fp16_t d[2];          // 超级块缩放/最小值
    uint8_t scales[2];         // 4 位块缩放/最小值
    uint8_t qs[QK_K/2];        // 4 位量化值
} block_q4_K;
static_assert(sizeof(block_q4_K) == 2*sizeof(ggml_fp16_t) + QK_K/2 + 2, "wrong q4_K block size/padding");
#else
typedef struct {
    ggml_fp16_t d;             // 用于量化缩放因子的超级块缩放
    ggml_fp16_t dmin;          // 用于量化最小值的超级块缩放
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
// 检查 block_q8_K 结构体的大小是否正确，包括 float 类型、QK_K 和 QK_K/16*sizeof(int16_t) 的大小
static_assert(sizeof(block_q8_K) == sizeof(float) + QK_K + QK_K/16*sizeof(int16_t), "wrong q8_K block size/padding");

// 定义 (Almost) "true" 2-bit 量化的结构体 block_iq2_xxs，包括 ggml_fp16_t 类型和 uint16_t 类型数组
// 由于需要按照 ggml 设计使用块，因此每个块的 256 个元素使用 16 位比特表示，因此实际使用 2.0625 bpw
typedef struct {
    ggml_fp16_t d;
    uint16_t qs[QK_K/8];
} block_iq2_xxs;
// 检查 block_iq2_xxs 结构体的大小是否正确，包括 ggml_fp16_t 类型和 QK_K/8*sizeof(uint16_t) 的大小
static_assert(sizeof(block_iq2_xxs) == sizeof(ggml_fp16_t) + QK_K/8*sizeof(uint16_t), "wrong iq2_xxs block size/padding");

// 定义 2.3125 bpw 量化的结构体 block_iq2_xs，包括 ggml_fp16_t 类型、uint16_t 类型数组和 uint8_t 类型数组
typedef struct {
    ggml_fp16_t d;
    uint16_t qs[QK_K/8];
    uint8_t  scales[QK_K/32];
} block_iq2_xs;
// 检查 block_iq2_xs 结构体的大小是否正确，包括 ggml_fp16_t 类型、QK_K/8*sizeof(uint16_t) 和 QK_K/32 的大小
static_assert(sizeof(block_iq2_xs) == sizeof(ggml_fp16_t) + QK_K/8*sizeof(uint16_t) + QK_K/32, "wrong iq2_xs block size/padding");

// 定义 (Almost) "true" 3-bit 量化的结构体 block_iq3_xxs，包括 ggml_fp16_t 类型和 uint8_t 类型数组
// 由于需要按照 ggml 设计使用块，因此每个块的 256 个元素使用 16 位比特表示，因此实际使用 3.0625 bpw
typedef struct {
    ggml_fp16_t d;
    uint8_t qs[3*QK_K/8];
} block_iq3_xxs;
// 检查 block_iq3_xxs 结构体的大小是否正确，包括 ggml_fp16_t 类型和 3*(QK_K/8) 的大小
static_assert(sizeof(block_iq3_xxs) == sizeof(ggml_fp16_t) + 3*(QK_K/8), "wrong iq3_xxs block size/padding");

// 定义量化函数，将输入的浮点数组 x 进行量化后存储到对应的结构体 y 中，k 为参数
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k);
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k);
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k);
void quantize_row_q5_1_reference(const float * restrict x, block_q5_1 * restrict y, int k);
void quantize_row_q8_0_reference(const float * restrict x, block_q8_0 * restrict y, int k);
void quantize_row_q8_1_reference(const float * restrict x, block_q8_1 * restrict y, int k);

void quantize_row_q2_K_reference(const float * restrict x, block_q2_K * restrict y, int k);
void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k);
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k);
// 定义一系列函数用于将浮点数数组量化为特定类型的块数据结构
void quantize_row_q5_K_reference(const float * restrict x, block_q5_K * restrict y, int k);
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k);
void quantize_row_q8_K_reference(const float * restrict x, block_q8_K * restrict y, int k);
void quantize_row_iq3_xxs_reference(const float * restrict x, block_iq3_xxs * restrict y, int k);

void quantize_row_q4_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q4_1(const float * restrict x, void * restrict y, int k);
void quantize_row_q5_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q5_1(const float * restrict x, void * restrict y, int k);
void quantize_row_q8_0(const float * restrict x, void * restrict y, int k);
void quantize_row_q8_1(const float * restrict x, void * restrict y, int k);

void quantize_row_q2_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q3_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q4_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q5_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q6_K(const float * restrict x, void * restrict y, int k);
void quantize_row_q8_K(const float * restrict x, void * restrict y, int k);
void quantize_row_iq3_xxs(const float * restrict x, void * restrict y, int k);

// 定义一系列函数用于将特定类型的块数据结构反量化为浮点数数组
void dequantize_row_q4_0(const block_q4_0 * restrict x, float * restrict y, int k);
void dequantize_row_q4_1(const block_q4_1 * restrict x, float * restrict y, int k);
void dequantize_row_q5_0(const block_q5_0 * restrict x, float * restrict y, int k);
void dequantize_row_q5_1(const block_q5_1 * restrict x, float * restrict y, int k);
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k);
//void dequantize_row_q8_1(const block_q8_1 * restrict x, float * restrict y, int k);

void dequantize_row_q2_K(const block_q2_K * restrict x, float * restrict y, int k);
// 对给定的 block_q3_K 结构体进行反量化，将结果存储到 float 类型数组中，处理第 k 行数据
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k);

// 对给定的 block_q4_K 结构体进行反量化，将结果存储到 float 类型数组中，处理第 k 行数据
void dequantize_row_q4_K(const block_q4_K * restrict x, float * restrict y, int k);

// 对给定的 block_q5_K 结构体进行反量化，将结果存储到 float 类型数组中，处理第 k 行数据
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k);

// 对给定的 block_q6_K 结构体进行反量化，将结果存储到 float 类型数组中，处理第 k 行数据
void dequantize_row_q6_K(const block_q6_K * restrict x, float * restrict y, int k);

// 对给定的 block_q8_K 结构体进行反量化，将结果存储到 float 类型数组中，处理第 k 行数据
void dequantize_row_q8_K(const block_q8_K * restrict x, float * restrict y, int k);

// 对给定的 block_iq2_xxs 结构体进行反量化，将结果存储到 float 类型数组中，处理第 k 行数据
void dequantize_row_iq2_xxs(const block_iq2_xxs * restrict x, float * restrict y, int k);

// 对给定的 block_iq2_xs 结构体进行反量化，将结果存储到 float 类型数组中，处理第 k 行数据
void dequantize_row_iq2_xs (const block_iq2_xs  * restrict x, float * restrict y, int k);

// 对给定的 block_iq3_xxs 结构体进行反量化，将结果存储到 float 类型数组中，处理第 k 行数据
void dequantize_row_iq3_xxs(const block_iq3_xxs * restrict x, float * restrict y, int k);

// 计算两个向量的点积，其中向量元素为 q4 和 q8 类型
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q4 和 q8 类型
void ggml_vec_dot_q4_1_q8_1(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q5 和 q8 类型
void ggml_vec_dot_q5_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q5 和 q8 类型
void ggml_vec_dot_q5_1_q8_1(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q8 和 q8 类型
void ggml_vec_dot_q8_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q2 和 q8 类型
void ggml_vec_dot_q2_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q3 和 q8 类型
void ggml_vec_dot_q3_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q4 和 q8 类型
void ggml_vec_dot_q4_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q5 和 q8 类型
void ggml_vec_dot_q5_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 q6 和 q8 类型
void ggml_vec_dot_q6_K_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

// 计算两个向量的点积，其中向量元素为 iq2_xxs 和 q8 类型
void ggml_vec_dot_iq2_xxs_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
// 声明函数 ggml_vec_dot_iq2_xs_q8_K，用于计算两个向量的点积
void ggml_vec_dot_iq2_xs_q8_K (int n, float * restrict s, const void * restrict vx, const void * restrict vy);
// 声明函数 ggml_vec_dot_iq3_xxs_q8_K，用于计算两个向量的点积
void ggml_vec_dot_iq3_xxs_q8_K(int n, float * restrict s, const void * restrict vx, const void * restrict vy);

//
// 利用重要性矩阵进行量化（也称为“激活感知量化”）
//
// 声明函数 quantize_iq2_xxs，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_iq2_xxs(const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_iq2_xs，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_iq2_xs (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_iq3_xxs，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_iq3_xxs(const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q2_K，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q2_K   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q3_K，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q3_K   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q4_K，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q4_K   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q5_K，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q5_K   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q6_K，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q6_K   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q4_0，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q4_0   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q4_1，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q4_1   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q5_0，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q5_0   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);
// 声明函数 quantize_q5_1，用于将浮点数数组进行量化，使用重要性矩阵
size_t quantize_q5_1   (const float * src, void * dst, int nrows, int n_per_row, int64_t * hist, const float * imatrix);

// 初始化 iq2xs 算法所需的资源
void iq2xs_init_impl(int grid_size);
// 释放 iq2xs 算法所占用的资源
void iq2xs_free_impl(int grid_size);
// 初始化 iq3xs 算法所需的资源
void iq3xs_init_impl(int grid_size);
// 释放 iq3xs 算法所占用的资源
void iq3xs_free_impl(int grid_size);
```