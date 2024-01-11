# `xmrig\src\backend\opencl\OclConfig_gen.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有适销性或特定用途的暗示保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_OCLCONFIG_GEN_H
#define XMRIG_OCLCONFIG_GEN_H


#include "backend/common/Threads.h"
#include "backend/opencl/OclThreads.h"


#include <algorithm>


namespace xmrig {


static inline size_t generate(const char *key, Threads<OclThreads> &threads, const Algorithm &algorithm, const std::vector<OclDevice> &devices)
{
    if (threads.isExist(algorithm) || threads.has(key)) {
        return 0;
    }

    return threads.move(key, OclThreads(devices, algorithm));
}


template<Algorithm::Family FAMILY>
static inline size_t generate(Threads<OclThreads> &, const std::vector<OclDevice> &) { return 0; }


template<>
size_t inline generate<Algorithm::CN>(Threads<OclThreads> &threads, const std::vector<OclDevice> &devices)
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
// 为指定的 CN_LITE 算法生成线程，并分配设备
size_t inline generate<Algorithm::CN_LITE>(Threads<OclThreads> &threads, const std::vector<OclDevice> &devices)
{
    // 调用 generate 函数生成 CN_LITE 算法的线程，并获取生成的线程数量
    size_t count = generate(Algorithm::kCN_LITE, threads, Algorithm::CN_LITE_1, devices);

    // 如果不存在 CN_LITE_0 算法线程，则禁用该线程并增加 count
    if (!threads.isExist(Algorithm::CN_LITE_0)) {
        threads.disable(Algorithm::CN_LITE_0);
        ++count;
    }

    // 返回生成的线程数量
    return count;
}
#endif


#ifdef XMRIG_ALGO_CN_HEAVY
// 为指定的 CN_HEAVY 算法生成线程，并分配设备
template<>
size_t inline generate<Algorithm::CN_HEAVY>(Threads<OclThreads> &threads, const std::vector<OclDevice> &devices)
{
    // 调用 generate 函数生成 CN_HEAVY 算法的线程，并返回生成的线程数量
    return generate(Algorithm::kCN_HEAVY, threads, Algorithm::CN_HEAVY_0, devices);
}
#endif


#ifdef XMRIG_ALGO_CN_PICO
// 为指定的 CN_PICO 算法生成线程，并分配设备
template<>
size_t inline generate<Algorithm::CN_PICO>(Threads<OclThreads> &threads, const std::vector<OclDevice> &devices)
{
    // 调用 generate 函数生成 CN_PICO 算法的线程，并返回生成的线程数量
    return generate(Algorithm::kCN_PICO, threads, Algorithm::CN_PICO_0, devices);
}
#endif


#ifdef XMRIG_ALGO_CN_FEMTO
// 为指定的 CN_FEMTO 算法生成线程，并分配设备
template<>
size_t inline generate<Algorithm::CN_FEMTO>(Threads<OclThreads>& threads, const std::vector<OclDevice>& devices)
{
    // 调用 generate 函数生成 CN_FEMTO 算法的线程，并返回生成的线程数量
    return generate(Algorithm::kCN_UPX2, threads, Algorithm::CN_UPX2, devices);
}
#endif


#ifdef XMRIG_ALGO_RANDOMX
// 为指定的 RANDOM_X 算法生成线程，并分配设备
template<>
size_t inline generate<Algorithm::RANDOM_X>(Threads<OclThreads> &threads, const std::vector<OclDevice> &devices)
{
    // 初始化 count 为 0
    size_t count = 0;

    // 为 RX_0 算法生成线程
    auto rx  = OclThreads(devices, Algorithm::RX_0);
    // 为 RX_WOW 算法生成线程
    auto wow = OclThreads(devices, Algorithm::RX_WOW);
    // 为 RX_ARQ 算法生成线程
    auto arq = OclThreads(devices, Algorithm::RX_ARQ);

    // 如果不存在 RX_WOW 算法线程，并且 wow 线程不等于 rx 线程，则将 wow 线程移动到 RX_WOW 算法，并增加 count
    if (!threads.isExist(Algorithm::RX_WOW) && wow != rx) {
        count += threads.move(Algorithm::kRX_WOW, std::move(wow));
    }

    // 如果不存在 RX_ARQ 算法线程，并且 arq 线程不等于 rx 线程，则将 arq 线程移动到 RX_ARQ 算法，并增加 count
    if (!threads.isExist(Algorithm::RX_ARQ) && arq != rx) {
        count += threads.move(Algorithm::kRX_ARQ, std::move(arq));
    }

    // 将 rx 线程移动到 RX 算法，并增加 count
    count += threads.move(Algorithm::kRX, std::move(rx));

    // 返回生成的线程数量
    return count;
}
#endif


#ifdef XMRIG_ALGO_KAWPOW
// 为指定的 KAWPOW 算法生成线程，并分配设备
template<>
size_t inline generate<Algorithm::KAWPOW>(Threads<OclThreads>& threads, const std::vector<OclDevice>& devices)
{
    # 调用 generate 函数，传入参数 Algorithm::kKAWPOW, threads, Algorithm::KAWPOW_RVN, devices，并返回结果
    return generate(Algorithm::kKAWPOW, threads, Algorithm::KAWPOW_RVN, devices);
// 结束条件判断，检查是否定义了宏
}
#endif

// 定义一个内联函数，用于筛选设备
static inline std::vector<OclDevice> filterDevices(const std::vector<OclDevice> &devices, const std::vector<uint32_t> &hints)
{
    // 创建一个空的 OclDevice 向量，预留空间以容纳最小数量的设备
    std::vector<OclDevice> out;
    out.reserve(std::min(devices.size(), hints.size()));

    // 遍历设备向量
    for (const auto &device  : devices) {
        // 在提示向量中查找设备的索引
        auto it = std::find(hints.begin(), hints.end(), device.index());
        // 如果找到了设备的索引
        if (it != hints.end()) {
            // 将设备添加到输出向量中
            out.emplace_back(device);
        }
    }

    // 返回筛选后的设备向量
    return out;
}

// 结束命名空间声明
} /* namespace xmrig */

// 结束条件判断，检查是否定义了宏
#endif /* XMRIG_OCLCONFIG_GEN_H */
```