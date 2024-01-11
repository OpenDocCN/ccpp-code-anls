# `xmrig\src\net\Network.h`

```
/* XMRig
 * 版权所有 (c) 2019      Howard Chu  <https://github.com/hyc>
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，发布本程序，无论是版本 3 还是
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适销性或特定用途的隐含担保。详见
 *   GNU 通用公共许可证获取更多详细信息。
 *
 *   如果没有收到 GNU 通用公共许可证的副本，
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_NETWORK_H
#define XMRIG_NETWORK_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/api/interfaces/IApiListener.h"
#include "base/kernel/interfaces/IBaseListener.h"
#include "base/kernel/interfaces/IStrategyListener.h"
#include "base/kernel/interfaces/ITimerListener.h"
#include "base/tools/Object.h"
#include "interfaces/IJobResultListener.h"


#include <vector>


namespace xmrig {


class Controller;
class IStrategy;
class NetworkState;


class Network : public IJobResultListener, public IStrategyListener, public IBaseListener, public ITimerListener, public IApiListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Network)

    // 构造函数，接受一个 Controller 对象指针作为参数
    Network(Controller *controller);
    // 析构函数
    ~Network() override;

    // 返回当前 Network 对象的策略对象指针
    inline IStrategy *strategy() const { return m_strategy; }

    // 连接到网络
    void connect();
    // 执行命令
    void execCommand(char command);

protected:
    // 定时器回调函数
    inline void onTimer(const Timer *) override { tick(); }

    // 当策略对象激活时的回调函数
    void onActive(IStrategy *strategy, IClient *client) override;
    // 当配置发生变化时的回调函数
    void onConfigChanged(Config *config, Config *previousConfig) override;


注释：
    # 当有新的任务到达时，调用此函数
    void onJob(IStrategy *strategy, IClient *client, const Job &job, const rapidjson::Value &params) override;
    
    # 当任务执行结果返回时，调用此函数
    void onJobResult(const JobResult &result) override;
    
    # 当策略登录成功时，调用此函数
    void onLogin(IStrategy *strategy, IClient *client, rapidjson::Document &doc, rapidjson::Value &params) override;
    
    # 当策略暂停时，调用此函数
    void onPause(IStrategy *strategy) override;
    
    # 当任务执行结果被接受时，调用此函数
    void onResultAccepted(IStrategy *strategy, IClient *client, const SubmitResult &result, const char *error) override;
    
    # 当需要验证算法时，调用此函数
    void onVerifyAlgorithm(IStrategy *strategy, const  IClient *client, const Algorithm &algorithm, bool *ok) override;
// 如果定义了XMRIG_FEATURE_API宏，则定义一个onRequest函数，该函数接收一个IApiRequest对象作为参数，并在重写时执行
void onRequest(IApiRequest &request) override;

// 定义一个常量kTickInterval，值为1秒的毫秒数
constexpr static int kTickInterval = 1 * 1000;

// 定义一个setJob函数，接收一个IClient指针、一个Job对象和一个布尔值donate作为参数，并在调用时执行
void setJob(IClient *client, const Job &job, bool donate);

// 定义一个tick函数，没有参数，用于执行一些操作
void tick();

// 如果定义了XMRIG_FEATURE_API宏，则定义一个getConnection函数，接收一个rapidjson::Value对象、一个rapidjson::Document对象和一个整数version作为参数，并在调用时执行
void getConnection(rapidjson::Value &reply, rapidjson::Document &doc, int version) const;

// 如果定义了XMRIG_FEATURE_API宏，则定义一个getResults函数，接收一个rapidjson::Value对象、一个rapidjson::Document对象和一个整数version作为参数，并在调用时执行
void getResults(rapidjson::Value &reply, rapidjson::Document &doc, int version) const;

// 定义一个Controller指针m_controller
Controller *m_controller;

// 定义一个IStrategy指针m_donate，并初始化为nullptr
IStrategy *m_donate = nullptr;

// 定义一个IStrategy指针m_strategy，并初始化为nullptr
IStrategy *m_strategy = nullptr;

// 定义一个NetworkState指针m_state，并初始化为nullptr
NetworkState *m_state = nullptr;

// 定义一个Timer指针m_timer，并初始化为nullptr
Timer *m_timer = nullptr;
```