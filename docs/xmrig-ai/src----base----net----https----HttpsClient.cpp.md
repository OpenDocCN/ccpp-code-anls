# `xmrig\src\base\net\https\HttpsClient.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 * 由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本，如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <cassert>
#include <openssl/ssl.h>
#include <uv.h>

#include "base/net/https/HttpsClient.h"
#include "base/io/log/Log.h"
#include "base/tools/Cvt.h"

#ifdef _MSC_VER
#   define strncasecmp(x,y,z) _strnicmp(x,y,z)
#endif

// 使用命名空间xmrig
xmrig::HttpsClient::HttpsClient(const char *tag, FetchRequest &&req, const std::weak_ptr<IHttpListener> &listener) :
    HttpClient(tag, std::move(req), listener)
{
    // 创建SSL上下文
    m_ctx = SSL_CTX_new(SSLv23_method());
    assert(m_ctx != nullptr);

    // 如果SSL上下文创建失败，则直接返回
    if (!m_ctx) {
        return;
    }

    // 创建写入和读取的BIO对象
    m_write = BIO_new(BIO_s_mem());
    m_read  = BIO_new(BIO_s_mem());
    // 设置SSL上下文选项
    SSL_CTX_set_options(m_ctx, SSL_OP_NO_SSLv2 | SSL_OP_NO_SSLv3);
}

// 析构函数
xmrig::HttpsClient::~HttpsClient()
{
    // 释放SSL上下文
    if (m_ctx) {
        SSL_CTX_free(m_ctx);
    }

    // 释放SSL对象
    if (m_ssl) {
        SSL_free(m_ssl);
    }
}

// 返回TLS指纹
const char *xmrig::HttpsClient::tlsFingerprint() const
{
    return m_ready ? m_fingerprint : nullptr;
}

// 返回TLS版本
const char *xmrig::HttpsClient::tlsVersion() const
{
    return m_ready ? SSL_get_version(m_ssl) : nullptr;
}

// 握手
void xmrig::HttpsClient::handshake()
{
    // 创建SSL对象
    m_ssl = SSL_new(m_ctx);
    # 断言 m_ssl 不为空指针
    assert(m_ssl != nullptr);

    # 如果 m_ssl 为空，则直接返回
    if (!m_ssl) {
        return;
    }

    # 设置 SSL 连接状态为连接状态
    SSL_set_connect_state(m_ssl);
    # 设置 SSL 对象的读写 BIO
    SSL_set_bio(m_ssl, m_read, m_write);
    # 设置 TLS 扩展的主机名
    SSL_set_tlsext_host_name(m_ssl, host());

    # 进行 SSL 握手
    SSL_do_handshake(m_ssl);

    # 刷新缓冲区，参数为 false
    flush(false);
}

// 从给定数据中读取内容
void xmrig::HttpsClient::read(const char *data, size_t size)
{
    // 将数据写入到读取的 BIO 中
    BIO_write(m_read, data, size);

    // 如果 SSL 连接还未完成
    if (!SSL_is_init_finished(m_ssl)) {
        // 尝试完成 SSL 连接
        const int rc = SSL_connect(m_ssl);

        // 如果连接失败且需要读取更多数据
        if (rc < 0 && SSL_get_error(m_ssl, rc) == SSL_ERROR_WANT_READ) {
            // 刷新数据，但不关闭连接
            flush(false);
        } 
        // 如果连接成功
        else if (rc == 1) {
            // 获取对方证书
            X509 *cert = SSL_get_peer_certificate(m_ssl);
            // 验证证书
            if (!verify(cert)) {
                X509_free(cert);
                // 关闭连接并返回协议错误
                return close(UV_EPROTO);
            }

            X509_free(cert);
            // 标记连接已准备就绪
            m_ready = true;

            // 执行握手
            HttpClient::handshake();
        }

        return;
    }

    // 读取 SSL 连接中的数据
    static char buf[16384]{};
    int rc = 0;
    while ((rc = SSL_read(m_ssl, buf, sizeof(buf))) > 0) {
        // 将读取的数据传递给 HttpClient::read 方法处理
        HttpClient::read(buf, static_cast<size_t>(rc));
    }

    // 如果读取到了 EOF
    if (rc == 0) {
        // 关闭连接并返回 EOF 错误
        close(UV_EOF);
    }
}

// 向服务器端写入数据
void xmrig::HttpsClient::write(std::string &&data, bool close)
{
    // 移动数据并写入 SSL 连接
    const std::string body = std::move(data);
    SSL_write(m_ssl, body.data(), body.size());

    // 刷新数据并根据参数决定是否关闭连接
    flush(close);
}

// 验证服务器端证书
bool xmrig::HttpsClient::verify(X509 *cert)
{
    // 如果证书为空，则验证失败
    if (cert == nullptr) {
        return false;
    }

    // 验证证书指纹
    if (!verifyFingerprint(cert)) {
        // 如果验证失败，记录错误信息并返回失败
        if (!isQuiet()) {
            LOG_ERR("[%s:%d] Failed to verify server certificate fingerprint", host(), port());

            if (strlen(m_fingerprint) == 64 && !req().fingerprint.isNull()) {
                LOG_ERR("\"%s\" was given", m_fingerprint);
                LOG_ERR("\"%s\" was configured", req().fingerprint.data());
            }
        }

        return false;
    }

    return true;
}

// 验证证书指纹
bool xmrig::HttpsClient::verifyFingerprint(X509 *cert)
{
    // 获取 SHA256 摘要算法
    const EVP_MD *digest = EVP_get_digestbyname("sha256");
    if (digest == nullptr) {
        return false;
    }

    // 计算证书的摘要
    unsigned char md[EVP_MAX_MD_SIZE];
    unsigned int dlen = 0;
    if (X509_digest(cert, digest, md, &dlen) != 1) {
        return false;
    }

    // 将摘要转换为十六进制格式的字符串
    Cvt::toHex(m_fingerprint, sizeof(m_fingerprint), md, 32);
}
    # 返回请求的指纹是否为空，或者请求的指纹与给定的指纹相同
    return req().fingerprint.isNull() || strncasecmp(m_fingerprint, req().fingerprint.data(), 64) == 0;
// 刷新 HTTPS 客户端，如果需要关闭则关闭
void xmrig::HttpsClient::flush(bool close)
{
    // 检查流是否可写，如果不可写则返回
    if (uv_is_writable(stream()) != 1) {
        return;
    }

    // 初始化数据指针为空
    char *data        = nullptr;
    // 获取写入内存中的数据和大小
    const size_t size = BIO_get_mem_data(m_write, &data); // NOLINT(cppcoreguidelines-pro-type-cstyle-cast)
    // 根据数据和大小创建字符串
    std::string body(data, size);

    // 重置写入内存
    (void) BIO_reset(m_write);

    // 调用 HttpContext 的 write 方法，传入数据和是否关闭的标志
    HttpContext::write(std::move(body), close);
}
```