# `xmrig\src\crypto\ghostrider\sph_bmw.c`

```cpp
/* $Id: bmw.c 227 2010-06-16 17:28:38Z tp $ */
/* 
 * BMW implementation.
 * 
 * ==========================(LICENSE BEGIN)============================
 * 
 * Copyright (c) 2007-2010  Projet RNRT SAPHIR
 * 
 * Permission is hereby granted, free of charge, to any person obtaining
 * a copy of this software and associated documentation files (the
 * "Software"), to deal in the Software without restriction, including
 * without limitation the rights to use, copy, modify, merge, publish,
 * distribute, sublicense, and/or sell copies of the Software, and to
 * permit persons to whom the Software is furnished to do so, subject to
 * the following conditions:
 * 
 * The above copyright notice and this permission notice shall be
 * included in all copies or substantial portions of the Software.
 * 
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 * IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
 * CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
 * TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
 * SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
 * 
 * ===========================(LICENSE END)=============================
 * 
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#include <stddef.h>
#include <string.h>
#include <limits.h>

#ifdef __cplusplus
extern "C"{
#endif

#include "sph_bmw.h"

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_BMW
#define SPH_SMALL_FOOTPRINT_BMW   1
#endif

#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif

#if !defined(__AVX2__)

static const sph_u32 IV224[] = {
    SPH_C32(0x00010203), SPH_C32(0x04050607),
    SPH_C32(0x08090A0B), SPH_C32(0x0C0D0E0F),
    SPH_C32(0x10111213), SPH_C32(0x14151617),
    SPH_C32(0x18191A1B), SPH_C32(0x1C1D1E1F),
    SPH_C32(0x20212223), SPH_C32(0x24252627),
    # 使用 SPH_C32 函数对十六进制数进行转换
    SPH_C32(0x28292A2B), SPH_C32(0x2C2D2E2F),
    SPH_C32(0x30313233), SPH_C32(0x34353637),
    SPH_C32(0x38393A3B), SPH_C32(0x3C3D3E3F)
// 定义静态常量数组 IV256，包含 16 个 32 位无符号整数
static const sph_u32 IV256[] = {
    SPH_C32(0x40414243), SPH_C32(0x44454647),
    SPH_C32(0x48494A4B), SPH_C32(0x4C4D4E4F),
    SPH_C32(0x50515253), SPH_C32(0x54555657),
    SPH_C32(0x58595A5B), SPH_C32(0x5C5D5E5F),
    SPH_C32(0x60616263), SPH_C32(0x64656667),
    SPH_C32(0x68696A6B), SPH_C32(0x6C6D6E6F),
    SPH_C32(0x70717273), SPH_C32(0x74757677),
    SPH_C32(0x78797A7B), SPH_C32(0x7C7D7E7F)
};

// 如果未定义 AVX2，则定义静态常量数组 IV384 和 IV512
#ifndef AVX2
#if SPH_64

// 定义 IV384，包含 16 个 64 位无符号整数
static const sph_u64 IV384[] = {
    SPH_C64(0x0001020304050607), SPH_C64(0x08090A0B0C0D0E0F),
    SPH_C64(0x1011121314151617), SPH_C64(0x18191A1B1C1D1E1F),
    SPH_C64(0x2021222324252627), SPH_C64(0x28292A2B2C2D2E2F),
    SPH_C64(0x3031323334353637), SPH_C64(0x38393A3B3C3D3E3F),
    SPH_C64(0x4041424344454647), SPH_C64(0x48494A4B4C4D4E4F),
    SPH_C64(0x5051525354555657), SPH_C64(0x58595A5B5C5D5E5F),
    SPH_C64(0x6061626364656667), SPH_C64(0x68696A6B6C6D6E6F),
    SPH_C64(0x7071727374757677), SPH_C64(0x78797A7B7C7D7E7F)
};

// 定义 IV512，包含 16 个 64 位无符号整数
static const sph_u64 IV512[] = {
    SPH_C64(0x8081828384858687), SPH_C64(0x88898A8B8C8D8E8F),
    SPH_C64(0x9091929394959697), SPH_C64(0x98999A9B9C9D9E9F),
    SPH_C64(0xA0A1A2A3A4A5A6A7), SPH_C64(0xA8A9AAABACADAEAF),
    SPH_C64(0xB0B1B2B3B4B5B6B7), SPH_C64(0xB8B9BABBBCBDBEBF),
    SPH_C64(0xC0C1C2C3C4C5C6C7), SPH_C64(0xC8C9CACBCCCDCECF),
    SPH_C64(0xD0D1D2D3D4D5D6D7), SPH_C64(0xD8D9DADBDCDDDEDF),
    SPH_C64(0xE0E1E2E3E4E5E6E7), SPH_C64(0xE8E9EAEBECEDEEEF),
    SPH_C64(0xF0F1F2F3F4F5F6F7), SPH_C64(0xF8F9FAFBFCFDFEFF)
};

#endif
#endif

// 定义宏 XCAT 和 XCAT_，用于连接两个标识符
#define XCAT(x, y)    XCAT_(x, y)
#define XCAT_(x, y)   x ## y

// 定义宏 LPAR，表示左括号
#define LPAR   (

// 定义一系列 16 位整数的宏，用于后续的操作
#define I16_16    0,  1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15
#define I16_17    1,  2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15, 16
#define I16_18    2,  3,  4,  5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15, 16, 17
#define I16_19    3,  4,  5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15, 16, 17, 18
#define I16_20    4,  5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19
#define I16_21    5,  6,  7,  8,  9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20
#define I16_22    6,  7,  8,  9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21
#define I16_23    7,  8,  9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22
#define I16_24    8,  9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23
#define I16_25    9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24
#define I16_26   10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25
#define I16_27   11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26
#define I16_28   12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27
#define I16_29   13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28
#define I16_30   14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29
#define I16_31   15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 30

#define M16_16    0,  1,  3,  4,  7, 10, 11
#define M16_17    1,  2,  4,  5,  8, 11, 12
#define M16_18    2,  3,  5,  6,  9, 12, 13
#define M16_19    3,  4,  6,  7, 10, 13, 14
#define M16_20    4,  5,  7,  8, 11, 14, 15
#define M16_21    5,  6,  8,  9, 12, 15, 16
#define M16_22    6,  7,  9, 10, 13,  0,  1
#define M16_23    7,  8, 10, 11, 14,  1,  2
#define M16_24    8,  9, 11, 12, 15,  2,  3
#define M16_25    9, 10, 12, 13,  0,  3,  4
#define M16_26   10, 11, 13, 14,  1,  4,  5
#define M16_27   11, 12, 14, 15,  2,  5,  6
#define M16_28   12, 13, 15, 16,  3,  6,  7
#define M16_29   13, 14,  0,  1,  4,  7,  8
#define M16_30   14, 15,  1,  2,  5,  8,  9
#define M16_31   15, 16,  2,  3,  6,  9, 10

#if !defined(__AVX2__)

#define ss0(x)    (((x) >> 1) ^ SPH_T32((x) << 3) \  # 定义宏 ss0，对 x 进行位运算
                  ^ SPH_ROTL32(x,  4) ^ SPH_ROTL32(x, 19))
#define ss1(x)    (((x) >> 1) ^ SPH_T32((x) << 2) \  # 定义宏 ss1，对 x 进行位运算
                  ^ SPH_ROTL32(x,  8) ^ SPH_ROTL32(x, 23))
#define ss2(x)    (((x) >> 2) ^ SPH_T32((x) << 1) \  # 定义宏 ss2，对 x 进行位运算
                  ^ SPH_ROTL32(x, 12) ^ SPH_ROTL32(x, 25))
# 定义一个宏，对输入进行位运算和循环移位操作
#define ss3(x)    (((x) >> 2) ^ SPH_T32((x) << 2) \
                  ^ SPH_ROTL32(x, 15) ^ SPH_ROTL32(x, 29))
# 定义一个宏，对输入进行位运算操作
#define ss4(x)    (((x) >> 1) ^ (x))
# 定义一个宏，对输入进行位运算操作
#define ss5(x)    (((x) >> 2) ^ (x))
# 定义一个宏，对输入进行循环左移位操作
#define rs1(x)    SPH_ROTL32(x,  3)
# 定义一个宏，对输入进行循环左移位操作
#define rs2(x)    SPH_ROTL32(x,  7)
# 定义一个宏，对输入进行循环左移位操作
#define rs3(x)    SPH_ROTL32(x, 13)
# 定义一个宏，对输入进行循环左移位操作
#define rs4(x)    SPH_ROTL32(x, 16)
# 定义一个宏，对输入进行循环左移位操作
#define rs5(x)    SPH_ROTL32(x, 19)
# 定义一个宏，对输入进行循环左移位操作
#define rs6(x)    SPH_ROTL32(x, 23)
# 定义一个宏，对输入进行循环左移位操作
#define rs7(x)    SPH_ROTL32(x, 27)

# 定义一个宏，对输入进行位运算和常数乘法操作
#define Ks(j)   SPH_T32((sph_u32)(j) * SPH_C32(0x05555555))

# 定义一个宏，对输入进行一系列位运算和加法操作
#define add_elt_s(mf, hf, j0m, j1m, j3m, j4m, j7m, j10m, j11m, j16) \
    (SPH_T32(SPH_ROTL32(mf(j0m), j1m) + SPH_ROTL32(mf(j3m), j4m) \
        - SPH_ROTL32(mf(j10m), j11m) + Ks(j16)) ^ hf(j7m))

# 定义一个宏，对输入进行一系列位运算和加法操作
#define expand1s_inner(qf, mf, hf, i16, \
        i0, i1, i2, i3, i4, i5, i6, i7, i8, \
        i9, i10, i11, i12, i13, i14, i15, \
        i0m, i1m, i3m, i4m, i7m, i10m, i11m) \
    SPH_T32(ss1(qf(i0)) + ss2(qf(i1)) + ss3(qf(i2)) + ss0(qf(i3)) \
        + ss1(qf(i4)) + ss2(qf(i5)) + ss3(qf(i6)) + ss0(qf(i7)) \
        + ss1(qf(i8)) + ss2(qf(i9)) + ss3(qf(i10)) + ss0(qf(i11)) \
        + ss1(qf(i12)) + ss2(qf(i13)) + ss3(qf(i14)) + ss0(qf(i15)) \
        + add_elt_s(mf, hf, i0m, i1m, i3m, i4m, i7m, i10m, i11m, i16))

# 定义一个宏，对输入进行一系列位运算和加法操作
#define expand1s(qf, mf, hf, i16) \
    expand1s_(qf, mf, hf, i16, I16_ ## i16, M16_ ## i16)
# 定义一个宏，对输入进行一系列位运算和加法操作
#define expand1s_(qf, mf, hf, i16, ix, iy) \
    expand1s_inner LPAR qf, mf, hf, i16, ix, iy)

# 定义一个宏，对输入进行一系列位运算和加法操作
#define expand2s_inner(qf, mf, hf, i16, \
        i0, i1, i2, i3, i4, i5, i6, i7, i8, \
        i9, i10, i11, i12, i13, i14, i15, \
        i0m, i1m, i3m, i4m, i7m, i10m, i11m) \
    SPH_T32(qf(i0) + rs1(qf(i1)) + qf(i2) + rs2(qf(i3)) \
        + qf(i4) + rs3(qf(i5)) + qf(i6) + rs4(qf(i7)) \
        + qf(i8) + rs5(qf(i9)) + qf(i10) + rs6(qf(i11)) \
        + qf(i12) + rs7(qf(i13)) + ss4(qf(i14)) + ss5(qf(i15)) \
        + add_elt_s(mf, hf, i0m, i1m, i3m, i4m, i7m, i10m, i11m, i16))

# 定义一个宏，对输入进行一系列位运算和加法操作
#define expand2s(qf, mf, hf, i16) \
    expand2s_(qf, mf, hf, i16, I16_ ## i16, M16_ ## i16)
#if SPH_64
# 宏定义，根据条件编译是否为64位系统
#define sb0(x)    (((x) >> 1) ^ SPH_T64((x) << 3) \
                  ^ SPH_ROTL64(x,  4) ^ SPH_ROTL64(x, 37))
# 宏定义，对x进行位运算
#define sb1(x)    (((x) >> 1) ^ SPH_T64((x) << 2) \
                  ^ SPH_ROTL64(x, 13) ^ SPH_ROTL64(x, 43))
# 宏定义，对x进行位运算
#define sb2(x)    (((x) >> 2) ^ SPH_T64((x) << 1) \
                  ^ SPH_ROTL64(x, 19) ^ SPH_ROTL64(x, 53))
# 宏定义，对x进行位运算
#define sb3(x)    (((x) >> 2) ^ SPH_T64((x) << 2) \
                  ^ SPH_ROTL64(x, 28) ^ SPH_ROTL64(x, 59))
# 宏定义，对x进行位运算
#define sb4(x)    (((x) >> 1) ^ (x))
# 宏定义，对x进行位运算
#define sb5(x)    (((x) >> 2) ^ (x))
# 宏定义，对x进行位运算
#define rb1(x)    SPH_ROTL64(x,  5)
# 宏定义，对x进行位运算
#define rb2(x)    SPH_ROTL64(x, 11)
# 宏定义，对x进行位运算
#define rb3(x)    SPH_ROTL64(x, 27)
# 宏定义，对x进行位运算
#define rb4(x)    SPH_ROTL64(x, 32)
# 宏定义，对x进行位运算
#define rb5(x)    SPH_ROTL64(x, 37)
# 宏定义，对x进行位运算
#define rb6(x)    SPH_ROTL64(x, 43)
# 宏定义，对x进行位运算
#define rb7(x)    SPH_ROTL64(x, 53)

# 宏定义，根据j计算Kb值
#define Kb(j)   SPH_T64((sph_u64)(j) * SPH_C64(0x0555555555555555))

# 如果为SPH_SMALL_FOOTPRINT_BMW，则定义Kb_tab数组
#if SPH_SMALL_FOOTPRINT_BMW
# 定义Kb_tab数组
static const sph_u64 Kb_tab[] = {
    Kb(16), Kb(17), Kb(18), Kb(19), Kb(20), Kb(21), Kb(22), Kb(23),
    Kb(24), Kb(25), Kb(26), Kb(27), Kb(28), Kb(29), Kb(30), Kb(31)
};
# 宏定义，根据条件编译是否为64位系统，计算mf和hf的偏移值
#define rol_off(mf, j, off) \
    SPH_ROTL64(mf(((j) + (off)) & 15), (((j) + (off)) & 15) + 1)
# 宏定义，根据条件编译是否为64位系统，计算mf和hf的元素值
#define add_elt_b(mf, hf, j) \
    (SPH_T64(rol_off(mf, j, 0) + rol_off(mf, j, 3) \
        - rol_off(mf, j, 10) + Kb_tab[j]) ^ hf(((j) + 7) & 15))
# 宏定义，根据条件编译是否为64位系统，扩展第一轮
#define expand1b(qf, mf, hf, i) \
    SPH_T64(sb1(qf((i) - 16)) + sb2(qf((i) - 15)) \
        + sb3(qf((i) - 14)) + sb0(qf((i) - 13)) \
        + sb1(qf((i) - 12)) + sb2(qf((i) - 11)) \
        + sb3(qf((i) - 10)) + sb0(qf((i) - 9)) \
        + sb1(qf((i) - 8)) + sb2(qf((i) - 7)) \
        + sb3(qf((i) - 6)) + sb0(qf((i) - 5)) \
        + sb1(qf((i) - 4)) + sb2(qf((i) - 3)) \
        + sb3(qf((i) - 2)) + sb0(qf((i) - 1)) \
        + add_elt_b(mf, hf, (i) - 16))
# 宏定义，根据条件编译是否为64位系统，扩展第二轮
#define expand2b(qf, mf, hf, i) \
    # 计算 SPH_T64 函数的参数，参数包括一系列函数的结果和 add_elt_b 函数的结果
    SPH_T64(qf((i) - 16) + rb1(qf((i) - 15)) \
        + qf((i) - 14) + rb2(qf((i) - 13)) \
        + qf((i) - 12) + rb3(qf((i) - 11)) \
        + qf((i) - 10) + rb4(qf((i) - 9)) \
        + qf((i) - 8) + rb5(qf((i) - 7)) \
        + qf((i) - 6) + rb6(qf((i) - 5)) \
        + qf((i) - 4) + rb7(qf((i) - 3)) \
        + sb4(qf((i) - 2)) + sb5(qf((i) - 1)) \
        + add_elt_b(mf, hf, (i) - 16))
#define add_elt_b(mf, hf, j0m, j1m, j3m, j4m, j7m, j10m, j11m, j16) \
    (SPH_T64(SPH_ROTL64(mf(j0m), j1m) + SPH_ROTL64(mf(j3m), j4m) \
        - SPH_ROTL64(mf(j10m), j11m) + Kb(j16)) ^ hf(j7m))
    // 定义一个宏，用于计算并返回加密算法中的某些元素

#define expand1b_inner(qf, mf, hf, i16, \
        i0, i1, i2, i3, i4, i5, i6, i7, i8, \
        i9, i10, i11, i12, i13, i14, i15, \
        i0m, i1m, i3m, i4m, i7m, i10m, i11m) \
    SPH_T64(sb1(qf(i0)) + sb2(qf(i1)) + sb3(qf(i2)) + sb0(qf(i3)) \
        + sb1(qf(i4)) + sb2(qf(i5)) + sb3(qf(i6)) + sb0(qf(i7)) \
        + sb1(qf(i8)) + sb2(qf(i9)) + sb3(qf(i10)) + sb0(qf(i11)) \
        + sb1(qf(i12)) + sb2(qf(i13)) + sb3(qf(i14)) + sb0(qf(i15)) \
        + add_elt_b(mf, hf, i0m, i1m, i3m, i4m, i7m, i10m, i11m, i16))
    // 定义一个宏，用于计算并返回加密算法中的某些元素

#define expand1b(qf, mf, hf, i16) \
    expand1b_(qf, mf, hf, i16, I16_ ## i16, M16_ ## i16)
#define expand1b_(qf, mf, hf, i16, ix, iy) \
    expand1b_inner LPAR qf, mf, hf, i16, ix, iy)
    // 定义一个宏，用于计算并返回加密算法中的某些元素

#define expand2b_inner(qf, mf, hf, i16, \
        i0, i1, i2, i3, i4, i5, i6, i7, i8, \
        i9, i10, i11, i12, i13, i14, i15, \
        i0m, i1m, i3m, i4m, i7m, i10m, i11m) \
    SPH_T64(qf(i0) + rb1(qf(i1)) + qf(i2) + rb2(qf(i3)) \
        + qf(i4) + rb3(qf(i5)) + qf(i6) + rb4(qf(i7)) \
        + qf(i8) + rb5(qf(i9)) + qf(i10) + rb6(qf(i11)) \
        + qf(i12) + rb7(qf(i13)) + sb4(qf(i14)) + sb5(qf(i15)) \
        + add_elt_b(mf, hf, i0m, i1m, i3m, i4m, i7m, i10m, i11m, i16))
    // 定义一个宏，用于计算并返回加密算法中的某些元素

#define expand2b(qf, mf, hf, i16) \
    expand2b_(qf, mf, hf, i16, I16_ ## i16, M16_ ## i16)
#define expand2b_(qf, mf, hf, i16, ix, iy) \
    expand2b_inner LPAR qf, mf, hf, i16, ix, iy)
    // 定义一个宏，用于计算并返回加密算法中的某些元素

#endif

#endif

#define MAKE_W(tt, i0, op01, i1, op12, i2, op23, i3, op34, i4) \
    tt((M(i0) ^ H(i0)) op01 (M(i1) ^ H(i1)) op12 (M(i2) ^ H(i2)) \
    op23 (M(i3) ^ H(i3)) op34 (M(i4) ^ H(i4)))
    // 定义一个宏，用于计算并返回加密算法中的某些元素

#if !defined(__AVX2__)

#define Ws0    MAKE_W(SPH_T32,  5, -,  7, +, 10, +, 13, +, 14)
#define Ws1    MAKE_W(SPH_T32,  6, -,  8, +, 11, +, 14, -, 15)
#define Ws2    MAKE_W(SPH_T32,  0, +,  7, +,  9, -, 12, +, 15)
    // 定义一些常量，用于计算并返回加密算法中的某些元素
#define Ws3    MAKE_W(SPH_T32,  0, -,  1, +,  8, -, 10, +, 13)
#define Ws4    MAKE_W(SPH_T32,  1, +,  2, +,  9, -, 11, -, 14)
#define Ws5    MAKE_W(SPH_T32,  3, -,  2, +, 10, -, 12, +, 15)
#define Ws6    MAKE_W(SPH_T32,  4, -,  0, -,  3, -, 11, +, 13)
#define Ws7    MAKE_W(SPH_T32,  1, -,  4, -,  5, -, 12, -, 14)
#define Ws8    MAKE_W(SPH_T32,  2, -,  5, -,  6, +, 13, -, 15)
#define Ws9    MAKE_W(SPH_T32,  0, -,  3, +,  6, -,  7, +, 14)
#define Ws10   MAKE_W(SPH_T32,  8, -,  1, -,  4, -,  7, +, 15)
#define Ws11   MAKE_W(SPH_T32,  8, -,  0, -,  2, -,  5, +,  9)
#define Ws12   MAKE_W(SPH_T32,  1, +,  3, -,  6, -,  9, +, 10)
#define Ws13   MAKE_W(SPH_T32,  2, +,  4, +,  7, +, 10, +, 11)
#define Ws14   MAKE_W(SPH_T32,  3, -,  5, +,  8, -, 11, -, 12)
#define Ws15   MAKE_W(SPH_T32, 12, -,  4, -,  6, -,  9, +, 13)

#if SPH_SMALL_FOOTPRINT_BMW

#define MAKE_Qas   do { \  # 定义一个宏，用于生成Qas
        unsigned u; \  # 声明一个无符号整数u
        sph_u32 Ws[16]; \  # 声明一个长度为16的sph_u32类型数组Ws
        Ws[ 0] = Ws0; \  # 将Ws0赋值给Ws数组的第一个元素
        Ws[ 1] = Ws1; \  # 将Ws1赋值给Ws数组的第二个元素
        Ws[ 2] = Ws2; \  # 将Ws2赋值给Ws数组的第三个元素
        Ws[ 3] = Ws3; \  # 将Ws3赋值给Ws数组的第四个元素
        Ws[ 4] = Ws4; \  # 将Ws4赋值给Ws数组的第五个元素
        Ws[ 5] = Ws5; \  # 将Ws5赋值给Ws数组的第六个元素
        Ws[ 6] = Ws6; \  # 将Ws6赋值给Ws数组的第七个元素
        Ws[ 7] = Ws7; \  # 将Ws7赋值给Ws数组的第八个元素
        Ws[ 8] = Ws8; \  # 将Ws8赋值给Ws数组的第九个元素
        Ws[ 9] = Ws9; \  # 将Ws9赋值给Ws数组的第十个元素
        Ws[10] = Ws10; \  # 将Ws10赋值给Ws数组的第十一个元素
        Ws[11] = Ws11; \  # 将Ws11赋值给Ws数组的第十二个元素
        Ws[12] = Ws12; \  # 将Ws12赋值给Ws数组的第十三个元素
        Ws[13] = Ws13; \  # 将Ws13赋值给Ws数组的第十四个元素
        Ws[14] = Ws14; \  # 将Ws14赋值给Ws数组的第十五个元素
        Ws[15] = Ws15; \  # 将Ws15赋值给Ws数组的第十六个元素
        for (u = 0; u < 15; u += 5) { \  # 循环，从0开始，每次增加5，直到小于15
            qt[u + 0] = SPH_T32(ss0(Ws[u + 0]) + H(u + 1)); \  # 计算qt数组的第u+0个元素
            qt[u + 1] = SPH_T32(ss1(Ws[u + 1]) + H(u + 2)); \  # 计算qt数组的第u+1个元素
            qt[u + 2] = SPH_T32(ss2(Ws[u + 2]) + H(u + 3)); \  # 计算qt数组的第u+2个元素
            qt[u + 3] = SPH_T32(ss3(Ws[u + 3]) + H(u + 4)); \  # 计算qt数组的第u+3个元素
            qt[u + 4] = SPH_T32(ss4(Ws[u + 4]) + H(u + 5)); \  # 计算qt数组的第u+4个元素
        } \  # 结束循环
        qt[15] = SPH_T32(ss0(Ws[15]) + H(0)); \  # 计算qt数组的第15个元素
    } while (0)  # 结束宏定义
#define MAKE_Qbs   do { \  // 定义宏 MAKE_Qbs，展开为一段代码块
        qt[16] = expand1s(Qs, M, H, 16); \  // 计算 qt[16] 的值
        qt[17] = expand1s(Qs, M, H, 17); \  // 计算 qt[17] 的值
        qt[18] = expand2s(Qs, M, H, 18); \  // 计算 qt[18] 的值
        qt[19] = expand2s(Qs, M, H, 19); \  // 计算 qt[19] 的值
        qt[20] = expand2s(Qs, M, H, 20); \  // 计算 qt[20] 的值
        qt[21] = expand2s(Qs, M, H, 21); \  // 计算 qt[21] 的值
        qt[22] = expand2s(Qs, M, H, 22); \  // 计算 qt[22] 的值
        qt[23] = expand2s(Qs, M, H, 23); \  // 计算 qt[23] 的值
        qt[24] = expand2s(Qs, M, H, 24); \  // 计算 qt[24] 的值
        qt[25] = expand2s(Qs, M, H, 25); \  // 计算 qt[25] 的值
        qt[26] = expand2s(Qs, M, H, 26); \  // 计算 qt[26] 的值
        qt[27] = expand2s(Qs, M, H, 27); \  // 计算 qt[27] 的值
        qt[28] = expand2s(Qs, M, H, 28); \  // 计算 qt[28] 的值
        qt[29] = expand2s(Qs, M, H, 29); \  // 计算 qt[29] 的值
        qt[30] = expand2s(Qs, M, H, 30); \  // 计算 qt[30] 的值
        qt[31] = expand2s(Qs, M, H, 31); \  // 计算 qt[31] 的值
    } while (0)  // 结束宏定义代码块

#else

#define MAKE_Qas   do { \  // 定义宏 MAKE_Qas，展开为一段代码块
        qt[ 0] = SPH_T32(ss0(Ws0 ) + H( 1)); \  // 计算 qt[0] 的值
        qt[ 1] = SPH_T32(ss1(Ws1 ) + H( 2)); \  // 计算 qt[1] 的值
        qt[ 2] = SPH_T32(ss2(Ws2 ) + H( 3)); \  // 计算 qt[2] 的值
        qt[ 3] = SPH_T32(ss3(Ws3 ) + H( 4)); \  // 计算 qt[3] 的值
        qt[ 4] = SPH_T32(ss4(Ws4 ) + H( 5)); \  // 计算 qt[4] 的值
        qt[ 5] = SPH_T32(ss0(Ws5 ) + H( 6)); \  // 计算 qt[5] 的值
        qt[ 6] = SPH_T32(ss1(Ws6 ) + H( 7)); \  // 计算 qt[6] 的值
        qt[ 7] = SPH_T32(ss2(Ws7 ) + H( 8)); \  // 计算 qt[7] 的值
        qt[ 8] = SPH_T32(ss3(Ws8 ) + H( 9)); \  // 计算 qt[8] 的值
        qt[ 9] = SPH_T32(ss4(Ws9 ) + H(10)); \  // 计算 qt[9] 的值
        qt[10] = SPH_T32(ss0(Ws10) + H(11)); \  // 计算 qt[10] 的值
        qt[11] = SPH_T32(ss1(Ws11) + H(12)); \  // 计算 qt[11] 的值
        qt[12] = SPH_T32(ss2(Ws12) + H(13)); \  // 计算 qt[12] 的值
        qt[13] = SPH_T32(ss3(Ws13) + H(14)); \  // 计算 qt[13] 的值
        qt[14] = SPH_T32(ss4(Ws14) + H(15)); \  // 计算 qt[14] 的值
        qt[15] = SPH_T32(ss0(Ws15) + H( 0)); \  // 计算 qt[15] 的值
    } while (0)  // 结束宏定义代码块
// 定义宏 MAKE_Qbs，展开为一系列的 expand2s 函数调用，将结果存入 qt 数组
#define MAKE_Qbs   do { \
        qt[16] = expand1s(Qs, M, H, 16); \
        qt[17] = expand1s(Qs, M, H, 17); \
        qt[18] = expand2s(Qs, M, H, 18); \
        qt[19] = expand2s(Qs, M, H, 19); \
        qt[20] = expand2s(Qs, M, H, 20); \
        qt[21] = expand2s(Qs, M, H, 21); \
        qt[22] = expand2s(Qs, M, H, 22); \
        qt[23] = expand2s(Qs, M, H, 23); \
        qt[24] = expand2s(Qs, M, H, 24); \
        qt[25] = expand2s(Qs, M, H, 25); \
        qt[26] = expand2s(Qs, M, H, 26); \
        qt[27] = expand2s(Qs, M, H, 27); \
        qt[28] = expand2s(Qs, M, H, 28); \
        qt[29] = expand2s(Qs, M, H, 29); \
        qt[30] = expand2s(Qs, M, H, 30); \
        qt[31] = expand2s(Qs, M, H, 31); \
    } while (0)

// 定义宏 MAKE_Qs，展开为一系列的 MAKE_Qas 和 MAKE_Qbs 的调用
#define MAKE_Qs   do { \
        MAKE_Qas; \
        MAKE_Qbs; \
    } while (0)

// 定义宏 Qs(j)，用于访问 qt 数组的特定索引位置的值
#define Qs(j)   (qt[j])

// 定义宏 Wb0 到 Wb15，展开为一系列的 MAKE_W 函数调用，用于生成特定的 W 值
#define Wb0    MAKE_W(SPH_T64,  5, -,  7, +, 10, +, 13, +, 14)
#define Wb1    MAKE_W(SPH_T64,  6, -,  8, +, 11, +, 14, -, 15)
#define Wb2    MAKE_W(SPH_T64,  0, +,  7, +,  9, -, 12, +, 15)
#define Wb3    MAKE_W(SPH_T64,  0, -,  1, +,  8, -, 10, +, 13)
#define Wb4    MAKE_W(SPH_T64,  1, +,  2, +,  9, -, 11, -, 14)
#define Wb5    MAKE_W(SPH_T64,  3, -,  2, +, 10, -, 12, +, 15)
#define Wb6    MAKE_W(SPH_T64,  4, -,  0, -,  3, -, 11, +, 13)
#define Wb7    MAKE_W(SPH_T64,  1, -,  4, -,  5, -, 12, -, 14)
#define Wb8    MAKE_W(SPH_T64,  2, -,  5, -,  6, +, 13, -, 15)
#define Wb9    MAKE_W(SPH_T64,  0, -,  3, +,  6, -,  7, +, 14)
#define Wb10   MAKE_W(SPH_T64,  8, -,  1, -,  4, -,  7, +, 15)
#define Wb11   MAKE_W(SPH_T64,  8, -,  0, -,  2, -,  5, +,  9)
#define Wb12   MAKE_W(SPH_T64,  1, +,  3, -,  6, -,  9, +, 10)
#define Wb13   MAKE_W(SPH_T64,  2, +,  4, +,  7, +, 10, +, 11)
#define Wb14   MAKE_W(SPH_T64,  3, -,  5, +,  8, -, 11, -, 12)
#define Wb15   MAKE_W(SPH_T64, 12, -,  4, -,  6, -,  9, +, 13)
#define MAKE_Qab   do { \  # 定义宏 MAKE_Qab，开始一个 do-while 代码块
        unsigned u; \  # 声明无符号整型变量 u
        sph_u64 Wb[16]; \  # 声明一个包含16个sph_u64类型元素的数组 Wb
        Wb[ 0] = Wb0; \  # 将 Wb0 赋值给 Wb[0]
        Wb[ 1] = Wb1; \  # 将 Wb1 赋值给 Wb[1]
        Wb[ 2] = Wb2; \  # 将 Wb2 赋值给 Wb[2]
        Wb[ 3] = Wb3; \  # 将 Wb3 赋值给 Wb[3]
        Wb[ 4] = Wb4; \  # 将 Wb4 赋值给 Wb[4]
        Wb[ 5] = Wb5; \  # 将 Wb5 赋值给 Wb[5]
        Wb[ 6] = Wb6; \  # 将 Wb6 赋值给 Wb[6]
        Wb[ 7] = Wb7; \  # 将 Wb7 赋值给 Wb[7]
        Wb[ 8] = Wb8; \  # 将 Wb8 赋值给 Wb[8]
        Wb[ 9] = Wb9; \  # 将 Wb9 赋值给 Wb[9]
        Wb[10] = Wb10; \  # 将 Wb10 赋值给 Wb[10]
        Wb[11] = Wb11; \  # 将 Wb11 赋值给 Wb[11]
        Wb[12] = Wb12; \  # 将 Wb12 赋值给 Wb[12]
        Wb[13] = Wb13; \  # 将 Wb13 赋值给 Wb[13]
        Wb[14] = Wb14; \  # 将 Wb14 赋值给 Wb[14]
        Wb[15] = Wb15; \  # 将 Wb15 赋值给 Wb[15]
        for (u = 0; u < 15; u += 5) { \  # 循环，u 从 0 开始，每次增加 5，直到小于 15
            qt[u + 0] = SPH_T64(sb0(Wb[u + 0]) + H(u + 1)); \  # 计算 qt[u+0] 的值
            qt[u + 1] = SPH_T64(sb1(Wb[u + 1]) + H(u + 2)); \  # 计算 qt[u+1] 的值
            qt[u + 2] = SPH_T64(sb2(Wb[u + 2]) + H(u + 3)); \  # 计算 qt[u+2] 的值
            qt[u + 3] = SPH_T64(sb3(Wb[u + 3]) + H(u + 4)); \  # 计算 qt[u+3] 的值
            qt[u + 4] = SPH_T64(sb4(Wb[u + 4]) + H(u + 5)); \  # 计算 qt[u+4] 的值
        } \  # 循环结束
        qt[15] = SPH_T64(sb0(Wb[15]) + H(0)); \  # 计算 qt[15] 的值
    } while (0)  # 结束 do-while 代码块
#define MAKE_Qbb   do { \  # 定义一个宏，用于生成一系列的扩展操作
        qt[16] = expand1b(Qb, M, H, 16); \  # 对 Qb 进行扩展操作，将结果存储到 qt[16] 中
        qt[17] = expand1b(Qb, M, H, 17); \  # 对 Qb 进行扩展操作，将结果存储到 qt[17] 中
        qt[18] = expand2b(Qb, M, H, 18); \  # 对 Qb 进行扩展操作，将结果存储到 qt[18] 中
        qt[19] = expand2b(Qb, M, H, 19); \  # 对 Qb 进行扩展操作，将结果存储到 qt[19] 中
        qt[20] = expand2b(Qb, M, H, 20); \  # 对 Qb 进行扩展操作，将结果存储到 qt[20] 中
        qt[21] = expand2b(Qb, M, H, 21); \  # 对 Qb 进行扩展操作，将结果存储到 qt[21] 中
        qt[22] = expand2b(Qb, M, H, 22); \  # 对 Qb 进行扩展操作，将结果存储到 qt[22] 中
        qt[23] = expand2b(Qb, M, H, 23); \  # 对 Qb 进行扩展操作，将结果存储到 qt[23] 中
        qt[24] = expand2b(Qb, M, H, 24); \  # 对 Qb 进行扩展操作，将结果存储到 qt[24] 中
        qt[25] = expand2b(Qb, M, H, 25); \  # 对 Qb 进行扩展操作，将结果存储到 qt[25] 中
        qt[26] = expand2b(Qb, M, H, 26); \  # 对 Qb 进行扩展操作，将结果存储到 qt[26] 中
        qt[27] = expand2b(Qb, M, H, 27); \  # 对 Qb 进行扩展操作，将结果存储到 qt[27] 中
        qt[28] = expand2b(Qb, M, H, 28); \  # 对 Qb 进行扩展操作，将结果存储到 qt[28] 中
        qt[29] = expand2b(Qb, M, H, 29); \  # 对 Qb 进行扩展操作，将结果存储到 qt[29] 中
        qt[30] = expand2b(Qb, M, H, 30); \  # 对 Qb 进行扩展操作，将结果存储到 qt[30] 中
        qt[31] = expand2b(Qb, M, H, 31); \  # 对 Qb 进行扩展操作，将结果存储到 qt[31] 中
    } while (0)  # 定义宏结束

#endif  # 结束宏定义

#define MAKE_Qb   do { \  # 定义一个宏，用于生成一系列的扩展操作
        MAKE_Qab; \  # 调用另一个宏 MAKE_Qab
        MAKE_Qbb; \  # 调用另一个宏 MAKE_Qbb
    } while (0)  # 定义宏结束

#define Qb(j)   (qt[j])  # 定义一个宏，用于获取 qt 数组中的值

#endif  # 结束宏定义

    } while (0)  # 结束 do-while 循环

#if SPH_64  # 如果定义了 SPH_64

#define FOLDb   FOLD(sph_u64, MAKE_Qb, SPH_T64, SPH_ROTL64, M, Qb, dH)  # 定义一个宏 FOLDb，用于执行 FOLD 操作

#endif  # 结束条件编译

#if !defined(__AVX2__)  # 如果未定义 __AVX2__

#define FOLDs   FOLD(sph_u32, MAKE_Qs, SPH_T32, SPH_ROTL32, M, Qs, dH)  # 定义一个宏 FOLDs，用于执行 FOLD 操作

static void  # 定义一个静态函数
compress_small(const unsigned char *data, const sph_u32 h[16], sph_u32 dh[16])  # 函数参数为 data、h 数组和 dh 数组
{
#if SPH_LITTLE_FAST  # 如果定义了 SPH_LITTLE_FAST
#define M(x)    sph_dec32le_aligned(data + 4 * (x))  # 定义一个宏 M，用于获取 data 数组中的值
#else  # 否则
    sph_u32 mv[16];  # 定义一个长度为 16 的无符号整数数组

    mv[ 0] = sph_dec32le_aligned(data +  0);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 1] = sph_dec32le_aligned(data +  4);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 2] = sph_dec32le_aligned(data +  8);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 3] = sph_dec32le_aligned(data + 12);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 4] = sph_dec32le_aligned(data + 16);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 5] = sph_dec32le_aligned(data + 20);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 6] = sph_dec32le_aligned(data + 24);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 7] = sph_dec32le_aligned(data + 28);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 8] = sph_dec32le_aligned(data + 32);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[ 9] = sph_dec32le_aligned(data + 36);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[10] = sph_dec32le_aligned(data + 40);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[11] = sph_dec32le_aligned(data + 44);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[12] = sph_dec32le_aligned(data + 48);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[13] = sph_dec32le_aligned(data + 52);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[14] = sph_dec32le_aligned(data + 56);  # 获取 data 数组中的值并存储到 mv 数组中
    mv[15] = sph_dec32le_aligned(data + 60);  # 获取 data 数组中的值并存储到 mv 数组中
#define M(x)    (mv[x])  # 定义一个宏 M，用于获取 mv 数组中的值
#endif
#define H(x)    (h[x])  # 定义宏 H(x)，用于访问哈希值数组 h 的元素
#define dH(x)   (dh[x])  # 定义宏 dH(x)，用于访问哈希值数组 dh 的元素

    FOLDs;  # 执行 FOLDs 操作

#undef M  # 取消宏 M 的定义
#undef H  # 取消宏 H 的定义
#undef dH  # 取消宏 dH 的定义
}  # 结束函数或代码块

static const sph_u32 final_s[16] = {  # 定义一个包含 16 个元素的常量数组 final_s
    SPH_C32(0xaaaaaaa0), SPH_C32(0xaaaaaaa1), SPH_C32(0xaaaaaaa2),  # 初始化数组元素
    SPH_C32(0xaaaaaaa3), SPH_C32(0xaaaaaaa4), SPH_C32(0xaaaaaaa5),
    SPH_C32(0xaaaaaaa6), SPH_C32(0xaaaaaaa7), SPH_C32(0xaaaaaaa8),
    SPH_C32(0xaaaaaaa9), SPH_C32(0xaaaaaaaa), SPH_C32(0xaaaaaaab),
    SPH_C32(0xaaaaaaac), SPH_C32(0xaaaaaaad), SPH_C32(0xaaaaaaae),
    SPH_C32(0xaaaaaaaf)
};

static void
bmw32_init(sph_bmw_small_context *sc, const sph_u32 *iv)  # 定义函数 bmw32_init，初始化 bmw32 算法上下文
{
    memcpy(sc->H, iv, sizeof sc->H);  # 将 iv 数组的内容复制到 sc->H 数组中
    sc->ptr = 0;  # 将 sc->ptr 置为 0
#if SPH_64
    sc->bit_count = 0;  # 如果 SPH_64 宏定义为真，则将 sc->bit_count 置为 0
#else
    sc->bit_count_high = 0;  # 如果 SPH_64 宏未定义为真，则将 sc->bit_count_high 置为 0
    sc->bit_count_low = 0;  # 如果 SPH_64 宏未定义为真，则将 sc->bit_count_low 置为 0
#endif
}

static void
bmw32(sph_bmw_small_context *sc, const void *data, size_t len)  # 定义函数 bmw32，对输入数据进行 bmw32 哈希计算
{
    unsigned char *buf;  # 声明指向无符号字符的指针 buf
    size_t ptr;  # 声明大小为 size_t 的变量 ptr
    sph_u32 htmp[16];  # 声明包含 16 个元素的 sph_u32 类型数组 htmp
    sph_u32 *h1, *h2;  # 声明指向 sph_u32 类型的指针 h1 和 h2
#if !SPH_64
    sph_u32 tmp;  # 如果 SPH_64 宏未定义为真，则声明 sph_u32 类型的变量 tmp
#endif

#if SPH_64
    sc->bit_count += (sph_u64)len << 3;  # 如果 SPH_64 宏定义为真，则更新 sc->bit_count
#else
    tmp = sc->bit_count_low;  # 如果 SPH_64 宏未定义为真，则将 sc->bit_count_low 赋给 tmp
    sc->bit_count_low = SPH_T32(tmp + ((sph_u32)len << 3));  # 如果 SPH_64 宏未定义为真，则更新 sc->bit_count_low
    if (sc->bit_count_low < tmp)  # 如果新的 sc->bit_count_low 小于原始值 tmp
        sc->bit_count_high ++;  # 则更新 sc->bit_count_high
    sc->bit_count_high += len >> 29;  # 更新 sc->bit_count_high
#endif
    buf = sc->buf;  # 将 sc->buf 赋给 buf
    ptr = sc->ptr;  # 将 sc->ptr 赋给 ptr
    h1 = sc->H;  # 将 sc->H 赋给 h1
    h2 = htmp;  # 将 htmp 赋给 h2
    while (len > 0) {  # 循环，直到处理完所有输入数据
        size_t clen;  # 声明大小为 size_t 的变量 clen

        clen = (sizeof sc->buf) - ptr;  # 计算剩余缓冲区的大小
        if (clen > len)  # 如果剩余缓冲区大小大于输入数据长度
            clen = len;  # 则将 clen 设为输入数据长度
        memcpy(buf + ptr, data, clen);  # 将输入数据复制到缓冲区中
        data = (const unsigned char *)data + clen;  # 更新输入数据指针
        len -= clen;  # 更新剩余输入数据长度
        ptr += clen;  # 更新缓冲区指针
        if (ptr == sizeof sc->buf) {  # 如果缓冲区已满
            sph_u32 *ht;  # 声明指向 sph_u32 类型的指针 ht

            compress_small(buf, h1, h2);  # 调用 compress_small 函数进行压缩
            ht = h1;  # 将 h1 赋给 ht
            h1 = h2;  # 将 h2 赋给 h1
            h2 = ht;  # 将 ht 赋给 h2
            ptr = 0;  # 重置缓冲区指针
        }
    }
    sc->ptr = ptr;  # 更新上下文中的缓冲区指针
    if (h1 != sc->H)  # 如果 h1 不等于 sc->H
        memcpy(sc->H, h1, sizeof sc->H);  # 则将 h1 的内容复制到 sc->H 中
}

static void
bmw32_close(sph_bmw_small_context *sc, unsigned ub, unsigned n,
    void *dst, size_t out_size_w32)  # 定义函数 bmw32_close，用于结束 bmw32 哈希计算并输出结果
{
    unsigned char *buf, *out;  # 声明指向无符号字符的指针 buf 和 out
    size_t ptr, u, v;  # 声明大小为 size_t 的变量 ptr, u, v
    unsigned z;  # 声明无符号整数变量 z
    # 定义三个32位整数数组h1、h2和h，用于存储哈希值
    sph_u32 h1[16], h2[16], *h;

    # 从sc对象中获取缓冲区和指针
    buf = sc->buf;
    ptr = sc->ptr;

    # 计算填充字节的值
    z = 0x80 >> n;

    # 将填充字节添加到缓冲区中，并更新指针位置
    buf[ptr ++] = ((ub & -z) | z) & 0xFF;

    # 从sc对象中获取哈希值
    h = sc->H;

    # 如果指针位置超过缓冲区大小减8，则进行处理
    if (ptr > (sizeof sc->buf) - 8) {
        # 将缓冲区剩余部分填充为0
        memset(buf + ptr, 0, (sizeof sc->buf) - ptr);
        # 对填充后的缓冲区进行压缩，得到新的哈希值h1
        compress_small(buf, h, h1);
        # 重置指针位置为0，并更新哈希值h为h1
        ptr = 0;
        h = h1;
    }

    # 将缓冲区剩余部分填充为0
    memset(buf + ptr, 0, (sizeof sc->buf) - 8 - ptr);
#if SPH_64
    // 如果定义了 SPH_64，则执行以下代码块
    sph_enc64le_aligned(buf + (sizeof sc->buf) - 8, SPH_T64(sc->bit_count + n));
    // 将 sc->bit_count + n 转换为小端序的64位整数，存储到 buf 的末尾减8的位置
#else
    // 如果未定义 SPH_64，则执行以下代码块
    sph_enc32le_aligned(buf + (sizeof sc->buf) - 8, sc->bit_count_low + n);
    // 将 sc->bit_count_low + n 转换为小端序的32位整数，存储到 buf 的末尾减8的位置
    sph_enc32le_aligned(buf + (sizeof sc->buf) - 4, SPH_T32(sc->bit_count_high));
    // 将 sc->bit_count_high 转换为小端序的32位整数，存储到 buf 的末尾减4的位置
#endif
    compress_small(buf, h, h2);
    // 调用 compress_small 函数，传入参数 buf, h, h2
    for (u = 0; u < 16; u ++)
        // 循环遍历 u 从 0 到 15
        sph_enc32le_aligned(buf + 4 * u, h2[u]);
        // 将 h2[u] 转换为小端序的32位整数，存储到 buf 的 4*u 的位置
    compress_small(buf, final_s, h1);
    // 调用 compress_small 函数，传入参数 buf, final_s, h1
    out = dst;
    // 将 dst 赋值给 out
    for (u = 0, v = 16 - out_size_w32; u < out_size_w32; u ++, v ++)
        // 循环遍历 u 从 0 到 out_size_w32-1，同时遍历 v 从 16 - out_size_w32 到 15
        sph_enc32le(out + 4 * u, h1[v]);
        // 将 h1[v] 转换为小端序的32位整数，存储到 out 的 4*u 的位置
}

#endif // !AVX2

#if SPH_64
    // 如果定义了 SPH_64，则执行以下代码块
static void
compress_big(const unsigned char *data, const sph_u64 h[16], sph_u64 dh[16])
{
    // 声明一个名为 compress_big 的静态函数，接受参数 data, h, dh
#if SPH_LITTLE_FAST
#define M(x)    sph_dec64le_aligned(data + 8 * (x))
    // 如果定义了 SPH_LITTLE_FAST，则使用 sph_dec64le_aligned 宏定义 M(x)
#else
    // 如果未定义 SPH_LITTLE_FAST，则执行以下代码块
    sph_u64 mv[16];
    // 声明一个名为 mv 的64位整数数组，长度为16
    mv[ 0] = sph_dec64le_aligned(data +   0);
    // 将 data + 0 处的数据转换为小端序的64位整数，存储到 mv[0] 中
    mv[ 1] = sph_dec64le_aligned(data +   8);
    // 将 data + 8 处的数据转换为小端序的64位整数，存储到 mv[1] 中
    // ... 以此类推，依次将 data 中的数据转换为小端序的64位整数，存储到 mv 数组中
#define M(x)    (mv[x])
    // 使用宏定义 M(x)，将 mv[x] 替换为 (mv[x])
#endif
#define H(x)    (h[x])
#define dH(x)   (dh[x])

    FOLDb;
    // 调用 FOLDb 宏

#undef M
#undef H
#undef dH
    // 取消宏定义 M, H, dH
}

static const sph_u64 final_b[16] = {
    // 声明一个名为 final_b 的64位整数数组，长度为16
    SPH_C64(0xaaaaaaaaaaaaaaa0), SPH_C64(0xaaaaaaaaaaaaaaa1),
    SPH_C64(0xaaaaaaaaaaaaaaa2), SPH_C64(0xaaaaaaaaaaaaaaa3),
    SPH_C64(0xaaaaaaaaaaaaaaa4), SPH_C64(0xaaaaaaaaaaaaaaa5),
    SPH_C64(0xaaaaaaaaaaaaaaa6), SPH_C64(0xaaaaaaaaaaaaaaa7),
    SPH_C64(0xaaaaaaaaaaaaaaa8), SPH_C64(0xaaaaaaaaaaaaaaa9),
    // 将给定的64位整数值存储到 final_b 数组中
    # 使用SPH_C64函数将十六进制数转换为64位整数，并依次传入参数
    SPH_C64(0xaaaaaaaaaaaaaaaa), SPH_C64(0xaaaaaaaaaaaaaaab),
    SPH_C64(0xaaaaaaaaaaaaaaac), SPH_C64(0xaaaaaaaaaaaaaaad),
    SPH_C64(0xaaaaaaaaaaaaaaae), SPH_C64(0xaaaaaaaaaaaaaaaf)
# 初始化函数，用于初始化 bmw64 算法的上下文和初始向量
static void
bmw64_init(sph_bmw_big_context *sc, const sph_u64 *iv)
{
    # 将初始向量 iv 的内容复制到上下文 sc 的 H 数组中
    memcpy(sc->H, iv, sizeof sc->H);
    # 将指针 ptr 置零
    sc->ptr = 0;
    # 将比特计数 bit_count 置零
    sc->bit_count = 0;
}

# bmw64 算法的处理函数，用于处理输入数据
static void
bmw64(sph_bmw_big_context *sc, const void *data, size_t len)
{
    unsigned char *buf;
    size_t ptr;
    sph_u64 htmp[16];
    sph_u64 *h1, *h2;

    # 更新比特计数，将输入数据的长度左移 3 位并加到比特计数中
    sc->bit_count += (sph_u64)len << 3;
    buf = sc->buf;
    ptr = sc->ptr;
    h1 = sc->H;
    h2 = htmp;
    # 处理输入数据
    while (len > 0) {
        size_t clen;

        clen = (sizeof sc->buf) - ptr;
        if (clen > len)
            clen = len;
        # 将数据复制到缓冲区中
        memcpy(buf + ptr, data, clen);
        data = (const unsigned char *)data + clen;
        len -= clen;
        ptr += clen;
        # 如果缓冲区满了，进行压缩操作
        if (ptr == sizeof sc->buf) {
            sph_u64 *ht;

            compress_big(buf, h1, h2);
            ht = h1;
            h1 = h2;
            h2 = ht;
            ptr = 0;
        }
    }
    sc->ptr = ptr;
    # 如果 h1 不等于上下文中的 H 数组，则将 h1 的内容复制到 H 数组中
    if (h1 != sc->H)
        memcpy(sc->H, h1, sizeof sc->H);
}

# bmw64 算法的结束函数，用于处理最后的数据块并生成输出
static void
bmw64_close(sph_bmw_big_context *sc, unsigned ub, unsigned n,
    void *dst, size_t out_size_w64)
{
    unsigned char *buf, *out;
    size_t ptr, u, v;
    unsigned z;
    sph_u64 h1[16], h2[16], *h;

    buf = sc->buf;
    ptr = sc->ptr;
    z = 0x80 >> n;
    buf[ptr ++] = ((ub & -z) | z) & 0xFF;
    h = sc->H;
    # 如果缓冲区快满了，进行压缩操作
    if (ptr > (sizeof sc->buf) - 8) {
        memset(buf + ptr, 0, (sizeof sc->buf) - ptr);
        compress_big(buf, h, h1);
        ptr = 0;
        h = h1;
    }
    memset(buf + ptr, 0, (sizeof sc->buf) - 8 - ptr);
    sph_enc64le_aligned(buf + (sizeof sc->buf) - 8,
        SPH_T64(sc->bit_count + n));
    compress_big(buf, h, h2);
    for (u = 0; u < 16; u ++)
        sph_enc64le_aligned(buf + 8 * u, h2[u]);
    compress_big(buf, final_b, h1);
    out = dst;
    for (u = 0, v = 16 - out_size_w64; u < out_size_w64; u ++, v ++)
        sph_enc64le(out + 8 * u, h1[v]);
}

#endif

# 如果未定义 __AVX2__，则初始化 bmw224 算法的上下文
#if !defined(__AVX2__)
void
sph_bmw224_init(void *cc)
{
    bmw32_init(cc, IV224);
}
/* see sph_bmw.h */
// 定义了一个名为sph_bmw224的函数，接受一个指向数据的指针和数据的长度作为参数
void
sph_bmw224(void *cc, const void *data, size_t len)
{
    // 调用bmw32函数处理数据
    bmw32(cc, data, len);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw224_close的函数，接受一个指向数据的指针和一个指向目标的指针作为参数
void
sph_bmw224_close(void *cc, void *dst)
{
    // 调用sph_bmw224_addbits_and_close函数，传入参数0, 0, dst
    sph_bmw224_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw224_addbits_and_close的函数，接受一个指向数据的指针，两个无符号整数和一个指向目标的指针作为参数
void
sph_bmw224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用bmw32_close函数，传入参数cc, ub, n, dst, 7
    bmw32_close(cc, ub, n, dst, 7);
//    sph_bmw224_init(cc);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw256_init的函数，接受一个指向数据的指针作为参数
void
sph_bmw256_init(void *cc)
{
    // 调用bmw32_init函数，传入参数cc, IV256
    bmw32_init(cc, IV256);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw256的函数，接受一个指向数据的指针和数据的长度作为参数
void
sph_bmw256(void *cc, const void *data, size_t len)
{
    // 调用bmw32函数处理数据
    bmw32(cc, data, len);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw256_close的函数，接受一个指向数据的指针和一个指向目标的指针作为参数
void
sph_bmw256_close(void *cc, void *dst)
{
    // 调用sph_bmw256_addbits_and_close函数，传入参数0, 0, dst
    sph_bmw256_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw256_addbits_and_close的函数，接受一个指向数据的指针，两个无符号整数和一个指向目标的指针作为参数
void
sph_bmw256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用bmw32_close函数，传入参数cc, ub, n, dst, 8
    bmw32_close(cc, ub, n, dst, 8);
//    sph_bmw256_init(cc);
}

#endif // !AVX2

#if SPH_64

/* see sph_bmw.h */
// 定义了一个名为sph_bmw384_init的函数，接受一个指向数据的指针作为参数
void
sph_bmw384_init(void *cc)
{
    // 调用bmw64_init函数，传入参数cc, IV384
    bmw64_init(cc, IV384);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw384的函数，接受一个指向数据的指针和数据的长度作为参数
void
sph_bmw384(void *cc, const void *data, size_t len)
{
    // 调用bmw64函数处理数据
    bmw64(cc, data, len);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw384_close的函数，接受一个指向数据的指针和一个指向目标的指针作为参数
void
sph_bmw384_close(void *cc, void *dst)
{
    // 调用sph_bmw384_addbits_and_close函数，传入参数0, 0, dst
    sph_bmw384_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw384_addbits_and_close的函数，接受一个指向数据的指针，两个无符号整数和一个指向目标的指针作为参数
void
sph_bmw384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用bmw64_close函数，传入参数cc, ub, n, dst, 6
    bmw64_close(cc, ub, n, dst, 6);
//    sph_bmw384_init(cc);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw512_init的函数，接受一个指向数据的指针作为参数
void
sph_bmw512_init(void *cc)
{
    // 调用bmw64_init函数，传入参数cc, IV512
    bmw64_init(cc, IV512);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw512的函数，接受一个指向数据的指针和数据的长度作为参数
void
sph_bmw512(void *cc, const void *data, size_t len)
{
    // 调用bmw64函数处理数据
    bmw64(cc, data, len);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw512_close的函数，接受一个指向数据的指针和一个指向目标的指针作为参数
void
sph_bmw512_close(void *cc, void *dst)
{
    // 调用sph_bmw512_addbits_and_close函数，传入参数0, 0, dst
    sph_bmw512_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_bmw.h */
// 定义了一个名为sph_bmw512_addbits_and_close的函数，接受一个指向数据的指针，两个无符号整数和一个指向目标的指针作为参数
void
sph_bmw512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用bmw64_close函数，传入参数cc, ub, n, dst, 8
    bmw64_close(cc, ub, n, dst, 8);
//    sph_bmw512_init(cc);
}

#endif

#ifdef __cplusplus
}
#endif
```