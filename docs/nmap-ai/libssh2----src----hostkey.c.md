# `nmap\libssh2\src\hostkey.c`

```
/* 版权声明和许可协议，保留所有权利 */
/* 在源代码和二进制形式下的重新分发和修改都是允许的，但需要满足以下条件：
 *   1. 源代码的重新分发必须保留上述版权声明、条件列表和下面的免责声明
 *   2. 二进制形式的重新分发必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和下面的免责声明
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或其他贡献者的名称来认可或推广从本软件派生的产品
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途的暗示担保，无论在任何情况下，版权所有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害的可能性
 */

#include "libssh2_priv.h"
#include "misc.h"

/* 在某些平台上需要 struct iovec */
#ifdef HAVE_SYS_UIO_H
#include <sys/uio.h>
#endif

#if LIBSSH2_RSA
/* ***********
 * ssh-rsa *
 *********** */
# 定义一个静态函数，用于释放 SSH RSA 方法的资源
static int hostkey_method_ssh_rsa_dtor(LIBSSH2_SESSION * session,
                                       void **abstract);

# 初始化服务器主机密钥工作区，使用 e/n 对
static int
hostkey_method_ssh_rsa_init(LIBSSH2_SESSION * session,
                            const unsigned char *hostkey_data,
                            size_t hostkey_data_len,
                            void **abstract)
{
    libssh2_rsa_ctx *rsactx;  # 声明 RSA 上下文
    unsigned char *e, *n;  # 声明 e 和 n
    size_t e_len, n_len;  # 声明 e 和 n 的长度
    struct string_buf buf;  # 声明字符串缓冲区

    if(*abstract) {  # 如果抽象对象已经存在
        hostkey_method_ssh_rsa_dtor(session, abstract);  # 调用释放资源的函数
        *abstract = NULL;  # 将抽象对象置为空
    }

    if(hostkey_data_len < 19) {  # 如果主机密钥数据长度小于19
        _libssh2_debug(session, LIBSSH2_TRACE_ERROR,  # 输出错误信息
                       "host key length too short");
        return -1;  # 返回错误代码
    }

    buf.data = (unsigned char *)hostkey_data;  # 将主机密钥数据转换为无符号字符指针
    buf.dataptr = buf.data;  # 将数据指针指向数据起始位置
    buf.len = hostkey_data_len;  # 设置缓冲区长度为主机密钥数据长度

    if(_libssh2_match_string(&buf, "ssh-rsa"))  # 如果匹配字符串 "ssh-rsa"
        return -1;  # 返回错误代码

    if(_libssh2_get_string(&buf, &e, &e_len))  # 获取字符串 e
        return -1;  # 返回错误代码

    if(_libssh2_get_string(&buf, &n, &n_len))  # 获取字符串 n
        return -1;  # 返回错误代码

    if(_libssh2_rsa_new(&rsactx, e, e_len, n, n_len, NULL, 0,  # 创建新的 RSA 上下文
                        NULL, 0, NULL, 0, NULL, 0, NULL, 0, NULL, 0)) {
        return -1;  # 返回错误代码
    }

    *abstract = rsactx;  # 将 RSA 上下文赋给抽象对象

    return 0;  # 返回成功代码
}

# 从 PEM 文件加载私钥
static int
hostkey_method_ssh_rsa_initPEM(LIBSSH2_SESSION * session,
                               const char *privkeyfile,
                               unsigned const char *passphrase,
                               void **abstract)
{
    libssh2_rsa_ctx *rsactx;  # 声明 RSA 上下文
    int ret;  # 声明返回值

    if(*abstract) {  # 如果抽象对象已经存在
        hostkey_method_ssh_rsa_dtor(session, abstract);  # 调用释放资源的函数
        *abstract = NULL;  # 将抽象对象置为空
    }

    ret = _libssh2_rsa_new_private(&rsactx, session, privkeyfile, passphrase);  # 使用私钥文件和口令创建新的 RSA 上下文
    if(ret) {  # 如果返回值不为0
        return -1;  # 返回错误代码
    }

    *abstract = rsactx;  # 将 RSA 上下文赋给抽象对象

    return 0;  # 返回成功代码
}
/*
 * hostkey_method_ssh_rsa_initPEMFromMemory
 *
 * 从内存中加载私钥
 */
static int
hostkey_method_ssh_rsa_initPEMFromMemory(LIBSSH2_SESSION * session,
                                         const char *privkeyfiledata,
                                         size_t privkeyfiledata_len,
                                         unsigned const char *passphrase,
                                         void **abstract)
{
    libssh2_rsa_ctx *rsactx;
    int ret;

    if(*abstract) {
        // 如果抽象对象已经存在，则调用析构函数进行清理
        hostkey_method_ssh_rsa_dtor(session, abstract);
        *abstract = NULL;
    }

    // 从内存中加载私钥
    ret = _libssh2_rsa_new_private_frommemory(&rsactx, session,
                                              privkeyfiledata,
                                              privkeyfiledata_len, passphrase);
    if(ret) {
        return -1;
    }

    *abstract = rsactx;

    return 0;
}

/*
 * hostkey_method_ssh_rsa_sig_verify
 *
 * 验证远程创建的签名
 */
static int
hostkey_method_ssh_rsa_sig_verify(LIBSSH2_SESSION * session,
                                  const unsigned char *sig,
                                  size_t sig_len,
                                  const unsigned char *m,
                                  size_t m_len, void **abstract)
{
    libssh2_rsa_ctx *rsactx = (libssh2_rsa_ctx *) (*abstract);
    (void) session;

    /* 跳过 keyname_len(4) + keyname(7){"ssh-rsa"} + signature_len(4) */
    if(sig_len < 15)
        return -1;

    sig += 15;
    sig_len -= 15;
    return _libssh2_rsa_sha1_verify(rsactx, sig, sig_len, m, m_len);
}

/*
 * hostkey_method_ssh_rsa_signv
 *
 * 从向量数组构造签名
 */
static int
hostkey_method_ssh_rsa_signv(LIBSSH2_SESSION * session,
                             unsigned char **signature,
                             size_t *signature_len,
                             int veccount,
                             const struct iovec datavec[],
                             void **abstract)
{
    // 声明一个指向libssh2_rsa_ctx类型的指针rsactx，并将abstract指针所指向的内容转换为libssh2_rsa_ctx类型
    libssh2_rsa_ctx *rsactx = (libssh2_rsa_ctx *) (*abstract);
#ifdef _libssh2_rsa_sha1_signv
    // 如果定义了_libssh2_rsa_sha1_signv，则调用该函数进行签名
    return _libssh2_rsa_sha1_signv(session, signature, signature_len,
                                   veccount, datavec, rsactx);
#else
    // 否则，使用 SHA-1 算法对数据进行哈希计算
    int ret;
    int i;
    unsigned char hash[SHA_DIGEST_LENGTH];
    libssh2_sha1_ctx ctx;

    libssh2_sha1_init(&ctx);
    for(i = 0; i < veccount; i++) {
        libssh2_sha1_update(ctx, datavec[i].iov_base, datavec[i].iov_len);
    }
    libssh2_sha1_final(ctx, hash);

    // 调用 _libssh2_rsa_sha1_sign 函数进行签名
    ret = _libssh2_rsa_sha1_sign(session, rsactx, hash, SHA_DIGEST_LENGTH,
                                 signature, signature_len);
    // 如果签名失败，则返回 -1
    if(ret) {
        return -1;
    }

    // 签名成功，返回 0
    return 0;
#endif
}

/*
 * hostkey_method_ssh_rsa_dtor
 *
 * 关闭 RSA 主机密钥
 */
static int
hostkey_method_ssh_rsa_dtor(LIBSSH2_SESSION * session, void **abstract)
{
    libssh2_rsa_ctx *rsactx = (libssh2_rsa_ctx *) (*abstract);
    (void) session;

    // 释放 RSA 上下文
    _libssh2_rsa_free(rsactx);

    // 将抽象指针置为空
    *abstract = NULL;

    return 0;
}

#ifdef OPENSSL_NO_MD5
#define MD5_DIGEST_LENGTH 16
#endif

// 定义 SSH RSA 主机密钥方法
static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ssh_rsa = {
    "ssh-rsa",
    MD5_DIGEST_LENGTH,
    hostkey_method_ssh_rsa_init,
    hostkey_method_ssh_rsa_initPEM,
    hostkey_method_ssh_rsa_initPEMFromMemory,
    hostkey_method_ssh_rsa_sig_verify,
    hostkey_method_ssh_rsa_signv,
    NULL,                       /* encrypt */
    hostkey_method_ssh_rsa_dtor,
};
#endif /* LIBSSH2_RSA */

#if LIBSSH2_DSA
/* ***********
 * ssh-dss *
 *********** */

// 关闭 DSS 主机密钥
static int hostkey_method_ssh_dss_dtor(LIBSSH2_SESSION * session,
                                       void **abstract);

/*
 * hostkey_method_ssh_dss_init
 *
 * 使用给定的 p/q/g/y 数据初始化 DSS 主机密钥
 */
static int
hostkey_method_ssh_dss_init(LIBSSH2_SESSION * session,
                            const unsigned char *hostkey_data,
                            size_t hostkey_data_len,
                            void **abstract)
{
    libssh2_dsa_ctx *dsactx;
    unsigned char *p, *q, *g, *y;
    # 定义变量 p_len, q_len, g_len, y_len，用于存储后续获取的字符串长度
    size_t p_len, q_len, g_len, y_len;
    # 定义结构体变量 buf，用于存储字符串数据
    struct string_buf buf;

    # 如果抽象指针不为空，则调用 hostkey_method_ssh_dss_dtor 函数释放资源，并将抽象指针置为空
    if(*abstract) {
        hostkey_method_ssh_dss_dtor(session, abstract);
        *abstract = NULL;
    }

    # 如果主机密钥数据长度小于 27，则记录错误信息并返回 -1
    if(hostkey_data_len < 27) {
        _libssh2_debug(session, LIBSSH2_TRACE_ERROR,
                       "host key length too short");
        return -1;
    }

    # 将主机密钥数据转换为无符号字符指针，并初始化 buf 结构体
    buf.data = (unsigned char *)hostkey_data;
    buf.dataptr = buf.data;
    buf.len = hostkey_data_len;

    # 如果主机密钥算法为 "ssh-dss"，则返回 -1
    if(_libssh2_match_string(&buf, "ssh-dss"))
        return -1;

    # 获取字符串数据 p，并记录其长度到 p_len
    if(_libssh2_get_string(&buf, &p, &p_len))
       return -1;

    # 获取字符串数据 q，并记录其长度到 q_len
    if(_libssh2_get_string(&buf, &q, &q_len))
        return -1;

    # 获取字符串数据 g，并记录其长度到 g_len
    if(_libssh2_get_string(&buf, &g, &g_len))
        return -1;

    # 获取字符串数据 y，并记录其长度到 y_len
    if(_libssh2_get_string(&buf, &y, &y_len))
        return -1;

    # 使用获取的 p、q、g、y 数据创建 DSA 上下文 dsactx
    if(_libssh2_dsa_new(&dsactx, p, p_len, q, q_len,
                        g, g_len, y, y_len, NULL, 0)) {
        return -1;
    }

    # 将创建的 DSA 上下文赋值给抽象指针
    *abstract = dsactx;

    # 返回成功标志 0
    return 0;
/*
 * hostkey_method_ssh_dss_initPEM
 *
 * 从 PEM 文件加载私钥
 */
static int
hostkey_method_ssh_dss_initPEM(LIBSSH2_SESSION * session,
                               const char *privkeyfile,
                               unsigned const char *passphrase,
                               void **abstract)
{
    libssh2_dsa_ctx *dsactx;
    int ret;

    if(*abstract) {
        // 如果抽象对象已经存在，则调用析构函数进行清理
        hostkey_method_ssh_dss_dtor(session, abstract);
        *abstract = NULL;
    }

    // 调用内部函数加载私钥
    ret = _libssh2_dsa_new_private(&dsactx, session, privkeyfile, passphrase);
    if(ret) {
        return -1;
    }

    // 将加载的私钥对象赋值给抽象对象
    *abstract = dsactx;

    return 0;
}

/*
 * hostkey_method_ssh_dss_initPEMFromMemory
 *
 * 从内存加载私钥
 */
static int
hostkey_method_ssh_dss_initPEMFromMemory(LIBSSH2_SESSION * session,
                                         const char *privkeyfiledata,
                                         size_t privkeyfiledata_len,
                                         unsigned const char *passphrase,
                                         void **abstract)
{
    libssh2_dsa_ctx *dsactx;
    int ret;

    if(*abstract) {
        // 如果抽象对象已经存在，则调用析构函数进行清理
        hostkey_method_ssh_dss_dtor(session, abstract);
        *abstract = NULL;
    }

    // 调用内部函数从内存加载私钥
    ret = _libssh2_dsa_new_private_frommemory(&dsactx, session,
                                              privkeyfiledata,
                                              privkeyfiledata_len, passphrase);
    if(ret) {
        return -1;
    }

    // 将加载的私钥对象赋值给抽象对象
    *abstract = dsactx;

    return 0;
}

/*
 * libssh2_hostkey_method_ssh_dss_sign
 *
 * 验证远程创建的签名
 */
static int
hostkey_method_ssh_dss_sig_verify(LIBSSH2_SESSION * session,
                                  const unsigned char *sig,
                                  size_t sig_len,
                                  const unsigned char *m,
                                  size_t m_len, void **abstract)
{
    // 将抽象对象转换为 DSA 上下文
    libssh2_dsa_ctx *dsactx = (libssh2_dsa_ctx *) (*abstract);
    # 跳过 keyname_len(4) + keyname(7){"ssh-dss"} + signature_len(4) 的长度，即跳过前15个字节
    if(sig_len != 55):  # 如果签名长度不等于55
        return _libssh2_error(session, LIBSSH2_ERROR_PROTO,  # 返回协议错误
                              "Invalid DSS signature length")  # 并提示 DSS 签名长度无效

    sig += 15  # 签名指针向后移动15个字节
    sig_len -= 15  # 签名长度减去15
    return _libssh2_dsa_sha1_verify(dsactx, sig, m, m_len)  # 返回 DSA SHA1 验证结果
/*
 * hostkey_method_ssh_dss_signv
 *
 * Construct a signature from an array of vectors
 */
static int
hostkey_method_ssh_dss_signv(LIBSSH2_SESSION * session,
                             unsigned char **signature,
                             size_t *signature_len,
                             int veccount,
                             const struct iovec datavec[],
                             void **abstract)
{
    libssh2_dsa_ctx *dsactx = (libssh2_dsa_ctx *) (*abstract);  // 定义并初始化 DSA 上下文
    unsigned char hash[SHA_DIGEST_LENGTH];  // 定义存储哈希值的数组
    libssh2_sha1_ctx ctx;  // 定义 SHA1 上下文
    int i;  // 定义循环变量

    *signature = LIBSSH2_CALLOC(session, 2 * SHA_DIGEST_LENGTH);  // 分配存储签名的内存空间
    if(!*signature) {  // 如果内存分配失败
        return -1;  // 返回错误
    }

    *signature_len = 2 * SHA_DIGEST_LENGTH;  // 设置签名长度

    libssh2_sha1_init(&ctx);  // 初始化 SHA1 上下文
    for(i = 0; i < veccount; i++) {  // 遍历数据向量数组
        libssh2_sha1_update(ctx, datavec[i].iov_base, datavec[i].iov_len);  // 更新 SHA1 上下文的哈希值
    }
    libssh2_sha1_final(ctx, hash);  // 计算最终的 SHA1 哈希值

    if(_libssh2_dsa_sha1_sign(dsactx, hash, SHA_DIGEST_LENGTH, *signature)) {  // 使用 DSA 上下文对哈希值进行签名
        LIBSSH2_FREE(session, *signature);  // 释放签名内存空间
        return -1;  // 返回错误
    }

    return 0;  // 返回成功
}

/*
 * libssh2_hostkey_method_ssh_dss_dtor
 *
 * Shutdown the hostkey method
 */
static int
hostkey_method_ssh_dss_dtor(LIBSSH2_SESSION * session, void **abstract)
{
    libssh2_dsa_ctx *dsactx = (libssh2_dsa_ctx *) (*abstract);  // 定义并初始化 DSA 上下文
    (void) session;  // 防止编译器警告

    _libssh2_dsa_free(dsactx);  // 释放 DSA 上下文

    *abstract = NULL;  // 将抽象指针置空

    return 0;  // 返回成功
}

static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ssh_dss = {
    "ssh-dss",  // 设置主机密钥方法的名称
    MD5_DIGEST_LENGTH,  // 设置哈希长度
    hostkey_method_ssh_dss_init,  // 初始化主机密钥方法
    hostkey_method_ssh_dss_initPEM,  // 以 PEM 格式初始化主机密钥方法
    hostkey_method_ssh_dss_initPEMFromMemory,  // 从内存中以 PEM 格式初始化主机密钥方法
    hostkey_method_ssh_dss_sig_verify,  // 验证主机密钥方法的签名
    hostkey_method_ssh_dss_signv,  // 对数据进行签名
    NULL,  // 加密方法为空
    hostkey_method_ssh_dss_dtor,  // 关闭主机密钥方法
};
#endif /* LIBSSH2_DSA */

#if LIBSSH2_ECDSA

/* ***********
 * ecdsa-sha2-nistp256/384/521 *
 *********** */

static int
hostkey_method_ssh_ecdsa_dtor(LIBSSH2_SESSION * session,
                              void **abstract);
/*
 * hostkey_method_ssh_ecdsa_init
 *
 * 初始化具有 e/n 对的服务器主机密钥工作区
 */
static int
hostkey_method_ssh_ecdsa_init(LIBSSH2_SESSION * session,
                          const unsigned char *hostkey_data,
                          size_t hostkey_data_len,
                          void **abstract)
{
    libssh2_ecdsa_ctx *ecdsactx = NULL;
    unsigned char *type_str, *domain, *public_key;
    size_t key_len, len;
    libssh2_curve_type type;
    struct string_buf buf;

    if(abstract != NULL && *abstract) {
        hostkey_method_ssh_ecdsa_dtor(session, abstract);
        *abstract = NULL;
    }

    if(hostkey_data_len < 39) {
        _libssh2_debug(session, LIBSSH2_TRACE_ERROR,
                       "host key length too short");
        return -1;
    }

    buf.data = (unsigned char *)hostkey_data;
    buf.dataptr = buf.data;
    buf.len = hostkey_data_len;

    if(_libssh2_get_string(&buf, &type_str, &len) || len != 19)
        return -1;

    if(strncmp((char *) type_str, "ecdsa-sha2-nistp256", 19) == 0) {
        type = LIBSSH2_EC_CURVE_NISTP256;
    }
    else if(strncmp((char *) type_str, "ecdsa-sha2-nistp384", 19) == 0) {
        type = LIBSSH2_EC_CURVE_NISTP384;
    }
    else if(strncmp((char *) type_str, "ecdsa-sha2-nistp521", 19) == 0) {
        type = LIBSSH2_EC_CURVE_NISTP521;
    }
    else {
        return -1;
    }

    if(_libssh2_get_string(&buf, &domain, &len) || len != 8)
        return -1;

    if(type == LIBSSH2_EC_CURVE_NISTP256 &&
       strncmp((char *)domain, "nistp256", 8) != 0) {
        return -1;
    }
    else if(type == LIBSSH2_EC_CURVE_NISTP384 &&
            strncmp((char *)domain, "nistp384", 8) != 0) {
        return -1;
    }
    else if(type == LIBSSH2_EC_CURVE_NISTP521 &&
            strncmp((char *)domain, "nistp521", 8) != 0) {
        return -1;
    }

    /* 公钥 */
    if(_libssh2_get_string(&buf, &public_key, &key_len))
        return -1;
}
    # 如果 _libssh2_ecdsa_curve_name_with_octal_new 函数返回非零值，表示出错，直接返回 -1
    if(_libssh2_ecdsa_curve_name_with_octal_new(&ecdsactx, public_key,
                                                key_len, type))
        return -1;

    # 如果 abstract 参数不为空，将 ecdsactx 赋值给 abstract
    if(abstract != NULL)
        *abstract = ecdsactx;

    # 返回 0 表示执行成功
    return 0;
/*
 * hostkey_method_ssh_ecdsa_initPEM
 *
 * 从 PEM 文件中加载私钥
 */
static int
hostkey_method_ssh_ecdsa_initPEM(LIBSSH2_SESSION * session,
                             const char *privkeyfile,
                             unsigned const char *passphrase,
                             void **abstract)
{
    libssh2_ecdsa_ctx *ec_ctx = NULL;
    int ret;

    if(abstract != NULL && *abstract) {
        hostkey_method_ssh_ecdsa_dtor(session, abstract);
        *abstract = NULL;
    }

    ret = _libssh2_ecdsa_new_private(&ec_ctx, session,
                                     privkeyfile, passphrase);

    if(abstract != NULL)
        *abstract = ec_ctx;

    return ret;
}

/*
 * hostkey_method_ssh_ecdsa_initPEMFromMemory
 *
 * 从内存中加载私钥
 */
static int
hostkey_method_ssh_ecdsa_initPEMFromMemory(LIBSSH2_SESSION * session,
                                         const char *privkeyfiledata,
                                         size_t privkeyfiledata_len,
                                         unsigned const char *passphrase,
                                         void **abstract)
{
    libssh2_ecdsa_ctx *ec_ctx = NULL;
    int ret;

    if(abstract != NULL && *abstract) {
        hostkey_method_ssh_ecdsa_dtor(session, abstract);
        *abstract = NULL;
    }

    ret = _libssh2_ecdsa_new_private_frommemory(&ec_ctx, session,
                                                privkeyfiledata,
                                                privkeyfiledata_len,
                                                passphrase);
    if(ret) {
        return -1;
    }

    if(abstract != NULL)
        *abstract = ec_ctx;

    return 0;
}

/*
 * hostkey_method_ecdsa_sig_verify
 *
 * 验证远程创建的签名
 */
static int
# 用于验证 SSH ECDSA 签名的函数
hostkey_method_ssh_ecdsa_sig_verify(LIBSSH2_SESSION * session,
                                    const unsigned char *sig,
                                    size_t sig_len,
                                    const unsigned char *m,
                                    size_t m_len, void **abstract)
{
    unsigned char *r, *s, *name;  // 定义变量 r, s, name
    size_t r_len, s_len, name_len;  // 定义变量 r_len, s_len, name_len
    uint32_t len;  // 定义变量 len
    struct string_buf buf;  // 定义结构体变量 buf
    libssh2_ecdsa_ctx *ctx = (libssh2_ecdsa_ctx *) (*abstract);  // 将 abstract 转换为 libssh2_ecdsa_ctx 类型的指针，并赋值给 ctx

    (void) session;  // 忽略 session 参数

    if(sig_len < 35)  // 如果签名长度小于 35，则返回 -1
        return -1;

    /* keyname_len(4) + keyname(19){"ecdsa-sha2-nistp256"} +
       signature_len(4) */
    buf.data = (unsigned char *)sig;  // 将 sig 转换为 unsigned char 类型，并赋值给 buf.data
    buf.dataptr = buf.data;  // 将 buf.data 的值赋给 buf.dataptr
    buf.len = sig_len;  // 将 sig_len 的值赋给 buf.len

   if(_libssh2_get_string(&buf, &name, &name_len) || name_len != 19)  // 调用 _libssh2_get_string 函数，如果返回值不为 0 或者 name_len 不等于 19，则返回 -1
        return -1;

    if(_libssh2_get_u32(&buf, &len) != 0 || len < 8)  // 调用 _libssh2_get_u32 函数，如果返回值不为 0 或者 len 小于 8，则返回 -1
        return -1;

    if(_libssh2_get_string(&buf, &r, &r_len))  // 调用 _libssh2_get_string 函数，如果返回值不为 0，则返回 -1
       return -1;

    if(_libssh2_get_string(&buf, &s, &s_len))  // 调用 _libssh2_get_string 函数，如果返回值不为 0，则返回 -1
        return -1;

    return _libssh2_ecdsa_verify(ctx, r, r_len, s, s_len, m, m_len);  // 调用 _libssh2_ecdsa_verify 函数，返回结果
}


#define LIBSSH2_HOSTKEY_METHOD_EC_SIGNV_HASH(digest_type)               \
    # 定义一个无符号字符数组来存储哈希值
    unsigned char hash[SHA##digest_type##_DIGEST_LENGTH];
    # 定义一个用于计算哈希值的上下文对象
    libssh2_sha##digest_type##_ctx ctx;
    # 定义一个整型变量 i
    int i;
    # 初始化哈希计算上下文对象
    libssh2_sha##digest_type##_init(&ctx);
    # 遍历数据向量数组，更新哈希计算上下文对象
    for(i = 0; i < veccount; i++) {
        libssh2_sha##digest_type##_update(ctx, datavec[i].iov_base,
                                          datavec[i].iov_len);
    }
    # 完成哈希计算，得到最终的哈希值
    libssh2_sha##digest_type##_final(ctx, hash);
    # 调用 _libssh2_ecdsa_sign 函数进行签名
    ret = _libssh2_ecdsa_sign(session, ec_ctx, hash,
                              SHA##digest_type##_DIGEST_LENGTH,
                              signature, signature_len);
/*
 * hostkey_method_ecdsa_signv
 *
 * Construct a signature from an array of vectors
 */
static int
hostkey_method_ssh_ecdsa_signv(LIBSSH2_SESSION * session,
                               unsigned char **signature,
                               size_t *signature_len,
                               int veccount,
                               const struct iovec datavec[],
                               void **abstract)
{
    libssh2_ecdsa_ctx *ec_ctx = (libssh2_ecdsa_ctx *) (*abstract); // 从抽象指针中获取 ECDSA 上下文
    libssh2_curve_type type = _libssh2_ecdsa_get_curve_type(ec_ctx); // 获取 ECDSA 曲线类型
    int ret = 0; // 初始化返回值

    if(type == LIBSSH2_EC_CURVE_NISTP256) { // 如果曲线类型是 NISTP256
        LIBSSH2_HOSTKEY_METHOD_EC_SIGNV_HASH(256); // 调用宏，构造签名
    }
    else if(type == LIBSSH2_EC_CURVE_NISTP384) { // 如果曲线类型是 NISTP384
        LIBSSH2_HOSTKEY_METHOD_EC_SIGNV_HASH(384); // 调用宏，构造签名
    }
    else if(type == LIBSSH2_EC_CURVE_NISTP521) { // 如果曲线类型是 NISTP521
        LIBSSH2_HOSTKEY_METHOD_EC_SIGNV_HASH(512); // 调用宏，构造签名
    }
    else {
        return -1; // 如果曲线类型不匹配，则返回错误
    }

    return ret; // 返回结果
}

/*
 * hostkey_method_ssh_ecdsa_dtor
 *
 * Shutdown the hostkey by freeing EC_KEY context
 */
static int
hostkey_method_ssh_ecdsa_dtor(LIBSSH2_SESSION * session, void **abstract)
{
    libssh2_ecdsa_ctx *keyctx = (libssh2_ecdsa_ctx *) (*abstract); // 从抽象指针中获取 ECDSA 上下文
    (void) session; // 忽略会话参数

    if(keyctx != NULL) // 如果上下文不为空
        _libssh2_ecdsa_free(keyctx); // 释放 ECDSA 上下文

    *abstract = NULL; // 将抽象指针置为空

    return 0; // 返回成功
}

static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ecdsa_ssh_nistp256 = {
    "ecdsa-sha2-nistp256", // ECDSA 算法名称
    SHA256_DIGEST_LENGTH, // SHA256 摘要长度
    hostkey_method_ssh_ecdsa_init, // 初始化 ECDSA 主机密钥
    hostkey_method_ssh_ecdsa_initPEM, // 从 PEM 格式初始化 ECDSA 主机密钥
    hostkey_method_ssh_ecdsa_initPEMFromMemory, // 从内存中的 PEM 格式初始化 ECDSA 主机密钥
    hostkey_method_ssh_ecdsa_sig_verify, // 验证 ECDSA 签名
    hostkey_method_ssh_ecdsa_signv, // ECDSA 签名
    NULL,                       /* encrypt */
    hostkey_method_ssh_ecdsa_dtor, // 销毁 ECDSA 主机密钥
};

static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ecdsa_ssh_nistp384 = {
    "ecdsa-sha2-nistp384", // ECDSA 算法名称
    SHA384_DIGEST_LENGTH, // SHA384 摘要长度
    hostkey_method_ssh_ecdsa_init, // 初始化 ECDSA 主机密钥
    hostkey_method_ssh_ecdsa_initPEM, // 从 PEM 格式初始化 ECDSA 主机密钥
    hostkey_method_ssh_ecdsa_initPEMFromMemory, // 从内存中的 PEM 格式初始化 ECDSA 主机密钥
    hostkey_method_ssh_ecdsa_sig_verify,  # 使用 ECDSA 签名验证方法
    hostkey_method_ssh_ecdsa_signv,  # 使用 ECDSA 签名方法
    NULL,                       /* encrypt */  # 空值，表示没有加密方法
    hostkey_method_ssh_ecdsa_dtor,  # 使用 ECDSA 析构方法
};

// 定义了一个名为hostkey_method_ecdsa_ssh_nistp521的静态常量，包含了ECDSA算法的相关信息
static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ecdsa_ssh_nistp521 = {
    "ecdsa-sha2-nistp521",  // 算法名称
    SHA512_DIGEST_LENGTH,   // 摘要长度
    hostkey_method_ssh_ecdsa_init,  // 初始化函数
    hostkey_method_ssh_ecdsa_initPEM,  // PEM格式初始化函数
    hostkey_method_ssh_ecdsa_initPEMFromMemory,  // 从内存中加载PEM格式初始化函数
    hostkey_method_ssh_ecdsa_sig_verify,  // 签名验证函数
    hostkey_method_ssh_ecdsa_signv,  // 签名函数
    NULL,  // 加密函数
    hostkey_method_ssh_ecdsa_dtor,  // 析构函数
};

// 定义了一个名为hostkey_method_ecdsa_ssh_nistp256_cert的静态常量，包含了ECDSA算法的相关信息
static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ecdsa_ssh_nistp256_cert = {
    "ecdsa-sha2-nistp256-cert-v01@openssh.com",  // 算法名称
    SHA256_DIGEST_LENGTH,  // 摘要长度
    NULL,  // 初始化函数
    hostkey_method_ssh_ecdsa_initPEM,  // PEM格式初始化函数
    hostkey_method_ssh_ecdsa_initPEMFromMemory,  // 从内存中加载PEM格式初始化函数
    NULL,  // 签名验证函数
    hostkey_method_ssh_ecdsa_signv,  // 签名函数
    NULL,  // 加密函数
    hostkey_method_ssh_ecdsa_dtor,  // 析构函数
};

// 定义了一个名为hostkey_method_ecdsa_ssh_nistp384_cert的静态常量，包含了ECDSA算法的相关信息
static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ecdsa_ssh_nistp384_cert = {
    "ecdsa-sha2-nistp384-cert-v01@openssh.com",  // 算法名称
    SHA384_DIGEST_LENGTH,  // 摘要长度
    NULL,  // 初始化函数
    hostkey_method_ssh_ecdsa_initPEM,  // PEM格式初始化函数
    hostkey_method_ssh_ecdsa_initPEMFromMemory,  // 从内存中加载PEM格式初始化函数
    NULL,  // 签名验证函数
    hostkey_method_ssh_ecdsa_signv,  // 签名函数
    NULL,  // 加密函数
    hostkey_method_ssh_ecdsa_dtor,  // 析构函数
};

// 定义了一个名为hostkey_method_ecdsa_ssh_nistp521_cert的静态常量，包含了ECDSA算法的相关信息
static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ecdsa_ssh_nistp521_cert = {
    "ecdsa-sha2-nistp521-cert-v01@openssh.com",  // 算法名称
    SHA512_DIGEST_LENGTH,  // 摘要长度
    NULL,  // 初始化函数
    hostkey_method_ssh_ecdsa_initPEM,  // PEM格式初始化函数
    hostkey_method_ssh_ecdsa_initPEMFromMemory,  // 从内存中加载PEM格式初始化函数
    NULL,  // 签名验证函数
    hostkey_method_ssh_ecdsa_signv,  // 签名函数
    NULL,  // 加密函数
    hostkey_method_ssh_ecdsa_dtor,  // 析构函数
};

#endif /* LIBSSH2_ECDSA */

#if LIBSSH2_ED25519

// 定义了一个名为hostkey_method_ssh_ed25519_dtor的静态函数，用于销毁ED25519算法的相关信息
static int hostkey_method_ssh_ed25519_dtor(LIBSSH2_SESSION * session,
                                           void **abstract);

/*
 * hostkey_method_ssh_ed25519_init
 *
 * Initialize the server hostkey working area with e/n pair
 */
static int
# 初始化 SSH ed25519 主机密钥方法
def hostkey_method_ssh_ed25519_init(LIBSSH2_SESSION * session,
                                    const unsigned char *hostkey_data,
                                    size_t hostkey_data_len,
                                    void **abstract)
{
    const unsigned char *s;  # 定义一个指向无符号字符的指针变量
    unsigned long len, key_len;  # 定义两个无符号长整型变量
    libssh2_ed25519_ctx *ctx = NULL;  # 定义一个 ed25519 上下文指针变量并初始化为 NULL

    if(*abstract) {  # 如果抽象指针不为空
        hostkey_method_ssh_ed25519_dtor(session, abstract);  # 调用主机密钥方法的析构函数
        *abstract = NULL;  # 将抽象指针置为空
    }

    if(hostkey_data_len < 19) {  # 如果主机密钥数据长度小于19
        _libssh2_debug(session, LIBSSH2_TRACE_ERROR,  # 调用 libssh2 调试函数
                       "host key length too short");  # 输出错误信息
        return -1;  # 返回-1
    }

    s = hostkey_data;  # 将主机密钥数据赋值给 s
    len = _libssh2_ntohu32(s);  # 将 s 转换为无符号长整型并赋值给 len
    s += 4;  # s 指针向后移动4个字节

    if(len != 11 || strncmp((char *) s, "ssh-ed25519", 11) != 0) {  # 如果 len 不等于11或者 s 不以 "ssh-ed25519" 开头
        return -1;  # 返回-1
    }

    s += 11;  # s 指针向后移动11个字节

    /* public key */
    key_len = _libssh2_ntohu32(s);  # 将 s 转换为无符号长整型并赋值给 key_len
    s += 4;  # s 指针向后移动4个字节

    if(_libssh2_ed25519_new_public(&ctx, session, s, key_len) != 0) {  # 如果创建新的 ed25519 公钥失败
        return -1;  # 返回-1
    }

    *abstract = ctx;  # 将 ctx 赋值给抽象指针

    return 0;  # 返回0
}

'''
 * hostkey_method_ssh_ed25519_initPEM
 *
 * 从 PEM 文件加载私钥
 */
static int
hostkey_method_ssh_ed25519_initPEM(LIBSSH2_SESSION * session,
                             const char *privkeyfile,
                             unsigned const char *passphrase,
                             void **abstract)
{
    libssh2_ed25519_ctx *ec_ctx = NULL;  # 定义一个 ed25519 上下文指针变量并初始化为 NULL
    int ret;  # 定义一个整型变量

    if(*abstract) {  # 如果抽象指针不为空
        hostkey_method_ssh_ed25519_dtor(session, abstract);  # 调用主机密钥方法的析构函数
        *abstract = NULL;  # 将抽象指针置为空
    }

    ret = _libssh2_ed25519_new_private(&ec_ctx, session,  # 调用 libssh2 创建新的 ed25519 私钥
                                       privkeyfile, passphrase);
    if(ret) {  # 如果返回值不为0
        return -1;  # 返回-1
    }

    *abstract = ec_ctx;  # 将 ec_ctx 赋值给抽象指针

    return ret;  # 返回 ret
}

'''
 * hostkey_method_ssh_ed25519_initPEMFromMemory
 *
 * 从内存加载私钥
 */
static int
# 从内存中初始化 SSH ED25519 主机密钥方法
def hostkey_method_ssh_ed25519_initPEMFromMemory(LIBSSH2_SESSION * session,
                                                 const char *privkeyfiledata,
                                                 size_t privkeyfiledata_len,
                                                 unsigned const char *passphrase,
                                                 void **abstract)
{
    # 初始化 ED25519 上下文
    libssh2_ed25519_ctx *ed_ctx = NULL;
    int ret;

    # 如果抽象指针不为空且指向的内容不为空
    if(abstract != NULL && *abstract) {
        # 调用主机密钥方法的析构函数
        hostkey_method_ssh_ed25519_dtor(session, abstract);
        # 将抽象指针置为空
        *abstract = NULL;
    }

    # 从内存中创建新的私钥
    ret = _libssh2_ed25519_new_private_frommemory(&ed_ctx, session,
                                                  privkeyfiledata,
                                                  privkeyfiledata_len,
                                                  passphrase);
    # 如果创建私钥失败
    if(ret) {
        # 返回错误
        return -1;
    }

    # 如果抽象指针不为空
    if(abstract != NULL)
        # 将抽象指针指向 ED25519 上下文
        *abstract = ed_ctx;

    # 返回成功
    return 0;
}

'''
 * hostkey_method_ssh_ed25519_sig_verify
 *
 * Verify signature created by remote
 */
static int
hostkey_method_ssh_ed25519_sig_verify(LIBSSH2_SESSION * session,
                                      const unsigned char *sig,
                                      size_t sig_len,
                                      const unsigned char *m,
                                      size_t m_len, void **abstract)
{
    # 获取抽象指针指向的 ED25519 上下文
    libssh2_ed25519_ctx *ctx = (libssh2_ed25519_ctx *) (*abstract);
    (void) session;

    # 如果签名长度小于19
    if(sig_len < 19)
        # 返回错误
        return -1;

    # 跳过 keyname_len(4) + keyname(11){"ssh-ed25519"} + signature_len(4)
    sig += 19;
    sig_len -= 19;

    # 如果签名长度不等于 ED25519 签名长度
    if(sig_len != LIBSSH2_ED25519_SIG_LEN)
        # 返回错误
        return -1;

    # 调用 ED25519 验证函数
    return _libssh2_ed25519_verify(ctx, sig, sig_len, m, m_len);
}

'''
 * hostkey_method_ssh_ed25519_signv
 *
 * Construct a signature from an array of vectors
 */
static int
# 定义一个函数，用于使用 SSH-ED25519 签名算法对数据进行签名
hostkey_method_ssh_ed25519_signv(LIBSSH2_SESSION * session,
                           unsigned char **signature,
                           size_t *signature_len,
                           int veccount,
                           const struct iovec datavec[],
                           void **abstract)
{
    # 从抽象指针中获取 ED25519 上下文
    libssh2_ed25519_ctx *ctx = (libssh2_ed25519_ctx *) (*abstract);

    # 如果数据向量的数量不等于1，则返回-1
    if(veccount != 1) {
        return -1;
    }

    # 调用内部函数对数据进行签名
    return _libssh2_ed25519_sign(ctx, session, signature, signature_len,
                                 datavec[0].iov_base, datavec[0].iov_len);
}


/*
 * hostkey_method_ssh_ed25519_dtor
 *
 * 关闭主机密钥，释放密钥上下文
 */
static int
hostkey_method_ssh_ed25519_dtor(LIBSSH2_SESSION * session, void **abstract)
{
    # 从抽象指针中获取 ED25519 上下文
    libssh2_ed25519_ctx *keyctx = (libssh2_ed25519_ctx*) (*abstract);
    (void) session;

    # 如果密钥上下文存在，则释放它
    if(keyctx)
        _libssh2_ed25519_free(keyctx);

    # 将抽象指针设置为NULL
    *abstract = NULL;

    # 返回0表示成功
    return 0;
}

# 定义一个 SSH-ED25519 主机密钥方法结构体
static const LIBSSH2_HOSTKEY_METHOD hostkey_method_ssh_ed25519 = {
    "ssh-ed25519",  # 算法名称
    SHA256_DIGEST_LENGTH,  # 摘要长度
    hostkey_method_ssh_ed25519_init,  # 初始化方法
    hostkey_method_ssh_ed25519_initPEM,  # 从PEM格式初始化方法
    hostkey_method_ssh_ed25519_initPEMFromMemory,  # 从内存中的PEM格式初始化方法
    hostkey_method_ssh_ed25519_sig_verify,  # 签名验证方法
    hostkey_method_ssh_ed25519_signv,  # 签名方法
    NULL,                       # 加密方法
    hostkey_method_ssh_ed25519_dtor,  # 关闭方法
};

#endif /*LIBSSH2_ED25519*/


# 定义一个主机密钥方法结构体数组
static const LIBSSH2_HOSTKEY_METHOD *hostkey_methods[] = {
#if LIBSSH2_ECDSA
    &hostkey_method_ecdsa_ssh_nistp256,  # ECDSA NIST P-256 算法
    &hostkey_method_ecdsa_ssh_nistp384,  # ECDSA NIST P-384 算法
    &hostkey_method_ecdsa_ssh_nistp521,  # ECDSA NIST P-521 算法
    &hostkey_method_ecdsa_ssh_nistp256_cert,  # ECDSA NIST P-256 证书算法
    &hostkey_method_ecdsa_ssh_nistp384_cert,  # ECDSA NIST P-384 证书算法
    &hostkey_method_ecdsa_ssh_nistp521_cert,  # ECDSA NIST P-521 证书算法
#endif
#if LIBSSH2_ED25519
    &hostkey_method_ssh_ed25519,  # SSH-ED25519 算法
#endif
#if LIBSSH2_RSA
    &hostkey_method_ssh_rsa,  # SSH-RSA 算法
#endif /* LIBSSH2_RSA */
#if LIBSSH2_DSA
    &hostkey_method_ssh_dss,  # SSH-DSA 算法
#endif /* LIBSSH2_DSA */
    NULL  # 结束标志
};

# 获取主机密钥方法结构体数组的指针
const LIBSSH2_HOSTKEY_METHOD **
libssh2_hostkey_methods(void)
{
    # 返回主机密钥方法
    return hostkey_methods;
}
/*
 * libssh2_hostkey_hash
 *
 * 返回哈希签名
 * 返回的缓冲区不应该被释放
 * 缓冲区的长度由哈希类型确定
 * 例如 MD5 == 16, SHA1 == 20, SHA256 == 32
 */
LIBSSH2_API const char *
libssh2_hostkey_hash(LIBSSH2_SESSION * session, int hash_type)
{
    switch(hash_type) {
#if LIBSSH2_MD5
    case LIBSSH2_HOSTKEY_HASH_MD5:
        return (session->server_hostkey_md5_valid)
          ? (char *) session->server_hostkey_md5
          : NULL;
        break;
#endif /* LIBSSH2_MD5 */
    case LIBSSH2_HOSTKEY_HASH_SHA1:
        return (session->server_hostkey_sha1_valid)
          ? (char *) session->server_hostkey_sha1
          : NULL;
        break;
    case LIBSSH2_HOSTKEY_HASH_SHA256:
        return (session->server_hostkey_sha256_valid)
          ? (char *) session->server_hostkey_sha256
          : NULL;
        break;
    default:
        return NULL;
    }
}

static int hostkey_type(const unsigned char *hostkey, size_t len)
{
    static const unsigned char rsa[] = {
        0, 0, 0, 0x07, 's', 's', 'h', '-', 'r', 's', 'a'
    };
    static const unsigned char dss[] = {
        0, 0, 0, 0x07, 's', 's', 'h', '-', 'd', 's', 's'
    };
    static const unsigned char ecdsa_256[] = {
        0, 0, 0, 0x13, 'e', 'c', 'd', 's', 'a', '-', 's', 'h', 'a', '2', '-',
        'n', 'i', 's', 't', 'p', '2', '5', '6'
    };
    static const unsigned char ecdsa_384[] = {
        0, 0, 0, 0x13, 'e', 'c', 'd', 's', 'a', '-', 's', 'h', 'a', '2', '-',
        'n', 'i', 's', 't', 'p', '3', '8', '4'
    };
    static const unsigned char ecdsa_521[] = {
        0, 0, 0, 0x13, 'e', 'c', 'd', 's', 'a', '-', 's', 'h', 'a', '2', '-',
        'n', 'i', 's', 't', 'p', '5', '2', '1'
    };
    static const unsigned char ed25519[] = {
        0, 0, 0, 0x0b, 's', 's', 'h', '-', 'e', 'd', '2', '5', '5', '1', '9'
    };

    if(len < 11)
        return LIBSSH2_HOSTKEY_TYPE_UNKNOWN;

    if(!memcmp(rsa, hostkey, 11))
        return LIBSSH2_HOSTKEY_TYPE_RSA;
    # 如果给定的主机密钥与 DSS 类型的主机密钥匹配，则返回 DSS 类型
    if(!memcmp(dss, hostkey, 11))
        return LIBSSH2_HOSTKEY_TYPE_DSS;

    # 如果主机密钥长度小于 15，则返回未知类型
    if(len < 15)
        return LIBSSH2_HOSTKEY_TYPE_UNKNOWN;

    # 如果给定的主机密钥与 ED25519 类型的主机密钥匹配，则返回 ED25519 类型
    if(!memcmp(ed25519, hostkey, 15))
        return LIBSSH2_HOSTKEY_TYPE_ED25519;

    # 如果主机密钥长度小于 23，则返回未知类型
    if(len < 23)
        return LIBSSH2_HOSTKEY_TYPE_UNKNOWN;

    # 如果给定的主机密钥与 ECDSA 256 类型的主机密钥匹配，则返回 ECDSA 256 类型
    if(!memcmp(ecdsa_256, hostkey, 23))
        return LIBSSH2_HOSTKEY_TYPE_ECDSA_256;

    # 如果给定的主机密钥与 ECDSA 384 类型的主机密钥匹配，则返回 ECDSA 384 类型
    if(!memcmp(ecdsa_384, hostkey, 23))
        return LIBSSH2_HOSTKEY_TYPE_ECDSA_384;

    # 如果给定的主机密钥与 ECDSA 521 类型的主机密钥匹配，则返回 ECDSA 521 类型
    if(!memcmp(ecdsa_521, hostkey, 23))
        return LIBSSH2_HOSTKEY_TYPE_ECDSA_521;

    # 如果以上条件都不满足，则返回未知类型
    return LIBSSH2_HOSTKEY_TYPE_UNKNOWN;
// 结束 libssh2_session_hostkey 函数的定义

/*
 * libssh2_session_hostkey()
 *
 * 返回服务器密钥和长度。
 *
 */
LIBSSH2_API const char *
libssh2_session_hostkey(LIBSSH2_SESSION *session, size_t *len, int *type)
{
    // 如果服务器密钥长度不为0
    if(session->server_hostkey_len) {
        // 如果 len 参数不为空，将服务器密钥长度赋给 len
        if(len)
            *len = session->server_hostkey_len;
        // 如果 type 参数不为空，调用 hostkey_type 函数获取服务器密钥类型，并赋给 type
        if(type)
            *type = hostkey_type(session->server_hostkey,
                                 session->server_hostkey_len);
        // 返回服务器密钥的字符指针类型
        return (char *) session->server_hostkey;
    }
    // 如果 len 参数不为空，将长度赋为0
    if(len)
        *len = 0;
    // 返回空指针
    return NULL;
}
```