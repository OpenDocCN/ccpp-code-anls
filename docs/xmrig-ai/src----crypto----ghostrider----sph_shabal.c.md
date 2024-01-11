# `xmrig\src\crypto\ghostrider\sph_shabal.c`

```
/*
 * 定义了 Shabal 算法的实现
 * 版权声明
 */
/* $Id: shabal.c 175 2010-05-07 16:03:20Z tp $ */
/*
 * Shabal implementation.
 * Shabal 算法的实现
 *
 * ==========================(LICENSE BEGIN)============================
 * 授权许可声明
 * Copyright (c) 2007-2010  Projet RNRT SAPHIR
 * 版权声明
 * Permission is hereby granted, free of charge, to any person obtaining
 * a copy of this software and associated documentation files (the
 * "Software"), to deal in the Software without restriction, including
 * without limitation the rights to use, copy, modify, merge, publish,
 * distribute, sublicense, and/or sell copies of the Software, and to
 * permit persons to whom the Software is furnished to do so, subject to
 * the following conditions:
 * 授权许可条件
 * The above copyright notice and this permission notice shall be
 * included in all copies or substantial portions of the Software.
 * 版权和许可声明应包含在所有副本或实质部分的软件中
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 * IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
 * CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
 * TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
 * SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
 * 免责声明
 * ===========================(LICENSE END)=============================
 * 授权许可声明结束
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 * 作者信息
 */

#include <stddef.h>
#include <string.h>

#include "sph_shabal.h"
#ifdef __cplusplus
extern "C"{
#endif

#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif

/*
 * Part of this code was automatically generated (the part between
 * the "BEGIN" and "END" markers).
 */
/*
 * 代码的一部分是自动生成的（在“BEGIN”和“END”标记之间的部分）。
 */

#define sM    16
/*
 * 定义 sM 为 16
 */

#define C32   SPH_C32
#define T32   SPH_T32
/*
 * 定义 C32 和 T32 为 SPH_C32 和 SPH_T32
 */

#define O1   13
#define O2    9
#define O3    6
/*
 * 定义 O1、O2 和 O3 的值分别为 13、9 和 6
 */

/*
 * We copy the state into local variables, so that the compiler knows
 * that it can optimize them at will.
 */
/*
 * 将状态复制到局部变量中，以便编译器知道它可以随意优化它们。
 */

/* BEGIN -- automatically generated code. */
/*
 * 开始 -- 自动生成的代码。
 */

#define DECL_STATE   \
/*
 * 定义 DECL_STATE
 */
    # 定义一系列32位无符号整数变量，用于存储数据
    sph_u32 A00, A01, A02, A03, A04, A05, A06, A07, \
            A08, A09, A0A, A0B; \
    sph_u32 B0, B1, B2, B3, B4, B5, B6, B7, \
            B8, B9, BA, BB, BC, BD, BE, BF; \
    sph_u32 C0, C1, C2, C3, C4, C5, C6, C7, \
            C8, C9, CA, CB, CC, CD, CE, CF; \
    sph_u32 M0, M1, M2, M3, M4, M5, M6, M7, \
            M8, M9, MA, MB, MC, MD, ME, MF; \
    sph_u32 Wlow, Whigh;
    
    
    这段代码定义了一系列32位无符号整数变量，用于存储数据。每个变量都有一个唯一的名称，例如A00、A01、B0、B1等等。这些变量将在后续的代码中使用。
# 宏定义，用于将状态结构体中的各个成员赋值给对应的变量
#define READ_STATE(state)   do { \
        A00 = (state)->A[0]; \  # 读取状态结构体中A数组的第一个元素赋值给A00变量
        A01 = (state)->A[1]; \  # 读取状态结构体中A数组的第二个元素赋值给A01变量
        A02 = (state)->A[2]; \  # 读取状态结构体中A数组的第三个元素赋值给A02变量
        A03 = (state)->A[3]; \  # 读取状态结构体中A数组的第四个元素赋值给A03变量
        A04 = (state)->A[4]; \  # 读取状态结构体中A数组的第五个元素赋值给A04变量
        A05 = (state)->A[5]; \  # 读取状态结构体中A数组的第六个元素赋值给A05变量
        A06 = (state)->A[6]; \  # 读取状态结构体中A数组的第七个元素赋值给A06变量
        A07 = (state)->A[7]; \  # 读取状态结构体中A数组的第八个元素赋值给A07变量
        A08 = (state)->A[8]; \  # 读取状态结构体中A数组的第九个元素赋值给A08变量
        A09 = (state)->A[9]; \  # 读取状态结构体中A数组的第十个元素赋值给A09变量
        A0A = (state)->A[10]; \  # 读取状态结构体中A数组的第十一个元素赋值给A0A变量
        A0B = (state)->A[11]; \  # 读取状态结构体中A数组的第十二个元素赋值给A0B变量
        B0 = (state)->B[0]; \  # 读取状态结构体中B数组的第一个元素赋值给B0变量
        B1 = (state)->B[1]; \  # 读取状态结构体中B数组的第二个元素赋值给B1变量
        B2 = (state)->B[2]; \  # 读取状态结构体中B数组的第三个元素赋值给B2变量
        B3 = (state)->B[3]; \  # 读取状态结构体中B数组的第四个元素赋值给B3变量
        B4 = (state)->B[4]; \  # 读取状态结构体中B数组的第五个元素赋值给B4变量
        B5 = (state)->B[5]; \  # 读取状态结构体中B数组的第六个元素赋值给B5变量
        B6 = (state)->B[6]; \  # 读取状态结构体中B数组的第七个元素赋值给B6变量
        B7 = (state)->B[7]; \  # 读取状态结构体中B数组的第八个元素赋值给B7变量
        B8 = (state)->B[8]; \  # 读取状态结构体中B数组的第九个元素赋值给B8变量
        B9 = (state)->B[9]; \  # 读取状态结构体中B数组的第十个元素赋值给B9变量
        BA = (state)->B[10]; \  # 读取状态结构体中B数组的第十一个元素赋值给BA变量
        BB = (state)->B[11]; \  # 读取状态结构体中B数组的第十二个元素赋值给BB变量
        BC = (state)->B[12]; \  # 读取状态结构体中B数组的第十三个元素赋值给BC变量
        BD = (state)->B[13]; \  # 读取状态结构体中B数组的第十四个元素赋值给BD变量
        BE = (state)->B[14]; \  # 读取状态结构体中B数组的第十五个元素赋值给BE变量
        BF = (state)->B[15]; \  # 读取状态结构体中B数组的第十六个元素赋值给BF变量
        C0 = (state)->C[0]; \  # 读取状态结构体中C数组的第一个元素赋值给C0变量
        C1 = (state)->C[1]; \  # 读取状态结构体中C数组的第二个元素赋值给C1变量
        C2 = (state)->C[2]; \  # 读取状态结构体中C数组的第三个元素赋值给C2变量
        C3 = (state)->C[3]; \  # 读取状态结构体中C数组的第四个元素赋值给C3变量
        C4 = (state)->C[4]; \  # 读取状态结构体中C数组的第五个元素赋值给C4变量
        C5 = (state)->C[5]; \  # 读取状态结构体中C数组的第六个元素赋值给C5变量
        C6 = (state)->C[6]; \  # 读取状态结构体中C数组的第七个元素赋值给C6变量
        C7 = (state)->C[7]; \  # 读取状态结构体中C数组的第八个元素赋值给C7变量
        C8 = (state)->C[8]; \  # 读取状态结构体中C数组的第九个元素赋值给C8变量
        C9 = (state)->C[9]; \  # 读取状态结构体中C数组的第十个元素赋值给C9变量
        CA = (state)->C[10]; \  # 读取状态结构体中C数组的第十一个元素赋值给CA变量
        CB = (state)->C[11]; \  # 读取状态结构体中C数组的第十二个元素赋值给CB变量
        CC = (state)->C[12]; \  # 读取状态结构体中C数组的第十三个元素赋值给CC变量
        CD = (state)->C[13]; \  # 读取状态结构体中C数组的第十四个元素赋值给CD变量
        CE = (state)->C[14]; \  # 读取状态结构体中C数组的第十五个元素赋值给CE变量
        CF = (state)->C[15]; \  # 读取状态结构体中C数组的第十六个元素赋值给CF变量
        Wlow = (state)->Wlow; \  # 读取状态结构体中Wlow成员的值赋值给Wlow变量
        Whigh = (state)->Whigh; \  # 读取状态结构体中Whigh成员的值赋值给Whigh变量
    } while (0)  # 使用do-while循环，实际上是为了让宏定义的多行代码作为一个整体，方便在其他地方使用
# 宏定义，用于将状态结构体中的各个成员赋值
#define WRITE_STATE(state)   do { \
        # 将状态结构体中的 A 数组的各个元素赋值
        (state)->A[0] = A00; \
        (state)->A[1] = A01; \
        (state)->A[2] = A02; \
        (state)->A[3] = A03; \
        (state)->A[4] = A04; \
        (state)->A[5] = A05; \
        (state)->A[6] = A06; \
        (state)->A[7] = A07; \
        (state)->A[8] = A08; \
        (state)->A[9] = A09; \
        (state)->A[10] = A0A; \
        (state)->A[11] = A0B; \
        # 将状态结构体中的 B 数组的各个元素赋值
        (state)->B[0] = B0; \
        (state)->B[1] = B1; \
        (state)->B[2] = B2; \
        (state)->B[3] = B3; \
        (state)->B[4] = B4; \
        (state)->B[5] = B5; \
        (state)->B[6] = B6; \
        (state)->B[7] = B7; \
        (state)->B[8] = B8; \
        (state)->B[9] = B9; \
        (state)->B[10] = BA; \
        (state)->B[11] = BB; \
        (state)->B[12] = BC; \
        (state)->B[13] = BD; \
        (state)->B[14] = BE; \
        (state)->B[15] = BF; \
        # 将状态结构体中的 C 数组的各个元素赋值
        (state)->C[0] = C0; \
        (state)->C[1] = C1; \
        (state)->C[2] = C2; \
        (state)->C[3] = C3; \
        (state)->C[4] = C4; \
        (state)->C[5] = C5; \
        (state)->C[6] = C6; \
        (state)->C[7] = C7; \
        (state)->C[8] = C8; \
        (state)->C[9] = C9; \
        (state)->C[10] = CA; \
        (state)->C[11] = CB; \
        (state)->C[12] = CC; \
        (state)->C[13] = CD; \
        (state)->C[14] = CE; \
        (state)->C[15] = CF; \
        # 将状态结构体中的 Wlow 和 Whigh 成员赋值
        (state)->Wlow = Wlow; \
        (state)->Whigh = Whigh; \
    } while (0)
# 定义解码块，将缓冲区中的数据解码为32位小端整数，并赋值给对应的变量
#define DECODE_BLOCK   do { \
        M0 = sph_dec32le_aligned(buf + 0); \
        M1 = sph_dec32le_aligned(buf + 4); \
        M2 = sph_dec32le_aligned(buf + 8); \
        M3 = sph_dec32le_aligned(buf + 12); \
        M4 = sph_dec32le_aligned(buf + 16); \
        M5 = sph_dec32le_aligned(buf + 20); \
        M6 = sph_dec32le_aligned(buf + 24); \
        M7 = sph_dec32le_aligned(buf + 28); \
        M8 = sph_dec32le_aligned(buf + 32); \
        M9 = sph_dec32le_aligned(buf + 36); \
        MA = sph_dec32le_aligned(buf + 40); \
        MB = sph_dec32le_aligned(buf + 44); \
        MC = sph_dec32le_aligned(buf + 48); \
        MD = sph_dec32le_aligned(buf + 52); \
        ME = sph_dec32le_aligned(buf + 56); \
        MF = sph_dec32le_aligned(buf + 60); \
    } while (0)

# 定义输入块加法操作，将对应的变量与M0-MF相加，并进行32位截断
#define INPUT_BLOCK_ADD   do { \
        B0 = T32(B0 + M0); \
        B1 = T32(B1 + M1); \
        B2 = T32(B2 + M2); \
        B3 = T32(B3 + M3); \
        B4 = T32(B4 + M4); \
        B5 = T32(B5 + M5); \
        B6 = T32(B6 + M6); \
        B7 = T32(B7 + M7); \
        B8 = T32(B8 + M8); \
        B9 = T32(B9 + M9); \
        BA = T32(BA + MA); \
        BB = T32(BB + MB); \
        BC = T32(BC + MC); \
        BD = T32(BD + MD); \
        BE = T32(BE + ME); \
        BF = T32(BF + MF); \
    } while (0)

# 定义输入块减法操作，将对应的变量与M0-MF相减，并进行32位截断
#define INPUT_BLOCK_SUB   do { \
        C0 = T32(C0 - M0); \
        C1 = T32(C1 - M1); \
        C2 = T32(C2 - M2); \
        C3 = T32(C3 - M3); \
        C4 = T32(C4 - M4); \
        C5 = T32(C5 - M5); \
        C6 = T32(C6 - M6); \
        C7 = T32(C7 - M7); \
        C8 = T32(C8 - M8); \
        C9 = T32(C9 - M9); \
        CA = T32(CA - MA); \
        CB = T32(CB - MB); \
        CC = T32(CC - MC); \
        CD = T32(CD - MD); \
        CE = T32(CE - ME); \
        CF = T32(CF - MF); \
    } while (0)

# 定义异或操作，将A00和A01分别与Wlow和Whigh进行异或操作
#define XOR_W   do { \
        A00 ^= Wlow; \
        A01 ^= Whigh; \
    } while (0)

# 定义交换操作，将v1和v2的值进行交换
#define SWAP(v1, v2)   do { \
        sph_u32 tmp = (v1); \
        (v1) = (v2); \
        (v2) = tmp; \
    # 使用 do-while 循环，循环体内的代码至少会执行一次
    do {
        # 循环体内的代码
    } while (0)
# 定义一个宏，用于交换变量 B 和 C 的值
#define SWAP_BC   do { \
        SWAP(B0, C0); \  # 交换 B0 和 C0 的值
        SWAP(B1, C1); \  # 交换 B1 和 C1 的值
        SWAP(B2, C2); \  # 交换 B2 和 C2 的值
        SWAP(B3, C3); \  # 交换 B3 和 C3 的值
        SWAP(B4, C4); \  # 交换 B4 和 C4 的值
        SWAP(B5, C5); \  # 交换 B5 和 C5 的值
        SWAP(B6, C6); \  # 交换 B6 和 C6 的值
        SWAP(B7, C7); \  # 交换 B7 和 C7 的值
        SWAP(B8, C8); \  # 交换 B8 和 C8 的值
        SWAP(B9, C9); \  # 交换 B9 和 C9 的值
        SWAP(BA, CA); \  # 交换 BA 和 CA 的值
        SWAP(BB, CB); \  # 交换 BB 和 CB 的值
        SWAP(BC, CC); \  # 交换 BC 和 CC 的值
        SWAP(BD, CD); \  # 交换 BD 和 CD 的值
        SWAP(BE, CE); \  # 交换 BE 和 CE 的值
        SWAP(BF, CF); \  # 交换 BF 和 CF 的值
    } while (0)  # 宏定义结束，使用 do-while 结构确保宏的完整性

# 定义一个宏，用于对变量进行置换操作
#define PERM_ELT(xa0, xa1, xb0, xb1, xb2, xb3, xc, xm)   do { \
        xa0 = T32((xa0 \  # 对 xa0 进行置换操作
            ^ (((xa1 << 15) | (xa1 >> 17)) * 5U) \  # 对 xa1 进行位移和乘法操作，并与 xa0 进行异或
            ^ xc) * 3U) \  # 对结果再进行异或和乘法操作
            ^ xb1 ^ (xb2 & ~xb3) ^ xm; \  # 对结果再进行异或和按位与操作
        xb0 = T32(~(((xb0 << 1) | (xb0 >> 31)) ^ xa0)); \  # 对 xb0 进行位移、异或和取反操作
    } while (0)  # 宏定义结束，使用 do-while 结构确保宏的完整性

# 定义一个宏，用于执行置换操作的一步
#define PERM_STEP_0   do { \
        PERM_ELT(A00, A0B, B0, BD, B9, B6, C8, M0); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A01, A00, B1, BE, BA, B7, C7, M1); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A02, A01, B2, BF, BB, B8, C6, M2); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A03, A02, B3, B0, BC, B9, C5, M3); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A04, A03, B4, B1, BD, BA, C4, M4); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A05, A04, B5, B2, BE, BB, C3, M5); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A06, A05, B6, B3, BF, BC, C2, M6); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A07, A06, B7, B4, B0, BD, C1, M7); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A08, A07, B8, B5, B1, BE, C0, M8); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A09, A08, B9, B6, B2, BF, CF, M9); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A0A, A09, BA, B7, B3, B0, CE, MA); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A0B, A0A, BB, B8, B4, B1, CD, MB); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A00, A0B, BC, B9, B5, B2, CC, MC); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A01, A00, BD, BA, B6, B3, CB, MD); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A02, A01, BE, BB, B7, B4, CA, ME); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
        PERM_ELT(A03, A02, BF, BC, B8, B5, C9, MF); \  # 执行 PERM_ELT 宏，对指定变量进行置换操作
    } while (0)  # 宏定义结束，使用 do-while 结构确保宏的完整性
#define PERM_STEP_1   do { \  // 定义宏 PERM_STEP_1，表示第一步的排列操作
        PERM_ELT(A04, A03, B0, BD, B9, B6, C8, M0); \  // 调用 PERM_ELT 宏进行排列操作
        PERM_ELT(A05, A04, B1, BE, BA, B7, C7, M1); \  // 调用 PERM_ELT 宏进行排列操作
        // ... 其余类似操作
    } while (0)  // 宏定义结束

#define PERM_STEP_2   do { \  // 定义宏 PERM_STEP_2，表示第二步的排列操作
        PERM_ELT(A08, A07, B0, BD, B9, B6, C8, M0); \  // 调用 PERM_ELT 宏进行排列操作
        PERM_ELT(A09, A08, B1, BE, BA, B7, C7, M1); \  // 调用 PERM_ELT 宏进行排列操作
        // ... 其余类似操作
    } while (0)  // 宏定义结束
# 定义一个宏，用于对变量进行一系列位移和逻辑运算操作
#define APPLY_P   do { \
        # 对变量B0进行位移和逻辑运算
        B0 = T32(B0 << 17) | (B0 >> 15); \
        # 对变量B1进行位移和逻辑运算
        B1 = T32(B1 << 17) | (B1 >> 15); \
        # 对变量B2进行位移和逻辑运算
        B2 = T32(B2 << 17) | (B2 >> 15); \
        # 对变量B3进行位移和逻辑运算
        B3 = T32(B3 << 17) | (B3 >> 15); \
        # 对变量B4进行位移和逻辑运算
        B4 = T32(B4 << 17) | (B4 >> 15); \
        # 对变量B5进行位移和逻辑运算
        B5 = T32(B5 << 17) | (B5 >> 15); \
        # 对变量B6进行位移和逻辑运算
        B6 = T32(B6 << 17) | (B6 >> 15); \
        # 对变量B7进行位移和逻辑运算
        B7 = T32(B7 << 17) | (B7 >> 15); \
        # 对变量B8进行位移和逻辑运算
        B8 = T32(B8 << 17) | (B8 >> 15); \
        # 对变量B9进行位移和逻辑运算
        B9 = T32(B9 << 17) | (B9 >> 15); \
        # 对变量BA进行位移和逻辑运算
        BA = T32(BA << 17) | (BA >> 15); \
        # 对变量BB进行位移和逻辑运算
        BB = T32(BB << 17) | (BB >> 15); \
        # 对变量BC进行位移和逻辑运算
        BC = T32(BC << 17) | (BC >> 15); \
        # 对变量BD进行位移和逻辑运算
        BD = T32(BD << 17) | (BD >> 15); \
        # 对变量BE进行位移和逻辑运算
        BE = T32(BE << 17) | (BE >> 15); \
        # 对变量BF进行位移和逻辑运算
        BF = T32(BF << 17) | (BF >> 15); \
        # 执行一系列置换步骤
        PERM_STEP_0; \
        PERM_STEP_1; \
        PERm_STEP_2; \
        # 对A0B进行加法运算
        A0B = T32(A0B + C6); \
        # 对A0A进行加法运算
        A0A = T32(A0A + C5); \
        # 对A09进行加法运算
        A09 = T32(A09 + C4); \
        # 对A08进行加法运算
        A08 = T32(A08 + C3); \
        # 对A07进行加法运算
        A07 = T32(A07 + C2); \
        # 对A06进行加法运算
        A06 = T32(A06 + C1); \
        # 对A05进行加法运算
        A05 = T32(A05 + C0); \
        # 对A04进行加法运算
        A04 = T32(A04 + CF); \
        # 对A03进行加法运算
        A03 = T32(A03 + CE); \
        # 对A02进行加法运算
        A02 = T32(A02 + CD); \
        # 对A01进行加法运算
        A01 = T32(A01 + CC); \
        # 对A00进行加法运算
        A00 = T32(A00 + CB); \
        # 对A0B进行加法运算
        A0B = T32(A0B + CA); \
        # 对A0A进行加法运算
        A0A = T32(A0A + C9); \
        # 对A09进行加法运算
        A09 = T32(A09 + C8); \
        # 对A08进行加法运算
        A08 = T32(A08 + C7); \
        # 对A07进行加法运算
        A07 = T32(A07 + C6); \
        # 对A06进行加法运算
        A06 = T32(A06 + C5); \
        # 对A05进行加法运算
        A05 = T32(A05 + C4); \
        # 对A04进行加法运算
        A04 = T32(A04 + C3); \
        # 对A03进行加法运算
        A03 = T32(A03 + C2); \
        # 对A02进行加法运算
        A02 = T32(A02 + C1); \
        # 对A01进行加法运算
        A01 = T32(A01 + C0); \
        # 对A00进行加法运算
        A00 = T32(A00 + CF); \
        # 对A0B进行加法运算
        A0B = T32(A0B + CE); \
        # 对A0A进行加法运算
        A0A = T32(A0A + CD); \
        # 对A09进行加法运算
        A09 = T32(A09 + CC); \
        # 对A08进行加法运算
        A08 = T32(A08 + CB); \
        # 对A07进行加法运算
        A07 = T32(A07 + CA); \
        # 对A06进行加法运算
        A06 = T32(A06 + C9); \
        # 对A05进行加法运算
        A05 = T32(A05 + C8); \
        # 对A04进行加法运算
        A04 = T32(A04 + C7); \
        # 对A03进行加法运算
        A03 = T32(A03 + C6); \
        # 对A02进行加法运算
        A02 = T32(A02 + C5); \
        # 对A01进行加法运算
        A01 = T32(A01 + C4); \
        # 对A00进行加法运算
        A00 = T32(A00 + C3); \
    } while (0)
# 定义宏，用于增加 Wlow 的值
#define INCR_W   do { \
        if ((Wlow = T32(Wlow + 1)) == 0) \
            Whigh = T32(Whigh + 1); \
    } while (0)

# 定义初始常量 A_init_192
static const sph_u32 A_init_192[] = {
    C32(0xFD749ED4), C32(0xB798E530), C32(0x33904B6F), C32(0x46BDA85E),
    C32(0x076934B4), C32(0x454B4058), C32(0x77F74527), C32(0xFB4CF465),
    C32(0x62931DA9), C32(0xE778C8DB), C32(0x22B3998E), C32(0xAC15CFB9)
}

# 定义初始常量 B_init_192
static const sph_u32 B_init_192[] = {
    C32(0x58BCBAC4), C32(0xEC47A08E), C32(0xAEE933B2), C32(0xDFCBC824),
    C32(0xA7944804), C32(0xBF65BDB0), C32(0x5A9D4502), C32(0x59979AF7),
    C32(0xC5CEA54E), C32(0x4B6B8150), C32(0x16E71909), C32(0x7D632319),
    C32(0x930573A0), C32(0xF34C63D1), C32(0xCAF914B4), C32(0xFDD6612C)
}

# 定义初始常量 C_init_192
static const sph_u32 C_init_192[] = {
    C32(0x61550878), C32(0x89EF2B75), C32(0xA1660C46), C32(0x7EF3855B),
    C32(0x7297B58C), C32(0x1BC67793), C32(0x7FB1C723), C32(0xB66FC640),
    C32(0x1A48B71C), C32(0xF0976D17), C32(0x088CE80A), C32(0xA454EDF3),
    C32(0x1C096BF4), C32(0xAC76224B), C32(0x5215781C), C32(0xCD5D2669)
}

# 定义初始常量 A_init_224
static const sph_u32 A_init_224[] = {
    C32(0xA5201467), C32(0xA9B8D94A), C32(0xD4CED997), C32(0x68379D7B),
    C32(0xA7FC73BA), C32(0xF1A2546B), C32(0x606782BF), C32(0xE0BCFD0F),
    C32(0x2F25374E), C32(0x069A149F), C32(0x5E2DFF25), C32(0xFAECF061)
}

# 定义初始常量 B_init_224
static const sph_u32 B_init_224[] = {
    C32(0xEC9905D8), C32(0xF21850CF), C32(0xC0A746C8), C32(0x21DAD498),
    C32(0x35156EEB), C32(0x088C97F2), C32(0x26303E40), C32(0x8A2D4FB5),
    C32(0xFEEE44B6), C32(0x8A1E9573), C32(0x7B81111A), C32(0xCBC139F0),
    C32(0xA3513861), C32(0x1D2C362E), C32(0x918C580E), C32(0xB58E1B9C)
}

# 定义初始常量 C_init_224
static const sph_u32 C_init_224[] = {
    C32(0xE4B573A1), C32(0x4C1A0880), C32(0x1E907C51), C32(0x04807EFD),
    C32(0x3AD8CDE5), C32(0x16B21302), C32(0x02512C53), C32(0x2204CB18),
    C32(0x99405F2D), C32(0xE5B648A1), C32(0x70AB1D43), C32(0xA10C25C2),
    C32(0x16F1AC05), C32(0x38BBEB56), C32(0x9B01DC60), C32(0xB1096D83)
}

# 定义初始常量 A_init_256
static const sph_u32 A_init_256[] = {
    # 使用C32函数对十六进制数进行转换
    C32(0x52F84552), C32(0xE54B7999), C32(0x2D8EE3EC), C32(0xB9645191),
    C32(0xE0078B86), C32(0xBB7C44C9), C32(0xD2B5C1CA), C32(0xB0D2EB8C),
    C32(0x14CE5A45), C32(0x22AF50DC), C32(0xEFFDBC6B), C32(0xEB21B74A)
# 定义一个静态的无符号32位整数数组，用于初始化B
static const sph_u32 B_init_256[] = {
    # 初始化B的每个元素，使用C32宏将十六进制数转换为32位整数
    C32(0xB555C6EE), C32(0x3E710596), C32(0xA72A652F), C32(0x9301515F),
    C32(0xDA28C1FA), C32(0x696FD868), C32(0x9CB6BF72), C32(0x0AFE4002),
    C32(0xA6E03615), C32(0x5138C1D4), C32(0xBE216306), C32(0xB38B8890),
    C32(0x3EA8B96B), C32(0x3299ACE4), C32(0x30924DD4), C32(0x55CB34A5)
};

# 定义一个静态的无符号32位整数数组，用于初始化C
static const sph_u32 C_init_256[] = {
    # 初始化C的每个元素，使用C32宏将十六进制数转换为32位整数
    C32(0xB405F031), C32(0xC4233EBA), C32(0xB3733979), C32(0xC0DD9D55),
    C32(0xC51C28AE), C32(0xA327B8E1), C32(0x56C56167), C32(0xED614433),
    C32(0x88B59D60), C32(0x60E2CEBA), C32(0x758B4B8B), C32(0x83E82A7F),
    C32(0xBC968828), C32(0xE6E00BF7), C32(0xBA839E55), C32(0x9B491C60)
};

# 定义一个静态的无符号32位整数数组，用于初始化A
static const sph_u32 A_init_384[] = {
    # 初始化A的每个元素，使用C32宏将十六进制数转换为32位整数
    C32(0xC8FCA331), C32(0xE55C504E), C32(0x003EBF26), C32(0xBB6B8D83),
    C32(0x7B0448C1), C32(0x41B82789), C32(0x0A7C9601), C32(0x8D659CFF),
    C32(0xB6E2673E), C32(0xCA54C77B), C32(0x1460FD7E), C32(0x3FCB8F2D)
};

# 定义一个静态的无符号32位整数数组，用于初始化B
static const sph_u32 B_init_384[] = {
    # 初始化B的每个元素，使用C32宏将十六进制数转换为32位整数
    C32(0x527291FC), C32(0x2A16455F), C32(0x78E627E5), C32(0x944F169F),
    C32(0x1CA6F016), C32(0xA854EA25), C32(0x8DB98ABE), C32(0xF2C62641),
    C32(0x30117DCB), C32(0xCF5C4309), C32(0x93711A25), C32(0xF9F671B8),
    C32(0xB01D2116), C32(0x333F4B89), C32(0xB285D165), C32(0x86829B36)
};

# 定义一个静态的无符号32位整数数组，用于初始化C
static const sph_u32 C_init_384[] = {
    # 初始化C的每个元素，使用C32宏将十六进制数转换为32位整数
    C32(0xF764B11A), C32(0x76172146), C32(0xCEF6934D), C32(0xC6D28399),
    C32(0xFE095F61), C32(0x5E6018B4), C32(0x5048ECF5), C32(0x51353261),
    C32(0x6E6E36DC), C32(0x63130DAD), C32(0xA9C69BD6), C32(0x1E90EA0C),
    C32(0x7C35073B), C32(0x28D95E6D), C32(0xAA340E0D), C32(0xCB3DEE70)
};

# 定义一个静态的无符号32位整数数组，用于初始化A
static const sph_u32 A_init_512[] = {
    # 初始化A的每个元素，使用C32宏将十六进制数转换为32位整数
    C32(0x20728DFD), C32(0x46C0BD53), C32(0xE782B699), C32(0x55304632),
    C32(0x71B4EF90), C32(0x0EA9E82C), C32(0xDBB930F1), C32(0xFAD06B8B),
    C32(0xBE0CAE40), C32(0x8BD14410), C32(0x76D2ADAC), C32(0x28ACAB7F)
};

# 定义一个静态的无符号32位整数数组，用于初始化B
static const sph_u32 B_init_512[] = {
    # 初始化B的每个元素，使用C32宏将十六进制数转换为32位整数
    C32(0xC1099CB7), C32(0x07B385F3), C32(0xE7442C26), C32(0xCC8AD640),
    # 这些是一系列调用函数C32的操作，每个函数调用都传入一个十六进制数作为参数
    C32(0xEB6F56C7), C32(0x1EA81AA9), C32(0x73B9D314), C32(0x1DE85D08),
    C32(0x48910A5A), C32(0x893B22DB), C32(0xC5A0DF44), C32(0xBBC4324E),
    C32(0x72D2F240), C32(0x75941D99), C32(0x6D8BDE82), C32(0xA1A7502B)
};

static const sph_u32 C_init_512[] = {
    C32(0xD9BF68D1), C32(0x58BAD750), C32(0x56028CB2), C32(0x8134F359),
    C32(0xB5D469D8), C32(0x941A8CC2), C32(0x418B2A6E), C32(0x04052780),
    C32(0x7F07D787), C32(0x5194358F), C32(0x3C60D665), C32(0xBE97D79A),
    C32(0x950C3434), C32(0xAED9A06D), C32(0x2537DC8D), C32(0x7CDB5969)
};

/* END -- automatically generated code. */

static void
shabal_init(void *cc, unsigned size)
{
    /*
     * We have precomputed initial states for all the supported
     * output bit lengths.
     */
    const sph_u32 *A_init, *B_init, *C_init;
    sph_shabal_context *sc;

    switch (size) {
    case 192:
        A_init = A_init_192;
        B_init = B_init_192;
        C_init = C_init_192;
        break;
    case 224:
        A_init = A_init_224;
        B_init = B_init_224;
        C_init = C_init_224;
        break;
    case 256:
        A_init = A_init_256;
        B_init = B_init_256;
        C_init = C_init_256;
        break;
    case 384:
        A_init = A_init_384;
        B_init = B_init_384;
        C_init = C_init_384;
        break;
    case 512:
        A_init = A_init_512;
        B_init = B_init_512;
        C_init = C_init_512;
        break;
    default:
        return;
    }
    sc = cc;
    memcpy(sc->A, A_init, sizeof sc->A);
    memcpy(sc->B, B_init, sizeof sc->B);
    memcpy(sc->C, C_init, sizeof sc->C);
    sc->Wlow = 1;
    sc->Whigh = 0;
    sc->ptr = 0;
}

static void
shabal_core(void *cc, const unsigned char *data, size_t len)
{
    sph_shabal_context *sc;
    unsigned char *buf;
    size_t ptr;
    DECL_STATE

    sc = cc;
    buf = sc->buf;
    ptr = sc->ptr;

    /*
     * We do not want to copy the state to local variables if the
     * amount of data is less than what is needed to complete the
     * current block. Note that it is anyway suboptimal to call
     * this method many times for small chunks of data.
     */
    # 如果剩余空间足够存放数据
    if (len < (sizeof sc->buf) - ptr) {
        # 将数据拷贝到缓冲区中
        memcpy(buf + ptr, data, len);
        # 更新指针位置
        ptr += len;
        sc->ptr = ptr;
        # 返回
        return;
    }

    # 读取状态
    READ_STATE(sc);
    # 当还有数据需要处理时
    while (len > 0) {
        size_t clen;

        # 计算剩余空间大小
        clen = (sizeof sc->buf) - ptr;
        # 如果剩余空间大于数据长度，则取数据长度，否则取剩余空间大小
        if (clen > len)
            clen = len;
        # 将数据拷贝到缓冲区中
        memcpy(buf + ptr, data, clen);
        # 更新指针位置
        ptr += clen;
        data += clen;
        len -= clen;
        # 如果缓冲区已满
        if (ptr == sizeof sc->buf) {
            # 解码数据块
            DECODE_BLOCK;
            # 输入块加法
            INPUT_BLOCK_ADD;
            # 异或操作
            XOR_W;
            # 应用置换
            APPLY_P;
            # 输入块减法
            INPUT_BLOCK_SUB;
            # 交换 B 和 C
            SWAP_BC;
            # 增加 W
            INCR_W;
            # 重置指针位置
            ptr = 0;
        }
    }
    # 写入状态
    WRITE_STATE(sc);
    sc->ptr = ptr;
# 关闭 Shabal 哈希函数，输出最终结果
static void
shabal_close(void *cc, unsigned ub, unsigned n, void *dst, unsigned size_words)
{
    # 声明 Shabal 上下文结构体指针
    sph_shabal_context *sc;
    # 声明缓冲区指针
    unsigned char *buf;
    # 声明缓冲区指针
    size_t ptr;
    # 声明循环变量
    int i;
    # 声明临时变量
    unsigned z;
    # 声明联合体，用于存储临时输出结果
    union {
        unsigned char tmp_out[64];
        sph_u32 dummy;
    } u;
    # 声明输出结果长度
    size_t out_len;
    # 声明状态变量
    DECL_STATE

    # 将输入参数转换为 Shabal 上下文结构体指针
    sc = cc;
    # 获取缓冲区指针
    buf = sc->buf;
    # 获取缓冲区指针
    ptr = sc->ptr;
    # 计算填充字节
    z = 0x80 >> n;
    # 将填充字节写入缓冲区
    buf[ptr] = ((ub & -z) | z) & 0xFF;
    # 将缓冲区剩余部分填充为0
    memset(buf + ptr + 1, 0, (sizeof sc->buf) - (ptr + 1));
    # 读取状态
    READ_STATE(sc);
    # 解码块
    DECODE_BLOCK;
    # 输入块添加
    INPUT_BLOCK_ADD;
    # 异或操作
    XOR_W;
    # 应用置换
    APPLY_P;
    # 进行3轮置换操作
    for (i = 0; i < 3; i ++) {
        # 交换 B 和 C
        SWAP_BC;
        # 异或操作
        XOR_W;
        # 应用置换
        APPLY_P;
    }

    """
    我们只使用局部变量，不需要通过状态结构体进行操作。
    为了共享一些代码，我们将相关的字写入临时缓冲区，
    最后将其复制到目标数组中。
    """
    # 根据输出结果长度选择不同的处理方式
    switch (size_words) {
    case 16:
        # 将 B0-B3 写入临时输出缓冲区
        sph_enc32le_aligned(u.tmp_out +  0, B0);
        sph_enc32le_aligned(u.tmp_out +  4, B1);
        sph_enc32le_aligned(u.tmp_out +  8, B2);
        sph_enc32le_aligned(u.tmp_out + 12, B3);
        # 继续执行下一个 case
        /* fall through */
    case 12:
        # 将 B4-B7 写入临时输出缓冲区
        sph_enc32le_aligned(u.tmp_out + 16, B4);
        sph_enc32le_aligned(u.tmp_out + 20, B5);
        sph_enc32le_aligned(u.tmp_out + 24, B6);
        sph_enc32le_aligned(u.tmp_out + 28, B7);
        # 继续执行下一个 case
        /* fall through */
    case 8:
        # 将 B8 写入临时输出缓冲区
        sph_enc32le_aligned(u.tmp_out + 32, B8);
        # 继续执行下一个 case
        /* fall through */
    case 7:
        # 将 B9 写入临时输出缓冲区
        sph_enc32le_aligned(u.tmp_out + 36, B9);
        # 继续执行下一个 case
        /* fall through */
    case 6:
        # 将 BA-BF 写入临时输出缓冲区
        sph_enc32le_aligned(u.tmp_out + 40, BA);
        sph_enc32le_aligned(u.tmp_out + 44, BB);
        sph_enc32le_aligned(u.tmp_out + 48, BC);
        sph_enc32le_aligned(u.tmp_out + 52, BD);
        sph_enc32le_aligned(u.tmp_out + 56, BE);
        sph_enc32le_aligned(u.tmp_out + 60, BF);
        break;
    default:
        # 输出结果长度不符合要求，直接返回
        return;
    }
    # 计算输出结果长度
    out_len = size_words << 2;
    # 使用 memcpy 函数将 u.tmp_out 数组中从末尾向前数 out_len 长度的数据复制到 dst 数组中
    memcpy(dst, u.tmp_out + (sizeof u.tmp_out) - out_len, out_len);
#if 0
/* see sph_shabal.h */
void
sph_shabal192_init(void *cc)
{
    // 调用 shabal_init 函数，初始化 cc 指向的对象，大小为 192 字节
    shabal_init(cc, 192);
}

/* see sph_shabal.h */
void
sph_shabal192(void *cc, const void *data, size_t len)
{
    // 调用 shabal_core 函数，处理数据 data，长度为 len 字节
    shabal_core(cc, data, len);
}

/* see sph_shabal.h */
void
sph_shabal192_close(void *cc, void *dst)
{
    // 调用 shabal_close 函数，关闭 cc 指向的对象，将结果存储到 dst 指向的位置，长度为 6 字节
    shabal_close(cc, 0, 0, dst, 6);
}

/* see sph_shabal.h */
void
sph_shabal192_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 shabal_close 函数，关闭 cc 指向的对象，将结果存储到 dst 指向的位置，长度为 6 字节，并添加额外的位数信息
    shabal_close(cc, ub, n, dst, 6);
}

/* see sph_shabal.h */
void
sph_shabal224_init(void *cc)
{
    // 调用 shabal_init 函数，初始化 cc 指向的对象，大小为 224 字节
    shabal_init(cc, 224);
}

/* see sph_shabal.h */
void
sph_shabal224(void *cc, const void *data, size_t len)
{
    // 调用 shabal_core 函数，处理数据 data，长度为 len 字节
    shabal_core(cc, data, len);
}

/* see sph_shabal.h */
void
sph_shabal224_close(void *cc, void *dst)
{
    // 调用 shabal_close 函数，关闭 cc 指向的对象，将结果存储到 dst 指向的位置，长度为 7 字节
    shabal_close(cc, 0, 0, dst, 7);
}

/* see sph_shabal.h */
void
sph_shabal224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 shabal_close 函数，关闭 cc 指向的对象，将结果存储到 dst 指向的位置，长度为 7 字节，并添加额外的位数信息
    shabal_close(cc, ub, n, dst, 7);
}

#endif
/* see sph_shabal.h */
void
sph_shabal256_init(void *cc)
{
    // 调用 shabal_init 函数，初始化 cc 指向的对象，大小为 256 字节
    shabal_init(cc, 256);
}

/* see sph_shabal.h */
void
sph_shabal256(void *cc, const void *data, size_t len)
{
    // 调用 shabal_core 函数，处理数据 data，长度为 len 字节
    shabal_core(cc, data, len);
}

/* see sph_shabal.h */
void
sph_shabal256_close(void *cc, void *dst)
{
    // 调用 shabal_close 函数，关闭 cc 指向的对象，将结果存储到 dst 指向的位置，长度为 8 字节
    shabal_close(cc, 0, 0, dst, 8);
}

/* see sph_shabal.h */
void
sph_shabal256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 shabal_close 函数，关闭 cc 指向的对象，将结果存储到 dst 指向的位置，长度为 8 字节，并添加额外的位数信息
    shabal_close(cc, ub, n, dst, 8);
}

#if 0
/* see sph_shabal.h */
void
sph_shabal384_init(void *cc)
{
    // 调用 shabal_init 函数，初始化 cc 指向的对象，大小为 384 字节
    shabal_init(cc, 384);
}

/* see sph_shabal.h */
void
sph_shabal384(void *cc, const void *data, size_t len)
{
    // 调用 shabal_core 函数，处理数据 data，长度为 len 字节
    shabal_core(cc, data, len);
}

/* see sph_shabal.h */
void
sph_shabal384_close(void *cc, void *dst)
{
    // 调用 shabal_close 函数，关闭 cc 指向的对象，将结果存储到 dst 指向的位置，长度为 12 字节
    shabal_close(cc, 0, 0, dst, 12);
}

/* see sph_shabal.h */
void
sph_shabal384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 shabal_close 函数，关闭 cc 指向的对象，将结果存储到 dst 指向的位置，长度为 12 字节，并添加额外的位数信息
    shabal_close(cc, ub, n, dst, 12);
}
#endif

/* see sph_shabal.h */
void
sph_shabal512_init(void *cc)
{
    // 调用 shabal_init 函数，初始化 cc 指向的对象，大小为 512 字节
    shabal_init(cc, 512);
}
    # 初始化 Shabal 算法，使用 cc 作为输入参数，设置哈希值的位数为 512 位
/* see sph_shabal.h */
// 定义了一个名为sph_shabal512的函数，接受一个指向void类型的指针cc和一个指向const void类型的指针data以及一个size_t类型的len作为参数
void
sph_shabal512(void *cc, const void *data, size_t len)
{
    // 调用shabal_core函数，传入cc、data和len作为参数
    shabal_core(cc, data, len);
}

/* see sph_shabal.h */
// 定义了一个名为sph_shabal512_close的函数，接受一个指向void类型的指针cc和一个指向void类型的指针dst作为参数
void
sph_shabal512_close(void *cc, void *dst)
{
    // 调用shabal_close函数，传入cc、0、0、dst和16作为参数
    shabal_close(cc, 0, 0, dst, 16);
}

/* see sph_shabal.h */
// 定义了一个名为sph_shabal512_addbits_and_close的函数，接受一个指向void类型的指针cc、两个unsigned类型的参数ub和n，以及一个指向void类型的指针dst作为参数
void
sph_shabal512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用shabal_close函数，传入cc、ub、n、dst和16作为参数
    shabal_close(cc, ub, n, dst, 16);
}
#ifdef __cplusplus
}
#endif
```