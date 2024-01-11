# `xmrig\src\base\net\http\HttpApiResponse.cpp`

```
/* XMRig
 * 版权所有（c）2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 *   其版本由自由软件基金会发布，可以选择遵循许可证的第3版或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "base/net/http/HttpApiResponse.h"
#include "3rdparty/rapidjson/prettywriter.h"
#include "3rdparty/rapidjson/stringbuffer.h"
#include "base/net/http/HttpData.h"


namespace xmrig {

static const char *kError  = "error";
static const char *kStatus = "status";

} // namespace xmrig


// 使用给定的ID初始化HttpApiResponse对象
xmrig::HttpApiResponse::HttpApiResponse(uint64_t id) :
    HttpResponse(id),
    m_doc(rapidjson::kObjectType)
{
}


// 使用给定的ID和状态码初始化HttpApiResponse对象
xmrig::HttpApiResponse::HttpApiResponse(uint64_t id, int status) :
    HttpResponse(id),
    m_doc(rapidjson::kObjectType)
{
    // 设置状态码
    setStatus(status);
}


// 结束HTTP响应
void xmrig::HttpApiResponse::end()
{
    using namespace rapidjson;

    // 设置响应头
    setHeader("Access-Control-Allow-Origin", "*");
    setHeader("Access-Control-Allow-Methods", "GET, PUT, POST, DELETE");
    setHeader("Access-Control-Allow-Headers", "Authorization, Content-Type");
    # 如果状态码大于等于400
    if (statusCode() >= 400) {
        # 如果JSON文档中没有状态字段，则添加状态字段
        if (!m_doc.HasMember(kStatus)) {
            m_doc.AddMember(StringRef(kStatus), statusCode(), m_doc.GetAllocator());
        }
        # 如果JSON文档中没有错误字段，则添加错误字段
        if (!m_doc.HasMember(kError)) {
            m_doc.AddMember(StringRef(kError), StringRef(HttpData::statusName(statusCode())), m_doc.GetAllocator());
        }
    }

    # 如果JSON文档中没有任何成员，则返回HTTP响应结束
    if (!m_doc.MemberCount()) {
        return HttpResponse::end();
    }

    # 设置HTTP响应头的Content-Type为application/json
    setHeader(HttpData::kContentType, HttpData::kApplicationJson);

    # 创建一个缓冲区，用于存储JSON数据
    StringBuffer buffer(nullptr, 4096);
    # 创建一个PrettyWriter对象，用于将JSON数据格式化为可读性更好的形式
    PrettyWriter<StringBuffer> writer(buffer);
    # 设置JSON数据中小数的最大位数为10
    writer.SetMaxDecimalPlaces(10);
    # 设置JSON数据的格式选项为单行数组
    writer.SetFormatOptions(kFormatSingleLineArray);

    # 将JSON文档接受到PrettyWriter中
    m_doc.Accept(writer);

    # 结束HTTP响应，将JSON数据作为响应体返回
    HttpResponse::end(buffer.GetString(), buffer.GetSize());
# 闭合前面的函数定义
```