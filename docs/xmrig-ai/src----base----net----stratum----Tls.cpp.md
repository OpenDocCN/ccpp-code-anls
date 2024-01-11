# `xmrig\src\base\net\stratum\Tls.cpp`

```
/* XMRig
 * 版权所有（C）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的版本为3或（在您的选择下）任何更高版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/stratum/Tls.h"  // 导入Tls类的头文件
#include "base/io/log/Log.h"  // 导入Log类的头文件
#include "base/net/stratum/Client.h"  // 导入Client类的头文件
#include "base/tools/Cvt.h"  // 导入Cvt类的头文件

#ifdef _MSC_VER
#   define strncasecmp(x,y,z) _strnicmp(x,y,z)  // 如果是MSC编译器，则定义strncasecmp为_strnicmp
#endif

#include <cassert>  // 导入断言头文件
#include <openssl/ssl.h>  // 导入OpenSSL的SSL头文件

// 定义Tls类的构造函数，传入一个Client对象指针
xmrig::Client::Tls::Tls(Client *client) :
    m_client(client)  // 初始化成员变量m_client
{
    m_ctx = SSL_CTX_new(SSLv23_method());  // 创建一个SSL上下文对象
    assert(m_ctx != nullptr);  // 断言SSL上下文对象不为空

    if (!m_ctx) {  // 如果SSL上下文对象为空
        return;  // 返回
    }

    m_write = BIO_new(BIO_s_mem());  // 创建一个内存BIO对象用于写入
    m_read  = BIO_new(BIO_s_mem());  // 创建一个内存BIO对象用于读取
    SSL_CTX_set_options(m_ctx, SSL_OP_NO_SSLv2 | SSL_OP_NO_SSLv3);  // 设置SSL上下文对象的选项，禁用SSLv2和SSLv3
}

// 定义Tls类的析构函数
xmrig::Client::Tls::~Tls()
{
    if (m_ctx) {  // 如果SSL上下文对象不为空
        SSL_CTX_free(m_ctx);  // 释放SSL上下文对象
    }

    if (m_ssl) {  // 如果SSL对象不为空
        SSL_free(m_ssl);  // 释放SSL对象
    }
}

// 定义Tls类的握手函数，传入服务器名
bool xmrig::Client::Tls::handshake(const char* servername)
{
    m_ssl = SSL_new(m_ctx);  // 创建一个SSL对象
    assert(m_ssl != nullptr);  // 断言SSL对象不为空

    if (!m_ssl) {  // 如果SSL对象为空
        return false;  // 返回false
    }

    if (servername) {  // 如果服务器名不为空
        SSL_set_tlsext_host_name(m_ssl, servername);  // 设置TLS扩展的服务器名
    }

    SSL_set_connect_state(m_ssl);  // 设置SSL对象的连接状态
    SSL_set_bio(m_ssl, m_read, m_write);  // 设置SSL对象的BIO
    SSL_do_handshake(m_ssl);  // 进行SSL握手

    return send();  // 返回send函数的结果
}
bool xmrig::Client::Tls::send(const char *data, size_t size)
{
    // 使用 SSL_write 函数发送数据
    SSL_write(m_ssl, data, size);

    // 调用 send 函数发送数据
    return send();
}


const char *xmrig::Client::Tls::fingerprint() const
{
    // 如果连接准备就绪，返回指纹信息，否则返回空指针
    return m_ready ? m_fingerprint : nullptr;
}


const char *xmrig::Client::Tls::version() const
{
    // 如果连接准备就绪，返回 SSL 协议版本，否则返回空指针
    return m_ready ? SSL_get_version(m_ssl) : nullptr;
}


void xmrig::Client::Tls::read(const char *data, size_t size)
{
    // 将数据写入读取的 BIO 对象
    BIO_write(m_read, data, size);

    // 如果 SSL 连接未完成初始化
    if (!SSL_is_init_finished(m_ssl)) {
        // 尝试完成 SSL 连接初始化
        const int rc = SSL_connect(m_ssl);

        // 如果连接需要继续读取数据
        if (rc < 0 && SSL_get_error(m_ssl, rc) == SSL_ERROR_WANT_READ) {
            // 调用 send 函数发送数据
            send();
        } else if (rc == 1) {
            // 获取对等方的证书
            X509 *cert = SSL_get_peer_certificate(m_ssl);
            // 如果证书验证失败
            if (!verify(cert)) {
                X509_free(cert);
                m_client->close();

                return;
            }

            X509_free(cert);
            m_ready = true;
            // 调用客户端的登录函数
            m_client->login();
        }

        return;
    }

    // 创建一个缓冲区
    static char buf[16384]{};
    int bytes_read = 0;

    // 循环读取 SSL 数据
    while ((bytes_read = SSL_read(m_ssl, buf, sizeof(buf))) > 0) {
        // 解析读取的数据
        m_client->m_reader.parse(buf, static_cast<size_t>(bytes_read));
    }
}


bool xmrig::Client::Tls::send()
{
    // 调用客户端的发送函数
    return m_client->send(m_write);
}


bool xmrig::Client::Tls::verify(X509 *cert)
{
    // 如果证书为空，记录错误并返回验证失败
    if (cert == nullptr) {
        LOG_ERR("[%s] Failed to get server certificate", m_client->url());

        return false;
    }

    // 如果指纹验证失败
    if (!verifyFingerprint(cert)) {
        LOG_ERR("[%s] Failed to verify server certificate fingerprint", m_client->url());

        const char *fingerprint = m_client->m_pool.fingerprint();
        // 如果指纹长度为 64 并且不为空
        if (strlen(m_fingerprint) == 64 && fingerprint != nullptr) {
            LOG_ERR("\"%s\" was given", m_fingerprint);
            LOG_ERR("\"%s\" was configured", fingerprint);
        }

        return false;
    }

    return true;
}


bool xmrig::Client::Tls::verifyFingerprint(X509 *cert)
{
    // 获取 SHA256 摘要算法
    const EVP_MD *digest = EVP_get_digestbyname("sha256");
}
    # 如果摘要为空指针，则返回 false
    if (digest == nullptr) {
        return false;
    }

    # 定义一个数组来存储消息摘要，以及一个变量来存储摘要的长度
    unsigned char md[EVP_MAX_MD_SIZE];
    unsigned int dlen = 0;

    # 使用 X509 证书对象和指定的摘要算法计算证书的摘要，如果计算失败则返回 false
    if (X509_digest(cert, digest, md, &dlen) != 1) {
        return false;
    }

    # 将摘要转换成十六进制格式的字符串
    Cvt::toHex(m_fingerprint, sizeof(m_fingerprint), md, 32);
    # 获取客户端对象中存储的指纹信息
    const char *fingerprint = m_client->m_pool.fingerprint();

    # 如果指纹信息为空指针，或者计算得到的指纹信息与存储的指纹信息相同，则返回 true，否则返回 false
    return fingerprint == nullptr || strncasecmp(m_fingerprint, fingerprint, 64) == 0;
# 闭合前面的函数定义
```