# `xmrig\src\backend\common\interfaces\IMemoryPool.h`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 它，无论是许可证的第3版，还是
 * （在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_IMEMORYPOOL_H
#define XMRIG_IMEMORYPOOL_H


#include "base/tools/Object.h"


#include <cstddef>
#include <cstdint>


namespace xmrig {


class IMemoryPool
{
public:
    XMRIG_DISABLE_COPY_MOVE(IMemoryPool)

    IMemoryPool()           = default;
    virtual ~IMemoryPool()  = default;

    virtual bool isHugePages(uint32_t node) const       = 0;
    virtual uint8_t *get(size_t size, uint32_t node)    = 0;
    virtual void release(uint32_t node)                 = 0;
};


} /* namespace xmrig */



#endif /* XMRIG_IMEMORYPOOL_H */
```