# `xmrig\src\crypto\ghostrider\sph_skein.c`

```
/* $Id: skein.c 254 2011-06-07 19:38:58Z tp $ */
/* 
 * Skein implementation.
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

#include "sph_skein.h"

#ifdef __cplusplus
extern "C"{
#endif

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_SKEIN
#define SPH_SMALL_FOOTPRINT_SKEIN   1
#endif

#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif

#if SPH_64

#if 0
/* obsolete */
/*
 * M5_ ## s ## _ ## i  evaluates to s+i mod 5 (0 <= s <= 18, 0 <= i <= 3).
 */

#define M5_0_0    0
#define M5_0_1    1
#define M5_0_2    2
#define M5_0_3    3

#define M5_1_0    1
#define M5_1_1    2
#define M5_1_2    3
#define M5_1_3    4

#define M5_2_0    2
#define M5_2_1    3
#define M5_2_2    4
#define M5_2_3    0

#define M5_3_0    3
#define M5_3_1    4
#define M5_3_2    0
#define M5_3_3    1

#define M5_4_0    4
#define M5_4_1    0
#define M5_4_2    1
#define M5_4_3    2

#define M5_5_0    0
#define M5_5_1    1
#define M5_5_2    2
#define M5_5_3    3

#define M5_6_0    1
#define M5_6_1    2
#define M5_6_2    3
#define M5_6_3    4

#define M5_7_0    2
#define M5_7_1    3
#define M5_7_2    4
#define M5_7_3    0

#define M5_8_0    3
#define M5_8_1    4
#define M5_8_2    0
#define M5_8_3    1

#define M5_9_0    4
#define M5_9_1    0
#define M5_9_2    1
#define M5_9_3    2

#define M5_10_0   0
#define M5_10_1   1
#define M5_10_2   2
#define M5_10_3   3

#define M5_11_0   1
#define M5_11_1   2
#define M5_11_2   3
#define M5_11_3   4

#define M5_12_0   2
#define M5_12_1   3
#define M5_12_2   4
#define M5_12_3   0

#define M5_13_0   3
#define M5_13_1   4
#define M5_13_2   0
#define M5_13_3   1

#define M5_14_0   4
#define M5_14_1   0
#define M5_14_2   1
#define M5_14_3   2

#define M5_15_0   0
#define M5_15_1   1
#define M5_15_2   2
#define M5_15_3   3

#define M5_16_0   1
#define M5_16_1   2
#define M5_16_2   3
#define M5_16_3   4

#define M5_17_0   2
#define M5_17_1   3
#define M5_17_2   4
#define M5_17_3   0

#define M5_18_0   3
#define M5_18_1   4
#define M5_18_2   0
#define M5_18_3   1
#endif

/*
 * M9_ ## s ## _ ## i  evaluates to s+i mod 9 (0 <= s <= 18, 0 <= i <= 7).
 */

#define M9_0_0    0
#define M9_0_1    1
#define M9_0_2    2
#define M9_0_3    3
#define M9_0_4    4
#define M9_0_5    5
#define M9_0_6    6
#define M9_0_7    7

#define M9_1_0    1
#define M9_1_1    2
#define M9_1_2    3
#define M9_1_3    4
#define M9_1_4    5
#define M9_1_5    6
#define M9_1_6    7
#define M9_1_7    8

#define M9_2_0    2
#define M9_2_1    3
#define M9_2_2    4
#define M9_2_3    5
#define M9_2_4    6
#define M9_2_5    7
#define M9_2_6    8
#define M9_2_7    0

#define M9_3_0    3
#define M9_3_1    4
#define M9_3_2    5


注释：


# 定义了一系列的宏，用于计算不同数值的模9结果
# M5_2_1 表示 3 mod 9
# M5_2_2 表示 4 mod 9
# M5_2_3 表示 0 mod 9
# ...
# M9_0_0 表示 0 mod 9
# M9_0_1 表示 1 mod 9
# M9_0_2 表示 2 mod 9
# ...
# M9_3_0 表示 3 mod 9
# M9_3_1 表示 4 mod 9
# M9_3_2 表示 5 mod 9
# 定义常量 M9_3_3，值为 6
#define M9_3_4    7
#define M9_3_5    8
#define M9_3_6    0
#define M9_3_7    1

#define M9_4_0    4
#define M9_4_1    5
#define M9_4_2    6
#define M9_4_3    7
#define M9_4_4    8
#define M9_4_5    0
#define M9_4_6    1
#define M9_4_7    2

# 定义常量 M9_5_0，值为 5
#define M9_5_1    6
#define M9_5_2    7
#define M9_5_3    8
#define M9_5_4    0
#define M9_5_5    1
#define M9_5_6    2
#define M9_5_7    3

# 定义常量 M9_6_0，值为 6
#define M9_6_1    7
#define M9_6_2    8
#define M9_6_3    0
#define M9_6_4    1
#define M9_6_5    2
#define M9_6_6    3
#define M9_6_7    4

# 定义常量 M9_7_0，值为 7
#define M9_7_1    8
#define M9_7_2    0
#define M9_7_3    1
#define M9_7_4    2
#define M9_7_5    3
#define M9_7_6    4
#define M9_7_7    5

# 定义常量 M9_8_0，值为 8
#define M9_8_1    0
#define M9_8_2    1
#define M9_8_3    2
#define M9_8_4    3
#define M9_8_5    4
#define M9_8_6    5
#define M9_8_7    6

# 定义常量 M9_9_0，值为 0
#define M9_9_1    1
#define M9_9_2    2
#define M9_9_3    3
#define M9_9_4    4
#define M9_9_5    5
#define M9_9_6    6
#define M9_9_7    7

# 定义常量 M9_10_0，值为 1
#define M9_10_1   2
#define M9_10_2   3
#define M9_10_3   4
#define M9_10_4   5
#define M9_10_5   6
#define M9_10_6   7
#define M9_10_7   8

# 定义常量 M9_11_0，值为 2
#define M9_11_1   3
#define M9_11_2   4
#define M9_11_3   5
#define M9_11_4   6
#define M9_11_5   7
#define M9_11_6   8
#define M9_11_7   0

# 定义常量 M9_12_0，值为 3
#define M9_12_1   4
#define M9_12_2   5
#define M9_12_3   6
#define M9_12_4   7
#define M9_12_5   8
#define M9_12_6   0
#define M9_12_7   1

# 定义常量 M9_13_0，值为 4
#define M9_13_1   5
#define M9_13_2   6
#define M9_13_3   7
#define M9_13_4   8
#define M9_13_5   0
#define M9_13_6   1
#define M9_13_7   2

# 定义常量 M9_14_0，值为 5
#define M9_14_1   6
#define M9_14_2   7
#define M9_14_3   8
#define M9_14_4   0
#define M9_14_5   1
#define M9_14_6   2
#define M9_14_7   3

# 定义常量 M9_15_0，值为 6
#define M9_15_1   7
#define M9_15_2   8
#define M9_15_3   0
#define M9_15_4   1
#define M9_15_5   2
#define M9_15_6   3   // 定义宏 M9_15_6 为 3
#define M9_15_7   4   // 定义宏 M9_15_7 为 4

#define M9_16_0   7   // 定义宏 M9_16_0 为 7
#define M9_16_1   8   // 定义宏 M9_16_1 为 8
#define M9_16_2   0   // 定义宏 M9_16_2 为 0
#define M9_16_3   1   // 定义宏 M9_16_3 为 1
#define M9_16_4   2   // 定义宏 M9_16_4 为 2
#define M9_16_5   3   // 定义宏 M9_16_5 为 3
#define M9_16_6   4   // 定义宏 M9_16_6 为 4
#define M9_16_7   5   // 定义宏 M9_16_7 为 5

#define M9_17_0   8   // 定义宏 M9_17_0 为 8
#define M9_17_1   0   // 定义宏 M9_17_1 为 0
#define M9_17_2   1   // 定义宏 M9_17_2 为 1
#define M9_17_3   2   // 定义宏 M9_17_3 为 2
#define M9_17_4   3   // 定义宏 M9_17_4 为 3
#define M9_17_5   4   // 定义宏 M9_17_5 为 4
#define M9_17_6   5   // 定义宏 M9_17_6 为 5
#define M9_17_7   6   // 定义宏 M9_17_7 为 6

#define M9_18_0   0   // 定义宏 M9_18_0 为 0
#define M9_18_1   1   // 定义宏 M9_18_1 为 1
#define M9_18_2   2   // 定义宏 M9_18_2 为 2
#define M9_18_3   3   // 定义宏 M9_18_3 为 3
#define M9_18_4   4   // 定义宏 M9_18_4 为 4
#define M9_18_5   5   // 定义宏 M9_18_5 为 5
#define M9_18_6   6   // 定义宏 M9_18_6 为 6
#define M9_18_7   7   // 定义宏 M9_18_7 为 7

/*
 * M3_ ## s ## _ ## i  evaluates to s+i mod 3 (0 <= s <= 18, 0 <= i <= 1).
 */

#define M3_0_0    0   // 定义宏 M3_0_0 为 0
#define M3_0_1    1   // 定义宏 M3_0_1 为 1
#define M3_1_0    1   // 定义宏 M3_1_0 为 1
#define M3_1_1    2   // 定义宏 M3_1_1 为 2
#define M3_2_0    2   // 定义宏 M3_2_0 为 2
#define M3_2_1    0   // 定义宏 M3_2_1 为 0
#define M3_3_0    0   // 定义宏 M3_3_0 为 0
#define M3_3_1    1   // 定义宏 M3_3_1 为 1
#define M3_4_0    1   // 定义宏 M3_4_0 为 1
#define M3_4_1    2   // 定义宏 M3_4_1 为 2
#define M3_5_0    2   // 定义宏 M3_5_0 为 2
#define M3_5_1    0   // 定义宏 M3_5_1 为 0
#define M3_6_0    0   // 定义宏 M3_6_0 为 0
#define M3_6_1    1   // 定义宏 M3_6_1 为 1
#define M3_7_0    1   // 定义宏 M3_7_0 为 1
#define M3_7_1    2   // 定义宏 M3_7_1 为 2
#define M3_8_0    2   // 定义宏 M3_8_0 为 2
#define M3_8_1    0   // 定义宏 M3_8_1 为 0
#define M3_9_0    0   // 定义宏 M3_9_0 为 0
#define M3_9_1    1   // 定义宏 M3_9_1 为 1
#define M3_10_0   1   // 定义宏 M3_10_0 为 1
#define M3_10_1   2   // 定义宏 M3_10_1 为 2
#define M3_11_0   2   // 定义宏 M3_11_0 为 2
#define M3_11_1   0   // 定义宏 M3_11_1 为 0
#define M3_12_0   0   // 定义宏 M3_12_0 为 0
#define M3_12_1   1   // 定义宏 M3_12_1 为 1
#define M3_13_0   1   // 定义宏 M3_13_0 为 1
#define M3_13_1   2   // 定义宏 M3_13_1 为 2
#define M3_14_0   2   // 定义宏 M3_14_0 为 2
#define M3_14_1   0   // 定义宏 M3_14_1 为 0
#define M3_15_0   0   // 定义宏 M3_15_0 为 0
#define M3_15_1   1   // 定义宏 M3_15_1 为 1
#define M3_16_0   1   // 定义宏 M3_16_0 为 1
#define M3_16_1   2   // 定义宏 M3_16_1 为 2
#define M3_17_0   2   // 定义宏 M3_17_0 为 2
#define M3_17_1   0   // 定义宏 M3_17_1 为 0
#define M3_18_0   0   // 定义宏 M3_18_0 为 0
#define M3_18_1   1   // 定义宏 M3_18_1 为 1

#define XCAT(x, y)     XCAT_(x, y)   // 定义宏 XCAT，用于连接两个宏
#define XCAT_(x, y)    x ## y   // 定义宏 XCAT_，用于将两个宏连接成一个新的宏

#if 0
/* obsolete */
#define SKSI(k, s, i)   XCAT(k, XCAT(XCAT(XCAT(M5_, s), _), i))   // 定义宏 SKSI，用于生成一个新的宏
#define SKST(t, s, v)   XCAT(t, XCAT(XCAT(XCAT(M3_, s), _), v))   // 定义宏 SKST，用于生成一个新的宏
#endif

#define SKBI(k, s, i)   XCAT(k, XCAT(XCAT(XCAT(M9_, s), _), i))   // 定义宏 SKBI，用于生成一个新的宏
#define SKBT(t, s, v)   XCAT(t, XCAT(XCAT(XCAT(M3_, s), _), v))   // 定义宏 SKBT，用于生成一个新的宏

#if 0
/* obsolete */
#define TFSMALL_KINIT(k0, k1, k2, k3, k4, t0, t1, t2)   do { \
        k4 = (k0 ^ k1) ^ (k2 ^ k3) ^ SPH_C64(0x1BD11BDAA9FC1A22); \
        t2 = t0 ^ t1; \
    } while (0)
#endif
#define TFBIG_KINIT(k0, k1, k2, k3, k4, k5, k6, k7, k8, t0, t1, t2)   do { \
        k8 = ((k0 ^ k1) ^ (k2 ^ k3)) ^ ((k4 ^ k5) ^ (k6 ^ k7)) \
            ^ SPH_C64(0x1BD11BDAA9FC1A22); \
        t2 = t0 ^ t1; \
    } while (0)



#if 0
/* obsolete */
#define TFSMALL_ADDKEY(w0, w1, w2, w3, k, t, s)   do { \
        w0 = SPH_T64(w0 + SKSI(k, s, 0)); \
        w1 = SPH_T64(w1 + SKSI(k, s, 1) + SKST(t, s, 0)); \
        w2 = SPH_T64(w2 + SKSI(k, s, 2) + SKST(t, s, 1)); \
        w3 = SPH_T64(w3 + SKSI(k, s, 3) + (sph_u64)s); \
    } while (0)
#endif



#if SPH_SMALL_FOOTPRINT_SKEIN

#define TFBIG_ADDKEY(s, tt0, tt1)   do { \
        p0 = SPH_T64(p0 + h[s + 0]); \
        p1 = SPH_T64(p1 + h[s + 1]); \
        p2 = SPH_T64(p2 + h[s + 2]); \
        p3 = SPH_T64(p3 + h[s + 3]); \
        p4 = SPH_T64(p4 + h[s + 4]); \
        p5 = SPH_T64(p5 + h[s + 5] + tt0); \
        p6 = SPH_T64(p6 + h[s + 6] + tt1); \
        p7 = SPH_T64(p7 + h[s + 7] + (sph_u64)s); \
    } while (0)

#else

#define TFBIG_ADDKEY(w0, w1, w2, w3, w4, w5, w6, w7, k, t, s)   do { \
        w0 = SPH_T64(w0 + SKBI(k, s, 0)); \
        w1 = SPH_T64(w1 + SKBI(k, s, 1)); \
        w2 = SPH_T64(w2 + SKBI(k, s, 2)); \
        w3 = SPH_T64(w3 + SKBI(k, s, 3)); \
        w4 = SPH_T64(w4 + SKBI(k, s, 4)); \
        w5 = SPH_T64(w5 + SKBI(k, s, 5) + SKBT(t, s, 0)); \
        w6 = SPH_T64(w6 + SKBI(k, s, 6) + SKBT(t, s, 1)); \
        w7 = SPH_T64(w7 + SKBI(k, s, 7) + (sph_u64)s); \
    } while (0)

#endif



#if 0
/* obsolete */
#define TFSMALL_MIX(x0, x1, rc)   do { \
        x0 = SPH_T64(x0 + x1); \
        x1 = SPH_ROTL64(x1, rc) ^ x0; \
    } while (0)
#endif



#define TFBIG_MIX(x0, x1, rc)   do { \
        x0 = SPH_T64(x0 + x1); \
        x1 = SPH_ROTL64(x1, rc) ^ x0; \
    } while (0)



#if 0
/* obsolete */
#define TFSMALL_MIX4(w0, w1, w2, w3, rc0, rc1)  do { \
        TFSMALL_MIX(w0, w1, rc0); \
        TFSMALL_MIX(w2, w3, rc1); \
    } while (0)
#endif


注释：
#define TFBIG_MIX8(w0, w1, w2, w3, w4, w5, w6, w7, rc0, rc1, rc2, rc3)  do { \
        // 定义一个宏，用于执行TFBIG_MIX函数，传入参数w0, w1, rc0
        TFBIG_MIX(w0, w1, rc0); \
        // 定义一个宏，用于执行TFBIG_MIX函数，传入参数w2, w3, rc1
        TFBIG_MIX(w2, w3, rc1); \
        // 定义一个宏，用于执行TFBIG_MIX函数，传入参数w4, w5, rc2
        TFBIG_MIX(w4, w5, rc2); \
        // 定义一个宏，用于执行TFBIG_MIX函数，传入参数w6, w7, rc3
        TFBIG_MIX(w6, w7, rc3); \
    } while (0)

#if 0
/* obsolete */
#define TFSMALL_4e(s)   do { \
        // 执行TFSMALL_ADDKEY、TFSMALL_MIX4等函数
        TFSMALL_ADDKEY(p0, p1, p2, p3, h, t, s); \
        TFSMALL_MIX4(p0, p1, p2, p3, 14, 16); \
        TFSMALL_MIX4(p0, p3, p2, p1, 52, 57); \
        TFSMALL_MIX4(p0, p1, p2, p3, 23, 40); \
        TFSMALL_MIX4(p0, p3, p2, p1,  5, 37); \
    } while (0)

#define TFSMALL_4o(s)   do { \
        // 执行TFSMALL_ADDKEY、TFSMALL_MIX4等函数
        TFSMALL_ADDKEY(p0, p1, p2, p3, h, t, s); \
        TFSMALL_MIX4(p0, p1, p2, p3, 25, 33); \
        TFSMALL_MIX4(p0, p3, p2, p1, 46, 12); \
        TFSMALL_MIX4(p0, p1, p2, p3, 58, 22); \
        TFSMALL_MIX4(p0, p3, p2, p1, 32, 32); \
    } while (0)
#endif

#if SPH_SMALL_FOOTPRINT_SKEIN

#define TFBIG_4e(s)   do { \
        // 执行TFBIG_ADDKEY、TFBIG_MIX8等函数
        TFBIG_ADDKEY(s, t0, t1); \
        TFBIG_MIX8(p0, p1, p2, p3, p4, p5, p6, p7, 46, 36, 19, 37); \
        TFBIG_MIX8(p2, p1, p4, p7, p6, p5, p0, p3, 33, 27, 14, 42); \
        TFBIG_MIX8(p4, p1, p6, p3, p0, p5, p2, p7, 17, 49, 36, 39); \
        TFBIG_MIX8(p6, p1, p0, p7, p2, p5, p4, p3, 44,  9, 54, 56); \
    } while (0)

#define TFBIG_4o(s)   do { \
        // 执行TFBIG_ADDKEY、TFBIG_MIX8等函数
        TFBIG_ADDKEY(s, t1, t2); \
        TFBIG_MIX8(p0, p1, p2, p3, p4, p5, p6, p7, 39, 30, 34, 24); \
        TFBIG_MIX8(p2, p1, p4, p7, p6, p5, p0, p3, 13, 50, 10, 17); \
        TFBIG_MIX8(p4, p1, p6, p3, p0, p5, p2, p7, 25, 29, 39, 43); \
        TFBIG_MIX8(p6, p1, p0, p7, p2, p5, p4, p3,  8, 35, 56, 22); \
    } while (0)

#else

#define TFBIG_4e(s)   do { \
        // 执行TFBIG_ADDKEY、TFBIG_MIX8等函数
        TFBIG_ADDKEY(p0, p1, p2, p3, p4, p5, p6, p7, h, t, s); \
        TFBIG_MIX8(p0, p1, p2, p3, p4, p5, p6, p7, 46, 36, 19, 37); \
        TFBIG_MIX8(p2, p1, p4, p7, p6, p5, p0, p3, 33, 27, 14, 42); \
        TFBIG_MIX8(p4, p1, p6, p3, p0, p5, p2, p7, 17, 49, 36, 39); \
        TFBIG_MIX8(p6, p1, p0, p7, p2, p5, p4, p3, 44,  9, 54, 56); \
    } while (0)
#if 0
/* obsolete */
#define UBI_SMALL(etype, extra)  do { \
        sph_u64 h4, t0, t1, t2; \
        sph_u64 m0 = sph_dec64le(buf +  0); \
        sph_u64 m1 = sph_dec64le(buf +  8); \
        sph_u64 m2 = sph_dec64le(buf + 16); \
        sph_u64 m3 = sph_dec64le(buf + 24); \
        sph_u64 p0 = m0; \
        sph_u64 p1 = m1; \
        sph_u64 p2 = m2; \
        sph_u64 p3 = m3; \
        t0 = SPH_T64(bcount << 5) + (sph_u64)(extra); \
        t1 = (bcount >> 59) + ((sph_u64)(etype) << 55); \
        TFSMALL_KINIT(h0, h1, h2, h3, h4, t0, t1, t2); \
        TFSMALL_4e(0); \
        TFSMALL_4o(1); \
        TFSMALL_4e(2); \
        TFSMALL_4o(3); \
        TFSMALL_4e(4); \
        TFSMALL_4o(5); \
        TFSMALL_4e(6); \
        TFSMALL_4o(7); \
        TFSMALL_4e(8); \
        TFSMALL_4o(9); \
        TFSMALL_4e(10); \
        TFSMALL_4o(11); \
        TFSMALL_4e(12); \
        TFSMALL_4o(13); \
        TFSMALL_4e(14); \
        TFSMALL_4o(15); \
        TFSMALL_4e(16); \
        TFSMALL_4o(17); \
        TFSMALL_ADDKEY(p0, p1, p2, p3, h, t, 18); \
        h0 = m0 ^ p0; \
        h1 = m1 ^ p1; \
        h2 = m2 ^ p2; \
        h3 = m3 ^ p3; \
    } while (0)
#endif

#if SPH_SMALL_FOOTPRINT_SKEIN


注释：这部分代码是被注释掉的，因此不会被执行。
# 定义一个宏函数UBI_BIG，用于处理大端序的数据块
# 宏函数的参数包括etype和extra
# 宏函数内部定义了一系列变量和常量
# 将输入的数据块按照64位的大小进行解码，并存储在对应的变量中
# 定义了一系列变量p0-p7，用于存储解码后的数据块
# 根据输入的参数etype和extra计算出t0、t1和t2的值
# 调用TFBIG_KINIT函数对h数组进行初始化
# 使用循环将h数组的值复制到h数组的后面，以便后续计算
# 使用循环对h数组进行一系列的操作，包括调用TFBIG_4e和TFBIG_4o函数，以及交换t0、t1和t2的值
# 调用TFBIG_ADDKEY函数对t0和t1进行处理
# 将解码后的数据块与p0-p7进行异或操作，并将结果存储在h数组中
# 定义一个宏，用于处理大端模式的数据
#define UBI_BIG(etype, extra)  do { \
        # 定义变量
        sph_u64 h8, t0, t1, t2; \
        # 读取8个字节的数据，并按照小端模式转换成64位整数
        sph_u64 m0 = sph_dec64le_aligned(buf +  0); \
        sph_u64 m1 = sph_dec64le_aligned(buf +  8); \
        sph_u64 m2 = sph_dec64le_aligned(buf + 16); \
        sph_u64 m3 = sph_dec64le_aligned(buf + 24); \
        sph_u64 m4 = sph_dec64le_aligned(buf + 32); \
        sph_u64 m5 = sph_dec64le_aligned(buf + 40); \
        sph_u64 m6 = sph_dec64le_aligned(buf + 48); \
        sph_u64 m7 = sph_dec64le_aligned(buf + 56); \
        # 复制m0-m7的值到p0-p7
        sph_u64 p0 = m0; \
        sph_u64 p1 = m1; \
        sph_u64 p2 = m2; \
        sph_u64 p3 = m3; \
        sph_u64 p4 = m4; \
        sph_u64 p5 = m5; \
        sph_u64 p6 = m6; \
        sph_u64 p7 = m7; \
        # 计算t0和t1的值
        t0 = SPH_T64(bcount << 6) + (sph_u64)(extra); \
        t1 = (bcount >> 58) + ((sph_u64)(etype) << 55); \
        # 初始化TFBIG的K值
        TFBIG_KINIT(h0, h1, h2, h3, h4, h5, h6, h7, h8, t0, t1, t2); \
        # 进行TFBIG的4e和4o操作
        TFBIG_4e(0); \
        TFBIG_4o(1); \
        TFBIG_4e(2); \
        TFBIG_4o(3); \
        TFBIG_4e(4); \
        TFBIG_4o(5); \
        TFBIG_4e(6); \
        TFBIG_4o(7); \
        TFBIG_4e(8); \
        TFBIG_4o(9); \
        TFBIG_4e(10); \
        TFBIG_4o(11); \
        TFBIG_4e(12); \
        TFBIG_4o(13); \
        TFBIG_4e(14); \
        TFBIG_4o(15); \
        TFBIG_4e(16); \
        TFBIG_4o(17); \
        # 添加密钥
        TFBIG_ADDKEY(p0, p1, p2, p3, p4, p5, p6, p7, h, t, 18); \
        # 计算h0-h7的值
        h0 = m0 ^ p0; \
        h1 = m1 ^ p1; \
        h2 = m2 ^ p2; \
        h3 = m3 ^ p3; \
        h4 = m4 ^ p4; \
        h5 = m5 ^ p5; \
        h6 = m6 ^ p6; \
        h7 = m7 ^ p7; \
    } while (0)

#endif

#if 0
/* obsolete */
# 定义一个宏，用于处理小端模式的数据
#define DECL_STATE_SMALL \
    sph_u64 h0, h1, h2, h3; \
    sph_u64 bcount;

# 读取小端模式的状态
#define READ_STATE_SMALL(sc)   do { \
        h0 = (sc)->h0; \
        h1 = (sc)->h1; \
        h2 = (sc)->h2; \
        h3 = (sc)->h3; \
        bcount = sc->bcount; \
    } while (0)
#define WRITE_STATE_SMALL(sc)   do { \  # 定义宏，用于将参数中的值赋给结构体成员变量
        (sc)->h0 = h0; \  # 将 h0 的值赋给结构体成员变量 h0
        (sc)->h1 = h1; \  # 将 h1 的值赋给结构体成员变量 h1
        (sc)->h2 = h2; \  # 将 h2 的值赋给结构体成员变量 h2
        (sc)->h3 = h3; \  # 将 h3 的值赋给结构体成员变量 h3
        sc->bcount = bcount; \  # 将 bcount 的值赋给结构体成员变量 bcount
    } while (0)  # 定义宏结束
#endif  # 结束宏定义

#if SPH_SMALL_FOOTPRINT_SKEIN  # 如果定义了 SPH_SMALL_FOOTPRINT_SKEIN
#define DECL_STATE_BIG \  # 定义宏，用于声明结构体成员变量
    sph_u64 h[27]; \  # 声明一个包含 27 个元素的数组 h
    sph_u64 bcount;  # 声明一个变量 bcount

#define READ_STATE_BIG(sc)   do { \  # 定义宏，用于将结构体成员变量的值赋给局部变量
        h[0] = (sc)->h0; \  # 将结构体成员变量 h0 的值赋给数组 h 的第一个元素
        h[1] = (sc)->h1; \  # 将结构体成员变量 h1 的值赋给数组 h 的第二个元素
        h[2] = (sc)->h2; \  # 将结构体成员变量 h2 的值赋给数组 h 的第三个元素
        h[3] = (sc)->h3; \  # 将结构体成员变量 h3 的值赋给数组 h 的第四个元素
        h[4] = (sc)->h4; \  # 将结构体成员变量 h4 的值赋给数组 h 的第五个元素
        h[5] = (sc)->h5; \  # 将结构体成员变量 h5 的值赋给数组 h 的第六个元素
        h[6] = (sc)->h6; \  # 将结构体成员变量 h6 的值赋给数组 h 的第七个元素
        h[7] = (sc)->h7; \  # 将结构体成员变量 h7 的值赋给数组 h 的第八个元素
        bcount = sc->bcount; \  # 将结构体成员变量 bcount 的值赋给变量 bcount
    } while (0)  # 定义宏结束

#define WRITE_STATE_BIG(sc)   do { \  # 定义宏，用于将局部变量的值赋给结构体成员变量
        (sc)->h0 = h[0]; \  # 将数组 h 的第一个元素的值赋给结构体成员变量 h0
        (sc)->h1 = h[1]; \  # 将数组 h 的第二个元素的值赋给结构体成员变量 h1
        (sc)->h2 = h[2]; \  # 将数组 h 的第三个元素的值赋给结构体成员变量 h2
        (sc)->h3 = h[3]; \  # 将数组 h 的第四个元素的值赋给结构体成员变量 h3
        (sc)->h4 = h[4]; \  # 将数组 h 的第五个元素的值赋给结构体成员变量 h4
        (sc)->h5 = h[5]; \  # 将数组 h 的第六个元素的值赋给结构体成员变量 h5
        (sc)->h6 = h[6]; \  # 将数组 h 的第七个元素的值赋给结构体成员变量 h6
        (sc)->h7 = h[7]; \  # 将数组 h 的第八个元素的值赋给结构体成员变量 h7
        sc->bcount = bcount; \  # 将变量 bcount 的值赋给结构体成员变量 bcount
    } while (0)  # 定义宏结束

#else  # 如果没有定义 SPH_SMALL_FOOTPRINT_SKEIN
#define DECL_STATE_BIG \  # 定义宏，用于声明结构体成员变量
    sph_u64 h0, h1, h2, h3, h4, h5, h6, h7; \  # 声明 8 个 sph_u64 类型的变量
    sph_u64 bcount;  # 声明一个变量 bcount

#define READ_STATE_BIG(sc)   do { \  # 定义宏，用于将结构体成员变量的值赋给局部变量
        h0 = (sc)->h0; \  # 将结构体成员变量 h0 的值赋给变量 h0
        h1 = (sc)->h1; \  # 将结构体成员变量 h1 的值赋给变量 h1
        h2 = (sc)->h2; \  # 将结构体成员变量 h2 的值赋给变量 h2
        h3 = (sc)->h3; \  # 将结构体成员变量 h3 的值赋给变量 h3
        h4 = (sc)->h4; \  # 将结构体成员变量 h4 的值赋给变量 h4
        h5 = (sc)->h5; \  # 将结构体成员变量 h5 的值赋给变量 h5
        h6 = (sc)->h6; \  # 将结构体成员变量 h6 的值赋给变量 h6
        h7 = (sc)->h7; \  # 将结构体成员变量 h7 的值赋给变量 h7
        bcount = sc->bcount; \  # 将结构体成员变量 bcount 的值赋给变量 bcount
    } while (0)  # 定义宏结束

#define WRITE_STATE_BIG(sc)   do { \  # 定义宏，用于将局部变量的值赋给结构体成员变量
        (sc)->h0 = h0; \  # 将变量 h0 的值赋给结构体成员变量 h0
        (sc)->h1 = h1; \  # 将变量 h1 的值赋给结构体成员变量 h1
        (sc)->h2 = h2; \  # 将变量 h2 的值赋给结构体成员变量 h2
        (sc)->h3 = h3; \  # 将变量 h3 的值赋给结构体成员变量 h3
        (sc)->h4 = h4; \  # 将变量 h4 的值赋给结构体成员变量 h4
        (sc)->h5 = h5; \  # 将变量 h5 的值赋给结构体成员变量 h5
        (sc)->h6 = h6; \  # 将变量 h6 的值赋给结构体成员变量 h6
        (sc)->h7 = h7; \  # 将变量 h7 的值赋给结构体成员变量 h7
        sc->bcount = bcount; \  # 将变量 bcount 的值赋给结构体成员变量 bcount
    } while (0)  # 定义宏结束

#endif  # 结束条件编译

#if 0  # 如果条件为 0
/* obsolete */  # 注释：过时的
static void  # 静态函数声明
skein_small_init(sph_skein_small_context *sc, const sph_u64 *iv)  # 函数声明，参数为指向 sph_skein_small_context 类型和 sph_u64 类型的指针
{  # 函数体开始
    sc->h0 = iv[0];  # 将 iv 数组的第一个元素的值赋给结构体成员变量 h0
    sc->h1 = iv[1];  # 将 iv 数组的第二个元素的值赋给结构体成员变量 h1
    sc->h2 = iv[2];  # 将 iv 数组的第三个元素的值赋给结构体成员变量 h2
    sc->h3 = iv[3];  # 将 iv 数组的第四个元素的值赋给结构体成员变量 h3
    sc->bcount = 0;  # 将 0 赋给结构体成员变量 bcount
    sc->ptr = 0;  # 将 0 赋给结构体成员变量 ptr
}  # 函数体结束
#endif  # 结束条件编译

static void  # 静态函数声明
skein_big_init(sph_skein_big_context *sc, const sph_u64 *iv)  # 函数声明，参数为指向 sph_skein_big_context 类型和 sph_u64 类型的指针
{  # 函数体开始
    sc->h0 = iv[0];  # 将 iv 数组的第一个元素的值赋给结构体成员变量 h0
    sc->h1 = iv[1];  # 将 iv 数组的第二个元素的值赋给结构体成员变量 h1
    sc->h2 = iv[2];  # 将 iv 数组的第三个元素的值赋给结构体成员变量 h2
    sc->h3 = iv[3];  # 将 iv 数组的第四个元素的值赋给结构体成员变量 h3
    sc->h4 = iv[4];  # 将 iv 数组的第五个元素的值赋给结构体成员变量 h4
    sc->h5 = iv[5];  # 将 iv 数组的第六个元素的值赋给结构体成员变量 h5
    # 将iv数组中的第6个元素赋值给结构体sc的h6成员
    sc->h6 = iv[6];
    # 将iv数组中的第7个元素赋值给结构体sc的h7成员
    sc->h7 = iv[7];
    # 将结构体sc的bcount成员赋值为0
    sc->bcount = 0;
    # 将结构体sc的ptr成员赋值为0
    sc->ptr = 0;
#if 0
/* obsolete */
static void
skein_small_core(sph_skein_small_context *sc, const void *data, size_t len)
{
    // 定义变量
    unsigned char *buf;
    size_t ptr, clen;
    unsigned first;
    DECL_STATE_SMALL

    // 初始化变量
    buf = sc->buf;
    ptr = sc->ptr;
    clen = (sizeof sc->buf) - ptr;
    // 如果数据长度小于等于缓冲区剩余空间，则将数据拷贝到缓冲区中
    if (len <= clen) {
        memcpy(buf + ptr, data, len);
        sc->ptr = ptr + len;
        return;
    }
    // 如果缓冲区剩余空间不为0，则将数据拷贝到缓冲区中，并更新数据和长度
    if (clen != 0) {
        memcpy(buf + ptr, data, clen);
        data = (const unsigned char *)data + clen;
        len -= clen;
    }

#if SPH_SMALL_FOOTPRINT_SKEIN

    // 读取状态
    READ_STATE_SMALL(sc);
    // 计算first的值
    first = (bcount == 0) << 7;
    // 循环处理数据
    for (;;) {
        // 更新数据块计数
        bcount ++;
        // 调用UBI_SMALL函数处理数据块
        UBI_SMALL(96 + first, 0);
        // 如果数据长度小于等于缓冲区大小，则退出循环
        if (len <= sizeof sc->buf)
            break;
        // 更新first的值
        first = 0;
        // 将数据拷贝到缓冲区中，并更新数据和长度
        memcpy(buf, data, sizeof sc->buf);
        data = (const unsigned char *)data + sizeof sc->buf;
        len -= sizeof sc->buf;
    }
    // 写入状态
    WRITE_STATE_SMALL(sc);
    // 更新缓冲区指针
    sc->ptr = len;
    // 将剩余数据拷贝到缓冲区中
    memcpy(buf, data, len);

#else

    /*
     * Unrolling the loop yields a slight performance boost, while
     * keeping the code size aorund 24 kB on 32-bit x86.
     */
    // 读取状态
    READ_STATE_SMALL(sc);
    // 计算first的值
    first = (bcount == 0) << 7;
    // 循环处理数据
    for (;;) {
        // 更新数据块计数
        bcount ++;
        // 调用UBI_SMALL函数处理数据块
        UBI_SMALL(96 + first, 0);
        // 如果数据长度小于等于缓冲区大小，则退出循环
        if (len <= sizeof sc->buf)
            break;
        // 将数据指针指向缓冲区
        buf = (unsigned char *)data;
        // 更新数据块计数
        bcount ++;
        // 调用UBI_SMALL函数处理数据块
        UBI_SMALL(96, 0);
        // 如果数据长度小于等于2倍缓冲区大小，则退出循环
        if (len <= 2 * sizeof sc->buf) {
            data = buf + sizeof sc->buf;
            len -= sizeof sc->buf;
            break;
        }
        // 更新数据指针
        buf += sizeof sc->buf;
        data = buf + sizeof sc->buf;
        // 更新first的值
        first = 0;
        len -= 2 * sizeof sc->buf;
    }
    // 写入状态
    WRITE_STATE_SMALL(sc);
    // 更新缓冲区指针
    sc->ptr = len;
    // 将剩余数据拷贝到缓冲区中
    memcpy(sc->buf, data, len);

#endif
}
#endif

static void
skein_big_core(sph_skein_big_context *sc, const void *data, size_t len)
{
    # 创建指向无符号字符的指针buf，用于存储缓冲区数据
    unsigned char *buf;
    # 存储指向缓冲区数据的指针的偏移量
    size_t ptr;
    # 存储是否为第一个块的标志位
    unsigned first;
    # 声明大端状态
    DECL_STATE_BIG

    # 将sc->buf的地址赋给buf
    buf = sc->buf;
    # 将sc->ptr的值赋给ptr
    ptr = sc->ptr;
    # 如果len小于等于缓冲区的剩余空间
    if (len <= (sizeof sc->buf) - ptr) {
        # 将data中的len个字节复制到buf + ptr的位置
        memcpy(buf + ptr, data, len);
        # 更新ptr的值
        ptr += len;
        # 更新sc->ptr的值
        sc->ptr = ptr;
        # 返回
        return;
    }

    # 读取大端状态
    READ_STATE_BIG(sc);

    # 设置first的值为(bcount == 0) << 7
    first = (bcount == 0) << 7;
    # 循环直到len为0
    do {
        # 定义clen变量
        size_t clen;

        # 如果ptr等于缓冲区的大小
        if (ptr == sizeof sc->buf) {
            # bcount加1
            bcount ++;
            # 调用UBI_BIG函数，传入96 + first和0作为参数
            UBI_BIG(96 + first, 0);
            # 将first的值设为0
            first = 0;
            # 将ptr的值设为0
            ptr = 0;
        }
        # 计算clen的值为缓冲区的大小减去ptr
        clen = (sizeof sc->buf) - ptr;
        # 如果clen大于len，则将clen的值设为len
        if (clen > len)
            clen = len;
        # 将data中的clen个字节复制到buf + ptr的位置
        memcpy(buf + ptr, data, clen);
        # 更新ptr的值
        ptr += clen;
        # 将data的指针向后移动clen个字节
        data = (const unsigned char *)data + clen;
        # 更新len的值
        len -= clen;
    } while (len > 0);
    # 写入大端状态
    WRITE_STATE_BIG(sc);
    # 更新sc->ptr的值
    sc->ptr = ptr;
# 关闭小型 Skein 哈希函数上下文
static void
skein_small_close(sph_skein_small_context *sc, unsigned ub, unsigned n,
    void *dst, size_t out_len)
{
    unsigned char *buf;  # 定义一个指向无符号字符的指针变量 buf
    size_t ptr;  # 定义一个大小为 size_t 的变量 ptr
    unsigned et;  # 定义一个无符号整型变量 et
    int i;  # 定义一个整型变量 i
    DECL_STATE_SMALL  # 定义一个小型状态

    if (n != 0) {  # 如果 n 不等于 0
        unsigned z;  # 定义一个无符号整型变量 z
        unsigned char x;  # 定义一个无符号字符变量 x

        z = 0x80 >> n;  # 计算 z 的值
        x = ((ub & -z) | z) & 0xFF;  # 计算 x 的值
        skein_small_core(sc, &x, 1);  # 调用 skein_small_core 函数
    }

    buf = sc->buf;  # 将 sc->buf 的值赋给 buf
    ptr = sc->ptr;  # 将 sc->ptr 的值赋给 ptr
    READ_STATE_SMALL(sc);  # 读取小型状态
    memset(buf + ptr, 0, (sizeof sc->buf) - ptr);  # 将 buf + ptr 处开始的 (sizeof sc->buf) - ptr 个字节设置为 0
    et = 352 + ((bcount == 0) << 7) + (n != 0);  # 计算 et 的值
    for (i = 0; i < 2; i ++) {  # 循环两次
        UBI_SMALL(et, ptr);  # 调用 UBI_SMALL 函数
        if (i == 0) {  # 如果 i 等于 0
            memset(buf, 0, sizeof sc->buf);  # 将 buf 处开始的 sizeof sc->buf 个字节设置为 0
            bcount = 0;  # 将 bcount 的值设置为 0
            et = 510;  # 将 et 的值设置为 510
            ptr = 8;  # 将 ptr 的值设置为 8
        }
    }

    sph_enc64le_aligned(buf +  0, h0);  # 将 h0 编码为小端序并存储到 buf + 0 处
    sph_enc64le_aligned(buf +  8, h1);  # 将 h1 编码为小端序并存储到 buf + 8 处
    sph_enc64le_aligned(buf + 16, h2);  # 将 h2 编码为小端序并存储到 buf + 16 处
    sph_enc64le_aligned(buf + 24, h3);  # 将 h3 编码为小端序并存储到 buf + 24 处
    memcpy(dst, buf, out_len);  # 将 buf 的内容复制到 dst
}

# 关闭大型 Skein 哈希函数上下文
static void
skein_big_close(sph_skein_big_context *sc, unsigned ub, unsigned n,
    void *dst, size_t out_len)
{
    unsigned char *buf;  # 定义一个指向无符号字符的指针变量 buf
    size_t ptr;  # 定义一个大小为 size_t 的变量 ptr
    unsigned et;  # 定义一个无符号整型变量 et
    int i;  # 定义一个整型变量 i
#if SPH_SMALL_FOOTPRINT_SKEIN
    size_t u;  # 定义一个大小为 size_t 的变量 u
#endif
    DECL_STATE_BIG  # 定义一个大型状态

    """
    Add bit padding if necessary.
    """
    # 如果需要，添加位填充
    if (n != 0) {  # 如果 n 不等于 0
        unsigned z;  # 定义一个无符号整型变量 z
        unsigned char x;  # 定义一个无符号字符变量 x

        z = 0x80 >> n;  # 计算 z 的值
        x = ((ub & -z) | z) & 0xFF;  # 计算 x 的值
        skein_big_core(sc, &x, 1);  # 调用 skein_big_core 函数
    }

    buf = sc->buf;  # 将 sc->buf 的值赋给 buf
    ptr = sc->ptr;  # 将 sc->ptr 的值赋给 ptr
    /*
     * 在这一点上，如果 ptr == 0，则消息为空；
     * 否则，还有1到64个字节（包括64个字节）尚未处理。
     * 无论如何，我们都将缓冲区填充为一个完整的块，用零填充
     * （Skein规范要求对空消息进行填充，以便至少有一个块进行处理）。
     *
     * 一旦这个块被处理，我们再次进行处理，用一个全是零的块，用于输出
     * （该块包含对“0”的编码，占8个字节，然后用零填充）。
     */
    READ_STATE_BIG(sc);  // 读取大端模式的状态
    memset(buf + ptr, 0, (sizeof sc->buf) - ptr);  // 用零填充缓冲区，直到填满一个完整的块
    et = 352 + ((bcount == 0) << 7) + (n != 0);  // 计算 et 的值
    for (i = 0; i < 2; i ++) {  // 循环两次
        UBI_BIG(et, ptr);  // 使用大端模式处理 et 和 ptr
        if (i == 0) {  // 如果是第一次循环
            memset(buf, 0, sizeof sc->buf);  // 用零填充整个缓冲区
            bcount = 0;  // 重置 bcount 为 0
            et = 510;  // 重新计算 et 的值
            ptr = 8;  // 重置 ptr 为 8
        }
    }
#if SPH_SMALL_FOOTPRINT_SKEIN
// 如果定义了 SPH_SMALL_FOOTPRINT_SKEIN，则执行以下代码块
    /*
     * We use a temporary buffer because we must support the case
     * where output size is not a multiple of 64 (namely, a 224-bit
     * output).
     */
    // 使用临时缓冲区是因为我们必须支持输出大小不是64的倍数的情况（即224位输出）
    for (u = 0; u < out_len; u += 8)
        // 将h[u >> 3]的值以小端格式编码写入buf + u
        sph_enc64le_aligned(buf + u, h[u >> 3]);
    // 将buf的内容复制到dst中，长度为out_len
    memcpy(dst, buf, out_len);

#else
// 如果未定义 SPH_SMALL_FOOTPRINT_SKEIN，则执行以下代码块
    // 将h0的值以小端格式编码写入buf + 0
    sph_enc64le_aligned(buf +  0, h0);
    // 将h1的值以小端格式编码写入buf + 8
    sph_enc64le_aligned(buf +  8, h1);
    // 将h2的值以小端格式编码写入buf + 16
    sph_enc64le_aligned(buf + 16, h2);
    // 将h3的值以小端格式编码写入buf + 24
    sph_enc64le_aligned(buf + 24, h3);
    // 将h4的值以小端格式编码写入buf + 32
    sph_enc64le_aligned(buf + 32, h4);
    // 将h5的值以小端格式编码写入buf + 40
    sph_enc64le_aligned(buf + 40, h5);
    // 将h6的值以小端格式编码写入buf + 48
    sph_enc64le_aligned(buf + 48, h6);
    // 将h7的值以小端格式编码写入buf + 56
    sph_enc64le_aligned(buf + 56, h7);
    // 将buf的内容复制到dst中，长度为out_len
    memcpy(dst, buf, out_len);

#endif
}

#if 0
/* obsolete */
// 下面的代码块已经过时
static const sph_u64 IV224[] = {
    SPH_C64(0xC6098A8C9AE5EA0B), SPH_C64(0x876D568608C5191C),
    SPH_C64(0x99CB88D7D7F53884), SPH_C64(0x384BDDB1AEDDB5DE)
};

static const sph_u64 IV256[] = {
    SPH_C64(0xFC9DA860D048B449), SPH_C64(0x2FCA66479FA7D833),
    SPH_C64(0xB33BC3896656840F), SPH_C64(0x6A54E920FDE8DA69)
};
#endif

// 初始化224位Skein算法的IV值
static const sph_u64 IV224[] = {
    SPH_C64(0xCCD0616248677224), SPH_C64(0xCBA65CF3A92339EF),
    SPH_C64(0x8CCD69D652FF4B64), SPH_C64(0x398AED7B3AB890B4),
    SPH_C64(0x0F59D1B1457D2BD0), SPH_C64(0x6776FE6575D4EB3D),
    SPH_C64(0x99FBC70E997413E9), SPH_C64(0x9E2CFCCFE1C41EF7)
};

// 初始化256位Skein算法的IV值
static const sph_u64 IV256[] = {
    SPH_C64(0xCCD044A12FDB3E13), SPH_C64(0xE83590301A79A9EB),
    SPH_C64(0x55AEA0614F816E6F), SPH_C64(0x2A2767A4AE9B94DB),
    SPH_C64(0xEC06025E74DD7683), SPH_C64(0xE7A436CDC4746251),
    SPH_C64(0xC36FBAF9393AD185), SPH_C64(0x3EEDBA1833EDFC13)
};

// 初始化384位Skein算法的IV值
static const sph_u64 IV384[] = {
    SPH_C64(0xA3F6C6BF3A75EF5F), SPH_C64(0xB0FEF9CCFD84FAA4),
    SPH_C64(0x9D77DD663D770CFE), SPH_C64(0xD798CBF3B468FDDA),
    SPH_C64(0x1BC4A6668A0E4465), SPH_C64(0x7ED7D434E5807407),
    SPH_C64(0x548FC1ACD4EC44D6), SPH_C64(0x266E17546AA18FF8)
};

// 初始化512位Skein算法的IV值
static const sph_u64 IV512[] = {
    SPH_C64(0x4903ADFF749C51CE), SPH_C64(0x0D95DE399746DF03),
    # 定义一系列64位的常量值
    SPH_C64(0x8FD1934127C79BCE), SPH_C64(0x9A255629FF352CB1),
    SPH_C64(0x5DB62599DF6CA7B0), SPH_C64(0xEABE394CA9D5C3F4),
    SPH_C64(0x991112C71A75B523), SPH_C64(0xAE18A40B660FCC33)
#if 0
/* obsolete */
/* see sph_skein.h */
void
sph_skein224_init(void *cc)
{
    skein_small_init(cc, IV224);
}

/* see sph_skein.h */
void
sph_skein224(void *cc, const void *data, size_t len)
{
    skein_small_core(cc, data, len);
}

/* see sph_skein.h */
void
sph_skein224_close(void *cc, void *dst)
{
    sph_skein224_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_skein.h */
void
sph_skein224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    skein_small_close(cc, ub, n, dst, 28);
    sph_skein224_init(cc);
}

/* see sph_skein.h */
void
sph_skein256_init(void *cc)
{
    skein_small_init(cc, IV256);
}

/* see sph_skein.h */
void
sph_skein256(void *cc, const void *data, size_t len)
{
    skein_small_core(cc, data, len);
}

/* see sph_skein.h */
void
sph_skein256_close(void *cc, void *dst)
{
    sph_skein256_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_skein.h */
void
sph_skein256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    skein_small_close(cc, ub, n, dst, 32);
    sph_skein256_init(cc);
}
#endif

/* see sph_skein.h */
void
sph_skein224_init(void *cc)
{
    // 使用 IV224 初始化 skein_big 状态
    skein_big_init(cc, IV224);
}

/* see sph_skein.h */
void
sph_skein224(void *cc, const void *data, size_t len)
{
    // 使用 skein_big 状态处理数据
    skein_big_core(cc, data, len);
}

/* see sph_skein.h */
void
sph_skein224_close(void *cc, void *dst)
{
    // 添加位数并关闭 skein_big 状态
    sph_skein224_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_skein.h */
void
sph_skein224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 关闭 skein_big 状态并输出结果
    skein_big_close(cc, ub, n, dst, 28);
//    sph_skein224_init(cc);
}

/* see sph_skein.h */
void
sph_skein256_init(void *cc)
{
    // 使用 IV256 初始化 skein_big 状态
    skein_big_init(cc, IV256);
}

/* see sph_skein.h */
void
sph_skein256(void *cc, const void *data, size_t len)
{
    // 使用 skein_big 状态处理数据
    skein_big_core(cc, data, len);
}

/* see sph_skein.h */
void
sph_skein256_close(void *cc, void *dst)
{
    // 添加位数并关闭 skein_big 状态
    sph_skein256_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_skein.h */
void
/* 在给定的 SKEIN256 状态上添加位并关闭，生成结果 */
sph_skein256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 skein_big_close 函数，生成 SKEIN256 结果
    skein_big_close(cc, ub, n, dst, 32);
//    sph_skein256_init(cc);
}

/* 初始化 SKEIN384 状态 */
void
sph_skein384_init(void *cc)
{
    // 调用 skein_big_init 函数，初始化 SKEIN384 状态
    skein_big_init(cc, IV384);
}

/* 计算 SKEIN384 哈希值 */
void
sph_skein384(void *cc, const void *data, size_t len)
{
    // 调用 skein_big_core 函数，计算 SKEIN384 哈希值
    skein_big_core(cc, data, len);
}

/* 在给定的 SKEIN384 状态上添加位并关闭，生成结果 */
void
sph_skein384_close(void *cc, void *dst)
{
    // 调用 sph_skein384_addbits_and_close 函数，添加位并关闭 SKEIN384 状态，生成结果
    sph_skein384_addbits_and_close(cc, 0, 0, dst);
}

/* 在给定的 SKEIN384 状态上添加位并关闭，生成结果 */
void
sph_skein384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 skein_big_close 函数，生成 SKEIN384 结果
    skein_big_close(cc, ub, n, dst, 48);
//    sph_skein384_init(cc);
}

/* 初始化 SKEIN512 状态 */
void
sph_skein512_init(void *cc)
{
    // 调用 skein_big_init 函数，初始化 SKEIN512 状态
    skein_big_init(cc, IV512);
}

/* 计算 SKEIN512 哈希值 */
void
sph_skein512(void *cc, const void *data, size_t len)
{
    // 调用 skein_big_core 函数，计算 SKEIN512 哈希值
    skein_big_core(cc, data, len);
}

/* 在给定的 SKEIN512 状态上添加位并关闭，生成结果 */
void
sph_skein512_close(void *cc, void *dst)
{
    // 调用 sph_skein512_addbits_and_close 函数，添加位并关闭 SKEIN512 状态，生成结果
    sph_skein512_addbits_and_close(cc, 0, 0, dst);
}

/* 在给定的 SKEIN512 状态上添加位并关闭，生成结果 */
void
sph_skein512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 skein_big_close 函数，生成 SKEIN512 结果
    skein_big_close(cc, ub, n, dst, 64);
//    sph_skein512_init(cc);
}

#endif

#ifdef __cplusplus
}
#endif
```