# `xmrig\src\base\net\dns\DnsRecords.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
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

#include <uv.h>


#include "base/net/dns/DnsRecords.h"
#include "base/net/dns/Dns.h"


const xmrig::DnsRecord &xmrig::DnsRecords::get(DnsRecord::Type prefered) const
{
    static const DnsRecord defaultRecord;

    if (isEmpty()) {
        return defaultRecord;
    }

    const size_t ipv4 = m_ipv4.size();
    const size_t ipv6 = m_ipv6.size();

    if (ipv6 && (prefered == DnsRecord::AAAA || Dns::config().isIPv6() || !ipv4)) {
        return m_ipv6[ipv6 == 1 ? 0 : static_cast<size_t>(rand()) % ipv6]; // NOLINT(concurrency-mt-unsafe, cert-msc30-c, cert-msc50-cpp)
    }

    if (ipv4) {
        return m_ipv4[ipv4 == 1 ? 0 : static_cast<size_t>(rand()) % ipv4]; // NOLINT(concurrency-mt-unsafe, cert-msc30-c, cert-msc50-cpp)
    }

    return defaultRecord;
}


size_t xmrig::DnsRecords::count(DnsRecord::Type type) const
{
    if (type == DnsRecord::A) {
        return m_ipv4.size();
    }

    if (type == DnsRecord::AAAA) {
        return m_ipv6.size();
    }

    return m_ipv4.size() + m_ipv6.size();
}


void xmrig::DnsRecords::clear()
{
    m_ipv4.clear();
    m_ipv6.clear();
}


void xmrig::DnsRecords::parse(addrinfo *res)
{
    clear();
    # 创建指向addrinfo结构体的指针，指向addrinfo链表的头部
    addrinfo *ptr = res;
    # 初始化IPv4地址数量和IPv6地址数量
    size_t ipv4   = 0;
    size_t ipv6   = 0;

    # 遍历addrinfo链表，统计IPv4和IPv6地址的数量
    while (ptr != nullptr) {
        if (ptr->ai_family == AF_INET) {
            # 如果地址族为IPv4，则IPv4地址数量加一
            ++ipv4;
        }
        else if (ptr->ai_family == AF_INET6) {
            # 如果地址族为IPv6，则IPv6地址数量加一
            ++ipv6;
        }

        # 指针指向下一个addrinfo结构体
        ptr = ptr->ai_next;
    }

    # 如果没有找到IPv4和IPv6地址，则直接返回
    if (ipv4 == 0 && ipv6 == 0) {
        return;
    }

    # 预留足够的空间来存储IPv4地址和IPv6地址
    m_ipv4.reserve(ipv4);
    m_ipv6.reserve(ipv6);

    # 重新将指针指向addrinfo链表的头部
    ptr = res;
    # 再次遍历addrinfo链表，将IPv4地址和IPv6地址分别存储到对应的容器中
    while (ptr != nullptr) {
        if (ptr->ai_family == AF_INET) {
            # 如果地址族为IPv4，则将该地址存储到IPv4地址容器中
            m_ipv4.emplace_back(ptr);
        }
        else if (ptr->ai_family == AF_INET6) {
            # 如果地址族为IPv6，则将该地址存储到IPv6地址容器中
            m_ipv6.emplace_back(ptr);
        }

        # 指针指向下一个addrinfo结构体
        ptr = ptr->ai_next;
    }
# 闭合前面的函数定义
```