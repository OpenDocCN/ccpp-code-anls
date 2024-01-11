# `xmrig\src\base\net\http\HttpData.cpp`

```
/* XMRig
 * 版权所有 (c) 2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本3，或者
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


#include "base/net/http/HttpData.h"
#include "3rdparty/llhttp/llhttp.h"
#include "3rdparty/rapidjson/document.h"
#include "3rdparty/rapidjson/error/en.h"
#include "base/io/json/Json.h"


#include <uv.h>
#include <stdexcept>


/* 状态码 */
enum http_status
{
#   define XX(num, name, string) HTTP_STATUS_##name = num,
    HTTP_STATUS_MAP(XX)
#   undef XX
};


namespace xmrig {


const std::string HttpData::kApplicationJson    = "application/json";
const std::string HttpData::kContentType        = "Content-Type";
const std::string HttpData::kContentTypeL       = "content-type";
const std::string HttpData::kTextPlain          = "text/plain";


static const char *http_status_str(enum http_status s)
{
    switch (s) {
#   define XX(num, name, string) case HTTP_STATUS_##name: return #string;
    HTTP_STATUS_MAP(XX)
#   undef XX
    default: return "<unknown>";
    }
}


} // namespace xmrig


bool xmrig::HttpData::isJSON() const
{
    if (!headers.count(kContentTypeL)) {
        return false;
    }

    auto &type = headers.at(kContentTypeL);
    # 返回一个布尔值，判断 type 是否等于 kApplicationJson 或者 type 是否等于 kTextPlain
    return type == kApplicationJson || type == kTextPlain;
// 返回当前 HTTP 请求的方法名
const char *xmrig::HttpData::methodName() const
{
    return llhttp_method_name(static_cast<llhttp_method>(method));
}

// 返回 JSON 格式的 HTTP 响应数据
rapidjson::Document xmrig::HttpData::json() const
{
    // 如果状态码小于 0，抛出运行时错误
    if (status < 0) {
        throw std::runtime_error(statusName());
    }

    // 如果响应不是有效的 JSON 格式，抛出运行时错误
    if (!isJSON()) {
        throw std::runtime_error("the response is not a valid JSON response");
    }

    // 使用 rapidjson 解析响应体，如果解析出错，抛出运行时错误
    using namespace rapidjson;
    Document doc;
    if (doc.Parse(body.c_str()).HasParseError()) {
        throw std::runtime_error(GetParseError_En(doc.GetParseError()));
    }

    // 如果解析出的 JSON 对象不为空且包含 "error" 字段，抛出运行时错误
    if (doc.IsObject() && !doc.ObjectEmpty()) {
        const char *error = Json::getString(doc, "error");
        if (error) {
            throw std::runtime_error(error);
        }
    }

    return doc;
}

// 返回指定状态码对应的状态名
const char *xmrig::HttpData::statusName(int status)
{
    // 如果状态码小于 0，返回 libuv 库中对应的错误信息
    if (status < 0) {
        return uv_strerror(status);
    }

    // 返回指定 HTTP 状态码对应的状态名
    return http_status_str(static_cast<http_status>(status));
}
```