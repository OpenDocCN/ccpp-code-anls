# `nmap\libssh2\src\libgcrypt.h`

```cpp
#ifndef __LIBSSH2_LIBGCRYPT_H
#define __LIBSSH2_LIBGCRYPT_H
/*
 * 版权声明
 * 版权所有（C）2008年，2009年，2010年Simon Josefsson
 * 版权所有（C）2006年，2007年，The Written Word, Inc.
 * 保留所有权利。
 *
 * 在源代码和二进制形式中重新分发和使用，无论是否经过修改，都是允许的，只要满足以下条件：
 *
 *   1. 源代码的再分发必须保留上述版权声明、此条件列表和以下免责声明。
 *
 *   2. 二进制形式的再分发必须在文档和/或其他提供的材料中重现上述版权声明、此条件列表和以下免责声明。
 *
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或任何其他贡献者的名称来认可或推广从本软件派生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */

#include <gcrypt.h>

#define LIBSSH2_MD5 1

#define LIBSSH2_HMAC_RIPEMD 1
#define LIBSSH2_HMAC_SHA256 1
#define LIBSSH2_HMAC_SHA512 1

#define LIBSSH2_AES 1
#define LIBSSH2_AES_CTR 1
#define LIBSSH2_BLOWFISH 1
#define LIBSSH2_RC4 1
#define LIBSSH2_CAST 1
#define LIBSSH2_3DES 1
#define LIBSSH2_RSA 1
#define LIBSSH2_DSA 1
#define LIBSSH2_ECDSA 0
#define LIBSSH2_ED25519 0
# 定义了一些常量，用于指定加密算法的支持情况

#define MD5_DIGEST_LENGTH 16
#define SHA_DIGEST_LENGTH 20
#define SHA256_DIGEST_LENGTH 32
#define SHA384_DIGEST_LENGTH 48
#define SHA512_DIGEST_LENGTH 64
# 定义了一些常量，用于指定不同哈希算法的摘要长度

#define EC_MAX_POINT_LEN ((528 * 2 / 8) + 1)
# 定义了椭圆曲线加密算法中点的最大长度

#define _libssh2_random(buf, len)                \
  (gcry_randomize ((buf), (len), GCRY_STRONG_RANDOM), 0)
# 定义了一个宏，用于生成随机数

#define libssh2_prepare_iovec(vec, len)  /* Empty. */
# 定义了一个空的函数，用于准备 I/O 向量

#define libssh2_sha1_ctx gcry_md_hd_t
# 定义了 SHA1 哈希算法的上下文类型

#define libssh2_sha1_init(ctx) \
  (GPG_ERR_NO_ERROR == gcry_md_open(ctx,  GCRY_MD_SHA1, 0))
# 定义了 SHA1 哈希算法的初始化函数

#define libssh2_sha1_update(ctx, data, len) \
  gcry_md_write(ctx, (unsigned char *) data, len)
# 定义了 SHA1 哈希算法的更新函数

#define libssh2_sha1_final(ctx, out) \
  memcpy(out, gcry_md_read(ctx, 0), SHA_DIGEST_LENGTH), gcry_md_close(ctx)
# 定义了 SHA1 哈希算法的最终化函数

#define libssh2_sha1(message, len, out) \
  gcry_md_hash_buffer(GCRY_MD_SHA1, out, message, len)
# 定义了 SHA1 哈希算法的哈希函数

#define libssh2_sha256_ctx gcry_md_hd_t
# 定义了 SHA256 哈希算法的上下文类型

#define libssh2_sha256_init(ctx) \
  (GPG_ERR_NO_ERROR == gcry_md_open(ctx,  GCRY_MD_SHA256, 0))
# 定义了 SHA256 哈希算法的初始化函数

#define libssh2_sha256_update(ctx, data, len) \
  gcry_md_write(ctx, (unsigned char *) data, len)
# 定义了 SHA256 哈希算法的更新函数

#define libssh2_sha256_final(ctx, out) \
  memcpy(out, gcry_md_read(ctx, 0), SHA256_DIGEST_LENGTH), gcry_md_close(ctx)
# 定义了 SHA256 哈希算法的最终化函数

#define libssh2_sha256(message, len, out) \
  gcry_md_hash_buffer(GCRY_MD_SHA256, out, message, len)
# 定义了 SHA256 哈希算法的哈希函数

#define libssh2_sha384_ctx gcry_md_hd_t
# 定义了 SHA384 哈希算法的上下文类型

#define libssh2_sha384_init(ctx) \
  (GPG_ERR_NO_ERROR == gcry_md_open(ctx,  GCRY_MD_SHA384, 0))
# 定义了 SHA384 哈希算法的初始化函数

#define libssh2_sha384_update(ctx, data, len) \
  gcry_md_write(ctx, (unsigned char *) data, len)
# 定义了 SHA384 哈希算法的更新函数

#define libssh2_sha384_final(ctx, out) \
  memcpy(out, gcry_md_read(ctx, 0), SHA384_DIGEST_LENGTH), gcry_md_close(ctx)
# 定义了 SHA384 哈希算法的最终化函数

#define libssh2_sha384(message, len, out) \
  gcry_md_hash_buffer(GCRY_MD_SHA384, out, message, len)
# 定义了 SHA384 哈希算法的哈希函数

#define libssh2_sha512_ctx gcry_md_hd_t
# 定义了 SHA512 哈希算法的上下文类型
/* 定义 libssh2_sha512_init 函数，初始化 SHA512 上下文 */
#define libssh2_sha512_init(ctx) \
  (GPG_ERR_NO_ERROR == gcry_md_open(ctx,  GCRY_MD_SHA512, 0))
/* 定义 libssh2_sha512_update 函数，更新 SHA512 上下文 */
#define libssh2_sha512_update(ctx, data, len) \
  gcry_md_write(ctx, (unsigned char *) data, len)
/* 定义 libssh2_sha512_final 函数，结束 SHA512 上下文 */
#define libssh2_sha512_final(ctx, out) \
  memcpy(out, gcry_md_read(ctx, 0), SHA512_DIGEST_LENGTH), gcry_md_close(ctx)
/* 定义 libssh2_sha512 函数，计算 SHA512 哈希值 */
#define libssh2_sha512(message, len, out) \
  gcry_md_hash_buffer(GCRY_MD_SHA512, out, message, len)

/* 定义 libssh2_md5_ctx 类型为 gcry_md_hd_t */
#define libssh2_md5_ctx gcry_md_hd_t
/* 定义 libssh2_md5_init 函数，初始化 MD5 上下文 */
#define libssh2_md5_init(ctx) \
  (GPG_ERR_NO_ERROR == gcry_md_open(ctx,  GCRY_MD_MD5, 0))
/* 定义 libssh2_md5_update 函数，更新 MD5 上下文 */
#define libssh2_md5_update(ctx, data, len) \
  gcry_md_write(ctx, (unsigned char *) data, len)
/* 定义 libssh2_md5_final 函数，结束 MD5 上下文 */
#define libssh2_md5_final(ctx, out) \
  memcpy(out, gcry_md_read(ctx, 0), MD5_DIGEST_LENGTH), gcry_md_close(ctx)
/* 定义 libssh2_md5 函数，计算 MD5 哈希值 */
#define libssh2_md5(message, len, out) \
  gcry_md_hash_buffer(GCRY_MD_MD5, out, message, len)

/* 定义 libssh2_hmac_ctx 类型为 gcry_md_hd_t */
#define libssh2_hmac_ctx gcry_md_hd_t
/* 定义 libssh2_hmac_ctx_init 函数，初始化 HMAC 上下文 */
#define libssh2_hmac_ctx_init(ctx)
/* 定义 libssh2_hmac_sha1_init 函数，初始化 HMAC-SHA1 上下文 */
#define libssh2_hmac_sha1_init(ctx, key, keylen) \
  gcry_md_open(ctx, GCRY_MD_SHA1, GCRY_MD_FLAG_HMAC), \
    gcry_md_setkey(*ctx, key, keylen)
/* 定义 libssh2_hmac_md5_init 函数，初始化 HMAC-MD5 上下文 */
#define libssh2_hmac_md5_init(ctx, key, keylen) \
  gcry_md_open(ctx, GCRY_MD_MD5, GCRY_MD_FLAG_HMAC), \
    gcry_md_setkey(*ctx, key, keylen)
/* 定义 libssh2_hmac_ripemd160_init 函数，初始化 HMAC-RIPEMD160 上下文 */
#define libssh2_hmac_ripemd160_init(ctx, key, keylen) \
  gcry_md_open(ctx, GCRY_MD_RMD160, GCRY_MD_FLAG_HMAC), \
    gcry_md_setkey(*ctx, key, keylen)
/* 定义 libssh2_hmac_sha256_init 函数，初始化 HMAC-SHA256 上下文 */
#define libssh2_hmac_sha256_init(ctx, key, keylen) \
  gcry_md_open(ctx, GCRY_MD_SHA256, GCRY_MD_FLAG_HMAC), \
    gcry_md_setkey(*ctx, key, keylen)
/* 定义 libssh2_hmac_sha512_init 函数，初始化 HMAC-SHA512 上下文 */
#define libssh2_hmac_sha512_init(ctx, key, keylen) \
  gcry_md_open(ctx, GCRY_MD_SHA512, GCRY_MD_FLAG_HMAC), \
    gcry_md_setkey(*ctx, key, keylen)
/* 定义 libssh2_hmac_update 函数，更新 HMAC 上下文 */
#define libssh2_hmac_update(ctx, data, datalen) \
  gcry_md_write(ctx, (unsigned char *) data, datalen)
/* 定义 libssh2_hmac_final 函数，结束 HMAC 上下文 */
#define libssh2_hmac_final(ctx, data) \
  memcpy(data, gcry_md_read(ctx, 0), \
      gcry_md_get_algo_dlen(gcry_md_get_algo(ctx)))
/* 定义 libssh2_hmac_cleanup 函数，清理 HMAC 上下文 */
#define libssh2_hmac_cleanup(ctx) gcry_md_close (*ctx);
# 定义 libssh2_crypto_init() 函数，用于禁用安全内存
#define libssh2_crypto_init() gcry_control (GCRYCTL_DISABLE_SECMEM)

# 定义 libssh2_crypto_exit() 函数

#define libssh2_crypto_exit()

# 定义 libssh2_rsa_ctx 结构体为 gcry_sexp 结构体

#define libssh2_rsa_ctx struct gcry_sexp

# 定义 _libssh2_rsa_free(rsactx) 函数，用于释放 rsactx 对象

#define _libssh2_rsa_free(rsactx)  gcry_sexp_release (rsactx)

# 定义 libssh2_dsa_ctx 结构体为 gcry_sexp 结构体

#define libssh2_dsa_ctx struct gcry_sexp

# 定义 _libssh2_dsa_free(dsactx) 函数，用于释放 dsactx 对象

#define _libssh2_dsa_free(dsactx)  gcry_sexp_release (dsactx)

# 如果 LIBSSH2_ECDSA 宏定义存在，则执行下面的代码，否则定义 _libssh2_ec_key 为空

#if LIBSSH2_ECDSA
#else
#define _libssh2_ec_key void
#endif

# 定义 _libssh2_cipher_type(name) 函数，返回整型 name

#define _libssh2_cipher_type(name) int name

# 定义 _libssh2_cipher_ctx 结构体为 gcry_cipher_hd_t 结构体

#define _libssh2_cipher_ctx gcry_cipher_hd_t

# 定义 _libssh2_gcry_ciphermode(c,m) 函数，返回 c 和 m 的按位或结果

#define _libssh2_gcry_ciphermode(c,m) ((c << 8) | m)

# 定义 _libssh2_gcry_cipher(c) 函数，返回 c 右移 8 位的结果

#define _libssh2_gcry_cipher(c) (c >> 8)

# 定义 _libssh2_gcry_mode(m) 函数，返回 m 与 0xFF 的按位与结果

#define _libssh2_gcry_mode(m) (m & 0xFF)

# 定义 _libssh2_cipher_aes256ctr 宏，返回 GCRY_CIPHER_AES256 和 GCRY_CIPHER_MODE_CTR 的按位或结果

#define _libssh2_cipher_aes256ctr \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_AES256, GCRY_CIPHER_MODE_CTR)

# 定义 _libssh2_cipher_aes192ctr 宏，返回 GCRY_CIPHER_AES192 和 GCRY_CIPHER_MODE_CTR 的按位或结果

#define _libssh2_cipher_aes192ctr \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_AES192, GCRY_CIPHER_MODE_CTR)

# 定义 _libssh2_cipher_aes128ctr 宏，返回 GCRY_CIPHER_AES128 和 GCRY_CIPHER_MODE_CTR 的按位或结果

#define _libssh2_cipher_aes128ctr \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_AES128, GCRY_CIPHER_MODE_CTR)

# 定义 _libssh2_cipher_aes256 宏，返回 GCRY_CIPHER_AES256 和 GCRY_CIPHER_MODE_CBC 的按位或结果

#define _libssh2_cipher_aes256 \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_AES256, GCRY_CIPHER_MODE_CBC)

# 定义 _libssh2_cipher_aes192 宏，返回 GCRY_CIPHER_AES192 和 GCRY_CIPHER_MODE_CBC 的按位或结果

#define _libssh2_cipher_aes192 \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_AES192, GCRY_CIPHER_MODE_CBC)

# 定义 _libssh2_cipher_aes128 宏，返回 GCRY_CIPHER_AES128 和 GCRY_CIPHER_MODE_CBC 的按位或结果

#define _libssh2_cipher_aes128 \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_AES128, GCRY_CIPHER_MODE_CBC)

# 定义 _libssh2_cipher_blowfish 宏，返回 GCRY_CIPHER_BLOWFISH 和 GCRY_CIPHER_MODE_CBC 的按位或结果

#define _libssh2_cipher_blowfish \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_BLOWFISH, GCRY_CIPHER_MODE_CBC)

# 定义 _libssh2_cipher_arcfour 宏，返回 GCRY_CIPHER_ARCFOUR 和 GCRY_CIPHER_MODE_STREAM 的按位或结果

#define _libssh2_cipher_arcfour \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_ARCFOUR, GCRY_CIPHER_MODE_STREAM)

# 定义 _libssh2_cipher_cast5 宏，返回 GCRY_CIPHER_CAST5 和 GCRY_CIPHER_MODE_CBC 的按位或结果

#define _libssh2_cipher_cast5 \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_CAST5, GCRY_CIPHER_MODE_CBC)

# 定义 _libssh2_cipher_3des 宏，返回 GCRY_CIPHER_3DES 和 GCRY_CIPHER_MODE_CBC 的按位或结果

#define _libssh2_cipher_3des \
  _libssh2_gcry_ciphermode(GCRY_CIPHER_3DES, GCRY_CIPHER_MODE_CBC)

# 定义 _libssh2_cipher_dtor(ctx) 函数，关闭 ctx 对象

#define _libssh2_cipher_dtor(ctx) gcry_cipher_close(*(ctx)

# 定义 _libssh2_bn 结构体为 gcry_mpi 结构体

#define _libssh2_bn struct gcry_mpi

# 定义 _libssh2_bn_ctx 为整型

#define _libssh2_bn_ctx int

# 定义 _libssh2_bn_ctx_new() 函数，返回 0

#define _libssh2_bn_ctx_new() 0

# 定义 _libssh2_bn_ctx_free(bnctx) 函数，不执行任何操作

#define _libssh2_bn_ctx_free(bnctx) ((void)0)

# 定义 _libssh2_bn_init() 函数，创建一个新的 gcry_mpi 对象

#define _libssh2_bn_init() gcry_mpi_new(0)
#define _libssh2_bn_init_from_bin() NULL /* because gcry_mpi_scan() creates a
                                            new bignum */
# 定义一个宏，用于将二进制数据初始化为大数，因为 gcry_mpi_scan() 会创建一个新的大数

#define _libssh2_bn_set_word(bn, val) gcry_mpi_set_ui(bn, val)
# 定义一个宏，用于将一个整数设置为大数的值，使用 gcry_mpi_set_ui() 函数

#define _libssh2_bn_from_bin(bn, len, val)                      \
    gcry_mpi_scan(&((bn)), GCRYMPI_FMT_USG, val, len, NULL)
# 定义一个宏，用于将二进制数据转换为大数，使用 gcry_mpi_scan() 函数

#define _libssh2_bn_to_bin(bn, val)                                     \
    gcry_mpi_print(GCRYMPI_FMT_USG, val, _libssh2_bn_bytes(bn), NULL, bn)
# 定义一个宏，用于将大数转换为二进制数据，使用 gcry_mpi_print() 函数

#define _libssh2_bn_bytes(bn)                                           \
    (gcry_mpi_get_nbits (bn) / 8 +                                      \
     ((gcry_mpi_get_nbits (bn) % 8 == 0) ? 0 : 1))
# 定义一个宏，用于计算大数所占的字节数，根据大数的位数计算

#define _libssh2_bn_bits(bn) gcry_mpi_get_nbits (bn)
# 定义一个宏，用于获取大数的位数，使用 gcry_mpi_get_nbits() 函数

#define _libssh2_bn_free(bn) gcry_mpi_release(bn)
# 定义一个宏，用于释放大数的内存，使用 gcry_mpi_release() 函数

#define _libssh2_dh_ctx struct gcry_mpi *
# 定义一个宏，表示 Diffie-Hellman 上下文，使用 gcry_mpi 结构体指针

#define libssh2_dh_init(dhctx) _libssh2_dh_init(dhctx)
# 定义一个函数，用于初始化 Diffie-Hellman 上下文，调用 _libssh2_dh_init() 函数

#define libssh2_dh_key_pair(dhctx, public, g, p, group_order, bnctx) \
        _libssh2_dh_key_pair(dhctx, public, g, p, group_order)
# 定义一个函数，用于生成 Diffie-Hellman 密钥对，调用 _libssh2_dh_key_pair() 函数

#define libssh2_dh_secret(dhctx, secret, f, p, bnctx) \
        _libssh2_dh_secret(dhctx, secret, f, p)
# 定义一个函数，用于计算 Diffie-Hellman 共享密钥，调用 _libssh2_dh_secret() 函数

#define libssh2_dh_dtor(dhctx) _libssh2_dh_dtor(dhctx)
# 定义一个函数，用于销毁 Diffie-Hellman 上下文，调用 _libssh2_dh_dtor() 函数

extern void _libssh2_dh_init(_libssh2_dh_ctx *dhctx);
# 声明一个函数，用于初始化 Diffie-Hellman 上下文

extern int _libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                                _libssh2_bn *g, _libssh2_bn *p,
                                int group_order);
# 声明一个函数，用于生成 Diffie-Hellman 密钥对

extern int _libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                              _libssh2_bn *f, _libssh2_bn *p);
# 声明一个函数，用于计算 Diffie-Hellman 共享密钥

extern void _libssh2_dh_dtor(_libssh2_dh_ctx *dhctx);
# 声明一个函数，用于销毁 Diffie-Hellman 上下文

#endif /* __LIBSSH2_LIBGCRYPT_H */
# 结束宏定义和函数声明的部分
```