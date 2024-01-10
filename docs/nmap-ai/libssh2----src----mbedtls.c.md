# `nmap\libssh2\src\mbedtls.c`

```
/*
 * 版权声明，版权所有
 * 未经特定条件的许可，不得使用、复制、修改和分发本软件
 * 如果满足以下条件，可以在源代码和二进制形式下进行重新分发
 * 1. 必须保留上述版权声明、条件列表和以下免责声明
 * 2. 在文档和/或其他提供的材料中，必须重现上述版权声明、条件列表和以下免责声明
 * 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品
 * 本软件由版权所有者和贡献者按原样提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的适用性担保
 * 无论在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）产生的任何损害，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责
 */

#include "libssh2_priv.h"

#ifdef LIBSSH2_MBEDTLS /* 只有在构建时使用 mbedtls 时才编译 */

/*******************************************************************/
/*
 * mbedTLS 后端：全局上下文句柄
 */

static mbedtls_entropy_context  _libssh2_mbedtls_entropy; // mbedTLS 熵上下文
static mbedtls_ctr_drbg_context _libssh2_mbedtls_ctr_drbg; // mbedTLS 随机数生成器上下文
/*******************************************************************/
/*
 * mbedTLS backend: Generic functions
 */

void
_libssh2_mbedtls_init(void)
{
    int ret;

    // 初始化随机数生成器的熵
    mbedtls_entropy_init(&_libssh2_mbedtls_entropy);
    // 初始化随机数生成器的伪随机数生成器
    mbedtls_ctr_drbg_init(&_libssh2_mbedtls_ctr_drbg);

    // 为随机数生成器设置种子
    ret = mbedtls_ctr_drbg_seed(&_libssh2_mbedtls_ctr_drbg,
                                mbedtls_entropy_func,
                                &_libssh2_mbedtls_entropy, NULL, 0);
    if(ret != 0)
        mbedtls_ctr_drbg_free(&_libssh2_mbedtls_ctr_drbg);
}

void
_libssh2_mbedtls_free(void)
{
    // 释放随机数生成器的伪随机数生成器
    mbedtls_ctr_drbg_free(&_libssh2_mbedtls_ctr_drbg);
    // 释放随机数生成器的熵
    mbedtls_entropy_free(&_libssh2_mbedtls_entropy);
}

int
_libssh2_mbedtls_random(unsigned char *buf, int len)
{
    int ret;
    // 生成随机数
    ret = mbedtls_ctr_drbg_random(&_libssh2_mbedtls_ctr_drbg, buf, len);
    return ret == 0 ? 0 : -1;
}

static void
_libssh2_mbedtls_safe_free(void *buf, int len)
{
#ifndef LIBSSH2_CLEAR_MEMORY
    (void)len;
#endif

    if(!buf)
        return;

#ifdef LIBSSH2_CLEAR_MEMORY
    if(len > 0)
        _libssh2_explicit_zero(buf, len);
#endif

    // 释放内存
    mbedtls_free(buf);
}

int
_libssh2_mbedtls_cipher_init(_libssh2_cipher_ctx *ctx,
                             _libssh2_cipher_type(algo),
                             unsigned char *iv,
                             unsigned char *secret,
                             int encrypt)
{
    const mbedtls_cipher_info_t *cipher_info;
    int ret, op;

    if(!ctx)
        return -1;

    op = encrypt == 0 ? MBEDTLS_ENCRYPT : MBEDTLS_DECRYPT;

    // 获取加密算法信息
    cipher_info = mbedtls_cipher_info_from_type(algo);
    if(!cipher_info)
        return -1;

    // 初始化加密上下文
    mbedtls_cipher_init(ctx);
    // 设置加密算法
    ret = mbedtls_cipher_setup(ctx, cipher_info);
    if(!ret)
        ret = mbedtls_cipher_setkey(ctx, secret, cipher_info->key_bitlen, op);

    if(!ret)
        ret = mbedtls_cipher_set_iv(ctx, iv, cipher_info->iv_size);

    return ret == 0 ? 0 : -1;
}

int
# 使用 mbedtls 加密算法对数据进行加密或解密
_libssh2_mbedtls_cipher_crypt(_libssh2_cipher_ctx *ctx,
                              _libssh2_cipher_type(algo),
                              int encrypt,
                              unsigned char *block,
                              size_t blocklen)
{
    int ret;  # 定义整型变量 ret，用于存储函数返回值
    unsigned char *output;  # 定义指向无符号字符的指针 output
    size_t osize, olen, finish_olen;  # 定义三个大小变量

    (void) encrypt;  # 忽略 encrypt 参数
    (void) algo;  # 忽略 algo 参数

    osize = blocklen + mbedtls_cipher_get_block_size(ctx);  # 计算输出大小

    output = (unsigned char *)mbedtls_calloc(osize, sizeof(char));  # 分配内存给 output
    if(output) {  # 如果内存分配成功
        ret = mbedtls_cipher_reset(ctx);  # 重置加密上下文

        if(!ret)  # 如果返回值为 0
            ret = mbedtls_cipher_update(ctx, block, blocklen, output, &olen);  # 更新加密上下文

        if(!ret)  # 如果返回值为 0
            ret = mbedtls_cipher_finish(ctx, output + olen, &finish_olen);  # 结束加密过程

        if(!ret) {  # 如果返回值为 0
            olen += finish_olen;  # 更新 olen
            memcpy(block, output, olen);  # 复制加密结果到 block
        }

        _libssh2_mbedtls_safe_free(output, osize);  # 释放内存
    }
    else
        ret = -1;  # 如果内存分配失败，设置返回值为 -1

    return ret == 0 ? 0 : -1;  # 如果返回值为 0，则返回 0，否则返回 -1
}

# 释放 mbedtls 加密算法上下文
void
_libssh2_mbedtls_cipher_dtor(_libssh2_cipher_ctx *ctx)
{
    mbedtls_cipher_free(ctx);  # 释放加密算法上下文
}


# 初始化 mbedtls 哈希算法上下文
int
_libssh2_mbedtls_hash_init(mbedtls_md_context_t *ctx,
                          mbedtls_md_type_t mdtype,
                          const unsigned char *key, unsigned long keylen)
{
    const mbedtls_md_info_t *md_info;  # 定义指向 mbedtls_md_info_t 结构体的指针
    int ret, hmac;  # 定义两个整型变量

    md_info = mbedtls_md_info_from_type(mdtype);  # 获取哈希算法信息
    if(!md_info)  # 如果哈希算法信息为空
        return 0;  # 返回 0

    hmac = key == NULL ? 0 : 1;  # 判断是否使用 HMAC

    mbedtls_md_init(ctx);  # 初始化哈希算法上下文
    ret = mbedtls_md_setup(ctx, md_info, hmac);  # 设置哈希算法上下文
    if(!ret) {  # 如果返回值为 0
        if(hmac)  # 如果使用 HMAC
            ret = mbedtls_md_hmac_starts(ctx, key, keylen);  # 开始 HMAC 运算
        else
            ret = mbedtls_md_starts(ctx);  # 开始哈希运算
    }

    return ret == 0 ? 1 : 0;  # 如果返回值为 0，则返回 1，否则返回 0
}

# 结束 mbedtls 哈希算法运算
int
_libssh2_mbedtls_hash_final(mbedtls_md_context_t *ctx, unsigned char *hash)
{
    int ret;  # 定义整型变量 ret

    ret = mbedtls_md_finish(ctx, hash);  # 结束哈希运算
    mbedtls_md_free(ctx);  # 释放哈希算法上下文

    return ret == 0 ? 0 : -1;  # 如果返回值为 0，则返回 0，否则返回 -1
}

# ...
# 计算给定数据的哈希值
_libssh2_mbedtls_hash(const unsigned char *data, unsigned long datalen,
                      mbedtls_md_type_t mdtype, unsigned char *hash)
{
    const mbedtls_md_info_t *md_info;
    int ret;

    # 根据哈希类型获取哈希信息
    md_info = mbedtls_md_info_from_type(mdtype);
    if(!md_info)
        return 0;

    # 计算哈希值
    ret = mbedtls_md(md_info, data, datalen, hash);

    return ret == 0 ? 0 : -1;
}

/*******************************************************************/
/*
 * mbedTLS backend: BigNumber functions
 */

# 初始化大数
_libssh2_bn *
_libssh2_mbedtls_bignum_init(void)
{
    _libssh2_bn *bignum;

    # 分配内存并初始化大数
    bignum = (_libssh2_bn *)mbedtls_calloc(1, sizeof(_libssh2_bn));
    if(bignum) {
        mbedtls_mpi_init(bignum);
    }

    return bignum;
}

# 释放大数
void
_libssh2_mbedtls_bignum_free(_libssh2_bn *bn)
{
    if(bn) {
        mbedtls_mpi_free(bn);
        mbedtls_free(bn);
    }
}

# 生成随机大数
static int
_libssh2_mbedtls_bignum_random(_libssh2_bn *bn, int bits, int top, int bottom)
{
    size_t len;
    int err;
    int i;

    if(!bn || bits <= 0)
        return -1;

    # 计算大数的长度
    len = (bits + 7) >> 3;
    # 生成随机数
    err = mbedtls_mpi_fill_random(bn, len, mbedtls_ctr_drbg_random,
                                  &_libssh2_mbedtls_ctr_drbg);
    if(err)
        return -1;

    # 将未使用的位设置为零
    for(i = len*8 - 1; bits <= i; --i) {
        err = mbedtls_mpi_set_bit(bn, i, 0);
        if(err)
            return -1;
    }

    # 根据参数设置最高位的值
    for(i = 0; i <= top; ++i) {
        err = mbedtls_mpi_set_bit(bn, bits-i-1, 1);
        if(err)
            return -1;
    }

    # 通过设置最低有效字节的第一位使大数变为奇数
    # 如果条件为真，执行以下代码块
    if(bottom) {
        # 设置大数对象的指定位为 1
        err = mbedtls_mpi_set_bit(bn, 0, 1);
        # 如果设置失败，返回错误码 -1
        if(err)
            return -1;
    }
    # 返回成功码 0
    return 0;
# mbedTLS后端：RSA函数
int
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
                        unsigned long coefflen)
{
    int ret;
    libssh2_rsa_ctx *ctx;

    # 分配内存给RSA上下文
    ctx = (libssh2_rsa_ctx *) mbedtls_calloc(1, sizeof(libssh2_rsa_ctx));
    if(ctx != NULL) {
        # 初始化RSA上下文
        mbedtls_rsa_init(ctx, MBEDTLS_RSA_PKCS_V15, 0);
    }
    else
        return -1;

    /* !checksrc! disable ASSIGNWITHINCONDITION 1 */
    # 从二进制数据中读取E和N的值
    if((ret = mbedtls_mpi_read_binary(&(ctx->E), edata, elen) ) != 0 ||
       (ret = mbedtls_mpi_read_binary(&(ctx->N), ndata, nlen) ) != 0) {
        ret = -1;
    }

    # 如果没有错误，计算N的长度
    if(!ret) {
        ctx->len = mbedtls_mpi_size(&(ctx->N));
    }
    # 如果 ret 为假且 ddata 存在
    if(!ret && ddata) {
        /* !checksrc! disable ASSIGNWITHINCONDITION 1 */
        # 从二进制数据中读取 D，并将结果赋给 ret；如果结果不为 0，则继续执行下一个赋值操作
        if((ret = mbedtls_mpi_read_binary(&(ctx->D), ddata, dlen) ) != 0 ||
           # 从二进制数据中读取 P，并将结果赋给 ret；如果结果不为 0，则继续执行下一个赋值操作
           (ret = mbedtls_mpi_read_binary(&(ctx->P), pdata, plen) ) != 0 ||
           # 从二进制数据中读取 Q，并将结果赋给 ret；如果结果不为 0，则继续执行下一个赋值操作
           (ret = mbedtls_mpi_read_binary(&(ctx->Q), qdata, qlen) ) != 0 ||
           # 从二进制数据中读取 DP，并将结果赋给 ret；如果结果不为 0，则继续执行下一个赋值操作
           (ret = mbedtls_mpi_read_binary(&(ctx->DP), e1data, e1len) ) != 0 ||
           # 从二进制数据中读取 DQ，并将结果赋给 ret；如果结果不为 0，则继续执行下一个赋值操作
           (ret = mbedtls_mpi_read_binary(&(ctx->DQ), e2data, e2len) ) != 0 ||
           # 从二进制数据中读取 QP，并将结果赋给 ret；如果结果不为 0，则继续执行下一个赋值操作
           (ret = mbedtls_mpi_read_binary(&(ctx->QP), coeffdata, coefflen) )
           != 0) {
            # 如果有任何一个赋值操作的结果不为 0，则将 ret 设为 -1
            ret = -1;
        }
        # 检查私钥是否有效
        ret = mbedtls_rsa_check_privkey(ctx);
    }
    # 如果 ret 为假
    else if(!ret) {
        # 检查公钥是否有效
        ret = mbedtls_rsa_check_pubkey(ctx);
    }

    # 如果 ret 为真且 ctx 存在
    if(ret && ctx) {
        # 释放 RSA 上下文
        _libssh2_mbedtls_rsa_free(ctx);
        # 将 ctx 设为 NULL
        ctx = NULL;
    }
    # 将 ctx 赋给 rsa
    *rsa = ctx;
    # 返回 ret
    return ret;
}

int
_libssh2_mbedtls_rsa_new_private(libssh2_rsa_ctx **rsa,
                                LIBSSH2_SESSION *session,
                                const char *filename,
                                const unsigned char *passphrase)
{
    int ret;
    mbedtls_pk_context pkey;
    mbedtls_rsa_context *pk_rsa;

    // 分配内存给 RSA 上下文
    *rsa = (libssh2_rsa_ctx *) LIBSSH2_ALLOC(session, sizeof(libssh2_rsa_ctx));
    if(*rsa == NULL)
        return -1;

    // 初始化 RSA 上下文
    mbedtls_rsa_init(*rsa, MBEDTLS_RSA_PKCS_V15, 0);
    mbedtls_pk_init(&pkey);

    // 从文件中解析私钥
    ret = mbedtls_pk_parse_keyfile(&pkey, filename, (char *)passphrase);
    if(ret != 0 || mbedtls_pk_get_type(&pkey) != MBEDTLS_PK_RSA) {
        mbedtls_pk_free(&pkey);
        mbedtls_rsa_free(*rsa);
        LIBSSH2_FREE(session, *rsa);
        *rsa = NULL;
        return -1;
    }

    // 获取 RSA 上下文
    pk_rsa = mbedtls_pk_rsa(pkey);
    // 复制 RSA 上下文
    mbedtls_rsa_copy(*rsa, pk_rsa);
    mbedtls_pk_free(&pkey);

    return 0;
}

int
_libssh2_mbedtls_rsa_new_private_frommemory(libssh2_rsa_ctx **rsa,
                                           LIBSSH2_SESSION *session,
                                           const char *filedata,
                                           size_t filedata_len,
                                           unsigned const char *passphrase)
{
    int ret;
    mbedtls_pk_context pkey;
    mbedtls_rsa_context *pk_rsa;
    void *filedata_nullterm;
    size_t pwd_len;

    // 分配内存给 RSA 上下文
    *rsa = (libssh2_rsa_ctx *) mbedtls_calloc(1, sizeof(libssh2_rsa_ctx));
    if(*rsa == NULL)
        return -1;

    /*
    mbedtls checks in "mbedtls/pkparse.c:1184" if "key[keylen - 1] != '\0'"
    private-key from memory will fail if the last byte is not a null byte
    */
    // 为文件数据添加结尾的空字符
    filedata_nullterm = mbedtls_calloc(filedata_len + 1, 1);
    if(filedata_nullterm == NULL) {
        return -1;
    }
    memcpy(filedata_nullterm, filedata, filedata_len);

    mbedtls_pk_init(&pkey);

    // 计算密码长度
    pwd_len = passphrase != NULL ? strlen((const char *)passphrase) : 0;
    # 使用 mbedtls_pk_parse_key 函数解析密钥文件数据，存储到 pkey 中
    ret = mbedtls_pk_parse_key(&pkey, (unsigned char *)filedata_nullterm,
                               filedata_len + 1,
                               passphrase, pwd_len);
    # 释放 filedata_nullterm 指向的内存空间
    _libssh2_mbedtls_safe_free(filedata_nullterm, filedata_len);

    # 如果解析密钥文件数据失败，或者密钥类型不是 MBEDTLS_PK_RSA，则执行以下操作
    if(ret != 0 || mbedtls_pk_get_type(&pkey) != MBEDTLS_PK_RSA) {
        # 释放 pkey 占用的内存空间
        mbedtls_pk_free(&pkey);
        # 释放 *rsa 占用的内存空间
        mbedtls_rsa_free(*rsa);
        # 释放 session 占用的内存空间
        LIBSSH2_FREE(session, *rsa);
        # 将 *rsa 指向 NULL
        *rsa = NULL;
        # 返回 -1，表示解析密钥文件失败
        return -1;
    }

    # 获取 pkey 中的 RSA 密钥
    pk_rsa = mbedtls_pk_rsa(pkey);
    # 复制 pk_rsa 中的 RSA 密钥到 *rsa 中
    mbedtls_rsa_copy(*rsa, pk_rsa);
    # 释放 pkey 占用的内存空间
    mbedtls_pk_free(&pkey);

    # 返回 0，表示成功解析并复制 RSA 密钥
    return 0;
}

int
_libssh2_mbedtls_rsa_sha1_verify(libssh2_rsa_ctx *rsa,
                                const unsigned char *sig,
                                unsigned long sig_len,
                                const unsigned char *m,
                                unsigned long m_len)
{
    // 创建一个用于存储 SHA1 哈希值的数组
    unsigned char hash[SHA_DIGEST_LENGTH];
    int ret;

    // 计算消息 m 的 SHA1 哈希值
    ret = _libssh2_mbedtls_hash(m, m_len, MBEDTLS_MD_SHA1, hash);
    if(ret)
        return -1; /* 失败 */

    // 使用 RSA 公钥验证签名
    ret = mbedtls_rsa_pkcs1_verify(rsa, NULL, NULL, MBEDTLS_RSA_PUBLIC,
                                   MBEDTLS_MD_SHA1, SHA_DIGEST_LENGTH,
                                   hash, sig);

    return (ret == 0) ? 0 : -1;
}

int
_libssh2_mbedtls_rsa_sha1_sign(LIBSSH2_SESSION *session,
                              libssh2_rsa_ctx *rsa,
                              const unsigned char *hash,
                              size_t hash_len,
                              unsigned char **signature,
                              size_t *signature_len)
{
    int ret;
    unsigned char *sig;
    unsigned int sig_len;

    (void)hash_len;

    // 获取签名的长度
    sig_len = rsa->len;
    // 分配内存用于存储签名
    sig = LIBSSH2_ALLOC(session, sig_len);
    if(!sig) {
        return -1;
    }

    // 使用 RSA 私钥对哈希值进行签名
    ret = mbedtls_rsa_pkcs1_sign(rsa, NULL, NULL, MBEDTLS_RSA_PRIVATE,
                                 MBEDTLS_MD_SHA1, SHA_DIGEST_LENGTH,
                                 hash, sig);
    if(ret) {
        LIBSSH2_FREE(session, sig);
        return -1;
    }

    // 将签名和签名长度返回
    *signature = sig;
    *signature_len = sig_len;

    return (ret == 0) ? 0 : -1;
}

void
_libssh2_mbedtls_rsa_free(libssh2_rsa_ctx *ctx)
{
    // 释放 RSA 上下文和内存
    mbedtls_rsa_free(ctx);
    mbedtls_free(ctx);
}

static unsigned char *
gen_publickey_from_rsa(LIBSSH2_SESSION *session,
                      mbedtls_rsa_context *rsa,
                      size_t *keylen)
{
    int            e_bytes, n_bytes;
    unsigned long  len;
    unsigned char *key;
    unsigned char *p;

    // 计算公钥指数 e 和模数 n 的字节长度
    e_bytes = mbedtls_mpi_size(&rsa->E);
    # 计算 RSA 公钥的 N 的字节大小
    n_bytes = mbedtls_mpi_size(&rsa->N);

    # 计算整个公钥的长度
    len = 4 + 7 + 4 + e_bytes + 4 + n_bytes;

    # 分配内存空间用于存储公钥
    key = LIBSSH2_ALLOC(session, len);
    if(!key) {
        return NULL;
    }

    # 处理公钥编码
    p = key;

    # 将密钥类型进行网络字节序转换并写入内存
    _libssh2_htonu32(p, 7);  /* Key type. */
    p += 4;
    # 将 "ssh-rsa" 写入内存
    memcpy(p, "ssh-rsa", 7);
    p += 7;

    # 将 e 的字节大小进行网络字节序转换并写入内存
    _libssh2_htonu32(p, e_bytes);
    p += 4;
    # 将 RSA 公钥的 e 写入内存
    mbedtls_mpi_write_binary(&rsa->E, p, e_bytes);

    # 将 N 的字节大小进行网络字节序转换并写入内存
    _libssh2_htonu32(p, n_bytes);
    p += 4;
    # 将 RSA 公钥的 N 写入内存
    mbedtls_mpi_write_binary(&rsa->N, p, n_bytes);

    # 计算公钥的长度并返回
    *keylen = (size_t)(p - key);
    return key;
static int
_libssh2_mbedtls_pub_priv_key(LIBSSH2_SESSION *session,
                               unsigned char **method,
                               size_t *method_len,
                               unsigned char **pubkeydata,
                               size_t *pubkeydata_len,
                               mbedtls_pk_context *pkey)
{
    unsigned char *key = NULL, *mth = NULL;  // 声明两个指向无符号字符的指针变量，并初始化为NULL
    size_t keylen = 0, mthlen = 0;  // 声明两个变量，用于存储长度，并初始化为0
    int ret;  // 声明一个整型变量

    if(mbedtls_pk_get_type(pkey) != MBEDTLS_PK_RSA) {  // 判断pkey的类型是否为MBEDTLS_PK_RSA
        mbedtls_pk_free(pkey);  // 释放pkey占用的内存
        return _libssh2_error(session, LIBSSH2_ERROR_FILE,  // 返回错误信息
                              "Key type not supported");
    }

    /* write method */  // 写入方法
    mthlen = 7;  // 设置mthlen的值为7
    mth = LIBSSH2_ALLOC(session, mthlen);  // 分配mthlen长度的内存空间
    if(mth) {  // 如果分配成功
        memcpy(mth, "ssh-rsa", mthlen);  // 将"ssh-rsa"复制到mth中
    }
    else {  // 如果分配失败
        ret = -1;  // 设置ret为-1
    }

    rsa = mbedtls_pk_rsa(*pkey);  // 获取pkey中的RSA密钥
    key = gen_publickey_from_rsa(session, rsa, &keylen);  // 生成RSA密钥对应的公钥
    if(key == NULL) {  // 如果公钥为空
        ret = -1;  // 设置ret为-1
    }

    /* write output */  // 写入输出
    if(ret) {  // 如果ret不为0
        if(mth)
            LIBSSH2_FREE(session, mth);  // 释放mth占用的内存
        if(key)
            LIBSSH2_FREE(session, key);  // 释放key占用的内存
    }
    else {  // 如果ret为0
        *method = mth;  // 将mth的值赋给method
        *method_len = mthlen;  // 将mthlen的值赋给method_len
        *pubkeydata = key;  // 将key的值赋给pubkeydata
        *pubkeydata_len = keylen;  // 将keylen的值赋给pubkeydata_len
    }

    return ret;  // 返回ret的值
}

int
_libssh2_mbedtls_pub_priv_keyfile(LIBSSH2_SESSION *session,
                                 unsigned char **method,
                                 size_t *method_len,
                                 unsigned char **pubkeydata,
                                 size_t *pubkeydata_len,
                                 const char *privatekey,
                                 const char *passphrase)
{
    mbedtls_pk_context pkey;  // 声明一个mbedtls_pk_context类型的变量pkey
    char buf[1024];  // 声明一个长度为1024的字符数组buf
    int ret;  // 声明一个整型变量ret

    mbedtls_pk_init(&pkey);  // 初始化pkey
    ret = mbedtls_pk_parse_keyfile(&pkey, privatekey, passphrase);  // 解析私钥文件，将结果存储在pkey中
    # 如果返回值不等于0，表示出现错误
    if(ret != 0) {
        # 将错误码转换为字符串描述，并存储在buf中
        mbedtls_strerror(ret, (char *)buf, sizeof(buf));
        # 释放pkey占用的内存
        mbedtls_pk_free(&pkey);
        # 返回文件相关的libssh2错误码和描述
        return _libssh2_error(session, LIBSSH2_ERROR_FILE, buf);
    }

    # 调用_libssh2_mbedtls_pub_priv_key函数，生成公私钥对
    ret = _libssh2_mbedtls_pub_priv_key(session, method, method_len,
                                       pubkeydata, pubkeydata_len, &pkey);

    # 释放pkey占用的内存
    mbedtls_pk_free(&pkey);

    # 返回生成公私钥对的结果
    return ret;
}

int
_libssh2_mbedtls_pub_priv_keyfilememory(LIBSSH2_SESSION *session,
                                       unsigned char **method,
                                       size_t *method_len,
                                       unsigned char **pubkeydata,
                                       size_t *pubkeydata_len,
                                       const char *privatekeydata,
                                       size_t privatekeydata_len,
                                       const char *passphrase)
{
    mbedtls_pk_context pkey; // 定义 mbedtls 的公私钥上下文
    char buf[1024]; // 定义一个长度为 1024 的字符数组
    int ret; // 定义一个整型变量用于存储返回值
    void *privatekeydata_nullterm; // 定义一个指针变量
    size_t pwd_len; // 定义一个用于存储密码长度的变量

    /*
    mbedtls checks in "mbedtls/pkparse.c:1184" if "key[keylen - 1] != '\0'"
    private-key from memory will fail if the last byte is not a null byte
    */
    privatekeydata_nullterm = mbedtls_calloc(privatekeydata_len + 1, 1); // 分配内存空间
    if(privatekeydata_nullterm == NULL) { // 检查内存分配是否成功
        return -1; // 内存分配失败，返回错误码
    }
    memcpy(privatekeydata_nullterm, privatekeydata, privatekeydata_len); // 将私钥数据复制到新分配的内存空间

    mbedtls_pk_init(&pkey); // 初始化公私钥上下文

    pwd_len = passphrase != NULL ? strlen((const char *)passphrase) : 0; // 计算密码长度
    ret = mbedtls_pk_parse_key(&pkey,
                               (unsigned char *)privatekeydata_nullterm,
                               privatekeydata_len + 1,
                               (const unsigned char *)passphrase, pwd_len); // 解析私钥
    _libssh2_mbedtls_safe_free(privatekeydata_nullterm, privatekeydata_len); // 释放内存空间

    if(ret != 0) { // 检查解析私钥是否成功
        mbedtls_strerror(ret, (char *)buf, sizeof(buf)); // 获取错误信息
        mbedtls_pk_free(&pkey); // 释放公私钥上下文
        return _libssh2_error(session, LIBSSH2_ERROR_FILE, buf); // 返回错误码
    }

    ret = _libssh2_mbedtls_pub_priv_key(session, method, method_len,
                                       pubkeydata, pubkeydata_len, &pkey); // 调用函数处理公私钥

    mbedtls_pk_free(&pkey); // 释放公私钥上下文

    return ret; // 返回结果
}

void _libssh2_init_aes_ctr(void)
{
    /* no implementation */
}


/*******************************************************************/
/*
 * mbedTLS backend: Diffie-Hellman functions
 */

void
_libssh2_dh_init(_libssh2_dh_ctx *dhctx)
{
    *dhctx = _libssh2_mbedtls_bignum_init();    /* 从客户端获取随机数 */
}

int
_libssh2_dh_key_pair(_libssh2_dh_ctx *dhctx, _libssh2_bn *public,
                     _libssh2_bn *g, _libssh2_bn *p, int group_order)
{
    /* 生成 x 和 e */
    _libssh2_mbedtls_bignum_random(*dhctx, group_order * 8 - 1, 0, -1);
    mbedtls_mpi_exp_mod(public, g, *dhctx, p, NULL);
    return 0;
}

int
_libssh2_dh_secret(_libssh2_dh_ctx *dhctx, _libssh2_bn *secret,
                   _libssh2_bn *f, _libssh2_bn *p)
{
    /* 计算共享密钥 */
    mbedtls_mpi_exp_mod(secret, f, *dhctx, p, NULL);
    return 0;
}

void
_libssh2_dh_dtor(_libssh2_dh_ctx *dhctx)
{
    _libssh2_mbedtls_bignum_free(*dhctx);
    *dhctx = NULL;
}

#if LIBSSH2_ECDSA

/*******************************************************************/
/*
 * mbedTLS backend: ECDSA functions
 */

/*
 * _libssh2_ecdsa_create_key
 *
 * Creates a local private key based on input curve
 * and returns octal value and octal length
 *
 */

int
_libssh2_mbedtls_ecdsa_create_key(LIBSSH2_SESSION *session,
                                  _libssh2_ec_key **privkey,
                                  unsigned char **pubkey_oct,
                                  size_t *pubkey_oct_len,
                                  libssh2_curve_type curve)
{
    size_t plen = 0;

    *privkey = LIBSSH2_ALLOC(session, sizeof(mbedtls_ecp_keypair));

    if(*privkey == NULL)
        goto failed;

    mbedtls_ecdsa_init(*privkey);

    if(mbedtls_ecdsa_genkey(*privkey, (mbedtls_ecp_group_id)curve,
                            mbedtls_ctr_drbg_random,
                            &_libssh2_mbedtls_ctr_drbg) != 0)
        goto failed;

    plen = 2 * mbedtls_mpi_size(&(*privkey)->grp.P) + 1;
    *pubkey_oct = LIBSSH2_ALLOC(session, plen);

    if(*pubkey_oct == NULL)
        goto failed;
    # 如果将椭圆曲线私钥的公钥以二进制格式写入到指定的缓冲区中，并且写入成功
    if(mbedtls_ecp_point_write_binary(&(*privkey)->grp, &(*privkey)->Q,
                                      MBEDTLS_ECP_PF_UNCOMPRESSED,
                                      pubkey_oct_len, *pubkey_oct, plen) == 0)
        # 返回成功标志
        return 0;
    # 释放 ECDSA 私钥资源
    _libssh2_mbedtls_ecdsa_free(*privkey);
    # 释放 ECDSA 公钥资源
    _libssh2_mbedtls_safe_free(*pubkey_oct, plen);
    # 将私钥指针置空
    *privkey = NULL;

    # 返回错误代码
    return -1;
}

/* _libssh2_ecdsa_curve_name_with_octal_new
 *
 * 根据八进制字符串、长度和类型创建新的公钥
 *
 */

int
_libssh2_mbedtls_ecdsa_curve_name_with_octal_new(libssh2_ecdsa_ctx **ctx,
                                                 const unsigned char *k,
                                                 size_t k_len,
                                                 libssh2_curve_type curve)
{
    # 为 ECDSA 上下文分配内存
    *ctx = mbedtls_calloc(1, sizeof(mbedtls_ecp_keypair));

    # 如果分配内存失败，则跳转到失败标签
    if(*ctx == NULL)
        goto failed;

    # 初始化 ECDSA 上下文
    mbedtls_ecdsa_init(*ctx);

    # 加载指定曲线类型的椭圆曲线参数
    if(mbedtls_ecp_group_load(&(*ctx)->grp, (mbedtls_ecp_group_id)curve) != 0)
        goto failed;

    # 从八进制字符串读取公钥点
    if(mbedtls_ecp_point_read_binary(&(*ctx)->grp, &(*ctx)->Q, k, k_len) != 0)
        goto failed;

    # 检查公钥是否有效，如果有效则返回成功
    if(mbedtls_ecp_check_pubkey(&(*ctx)->grp, &(*ctx)->Q) == 0)
        return 0;

failed:

    # 释放 ECDSA 上下文资源
    _libssh2_mbedtls_ecdsa_free(*ctx);
    # 将上下文指针置空
    *ctx = NULL;

    # 返回错误代码
    return -1;
}

/* _libssh2_ecdh_gen_k
 *
 * 根据本地私钥、远程公钥和长度计算共享密钥 K
 */

int
_libssh2_mbedtls_ecdh_gen_k(_libssh2_bn **k,
                            _libssh2_ec_key *privkey,
                            const unsigned char *server_pubkey,
                            size_t server_pubkey_len)
{
    mbedtls_ecp_point pubkey;
    int rc = 0;

    # 如果共享密钥指针为空，则返回错误代码
    if(*k == NULL)
        return -1;

    # 初始化公钥点
    mbedtls_ecp_point_init(&pubkey);

    # 从远程公钥数据中读取公钥点
    if(mbedtls_ecp_point_read_binary(&privkey->grp, &pubkey,
                                     server_pubkey, server_pubkey_len) != 0) {
        rc = -1;
        goto cleanup;
    }

    # 计算共享密钥 K
    if(mbedtls_ecdh_compute_shared(&privkey->grp, *k,
                                   &pubkey, &privkey->d,
                                   mbedtls_ctr_drbg_random,
                                   &_libssh2_mbedtls_ctr_drbg) != 0) {
        rc = -1;
        goto cleanup;
    # 如果私钥不符合椭圆曲线的要求
    if(mbedtls_ecp_check_privkey(&privkey->grp, *k) != 0)
        # 设置返回值为-1
        rc = -1;
cleanup:
    # 释放公钥的内存
    mbedtls_ecp_point_free(&pubkey);
    # 返回结果
    return rc;
}

# 定义宏，用于验证 ECDSA 签名
#define LIBSSH2_MBEDTLS_ECDSA_VERIFY(digest_type)                   \
{                                                                   \
    unsigned char hsh[SHA##digest_type##_DIGEST_LENGTH];            \
    # 计算消息的哈希值
    if(libssh2_sha##digest_type(m, m_len, hsh) == 0) {              \
        # 使用 mbedtls_ecdsa_verify 函数验证 ECDSA 签名
        rc = mbedtls_ecdsa_verify(&ctx->grp, hsh,                   \
                                  SHA##digest_type##_DIGEST_LENGTH, \
                                  &ctx->Q, &pr, &ps);               \
    }                                                               \
}

/* _libssh2_ecdsa_sign
 *
 * Verifies the ECDSA signature of a hashed message
 *
 */

# 验证 ECDSA 签名的函数
int
_libssh2_mbedtls_ecdsa_verify(libssh2_ecdsa_ctx *ctx,
                              const unsigned char *r, size_t r_len,
                              const unsigned char *s, size_t s_len,
                              const unsigned char *m, size_t m_len)
{
    mbedtls_mpi pr, ps;
    int rc = -1;
    # 初始化 pr 和 ps
    mbedtls_mpi_init(&pr);
    mbedtls_mpi_init(&ps);
    # 将 r 和 s 转换为 mbedtls_mpi 类型
    if(mbedtls_mpi_read_binary(&pr, r, r_len) != 0)
        goto cleanup;
    if(mbedtls_mpi_read_binary(&ps, s, s_len) != 0)
        goto cleanup;
    # 根据椭圆曲线类型选择相应的验证函数
    switch(_libssh2_ecdsa_get_curve_type(ctx)) {
    case LIBSSH2_EC_CURVE_NISTP256:
        LIBSSH2_MBEDTLS_ECDSA_VERIFY(256);
        break;
    case LIBSSH2_EC_CURVE_NISTP384:
        LIBSSH2_MBEDTLS_ECDSA_VERIFY(384);
        break;
    case LIBSSH2_EC_CURVE_NISTP521:
        LIBSSH2_MBEDTLS_ECDSA_VERIFY(512);
        break;
    default:
        rc = -1;
    }

cleanup:
    # 释放 pr 和 ps 的内存
    mbedtls_mpi_free(&pr);
    mbedtls_mpi_free(&ps);
    # 返回验证结果
    return (rc == 0) ? 0 : -1;
}

static int
# 解析 mbedtls 的 EC 密钥，将结果存储在 ctx 中
_libssh2_mbedtls_parse_eckey(libssh2_ecdsa_ctx **ctx,
                             mbedtls_pk_context *pkey,
                             LIBSSH2_SESSION *session,
                             const unsigned char *data,
                             size_t data_len,
                             const unsigned char *pwd)
{
    size_t pwd_len;

    # 如果密码存在，计算密码长度，否则密码长度为 0
    pwd_len = pwd ? strlen((const char *) pwd) : 0;

    # 解析密钥数据，使用密码进行解密
    if(mbedtls_pk_parse_key(pkey, data, data_len, pwd, pwd_len) != 0)
        goto failed;

    # 检查解析出的密钥类型是否为 EC 密钥
    if(mbedtls_pk_get_type(pkey) != MBEDTLS_PK_ECKEY)
        goto failed;

    # 为 ECDSA 上下文分配内存
    *ctx = LIBSSH2_ALLOC(session, sizeof(libssh2_ecdsa_ctx));

    # 如果内存分配失败，则跳转到失败处理
    if(*ctx == NULL)
        goto failed;

    # 初始化 ECDSA 上下文
    mbedtls_ecdsa_init(*ctx);

    # 从密钥对中提取 ECDSA 密钥
    if(mbedtls_ecdsa_from_keypair(*ctx, mbedtls_pk_ec(*pkey)) == 0)
        return 0;

failed:

    # 释放 ECDSA 上下文的内存
    _libssh2_mbedtls_ecdsa_free(*ctx);
    *ctx = NULL;

    return -1;
}

# 解析 OpenSSH 密钥，将结果存储在 ctx 中
static int
_libssh2_mbedtls_parse_openssh_key(libssh2_ecdsa_ctx **ctx,
                                   LIBSSH2_SESSION *session,
                                   const unsigned char *data,
                                   size_t data_len,
                                   const unsigned char *pwd)
{
    libssh2_curve_type type;
    unsigned char *name = NULL;
    struct string_buf *decrypted = NULL;
    size_t curvelen, exponentlen, pointlen;
    unsigned char *curve, *exponent, *point_buf;

    # 使用 OpenSSH PEM 格式解析密钥数据
    if(_libssh2_openssh_pem_parse_memory(session, pwd,
                                         (const char *)data, data_len,
                                         &decrypted) != 0)
        goto failed;

    # 获取密钥名称
    if(_libssh2_get_string(decrypted, &name, NULL) != 0)
        goto failed;

    # 根据密钥名称获取曲线类型
    if(_libssh2_mbedtls_ecdsa_curve_type_from_name((const char *)name,
                                                   &type) != 0)
        goto failed;

    # 获取曲线参数
    if(_libssh2_get_string(decrypted, &curve, &curvelen) != 0)
        goto failed;

    # 获取点参数
    if(_libssh2_get_string(decrypted, &point_buf, &pointlen) != 0)
        goto failed;
    # 如果获取解密后的数据的指数和长度不为0，则跳转到失败标签
    if(_libssh2_get_bignum_bytes(decrypted, &exponent, &exponentlen) != 0)
        goto failed;

    # 为 ECDSA 上下文分配内存
    *ctx = LIBSSH2_ALLOC(session, sizeof(libssh2_ecdsa_ctx));

    # 如果分配内存失败，则跳转到失败标签
    if(*ctx == NULL)
        goto failed;

    # 初始化 mbedtls 的 ECDSA 上下文
    mbedtls_ecdsa_init(*ctx);

    # 根据给定的类型加载 ECDSA 曲线
    if(mbedtls_ecp_group_load(&(*ctx)->grp, (mbedtls_ecp_group_id)type) != 0)
        goto failed;

    # 从二进制数据中读取私钥
    if(mbedtls_mpi_read_binary(&(*ctx)->d, exponent, exponentlen) != 0)
        goto failed;

    # 使用私钥和基点生成公钥
    if(mbedtls_ecp_mul(&(*ctx)->grp, &(*ctx)->Q,
                       &(*ctx)->d, &(*ctx)->grp.G,
                       mbedtls_ctr_drbg_random,
                       &_libssh2_mbedtls_ctr_drbg) != 0)
        goto failed;

    # 检查生成的私钥是否有效，如果有效则跳转到清理标签
    if(mbedtls_ecp_check_privkey(&(*ctx)->grp, &(*ctx)->d) == 0)
        goto cleanup;
# 释放 mbedtls_ecdsa_ctx 结构体指针所指向的内存，并将指针置为 NULL
_libssh2_mbedtls_ecdsa_free(*ctx);
*ctx = NULL;

# 清理工作，释放资源
cleanup:

# 如果 decrypted 不为空，则释放其所占用的内存
if(decrypted) {
    _libssh2_string_buf_free(session, decrypted);
}

# 如果 ctx 为 NULL，则返回 -1，否则返回 0
return (*ctx == NULL) ? -1 : 0;
}

/* _libssh2_ecdsa_new_private
 *
 * 根据文件路径和密码创建一个新的私钥
 *
 */

int
_libssh2_mbedtls_ecdsa_new_private(libssh2_ecdsa_ctx **ctx,
                                   LIBSSH2_SESSION *session,
                                   const char *filename,
                                   const unsigned char *pwd)
{
    mbedtls_pk_context pkey;
    unsigned char *data;
    size_t data_len;

    # 如果无法从文件中加载私钥数据，则跳转到 cleanup 标签处
    if(mbedtls_pk_load_file(filename, &data, &data_len) != 0)
        goto cleanup;

    # 初始化 mbedtls_pk_context 结构体
    mbedtls_pk_init(&pkey);

    # 如果成功解析椭圆曲线密钥，则跳转到 cleanup 标签处
    if(_libssh2_mbedtls_parse_eckey(ctx, &pkey, session,
                                    data, data_len, pwd) == 0)
        goto cleanup;

    # 解析 OpenSSH 格式的密钥
    _libssh2_mbedtls_parse_openssh_key(ctx, session, data, data_len, pwd);

cleanup:

    # 释放 mbedtls_pk_context 结构体所占用的资源
    mbedtls_pk_free(&pkey);

    # 安全释放 data 所占用的内存
    _libssh2_mbedtls_safe_free(data, data_len);

    # 如果 ctx 为 NULL，则返回 -1，否则返回 0
    return (*ctx == NULL) ? -1 : 0;
}

/* _libssh2_ecdsa_new_private
 *
 * 根据文件数据和密码创建一个新的私钥
 *
 */

int
_libssh2_mbedtls_ecdsa_new_private_frommemory(libssh2_ecdsa_ctx **ctx,
                                              LIBSSH2_SESSION *session,
                                              const char *data,
                                              size_t data_len,
                                              const unsigned char *pwd)
{
    unsigned char *ntdata;
    mbedtls_pk_context pkey;

    # 初始化 mbedtls_pk_context 结构体
    mbedtls_pk_init(&pkey);

    # 分配内存用于存储文件数据
    ntdata = LIBSSH2_ALLOC(session, data_len + 1);

    # 如果内存分配失败，则跳转到 cleanup 标签处
    if(ntdata == NULL)
        goto cleanup;

    # 将文件数据复制到新分配的内存中
    memcpy(ntdata, data, data_len);

    # 如果成功解析椭圆曲线密钥，则跳转到 cleanup 标签处
    if(_libssh2_mbedtls_parse_eckey(ctx, &pkey, session,
                                    ntdata, data_len + 1, pwd) == 0)
        goto cleanup;
    # 使用libssh2_mbedtls_parse_openssh_key函数解析OpenSSH格式的密钥
    # ctx: libssh2会话上下文
    # session: libssh2会话
    # ntdata: 密钥数据
    # data_len + 1: 数据长度加1
    # pwd: 密码
    _libssh2_mbedtls_parse_openssh_key(ctx, session, ntdata, data_len + 1, pwd);
# 释放 ECDSA 密钥
cleanup:
    mbedtls_pk_free(&pkey);
    # 安全释放内存
    _libssh2_mbedtls_safe_free(ntdata, data_len);
    # 如果上下文为空，则返回 -1，否则返回 0
    return (*ctx == NULL) ? -1 : 0;
}

# 将 MPI 写入二进制
static unsigned char *
_libssh2_mbedtls_mpi_write_binary(unsigned char *buf,
                                  const mbedtls_mpi *mpi,
                                  size_t bytes)
{
    unsigned char *p = buf;
    # 如果 p 的大小小于 4，则跳转到 done 标签
    if(sizeof(&p) / sizeof(p[0]) < 4) {
        goto done;
    }
    # p 向后移动 4 个位置
    p += 4;
    *p = 0;
    # 如果字节数大于 0，则将 MPI 写入二进制
    if(bytes > 0) {
        mbedtls_mpi_write_binary(mpi, p + 1, bytes - 1);
    }
    # 如果字节数大于 0 并且 p+1 不是 0x80，则移动内存
    if(bytes > 0 && !(*(p + 1) & 0x80)) {
        memmove(p, p + 1, --bytes);
    }
    # 将 p-4 处的 4 字节整数转换为网络字节序
    _libssh2_htonu32(p - 4, bytes);

done:
    # 返回 p+bytes
    return p + bytes;
}

/* _libssh2_ecdsa_sign
 *
 * 计算先前哈希消息的 ECDSA 签名
 *
 */

# 计算 ECDSA 签名
int
_libssh2_mbedtls_ecdsa_sign(LIBSSH2_SESSION *session,
                            libssh2_ecdsa_ctx *ctx,
                            const unsigned char *hash,
                            unsigned long hash_len,
                            unsigned char **sign,
                            size_t *sign_len)
{
    size_t r_len, s_len, tmp_sign_len = 0;
    unsigned char *sp, *tmp_sign = NULL;
    mbedtls_mpi pr, ps;

    mbedtls_mpi_init(&pr);
    mbedtls_mpi_init(&ps);
    # 如果 mbedtls_ecdsa_sign 返回非 0，则跳转到 cleanup 标签
    if(mbedtls_ecdsa_sign(&ctx->grp, &pr, &ps, &ctx->d,
                          hash, hash_len,
                          mbedtls_ctr_drbg_random,
                          &_libssh2_mbedtls_ctr_drbg) != 0)
        goto cleanup;
    # 计算 r 和 s 的长度
    r_len = mbedtls_mpi_size(&pr) + 1;
    s_len = mbedtls_mpi_size(&ps) + 1;
    tmp_sign_len = r_len + s_len + 8;
    # 分配临时签名内存
    tmp_sign = LIBSSH2_CALLOC(session, tmp_sign_len);
    # 如果分配失败，则跳转到 cleanup 标签
    if(tmp_sign == NULL)
        goto cleanup;
    # 设置 sp 指针，并将 pr 和 ps 写入二进制
    sp = tmp_sign;
    sp = _libssh2_mbedtls_mpi_write_binary(sp, &pr, r_len);
    sp = _libssh2_mbedtls_mpi_write_binary(sp, &ps, s_len);
    # 计算签名长度
    *sign_len = (size_t)(sp - tmp_sign);
    # 分配签名内存
    *sign = LIBSSH2_CALLOC(session, *sign_len);
    # 如果分配失败，则跳转到 cleanup 标签
    if(*sign == NULL)
        goto cleanup;
    # 使用 memcpy 函数将 tmp_sign 指向的内容复制到 sign 指向的位置，复制长度为 sign_len 指向的值
    memcpy(*sign, tmp_sign, *sign_len);
cleanup:

    // 释放 pr 变量所占用的内存
    mbedtls_mpi_free(&pr);
    // 释放 ps 变量所占用的内存
    mbedtls_mpi_free(&ps);

    // 安全释放 tmp_sign 变量所占用的内存
    _libssh2_mbedtls_safe_free(tmp_sign, tmp_sign_len);

    // 如果 sign 指向 NULL，则返回 -1，否则返回 0
    return (*sign == NULL) ? -1 : 0;
}

/* _libssh2_ecdsa_get_curve_type
 *
 * 返回与 libssh2_curve_type 相对应的密钥曲线类型
 *
 */

libssh2_curve_type
_libssh2_mbedtls_ecdsa_get_curve_type(libssh2_ecdsa_ctx *ctx)
{
    // 返回与 libssh2_curve_type 相对应的密钥曲线类型
    return (libssh2_curve_type) ctx->grp.id;
}

/* _libssh2_ecdsa_curve_type_from_name
 *
 * 返回成功时返回 0，否则返回与 libssh2_curve_type 相对应的密钥曲线类型
 *
 */

int
_libssh2_mbedtls_ecdsa_curve_type_from_name(const char *name,
                                            libssh2_curve_type *out_type)
{
    int ret = 0;
    libssh2_curve_type type;

    // 如果 name 为 NULL 或者长度不等于 19，则返回 -1
    if(name == NULL || strlen(name) != 19)
        return -1;

    // 根据 name 的值设置 type 的值
    if(strcmp(name, "ecdsa-sha2-nistp256") == 0)
        type = LIBSSH2_EC_CURVE_NISTP256;
    else if(strcmp(name, "ecdsa-sha2-nistp384") == 0)
        type = LIBSSH2_EC_CURVE_NISTP384;
    else if(strcmp(name, "ecdsa-sha2-nistp521") == 0)
        type = LIBSSH2_EC_CURVE_NISTP521;
    else {
        ret = -1;
    }

    // 如果 ret 为 0 并且 out_type 不为 NULL，则将 type 的值赋给 out_type
    if(ret == 0 && out_type) {
        *out_type = type;
    }

    // 返回 ret 的值
    return ret;
}

// 释放 libssh2_ecdsa_ctx 对象所占用的内存
void
_libssh2_mbedtls_ecdsa_free(libssh2_ecdsa_ctx *ctx)
{
    mbedtls_ecdsa_free(ctx);
    mbedtls_free(ctx);
}

#endif /* LIBSSH2_ECDSA */
#endif /* LIBSSH2_MBEDTLS */
```