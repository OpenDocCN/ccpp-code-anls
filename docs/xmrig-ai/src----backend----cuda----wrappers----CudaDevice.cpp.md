# `xmrig\src\backend\cuda\wrappers\CudaDevice.cpp`

```
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本 3 或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。详细信息请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "backend/cuda/wrappers/CudaDevice.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/cuda/CudaThreads.h"
#include "backend/cuda/wrappers/CudaLib.h"
#include "base/crypto/Algorithm.h"
#include "base/io/log/Log.h"


#ifdef XMRIG_FEATURE_NVML
#   include "backend/cuda/wrappers/NvmlLib.h"
#endif

#include <algorithm>


// 初始化 CUDA 设备
xmrig::CudaDevice::CudaDevice(uint32_t index, int32_t bfactor, int32_t bsleep) :
    m_index(index)
{
    // 分配 CUDA 上下文
    auto ctx = CudaLib::alloc(index, bfactor, bsleep);
    // 如果无法获取设备信息，则释放上下文并返回
    if (!CudaLib::deviceInfo(ctx, 0, 0, Algorithm::INVALID)) {
        CudaLib::release(ctx);
        return;
    }

    // 保存上下文
    m_ctx       = ctx;
    // 获取设备名称
    m_name      = CudaLib::deviceName(ctx);
    # 使用CudaLib库中的函数获取当前设备的PCI总线ID和设备ID，并传入PciTopology构造函数中，创建PciTopology对象并赋值给m_topology变量
    m_topology  = PciTopology(CudaLib::deviceUint(ctx, CudaLib::DevicePciBusID), CudaLib::deviceUint(ctx, CudaLib::DevicePciDeviceID), 0);
// 移动构造函数，用于将另一个设备对象的资源移动到当前对象
xmrig::CudaDevice::CudaDevice(CudaDevice &&other) noexcept :
    // 将另一个设备对象的索引移动到当前对象
    m_index(other.m_index),
    // 将另一个设备对象的上下文移动到当前对象
    m_ctx(other.m_ctx),
    // 将另一个设备对象的拓扑结构移动到当前对象
    m_topology(other.m_topology),
    // 将另一个设备对象的名称移动到当前对象
    m_name(std::move(other.m_name))
{
    // 将另一个设备对象的上下文置空
    other.m_ctx = nullptr;
}

// 析构函数，用于释放设备对象的资源
xmrig::CudaDevice::~CudaDevice()
{
    // 释放设备对象的上下文资源
    CudaLib::release(m_ctx);
}

// 返回设备的空闲内存大小
size_t xmrig::CudaDevice::freeMemSize() const
{
    return CudaLib::deviceUlong(m_ctx, CudaLib::DeviceMemoryFree);
}

// 返回设备的全局内存大小
size_t xmrig::CudaDevice::globalMemSize() const
{
    return CudaLib::deviceUlong(m_ctx, CudaLib::DeviceMemoryTotal);
}

// 返回设备的时钟频率
uint32_t xmrig::CudaDevice::clock() const
{
    return CudaLib::deviceUint(m_ctx, CudaLib::DeviceClockRate) / 1000;
}

// 返回设备的计算能力
uint32_t xmrig::CudaDevice::computeCapability(bool major) const
{
    return CudaLib::deviceUint(m_ctx, major ? CudaLib::DeviceArchMajor : CudaLib::DeviceArchMinor);
}

// 返回设备的内存时钟频率
uint32_t xmrig::CudaDevice::memoryClock() const
{
    return CudaLib::deviceUint(m_ctx, CudaLib::DeviceMemoryClockRate) / 1000;
}

// 返回设备的SMX数量
uint32_t xmrig::CudaDevice::smx() const
{
    return CudaLib::deviceUint(m_ctx, CudaLib::DeviceSmx);
}

// 生成设备的算法和线程
void xmrig::CudaDevice::generate(const Algorithm &algorithm, CudaThreads &threads) const
{
    // 如果设备信息不可用，则返回
    if (!CudaLib::deviceInfo(m_ctx, -1, -1, algorithm)) {
        return;
    }
    // 添加设备线程
    threads.add(CudaThread(m_index, m_ctx));
}

// 将设备信息转换为JSON格式
#ifdef XMRIG_FEATURE_API
void xmrig::CudaDevice::toJSON(rapidjson::Value &out, rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 添加设备名称到JSON对象
    out.AddMember("name",           name().toJSON(doc), allocator);
    // 添加设备总线ID到JSON对象
    out.AddMember("bus_id",         topology().toString().toJSON(doc), allocator);
    // 添加设备SMX数量到JSON对象
    out.AddMember("smx",            smx(), allocator);
    // 添加设备架构到JSON对象
    out.AddMember("arch",           arch(), allocator);
    // 添加设备全局内存大小到JSON对象
    out.AddMember("global_mem",     static_cast<uint64_t>(globalMemSize()), allocator);
    // 添加设备时钟频率到JSON对象
    out.AddMember("clock",          clock(), allocator);
    // 添加设备内存时钟频率到JSON对象
    out.AddMember("memory_clock",   memoryClock(), allocator);
}
    # 如果存在 NVML 设备
    if (m_nvmlDevice) {
        # 调用 NvmlLib 库的 health 函数，获取设备健康数据
        auto data = NvmlLib::health(m_nvmlDevice);

        # 创建健康数据的 JSON 对象
        Value health(kObjectType);
        # 添加温度数据到健康数据对象
        health.AddMember("temperature", data.temperature, allocator);
        # 添加功耗数据到健康数据对象
        health.AddMember("power",       data.power, allocator);
        # 添加时钟数据到健康数据对象
        health.AddMember("clock",       data.clock, allocator);
        # 添加内存时钟数据到健康数据对象
        health.AddMember("mem_clock",   data.memClock, allocator);

        # 创建风扇速度数据的 JSON 数组
        Value fanSpeed(kArrayType);
        # 遍历风扇速度数据，添加到风扇速度数据数组中
        for (auto speed : data.fanSpeed) {
            fanSpeed.PushBack(speed, allocator);
        }
        # 添加风扇速度数据数组到健康数据对象
        health.AddMember("fan_speed", fanSpeed, allocator);

        # 添加健康数据对象到输出 JSON 对象中
        out.AddMember("health", health, allocator);
    }
#   endif
# 结束条件编译指令
}
# 结束条件编译指令块
#endif
# 结束条件编译指令
```