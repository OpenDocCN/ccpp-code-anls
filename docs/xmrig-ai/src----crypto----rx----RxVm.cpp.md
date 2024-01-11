# `xmrig\src\crypto\rx\RxVm.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新发布和/或修改
 *   根据 GNU 通用公共许可证的条款，版本 3 或
 *   （根据您的选择）任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的保证。详细了解
 *   GNU 通用公共许可证的更多细节。
 *
 *   如果没有收到 GNU 通用公共许可证的副本
 *   请参阅 <http://www.gnu.org/licenses/>。
 */


#include "crypto/randomx/randomx.h"
#include "backend/cpu/Cpu.h"
#include "crypto/rx/RxCache.h"
#include "crypto/rx/RxDataset.h"
#include "crypto/rx/RxVm.h"


// 创建 RandomX 虚拟机
randomx_vm *xmrig::RxVm::create(RxDataset *dataset, uint8_t *scratchpad, bool softAes, const Assembly &assembly, uint32_t node)
{
    // 初始化标志位
    int flags = 0;

    // 如果不使用软件 AES，则设置 HARD_AES 标志位
    if (!softAes) {
       flags |= RANDOMX_FLAG_HARD_AES;
    }

    // 如果数据集已存在，则设置 FULL_MEM 标志位
    if (dataset->get()) {
        flags |= RANDOMX_FLAG_FULL_MEM;
    }

    // 如果数据集缓存不存在或者是 JIT 编译，则设置 JIT 标志位
    if (!dataset->cache() || dataset->cache()->isJIT()) {
        flags |= RANDOMX_FLAG_JIT;
    }

    // 根据给定的 Assembly 类型设置 AMD 标志位
    const auto asmId = assembly == Assembly::AUTO ? Cpu::info()->assembly() : assembly.id();
    if ((asmId == Assembly::RYZEN) || (asmId == Assembly::BULLDOZER)) {
        flags |= RANDOMX_FLAG_AMD;
    }

    // 创建 RandomX 虚拟机
    return randomx_create_vm(static_cast<randomx_flags>(flags), !dataset->get() ? dataset->cache()->get() : nullptr, dataset->get(), scratchpad, node);
}


// 销毁 RandomX 虚拟机
void xmrig::RxVm::destroy(randomx_vm* vm)
{
    // 如果虚拟机存在，则销毁虚拟机
    if (vm) {
        randomx_destroy_vm(vm);
    }
}
```