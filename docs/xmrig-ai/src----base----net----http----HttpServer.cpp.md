# `xmrig\src\base\net\http\HttpServer.cpp`

```cpp
/* XMRig
 * 版权所有（c）2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多详情请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <functional>
#include <uv.h>


#include "base/net/http/HttpServer.h"
#include "3rdparty/llhttp/llhttp.h"
#include "base/net/http/HttpContext.h"
#include "base/net/tools/NetBuffer.h"


// 使用xmrig命名空间
xmrig::HttpServer::HttpServer(const std::shared_ptr<IHttpListener> &listener) :
    m_listener(listener)
{
}


// 析构函数
xmrig::HttpServer::~HttpServer()
{
    // 关闭所有的HttpContext
    HttpContext::closeAll();
}


// 处理连接
void xmrig::HttpServer::onConnection(uv_stream_t *stream, uint16_t)
{
    // 创建一个新的HttpContext对象
    auto ctx = new HttpContext(HTTP_REQUEST, m_listener);
    // 接受连接
    uv_accept(stream, ctx->stream());

    // 开始读取数据
    uv_read_start(ctx->stream(), NetBuffer::onAlloc,
        [](uv_stream_t *tcp, ssize_t nread, const uv_buf_t *buf)
        {
            auto ctx = static_cast<HttpContext*>(tcp->data);

            // 如果读取失败或者无法解析数据，则关闭连接
            if (nread < 0 || !ctx->parse(buf->base, static_cast<size_t>(nread))) {
                ctx->close();
            }

            // 释放缓冲区
            NetBuffer::release(buf);
        });
}
```