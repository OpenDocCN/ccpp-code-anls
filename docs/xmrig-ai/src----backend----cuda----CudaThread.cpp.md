# `xmrig\src\backend\cuda\CudaThread.cpp`

```
/*
 * XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "backend/cuda/CudaThread.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/cuda/wrappers/CudaLib.h"
#include "base/io/json/Json.h"


#include <algorithm>


namespace xmrig {

static const char *kAffinity    = "affinity";
static const char *kBFactor     = "bfactor";
static const char *kBlocks      = "blocks";
static const char *kBSleep      = "bsleep";
static const char *kIndex       = "index";
static const char *kThreads     = "threads";
static const char *kDatasetHost = "dataset_host";

} // namespace xmrig


xmrig::CudaThread::CudaThread(const rapidjson::Value &value)
{
    // 如果传入的值不是对象，则直接返回
    if (!value.IsObject()) {
        return;
    }

    // 从 JSON 对象中获取指定键对应的值，并赋给成员变量
    m_index     = Json::getUint(value, kIndex);
    m_threads   = Json::getInt(value, kThreads);
    m_blocks    = Json::getInt(value, kBlocks);
    // 从 JSON 对象中获取指定键对应的值，并与 m_bfactor 取最小值后赋给 m_bfactor
    m_bfactor   = std::min(Json::getUint(value, kBFactor, m_bfactor), 12U);
    m_bsleep    = Json::getUint(value, kBSleep, m_bsleep);
    m_affinity  = Json::getUint64(value, kAffinity, m_affinity);

    // 如果 JSON 对象中的指定键对应的值是整数，则将其转换为布尔值并赋给 m_datasetHost
    if (Json::getValue(value, kDatasetHost).IsInt()) {
        m_datasetHost = Json::getInt(value, kDatasetHost, m_datasetHost) != 0;
    }
}
    # 如果条件不满足，则将 value 中的 kDatasetHost 转换为布尔值，并赋值给 m_datasetHost
    else {
        m_datasetHost = Json::getBool(value, kDatasetHost);
    }
// 构造函数，初始化 CUDA 线程对象
xmrig::CudaThread::CudaThread(uint32_t index, nvid_ctx *ctx) :
    // 从 CUDA 上下文中获取设备块数，初始化 m_blocks
    m_blocks(CudaLib::deviceInt(ctx, CudaLib::DeviceBlocks)),
    // 从 CUDA 上下文中获取设备数据集主机内存，初始化 m_datasetHost
    m_datasetHost(CudaLib::deviceInt(ctx, CudaLib::DeviceDatasetHost)),
    // 从 CUDA 上下文中获取设备线程数，初始化 m_threads
    m_threads(CudaLib::deviceInt(ctx, CudaLib::DeviceThreads)),
    // 初始化 m_index
    m_index(index),
    // 从 CUDA 上下文中获取设备 B 因子，初始化 m_bfactor
    m_bfactor(CudaLib::deviceUint(ctx, CudaLib::DeviceBFactor)),
    // 从 CUDA 上下文中获取设备休眠时间，初始化 m_bsleep
    m_bsleep(CudaLib::deviceUint(ctx, CudaLib::DeviceBSleep))
{
    // 构造函数结束
}

// 比较函数，判断两个 CUDA 线程对象是否相等
bool xmrig::CudaThread::isEqual(const CudaThread &other) const
{
    // 判断各个成员变量是否相等
    return m_blocks      == other.m_blocks &&
           m_threads     == other.m_threads &&
           m_affinity    == other.m_affinity &&
           m_index       == other.m_index &&
           m_bfactor     == other.m_bfactor &&
           m_bsleep      == other.m_bsleep &&
           m_datasetHost == other.m_datasetHost;
}

// 转换为 JSON 格式的函数
rapidjson::Value xmrig::CudaThread::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 创建一个 JSON 对象
    Value out(kObjectType);

    // 添加成员到 JSON 对象中
    out.AddMember(StringRef(kIndex),        index(), allocator);
    out.AddMember(StringRef(kThreads),      threads(), allocator);
    out.AddMember(StringRef(kBlocks),       blocks(), allocator);
    out.AddMember(StringRef(kBFactor),      bfactor(), allocator);
    out.AddMember(StringRef(kBSleep),       bsleep(), allocator);
    out.AddMember(StringRef(kAffinity),     affinity(), allocator);

    // 如果 m_datasetHost 大于等于 0，则添加 kDatasetHost 成员到 JSON 对象中
    if (m_datasetHost >= 0) {
        out.AddMember(StringRef(kDatasetHost), m_datasetHost > 0, allocator);
    }

    // 返回 JSON 对象
    return out;
}
```