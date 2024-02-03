# `xmrig\src\base\net\http\HttpClient.h`

```cpp
/* XMRig
 * 版权所有（c）2014-2019 heapwolf    <https://github.com/heapwolf>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 * 由自由软件基金会发布，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的，但没有任何保证；甚至没有暗示的保证适用于特定目的。有关更多详细信息，请参见GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HTTPCLIENT_H
#define XMRIG_HTTPCLIENT_H

#include "base/kernel/interfaces/IDnsListener.h"
#include "base/kernel/interfaces/ITimerListener.h"
#include "base/net/http/Fetch.h"
#include "base/net/http/HttpContext.h"
#include "base/tools/Object.h"

namespace xmrig {

class DnsRequest;

class HttpClient : public HttpContext, public IDnsListener, public ITimerListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(HttpClient);

    // 构造函数，初始化 HttpClient 对象
    HttpClient(const char *tag, FetchRequest &&req, const std::weak_ptr<IHttpListener> &listener);
    // 虚析构函数
    ~HttpClient() override = default;

    // 返回是否安静模式
    inline bool isQuiet() const                 { return m_req.quiet; }
    // 返回主机名
    inline const char *host() const override    { return m_req.host; }
    // 返回标签
    inline const char *tag() const              { return m_tag; }
    // 返回端口号
    inline uint16_t port() const override       { return m_req.port; }

    // 连接到服务器
    bool connect();

protected:
    // DNS 解析完成后的回调函数
    void onResolved(const DnsRecords &records, int status, const char *error) override;
    // 定时器超时后的回调函数
    void onTimer(const Timer *timer) override;

    // 握手函数
    virtual void handshake();
    // 定义一个虚拟函数，用于读取指定大小的数据
    virtual void read(const char *data, size_t size);
# 定义一个保护成员函数，返回 m_req 成员变量的引用
protected:
    inline const FetchRequest &req() const  { return m_req; }

# 定义一个私有静态成员函数，用于处理连接事件
private:
    static void onConnect(uv_connect_t *req, int status);

# 声明一个指向常量字符的指针成员变量 m_tag
    const char *m_tag;
# 声明一个成员变量 m_req，类型为 FetchRequest
    FetchRequest m_req;
# 声明一个指向 DnsRequest 对象的共享指针成员变量 m_dns
    std::shared_ptr<DnsRequest> m_dns;
# 声明一个指向 Timer 对象的共享指针成员变量 m_timer
    std::shared_ptr<Timer> m_timer;
};

# 结束命名空间 xmrig
} // namespace xmrig

# 结束条件编译指令，关闭头文件的定义
#endif // XMRIG_HTTPCLIENT_H
```