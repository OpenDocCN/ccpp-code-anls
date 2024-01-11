# `xmrig\src\crypto\common\HugePagesInfo.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证
 *   商品性或适用于特定目的。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_HUGEPAGESINFO_H
#define XMRIG_HUGEPAGESINFO_H


#include <cstdint>
#include <cstddef>


namespace xmrig {


class VirtualMemory;


class HugePagesInfo
{
public:
    HugePagesInfo() = default;
    HugePagesInfo(const VirtualMemory *memory);

    size_t allocated    = 0;  // 已分配的大页数
    size_t total        = 0;  // 总大页数
    size_t size         = 0;  // 大页的大小

    inline bool isFullyAllocated() const { return allocated == total; }  // 检查是否完全分配
    inline double percent() const        { return total == 0 ? 0.0 : static_cast<double>(allocated) / total * 100.0; }  // 计算已分配大页的百分比
    inline void reset()                  { allocated = 0; total = 0; size = 0; }  // 重置已分配、总数和大小

    inline HugePagesInfo &operator+=(const HugePagesInfo &other)
    {
        allocated += other.allocated;  // 累加已分配的大页数
        total     += other.total;      // 累加总大页数
        size      += other.size;       // 累加大页的大小

        return *this;
    }
};


} /* namespace xmrig */


#endif /* XMRIG_HUGEPAGESINFO_H */
```