# `xmrig\src\base\net\http\HttpResponse.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发和/或修改它，
 *   其发布由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，但没有任何担保；甚至没有适销性或特定用途的隐含担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本，如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "base/net/http/HttpResponse.h"  // 包含 HttpResponse 类的头文件
#include "3rdparty/llhttp/llhttp.h"       // 包含 llhttp 库的头文件
#include "base/io/log/Log.h"              // 包含 Log 类的头文件
#include "base/net/http/HttpContext.h"    // 包含 HttpContext 类的头文件

#include <cinttypes>                     // 包含 cinttypes 库
#include <cstring>                       // 包含 cstring 库
#include <sstream>                       // 包含 stringstream 库
#include <uv.h>                          // 包含 uv.h 头文件

namespace xmrig {

static const char *kCRLF      = "\r\n";  // 定义换行符常量
static const char *kUserAgent = "user-agent";  // 定义用户代理常量

} // namespace xmrig

// 构造函数，初始化 HttpResponse 对象的 id 和 statusCode 成员变量
xmrig::HttpResponse::HttpResponse(uint64_t id, int statusCode) :
    m_id(id),
    m_statusCode(statusCode)
{
}

// 判断 HttpResponse 对象是否存活
bool xmrig::HttpResponse::isAlive() const
{
    auto ctx = HttpContext::get(m_id);  // 获取与 id 对应的 HttpContext 对象

    return ctx && uv_is_writable(ctx->stream());  // 返回是否可写
}

// 结束响应，发送数据
void xmrig::HttpResponse::end(const char *data, size_t size)
{
    if (!isAlive()) {  // 如果不存活，直接返回
        return;
    }

    if (data && !size) {  // 如果有数据但大小为0，计算数据大小
        size = strlen(data);
    }

    if (size) {  // 如果有数据，设置 Content-Length 头部字段
        setHeader("Content-Length", std::to_string(size));
    }

    setHeader("Connection", "close");  // 设置 Connection 头部字段为 close

    std::stringstream ss;  // 创建字符串流对象
    ss << "HTTP/1.1 " << statusCode() << " " << HttpData::statusName(statusCode()) << kCRLF;  // 构建响应行
    // 遍历 m_headers 容器中的每个元素，将 header.first 和 header.second 拼接成字符串，添加到 ss 中
    for (auto &header : m_headers) {
        ss << header.first << ": " << header.second << kCRLF;
    }

    // 在 ss 中添加一个空行
    ss << kCRLF;

    // 获取当前 HTTP 上下文的指针
    auto ctx         = HttpContext::get(m_id);
    // 如果 data 不为空，则将 ss 中的内容和 data 拼接成字符串，否则将 ss 的内容赋值给 body
    std::string body = data ? (ss.str() + std::string(data, size)) : ss.str();
#   ifndef APP_DEBUG
    # 如果不是调试模式
    if (statusCode() >= 400)
#   endif
    # 如果状态码大于等于400
    {
        # 定义一个布尔变量，表示状态码是否大于等于400
        const bool err = statusCode() >= 400;

        # 根据错误状态选择不同的日志级别，打印请求信息
        Log::print(err ? Log::ERR : Log::INFO, CYAN("%s ") CLEAR MAGENTA_BOLD("%s") WHITE_BOLD(" %s ") CSI "1;%dm%d " CLEAR WHITE_BOLD("%zu ") CYAN_BOLD("%" PRIu64 "ms ") BLACK_BOLD("\"%s\""),
                   ctx->ip().c_str(),
                   llhttp_method_name(static_cast<llhttp_method>(ctx->method)),
                   ctx->url.c_str(),
                   err ? 31 : 32,
                   statusCode(),
                   body.size(),
                   ctx->elapsed(),
                   ctx->headers.count(kUserAgent) ? ctx->headers.at(kUserAgent).c_str() : nullptr
                   );
    }

    # 将响应体写入到上下文中
    ctx->write(std::move(body), true);
}
```