# `xmrig\src\base\net\https\HttpsServer.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多细节请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <uv.h>


#include "base/net/https/HttpsServer.h"
#include "base/net/https/HttpsContext.h"
#include "base/net/tls/TlsContext.h"
#include "base/net/tools/NetBuffer.h"


// 使用给定的 IHttpListener 共享指针初始化 HttpsServer
xmrig::HttpsServer::HttpsServer(const std::shared_ptr<IHttpListener> &listener) :
    m_listener(listener)
{
}


// HttpsServer 析构函数
xmrig::HttpsServer::~HttpsServer()
{
    // 关闭所有的 HttpContext
    HttpContext::closeAll();

    // 释放 m_tls 指针指向的内存
    delete m_tls;
}


// 设置 TlsContext 的配置
bool xmrig::HttpsServer::setTls(const TlsConfig &config)
{
    // 创建 TlsContext 对象
    m_tls = TlsContext::create(config);

    // 返回是否成功创建了 TlsContext 对象
    return m_tls != nullptr;
}


// 处理连接事件
void xmrig::HttpsServer::onConnection(uv_stream_t *stream, uint16_t)
{
    // 创建新的 HttpsContext 对象
    auto ctx = new HttpsContext(m_tls, m_listener);
    // 接受连接
    uv_accept(stream, ctx->stream());

    // 开始读取数据
    uv_read_start(ctx->stream(), NetBuffer::onAlloc, onRead); // NOLINT(clang-analyzer-cplusplus.NewDeleteLeaks)
}


// 处理读取事件
void xmrig::HttpsServer::onRead(uv_stream_t *stream, ssize_t nread, const uv_buf_t *buf)
{
    // 获取 HttpsContext 对象
    auto ctx = static_cast<HttpsContext*>(stream->data);
    if (nread >= 0) {
        // 将读取的数据追加到上下文中
        ctx->append(buf->base, static_cast<size_t>(nread));
    }
    else {
        // 关闭上下文
        ctx->close();
    }

    // 释放缓冲区
    NetBuffer::release(buf);
}
```