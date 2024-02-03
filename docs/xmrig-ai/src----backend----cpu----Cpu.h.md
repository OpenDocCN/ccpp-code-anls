# `xmrig\src\backend\cpu\Cpu.h`

```cpp
/*
 * XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布和/或修改
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款
 * 许可证的第3版或
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

#ifndef XMRIG_CPU_H
#define XMRIG_CPU_H

#include "backend/cpu/interfaces/ICpuInfo.h"

namespace xmrig {

class Cpu
{
public:
    static ICpuInfo *info();
    static rapidjson::Value toJSON(rapidjson::Document &doc);
    static void release();

    inline static Assembly::Id assembly(Assembly::Id hint) { return hint == Assembly::AUTO ? Cpu::info()->assembly() : hint; }
};

} // namespace xmrig

#endif // XMRIG_CPU_H
```