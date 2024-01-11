# `xmrig\src\base\net\dns\DnsConfig.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DNSCONFIG_H
#define XMRIG_DNSCONFIG_H


#include "3rdparty/rapidjson/fwd.h"


namespace xmrig {


class DnsConfig
{
public:
    static const char *kField;
    static const char *kIPv6;
    static const char *kTTL;

    DnsConfig() = default;
    DnsConfig(const rapidjson::Value &value);

    // 返回是否为IPv6
    inline bool isIPv6() const  { return m_ipv6; }
    // 返回TTL的毫秒数
    inline uint32_t ttl() const { return m_ttl * 1000U; }

    // 将DnsConfig对象转换为JSON格式
    rapidjson::Value toJSON(rapidjson::Document &doc) const;


private:
    // 是否为IPv6，默认为false
    bool m_ipv6     = false;
    // TTL的秒数，默认为30秒
    uint32_t m_ttl  = 30U;
};


} /* namespace xmrig */


#endif /* XMRIG_DNSCONFIG_H */
```