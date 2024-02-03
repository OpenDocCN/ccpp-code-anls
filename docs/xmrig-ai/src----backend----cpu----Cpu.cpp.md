# `xmrig\src\backend\cpu\Cpu.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include <cassert>


#include "backend/cpu/Cpu.h"
#include "3rdparty/rapidjson/document.h"


#if defined(XMRIG_FEATURE_HWLOC)
#   include "backend/cpu/platform/HwlocCpuInfo.h"
#else
#   include "backend/cpu/platform/BasicCpuInfo.h"
#endif


static xmrig::ICpuInfo *cpuInfo = nullptr;


// 返回 CPU 信息对象
xmrig::ICpuInfo *xmrig::Cpu::info()
{
    // 如果 CPU 信息对象为空
    if (cpuInfo == nullptr) {
#       if defined(XMRIG_FEATURE_HWLOC)
        // 创建 HwlocCpuInfo 对象
        cpuInfo = new HwlocCpuInfo();
#       else
        // 创建 BasicCpuInfo 对象
        cpuInfo = new BasicCpuInfo();
#       endif
    }

    // 返回 CPU 信息对象
    return cpuInfo;
}


// 将 CPU 信息转换为 JSON 格式
rapidjson::Value xmrig::Cpu::toJSON(rapidjson::Document &doc)
{
    // 调用 info() 方法获取 CPU 信息对象，然后将其转换为 JSON 格式
    return info()->toJSON(doc);
}


// 释放 CPU 信息对象
void xmrig::Cpu::release()
{
    // 删除 CPU 信息对象
    delete cpuInfo;
    // 将 CPU 信息对象指针置为空
    cpuInfo = nullptr;
}
```