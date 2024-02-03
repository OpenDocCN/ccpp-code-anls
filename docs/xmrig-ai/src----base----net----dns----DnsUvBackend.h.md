# `xmrig\src\base\net\dns\DnsUvBackend.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是基于希望它有用而分发的，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DNSUVBACKEND_H
#define XMRIG_DNSUVBACKEND_H


#include "base/kernel/interfaces/IDnsBackend.h"
#include "base/net/dns/DnsRecords.h"
#include "base/net/tools/Storage.h"


#include <queue>


using uv_getaddrinfo_t = struct uv_getaddrinfo_s;


namespace xmrig {


class DnsUvBackend : public IDnsBackend
{
public:
    XMRIG_DISABLE_COPY_MOVE(DnsUvBackend)

    DnsUvBackend();
    ~DnsUvBackend() override;

protected:
    // 返回DNS记录
    inline const DnsRecords &records() const override   { return m_records; }

    // 解析主机名并返回DNS请求的共享指针
    std::shared_ptr<DnsRequest> resolve(const String &host, IDnsListener *listener, uint64_t ttl) override;

private:
    // 解析主机名
    bool resolve(const String &host);
    // 完成解析
    void done();
    // 当解析完成时调用
    void onResolved(int status, addrinfo *res);

    // 静态方法，当解析完成时调用
    static void onResolved(uv_getaddrinfo_t *req, int status, addrinfo *res);

    DnsRecords m_records;
    int m_status            = 0;
    std::queue<std::weak_ptr<DnsRequest> > m_queue;
    std::shared_ptr<uv_getaddrinfo_t> m_req;
    uint64_t m_ts           = 0;
    uintptr_t m_key;

    // 获取存储
    static Storage<DnsUvBackend>& getStorage();
    // 释放存储
    void releaseStorage();
};


} /* namespace xmrig */


#endif /* XMRIG_DNSUVBACKEND_H */
```