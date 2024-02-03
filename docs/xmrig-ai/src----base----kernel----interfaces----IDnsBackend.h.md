# `xmrig\src\base\kernel\interfaces\IDnsBackend.h`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IDNSBACKEND_H
#define XMRIG_IDNSBACKEND_H


#include "base/tools/Object.h"


#include <memory>


namespace xmrig {


class DnsRecords;
class DnsRequest;
class IDnsListener;
class String;


class IDnsBackend
{
public:
    XMRIG_DISABLE_COPY_MOVE(IDnsBackend)

    // 默认构造函数
    IDnsBackend()           = default;
    // 虚析构函数
    virtual ~IDnsBackend()  = default;

    // 返回 DNS 记录
    virtual const DnsRecords &records() const                                                               = 0;
    // 解析主机名，返回 DNS 请求的共享指针
    virtual std::shared_ptr<DnsRequest> resolve(const String &host, IDnsListener *listener, uint64_t ttl)   = 0;
};


} /* namespace xmrig */


#endif // XMRIG_IDNSBACKEND_H
```