# `nmap\libssh2\src\bcrypt_pbkdf.c`

```cpp
# 版权声明和许可声明
/* $OpenBSD: bcrypt_pbkdf.c,v 1.4 2013/07/29 00:55:53 tedu Exp $ */
/*
 * Copyright (c) 2013 Ted Unangst <tedu@openbsd.org>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 */

# 如果没有定义 HAVE_BCRYPT_PBKDF，则包含以下代码
#ifndef HAVE_BCRYPT_PBKDF

# 包含 libssh2_priv.h 文件
#include "libssh2_priv.h"
# 包含标准库文件
#include <stdlib.h>
# 包含系统类型文件
#include <sys/types.h>
# 如果定义了 HAVE_SYS_PARAM_H，则包含该文件
#ifdef HAVE_SYS_PARAM_H
#include <sys/param.h>
#endif

# 包含 blf.h 文件
#include "blf.h"

# 定义一个宏函数，返回两个数中的最小值
#define MINIMUM(a,b) (((a) < (b)) ? (a) : (b))
/*
 * 使用“bcrypt”哈希函数实现的pkcs #5 pbkdf2
 *
 * bcrypt哈希函数是从bcrypt密码哈希函数派生而来，具有以下修改：
 * 1. 输入密码和盐值经过SHA512预处理。
 * 2. 输出长度扩展为256位。
 * 3. 随后要加密的魔术字符串长度增加并修改为“OxychromaticBlowfishSwatDynamite”。
 * 4. 哈希函数被定义为执行64轮初始状态扩展。（通过迭代哈希执行更多轮次。）
 *
 * 注意，此实现将SHA512操作拉入调用者作为性能优化。
 *
 * 与官方pbkdf2的一个修改。我们对输出的密钥材料进行混合处理。pbkdf2存在一个已知的弱点，即如果使用它生成（例如）512位密钥材料用作两个256位密钥，攻击者只需通过下面的外部循环运行一次，但用户总是运行两次。混洗输出字节需要计算整个密钥材料以组装任何子密钥。这是一个明智的调用者可以做的事情；我们只是为您做了。
 */

#define BCRYPT_BLOCKS 8
#define BCRYPT_HASHSIZE (BCRYPT_BLOCKS * 4)

static void
bcrypt_hash(uint8_t *sha2pass, uint8_t *sha2salt, uint8_t *out)
{
    blf_ctx state;
    uint8_t ciphertext[BCRYPT_HASHSIZE] =
        "OxychromaticBlowfishSwatDynamite";
    uint32_t cdata[BCRYPT_BLOCKS];
    int i;
    uint16_t j;
    size_t shalen = SHA512_DIGEST_LENGTH;

    /* 密钥扩展 */
    Blowfish_initstate(&state);
    Blowfish_expandstate(&state, sha2salt, shalen, sha2pass, shalen);
    for(i = 0; i < 64; i++) {
        Blowfish_expand0state(&state, sha2salt, shalen);
        Blowfish_expand0state(&state, sha2pass, shalen);
    }

    /* 加密 */
    j = 0;
    # 使用循环将加密后的数据解密成原始数据块
    for(i = 0; i < BCRYPT_BLOCKS; i++)
        cdata[i] = Blowfish_stream2word(ciphertext, sizeof(ciphertext),
                                        &j);
    # 使用循环对解密后的数据块进行再次加密
    for(i = 0; i < 64; i++)
        blf_enc(&state, cdata, BCRYPT_BLOCKS / 2);

    # 将加密后的数据块按照特定顺序复制到输出数组中
    for(i = 0; i < BCRYPT_BLOCKS; i++) {
        out[4 * i + 3] = (cdata[i] >> 24) & 0xff;
        out[4 * i + 2] = (cdata[i] >> 16) & 0xff;
        out[4 * i + 1] = (cdata[i] >> 8) & 0xff;
        out[4 * i + 0] = cdata[i] & 0xff;
    }

    # 清空加密后的数据和状态
    _libssh2_explicit_zero(ciphertext, sizeof(ciphertext));
    _libssh2_explicit_zero(cdata, sizeof(cdata));
    _libssh2_explicit_zero(&state, sizeof(state));
}

int
bcrypt_pbkdf(const char *pass, size_t passlen, const uint8_t *salt,
             size_t saltlen,
             uint8_t *key, size_t keylen, unsigned int rounds)
{
    uint8_t sha2pass[SHA512_DIGEST_LENGTH];  // 用于存储经过 SHA512 哈希后的密码
    uint8_t sha2salt[SHA512_DIGEST_LENGTH];  // 用于存储经过 SHA512 哈希后的盐值
    uint8_t out[BCRYPT_HASHSIZE];  // 用于存储生成的密钥
    uint8_t tmpout[BCRYPT_HASHSIZE];  // 用于临时存储生成的密钥
    uint8_t *countsalt;  // 用于存储盐值和计数器
    size_t i, j, amt, stride;  // 用于循环计数和存储计算结果
    uint32_t count;  // 用于存储计数器
    size_t origkeylen = keylen;  // 用于存储原始密钥长度
    libssh2_sha512_ctx ctx;  // 用于存储 SHA512 上下文

    /* nothing crazy */
    if(rounds < 1)  // 如果轮数小于 1，则返回错误
        return -1;
    if(passlen == 0 || saltlen == 0 || keylen == 0 ||
       keylen > sizeof(out) * sizeof(out) || saltlen > 1<<20)  // 如果密码长度、盐值长度、密钥长度不合法，或者密钥长度超出范围，或者盐值长度过大，则返回错误
        return -1;
    countsalt = calloc(1, saltlen + 4);  // 分配内存用于存储盐值和计数器
    if(countsalt == NULL)  // 如果内存分配失败，则返回错误
        return -1;
    stride = (keylen + sizeof(out) - 1) / sizeof(out);  // 计算密钥长度所需的步长
    amt = (keylen + stride - 1) / stride;  // 计算每次生成密钥的长度

    memcpy(countsalt, salt, saltlen);  // 将盐值复制到计数盐值中

    /* collapse password */
    libssh2_sha512_init(&ctx);  // 初始化 SHA512 上下文
    libssh2_sha512_update(ctx, pass, passlen);  // 更新 SHA512 上下文，传入密码和密码长度
    libssh2_sha512_final(ctx, sha2pass);  // 计算 SHA512 哈希值，得到经过哈希后的密码

    /* generate key, sizeof(out) at a time */
    for(count = 1; keylen > 0; count++) {
        // 循环计算哈希值，直到 keylen 为 0
        countsalt[saltlen + 0] = (count >> 24) & 0xff;
        countsalt[saltlen + 1] = (count >> 16) & 0xff;
        countsalt[saltlen + 2] = (count >> 8) & 0xff;
        countsalt[saltlen + 3] = count & 0xff;

        /* first round, salt is salt */
        // 初始化 SHA512 上下文
        libssh2_sha512_init(&ctx);
        // 更新 SHA512 上下文
        libssh2_sha512_update(ctx, countsalt, saltlen + 4);
        // 计算 SHA512 哈希值
        libssh2_sha512_final(ctx, sha2salt);

        // 计算 bcrypt 哈希值
        bcrypt_hash(sha2pass, sha2salt, tmpout);
        // 复制哈希值到输出
        memcpy(out, tmpout, sizeof(out));

        for(i = 1; i < rounds; i++) {
            /* subsequent rounds, salt is previous output */
            // 初始化 SHA512 上下文
            libssh2_sha512_init(&ctx);
            // 更新 SHA512 上下文
            libssh2_sha512_update(ctx, tmpout, sizeof(tmpout));
            // 计算 SHA512 哈希值
            libssh2_sha512_final(ctx, sha2salt);

            // 计算 bcrypt 哈希值
            bcrypt_hash(sha2pass, sha2salt, tmpout);
            for(j = 0; j < sizeof(out); j++)
                // 对输出进行异或操作
                out[j] ^= tmpout[j];
        }

        /*
         * pbkdf2 deviation: output the key material non-linearly.
         */
        // 计算非线性输出的密钥材料
        amt = MINIMUM(amt, keylen);
        for(i = 0; i < amt; i++) {
            size_t dest = i * stride + (count - 1);
            if(dest >= origkeylen) {
                break;
            }
            // 将输出复制到密钥中
            key[dest] = out[i];
        }
        keylen -= i;
    }

    /* zap */
    // 清空输出
    _libssh2_explicit_zero(out, sizeof(out));
    // 释放内存
    free(countsalt);

    // 返回 0
    return 0;
# 结束条件编译指令，如果定义了 HAVE_BCRYPT_PBKDF 则编译该部分代码
#endif /* HAVE_BCRYPT_PBKDF */
```