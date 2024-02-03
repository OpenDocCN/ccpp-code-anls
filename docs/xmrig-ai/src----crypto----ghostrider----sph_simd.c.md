# `xmrig\src\crypto\ghostrider\sph_simd.c`

```cpp
/* $Id: simd.c 227 2010-06-16 17:28:38Z tp $ */
# 定义了一个字符串常量，用于标识代码的版本信息
/*
 * SIMD implementation.
 * 实现了SIMD（单指令多数据流）的功能
 *
 * ==========================(LICENSE BEGIN)============================
 *
 * Copyright (c) 2007-2010  Projet RNRT SAPHIR
 * 版权所有 (c) 2007-2010  Projet RNRT SAPHIR
 *
 * Permission is hereby granted, free of charge, to any person obtaining
 * a copy of this software and associated documentation files (the
 * "Software"), to deal in the Software without restriction, including
 * without limitation the rights to use, copy, modify, merge, publish,
 * distribute, sublicense, and/or sell copies of the Software, and to
 * permit persons to whom the Software is furnished to do so, subject to
 * the following conditions:
 * 授予任何获得本软件及其相关文档文件（以下简称“软件”）副本的人免费使用、复制、修改、合并、发布、分发、再许可和/或销售软件的权利，并允许获得软件的人遵守以下条件：
 *
 * The above copyright notice and this permission notice shall be
 * included in all copies or substantial portions of the Software.
 * 上述版权声明和此许可声明应包含在所有副本或实质部分的软件中。
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 * IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
 * CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
 * TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
 * SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
 * 本软件按“原样”提供，不提供任何形式的明示或暗示担保，包括但不限于对适销性、特定用途的适用性和非侵权性的担保。无论在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是合同、侵权行为或其他行为的结果，与软件或使用或其他方式有关。
 *
 * ===========================(LICENSE END)=============================
 *
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 * 作者：Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#include <stddef.h>
#include <string.h>
#include <limits.h>

#include "sph_simd.h"

#ifdef __cplusplus
extern "C"{
#endif

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_SIMD
#define SPH_SMALL_FOOTPRINT_SIMD   1
#endif

#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif

typedef sph_u32 u32;
typedef sph_s32 s32;
#define C32     SPH_C32
#define T32     SPH_T32
#define ROL32   SPH_ROTL32

#define XCAT(x, y)    XCAT_(x, y)
#define XCAT_(x, y)   x ## y

/*
 * The powers of 41 modulo 257. We use exponents from 0 to 255, inclusive.
 */
# 定义了一个宏函数，用于将两个字符串拼接在一起
# 定义一个静态的整型数组，包含一系列预先设定的数值
static const s32 alpha_tab[] = {
      1,  41, 139,  45,  46,  87, 226,  14,  60, 147, 116, 130,
    190,  80, 196,  69,   2,  82,  21,  90,  92, 174, 195,  28,
    120,  37, 232,   3, 123, 160, 135, 138,   4, 164,  42, 180,
    184,  91, 133,  56, 240,  74, 207,   6, 246,  63,  13,  19,
      8,  71,  84, 103, 111, 182,   9, 112, 223, 148, 157,  12,
    235, 126,  26,  38,  16, 142, 168, 206, 222, 107,  18, 224,
    189,  39,  57,  24, 213, 252,  52,  76,  32,  27,  79, 155,
    187, 214,  36, 191, 121,  78, 114,  48, 169, 247, 104, 152,
     64,  54, 158,  53, 117, 171,  72, 125, 242, 156, 228,  96,
     81, 237, 208,  47, 128, 108,  59, 106, 234,  85, 144, 250,
    227,  55, 199, 192, 162, 217, 159,  94, 256, 216, 118, 212,
    211, 170,  31, 243, 197, 110, 141, 127,  67, 177,  61, 188,
    255, 175, 236, 167, 165,  83,  62, 229, 137, 220,  25, 254,
    134,  97, 122, 119, 253,  93, 215,  77,  73, 166, 124, 201,
     17, 183,  50, 251,  11, 194, 244, 238, 249, 186, 173, 154,
    146,  75, 248, 145,  34, 109, 100, 245,  22, 131, 231, 219,
    241, 115,  89,  51,  35, 150, 239,  33,  68, 218, 200, 233,
     44,   5, 205, 181, 225, 230, 178, 102,  70,  43, 221,  66,
    136, 179, 143, 209,  88,  10, 153, 105, 193, 203,  99, 204,
    140,  86, 185, 132,  15, 101,  29, 161, 176,  20,  49, 210,
    129, 149, 198, 151,  23, 172, 113,   7,  30, 202,  58,  65,
     95,  40,  98, 163
};

# 定义宏，用于将输入值映射到特定范围内的值
# REDS1: 将输入值映射到 -383 到 383 的范围内
#define REDS1(x)    (((x) & 0xFF) - ((x) >> 8))
# REDS2: 将输入值映射到 -32768 到 98302 的范围内
#define REDS2(x)    (((x) & 0xFFFF) + ((x) >> 16)

# 如果 q[] 数组中的值都在 -N 到 N 的范围内（其中 N >= 98302），则将新的 q[] 数组中的值映射到 -2N 到 2N 的范围内
# 由于 alpha_tab[v] <= 256，所以 N 的最大允许范围为 8388608
#define FFT_LOOP(rb, hk, as, id)   do { \
        size_t u, v; \
        s32 m = q[(rb)]; \
        s32 n = q[(rb) + (hk)]; \
        q[(rb)] = m + n; \
        q[(rb) + (hk)] = m - n; \
        u = v = 0; \
        goto id; \
        for (; u < (hk); u += 4, v += 4 * (as)) { \
            s32 t; \
            m = q[(rb) + u + 0]; \
            n = q[(rb) + u + 0 + (hk)]; \
            t = REDS2(n * alpha_tab[v + 0 * (as)]); \
            q[(rb) + u + 0] = m + t; \
            q[(rb) + u + 0 + (hk)] = m - t; \
        id: \
            m = q[(rb) + u + 1]; \
            n = q[(rb) + u + 1 + (hk)]; \
            t = REDS2(n * alpha_tab[v + 1 * (as)]); \
            q[(rb) + u + 1] = m + t; \
            q[(rb) + u + 1 + (hk)] = m - t; \
            m = q[(rb) + u + 2]; \
            n = q[(rb) + u + 2 + (hk)]; \
            t = REDS2(n * alpha_tab[v + 2 * (as)]); \
            q[(rb) + u + 2] = m + t; \
            q[(rb) + u + 2 + (hk)] = m - t; \
            m = q[(rb) + u + 3]; \
            n = q[(rb) + u + 3 + (hk)]; \
            t = REDS2(n * alpha_tab[v + 3 * (as)]); \
            q[(rb) + u + 3] = m + t; \
            q[(rb) + u + 3 + (hk)] = m - t; \
        } \
    } while (0)



/*
 * Output ranges:
 *   d0:   min=    0   max= 1020
 *   d1:   min=  -67   max= 4587
 *   d2:   min=-4335   max= 4335
 *   d3:   min=-4147   max=  507
 *   d4:   min= -510   max=  510
 *   d5:   min= -252   max= 4402
 *   d6:   min=-4335   max= 4335
 *   d7:   min=-4332   max=  322
 */


注释：这段代码是一个宏定义，用于执行快速傅里叶变换（FFT）。宏定义中的代码实现了一个循环，用于计算傅里叶变换的结果。宏定义中的注释提供了输出结果的范围。
# 定义一个宏函数FFT8，用于计算长度为8的快速傅里叶变换
# 参数xb表示输入数组x的起始索引，xs表示输入数组x的步长，d表示输出数组d的前缀
#define FFT8(xb, xs, d)   do { \
        # 将输入数组x的元素赋值给局部变量x0、x1、x2、x3
        s32 x0 = x[(xb)]; \
        s32 x1 = x[(xb) + (xs)]; \
        s32 x2 = x[(xb) + 2 * (xs)]; \
        s32 x3 = x[(xb) + 3 * (xs)]; \
        # 计算中间变量a0、a1、a2、a3和b0、b1、b2、b3
        s32 a0 = x0 + x2; \
        s32 a1 = x0 + (x2 << 4); \
        s32 a2 = x0 - x2; \
        s32 a3 = x0 - (x2 << 4); \
        s32 b0 = x1 + x3; \
        s32 b1 = REDS1((x1 << 2) + (x3 << 6)); \
        s32 b2 = (x1 << 4) - (x3 << 4); \
        s32 b3 = REDS1((x1 << 6) + (x3 << 2)); \
        # 将计算结果赋值给输出数组d的对应位置
        d ## 0 = a0 + b0; \
        d ## 1 = a1 + b1; \
        d ## 2 = a2 + b2; \
        d ## 3 = a3 + b3; \
        d ## 4 = a0 - b0; \
        d ## 5 = a1 - b1; \
        d ## 6 = a2 - b2; \
        d ## 7 = a3 - b3; \
    } while (0)

/*
 * 当k=16时，alpha=2。乘以alpha^i可以通过移位操作实现。
 *
 * 输出范围：-591471..591723
 */
#define FFT16(xb, xs, rb)   do { \
        # 定义局部变量d1_0, d1_1, d1_2, d1_3, d1_4, d1_5, d1_6, d1_7
        # 定义局部变量d2_0, d2_1, d2_2, d2_3, d2_4, d2_5, d2_6, d2_7
        s32 d1_0, d1_1, d1_2, d1_3, d1_4, d1_5, d1_6, d1_7; \
        s32 d2_0, d2_1, d2_2, d2_3, d2_4, d2_5, d2_6, d2_7; \
        # 调用FFT8宏函数计算输入数组x的前半部分和后半部分的快速傅里叶变换
        FFT8(xb, (xs) << 1, d1_); \
        FFT8((xb) + (xs), (xs) << 1, d2_); \
        # 将计算结果赋值给输出数组q的对应位置
        q[(rb) +  0] = d1_0 + d2_0; \
        q[(rb) +  1] = d1_1 + (d2_1 << 1); \
        q[(rb) +  2] = d1_2 + (d2_2 << 2); \
        q[(rb) +  3] = d1_3 + (d2_3 << 3); \
        q[(rb) +  4] = d1_4 + (d2_4 << 4); \
        q[(rb) +  5] = d1_5 + (d2_5 << 5); \
        q[(rb) +  6] = d1_6 + (d2_6 << 6); \
        q[(rb) +  7] = d1_7 + (d2_7 << 7); \
        q[(rb) +  8] = d1_0 - d2_0; \
        q[(rb) +  9] = d1_1 - (d2_1 << 1); \
        q[(rb) + 10] = d1_2 - (d2_2 << 2); \
        q[(rb) + 11] = d1_3 - (d2_3 << 3); \
        q[(rb) + 12] = d1_4 - (d2_4 << 4); \
        q[(rb) + 13] = d1_5 - (d2_5 << 5); \
        q[(rb) + 14] = d1_6 - (d2_6 << 6); \
        q[(rb) + 15] = d1_7 - (d2_7 << 7); \
    } while (0)

/*
 * 输出范围：|q| <= 1183446
 */
#define FFT32(xb, xs, rb, id)   do { \  # 定义宏 FFT32，参数为 xb, xs, rb, id
        FFT16(xb, (xs) << 1, rb); \  # 调用 FFT16 函数，参数为 xb, (xs) 左移 1 位, rb
        FFT16((xb) + (xs), (xs) << 1, (rb) + 16); \  # 调用 FFT16 函数，参数为 (xb) + (xs), (xs) 左移 1 位, (rb) + 16
        FFT_LOOP(rb, 16, 8, id); \  # 调用 FFT_LOOP 函数，参数为 rb, 16, 8, id
    } while (0)  # 循环结束

/*
 * Output range: |q| <= 2366892
 */
#define FFT64(xb, xs, rb, id)   do { \  # 定义宏 FFT64，参数为 xb, xs, rb, id
        FFT32(xb, (xs) << 1, rb, XCAT(id, a)); \  # 调用 FFT32 函数，参数为 xb, (xs) 左移 1 位, rb, XCAT(id, a)
        FFT32((xb) + (xs), (xs) << 1, (rb) + 32, XCAT(id, b)); \  # 调用 FFT32 函数，参数为 (xb) + (xs), (xs) 左移 1 位, (rb) + 32, XCAT(id, b)
        FFT_LOOP(rb, 32, 4, id); \  # 调用 FFT_LOOP 函数，参数为 rb, 32, 4, id
    } while (0)  # 循环结束

#if SPH_SMALL_FOOTPRINT_SIMD

static void
fft32(unsigned char *x, size_t xs, s32 *q)  # 定义函数 fft32，参数为 unsigned char *x, size_t xs, s32 *q
{
    size_t xd;  # 声明变量 xd

    xd = xs << 1;  # 计算 xs 左移 1 位的值，赋给 xd
    FFT16(0, xd, 0);  # 调用 FFT16 函数，参数为 0, xd, 0
    FFT16(xs, xd, 16);  # 调用 FFT16 函数，参数为 xs, xd, 16
    FFT_LOOP(0, 16, 8, label_);  # 调用 FFT_LOOP 函数，参数为 0, 16, 8, label_
}

#define FFT128(xb, xs, rb, id)   do { \  # 定义宏 FFT128，参数为 xb, xs, rb, id
        fft32(x + (xb) + ((xs) * 0), (xs) << 2, &q[(rb) +  0]); \  # 调用 fft32 函数，参数为 x + (xb) + ((xs) * 0), (xs) 左移 2 位, &q[(rb) +  0]
        fft32(x + (xb) + ((xs) * 2), (xs) << 2, &q[(rb) + 32]); \  # 调用 fft32 函数，参数为 x + (xb) + ((xs) * 2), (xs) 左移 2 位, &q[(rb) + 32]
        FFT_LOOP(rb, 32, 4, XCAT(id, aa)); \  # 调用 FFT_LOOP 函数，参数为 rb, 32, 4, XCAT(id, aa)
        fft32(x + (xb) + ((xs) * 1), (xs) << 2, &q[(rb) + 64]); \  # 调用 fft32 函数，参数为 x + (xb) + ((xs) * 1), (xs) 左移 2 位, &q[(rb) + 64]
        fft32(x + (xb) + ((xs) * 3), (xs) << 2, &q[(rb) + 96]); \  # 调用 fft32 函数，参数为 x + (xb) + ((xs) * 3), (xs) 左移 2 位, &q[(rb) + 96]
        FFT_LOOP((rb) + 64, 32, 4, XCAT(id, ab)); \  # 调用 FFT_LOOP 函数，参数为 (rb) + 64, 32, 4, XCAT(id, ab)
        FFT_LOOP(rb, 64, 2, XCAT(id, a)); \  # 调用 FFT_LOOP 函数，参数为 rb, 64, 2, XCAT(id, a)
    } while (0)  # 循环结束

#else

/*
 * Output range: |q| <= 4733784
 */
#define FFT128(xb, xs, rb, id)   do { \  # 定义宏 FFT128，参数为 xb, xs, rb, id
        FFT64(xb, (xs) << 1, rb, XCAT(id, a)); \  # 调用 FFT64 函数，参数为 xb, (xs) 左移 1 位, rb, XCAT(id, a)
        FFT64((xb) + (xs), (xs) << 1, (rb) + 64, XCAT(id, b)); \  # 调用 FFT64 函数，参数为 (xb) + (xs), (xs) 左移 1 位, (rb) + 64, XCAT(id, b)
        FFT_LOOP(rb, 64, 2, id); \  # 调用 FFT_LOOP 函数，参数为 rb, 64, 2, id
    } while (0)  # 循环结束

#endif

/*
 * For SIMD-384 / SIMD-512, the fully unrolled FFT yields a compression
 * function which does not fit in the 32 kB L1 cache of a typical x86
 * Intel. We therefore add a function call layer at the FFT64 level.
 */

static void
fft64(unsigned char *x, size_t xs, s32 *q)  # 定义函数 fft64，参数为 unsigned char *x, size_t xs, s32 *q
{
    size_t xd;  # 声明变量 xd

    xd = xs << 1;  # 计算 xs 左移 1 位的值，赋给 xd
    FFT32(0, xd, 0, label_a);  # 调用 FFT32 函数，参数为 0, xd, 0, label_a
    FFT32(xs, xd, 32, label_b);  # 调用 FFT32 函数，参数为 xs, xd, 32, label_b
    FFT_LOOP(0, 32, 4, label_);  # 调用 FFT_LOOP 函数，参数为 0, 32, 4, label_
}

/*
 * Output range: |q| <= 9467568
 */
#define FFT256(xb, xs, rb, id)   do { \  # 定义一个宏，用于执行一系列 FFT 操作
        fft64(x + (xb) + ((xs) * 0), (xs) << 2, &q[(rb) +   0]); \  # 执行 fft64 函数，计算从 x 中指定位置开始的数据的快速傅里叶变换，结果存储到 q 数组中指定位置
        fft64(x + (xb) + ((xs) * 2), (xs) << 2, &q[(rb) +  64]); \  # 执行 fft64 函数，计算从 x 中指定位置开始的数据的快速傅里叶变换，结果存储到 q 数组中指定位置
        FFT_LOOP(rb, 64, 2, XCAT(id, aa)); \  # 调用 FFT_LOOP 宏，执行一系列 FFT 操作
        fft64(x + (xb) + ((xs) * 1), (xs) << 2, &q[(rb) + 128]); \  # 执行 fft64 函数，计算从 x 中指定位置开始的数据的快速傅里叶变换，结果存储到 q 数组中指定位置
        fft64(x + (xb) + ((xs) * 3), (xs) << 2, &q[(rb) + 192]); \  # 执行 fft64 函数，计算从 x 中指定位置开始的数据的快速傅里叶变换，结果存储到 q 数组中指定位置
        FFT_LOOP((rb) + 128, 64, 2, XCAT(id, ab)); \  # 调用 FFT_LOOP 宏，执行一系列 FFT 操作
        FFT_LOOP(rb, 128, 1, XCAT(id, a)); \  # 调用 FFT_LOOP 宏，执行一系列 FFT 操作
    } while (0)  # 宏定义结束

/*
 * alpha^(127*i) mod 257
 */
static const unsigned short yoff_s_n[] = {  # 定义一个静态的无符号短整型数组
      1,  98,  95,  58,  30, 113,  23, 198, 129,  49, 176,  29,  # 数组元素
     15, 185, 140,  99, 193, 153,  88, 143, 136, 221,  70, 178,  # 数组元素
    225, 205,  44, 200,  68, 239,  35,  89, 241, 231,  22, 100,  # 数组元素
     34, 248, 146, 173, 249, 244,  11,  50,  17, 124,  73, 215,  # 数组元素
    253, 122, 134,  25, 137,  62, 165, 236, 255,  61,  67, 141,  # 数组元素
    197,  31, 211, 118, 256, 159, 162, 199, 227, 144, 234,  59,  # 数组元素
    128, 208,  81, 228, 242,  72, 117, 158,  64, 104, 169, 114,  # 数组元素
    121,  36, 187,  79,  32,  52, 213,  57, 189,  18, 222, 168,  # 数组元素
     16,  26, 235, 157, 223,   9, 111,  84,   8,  13, 246, 207,  # 数组元素
    240, 133, 184,  42,   4, 135, 123, 232, 120, 195,  92,  21,  # 数组元素
      2, 196, 190, 116,  60, 226,  46, 139  # 数组元素
};

/*
 * alpha^(127*i) + alpha^(125*i) mod 257
 */
static const unsigned short yoff_s_f[] = {  # 定义一个静态的无符号短整型数组
      2, 156, 118, 107,  45, 212, 111, 162,  97, 249, 211,   3,  # 数组元素
     49, 101, 151, 223, 189, 178, 253, 204,  76,  82, 232,  65,  # 数组元素
     96, 176, 161,  47, 189,  61, 248, 107,   0, 131, 133, 113,  # 数组元素
     17,  33,  12, 111, 251, 103,  57, 148,  47,  65, 249, 143,  # 数组元素
    189,   8, 204, 230, 205, 151, 187, 227, 247, 111, 140,   6,  # 数组元素
     77,  10,  21, 149, 255, 101, 139, 150, 212,  45, 146,  95,  # 数组元素
    160,   8,  46, 254, 208, 156, 106,  34,  68,  79,   4,  53,  # 数组元素
    181, 175,  25, 192, 161,  81,  96, 210,  68, 196,   9, 150,  # 数组元素
      0, 126, 124, 144, 240, 224, 245, 146,   6, 154, 200, 109,  # 数组元素
    这是一个整数列表，表示一组数字。
// 存储 beta^(255*i) mod 257 的结果
static const unsigned short yoff_b_n[] = {
    // 数组元素依次为 beta^(255*i) mod 257 的结果
};

// 存储 beta^(255*i) + beta^(253*i) mod 257 的结果
static const unsigned short yoff_b_f[] = {
    // 数组元素依次为 beta^(255*i) + beta^(253*i) mod 257 的结果
};
    # 定义一个包含 248,  90, 107,  64,   0,  88, 131, 243, 133,  59, 113, 115,
    # 17, 236,  33, 213,  12, 191, 111,  19, 251,  61, 103, 208,
    # 57,  35, 148, 248,  47, 116,  65, 119, 249, 178, 143,  40,
    # 189, 129,   8, 163, 204, 227, 230, 196, 205, 122, 151,  45,
    # 187,  19, 227,  72, 247, 125, 111, 121, 140, 220,   6, 107,
    # 77,  69,  10, 101,  21,  65, 149, 171, 255,  54, 101, 210,
    # 139,  43, 150, 151, 212, 164,  45, 237, 146, 184,  95,   6,
    # 160,  42,   8, 204,  46, 238, 254, 168, 208,  50, 156, 190,
    # 106, 127,  34, 234,  68,  55,  79,  18,   4, 130,  53, 208,
    # 181,  21, 175, 120,  25, 100, 192, 178, 161,  96,  81, 127,
    # 96, 227, 210, 248,  68,  10, 196,  31,   9, 167, 150, 193,
    # 0, 169, 126,  14, 124, 198, 144, 142, 240,  21, 224,  44,
    # 245,  66, 146, 238,   6, 196, 154,  49, 200, 222, 109,   9,
    # 210, 141, 192, 138,   8,  79, 114, 217,  68, 128, 249,  94,
    # 53,  30,  27,  61,  52, 135, 106, 212,  70, 238,  30, 185,
    # 10, 132, 146, 136, 117,  37, 251, 150, 180, 188, 247, 156,
    # 236, 192, 108,  86 的列表
    data = [
        248,  90, 107,  64,   0,  88, 131, 243, 133,  59, 113, 115,
        17, 236,  33, 213,  12, 191, 111,  19, 251,  61, 103, 208,
        57,  35, 148, 248,  47, 116,  65, 119, 249, 178, 143,  40,
        189, 129,   8, 163, 204, 227, 230, 196, 205, 122, 151,  45,
        187,  19, 227,  72, 247, 125, 111, 121, 140, 220,   6, 107,
        77,  69,  10, 101,  21,  65, 149, 171, 255,  54, 101, 210,
        139,  43, 150, 151, 212, 164,  45, 237, 146, 184,  95,   6,
        160,  42,   8, 204,  46, 238, 254, 168, 208,  50, 156, 190,
        106, 127,  34, 234,  68,  55,  79,  18,   4, 130,  53, 208,
        181,  21, 175, 120,  25, 100, 192, 178, 161,  96,  81, 127,
        96, 227, 210, 248,  68,  10, 196,  31,   9, 167, 150, 193,
        0, 169, 126,  14, 124, 198, 144, 142, 240,  21, 224,  44,
        245,  66, 146, 238,   6, 196, 154,  
# 定义一个宏函数，用于计算内部值
#define INNER(l, h, mm)   (((u32)((l) * (mm)) & 0xFFFFU) \
                          + ((u32)((h) * (mm)) << 16))

# 定义一个宏函数，用于计算小的W值
#define W_SMALL(sb, o1, o2, mm) \
    (INNER(q[8 * (sb) + 2 * 0 + o1], q[8 * (sb) + 2 * 0 + o2], mm), \
     INNER(q[8 * (sb) + 2 * 1 + o1], q[8 * (sb) + 2 * 1 + o2], mm), \
     INNER(q[8 * (sb) + 2 * 2 + o1], q[8 * (sb) + 2 * 2 + o2], mm), \
     INNER(q[8 * (sb) + 2 * 3 + o1], q[8 * (sb) + 2 * 3 + o2], mm)

# 定义一系列宏函数，用于计算不同位置的小的W值
#define WS_0_0   W_SMALL( 4,    0,    1, 185)
#define WS_0_1   W_SMALL( 6,    0,    1, 185)
#define WS_0_2   W_SMALL( 0,    0,    1, 185)
#define WS_0_3   W_SMALL( 2,    0,    1, 185)
#define WS_0_4   W_SMALL( 7,    0,    1, 185)
#define WS_0_5   W_SMALL( 5,    0,    1, 185)
#define WS_0_6   W_SMALL( 3,    0,    1, 185)
#define WS_0_7   W_SMALL( 1,    0,    1, 185)
#define WS_1_0   W_SMALL(15,    0,    1, 185)
#define WS_1_1   W_SMALL(11,    0,    1, 185)
#define WS_1_2   W_SMALL(12,    0,    1, 185)
#define WS_1_3   W_SMALL( 8,    0,    1, 185)
#define WS_1_4   W_SMALL( 9,    0,    1, 185)
#define WS_1_5   W_SMALL(13,    0,    1, 185)
#define WS_1_6   W_SMALL(10,    0,    1, 185)
#define WS_1_7   W_SMALL(14,    0,    1, 185)
#define WS_2_0   W_SMALL(17, -128,  -64, 233)
#define WS_2_1   W_SMALL(18, -128,  -64, 233)
#define WS_2_2   W_SMALL(23, -128,  -64, 233)
#define WS_2_3   W_SMALL(20, -128,  -64, 233)
#define WS_2_4   W_SMALL(22, -128,  -64, 233)
#define WS_2_5   W_SMALL(21, -128,  -64, 233)
#define WS_2_6   W_SMALL(16, -128,  -64, 233)
#define WS_2_7   W_SMALL(19, -128,  -64, 233)
#define WS_3_0   W_SMALL(30, -191, -127, 233)
#define WS_3_1   W_SMALL(24, -191, -127, 233)
#define WS_3_2   W_SMALL(25, -191, -127, 233)
#define WS_3_3   W_SMALL(31, -191, -127, 233)
#define WS_3_4   W_SMALL(27, -191, -127, 233)
#define WS_3_5   W_SMALL(29, -191, -127, 233)
#define WS_3_6   W_SMALL(28, -191, -127, 233)
#define WS_3_7   W_SMALL(26, -191, -127, 233)

# 定义一个宏函数，用于计算大的W值
#define W_BIG(sb, o1, o2, mm) \
    # 使用INNER函数对q列表中的元素进行处理，并将结果组成元组返回
    (INNER(q[16 * (sb) + 2 * 0 + o1], q[16 * (sb) + 2 * 0 + o2], mm), \
     INNER(q[16 * (sb) + 2 * 1 + o1], q[16 * (sb) + 2 * 1 + o2], mm), \
     INNER(q[16 * (sb) + 2 * 2 + o1], q[16 * (sb) + 2 * 2 + o2], mm), \
     INNER(q[16 * (sb) + 2 * 3 + o1], q[16 * (sb) + 2 * 3 + o2], mm), \
     INNER(q[16 * (sb) + 2 * 4 + o1], q[16 * (sb) + 2 * 4 + o2], mm), \
     INNER(q[16 * (sb) + 2 * 5 + o1], q[16 * (sb) + 2 * 5 + o2], mm), \
     INNER(q[16 * (sb) + 2 * 6 + o1], q[16 * (sb) + 2 * 6 + o2], mm), \
     INNER(q[16 * (sb) + 2 * 7 + o1], q[16 * (sb) + 2 * 7 + o2], mm))
# 定义宏，用于生成常量值
#define WB_0_0   W_BIG( 4,    0,    1, 185)
#define WB_0_1   W_BIG( 6,    0,    1, 185)
#define WB_0_2   W_BIG( 0,    0,    1, 185)
#define WB_0_3   W_BIG( 2,    0,    1, 185)
#define WB_0_4   W_BIG( 7,    0,    1, 185)
#define WB_0_5   W_BIG( 5,    0,    1, 185)
#define WB_0_6   W_BIG( 3,    0,    1, 185)
#define WB_0_7   W_BIG( 1,    0,    1, 185)
#define WB_1_0   W_BIG(15,    0,    1, 185)
#define WB_1_1   W_BIG(11,    0,    1, 185)
#define WB_1_2   W_BIG(12,    0,    1, 185)
#define WB_1_3   W_BIG( 8,    0,    1, 185)
#define WB_1_4   W_BIG( 9,    0,    1, 185)
#define WB_1_5   W_BIG(13,    0,    1, 185)
#define WB_1_6   W_BIG(10,    0,    1, 185)
#define WB_1_7   W_BIG(14,    0,    1, 185)
#define WB_2_0   W_BIG(17, -256, -128, 233)
#define WB_2_1   W_BIG(18, -256, -128, 233)
#define WB_2_2   W_BIG(23, -256, -128, 233)
#define WB_2_3   W_BIG(20, -256, -128, 233)
#define WB_2_4   W_BIG(22, -256, -128, 233)
#define WB_2_5   W_BIG(21, -256, -128, 233)
#define WB_2_6   W_BIG(16, -256, -128, 233)
#define WB_2_7   W_BIG(19, -256, -128, 233)
#define WB_3_0   W_BIG(30, -383, -255, 233)
#define WB_3_1   W_BIG(24, -383, -255, 233)
#define WB_3_2   W_BIG(25, -383, -255, 233)
#define WB_3_3   W_BIG(31, -383, -255, 233)
#define WB_3_4   W_BIG(27, -383, -255, 233)
#define WB_3_5   W_BIG(29, -383, -255, 233)
#define WB_3_6   W_BIG(28, -383, -255, 233)
#define WB_3_7   W_BIG(26, -383, -255, 233)

#define IF(x, y, z)    ((((y) ^ (z)) & (x)) ^ (z))
#define MAJ(x, y, z)   (((x) & (y)) | (((x) | (y)) & (z)))

#define PP4_0_0   1
#define PP4_0_1   0
#define PP4_0_2   3
#define PP4_0_3   2
#define PP4_1_0   2
#define PP4_1_1   3
#define PP4_1_2   0
#define PP4_1_3   1
#define PP4_2_0   3
#define PP4_2_1   2
#define PP4_2_2   1
#define PP4_2_3   0

#define PP8_0_0   1
#define PP8_0_1   0
#define PP8_0_2   3
#define PP8_0_3   2
#define PP8_0_4   5
#define PP8_0_5   4
#define PP8_0_6   7
#define PP8_0_7   6

#define PP8_1_0   6
#define PP8_1_1   7
#define PP8_1_2   4
#define PP8_1_3   5
#define PP8_1_4   2
#define PP8_1_5   3
#define PP8_1_6   0
#define PP8_1_7   1

#define PP8_2_0   2
#define PP8_2_1   3
#define PP8_2_2   0
#define PP8_2_3   1
#define PP8_2_4   6
#define PP8_2_5   7
#define PP8_2_6   4
#define PP8_2_7   5

#define PP8_3_0   3
#define PP8_3_1   2
#define PP8_3_2   1
#define PP8_3_3   0
#define PP8_3_4   7
#define PP8_3_5   6
#define PP8_3_6   5
#define PP8_3_7   4

#define PP8_4_0   5
#define PP8_4_1   4
#define PP8_4_2   7
#define PP8_4_3   6
#define PP8_4_4   1
#define PP8_4_5   0
#define PP8_4_6   3
#define PP8_4_7   2

#define PP8_5_0   7
#define PP8_5_1   6
#define PP8_5_2   5
#define PP8_5_3   4
#define PP8_5_4   3
#define PP8_5_5   2
#define PP8_5_6   1
#define PP8_5_7   0

#define PP8_6_0   4
#define PP8_6_1   5
#define PP8_6_2   6
#define PP8_6_3   7
#define PP8_6_4   0
#define PP8_6_5   1
#define PP8_6_6   2
#define PP8_6_7   3

这部分代码定义了一系列的宏，用于给不同的变量名赋予特定的值。


#if SPH_SIMD_NOCOPY

#define DECL_STATE_SMALL
#define READ_STATE_SMALL(sc)
#define WRITE_STATE_SMALL(sc)
#define DECL_STATE_BIG
#define READ_STATE_BIG(sc)
#define WRITE_STATE_BIG(sc)

#else

#define DECL_STATE_SMALL   \
    u32 A0, A1, A2, A3, B0, B1, B2, B3, C0, C1, C2, C3, D0, D1, D2, D3;

#define READ_STATE_SMALL(sc)   do { \
        A0 = (sc)->state[ 0]; \
        A1 = (sc)->state[ 1]; \
        A2 = (sc)->state[ 2]; \
        A3 = (sc)->state[ 3]; \
        B0 = (sc)->state[ 4]; \
        B1 = (sc)->state[ 5]; \
        B2 = (sc)->state[ 6]; \
        B3 = (sc)->state[ 7]; \
        C0 = (sc)->state[ 8]; \
        C1 = (sc)->state[ 9]; \
        C2 = (sc)->state[10]; \
        C3 = (sc)->state[11]; \
        D0 = (sc)->state[12]; \
        D1 = (sc)->state[13]; \
        D2 = (sc)->state[14]; \
        D3 = (sc)->state[15]; \
    } while (0)

这部分代码使用了条件编译，根据`SPH_SIMD_NOCOPY`宏的定义来选择不同的代码块。

如果`SPH_SIMD_NOCOPY`宏被定义，则以下宏都为空。

如果`SPH_SIMD_NOCOPY`宏未被定义，则以下宏定义了一系列变量，并且定义了一个宏函数`READ_STATE_SMALL(sc)`，用于将`sc`结构体中的`state`数组的值赋给这些变量。
# 定义宏，将给定的参数赋值给状态数组中的对应位置
#define WRITE_STATE_SMALL(sc)   do { \
        (sc)->state[ 0] = A0; \
        (sc)->state[ 1] = A1; \
        (sc)->state[ 2] = A2; \
        (sc)->state[ 3] = A3; \
        (sc)->state[ 4] = B0; \
        (sc)->state[ 5] = B1; \
        (sc)->state[ 6] = B2; \
        (sc)->state[ 7] = B3; \
        (sc)->state[ 8] = C0; \
        (sc)->state[ 9] = C1; \
        (sc)->state[10] = C2; \
        (sc)->state[11] = C3; \
        (sc)->state[12] = D0; \
        (sc)->state[13] = D1; \
        (sc)->state[14] = D2; \
        (sc)->state[15] = D3; \
    } while (0)

# 定义宏，声明大状态变量
#define DECL_STATE_BIG   \
    u32 A0, A1, A2, A3, A4, A5, A6, A7; \
    u32 B0, B1, B2, B3, B4, B5, B6, B7; \
    u32 C0, C1, C2, C3, C4, C5, C6, C7; \
    u32 D0, D1, D2, D3, D4, D5, D6, D7;

# 定义宏，将状态数组中的值赋给给定的变量
#define READ_STATE_BIG(sc)   do { \
        A0 = (sc)->state[ 0]; \
        A1 = (sc)->state[ 1]; \
        A2 = (sc)->state[ 2]; \
        A3 = (sc)->state[ 3]; \
        A4 = (sc)->state[ 4]; \
        A5 = (sc)->state[ 5]; \
        A6 = (sc)->state[ 6]; \
        A7 = (sc)->state[ 7]; \
        B0 = (sc)->state[ 8]; \
        B1 = (sc)->state[ 9]; \
        B2 = (sc)->state[10]; \
        B3 = (sc)->state[11]; \
        B4 = (sc)->state[12]; \
        B5 = (sc)->state[13]; \
        B6 = (sc)->state[14]; \
        B7 = (sc)->state[15]; \
        C0 = (sc)->state[16]; \
        C1 = (sc)->state[17]; \
        C2 = (sc)->state[18]; \
        C3 = (sc)->state[19]; \
        C4 = (sc)->state[20]; \
        C5 = (sc)->state[21]; \
        C6 = (sc)->state[22]; \
        C7 = (sc)->state[23]; \
        D0 = (sc)->state[24]; \
        D1 = (sc)->state[25]; \
        D2 = (sc)->state[26]; \
        D3 = (sc)->state[27]; \
        D4 = (sc)->state[28]; \
        D5 = (sc)->state[29]; \
        D6 = (sc)->state[30]; \
        D7 = (sc)->state[31]; \
    } while (0)
#define WRITE_STATE_BIG(sc)   do { \  # 定义宏，用于将参数中的值写入状态数组中
        (sc)->state[ 0] = A0; \  # 将 A0 的值写入状态数组的第一个位置
        (sc)->state[ 1] = A1; \  # 将 A1 的值写入状态数组的第二个位置
        (sc)->state[ 2] = A2; \  # 将 A2 的值写入状态数组的第三个位置
        (sc)->state[ 3] = A3; \  # 将 A3 的值写入状态数组的第四个位置
        (sc)->state[ 4] = A4; \  # 将 A4 的值写入状态数组的第五个位置
        (sc)->state[ 5] = A5; \  # 将 A5 的值写入状态数组的第六个位置
        (sc)->state[ 6] = A6; \  # 将 A6 的值写入状态数组的第七个位置
        (sc)->state[ 7] = A7; \  # 将 A7 的值写入状态数组的第八个位置
        (sc)->state[ 8] = B0; \  # 将 B0 的值写入状态数组的第九个位置
        (sc)->state[ 9] = B1; \  # 将 B1 的值写入状态数组的第十个位置
        (sc)->state[10] = B2; \  # 将 B2 的值写入状态数组的第十一个位置
        (sc)->state[11] = B3; \  # 将 B3 的值写入状态数组的第十二个位置
        (sc)->state[12] = B4; \  # 将 B4 的值写入状态数组的第十三个位置
        (sc)->state[13] = B5; \  # 将 B5 的值写入状态数组的第十四个位置
        (sc)->state[14] = B6; \  # 将 B6 的值写入状态数组的第十五个位置
        (sc)->state[15] = B7; \  # 将 B7 的值写入状态数组的第十六个位置
        (sc)->state[16] = C0; \  # 将 C0 的值写入状态数组的第十七个位置
        (sc)->state[17] = C1; \  # 将 C1 的值写入状态数组的第十八个位置
        (sc)->state[18] = C2; \  # 将 C2 的值写入状态数组的第十九个位置
        (sc)->state[19] = C3; \  # 将 C3 的值写入状态数组的第二十个位置
        (sc)->state[20] = C4; \  # 将 C4 的值写入状态数组的第二十一个位置
        (sc)->state[21] = C5; \  # 将 C5 的值写入状态数组的第二十二个位置
        (sc)->state[22] = C6; \  # 将 C6 的值写入状态数组的第二十三个位置
        (sc)->state[23] = C7; \  # 将 C7 的值写入状态数组的第二十四个位置
        (sc)->state[24] = D0; \  # 将 D0 的值写入状态数组的第二十五个位置
        (sc)->state[25] = D1; \  # 将 D1 的值写入状态数组的第二十六个位置
        (sc)->state[26] = D2; \  # 将 D2 的值写入状态数组的第二十七个位置
        (sc)->state[27] = D3; \  # 将 D3 的值写入状态数组的第二十八个位置
        (sc)->state[28] = D4; \  # 将 D4 的值写入状态数组的第二十九个位置
        (sc)->state[29] = D5; \  # 将 D5 的值写入状态数组的第三十个位置
        (sc)->state[30] = D6; \  # 将 D6 的值写入状态数组的第三十一个位置
        (sc)->state[31] = D7; \  # 将 D7 的值写入状态数组的第三十二个位置
    } while (0)  # 宏定义结束
#endif  # 条件编译结束

#define STEP_ELT(n, w, fun, s, ppb)   do { \  # 定义宏，用于执行一系列步骤
        u32 tt = T32(D ## n + (w) + fun(A ## n, B ## n, C ## n)); \  # 计算 tt 的值
        A ## n = T32(ROL32(tt, s) + XCAT(tA, XCAT(ppb, n))); \  # 计算 A 的值
        D ## n = C ## n; \  # 将 C 的值赋给 D
        C ## n = B ## n; \  # 将 B 的值赋给 C
        B ## n = tA ## n; \  # 将 tA 的值赋给 B
    } while (0)  # 宏定义结束

#define STEP_SMALL(w0, w1, w2, w3, fun, r, s, pp4b)   do { \  # 定义宏，用于执行一系列步骤
        u32 tA0 = ROL32(A0, r); \  # 计算 tA0 的值
        u32 tA1 = ROL32(A1, r); \  # 计算 tA1 的值
        u32 tA2 = ROL32(A2, r); \  # 计算 tA2 的值
        u32 tA3 = ROL32(A3, r); \  # 计算 tA3 的值
        STEP_ELT(0, w0, fun, s, pp4b); \  # 执行 STEP_ELT 宏
        STEP_ELT(1, w1, fun, s, pp4b); \  # 执行 STEP_ELT 宏
        STEP_ELT(2, w2, fun, s, pp4b); \  # 执行 STEP_ELT 宏
        STEP_ELT(3, w3, fun, s, pp4b); \  # 执行 STEP_ELT 宏
    } while (0)  # 宏定义结束
# 定义一个宏，用于进行大步骤的处理
#define STEP_BIG(w0, w1, w2, w3, w4, w5, w6, w7, fun, r, s, pp8b)   do { \
        # 对A0~A7进行循环左移操作
        u32 tA0 = ROL32(A0, r); \
        u32 tA1 = ROL32(A1, r); \
        u32 tA2 = ROL32(A2, r); \
        u32 tA3 = ROL32(A3, r); \
        u32 tA4 = ROL32(A4, r); \
        u32 tA5 = ROL32(A5, r); \
        u32 tA6 = ROL32(A6, r); \
        u32 tA7 = ROL32(A7, r); \
        # 对w0~w7进行单步骤的处理
        STEP_ELT(0, w0, fun, s, pp8b); \
        STEP_ELT(1, w1, fun, s, pp8b); \
        STEP_ELT(2, w2, fun, s, pp8b); \
        STEP_ELT(3, w3, fun, s, pp8b); \
        STEP_ELT(4, w4, fun, s, pp8b); \
        STEP_ELT(5, w5, fun, s, pp8b); \
        STEP_ELT(6, w6, fun, s, pp8b); \
        STEP_ELT(7, w7, fun, s, pp8b); \
    } while (0)

# 定义一系列宏，用于表示M3_0_0到M3_7_2的值
#define M3_0_0   0_
#define M3_1_0   1_
#define M3_2_0   2_
#define M3_3_0   0_
#define M3_4_0   1_
#define M3_5_0   2_
#define M3_6_0   0_
#define M3_7_0   1_

#define M3_0_1   1_
#define M3_1_1   2_
#define M3_2_1   0_
#define M3_3_1   1_
#define M3_4_1   2_
#define M3_5_1   0_
#define M3_6_1   1_
#define M3_7_1   2_

#define M3_0_2   2_
#define M3_1_2   0_
#define M3_2_2   1_
#define M3_3_2   2_
#define M3_4_2   0_
#define M3_5_2   1_
#define M3_6_2   2_
#define M3_7_2   0_

# 定义一个宏，用于进行小步骤的处理
#define STEP_SMALL_(w, fun, r, s, pp4b)   STEP_SMALL w, fun, r, s, pp4b)
# 定义宏函数ONE_ROUND_SMALL，用于执行一轮小步骤
# 参数ri: 轮数
# 参数isp: 轮数对应的索引
# 参数p0, p1, p2, p3: 四个变量
#define ONE_ROUND_SMALL(ri, isp, p0, p1, p2, p3)   do { \
        # 调用宏函数STEP_SMALL_，执行小步骤，更新变量p0
        STEP_SMALL_(WS_ ## ri ## 0, \
            IF,  p0, p1, XCAT(PP4_, M3_0_ ## isp)); \
        # 调用宏函数STEP_SMALL_，执行小步骤，更新变量p1
        STEP_SMALL_(WS_ ## ri ## 1, \
            IF,  p1, p2, XCAT(PP4_, M3_1_ ## isp)); \
        # 调用宏函数STEP_SMALL_，执行小步骤，更新变量p2
        STEP_SMALL_(WS_ ## ri ## 2, \
            IF,  p2, p3, XCAT(PP4_, M3_2_ ## isp)); \
        # 调用宏函数STEP_SMALL_，执行小步骤，更新变量p3
        STEP_SMALL_(WS_ ## ri ## 3, \
            IF,  p3, p0, XCAT(PP4_, M3_3_ ## isp)); \
        # 调用宏函数STEP_SMALL_，执行小步骤，更新变量p0
        STEP_SMALL_(WS_ ## ri ## 4, \
            MAJ, p0, p1, XCAT(PP4_, M3_4_ ## isp)); \
        # 调用宏函数STEP_SMALL_，执行小步骤，更新变量p1
        STEP_SMALL_(WS_ ## ri ## 5, \
            MAJ, p1, p2, XCAT(PP4_, M3_5_ ## isp)); \
        # 调用宏函数STEP_SMALL_，执行小步骤，更新变量p2
        STEP_SMALL_(WS_ ## ri ## 6, \
            MAJ, p2, p3, XCAT(PP4_, M3_6_ ## isp)); \
        # 调用宏函数STEP_SMALL_，执行小步骤，更新变量p3
        STEP_SMALL_(WS_ ## ri ## 7, \
            MAJ, p3, p0, XCAT(PP4_, M3_7_ ## isp)); \
    } while (0)

# 定义宏函数M7_0_0, M7_1_0, ..., M7_7_3，用于表示索引
#define M7_0_0   0_
#define M7_1_0   1_
#define M7_2_0   2_
#define M7_3_0   3_
#define M7_4_0   4_
#define M7_5_0   5_
#define M7_6_0   6_
#define M7_7_0   0_

#define M7_0_1   1_
#define M7_1_1   2_
#define M7_2_1   3_
#define M7_3_1   4_
#define M7_4_1   5_
#define M7_5_1   6_
#define M7_6_1   0_
#define M7_7_1   1_

#define M7_0_2   2_
#define M7_1_2   3_
#define M7_2_2   4_
#define M7_3_2   5_
#define M7_4_2   6_
#define M7_5_2   0_
#define M7_6_2   1_
#define M7_7_2   2_

#define M7_0_3   3_
#define M7_1_3   4_
#define M7_2_3   5_
#define M7_3_3   6_
#define M7_4_3   0_
#define M7_5_3   1_
#define M7_6_3   2_
#define M7_7_3   3_

#define STEP_BIG_(w, fun, r, s, pp8b)   STEP_BIG w, fun, r, s, pp8b)
#define ONE_ROUND_BIG(ri, isp, p0, p1, p2, p3)   do { \  # 定义一个宏，用于执行一轮大循环
        STEP_BIG_(WB_ ## ri ## 0, \  # 调用 STEP_BIG_ 宏，执行大循环的第一步
            IF,  p0, p1, XCAT(PP8_, M7_0_ ## isp)); \  # 使用 IF 条件执行步骤，传入参数 p0, p1 和 XCAT(PP8_, M7_0_ ## isp)
        STEP_BIG_(WB_ ## ri ## 1, \  # 调用 STEP_BIG_ 宏，执行大循环的第二步
            IF,  p1, p2, XCAT(PP8_, M7_1_ ## isp)); \  # 使用 IF 条件执行步骤，传入参数 p1, p2 和 XCAT(PP8_, M7_1_ ## isp)
        STEP_BIG_(WB_ ## ri ## 2, \  # 调用 STEP_BIG_ 宏，执行大循环的第三步
            IF,  p2, p3, XCAT(PP8_, M7_2_ ## isp)); \  # 使用 IF 条件执行步骤，传入参数 p2, p3 和 XCAT(PP8_, M7_2_ ## isp)
        STEP_BIG_(WB_ ## ri ## 3, \  # 调用 STEP_BIG_ 宏，执行大循环的第四步
            IF,  p3, p0, XCAT(PP8_, M7_3_ ## isp)); \  # 使用 IF 条件执行步骤，传入参数 p3, p0 和 XCAT(PP8_, M7_3_ ## isp)
        STEP_BIG_(WB_ ## ri ## 4, \  # 调用 STEP_BIG_ 宏，执行大循环的第五步
            MAJ, p0, p1, XCAT(PP8_, M7_4_ ## isp)); \  # 使用 MAJ 条件执行步骤，传入参数 p0, p1 和 XCAT(PP8_, M7_4_ ## isp)
        STEP_BIG_(WB_ ## ri ## 5, \  # 调用 STEP_BIG_ 宏，执行大循环的第六步
            MAJ, p1, p2, XCAT(PP8_, M7_5_ ## isp)); \  # 使用 MAJ 条件执行步骤，传入参数 p1, p2 和 XCAT(PP8_, M7_5_ ## isp)
        STEP_BIG_(WB_ ## ri ## 6, \  # 调用 STEP_BIG_ 宏，执行大循环的第七步
            MAJ, p2, p3, XCAT(PP8_, M7_6_ ## isp)); \  # 使用 MAJ 条件执行步骤，传入参数 p2, p3 和 XCAT(PP8_, M7_6_ ## isp)
        STEP_BIG_(WB_ ## ri ## 7, \  # 调用 STEP_BIG_ 宏，执行大循环的第八步
            MAJ, p3, p0, XCAT(PP8_, M7_7_ ## isp)); \  # 使用 MAJ 条件执行步骤，传入参数 p3, p0 和 XCAT(PP8_, M7_7_ ## isp)
    } while (0)  # 结束 do-while 循环

#if SPH_SMALL_FOOTPRINT_SIMD  # 如果定义了 SPH_SMALL_FOOTPRINT_SIMD

#define A0   state[ 0]  # 定义宏 A0，表示 state 数组的第一个元素
#define A1   state[ 1]  # 定义宏 A1，表示 state 数组的第二个元素
#define A2   state[ 2]  # 定义宏 A2，表示 state 数组的第三个元素
#define A3   state[ 3]  # 定义宏 A3，表示 state 数组的第四个元素
#define B0   state[ 4]  # 定义宏 B0，表示 state 数组的第五个元素
#define B1   state[ 5]  # 定义宏 B1，表示 state 数组的第六个元素
#define B2   state[ 6]  # 定义宏 B2，表示 state 数组的第七个元素
#define B3   state[ 7]  # 定义宏 B3，表示 state 数组的第八个元素
#define C0   state[ 8]  # 定义宏 C0，表示 state 数组的第九个元素
#define C1   state[ 9]  # 定义宏 C1，表示 state 数组的第十个元素
#define C2   state[10]  # 定义宏 C2，表示 state 数组的第十一个元素
#define C3   state[11]  # 定义宏 C3，表示 state 数组的第十二个元素
#define D0   state[12]  # 定义宏 D0，表示 state 数组的第十三个元素
#define D1   state[13]  # 定义宏 D1，表示 state 数组的第十四个元素
#define D2   state[14]  # 定义宏 D2，表示 state 数组的第十五个元素
#define D3   state[15]  # 定义宏 D3，表示 state 数组的第十六个元素

#define STEP2_ELT(n, w, fun, s, ppb)   do { \  # 定义一个宏，用于执行步骤2的元素
        u32 tt = T32(D ## n + (w) + fun(A ## n, B ## n, C ## n)); \  # 计算 tt 的值
        A ## n = T32(ROL32(tt, s) + tA[(ppb) ^ n]); \  # 计算 A 的值
        D ## n = C ## n; \  # 将 C 的值赋给 D
        C ## n = B ## n; \  # 将 B 的值赋给 C
        B ## n = tA[n]; \  # 将 tA[n] 的值赋给 B
    } while (0)  # 结束 do-while 循环

#define STEP2_SMALL(w0, w1, w2, w3, fun, r, s, pp4b)   do { \  # 定义一个宏，用于执行步骤2的小循环
        u32 tA[4]; \  # 定义一个长度为4的数组 tA
        tA[0] = ROL32(A0, r); \  # 计算 tA[0] 的值
        tA[1] = ROL32(A1, r); \  # 计算 tA[1] 的值
        tA[2] = ROL32(A2, r); \  # 计算 tA[2] 的值
        tA[3] = ROL32(A3, r); \  # 计算 tA[3] 的值
        STEP2_ELT(0, w0, fun, s, pp4b); \  # 调用 STEP2_ELT 宏，执行步骤2的元素
        STEP2_ELT(1, w1, fun, s, pp4b); \  # 调用 STEP2_ELT 宏，执行步骤2的元素
        STEP2_ELT(2, w2, fun, s, pp4b); \  # 调用 STEP2_ELT 宏，执行步骤2的元素
        STEP2_ELT(3, w3, fun, s, pp4b); \  # 调用 STEP2_ELT 宏，执行步骤2的元素
    } while (0)  # 结束 do-while 循环

static void
one_round_small(u32 *state, u32 *w, int isp, int p0, int p1, int p2, int p3)  # 定义一个静态函数 one_round_small，接受参数 state, w, isp, p0, p1, p2, p3
    // 定义一个包含固定值的整型数组
    static const int pp4k[] = { 1, 2, 3, 1, 2, 3, 1, 2, 3, 1, 2 };

    // 调用名为 STEP2_SMALL 的函数，传入参数 w[ 0], w[ 1], w[ 2], w[ 3], IF,  p0, p1, pp4k[isp + 0]
    STEP2_SMALL(w[ 0], w[ 1], w[ 2], w[ 3], IF,  p0, p1, pp4k[isp + 0]);
    // 调用名为 STEP2_SMALL 的函数，传入参数 w[ 4], w[ 5], w[ 6], w[ 7], IF,  p1, p2, pp4k[isp + 1]
    STEP2_SMALL(w[ 4], w[ 5], w[ 6], w[ 7], IF,  p1, p2, pp4k[isp + 1]);
    // 调用名为 STEP2_SMALL 的函数，传入参数 w[ 8], w[ 9], w[10], w[11], IF,  p2, p3, pp4k[isp + 2]
    STEP2_SMALL(w[ 8], w[ 9], w[10], w[11], IF,  p2, p3, pp4k[isp + 2]);
    // 调用名为 STEP2_SMALL 的函数，传入参数 w[12], w[13], w[14], w[15], IF,  p3, p0, pp4k[isp + 3]
    STEP2_SMALL(w[12], w[13], w[14], w[15], IF,  p3, p0, pp4k[isp + 3]);
    // 调用名为 STEP2_SMALL 的函数，传入参数 w[16], w[17], w[18], w[19], MAJ, p0, p1, pp4k[isp + 4]
    STEP2_SMALL(w[16], w[17], w[18], w[19], MAJ, p0, p1, pp4k[isp + 4]);
    // 调用名为 STEP2_SMALL 的函数，传入参数 w[20], w[21], w[22], w[23], MAJ, p1, p2, pp4k[isp + 5]
    STEP2_SMALL(w[20], w[21], w[22], w[23], MAJ, p1, p2, pp4k[isp + 5]);
    // 调用名为 STEP2_SMALL 的函数，传入参数 w[24], w[25], w[26], w[27], MAJ, p2, p3, pp4k[isp + 6]
    STEP2_SMALL(w[24], w[25], w[26], w[27], MAJ, p2, p3, pp4k[isp + 6]);
    // 调用名为 STEP2_SMALL 的函数，传入参数 w[28], w[29], w[30], w[31], MAJ, p3, p0, pp4k[isp + 7]
    STEP2_SMALL(w[28], w[29], w[30], w[31], MAJ, p3, p0, pp4k[isp + 7]);
# 压缩小块数据的函数
static void
compress_small(sph_simd_small_context *sc, int last)
{
    # 定义变量
    unsigned char *x;
    s32 q[128];
    int i;
    u32 w[32];
    u32 state[16];
    size_t u;

    # 预定义的常量数组
    static const size_t wsp[32] = {
         4 << 3,  6 << 3,  0 << 3,  2 << 3,
         7 << 3,  5 << 3,  3 << 3,  1 << 3,
        15 << 3, 11 << 3, 12 << 3,  8 << 3,
         9 << 3, 13 << 3, 10 << 3, 14 << 3,
        17 << 3, 18 << 3, 23 << 3, 20 << 3,
        22 << 3, 21 << 3, 16 << 3, 19 << 3,
        30 << 3, 24 << 3, 25 << 3, 31 << 3,
        27 << 3, 29 << 3, 28 << 3, 26 << 3
    };

    # 将 sc->buf 赋值给 x
    x = sc->buf;
    # 调用 FFT128 函数
    FFT128(0, 1, 0, ll);
    # 如果是最后一个块
    if (last) {
        # 对 q 数组进行处理
        for (i = 0; i < 128; i ++) {
            s32 tq;

            tq = q[i] + yoff_s_f[i];
            tq = REDS2(tq);
            tq = REDS1(tq);
            tq = REDS1(tq);
            q[i] = (tq <= 128 ? tq : tq - 257);
        }
    } else {
        # 对 q 数组进行处理
        for (i = 0; i < 128; i ++) {
            s32 tq;

            tq = q[i] + yoff_s_n[i];
            tq = REDS2(tq);
            tq = REDS1(tq);
            tq = REDS1(tq);
            q[i] = (tq <= 128 ? tq : tq - 257);
        }
    }

    # 将 sc->state 和 x 中的数据进行异或运算，并赋值给 state 数组
    for (i = 0; i < 16; i += 4) {
        state[i + 0] = sc->state[i + 0]
            ^ sph_dec32le_aligned(x + 4 * (i + 0));
        state[i + 1] = sc->state[i + 1]
            ^ sph_dec32le_aligned(x + 4 * (i + 1));
        state[i + 2] = sc->state[i + 2]
            ^ sph_dec32le_aligned(x + 4 * (i + 2));
        state[i + 3] = sc->state[i + 3]
            ^ sph_dec32le_aligned(x + 4 * (i + 3));
    }
}
#define WSREAD(sb, o1, o2, mm)   do { \
        for (u = 0; u < 32; u += 4) { \
            size_t v = wsp[(u >> 2) + (sb)]; \
            w[u + 0] = INNER(q[v + 2 * 0 + (o1)], \
                q[v + 2 * 0 + (o2)], mm); \
            w[u + 1] = INNER(q[v + 2 * 1 + (o1)], \
                q[v + 2 * 1 + (o2)], mm); \
            w[u + 2] = INNER(q[v + 2 * 2 + (o1)], \
                q[v + 2 * 2 + (o2)], mm); \
            w[u + 3] = INNER(q[v + 2 * 3 + (o1)], \
                q[v + 2 * 3 + (o2)], mm); \
        } \
    } while (0)

这段代码定义了一个宏函数`WSREAD`，用于读取数据并进行一系列操作。


WSREAD( 0,    0,    1, 185);

调用`WSREAD`宏函数，传入参数`0, 0, 1, 185`。


one_round_small(state, w, 0,  3, 23, 17, 27);

调用`one_round_small`函数，传入参数`state, w, 0, 3, 23, 17, 27`。


WSREAD( 8,    0,    1, 185);

调用`WSREAD`宏函数，传入参数`8, 0, 1, 185`。


one_round_small(state, w, 2, 28, 19, 22,  7);

调用`one_round_small`函数，传入参数`state, w, 2, 28, 19, 22, 7`。


WSREAD(16, -128,  -64, 233);

调用`WSREAD`宏函数，传入参数`16, -128, -64, 233`。


one_round_small(state, w, 1, 29,  9, 15,  5);

调用`one_round_small`函数，传入参数`state, w, 1, 29, 9, 15, 5`。


WSREAD(24, -191, -127, 233);

调用`WSREAD`宏函数，传入参数`24, -191, -127, 233`。


one_round_small(state, w, 0,  4, 13, 10, 25);

调用`one_round_small`函数，传入参数`state, w, 0, 4, 13, 10, 25`。


#undef WSREAD

取消对`WSREAD`宏函数的定义。


STEP_SMALL(sc->state[ 0], sc->state[ 1], sc->state[ 2], sc->state[ 3],
        IF,  4, 13, PP4_2_);

调用`STEP_SMALL`函数，传入参数`sc->state[0], sc->state[1], sc->state[2], sc->state[3], IF, 4, 13, PP4_2_`。


STEP_SMALL(sc->state[ 4], sc->state[ 5], sc->state[ 6], sc->state[ 7],
        IF, 13, 10, PP4_0_);

调用`STEP_SMALL`函数，传入参数`sc->state[4], sc->state[5], sc->state[6], sc->state[7], IF, 13, 10, PP4_0_`。


STEP_SMALL(sc->state[ 8], sc->state[ 9], sc->state[10], sc->state[11],
        IF, 10, 25, PP4_1_);

调用`STEP_SMALL`函数，传入参数`sc->state[8], sc->state[9], sc->state[10], sc->state[11], IF, 10, 25, PP4_1_`。


STEP_SMALL(sc->state[12], sc->state[13], sc->state[14], sc->state[15],
        IF, 25,  4, PP4_2_);

调用`STEP_SMALL`函数，传入参数`sc->state[12], sc->state[13], sc->state[14], sc->state[15], IF, 25, 4, PP4_2_`。


memcpy(sc->state, state, sizeof state);

将`state`的内容复制到`sc->state`中。
#define D3   (sc->state[15])
#endif

static void
compress_small(sph_simd_small_context *sc, int last)
{
    // 定义指向 unsigned char 类型的指针 x
    unsigned char *x;
    // 定义长度为 128 的整型数组 q
    s32 q[128];
    // 定义整型变量 i
    int i;
    // 声明 STATE_SMALL 宏定义
    DECL_STATE_SMALL
    // 如果定义了 SPH_SIMD_NOCOPY 宏
    #if SPH_SIMD_NOCOPY
        // 将 sc->state 数组的内容拷贝到 saved 数组中
        memcpy(saved, sc->state, sizeof saved);
    #endif
    // 将 sc->buf 的地址赋值给指针 x
    x = sc->buf;
    // 调用 FFT128 函数，参数为 0, 1, 0, ll
    FFT128(0, 1, 0, ll);
    // 如果 last 为真
    if (last) {
        // 循环 128 次
        for (i = 0; i < 128; i ++) {
            // 定义整型变量 tq
            s32 tq;
            // 将 q[i] 和 yoff_s_f[i] 相加，结果赋值给 tq
            tq = q[i] + yoff_s_f[i];
            // 对 tq 进行 REDS2 运算
            tq = REDS2(tq);
            // 对 tq 进行 REDS1 运算
            tq = REDS1(tq);
            // 对 tq 进行 REDS1 运算
            tq = REDS1(tq);
            // 如果 tq 小于等于 128，则将 tq 赋值给 q[i]，否则将 tq - 257 赋值给 q[i]
            q[i] = (tq <= 128 ? tq : tq - 257);
        }
    } else {
        // 循环 128 次
        for (i = 0; i < 128; i ++) {
            // 定义整型变量 tq
            s32 tq;
            // 将 q[i] 和 yoff_s_n[i] 相加，结果赋值给 tq
            tq = q[i] + yoff_s_n[i];
            // 对 tq 进行 REDS2 运算
            tq = REDS2(tq);
            // 对 tq 进行 REDS1 运算
            tq = REDS1(tq);
            // 对 tq 进行 REDS1 运算
            tq = REDS1(tq);
            // 如果 tq 小于等于 128，则将 tq 赋值给 q[i]，否则将 tq - 257 赋值给 q[i]
            q[i] = (tq <= 128 ? tq : tq - 257);
        }
    }
    // 读取 sc 的状态
    READ_STATE_SMALL(sc);
    // 对 A0 进行异或运算
    A0 ^= sph_dec32le_aligned(x +  0);
    // 对 A1 进行异或运算
    A1 ^= sph_dec32le_aligned(x +  4);
    // 对 A2 进行异或运算
    A2 ^= sph_dec32le_aligned(x +  8);
    // 对 A3 进行异或运算
    A3 ^= sph_dec32le_aligned(x + 12);
    // 对 B0 进行异或运算
    B0 ^= sph_dec32le_aligned(x + 16);
    // 对 B1 进行异或运算
    B1 ^= sph_dec32le_aligned(x + 20);
    // 对 B2 进行异或运算
    B2 ^= sph_dec32le_aligned(x + 24);
    // 对 B3 进行异或运算
    B3 ^= sph_dec32le_aligned(x + 28);
    // 对 C0 进行异或运算
    C0 ^= sph_dec32le_aligned(x + 32);
    // 对 C1 进行异或运算
    C1 ^= sph_dec32le_aligned(x + 36);
    // 对 C2 进行异或运算
    C2 ^= sph_dec32le_aligned(x + 40);
    // 对 C3 进行异或运算
    C3 ^= sph_dec32le_aligned(x + 44);
    // 对 D0 进行异或运算
    D0 ^= sph_dec32le_aligned(x + 48);
    // 对 D1 进行异或运算
    D1 ^= sph_dec32le_aligned(x + 52);
    // 对 D2 进行异或运算
    D2 ^= sph_dec32le_aligned(x + 56);
    // 对 D3 进行异或运算
    D3 ^= sph_dec32le_aligned(x + 60);
    // 调用 ONE_ROUND_SMALL 函数，参数为 0_, 0,  3, 23, 17, 27
    ONE_ROUND_SMALL(0_, 0,  3, 23, 17, 27);
    // 调用 ONE_ROUND_SMALL 函数，参数为 1_, 2, 28, 19, 22,  7
    ONE_ROUND_SMALL(1_, 2, 28, 19, 22,  7);
    // 调用 ONE_ROUND_SMALL 函数，参数为 2_, 1, 29,  9, 15,  5
    ONE_ROUND_SMALL(2_, 1, 29,  9, 15,  5);
    // 调用 ONE_ROUND_SMALL 函数，参数为 3_, 0,  4, 13, 10, 25
    ONE_ROUND_SMALL(3_, 0,  4, 13, 10, 25);
    // 如果定义了 SPH_SIMD_NOCOPY 宏
    #if SPH_SIMD_NOCOPY
        // 调用 STEP_SMALL 函数，参数为 saved[ 0], saved[ 1], saved[ 2], saved[ 3], IF,  4, 13, PP4_2_
        STEP_SMALL(saved[ 0], saved[ 1], saved[ 2], saved[ 3],
            IF,  4, 13, PP4_2_);
        // 调用 STEP_SMALL 函数，参数为 saved[ 4], saved[ 5], saved[ 6], saved[ 7], IF, 13, 10, PP4_0_
        STEP_SMALL(saved[ 4], saved[ 5], saved[ 6], saved[ 7],
            IF, 13, 10, PP4_0_);
        // 调用 STEP_SMALL 函数，参数为 saved[ 8], saved[ 9], saved[10], saved[11], IF, 10, 25, PP4_1_
        STEP_SMALL(saved[ 8], saved[ 9], saved[10], saved[11],
            IF, 10, 25, PP4_1_);
    # 调用 STEP_SMALL 函数，传入参数 saved[12], saved[13], saved[14], saved[15], IF, 25,  4, PP4_2_
    STEP_SMALL(saved[12], saved[13], saved[14], saved[15],
        IF, 25,  4, PP4_2_);
#else
    # 如果不满足条件，则执行以下代码块
    # 调用STEP_SMALL函数，对sc->state数组中的元素进行处理
    STEP_SMALL(sc->state[ 0], sc->state[ 1], sc->state[ 2], sc->state[ 3],
        IF,  4, 13, PP4_2_);
    STEP_SMALL(sc->state[ 4], sc->state[ 5], sc->state[ 6], sc->state[ 7],
        IF, 13, 10, PP4_0_);
    STEP_SMALL(sc->state[ 8], sc->state[ 9], sc->state[10], sc->state[11],
        IF, 10, 25, PP4_1_);
    STEP_SMALL(sc->state[12], sc->state[13], sc->state[14], sc->state[15],
        IF, 25,  4, PP4_2_);
    # 调用WRITE_STATE_SMALL函数，将处理后的结果写入sc
    WRITE_STATE_SMALL(sc);
#endif
}

#if SPH_SIMD_NOCOPY
# 取消之前定义的宏
#undef A0
#undef A1
#undef A2
#undef A3
#undef B0
#undef B1
#undef B2
#undef B3
#undef C0
#undef C1
#undef C2
#undef C3
#undef D0
#undef D1
#undef D2
#undef D3
#endif

#endif

#if SPH_SMALL_FOOTPRINT_SIMD

# 定义宏，将state数组中的元素赋值给对应的变量
#define A0   state[ 0]
#define A1   state[ 1]
#define A2   state[ 2]
#define A3   state[ 3]
#define A4   state[ 4]
#define A5   state[ 5]
#define A6   state[ 6]
#define A7   state[ 7]
#define B0   state[ 8]
#define B1   state[ 9]
#define B2   state[10]
#define B3   state[11]
#define B4   state[12]
#define B5   state[13]
#define B6   state[14]
#define B7   state[15]
#define C0   state[16]
#define C1   state[17]
#define C2   state[18]
#define C3   state[19]
#define C4   state[20]
#define C5   state[21]
#define C6   state[22]
#define C7   state[23]
#define D0   state[24]
#define D1   state[25]
#define D2   state[26]
#define D3   state[27]
#define D4   state[28]
#define D5   state[29]
#define D6   state[30]
#define D7   state[31]

'''
 * Not needed -- already defined for SIMD-224 / SIMD-256
 *
#define STEP2_ELT(n, w, fun, s, ppb)   do { \
        u32 tt = T32(D ## n + (w) + fun(A ## n, B ## n, C ## n)); \
        A ## n = T32(ROL32(tt, s) + tA[(ppb) ^ n]); \
        D ## n = C ## n; \
        C ## n = B ## n; \
        B ## n = tA[n]; \
    } while (0)
 '''


注释：
#define STEP2_BIG(w0, w1, w2, w3, w4, w5, w6, w7, fun, r, s, pp8b)   do { \  # 定义一个宏，用于执行一系列操作
        u32 tA[8]; \  # 声明一个包含8个元素的无符号32位整数数组
        tA[0] = ROL32(A0, r); \  # 对A0进行循环左移r位，并赋值给tA[0]
        tA[1] = ROL32(A1, r); \  # 对A1进行循环左移r位，并赋值给tA[1]
        tA[2] = ROL32(A2, r); \  # 对A2进行循环左移r位，并赋值给tA[2]
        tA[3] = ROL32(A3, r); \  # 对A3进行循环左移r位，并赋值给tA[3]
        tA[4] = ROL32(A4, r); \  # 对A4进行循环左移r位，并赋值给tA[4]
        tA[5] = ROL32(A5, r); \  # 对A5进行循环左移r位，并赋值给tA[5]
        tA[6] = ROL32(A6, r); \  # 对A6进行循环左移r位，并赋值给tA[6]
        tA[7] = ROL32(A7, r); \  # 对A7进行循环左移r位，并赋值给tA[7]
        STEP2_ELT(0, w0, fun, s, pp8b); \  # 调用STEP2_ELT宏，传入参数0, w0, fun, s, pp8b
        STEP2_ELT(1, w1, fun, s, pp8b); \  # 调用STEP2_ELT宏，传入参数1, w1, fun, s, pp8b
        STEP2_ELT(2, w2, fun, s, pp8b); \  # 调用STEP2_ELT宏，传入参数2, w2, fun, s, pp8b
        STEP2_ELT(3, w3, fun, s, pp8b); \  # 调用STEP2_ELT宏，传入参数3, w3, fun, s, pp8b
        STEP2_ELT(4, w4, fun, s, pp8b); \  # 调用STEP2_ELT宏，传入参数4, w4, fun, s, pp8b
        STEP2_ELT(5, w5, fun, s, pp8b); \  # 调用STEP2_ELT宏，传入参数5, w5, fun, s, pp8b
        STEP2_ELT(6, w6, fun, s, pp8b); \  # 调用STEP2_ELT宏，传入参数6, w6, fun, s, pp8b
        STEP2_ELT(7, w7, fun, s, pp8b); \  # 调用STEP2_ELT宏，传入参数7, w7, fun, s, pp8b
    } while (0)  # 宏定义结束，do-while循环结束

static void  # 声明一个静态的无返回值函数
one_round_big(u32 *state, u32 *w, int isp, int p0, int p1, int p2, int p3)  # 函数名为one_round_big，参数为state指针，w指针，isp，p0，p1，p2，p3
{
    static const int pp8k[] = { 1, 6, 2, 3, 5, 7, 4, 1, 6, 2, 3 };  # 声明一个静态的整型数组pp8k，并初始化

    STEP2_BIG(w[ 0], w[ 1], w[ 2], w[ 3], w[ 4], w[ 5], w[ 6], w[ 7],  # 调用STEP2_BIG宏，传入参数w[ 0], w[ 1], w[ 2], w[ 3], w[ 4], w[ 5], w[ 6], w[ 7],
        IF,  p0, p1, pp8k[isp + 0]);  # 传入参数IF,  p0, p1, pp8k[isp + 0]
    STEP2_BIG(w[ 8], w[ 9], w[10], w[11], w[12], w[13], w[14], w[15],  # 调用STEP2_BIG宏，传入参数w[ 8], w[ 9], w[10], w[11], w[12], w[13], w[14], w[15],
        IF,  p1, p2, pp8k[isp + 1]);  # 传入参数IF,  p1, p2, pp8k[isp + 1]
    STEP2_BIG(w[16], w[17], w[18], w[19], w[20], w[21], w[22], w[23],  # 调用STEP2_BIG宏，传入参数w[16], w[17], w[18], w[19], w[20], w[21], w[22], w[23],
        IF,  p2, p3, pp8k[isp + 2]);  # 传入参数IF,  p2, p3, pp8k[isp + 2]
    STEP2_BIG(w[24], w[25], w[26], w[27], w[28], w[29], w[30], w[31],  # 调用STEP2_BIG宏，传入参数w[24], w[25], w[26], w[27], w[28], w[29], w[30], w[31],
        IF,  p3, p0, pp8k[isp + 3]);  # 传入参数IF,  p3, p0, pp8k[isp + 3]
    STEP2_BIG(w[32], w[33], w[34], w[35], w[36], w[37], w[38], w[39],  # 调用STEP2_BIG宏，传入参数w[32], w[33], w[34], w[35], w[36], w[37], w[38], w[39],
        MAJ, p0, p1, pp8k[isp + 4]);  # 传入参数MAJ, p0, p1, pp8k[isp + 4]
    STEP2_BIG(w[40], w[41], w[42], w[43], w[44], w[45], w[46], w[47],  # 调用STEP2_BIG宏，传入参数w[40], w[41], w[42], w[43], w[44], w[45], w[46], w[47],
        MAJ, p1, p2, pp8k[isp + 5]);  # 传入参数MAJ, p1, p2, pp8k[isp + 5]
    STEP2_BIG(w[48], w[49], w[50], w[51], w[52], w[53], w[54], w[55],  # 调用STEP2_BIG宏，传入参数w[48], w[49], w[50], w[51], w[52], w[53], w[54], w[55],
        MAJ, p2, p3, pp8k[isp + 6]);  # 传入参数MAJ, p2, p3, pp8k[isp + 6]
    STEP2_BIG(w[56], w[57], w[58], w[59], w[60], w[61], w[62], w[63],  # 调用STEP2_BIG宏，传入参数w[56], w[57], w[58], w[59], w[60], w[61], w[62], w[63],
        MAJ, p3, p0, pp8k[isp + 7]);  # 传入参数MAJ, p3, p0, pp8k[isp + 7]
}

static void  # 声明一个静态的无返回值函数
compress_big(sph_simd_big_context *sc, int last)  # 函数名为compress_big，参数为sph_simd_big_context指针和整型变量last
{
    unsigned char *x;  # 声明一个无符号字符指针
    s32 q[256];  # 声明一个包含256个元素的s32类型数组
    int i;  # 声明一个整型变量i
    u32 w[64];  # 声明一个包含64个元素的无符号32位整数数组
    u32 state[32];  # 声明一个包含32个元素的无符号32位整数数组
    size_t u;  # 声明一个size_t类型变量u
    # 定义一个静态常量数组，包含32个元素
    # 每个元素都是通过位移操作得到的值
    static const size_t wbp[32] = {
         4 << 4,  6 << 4,  0 << 4,  2 << 4,
         7 << 4,  5 << 4,  3 << 4,  1 << 4,
        15 << 4, 11 << 4, 12 << 4,  8 << 4,
         9 << 4, 13 << 4, 10 << 4, 14 << 4,
        17 << 4, 18 << 4, 23 << 4, 20 << 4,
        22 << 4, 21 << 4, 16 << 4, 19 << 4,
        30 << 4, 24 << 4, 25 << 4, 31 << 4,
        27 << 4, 29 << 4, 28 << 4, 26 << 4
    };
    
    # 将指针 x 指向 sc->buf
    x = sc->buf;
    # 调用 FFT256 函数，传入参数 0, 1, 0, ll
    FFT256(0, 1, 0, ll);
    
    # 如果 last 为真
    if (last) {
        # 遍历 0 到 255
        for (i = 0; i < 256; i ++) {
            # 定义一个变量 tq
            s32 tq;
    
            # tq 等于 q[i] 加上 yoff_b_f[i]
            tq = q[i] + yoff_b_f[i];
            # 对 tq 进行 REDS2 操作
            tq = REDS2(tq);
            # 对 tq 进行 REDS1 操作
            tq = REDS1(tq);
            # 对 tq 进行 REDS1 操作
            tq = REDS1(tq);
            # 如果 tq 小于等于 128，则 q[i] 等于 tq，否则 q[i] 等于 tq 减去 257
            q[i] = (tq <= 128 ? tq : tq - 257);
        }
    } else {
        # 遍历 0 到 255
        for (i = 0; i < 256; i ++) {
            # 定义一个变量 tq
            s32 tq;
    
            # tq 等于 q[i] 加上 yoff_b_n[i]
            tq = q[i] + yoff_b_n[i];
            # 对 tq 进行 REDS2 操作
            tq = REDS2(tq);
            # 对 tq 进行 REDS1 操作
            tq = REDS1(tq);
            # 对 tq 进行 REDS1 操作
            tq = REDS1(tq);
            # 如果 tq 小于等于 128，则 q[i] 等于 tq，否则 q[i] 等于 tq 减去 257
            q[i] = (tq <= 128 ? tq : tq - 257);
        }
    }
    
    # 遍历 0 到 24，每次增加 8
    for (i = 0; i < 32; i += 8) {
        # 将 state[i + 0] 等于 sc->state[i + 0] 异或上 x + 4 * (i + 0) 处的值
        state[i + 0] = sc->state[i + 0]
            ^ sph_dec32le_aligned(x + 4 * (i + 0));
        # 将 state[i + 1] 等于 sc->state[i + 1] 异或上 x + 4 * (i + 1) 处的值
        state[i + 1] = sc->state[i + 1]
            ^ sph_dec32le_aligned(x + 4 * (i + 1));
        # 将 state[i + 2] 等于 sc->state[i + 2] 异或上 x + 4 * (i + 2) 处的值
        state[i + 2] = sc->state[i + 2]
            ^ sph_dec32le_aligned(x + 4 * (i + 2));
        # 将 state[i + 3] 等于 sc->state[i + 3] 异或上 x + 4 * (i + 3) 处的值
        state[i + 3] = sc->state[i + 3]
            ^ sph_dec32le_aligned(x + 4 * (i + 3));
        # 将 state[i + 4] 等于 sc->state[i + 4] 异或上 x + 4 * (i + 4) 处的值
        state[i + 4] = sc->state[i + 4]
            ^ sph_dec32le_aligned(x + 4 * (i + 4));
        # 将 state[i + 5] 等于 sc->state[i + 5] 异或上 x + 4 * (i + 5) 处的值
        state[i + 5] = sc->state[i + 5]
            ^ sph_dec32le_aligned(x + 4 * (i + 5));
        # 将 state[i + 6] 等于 sc->state[i + 6] 异或上 x + 4 * (i + 6) 处的值
        state[i + 6] = sc->state[i + 6]
            ^ sph_dec32le_aligned(x + 4 * (i + 6));
        # 将 state[i + 7] 等于 sc->state[i + 7] 异或上 x + 4 * (i + 7) 处的值
        state[i + 7] = sc->state[i + 7]
            ^ sph_dec32le_aligned(x + 4 * (i + 7));
    }
#define WBREAD(sb, o1, o2, mm)   do { \  # 定义宏，用于读取数据块
        for (u = 0; u < 64; u += 8) { \  # 循环遍历数据块
            size_t v = wbp[(u >> 3) + (sb)]; \  # 计算索引值
            w[u + 0] = INNER(q[v + 2 * 0 + (o1)], \  # 读取数据块中的值
                q[v + 2 * 0 + (o2)], mm); \  # 读取数据块中的值
            w[u + 1] = INNER(q[v + 2 * 1 + (o1)], \  # 读取数据块中的值
                q[v + 2 * 1 + (o2)], mm); \  # 读取数据块中的值
            w[u + 2] = INNER(q[v + 2 * 2 + (o1)], \  # 读取数据块中的值
                q[v + 2 * 2 + (o2)], mm); \  # 读取数据块中的值
            w[u + 3] = INNER(q[v + 2 * 3 + (o1)], \  # 读取数据块中的值
                q[v + 2 * 3 + (o2)], mm); \  # 读取数据块中的值
            w[u + 4] = INNER(q[v + 2 * 4 + (o1)], \  # 读取数据块中的值
                q[v + 2 * 4 + (o2)], mm); \  # 读取数据块中的值
            w[u + 5] = INNER(q[v + 2 * 5 + (o1)], \  # 读取数据块中的值
                q[v + 2 * 5 + (o2)], mm); \  # 读取数据块中的值
            w[u + 6] = INNER(q[v + 2 * 6 + (o1)], \  # 读取数据块中的值
                q[v + 2 * 6 + (o2)], mm); \  # 读取数据块中的值
            w[u + 7] = INNER(q[v + 2 * 7 + (o1)], \  # 读取数据块中的值
                q[v + 2 * 7 + (o2)], mm); \  # 读取数据块中的值
        } \  # 结束循环
    } while (0)  # 结束宏定义

    WBREAD( 0,    0,    1, 185);  # 调用宏，读取数据块
    one_round_big(state, w, 0,  3, 23, 17, 27);  # 调用函数，处理数据块
    WBREAD( 8,    0,    1, 185);  # 调用宏，读取数据块
    one_round_big(state, w, 1, 28, 19, 22,  7);  # 调用函数，处理数据块
    WBREAD(16, -256, -128, 233);  # 调用宏，读取数据块
    one_round_big(state, w, 2, 29,  9, 15,  5);  # 调用函数，处理数据块
    WBREAD(24, -383, -255, 233);  # 调用宏，读取数据块
    one_round_big(state, w, 3,  4, 13, 10, 25);  # 调用函数，处理数据块

#undef WBREAD  # 取消宏定义

    STEP_BIG(  # 调用函数，处理数据块
        sc->state[ 0], sc->state[ 1], sc->state[ 2], sc->state[ 3],
        sc->state[ 4], sc->state[ 5], sc->state[ 6], sc->state[ 7],
        IF,  4, 13, PP8_4_);
    STEP_BIG(  # 调用函数，处理数据块
        sc->state[ 8], sc->state[ 9], sc->state[10], sc->state[11],
        sc->state[12], sc->state[13], sc->state[14], sc->state[15],
        IF, 13, 10, PP8_5_);
    STEP_BIG(  # 调用函数，处理数据块
        sc->state[16], sc->state[17], sc->state[18], sc->state[19],
        sc->state[20], sc->state[21], sc->state[22], sc->state[23],
        IF, 10, 25, PP8_6_);
    STEP_BIG(  # 调用函数，处理数据块
        sc->state[24], sc->state[25], sc->state[26], sc->state[27],
        sc->state[28], sc->state[29], sc->state[30], sc->state[31],
        IF, 25,  4, PP8_0_);
    # 使用 memcpy 函数将 state 数组的内容复制到 sc->state 数组中
    memcpy(sc->state, state, sizeof state);
#else

#if SPH_SIMD_NOCOPY
#define A0   (sc->state[ 0])
#define A1   (sc->state[ 1])
#define A2   (sc->state[ 2])
#define A3   (sc->state[ 3])
#define A4   (sc->state[ 4])
#define A5   (sc->state[ 5])
#define A6   (sc->state[ 6])
#define A7   (sc->state[ 7])
#define B0   (sc->state[ 8])
#define B1   (sc->state[ 9])
#define B2   (sc->state[10])
#define B3   (sc->state[11])
#define B4   (sc->state[12])
#define B5   (sc->state[13])
#define B6   (sc->state[14])
#define B7   (sc->state[15])
#define C0   (sc->state[16])
#define C1   (sc->state[17])
#define C2   (sc->state[18])
#define C3   (sc->state[19])
#define C4   (sc->state[20])
#define C5   (sc->state[21])
#define C6   (sc->state[22])
#define C7   (sc->state[23])
#define D0   (sc->state[24])
#define D1   (sc->state[25])
#define D2   (sc->state[26])
#define D3   (sc->state[27])
#define D4   (sc->state[28])
#define D5   (sc->state[29])
#define D6   (sc->state[30])
#define D7   (sc->state[31])
#endif

这部分代码定义了一系列的宏，用于简化后续代码中对于`sc->state`数组元素的引用。


static void
compress_big(sph_simd_big_context *sc, int last)

这是一个静态函数，函数名为`compress_big`，接受两个参数：一个指向`sph_simd_big_context`类型的指针`sc`和一个整数`last`。


unsigned char *x;
s32 q[256];
int i;
DECL_STATE_BIG
#if SPH_SIMD_NOCOPY
sph_u32 saved[32];
#endif

这部分代码定义了一些变量：指向`unsigned char`类型的指针`x`，一个包含256个`s32`类型元素的数组`q`，一个整数`i`，以及一个宏`DECL_STATE_BIG`。如果定义了宏`SPH_SIMD_NOCOPY`，还会定义一个包含32个`sph_u32`类型元素的数组`saved`。


#if SPH_SIMD_NOCOPY
memcpy(saved, sc->state, sizeof saved);
#endif

如果定义了宏`SPH_SIMD_NOCOPY`，则将`sc->state`数组的内容复制到`saved`数组中。


x = sc->buf;
FFT256(0, 1, 0, ll);

将`sc->buf`的值赋给`x`，然后调用`FFT256`函数，传入参数`0, 1, 0, ll`。


if (last) {
    for (i = 0; i < 256; i ++) {
        s32 tq;

        tq = q[i] + yoff_b_f[i];
        tq = REDS2(tq);
        tq = REDS1(tq);
        tq = REDS1(tq);
        q[i] = (tq <= 128 ? tq : tq - 257);
    }

如果`last`为真，则进入循环，循环变量`i`从0到255。在循环中，将`q[i]`与`yoff_b_f[i]`相加，然后依次经过`REDS2`和两次`REDS1`函数的处理，最后将结果赋给`tq`。然后判断`tq`是否小于等于128，如果是，则将`tq`赋给`q[i]`，否则将`tq - 257`赋给`q[i]`。
    } else {
        // 如果不是第一轮，则进行以下操作
        for (i = 0; i < 256; i ++) {
            // 定义变量 tq
            s32 tq;
            // 计算 q[i] + yoff_b_n[i] 的值
            tq = q[i] + yoff_b_n[i];
            // 对 tq 进行 REDS2 操作
            tq = REDS2(tq);
            // 对 tq 进行 REDS1 操作
            tq = REDS1(tq);
            // 对 tq 进行 REDS1 操作
            tq = REDS1(tq);
            // 如果 tq <= 128，则将 tq 赋值给 q[i]，否则将 tq - 257 赋值给 q[i]
            q[i] = (tq <= 128 ? tq : tq - 257);
        }
    }
    // 读取大端模式的状态
    READ_STATE_BIG(sc);
    // 对 A0 到 D7 进行异或操作
    A0 ^= sph_dec32le_aligned(x +   0);
    A1 ^= sph_dec32le_aligned(x +   4);
    A2 ^= sph_dec32le_aligned(x +   8);
    A3 ^= sph_dec32le_aligned(x +  12);
    A4 ^= sph_dec32le_aligned(x +  16);
    A5 ^= sph_dec32le_aligned(x +  20);
    A6 ^= sph_dec32le_aligned(x +  24);
    A7 ^= sph_dec32le_aligned(x +  28);
    B0 ^= sph_dec32le_aligned(x +  32);
    B1 ^= sph_dec32le_aligned(x +  36);
    B2 ^= sph_dec32le_aligned(x +  40);
    B3 ^= sph_dec32le_aligned(x +  44);
    B4 ^= sph_dec32le_aligned(x +  48);
    B5 ^= sph_dec32le_aligned(x +  52);
    B6 ^= sph_dec32le_aligned(x +  56);
    B7 ^= sph_dec32le_aligned(x +  60);
    C0 ^= sph_dec32le_aligned(x +  64);
    C1 ^= sph_dec32le_aligned(x +  68);
    C2 ^= sph_dec32le_aligned(x +  72);
    C3 ^= sph_dec32le_aligned(x +  76);
    C4 ^= sph_dec32le_aligned(x +  80);
    C5 ^= sph_dec32le_aligned(x +  84);
    C6 ^= sph_dec32le_aligned(x +  88);
    C7 ^= sph_dec32le_aligned(x +  92);
    D0 ^= sph_dec32le_aligned(x +  96);
    D1 ^= sph_dec32le_aligned(x + 100);
    D2 ^= sph_dec32le_aligned(x + 104);
    D3 ^= sph_dec32le_aligned(x + 108);
    D4 ^= sph_dec32le_aligned(x + 112);
    D5 ^= sph_dec32le_aligned(x + 116);
    D6 ^= sph_dec32le_aligned(x + 120);
    D7 ^= sph_dec32le_aligned(x + 124);

    // 进行一轮大端模式的操作
    ONE_ROUND_BIG(0_, 0,  3, 23, 17, 27);
    ONE_ROUND_BIG(1_, 1, 28, 19, 22,  7);
    ONE_ROUND_BIG(2_, 2, 29,  9, 15,  5);
    ONE_ROUND_BIG(3_, 3,  4, 13, 10, 25);
#if SPH_SIMD_NOCOPY
    # 如果定义了 SPH_SIMD_NOCOPY，则执行以下代码块
    STEP_BIG(
        saved[ 0], saved[ 1], saved[ 2], saved[ 3],
        saved[ 4], saved[ 5], saved[ 6], saved[ 7],
        IF,  4, 13, PP8_4_);
    # 调用 STEP_BIG 函数，传入参数，执行一系列操作
    STEP_BIG(
        saved[ 8], saved[ 9], saved[10], saved[11],
        saved[12], saved[13], saved[14], saved[15],
        IF, 13, 10, PP8_5_);
    # 调用 STEP_BIG 函数，传入参数，执行一系列操作
    STEP_BIG(
        saved[16], saved[17], saved[18], saved[19],
        saved[20], saved[21], saved[22], saved[23],
        IF, 10, 25, PP8_6_);
    # 调用 STEP_BIG 函数，传入参数，执行一系列操作
    STEP_BIG(
        saved[24], saved[25], saved[26], saved[27],
        saved[28], saved[29], saved[30], saved[31],
        IF, 25,  4, PP8_0_);
    # 调用 STEP_BIG 函数，传入参数，执行一系列操作
#else
    # 如果未定义 SPH_SIMD_NOCOPY，则执行以下代码块
    STEP_BIG(
        sc->state[ 0], sc->state[ 1], sc->state[ 2], sc->state[ 3],
        sc->state[ 4], sc->state[ 5], sc->state[ 6], sc->state[ 7],
        IF,  4, 13, PP8_4_);
    # 调用 STEP_BIG 函数，传入参数，执行一系列操作
    STEP_BIG(
        sc->state[ 8], sc->state[ 9], sc->state[10], sc->state[11],
        sc->state[12], sc->state[13], sc->state[14], sc->state[15],
        IF, 13, 10, PP8_5_);
    # 调用 STEP_BIG 函数，传入参数，执行一系列操作
    STEP_BIG(
        sc->state[16], sc->state[17], sc->state[18], sc->state[19],
        sc->state[20], sc->state[21], sc->state[22], sc->state[23],
        IF, 10, 25, PP8_6_);
    # 调用 STEP_BIG 函数，传入参数，执行一系列操作
    STEP_BIG(
        sc->state[24], sc->state[25], sc->state[26], sc->state[27],
        sc->state[28], sc->state[29], sc->state[30], sc->state[31],
        IF, 25,  4, PP8_0_);
    # 调用 STEP_BIG 函数，传入参数，执行一系列操作
    WRITE_STATE_BIG(sc);
    # 调用 WRITE_STATE_BIG 函数，传入参数 sc，执行一系列操作
#endif
}
# 结束条件编译指令

#if SPH_SIMD_NOCOPY
# 取消定义 SPH_SIMD_NOCOPY
#undef A0
# 取消定义 A0
#undef A1
# 取消定义 A1
#undef A2
# 取消定义 A2
#undef A3
# 取消定义 A3
#undef A4
# 取消定义 A4
#undef A5
# 取消定义 A5
#undef A6
# 取消定义 A6
#undef A7
# 取消定义 A7
#undef B0
# 取消定义 B0
#undef B1
# 取消定义 B1
#undef B2
# 取消定义 B2
#undef B3
# 取消定义 B3
#undef B4
# 取消定义 B4
#undef B5
# 取消定义 B5
#undef B6
# 取消定义 B6
#undef B7
# 取消定义 B7
#undef C0
# 取消定义 C0
#undef C1
# 取消定义 C1
#undef C2
# 取消定义 C2
#undef C3
# 取消定义 C3
#undef C4
# 取消定义 C4
#undef C5
# 取消定义 C5
#undef C6
# 取消定义 C6
#undef C7
# 取消定义 C7
#undef D0
# 取消定义 D0
#undef D1
# 取消定义 D1
#undef D2
# 取消定义 D2
#undef D3
# 取消定义 D3
#undef D4
# 取消定义 D4
#undef D5
# 取消定义 D5
#undef D6
# 取消定义 D6
#undef D7
# 取消定义 D7
#endif
# 结束条件编译指令

#endif
# 结束条件编译指令

static const u32 IV224[] = {
    # 定义名为 IV224 的常量数组，包含多个 32 位无符号整数
    C32(0x33586E9F), C32(0x12FFF033), C32(0xB2D9F64D), C32(0x6F8FEA53),
    C32(0xDE943106), C32(0x2742E439), C32(0x4FBAB5AC), C32(0x62B9FF96),
    C32(0x22E7B0AF), C32(0xC862B3A8), C32(0x33E00CDC), C32(0x236B86A6),
    # 初始化数组元素的值
    # 定义了四个常量，这些常量是32位无符号整数的十六进制表示形式。
// 定义一个名为 IV256 的静态常量数组，包含32位无符号整数
static const u32 IV256[] = {
    C32(0x4D567983), C32(0x07190BA9), C32(0x8474577B), C32(0x39D726E9),
    C32(0xAAF3D925), C32(0x3EE20B03), C32(0xAFD5E751), C32(0xC96006D3),
    C32(0xC2C2BA14), C32(0x49B3BCB4), C32(0xF67CAF46), C32(0x668626C9),
    C32(0xE2EAA8D2), C32(0x1FF47833), C32(0xD0C661A5), C32(0x55693DE1)
};

// 定义一个名为 IV384 的静态常量数组，包含32位无符号整数
static const u32 IV384[] = {
    C32(0x8A36EEBC), C32(0x94A3BD90), C32(0xD1537B83), C32(0xB25B070B),
    C32(0xF463F1B5), C32(0xB6F81E20), C32(0x0055C339), C32(0xB4D144D1),
    C32(0x7360CA61), C32(0x18361A03), C32(0x17DCB4B9), C32(0x3414C45A),
    C32(0xA699A9D2), C32(0xE39E9664), C32(0x468BFE77), C32(0x51D062F8),
    C32(0xB9E3BFE8), C32(0x63BECE2A), C32(0x8FE506B9), C32(0xF8CC4AC2),
    C32(0x7AE11542), C32(0xB1AADDA1), C32(0x64B06794), C32(0x28D2F462),
    C32(0xE64071EC), C32(0x1DEB91A8), C32(0x8AC8DB23), C32(0x3F782AB5),
    C32(0x039B5CB8), C32(0x71DDD962), C32(0xFADE2CEA), C32(0x1416DF71)
};

// 定义一个名为 IV512 的静态常量数组，包含32位无符号整数
static const u32 IV512[] = {
    C32(0x0BA16B95), C32(0x72F999AD), C32(0x9FECC2AE), C32(0xBA3264FC),
    C32(0x5E894929), C32(0x8E9F30E5), C32(0x2F1DAA37), C32(0xF0F2C558),
    C32(0xAC506643), C32(0xA90635A5), C32(0xE25B878B), C32(0xAAB7878F),
    C32(0x88817F7A), C32(0x0A02892B), C32(0x559A7550), C32(0x598F657E),
    C32(0x7EEF60A1), C32(0x6B70E3E8), C32(0x9C1714D1), C32(0xB958E2A8),
    C32(0xAB02675E), C32(0xED1C014F), C32(0xCD8D65BB), C32(0xFDB7A257),
    C32(0x09254899), C32(0xD699C7BC), C32(0x9019B6DC), C32(0x2B9022E4),
    C32(0x8FA14956), C32(0x21BF9BD3), C32(0xB94D0943), C32(0x6FFDDC22)
};

// 初始化小型 SIMD 上下文
static void
init_small(void *cc, const u32 *iv)
{
    // 将传入的上下文指针转换为小型 SIMD 上下文指针
    sph_simd_small_context *sc;
    sc = cc;
    // 将 IV 数组的内容复制到上下文的状态中
    memcpy(sc->state, iv, sizeof sc->state);
    // 将计数器置零
    sc->count_low = sc->count_high = 0;
    // 将指针置零
    sc->ptr = 0;
}

// 初始化大型 SIMD 上下文
static void
init_big(void *cc, const u32 *iv)
{
    // 将传入的上下文指针转换为大型 SIMD 上下文指针
    sph_simd_big_context *sc;
    sc = cc;
    // 将 IV 数组的内容复制到上下文的状态中
    memcpy(sc->state, iv, sizeof sc->state);
    // 将计数器置零
    sc->count_low = sc->count_high = 0;
    // 将指针置零
    sc->ptr = 0;
}

// 空函数
static void
# 更新小型 SIMD 上下文的数据
update_small(void *cc, const void *data, size_t len)
{
    # 将 void 指针类型转换为 sph_simd_small_context 指针类型
    sph_simd_small_context *sc;

    # 将传入的指针赋值给 sc
    sc = cc;
    # 当数据长度大于 0 时，执行循环
    while (len > 0) {
        size_t clen;

        # 计算剩余空间的长度
        clen = (sizeof sc->buf) - sc->ptr;
        # 如果剩余空间长度大于数据长度，则取数据长度，否则取剩余空间长度
        if (clen > len)
            clen = len;
        # 将数据拷贝到缓冲区中
        memcpy(sc->buf + sc->ptr, data, clen);
        # 更新数据指针和长度
        data = (const unsigned char *)data + clen;
        len -= clen;
        # 如果缓冲区已满，则进行压缩操作，并重置指针
        if ((sc->ptr += clen) == sizeof sc->buf) {
            compress_small(sc, 0);
            sc->ptr = 0;
            sc->count_low = T32(sc->count_low + 1);
            if (sc->count_low == 0)
                sc->count_high ++;
        }
    }
}

# 更新大型 SIMD 上下文的数据
static void
update_big(void *cc, const void *data, size_t len)
{
    # 将 void 指针类型转换为 sph_simd_big_context 指针类型
    sph_simd_big_context *sc;

    # 将传入的指针赋值给 sc
    sc = cc;
    # 当数据长度大于 0 时，执行循环
    while (len > 0) {
        size_t clen;

        # 计算剩余空间的长度
        clen = (sizeof sc->buf) - sc->ptr;
        # 如果剩余空间长度大于数据长度，则取数据长度，否则取剩余空间长度
        if (clen > len)
            clen = len;
        # 将数据拷贝到缓冲区中
        memcpy(sc->buf + sc->ptr, data, clen);
        # 更新数据指针和长度
        data = (const unsigned char *)data + clen;
        len -= clen;
        # 如果缓冲区已满，则进行压缩操作，并重置指针
        if ((sc->ptr += clen) == sizeof sc->buf) {
            compress_big(sc, 0);
            sc->ptr = 0;
            sc->count_low = T32(sc->count_low + 1);
            if (sc->count_low == 0)
                sc->count_high ++;
        }
    }
}

# 对小型 SIMD 上下文的计数进行编码
static void
encode_count_small(unsigned char *dst,
    u32 low, u32 high, size_t ptr, unsigned n)
{
    # 对低位计数进行左移 9 位，并进行模 2^32 运算
    low = T32(low << 9);
    # 对高位计数进行左移 9 位，并加上低位计数右移 23 位的结果，再进行模 2^32 运算
    high = T32(high << 9) + (low >> 23);
    # 加上指针左移 3 位和 n，更新低位计数
    low += (ptr << 3) + n;
    # 将低位计数和高位计数以小端序写入目标数组
    sph_enc32le(dst, low);
    sph_enc32le(dst + 4, high);
}

# 对大型 SIMD 上下文的计数进行编码
static void
encode_count_big(unsigned char *dst,
    u32 low, u32 high, size_t ptr, unsigned n)
{
    # 对低位计数进行左移 10 位，并进行模 2^32 运算
    low = T32(low << 10);
    # 对高位计数进行左移 10 位，并加上低位计数右移 22 位的结果，再进行模 2^32 运算
    high = T32(high << 10) + (low >> 22);
    # 加上指针左移 3 位和 n，更新低位计数
    low += (ptr << 3) + n;
    # 将低位计数和高位计数以小端序写入目标数组
    sph_enc32le(dst, low);
    sph_enc32le(dst + 4, high);
}

# 对小型 SIMD 上下文进行最终处理
static void
finalize_small(void *cc, unsigned ub, unsigned n, void *dst, size_t dst_len)
{
    # 将 void 指针类型转换为 sph_simd_small_context 指针类型
    sph_simd_small_context *sc;
    # 定义无符号字符指针和无符号整型
    unsigned char *d;
    size_t u;

    # 将传入的指针赋值给 sc
    sc = cc;
    # 如果 sc->ptr 大于 0 或者 n 大于 0，则执行以下代码块
    if (sc->ptr > 0 || n > 0) {
        # 将 sc->buf + sc->ptr 之后的 (sizeof sc->buf) - sc->ptr 个字节设置为 0
        memset(sc->buf + sc->ptr, 0, (sizeof sc->buf) - sc->ptr);
        # 将 ub & (0xFF << (8 - n)) 的结果赋值给 sc->buf[sc->ptr]
        sc->buf[sc->ptr] = ub & (0xFF << (8 - n));
        # 调用 compress_small 函数，参数为 sc 和 0
        compress_small(sc, 0);
    }
    # 将 sc->buf 的所有字节设置为 0
    memset(sc->buf, 0, sizeof sc->buf);
    # 调用 encode_count_small 函数，参数为 sc->buf、sc->count_low、sc->count_high、sc->ptr 和 n
    encode_count_small(sc->buf, sc->count_low, sc->count_high, sc->ptr, n);
    # 调用 compress_small 函数，参数为 sc 和 1
    compress_small(sc, 1);
    # 将 dst 赋值给 d
    d = dst;
    # 循环遍历 dst_len 次，每次执行以下代码块
    for (d = dst, u = 0; u < dst_len; u ++)
        # 将 sc->state[u] 的值以小端序写入 d + (u << 2) 处
        sph_enc32le(d + (u << 2), sc->state[u]);
# 定义一个静态函数，用于处理大端模式的哈希计算
static void
finalize_big(void *cc, unsigned ub, unsigned n, void *dst, size_t dst_len)
{
    sph_simd_big_context *sc;  # 定义一个指向大端模式哈希计算上下文的指针
    unsigned char *d;  # 定义一个指向无符号字符的指针
    size_t u;  # 定义一个无符号整数

    sc = cc;  # 将传入的上下文赋值给指针
    if (sc->ptr > 0 || n > 0) {  # 如果指针指向的位置大于0，或者n大于0
        memset(sc->buf + sc->ptr, 0, (sizeof sc->buf) - sc->ptr);  # 将指针指向的位置开始，后面的空间填充为0
        sc->buf[sc->ptr] = ub & (0xFF << (8 - n));  # 将指针指向的位置的值设置为ub和(0xFF << (8 - n))的与运算结果
        compress_big(sc, 0);  # 调用compress_big函数
    }
    memset(sc->buf, 0, sizeof sc->buf);  # 将指针指向的位置开始，后面的空间填充为0
    encode_count_big(sc->buf, sc->count_low, sc->count_high, sc->ptr, n);  # 调用encode_count_big函数
    compress_big(sc, 1);  # 调用compress_big函数
    d = dst;  # 将目标指针赋值给d
    for (d = dst, u = 0; u < dst_len; u ++)  # 循环，从目标指针开始，长度为dst_len
        sph_enc32le(d + (u << 2), sc->state[u]);  # 调用sph_enc32le函数
}

# 初始化一个SIMD-224哈希计算上下文
void
sph_simd224_init(void *cc)
{
    init_small(cc, IV224);  # 调用init_small函数，传入IV224参数
}

# 处理SIMD-224哈希计算
void
sph_simd224(void *cc, const void *data, size_t len)
{
    update_small(cc, data, len);  # 调用update_small函数，传入data和len参数
}

# 关闭SIMD-224哈希计算并输出结果
void
sph_simd224_close(void *cc, void *dst)
{
    sph_simd224_addbits_and_close(cc, 0, 0, dst);  # 调用sph_simd224_addbits_and_close函数，传入0, 0, dst参数
}

# 处理SIMD-224哈希计算并添加额外的比特，并关闭并输出结果
void
sph_simd224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    finalize_small(cc, ub, n, dst, 7);  # 调用finalize_small函数，传入ub, n, dst, 7参数
    sph_simd224_init(cc);  # 调用sph_simd224_init函数，传入cc参数
}

# 初始化一个SIMD-256哈希计算上下文
void
sph_simd256_init(void *cc)
{
    init_small(cc, IV256);  # 调用init_small函数，传入IV256参数
}

# 处理SIMD-256哈希计算
void
sph_simd256(void *cc, const void *data, size_t len)
{
    update_small(cc, data, len);  # 调用update_small函数，传入data和len参数
}

# 关闭SIMD-256哈希计算并输出结果
void
sph_simd256_close(void *cc, void *dst)
{
    sph_simd256_addbits_and_close(cc, 0, 0, dst);  # 调用sph_simd256_addbits_and_close函数，传入0, 0, dst参数
}

# 处理SIMD-256哈希计算并添加额外的比特，并关闭并输出结果
void
sph_simd256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    finalize_small(cc, ub, n, dst, 8);  # 调用finalize_small函数，传入ub, n, dst, 8参数
    sph_simd256_init(cc);  # 调用sph_simd256_init函数，传入cc参数
}

# 初始化一个SIMD-384哈希计算上下文
void
sph_simd384_init(void *cc)
{
    init_big(cc, IV384);  # 调用init_big函数，传入IV384参数
}

# 处理SIMD-384哈希计算
void
sph_simd384(void *cc, const void *data, size_t len)
{
    update_big(cc, data, len);  # 调用update_big函数，传入data和len参数
}

# 关闭SIMD-384哈希计算并输出结果
void
sph_simd384_close(void *cc, void *dst)
{
    sph_simd384_addbits_and_close(cc, 0, 0, dst);  # 调用sph_simd384_addbits_and_close函数，传入0, 0, dst参数
}

# 处理SIMD-384哈希计算并添加额外的比特，并关闭并输出结果
void
sph_simd384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    finalize_big(cc, ub, n, dst, 12);  # 调用finalize_big函数，传入ub, n, dst, 12参数
    sph_simd384_init(cc);  # 调用sph_simd384_init函数，传入cc参数
}

# 初始化一个SIMD-512哈希计算上下文
void
sph_simd512_init(void *cc)
{
    init_big(cc, IV512);  # 调用init_big函数，传入IV512参数
}

# 处理SIMD-512哈希计算
void
sph_simd512(void *cc, const void *data, size_t len)
{
    update_big(cc, data, len);  # 调用update_big函数，传入data和len参数
}
# 关闭 SIMD512 算法的上下文，并将结果写入目标内存
sph_simd512_close(void *cc, void *dst)
{
    # 调用函数将剩余的位数和字节数添加到结果中，并关闭上下文
    sph_simd512_addbits_and_close(cc, 0, 0, dst);
}

# 将剩余的位数和字节数添加到结果中，并关闭上下文
void
sph_simd512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    # 调用函数将剩余的位数和字节数添加到结果中，并关闭上下文
    finalize_big(cc, ub, n, dst, 16);
    # 重新初始化 SIMD512 算法的上下文
    sph_simd512_init(cc);
}
#ifdef __cplusplus
}
#endif
```