# `xmrig\src\backend\cuda\interfaces\ICudaRunner.h`

```
/*
 * XMRig - 代码版权声明
 * 该程序是自由软件：您可以重新分发和/或修改它
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版或
 * （根据您的选择）任何以后的版本。
 *
 * 该程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证。
 * 有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_ICUDARUNNER_H
#define XMRIG_ICUDARUNNER_H

#include "base/tools/Object.h"  // 引入Object类

#include <cstdint>  // 引入标准整数类型

namespace xmrig {

class Job;  // 声明Job类

class ICudaRunner  // 定义ICudaRunner类
{
public:
    XMRIG_DISABLE_COPY_MOVE(ICudaRunner)  // 禁用拷贝和移动构造函数

    ICudaRunner()          = default;  // 默认构造函数
    virtual ~ICudaRunner() = default;  // 虚析构函数

    virtual size_t intensity() const                                                = 0;  // 虚函数，返回强度
    virtual size_t roundSize() const                                                = 0;  // 虚函数，返回轮大小
    virtual size_t processedHashes() const                                          = 0;  // 虚函数，返回处理的哈希数
    virtual bool init()                                                             = 0;  // 虚函数，初始化
    virtual bool run(uint32_t startNonce, uint32_t *rescount, uint32_t *resnonce)   = 0;  // 虚函数，运行
    # 声明一个虚拟函数，用于设置作业和相关数据，返回布尔类型值
    virtual bool set(const Job &job, uint8_t *blob)                                 = 0;
    # 声明一个虚拟函数，用于作业的提前通知，不返回数值
    virtual void jobEarlyNotification(const Job&)                                   = 0;
}; 
// 结束了 xmrig 命名空间的定义

} /* namespace xmrig */
// 声明代码块结束了 xmrig 命名空间的定义

#endif // XMRIG_ICUDARUNNER_H
// 结束了对 XMRIG_ICUDARUNNER_H 的 ifndef 条件编译
```