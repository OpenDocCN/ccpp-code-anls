# `PowerInfer\ggml-opencl.cpp`

```
#include "ggml-opencl.h"

#include <array>
#include <atomic>
#include <sstream>
#include <vector>
#include <limits>

#define CL_TARGET_OPENCL_VERSION 110
#include <clblast.h>

#include <stdlib.h>
#include <stdio.h>
#include <string.h>

#include "ggml.h"

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

#define CL_DMMV_LOCAL_SIZE 32

#ifndef K_QUANTS_PER_ITERATION
#define K_QUANTS_PER_ITERATION 1
#else
static_assert(K_QUANTS_PER_ITERATION == 1 || K_QUANTS_PER_ITERATION == 2, "K_QUANTS_PER_ITERATION must be 1 or 2");
#endif

#define MULTILINE_QUOTE(...) #__VA_ARGS__
static std::string program_source = MULTILINE_QUOTE(
// 定义不同数据类型的别名
typedef char int8_t;
typedef uchar uint8_t;
typedef short int16_t;
typedef ushort uint16_t;
typedef int int32_t;
typedef uint uint32_t;

// 定义不同数据结构体，使用 packed 属性进行内存对齐
struct __attribute__ ((packed)) block_q4_0
{
    half d;
    uint8_t qs[QK4_0 / 2];
};

struct __attribute__ ((packed)) block_q4_1
{
    half d;
    half m;
    uint8_t qs[QK4_1 / 2];
};

struct __attribute__ ((packed)) block_q5_0
{
    half d;
    uint32_t qh;
    uint8_t qs[QK5_0 / 2];
};

struct __attribute__ ((packed)) block_q5_1
{
    half d;
    half m;
    uint32_t qh;
    uint8_t qs[QK5_1 / 2];
};

struct __attribute__ ((packed)) block_q8_0
{
    half d;
    int8_t qs[QK8_0];
};

struct __attribute__((packed)) block_q2_K
{
    uint8_t scales[16];
    uint8_t qs[64];
    half d;
    half dmin;
};

struct __attribute__((packed)) block_q3_K
{
    uint8_t hmask[32];
    uint8_t qs[64];
    uint8_t scales[12];
    half d;
};

struct __attribute__((packed)) block_q4_K
{
    half d;
    half dmin;
    uint8_t scales[12];
    uint8_t qs[128];
};

struct __attribute__((packed)) block_q5_K
{
    half d;
    half dmin;
    uint8_t scales[12];
    uint8_t qh[32];
    uint8_t qs[128];
};

struct __attribute__((packed)) block_q6_K
{
    uint8_t ql[128];
    uint8_t qh[64];
    int8_t scales[16];
    half d;
};
# 将 FP16 类型的数组转换为 FP32 类型的数组
__kernel void convert_fp16_to_fp32(__global half* x, __global float* y) {
    # 获取全局唯一的 ID
    const uint i = get_global_id(0);
    # 从 x 数组中加载一个 FP16 类型的值，转换为 FP32 类型后存入 y 数组
    y[i] = vload_half(0, &x[i]);
}

# 对 Q4_0 类型的数据进行反量化
void dequantize_q4_0(__global const struct block_q4_0* x, const int ib, const int iqs, float* v0, float* v1) {
    # 从 x 数组中加载一个 FP16 类型的值
    const float d = vload_half(0, &x[ib].d);
    # 从 x 数组中加载一个 uint8_t 类型的值
    const uint8_t vui = x[ib].qs[iqs];
    # 对 vui 进行位运算，得到两个 int8_t 类型的值
    const int8_t vi0 = vui & 0xF;
    const int8_t vi1 = vui >> 4;
    # 根据 vi0、vi1 和 d 计算 v0、v1 的值
    *v0 = (vi0 - 8)*d;
    *v1 = (vi1 - 8)*d;
}

# 对 Q4_1 类型的数据进行反量化
void dequantize_q4_1(__global const struct block_q4_1* x, const int ib, const int iqs, float* v0, float* v1) {
    # 从 x 数组中加载两个 FP16 类型的值
    const float d = vload_half(0, &x[ib].d);
    const float m = vload_half(0, &x[ib].m);
    # 从 x 数组中加载一个 uint8_t 类型的值
    const uint8_t vui = x[ib].qs[iqs];
    # 对 vui 进行位运算，得到两个 int8_t 类型的值
    const int8_t vi0 = vui & 0xF;
    const int8_t vi1 = vui >> 4;
    # 根据 vi0、vi1、d 和 m 计算 v0、v1 的值
    *v0 = vi0*d + m;
    *v1 = vi1*d + m;
}

# 对 Q5_0 类型的数据进行反量化
void dequantize_q5_0(__global const struct block_q5_0* x, const int ib, const int iqs, float* v0, float* v1) {
    # 从 x 数组中加载一个 FP16 类型的值
    const float d = vload_half(0, &x[ib].d);
    # 从 x 数组中加载一个 uint32_t 类型的值
    uint32_t qh = x[ib].qh;
    # 对 qh 进行位运算，得到两个 uint8_t 类型的值
    const uint8_t xh_0 = ((qh >> (iqs +  0)) << 4) & 0x10;
    const uint8_t xh_1 = ((qh >> (iqs + 12))     ) & 0x10;
    # 对 x[ib].qs[iqs] 和 xh_0 进行运算，得到两个 int32_t 类型的值
    const int32_t x0 = ((x[ib].qs[iqs] & 0xf) | xh_0) - 16;
    const int32_t x1 = ((x[ib].qs[iqs] >>  4) | xh_1) - 16;
    # 根据 x0、x1 和 d 计算 v0、v1 的值
    *v0 = x0*d;
    *v1 = x1*d;
}

# 对 Q5_1 类型的数据进行反量化
void dequantize_q5_1(__global const struct block_q5_1* x, const int ib, const int iqs, float* v0, float* v1) {
    # 从 x 数组中加载两个 FP16 类型的值
    const float d = vload_half(0, &x[ib].d);
    const float m = vload_half(0, &x[ib].m);
    # 从 x 数组中加载一个 uint32_t 类型的值
    uint32_t qh = x[ib].qh;
    # 对 qh 进行位运算，得到两个 uint8_t 类型的值
    const uint8_t xh_0 = ((qh >> (iqs +  0)) << 4) & 0x10;
    const uint8_t xh_1 = ((qh >> (iqs + 12))     ) & 0x10;
    # 对 x[ib].qs[iqs] 和 xh_0 进行运算，得到两个 int32_t 类型的值
    const int32_t x0 = ((x[ib].qs[iqs] & 0xf) | xh_0);
    const int32_t x1 = ((x[ib].qs[iqs] >>  4) | xh_1);
    # 根据 x0、x1、d 和 m 计算 v0、v1 的值
    *v0 = x0*d + m;
    *v1 = x1*d + m;
}

# 对 Q8_0 类型的数据进行反量化
void dequantize_q8_0(__global const struct block_q8_0* x, const int ib, const int iqs, float* v0, float* v1) {
    # 从 x 数组中加载一个 FP16 类型的值
    const float d = vload_half(0, &x[ib].d);
    # 从 x 数组中加载一个 int8_t 类型的值
    const int8_t vi0 = x[ib].qs[iqs + 0];
}
    # 从数组 x 中取出索引为 ib 的元素，再取出其内部索引为 iqs+1 的元素，赋值给常量 vi1
    const int8_t vi1 = x[ib].qs[iqs + 1];

    # 将 vi0 乘以常量 d 的结果赋值给指针 v0 指向的位置
    *v0 = vi0*d;
    # 将 vi1 乘以常量 d 的结果赋值给指针 v1 指向的位置
    *v1 = vi1*d;
// 定义一个函数，将输入的 half 类型数据转换为 float 类型数据
void convert_f16(__global half* x, const int ib, const int iqs, float* v0, float* v1){
    // 从 x 数组中读取第 ib 个元素，并将其转换为 float 类型，存储到 v0 指针指向的位置
    *v0 = vload_half(0, &x[ib + 0]);
    // 从 x 数组中读取第 ib+1 个元素，并将其转换为 float 类型，存储到 v1 指针指向的位置
    *v1 = vload_half(0, &x[ib + 1]);
}

// 定义一个内联函数，用于计算 scale 和 min 值
static std::string k_quants_source = MULTILINE_QUOTE(
inline void get_scale_min_k4(int j, const __global uint8_t *q, uint8_t *d, uint8_t *m)
{
    // 如果 j 小于 4，则执行以下操作
    if (j < 4)
    {
        // 将 q 数组中的第 j 个元素的低 6 位存储到 d 指针指向的位置
        *d = q[j] & 63;
        // 将 q 数组中的第 j+4 个元素的低 6 位存储到 m 指针指向的位置
        *m = q[j + 4] & 63;
    }
    // 如果 j 大于等于 4，则执行以下操作
    else
    {
        // 将 q 数组中的第 j+4 个元素的低 4 位和第 j-4 个元素的高 2 位合并后存储到 d 指针指向的位置
        *d = (q[j + 4] & 0xF) | ((q[j - 4] >> 6) << 4);
        // 将 q 数组中的第 j+4 个元素的高 4 位和第 j+0 个元素的高 2 位合并后存储到 m 指针指向的位置
        *m = (q[j + 4] >> 4) | ((q[j - 0] >> 6) << 4);
    }
}

// 定义一个内核函数，用于对输入的数据进行反量化处理
__kernel void dequantize_block_q2_K(__global const struct block_q2_K *x, __global float *yy)
{
    // 获取当前工作组的 ID，并加上全局偏移量，得到 i 的值
    const int i = get_group_id(0) + get_global_offset(0);
    // 获取当前工作项的 ID
    const int tid = get_local_id(0);
    // 计算 n 的值
    const int n = tid / 32;
    // 计算 l 的值
    const int l = tid - 32 * n;
    // 计算 is 的值
    const int is = 8 * n + l / 16;

    // 从 x[i].qs 数组中读取第 32*n+l 个元素，存储到 q 变量中
    const uint8_t q = x[i].qs[32 * n + l];
    // 计算 y 数组的起始位置
    __global float *y = yy + get_group_id(0) * QK_K + 128 * n;

    // 从 x[i].d 中读取 half 类型数据，存储到 dall 变量中
    const float dall = vload_half(0, &x[i].d);
    // 从 x[i].dmin 中读取 half 类型数据，存储到 dmin 变量中
    const float dmin = vload_half(0, &x[i].dmin);

    // 根据一定规则计算 y 数组中的值
    y[l + 0] = dall * (x[i].scales[is + 0] & 0xF) * ((q >> 0) & 3) - dmin * (x[i].scales[is + 0] >> 4);
    y[l + 32] = dall * (x[i].scales[is + 2] & 0xF) * ((q >> 2) & 3) - dmin * (x[i].scales[is + 2] >> 4);
    y[l + 64] = dall * (x[i].scales[is + 4] & 0xF) * ((q >> 4) & 3) - dmin * (x[i].scales[is + 4] >> 4);
    y[l + 96] = dall * (x[i].scales[is + 6] & 0xF) * ((q >> 6) & 3) - dmin * (x[i].scales[is + 6] >> 4);
}

// 定义一个内核函数，用于对输入的数据进行反量化处理
__kernel void dequantize_block_q3_K(__global const struct block_q3_K *x, __global float *yy)
{
    // 计算 r 的值
    int r = get_local_id(0) / 4;
    // 计算 i 的值
    int i = get_group_id(0) + get_global_offset(0);
    // 计算 tid 的值
    int tid = r / 2;
    // 计算 is0 的值
    int is0 = r % 2;
    // 计算 l0 的值
    int l0 = 16 * is0 + 4 * (get_local_id(0) % 4);
    // 计算 n 的值
    int n = tid / 4;
    // 计算 j 的值
    int j = tid - 4 * n;

    // 计算 m 的值
    uint8_t m = 1 << (4 * n + j);
    // 计算 is 的值
    int is = 8 * n + 2 * j + is0;
    // 计算 shift 的值
    int shift = 2 * j;
}
    // 根据条件判断选择不同的操作数
    int8_t us = is < 4 ? (x[i].scales[is - 0] & 0xF) | (((x[i].scales[is + 8] >> 0) & 3) << 4)
              : is < 8 ? (x[i].scales[is - 0] & 0xF) | (((x[i].scales[is + 4] >> 2) & 3) << 4)
              : is < 12  ? (x[i].scales[is - 8] >> 4) | (((x[i].scales[is + 0] >> 4) & 3) << 4)
              : (x[i].scales[is - 8] >> 4) | (((x[i].scales[is - 4] >> 6) & 3) << 4);
    // 从内存中加载半精度浮点数
    float d_all = vload_half(0, &x[i].d);
    // 计算 dl
    float dl = d_all * (us - 32);

    // 定义指向 yy 数组的指针
    __global float *y = yy + get_group_id(0) * QK_K + 128 * n + 32 * j;
    // 定义指向 x[i].qs 数组的指针
    const __global uint8_t *q = x[i].qs + 32 * n;
    // 定义指向 x[i].hmask 数组的指针
    const __global uint8_t *hm = x[i].hmask;

    // 循环遍历 l0 到 l0 + 4
    for (int l = l0; l < l0 + 4; ++l)
        // 计算 y[l] 的值
        y[l] = dl * ((int8_t)((q[l] >> shift) & 3) - ((hm[l] & m) ? 0 : 4));
# 定义 OpenCL 内核函数，用于将 Q4 类型的块反量化为浮点数
__kernel void dequantize_block_q4_K(__global const struct block_q4_K *x, __global float *yy)
{
    # 获取当前工作组的 ID，并加上全局偏移量，得到当前处理的块索引
    const int i = get_group_id(0) + get_global_offset(0);
    # 获取当前工作组中的本地 ID
    const int tid = get_local_id(0);
    # 计算当前本地 ID 对应的左右索引
    const int il = tid / 8;
    const int ir = tid % 8;
    # 计算当前本地 ID 对应的起始索引
    const int is = 2 * il;
    const int n = 4;

    # 计算当前工作组对应的输出数组指针
    __global float *y = yy + get_group_id(0) * QK_K + 64 * il + n * ir;

    # 从输入结构体中加载 d 和 dmin 字段的值
    const float dall = vload_half(0, &x[i].d);
    const float dmin = vload_half(0, &x[i].dmin);

    # 从输入结构体中加载 qs 字段的值
    __global const uint8_t *q = x[i].qs + 32 * il + n * ir;

    # 定义变量 sc 和 m，调用 get_scale_min_k4 函数获取对应的值
    uint8_t sc, m;
    get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
    float d1 = dall * sc;
    float m1 = dmin * m;
    get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
    float d2 = dall * sc;
    float m2 = dmin * m;
    # 循环处理 n 个元素
    for (int l = 0; l < n; ++l)
    {
        y[l + 0] = d1 * (q[l] & 0xF) - m1;
        y[l + 32] = d2 * (q[l] >> 4) - m2;
    }
}

# 定义 OpenCL 内核函数，用于将 Q5 类型的块反量化为浮点数
__kernel void dequantize_block_q5_K(__global const struct block_q5_K *x, __global float *yy)
{
    # 获取当前工作组的 ID，并加上全局偏移量，得到当前处理的块索引
    const int i = get_group_id(0) + get_global_offset(0);
    # 获取当前工作组中的本地 ID
    const int tid = get_local_id(0);
    # 计算当前本地 ID 对应的左右索引
    const int il = tid / 16;
    const int ir = tid % 16;
    # 计算当前本地 ID 对应的起始索引
    const int is = 2 * il;

    # 计算当前工作组对应的输出数组指针
    __global float *y = yy + get_group_id(0) * QK_K + 64 * il + 2 * ir;

    # 从输入结构体中加载 d 和 dmin 字段的值
    const float dall = vload_half(0, &x[i].d);
    const float dmin = vload_half(0, &x[i].dmin);

    # 从输入结构体中加载 qs 和 qh 字段的值
    __global const uint8_t *ql = x[i].qs + 32 * il + 2 * ir;
    __global const uint8_t *qh = x[i].qh + 2 * ir;

    # 定义变量 sc 和 m，调用 get_scale_min_k4 函数获取对应的值
    uint8_t sc, m;
    get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
    const float d1 = dall * sc;
    const float m1 = dmin * m;
    get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
    const float d2 = dall * sc;
    const float m2 = dmin * m;

    # 计算 hm 的值
    uint8_t hm = 1 << (2 * il);
    # 计算输出数组中的值
    y[0] = d1 * ((ql[0] & 0xF) + (qh[0] & hm ? 16 : 0)) - m1;
    y[1] = d1 * ((ql[1] & 0xF) + (qh[1] & hm ? 16 : 0)) - m1;
    hm <<= 1;
    y[32] = d2 * ((ql[0] >> 4) + (qh[0] & hm ? 16 : 0)) - m2;
    y[33] = d2 * ((ql[1] >> 4) + (qh[1] & hm ? 16 : 0)) - m2;
}
# 定义一个名为 dequantize_block_q6_K 的内核函数，接受一个结构体指针和一个全局浮点数指针作为参数
__kernel void dequantize_block_q6_K(__global const struct block_q6_K *x, __global float *yy)
{
    # 获取当前工作组的 ID，并加上全局偏移量，得到索引 i
    const int i = get_group_id(0) + get_global_offset(0);
    # 获取当前工作组的本地 ID，作为线程 ID
    const int tid = get_local_id(0);
    # 计算 ip，il，is 的值
    const int ip = tid / 32;
    const int il = tid - 32 * ip;
    const int is = 8 * ip + il / 16;

    # 计算 y 的地址
    __global float *y = yy + get_group_id(0) * QK_K + 128 * ip + il;

    # 从 x[i] 中加载 d 的值
    const float d = vload_half(0, &x[i].d);

    # 加载 ql，qh，sc 的值
    __global const uint8_t *ql = x[i].ql + 64 * ip + il;
    const uint8_t qh = x[i].qh[32 * ip + il];
    __global const int8_t *sc = x[i].scales + is;

    # 计算 y 的值
    y[0] = d * sc[0] * ((int8_t)((ql[0] & 0xF) | (((qh >> 0) & 3) << 4)) - 32);
    y[32] = d * sc[2] * ((int8_t)((ql[32] & 0xF) | (((qh >> 2) & 3) << 4)) - 32);
    y[64] = d * sc[4] * ((int8_t)((ql[0] >> 4) | (((qh >> 4) & 3) << 4)) - 32);
    y[96] = d * sc[6] * ((int8_t)((ql[32] >> 4) | (((qh >> 6) & 3) << 4)) - 32);
}

# 定义一个名为 dequantize_mul_mat_vec_q2_K 的内核函数，接受四个结构体指针和一个整数作为参数
__kernel void dequantize_mul_mat_vec_q2_K(__global const struct block_q2_K * xx, __local float* tmp, __global float* yy, __global float* dst, const int ncols) {

    # 获取当前工作组的 ID，作为行索引
    const int row = get_group_id(0);

    # 计算每行的块数，以及当前行的起始块索引
    const int num_blocks_per_row = ncols / QK_K;
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    # 计算当前线程的本地 ID 和块索引
    const int tid = get_local_id(0)/K_QUANTS_PER_ITERATION;  // 0...31 or 0...15
    const int ix  = get_local_id(0)%K_QUANTS_PER_ITERATION;  // 0 or 0,1

    # 计算步长和块索引
    const int step = 16/K_QUANTS_PER_ITERATION;
    const int im = tid/step;                             // 0 or 1. 0 computes 0..., 1 computes 128...
    const int in = tid - step*im;                        // 0...15 or 0...7

    # 计算偏移量
    const int l0 = K_QUANTS_PER_ITERATION*in;            // 0...15 or 0...14 in steps of 2
    const int q_offset = 32*im + l0;
    const int s_offset = 8*im;
    const int y_offset = 128*im + l0;

    # 初始化临时数组
    tmp[16 * ix + tid] = 0;

    # 定义辅���数组和指针
    uint32_t aux[4];
    const uint8_t * d = (const uint8_t *)aux;
    const uint8_t * m = (const uint8_t *)(aux + 2);
}
    // 遍历每一行的数据块
    for (int i = ix; i < num_blocks_per_row; i += K_QUANTS_PER_ITERATION) {

        // 定义指向 yy 数组的指针，偏移量为 i * QK_K + y_offset
        __global const float   * y = yy + i * QK_K + y_offset;
        // 定义指向 x[i].qs 数组的指针，偏移量为 q_offset
        __global const uint8_t * q = x[i].qs + q_offset;

        // 从 x[i].d 中加载半精度浮点数到 dall
        const float dall = vload_half(0, &x[i].d);
        // 从 x[i].dmin 中加载半精度浮点数到 dmin
        const float dmin = vload_half(0, &x[i].dmin);

        // 定义指向 x[i].scales 数组的指针，偏移量为 s_offset
        __global const uint32_t * a = (__global const uint32_t *)(x[i].scales + s_offset);
        // 将 a 数组中的值与 0x0f0f0f0f 进行按位与操作，存储到 aux 数组中
        aux[0] = a[0] & 0x0f0f0f0f;
        aux[1] = a[1] & 0x0f0f0f0f;
        aux[2] = (a[0] >> 4) & 0x0f0f0f0f;
        aux[3] = (a[1] >> 4) & 0x0f0f0f0f;

        // 初始化 sum1 和 sum2
        float sum1 = 0, sum2 = 0;
        // 遍历 K_QUANTS_PER_ITERATION 次
        for (int l = 0; l < K_QUANTS_PER_ITERATION; ++l) {
            // 计算 sum1
            sum1 += y[l+ 0] * d[0] * ((q[l+ 0] >> 0) & 3)
                  + y[l+32] * d[2] * ((q[l+ 0] >> 2) & 3)
                  + y[l+64] * d[4] * ((q[l+ 0] >> 4) & 3)
                  + y[l+96] * d[6] * ((q[l+ 0] >> 6) & 3)
                  + y[l+16] * d[1] * ((q[l+16] >> 0) & 3)
                  + y[l+48] * d[3] * ((q[l+16] >> 2) & 3)
                  + y[l+80] * d[5] * ((q[l+16] >> 4) & 3)
                  +y[l+112] * d[7] * ((q[l+16] >> 6) & 3);
            // 计算 sum2
            sum2 += y[l+ 0] * m[0] + y[l+32] * m[2] + y[l+64] * m[4] + y[ l+96] * m[6]
                  + y[l+16] * m[1] + y[l+48] * m[3] + y[l+80] * m[5] + y[l+112] * m[7];

        }
        // 将计算结果存储到 tmp 数组中
        tmp[16 * ix + tid] += dall * sum1 - dmin * sum2;

    }

    // 汇总部分和并写回结果
    barrier(CLK_LOCAL_MEM_FENCE);
    for (int s=16; s>0; s>>=1) {
        if (tid < s) {
            tmp[tid] += tmp[tid + s];
        }
        barrier(CLK_LOCAL_MEM_FENCE);
    }
    if (tid == 0) {
        // 将最终结果写回 dst 数组中
        dst[row] = tmp[0];
    }
# 定义一个 OpenCL 内核函数，用于对矩阵向量乘法进行反量化操作
__kernel void dequantize_mul_mat_vec_q3_K(__global const struct block_q3_K * xx, __local float* tmp, __global float* yy, __global float* dst, const int ncols) {
    # 定义两个 16 位整数常量，用于掩码操作
    const uint16_t kmask1 = 0x0303;
    const uint16_t kmask2 = 0x0f0f;

    # 获取当前工作组的 ID，用于确定行数
    const int row = get_group_id(0);

    # 计算每行的块数
    const int num_blocks_per_row = ncols / QK_K;
    # 计算当前块的索引
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    # 获取当前块的指针
    __global const struct block_q3_K * x = xx + ib0;

    # 计算本地 ID 对应的线程 ID，用于确定迭代次数
    const int tid = get_local_id(0)/K_QUANTS_PER_ITERATION;  # 0...31 or 0...16
    # 计算本地 ID 对应的索引，用于确定迭代次数
    const int ix  = get_local_id(0)%K_QUANTS_PER_ITERATION;  # 0 or 0,1

    # 定义迭代次数
    const int n  = K_QUANTS_PER_ITERATION;               # iterations in the inner loop
    # 计算步长
    const int step = 16/K_QUANTS_PER_ITERATION;
    # 计算 m 的值
    const int im = tid/step;                             # 0 or 1. 0 computes 0..., 1 computes 128...
    # 计算 n 的值
    const int in = tid - step*im;                        # 0....15 or 0...7

    # 计算 m 的位移
    const uint8_t m = 1 << (4*im);

    # 计算 l0 的值
    const int l0 = n*in;                                 # 0...15 or 0...14 in steps of 2
    # 计算 q_offset 的值
    const int q_offset =  32*im + l0;
    # 计算 y_offset 的值
    const int y_offset = 128*im + l0;

    # 定义一个 16 位无符号整数数组
    uint16_t utmp[4];
    # 定义一个指向 utmp 的 int8_t 类型指针
    const int8_t * s = (const int8_t *)utmp;

    # 计算 s_shift 的值
    const uint16_t s_shift = 4*im;

    # 将 tmp 数组对应位置的值初始化为 0
    tmp[16 * ix + tid] = 0;
    // 遍历每一行的数据块
    for (int i = ix; i < num_blocks_per_row; i += K_QUANTS_PER_ITERATION) {

        // 定义指向 yy 数组的指针，偏移量为 i * QK_K + y_offset
        __global const float   * y  = yy + i * QK_K + y_offset;
        // 定义指向 x[i].qs 数组的指针，偏移量为 q_offset
        __global const uint8_t * q = x[i].qs + q_offset;
        // 定义指向 x[i].hmask 数组的指针，偏移量为 l0
        __global const uint8_t * h = x[i].hmask + l0;

        // 定义指向 x[i].scales 数组的指针，转换为 uint16_t 类型后取值，存入 utmp 数组
        __global const uint16_t * a = (__global const uint16_t *)x[i].scales;
        utmp[0] = ((a[0] >> s_shift) & kmask2) | (((a[4] >> (s_shift + 0)) & kmask1) << 4);
        utmp[1] = ((a[1] >> s_shift) & kmask2) | (((a[5] >> (s_shift + 0)) & kmask1) << 4);
        utmp[2] = ((a[2] >> s_shift) & kmask2) | (((a[4] >> (s_shift + 2)) & kmask1) << 4);
        utmp[3] = ((a[3] >> s_shift) & kmask2) | (((a[5] >> (s_shift + 2)) & kmask1) << 4);

        // 从 x[i].d 中加载半精度浮点数，存入 d 变量
        const float d = vload_half(0, &x[i].d);

        // 初始化 sum 变量
        float sum = 0;
        // 遍历 n 次
        for (int l = 0; l < n; ++l) {
            // 计算 sum 的值
            sum += y[l+ 0] * (s[0] - 32) * (((q[l] >> 0) & 3) - (h[l] & (m << 0) ? 0 : 4))
                 + y[l+32] * (s[2] - 32) * (((q[l] >> 2) & 3) - (h[l] & (m << 1) ? 0 : 4))
                 + y[l+64] * (s[4] - 32) * (((q[l] >> 4) & 3) - (h[l] & (m << 2) ? 0 : 4))
                 + y[l+96] * (s[6] - 32) * (((q[l] >> 6) & 3) - (h[l] & (m << 3) ? 0 : 4));
            sum += y[l+16] * (s[1] - 32) * (((q[l+16] >> 0) & 3) - (h[l+16] & (m << 0) ? 0 : 4))
                 + y[l+48] * (s[3] - 32) * (((q[l+16] >> 2) & 3) - (h[l+16] & (m << 1) ? 0 : 4))
                 + y[l+80] * (s[5] - 32) * (((q[l+16] >> 4) & 3) - (h[l+16] & (m << 2) ? 0 : 4))
                + y[l+112] * (s[7] - 32) * (((q[l+16] >> 6) & 3) - (h[l+16] & (m << 3) ? 0 : 4));
        }
        // 将 d 与 sum 相乘后加到 tmp 数组的对应位置上
        tmp[16 * ix + tid] += d * sum;

    }

    // 局部内存屏障，等待所有线程完成
    barrier(CLK_LOCAL_MEM_FENCE);
    // 对 tmp 数组进行归约求和
    for (int s=16; s>0; s>>=1) {
        if (tid < s) {
            tmp[tid] += tmp[tid + s];
        }
        // 局部内存屏障，等待所有线程完成
        barrier(CLK_LOCAL_MEM_FENCE);
    }
    // 如果是线程块中的第一个线程，则将 tmp[0] 的值写入 dst 数组对应位置
    if (tid == 0) {
        dst[row] = tmp[0];
    }
}
// 结束函数定义

__kernel void dequantize_mul_mat_vec_q4_K(__global const struct block_q4_K * xx, __local float* tmp, __global float* yy, __global float* dst, const int ncols) {
// 定义 OpenCL 内核函数，接受输入参数 xx、tmp、yy、dst 和 ncols

    //to rename it later, just to test now
    // 为了以后重命名，现在只是测试
    const uint16_t kmask1 = 0x3f3f;
    const uint16_t kmask2 = 0x0f0f;
    const uint16_t kmask3 = 0xc0c0;

    const int row = get_group_id(0);
    // 获取当前工作组的 ID
    const int num_blocks_per_row = ncols / QK_K;
    // 计算每行的块数
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);
    // 计算当前行的起始块索引

    const int tid = get_local_id(0)/K_QUANTS_PER_ITERATION;  // 0...15
    // 获取本地工作组中的线程 ID，用于计算 tid
    const int ix  = get_local_id(0)%K_QUANTS_PER_ITERATION;
    // 计算 ix

    const int step = 8/K_QUANTS_PER_ITERATION;
    // 计算步长

    const int il  = tid/step;     // 0...3
    // 计算 il
    const int ir  = tid - step*il;// 0...3
    // 计算 ir
    const int n   = 2*K_QUANTS_PER_ITERATION;
    // 计算 n

    const int im = il/2;  // 0 or 1. 0 computes 0,32 + 128,160, 1 computes 64,96 + 192,224
    // 计算 im
    const int in = il%2;
    // 计算 in

    const int l0 = n*(2*ir + in);
    // 计算 l0
    const int q_offset = 32*im + l0;
    // 计算 q_offset
    const int y_offset = 64*im + l0;
    // 计算 y_offset

    uint16_t aux[4];
    // 定义长度为 4 的 uint16_t 数组 aux
    const uint8_t * sc = (const uint8_t *)aux;
    // 定义指向 aux 的 uint8_t 指针 sc

    __global const struct block_q4_K * x = xx + ib0;
    // 定义指向 xx + ib0 的 __global const struct block_q4_K 指针 x

    tmp[16 * ix + tid] = 0;
    // 将 tmp 中指定位置的值设为 0
    // 对每一行的数据块进行处理，每次处理 K_QUANTS_PER_ITERATION 个数据块
    for (int i = ix; i < num_blocks_per_row; i += K_QUANTS_PER_ITERATION) {

        // 获取第一个量化表和第二个量化表的指针
        __global const uint8_t * q1 = x[i].qs + q_offset;
        __global const uint8_t * q2 = q1 + 64;
        // 获取第一个输出数据和第二个输出数据的指针
        __global const float   * y1 = yy + i*QK_K + y_offset;
        __global const float   * y2 = y1 + 128;

        // 读取 x[i].d 和 x[i].dmin 的值
        const float dall = vload_half(0, &x[i].d);
        const float dmin = vload_half(0, &x[i].dmin);

        // 获取 x[i].scales 中的值，并进行位运算
        __global const uint16_t * a = (__global const uint16_t *)x[i].scales;
        aux[0] = a[im+0] & kmask1;
        aux[1] = a[im+2] & kmask1;
        aux[2] = ((a[im+4] >> 0) & kmask2) | ((a[im+0] & kmask3) >> 2);
        aux[3] = ((a[im+4] >> 4) & kmask2) | ((a[im+2] & kmask3) >> 2);

        // 初始化 s 和 smin 的值
        float4 s = (float4)(0.f);
        float smin = 0;
        // 对每个数据块进行处理
        for (int l = 0; l < n; ++l) {
            // 计算 s 和 smin 的值
            s.x += y1[l] * (q1[l] & 0xF); s.y += y1[l+32] * (q1[l] >> 4);
            s.z += y2[l] * (q2[l] & 0xF); s.w += y2[l+32] * (q2[l] >> 4);
            smin += y1[l] * sc[2] + y1[l+32] * sc[3] + y2[l] * sc[6] + y2[l+32] * sc[7];
        }
        // 计算 tmp 数组的值
        tmp[16 * ix + tid] += dall * (s.x * sc[0] + s.y * sc[1] + s.z * sc[4] + s.w * sc[5]) - dmin * smin;

    }

    // 对局部和进行求和，并将结果写回
    barrier(CLK_LOCAL_MEM_FENCE);
    for (int s=16; s>0; s>>=1) {
        if (tid < s) {
            tmp[tid] += tmp[tid + s];
        }
        barrier(CLK_LOCAL_MEM_FENCE);
    }
    if (tid == 0) {
        dst[row] = tmp[0];
    }
# 定义一个 OpenCL 内核函数，用于对 q5_K 类型的矩阵向量进行去量化和乘法操作
__kernel void dequantize_mul_mat_vec_q5_K(__global const struct block_q5_K * xx, __local float* tmp, __global float* yy, __global float* dst, const int ncols) {

    # 定义三个 16 位整数常量，用于掩码操作
    const uint16_t kmask1 = 0x3f3f;
    const uint16_t kmask2 = 0x0f0f;
    const uint16_t kmask3 = 0xc0c0;

    # 获取当前工作组的 ID，即矩阵的行索引
    const int row = get_group_id(0);
    # 计算每行包含的块数
    const int num_blocks_per_row = ncols / QK_K;
    # 计算当前块的索引
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    # 获取当前工作项在工作组中的局部 ID，并计算出对应的索引
    const int tid = get_local_id(0)/2;  // 0...15
    const int ix  = get_local_id(0)%2;

    # 根据 tid 计算出 il 和 ir 的值
    const int il  = tid/4;     # 0...3
    const int ir  = tid - 4*il;# 0...3
    const int n   = 2;

    # 根据 il 计算出 im 和 in 的值
    const int im = il/2;  # 0 or 1. 0 computes 0,32 + 128,160, 1 computes 64,96 + 192,224
    const int in = il%2;

    # 根据 il 和 ir 计算出 l0 的值
    const int l0 = n*(2*ir + in);
    # 计算 q_offset 和 y_offset 的值
    const int q_offset = 32*im + l0;
    const int y_offset = 64*im + l0;

    # 定义两个 8 位整数常量，用于位移操作
    const uint8_t hm1  = 1 << (2*im);
    const uint8_t hm2  = hm1 << 4;

    # 定义一个长度为 4 的 uint16_t 数组，用于存储临时数据
    uint16_t aux[4];
    # 定义一个指向 aux 的 uint8_t 类型指针
    const uint8_t * sc = (const uint8_t *)aux;

    # 定义一个指向 xx + ib0 的指针，用于获取输入数据
    __global const struct block_q5_K * x = xx + ib0;

    # 将 tmp 数组的一部分初始化为 0
    tmp[16 * ix + tid] = 0;
    // 对每一行的块进行循环，每次增加2
    for (int i = ix; i < num_blocks_per_row; i += 2) {

        // 获取指向第一个查询序列的指针
        __global const uint8_t * ql1 = x[i].qs + q_offset;
        // 获取指向第二个查询序列的指针
        __global const uint8_t * ql2 = ql1 + 64;
        // 获取指向高位查询序列的指针
        __global const uint8_t * qh  = x[i].qh + l0;
        // 获取指向第一个查询序列对应的权重向量的指针
        __global const float   * y1  = yy + i*QK_K + y_offset;
        // 获取指向第二个查询序列对应的权重向量的指针
        __global const float   * y2  = y1 + 128;

        // 从内存中加载单精度浮点数到寄存器中
        const float dall = vload_half(0, &x[i].d);
        // 从内存中加载单精度浮点数到寄存器中
        const float dmin = vload_half(0, &x[i].dmin);

        // 获取指向缩放因子数组的指针
        __global const uint16_t * a = (__global const uint16_t *)x[i].scales;
        // 计算辅助变量
        aux[0] = a[im+0] & kmask1;
        aux[1] = a[im+2] & kmask1;
        aux[2] = ((a[im+4] >> 0) & kmask2) | ((a[im+0] & kmask3) >> 2);
        aux[3] = ((a[im+4] >> 4) & kmask2) | ((a[im+2] & kmask3) >> 2);

        // 初始化四个单精度浮点数的和
        float4 sum = (float4)(0.f);
        // 初始化最小值
        float smin = 0;
        // 对每个查询序列进行循环
        for (int l = 0; l < n; ++l) {
            // 计算第一个查询序列的加权和
            sum.x += y1[l+ 0] * ((ql1[l+ 0] & 0xF) + (qh[l+ 0] & (hm1 << 0) ? 16 : 0))
                   + y1[l+16] * ((ql1[l+16] & 0xF) + (qh[l+16] & (hm1 << 0) ? 16 : 0));
            // 计算第一个查询序列的加权和
            sum.y += y1[l+32] * ((ql1[l+ 0] >>  4) + (qh[l+ 0] & (hm1 << 1) ? 16 : 0))
                   + y1[l+48] * ((ql1[l+16] >>  4) + (qh[l+16] & (hm1 << 1) ? 16 : 0));
            // 计算第二个查询序列的加权和
            sum.z += y2[l+ 0] * ((ql2[l+ 0] & 0xF) + (qh[l+ 0] & (hm2 << 0) ? 16 : 0))
                   + y2[l+16] * ((ql2[l+16] & 0xF) + (qh[l+16] & (hm2 << 0) ? 16 : 0));
            // 计算第二个查询序列的加权和
            sum.w += y2[l+32] * ((ql2[l+ 0] >>  4) + (qh[l+ 0] & (hm2 << 1) ? 16 : 0))
                   + y2[l+48] * ((ql2[l+16] >>  4) + (qh[l+16] & (hm2 << 1) ? 16 : 0));
            // 计算最小值
            smin += (y1[l] + y1[l+16]) * sc[2] + (y1[l+32] + y1[l+48]) * sc[3]
                  + (y2[l] + y2[l+16]) * sc[6] + (y2[l+32] + y2[l+48]) * sc[7];
        }
        // 更新临时数组
        tmp[16 * ix + tid] += dall * (sum.x * sc[0] + sum.y * sc[1] + sum.z * sc[4] + sum.w * sc[5]) - dmin * smin;

    }

    // 汇总部分和并写回结果
    barrier(CLK_LOCAL_MEM_FENCE);
    # 使用并行计算的方式对数组进行归约操作，每次将数组大小减半
    for (int s=16; s>0; s>>=1) {
        # 如果线程ID小于当前数组大小的一半，则将当前元素与对应位置的元素相加
        if (tid < s) {
            tmp[tid] += tmp[tid + s];
        }
        # 等待本地内存操作完成
        barrier(CLK_LOCAL_MEM_FENCE);
    }
    # 如果线程ID为0，则将归约后的结果写入目标数组中
    if (tid == 0) {
        dst[row] = tmp[0];
    }
}

// 定义 OpenCL 内核函数，用于对输入进行反量化和矩阵向量乘法
__kernel void dequantize_mul_mat_vec_q6_K(__global const struct block_q6_K * xx, __local float* tmp, __global const float * yy, __global float * dst, const int ncols) {

    // 获取当前工作组的 ID，即行号
    const int row = get_group_id(0);

    // 计算每行的块数
    const int num_blocks_per_row = ncols / QK_K;
    // 计算当前块的索引
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    // 获取当前块的指针
    __global const struct block_q6_K * x = xx + ib0;

    // 计算当前线程的 ID，用于反量化操作
    const int tid = get_local_id(0)/K_QUANTS_PER_ITERATION;  // 0...31 or 0...16
    // 计算当前线程在块内的索引，用于反量化操作
    const int ix  = get_local_id(0)%K_QUANTS_PER_ITERATION;  // 0 or 0, 1

    // 计算步长
    const int step = 16/K_QUANTS_PER_ITERATION;          // 16 or 8

    // 计算反量化操作的索引
    const int im = tid/step;                             // 0 or 1. 0 computes 0..., 1 computes 128...
    const int in = tid - step*im;                        // 0...15 or 0...7

\n#if K_QUANTS_PER_ITERATION == 1\n
    // 根据 K_QUANTS_PER_ITERATION 的值计算 l0 和 is
    const int l0 = K_QUANTS_PER_ITERATION*in;            // 0...15
    const int is = 0;

\n#else\n
    // 根据 K_QUANTS_PER_ITERATION 的值计算 l0 和 is
    const int l0 = 4 * in;                               // 0, 4, 8, ..., 28
    const int is = in / 4;

\n#endif\n

    // 计算反量化操作的偏移量
    const int ql_offset = 64*im + l0;
    const int qh_offset = 32*im + l0;
    const int s_offset  =  8*im + is;
    const int y_offset = 128*im + l0;

    // 初始化临时数组
    tmp[16 * ix + tid] = 0; // partial sum for thread in warp

    // 遍历每个块进行反量化和矩阵向量乘法
    for (int i = ix; i < num_blocks_per_row; i += K_QUANTS_PER_ITERATION) {

        // 获取当前块的输入数据指针
        __global const float   * y  = yy + i * QK_K + y_offset;
        __global const uint8_t * ql = x[i].ql + ql_offset;
        __global const uint8_t * qh = x[i].qh + qh_offset;
        __global const int8_t  * s  = x[i].scales + s_offset;

        // 从块中加载数据
        const float d = vload_half(0, &x[i].d);
\n#if K_QUANTS_PER_ITERATION == 1\n
        // 如果每次迭代只有一个量化值
        float sum = y[ 0] * s[0] * d * ((int8_t)((ql[ 0] & 0xF) | ((qh[ 0] & 0x03) << 4)) - 32)
                  + y[16] * s[1] * d * ((int8_t)((ql[16] & 0xF) | ((qh[16] & 0x03) << 4)) - 32)
                  + y[32] * s[2] * d * ((int8_t)((ql[32] & 0xF) | ((qh[ 0] & 0x0c) << 2)) - 32)
                  + y[48] * s[3] * d * ((int8_t)((ql[48] & 0xF) | ((qh[16] & 0x0c) << 2)) - 32)
                  + y[64] * s[4] * d * ((int8_t)((ql[ 0]  >> 4) | ((qh[ 0] & 0x30) >> 0)) - 32)
                  + y[80] * s[5] * d * ((int8_t)((ql[16]  >> 4) | ((qh[16] & 0x30) >> 0)) - 32)
                  + y[96] * s[6] * d * ((int8_t)((ql[32]  >> 4) | ((qh[ 0] & 0xc0) >> 2)) - 32)
                  +y[112] * s[7] * d * ((int8_t)((ql[48]  >> 4) | ((qh[16] & 0xc0) >> 2)) - 32);
        tmp[16 * ix + tid] += sum;
\n#else\n
        // 如果每次迭代有多个量化值
        float sum = 0;
        for (int l = 0; l < 4; ++l) {
            sum += y[l+ 0] * s[0] * d * ((int8_t)((ql[l+ 0] & 0xF) | (((qh[l] >> 0) & 3) << 4)) - 32)
                 + y[l+32] * s[2] * d * ((int8_t)((ql[l+32] & 0xF) | (((qh[l] >> 2) & 3) << 4)) - 32)
                 + y[l+64] * s[4] * d * ((int8_t)((ql[l+ 0]  >> 4) | (((qh[l] >> 4) & 3) << 4)) - 32)
                 + y[l+96] * s[6] * d * ((int8_t)((ql[l+32]  >> 4) | (((qh[l] >> 6) & 3) << 4)) - 32);
        }
        tmp[16 * ix + tid] += sum;
\n#endif\n

    }

    // 汇总部分和并写回结果
    barrier(CLK_LOCAL_MEM_FENCE);
    for (int s=16; s>0; s>>=1) {
        if (tid < s) {
            tmp[tid] += tmp[tid + s];
        }
        barrier(CLK_LOCAL_MEM_FENCE);
    }
    if (tid == 0) {
        dst[row] = tmp[0];
    }
}

);


std::string dequant_template = MULTILINE_QUOTE(
__kernel void KERNEL_NAME(__global X_TYPE* x, __global float* y) {
    const int i = get_group_id(0)*get_local_size(0) + get_local_id(0)*2;

    if (i >= get_global_size(0)) {
        return;
    }

    const uint qk = QUANT_K;
    const uint qr = QUANT_R;
    // 计算块索引
    const int ib = i/qk + get_global_offset(0); // block index
    // 计算量化索引
    const int iqs = (i%qk)/qr; // quant index
    // 计算 y 块的起始索引
    const int iybs = i - i%qk; // y block start index
    // 计算 y 偏移量
    const int y_offset = qr == 1 ? 1 : qk/2;

    // 反量化
    float v0, v1;
    DEQUANT_FUNC(x, ib, iqs, &v0, &v1);
    // 将反量化后的值存入 y 数组
    y[iybs + iqs + 0] = v0;
    y[iybs + iqs + y_offset] = v1;
// 定义了一个模板字符串，包含了一个 OpenCL 内核函数，用于矩阵向量乘法
std::string dequant_mul_mat_vec_template = MULTILINE_QUOTE(
__kernel void KERNEL_NAME(__global X_TYPE* x, __local float* tmp, __global float* y, __global float* dst, const int ncols) {
    // 获取本地工作组大小
    const int local_size = get_local_size(0);
    // 获取组 ID 作为行索引
    const int row = get_group_id(0);
    // 获取本地 ID 作为线程 ID
    const int tid = get_local_id(0);

    // 定义量化参数
    const uint qk = QUANT_K;
    const uint qr = QUANT_R;

    // 计算每次处理的列数
    const int col_step = local_size * 2;
    // 计算 y 的偏移量
    const int y_offset = qr == 1 ? 1 : qk/2;

    // 根据全局偏移调整 x 指针
    x += get_global_offset(0);

    // 初始化临时变量为 0
    tmp[tid] = 0;

    // 循环处理矩阵向量乘法
    for (int col = tid*2; col < ncols; col += col_step) {
        // 计算块索引
        const int ib = (row*ncols + col)/qk; // block index
        // 计算量化索引
        const int iqs = (col%qk)/qr; // quant index
        // 计算 y 块的起始索引
        const int iybs = col - col%qk; // y block start index

        // 反量化
        float v0, v1;
        DEQUANT_FUNC(x, ib, iqs, &v0, &v1);

        // 矩阵乘法
        tmp[tid] += v0 * y[iybs + iqs + 0];
        tmp[tid] += v1 * y[iybs + iqs + y_offset];
    }

    // 合并局部和写回结果
    barrier(CLK_LOCAL_MEM_FENCE);
    for (int s=local_size/2; s>0; s>>=1) {
        if (tid < s) {
            tmp[tid] += tmp[tid + s];
        }
        barrier(CLK_LOCAL_MEM_FENCE);
    }
    if (tid == 0) {
        dst[row] = tmp[0];
    }
}
);

// 定义了一个模板字符串，包含了一个 OpenCL 内核函数，用于矩阵乘法
std::string mul_template = MULTILINE_QUOTE(
__kernel void KERNEL_NAME(__global TYPE* x, const int x_offset, __global TYPE* y, const int y_offset, __global TYPE* dst, const int dst_offset, const int ky) {
    // 获取全局 ID
    const int i = get_group_id(0)*get_local_size(0) + get_local_id(0);

    // 如果超出全局大小则返回
    if (i >= get_global_size(0)) {
        return;
    }

    // 执行矩阵乘法
    dst[dst_offset + i] = x[x_offset + i] * y[y_offset + i%ky];
}
);

// 定义了一个宏，用于检查 OpenCL 错误
#define CL_CHECK(err)                                               \
    # 使用宏定义创建一个 do-while 循环，用于检查 OpenCL 函数返回的错误码
    do {                                                            \  # 开始 do-while 循环
        cl_int err_ = (err);                                        \  # 定义一个变量 err_，并将传入的 err 赋值给它
        if (err_ != CL_SUCCESS) {                                   \  # 如果 err_ 不等于 CL_SUCCESS
            fprintf(stderr, "ggml_opencl: %s error %d at %s:%d\n",  \  # 打印错误信息到标准错误流
                #err, err_, __FILE__, __LINE__);                    \  # 使用预处理器指令将错误码、错误值、文件名和行号打印出来
            exit(1);                                                \  # 退出程序
        }                                                           \  # 结束 if 语句
    } while (0)                                                     \  # 结束 do-while 循环
// 定义一个宏，用于检查 CLBlast 函数的返回值，如果不是 CLBlastSuccess 则输出错误信息并退出程序
#define CLBLAST_CHECK(err)                                          \
    do {                                                            \
        CLBlastStatusCode err_ = (err);                             \
        if (err_ != CLBlastSuccess) {                               \
            fprintf(stderr, "ggml_opencl: %s error %d at %s:%d\n",  \
                #err, err_, __FILE__, __LINE__);                    \
            exit(1);                                                \
        }                                                           \
    } while (0)

// 定义包含 5 个字符串的数组，用于存储反量化操作所需的键值
std::array<std::string, 5> dequant_str_keys = {
    "KERNEL_NAME", "X_TYPE", "QUANT_K", "QUANT_R", "DEQUANT_FUNC"
};

// 定义包含 30 个字符串的数组，用于存储反量化操作所需的值
std::array<std::string, 30> dequant_str_values = {
    // ...（省略部分内容）
};

// 定义包含 30 个字符串的数组，用于存储反量化矩阵向量乘法操作所需的值
std::array<std::string, 30> dequant_mul_mat_vec_str_values = {
    // ...（省略部分内容）
};

// 定义包含 2 个字符串的数组，用于存储乘法操作所需的键值
std::array<std::string, 2> mul_str_keys = {
    "KERNEL_NAME", "TYPE"
};

// 定义包含 2 个字符串的数组，用于存储乘法操作所需的值
std::array<std::string, 2> mul_str_values = {
    "mul_f32", "float"
};
// 替换字符串 s 中的 from 字符串为 to 字符串
static std::string& replace(std::string& s, const std::string& from, const std::string& to) {
    // 初始化 pos 为 0
    size_t pos = 0;
    // 循环查找并替换 from 字符串
    while ((pos = s.find(from, pos)) != std::string::npos) {
         // 替换 from 字符串为 to 字符串
         s.replace(pos, from.length(), to);
         // 更新 pos 位置
         pos += to.length();
    }
    // 返回替换后的字符串
    return s;
}

// 生成 OpenCL 内核代码
static std::string generate_kernels() {
    // 创建字符串流对象 src
    std::stringstream src;
    // 将 program_source 和 k_quants_source 写入字符串流
    src << program_source << '\n';
    src << k_quants_source << '\n';
    // 遍历 dequant_str_values，生成对应的内核代码
    for (size_t i = 0; i < dequant_str_values.size(); i += dequant_str_keys.size()) {
        // 创建 dequant_kernel 和 dmmv_kernel 字符串
        std::string dequant_kernel = dequant_template;
        std::string dmmv_kernel = dequant_mul_mat_vec_template;
        // 遍历 dequant_str_keys，替换 dequant_kernel 和 dmmv_kernel 中的字符串
        for (size_t j = 0; j < dequant_str_keys.size(); j++) {
            replace(dequant_kernel, dequant_str_keys[j], dequant_str_values[i + j]);
            replace(dmmv_kernel, dequant_str_keys[j], dequant_mul_mat_vec_str_values[i + j]);
        }
        // 将生成的内核代码写入字符串流
        src << dequant_kernel << '\n';
        src << dmmv_kernel << '\n';
    }
    // 遍历 mul_str_values，生成对应的内核代码
    for (size_t i = 0; i < mul_str_values.size(); i += mul_str_keys.size()) {
        // 创建 mul_kernel 字符串
        std::string mul_kernel = mul_template;
        // 遍历 mul_str_keys，替换 mul_kernel 中的字符串
        for (size_t j = 0; j < mul_str_keys.size(); j++) {
            replace(mul_kernel, mul_str_keys[j], mul_str_values[i + j]);
        }
        // 将生成的内核代码写入字符串流
        src << mul_kernel << '\n';
    }

    // 返回生成的内核代码字符串
    return src.str();
}

// 定义 OpenCL 相关变量
static cl_platform_id platform;
static cl_device_id device;
static cl_context context;
static cl_command_queue queue;
static cl_program program;
static cl_kernel convert_row_f16_cl;
static cl_kernel dequantize_row_q4_0_cl, dequantize_row_q4_1_cl, dequantize_row_q5_0_cl, dequantize_row_q5_1_cl, dequantize_row_q8_0_cl;
static cl_kernel dequantize_mul_mat_vec_q4_0_cl, dequantize_mul_mat_vec_q4_1_cl, dequantize_mul_mat_vec_q5_0_cl, dequantize_mul_mat_vec_q5_1_cl, dequantize_mul_mat_vec_q8_0_cl, convert_mul_mat_vec_f16_cl;
static cl_kernel dequantize_block_q2_k_cl, dequantize_block_q3_k_cl, dequantize_block_q4_k_cl, dequantize_block_q5_k_cl, dequantize_block_q6_k_cl;
// 声明静态 OpenCL 内核对象
static cl_kernel dequantize_mul_mat_vec_q2_K_cl, dequantize_mul_mat_vec_q3_K_cl, dequantize_mul_mat_vec_q4_K_cl, dequantize_mul_mat_vec_q5_K_cl, dequantize_mul_mat_vec_q6_K_cl;
// 声明静态 OpenCL 内核对象
static cl_kernel mul_f32_cl;
// 声明静态布尔变量，表示是否支持 fp16
static bool fp16_support;

// 从源代码构建 OpenCL 程序
static cl_program build_program_from_source(cl_context ctx, cl_device_id dev, const char* program_buffer) {
    cl_program p;
    char *program_log;
    size_t program_size;
    size_t log_size;
    int err;

    // 获取程序缓冲区的大小
    program_size = strlen(program_buffer);

    // 创建包含源代码的程序对象
    p = clCreateProgramWithSource(ctx, 1, (const char**)&program_buffer, &program_size, &err);
    if(err < 0) {
        fprintf(stderr, "OpenCL error creating program");
        exit(1);
    }

    // 设置编译选项
    std::string compile_opts = "-cl-mad-enable -cl-unsafe-math-optimizations -cl-finite-math-only -cl-fast-relaxed-math "
                               "-DQK4_0=32 -DQR4_0=2 -DQK4_1=32 -DQR4_1=2 -DQK5_0=32 -DQR5_0=2 -DQK5_1=32 -DQR5_1=2 -DQK8_0=32 -DQR8_0=1 "
                               "-DQK_K=256 -DK_QUANTS_PER_ITERATION=" + std::to_string(K_QUANTS_PER_ITERATION);

    // 编译程序
    err = clBuildProgram(p, 0, NULL, compile_opts.c_str(), NULL, NULL);
    if(err < 0) {
        // 获取编译日志
        clGetProgramBuildInfo(p, dev, CL_PROGRAM_BUILD_LOG, 0, NULL, &log_size);
        program_log = (char*) malloc(log_size + 1);
        program_log[log_size] = '\0';
        clGetProgramBuildInfo(p, dev, CL_PROGRAM_BUILD_LOG, log_size + 1, program_log, NULL);
        fprintf(stderr, "ggml_opencl: kernel compile error:\n\n%s\n", program_log);
        free(program_log);
        exit(1);
    }

    return p;
}

// 初始化 OpenCL 环境
void ggml_cl_init(void) {
    cl_int err;

    // 声明 OpenCL 设备和平台结构体
    struct cl_device;
    struct cl_platform {
        cl_platform_id id;
        unsigned number;
        char name[128];
        char vendor[128];
        struct cl_device * devices;
        unsigned n_devices;
        struct cl_device * default_device;
    };
}
    // 定义表示 OpenCL 设备的结构体
    struct cl_device {
        struct cl_platform * platform;  // 指向设备所在平台的指针
        cl_device_id id;  // 设备的 ID
        unsigned number;  // 设备的编号
        cl_device_type type;  // 设备的类型
        char name[128];  // 设备的名称
    };

    // 定义表示 OpenCL 平台的结构体数组
    enum { NPLAT = 16, NDEV = 16 };
    struct cl_platform platforms[NPLAT];  // 存储平台信息的数组
    unsigned n_platforms = 0;  // 平台数量
    struct cl_device devices[NDEV];  // 存储设备信息的数组
    unsigned n_devices = 0;  // 设备数量
    struct cl_device * default_device = NULL;  // 默认设备指针

    platform = NULL;  // 初始化平台指针为 NULL
    device = NULL;  // 初始化设备指针为 NULL

    cl_platform_id platform_ids[NPLAT];  // 存储平台 ID 的数组
    // 获取平台 ID 数组
    CL_CHECK(clGetPlatformIDs(NPLAT, platform_ids, &n_platforms));

    // 遍历每个平台
    for (unsigned i = 0; i < n_platforms; i++) {
        struct cl_platform * p = &platforms[i];  // 获取当前平台的指针
        p->number = i;  // 设置平台的编号
        p->id = platform_ids[i];  // 设置平台的 ID
        // 获取平台的名称
        CL_CHECK(clGetPlatformInfo(p->id, CL_PLATFORM_NAME, sizeof(p->name), &p->name, NULL));
        // 获取平台的供应商
        CL_CHECK(clGetPlatformInfo(p->id, CL_PLATFORM_VENDOR, sizeof(p->vendor), &p->vendor, NULL));

        cl_device_id device_ids[NDEV];  // 存储设备 ID 的数组
        // 获取设备 ID 数组
        cl_int clGetDeviceIDsError = clGetDeviceIDs(p->id, CL_DEVICE_TYPE_ALL, NDEV, device_ids, &p->n_devices);
        // 如果没有找到设备，则设备数量为 0
        if (clGetDeviceIDsError == CL_DEVICE_NOT_FOUND) {
            p->n_devices = 0;
        } else {
            CL_CHECK(clGetDeviceIDsError);
        }
        // 设置平台的设备数组指针
        p->devices = p->n_devices > 0 ? &devices[n_devices] : NULL;
        p->default_device = NULL;  // 初始化平台的默认设备指针为 NULL

        // 遍历每个设备
        for (unsigned j = 0; j < p->n_devices; j++) {
            struct cl_device * d = &devices[n_devices];  // 获取当前设备的指针
            d->number = n_devices++;  // 设置设备的编号并递增
            d->id = device_ids[j];  // 设置设备的 ID
            d->platform = p;  // 设置设备所在平台的指针
            // 获取设备的名称
            CL_CHECK(clGetDeviceInfo(d->id, CL_DEVICE_NAME, sizeof(d->name), &d->name, NULL));
            // 获取设备的类型
            CL_CHECK(clGetDeviceInfo(d->id, CL_DEVICE_TYPE, sizeof(d->type), &d->type, NULL));

            // 如果当前设备是 GPU 类型且平台的默认设备指针为 NULL，则设置默认设备为当前设备
            if (p->default_device == NULL && d->type == CL_DEVICE_TYPE_GPU) {
                p->default_device = d;
            }
        }

        // 如果全局的默认设备指针为 NULL 且当前平台的默认设备指针不为 NULL，则设置全局默认设备为当前平台的默认设备
        if (default_device == NULL && p->default_device != NULL) {
            default_device = p->default_device;
        }
    }
    // 如果没有找到任何 OpenCL 设备，则输出错误信息并退出程序
    if (n_devices == 0) {
        fprintf(stderr, "ggml_opencl: could find any OpenCL devices.\n");
        exit(1);
    }

    // 获取环境变量中的用户指定的平台和设备信息
    char * user_platform_string = getenv("GGML_OPENCL_PLATFORM");
    char * user_device_string = getenv("GGML_OPENCL_DEVICE");
    int user_platform_number = -1;
    int user_device_number = -1;

    unsigned n;
    // 如果用户指定了平台并且符合条件，则将其转换为整数
    if (user_platform_string != NULL && sscanf(user_platform_string, " %u", &n) == 1 && n < n_platforms) {
        user_platform_number = (int)n;
    }
    // 如果用户指定了设备并且符合条件，则将其转换为整数
    if (user_device_string != NULL && sscanf(user_device_string, " %u", &n) == 1 && n < n_devices) {
        user_device_number = (int)n;
    }
    // 如果用户指定了平台和设备，则选择对应的平台和设备
    if (user_platform_number != -1 && user_device_number != -1) {
        cl_platform* platform = &platforms[user_platform_number];
        // 如果用户指定的设备号超出了平台所包含的设备数，则输出错误信息并退出程序
        if ((unsigned)user_device_number >= platform->n_devices) {
            fprintf(stderr, "ggml_opencl: invalid device number %d\n", user_device_number);
            exit(1);
        }
        default_device = &platform->devices[user_device_number];
    }

    // 输出选择的平台和设备信息
    fprintf(stderr, "ggml_opencl: selecting platform: '%s'\n", default_device->platform->name);
    fprintf(stderr, "ggml_opencl: selecting device: '%s'\n", default_device->name);
    // 如果选择的设备不是 GPU 类型，则输出警告信息
    if (default_device->type != CL_DEVICE_TYPE_GPU) {
        fprintf(stderr, "ggml_opencl: warning, not a GPU: '%s'.\n", default_device->name);
    }

    // 设置全局变量 platform 和 device 为选择的设备的平台和设备 ID
    platform = default_device->platform->id;
    device = default_device->id;

    // 获取设备支持的扩展信息，并检查是否支持 cl_khr_fp16
    size_t ext_str_size;
    clGetDeviceInfo(device, CL_DEVICE_EXTENSIONS, 0, NULL, &ext_str_size);
    char *ext_buffer = (char *)alloca(ext_str_size + 1);
    clGetDeviceInfo(device, CL_DEVICE_EXTENSIONS, ext_str_size, ext_buffer, NULL);
    ext_buffer[ext_str_size] = '\0'; // 确保以 null 结尾
    fp16_support = strstr(ext_buffer, "cl_khr_fp16") != NULL; // 检查是否支持 cl_khr_fp16
    fprintf(stderr, "ggml_opencl: device FP16 support: %s\n", fp16_support ? "true" : "false");
    // 定义 OpenCL 上下文属性数组，指定使用的平台
    cl_context_properties properties[] = {
        (intptr_t)CL_CONTEXT_PLATFORM, (intptr_t)platform, 0
    };

    // 创建 OpenCL 上下文
    CL_CHECK((context = clCreateContext(properties, 1, &device, NULL, NULL, &err), err));

    // 创建 OpenCL 命令队列，启用乱序执行模式
    CL_CHECK((queue = clCreateCommandQueue(context, device, CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE, &err),
        (err != CL_INVALID_QUEUE_PROPERTIES && err != CL_INVALID_VALUE ? err :
        (queue = clCreateCommandQueue(context, device, 0, &err), err)
    )));

    // 生成内核源代码
    const std::string kernel_src = generate_kernels();

    // 从源代码构建程序
    program = build_program_from_source(context, device, kernel_src.c_str());

    // 创建 FP16 到 FP32 的转换内核
    CL_CHECK((convert_row_f16_cl = clCreateKernel(program, "convert_row_f16", &err), err));

    // 创建去量化内核
    CL_CHECK((dequantize_row_q4_0_cl = clCreateKernel(program, "dequantize_row_q4_0", &err), err));
    CL_CHECK((dequantize_row_q4_1_cl = clCreateKernel(program, "dequantize_row_q4_1", &err), err));
    CL_CHECK((dequantize_row_q5_0_cl = clCreateKernel(program, "dequantize_row_q5_0", &err), err));
    CL_CHECK((dequantize_row_q5_1_cl = clCreateKernel(program, "dequantize_row_q5_1", &err), err));
    CL_CHECK((dequantize_row_q8_0_cl = clCreateKernel(program, "dequantize_row_q8_0", &err), err));
    CL_CHECK((dequantize_row_q8_0_cl = clCreateKernel(program, "dequantize_row_q8_0", &err), err));
    CL_CHECK((dequantize_block_q2_k_cl = clCreateKernel(program, "dequantize_block_q2_K", &err), err));
    CL_CHECK((dequantize_block_q3_k_cl = clCreateKernel(program, "dequantize_block_q3_K", &err), err));
    CL_CHECK((dequantize_block_q4_k_cl = clCreateKernel(program, "dequantize_block_q4_K", &err), err));
    CL_CHECK((dequantize_block_q5_k_cl = clCreateKernel(program, "dequantize_block_q5_K", &err), err));
    CL_CHECK((dequantize_block_q6_k_cl = clCreateKernel(program, "dequantize_block_q6_K", &err), err));

    // 创建去量化乘法矩阵内核
    # 创建名为 dequantize_mul_mat_vec_q4_0_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q4_0 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q4_0_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q4_0", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q4_1_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q4_1 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q4_1_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q4_1", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q5_0_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q5_0 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q5_0_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q5_0", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q5_1_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q5_1 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q5_1_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q5_1", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q8_0_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q8_0 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q8_0_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q8_0", &err), err));
    # 创建名为 convert_mul_mat_vec_f16_cl 的 OpenCL 内核对象，用于执行指定程序中的 convert_mul_mat_vec_f16 内核函数
    CL_CHECK((convert_mul_mat_vec_f16_cl = clCreateKernel(program, "convert_mul_mat_vec_f16", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q2_K_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q2_K 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q2_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q2_K", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q3_K_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q3_K 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q3_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q3_K", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q4_K_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q4_K 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q4_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q4_K", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q5_K_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q5_K 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q5_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q5_K", &err), err));
    # 创建名为 dequantize_mul_mat_vec_q6_K_cl 的 OpenCL 内核对象，用于执行指定程序中的 dequantize_mul_mat_vec_q6_K 内核函数
    CL_CHECK((dequantize_mul_mat_vec_q6_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q6_K", &err), err));

    # 创建名为 mul_f32_cl 的 OpenCL 内核对象，用于执行指定程序中的 mul_f32 内核函数
    # mul kernel
    CL_CHECK((mul_f32_cl = clCreateKernel(program, "mul_f32", &err), err));
# 根据输入的类型返回对应的 OpenCL 内核指针
static cl_kernel* ggml_get_to_fp32_cl(ggml_type type) {
    # 根据不同的类型返回对应的内核指针
    switch (type) {
        case GGML_TYPE_Q4_0:
            return &dequantize_row_q4_0_cl;
        case GGML_TYPE_Q4_1:
            return &dequantize_row_q4_1_cl;
        case GGML_TYPE_Q5_0:
            return &dequantize_row_q5_0_cl;
        case GGML_TYPE_Q5_1:
            return &dequantize_row_q5_1_cl;
        case GGML_TYPE_Q8_0:
            return &dequantize_row_q8_0_cl;
        case GGML_TYPE_Q2_K:
            return &dequantize_block_q2_k_cl;
        case GGML_TYPE_Q3_K:
            return &dequantize_block_q3_k_cl;
        case GGML_TYPE_Q4_K:
            return &dequantize_block_q4_k_cl;
        case GGML_TYPE_Q5_K:
            return &dequantize_block_q5_k_cl;
        case GGML_TYPE_Q6_K:
            return &dequantize_block_q6_k_cl;
        case GGML_TYPE_F16:
            return &convert_row_f16_cl;
        default:
            return nullptr;
    }
}

# 根据输入的类型返回全局工作组大小
static size_t ggml_cl_global_denom(ggml_type type) {
    # 根据不同的类型返回对应的全局工作组大小
    switch (type) {
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
            return 1;
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
            return 4;
        case GGML_TYPE_Q4_K:
            return 8;
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
            return 4;
        case GGML_TYPE_F16:
        default:
            return 1;
    }
}

# 根据输入的类型返回局部工作组大小
static size_t ggml_cl_local_size(ggml_type type) {
    # 根据不同的类型返回对应的局部工作组大小
    switch (type) {
        case GGML_TYPE_Q4_0:
        case GGML_TYPE_Q4_1:
        case GGML_TYPE_Q5_0:
        case GGML_TYPE_Q5_1:
        case GGML_TYPE_Q8_0:
            return 0;
        case GGML_TYPE_Q2_K:
        case GGML_TYPE_Q3_K:
            return 64;
        case GGML_TYPE_Q4_K:
            return 32;
        case GGML_TYPE_Q5_K:
        case GGML_TYPE_Q6_K:
            return 64;
        case GGML_TYPE_F16:
        default:
            return 0;
    }
}
// 根据输入的类型返回对应的 OpenCL 内核指针
static cl_kernel* ggml_get_dequantize_mul_mat_vec_cl(ggml_type type) {
    switch (type) {
        case GGML_TYPE_Q4_0:
            return &dequantize_mul_mat_vec_q4_0_cl;
        case GGML_TYPE_Q4_1:
            return &dequantize_mul_mat_vec_q4_1_cl;
        case GGML_TYPE_Q5_0:
            return &dequantize_mul_mat_vec_q5_0_cl;
        case GGML_TYPE_Q5_1:
            return &dequantize_mul_mat_vec_q5_1_cl;
        case GGML_TYPE_Q8_0:
            return &dequantize_mul_mat_vec_q8_0_cl;
        case GGML_TYPE_F16:
            return &convert_mul_mat_vec_f16_cl;
        case GGML_TYPE_Q2_K:
            return &dequantize_mul_mat_vec_q2_K_cl;
        case GGML_TYPE_Q3_K:
            return &dequantize_mul_mat_vec_q3_K_cl;
        case GGML_TYPE_Q4_K:
            return &dequantize_mul_mat_vec_q4_K_cl;
        case GGML_TYPE_Q5_K:
            return &dequantize_mul_mat_vec_q5_K_cl;
        case GGML_TYPE_Q6_K:
            return &dequantize_mul_mat_vec_q6_K_cl;
        default:
            return nullptr;
    }
}

// 定义最大的 OpenCL 缓冲区数量
#define MAX_CL_BUFFERS 256

// 定义一个结构体，用于实现作用域内的自旋锁
struct scoped_spin_lock {
    std::atomic_flag& lock;
    scoped_spin_lock(std::atomic_flag& lock) : lock(lock) {
        while (lock.test_and_set(std::memory_order_acquire)) {
            ; // 自旋
        }
    }
    ~scoped_spin_lock() {
        lock.clear(std::memory_order_release);
    }
    scoped_spin_lock(const scoped_spin_lock&) = delete;
    scoped_spin_lock& operator=(const scoped_spin_lock&) = delete;
};

// 定义一个 OpenCL 缓冲区结构体
struct cl_buffer {
    cl_mem mem;
    size_t size = 0;
};

// 定义一个静态的 OpenCL 缓冲区池
static cl_buffer g_cl_buffer_pool[MAX_CL_BUFFERS];
// 定义一个原子标志，用于实现自旋锁
static std::atomic_flag g_cl_pool_lock = ATOMIC_FLAG_INIT;

// 分配 OpenCL 缓冲区的函数
static cl_mem ggml_cl_pool_malloc(size_t size, size_t * actual_size) {
    // 获取自旋锁
    scoped_spin_lock lock(g_cl_pool_lock);
    cl_int err;

    int best_i = -1;
    size_t best_size = std::numeric_limits<size_t>::max(); // 最小的未使用的缓冲区，适合我们的需求
    int worst_i = -1;
    size_t worst_size = 0; //记录目前为止最大的未使用的缓冲区大小
    for (int i = 0; i < MAX_CL_BUFFERS; ++i) {
        cl_buffer &b = g_cl_buffer_pool[i]; //获取第i个缓冲区的引用
        if (b.size > 0 && b.size >= size && b.size < best_size) //如果缓冲区大小大于0且大于等于所需大小且小于最佳大小
        {
            best_i = i; //更新最佳缓冲区的索引
            best_size = b.size; //更新最佳缓冲区的大小
        }
        if (b.size > 0 && b.size > worst_size) //如果缓冲区大小大于0且大于最大大小
        {
            worst_i = i; //更新最差缓冲区的索引
            worst_size = b.size; //更新最差缓冲区的大小
        }
    }
    if(best_i!=-1) //找到了符合需求的最小缓冲区
    {
        cl_buffer& b = g_cl_buffer_pool[best_i]; //获取最佳缓冲区的引用
        cl_mem mem = b.mem; //获取最佳缓冲区的内存对象
        *actual_size = b.size; //更新实际大小
        b.size = 0; //重置缓冲区大小
        return mem; //返回内存对象
    }
    if(worst_i!=-1) //没有符合需求的缓冲区，调整最大的缓冲区以节省内存
    {
         cl_buffer& b = g_cl_buffer_pool[worst_i]; //获取最差缓冲区的引用
         cl_mem mem = b.mem; //获取最差缓冲区的内存对象
         b.size = 0; //重置缓冲区大小
         clReleaseMemObject(mem); //释放内存对象
    }
    cl_mem mem; //声明内存对象
    CL_CHECK((mem = clCreateBuffer(context, CL_MEM_READ_WRITE, size, NULL, &err), err)); //创建新的内存对象
    *actual_size = size; //更新实际大小
    return mem; //返回内存对象
static void ggml_cl_pool_free(cl_mem mem, size_t size) {
    // 使用 scoped_spin_lock 对象进行加锁，确保线程安全
    scoped_spin_lock lock(g_cl_pool_lock);

    // 遍历全局的 cl_buffer_pool 数组
    for (int i = 0; i < MAX_CL_BUFFERS; ++i) {
        // 获取当前索引对应的 cl_buffer 对象
        cl_buffer& b = g_cl_buffer_pool[i];
        // 如果当前 cl_buffer 对象的 size 为 0，表示该对象未被使用
        if (b.size == 0) {
            // 将传入的内存和大小信息存储到当前 cl_buffer 对象中
            b.mem = mem;
            b.size = size;
            // 返回，表示成功存储
            return;
        }
    }
    // 如果 cl_buffer_pool 已满，输出警告信息
    fprintf(stderr, "WARNING: cl buffer pool full, increase MAX_CL_BUFFERS\n");
    // 释放传入的内存对象
    clReleaseMemObject(mem);
}

void ggml_cl_free_data(const struct ggml_tensor* tensor) {
    // 如果 tensor 的 backend 不是 GGML_BACKEND_GPU，直接返回
    if (tensor->backend != GGML_BACKEND_GPU) {
        return;
    }
    // 将 tensor 的额外信息转换为 cl_mem 对象
    cl_mem mem = (cl_mem)tensor->extra;
    // 释放 cl_mem 对象
    clReleaseMemObject(mem);
}

static cl_int ggml_cl_h2d_tensor_2d(cl_command_queue queue, cl_mem dst, size_t offset, const struct ggml_tensor * src, uint64_t i3, uint64_t i2, cl_event* ev) {
    cl_int err;
    // 获取 src 的维度信息和数据类型
    const uint64_t ne0 = src->ne[0];
    const uint64_t ne1 = src->ne[1];
    const uint64_t nb0 = src->nb[0];
    const uint64_t nb1 = src->nb[1];
    const uint64_t nb2 = src->nb[2];
    const uint64_t nb3 = src->nb[3];
    const enum ggml_type type = src->type;
    // 计算数据类型的大小和块大小
    const size_t ts = ggml_type_size(type);
    const size_t bs = ggml_blck_size(type);
    // 计算每行数据的大小
    const uint64_t row_size = ts*ne0/bs;

    // 计算数据在源内存中的偏移位置
    const char * x = (const char *) src->data + i2*nb2 + i3*nb3;
    // 如果数据在内存中的布局和目标内存一致，直接写入数据
    if (nb0 == ts && nb1 == row_size) {
        return clEnqueueWriteBuffer(queue, dst, CL_FALSE, offset, ne1*row_size, x, 0, NULL, ev);
    }
    // 如果数据在内存中的布局和目标内存不一致，使用 clEnqueueWriteBufferRect 进行写入
    if (nb0 == ts) {
        const size_t buffer_origin[3] = { offset, 0, 0 };
        const size_t host_origin[3] = { 0, 0, 0 };
        const size_t region[3] = { row_size, ne1, 1 };
        return clEnqueueWriteBufferRect(queue, dst, CL_FALSE, buffer_origin, host_origin, region, row_size, 0, nb1, 0, x, 0, NULL, ev);
    }
    // 如果数据在内存中的布局和目标内存不一致，并且需要进行转换，使用 std::vector<cl_event> 对象进行处理
    std::vector<cl_event> events;
    if (ev && ne1>1) events.reserve(ne1-1);
}
    // 使用循环遍历每个元素，i1 从 0 开始，小于 ne1 结束
    for (uint64_t i1 = 0; i1 < ne1; i1++) {
        // 假设行是一个列数为 1 的矩阵，计算起始位置
        const size_t buffer_origin[3] = { offset + i1*row_size, 0, 0 };
        // 设置主机内存中的起始位置
        const size_t host_origin[3] = { 0, 0, 0 };
        // 设置区域大小
        const size_t region[3] = { ts, ne0/bs, 1 };
        // 如果请求了事件，使最后一个写操作等待所有先前的写操作完成
        if (ev && i1) {
            events.push_back(*ev);
        }
        // 计算事件数量
        cl_uint nevents = i1 == ne1-1 ? events.size() : 0U;
        // 将数据从主机内存写入缓冲区
        err = clEnqueueWriteBufferRect(queue, dst, CL_FALSE, buffer_origin, host_origin, region, ts, 0, nb0, 0, x + i1*nb1, nevents, nevents ? events.data() : nullptr, ev);
        // 如果写入出错，释放事件并返回错误码
        if (err != CL_SUCCESS) {
            for (auto event : events) {
                clReleaseEvent(event);
            }
            return err;
        }
    }
    // 释放所有事件
    for (auto event : events) {
        CL_CHECK(clReleaseEvent(event));
    }
    // 返回成功状态码
    return CL_SUCCESS;
    # 定义一个静态函数，用于在 GPU 上执行两个浮点数张量的乘法操作
    static void ggml_cl_mul_f32(const ggml_tensor * src0, const ggml_tensor * src1, ggml_tensor * dst) {
        # 断言 src1 的后端是 GPU
        GGML_ASSERT(src1->backend == GGML_BACKEND_GPU);
        # 获取 src0 的各维度大小
        const int64_t ne00 = src0->ne[0];
        const int64_t ne01 = src0->ne[1];
        const int64_t ne02 = src0->ne[2];
        const int64_t ne03 = src0->ne[3];
        # 获取 src1 的各维度大小
        const int64_t ne10 = src1->ne[0];
        const int64_t ne11 = src1->ne[1];
        const int64_t ne12 = src1->ne[2];
        const int64_t ne13 = src1->ne[3];
        # 获取 dst 的第二和第三维度大小
        const int nb2  = dst->nb[2];
        const int nb3  = dst->nb[3];
        # 定义变量用于存储数据大小
        size_t x_size;
        size_t d_size;
        
        # 在 GPU 上分配内存，用于存储 src0 的数据，并获取数据大小
        cl_mem d_X = ggml_cl_pool_malloc(ne00 * ne01 * sizeof(float), &x_size); // src0
        # 将 src1 的数据转换为 cl_mem 类型，因为 src1 已经在设备上，所以不需要再次分配内存
        cl_mem d_Y = (cl_mem) src1->extra; // src1 is already on device, broadcasted.
        # 在 GPU 上分配内存，用于存储 dst 的数据，并获取数据大小
        cl_mem d_D = ggml_cl_pool_malloc(ne00 * ne01 * sizeof(float), &d_size); // dst
    // 循环遍历ne03次
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        // 循环遍历ne02次
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            // 创建事件对象
            cl_event ev;

            // 将src0的数据拷贝到设备
            CL_CHECK(ggml_cl_h2d_tensor_2d(queue, d_X, 0, src0, i03, i02, &ev));

            // 计算i03和i02对ne13和ne12的取模
            const int64_t i13 = i03%ne13;
            const int64_t i12 = i02%ne12;
            const int i1 = i13*ne12*ne11 + i12*ne11;

            // 设置偏移量
            cl_int x_offset = 0;
            cl_int y_offset = i1*ne10;
            cl_int d_offset = 0;

            // 计算全局工作组大小
            size_t global = ne00 * ne01;
            cl_int ky = ne10 * ne11;

            // 设置内核参数
            CL_CHECK(clSetKernelArg(mul_f32_cl, 0, sizeof(cl_mem), &d_X));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 1, sizeof(cl_int), &x_offset));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 2, sizeof(cl_mem), &d_Y));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 3, sizeof(cl_int), &y_offset));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 4, sizeof(cl_mem), &d_D));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 5, sizeof(cl_int), &d_offset));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 6, sizeof(cl_int), &ky));
            // 执行内核
            CL_CHECK(clEnqueueNDRangeKernel(queue, mul_f32_cl, 1, NULL, &global, NULL, 1, &ev, NULL));

            // 释放事件对象
            CL_CHECK(clReleaseEvent(ev));
            // 等待队列中的命令执行完毕
            CL_CHECK(clFinish(queue));

            // 将结果拷贝到主机
            float * d = (float *) ((char *) dst->data + i02*nb2 + i03*nb3);
            CL_CHECK(clEnqueueReadBuffer(queue, d_D, true, 0, sizeof(float) * ne00*ne01, d, 0, NULL, NULL));
        }
    }
    // 释放内存
    ggml_cl_pool_free(d_X, x_size);
    ggml_cl_pool_free(d_D, d_size);
}

void ggml_cl_mul(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst) {
    // 确保输入和输出张量的类型都是 GGML_TYPE_F32
    GGML_ASSERT(src0->type == GGML_TYPE_F32 && src1->type == GGML_TYPE_F32 && dst->type == GGML_TYPE_F32);
    // 调用 ggml_cl_mul_f32 函数进行张量相乘
    ggml_cl_mul_f32(src0, src1, dst);
}

static void ggml_cl_mul_mat_f32(const ggml_tensor * src0, const ggml_tensor * src1, ggml_tensor * dst) {
    // 获取输入张量的维度信息
    const int64_t ne00 = src0->ne[0];
    const int64_t ne01 = src0->ne[1];
    const int64_t ne02 = src0->ne[2];
    const int64_t ne03 = src0->ne[3];

    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
    const int64_t ne13 = src1->ne[3];

    const int nb2  = dst->nb[2];
    const int nb3  = dst->nb[3];

    // 计算一些中间变量
    const int64_t r2 = ne12 / ne02;
    const int64_t r3 = ne13 / ne03;

    const float alpha = 1.0f;
    const float beta = 0.0f;
    const int x_ne = ne01 * ne00;
    const int y_ne = ne11 * ne10;
    const int d_ne = ne11 * ne01;

    size_t x_size;
    size_t y_size;
    size_t d_size;
    cl_mem d_X;
    // 根据输入张量的后端类型分配内存
    if (src0->backend == GGML_BACKEND_GPU) { // NOLINT
        d_X = (cl_mem) src0->extra;
    } else {
        d_X = ggml_cl_pool_malloc(sizeof(float) * x_ne, &x_size);
    }
    cl_mem d_Y = ggml_cl_pool_malloc(sizeof(float) * y_ne, &y_size);
    cl_mem d_D = ggml_cl_pool_malloc(sizeof(float) * d_ne, &d_size;

    size_t x_offset = 0;

    }

    // 根据输入张量的后端类型释放内存
    if (src0->backend != GGML_BACKEND_GPU) {
        ggml_cl_pool_free(d_X, x_size);
    }
    ggml_cl_pool_free(d_Y, y_size);
    ggml_cl_pool_free(d_D, d_size);
}

static void ggml_cl_mul_mat_f16(const ggml_tensor * src0, const ggml_tensor * src1, ggml_tensor * dst, void * wdata, size_t wsize) {
    // 确保当前环境支持 fp16 类型
    GGML_ASSERT(fp16_support);

    // 获取输入张量的维度信息
    const int64_t ne00 = src0->ne[0];
    const int64_t ne01 = src0->ne[1];
    const int64_t ne02 = src0->ne[2];
    const int64_t ne03 = src0->ne[3];

    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
}
    // 获取src1的第4个维度大小
    const int64_t ne13 = src1->ne[3];

    // 获取src1的第1个维度大小
    const int nb10 = src1->nb[0];
    // 获取src1的第2个维度大小
    const int nb11 = src1->nb[1];
    // 获取src1的第3个维度大小
    const int nb12 = src1->nb[2];
    // 获取src1的第4个维度大小
    const int nb13 = src1->nb[3];

    // 获取dst的第3个维度大小
    const int nb2  = dst->nb[2];
    // 获取dst的第4个维度大小
    const int nb3  = dst->nb[3];

    // 计算r2，ne12除以ne02
    const int64_t r2 = ne12 / ne02;
    // 计算r3，ne13除以ne03
    const int64_t r3 = ne13 / ne03;

    // 将浮点数1.0转换为fp16格式的alpha
    const ggml_fp16_t alpha = ggml_fp32_to_fp16(1.0f);
    // 将浮点数0.0转换为fp16格式的beta
    const ggml_fp16_t beta = ggml_fp32_to_fp16(0.0f);
    // 计算x_ne，ne01乘以ne00
    const int x_ne = ne01 * ne00;
    // 计算y_ne，ne11乘以ne10
    const int y_ne = ne11 * ne10;
    // 计算d_ne，ne11乘以ne01
    const int d_ne = ne11 * ne01;

    // 检查wsize是否大于等于y_ne个fp16的大小
    GGML_ASSERT(wsize >= sizeof(ggml_fp16_t) * y_ne);
    // 检查wsize是否大于等于d_ne个fp16的大小
    GGML_ASSERT(wsize >= sizeof(ggml_fp16_t) * d_ne);
    // 将wdata强制转换为ggml_fp16_t指针类型
    ggml_fp16_t * const tmp = (ggml_fp16_t *) wdata;

    // 定义变量
    size_t x_size;
    size_t y_size;
    size_t d_size;
    cl_mem d_X;
    // 如果src0的backend是GGML_BACKEND_GPU
    if (src0->backend == GGML_BACKEND_GPU) { // NOLINT
        // 将src0的extra强制转换为cl_mem类型
        d_X = (cl_mem) src0->extra;
    } else {
        // 分配内存空间，大小为x_ne个fp16的大小
        d_X = ggml_cl_pool_malloc(sizeof(ggml_fp16_t) * x_ne, &x_size);
    }
    // 分配内存空间，大小为y_ne个fp16的大小
    cl_mem d_Y = ggml_cl_pool_malloc(sizeof(ggml_fp16_t) * y_ne, &y_size);
    // 分配内存空间，大小为d_ne个fp16的大小
    cl_mem d_D = ggml_cl_pool_malloc(sizeof(ggml_fp16_t) * d_ne, &d_size);

    // 检查src1是否按行连续存储
    bool src1_cont_rows = nb10 == sizeof(float);
    // 检查src1是否按列连续存储
    bool src1_cont_cols = (size_t)nb11 == ne11*sizeof(float);

    // 定义变量，初始化为0
    size_t x_offset = 0;

    // 如果src0的backend不是GGML_BACKEND_GPU
    if (src0->backend != GGML_BACKEND_GPU) {
        // 释放d_X所占用的内存空间
        ggml_cl_pool_free(d_X, x_size);
    }
    // 释放d_Y所占用的内存空间
    ggml_cl_pool_free(d_Y, y_size);
    // 释放d_D所占用的内存空间
    ggml_cl_pool_free(d_D, d_size);
static void ggml_cl_mul_mat_q_f32(const ggml_tensor * src0, const ggml_tensor * src1, ggml_tensor * dst) {
    // 获取 src0 的维度信息
    const int64_t ne00 = src0->ne[0];
    const int64_t ne01 = src0->ne[1];
    const int64_t ne02 = src0->ne[2];
    const int64_t ne03 = src0->ne[3];

    // 获取 src1 的维度信息
    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
    const int64_t ne13 = src1->ne[3];

    // 获取 dst 的部分维度信息
    const int nb2  = dst->nb[2];
    const int nb3  = dst->nb[3];
    const ggml_type type = src0->type;
    // 判断是否是矩阵乘以向量
    const bool mul_mat_vec = ne11 == 1 && ne00%2 == 0;

    // 计算矩阵乘法的结果维度
    const int64_t r2 = ne12 / ne02;
    const int64_t r3 = ne13 / ne03;

    // 设置矩阵乘法的参数
    const float alpha = 1.0f;
    const float beta = 0.0f;
    const int x_ne = ne01 * ne00;
    const int y_ne = ne11 * ne10;
    const int d_ne = ne11 * ne01;
    const int x_bps = x_ne / ggml_blck_size(type); // blocks per 2D slice
    const size_t q_sz = ggml_type_size(type) * x_bps;

    // 计算内存大小
    size_t x_size;
    size_t y_size;
    size_t d_size;
    size_t q_size;
    cl_mem d_X;
    // 分配内存空间
    if (!mul_mat_vec) {
        d_X = ggml_cl_pool_malloc(sizeof(float) * x_ne, &x_size);
    }
    cl_mem d_Y = ggml_cl_pool_malloc(sizeof(float) * y_ne, &y_size);
    cl_mem d_D = ggml_cl_pool_malloc(sizeof(float) * d_ne, &d_size);
    cl_mem d_Q;
    // 如果 src0 的 backend 是 CPU，则分配内存空间
    if (src0->backend == GGML_BACKEND_CPU) {
        d_Q = ggml_cl_pool_malloc(q_sz, &q_size);
    }

    // 获取转换为 float32 的内核
    cl_kernel* to_fp32_cl = ggml_get_to_fp32_cl(type);
    cl_kernel* dmmv = ggml_get_dequantize_mul_mat_vec_cl(type);
    GGML_ASSERT(to_fp32_cl != nullptr);

    // 计算全局工作组大小和局部工作组大小
    const size_t global_denom = ggml_cl_global_denom(type);
    const size_t local = mul_mat_vec ? CL_DMMV_LOCAL_SIZE : ggml_cl_local_size(type);

    size_t ev_idx = 0;
    std::vector<cl_event> events;

    // 释放内存空间
    if (!mul_mat_vec) {
        ggml_cl_pool_free(d_X, x_size);
    }
    ggml_cl_pool_free(d_Y, y_size);
    ggml_cl_pool_free(d_D, d_size);
    if (src0->backend == GGML_BACKEND_CPU) {
        ggml_cl_pool_free(d_Q, q_size);
    }
}
// 检查是否可以对两个张量进行矩阵乘法操作
bool ggml_cl_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst) {
    // 获取第二个张量的第一个维度大小
    const int64_t ne10 = src1->ne[0];

    // 获取目标张量的第一个和第二个维度大小
    const int64_t ne0 = dst->ne[0];
    const int64_t ne1 = dst->ne[1];

    // TODO: 找到这些参数的最佳值
    // 检查张量类型和维度大小是否满足条件
    if ((src0->type == GGML_TYPE_F32 || src0->type == GGML_TYPE_F16 || ggml_is_quantized(src0->type)) &&
        src1->type == GGML_TYPE_F32 &&
        dst->type == GGML_TYPE_F32 &&
        ((ne0 >= 32 && ne1 >= 32 && ne10 >= 32) || src0->backend == GGML_BACKEND_GPU)) {
        return true;
    }

    return false;
}

// 检查是否可以使用FP16进行矩阵乘法操作
static bool ggml_cl_mul_mat_use_f16(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * /* dst */) {
    // 如果设备不支持FP16，则返回false
    if (!fp16_support) {
        return false;
    }

    // 计算源张量的字节大小
    size_t src0_sz = ggml_nbytes(src0);
    size_t src1_sz = ggml_nbytes(src1);

    // 计算转换为FP32后在设备上的传输大小
    size_t mul_mat_q_transfer = src0_sz + src1_sz;

    // 计算转换为FP16后在CPU上的传输大小
    size_t mul_mat_f16_transfer = src0_sz + sizeof(ggml_fp16_t) * ggml_nelements(src1);

    // 选择较小的传输大小传输到设备
    // TODO: 由于转换为FP16的开销，这并不总是最佳选择
    return mul_mat_f16_transfer < mul_mat_q_transfer;
}

// 执行矩阵乘法操作
void ggml_cl_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst, void * wdata, size_t wsize) {
    // 断言是否可以对两个张量进行矩阵乘法操作
    GGML_ASSERT(ggml_cl_can_mul_mat(src0, src1, dst));

    // 根据源张量的类型选择不同的矩阵乘法操作
    if (src0->type == GGML_TYPE_F32) {
        ggml_cl_mul_mat_f32(src0, src1, dst);
    }
    else if (src0->type == GGML_TYPE_F16) {
        // 如果可以使用FP16进行矩阵乘法操作，则执行FP16矩阵乘法
        if (ggml_cl_mul_mat_use_f16(src0, src1, dst)) {
            ggml_cl_mul_mat_f16(src0, src1, dst, wdata, wsize);
        }
        // 否则执行将源张量转换为FP32后的矩阵乘法
        else {
            ggml_cl_mul_mat_q_f32(src0, src1, dst);
        }
    }
    // 如果源张量是量化类型，则执行将源张量转换为FP32后的矩阵乘法
    else if (ggml_is_quantized(src0->type)) {
        ggml_cl_mul_mat_q_f32(src0, src1, dst);
    }
}
    # 如果条件不成立，执行以下代码块
    }
    # 否则，触发断言错误
    else {
        GGML_ASSERT(false);
    }
# 获取两个矩阵相乘后的结果的大小
size_t ggml_cl_mul_mat_get_wsize(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst) {
    # 如果输入矩阵的类型为 GGML_TYPE_F16，并且可以使用 F16 进行矩阵相乘
    if (src0->type == GGML_TYPE_F16 && ggml_cl_mul_mat_use_f16(src0, src1, dst)) {
        # 返回结果的大小为两个矩阵中较大的行列数乘以 F16 类型的大小
        return sizeof(ggml_fp16_t) * std::max(src1->ne[0] * src1->ne[1], dst->ne[0] * dst->ne[1]);
    }
    # 如果不满足上述条件，返回 0
    return 0;
}

# 将张量数据转换为设备可用的形式
void ggml_cl_transform_tensor(void * data, ggml_tensor * tensor) {
    # 获取张量的各个维度大小
    const int64_t ne0 = tensor->ne[0];
    const int64_t ne1 = tensor->ne[1];
    const int64_t ne2 = tensor->ne[2];
    const int64_t ne3 = tensor->ne[3];

    # 获取张量的数据类型
    const ggml_type type = tensor->type;
    # 计算每个张量元素的大小
    const size_t s_sz = ggml_type_size(type) * (size_t) (ne0 * ne1 / ggml_blck_size(type));
    # 计算整个张量的大小
    const size_t q_sz = s_sz * (size_t) (ne2 * ne3);

    # 分配设备内存空间
    size_t q_size;
    cl_mem dst = ggml_cl_pool_malloc(q_sz, &q_size);

    # 将张量的数据指针指向输入的数据
    tensor->data = data;
    # 将张量数据复制到设备
    size_t offset = 0;
    for (int64_t i3 = 0; i3 < ne3; i3++) {
        for (int64_t i2 = 0; i2 < ne2; i2++) {
            CL_CHECK(ggml_cl_h2d_tensor_2d(queue, dst, offset, tensor, i3, i2, NULL));
            offset += s_sz;
        }
    }

    # 等待设备操作完成
    CL_CHECK(clFinish(queue));

    # 将设备内存地址保存到张量的额外信息中
    tensor->extra = dst;
    # 确保张量的后端为 GPU
    GGML_ASSERT(tensor->backend == GGML_BACKEND_GPU);
}
```