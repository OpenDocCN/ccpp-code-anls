# `nmap\libssh2\src\blf.h`

```
#ifndef __LIBSSH2_BLF_H
#define __LIBSSH2_BLF_H
/* $OpenBSD: blf.h,v 1.7 2007/03/14 17:59:41 grunk Exp $ */
/*
 * Blowfish - a fast block cipher designed by Bruce Schneier
 * Blowfish - 由 Bruce Schneier 设计的快速分组密码
 *
 * Copyright 1997 Niels Provos <provos@physnet.uni-hamburg.de>
 * All rights reserved.
 * 版权所有，保留所有权利
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 *    源代码的再分发和使用，无论是否经过修改，都必须满足以下条件：
 *    1. 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明。
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 *    2. 以二进制形式再分发时，必须在文档和/或其他提供的材料中重现上述版权声明、此条件列表和以下免责声明。
 * 3. All advertising materials mentioning features or use of this software
 *    must display the following acknowledgement:
 *      This product includes software developed by Niels Provos.
 *    3. 所有提及此软件特性或使用的广告材料必须显示以下声明：
 *      本产品包含 Niels Provos 开发的软件。
 * 4. The name of the author may not be used to endorse or promote products
 *    derived from this software without specific prior written permission.
 *    4. 未经特定事先书面许可，不得使用作者的姓名来代言或推广基于此软件的产品。
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
 * 此软件由作者按原样提供，任何明示或暗示的保证，包括但不限于对适销性和特定用途的暗示保证，都被拒绝。
 * 在任何情况下，作者都不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用损失、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何责任理论下，即使已被告知可能发生此类损害。
 */

#if !defined(HAVE_BCRYPT_PBKDF) && !defined(HAVE_BLH_H)
/* Schneier specifies a maximum key length of 56 bytes.
 * This ensures that every key bit affects every cipher
 * bit.  However, the subkeys can hold up to 72 bytes.
 * Warning: For normal blowfish encryption only 56 bytes
 * of the key affect all cipherbits.
 */
 
#define BLF_N   16                      /* Number of Subkeys */
#define BLF_MAXKEYLEN ((BLF_N-2)*4)     /* 448 bits */
#define BLF_MAXUTILIZED ((BLF_N + 2)*4)   /* 576 bits */

/* Blowfish context */
typedef struct BlowfishContext {
        uint32_t S[4][256];     /* S-Boxes */
        uint32_t P[BLF_N + 2];  /* Subkeys */
} blf_ctx;

/* Raw access to customized Blowfish
 *      blf_key is just:
 *      Blowfish_initstate( state )
 *      Blowfish_expand0state( state, key, keylen )
 */
 
void Blowfish_encipher(blf_ctx *, uint32_t *, uint32_t *);
void Blowfish_decipher(blf_ctx *, uint32_t *, uint32_t *);
void Blowfish_initstate(blf_ctx *);
void Blowfish_expand0state(blf_ctx *, const uint8_t *, uint16_t);
void Blowfish_expandstate
(blf_ctx *, const uint8_t *, uint16_t, const uint8_t *, uint16_t);

/* Standard Blowfish */

void blf_key(blf_ctx *, const uint8_t *, uint16_t);
void blf_enc(blf_ctx *, uint32_t *, uint16_t);
void blf_dec(blf_ctx *, uint32_t *, uint16_t);

void blf_ecb_encrypt(blf_ctx *, uint8_t *, uint32_t);
void blf_ecb_decrypt(blf_ctx *, uint8_t *, uint32_t);

void blf_cbc_encrypt(blf_ctx *, uint8_t *, uint8_t *, uint32_t);
void blf_cbc_decrypt(blf_ctx *, uint8_t *, uint8_t *, uint32_t);

/* Converts uint8_t to uint32_t */
uint32_t Blowfish_stream2word(const uint8_t *, uint16_t, uint16_t *);

/* bcrypt with pbkd */
int bcrypt_pbkdf(const char *pass, size_t passlen, const uint8_t *salt,
                 size_t saltlen,
                 uint8_t *key, size_t keylen, unsigned int rounds);

#endif /* !defined(HAVE_BCRYPT_PBKDF) && !defined(HAVE_BLH_H) */
#endif /* __LIBSSH2_BLF_H */
```