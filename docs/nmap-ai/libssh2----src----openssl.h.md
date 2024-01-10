# `nmap\libssh2\src\openssl.h`

```
#ifndef __LIBSSH2_OPENSSL_H
#define __LIBSSH2_OPENSSL_H
/* 定义条件编译，如果__LIBSSH2_OPENSSL_H未定义，则定义__LIBSSH2_OPENSSL_H */
/* 包含版权声明和许可条件 */
/* 作者信息 */
/* 允许在源码和二进制形式下重新分发和使用，需满足以下条件 */
/* 在源码形式下重新分发必须保留版权声明、条件列表和以下免责声明 */
/* 在二进制形式下重新分发必须在文档和/或其他提供的材料中重现版权声明、条件列表和以下免责声明 */
/* 不得使用版权所有者的名称或任何其他贡献者的名称来认可或推广从本软件派生的产品，除非事先书面许可 */
/* 免责声明 */
/* 如果出现任何形式的损害，版权所有者或贡献者均不承担任何责任，包括但不限于直接、间接、附带、特殊、惩罚性或后果性的损害，包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他方式）引起的，即使已被告知可能发生此类损害 */
/* 包含 OpenSSL 配置文件 */
#include <openssl/opensslconf.h>
/* 包含 SHA 加密算法相关的头文件 */
#include <openssl/sha.h>
/* 包含 RSA 加密算法相关的头文件 */
#include <openssl/rsa.h>
/* 如果未定义 OPENSSL_NO_ENGINE，则包含加密引擎相关的头文件 */
#ifndef OPENSSL_NO_ENGINE
#include <openssl/engine.h>
#endif
/* 如果未定义 OPENSSL_NO_DSA，则包含 DSA 加密算法相关的头文件 */
#ifndef OPENSSL_NO_DSA
#include <openssl/dsa.h>
#endif
/* 如果未定义 OPENSSL_NO_MD5，则包含 MD5 加密算法相关的头文件 */
#ifndef OPENSSL_NO_MD5
#include <openssl/md5.h>
#endif
#include <openssl/evp.h>  // 包含 OpenSSL 的 EVP 头文件
#include <openssl/hmac.h>  // 包含 OpenSSL 的 HMAC 头文件
#include <openssl/bn.h>    // 包含 OpenSSL 的 BN 头文件
#include <openssl/pem.h>   // 包含 OpenSSL 的 PEM 头文件
#include <openssl/rand.h>  // 包含 OpenSSL 的 RAND 头文件

#if OPENSSL_VERSION_NUMBER >= 0x10100000L && \
    !defined(LIBRESSL_VERSION_NUMBER)
# define HAVE_OPAQUE_STRUCTS 1  // 如果 OpenSSL 版本号大于等于 1.1.0 并且不是 LibreSSL，则定义 HAVE_OPAQUE_STRUCTS 为 1
#endif

#ifdef OPENSSL_NO_RSA
# define LIBSSH2_RSA 0  // 如果 OpenSSL 不支持 RSA，则定义 LIBSSH2_RSA 为 0
#else
# define LIBSSH2_RSA 1  // 否则定义 LIBSSH2_RSA 为 1
#endif

#ifdef OPENSSL_NO_DSA
# define LIBSSH2_DSA 0  // 如果 OpenSSL 不支持 DSA，则定义 LIBSSH2_DSA 为 0
#else
# define LIBSSH2_DSA 1  // 否则定义 LIBSSH2_DSA 为 1
#endif

#ifdef OPENSSL_NO_ECDSA
# define LIBSSH2_ECDSA 0  // 如果 OpenSSL 不支持 ECDSA，则定义 LIBSSH2_ECDSA 为 0
#else
# define LIBSSH2_ECDSA 1  // 否则定义 LIBSSH2_ECDSA 为 1
#endif

#if OPENSSL_VERSION_NUMBER >= 0x10101000L && \
!defined(LIBRESSL_VERSION_NUMBER)
# define LIBSSH2_ED25519 1  // 如果 OpenSSL 版本号大于等于 1.1.1 并且不是 LibreSSL，则定义 LIBSSH2_ED25519 为 1
#else
# define LIBSSH2_ED25519 0  // 否则定义 LIBSSH2_ED25519 为 0
#endif

#ifdef OPENSSL_NO_MD5
# define LIBSSH2_MD5 0  // 如果 OpenSSL 不支持 MD5，则定义 LIBSSH2_MD5 为 0
#else
# define LIBSSH2_MD5 1  // 否则定义 LIBSSH2_MD5 为 1
#endif

#ifdef OPENSSL_NO_RIPEMD
# define LIBSSH2_HMAC_RIPEMD 0  // 如果 OpenSSL 不支持 RIPEMD，则定义 LIBSSH2_HMAC_RIPEMD 为 0
#else
# define LIBSSH2_HMAC_RIPEMD 1  // 否则定义 LIBSSH2_HMAC_RIPEMD 为 1
#endif

#define LIBSSH2_HMAC_SHA256 1  // 定义 LIBSSH2_HMAC_SHA256 为 1
#define LIBSSH2_HMAC_SHA512 1  // 定义 LIBSSH2_HMAC_SHA512 为 1

#if OPENSSL_VERSION_NUMBER >= 0x00907000L && !defined(OPENSSL_NO_AES)
# define LIBSSH2_AES_CTR 1  // 如果 OpenSSL 版本号大于等于 0.9.7 并且不禁用 AES，则定义 LIBSSH2_AES_CTR 为 1
# define LIBSSH2_AES 1       // 同时定义 LIBSSH2_AES 为 1
#else
# define LIBSSH2_AES_CTR 0  // 否则定义 LIBSSH2_AES_CTR 为 0
# define LIBSSH2_AES 0       // 同时定义 LIBSSH2_AES 为 0
#endif

#ifdef OPENSSL_NO_BF
# define LIBSSH2_BLOWFISH 0  // 如果 OpenSSL 不支持 BLOWFISH，则定义 LIBSSH2_BLOWFISH 为 0
#else
# define LIBSSH2_BLOWFISH 1  // 否则定义 LIBSSH2_BLOWFISH 为 1
#endif

#ifdef OPENSSL_NO_RC4
# define LIBSSH2_RC4 0  // 如果 OpenSSL 不支持 RC4，则定义 LIBSSH2_RC4 为 0
#else
# define LIBSSH2_RC4 1  // 否则定义 LIBSSH2_RC4 为 1
#endif

#ifdef OPENSSL_NO_CAST
# define LIBSSH2_CAST 0  // 如果 OpenSSL 不支持 CAST，则定义 LIBSSH2_CAST 为 0
#else
# define LIBSSH2_CAST 1  // 否则定义 LIBSSH2_CAST 为 1
#endif

#ifdef OPENSSL_NO_DES
# define LIBSSH2_3DES 0  // 如果 OpenSSL 不支持 DES，则定义 LIBSSH2_3DES 为 0
#else
# define LIBSSH2_3DES 1  // 否则定义 LIBSSH2_3DES 为 1
#endif

#define EC_MAX_POINT_LEN ((528 * 2 / 8) + 1)  // 定义 EC_MAX_POINT_LEN 为 (528 * 2 / 8) + 1

#define _libssh2_random(buf, len) (RAND_bytes((buf), (len)) == 1 ? 0 : -1)  // 定义 _libssh2_random 函数，使用 RAND_bytes 生成随机数

#define libssh2_prepare_iovec(vec, len)  /* Empty. */  // 定义 libssh2_prepare_iovec 宏为空

#ifdef HAVE_OPAQUE_STRUCTS
#define libssh2_sha1_ctx EVP_MD_CTX *  // 如果有不透明结构体，则定义 libssh2_sha1_ctx 为 EVP_MD_CTX 指针类型
#else
#define libssh2_sha1_ctx EVP_MD_CTX  // 否则定义 libssh2_sha1_ctx 为 EVP_MD_CTX 类型
#endif

/* returns 0 in case of failure */
int _libssh2_sha1_init(libssh2_sha1_ctx *ctx);  // 定义 _libssh2_sha1_init 函数
#define libssh2_sha1_init(x) _libssh2_sha1_init(x)  // 定义 libssh2_sha1_init 宏

#ifdef HAVE_OPAQUE_STRUCTS
#define libssh2_sha1_update(ctx, data, len) EVP_DigestUpdate(ctx, data, len)  // 如果有不透明结构体，则定义 libssh2_sha1_update 宏为 EVP_DigestUpdate 函数
#ifdef HAVE_OPAQUE_STRUCTS
// 如果存在不透明结构体，则定义 libssh2_sha1_final 函数
#define libssh2_sha1_final(ctx, out) do { \
                                         // 调用 EVP_DigestFinal 函数，将结果输出到 out 中
                                         EVP_DigestFinal(ctx, out, NULL); \
                                         // 释放 EVP_MD_CTX 结构体
                                         EVP_MD_CTX_free(ctx); \
                                     } while(0)
#else
// 如果不存在不透明结构体，则定义 libssh2_sha1_update 和 libssh2_sha1_final 函数
#define libssh2_sha1_update(ctx, data, len) EVP_DigestUpdate(&(ctx), data, len)
#define libssh2_sha1_final(ctx, out) EVP_DigestFinal(&(ctx), out, NULL)
#endif
// 定义 _libssh2_sha1 函数，计算 SHA1 哈希值
int _libssh2_sha1(const unsigned char *message, unsigned long len,
                  unsigned char *out);
// 定义 libssh2_sha1 宏，调用 _libssh2_sha1 函数
#define libssh2_sha1(x,y,z) _libssh2_sha1(x,y,z)

#ifdef HAVE_OPAQUE_STRUCTS
// 如果存在不透明结构体，则定义 libssh2_sha256_ctx 类型为 EVP_MD_CTX 指针
#define libssh2_sha256_ctx EVP_MD_CTX *
#else
// 如果不存在不透明结构体，则定义 libssh2_sha256_ctx 类型为 EVP_MD_CTX 结构体
#define libssh2_sha256_ctx EVP_MD_CTX
#endif
// 定义 _libssh2_sha256_init 函数，初始化 SHA256 哈希计算上下文
int _libssh2_sha256_init(libssh2_sha256_ctx *ctx);
// 定义 libssh2_sha256_init 宏，调用 _libssh2_sha256_init 函数
#define libssh2_sha256_init(x) _libssh2_sha256_init(x)
#ifdef HAVE_OPAQUE_STRUCTS
// 如果存在不透明结构体，则定义 libssh2_sha256_update 和 libssh2_sha256_final 函数
#define libssh2_sha256_update(ctx, data, len) EVP_DigestUpdate(ctx, data, len)
#define libssh2_sha256_final(ctx, out) do { \
                                           // 调用 EVP_DigestFinal 函数，将结果输出到 out 中
                                           EVP_DigestFinal(ctx, out, NULL); \
                                           // 释放 EVP_MD_CTX 结构体
                                           EVP_MD_CTX_free(ctx); \
                                       } while(0)
#else
// 如果不存在不透明结构体，则定义 libssh2_sha256_update 和 libssh2_sha256_final 函数
#define libssh2_sha256_update(ctx, data, len) \
    EVP_DigestUpdate(&(ctx), data, len)
#define libssh2_sha256_final(ctx, out) EVP_DigestFinal(&(ctx), out, NULL)
#endif
// 定义 _libssh2_sha256 函数，计算 SHA256 哈希值
int _libssh2_sha256(const unsigned char *message, unsigned long len,
                  unsigned char *out);
// 定义 libssh2_sha256 宏，调用 _libssh2_sha256 函数
#define libssh2_sha256(x,y,z) _libssh2_sha256(x,y,z)

#ifdef HAVE_OPAQUE_STRUCTS
// 如果存在不透明结构体，则定义 libssh2_sha384_ctx 类型为 EVP_MD_CTX 指针
#define libssh2_sha384_ctx EVP_MD_CTX *
#else
// 如果不存在不透明结构体，则定义 libssh2_sha384_ctx 类型为 EVP_MD_CTX 结构体
#define libssh2_sha384_ctx EVP_MD_CTX
#endif
// 定义 _libssh2_sha384_init 函数，初始化 SHA384 哈希计算上下文
int _libssh2_sha384_init(libssh2_sha384_ctx *ctx);
// 定义 libssh2_sha384_init 宏，调用 _libssh2_sha384_init 函数
#define libssh2_sha384_init(x) _libssh2_sha384_init(x)
#ifdef HAVE_OPAQUE_STRUCTS
// 如果存在不透明结构体，则定义 libssh2_sha384_update 函数
#define libssh2_sha384_update(ctx, data, len) EVP_DigestUpdate(ctx, data, len)
#ifdef HAVE_OPAQUE_STRUCTS
#define libssh2_sha384_final(ctx, out) do { \  // 定义宏，如果存在不透明结构，则调用EVP_DigestFinal函数和EVP_MD_CTX_free函数
                                            EVP_DigestFinal(ctx, out, NULL); \
                                            EVP_MD_CTX_free(ctx); \
                                       } while(0)
#else
#define libssh2_sha384_update(ctx, data, len)   \  // 定义宏，如果不存在不透明结构，则调用EVP_DigestUpdate函数
    EVP_DigestUpdate(&(ctx), data, len)
#define libssh2_sha384_final(ctx, out) EVP_DigestFinal(&(ctx), out, NULL)  // 定义宏，如果不存在不透明结构，则调用EVP_DigestFinal函数
#endif
int _libssh2_sha384(const unsigned char *message, unsigned long len,  // 定义_libssh2_sha384函数，接受消息、长度和输出参数
                    unsigned char *out);
#define libssh2_sha384(x,y,z) _libssh2_sha384(x,y,z)  // 定义libssh2_sha384宏，调用_libssh2_sha384函数

#ifdef HAVE_OPAQUE_STRUCTS
#define libssh2_sha512_ctx EVP_MD_CTX *  // 如果存在不透明结构，则定义libssh2_sha512_ctx为EVP_MD_CTX指针类型
#else
#define libssh2_sha512_ctx EVP_MD_CTX  // 如果不存在不透明结构，则定义libssh2_sha512_ctx为EVP_MD_CTX类型
#endif

/* returns 0 in case of failure */
int _libssh2_sha512_init(libssh2_sha512_ctx *ctx);  // 定义_libssh2_sha512_init函数，接受libssh2_sha512_ctx指针参数
#define libssh2_sha512_init(x) _libssh2_sha512_init(x)  // 定义libssh2_sha512_init宏，调用_libssh2_sha512_init函数
#ifdef HAVE_OPAQUE_STRUCTS
#define libssh2_sha512_update(ctx, data, len) EVP_DigestUpdate(ctx, data, len)  // 如果存在不透明结构，则调用EVP_DigestUpdate函数
#define libssh2_sha512_final(ctx, out) do { \  // 如果存在不透明结构，则调用EVP_DigestFinal函数和EVP_MD_CTX_free函数
                                            EVP_DigestFinal(ctx, out, NULL); \
                                            EVP_MD_CTX_free(ctx); \
                                       } while(0)
#else
#define libssh2_sha512_update(ctx, data, len)   \  // 如果不存在不透明结构，则调用EVP_DigestUpdate函数
    EVP_DigestUpdate(&(ctx), data, len)
#define libssh2_sha512_final(ctx, out) EVP_DigestFinal(&(ctx), out, NULL)  // 如果不存在不透明结构，则调用EVP_DigestFinal函数
#endif
int _libssh2_sha512(const unsigned char *message, unsigned long len,  // 定义_libssh2_sha512函数，接受消息、长度和输出参数
                    unsigned char *out);
#define libssh2_sha512(x,y,z) _libssh2_sha512(x,y,z)  // 定义libssh2_sha512宏，调用_libssh2_sha512函数

#ifdef HAVE_OPAQUE_STRUCTS
#define libssh2_md5_ctx EVP_MD_CTX *  // 如果存在不透明结构，则定义libssh2_md5_ctx为EVP_MD_CTX指针类型
#else
#define libssh2_md5_ctx EVP_MD_CTX  // 如果不存在不透明结构，则定义libssh2_md5_ctx为EVP_MD_CTX类型
#endif

/* returns 0 in case of failure */
int _libssh2_md5_init(libssh2_md5_ctx *ctx);  // 定义_libssh2_md5_init函数，接受libssh2_md5_ctx指针参数
#define libssh2_md5_init(x) _libssh2_md5_init(x)  // 定义libssh2_md5_init宏，调用_libssh2_md5_init函数
#ifdef HAVE_OPAQUE_STRUCTS
#define libssh2_md5_update(ctx, data, len) EVP_DigestUpdate(ctx, data, len)  // 如果存在不透明结构，则调用EVP_DigestUpdate函数
#ifdef HAVE_OPAQUE_STRUCTS
// 如果支持不透明结构体，则定义 libssh2_hmac_ctx 为 HMAC_CTX 指针
#define libssh2_hmac_ctx HMAC_CTX *
// 初始化 libssh2_hmac_ctx
#define libssh2_hmac_ctx_init(ctx) ctx = HMAC_CTX_new()
// 使用 SHA1 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_sha1_init(ctx, key, keylen) \
  HMAC_Init_ex(*(ctx), key, keylen, EVP_sha1(), NULL)
// 使用 MD5 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_md5_init(ctx, key, keylen) \
  HMAC_Init_ex(*(ctx), key, keylen, EVP_md5(), NULL)
// 使用 RIPEMD160 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_ripemd160_init(ctx, key, keylen) \
  HMAC_Init_ex(*(ctx), key, keylen, EVP_ripemd160(), NULL)
// 使用 SHA256 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_sha256_init(ctx, key, keylen) \
  HMAC_Init_ex(*(ctx), key, keylen, EVP_sha256(), NULL)
// 使用 SHA512 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_sha512_init(ctx, key, keylen) \
  HMAC_Init_ex(*(ctx), key, keylen, EVP_sha512(), NULL)
// 更新 libssh2_hmac_ctx 的数据
#define libssh2_hmac_update(ctx, data, datalen) \
  HMAC_Update(ctx, data, datalen)
// 完成 libssh2_hmac_ctx 的计算
#define libssh2_hmac_final(ctx, data) HMAC_Final(ctx, data, NULL)
// 清理 libssh2_hmac_ctx
#define libssh2_hmac_cleanup(ctx) HMAC_CTX_free(*(ctx))
#else
// 如果不支持不透明结构体，则定义 libssh2_hmac_ctx 为 HMAC_CTX
#define libssh2_hmac_ctx HMAC_CTX
// 初始化 libssh2_hmac_ctx
#define libssh2_hmac_ctx_init(ctx) \
  HMAC_CTX_init(&ctx)
// 使用 SHA1 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_sha1_init(ctx, key, keylen) \
  HMAC_Init_ex(ctx, key, keylen, EVP_sha1(), NULL)
// 使用 MD5 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_md5_init(ctx, key, keylen) \
  HMAC_Init_ex(ctx, key, keylen, EVP_md5(), NULL)
// 使用 RIPEMD160 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_ripemd160_init(ctx, key, keylen) \
  HMAC_Init_ex(ctx, key, keylen, EVP_ripemd160(), NULL)
// 使用 SHA256 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_sha256_init(ctx, key, keylen) \
  HMAC_Init_ex(ctx, key, keylen, EVP_sha256(), NULL)
// 使用 SHA512 算法初始化 libssh2_hmac_ctx
#define libssh2_hmac_sha512_init(ctx, key, keylen) \
  HMAC_Init_ex(ctx, key, keylen, EVP_sha512(), NULL)
// 更新 libssh2_hmac_ctx 的数据
#define libssh2_hmac_update(ctx, data, datalen) \
  HMAC_Update(&(ctx), data, datalen)
#define libssh2_hmac_final(ctx, data) HMAC_Final(&(ctx), data, NULL)
#define libssh2_hmac_cleanup(ctx) HMAC_cleanup(ctx)
#endif

extern void _libssh2_openssl_crypto_init(void);
extern void _libssh2_openssl_crypto_exit(void);
#define libssh2_crypto_init() _libssh2_openssl_crypto_init()
#define libssh2_crypto_exit() _libssh2_openssl_crypto_exit()

#define libssh2_rsa_ctx RSA

#define _libssh2_rsa_free(rsactx) RSA_free(rsactx)

#define libssh2_dsa_ctx DSA

#define _libssh2_dsa_free(dsactx) DSA_free(dsactx)

#if LIBSSH2_ECDSA
#define libssh2_ecdsa_ctx EC_KEY
#define _libssh2_ecdsa_free(ecdsactx) EC_KEY_free(ecdsactx)
#define _libssh2_ec_key EC_KEY

typedef enum {
    LIBSSH2_EC_CURVE_NISTP256 = NID_X9_62_prime256v1,
    LIBSSH2_EC_CURVE_NISTP384 = NID_secp384r1,
    LIBSSH2_EC_CURVE_NISTP521 = NID_secp521r1
}
libssh2_curve_type;
#else
#define _libssh2_ec_key void
#endif /* LIBSSH2_ECDSA */

#if LIBSSH2_ED25519
#define libssh2_ed25519_ctx EVP_PKEY

#define _libssh2_ed25519_free(ctx) EVP_PKEY_free(ctx)
#endif /* ED25519 */

#define _libssh2_cipher_type(name) const EVP_CIPHER *(*name)(void)
#ifdef HAVE_OPAQUE_STRUCTS
#define _libssh2_cipher_ctx EVP_CIPHER_CTX *
#else
#define _libssh2_cipher_ctx EVP_CIPHER_CTX
#endif

#define _libssh2_cipher_aes256 EVP_aes_256_cbc
#define _libssh2_cipher_aes192 EVP_aes_192_cbc
#define _libssh2_cipher_aes128 EVP_aes_128_cbc
#ifdef HAVE_EVP_AES_128_CTR
#define _libssh2_cipher_aes128ctr EVP_aes_128_ctr
#define _libssh2_cipher_aes192ctr EVP_aes_192_ctr
#define _libssh2_cipher_aes256ctr EVP_aes_256_ctr
#else
#define _libssh2_cipher_aes128ctr _libssh2_EVP_aes_128_ctr
#define _libssh2_cipher_aes192ctr _libssh2_EVP_aes_192_ctr
#define _libssh2_cipher_aes256ctr _libssh2_EVP_aes_256_ctr
#endif
#define _libssh2_cipher_blowfish EVP_bf_cbc
#define _libssh2_cipher_arcfour EVP_rc4
#define _libssh2_cipher_cast5 EVP_cast5_cbc
#define _libssh2_cipher_3des EVP_des_ede3_cbc

#ifdef HAVE_OPAQUE_STRUCTS
#ifndef __LIBSSH2_OPENSSL_H
#define __LIBSSH2_OPENSSL_H

// 定义 libssh2_cipher_dtor 宏，根据不同情况选择不同的函数
#define _libssh2_cipher_dtor(ctx) EVP_CIPHER_CTX_free(*(ctx))
#else
#define _libssh2_cipher_dtor(ctx) EVP_CIPHER_CTX_cleanup(ctx)

// 定义 libssh2_bn 和 libssh2_bn_ctx 宏，以及相关操作函数
#define _libssh2_bn BIGNUM
#define _libssh2_bn_ctx BN_CTX
#define _libssh2_bn_ctx_new() BN_CTX_new()
#define _libssh2_bn_ctx_free(bnctx) BN_CTX_free(bnctx)
#define _libssh2_bn_init() BN_new()
#define _libssh2_bn_init_from_bin() _libssh2_bn_init()
#define _libssh2_bn_set_word(bn, val) BN_set_word(bn, val)
#define _libssh2_bn_from_bin(bn, len, val) BN_bin2bn(val, len, bn)
#define _libssh2_bn_to_bin(bn, val) BN_bn2bin(bn, val)
#define _libssh2_bn_bytes(bn) BN_num_bytes(bn)
#define _libssh2_bn_bits(bn) BN_num_bits(bn)
#define _libssh2_bn_free(bn) BN_clear_free(bn)

// 定义 libssh2_dh_ctx 宏，以及相关操作函数
#define _libssh2_dh_ctx BIGNUM *
#define libssh2_dh_init(dhctx) _libssh2_dh_init(dhctx)
#define libssh2_dh_key_pair(dhctx, public, g, p, group_order, bnctx) \
        _libssh2_dh_key_pair(dhctx, public, g, p, group_order, bnctx)
#define libssh2_dh_secret(dhctx, secret, f, p, bnctx) \
        _libssh2_dh_secret(dhctx, secret, f, p, bnctx)
#define libssh2_dh_dtor(dhctx) _libssh2_dh_dtor(dhctx)
extern void _libssh2_dh_init(_libssh2_dh_ctx *dhctx);
extern int _libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                                _libssh2_bn *g, _libssh2_bn *p,
                                int group_order,
                                _libssh2_bn_ctx *bnctx);
extern int _libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                              _libssh2_bn *f, _libssh2_bn *p,
                              _libssh2_bn_ctx *bnctx);
extern void _libssh2_dh_dtor(_libssh2_dh_ctx *dhctx);

// 定义 libssh2_EVP_aes_128_ctr、libssh2_EVP_aes_192_ctr、libssh2_EVP_aes_256_ctr 函数
const EVP_CIPHER *_libssh2_EVP_aes_128_ctr(void);
const EVP_CIPHER *_libssh2_EVP_aes_192_ctr(void);
const EVP_CIPHER *_libssh2_EVP_aes_256_ctr(void);

#endif /* __LIBSSH2_OPENSSL_H */
```