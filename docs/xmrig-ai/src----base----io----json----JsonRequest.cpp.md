# `xmrig\src\base\io\json\JsonRequest.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。更多详情请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/io/json/JsonRequest.h"
#include "3rdparty/rapidjson/document.h"

// 命名空间 xmrig
namespace xmrig {

// 定义静态常量字符串
const char *JsonRequest::k2_0               = "2.0";
const char *JsonRequest::kId                = "id";
const char *JsonRequest::kJsonRPC           = "jsonrpc";
const char *JsonRequest::kMethod            = "method";
const char *JsonRequest::kOK                = "OK";
const char *JsonRequest::kParams            = "params";
const char *JsonRequest::kResult            = "result";
const char *JsonRequest::kStatus            = "status";

const char *JsonRequest::kParseError        = "parse error";
const char *JsonRequest::kInvalidRequest    = "invalid request";
const char *JsonRequest::kMethodNotFound    = "method not found";
const char *JsonRequest::kInvalidParams     = "invalid params";
const char *JsonRequest::kInternalError     = "internal error";

// 定义静态变量 nextId
static uint64_t nextId                      = 0;

} // namespace xmrig

// 使用 rapidjson 命名空间
rapidjson::Document xmrig::JsonRequest::create(const char *method)
{
    return create(++nextId, method);
}

// 创建 JSON 请求
rapidjson::Document xmrig::JsonRequest::create(int64_t id, const char *method)
{
    using namespace rapidjson;
    // 创建一个 JSON 文档对象
    Document doc(kObjectType);
    // 获取 JSON 文档对象的分配器
    auto &allocator = doc.GetAllocator();

    // 向 JSON 文档对象添加成员，成员名为 kId，值为 id
    doc.AddMember(StringRef(kId),      id, allocator);
    // 向 JSON 文档对象添加成员，成员名为 kJsonRPC，值为 k2_0
    doc.AddMember(StringRef(kJsonRPC), StringRef(k2_0), allocator);
    // 向 JSON 文档对象添加成员，成员名为 kMethod，值为 method
    doc.AddMember(StringRef(kMethod),  StringRef(method), allocator);

    // 返回 JSON 文档对象
    return doc;
// 创建一个 JSON 请求，使用给定的方法和参数
uint64_t xmrig::JsonRequest::create(rapidjson::Document &doc, const char *method, rapidjson::Value &params)
{
    // 调用另一个重载的 create 方法，传入文档对象、下一个 ID、方法和参数
    return create(doc, ++nextId, method, params);
}

// 创建一个 JSON 请求，使用给定的 ID、方法和参数
uint64_t xmrig::JsonRequest::create(rapidjson::Document &doc, int64_t id, const char *method, rapidjson::Value &params)
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;
    // 获取文档对象的分配器
    auto &allocator = doc.GetAllocator();

    // 向文档对象添加成员：ID
    doc.AddMember(StringRef(kId),      id, allocator);
    // 向文档对象添加成员：JSON-RPC 版本
    doc.AddMember(StringRef(kJsonRPC), StringRef(k2_0), allocator);
    // 向文档对象添加成员：方法
    doc.AddMember(StringRef(kMethod),  StringRef(method), allocator);
    // 向文档对象添加成员：参数
    doc.AddMember(StringRef(kParams),  params, allocator);

    // 返回 ID
    return id;
}
```