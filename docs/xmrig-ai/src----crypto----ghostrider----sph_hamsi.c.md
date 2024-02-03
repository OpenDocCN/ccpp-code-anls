# `xmrig\src\crypto\ghostrider\sph_hamsi.c`

```cpp
/* $Id: hamsi.c 251 2010-10-19 14:31:51Z tp $ */
/* 
 * Hamsi implementation.
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

#include "sph_hamsi.h"

#ifdef __cplusplus
extern "C"{
#endif

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_HAMSI
#define SPH_SMALL_FOOTPRINT_HAMSI   1
#endif

#if !defined SPH_HAMSI_EXPAND_SMALL
#if SPH_SMALL_FOOTPRINT_HAMSI
#define SPH_HAMSI_EXPAND_SMALL  4
#else
#define SPH_HAMSI_EXPAND_SMALL  8
#endif
#endif

#if !defined SPH_HAMSI_EXPAND_BIG
#define SPH_HAMSI_EXPAND_BIG    8
#endif

#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif

#include "sph_hamsi_helper.c"

static const sph_u32 IV224[] = {

- 代码开头的注释，包含了代码的版本信息和许可证信息
- 引入了一些头文件和宏定义
- 定义了一个名为IV224的静态常量数组，类型为sph_u32
    # 使用SPH_C32函数对十六进制数进行转换
    SPH_C32(0xc3967a67), SPH_C32(0xc3bc6c20), SPH_C32(0x4bc3bcc3),
    SPH_C32(0xa7c3bc6b), SPH_C32(0x2c204b61), SPH_C32(0x74686f6c),
    SPH_C32(0x69656b65), SPH_C32(0x20556e69)
};

/*
 * 这个版本是 SHA-3 竞赛第二轮中 Hamsi 提交包中使用的版本；
 * UTF-8 编码是错误的，很快将在官方 Hamsi 规范中得到修正。
 *
static const sph_u32 IV224[] = {
    SPH_C32(0x3c967a67), SPH_C32(0x3cbc6c20), SPH_C32(0xb4c343c3),
    SPH_C32(0xa73cbc6b), SPH_C32(0x2c204b61), SPH_C32(0x74686f6c),
    SPH_C32(0x69656b65), SPH_C32(0x20556e69)
};
 */

static const sph_u32 IV256[] = {
    SPH_C32(0x76657273), SPH_C32(0x69746569), SPH_C32(0x74204c65),
    SPH_C32(0x7576656e), SPH_C32(0x2c204465), SPH_C32(0x70617274),
    SPH_C32(0x656d656e), SPH_C32(0x7420456c)
};

static const sph_u32 IV384[] = {
    SPH_C32(0x656b7472), SPH_C32(0x6f746563), SPH_C32(0x686e6965),
    SPH_C32(0x6b2c2043), SPH_C32(0x6f6d7075), SPH_C32(0x74657220),
    SPH_C32(0x53656375), SPH_C32(0x72697479), SPH_C32(0x20616e64),
    SPH_C32(0x20496e64), SPH_C32(0x75737472), SPH_C32(0x69616c20),
    SPH_C32(0x43727970), SPH_C32(0x746f6772), SPH_C32(0x61706879),
    SPH_C32(0x2c204b61)
};

static const sph_u32 IV512[] = {
    SPH_C32(0x73746565), SPH_C32(0x6c706172), SPH_C32(0x6b204172),
    SPH_C32(0x656e6265), SPH_C32(0x72672031), SPH_C32(0x302c2062),
    SPH_C32(0x75732032), SPH_C32(0x3434362c), SPH_C32(0x20422d33),
    SPH_C32(0x30303120), SPH_C32(0x4c657576), SPH_C32(0x656e2d48),
    SPH_C32(0x65766572), SPH_C32(0x6c65652c), SPH_C32(0x2042656c),
    SPH_C32(0x6769756d)
};

static const sph_u32 alpha_n[] = {
    SPH_C32(0xff00f0f0), SPH_C32(0xccccaaaa), SPH_C32(0xf0f0cccc),
    SPH_C32(0xff00aaaa), SPH_C32(0xccccaaaa), SPH_C32(0xf0f0ff00),
    SPH_C32(0xaaaacccc), SPH_C32(0xf0f0ff00), SPH_C32(0xf0f0cccc),
    SPH_C32(0xaaaaff00), SPH_C32(0xccccff00), SPH_C32(0xaaaaf0f0),
    SPH_C32(0xaaaaf0f0), SPH_C32(0xff00cccc), SPH_C32(0xccccf0f0),
    SPH_C32(0xff00aaaa), SPH_C32(0xccccaaaa), SPH_C32(0xff00f0f0),
    SPH_C32(0xff00aaaa), SPH_C32(0xf0f0cccc), SPH_C32(0xf0f0ff00),
    # 定义一系列32位的常量，使用SPH_C32宏进行转换
    SPH_C32(0xccccaaaa), SPH_C32(0xf0f0ff00), SPH_C32(0xaaaacccc),
    SPH_C32(0xaaaaff00), SPH_C32(0xf0f0cccc), SPH_C32(0xaaaaf0f0),
    SPH_C32(0xccccff00), SPH_C32(0xff00cccc), SPH_C32(0xaaaaf0f0),
    SPH_C32(0xff00aaaa), SPH_C32(0xccccf0f0)
// 结束大括号，表示常量数组定义结束
};

// 定义静态常量数组 alpha_f
static const sph_u32 alpha_f[] = {
    SPH_C32(0xcaf9639c), SPH_C32(0x0ff0f9c0), SPH_C32(0x639c0ff0),
    SPH_C32(0xcaf9f9c0), SPH_C32(0x0ff0f9c0), SPH_C32(0x639ccaf9),
    SPH_C32(0xf9c00ff0), SPH_C32(0x639ccaf9), SPH_C32(0x639c0ff0),
    SPH_C32(0xf9c0caf9), SPH_C32(0x0ff0caf9), SPH_C32(0xf9c0639c),
    SPH_C32(0xf9c0639c), SPH_C32(0xcaf90ff0), SPH_C32(0x0ff0639c),
    SPH_C32(0xcaf9f9c0), SPH_C32(0x0ff0f9c0), SPH_C32(0xcaf9639c),
    SPH_C32(0xcaf9f9c0), SPH_C32(0x639c0ff0), SPH_C32(0x639ccaf9),
    SPH_C32(0x0ff0f9c0), SPH_C32(0x639ccaf9), SPH_C32(0xf9c00ff0),
    SPH_C32(0xf9c0caf9), SPH_C32(0x639c0ff0), SPH_C32(0xf9c0639c),
    SPH_C32(0x0ff0caf9), SPH_C32(0xcaf90ff0), SPH_C32(0xf9c0639c),
    SPH_C32(0xcaf9f9c0), SPH_C32(0x0ff0639c)
};

// 定义宏，用于声明小状态
#define DECL_STATE_SMALL \
    sph_u32 c0, c1, c2, c3, c4, c5, c6, c7;

// 定义宏，用于读取小状态
#define READ_STATE_SMALL(sc)   do { \
        c0 = sc->h[0x0]; \
        c1 = sc->h[0x1]; \
        c2 = sc->h[0x2]; \
        c3 = sc->h[0x3]; \
        c4 = sc->h[0x4]; \
        c5 = sc->h[0x5]; \
        c6 = sc->h[0x6]; \
        c7 = sc->h[0x7]; \
    } while (0)

// 定义宏，用于写入小状态
#define WRITE_STATE_SMALL(sc)   do { \
        sc->h[0x0] = c0; \
        sc->h[0x1] = c1; \
        sc->h[0x2] = c2; \
        sc->h[0x3] = c3; \
        sc->h[0x4] = c4; \
        sc->h[0x5] = c5; \
        sc->h[0x6] = c6; \
        sc->h[0x7] = c7; \
    } while (0)

// 定义宏，用于将 m0-m7 和 c0-c7 映射到 s0-sF
#define s0   m0
#define s1   m1
#define s2   c0
#define s3   c1
#define s4   c2
#define s5   c3
#define s6   m2
#define s7   m3
#define s8   m4
#define s9   m5
#define sA   c4
#define sB   c5
#define sC   c6
#define sD   c7
#define sE   m6
#define sF   m7
# 定义一个宏，用于对四个变量进行 SBOX 变换
#define SBOX(a, b, c, d)   do { \
        # 定义一个临时变量 t
        sph_u32 t; \
        # 将 a 的值赋给 t
        t = (a); \
        # 对 a 进行 SBOX 变换
        (a) &= (c); \
        (a) ^= (d); \
        (c) ^= (b); \
        (c) ^= (a); \
        (d) |= t; \
        (d) ^= (b); \
        t ^= (c); \
        (b) = (d); \
        (d) |= t; \
        (d) ^= (a); \
        (a) &= (b); \
        t ^= (a); \
        (b) ^= (d); \
        (b) ^= t; \
        (a) = (c); \
        (c) = (b); \
        (b) = (d); \
        (d) = SPH_T32(~t); \
    } while (0)

# 定义一个宏，用于对四个变量进行 L 变换
#define L(a, b, c, d)   do { \
        # 对 a 和 c 进行循环左移
        (a) = SPH_ROTL32(a, 13); \
        (c) = SPH_ROTL32(c, 3); \
        # 对 b 和 d 进行异或操作
        (b) ^= (a) ^ (c); \
        (d) ^= (c) ^ SPH_T32((a) << 3); \
        # 对 b 和 d 进行循环左移
        (b) = SPH_ROTL32(b, 1); \
        (d) = SPH_ROTL32(d, 7); \
        # 对 a 和 c 进行异或操作
        (a) ^= (b) ^ (d); \
        (c) ^= (d) ^ SPH_T32((b) << 7); \
        # 对 a 和 c 进行循环左移
        (a) = SPH_ROTL32(a, 5); \
        (c) = SPH_ROTL32(c, 22); \
    } while (0)

# 定义一个宏，用于进行三轮的 SBOX 和 L 变换
#define ROUND_SMALL(rc, alpha)   do { \
        # 对 s0-sF 进行异或操作
        s0 ^= alpha[0x00]; \
        s1 ^= alpha[0x01] ^ (sph_u32)(rc); \
        s2 ^= alpha[0x02]; \
        s3 ^= alpha[0x03]; \
        s4 ^= alpha[0x08]; \
        s5 ^= alpha[0x09]; \
        s6 ^= alpha[0x0A]; \
        s7 ^= alpha[0x0B]; \
        s8 ^= alpha[0x10]; \
        s9 ^= alpha[0x11]; \
        sA ^= alpha[0x12]; \
        sB ^= alpha[0x13]; \
        sC ^= alpha[0x18]; \
        sD ^= alpha[0x19]; \
        sE ^= alpha[0x1A]; \
        sF ^= alpha[0x1B]; \
        # 对 s0-sF 进行 SBOX 变换
        SBOX(s0, s4, s8, sC); \
        SBOX(s1, s5, s9, sD); \
        SBOX(s2, s6, sA, sE); \
        SBOX(s3, s7, sB, sF); \
        # 对 s0-sF 进行 L 变换
        L(s0, s5, sA, sF); \
        L(s1, s6, sB, sC); \
        L(s2, s7, s8, sD); \
        L(s3, s4, s9, sE); \
    } while (0)

# 定义一个宏，用于进行三轮的 SBOX 和 L 变换
#define P_SMALL   do { \
        # 对 alpha_n 进行三轮的 SBOX 和 L 变换
        ROUND_SMALL(0, alpha_n); \
        ROUND_SMALL(1, alpha_n); \
        ROUND_SMALL(2, alpha_n); \
    } while (0)
#define PF_SMALL   do { \  # 定义一个宏，用于执行一系列小轮函数
        ROUND_SMALL(0, alpha_f); \  # 调用ROUND_SMALL函数，传入参数0和alpha_f
        ROUND_SMALL(1, alpha_f); \  # 调用ROUND_SMALL函数，传入参数1和alpha_f
        ROUND_SMALL(2, alpha_f); \  # 调用ROUND_SMALL函数，传入参数2和alpha_f
        ROUND_SMALL(3, alpha_f); \  # 调用ROUND_SMALL函数，传入参数3和alpha_f
        ROUND_SMALL(4, alpha_f); \  # 调用ROUND_SMALL函数，传入参数4和alpha_f
        ROUND_SMALL(5, alpha_f); \  # 调用ROUND_SMALL函数，传入参数5和alpha_f
    } while (0)  # 定义宏结束

#define T_SMALL   do { \  # 定义一个宏，用于执行一系列小轮函数
        /* order is important */ \  # 顺序很重要
        c7 = (sc->h[7] ^= sB); \  # 对sc->h[7]和sB进行异或操作，并赋值给c7
        c6 = (sc->h[6] ^= sA); \  # 对sc->h[6]和sA进行异或操作，并赋值给c6
        c5 = (sc->h[5] ^= s9); \  # 对sc->h[5]和s9进行异或操作，并赋值给c5
        c4 = (sc->h[4] ^= s8); \  # 对sc->h[4]和s8进行异或操作，并赋值给c4
        c3 = (sc->h[3] ^= s3); \  # 对sc->h[3]和s3进行异或操作，并赋值给c3
        c2 = (sc->h[2] ^= s2); \  # 对sc->h[2]和s2进行异或操作，并赋值给c2
        c1 = (sc->h[1] ^= s1); \  # 对sc->h[1]和s1进行异或操作，并赋值给c1
        c0 = (sc->h[0] ^= s0); \  # 对sc->h[0]和s0进行异或操作，并赋值给c0
    } while (0)  # 定义宏结束

static void  # 定义一个静态函数
hamsi_small(sph_hamsi_small_context *sc, const unsigned char *buf, size_t num)  # 函数接受sph_hamsi_small_context类型指针sc，无符号字符型指针buf和大小为num的size_t类型参数
{
    DECL_STATE_SMALL  # 声明一个小状态

#if !SPH_64  # 如果不是64位系统
    sph_u32 tmp;  # 声明一个32位无符号整数tmp
#endif

#if SPH_64  # 如果是64位系统
    sc->count += (sph_u64)num << 5;  # sc->count加上num左移5位的值
#else  # 如果不是64位系统
    tmp = SPH_T32((sph_u32)num << 5);  # tmp等于num左移5位的值
    sc->count_low = SPH_T32(sc->count_low + tmp);  # sc->count_low等于sc->count_low加上tmp的值
    sc->count_high += (sph_u32)((num >> 13) >> 14);  # sc->count_high加上num右移13位再右移14位的值
    if (sc->count_low < tmp)  # 如果sc->count_low小于tmp
        sc->count_high ++;  # sc->count_high加1
#endif

    READ_STATE_SMALL(sc);  # 调用READ_STATE_SMALL函数，传入参数sc
    while (num -- > 0) {  # 当num大于0时循环
        sph_u32 m0, m1, m2, m3, m4, m5, m6, m7;  # 声明8个32位无符号整数m0到m7

        INPUT_SMALL;  # 调用INPUT_SMALL宏
        P_SMALL;  # 调用P_SMALL宏
        T_SMALL;  # 调用T_SMALL宏
        buf += 4;  # buf加上4
    }
    WRITE_STATE_SMALL(sc);  # 调用WRITE_STATE_SMALL函数，传入参数sc
}

static void  # 定义一个静态函数
hamsi_small_final(sph_hamsi_small_context *sc, const unsigned char *buf)  # 函数接受sph_hamsi_small_context类型指针sc和无符号字符型指针buf
{
    sph_u32 m0, m1, m2, m3, m4, m5, m6, m7;  # 声明8个32位无符号整数m0到m7
    DECL_STATE_SMALL  # 声明一个小状态

    READ_STATE_SMALL(sc);  # 调用READ_STATE_SMALL函数，传入参数sc
    INPUT_SMALL;  # 调用INPUT_SMALL宏
    PF_SMALL;  # 调用PF_SMALL宏
    T_SMALL;  # 调用T_SMALL宏
    WRITE_STATE_SMALL(sc);  # 调用WRITE_STATE_SMALL函数，传入参数sc
}

static void  # 定义一个静态函数
hamsi_small_init(sph_hamsi_small_context *sc, const sph_u32 *iv)  # 函数接受sph_hamsi_small_context类型指针sc和32位无符号整数指针iv
{
    sc->partial_len = 0;  # sc->partial_len赋值为0
    memcpy(sc->h, iv, sizeof sc->h);  # 将iv的内容复制到sc->h中，大小为sc->h的大小
#if SPH_64  # 如果是64位系统
    sc->count = 0;  # sc->count赋值为0
#else  # 如果不是64位系统
    sc->count_high = sc->count_low = 0;  # sc->count_high和sc->count_low赋值为0
#endif
}

static void  # 定义一个静态函数
hamsi_small_core(sph_hamsi_small_context *sc, const void *data, size_t len)  # 函数接受sph_hamsi_small_context类型指针sc，void类型指针data和大小为len的size_t类型参数
    # 如果存在部分数据未处理
    if (sc->partial_len != 0) {
        # 计算需要补齐的字节数
        size_t mlen;
        mlen = 4 - sc->partial_len;
        # 如果剩余数据长度小于需要补齐的字节数
        if (len < mlen) {
            # 将剩余数据拷贝到部分数据中，更新部分数据长度，然后返回
            memcpy(sc->partial + sc->partial_len, data, len);
            sc->partial_len += len;
            return;
        } else {
            # 将部分数据补齐，更新剩余数据长度和数据指针，然后进行哈希计算
            memcpy(sc->partial + sc->partial_len, data, mlen);
            len -= mlen;
            data = (const unsigned char *)data + mlen;
            hamsi_small(sc, sc->partial, 1);
            sc->partial_len = 0;
        }
    }
    # 对剩余数据进行哈希计算
    hamsi_small(sc, data, (len >> 2));
    # 更新数据指针，使其指向未处理的数据部分
    data = (const unsigned char *)data + (len & ~(size_t)3);
    len &= (size_t)3;
    # 将未处理的数据拷贝到部分数据中，更新部分数据长度
    memcpy(sc->partial, data, len);
    sc->partial_len = len;
}

static void
hamsi_small_close(sph_hamsi_small_context *sc,
    unsigned ub, unsigned n, void *dst, size_t out_size_w32)
{
    unsigned char pad[12];  // 创建一个长度为12的无符号字符数组
    size_t ptr, u;  // 声明两个大小变量
    unsigned z;  // 声明一个无符号整数变量
    unsigned char *out;  // 声明一个指向无符号字符的指针变量

    ptr = sc->partial_len;  // 将 sc->partial_len 的值赋给 ptr
    memcpy(pad, sc->partial, ptr);  // 将 sc->partial 中的 ptr 个字节复制到 pad 中
#if SPH_64
    sph_enc64be(pad + 4, sc->count + (ptr << 3) + n);  // 将 sc->count + (ptr << 3) + n 的值以大端序写入 pad + 4
#else
    sph_enc32be(pad + 4, sc->count_high);  // 将 sc->count_high 的值以大端序写入 pad + 4
    sph_enc32be(pad + 8, sc->count_low + (ptr << 3) + n);  // 将 sc->count_low + (ptr << 3) + n 的值以大端序写入 pad + 8
#endif
    z = 0x80 >> n;  // 计算 0x80 右移 n 位的值
    pad[ptr ++] = ((ub & -z) | z) & 0xFF;  // 计算 ((ub & -z) | z) & 0xFF 的值，并赋给 pad[ptr]
    while (ptr < 4)  // 循环，直到 ptr 大于等于 4
        pad[ptr ++] = 0;  // 将 0 赋给 pad[ptr]，然后 ptr 自增
    hamsi_small(sc, pad, 2);  // 调用 hamsi_small 函数
    hamsi_small_final(sc, pad + 8);  // 调用 hamsi_small_final 函数
    out = dst;  // 将 dst 赋给 out
    for (u = 0; u < out_size_w32; u ++)  // 循环，直到 u 大于等于 out_size_w32
        sph_enc32be(out + (u << 2), sc->h[u]);  // 将 sc->h[u] 的值以大端序写入 out + (u << 2)
}

#define DECL_STATE_BIG \  // 定义一个宏，用于声明大状态
    sph_u32 c0, c1, c2, c3, c4, c5, c6, c7; \  // 声明一组无符号整数变量
    sph_u32 c8, c9, cA, cB, cC, cD, cE, cF;  // 继续声明一组无符号整数变量

#define READ_STATE_BIG(sc)   do { \  // 定义一个宏，用于读取大状态
        c0 = sc->h[0x0]; \  // 读取 sc->h[0x0] 的值
        c1 = sc->h[0x1]; \  // 读取 sc->h[0x1] 的值
        c2 = sc->h[0x2]; \  // 读取 sc->h[0x2] 的值
        c3 = sc->h[0x3]; \  // 读取 sc->h[0x3] 的值
        c4 = sc->h[0x4]; \  // 读取 sc->h[0x4] 的值
        c5 = sc->h[0x5]; \  // 读取 sc->h[0x5] 的值
        c6 = sc->h[0x6]; \  // 读取 sc->h[0x6] 的值
        c7 = sc->h[0x7]; \  // 读取 sc->h[0x7] 的值
        c8 = sc->h[0x8]; \  // 读取 sc->h[0x8] 的值
        c9 = sc->h[0x9]; \  // 读取 sc->h[0x9] 的值
        cA = sc->h[0xA]; \  // 读取 sc->h[0xA] 的值
        cB = sc->h[0xB]; \  // 读取 sc->h[0xB] 的值
        cC = sc->h[0xC]; \  // 读取 sc->h[0xC] 的值
        cD = sc->h[0xD]; \  // 读取 sc->h[0xD] 的值
        cE = sc->h[0xE]; \  // 读取 sc->h[0xE] 的值
        cF = sc->h[0xF]; \  // 读取 sc->h[0xF] 的值
    } while (0)

#define WRITE_STATE_BIG(sc)   do { \  // 定义一个宏，用于写入大状态
        sc->h[0x0] = c0; \  // 将 c0 的值写入 sc->h[0x0]
        sc->h[0x1] = c1; \  // 将 c1 的值写入 sc->h[0x1]
        sc->h[0x2] = c2; \  // 将 c2 的值写入 sc->h[0x2]
        sc->h[0x3] = c3; \  // 将 c3 的值写入 sc->h[0x3]
        sc->h[0x4] = c4; \  // 将 c4 的值写入 sc->h[0x4]
        sc->h[0x5] = c5; \  // 将 c5 的值写入 sc->h[0x5]
        sc->h[0x6] = c6; \  // 将 c6 的值写入 sc->h[0x6]
        sc->h[0x7] = c7; \  // 将 c7 的值写入 sc->h[0x7]
        sc->h[0x8] = c8; \  // 将 c8 的值写入 sc->h[0x8]
        sc->h[0x9] = c9; \  // 将 c9 的值写入 sc->h[0x9]
        sc->h[0xA] = cA; \  // 将 cA 的值写入 sc->h[0xA]
        sc->h[0xB] = cB; \  // 将 cB 的值写入 sc->h[0xB]
        sc->h[0xC] = cC; \  // 将 cC 的值写入 sc->h[0xC]
        sc->h[0xD] = cD; \  // 将 cD 的值写入 sc->h[0xD]
        sc->h[0xE] = cE; \  // 将 cE 的值写入 sc->h[0xE]
        sc->h[0xF] = cF; \  // 将 cF 的值写入 sc->h[0xF]
    } while (0)

#define s00   m0  // 定义宏 s00 为 m0
#define s01   m1  // 定义宏 s01 为 m1
#define s02   c0  // 定义宏 s02 为 c0
#define s03   c1  // 定义宏 s03 为 c1
#define s04   m2  // 定义宏 s04 为 m2
#define s05   m3  // 定义宏 s05 为 m3
#define s06   c2  // 定义宏 s06 为 c2
#define s07   c3  // 定义宏 s07 为 c3
#define s08   c4  // 定义宏 s08 为 c4
#define s09   c5  // 定义宏 s09 为 c5
# 定义16进制数值对应的变量名
#define s0A   m4
#define s0B   m5
#define s0C   c6
#define s0D   c7
#define s0E   m6
#define s0F   m7
#define s10   m8
#define s11   m9
#define s12   c8
#define s13   c9
#define s14   mA
#define s15   mB
#define s16   cA
#define s17   cB
#define s18   cC
#define s19   cD
#define s1A   mC
#define s1B   mD
#define s1C   cE
#define s1D   cF
#define s1E   mE
#define s1F   mF
#define ROUND_BIG(rc, alpha)   do { \  # 定义一个宏，用于执行一系列操作
        s00 ^= alpha[0x00]; \  # 对 s00 执行按位异或操作
        s01 ^= alpha[0x01] ^ (sph_u32)(rc); \  # 对 s01 执行按位异或操作，并加上 rc 的值
        s02 ^= alpha[0x02]; \  # 对 s02 执行按位异或操作
        s03 ^= alpha[0x03]; \  # 对 s03 执行按位异或操作
        s04 ^= alpha[0x04]; \  # 对 s04 执行按位异或操作
        s05 ^= alpha[0x05]; \  # 对 s05 执行按位异或操作
        s06 ^= alpha[0x06]; \  # 对 s06 执行按位异或操作
        s07 ^= alpha[0x07]; \  # 对 s07 执行按位异或操作
        s08 ^= alpha[0x08]; \  # 对 s08 执行按位异或操作
        s09 ^= alpha[0x09]; \  # 对 s09 执行按位异或操作
        s0A ^= alpha[0x0A]; \  # 对 s0A 执行按位异或操作
        s0B ^= alpha[0x0B]; \  # 对 s0B 执行按位异或操作
        s0C ^= alpha[0x0C]; \  # 对 s0C 执行按位异或操作
        s0D ^= alpha[0x0D]; \  # 对 s0D 执行按位异或操作
        s0E ^= alpha[0x0E]; \  # 对 s0E 执行按位异或操作
        s0F ^= alpha[0x0F]; \  # 对 s0F 执行按位异或操作
        s10 ^= alpha[0x10]; \  # 对 s10 执行按位异或操作
        s11 ^= alpha[0x11]; \  # 对 s11 执行按位异或操作
        s12 ^= alpha[0x12]; \  # 对 s12 执行按位异或操作
        s13 ^= alpha[0x13]; \  # 对 s13 执行按位异或操作
        s14 ^= alpha[0x14]; \  # 对 s14 执行按位异或操作
        s15 ^= alpha[0x15]; \  # 对 s15 执行按位异或操作
        s16 ^= alpha[0x16]; \  # 对 s16 执行按位异或操作
        s17 ^= alpha[0x17]; \  # 对 s17 执行按位异或操作
        s18 ^= alpha[0x18]; \  # 对 s18 执行按位异或操作
        s19 ^= alpha[0x19]; \  # 对 s19 执行按位异或操作
        s1A ^= alpha[0x1A]; \  # 对 s1A 执行按位异或操作
        s1B ^= alpha[0x1B]; \  # 对 s1B 执行按位异或操作
        s1C ^= alpha[0x1C]; \  # 对 s1C 执行按位异或操作
        s1D ^= alpha[0x1D]; \  # 对 s1D 执行按位异或操作
        s1E ^= alpha[0x1E]; \  # 对 s1E 执行按位异或操作
        s1F ^= alpha[0x1F]; \  # 对 s1F 执行按位异或操作
        SBOX(s00, s08, s10, s18); \  # 调用 SBOX 函数
        SBOX(s01, s09, s11, s19); \  # 调用 SBOX 函数
        SBOX(s02, s0A, s12, s1A); \  # 调用 SBOX 函数
        SBOX(s03, s0B, s13, s1B); \  # 调用 SBOX 函数
        SBOX(s04, s0C, s14, s1C); \  # 调用 SBOX 函数
        SBOX(s05, s0D, s15, s1D); \  # 调用 SBOX 函数
        SBOX(s06, s0E, s16, s1E); \  # 调用 SBOX 函数
        SBOX(s07, s0F, s17, s1F); \  # 调用 SBOX 函数
        L(s00, s09, s12, s1B); \  # 调用 L 函数
        L(s01, s0A, s13, s1C); \  # 调用 L 函数
        L(s02, s0B, s14, s1D); \  # 调用 L 函数
        L(s03, s0C, s15, s1E); \  # 调用 L 函数
        L(s04, s0D, s16, s1F); \  # 调用 L 函数
        L(s05, s0E, s17, s18); \  # 调用 L 函数
        L(s06, s0F, s10, s19); \  # 调用 L 函数
        L(s07, s08, s11, s1A); \  # 调用 L 函数
        L(s00, s02, s05, s07); \  # 调用 L 函数
        L(s10, s13, s15, s16); \  # 调用 L 函数
        L(s09, s0B, s0C, s0E); \  # 调用 L 函数
        L(s19, s1A, s1C, s1F); \  # 调用 L 函数
    } while (0)  # 结束宏定义

#if SPH_SMALL_FOOTPRINT_HAMSI  # 如果定义了 SPH_SMALL_FOOTPRINT_HAMSI

#define P_BIG   do { \  # 定义一个宏，用于执行一系列操作
        unsigned r; \  # 声明一个无符号整数变量 r
        for (r = 0; r < 6; r ++) \  # 循环执行6次
            ROUND_BIG(r, alpha_n); \  # 调用 ROUND_BIG 宏
    } while (0)  # 结束宏定义

#define PF_BIG   do { \  # 定义一个宏，用于执行一系列操作
        unsigned r; \  # 声明一个无符号整数变量 r
        for (r = 0; r < 12; r ++) \  # 循环执行12次
            ROUND_BIG(r, alpha_f); \  # 调用 ROUND_BIG 宏
    # 这是一个 do-while 循环的结束标记，表示循环体的结束
#else

#define P_BIG   do { \  # 定义宏 P_BIG，执行一系列 ROUND_BIG 操作
        ROUND_BIG(0, alpha_n); \  # 执行 ROUND_BIG 操作，参数为 0 和 alpha_n
        ROUND_BIG(1, alpha_n); \  # 执行 ROUND_BIG 操作，参数为 1 和 alpha_n
        ROUND_BIG(2, alpha_n); \  # 执行 ROUND_BIG 操作，参数为 2 和 alpha_n
        ROUND_BIG(3, alpha_n); \  # 执行 ROUND_BIG 操作，参数为 3 和 alpha_n
        ROUND_BIG(4, alpha_n); \  # 执行 ROUND_BIG 操作，参数为 4 和 alpha_n
        ROUND_BIG(5, alpha_n); \  # 执行 ROUND_BIG 操作，参数为 5 和 alpha_n
    } while (0)  # 宏定义结束

#define PF_BIG   do { \  # 定义宏 PF_BIG，执行一系列 ROUND_BIG 操作
        ROUND_BIG(0, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 0 和 alpha_f
        ROUND_BIG(1, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 1 和 alpha_f
        ROUND_BIG(2, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 2 和 alpha_f
        ROUND_BIG(3, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 3 和 alpha_f
        ROUND_BIG(4, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 4 和 alpha_f
        ROUND_BIG(5, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 5 和 alpha_f
        ROUND_BIG(6, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 6 和 alpha_f
        ROUND_BIG(7, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 7 和 alpha_f
        ROUND_BIG(8, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 8 和 alpha_f
        ROUND_BIG(9, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 9 和 alpha_f
        ROUND_BIG(10, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 10 和 alpha_f
        ROUND_BIG(11, alpha_f); \  # 执行 ROUND_BIG 操作，参数为 11 和 alpha_f
    } while (0)  # 宏定义结束

#endif  # 结束条件编译指令

#define T_BIG   do { \  # 定义宏 T_BIG，执行一系列赋值操作
        /* order is important */ \  # 注释：顺序很重要
        cF = (sc->h[0xF] ^= s17); \  # 对 cF 进行赋值操作，并执行异或操作
        cE = (sc->h[0xE] ^= s16); \  # 对 cE 进行赋值操作，并执行异或操作
        cD = (sc->h[0xD] ^= s15); \  # 对 cD 进行赋值操作，并执行异或操作
        cC = (sc->h[0xC] ^= s14); \  # 对 cC 进行赋值操作，并执行异或操作
        cB = (sc->h[0xB] ^= s13); \  # 对 cB 进行赋值操作，并执行异或操作
        cA = (sc->h[0xA] ^= s12); \  # 对 cA 进行赋值操作，并执行异或操作
        c9 = (sc->h[0x9] ^= s11); \  # 对 c9 进行赋值操作，并执行异或操作
        c8 = (sc->h[0x8] ^= s10); \  # 对 c8 进行赋值操作，并执行异或操作
        c7 = (sc->h[0x7] ^= s07); \  # 对 c7 进行赋值操作，并执行异或操作
        c6 = (sc->h[0x6] ^= s06); \  # 对 c6 进行赋值操作，并执行异或操作
        c5 = (sc->h[0x5] ^= s05); \  # 对 c5 进行赋值操作，并执行异或操作
        c4 = (sc->h[0x4] ^= s04); \  # 对 c4 进行赋值操作，并执行异或操作
        c3 = (sc->h[0x3] ^= s03); \  # 对 c3 进行赋值操作，并执行异或操作
        c2 = (sc->h[0x2] ^= s02); \  # 对 c2 进行赋值操作，并执行异或操作
        c1 = (sc->h[0x1] ^= s01); \  # 对 c1 进行赋值操作，并执行异或操作
        c0 = (sc->h[0x0] ^= s00); \  # 对 c0 进行赋值操作，并执行异或操作
    } while (0)  # 宏定义结束

static void  # 静态函数声明
hamsi_big(sph_hamsi_big_context *sc, const unsigned char *buf, size_t num)  # 函数定义，参数为 sc、buf 和 num
{
    DECL_STATE_BIG  # 定义状态变量
#if !SPH_64  # 如果不是 64 位系统
    sph_u32 tmp;  # 定义临时变量 tmp
#endif

#if SPH_64  # 如果是 64 位系统
    sc->count += (sph_u64)num << 6;  # sc->count 增加 num 左移 6 位的值
#else  # 如果不是 64 位系统
    tmp = SPH_T32((sph_u32)num << 6);  # 计算 (sph_u32)num 左移 6 位的值，并转换为 32 位
    sc->count_low = SPH_T32(sc->count_low + tmp);  # sc->count_low 增加 tmp 的值
    sc->count_high += (sph_u32)((num >> 13) >> 13);  # sc->count_high 增加 num 右移 13 位再右移 13 位的值
    if (sc->count_low < tmp)  # 如果 sc->count_low 小于 tmp
        sc->count_high ++;  # sc->count_high 增加 1
#endif  # 结束条件编译指令
    READ_STATE_BIG(sc);  # 读取状态变量 sc
    while (num -- > 0) {  # 循环，直到 num 减为 0
        sph_u32 m0, m1, m2, m3, m4, m5, m6, m7;  # 定义多个临时变量
        sph_u32 m8, m9, mA, mB, mC, mD, mE, mF;  # 定义多个临时变量

        INPUT_BIG;  # 输入数据
        P_BIG;  # 执行宏 P_BIG
        T_BIG;  # 执行宏 T_BIG
        buf += 8;  # buf 指针右移 8 个字节
    }
    WRITE_STATE_BIG(sc);  # 写入状态变量 sc
}
# 定义一个函数，用于处理 Hamsi 大版本的最终处理
static void
hamsi_big_final(sph_hamsi_big_context *sc, const unsigned char *buf)
{
    # 定义局部变量
    sph_u32 m0, m1, m2, m3, m4, m5, m6, m7;
    sph_u32 m8, m9, mA, mB, mC, mD, mE, mF;
    # 声明大版本状态
    DECL_STATE_BIG
    # 读取状态
    READ_STATE_BIG(sc);
    # 输入数据
    INPUT_BIG;
    # 进行部分压缩
    PF_BIG;
    # 进行 T 处理
    T_BIG;
    # 写入状态
    WRITE_STATE_BIG(sc);
}

# 定义一个函数，用于初始化 Hamsi 大版本的上下文
static void
hamsi_big_init(sph_hamsi_big_context *sc, const sph_u32 *iv)
{
    # 设置部分长度为 0
    sc->partial_len = 0;
    # 将初始向量拷贝到上下文的哈希值中
    memcpy(sc->h, iv, sizeof sc->h);
    # 根据条件编译设置计数器的初始值
    # 如果是 64 位系统，则设置 count 为 0
    # 否则，设置 count_high 和 count_low 为 0
#if SPH_64
    sc->count = 0;
#else
    sc->count_high = sc->count_low = 0;
#endif
}

# 定义一个函数，用于处理 Hamsi 大版本的核心压缩函数
static void
hamsi_big_core(sph_hamsi_big_context *sc, const void *data, size_t len)
{
    # 如果部分长度不为 0
    if (sc->partial_len != 0) {
        size_t mlen;
        # 计算剩余空间的长度
        mlen = 8 - sc->partial_len;
        # 如果输入数据长度小于剩余空间长度
        if (len < mlen) {
            # 将数据拷贝到部分数据中，更新部分长度，然后返回
            memcpy(sc->partial + sc->partial_len, data, len);
            sc->partial_len += len;
            return;
        } else {
            # 否则，将数据拷贝到部分数据中，更新长度，然后进行一次 Hamsi 处理
            memcpy(sc->partial + sc->partial_len, data, mlen);
            len -= mlen;
            data = (const unsigned char *)data + mlen;
            hamsi_big(sc, sc->partial, 1);
            sc->partial_len = 0;
        }
    }
    # 对输入数据进行 Hamsi 处理
    hamsi_big(sc, data, (len >> 3));
    data = (const unsigned char *)data + (len & ~(size_t)7);
    len &= (size_t)7;
    memcpy(sc->partial, data, len);
    sc->partial_len = len;
}

# 定义一个函数，用于关闭 Hamsi 大版本的上下文
static void
hamsi_big_close(sph_hamsi_big_context *sc,
    unsigned ub, unsigned n, void *dst, size_t out_size_w32)
{
    unsigned char pad[8];
    size_t ptr, u;
    unsigned z;
    unsigned char *out;
    # 获取部分长度
    ptr = sc->partial_len;
    # 根据条件编译设置计数器的值
    # 如果是 64 位系统，则使用 64 位编码填充 pad
    # 否则，使用 32 位编码填充 pad
#if SPH_64
    sph_enc64be(pad, sc->count + (ptr << 3) + n);
#else
    sph_enc32be(pad, sc->count_high);
    sph_enc32be(pad + 4, sc->count_low + (ptr << 3) + n);
#endif
    # 计算填充值
    z = 0x80 >> n;
    sc->partial[ptr ++] = ((ub & -z) | z) & 0xFF;
    # 填充剩余部分
    while (ptr < 8)
        sc->partial[ptr ++] = 0;
    # 对部分数据进行 Hamsi 处理
    hamsi_big(sc, sc->partial, 1);
    # 对填充数据进行最终处理
    hamsi_big_final(sc, pad);
    out = dst;
}
    # 如果输出大小为 12，则按照指定顺序将 sc->h 数组中的值转换为大端字节序，并存入输出数组 out 中
    if (out_size_w32 == 12) {
        sph_enc32be(out +  0, sc->h[ 0]);
        sph_enc32be(out +  4, sc->h[ 1]);
        sph_enc32be(out +  8, sc->h[ 3]);
        sph_enc32be(out + 12, sc->h[ 4]);
        sph_enc32be(out + 16, sc->h[ 5]);
        sph_enc32be(out + 20, sc->h[ 6]);
        sph_enc32be(out + 24, sc->h[ 8]);
        sph_enc32be(out + 28, sc->h[ 9]);
        sph_enc32be(out + 32, sc->h[10]);
        sph_enc32be(out + 36, sc->h[12]);
        sph_enc32be(out + 40, sc->h[13]);
        sph_enc32be(out + 44, sc->h[15]);
    } 
    # 如果输出大小不为 12，则遍历 sc->h 数组，将每个元素转换为大端字节序，并存入输出数组 out 中
    else {
        for (u = 0; u < 16; u ++)
            sph_enc32be(out + (u << 2), sc->h[u]);
    }
/* see sph_hamsi.h */
// 初始化 Hamsi224 算法的上下文
void
sph_hamsi224_init(void *cc)
{
    // 调用 hamsi_small_init 函数，传入上下文和 IV224 常量
    hamsi_small_init(cc, IV224);
}

/* see sph_hamsi.h */
// 执行 Hamsi224 算法的压缩函数
void
sph_hamsi224(void *cc, const void *data, size_t len)
{
    // 调用 hamsi_small_core 函数，传入上下文、数据和数据长度
    hamsi_small_core(cc, data, len);
}

/* see sph_hamsi.h */
// 完成 Hamsi224 算法的计算，生成哈希值
void
sph_hamsi224_close(void *cc, void *dst)
{
    // 调用 hamsi_small_close 函数，传入上下文、0、0、目标地址和 7
    hamsi_small_close(cc, 0, 0, dst, 7);
//    hamsi_small_init(cc, IV224);
}

/* see sph_hamsi.h */
// 添加位并完成 Hamsi224 算法的计算，生成哈希值
void
sph_hamsi224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 hamsi_small_close 函数，传入上下文、ub、n、目标地址和 7
    hamsi_small_close(cc, ub, n, dst, 7);
//    hamsi_small_init(cc, IV224);
}

// 以下代码与上述类似，只是针对不同的 Hamsi 算法版本，作用相同
/* see sph_hamsi.h */
// 定义一个函数，用于将数据添加到 Hamsi512 算法中，并关闭算法
void
sph_hamsi512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 hamsi_big_close 函数，将数据添加到 Hamsi512 算法中，并关闭算法，将结果存储在 dst 中
    hamsi_big_close(cc, ub, n, dst, 16);
    // 被注释掉的代码，调用 hamsi_big_init 函数，初始化 Hamsi512 算法，使用 IV512 作为参数
//    hamsi_big_init(cc, IV512);
}

#ifdef __cplusplus
}
#endif
```