# `xmrig\src\backend\opencl\OclConfig.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本3或
 *   （根据您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/OclConfig.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/common/Tags.h"
#include "backend/opencl/OclConfig_gen.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/io/json/Json.h"
#include "base/io/log/Log.h"


namespace xmrig {


static const char *kCache       = "cache";  // 定义字符串常量"kCache"
static const char *kDevicesHint = "devices-hint";  // 定义字符串常量"kDevicesHint"
static const char *kEnabled     = "enabled";  // 定义字符串常量"kEnabled"
static const char *kLoader      = "loader";  // 定义字符串常量"kLoader"

#ifndef XMRIG_OS_APPLE
static const char *kAMD         = "AMD";  // 定义字符串常量"kAMD"
static const char *kINTEL       = "INTEL";  // 定义字符串常量"kINTEL"
static const char *kNVIDIA      = "NVIDIA";  // 定义字符串常量"kNVIDIA"
static const char *kPlatform    = "platform";  // 定义字符串常量"kPlatform"
#endif

#ifdef XMRIG_FEATURE_ADL
static const char *kAdl         = "adl";  // 定义字符串常量"kAdl"
#endif


extern template class Threads<OclThreads>;


} // namespace xmrig


#ifndef XMRIG_OS_APPLE
xmrig::OclConfig::OclConfig() : m_platformVendor(kAMD) {}  // 初始化OclConfig类的构造函数，设置m_platformVendor为"kAMD"
#else
xmrig::OclConfig::OclConfig() = default;  // 如果是苹果系统，则使用默认构造函数
#endif


xmrig::OclPlatform xmrig::OclConfig::platform() const  // 获取OclConfig类的platform()方法
{
    const auto platforms = OclPlatform::get();  // 获取OclPlatform的实例
    if (platforms.empty()) {  // 如果platforms为空
        return {};  // 返回空
    }

#   ifndef XMRIG_OS_APPLE
    # 如果平台供应商不为空
    if (!m_platformVendor.isEmpty()) {
        # 定义搜索字符串
        String search;
        # 将平台供应商转换为大写
        String vendor = m_platformVendor;
        vendor.toUpper();

        # 根据供应商名称匹配对应的搜索字符串
        if (vendor == kAMD) {
            search = "Advanced Micro Devices";
        }
        else if (vendor == kNVIDIA) {
            search = kNVIDIA;
        }
        else if (vendor == kINTEL) {
            search = "Intel";
        }
        else {
            search = m_platformVendor;
        }

        # 遍历平台列表，查找包含搜索字符串的平台
        for (const auto &platform : platforms) {
            if (platform.vendor().contains(search)) {
                return platform;
            }
        }
    }
    # 如果平台供应商为空，且平台索引小于平台列表大小
    else if (m_platformIndex < platforms.size()) {
        # 返回对应索引的平台
        return platforms[m_platformIndex];
    }

    # 返回空对象
    return {};
// 返回第一个平台，如果没有其他平台
    return platforms[0];
// 将 OclConfig 对象转换为 JSON 格式
rapidjson::Value xmrig::OclConfig::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 创建一个空的 JSON 对象
    Value obj(kObjectType);

    // 向 JSON 对象中添加成员
    obj.AddMember(StringRef(kEnabled),  m_enabled, allocator);
    obj.AddMember(StringRef(kCache),    m_cache, allocator);
    obj.AddMember(StringRef(kLoader),   m_loader.toJSON(), allocator);

// 如果不是苹果系统，向 JSON 对象中添加平台信息
#   ifndef XMRIG_OS_APPLE
    obj.AddMember(StringRef(kPlatform), m_platformVendor.isEmpty() ? Value(m_platformIndex) : m_platformVendor.toJSON(), allocator);
#   endif

// 如果支持 ADL 特性，向 JSON 对象中添加 ADL 信息
#   ifdef XMRIG_FEATURE_ADL
    obj.AddMember(StringRef(kAdl),      m_adl, allocator);
#   endif

    // 将线程信息转换为 JSON 格式并添加到 JSON 对象中
    m_threads.toJSON(obj, doc);

    // 返回 JSON 对象
    return obj;
}

// 获取 OpenCL 配置信息
std::vector<xmrig::OclLaunchData> xmrig::OclConfig::get(const Miner *miner, const Algorithm &algorithm, const OclPlatform &platform, const std::vector<OclDevice> &devices) const
{
    // 创建一个空的 OclLaunchData 对象数组
    std::vector<OclLaunchData> out;
    const auto &threads = m_threads.get(algorithm);

    // 如果线程信息为空，直接返回空数组
    if (threads.isEmpty()) {
        return out;
    }

    // 预留足够的空间
    out.reserve(threads.count() * 2);

    // 遍历线程信息
    for (const auto &thread : threads.data()) {
        // 如果线程索引超出设备数量，记录日志并继续下一次循环
        if (thread.index() >= devices.size()) {
            LOG_INFO("%s" YELLOW(" skip non-existing device with index ") YELLOW_BOLD("%u"), ocl_tag(), thread.index());
            continue;
        }

        // 如果线程数大于1，为每个线程创建 OclLaunchData 对象并添加到数组中
        if (thread.threads().size() > 1) {
            for (int64_t affinity : thread.threads()) {
                out.emplace_back(miner, algorithm, *this, platform, thread, devices[thread.index()], affinity);
            }
        }
        // 否则，为单个线程创建 OclLaunchData 对象并添加到数组中
        else {
            out.emplace_back(miner, algorithm, *this, platform, thread, devices[thread.index()], thread.threads().front());
        }
    }

    // 返回 OclLaunchData 对象数组
    return out;
}

// 从 JSON 对象中读取 OpenCL 配置信息
void xmrig::OclConfig::read(const rapidjson::Value &value)
    # 如果值是一个对象
    if (value.IsObject()) {
        # 从 JSON 对象中获取布尔类型的值，并赋给 m_enabled，如果获取失败则保持原值
        m_enabled   = Json::getBool(value, kEnabled, m_enabled);
        # 从 JSON 对象中获取布尔类型的值，并赋给 m_cache，如果获取失败则保持原值
        m_cache     = Json::getBool(value, kCache, m_cache);
        # 从 JSON 对象中获取字符串类型的值，并赋给 m_loader
        m_loader    = Json::getString(value, kLoader);
// 如果不是苹果系统，则设置平台信息
#ifndef XMRIG_OS_APPLE
void xmrig::OclConfig::setPlatform(const rapidjson::Value &platform)
{
    // 如果平台是字符串类型，则设置平台供应商
    if (platform.IsString()) {
        m_platformVendor = platform.GetString();
    }
    // 如果平台是无符号整数类型，则设置平台供应商为空，设置平台索引
    else if (platform.IsUint()) {
        m_platformVendor = nullptr;
        m_platformIndex  = platform.GetUint();
    }
}
#endif

// 设置设备提示信息
void xmrig::OclConfig::setDevicesHint(const char *devicesHint)
{
    // 如果设备提示信息为空，则直接返回
    if (devicesHint == nullptr) {
        return;
    }

    // 将设备提示信息按逗号分割，并转换为整数后存入设备提示列表
    const auto indexes = String(devicesHint).split(',');
    m_devicesHint.reserve(indexes.size());

    for (const auto &index : indexes) {
        m_devicesHint.push_back(strtoul(index, nullptr, 10));
    }
}

// 生成 OpenCL 配置
void xmrig::OclConfig::generate()
{
    // 如果未启用或者线程中包含通配符，则直接返回
    if (!isEnabled() || m_threads.has("*")) {
        return;
    }

    // 初始化 OpenCL 库，如果失败则直接返回
    if (!OclLib::init(loader())) {
        return;
    }

    // 获取设备列表，如果为空则直接返回
    const auto devices = m_devicesHint.empty() ? platform().devices() : filterDevices(platform().devices(), m_devicesHint);
    if (devices.empty()) {
        return;
    }

    // 生成各种算法的配置，并统计生成的数量
    size_t count = 0;

    count += xmrig::generate<Algorithm::CN>(m_threads, devices);
    count += xmrig::generate<Algorithm::CN_LITE>(m_threads, devices);
    count += xmrig::generate<Algorithm::CN_HEAVY>(m_threads, devices);
    count += xmrig::generate<Algorithm::CN_PICO>(m_threads, devices);
    count += xmrig::generate<Algorithm::CN_FEMTO>(m_threads, devices);
    count += xmrig::generate<Algorithm::RANDOM_X>(m_threads, devices);
    count += xmrig::generate<Algorithm::KAWPOW>(m_threads, devices);

    // 如果生成数量大于0，则设置应该保存标志为真
    m_shouldSave = count > 0;
}
    # 代码块结束
}
# 结束条件编译指令的代码块
#endif
# 结束条件编译指令
```