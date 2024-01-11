# `xmrig\src\crypto\ghostrider\sph_sha2.c`

```
/* $Id: sha2.c 227 2010-06-16 17:28:38Z tp $ */
/*
 * SHA-224 / SHA-256 implementation.
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

#include "sph_sha2.h"

#if SPH_SMALL_FOOTPRINT && !defined SPH_SMALL_FOOTPRINT_SHA2
#define SPH_SMALL_FOOTPRINT_SHA2   1
#endif

#define CH(X, Y, Z)    ((((Y) ^ (Z)) & (X)) ^ (Z))
//#define MAJ(X, Y, Z)   (((Y) & (Z)) | (((Y) | (Z)) & (X)))
#define MAJ( X, Y, Z )   ( Y  ^ ( ( X_xor_Y = X ^ Y ) & ( Y_xor_Z ) ) )
#define ROTR    SPH_ROTR32

#define BSG2_0(x)      (ROTR(x, 2) ^ ROTR(x, 13) ^ ROTR(x, 22))
#define BSG2_1(x)      (ROTR(x, 6) ^ ROTR(x, 11) ^ ROTR(x, 25))


注释：这段代码是SHA-224 / SHA-256算法的实现。它包含了版权声明和许可证信息。其中定义了一些宏和函数。
#define SSG2_0(x)      (ROTR(x, 7) ^ ROTR(x, 18) ^ SPH_T32((x) >> 3))
// 定义宏SSG2_0，对输入x进行位旋转和位移操作

#define SSG2_1(x)      (ROTR(x, 17) ^ ROTR(x, 19) ^ SPH_T32((x) >> 10))
// 定义宏SSG2_1，对输入x进行位旋转和位移操作

static const sph_u32 H224[8] = {
    SPH_C32(0xC1059ED8), SPH_C32(0x367CD507), SPH_C32(0x3070DD17),
    SPH_C32(0xF70E5939), SPH_C32(0xFFC00B31), SPH_C32(0x68581511),
    SPH_C32(0x64F98FA7), SPH_C32(0xBEFA4FA4)
};
// 定义SHA-224的初始哈希值

static const sph_u32 H256[8] = {
    SPH_C32(0x6A09E667), SPH_C32(0xBB67AE85), SPH_C32(0x3C6EF372),
    SPH_C32(0xA54FF53A), SPH_C32(0x510E527F), SPH_C32(0x9B05688C),
    SPH_C32(0x1F83D9AB), SPH_C32(0x5BE0CD19)
};
// 定义SHA-256的初始哈希值

/*
 * The SHA2_ROUND_BODY defines the body for a SHA-224 / SHA-256
 * compression function implementation. The "in" parameter should
 * evaluate, when applied to a numerical input parameter from 0 to 15,
 * to an expression which yields the corresponding input block. The "r"
 * parameter should evaluate to an array or pointer expression
 * designating the array of 8 words which contains the input and output
 * of the compression function.
 */
// 定义SHA-224 / SHA-256压缩函数的实现体

/*
static const sph_u32 K[64] = {
    SPH_C32(0x428A2F98), SPH_C32(0x71374491),
    SPH_C32(0xB5C0FBCF), SPH_C32(0xE9B5DBA5),
    SPH_C32(0x3956C25B), SPH_C32(0x59F111F1),
    SPH_C32(0x923F82A4), SPH_C32(0xAB1C5ED5),
    SPH_C32(0xD807AA98), SPH_C32(0x12835B01),
    SPH_C32(0x243185BE), SPH_C32(0x550C7DC3),
    SPH_C32(0x72BE5D74), SPH_C32(0x80DEB1FE),
    SPH_C32(0x9BDC06A7), SPH_C32(0xC19BF174),
    SPH_C32(0xE49B69C1), SPH_C32(0xEFBE4786),
    SPH_C32(0x0FC19DC6), SPH_C32(0x240CA1CC),
    SPH_C32(0x2DE92C6F), SPH_C32(0x4A7484AA),
    SPH_C32(0x5CB0A9DC), SPH_C32(0x76F988DA),
    SPH_C32(0x983E5152), SPH_C32(0xA831C66D),
    SPH_C32(0xB00327C8), SPH_C32(0xBF597FC7),
    SPH_C32(0xC6E00BF3), SPH_C32(0xD5A79147),
    SPH_C32(0x06CA6351), SPH_C32(0x14292967),
    SPH_C32(0x27B70A85), SPH_C32(0x2E1B2138),
    SPH_C32(0x4D2C6DFC), SPH_C32(0x53380D13),
    SPH_C32(0x650A7354), SPH_C32(0x766A0ABB),
    SPH_C32(0x81C2C92E), SPH_C32(0x92722C85),

// 定义SHA-224 / SHA-256压缩函数中使用的常量K
    # 定义一系列常量，使用 SPH_C32 函数将十六进制数转换为 32 位整数
    SPH_C32(0xA2BFE8A1), SPH_C32(0xA81A664B),
    SPH_C32(0xC24B8B70), SPH_C32(0xC76C51A3),
    SPH_C32(0xD192E819), SPH_C32(0xD6990624),
    SPH_C32(0xF40E3585), SPH_C32(0x106AA070),
    SPH_C32(0x19A4C116), SPH_C32(0x1E376C08),
    SPH_C32(0x2748774C), SPH_C32(0x34B0BCB5),
    SPH_C32(0x391C0CB3), SPH_C32(0x4ED8AA4A),
    SPH_C32(0x5B9CCA4F), SPH_C32(0x682E6FF3),
    SPH_C32(0x748F82EE), SPH_C32(0x78A5636F),
    SPH_C32(0x84C87814), SPH_C32(0x8CC70208),
    SPH_C32(0x90BEFFFA), SPH_C32(0xA4506CEB),
    SPH_C32(0xBEF9A3F7), SPH_C32(0xC67178F2)
// 如果定义了 SPH_SMALL_FOOTPRINT_SHA2，则使用 SHA2_MEXP1 宏定义
#define SHA2_MEXP1(in, pc)   do { \
        W[pc] = in(pc); \
    } while (0)

// 如果定义了 SPH_SMALL_FOOTPRINT_SHA2，则使用 SHA2_MEXP2 宏定义
#define SHA2_MEXP2(in, pc)   do { \
        W[(pc) & 0x0F] = SPH_T32(SSG2_1(W[((pc) - 2) & 0x0F]) \
            + W[((pc) - 7) & 0x0F] \
            + SSG2_0(W[((pc) - 15) & 0x0F]) + W[(pc) & 0x0F]); \
    } while (0)

// 如果定义了 SPH_SMALL_FOOTPRINT_SHA2，则使用 SHA2_STEPn 宏定义
#define SHA2_STEPn(n, a, b, c, d, e, f, g, h, in, pc)   do { \
        sph_u32 t1, t2; \
        SHA2_MEXP ## n(in, pc); \
        t1 = SPH_T32(h + BSG2_1(e) + CH(e, f, g) \
            + K[pcount + (pc)] + W[(pc) & 0x0F]); \
        t2 = SPH_T32(BSG2_0(a) + MAJ(a, b, c)); \
      Y_xor_Z = X_xor_Y; \
        d = SPH_T32(d + t1); \
        h = SPH_T32(t1 + t2); \
    } while (0)

// 如果定义了 SPH_SMALL_FOOTPRINT_SHA2，则使用 SHA2_STEP1 宏定义
#define SHA2_STEP1(a, b, c, d, e, f, g, h, in, pc) \
    SHA2_STEPn(1, a, b, c, d, e, f, g, h, in, pc)
// 如果定义了 SPH_SMALL_FOOTPRINT_SHA2，则使用 SHA2_STEP2 宏定义
#define SHA2_STEP2(a, b, c, d, e, f, g, h, in, pc) \
    SHA2_STEPn(2, a, b, c, d, e, f, g, h, in, pc)

// 执行 SHA-224 / SHA-256 的一轮运算
static void
sha2_round(const unsigned char *data, sph_u32 r[8])
{
#define SHA2_IN(x)   sph_dec32be_aligned(data + (4 * (x)))
    SHA2_ROUND_BODY(SHA2_IN, r);
#undef SHA2_IN
}

// 执行 SHA-256 的转换，数据是小端存储
void sph_sha256_transform_le( uint32_t *state_out, const uint32_t *data,
                              const uint32_t *state_in )
{
memcpy( state_out, state_in, 32 );
#define SHA2_IN(x)   (data[x])
   SHA2_ROUND_BODY( SHA2_IN, state_out );
#undef SHA2_IN
}

// 执行 SHA-256 的转换，数据是大端存储
void sph_sha256_transform_be( uint32_t *state_out, const uint32_t *data,
                              const uint32_t *state_in )
{
memcpy( state_out, state_in, 32 );
#define SHA2_IN(x)   sph_dec32be_aligned( data+(x) )
   SHA2_ROUND_BODY( SHA2_IN, state_out );
#undef SHA2_IN

}

// 初始化 SHA-224 算法
void
sph_sha224_init(void *cc)
{
    sph_sha224_context *sc;

    sc = cc;
    memcpy(sc->val, H224, sizeof H224);
#if SPH_64
    sc->count = 0;
#else
    sc->count_high = sc->count_low = 0;
#endif
}

/* see sph_sha2.h */
void
sph_sha256_init(void *cc)
{
    sph_sha256_context *sc;

    sc = cc;
    // 将预定义的 H256 数组的值复制到 sc->val 数组中
    memcpy(sc->val, H256, sizeof H256);
#if SPH_64
    sc->count = 0;
#else
    // 如果不是 64 位系统，则将 sc->count_high 和 sc->count_low 都设置为 0
    sc->count_high = sc->count_low = 0;
#endif
}

#define RFUN   sha2_round
#define HASH   sha224
#define BE32   1
#include "md_helper.c"

/* see sph_sha2.h */
void
sph_sha224_close(void *cc, void *dst)
{
    // 调用 sha224_close 函数，传入 cc、dst 和 7 作为参数
    sha224_close(cc, dst, 7);
//    sph_sha224_init(cc);
}

/* see sph_sha2.h */
void
sph_sha224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 sha224_addbits_and_close 函数，传入 cc、ub、n、dst 和 7 作为参数
    sha224_addbits_and_close(cc, ub, n, dst, 7);
//    sph_sha224_init(cc);
}

/* see sph_sha2.h */
void
sph_sha256_close(void *cc, void *dst)
{
    // 调用 sha224_close 函数，传入 cc、dst 和 8 作为参数
    sha224_close(cc, dst, 8);
//    sph_sha256_init(cc);
}

/* see sph_sha2.h */
void
sph_sha256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst)
{
    // 调用 sha224_addbits_and_close 函数，传入 cc、ub、n、dst 和 8 作为参数
    sha224_addbits_and_close(cc, ub, n, dst, 8);
//    sph_sha256_init(cc);
}

void sph_sha256_full( void *dst, const void *data, size_t len )
{
   sph_sha256_context cc;
   // 初始化 SHA-256 上下文
   sph_sha256_init( &cc );
   // 对数据进行 SHA-256 哈希计算
   sph_sha256( &cc, data, len );
   // 关闭 SHA-256 上下文，将结果写入 dst
   sph_sha256_close( &cc, dst );
}

void sha256d(void* hash, const void* data, int len)
{
    // 对数据进行两次 SHA-256 哈希计算
    sph_sha256_full(hash, data, len);
    sph_sha256_full(hash, hash, 32);
}

/* see sph_sha2.h */
//void
//sph_sha224_comp(const sph_u32 msg[16], sph_u32 val[8])
//{
//#define SHA2_IN(x)   msg[x]
//    SHA2_ROUND_BODY(SHA2_IN, val);
//#undef SHA2_IN
//}
```