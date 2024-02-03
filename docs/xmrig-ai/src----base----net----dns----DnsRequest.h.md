# `xmrig\src\base\net\dns\DnsRequest.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （在您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DNSREQUEST_H
#define XMRIG_DNSREQUEST_H


#include "base/tools/Object.h"


#include <cstdint>


namespace xmrig {


class IDnsListener;


class DnsRequest
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(DnsRequest)

    // 构造函数，初始化 DNS 请求的监听器
    DnsRequest(IDnsListener *listener) : listener(listener) {}
    // 默认析构函数
    ~DnsRequest() = default;

    // DNS 请求的监听器
    IDnsListener *listener;
};


} /* namespace xmrig */


#endif /* XMRIG_DNSREQUEST_H */
```