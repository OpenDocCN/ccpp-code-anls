# `xmrig\src\base\net\dns\DnsRecord.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款
 *   许可证的版本，或（在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <uv.h>


#include "base/net/dns/DnsRecord.h"


// 构造函数，根据地址信息初始化DNS记录
xmrig::DnsRecord::DnsRecord(const addrinfo *addr) :
    m_type(addr->ai_family == AF_INET6 ? AAAA : (addr->ai_family == AF_INET ? A : Unknown))
{
    // 静态断言，确保存储IPv6地址的空间足够
    static_assert(sizeof(m_data) >= sizeof(sockaddr_in6), "Not enough storage for IPv6 address.");

    // 将地址信息拷贝到m_data中
    memcpy(m_data, addr->ai_addr, m_type == AAAA ? sizeof(sockaddr_in6) : sizeof(sockaddr_in));
}


// 返回带有指定端口的地址信息
const sockaddr *xmrig::DnsRecord::addr(uint16_t port) const
{
    // 设置端口号
    reinterpret_cast<sockaddr_in*>(m_data)->sin_port = htons(port);

    // 返回地址信息
    return reinterpret_cast<const sockaddr *>(m_data);
}


// 返回IP地址字符串
xmrig::String xmrig::DnsRecord::ip() const
{
    char *buf = nullptr;

    // 如果是IPv6地址
    if (m_type == AAAA) {
        // 分配内存
        buf = new char[45]();
        // 获取IPv6地址字符串
        uv_ip6_name(reinterpret_cast<const sockaddr_in6*>(m_data), buf, 45);
    }
    else {
        // 分配内存
        buf = new char[16]();
        // 获取IPv4地址字符串
        uv_ip4_name(reinterpret_cast<const sockaddr_in*>(m_data), buf, 16);
    }

    // 返回IP地址字符串
    return buf;
}
```