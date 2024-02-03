# `xmrig\src\base\net\stratum\strategies\SinglePoolStrategy.h`

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
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_SINGLEPOOLSTRATEGY_H
#define XMRIG_SINGLEPOOLSTRATEGY_H


#include "base/kernel/interfaces/IClientListener.h"
#include "base/kernel/interfaces/IStrategy.h"
#include "base/tools/Object.h"


namespace xmrig {


class Client;
class IStrategyListener;
class Pool;


class SinglePoolStrategy : public IStrategy, public IClientListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(SinglePoolStrategy)

    SinglePoolStrategy(const Pool &pool, int retryPause, int retries, IStrategyListener *listener, bool quiet = false);
    ~SinglePoolStrategy() override;

protected:
    inline bool isActive() const override           { return m_active; }
    inline IClient *client() const override         { return m_client; }
    # 提交作业结果的函数，参数为JobResult类型，重写父类的submit方法
    int64_t submit(const JobResult &result) override;
    
    # 连接的函数，重写父类的connect方法
    void connect() override;
    
    # 恢复的函数，重写父类的resume方法
    void resume() override;
    
    # 设置算法的函数，参数为Algorithm类型，重写父类的setAlgo方法
    void setAlgo(const Algorithm &algo) override;
    
    # 设置代理的函数，参数为ProxyUrl类型，重写父类的setProxy方法
    void setProxy(const ProxyUrl &proxy) override;
    
    # 停止的函数，重写父类的stop方法
    void stop() override;
    
    # 时间片的函数，参数为uint64_t类型，重写父类的tick方法
    void tick(uint64_t now) override;
    
    # 关闭的回调函数，参数为IClient指针和失败次数，重写父类的onClose方法
    void onClose(IClient *client, int failures) override;
    
    # 接收作业的回调函数，参数为IClient指针、Job类型和rapidjson::Value类型，重写父类的onJobReceived方法
    void onJobReceived(IClient *client, const Job &job, const rapidjson::Value &params) override;
    
    # 登录的回调函数，参数为IClient指针、rapidjson::Document类型和rapidjson::Value类型，重写父类的onLogin方法
    void onLogin(IClient *client, rapidjson::Document &doc, rapidjson::Value &params) override;
    
    # 登录成功的回调函数，参数为IClient指针，重写父类的onLoginSuccess方法
    void onLoginSuccess(IClient *client) override;
    
    # 接受结果的回调函数，参数为IClient指针、SubmitResult类型和const char*类型，重写父类的onResultAccepted方法
    void onResultAccepted(IClient *client, const SubmitResult &result, const char *error) override;
    
    # 验证算法的回调函数，参数为const IClient指针、Algorithm类型和bool指针，重写父类的onVerifyAlgorithm方法
    void onVerifyAlgorithm(const IClient *client, const Algorithm &algorithm, bool *ok) override;
# 私有成员变量，表示策略是否激活
bool m_active;
# 指向IClient接口的指针，用于处理客户端相关操作
IClient *m_client;
# 指向IStrategyListener接口的指针，用于处理策略监听器相关操作
IStrategyListener *m_listener;
```