# `xmrig\src\crypto\common\HugePagesInfo.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "crypto/common/HugePagesInfo.h"
#include "crypto/common/VirtualMemory.h"


xmrig::HugePagesInfo::HugePagesInfo(const VirtualMemory *memory)
{
    // 如果内存是1GB页面
    if (memory->isOneGbPages()) {
        // 将大小对齐到1GB
        size        = VirtualMemory::align(memory->size(), VirtualMemory::kOneGiB);
        // 计算总共有多少个1GB页面
        total       = size / VirtualMemory::kOneGiB;
        // 已分配的1GB页面数量等于总共的1GB页面数量
        allocated   = size / VirtualMemory::kOneGiB;
    }
    else {
        // 将大小对齐到巨大页面大小
        size        = VirtualMemory::alignToHugePageSize(memory->size());
        // 计算总共有多少个巨大页面
        total       = size / VirtualMemory::hugePageSize();
        // 如果内存使用了巨大页面，则已分配的巨大页面数量等于总共的巨大页面数量，否则为0
        allocated   = memory->isHugePages() ? total : 0;
    }
}
```