# `nmap\libssh2\src\crypto.h`

```
#ifndef __LIBSSH2_CRYPTO_H
#define __LIBSSH2_CRYPTO_H
/* 定义了一个条件编译宏，用于避免重复包含同一头文件 */
/* 版权声明，包括作者和版权信息 */
/* 允许在源码和二进制形式下重新分发和使用，需满足以下条件 */
/* 在源码形式下重新分发时，必须保留版权声明、条件列表和下面的免责声明 */
/* 在二进制形式下重新分发时，必须在文档和/或其他提供的材料中重现版权声明、条件列表和下面的免责声明 */
/* 未经特定事先书面许可，不得使用版权所有者或其他贡献者的名字来认可或推广从本软件衍生的产品 */
/* 免责声明，声明本软件按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保，不得因使用本软件而产生任何直接、间接、附带、特殊、惩罚性或后果性的损害赔偿责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他） */
/* 条件编译，根据不同的加密库选择不同的头文件进行包含 */
#ifdef LIBSSH2_OPENSSL
#include "openssl.h"
#endif

#ifdef LIBSSH2_LIBGCRYPT
#include "libgcrypt.h"
#endif

#ifdef LIBSSH2_WINCNG
#include "wincng.h"
#endif

#ifdef LIBSSH2_OS400QC3
#include "os400qc3.h"
#endif

#ifdef LIBSSH2_MBEDTLS
#ifdef mbedtls.h
#endif

// 定义 ED25519 密钥长度
#define LIBSSH2_ED25519_KEY_LEN 32
// 定义 ED25519 私钥长度
#define LIBSSH2_ED25519_PRIVATE_KEY_LEN 64
// 定义 ED25519 签名长度
#define LIBSSH2_ED25519_SIG_LEN 64

#if LIBSSH2_RSA
// 创建新的 RSA 密钥对
int _libssh2_rsa_new(libssh2_rsa_ctx ** rsa,
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
                     const unsigned char *coeffdata, unsigned long coefflen);
// 从私钥文件中创建 RSA 密钥对
int _libssh2_rsa_new_private(libssh2_rsa_ctx ** rsa,
                             LIBSSH2_SESSION * session,
                             const char *filename,
                             unsigned const char *passphrase);
// 使用 SHA1 算法验证 RSA 签名
int _libssh2_rsa_sha1_verify(libssh2_rsa_ctx * rsa,
                             const unsigned char *sig,
                             unsigned long sig_len,
                             const unsigned char *m, unsigned long m_len);
// 使用 SHA1 算法对数据进行 RSA 签名
int _libssh2_rsa_sha1_sign(LIBSSH2_SESSION * session,
                           libssh2_rsa_ctx * rsactx,
                           const unsigned char *hash,
                           size_t hash_len,
                           unsigned char **signature,
                           size_t *signature_len);
# 定义一个函数，从内存中创建一个 RSA 私钥
int _libssh2_rsa_new_private_frommemory(libssh2_rsa_ctx ** rsa,
                                        LIBSSH2_SESSION * session,
                                        const char *filedata,
                                        size_t filedata_len,
                                        unsigned const char *passphrase);
#endif

# 如果支持 DSA，定义一系列与 DSA 相关的函数
#if LIBSSH2_DSA
# 定义一个函数，创建一个新的 DSA 上下文
int _libssh2_dsa_new(libssh2_dsa_ctx ** dsa,
                     const unsigned char *pdata,
                     unsigned long plen,
                     const unsigned char *qdata,
                     unsigned long qlen,
                     const unsigned char *gdata,
                     unsigned long glen,
                     const unsigned char *ydata,
                     unsigned long ylen,
                     const unsigned char *x, unsigned long x_len);
# 定义一个函数，从文件中创建一个 DSA 私钥
int _libssh2_dsa_new_private(libssh2_dsa_ctx ** dsa,
                             LIBSSH2_SESSION * session,
                             const char *filename,
                             unsigned const char *passphrase);
# 定义一个函数，验证 DSA 签名
int _libssh2_dsa_sha1_verify(libssh2_dsa_ctx * dsactx,
                             const unsigned char *sig,
                             const unsigned char *m, unsigned long m_len);
# 定义一个函数，对数据进行 DSA 签名
int _libssh2_dsa_sha1_sign(libssh2_dsa_ctx * dsactx,
                           const unsigned char *hash,
                           unsigned long hash_len, unsigned char *sig);
# 定义一个函数，从内存中创建一个 DSA 私钥
int _libssh2_dsa_new_private_frommemory(libssh2_dsa_ctx ** dsa,
                                        LIBSSH2_SESSION * session,
                                        const char *filedata,
                                        size_t filedata_len,
                                        unsigned const char *passphrase);
#endif

# 如果支持 ECDSA，定义一系列与 ECDSA 相关的函数
#if LIBSSH2_ECDSA
int
# 创建一个新的 ECDSA 上下文，使用给定的公钥和曲线类型
_libssh2_ecdsa_curve_name_with_octal_new(libssh2_ecdsa_ctx ** ecdsactx,
                                         const unsigned char *k,
                                         size_t k_len,
                                         libssh2_curve_type type);

# 创建一个新的 ECDSA 私钥
int
_libssh2_ecdsa_new_private(libssh2_ecdsa_ctx ** ec_ctx,
                           LIBSSH2_SESSION * session,
                           const char *filename,
                           unsigned const char *passphrase);

# 验证 ECDSA 签名
int
_libssh2_ecdsa_verify(libssh2_ecdsa_ctx * ctx,
                      const unsigned char *r, size_t r_len,
                      const unsigned char *s, size_t s_len,
                      const unsigned char *m, size_t m_len);

# 创建一个新的 ECDSA 密钥对
int
_libssh2_ecdsa_create_key(LIBSSH2_SESSION *session,
                          _libssh2_ec_key **out_private_key,
                          unsigned char **out_public_key_octal,
                          size_t *out_public_key_octal_len,
                          libssh2_curve_type curve_type);

# 生成 ECDH 密钥
int
_libssh2_ecdh_gen_k(_libssh2_bn **k, _libssh2_ec_key *private_key,
                    const unsigned char *server_public_key,
                    size_t server_public_key_len);

# 对给定的哈希值进行 ECDSA 签名
int
_libssh2_ecdsa_sign(LIBSSH2_SESSION *session, libssh2_ecdsa_ctx *ec_ctx,
                    const unsigned char *hash, unsigned long hash_len,
                    unsigned char **signature, size_t *signature_len);

# 从内存中创建一个新的 ECDSA 私钥
int _libssh2_ecdsa_new_private_frommemory(libssh2_ecdsa_ctx ** ec_ctx,
                                          LIBSSH2_SESSION * session,
                                          const char *filedata,
                                          size_t filedata_len,
                                          unsigned const char *passphrase);

# 获取 ECDSA 上下文的曲线类型
libssh2_curve_type
_libssh2_ecdsa_get_curve_type(libssh2_ecdsa_ctx *ec_ctx);

# 根据曲线名称获取曲线类型
int
_libssh2_ecdsa_curve_type_from_name(const char *name,
                                    libssh2_curve_type *out_type);

#endif /* LIBSSH2_ECDSA */
#if LIBSSH2_ED25519

// 生成新的 Curve25519 密钥对
int
_libssh2_curve25519_new(LIBSSH2_SESSION *session, uint8_t **out_public_key,
                        uint8_t **out_private_key);

// 生成 Curve25519 公钥对应的共享密钥
int
_libssh2_curve25519_gen_k(_libssh2_bn **k,
                          uint8_t private_key[LIBSSH2_ED25519_KEY_LEN],
                          uint8_t server_public_key[LIBSSH2_ED25519_KEY_LEN]);

// 验证 Ed25519 签名
int
_libssh2_ed25519_verify(libssh2_ed25519_ctx *ctx, const uint8_t *s,
                        size_t s_len, const uint8_t *m, size_t m_len);

// 从文件中读取 Ed25519 私钥
int
_libssh2_ed25519_new_private(libssh2_ed25519_ctx **ed_ctx,
                            LIBSSH2_SESSION *session,
                            const char *filename, const uint8_t *passphrase);

// 从公钥数据创建 Ed25519 公钥
int
_libssh2_ed25519_new_public(libssh2_ed25519_ctx **ed_ctx,
                            LIBSSH2_SESSION *session,
                            const unsigned char *raw_pub_key,
                            const uint8_t key_len);

// 对消息进行 Ed25519 签名
int
_libssh2_ed25519_sign(libssh2_ed25519_ctx *ctx, LIBSSH2_SESSION *session,
                      uint8_t **out_sig, size_t *out_sig_len,
                      const uint8_t *message, size_t message_len);

// 从内存中读取 Ed25519 私钥
int
_libssh2_ed25519_new_private_frommemory(libssh2_ed25519_ctx **ed_ctx,
                                        LIBSSH2_SESSION *session,
                                        const char *filedata,
                                        size_t filedata_len,
                                        unsigned const char *passphrase);

#endif /* LIBSSH2_ED25519 */

// 初始化加密算法
int _libssh2_cipher_init(_libssh2_cipher_ctx * h,
                         _libssh2_cipher_type(algo),
                         unsigned char *iv,
                         unsigned char *secret, int encrypt);

// 加密或解密数据
int _libssh2_cipher_crypt(_libssh2_cipher_ctx * ctx,
                          _libssh2_cipher_type(algo),
                          int encrypt, unsigned char *block, size_t blocksize);
# 定义 libssh2 库中用于处理公私钥文件的函数 _libssh2_pub_priv_keyfile
int _libssh2_pub_priv_keyfile(LIBSSH2_SESSION *session,  // SSH 会话对象
                              unsigned char **method,    // 用于存储认证方法的指针
                              size_t *method_len,        // 用于存储认证方法长度的指针
                              unsigned char **pubkeydata, // 用于存储公钥数据的指针
                              size_t *pubkeydata_len,     // 用于存储公钥数据长度的指针
                              const char *privatekey,     // 私钥文件路径
                              const char *passphrase);    // 私钥密码

# 定义 libssh2 库中用于处理内存中公私钥数据的函数 _libssh2_pub_priv_keyfilememory
int _libssh2_pub_priv_keyfilememory(LIBSSH2_SESSION *session,  // SSH 会话对象
                                    unsigned char **method,    // 用于存储认证方法的指针
                                    size_t *method_len,        // 用于存储认证方法长度的指针
                                    unsigned char **pubkeydata, // 用于存储公钥数据的指针
                                    size_t *pubkeydata_len,     // 用于存储公钥数据长度的指针
                                    const char *privatekeydata, // 内存中的私钥数据
                                    size_t privatekeydata_len,  // 私钥数据长度
                                    const char *passphrase);    // 私钥密码

# 结束 libssh2_crypto.h 文件的定义
#endif /* __LIBSSH2_CRYPTO_H */
```