# `xmrig\src\3rdparty\argon2\lib\encoding.c`

```
#include <stdio.h>  // 包含标准输入输出库
#include <stdlib.h>  // 包含标准库
#include <string.h>  // 包含字符串处理库
#include <limits.h>  // 包含整数限制库
#include "encoding.h"  // 包含自定义的编码库
#include "core.h"  // 包含自定义的核心功能库

/*
 * Example code for a decoder and encoder of "hash strings", with Argon2
 * parameters.
 *
 * This code comprises three sections:
 *
 *   -- The first section contains generic Base64 encoding and decoding
 *   functions. It is conceptually applicable to any hash function
 *   implementation that uses Base64 to encode and decode parameters,
 *   salts and outputs. It could be made into a library, provided that
 *   the relevant functions are made public (non-static) and be given
 *   reasonable names to avoid collisions with other functions.
 *
 *   -- The second section is specific to Argon2. It encodes and decodes
 *   the parameters, salts and outputs. It does not compute the hash
 *   itself.
 *
 * The code was originally written by Thomas Pornin <pornin@bolet.org>,
 * to whom comments and remarks may be sent. It is released under what
 * should amount to Public Domain or its closest equivalent; the
 * following mantra is supposed to incarnate that fact with all the
 * proper legal rituals:
 *
 * ---------------------------------------------------------------------
 * This file is provided under the terms of Creative Commons CC0 1.0
 * Public Domain Dedication. To the extent possible under law, the
 * author (Thomas Pornin) has waived all copyright and related or
 * neighboring rights to this file. This work is published from: Canada.
 * ---------------------------------------------------------------------
 *
 * Copyright (c) 2015 Thomas Pornin
 */

/* ==================================================================== */
/*
 * Common code; could be shared between different hash functions.
 *
 * Note: the Base64 functions below assume that uppercase letters (resp.
 * lowercase letters) have consecutive numerical codes, that fit on 8
 * bits. All modern systems use ASCII-compatible charsets, where these
 * properties are true. If you are stuck with a dinosaur of a system
 * that still defaults to EBCDIC then you already have much bigger
 * interoperability issues to deal with.
 */
 
/*
 * Some macros for constant-time comparisons. These work over values in
 * the 0..255 range. Returned value is 0x00 on "false", 0xFF on "true".
 */
#define EQ(x, y) ((((0U - ((unsigned)(x) ^ (unsigned)(y))) >> 8) & 0xFF) ^ 0xFF)  // 宏定义，用于比较两个值是否相等
#define GT(x, y) ((((unsigned)(y) - (unsigned)(x)) >> 8) & 0xFF)  // 宏定义，用于比较两个值的大小关系
#define GE(x, y) (GT(y, x) ^ 0xFF)  // 宏定义，用于比较两个值的大小关系
#define LT(x, y) GT(y, x)  // 宏定义，用于比较两个值的大小关系
#define LE(x, y) GE(y, x)  // 宏定义，用于比较两个值的大小关系

/*
 * Convert value x (0..63) to corresponding Base64 character.
 */
static int b64_byte_to_char(unsigned x) {
    return (LT(x, 26) & (x + 'A')) |  // 将值转换为对应的 Base64 字符
           (GE(x, 26) & LT(x, 52) & (x + ('a' - 26)) |  // 将值转换为对应的 Base64 字符
           (GE(x, 52) & LT(x, 62) & (x + ('0' - 52)) |  // 将值转换为对应的 Base64 字符
           (EQ(x, 62) & '+') |  // 将值转换为对应的 Base64 字符
           (EQ(x, 63) & '/'));  // 将值转换为对应的 Base64 字符
}

/*
 * Convert character c to the corresponding 6-bit value. If character c
 * is not a Base64 character, then 0xFF (255) is returned.
 */
static unsigned b64_char_to_byte(int c) {
    unsigned x;

    x = (GE(c, 'A') & LE(c, 'Z') & (c - 'A')) |  // 将字符转换为对应的 6 位值
        (GE(c, 'a') & LE(c, 'z') & (c - ('a' - 26))) |  // 将字符转换为对应的 6 位值
        (GE(c, '0') & LE(c, '9') & (c - ('0' - 52))) |  // 将字符转换为对应的 6 位值
        (EQ(c, '+') & 62) |  // 将字符转换为对应的 6 位值
        (EQ(c, '/') & 63);  // 将字符转换为对应的 6 位值
    return x | (EQ(x, 0) & (EQ(c, 'A') ^ 0xFF));  // 如果字符不是 Base64 字符，则返回 0xFF
}
/*
 * 将一些字节转换为 Base64。'dst_len' 是输出缓冲区 'dst' 的长度（以字符计算）；
 * 如果该缓冲区不足以接收结果（包括结尾的 0），则返回 (size_t)-1。
 * 否则，将以零结尾的 Base64 字符串写入缓冲区，并返回输出长度（不包括结尾的零）。
 */
static size_t to_base64(char *dst, size_t dst_len, const void *src,
                        size_t src_len) {
    size_t olen;
    const unsigned char *buf;
    unsigned acc, acc_len;

    olen = (src_len / 3) << 2;
    switch (src_len % 3) {
    case 2:
        olen++;
    /* fall through */
    case 1:
        olen += 2;
        break;
    }
    if (dst_len <= olen) {
        return (size_t)-1;
    }
    acc = 0;
    acc_len = 0;
    buf = (const unsigned char *)src;
    while (src_len-- > 0) {
        acc = (acc << 8) + (*buf++);
        acc_len += 8;
        while (acc_len >= 6) {
            acc_len -= 6;
            *dst++ = (char)b64_byte_to_char((acc >> acc_len) & 0x3F);
        }
    }
    if (acc_len > 0) {
        *dst++ = (char)b64_byte_to_char((acc << (6 - acc_len)) & 0x3F);
    }
    *dst++ = 0;
    return olen;
}

/*
 * 将 Base64 字符解码为字节。'*dst_len' 的值必须最初包含输出缓冲区 '*dst' 的长度；
 * 解码结束时，解码后的字节数会被写回 '*dst_len'。
 *
 * 当遇到非 Base64 字符或超出输出缓冲区容量时，解码停止。
 * 如果发生错误（输出缓冲区太小，无效的最后字符导致未处理的缓冲位），则返回 NULL；
 * 否则，返回值指向源流中第一个非 Base64 字符，可能是结尾的零。
 */
static const char *from_base64(void *dst, size_t *dst_len, const char *src) {
    size_t len;
    unsigned char *buf;
    unsigned acc, acc_len;
    # 将目标地址强制转换为无符号字符指针
    buf = (unsigned char *)dst;
    # 初始化长度为0
    len = 0;
    # 初始化累加器为0
    acc = 0;
    # 初始化累加器长度为0
    acc_len = 0;
    # 无限循环
    for (;;) {
        # 定义变量d，将当前源地址对应的base64字符转换为字节
        d = b64_char_to_byte(*src);
        # 如果转换结果为0xFF，跳出循环
        if (d == 0xFF) {
            break;
        }
        # 源地址指针后移
        src++;
        # 将转换结果加入累加器，并将累加器长度增加6
        acc = (acc << 6) + d;
        acc_len += 6;
        # 如果累加器长度大于等于8
        if (acc_len >= 8) {
            # 累加器长度减去8
            acc_len -= 8;
            # 如果长度超过目标长度，返回空指针
            if ((len++) >= *dst_len) {
                return NULL;
            }
            # 将累加器右移后的值存入目标地址，并指针后移
            *buf++ = (acc >> acc_len) & 0xFF;
        }
    }

    '''
     * 如果输入长度模4余1（无效情况），则会剩下6个未处理的位；
     * 否则，只有0、2或4位被缓冲。缓冲的位也必须全部为零。
     '''
    # 如果累加器长度大于4，或者累加器和（2的累加器长度次方 - 1）的与结果不为0
    if (acc_len > 4 || (acc & (((unsigned)1 << acc_len) - 1)) != 0) {
        # 返回空指针
        return NULL;
    }
    # 将目标长度设置为len
    *dst_len = len;
    # 返回源地址
    return src;
# 从字符串 'str' 中解码十进制整数；将值写入 '*v' 中。
# 返回值是指向字符串中下一个非十进制字符的指针。如果根本没有数字，或者值的编码不是最小的（额外的前导零），或者值不适合 'unsigned long'，则返回 NULL。
static const char *decode_decimal(const char *str, unsigned long *v) {
    const char *orig;
    unsigned long acc;

    acc = 0;
    for (orig = str;; str++) {
        int c;

        c = *str;
        if (c < '0' || c > '9') {
            break;
        }
        c -= '0';
        if (acc > (ULONG_MAX / 10)) {
            return NULL;
        }
        acc *= 10;
        if ((unsigned long)c > (ULONG_MAX - acc)) {
            return NULL;
        }
        acc += (unsigned long)c;
    }
    if (str == orig || (*orig == '0' && str != (orig + 1))) {
        return NULL;
    }
    *v = acc;
    return str;
}

/* ==================================================================== */
/*
 * 专用于 Argon2 的代码。
 *
 * 下面的代码应用以下格式：
 *
 *  $argon2<T>[$v=<num>]$m=<num>,t=<num>,p=<num>$<bin>$<bin>
 *
 * 其中 <T> 可以是 'd'、'id' 或 'i'，<num> 是十进制整数（正数，适合于 'unsigned long'），<bin> 是 Base64 编码的数据（没有 '=' 填充字符，没有换行或空格）。
 *
 * 最后两个二进制块（以Base64编码）依次是盐和输出。两者都是必需的。二进制盐长度和输出长度必须在 argon2.h 中定义的允许范围内。
 *
 * 当传入 decode_string 时，ctx 结构必须包含足够大的缓冲区来容纳盐和密码。
 */
int decode_string(argon2_context *ctx, const char *str, argon2_type type) {

/* 检查前缀 */
#define CC(prefix)                                                             \
    # 使用 do-while 循环来执行一系列操作
    do {                                                                       \
        # 计算前缀字符串的长度
        size_t cc_len = strlen(prefix);                                        \
        # 检查字符串是否以指定前缀开头，如果不是则返回解码失败
        if (strncmp(str, prefix, cc_len) != 0) {                               \
            return ARGON2_DECODING_FAIL;                                       \
        }                                                                      \
        # 将字符串指针移动到前缀字符串之后
        str += cc_len;                                                         \
    } while ((void)0, 0)
/* optional prefix checking with supplied code */
#define CC_opt(prefix, code)                                                   \  // 定义一个宏，用于检查前缀并执行相应的代码

    do {                                                                       \  // 开始一个 do-while 循环
        size_t cc_len = strlen(prefix);                                        \  // 计算前缀的长度
        if (strncmp(str, prefix, cc_len) == 0) {                               \  // 如果字符串以指定前缀开头
            str += cc_len;                                                     \  // 将字符串指针移动到前缀之后
            { code; }                                                          \  // 执行指定的代码
        }                                                                      \  // 结束 if 语句
    } while ((void)0, 0)                                                       \  // 结束 do-while 循环

/* Decoding prefix into uint32_t decimal */
#define DECIMAL_U32(x)                                                         \  // 定义一个宏，用于将前缀解码为 uint32_t 十进制数

    do {                                                                       \  // 开始一个 do-while 循环
        unsigned long dec_x;                                                   \  // 定义一个无符号长整型变量
        str = decode_decimal(str, &dec_x);                                     \  // 调用 decode_decimal 函数解码前缀
        if (str == NULL || dec_x > UINT32_MAX) {                               \  // 如果解码失败或者解码结果超出范围
            return ARGON2_DECODING_FAIL;                                       \  // 返回解码失败的错误码
        }                                                                      \  // 结束 if 语句
        (x) = (uint32_t)dec_x;                                                           \  // 将解码结果赋值给 x
    } while ((void)0, 0)                                                       \  // 结束 do-while 循环

/* Decoding base64 into a binary buffer */
#define BIN(buf, max_len, len)                                                 \  // 定义一个宏，用于将 base64 解码为二进制缓冲区
    # 使用 do-while 循环来执行一系列操作，直到满足条件为止
    do {                                                                       \
        # 定义变量 bin_len，并初始化为 max_len
        size_t bin_len = (max_len);                                            \
        # 将字符串 str 进行 base64 解码，得到二进制数据，并更新 bin_len
        str = from_base64(buf, &bin_len, str);                                 \
        # 如果解码失败或者二进制数据长度超过 UINT32_MAX，则返回解码失败
        if (str == NULL || bin_len > UINT32_MAX) {                             \
            return ARGON2_DECODING_FAIL;                                       \
        }                                                                      \
        # 将 bin_len 赋值给 len，并转换为 uint32_t 类型
        (len) = (uint32_t)bin_len;                                             \
    } while ((void)0, 0)

    # 定义变量 maxsaltlen，并初始化为 ctx->saltlen
    size_t maxsaltlen = ctx->saltlen;
    # 定义变量 maxoutlen，并初始化为 ctx->outlen
    size_t maxoutlen = ctx->outlen;
    # 定义变量 validation_result
    int validation_result;
    # 定义指向常量的指针 type_string
    const char* type_string;

    # 将 argon2_type 转换为字符串，赋值给 type_string
    type_string = argon2_type2string(type, 0);
    # 如果转换失败，则返回类型不正确
    if (!type_string) {
        return ARGON2_INCORRECT_TYPE;
    }

    # 输出字符串 "$"
    CC("$");
    # 输出 type_string
    CC(type_string);

    # 将 ARGON2_VERSION_10 赋值给 ctx->version
    ctx->version = ARGON2_VERSION_10;
    # 输出字符串 "$v="，并输出 ctx->version 的值
    CC_opt("$v=", DECIMAL_U32(ctx->version));

    # 输出字符串 "$m="，并输出 ctx->m_cost 的值
    CC("$m=");
    DECIMAL_U32(ctx->m_cost);
    # 输出字符串 ",t="，并输出 ctx->t_cost 的值
    CC(",t=");
    DECIMAL_U32(ctx->t_cost);
    # 输出字符串 ",p="，并输出 ctx->lanes 的值
    CC(",p=");
    DECIMAL_U32(ctx->lanes);
    # 将 ctx->lanes 的值赋值给 ctx->threads
    ctx->threads = ctx->lanes;

    # 输出字符串 "$"，并输出 ctx->salt 的值
    CC("$");
    BIN(ctx->salt, maxsaltlen, ctx->saltlen);
    # 输出字符串 "$"，并输出 ctx->out 的值
    CC("$");
    BIN(ctx->out, maxoutlen, ctx->outlen);

    # 将 NULL 赋值给 ctx->secret
    ctx->secret = NULL;
    # 将 0 赋值给 ctx->secretlen
    ctx->secretlen = 0;
    # 将 NULL 赋值给 ctx->ad
    ctx->ad = NULL;
    # 将 0 赋值给 ctx->adlen
    ctx->adlen = 0;
    # 将 NULL 赋值给 ctx->allocate_cbk
    ctx->allocate_cbk = NULL;
    # 将 NULL 赋值给 ctx->free_cbk
    ctx->free_cbk = NULL;
    # 将 ARGON2_DEFAULT_FLAGS 赋值给 ctx->flags
    ctx->flags = ARGON2_DEFAULT_FLAGS;

    # 验证输入参数是否有效，将结果赋值给 validation_result
    validation_result = xmrig_ar2_validate_inputs(ctx);
    # 如果验证失败，则返回验证结果
    if (validation_result != ARGON2_OK) {
        return validation_result;
    }

    # 如果字符串 str 为空，则返回操作成功，否则返回解码失败
    if (*str == 0) {
        return ARGON2_OK;
    } else {
        return ARGON2_DECODING_FAIL;
    }
// 取消定义 CC
#undef CC
// 取消定义 CC_opt
#undef CC_opt
// 取消定义 DECIMAL_U32
#undef DECIMAL_U32
// 取消定义 BIN
#undef BIN
}

// 编码字符串
int encode_string(char *dst, size_t dst_len, argon2_context *ctx,
                  argon2_type type) {
// 定义宏 SS，用于将字符串复制到目标缓冲区中
#define SS(str)                                                                \
    do {                                                                       \
        // 获取字符串长度
        size_t pp_len = strlen(str);                                           \
        // 如果字符串长度超过目标缓冲区长度，则返回编码失败
        if (pp_len >= dst_len) {                                               \
            return ARGON2_ENCODING_FAIL;                                       \
        }                                                                      \
        // 将字符串复制到目标缓冲区中
        memcpy(dst, str, pp_len + 1);                                          \
        dst += pp_len;                                                         \
        dst_len -= pp_len;                                                     \
    } while ((void)0, 0)

// 定义宏 SX，用于将整数转换为字符串并复制到目标缓冲区中
#define SX(x)                                                                  \
    do {                                                                       \
        char tmp[30];                                                          \
        // 将整数转换为字符串
        sprintf(tmp, "%lu", (unsigned long)(x));                               \
        // 调用宏 SS 将转换后的字符串复制到目标缓冲区中
        SS(tmp);                                                               \
    } while ((void)0, 0)

// 定义宏 SB，用于将二进制数据转换为 Base64 编码并复制到目标缓冲区中
#define SB(buf, len)                                                           \
    do {                                                                       \
        // 调用函数将二进制数据转换为 Base64 编码
        size_t sb_len = to_base64(dst, dst_len, buf, len);                     \
        // 如果转换失败，则返回编码失败
        if (sb_len == (size_t)-1) {                                            \
            return ARGON2_ENCODING_FAIL;                                       \
        }                                                                      \
        // 将转换后的 Base64 编码复制到目标缓冲区中
        dst += sb_len;                                                         \
        dst_len -= sb_len;                                                     \
    // 使用 do-while 循环，循环条件为 (void)0, 0，相当于无限循环
    } while ((void)0, 0)

    // 将 argon2_type 转换为对应的字符串表示
    const char* type_string = argon2_type2string(type, 0);
    // 验证输入参数是否有效
    int validation_result = xmrig_ar2_validate_inputs(ctx);

    // 如果 type_string 为空，则返回编码失败
    if (!type_string) {
        return ARGON2_ENCODING_FAIL;
    }

    // 如果验证结果不为 ARGON2_OK，则返回验证结果
    if (validation_result != ARGON2_OK) {
        return validation_result;
    }

    // 输出字符串 "$" 和 type_string
    SS("$");
    SS(type_string);

    // 输出字符串 "$v=" 和 ctx->version 的值
    SS("$v=");
    SX(ctx->version);

    // 输出字符串 "$m=" 和 ctx->m_cost 的值，以及字符串 ",t=" 和 ctx->t_cost 的值，以及字符串 ",p=" 和 ctx->lanes 的值
    SS("$m=");
    SX(ctx->m_cost);
    SS(",t=");
    SX(ctx->t_cost);
    SS(",p=");
    SX(ctx->lanes);

    // 输出字符串 "$" 和 ctx->salt 的内容，长度为 ctx->saltlen
    SS("$");
    SB(ctx->salt, ctx->saltlen);

    // 输出字符串 "$" 和 ctx->out 的内容，长度为 ctx->outlen
    SS("$");
    SB(ctx->out, ctx->outlen);
    // 返回 ARGON2_OK，表示操作成功
    return ARGON2_OK;
# 定义一个名为 b64len 的函数，用于计算 base64 编码后的长度
size_t b64len(uint32_t len) {
    # 计算 base64 编码后的长度
    size_t olen = ((size_t)len / 3) << 2;

    # 根据余数判断是否需要额外的长度
    switch (len % 3) {
    case 2:
        olen++;
    /* fall through */
    case 1:
        olen += 2;
        break;
    }

    # 返回计算出的长度
    return olen;
}

# 定义一个名为 numlen 的函数，用于计算数字的长度
size_t numlen(uint32_t num) {
    # 初始化长度为 1
    size_t len = 1;
    # 循环计算数字的长度
    while (num >= 10) {
        ++len;
        num = num / 10;
    }
    # 返回计算出的长度
    return len;
}
```