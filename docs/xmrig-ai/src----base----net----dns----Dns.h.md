# `xmrig\src\base\net\dns\Dns.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，无论是许可证的第3版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DNS_H
#define XMRIG_DNS_H


#include "base/net/dns/DnsConfig.h"
#include "base/tools/String.h"


#include <map>
#include <memory>


namespace xmrig {


class DnsConfig;
class DnsRequest;
class IDnsBackend;
class IDnsListener;


class Dns
{
public:
    // 返回 DNS 配置
    inline static const DnsConfig &config()             { return m_config; }
    // 设置 DNS 配置
    inline static void set(const DnsConfig &config)     { m_config = config; }

    // 解析主机名并返回 DNS 请求对象
    static std::shared_ptr<DnsRequest> resolve(const String &host, IDnsListener *listener, uint64_t ttl = 0);

private:
    // DNS 配置
    static DnsConfig m_config;
    // DNS 后端映射
    static std::map<String, std::shared_ptr<IDnsBackend> > m_backends;
};


} /* namespace xmrig */


#endif /* XMRIG_DNS_H */
```