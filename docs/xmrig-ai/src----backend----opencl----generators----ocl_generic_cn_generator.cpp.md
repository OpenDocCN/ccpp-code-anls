# `xmrig\src\backend\opencl\generators\ocl_generic_cn_generator.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2017-2018 XMR-Stak <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有（c）2018 Lee Clagett <https://github.com/vtnerd>
 * 版权所有（c）2018-2021 SChernykh <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新分发或修改它
 * 根据GNU通用公共许可证的条款发布
 * 由自由软件基金会发布，无论是许可证的第3版
 * （在您的选择）任何后续版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/OclThreads.h"
#include "backend/opencl/wrappers/OclDevice.h"
#include "base/crypto/Algorithm.h"
#include "crypto/cn/CnAlgo.h"


#include <algorithm>


namespace xmrig {


// 定义常量oneMiB，表示1MB的大小
constexpr const size_t oneMiB = 1024U * 1024U;


// 定义静态内联函数，用于获取最大线程数
static inline uint32_t getMaxThreads(const OclDevice &device, const Algorithm &algorithm)
{
    // 如果设备的供应商ID是NVIDIA，并且设备名称包含"P100"或"V100"，则返回40000
    if (device.vendorId() == OCL_VENDOR_NVIDIA && (device.name().contains("P100") || device.name().contains("V100"))) {
        return 40000U;
    }
    // 如果设备的供应商ID是NVIDIA，则返回4096
    if (device.vendorId() == OCL_VENDOR_NVIDIA) {
        return 4096U;
    }
    // 计算比率，如果算法的L3缓存小于等于1MB，则比率为2，否则为1
    const uint32_t ratio = (algorithm.l3() <= oneMiB) ? 2U : 1U;
    // 如果设备的供应商ID是INTEL，则返回计算得到的线程数
    if (device.vendorId() == OCL_VENDOR_INTEL) {
        return ratio * device.computeUnits() * 8;
    }
    // 其他情况返回固定值1000
    return ratio * 1000U;
}


// 定义静态内联函数，用于获取可能的强度
static inline uint32_t getPossibleIntensity(const OclDevice &device, const Algorithm &algorithm)
{
    // 获取最大线程数
    const uint32_t maxThreads   = getMaxThreads(device, algorithm);
    // 计算最小空闲内存，如果最大线程数为40000，则为512MB，否则为128MB
    const size_t minFreeMem     = (maxThreads == 40000U ? 512U : 128U) * oneMiB;
    // 计算可用内存，为设备可用内存减去最小空闲内存
    const size_t availableMem   = device.freeMemSize() - minFreeMem;
    // 计算每个线程需要的内存，为算法L3缓存大小加上224
    const size_t perThread      = algorithm.l3() + 224U;
    // 计算最大强度，为可用内存除以每个线程需要的内存
    const auto maxIntensity     = static_cast<uint32_t>(availableMem / perThread);

    // 返回最大线程数和最大强度中的较小值
    return std::min<uint32_t>(maxThreads, maxIntensity);
// 获取设备强度的函数
static uint32_t getIntensity(const OclDevice &device, const Algorithm &algorithm)
{
    // 如果设备类型为 Raven，则返回强度为 0
    if (device.type() == OclDevice::Raven) {
        return 0;
    }

    // 获取可能的强度上限
    const uint32_t maxIntensity = getPossibleIntensity(device, algorithm);

    // 根据设备和算法计算强度
    uint32_t intensity = (maxIntensity / (8 * device.computeUnits())) * device.computeUnits() * 8;
    // 如果强度为 0，则返回 0
    if (intensity == 0) {
        return 0;
    }

    // 如果设备为 AMD 品牌且类型为 Lexa、Baffin 或计算单元数小于等于 16，则调整强度
    if (device.vendorId() == OCL_VENDOR_AMD && (device.type() == OclDevice::Lexa || device.type() == OclDevice::Baffin || device.computeUnits() <= 16)) {
        intensity /= 2;

        // 如果算法族为 CN_HEAVY，则再次调整强度
        if (algorithm.family() == Algorithm::CN_HEAVY) {
            intensity /= 2;
        }
    }

    // 返回计算后的强度
    return intensity;
}

// 获取设备的步进索引
static uint32_t getStridedIndex(const OclDevice &device, const Algorithm &algorithm)
{
    // 如果设备不是 AMD 品牌，则返回 0
    if (device.vendorId() != OCL_VENDOR_AMD) {
        return 0;
    }

    // 如果算法基于 CN_2，则返回 2，否则返回 1
    return algorithm.base() == Algorithm::CN_2 ? 2 : 1;
}

// OpenCL 通用 CN 生成器函数
bool ocl_generic_cn_generator(const OclDevice &device, const Algorithm &algorithm, OclThreads &threads)
{
    // 如果算法不是 CN 类型，则返回 false
    if (!algorithm.isCN()) {
        return false;
    }

    // 获取强度
    const uint32_t intensity = getIntensity(device, algorithm);
    // 如果强度为 0，则返回 false
    if (intensity == 0) {
        return false;
    }

    // 根据设备的全局内存大小和强度计算线程数量
    const uint32_t threadCount = (device.vendorId() == OCL_VENDOR_AMD && (device.globalMemSize() - intensity * 2 * algorithm.l3()) > 128 * oneMiB) ? 2 : 1;

    // 添加线程到线程组
    threads.add(OclThread(device.index(), intensity, 8, getStridedIndex(device, algorithm), 2, threadCount, 8));

    // 返回 true
    return true;
}

// 命名空间结束
} // namespace xmrig
```