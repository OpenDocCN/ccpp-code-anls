# `xmrig\src\base\net\http\Http.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或
 *   （在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/net/http/Http.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"


namespace xmrig {


// 定义常量
const char *Http::kEnabled    = "enabled";
const char *Http::kHost       = "host";
const char *Http::kLocalhost  = "127.0.0.1";
const char *Http::kPort       = "port";
const char *Http::kRestricted = "restricted";
const char *Http::kToken      = "access-token";


} // namespace xmrig


// 实现构造函数
xmrig::Http::Http() :
    m_host(kLocalhost)
{
}


// 实现isEqual方法
bool xmrig::Http::isEqual(const Http &other) const
{
    return other.m_enabled    == m_enabled &&
           other.m_restricted == m_restricted &&
           other.m_host       == m_host &&
           other.m_token      == m_token &&
           other.m_port       == m_port;
}


// 实现toJSON方法
rapidjson::Value xmrig::Http::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    Value obj(kObjectType);

    // 添加成员
    obj.AddMember(StringRef(kEnabled),    m_enabled, allocator);
    obj.AddMember(StringRef(kHost),       m_host.toJSON(), allocator);
    obj.AddMember(StringRef(kPort),       m_port, allocator);
    # 将名为 kToken 的键和 m_token.toJSON() 的值添加到 obj 对象中
    obj.AddMember(StringRef(kToken),      m_token.toJSON(), allocator);
    # 将名为 kRestricted 的键和 m_restricted 的值添加到 obj 对象中
    obj.AddMember(StringRef(kRestricted), m_restricted, allocator);
    
    # 返回包含添加键值对后的 obj 对象
    return obj;
# 加载 HTTP 配置信息
void xmrig::Http::load(const rapidjson::Value &http)
{
    # 如果传入的参数不是对象，则直接返回
    if (!http.IsObject()) {
        return;
    }

    # 从 JSON 对象中获取并设置 m_enabled 属性
    m_enabled    = Json::getBool(http, kEnabled);
    # 从 JSON 对象中获取并设置 m_restricted 属性，如果不存在则默认为 true
    m_restricted = Json::getBool(http, kRestricted, true);
    # 从 JSON 对象中获取并设置 m_host 属性，如果不存在则默认为 localhost
    m_host       = Json::getString(http, kHost, kLocalhost);
    # 从 JSON 对象中获取并设置 m_token 属性
    m_token      = Json::getString(http, kToken);

    # 调用 setPort 方法，设置端口号
    setPort(Json::getInt(http, kPort));
}

# 设置端口号
void xmrig::Http::setPort(int port)
{
    # 如果端口号在合法范围内，则设置 m_port 属性
    if (port >= 0 && port <= 65536) {
        m_port = static_cast<uint16_t>(port);
    }
}
```