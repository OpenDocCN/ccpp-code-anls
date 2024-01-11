# `xmrig\src\net\strategies\DonateStrategy.h`

```
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，发布本程序的副本
 *   或者（根据您的选择）任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的适用性。详细了解
 *   GNU 通用公共许可证的更多细节。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_DONATESTRATEGY_H
#define XMRIG_DONATESTRATEGY_H


#include "base/kernel/interfaces/IClientListener.h"
#include "base/kernel/interfaces/IStrategy.h"
#include "base/kernel/interfaces/IStrategyListener.h"
#include "base/kernel/interfaces/ITimerListener.h"
#include "base/net/stratum/Pool.h"
#include "base/tools/Buffer.h"


namespace xmrig {


class Client;
class Controller;


// 捐赠策略类，继承自IStrategy、IStrategyListener、ITimerListener和IClientListener
class DonateStrategy : public IStrategy, public IStrategyListener, public ITimerListener, public IClientListener
{
public:
    // 禁用默认的拷贝和移动构造函数
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(DonateStrategy)

    // 构造函数，接受一个Controller指针和一个IStrategyListener指针作为参数
    DonateStrategy(Controller *controller, IStrategyListener *listener);
    // 析构函数
    ~DonateStrategy() override;

    // 更新策略，接受一个IClient指针和一个Job对象作为参数
    void update(IClient *client, const Job &job);

protected:
    // 判断策略是否处于活动状态
    inline bool isActive() const override                                                                              { return state() == STATE_ACTIVE; }
    // 获取策略的IClient指针
    inline IClient *client() const override                                                                            { return m_proxy ? m_proxy : m_strategy->client(); }
    # 当接收到作业时，设置作业信息
    inline void onJob(IStrategy *, IClient *client, const Job &job, const rapidjson::Value &params) override           { setJob(client, job, params); }
    
    # 当接收到作业时，设置作业信息
    inline void onJobReceived(IClient *client, const Job &job, const rapidjson::Value &params) override                { setJob(client, job, params); }
    
    # 当结果被接受时，设置结果信息
    inline void onResultAccepted(IClient *client, const SubmitResult &result, const char *error) override              { setResult(client, result, error); }
    
    # 当结果被接受时，设置结果信息
    inline void onResultAccepted(IStrategy *, IClient *client, const SubmitResult &result, const char *error) override { setResult(client, result, error); }
    
    # 恢复策略运行
    inline void resume() override                                                                                      {}
    
    # 提交作业结果
    int64_t submit(const JobResult &result) override;
    
    # 连接到服务器
    void connect() override;
    
    # 设置算法
    void setAlgo(const Algorithm &algo) override;
    
    # 设置代理
    void setProxy(const ProxyUrl &proxy) override;
    
    # 停止策略运行
    void stop() override;
    
    # 每个时间片的处理函数
    void tick(uint64_t now) override;
    
    # 当策略激活时的处理函数
    void onActive(IStrategy *strategy, IClient *client) override;
    
    # 当策略暂停时的处理函数
    void onPause(IStrategy *strategy) override;
    
    # 当连接关闭时的处理函数
    void onClose(IClient *client, int failures) override;
    
    # 当登录成功时的处理函数
    void onLogin(IClient *client, rapidjson::Document &doc, rapidjson::Value &params) override;
    
    # 当登录成功时的处理函数
    void onLogin(IStrategy *strategy, IClient *client, rapidjson::Document &doc, rapidjson::Value &params) override;
    
    # 当验证算法时的处理函数
    void onVerifyAlgorithm(const IClient *client, const Algorithm &algorithm, bool *ok) override;
    
    # 当验证算法时的处理函数
    void onVerifyAlgorithm(IStrategy *strategy, const  IClient *client, const Algorithm &algorithm, bool *ok) override;
    
    # 定时器触发时的处理函数
    void onTimer(const Timer *timer) override;
// 私有成员变量
private:
    // 状态枚举
    enum State {
        STATE_NEW,      // 新建状态
        STATE_IDLE,     // 空闲状态
        STATE_CONNECT,  // 连接状态
        STATE_ACTIVE,   // 活动状态
        STATE_WAIT      // 等待状态
    };

    // 返回当前状态
    inline State state() const { return m_state; }

    // 创建代理
    IClient *createProxy();
    // 空闲状态
    void idle(double min, double max);
    // 设置作业
    void setJob(IClient *client, const Job &job, const rapidjson::Value &params);
    // 设置参数
    void setParams(rapidjson::Document &doc, rapidjson::Value &params);
    // 设置结果
    void setResult(IClient *client, const SubmitResult &result, const char *error);
    // 设置状态
    void setState(State state);

    // 算法
    Algorithm m_algorithm;
    // 是否使用 TLS
    bool m_tls                      = false;
    // 种子
    Buffer m_seed;
    // 用户ID
    char m_userId[65]               = { 0 };
    // 捐赠时间
    const uint64_t m_donateTime;
    // 空闲时间
    const uint64_t m_idleTime;
    // 控制器
    Controller *m_controller;
    // 代理
    IClient *m_proxy                = nullptr;
    // 策略
    IStrategy *m_strategy           = nullptr;
    // 策略监听器
    IStrategyListener *m_listener;
    // 状态
    State m_state                   = STATE_NEW;
    // 矿池列表
    std::vector<Pool> m_pools;
    // 定时器
    Timer *m_timer                  = nullptr;
    // 难度
    uint64_t m_diff                 = 0;
    // 高度
    uint64_t m_height               = 0;
    // 当前时间
    uint64_t m_now                  = 0;
    // 时间戳
    uint64_t m_timestamp            = 0;
};


} // namespace xmrig


#endif // XMRIG_DONATESTRATEGY_H
```