# `xmrig\src\base\net\tools\TcpServer.cpp`

```cpp
/* XMRig
 * 版权声明
 * 该程序是自由软件：您可以重新发布或修改它，遵循 GNU 通用公共许可证的条款，可以选择遵循许可证的第三版或之后的版本。
 * 该程序是基于希望它能有用而发布的，但没有任何担保；甚至没有暗示的担保适用于特定用途。更多详情请参阅 GNU 通用公共许可证。
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请访问 http://www.gnu.org/licenses/。
 */

#include "base/kernel/interfaces/ITcpServerListener.h"
#include "base/net/tools/TcpServer.h"
#include "base/tools/Handle.h"
#include "base/tools/String.h"

// 定义本地主机地址
static const xmrig::String kLocalHost("127.0.0.1");

// TCP 服务器类的构造函数
xmrig::TcpServer::TcpServer(const String &host, uint16_t port, ITcpServerListener *listener) :
    // 如果传入的主机地址为空，则使用本地主机地址
    m_host(host.isNull() ? kLocalHost : host),
    // 设置监听器
    m_listener(listener),
    // 设置端口号
    m_port(port)
{
    // 断言监听器不为空
    assert(m_listener != nullptr);

    // 初始化 TCP 对象
    m_tcp = new uv_tcp_t;
    uv_tcp_init(uv_default_loop(), m_tcp);
    // 将 TCP 对象与当前对象关联
    m_tcp->data = this;

    // 设置 TCP 无延迟
    uv_tcp_nodelay(m_tcp, 1);

    // 如果主机地址包含 "："并且成功解析为 IPv6 地址，则设置版本为 6
    if (m_host.contains(":") && uv_ip6_addr(m_host.data(), m_port, reinterpret_cast<sockaddr_in6 *>(&m_addr)) == 0) {
        m_version = 6;
    }
}
    # 如果 m_host 是一个 IPv4 地址，并且端口号有效，则执行以下代码
    else if (uv_ip4_addr(m_host.data(), m_port, reinterpret_cast<sockaddr_in *>(&m_addr)) == 0) {
        # 将 m_version 设置为 4，表示使用 IPv4
        m_version = 4;
    }
# TcpServer 类的析构函数
xmrig::TcpServer::~TcpServer()
{
    # 关闭 TCP 句柄
    Handle::close(m_tcp);
}


# TcpServer 类的绑定函数
int xmrig::TcpServer::bind()
{
    # 如果版本号为空，则返回地址族错误
    if (!m_version) {
        return UV_EAI_ADDRFAMILY;
    }

    # 将 TCP 句柄绑定到指定地址
    uv_tcp_bind(m_tcp, reinterpret_cast<const sockaddr*>(&m_addr), 0);

    # 开始监听连接，最大连接数为 511
    const int rc = uv_listen(reinterpret_cast<uv_stream_t*>(m_tcp), 511, TcpServer::onConnection);
    if (rc != 0) {
        return rc;
    }

    # 如果端口号为空，则获取绑定的端口号
    if (!m_port) {
        sockaddr_storage storage = {};
        int size = sizeof(storage);

        uv_tcp_getsockname(m_tcp, reinterpret_cast<sockaddr*>(&storage), &size);

        m_port = ntohs(reinterpret_cast<sockaddr_in *>(&storage)->sin_port);
    }

    # 返回端口号
    return m_port;
}


# 创建 TCP 连接
void xmrig::TcpServer::create(uv_stream_t *stream, int status)
{
    # 如果状态小于 0，则返回
    if (status < 0) {
        return;
    }

    # 调用监听器的连接函数
    m_listener->onConnection(stream, m_port);
}


# 处理连接事件
void xmrig::TcpServer::onConnection(uv_stream_t *stream, int status)
{
    # 调用 create 函数
    static_cast<TcpServer *>(stream->data)->create(stream, status);
}
```