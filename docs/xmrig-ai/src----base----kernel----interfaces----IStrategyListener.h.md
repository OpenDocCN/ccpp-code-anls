# `xmrig\src\base\kernel\interfaces\IStrategyListener.h`

```
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
 * 本程序是自由软件：您可以重新发布和/或修改
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的版本3，或者
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ISTRATEGYLISTENER_H
#define XMRIG_ISTRATEGYLISTENER_H


#include "3rdparty/rapidjson/fwd.h"
#include "base/tools/Object.h"


namespace xmrig {


class Algorithm;
class IClient;
class IStrategy;
class Job;
class SubmitResult;


class IStrategyListener
{
public:
    XMRIG_DISABLE_COPY_MOVE(IStrategyListener);

    IStrategyListener()             = default;
    virtual ~IStrategyListener()    = default;

    virtual void onActive(IStrategy *strategy, IClient *client)                                                        = 0;
    virtual void onJob(IStrategy *strategy, IClient *client, const Job &job, const rapidjson::Value &params)           = 0;
    # 当登录时触发的回调函数，传入策略对象、客户端对象、JSON文档和参数值
    virtual void onLogin(IStrategy *strategy, IClient *client, rapidjson::Document &doc, rapidjson::Value &params)     = 0;
    # 当策略暂停时触发的回调函数，传入策略对象
    virtual void onPause(IStrategy *strategy)                                                                          = 0;
    # 当结果被接受时触发的回调函数，传入策略对象、客户端对象、提交结果和错误信息
    virtual void onResultAccepted(IStrategy *strategy, IClient *client, const SubmitResult &result, const char *error) = 0;
    # 当验证算法时触发的回调函数，传入策略对象、客户端对象、算法对象和验证结果
    virtual void onVerifyAlgorithm(IStrategy *strategy, const IClient *client, const Algorithm &algorithm, bool *ok)   = 0;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明当前代码块在 xmrig 命名空间中

#endif // XMRIG_ISTRATEGYLISTENER_H
// 结束了 XMRIG_ISTRATEGYLISTENER_H 的定义，并且标记了条件编译的结束
```