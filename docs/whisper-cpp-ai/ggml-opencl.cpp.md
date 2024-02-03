# `whisper.cpp\ggml-opencl.cpp`

```cpp
// 包含 GGML 头文件
#include "ggml.h"
#include "ggml-opencl.h"
#include "ggml-backend-impl.h"

// 包含标准库头文件
#include <array>
#include <atomic>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <limits>
#include <sstream>
#include <vector>

// 定义要使用的 OpenCL 版本
#define CL_TARGET_OPENCL_VERSION 120
#include <clblast.h>

// 禁止特定编译器警告
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义本地工作组大小
#define CL_DMMV_LOCAL_SIZE 32

// 如果未定义 K_QUANTS_PER_ITERATION，则设置默认值为 1，否则检查其值是否为 1 或 2
#ifndef K_QUANTS_PER_ITERATION
#define K_QUANTS_PER_ITERATION 1
#else
static_assert(K_QUANTS_PER_ITERATION == 1 || K_QUANTS_PER_ITERATION == 2, "K_QUANTS_PER_ITERATION must be 1 or 2");
#endif

// 定义多行字符串宏
#define MULTILINE_QUOTE(...) #__VA_ARGS__
// 定义包含 OpenCL 程序源代码的字符串
static std::string program_source = MULTILINE_QUOTE(

// 定义不同数据类型的别名
typedef char int8_t;
typedef uchar uint8_t;
typedef short int16_t;
typedef ushort uint16_t;
typedef int int32_t;
typedef uint uint32_t;

// 定义不同数据结构，使用 packed 属性进行内存对齐
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
__kernel void convert_fp16_to_fp32(__global half* x, __global float* y) {
    // 获取当前工作项的全局 ID
    const uint i = get_global_id(0);

    // 将输入的半精度浮点数转换为单精度浮点数
    y[i] = vload_half(0, &x[i]);
}

void dequantize_q4_0(__global const struct block_q4_0* x, const int ib, const int iqs, float* v0, float* v1) {
    // 加载结构体中的半精度浮点数
    const float d = vload_half(0, &x[ib].d);

    // 获取结构体中的无符号整数
    const uint8_t vui = x[ib].qs[iqs];

    // 从无符号整数中提取两个有符号整数
    const int8_t vi0 = vui & 0xF;
    const int8_t vi1 = vui >> 4;

    // 计算两个浮点数的值
    *v0 = (vi0 - 8)*d;
    *v1 = (vi1 - 8)*d;
}

void dequantize_q4_1(__global const struct block_q4_1* x, const int ib, const int iqs, float* v0, float* v1) {
    // 加载结构体中的半精度浮点数
    const float d = vload_half(0, &x[ib].d);
    const float m = vload_half(0, &x[ib].m);

    // 获取结构体中的无符号整数
    const uint8_t vui = x[ib].qs[iqs];

    // 从无符号整数中提取两个有符号整数
    const int8_t vi0 = vui & 0xF;
    const int8_t vi1 = vui >> 4;

    // 计算两个浮点数的值
    *v0 = vi0*d + m;
    *v1 = vi1*d + m;
}

void dequantize_q5_0(__global const struct block_q5_0* x, const int ib, const int iqs, float* v0, float* v1) {
    // 加载结构体中的半精度浮点数
    const float d = vload_half(0, &x[ib].d);

    // 获取结构体中的无符号整数和无符号整数
    uint32_t qh = x[ib].qh;

    // 从无符号整数中提取两个有符号整数
    const uint8_t xh_0 = ((qh >> (iqs +  0)) << 4) & 0x10;
    const uint8_t xh_1 = ((qh >> (iqs + 12))     ) & 0x10;

    const int32_t x0 = ((x[ib].qs[iqs] & 0xf) | xh_0) - 16;
    const int32_t x1 = ((x[ib].qs[iqs] >>  4) | xh_1) - 16;

    // 计算两个浮点数的值
    *v0 = x0*d;
    *v1 = x1*d;
}

void dequantize_q5_1(__global const struct block_q5_1* x, const int ib, const int iqs, float* v0, float* v1) {
    // 加载结构体中的半精度浮点数
    const float d = vload_half(0, &x[ib].d);
    const float m = vload_half(0, &x[ib].m);

    // 获取结构体中的无符号整数和无符号整数
    uint32_t qh = x[ib].qh;

    // 从无符号整数中提取两个有符号整数
    const uint8_t xh_0 = ((qh >> (iqs +  0)) << 4) & 0x10;
    const uint8_t xh_1 = ((qh >> (iqs + 12))     ) & 0x10;

    const int32_t x0 = ((x[ib].qs[iqs] & 0xf) | xh_0);
    const int32_t x1 = ((x[ib].qs[iqs] >>  4) | xh_1);

    // 计算两个浮点数的值
    *v0 = x0*d + m;
    *v1 = x1*d + m;
}

void dequantize_q8_0(__global const struct block_q8_0* x, const int ib, const int iqs, float* v0, float* v1) {
    // 加载结构体中的半精度浮点数
    const float d = vload_half(0, &x[ib].d);

    // 获取结构体中的有符号整数
    const int8_t vi0 = x[ib].qs[iqs + 0];
}
    # 从数组 x 中获取索引为 ib 的元素，再从该元素中获取索引为 iqs+1 的值，存储在常量 vi1 中
    const int8_t vi1 = x[ib].qs[iqs + 1];

    # 将 vi0 乘以常量 d 的结果存储在指针 v0 指向的位置
    *v0 = vi0*d;
    # 将 vi1 乘以常量 d 的结果存储在指针 v1 指向的位置
    *v1 = vi1*d;
// 定义一个函数，将输入的 half 类型数据转换为 float 类型数据
void convert_f16(__global half* x, const int ib, const int iqs, float* v0, float* v1){
    // 从 x 数组中读取第 ib 和 ib+1 位置的 half 类型数据，分别存储到 v0 和 v1 中
    *v0 = vload_half(0, &x[ib + 0]);
    *v1 = vload_half(0, &x[ib + 1]);
}

// 定义一个内联函数，用于计算 scale 和 min 值
static std::string k_quants_source = MULTILINE_QUOTE(
inline void get_scale_min_k4(int j, const __global uint8_t *q, uint8_t *d, uint8_t *m)
{
    // 如果 j 小于 4，则分别计算 d 和 m 的值
    if (j < 4)
    {
        *d = q[j] & 63;
        *m = q[j + 4] & 63;
    }
    // 否则，根据不同情况计算 d 和 m 的值
    else
    {
        *d = (q[j + 4] & 0xF) | ((q[j - 4] >> 6) << 4);
        *m = (q[j + 4] >> 4) | ((q[j - 0] >> 6) << 4);
    }
}

// 定义一个内核函数，用于对输入的 block_q2_K 结构进行反量化操作
__kernel void dequantize_block_q2_K(__global const struct block_q2_K *x, __global float *yy)
{
    // 获取当前工作组的索引和本地索引
    const int i = get_group_id(0) + get_global_offset(0);
    const int tid = get_local_id(0);
    const int n = tid / 32;
    const int l = tid - 32 * n;
    const int is = 8 * n + l / 16;

    // 从 x 中读取量化参数 q
    const uint8_t q = x[i].qs[32 * n + l];
    // 计算 y 的起始位置
    __global float *y = yy + get_group_id(0) * QK_K + 128 * n;

    // 从 x 中读取 d 和 dmin 的值
    const float dall = vload_half(0, &x[i].d);
    const float dmin = vload_half(0, &x[i].dmin);

    // 根据量化参数 q 和 x 中的 scales 计算 y 的值
    y[l + 0] = dall * (x[i].scales[is + 0] & 0xF) * ((q >> 0) & 3) - dmin * (x[i].scales[is + 0] >> 4);
    y[l + 32] = dall * (x[i].scales[is + 2] & 0xF) * ((q >> 2) & 3) - dmin * (x[i].scales[is + 2] >> 4);
    y[l + 64] = dall * (x[i].scales[is + 4] & 0xF) * ((q >> 4) & 3) - dmin * (x[i].scales[is + 4] >> 4);
    y[l + 96] = dall * (x[i].scales[is + 6] & 0xF) * ((q >> 6) & 3) - dmin * (x[i].scales[is + 6] >> 4);
}

// 定义一个内核函数，用于对输入的 block_q3_K 结构进行反量化操作
__kernel void dequantize_block_q3_K(__global const struct block_q3_K *x, __global float *yy)
{
    // 计算 r、i、tid、is0、l0、n、j 等变量的值
    int r = get_local_id(0) / 4;
    int i = get_group_id(0) + get_global_offset(0);
    int tid = r / 2;
    int is0 = r % 2;
    int l0 = 16 * is0 + 4 * (get_local_id(0) % 4);
    int n = tid / 4;
    int j = tid - 4 * n;

    // 计算 m、is、shift 等变量的值
    uint8_t m = 1 << (4 * n + j);
    int is = 8 * n + 2 * j + is0;
    int shift = 2 * j;
}
    // 根据条件判断选择不同的位运算操作，计算出 us 的值
    int8_t us = is < 4 ? (x[i].scales[is - 0] & 0xF) | (((x[i].scales[is + 8] >> 0) & 3) << 4)
              : is < 8 ? (x[i].scales[is - 0] & 0xF) | (((x[i].scales[is + 4] >> 2) & 3) << 4)
              : is < 12  ? (x[i].scales[is - 8] >> 4) | (((x[i].scales[is + 0] >> 4) & 3) << 4)
              : (x[i].scales[is - 8] >> 4) | (((x[i].scales[is - 4] >> 6) & 3) << 4);
    // 计算 dl 的值
    float d_all = vload_half(0, &x[i].d);
    float dl = d_all * (us - 32);

    // 计算 y 的地址
    __global float *y = yy + get_group_id(0) * QK_K + 128 * n + 32 * j;
    // 获取 q 和 hm 的地址
    const __global uint8_t *q = x[i].qs + 32 * n;
    const __global uint8_t *hm = x[i].hmask;

    // 循环遍历 l0 到 l0 + 4 的范围
    for (int l = l0; l < l0 + 4; ++l)
        // 根据位移和掩码计算 y[l] 的值
        y[l] = dl * ((int8_t)((q[l] >> shift) & 3) - ((hm[l] & m) ? 0 : 4));
// 定义一个 OpenCL 内核函数，用于对 Q4 类型的数据块进行反量化
__kernel void dequantize_block_q4_K(__global const struct block_q4_K *x, __global float *yy)
{
    // 获取当前工作组的 ID，并加上全局偏移量得到索引 i
    const int i = get_group_id(0) + get_global_offset(0);
    // 获取当前工作项的本地 ID
    const int tid = get_local_id(0);
    // 计算当前工作项在行和列上的索引
    const int il = tid / 8;
    const int ir = tid % 8;
    // 计算起始索引 is 和数据块大小 n
    const int is = 2 * il;
    const int n = 4;

    // 计算当前工作项对应的输出数组 y 的指针位置
    __global float *y = yy + get_group_id(0) * QK_K + 64 * il + n * ir;

    // 从输入数据结构中加载 d 和 dmin 的值
    const float dall = vload_half(0, &x[i].d);
    const float dmin = vload_half(0, &x[i].dmin);

    // 获取当前工作项对应的量化数据指针 q
    __global const uint8_t *q = x[i].qs + 32 * il + n * ir;

    uint8_t sc, m;
    // 获取当前工作项的缩放因子和最小值
    get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
    float d1 = dall * sc;
    float m1 = dmin * m;
    get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
    float d2 = dall * sc;
    float m2 = dmin * m;
    // 对每个数据进行反量化操作
    for (int l = 0; l < n; ++l)
    {
        y[l + 0] = d1 * (q[l] & 0xF) - m1;
        y[l + 32] = d2 * (q[l] >> 4) - m2;
    }
}

// 定义一个 OpenCL 内核函数，用于对 Q5 类型的数据块进行反量化
__kernel void dequantize_block_q5_K(__global const struct block_q5_K *x, __global float *yy)
{
    // 获取当前工作组的 ID，并加上全局偏移量得到索引 i
    const int i = get_group_id(0) + get_global_offset(0);
    // 获取当前工作项的本地 ID
    const int tid = get_local_id(0);
    // 计算当前工作项在行和列上的索引
    const int il = tid / 16;
    const int ir = tid % 16;
    // 计算起始索引 is
    const int is = 2 * il;

    // 计算当前工作项对应的输出数组 y 的指针位置
    __global float *y = yy + get_group_id(0) * QK_K + 64 * il + 2 * ir;

    // 从输入数据结构中加载 d 和 dmin 的值
    const float dall = vload_half(0, &x[i].d);
    const float dmin = vload_half(0, &x[i].dmin);

    // 获取当前工作项对应的低位量化数据指针 ql 和高位量化数据指针 qh
    __global const uint8_t *ql = x[i].qs + 32 * il + 2 * ir;
    __global const uint8_t *qh = x[i].qh + 2 * ir;

    uint8_t sc, m;
    // 获取当前工作项的缩放因子和最小值
    get_scale_min_k4(is + 0, x[i].scales, &sc, &m);
    const float d1 = dall * sc;
    const float m1 = dmin * m;
    get_scale_min_k4(is + 1, x[i].scales, &sc, &m);
    const float d2 = dall * sc;
    const float m2 = dmin * m;

    uint8_t hm = 1 << (2 * il);
    // 对每个数据进行反量化操作
    y[0] = d1 * ((ql[0] & 0xF) + (qh[0] & hm ? 16 : 0)) - m1;
    y[1] = d1 * ((ql[1] & 0xF) + (qh[1] & hm ? 16 : 0)) - m1;
    hm <<= 1;
    y[32] = d2 * ((ql[0] >> 4) + (qh[0] & hm ? 16 : 0)) - m2;
    y[33] = d2 * ((ql[1] >> 4) + (qh[1] & hm ? 16 : 0)) - m2;
}
// 定义一个 OpenCL 内核函数，用于对 Q6_K 结构体进行反量化操作
__kernel void dequantize_block_q6_K(__global const struct block_q6_K *x, __global float *yy)
{
    // 获取当前工作组的 ID，并加上全局偏移量得到索引 i
    const int i = get_group_id(0) + get_global_offset(0);
    // 获取当前工作项的本地 ID
    const int tid = get_local_id(0);
    // 计算 ip，il，is 的值
    const int ip = tid / 32;
    const int il = tid - 32 * ip;
    const int is = 8 * ip + il / 16;

    // 计算 y 的地址
    __global float *y = yy + get_group_id(0) * QK_K + 128 * ip + il;

    // 从 x 结构体中加载 d 的值
    const float d = vload_half(0, &x[i].d);

    // 获取 ql，qh，sc 的地址
    __global const uint8_t *ql = x[i].ql + 64 * ip + il;
    const uint8_t qh = x[i].qh[32 * ip + il];
    __global const int8_t *sc = x[i].scales + is;

    // 计算 y 的值
    y[0] = d * sc[0] * ((int8_t)((ql[0] & 0xF) | (((qh >> 0) & 3) << 4)) - 32);
    y[32] = d * sc[2] * ((int8_t)((ql[32] & 0xF) | (((qh >> 2) & 3) << 4)) - 32);
    y[64] = d * sc[4] * ((int8_t)((ql[0] >> 4) | (((qh >> 4) & 3) << 4)) - 32);
    y[96] = d * sc[6] * ((int8_t)((ql[32] >> 4) | (((qh >> 6) & 3) << 4)) - 32);
}

// 定义一个 OpenCL 内核函数，用于对 Q2_K 结构体进行矩阵向量乘法和反量化操作
__kernel void dequantize_mul_mat_vec_q2_K(__global const struct block_q2_K * xx, __local float* tmp, __global float* yy, __global float* dst, const int ncols) {

    // 获取当前工作组的 ID，作为行索引
    const int row = get_group_id(0);

    // 计算每行的块数和起始块索引
    const int num_blocks_per_row = ncols / QK_K;
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    // 获取 x 的地址
    __global const struct block_q2_K * x = xx + ib0;

    // 计算 tid，ix，step，im，in，l0，q_offset，s_offset，y_offset 的值
    const int tid = get_local_id(0)/K_QUANTS_PER_ITERATION;  // 0...31 or 0...15
    const int ix  = get_local_id(0)%K_QUANTS_PER_ITERATION;  // 0 or 0,1
    const int step = 16/K_QUANTS_PER_ITERATION;
    const int im = tid/step;                             // 0 or 1. 0 computes 0..., 1 computes 128...
    const int in = tid - step*im;                        // 0...15 or 0...7
    const int l0 = K_QUANTS_PER_ITERATION*in;            // 0...15 or 0...14 in steps of 2
    const int q_offset = 32*im + l0;
    const int s_offset = 8*im;
    const int y_offset = 128*im + l0;

    // 初始化 tmp 数组
    tmp[16 * ix + tid] = 0;

    // 定义辅助数组 aux，以及指向 d 和 m 的指针
    uint32_t aux[4];
    const uint8_t * d = (const uint8_t *)aux;
    const uint8_t * m = (const uint8_t *)(aux + 2);
}
    // 对每一行的数据块进行处理，每次处理 K_QUANTS_PER_ITERATION 个数据块
    for (int i = ix; i < num_blocks_per_row; i += K_QUANTS_PER_ITERATION) {

        // 获取当前行的偏移量
        __global const float   * y = yy + i * QK_K + y_offset;
        // 获取当前行的量化值偏移量
        __global const uint8_t * q = x[i].qs + q_offset;

        // 从 x[i].d 和 x[i].dmin 中加载数据
        const float dall = vload_half(0, &x[i].d);
        const float dmin = vload_half(0, &x[i].dmin);

        // 获取当前行的缩放值偏移量
        __global const uint32_t * a = (__global const uint32_t *)(x[i].scales + s_offset);
        // 对缩放值进行处理，存储到 aux 数组中
        aux[0] = a[0] & 0x0f0f0f0f;
        aux[1] = a[1] & 0x0f0f0f0f;
        aux[2] = (a[0] >> 4) & 0x0f0f0f0f;
        aux[3] = (a[1] >> 4) & 0x0f0f0f0f;

        // 初始化两个求和变量
        float sum1 = 0, sum2 = 0;
        // 遍历 K_QUANTS_PER_ITERATION 个数据块
        for (int l = 0; l < K_QUANTS_PER_ITERATION; ++l) {
            // 计算 sum1
            sum1 += y[l+ 0] * d[0] * ((q[l+ 0] >> 0) & 3)
                  + y[l+32] * d[2] * ((q[l+ 0] >> 2) & 3)
                  + y[l+64] * d[4] * ((q[l+ 0] >> 4) & 3)
                  + y[l+96] * d[6] * ((q[l+ 0] >> 6) & 3)
                  + y[l+16] * d[1] * ((q[l+16] >> 0) & 3)
                  + y[l+48] * d[3] * ((q[l+16] >> 2) & 3)
                  + y[l+80] * d[5] * ((q[l+16] >> 4) & 3)
                  + y[l+112] * d[7] * ((q[l+16] >> 6) & 3);
            // 计算 sum2
            sum2 += y[l+ 0] * m[0] + y[l+32] * m[2] + y[l+64] * m[4] + y[ l+96] * m[6]
                  + y[l+16] * m[1] + y[l+48] * m[3] + y[l+80] * m[5] + y[l+112] * m[7];

        }
        // 计算临时结果并存储到 tmp 数组中
        tmp[16 * ix + tid] += dall * sum1 - dmin * sum2;

    }

    // 对局部和进行求和，并写回结果
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
# 定义一个 OpenCL 内核函数，用于对矩阵向量乘法进行反量化操作
__kernel void dequantize_mul_mat_vec_q3_K(__global const struct block_q3_K * xx, __local float* tmp, __global float* yy, __global float* dst, const int ncols) {
    # 定义两个掩码常量
    const uint16_t kmask1 = 0x0303;
    const uint16_t kmask2 = 0x0f0f;

    # 获取当前工作组的 ID，即行号
    const int row = get_group_id(0);

    # 计算每行中块的数量
    const int num_blocks_per_row = ncols / QK_K;
    # 计算当前块的索引
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    # 获取当前块的指针
    __global const struct block_q3_K * x = xx + ib0;

    # 计算当前线程在块内的索引
    const int tid = get_local_id(0)/K_QUANTS_PER_ITERATION;  # 0...31 or 0...16
    const int ix  = get_local_id(0)%K_QUANTS_PER_ITERATION;  # 0 or 0,1

    # 定义内循环的迭代次数
    const int n  = K_QUANTS_PER_ITERATION;               # iterations in the inner loop
    const int step = 16/K_QUANTS_PER_ITERATION;
    # 计算当前迭代的索引
    const int im = tid/step;                             # 0 or 1. 0 computes 0..., 1 computes 128...
    const int in = tid - step*im;                        # 0....15 or 0...7

    # 计算当前迭代的掩码值
    const uint8_t m = 1 << (4*im);

    # 计算当前迭代的偏移量
    const int l0 = n*in;                                 # 0...15 or 0...14 in steps of 2
    const int q_offset =  32*im + l0;
    const int y_offset = 128*im + l0;

    # 定义一个临时数组用于存储数据
    uint16_t utmp[4];
    const int8_t * s = (const int8_t *)utmp;

    # 计算移位量
    const uint16_t s_shift = 4*im;

    # 初始化临时数组
    tmp[16 * ix + tid] = 0;
    // 对每一行的数据块进行处理，每次处理 K_QUANTS_PER_ITERATION 个数据块
    for (int i = ix; i < num_blocks_per_row; i += K_QUANTS_PER_ITERATION) {

        // 获取当前数据块的相关数据
        __global const float   * y  = yy + i * QK_K + y_offset;
        __global const uint8_t * q = x[i].qs + q_offset;
        __global const uint8_t * h = x[i].hmask + l0;

        // 获取当前数据块的缩放因子
        __global const uint16_t * a = (__global const uint16_t *)x[i].scales;
        utmp[0] = ((a[0] >> s_shift) & kmask2) | (((a[4] >> (s_shift + 0)) & kmask1) << 4);
        utmp[1] = ((a[1] >> s_shift) & kmask2) | (((a[5] >> (s_shift + 0)) & kmask1) << 4);
        utmp[2] = ((a[2] >> s_shift) & kmask2) | (((a[4] >> (s_shift + 2)) & kmask1) << 4);
        utmp[3] = ((a[3] >> s_shift) & kmask2) | (((a[5] >> (s_shift + 2)) & kmask1) << 4);

        // 获取当前数据块的偏移量
        const float d = vload_half(0, &x[i].d);

        // 计算当前数据块的结果
        float sum = 0;
        for (int l = 0; l < n; ++l) {
            sum += y[l+ 0] * (s[0] - 32) * (((q[l] >> 0) & 3) - (h[l] & (m << 0) ? 0 : 4))
                 + y[l+32] * (s[2] - 32) * (((q[l] >> 2) & 3) - (h[l] & (m << 1) ? 0 : 4))
                 + y[l+64] * (s[4] - 32) * (((q[l] >> 4) & 3) - (h[l] & (m << 2) ? 0 : 4))
                 + y[l+96] * (s[6] - 32) * (((q[l] >> 6) & 3) - (h[l] & (m << 3) ? 0 : 4));
            sum += y[l+16] * (s[1] - 32) * (((q[l+16] >> 0) & 3) - (h[l+16] & (m << 0) ? 0 : 4))
                 + y[l+48] * (s[3] - 32) * (((q[l+16] >> 2) & 3) - (h[l+16] & (m << 1) ? 0 : 4))
                 + y[l+80] * (s[5] - 32) * (((q[l+16] >> 4) & 3) - (h[l+16] & (m << 2) ? 0 : 4))
                + y[l+112] * (s[7] - 32) * (((q[l+16] >> 6) & 3) - (h[l+16] & (m << 3) ? 0 : 4));
        }
        tmp[16 * ix + tid] += d * sum;

    }

    // 合并部分和并写回结果
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

// 定义 OpenCL 内核函数，用于对 Q4_K 类型的矩阵向量进行去量化和乘法操作
__kernel void dequantize_mul_mat_vec_q4_K(__global const struct block_q4_K * xx, __local float* tmp, __global float* yy, __global float* dst, const int ncols) {

    // 定义常量，用于后续重命名，目前仅用于测试
    const uint16_t kmask1 = 0x3f3f;
    const uint16_t kmask2 = 0x0f0f;
    const uint16_t kmask3 = 0xc0c0;

    // 获取当前工作组的 ID，即行号
    const int row = get_group_id(0);
    // 计算每行的块数
    const int num_blocks_per_row = ncols / QK_K;
    // 计算当前块的索引
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    // 获取当前线程在工作组中的 ID，并计算出对应的索引
    const int tid = get_local_id(0)/K_QUANTS_PER_ITERATION;  // 0...15
    const int ix  = get_local_id(0)%K_QUANTS_PER_ITERATION;

    // 计算步长
    const int step = 8/K_QUANTS_PER_ITERATION;

    // 计算左右索引
    const int il  = tid/step;     // 0...3
    const int ir  = tid - step*il;// 0...3
    const int n   = 2*K_QUANTS_PER_ITERATION;

    // 计算 m 和 n
    const int im = il/2;  // 0 or 1. 0 computes 0,32 + 128,160, 1 computes 64,96 + 192,224
    const int in = il%2;

    // 计算偏移量
    const int l0 = n*(2*ir + in);
    const int q_offset = 32*im + l0;
    const int y_offset = 64*im + l0;

    // 定义辅助数组和指针
    uint16_t aux[4];
    const uint8_t * sc = (const uint8_t *)aux;

    // 获取当前块的指针
    __global const struct block_q4_K * x = xx + ib0;

    // 初始化临时数组
    tmp[16 * ix + tid] = 0;
    // 对每一行的数据块进行处理，每次处理 K_QUANTS_PER_ITERATION 个数据块
    for (int i = ix; i < num_blocks_per_row; i += K_QUANTS_PER_ITERATION) {

        // 获取第一个量化表和第二个量化表的指针
        __global const uint8_t * q1 = x[i].qs + q_offset;
        __global const uint8_t * q2 = q1 + 64;
        // 获取第一个输出数据和第二个输出数据的指针
        __global const float   * y1 = yy + i*QK_K + y_offset;
        __global const float   * y2 = y1 + 128;

        // 加载 x[i].d 和 x[i].dmin 的值
        const float dall = vload_half(0, &x[i].d);
        const float dmin = vload_half(0, &x[i].dmin);

        // 将 x[i].scales 转换为 uint16_t 类型指针，并根据不同位数进行位运算
        __global const uint16_t * a = (__global const uint16_t *)x[i].scales;
        aux[0] = a[im+0] & kmask1;
        aux[1] = a[im+2] & kmask1;
        aux[2] = ((a[im+4] >> 0) & kmask2) | ((a[im+0] & kmask3) >> 2);
        aux[3] = ((a[im+4] >> 4) & kmask2) | ((a[im+2] & kmask3) >> 2);

        // 初始化 s 和 smin
        float4 s = (float4)(0.f);
        float smin = 0;
        // 遍历 n 个元素，计算 s 和 smin 的值
        for (int l = 0; l < n; ++l) {
            s.x += y1[l] * (q1[l] & 0xF); s.y += y1[l+32] * (q1[l] >> 4);
            s.z += y2[l] * (q2[l] & 0xF); s.w += y2[l+32] * (q2[l] >> 4);
            smin += y1[l] * sc[2] + y1[l+32] * sc[3] + y2[l] * sc[6] + y2[l+32] * sc[7];
        }
        // 计算 tmp 数组中的值
        tmp[16 * ix + tid] += dall * (s.x * sc[0] + s.y * sc[1] + s.z * sc[4] + s.w * sc[5]) - dmin * smin;

    }

    // 对局部和进行求和，并写回结果
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

// 定义一个 OpenCL 内核函数，用于对 Q5_K 类型的矩阵向量乘法进行反量化
__kernel void dequantize_mul_mat_vec_q5_K(__global const struct block_q5_K * xx, __local float* tmp, __global float* yy, __global float* dst, const int ncols) {

    // 定义三个掩码常量
    const uint16_t kmask1 = 0x3f3f;
    const uint16_t kmask2 = 0x0f0f;
    const uint16_t kmask3 = 0xc0c0;

    // 获取当前工作组的 ID 作为行索引
    const int row = get_group_id(0);
    // 计算每行的块数
    const int num_blocks_per_row = ncols / QK_K;
    // 计算当前块的索引
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    // 获取本地 ID 并计算线程 ID 和索引
    const int tid = get_local_id(0)/2;  // 0...15
    const int ix  = get_local_id(0)%2;

    // 计算线程 ID 的左右索引
    const int il  = tid/4;     // 0...3
    const int ir  = tid - 4*il;// 0...3
    const int n   = 2;

    // 计算 im 和 in
    const int im = il/2;  // 0 or 1. 0 computes 0,32 + 128,160, 1 computes 64,96 + 192,224
    const int in = il%2;

    // 计算 l0, q_offset 和 y_offset
    const int l0 = n*(2*ir + in);
    const int q_offset = 32*im + l0;
    const int y_offset = 64*im + l0;

    // 计算 hm1 和 hm2
    const uint8_t hm1  = 1 << (2*im);
    const uint8_t hm2  = hm1 << 4;

    // 定义一个辅助数组和指向该数组的指针
    uint16_t aux[4];
    const uint8_t * sc = (const uint8_t *)aux;

    // 获取当前块的指针
    __global const struct block_q5_K * x = xx + ib0;

    // 初始化临时数组
    tmp[16 * ix + tid] = 0;
    // 对每个奇数索引的块进行处理
    for (int i = ix; i < num_blocks_per_row; i += 2) {

        // 获取当前块的查询序列
        __global const uint8_t * ql1 = x[i].qs + q_offset;
        __global const uint8_t * ql2 = ql1 + 64;
        __global const uint8_t * qh  = x[i].qh + l0;
        __global const float   * y1  = yy + i*QK_K + y_offset;
        __global const float   * y2  = y1 + 128;

        // 加载当前块的 d 和 dmin 值
        const float dall = vload_half(0, &x[i].d);
        const float dmin = vload_half(0, &x[i].dmin);

        // 获取当前块的缩放因子
        __global const uint16_t * a = (__global const uint16_t *)x[i].scales;
        aux[0] = a[im+0] & kmask1;
        aux[1] = a[im+2] & kmask1;
        aux[2] = ((a[im+4] >> 0) & kmask2) | ((a[im+0] & kmask3) >> 2);
        aux[3] = ((a[im+4] >> 4) & kmask2) | ((a[im+2] & kmask3) >> 2);

        // 初始化求和变量和最小值变量
        float4 sum = (float4)(0.f);
        float smin = 0;
        // 遍历当前块的每个元素
        for (int l = 0; l < n; ++l) {
            // 计算加权和
            sum.x += y1[l+ 0] * ((ql1[l+ 0] & 0xF) + (qh[l+ 0] & (hm1 << 0) ? 16 : 0))
                   + y1[l+16] * ((ql1[l+16] & 0xF) + (qh[l+16] & (hm1 << 0) ? 16 : 0));
            sum.y += y1[l+32] * ((ql1[l+ 0] >>  4) + (qh[l+ 0] & (hm1 << 1) ? 16 : 0))
                   + y1[l+48] * ((ql1[l+16] >>  4) + (qh[l+16] & (hm1 << 1) ? 16 : 0));
            sum.z += y2[l+ 0] * ((ql2[l+ 0] & 0xF) + (qh[l+ 0] & (hm2 << 0) ? 16 : 0))
                   + y2[l+16] * ((ql2[l+16] & 0xF) + (qh[l+16] & (hm2 << 0) ? 16 : 0));
            sum.w += y2[l+32] * ((ql2[l+ 0] >>  4) + (qh[l+ 0] & (hm2 << 1) ? 16 : 0))
                   + y2[l+48] * ((ql2[l+16] >>  4) + (qh[l+16] & (hm2 << 1) ? 16 : 0));
            // 计算最小值
            smin += (y1[l] + y1[l+16]) * sc[2] + (y1[l+32] + y1[l+48]) * sc[3]
                  + (y2[l] + y2[l+16]) * sc[6] + (y2[l+32] + y2[l+48]) * sc[7];
        }
        // 更新结果数组
        tmp[16 * ix + tid] += dall * (sum.x * sc[0] + sum.y * sc[1] + sum.z * sc[4] + sum.w * sc[5]) - dmin * smin;

    }

    // 汇总部分和并写回结果
    barrier(CLK_LOCAL_MEM_FENCE);
    # 使用并行计算，循环16次，每次右移1位
    for (int s=16; s>0; s>>=1) {
        # 如果线程ID小于当前步长s，则将当前线程的值与后面相隔s的线程的值相加
        if (tid < s) {
            tmp[tid] += tmp[tid + s];
        }
        # 等待本地内存屏障，确保所有线程完成当前步骤的计算
        barrier(CLK_LOCAL_MEM_FENCE);
    }
    # 如果线程ID为0，则将当前行的结果存入目标数组
    if (tid == 0) {
        dst[row] = tmp[0];
    }
# 定义 OpenCL 内核函数，用于对矩阵和向量进行反量化和乘法操作
__kernel void dequantize_mul_mat_vec_q6_K(__global const struct block_q6_K * xx, __local float* tmp, __global const float * yy, __global float * dst, const int ncols) {

    # 获取当前工作组的 ID，即行号
    const int row = get_group_id(0);

    # 计算每行包含的块数
    const int num_blocks_per_row = ncols / QK_K;
    # 计算当前块的索引
    const int ib0 = row*num_blocks_per_row + get_global_offset(0);

    # 获取当前块的指针
    __global const struct block_q6_K * x = xx + ib0;

    # 计算当前线程在 warp 中的索引
    const int tid = get_local_id(0)/K_QUANTS_PER_ITERATION;  // 0...31 or 0...16
    # 计算当前线程在 warp 中的索引
    const int ix  = get_local_id(0)%K_QUANTS_PER_ITERATION;  // 0 or 0, 1

    # 计算步长
    const int step = 16/K_QUANTS_PER_ITERATION;          // 16 or 8

    # 计算 m 维度的索引
    const int im = tid/step;                             // 0 or 1. 0 computes 0..., 1 computes 128...
    # 计算 n 维度的索引
    const int in = tid - step*im;                        // 0...15 or 0...7

    # 根据 K_QUANTS_PER_ITERATION 的值选择不同的计算方式
    # 如果 K_QUANTS_PER_ITERATION 为 1
    const int l0 = K_QUANTS_PER_ITERATION*in;            // 0...15
    const int is = 0;

    # 如果 K_QUANTS_PER_ITERATION 不为 1
    const int l0 = 4 * in;                               // 0, 4, 8, ..., 28
    const int is = in / 4;

    # 计算 ql、qh、s、y 的偏移量
    const int ql_offset = 64*im + l0;
    const int qh_offset = 32*im + l0;
    const int s_offset  =  8*im + is;
    const int y_offset = 128*im + l0;

    # 初始化临时数组中当前线程的部分和为 0
    tmp[16 * ix + tid] = 0; // partial sum for thread in warp

    # 遍历每个块
    for (int i = ix; i < num_blocks_per_row; i += K_QUANTS_PER_ITERATION) {

        # 获取当前块的相关数据指针
        __global const float   * y  = yy + i * QK_K + y_offset;
        __global const uint8_t * ql = x[i].ql + ql_offset;
        __global const uint8_t * qh = x[i].qh + qh_offset;
        __global const int8_t  * s  = x[i].scales + s_offset;

        # 加载当前块的 d 值
        const float d = vload_half(0, &x[i].d);
\n#if K_QUANTS_PER_ITERATION == 1\n
        // 如果每次迭代只处理一个量化值
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
        // 如果每次迭代处理多个量化值
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
// 定义了一个模板字符串，用于生成矩阵向量乘法的 OpenCL 内核函数
std::string dequant_mul_mat_vec_template = MULTILINE_QUOTE(
__kernel void KERNEL_NAME(__global X_TYPE* x, __local float* tmp, __global float* y, __global float* dst, const int ncols) {
    // 获取本地工作组大小
    const int local_size = get_local_size(0);
    // 获取当前组的索引
    const int row = get_group_id(0);
    // 获取本地 ID
    const int tid = get_local_id(0);

    // 定义量化参数
    const uint qk = QUANT_K;
    const uint qr = QUANT_R;

    // 计算列步长
    const int col_step = local_size * 2;
    // 计算 y 偏移量
    const int y_offset = qr == 1 ? 1 : qk/2;

    // 更新 x 指针
    x += get_global_offset(0);

    // 初始化临时变量
    tmp[tid] = 0;

    // 循环计算矩阵向量乘法
    for (int col = tid*2; col < ncols; col += col_step) {
        // 计算块索引
        const int ib = (row*ncols + col)/qk;
        // 计算量化索引
        const int iqs = (col%qk)/qr;
        // 计算 y 块起始索引
        const int iybs = col - col%qk;

        // 反量化
        float v0, v1;
        DEQUANT_FUNC(x, ib, iqs, &v0, &v1);

        // 矩阵乘法
        tmp[tid] += v0 * y[iybs + iqs + 0];
        tmp[tid] += v1 * y[iybs + iqs + y_offset];
    }

    // 合并局部和并写回结果
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

// 定义了一个模板字符串，用于生成矩阵乘法的 OpenCL 内核函数
std::string mul_template = MULTILINE_QUOTE(
__kernel void KERNEL_NAME(__global TYPE* x, const int x_offset, __global TYPE* y, const int y_offset, __global TYPE* dst, const int dst_offset, const int ky) {
    // 获取全局索引
    const int i = get_group_id(0)*get_local_size(0) + get_local_id(0);

    // 如果索引超出范围则返回
    if (i >= get_global_size(0)) {
        return;
    }

    // 计算矩阵乘法并写回结果
    dst[dst_offset + i] = x[x_offset + i] * y[y_offset + i%ky];
}
);

// 定义了一个模板字符串，用于生成加法的 OpenCL 内核函数
std::string add_template = MULTILINE_QUOTE(
__kernel void add_f32(__global float * x, const int x_offset, __global float * y, const int y_offset, __global float * dst, const int dst_offset, const int ky) {
    // 获取全局索引
    const int i = get_group_id(0)*get_local_size(0) + get_local_id(0);
    # 如果当前索引超过全局大小，则直接返回，不进行后续操作
    if (i >= get_global_size(0)) {
        return;
    }

    # 将 x 数组和 y 数组对应索引位置的值相加，存储到 dst 数组对应索引位置
    dst[dst_offset + i] = x[x_offset + i] + y[y_offset + i%ky];
// 定义一个宏，用于检查 OpenCL 函数调用是否成功，如果失败则输出错误信息并退出程序
#define CL_CHECK(err)                                               \
    do {                                                            \
        cl_int err_ = (err);                                        \
        if (err_ != CL_SUCCESS) {                                   \
            fprintf(stderr, "ggml_opencl: %s error %d at %s:%d\n",  \
                #err, err_, __FILE__, __LINE__);                    \
            exit(1);                                                \
        }                                                           \
    } while (0)

// 定义一个宏，用于检查 CLBlast 函数调用是否成功，如果失败则输出错误信息并退出程序
#define CLBLAST_CHECK(err)                                          \
    do {                                                            \
        CLBlastStatusCode err_ = (err);                             \
        if (err_ != CLBlastSuccess) {                               \
            fprintf(stderr, "ggml_opencl: %s error %d at %s:%d\n",  \
                #err, err_, __FILE__, __LINE__);                    \
            exit(1);                                                \
        }                                                           \
    } while (0)

// 定义一个包含 5 个字符串的数组，用于存储解量化操作所需的关键字
std::array<std::string, 5> dequant_str_keys = {
    "KERNEL_NAME", "X_TYPE", "QUANT_K", "QUANT_R", "DEQUANT_FUNC"
};

// 定义一个包含 30 个字符串的数组，用于存储解量化操作所需的关键值
std::array<std::string, 30> dequant_str_values = {
    "dequantize_row_q4_0", "struct block_q4_0", "QK4_0", "QR4_0", "dequantize_q4_0",
    "dequantize_row_q4_1", "struct block_q4_1", "QK4_1", "QR4_1", "dequantize_q4_1",
    "dequantize_row_q5_0", "struct block_q5_0", "QK5_0", "QR5_0", "dequantize_q5_0",
    "dequantize_row_q5_1", "struct block_q5_1", "QK5_1", "QR5_1", "dequantize_q5_1",
    "dequantize_row_q8_0", "struct block_q8_0", "QK8_0", "QR8_0", "dequantize_q8_0",
    "convert_row_f16", "half", "1", "1", "convert_f16"
};

// 定义一个包含 30 个字符串的数组，用于存储解量化操作中矩阵向量乘法所需的关键值
std::array<std::string, 30> dequant_mul_mat_vec_str_values = {
    "dequantize_mul_mat_vec_q4_0", "struct block_q4_0", "QK4_0", "QR4_0", "dequantize_q4_0",
    # 定义一系列字符串，包括函数名、结构体名、常量等
    "dequantize_mul_mat_vec_q4_1", "struct block_q4_1", "QK4_1", "QR4_1", "dequantize_q4_1",
    "dequantize_mul_mat_vec_q5_0", "struct block_q5_0", "QK5_0", "QR5_0", "dequantize_q5_0",
    "dequantize_mul_mat_vec_q5_1", "struct block_q5_1", "QK5_1", "QR5_1", "dequantize_q5_1",
    "dequantize_mul_mat_vec_q8_0", "struct block_q8_0", "QK8_0", "QR8_0", "dequantize_q8_0",
    "convert_mul_mat_vec_f16", "half", "1", "1", "convert_f16"
// 定义包含两个字符串的数组，用于存储多个字符串键
std::array<std::string, 2> mul_str_keys = {
    "KERNEL_NAME", "TYPE"
};
// 定义包含两个字符串的数组，用于存储多个字符串值
std::array<std::string, 2> mul_str_values = {
    "mul_f32", "float"
};

// 替换字符串中的指定内容
static std::string& replace(std::string& s, const std::string& from, const std::string& to) {
    size_t pos = 0;
    while ((pos = s.find(from, pos)) != std::string::npos) {
         s.replace(pos, from.length(), to);
         pos += to.length();
    }
    return s;
}

// 生成内核代码
static std::string generate_kernels() {
    // 创建字符串流对象
    std::stringstream src;
    // 将程序源码和量化源码添加到字符串流中
    src << program_source << '\n';
    src << k_quants_source << '\n';
    // 遍历量化字符串值数组
    for (size_t i = 0; i < dequant_str_values.size(); i += dequant_str_keys.size()) {
        // 使用模板替换量化内核字符串中的键值对
        std::string dequant_kernel = dequant_template;
        std::string dmmv_kernel = dequant_mul_mat_vec_template;
        for (size_t j = 0; j < dequant_str_keys.size(); j++) {
            replace(dequant_kernel, dequant_str_keys[j], dequant_str_values[i + j]);
            replace(dmmv_kernel, dequant_str_keys[j], dequant_mul_mat_vec_str_values[i + j]);
        }
        // 将替换后的内核字符串添加到字符串流中
        src << dequant_kernel << '\n';
        src << dmmv_kernel << '\n';
    }
    // 遍历乘法字符串值数组
    for (size_t i = 0; i < mul_str_values.size(); i += mul_str_keys.size()) {
        // 使用模板替换乘法内核字符串中的键值对
        std::string mul_kernel = mul_template;
        for (size_t j = 0; j < mul_str_keys.size(); j++) {
            replace(mul_kernel, mul_str_keys[j], mul_str_values[i + j]);
        }
        // 将替换后的内核字符串添加到字符串流中
        src << mul_kernel << '\n';
    }
    // 将加法模板添加到字符串流中
    src << add_template << '\n';

    return src.str();
}

// 静态变量声明
static cl_platform_id platform;
static cl_device_id device;
static cl_context context;
static cl_command_queue queue;
static cl_program program;
static cl_kernel convert_row_f16_cl;
static cl_kernel dequantize_row_q4_0_cl, dequantize_row_q4_1_cl, dequantize_row_q5_0_cl, dequantize_row_q5_1_cl, dequantize_row_q8_0_cl;
static cl_kernel dequantize_mul_mat_vec_q4_0_cl, dequantize_mul_mat_vec_q4_1_cl, dequantize_mul_mat_vec_q5_0_cl, dequantize_mul_mat_vec_q5_1_cl, dequantize_mul_mat_vec_q8_0_cl, convert_mul_mat_vec_f16_cl;
// 定义多个静态的 OpenCL 内核对象
static cl_kernel dequantize_block_q2_k_cl, dequantize_block_q3_k_cl, dequantize_block_q4_k_cl, dequantize_block_q5_k_cl, dequantize_block_q6_k_cl;
static cl_kernel dequantize_mul_mat_vec_q2_K_cl, dequantize_mul_mat_vec_q3_K_cl, dequantize_mul_mat_vec_q4_K_cl, dequantize_mul_mat_vec_q5_K_cl, dequantize_mul_mat_vec_q6_K_cl;
static cl_kernel mul_f32_cl;
static cl_kernel add_f32_cl;
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
    static bool initialized = false;
    if (initialized) {
        return;
    }
    initialized = true;

    cl_int err;

    struct cl_device;
}
    // 定义 OpenCL 平台结构体，包含平台 ID、编号、名称、供应商、设备列表、设备数量和默认设备
    struct cl_platform {
        cl_platform_id id;
        unsigned number;
        char name[128];
        char vendor[128];
        struct cl_device * devices;
        unsigned n_devices;
        struct cl_device * default_device;
    };

    // 定义 OpenCL 设备结构体，包含平台指针、设备 ID、编号、设备类型和名称
    struct cl_device {
        struct cl_platform * platform;
        cl_device_id id;
        unsigned number;
        cl_device_type type;
        char name[128];
    };

    // 定义常量，表示最大平台数和最大设备数
    enum { NPLAT = 16, NDEV = 16 };

    // 创建存储平台信息的数组和平台数量变量
    struct cl_platform platforms[NPLAT];
    unsigned n_platforms = 0;
    // 创建存储设备信息的数组和设备数量变量
    struct cl_device devices[NDEV];
    unsigned n_devices = 0;
    // 创建指向默认设备的指针，默认为 NULL
    struct cl_device * default_device = NULL;

    // 初始化平台和设备指针
    platform = NULL;
    device = NULL;

    // 创建用于存储平台 ID 的数组，并通过 clGetPlatformIDs 函数获取平台信息
    cl_platform_id platform_ids[NPLAT];
    CL_CHECK(clGetPlatformIDs(NPLAT, platform_ids, &n_platforms));
    // 遍历平台列表
    for (unsigned i = 0; i < n_platforms; i++) {
        // 获取当前平台的指针
        struct cl_platform * p = &platforms[i];
        // 设置平台的编号和 ID
        p->number = i;
        p->id = platform_ids[i];
        // 获取平台的名称和供应商信息
        CL_CHECK(clGetPlatformInfo(p->id, CL_PLATFORM_NAME, sizeof(p->name), &p->name, NULL));
        CL_CHECK(clGetPlatformInfo(p->id, CL_PLATFORM_VENDOR, sizeof(p->vendor), &p->vendor, NULL));

        // 获取平台的设备列表
        cl_device_id device_ids[NDEV];
        cl_int clGetDeviceIDsError = clGetDeviceIDs(p->id, CL_DEVICE_TYPE_ALL, NDEV, device_ids, &p->n_devices);
        // 处理设备列表为空的情况
        if (clGetDeviceIDsError == CL_DEVICE_NOT_FOUND) {
            p->n_devices = 0;
        } else {
            CL_CHECK(clGetDeviceIDsError);
        }
        // 设置平台的设备列表和默认设备
        p->devices = p->n_devices > 0 ? &devices[n_devices] : NULL;
        p->default_device = NULL;

        // 遍历设备列表
        for (unsigned j = 0; j < p->n_devices; j++) {
            // 获取当前设备的指针
            struct cl_device * d = &devices[n_devices];
            // 设置设备的编号、ID和所属平台
            d->number = n_devices++;
            d->id = device_ids[j];
            d->platform = p;
            // 获取设备的名称和类型信息
            CL_CHECK(clGetDeviceInfo(d->id, CL_DEVICE_NAME, sizeof(d->name), &d->name, NULL));
            CL_CHECK(clGetDeviceInfo(d->id, CL_DEVICE_TYPE, sizeof(d->type), &d->type, NULL));

            // 设置默认设备为第一个 GPU 设备
            if (p->default_device == NULL && d->type == CL_DEVICE_TYPE_GPU) {
                p->default_device = d;
            }
        }

        // 设置全局默认设备为当前平台的默认设备
        if (default_device == NULL && p->default_device != NULL) {
            default_device = p->default_device;
        }
    }

    // 检查是否找到了 OpenCL 设备，否则输出错误信息并退出程序
    if (n_devices == 0) {
        fprintf(stderr, "ggml_opencl: could find any OpenCL devices.\n");
        exit(1);
    }

    // 获取用户设置的平台和设备信息
    char * user_platform_string = getenv("GGML_OPENCL_PLATFORM");
    char * user_device_string = getenv("GGML_OPENCL_DEVICE");
    int user_platform_number = -1;
    int user_device_number = -1;

    unsigned n;
    // 解析用户设置的平台信息
    if (user_platform_string != NULL && sscanf(user_platform_string, " %u", &n) == 1 && n < n_platforms) {
        user_platform_number = (int)n;
    }
    // 检查用户提供的设备字符串是否不为空，并且成功解析为无符号整数，并且小于设备数量
    if (user_device_string != NULL && sscanf(user_device_string, " %u", &n) == 1 && n < n_devices) {
        // 将解析得到的设备号转换为整数
        user_device_number = (int)n;
    }
    // 如果用户提供的平台号和设备号都有效
    if (user_platform_number != -1 && user_device_number != -1) {
        // 获取用户选择的平台
        cl_platform* platform = &platforms[user_platform_number];
        // 如果用户选择的设备号超出平台所包含设备数量
        if ((unsigned)user_device_number >= platform->n_devices) {
            // 输出错误信息并退出程序
            fprintf(stderr, "ggml_opencl: invalid device number %d\n", user_device_number);
            exit(1);
        }
        // 获取用户选择的设备
        default_device = &platform->devices[user_device_number];
    }

    // 输出选择的平台和设备信息
    fprintf(stderr, "ggml_opencl: selecting platform: '%s'\n", default_device->platform->name);
    fprintf(stderr, "ggml_opencl: selecting device: '%s'\n", default_device->name);
    // 如果选择的设备不是 GPU 类型，输出警告信息
    if (default_device->type != CL_DEVICE_TYPE_GPU) {
        fprintf(stderr, "ggml_opencl: warning, not a GPU: '%s'.\n", default_device->name);
    }

    // 设置平台和设备
    platform = default_device->platform->id;
    device = default_device->id;

    // 获取设备支持的扩展信息
    size_t ext_str_size;
    clGetDeviceInfo(device, CL_DEVICE_EXTENSIONS, 0, NULL, &ext_str_size);
    char *ext_buffer = (char *)alloca(ext_str_size + 1);
    clGetDeviceInfo(device, CL_DEVICE_EXTENSIONS, ext_str_size, ext_buffer, NULL);
    ext_buffer[ext_str_size] = '\0'; // 确保以空字符结尾
    // 检查设备是否支持 cl_khr_fp16，暂时禁用
    fp16_support = false;  // strstr(ext_buffer, "cl_khr_fp16") != NULL;

    // 设置 OpenCL 上下文属性
    cl_context_properties properties[] = {
        (intptr_t)CL_CONTEXT_PLATFORM, (intptr_t)platform, 0
    };

    // 创建 OpenCL 上下文
    CL_CHECK((context = clCreateContext(properties, 1, &device, NULL, NULL, &err), err));

    // 创建 OpenCL 命令队列
    CL_CHECK((queue = clCreateCommandQueue(context, device, CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE, &err),
        (err != CL_INVALID_QUEUE_PROPERTIES && err != CL_INVALID_VALUE ? err :
        (queue = clCreateCommandQueue(context, device, 0, &err), err)
    ));
    // 生成包含内核的字符串
    const std::string kernel_src = generate_kernels();

    // 从源代码构建程序
    program = build_program_from_source(context, device, kernel_src.c_str());

    // 创建 FP16 到 FP32 的内核
    CL_CHECK((convert_row_f16_cl = clCreateKernel(program, "convert_row_f16", &err), err));

    // 创建反量化内核
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

    // 创建反量化乘矩阵向量的内核
    CL_CHECK((dequantize_mul_mat_vec_q4_0_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q4_0", &err), err));
    CL_CHECK((dequantize_mul_mat_vec_q4_1_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q4_1", &err), err));
    CL_CHECK((dequantize_mul_mat_vec_q5_0_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q5_0", &err), err));
    CL_CHECK((dequantize_mul_mat_vec_q5_1_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q5_1", &err), err));
    // 创建用于执行 dequantize_mul_mat_vec_q8_0 的 OpenCL 内核
    CL_CHECK((dequantize_mul_mat_vec_q8_0_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q8_0", &err), err));
    // 创建用于执行 convert_mul_mat_vec_f16 的 OpenCL 内核
    CL_CHECK((convert_mul_mat_vec_f16_cl = clCreateKernel(program, "convert_mul_mat_vec_f16", &err), err));
    // 创建用于执行 dequantize_mul_mat_vec_q2_K 的 OpenCL 内核
    CL_CHECK((dequantize_mul_mat_vec_q2_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q2_K", &err), err));
    // 创建用于执行 dequantize_mul_mat_vec_q3_K 的 OpenCL 内核
    CL_CHECK((dequantize_mul_mat_vec_q3_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q3_K", &err), err));
    // 创建用于执行 dequantize_mul_mat_vec_q4_K 的 OpenCL 内核
    CL_CHECK((dequantize_mul_mat_vec_q4_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q4_K", &err), err));
    // 创建用于执行 dequantize_mul_mat_vec_q5_K 的 OpenCL 内核
    CL_CHECK((dequantize_mul_mat_vec_q5_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q5_K", &err), err));
    // 创建用于执行 dequantize_mul_mat_vec_q6_K 的 OpenCL 内核
    CL_CHECK((dequantize_mul_mat_vec_q6_K_cl = clCreateKernel(program, "dequantize_mul_mat_vec_q6_K", &err), err));

    // 创建用于执行 mul_f32 的 OpenCL 内核
    CL_CHECK((mul_f32_cl = clCreateKernel(program, "mul_f32", &err), err));

    // 创建用于执行 add_f32 的 OpenCL 内核
    CL_CHECK((add_f32_cl = clCreateKernel(program, "add_f32", &err), err));
// 根据输入的类型返回对应的 OpenCL 内核指针
static cl_kernel* ggml_get_to_fp32_cl(ggml_type type) {
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

// 根据输入的类型返回全局工作组大小的分母
static size_t ggml_cl_global_denom(ggml_type type) {
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

// 根据输入的类型返回本地工作组大小
static size_t ggml_cl_local_size(ggml_type type) {
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

// 定义一个结构体 scoped_spin_lock，用于实现自旋锁
struct scoped_spin_lock {
    std::atomic_flag& lock;
    scoped_spin_lock(std::atomic_flag& lock) : lock(lock) {
        // 自旋等待锁释放
        while (lock.test_and_set(std::memory_order_acquire)) {
            ; // spin
        }
    }
    // 析构函数，在作用域结束时释放锁
    ~scoped_spin_lock() {
        lock.clear(std::memory_order_release);
    }
    // 禁止拷贝构造函数和赋值运算符
    scoped_spin_lock(const scoped_spin_lock&) = delete;
    scoped_spin_lock& operator=(const scoped_spin_lock&) = delete;
};

// 定义一个结构体 cl_buffer，用于表示 OpenCL 缓冲区
struct cl_buffer {
    cl_mem mem; // OpenCL 内存对象
    size_t size = 0; // 缓冲区大小，默认为0
};

// 定义一个固定大小的 OpenCL 缓冲区池
static cl_buffer g_cl_buffer_pool[MAX_CL_BUFFERS];
// 定义一个原子标志，用于控制缓冲区池的访问
static std::atomic_flag g_cl_pool_lock = ATOMIC_FLAG_INIT;

// 分配 OpenCL 缓冲区的函数
static cl_mem ggml_cl_pool_malloc(size_t size, size_t * actual_size) {
    // 获取缓冲区池的锁
    scoped_spin_lock lock(g_cl_pool_lock);
    cl_int err;

    int best_i = -1; // 最佳缓冲区索引
    size_t best_size = std::numeric_limits<size_t>::max(); // 最适合需求的最小未使用缓冲区大小
    int worst_i = -1; // 最差缓冲区索引
    size_t worst_size = 0; //记录目前为止最大的未使用缓冲区大小
    for (int i = 0; i < MAX_CL_BUFFERS; ++i) {
        cl_buffer &b = g_cl_buffer_pool[i]; //获取第i个缓冲区的引用
        if (b.size > 0 && b.size >= size && b.size < best_size)
        {
            best_i = i; //记录最适合大小的缓冲区索引
            best_size = b.size; //更新最适合大小的缓冲区大小
        }
        if (b.size > 0 && b.size > worst_size)
        {
            worst_i = i; //记录最大的缓冲区索引
            worst_size = b.size; //更新最大的缓冲区大小
        }
    }
    if(best_i!=-1) //找到了最适合我们需求的最小缓冲区
    {
        cl_buffer& b = g_cl_buffer_pool[best_i]; //获取最适合大小的缓冲区的引用
        cl_mem mem = b.mem; //获取缓冲区对应的内存对象
        *actual_size = b.size; //更新实际大小
        b.size = 0; //重置缓冲区大小
        return mem; //返回内存对象
    }
    if(worst_i!=-1) //没有符合我们需求的缓冲区，将最大的缓冲区大小调整以节省内存
    {
         cl_buffer& b = g_cl_buffer_pool[worst_i]; //获取最大的缓冲区的引用
         cl_mem mem = b.mem; //获取缓冲区对应的内存对象
         b.size = 0; //重置缓冲区大小
         clReleaseMemObject(mem); //释放内存对象
    }
    cl_mem mem; //声明内存对象
    CL_CHECK((mem = clCreateBuffer(context, CL_MEM_READ_WRITE, size, NULL, &err), err)); //创建新的内存对象
    *actual_size = size; //更新实际大小
    return mem; //返回内存对象
}

// 释放 OpenCL 内存对象并将其放回内存池
static void ggml_cl_pool_free(cl_mem mem, size_t size) {
    // 创建一个作用域自旋锁对象
    scoped_spin_lock lock(g_cl_pool_lock);

    // 遍历内存池中的缓冲区
    for (int i = 0; i < MAX_CL_BUFFERS; ++i) {
        // 获取当前缓冲区对象
        cl_buffer& b = g_cl_buffer_pool[i];
        // 如果缓冲区大小为 0，则将传入的内存对象和大小放入该缓冲区，并返回
        if (b.size == 0) {
            b.mem = mem;
            b.size = size;
            return;
        }
    }
    // 如果内存池已满，则输出警告信息
    fprintf(stderr, "WARNING: cl buffer pool full, increase MAX_CL_BUFFERS\n");
    // 释放传入的内存对象
    clReleaseMemObject(mem);
}

// 释放 OpenCL 内存对象
void ggml_cl_free_data(const struct ggml_tensor* tensor) {
    // 如果张量的后端不是 GPU，则直接返回
    if (tensor->backend != GGML_BACKEND_GPU) {
        return;
    }

    // 将张量的额外数据转换为 cl_mem 类型
    cl_mem mem = (cl_mem)tensor->extra;
    // 释放内存对象
    clReleaseMemObject(mem);
}

// 将二维张量数据从主机内存复制到设备内存
static cl_int ggml_cl_h2d_tensor_2d(cl_command_queue queue, cl_mem dst, size_t offset, const struct ggml_tensor * src, uint64_t i3, uint64_t i2, cl_event* ev) {
    cl_int err;
    // 获取张量的维度信息和数据类型
    const uint64_t ne0 = src->ne[0];
    const uint64_t ne1 = src->ne[1];
    const uint64_t nb0 = src->nb[0];
    const uint64_t nb1 = src->nb[1];
    const uint64_t nb2 = src->nb[2];
    const uint64_t nb3 = src->nb[3];
    const enum ggml_type type = src->type;
    const size_t ts = ggml_type_size(type);
    const size_t bs = ggml_blck_size(type);
    const uint64_t row_size = ts*ne0/bs;

    // 计算数据在源张量中的偏移量
    const char * x = (const char *) src->data + i2*nb2 + i3*nb3;
    // 如果数据在源张量中是连续存储的，则直接写入到目标内存
    if (nb0 == ts && nb1 == row_size) {
        return clEnqueueWriteBuffer(queue, dst, CL_FALSE, offset, ne1*row_size, x, 0, NULL, ev);
    }
    // 如果数据在源张量中不是连续存储的，则使用矩形写入方式
    if (nb0 == ts) {
        const size_t buffer_origin[3] = { offset, 0, 0 };
        const size_t host_origin[3] = { 0, 0, 0 };
        const size_t region[3] = { row_size, ne1, 1 };
        return clEnqueueWriteBufferRect(queue, dst, CL_FALSE, buffer_origin, host_origin, region, row_size, 0, nb1, 0, x, 0, NULL, ev);
    }
    // 如果数据在源张量中不是连续存储的且需要分块写入，则使用事件数组来保存事件
    std::vector<cl_event> events;
    if (ev && ne1>1) events.reserve(ne1-1);
}
    // 遍历第一个维度的元素
    for (uint64_t i1 = 0; i1 < ne1; i1++) {
        // 假设行是一个列数为1的矩阵
        const size_t buffer_origin[3] = { offset + i1*row_size, 0, 0 };
        const size_t host_origin[3] = { 0, 0, 0 };
        const size_t region[3] = { ts, ne0/bs, 1 };
        // 如果请求了事件，使最后一个写操作等待所有先前的写操作完成
        if (ev && i1) {
            events.push_back(*ev);
        }
        cl_uint nevents = i1 == ne1-1 ? events.size() : 0U;
        // 将数据从主机内存复制到设备缓冲区
        err = clEnqueueWriteBufferRect(queue, dst, CL_FALSE, buffer_origin, host_origin, region, ts, 0, nb0, 0, x + i1*nb1, nevents, nevents ? events.data() : nullptr, ev);
        // 如果写操作失败，释放事件并返回错误码
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
    // 返回成功状态
    return CL_SUCCESS;
// 定义一个静态函数，用于在 GPU 上计算两个浮点数张量的乘积并存储到目标张量中
static void ggml_cl_mul_f32(const ggml_tensor * src0, const ggml_tensor * src1, ggml_tensor * dst) {
    // 断言 src1 的后端为 GPU
    GGML_ASSERT(src1->backend == GGML_BACKEND_GPU);
    // 获取 src0 的各维度大小
    const int64_t ne00 = src0->ne[0];
    const int64_t ne01 = src0->ne[1];
    const int64_t ne02 = src0->ne[2];
    const int64_t ne03 = src0->ne[3];
    // 获取 src1 的各维度大小
    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
    const int64_t ne13 = src1->ne[3];
    // 获取目标张量的第二维和第三维大小
    const int nb2  = dst->nb[2];
    const int nb3  = dst->nb[3];
    // 定义变量用于存储数据大小
    size_t x_size;
    size_t d_size;

    // 在 GPU 内存中分配空间用于存储 src0 的数据，并返回数据大小
    cl_mem d_X = ggml_cl_pool_malloc(ne00 * ne01 * sizeof(float), &x_size); // src0
    // 将 src1 的额外数据转换为 cl_mem 类型，表示 src1 已经在设备上，已经广播
    cl_mem d_Y = (cl_mem) src1->extra; // src1 is already on device, broadcasted.
    // 在 GPU 内存中分配空间用于存储目标张量的数据，并返回数据大小
    cl_mem d_D = ggml_cl_pool_malloc(ne00 * ne01 * sizeof(float), &d_size); // dst
    // 外层循环，遍历 i03 从 0 到 ne03
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        // 内层循环，遍历 i02 从 0 到 ne02
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            // 定义 OpenCL 事件对象
            cl_event ev;

            // 将 src0 的数据复制到设备中
            CL_CHECK(ggml_cl_h2d_tensor_2d(queue, d_X, 0, src0, i03, i02, &ev));

            // 计算 i13 和 i12
            const int64_t i13 = i03%ne13;
            const int64_t i12 = i02%ne12;
            const int i1 = i13*ne12*ne11 + i12*ne11;

            // 设置偏移量
            cl_int x_offset = 0;
            cl_int y_offset = i1*ne10;
            cl_int d_offset = 0;

            // 计算全局大小和 ky
            size_t global = ne00 * ne01;
            cl_int ky = ne10 * ne11;

            // 设置 OpenCL 内核参数
            CL_CHECK(clSetKernelArg(mul_f32_cl, 0, sizeof(cl_mem), &d_X));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 1, sizeof(cl_int), &x_offset));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 2, sizeof(cl_mem), &d_Y));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 3, sizeof(cl_int), &y_offset));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 4, sizeof(cl_mem), &d_D));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 5, sizeof(cl_int), &d_offset));
            CL_CHECK(clSetKernelArg(mul_f32_cl, 6, sizeof(cl_int), &ky));
            // 执行 OpenCL 内核
            CL_CHECK(clEnqueueNDRangeKernel(queue, mul_f32_cl, 1, NULL, &global, NULL, 1, &ev, NULL));

            // 释放事件对象
            CL_CHECK(clReleaseEvent(ev));
            // 等待队列执行完毕
            CL_CHECK(clFinish(queue));

            // 将结果数据复制到主机
            float * d = (float *) ((char *) dst->data + i02*nb2 + i03*nb3);
            CL_CHECK(clEnqueueReadBuffer(queue, d_D, true, 0, sizeof(float) * ne00*ne01, d, 0, NULL, NULL));
        }
    }
    // 释放内存池中的设备内存
    ggml_cl_pool_free(d_X, x_size);
    ggml_cl_pool_free(d_D, d_size);
// 乘法操作，计算两个张量的乘积并存储到目标张量中
void ggml_cl_mul(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst) {
    // 断言输入张量的类型都为 GGML_TYPE_F32，并且目标张量的类型也为 GGML_TYPE_F32
    GGML_ASSERT(src0->type == GGML_TYPE_F32 && src1->type == GGML_TYPE_F32 && dst->type == GGML_TYPE_F32);
    // 调用 ggml_cl_mul_f32 函数进行浮点数类型的张量乘法操作
    ggml_cl_mul_f32(src0, src1, dst);
}

// 静态函数，用于在 GPU 上执行浮点数类型的张量加法操作
static void ggml_cl_add_f32(const ggml_tensor * src0, const ggml_tensor * src1, ggml_tensor * dst) {
    // 断言输入张量 src1 的后端为 GGML_BACKEND_GPU
    GGML_ASSERT(src1->backend == GGML_BACKEND_GPU);
    // 获取输入张量 src0 和 src1 的各维度大小
    const int64_t ne00 = src0->ne[0];
    const int64_t ne01 = src0->ne[1];
    const int64_t ne02 = src0->ne[2];
    const int64_t ne03 = src0->ne[3];
    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
    const int64_t ne13 = src1->ne[3];
    // 获取目标张量 dst 的第二维和第三维大小
    const int nb2  = dst->nb[2];
    const int nb3  = dst->nb[3];
    // 定义变量用于存储数据大小
    size_t x_size;
    size_t d_size;

    // 在 GPU 上分配内存用于存储 src0 的数据，并返回内存对象 d_X
    cl_mem d_X = ggml_cl_pool_malloc(ne00 * ne01 * sizeof(float), &x_size); // src0
    // 将 src1 的额外数据转换为 cl_mem 类型，并赋值给 d_Y，表示 src1 已经在设备上，已经广播
    cl_mem d_Y = (cl_mem) src1->extra; // src1 is already on device, broadcasted.
    // 在 GPU 上分配内存用于存储 dst 的数据，并返回内存对象 d_D
    cl_mem d_D = ggml_cl_pool_malloc(ne00 * ne01 * sizeof(float), &d_size); // dst
    // 循环遍历 i03，范围是 [0, ne03)
    for (int64_t i03 = 0; i03 < ne03; i03++) {
        // 循环遍历 i02，范围是 [0, ne02)
        for (int64_t i02 = 0; i02 < ne02; i02++) {
            // 创建 OpenCL 事件对象
            cl_event ev;

            // 将 src0 的数据复制到设备中
            CL_CHECK(ggml_cl_h2d_tensor_2d(queue, d_X, 0, src0, i03, i02, &ev));

            // 计算 i03 和 i02 对 ne13 和 ne12 取模的结果
            const int64_t i13 = i03%ne13;
            const int64_t i12 = i02%ne12;
            const int i1 = i13*ne12*ne11 + i12*ne11;

            // 设置偏移量
            cl_int x_offset = 0;
            cl_int y_offset = i1*ne10;
            cl_int d_offset = 0;

            // 计算全局工作项数量
            size_t global = ne00 * ne01;
            cl_int ky = ne10 * ne11;

            // 设置内核参数
            CL_CHECK(clSetKernelArg(add_f32_cl, 0, sizeof(cl_mem), &d_X));
            CL_CHECK(clSetKernelArg(add_f32_cl, 1, sizeof(cl_int), &x_offset));
            CL_CHECK(clSetKernelArg(add_f32_cl, 2, sizeof(cl_mem), &d_Y));
            CL_CHECK(clSetKernelArg(add_f32_cl, 3, sizeof(cl_int), &y_offset));
            CL_CHECK(clSetKernelArg(add_f32_cl, 4, sizeof(cl_mem), &d_D));
            CL_CHECK(clSetKernelArg(add_f32_cl, 5, sizeof(cl_int), &d_offset));
            CL_CHECK(clSetKernelArg(add_f32_cl, 6, sizeof(cl_int), &ky));
            // 执行内核函数
            CL_CHECK(clEnqueueNDRangeKernel(queue, add_f32_cl, 1, NULL, &global, NULL, 1, &ev, NULL));

            // 释放事件对象
            CL_CHECK(clReleaseEvent(ev));
            // 等待队列中的所有命令执行完毕
            CL_CHECK(clFinish(queue));

            // 将结果数据从设备复制到主机
            float * d = (float *) ((char *) dst->data + i02*nb2 + i03*nb3);
            CL_CHECK(clEnqueueReadBuffer(queue, d_D, true, 0, sizeof(float) * ne00*ne01, d, 0, NULL, NULL));
        }
    }
    // 释放设备内存
    ggml_cl_pool_free(d_X, x_size);
    ggml_cl_pool_free(d_D, d_size);
// 将两个浮点型张量相加，结果存储在目标张量中
void ggml_cl_add(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst) {
    // 断言输入张量类型为浮点型
    GGML_ASSERT(src0->type == GGML_TYPE_F32 && src1->type == GGML_TYPE_F32 && dst->type == GGML_TYPE_F32);
    // 调用浮点型张量相加函数
    ggml_cl_add_f32(src0, src1, dst);
}

// 计算两个浮点型矩阵的乘积
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

    // 计算矩阵乘法的相关参数
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
    cl_mem d_Y = src1->backend == GGML_BACKEND_GPU ? (cl_mem) src1->extra : ggml_cl_pool_malloc(sizeof(float) * y_ne, &y_size);
    cl_mem d_D =  dst->backend == GGML_BACKEND_GPU ? (cl_mem)  dst->extra : ggml_cl_pool_malloc(sizeof(float) * d_ne, &d_size);

    size_t x_offset = 0;

    // 根据后端类型释放内存
    if (src0->backend != GGML_BACKEND_GPU) {
        ggml_cl_pool_free(d_X, x_size);
    }
    if (src1->backend != GGML_BACKEND_GPU) {
        ggml_cl_pool_free(d_Y, y_size);
    }
    if (dst->backend != GGML_BACKEND_GPU) {
        ggml_cl_pool_free(d_D, d_size);
    }
}

// 计算两个半精度浮点型矩阵的乘积
static void ggml_cl_mul_mat_f16(const ggml_tensor * src0, const ggml_tensor * src1, ggml_tensor * dst, void * wdata, size_t wsize) {
    // 断言半精度浮点数支持
    GGML_ASSERT(fp16_support);

    const int64_t ne00 = src0->ne[0];
    // 从源数据结构 src0 中获取 ne 数组的元素值
    const int64_t ne01 = src0->ne[1];
    const int64_t ne02 = src0->ne[2];
    const int64_t ne03 = src0->ne[3];

    // 从源数据结构 src1 中获取 ne 和 nb 数组的元素值
    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
    const int64_t ne13 = src1->ne[3];

    const int nb10 = src1->nb[0];
    const int nb11 = src1->nb[1];
    const int nb12 = src1->nb[2];
    const int nb13 = src1->nb[3];

    // 从目标数据结构 dst 中获取 nb 数组的元素值
    const int nb2  = dst->nb[2];
    const int nb3  = dst->nb[3];

    // 计算比例值 r2 和 r3
    const int64_t r2 = ne12 / ne02;
    const int64_t r3 = ne13 / ne03;

    // 初始化 alpha 和 beta 值
    const ggml_fp16_t alpha = ggml_fp32_to_fp16(1.0f);
    const ggml_fp16_t beta = ggml_fp32_to_fp16(0.0f);

    // 计算 x_ne, y_ne, d_ne 的值
    const int x_ne = ne01 * ne00;
    const int y_ne = ne11 * ne10;
    const int d_ne = ne11 * ne01;

    // 断言确保 wsize 大于等于指定大小
    GGML_ASSERT(wsize >= sizeof(ggml_fp16_t) * y_ne);
    GGML_ASSERT(wsize >= sizeof(ggml_fp16_t) * d_ne);
    ggml_fp16_t * const tmp = (ggml_fp16_t *) wdata;

    // 初始化一些变量
    size_t x_size;
    size_t y_size;
    size_t d_size;
    cl_mem d_X;
    
    // 根据不同情况分配内存
    if (src0->backend == GGML_BACKEND_GPU) { // NOLINT
        d_X = (cl_mem) src0->extra;
    } else {
        d_X = ggml_cl_pool_malloc(sizeof(ggml_fp16_t) * x_ne, &x_size);
    }
    cl_mem d_Y = ggml_cl_pool_malloc(sizeof(ggml_fp16_t) * y_ne, &y_size);
    cl_mem d_D = ggml_cl_pool_malloc(sizeof(ggml_fp16_t) * d_ne, &d_size);

    // 检查 src1 是否按行连续和按列连续
    bool src1_cont_rows = nb10 == sizeof(float);
    bool src1_cont_cols = (size_t)nb11 == ne11*sizeof(float);

    // 初始化 x_offset

    }

    // 根据不同情况释放内存
    if (src0->backend != GGML_BACKEND_GPU) {
        ggml_cl_pool_free(d_X, x_size);
    }
    ggml_cl_pool_free(d_Y, y_size);
    ggml_cl_pool_free(d_D, d_size);
// 实现矩阵乘法操作，将结果存储在目标张量中
static void ggml_cl_mul_mat_q_f32(const ggml_tensor * src0, const ggml_tensor * src1, ggml_tensor * dst) {
    // 获取源张量 src0 的各维度大小
    const int64_t ne00 = src0->ne[0];
    const int64_t ne01 = src0->ne[1];
    const int64_t ne02 = src0->ne[2];
    const int64_t ne03 = src0->ne[3];

    // 获取源张量 src1 的各维度大小
    const int64_t ne10 = src1->ne[0];
    const int64_t ne11 = src1->ne[1];
    const int64_t ne12 = src1->ne[2];
    const int64_t ne13 = src1->ne[3];

    // 获取目标张量 dst 的部分维度大小
    const int nb2  = dst->nb[2];
    const int nb3  = dst->nb[3];
    const ggml_type type = src0->type;
    // 判断是否为矩阵乘向量操作
    const bool mul_mat_vec = ne11 == 1 && ne00%2 == 0;

    // 计算矩阵乘法结果的维度大小
    const int64_t r2 = ne12 / ne02;
    const int64_t r3 = ne13 / ne03;

    // 设置乘法操作的参数
    const float alpha = 1.0f;
    const float beta = 0.0f;
    const int x_ne = ne01 * ne00;
    const int y_ne = ne11 * ne10;
    const int d_ne = ne11 * ne01;
    const int x_bps = x_ne / ggml_blck_size(type); // 每个 2D 切片的块数
    const size_t q_sz = ggml_type_size(type) * x_bps;

    // 初始化一些变量
    size_t x_size;
    size_t y_size;
    size_t d_size;
    size_t q_size;
    cl_mem d_X;
    if (!mul_mat_vec) {
        // 分配内存给 d_X
        d_X = ggml_cl_pool_malloc(sizeof(float) * x_ne, &x_size);
    }
    // 分配内存给 d_Y 和 d_D
    cl_mem d_Y = ggml_cl_pool_malloc(sizeof(float) * y_ne, &y_size);
    cl_mem d_D = ggml_cl_pool_malloc(sizeof(float) * d_ne, &d_size);
    cl_mem d_Q;
    if (src0->backend == GGML_BACKEND_CPU) {
        // 如果源张量在 CPU 上，则分配内存给 d_Q
        d_Q = ggml_cl_pool_malloc(q_sz, &q_size);
    }

    // 获取转换为 float32 的内核和矩阵乘向量的内核
    cl_kernel* to_fp32_cl = ggml_get_to_fp32_cl(type);
    cl_kernel* dmmv = ggml_get_dequantize_mul_mat_vec_cl(type);
    GGML_ASSERT(to_fp32_cl != nullptr);

    // 计算全局工作组大小和局部工作组大小
    const size_t global_denom = ggml_cl_global_denom(type);
    const size_t local = mul_mat_vec ? CL_DMMV_LOCAL_SIZE : ggml_cl_local_size(type);

    // 初始化事件索引和事件向量
    size_t ev_idx = 0;
    std::vector<cl_event> events;

    // 释放内存
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
bool ggml_cl_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, const struct ggml_tensor * dst) {
    // 获取第一个张量的第一个维度大小
    const int64_t ne10 = src1->ne[0];

    // 获取目标张量的两个维度大小
    const int64_t ne0 = dst->ne[0];
    const int64_t ne1 = dst->ne[1];

    // TODO: 找到这些参数的最佳值
    // 检查是否满足矩阵乘法的条件
    if ((src0->type == GGML_TYPE_F32 || src0->type == GGML_TYPE_F16 || ggml_is_quantized(src0->type)) &&
        src1->type == GGML_TYPE_F32 &&
        dst->type == GGML_TYPE_F32 &&
        ((ne0 >= 32 && ne1 >= 32 && ne10 >= 32) || src0->backend == GGML_BACKEND_GPU)) {
        return true;
    }

    return false;
}

// 检查是否应该使用 FP16 进行矩阵乘法操作
static bool ggml_cl_mul_mat_use_f16(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * /* dst */) {
    // 如果设备不支持 FP16
    if (!fp16_support) {
        return false;
    }

    // 计算第一个张量和第二个张量的字节大小
    size_t src0_sz = ggml_nbytes(src0);
    size_t src1_sz = ggml_nbytes(src1);

    // 计算使用量化的矩阵乘法所需的数据传输量
    size_t mul_mat_q_transfer = src0_sz + src1_sz;

    // 计算使用 FP16 的矩阵乘法所需的数据传输量
    size_t mul_mat_f16_transfer = src0_sz + sizeof(ggml_fp16_t) * ggml_nelements(src1);

    // 选择较小的传输量来传输到设备
    // TODO: 由于转换为 FP16 的开销，这并不总是最佳选择
    return mul_mat_f16_transfer < mul_mat_q_transfer;
}

// 执行矩阵乘法操作
void ggml_cl_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst, void * wdata, size_t wsize) {
    // 断言是否可以进行矩阵乘法操作
    GGML_ASSERT(ggml_cl_can_mul_mat(src0, src1, dst));

    // 如果第一个张量的类型为 F32
    if (src0->type == GGML_TYPE_F32) {
        // 执行 F32 类型的矩阵乘法操作
        ggml_cl_mul_mat_f32(src0, src1, dst);
    }
    // 如果第一个张量的类型为 F16
    else if (src0->type == GGML_TYPE_F16) {
        // 如果应该使用 FP16 进行矩阵乘法操作
        if (ggml_cl_mul_mat_use_f16(src0, src1, dst)) {
            // 执行 FP16 类型的矩阵乘法操作
            ggml_cl_mul_mat_f16(src0, src1, dst, wdata, wsize);
        }
        else {
            // 执行量化后转为 F32 类型的矩阵乘法操作
            ggml_cl_mul_mat_q_f32(src0, src1, dst);
        }
    }
}
    # 如果第一个输入矩阵是量化的，则调用 ggml_cl_mul_mat_q_f32 函数进行矩阵乘法
    else if (ggml_is_quantized(src0->type)) {
        ggml_cl_mul_mat_q_f32(src0, src1, dst);
    }
    # 如果不是量化的，则抛出断言错误
    else {
        GGML_ASSERT(false);
    }
// 获取两个输入张量和一个输出张量的大小，用于矩阵乘法操作
size_t ggml_cl_mul_mat_get_wsize(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst) {
    // 如果输入张量的类型为 GGML_TYPE_F16 并且可以使用 F16 进行矩阵乘法操作
    if (src0->type == GGML_TYPE_F16 && ggml_cl_mul_mat_use_f16(src0, src1, dst)) {
        // 返回 F16 类型的元素大小乘以输入张量和输出张量中较大的行列数
        return sizeof(ggml_fp16_t) * std::max(src1->ne[0] * src1->ne[1], dst->ne[0] * dst->ne[1]);
    }
    // 否则返回 0
    return 0;
}

// 将数据转换为张量
void ggml_cl_transform_tensor(void * data, ggml_tensor * tensor) {
    // 获取张量的各维度大小
    const int64_t ne0 = tensor->ne[0];
    const int64_t ne1 = tensor->ne[1];
    const int64_t ne2 = tensor->ne[2];
    const int64_t ne3 = tensor->ne[3];

    // 获取张量的数据类型和元素大小
    const ggml_type type = tensor->type;
    const size_t s_sz = ggml_type_size(type) * (size_t) (ne0 * ne1 / ggml_blck_size(type));
    const size_t q_sz = s_sz * (size_t) (ne2 * ne3);

    // 在 OpenCL 设备上分配内存空间
    size_t q_size;
    cl_mem dst = ggml_cl_pool_malloc(q_sz, &q_size);

    // 将数据赋值给张量
    tensor->data = data;
    // 将张量数据拷贝到设备
    size_t offset = 0;
    for (int64_t i3 = 0; i3 < ne3; i3++) {
        for (int64_t i2 = 0; i2 < ne2; i2++) {
            CL_CHECK(ggml_cl_h2d_tensor_2d(queue, dst, offset, tensor, i3, i2, NULL));
            offset += s_sz;
        }
    }

    // 等待设备操作完成
    CL_CHECK(clFinish(queue));

    // 将设备内存地址赋值给张量的额外信息
    tensor->extra = dst;
    // 确保张量的后端为 GPU
    GGML_ASSERT(tensor->backend == GGML_BACKEND_GPU);
}

// OpenCL 后端缓冲区相关函数

// OpenCL 后端缓冲区上下文结构
struct ggml_backend_opencl_buffer_context {
    // 析构函数，释放内存对象
    ~ggml_backend_opencl_buffer_context() {
        if (buffer) {
            clReleaseMemObject(buffer);
        }
        for (auto * sub_buffer : sub_buffers) {
            clReleaseMemObject(sub_buffer);
        }
    }

    cl_mem buffer;
    std::vector<cl_mem> sub_buffers;
};

// OpenCL 指针基地址
static void * const cl_ptr_base = (void *)(uintptr_t) 0x1000;

// 获取 OpenCL 缓冲区的名称
static const char * ggml_backend_opencl_buffer_get_name(ggml_backend_buffer_t buffer) {
    return "OpenCL";

    GGML_UNUSED(buffer);
}

// 释放 OpenCL 缓冲区
static void ggml_backend_opencl_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    ggml_backend_opencl_buffer_context * ctx = (ggml_backend_opencl_buffer_context *) buffer->context;
    delete ctx;
}
// 获取 OpenCL 缓冲区的基地址
static void * ggml_backend_opencl_buffer_get_base(ggml_backend_buffer_t buffer) {
    return cl_ptr_base;

    GGML_UNUSED(buffer);
}

// 初始化张量的 OpenCL 缓冲区
static void ggml_backend_opencl_buffer_init_tensor(ggml_backend_buffer_t buffer, ggml_tensor * tensor) {
    // 如果张量的视图源不为空且偏移为0，则将额外信息设置为视图源的额外信息
    if (tensor->view_src != NULL && tensor->view_offs == 0) {
        tensor->extra = tensor->view_src->extra;
    } else {
        // 否则，根据张量数据和大小创建子缓冲区
        ggml_backend_opencl_buffer_context * ctx = (ggml_backend_opencl_buffer_context *) buffer->context;
        cl_buffer_region region = {(size_t)((char *)tensor->data - (char *)cl_ptr_base), ggml_nbytes(tensor)};
        cl_int err;
        cl_mem sub_buffer = clCreateSubBuffer(ctx->buffer, CL_MEM_READ_WRITE, CL_BUFFER_CREATE_TYPE_REGION, &region, &err);
        CL_CHECK(err);
        ctx->sub_buffers.push_back(sub_buffer);
        tensor->extra = sub_buffer;
    }
    tensor->backend = GGML_BACKEND_GPU;
}

// 设置张量的 OpenCL 缓冲区
static void ggml_backend_opencl_buffer_set_tensor(ggml_backend_buffer_t buffer, ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    cl_mem tensor_buffer = (cl_mem) tensor->extra;
    // 将数据写入缓冲区
    CL_CHECK(clEnqueueWriteBuffer(queue, tensor_buffer, true, offset, size, data, 0, NULL, NULL));
    CL_CHECK(clFinish(queue));

    GGML_UNUSED(buffer);
}

// 获取张量的 OpenCL 缓冲区
static void ggml_backend_opencl_buffer_get_tensor(ggml_backend_buffer_t buffer, const ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    cl_mem tensor_buffer = (cl_mem) tensor->extra;
    // 从缓冲区读取数据
    CL_CHECK(clEnqueueReadBuffer(queue, tensor_buffer, true, offset, size, data, 0, NULL, NULL));
    CL_CHECK(clFinish(queue));

    GGML_UNUSED(buffer);
}

// 清空 OpenCL 缓冲区
static void ggml_backend_opencl_buffer_clear(ggml_backend_buffer_t buffer, uint8_t value) {
    ggml_backend_opencl_buffer_context * ctx = (ggml_backend_opencl_buffer_context *) buffer->context;
    // 使用指定值填充缓冲区
    CL_CHECK(clEnqueueFillBuffer(queue, ctx->buffer, &value, sizeof(value), 0, buffer->size, 0, NULL, NULL));
    CL_CHECK(clFinish(queue));
}
// 重置 OpenCL 缓冲区，释放子缓冲区资源
static void ggml_backend_opencl_buffer_reset(ggml_backend_buffer_t buffer) {
    // 获取 OpenCL 缓冲区上下文
    ggml_backend_opencl_buffer_context * ctx = (ggml_backend_opencl_buffer_context *) buffer->context;
    // 释放所有子缓冲区资源
    for (auto * sub_buffer : ctx->sub_buffers) {
        clReleaseMemObject(sub_buffer);
    }
    // 清空子缓冲区列表
    ctx->sub_buffers.clear();
}

// 定义 OpenCL 缓冲区接口
static ggml_backend_buffer_i ggml_backend_opencl_buffer_interface = {
    /* .get_name        = */ ggml_backend_opencl_buffer_get_name,
    /* .free_buffer     = */ ggml_backend_opencl_buffer_free_buffer,
    /* .get_base        = */ ggml_backend_opencl_buffer_get_base,
    /* .init_tensor     = */ ggml_backend_opencl_buffer_init_tensor,
    /* .set_tensor      = */ ggml_backend_opencl_buffer_set_tensor,
    /* .get_tensor      = */ ggml_backend_opencl_buffer_get_tensor,
    /* .cpy_tensor      = */ NULL,
    /* .clear           = */ ggml_backend_opencl_buffer_clear,
    /* .reset           = */ ggml_backend_opencl_buffer_reset,
};

// 获取 OpenCL 缓冲区类型名称
static const char * ggml_backend_opencl_buffer_type_name(ggml_backend_buffer_type_t buffer_type) {
    return "OpenCL";

    GGML_UNUSED(buffer_type);
}

// 分配 OpenCL 缓冲区
static ggml_backend_buffer_t ggml_backend_opencl_buffer_type_alloc_buffer(ggml_backend_buffer_type_t buffer_type, size_t size) {
    // 初始化 OpenCL
    ggml_cl_init();

    cl_int err;
    // 创建 OpenCL 缓冲区
    cl_mem mem = clCreateBuffer(context, CL_MEM_READ_WRITE, size, NULL, &err);
    if (err != CL_SUCCESS) {
        fprintf(stderr, "%s: failed to allocate %.2f MiB\n", __func__, size / 1024.0 / 1024.0);
        return nullptr;
    }

    // 创建 OpenCL 缓冲区上下文
    ggml_backend_opencl_buffer_context * ctx = new ggml_backend_opencl_buffer_context{mem, {}};

    return ggml_backend_buffer_init(buffer_type, ggml_backend_opencl_buffer_interface, ctx, size);
}

// 获取 OpenCL 缓冲区类型对齐方式
static size_t ggml_backend_opencl_buffer_type_get_alignment(ggml_backend_buffer_type_t buffer_type) {
    // FIXME: not thread safe, device may not be initialized yet
    static cl_uint alignment = -1;
}
    # 如果对齐值为-1，则执行以下操作
    if (alignment == (cl_uint)-1) {
        # 初始化 OpenCL 环境
        ggml_cl_init();
        # 获取设备信息中的内存基地址对齐值
        clGetDeviceInfo(device, CL_DEVICE_MEM_BASE_ADDR_ALIGN, sizeof(cl_uint), &alignment, NULL);
    }
    # 返回对齐值
    return alignment;

    # 忽略未使用的 buffer_type 变量
    GGML_UNUSED(buffer_type);
// 获取指定缓冲区类型的最大大小
static size_t ggml_backend_opencl_buffer_type_get_max_size(ggml_backend_buffer_type_t buffer_type) {
    // 静态变量，用于存储最大大小，初始值为-1
    static size_t max_size = -1;
    // 如果最大大小为-1，则进行初始化
    if (max_size == (size_t)-1) {
        // 初始化 OpenCL 环境
        ggml_cl_init();
        // 获取设备信息中的最大内存分配大小
        clGetDeviceInfo(device, CL_DEVICE_MAX_MEM_ALLOC_SIZE, sizeof(size_t), &max_size, NULL);
    }
    // 返回最大大小
    return max_size;
}

// 检查指定缓冲区类型是否支持指定后端
static bool ggml_backend_opencl_buffer_type_supports_backend(ggml_backend_buffer_type_t buffer_type, ggml_backend_t backend) {
    // 返回指定后端是否为 CPU 后端
    return ggml_backend_is_cpu(backend);

    // 未使用的参数
    GGML_UNUSED(buffer_type);
}

// OpenCL 缓冲区类型接口
static ggml_backend_buffer_type_i ggml_backend_opencl_buffer_type_interface = {
    /* .get_name         = */ ggml_backend_opencl_buffer_type_name,
    /* .alloc_buffer     = */ ggml_backend_opencl_buffer_type_alloc_buffer,
    /* .get_alignment    = */ ggml_backend_opencl_buffer_type_get_alignment,
    /* .get_max_size     = */ ggml_backend_opencl_buffer_type_get_max_size,
    /* .get_alloc_size   = */ NULL,
    /* .supports_backend = */ ggml_backend_opencl_buffer_type_supports_backend,
    /* .is_host          = */ NULL,
};

// 获取 OpenCL 缓冲区类型
ggml_backend_buffer_type_t ggml_backend_opencl_buffer_type() {
    // 静态变量，存储缓冲区类型
    static ggml_backend_buffer_type buffer_type = {
        /* .iface   = */ ggml_backend_opencl_buffer_type_interface,
        /* .context = */ nullptr,
    };

    // 返回缓冲区类型
    return &buffer_type;
}

// 主机缓冲区类型的相关函数被注释掉，不参与代码执行
    // 在主机上分配一块内存，大小为 size
    void * ptr = ggml_cl_host_malloc(size);

    // 如果内存分配失败
    if (ptr == nullptr) {
        // 回退到 CPU 缓冲区
        return ggml_backend_buft_alloc_buffer(ggml_backend_cpu_buffer_type(), size);
    }

    // 从指针创建一个 CPU 缓冲区
    ggml_backend_buffer_t buffer = ggml_backend_cpu_buffer_from_ptr(ptr, size);
    // 设置缓冲区的类型为 buft
    buffer->buft = buft;
    // 设置缓冲区接口的获取名称函数为 ggml_backend_opencl_host_buffer_name
    buffer->iface.get_name = ggml_backend_opencl_host_buffer_name;
    // 设置缓冲区接口的释放缓冲区函数为 ggml_backend_opencl_host_buffer_free_buffer

    return buffer;
// 返回 OpenCL 主机缓冲区类型
ggml_backend_buffer_type_t ggml_backend_opencl_host_buffer_type() {
    // 创建静态的 OpenCL 主机缓冲区类型结构体
    static struct ggml_backend_buffer_type ggml_backend_opencl_buffer_type_host = {
        /* .iface    = */ {
            // 获取 OpenCL 主机缓冲区类型的名称函数
            /* .get_name         = */ ggml_backend_opencl_host_buffer_type_name,
            // 分配 OpenCL 主机缓冲区函数
            /* .alloc_buffer     = */ ggml_backend_opencl_host_buffer_type_alloc_buffer,
            // 获取对齐方式函数，继承自 CPU 缓冲区类型
            /* .get_alignment    = */ ggml_backend_cpu_buffer_type()->iface.get_alignment,
            // 获取最大大小函数，默认为 SIZE_MAX
            /* .get_max_size     = */ NULL,
            // 获取分配大小函数，继承自 CPU 缓冲区类型
            /* .get_alloc_size   = */ ggml_backend_cpu_buffer_type()->iface.get_alloc_size,
            // 支持的后端函数，继承自 CPU 缓冲区类型
            /* .supports_backend = */ ggml_backend_cpu_buffer_type()->iface.supports_backend,
            // 是否为主机函数，继承自 CPU 缓冲区类型
            /* .is_host          = */ ggml_backend_cpu_buffer_type()->iface.is_host,
        },
        // 上下文为空指针
        /* .context  = */ nullptr,
    };

    // 返回 OpenCL 主机缓冲区类型结构体指针
    return &ggml_backend_opencl_buffer_type_host;
}

// 返回 OpenCL 的名称
static const char * ggml_backend_opencl_name(ggml_backend_t backend) {
    // 返回字符串 "OpenCL"
    return "OpenCL";

    // 使用 GGML_UNUSED 宏避免未使用的参数警告
    GGML_UNUSED(backend);
}

// 释放 OpenCL 后端资源
static void ggml_backend_opencl_free(ggml_backend_t backend) {
    // 使用 GGML_UNUSED 宏避免未使用的参数警告
    GGML_UNUSED(backend);
}

// 获取 OpenCL 默认缓冲区类型
static ggml_backend_buffer_type_t ggml_backend_opencl_get_default_buffer_type(ggml_backend_t backend) {
    // 返回 OpenCL 主机缓冲区类型
    return ggml_backend_opencl_buffer_type();

    // 使用 GGML_UNUSED 宏避免未使用的参数警告
    GGML_UNUSED(backend);
}

// 在 OpenCL 后端上执行计算图
static bool ggml_backend_opencl_graph_compute(ggml_backend_t backend, ggml_cgraph * graph) {
    // 遍历计算图中的节点
    for (int i = 0; i < graph->n_nodes; ++i) {
        ggml_tensor * node = graph->nodes[i];
        // 根据节点操作类型执行相应操作
        switch (node->op) {
            case GGML_OP_MUL_MAT:
                ggml_cl_mul_mat(node->src[0], node->src[1], node, nullptr, 0);
                break;
            case GGML_OP_MUL:
                ggml_cl_mul(node->src[0], node->src[1], node);
                break;
            default:
                // 断言不应该执行到这里
                GGML_ASSERT(false);
        }
    }

    // 返回 true 表示计算成功
    return true;

    // 使用 GGML_UNUSED 宏避免未使用的参数警告
    GGML_UNUSED(backend);
}

// 检查 OpenCL 后端是否支持特定操作
static bool ggml_backend_opencl_supports_op(ggml_backend_t backend, const ggml_tensor * op) {
    # 根据操作符的类型进行不同的操作
    switch (op->op) {
        # 如果操作符是矩阵相乘操作
        case GGML_OP_MUL_MAT:
            # 调用 ggml_cl_can_mul_mat 函数来检查是否可以进行矩阵相乘
            return ggml_cl_can_mul_mat(op->src[0], op->src[1], op);
        # 如果操作符是普通乘法操作
        case GGML_OP_MUL:
            # 暂时注释掉的代码，原本是调用 ggml_can_repeat_rows 函数来检查是否可以重复行
            # return ggml_can_repeat_rows(op->src[1], op->src[0]);
            # 直接返回 true，表示可以进行乘法操作
            return true;
        # 其他情况
        default:
            # 返回 false，表示无法进行操作
            return false;
    }

    # 忽略未使用的变量 backend
    GGML_UNUSED(backend);
}

static ggml_backend_i opencl_backend_i = {
    /* 定义一个包含函数指针的结构体，表示 OpenCL 后端接口 */
    /* 获取 OpenCL 后端名称的函数指针 */
    /* 释放 OpenCL 后端资源的函数指针 */
    /* 获取默认缓冲区类型的函数指针 */
    /* 设置异步张量的函数指针 */
    /* 获取异步张量的函数指针 */
    /* 从异步张量复制到内存的函数指针 */
    /* 从内存复制到异步张量的函数指针 */
    /* 同步函数指针 */
    /* 创建图计划的函数指针 */
    /* 释放图计划的函数指针 */
    /* 计算图计划的函数指针 */
    /* 计算图的函数指针 */
    /* 判断是否支持某个操作的函数指针 */
};

ggml_backend_t ggml_backend_opencl_init() {
    /* 初始化一个 OpenCL 后端 */
    ggml_backend_t backend = new ggml_backend {
        /* 设置后端接口为 OpenCL 后端接口 */
        /* 设置上下文为空指针 */
    };

    return backend;
}

bool ggml_backend_is_opencl(ggml_backend_t backend) {
    /* 判断给定的后端是否为 OpenCL 后端 */
    return backend && backend->iface.get_name == ggml_backend_opencl_name;
}
#endif
```