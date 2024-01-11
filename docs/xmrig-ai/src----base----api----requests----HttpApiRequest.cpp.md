# `xmrig\src\base\api\requests\HttpApiRequest.cpp`

```
/* XMRig
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/api/requests/HttpApiRequest.h"
#include "3rdparty/llhttp/llhttp.h"
#include "3rdparty/rapidjson/error/en.h"
#include "base/io/json/Json.h"
#include "base/net/http/HttpData.h"


namespace xmrig {


static const char *kError  = "error";
static const char *kId     = "id";
static const char *kResult = "result";


static inline const char *rpcError(int code) {
    switch (code) {
    case IApiRequest::RPC_PARSE_ERROR:
        return "解析错误";

    case IApiRequest::RPC_INVALID_REQUEST:
        return "无效请求";

    case IApiRequest::RPC_METHOD_NOT_FOUND:
        return "找不到方法";

    case IApiRequest::RPC_INVALID_PARAMS:
        return "无效参数";

    default:
        break;
    }

    if (code >= 400 && code < 600) {
        return HttpData::statusName(code);
    }

    return "内部错误";
}


} // namespace xmrig


xmrig::HttpApiRequest::HttpApiRequest(const HttpData &req, bool restricted) :
    ApiRequest(SOURCE_HTTP, restricted),
    m_req(req),
    m_res(req.id()),
    m_url(req.url.c_str())
{
    # 如果请求方法为 GET
    if (method() == METHOD_GET) {
        # 如果请求的 URL 是 "/1/summary" 或 "/2/summary" 或 "/api.json"
        if (url() == "/1/summary" || url() == "/2/summary" || url() == "/api.json") {
            # 设置请求类型为 SUMMARY
            m_type = REQ_SUMMARY;
        }
    }

    # 如果请求方法为 POST 且 URL 是 "/json_rpc"
    if (method() == METHOD_POST && url() == "/json_rpc") {
        # 设置请求类型为 JSON_RPC
        m_type = REQ_JSON_RPC;
        # 接受请求
        accept();

        # 如果存在解析错误
        if (hasParseError()) {
            # 完成请求，返回解析错误
            done(RPC_PARSE_ERROR);
            return;
        }

        # 从请求体中获取 JSON-RPC 方法
        m_rpcMethod = Json::getString(m_body, "method");
        # 如果方法为空
        if (m_rpcMethod.isEmpty()) {
            # 完成请求，返回无效请求
            done(RPC_INVALID_REQUEST);
            return;
        }

        # 设置状态为新
        m_state = STATE_NEW;

        return;
    }

    # 如果 URL 的长度大于 4
    if (url().size() > 4) {
        # 如果 URL 的前三个字符是 "/2/"
        if (memcmp(url().data(), "/2/", 3) == 0) {
            # 设置版本为 2
            m_version = 2;
        }
    }
# 定义 xmrig 命名空间下的 HttpApiRequest 类的 accept 方法
bool xmrig::HttpApiRequest::accept()
{
    # 使用 rapidjson 命名空间
    using namespace rapidjson;

    # 调用 ApiRequest 类的 accept 方法
    ApiRequest::accept();

    # 如果 m_parsed 为 0 并且请求体不为空
    if (m_parsed == 0 && !m_req.body.empty()) {
        # 解析请求体，允许注释和尾随逗号
        m_body.Parse<kParseCommentsFlag | kParseTrailingCommasFlag>(m_req.body.c_str());
        # 如果解析出错，将 m_parsed 设置为 2，否则设置为 1
        m_parsed = m_body.HasParseError() ? 2 : 1;

        # 如果没有解析错误，返回 true
        if (!hasParseError()) {
            return true;
        }

        # 如果请求类型不是 REQ_JSON_RPC
        if (type() != REQ_JSON_RPC) {
            # 向回复中添加错误信息
            reply().AddMember(StringRef(kError), StringRef(GetParseError_En(m_body.GetParseError())), doc().GetAllocator());
        }

        # 返回 false
        return false;
    }

    # 返回是否有解析错误
    return hasParseError();
}

# 定义 xmrig 命名空间下的 HttpApiRequest 类的 json 方法
const rapidjson::Value &xmrig::HttpApiRequest::json() const
{
    # 如果请求类型是 REQ_JSON_RPC
    if (type() == REQ_JSON_RPC) {
        # 返回 m_body 中 "params" 的值
        return Json::getValue(m_body, "params");
    }

    # 返回 m_body
    return m_body;
}

# 定义 xmrig 命名空间下的 HttpApiRequest 类的 method 方法
xmrig::IApiRequest::Method xmrig::HttpApiRequest::method() const
{
    # 返回 m_req.method 转换为 IApiRequest::Method 类型的值
    return static_cast<IApiRequest::Method>(m_req.method);
}

# 定义 xmrig 命名空间下的 HttpApiRequest 类的 done 方法
void xmrig::HttpApiRequest::done(int status)
{
    # 调用 ApiRequest 类的 done 方法
    ApiRequest::done(status);

    # 如果请求类型是 REQ_JSON_RPC
    if (type() == REQ_JSON_RPC) {
        # 使用 rapidjson 命名空间
        using namespace rapidjson;
        # 获取分配器
        auto &allocator = doc().GetAllocator();

        # 设置响应状态为 200
        m_res.setStatus(200);

        # 如果状态不为 200
        if (status != 200) {
            # 设置 RPC 错误
            setRpcError(status == 404 /* NOT_FOUND */ ? RPC_METHOD_NOT_FOUND : status);
        }
        # 如果回复中没有 kResult 成员
        else if (!reply().HasMember(kResult)) {
            # 创建一个包含 "status" 为 "OK" 的 result 对象
            Value result(kObjectType);
            result.AddMember("status", "OK", allocator);

            # 设置 RPC 结果
            setRpcResult(result);
        }
    }
    # 否则
    else {
        # 设置响应状态为传入的 status
        m_res.setStatus(status);
    }

    # 结束响应
    m_res.end();
}

# 定义 xmrig 命名空间下的 HttpApiRequest 类的 setRpcError 方法
void xmrig::HttpApiRequest::setRpcError(int code, const char *message)
{
    # 使用 rapidjson 命名空间
    using namespace rapidjson;
    # 获取分配器
    auto &allocator = doc().GetAllocator();

    # 创建一个包含 "code" 和 "message" 的 error 对象
    Value error(kObjectType);
    error.AddMember("code",    code, allocator);
    error.AddMember("message", message ? StringRef(message) : StringRef(rpcError(code)), allocator);

    # 设置 RPC 错误
    rpcDone(kError, error);
}

# 定义 xmrig 命名空间下的 HttpApiRequest 类的 setRpcResult 方法
void xmrig::HttpApiRequest::setRpcResult(rapidjson::Value &result)
{
    # 设置 RPC 结果
    rpcDone(kResult, result);
}
// 定义 xmrig 命名空间下的 HttpApiRequest 类的 rpcDone 方法，参数为 key 和 value
void xmrig::HttpApiRequest::rpcDone(const char *key, rapidjson::Value &value)
{
    // 调用 ApiRequest 类的 done 方法，参数为 0
    ApiRequest::done(0);

    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 获取当前文档的分配器
    auto &allocator = doc().GetAllocator();

    // 向响应中添加成员，成员名为 key，值为 value，使用分配器分配内存
    reply().AddMember(StringRef(key), value, allocator);
    // 向响应中添加成员，成员名为 "jsonrpc"，值为 "2.0"，使用分配器分配内存
    reply().AddMember("jsonrpc", "2.0", allocator);
    // 向响应中添加成员，成员名为 kId，值为从 m_body 中获取的值，使用分配器分配内存
    reply().AddMember(StringRef(kId), Value().CopyFrom(Json::getValue(m_body, kId), allocator), allocator);

    // 设置响应的状态码为 200
    m_res.setStatus(200);
    // 结束响应
    m_res.end();
}
```