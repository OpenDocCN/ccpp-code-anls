# `xmrig\src\hw\msr\Msr.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多详情请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_MSR_H
#define XMRIG_MSR_H


#include "base/tools/Object.h"
#include "hw/msr/MsrItem.h"


#include <functional>
#include <memory>


namespace xmrig
{


class MsrPrivate;


class Msr
{
public:
    XMRIG_DISABLE_COPY_MOVE(Msr)

    using Callback = std::function<bool(int32_t cpu)>;

    Msr();
    ~Msr();

    static const char *tag();
    static std::shared_ptr<Msr> get();

    inline bool write(const MsrItem &item, int32_t cpu = -1, bool verbose = true)   { return write(item.reg(), item.value(), cpu, item.mask(), verbose); }

    bool isAvailable() const;
    bool write(uint32_t reg, uint64_t value, int32_t cpu = -1, uint64_t mask = MsrItem::kNoMask, bool verbose = true);
    bool write(Callback &&callback);
    MsrItem read(uint32_t reg, int32_t cpu = -1, bool verbose = true) const;

private:
    bool rdmsr(uint32_t reg, int32_t cpu, uint64_t &value) const;
    bool wrmsr(uint32_t reg, uint64_t value, int32_t cpu);

    MsrPrivate *d_ptr = nullptr;
};


} /* namespace xmrig */


#endif /* XMRIG_MSR_H */
```