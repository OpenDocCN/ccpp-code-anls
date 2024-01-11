# `xmrig\src\base\net\http\Fetch.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/http/Fetch.h"
#include "3rdparty/rapidjson/document.h"
#include "3rdparty/rapidjson/stringbuffer.h"
#include "3rdparty/rapidjson/writer.h"
#include "base/io/log/Log.h"
#include "base/net/http/HttpClient.h"


#ifdef XMRIG_FEATURE_TLS
#   include "base/net/https/HttpsClient.h"
#endif

// 定义FetchRequest类的构造函数，初始化成员变量
xmrig::FetchRequest::FetchRequest(llhttp_method method, const String &host, uint16_t port, const String &path, bool tls, bool quiet, const char *data, size_t size, const char *contentType) :
    quiet(quiet),  // 初始化quiet成员变量
    tls(tls),  // 初始化tls成员变量
    method(method),  // 初始化method成员变量
    host(host),  // 初始化host成员变量
    path(path),  // 初始化path成员变量
    port(port)  // 初始化port成员变量
{
    assert(port > 0);  // 断言端口号大于0

    setBody(data, size, contentType);  // 调用setBody方法设置请求体
}

// 定义FetchRequest类的构造函数，初始化成员变量
xmrig::FetchRequest::FetchRequest(llhttp_method method, const String &host, uint16_t port, const String &path, const rapidjson::Value &value, bool tls, bool quiet) :
    quiet(quiet),  // 初始化quiet成员变量
    tls(tls),  // 初始化tls成员变量
    method(method),  // 初始化method成员变量
    host(host),  // 初始化host成员变量
    path(path),  // 初始化path成员变量
    port(port)  // 初始化port成员变量
{
    assert(port > 0);  // 断言端口号大于0

    setBody(value);  // 调用setBody方法设置请求体
}

// 定义setBody方法，设置请求体
void xmrig::FetchRequest::setBody(const char *data, size_t size, const char *contentType)
{
    if (!data) {  // 如果请求体数据为空
        return;  // 直接返回
    }
}
    # 确保方法不是 HTTP_GET 和 HTTP_HEAD
    assert(method != HTTP_GET && method != HTTP_HEAD);

    # 如果方法是 HTTP_GET 或者 HTTP_HEAD，则直接返回
    if (method == HTTP_GET || method == HTTP_HEAD) {
        return;
    }

    # 根据数据和大小创建字符串，如果大小为0，则直接使用数据
    body = size ? std::string(data, size) : data;
    
    # 如果存在内容类型，则将内容类型添加到头部中
    if (contentType) {
        headers.insert({ HttpData::kContentType, contentType });
    }
}

// 设置请求体内容
void xmrig::FetchRequest::setBody(const rapidjson::Value &value)
{
    // 断言请求方法不是 HTTP_GET 或 HTTP_HEAD
    assert(method != HTTP_GET && method != HTTP_HEAD);

    // 如果请求方法是 HTTP_GET 或 HTTP_HEAD，则返回
    if (method == HTTP_GET || method == HTTP_HEAD) {
        return;
    }

    // 使用 rapidjson 命名空间
    using namespace rapidjson;

    // 创建一个字符串缓冲区
    StringBuffer buffer(nullptr, 512);
    // 创建一个写入器，并将值写入缓冲区
    Writer<StringBuffer> writer(buffer);
    value.Accept(writer);

    // 设置请求体内容
    setBody(buffer.GetString(), buffer.GetSize(), HttpData::kApplicationJson.c_str());
}

// 发起 HTTP 请求
void xmrig::fetch(const char *tag, FetchRequest &&req, const std::weak_ptr<IHttpListener> &listener, int type, uint64_t rpcId)
{
#   ifdef APP_DEBUG
    // 如果是调试模式，则输出请求信息和请求体内容
    LOG_DEBUG(CYAN("http%s://%s:%u ") MAGENTA_BOLD("\"%s %s\"") BLACK_BOLD(" body: ") CYAN_BOLD("%zu") BLACK_BOLD(" bytes"),
              req.tls ? "s" : "", req.host.data(), req.port, llhttp_method_name(req.method), req.path.data(), req.body.size());

    if (req.hasBody() && req.body.size() < (Log::kMaxBufferSize - 1024) && req.headers.count(HttpData::kContentType) && req.headers.at(HttpData::kContentType) == HttpData::kApplicationJson) {
        Log::print(BLUE_BG_BOLD("%s:") BLACK_BOLD_S " %.*s", req.headers.at(HttpData::kContentType).c_str(), static_cast<int>(req.body.size()), req.body.c_str());
    }
#   endif

    // 创建一个 HTTP 客户端指针
    HttpClient *client = nullptr;
#   ifdef XMRIG_FEATURE_TLS
    // 如果是 TLS 请求，则创建一个 HTTPS 客户端
    if (req.tls) {
        client = new HttpsClient(tag, std::move(req), listener);
    }
    // 否则创建一个 HTTP 客户端
    else
#   endif
    {
        client = new HttpClient(tag, std::move(req), listener);
    }

    // 设置用户类型和 RPC ID
    client->userType = type;
    client->rpcId    = rpcId;
    // 连接服务器
    client->connect();
}
```