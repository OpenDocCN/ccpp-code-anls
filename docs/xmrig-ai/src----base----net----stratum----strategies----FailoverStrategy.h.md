# `xmrig\src\base\net\stratum\strategies\FailoverStrategy.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 要么是许可证的第3版，要么（在您的选择下）是任何更高版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_FAILOVERSTRATEGY_H
#define XMRIG_FAILOVERSTRATEGY_H


#include <vector>


#include "base/kernel/interfaces/IClientListener.h"
#include "base/kernel/interfaces/IStrategy.h"
#include "base/net/stratum/Pool.h"
#include "base/tools/Object.h"


namespace xmrig {


class Client;
class IStrategyListener;


class FailoverStrategy : public IStrategy, public IClientListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(FailoverStrategy)

    FailoverStrategy(const std::vector<Pool> &pool, int retryPause, int retries, IStrategyListener *listener, bool quiet = false);
    FailoverStrategy(int retryPause, int retries, IStrategyListener *listener, bool quiet = false);
    ~FailoverStrategy() override;

    void add(const Pool &pool);

protected:
    inline bool isActive() const override           { return m_active >= 0; }
    inline IClient *client() const override         { return isActive() ? active() : m_pools[m_index]; }

    int64_t submit(const JobResult &result) override;
    void connect() override;
    void resume() override;
    void setAlgo(const Algorithm &algo) override;
    void setProxy(const ProxyUrl &proxy) override;
    // 声明一个重写的停止函数
    void stop() override;
    // 声明一个重写的时钟函数，传入当前时间
    void tick(uint64_t now) override;

    // 声明一个重写的客户端关闭函数，传入客户端指针和失败次数
    void onClose(IClient *client, int failures) override;
    // 声明一个重写的接收作业函数，传入客户端指针、作业和参数
    void onJobReceived(IClient *client, const Job &job, const rapidjson::Value &params) override;
    // 声明一个重写的登录函数，传入客户端指针、rapidjson文档和参数
    void onLogin(IClient *client, rapidjson::Document &doc, rapidjson::Value &params) override;
    // 声明一个重写的登录成功函数，传入客户端指针
    void onLoginSuccess(IClient *client) override;
    // 声明一个重写的接受结果函数，传入客户端指针、提交结果和错误信息
    void onResultAccepted(IClient *client, const SubmitResult &result, const char *error) override;
    // 声明一个重写的验证算法函数，传入客户端指针、算法和验证结果指针
    void onVerifyAlgorithm(const IClient *client, const Algorithm &algorithm, bool *ok) override;
private:
    // 返回当前活动的客户端指针
    inline IClient *active() const { return m_pools[static_cast<size_t>(m_active)]; }

    // 是否安静模式
    const bool m_quiet;
    // 重试次数
    const int m_retries;
    // 重试暂停时间
    const int m_retryPause;
    // 当前活动的客户端索引，默认为-1
    int m_active            = -1;
    // 策略监听器指针
    IStrategyListener *m_listener;
    // 索引大小，默认为0
    size_t m_index          = 0;
    // 客户端指针向量
    std::vector<IClient*> m_pools;
};


} /* namespace xmrig */

#endif /* XMRIG_FAILOVERSTRATEGY_H */
```