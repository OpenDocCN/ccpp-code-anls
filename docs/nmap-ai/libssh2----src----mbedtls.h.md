# `nmap\libssh2\src\mbedtls.h`

```cpp
#ifndef __LIBSSH2_MBEDTLS_H
#define __LIBSSH2_MBEDTLS_H
/* 定义宏，用于避免重复包含该头文件 */
/* 版权声明，版权所有 */
/* 在源代码和二进制形式中允许进行再发布和使用，需满足以下条件 */
/* 在源代码中重新发布时，需保留上述版权声明、条件列表和以下免责声明 */
/* 在二进制形式中重新发布时，需在文档和/或其他提供的材料中复制上述版权声明、条件列表和以下免责声明 */
/* 不得使用版权所有者的名称或其他贡献者的名称，未经特定事先书面许可不得用于认可或推广从本软件衍生的产品 */
/* 本软件由版权所有者和贡献者按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保 */
/* 无论在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害 */
#include <stdlib.h>
#include <string.h>

#include <mbedtls/platform.h>
#include <mbedtls/md.h>
#include <mbedtls/rsa.h>
#include <mbedtls/bignum.h>
#include <mbedtls/cipher.h>
#ifdef MBEDTLS_ECDH_C
# include <mbedtls/ecdh.h>
#endif
#ifdef MBEDTLS_ECDSA_C
# include <mbedtls/ecdsa.h>
#endif
/* 包含所需的头文件 */
#include <mbedtls/entropy.h>
#include <mbedtls/ctr_drbg.h>
#include <mbedtls/pk.h>
#include <mbedtls/error.h>

/* 定义支持的特性 */
#define LIBSSH2_MD5             1

#define LIBSSH2_HMAC_RIPEMD     1
#define LIBSSH2_HMAC_SHA256     1
#define LIBSSH2_HMAC_SHA512     1

#define LIBSSH2_AES             1
#define LIBSSH2_AES_CTR         1
#define LIBSSH2_BLOWFISH        1
#define LIBSSH2_RC4             1
#define LIBSSH2_CAST            0
#define LIBSSH2_3DES            1

#define LIBSSH2_RSA             1
#define LIBSSH2_DSA             0
#ifdef MBEDTLS_ECDSA_C
# define LIBSSH2_ECDSA          1
#else
# define LIBSSH2_ECDSA          0
#endif
#define LIBSSH2_ED25519         0

#define MD5_DIGEST_LENGTH      16
#define SHA_DIGEST_LENGTH      20
#define SHA256_DIGEST_LENGTH   32
#define SHA384_DIGEST_LENGTH   48
#define SHA512_DIGEST_LENGTH   64

#define EC_MAX_POINT_LEN ((528 * 2 / 8) + 1)


/*******************************************************************/
/*
 * mbedTLS backend: Generic functions
 */

/* 初始化加密库 */
#define libssh2_crypto_init() \
  _libssh2_mbedtls_init()
/* 退出加密库 */
#define libssh2_crypto_exit() \
  _libssh2_mbedtls_free()

/* 生成随机数 */
#define _libssh2_random(buf, len) \
  _libssh2_mbedtls_random(buf, len)

/* 准备 I/O 向量 */
#define libssh2_prepare_iovec(vec, len)  /* Empty. */


/*******************************************************************/
/*
 * mbedTLS backend: HMAC functions
 */

/* 定义 HMAC 上下文 */
#define libssh2_hmac_ctx    mbedtls_md_context_t

/* 初始化 HMAC 上下文 */
#define libssh2_hmac_ctx_init(ctx)
/* 清理 HMAC 上下文 */
#define libssh2_hmac_cleanup(pctx) \
  mbedtls_md_free(pctx)
/* 更新 HMAC 上下文 */
#define libssh2_hmac_update(ctx, data, datalen) \
  mbedtls_md_hmac_update(&ctx, (unsigned char *) data, datalen)
/* 完成 HMAC 计算 */
#define libssh2_hmac_final(ctx, hash) \
  mbedtls_md_hmac_finish(&ctx, hash)

/* 初始化 SHA1 HMAC 上下文 */
#define libssh2_hmac_sha1_init(pctx, key, keylen) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_SHA1, key, keylen)
/* 初始化 MD5 HMAC 上下文 */
#define libssh2_hmac_md5_init(pctx, key, keylen) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_MD5, key, keylen)
#define libssh2_hmac_ripemd160_init(pctx, key, keylen) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_RIPEMD160, key, keylen)
# 定义一个宏，用于初始化 RIPEDM160 HMAC 算法的上下文

#define libssh2_hmac_sha256_init(pctx, key, keylen) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_SHA256, key, keylen)
# 定义一个宏，用于初始化 SHA256 HMAC 算法的上下文

#define libssh2_hmac_sha384_init(pctx, key, keylen) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_SHA384, key, keylen)
# 定义一个宏，用于初始化 SHA384 HMAC 算法的上下文

#define libssh2_hmac_sha512_init(pctx, key, keylen) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_SHA512, key, keylen)
# 定义一个宏，用于初始化 SHA512 HMAC 算法的上下文

/*******************************************************************/
/*
 * mbedTLS backend: SHA1 functions
 */

#define libssh2_sha1_ctx      mbedtls_md_context_t
# 定义一个宏，用于表示 SHA1 上下文的类型

#define libssh2_sha1_init(pctx) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_SHA1, NULL, 0)
# 定义一个宏，用于初始化 SHA1 算法的上下文

#define libssh2_sha1_update(ctx, data, datalen) \
  mbedtls_md_update(&ctx, (unsigned char *) data, datalen)
# 定义一个宏，用于更新 SHA1 算法的上下文

#define libssh2_sha1_final(ctx, hash) \
  _libssh2_mbedtls_hash_final(&ctx, hash)
# 定义一个宏，用于完成 SHA1 算法的计算

#define libssh2_sha1(data, datalen, hash) \
  _libssh2_mbedtls_hash(data, datalen, MBEDTLS_MD_SHA1, hash)
# 定义一个宏，用于计算 SHA1 哈希值

/*******************************************************************/
/*
 * mbedTLS backend: SHA256 functions
 */

#define libssh2_sha256_ctx      mbedtls_md_context_t
# 定义一个宏，用于表示 SHA256 上下文的类型

#define libssh2_sha256_init(pctx) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_SHA256, NULL, 0)
# 定义一个宏，用于初始化 SHA256 算法的上下文

#define libssh2_sha256_update(ctx, data, datalen) \
  mbedtls_md_update(&ctx, (unsigned char *) data, datalen)
# 定义一个宏，用于更新 SHA256 算法的上下文

#define libssh2_sha256_final(ctx, hash) \
  _libssh2_mbedtls_hash_final(&ctx, hash)
# 定义一个宏，用于完成 SHA256 算法的计算

#define libssh2_sha256(data, datalen, hash) \
  _libssh2_mbedtls_hash(data, datalen, MBEDTLS_MD_SHA256, hash)
# 定义一个宏，用于计算 SHA256 哈希值

/*******************************************************************/
/*
 * mbedTLS backend: SHA384 functions
 */

#define libssh2_sha384_ctx      mbedtls_md_context_t
# 定义一个宏，用于表示 SHA384 上下文的类型

#define libssh2_sha384_init(pctx) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_SHA384, NULL, 0)
# 定义一个宏，用于初始化 SHA384 算法的上下文
/*
 * 定义 libssh2_sha384_update 函数，使用 mbedtls_md_update 函数更新 SHA384 上下文
 */
#define libssh2_sha384_update(ctx, data, datalen) \
  mbedtls_md_update(&ctx, (unsigned char *) data, datalen)

/*
 * 定义 libssh2_sha384_final 函数，使用 _libssh2_mbedtls_hash_final 函数完成 SHA384 计算
 */
#define libssh2_sha384_final(ctx, hash) \
  _libssh2_mbedtls_hash_final(&ctx, hash)

/*
 * 定义 libssh2_sha384 函数，使用 _libssh2_mbedtls_hash 函数计算 SHA384 哈希值
 */
#define libssh2_sha384(data, datalen, hash) \
  _libssh2_mbedtls_hash(data, datalen, MBEDTLS_MD_SHA384, hash)


/*******************************************************************/
/*
 * mbedTLS backend: SHA512 functions
 */

/*
 * 定义 libssh2_sha512_ctx 类型，表示 SHA512 上下文
 */
#define libssh2_sha512_ctx      mbedtls_md_context_t

/*
 * 定义 libssh2_sha512_init 函数，使用 _libssh2_mbedtls_hash_init 函数初始化 SHA512 上下文
 */
#define libssh2_sha512_init(pctx) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_SHA512, NULL, 0)

/*
 * 定义 libssh2_sha512_update 函数，使用 mbedtls_md_update 函数更新 SHA512 上下文
 */
#define libssh2_sha512_update(ctx, data, datalen) \
  mbedtls_md_update(&ctx, (unsigned char *) data, datalen)

/*
 * 定义 libssh2_sha512_final 函数，使用 _libssh2_mbedtls_hash_final 函数完成 SHA512 计算
 */
#define libssh2_sha512_final(ctx, hash) \
  _libssh2_mbedtls_hash_final(&ctx, hash)

/*
 * 定义 libssh2_sha512 函数，使用 _libssh2_mbedtls_hash 函数计算 SHA512 哈希值
 */
#define libssh2_sha512(data, datalen, hash) \
  _libssh2_mbedtls_hash(data, datalen, MBEDTLS_MD_SHA512, hash)


/*******************************************************************/
/*
 * mbedTLS backend: MD5 functions
 */

/*
 * 定义 libssh2_md5_ctx 类型，表示 MD5 上下文
 */
#define libssh2_md5_ctx      mbedtls_md_context_t

/*
 * 定义 libssh2_md5_init 函数，使用 _libssh2_mbedtls_hash_init 函数初始化 MD5 上下文
 */
#define libssh2_md5_init(pctx) \
  _libssh2_mbedtls_hash_init(pctx, MBEDTLS_MD_MD5, NULL, 0)

/*
 * 定义 libssh2_md5_update 函数，使用 mbedtls_md_update 函数更新 MD5 上下文
 */
#define libssh2_md5_update(ctx, data, datalen) \
  mbedtls_md_update(&ctx, (unsigned char *) data, datalen)

/*
 * 定义 libssh2_md5_final 函数，使用 _libssh2_mbedtls_hash_final 函数完成 MD5 计算
 */
#define libssh2_md5_final(ctx, hash) \
  _libssh2_mbedtls_hash_final(&ctx, hash)

/*
 * 定义 libssh2_md5 函数，使用 _libssh2_mbedtls_hash 函数计算 MD5 哈希值
 */
#define libssh2_md5(data, datalen, hash) \
  _libssh2_mbedtls_hash(data, datalen, MBEDTLS_MD_MD5, hash)


/*******************************************************************/
/*
 * mbedTLS backend: RSA functions
 */

/*
 * 定义 libssh2_rsa_ctx 类型，表示 RSA 上下文
 */
#define libssh2_rsa_ctx  mbedtls_rsa_context

/*
 * 定义 _libssh2_rsa_new 函数，使用 _libssh2_mbedtls_rsa_new 函数创建 RSA 上下文
 */
#define _libssh2_rsa_new(rsactx, e, e_len, n, n_len, \
                         d, d_len, p, p_len, q, q_len, \
                         e1, e1_len, e2, e2_len, c, c_len) \
  _libssh2_mbedtls_rsa_new(rsactx, e, e_len, n, n_len, \
                          d, d_len, p, p_len, q, q_len, \
                          e1, e1_len, e2, e2_len, c, c_len)
# 定义一个宏，用于创建一个新的 RSA 私钥
#define _libssh2_rsa_new_private(rsactx, s, filename, passphrase) \
  _libssh2_mbedtls_rsa_new_private(rsactx, s, filename, passphrase)

# 定义一个宏，用于从内存中创建一个新的 RSA 私钥
#define _libssh2_rsa_new_private_frommemory(rsactx, s, filedata, \
                                            filedata_len, passphrase) \
  _libssh2_mbedtls_rsa_new_private_frommemory(rsactx, s, filedata, \
                                             filedata_len, passphrase)

# 定义一个宏，用于对 SHA1 哈希进行 RSA 签名
#define _libssh2_rsa_sha1_sign(s, rsactx, hash, hash_len, sig, sig_len) \
  _libssh2_mbedtls_rsa_sha1_sign(s, rsactx, hash, hash_len, sig, sig_len)

# 定义一个宏，用于对 SHA1 哈希进行 RSA 验证
#define _libssh2_rsa_sha1_verify(rsactx, sig, sig_len, m, m_len) \
  _libssh2_mbedtls_rsa_sha1_verify(rsactx, sig, sig_len, m, m_len)

# 定义一个宏，用于释放 RSA 上下文
#define _libssh2_rsa_free(rsactx) \
  _libssh2_mbedtls_rsa_free(rsactx)

/*******************************************************************/
/*
 * mbedTLS backend: ECDSA structures
 */

# 如果支持 ECDSA
#if LIBSSH2_ECDSA

# 定义一个枚举类型，表示椭圆曲线类型
typedef enum {
#ifdef MBEDTLS_ECP_DP_SECP256R1_ENABLED
    LIBSSH2_EC_CURVE_NISTP256 = MBEDTLS_ECP_DP_SECP256R1,
#else
    LIBSSH2_EC_CURVE_NISTP256 = MBEDTLS_ECP_DP_NONE,
#endif
#ifdef MBEDTLS_ECP_DP_SECP384R1_ENABLED
    LIBSSH2_EC_CURVE_NISTP384 = MBEDTLS_ECP_DP_SECP384R1,
#else
    LIBSSH2_EC_CURVE_NISTP384 = MBEDTLS_ECP_DP_NONE,
#endif
#ifdef MBEDTLS_ECP_DP_SECP521R1_ENABLED
    LIBSSH2_EC_CURVE_NISTP521 = MBEDTLS_ECP_DP_SECP521R1
#else
    LIBSSH2_EC_CURVE_NISTP521 = MBEDTLS_ECP_DP_NONE,
#endif
} libssh2_curve_type;

# 定义一个结构体，表示 ECDSA 密钥对
# define _libssh2_ec_key mbedtls_ecp_keypair
#else
# 如果不支持 ECDSA，则将 _libssh2_ec_key 定义为 void
# define _libssh2_ec_key void
#endif /* LIBSSH2_ECDSA */

/*******************************************************************/
/*
 * mbedTLS backend: ECDSA functions
 */

# 如果支持 ECDSA
#if LIBSSH2_ECDSA

# 定义一个类型别名，表示 ECDSA 上下文
#define libssh2_ecdsa_ctx mbedtls_ecdsa_context

# 定义一个宏，用于创建一个新的 ECDSA 密钥对
#define _libssh2_ecdsa_create_key(session, privkey, pubkey_octal, \
                                  pubkey_octal_len, curve) \
  _libssh2_mbedtls_ecdsa_create_key(session, privkey, pubkey_octal, \
                                    pubkey_octal_len, curve)
#define _libssh2_ecdsa_curve_name_with_octal_new(ctx, k, k_len, curve) \
  _libssh2_mbedtls_ecdsa_curve_name_with_octal_new(ctx, k, k_len, curve)
# 定义一个宏，将参数传递给_mbedtls_ecdsa_curve_name_with_octal_new函数

#define _libssh2_ecdh_gen_k(k, privkey, server_pubkey, server_pubkey_len) \
  _libssh2_mbedtls_ecdh_gen_k(k, privkey, server_pubkey, server_pubkey_len)
# 定义一个宏，将参数传递给_mbedtls_ecdh_gen_k函数

#define _libssh2_ecdsa_verify(ctx, r, r_len, s, s_len, m, m_len) \
  _libssh2_mbedtls_ecdsa_verify(ctx, r, r_len, s, s_len, m, m_len)
# 定义一个宏，将参数传递给_mbedtls_ecdsa_verify函数

#define _libssh2_ecdsa_new_private(ctx, session, filename, passphrase) \
  _libssh2_mbedtls_ecdsa_new_private(ctx, session, filename, passphrase)
# 定义一个宏，将参数传递给_mbedtls_ecdsa_new_private函数

#define _libssh2_ecdsa_new_private_frommemory(ctx, session, filedata, \
                                              filedata_len, passphrase) \
  _libssh2_mbedtls_ecdsa_new_private_frommemory(ctx, session, filedata, \
                                                filedata_len, passphrase)
# 定义一个宏，将参数传递给_mbedtls_ecdsa_new_private_frommemory函数

#define _libssh2_ecdsa_sign(session, ctx, hash, hash_len, sign, sign_len) \
  _libssh2_mbedtls_ecdsa_sign(session, ctx, hash, hash_len, sign, sign_len)
# 定义一个宏，将参数传递给_mbedtls_ecdsa_sign函数

#define _libssh2_ecdsa_get_curve_type(ctx) \
  _libssh2_mbedtls_ecdsa_get_curve_type(ctx)
# 定义一个宏，将参数传递给_mbedtls_ecdsa_get_curve_type函数

#define _libssh2_ecdsa_free(ctx) \
  _libssh2_mbedtls_ecdsa_free(ctx)
# 定义一个宏，将参数传递给_mbedtls_ecdsa_free函数

#endif /* LIBSSH2_ECDSA */
# 结束宏定义部分

/*******************************************************************/
/*
 * mbedTLS backend: Key functions
 */

#define _libssh2_pub_priv_keyfile(s, m, m_len, p, p_len, pk, pw) \
  _libssh2_mbedtls_pub_priv_keyfile(s, m, m_len, p, p_len, pk, pw)
# 定义一个宏，将参数传递给_mbedtls_pub_priv_keyfile函数

#define _libssh2_pub_priv_keyfilememory(s, m, m_len, p, p_len, \
                                                     pk, pk_len, pw) \
  _libssh2_mbedtls_pub_priv_keyfilememory(s, m, m_len, p, p_len, \
                                                      pk, pk_len, pw)
# 定义一个宏，将参数传递给_mbedtls_pub_priv_keyfilememory函数

/*******************************************************************/
/*
 * mbedTLS backend: Cipher Context structure
 */

#define _libssh2_cipher_ctx         mbedtls_cipher_context_t
# 定义一个宏，将_mbedtls_cipher_context_t赋值给_libssh2_cipher_ctx

#define _libssh2_cipher_type(algo)  mbedtls_cipher_type_t algo
# 定义一个宏，将mbedtls_cipher_type_t algo赋值给_libssh2_cipher_type
#define _libssh2_cipher_aes256ctr MBEDTLS_CIPHER_AES_256_CTR
#define _libssh2_cipher_aes192ctr MBEDTLS_CIPHER_AES_192_CTR
#define _libssh2_cipher_aes128ctr MBEDTLS_CIPHER_AES_128_CTR
#define _libssh2_cipher_aes256    MBEDTLS_CIPHER_AES_256_CBC
#define _libssh2_cipher_aes192    MBEDTLS_CIPHER_AES_192_CBC
#define _libssh2_cipher_aes128    MBEDTLS_CIPHER_AES_128_CBC
#define _libssh2_cipher_blowfish  MBEDTLS_CIPHER_BLOWFISH_CBC
#define _libssh2_cipher_arcfour   MBEDTLS_CIPHER_ARC4_128
#define _libssh2_cipher_cast5     MBEDTLS_CIPHER_NULL
#define _libssh2_cipher_3des      MBEDTLS_CIPHER_DES_EDE3_CBC
// 定义了一系列 libssh2 的加密算法和对应的 mbedTLS 加密算法的映射关系


/*******************************************************************/
/*
 * mbedTLS backend: Cipher functions
 */

#define _libssh2_cipher_init(ctx, type, iv, secret, encrypt) \
  _libssh2_mbedtls_cipher_init(ctx, type, iv, secret, encrypt)
#define _libssh2_cipher_crypt(ctx, type, encrypt, block, blocklen) \
  _libssh2_mbedtls_cipher_crypt(ctx, type, encrypt, block, blocklen)
#define _libssh2_cipher_dtor(ctx) \
  _libssh2_mbedtls_cipher_dtor(ctx)
// 定义了 libssh2 的加密函数在 mbedTLS 中的对应函数


/*******************************************************************/
/*
 * mbedTLS backend: BigNumber Support
 */

#define _libssh2_bn_ctx int /* not used */
#define _libssh2_bn_ctx_new() 0 /* not used */
#define _libssh2_bn_ctx_free(bnctx) ((void)0) /* not used */

#define _libssh2_bn mbedtls_mpi

#define _libssh2_bn_init() \
  _libssh2_mbedtls_bignum_init()
#define _libssh2_bn_init_from_bin() \
  _libssh2_mbedtls_bignum_init()
#define _libssh2_bn_set_word(bn, word) \
  mbedtls_mpi_lset(bn, word)
#define _libssh2_bn_from_bin(bn, len, bin) \
  mbedtls_mpi_read_binary(bn, bin, len)
#define _libssh2_bn_to_bin(bn, bin) \
  mbedtls_mpi_write_binary(bn, bin, mbedtls_mpi_size(bn))
#define _libssh2_bn_bytes(bn) \
  mbedtls_mpi_size(bn)
#define _libssh2_bn_bits(bn) \
  mbedtls_mpi_bitlen(bn)
#define _libssh2_bn_free(bn) \
  _libssh2_mbedtls_bignum_free(bn)
// 定义了 libssh2 的大数支持函数在 mbedTLS 中的对应函数
/*
 * mbedTLS backend: Diffie-Hellman support.
 */

// 定义 libssh2 DH 上下文为 mbedtls 的大整数指针
#define _libssh2_dh_ctx mbedtls_mpi *
// 初始化 DH 上下文
#define libssh2_dh_init(dhctx) _libssh2_dh_init(dhctx)
// 生成 DH 密钥对
#define libssh2_dh_key_pair(dhctx, public, g, p, group_order, bnctx) \
        _libssh2_dh_key_pair(dhctx, public, g, p, group_order)
// 计算 DH 共享密钥
#define libssh2_dh_secret(dhctx, secret, f, p, bnctx) \
        _libssh2_dh_secret(dhctx, secret, f, p)
// 销毁 DH 上下文
#define libssh2_dh_dtor(dhctx) _libssh2_dh_dtor(dhctx)


/*******************************************************************/
/*
 * mbedTLS backend: forward declarations
 */

// 初始化 mbedTLS
void
_libssh2_mbedtls_init(void);

// 释放 mbedTLS 资源
void
_libssh2_mbedtls_free(void);

// 生成随机数
int
_libssh2_mbedtls_random(unsigned char *buf, int len);

// 初始化加密算法上下文
int
_libssh2_mbedtls_cipher_init(_libssh2_cipher_ctx *ctx,
                            _libssh2_cipher_type(type),
                            unsigned char *iv,
                            unsigned char *secret,
                            int encrypt);

// 加密/解密数据
int
_libssh2_mbedtls_cipher_crypt(_libssh2_cipher_ctx *ctx,
                             _libssh2_cipher_type(type),
                             int encrypt,
                             unsigned char *block,
                             size_t blocklen);

// 销毁加密算法上下文
void
_libssh2_mbedtls_cipher_dtor(_libssh2_cipher_ctx *ctx);

// 初始化哈希算法上下文
int
_libssh2_mbedtls_hash_init(mbedtls_md_context_t *ctx,
                          mbedtls_md_type_t mdtype,
                          const unsigned char *key, unsigned long keylen);

// 计算哈希值
int
_libssh2_mbedtls_hash_final(mbedtls_md_context_t *ctx, unsigned char *hash);

// 计算哈希值
int
_libssh2_mbedtls_hash(const unsigned char *data, unsigned long datalen,
                      mbedtls_md_type_t mdtype, unsigned char *hash);

// 初始化大整数
_libssh2_bn *
_libssh2_mbedtls_bignum_init(void);

// 释放大整数资源
void
_libssh2_mbedtls_bignum_free(_libssh2_bn *bn);
# 创建一个新的 RSA 密钥对，并将其存储在 rsa 指针所指向的位置
# 参数包括公钥和私钥的各种数据，如 e、n、d、p、q 等
_libssh2_mbedtls_rsa_new(libssh2_rsa_ctx **rsa,
                        const unsigned char *edata,
                        unsigned long elen,
                        const unsigned char *ndata,
                        unsigned long nlen,
                        const unsigned char *ddata,
                        unsigned long dlen,
                        const unsigned char *pdata,
                        unsigned long plen,
                        const unsigned char *qdata,
                        unsigned long qlen,
                        const unsigned char *e1data,
                        unsigned long e1len,
                        const unsigned char *e2data,
                        unsigned long e2len,
                        const unsigned char *coeffdata,
                        unsigned long coefflen);

# 从文件中读取私钥数据，创建一个新的 RSA 密钥对
# 参数包括存储私钥数据的文件名和可能的密码
_libssh2_mbedtls_rsa_new_private(libssh2_rsa_ctx **rsa,
                                LIBSSH2_SESSION *session,
                                const char *filename,
                                const unsigned char *passphrase);

# 从内存中读取私钥数据，创建一个新的 RSA 密钥对
# 参数包括存储私钥数据的内存块和长度，以及可能的密码
_libssh2_mbedtls_rsa_new_private_frommemory(libssh2_rsa_ctx **rsa,
                                           LIBSSH2_SESSION *session,
                                           const char *filedata,
                                           size_t filedata_len,
                                           unsigned const char *passphrase);

# 使用 RSA 密钥对对数据进行 SHA1 签名验证
# 参数包括 RSA 上下文、签名数据、签名数据长度、原始数据和原始数据长度
_libssh2_mbedtls_rsa_sha1_verify(libssh2_rsa_ctx *rsa,
                                const unsigned char *sig,
                                unsigned long sig_len,
                                const unsigned char *m,
                                unsigned long m_len);
# 使用 mbedtls 对 RSA-SHA1 签名进行加密
_libssh2_mbedtls_rsa_sha1_sign(LIBSSH2_SESSION *session,
                              libssh2_rsa_ctx *rsa,
                              const unsigned char *hash,
                              size_t hash_len,
                              unsigned char **signature,
                              size_t *signature_len);
# 释放 mbedtls 中 RSA 上下文的内存
void
_libssh2_mbedtls_rsa_free(libssh2_rsa_ctx *rsa);

# 从公私钥文件中获取公私钥数据
int
_libssh2_mbedtls_pub_priv_keyfile(LIBSSH2_SESSION *session,
                                 unsigned char **method,
                                 size_t *method_len,
                                 unsigned char **pubkeydata,
                                 size_t *pubkeydata_len,
                                 const char *privatekey,
                                 const char *passphrase);
# 从内存中获取公私钥文件数据
int
_libssh2_mbedtls_pub_priv_keyfilememory(LIBSSH2_SESSION *session,
                                       unsigned char **method,
                                       size_t *method_len,
                                       unsigned char **pubkeydata,
                                       size_t *pubkeydata_len,
                                       const char *privatekeydata,
                                       size_t privatekeydata_len,
                                       const char *passphrase);
# 如果支持 ECDSA，使用 mbedtls 创建 ECDSA 密钥
#if LIBSSH2_ECDSA
int
_libssh2_mbedtls_ecdsa_create_key(LIBSSH2_SESSION *session,
                                  _libssh2_ec_key **privkey,
                                  unsigned char **pubkey_octal,
                                  size_t *pubkey_octal_len,
                                  libssh2_curve_type curve);
# 使用 mbedtls 创建 ECDSA 曲线名称与八进制新密钥
int
_libssh2_mbedtls_ecdsa_curve_name_with_octal_new(libssh2_ecdsa_ctx **ctx,
                                                 const unsigned char *k,
                                                 size_t k_len,
                                                 libssh2_curve_type curve);
# 生成 ECDH 密钥
_libssh2_mbedtls_ecdh_gen_k(_libssh2_bn **k,
                            _libssh2_ec_key *privkey,
                            const unsigned char *server_pubkey,
                            size_t server_pubkey_len);
# 验证 ECDSA 签名
int
_libssh2_mbedtls_ecdsa_verify(libssh2_ecdsa_ctx *ctx,
                              const unsigned char *r, size_t r_len,
                              const unsigned char *s, size_t s_len,
                              const unsigned char *m, size_t m_len);
# 创建新的 ECDSA 私钥
int
_libssh2_mbedtls_ecdsa_new_private(libssh2_ecdsa_ctx **ctx,
                                  LIBSSH2_SESSION *session,
                                  const char *filename,
                                  const unsigned char *passphrase);
# 从内存中创建新的 ECDSA 私钥
int
_libssh2_mbedtls_ecdsa_new_private_frommemory(libssh2_ecdsa_ctx **ctx,
                                              LIBSSH2_SESSION *session,
                                              const char *filedata,
                                              size_t filedata_len,
                                              const unsigned char *passphrase);
# 对数据进行 ECDSA 签名
int
_libssh2_mbedtls_ecdsa_sign(LIBSSH2_SESSION *session,
                            libssh2_ecdsa_ctx *ctx,
                            const unsigned char *hash,
                            unsigned long hash_len,
                            unsigned char **signature,
                            size_t *signature_len);
# 获取 ECDSA 密钥的曲线类型
libssh2_curve_type
_libssh2_mbedtls_ecdsa_key_get_curve_type(libssh2_ecdsa_ctx *ctx);
# 根据名称获取 ECDSA 曲线类型
int
_libssh2_mbedtls_ecdsa_curve_type_from_name(const char *name,
                                            libssh2_curve_type *type);
# 释放 ECDSA 上下文
void
_libssh2_mbedtls_ecdsa_free(libssh2_ecdsa_ctx *ctx);
# 初始化 DH 上下文
extern void
_libssh2_dh_init(_libssh2_dh_ctx *dhctx);
# 生成 DH 密钥对
extern int
_libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                    _libssh2_bn *g, _libssh2_bn *p, int group_order);
# ...
# 定义一个函数_libssh2_dh_secret，接受一个_libssh2_dh_ctx类型的参数dhctx，以及三个_libssh2_bn类型的参数f、p和secret
_libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                  _libssh2_bn *f, _libssh2_bn *p);
# 声明一个函数_libssh2_dh_dtor，接受一个_libssh2_dh_ctx类型的参数dhctx，无返回值
extern void
_libssh2_dh_dtor(_libssh2_dh_ctx *dhctx);
# 结束if条件编译，结束头文件的声明
#endif /* __LIBSSH2_MBEDTLS_H */
```