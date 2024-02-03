# `xmrig\src\base\net\dns\DnsConfig.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
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

#include "base/net/dns/DnsConfig.h"
#include "3rdparty/rapidjson/document.h"
#include "base/io/json/Json.h"


#include <algorithm>


namespace xmrig {


const char *DnsConfig::kField   = "dns";
const char *DnsConfig::kIPv6    = "ipv6";
const char *DnsConfig::kTTL     = "ttl";


} // namespace xmrig


xmrig::DnsConfig::DnsConfig(const rapidjson::Value &value)
{
    // 从JSON值中获取IPv6配置，如果不存在则使用默认值
    m_ipv6  = Json::getBool(value, kIPv6, m_ipv6);
    // 从JSON值中获取TTL配置，如果不存在则使用默认值，并确保最小值为1
    m_ttl   = std::max(Json::getUint(value, kTTL, m_ttl), 1U);
}


rapidjson::Value xmrig::DnsConfig::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;

    auto &allocator = doc.GetAllocator();
    // 创建一个空的JSON对象
    Value obj(kObjectType);

    // 向JSON对象中添加IPv6配置
    obj.AddMember(StringRef(kIPv6), m_ipv6, allocator);
    // 向JSON对象中添加TTL配置
    obj.AddMember(StringRef(kTTL),  m_ttl, allocator);

    // 返回JSON对象
    return obj;
}
```