# `xmrig\src\crypto\ghostrider\sph_cubehash.c`

```
/* $Id: cubehash.c 227 2010-06-16 17:28:38Z tp $ */
/* 
 * CubeHash implementation.
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

#include <stddef.h>  // 包含标准库stddef.h，定义了各种变量和类型
#include <string.h>  // 包含标准库string.h，提供了一些字符串处理函数
#include <limits.h>  // 包含标准库limits.h，定义了各种数据类型的取值范围

#include "sph_cubehash.h"  // 包含自定义头文件sph_cubehash.h
#ifdef __cplusplus  // 如果是 C++ 环境
extern "C"{  // 使用 C 语言的调用约定
#endif

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_CUBEHASH
#define SPH_SMALL_FOOTPRINT_CUBEHASH   1  // 如果 SPH_SMALL_FOOTPRINT 为真且 SPH_SMALL_FOOTPRINT_CUBEHASH 未定义，则定义 SPH_SMALL_FOOTPRINT_CUBEHASH 为 1
#endif

/*
 * Some tests were conducted on an Intel Core2 Q6600 (32-bit and 64-bit
 * mode), a PowerPC G3, and a MIPS-compatible CPU (Broadcom BCM3302).
 * It appears that the optimal settings are:
 *  -- full unroll, no state copy on the "big" systems (x86, PowerPC)
 *  -- unroll to 4 or 8, state copy on the "small" system (MIPS)
 */
#if SPH_SMALL_FOOTPRINT_CUBEHASH
# 如果定义了 SPH_SMALL_FOOTPRINT_CUBEHASH，则执行以下代码块

#if !defined SPH_CUBEHASH_UNROLL
# 如果未定义 SPH_CUBEHASH_UNROLL，则将其定义为 4
#define SPH_CUBEHASH_UNROLL   4
#endif
#if !defined SPH_CUBEHASH_NOCOPY
# 如果未定义 SPH_CUBEHASH_NOCOPY，则将其定义为 1
#define SPH_CUBEHASH_NOCOPY   1
#endif

#else
# 如果未定义 SPH_SMALL_FOOTPRINT_CUBEHASH，则执行以下代码块

#if !defined SPH_CUBEHASH_UNROLL
# 如果未定义 SPH_CUBEHASH_UNROLL，则将其定义为 0
#define SPH_CUBEHASH_UNROLL   0
#endif
#if !defined SPH_CUBEHASH_NOCOPY
# 如果未定义 SPH_CUBEHASH_NOCOPY，则将其定义为 0
#define SPH_CUBEHASH_NOCOPY   0
#endif

#endif

#ifdef _MSC_VER
# 如果定义了 _MSC_VER，则禁用警告 4146
#pragma warning (disable: 4146)
#endif

static const sph_u32 IV224[] = {
# 定义一个名为 IV224 的常量数组，包含32位无符号整数
    SPH_C32(0xB0FC8217), SPH_C32(0x1BEE1A90), SPH_C32(0x829E1A22),
    SPH_C32(0x6362C342), SPH_C32(0x24D91C30), SPH_C32(0x03A7AA24),
    SPH_C32(0xA63721C8), SPH_C32(0x85B0E2EF), SPH_C32(0xF35D13F3),
    SPH_C32(0x41DA807D), SPH_C32(0x21A70CA6), SPH_C32(0x1F4E9774),
    SPH_C32(0xB3E1C932), SPH_C32(0xEB0A79A8), SPH_C32(0xCDDAAA66),
    SPH_C32(0xE2F6ECAA), SPH_C32(0x0A713362), SPH_C32(0xAA3080E0),
    SPH_C32(0xD8F23A32), SPH_C32(0xCEF15E28), SPH_C32(0xDB086314),
    SPH_C32(0x7F709DF7), SPH_C32(0xACD228A4), SPH_C32(0x704D6ECE),
    SPH_C32(0xAA3EC95F), SPH_C32(0xE387C214), SPH_C32(0x3A6445FF),
    SPH_C32(0x9CAB81C3), SPH_C32(0xC73D4B98), SPH_C32(0xD277AEBE),
    SPH_C32(0xFD20151C), SPH_C32(0x00CB573E)
};

static const sph_u32 IV256[] = {
# 定义一个名为 IV256 的常量数组，包含32位无符号整数
    SPH_C32(0xEA2BD4B4), SPH_C32(0xCCD6F29F), SPH_C32(0x63117E71),
    SPH_C32(0x35481EAE), SPH_C32(0x22512D5B), SPH_C32(0xE5D94E63),
    SPH_C32(0x7E624131), SPH_C32(0xF4CC12BE), SPH_C32(0xC2D0B696),
    SPH_C32(0x42AF2070), SPH_C32(0xD0720C35), SPH_C32(0x3361DA8C),
    SPH_C32(0x28CCECA4), SPH_C32(0x8EF8AD83), SPH_C32(0x4680AC00),
    SPH_C32(0x40E5FBAB), SPH_C32(0xD89041C3), SPH_C32(0x6107FBD5),
    SPH_C32(0x6C859D41), SPH_C32(0xF0B26679), SPH_C32(0x09392549),
    SPH_C32(0x5FA25603), SPH_C32(0x65C892FD), SPH_C32(0x93CB6285),
    SPH_C32(0x2AF2B5AE), SPH_C32(0x9E4B4E60), SPH_C32(0x774ABFDD),
    SPH_C32(0x85254725), SPH_C32(0x15815AEB), SPH_C32(0x4AB6AAD6),
    SPH_C32(0x9CDAF8AF), SPH_C32(0xD6032C0A)
};

static const sph_u32 IV384[] = {
# 定义一个名为 IV384 的常量数组，包含32位无符号整数
    SPH_C32(0xE623087E), SPH_C32(0x04C00C87), SPH_C32(0x5EF46453),
    # 使用 SPH_C32 函数对给定的十六进制数进行处理
    SPH_C32(0x69524B13), SPH_C32(0x1A05C7A9), SPH_C32(0x3528DF88),
    SPH_C32(0x6BDD01B5), SPH_C32(0x5057B792), SPH_C32(0x6AA7A922),
    SPH_C32(0x649C7EEE), SPH_C32(0xF426309F), SPH_C32(0xCB629052),
    SPH_C32(0xFC8E20ED), SPH_C32(0xB3482BAB), SPH_C32(0xF89E5E7E),
    SPH_C32(0xD83D4DE4), SPH_C32(0x44BFC10D), SPH_C32(0x5FC1E63D),
    SPH_C32(0x2104E6CB), SPH_C32(0x17958F7F), SPH_C32(0xDBEAEF70),
    SPH_C32(0xB4B97E1E), SPH_C32(0x32C195F6), SPH_C32(0x6184A8E4),
    SPH_C32(0x796C2543), SPH_C32(0x23DE176D), SPH_C32(0xD33BBAEC),
    SPH_C32(0x0C12E5D2), SPH_C32(0x4EB95A7B), SPH_C32(0x2D18BA01),
    SPH_C32(0x04EE475F), SPH_C32(0x1FC5F22E)
# 定义一个静态的常量数组，包含了初始向量的值
static const sph_u32 IV512[] = {
    SPH_C32(0x2AEA2A61), SPH_C32(0x50F494D4), SPH_C32(0x2D538B8B),
    SPH_C32(0x4167D83E), SPH_C32(0x3FEE2313), SPH_C32(0xC701CF8C),
    SPH_C32(0xCC39968E), SPH_C32(0x50AC5695), SPH_C32(0x4D42C787),
    SPH_C32(0xA647A8B3), SPH_C32(0x97CF0BEF), SPH_C32(0x825B4537),
    SPH_C32(0xEEF864D2), SPH_C32(0xF22090C4), SPH_C32(0xD0E5CD33),
    SPH_C32(0xA23911AE), SPH_C32(0xFCD398D9), SPH_C32(0x148FE485),
    SPH_C32(0x1B017BEF), SPH_C32(0xB6444532), SPH_C32(0x6A536159),
    SPH_C32(0x2FF5781C), SPH_C32(0x91FA7934), SPH_C32(0x0DBADEA9),
    SPH_C32(0xD65C8A2B), SPH_C32(0xA5A70E75), SPH_C32(0xB1C62456),
    SPH_C32(0xBC796576), SPH_C32(0x1921C8F7), SPH_C32(0xE7989AF1),
    SPH_C32(0x7795D246), SPH_C32(0xD43E3B44)
};

# 定义宏，用于对32位无符号整数进行位运算
#define T32      SPH_T32
# 定义宏，用于对32位无符号整数进行循环左移
#define ROTL32   SPH_ROTL32

# 如果定义了 SPH_CUBEHASH_NOCOPY，则进行条件编译
# 定义一系列宏，用于简化对状态数组的访问
# 如果没有定义 SPH_CUBEHASH_NOCOPY，则进行条件编译
# 定义一系列宏，用于简化对状态数组的访问
    # 声明一组32位无符号整数变量
    sph_u32 x8, x9, xa, xb, xc, xd, xe, xf; \
    sph_u32 xg, xh, xi, xj, xk, xl, xm, xn; \
    sph_u32 xo, xp, xq, xr, xs, xt, xu, xv;
# 定义宏，用于将状态数组中的值赋给对应的变量
#define READ_STATE(cc)   do { \
        x0 = (cc)->state[ 0]; \  # 将状态数组中的第0个元素赋给变量x0
        x1 = (cc)->state[ 1]; \  # 将状态数组中的第1个元素赋给变量x1
        x2 = (cc)->state[ 2]; \  # 将状态数组中的第2个元素赋给变量x2
        x3 = (cc)->state[ 3]; \  # 将状态数组中的第3个元素赋给变量x3
        x4 = (cc)->state[ 4]; \  # 将状态数组中的第4个元素赋给变量x4
        x5 = (cc)->state[ 5]; \  # 将状态数组中的第5个元素赋给变量x5
        x6 = (cc)->state[ 6]; \  # 将状态数组中的第6个元素赋给变量x6
        x7 = (cc)->state[ 7]; \  # 将状态数组中的第7个元素赋给变量x7
        x8 = (cc)->state[ 8]; \  # 将状态数组中的第8个元素赋给变量x8
        x9 = (cc)->state[ 9]; \  # 将状态数组中的第9个元素赋给变量x9
        xa = (cc)->state[10]; \  # 将状态数组中的第10个元素赋给变量xa
        xb = (cc)->state[11]; \  # 将状态数组中的第11个元素赋给变量xb
        xc = (cc)->state[12]; \  # 将状态数组中的第12个元素赋给变量xc
        xd = (cc)->state[13]; \  # 将状态数组中的第13个元素赋给变量xd
        xe = (cc)->state[14]; \  # 将状态数组中的第14个元素赋给变量xe
        xf = (cc)->state[15]; \  # 将状态数组中的第15个元素赋给变量xf
        xg = (cc)->state[16]; \  # 将状态数组中的第16个元素赋给变量xg
        xh = (cc)->state[17]; \  # 将状态数组中的第17个元素赋给变量xh
        xi = (cc)->state[18]; \  # 将状态数组中的第18个元素赋给变量xi
        xj = (cc)->state[19]; \  # 将状态数组中的第19个元素赋给变量xj
        xk = (cc)->state[20]; \  # 将状态数组中的第20个元素赋给变量xk
        xl = (cc)->state[21]; \  # 将状态数组中的第21个元素赋给变量xl
        xm = (cc)->state[22]; \  # 将状态数组中的第22个元素赋给变量xm
        xn = (cc)->state[23]; \  # 将状态数组中的第23个元素赋给变量xn
        xo = (cc)->state[24]; \  # 将状态数组中的第24个元素赋给变量xo
        xp = (cc)->state[25]; \  # 将状态数组中的第25个元素赋给变量xp
        xq = (cc)->state[26]; \  # 将状态数组中的第26个元素赋给变量xq
        xr = (cc)->state[27]; \  # 将状态数组中的第27个元素赋给变量xr
        xs = (cc)->state[28]; \  # 将状态数组中的第28个元素赋给变量xs
        xt = (cc)->state[29]; \  # 将状态数组中的第29个元素赋给变量xt
        xu = (cc)->state[30]; \  # 将状态数组中的第30个元素赋给变量xu
        xv = (cc)->state[31]; \  # 将状态数组中的第31个元素赋给变量xv
    } while (0)  # 定义宏结束
#define WRITE_STATE(cc)   do { \  # 定义宏，用于将状态值写入cc对象
        (cc)->state[ 0] = x0; \  # 将x0的值写入cc对象的state数组中
        (cc)->state[ 1] = x1; \  # 将x1的值写入cc对象的state数组中
        (cc)->state[ 2] = x2; \  # 将x2的值写入cc对象的state数组中
        (cc)->state[ 3] = x3; \  # 将x3的值写入cc对象的state数组中
        (cc)->state[ 4] = x4; \  # 将x4的值写入cc对象的state数组中
        (cc)->state[ 5] = x5; \  # 将x5的值写入cc对象的state数组中
        (cc)->state[ 6] = x6; \  # 将x6的值写入cc对象的state数组中
        (cc)->state[ 7] = x7; \  # 将x7的值写入cc对象的state数组中
        (cc)->state[ 8] = x8; \  # 将x8的值写入cc对象的state数组中
        (cc)->state[ 9] = x9; \  # 将x9的值写入cc对象的state数组中
        (cc)->state[10] = xa; \  # 将xa的值写入cc对象的state数组中
        (cc)->state[11] = xb; \  # 将xb的值写入cc对象的state数组中
        (cc)->state[12] = xc; \  # 将xc的值写入cc对象的state数组中
        (cc)->state[13] = xd; \  # 将xd的值写入cc对象的state数组中
        (cc)->state[14] = xe; \  # 将xe的值写入cc对象的state数组中
        (cc)->state[15] = xf; \  # 将xf的值写入cc对象的state数组中
        (cc)->state[16] = xg; \  # 将xg的值写入cc对象的state数组中
        (cc)->state[17] = xh; \  # 将xh的值写入cc对象的state数组中
        (cc)->state[18] = xi; \  # 将xi的值写入cc对象的state数组中
        (cc)->state[19] = xj; \  # 将xj的值写入cc对象的state数组中
        (cc)->state[20] = xk; \  # 将xk的值写入cc对象的state数组中
        (cc)->state[21] = xl; \  # 将xl的值写入cc对象的state数组中
        (cc)->state[22] = xm; \  # 将xm的值写入cc对象的state数组中
        (cc)->state[23] = xn; \  # 将xn的值写入cc对象的state数组中
        (cc)->state[24] = xo; \  # 将xo的值写入cc对象的state数组中
        (cc)->state[25] = xp; \  # 将xp的值写入cc对象的state数组中
        (cc)->state[26] = xq; \  # 将xq的值写入cc对象的state数组中
        (cc)->state[27] = xr; \  # 将xr的值写入cc对象的state数组中
        (cc)->state[28] = xs; \  # 将xs的值写入cc对象的state数组中
        (cc)->state[29] = xt; \  # 将xt的值写入cc对象的state数组中
        (cc)->state[30] = xu; \  # 将xu的值写入cc对象的state数组中
        (cc)->state[31] = xv; \  # 将xv的值写入cc对象的state数组中
    } while (0)  # 宏定义结束

#endif  # 结束宏定义部分

#define INPUT_BLOCK   do { \  # 定义宏，用于处理输入块
        x0 ^= sph_dec32le_aligned(buf +  0); \  # 对x0进行异或操作
        x1 ^= sph_dec32le_aligned(buf +  4); \  # 对x1进行异或操作
        x2 ^= sph_dec32le_aligned(buf +  8); \  # 对x2进行异或操作
        x3 ^= sph_dec32le_aligned(buf + 12); \  # 对x3进行异或操作
        x4 ^= sph_dec32le_aligned(buf + 16); \  # 对x4进行异或操作
        x5 ^= sph_dec32le_aligned(buf + 20); \  # 对x5进行异或操作
        x6 ^= sph_dec32le_aligned(buf + 24); \  # 对x6进行异或操作
        x7 ^= sph_dec32le_aligned(buf + 28); \  # 对x7进行异或操作
    } while (0)  # 宏定义结束
// 定义一个包含 16 轮迭代的宏，根据 SPH_CUBEHASH_UNROLL 的值不同，展开不同次数的迭代
#if SPH_CUBEHASH_UNROLL == 1
// 如果 SPH_CUBEHASH_UNROLL 的值为 1，则展开 16 轮迭代
#define SIXTEEN_ROUNDS   do { \
        int j; \
        for (j = 0; j < 8; j ++) { \
            ROUND_EVEN; \
            ROUND_ODD; \
        } \
    } while (0)

#elif SPH_CUBEHASH_UNROLL == 4
// 如果 SPH_CUBEHASH_UNROLL 的值为 4，则展开 16 轮迭代
#define SIXTEEN_ROUNDS   do { \
        int j; \
        for (j = 0; j < 4; j ++) { \
            ROUND_EVEN; \
            ROUND_ODD; \
            ROUND_EVEN; \
            ROUND_ODD; \
        } \
    } while (0)

#elif SPH_CUBEHASH_UNROLL == 8
// 如果 SPH_CUBEHASH_UNROLL 的值为 8，则展开 16 轮迭代
#define SIXTEEN_ROUNDS   do { \
        int j; \
        for (j = 0; j < 2; j ++) { \
            ROUND_EVEN; \
            ROUND_ODD; \
            ROUND_EVEN; \
            ROUND_ODD; \
            ROUND_EVEN; \
            ROUND_ODD; \
            ROUND_EVEN; \
            ROUND_ODD; \
        } \
    } while (0)

#else
// 如果 SPH_CUBEHASH_UNROLL 的值不在上述范围内，则默认展开 16 轮迭代
#define SIXTEEN_ROUNDS   do { \
        ROUND_EVEN; \
        ROUND_ODD; \
        ROUND_EVEN; \
        ROUND_ODD; \
        ROUND_EVEN; \
        ROUND_ODD; \
        ROUND_EVEN; \
        ROUND_ODD; \
        ROUND_EVEN; \
        ROUND_ODD; \
        ROUND_EVEN; \
        ROUND_ODD; \
        ROUND_EVEN; \
        ROUND_ODD; \
        ROUND_EVEN; \
        ROUND_ODD; \
    } while (0)

#endif

// 初始化 cubehash 算法的上下文，使用给定的初始化向量 iv
static void
cubehash_init(sph_cubehash_context *sc, const sph_u32 *iv)
{
    // 将初始化向量 iv 的内容复制到状态 state 中
    memcpy(sc->state, iv, sizeof sc->state);
    // 初始化指针 ptr 为 0
    sc->ptr = 0;
}

// cubehash 算法的核心函数，用于处理输入数据
static void
cubehash_core(sph_cubehash_context *sc, const void *data, size_t len)
{
    unsigned char *buf;
    size_t ptr;
    DECL_STATE

    buf = sc->buf;
    ptr = sc->ptr;
    // 如果输入数据长度小于缓冲区剩余空间
    if (len < (sizeof sc->buf) - ptr) {
        // 将输入数据复制到缓冲区中
        memcpy(buf + ptr, data, len);
        // 更新指针位置
        ptr += len;
        sc->ptr = ptr;
        return;
    }

    // 读取状态信息
    READ_STATE(sc);
    # 当剩余长度大于 0 时，执行循环
    while (len > 0) {
        # 定义一个变量 clen，表示剩余空间大小
        size_t clen;

        # 计算剩余空间大小
        clen = (sizeof sc->buf) - ptr;
        # 如果剩余空间大于当前长度，则取当前长度
        if (clen > len)
            clen = len;
        # 将数据从 data 拷贝到 buf 中
        memcpy(buf + ptr, data, clen);
        # 更新指针位置
        ptr += clen;
        # 更新数据位置
        data = (const unsigned char *)data + clen;
        # 更新剩余长度
        len -= clen;
        # 如果指针位置达到 buf 的大小
        if (ptr == sizeof sc->buf) {
            # 输入一个数据块
            INPUT_BLOCK;
            # 执行 16 轮加密
            SIXTEEN_ROUNDS;
            # 重置指针位置
            ptr = 0;
        }
    }
    # 将状态写入 sc
    WRITE_STATE(sc);
    # 更新指针位置
    sc->ptr = ptr;
}
# 关闭 Cubehash 算法上下文，输出结果
static void
cubehash_close(sph_cubehash_context *sc, unsigned ub, unsigned n,
    void *dst, size_t out_size_w32)
{
    unsigned char *buf, *out;  # 声明两个指向无符号字符的指针变量
    size_t ptr;  # 声明一个大小为 size_t 的变量
    unsigned z;  # 声明一个无符号整数变量
    int i;  # 声明一个整数变量
    DECL_STATE  # 声明一个状态变量

    buf = sc->buf;  # 将 sc->buf 的值赋给 buf
    ptr = sc->ptr;  # 将 sc->ptr 的值赋给 ptr
    z = 0x80 >> n;  # 计算 z 的值
    buf[ptr ++] = ((ub & -z) | z) & 0xFF;  # 计算 buf[ptr] 的值
    memset(buf + ptr, 0, (sizeof sc->buf) - ptr);  # 将 buf + ptr 开始的 (sizeof sc->buf) - ptr 个字节设置为 0
    READ_STATE(sc);  # 读取状态
    INPUT_BLOCK;  # 输入块
    for (i = 0; i < 11; i ++) {  # 循环 11 次
        SIXTEEN_ROUNDS;  # 执行 16 轮
        if (i == 0)  # 如果 i 等于 0
            xv ^= SPH_C32(1);  # 执行异或操作
    }
    WRITE_STATE(sc);  # 写入状态
    out = dst;  # 将 dst 的值赋给 out
    for (z = 0; z < out_size_w32; z ++)  # 循环输出结果
        sph_enc32le(out + (z << 2), sc->state[z]);  # 将 sc->state[z] 的值以小端格式写入 out
}

/* see sph_cubehash.h */
void
sph_cubehash224_init(void *cc)
{
    cubehash_init(cc, IV224);  # 初始化 Cubehash224
}

/* see sph_cubehash.h */
void
sph_cubehash224(void *cc, const void *data, size_t len)
{
    cubehash_core(cc, data, len);  # 执行 Cubehash224 核心操作
}

# 省略部分代码注释
# 关闭 cubehash384 算法上下文，并将结果写入目标内存
sph_cubehash384_close(void *cc, void *dst)
{
    # 调用 cubehash384_addbits_and_close 函数，传入参数 0, 0, dst
    sph_cubehash384_addbits_and_close(cc, 0, 0, dst);
}

/* see sph_cubehash.h */
# cubehash384_addbits_and_close 函数，用于处理最后的数据块并关闭 cubehash384 算法上下文
void
sph_cubehash384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    # 调用 cubehash_close 函数，传入参数 cc, ub, n, dst, 12
    cubehash_close(cc, ub, n, dst, 12);
    # 重新初始化 cubehash384 算法上下文
    sph_cubehash384_init(cc);
}

/* see sph_cubehash.h */
# cubehash512_init 函数，用于初始化 cubehash512 算法上下文
void
sph_cubehash512_init(void *cc)
{
    # 调用 cubehash_init 函数，传入参数 cc, IV512
    cubehash_init(cc, IV512);
}

/* see sph_cubehash.h */
# cubehash512 函数，用于处理输入数据的 cubehash512 算法核心
void
sph_cubehash512(void *cc, const void *data, size_t len)
{
    # 调用 cubehash_core 函数，传入参数 cc, data, len
    cubehash_core(cc, data, len);
}

/* see sph_cubehash.h */
# cubehash512_close 函数，用于关闭 cubehash512 算法上下文并将结果写入目标内存
void
sph_cubehash512_close(void *cc, void *dst)
{
    # 调用 cubehash_close 函数，传入参数 cc, ub, n, dst, 16
    cubehash_close(cc, ub, n, dst, 16);
    # 重新初始化 cubehash512 算法上下文
    sph_cubehash512_init(cc);
}
#ifdef __cplusplus
}
#endif
```