# `xmrig\src\base\net\dns\DnsUvBackend.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
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
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include <uv.h>


#include "base/net/dns/DnsUvBackend.h"
#include "base/kernel/interfaces/IDnsListener.h"
#include "base/net/dns/DnsRequest.h"
#include "base/tools/Chrono.h"


namespace xmrig {


static Storage<DnsUvBackend> *storage = nullptr;


// 获取DnsUvBackend的存储引用
Storage<DnsUvBackend> &DnsUvBackend::getStorage()
{
    if (storage == nullptr) {
        storage = new Storage<DnsUvBackend>();
    }

    return *storage;
}


static addrinfo hints{};


} // namespace xmrig


// 构造函数
xmrig::DnsUvBackend::DnsUvBackend()
{
    if (!hints.ai_protocol) {
        hints.ai_family     = AF_UNSPEC;
        hints.ai_socktype   = SOCK_STREAM;
        hints.ai_protocol   = IPPROTO_TCP;
    }

    m_key = getStorage().add(this);
}


// 析构函数
xmrig::DnsUvBackend::~DnsUvBackend()
{
    assert(storage);

    storage->release(m_key);

    if (storage->isEmpty()) {
        delete storage;
        storage = nullptr;
    }
}


// 解析主机名
std::shared_ptr<xmrig::DnsRequest> xmrig::DnsUvBackend::resolve(const String &host, IDnsListener *listener, uint64_t ttl)
{
    auto req = std::make_shared<DnsRequest>(listener);
    # 如果当前时间减去上次解析的时间小于等于TTL并且记录不为空
    if (Chrono::currentMSecsSinceEpoch() - m_ts <= ttl && !m_records.isEmpty()) {
        # 调用请求的监听器的onResolved方法，传入记录和0以及空指针
        req->listener->onResolved(m_records, 0, nullptr);
    } else {
        # 将请求加入队列
        m_queue.emplace(req);
    }

    # 如果队列大小为1并且解析主机失败
    if (m_queue.size() == 1 && !resolve(host)) {
        # 调用done方法
        done();
    }

    # 返回请求
    return req;
# 解析域名对应的 IP 地址
bool xmrig::DnsUvBackend::resolve(const String &host)
{
    # 创建一个共享指针，用于存储获取地址信息的请求
    m_req = std::make_shared<uv_getaddrinfo_t>();
    # 将请求的数据指针设置为存储中的指针
    m_req->data = getStorage().ptr(m_key);
    # 使用 libuv 库中的函数获取地址信息
    m_status = uv_getaddrinfo(uv_default_loop(), m_req.get(), DnsUvBackend::onResolved, host.data(), nullptr, &hints);
    # 返回获取地址信息的状态
    return m_status == 0;
}

# 完成解析后的处理
void xmrig::DnsUvBackend::done()
{
    # 如果获取地址信息的状态小于 0，则将错误信息存储到 error 中
    const char *error = m_status < 0 ? uv_strerror(m_status) : nullptr;
    # 循环处理请求队列中的请求
    while (!m_queue.empty()) {
        # 获取队列中的请求，并将其转移为弱引用
        auto req = std::move(m_queue.front()).lock();
        # 如果请求存在，则调用监听器的 onResolved 方法
        if (req) {
            req->listener->onResolved(m_records, m_status, error);
        }
        # 弹出队列中的请求
        m_queue.pop();
    }
    # 重置获取地址信息的请求
    m_req.reset();
}

# 获取地址信息完成后的处理
void xmrig::DnsUvBackend::onResolved(int status, addrinfo *res)
{
    # 获取当前时间戳
    m_ts = Chrono::currentMSecsSinceEpoch();
    # 如果获取地址信息的状态小于 0，则调用 done 方法处理
    if ((m_status = status) < 0) {
        return done();
    }
    # 解析获取的地址信息
    m_records.parse(res);
    # 如果解析后的记录为空，则将状态设置为 UV_EAI_NONAME
    if (m_records.isEmpty()) {
        m_status = UV_EAI_NONAME;
    }
    # 调用 done 方法处理
    done();
}

# libuv 获取地址信息完成后的回调函数
void xmrig::DnsUvBackend::onResolved(uv_getaddrinfo_t *req, int status, addrinfo *res)
{
    # 获取存储中与请求数据指针对应的后端对象
    auto backend = getStorage().get(req->data);
    # 如果后端对象存在，则调用其 onResolved 方法处理
    if (backend) {
        backend->onResolved(status, res);
    }
    # 释放地址信息的内存
    uv_freeaddrinfo(res);
}
```