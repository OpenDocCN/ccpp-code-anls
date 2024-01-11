# `xmrig\src\core\Controller.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CONTROLLER_H
#define XMRIG_CONTROLLER_H


#include "base/kernel/Base.h"


#include <memory>


namespace xmrig {


class HwApi;
class Job;
class Miner;
class Network;


class Controller : public Base
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Controller)

    // 构造函数，接受一个进程指针
    Controller(Process *process);
    // 析构函数
    ~Controller() override;

    // 初始化函数
    int init() override;
    // 启动函数
    void start() override;
    // 停止函数
    void stop() override;

    // 获取矿工指针
    Miner *miner() const;
    // 获取网络指针
    Network *network() const;
    // 执行命令
    void execCommand(char command) const;

private:
    // 矿工指针
    std::shared_ptr<Miner> m_miner;
    // 网络指针
    std::shared_ptr<Network> m_network;

#   ifdef XMRIG_FEATURE_API
    // 硬件API指针
    std::shared_ptr<HwApi> m_hwApi;
#   endif
};


} // namespace xmrig


#endif /* XMRIG_CONTROLLER_H */
```