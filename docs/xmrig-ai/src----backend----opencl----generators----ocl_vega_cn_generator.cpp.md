# `xmrig\src\backend\opencl\generators\ocl_vega_cn_generator.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据GNU通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/OclThreads.h"  // 引入OpenCL线程头文件
#include "backend/opencl/wrappers/OclDevice.h"  // 引入OpenCL设备包装头文件
#include "base/crypto/Algorithm.h"  // 引入算法头文件
#include "crypto/cn/CnAlgo.h"  // 引入Cn算法头文件

#include <algorithm>  // 引入算法标准库

namespace xmrig {

constexpr const size_t oneMiB = 1024U * 1024U;  // 定义常量oneMiB为1MB的大小

static inline bool isMatch(const OclDevice &device, const Algorithm &algorithm)  // 定义内联函数isMatch，判断设备和算法是否匹配
{
    return algorithm.isCN() &&  // 判断算法是否为CN算法
           device.vendorId() == OCL_VENDOR_AMD &&  // 判断设备的供应商ID是否为AMD
           (device.type() == OclDevice::Vega_10 || device.type() == OclDevice::Vega_20);  // 判断设备类型是否为Vega_10或Vega_20
}

static inline uint32_t getMaxThreads(const OclDevice &device, const Algorithm &algorithm)  // 定义内联函数getMaxThreads，获取最大线程数
{
    const uint32_t ratio = (algorithm.l3() <= oneMiB) ? 2U : 1U;  // 根据算法的L3缓存大小判断比例

    if (device.type() == OclDevice::Vega_10) {  // 如果设备类型为Vega_10
        if (device.computeUnits() == 56 && algorithm.family() == Algorithm::CN && algorithm.base() == Algorithm::CN_2) {  // 如果设备计算单元为56且算法家族为CN且基础算法为CN_2
            return 1792U;  // 返回1792线程
        }
    }

    return ratio * 2024U;  // 返回根据比例计算的线程数
}

static inline uint32_t getPossibleIntensity(const OclDevice &device, const Algorithm &algorithm)  // 定义内联函数getPossibleIntensity，获取可能的强度
{
    const uint32_t maxThreads   = getMaxThreads(device, algorithm);  // 获取最大线程数
    const size_t availableMem   = device.freeMemSize() - (128U * oneMiB);  // 计算可用内存大小
    # 定义每个线程需要的内存大小，包括算法所需的 L3 缓存大小和额外的 224 个单位
    const size_t perThread      = algorithm.l3() + 224U;
    # 计算可用内存能够支持的最大线程强度，即可用内存除以每个线程需要的内存大小
    const auto maxIntensity     = static_cast<uint32_t>(availableMem / perThread);
    # 返回最大线程数和最大线程强度中的较小值
    return std::min<uint32_t>(maxThreads, maxIntensity);
// 获取算法强度的函数，参数为设备和算法对象
static inline uint32_t getIntensity(const OclDevice &device, const Algorithm &algorithm)
{
    // 获取设备和算法允许的最大强度
    const uint32_t maxIntensity = getPossibleIntensity(device, algorithm);

    // 如果设备类型为 Vega_10
    if (device.type() == OclDevice::Vega_10) {
        // 如果算法家族为 CN_HEAVY 且设备计算单元为 64 且最大强度大于 976
        if (algorithm.family() == Algorithm::CN_HEAVY && device.computeUnits() == 64 && maxIntensity > 976) {
            // 返回 976
            return 976;
        }
    }

    // 返回最大强度除以设备计算单元数再乘以设备计算单元数的结果
    return maxIntensity / device.computeUnits() * device.computeUnits();
}


// 获取工作大小的函数，参数为算法对象
static inline uint32_t getWorksize(const Algorithm &algorithm)
{
    // 获取算法家族
    const auto f = algorithm.family();
    // 如果算法家族为 CN_PICO 或 CN_FEMTO
    if (f == Algorithm::CN_PICO || f == Algorithm::CN_FEMTO) {
        // 返回 64
        return 64;
    }

    // 如果算法基础为 CN_2，返回 16，否则返回 8
    return algorithm.base() == Algorithm::CN_2 ? 16 : 8;
}


// 获取步进索引的函数，参数为算法对象
static uint32_t getStridedIndex(const Algorithm &algorithm)
{
    // 如果算法基础为 CN_2，返回 2，否则返回 1
    return algorithm.base() == Algorithm::CN_2 ? 2 : 1;
}


// 获取内存块大小的函数，参数为算法对象
static inline uint32_t getMemChunk(const Algorithm &algorithm)
{
    // 如果算法基础为 CN_2，返回 1，否则返回 2
    return algorithm.base() == Algorithm::CN_2 ? 1 : 2;
}


// OpenCL Vega 算法生成器函数，参数为设备、算法和线程对象
bool ocl_vega_cn_generator(const OclDevice &device, const Algorithm &algorithm, OclThreads &threads)
{
    // 如果设备和算法不匹配，返回 false
    if (!isMatch(device, algorithm)) {
        return false;
    }

    // 获取强度
    const uint32_t intensity = getIntensity(device, algorithm);
    // 如果强度为 0，返回 false
    if (intensity == 0) {
        return false;
    }

    // 获取工作大小
    const uint32_t worksize = getWorksize(algorithm);
    // 获取内存块大小
    const uint32_t memChunk = getMemChunk(algorithm);

    // 向线程对象添加线程
    threads.add(OclThread(device.index(), intensity, worksize, getStridedIndex(algorithm), memChunk, 2, 8));

    // 返回 true
    return true;
}


} // namespace xmrig
```