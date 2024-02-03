# `xmrig\src\base\net\tools\TcpServer.h`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 * 其发布由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有适销性或特定用途的暗示担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_TCPSERVER_H
#define XMRIG_TCPSERVER_H

#include <uv.h>
#include "base/tools/Object.h"

namespace xmrig {

class ITcpServerListener;
class String;

class TcpServer
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(TcpServer)

    // 构造函数，传入主机名、端口号和监听器对象
    TcpServer(const String &host, uint16_t port, ITcpServerListener *listener);
    // 析构函数
    ~TcpServer();
    // 绑定函数
    int bind();

private:
    // 创建函数，传入流对象和状态
    void create(uv_stream_t *stream, int status);
    // 静态连接函数，传入流对象和状态
    static void onConnection(uv_stream_t *stream, int status);

    // 主机名
    const String &m_host;
    // 版本号
    int m_version   = 0;
    // 监听器对象指针
    ITcpServerListener *m_listener;
    // 地址存储
    sockaddr_storage m_addr{};
    // 端口号
    uint16_t m_port;
    // TCP 对象指针
    uv_tcp_t *m_tcp;
};

} /* namespace xmrig */

#endif /* XMRIG_TCPSERVER_H */
```