# `xmrig\src\backend\common\Worker.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/common/Worker.h"  // 引入Worker类的头文件
#include "base/kernel/Platform.h"    // 引入Platform类的头文件
#include "crypto/common/VirtualMemory.h"  // 引入VirtualMemory类的头文件

// Worker类的构造函数，接受id、affinity和priority作为参数
xmrig::Worker::Worker(size_t id, int64_t affinity, int priority) :
    m_affinity(affinity),  // 初始化成员变量m_affinity
    m_id(id)  // 初始化成员变量m_id
{
    m_node = VirtualMemory::bindToNUMANode(affinity);  // 将affinity绑定到NUMA节点

    Platform::trySetThreadAffinity(affinity);  // 尝试设置线程亲和性
    Platform::setThreadPriority(priority);  // 设置线程优先级
}
```