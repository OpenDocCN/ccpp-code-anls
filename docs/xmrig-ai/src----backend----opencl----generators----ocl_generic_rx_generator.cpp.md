# `xmrig\src\backend\opencl\generators\ocl_generic_rx_generator.cpp`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 要么是许可证的第3版，要么（在您的选择下）是任何更高版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有暗示的担保。
 * 有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/OclThreads.h"  // 引入 OpenCL 线程管理头文件
#include "backend/opencl/wrappers/OclDevice.h"  // 引入 OpenCL 设备封装头文件
#include "base/crypto/Algorithm.h"  // 引入加密算法头文件
#include "crypto/randomx/randomx.h"  // 引入随机算法头文件
#include "crypto/rx/RxAlgo.h"  // 引入 Rx 算法头文件

namespace xmrig {

// 使用 OpenCL 设备和算法生成随机数
bool ocl_generic_rx_generator(const OclDevice &device, const Algorithm &algorithm, OclThreads &threads) {
    // 如果算法不是 RANDOM_X 类型，则返回 false
    if (algorithm.family() != Algorithm::RANDOM_X) {
        return false;
    }

    // 如果设备类型是 OclDevice::Raven，则添加相应的线程配置并返回 true
    if (device.type() == OclDevice::Raven) {
        threads.add(OclThread(device.index(), (device.computeUnits() > 4) ? 256 : 128, 8, 1, true, true, 6));
        return true;
    }

    // 获取设备的全局内存大小
    const size_t mem = device.globalMemSize();
    // 根据算法获取配置信息
    auto config      = RxAlgo::base(algorithm);
    // 初始化变量
    bool gcnAsm      = false;
    bool isNavi      = false;

    // 根据设备类型进行不同的处理
    switch (device.type()) {
    // 对于以下设备类型，设置 gcnAsm 为 true
    case OclDevice::Baffin:
    case OclDevice::Ellesmere:
    case OclDevice::Polaris:
    case OclDevice::Lexa:
    case OclDevice::Vega_10:
    case OclDevice::Vega_20:
        gcnAsm = true;
        break;
    // 对于以下设备类型，设置 gcnAsm 和 isNavi 为 true
    case OclDevice::Navi_10:
    case OclDevice::Navi_12:
    case OclDevice::Navi_14:
        gcnAsm = true;
        isNavi = true;
        break;
    // 对于 Navi_21 设备类型，设置 isNavi 为 true
    case OclDevice::Navi_21:
        isNavi = true;
        break;
    // 默认情况下不做任何处理
    default:
        break;
    }

    // 计算数据集所需内存大小
    const uint32_t dataset_mem = config->DatasetBaseSize + config->DatasetExtraSize + (128U << 20);

    // 如果内存不足，将数据集放在主机上
    bool datasetHost = (mem < dataset_mem);

    // 每个线程使用一个 scratchpad 和一些小缓冲区在 GPU 上
    const uint32_t per_thread_mem = config->ScratchpadL3_Size + 32768;

    // 计算强度
    uint32_t intensity = static_cast<uint32_t>((mem - (datasetHost ? 0 : dataset_mem)) / per_thread_mem / 2);

    // 如果强度过高，会影响哈希率
    const uint32_t intensityCoeff = isNavi ? 64 : 16;
    if (intensity > device.computeUnits() * intensityCoeff) {
        intensity = device.computeUnits() * intensityCoeff;
    }

    // 将强度调整为 64 的倍数
    intensity -= intensity % 64;

    // 如果线程数过少，将数据集放在主机上以获取更多线程
    if (intensity < device.computeUnits() * 4) {
        datasetHost = true;
        intensity = static_cast<uint32_t>(mem / per_thread_mem / 2);
        intensity -= intensity % 64;
    }

    // 如果强度为 0，则返回 false
    if (!intensity) {
        return false;
    }

    // 添加线程
    threads.add(OclThread(device.index(), intensity, 8, device.vendorId() == OCL_VENDOR_AMD ? 2 : 1, gcnAsm, datasetHost, 6));

    // 返回 true
    return true;
# 结束 xmrig 命名空间
}

# 结束 xmrig 命名空间
} // namespace xmrig
```