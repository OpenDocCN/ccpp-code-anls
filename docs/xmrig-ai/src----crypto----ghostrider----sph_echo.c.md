# `xmrig\src\crypto\ghostrider\sph_echo.c`

```cpp
/* $Id: echo.c 227 2010-06-16 17:28:38Z tp $ */
/* 
 * ECHO implementation.
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

#include "sph_echo.h"

#ifdef __cplusplus
extern "C"{
#endif

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_ECHO
#define SPH_SMALL_FOOTPRINT_ECHO   1
#endif

/*
 * Some measures tend to show that the 64-bit implementation offers
 * better performance only on a "64-bit architectures", those which have
 * actual 64-bit registers.
 */
#if !defined SPH_ECHO_64 && SPH_64_TRUE
#define SPH_ECHO_64   1
#endif

/*
 * We can use a 64-bit implementation only if a 64-bit type is available.
 */
#if !SPH_64
#ifndef SPH_ECHO_64
#endif

#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif

#define T32   SPH_T32
#define C32   SPH_C32
#if SPH_64
#define C64   SPH_C64
#endif

#define AES_BIG_ENDIAN   0
#include "aes_helper.c"

#if SPH_ECHO_64

#define DECL_STATE_SMALL   \  # 定义小状态
    sph_u64 W[16][2];  # 定义一个二维数组W，包含16行2列

#define DECL_STATE_BIG   \  # 定义大状态
    sph_u64 W[16][2];  # 定义一个二维数组W，包含16行2列

#define INPUT_BLOCK_SMALL(sc)   do { \  # 定义输入块函数，参数为sc
        unsigned u;  # 定义一个无符号整数u
        memcpy(W, sc->u.Vb, 8 * sizeof(sph_u64));  # 将sc->u.Vb的内容复制到W中
        for (u = 0; u < 12; u ++) {  # 循环12次
            W[u + 4][0] = sph_dec64le_aligned( \  # 对W进行赋值
                sc->buf + 16 * u);  # 计算偏移量
            W[u + 4][1] = sph_dec64le_aligned( \  # 对W进行赋值
                sc->buf + 16 * u + 8);  # 计算偏移量
        }  # 结束循环
    } while (0)  # 结束do-while循环

#define INPUT_BLOCK_BIG(sc)   do { \  # 定义输入块函数，参数为sc
        unsigned u;  # 定义一个无符号整数u
        memcpy(W, sc->u.Vb, 16 * sizeof(sph_u64));  # 将sc->u.Vb的内容复制到W中
        for (u = 0; u < 8; u ++) {  # 循环8次
            W[u + 8][0] = sph_dec64le_aligned( \  # 对W进行赋值
                sc->buf + 16 * u);  # 计算偏移量
            W[u + 8][1] = sph_dec64le_aligned( \  # 对W进行赋值
                sc->buf + 16 * u + 8);  # 计算偏移量
        }  # 结束循环
    } while (0)  # 结束do-while循环

#if SPH_SMALL_FOOTPRINT_ECHO

static void
aes_2rounds_all(sph_u64 W[16][2],  # 定义函数aes_2rounds_all，参数为W、pK0、pK1、pK2、pK3
    sph_u32 *pK0, sph_u32 *pK1, sph_u32 *pK2, sph_u32 *pK3)  # 定义参数类型为指向sph_u32类型的指针
{
    int n;  # 定义一个整数n
    sph_u32 K0 = *pK0;  # 定义一个sph_u32类型的变量K0，赋值为指针pK0所指向的值
    sph_u32 K1 = *pK1;  # 定义一个sph_u32类型的变量K1，赋值为指针pK1所指向的值
    sph_u32 K2 = *pK2;  # 定义一个sph_u32类型的变量K2，赋值为指针pK2所指向的值
    sph_u32 K3 = *pK3;  # 定义一个sph_u32类型的变量K3，赋值为指针pK3所指向的值

    for (n = 0; n < 16; n ++) {  # 循环16次
        sph_u64 Wl = W[n][0];  # 定义一个sph_u64类型的变量Wl，赋值为W[n][0]
        sph_u64 Wh = W[n][1];  # 定义一个sph_u64类型的变量Wh，赋值为W[n][1]
        sph_u32 X0 = (sph_u32)Wl;  # 定义一个sph_u32类型的变量X0，赋值为Wl的低32位
        sph_u32 X1 = (sph_u32)(Wl >> 32);  # 定义一个sph_u32类型的变量X1，赋值为Wl的高32位
        sph_u32 X2 = (sph_u32)Wh;  # 定义一个sph_u32类型的变量X2，赋值为Wh的低32位
        sph_u32 X3 = (sph_u32)(Wh >> 32);  # 定义一个sph_u32类型的变量X3，赋值为Wh的高32位
        sph_u32 Y0, Y1, Y2, Y3; \  # 定义四个sph_u32类型的变量Y0、Y1、Y2、Y3
        AES_ROUND_LE(X0, X1, X2, X3, K0, K1, K2, K3, Y0, Y1, Y2, Y3);  # 调用AES_ROUND_LE函数
        AES_ROUND_NOKEY_LE(Y0, Y1, Y2, Y3, X0, X1, X2, X3);  # 调用AES_ROUND_NOKEY_LE函数
        W[n][0] = (sph_u64)X0 | ((sph_u64)X1 << 32);  # 对W进行赋值
        W[n][1] = (sph_u64)X2 | ((sph_u64)X3 << 32);  # 对W进行赋值
        if ((K0 = T32(K0 + 1)) == 0) {  # 判断条件
            if ((K1 = T32(K1 + 1)) == 0)  # 判断条件
                if ((K2 = T32(K2 + 1)) == 0)  # 判断条件
                    K3 = T32(K3 + 1);  # 对K3进行赋值
        }  # 结束if条件
    }  # 结束循环
    *pK0 = K0;  # 对pK0所指向的值进行赋值
    # 将指针pK1指向的地址赋值为K1的值
    *pK1 = K1;
    # 将指针pK2指向的地址赋值为K2的值
    *pK2 = K2;
    # 将指针pK3指向的地址赋值为K3的值
    *pK3 = K3;
#define BIG_SUB_WORDS   do { \  # 定义一个宏，用于执行多轮AES加密
        AES_2ROUNDS(W[ 0]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 1]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 2]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 3]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 4]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 5]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 6]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 7]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 8]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[ 9]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[10]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[11]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[12]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[13]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[14]); \  # 执行AES加密的两轮操作
        AES_2ROUNDS(W[15]); \  # 执行AES加密的两轮操作
    } while (0)  # 结束宏定义

#endif  # 结束条件编译

#define SHIFT_ROW1(a, b, c, d)   do { \  # 定义一个宏，用于执行行移位操作
        sph_u64 tmp; \  # 声明一个64位整数变量tmp
        tmp = W[a][0]; \  # 将W数组中指定位置的值赋给tmp
        W[a][0] = W[b][0]; \  # 将W数组中指定位置的值进行交换
        W[b][0] = W[c][0]; \  # 将W数组中指定位置的值进行交换
        W[c][0] = W[d][0]; \  # 将W数组中指定位置的值进行交换
        W[d][0] = tmp; \  # 将tmp的值赋给W数组中指定位置
        tmp = W[a][1]; \  # 将W数组中指定位置的值赋给tmp
        W[a][1] = W[b][1]; \  # 将W数组中指定位置的值进行交换
        W[b][1] = W[c][1]; \  # 将W数组中指定位置的值进行交换
        W[c][1] = W[d][1]; \  # 将W数组中指定位置的值进行交换
        W[d][1] = tmp; \  # 将tmp的值赋给W数组中指定位置
    } while (0)  # 结束宏定义
// 定义一个宏，用于对指定的行进行数据交换
#define SHIFT_ROW2(a, b, c, d)   do { \
        // 声明一个临时变量
        sph_u64 tmp; \
        // 交换指定位置的数据
        tmp = W[a][0]; \
        W[a][0] = W[c][0]; \
        W[c][0] = tmp; \
        tmp = W[b][0]; \
        W[b][0] = W[d][0]; \
        W[d][0] = tmp; \
        tmp = W[a][1]; \
        W[a][1] = W[c][1]; \
        W[c][1] = tmp; \
        tmp = W[b][1]; \
        W[b][1] = W[d][1]; \
        W[d][1] = tmp; \
    } while (0)

// 定义一个宏，用于对指定的行进行数据交换
#define SHIFT_ROW3(a, b, c, d)   SHIFT_ROW1(d, c, b, a)

// 定义一个宏，用于对多行进行数据交换
#define BIG_SHIFT_ROWS   do { \
        // 调用SHIFT_ROW1、SHIFT_ROW2和SHIFT_ROW3宏，对指定的行进行数据交换
        SHIFT_ROW1(1, 5, 9, 13); \
        SHIFT_ROW2(2, 6, 10, 14); \
        SHIFT_ROW3(3, 7, 11, 15); \
    } while (0)

// 如果定义了SPH_SMALL_FOOTPRINT_ECHO，则定义一个静态函数mix_column，用于对指定的列进行混合
#if SPH_SMALL_FOOTPRINT_ECHO

static void
mix_column(sph_u64 W[16][2], int ia, int ib, int ic, int id)
{
    int n;

    // 遍历每一列
    for (n = 0; n < 2; n ++) {
        // 获取指定位置的数据
        sph_u64 a = W[ia][n];
        sph_u64 b = W[ib][n];
        sph_u64 c = W[ic][n];
        sph_u64 d = W[id][n];
        // 对数据进行混合运算
        sph_u64 ab = a ^ b;
        sph_u64 bc = b ^ c;
        sph_u64 cd = c ^ d;
        sph_u64 abx = ((ab & C64(0x8080808080808080)) >> 7) * 27U
            ^ ((ab & C64(0x7F7F7F7F7F7F7F7F)) << 1);
        sph_u64 bcx = ((bc & C64(0x8080808080808080)) >> 7) * 27U
            ^ ((bc & C64(0x7F7F7F7F7F7F7F7F)) << 1);
        sph_u64 cdx = ((cd & C64(0x8080808080808080)) >> 7) * 27U
            ^ ((cd & C64(0x7F7F7F7F7F7F7F7F)) << 1);
        // 更新指定位置的数据
        W[ia][n] = abx ^ bc ^ d;
        W[ib][n] = bcx ^ a ^ cd;
        W[ic][n] = cdx ^ ab ^ d;
        W[id][n] = abx ^ bcx ^ cdx ^ ab ^ c;
    }
}

// 定义一个宏，用于调用mix_column函数，对指定的列进行混合
#define MIX_COLUMN(a, b, c, d)   mix_column(W, a, b, c, d)

#else
# 定义一个宏，用于对列进行混合操作
#define MIX_COLUMN1(ia, ib, ic, id, n)   do { \
        # 定义并初始化变量a，b，c，d，分别为W数组中ia，ib，ic，id位置的元素
        sph_u64 a = W[ia][n]; \
        sph_u64 b = W[ib][n]; \
        sph_u64 c = W[ic][n]; \
        sph_u64 d = W[id][n]; \
        # 计算ab，bc，cd的异或结果
        sph_u64 ab = a ^ b; \
        sph_u64 bc = b ^ c; \
        sph_u64 cd = c ^ d; \
        # 计算abx，bcx，cdx的值
        sph_u64 abx = ((ab & C64(0x8080808080808080)) >> 7) * 27U \
            ^ ((ab & C64(0x7F7F7F7F7F7F7F7F)) << 1); \
        sph_u64 bcx = ((bc & C64(0x8080808080808080)) >> 7) * 27U \
            ^ ((bc & C64(0x7F7F7F7F7F7F7F7F)) << 1); \
        sph_u64 cdx = ((cd & C64(0x8080808080808080)) >> 7) * 27U \
            ^ ((cd & C64(0x7F7F7F7F7F7F7F7F)) << 1); \
        # 更新W数组中ia，ib，ic，id位置的元素
        W[ia][n] = abx ^ bc ^ d; \
        W[ib][n] = bcx ^ a ^ cd; \
        W[ic][n] = cdx ^ ab ^ d; \
        W[id][n] = abx ^ bcx ^ cdx ^ ab ^ c; \
    } while (0)

# 定义一个宏，用于对多列进行混合操作
#define MIX_COLUMN(a, b, c, d)   do { \
        # 调用MIX_COLUMN1宏，对指定的四列进行混合操作
        MIX_COLUMN1(a, b, c, d, 0); \
        MIX_COLUMN1(a, b, c, d, 1); \
    } while (0)

# 定义一个宏，用于对大列进行混合操作
#define BIG_MIX_COLUMNS   do { \
        # 调用MIX_COLUMN宏，对指定的四组列进行混合操作
        MIX_COLUMN(0, 1, 2, 3); \
        MIX_COLUMN(4, 5, 6, 7); \
        MIX_COLUMN(8, 9, 10, 11); \
        MIX_COLUMN(12, 13, 14, 15); \
    } while (0)

# 定义一个宏，用于对大轮进行操作
#define BIG_ROUND   do { \
        # 调用其他宏，对大轮进行操作
        BIG_SUB_WORDS; \
        BIG_SHIFT_ROWS; \
        BIG_MIX_COLUMNS; \
    } while (0)

# 定义一个宏，用于对最后的小轮进行操作
#define FINAL_SMALL   do { \
        # 定义并初始化变量u，VV，WW
        unsigned u; \
        sph_u64 *VV = &sc->u.Vb[0][0]; \
        sph_u64 *WW = &W[0][0]; \
        # 循环对VV数组进行更新操作
        for (u = 0; u < 8; u ++) { \
            VV[u] ^= sph_dec64le_aligned(sc->buf + (u * 8)) \
                ^ sph_dec64le_aligned(sc->buf + (u * 8) + 64) \
                ^ sph_dec64le_aligned(sc->buf + (u * 8) + 128) \
                ^ WW[u] ^ WW[u + 8] \
                ^ WW[u + 16] ^ WW[u + 24]; \
        } \
    } while (0)
#define FINAL_BIG   do { \  # 定义一个宏，用于进行最终的大块压缩
        unsigned u; \  # 声明一个无符号整数变量 u
        sph_u64 *VV = &sc->u.Vb[0][0]; \  # 声明一个指向 sc->u.Vb[0][0] 的 sph_u64 指针 VV
        sph_u64 *WW = &W[0][0]; \  # 声明一个指向 W[0][0] 的 sph_u64 指针 WW
        for (u = 0; u < 16; u ++) { \  # 循环遍历 16 次
            VV[u] ^= sph_dec64le_aligned(sc->buf + (u * 8)) \  # 对 VV[u] 进行异或操作
                ^ WW[u] ^ WW[u + 16]; \  # 再与 WW[u] 和 WW[u + 16] 进行异或操作
        } \  # 结束循环
    } while (0)  # 结束宏定义

#define COMPRESS_SMALL(sc)   do { \  # 定义一个宏，用于进行小块压缩
        sph_u32 K0 = sc->C0; \  # 声明一个 sph_u32 类型的变量 K0，并赋值为 sc->C0
        sph_u32 K1 = sc->C1; \  # 声明一个 sph_u32 类型的变量 K1，并赋值为 sc->C1
        sph_u32 K2 = sc->C2; \  # 声明一个 sph_u32 类型的变量 K2，并赋值为 sc->C2
        sph_u32 K3 = sc->C3; \  # 声明一个 sph_u32 类型的变量 K3，并赋值为 sc->C3
        unsigned u; \  # 声明一个无符号整数变量 u
        INPUT_BLOCK_SMALL(sc); \  # 调用 INPUT_BLOCK_SMALL 宏
        for (u = 0; u < 8; u ++) { \  # 循环遍历 8 次
            BIG_ROUND; \  # 调用 BIG_ROUND 宏
        } \  # 结束循环
        FINAL_SMALL; \  # 调用 FINAL_SMALL 宏
    } while (0)  # 结束宏定义

#define COMPRESS_BIG(sc)   do { \  # 定义一个宏，用于进行大块压缩
        sph_u32 K0 = sc->C0; \  # 声明一个 sph_u32 类型的变量 K0，并赋值为 sc->C0
        sph_u32 K1 = sc->C1; \  # 声明一个 sph_u32 类型的变量 K1，并赋值为 sc->C1
        sph_u32 K2 = sc->C2; \  # 声明一个 sph_u32 类型的变量 K2，并赋值为 sc->C2
        sph_u32 K3 = sc->C3; \  # 声明一个 sph_u32 类型的变量 K3，并赋值为 sc->C3
        unsigned u; \  # 声明一个无符号整数变量 u
        INPUT_BLOCK_BIG(sc); \  # 调用 INPUT_BLOCK_BIG 宏
        for (u = 0; u < 10; u ++) { \  # 循环遍历 10 次
            BIG_ROUND; \  # 调用 BIG_ROUND 宏
        } \  # 结束循环
        FINAL_BIG; \  # 调用 FINAL_BIG 宏
    } while (0)  # 结束宏定义

#else

#define DECL_STATE_SMALL   \  # 定义一个宏，用于声明小块状态
    sph_u32 W[16][4];  # 声明一个 16x4 的 sph_u32 类型的数组 W

#define DECL_STATE_BIG   \  # 定义一个宏，用于声明大块状态
    sph_u32 W[16][4];  # 声明一个 16x4 的 sph_u32 类型的数组 W

#define INPUT_BLOCK_SMALL(sc)   do { \  # 定义一个宏，用于处理小块输入
        unsigned u; \  # 声明一个无符号整数变量 u
        memcpy(W, sc->u.Vs, 16 * sizeof(sph_u32)); \  # 将 sc->u.Vs 的内容复制到 W 中
        for (u = 0; u < 12; u ++) { \  # 循环遍历 12 次
            W[u + 4][0] = sph_dec32le_aligned( \  # 对 W[u + 4][0] 进行赋值
                sc->buf + 16 * u); \  # 计算偏移量并赋值
            W[u + 4][1] = sph_dec32le_aligned( \  # 对 W[u + 4][1] 进行赋值
                sc->buf + 16 * u + 4); \  # 计算偏移量并赋值
            W[u + 4][2] = sph_dec32le_aligned( \  # 对 W[u + 4][2] 进行赋值
                sc->buf + 16 * u + 8); \  # 计算偏移量并赋值
            W[u + 4][3] = sph_dec32le_aligned( \  # 对 W[u + 4][3] 进行赋值
                sc->buf + 16 * u + 12); \  # 计算偏移量并赋值
        } \  # 结束循环
    } while (0)  # 结束宏定义
#define INPUT_BLOCK_BIG(sc)   do { \  # 定义宏，用于处理大输入块
        unsigned u; \  # 声明无符号整数变量u
        memcpy(W, sc->u.Vs, 32 * sizeof(sph_u32)); \  # 将sc->u.Vs的内容复制到W中
        for (u = 0; u < 8; u ++) { \  # 循环8次
            W[u + 8][0] = sph_dec32le_aligned( \  # 将sc->buf中的数据解码为小端格式的32位整数，并存储到W中
                sc->buf + 16 * u); \  # 计算偏移量并获取数据
            W[u + 8][1] = sph_dec32le_aligned( \  # 将sc->buf中的数据解码为小端格式的32位整数，并存储到W中
                sc->buf + 16 * u + 4); \  # 计算偏移量并获取数据
            W[u + 8][2] = sph_dec32le_aligned( \  # 将sc->buf中的数据解码为小端格式的32位整数，并存储到W中
                sc->buf + 16 * u + 8); \  # 计算偏移量并获取数据
            W[u + 8][3] = sph_dec32le_aligned( \  # 将sc->buf中的数据解码为小端格式的32位整数，并存储到W中
                sc->buf + 16 * u + 12); \  # 计算偏移量并获取数据
        } \  # 结束for循环
    } while (0)  # 结束do-while循环，0表示循环只执行一次

#if SPH_SMALL_FOOTPRINT_ECHO  # 如果定义了SPH_SMALL_FOOTPRINT_ECHO

static void  # 声明静态函数
aes_2rounds_all(sph_u32 W[16][4],  # 函数参数为16x4的二维数组W
    sph_u32 *pK0, sph_u32 *pK1, sph_u32 *pK2, sph_u32 *pK3)  # 函数参数为指向sph_u32类型的指针的指针
{
    int n;  # 声明整数变量n
    sph_u32 K0 = *pK0;  # 初始化sph_u32类型变量K0
    sph_u32 K1 = *pK1;  # 初始化sph_u32类型变量K1
    sph_u32 K2 = *pK2;  # 初始化sph_u32类型变量K2
    sph_u32 K3 = *pK3;  # 初始化sph_u32类型变量K3

    for (n = 0; n < 16; n ++) {  # 循环16次
        sph_u32 *X = W[n];  # 获取W中第n行的指针
        sph_u32 Y0, Y1, Y2, Y3;  # 声明四个sph_u32类型变量
        AES_ROUND_LE(X[0], X[1], X[2], X[3],  # 调用AES_ROUND_LE函数
            K0, K1, K2, K3, Y0, Y1, Y2, Y3);  # 传入参数并获取返回值
        AES_ROUND_NOKEY_LE(Y0, Y1, Y2, Y3, X[0], X[1], X[2], X[3]);  # 调用AES_ROUND_NOKEY_LE函数
        if ((K0 = T32(K0 + 1)) == 0) {  # 如果K0加1后等于0
            if ((K1 = T32(K1 + 1)) == 0)  # 如果K1加1后等于0
                if ((K2 = T32(K2 + 1)) == 0)  # 如果K2加1后等于0
                    K3 = T32(K3 + 1);  # K3加1
        }
    }  # 结束for循环
    *pK0 = K0;  # 更新pK0的值
    *pK1 = K1;  # 更新pK1的值
    *pK2 = K2;  # 更新pK2的值
    *pK3 = K3;  # 更新pK3的值
}

#define BIG_SUB_WORDS   do { \  # 定义宏，用于处理大输入块
        aes_2rounds_all(W, &K0, &K1, &K2, &K3); \  # 调用aes_2rounds_all函数
    } while (0)  # 结束do-while循环，0表示循环只执行一次

#else  # 如果没有定义SPH_SMALL_FOOTPRINT_ECHO

#define AES_2ROUNDS(X)   do { \  # 定义宏，用于处理AES的2轮加密
        sph_u32 Y0, Y1, Y2, Y3; \  # 声明四个sph_u32类型变量
        AES_ROUND_LE(X[0], X[1], X[2], X[3], \  # 调用AES_ROUND_LE函数
            K0, K1, K2, K3, Y0, Y1, Y2, Y3); \  # 传入参数并获取返回值
        AES_ROUND_NOKEY_LE(Y0, Y1, Y2, Y3, X[0], X[1], X[2], X[3]); \  # 调用AES_ROUND_NOKEY_LE函数
        if ((K0 = T32(K0 + 1)) == 0) { \  # 如果K0加1后等于0
            if ((K1 = T32(K1 + 1)) == 0) \  # 如果K1加1后等于0
                if ((K2 = T32(K2 + 1)) == 0) \  # 如果K2加1后等于0
                    K3 = T32(K3 + 1); \  # K3加1
        } \  # 结束if语句
    } while (0)  # 结束do-while循环，0表示循环只执行一次
#define BIG_SUB_WORDS   do { \  # 定义一个宏，用于执行一系列的AES_2ROUNDS操作
        AES_2ROUNDS(W[ 0]); \  # 对W[0]执行两轮AES操作
        AES_2ROUNDS(W[ 1]); \  # 对W[1]执行两轮AES操作
        AES_2ROUNDS(W[ 2]); \  # 对W[2]执行两轮AES操作
        AES_2ROUNDS(W[ 3]); \  # 对W[3]执行两轮AES操作
        AES_2ROUNDS(W[ 4]); \  # 对W[4]执行两轮AES操作
        AES_2ROUNDS(W[ 5]); \  # 对W[5]执行两轮AES操作
        AES_2ROUNDS(W[ 6]); \  # 对W[6]执行两轮AES操作
        AES_2ROUNDS(W[ 7]); \  # 对W[7]执行两轮AES操作
        AES_2ROUNDS(W[ 8]); \  # 对W[8]执行两轮AES操作
        AES_2ROUNDS(W[ 9]); \  # 对W[9]执行两轮AES操作
        AES_2ROUNDS(W[10]); \  # 对W[10]执行两轮AES操作
        AES_2ROUNDS(W[11]); \  # 对W[11]执行两轮AES操作
        AES_2ROUNDS(W[12]); \  # 对W[12]执行两轮AES操作
        AES_2ROUNDS(W[13]); \  # 对W[13]执行两轮AES操作
        AES_2ROUNDS(W[14]); \  # 对W[14]执行两轮AES操作
        AES_2ROUNDS(W[15]); \  # 对W[15]执行两轮AES操作
    } while (0)  # 宏定义结束

#endif  # 结束宏定义部分

#define SHIFT_ROW1(a, b, c, d)   do { \  # 定义一个宏，用于执行行移位操作
        sph_u32 tmp; \  # 声明一个临时变量tmp
        tmp = W[a][0]; \  # 将W[a][0]的值赋给tmp
        W[a][0] = W[b][0]; \  # 将W[b][0]的值赋给W[a][0]
        W[b][0] = W[c][0]; \  # 将W[c][0]的值赋给W[b][0]
        W[c][0] = W[d][0]; \  # 将W[d][0]的值赋给W[c][0]
        W[d][0] = tmp; \  # 将tmp的值赋给W[d][0]
        tmp = W[a][1]; \  # 重复上述操作，但是针对W[a][1]和W[b][1]等
        W[a][1] = W[b][1]; \
        W[b][1] = W[c][1]; \
        W[c][1] = W[d][1]; \
        W[d][1] = tmp; \
        tmp = W[a][2]; \
        W[a][2] = W[b][2]; \
        W[b][2] = W[c][2]; \
        W[c][2] = W[d][2]; \
        W[d][2] = tmp; \
        tmp = W[a][3]; \
        W[a][3] = W[b][3]; \
        W[b][3] = W[c][3]; \
        W[c][3] = W[d][3]; \
        W[d][3] = tmp; \
    } while (0)  # 宏定义结束

#define SHIFT_ROW2(a, b, c, d)   do { \  # 定义一个宏，用于执行行移位操作
        sph_u32 tmp; \  # 声明一个临时变量tmp
        tmp = W[a][0]; \  # 将W[a][0]的值赋给tmp
        W[a][0] = W[c][0]; \  # 将W[c][0]的值赋给W[a][0]
        W[c][0] = tmp; \  # 将tmp的值赋给W[c][0]
        tmp = W[b][0]; \  # 重复上述操作，但是针对W[b][0]和W[d][0]等
        W[b][0] = W[d][0]; \
        W[d][0] = tmp; \
        tmp = W[a][1]; \
        W[a][1] = W[c][1]; \
        W[c][1] = tmp; \
        tmp = W[b][1]; \
        W[b][1] = W[d][1]; \
        W[d][1] = tmp; \
        tmp = W[a][2]; \
        W[a][2] = W[c][2]; \
        W[c][2] = tmp; \
        tmp = W[b][2]; \
        W[b][2] = W[d][2]; \
        W[d][2] = tmp; \
        tmp = W[a][3]; \
        W[a][3] = W[c][3]; \
        W[c][3] = tmp; \
        tmp = W[b][3]; \
        W[b][3] = W[d][3]; \
        W[d][3] = tmp; \
    } while (0)  # 宏定义结束

#define SHIFT_ROW3(a, b, c, d)   SHIFT_ROW1(d, c, b, a)  # 定义一个宏，用于执行行移位操作，调用了SHIFT_ROW1宏
#define BIG_SHIFT_ROWS   do { \  # 定义一个宏，用于执行行移位操作
        SHIFT_ROW1(1, 5, 9, 13); \  # 调用SHIFT_ROW1宏，对指定位置的行进行移位操作
        SHIFT_ROW2(2, 6, 10, 14); \  # 调用SHIFT_ROW2宏，对指定位置的行进行移位操作
        SHIFT_ROW3(3, 7, 11, 15); \  # 调用SHIFT_ROW3宏，对指定位置的行进行移位操作
    } while (0)  # 宏定义结束

#if SPH_SMALL_FOOTPRINT_ECHO  # 如果定义了SPH_SMALL_FOOTPRINT_ECHO

static void  # 静态函数mix_column的定义
mix_column(sph_u32 W[16][4], int ia, int ib, int ic, int id)  # mix_column函数的参数列表
{  # 函数体开始
    int n;  # 声明整型变量n

    for (n = 0; n < 4; n ++) {  # 循环，n从0到3
        sph_u32 a = W[ia][n];  # 获取W数组中指定位置的值，赋给a
        sph_u32 b = W[ib][n];  # 获取W数组中指定位置的值，赋给b
        sph_u32 c = W[ic][n];  # 获取W数组中指定位置的值，赋给c
        sph_u32 d = W[id][n];  # 获取W数组中指定位置的值，赋给d
        sph_u32 ab = a ^ b;  # 对a和b进行按位异或操作，结果赋给ab
        sph_u32 bc = b ^ c;  # 对b和c进行按位异或操作，结果赋给bc
        sph_u32 cd = c ^ d;  # 对c和d进行按位异或操作，结果赋给cd
        sph_u32 abx = ((ab & C32(0x80808080)) >> 7) * 27U  # 对ab进行位与、位移和乘法操作，结果赋给abx
            ^ ((ab & C32(0x7F7F7F7F)) << 1);  # 对ab进行位与、位移和乘法操作，结果与abx进行按位异或
        sph_u32 bcx = ((bc & C32(0x80808080)) >> 7) * 27U  # 对bc进行位与、位移和乘法操作，结果赋给bcx
            ^ ((bc & C32(0x7F7F7F7F)) << 1);  # 对bc进行位与、位移和乘法操作，结果与bcx进行按位异或
        sph_u32 cdx = ((cd & C32(0x80808080)) >> 7) * 27U  # 对cd进行位与、位移和乘法操作，结果赋给cdx
            ^ ((cd & C32(0x7F7F7F7F)) << 1);  # 对cd进行位与、位移和乘法操作，结果与cdx进行按位异或
        W[ia][n] = abx ^ bc ^ d;  # 对abx、bc和d进行按位异或操作，结果赋给W数组指定位置
        W[ib][n] = bcx ^ a ^ cd;  # 对bcx、a和cd进行按位异或操作，结果赋给W数组指定位置
        W[ic][n] = cdx ^ ab ^ d;  # 对cdx、ab和d进行按位异或操作，结果赋给W数组指定位置
        W[id][n] = abx ^ bcx ^ cdx ^ ab ^ c;  # 对abx、bcx、cdx、ab和c进行按位异或操作，结果赋给W数组指定位置
    }  # 循环结束
}  # 函数体结束

#define MIX_COLUMN(a, b, c, d)   mix_column(W, a, b, c, d)  # 定义一个宏，用于调用mix_column函数

#else  # 如果没有定义SPH_SMALL_FOOTPRINT_ECHO

#define MIX_COLUMN1(ia, ib, ic, id, n)   do { \  # 定义一个宏，用于执行mix_column1操作
        sph_u32 a = W[ia][n]; \  # 获取W数组中指定位置的值，赋给a
        sph_u32 b = W[ib][n]; \  # 获取W数组中指定位置的值，赋给b
        sph_u32 c = W[ic][n]; \  # 获取W数组中指定位置的值，赋给c
        sph_u32 d = W[id][n]; \  # 获取W数组中指定位置的值，赋给d
        sph_u32 ab = a ^ b; \  # 对a和b进行按位异或操作，结果赋给ab
        sph_u32 bc = b ^ c; \  # 对b和c进行按位异或操作，结果赋给bc
        sph_u32 cd = c ^ d; \  # 对c和d进行按位异或操作，结果赋给cd
        sph_u32 abx = ((ab & C32(0x80808080)) >> 7) * 27U \  # 对ab进行位与、位移和乘法操作，结果赋给abx
            ^ ((ab & C32(0x7F7F7F7F)) << 1); \  # 对ab进行位与、位移和乘法操作，结果与abx进行按位异或
        sph_u32 bcx = ((bc & C32(0x80808080)) >> 7) * 27U \  # 对bc进行位与、位移和乘法操作，结果赋给bcx
            ^ ((bc & C32(0x7F7F7F7F)) << 1); \  # 对bc进行位与、位移和乘法操作，结果与bcx进行按位异或
        sph_u32 cdx = ((cd & C32(0x80808080)) >> 7) * 27U \  # 对cd进行位与、位移和乘法操作，结果赋给cdx
            ^ ((cd & C32(0x7F7F7F7F)) << 1); \  # 对cd进行位与、位移和乘法操作，结果与cdx进行按位异或
        W[ia][n] = abx ^ bc ^ d; \  # 对abx、bc和d进行按位异或操作，结果赋给W数组指定位置
        W[ib][n] = bcx ^ a ^ cd; \  # 对bcx、a和cd进行按位异或操作，结果赋给W数组指定位置
        W[ic][n] = cdx ^ ab ^ d; \  # 对cdx、ab和d进行按位异或操作，结果赋给W数组指定位置
        W[id][n] = abx ^ bcx ^ cdx ^ ab ^ c; \  # 对abx、bcx、cdx、ab和c进行按位异或操作，结果赋给W数组指定位置
    } while (0)  # 宏定义结束

#define MIX_COLUMN(a, b, c, d)   do { \  # 定义一个宏，用于执行mix_column操作
        MIX_COLUMN1(a, b, c, d, 0); \  # 调用MIX_COLUMN1宏，对指定位置的列进行混合操作
        MIX_COLUMN1(a, b, c, d, 1); \  # 调用MIX_COLUMN1宏，对指定位置的列进行混合操作
        MIX_COLUMN1(a, b, c, d, 2); \  # 调用MIX_COLUMN1宏，对指定位置的列进行混合操作
        MIX_COLUMN1(a, b, c, d, 3); \  # 调用MIX_COLUMN1宏，对指定位置的列进行混合操作
    # 这是一个 do-while 循环的结束标记，表示循环体结束
#endif

#define BIG_MIX_COLUMNS   do { \  // 定义宏，用于执行大型列混淆操作
        MIX_COLUMN(0, 1, 2, 3); \  // 调用 MIX_COLUMN 宏，对指定列进行混淆
        MIX_COLUMN(4, 5, 6, 7); \  // 调用 MIX_COLUMN 宏，对指定列进行混淆
        MIX_COLUMN(8, 9, 10, 11); \  // 调用 MIX_COLUMN 宏，对指定列进行混淆
        MIX_COLUMN(12, 13, 14, 15); \  // 调用 MIX_COLUMN 宏，对指定列进行混淆
    } while (0)

#define BIG_ROUND   do { \  // 定义宏，用于执行大轮操作
        BIG_SUB_WORDS; \  // 调用 BIG_SUB_WORDS 宏，执行大型字替换操作
        BIG_SHIFT_ROWS; \  // 调用 BIG_SHIFT_ROWS 宏，执行大型行移位操作
        BIG_MIX_COLUMNS; \  // 调用 BIG_MIX_COLUMNS 宏，执行大型列混淆操作
    } while (0)

#define FINAL_SMALL   do { \  // 定义宏，用于执行最终小型操作
        unsigned u; \  // 声明无符号整数变量 u
        sph_u32 *VV = &sc->u.Vs[0][0]; \  // 定义指向 u.Vs[0][0] 的指针 VV
        sph_u32 *WW = &W[0][0]; \  // 定义指向 W[0][0] 的指针 WW
        for (u = 0; u < 16; u ++) { \  // 循环遍历 16 次
            VV[u] ^= sph_dec32le_aligned(sc->buf + (u * 4)) \  // 对 VV[u] 进行异或操作
                ^ sph_dec32le_aligned(sc->buf + (u * 4) + 64) \  // 对 VV[u] 进行异或操作
                ^ sph_dec32le_aligned(sc->buf + (u * 4) + 128) \  // 对 VV[u] 进行异或操作
                ^ WW[u] ^ WW[u + 16] \  // 对 VV[u] 进行异或操作
                ^ WW[u + 32] ^ WW[u + 48]; \  // 对 VV[u] 进行异或操作
        } \  // 结束循环
    } while (0)

#define FINAL_BIG   do { \  // 定义宏，用于执行最终大型操作
        unsigned u; \  // 声明无符号整数变量 u
        sph_u32 *VV = &sc->u.Vs[0][0]; \  // 定义指向 u.Vs[0][0] 的指针 VV
        sph_u32 *WW = &W[0][0]; \  // 定义指向 W[0][0] 的指针 WW
        for (u = 0; u < 32; u ++) { \  // 循环遍历 32 次
            VV[u] ^= sph_dec32le_aligned(sc->buf + (u * 4)) \  // 对 VV[u] 进行异或操作
                ^ WW[u] ^ WW[u + 32]; \  // 对 VV[u] 进行异或操作
        } \  // 结束循环
    } while (0)

#define COMPRESS_SMALL(sc)   do { \  // 定义宏，用于执行小型压缩操作
        sph_u32 K0 = sc->C0; \  // 声明并初始化无符号整数变量 K0
        sph_u32 K1 = sc->C1; \  // 声明并初始化无符号整数变量 K1
        sph_u32 K2 = sc->C2; \  // 声明并初始化无符号整数变量 K2
        sph_u32 K3 = sc->C3; \  // 声明并初始化无符号整数变量 K3
        unsigned u; \  // 声明无符号整数变量 u
        INPUT_BLOCK_SMALL(sc); \  // 调用 INPUT_BLOCK_SMALL 宏，执行小型输入块操作
        for (u = 0; u < 8; u ++) { \  // 循环遍历 8 次
            BIG_ROUND; \  // 调用 BIG_ROUND 宏，执行大轮操作
        } \  // 结束循环
        FINAL_SMALL; \  // 调用 FINAL_SMALL 宏，执行最终小型操作
    } while (0)

#define COMPRESS_BIG(sc)   do { \  // 定义宏，用于执行大型压缩操作
        sph_u32 K0 = sc->C0; \  // 声明并初始化无符号整数变量 K0
        sph_u32 K1 = sc->C1; \  // 声明并初始化无符号整数变量 K1
        sph_u32 K2 = sc->C2; \  // 声明并初始化无符号整数变量 K2
        sph_u32 K3 = sc->C3; \  // 声明并初始化无符号整数变量 K3
        unsigned u; \  // 声明无符号整数变量 u
        INPUT_BLOCK_BIG(sc); \  // 调用 INPUT_BLOCK_BIG 宏，执行大型输入块操作
        for (u = 0; u < 10; u ++) { \  // 循环遍历 10 次
            BIG_ROUND; \  // 调用 BIG_ROUND 宏，执行大轮操作
        } \  // 结束循环
        FINAL_BIG; \  // 调用 FINAL_BIG 宏，执行最终大型操作
    } while (0)

#endif
#define INCR_COUNTER(sc, val)   do { \  # 定义一个宏，用于增加计数器的值
        sc->C0 = T32(sc->C0 + (sph_u32)(val)); \  # 将计数器C0的值增加val，并使用T32函数进行处理
        if (sc->C0 < (sph_u32)(val)) { \  # 如果C0的值小于val
            if ((sc->C1 = T32(sc->C1 + 1)) == 0) \  # 将C1的值增加1，并使用T32函数进行处理，如果结果为0
                if ((sc->C2 = T32(sc->C2 + 1)) == 0) \  # 将C2的值增加1，并使用T32函数进行处理，如果结果为0
                    sc->C3 = T32(sc->C3 + 1); \  # 将C3的值增加1，并使用T32函数进行处理
        } \  # 结束if语句
    } while (0)  # 结束do-while循环

static void  # 定义一个静态的void函数
echo_small_init(sph_echo_small_context *sc, unsigned out_len)  # 函数名为echo_small_init，参数为sc和out_len
{  # 函数体开始
#if SPH_ECHO_64  # 如果SPH_ECHO_64宏被定义
    sc->u.Vb[0][0] = (sph_u64)out_len;  # 将u.Vb[0][0]的值设置为out_len的sph_u64类型
    sc->u.Vb[0][1] = 0;  # 将u.Vb[0][1]的值设置为0
    ...  # 后续类似操作省略
#else  # 如果SPH_ECHO_64宏未被定义
    sc->u.Vs[0][0] = (sph_u32)out_len;  # 将u.Vs[0][0]的值设置为out_len的sph_u32类型
    ...  # 后续类似操作省略
#endif  # 结束条件编译
    sc->ptr = 0;  # 将ptr的值设置为0
    sc->C0 = sc->C1 = sc->C2 = sc->C3 = 0;  # 将C0、C1、C2、C3的值都设置为0
}  # 函数体结束

static void  # 定义一个静态的void函数
echo_big_init(sph_echo_big_context *sc, unsigned out_len)  # 函数名为echo_big_init，参数为sc和out_len
{  # 函数体开始
#if SPH_ECHO_64  # 如果SPH_ECHO_64宏被定义
    sc->u.Vb[0][0] = (sph_u64)out_len;  # 将u.Vb[0][0]的值设置为out_len的sph_u64类型
    sc->u.Vb[0][1] = 0;  # 将u.Vb[0][1]的值设置为0
    ...  # 后续类似操作省略
#else  # 如果SPH_ECHO_64宏未被定义
    sc->u.Vs[0][0] = (sph_u32)out_len;  # 将u.Vs[0][0]的值设置为out_len的sph_u32类型
    ...  # 后续类似操作省略
#endif  # 结束条件编译
}  # 函数体结束
    # 将数组 sc->u.Vs[2][1]、sc->u.Vs[2][2]、sc->u.Vs[2][3] 的值都设置为 0
    sc->u.Vs[2][1] = sc->u.Vs[2][2] = sc->u.Vs[2][3] = 0;
    # 将数组 sc->u.Vs[3][0] 的值设置为 out_len，sc->u.Vs[3][1]、sc->u.Vs[3][2]、sc->u.Vs[3][3] 的值都设置为 0
    sc->u.Vs[3][0] = (sph_u32)out_len;
    sc->u.Vs[3][1] = sc->u.Vs[3][2] = sc->u.Vs[3][3] = 0;
    # 将数组 sc->u.Vs[4][0] 的值设置为 out_len，sc->u.Vs[4][1]、sc->u.Vs[4][2]、sc->u.Vs[4][3] 的值都设置为 0
    sc->u.Vs[4][0] = (sph_u32)out_len;
    sc->u.Vs[4][1] = sc->u.Vs[4][2] = sc->u.Vs[4][3] = 0;
    # 将数组 sc->u.Vs[5][0] 的值设置为 out_len，sc->u.Vs[5][1]、sc->u.Vs[5][2]、sc->u.Vs[5][3] 的值都设置为 0
    sc->u.Vs[5][0] = (sph_u32)out_len;
    sc->u.Vs[5][1] = sc->u.Vs[5][2] = sc->u.Vs[5][3] = 0;
    # 将数组 sc->u.Vs[6][0] 的值设置为 out_len，sc->u.Vs[6][1]、sc->u.Vs[6][2]、sc->u.Vs[6][3] 的值都设置为 0
    sc->u.Vs[6][0] = (sph_u32)out_len;
    sc->u.Vs[6][1] = sc->u.Vs[6][2] = sc->u.Vs[6][3] = 0;
    # 将数组 sc->u.Vs[7][0] 的值设置为 out_len，sc->u.Vs[7][1]、sc->u.Vs[7][2]、sc->u.Vs[7][3] 的值都设置为 0
    sc->u.Vs[7][0] = (sph_u32)out_len;
    sc->u.Vs[7][1] = sc->u.Vs[7][2] = sc->u.Vs[7][3] = 0;
#endif
    # 重置指针位置
    sc->ptr = 0;
    # 重置压缩函数中的状态变量
    sc->C0 = sc->C1 = sc->C2 = sc->C3 = 0;
}

static void
echo_small_compress(sph_echo_small_context *sc)
{
    DECL_STATE_SMALL
    # 调用小型压缩函数
    COMPRESS_SMALL(sc);
}

static void
echo_big_compress(sph_echo_big_context *sc)
{
    DECL_STATE_BIG
    # 调用大型压缩函数
    COMPRESS_BIG(sc);
}

static void
echo_small_core(sph_echo_small_context *sc,
    const unsigned char *data, size_t len)
{
    unsigned char *buf;
    size_t ptr;

    buf = sc->buf;
    ptr = sc->ptr;
    if (len < (sizeof sc->buf) - ptr) {
        # 如果数据长度小于缓冲区剩余空间大小，则直接拷贝数据到缓冲区
        memcpy(buf + ptr, data, len);
        ptr += len;
        sc->ptr = ptr;
        return;
    }

    while (len > 0) {
        size_t clen;

        clen = (sizeof sc->buf) - ptr;
        if (clen > len)
            clen = len;
        # 拷贝数据到缓冲区
        memcpy(buf + ptr, data, clen);
        ptr += clen;
        data += clen;
        len -= clen;
        if (ptr == sizeof sc->buf) {
            # 更新计数器，调用小型压缩函数，重置指针位置
            INCR_COUNTER(sc, 1536);
            echo_small_compress(sc);
            ptr = 0;
        }
    }
    sc->ptr = ptr;
}

static void
echo_big_core(sph_echo_big_context *sc,
    const unsigned char *data, size_t len)
{
    unsigned char *buf;
    size_t ptr;

    buf = sc->buf;
    ptr = sc->ptr;
    if (len < (sizeof sc->buf) - ptr) {
        # 如果数据长度小于缓冲区剩余空间大小，则直接拷贝数据到缓冲区
        memcpy(buf + ptr, data, len);
        ptr += len;
        sc->ptr = ptr;
        return;
    }

    while (len > 0) {
        size_t clen;

        clen = (sizeof sc->buf) - ptr;
        if (clen > len)
            clen = len;
        # 拷贝数据到缓冲区
        memcpy(buf + ptr, data, clen);
        ptr += clen;
        data += clen;
        len -= clen;
        if (ptr == sizeof sc->buf) {
            # 更新计数器，调用大型压缩函数，重置指针位置
            INCR_COUNTER(sc, 1024);
            echo_big_compress(sc);
            ptr = 0;
        }
    }
    sc->ptr = ptr;
}

static void
echo_small_close(sph_echo_small_context *sc, unsigned ub, unsigned n,
    void *dst, unsigned out_size_w32)
{
    unsigned char *buf;
    size_t ptr;
    unsigned z;
    unsigned elen;
    union {
        unsigned char tmp[32];
        sph_u32 dummy;
#if SPH_ECHO_64
        // 如果定义了 SPH_ECHO_64，则声明一个64位整数变量dummy2
        sph_u64 dummy2;
#endif
    } u;
#if SPH_ECHO_64
    // 如果定义了 SPH_ECHO_64，则声明一个指向64位整数的指针VV，否则声明一个指向32位整数的指针VV
    sph_u64 *VV;
#else
    sph_u32 *VV;
#endif
    unsigned k;

    // 获取 sc 对象的 buf 属性
    buf = sc->buf;
    // 获取 sc 对象的 ptr 属性
    ptr = sc->ptr;
    // 计算 elen 的值，即ptr左移3位再加上n
    elen = ((unsigned)ptr << 3) + n;
    // 调用 INCR_COUNTER 函数，更新计数器
    INCR_COUNTER(sc, elen);
    // 将 sc 对象的C0属性编码成小端字节序，存入u.tmp
    sph_enc32le_aligned(u.tmp, sc->C0);
    // 将 sc 对象的C1属性编码成小端字节序，存入u.tmp+4
    sph_enc32le_aligned(u.tmp + 4, sc->C1);
    // 将 sc 对象的C2属性编码成小端字节序，存入u.tmp+8
    sph_enc32le_aligned(u.tmp + 8, sc->C2);
    // 将 sc 对象的C3属性编码成小端字节序，存入u.tmp+12
    sph_enc32le_aligned(u.tmp + 12, sc->C3);
    /*
     * 如果 elen 为零，则该块实际上不包含消息位，只包含第一个填充位。
     */
    if (elen == 0) {
        // 将 sc 对象的C0、C1、C2、C3属性都置为0
        sc->C0 = sc->C1 = sc->C2 = sc->C3 = 0;
    }
    // 计算z的值，即0x80右移n位
    z = 0x80 >> n;
    // 将buf[ptr]的值更新为((ub & -z) | z) & 0xFF
    buf[ptr ++] = ((ub & -z) | z) & 0xFF;
    // 将buf+ptr开始的内存块清零，大小为sizeof sc->buf - ptr
    memset(buf + ptr, 0, (sizeof sc->buf) - ptr);
    // 如果ptr大于sizeof sc->buf - 18，则调用echo_small_compress函数
    if (ptr > ((sizeof sc->buf) - 18)) {
        echo_small_compress(sc);
        // 将sc对象的C0、C1、C2、C3属性都置为0
        sc->C0 = sc->C1 = sc->C2 = sc->C3 = 0;
        // 将buf开始的内存块清零，大小为sizeof sc->buf
        memset(buf, 0, sizeof sc->buf);
    }
    // 将out_size_w32左移5位后的值编码成小端字节序，存入buf+(sizeof sc->buf)-18
    sph_enc16le(buf + (sizeof sc->buf) - 18, out_size_w32 << 5);
    // 将u.tmp的内容复制到buf+(sizeof sc->buf)-16
    memcpy(buf + (sizeof sc->buf) - 16, u.tmp, 16);
    // 调用echo_small_compress函数
    echo_small_compress(sc);
#if SPH_ECHO_64
    // 如果定义了 SPH_ECHO_64，则遍历VV数组，将VV中的值编码成小端字节序存入u.tmp
    for (VV = &sc->u.Vb[0][0], k = 0; k < ((out_size_w32 + 1) >> 1); k ++)
        sph_enc64le_aligned(u.tmp + (k << 3), VV[k]);
#else
    // 如果未定义 SPH_ECHO_64，则遍历VV数组，将VV中的值编码成小端字节序存入u.tmp
    for (VV = &sc->u.Vs[0][0], k = 0; k < out_size_w32; k ++)
        sph_enc32le_aligned(u.tmp + (k << 2), VV[k]);
#endif
    // 将u.tmp的内容复制到dst，大小为out_size_w32左移2位
    memcpy(dst, u.tmp, out_size_w32 << 2);
    // 调用echo_small_init函数，初始化echo_small_context对象
    echo_small_init(sc, out_size_w32 << 5);
}

static void
echo_big_close(sph_echo_big_context *sc, unsigned ub, unsigned n,
    void *dst, unsigned out_size_w32)
{
    unsigned char *buf;
    size_t ptr;
    unsigned z;
    unsigned elen;
    union {
        unsigned char tmp[64];
        sph_u32 dummy;
#if SPH_ECHO_64
        sph_u64 dummy2;
#endif
    } u;
#if SPH_ECHO_64
    sph_u64 *VV;
#else
    sph_u32 *VV;
#endif
    unsigned k;

    // 获取 sc 对象的 buf 属性
    buf = sc->buf;
    // 获取 sc 对象的 ptr 属性
    ptr = sc->ptr;
    // 计算 elen 的值，即ptr左移3位再加上n
    elen = ((unsigned)ptr << 3) + n;
    // 调用 INCR_COUNTER 函数，更新计数器
    INCR_COUNTER(sc, elen);
    // 将 sc 对象的C0属性编码成小端字节序，存入u.tmp
    sph_enc32le_aligned(u.tmp, sc->C0);
    // 将 sc 对象的C1属性编码成小端字节序，存入u.tmp+4
    sph_enc32le_aligned(u.tmp + 4, sc->C1);
    # 将sc->C2的值以小端格式编码后存入u.tmp的第8个字节开始的位置
    sph_enc32le_aligned(u.tmp + 8, sc->C2);
    # 将sc->C3的值以小端格式编码后存入u.tmp的第12个字节开始的位置
    sph_enc32le_aligned(u.tmp + 12, sc->C3);
    # 如果elen为0，则将sc->C0、sc->C1、sc->C2、sc->C3的值都设为0
    if (elen == 0) {
        sc->C0 = sc->C1 = sc->C2 = sc->C3 = 0;
    }
    # 计算z的值，用于设置buf中的特定位
    z = 0x80 >> n;
    # 将计算后的值存入buf中，并根据条件进行位运算
    buf[ptr ++] = ((ub & -z) | z) & 0xFF;
    # 将buf中ptr之后的内容全部置为0
    memset(buf + ptr, 0, (sizeof sc->buf) - ptr);
    # 如果ptr大于((sizeof sc->buf) - 18)，则执行一系列操作
    if (ptr > ((sizeof sc->buf) - 18)) {
        # 调用echo_big_compress函数
        echo_big_compress(sc);
        # 将sc->C0、sc->C1、sc->C2、sc->C3的值都设为0
        sc->C0 = sc->C1 = sc->C2 = sc->C3 = 0;
        # 将buf全部置为0
        memset(buf, 0, sizeof sc->buf);
    }
    # 将out_size_w32左移5位后以小端格式编码后存入buf的末尾18个字节开始的位置
    sph_enc16le(buf + (sizeof sc->buf) - 18, out_size_w32 << 5);
    # 将u.tmp中的内容复制到buf的末尾16个字节开始的位置
    memcpy(buf + (sizeof sc->buf) - 16, u.tmp, 16);
    # 调用echo_big_compress函数
    echo_big_compress(sc);
#if SPH_ECHO_64
    // 如果定义了 SPH_ECHO_64，则执行以下代码块
    for (VV = &sc->u.Vb[0][0], k = 0; k < ((out_size_w32 + 1) >> 1); k ++)
        // 遍历 u.Vb 数组，将数据以小端序写入 u.tmp 数组
        sph_enc64le_aligned(u.tmp + (k << 3), VV[k]);
#else
    // 如果未定义 SPH_ECHO_64，则执行以下代码块
    for (VV = &sc->u.Vs[0][0], k = 0; k < out_size_w32; k ++)
        // 遍历 u.Vs 数组，将数据以小端序写入 u.tmp 数组
        sph_enc32le_aligned(u.tmp + (k << 2), VV[k]);
#endif
    // 将 u.tmp 数组的内容复制到 dst 数组
    memcpy(dst, u.tmp, out_size_w32 << 2);
    // 调用 echo_big_init 函数，初始化 echo 算法的大版本
    echo_big_init(sc, out_size_w32 << 5);
}

/* see sph_echo.h */
// 初始化 echo224 算法
void
sph_echo224_init(void *cc)
{
    echo_small_init(cc, 224);
}

/* see sph_echo.h */
// 执行 echo224 算法的核心计算
void
sph_echo224(void *cc, const void *data, size_t len)
{
    echo_small_core(cc, data, len);
}

/* see sph_echo.h */
// 结束 echo224 算法的计算，将结果写入 dst 数组
void
sph_echo224_close(void *cc, void *dst)
{
    echo_small_close(cc, 0, 0, dst, 7);
}

/* see sph_echo.h */
// 在结束 echo224 算法的计算前，添加额外的比特数据，并将结果写入 dst 数组
void
sph_echo224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    echo_small_close(cc, ub, n, dst, 7);
}

// 以下类似的注释略，分别对应 echo256, echo384, echo512 算法的初始化、核心计算、结束计算和添加额外比特数据并结束计算的函数
# 关闭 echo512 算法的上下文，并将结果写入目标内存
sph_echo512_close(void *cc, void *dst)
{
    # 调用 echo_big_close 函数，关闭 echo512 算法的上下文，写入目标内存，长度为16字节
    echo_big_close(cc, 0, 0, dst, 16);
}

/* see sph_echo.h */
# 将剩余的未处理数据添加到 echo512 算法的上下文中，并关闭算法，将结果写入目标内存
void
sph_echo512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    # 调用 echo_big_close 函数，将剩余的未处理数据添加到 echo512 算法的上下文中，关闭算法，写入目标内存，长度为16字节
    echo_big_close(cc, ub, n, dst, 16);
}
#ifdef __cplusplus
}
#endif
```