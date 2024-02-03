# `xmrig\src\base\net\dns\Dns.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新分发或修改
 *   根据GNU通用公共许可证的条款发布
 *   由自由软件基金会发布的许可证的第3版或
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


#include "base/net/dns/Dns.h"
#include "base/net/dns/DnsUvBackend.h"


namespace xmrig {


DnsConfig Dns::m_config;
std::map<String, std::shared_ptr<IDnsBackend> > Dns::m_backends;


} // namespace xmrig


// 解析主机名并返回DnsRequest对象的共享指针
std::shared_ptr<xmrig::DnsRequest> xmrig::Dns::resolve(const String &host, IDnsListener *listener, uint64_t ttl)
{
    // 如果主机名在后端中不存在，则插入DnsUvBackend对象
    if (m_backends.find(host) == m_backends.end()) {
        m_backends.insert({ host, std::make_shared<DnsUvBackend>() });
    }

    // 调用后端的解析方法，并返回结果
    return m_backends.at(host)->resolve(host, listener, ttl == 0 ? m_config.ttl() : ttl);
}
```