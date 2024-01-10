# `nmap\libssh2\src\wincng.h`

```
#ifndef __LIBSSH2_WINCNG_H
#define __LIBSSH2_WINCNG_H
/*
 * 版权声明
 * 版权所有（C）2013-2020 Marc Hoersken <info@marc-hoersken.de>
 * 保留所有权利。
 *
 * 在源代码和二进制形式中重新分发和使用时，
 * 可以在满足以下条件的情况下进行：
 *
 *   必须保留上述版权声明、此条件列表和以下免责声明。
 *
 *   在文档和/或其他提供的材料中，必须在二进制形式中再现上述版权声明、此条件列表和以下免责声明。
 *
 *   未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称
 *   来支持或推广从本软件派生的产品。
 *
 * 本软件由版权所有者和贡献者“按原样”提供，任何明示或暗示的担保，
 * 包括但不限于对适销性和特定用途的暗示担保，都是不被允许的。
 * 在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）
 * 的情况下，版权所有者或贡献者均不对任何直接、间接、附带、
 * 特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、
 * 使用、数据或利润的损失或业务中断）负责，即使已被告知可能发生此类损害的可能性。
 */

/* 用于跨编译 w64 mingw-runtime 包的必需条件 */
#if defined(_WIN32_WINNT) && (_WIN32_WINNT < 0x0600)
#undef _WIN32_WINNT
#endif
#ifndef _WIN32_WINNT
#define _WIN32_WINNT 0x0600
#endif

#include <windows.h>
#include <bcrypt.h>

#define LIBSSH2_MD5 1

#define LIBSSH2_HMAC_RIPEMD 0
#define LIBSSH2_HMAC_SHA256 1
#define LIBSSH2_HMAC_SHA512 1

#define LIBSSH2_AES 1
#define LIBSSH2_AES_CTR 1
#define LIBSSH2_BLOWFISH 0
#define LIBSSH2_RC4 1
#define LIBSSH2_CAST 0
#define LIBSSH2_3DES 1

#define LIBSSH2_RSA 1
#define LIBSSH2_DSA 1
#define LIBSSH2_ECDSA 0
#define LIBSSH2_ED25519 0

#define MD5_DIGEST_LENGTH 16
#define SHA_DIGEST_LENGTH 20
#define SHA256_DIGEST_LENGTH 32
#define SHA384_DIGEST_LENGTH 48
#define SHA512_DIGEST_LENGTH 64

#define EC_MAX_POINT_LEN ((528 * 2 / 8) + 1)

#if LIBSSH2_ECDSA
#else
#define _libssh2_ec_key void
#endif

/*******************************************************************/
/*
 * Windows CNG backend: Global context handles
 */

struct _libssh2_wincng_ctx {
    BCRYPT_ALG_HANDLE hAlgRNG;  // 用于随机数生成的算法句柄
    BCRYPT_ALG_HANDLE hAlgHashMD5;  // MD5 哈希算法句柄
    BCRYPT_ALG_HANDLE hAlgHashSHA1;  // SHA1 哈希算法句柄
    BCRYPT_ALG_HANDLE hAlgHashSHA256;  // SHA256 哈希算法句柄
    BCRYPT_ALG_HANDLE hAlgHashSHA384;  // SHA384 哈希算法句柄
    BCRYPT_ALG_HANDLE hAlgHashSHA512;  // SHA512 哈希算法句柄
    BCRYPT_ALG_HANDLE hAlgHmacMD5;  // MD5 HMAC 算法句柄
    BCRYPT_ALG_HANDLE hAlgHmacSHA1;  // SHA1 HMAC 算法句柄
    BCRYPT_ALG_HANDLE hAlgHmacSHA256;  // SHA256 HMAC 算法句柄
    BCRYPT_ALG_HANDLE hAlgHmacSHA384;  // SHA384 HMAC 算法句柄
    BCRYPT_ALG_HANDLE hAlgHmacSHA512;  // SHA512 HMAC 算法句柄
    BCRYPT_ALG_HANDLE hAlgRSA;  // RSA 算法句柄
    BCRYPT_ALG_HANDLE hAlgDSA;  // DSA 算法句柄
    BCRYPT_ALG_HANDLE hAlgAES_CBC;  // AES CBC 算法句柄
    BCRYPT_ALG_HANDLE hAlgAES_ECB;  // AES ECB 算法句柄
    BCRYPT_ALG_HANDLE hAlgRC4_NA;  // RC4 算法句柄
    BCRYPT_ALG_HANDLE hAlg3DES_CBC;  // 3DES CBC 算法句柄
    BCRYPT_ALG_HANDLE hAlgDH;  // DH 算法句柄
    volatile int hasAlgDHwithKDF; /* -1=no, 0=maybe, 1=yes */  // 是否支持 DH with KDF
};

extern struct _libssh2_wincng_ctx _libssh2_wincng;


/*******************************************************************/
/*
 * Windows CNG backend: Generic functions
 */

void _libssh2_wincng_init(void);  // 初始化 Windows CNG 后端
void _libssh2_wincng_free(void);  // 释放 Windows CNG 后端资源

#define libssh2_crypto_init() \
  _libssh2_wincng_init()  // 加密库初始化
#define libssh2_crypto_exit() \
  _libssh2_wincng_free()  // 加密库退出

#define _libssh2_random(buf, len) \
  _libssh2_wincng_random(buf, len)  // 生成随机数

#define libssh2_prepare_iovec(vec, len)  /* Empty. */  // 准备 I/O 向量


/*******************************************************************/
/*
 * Windows CNG backend: Hash structure
 */
// 定义 Windows CNG 后端的哈希结构
typedef struct __libssh2_wincng_hash_ctx {
    BCRYPT_HASH_HANDLE hHash; // 哈希句柄
    unsigned char *pbHashObject; // 哈希对象的指针
    unsigned long dwHashObject; // 哈希对象的长度
    unsigned long cbHash; // 哈希的长度
} _libssh2_wincng_hash_ctx;

/*
 * Windows CNG backend: Hash functions
 */
// 定义 Windows CNG 后端的哈希函数

// SHA1 哈希函数
#define libssh2_sha1_ctx _libssh2_wincng_hash_ctx
#define libssh2_sha1_init(ctx) \
  (_libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHashSHA1, \
                            SHA_DIGEST_LENGTH, NULL, 0) == 0)
#define libssh2_sha1_update(ctx, data, datalen) \
  _libssh2_wincng_hash_update(&ctx, (unsigned char *) data, datalen)
#define libssh2_sha1_final(ctx, hash) \
  _libssh2_wincng_hash_final(&ctx, hash)
#define libssh2_sha1(data, datalen, hash) \
  _libssh2_wincng_hash(data, datalen, _libssh2_wincng.hAlgHashSHA1, \
                       hash, SHA_DIGEST_LENGTH)

// SHA256 哈希函数
#define libssh2_sha256_ctx _libssh2_wincng_hash_ctx
#define libssh2_sha256_init(ctx) \
  (_libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHashSHA256, \
                            SHA256_DIGEST_LENGTH, NULL, 0) == 0)
#define libssh2_sha256_update(ctx, data, datalen) \
  _libssh2_wincng_hash_update(&ctx, (unsigned char *) data, datalen)
#define libssh2_sha256_final(ctx, hash) \
  _libssh2_wincng_hash_final(&ctx, hash)
#define libssh2_sha256(data, datalen, hash) \
  _libssh2_wincng_hash(data, datalen, _libssh2_wincng.hAlgHashSHA256, \
                       hash, SHA256_DIGEST_LENGTH)

// SHA384 哈希函数
#define libssh2_sha384_ctx _libssh2_wincng_hash_ctx
#define libssh2_sha384_init(ctx) \
 (_libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHashSHA384, \
                           SHA384_DIGEST_LENGTH, NULL, 0) == 0)
#define libssh2_sha384_update(ctx, data, datalen) \
 _libssh2_wincng_hash_update(&ctx, (unsigned char *) data, datalen)
#define libssh2_sha384_final(ctx, hash) \
 _libssh2_wincng_hash_final(&ctx, hash)
#define libssh2_sha384(data, datalen, hash) \
# 使用 Windows CNG 后端进行哈希计算，计算 SHA384 哈希值
_libssh2_wincng_hash(data, datalen, _libssh2_wincng.hAlgHashSHA384, \
                     hash, SHA384_DIGEST_LENGTH)
# 定义 libssh2_sha512_ctx 为 _libssh2_wincng_hash_ctx
#define libssh2_sha512_ctx _libssh2_wincng_hash_ctx
# 初始化 SHA512 上下文
#define libssh2_sha512_init(ctx) \
  (_libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHashSHA512, \
                            SHA512_DIGEST_LENGTH, NULL, 0) == 0)
# 更新 SHA512 上下文
#define libssh2_sha512_update(ctx, data, datalen) \
  _libssh2_wincng_hash_update(&ctx, (unsigned char *) data, datalen)
# 完成 SHA512 计算
#define libssh2_sha512_final(ctx, hash) \
  _libssh2_wincng_hash_final(&ctx, hash)
# 计算 SHA512 哈希值
#define libssh2_sha512(data, datalen, hash) \
  _libssh2_wincng_hash(data, datalen, _libssh2_wincng.hAlgHashSHA512, \
                       hash, SHA512_DIGEST_LENGTH)

# 定义 libssh2_md5_ctx 为 _libssh2_wincng_hash_ctx
#define libssh2_md5_ctx _libssh2_wincng_hash_ctx
# 初始化 MD5 上下文
#define libssh2_md5_init(ctx) \
  (_libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHashMD5, \
                            MD5_DIGEST_LENGTH, NULL, 0) == 0)
# 更新 MD5 上下文
#define libssh2_md5_update(ctx, data, datalen) \
  _libssh2_wincng_hash_update(&ctx, (unsigned char *) data, datalen)
# 完成 MD5 计算
#define libssh2_md5_final(ctx, hash) \
  _libssh2_wincng_hash_final(&ctx, hash)
# 计算 MD5 哈希值
#define libssh2_md5(data, datalen, hash) \
  _libssh2_wincng_hash(data, datalen, _libssh2_wincng.hAlgHashMD5, \
                       hash, MD5_DIGEST_LENGTH)

# 定义 libssh2_hmac_ctx 为 _libssh2_wincng_hash_ctx
#define libssh2_hmac_ctx _libssh2_wincng_hash_ctx
# 初始化 HMAC 上下文
#define libssh2_hmac_ctx_init(ctx)
# 初始化 HMAC-SHA1 上下文
#define libssh2_hmac_sha1_init(ctx, key, keylen) \
  _libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHmacSHA1, \
                            SHA_DIGEST_LENGTH, key, keylen)
# 初始化 HMAC-MD5 上下文
#define libssh2_hmac_md5_init(ctx, key, keylen) \
  _libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHmacMD5, \
                            MD5_DIGEST_LENGTH, key, keylen)
# 初始化 HMAC-RIPEMD160 上下文，但未实现
#define libssh2_hmac_ripemd160_init(ctx, key, keylen)
  /* not implemented */
# 定义 libssh2_hmac_sha256_init 函数，初始化 SHA256 哈希算法上下文
# ctx: 哈希算法上下文
# key: 密钥
# keylen: 密钥长度
#define libssh2_hmac_sha256_init(ctx, key, keylen) \
  _libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHmacSHA256, \
                            SHA256_DIGEST_LENGTH, key, keylen)

# 定义 libssh2_hmac_sha512_init 函数，初始化 SHA512 哈希算法上下文
# ctx: 哈希算法上下文
# key: 密钥
# keylen: 密钥长度
#define libssh2_hmac_sha512_init(ctx, key, keylen) \
  _libssh2_wincng_hash_init(ctx, _libssh2_wincng.hAlgHmacSHA512, \
                            SHA512_DIGEST_LENGTH, key, keylen)

# 定义 libssh2_hmac_update 函数，更新哈希算法上下文
# ctx: 哈希算法上下文
# data: 数据
# datalen: 数据长度
#define libssh2_hmac_update(ctx, data, datalen) \
  _libssh2_wincng_hash_update(&ctx, (unsigned char *) data, datalen)

# 定义 libssh2_hmac_final 函数，完成哈希计算
# ctx: 哈希算法上下文
# hash: 哈希值
#define libssh2_hmac_final(ctx, hash) \
  _libssh2_wincng_hmac_final(&ctx, hash)

# 定义 libssh2_hmac_cleanup 函数，清理哈希算法上下文
# ctx: 哈希算法上下文
#define libssh2_hmac_cleanup(ctx) \
  _libssh2_wincng_hmac_cleanup(ctx)

# 定义 Windows CNG 后端的密钥上下文结构
typedef struct __libssh2_wincng_key_ctx {
    BCRYPT_KEY_HANDLE hKey;  # 密钥句柄
    unsigned char *pbKeyObject;  # 密钥对象
    unsigned long cbKeyObject;  # 密钥对象长度
} _libssh2_wincng_key_ctx;

# 定义 Windows CNG 后端的 RSA 函数
#define libssh2_rsa_ctx _libssh2_wincng_key_ctx  # RSA 上下文
#define _libssh2_rsa_new(rsactx, e, e_len, n, n_len, \
                         d, d_len, p, p_len, q, q_len, \
                         e1, e1_len, e2, e2_len, c, c_len) \
  _libssh2_wincng_rsa_new(rsactx, e, e_len, n, n_len, \
                          d, d_len, p, p_len, q, q_len, \
                          e1, e1_len, e2, e2_len, c, c_len)  # 创建 RSA 上下文
#define _libssh2_rsa_new_private(rsactx, s, filename, passphrase) \
  _libssh2_wincng_rsa_new_private(rsactx, s, filename, passphrase)  # 从文件中创建私钥 RSA 上下文
#define _libssh2_rsa_new_private_frommemory(rsactx, s, filedata, \
                                            filedata_len, passphrase) \
  _libssh2_wincng_rsa_new_private_frommemory(rsactx, s, filedata, \
                                             filedata_len, passphrase)  # 从内存中创建私钥 RSA 上下文
#define _libssh2_rsa_sha1_sign(s, rsactx, hash, hash_len, sig, sig_len) \
  _libssh2_wincng_rsa_sha1_sign(s, rsactx, hash, hash_len, sig, sig_len)  # 使用 SHA1 对数据进行签名
// 定义一个宏，用于调用 Windows CNG RSA SHA1 验证函数
#define _libssh2_rsa_sha1_verify(rsactx, sig, sig_len, m, m_len) \
  _libssh2_wincng_rsa_sha1_verify(rsactx, sig, sig_len, m, m_len)
// 定义一个宏，用于调用 Windows CNG RSA 释放函数
#define _libssh2_rsa_free(rsactx) \
  _libssh2_wincng_rsa_free(rsactx)

/*
 * Windows CNG backend: DSA functions
 */

// 定义 DSA 上下文为 Windows CNG 密钥上下文
#define libssh2_dsa_ctx _libssh2_wincng_key_ctx
// 定义一个宏，用于调用 Windows CNG DSA 创建函数
#define _libssh2_dsa_new(dsactx, p, p_len, q, q_len, \
                         g, g_len, y, y_len, x, x_len) \
  _libssh2_wincng_dsa_new(dsactx, p, p_len, q, q_len, \
                          g, g_len, y, y_len, x, x_len)
// 定义一个宏，用于调用 Windows CNG DSA 创建私钥函数
#define _libssh2_dsa_new_private(dsactx, s, filename, passphrase) \
  _libssh2_wincng_dsa_new_private(dsactx, s, filename, passphrase)
// 定义一个宏，用于调用 Windows CNG DSA 从内存中创建私钥函数
#define _libssh2_dsa_new_private_frommemory(dsactx, s, filedata, \
                                            filedata_len, passphrase) \
  _libssh2_wincng_dsa_new_private_frommemory(dsactx, s, filedata, \
                                             filedata_len, passphrase)
// 定义一个宏，用于调用 Windows CNG DSA SHA1 签名函数
#define _libssh2_dsa_sha1_sign(dsactx, hash, hash_len, sig) \
  _libssh2_wincng_dsa_sha1_sign(dsactx, hash, hash_len, sig)
// 定义一个宏，用于调用 Windows CNG DSA SHA1 验证函数
#define _libssh2_dsa_sha1_verify(dsactx, sig, m, m_len) \
  _libssh2_wincng_dsa_sha1_verify(dsactx, sig, m, m_len)
// 定义一个宏，用于调用 Windows CNG DSA 释放函数
#define _libssh2_dsa_free(dsactx) \
  _libssh2_wincng_dsa_free(dsactx)

/*
 * Windows CNG backend: Key functions
 */

// 定义一个宏，用于调用 Windows CNG 公私钥文件函数
#define _libssh2_pub_priv_keyfile(s, m, m_len, p, p_len, pk, pw) \
  _libssh2_wincng_pub_priv_keyfile(s, m, m_len, p, p_len, pk, pw)
// 定义一个宏，用于调用 Windows CNG 公私钥内存函数
#define _libssh2_pub_priv_keyfilememory(s, m, m_len, p, p_len, \
                                                     pk, pk_len, pw) \
  _libssh2_wincng_pub_priv_keyfilememory(s, m, m_len, p, p_len, \
                                                      pk, pk_len, pw)


/*******************************************************************/
/*
 * Windows CNG backend: Cipher Context structure
 */

// 定义 Windows CNG 密码上下文结构
struct _libssh2_wincng_cipher_ctx {
    BCRYPT_KEY_HANDLE hKey; // 密钥句柄
    unsigned char *pbKeyObject; // 密钥对象
    unsigned char *pbIV; // 初始化向量
    unsigned char *pbCtr; // 计数器
    # 定义一个无符号长整型变量，用于存储密钥对象
    unsigned long dwKeyObject;
    # 定义一个无符号长整型变量，用于存储初始化向量
    unsigned long dwIV;
    # 定义一个无符号长整型变量，用于存储块长度
    unsigned long dwBlockLength;
    # 定义一个无符号长整型变量，用于存储计数器长度
    unsigned long dwCtrLength;
};

#define _libssh2_cipher_ctx struct _libssh2_wincng_cipher_ctx
// 定义 _libssh2_cipher_ctx 为 struct _libssh2_wincng_cipher_ctx

/*
 * Windows CNG backend: Cipher Type structure
 */
// Windows CNG 后端：密码类型结构

struct _libssh2_wincng_cipher_type {
    BCRYPT_ALG_HANDLE *phAlg;
    unsigned long dwKeyLength;
    int useIV;      /* TODO: Convert to bool when a C89 compatible bool type
                       is defined */
    int ctrMode;
};
// 定义 _libssh2_wincng_cipher_type 结构体，包含 BCRYPT_ALG_HANDLE 指针、密钥长度、是否使用 IV、计数器模式

#define _libssh2_cipher_type(type) struct _libssh2_wincng_cipher_type type
// 定义 _libssh2_cipher_type 宏，用于创建 _libssh2_wincng_cipher_type 结构体

#define _libssh2_cipher_aes256ctr { &_libssh2_wincng.hAlgAES_ECB, 32, 0, 1 }
#define _libssh2_cipher_aes192ctr { &_libssh2_wincng.hAlgAES_ECB, 24, 0, 1 }
#define _libssh2_cipher_aes128ctr { &_libssh2_wincng.hAlgAES_ECB, 16, 0, 1 }
#define _libssh2_cipher_aes256 { &_libssh2_wincng.hAlgAES_CBC, 32, 1, 0 }
#define _libssh2_cipher_aes192 { &_libssh2_wincng.hAlgAES_CBC, 24, 1, 0 }
#define _libssh2_cipher_aes128 { &_libssh2_wincng.hAlgAES_CBC, 16, 1, 0 }
#define _libssh2_cipher_arcfour { &_libssh2_wincng.hAlgRC4_NA, 16, 0, 0 }
#define _libssh2_cipher_3des { &_libssh2_wincng.hAlg3DES_CBC, 24, 1, 0 }
// 定义不同类型的密码，包括算法句柄、密钥长度、是否使用 IV、计数器模式

/*
 * Windows CNG backend: Cipher functions
 */
// Windows CNG 后端：密码函数

#define _libssh2_cipher_init(ctx, type, iv, secret, encrypt) \
  _libssh2_wincng_cipher_init(ctx, type, iv, secret, encrypt)
#define _libssh2_cipher_crypt(ctx, type, encrypt, block, blocklen) \
  _libssh2_wincng_cipher_crypt(ctx, type, encrypt, block, blocklen)
#define _libssh2_cipher_dtor(ctx) \
  _libssh2_wincng_cipher_dtor(ctx)
// 定义密码初始化、加密/解密、析构函数的宏，调用相应的 Windows CNG 后端函数

/*******************************************************************/
/*
 * Windows CNG backend: BigNumber Context
 */
// Windows CNG 后端：大数上下文

#define _libssh2_bn_ctx int /* not used */
#define _libssh2_bn_ctx_new() 0 /* not used */
#define _libssh2_bn_ctx_free(bnctx) ((void)0) /* not used */
// 定义大数上下文的宏，但未使用

/*******************************************************************/
/*
 * Windows CNG backend: BigNumber structure
 */
// Windows CNG 后端：大数结构

struct _libssh2_wincng_bignum {
    unsigned char *bignum;
    unsigned long length;
};
// 定义 _libssh2_wincng_bignum 结构体，包含大数和长度

#define _libssh2_bn struct _libssh2_wincng_bignum
// 定义 _libssh2_bn 宏，用于创建 _libssh2_wincng_bignum 结构体
/*
 * Windows CNG backend: BigNumber functions
 */

// 初始化大数对象
_libssh2_bn *_libssh2_wincng_bignum_init(void);

// 初始化大数对象
#define _libssh2_bn_init() \
  _libssh2_wincng_bignum_init()
// 从二进制数据初始化大数对象
#define _libssh2_bn_init_from_bin() \
  _libssh2_bn_init()
// 设置大数对象的值为一个字
#define _libssh2_bn_set_word(bn, word) \
  _libssh2_wincng_bignum_set_word(bn, word)
// 从二进制数据设置大数对象的值
#define _libssh2_bn_from_bin(bn, len, bin) \
  _libssh2_wincng_bignum_from_bin(bn, len, bin)
// 将大数对象转换为二进制数据
#define _libssh2_bn_to_bin(bn, bin) \
  _libssh2_wincng_bignum_to_bin(bn, bin)
// 返回大数对象的字节数
#define _libssh2_bn_bytes(bn) bn->length
// 返回大数对象的位数
#define _libssh2_bn_bits(bn) \
  _libssh2_wincng_bignum_bits(bn)
// 释放大数对象
#define _libssh2_bn_free(bn) \
  _libssh2_wincng_bignum_free(bn)

/*
 * Windows CNG backend: Diffie-Hellman support
 */

// 定义 Diffie-Hellman 上下文结构
typedef struct {
    /* holds our private and public key components */
    BCRYPT_KEY_HANDLE dh_handle;
    /* records the parsed out modulus and generator
     * parameters that are shared  with the peer */
    BCRYPT_DH_PARAMETER_HEADER *dh_params;
    /* records the parsed out private key component for
     * fallback if the DH API raw KDF is not supported */
    struct _libssh2_wincng_bignum *bn;
} _libssh2_dh_ctx;

// 初始化 Diffie-Hellman 上下文
#define libssh2_dh_init(dhctx) _libssh2_dh_init(dhctx)
// 生成 Diffie-Hellman 密钥对
#define libssh2_dh_key_pair(dhctx, public, g, p, group_order, bnctx) \
        _libssh2_dh_key_pair(dhctx, public, g, p, group_order)
// 计算 Diffie-Hellman 共享密钥
#define libssh2_dh_secret(dhctx, secret, f, p, bnctx) \
        _libssh2_dh_secret(dhctx, secret, f, p)
// 释放 Diffie-Hellman 上下文
#define libssh2_dh_dtor(dhctx) _libssh2_dh_dtor(dhctx)

/*******************************************************************/
/*
 * Windows CNG backend: forward declarations
 */

// 初始化 Windows CNG 后端
void _libssh2_wincng_init(void);
// 释放 Windows CNG 后端资源
void _libssh2_wincng_free(void);
// 生成指定长度的随机数据
int _libssh2_wincng_random(void *buf, int len);

// 初始化哈希计算上下文
int
_libssh2_wincng_hash_init(_libssh2_wincng_hash_ctx *ctx,
                          BCRYPT_ALG_HANDLE hAlg, unsigned long hashlen,
                          unsigned char *key, unsigned long keylen);
int
# 更新 Windows CNG 哈希上下文中的数据
_libssh2_wincng_hash_update(_libssh2_wincng_hash_ctx *ctx,
                            const unsigned char *data, unsigned long datalen);
# 完成 Windows CNG 哈希计算，将结果存储在指定的哈希值中
int
_libssh2_wincng_hash_final(_libssh2_wincng_hash_ctx *ctx,
                           unsigned char *hash);
# 使用 Windows CNG 哈希算法计算数据的哈希值
int
_libssh2_wincng_hash(unsigned char *data, unsigned long datalen,
                     BCRYPT_ALG_HANDLE hAlg,
                     unsigned char *hash, unsigned long hashlen);
# 完成 Windows CNG HMAC 计算，将结果存储在指定的哈希值中
int
_libssh2_wincng_hmac_final(_libssh2_wincng_hash_ctx *ctx,
                           unsigned char *hash);
# 清理 Windows CNG HMAC 上下文
void
_libssh2_wincng_hmac_cleanup(_libssh2_wincng_hash_ctx *ctx);
# 验证 Windows CNG RSA 密钥的 SHA1 签名
int
_libssh2_wincng_key_sha1_verify(_libssh2_wincng_key_ctx *ctx,
                                const unsigned char *sig,
                                unsigned long sig_len,
                                const unsigned char *m,
                                unsigned long m_len,
                                unsigned long flags);
# 创建新的 Windows CNG RSA 密钥
int
_libssh2_wincng_rsa_new(libssh2_rsa_ctx **rsa,
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
# 创建一个新的 RSA 私钥对象，并将其存储在 rsa 指针所指向的位置
_libssh2_wincng_rsa_new_private(libssh2_rsa_ctx **rsa,
                                LIBSSH2_SESSION *session,
                                const char *filename,
                                const unsigned char *passphrase);

# 从内存中的数据创建一个新的 RSA 私钥对象，并将其存储在 rsa 指针所指向的位置
int
_libssh2_wincng_rsa_new_private_frommemory(libssh2_rsa_ctx **rsa,
                                           LIBSSH2_SESSION *session,
                                           const char *filedata,
                                           size_t filedata_len,
                                           unsigned const char *passphrase);

# 使用 SHA1 算法验证 RSA 签名
int
_libssh2_wincng_rsa_sha1_verify(libssh2_rsa_ctx *rsa,
                                const unsigned char *sig,
                                unsigned long sig_len,
                                const unsigned char *m,
                                unsigned long m_len);

# 使用 SHA1 算法对数据进行 RSA 签名
int
_libssh2_wincng_rsa_sha1_sign(LIBSSH2_SESSION *session,
                              libssh2_rsa_ctx *rsa,
                              const unsigned char *hash,
                              size_t hash_len,
                              unsigned char **signature,
                              size_t *signature_len);

# 释放 RSA 对象所占用的内存
void
_libssh2_wincng_rsa_free(libssh2_rsa_ctx *rsa);

# 如果支持 DSA，创建一个新的 DSA 对象
#if LIBSSH2_DSA
int
_libssh2_wincng_dsa_new(libssh2_dsa_ctx **dsa,
                        const unsigned char *pdata,
                        unsigned long plen,
                        const unsigned char *qdata,
                        unsigned long qlen,
                        const unsigned char *gdata,
                        unsigned long glen,
                        const unsigned char *ydata,
                        unsigned long ylen,
                        const unsigned char *xdata,
                        unsigned long xlen);
# 创建一个新的 DSA 密钥对，并将私钥保存到文件中
_libssh2_wincng_dsa_new_private(libssh2_dsa_ctx **dsa,
                                LIBSSH2_SESSION *session,
                                const char *filename,
                                const unsigned char *passphrase);

# 从内存中的数据创建一个新的 DSA 私钥
_libssh2_wincng_dsa_new_private_frommemory(libssh2_dsa_ctx **dsa,
                                           LIBSSH2_SESSION *session,
                                           const char *filedata,
                                           size_t filedata_len,
                                           unsigned const char *passphrase);

# 使用 SHA1 算法验证 DSA 签名
_libssh2_wincng_dsa_sha1_verify(libssh2_dsa_ctx *dsa,
                                const unsigned char *sig_fixed,
                                const unsigned char *m,
                                unsigned long m_len);

# 使用 SHA1 算法对数据进行 DSA 签名
_libssh2_wincng_dsa_sha1_sign(libssh2_dsa_ctx *dsa,
                              const unsigned char *hash,
                              unsigned long hash_len,
                              unsigned char *sig_fixed);

# 释放 DSA 密钥对占用的内存
void
_libssh2_wincng_dsa_free(libssh2_dsa_ctx *dsa);
#endif

# 从公钥/私钥文件中获取公钥和私钥数据
int
_libssh2_wincng_pub_priv_keyfile(LIBSSH2_SESSION *session,
                                 unsigned char **method,
                                 size_t *method_len,
                                 unsigned char **pubkeydata,
                                 size_t *pubkeydata_len,
                                 const char *privatekey,
                                 const char *passphrase);
int
# 使用内存中的公私钥文件初始化会话
_libssh2_wincng_pub_priv_keyfilememory(LIBSSH2_SESSION *session,
                                       unsigned char **method,
                                       size_t *method_len,
                                       unsigned char **pubkeydata,
                                       size_t *pubkeydata_len,
                                       const char *privatekeydata,
                                       size_t privatekeydata_len,
                                       const char *passphrase);

# 初始化加密上下文
int
_libssh2_wincng_cipher_init(_libssh2_cipher_ctx *ctx,
                            _libssh2_cipher_type(type),
                            unsigned char *iv,
                            unsigned char *secret,
                            int encrypt);

# 加密或解密数据
int
_libssh2_wincng_cipher_crypt(_libssh2_cipher_ctx *ctx,
                             _libssh2_cipher_type(type),
                             int encrypt,
                             unsigned char *block,
                             size_t blocklen);

# 释放加密上下文
void
_libssh2_wincng_cipher_dtor(_libssh2_cipher_ctx *ctx);

# 初始化大数
_libssh2_bn *
_libssh2_wincng_bignum_init(void);

# 设置大数的值
int
_libssh2_wincng_bignum_set_word(_libssh2_bn *bn, unsigned long word);

# 获取大数的位数
unsigned long
_libssh2_wincng_bignum_bits(const _libssh2_bn *bn);

# 从二进制数据中初始化大数
void
_libssh2_wincng_bignum_from_bin(_libssh2_bn *bn, unsigned long len,
                                const unsigned char *bin);

# 将大数转换为二进制数据
void
_libssh2_wincng_bignum_to_bin(const _libssh2_bn *bn, unsigned char *bin);

# 释放大数
void
_libssh2_wincng_bignum_free(_libssh2_bn *bn);

# 初始化 DH 上下文
extern void
_libssh2_dh_init(_libssh2_dh_ctx *dhctx);

# 生成 DH 密钥对
extern int
_libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                     _libssh2_bn *g, _libssh2_bn *p, int group_order);

# 计算 DH 共享密钥
extern int
_libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                   _libssh2_bn *f, _libssh2_bn *p);

# 释放 DH 上下文
extern void
_libssh2_dh_dtor(_libssh2_dh_ctx *dhctx);

# 结束条件编译
#endif /* __LIBSSH2_WINCNG_H */
```