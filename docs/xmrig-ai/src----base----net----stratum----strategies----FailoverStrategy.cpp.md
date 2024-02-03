# `xmrig\src\base\net\stratum\strategies\FailoverStrategy.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
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


#include "base/net/stratum/strategies/FailoverStrategy.h"
#include "3rdparty/rapidjson/document.h"
#include "base/kernel/interfaces/IClient.h"
#include "base/kernel/interfaces/IStrategyListener.h"
#include "base/kernel/Platform.h"


xmrig::FailoverStrategy::FailoverStrategy(const std::vector<Pool> &pools, int retryPause, int retries, IStrategyListener *listener, bool quiet) :
    m_quiet(quiet),  // 初始化成员变量 quiet
    m_retries(retries),  // 初始化成员变量 retries
    m_retryPause(retryPause),  // 初始化成员变量 retryPause
    m_listener(listener)  // 初始化成员变量 listener
{
    for (const Pool &pool : pools) {  // 遍历传入的 pools
        add(pool);  // 调用 add 方法添加 pool
    }
}


xmrig::FailoverStrategy::FailoverStrategy(int retryPause, int retries, IStrategyListener *listener, bool quiet) :
    m_quiet(quiet),  // 初始化成员变量 quiet
    m_retries(retries),  // 初始化成员变量 retries
    m_retryPause(retryPause),  // 初始化成员变量 retryPause
    m_listener(listener)  // 初始化成员变量 listener
{
}


xmrig::FailoverStrategy::~FailoverStrategy()
{
    for (IClient *client : m_pools) {  // 遍历 m_pools
        client->deleteLater();  // 调用 deleteLater 方法
    }
}


void xmrig::FailoverStrategy::add(const Pool &pool)
{
    IClient *client = pool.createClient(static_cast<int>(m_pools.size()), this);  // 创建一个客户端

    client->setRetries(m_retries);  // 设置客户端的重试次数
    client->setRetryPause(m_retryPause * 1000);  // 设置客户端的重试暂停时间
    client->setQuiet(m_quiet);  // 设置客户端的安静模式
    # 将client对象添加到m_pools中
    m_pools.push_back(client);
// 提交作业结果，如果策略不活跃则返回-1
int64_t xmrig::FailoverStrategy::submit(const JobResult &result)
{
    if (!isActive()) {
        return -1;
    }
    // 调用当前活跃池的submit方法
    return active()->submit(result);
}


// 连接到当前索引指向的矿池
void xmrig::FailoverStrategy::connect()
{
    m_pools[m_index]->connect();
}


// 恢复活跃状态，如果不活跃则返回
void xmrig::FailoverStrategy::resume()
{
    if (!isActive()) {
        return;
    }
    // 调用监听器的onJob方法
    m_listener->onJob(this, active(), active()->job(), rapidjson::Value(rapidjson::kNullType));
}


// 设置所有矿池的算法
void xmrig::FailoverStrategy::setAlgo(const Algorithm &algo)
{
    for (IClient *client : m_pools) {
        client->setAlgo(algo);
    }
}


// 设置所有矿池的代理
void xmrig::FailoverStrategy::setProxy(const ProxyUrl &proxy)
{
    for (IClient *client : m_pools) {
        client->setProxy(proxy);
    }
}


// 停止所有矿池的连接，并重置索引和活跃状态
void xmrig::FailoverStrategy::stop()
{
    for (auto &pool : m_pools) {
        pool->disconnect();
    }

    m_index  = 0;
    m_active = -1;

    m_listener->onPause(this);
}


// 对所有矿池执行tick操作
void xmrig::FailoverStrategy::tick(uint64_t now)
{
    for (IClient *client : m_pools) {
        client->tick(now);
    }
}


// 当矿池连接关闭时的处理逻辑
void xmrig::FailoverStrategy::onClose(IClient *client, int failures)
{
    if (failures == -1) {
        return;
    }

    if (m_active == client->id()) {
        m_active = -1;
        m_listener->onPause(this);
    }

    if (m_index == 0 && failures < m_retries) {
        return;
    }

    if (m_index == static_cast<size_t>(client->id()) && (m_pools.size() - m_index) > 1) {
        m_pools[++m_index]->connect();
    }
}


// 当矿池登录成功时的处理逻辑
void xmrig::FailoverStrategy::onLoginSuccess(IClient *client)
{
    int active = m_active;

    if (client->id() == 0 || !isActive()) {
        active = client->id();
    }
}
    }
    // 遍历连接池列表，从第二个连接池开始
    for (size_t i = 1; i < m_pools.size(); ++i) {
        // 如果当前连接池不是活动连接池
        if (active != static_cast<int>(i)) {
            // 断开当前连接池的连接
            m_pools[i]->disconnect();
        }
    }
    // 如果活动连接池的索引大于等于0且不等于当前活动连接池的索引
    if (active >= 0 && active != m_active) {
        // 更新活动连接池的索引
        m_index = m_active = active;
        // 调用监听器的onActive方法，通知活动连接池的变化
        m_listener->onActive(this, client);
    }
# 结束 FailoverStrategy 类的定义，包括 onResultAccepted 和 onVerifyAlgorithm 方法的实现
}

# 当结果被接受时调用的方法，将结果传递给监听器
void xmrig::FailoverStrategy::onResultAccepted(IClient *client, const SubmitResult &result, const char *error)
{
    m_listener->onResultAccepted(this, client, result, error);
}

# 当验证算法时调用的方法，将算法和结果传递给监听器
void xmrig::FailoverStrategy::onVerifyAlgorithm(const IClient *client, const Algorithm &algorithm, bool *ok)
{
    m_listener->onVerifyAlgorithm(this, client, algorithm, ok);
}
```