# `xmrig\src\base\net\https\HttpsContext.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/https/HttpsContext.h"
#include "3rdparty/llhttp/llhttp.h"
#include "base/net/tls/TlsContext.h"


#include <openssl/bio.h>
#include <uv.h>


// 使用TlsContext和IHttpListener的弱引用初始化HttpsContext
xmrig::HttpsContext::HttpsContext(TlsContext *tls, const std::weak_ptr<IHttpListener> &listener) :
    HttpContext(HTTP_REQUEST, listener),
    ServerTls(tls ? tls->ctx() : nullptr)
{
    // 如果没有传入TlsContext，则将模式设置为TLS_OFF
    if (!tls) {
        m_mode = TLS_OFF;
    }
}


// 默认析构函数
xmrig::HttpsContext::~HttpsContext() = default;


// 向数据中追加内容
void xmrig::HttpsContext::append(char *data, size_t size)
{
    // 如果模式为TLS_AUTO，则根据数据判断是否开启TLS
    if (m_mode == TLS_AUTO) {
        m_mode = isTLS(data, size) ? TLS_ON : TLS_OFF;
    }

    // 如果模式为TLS_ON，则读取数据
    if (m_mode == TLS_ON) {
        read(data, size);
    }
    // 否则解析数据
    else {
        parse(data, size);
    }
}


// 写入数据到BIO
bool xmrig::HttpsContext::write(BIO *bio)
{
    // 如果流可写，则继续执行
    if (uv_is_writable(stream()) != 1) {
        return false;
    }

    char *data        = nullptr;
    const size_t size = BIO_get_mem_data(bio, &data); // NOLINT(cppcoreguidelines-pro-type-cstyle-cast)
    std::string body(data, size);

    (void) BIO_reset(bio);

    // 调用HttpContext的write方法，并传入数据和关闭标志
    HttpContext::write(std::move(body), m_close);

    return true;
}


// 解析数据
void xmrig::HttpsContext::parse(char *data, size_t size)
{
    // 解析数据的具体实现
    # 如果HTTP上下文无法解析给定的数据和大小
    if (!HttpContext::parse(data, size)) {
        # 关闭连接
        close();
    }
# 关闭 HTTPS 上下文
void xmrig::HttpsContext::shutdown()
{
    # 调用 close() 方法关闭 HTTPS 上下文
    close();
}

# 写入数据到 HTTPS 上下文
void xmrig::HttpsContext::write(std::string &&data, bool close)
{
    # 设置是否关闭标志位
    m_close = close;

    # 如果模式为 TLS_ON，则调用 send() 方法发送数据
    if (m_mode == TLS_ON) {
        send(data.data(), data.size());
    }
    # 否则，调用 HttpContext 类的 write() 方法写入数据
    else {
        HttpContext::write(std::move(data), close);
    }
}
```