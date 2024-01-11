# `xmrig\src\base\net\dns\DnsRecord.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DNSRECORD_H
#define XMRIG_DNSRECORD_H


struct addrinfo;
struct sockaddr;


#include "base/tools/String.h"


namespace xmrig {


class DnsRecord
{
public:
    enum Type : uint32_t {
        Unknown,
        A,
        AAAA
    };

    // 默认构造函数
    DnsRecord() {}
    // 从addrinfo指针构造DnsRecord对象
    DnsRecord(const addrinfo *addr);

    // 返回sockaddr指针，可选参数port默认值为0
    const sockaddr *addr(uint16_t port = 0) const;
    // 返回IP地址字符串
    String ip() const;

    // 返回是否有效
    inline bool isValid() const     { return m_type != Unknown; }
    // 返回记录类型
    inline Type type() const        { return m_type; }

private:
    // 可变长度为28的字节数组
    mutable uint8_t m_data[28]{};
    // 常量记录类型，默认值为Unknown
    const Type m_type = Unknown;
};


} /* namespace xmrig */


#endif /* XMRIG_DNSRECORD_H */
```