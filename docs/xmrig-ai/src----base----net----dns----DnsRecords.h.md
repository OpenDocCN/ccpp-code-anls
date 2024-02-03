# `xmrig\src\base\net\dns\DnsRecords.h`

```cpp
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
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DNSRECORDS_H
#define XMRIG_DNSRECORDS_H


#include "base/net/dns/DnsRecord.h"


namespace xmrig {


class DnsRecords
{
public:
    // 检查DNS记录是否为空
    inline bool isEmpty() const       { return m_ipv4.empty() && m_ipv6.empty(); }

    // 获取DNS记录
    const DnsRecord &get(DnsRecord::Type prefered = DnsRecord::Unknown) const;
    // 计算指定类型的DNS记录数量
    size_t count(DnsRecord::Type type = DnsRecord::Unknown) const;
    // 清空DNS记录
    void clear();
    // 解析addrinfo结构中的DNS记录
    void parse(addrinfo *res);

private:
    // IPv4地址的DNS记录
    std::vector<DnsRecord> m_ipv4;
    // IPv6地址的DNS记录
    std::vector<DnsRecord> m_ipv6;
};


} /* namespace xmrig */


#endif /* XMRIG_DNSRECORDS_H */
```