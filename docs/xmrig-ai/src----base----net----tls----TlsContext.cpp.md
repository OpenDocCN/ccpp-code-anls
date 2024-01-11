# `xmrig\src\base\net\tls\TlsContext.cpp`

```
/* XMRig
 * 版权所有（C）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（C）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 *   由自由软件基金会发布的版本3或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是在希望它有用的情况下分发的，
 *   但没有任何保证；甚至没有适用于特定目的的隐含保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/tls/TlsContext.h"
#include "base/io/Env.h"
#include "base/io/log/Log.h"
#include "base/net/tls/TlsConfig.h"


#include <openssl/ssl.h>
#include <openssl/err.h>


// https://wiki.openssl.org/index.php/OpenSSL_1.1.0_Changes#Compatibility_Layer
#if OPENSSL_VERSION_NUMBER < 0x10100000L
// 如果 OpenSSL 版本号小于 1.1.0，则定义 DH_set0_pqg 函数
int DH_set0_pqg(DH *dh, BIGNUM *p, BIGNUM *q, BIGNUM *g)
{
    assert(q == nullptr);

    // 设置 DH 对象的 p 和 g 参数
    dh->p = p;
    dh->g = g;

    return 1;
 }
#endif


namespace xmrig {


// https://wiki.openssl.org/index.php/Diffie-Hellman_parameters
#if OPENSSL_VERSION_NUMBER < 0x30000000L || defined(LIBRESSL_VERSION_NUMBER)
// 如果 OpenSSL 版本号小于 3.0.0 或者定义了 LIBRESSL_VERSION_NUMBER，则定义 get_dh2048 函数
static DH *get_dh2048()
{
    // 定义一个静态的 2048 位 DH 参数的质数
    static unsigned char dhp_2048[] = {
        0xB2, 0x91, 0xA7, 0x05, 0x31, 0xCE, 0x12, 0x9D, 0x03, 0x43,
        0xAF, 0x13, 0xAF, 0x4B, 0x8E, 0x4C, 0x04, 0x13, 0x4F, 0x72,
        0x00, 0x73, 0x2C, 0x67, 0xC3, 0xE0, 0x50, 0xBF, 0x72, 0x5E,
        0xBE, 0x45, 0x89, 0x4C, 0x01, 0x45, 0xA6, 0x5E, 0xA7, 0xA8,
        0xDC, 0x2F, 0x1D, 0x91, 0x2D, 0x58, 0x0D, 0x71, 0x97, 0x3D,
        0xAE, 0xFE, 0x86, 0x29, 0x37, 0x5F, 0x5E, 0x6D, 0x81, 0x56,
        0x07, 0x83, 0xF2, 0xF8, 0xEC, 0x4E, 0xF8, 0x7A, 0xEC, 0xEA,
        0xD9, 0xEA, 0x61, 0x3C, 0xAF, 0x51, 0x30, 0xB7, 0xA7, 0x67,
        0x3F, 0x59, 0xAD, 0x2E, 0x23, 0x57, 0x64, 0xA2, 0x99, 0x15,
        0xBD, 0xD9, 0x8D, 0xBA, 0xE6, 0x8F, 0xFB, 0xB3, 0x77, 0x3B,
        0xE6, 0x5C, 0xC1, 0x03, 0xCF, 0x38, 0xD4, 0xF6, 0x2E, 0x0B,
        0xF3, 0x20, 0xBE, 0xF0, 0xFC, 0x85, 0xEF, 0x5F, 0xCE, 0x0E,
        0x42, 0x17, 0x3B, 0x72, 0x43, 0x4C, 0x3A, 0xF5, 0xC8, 0xB4,
        0x40, 0x52, 0x03, 0x72, 0x9A, 0x2C, 0xA4, 0x23, 0x2A, 0xA2,
        0x52, 0xA3, 0xC2, 0x76, 0x08, 0x1C, 0x2E, 0x60, 0x44, 0xE4,
        0x12, 0x5D, 0x80, 0x47, 0x6C, 0x7A, 0x5A, 0x8E, 0x18, 0xC9,
        0x8C, 0x22, 0xC8, 0x07, 0x75, 0xE2, 0x77, 0x3A, 0x90, 0x2E,
        0x79, 0xC3, 0xF5, 0x4E, 0x4E, 0xDE, 0x14, 0x29, 0xA4, 0x5B,
        0x32, 0xCC, 0xE5, 0x05, 0x09, 0x2A, 0xC9, 0x1C, 0xB4, 0x8E,
        0x99, 0xCF, 0x57, 0xF2, 0x1B, 0x5F, 0x18, 0x89, 0x29, 0xF2,
        0xB0, 0xF3, 0xAC, 0x67, 0x16, 0x90, 0x4A, 0x1D, 0xD6, 0xF5,
        0x84, 0x71, 0x1D, 0x0E, 0x61, 0x5F, 0xE2, 0x2D, 0x52, 0x87,
        0x0D, 0x8F, 0x84, 0xCB, 0xFC, 0xF0, 0x5D, 0x4C, 0x9F, 0x59,
        0xA9, 0xD6, 0x83, 0x70, 0x4B, 0x98, 0x6A, 0xCA, 0x78, 0x53,
        0x27, 0x32, 0x59, 0x35, 0x0A, 0xB8, 0x29, 0x18, 0xAF, 0x58,
        0x45, 0x63, 0xEB, 0x43, 0x28, 0x7B
    };

    // 定义一个静态的 2048 位 DH 参数的生成元
    static unsigned char dhg_2048[] = { 0x02 };

    // 创建一个 DH 对象
    auto dh = DH_new();
    // 如果创建失败，返回空指针
    if (dh == nullptr) {
        return nullptr;
    }

    // 将质数转换为 BIGNUM 类型的数据
    auto p = BN_bin2bn(dhp_2048, sizeof(dhp_2048), nullptr);
    // 使用二进制数据创建大数对象 g
    auto g = BN_bin2bn(dhg_2048, sizeof(dhg_2048), nullptr);

    // 检查 p 和 g 是否为空，以及设置 DH 对象的 p 和 g，如果有任何一个条件不满足，则释放内存并返回空指针
    if (p == nullptr || g == nullptr || !DH_set0_pqg(dh, p, nullptr, g)) {
        DH_free(dh); // 释放 DH 对象的内存
        BN_free(p); // 释放 p 的内存
        BN_free(g); // 释放 g 的内存

        return nullptr; // 返回空指针
    }

    return dh; // 返回 DH 对象
} // 结束命名空间 xmrig

} // 结束条件编译

xmrig::TlsContext::~TlsContext()
{
    // 释放 SSL 上下文
    SSL_CTX_free(m_ctx);
}

xmrig::TlsContext *xmrig::TlsContext::create(const TlsConfig &config)
{
    // 如果 TLS 配置未启用，则返回空指针
    if (!config.isEnabled()) {
        return nullptr;
    }

    // 创建 TlsContext 对象
    auto tls = new TlsContext();
    // 如果加载配置失败，则释放对象并返回空指针
    if (!tls->load(config)) {
        delete tls;
        return nullptr;
    }

    return tls;
}

bool xmrig::TlsContext::load(const TlsConfig &config)
{
    // 创建 SSL 上下文
    m_ctx = SSL_CTX_new(SSLv23_server_method());
    // 如果创建失败，则记录错误并返回 false
    if (m_ctx == nullptr) {
        LOG_ERR("Unable to create SSL context");
        return false;
    }

    // 展开证书路径并使用证书链文件
    const auto cert = Env::expand(config.cert());
    if (SSL_CTX_use_certificate_chain_file(m_ctx, cert) <= 0) {
        LOG_ERR("SSL_CTX_use_certificate_chain_file(\"%s\") failed.", config.cert());
        return false;
    }

    // 展开密钥路径并使用私钥文件
    const auto key = Env::expand(config.key());
    if (SSL_CTX_use_PrivateKey_file(m_ctx, key, SSL_FILETYPE_PEM) <= 0) {
        LOG_ERR("SSL_CTX_use_PrivateKey_file(\"%s\") failed.", config.key());
        return false;
    }

    // 设置 SSL 上下文选项
    SSL_CTX_set_options(m_ctx, SSL_OP_NO_SSLv2 | SSL_OP_NO_SSLv3);
    SSL_CTX_set_options(m_ctx, SSL_OP_CIPHER_SERVER_PREFERENCE);

#   if OPENSSL_VERSION_NUMBER >= 0x1010100fL && !defined(LIBRESSL_VERSION_NUMBER)
    // 设置最大早期数据大小为 0
    SSL_CTX_set_max_early_data(m_ctx, 0);
#   endif

    // 设置协议
    setProtocols(config.protocols());

    // 设置密码
    return setCiphers(config.ciphers()) && setCipherSuites(config.cipherSuites()) && setDH(config.dhparam());
}

bool xmrig::TlsContext::setCiphers(const char *ciphers)
{
    // 如果密码为空或设置成功，则返回 true
    if (ciphers == nullptr || SSL_CTX_set_cipher_list(m_ctx, ciphers) == 1) {
        return true;
    }

    // 记录错误并返回 true
    LOG_ERR("SSL_CTX_set_cipher_list(\"%s\") failed.", ciphers);
    return true;
}

bool xmrig::TlsContext::setCipherSuites(const char *ciphersuites)
{
    // 如果密码套件为空，则返回 true
    if (ciphersuites == nullptr) {
        return true;
    }

#   if OPENSSL_VERSION_NUMBER >= 0x1010100fL && !defined(LIBRESSL_VERSION_NUMBER)
    # 如果设置 SSL 上下文的密码套件成功，则返回 true
    if (SSL_CTX_set_ciphersuites(m_ctx, ciphersuites) == 1) {
        # 返回 true
        return true;
    }
#   endif
#   endif 语句的结束

    LOG_ERR("SSL_CTX_set_ciphersuites(\"%s\") failed.", ciphersuites);
    记录错误日志，指明 SSL_CTX_set_ciphersuites 函数调用失败，打印失败的密码套件

    return false;
    返回 false，表示函数执行失败
}


bool xmrig::TlsContext::setDH(const char *dhparam)
{
#   if OPENSSL_VERSION_NUMBER < 0x30000000L || defined(LIBRESSL_VERSION_NUMBER)
    如果 OpenSSL 版本号小于 3.0 或者定义了 LIBRESSL_VERSION_NUMBER

    DH *dh = nullptr;
    初始化 DH 指针为 nullptr

    if (dhparam != nullptr) {
        如果 dhparam 不为空
        BIO *bio = BIO_new_file(Env::expand(dhparam), "r");
        从文件中创建一个 BIO 对象，用于读取 DH 参数
        if (bio == nullptr) {
            如果创建 BIO 对象失败
            LOG_ERR("BIO_new_file(\"%s\") failed.", dhparam);
            记录错误日志，指明 BIO_new_file 函数调用失败，打印失败的文件名

            return false;
            返回 false，表示函数执行失败
        }

        dh = PEM_read_bio_DHparams(bio, nullptr, nullptr, nullptr);
        从 BIO 对象中读取 DH 参数
        if (dh == nullptr) {
            如果读取 DH 参数失败
            LOG_ERR("PEM_read_bio_DHparams(\"%s\") failed.", dhparam);
            记录错误日志，指明 PEM_read_bio_DHparams 函数调用失败，打印失败的文件名

            BIO_free(bio);
            释放 BIO 对象

            return false;
            返回 false，表示函数执行失败
        }

        BIO_free(bio);
        释放 BIO 对象
    }
    else {
        否则
        dh = get_dh2048();
        获取一个 2048 位的 DH 参数
    }

    const int rc = SSL_CTX_set_tmp_dh(m_ctx, dh); // NOLINT(cppcoreguidelines-pro-type-cstyle-cast)
    设置临时 DH 参数到 SSL 上下文中，NOLINT 用于禁止特定的 Lint 检查

    DH_free(dh);
    释放 DH 参数

    if (rc == 0) {
        如果设置失败
        LOG_ERR("SSL_CTX_set_tmp_dh(\"%s\") failed.", dhparam);
        记录错误日志，指明 SSL_CTX_set_tmp_dh 函数调用失败，打印失败的 DH 参数

        return false;
        返回 false，表示函数执行失败
    }
#   else
    否则
    if (dhparam != nullptr) {
        如果 dhparam 不为空
        EVP_PKEY *dh = nullptr;
        初始化 EVP_PKEY 指针为 nullptr
        BIO *bio     = BIO_new_file(Env::expand(dhparam), "r");
        从文件中创建一个 BIO 对象，用于读取 DH 参数

        if (bio) {
            如果创建 BIO 对象成功
            dh = PEM_read_bio_Parameters(bio, nullptr);
            从 BIO 对象中读取 DH 参数
            BIO_free(bio);
            释放 BIO 对象
        }

        if (!dh) {
            如果 DH 参数为空
            LOG_ERR("PEM_read_bio_Parameters(\"%s\") failed.", dhparam);
            记录错误日志，指明 PEM_read_bio_Parameters 函数调用失败，打印失败的文件名

            return false;
            返回 false，表示函数执行失败
        }

        if (SSL_CTX_set0_tmp_dh_pkey(m_ctx, dh) != 1) {
            如果设置临时 DH 参数到 SSL 上下文失败
            EVP_PKEY_free(dh);
            释放 DH 参数

            LOG_ERR("SSL_CTX_set0_tmp_dh_pkey(\"%s\") failed.", dhparam);
            记录错误日志，指明 SSL_CTX_set0_tmp_dh_pkey 函数调用失败，打印失败的 DH 参数

            return false;
            返回 false，表示函数执行失败
        }
    }
    else {
        否则
        SSL_CTX_set_dh_auto(m_ctx, 1);
        设置 SSL 上下文自动选择 DH 参数
    }
#   endif

    return true;
    返回 true，表示函数执行成功
}


void xmrig::TlsContext::setProtocols(uint32_t protocols)
{
    设置支持的协议

    if (protocols == 0) {
        如果协议为 0，表示不支持任何协议
        return;
        直接返回
    }

    if (!(protocols & TlsConfig::TLSv1)) {
        如果不支持 TLSv1 协议
        SSL_CTX_set_options(m_ctx, SSL_OP_NO_TLSv1);
        设置 SSL 上下文不支持 TLSv1 协议
    }

#   ifdef SSL_OP_NO_TLSv1_1
    如果定义了 SSL_OP_NO_TLSv1_1
    # 如果不支持 TLSv1.1 协议
    if (!(protocols & TlsConfig::TLSv1_1)) {
        # 设置 SSL 上下文选项，禁用 TLSv1.1 协议
        SSL_CTX_set_options(m_ctx, SSL_OP_NO_TLSv1_1);
    }
#   endif

#   ifdef SSL_OP_NO_TLSv1_2
    # 清除上下文中的 SSL_OP_NO_TLSv1_2 选项
    SSL_CTX_clear_options(m_ctx, SSL_OP_NO_TLSv1_2);
    # 如果不包含 TLSv1_2 协议，则设置上下文中的 SSL_OP_NO_TLSv1_2 选项
    if (!(protocols & TlsConfig::TLSv1_2)) {
        SSL_CTX_set_options(m_ctx, SSL_OP_NO_TLSv1_2);
    }
#   endif

#   ifdef SSL_OP_NO_TLSv1_3
    # 清除上下文中的 SSL_OP_NO_TLSv1_3 选项
    SSL_CTX_clear_options(m_ctx, SSL_OP_NO_TLSv1_3);
    # 如果不包含 TLSv1_3 协议，则设置上下文中的 SSL_OP_NO_TLSv1_3 选项
    if (!(protocols & TlsConfig::TLSv1_3)) {
        SSL_CTX_set_options(m_ctx, SSL_OP_NO_TLSv1_3);
    }
#   endif
}
```