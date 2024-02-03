# `xmrig\src\base\net\stratum\strategies\SinglePoolStrategy.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "base/net/stratum/strategies/SinglePoolStrategy.h"
#include "3rdparty/rapidjson/document.h"
#include "base/kernel/interfaces/IClient.h"
#include "base/kernel/interfaces/IStrategyListener.h"
#include "base/kernel/Platform.h"
#include "base/net/stratum/Pool.h"

// 使用命名空间 xmrig
xmrig::SinglePoolStrategy::SinglePoolStrategy(const Pool &pool, int retryPause, int retries, IStrategyListener *listener, bool quiet) :
    m_active(false), // 初始化 m_active 为 false
    m_listener(listener) // 初始化 m_listener 为传入的 listener
{
    // 从给定的 pool 创建客户端
    m_client = pool.createClient(0, this);
    // 设置客户端的重试次数
    m_client->setRetries(retries);
    // 设置客户端的重试暂停时间
    m_client->setRetryPause(retryPause * 1000);
    // 设置客户端的静默模式
    m_client->setQuiet(quiet);
}

// 析构函数
xmrig::SinglePoolStrategy::~SinglePoolStrategy()
{
    // 延迟删除客户端
    m_client->deleteLater();
}
// 提交作业结果给矿池客户端
int64_t xmrig::SinglePoolStrategy::submit(const JobResult &result)
{
    return m_client->submit(result);
}


// 连接到矿池
void xmrig::SinglePoolStrategy::connect()
{
    m_client->connect();
}


// 恢复策略
void xmrig::SinglePoolStrategy::resume()
{
    // 如果策略不活跃，则返回
    if (!isActive()) {
        return;
    }
    // 触发监听器的 onJob 事件
    m_listener->onJob(this, m_client, m_client->job(), rapidjson::Value(rapidjson::kNullType));
}


// 设置算法
void xmrig::SinglePoolStrategy::setAlgo(const Algorithm &algo)
{
    m_client->setAlgo(algo);
}


// 设置代理
void xmrig::SinglePoolStrategy::setProxy(const ProxyUrl &proxy)
{
    m_client->setProxy(proxy);
}


// 停止策略
void xmrig::SinglePoolStrategy::stop()
{
    m_client->disconnect();
}


// 执行策略的时间片
void xmrig::SinglePoolStrategy::tick(uint64_t now)
{
    m_client->tick(now);
}


// 当客户端关闭时触发的事件
void xmrig::SinglePoolStrategy::onClose(IClient *, int)
{
    // 如果策略不活跃，则返回
    if (!isActive()) {
        return;
    }
    // 设置策略为非活跃状态，并触发监听器的 onPause 事件
    m_active = false;
    m_listener->onPause(this);
}


// 当接收到作业时触发的事件
void xmrig::SinglePoolStrategy::onJobReceived(IClient *client, const Job &job, const rapidjson::Value &params)
{
    // 触发监听器的 onJob 事件
    m_listener->onJob(this, client, job, params);
}


// 当登录时触发的事件
void xmrig::SinglePoolStrategy::onLogin(IClient *client, rapidjson::Document &doc, rapidjson::Value &params)
{
    // 触发监听器的 onLogin 事件
    m_listener->onLogin(this, client, doc, params);
}


// 当登录成功时触发的事件
void xmrig::SinglePoolStrategy::onLoginSuccess(IClient *client)
{
    // 设置策略为活跃状态，并触发监听器的 onActive 事件
    m_active = true;
    m_listener->onActive(this, client);
}


// 当接受到结果被接受时触发的事件
void xmrig::SinglePoolStrategy::onResultAccepted(IClient *client, const SubmitResult &result, const char *error)
{
    // 触发监听器的 onResultAccepted 事件
    m_listener->onResultAccepted(this, client, result, error);
}


// 当验证算法时触发的事件
void xmrig::SinglePoolStrategy::onVerifyAlgorithm(const IClient *client, const Algorithm &algorithm, bool *ok)
{
    // 触发监听器的 onVerifyAlgorithm 事件
    m_listener->onVerifyAlgorithm(this, client, algorithm, ok);
}
```