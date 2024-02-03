# `xmrig\src\crypto\ghostrider\sph_luffa.c`

```cpp
/* $Id: luffa.c 219 2010-06-08 17:24:41Z tp $ */
# 版本信息和作者信息

/*
 * Luffa implementation.
 * Luffa算法的实现
 *
 * ==========================(LICENSE BEGIN)============================
 * 授权信息开始
 *
 * Copyright (c) 2007-2010  Projet RNRT SAPHIR
 * 版权信息
 *
 * Permission is hereby granted, free of charge, to any person obtaining
 * a copy of this software and associated documentation files (the
 * "Software"), to deal in the Software without restriction, including
 * without limitation the rights to use, copy, modify, merge, publish,
 * distribute, sublicense, and/or sell copies of the Software, and to
 * permit persons to whom the Software is furnished to do so, subject to
 * the following conditions:
 * 授权许可
 *
 * The above copyright notice and this permission notice shall be
 * included in all copies or substantial portions of the Software.
 * 版权和许可信息
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 * IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
 * CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
 * TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
 * SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
 * 软件的使用和免责声明
 *
 * ===========================(LICENSE END)=============================
 * 授权信息结束
 *
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 * 作者信息
 */

#include <stddef.h>
#include <string.h>
#include <limits.h>

#include "sph_luffa.h"

#ifdef __cplusplus
extern "C"{
#endif

#if SPH_64_TRUE && !defined SPH_LUFFA_PARALLEL
#define SPH_LUFFA_PARALLEL   1
#endif

#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif

static const sph_u32 V_INIT[5][8] = {
    {
        SPH_C32(0x6d251e69), SPH_C32(0x44b051e0),
        SPH_C32(0x4eaa6fb4), SPH_C32(0xdbf78465),
        SPH_C32(0x6e292011), SPH_C32(0x90152df4),
        SPH_C32(0xee058139), SPH_C32(0xdef610bb)


注释：
    # 定义一个包含多个元素的列表，每个元素都是一个32位的常量
    # 这些常量将用于后续的计算
    }, {
        SPH_C32(0xc3b44b95), SPH_C32(0xd9d2f256),
        SPH_C32(0x70eee9a0), SPH_C32(0xde099fa3),
        SPH_C32(0x5d9b0557), SPH_C32(0x8fc944b3),
        SPH_C32(0xcf1ccf0e), SPH_C32(0x746cd581)
    }, {
        SPH_C32(0xf7efc89d), SPH_C32(0x5dba5781),
        SPH_C32(0x04016ce5), SPH_C32(0xad659c05),
        SPH_C32(0x0306194f), SPH_C32(0x666d1836),
        SPH_C32(0x24aa230a), SPH_C32(0x8b264ae7)
    }, {
        SPH_C32(0x858075d5), SPH_C32(0x36d79cce),
        SPH_C32(0xe571f7d7), SPH_C32(0x204b1f67),
        SPH_C32(0x35870c6a), SPH_C32(0x57e9e923),
        SPH_C32(0x14bcb808), SPH_C32(0x7cde72ce)
    }, {
        SPH_C32(0x6c68e9be), SPH_C32(0x5ec41e22),
        SPH_C32(0xc825b7c7), SPH_C32(0xaffb4363),
        SPH_C32(0xf5df3999), SPH_C32(0x0fc688f1),
        SPH_C32(0xb07224cc), SPH_C32(0x03e86cea)
    }
// 定义常量数组 RC00，包含8个32位无符号整数
static const sph_u32 RC00[8] = {
    SPH_C32(0x303994a6), SPH_C32(0xc0e65299),
    SPH_C32(0x6cc33a12), SPH_C32(0xdc56983e),
    SPH_C32(0x1e00108f), SPH_C32(0x7800423d),
    SPH_C32(0x8f5b7882), SPH_C32(0x96e1db12)
};

// 定义常量数组 RC04，包含8个32位无符号整数
static const sph_u32 RC04[8] = {
    SPH_C32(0xe0337818), SPH_C32(0x441ba90d),
    SPH_C32(0x7f34d442), SPH_C32(0x9389217f),
    SPH_C32(0xe5a8bce6), SPH_C32(0x5274baf4),
    SPH_C32(0x26889ba7), SPH_C32(0x9a226e9d)
};

// 定义常量数组 RC10，包含8个32位无符号整数
static const sph_u32 RC10[8] = {
    SPH_C32(0xb6de10ed), SPH_C32(0x70f47aae),
    SPH_C32(0x0707a3d4), SPH_C32(0x1c1e8f51),
    SPH_C32(0x707a3d45), SPH_C32(0xaeb28562),
    SPH_C32(0xbaca1589), SPH_C32(0x40a46f3e)
};

// 定义常量数组 RC14，包含8个32位无符号整数
static const sph_u32 RC14[8] = {
    SPH_C32(0x01685f3d), SPH_C32(0x05a17cf4),
    SPH_C32(0xbd09caca), SPH_C32(0xf4272b28),
    SPH_C32(0x144ae5cc), SPH_C32(0xfaa7ae2b),
    SPH_C32(0x2e48f1c1), SPH_C32(0xb923c704)
};

#if SPH_LUFFA_PARALLEL

// 定义常量数组 RCW010，包含8个64位无符号整数
static const sph_u64 RCW010[8] = {
    SPH_C64(0xb6de10ed303994a6), SPH_C64(0x70f47aaec0e65299),
    SPH_C64(0x0707a3d46cc33a12), SPH_C64(0x1c1e8f51dc56983e),
    SPH_C64(0x707a3d451e00108f), SPH_C64(0xaeb285627800423d),
    SPH_C64(0xbaca15898f5b7882), SPH_C64(0x40a46f3e96e1db12)
};

// 定义常量数组 RCW014，包含8个64位无符号整数
static const sph_u64 RCW014[8] = {
    SPH_C64(0x01685f3de0337818), SPH_C64(0x05a17cf4441ba90d),
    SPH_C64(0xbd09caca7f34d442), SPH_C64(0xf4272b289389217f),
    SPH_C64(0x144ae5cce5a8bce6), SPH_C64(0xfaa7ae2b5274baf4),
    SPH_C64(0x2e48f1c126889ba7), SPH_C64(0xb923c7049a226e9d)
};

#endif

// 定义常量数组 RC20，包含8个32位无符号整数
static const sph_u32 RC20[8] = {
    SPH_C32(0xfc20d9d2), SPH_C32(0x34552e25),
    SPH_C32(0x7ad8818f), SPH_C32(0x8438764a),
    SPH_C32(0xbb6de032), SPH_C32(0xedb780c8),
    SPH_C32(0xd9847356), SPH_C32(0xa2c78434)
};

// 定义常量数组 RC24，包含8个32位无符号整数
static const sph_u32 RC24[8] = {
    SPH_C32(0xe25e72c1), SPH_C32(0xe623bb72),
    SPH_C32(0x5c58a4a4), SPH_C32(0x1e38e2e7),
    SPH_C32(0x78e38b9d), SPH_C32(0x27586719),
    SPH_C32(0x36eda57f), SPH_C32(0x703aace7)
};

// 定义常量数组 RC30，包含8个32位无符号整数
static const sph_u32 RC30[8] = {
    # 定义一系列常量，使用 SPH_C32 函数将十六进制数转换为 32 位整数
    SPH_C32(0xb213afa5), SPH_C32(0xc84ebe95),
    SPH_C32(0x4e608a22), SPH_C32(0x56d858fe),
    SPH_C32(0x343b138f), SPH_C32(0xd0ec4e3d),
    SPH_C32(0x2ceb4882), SPH_C32(0xb3ad2208)
};

# 定义常量数组RC34，包含8个32位无符号整数
static const sph_u32 RC34[8] = {
    SPH_C32(0xe028c9bf), SPH_C32(0x44756f91),
    SPH_C32(0x7e8fce32), SPH_C32(0x956548be),
    SPH_C32(0xfe191be2), SPH_C32(0x3cb226e5),
    SPH_C32(0x5944a28e), SPH_C32(0xa1c4c355)
};

#if SPH_LUFFA_PARALLEL

# 定义常量数组RCW230，包含8个64位无符号整数
static const sph_u64 RCW230[8] = {
    SPH_C64(0xb213afa5fc20d9d2), SPH_C64(0xc84ebe9534552e25),
    SPH_C64(0x4e608a227ad8818f), SPH_C64(0x56d858fe8438764a),
    SPH_C64(0x343b138fbb6de032), SPH_C64(0xd0ec4e3dedb780c8),
    SPH_C64(0x2ceb4882d9847356), SPH_C64(0xb3ad2208a2c78434)
};


# 定义常量数组RCW234，包含8个64位无符号整数
static const sph_u64 RCW234[8] = {
    SPH_C64(0xe028c9bfe25e72c1), SPH_C64(0x44756f91e623bb72),
    SPH_C64(0x7e8fce325c58a4a4), SPH_C64(0x956548be1e38e2e7),
    SPH_C64(0xfe191be278e38b9d), SPH_C64(0x3cb226e527586719),
    SPH_C64(0x5944a28e36eda57f), SPH_C64(0xa1c4c355703aace7)
};

#endif

# 定义常量数组RC40，包含8个32位无符号整数
static const sph_u32 RC40[8] = {
    SPH_C32(0xf0d2e9e3), SPH_C32(0xac11d7fa),
    SPH_C32(0x1bcb66f2), SPH_C32(0x6f2d9bc9),
    SPH_C32(0x78602649), SPH_C32(0x8edae952),
    SPH_C32(0x3b6ba548), SPH_C32(0xedae9520)
};

# 定义常量数组RC44，包含8个32位无符号整数
static const sph_u32 RC44[8] = {
    SPH_C32(0x5090d577), SPH_C32(0x2d1925ab),
    SPH_C32(0xb46496ac), SPH_C32(0xd1925ab0),
    SPH_C32(0x29131ab6), SPH_C32(0x0fc053c3),
    SPH_C32(0x3f014f0c), SPH_C32(0xfc053c31)
};

# 定义宏，用于声明8个临时变量
#define DECL_TMP8(w) \
    sph_u32 w ## 0, w ## 1, w ## 2, w ## 3, w ## 4, w ## 5, w ## 6, w ## 7;

# 定义宏，用于将s的值复制给d
# 这里的s和d是临时变量，通过宏的参数传入
# 宏的作用是将s的值复制给d的对应位
#define M2(d, s)   do { \
        sph_u32 tmp = s ## 7; \
        d ## 7 = s ## 6; \
        d ## 6 = s ## 5; \
        d ## 5 = s ## 4; \
        d ## 4 = s ## 3 ^ tmp; \
        d ## 3 = s ## 2 ^ tmp; \
        d ## 2 = s ## 1; \
        d ## 1 = s ## 0 ^ tmp; \
        d ## 0 = tmp; \
    } while (0)
#define XOR(d, s1, s2)   do { \  # 定义一个宏，实现对两个参数进行按位异或操作，并将结果赋值给第一个参数
        d ## 0 = s1 ## 0 ^ s2 ## 0; \  # 对每个参数的对应位进行异或操作，并将结果赋值给目标参数的对应位
        d ## 1 = s1 ## 1 ^ s2 ## 1; \
        d ## 2 = s1 ## 2 ^ s2 ## 2; \
        d ## 3 = s1 ## 3 ^ s2 ## 3; \
        d ## 4 = s1 ## 4 ^ s2 ## 4; \
        d ## 5 = s1 ## 5 ^ s2 ## 5; \
        d ## 6 = s1 ## 6 ^ s2 ## 6; \
        d ## 7 = s1 ## 7 ^ s2 ## 7; \
    } while (0)  # 宏定义结束

#if SPH_LUFFA_PARALLEL  # 如果定义了 SPH_LUFFA_PARALLEL

#define SUB_CRUMB_GEN(a0, a1, a2, a3, width)   do { \  # 定义一个宏，实现对四个参数进行一系列位运算操作
        sph_u ## width tmp; \  # 声明一个临时变量
        tmp = (a0); \  # 将 a0 的值赋给临时变量
        (a0) |= (a1); \  # 对 a0 和 a1 进行按位或操作，并将结果赋给 a0
        (a2) ^= (a3); \  # 对 a2 和 a3 进行按位异或操作，并将结果赋给 a2
        (a1) = SPH_T ## width(~(a1)); \  # 对 a1 按位取反，并将结果赋给 a1
        (a0) ^= (a3); \  # 对 a0 和 a3 进行按位异或操作，并将结果赋给 a0
        (a3) &= tmp; \  # 对 a3 和临时变量进行按位与操作，并将结果赋给 a3
        (a1) ^= (a3); \  # 对 a1 和 a3 进行按位异或操作，并将结果赋给 a1
        (a3) ^= (a2); \  # 对 a3 和 a2 进行按位异或操作，并将结果赋给 a3
        (a2) &= (a0); \  # 对 a2 和 a0 进行按位与操作，并将结果赋给 a2
        (a0) = SPH_T ## width(~(a0)); \  # 对 a0 按位取反，并将结果赋给 a0
        (a2) ^= (a1); \  # 对 a2 和 a1 进行按位异或操作，并将结果赋给 a2
        (a1) |= (a3); \  # 对 a1 和 a3 进行按位或操作，并将结果赋给 a1
        tmp ^= (a1); \  # 对临时变量和 a1 进行按位异或操作，并将结果赋给临时变量
        (a3) ^= (a2); \  # 对 a3 和 a2 进行按位异或操作，并将结果赋给 a3
        (a2) &= (a1); \  # 对 a2 和 a1 进行按位与操作，并将结果赋给 a2
        (a1) ^= (a0); \  # 对 a1 和 a0 进行按位异或操作，并将结果赋给 a1
        (a0) = tmp; \  # 将临时变量的值赋给 a0
    } while (0)  # 宏定义结束

#define SUB_CRUMB(a0, a1, a2, a3)    SUB_CRUMB_GEN(a0, a1, a2, a3, 32)  # 定义一个宏，调用上面定义的宏，传入参数和宽度
#define SUB_CRUMBW(a0, a1, a2, a3)   SUB_CRUMB_GEN(a0, a1, a2, a3, 64)  # 定义一个宏，调用上面定义的宏，传入参数和宽度

#if 0  # 如果条件为假

#define ROL32W(x, n)   SPH_T64( \  # 定义一个宏，实现对参数进行循环左移操作
                       (((x) << (n)) \  # 对参数进行左移操作
                       & ~((SPH_C64(0xFFFFFFFF) >> (32 - (n))) << 32)) \  # 对参数进行按位取反和位与操作
                       | (((x) >> (32 - (n))) \  # 对参数进行右移操作
                       & ~((SPH_C64(0xFFFFFFFF) >> (n)) << (n))))  # 对参数进行按位取反和位与操作

#define MIX_WORDW(u, v)   do { \  # 定义一个宏，实现对两个参数进行一系列位运算操作
        (v) ^= (u); \  # 对 v 和 u 进行按位异或操作，并将结果赋给 v
        (u) = ROL32W((u), 2) ^ (v); \  # 对 u 进行循环左移操作，然后与 v 进行按位异或操作，并将结果赋给 u
        (v) = ROL32W((v), 14) ^ (u); \  # 对 v 进行循环左移操作，然后与 u 进行按位异或操作，并将结果赋给 v
        (u) = ROL32W((u), 10) ^ (v); \  # 对 u 进行循环左移操作，然后与 v 进行按位异或操作，并将结果赋给 u
        (v) = ROL32W((v), 1); \  # 对 v 进行循环左移操作，并将结果赋给 v
    } while (0)  # 宏定义结束

#endif  # 条件结束
#define MIX_WORDW(u, v)   do { \
        sph_u32 ul, uh, vl, vh; \  # 定义四个32位无符号整数变量ul, uh, vl, vh
        (v) ^= (u); \  # 将v与u异或
        ul = SPH_T32((sph_u32)(u)); \  # 将u转换为32位无符号整数，并赋值给ul
        uh = SPH_T32((sph_u32)((u) >> 32)); \  # 将u右移32位，再转换为32位无符号整数，并赋值给uh
        vl = SPH_T32((sph_u32)(v)); \  # 将v转换为32位无符号整数，并赋值给vl
        vh = SPH_T32((sph_u32)((v) >> 32)); \  # 将v右移32位，再转换为32位无符号整数，并赋值给vh
        ul = SPH_ROTL32(ul, 2) ^ vl; \  # 将ul左循环移位2位，并与vl异或
        vl = SPH_ROTL32(vl, 14) ^ ul; \  # 将vl左循环移位14位，并与ul异或
        ul = SPH_ROTL32(ul, 10) ^ vl; \  # 将ul左循环移位10位，并与vl异或
        vl = SPH_ROTL32(vl, 1); \  # 将vl左循环移位1位
        uh = SPH_ROTL32(uh, 2) ^ vh; \  # 将uh左循环移位2位，并与vh异或
        vh = SPH_ROTL32(vh, 14) ^ uh; \  # 将vh左循环移位14位，并与uh异或
        uh = SPH_ROTL32(uh, 10) ^ vh; \  # 将uh左循环移位10位，并与vh异或
        vh = SPH_ROTL32(vh, 1); \  # 将vh左循环移位1位
        (u) = (sph_u64)ul | ((sph_u64)uh << 32); \  # 将ul转换为64位无符号整数，将uh左移32位后转换为64位无符号整数，再进行或运算，并赋值给u
        (v) = (sph_u64)vl | ((sph_u64)vh << 32); \  # 将vl转换为64位无符号整数，将vh左移32位后转换为64位无符号整数，再进行或运算，并赋值给v
    } while (0)  # do-while循环，循环体只执行一次

#else

#define SUB_CRUMB(a0, a1, a2, a3)   do { \  # 定义一个宏，接收四个参数a0, a1, a2, a3
        sph_u32 tmp; \  # 定义一个32位无符号整数变量tmp
        tmp = (a0); \  # 将a0赋值给tmp
        (a0) |= (a1); \  # 将a0与a1进行或运算，并赋值给a0
        (a2) ^= (a3); \  # 将a2与a3进行异或运算，并赋值给a2
        (a1) = SPH_T32(~(a1)); \  # 将a1按位取反，并转换为32位无符号整数，并赋值给a1
        (a0) ^= (a3); \  # 将a0与a3进行异或运算，并赋值给a0
        (a3) &= tmp; \  # 将a3与tmp进行与运算，并赋值给a3
        (a1) ^= (a3); \  # 将a1与a3进行异或运算，并赋值给a1
        (a3) ^= (a2); \  # 将a3与a2进行异或运算，并赋值给a3
        (a2) &= (a0); \  # 将a2与a0进行与运算，并赋值给a2
        (a0) = SPH_T32(~(a0)); \  # 将a0按位取反，并转换为32位无符号整数，并赋值给a0
        (a2) ^= (a1); \  # 将a2与a1进行异或运算，并赋值给a2
        (a1) |= (a3); \  # 将a1与a3进行或运算，并赋值给a1
        tmp ^= (a1); \  # 将tmp与a1进行异或运算，并赋值给tmp
        (a3) ^= (a2); \  # 将a3与a2进行异或运算，并赋值给a3
        (a2) &= (a1); \  # 将a2与a1进行与运算，并赋值给a2
        (a1) ^= (a0); \  # 将a1与a0进行异或运算，并赋值给a1
        (a0) = tmp; \  # 将tmp赋值给a0
    } while (0)  # do-while循环，循环体只执行一次

#endif

#define MIX_WORD(u, v)   do { \  # 定义一个宏，接收两个参数u, v
        (v) ^= (u); \  # 将v与u进行异或运算，并赋值给v
        (u) = SPH_ROTL32((u), 2) ^ (v); \  # 将u左循环移位2位，并与v进行异或运算，并赋值给u
        (v) = SPH_ROTL32((v), 14) ^ (u); \  # 将v左循环移位14位，并与u进行异或运算，并赋值给v
        (u) = SPH_ROTL32((u), 10) ^ (v); \  # 将u左循环移位10位，并与v进行异或运算，并赋值给u
        (v) = SPH_ROTL32((v), 1); \  # 将v左循环移位1位
    } while (0)  # do-while循环，循环体只执行一次

#define DECL_STATE3 \  # 定义一个宏
    sph_u32 V00, V01, V02, V03, V04, V05, V06, V07; \  # 声明8个32位无符号整数变量
    sph_u32 V10, V11, V12, V13, V14, V15, V16, V17; \  # 声明8个32位无符号整数变量
    sph_u32 V20, V21, V22, V23, V24, V25, V26, V27;  # 声明8个32位无符号整数变量
# 定义宏，用于将状态中的值赋给对应的变量
#define READ_STATE3(state)   do { \
        # 读取状态中的值到对应的变量
        V00 = (state)->V[0][0]; \
        V01 = (state)->V[0][1]; \
        V02 = (state)->V[0][2]; \
        V03 = (state)->V[0][3]; \
        V04 = (state)->V[0][4]; \
        V05 = (state)->V[0][5]; \
        V06 = (state)->V[0][6]; \
        V07 = (state)->V[0][7]; \
        V10 = (state)->V[1][0]; \
        V11 = (state)->V[1][1]; \
        V12 = (state)->V[1][2]; \
        V13 = (state)->V[1][3]; \
        V14 = (state)->V[1][4]; \
        V15 = (state)->V[1][5]; \
        V16 = (state)->V[1][6]; \
        V17 = (state)->V[1][7]; \
        V20 = (state)->V[2][0]; \
        V21 = (state)->V[2][1]; \
        V22 = (state)->V[2][2]; \
        V23 = (state)->V[2][3]; \
        V24 = (state)->V[2][4]; \
        V25 = (state)->V[2][5]; \
        V26 = (state)->V[2][6]; \
        V27 = (state)->V[2][7]; \
    } while (0)

# 定义宏，用于将变量的值赋给状态中对应的位置
#define WRITE_STATE3(state)   do { \
        # 将变量的值赋给状态中对应的位置
        (state)->V[0][0] = V00; \
        (state)->V[0][1] = V01; \
        (state)->V[0][2] = V02; \
        (state)->V[0][3] = V03; \
        (state)->V[0][4] = V04; \
        (state)->V[0][5] = V05; \
        (state)->V[0][6] = V06; \
        (state)->V[0][7] = V07; \
        (state)->V[1][0] = V10; \
        (state)->V[1][1] = V11; \
        (state)->V[1][2] = V12; \
        (state)->V[1][3] = V13; \
        (state)->V[1][4] = V14; \
        (state)->V[1][5] = V15; \
        (state)->V[1][6] = V16; \
        (state)->V[1][7] = V17; \
        (state)->V[2][0] = V20; \
        (state)->V[2][1] = V21; \
        (state)->V[2][2] = V22; \
        (state)->V[2][3] = V23; \
        (state)->V[2][4] = V24; \
        (state)->V[2][5] = V25; \
        (state)->V[2][6] = V26; \
        (state)->V[2][7] = V27; \
    } while (0)
# 定义 MI3 宏，用于处理输入数据
#define MI3   do { \
        # 定义临时变量 M 和 a
        DECL_TMP8(M) \
        DECL_TMP8(a) \
        # 从输入缓冲区中读取 32 位数据，并按大端序转换为主机字节序
        M0 = sph_dec32be_aligned(buf +  0); \
        M1 = sph_dec32be_aligned(buf +  4); \
        M2 = sph_dec32be_aligned(buf +  8); \
        M3 = sph_dec32be_aligned(buf + 12); \
        M4 = sph_dec32be_aligned(buf + 16); \
        M5 = sph_dec32be_aligned(buf + 20); \
        M6 = sph_dec32be_aligned(buf + 24); \
        M7 = sph_dec32be_aligned(buf + 28); \
        # 将 V0 和 V1 异或结果保存到 a
        XOR(a, V0, V1); \
        # 将 a 和 V2 异或结果保存到 a
        XOR(a, a, V2); \
        # 对 a 进行平方运算
        M2(a, a); \
        # 将 a 和 V0 异或结果保存到 V0
        XOR(V0, a, V0); \
        # 将 V0 和 M 异或结果保存到 V0
        XOR(V0, M, V0); \
        # 对 M 进行平方运算
        M2(M, M); \
        # 将 a 和 V1 异或结果保存到 V1
        XOR(V1, a, V1); \
        # 将 V1 和 M 异或结果保存到 V1
        XOR(V1, M, V1); \
        # 对 M 进行平方运算
        M2(M, M); \
        # 将 a 和 V2 异或结果保存到 V2
        XOR(V2, a, V2); \
        # 将 V2 和 M 异或结果保存到 V2
        XOR(V2, M, V2); \
    } while (0)

# 定义 TWEAK3 宏，用于处理 Tweak
#define TWEAK3   do { \
        # 将 V14、V15、V16、V17 向左循环移位 1 位
        V14 = SPH_ROTL32(V14, 1); \
        V15 = SPH_ROTL32(V15, 1); \
        V16 = SPH_ROTL32(V16, 1); \
        V17 = SPH_ROTL32(V17, 1); \
        # 将 V24、V25、V26、V27 向左循环移位 2 位
        V24 = SPH_ROTL32(V24, 2); \
        V25 = SPH_ROTL32(V25, 2); \
        V26 = SPH_ROTL32(V26, 2); \
        V27 = SPH_ROTL32(V27, 2); \
    } while (0)

#if SPH_LUFFA_PARALLEL
// 定义 P3 宏，用于执行一系列操作
#define P3   do { \
        int r; \  // 定义整型变量 r
        sph_u64 W0, W1, W2, W3, W4, W5, W6, W7; \  // 定义 8 个 64 位整型变量
        TWEAK3; \  // 调用 TWEAK3 宏
        W0 = (sph_u64)V00 | ((sph_u64)V10 << 32); \  // 将 V00 和 V10 组合成 64 位整数赋值给 W0
        W1 = (sph_u64)V01 | ((sph_u64)V11 << 32); \  // 将 V01 和 V11 组合成 64 位整数赋值给 W1
        W2 = (sph_u64)V02 | ((sph_u64)V12 << 32); \  // 将 V02 和 V12 组合成 64 位整数赋值给 W2
        W3 = (sph_u64)V03 | ((sph_u64)V13 << 32); \  // 将 V03 和 V13 组合成 64 位整数赋值给 W3
        W4 = (sph_u64)V04 | ((sph_u64)V14 << 32); \  // 将 V04 和 V14 组合成 64 位整数赋值给 W4
        W5 = (sph_u64)V05 | ((sph_u64)V15 << 32); \  // 将 V05 和 V15 组合成 64 位整数赋值给 W5
        W6 = (sph_u64)V06 | ((sph_u64)V16 << 32); \  // 将 V06 和 V16 组合成 64 位整数赋值给 W6
        W7 = (sph_u64)V07 | ((sph_u64)V17 << 32); \  // 将 V07 和 V17 组合成 64 位整数赋值给 W7
        for (r = 0; r < 8; r ++) { \  // 循环 8 次
            SUB_CRUMBW(W0, W1, W2, W3); \  // 调用 SUB_CRUMBW 宏
            SUB_CRUMBW(W5, W6, W7, W4); \  // 调用 SUB_CRUMBW 宏
            MIX_WORDW(W0, W4); \  // 调用 MIX_WORDW 宏
            MIX_WORDW(W1, W5); \  // 调用 MIX_WORDW 宏
            MIX_WORDW(W2, W6); \  // 调用 MIX_WORDW 宏
            MIX_WORDW(W3, W7); \  // 调用 MIX_WORDW 宏
            W0 ^= RCW010[r]; \  // 对 W0 进行异或操作
            W4 ^= RCW014[r]; \  // 对 W4 进行异或操作
        } \
        V00 = SPH_T32((sph_u32)W0); \  // 将 W0 的低 32 位赋值给 V00
        V10 = SPH_T32((sph_u32)(W0 >> 32)); \  // 将 W0 的高 32 位赋值给 V10
        V01 = SPH_T32((sph_u32)W1); \  // 将 W1 的低 32 位赋值给 V01
        V11 = SPH_T32((sph_u32)(W1 >> 32)); \  // 将 W1 的高 32 位赋值给 V11
        V02 = SPH_T32((sph_u32)W2); \  // 将 W2 的低 32 位赋值给 V02
        V12 = SPH_T32((sph_u32)(W2 >> 32)); \  // 将 W2 的高 32 位赋值给 V12
        V03 = SPH_T32((sph_u32)W3); \  // 将 W3 的低 32 位赋值给 V03
        V13 = SPH_T32((sph_u32)(W3 >> 32)); \  // 将 W3 的高 32 位赋值给 V13
        V04 = SPH_T32((sph_u32)W4); \  // 将 W4 的低 32 位赋值给 V04
        V14 = SPH_T32((sph_u32)(W4 >> 32)); \  // 将 W4 的高 32 位赋值给 V14
        V05 = SPH_T32((sph_u32)W5); \  // 将 W5 的低 32 位赋值给 V05
        V15 = SPH_T32((sph_u32)(W5 >> 32)); \  // 将 W5 的高 32 位赋值给 V15
        V06 = SPH_T32((sph_u32)W6); \  // 将 W6 的低 32 位赋值给 V06
        V16 = SPH_T32((sph_u32)(W6 >> 32)); \  // 将 W6 的高 32 位赋值给 V16
        V07 = SPH_T32((sph_u32)W7); \  // 将 W7 的低 32 位赋值给 V07
        V17 = SPH_T32((sph_u32)(W7 >> 32)); \  // 将 W7 的高 32 位赋值给 V17
        for (r = 0; r < 8; r ++) { \  // 循环 8 次
            SUB_CRUMB(V20, V21, V22, V23); \  // 调用 SUB_CRUMB 宏
            SUB_CRUMB(V25, V26, V27, V24); \  // 调用 SUB_CRUMB 宏
            MIX_WORD(V20, V24); \  // 调用 MIX_WORD 宏
            MIX_WORD(V21, V25); \  // 调用 MIX_WORD 宏
            MIX_WORD(V22, V26); \  // 调用 MIX_WORD 宏
            MIX_WORD(V23, V27); \  // 调用 MIX_WORD 宏
            V20 ^= RC20[r]; \  // 对 V20 进行异或操作
            V24 ^= RC24[r]; \  // 对 V24 进行异或操作
        } \
    } while (0)  // 结束 P3 宏定义
#define P3   do { \
        int r; \
        TWEAK3; \
        for (r = 0; r < 8; r ++) { \
            SUB_CRUMB(V00, V01, V02, V03); \
            SUB_CRUMB(V05, V06, V07, V04); \
            MIX_WORD(V00, V04); \
            MIX_WORD(V01, V05); \
            MIX_WORD(V02, V06); \
            MIX_WORD(V03, V07); \
            V00 ^= RC00[r]; \
            V04 ^= RC04[r]; \
        } \
        for (r = 0; r < 8; r ++) { \
            SUB_CRUMB(V10, V11, V12, V13); \
            SUB_CRUMB(V15, V16, V17, V14); \
            MIX_WORD(V10, V14); \
            MIX_WORD(V11, V15); \
            MIX_WORD(V12, V16); \
            MIX_WORD(V13, V17); \
            V10 ^= RC10[r]; \
            V14 ^= RC14[r]; \
        } \
        for (r = 0; r < 8; r ++) { \
            SUB_CRUMB(V20, V21, V22, V23); \
            SUB_CRUMB(V25, V26, V27, V24); \
            MIX_WORD(V20, V24); \
            MIX_WORD(V21, V25); \
            MIX_WORD(V22, V26); \
            MIX_WORD(V23, V27); \
            V20 ^= RC20[r]; \
            V24 ^= RC24[r]; \
        } \
    } while (0)

这段代码定义了一个宏P3，它包含了一系列的操作，用于对一组变量进行处理。


#define DECL_STATE4 \
    sph_u32 V00, V01, V02, V03, V04, V05, V06, V07; \
    sph_u32 V10, V11, V12, V13, V14, V15, V16, V17; \
    sph_u32 V20, V21, V22, V23, V24, V25, V26, V27; \
    sph_u32 V30, V31, V32, V33, V34, V35, V36, V37;

这段代码定义了一个宏DECL_STATE4，它声明了一组变量V00到V37，这些变量的类型是sph_u32。
# 定义宏，用于将状态结构体中的数据分别赋值给对应的变量
#define READ_STATE4(state)   do { \
        # 读取状态结构体中第一行的8个数据
        V00 = (state)->V[0][0]; \
        V01 = (state)->V[0][1]; \
        V02 = (state)->V[0][2]; \
        V03 = (state)->V[0][3]; \
        V04 = (state)->V[0][4]; \
        V05 = (state)->V[0][5]; \
        V06 = (state)->V[0][6]; \
        V07 = (state)->V[0][7]; \
        # 读取状态结构体中第二行的8个数据
        V10 = (state)->V[1][0]; \
        V11 = (state)->V[1][1]; \
        V12 = (state)->V[1][2]; \
        V13 = (state)->V[1][3]; \
        V14 = (state)->V[1][4]; \
        V15 = (state)->V[1][5]; \
        V16 = (state)->V[1][6]; \
        V17 = (state)->V[1][7]; \
        # 读取状态结构体中第三行的8个数据
        V20 = (state)->V[2][0]; \
        V21 = (state)->V[2][1]; \
        V22 = (state)->V[2][2]; \
        V23 = (state)->V[2][3]; \
        V24 = (state)->V[2][4]; \
        V25 = (state)->V[2][5]; \
        V26 = (state)->V[2][6]; \
        V27 = (state)->V[2][7]; \
        # 读取状态结构体中第四行的8个数据
        V30 = (state)->V[3][0]; \
        V31 = (state)->V[3][1]; \
        V32 = (state)->V[3][2]; \
        V33 = (state)->V[3][3]; \
        V34 = (state)->V[3][4]; \
        V35 = (state)->V[3][5]; \
        V36 = (state)->V[3][6]; \
        V37 = (state)->V[3][7]; \
    } while (0)
# 定义宏，用于将状态中的值写入到状态对象中
#define WRITE_STATE4(state)   do { \
        # 将状态中的值写入到状态对象中的对应位置
        (state)->V[0][0] = V00; \
        (state)->V[0][1] = V01; \
        (state)->V[0][2] = V02; \
        (state)->V[0][3] = V03; \
        (state)->V[0][4] = V04; \
        (state)->V[0][5] = V05; \
        (state)->V[0][6] = V06; \
        (state)->V[0][7] = V07; \
        (state)->V[1][0] = V10; \
        (state)->V[1][1] = V11; \
        (state)->V[1][2] = V12; \
        (state)->V[1][3] = V13; \
        (state)->V[1][4] = V14; \
        (state)->V[1][5] = V15; \
        (state)->V[1][6] = V16; \
        (state)->V[1][7] = V17; \
        (state)->V[2][0] = V20; \
        (state)->V[2][1] = V21; \
        (state)->V[2][2] = V22; \
        (state)->V[2][3] = V23; \
        (state)->V[2][4] = V24; \
        (state)->V[2][5] = V25; \
        (state)->V[2][6] = V26; \
        (state)->V[2][7] = V27; \
        (state)->V[3][0] = V30; \
        (state)->V[3][1] = V31; \
        (state)->V[3][2] = V32; \
        (state)->V[3][3] = V33; \
        (state)->V[3][4] = V34; \
        (state)->V[3][5] = V35; \
        (state)->V[3][6] = V36; \
        (state)->V[3][7] = V37; \
    } while (0)
# 定义 MI4 宏，用于进行一系列的位运算操作
#define MI4   do { \
        # 定义临时变量 M、a、b
        DECL_TMP8(M) \
        DECL_TMP8(a) \
        DECL_TMP8(b) \
        # 从缓冲区中读取32位的数据，并按大端序转换为无符号整数
        M0 = sph_dec32be_aligned(buf +  0); \
        M1 = sph_dec32be_aligned(buf +  4); \
        M2 = sph_dec32be_aligned(buf +  8); \
        M3 = sph_dec32be_aligned(buf + 12); \
        M4 = sph_dec32be_aligned(buf + 16); \
        M5 = sph_dec32be_aligned(buf + 20); \
        M6 = sph_dec32be_aligned(buf + 24); \
        M7 = sph_dec32be_aligned(buf + 28); \
        # 对 V0 和 V1 进行异或操作，并将结果保存到 a 中
        XOR(a, V0, V1); \
        # 对 V2 和 V3 进行异或操作，并将结果保存到 b 中
        XOR(b, V2, V3); \
        # 对 a 和 b 进行异或操作，并将结果保存到 a 中
        XOR(a, a, b); \
        # 对 a 进行平方运算，并将结果保存到 a 中
        M2(a, a); \
        # 对 V0 和 a 进行异或操作，并将结果保存到 V0 中
        XOR(V0, a, V0); \
        # 对 V1 和 a 进行异或操作，并将结果保存到 V1 中
        XOR(V1, a, V1); \
        # 对 V2 和 a 进行异或操作，并将结果保存到 V2 中
        XOR(V2, a, V2); \
        # 对 V3 和 a 进行异或操作，并将结果保存到 V3 中
        XOR(V3, a, V3); \
        # 对 V0 和 b 进行平方运算，并将结果保存到 b 中
        M2(b, V0); \
        # 对 b 和 V3 进行异或操作，并将结果保存到 b 中
        XOR(b, b, V3); \
        # 对 V3 进行平方运算，并将结果保存到 V3 中
        M2(V3, V3); \
        # 对 V3 和 V2 进行异或操作，并将结果保存到 V3 中
        XOR(V3, V3, V2); \
        # 对 V2 进行平方运算，并将结果保存到 V2 中
        M2(V2, V2); \
        # 对 V2 和 V1 进行异或操作，并将结果保存到 V2 中
        XOR(V2, V2, V1); \
        # 对 V1 进行平方运算，并将结果保存到 V1 中
        M2(V1, V1); \
        # 对 V1 和 V0 进行异或操作，并将结果保存到 V1 中
        XOR(V1, V1, V0); \
        # 对 b 和 M 进行异或操作，并将结果保存到 V0 中
        XOR(V0, b, M); \
        # 对 M 进行平方运算，并将结果保存到 M 中
        M2(M, M); \
        # 对 V1 和 M 进行异或操作，并将结果保存到 V1 中
        XOR(V1, V1, M); \
        # 对 M 进行平方运算，并将结果保存到 M 中
        M2(M, M); \
        # 对 V2 和 M 进行异或操作，并将结果保存到 V2 中
        XOR(V2, V2, M); \
        # 对 M 进行平方运算，并将结果保存到 M 中
        M2(M, M); \
        # 对 V3 和 M 进行异或操作，并将结果保存到 V3 中
        XOR(V3, V3, M); \
    } while (0)

#define TWEAK4   do { \
        # 对 V14、V15、V16、V17 进行循环左移1位
        V14 = SPH_ROTL32(V14, 1); \
        V15 = SPH_ROTL32(V15, 1); \
        V16 = SPH_ROTL32(V16, 1); \
        V17 = SPH_ROTL32(V17, 1); \
        # 对 V24、V25、V26、V27 进行循环左移2位
        V24 = SPH_ROTL32(V24, 2); \
        V25 = SPH_ROTL32(V25, 2); \
        V26 = SPH_ROTL32(V26, 2); \
        V27 = SPH_ROTL32(V27, 2); \
        # 对 V34、V35、V36、V37 进行循环左移3位
        V34 = SPH_ROTL32(V34, 3); \
        V35 = SPH_ROTL32(V35, 3); \
        V36 = SPH_ROTL32(V36, 3); \
        V37 = SPH_ROTL32(V37, 3); \
    } while (0)

#if SPH_LUFFA_PARALLEL

    } while (0)

#else
# 定义宏 P4，用于进行一系列操作
#define P4   do { \
        int r; \  # 定义变量 r，用于循环计数
        TWEAK4; \  # 调用宏 TWEAK4，进行一些操作
        for (r = 0; r < 8; r ++) { \  # 循环8次
            SUB_CRUMB(V00, V01, V02, V03); \  # 调用宏 SUB_CRUMB，进行一些操作
            SUB_CRUMB(V05, V06, V07, V04); \  # 调用宏 SUB_CRUMB，进行一些操作
            MIX_WORD(V00, V04); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V01, V05); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V02, V06); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V03, V07); \  # 调用宏 MIX_WORD，进行一些操作
            V00 ^= RC00[r]; \  # 对 V00 进行异或操作
            V04 ^= RC04[r]; \  # 对 V04 进行异或操作
        } \
        for (r = 0; r < 8; r ++) { \  # 循环8次
            SUB_CRUMB(V10, V11, V12, V13); \  # 调用宏 SUB_CRUMB，进行一些操作
            SUB_CRUMB(V15, V16, V17, V14); \  # 调用宏 SUB_CRUMB，进行一些操作
            MIX_WORD(V10, V14); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V11, V15); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V12, V16); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V13, V17); \  # 调用宏 MIX_WORD，进行一些操作
            V10 ^= RC10[r]; \  # 对 V10 进行异或操作
            V14 ^= RC14[r]; \  # 对 V14 进行异或操作
        } \
        for (r = 0; r < 8; r ++) { \  # 循环8次
            SUB_CRUMB(V20, V21, V22, V23); \  # 调用宏 SUB_CRUMB，进行一些操作
            SUB_CRUMB(V25, V26, V27, V24); \  # 调用宏 SUB_CRUMB，进行一些操作
            MIX_WORD(V20, V24); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V21, V25); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V22, V26); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V23, V27); \  # 调用宏 MIX_WORD，进行一些操作
            V20 ^= RC20[r]; \  # 对 V20 进行异或操作
            V24 ^= RC24[r]; \  # 对 V24 进行异或操作
        } \
        for (r = 0; r < 8; r ++) { \  # 循环8次
            SUB_CRUMB(V30, V31, V32, V33); \  # 调用宏 SUB_CRUMB，进行一些操作
            SUB_CRUMB(V35, V36, V37, V34); \  # 调用宏 SUB_CRUMB，进行一些操作
            MIX_WORD(V30, V34); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V31, V35); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V32, V36); \  # 调用宏 MIX_WORD，进行一些操作
            MIX_WORD(V33, V37); \  # 调用宏 MIX_WORD，进行一些操作
            V30 ^= RC30[r]; \  # 对 V30 进行异或操作
            V34 ^= RC34[r]; \  # 对 V34 进行异或操作
        } \
    } while (0)  # 结束宏 P4 的定义

#endif  # 结束宏 DECL_STATE5 的定义

#define DECL_STATE5 \  # 定义宏 DECL_STATE5
    sph_u32 V00, V01, V02, V03, V04, V05, V06, V07; \  # 定义变量 V00-V07
    sph_u32 V10, V11, V12, V13, V14, V15, V16, V17; \  # 定义变量 V10-V17
    sph_u32 V20, V21, V22, V23, V24, V25, V26, V27; \  # 定义变量 V20-V27
    sph_u32 V30, V31, V32, V33, V34, V35, V36, V37; \  # 定义变量 V30-V37
    sph_u32 V40, V41, V42, V43, V44, V45, V46, V47; \  # 定义变量 V40-V47
# 定义宏，用于将状态中的值分别赋给对应的变量
#define READ_STATE5(state)   do { \
        # 读取状态中第一行的值
        V00 = (state)->V[0][0]; \
        V01 = (state)->V[0][1]; \
        V02 = (state)->V[0][2]; \
        V03 = (state)->V[0][3]; \
        V04 = (state)->V[0][4]; \
        V05 = (state)->V[0][5]; \
        V06 = (state)->V[0][6]; \
        V07 = (state)->V[0][7]; \
        # 读取状态中第二行的值
        V10 = (state)->V[1][0]; \
        V11 = (state)->V[1][1]; \
        V12 = (state)->V[1][2]; \
        V13 = (state)->V[1][3]; \
        V14 = (state)->V[1][4]; \
        V15 = (state)->V[1][5]; \
        V16 = (state)->V[1][6]; \
        V17 = (state)->V[1][7]; \
        # 读取状态中第三行的值
        V20 = (state)->V[2][0]; \
        V21 = (state)->V[2][1]; \
        V22 = (state)->V[2][2]; \
        V23 = (state)->V[2][3]; \
        V24 = (state)->V[2][4]; \
        V25 = (state)->V[2][5]; \
        V26 = (state)->V[2][6]; \
        V27 = (state)->V[2][7]; \
        # 读取状态中第四行的值
        V30 = (state)->V[3][0]; \
        V31 = (state)->V[3][1]; \
        V32 = (state)->V[3][2]; \
        V33 = (state)->V[3][3]; \
        V34 = (state)->V[3][4]; \
        V35 = (state)->V[3][5]; \
        V36 = (state)->V[3][6]; \
        V37 = (state)->V[3][7]; \
        # 读取状态中第五行的值
        V40 = (state)->V[4][0]; \
        V41 = (state)->V[4][1]; \
        V42 = (state)->V[4][2]; \
        V43 = (state)->V[4][3]; \
        V44 = (state)->V[4][4]; \
        V45 = (state)->V[4][5]; \
        V46 = (state)->V[4][6]; \
        V47 = (state)->V[4][7]; \
    } while (0)
# 定义宏，用于将状态数组中的值赋给状态对象
#define WRITE_STATE5(state)   do { \
        # 将状态数组中的值赋给状态对象的对应位置
        (state)->V[0][0] = V00; \
        (state)->V[0][1] = V01; \
        (state)->V[0][2] = V02; \
        (state)->V[0][3] = V03; \
        (state)->V[0][4] = V04; \
        (state)->V[0][5] = V05; \
        (state)->V[0][6] = V06; \
        (state)->V[0][7] = V07; \
        (state)->V[1][0] = V10; \
        (state)->V[1][1] = V11; \
        (state)->V[1][2] = V12; \
        (state)->V[1][3] = V13; \
        (state)->V[1][4] = V14; \
        (state)->V[1][5] = V15; \
        (state)->V[1][6] = V16; \
        (state)->V[1][7] = V17; \
        (state)->V[2][0] = V20; \
        (state)->V[2][1] = V21; \
        (state)->V[2][2] = V22; \
        (state)->V[2][3] = V23; \
        (state)->V[2][4] = V24; \
        (state)->V[2][5] = V25; \
        (state)->V[2][6] = V26; \
        (state)->V[2][7] = V27; \
        (state)->V[3][0] = V30; \
        (state)->V[3][1] = V31; \
        (state)->V[3][2] = V32; \
        (state)->V[3][3] = V33; \
        (state)->V[3][4] = V34; \
        (state)->V[3][5] = V35; \
        (state)->V[3][6] = V36; \
        (state)->V[3][7] = V37; \
        (state)->V[4][0] = V40; \
        (state)->V[4][1] = V41; \
        (state)->V[4][2] = V42; \
        (state)->V[4][3] = V43; \
        (state)->V[4][4] = V44; \
        (state)->V[4][5] = V45; \
        (state)->V[4][6] = V46; \
        (state)->V[4][7] = V47; \
    } while (0)
# 定义 MI5 宏，用于执行一系列操作
#define MI5   do { \
    # 声明临时变量 M
    DECL_TMP8(M) \
    # 声明临时变量 a
    DECL_TMP8(a) \
    # 声明临时变量 b
    DECL_TMP8(b) \
    # 从缓冲区中读取 32 位大端序的数据，存入 M0
    M0 = sph_dec32be_aligned(buf +  0); \
    # 从缓冲区中读取 32 位大端序的数据，存入 M1
    M1 = sph_dec32be_aligned(buf +  4); \
    # 从缓冲区中读取 32 位大端序的数据，存入 M2
    M2 = sph_dec32be_aligned(buf +  8); \
    # 从缓冲区中读取 32 位大端序的数据，存入 M3
    M3 = sph_dec32be_aligned(buf + 12); \
    # 从缓冲区中读取 32 位大端序的数据，存入 M4
    M4 = sph_dec32be_aligned(buf + 16); \
    # 从缓冲区中读取 32 位大端序的数据，存入 M5
    M5 = sph_dec32be_aligned(buf + 20); \
    # 从缓冲区中读取 32 位大端序的数据，存入 M6
    M6 = sph_dec32be_aligned(buf + 24); \
    # 从缓冲区中读取 32 位大端序的数据，存入 M7
    M7 = sph_dec32be_aligned(buf + 28); \
    # 对 V0 和 V1 进行异或操作，结果存入 a
    XOR(a, V0, V1); \
    # 对 V2 和 V3 进行异或操作，结果存入 b
    XOR(b, V2, V3); \
    # 对 a 和 b 进行异或操作，结果存入 a
    XOR(a, a, b); \
    # 对 a 和 V4 进行异或操作，结果存入 a
    XOR(a, a, V4); \
    # 对 a 进行 M2 操作
    M2(a, a); \
    # 对 V0 和 a 进行异或操作，结果存入 V0
    XOR(V0, a, V0); \
    # 对 V1 和 a 进行异或操作，结果存入 V1
    XOR(V1, a, V1); \
    # 对 V2 和 a 进行异或操作，结果存入 V2
    XOR(V2, a, V2); \
    # 对 V3 和 a 进行异或操作，结果存入 V3
    XOR(V3, a, V3); \
    # 对 V4 和 a 进行异或操作，结果存入 V4
    XOR(V4, a, V4); \
    # 对 V0 进行 M2 操作，结果存入 b
    M2(b, V0); \
    # 对 b 和 V1 进行异或操作，结果存入 b
    XOR(b, b, V1); \
    # 对 V1 进行 M2 操作
    M2(V1, V1); \
    # 对 V1 和 V2 进行异或操作，结果存入 V1
    XOR(V1, V1, V2); \
    # 对 V2 进行 M2 操作
    M2(V2, V2); \
    # 对 V2 和 V3 进行异或操作，结果存入 V2
    XOR(V2, V2, V3); \
    # 对 V3 进行 M2 操作
    M2(V3, V3); \
    # 对 V3 和 V4 进行异或操作，结果存入 V3
    XOR(V3, V3, V4); \
    # 对 V4 进行 M2 操作
    M2(V4, V4); \
    # 对 V4 和 V0 进行异或操作，结果存入 V4
    XOR(V4, V4, V0); \
    # 对 b 进行 M2 操作，结果存入 V0
    M2(V0, b); \
    # 对 V0 和 V4 进行异或操作，结果存入 V0
    XOR(V0, V0, V4); \
    # 对 V4 进行 M2 操作
    M2(V4, V4); \
    # 对 V4 和 V3 进行异或操作，结果存入 V4
    XOR(V4, V4, V3); \
    # 对 V3 进行 M2 操作
    M2(V3, V3); \
    # 对 V3 和 V2 进行异或操作，结果存入 V3
    XOR(V3, V3, V2); \
    # 对 V2 进行 M2 操作
    M2(V2, V2); \
    # 对 V2 和 V1 进行异或操作，结果存入 V2
    XOR(V2, V2, V1); \
    # 对 V1 进行 M2 操作
    M2(V1, V1); \
    # 对 V1 和 b 进行异或操作，结果存入 V1
    XOR(V1, V1, b); \
    # 对 V0 和 M 进行异或操作，结果存入 V0
    XOR(V0, V0, M); \
    # 对 M 进行 M2 操作
    M2(M, M); \
    # 对 V1 和 M 进行异或操作，结果存入 V1
    XOR(V1, V1, M); \
    # 对 M 进行 M2 操作
    M2(M, M); \
    # 对 V2 和 M 进行异或操作，结果存入 V2
    XOR(V2, V2, M); \
    # 对 M 进行 M2 操作
    M2(M, M); \
    # 对 V3 和 M 进行异或操作，结果存入 V3
    XOR(V3, V3, M); \
    # 对 M 进行 M2 操作
    M2(M, M); \
    # 对 V4 和 M 进行异或操作，结果存入 V4
    XOR(V4, V4, M); \
} while (0)
# 定义一个宏，用于对变量进行位移操作
#define TWEAK5   do { \
        V14 = SPH_ROTL32(V14, 1); \
        V15 = SPH_ROTL32(V15, 1); \
        V16 = SPH_ROTL32(V16, 1); \
        V17 = SPH_ROTL32(V17, 1); \
        V24 = SPH_ROTL32(V24, 2); \
        V25 = SPH_ROTL32(V25, 2); \
        V26 = SPH_ROTL32(V26, 2); \
        V27 = SPH_ROTL32(V27, 2); \
        V34 = SPH_ROTL32(V34, 3); \
        V35 = SPH_ROTL32(V35, 3); \
        V36 = SPH_ROTL32(V36, 3); \
        V37 = SPH_ROTL32(V37, 3); \
        V44 = SPH_ROTL32(V44, 4); \
        V45 = SPH_ROTL32(V45, 4); \
        V46 = SPH_ROTL32(V46, 4); \
        V47 = SPH_ROTL32(V47, 4); \
    } while (0)

这段代码定义了一个宏，用于对变量进行位移操作。宏的名称是TWEAK5，它包含了一系列的位移操作，每个操作都是将变量进行左移位操作，并且移动的位数不同。这些变量的名称是V14、V15、V16、V17、V24、V25、V26、V27、V34、V35、V36、V37、V44、V45、V46、V47。这个宏的作用是将这些变量进行位移操作。


#if SPH_LUFFA_PARALLEL

    } while (0)

#else

这段代码是一个条件编译的语句，用于根据条件来选择不同的代码块。如果宏SPH_LUFFA_PARALLEL被定义了，那么就执行下面的代码块，否则执行#else后面的代码块。在这个例子中，由于没有给出完整的代码，所以无法确定具体的作用。
#define P5   do { \
        int r; \
        TWEAK5; \
        for (r = 0; r < 8; r ++) { \
            SUB_CRUMB(V00, V01, V02, V03); \  # 定义宏，将V00、V01、V02、V03传递给SUB_CRUMB函数
            SUB_CRUMB(V05, V06, V07, V04); \  # 定义宏，将V05、V06、V07、V04传递给SUB_CRUMB函数
            MIX_WORD(V00, V04); \  # 定义宏，将V00、V04传递给MIX_WORD函数
            MIX_WORD(V01, V05); \  # 定义宏，将V01、V05传递给MIX_WORD函数
            MIX_WORD(V02, V06); \  # 定义宏，将V02、V06传递给MIX_WORD函数
            MIX_WORD(V03, V07); \  # 定义宏，将V03、V07传递给MIX_WORD函数
            V00 ^= RC00[r]; \  # 将V00与RC00[r]进行异或运算
            V04 ^= RC04[r]; \  # 将V04与RC04[r]进行异或运算
        } \
        for (r = 0; r < 8; r ++) { \
            SUB_CRUMB(V10, V11, V12, V13); \  # 定义宏，将V10、V11、V12、V13传递给SUB_CRUMB函数
            SUB_CRUMB(V15, V16, V17, V14); \  # 定义宏，将V15、V16、V17、V14传递给SUB_CRUMB函数
            MIX_WORD(V10, V14); \  # 定义宏，将V10、V14传递给MIX_WORD函数
            MIX_WORD(V11, V15); \  # 定义宏，将V11、V15传递给MIX_WORD函数
            MIX_WORD(V12, V16); \  # 定义宏，将V12、V16传递给MIX_WORD函数
            MIX_WORD(V13, V17); \  # 定义宏，将V13、V17传递给MIX_WORD函数
            V10 ^= RC10[r]; \  # 将V10与RC10[r]进行异或运算
            V14 ^= RC14[r]; \  # 将V14与RC14[r]进行异或运算
        } \
        for (r = 0; r < 8; r ++) { \
            SUB_CRUMB(V20, V21, V22, V23); \  # 定义宏，将V20、V21、V22、V23传递给SUB_CRUMB函数
            SUB_CRUMB(V25, V26, V27, V24); \  # 定义宏，将V25、V26、V27、V24传递给SUB_CRUMB函数
            MIX_WORD(V20, V24); \  # 定义宏，将V20、V24传递给MIX_WORD函数
            MIX_WORD(V21, V25); \  # 定义宏，将V21、V25传递给MIX_WORD函数
            MIX_WORD(V22, V26); \  # 定义宏，将V22、V26传递给MIX_WORD函数
            MIX_WORD(V23, V27); \  # 定义宏，将V23、V27传递给MIX_WORD函数
            V20 ^= RC20[r]; \  # 将V20与RC20[r]进行异或运算
            V24 ^= RC24[r]; \  # 将V24与RC24[r]进行异或运算
        } \
        for (r = 0; r < 8; r ++) { \
            SUB_CRUMB(V30, V31, V32, V33); \  # 定义宏，将V30、V31、V32、V33传递给SUB_CRUMB函数
            SUB_CRUMB(V35, V36, V37, V34); \  # 定义宏，将V35、V36、V37、V34传递给SUB_CRUMB函数
            MIX_WORD(V30, V34); \  # 定义宏，将V30、V34传递给MIX_WORD函数
            MIX_WORD(V31, V35); \  # 定义宏，将V31、V35传递给MIX_WORD函数
            MIX_WORD(V32, V36); \  # 定义宏，将V32、V36传递给MIX_WORD函数
            MIX_WORD(V33, V37); \  # 定义宏，将V33、V37传递给MIX_WORD函数
            V30 ^= RC30[r]; \  # 将V30与RC30[r]进行异或运算
            V34 ^= RC34[r]; \  # 将V34与RC34[r]进行异或运算
        } \
        for (r = 0; r < 8; r ++) { \
            SUB_CRUMB(V40, V41, V42, V43); \  # 定义宏，将V40、V41、V42、V43传递给SUB_CRUMB函数
            SUB_CRUMB(V45, V46, V47, V44); \  # 定义宏，将V45、V46、V47、V44传递给SUB_CRUMB函数
            MIX_WORD(V40, V44); \  # 定义宏，将V40、V44传递给MIX_WORD函数
            MIX_WORD(V41, V45); \  # 定义宏，将V41、V45传递给MIX_WORD函数
            MIX_WORD(V42, V46); \  # 定义宏，将V42、V46传递给MIX_WORD函数
            MIX_WORD(V43, V47); \  # 定义宏，将V43、V47传递给MIX_WORD函数
            V40 ^= RC40[r]; \  # 将V40与RC40[r]进行异或运算
            V44 ^= RC44[r]; \  # 将V44与RC44[r]进行异或运算
        } \
    } while (0)  # 定义宏结束

#endif

static void
luffa3(sph_luffa224_context *sc, const void *data, size_t len)
{
    unsigned char *buf;  # 定义指向无符号字符的指针变量buf
    size_t ptr;  # 定义size_t类型的变量ptr
    DECL_STATE3  # 定义宏，声明STATE3

    buf = sc->buf;  # 将sc->buf的值赋给buf
    ptr = sc->ptr;  # 将sc->ptr的值赋给ptr
    # 如果剩余空间足够存放数据
    if (len < (sizeof sc->buf) - ptr) {
        # 将数据拷贝到缓冲区中
        memcpy(buf + ptr, data, len);
        # 更新指针位置
        ptr += len;
        sc->ptr = ptr;
        return;
    }

    # 读取状态3
    READ_STATE3(sc);
    # 当还有数据需要处理时
    while (len > 0) {
        size_t clen;

        # 计算剩余空间的大小
        clen = (sizeof sc->buf) - ptr;
        # 如果剩余空间大于数据长度，则只拷贝数据长度的内容
        if (clen > len)
            clen = len;
        # 将数据拷贝到缓冲区中
        memcpy(buf + ptr, data, clen);
        # 更新指针位置
        ptr += clen;
        # 更新数据指针位置
        data = (const unsigned char *)data + clen;
        # 更新剩余数据长度
        len -= clen;
        # 如果缓冲区已满
        if (ptr == sizeof sc->buf) {
            # 执行MI3操作
            MI3;
            # 执行P3操作
            P3;
            # 重置指针位置
            ptr = 0;
        }
    }
    # 写入状态3
    WRITE_STATE3(sc);
    # 更新指针位置
    sc->ptr = ptr;
# 关闭 Luffa-224 算法的上下文，输出结果到指定的目标内存
static void
luffa3_close(sph_luffa224_context *sc, unsigned ub, unsigned n,
    void *dst, unsigned out_size_w32)
{
    # 定义局部变量
    unsigned char *buf, *out;
    size_t ptr;
    unsigned z;
    int i;
    DECL_STATE3

    # 获取缓冲区和指针
    buf = sc->buf;
    ptr = sc->ptr;
    # 计算填充字节
    z = 0x80 >> n;
    buf[ptr ++] = ((ub & -z) | z) & 0xFF;
    # 填充剩余字节
    memset(buf + ptr, 0, (sizeof sc->buf) - ptr);
    # 读取状态
    READ_STATE3(sc);
    # 执行两轮压缩函数
    for (i = 0; i < 2; i ++) {
        MI3;
        P3;
        memset(buf, 0, sizeof sc->buf);
    }
    # 输出结果
    out = dst;
    sph_enc32be(out +  0, V00 ^ V10 ^ V20);
    sph_enc32be(out +  4, V01 ^ V11 ^ V21);
    sph_enc32be(out +  8, V02 ^ V12 ^ V22);
    sph_enc32be(out + 12, V03 ^ V13 ^ V23);
    sph_enc32be(out + 16, V04 ^ V14 ^ V24);
    sph_enc32be(out + 20, V05 ^ V15 ^ V25);
    sph_enc32be(out + 24, V06 ^ V16 ^ V26);
    # 如果输出结果长度大于7个字，继续输出
    if (out_size_w32 > 7)
        sph_enc32be(out + 28, V07 ^ V17 ^ V27);
}

# Luffa-384 算法的处理函数
static void
luffa4(sph_luffa384_context *sc, const void *data, size_t len)
{
    # 定义局部变量
    unsigned char *buf;
    size_t ptr;
    DECL_STATE4

    # 获取缓冲区和指针
    buf = sc->buf;
    ptr = sc->ptr;
    # 如果数据长度小于缓冲区剩余空间，直接拷贝数据到缓冲区
    if (len < (sizeof sc->buf) - ptr) {
        memcpy(buf + ptr, data, len);
        ptr += len;
        sc->ptr = ptr;
        return;
    }

    # 读取状态
    READ_STATE4(sc);
    # 循环处理数据
    while (len > 0) {
        size_t clen;

        clen = (sizeof sc->buf) - ptr;
        if (clen > len)
            clen = len;
        memcpy(buf + ptr, data, clen);
        ptr += clen;
        data = (const unsigned char *)data + clen;
        len -= clen;
        # 如果缓冲区已满，执行一轮压缩函数
        if (ptr == sizeof sc->buf) {
            MI4;
            P4;
            ptr = 0;
        }
    }
    # 输出状态
    WRITE_STATE4(sc);
    sc->ptr = ptr;
}

# 关闭 Luffa-384 算法的上下文，输出结果到指定的目标内存
static void
luffa4_close(sph_luffa384_context *sc, unsigned ub, unsigned n, void *dst)
{
    # 定义局部变量
    unsigned char *buf, *out;
    size_t ptr;
    unsigned z;
    int i;
    DECL_STATE4

    # 获取缓冲区和指针
    buf = sc->buf;
    ptr = sc->ptr;
    out = dst;
    # 计算填充字节
    z = 0x80 >> n;
    buf[ptr ++] = ((ub & -z) | z) & 0xFF;
    # 填充剩余字节
    memset(buf + ptr, 0, (sizeof sc->buf) - ptr);
    # 读取状态
    READ_STATE4(sc);
    # 循环3次，每次执行以下操作
    for (i = 0; i < 3; i ++) {
        # 执行 MI4 函数
        MI4;
        # 执行 P4 函数
        P4;
        # 根据 i 的值进行不同的操作
        switch (i) {
        # 当 i 为 0 时
        case 0:
            # 将 buf 数组的内容全部置为 0
            memset(buf, 0, sizeof sc->buf);
            break;
        # 当 i 为 1 时
        case 1:
            # 将 V00 ^ V10 ^ V20 ^ V30 的结果以大端字节序写入 out 数组的前4个字节
            sph_enc32be(out +  0, V00 ^ V10 ^ V20 ^ V30);
            # 将 V01 ^ V11 ^ V21 ^ V31 的结果以大端字节序写入 out 数组的第5到第8个字节
            sph_enc32be(out +  4, V01 ^ V11 ^ V21 ^ V31);
            # 将 V02 ^ V12 ^ V22 ^ V32 的结果以大端字节序写入 out 数组的第9到第12个字节
            sph_enc32be(out +  8, V02 ^ V12 ^ V22 ^ V32);
            # 将 V03 ^ V13 ^ V23 ^ V33 的结果以大端字节序写入 out 数组的第13到第16个字节
            sph_enc32be(out + 12, V03 ^ V13 ^ V23 ^ V33);
            # 将 V04 ^ V14 ^ V24 ^ V34 的结果以大端字节序写入 out 数组的第17到第20个字节
            sph_enc32be(out + 16, V04 ^ V14 ^ V24 ^ V34);
            # 将 V05 ^ V15 ^ V25 ^ V35 的结果以大端字节序写入 out 数组的第21到第24个字节
            sph_enc32be(out + 20, V05 ^ V15 ^ V25 ^ V35);
            # 将 V06 ^ V16 ^ V26 ^ V36 的结果以大端字节序写入 out 数组的第25到第28个字节
            sph_enc32be(out + 24, V06 ^ V16 ^ V26 ^ V36);
            # 将 V07 ^ V17 ^ V27 ^ V37 的结果以大端字节序写入 out 数组的第29到第32个字节
            sph_enc32be(out + 28, V07 ^ V17 ^ V27 ^ V37);
            break;
        # 当 i 为 2 时
        case 2:
            # 将 V00 ^ V10 ^ V20 ^ V30 的结果以大端字节序写入 out 数组的第33到第36个字节
            sph_enc32be(out + 32, V00 ^ V10 ^ V20 ^ V30);
            # 将 V01 ^ V11 ^ V21 ^ V31 的结果以大端字节序写入 out 数组的第37到第40个字节
            sph_enc32be(out + 36, V01 ^ V11 ^ V21 ^ V31);
            # 将 V02 ^ V12 ^ V22 ^ V32 的结果以大端字节序写入 out 数组的第41到第44个字节
            sph_enc32be(out + 40, V02 ^ V12 ^ V22 ^ V32);
            # 将 V03 ^ V13 ^ V23 ^ V33 的结果以大端字节序写入 out 数组的第45到第48个字节
            sph_enc32be(out + 44, V03 ^ V13 ^ V23 ^ V33);
            break;
        }
    }
# 关闭 luffa512 算法的上下文，处理最后的数据块
static void
luffa5(sph_luffa512_context *sc, const void *data, size_t len)
{
    unsigned char *buf;  # 定义一个指向无符号字符的指针 buf
    size_t ptr;  # 定义一个表示大小的变量 ptr
    DECL_STATE5  # 声明状态5

    buf = sc->buf;  # 将 sc->buf 的值赋给 buf
    ptr = sc->ptr;  # 将 sc->ptr 的值赋给 ptr
    if (len < (sizeof sc->buf) - ptr) {  # 如果 len 小于 (sc->buf 的大小 - ptr)
        memcpy(buf + ptr, data, len);  # 将 data 的内容复制到 buf + ptr 处，长度为 len
        ptr += len;  # ptr 增加 len
        sc->ptr = ptr;  # 将 ptr 的值赋给 sc->ptr
        return;  # 返回
    }

    READ_STATE5(sc);  # 读取状态5
    while (len > 0) {  # 当 len 大于 0 时循环
        size_t clen;  # 定义一个表示大小的变量 clen

        clen = (sizeof sc->buf) - ptr;  # 将 (sc->buf 的大小 - ptr) 的值赋给 clen
        if (clen > len)  # 如果 clen 大于 len
            clen = len;  # 将 len 的值赋给 clen
        memcpy(buf + ptr, data, clen);  # 将 data 的内容复制到 buf + ptr 处，长度为 clen
        ptr += clen;  # ptr 增加 clen
        data = (const unsigned char *)data + clen;  # 将 data 强制转换为无符号字符指针，加上 clen
        len -= clen;  # len 减去 clen
        if (ptr == sizeof sc->buf) {  # 如果 ptr 等于 sc->buf 的大小
            MI5;  # 调用 MI5 函数
            P5;  # 调用 P5 函数
            ptr = 0;  # 将 0 赋给 ptr
        }
    }
    WRITE_STATE5(sc);  # 写入状态5
    sc->ptr = ptr;  # 将 ptr 的值赋给 sc->ptr
}

# 关闭 luffa512 算法的上下文，处理最后的数据块
static void
luffa5_close(sph_luffa512_context *sc, unsigned ub, unsigned n, void *dst)
{
    unsigned char *buf, *out;  # 定义两个指向无符号字符的指针 buf 和 out
    size_t ptr;  # 定义一个表示大小的变量 ptr
    unsigned z;  # 定义一个无符号整数 z
    int i;  # 定义一个整数 i
    DECL_STATE5  # 声明状态5

    buf = sc->buf;  # 将 sc->buf 的值赋给 buf
    ptr = sc->ptr;  # 将 sc->ptr 的值赋给 ptr
    out = dst;  # 将 dst 的值赋给 out
    z = 0x80 >> n;  # 将 (0x80 右移 n 位) 的值赋给 z
    buf[ptr ++] = ((ub & -z) | z) & 0xFF;  # 将 ((ub 与 -z) 或 z) 与 0xFF 的值赋给 buf[ptr ++]
    memset(buf + ptr, 0, (sizeof sc->buf) - ptr);  # 将 buf + ptr 处开始的 (sc->buf 的大小 - ptr) 个字节设置为 0
    READ_STATE5(sc);  # 读取状态5
    # 循环3次，i的取值为0、1、2
    for (i = 0; i < 3; i ++) {
        # MI5函数的作用
        MI5;
        # P5函数的作用
        P5;
        # 根据i的值进行不同的操作
        switch (i) {
        # 当i为0时，执行以下操作
        case 0:
            # 将buf数组的前sizeof sc->buf个字节置为0
            memset(buf, 0, sizeof sc->buf);
            break;
        # 当i为1时，执行以下操作
        case 1:
            # 将V00 ^ V10 ^ V20 ^ V30 ^ V40的结果以大端字节序写入out数组的0~3字节
            sph_enc32be(out +  0, V00 ^ V10 ^ V20 ^ V30 ^ V40);
            # 将V01 ^ V11 ^ V21 ^ V31 ^ V41的结果以大端字节序写入out数组的4~7字节
            sph_enc32be(out +  4, V01 ^ V11 ^ V21 ^ V31 ^ V41);
            # 将V02 ^ V12 ^ V22 ^ V32 ^ V42的结果以大端字节序写入out数组的8~11字节
            sph_enc32be(out +  8, V02 ^ V12 ^ V22 ^ V32 ^ V42);
            # 将V03 ^ V13 ^ V23 ^ V33 ^ V43的结果以大端字节序写入out数组的12~15字节
            sph_enc32be(out + 12, V03 ^ V13 ^ V23 ^ V33 ^ V43);
            # 将V04 ^ V14 ^ V24 ^ V34 ^ V44的结果以大端字节序写入out数组的16~19字节
            sph_enc32be(out + 16, V04 ^ V14 ^ V24 ^ V34 ^ V44);
            # 将V05 ^ V15 ^ V25 ^ V35 ^ V45的结果以大端字节序写入out数组的20~23字节
            sph_enc32be(out + 20, V05 ^ V15 ^ V25 ^ V35 ^ V45);
            # 将V06 ^ V16 ^ V26 ^ V36 ^ V46的结果以大端字节序写入out数组的24~27字节
            sph_enc32be(out + 24, V06 ^ V16 ^ V26 ^ V36 ^ V46);
            # 将V07 ^ V17 ^ V27 ^ V37 ^ V47的结果以大端字节序写入out数组的28~31字节
            sph_enc32be(out + 28, V07 ^ V17 ^ V27 ^ V37 ^ V47);
            break;
        # 当i为2时，执行以下操作
        case 2:
            # 将V00 ^ V10 ^ V20 ^ V30 ^ V40的结果以大端字节序写入out数组的32~35字节
            sph_enc32be(out + 32, V00 ^ V10 ^ V20 ^ V30 ^ V40);
            # 将V01 ^ V11 ^ V21 ^ V31 ^ V41的结果以大端字节序写入out数组的36~39字节
            sph_enc32be(out + 36, V01 ^ V11 ^ V21 ^ V31 ^ V41);
            # 将V02 ^ V12 ^ V22 ^ V32 ^ V42的结果以大端字节序写入out数组的40~43字节
            sph_enc32be(out + 40, V02 ^ V12 ^ V22 ^ V32 ^ V42);
            # 将V03 ^ V13 ^ V23 ^ V33 ^ V43的结果以大端字节序写入out数组的44~47字节
            sph_enc32be(out + 44, V03 ^ V13 ^ V23 ^ V33 ^ V43);
            # 将V04 ^ V14 ^ V24 ^ V34 ^ V44的结果以大端字节序写入out数组的48~51字节
            sph_enc32be(out + 48, V04 ^ V14 ^ V24 ^ V34 ^ V44);
            # 将V05 ^ V15 ^ V25 ^ V35 ^ V45的结果以大端字节序写入out数组的52~55字节
            sph_enc32be(out + 52, V05 ^ V15 ^ V25 ^ V35 ^ V45);
            # 将V06 ^ V16 ^ V26 ^ V36 ^ V46的结果以大端字节序写入out数组的56~59字节
            sph_enc32be(out + 56, V06 ^ V16 ^ V26 ^ V36 ^ V46);
            # 将V07 ^ V17 ^ V27 ^ V37 ^ V47的结果以大端字节序写入out数组的60~63字节
            sph_enc32be(out + 60, V07 ^ V17 ^ V27 ^ V37 ^ V47);
            break;
        }
    }
/* see sph_luffa.h */
// 初始化 Luffa224 算法的上下文
void
sph_luffa224_init(void *cc)
{
    // 将上下文转换为 Luffa224 上下文类型
    sph_luffa224_context *sc;

    sc = cc;
    // 将初始向量 V_INIT 复制到上下文的 V 数组中
    memcpy(sc->V, V_INIT, sizeof(sc->V));
    // 将指针 ptr 置为 0
    sc->ptr = 0;
}

/* see sph_luffa.h */
// 执行 Luffa224 算法的压缩函数
void
sph_luffa224(void *cc, const void *data, size_t len)
{
    // 调用 luffa3 函数执行 Luffa224 算法的压缩函数
    luffa3(cc, data, len);
}

/* see sph_luffa.h */
// 关闭 Luffa224 算法的上下文并输出结果
void
sph_luffa224_close(void *cc, void *dst)
{
    // 调用 sph_luffa224_addbits_and_close 函数关闭上下文并输出结果
    sph_luffa224_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_luffa.h */
// 添加位并关闭 Luffa224 算法的上下文并输出结果
void
sph_luffa224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 luffa3_close 函数关闭上下文并输出结果
    luffa3_close(cc, ub, n, dst, 7);
    // 重新初始化 Luffa224 上下文
    sph_luffa224_init(cc);
}

/* see sph_luffa.h */
// 初始化 Luffa256 算法的上下文
void
sph_luffa256_init(void *cc)
{
    // 将上下文转换为 Luffa256 上下文类型
    sph_luffa256_context *sc;

    sc = cc;
    // 将初始向量 V_INIT 复制到上下文的 V 数组中
    memcpy(sc->V, V_INIT, sizeof(sc->V));
    // 将指针 ptr 置为 0
    sc->ptr = 0;
}

/* see sph_luffa.h */
// 执行 Luffa256 算法的压缩函数
void
sph_luffa256(void *cc, const void *data, size_t len)
{
    // 调用 luffa3 函数执行 Luffa256 算法的压缩函数
    luffa3(cc, data, len);
}

/* see sph_luffa.h */
// 关闭 Luffa256 算法的上下文并输出结果
void
sph_luffa256_close(void *cc, void *dst)
{
    // 调用 sph_luffa256_addbits_and_close 函数关闭上下文并输出结果
    sph_luffa256_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_luffa.h */
// 添加位并关闭 Luffa256 算法的上下文并输出结果
void
sph_luffa256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 luffa3_close 函数关闭上下文并输出结果
    luffa3_close(cc, ub, n, dst, 8);
    // 重新初始化 Luffa256 上下文
    sph_luffa256_init(cc);
}

/* see sph_luffa.h */
// 初始化 Luffa384 算法的上下文
void
sph_luffa384_init(void *cc)
{
    // 将上下文转换为 Luffa384 上下文类型
    sph_luffa384_context *sc;

    sc = cc;
    // 将初始向量 V_INIT 复制到上下文的 V 数组中
    memcpy(sc->V, V_INIT, sizeof(sc->V));
    // 将指针 ptr 置为 0
    sc->ptr = 0;
}

/* see sph_luffa.h */
// 执行 Luffa384 算法的压缩函数
void
sph_luffa384(void *cc, const void *data, size_t len)
{
    // 调用 luffa4 函数执行 Luffa384 算法的压缩函数
    luffa4(cc, data, len);
}

/* see sph_luffa.h */
// 关闭 Luffa384 算法的上下文并输出结果
void
sph_luffa384_close(void *cc, void *dst)
{
    // 调用 sph_luffa384_addbits_and_close 函数关闭上下文并输出结果
    sph_luffa384_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_luffa.h */
// 添加位并关闭 Luffa384 算法的上下文并输出结果
void
sph_luffa384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 luffa4_close 函数关闭上下文并输出结果
    luffa4_close(cc, ub, n, dst);
    // 重新初始化 Luffa384 上下文
    sph_luffa384_init(cc);
}

/* see sph_luffa.h */
// 初始化 Luffa512 算法的上下文
void
sph_luffa512_init(void *cc)
{
    // 将上下文转换为 Luffa512 上下文类型
    sph_luffa512_context *sc;

    sc = cc;
    // 将初始向量 V_INIT 复制到上下文的 V 数组中
    memcpy(sc->V, V_INIT, sizeof(sc->V));
    // 将指针 ptr 置为 0
    sc->ptr = 0;
}

/* see sph_luffa.h */
// 执行 Luffa512 算法的压缩函数
void
sph_luffa512(void *cc, const void *data, size_t len)
{
    // 调用 luffa5 函数执行 Luffa512 算法的压缩函数
    luffa5(cc, data, len);
}
/* see sph_luffa.h */
// 关闭 luffa512 算法的上下文，将结果输出到目标内存
void
sph_luffa512_close(void *cc, void *dst)
{
    // 调用 sph_luffa512_addbits_and_close 函数，传入参数 0, 0，将结果输出到目标内存
    sph_luffa512_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_luffa.h */
// 将额外的比特数和消息长度添加到 luffa512 算法的上下文中，并关闭算法，将结果输出到目标内存
void
sph_luffa512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 luffa5_close 函数，传入参数 ub, n，将结果输出到目标内存
    luffa5_close(cc, ub, n, dst);
    // 重新初始化 luffa512 算法的上下文
    sph_luffa512_init(cc);
}

#ifdef __cplusplus
}
#endif
```