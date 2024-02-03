# `xmrig\src\base\net\https\HttpsServer.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   许可证的第3版或（根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HTTPSSERVER_H
#define XMRIG_HTTPSSERVER_H


使用 uv_tcp_t  = struct uv_tcp_s;

struct http_parser;
struct http_parser_settings;
struct uv_buf_t;


#include "base/kernel/interfaces/ITcpServerListener.h"
#include "base/tools/Object.h"


#include <memory>


namespace xmrig {


class IHttpListener;
class TlsContext;
class TlsConfig;


class HttpsServer : public ITcpServerListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HttpsServer)

    // 构造函数，接受一个 IHttpListener 类型的共享指针作为参数
    HttpsServer(const std::shared_ptr<IHttpListener> &listener);
    // 析构函数
    ~HttpsServer() override;

    // 设置 TLS 配置
    bool setTls(const TlsConfig &config);

protected:
    // 当有连接时触发的回调函数
    void onConnection(uv_stream_t *stream, uint16_t port) override;

private:
    // 当有数据可读时触发的回调函数
    static void onRead(uv_stream_t *stream, ssize_t nread, const uv_buf_t *buf);

    // 弱引用指向 IHttpListener 类型的共享指针
    std::weak_ptr<IHttpListener> m_listener;
    // 指向 TlsContext 类型的指针
    TlsContext *m_tls   = nullptr;
};


} // namespace xmrig


#endif // XMRIG_HTTPSSERVER_H
```