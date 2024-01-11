# `xmrig\src\crypto\ghostrider\sph_keccak.c`

```
/* $Id: keccak.c 259 2011-07-19 22:11:27Z tp $ */
/*
 * Keccak implementation.
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

#include "sph_keccak.h"
#include <stddef.h>
#include <string.h>

#ifdef __cplusplus
extern "C" {
#endif

// Taken from keccak-gate.c
int hard_coded_eb = 1;

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_KECCAK
#define SPH_SMALL_FOOTPRINT_KECCAK 1
#endif

/*
 * By default, we select the 64-bit implementation if a 64-bit type
 * is available, unless a 32-bit x86 is detected.
 */
#if !defined SPH_KECCAK_64 && SPH_64 &&                                        \
    !(defined __i386__ || SPH_I386_GCC || SPH_I386_MSVC)
#define SPH_KECCAK_64 1
#endif
/*
 * 如果使用 32 位实现，我们更喜欢交错。
 */
#if !SPH_KECCAK_64 && !defined SPH_KECCAK_INTERLEAVE
#define SPH_KECCAK_INTERLEAVE 1
#endif

/*
 * 在大系统上展开 8 轮，小系统上展开 2 轮。
 */
#ifndef SPH_KECCAK_UNROLL
#if SPH_SMALL_FOOTPRINT_KECCAK
#define SPH_KECCAK_UNROLL 2
#else
#define SPH_KECCAK_UNROLL 8
#endif
#endif

/*
 * 我们不希望在 x86（32 位和 64 位）上将状态复制到本地变量。
 */
#ifndef SPH_KECCAK_NOCOPY
#if defined __i386__ || defined __x86_64 || SPH_I386_MSVC || SPH_I386_GCC
#define SPH_KECCAK_NOCOPY 1
#else
#define SPH_KECCAK_NOCOPY 0
#endif
#endif

#ifdef _MSC_VER
#pragma warning(disable : 4146)
#endif

#if SPH_KECCAK_64

static const sph_u64 RC[] = {
    SPH_C64(0x0000000000000001), SPH_C64(0x0000000000008082),
    SPH_C64(0x800000000000808A), SPH_C64(0x8000000080008000),
    SPH_C64(0x000000000000808B), SPH_C64(0x0000000080000001),
    SPH_C64(0x8000000080008081), SPH_C64(0x8000000000008009),
    SPH_C64(0x000000000000008A), SPH_C64(0x0000000000000088),
    SPH_C64(0x0000000080008009), SPH_C64(0x000000008000000A),
    SPH_C64(0x000000008000808B), SPH_C64(0x800000000000008B),
    SPH_C64(0x8000000000008089), SPH_C64(0x8000000000008003),
    SPH_C64(0x8000000000008002), SPH_C64(0x8000000000000080),
    SPH_C64(0x000000000000800A), SPH_C64(0x800000008000000A),
    SPH_C64(0x8000000080008081), SPH_C64(0x8000000000008080),
    SPH_C64(0x0000000080000001), SPH_C64(0x8000000080008008)};

#if SPH_KECCAK_NOCOPY

#define a00 (kc->u.wide[0])
#define a10 (kc->u.wide[1])
#define a20 (kc->u.wide[2])
#define a30 (kc->u.wide[3])
#define a40 (kc->u.wide[4])
#define a01 (kc->u.wide[5])
#define a11 (kc->u.wide[6])
#define a21 (kc->u.wide[7])
#define a31 (kc->u.wide[8])
#define a41 (kc->u.wide[9])
#define a02 (kc->u.wide[10])
#define a12 (kc->u.wide[11])
#define a22 (kc->u.wide[12])
#define a32 (kc->u.wide[13])
#define a42 (kc->u.wide[14])
#define a03 (kc->u.wide[15])
#define a13 (kc->u.wide[16])
#define a23 (kc->u.wide[17])  // 定义宏a23，表示kc结构体中u.wide数组的第18个元素
#define a33 (kc->u.wide[18])  // 定义宏a33，表示kc结构体中u.wide数组的第19个元素
#define a43 (kc->u.wide[19])  // 定义宏a43，表示kc结构体中u.wide数组的第20个元素
#define a04 (kc->u.wide[20])  // 定义宏a04，表示kc结构体中u.wide数组的第21个元素
#define a14 (kc->u.wide[21])  // 定义宏a14，表示kc结构体中u.wide数组的第22个元素
#define a24 (kc->u.wide[22])  // 定义宏a24，表示kc结构体中u.wide数组的第23个元素
#define a34 (kc->u.wide[23])  // 定义宏a34，表示kc结构体中u.wide数组的第24个元素
#define a44 (kc->u.wide[24])  // 定义宏a44，表示kc结构体中u.wide数组的第25个元素

#define DECL_STATE  // 定义宏DECL_STATE
#define READ_STATE(sc)  // 定义宏READ_STATE，参数为sc
#define WRITE_STATE(sc)  // 定义宏WRITE_STATE，参数为sc

#define INPUT_BUF(size)                                                        \  // 定义宏INPUT_BUF，参数为size
  do {                                                                         \
    size_t j;                                                                  \
    for (j = 0; j < (size); j += 8) {                                          \
      kc->u.wide[j >> 3] ^= sph_dec64le_aligned(buf + j);                      \
    }                                                                          \
  } while (0)

#define INPUT_BUF144 INPUT_BUF(144)  // 定义宏INPUT_BUF144，表示调用INPUT_BUF宏，参数为144
#define INPUT_BUF136 INPUT_BUF(136)  // 定义宏INPUT_BUF136，表示调用INPUT_BUF宏，参数为136
#define INPUT_BUF104 INPUT_BUF(104)  // 定义宏INPUT_BUF104，表示调用INPUT_BUF宏，参数为104
#define INPUT_BUF72 INPUT_BUF(72)    // 定义宏INPUT_BUF72，表示调用INPUT_BUF宏，参数为72

#else

#define DECL_STATE                                                             \  // 定义宏DECL_STATE
  sph_u64 a00, a01, a02, a03, a04;                                             \
  sph_u64 a10, a11, a12, a13, a14;                                             \
  sph_u64 a20, a21, a22, a23, a24;                                             \
  sph_u64 a30, a31, a32, a33, a34;                                             \
  sph_u64 a40, a41, a42, a43, a44;

#define READ_STATE(state)                                                      \  // 定义宏READ_STATE，参数为state
  do {                                                                         \
    a00 = (state)->u.wide[0];                                                  \
    a10 = (state)->u.wide[1];                                                  \
    a20 = (state)->u.wide[2];                                                  \
    a30 = (state)->u.wide[3];                                                  \
    a40 = (state)->u.wide[4];                                                  \
    # 从状态对象中获取对应位置的数据，存入变量a01到a44中
    a01 = (state)->u.wide[5];                                                  
    a11 = (state)->u.wide[6];                                                  
    a21 = (state)->u.wide[7];                                                  
    a31 = (state)->u.wide[8];                                                  
    a41 = (state)->u.wide[9];                                                  
    a02 = (state)->u.wide[10];                                                 
    a12 = (state)->u.wide[11];                                                 
    a22 = (state)->u.wide[12];                                                 
    a32 = (state)->u.wide[13];                                                 
    a42 = (state)->u.wide[14];                                                 
    a03 = (state)->u.wide[15];                                                 
    a13 = (state)->u.wide[16];                                                 
    a23 = (state)->u.wide[17];                                                 
    a33 = (state)->u.wide[18];                                                 
    a43 = (state)->u.wide[19];                                                 
    a04 = (state)->u.wide[20];                                                 
    a14 = (state)->u.wide[21];                                                 
    a24 = (state)->u.wide[22];                                                 
    a34 = (state)->u.wide[23];                                                 
    a44 = (state)->u.wide[24];                                                 
  } while (0)
# 宏定义，用于将状态数据写入状态对象
#define WRITE_STATE(state)                                                     \
  do {                                                                         \
    # 将状态数据写入状态对象的不同位置
    (state)->u.wide[0] = a00;                                                  
    (state)->u.wide[1] = a10;                                                  
    (state)->u.wide[2] = a20;                                                  
    (state)->u.wide[3] = a30;                                                  
    (state)->u.wide[4] = a40;                                                  
    (state)->u.wide[5] = a01;                                                  
    (state)->u.wide[6] = a11;                                                  
    (state)->u.wide[7] = a21;                                                  
    (state)->u.wide[8] = a31;                                                  
    (state)->u.wide[9] = a41;                                                  
    (state)->u.wide[10] = a02;                                                 
    (state)->u.wide[11] = a12;                                                 
    (state)->u.wide[12] = a22;                                                 
    (state)->u.wide[13] = a32;                                                 
    (state)->u.wide[14] = a42;                                                 
    (state)->u.wide[15] = a03;                                                 
    (state)->u.wide[16] = a13;                                                 
    (state)->u.wide[17] = a23;                                                 
    (state)->u.wide[18] = a33;                                                 
    (state)->u.wide[19] = a43;                                                 
    (state)->u.wide[20] = a04;                                                 
    (state)->u.wide[21] = a14;                                                 
    # 将变量 a14 赋值给 state 对象中的 u.wide[20]
    (state)->u.wide[20] = a14;                                                 \
    # 将变量 a24 赋值给 state 对象中的 u.wide[22]
    (state)->u.wide[22] = a24;                                                 \
    # 将变量 a34 赋值给 state 对象中的 u.wide[23]
    (state)->u.wide[23] = a34;                                                 \
    # 将变量 a44 赋值给 state 对象中的 u.wide[24]
    (state)->u.wide[24] = a44;                                                 \
# 定义一个宏，用于将输入缓冲区中的数据按照特定规则进行异或操作
#define INPUT_BUF144                                                           \
  do {                                                                         \
    # 对输入缓冲区中的数据进行64位小端解码，并与a00进行异或操作
    a00 ^= sph_dec64le_aligned(buf + 0);                                       \
    # 对输入缓冲区中的数据进行64位小端解码，并与a10进行异或操作
    a10 ^= sph_dec64le_aligned(buf + 8);                                       \
    # 对输入缓冲区中的数据进行64位小端解码，并与a20进行异或操作
    a20 ^= sph_dec64le_aligned(buf + 16);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a30进行异或操作
    a30 ^= sph_dec64le_aligned(buf + 24);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a40进行异或操作
    a40 ^= sph_dec64le_aligned(buf + 32);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a01进行异或操作
    a01 ^= sph_dec64le_aligned(buf + 40);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a11进行异或操作
    a11 ^= sph_dec64le_aligned(buf + 48);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a21进行异或操作
    a21 ^= sph_dec64le_aligned(buf + 56);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a31进行异或操作
    a31 ^= sph_dec64le_aligned(buf + 64);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a41进行异或操作
    a41 ^= sph_dec64le_aligned(buf + 72);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a02进行异或操作
    a02 ^= sph_dec64le_aligned(buf + 80);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a12进行异或操作
    a12 ^= sph_dec64le_aligned(buf + 88);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a22进行异或操作
    a22 ^= sph_dec64le_aligned(buf + 96);                                      \
    # 对输入缓冲区中的数据进行64位小端解码，并与a32进行异或操作
    a32 ^= sph_dec64le_aligned(buf + 104);                                     \
    # 对输入缓冲区中的数据进行64位小端解码，并与a42进行异或操作
    a42 ^= sph_dec64le_aligned(buf + 112);                                     \
    # 对输入缓冲区中的数据进行64位小端解码，并与a03进行异或操作
    a03 ^= sph_dec64le_aligned(buf + 120);                                     \
    # 对输入缓冲区中的数据进行64位小端解码，并与a13进行异或操作
    a13 ^= sph_dec64le_aligned(buf + 128);                                     \
    # 对输入缓冲区中的数据进行64位小端解码，并与a23进行异或操作
    a23 ^= sph_dec64le_aligned(buf + 136);                                     \
  } while (0)

# 定义一个宏，用于将输入缓冲区中的数据按照特定规则进行异或操作
#define INPUT_BUF136                                                           \
  do {                                                                         \
    # 对输入缓冲区中的数据进行64位小端解码，并与a00进行异或操作
    a00 ^= sph_dec64le_aligned(buf + 0);                                       \
    # 对输入缓冲区中的数据进行64位小端解码，并与a10进行异或操作
    a10 ^= sph_dec64le_aligned(buf + 8);                                       \
    # 对 a20 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 16 处的数据
    a20 ^= sph_dec64le_aligned(buf + 16);
    # 对 a30 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 24 处的数据
    a30 ^= sph_dec64le_aligned(buf + 24);
    # 对 a40 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 32 处的数据
    a40 ^= sph_dec64le_aligned(buf + 32);
    # 对 a01 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 40 处的数据
    a01 ^= sph_dec64le_aligned(buf + 40);
    # 对 a11 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 48 处的数据
    a11 ^= sph_dec64le_aligned(buf + 48);
    # 对 a21 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 56 处的数据
    a21 ^= sph_dec64le_aligned(buf + 56);
    # 对 a31 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 64 处的数据
    a31 ^= sph_dec64le_aligned(buf + 64);
    # 对 a41 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 72 处的数据
    a41 ^= sph_dec64le_aligned(buf + 72);
    # 对 a02 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 80 处的数据
    a02 ^= sph_dec64le_aligned(buf + 80);
    # 对 a12 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 88 处的数据
    a12 ^= sph_dec64le_aligned(buf + 88);
    # 对 a22 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 96 处的数据
    a22 ^= sph_dec64le_aligned(buf + 96);
    # 对 a32 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 104 处的数据
    a32 ^= sph_dec64le_aligned(buf + 104);
    # 对 a42 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 112 处的数据
    a42 ^= sph_dec64le_aligned(buf + 112);
    # 对 a03 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 120 处的数据
    a03 ^= sph_dec64le_aligned(buf + 120);
    # 对 a13 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 128 处的数据
    a13 ^= sph_dec64le_aligned(buf + 128);
# 定义一个宏，用于将输入缓冲区中的数据按照64位小端格式解码，并与对应的变量进行异或操作
#define INPUT_BUF104                                                           \
  do {                                                                         \
    a00 ^= sph_dec64le_aligned(buf + 0);                                       \  # 对 a00 变量进行异或操作
    a10 ^= sph_dec64le_aligned(buf + 8);                                       \  # 对 a10 变量进行异或操作
    a20 ^= sph_dec64le_aligned(buf + 16);                                      \  # 对 a20 变量进行异或操作
    a30 ^= sph_dec64le_aligned(buf + 24);                                      \  # 对 a30 变量进行异或操作
    a40 ^= sph_dec64le_aligned(buf + 32);                                      \  # 对 a40 变量进行异或操作
    a01 ^= sph_dec64le_aligned(buf + 40);                                      \  # 对 a01 变量进行异或操作
    a11 ^= sph_dec64le_aligned(buf + 48);                                      \  # 对 a11 变量进行异或操作
    a21 ^= sph_dec64le_aligned(buf + 56);                                      \  # 对 a21 变量进行异或操作
    a31 ^= sph_dec64le_aligned(buf + 64);                                      \  # 对 a31 变量进行异或操作
    a41 ^= sph_dec64le_aligned(buf + 72);                                      \  # 对 a41 变量进行异或操作
    a02 ^= sph_dec64le_aligned(buf + 80);                                      \  # 对 a02 变量进行异或操作
    a12 ^= sph_dec64le_aligned(buf + 88);                                      \  # 对 a12 变量进行异或操作
    a22 ^= sph_dec64le_aligned(buf + 96);                                      \  # 对 a22 变量进行异或操作
  } while (0)                                                                  \  # 宏定义结束

# 定义另一个宏，用于将输入缓冲区中的数据按照64位小端格式解码，并与对应的变量进行异或操作
#define INPUT_BUF72                                                            \
  do {                                                                         \
    a00 ^= sph_dec64le_aligned(buf + 0);                                       \  # 对 a00 变量进行异或操作
    a10 ^= sph_dec64le_aligned(buf + 8);                                       \  # 对 a10 变量进行异或操作
    a20 ^= sph_dec64le_aligned(buf + 16);                                      \  # 对 a20 变量进行异或操作
    a30 ^= sph_dec64le_aligned(buf + 24);                                      \  # 对 a30 变量进行异或操作
    a40 ^= sph_dec64le_aligned(buf + 32);                                      \  # 对 a40 变量进行异或操作
    a01 ^= sph_dec64le_aligned(buf + 40);                                      \  # 对 a01 变量进行异或操作
    a11 ^= sph_dec64le_aligned(buf + 48);                                      \  # 对 a11 变量进行异或操作
    # 对 a21 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 56 处的数据
    a21 ^= sph_dec64le_aligned(buf + 56);
    # 对 a31 进行异或操作，使用 sph_dec64le_aligned 函数解析 buf + 64 处的数据
    a31 ^= sph_dec64le_aligned(buf + 64);
# 定义一个宏，用于处理输入缓冲区
#define INPUT_BUF(lim)                                                         \
  do {                                                                         \
    # 从缓冲区中提取64位数据，并与a00进行异或操作
    a00 ^= sph_dec64le_aligned(buf + 0);                                       \
    # 从缓冲区中提取64位数据，并与a10进行异或操作
    a10 ^= sph_dec64le_aligned(buf + 8);                                       \
    # 从缓冲区中提取64位数据，并与a20进行异或操作
    a20 ^= sph_dec64le_aligned(buf + 16);                                      \
    # 从缓冲区中提取64位数据，并与a30进行异或操作
    a30 ^= sph_dec64le_aligned(buf + 24);                                      \
    # 从缓冲区中提取64位数据，并与a40进行异或操作
    a40 ^= sph_dec64le_aligned(buf + 32);                                      \
    # 从缓冲区中提取64位数据，并与a01进行异或操作
    a01 ^= sph_dec64le_aligned(buf + 40);                                      \
    # 从缓冲区中提取64位数据，并与a11进行异或操作
    a11 ^= sph_dec64le_aligned(buf + 48);                                      \
    # 从缓冲区中提取64位数据，并与a21进行异或操作
    a21 ^= sph_dec64le_aligned(buf + 56);                                      \
    # 从缓冲区中提取64位数据，并与a31进行异或操作
    a31 ^= sph_dec64le_aligned(buf + 64);                                      \
    # 如果输入限制为72，则跳出循环
    if ((lim) == 72)                                                           \
      break;                                                                   \
    # 从缓冲区中提取64位数据，并与a41进行异或操作
    a41 ^= sph_dec64le_aligned(buf + 72);                                      \
    # 从缓冲区中提取64位数据，并与a02进行异或操作
    a02 ^= sph_dec64le_aligned(buf + 80);                                      \
    # 从缓冲区中提取64位数据，并与a12进行异或操作
    a12 ^= sph_dec64le_aligned(buf + 88);                                      \
    # 从缓冲区中提取64位数据，并与a22进行异或操作
    a22 ^= sph_dec64le_aligned(buf + 96);                                      \
    # 如果输入限制为104，则跳出循环
    if ((lim) == 104)                                                          \
      break;                                                                   \
    # 从缓冲区中提取64位数据，并与a32进行异或操作
    a32 ^= sph_dec64le_aligned(buf + 104);                                     \
    # 从缓冲区中提取64位数据，并与a42进行异或操作
    a42 ^= sph_dec64le_aligned(buf + 112);                                     \
    # 从缓冲区中提取64位数据，并与a03进行异或操作
    a03 ^= sph_dec64le_aligned(buf + 120);                                     \
    # 从缓冲区中提取64位数据，并与a13进行异或操作
    a13 ^= sph_dec64le_aligned(buf + 128);                                     \
    # 如果 lim 的值等于 136，则跳出循环
    if ((lim) == 136)                                                          \
      break;                                                                   \
    # 对 a23 进行异或操作，使用 sph_dec64le_aligned 函数从 buf 的偏移量 136 处读取数据并解码
    a23 ^= sph_dec64le_aligned(buf + 136);                                     \
// 如果定义了 SPH_KECCAK_INTERLEAVE，则使用交错的常量
// 否则使用非交错的常量
#ifdef SPH_KECCAK_INTERLEAVE

// 定义一个 64 位无符号整数
#define DECL64(x) sph_u64 x
// 将 s 赋值给 d
#define MOV64(d, s) (d = s)
// 将 a 和 b 的异或结果赋值给 d
#define XOR64(d, a, b) (d = a ^ b)
// 将 a 和 b 的与结果赋值给 d
#define AND64(d, a, b) (d = a & b)
// 将 a 和 b 的或结果赋值给 d
#define OR64(d, a, b) (d = a | b)
// 将 s 的按位取反结果赋值给 d
#define NOT64(d, s) (d = SPH_T64(~s))
// 将 v 向左循环移位 n 位后的结果赋值给 d
#define ROL64(d, v, n) (d = SPH_ROTL64(v, n))
// 定义 XOR64_IOTA 为 XOR64

#else

// 定义一个包含高位和低位的结构体数组 RC
static const struct {
  sph_u32 high, low;
} RC[] = {
// 如果定义了 SPH_KECCAK_INTERLEAVE，则使用交错的常量
// 否则使用非交错的常量
#if SPH_KECCAK_INTERLEAVE
    // ...
#else
    // ...
#endif
    # 使用SPH_C32宏将十六进制数转换为32位无符号整数，并将其作为数组元素的值
    {SPH_C32(0x00000000), SPH_C32(0x00000088)},
    {SPH_C32(0x00000000), SPH_C32(0x80008009)},
    {SPH_C32(0x00000000), SPH_C32(0x8000000A)},
    {SPH_C32(0x00000000), SPH_C32(0x8000808B)},
    {SPH_C32(0x80000000), SPH_C32(0x0000008B)},
    {SPH_C32(0x80000000), SPH_C32(0x00008089)},
    {SPH_C32(0x80000000), SPH_C32(0x00008003)},
    {SPH_C32(0x80000000), SPH_C32(0x00008002)},
    {SPH_C32(0x80000000), SPH_C32(0x00000080)},
    {SPH_C32(0x00000000), SPH_C32(0x0000800A)},
    {SPH_C32(0x80000000), SPH_C32(0x8000000A)},
    {SPH_C32(0x80000000), SPH_C32(0x80008081)},
    {SPH_C32(0x80000000), SPH_C32(0x00008080)},
    {SPH_C32(0x00000000), SPH_C32(0x80000001)},
    {SPH_C32(0x80000000), SPH_C32(0x80008008)}
#endif
};

#if SPH_KECCAK_INTERLEAVE

// 定义一个宏，用于对两个32位整数进行交错处理
#define INTERLEAVE(xl, xh)                                                     \
  do {                                                                         \
    // 定义局部变量l、h、t，分别表示低32位、高32位、临时变量
    sph_u32 l, h, t;                                                           \
    // 对低32位进行交错处理
    l = (xl);                                                                  \
    h = (xh);                                                                  \
    t = (l ^ (l >> 1)) & SPH_C32(0x22222222);                                  \
    l ^= t ^ (t << 1);                                                         \
    t = (h ^ (h >> 1)) & SPH_C32(0x22222222);                                  \
    h ^= t ^ (t << 1);                                                         \
    t = (l ^ (l >> 2)) & SPH_C32(0x0C0C0C0C);                                  \
    l ^= t ^ (t << 2);                                                         \
    t = (h ^ (h >> 2)) & SPH_C32(0x0C0C0C0C);                                  \
    h ^= t ^ (t << 2);                                                         \
    t = (l ^ (l >> 4)) & SPH_C32(0x00F000F0);                                  \
    l ^= t ^ (t << 4);                                                         \
    t = (h ^ (h >> 4)) & SPH_C32(0x00F000F0);                                  \
    h ^= t ^ (t << 4);                                                         \
    t = (l ^ (l >> 8)) & SPH_C32(0x0000FF00);                                  \
    l ^= t ^ (t << 8);                                                         \
    t = (h ^ (h >> 8)) & SPH_C32(0x0000FF00);                                  \
    h ^= t ^ (t << 8);                                                         \
    t = (l ^ SPH_T32(h << 16)) & SPH_C32(0xFFFF0000);                          \
    l ^= t;                                                                    \
    h ^= t >> 16;                                                              \
    # 将 l 的值赋给 xl
    (xl) = l;                                                                  
    # 将 h 的值赋给 xh
    (xh) = h;                                                                  
  } while (0)
# 定义一个宏，用于对两个32位整数进行反交错操作
#define UNINTERLEAVE(xl, xh)                                                   \
  do {                                                                         \
    # 定义局部变量l，h，t，分别用于存储低位，高位，临时值
    sph_u32 l, h, t;                                                           \
    # 将xl和xh分别赋值给l和h
    l = (xl);                                                                  \
    h = (xh);                                                                  \
    # 对高位进行左移16位，并与低位进行异或操作，然后与0xFFFF0000进行按位与操作
    t = (l ^ SPH_T32(h << 16)) & SPH_C32(0xFFFF0000);                          \
    l ^= t;                                                                    \
    h ^= t >> 16;                                                              \
    # 对低位进行右移8位，并与自身进行异或操作，然后与0x0000FF00进行按位与操作
    t = (l ^ (l >> 8)) & SPH_C32(0x0000FF00);                                  \
    l ^= t ^ (t << 8);                                                         \
    # 对高位进行右移8位，并与自身进行异或操作，然后与0x0000FF00进行按位与操作
    t = (h ^ (h >> 8)) & SPH_C32(0x0000FF00);                                  \
    h ^= t ^ (t << 8);                                                         \
    # 对低位进行右移4位，并与自身进行异或操作，然后与0x00F000F0进行按位与操作
    t = (l ^ (l >> 4)) & SPH_C32(0x00F000F0);                                  \
    l ^= t ^ (t << 4);                                                         \
    # 对高位进行右移4位，并与自身进行异或操作，然后与0x00F000F0进行按位与操作
    t = (h ^ (h >> 4)) & SPH_C32(0x00F000F0);                                  \
    h ^= t ^ (t << 4);                                                         \
    # 对低位进行右移2位，并与自身进行异或操作，然后与0x0C0C0C0C进行按位与操作
    t = (l ^ (l >> 2)) & SPH_C32(0x0C0C0C0C);                                  \
    l ^= t ^ (t << 2);                                                         \
    # 对高位进行右移2位，并与自身进行异或操作，然后与0x0C0C0C0C进行按位与操作
    t = (h ^ (h >> 2)) & SPH_C32(0x0C0C0C0C);                                  \
    h ^= t ^ (t << 2);                                                         \
    # 对低位进行右移1位，并与自身进行异或操作，然后与0x22222222进行按位与操作
    t = (l ^ (l >> 1)) & SPH_C32(0x22222222);                                  \
    l ^= t ^ (t << 1);                                                         \
    # 对高位进行右移1位，并与自身进行异或操作，然后与0x22222222进行按位与操作
    t = (h ^ (h >> 1)) & SPH_C32(0x22222222);                                  \
    h ^= t ^ (t << 1);                                                         \
    # 将l的值赋给xl
    (xl) = l;                                                                  
    # 将h的值赋给xh
    (xh) = h;                                                                  
  } while (0)
#else

#define INTERLEAVE(l, h)  // 定义一个宏，用于交错存储
#define UNINTERLEAVE(l, h)  // 定义一个宏，用于取消交错存储

#endif

#if SPH_KECCAK_NOCOPY

#define a00l (kc->u.narrow[2 * 0 + 0])  // 定义宏a00l，表示kc->u.narrow[0]
#define a00h (kc->u.narrow[2 * 0 + 1])  // 定义宏a00h，表示kc->u.narrow[1]
// ... 依此类推，定义了很多宏，用于简化代码中的表达式
#define a34l (kc->u.narrow[2 * 23 + 0])  // 定义宏a34l，表示kc->u.narrow[46]
#define a34h (kc->u.narrow[2 * 23 + 1])  // 定义宏a34h，表示kc->u.narrow[47]
#define a44l (kc->u.narrow[2 * 24 + 0])  // 定义宏，将指针 kc 指向的 u.narrow 数组中的特定位置赋值给 a44l
#define a44h (kc->u.narrow[2 * 24 + 1])  // 定义宏，将指针 kc 指向的 u.narrow 数组中的特定位置赋值给 a44h

#define DECL_STATE  // 定义宏，用于声明状态变量
#define READ_STATE(state)  // 定义宏，用于读取状态
#define WRITE_STATE(state)  // 定义宏，用于写入状态

#define INPUT_BUF(size)  // 定义宏，用于处理输入缓冲区
  do {  // 执行输入缓冲区处理
    size_t j;  // 声明变量 j 用于循环
    for (j = 0; j < (size); j += 8) {  // 循环处理输入缓冲区
      sph_u32 tl, th;  // 声明变量 tl 和 th 用于存储数据
      tl = sph_dec32le_aligned(buf + j + 0);  // 从输入缓冲区中读取数据并赋值给 tl
      th = sph_dec32le_aligned(buf + j + 4);  // 从输入缓冲区中读取数据并赋值给 th
      INTERLEAVE(tl, th);  // 调用 INTERLEAVE 函数对 tl 和 th 进行交错处理
      kc->u.narrow[(j >> 2) + 0] ^= tl;  // 将 tl 与 u.narrow 数组中的特定位置进行异或运算
      kc->u.narrow[(j >> 2) + 1] ^= th;  // 将 th 与 u.narrow 数组中的特定位置进行异或运算
    }  // 结束循环
  } while (0)  // 结束输入缓冲区处理

#define INPUT_BUF144 INPUT_BUF(144)  // 定义宏，用于处理输入缓冲区大小为 144
#define INPUT_BUF136 INPUT_BUF(136)  // 定义宏，用于处理输入缓冲区大小为 136
#define INPUT_BUF104 INPUT_BUF(104)  // 定义宏，用于处理输入缓冲区大小为 104
#define INPUT_BUF72 INPUT_BUF(72)  // 定义宏，用于处理输入缓冲区大小为 72

#else  // 如果不是以上情况

#define DECL_STATE  // 定义宏，用于声明状态变量
  sph_u32 a00l, a00h, a01l, a01h, a02l, a02h, a03l, a03h, a04l, a04h;  // 声明一系列状态变量
  sph_u32 a10l, a10h, a11l, a11h, a12l, a12h, a13l, a13h, a14l, a14h;  // 声明一系列状态变量
  sph_u32 a20l, a20h, a21l, a21h, a22l, a22h, a23l, a23h, a24l, a24h;  // 声明一系列状态变量
  sph_u32 a30l, a30h, a31l, a31h, a32l, a32h, a33l, a33h, a34l, a34h;  // 声明一系列状态变量
  sph_u32 a40l, a40h, a41l, a41h, a42l, a42h, a43l, a43h, a44l, a44h;  // 声明一系列状态变量

#define READ_STATE(state)  // 定义宏，用于读取状态
  do {  // 执行读取状态
    a00l = (state)->u.narrow[2 * 0 + 0];  // 从状态中读取数据并赋值给 a00l
    # 从状态对象中获取特定位置的值，并赋给对应的变量
    a00h = (state)->u.narrow[2 * 0 + 1];                                       
    a10l = (state)->u.narrow[2 * 1 + 0];                                       
    a10h = (state)->u.narrow[2 * 1 + 1];                                       
    a20l = (state)->u.narrow[2 * 2 + 0];                                       
    a20h = (state)->u.narrow[2 * 2 + 1];                                       
    a30l = (state)->u.narrow[2 * 3 + 0];                                       
    a30h = (state)->u.narrow[2 * 3 + 1];                                       
    a40l = (state)->u.narrow[2 * 4 + 0];                                       
    a40h = (state)->u.narrow[2 * 4 + 1];                                       
    a01l = (state)->u.narrow[2 * 5 + 0];                                       
    a01h = (state)->u.narrow[2 * 5 + 1];                                       
    a11l = (state)->u.narrow[2 * 6 + 0];                                       
    a11h = (state)->u.narrow[2 * 6 + 1];                                       
    a21l = (state)->u.narrow[2 * 7 + 0];                                       
    a21h = (state)->u.narrow[2 * 7 + 1];                                       
    a31l = (state)->u.narrow[2 * 8 + 0];                                       
    a31h = (state)->u.narrow[2 * 8 + 1];                                       
    a41l = (state)->u.narrow[2 * 9 + 0];                                       
    a41h = (state)->u.narrow[2 * 9 + 1];                                       
    a02l = (state)->u.narrow[2 * 10 + 0];                                      
    a02h = (state)->u.narrow[2 * 10 + 1];                                      
    a12l = (state)->u.narrow[2 * 11 + 0];                                      
    a12h = (state)->u.narrow[2 * 11 + 1];                                      
    a22l = (state)->u.narrow[2 * 12 + 0];                                      
    
    
    注释：从状态对象中获取特定位置的值，并赋给对应的变量。这些变量的命名方式是根据位置计算得出的。
    # 从状态对象中获取特定位置的值，并赋给变量a22h
    a22h = (state)->u.narrow[2 * 12 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a32l
    a32l = (state)->u.narrow[2 * 13 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a32h
    a32h = (state)->u.narrow[2 * 13 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a42l
    a42l = (state)->u.narrow[2 * 14 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a42h
    a42h = (state)->u.narrow[2 * 14 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a03l
    a03l = (state)->u.narrow[2 * 15 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a03h
    a03h = (state)->u.narrow[2 * 15 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a13l
    a13l = (state)->u.narrow[2 * 16 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a13h
    a13h = (state)->u.narrow[2 * 16 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a23l
    a23l = (state)->u.narrow[2 * 17 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a23h
    a23h = (state)->u.narrow[2 * 17 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a33l
    a33l = (state)->u.narrow[2 * 18 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a33h
    a33h = (state)->u.narrow[2 * 18 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a43l
    a43l = (state)->u.narrow[2 * 19 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a43h
    a43h = (state)->u.narrow[2 * 19 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a04l
    a04l = (state)->u.narrow[2 * 20 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a04h
    a04h = (state)->u.narrow[2 * 20 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a14l
    a14l = (state)->u.narrow[2 * 21 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a14h
    a14h = (state)->u.narrow[2 * 21 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a24l
    a24l = (state)->u.narrow[2 * 22 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a24h
    a24h = (state)->u.narrow[2 * 22 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a34l
    a34l = (state)->u.narrow[2 * 23 + 0];
    # 从状态对象中获取特定位置的值，并赋给变量a34h
    a34h = (state)->u.narrow[2 * 23 + 1];
    # 从状态对象中获取特定位置的值，并赋给变量a44l
    a44l = (state)->u.narrow[2 * 24 + 0];
    # 定义一个变量a44h，其值为state中u.narrow[2 * 24 + 1]的值
    a44h = (state)->u.narrow[2 * 24 + 1];                                      \
# 宏定义，用于将状态数据写入状态对象
#define WRITE_STATE(state)                                                     \
  do {                                                                         \
    # 将状态数据写入状态对象的第一个窄型数组元素
    (state)->u.narrow[2 * 0 + 0] = a00l;                                      
    (state)->u.narrow[2 * 0 + 1] = a00h;                                      
    (state)->u.narrow[2 * 1 + 0] = a10l;                                      
    (state)->u.narrow[2 * 1 + 1] = a10h;                                      
    (state)->u.narrow[2 * 2 + 0] = a20l;                                      
    (state)->u.narrow[2 * 2 + 1] = a20h;                                      
    (state)->u.narrow[2 * 3 + 0] = a30l;                                      
    (state)->u.narrow[2 * 3 + 1] = a30h;                                      
    (state)->u.narrow[2 * 4 + 0] = a40l;                                      
    (state)->u.narrow[2 * 4 + 1] = a40h;                                      
    (state)->u.narrow[2 * 5 + 0] = a01l;                                      
    (state)->u.narrow[2 * 5 + 1] = a01h;                                      
    (state)->u.narrow[2 * 6 + 0] = a11l;                                      
    (state)->u.narrow[2 * 6 + 1] = a11h;                                      
    (state)->u.narrow[2 * 7 + 0] = a21l;                                      
    (state)->u.narrow[2 * 7 + 1] = a21h;                                      
    (state)->u.narrow[2 * 8 + 0] = a31l;                                      
    (state)->u.narrow[2 * 8 + 1] = a31h;                                      
    (state)->u.narrow[2 * 9 + 0] = a41l;                                      
    (state)->u.narrow[2 * 9 + 1] = a41h;                                      
    (state)->u.narrow[2 * 10 + 0] = a02l;                                     
    (state)->u.narrow[2 * 10 + 1] = a02h;                                     
    # 将变量 a12l 赋值给 state 结构体中 u.narrow 数组的第 2 * 11 + 0 个元素
    (state)->u.narrow[2 * 11 + 0] = a12l;
    # 将变量 a12h 赋值给 state 结构体中 u.narrow 数组的第 2 * 11 + 1 个元素
    (state)->u.narrow[2 * 11 + 1] = a12h;
    # 将变量 a22l 赋值给 state 结构体中 u.narrow 数组的第 2 * 12 + 0 个元素
    (state)->u.narrow[2 * 12 + 0] = a22l;
    # 将变量 a22h 赋值给 state 结构体中 u.narrow 数组的第 2 * 12 + 1 个元素
    (state)->u.narrow[2 * 12 + 1] = a22h;
    # 将变量 a32l 赋值给 state 结构体中 u.narrow 数组的第 2 * 13 + 0 个元素
    (state)->u.narrow[2 * 13 + 0] = a32l;
    # 将变量 a32h 赋值给 state 结构体中 u.narrow 数组的第 2 * 13 + 1 个元素
    (state)->u.narrow[2 * 13 + 1] = a32h;
    # 将变量 a42l 赋值给 state 结构体中 u.narrow 数组的第 2 * 14 + 0 个元素
    (state)->u.narrow[2 * 14 + 0] = a42l;
    # 将变量 a42h 赋值给 state 结构体中 u.narrow 数组的第 2 * 14 + 1 个元素
    (state)->u.narrow[2 * 14 + 1] = a42h;
    # 将变量 a03l 赋值给 state 结构体中 u.narrow 数组的第 2 * 15 + 0 个元素
    (state)->u.narrow[2 * 15 + 0] = a03l;
    # 将变量 a03h 赋值给 state 结构体中 u.narrow 数组的第 2 * 15 + 1 个元素
    (state)->u.narrow[2 * 15 + 1] = a03h;
    # 将变量 a13l 赋值给 state 结构体中 u.narrow 数组的第 2 * 16 + 0 个元素
    (state)->u.narrow[2 * 16 + 0] = a13l;
    # 将变量 a13h 赋值给 state 结构体中 u.narrow 数组的第 2 * 16 + 1 个元素
    (state)->u.narrow[2 * 16 + 1] = a13h;
    # 将变量 a23l 赋值给 state 结构体中 u.narrow 数组的第 2 * 17 + 0 个元素
    (state)->u.narrow[2 * 17 + 0] = a23l;
    # 将变量 a23h 赋值给 state 结构体中 u.narrow 数组的第 2 * 17 + 1 个元素
    (state)->u.narrow[2 * 17 + 1] = a23h;
    # 将变量 a33l 赋值给 state 结构体中 u.narrow 数组的第 2 * 18 + 0 个元素
    (state)->u.narrow[2 * 18 + 0] = a33l;
    # 将变量 a33h 赋值给 state 结构体中 u.narrow 数组的第 2 * 18 + 1 个元素
    (state)->u.narrow[2 * 18 + 1] = a33h;
    # 将变量 a43l 赋值给 state 结构体中 u.narrow 数组的第 2 * 19 + 0 个元素
    (state)->u.narrow[2 * 19 + 0] = a43l;
    # 将变量 a43h 赋值给 state 结构体中 u.narrow 数组的第 2 * 19 + 1 个元素
    (state)->u.narrow[2 * 19 + 1] = a43h;
    # 将变量 a04l 赋值给 state 结构体中 u.narrow 数组的第 2 * 20 + 0 个元素
    (state)->u.narrow[2 * 20 + 0] = a04l;
    # 将变量 a04h 赋值给 state 结构体中 u.narrow 数组的第 2 * 20 + 1 个元素
    (state)->u.narrow[2 * 20 + 1] = a04h;
    # 将变量 a14l 赋值给 state 结构体中 u.narrow 数组的第 2 * 21 + 0 个元素
    (state)->u.narrow[2 * 21 + 0] = a14l;
    # 将变量 a14h 赋值给 state 结构体中 u.narrow 数组的第 2 * 21 + 1 个元素
    (state)->u.narrow[2 * 21 + 1] = a14h;
    # 将变量 a24l 赋值给 state 结构体中 u.narrow 数组的第 2 * 22 + 0 个元素
    (state)->u.narrow[2 * 22 + 0] = a24l;
    # 将变量 a24h 赋值给 state 结构体中 u.narrow 数组的第 2 * 22 + 1 个元素
    (state)->u.narrow[2 * 22 + 1] = a24h;
    # 将 a34l 的值赋给 state 结构体中 u.narrow[2 * 23 + 0] 的位置
    (state)->u.narrow[2 * 23 + 0] = a34l;
    # 将 a34h 的值赋给 state 结构体中 u.narrow[2 * 23 + 1] 的位置
    (state)->u.narrow[2 * 23 + 1] = a34h;
    # 将 a44l 的值赋给 state 结构体中 u.narrow[2 * 24 + 0] 的位置
    (state)->u.narrow[2 * 24 + 0] = a44l;
    # 将 a44h 的值赋给 state 结构体中 u.narrow[2 * 24 + 1] 的位置
    (state)->u.narrow[2 * 24 + 1] = a44h;
# 定义一个宏，用于读取64位数据并进行处理
#define READ64(d, off)                                                         \
  do {                                                                         \
    # 读取低32位数据
    sph_u32 tl = sph_dec32le_aligned(buf + (off));                                     \
    # 读取高32位数据
    sph_u32 th = sph_dec32le_aligned(buf + (off) + 4);                                 \
    # 将低32位和高32位数据交错
    INTERLEAVE(tl, th);                                                        \
    # 将交错后的数据与d##l进行异或操作
    d##l ^= tl;                                                                \
    # 将交错后的数据与d##h进行异或操作
    d##h ^= th;                                                                \
  } while (0)

# 定义一个宏，用于读取输入缓冲区的前144字节数据
#define INPUT_BUF144                                                           \
  do {                                                                         \
    # 读取64位数据并进行处理
    READ64(a00, 0);                                                            \
    READ64(a10, 8);                                                            \
    READ64(a20, 16);                                                           \
    READ64(a30, 24);                                                           \
    READ64(a40, 32);                                                           \
    READ64(a01, 40);                                                           \
    READ64(a11, 48);                                                           \
    READ64(a21, 56);                                                           \
    READ64(a31, 64);                                                           \
    READ64(a41, 72);                                                           \
    READ64(a02, 80);                                                           \
    READ64(a12, 88);                                                           \
    READ64(a22, 96);                                                           \
    READ64(a32, 104);                                                          \
    // 从指定地址读取64位数据到变量a42
    READ64(a42, 112);
    // 从指定地址读取64位数据到变量a03
    READ64(a03, 120);
    // 从指定地址读取64位数据到变量a13
    READ64(a13, 128);
    // 从指定地址读取64位数据到变量a23
    READ64(a23, 136);
  } while (0)
# 定义一个宏，用于读取输入缓冲区中的136字节数据
#define INPUT_BUF136                                                           \
  do {                                                                         \
    # 依次读取64位数据到a00-a13变量中
    READ64(a00, 0);                                                            \
    READ64(a10, 8);                                                            \
    READ64(a20, 16);                                                           \
    READ64(a30, 24);                                                           \
    READ64(a40, 32);                                                           \
    READ64(a01, 40);                                                           \
    READ64(a11, 48);                                                           \
    READ64(a21, 56);                                                           \
    READ64(a31, 64);                                                           \
    READ64(a41, 72);                                                           \
    READ64(a02, 80);                                                           \
    READ64(a12, 88);                                                           \
    READ64(a22, 96);                                                           \
    READ64(a32, 104);                                                          \
    READ64(a42, 112);                                                          \
    READ64(a03, 120);                                                          \
    READ64(a13, 128);                                                          \
  } while (0)

# 定义一个宏，用于读取输入缓冲区中的104字节数据
#define INPUT_BUF104                                                           \
  do {                                                                         \
    # 依次读取64位数据到a00-a12变量中
    READ64(a00, 0);                                                            \
    READ64(a10, 8);                                                            \
    READ64(a20, 16);                                                           \
    # 读取64位数据到变量a30，偏移量为24
    READ64(a30, 24);
    # 读取64位数据到变量a40，偏移量为32
    READ64(a40, 32);
    # 读取64位数据到变量a01，偏移量为40
    READ64(a01, 40);
    # 读取64位数据到变量a11，偏移量为48
    READ64(a11, 48);
    # 读取64位数据到变量a21，偏移量为56
    READ64(a21, 56);
    # 读取64位数据到变量a31，偏移量为64
    READ64(a31, 64);
    # 读取64位数据到变量a41，偏移量为72
    READ64(a41, 72);
    # 读取64位数据到变量a02，偏移量为80
    READ64(a02, 80);
    # 读取64位数据到变量a12，偏移量为88
    READ64(a12, 88);
    # 读取64位数据到变量a22，偏移量为96
    READ64(a22, 96);
# 定义一个宏，用于读取输入缓冲区的前72个字节数据
#define INPUT_BUF72                                                            \
  do {                                                                         \
    READ64(a00, 0);                                                            # 读取64位数据到变量a00，偏移0字节
    READ64(a10, 8);                                                            # 读取64位数据到变量a10，偏移8字节
    READ64(a20, 16);                                                           # 读取64位数据到变量a20，偏移16字节
    READ64(a30, 24);                                                           # 读取64位数据到变量a30，偏移24字节
    READ64(a40, 32);                                                           # 读取64位数据到变量a40，偏移32字节
    READ64(a01, 40);                                                           # 读取64位数据到变量a01，偏移40字节
    READ64(a11, 48);                                                           # 读取64位数据到变量a11，偏移48字节
    READ64(a21, 56);                                                           # 读取64位数据到变量a21，偏移56字节
    READ64(a31, 64);                                                           # 读取64位数据到变量a31，偏移64字节
  } while (0)

# 定义一个宏，用于读取输入缓冲区的指定长度的数据
#define INPUT_BUF(lim)                                                         \
  do {                                                                         \
    READ64(a00, 0);                                                            # 读取64位数据到变量a00，偏移0字节
    READ64(a10, 8);                                                            # 读取64位数据到变量a10，偏移8字节
    READ64(a20, 16);                                                           # 读取64位数据到变量a20，偏移16字节
    READ64(a30, 24);                                                           # 读取64位数据到变量a30，偏移24字节
    READ64(a40, 32);                                                           # 读取64位数据到变量a40，偏移32字节
    READ64(a01, 40);                                                           # 读取64位数据到变量a01，偏移40字节
    READ64(a11, 48);                                                           # 读取64位数据到变量a11，偏移48字节
    READ64(a21, 56);                                                           # 读取64位数据到变量a21，偏移56字节
    READ64(a31, 64);                                                           # 读取64位数据到变量a31，偏移64字节
    if ((lim) == 72)                                                           # 如果指定长度为72字节
      break;                                                                   # 跳出循环
    # 从指定位置读取64位数据，并将其存储在变量a41中
    READ64(a41, 72);
    # 从指定位置读取64位数据，并将其存储在变量a02中
    READ64(a02, 80);
    # 从指定位置读取64位数据，并将其存储在变量a12中
    READ64(a12, 88);
    # 从指定位置读取64位数据，并将其存储在变量a22中
    READ64(a22, 96);
    # 如果lim等于104，则跳出循环
    if ((lim) == 104)
      break;
    # 从指定位置读取64位数据，并将其存储在变量a32中
    READ64(a32, 104);
    # 从指定位置读取64位数据，并将其存储在变量a42中
    READ64(a42, 112);
    # 从指定位置读取64位数据，并将其存储在变量a03中
    READ64(a03, 120);
    # 从指定位置读取64位数据，并将其存储在变量a13中
    READ64(a13, 128);
    # 如果lim等于136，则跳出循环
    if ((lim) == 136)
      break;
    # 从指定位置读取64位数据，并将其存储在变量a23中
    READ64(a23, 136);
#endif

// 定义一个宏，用于声明一个64位的变量
#define DECL64(x) sph_u64 x##l, x##h
// 定义一个宏，用于将一个64位的变量赋值给另一个64位的变量
#define MOV64(d, s) (d##l = s##l, d##h = s##h)
// 定义一个宏，用于将两个64位的变量进行按位异或操作，并将结果赋值给另一个64位的变量
#define XOR64(d, a, b) (d##l = a##l ^ b##l, d##h = a##h ^ b##h)
// 定义一个宏，用于将两个64位的变量进行按位与操作，并将结果赋值给另一个64位的变量
#define AND64(d, a, b) (d##l = a##l & b##l, d##h = a##h & b##h)
// 定义一个宏，用于将两个64位的变量进行按位或操作，并将结果赋值给另一个64位的变量
#define OR64(d, a, b) (d##l = a##l | b##l, d##h = a##h | b##h)
// 定义一个宏，用于将一个64位的变量进行按位取反操作，并将结果赋值给另一个64位的变量
#define NOT64(d, s) (d##l = SPH_T32(~s##l), d##h = SPH_T32(~s##h))
// 定义一个宏，用于将一个64位的变量进行循环左移操作，并将结果赋值给另一个64位的变量
#define ROL64(d, v, n) ROL64_##n(d, v)

#if SPH_KECCAK_INTERLEAVE

// 定义一个宏，用于将一个64位的变量进行奇数位数的循环左移操作，并将结果赋值给另一个64位的变量
#define ROL64_odd1(d, v)                                                       \
  do {                                                                         \
    sph_u32 tmp;                                                               \
    tmp = v##l;                                                                \
    d##l = SPH_T32(v##h << 1) | (v##h >> 31);                                  \
    d##h = tmp;                                                                \
  } while (0)

// 定义一个宏，用于将一个64位的变量进行奇数位数的循环左移操作，并将结果赋值给另一个64位的变量
#define ROL64_odd63(d, v)                                                      \
  do {                                                                         \
    sph_u32 tmp;                                                               \
    tmp = SPH_T32(v##l << 31) | (v##l >> 1);                                   \
    d##l = v##h;                                                               \
    d##h = tmp;                                                                \
  } while (0)

// 定义一个宏，用于将一个64位的变量进行奇数位数的循环左移操作，并将结果赋值给另一个64位的变量
#define ROL64_odd(d, v, n)                                                     \
  do {                                                                         \
    sph_u32 tmp;                                                               \
    tmp = SPH_T32(v##l << (n - 1)) | (v##l >> (33 - n));                       \
    d##l = SPH_T32(v##h << n) | (v##h >> (32 - n));                            \
    d##h = tmp;                                                                \
  } while (0)
#define ROL64_even(d, v, n)                                                    \  # 定义一个宏，用于将64位数v的每个32位部分循环左移n位，结果存储在d中
  do {                                                                         \  # 宏定义的开始
    d##l = SPH_T32(v##l << n) | (v##l >> (32 - n));                            \  # 将v的低32位左移n位，然后与v的低32位右移(32-n)位的结果进行或运算，存储在d的低32位中
    d##h = SPH_T32(v##h << n) | (v##h >> (32 - n));                            \  # 将v的高32位左移n位，然后与v的高32位右移(32-n)位的结果进行或运算，存储在d的高32位中
  } while (0)                                                                  \  # 宏定义的结束

#define ROL64_0(d, v)                                                            # 定义一个宏，无实际操作
#define ROL64_1(d, v) ROL64_odd1(d, v)                                           # 定义一个宏，调用ROL64_odd1宏
#define ROL64_2(d, v) ROL64_even(d, v, 1)                                        # 定义一个宏，调用ROL64_even宏
#define ROL64_3(d, v) ROL64_odd(d, v, 2)                                         # 定义一个宏，调用ROL64_odd宏
...                                                                              # 其余宏的定义与上述类似，分别调用不同的宏
#define ROL64_39(d, v) ROL64_odd(d, v, 20)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 20
#define ROL64_40(d, v) ROL64_even(d, v, 20)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 20
#define ROL64_41(d, v) ROL64_odd(d, v, 21)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 21
#define ROL64_42(d, v) ROL64_even(d, v, 21)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 21
#define ROL64_43(d, v) ROL64_odd(d, v, 22)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 22
#define ROL64_44(d, v) ROL64_even(d, v, 22)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 22
#define ROL64_45(d, v) ROL64_odd(d, v, 23)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 23
#define ROL64_46(d, v) ROL64_even(d, v, 23)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 23
#define ROL64_47(d, v) ROL64_odd(d, v, 24)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 24
#define ROL64_48(d, v) ROL64_even(d, v, 24)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 24
#define ROL64_49(d, v) ROL64_odd(d, v, 25)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 25
#define ROL64_50(d, v) ROL64_even(d, v, 25)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 25
#define ROL64_51(d, v) ROL64_odd(d, v, 26)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 26
#define ROL64_52(d, v) ROL64_even(d, v, 26)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 26
#define ROL64_53(d, v) ROL64_odd(d, v, 27)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 27
#define ROL64_54(d, v) ROL64_even(d, v, 27)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 27
#define ROL64_55(d, v) ROL64_odd(d, v, 28)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 28
#define ROL64_56(d, v) ROL64_even(d, v, 28)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 28
#define ROL64_57(d, v) ROL64_odd(d, v, 29)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 29
#define ROL64_58(d, v) ROL64_even(d, v, 29)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 29
#define ROL64_59(d, v) ROL64_odd(d, v, 30)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 30
#define ROL64_60(d, v) ROL64_even(d, v, 30)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 30
#define ROL64_61(d, v) ROL64_odd(d, v, 31)  // 定义一个宏，将参数传递给 ROL64_odd 函数，参数为 d, v, 31
#define ROL64_62(d, v) ROL64_even(d, v, 31)  // 定义一个宏，将参数传递给 ROL64_even 函数，参数为 d, v, 31
#define ROL64_63(d, v) ROL64_odd63(d, v)  // 定义一个宏，将参数传递给 ROL64_odd63 函数
#else

#define ROL64_small(d, v, n)                                                   \
  do {                                                                         \
    sph_u32 tmp;                                                               \
    tmp = SPH_T32(v##l << n) | (v##h >> (32 - n));                             \
    d##h = SPH_T32(v##h << n) | (v##l >> (32 - n));                            \
    d##l = tmp;                                                                \
  } while (0)

#define ROL64_0(d, v) 0  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 0
#define ROL64_1(d, v) ROL64_small(d, v, 1)  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 1
#define ROL64_2(d, v) ROL64_small(d, v, 2)  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 2
#define ROL64_3(d, v) ROL64_small(d, v, 3)  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 3
#define ROL64_4(d, v) ROL64_small(d, v, 4)  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 4
#define ROL64_5(d, v) ROL64_small(d, v, 5)  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 5
#define ROL64_6(d, v) ROL64_small(d, v, 6)  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 6
#define ROL64_7(d, v) ROL64_small(d, v, 7)  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 7
#define ROL64_8(d, v) ROL64_small(d, v, 8)  // 定义一个宏，将参数传递给 ROL64_small 函数，参数为 d, v, 8
# 定义一个宏，将给定的值按照指定的位数向左循环移动
# 参数d：目标变量
# 参数v：要移动的值
# 参数n：移动的位数
#define ROL64_small(d, v, n) ...

# 定义一系列宏，将给定的值按照指定的位数向左循环移动
# 参数d：目标变量
# 参数v：要移动的值
# 参数n：移动的位数
#define ROL64_9(d, v) ...
#define ROL64_10(d, v) ...
#define ROL64_11(d, v) ...
#define ROL64_12(d, v) ...
#define ROL64_13(d, v) ...
#define ROL64_14(d, v) ...
#define ROL64_15(d, v) ...
#define ROL64_16(d, v) ...
#define ROL64_17(d, v) ...
#define ROL64_18(d, v) ...
#define ROL64_19(d, v) ...
#define ROL64_20(d, v) ...
#define ROL64_21(d, v) ...
#define ROL64_22(d, v) ...
#define ROL64_23(d, v) ...
#define ROL64_24(d, v) ...
#define ROL64_25(d, v) ...
#define ROL64_26(d, v) ...
#define ROL64_27(d, v) ...
#define ROL64_28(d, v) ...
#define ROL64_29(d, v) ...
#define ROL64_30(d, v) ...
#define ROL64_31(d, v) ...

# 定义一个宏，将给定的值按照指定的位数向左循环移动
# 参数d：目标变量
# 参数v：要移动的值
# 参数n：移动的位数
#define ROL64_32(d, v) ...

# 定义一个宏，将给定的值按照指定的位数向左循环移动
# 参数d：目标变量
# 参数v：要移动的值
# 参数n：移动的位数
#define ROL64_big(d, v, n) ...
    # 这段代码是一个宏定义，用于将变量d##l的值设置为trh
    # 宏定义是一种在编译时进行文本替换的机制，这里的d##l是一个拼接的标识符
    # 该宏定义使用了do-while循环，目的是为了保证宏定义的语句块能够被当作一个整体使用
#define ROL64_33(d, v) ROL64_big(d, v, 1)
# 定义一个宏，用于将64位整数v左循环移位1位，结果存入d
#define ROL64_34(d, v) ROL64_big(d, v, 2)
# 定义一个宏，用于将64位整数v左循环移位2位，结果存入d
#define ROL64_35(d, v) ROL64_big(d, v, 3)
# 定义一个宏，用于将64位整数v左循环移位3位，结果存入d
#define ROL64_36(d, v) ROL64_big(d, v, 4)
# 定义一个宏，用于将64位整数v左循环移位4位，结果存入d
#define ROL64_37(d, v) ROL64_big(d, v, 5)
# 定义一个宏，用于将64位整数v左循环移位5位，结果存入d
#define ROL64_38(d, v) ROL64_big(d, v, 6)
# 定义一个宏，用于将64位整数v左循环移位6位，结果存入d
#define ROL64_39(d, v) ROL64_big(d, v, 7)
# 定义一个宏，用于将64位整数v左循环移位7位，结果存入d
#define ROL64_40(d, v) ROL64_big(d, v, 8)
# 定义一个宏，用于将64位整数v左循环移位8位，结果存入d
#define ROL64_41(d, v) ROL64_big(d, v, 9)
# 定义一个宏，用于将64位整数v左循环移位9位，结果存入d
#define ROL64_42(d, v) ROL64_big(d, v, 10)
# 定义一个宏，用于将64位整数v左循环移位10位，结果存入d
#define ROL64_43(d, v) ROL64_big(d, v, 11)
# 定义一个宏，用于将64位整数v左循环移位11位，结果存入d
#define ROL64_44(d, v) ROL64_big(d, v, 12)
# 定义一个宏，用于将64位整数v左循环移位12位，结果存入d
#define ROL64_45(d, v) ROL64_big(d, v, 13)
# 定义一个宏，用于将64位整数v左循环移位13位，结果存入d
#define ROL64_46(d, v) ROL64_big(d, v, 14)
# 定义一个宏，用于将64位整数v左循环移位14位，结果存入d
#define ROL64_47(d, v) ROL64_big(d, v, 15)
# 定义一个宏，用于将64位整数v左循环移位15位，结果存入d
#define ROL64_48(d, v) ROL64_big(d, v, 16)
# 定义一个宏，用于将64位整数v左循环移位16位，结果存入d
#define ROL64_49(d, v) ROL64_big(d, v, 17)
# 定义一个宏，用于将64位整数v左循环移位17位，结果存入d
#define ROL64_50(d, v) ROL64_big(d, v, 18)
# 定义一个宏，用于将64位整数v左循环移位18位，结果存入d
#define ROL64_51(d, v) ROL64_big(d, v, 19)
# 定义一个宏，用于将64位整数v左循环移位19位，结果存入d
#define ROL64_52(d, v) ROL64_big(d, v, 20)
# 定义一个宏，用于将64位整数v左循环移位20位，结果存入d
#define ROL64_53(d, v) ROL64_big(d, v, 21)
# 定义一个宏，用于将64位整数v左循环移位21位，结果存入d
#define ROL64_54(d, v) ROL64_big(d, v, 22)
# 定义一个宏，用于将64位整数v左循环移位22位，结果存入d
#define ROL64_55(d, v) ROL64_big(d, v, 23)
# 定义一个宏，用于将64位整数v左循环移位23位，结果存入d
#define ROL64_56(d, v) ROL64_big(d, v, 24)
# 定义一个宏，用于将64位整数v左循环移位24位，结果存入d
#define ROL64_57(d, v) ROL64_big(d, v, 25)
# 定义一个宏，用于将64位整数v左循环移位25位，结果存入d
#define ROL64_58(d, v) ROL64_big(d, v, 26)
# 定义一个宏，用于将64位整数v左循环移位26位，结果存入d
#define ROL64_59(d, v) ROL64_big(d, v, 27)
# 定义一个宏，用于将64位整数v左循环移位27位，结果存入d
#define ROL64_60(d, v) ROL64_big(d, v, 28)
# 定义一个宏，用于将64位整数v左循环移位28位，结果存入d
#define ROL64_61(d, v) ROL64_big(d, v, 29)
# 定义一个宏，用于将64位整数v左循环移位29位，结果存入d
#define ROL64_62(d, v) ROL64_big(d, v, 30)
# 定义一个宏，用于将64位整数v左循环移位30位，结果存入d
#define ROL64_63(d, v) ROL64_big(d, v, 31)
# 定义一个宏，用于将64位整数v左循环移位31位，结果存入d
#endif
# 结束条件编译

#define XOR64_IOTA(d, s, k) (d##l = s##l ^ k.low, d##h = s##h ^ k.high)
# 定义一个宏，用于将64位整数s和64位整数k进行按位异或，结果存入64位整数d

#define TH_ELT(t, c0, c1, c2, c3, c4, d0, d1, d2, d3, d4)                      \
  do {                                                                         \
    DECL64(tt0);                                                               \
    DECL64(tt1);                                                               \
    DECL64(tt2);                                                               \
    DECL64(tt3);                                                               \
    XOR64(tt0, d0, d1);
# 定义一个宏，用于将64位整数d0和64位整数d1进行按位异或，结果存入64位整数tt0
    # 对输入的四个64位数据进行按位异或操作
    XOR64(tt1, d2, d3);                                                    
    XOR64(tt0, tt0, d4);                                                   
    XOR64(tt0, tt0, tt1);                                                  
    # 对tt0进行循环左移1位
    ROL64(tt0, tt0, 1);                                                    
    XOR64(tt2, c0, c1);                                                    
    XOR64(tt3, c2, c3);                                                    
    XOR64(tt0, tt0, c4);                                                   
    XOR64(tt2, tt2, tt3);                                                  
    # 对tt0和tt2进行按位异或操作
    XOR64(t, tt0, tt2);                                                      
  } while (0)
# 定义一个宏函数，用于计算 THETA 变换
#define THETA(b00, b01, b02, b03, b04, b10, b11, b12, b13, b14, b20, b21, b22, \
              b23, b24, b30, b31, b32, b33, b34, b40, b41, b42, b43, b44)      \
  do {                                                                         \
    # 声明 64 位整数变量
    DECL64(t0);                                                                \
    DECL64(t1);                                                                \
    DECL64(t2);                                                                \
    DECL64(t3);                                                                \
    DECL64(t4);                                                                \
    # 计算 TH_ELT 变换
    TH_ELT(t0, b40, b41, b42, b43, b44, b10, b11, b12, b13, b14);              \
    TH_ELT(t1, b00, b01, b02, b03, b04, b20, b21, b22, b23, b24);              \
    TH_ELT(t2, b10, b11, b12, b13, b14, b30, b31, b32, b33, b34);              \
    TH_ELT(t3, b20, b21, b22, b23, b24, b40, b41, b42, b43, b44);              \
    TH_ELT(t4, b30, b31, b32, b33, b34, b00, b01, b02, b03, b04);              \
    # 对 b00 到 b04 进行异或运算
    XOR64(b00, b00, t0);                                                       \
    XOR64(b01, b01, t0);                                                       \
    XOR64(b02, b02, t0);                                                       \
    XOR64(b03, b03, t0);                                                       \
    XOR64(b04, b04, t0);                                                       \
    # 对 b10 到 b14 进行异或运算
    XOR64(b10, b10, t1);                                                       \
    XOR64(b11, b11, t1);                                                       \
    XOR64(b12, b12, t1);                                                       \
    XOR64(b13, b13, t1);                                                       \
    XOR64(b14, b14, t1);                                                       \
    # 对 b20 到 b24 进行异或运算
    XOR64(b20, b20, t2);                                                       \
    # 对 b21 到 b44 进行按位异或操作，结果存储在对应的变量中
    XOR64(b21, b21, t2);                                                       
    XOR64(b22, b22, t2);                                                       
    XOR64(b23, b23, t2);                                                       
    XOR64(b24, b24, t2);                                                       
    XOR64(b30, b30, t3);                                                       
    XOR64(b31, b31, t3);                                                       
    XOR64(b32, b32, t3);                                                       
    XOR64(b33, b33, t3);                                                       
    XOR64(b34, b34, t3);                                                       
    XOR64(b40, b40, t4);                                                       
    XOR64(b41, b41, t4);                                                       
    XOR64(b42, b42, t4);                                                       
    XOR64(b43, b43, t4);                                                       
    XOR64(b44, b44, t4);                                                       
  } while (0)
# 定义一个宏函数，用于对输入的25个变量进行循环左移操作
#define RHO(b00, b01, b02, b03, b04, b10, b11, b12, b13, b14, b20, b21, b22,   \
            b23, b24, b30, b31, b32, b33, b34, b40, b41, b42, b43, b44)        \
  do {                                                                         \
    # 对变量b01进行36位循环左移操作
    ROL64(b01, b01, 36);                                                       \
    # 对变量b02进行3位循环左移操作
    ROL64(b02, b02, 3);                                                        \
    # 对变量b03进行41位循环左移操作
    ROL64(b03, b03, 41);                                                       \
    # 对变量b04进行18位循环左移操作
    ROL64(b04, b04, 18);                                                       \
    # 对变量b10进行1位循环左移操作
    ROL64(b10, b10, 1);                                                        \
    # 对变量b11进行44位循环左移操作
    ROL64(b11, b11, 44);                                                       \
    # 对变量b12进行10位循环左移操作
    ROL64(b12, b12, 10);                                                       \
    # 对变量b13进行45位循环左移操作
    ROL64(b13, b13, 45);                                                       \
    # 对变量b14进行2位循环左移操作
    ROL64(b14, b14, 2);                                                        \
    # 对变量b20进行62位循环左移操作
    ROL64(b20, b20, 62);                                                       \
    # 对变量b21进行6位循环左移操作
    ROL64(b21, b21, 6);                                                        \
    # 对变量b22进行43位循环左移操作
    ROL64(b22, b22, 43);                                                       \
    # 对变量b23进行15位循环左移操作
    ROL64(b23, b23, 15);                                                       \
    # 对变量b24进行61位循环左移操作
    ROL64(b24, b24, 61);                                                       \
    # 对变量b30进行28位循环左移操作
    ROL64(b30, b30, 28);                                                       \
    # 对变量b31进行55位循环左移操作
    ROL64(b31, b31, 55);                                                       \
    # 对变量b32进行25位循环左移操作
    ROL64(b32, b32, 25);                                                       \
    # 对变量b33进行21位循环左移操作
    ROL64(b33, b33, 21);                                                       \
    # 对变量b34进行56位循环左移操作
    ROL64(b34, b34, 56);                                                       \
    # 对变量b40进行27位循环左移操作
    ROL64(b40, b40, 27);                                                       \
    # 对 b41 进行 64 位循环左移 20 位
    ROL64(b41, b41, 20);                                                       \
    # 对 b42 进行 64 位循环左移 39 位
    ROL64(b42, b42, 39);                                                       \
    # 对 b43 进行 64 位循环左移 8 位
    ROL64(b43, b43, 8);                                                        \
    # 对 b44 进行 64 位循环左移 14 位
    ROL64(b44, b44, 14);                                                       \
  } while (0)
/*
 * 宏 KHI_XO 实现“lane complement”优化，对输入的一些字进行补码操作
 * 输入时，以下字进行补码操作：
 *    a00 a01 a02 a04 a13 a20 a21 a22 a30 a33 a34 a43
 * 输出时，以下字进行补码操作：
 *    a04 a10 a20 a22 a23 a31
 *
 * 隐式的置换和θ扩展将为下一轮带回输入掩码
 */

#define KHI_XO(d, a, b, c)                                                     \
  do {                                                                         \
    DECL64(kt);                                                                \
    OR64(kt, b, c);                                                            \
    XOR64(d, a, kt);                                                           \
  } while (0)

#define KHI_XA(d, a, b, c)                                                     \
  do {                                                                         \
    DECL64(kt);                                                                \
    AND64(kt, b, c);                                                           \
    XOR64(d, a, kt);                                                           \
  } while (0)

#define KHI(b00, b01, b02, b03, b04, b10, b11, b12, b13, b14, b20, b21, b22,   \
            b23, b24, b30, b31, b32, b33, b34, b40, b41, b42, b43, b44)        \
  do {                                                                         \
    DECL64(c0);                                                                \
    DECL64(c1);                                                                \
    DECL64(c2);                                                                \
    DECL64(c3);                                                                \
    DECL64(c4);                                                                \
    DECL64(bnn);                                                               \
    # 对 bnn 和 b20 进行逻辑非操作
    NOT64(bnn, b20);                                                           \
    # 使用 b00, b10, b20 进行 KHI_XO 操作，结果存入 c0
    KHI_XO(c0, b00, b10, b20);                                                 \
    # 使用 b10, bnn, b30 进行 KHI_XO 操作，结果存入 c1
    KHI_XO(c1, b10, bnn, b30);                                                 \
    # 使用 b20, b30, b40 进行 KHI_XA 操作，结果存入 c2
    KHI_XA(c2, b20, b30, b40);                                                 \
    # 使用 b30, b40, b00 进行 KHI_XO 操作，结果存入 c3
    KHI_XO(c3, b30, b40, b00);                                                 \
    # 使用 b40, b00, b10 进行 KHI_XA 操作，结果存入 c4
    KHI_XA(c4, b40, b00, b10);                                                 \
    # 将 c0 的值赋给 b00
    MOV64(b00, c0);                                                            \
    # 将 c1 的值赋给 b10
    MOV64(b10, c1);                                                            \
    # 将 c2 的值赋给 b20
    MOV64(b20, c2);                                                            \
    # 将 c3 的值赋给 b30
    MOV64(b30, c3);                                                            \
    # 将 c4 的值赋给 b40
    MOV64(b40, c4);                                                            \
    # 对 bnn 和 b41 进行逻辑非操作
    NOT64(bnn, b41);                                                           \
    # 使用 b01, b11, b21 进行 KHI_XO 操作，结果存入 c0
    KHI_XO(c0, b01, b11, b21);                                                 \
    # 使用 b11, b21, b31 进行 KHI_XA 操作，结果存入 c1
    KHI_XA(c1, b11, b21, b31);                                                 \
    # 使用 b21, b31, bnn 进行 KHI_XO 操作，结果存入 c2
    KHI_XO(c2, b21, b31, bnn);                                                 \
    # 使用 b31, b41, b01 进行 KHI_XO 操作，结果存入 c3
    KHI_XO(c3, b31, b41, b01);                                                 \
    # 使用 b41, b01, b11 进行 KHI_XA 操作，结果存入 c4
    KHI_XA(c4, b41, b01, b11);                                                 \
    # 将 c0 的值赋给 b01
    MOV64(b01, c0);                                                            \
    # 将 c1 的值赋给 b11
    MOV64(b11, c1);                                                            \
    # 将 c2 的值赋给 b21
    MOV64(b21, c2);                                                            \
    # 将 c3 的值赋给 b31
    MOV64(b31, c3);                                                            \
    # 将 c4 的值赋给 b41
    MOV64(b41, c4);                                                            \
    # 对 bnn 和 b32 进行逻辑非操作
    NOT64(bnn, b32);                                                           \
    # 使用 b02, b12, b22 进行 KHI_XO 操作，结果存入 c0
    KHI_XO(c0, b02, b12, b22);                                                 \
    # 使用 KHI_XA 函数对 c1, b12, b22, b32 进行操作
    KHI_XA(c1, b12, b22, b32);
    # 使用 KHI_XA 函数对 c2, b22, bnn, b42 进行操作
    KHI_XA(c2, b22, bnn, b42);
    # 使用 KHI_XO 函数对 c3, bnn, b42, b02 进行操作
    KHI_XO(c3, bnn, b42, b02);
    # 使用 KHI_XA 函数对 c4, b42, b02, b12 进行操作
    KHI_XA(c4, b42, b02, b12);
    # 将 c0 的值赋给 b02
    MOV64(b02, c0);
    # 将 c1 的值赋给 b12
    MOV64(b12, c1);
    # 将 c2 的值赋给 b22
    MOV64(b22, c2);
    # 将 c3 的值赋给 b32
    MOV64(b32, c3);
    # 将 c4 的值赋给 b42
    MOV64(b42, c4);
    # 对 b33 取反，并将结果赋给 bnn
    NOT64(bnn, b33);
    # 使用 KHI_XA 函数对 c0, b03, b13, b23 进行操作
    KHI_XA(c0, b03, b13, b23);
    # 使用 KHI_XO 函数对 c1, b13, b23, b33 进行操作
    KHI_XO(c1, b13, b23, b33);
    # 使用 KHI_XO 函数对 c2, b23, bnn, b43 进行操作
    KHI_XO(c2, b23, bnn, b43);
    # 使用 KHI_XA 函数对 c3, bnn, b43, b03 进行操作
    KHI_XA(c3, bnn, b43, b03);
    # 使用 KHI_XO 函数对 c4, b43, b03, b13 进行操作
    KHI_XO(c4, b43, b03, b13);
    # 将 c0 的值赋给 b03
    MOV64(b03, c0);
    # 将 c1 的值赋给 b13
    MOV64(b13, c1);
    # 将 c2 的值赋给 b23
    MOV64(b23, c2);
    # 将 c3 的值赋给 b33
    MOV64(b33, c3);
    # 将 c4 的值赋给 b43
    MOV64(b43, c4);
    # 对 b14 取反，并将结果赋给 bnn
    NOT64(bnn, b14);
    # 使用 KHI_XA 函数对 c0, b04, bnn, b24 进行操作
    KHI_XA(c0, b04, bnn, b24);
    # 使用 KHI_XO 函数对 c1, bnn, b24, b34 进行操作
    KHI_XO(c1, bnn, b24, b34);
    # 使用 KHI_XA 函数对 c2, b24, b34, b44 进行操作
    KHI_XA(c2, b24, b34, b44);
    # 调用 KHI_XO 函数，传入参数 c3, b34, b44, b04
    KHI_XO(c3, b34, b44, b04);                                                 \
    # 调用 KHI_XA 函数，传入参数 c4, b44, b04, b14
    KHI_XA(c4, b44, b04, b14);                                                 \
    # 将 c0 的值赋给 b04
    MOV64(b04, c0);                                                            \
    # 将 c1 的值赋给 b14
    MOV64(b14, c1);                                                            \
    # 将 c2 的值赋给 b24
    MOV64(b24, c2);                                                            \
    # 将 c3 的值赋给 b34
    MOV64(b34, c3);                                                            \
    # 将 c4 的值赋给 b44
    MOV64(b44, c4);                                                            \
  } while (0)
# 定义宏，用于执行IOTA操作
#define IOTA(r) XOR64_IOTA(a00, a00, r)

# 定义8个置换操作P0-P8，每个操作包含不同的置换顺序
#define P0                                                                     \
  a00, a01, a02, a03, a04, a10, a11, a12, a13, a14, a20, a21, a22, a23, a24,   \
      a30, a31, a32, a33, a34, a40, a41, a42, a43, a44
#define P1                                                                     \
  a00, a30, a10, a40, a20, a11, a41, a21, a01, a31, a22, a02, a32, a12, a42,   \
      a33, a13, a43, a23, a03, a44, a24, a04, a34, a14
#define P2                                                                     \
  a00, a33, a11, a44, a22, a41, a24, a02, a30, a13, a32, a10, a43, a21, a04,   \
      a23, a01, a34, a12, a40, a14, a42, a20, a03, a31
#define P3                                                                     \
  a00, a23, a41, a14, a32, a24, a42, a10, a33, a01, a43, a11, a34, a02, a20,   \
      a12, a30, a03, a21, a44, a31, a04, a22, a40, a13
#define P4                                                                     \
  a00, a12, a24, a31, a43, a42, a04, a11, a23, a30, a34, a41, a03, a10, a22,   \
      a21, a33, a40, a02, a14, a13, a20, a32, a44, a01
#define P5                                                                     \
  a00, a21, a42, a13, a34, a04, a20, a41, a12, a33, a03, a24, a40, a11, a32,   \
      a02, a23, a44, a10, a31, a01, a22, a43, a14, a30
#define P6                                                                     \
  a00, a02, a04, a01, a03, a20, a22, a24, a21, a23, a40, a42, a44, a41, a43,   \
      a10, a12, a14, a11, a13, a30, a32, a34, a31, a33
#define P7                                                                     \
  a00, a10, a20, a30, a40, a22, a32, a42, a02, a12, a44, a04, a14, a24, a34,   \
      a11, a21, a31, a41, a01, a33, a43, a03, a13, a23
#define P8                                                                     \
  a00, a11, a22, a33, a44, a32, a43, a04, a10, a21, a14, a20, a31, a42, a03,   \
      a41, a02, a13, a24, a30, a23, a34, a40, a01, a12
# 定义置换表 P9，用于将输入的 9 位二进制数重新排列
#define P9                                                                     \
  a00, a41, a32, a23, a14, a43, a34, a20, a11, a02, a31, a22, a13, a04, a40,   \
      a24, a10, a01, a42, a33, a12, a03, a44, a30, a21

# 定义置换表 P10，用于将输入的 10 位二进制数重新排列
#define P10                                                                    \
  a00, a24, a43, a12, a31, a34, a03, a22, a41, a10, a13, a32, a01, a20, a44,   \
      a42, a11, a30, a04, a23, a21, a40, a14, a33, a02

# 定义置换表 P11，用于将输入的 11 位二进制数重新排列
#define P11                                                                    \
  a00, a42, a34, a21, a13, a03, a40, a32, a24, a11, a01, a43, a30, a22, a14,   \
      a04, a41, a33, a20, a12, a02, a44, a31, a23, a10

# 定义置换表 P12，用于将输入的 12 位二进制数重新排列
#define P12                                                                    \
  a00, a04, a03, a02, a01, a40, a44, a43, a42, a41, a30, a34, a33, a32, a31,   \
      a20, a24, a23, a22, a21, a10, a14, a13, a12, a11

# 定义置换表 P13，用于将输入的 13 位二进制数重新排列
#define P13                                                                    \
  a00, a20, a40, a10, a30, a44, a14, a34, a04, a24, a33, a03, a23, a43, a13,   \
      a22, a42, a12, a32, a02, a11, a31, a01, a21, a41

# 定义置换表 P14，用于将输入的 14 位二进制数重新排列
#define P14                                                                    \
  a00, a22, a44, a11, a33, a14, a31, a03, a20, a42, a23, a40, a12, a34, a01,   \
      a32, a04, a21, a43, a10, a41, a13, a30, a02, a24

# 定义置换表 P15，用于将输入的 15 位二进制数重新排列
#define P15                                                                    \
  a00, a32, a14, a41, a23, a31, a13, a40, a22, a04, a12, a44, a21, a03, a30,   \
      a43, a20, a02, a34, a11, a24, a01, a33, a10, a42

# 定义置换表 P16，用于将输入的 16 位二进制数重新排列
#define P16                                                                    \
  a00, a43, a31, a24, a12, a13, a01, a44, a32, a20, a21, a14, a02, a40, a33,   \
      a34, a22, a10, a03, a41, a42, a30, a23, a11, a04

# 定义置换表 P17，用于将输入的 17 位二进制数重新排列
#define P17                                                                    \
  a00, a34, a13, a42, a21, a01, a30, a14, a43, a22, a02, a31, a10, a44, a23,   \
      a03, a32, a11, a40, a24, a04, a33, a12, a41, a20
#define P18                                                                    \
  a00, a03, a01, a04, a02, a30, a33, a31, a34, a32, a10, a13, a11, a14, a12,   \
      a40, a43, a41, a44, a42, a20, a23, a21, a24, a22
# 定义宏 P18，用于将变量重新排列成指定顺序的列表

#define P19                                                                    \
  a00, a40, a30, a20, a10, a33, a23, a13, a03, a43, a11, a01, a41, a31, a21,   \
      a44, a34, a24, a14, a04, a22, a12, a02, a42, a32
# 定义宏 P19，用于将变量重新排列成指定顺序的列表

#define P20                                                                    \
  a00, a44, a33, a22, a11, a23, a12, a01, a40, a34, a41, a30, a24, a13, a02,   \
      a14, a03, a42, a31, a20, a32, a21, a10, a04, a43
# 定义宏 P20，用于将变量重新排列成指定顺序的列表

#define P21                                                                    \
  a00, a14, a23, a32, a41, a12, a21, a30, a44, a03, a24, a33, a42, a01, a10,   \
      a31, a40, a04, a13, a22, a43, a02, a11, a20, a34
# 定义宏 P21，用于将变量重新排列成指定顺序的列表

#define P22                                                                    \
  a00, a31, a12, a43, a24, a21, a02, a33, a14, a40, a42, a23, a04, a30, a11,   \
      a13, a44, a20, a01, a32, a34, a10, a41, a22, a03
# 定义宏 P22，用于将变量重新排列成指定顺序的列表

#define P23                                                                    \
  a00, a13, a21, a34, a42, a02, a10, a23, a31, a44, a04, a12, a20, a33, a41,   \
      a01, a14, a22, a30, a43, a03, a11, a24, a32, a40
# 定义宏 P23，用于将变量重新排列成指定顺序的列表

#define P1_TO_P0                                                               \
  do {                                                                         \
    DECL64(t);                                                                 \
    MOV64(t, a01);                                                             \
    MOV64(a01, a30);                                                           \
    MOV64(a30, a33);                                                           \
    MOV64(a33, a23);                                                           \
    MOV64(a23, a12);                                                           \
# 定义宏 P1_TO_P0，用于将变量按照指定顺序进行赋值
    # 将寄存器 a12 的值移动到寄存器 a21
    MOV64(a12, a21);                                                           
    # 将寄存器 a21 的值移动到寄存器 a02
    MOV64(a21, a02);                                                           
    # 将寄存器 a02 的值移动到寄存器 a10
    MOV64(a02, a10);                                                           
    # 将寄存器 a10 的值移动到寄存器 a11
    MOV64(a10, a11);                                                           
    # 将寄存器 a11 的值移动到寄存器 a41
    MOV64(a11, a41);                                                           
    # 将寄存器 a41 的值移动到寄存器 a24
    MOV64(a41, a24);                                                           
    # 将寄存器 a24 的值移动到寄存器 a42
    MOV64(a24, a42);                                                           
    # 将寄存器 a42 的值移动到寄存器 a04
    MOV64(a42, a04);                                                           
    # 将寄存器 a04 的值移动到寄存器 a20
    MOV64(a04, a20);                                                           
    # 将寄存器 a20 的值移动到寄存器 a22
    MOV64(a20, a22);                                                           
    # 将寄存器 a22 的值移动到寄存器 a32
    MOV64(a22, a32);                                                           
    # 将寄存器 a32 的值移动到寄存器 a43
    MOV64(a32, a43);                                                           
    # 将寄存器 a43 的值移动到寄存器 a34
    MOV64(a43, a34);                                                           
    # 将寄存器 a34 的值移动到寄存器 a03
    MOV64(a34, a03);                                                           
    # 将寄存器 a03 的值移动到寄存器 a40
    MOV64(a03, a40);                                                           
    # 将寄存器 a40 的值移动到寄存器 a44
    MOV64(a40, a44);                                                           
    # 将寄存器 a44 的值移动到寄存器 a14
    MOV64(a44, a14);                                                           
    # 将寄存器 a14 的值移动到寄存器 a31
    MOV64(a14, a31);                                                           
    # 将寄存器 a31 的值移动到寄存器 a13
    MOV64(a31, a13);                                                           
    # 将寄存器 a13 的值移动到寄存器 t
    MOV64(a13, t);                                                             
  } while (0)
# 定义宏 P2_TO_P0，用于将变量的值按照指定的顺序进行交换
do {
    # 声明一个64位整数变量 t
    DECL64(t);
    # 依次交换变量的值
    MOV64(t, a01);
    MOV64(a01, a33);
    MOV64(a33, a12);
    MOV64(a12, a02);
    MOV64(a02, a11);
    MOV64(a11, a24);
    MOV64(a24, a04);
    MOV64(a04, a22);
    MOV64(a22, a43);
    MOV64(a43, a03);
    MOV64(a03, a44);
    MOV64(a44, a31);
    MOV64(a31, t);
    MOV64(t, a10);
    MOV64(a10, a41);
    MOV64(a41, a42);
    MOV64(a42, a20);
    MOV64(a20, a32);
    MOV64(a32, a34);
    MOV64(a34, a40);
    MOV64(a40, a14);
    # 将 a14 的值赋给 a13
    MOV64(a14, a13);
    # 将 a13 的值赋给 a30
    MOV64(a13, a30);
    # 将 a30 的值赋给 a23
    MOV64(a30, a23);
    # 将 a23 的值赋给 a21
    MOV64(a23, a21);
    # 将 a21 的值赋给 t
    MOV64(a21, t);
# 定义一个宏，将变量a01到a23的值按照特定顺序进行交换
#define P4_TO_P0                                                               \
  do {                                                                         \
    # 定义一个临时变量t，用于存储交换过程中的中间值
    DECL64(t);                                                                 \
    # 交换a01和a12的值
    MOV64(t, a01);                                                             \
    MOV64(a01, a12);                                                           \
    MOV64(a12, a11);                                                           \
    MOV64(a11, a04);                                                           \
    MOV64(a04, a43);                                                           \
    MOV64(a43, a44);                                                           \
    MOV64(a44, t);                                                             \
    MOV64(t, a02);                                                             \
    MOV64(a02, a24);                                                           \
    MOV64(a24, a22);                                                           \
    MOV64(a22, a03);                                                           \
    MOV64(a03, a31);                                                           \
    MOV64(a31, a33);                                                           \
    MOV64(a33, t);                                                             \
    MOV64(t, a10);                                                             \
    MOV64(a10, a42);                                                           \
    MOV64(a42, a32);                                                           \
    MOV64(a32, a40);                                                           \
    MOV64(a40, a13);                                                           \
    MOV64(a13, a23);                                                           \
    MOV64(a23, t);                                                             \
    # 将寄存器 a14 的值移动到寄存器 t
    MOV64(t, a14);                                                             
    # 将寄存器 a30 的值移动到寄存器 a14
    MOV64(a14, a30);                                                           
    # 将寄存器 a21 的值移动到寄存器 a30
    MOV64(a30, a21);                                                           
    # 将寄存器 a41 的值移动到寄存器 a21
    MOV64(a21, a41);                                                           
    # 将寄存器 a20 的值移动到寄存器 a41
    MOV64(a41, a20);                                                           
    # 将寄存器 a34 的值移动到寄存器 a20
    MOV64(a20, a34);                                                           
    # 将寄存器 t 的值移动到寄存器 a34
    MOV64(a34, t);                                                             
  } while (0)
# 定义一个宏，将变量a01、a02、a04、a03、t的值互相交换
#define P6_TO_P0                                                               \
  do {                                                                         \
    # 临时变量t用于存储a01的值
    DECL64(t);                                                                 \
    # 将a01的值赋给a02
    MOV64(t, a01);                                                             \
    # 将a02的值赋给a04
    MOV64(a01, a02);                                                           \
    # 将a04的值赋给a03
    MOV64(a02, a04);                                                           \
    # 将a03的值赋给a04
    MOV64(a04, a03);                                                           \
    # 将t的值赋给a03
    MOV64(a03, t);                                                             \
    # 将a10的值赋给t
    MOV64(t, a10);                                                             \
    # 将a20的值赋给a10
    MOV64(a10, a20);                                                           \
    # 将a40的值赋给a20
    MOV64(a20, a40);                                                           \
    # 将a30的值赋给a40
    MOV64(a40, a30);                                                           \
    # 将t的值赋给a30
    MOV64(a30, t);                                                             \
    # 将a11的值赋给t
    MOV64(t, a11);                                                             \
    # 将a22的值赋给a11
    MOV64(a11, a22);                                                           \
    # 将a44的值赋给a22
    MOV64(a22, a44);                                                           \
    # 将a33的值赋给a44
    MOV64(a44, a33);                                                           \
    # 将t的值赋给a33
    MOV64(a33, t);                                                             \
    # 将a12的值赋给t
    MOV64(t, a12);                                                             \
    # 将a24的值赋给a12
    MOV64(a12, a24);                                                           \
    # 将a43的值赋给a24
    MOV64(a24, a43);                                                           \
    # 将a31的值赋给a43
    MOV64(a43, a31);                                                           \
    # 将t的值赋给a31
    MOV64(a31, t);                                                             \
    # 将a13的值赋给t
    MOV64(t, a13);                                                             \
    # 将寄存器 a13 的值移动到寄存器 a21
    MOV64(a13, a21);                                                           \
    # 将寄存器 a21 的值移动到寄存器 a42
    MOV64(a21, a42);                                                           \
    # 将寄存器 a42 的值移动到寄存器 a34
    MOV64(a42, a34);                                                           \
    # 将寄存器 a34 的值移动到寄存器 t
    MOV64(a34, t);                                                             \
    # 将寄存器 t 的值移动到寄存器 a14
    MOV64(t, a14);                                                             \
    # 将寄存器 a14 的值移动到寄存器 a23
    MOV64(a14, a23);                                                           \
    # 将寄存器 a23 的值移动到寄存器 a41
    MOV64(a23, a41);                                                           \
    # 将寄存器 a41 的值移动到寄存器 a32
    MOV64(a41, a32);                                                           \
    # 将寄存器 a32 的值移动到寄存器 t
    MOV64(a32, t);                                                             \
  } while (0)
# 定义一个宏，用于交换变量的值
#define P8_TO_P0                                                               \
  do {                                                                         \
    # 临时变量t用于存储a01的值
    DECL64(t);                                                                 \
    # 交换变量的值
    MOV64(t, a01);                                                             \
    MOV64(a01, a11);                                                           \
    MOV64(a11, a43);                                                           \
    MOV64(a43, t);                                                             \
    MOV64(t, a02);                                                             \
    MOV64(a02, a22);                                                           \
    MOV64(a22, a31);                                                           \
    MOV64(a31, t);                                                             \
    MOV64(t, a03);                                                             \
    MOV64(a03, a33);                                                           \
    MOV64(a33, a24);                                                           \
    MOV64(a24, t);                                                             \
    MOV64(t, a04);                                                             \
    MOV64(a04, a44);                                                           \
    MOV64(a44, a12);                                                           \
    MOV64(a12, t);                                                             \
    MOV64(t, a10);                                                             \
    MOV64(a10, a32);                                                           \
    MOV64(a32, a13);                                                           \
    MOV64(a13, t);                                                             \
    MOV64(t, a14);                                                             \
  # 结束宏定义
  \```
    # 将寄存器 a21 的值移动到寄存器 a14
    MOV64(a14, a21);                                                           \
    # 将寄存器 a20 的值移动到寄存器 a21
    MOV64(a21, a20);                                                           \
    # 将寄存器 t 的值移动到寄存器 a20
    MOV64(a20, t);                                                             \
    # 将寄存器 a23 的值移动到寄存器 t
    MOV64(t, a23);                                                             \
    # 将寄存器 a42 的值移动到寄存器 a23
    MOV64(a23, a42);                                                           \
    # 将寄存器 a40 的值移动到寄存器 a42
    MOV64(a42, a40);                                                           \
    # 将寄存器 t 的值移动到寄存器 a40
    MOV64(a40, t);                                                             \
    # 将寄存器 a30 的值移动到寄存器 t
    MOV64(t, a30);                                                             \
    # 将寄存器 a41 的值移动到寄存器 a30
    MOV64(a30, a41);                                                           \
    # 将寄存器 a34 的值移动到寄存器 a41
    MOV64(a41, a34);                                                           \
    # 将寄存器 t 的值移动到寄存器 a34
    MOV64(a34, t);                                                             \
  } while (0)
# 定义一个宏，将变量a01、a04、a02、a03、a10、a40、a11、a44、a12、a43、a13、a42、a14、a41的值两两交换
#define P12_TO_P0                                                              \
  do {                                                                         \
    # 定义一个临时变量t，用于交换变量的值
    DECL64(t);                                                                 \
    # 交换a01和a04的值
    MOV64(t, a01);                                                             \
    MOV64(a01, a04);                                                           \
    MOV64(a04, t);                                                             \
    # 交换a02和a03的值
    MOV64(t, a02);                                                             \
    MOV64(a02, a03);                                                           \
    MOV64(a03, t);                                                             \
    # 交换a10和a40的值
    MOV64(t, a10);                                                             \
    MOV64(a10, a40);                                                           \
    MOV64(a40, t);                                                             \
    # 交换a11和a44的值
    MOV64(t, a11);                                                             \
    MOV64(a11, a44);                                                           \
    MOV64(a44, t);                                                             \
    # 交换a12和a43的值
    MOV64(t, a12);                                                             \
    MOV64(a12, a43);                                                           \
    MOV64(a43, t);                                                             \
    # 交换a13和a42的值
    MOV64(t, a13);                                                             \
    MOV64(a13, a42);                                                           \
    MOV64(a42, t);                                                             \
    # 交换a14和a41的值
    MOV64(t, a14);                                                             \
    MOV64(a14, a41);                                                           \
    MOV64(a41, t);                                                             \
    # 交换变量 t 和 a20 的值
    MOV64(t, a20);
    MOV64(a20, a30);
    MOV64(a30, t);
    
    # 交换变量 t 和 a21 的值
    MOV64(t, a21);
    MOV64(a21, a34);
    MOV64(a34, t);
    
    # 交换变量 t 和 a22 的值
    MOV64(t, a22);
    MOV64(a22, a33);
    MOV64(a33, t);
    
    # 交换变量 t 和 a23 的值
    MOV64(t, a23);
    MOV64(a23, a32);
    MOV64(a32, t);
    
    # 交换变量 t 和 a24 的值
    MOV64(t, a24);
    MOV64(a24, a31);
    MOV64(a31, t);
#define LPAR   (  # 定义宏 LPAR 为左括号
#define RPAR   )  # 定义宏 RPAR 为右括号

#define KF_ELT(r, s, k)                                                        \  # 定义宏 KF_ELT，参数为 r, s, k
  do {                                                                         \  # 定义宏 KF_ELT 的执行体
    THETA LPAR P##r RPAR;                                                      \  # 执行 THETA 操作，参数为 P##r
    RHO LPAR P##r RPAR;                                                        \  # 执行 RHO 操作，参数为 P##r
    KHI LPAR P##s RPAR;                                                        \  # 执行 KHI 操作，参数为 P##s
    IOTA(k);                                                                   \  # 执行 IOTA 操作，参数为 k
  } while (0)  # 定义宏 KF_ELT 的结束

#define DO(x) x  # 定义宏 DO，表示执行 x

#define KECCAK_F_1600 DO(KECCAK_F_1600_)  # 定义宏 KECCAK_F_1600，表示执行 KECCAK_F_1600_

#if SPH_KECCAK_UNROLL == 1  # 如果宏 SPH_KECCAK_UNROLL 的值为 1

#define KECCAK_F_1600_                                                         \  # 定义宏 KECCAK_F_1600_ 的执行体
  do {                                                                         \  
    int j;                                                                     \  # 定义整型变量 j
    for (j = 0; j < 24; j++) {                                                 \  # 循环，j 从 0 到 23
      KF_ELT(0, 1, RC[j + 0]);                                                 \  # 执行宏 KF_ELT，参数为 0, 1, RC[j + 0]
      P1_TO_P0;                                                                \  # 执行 P1_TO_P0 操作
    }                                                                          \  
  } while (0)  # 定义宏 KECCAK_F_1600_ 的结束

#elif SPH_KECCAK_UNROLL == 2  # 如果宏 SPH_KECCAK_UNROLL 的值为 2

#define KECCAK_F_1600_                                                         \  # 定义宏 KECCAK_F_1600_ 的执行体
  do {                                                                         \  
    int j;                                                                     \  # 定义整型变量 j
    for (j = 0; j < 24; j += 2) {                                              \  # 循环，j 从 0 到 22，步长为 2
      KF_ELT(0, 1, RC[j + 0]);                                                 \  # 执行宏 KF_ELT，参数为 0, 1, RC[j + 0]
      KF_ELT(1, 2, RC[j + 1]);                                                 \  # 执行宏 KF_ELT，参数为 1, 2, RC[j + 1]
      P2_TO_P0;                                                                \  # 执行 P2_TO_P0 操作
    }                                                                          \  
  } while (0)  # 定义宏 KECCAK_F_1600_ 的结束

#elif SPH_KECCAK_UNROLL == 4  # 如果宏 SPH_KECCAK_UNROLL 的值为 4
# 定义 KECCAK_F_1600_ 宏，用于实现 Keccak-1600 算法的一部分
# 根据 SPH_KECCAK_UNROLL 的不同取值，循环展开不同次数
# 当展开 4 次时
#define KECCAK_F_1600_                                                         \
  do {                                                                         \
    int j;                                                                     \
    for (j = 0; j < 24; j += 4) {                                              \
      # 调用 KF_ELT 宏，传入参数 0, 1, RC[j + 0]
      KF_ELT(0, 1, RC[j + 0]);                                                 \
      # 调用 KF_ELT 宏，传入参数 1, 2, RC[j + 1]
      KF_ELT(1, 2, RC[j + 1]);                                                 \
      # 调用 KF_ELT 宏，传入参数 2, 3, RC[j + 2]
      KF_ELT(2, 3, RC[j + 2]);                                                 \
      # 调用 KF_ELT 宏，传入参数 3, 4, RC[j + 3]
      KF_ELT(3, 4, RC[j + 3]);                                                 \
      # 调用 P4_TO_P0 宏
      P4_TO_P0;                                                                \
    }                                                                          \
  } while (0)
# 当展开 6 次时
#define KECCAK_F_1600_                                                         \
  do {                                                                         \
    int j;                                                                     \
    for (j = 0; j < 24; j += 6) {                                              \
      # 调用 KF_ELT 宏，传入参数 0, 1, RC[j + 0]
      KF_ELT(0, 1, RC[j + 0]);                                                 \
      # 调用 KF_ELT 宏，传入参数 1, 2, RC[j + 1]
      KF_ELT(1, 2, RC[j + 1]);                                                 \
      # 调用 KF_ELT 宏，传入参数 2, 3, RC[j + 2]
      KF_ELT(2, 3, RC[j + 2]);                                                 \
      # 调用 KF_ELT 宏，传入参数 3, 4, RC[j + 3]
      KF_ELT(3, 4, RC[j + 3]);                                                 \
      # 调用 KF_ELT 宏，传入参数 4, 5, RC[j + 4]
      KF_ELT(4, 5, RC[j + 4]);                                                 \
      # 调用 KF_ELT 宏，传入参数 5, 6, RC[j + 5]
      KF_ELT(5, 6, RC[j + 5]);                                                 \
      # 调用 P6_TO_P0 宏
      P6_TO_P0;                                                                \
    }                                                                          \
  } while (0)
# 当展开 8 次时
# 定义一个宏，用于计算 KECCAK_F_1600_ 的值
#define KECCAK_F_1600_                                                         \
  do {                                                                         \
    int j;                                                                     \
    # 循环24次，每次增加8
    for (j = 0; j < 24; j += 8) {                                              \
      # 调用宏 KF_ELT，传入参数0, 1, RC[j + 0]
      KF_ELT(0, 1, RC[j + 0]);                                                 \
      # 调用宏 KF_ELT，传入参数1, 2, RC[j + 1]
      KF_ELT(1, 2, RC[j + 1]);                                                 \
      # 调用宏 KF_ELT，传入参数2, 3, RC[j + 2]
      KF_ELT(2, 3, RC[j + 2]);                                                 \
      # 调用宏 KF_ELT，传入参数3, 4, RC[j + 3]
      KF_ELT(3, 4, RC[j + 3]);                                                 \
      # 调用宏 KF_ELT，传入参数4, 5, RC[j + 4]
      KF_ELT(4, 5, RC[j + 4]);                                                 \
      # 调用宏 KF_ELT，传入参数5, 6, RC[j + 5]
      KF_ELT(5, 6, RC[j + 5]);                                                 \
      # 调用宏 KF_ELT，传入参数6, 7, RC[j + 6]
      KF_ELT(6, 7, RC[j + 6]);                                                 \
      # 调用宏 KF_ELT，传入参数7, 8, RC[j + 7]
      KF_ELT(7, 8, RC[j + 7]);                                                 \
      # 调用宏 P8_TO_P0
      P8_TO_P0;                                                                \
    }                                                                          \
  } while (0)

#elif SPH_KECCAK_UNROLL == 12

#define KECCAK_F_1600_                                                         \
  do {                                                                         \
    int j;                                                                     \
    # 循环24次，每次增加12
    for (j = 0; j < 24; j += 12) {                                             \
      # 将RC数组中的第j个元素赋值给KF_ELT(0, 1)
      KF_ELT(0, 1, RC[j + 0]);                                                 \
      # 将RC数组中的第j+1个元素赋值给KF_ELT(1, 2)
      KF_ELT(1, 2, RC[j + 1]);                                                 \
      # 将RC数组中的第j+2个元素赋值给KF_ELT(2, 3)
      KF_ELT(2, 3, RC[j + 2]);                                                 \
      # 将RC数组中的第j+3个元素赋值给KF_ELT(3, 4)
      KF_ELT(3, 4, RC[j + 3]);                                                 \
      # 将RC数组中的第j+4个元素赋值给KF_ELT(4, 5)
      KF_ELT(4, 5, RC[j + 4]);                                                 \
      # 将RC数组中的第j+5个元素赋值给KF_ELT(5, 6)
      KF_ELT(5, 6, RC[j + 5]);                                                 \
      # 将RC数组中的第j+6个元素赋值给KF_ELT(6, 7)
      KF_ELT(6, 7, RC[j + 6]);                                                 \
      # 将RC数组中的第j+7个元素赋值给KF_ELT(7, 8)
      KF_ELT(7, 8, RC[j + 7]);                                                 \
      # 将RC数组中的第j+8个元素赋值给KF_ELT(8, 9)
      KF_ELT(8, 9, RC[j + 8]);                                                 \
      # 将RC数组中的第j+9个元素赋值给KF_ELT(9, 10)
      KF_ELT(9, 10, RC[j + 9]);                                                \
      # 将RC数组中的第j+10个元素赋值给KF_ELT(10, 11)
      KF_ELT(10, 11, RC[j + 10]);                                              \
      # 将RC数组中的第j+11个元素赋值给KF_ELT(11, 12)
      KF_ELT(11, 12, RC[j + 11]);                                              \
      # 将P12_TO_P0的值赋给P0
      P12_TO_P0;                                                               \
    }                                                                          \
  } while (0)
#elif SPH_KECCAK_UNROLL == 0
# 如果宏 SPH_KECCAK_UNROLL 的值等于 0，则执行以下代码块

#define KECCAK_F_1600_                                                         \
  do {                                                                         \
    KF_ELT(0, 1, RC[0]);                                                       \
    KF_ELT(1, 2, RC[1]);                                                       \
    KF_ELT(2, 3, RC[2]);                                                       \
    KF_ELT(3, 4, RC[3]);                                                       \
    KF_ELT(4, 5, RC[4]);                                                       \
    KF_ELT(5, 6, RC[5]);                                                       \
    KF_ELT(6, 7, RC[6]);                                                       \
    KF_ELT(7, 8, RC[7]);                                                       \
    KF_ELT(8, 9, RC[8]);                                                       \
    KF_ELT(9, 10, RC[9]);                                                      \
    KF_ELT(10, 11, RC[10]);                                                    \
    KF_ELT(11, 12, RC[11]);                                                    \
    KF_ELT(12, 13, RC[12]);                                                    \
    KF_ELT(13, 14, RC[13]);                                                    \
    KF_ELT(14, 15, RC[14]);                                                    \
    KF_ELT(15, 16, RC[15]);                                                    \
    KF_ELT(16, 17, RC[16]);                                                    \
    KF_ELT(17, 18, RC[17]);                                                    \
    KF_ELT(18, 19, RC[18]);                                                    \
    KF_ELT(19, 20, RC[19]);                                                    \
    KF_ELT(20, 21, RC[20]);                                                    \
    KF_ELT(21, 22, RC[21]);                                                    \
    # 定义了一个名为 KECCAK_F_1600_ 的宏，展开后执行一系列 KF_ELT 宏的调用
    # 使用宏KF_ELT对RC数组中的元素进行赋值操作，参数为(22, 23, RC[22])
    KF_ELT(22, 23, RC[22]);                                                    
    # 使用宏KF_ELT对RC数组中的元素进行赋值操作，参数为(23, 0, RC[23])
    KF_ELT(23, 0, RC[23]);                                                     
  } while (0)
#else

#error Unimplemented unroll count for Keccak.

#endif

static void keccak_init(sph_keccak_context *kc, unsigned out_size) {
  int i;

#if SPH_KECCAK_64
  for (i = 0; i < 25; i++)
    kc->u.wide[i] = 0;
  /*
   * 初始化"lane complement"。
   */
  kc->u.wide[1] = SPH_C64(0xFFFFFFFFFFFFFFFF);
  kc->u.wide[2] = SPH_C64(0xFFFFFFFFFFFFFFFF);
  kc->u.wide[8] = SPH_C64(0xFFFFFFFFFFFFFFFF);
  kc->u.wide[12] = SPH_C64(0xFFFFFFFFFFFFFFFF);
  kc->u.wide[17] = SPH_C64(0xFFFFFFFFFFFFFFFF);
  kc->u.wide[20] = SPH_C64(0xFFFFFFFFFFFFFFFF);
#else

  for (i = 0; i < 50; i++)
    kc->u.narrow[i] = 0;
  /*
   * 初始化"lane complement"。
   * 注意：由于我们设置了全为1的完整64位字，交错（如果适用）是一个无操作。
   */
  kc->u.narrow[2] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[3] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[4] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[5] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[16] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[17] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[24] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[25] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[34] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[35] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[40] = SPH_C32(0xFFFFFFFF);
  kc->u.narrow[41] = SPH_C32(0xFFFFFFFF);
#endif
  kc->ptr = 0;
  kc->lim = 200 - (out_size >> 2);
}

static void keccak_core(sph_keccak_context *kc, const void *data, size_t len,
                        size_t lim) {
  unsigned char *buf;
  size_t ptr;
  DECL_STATE

  buf = kc->buf;
  ptr = kc->ptr;

  if (len < (lim - ptr)) {
    memcpy(buf + ptr, data, len);
    kc->ptr = ptr + len;
    return;
  }

  READ_STATE(kc);
  while (len > 0) {
    size_t clen;

    clen = (lim - ptr);
    if (clen > len)
      clen = len;
    memcpy(buf + ptr, data, clen);
    ptr += clen;
    data = (const unsigned char *)data + clen;
    len -= clen;
    if (ptr == lim) {
      INPUT_BUF(lim);
      KECCAK_F_1600;
      ptr = 0;
    }
  }
  WRITE_STATE(kc);
  kc->ptr = ptr;
}
#if SPH_KECCAK_64
# 如果定义了 SPH_KECCAK_64 宏，则编译以下代码

#define DEFCLOSE(d, lim)                                                       \
# 定义一个名为 DEFCLOSE 的宏，带有参数 d 和 lim

  static void keccak_close##d(sph_keccak_context *kc,                          \
                              unsigned ub, unsigned n,                         \
                              void *dst) {                                     \
  # 定义一个名为 keccak_close##d 的静态函数，接受 sph_keccak_context 结构体指针 kc、无符号整数 ub、n，以及指向 void 类型的指针 dst 作为参数

    unsigned eb;                                                               \
    # 声明一个名为 eb 的无符号整数变量
    union {                                                                    \
      unsigned char tmp[lim + 1];                                              \
      sph_u64 dummy; /* for alignment */                                       \
    } u;                                                                       \
    # 声明一个名为 u 的联合体，包含一个名为 tmp 的无符号字符数组和一个名为 dummy 的 sph_u64 类型变量，用于对齐
    size_t j;                                                                  \
    # 声明一个名为 j 的 size_t 类型变量

    eb = hard_coded_eb;                                                        \
    # 将 hard_coded_eb 的值赋给 eb
    if (kc->ptr == (lim - 1)) {                                                \
    # 如果 kc 指针的 ptr 成员等于 (lim - 1) 

      if (n == 7) {                                                            \
      # 如果 n 等于 7
        u.tmp[0] = eb;                                                         \
        # 将 eb 的值赋给 u.tmp[0]
        memset(u.tmp + 1, 0, lim - 1);                                         \
        # 将 u.tmp[1] 到 u.tmp[lim-1] 的值设置为 0
        u.tmp[lim] = 0x80;                                                     \
        # 将 u.tmp[lim] 的值设置为 0x80
        j = 1 + lim;                                                           \
        # 将 j 的值设置为 1 + lim
      } else {                                                                 \
      # 如果 n 不等于 7
        u.tmp[0] = eb | 0x80;                                                  \
        # 将 (eb | 0x80) 的值赋给 u.tmp[0]
        j = 1;                                                                 \
        # 将 j 的值设置为 1
      }                                                                        \
      # 结束 if 语句
    } else {                                                                   \  # 如果不满足条件，执行以下操作
      j = lim - kc->ptr;                                                       \  # 计算 j 的值
      u.tmp[0] = eb;                                                           \  # 将 u.tmp 的第一个元素赋值为 eb
      memset(u.tmp + 1, 0, j - 2);                                             \  # 将 u.tmp 中指定范围的元素设置为 0
      u.tmp[j - 1] = 0x80;                                                     \  # 将 u.tmp 的倒数第二个元素赋值为 0x80
    }                                                                          \  # 结束 if-else 语句块
    keccak_core(kc, u.tmp, j, lim);                                            \  # 调用 keccak_core 函数
    /* Finalize the "lane complement" */                                       \  # 完成“lane complement”
    kc->u.wide[1] = ~kc->u.wide[1];                                            \  # 对 kc->u.wide[1] 进行按位取反操作
    kc->u.wide[2] = ~kc->u.wide[2];                                            \  # 对 kc->u.wide[2] 进行按位取反操作
    kc->u.wide[8] = ~kc->u.wide[8];                                            \  # 对 kc->u.wide[8] 进行按位取反操作
    kc->u.wide[12] = ~kc->u.wide[12];                                          \  # 对 kc->u.wide[12] 进行按位取反操作
    kc->u.wide[17] = ~kc->u.wide[17];                                          \  # 对 kc->u.wide[17] 进行按位取反操作
    kc->u.wide[20] = ~kc->u.wide[20];                                          \  # 对 kc->u.wide[20] 进行按位取反操作
    for (j = 0; j < d; j += 8)                                                 \  # 循环，每次增加 8
      sph_enc64le_aligned(u.tmp + j, kc->u.wide[j >> 3]);                      \  # 调用 sph_enc64le_aligned 函数
    memcpy(dst, u.tmp, d);                                                     \  # 将 u.tmp 中的数据复制到 dst 中，长度为 d
  }
#else

#define DEFCLOSE(d, lim)                                                       \
  static void keccak_close##d(sph_keccak_context *kc, unsigned ub, unsigned n, \
                              void *dst) {                                     \
    unsigned eb;                                                               \
    union {                                                                    \
      unsigned char tmp[lim + 1];                                              \
      sph_u64 dummy; /* for alignment */                                       \
    } u;                                                                       \
    size_t j;                                                                  \
                                                                               \
    eb = (0x100 | (ub & 0xFF)) >> (8 - n);                                     \
    # 计算eb的值，将ub的低8位与0xFF进行按位与运算，然后将结果与0x100进行按位或运算，再将结果右移(8-n)位
    if (kc->ptr == (lim - 1)) {                                                \
      # 如果kc->ptr的值等于lim-1
      if (n == 7) {                                                            \
        # 如果n的值等于7
        u.tmp[0] = eb;                                                         \
        # 将eb的值赋给u.tmp的第一个元素
        memset(u.tmp + 1, 0, lim - 1);                                         \
        # 将u.tmp的第二个元素到倒数第二个元素之间的所有元素置为0
        u.tmp[lim] = 0x80;                                                     \
        # 将u.tmp的最后一个元素置为0x80
        j = 1 + lim;                                                           \
        # 将j的值设置为1+lim
      } else {                                                                 \
        # 如果n的值不等于7
        u.tmp[0] = eb | 0x80;                                                  \
        # 将eb的值与0x80进行按位或运算，然后赋给u.tmp的第一个元素
        j = 1;                                                                 \
        # 将j的值设置为1
      }                                                                        \
    } else {                                                                   \
      j = lim - kc->ptr;                                                       \
      u.tmp[0] = eb;                                                           \
      memset(u.tmp + 1, 0, j - 2);                                             \
      u.tmp[j - 1] = 0x80;                                                     \
    }                                                                          \
    keccak_core(kc, u.tmp, j, lim);                                            \
    # 完成"lane complement"的最后处理，将每个元素取反
    kc->u.narrow[2] = ~kc->u.narrow[2];                                        \
    kc->u.narrow[3] = ~kc->u.narrow[3];                                        \
    kc->u.narrow[4] = ~kc->u.narrow[4];                                        \
    kc->u.narrow[5] = ~kc->u.narrow[5];                                        \
    kc->u.narrow[16] = ~kc->u.narrow[16];                                      \
    kc->u.narrow[17] = ~kc->u.narrow[17];                                      \
    kc->u.narrow[24] = ~kc->u.narrow[24];                                      \
    kc->u.narrow[25] = ~kc->u.narrow[25];                                      \
    kc->u.narrow[34] = ~kc->u.narrow[34];                                      \
    kc->u.narrow[35] = ~kc->u.narrow[35];                                      \
    kc->u.narrow[40] = ~kc->u.narrow[40];                                      \
    kc->u.narrow[41] = ~kc->u.narrow[41];                                      \
    # 解交织操作
    for (j = 0; j < 50; j += 2)                                                \
      UNINTERLEAVE(kc->u.narrow[j], kc->u.narrow[j + 1]);                      \


注释：
    # 对于每个 j 从 0 开始，每次增加 4，直到 j 小于 d
    for (j = 0; j < d; j += 4)                                                 \
      # 将 kc->u.narrow[j >> 2] 的值以小端序写入 u.tmp + j
      sph_enc32le_aligned(u.tmp + j, kc->u.narrow[j >> 2]);                    \
    # 将 u.tmp 的内容复制到 dst
    memcpy(dst, u.tmp, d);                                                     \
    # 使用 (unsigned)d << 3 初始化 keccak 算法的状态
    keccak_init(kc, (unsigned)d << 3);                                         \
    }
// 关闭宏定义，结束条件编译
#endif

// 调用 DEFCLOSE 宏定义，参数为 28 和 144
DEFCLOSE(28, 144)
// 调用 DEFCLOSE 宏定义，参数为 32 和 136
DEFCLOSE(32, 136)
// 调用 DEFCLOSE 宏定义，参数为 48 和 104
DEFCLOSE(48, 104)
// 调用 DEFCLOSE 宏定义，参数为 64 和 72
DEFCLOSE(64, 72)

/* see sph_keccak.h */
// 初始化 Keccak224 算法
void sph_keccak224_init(void *cc) { keccak_init(cc, 224); }

/* see sph_keccak.h */
// 执行 Keccak224 算法
void sph_keccak224(void *cc, const void *data, size_t len) {
  keccak_core(cc, data, len, 144);
}

/* see sph_keccak.h */
// 关闭 Keccak224 算法
void sph_keccak224_close(void *cc, void *dst) {
  sph_keccak224_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_keccak.h */
// 添加比特并关闭 Keccak224 算法
void sph_keccak224_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                     void *dst) {
  keccak_close28(cc, ub, n, dst);
}

// 以下代码类似，分别对 Keccak256, Keccak384, Keccak512 算法进行了初始化、执行、关闭和添加比特并关闭的操作
// 用于将位数据添加到 Keccak 算法中，并关闭 Keccak 算法
void sph_keccak512_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                     void *dst) {
  // 调用 keccak_close64 函数，将位数据添加到 Keccak 算法中，并关闭 Keccak 算法
  keccak_close64(cc, ub, n, dst);
}

#ifdef __cplusplus
}
#endif
// 结束 C++ 语言环境
```