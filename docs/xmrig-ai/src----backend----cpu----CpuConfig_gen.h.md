# `xmrig\src\backend\cpu\CpuConfig_gen.h`

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
 *   但没有任何保证；甚至没有暗示的保证适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CPUCONFIG_GEN_H
#define XMRIG_CPUCONFIG_GEN_H


#include "backend/common/Threads.h"
#include "backend/cpu/Cpu.h"
#include "backend/cpu/CpuThreads.h"


namespace xmrig {


static inline size_t generate(const char *key, Threads<CpuThreads> &threads, const Algorithm &algorithm, uint32_t limit)
{
    if (threads.isExist(algorithm) || threads.has(key)) {
        return 0;
    }

    return threads.move(key, Cpu::info()->threads(algorithm, limit));
}


template<Algorithm::Family FAMILY>
static inline size_t generate(Threads<CpuThreads> &, uint32_t) { return 0; }


template<>
size_t inline generate<Algorithm::CN>(Threads<CpuThreads> &threads, uint32_t limit)
{
    size_t count = 0;

    count += generate(Algorithm::kCN, threads, Algorithm::CN_1, limit);

    if (!threads.isExist(Algorithm::CN_0)) {
        threads.disable(Algorithm::CN_0);
        ++count;
    }

    return count;
}


#ifdef XMRIG_ALGO_CN_LITE
template<>
size_t inline generate<Algorithm::CN_LITE>(Threads<CpuThreads> &threads, uint32_t limit)
{
    size_t count = 0;

    count += generate(Algorithm::kCN_LITE, threads, Algorithm::CN_LITE_1, limit);
    # 如果算法CN_LITE_0不存在于线程中
    if (!threads.isExist(Algorithm::CN_LITE_0)) {
        # 禁用算法CN_LITE_0
        threads.disable(Algorithm::CN_LITE_0);
        # 计数加一
        ++count;
    }
    # 返回计数值
    return count;
#ifdef XMRIG_ALGO_CN_HEAVY
template<>
size_t inline generate<Algorithm::CN_HEAVY>(Threads<CpuThreads> &threads, uint32_t limit)
{
    return generate(Algorithm::kCN_HEAVY, threads, Algorithm::CN_HEAVY_0, limit);
}
#endif


#ifdef XMRIG_ALGO_CN_HEAVY
// 如果定义了 XMRIG_ALGO_CN_HEAVY，则实例化生成函数，使用 CN_HEAVY 算法
template<>
size_t inline generate<Algorithm::CN_HEAVY>(Threads<CpuThreads> &threads, uint32_t limit)
{
    return generate(Algorithm::kCN_HEAVY, threads, Algorithm::CN_HEAVY_0, limit);
}
#endif



#ifdef XMRIG_ALGO_CN_PICO
template<>
size_t inline generate<Algorithm::CN_PICO>(Threads<CpuThreads> &threads, uint32_t limit)
{
    return generate(Algorithm::kCN_PICO, threads, Algorithm::CN_PICO_0, limit);
}
#endif


#ifdef XMRIG_ALGO_CN_PICO
// 如果定义了 XMRIG_ALGO_CN_PICO，则实例化生成函数，使用 CN_PICO 算法
template<>
size_t inline generate<Algorithm::CN_PICO>(Threads<CpuThreads> &threads, uint32_t limit)
{
    return generate(Algorithm::kCN_PICO, threads, Algorithm::CN_PICO_0, limit);
}
#endif



#ifdef XMRIG_ALGO_CN_FEMTO
template<>
size_t inline generate<Algorithm::CN_FEMTO>(Threads<CpuThreads>& threads, uint32_t limit)
{
    return generate(Algorithm::kCN_UPX2, threads, Algorithm::CN_UPX2, limit);
}
#endif


#ifdef XMRIG_ALGO_CN_FEMTO
// 如果定义了 XMRIG_ALGO_CN_FEMTO，则实例化生成函数，使用 CN_FEMTO 算法
template<>
size_t inline generate<Algorithm::CN_FEMTO>(Threads<CpuThreads>& threads, uint32_t limit)
{
    return generate(Algorithm::kCN_UPX2, threads, Algorithm::CN_UPX2, limit);
}
#endif



#ifdef XMRIG_ALGO_RANDOMX
template<>
size_t inline generate<Algorithm::RANDOM_X>(Threads<CpuThreads> &threads, uint32_t limit)
{
    size_t count = 0;
    auto cpuInfo = Cpu::info();
    auto wow     = cpuInfo->threads(Algorithm::RX_WOW, limit);

    if (!threads.isExist(Algorithm::RX_ARQ)) {
        auto arq = cpuInfo->threads(Algorithm::RX_ARQ, limit);
        if (arq == wow) {
            threads.setAlias(Algorithm::RX_ARQ, Algorithm::kRX_WOW);
            ++count;
        }
        else {
            count += threads.move(Algorithm::kRX_ARQ, std::move(arq));
        }
    }

    if (!threads.isExist(Algorithm::RX_KEVA)) {
        auto keva = cpuInfo->threads(Algorithm::RX_KEVA, limit);
        if (keva == wow) {
            threads.setAlias(Algorithm::RX_KEVA, Algorithm::kRX_WOW);
            ++count;
        }
        else {
            count += threads.move(Algorithm::kRX_KEVA, std::move(keva));
        }
    }

    if (!threads.isExist(Algorithm::RX_WOW)) {
        count += threads.move(Algorithm::kRX_WOW, std::move(wow));
    }

    count += generate(Algorithm::kRX, threads, Algorithm::RX_0, limit);

    return count;
}
#endif


#ifdef XMRIG_ALGO_RANDOMX
// 如果定义了 XMRIG_ALGO_RANDOMX，则实例化生成函数，使用 RANDOM_X 算法
template<>
size_t inline generate<Algorithm::RANDOM_X>(Threads<CpuThreads> &threads, uint32_t limit)
{
    size_t count = 0;
    auto cpuInfo = Cpu::info();
    auto wow     = cpuInfo->threads(Algorithm::RX_WOW, limit);

    // 检查是否存在 RX_ARQ 线程，如果不存在则进行处理
    if (!threads.isExist(Algorithm::RX_ARQ)) {
        auto arq = cpuInfo->threads(Algorithm::RX_ARQ, limit);
        // 如果 ARQ 线程与 WOW 线程相同，则设置别名并增加计数
        if (arq == wow) {
            threads.setAlias(Algorithm::RX_ARQ, Algorithm::kRX_WOW);
            ++count;
        }
        else {
            count += threads.move(Algorithm::kRX_ARQ, std::move(arq));
        }
    }

    // 检查是否存在 RX_KEVA 线程，如果不存在则进行处理
    if (!threads.isExist(Algorithm::RX_KEVA)) {
        auto keva = cpuInfo->threads(Algorithm::RX_KEVA, limit);
        // 如果 KEVA 线程与 WOW 线程相同，则设置别名并增加计数
        if (keva == wow) {
            threads.setAlias(Algorithm::RX_KEVA, Algorithm::kRX_WOW);
            ++count;
        }
        else {
            count += threads.move(Algorithm::kRX_KEVA, std::move(keva));
        }
    }

    // 检查是否存在 RX_WOW 线程，如果不存在则进行处理
    if (!threads.isExist(Algorithm::RX_WOW)) {
        count += threads.move(Algorithm::kRX_WOW, std::move(wow));
    }

    // 调用生成函数，使用 RANDOM_X 算法，增加计数
    count += generate(Algorithm::kRX, threads, Algorithm::RX_0, limit);

    return count;
}
#endif



#ifdef XMRIG_ALGO_ARGON2
template<>
size_t inline generate<Algorithm::ARGON2>(Threads<CpuThreads> &threads, uint32_t limit)
{


#ifdef XMRIG_ALGO_ARGON2
// 如果定义了 XMRIG_ALGO_ARGON2，则实例化生成函数，使用 ARGON2 算法
template<>
size_t inline generate<Algorithm::ARGON2>(Threads<CpuThreads> &threads, uint32_t limit)
{
    # 调用 generate 函数，传入 Algorithm::kAR2, threads, Algorithm::AR2_CHUKWA_V2, limit 参数，并返回结果
    return generate(Algorithm::kAR2, threads, Algorithm::AR2_CHUKWA_V2, limit);
} // 结束 #endif 块

#ifdef XMRIG_ALGO_GHOSTRIDER
// 为 GHOSTRIDER 算法生成哈希
template<>
size_t inline generate<Algorithm::GHOSTRIDER>(Threads<CpuThreads>& threads, uint32_t limit)
{
    return generate(Algorithm::kGHOSTRIDER, threads, Algorithm::GHOSTRIDER_RTM, limit);
}
#endif

} /* namespace xmrig */ // 结束 xmrig 命名空间

#endif /* XMRIG_CPUCONFIG_GEN_H */ // 结束 #endif 块
```