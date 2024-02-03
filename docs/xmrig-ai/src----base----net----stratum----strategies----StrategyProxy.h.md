# `xmrig\src\base\net\stratum\strategies\StrategyProxy.h`

```cpp
/* XMRig
 * 版权声明
 */

#ifndef XMRIG_STRATEGYPROXY_H
#define XMRIG_STRATEGYPROXY_H

#include "base/kernel/interfaces/IStrategyListener.h"
// 引入IStrategyListener接口

namespace xmrig {
// 命名空间xmrig

class StrategyProxy : public IStrategyListener
// 定义StrategyProxy类，继承自IStrategyListener接口
{
public:
    inline StrategyProxy(IStrategyListener *listener) : m_listener(listener) {}
    // 构造函数，接受IStrategyListener指针作为参数，并初始化m_listener成员变量

protected:
    inline void onActive(IStrategy *strategy, IClient *client) override
    // 重写IStrategyListener接口的onActive方法
    {
        m_listener->onActive(strategy, client);
        // 调用m_listener的onActive方法
    }

    inline void onJob(IStrategy *strategy, IClient *client, const Job &job, const rapidjson::Value &params) override
    // 重写IStrategyListener接口的onJob方法
    {
        m_listener->onJob(strategy, client, job, params);
        // 调用m_listener的onJob方法
    }

    inline void onLogin(IStrategy *strategy, IClient *client, rapidjson::Document &doc, rapidjson::Value &params) override
    // 重写IStrategyListener接口的onLogin方法
    {
        // 调用监听器的登录方法，传入策略、客户端、文档和参数
        m_listener->onLogin(strategy, client, doc, params);
    }

    // 重写暂停方法
    inline void onPause(IStrategy *strategy) override
    {
        // 调用监听器的暂停方法，传入策略
        m_listener->onPause(strategy);
    }

    // 重写结果接受方法
    inline void onResultAccepted(IStrategy *strategy, IClient *client, const SubmitResult &result, const char *error) override
    {
        // 调用监听器的结果接受方法，传入策略、客户端、结果和错误信息
        m_listener->onResultAccepted(strategy, client, result, error);
    }

    // 重写验证算法方法
    inline void onVerifyAlgorithm(IStrategy *strategy, const IClient *client, const Algorithm &algorithm, bool *ok) override
    {
        // 调用监听器的验证算法方法，传入策略、客户端、算法和验证结果
        m_listener->onVerifyAlgorithm(strategy, client, algorithm, ok);
    }
# 声明私有成员变量，用于存储策略监听器的指针
private:
    IStrategyListener *m_listener;
};  # 结束类定义

} /* namespace xmrig */  # 结束命名空间定义

#endif /* XMRIG_STRATEGYPROXY_H */  # 结束头文件定义
```