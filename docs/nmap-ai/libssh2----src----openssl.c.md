# `nmap\libssh2\src\openssl.c`

```
/*
 * 版权声明和作者信息
 */
/* 版权声明和作者信息 */
/* 版权声明和作者信息 */
/* 作者信息 */
 * Author: Simon Josefsson
 *
 * 条款和条件
 */
/* Redistribution and use in source and binary forms,
 * with or without modification, are permitted provided
 * that the following conditions are met:
 *
 *   Redistributions of source code must retain the above
 *   copyright notice, this list of conditions and the
 *   following disclaimer.
 *
 *   Redistributions in binary form must reproduce the above
 *   copyright notice, this list of conditions and the following
 *   disclaimer in the documentation and/or other materials
 *   provided with the distribution.
 *
 *   Neither the name of the copyright holder nor the names
 *   of any other contributors may be used to endorse or
 *   promote products derived from this software without
 *   specific prior written permission.
 *
 * THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND
 * CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES,
 * INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES
 * OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
 * ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR
 * CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
 * SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING,
 * BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
 * SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY,
 * WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
 * NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE
 * USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY
 * OF SUCH DAMAGE.
 */
/* 包含头文件 */
#include "libssh2_priv.h"
/* 如果使用 OpenSSL 编译，则包含以下内容 */
#ifdef LIBSSH2_OPENSSL /* compile only if we build with openssl */
/* 包含标准库头文件 */
#include <string.h>
/* 包含自定义的杂项函数头文件 */
#include "misc.h"
/* 定义 EVP_MAX_BLOCK_LENGTH */
#ifndef EVP_MAX_BLOCK_LENGTH
#define EVP_MAX_BLOCK_LENGTH 32
#endif
/* 函数定义开始 */
int
# 从内存中读取 OpenSSH 私钥
def read_openssh_private_key_from_memory(void **key_ctx, LIBSSH2_SESSION *session,
                                         const char *key_type,
                                         const char *filedata,
                                         size_t filedata_len,
                                         unsigned const char *passphrase):
    # 未提供具体代码，无法添加注释

# 将大数（BIGNUM）写入缓冲区
static unsigned char *
write_bn(unsigned char *buf, const BIGNUM *bn, int bn_bytes):
    # 初始化指针 p，指向缓冲区
    unsigned char *p = buf;

    # 留出空间用于写入 bn 的大小，该大小将在下面写入
    p += 4;

    # 将 bn 转换为二进制数据，写入缓冲区
    *p = 0;
    BN_bn2bin(bn, p + 1);
    # 如果二进制数据的第一个字节不是 0x80，则移动数据，减小 bn_bytes
    if(!(*(p + 1) & 0x80)):
        memmove(p, p + 1, --bn_bytes);
    # 将 bn 的大小转换为网络字节顺序，写入缓冲区
    _libssh2_htonu32(p - 4, bn_bytes);

    # 返回指向写入数据后的指针
    return p + bn_bytes;

# 创建新的 RSA 上下文
int
_libssh2_rsa_new(libssh2_rsa_ctx ** rsa,
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
                 const unsigned char *coeffdata, unsigned long coefflen):
    # 初始化大数变量
    BIGNUM * e;
    BIGNUM * n;
    BIGNUM * d = 0;
    BIGNUM * p = 0;
    BIGNUM * q = 0;
    BIGNUM * dmp1 = 0;
    BIGNUM * dmq1 = 0;
    BIGNUM * iqmp = 0;

    # 创建新的大数 e，并将二进制数据 edata 转换为大数
    e = BN_new();
    BN_bin2bn(edata, elen, e);

    # 创建新的大数 n，并将二进制数据 ndata 转换为大数
    n = BN_new();
    BN_bin2bn(ndata, nlen, n);
    # 如果 ddata 存在（非空），则执行以下操作
    if(ddata) {
        # 创建一个新的大数对象 d，并将 ddata 转换为大数对象
        d = BN_new();
        BN_bin2bn(ddata, dlen, d);

        # 创建一个新的大数对象 p，并将 pdata 转换为大数对象
        p = BN_new();
        BN_bin2bn(pdata, plen, p);

        # 创建一个新的大数对象 q，并将 qdata 转换为大数对象
        q = BN_new();
        BN_bin2bn(qdata, qlen, q);

        # 创建一个新的大数对象 dmp1，并将 e1data 转换为大数对象
        dmp1 = BN_new();
        BN_bin2bn(e1data, e1len, dmp1);

        # 创建一个新的大数对象 dmq1，并将 e2data 转换为大数对象
        dmq1 = BN_new();
        BN_bin2bn(e2data, e2len, dmq1);

        # 创建一个新的大数对象 iqmp，并将 coeffdata 转换为大数对象
        iqmp = BN_new();
        BN_bin2bn(coeffdata, coefflen, iqmp);
    }

    # 为 RSA 对象分配内存空间
    *rsa = RSA_new();
#ifdef HAVE_OPAQUE_STRUCTS
    # 如果支持不透明结构，则使用 RSA_set0_key 函数设置 RSA 对象的公钥、私钥和模数
    RSA_set0_key(*rsa, n, e, d);
#else
    # 否则直接设置 RSA 对象的公钥、私钥和模数
    (*rsa)->e = e;
    (*rsa)->n = n;
    (*rsa)->d = d;
#endif

#ifdef HAVE_OPAQUE_STRUCTS
    # 如果支持不透明结构，则使用 RSA_set0_factors 函数设置 RSA 对象的因子 p 和 q
    RSA_set0_factors(*rsa, p, q);
#else
    # 否则直接设置 RSA 对象的因子 p 和 q
    (*rsa)->p = p;
    (*rsa)->q = q;
#endif

#ifdef HAVE_OPAQUE_STRUCTS
    # 如果支持不透明结构，则使用 RSA_set0_crt_params 函数设置 RSA 对象的 CRT 参数
    RSA_set0_crt_params(*rsa, dmp1, dmq1, iqmp);
#else
    # 否则直接设置 RSA 对象的 CRT 参数
    (*rsa)->dmp1 = dmp1;
    (*rsa)->dmq1 = dmq1;
    (*rsa)->iqmp = iqmp;
#endif
    # 返回成功
    return 0;
}

int
_libssh2_rsa_sha1_verify(libssh2_rsa_ctx * rsactx,
                         const unsigned char *sig,
                         unsigned long sig_len,
                         const unsigned char *m, unsigned long m_len)
{
    # 计算消息 m 的 SHA1 哈希值
    unsigned char hash[SHA_DIGEST_LENGTH];
    int ret;

    if(_libssh2_sha1(m, m_len, hash))
        return -1; /* 失败 */
    # 使用 RSA_verify 函数验证签名
    ret = RSA_verify(NID_sha1, hash, SHA_DIGEST_LENGTH,
                     (unsigned char *) sig, sig_len, rsactx);
    return (ret == 1) ? 0 : -1;
}

#if LIBSSH2_DSA
int
_libssh2_dsa_new(libssh2_dsa_ctx ** dsactx,
                 const unsigned char *p,
                 unsigned long p_len,
                 const unsigned char *q,
                 unsigned long q_len,
                 const unsigned char *g,
                 unsigned long g_len,
                 const unsigned char *y,
                 unsigned long y_len,
                 const unsigned char *x, unsigned long x_len)
{
    BIGNUM * p_bn;
    BIGNUM * q_bn;
    BIGNUM * g_bn;
    BIGNUM * pub_key;
    BIGNUM * priv_key = NULL;

    # 创建 BIGNUM 对象并将字节数据转换为大数
    p_bn = BN_new();
    BN_bin2bn(p, p_len, p_bn);

    q_bn = BN_new();
    BN_bin2bn(q, q_len, q_bn);

    g_bn = BN_new();
    BN_bin2bn(g, g_len, g_bn);

    pub_key = BN_new();
    BN_bin2bn(y, y_len, pub_key);

    if(x_len) {
        priv_key = BN_new();
        BN_bin2bn(x, x_len, priv_key);
    }

    # 创建 DSA 对象
    *dsactx = DSA_new();

#ifdef HAVE_OPAQUE_STRUCTS
    # 如果支持不透明结构，则使用 DSA_set0_pqg 函数设置 DSA 对象的参数 p、q 和 g
    DSA_set0_pqg(*dsactx, p_bn, q_bn, g_bn);
#else
    # 否则直接设置 DSA 对象的参数 p、q 和 g
    (*dsactx)->p = p_bn;
    (*dsactx)->g = g_bn;
    (*dsactx)->q = q_bn;
#endif
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果支持不透明结构体，则设置 DSA 的公钥和私钥
    DSA_set0_key(*dsactx, pub_key, priv_key);
#else
    // 如果不支持不透明结构体，则直接设置 DSA 的公钥和私钥
    (*dsactx)->pub_key = pub_key;
    (*dsactx)->priv_key = priv_key;
#endif
    // 返回成功
    return 0;
}

int
_libssh2_dsa_sha1_verify(libssh2_dsa_ctx * dsactx,
                         const unsigned char *sig,
                         const unsigned char *m, unsigned long m_len)
{
    // 声明变量
    unsigned char hash[SHA_DIGEST_LENGTH];
    DSA_SIG * dsasig;
    BIGNUM * r;
    BIGNUM * s;
    int ret = -1;

    // 初始化变量
    r = BN_new();
    BN_bin2bn(sig, 20, r);
    s = BN_new();
    BN_bin2bn(sig + 20, 20, s);

    dsasig = DSA_SIG_new();
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果支持不透明结构体，则设置 DSA 签名的 r 和 s
    DSA_SIG_set0(dsasig, r, s);
#else
    // 如果不支持不透明结构体，则直接设置 DSA 签名的 r 和 s
    dsasig->r = r;
    dsasig->s = s;
#endif
    // 计算消息的 SHA1 哈希值
    if(!_libssh2_sha1(m, m_len, hash))
        /* _libssh2_sha1() succeeded */
        // 验证 DSA 签名
        ret = DSA_do_verify(hash, SHA_DIGEST_LENGTH, dsasig, dsactx);

    // 释放 DSA 签名对象
    DSA_SIG_free(dsasig);

    // 返回验证结果
    return (ret == 1) ? 0 : -1;
}
#endif /* LIBSSH_DSA */

#if LIBSSH2_ECDSA

/* _libssh2_ecdsa_get_curve_type
 *
 * 返回与 libssh2_curve_type 对应的密钥曲线类型
 *
 */

libssh2_curve_type
_libssh2_ecdsa_get_curve_type(libssh2_ecdsa_ctx *ec_ctx)
{
    // 获取 ECDSA 密钥的曲线类型
    const EC_GROUP *group = EC_KEY_get0_group(ec_ctx);
    return EC_GROUP_get_curve_name(group);
}

/* _libssh2_ecdsa_curve_type_from_name
 *
 * 返回成功的情况下为 0，同时返回与 libssh2_curve_type 对应的密钥曲线类型
 *
 */

int
_libssh2_ecdsa_curve_type_from_name(const char *name,
                                    libssh2_curve_type *out_type)
{
    // 声明变量
    int ret = 0;
    libssh2_curve_type type;

    // 检查输入的曲线名称是否合法
    if(name == NULL || strlen(name) != 19)
        return -1;

    // 根据曲线名称设置对应的密钥曲线类型
    if(strcmp(name, "ecdsa-sha2-nistp256") == 0)
        type = LIBSSH2_EC_CURVE_NISTP256;
    else if(strcmp(name, "ecdsa-sha2-nistp384") == 0)
        type = LIBSSH2_EC_CURVE_NISTP384;
    else if(strcmp(name, "ecdsa-sha2-nistp521") == 0)
        type = LIBSSH2_EC_CURVE_NISTP521;
    else {
        ret = -1;
    }

    // 如果设置成功且输出参数不为空，则将密钥曲线类型赋值给输出参数
    if(ret == 0 && out_type) {
        *out_type = type;
    }

    // 返回结果
    return ret;
}
/* _libssh2_ecdsa_curve_name_with_octal_new
 *
 * 创建一个新的公钥，给定一个八进制字符串、长度和类型
 *
 */

int
_libssh2_ecdsa_curve_name_with_octal_new(libssh2_ecdsa_ctx ** ec_ctx,
     const unsigned char *k,
     size_t k_len, libssh2_curve_type curve)
{

    int ret = 0;
    const EC_GROUP *ec_group = NULL;
    // 通过曲线名称创建一个新的 EC_KEY 对象
    EC_KEY *ec_key = EC_KEY_new_by_curve_name(curve);
    EC_POINT *point = NULL;

    if(ec_key) {
        // 获取 EC_KEY 对象的 EC_GROUP
        ec_group = EC_KEY_get0_group(ec_key);
        // 创建一个新的 EC_POINT 对象
        point = EC_POINT_new(ec_group);
        // 将八进制字符串转换为 EC_POINT 对象
        ret = EC_POINT_oct2point(ec_group, point, k, k_len, NULL);
        // 设置 EC_KEY 对象的公钥
        ret = EC_KEY_set_public_key(ec_key, point);

        if(point != NULL)
            // 释放 EC_POINT 对象
            EC_POINT_free(point);

        if(ec_ctx != NULL)
            // 将 EC_KEY 对象赋值给 ec_ctx
            *ec_ctx = ec_key;
    }

    return (ret == 1) ? 0 : -1;
}

// 定义一个宏，用于验证 ECDSA 签名
#define LIBSSH2_ECDSA_VERIFY(digest_type)                           \
{                                                                   \
    unsigned char hash[SHA##digest_type##_DIGEST_LENGTH];           \
    libssh2_sha##digest_type(m, m_len, hash);                       \
    // 执行 ECDSA 签名验证
    ret = ECDSA_do_verify(hash, SHA##digest_type##_DIGEST_LENGTH,   \
      ecdsa_sig, ec_key);                                           \
                                                                    \
}

int
_libssh2_ecdsa_verify(libssh2_ecdsa_ctx * ctx,
      const unsigned char *r, size_t r_len,
      const unsigned char *s, size_t s_len,
      const unsigned char *m, size_t m_len)
{
    int ret = 0;
    EC_KEY *ec_key = (EC_KEY*)ctx;
    // 获取 EC_KEY 对象的曲线类型
    libssh2_curve_type type = _libssh2_ecdsa_get_curve_type(ec_key);

#ifdef HAVE_OPAQUE_STRUCTS
    // 创建一个 ECDSA_SIG 对象
    ECDSA_SIG *ecdsa_sig = ECDSA_SIG_new();
    BIGNUM *pr = BN_new();
    BIGNUM *ps = BN_new();
    // 将 r 和 s 转换为 BIGNUM 对象
    BN_bin2bn(r, r_len, pr);
    BN_bin2bn(s, s_len, ps);
    // 设置 ECDSA_SIG 对象的值
    ECDSA_SIG_set0(ecdsa_sig, pr, ps);

#else
    ECDSA_SIG ecdsa_sig_;
    ECDSA_SIG *ecdsa_sig = &ecdsa_sig_;
    ecdsa_sig_.r = BN_new();
    BN_bin2bn(r, r_len, ecdsa_sig_.r);
    ecdsa_sig_.s = BN_new();
    # 将二进制数据转换为大数形式，并存储到ecdsa_sig_.s中
    BN_bin2bn(s, s_len, ecdsa_sig_.s);
#endif

    // 如果类型是 NISTP256，则调用 LIBSSH2_ECDSA_VERIFY 函数，参数为 256
    if(type == LIBSSH2_EC_CURVE_NISTP256) {
        LIBSSH2_ECDSA_VERIFY(256);
    }
    // 如果类型是 NISTP384，则调用 LIBSSH2_ECDSA_VERIFY 函数，参数为 384
    else if(type == LIBSSH2_EC_CURVE_NISTP384) {
        LIBSSH2_ECDSA_VERIFY(384);
    }
    // 如果类型是 NISTP521，则调用 LIBSSH2_ECDSA_VERIFY 函数，参数为 512
    else if(type == LIBSSH2_EC_CURVE_NISTP521) {
        LIBSSH2_ECDSA_VERIFY(512);
    }

#ifdef HAVE_OPAQUE_STRUCTS
    // 如果 ecdsa_sig 存在，则释放其内存
    if(ecdsa_sig)
        ECDSA_SIG_free(ecdsa_sig);
#else
    // 否则，清空 ecdsa_sig_.s 和 ecdsa_sig_.r 的内存
    BN_clear_free(ecdsa_sig_.s);
    BN_clear_free(ecdsa_sig_.r);
#endif

    // 如果 ret 等于 1，则返回 0，否则返回 -1
    return (ret == 1) ? 0 : -1;
}

#endif /* LIBSSH2_ECDSA */

// 初始化加密算法
int
_libssh2_cipher_init(_libssh2_cipher_ctx * h,
                     _libssh2_cipher_type(algo),
                     unsigned char *iv, unsigned char *secret, int encrypt)
{
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果支持不透明结构，则创建新的 EVP_CIPHER_CTX 对象
    *h = EVP_CIPHER_CTX_new();
    // 调用 EVP_CipherInit 函数进行初始化，并返回结果
    return !EVP_CipherInit(*h, algo(), secret, iv, encrypt);
#else
    // 否则，初始化 EVP_CIPHER_CTX 对象
    EVP_CIPHER_CTX_init(h);
    // 调用 EVP_CipherInit 函数进行初始化，并返回结果
    return !EVP_CipherInit(h, algo(), secret, iv, encrypt);
#endif
}

// 加密或解密数据
int
_libssh2_cipher_crypt(_libssh2_cipher_ctx * ctx,
                      _libssh2_cipher_type(algo),
                      int encrypt, unsigned char *block, size_t blocksize)
{
    unsigned char buf[EVP_MAX_BLOCK_LENGTH];
    int ret;
    (void) algo;
    (void) encrypt;

#ifdef HAVE_OPAQUE_STRUCTS
    // 如果支持不透明结构，则调用 EVP_Cipher 函数进行加密或解密
    ret = EVP_Cipher(*ctx, buf, block, blocksize);
#else
    // 否则，调用 EVP_Cipher 函数进行加密或解密
    ret = EVP_Cipher(ctx, buf, block, blocksize);
#endif
    // 如果 OpenSSL 版本大于等于 3，则判断返回值是否不为 -1
#if defined(OPENSSL_VERSION_MAJOR) && OPENSSL_VERSION_MAJOR >= 3
    if(ret != -1) {
#else
    // 否则，判断返回值是否为 1
    if(ret == 1) {
#endif
        // 如果条件成立，则将 buf 中的数据复制到 block 中
        memcpy(block, buf, blocksize);
    }

    // 如果 OpenSSL 版本大于等于 3，则返回值不为 -1 则返回 0，否则返回 1
#if defined(OPENSSL_VERSION_MAJOR) && OPENSSL_VERSION_MAJOR >= 3
    return ret != -1 ? 0 : 1;
#else
    // 否则，返回值为 1 则返回 0，否则返回 1
    return ret == 1 ? 0 : 1;
#endif
}

// 如果支持 AES_CTR 且未定义 HAVE_EVP_AES_128_CTR，则包含以下代码
#if LIBSSH2_AES_CTR && !defined(HAVE_EVP_AES_128_CTR)

#include <openssl/aes.h>
#include <openssl/evp.h>

typedef struct
{
    AES_KEY       key;
    EVP_CIPHER_CTX *aes_ctx;
    unsigned char ctr[AES_BLOCK_SIZE];
} aes_ctr_ctx;

static EVP_CIPHER * aes_128_ctr_cipher = NULL;
static EVP_CIPHER * aes_192_ctr_cipher = NULL;
static EVP_CIPHER * aes_256_ctr_cipher = NULL;

static int
/* 初始化 AES-CTR 加密上下文，设置密钥和初始向量，以及加密/解密标志 */
aes_ctr_init(EVP_CIPHER_CTX *ctx, const unsigned char *key,
             const unsigned char *iv, int enc) /* init key */
{
    /*
     * 变量 "c" 从这个作用域泄漏出去，但稍后在 aes_ctr_cleanup 中被释放
     */
    aes_ctr_ctx *c;  // 定义 AES-CTR 上下文指针
    const EVP_CIPHER *aes_cipher;  // 定义 EVP 加密算法指针
    (void) enc;  // 防止编译器警告

    switch(EVP_CIPHER_CTX_key_length(ctx)) {
    case 16:
        aes_cipher = EVP_aes_128_ecb();  // 选择 AES-128-ECB 加密算法
        break;
    case 24:
        aes_cipher = EVP_aes_192_ecb();  // 选择 AES-192-ECB 加密算法
        break;
    case 32:
        aes_cipher = EVP_aes_256_ecb();  // 选择 AES-256-ECB 加密算法
        break;
    default:
        return 0;  // 密钥长度不符合要求，返回错误
    }

    c = malloc(sizeof(*c));  // 分配 AES-CTR 上下文内存
    if(c == NULL)
        return 0;  // 分配内存失败，返回错误

#ifdef HAVE_OPAQUE_STRUCTS
    c->aes_ctx = EVP_CIPHER_CTX_new();  // 创建 EVP 加密上下文
#else
    c->aes_ctx = malloc(sizeof(EVP_CIPHER_CTX));  // 分配 EVP 加密上下文内存
#endif
    if(c->aes_ctx == NULL) {
        free(c);  // 分配内存失败，释放 AES-CTR 上下文内存
        return 0;  // 返回错误
    }

    if(EVP_EncryptInit(c->aes_ctx, aes_cipher, key, NULL) != 1) {
#ifdef HAVE_OPAQUE_STRUCTS
        EVP_CIPHER_CTX_free(c->aes_ctx);  // 释放 EVP 加密上下文内存
#else
        free(c->aes_ctx);  // 释放 EVP 加密上下文内存
#endif
        free(c);  // 释放 AES-CTR 上下文内存
        return 0;  // 返回错误
    }

    EVP_CIPHER_CTX_set_padding(c->aes_ctx, 0);  // 设置 EVP 加密上下文不使用填充

    memcpy(c->ctr, iv, AES_BLOCK_SIZE);  // 复制初始向量到 AES-CTR 上下文

    EVP_CIPHER_CTX_set_app_data(ctx, c);  // 设置应用数据为 AES-CTR 上下文

    return 1;  // 返回成功
}

static int
aes_ctr_do_cipher(EVP_CIPHER_CTX *ctx, unsigned char *out,
                  const unsigned char *in,
                  size_t inl) /* encrypt/decrypt data */
{
    aes_ctr_ctx *c = EVP_CIPHER_CTX_get_app_data(ctx);  // 获取 AES-CTR 上下文
    unsigned char b1[AES_BLOCK_SIZE];  // 定义一个 AES 块大小的缓冲区
    int outlen = 0;  // 输出数据长度初始化为 0

    if(inl != 16) /* libssh2 only ever encrypt one block */
        return 0;  // 输入数据长度不符合要求，返回错误

    if(c == NULL) {
        return 0;  // AES-CTR 上下文为空，返回错误
    }

    /*
    加密一个数据包 P=P1||P2||...||Pn（其中 P1、P2、...、Pn 每个都是长度为 L 的块），
    加密器首先使用 <cipher> 加密 <X> 以获得块 B1。然后将块 B1 与 P1 进行异或运算，
    生成密文块 C1。然后递增计数器 X
    */
}
    # 如果使用 AES 加密算法对输入数据进行加密更新，将结果存储在 b1 中，并返回加密后的数据长度
    if(EVP_EncryptUpdate(c->aes_ctx, b1, &outlen,
                         c->ctr, AES_BLOCK_SIZE) != 1) {
        # 如果加密更新失败，返回 0
        return 0;
    }

    # 对输入数据进行异或操作，将结果存储在 out 中
    _libssh2_xor_data(out, in, b1, AES_BLOCK_SIZE);
    # 对 AES 计数器进行增量操作
    _libssh2_aes_ctr_increment(c->ctr, AES_BLOCK_SIZE);

    # 返回 1，表示加密成功
    return 1;
static int
aes_ctr_cleanup(EVP_CIPHER_CTX *ctx) /* cleanup ctx */
{
    // 从上下文中获取应用数据
    aes_ctr_ctx *c = EVP_CIPHER_CTX_get_app_data(ctx);

    // 如果应用数据为空，返回1
    if(c == NULL) {
        return 1;
    }

    // 如果 AES 上下文不为空
    if(c->aes_ctx != NULL) {
#ifdef HAVE_OPAQUE_STRUCTS
        // 释放 AES 上下文
        EVP_CIPHER_CTX_free(c->aes_ctx);
#else
        // 调用自定义的 AES 清理函数
        _libssh2_cipher_dtor(c->aes_ctx);
        // 释放 AES 上下文内存
        free(c->aes_ctx);
#endif
    }

    // 释放应用数据内存
    free(c);

    // 返回1
    return 1;
}

static const EVP_CIPHER *
make_ctr_evp (size_t keylen, EVP_CIPHER **aes_ctr_cipher, int type)
{
#ifdef HAVE_OPAQUE_STRUCTS
    // 创建新的 EVP_CIPHER 对象
    *aes_ctr_cipher = EVP_CIPHER_meth_new(type, 16, keylen);
    // 如果成功创建
    if(*aes_ctr_cipher) {
        // 设置 IV 长度
        EVP_CIPHER_meth_set_iv_length(*aes_ctr_cipher, 16);
        // 设置初始化函数
        EVP_CIPHER_meth_set_init(*aes_ctr_cipher, aes_ctr_init);
        // 设置加密函数
        EVP_CIPHER_meth_set_do_cipher(*aes_ctr_cipher, aes_ctr_do_cipher);
        // 设置清理函数
        EVP_CIPHER_meth_set_cleanup(*aes_ctr_cipher, aes_ctr_cleanup);
    }
#else
    // 设置 AES_CIPHER 对象的属性
    (*aes_ctr_cipher)->nid = type;
    (*aes_ctr_cipher)->block_size = 16;
    (*aes_ctr_cipher)->key_len = keylen;
    (*aes_ctr_cipher)->iv_len = 16;
    (*aes_ctr_cipher)->init = aes_ctr_init;
    (*aes_ctr_cipher)->do_cipher = aes_ctr_do_cipher;
    (*aes_ctr_cipher)->cleanup = aes_ctr_cleanup;
#endif

    // 返回 AES_CIPHER 对象
    return *aes_ctr_cipher;
}

const EVP_CIPHER *
_libssh2_EVP_aes_128_ctr(void)
{
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果 aes_128_ctr_cipher 为空
    return !aes_128_ctr_cipher ?
        // 创建 AES 128 CTR EVP 对象
        make_ctr_evp(16, &aes_128_ctr_cipher, NID_aes_128_ctr) :
        // 返回已存在的 AES 128 CTR EVP 对象
        aes_128_ctr_cipher;
#else
    static EVP_CIPHER aes_ctr_cipher;
    // 如果 aes_128_ctr_cipher 为空
    if(!aes_128_ctr_cipher) {
        // 设置 aes_128_ctr_cipher 为 AES CTR EVP 对象
        aes_128_ctr_cipher = &aes_ctr_cipher;
        // 创建 AES 128 CTR EVP 对象
        make_ctr_evp(16, &aes_128_ctr_cipher, 0);
    }
    // 返回 AES 128 CTR EVP 对象
    return aes_128_ctr_cipher;
#endif
}

const EVP_CIPHER *
_libssh2_EVP_aes_192_ctr(void)
{
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果 aes_192_ctr_cipher 为空
    return !aes_192_ctr_cipher ?
        // 创建 AES 192 CTR EVP 对象
        make_ctr_evp(24, &aes_192_ctr_cipher, NID_aes_192_ctr) :
        // 返回已存在的 AES 192 CTR EVP 对象
        aes_192_ctr_cipher;
#else
    static EVP_CIPHER aes_ctr_cipher;
    # 如果 aes_192_ctr_cipher 为空，则执行以下操作
    if(!aes_192_ctr_cipher) {
        # 将 aes_ctr_cipher 的地址赋值给 aes_192_ctr_cipher
        aes_192_ctr_cipher = &aes_ctr_cipher;
        # 调用 make_ctr_evp 函数，传入密钥长度为 24，aes_192_ctr_cipher 的地址，和 0 作为参数
        make_ctr_evp(24, &aes_192_ctr_cipher, 0);
    }
    # 返回 aes_192_ctr_cipher
    return aes_192_ctr_cipher;
#endif
}

const EVP_CIPHER *
_libssh2_EVP_aes_256_ctr(void)
{
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果不支持透明结构，则返回 AES-256-CTR 加密算法
    return !aes_256_ctr_cipher ?
        make_ctr_evp(32, &aes_256_ctr_cipher, NID_aes_256_ctr) :
        aes_256_ctr_cipher;
#else
    // 如果支持透明结构，则创建静态的 AES-CTR 加密算法对象
    static EVP_CIPHER aes_ctr_cipher;
    if(!aes_256_ctr_cipher) {
        aes_256_ctr_cipher = &aes_ctr_cipher;
        make_ctr_evp(32, &aes_256_ctr_cipher, 0);
    }
    return aes_256_ctr_cipher;
#endif
}

#endif /* LIBSSH2_AES_CTR && !defined(HAVE_EVP_AES_128_CTR) */

void _libssh2_openssl_crypto_init(void)
{
#if OPENSSL_VERSION_NUMBER >= 0x10100000L && \
    !defined(LIBRESSL_VERSION_NUMBER)
#ifndef OPENSSL_NO_ENGINE
    // 如果 OpenSSL 版本大于等于 1.1.0，则加载内置引擎并注册所有完整的引擎
    ENGINE_load_builtin_engines();
    ENGINE_register_all_complete();
#endif
#else
    // 如果 OpenSSL 版本小于 1.1.0，则添加所有算法、密码和摘要
    OpenSSL_add_all_algorithms();
    OpenSSL_add_all_ciphers();
    OpenSSL_add_all_digests();
#ifndef OPENSSL_NO_ENGINE
    // 如果不禁用引擎，则加载内置引擎并注册所有完整的引擎
    ENGINE_load_builtin_engines();
    ENGINE_register_all_complete();
#endif
#endif
#if LIBSSH2_AES_CTR && !defined(HAVE_EVP_AES_128_CTR)
    // 如果支持 AES-CTR 且未定义 EVP_AES_128_CTR，则分别获取 AES-128-CTR、AES-192-CTR 和 AES-256-CTR 加密算法
    aes_128_ctr_cipher = (EVP_CIPHER *) _libssh2_EVP_aes_128_ctr();
    aes_192_ctr_cipher = (EVP_CIPHER *) _libssh2_EVP_aes_192_ctr();
    aes_256_ctr_cipher = (EVP_CIPHER *) _libssh2_EVP_aes_256_ctr();
#endif
}

void _libssh2_openssl_crypto_exit(void)
{
#if LIBSSH2_AES_CTR && !defined(HAVE_EVP_AES_128_CTR)
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果支持透明结构，则释放 AES-128-CTR、AES-192-CTR 和 AES-256-CTR 加密算法对象
    if(aes_128_ctr_cipher) {
        EVP_CIPHER_meth_free(aes_128_ctr_cipher);
    }

    if(aes_192_ctr_cipher) {
        EVP_CIPHER_meth_free(aes_192_ctr_cipher);
    }

    if(aes_256_ctr_cipher) {
        EVP_CIPHER_meth_free(aes_256_ctr_cipher);
    }
#endif

    // 将 AES-128-CTR、AES-192-CTR 和 AES-256-CTR 加密算法对象置空
    aes_128_ctr_cipher = NULL;
    aes_192_ctr_cipher = NULL;
    aes_256_ctr_cipher = NULL;
#endif
}

/* TODO: Optionally call a passphrase callback specified by the
 * calling program
 */
static int
passphrase_cb(char *buf, int size, int rwflag, char *passphrase)
{
    // 获取口令长度
    int passphrase_len = strlen(passphrase);
    (void) rwflag;
    # 如果密码长度大于（缓冲区大小-1），则将密码长度设置为缓冲区大小-1
    if(passphrase_len > (size - 1)) {
        passphrase_len = size - 1;
    }
    # 将密码拷贝到缓冲区中
    memcpy(buf, passphrase, passphrase_len);
    # 在缓冲区末尾添加字符串结束符
    buf[passphrase_len] = '\0';
    # 返回密码长度
    return passphrase_len;
# 结束函数定义
}

# 定义一个函数指针类型，指向一个函数，该函数接受四个参数并返回一个指向 void 类型的指针
typedef void * (*pem_read_bio_func)(BIO *, void **, pem_password_cb *, void *u);

# 从内存中读取私钥
static int
read_private_key_from_memory(void **key_ctx,
                             pem_read_bio_func read_private_key,
                             const char *filedata,
                             size_t filedata_len,
                             unsigned const char *passphrase)
{
    BIO * bp;

    # 初始化 key_ctx 为 NULL
    *key_ctx = NULL;

    # 从内存中创建一个 BIO 对象
    bp = BIO_new_mem_buf((char *)filedata, filedata_len);
    if(!bp) {
        return -1;
    }
    # 从 BIO 对象中读取私钥
    *key_ctx = read_private_key(bp, NULL, (pem_password_cb *) passphrase_cb, (void *) passphrase);

    # 释放 BIO 对象
    BIO_free(bp);
    return (*key_ctx) ? 0 : -1;
}

# 从文件中读取私钥
static int
read_private_key_from_file(void **key_ctx,
                           pem_read_bio_func read_private_key,
                           const char *filename,
                           unsigned const char *passphrase)
{
    BIO * bp;

    # 初始化 key_ctx 为 NULL
    *key_ctx = NULL;

    # 从文件中创建一个 BIO 对象
    bp = BIO_new_file(filename, "r");
    if(!bp) {
        return -1;
    }

    # 从 BIO 对象中读取私钥
    *key_ctx = read_private_key(bp, NULL, (pem_password_cb *) passphrase_cb, (void *) passphrase);

    # 释放 BIO 对象
    BIO_free(bp);
    return (*key_ctx) ? 0 : -1;
}

# 从内存中创建一个新的 RSA 私钥
int
_libssh2_rsa_new_private_frommemory(libssh2_rsa_ctx ** rsa,
                                    LIBSSH2_SESSION * session,
                                    const char *filedata, size_t filedata_len,
                                    unsigned const char *passphrase)
{
    int rc;

    # 定义一个函数指针，指向一个函数，该函数用于从 BIO 对象中读取 RSA 私钥
    pem_read_bio_func read_rsa = (pem_read_bio_func) &PEM_read_bio_RSAPrivateKey;

    # 初始化 libssh2 库
    _libssh2_init_if_needed();

    # 从内存中读取私钥
    rc = read_private_key_from_memory((void **) rsa, read_rsa, filedata, filedata_len, passphrase);

    # 如果读取失败，则尝试从内存中读取 OpenSSH 私钥
    if(rc) {
        rc = read_openssh_private_key_from_memory((void **)rsa, session, "ssh-rsa", filedata, filedata_len, passphrase);
    }

    return rc;
}
static unsigned char *
gen_publickey_from_rsa(LIBSSH2_SESSION *session, RSA *rsa,
                       size_t *key_len)
{
    int            e_bytes, n_bytes;  // 声明变量用于存储 e 和 n 的字节数
    unsigned long  len;  // 声明变量用于存储公钥的总长度
    unsigned char *key;  // 声明指针变量用于存储生成的公钥
    unsigned char *p;  // 声明指针变量用于处理公钥编码
    const BIGNUM * e;  // 声明指针变量用于存储 RSA 公钥的指数 e
    const BIGNUM * n;  // 声明指针变量用于存储 RSA 公钥的模数 n
#ifdef HAVE_OPAQUE_STRUCTS
    RSA_get0_key(rsa, &n, &e, NULL);  // 获取 RSA 公钥的模数和指数
#else
    e = rsa->e;  // 获取 RSA 公钥的指数 e
    n = rsa->n;  // 获取 RSA 公钥的模数 n
#endif
    e_bytes = BN_num_bytes(e) + 1;  // 计算指数 e 的字节数
    n_bytes = BN_num_bytes(n) + 1;  // 计算模数 n 的字节数

    /* Key form is "ssh-rsa" + e + n. */
    len = 4 + 7 + 4 + e_bytes + 4 + n_bytes;  // 计算公钥的总长度

    key = LIBSSH2_ALLOC(session, len);  // 分配存储公钥的内存空间
    if(key == NULL) {
        return NULL;  // 如果内存分配失败，则返回空指针
    }

    /* Process key encoding. */
    p = key;  // 指针 p 指向公钥的起始位置

    _libssh2_htonu32(p, 7);  /* Key type. */  // 将 7 转换为网络字节序后存入 p 指向的位置
    p += 4;  // 指针 p 向后移动 4 个字节
    memcpy(p, "ssh-rsa", 7);  // 将字符串 "ssh-rsa" 复制到 p 指向的位置
    p += 7;  // 指针 p 向后移动 7 个字节

    p = write_bn(p, e, e_bytes);  // 将指数 e 编码后存入 p 指向的位置
    p = write_bn(p, n, n_bytes);  // 将模数 n 编码后存入 p 指向的位置

    *key_len = (size_t)(p - key);  // 计算生成的公钥的长度
    return key;  // 返回生成的公钥
}

static int
gen_publickey_from_rsa_evp(LIBSSH2_SESSION *session,
                           unsigned char **method,
                           size_t *method_len,
                           unsigned char **pubkeydata,
                           size_t *pubkeydata_len,
                           EVP_PKEY *pk)
{
    RSA*           rsa = NULL;  // 声明 RSA 结构体指针
    unsigned char *key;  // 声明指针变量用于存储生成的公钥
    unsigned char *method_buf = NULL;  // 声明指针变量用于存储公钥的方法
    size_t  key_len;  // 声明变量用于存储生成的公钥的长度

    _libssh2_debug(session,
                   LIBSSH2_TRACE_AUTH,
                   "Computing public key from RSA private key envelope");  // 调试信息

    rsa = EVP_PKEY_get1_RSA(pk);  // 从 EVP_PKEY 结构体中获取 RSA 公钥
    if(rsa == NULL) {
        /* Assume memory allocation error... what else could it be ? */
        goto __alloc_error;  // 如果获取失败，则跳转到错误处理标签
    }

    method_buf = LIBSSH2_ALLOC(session, 7);  /* ssh-rsa. */  // 分配存储公钥方法的内存空间
    if(method_buf == NULL) {
        goto __alloc_error;  // 如果内存分配失败，则跳转到错误处理标签
    }

    key = gen_publickey_from_rsa(session, rsa, &key_len);  // 生成公钥
    if(key == NULL) {
        goto __alloc_error;  // 如果生成公钥失败，则跳转到错误处理标签
    }
    RSA_free(rsa);  // 释放 RSA 结构体占用的内存空间

    memcpy(method_buf, "ssh-rsa", 7);  // 将字符串 "ssh-rsa" 复制到 method_buf 指向的位置
    *method         = method_buf;  // 将公钥方法存入指定的位置
    *method_len     = 7;  // 设置公钥方法的长度为 7
    # 将指针 *pubkeydata 指向 key 变量
    *pubkeydata     = key;
    # 将指针 *pubkeydata_len 指向 key_len 变量
    *pubkeydata_len = key_len;
    # 返回 0，表示执行成功
    return 0;

  # 分配内存失败的错误处理
  __alloc_error:
    # 如果 rsa 指针不为空，释放 rsa 指向的内存
    if(rsa != NULL) {
        RSA_free(rsa);
    }
    # 如果 method_buf 指针不为空，释放 method_buf 指向的内存
    if(method_buf != NULL) {
        LIBSSH2_FREE(session, method_buf);
    }

    # 返回内存分配错误，并提供错误信息
    return _libssh2_error(session,
                          LIBSSH2_ERROR_ALLOC,
                          "Unable to allocate memory for private key data");
# 定义一个静态函数，用于为 RSA 密钥生成额外的参数
static int _libssh2_rsa_new_additional_parameters(RSA *rsa)
{
    # 定义大数运算的上下文
    BN_CTX *ctx = NULL;
    # 定义大数变量
    BIGNUM *aux = NULL;
    BIGNUM *dmp1 = NULL;
    BIGNUM *dmq1 = NULL;
    const BIGNUM *p = NULL;
    const BIGNUM *q = NULL;
    const BIGNUM *d = NULL;
    int rc = 0;

    # 根据是否支持不透明结构来获取 RSA 密钥的参数
    # 如果支持不透明结构，则使用 RSA_get0_key 和 RSA_get0_factors 函数
    # 否则直接访问 RSA 结构体的成员变量
#ifdef HAVE_OPAQUE_STRUCTS
    RSA_get0_key(rsa, NULL, NULL, &d);
    RSA_get0_factors(rsa, &p, &q);
#else
    d = (*rsa).d;
    p = (*rsa).p;
    q = (*rsa).q;
#endif

    # 创建大数运算的上下文
    ctx = BN_CTX_new();
    if(ctx == NULL)
        return -1;

    # 创建大数变量
    aux = BN_new();
    if(aux == NULL) {
        rc = -1;
        goto out;
    }

    dmp1 = BN_new();
    if(dmp1 == NULL) {
        rc = -1;
        goto out;
    }

    dmq1 = BN_new();
    if(dmq1 == NULL) {
        rc = -1;
        goto out;
    }

    # 计算 dmp1 和 dmq1 参数
    if((BN_sub(aux, q, BN_value_one()) == 0) ||
        (BN_mod(dmq1, d, aux, ctx) == 0) ||
        (BN_sub(aux, p, BN_value_one()) == 0) ||
        (BN_mod(dmp1, d, aux, ctx) == 0)) {
        rc = -1;
        goto out;
    }

    # 根据是否支持不透明结构来设置 RSA 密钥的参数
#ifdef HAVE_OPAQUE_STRUCTS
    RSA_set0_crt_params(rsa, dmp1, dmq1, NULL);
#else
    (*rsa).dmp1 = dmp1;
    (*rsa).dmq1 = dmq1;
#endif

out:
    # 释放资源
    if(aux)
        BN_clear_free(aux);
    BN_CTX_free(ctx);

    if(rc != 0) {
        if(dmp1)
            BN_clear_free(dmp1);
        if(dmq1)
            BN_clear_free(dmq1);
    }

    return rc;
}

# 从 RSA OpenSSH 私钥数据生成公钥
static int
gen_publickey_from_rsa_openssh_priv_data(LIBSSH2_SESSION *session,
                                         struct string_buf *decrypted,
                                         unsigned char **method,
                                         size_t *method_len,
                                         unsigned char **pubkeydata,
                                         size_t *pubkeydata_len,
                                         libssh2_rsa_ctx **rsa_ctx)
{
    int rc = 0;
    size_t nlen, elen, dlen, plen, qlen, coefflen, commentlen;
    unsigned char *n, *e, *d, *p, *q, *coeff, *comment;
    RSA *rsa = NULL;
    # 输出调试信息，表示正在从私钥数据计算 RSA 密钥
    _libssh2_debug(session,
                   LIBSSH2_TRACE_AUTH,
                   "Computing RSA keys from private key data");

    # 获取公钥数据中的 n
    if(_libssh2_get_bignum_bytes(decrypted, &n, &nlen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "RSA no n");
        return -1;
    }

    # 获取公钥数据中的 e
    if(_libssh2_get_bignum_bytes(decrypted, &e, &elen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "RSA no e");
        return -1;
    }

    # 获取私钥数据中的 d
    if(_libssh2_get_bignum_bytes(decrypted, &d, &dlen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "RSA no d");
        return -1;
    }

    # 获取私钥数据中的 coeff
    if(_libssh2_get_bignum_bytes(decrypted, &coeff, &coefflen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "RSA no coeff");
        return -1;
    }

    # 获取私钥数据中的 p
    if(_libssh2_get_bignum_bytes(decrypted, &p, &plen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "RSA no p");
        return -1;
    }

    # 获取私钥数据中的 q
    if(_libssh2_get_bignum_bytes(decrypted, &q, &qlen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "RSA no q");
        return -1;
    }

    # 获取私钥数据中的注释信息
    if(_libssh2_get_string(decrypted, &comment, &commentlen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "RSA no comment");
        return -1;
    }

    # 使用获取的数据创建 RSA 私钥
    if((rc = _libssh2_rsa_new(&rsa, e, elen, n, nlen, d, dlen, p, plen,
                              q, qlen, NULL, 0, NULL, 0,
                              coeff, coefflen)) != 0) {
        _libssh2_debug(session,
                       LIBSSH2_TRACE_AUTH,
                       "Could not create RSA private key");
        goto fail;
    }

    # 如果成功创建了 RSA 私钥，则继续设置额外的参数
    if(rsa != NULL)
        rc = _libssh2_rsa_new_additional_parameters(rsa);
    # 检查 rsa、pubkeydata 和 method 是否都不为空
    if(rsa != NULL && pubkeydata != NULL && method != NULL) {
        # 创建一个新的 EVP_PKEY 对象
        EVP_PKEY *pk = EVP_PKEY_new();
        # 将 RSA 密钥设置到 EVP_PKEY 对象中
        EVP_PKEY_set1_RSA(pk, rsa);

        # 调用 gen_publickey_from_rsa_evp 函数生成公钥
        rc = gen_publickey_from_rsa_evp(session, method, method_len,
                                        pubkeydata, pubkeydata_len,
                                        pk);

        # 如果 pk 不为空，则释放 pk
        if(pk)
            EVP_PKEY_free(pk);
    }

    # 如果 rsa_ctx 不为空，则将 rsa 赋值给 *rsa_ctx，否则释放 rsa
    if(rsa_ctx != NULL)
        *rsa_ctx = rsa;
    else
        RSA_free(rsa);

    # 返回 rc
    return rc;
# 如果 RSA 对象不为空，则释放 RSA 对象
    if(rsa != NULL)
        RSA_free(rsa);

    # 返回内存分配错误，无法为私钥数据分配内存
    return _libssh2_error(session,
                          LIBSSH2_ERROR_ALLOC,
                          "Unable to allocate memory for private key data");
}

# 创建一个新的 OpenSSH 私钥
static int
_libssh2_rsa_new_openssh_private(libssh2_rsa_ctx ** rsa,
                                 LIBSSH2_SESSION * session,
                                 const char *filename,
                                 unsigned const char *passphrase)
{
    FILE *fp;
    int rc;
    unsigned char *buf = NULL;
    struct string_buf *decrypted = NULL;

    # 如果会话为空，则返回协议错误，需要会话
    if(session == NULL) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Session is required");
        return -1;
    }

    # 初始化 libssh2 库
    _libssh2_init_if_needed();

    # 打开文件，以只读方式
    fp = fopen(filename, "r");
    if(!fp) {
        # 返回文件错误，无法打开 OpenSSH RSA 私钥文件
        _libssh2_error(session, LIBSSH2_ERROR_FILE,
                       "Unable to open OpenSSH RSA private key file");
        return -1;
    }

    # 解析 OpenSSH PEM 格式的私钥文件
    rc = _libssh2_openssh_pem_parse(session, passphrase, fp, &decrypted);
    fclose(fp);
    if(rc) {
        return rc;
    }

    # 解析解密后的密钥数据，获取公钥类型
    rc = _libssh2_get_string(decrypted, &buf, NULL);

    # 如果解析失败或者 buf 为空，则返回协议错误，解密后的密钥数据中未找到公钥类型
    if(rc != 0 || buf == NULL) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Public key type in decrypted key data not found");
        return -1;
    }

    # 如果公钥类型为 ssh-rsa，则调用函数生成 RSA 公钥
    if(strcmp("ssh-rsa", (const char *)buf) == 0) {
        rc = gen_publickey_from_rsa_openssh_priv_data(session, decrypted,
                                                      NULL, 0,
                                                      NULL, 0, rsa);
    }
    else {
        rc = -1;
    }

    # 如果解密后的密钥数据不为空，则释放解密后的密钥数据
    if(decrypted)
        _libssh2_string_buf_free(session, decrypted);

    return rc;
}

# 创建一个新的私钥
int
_libssh2_rsa_new_private(libssh2_rsa_ctx ** rsa,
                         LIBSSH2_SESSION * session,
                         const char *filename, unsigned const char *passphrase)
{
    int rc;
    # 定义一个函数指针 read_rsa，指向 PEM_read_bio_RSAPrivateKey 函数
    pem_read_bio_func read_rsa =
        (pem_read_bio_func) &PEM_read_bio_RSAPrivateKey;

    # 初始化 libssh2 库
    _libssh2_init_if_needed();

    # 从文件中读取私钥，并存储到 rsa 指针指向的位置
    rc = read_private_key_from_file((void **) rsa, read_rsa,
                                    filename, passphrase);

    # 如果读取私钥失败
    if(rc) {
        # 尝试使用 openssh 格式的私钥文件创建新的 RSA 私钥
        rc = _libssh2_rsa_new_openssh_private(rsa, session,
                                              filename, passphrase);
    }

    # 返回操作结果
    return rc;
# 如果 LIBSSH2_DSA 宏定义存在，则定义 libssh2_dsa_new_private_frommemory 函数
int
_libssh2_dsa_new_private_frommemory(libssh2_dsa_ctx ** dsa,
                                    LIBSSH2_SESSION * session,
                                    const char *filedata, size_t filedata_len,
                                    unsigned const char *passphrase)
{
    int rc;  # 定义整型变量 rc

    pem_read_bio_func read_dsa =  # 定义函数指针 read_dsa，指向 PEM_read_bio_DSAPrivateKey 函数
        (pem_read_bio_func) &PEM_read_bio_DSAPrivateKey;

    _libssh2_init_if_needed();  # 调用 _libssh2_init_if_needed 函数

    rc = read_private_key_from_memory((void **)dsa, read_dsa,  # 调用 read_private_key_from_memory 函数，将结果赋值给 rc
                                      filedata, filedata_len, passphrase);

    if(rc) {  # 如果 rc 不为 0
        rc = read_openssh_private_key_from_memory((void **)dsa, session,  # 调用 read_openssh_private_key_from_memory 函数，将结果赋值给 rc
                            "ssh-dsa", filedata, filedata_len, passphrase);
    }

    return rc;  # 返回 rc
}

# 定义 gen_publickey_from_dsa 函数，生成 DSA 公钥
static unsigned char *
gen_publickey_from_dsa(LIBSSH2_SESSION* session, DSA *dsa,
                       size_t *key_len)
{
    int            p_bytes, q_bytes, g_bytes, k_bytes;  # 定义整型变量
    unsigned long  len;  # 定义无符号长整型变量 len
    unsigned char *key;  # 定义无符号字符指针 key
    unsigned char *p;  # 定义无符号字符指针 p

    const BIGNUM * p_bn;  # 定义指向 BIGNUM 结构体的常量指针
    const BIGNUM * q;  # 定义指向 BIGNUM 结构体的常量指针
    const BIGNUM * g;  # 定义指向 BIGNUM 结构体的常量指针
    const BIGNUM * pub_key;  # 定义指向 BIGNUM 结构体的常量指针
#ifdef HAVE_OPAQUE_STRUCTS
    DSA_get0_pqg(dsa, &p_bn, &q, &g);  # 如果 HAVE_OPAQUE_STRUCTS 宏定义存在，则调用 DSA_get0_pqg 函数
#else
    p_bn = dsa->p;  # 否则将 dsa 结构体中的 p 赋值给 p_bn
    q = dsa->q;  # 将 dsa 结构体中的 q 赋值给 q
    g = dsa->g;  # 将 dsa 结构体中的 g 赋值给 g
#endif

#ifdef HAVE_OPAQUE_STRUCTS
    DSA_get0_key(dsa, &pub_key, NULL);  # 如果 HAVE_OPAQUE_STRUCTS 宏定义存在，则调用 DSA_get0_key 函数
#else
    pub_key = dsa->pub_key;  # 否则将 dsa 结构体中的 pub_key 赋值给 pub_key
#endif
    p_bytes = BN_num_bytes(p_bn) + 1;  # 计算 p_bn 的字节数
    q_bytes = BN_num_bytes(q) + 1;  # 计算 q 的字节数
    g_bytes = BN_num_bytes(g) + 1;  # 计算 g 的字节数
    k_bytes = BN_num_bytes(pub_key) + 1;  # 计算 pub_key 的字节数

    /* Key form is "ssh-dss" + p + q + g + pub_key. */
    len = 4 + 7 + 4 + p_bytes + 4 + q_bytes + 4 + g_bytes + 4 + k_bytes;  # 计算公钥长度

    key = LIBSSH2_ALLOC(session, len);  # 分配内存空间给 key
    if(key == NULL) {  # 如果 key 为空
        return NULL;  # 返回空指针
    }

    /* Process key encoding. */
    p = key;  # 将 key 赋值给 p

    _libssh2_htonu32(p, 7);  /* Key type. */  # 调用 _libssh2_htonu32 函数，将结果赋值给 p
    p += 4;  # p 指针偏移 4 个字节
    memcpy(p, "ssh-dss", 7);  # 将 "ssh-dss" 复制到 p 指向的位置
    p += 7;  # p 指针偏移 7 个字节

    p = write_bn(p, p_bn, p_bytes);  # 调用 write_bn 函数，将结果赋值给 p
    p = write_bn(p, q, q_bytes);  # 调用 write_bn 函数，将结果赋值给 p
    p = write_bn(p, g, g_bytes);  # 调用 write_bn 函数，将结果赋值给 p
    p = write_bn(p, pub_key, k_bytes);  # 调用 write_bn 函数，将结果赋值给 p
    # 计算 key_len 的值，即指针 p 减去指针 key 的大小
    *key_len = (size_t)(p - key);
    # 返回 key 指针，即指向 key 字符串的指针
    return key;
# 从 DSA 私钥生成公钥
static int
gen_publickey_from_dsa_evp(LIBSSH2_SESSION *session,  # 传入会话对象
                           unsigned char **method,  # 公钥方法
                           size_t *method_len,  # 公钥方法长度
                           unsigned char **pubkeydata,  # 公钥数据
                           size_t *pubkeydata_len,  # 公钥数据长度
                           EVP_PKEY *pk)  # EVP 公钥对象
{
    DSA*           dsa = NULL;  # DSA 对象
    unsigned char *key;  # 公钥
    unsigned char *method_buf = NULL;  # 方法缓冲区
    size_t  key_len;  # 公钥长度

    _libssh2_debug(session,  # 调试信息
                   LIBSSH2_TRACE_AUTH,  # 认证跟踪
                   "Computing public key from DSA private key envelope");  # 计算公钥

    dsa = EVP_PKEY_get1_DSA(pk);  # 从 EVP 公钥对象中获取 DSA 对象
    if(dsa == NULL) {  # 如果 DSA 对象为空
        /* Assume memory allocation error... what else could it be ? */
        goto __alloc_error;  # 跳转到内存分配错误处理
    }

    method_buf = LIBSSH2_ALLOC(session, 7);  /* ssh-dss. */  # 分配方法缓冲区内存
    if(method_buf == NULL) {  # 如果方法缓冲区为空
        goto __alloc_error;  # 跳转到内存分配错误处理
    }

    key = gen_publickey_from_dsa(session, dsa, &key_len);  # 生成 DSA 公钥
    if(key == NULL) {  # 如果公钥为空
        goto __alloc_error;  # 跳转到内存分配错误处理
    }
    DSA_free(dsa);  # 释放 DSA 对象

    memcpy(method_buf, "ssh-dss", 7);  # 复制方法名称到方法缓冲区
    *method         = method_buf;  # 设置公钥方法
    *method_len     = 7;  # 设置公钥方法长度
    *pubkeydata     = key;  # 设置公钥数据
    *pubkeydata_len = key_len;  # 设置公钥数据长度
    return 0;  # 返回成功

  __alloc_error:  # 内存分配错误处理标签
    if(dsa != NULL) {  # 如果 DSA 对象不为空
        DSA_free(dsa);  # 释放 DSA 对象
    }
    if(method_buf != NULL) {  # 如果方法缓冲区不为空
        LIBSSH2_FREE(session, method_buf);  # 释放方法缓冲区内存
    }

    return _libssh2_error(session,  # 返回 libssh2 错误
                          LIBSSH2_ERROR_ALLOC,  # 分配错误
                          "Unable to allocate memory for private key data");  # 无法为私钥数据分配内存
}

static int
gen_publickey_from_dsa_openssh_priv_data(LIBSSH2_SESSION *session,  # 传入会话对象
                                         struct string_buf *decrypted,  # 解密后的字符串缓冲区
                                         unsigned char **method,  # 公钥方法
                                         size_t *method_len,  # 公钥方法长度
                                         unsigned char **pubkeydata,  # 公钥数据
                                         size_t *pubkeydata_len,  # 公钥数据长度
                                         libssh2_dsa_ctx **dsa_ctx)  # DSA 上下文
{
    int rc = 0;  # 返回码初始化为 0
    # 定义变量存储长度信息
    size_t plen, qlen, glen, pub_len, priv_len;
    # 定义指针变量存储密钥数据
    unsigned char *p, *q, *g, *pub_key, *priv_key;
    # 定义 DSA 结构体指针并初始化为 NULL
    DSA *dsa = NULL;

    # 输出调试信息
    _libssh2_debug(session,
                   LIBSSH2_TRACE_AUTH,
                   "Computing DSA keys from private key data");

    # 从解密后的数据中获取 p 的字节数和数据存储在 p 中
    if(_libssh2_get_bignum_bytes(decrypted, &p, &plen)) {
        # 输出错误信息
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "DSA no p");
        return -1;
    }

    # 从解密后的数据中获取 q 的字节数和数据存储在 q 中
    if(_libssh2_get_bignum_bytes(decrypted, &q, &qlen)) {
        # 输出错误信息
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "DSA no q");
        return -1;
    }

    # 从解密后的数据中获取 g 的字节数和数据存储在 g 中
    if(_libssh2_get_bignum_bytes(decrypted, &g, &glen)) {
        # 输出错误信息
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "DSA no g");
        return -1;
    }

    # 从解密后的数据中获取公钥的字节数和数据存储在 pub_key 中
    if(_libssh2_get_bignum_bytes(decrypted, &pub_key, &pub_len)) {
        # 输出错误信息
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "DSA no public key");
        return -1;
    }

    # 从解密后的数据中获取私钥的字节数和数据存储在 priv_key 中
    if(_libssh2_get_bignum_bytes(decrypted, &priv_key, &priv_len)) {
        # 输出错误信息
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "DSA no private key");
        return -1;
    }

    # 使用获取的数据创建 DSA 结构体
    rc = _libssh2_dsa_new(&dsa, p, plen, q, qlen, g, glen, pub_key, pub_len,
                          priv_key, priv_len);
    if(rc != 0) {
        # 输出错误信息
        _libssh2_debug(session,
                       LIBSSH2_ERROR_PROTO,
                       "Could not create DSA private key");
        # 跳转到失败处理部分
        goto fail;
    }

    # 如果 DSA 结构体和公钥数据不为空，则创建 EVP_PKEY 结构体并设置 DSA 结构体
    if(dsa != NULL && pubkeydata != NULL && method != NULL) {
        EVP_PKEY *pk = EVP_PKEY_new();
        EVP_PKEY_set1_DSA(pk, dsa);

        # 从 DSA 结构体创建公钥数据
        rc = gen_publickey_from_dsa_evp(session, method, method_len,
                                        pubkeydata, pubkeydata_len,
                                        pk);

        # 释放 EVP_PKEY 结构体
        if(pk)
            EVP_PKEY_free(pk);
    }

    # 如果 DSA 上下文不为空，则将 DSA 结构体赋值给上下文，否则释放 DSA 结构体
    if(dsa_ctx != NULL)
        *dsa_ctx = dsa;
    else
        DSA_free(dsa);

    # 返回结果
    return rc;
// 如果 DSA 对象不为空，则释放其内存
if(dsa != NULL)
    DSA_free(dsa);

// 返回内存分配失败的错误信息
return _libssh2_error(session,
                      LIBSSH2_ERROR_ALLOC,
                      "Unable to allocate memory for private key data");
}

// 从 OpenSSH 格式的私钥文件中创建 DSA 对象
static int
_libssh2_dsa_new_openssh_private(libssh2_dsa_ctx ** dsa,
                                 LIBSSH2_SESSION * session,
                                 const char *filename,
                                 unsigned const char *passphrase)
{
    FILE *fp;
    int rc;
    unsigned char *buf = NULL;
    struct string_buf *decrypted = NULL;

    // 检查会话是否为空，如果为空则返回错误信息
    if(session == NULL) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Session is required");
        return -1;
    }

    // 初始化 libssh2 库
    _libssh2_init_if_needed();

    // 打开文件，如果失败则返回错误信息
    fp = fopen(filename, "r");
    if(!fp) {
        _libssh2_error(session, LIBSSH2_ERROR_FILE,
                       "Unable to open OpenSSH DSA private key file");
        return -1;
    }

    // 解析 OpenSSH PEM 格式的私钥文件，如果失败则返回错误信息
    rc = _libssh2_openssh_pem_parse(session, passphrase, fp, &decrypted);
    fclose(fp);
    if(rc) {
        return rc;
    }

    // 尝试解析支持的公钥类型
    rc = _libssh2_get_string(decrypted, &buf, NULL);

    // 如果解析失败或者 buf 为空，则返回错误信息
    if(rc != 0 || buf == NULL) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Public key type in decrypted key data not found");
        return -1;
    }

    // 如果公钥类型为 "ssh-dss"，则调用函数生成 DSA 对象
    if(strcmp("ssh-dss", (const char *)buf) == 0) {
        rc = gen_publickey_from_dsa_openssh_priv_data(session, decrypted,
                                                      NULL, 0,
                                                      NULL, 0, dsa);
    }
    else {
        rc = -1;
    }

    // 释放解密后的数据
    if(decrypted)
        _libssh2_string_buf_free(session, decrypted);

    return rc;
}

// 从私钥文件中创建 DSA 对象
int
_libssh2_dsa_new_private(libssh2_dsa_ctx ** dsa,
                         LIBSSH2_SESSION * session,
                         const char *filename, unsigned const char *passphrase)
{
    int rc;
    # 定义一个函数指针 read_dsa，指向 PEM_read_bio_DSAPrivateKey 函数
    pem_read_bio_func read_dsa =
        (pem_read_bio_func) &PEM_read_bio_DSAPrivateKey;

    # 初始化 libssh2 库
    _libssh2_init_if_needed();

    # 从文件中读取私钥，并存储到 dsa 指针指向的位置
    rc = read_private_key_from_file((void **) dsa, read_dsa,
                                    filename, passphrase);

    # 如果读取私钥出错
    if(rc) {
        # 从 OpenSSH 格式的私钥文件中创建一个新的 DSA 私钥
        rc = _libssh2_dsa_new_openssh_private(dsa, session,
                                              filename, passphrase);
    }

    # 返回操作结果
    return rc;
#endif /* LIBSSH_DSA */

#if LIBSSH2_ECDSA

// 从内存中创建新的 ECDSA 私钥
int
_libssh2_ecdsa_new_private_frommemory(libssh2_ecdsa_ctx ** ec_ctx,
                                    LIBSSH2_SESSION * session,
                                    const char *filedata, size_t filedata_len,
                                    unsigned const char *passphrase)
{
    int rc;

    // 定义 PEM 格式的 ECDSA 私钥读取函数
    pem_read_bio_func read_ec =
        (pem_read_bio_func) &PEM_read_bio_ECPrivateKey;

    // 初始化 libssh2 库
    _libssh2_init_if_needed();

    // 从内存中读取私钥
    rc = read_private_key_from_memory((void **) ec_ctx, read_ec,
                                      filedata, filedata_len, passphrase);

    // 如果读取失败，则尝试从内存中读取 OpenSSH 格式的私钥
    if(rc) {
        rc = read_openssh_private_key_from_memory((void **)ec_ctx, session,
                                                  "ssh-ecdsa", filedata,
                                                  filedata_len, passphrase);
    }

    return rc;
}

#endif /* LIBSSH2_ECDSA */


#if LIBSSH2_ED25519

// 创建新的 Curve25519 密钥对
int
_libssh2_curve25519_new(LIBSSH2_SESSION *session,
                        unsigned char **out_public_key,
                        unsigned char **out_private_key)
{
    EVP_PKEY *key = NULL;
    EVP_PKEY_CTX *pctx = NULL;
    unsigned char *priv = NULL, *pub = NULL;
    size_t privLen, pubLen;
    int rc = -1;

    // 创建新的 EVP_PKEY_CTX 对象
    pctx = EVP_PKEY_CTX_new_id(EVP_PKEY_X25519, NULL);
    if(pctx == NULL)
        return -1;

    // 初始化密钥生成操作
    if(EVP_PKEY_keygen_init(pctx) != 1 ||
       EVP_PKEY_keygen(pctx, &key) != 1) {
        goto cleanExit;
    }

    // 如果需要输出私钥，则获取并分配内存
    if(out_private_key != NULL) {
        privLen = LIBSSH2_ED25519_KEY_LEN;
        priv = LIBSSH2_ALLOC(session, privLen);
        if(priv == NULL)
            goto cleanExit;

        // 获取原始私钥数据
        if(EVP_PKEY_get_raw_private_key(key, priv, &privLen) != 1 ||
           privLen != LIBSSH2_ED25519_KEY_LEN) {
            goto cleanExit;
        }

        *out_private_key = priv;
        priv = NULL;
    }
    # 如果输出公钥不为空
    if(out_public_key != NULL) {
        # 设置公钥长度为ED25519密钥长度
        pubLen = LIBSSH2_ED25519_KEY_LEN;
        # 分配内存以存储公钥
        pub = LIBSSH2_ALLOC(session, pubLen);
        # 如果分配内存失败，则跳转到清理退出
        if(pub == NULL)
            goto cleanExit;

        # 获取原始公钥并存储到pub中，如果失败或者长度不符合ED25519密钥长度，则跳转到清理退出
        if(EVP_PKEY_get_raw_public_key(key, pub, &pubLen) != 1 ||
           pubLen != LIBSSH2_ED25519_KEY_LEN) {
            goto cleanExit;
        }

        # 将pub赋值给输出公钥，并将pub设置为空，避免重复释放内存
        *out_public_key = pub;
        pub = NULL;
    }

    # 成功
    rc = 0;
# 清理并退出函数
cleanExit:

    # 如果上下文存在，则释放上下文
    if(pctx)
        EVP_PKEY_CTX_free(pctx);
    # 如果密钥存在，则释放密钥
    if(key)
        EVP_PKEY_free(key);
    # 如果私钥存在，则释放私钥
    if(priv)
        LIBSSH2_FREE(session, priv);
    # 如果公钥存在，则释放公钥
    if(pub)
        LIBSSH2_FREE(session, pub);

    # 返回结果
    return rc;
}

# 从 ED 私钥生成公钥
static int
gen_publickey_from_ed_evp(LIBSSH2_SESSION *session,
                          unsigned char **method,
                          size_t *method_len,
                          unsigned char **pubkeydata,
                          size_t *pubkeydata_len,
                          EVP_PKEY *pk)
{
    # 定义方法名
    const char methodName[] = "ssh-ed25519";
    unsigned char *methodBuf = NULL;
    size_t rawKeyLen = 0;
    unsigned char *keyBuf = NULL;
    size_t bufLen = 0;
    unsigned char *bufPos = NULL;

    # 调试信息
    _libssh2_debug(session, LIBSSH2_TRACE_AUTH,
                   "Computing public key from ED private key envelope");

    # 分配内存给方法名缓冲区
    methodBuf = LIBSSH2_ALLOC(session, sizeof(methodName) - 1);
    if(!methodBuf) {
        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                       "Unable to allocate memory for private key data");
        goto fail;
    }
    # 复制方法名到缓冲区
    memcpy(methodBuf, methodName, sizeof(methodName) - 1);

    # 获取原始公钥长度
    if(EVP_PKEY_get_raw_public_key(pk, NULL, &rawKeyLen) != 1) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "EVP_PKEY_get_raw_public_key failed");
        goto fail;
    }

    # 计算密钥缓冲区长度
    bufLen = 4 + sizeof(methodName) - 1  + 4 + rawKeyLen;
    bufPos = keyBuf = LIBSSH2_ALLOC(session, bufLen);
    if(!keyBuf) {
        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                       "Unable to allocate memory for private key data");
        goto fail;
    }

    # 存储方法名到缓冲区
    _libssh2_store_str(&bufPos, methodName, sizeof(methodName) - 1);
    # 存储原始公钥长度到缓冲区
    _libssh2_store_u32(&bufPos, rawKeyLen);
    # 如果获取原始公钥失败
    if(EVP_PKEY_get_raw_public_key(pk, bufPos, &rawKeyLen) != 1) {
        # 记录错误信息
        _libssh2_error(session, LIBSSH2_ERROR_PROTO, "EVP_PKEY_get_raw_public_key failed");
        # 跳转到失败标签
        goto fail;
    }

    # 设置方法指针
    *method         = methodBuf;
    # 设置方法长度
    *method_len     = sizeof(methodName) - 1;
    # 设置公钥数据指针
    *pubkeydata     = keyBuf;
    # 设置公钥数据长度
    *pubkeydata_len = bufLen;
    # 返回成功
    return 0;
fail:
    # 如果 methodBuf 存在，则释放其内存
    if(methodBuf)
        LIBSSH2_FREE(session, methodBuf);
    # 如果 keyBuf 存在，则释放其内存
    if(keyBuf)
        LIBSSH2_FREE(session, keyBuf);
    # 返回 -1
    return -1;
}


static int
gen_publickey_from_ed25519_openssh_priv_data(LIBSSH2_SESSION *session,
                                             struct string_buf *decrypted,
                                             unsigned char **method,
                                             size_t *method_len,
                                             unsigned char **pubkeydata,
                                             size_t *pubkeydata_len,
                                             libssh2_ed25519_ctx **out_ctx)
{
    libssh2_ed25519_ctx *ctx = NULL;
    unsigned char *method_buf = NULL;
    unsigned char *key = NULL;
    int i, ret = 0;
    unsigned char *pub_key, *priv_key, *buf;
    size_t key_len = 0, tmp_len = 0;
    unsigned char *p;

    _libssh2_debug(session,
                   LIBSSH2_TRACE_AUTH,
                   "Computing ED25519 keys from private key data");

    # 从解密后的数据中获取公钥和长度，并检查长度是否正确
    if(_libssh2_get_string(decrypted, &pub_key, &tmp_len) ||
       tmp_len != LIBSSH2_ED25519_KEY_LEN) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Wrong public key length");
        return -1;
    }

    # 从解密后的数据中获取私钥和长度，并检查长度是否正确
    if(_libssh2_get_string(decrypted, &priv_key, &tmp_len) ||
       tmp_len != LIBSSH2_ED25519_PRIVATE_KEY_LEN) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Wrong private key length");
        ret = -1;
        # 跳转到清理退出的位置
        goto clean_exit;
    }

    /* first 32 bytes of priv_key is the private key, the last 32 bytes are
       the public key */
    # 使用 priv_key 的前 32 字节作为私钥，后 32 字节作为公钥创建 ED25519 上下文
    ctx = EVP_PKEY_new_raw_private_key(EVP_PKEY_ED25519, NULL,
                                       (const unsigned char *)priv_key,
                                       LIBSSH2_ED25519_KEY_LEN);

    /* comment */
    # 如果无法获取解密后的字符串，则记录错误信息，返回-1
    if(_libssh2_get_string(decrypted, &buf, &tmp_len)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Unable to read comment");
        ret = -1;
        goto clean_exit;
    }

    # 如果获取到的字符串长度大于0，则分配内存空间存储注释
    if(tmp_len > 0) {
        unsigned char *comment = LIBSSH2_CALLOC(session, tmp_len + 1);
        # 如果成功分配内存，则将解密后的字符串复制到注释中
        if(comment != NULL) {
            memcpy(comment, buf, tmp_len);
            # 在注释末尾添加字符串结束符
            memcpy(comment + tmp_len, "\0", 1);

            # 打印注释信息
            _libssh2_debug(session, LIBSSH2_TRACE_AUTH, "Key comment: %s",
                           comment);

            # 释放注释内存
            LIBSSH2_FREE(session, comment);
        }
    }

    # 填充
    i = 1;
    # 检查解密后的数据是否有正确的填充
    while(decrypted->dataptr < decrypted->data + decrypted->len) {
        if(*decrypted->dataptr != i) {
            _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                           "Wrong padding");
            ret = -1;
            goto clean_exit;
        }
        i++;
        decrypted->dataptr++;
    }
    # 如果返回值为0，则执行以下操作
    if(ret == 0) {
        # 在调试日志中记录正在计算ED25519私钥信封的公钥
        _libssh2_debug(session,
                       LIBSSH2_TRACE_AUTH,
                       "Computing public key from ED25519 "
                       "private key envelope");

        # 分配内存以存储方法名称 "ssh-ed25519."
        method_buf = LIBSSH2_ALLOC(session, 11);  /* ssh-ed25519. */
        # 如果内存分配失败，则记录错误并跳转到清理退出
        if(method_buf == NULL) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for ED25519 key");
            goto clean_exit;
        }

        # 计算公钥长度
        key_len = LIBSSH2_ED25519_KEY_LEN + 19;
        # 分配内存以存储公钥
        key = LIBSSH2_CALLOC(session, key_len);
        # 如果内存分配失败，则记录错误并跳转到清理退出
        if(key == NULL) {
            _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                           "Unable to allocate memory for ED25519 key");
            goto clean_exit;
        }

        # 将方法名称 "ssh-ed25519" 复制到方法缓冲区
        memcpy(method_buf, "ssh-ed25519", 11);

        # 如果method不为空，则将method_buf赋值给method，否则释放method_buf内存
        if(method != NULL)
            *method = method_buf;
        else
            LIBSSH2_FREE(session, method_buf);

        # 如果method_len不为空，则将11赋值给method_len
        if(method_len != NULL)
            *method_len = 11;

        # 如果pubkeydata不为空，则将key赋值给pubkeydata，否则释放key内存
        if(pubkeydata != NULL)
            *pubkeydata = key;
        else
            LIBSSH2_FREE(session, key);

        # 如果pubkeydata_len不为空，则将key_len赋值给pubkeydata_len
        if(pubkeydata_len != NULL)
            *pubkeydata_len = key_len;

        # 如果out_ctx不为空，则将ctx赋值给out_ctx，否则释放ctx内存；如果ctx不为空，则释放ctx
        if(out_ctx != NULL)
            *out_ctx = ctx;
        else if(ctx != NULL)
            _libssh2_ed25519_free(ctx);

        # 返回0
        return 0;
    }
# 清理并退出函数
clean_exit:
    # 如果上下文存在，则释放 ED25519 上下文
    if(ctx)
        _libssh2_ed25519_free(ctx);
    # 释放方法缓冲区
    if(method_buf)
        LIBSSH2_FREE(session, method_buf);
    # 释放密钥
    if(key)
        LIBSSH2_FREE(session, key);
    # 返回错误代码
    return -1;
}

# 创建新的私钥
int
_libssh2_ed25519_new_private(libssh2_ed25519_ctx ** ed_ctx,
                             LIBSSH2_SESSION * session,
                             const char *filename, const uint8_t *passphrase)
{
    int rc;
    FILE *fp;
    unsigned char *buf;
    struct string_buf *decrypted = NULL;
    libssh2_ed25519_ctx *ctx = NULL;

    # 如果会话为空，则返回错误
    if(session == NULL) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Session is required");
        return -1;
    }

    # 初始化 libssh2
    _libssh2_init_if_needed();

    # 打开私钥文件
    fp = fopen(filename, "r");
    if(!fp) {
        _libssh2_error(session, LIBSSH2_ERROR_FILE,
                       "Unable to open ED25519 private key file");
        return -1;
    }

    # 解析 PEM 格式的私钥文件
    rc = _libssh2_openssh_pem_parse(session, passphrase, fp, &decrypted);
    fclose(fp);
    if(rc) {
        return rc;
    }

    # 解析密钥数据，获取公钥类型
    rc = _libssh2_get_string(decrypted, &buf, NULL);

    if(rc != 0 || buf == NULL) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Public key type in decrypted key data not found");
        return -1;
    }

    # 检查公钥类型，如果是 ssh-ed25519 则生成公钥
    if(strcmp("ssh-ed25519", (const char *)buf) == 0) {
        rc = gen_publickey_from_ed25519_openssh_priv_data(session,
                                                          decrypted,
                                                          NULL,
                                                          NULL,
                                                          NULL,
                                                          NULL,
                                                          &ctx);
    }
    else {
        rc = -1;
    }

    # 释放解密后的数据
    if(decrypted)
        _libssh2_string_buf_free(session, decrypted);
}
    # 如果返回值为0，则执行以下操作
    if(rc == 0) {
        # 如果ed_ctx不为空，则将ctx的值赋给ed_ctx
        if(ed_ctx != NULL)
            *ed_ctx = ctx;
        # 如果ctx不为空，则释放ed25519上下文
        else if(ctx != NULL)
            _libssh2_ed25519_free(ctx);
    }
    # 返回rc的值
    return rc;
}

int
_libssh2_ed25519_new_private_frommemory(libssh2_ed25519_ctx ** ed_ctx,
                                        LIBSSH2_SESSION * session,
                                        const char *filedata,
                                        size_t filedata_len,
                                        unsigned const char *passphrase)
{
    libssh2_ed25519_ctx *ctx = NULL;

    _libssh2_init_if_needed();  // 初始化 libssh2 库

    if(read_private_key_from_memory((void **)&ctx,  // 从内存中读取私钥
                                    (pem_read_bio_func)
                                    &PEM_read_bio_PrivateKey,
                                    filedata, filedata_len, passphrase) == 0) {
        if(EVP_PKEY_id(ctx) != EVP_PKEY_ED25519) {  // 检查私钥是否为 ED25519 类型
            _libssh2_ed25519_free(ctx);  // 释放内存
            return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                                  "Private key is not an ED25519 key");  // 返回错误信息
        }

        *ed_ctx = ctx;  // 将私钥上下文赋值给传入的指针
        return 0;  // 返回成功
    }

    return read_openssh_private_key_from_memory((void **)ed_ctx, session,  // 从内存中读取 OpenSSH 私钥
                                                "ssh-ed25519",
                                                filedata, filedata_len,
                                                passphrase);
}

int
_libssh2_ed25519_new_public(libssh2_ed25519_ctx ** ed_ctx,
                            LIBSSH2_SESSION * session,
                            const unsigned char *raw_pub_key,
                            const uint8_t key_len)
{
    libssh2_ed25519_ctx *ctx = NULL;

    if(ed_ctx == NULL)  // 检查传入的指针是否为空
        return -1;

    ctx = EVP_PKEY_new_raw_public_key(EVP_PKEY_ED25519, NULL,  // 创建 ED25519 公钥上下文
                                      raw_pub_key, key_len);
    if(!ctx)  // 如果上下文为空
        return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                              "could not create ED25519 public key");  // 返回错误信息

    if(ed_ctx != NULL)  // 如果传入的指针不为空
        *ed_ctx = ctx;  // 将公钥上下文赋值给传入的指针
    else if(ctx)  // 如果上下文不为空
        _libssh2_ed25519_free(ctx);  // 释放内存

    return 0;  // 返回成功
}
#endif /* LIBSSH2_ED25519 */


int
# 使用 RSA 算法对给定的哈希值进行签名
_libssh2_rsa_sha1_sign(LIBSSH2_SESSION * session,
                       libssh2_rsa_ctx * rsactx,
                       const unsigned char *hash,
                       size_t hash_len,
                       unsigned char **signature, size_t *signature_len)
{
    int ret;
    unsigned char *sig;
    unsigned int sig_len;

    # 计算 RSA 签名的长度
    sig_len = RSA_size(rsactx);
    # 分配内存用于存储签名
    sig = LIBSSH2_ALLOC(session, sig_len);

    # 如果内存分配失败，则返回错误
    if(!sig) {
        return -1;
    }

    # 使用 RSA 算法对哈希值进行签名
    ret = RSA_sign(NID_sha1, hash, hash_len, sig, &sig_len, rsactx);

    # 如果签名失败，则释放内存并返回错误
    if(!ret) {
        LIBSSH2_FREE(session, sig);
        return -1;
    }

    # 将签名和签名长度返回
    *signature = sig;
    *signature_len = sig_len;

    return 0;
}

# 如果支持 DSA 算法，则使用 DSA 算法对给定的哈希值进行签名
#if LIBSSH2_DSA
int
_libssh2_dsa_sha1_sign(libssh2_dsa_ctx * dsactx,
                       const unsigned char *hash,
                       unsigned long hash_len, unsigned char *signature)
{
    DSA_SIG *sig;
    const BIGNUM * r;
    const BIGNUM * s;
    int r_len, s_len;
    (void) hash_len;

    # 使用 DSA 算法对哈希值进行签名
    sig = DSA_do_sign(hash, SHA_DIGEST_LENGTH, dsactx);
    # 如果签名失败，则返回错误
    if(!sig) {
        return -1;
    }

    # 获取签名中的 r 和 s 值
    DSA_SIG_get0(sig, &r, &s);
    r_len = BN_num_bytes(r);
    # 如果 r 值长度不合法，则释放内存并返回错误
    if(r_len < 1 || r_len > 20) {
        DSA_SIG_free(sig);
        return -1;
    }
    s_len = BN_num_bytes(s);
    # 如果 s 值长度不合法，则释放内存并返回错误
    if(s_len < 1 || s_len > 20) {
        DSA_SIG_free(sig);
        return -1;
    }

    # 将 r 和 s 值转换为二进制并存储在签名中
    memset(signature, 0, 40);
    BN_bn2bin(r, signature + (20 - r_len));
    BN_bn2bin(s, signature + 20 + (20 - s_len));

    # 释放签名内存并返回成功
    DSA_SIG_free(sig);

    return 0;
}
#endif /* LIBSSH_DSA */

# 如果支持 ECDSA 算法，则使用 ECDSA 算法对给定的哈希值进行签名
#if LIBSSH2_ECDSA
int
_libssh2_ecdsa_sign(LIBSSH2_SESSION * session, libssh2_ecdsa_ctx * ec_ctx,
    const unsigned char *hash, unsigned long hash_len,
    unsigned char **signature, size_t *signature_len)
{
    int r_len, s_len;
    int rc = 0;
    size_t out_buffer_len = 0;
    unsigned char *sp;
    const BIGNUM *pr = NULL, *ps = NULL;
    unsigned char *temp_buffer = NULL;
    # 声明一个指向无符号字符的指针，并初始化为 NULL
    unsigned char *out_buffer = NULL;
    
    # 使用 ECDSA_do_sign 函数对给定的哈希值进行签名，返回签名结果
    ECDSA_SIG *sig = ECDSA_do_sign(hash, hash_len, ec_ctx);
    # 如果签名结果为空，则返回 -1
    if(sig == NULL)
        return -1;
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果支持不透明结构，则使用ECDSA_SIG_get0函数获取签名的r和s值
    ECDSA_SIG_get0(sig, &pr, &ps);
#else
    // 否则直接获取签名的r和s值
    pr = sig->r;
    ps = sig->s;
#endif

    // 计算r和s值的字节长度
    r_len = BN_num_bytes(pr) + 1;
    s_len = BN_num_bytes(ps) + 1;

    // 分配临时缓冲区
    temp_buffer = malloc(r_len + s_len + 8);
    if(temp_buffer == NULL) {
        rc = -1;
        goto clean_exit;
    }

    // 将r和s值写入临时缓冲区
    sp = temp_buffer;
    sp = write_bn(sp, pr, r_len);
    sp = write_bn(sp, ps, s_len);

    // 计算输出缓冲区的长度
    out_buffer_len = (size_t)(sp - temp_buffer);

    // 分配输出缓冲区
    out_buffer = LIBSSH2_CALLOC(session, out_buffer_len);
    if(out_buffer == NULL) {
        rc = -1;
        goto clean_exit;
    }

    // 将临时缓冲区的内容复制到输出缓冲区
    memcpy(out_buffer, temp_buffer, out_buffer_len);

    // 设置签名和签名长度
    *signature = out_buffer;
    *signature_len = out_buffer_len;

clean_exit:

    // 释放临时缓冲区
    if(temp_buffer != NULL)
        free(temp_buffer);

    // 释放ECDSA_SIG结构
    if(sig)
        ECDSA_SIG_free(sig);

    // 返回结果
    return rc;
}
#endif /* LIBSSH2_ECDSA */

// 初始化SHA1上下文
int
_libssh2_sha1_init(libssh2_sha1_ctx *ctx)
{
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果支持不透明结构，则使用EVP_MD_CTX_new函数创建上下文
    *ctx = EVP_MD_CTX_new();

    // 检查上下文是否创建成功，并初始化为SHA1算法
    if(*ctx == NULL)
        return 0;

    if(EVP_DigestInit(*ctx, EVP_get_digestbyname("sha1")))
        return 1;

    // 释放上下文并返回错误码
    EVP_MD_CTX_free(*ctx);
    *ctx = NULL;

    return 0;
#else
    // 如果不支持不透明结构，则初始化上下文为SHA1算法
    EVP_MD_CTX_init(ctx);
    return EVP_DigestInit(ctx, EVP_get_digestbyname("sha1"));
#endif
}

// 计算SHA1哈希值
int
_libssh2_sha1(const unsigned char *message, unsigned long len,
              unsigned char *out)
{
#ifdef HAVE_OPAQUE_STRUCTS
    // 如果支持不透明结构，则使用EVP_MD_CTX_new函数创建上下文
    EVP_MD_CTX * ctx = EVP_MD_CTX_new();

    // 检查上下文是否创建成功
    if(ctx == NULL)
        return 1; /* error */

    // 初始化上下文为SHA1算法，并计算哈希值
    if(EVP_DigestInit(ctx, EVP_get_digestbyname("sha1"))) {
        EVP_DigestUpdate(ctx, message, len);
        EVP_DigestFinal(ctx, out, NULL);
        EVP_MD_CTX_free(ctx);
        return 0; /* success */
    }
    EVP_MD_CTX_free(ctx);
#else
    // 如果不支持不透明结构，则初始化上下文为SHA1算法，并计算哈希值
    EVP_MD_CTX ctx;

    EVP_MD_CTX_init(&ctx);
    if(EVP_DigestInit(&ctx, EVP_get_digestbyname("sha1"))) {
        EVP_DigestUpdate(&ctx, message, len);
        EVP_DigestFinal(&ctx, out, NULL);
        return 0; /* success */
    }
#endif
    return 1; /* error */
}

// ...
# 初始化 SHA256 上下文
_libssh2_sha256_init(libssh2_sha256_ctx *ctx)
{
#ifdef HAVE_OPAQUE_STRUCTS
    # 使用 EVP_MD_CTX_new() 创建一个新的 SHA256 上下文
    *ctx = EVP_MD_CTX_new();

    # 如果上下文创建失败，返回 0
    if(*ctx == NULL)
        return 0;

    # 使用 EVP_DigestInit() 初始化 SHA256 上下文
    if(EVP_DigestInit(*ctx, EVP_get_digestbyname("sha256")))
        return 1;

    # 释放上下文内存
    EVP_MD_CTX_free(*ctx);
    *ctx = NULL;

    return 0;
#else
    # 初始化 SHA256 上下文
    EVP_MD_CTX_init(ctx);
    return EVP_DigestInit(ctx, EVP_get_digestbyname("sha256"));
#endif
}

# 计算 SHA256 哈希值
int
_libssh2_sha256(const unsigned char *message, unsigned long len,
                unsigned char *out)
{
#ifdef HAVE_OPAQUE_STRUCTS
    # 使用 EVP_MD_CTX_new() 创建一个新的 SHA256 上下文
    EVP_MD_CTX * ctx = EVP_MD_CTX_new();

    # 如果上下文创建失败，返回 1
    if(ctx == NULL)
        return 1; /* error */

    # 使用 EVP_DigestInit() 初始化 SHA256 上下文
    if(EVP_DigestInit(ctx, EVP_get_digestbyname("sha256"))) {
        # 更新消息内容
        EVP_DigestUpdate(ctx, message, len);
        # 计算最终哈希值
        EVP_DigestFinal(ctx, out, NULL);
        # 释放上下文内存
        EVP_MD_CTX_free(ctx);
        return 0; /* success */
    }
    # 释放上下文内存
    EVP_MD_CTX_free(ctx);
#else
    # 初始化 SHA256 上下文
    EVP_MD_CTX ctx;

    EVP_MD_CTX_init(&ctx);
    # 使用 EVP_DigestInit() 初始化 SHA256 上下文
    if(EVP_DigestInit(&ctx, EVP_get_digestbyname("sha256"))) {
        # 更新消息内容
        EVP_DigestUpdate(&ctx, message, len);
        # 计算最终哈希值
        EVP_DigestFinal(&ctx, out, NULL);
        return 0; /* success */
    }
#endif
    return 1; /* error */
}

# 初始化 SHA384 上下文
int
_libssh2_sha384_init(libssh2_sha384_ctx *ctx)
{
#ifdef HAVE_OPAQUE_STRUCTS
    # 使用 EVP_MD_CTX_new() 创建一个新的 SHA384 上下文
    *ctx = EVP_MD_CTX_new();

    # 如果上下文创建失败，返回 0
    if(*ctx == NULL)
        return 0;

    # 使用 EVP_DigestInit() 初始化 SHA384 上下文
    if(EVP_DigestInit(*ctx, EVP_get_digestbyname("sha384")))
        return 1;

    # 释放上下文内存
    EVP_MD_CTX_free(*ctx);
    *ctx = NULL;

    return 0;
#else
    # 初始化 SHA384 上下文
    EVP_MD_CTX_init(ctx);
    return EVP_DigestInit(ctx, EVP_get_digestbyname("sha384"));
#endif
}

# 计算 SHA384 哈希值
int
_libssh2_sha384(const unsigned char *message, unsigned long len,
    unsigned char *out)
{
#ifdef HAVE_OPAQUE_STRUCTS
    # 使用 EVP_MD_CTX_new() 创建一个新的 SHA384 上下文
    EVP_MD_CTX * ctx = EVP_MD_CTX_new();

    # 如果上下文创建失败，返回 1
    if(ctx == NULL)
        return 1; /* error */

    # 使用 EVP_DigestInit() 初始化 SHA384 上下文
    if(EVP_DigestInit(ctx, EVP_get_digestbyname("sha384"))) {
        # 更新消息内容
        EVP_DigestUpdate(ctx, message, len);
        # 计算最终哈希值
        EVP_DigestFinal(ctx, out, NULL);
        # 释放上下文内存
        EVP_MD_CTX_free(ctx);
        return 0; /* success */
    }
    # 释放上下文内存
    EVP_MD_CTX_free(ctx);
#else
    # 初始化消息摘要上下文
    EVP_MD_CTX ctx;

    # 初始化消息摘要上下文结构体
    EVP_MD_CTX_init(&ctx);
    
    # 根据摘要算法名称获取摘要算法对象，并初始化摘要上下文
    if(EVP_DigestInit(&ctx, EVP_get_digestbyname("sha384"))) {
        
        # 更新消息摘要上下文的数据
        EVP_DigestUpdate(&ctx, message, len);
        
        # 完成消息摘要计算，将结果存储在 out 中
        EVP_DigestFinal(&ctx, out, NULL);
        
        # 返回成功
        return 0; /* success */
    }
#endif
    return 1; /* 如果条件不满足，则返回错误代码1 */
}

int
_libssh2_sha512_init(libssh2_sha512_ctx *ctx)
{
#ifdef HAVE_OPAQUE_STRUCTS
    *ctx = EVP_MD_CTX_new(); /* 为SHA512上下文分配内存 */

    if(*ctx == NULL)
        return 0; /* 如果分配内存失败，则返回错误代码0 */

    if(EVP_DigestInit(*ctx, EVP_get_digestbyname("sha512")))
        return 1; /* 如果初始化SHA512哈希算法成功，则返回成功代码1 */

    EVP_MD_CTX_free(*ctx); /* 释放SHA512上下文内存 */
    *ctx = NULL; /* 将上下文指针置为空 */

    return 0; /* 返回成功代码0 */
#else
    EVP_MD_CTX_init(ctx); /* 初始化SHA512上下文 */
    return EVP_DigestInit(ctx, EVP_get_digestbyname("sha512")); /* 初始化SHA512哈希算法 */
#endif
}

int
_libssh2_sha512(const unsigned char *message, unsigned long len,
    unsigned char *out)
{
#ifdef HAVE_OPAQUE_STRUCTS
    EVP_MD_CTX * ctx = EVP_MD_CTX_new(); /* 为SHA512上下文分配内存 */

    if(ctx == NULL)
        return 1; /* 如果分配内存失败，则返回错误代码1 */

    if(EVP_DigestInit(ctx, EVP_get_digestbyname("sha512"))) {
        EVP_DigestUpdate(ctx, message, len); /* 更新SHA512哈希算法上下文 */
        EVP_DigestFinal(ctx, out, NULL); /* 计算SHA512哈希值 */
        EVP_MD_CTX_free(ctx); /* 释放SHA512上下文内存 */
        return 0; /* 返回成功代码0 */
    }
    EVP_MD_CTX_free(ctx); /* 释放SHA512上下文内存 */
#else
    EVP_MD_CTX ctx; /* 定义SHA512上下文 */

    EVP_MD_CTX_init(&ctx); /* 初始化SHA512上下文 */
    if(EVP_DigestInit(&ctx, EVP_get_digestbyname("sha512"))) {
        EVP_DigestUpdate(&ctx, message, len); /* 更新SHA512哈希算法上下文 */
        EVP_DigestFinal(&ctx, out, NULL); /* 计算SHA512哈希值 */
        return 0; /* 返回成功代码0 */
    }
#endif
    return 1; /* 返回错误代码1 */
}

int
_libssh2_md5_init(libssh2_md5_ctx *ctx)
{
    /* 在OpenSSL FIPS模式下不支持MD5摘要
     * 尝试初始化会导致潜在的OpenSSL错误:
     * "digital envelope routines:FIPS_DIGESTINIT:disabled for fips"
     * 因此，在FIPS模式下返回0
     */
#if OPENSSL_VERSION_NUMBER >= 0x000907000L && \
    defined(OPENSSL_VERSION_MAJOR) && \
    OPENSSL_VERSION_MAJOR < 3 && \
    !defined(LIBRESSL_VERSION_NUMBER)
     if(FIPS_mode() != 0)
         return 0; /* 如果处于FIPS模式，则返回0 */
#endif

#ifdef HAVE_OPAQUE_STRUCTS
    *ctx = EVP_MD_CTX_new(); /* 为MD5上下文分配内存 */

    if(*ctx == NULL)
        return 0; /* 如果分配内存失败，则返回错误代码0 */

    if(EVP_DigestInit(*ctx, EVP_get_digestbyname("md5")))
        return 1; /* 如果初始化MD5哈希算法成功，则返回成功代码1 */

    EVP_MD_CTX_free(*ctx); /* 释放MD5上下文内存 */
    *ctx = NULL; /* 将上下文指针置为空 */

    return 0; /* 返回成功代码0 */
#else
    EVP_MD_CTX_init(ctx); /* 初始化MD5上下文 */
    return EVP_DigestInit(ctx, EVP_get_digestbyname("md5")); /* 初始化MD5哈希算法 */
#endif
}
#if LIBSSH2_ECDSA

static int
gen_publickey_from_ec_evp(LIBSSH2_SESSION *session,
                          unsigned char **method,
                          size_t *method_len,
                          unsigned char **pubkeydata,
                          size_t *pubkeydata_len,
                          EVP_PKEY *pk)
{
    int rc = 0;  // 定义整型变量 rc，用于存储函数执行结果
    EC_KEY *ec = NULL;  // 定义 EC_KEY 指针变量 ec，初始化为 NULL
    unsigned char *p;  // 定义无符号字符指针变量 p
    unsigned char *method_buf = NULL;  // 定义无符号字符指针变量 method_buf，初始化为 NULL
    unsigned char *key;  // 定义无符号字符指针变量 key
    size_t  key_len = 0;  // 定义大小为 size_t 的 key_len 变量，初始化为 0
    unsigned char *octal_value = NULL;  // 定义无符号字符指针变量 octal_value，初始化为 NULL
    size_t octal_len;  // 定义大小为 size_t 的 octal_len 变量
    const EC_POINT *public_key;  // 定义 EC_POINT 类型的常量指针 public_key
    const EC_GROUP *group;  // 定义 EC_GROUP 类型的常量指针 group
    BN_CTX *bn_ctx;  // 定义 BN_CTX 类型的指针变量 bn_ctx
    libssh2_curve_type type;  // 定义 libssh2_curve_type 类型的 type 变量

    _libssh2_debug(session,  // 调用 _libssh2_debug 函数，用于输出调试信息
       LIBSSH2_TRACE_AUTH,  // 传入调试信息类型参数
       "Computing public key from EC private key envelope");  // 输出调试信息内容

    bn_ctx = BN_CTX_new();  // 调用 BN_CTX_new 函数，为 bn_ctx 分配内存空间
    if(bn_ctx == NULL)  // 判断 bn_ctx 是否为 NULL
        return -1;  // 如果是，则返回 -1

    ec = EVP_PKEY_get1_EC_KEY(pk);  // 调用 EVP_PKEY_get1_EC_KEY 函数，将 pk 转换为 EC_KEY 类型，并赋值给 ec
    if(ec == NULL) {  // 判断 ec 是否为 NULL
        rc = -1;  // 如果是，则将 rc 赋值为 -1
        goto clean_exit;  // 跳转到 clean_exit 标签处
    }

    public_key = EC_KEY_get0_public_key(ec);  // 获取 EC_KEY 对象的公钥，赋值给 public_key
    group = EC_KEY_get0_group(ec);  // 获取 EC_KEY 对象的椭圆曲线群，赋值给 group
    type = _libssh2_ecdsa_get_curve_type(ec);  // 调用 _libssh2_ecdsa_get_curve_type 函数，获取 EC_KEY 对象的曲线类型，赋值给 type

    method_buf = LIBSSH2_ALLOC(session, 19);  // 调用 LIBSSH2_ALLOC 函数，为 method_buf 分配内存空间
    if(method_buf == NULL) {  // 判断 method_buf 是否为 NULL
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,  // 如果是，则返回分配内存错误
            "out of memory");  // 返回错误信息
    }

    if(type == LIBSSH2_EC_CURVE_NISTP256)  // 判断 type 是否为 LIBSSH2_EC_CURVE_NISTP256
        memcpy(method_buf, "ecdsa-sha2-nistp256", 19);  // 如果是，则将 "ecdsa-sha2-nistp256" 复制到 method_buf 中
    else if(type == LIBSSH2_EC_CURVE_NISTP384)  // 判断 type 是否为 LIBSSH2_EC_CURVE_NISTP384
        memcpy(method_buf, "ecdsa-sha2-nistp384", 19);  // 如果是，则将 "ecdsa-sha2-nistp384" 复制到 method_buf 中
    else if(type == LIBSSH2_EC_CURVE_NISTP521)  // 判断 type 是否为 LIBSSH2_EC_CURVE_NISTP521
        memcpy(method_buf, "ecdsa-sha2-nistp521", 19);  // 如果是，则将 "ecdsa-sha2-nistp521" 复制到 method_buf 中
    else {
        _libssh2_debug(session,  // 调用 _libssh2_debug 函数，用于输出调试信息
            LIBSSH2_TRACE_ERROR,  // 传入调试信息类型参数
            "Unsupported EC private key type");  // 输出调试信息内容
        rc = -1;  // 将 rc 赋值为 -1
        goto clean_exit;  // 跳转到 clean_exit 标签处
    }

    /* get length */
    octal_len = EC_POINT_point2oct(group, public_key,  // 调用 EC_POINT_point2oct 函数，获取 EC_POINT 对象的压缩格式数据长度
                                   POINT_CONVERSION_UNCOMPRESSED,  // 传入压缩格式参数
                                   NULL, 0, bn_ctx);  // 传入其他参数
    if(octal_len > EC_MAX_POINT_LEN) {  // 判断 octal_len 是否大于 EC_MAX_POINT_LEN
        rc = -1;  // 如果是，则将 rc 赋值为 -1
        goto clean_exit;  // 跳转到 clean_exit 标签处
    }
    # 分配内存以存储八进制值
    octal_value = malloc(octal_len);
    # 检查内存分配是否成功
    if(octal_value == NULL) {
        rc = -1;
        goto clean_exit;
    }

    # 转换为八进制
    if(EC_POINT_point2oct(group, public_key, POINT_CONVERSION_UNCOMPRESSED,
       octal_value, octal_len, bn_ctx) != octal_len) {
           rc = -1;
           goto clean_exit;
    }

    # 计算密钥长度
    key_len = 4 + 19 + 4 + 8 + 4 + octal_len;
    # 分配内存以存储密钥
    key = LIBSSH2_ALLOC(session, key_len);
    # 检查内存分配是否成功
    if(key == NULL) {
        rc = -1;
        goto  clean_exit;
    }

    # 处理密钥编码
    p = key;

    # 存储密钥类型
    _libssh2_store_str(&p, (const char *)method_buf, 19);

    # 存储名称域
    _libssh2_store_str(&p, (const char *)method_buf + 11, 8);

    # 存储公钥
    _libssh2_store_str(&p, (const char *)octal_value, octal_len);

    # 设置返回值
    *method         = method_buf;
    *method_len     = 19;
    *pubkeydata     = key;
    *pubkeydata_len = key_len;
# 清理并退出函数
clean_exit:
    # 如果 EC 对象不为空，释放 EC 对象
    if(ec != NULL)
        EC_KEY_free(ec);
    # 如果大数上下文不为空，释放大数上下文
    if(bn_ctx != NULL) {
        BN_CTX_free(bn_ctx);
    }
    # 如果八进制值不为空，释放八进制值
    if(octal_value != NULL)
        free(octal_value);
    # 如果返回码为 0，返回 0
    if(rc == 0)
        return 0;
    # 如果方法缓冲区不为空，释放方法缓冲区
    if(method_buf != NULL)
        LIBSSH2_FREE(session, method_buf);
    # 返回 -1
    return -1;
}

# 从 ECDSA OpenSSH 私钥数据生成公钥
static int
gen_publickey_from_ecdsa_openssh_priv_data(LIBSSH2_SESSION *session,
                                           libssh2_curve_type curve_type,
                                           struct string_buf *decrypted,
                                           unsigned char **method,
                                           size_t *method_len,
                                           unsigned char **pubkeydata,
                                           size_t *pubkeydata_len,
                                           libssh2_ecdsa_ctx **ec_ctx)
{
    int rc = 0;
    size_t curvelen, exponentlen, pointlen;
    unsigned char *curve, *exponent, *point_buf;
    EC_KEY *ec_key = NULL;
    BIGNUM *bn_exponent;

    # 调试信息：从私钥数据计算 ECDSA 密钥
    _libssh2_debug(session,
                   LIBSSH2_TRACE_AUTH,
                   "Computing ECDSA keys from private key data");

    # 获取曲线参数
    if(_libssh2_get_string(decrypted, &curve, &curvelen) ||
        curvelen == 0) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "ECDSA no curve");
        return -1;
    }

    # 获取点参数
    if(_libssh2_get_string(decrypted, &point_buf, &pointlen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "ECDSA no point");
        return -1;
    }

    # 获取指数参数
    if(_libssh2_get_bignum_bytes(decrypted, &exponent, &exponentlen)) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "ECDSA no exponent");
        return -1;
    }
    # 如果调用 _libssh2_ecdsa_curve_name_with_octal_new 函数返回值不为 0，则执行以下代码
    if((rc = _libssh2_ecdsa_curve_name_with_octal_new(&ec_key, point_buf,
        pointlen, curve_type)) != 0) {
        # 将返回值设为 -1
        rc = -1;
        # 抛出错误信息
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "ECDSA could not create key");
        # 跳转到 fail 标签处
        goto fail;
    }

    # 为 bn_exponent 分配内存
    bn_exponent = BN_new();
    # 如果分配内存失败，则执行以下代码
    if(bn_exponent == NULL) {
        # 将返回值设为 -1
        rc = -1;
        # 抛出错误信息
        _libssh2_error(session, LIBSSH2_ERROR_ALLOC,
                       "Unable to allocate memory for private key data");
        # 跳转到 fail 标签处
        goto fail;
    }

    # 将二进制指数转换为大数
    BN_bin2bn(exponent, exponentlen, bn_exponent);
    # 将私钥数据设置到 EC_KEY 结构体中
    rc = (EC_KEY_set_private_key(ec_key, bn_exponent) != 1);

    # 如果 rc 为 0 且 ec_key、pubkeydata、method 都不为空，则执行以下代码
    if(rc == 0 && ec_key != NULL && pubkeydata != NULL && method != NULL) {
        # 创建一个 EVP_PKEY 结构体
        EVP_PKEY *pk = EVP_PKEY_new();
        # 将 EC_KEY 结构体设置到 EVP_PKEY 结构体中
        EVP_PKEY_set1_EC_KEY(pk, ec_key);

        # 调用 gen_publickey_from_ec_evp 函数生成公钥
        rc = gen_publickey_from_ec_evp(session, method, method_len,
                                       pubkeydata, pubkeydata_len,
                                       pk);

        # 如果 pk 不为空，则释放 pk
        if(pk)
            EVP_PKEY_free(pk);
    }

    # 如果 ec_ctx 不为空，则将 ec_key 赋值给 ec_ctx，否则释放 ec_key
    if(ec_ctx != NULL)
        *ec_ctx = ec_key;
    else
        EC_KEY_free(ec_key);

    # 返回 rc
    return rc;
# 如果 EC_KEY 对象不为空，则释放其内存
fail:
    if(ec_key != NULL)
        EC_KEY_free(ec_key);

    # 返回 rc 变量的值
    return rc;
}

# 创建一个新的 ECDSA 私钥对象，使用 OpenSSH 格式的私钥文件
static int
_libssh2_ecdsa_new_openssh_private(libssh2_ecdsa_ctx ** ec_ctx,
                                   LIBSSH2_SESSION * session,
                                   const char *filename,
                                   unsigned const char *passphrase)
{
    FILE *fp;
    int rc;
    unsigned char *buf = NULL;
    libssh2_curve_type type;
    struct string_buf *decrypted = NULL;

    # 如果会话对象为空，则返回错误
    if(session == NULL) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                           "Session is required");
        return -1;
    }

    # 初始化 libssh2 库
    _libssh2_init_if_needed();

    # 打开指定文件名的私钥文件
    fp = fopen(filename, "r");
    if(!fp) {
        _libssh2_error(session, LIBSSH2_ERROR_FILE,
                       "Unable to open OpenSSH ECDSA private key file");
        return -1;
    }

    # 解析私钥文件，获取解密后的数据
    rc = _libssh2_openssh_pem_parse(session, passphrase, fp, &decrypted);
    fclose(fp);
    if(rc) {
        return rc;
    }

    # 尝试从解密后的数据中解析公钥类型
    rc = _libssh2_get_string(decrypted, &buf, NULL);

    # 如果解析失败或者 buf 为空，则返回错误
    if(rc != 0 || buf == NULL) {
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Public key type in decrypted key data not found");
        return -1;
    }

    # 根据公钥类型名称获取曲线类型
    rc = _libssh2_ecdsa_curve_type_from_name((const char *)buf, &type);

    # 如果获取成功，则生成 ECDSA 公钥对象
    if(rc == 0) {
        rc = gen_publickey_from_ecdsa_openssh_priv_data(session, type,
                                                        decrypted, NULL, 0,
                                                        NULL, 0, ec_ctx);
    }
    else {
        rc = -1;
    }

    # 释放解密后的数据内存
    if(decrypted)
        _libssh2_string_buf_free(session, decrypted);

    # 返回结果
    return rc;
}

# 创建一个新的 ECDSA 私钥对象
int
_libssh2_ecdsa_new_private(libssh2_ecdsa_ctx ** ec_ctx,
       LIBSSH2_SESSION * session,
       const char *filename, unsigned const char *passphrase)
{
    int rc;

    # 定义一个函数指针，用于读取 ECDSA 私钥数据
    pem_read_bio_func read_ec = (pem_read_bio_func) &PEM_read_bio_ECPrivateKey;
    # 如果需要的话，初始化 libssh2 库
    _libssh2_init_if_needed();
    
    # 从文件中读取私钥，存储到 ec_ctx 中，使用 read_ec 函数读取，传入文件名和密码
    rc = read_private_key_from_file((void **) ec_ctx, read_ec, filename, passphrase);
    
    # 如果读取私钥出错，返回新的 ECDSA 私钥对象
    if(rc) {
        return _libssh2_ecdsa_new_openssh_private(ec_ctx, session, filename, passphrase);
    }
    
    # 返回读取私钥的结果
    return rc;
}

/*
 * _libssh2_ecdsa_create_key
 *
 * 创建基于输入曲线的本地私钥，并返回八进制值和八进制长度
 *
 */

int
_libssh2_ecdsa_create_key(LIBSSH2_SESSION *session,
                          _libssh2_ec_key **out_private_key,
                          unsigned char **out_public_key_octal,
                          size_t *out_public_key_octal_len,
                          libssh2_curve_type curve_type)
{
    int ret = 1;
    size_t octal_len = 0;
    unsigned char octal_value[EC_MAX_POINT_LEN];
    const EC_POINT *public_key = NULL;
    EC_KEY *private_key = NULL;
    const EC_GROUP *group = NULL;

    /* 创建密钥 */
    BN_CTX *bn_ctx = BN_CTX_new();
    if(!bn_ctx)
        return -1;

    private_key = EC_KEY_new_by_curve_name(curve_type);
    group = EC_KEY_get0_group(private_key);

    EC_KEY_generate_key(private_key);
    public_key = EC_KEY_get0_public_key(private_key);

    /* 获取长度 */
    octal_len = EC_POINT_point2oct(group, public_key,
                                   POINT_CONVERSION_UNCOMPRESSED,
                                   NULL, 0, bn_ctx);
    if(octal_len > EC_MAX_POINT_LEN) {
        ret = -1;
        goto clean_exit;
    }

    /* 转换为八进制 */
    if(EC_POINT_point2oct(group, public_key, POINT_CONVERSION_UNCOMPRESSED,
       octal_value, octal_len, bn_ctx) != octal_len) {
           ret = -1;
           goto clean_exit;
    }

    if(out_private_key != NULL)
        *out_private_key = private_key;

    if(out_public_key_octal) {
        *out_public_key_octal = LIBSSH2_ALLOC(session, octal_len);
        if(*out_public_key_octal == NULL) {
            ret = -1;
            goto clean_exit;
        }

        memcpy(*out_public_key_octal, octal_value, octal_len);
    }

    if(out_public_key_octal_len != NULL)
        *out_public_key_octal_len = octal_len;

clean_exit:

    if(bn_ctx)
        BN_CTX_free(bn_ctx);

    return (ret == 1) ? 0 : -1;
}
/* _libssh2_ecdh_gen_k
 *
 * 计算共享密钥 K，给定本地私钥、远程公钥和长度
 */

int
_libssh2_ecdh_gen_k(_libssh2_bn **k, _libssh2_ec_key *private_key,
    const unsigned char *server_public_key, size_t server_public_key_len)
{
    int ret = 0; // 返回值初始化为 0
    int rc; // 定义整型变量 rc
    size_t secret_len; // 定义秘密长度
    unsigned char *secret = NULL; // 初始化秘密为空
    const EC_GROUP *private_key_group; // 定义私钥组
    EC_POINT *server_public_key_point; // 定义远程公钥点

    BN_CTX *bn_ctx = BN_CTX_new(); // 创建大数上下文

    if(!bn_ctx) // 如果上下文为空
        return -1; // 返回 -1

    if(k == NULL) // 如果 k 为空
        return -1; // 返回 -1

    private_key_group = EC_KEY_get0_group(private_key); // 获取私钥组

    server_public_key_point = EC_POINT_new(private_key_group); // 创建远程公钥点
    if(server_public_key_point == NULL) // 如果远程公钥点为空
        return -1; // 返回 -1

    rc = EC_POINT_oct2point(private_key_group, server_public_key_point,
                            server_public_key, server_public_key_len, bn_ctx); // 将远程公钥转换为点
    if(rc != 1) { // 如果转换不成功
        ret = -1; // 返回 -1
        goto clean_exit; // 跳转到清理退出
    }

    secret_len = (EC_GROUP_get_degree(private_key_group) + 7) / 8; // 计算秘密长度
    secret = malloc(secret_len); // 分配秘密内存
    if(!secret) { // 如果秘密为空
        ret = -1; // 返回 -1
        goto clean_exit; // 跳转到清理退出
    }

    secret_len = ECDH_compute_key(secret, secret_len, server_public_key_point,
                                  private_key, NULL); // 计算共享密钥

    if(secret_len <= 0 || secret_len > EC_MAX_POINT_LEN) { // 如果秘密长度小于等于 0 或者大于最大点长度
        ret = -1; // 返回 -1
        goto clean_exit; // 跳转到清理退出
    }

    BN_bin2bn(secret, secret_len, *k); // 将秘密转换为大数

clean_exit:

    if(server_public_key_point != NULL) // 如果远程公钥点不为空
        EC_POINT_free(server_public_key_point); // 释放远程公钥点

    if(bn_ctx != NULL) // 如果上下文不为空
        BN_CTX_free(bn_ctx); // 释放上下文

    if(secret != NULL) // 如果秘密不为空
        free(secret); // 释放秘密

    return ret; // 返回结果
}


#endif /* LIBSSH2_ECDSA */

#if LIBSSH2_ED25519

int
_libssh2_ed25519_sign(libssh2_ed25519_ctx *ctx, LIBSSH2_SESSION *session,
                      uint8_t **out_sig, size_t *out_sig_len,
                      const uint8_t *message, size_t message_len)
{
    int rc = -1; // 初始化 rc 为 -1
    EVP_MD_CTX *md_ctx = EVP_MD_CTX_new(); // 创建消息摘要上下文
    size_t sig_len = 0; // 初始化签名长度为 0
    unsigned char *sig = NULL; // 初始化签名为空
    # 如果消息摘要上下文不为空
    if(md_ctx != NULL) {
        # 初始化签名上下文
        if(EVP_DigestSignInit(md_ctx, NULL, NULL, NULL, ctx) != 1)
            # 跳转到清理退出
            goto clean_exit;
        # 对消息进行签名
        if(EVP_DigestSign(md_ctx, NULL, &sig_len, message, message_len) != 1)
            # 跳转到清理退出
            goto clean_exit;

        # 如果签名长度不等于指定的长度
        if(sig_len != LIBSSH2_ED25519_SIG_LEN)
            # 跳转到清理退出
            goto clean_exit;

        # 分配指定长度的内存给签名
        sig = LIBSSH2_CALLOC(session, sig_len);
        # 如果分配内存失败
        if(sig == NULL)
            # 跳转到清理退出
            goto clean_exit;

        # 对消息进行签名
        rc = EVP_DigestSign(md_ctx, sig, &sig_len, message, message_len);
    }

    # 如果签名成功
    if(rc == 1) {
        # 将签名和签名长度赋值给输出参数
        *out_sig = sig;
        *out_sig_len = sig_len;
    }
    # 如果签名失败
    else {
        # 将签名长度置为0，签名置为空
        *out_sig_len = 0;
        *out_sig = NULL;
        # 释放签名内存
        LIBSSH2_FREE(session, sig);
    }
# 清理并退出函数
clean_exit:

    # 如果消息摘要上下文存在，则释放其内存
    if(md_ctx)
        EVP_MD_CTX_free(md_ctx);

    # 返回成功或失败的结果
    return (rc == 1 ? 0 : -1);
}

# 生成 Curve25519 密钥对
int
_libssh2_curve25519_gen_k(_libssh2_bn **k,
                          uint8_t private_key[LIBSSH2_ED25519_KEY_LEN],
                          uint8_t server_public_key[LIBSSH2_ED25519_KEY_LEN])
{
    # 初始化变量
    int rc = -1;
    unsigned char out_shared_key[LIBSSH2_ED25519_KEY_LEN];
    EVP_PKEY *peer_key = NULL, *server_key = NULL;
    EVP_PKEY_CTX *server_key_ctx = NULL;
    BN_CTX *bn_ctx = NULL;
    size_t out_len = 0;

    # 检查输入参数是否为空
    if(k == NULL || *k == NULL)
        return -1;

    # 创建大数上下文
    bn_ctx = BN_CTX_new();
    if(bn_ctx == NULL)
        return -1;

    # 从服务器公钥创建对等密钥
    peer_key = EVP_PKEY_new_raw_public_key(EVP_PKEY_X25519, NULL,
                                           server_public_key,
                                           LIBSSH2_ED25519_KEY_LEN);

    # 从私钥创建服务器密钥
    server_key = EVP_PKEY_new_raw_private_key(EVP_PKEY_X25519, NULL,
                                              private_key,
                                              LIBSSH2_ED25519_KEY_LEN);

    # 检查密钥是否创建成功
    if(peer_key == NULL || server_key == NULL) {
        goto cleanExit;
    }

    # 创建服务器密钥上下文
    server_key_ctx = EVP_PKEY_CTX_new(server_key, NULL);
    if(server_key_ctx == NULL) {
        goto cleanExit;
    }

    # 初始化密钥协商
    rc = EVP_PKEY_derive_init(server_key_ctx);
    if(rc <= 0) goto cleanExit;

    # 设置对等密钥
    rc = EVP_PKEY_derive_set_peer(server_key_ctx, peer_key);
    if(rc <= 0) goto cleanExit;

    # 执行密钥协商
    rc = EVP_PKEY_derive(server_key_ctx, NULL, &out_len);
    if(rc <= 0) goto cleanExit;

    # 检查协商结果长度是否正确
    if(out_len != LIBSSH2_ED25519_KEY_LEN) {
        rc = -1;
        goto cleanExit;
    }

    # 获取协商结果
    rc = EVP_PKEY_derive(server_key_ctx, out_shared_key, &out_len);

    # 检查协商结果是否成功并且长度正确
    if(rc == 1 && out_len == LIBSSH2_ED25519_KEY_LEN) {
        BN_bin2bn(out_shared_key, LIBSSH2_ED25519_KEY_LEN, *k);
    }
    else {
        rc = -1;
    }

cleanExit:

    # 清理内存
    if(server_key_ctx)
        EVP_PKEY_CTX_free(server_key_ctx);
    if(peer_key)
        EVP_PKEY_free(peer_key);
    # 如果 server_key 存在，则释放其内存
    if(server_key)
        EVP_PKEY_free(server_key);
    # 如果 bn_ctx 不为空，则释放其内存
    if(bn_ctx != NULL)
        BN_CTX_free(bn_ctx);

    # 如果 rc 等于 1，则返回 0，否则返回 -1
    return (rc == 1) ? 0 : -1;
}

int
_libssh2_ed25519_verify(libssh2_ed25519_ctx *ctx, const uint8_t *s,
                        size_t s_len, const uint8_t *m, size_t m_len)
{
    int ret = -1;

    // 创建一个新的消息摘要上下文
    EVP_MD_CTX *md_ctx = EVP_MD_CTX_new();
    if(NULL == md_ctx)
        return -1;

    // 初始化消息摘要上下文以进行验证
    ret = EVP_DigestVerifyInit(md_ctx, NULL, NULL, NULL, ctx);
    if(ret != 1)
        goto clean_exit;

    // 使用消息摘要上下文进行验证
    ret = EVP_DigestVerify(md_ctx, s, s_len, m, m_len);

    clean_exit:

    // 释放消息摘要上下文
    EVP_MD_CTX_free(md_ctx);

    return (ret == 1) ? 0 : -1;
}

#endif /* LIBSSH2_ED25519 */

static int
_libssh2_pub_priv_openssh_keyfile(LIBSSH2_SESSION *session,
                                  unsigned char **method,
                                  size_t *method_len,
                                  unsigned char **pubkeydata,
                                  size_t *pubkeydata_len,
                                  const char *privatekey,
                                  const char *passphrase)
{
    FILE *fp;
    unsigned char *buf = NULL;
    struct string_buf *decrypted = NULL;
    int rc = 0;

    if(session == NULL) {
        // 如果会话为空，则返回错误
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Session is required");
        return -1;
    }

    _libssh2_init_if_needed();

    // 打开私钥文件
    fp = fopen(privatekey, "r");
    if(!fp) {
        // 如果无法打开私钥文件，则返回错误
        _libssh2_error(session, LIBSSH2_ERROR_FILE,
                       "Unable to open private key file");
        return -1;
    }

    // 解析私钥文件
    rc = _libssh2_openssh_pem_parse(session, (const unsigned char *)passphrase,
                                    fp, &decrypted);
    fclose(fp);
    if(rc) {
        // 如果不是一个OpenSSH密钥文件，则返回错误
        _libssh2_error(session, LIBSSH2_ERROR_FILE,
                       "Not an OpenSSH key file");
        return rc;
    }

    /* We have a new key file, now try and parse it using supported types  */
    // 解析新的密钥文件
    rc = _libssh2_get_string(decrypted, &buf, NULL);
    # 如果 rc 不等于 0 或者 buf 为空，则执行以下操作
    if(rc != 0 || buf == NULL) {
        # 在会话中记录错误信息，指明协议错误，公钥类型在解密的密钥数据中未找到
        _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                       "Public key type in decrypted key data not found");
        # 返回 -1，表示出现错误
        return -1;
    }
    
    # 将 rc 设置为 -1
    rc = -1;
#if LIBSSH2_ED25519
    # 如果支持 ED25519 加密算法，比较 buf 中的字符串是否为 "ssh-ed25519"，如果是则调用相应的函数生成公钥
    if(strcmp("ssh-ed25519", (const char *)buf) == 0) {
        rc = gen_publickey_from_ed25519_openssh_priv_data(session, decrypted,
                                                          method, method_len,
                                                          pubkeydata,
                                                          pubkeydata_len,
                                                          NULL);
    }
#endif
#if LIBSSH2_RSA
    # 如果支持 RSA 加密算法，比较 buf 中的字符串是否为 "ssh-rsa"，如果是则调用相应的函数生成公钥
    if(strcmp("ssh-rsa", (const char *)buf) == 0) {
        rc = gen_publickey_from_rsa_openssh_priv_data(session, decrypted,
                                                      method, method_len,
                                                      pubkeydata,
                                                      pubkeydata_len,
                                                      NULL);
    }
#endif
#if LIBSSH2_DSA
    # 如果支持 DSA 加密算法，比较 buf 中的字符串是否为 "ssh-dss"，如果是则调用相应的函数生成公钥
    if(strcmp("ssh-dss", (const char *)buf) == 0) {
        rc = gen_publickey_from_dsa_openssh_priv_data(session, decrypted,
                                                      method, method_len,
                                                      pubkeydata,
                                                      pubkeydata_len,
                                                      NULL);
    }
#endif
#if LIBSSH2_ECDSA
    # 如果支持 ECDSA 加密算法，根据 buf 中的字符串获取曲线类型，然后调用相应的函数生成公钥
    {
        libssh2_curve_type type;

        if(_libssh2_ecdsa_curve_type_from_name((const char *)buf,
                                               &type) == 0) {
            rc = gen_publickey_from_ecdsa_openssh_priv_data(session, type,
                                                            decrypted,
                                                            method, method_len,
                                                            pubkeydata,
                                                            pubkeydata_len,
                                                            NULL);
        }
    }
#endif
    # 如果解密成功，则释放解密后的数据缓冲区
    if(decrypted)
        _libssh2_string_buf_free(session, decrypted);

    # 如果返回值不为0，表示不支持的OpenSSH密钥类型，记录错误信息
    if(rc != 0) {
        _libssh2_error(session, LIBSSH2_ERROR_FILE,
                       "Unsupported OpenSSH key type");
    }

    # 返回函数执行结果
    return rc;
}

int
_libssh2_pub_priv_keyfile(LIBSSH2_SESSION *session,
                          unsigned char **method,
                          size_t *method_len,
                          unsigned char **pubkeydata,
                          size_t *pubkeydata_len,
                          const char *privatekey,
                          const char *passphrase)
{
    int       st;  // 定义整型变量 st
    BIO*      bp;  // 定义 BIO 结构指针 bp
    EVP_PKEY* pk;  // 定义 EVP_PKEY 结构指针 pk
    int       pktype;  // 定义整型变量 pktype
    int       rc;  // 定义整型变量 rc

    _libssh2_debug(session,  // 调用 _libssh2_debug 函数，用于调试输出
                   LIBSSH2_TRACE_AUTH,  // 调试输出的级别
                   "Computing public key from private key file: %s",  // 输出的调试信息
                   privatekey);  // 输出的私钥文件名

    bp = BIO_new_file(privatekey, "r");  // 打开私钥文件，创建 BIO 结构指针 bp
    if(bp == NULL) {  // 判断 bp 是否为空
        return _libssh2_error(session,  // 返回错误信息
                              LIBSSH2_ERROR_FILE,  // 错误类型
                              "Unable to extract public key from private key "
                              "file: Unable to open private key file");  // 错误信息内容
    }

    BIO_reset(bp);  // 重置 BIO 结构指针 bp
    pk = PEM_read_bio_PrivateKey(bp, NULL, NULL, (void *)passphrase);  // 从 BIO 结构指针 bp 中读取私钥
    BIO_free(bp);  // 释放 BIO 结构指针 bp

    if(pk == NULL) {  // 判断 pk 是否为空

        /* Try OpenSSH format */
        rc = _libssh2_pub_priv_openssh_keyfile(session,  // 调用 _libssh2_pub_priv_openssh_keyfile 函数
                                               method,  // 方法
                                               method_len,  // 方法长度
                                               pubkeydata, pubkeydata_len,  // 公钥数据及长度
                                               privatekey, passphrase);  // 私钥文件名及密码
        if(rc != 0) {  // 判断 rc 是否为 0
            return _libssh2_error(session,  // 返回错误信息
                                  LIBSSH2_ERROR_FILE,  // 错误类型
                                  "Unable to extract public key "
                                  "from private key file: "
                                  "Wrong passphrase or invalid/unrecognized "
                                  "private key file format");  // 错误信息内容
        }

        return 0;  // 返回 0
    }

#ifdef HAVE_OPAQUE_STRUCTS
    pktype = EVP_PKEY_id(pk);  // 获取私钥类型
#else
    pktype = pk->type;  // 获取私钥类型
#endif

    switch(pktype) {  // 根据私钥类型进行判断
#if LIBSSH2_ED25519
    # 如果是 ED25519 类型的公钥
    case EVP_PKEY_ED25519 :
        # 调用函数生成 ED25519 类型的公钥
        st = gen_publickey_from_ed_evp(
            session, method, method_len, pubkeydata, pubkeydata_len, pk);
        # 跳出循环
        break;
#endif /* LIBSSH2_ED25519 */
    // 如果编译选项中包含 ED25519，执行以下代码
    case EVP_PKEY_RSA :
        // 使用 RSA EVP 公钥生成方法生成公钥
        st = gen_publickey_from_rsa_evp(
            session, method, method_len, pubkeydata, pubkeydata_len, pk);
        break;

#if LIBSSH2_DSA
    case EVP_PKEY_DSA :
        // 如果编译选项中包含 DSA，使用 DSA EVP 公钥生成方法生成公钥
        st = gen_publickey_from_dsa_evp(
            session, method, method_len, pubkeydata, pubkeydata_len, pk);
        break;
#endif /* LIBSSH_DSA */

#if LIBSSH2_ECDSA
    case EVP_PKEY_EC :
        // 如果编译选项中包含 ECDSA，使用 EC EVP 公钥生成方法生成公钥
        st = gen_publickey_from_ec_evp(
            session, method, method_len, pubkeydata, pubkeydata_len, pk);
    break;
#endif

    default :
        // 如果不支持以上任何一种私钥格式，返回错误信息
        st = _libssh2_error(session,
                            LIBSSH2_ERROR_FILE,
                            "Unable to extract public key "
                            "from private key file: "
                            "Unsupported private key file format");
        break;
    }

    // 释放 EVP_PKEY 结构体占用的内存
    EVP_PKEY_free(pk);
    // 返回生成公钥的状态
    return st;
}

static int
_libssh2_pub_priv_openssh_keyfilememory(LIBSSH2_SESSION *session,
                                        void **key_ctx,
                                        const char *key_type,
                                        unsigned char **method,
                                        size_t *method_len,
                                        unsigned char **pubkeydata,
                                        size_t *pubkeydata_len,
                                        const char *privatekeydata,
                                        size_t privatekeydata_len,
                                        unsigned const char *passphrase)
{
    int rc;
    unsigned char *buf = NULL;
    struct string_buf *decrypted = NULL;

    // 如果 key_ctx 不为空，将其置为 NULL
    if(key_ctx != NULL)
        *key_ctx = NULL;

    // 如果 session 为空，返回协议错误
    if(session == NULL)
        return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                              "Session is required");
    # 检查 key_type 是否不为空且长度大于11或小于7，如果是则返回类型无效的错误
    if(key_type != NULL && (strlen(key_type) > 11 || strlen(key_type) < 7))
        return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                              "type is invalid");

    # 初始化 libssh2 库
    _libssh2_init_if_needed();

    # 解析 PEM 格式的私钥数据
    rc = _libssh2_openssh_pem_parse_memory(session, passphrase,
                                           privatekeydata,
                                           privatekeydata_len, &decrypted);

    # 如果解析出错，则返回错误码
    if(rc)
        return rc;

   # 解析解密后的数据，获取公钥类型
   rc = _libssh2_get_string(decrypted, &buf, NULL);

   # 如果解析出错或者 buf 为空，则返回公钥类型未找到的错误
   if(rc != 0 || buf == NULL)
       return _libssh2_error(session, LIBSSH2_ERROR_PROTO,
                             "Public key type in decrypted "
                             "key data not found");

   # 设置错误码为文件错误
   rc = LIBSSH2_ERROR_FILE;
#if LIBSSH2_ED25519
    # 如果支持 ED25519 算法
    if(strcmp("ssh-ed25519", (const char *)buf) == 0) {
        # 如果私钥中的算法是 ED25519
        if(key_type == NULL || strcmp("ssh-ed25519", key_type) == 0) {
            # 从 Openssh 格式的私钥数据生成 ED25519 公钥
            rc = gen_publickey_from_ed25519_openssh_priv_data(session,
                                                              decrypted,
                                                              method,
                                                              method_len,
                                                              pubkeydata,
                                                              pubkeydata_len,
                                              (libssh2_ed25519_ctx**)key_ctx);
        }
   }
#endif
#if LIBSSH2_RSA
    # 如果支持 RSA 算法
    if(strcmp("ssh-rsa", (const char *)buf) == 0) {
        # 如果私钥中的算法是 RSA
        if(key_type == NULL || strcmp("ssh-rsa", key_type) == 0) {
            # 从 Openssh 格式的私钥数据生成 RSA 公钥
            rc = gen_publickey_from_rsa_openssh_priv_data(session, decrypted,
                                                          method, method_len,
                                                          pubkeydata,
                                                          pubkeydata_len,
                                                (libssh2_rsa_ctx**)key_ctx);
        }
   }
#endif
#if LIBSSH2_DSA
    # 如果支持 DSA 算法
    if(strcmp("ssh-dss", (const char *)buf) == 0) {
        # 如果私钥中的算法是 DSA
        if(key_type == NULL || strcmp("ssh-dss", key_type) == 0) {
            # 从 Openssh 格式的私钥数据生成 DSA 公钥
            rc = gen_publickey_from_dsa_openssh_priv_data(session, decrypted,
                                                         method, method_len,
                                                          pubkeydata,
                                                          pubkeydata_len,
                                                 (libssh2_dsa_ctx**)key_ctx);
        }
   }
#endif
#if LIBSSH2_ECDSA
{
   // 声明一个变量 type，用于存储 libssh2_curve_type 类型的数据
   libssh2_curve_type type;

   // 如果能够从 buf 中获取到有效的 ecdsa 曲线类型
   if(_libssh2_ecdsa_curve_type_from_name((const char *)buf, &type) == 0) {
       // 如果 key_type 为空，或者 key_type 为 "ssh-ecdsa"
       if(key_type == NULL || strcmp("ssh-ecdsa", key_type) == 0) {
           // 调用 gen_publickey_from_ecdsa_openssh_priv_data 函数生成 ecdsa 的公钥
           rc = gen_publickey_from_ecdsa_openssh_priv_data(session, type,
                                                           decrypted,
                                                           method, method_len,
                                                           pubkeydata,
                                                           pubkeydata_len,
                                               (libssh2_ecdsa_ctx**)key_ctx);
        }
    }
}
#endif

// 如果 rc 的值为 LIBSSH2_ERROR_FILE
if(rc == LIBSSH2_ERROR_FILE)
    // 将错误信息添加到错误栈中
    rc = _libssh2_error(session, LIBSSH2_ERROR_FILE,
                     "Unable to extract public key from private key file: "
                     "invalid/unrecognized private key file format");

// 如果 decrypted 不为空
if(decrypted)
    // 释放 decrypted 占用的内存
    _libssh2_string_buf_free(session, decrypted);

// 返回 rc 的值
return rc;
}

// 从内存中读取 openssh 私钥
int
read_openssh_private_key_from_memory(void **key_ctx, LIBSSH2_SESSION *session,
                                     const char *key_type,
                                     const char *filedata,
                                     size_t filedata_len,
                                     unsigned const char *passphrase)
{
    // 调用 _libssh2_pub_priv_openssh_keyfilememory 函数，从内存中读取 openssh 私钥
    return _libssh2_pub_priv_openssh_keyfilememory(session, key_ctx, key_type,
                                                   NULL, NULL, NULL, NULL,
                                                   filedata, filedata_len,
                                                   passphrase);
}

// 结束
int
# 从私钥数据中计算公钥，并返回公钥的方法和数据
def _libssh2_pub_priv_keyfilememory(LIBSSH2_SESSION *session,
                                    unsigned char **method,
                                    size_t *method_len,
                                    unsigned char **pubkeydata,
                                    size_t *pubkeydata_len,
                                    const char *privatekeydata,
                                    size_t privatekeydata_len,
                                    const char *passphrase)
{
    int       st;  # 定义整型变量 st
    BIO*      bp;  # 定义 BIO 对象指针 bp
    EVP_PKEY* pk;  # 定义 EVP_PKEY 对象指针 pk
    int       pktype;  # 定义整型变量 pktype

    _libssh2_debug(session,  # 调用 _libssh2_debug 函数，输出调试信息
                   LIBSSH2_TRACE_AUTH,  # 传入调试信息类型参数
                   "Computing public key from private key.")  # 传入调试信息内容参数

    bp = BIO_new_mem_buf((char *)privatekeydata, privatekeydata_len);  # 使用私钥数据创建内存缓冲区 BIO 对象
    if(!bp)  # 如果创建失败
        return _libssh2_error(session, LIBSSH2_ERROR_ALLOC,  # 返回分配内存失败的错误信息
                              "Unable to allocate memory when"
                              "computing public key")
    BIO_reset(bp);  # 重置 BIO 对象
    pk = PEM_read_bio_PrivateKey(bp, NULL, NULL, (void *)passphrase);  # 从 BIO 对象中读取私钥数据，返回 EVP_PKEY 对象
    BIO_free(bp);  # 释放 BIO 对象

    if(pk == NULL) {  # 如果未成功读取私钥数据
        # 尝试 OpenSSH 格式
        st = _libssh2_pub_priv_openssh_keyfilememory(session, NULL, NULL,  # 调用 _libssh2_pub_priv_openssh_keyfilememory 函数
                                                     method,  # 传入方法参数
                                                     method_len,  # 传入方法长度参数
                                                     pubkeydata,  # 传入公钥数据参数
                                                     pubkeydata_len,  # 传入公钥数据长度参数
                                                     privatekeydata,  # 传入私钥数据参数
                                                     privatekeydata_len,  # 传入私钥数据长度参数
                                           (unsigned const char *)passphrase);  # 传入口令参数
        if(st != 0)  # 如果执行失败
            return st  # 返回执行失败的状态
        return 0  # 返回成功状态
    }

#ifdef HAVE_OPAQUE_STRUCTS
    pktype = EVP_PKEY_id(pk);  # 获取 EVP_PKEY 对象的 ID
#else
    pktype = pk->type;  # 获取 EVP_PKEY 对象的类型
#endif

    switch(pktype) {  # 根据公钥类型进行判断
#if LIBSSH2_ED25519
    # 如果算法是 EVP_PKEY_ED25519，则调用 gen_publickey_from_ed_evp 函数生成公钥
    case EVP_PKEY_ED25519 :
        st = gen_publickey_from_ed_evp(
            session, method, method_len, pubkeydata, pubkeydata_len, pk);
        # 跳出循环
        break;
#endif /* LIBSSH2_ED25519 */
    # 如果编译选项中包含 ED25519，执行以下代码
    case EVP_PKEY_RSA :
        # 使用 RSA 公钥生成函数生成公钥
        st = gen_publickey_from_rsa_evp(session, method, method_len,
                                        pubkeydata, pubkeydata_len, pk);
        break;
#if LIBSSH2_DSA
    case EVP_PKEY_DSA :
        # 如果编译选项中包含 DSA，使用 DSA 公钥生成函数生成公钥
        st = gen_publickey_from_dsa_evp(session, method, method_len,
                                        pubkeydata, pubkeydata_len, pk);
        break;
#endif /* LIBSSH_DSA */
#if LIBSSH2_ECDSA
    case EVP_PKEY_EC :
        # 如果编译选项中包含 ECDSA，使用 EC 公钥生成函数生成公钥
        st = gen_publickey_from_ec_evp(session, method, method_len,
                                       pubkeydata, pubkeydata_len, pk);
        break;
#endif /* LIBSSH2_ECDSA */
    default :
        # 如果以上条件都不满足，返回错误信息
        st = _libssh2_error(session,
                            LIBSSH2_ERROR_FILE,
                            "Unable to extract public key "
                            "from private key file: "
                            "Unsupported private key file format");
        break;
    }
    # 释放公钥对象
    EVP_PKEY_free(pk);
    # 返回结果
    return st;
}

void
_libssh2_dh_init(_libssh2_dh_ctx *dhctx)
{
    # 初始化 DH 上下文
    *dhctx = BN_new();                          /* Random from client */
}

int
_libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                     _libssh2_bn *g, _libssh2_bn *p, int group_order,
                     _libssh2_bn_ctx *bnctx)
{
    /* 生成 x 和 e */
    BN_rand(*dhctx, group_order * 8 - 1, 0, -1);
    # 计算公钥
    BN_mod_exp(public, g, *dhctx, p, bnctx);
    return 0;
}

int
_libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                   _libssh2_bn *f, _libssh2_bn *p,
                   _libssh2_bn_ctx *bnctx)
{
    /* 计算共享密钥 */
    BN_mod_exp(secret, f, *dhctx, p, bnctx);
    return 0;
}

void
_libssh2_dh_dtor(_libssh2_dh_ctx *dhctx)
{
    # 清除并释放 DH 上下文
    BN_clear_free(*dhctx);
    *dhctx = NULL;
}

#endif /* LIBSSH2_OPENSSL */
```