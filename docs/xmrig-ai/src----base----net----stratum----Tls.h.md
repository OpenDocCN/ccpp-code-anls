# `xmrig\src\base\net\stratum\Tls.h`

```
/* XMRig
 * 版权所有（C）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证。
 * 有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CLIENT_TLS_H
#define XMRIG_CLIENT_TLS_H

使用BIO       = 结构体bio_st;
使用SSL       = 结构体ssl_st;
使用SSL_CTX   = 结构体ssl_ctx_st;
使用X509      = 结构体x509_st;

包括 "base/net/stratum/Client.h"
包括 "base/tools/Object.h"

在xmrig命名空间中

类Client::Tls
{
public:
    禁用默认的复制和移动构造函数
    Tls(Client *client); // Tls构造函数，接受一个Client指针作为参数
    ~Tls(); // Tls析构函数

    握手函数，接受服务器名称作为参数
    bool handshake(const char* servername);
    发送函数，接受数据和大小作为参数
    bool send(const char *data, size_t size);
    返回指纹的函数
    const char *fingerprint() const;
    返回版本的函数
    const char *version() const;
    读取函数，接受数据和大小作为参数
    void read(const char *data, size_t size);

private:
    发送函数
    bool send();
    验证函数，接受证书作为参数
    bool verify(X509 *cert);
    验证指纹函数，接受证书作为参数
    bool verifyFingerprint(X509 *cert);

    BIO *m_read     = nullptr; // 读取BIO对象指针初始化为nullptr
    BIO *m_write    = nullptr; // 写入BIO对象指针初始化为nullptr
    准备状态初始化为false
    bool m_ready    = false;
    指纹数组初始化为空
    char m_fingerprint[32 * 2 + 8]{};
    Client *m_client; // Client对象指针
    SSL *m_ssl      = nullptr; // SSL对象指针初始化为nullptr
    SSL_CTX *m_ctx; // SSL_CTX对象指针
};

在xmrig命名空间中

#endif /* XMRIG_CLIENT_TLS_H */
```