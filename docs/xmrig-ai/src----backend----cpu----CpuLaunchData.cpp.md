# `xmrig\src\backend\cpu\CpuLaunchData.cpp`

```cpp
/*
 * 以下是XMRig程序的版权声明
 * 该程序是自由软件，您可以根据GNU通用公共许可证的条款重新分发或修改它
 * 该程序是希望能够有用，但没有任何保证；甚至没有暗示的保证
 * 您应该已经收到了GNU通用公共许可证的副本。如果没有，请访问http://www.gnu.org/licenses/
 */

// 包含所需的头文件
#include "backend/cpu/CpuLaunchData.h"
#include "backend/common/Tags.h"
#include "backend/cpu/CpuConfig.h"

// 包含标准库头文件
#include <algorithm>

// 构造函数，初始化CPU启动数据
xmrig::CpuLaunchData::CpuLaunchData(const Miner *miner, const Algorithm &algorithm, const CpuConfig &config, const CpuThread &thread, size_t threads, const std::vector<int64_t>& affinities) :
    // 初始化算法
    algorithm(algorithm),
    // 初始化汇编
    assembly(config.assembly()),
    // 初始化是否使用大页
    hugePages(config.isHugePages()),
    // 初始化是否使用硬件AES
    hwAES(config.isHwAES()),
    // 初始化是否使用yield
    yield(config.isYield()),
    // 初始化优先级
    priority(config.priority()),
    // 初始化线程亲和性
    affinity(thread.affinity()),
    // 初始化矿工
    miner(miner),
    // 初始化线程数
    threads(threads),
    # 使用algorithm.maxIntensity()和thread.intensity()中的较大值作为强度值，并将其限制在algorithm.minIntensity()和uint32_t类型的最大值之间
    intensity(std::max<uint32_t>(std::min<uint32_t>(thread.intensity(), algorithm.maxIntensity()), algorithm.minIntensity())),
    # 将affinities赋值给当前对象的affinities属性
    affinities(affinities)
{
}
# 空的代码块

bool xmrig::CpuLaunchData::isEqual(const CpuLaunchData &other) const
{
    # 检查当前对象的算法的 L3 缓存大小是否等于另一个对象的算法的 L3 缓存大小，并且其他属性是否相等
    return (algorithm.l3()      == other.algorithm.l3()
            && assembly         == other.assembly
            && hugePages        == other.hugePages
            && hwAES            == other.hwAES
            && intensity        == other.intensity
            && priority         == other.priority
            && affinity         == other.affinity
            );
}

xmrig::CnHash::AlgoVariant xmrig::CpuLaunchData::av() const
{
    # 如果强度小于等于2，则返回算法变体，否则根据是否支持硬件 AES 返回不同的值
    if (intensity <= 2) {
        return static_cast<CnHash::AlgoVariant>(!hwAES ? (intensity + 2) : intensity);
    }
    return static_cast<CnHash::AlgoVariant>(!hwAES ? (intensity + 5) : (intensity + 2));
}

const char *xmrig::CpuLaunchData::tag()
{
    # 返回 CPU 标签
    return cpu_tag();
}
```