# `xmrig\src\crypto\ghostrider\md_helper.c`

```cpp
/* $Id: md_helper.c 216 2010-06-08 09:46:57Z tp $ */
// 定义了一个版本信息的标识符

#ifdef _MSC_VER
#pragma warning (disable: 4146)
#endif
// 如果是在 MSC 编译器下，则禁用特定警告

#undef SPH_XCAT
#define SPH_XCAT(a, b)     SPH_XCAT_(a, b)
// 取消之前定义的宏 SPH_XCAT，并重新定义为将两个参数连接起来的宏

#undef SPH_XCAT_
#define SPH_XCAT_(a, b)    a ## b
// 取消之前定义的宏 SPH_XCAT_，重新定义为将两个参数连接起来的宏

#undef SPH_BLEN
#undef SPH_WLEN
#if defined BE64 || defined LE64
#define SPH_BLEN    128U
#define SPH_WLEN      8U
#else
#define SPH_BLEN     64U
#define SPH_WLEN      4U
#endif
// 取消之前定义的宏 SPH_BLEN 和 SPH_WLEN，根据条件定义宏的值

#ifdef BLEN
#undef SPH_BLEN
#define SPH_BLEN    BLEN
#endif
// 如果定义了 BLEN 宏，则重新定义 SPH_BLEN 宏的值为 BLEN

#undef SPH_MAXPAD
#if defined PLW1
#define SPH_MAXPAD   (SPH_BLEN - SPH_WLEN)
#elif defined PLW4
#define SPH_MAXPAD   (SPH_BLEN - (SPH_WLEN << 2))
#else
#define SPH_MAXPAD   (SPH_BLEN - (SPH_WLEN << 1))
#endif
// 取消之前定义的宏 SPH_MAXPAD，根据条件重新定义宏的值

#undef SPH_VAL
#undef SPH_NO_OUTPUT
#ifdef SVAL
#define SPH_VAL         SVAL
#define SPH_NO_OUTPUT   1
#else
#define SPH_VAL   sc->val
#endif
// 取消之前定义的宏 SPH_VAL 和 SPH_NO_OUTPUT，根据条件重新定义宏的值

#ifndef CLOSE_ONLY
// 如果没有定义 CLOSE_ONLY 宏，则执行以下代码

#ifdef SPH_UPTR
static void
SPH_XCAT(HASH, _short)(void *cc, const void *data, size_t len)
#else
void
SPH_XCAT(sph_, HASH)(void *cc, const void *data, size_t len)
#endif
// 如果定义了 SPH_UPTR 宏，则定义一个静态函数，否则定义一个全局函数

{
    SPH_XCAT(sph_, SPH_XCAT(HASH, _context)) *sc;
    size_t current;
    // 定义变量 sc 和 current

    sc = cc;
#if SPH_64
    current = (unsigned)sc->count & (SPH_BLEN - 1U);
#else
    current = (unsigned)sc->count_low & (SPH_BLEN - 1U);
#endif
    // 根据条件给 current 赋值

    while (len > 0) {
        size_t clen;
#if !SPH_64
        sph_u32 clow, clow2;
#endif
        // 定义变量 clen，以及根据条件定义变量 clow 和 clow2

        clen = SPH_BLEN - current;
        if (clen > len)
            clen = len;
        // 计算 clen 的值

        memcpy(sc->buf + current, data, clen);
        data = (const unsigned char *)data + clen;
        current += clen;
        len -= clen;
        // 将数据拷贝到缓冲区中，并更新相关变量

        if (current == SPH_BLEN) {
            RFUN(sc->buf, SPH_VAL);
            current = 0;
        }
        // 如果缓冲区已满，则调用 RFUN 函数，并重置 current

#if SPH_64
        sc->count += clen;
#else
        clow = sc->count_low;
        clow2 = SPH_T32(clow + clen);
        sc->count_low = clow2;
        if (clow2 < clow)
            sc->count_high ++;
#endif
        // 根据条件更新 count 和 count_low 变量
    }
}

#ifdef SPH_UPTR
void
SPH_XCAT(sph_, HASH)(void *cc, const void *data, size_t len)
{
    SPH_XCAT(sph_, SPH_XCAT(HASH, _context)) *sc;
    // 定义变量 sc
    # 定义一个无符号整数变量current
    unsigned current;
    # 定义一个表示长度的变量orig_len，使用size_t类型
    size_t orig_len;
#if !SPH_64
    // 如果不是64位系统，定义两个32位无符号整数变量
    sph_u32 clow, clow2;
#endif

    // 如果数据长度小于两个块大小，调用短消息处理函数并返回
    if (len < (2 * SPH_BLEN)) {
        SPH_XCAT(HASH, _short)(cc, data, len);
        return;
    }
    // 将cc赋值给sc
    sc = cc;
#if SPH_64
    // 如果是64位系统，取sc->count的低32位
    current = (unsigned)sc->count & (SPH_BLEN - 1U);
#else
    // 如果不是64位系统，取sc->count_low的低32位
    current = (unsigned)sc->count_low & (SPH_BLEN - 1U);
#endif
    // 如果当前位置大于0
    if (current > 0) {
        unsigned t;

        // 计算需要填充的长度t
        t = SPH_BLEN - current;
        // 调用短消息处理函数
        SPH_XCAT(HASH, _short)(cc, data, t);
        // 更新data和len
        data = (const unsigned char *)data + t;
        len -= t;
    }
#if !SPH_UNALIGNED
    // 如果数据不是按字对齐
    if (((SPH_UPTR)data & (SPH_WLEN - 1U)) != 0) {
        // 调用短消息处理函数并返回
        SPH_XCAT(HASH, _short)(cc, data, len);
        return;
    }
#endif
    // 保存原始数据长度
    orig_len = len;
    // 循环处理数据
    while (len >= SPH_BLEN) {
        // 调用压缩函数处理数据
        RFUN(data, SPH_VAL);
        // 更新len和data
        len -= SPH_BLEN;
        data = (const unsigned char *)data + SPH_BLEN;
    }
    // 如果还有剩余数据，拷贝到sc->buf中
    if (len > 0)
        memcpy(sc->buf, data, len);
#if SPH_64
    // 如果是64位系统，更新sc->count
    sc->count += (sph_u64)orig_len;
#else
    // 如果不是64位系统，更新sc->count_low和sc->count_high
    clow = sc->count_low;
    clow2 = SPH_T32(clow + orig_len);
    sc->count_low = clow2;
    if (clow2 < clow)
        sc->count_high ++;
    /*
     * This code handles the improbable situation where "size_t" is
     * greater than 32 bits, and yet we do not have a 64-bit type.
     */
    // 处理size_t大于32位的情况
    orig_len >>= 12;
    orig_len >>= 10;
    orig_len >>= 10;
    sc->count_high += orig_len;
#endif
}
#endif

#endif

/*
 * Perform padding and produce result. The context is NOT reinitialized
 * by this function.
 */
static void
SPH_XCAT(HASH, _addbits_and_close)(void *cc,
    unsigned ub, unsigned n, void *dst, unsigned rnum)
{
    SPH_XCAT(sph_, SPH_XCAT(HASH, _context)) *sc;
    unsigned current, u;
#if !SPH_64
    sph_u32 low, high;
#endif

    // 将cc赋值给sc
    sc = cc;
#if SPH_64
    // 如果是64位系统，取sc->count的低32位
    current = (unsigned)sc->count & (SPH_BLEN - 1U);
#else
    // 如果不是64位系统，取sc->count_low的低32位
    current = (unsigned)sc->count_low & (SPH_BLEN - 1U);
#endif
#ifdef PW01
    // 如果定义了PW01，进行填充操作
    sc->buf[current ++] = (0x100 | (ub & 0xFF)) >> (8 - n);
#else
    {
        unsigned z;

        z = 0x80 >> n;
        sc->buf[current ++] = ((ub & -z) | z) & 0xFF;
    }
#endif
    # 如果当前填充的字节数大于最大填充字节数
    if (current > SPH_MAXPAD) {
        # 将缓冲区中从当前位置到末尾的字节全部置为0
        memset(sc->buf + current, 0, SPH_BLEN - current);
        # 对缓冲区中的数据进行哈希计算
        RFUN(sc->buf, SPH_VAL);
        # 将缓冲区中从开头到最大填充字节数的字节全部置为0
        memset(sc->buf, 0, SPH_MAXPAD);
    } 
    # 如果当前填充的字节数小于等于最大填充字节数
    else {
        # 将缓冲区中从当前位置到最大填充字节数的字节全部置为0
        memset(sc->buf + current, 0, SPH_MAXPAD - current);
    }
#if defined BE64
// 如果定义了 BE64，则执行以下代码块
#if defined PLW1
    // 如果定义了 PLW1，则执行以下代码块
    // 将 sc->count 左移 3 位，并加上 n，然后以大端序写入 sc->buf + SPH_MAXPAD
    sph_enc64be_aligned(sc->buf + SPH_MAXPAD,
        SPH_T64(sc->count << 3) + (sph_u64)n);
#elif defined PLW4
    // 如果定义了 PLW4，则执行以下代码块
    // 将 sc->buf + SPH_MAXPAD 后的 2 * SPH_WLEN 字节设置为 0
    memset(sc->buf + SPH_MAXPAD, 0, 2 * SPH_WLEN);
    // 将 sc->count 右移 61 位，并以大端序写入 sc->buf + SPH_MAXPAD + 2 * SPH_WLEN
    sph_enc64be_aligned(sc->buf + SPH_MAXPAD + 2 * SPH_WLEN,
        sc->count >> 61);
    // 将 sc->count 左移 3 位，并加上 n，然后以大端序写入 sc->buf + SPH_MAXPAD + 3 * SPH_WLEN
    sph_enc64be_aligned(sc->buf + SPH_MAXPAD + 3 * SPH_WLEN,
        SPH_T64(sc->count << 3) + (sph_u64)n);
#else
    // 如果未定义 PLW1 或 PLW4，则执行以下代码块
    // 将 sc->count 右移 61 位，并以大端序写入 sc->buf + SPH_MAXPAD
    sph_enc64be_aligned(sc->buf + SPH_MAXPAD, sc->count >> 61);
    // 将 sc->count 左移 3 位，并加上 n，然后以大端序写入 sc->buf + SPH_MAXPAD + SPH_WLEN
    sph_enc64be_aligned(sc->buf + SPH_MAXPAD + SPH_WLEN,
        SPH_T64(sc->count << 3) + (sph_u64)n);
#endif
#elif defined LE64
// 如果定义了 LE64，则执行以下代码块
#if defined PLW1
    // 如果定义了 PLW1，则执行以下代码块
    // 将 sc->count 左移 3 位，并加上 n，然后以小端序写入 sc->buf + SPH_MAXPAD
    sph_enc64le_aligned(sc->buf + SPH_MAXPAD,
        SPH_T64(sc->count << 3) + (sph_u64)n);
#elif defined PLW1
    // 如果定义了 PLW1，则执行以下代码块
    // 将 sc->count 左移 3 位，并加上 n，然后以小端序写入 sc->buf + SPH_MAXPAD
    sph_enc64le_aligned(sc->buf + SPH_MAXPAD,
        SPH_T64(sc->count << 3) + (sph_u64)n);
    // 将 sc->count 右移 61 位，并以小端序写入 sc->buf + SPH_MAXPAD + SPH_WLEN
    sph_enc64le_aligned(sc->buf + SPH_MAXPAD + SPH_WLEN, sc->count >> 61);
    // 将 sc->buf + SPH_MAXPAD + 2 * SPH_WLEN 后的 2 * SPH_WLEN 字节设置为 0
    memset(sc->buf + SPH_MAXPAD + 2 * SPH_WLEN, 0, 2 * SPH_WLEN);
#else
    // 如果未定义 PLW1，则执行以下代码块
    // 将 sc->count 左移 3 位，并加上 n，然后以小端序写入 sc->buf + SPH_MAXPAD
    sph_enc64le_aligned(sc->buf + SPH_MAXPAD,
        SPH_T64(sc->count << 3) + (sph_u64)n);
    // 将 sc->count 右移 61 位，并以小端序写入 sc->buf + SPH_MAXPAD + SPH_WLEN
    sph_enc64le_aligned(sc->buf + SPH_MAXPAD + SPH_WLEN, sc->count >> 61);
#endif
#else
// 如果未定义 BE64 或 LE64，则执行以下代码块
#if SPH_64
// 如果定义了 SPH_64，则执行以下代码块
#ifdef BE32
    // 如果定义了 BE32，则执行以下代码块
    // 将 sc->count 左移 3 位，并加上 n，然后以大端序写入 sc->buf + SPH_MAXPAD
    sph_enc64be_aligned(sc->buf + SPH_MAXPAD,
        SPH_T64(sc->count << 3) + (sph_u64)n);
#else
    // 如果未定义 BE32，则执行以下代码块
    // 将 sc->count 左移 3 位，并加上 n，然后以小端序写入 sc->buf + SPH_MAXPAD
    sph_enc64le_aligned(sc->buf + SPH_MAXPAD,
        SPH_T64(sc->count << 3) + (sph_u64)n);
#endif
#else
// 如果未定义 SPH_64，则执行以下代码块
    // 将 sc->count_low 赋值给 low
    low = sc->count_low;
    // 将 (sc->count_high 左移 3 位) 或 (low 右移 29 位) 的结果赋值给 high
    high = SPH_T32((sc->count_high << 3) | (low >> 29));
    // 将 low 左移 3 位，并加上 n，然后以小端序或大端序写入 sc->buf + SPH_MAXPAD 或 sc->buf + SPH_MAXPAD + SPH_WLEN
    low = SPH_T32(low << 3) + (sph_u32)n;
#ifdef BE32
    // 如果定义了 BE32，则执行以下代码块
    // 以大端序写入 sc->buf + SPH_MAXPAD 或 sc->buf + SPH_MAXPAD + SPH_WLEN
    sph_enc32be(sc->buf + SPH_MAXPAD, high);
    sph_enc32be(sc->buf + SPH_MAXPAD + SPH_WLEN, low);
#else
    // 如果未定义 BE32，则执行以下代码块
    // 以小端序写入 sc->buf + SPH_MAXPAD 或 sc->buf + SPH_MAXPAD + SPH_WLEN
    sph_enc32le(sc->buf + SPH_MAXPAD, low);
    sph_enc32le(sc->buf + SPH_MAXPAD + SPH_WLEN, high);
#endif
#endif
#endif
    // 调用 RFUN 函数，传入 sc->buf 和 SPH_VAL 作为参数
    RFUN(sc->buf, SPH_VAL);
#ifdef SPH_NO_OUTPUT
    // 如果定义了 SPH_NO_OUTPUT，则执行以下代码块
    // 忽略 dst、rnum、u 的值
    (void)dst;
    (void)rnum;
    (void)u;
#else
    // 如果未定义 SPH_NO_OUTPUT，则执行以下代码块
    // 遍历 sc->val 数组，将其值以大端序写入 dst
    for (u = 0; u < rnum; u ++) {
#if defined BE64
        // 如果定义了 BE64，则执行以下代码块
        // 将 sc->val[u] 以大端序写入 dst + 8 * u
        sph_enc64be((unsigned char *)dst + 8 * u, sc->val[u]);
#elif defined LE64
        # 如果定义了 LE64，则使用小端序编码将 sc->val[u] 的值写入目标内存
        sph_enc64le((unsigned char *)dst + 8 * u, sc->val[u]);
#elif defined BE32
        # 如果定义了 BE32，则使用大端序编码将 sc->val[u] 的值写入目标内存
        sph_enc32be((unsigned char *)dst + 4 * u, sc->val[u]);
#else
        # 否则使用小端序编码将 sc->val[u] 的值写入目标内存
        sph_enc32le((unsigned char *)dst + 4 * u, sc->val[u]);
#endif
    }
#endif
}

static void
SPH_XCAT(HASH, _close)(void *cc, void *dst, unsigned rnum)
{
    # 调用 SPH_XCAT(HASH, _addbits_and_close) 函数，传入参数 cc, 0, 0, dst, rnum
    SPH_XCAT(HASH, _addbits_and_close)(cc, 0, 0, dst, rnum);
}
```