# `xmrig\src\backend\cuda\CudaConfig_gen.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   它要么是许可证的第3版，要么是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CUDACONFIG_GEN_H
#define XMRIG_CUDACONFIG_GEN_H


#include "backend/common/Threads.h"
#include "backend/cuda/CudaThreads.h"
#include "backend/cuda/wrappers/CudaDevice.h"


#include <algorithm>


namespace xmrig {


static inline size_t generate(const char *key, Threads<CudaThreads> &threads, const Algorithm &algorithm, const std::vector<CudaDevice> &devices)
{
    if (threads.isExist(algorithm) || threads.has(key)) {
        return 0;
    }

    return threads.move(key, CudaThreads(devices, algorithm));
}


template<Algorithm::Family FAMILY>
static inline size_t generate(Threads<CudaThreads> &, const std::vector<CudaDevice> &) { return 0; }


template<>
size_t inline generate<Algorithm::CN>(Threads<CudaThreads> &threads, const std::vector<CudaDevice> &devices)
{
    size_t count = 0;

    count += generate(Algorithm::kCN, threads, Algorithm::CN_1, devices);
    count += generate(Algorithm::kCN_2, threads, Algorithm::CN_2, devices);

    if (!threads.isExist(Algorithm::CN_0)) {
        threads.disable(Algorithm::CN_0);
        count++;
    }

    return count;
}


#ifdef XMRIG_ALGO_CN_LITE
template<>
#ifdef XMRIG_ALGO_CN_HEAVY
// 为 CN_HEAVY 算法生成线程
template<>
// 重载 generate 函数，返回生成的线程数
size_t inline generate<Algorithm::CN_HEAVY>(Threads<CudaThreads> &threads, const std::vector<CudaDevice> &devices)
{
    // 调用 generate 函数生成 CN_HEAVY 算法的线程
    return generate(Algorithm::kCN_HEAVY, threads, Algorithm::CN_HEAVY_0, devices);
}
#endif


#ifdef XMRIG_ALGO_CN_PICO
// 为 CN_PICO 算法生成线程
template<>
// 重载 generate 函数，返回生成的线程数
size_t inline generate<Algorithm::CN_PICO>(Threads<CudaThreads> &threads, const std::vector<CudaDevice> &devices)
{
    // 调用 generate 函数生成 CN_PICO 算法的线程
    return generate(Algorithm::kCN_PICO, threads, Algorithm::CN_PICO_0, devices);
}
#endif


#ifdef XMRIG_ALGO_CN_FEMTO
// 为 CN_FEMTO 算法生成线程
template<>
// 重载 generate 函数，返回生成的线程数
size_t inline generate<Algorithm::CN_FEMTO>(Threads<CudaThreads>& threads, const std::vector<CudaDevice>& devices)
{
    // 调用 generate 函数生成 CN_UPX2 算法的线程
    return generate(Algorithm::kCN_UPX2, threads, Algorithm::CN_UPX2, devices);
}
#endif


#ifdef XMRIG_ALGO_RANDOMX
// 为 RANDOMX 算法生成线程
template<>
// 重载 generate 函数，返回生成的线程数
size_t inline generate<Algorithm::RANDOM_X>(Threads<CudaThreads> &threads, const std::vector<CudaDevice> &devices)
{
    // 初始化生成的线程数为 0
    size_t count = 0;

    // 为 RX_0 算法创建线程
    auto rx  = CudaThreads(devices, Algorithm::RX_0);
    // 为 RX_WOW 算法创建线程
    auto wow = CudaThreads(devices, Algorithm::RX_WOW);
    // 为 RX_ARQ 算法创建线程
    auto arq = CudaThreads(devices, Algorithm::RX_ARQ);
    // 为 RX_KEVA 算法创建线程
    auto kva = CudaThreads(devices, Algorithm::RX_KEVA);

    // 如果不存在 RX_WOW 算法线程，并且 wow 线程不等于 rx 线程
    if (!threads.isExist(Algorithm::RX_WOW) && wow != rx) {
        // 将 wow 线程移动到 RX_WOW 算法，并增加生成的线程数
        count += threads.move(Algorithm::kRX_WOW, std::move(wow));
    }

    // 如果不存在 RX_ARQ 算法线程，并且 arq 线程不等于 rx 线程
    if (!threads.isExist(Algorithm::RX_ARQ) && arq != rx) {
        // 将 arq 线程移动到 RX_ARQ 算法，并增加生成的线程数
        count += threads.move(Algorithm::kRX_ARQ, std::move(arq));
    }

    // 如果不存在 RX_KEVA 算法线程，并且 kva 线程不等于 rx 线程
    if (!threads.isExist(Algorithm::RX_KEVA) && kva != rx) {
        // 将 kva 线程移动到 RX_KEVA 算法，并增加生成的线程数
        count += threads.move(Algorithm::kRX_KEVA, std::move(kva));
    }

    // 将 rx 线程移动到 RX 算法，并增加生成的线程数
    count += threads.move(Algorithm::kRX, std::move(rx));

    // 返回生成的线程数
    return count;
}
#endif
#ifdef XMRIG_ALGO_KAWPOW
// 如果定义了 XMRIG_ALGO_KAWPOW，则进行以下操作
template<>
// 模板特化，指定生成函数为 Algorithm::KAWPOW 类型
size_t inline generate<Algorithm::KAWPOW>(Threads<CudaThreads> &threads, const std::vector<CudaDevice> &devices)
{
    // 调用 generate 函数，传入 Algorithm::kKAWPOW, threads, Algorithm::KAWPOW_RVN, devices 参数
    return generate(Algorithm::kKAWPOW, threads, Algorithm::KAWPOW_RVN, devices);
}
#endif
// 结束条件编译指令

} /* namespace xmrig */
// 结束 xmrig 命名空间

#endif /* XMRIG_CUDACONFIG_GEN_H */
// 结束条件编译指令
```