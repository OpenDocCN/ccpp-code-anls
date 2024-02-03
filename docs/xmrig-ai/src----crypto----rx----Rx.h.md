# `xmrig\src\crypto\rx\Rx.h`

```cpp
/* XMRig
 * 版权所有（C）2018-2019 tevador     <tevador@gmail.com>
 * 版权所有（C）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（C）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用的前提下分发的，
 * 但没有任何保证；甚至没有适销性或特定用途适用性的暗示保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本。
 * 如果没有，请参阅<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_RX_H
#define XMRIG_RX_H

#include <cstdint>
#include <utility>
#include <vector>

#include "crypto/common/HugePagesInfo.h"

namespace xmrig
{
// Algorithm, CpuConfig, CpuThread, IRxListener, Job, RxConfig, RxDataset 类的前置声明
class Algorithm;
class CpuConfig;
class CpuThread;
class IRxListener;
class Job;
class RxConfig;
class RxDataset;

// Rx 类的声明
class Rx
{
public:
    // 返回巨大页面信息
    static HugePagesInfo hugePages();
    // 返回数据集指针
    static RxDataset *dataset(const Job &job, uint32_t nodeId);
    // 销毁数据集
    static void destroy();
    // 初始化
    static void init(IRxListener *listener);
    // 使用给定的种子、配置和CPU信息进行初始化
    template<typename T> static bool init(const T &seed, const RxConfig &config, const CpuConfig &cpu);
    // 检查是否准备就绪
    template<typename T> static bool isReady(const T &seed);

#   ifdef XMRIG_FEATURE_MSR
    // 检查是否支持MSR
    static bool isMSR();
#   else
    // 如果不支持MSR，则返回false
    static constexpr bool isMSR()   { return false; }
#   endif
};

} /* namespace xmrig */

#endif /* XMRIG_RX_H */
```