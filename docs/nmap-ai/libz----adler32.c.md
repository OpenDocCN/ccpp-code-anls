# `nmap\libz\adler32.c`

```cpp
/* adler32.c -- compute the Adler-32 checksum of a data stream
 * 计算数据流的Adler-32校验和
 * 版权所有（C）1995-2011, 2016 Mark Adler
 * 分发和使用条件，请参阅zlib.h中的版权声明
 */

/* @(#) $Id$ */

#include "zutil.h"

local uLong adler32_combine_ OF((uLong adler1, uLong adler2, z_off64_t len2));

#define BASE 65521U     /* largest prime smaller than 65536 */
#define NMAX 5552
/* NMAX is the largest n such that 255n(n+1)/2 + (n+1)(BASE-1) <= 2^32-1 */
/* NMAX是最大的n，使得255n(n+1)/2 + (n+1)(BASE-1) <= 2^32-1 */

#define DO1(buf,i)  {adler += (buf)[i]; sum2 += adler;}
#define DO2(buf,i)  DO1(buf,i); DO1(buf,i+1);
#define DO4(buf,i)  DO2(buf,i); DO2(buf,i+2);
#define DO8(buf,i)  DO4(buf,i); DO4(buf,i+4);
#define DO16(buf)   DO8(buf,0); DO8(buf,8);
/* 定义了一系列的宏，用于计算Adler-32校验和 */

/* use NO_DIVIDE if your processor does not do division in hardware --
   try it both ways to see which is faster */
#ifdef NO_DIVIDE
/* note that this assumes BASE is 65521, where 65536 % 65521 == 15
   (thank you to John Reiser for pointing this out) */
#  define CHOP(a) \
    do { \
        unsigned long tmp = a >> 16; \
        a &= 0xffffUL; \
        a += (tmp << 4) - tmp; \
    } while (0)
#  define MOD28(a) \
    do { \
        CHOP(a); \
        if (a >= BASE) a -= BASE; \
    } while (0)
#  define MOD(a) \
    do { \
        CHOP(a); \
        MOD28(a); \
    } while (0)
#  define MOD63(a) \
    do { /* this assumes a is not negative */ \
        z_off64_t tmp = a >> 32; \
        a &= 0xffffffffL; \
        a += (tmp << 8) - (tmp << 5) + tmp; \
        tmp = a >> 16; \
        a &= 0xffffL; \
        a += (tmp << 4) - tmp; \
        tmp = a >> 16; \
        a &= 0xffffL; \
        a += (tmp << 4) - tmp; \
        if (a >= BASE) a -= BASE; \
    } while (0)
#else
#  define MOD(a) a %= BASE
#  define MOD28(a) a %= BASE
#  define MOD63(a) a %= BASE
#endif
/* 定义了一系列的宏，用于在处理器不支持硬件除法时计算Adler-32校验和 */

/* ========================================================================= */
uLong ZEXPORT adler32_z(adler, buf, len)
    uLong adler;
    const Bytef *buf;
    z_size_t len;
{
    unsigned long sum2;
    /* 定义了一个无符号长整型变量sum2 */
    # 定义一个无符号整数变量n
    unsigned n;

    # 将Adler-32拆分为组成部分的和
    sum2 = (adler >> 16) & 0xffff;
    adler &= 0xffff;

    # 如果长度为1，则逐字节处理以保持速度
    if (len == 1) {
        adler += buf[0];
        if (adler >= BASE)
            adler -= BASE;
        sum2 += adler;
        if (sum2 >= BASE)
            sum2 -= BASE;
        return adler | (sum2 << 16);
    }

    # 初始Adler-32值（延迟检查len == 1以提高速度）
    if (buf == Z_NULL)
        return 1L;

    # 如果提供了较短的长度，则保持相对较快的速度
    if (len < 16) {
        while (len--) {
            adler += *buf++;
            sum2 += adler;
        }
        if (adler >= BASE)
            adler -= BASE;
        MOD28(sum2);            # 只添加了这么多个BASE
        return adler | (sum2 << 16);
    }

    # 处理长度为NMAX的块--仅需要一次模运算
    while (len >= NMAX) {
        len -= NMAX;
        n = NMAX / 16;          # NMAX可被16整除
        do {
            DO16(buf);          # 展开了16个和
            buf += 16;
        } while (--n);
        MOD(adler);
        MOD(sum2);
    }

    # 处理剩余的字节（小于NMAX，仍然只需一次模运算）
    if (len) {                  # 如果没有剩余，则避免模运算
        while (len >= 16) {
            len -= 16;
            DO16(buf);
            buf += 16;
        }
        while (len--) {
            adler += *buf++;
            sum2 += adler;
        }
        MOD(adler);
        MOD(sum2);
    }

    # 返回重新组合的和
    return adler | (sum2 << 16);
/* ========================================================================= */
// 计算给定数据的 Adler-32 校验和
uLong ZEXPORT adler32(adler, buf, len)
    uLong adler;  // 初始的 Adler-32 值
    const Bytef *buf;  // 数据缓冲区
    uInt len;  // 数据长度
{
    return adler32_z(adler, buf, len);  // 调用 adler32_z 函数计算 Adler-32 校验和
}

/* ========================================================================= */
// 合并两个 Adler-32 校验和
local uLong adler32_combine_(adler1, adler2, len2)
    uLong adler1;  // 第一个 Adler-32 值
    uLong adler2;  // 第二个 Adler-32 值
    z_off64_t len2;  // 第二个数据的长度
{
    unsigned long sum1;  // 第一个 Adler-32 校验和的和
    unsigned long sum2;  // 第二个 Adler-32 校验和的和
    unsigned rem;  // 余数

    /* for negative len, return invalid adler32 as a clue for debugging */
    if (len2 < 0)
        return 0xffffffffUL;  // 如果长度为负数，返回无效的 Adler-32 校验和作为调试线索

    /* the derivation of this formula is left as an exercise for the reader */
    MOD63(len2);  // 对长度进行 MOD63 运算
    rem = (unsigned)len2;  // 获取余数
    sum1 = adler1 & 0xffff;  // 获取第一个 Adler-32 校验和的低 16 位
    sum2 = rem * sum1;  // 计算第二个 Adler-32 校验和的和
    MOD(sum2);  // 对 sum2 进行 MOD 运算
    sum1 += (adler2 & 0xffff) + BASE - 1;  // 计算第一个 Adler-32 校验和的和
    sum2 += ((adler1 >> 16) & 0xffff) + ((adler2 >> 16) & 0xffff) + BASE - rem;  // 计算第二个 Adler-32 校验和的和
    if (sum1 >= BASE) sum1 -= BASE;  // 如果 sum1 大于等于 BASE，则减去 BASE
    if (sum1 >= BASE) sum1 -= BASE;  // 如果 sum1 大于等于 BASE，则减去 BASE
    if (sum2 >= ((unsigned long)BASE << 1)) sum2 -= ((unsigned long)BASE << 1);  // 如果 sum2 大于等于 BASE 的两倍，则减去 BASE 的两倍
    if (sum2 >= BASE) sum2 -= BASE;  // 如果 sum2 大于等于 BASE，则减去 BASE
    return sum1 | (sum2 << 16);  // 返回合并后的 Adler-32 校验和
}

/* ========================================================================= */
// 合并两个 Adler-32 校验和
uLong ZEXPORT adler32_combine(adler1, adler2, len2)
    uLong adler1;  // 第一个 Adler-32 值
    uLong adler2;  // 第二个 Adler-32 值
    z_off_t len2;  // 第二个数据的长度
{
    return adler32_combine_(adler1, adler2, len2);  // 调用 adler32_combine_ 函数进行 Adler-32 校验和合并
}

// 合并两个 Adler-32 校验和（64 位）
uLong ZEXPORT adler32_combine64(adler1, adler2, len2)
    uLong adler1;  // 第一个 Adler-32 值
    uLong adler2;  // 第二个 Adler-32 值
    z_off64_t len2;  // 第二个数据的长度
{
    return adler32_combine_(adler1, adler2, len2);  // 调用 adler32_combine_ 函数进行 Adler-32 校验和合并
}
```