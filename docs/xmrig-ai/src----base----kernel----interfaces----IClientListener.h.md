# `xmrig\src\base\kernel\interfaces\IClientListener.h`

```
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本 3 或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ICLIENTLISTENER_H
#define XMRIG_ICLIENTLISTENER_H


#include <cstdint>


#include "3rdparty/rapidjson/fwd.h"


namespace xmrig {


class Algorithm;
class IClient;
class Job;
class SubmitResult;


class IClientListener
{
public:
    virtual ~IClientListener() = default;

    // 当客户端关闭时调用，参数为客户端指针和失败次数
    virtual void onClose(IClient *client, int failures)                                           = 0;
    // 当接收到作业时调用，参数为客户端指针、作业对象和参数对象
    virtual void onJobReceived(IClient *client, const Job &job, const rapidjson::Value &params)   = 0;
    // 当登录时调用，参数为客户端指针、rapidjson 文档对象和参数对象
    virtual void onLogin(IClient *client, rapidjson::Document &doc, rapidjson::Value &params)     = 0;
    // 当登录成功时调用，参数为客户端指针
    virtual void onLoginSuccess(IClient *client)                                                  = 0;
    # 定义一个虚拟函数，当结果被接受时调用，传入客户端指针、提交结果和错误信息
    virtual void onResultAccepted(IClient *client, const SubmitResult &result, const char *error) = 0;
    # 定义一个虚拟函数，当需要验证算法时调用，传入客户端指针、算法和一个布尔指针用于返回验证结果
    virtual void onVerifyAlgorithm(const IClient *client, const Algorithm &algorithm, bool *ok)   = 0;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明当前代码块在 xmrig 命名空间中

#endif // XMRIG_ICLIENTLISTENER_H
// 结束了 XMRIG_ICLIENTLISTENER_H 的定义，并且标记了条件编译的结束
```