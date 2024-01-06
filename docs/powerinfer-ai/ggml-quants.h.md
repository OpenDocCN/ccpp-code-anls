# `PowerInfer\ggml-quants.h`

```
#pragma once

#include "ggml-impl.h"

// GGML internal header

#include <stdint.h>  // 包含标准整数类型的头文件
#include <stddef.h>  // 包含标准大小类型的头文件

#define QK4_0 32  // 定义宏，表示 QK4_0 的值为 32
typedef struct {
    ggml_fp16_t d;          // delta，表示 ggml_fp16_t 类型的变量 d，用于存储 delta 值
    uint8_t qs[QK4_0 / 2];  // nibbles / quants，表示 uint8_t 类型的数组 qs，用于存储 nibbles / quants
} block_q4_0;  // 定义结构体 block_q4_0，包含 delta 和数组 qs
static_assert(sizeof(block_q4_0) == sizeof(ggml_fp16_t) + QK4_0 / 2, "wrong q4_0 block size/padding");  // 静态断言，检查 block_q4_0 结构体的大小是否符合预期

#define QK4_1 32  // 定义宏，表示 QK4_1 的值为 32
typedef struct {
    ggml_fp16_t d;          // delta，表示 ggml_fp16_t 类型的变量 d，用于存储 delta 值
    ggml_fp16_t m;          // min，表示 ggml_fp16_t 类型的变量 m，用于存储 min 值
// 定义一个包含 QK4_1 / 2 个 uint8_t 类型元素的数组，用于存储 nibbles / quants
} block_q4_1;
// 使用 static_assert 断言检查 block_q4_1 结构体的大小是否符合预期
static_assert(sizeof(block_q4_1) == 2 * sizeof(ggml_fp16_t) + QK4_1 / 2, "wrong q4_1 block size/padding");

// 定义一个包含 QK5_0 个 uint8_t 类型元素的数组，用于存储 nibbles / quants
} block_q5_0;
// 使用 static_assert 断言检查 block_q5_0 结构体的大小是否符合预期
static_assert(sizeof(block_q5_0) == sizeof(ggml_fp16_t) + sizeof(uint32_t) + QK5_0 / 2, "wrong q5_0 block size/padding");

// 定义一个包含 QK5_1 个 uint8_t 类型元素的数组，用于存储 nibbles / quants
} block_q5_1;
// 使用 static_assert 断言检查 block_q5_1 结构体的大小是否符合预期
static_assert(sizeof(block_q5_1) == 2 * sizeof(ggml_fp16_t) + sizeof(uint32_t) + QK5_1 / 2, "wrong q5_1 block size/padding");
// 定义了一个名为 QK8_0 的宏，其值为 32
typedef struct {
    ggml_fp16_t d;         // delta，表示 delta 值
    int8_t  qs[QK8_0];     // quants，表示长度为 QK8_0 的整型数组
} block_q8_0;
// 使用 static_assert 断言结构体 block_q8_0 的大小等于 ggml_fp16_t 类型的大小加上 QK8_0 的大小，如果不相等则输出错误信息
static_assert(sizeof(block_q8_0) == sizeof(ggml_fp16_t) + QK8_0, "wrong q8_0 block size/padding");

// 定义了一个名为 QK8_1 的宏，其值为 32
typedef struct {
    float d;               // delta，表示 delta 值
    float s;               // d * sum(qs[i])，表示 d 乘以 qs[i] 的和
    int8_t  qs[QK8_1];     // quants，表示长度为 QK8_1 的整型数组
} block_q8_1;
// 使用 static_assert 断言结构体 block_q8_1 的大小等于 2 个 float 类型的大小加上 QK8_1 的大小，如果不相等则输出错误信息
static_assert(sizeof(block_q8_1) == 2*sizeof(float) + QK8_1, "wrong q8_1 block size/padding");

//
// 超级块量化结构
//
// 根据不同的宏定义设置超级块大小和比例尺大小
#ifdef GGML_QKK_64
#define QK_K 64
#define K_SCALE_SIZE 4
#else
#define QK_K 256
#define K_SCALE_SIZE 12
#endif

// 2位量化
// 权重表示为 x = a * q + b
// 每个16个元素的16个块
// 实际上每个权重2.5625位
typedef struct {
    uint8_t scales[QK_K/16]; // 用4位量化的比例尺和最小值
    uint8_t qs[QK_K/4];      // 量化值
    ggml_fp16_t d;           // 用于量化比例尺的超级块比例
    ggml_fp16_t dmin;        // 用于量化最小值的超级块比例
} block_q2_K;
static_assert(sizeof(block_q2_K) == 2*sizeof(ggml_fp16_t) + QK_K/16 + QK_K/4, "wrong q2_K block size/padding");
// 定义了一个结构体 block_q3_K，用于表示3位量化的权重
// 权重表示为 x = a * q
// 16个包含16个元素的块
// 每个权重占用3.4375位
#ifdef GGML_QKK_64
// 如果定义了 GGML_QKK_64，则使用以下结构体
typedef struct {
    uint8_t hmask[QK_K/8];     // 用于存储量化的高位
    uint8_t qs[QK_K/4];        // 用于存储量化的低2位
    uint8_t scales[2];         // 用于存储尺度
    ggml_fp16_t d;             // 用于存储超级块的尺度
} block_q3_K;
// 检查结构体的大小是否符合预期
static_assert(sizeof(block_q3_K) == sizeof(ggml_fp16_t) + QK_K / 4 + QK_K / 8 + 2, "wrong q3_K block size/padding");
#else
// 如果未定义 GGML_QKK_64，则使用以下结构体
typedef struct {
    uint8_t hmask[QK_K/8];     // 用于存储量化的高位
    uint8_t qs[QK_K/4];        // 用于存储量化的低2位
    uint8_t scales[12];        // 用于存储尺度，使用6位进行量化
    ggml_fp16_t d;             // 用于存储超级块的尺度
} block_q3_K;
#endif
// 使用静态断言检查 block_q3_K 结构体的大小是否符合预期，如果不符合则输出错误信息
static_assert(sizeof(block_q3_K) == sizeof(ggml_fp16_t) + QK_K / 4 + QK_K / 8 + 12, "wrong q3_K block size/padding");

// 如果定义了 GGML_QKK_64，则使用 block_q4_K 结构体
// block_q4_K 结构体包含了 4 位量化的相关信息，包括超级块的缩放/最小值、4 位块的缩放/最小值和4位量化值
#ifdef GGML_QKK_64
typedef struct {
    ggml_fp16_t d[2];          // 超级块的缩放/最小值
    uint8_t scales[2];         // 4 位块的缩放/最小值
    uint8_t qs[QK_K/2];        // 4 位量化值
} block_q4_K;
// 使用静态断言检查 block_q4_K 结构体的大小是否符合预期，如果不符合则输出错误信息
static_assert(sizeof(block_q4_K) == 2*sizeof(ggml_fp16_t) + QK_K/2 + 2, "wrong q4_K block size/padding");
// 如果没有定义 GGML_QKK_64，则使用另一种结构体
#else
typedef struct {
    ggml_fp16_t d;             // 用于量化缩放的超级块缩放
    ggml_fp16_t dmin;          // 用于量化最小值的超级块缩放
    uint8_t scales[K_SCALE_SIZE]; // 使用 6 位量化的缩放和最小值
    uint8_t qs[QK_K/2];        // 4 位量化值
// 定义一个名为 block_q4_K 的结构体
} block_q4_K;
// 使用静态断言检查 block_q4_K 结构体的大小是否符合预期
static_assert(sizeof(block_q4_K) == 2*sizeof(ggml_fp16_t) + K_SCALE_SIZE + QK_K/2, "wrong q4_K block size/padding");
#endif

// 5位量化
// 每个包含32个元素的8个块
// 权重表示为 x = a * q + b
// 实际上每个权重占用5.5位
#ifdef GGML_QKK_64
// 定义一个名为 block_q5_K 的结构体
typedef struct {
    ggml_fp16_t d;               // 超级块的尺度
    int8_t  scales[QK_K/16];     // 8位块尺度
    uint8_t qh[QK_K/8];          // 量化值，高位
    uint8_t qs[QK_K/2];          // 量化值，低4位
} block_q5_K;
// 使用静态断言检查 block_q5_K 结构体的大小是否符合预期
static_assert(sizeof(block_q5_K) == sizeof(ggml_fp16_t) + QK_K/2 + QK_K/8 + QK_K/16, "wrong q5_K block size/padding");
#else
// 定义一个名为 block_q5_K 的结构体
typedef struct {
    ggml_fp16_t d;               // 用于量化尺度的超级块尺度
    ggml_fp16_t dmin;            // 用于量化最小值的超级块尺度
// 定义一个包含 K_SCALE_SIZE 大小的 uint8_t 数组，用于存储经过量化的比例和最小值
// 定义一个包含 QK_K/8 大小的 uint8_t 数组，用于存储量化后的权重，高位
// 定义一个包含 QK_K/2 大小的 uint8_t 数组，用于存储量化后的权重，低4位
} block_q5_K;
// 使用 static_assert 断言检查 block_q5_K 结构体的大小是否符合预期
static_assert(sizeof(block_q5_K) == 2*sizeof(ggml_fp16_t) + K_SCALE_SIZE + QK_K/2 + QK_K/8, "wrong q5_K block size/padding");
#endif

// 6位量化
// 权重表示为 x = a * q
// 16个包含16个元素的块
// 每个权重占用6.5625位
typedef struct {
    // 定义一个包含 QK_K/2 大小的 uint8_t 数组，用于存储量化后的权重，低4位
    uint8_t ql[QK_K/2];
    // 定义一个包含 QK_K/4 大小的 uint8_t 数组，用于存储量化后的权重，高2位
    uint8_t qh[QK_K/4];
    // 定义一个包含 QK_K/16 大小的 int8_t 数组，用于存储经过量化的比例，使用8位表示
    int8_t  scales[QK_K/16];
    // 定义一个 ggml_fp16_t 类型的变量，用于存储超级块的比例
    ggml_fp16_t d;
} block_q6_K;
// 使用 static_assert 断言检查 block_q6_K 结构体的大小是否符合预期
static_assert(sizeof(block_q6_K) == sizeof(ggml_fp16_t) + QK_K / 16 + 3*QK_K/4, "wrong q6_K block size/padding");

// 该结构体仅用于中间量化和点积运算
// 定义一个结构体，包含一个浮点数和两个数组，用于存储量化相关的数据
typedef struct {
    float   d;              // delta，浮点数
    int8_t  qs[QK_K];       // quants，长度为 QK_K 的 int8_t 数组
    int16_t bsums[QK_K/16]; // sum of quants in groups of 16，长度为 QK_K/16 的 int16_t 数组
} block_q8_K;
// 使用 static_assert 来确保 block_q8_K 结构体的大小和内存布局符合预期

// 量化函数的声明，用于将输入的浮点数数组进行量化处理，并存储到对应的结构体中
void quantize_row_q4_0_reference(const float * restrict x, block_q4_0 * restrict y, int k);
void quantize_row_q4_1_reference(const float * restrict x, block_q4_1 * restrict y, int k);
void quantize_row_q5_0_reference(const float * restrict x, block_q5_0 * restrict y, int k);
void quantize_row_q5_1_reference(const float * restrict x, block_q5_1 * restrict y, int k);
void quantize_row_q8_0_reference(const float * restrict x, block_q8_0 * restrict y, int k);
void quantize_row_q8_1_reference(const float * restrict x, block_q8_1 * restrict y, int k);

// 针对不同的量化精度和数组长度，声明对应的量化函数
void quantize_row_q2_K_reference(const float * restrict x, block_q2_K * restrict y, int k);
void quantize_row_q3_K_reference(const float * restrict x, block_q3_K * restrict y, int k);
void quantize_row_q4_K_reference(const float * restrict x, block_q4_K * restrict y, int k);
void quantize_row_q5_K_reference(const float * restrict x, block_q5_K * restrict y, int k);
// 定义了一系列的函数用于对输入的浮点数进行量化操作，将结果存储到指定的数据结构中
void quantize_row_q6_K_reference(const float * restrict x, block_q6_K * restrict y, int k);
void quantize_row_q8_K_reference(const float * restrict x, block_q8_K * restrict y, int k);

// 定义了一系列的函数用于对输入的浮点数进行不同精度的量化操作，将结果存储到指定的数据结构中
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

// 定义了一系列的函数用于对输入的特定数据结构进行反量化操作，将结果存储到浮点数数组中
void dequantize_row_q4_0(const block_q4_0 * restrict x, float * restrict y, int k);
void dequantize_row_q4_1(const block_q4_1 * restrict x, float * restrict y, int k);
// 对输入的 block_q5_0 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q5_0(const block_q5_0 * restrict x, float * restrict y, int k);

// 对输入的 block_q5_1 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q5_1(const block_q5_1 * restrict x, float * restrict y, int k);

// 对输入的 block_q8_0 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q8_0(const block_q8_0 * restrict x, float * restrict y, int k);

// 对输入的 block_q2_K 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q2_K(const block_q2_K * restrict x, float * restrict y, int k);

// 对输入的 block_q3_K 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q3_K(const block_q3_K * restrict x, float * restrict y, int k);

// 对输入的 block_q4_K 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q4_K(const block_q4_K * restrict x, float * restrict y, int k);

// 对输入的 block_q5_K 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q5_K(const block_q5_K * restrict x, float * restrict y, int k);

// 对输入的 block_q6_K 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q6_K(const block_q6_K * restrict x, float * restrict y, int k);

// 对输入的 block_q8_K 结构体进行反量化，并将结果存储到 float 类型的数组中
void dequantize_row_q8_K(const block_q8_K * restrict x, float * restrict y, int k);

// 对输入的两个向量进行加权和运算，结果存储到第三个向量中
void ggml_axpy_q4_0_q8_0(const int n, const void * restrict vx, const void * restrict vy, const void * restrict vz, int8_t alpha, ggml_fp16_t scale);

// 计算两个向量的点积，结果存储到 float 类型的变量中
void ggml_vec_dot_q4_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q4_1_q8_1(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q5_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q5_1_q8_1(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
void ggml_vec_dot_q8_0_q8_0(int n, float * restrict s, const void * restrict vx, const void * restrict vy);
# 这些是函数声明，声明了一系列的函数，每个函数都接受一个整数参数n和两个指针参数vx和vy，然后将结果存储在指针参数s所指向的位置。
# 函数ggml_vec_dot_q2_K_q8_K计算两个向量的点积，并将结果存储在指针参数s所指向的位置。
# 函数ggml_vec_dot_q3_K_q8_K计算两个向量的点积，并将结果存储在指针参数s所指向的位置。
# 函数ggml_vec_dot_q4_K_q8_K计算两个向量的点积，并将结果存储在指针参数s所指向的位置。
# 函数ggml_vec_dot_q5_K_q8_K计算两个向量的点积，并将结果存储在指针参数s所指向的位置。
# 函数ggml_vec_dot_q6_K_q8_K计算两个向量的点积，并将结果存储在指针参数s所指向的位置。
```