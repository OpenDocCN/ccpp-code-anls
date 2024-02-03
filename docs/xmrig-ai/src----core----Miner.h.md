# `xmrig\src\core\Miner.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，版本为3或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用的
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_MINER_H
#define XMRIG_MINER_H


#include <vector>


#include "backend/common/interfaces/IRxListener.h"
#include "base/api/interfaces/IApiListener.h"
#include "base/crypto/Algorithm.h"
#include "base/kernel/interfaces/IBaseListener.h"
#include "base/kernel/interfaces/ITimerListener.h"
#include "base/tools/Object.h"


namespace xmrig {


class Controller;
class Job;
class MinerPrivate;
class IBackend;


class Miner : public ITimerListener, public IBaseListener, public IApiListener, public IRxListener
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(Miner)  // 禁用默认的拷贝和移动构造函数

    Miner(Controller *controller);  // 构造函数，传入Controller指针
    ~Miner() override;  // 析构函数

    bool isEnabled() const;  // 返回是否启用
    bool isEnabled(const Algorithm &algorithm) const;  // 返回指定算法是否启用
    const Algorithms &algorithms() const;  // 返回算法列表
    const std::vector<IBackend *> &backends() const;  // 返回后端列表
    Job job() const;  // 返回当前作业
    void execCommand(char command);  // 执行命令
    void pause();  // 暂停
    void setEnabled(bool enabled);  // 设置是否启用
    void setJob(const Job &job, bool donate);  // 设置作业
    void stop();  // 停止

protected:
    void onConfigChanged(Config *config, Config *previousConfig) override;  // 配置改变时的回调函数
    void onTimer(const Timer *timer) override;  // 定时器回调函数

#   ifdef XMRIG_FEATURE_API
    // 重写接口的请求方法，接收一个实现了IApiRequest接口的对象的引用
    void onRequest(IApiRequest &request) override;
#   endif
#   ifdef XMRIG_ALGO_RANDOMX
    // 如果定义了 XMRIG_ALGO_RANDOMX，则重写 onDatasetReady 方法
    void onDatasetReady() override;
#   endif
// Miner 类的私有成员变量和方法声明结束
private:
    // 指向 Miner 类的私有成员变量和方法的指针
    MinerPrivate *d_ptr;
};
// 命名空间 xmrig 结束
} // namespace xmrig
// 结束条件编译指令，关闭 XMRIG_MINER_H 头文件
#endif /* XMRIG_MINER_H */
```