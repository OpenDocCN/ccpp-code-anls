# `xmrig\src\crypto\common\VirtualMemory_hwloc.cpp`

```cpp
/*
 * XMRig
 * 版权所有（C）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（C）2018-2019 tevador     <tevador@gmail.com>
 * 版权所有（C）2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 许可证的版本为3，或者（在您的选择下）任何更高版本。
 *
 * 本程序是基于希望它有用而分发的，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "crypto/common/VirtualMemory.h"  // 导入虚拟内存模块
#include "backend/cpu/Cpu.h"  // 导入CPU模块
#include "base/io/log/Log.h"  // 导入日志模块

#include <hwloc.h>  // 导入硬件定位模块

// 将虚拟内存绑定到NUMA节点
uint32_t xmrig::VirtualMemory::bindToNUMANode(int64_t affinity)
{
    // 如果亲和力小于0或CPU节点数小于2，则返回0
    if (affinity < 0 || Cpu::info()->nodes() < 2) {
        return 0;
    }

    // 根据亲和力获取处理器对象
    auto pu = hwloc_get_pu_obj_by_os_index(Cpu::info()->topology(), static_cast<unsigned>(affinity));

    // 如果处理器对象为空或无法绑定内存，则记录警告并返回0
    if (pu == nullptr || !Cpu::info()->membind(pu->nodeset)) {
        LOG_WARN("CPU #%02" PRId64 " warning: \"can't bind memory\"", affinity);
        return 0;
    }

    // 返回处理器对象的第一个NUMA节点
    return hwloc_bitmap_first(pu->nodeset);
}
```