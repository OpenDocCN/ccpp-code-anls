# `xmrig\src\hw\msr\Msr.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据 GNU 通用公共许可证的条款，发布
 *   版本 3 或 (根据您的选择) 任何更高版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有暗示的担保
 *   适销性或特定用途的适用性。详细了解
 *   GNU 通用公共许可证的更多细节。
 *
 *   如果没有，您应该收到 GNU 通用公共许可证的副本
 *   这个程序。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */

#include "hw/msr/Msr.h"
#include "base/io/log/Log.h"


namespace xmrig {


static const char *kTag = YELLOW_BG_BOLD(WHITE_BOLD_S " msr     ");
static std::weak_ptr<Msr> instance;


} // namespace xmrig



const char *xmrig::Msr::tag()
{
    return kTag;
}



std::shared_ptr<xmrig::Msr> xmrig::Msr::get()
{
    auto msr = instance.lock();
    if (!msr) {
        msr      = std::make_shared<Msr>();
        instance = msr;
    }

    if (msr->isAvailable()) {
        return msr;
    }

    return {};
}


bool xmrig::Msr::write(uint32_t reg, uint64_t value, int32_t cpu, uint64_t mask, bool verbose)
{
    // 如果掩码不等于 MsrItem::kNoMask，则进行掩码操作
    if (mask != MsrItem::kNoMask) {
        uint64_t old_value = 0;
        // 读取寄存器的当前值
        if (rdmsr(reg, cpu, old_value)) {
            // 使用掩码对新值进行掩码操作
            value = MsrItem::maskedValue(old_value, value, mask);
        }
    }

    // 将新值写入寄存器
    const bool result = wrmsr(reg, value, cpu);
    // 如果写入失败且 verbose 为真，则输出警告信息
    if (!result && verbose) {
        LOG_WARN("%s " YELLOW_BOLD("cannot set MSR 0x%08" PRIx32 " to 0x%016" PRIx64), tag(), reg, value);
    }

    return result;
}


xmrig::MsrItem xmrig::Msr::read(uint32_t reg, int32_t cpu, bool verbose) const
{
    uint64_t value = 0;
    # 如果成功读取寄存器的值
    if (rdmsr(reg, cpu, value)) {
        # 返回寄存器和值的字典
        return { reg, value };
    }
    
    # 如果开启了详细输出
    if (verbose) {
        # 输出警告信息，指示无法读取指定的 MSR 寄存器
        LOG_WARN("%s " YELLOW_BOLD("cannot read MSR 0x%08" PRIx32), tag(), reg);
    }
    
    # 返回空字典
    return {};
# 闭合前面的函数定义
```