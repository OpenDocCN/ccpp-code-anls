# `xmrig\src\base\kernel\Platform_hwloc.cpp`

```
/*
 * XMRig
 * 版权所有（c）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据GNU通用公共许可证的条款发布
 * 由自由软件基金会发布的许可证的第3版
 * （在您的选择下）任何后续版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/kernel/Platform.h"
#include "backend/cpu/Cpu.h"


#include <hwloc.h>
#include <thread>


#ifndef XMRIG_OS_APPLE
// 设置线程亲和性的函数
bool xmrig::Platform::setThreadAffinity(uint64_t cpu_id)
{
    // 获取CPU拓扑信息
    auto topology = Cpu::info()->topology();
    // 根据CPU ID获取处理器对象
    auto pu       = hwloc_get_pu_obj_by_os_index(topology, static_cast<unsigned>(cpu_id));

    // 如果处理器对象为空，则返回false
    if (pu == nullptr) {
        return false;
    }

    // 尝试将线程绑定到指定的CPU
    if (hwloc_set_cpubind(topology, pu->cpuset, HWLOC_CPUBIND_THREAD | HWLOC_CPUBIND_STRICT) >= 0) {
        // 等待1毫秒
        std::this_thread::sleep_for(std::chrono::milliseconds(1));
        return true;
    }

    // 如果严格绑定失败，则尝试非严格绑定
    const bool result = (hwloc_set_cpubind(topology, pu->cpuset, HWLOC_CPUBIND_THREAD) >= 0);
    // 等待1毫秒
    std::this_thread::sleep_for(std::chrono::milliseconds(1));

    return result;
}
#endif
```