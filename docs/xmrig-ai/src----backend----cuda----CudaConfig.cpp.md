# `xmrig\src\backend\cuda\CudaConfig.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/cuda/CudaConfig.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/common/Tags.h"
#include "backend/cuda/CudaConfig_gen.h"
#include "backend/cuda/wrappers/CudaLib.h"
#include "base/io/json/Json.h"
#include "base/io/log/Log.h"


namespace xmrig {


static bool generated           = false;  // 静态布尔变量，标识是否已生成
static const char *kBfactorHint = "bfactor-hint";  // 静态常量字符指针，表示bfactor提示
static const char *kBsleepHint  = "bsleep-hint";  // 静态常量字符指针，表示bsleep提示
static const char *kDevicesHint = "devices-hint";  // 静态常量字符指针，表示设备提示
static const char *kEnabled     = "enabled";  // 静态常量字符指针，表示启用
static const char *kLoader      = "loader";  // 静态常量字符指针，表示加载器

#ifdef XMRIG_FEATURE_NVML
static const char *kNvml        = "nvml";  // 静态常量字符指针，表示nvml
#endif


extern template class Threads<CudaThreads>;


} // namespace xmrig


rapidjson::Value xmrig::CudaConfig::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    Value obj(kObjectType);  // 创建一个对象类型的rapidjson值

    obj.AddMember(StringRef(kEnabled),  m_enabled, allocator);  // 向对象中添加成员，成员名为kEnabled，值为m_enabled
    obj.AddMember(StringRef(kLoader),   m_loader.toJSON(), allocator);  // 向对象中添加成员，成员名为kLoader，值为m_loader.toJSON()

#   ifdef XMRIG_FEATURE_NVML
    if (m_nvmlLoader.isNull()) {
        obj.AddMember(StringRef(kNvml), m_nvml, allocator);  // 如果m_nvmlLoader为空，向对象中添加成员，成员名为kNvml，值为m_nvml
    }
    # 如果条件不满足，则执行以下代码块
    else {
        # 将名为 kNvml 的成员添加到 obj 对象中，值为 m_nvmlLoader.toJSON() 的结果
        obj.AddMember(StringRef(kNvml), m_nvmlLoader.toJSON(), allocator);
    }
# 结束条件判断
    m_threads.toJSON(obj, doc);
    # 返回生成的对象
    return obj;
}

# 获取 CUDA 配置信息
std::vector<xmrig::CudaLaunchData> xmrig::CudaConfig::get(const Miner *miner, const Algorithm &algorithm, const std::vector<CudaDevice> &devices) const
{
    # 定义一个 lambda 函数，用于获取设备索引
    auto deviceIndex = [&devices](uint32_t index) -> int {
        for (uint32_t i = 0; i < devices.size(); ++i) {
            if (devices[i].index() == index) {
                return i;
            }
        }
        return -1;
    };

    # 初始化一个空的向量
    std::vector<CudaLaunchData> out;
    # 获取指定算法的线程配置
    const auto &threads = m_threads.get(algorithm);

    # 如果线程配置为空，则返回空向量
    if (threads.isEmpty()) {
        return out;
    }

    # 预留足够的空间
    out.reserve(threads.count());

    # 遍历线程配置
    for (const auto &thread : threads.data()) {
        # 获取设备索引
        const int index = deviceIndex(thread.index());
        # 如果设备索引为-1，则跳过当前线程配置
        if (index == -1) {
            LOG_INFO("%s" YELLOW(" skip non-existing device with index ") YELLOW_BOLD("%u"), cuda_tag(), thread.index());
            continue;
        }
        # 将线程配置添加到向量中
        out.emplace_back(miner, algorithm, thread, devices[static_cast<size_t>(index)]);
    }

    # 返回结果向量
    return out;
}

# 读取 CUDA 配置信息
void xmrig::CudaConfig::read(const rapidjson::Value &value)
{
    # 如果值为对象
    if (value.IsObject()) {
        # 从 JSON 中获取配置信息
        m_enabled   = Json::getBool(value, kEnabled, m_enabled);
        m_loader    = Json::getString(value, kLoader);
        m_bfactor   = std::min(Json::getUint(value, kBfactorHint, m_bfactor), 12U);
        m_bsleep    = Json::getUint(value, kBsleepHint, m_bsleep);
        # 设置设备提示信息
        setDevicesHint(Json::getString(value, kDevicesHint));
        # 如果支持 NVML，则读取 NVML 配置信息
#       ifdef XMRIG_FEATURE_NVML
        auto &nvml = Json::getValue(value, kNvml);
        if (nvml.IsString()) {
            m_nvmlLoader = nvml.GetString();
        }
        else if (nvml.IsBool()) {
            m_nvml = nvml.GetBool();
        }
#       endif
        # 读取线程配置信息
        m_threads.read(value);
        # 生成配置信息
        generate();
    }
    # 如果值为布尔类型
    else if (value.IsBool()) {
        # 设置启用状态
        m_enabled = value.GetBool();
        # 生成配置信息
        generate();
    }
    # 如果值为其他类型
    else {
        # 标记需要保存配置信息
        m_shouldSave = true;
        # 生成配置信息
        generate();
    }
}

# 生成 CUDA 配置信息
void xmrig::CudaConfig::generate()
    # 如果已经生成过，则直接返回，不再执行后续代码
    if (generated) {
        return;
    }

    # 如果禁用或者线程包含通配符，则直接返回，不再执行后续代码
    if (!isEnabled() || m_threads.has("*")) {
        return;
    }

    # 如果 CUDA 库初始化失败，则直接返回，不再执行后续代码
    if (!CudaLib::init(loader())) {
        return;
    }

    # 如果 CUDA 运行时版本、驱动版本或设备数量获取失败，则直接返回，不再执行后续代码
    if (!CudaLib::runtimeVersion() || !CudaLib::driverVersion() || !CudaLib::deviceCount()) {
        return;
    }

    # 获取可用设备列表
    const auto devices = CudaLib::devices(bfactor(), bsleep(), m_devicesHint);
    # 如果设备列表为空，则直接返回，不再执行后续代码
    if (devices.empty()) {
        return;
    }

    # 初始化计数器
    size_t count = 0;

    # 生成不同算法的任务并计数
    count += xmrig::generate<Algorithm::CN>(m_threads, devices);
    count += xmrig::generate<Algorithm::CN_LITE>(m_threads, devices);
    count += xmrig::generate<Algorithm::CN_HEAVY>(m_threads, devices);
    count += xmrig::generate<Algorithm::CN_PICO>(m_threads, devices);
    count += xmrig::generate<Algorithm::CN_FEMTO>(m_threads, devices);
    count += xmrig::generate<Algorithm::RANDOM_X>(m_threads, devices);
    count += xmrig::generate<Algorithm::KAWPOW>(m_threads, devices);

    # 设置生成标志为 true
    generated    = true;
    # 如果生成的任务数量大于 0，则设置应该保存的标志为 true
    m_shouldSave = count > 0;
// 设置 CUDA 设备的提示信息
void xmrig::CudaConfig::setDevicesHint(const char *devicesHint)
{
    // 如果设备提示信息为空，则直接返回
    if (devicesHint == nullptr) {
        return;
    }

    // 将设备提示信息按逗号分隔，并存储到索引数组中
    const auto indexes = String(devicesHint).split(',');
    // 预留索引数组的空间
    m_devicesHint.reserve(indexes.size());

    // 遍历索引数组，将索引转换为无符号长整型，并存储到设备提示数组中
    for (const auto &index : indexes) {
        m_devicesHint.push_back(strtoul(index, nullptr, 10));
    }
}
```