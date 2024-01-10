# `nmap\libssh2\src\crypt.c`

```
/*
 * 版权声明和许可声明
 * 保留所有权利
 * 允许在源代码和二进制形式下进行再发布和使用，无论是否进行修改，前提是满足以下条件：
 *   1. 源代码的再发布必须保留上述版权声明、条件列表和以下免责声明。
 *   2. 二进制形式的再发布必须在文档和/或其他提供的材料中重现上述版权声明、条件列表和以下免责声明。
 *   3. 未经特定事先书面许可，不得使用版权所有者的名称或任何其他贡献者的名称来认可或推广从本软件派生的产品。
 * 本软件由版权所有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于适销性和特定用途适用性的暗示担保。无论在任何情况下，版权所有者或贡献者均不对任何直接、间接、附带、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）承担责任，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害。
 */

#include "libssh2_priv.h"

#ifdef LIBSSH2_CRYPT_NONE

/* crypt_none_crypt
 * Minimalist cipher: VERY secure *wink*
 */
static int
crypt_none_crypt(LIBSSH2_SESSION * session, unsigned char *buf,
                         void **abstract)
{
    /* 对数据不做任何处理 */
    return 0;
}
# 定义一个名为libssh2_crypt_method_none的常量，表示没有加密方法
static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_none = {
    "none",                  # 加密方法名称
    "DEK-Info: NONE",        # 加密信息
    8,                       # 块大小（SSH2将最小块大小定义为8）
    0,                       # 初始化向量长度
    0,                       # 密钥长度
    0,                       # 标志
    NULL,                    # 无
    crypt_none_crypt,        # 使用crypt_none_crypt函数进行加密
    NULL                     # 无
};
#endif /* LIBSSH2_CRYPT_NONE */

# 定义一个名为crypt_ctx的结构体，表示加密上下文
struct crypt_ctx
{
    int encrypt;              # 加密标志
    _libssh2_cipher_type(algo);  # 加密算法类型
    _libssh2_cipher_ctx h;    # 加密上下文
};

# 定义一个名为crypt_init的函数，用于初始化加密上下文
static int
crypt_init(LIBSSH2_SESSION * session,
           const LIBSSH2_CRYPT_METHOD * method,
           unsigned char *iv, int *free_iv,
           unsigned char *secret, int *free_secret,
           int encrypt, void **abstract)
{
    struct crypt_ctx *ctx = LIBSSH2_ALLOC(session, sizeof(struct crypt_ctx));  # 分配加密上下文内存
    if(!ctx)
        return LIBSSH2_ERROR_ALLOC;  # 如果分配失败，则返回分配错误

    ctx->encrypt = encrypt;  # 设置加密标志
    ctx->algo = method->algo;  # 设置加密算法
    if(_libssh2_cipher_init(&ctx->h, ctx->algo, iv, secret, encrypt)) {  # 初始化加密上下文
        LIBSSH2_FREE(session, ctx);  # 释放加密上下文内存
        return -1;  # 返回错误
    }
    *abstract = ctx;  # 设置抽象指针
    *free_iv = 1;  # 释放初始化向量
    *free_secret = 1;  # 释放密钥
    return 0;  # 返回成功
}

# 定义一个名为crypt_encrypt的函数，用于加密数据
static int
crypt_encrypt(LIBSSH2_SESSION * session, unsigned char *block,
              size_t blocksize, void **abstract)
{
    struct crypt_ctx *cctx = *(struct crypt_ctx **) abstract;  # 获取加密上下文
    (void) session;  # 未使用的参数
    return _libssh2_cipher_crypt(&cctx->h, cctx->algo, cctx->encrypt, block,  # 使用加密上下文对数据进行加密
                                 blocksize);
}

# 定义一个名为crypt_dtor的函数，用于销毁加密上下文
static int
crypt_dtor(LIBSSH2_SESSION * session, void **abstract)
{
    struct crypt_ctx **cctx = (struct crypt_ctx **) abstract;  # 获取加密上下文指针
    if(cctx && *cctx) {
        _libssh2_cipher_dtor(&(*cctx)->h);  # 销毁加密上下文
        LIBSSH2_FREE(session, *cctx);  # 释放加密上下文内存
        *abstract = NULL;  # 设置抽象指针为空
    }
    return 0;  # 返回成功
}

#if LIBSSH2_AES_CTR
# 定义一个名为libssh2_crypt_method_aes128_ctr的常量，表示AES-128-CTR加密方法
static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_aes128_ctr = {
    "aes128-ctr",  # 加密方法名称
    "",             # 空字符串
    16,             # 块大小
    16,             # 初始值长度
    16,                         /* secret length -- 16*8 == 128bit */ 
    // 设置密钥长度为16，即128位
    0,                          /* flags */
    // 设置标志位为0
    &crypt_init,
    // 设置加密初始化函数
    &crypt_encrypt,
    // 设置加密函数
    &crypt_dtor,
    // 设置加密对象销毁函数
    _libssh2_cipher_aes128ctr
    // 使用AES128CTR算法进行加密
};  // 结束 libssh2_crypt_method_aes256_cbc 结构体定义

static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_aes192_ctr = {
    "aes192-ctr",  // 加密方法名称
    "",  // 附加信息为空
    16,  // 块大小
    16,  // 初始值长度
    24,  // 密钥长度 -- 24*8 == 192bit
    0,   // 标志位
    &crypt_init,  // 初始化函数指针
    &crypt_encrypt,  // 加密函数指针
    &crypt_dtor,  // 析构函数指针
    _libssh2_cipher_aes192ctr  // AES-192-CTR 加密算法
};

static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_aes256_ctr = {
    "aes256-ctr",  // 加密方法名称
    "",  // 附加信息为空
    16,  // 块大小
    16,  // 初始值长度
    32,  // 密钥长度 -- 32*8 == 256bit
    0,   // 标志位
    &crypt_init,  // 初始化函数指针
    &crypt_encrypt,  // 加密函数指针
    &crypt_dtor,  // 析构函数指针
    _libssh2_cipher_aes256ctr  // AES-256-CTR 加密算法
};
#endif

#if LIBSSH2_AES
static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_aes128_cbc = {
    "aes128-cbc",  // 加密方法名称
    "DEK-Info: AES-128-CBC",  // 附加信息
    16,  // 块大小
    16,  // 初始值长度
    16,  // 密钥长度 -- 16*8 == 128bit
    0,   // 标志位
    &crypt_init,  // 初始化函数指针
    &crypt_encrypt,  // 加密函数指针
    &crypt_dtor,  // 析构函数指针
    _libssh2_cipher_aes128  // AES-128-CBC 加密算法
};

static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_aes192_cbc = {
    "aes192-cbc",  // 加密方法名称
    "DEK-Info: AES-192-CBC",  // 附加信息
    16,  // 块大小
    16,  // 初始值长度
    24,  // 密钥长度 -- 24*8 == 192bit
    0,   // 标志位
    &crypt_init,  // 初始化函数指针
    &crypt_encrypt,  // 加密函数指针
    &crypt_dtor,  // 析构函数指针
    _libssh2_cipher_aes192  // AES-192-CBC 加密算法
};

static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_aes256_cbc = {
    "aes256-cbc",  // 加密方法名称
    "DEK-Info: AES-256-CBC",  // 附加信息
    16,  // 块大小
    16,  // 初始值长度
    32,  // 密钥长度 -- 32*8 == 256bit
    0,                          /* flags */ 
    // 设置标志位为0

    &crypt_init,
    // 指向加密初始化函数的指针

    &crypt_encrypt,
    // 指向加密函数的指针

    &crypt_dtor,
    // 指向加密销毁函数的指针

    _libssh2_cipher_aes256
    // 使用AES256算法进行加密
/* 定义 rijndael-cbc@lysator.liu.se 加密方法的参数和函数指针 */
static const LIBSSH2_CRYPT_METHOD
    libssh2_crypt_method_rijndael_cbc_lysator_liu_se = {
    "rijndael-cbc@lysator.liu.se",  // 加密方法名称
    "DEK-Info: AES-256-CBC",        // 加密信息
    16,                             // 块大小
    16,                             // 初始值长度
    32,                             // 密钥长度
    0,                              // 标志位
    &crypt_init,                    // 初始化函数指针
    &crypt_encrypt,                 // 加密函数指针
    &crypt_dtor,                    // 清理函数指针
    _libssh2_cipher_aes256          // AES256加密算法
};

/* 定义 blowfish-cbc 加密方法的参数和函数指针 */
static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_blowfish_cbc = {
    "blowfish-cbc",                 // 加密方法名称
    "",                             // 加密信息
    8,                              // 块大小
    8,                              // 初始值长度
    16,                             // 密钥长度
    0,                              // 标志位
    &crypt_init,                    // 初始化函数指针
    &crypt_encrypt,                 // 加密函数指针
    &crypt_dtor,                    // 清理函数指针
    _libssh2_cipher_blowfish        // Blowfish加密算法
};

/* 定义 arcfour 加密方法的参数和函数指针 */
static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_arcfour = {
    "arcfour",                      // 加密方法名称
    "DEK-Info: RC4",                // 加密信息
    8,                              // 块大小
    8,                              // 初始值长度
    16,                             // 密钥长度
    0,                              // 标志位
    &crypt_init,                    // 初始化函数指针
    &crypt_encrypt,                 // 加密函数指针
    &crypt_dtor,                    // 清理函数指针
    _libssh2_cipher_arcfour         // Arcfour加密算法
};

/* 初始化 arcfour128 加密方法 */
static int
crypt_init_arcfour128(LIBSSH2_SESSION * session,
                      const LIBSSH2_CRYPT_METHOD * method,
                      unsigned char *iv, int *free_iv,
                      unsigned char *secret, int *free_secret,
                      int encrypt, void **abstract)
{
    int rc;

    // 调用通用的初始化函数
    rc = crypt_init(session, method, iv, free_iv, secret, free_secret,
                    encrypt, abstract);
    # 如果 rc 等于 0，则执行以下代码块
    if(rc == 0) {
        # 从抽象指针中获取加密上下文结构体指针
        struct crypt_ctx *cctx = *(struct crypt_ctx **) abstract;
        # 创建一个长度为 8 的无符号字符数组
        unsigned char block[8];
        # 设置丢弃的字节数为 1536
        size_t discard = 1536;
        # 循环执行以下操作，直到丢弃的字节数为 0
        for(; discard; discard -= 8)
            # 使用加密上下文结构体中的加密算法对 block 进行加密
            _libssh2_cipher_crypt(&cctx->h, cctx->algo, cctx->encrypt, block,
                                  method->blocksize);
    }
    # 返回 rc 的值
    return rc;
}
# 定义一个名为libssh2_crypt_method_arcfour128的静态常量，表示arcfour128加密方法
static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_arcfour128 = {
    "arcfour128",               # 加密方法名称
    "",                         # 附加信息为空
    8,                          # 块大小为8
    8,                          # 初始值长度为8
    16,                         # 密钥长度为16
    0,                          # 标志位为0
    &crypt_init_arcfour128,     # 初始化加密方法的函数指针
    &crypt_encrypt,             # 加密函数的函数指针
    &crypt_dtor,                # 加密方法析构函数的函数指针
    _libssh2_cipher_arcfour     # 加密方法的具体实现
};
#endif /* LIBSSH2_RC4 */

#if LIBSSH2_CAST
# 定义一个名为libssh2_crypt_method_cast128_cbc的静态常量，表示cast128-cbc加密方法
static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_cast128_cbc = {
    "cast128-cbc",              # 加密方法名称
    "",                         # 附加信息为空
    8,                          # 块大小为8
    8,                          # 初始值长度为8
    16,                         # 密钥长度为16
    0,                          # 标志位为0
    &crypt_init,                # 初始化加密方法的函数指针
    &crypt_encrypt,             # 加密函数的函数指针
    &crypt_dtor,                # 加密方法析构函数的函数指针
    _libssh2_cipher_cast5       # 加密方法的具体实现
};
#endif /* LIBSSH2_CAST */

#if LIBSSH2_3DES
# 定义一个名为libssh2_crypt_method_3des_cbc的静态常量，表示3des-cbc加密方法
static const LIBSSH2_CRYPT_METHOD libssh2_crypt_method_3des_cbc = {
    "3des-cbc",                 # 加密方法名称
    "DEK-Info: DES-EDE3-CBC",   # 附加信息为DEK-Info: DES-EDE3-CBC
    8,                          # 块大小为8
    8,                          # 初始值长度为8
    24,                         # 密钥长度为24
    0,                          # 标志位为0
    &crypt_init,                # 初始化加密方法的函数指针
    &crypt_encrypt,             # 加密函数的函数指针
    &crypt_dtor,                # 加密方法析构函数的函数指针
    _libssh2_cipher_3des        # 加密方法的具体实现
};
#endif

# 定义一个名为_libssh2_crypt_methods的静态常量数组，包含各种加密方法
static const LIBSSH2_CRYPT_METHOD *_libssh2_crypt_methods[] = {
#if LIBSSH2_AES_CTR
  &libssh2_crypt_method_aes128_ctr,      # AES-CTR加密方法
  &libssh2_crypt_method_aes192_ctr,      # AES-CTR加密方法
  &libssh2_crypt_method_aes256_ctr,      # AES-CTR加密方法
#endif /* LIBSSH2_AES */
#if LIBSSH2_AES
    &libssh2_crypt_method_aes256_cbc,     # AES-CBC加密方法
    &libssh2_crypt_method_rijndael_cbc_lysator_liu_se,  # AES-CBC加密方法
    &libssh2_crypt_method_aes192_cbc,     # AES-CBC加密方法
    &libssh2_crypt_method_aes128_cbc,     # AES-CBC加密方法
#endif /* LIBSSH2_AES */
#if LIBSSH2_BLOWFISH
    &libssh2_crypt_method_blowfish_cbc,   # BLOWFISH-CBC加密方法
#endif /* LIBSSH2_BLOWFISH */
#if LIBSSH2_RC4
    &libssh2_crypt_method_arcfour128,     # ARCFOUR128加密方法
    &libssh2_crypt_method_arcfour,        # ARCFOUR加密方法
#endif /* LIBSSH2_RC4 */
#if LIBSSH2_CAST
    &libssh2_crypt_method_cast128_cbc,    # CAST128-CBC加密方法
#endif /* LIBSSH2_CAST */
#if LIBSSH2_3DES
    &libssh2_crypt_method_3des_cbc,  // 如果支持3DES，则添加3DES加密方法
#endif /*  LIBSSH2_DES */
#ifdef LIBSSH2_CRYPT_NONE
    &libssh2_crypt_method_none,  // 如果支持无加密，则添加无加密方法
#endif
    NULL  // 结束加密方法列表
};

/* Expose to kex.c */
const LIBSSH2_CRYPT_METHOD **
libssh2_crypt_methods(void)
{
    return _libssh2_crypt_methods;  // 返回加密方法列表
}
```