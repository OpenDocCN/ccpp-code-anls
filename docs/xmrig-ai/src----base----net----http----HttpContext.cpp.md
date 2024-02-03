# `xmrig\src\base\net\http\HttpContext.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，无论是许可证的第 3 版还是
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用的分发，但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "base/net/http/HttpContext.h"  // 导入 HttpContext 类的头文件
#include "3rdparty/llhttp/llhttp.h"     // 导入 llhttp 库的头文件
#include "base/kernel/interfaces/IHttpListener.h"  // 导入 IHttpListener 接口的头文件
#include "base/tools/Baton.h"           // 导入 Baton 类的头文件
#include "base/tools/Chrono.h"          // 导入 Chrono 类的头文件

#include <algorithm>                   // 导入算法库
#include <uv.h>                        // 导入 libuv 库

namespace xmrig {

static llhttp_settings_t http_settings;  // 定义 llhttp_settings_t 类型的静态变量 http_settings
static std::map<uint64_t, HttpContext *> storage;  // 定义存储 HttpContext 指针的静态映射 storage
static uint64_t SEQUENCE = 0;  // 定义静态变量 SEQUENCE，并初始化为 0

class HttpWriteBaton : public Baton<uv_write_t>  // 定义 HttpWriteBaton 类，继承自 Baton<uv_write_t>
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HttpWriteBaton)  // 禁用默认的拷贝和移动构造函数

    inline HttpWriteBaton(std::string &&body, HttpContext *ctx) :  // HttpWriteBaton 类的构造函数，接受字符串和 HttpContext 指针作为参数
        m_ctx(ctx),  // 初始化 m_ctx 成员变量
        m_body(std::move(body))  // 初始化 m_body 成员变量
    {
        m_buf = uv_buf_init(&m_body.front(), m_body.size());  // 使用 m_body 的数据初始化 m_buf
    }

    inline ~HttpWriteBaton()  // HttpWriteBaton 类的析构函数
    {
        if (m_ctx) {  // 如果 m_ctx 不为空
            m_ctx->close();  // 调用 m_ctx 的 close 方法
        }
    }

    void write(uv_stream_t *stream)  // 写入方法，接受 uv_stream_t 指针作为参数
    {
        uv_write(&req, stream, &m_buf, 1, [](uv_write_t *req, int) { delete reinterpret_cast<HttpWriteBaton *>(req->data); });  // 调用 libuv 的写入方法，并在写入完成后删除 HttpWriteBaton 对象
    }

private:
    HttpContext *m_ctx;  // HttpContext 指针成员变量
    std::string m_body;  // 字符串成员变量
    uv_buf_t m_buf{};  // uv_buf_t 类型成员变量
};

} // namespace xmrig
// 构造函数，初始化 HttpContext 对象
xmrig::HttpContext::HttpContext(int parser_type, const std::weak_ptr<IHttpListener> &listener) :
    // 调用基类构造函数，设置序列号
    HttpData(SEQUENCE++),
    // 获取当前时间戳
    m_timestamp(Chrono::steadyMSecs()),
    // 设置监听器
    m_listener(listener)
{
    // 将当前对象存储到 storage 中
    storage[id()] = this;

    // 创建 llhttp_t 和 uv_tcp_t 对象
    m_parser = new llhttp_t;
    m_tcp    = new uv_tcp_t;

    // 初始化 TCP 对象
    uv_tcp_init(uv_default_loop(), m_tcp);
    // 设置 TCP 为无延迟模式
    uv_tcp_nodelay(m_tcp, 1);

    // 初始化 HTTP 解析器
    llhttp_init(m_parser, static_cast<llhttp_type_t>(parser_type), &http_settings);

    // 设置解析器和 TCP 对象的数据为当前对象
    m_parser->data = m_tcp->data = this;

    // 如果消息完成回调为空，则附加默认的消息完成回调
    if (http_settings.on_message_complete == nullptr) {
        attach(&http_settings);
    }
}

// 析构函数，释放内存
xmrig::HttpContext::~HttpContext()
{
    delete m_tcp;
    delete m_parser;
}

// 写入数据到连接
void xmrig::HttpContext::write(std::string &&data, bool close)
{
    // 如果连接不可写，则返回
    if (uv_is_writable(stream()) != 1) {
        return;
    }

    // 创建 HttpWriteBaton 对象，写入数据到连接
    auto baton = new HttpWriteBaton(std::move(data), close ? this : nullptr);
    baton->write(stream());
}

// 判断是否为请求
bool xmrig::HttpContext::isRequest() const
{
    return m_parser->type == HTTP_REQUEST;
}

// 解析数据
bool xmrig::HttpContext::parse(const char *data, size_t size)
{
    // 如果数据大小为0，则返回true
    if (size == 0) {
        return true;
    }

    // 执行解析器，返回解析结果
    return llhttp_execute(m_parser, data, size) == HPE_OK;
}

// 获取客户端IP地址
std::string xmrig::HttpContext::ip() const
{
    char ip[46]           = {};
    sockaddr_storage addr = {};
    int size              = sizeof(addr);

    // 获取客户端地址信息
    uv_tcp_getpeername(m_tcp, reinterpret_cast<sockaddr*>(&addr), &size);
    // 根据地址类型获取IP地址
    if (reinterpret_cast<sockaddr_in *>(&addr)->sin_family == AF_INET6) {
        uv_ip6_name(reinterpret_cast<sockaddr_in6*>(&addr), ip, 45);
    }
    else {
        uv_ip4_name(reinterpret_cast<sockaddr_in*>(&addr), ip, 16);
    }

    return ip;
}

// 获取对象创建后经过的时间
uint64_t xmrig::HttpContext::elapsed() const
{
    return Chrono::steadyMSecs() - m_timestamp;
}

// 关闭连接
void xmrig::HttpContext::close(int status)
{
    // 如果对象不存在，则返回
    if (!get(id())) {
        return;
    }

    // 获取监听器
    auto listener = httpListener();

    // 如果状态小于0且存在监听器，则设置状态并调用监听器的回调函数
    if (status < 0 && listener) {
        this->status = status;
        listener->onHttpData(*this);
    }
}
    # 调用 storage 对象的 erase 方法，删除指定 id 的数据
    storage.erase(id());

    # 检查当前的 handle 是否处于关闭状态
    if (!uv_is_closing(handle())) {
        # 如果 handle 没有处于关闭状态，则调用 uv_close 方法关闭 handle，并传入一个 lambda 表达式作为回调函数
        uv_close(handle(), [](uv_handle_t *handle) -> void { delete reinterpret_cast<HttpContext*>(handle->data); });
    }
}

// 根据 id 获取 HttpContext 对象
xmrig::HttpContext *xmrig::HttpContext::get(uint64_t id)
{
    // 在存储中查找对应 id 的 HttpContext 对象
    const auto it = storage.find(id);

    // 如果找不到，则返回空指针；否则返回对应的 HttpContext 对象
    return it == storage.end() ? nullptr : it->second;
}

// 关闭所有 HttpContext 对象
void xmrig::HttpContext::closeAll()
{
    // 遍历存储中的所有 HttpContext 对象
    for (auto &kv : storage) {
        // 如果 HttpContext 对象的句柄没有关闭，则关闭它
        if (!uv_is_closing(kv.second->handle())) {
            uv_close(kv.second->handle(), [](uv_handle_t *handle) -> void { delete reinterpret_cast<HttpContext*>(handle->data); });
        }
    }
}

// 处理 HTTP 请求头字段
int xmrig::HttpContext::onHeaderField(llhttp_t *parser, const char *at, size_t length)
{
    // 获取当前的 HttpContext 对象
    auto ctx = static_cast<HttpContext*>(parser->data);

    // 如果上一个字段是值，则设置上一个字段，并重置标记
    if (ctx->m_wasHeaderValue) {
        if (!ctx->m_lastHeaderField.empty()) {
            ctx->setHeader();
        }
        ctx->m_lastHeaderField = std::string(at, length);
        ctx->m_wasHeaderValue  = false;
    } else {
        ctx->m_lastHeaderField += std::string(at, length);
    }

    return 0;
}

// 处理 HTTP 请求头字段值
int xmrig::HttpContext::onHeaderValue(llhttp_t *parser, const char *at, size_t length)
{
    // 获取当前的 HttpContext 对象
    auto ctx = static_cast<HttpContext*>(parser->data);

    // 如果上一个字段不是值，则设置当前字段值，并设置标记
    if (!ctx->m_wasHeaderValue) {
        ctx->m_lastHeaderValue = std::string(at, length);
        ctx->m_wasHeaderValue  = true;
    } else {
        ctx->m_lastHeaderValue += std::string(at, length);
    }

    return 0;
}

// 将 llhttp_settings_t 与 HttpContext 关联
void xmrig::HttpContext::attach(llhttp_settings_t *settings)
{
    // 清空消息开始、状态、分块头和分块完成的处理函数
    settings->on_message_begin  = nullptr;
    settings->on_status         = nullptr;
    settings->on_chunk_header   = nullptr;
    settings->on_chunk_complete = nullptr;

    // 设置处理 URL 的函数
    settings->on_url = [](llhttp_t *parser, const char *at, size_t length) -> int
    {
        static_cast<HttpContext*>(parser->data)->url = std::string(at, length);
        return 0;
    };

    // 设置处理头字段和头字段值的函数
    settings->on_header_field = onHeaderField;
    settings->on_header_value = onHeaderValue;
}
    // 设置 HTTP 请求头解析完成后的回调函数
    settings->on_headers_complete = [](llhttp_t *parser) -> int {
        // 将解析器中的数据转换为 HttpContext 对象
        auto ctx = static_cast<HttpContext*>(parser->data);
        // 设置 HttpContext 对象的状态码为解析器中的状态码
        ctx->status = parser->status_code;

        // 如果解析器类型为 HTTP_REQUEST，则设置 HttpContext 对象的方法为解析器中的方法
        if (parser->type == HTTP_REQUEST) {
            ctx->method = parser->method;
        }

        // 如果上一个头字段不为空，则设置头字段
        if (!ctx->m_lastHeaderField.empty()) {
            ctx->setHeader();
        }

        return 0;
    };

    // 设置 HTTP 请求体解析完成后的回调函数
    settings->on_body = [](llhttp_t *parser, const char *at, size_t len) -> int
    {
        // 将解析器中的数据转换为 HttpContext 对象，并将解析得到的请求体追加到 HttpContext 对象的 body 属性中
        static_cast<HttpContext*>(parser->data)->body.append(at, len);

        return 0;
    };

    // 设置 HTTP 消息解析完成后的回调函数
    settings->on_message_complete = [](llhttp_t *parser) -> int
    {
        // 将解析器中的数据转换为 HttpContext 对象
        auto ctx      = static_cast<HttpContext*>(parser->data);
        // 获取 HttpContext 对象的 HTTP 监听器
        auto listener = ctx->httpListener();

        // 如果监听器存在，则调用监听器的 onHttpData 方法，并重置 HttpContext 对象的监听器
        if (listener) {
            listener->onHttpData(*ctx);
            ctx->m_listener.reset();
        }

        return 0;
    };
# 设置 HTTP 请求头部信息
void xmrig::HttpContext::setHeader()
{
    # 将最后一个头部字段转换为小写
    std::transform(m_lastHeaderField.begin(), m_lastHeaderField.end(), m_lastHeaderField.begin(), ::tolower);
    # 将最后一个头部字段和对应的值插入到头部信息中
    headers.insert({ m_lastHeaderField, m_lastHeaderValue });

    # 清空最后一个头部字段和对应的值
    m_lastHeaderField.clear();
    m_lastHeaderValue.clear();
}
```