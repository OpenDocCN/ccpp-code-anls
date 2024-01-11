# `xmrig\src\base\net\http\HttpContext.h`

```
/* XMRig
 * 版权所有（c）2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用的情况下分发的，但没有任何保证；甚至没有适用于特定目的的隐含保证。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HTTPCONTEXT_H
#define XMRIG_HTTPCONTEXT_H


using llhttp_settings_t     = struct llhttp_settings_s;
using llhttp_t              = struct llhttp__internal_s;
using uv_connect_t          = struct uv_connect_s;
using uv_handle_t           = struct uv_handle_s;
using uv_stream_t           = struct uv_stream_s;
using uv_tcp_t              = struct uv_tcp_s;


#include "base/net/http/HttpData.h"
#include "base/tools/Object.h"


#include <memory>


namespace xmrig {


class IHttpListener;


class HttpContext : public HttpData
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HttpContext)

    // 构造函数，初始化HttpContext对象
    HttpContext(int parser_type, const std::weak_ptr<IHttpListener> &listener);
    // 析构函数，释放HttpContext对象
    ~HttpContext() override;

    // 返回TCP流对象
    inline uv_stream_t *stream() const { return reinterpret_cast<uv_stream_t *>(m_tcp); }
    // 返回TCP句柄对象
    inline uv_handle_t *handle() const { return reinterpret_cast<uv_handle_t *>(m_tcp); }

    // 返回主机名
    inline const char *host() const override            { return nullptr; }
    // 返回TLS指纹
    inline const char *tlsFingerprint() const override  { return nullptr; }
    // 返回空指针，表示不支持 TLS 版本
    inline const char *tlsVersion() const override      { return nullptr; }
    // 返回 0，表示未指定端口
    inline uint16_t port() const override               { return 0; }

    // 写入数据并关闭连接
    void write(std::string &&data, bool close) override;

    // 判断是否为请求
    bool isRequest() const override;
    // 解析数据
    bool parse(const char *data, size_t size);
    // 返回 IP 地址
    std::string ip() const override;
    // 返回经过的时间
    uint64_t elapsed() const;
    // 关闭连接
    void close(int status = 0);

    // 根据 ID 获取 HttpContext 对象
    static HttpContext *get(uint64_t id);
    // 关闭所有连接
    static void closeAll();
protected:
    // 指向 libuv 中的 TCP 对象
    uv_tcp_t *m_tcp;

private:
    // 返回 HTTP 监听器指针，如果已经过期则返回空指针
    inline IHttpListener *httpListener() const { return m_listener.expired() ? nullptr : m_listener.lock().get(); }

    // 处理 HTTP 请求头字段的回调函数
    static int onHeaderField(llhttp_t *parser, const char *at, size_t length);
    // 处理 HTTP 请求头值的回调函数
    static int onHeaderValue(llhttp_t *parser, const char *at, size_t length);
    // 将 llhttp_settings_t 与当前对象关联
    static void attach(llhttp_settings_t *settings);

    // 设置 HTTP 请求头
    void setHeader();

    // 标记上一个值是否为 HTTP 请求头值
    bool m_wasHeaderValue           = false;
    // 时间戳
    const uint64_t m_timestamp;
    // HTTP 解析器
    llhttp_t *m_parser;
    // 上一个 HTTP 请求头字段
    std::string m_lastHeaderField;
    // 上一个 HTTP 请求头值
    std::string m_lastHeaderValue;
    // 弱引用指向 HTTP 监听器
    std::weak_ptr<IHttpListener> m_listener;
};


} // namespace xmrig


#endif // XMRIG_HTTPCONTEXT_H
```