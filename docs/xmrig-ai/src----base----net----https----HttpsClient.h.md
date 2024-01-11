# `xmrig\src\base\net\https\HttpsClient.h`

```
/* XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它
 *   由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的保证。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#ifndef XMRIG_HTTPSCLIENT_H
#define XMRIG_HTTPSCLIENT_H


using BIO       = struct bio_st;
using SSL_CTX   = struct ssl_ctx_st;
using SSL       = struct ssl_st;
using X509      = struct x509_st;


#include "base/net/http/HttpClient.h"
#include "base/tools/String.h"


namespace xmrig {


class HttpsClient : public HttpClient
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HttpsClient)

    // 构造函数，接受标签、请求和监听器的弱引用
    HttpsClient(const char *tag, FetchRequest &&req, const std::weak_ptr<IHttpListener> &listener);
    // 析构函数
    ~HttpsClient() override;

    // 返回 TLS 指纹
    const char *tlsFingerprint() const override;
    // 返回 TLS 版本
    const char *tlsVersion() const override;

protected:
    // 握手方法
    void handshake() override;
    // 读取数据方法
    void read(const char *data, size_t size) override;

private:
    // 写入数据方法
    void write(std::string &&data, bool close) override;

    // 验证证书
    bool verify(X509 *cert);
    // 验证指纹
    bool verifyFingerprint(X509 *cert);
    // 刷新方法
    void flush(bool close);

    // 读取 BIO 对象
    BIO *m_read                         = nullptr;
    // 写入 BIO 对象
    BIO *m_write                        = nullptr;
    // 就绪状态
    bool m_ready                        = false;
    // 指纹数组
    char m_fingerprint[32 * 2 + 8]{};
    # 创建一个指向 SSL 结构的指针，并初始化为 nullptr
    SSL *m_ssl                          = nullptr;
    # 创建一个指向 SSL_CTX 结构的指针，并初始化为 nullptr
    SSL_CTX *m_ctx                      = nullptr;
}; 
// 结束命名空间 xmrig

} // namespace xmrig
// 结束命名空间 xmrig

#endif // XMRIG_HTTPSCLIENT_H
// 结束条件编译指令，检查是否定义了 XMRIG_HTTPSCLIENT_H，如果定义了则包含该头文件，否则跳过
```