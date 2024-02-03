# `nmap\libssh2\src\blowfish.c`

```cpp
/*
 * $OpenBSD: blowfish.c,v 1.18 2004/11/02 17:23:26 hshoexer Exp $
 * 版本信息和修改记录
 */

/*
 * Blowfish block cipher for OpenBSD
 * 版权声明和说明这是为 OpenBSD 实现的 Blowfish 块密码
 * Copyright 1997 Niels Provos <provos@physnet.uni-hamburg.de>
 * 版权声明，作者信息
 * All rights reserved.
 * 保留所有权利
 *
 * Implementation advice by David Mazieres <dm@lcs.mit.edu>.
 * 实现建议来自 David Mazieres
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *      This product includes software developed by Niels Provos.
 * 4. The name of the author may not be used to endorse or promote products
 *    derived from this software without specific prior written permission.
 * 条款和条件，允许在源代码和二进制形式中重新分发和使用，但需要满足一定条件
 *
 * THIS SOFTWARE IS PROVIDED BY THE AUTHOR ``AS IS'' AND ANY EXPRESS OR
 * IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED.
 * IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY DIRECT, INDIRECT,
 * INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT
 * NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
 * DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
 * THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
 * (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF
 * THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
 * 软件的免责声明和责任限制
 */
/*
 * This code is derived from section 14.3 and the given source
 * in section V of Applied Cryptography, second edition.
 * Blowfish is an unpatented fast block cipher designed by
 * Bruce Schneier.
 */

#if !defined(HAVE_BCRYPT_PBKDF) && (!defined(HAVE_BLOWFISH_INITSTATE) || \
                                    !defined(HAVE_BLOWFISH_EXPAND0STATE) || \
                                    !defined(HAVE_BLF_ENC))

#if 0
#include <stdio.h>              /* used for debugging */
#include <string.h>
#endif

#include <sys/types.h>

#include "libssh2.h"
#include "blf.h"

#undef inline
#ifdef __GNUC__
#define inline __inline
#else                           /* !__GNUC__ */
#define inline
#endif                          /* !__GNUC__ */

/* Function for Feistel Networks */

#define F(s, x) ((((s)[        (((x)>>24)&0xFF)]        \
                   + (s)[0x100 + (((x)>>16)&0xFF)])     \
                  ^ (s)[0x200 + (((x)>> 8)&0xFF)])      \
                 + (s)[0x300 + ( (x)     &0xFF)])

#define BLFRND(s,p,i,j,n) (i ^= F(s,j) ^ (p)[n])

void
Blowfish_encipher(blf_ctx *c, uint32_t *xl, uint32_t *xr)
{
    uint32_t Xl;
    uint32_t Xr;
    uint32_t *s = c->S[0];
    uint32_t *p = c->P;

    Xl = *xl;
    Xr = *xr;

    Xl ^= p[0];
    BLFRND(s, p, Xr, Xl, 1); BLFRND(s, p, Xl, Xr, 2);
    BLFRND(s, p, Xr, Xl, 3); BLFRND(s, p, Xl, Xr, 4);
    BLFRND(s, p, Xr, Xl, 5); BLFRND(s, p, Xl, Xr, 6);
    BLFRND(s, p, Xr, Xl, 7); BLFRND(s, p, Xl, Xr, 8);
    BLFRND(s, p, Xr, Xl, 9); BLFRND(s, p, Xl, Xr, 10);
    BLFRND(s, p, Xr, Xl, 11); BLFRND(s, p, Xl, Xr, 12);
    BLFRND(s, p, Xr, Xl, 13); BLFRND(s, p, Xl, Xr, 14);
    BLFRND(s, p, Xr, Xl, 15); BLFRND(s, p, Xl, Xr, 16);

    *xl = Xr ^ p[17];
    *xr = Xl;
}

void
Blowfish_decipher(blf_ctx *c, uint32_t *xl, uint32_t *xr)
{
    uint32_t Xl;
    uint32_t Xr;
    uint32_t *s = c->S[0];
    uint32_t *p = c->P;

    Xl = *xl;
    Xr = *xr;

    Xl ^= p[17];
    BLFRND(s, p, Xr, Xl, 16); BLFRND(s, p, Xl, Xr, 15);
    # 调用 BLFRND 函数，进行一系列的数据处理和变换
    BLFRND(s, p, Xr, Xl, 14); 
    BLFRND(s, p, Xl, Xr, 13);
    BLFRND(s, p, Xr, Xl, 12); 
    BLFRND(s, p, Xl, Xr, 11);
    BLFRND(s, p, Xr, Xl, 10); 
    BLFRND(s, p, Xl, Xr, 9);
    BLFRND(s, p, Xr, Xl, 8); 
    BLFRND(s, p, Xl, Xr, 7);
    BLFRND(s, p, Xr, Xl, 6); 
    BLFRND(s, p, Xl, Xr, 5);
    BLFRND(s, p, Xr, Xl, 4); 
    BLFRND(s, p, Xl, Xr, 3);
    BLFRND(s, p, Xr, Xl, 2); 
    BLFRND(s, p, Xl, Xr, 1);

    # 对xl和xr进行赋值操作
    *xl = Xr ^ p[0];
    *xr = Xl;
}

void
Blowfish_initstate(blf_ctx *c)
{
    /* 初始化 Blowfish 算法的状态 */

    *c = initstate;
}

uint32_t
Blowfish_stream2word(const uint8_t *data, uint16_t databytes,
                     uint16_t *current)
{
    uint8_t i;
    uint16_t j;
    uint32_t temp;

    temp = 0x00000000;
    j = *current;

    for(i = 0; i < 4; i++, j++) {
        if(j >= databytes)
            j = 0;
        temp = (temp << 8) | data[j];
    }

    *current = j;
    return temp;
}

void
Blowfish_expand0state(blf_ctx *c, const uint8_t *key, uint16_t keybytes)
{
    uint16_t i;
    uint16_t j;
    uint16_t k;
    uint32_t temp;
    uint32_t datal;
    uint32_t datar;

    j = 0;
    for(i = 0; i < BLF_N + 2; i++) {
        /* 从密钥流中提取4个int8到1个int32 */
        temp = Blowfish_stream2word(key, keybytes, &j);
        c->P[i] = c->P[i] ^ temp;
    }

    j = 0;
    datal = 0x00000000;
    datar = 0x00000000;
    for(i = 0; i < BLF_N + 2; i += 2) {
        Blowfish_encipher(c, &datal, &datar);

        c->P[i] = datal;
        c->P[i + 1] = datar;
    }

    for(i = 0; i < 4; i++) {
        for(k = 0; k < 256; k += 2) {
            Blowfish_encipher(c, &datal, &datar);

            c->S[i][k] = datal;
            c->S[i][k + 1] = datar;
        }
    }
}


void
Blowfish_expandstate(blf_ctx *c, const uint8_t *data, uint16_t databytes,
                     const uint8_t *key, uint16_t keybytes)
{
    uint16_t i;
    uint16_t j;
    uint16_t k;
    uint32_t temp;
    uint32_t datal;
    uint32_t datar;

    j = 0;
    for(i = 0; i < BLF_N + 2; i++) {
        /* 从密钥流中提取4个int8到1个int32 */
        temp = Blowfish_stream2word(key, keybytes, &j);
        c->P[i] = c->P[i] ^ temp;
    }

    j = 0;
    datal = 0x00000000;
    datar = 0x00000000;
    # 对于每个 i 从 0 到 BLF_N + 2，每次增加 2
    for(i = 0; i < BLF_N + 2; i += 2) {
        # 对 datal 和 datar 进行异或操作
        datal ^= Blowfish_stream2word(data, databytes, &j);
        datar ^= Blowfish_stream2word(data, databytes, &j);
        # 使用 Blowfish 算法对 datal 和 datar 进行加密
        Blowfish_encipher(c, &datal, &datar);

        # 将加密后的 datal 和 datar 存入密钥数组的相应位置
        c->P[i] = datal;
        c->P[i + 1] = datar;
    }

    # 对于每个 i 从 0 到 4
    for(i = 0; i < 4; i++) {
        # 对于每个 k 从 0 到 256，每次增加 2
        for(k = 0; k < 256; k += 2) {
            # 对 datal 和 datar 进行异或操作
            datal ^= Blowfish_stream2word(data, databytes, &j);
            datar ^= Blowfish_stream2word(data, databytes, &j);
            # 使用 Blowfish 算法对 datal 和 datar 进行加密
            Blowfish_encipher(c, &datal, &datar);

            # 将加密后的 datal 和 datar 存入 S 盒的相应位置
            c->S[i][k] = datal;
            c->S[i][k + 1] = datar;
        }
    }
# 结束函数定义
}

# 定义函数，用于设置初始的 S 盒和子密钥
void
blf_key(blf_ctx *c, const uint8_t *k, uint16_t len)
{
    # 使用初始置换表初始化 S 盒和子密钥
    Blowfish_initstate(c);

    # 使用密钥对 S 盒和子密钥进行变换
    Blowfish_expand0state(c, k, len);
}

# 定义函数，用于对数据进行加密
void
blf_enc(blf_ctx *c, uint32_t *data, uint16_t blocks)
{
    uint32_t *d;
    uint16_t i;

    d = data;
    for(i = 0; i < blocks; i++) {
        # 对数据进行加密
        Blowfish_encipher(c, d, d + 1);
        d += 2;
    }
}

# 定义函数，用于对数据进行解密
void
blf_dec(blf_ctx *c, uint32_t *data, uint16_t blocks)
{
    uint32_t *d;
    uint16_t i;

    d = data;
    for(i = 0; i < blocks; i++) {
        # 对数据进行解密
        Blowfish_decipher(c, d, d + 1);
        d += 2;
    }
}

# 定义函数，用于对数据进行 ECB 模式加密
void
blf_ecb_encrypt(blf_ctx *c, uint8_t *data, uint32_t len)
{
    uint32_t l, r;
    uint32_t i;

    for(i = 0; i < len; i += 8) {
        # 将数据分成左右两部分，进行加密
        l = data[0] << 24 | data[1] << 16 | data[2] << 8 | data[3];
        r = data[4] << 24 | data[5] << 16 | data[6] << 8 | data[7];
        Blowfish_encipher(c, &l, &r);
        # 将加密后的数据重新组合
        data[0] = l >> 24 & 0xff;
        data[1] = l >> 16 & 0xff;
        data[2] = l >> 8 & 0xff;
        data[3] = l & 0xff;
        data[4] = r >> 24 & 0xff;
        data[5] = r >> 16 & 0xff;
        data[6] = r >> 8 & 0xff;
        data[7] = r & 0xff;
        data += 8;
    }
}

# 定义函数，用于对数据进行 ECB 模式解密
void
blf_ecb_decrypt(blf_ctx *c, uint8_t *data, uint32_t len)
{
    uint32_t l, r;
    uint32_t i;

    for(i = 0; i < len; i += 8) {
        # 将数据分成左右两部分，进行解密
        l = data[0] << 24 | data[1] << 16 | data[2] << 8 | data[3];
        r = data[4] << 24 | data[5] << 16 | data[6] << 8 | data[7];
        Blowfish_decipher(c, &l, &r);
        # 将解密后的数据重新组合
        data[0] = l >> 24 & 0xff;
        data[1] = l >> 16 & 0xff;
        data[2] = l >> 8 & 0xff;
        data[3] = l & 0xff;
        data[4] = r >> 24 & 0xff;
        data[5] = r >> 16 & 0xff;
        data[6] = r >> 8 & 0xff;
        data[7] = r & 0xff;
        data += 8;
    }
}

# 定义函数，用于对数据进行 CBC 模式加密
void
blf_cbc_encrypt(blf_ctx *c, uint8_t *iv, uint8_t *data, uint32_t len)
{
    uint32_t l, r;
    uint32_t i, j;
    # 每次循环增加8，对数据进行分组处理
    for(i = 0; i < len; i += 8) {
        # 对数据和初始化向量进行异或操作
        for(j = 0; j < 8; j++)
            data[j] ^= iv[j];
        # 将数据分成左右两部分，每部分4个字节，合并成32位整数
        l = data[0] << 24 | data[1] << 16 | data[2] << 8 | data[3];
        r = data[4] << 24 | data[5] << 16 | data[6] << 8 | data[7];
        # 使用Blowfish算法对数据进行加密
        Blowfish_encipher(c, &l, &r);
        # 将加密后的左右两部分数据重新分配到原始数据数组中
        data[0] = l >> 24 & 0xff;
        data[1] = l >> 16 & 0xff;
        data[2] = l >> 8 & 0xff;
        data[3] = l & 0xff;
        data[4] = r >> 24 & 0xff;
        data[5] = r >> 16 & 0xff;
        data[6] = r >> 8 & 0xff;
        data[7] = r & 0xff;
        # 更新初始化向量为加密后的数据
        iv = data;
        # 数据指针移动到下一个分组
        data += 8;
    }
void
blf_cbc_decrypt(blf_ctx *c, uint8_t *iva, uint8_t *data, uint32_t len)
{
    uint32_t l, r;  // 声明两个32位整型变量l和r
    uint8_t *iv;  // 声明一个指向uint8_t类型的指针变量iv
    uint32_t i, j;  // 声明两个32位整型变量i和j

    iv = data + len - 16;  // 将指针iv指向data + len - 16
    data = data + len - 8;  // 将指针data指向data + len - 8
    for(i = len - 8; i >= 8; i -= 8) {  // 循环，i从len-8开始递减，直到小于8
        l = data[0] << 24 | data[1] << 16 | data[2] << 8 | data[3];  // 将data数组中的4个元素按位左移并按位或，赋值给l
        r = data[4] << 24 | data[5] << 16 | data[6] << 8 | data[7];  // 将data数组中的4个元素按位左移并按位或，赋值给r
        Blowfish_decipher(c, &l, &r);  // 调用Blowfish_decipher函数
        // 将l和r的值按位右移并按位与，赋值给data数组中的8个元素
        data[0] = l >> 24 & 0xff;
        data[1] = l >> 16 & 0xff;
        data[2] = l >> 8 & 0xff;
        data[3] = l & 0xff;
        data[4] = r >> 24 & 0xff;
        data[5] = r >> 16 & 0xff;
        data[6] = r >> 8 & 0xff;
        data[7] = r & 0xff;
        for(j = 0; j < 8; j++)  // 循环，j从0到7
            data[j] ^= iv[j];  // data数组中的每个元素与iv数组中对应元素进行异或操作
        iv -= 8;  // 指针iv向前移动8个位置
        data -= 8;  // 指针data向前移动8个位置
    }
    l = data[0] << 24 | data[1] << 16 | data[2] << 8 | data[3];  // 将data数组中的4个元素按位左移并按位或，赋值给l
    r = data[4] << 24 | data[5] << 16 | data[6] << 8 | data[7];  // 将data数组中的4个元素按位左移并按位或，赋值给r
    Blowfish_decipher(c, &l, &r);  // 调用Blowfish_decipher函数
    // 将l和r的值按位右移并按位与，赋值给data数组中的8个元素
    data[0] = l >> 24 & 0xff;
    data[1] = l >> 16 & 0xff;
    data[2] = l >> 8 & 0xff;
    data[3] = l & 0xff;
    data[4] = r >> 24 & 0xff;
    data[5] = r >> 16 & 0xff;
    data[6] = r >> 8 & 0xff;
    data[7] = r & 0xff;
    for(j = 0; j < 8; j++)  // 循环，j从0到7
        data[j] ^= iva[j];  // data数组中的每个元素与iva数组中对应元素进行异或操作
}
    # 打印提示信息，应该读取为: 0x324ed0fe 0xf413a203
    printf("\nShould read as: 0x324ed0fe 0xf413a203.\n");
    # 报告数据内容，显示2个数据
    report(data2, 2);
    # 使用blf_dec函数对数据进行解密
    blf_dec(&c, data2, 1);
    # 再次报告数据内容，显示2个数据
    report(data2, 2);
}
#endif

#endif /* !defined(HAVE_BCRYPT_PBKDF) && \
          (!defined(HAVE_BLOWFISH_INITSTATE) ||   \
          !defined(HAVE_BLOWFISH_EXPAND0STATE) || \
          '!defined(HAVE_BLF_ENC)) */

这部分代码是条件编译的结束部分，用于结束#if和#endif之间的代码块。在这里，检查了一系列宏定义的条件，如果条件不满足，则执行#endif下面的代码。
```