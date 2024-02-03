# `xmrig\src\backend\common\interfaces\IWorker.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 其中包括许可证的第3版或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IWORKER_H
#define XMRIG_IWORKER_H


#include "base/tools/Object.h"


#include <cstdint>
#include <cstddef>


namespace xmrig {


class Job;
class VirtualMemory;


class IWorker
{
public:
    XMRIG_DISABLE_COPY_MOVE(IWorker)

    IWorker()           = default;
    virtual ~IWorker()  = default;

    // 进行自检，返回布尔值
    virtual bool selfTest()                                                                         = 0;
    // 返回虚拟内存的指针
    virtual const VirtualMemory *memory() const                                                     = 0;
    // 返回工作器的ID
    virtual size_t id() const                                                                       = 0;
    // 返回工作器的强度
    virtual size_t intensity() const                                                                = 0;
    // 返回工作器的线程数
    virtual size_t threads() const                                                                  = 0;
    // 返回哈希率数据
    virtual void hashrateData(uint64_t &hashCount, uint64_t &timeStamp, uint64_t &rawHashes) const  = 0;
    // 通知工作器新的工作
    virtual void jobEarlyNotification(const Job &job)                                               = 0;
    // 启动工作器
    virtual void start()                                                                            = 0;
};
} // 结束 xmrig 命名空间
#endif // 结束 XMRIG_IWORKER_H
```