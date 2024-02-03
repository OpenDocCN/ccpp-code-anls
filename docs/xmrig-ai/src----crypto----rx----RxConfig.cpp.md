# `xmrig\src\crypto\rx\RxConfig.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据 GNU 通用公共许可证的条款，发布版本 3 或
 *   (根据您的选择) 任何更高版本。
 *
 *   本程序发布的目的是希望它有用，
 *   但没有任何担保；甚至没有适用于特定目的的隐含担保。详见
 *   GNU 通用公共许可证获取更多详细信息。
 *
 *   如果没有收到 GNU 通用公共许可证的副本，
 *   请参阅 <http://www.gnu.org/licenses/>。
 */

#include "crypto/rx/RxConfig.h"
#include "3rdparty/rapidjson/document.h"
#include "backend/cpu/Cpu.h"
#include "base/io/json/Json.h"


#include <array>
#include <algorithm>
#include <cmath>


#ifdef _MSC_VER
#   define strcasecmp  _stricmp
#endif


namespace xmrig {

// 初始化常量
const char *RxConfig::kInit                     = "init";
const char *RxConfig::kInitAVX2                 = "init-avx2";
const char *RxConfig::kField                    = "randomx";
const char *RxConfig::kMode                     = "mode";
const char *RxConfig::kOneGbPages               = "1gb-pages";
const char *RxConfig::kRdmsr                    = "rdmsr";
const char *RxConfig::kWrmsr                    = "wrmsr";
const char *RxConfig::kScratchpadPrefetchMode   = "scratchpad_prefetch_mode";
const char *RxConfig::kCacheQoS                 = "cache_qos";

#ifdef XMRIG_FEATURE_HWLOC
const char *RxConfig::kNUMA                     = "numa";
#endif


// 模式名称数组
static const std::array<const char *, RxConfig::ModeMax> modeNames = { "auto", "fast", "light" };


#ifdef XMRIG_FEATURE_MSR
constexpr size_t kMsrArraySize = 6;

// MSR 预设数组
static const std::array<MsrItems, kMsrArraySize> msrPresets = {


注释：
    # 创建一个 MsrItems 对象
    MsrItems(),
    # 创建一个 MsrItems 对象，并初始化其中的键值对
    MsrItems{{ 0xC0011020, 0ULL }, { 0xC0011021, 0x40ULL, ~0x20ULL }, { 0xC0011022, 0x1510000ULL }, { 0xC001102b, 0x2000cc16ULL }},
    # 创建一个 MsrItems 对象，并初始化其中的键值对
    MsrItems{{ 0xC0011020, 0x0004480000000000ULL }, { 0xC0011021, 0x001c000200000040ULL, ~0x20ULL }, { 0xC0011022, 0xc000000401570000ULL }, { 0xC001102b, 0x2000cc10ULL }},
    # 创建一个 MsrItems 对象，并初始化其中的键值对
    MsrItems{{ 0xC0011020, 0x0004400000000000ULL }, { 0xC0011021, 0x0004000000000040ULL, ~0x20ULL }, { 0xC0011022, 0x8680000401570000ULL }, { 0xC001102b, 0x2040cc10ULL }},
    # 创建一个 MsrItems 对象，并初始化其中的键值对
    MsrItems{{ 0x1a4, 0xf }},
    # 创建一个空的 MsrItems 对象
    MsrItems()
};

// 定义静态常量数组，包含 MSR 名称列表
static const std::array<const char *, kMsrArraySize> modNames = { MSR_NAMES_LIST };

// 断言，确保 kMsrArraySize 和 ICpuInfo::MSR_MOD_MAX 相等
static_assert (kMsrArraySize == ICpuInfo::MSR_MOD_MAX, "kMsrArraySize and MSR_MOD_MAX mismatch");
#endif


} // namespace xmrig


// 从 JSON 值中读取配置信息
bool xmrig::RxConfig::read(const rapidjson::Value &value)
{
    // 如果值是对象
    if (value.IsObject()) {
        // 从 JSON 中获取整数值，如果不存在则使用默认值
        m_threads         = Json::getInt(value, kInit, m_threads);
        m_initDatasetAVX2 = Json::getInt(value, kInitAVX2, m_initDatasetAVX2);
        // 从 JSON 中读取模式
        m_mode            = readMode(Json::getValue(value, kMode));
        // 从 JSON 中获取布尔值，如果不存在则使用默认值
        m_rdmsr           = Json::getBool(value, kRdmsr, m_rdmsr);

#       ifdef XMRIG_FEATURE_MSR
        // 从 JSON 中读取 MSR 配置
        readMSR(Json::getValue(value, kWrmsr));
#       endif

        // 从 JSON 中获取布尔值，如果不存在则使用默认值
        m_cacheQoS = Json::getBool(value, kCacheQoS, m_cacheQoS);

#       ifdef XMRIG_OS_LINUX
        // 从 JSON 中获取布尔值，如果不存在则使用默认值
        m_oneGbPages = Json::getBool(value, kOneGbPages, m_oneGbPages);
#       endif

#       ifdef XMRIG_FEATURE_HWLOC
        // 如果模式是 LightMode，则设置 numa 为 false
        if (m_mode == LightMode) {
            m_numa = false;

            return true;
        }

        // 从 JSON 中获取 NUMA 配置
        const auto &numa = Json::getValue(value, kNUMA);
        if (numa.IsArray()) {
            m_nodeset.reserve(numa.Size());

            // 遍历 NUMA 数组，将其添加到节点集合中
            for (const auto &node : numa.GetArray()) {
                if (node.IsUint()) {
                    m_nodeset.emplace_back(node.GetUint());
                }
            }
        }
        else if (numa.IsBool()) {
            // 从 JSON 中获取布尔值，如果不存在则使用默认值
            m_numa = numa.GetBool();
        }
#       endif

        // 从 JSON 中获取整数值，转换为枚举类型
        const auto mode = static_cast<uint32_t>(Json::getInt(value, kScratchpadPrefetchMode, static_cast<int>(m_scratchpadPrefetchMode)));
        if (mode < ScratchpadPrefetchMax) {
            m_scratchpadPrefetchMode = static_cast<ScratchpadPrefetchMode>(mode);
        }

        return true;
    }

    return false;
}

// 将配置信息转换为 JSON
rapidjson::Value xmrig::RxConfig::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 创建 JSON 对象
    Value obj(kObjectType);
    // 添加成员到 JSON 对象中
    obj.AddMember(StringRef(kInit),         m_threads, allocator);
    # 向对象添加成员，键为kInitAVX2，值为m_initDatasetAVX2，使用给定的分配器
    obj.AddMember(StringRef(kInitAVX2), m_initDatasetAVX2, allocator);
    
    # 向对象添加成员，键为kMode，值为modeName()的引用，使用给定的分配器
    obj.AddMember(StringRef(kMode), StringRef(modeName()), allocator);
    
    # 向对象添加成员，键为kOneGbPages，值为m_oneGbPages，使用给定的分配器
    obj.AddMember(StringRef(kOneGbPages), m_oneGbPages, allocator);
    
    # 向对象添加成员，键为kRdmsr，值为m_rdmsr，使用给定的分配器
    obj.AddMember(StringRef(kRdmsr), m_rdmsr, allocator);
// 如果定义了 XMRIG_FEATURE_MSR，则执行以下代码块
#ifdef XMRIG_FEATURE_MSR
    // 如果 m_msrPreset 不为空
    if (!m_msrPreset.empty()) {
        // 创建一个 JSON 数组对象 wrmsr
        Value wrmsr(kArrayType);
        // 预留空间给 wrmsr 数组
        wrmsr.Reserve(m_msrPreset.size(), allocator);

        // 遍历 m_msrPreset 中的元素，将其转换为 JSON 对象并添加到 wrmsr 数组中
        for (const auto &i : m_msrPreset) {
            wrmsr.PushBack(i.toJSON(doc), allocator);
        }

        // 将 wrmsr 数组添加到 obj 对象中，键名为 kWrmsr
        obj.AddMember(StringRef(kWrmsr), wrmsr, allocator);
    }
    // 如果 m_msrPreset 为空
    else {
        // 将 m_wrmsr 添加到 obj 对象中，键名为 kWrmsr
        obj.AddMember(StringRef(kWrmsr), m_wrmsr, allocator);
    }
// 如果未定义 XMRIG_FEATURE_MSR，则执行以下代码块
#else
    // 将 false 添加到 obj 对象中，键名为 kWrmsr
    obj.AddMember(StringRef(kWrmsr), false, allocator);
// 结束条件编译指令
#endif

// 将 m_cacheQoS 添加到 obj 对象中，键名为 kCacheQoS
obj.AddMember(StringRef(kCacheQoS), m_cacheQoS, allocator);

// 如果定义了 XMRIG_FEATURE_HWLOC，则执行以下代码块
#ifdef XMRIG_FEATURE_HWLOC
    // 如果 m_nodeset 不为空
    if (!m_nodeset.empty()) {
        // 创建一个 JSON 数组对象 numa
        Value numa(kArrayType);

        // 遍历 m_nodeset 中的元素，将其添加到 numa 数组中
        for (uint32_t i : m_nodeset) {
            numa.PushBack(i, allocator);
        }

        // 将 numa 数组添加到 obj 对象中，键名为 kNUMA
        obj.AddMember(StringRef(kNUMA), numa, allocator);
    }
    // 如果 m_nodeset 为空
    else {
        // 将 m_numa 添加到 obj 对象中，键名为 kNUMA
        obj.AddMember(StringRef(kNUMA), m_numa, allocator);
    }
// 结束条件编译指令
#endif

// 将 m_scratchpadPrefetchMode 转换为整数后添加到 obj 对象中，键名为 kScratchpadPrefetchMode
obj.AddMember(StringRef(kScratchpadPrefetchMode), static_cast<int>(m_scratchpadPrefetchMode), allocator);

// 返回 obj 对象
return obj;
// 如果定义了 XMRIG_FEATURE_HWLOC，则执行以下代码块
#ifdef XMRIG_FEATURE_HWLOC
// 结束条件编译指令
#endif

// 如果定义了 XMRIG_FEATURE_HWLOC，则执行以下代码块
#ifdef XMRIG_FEATURE_HWLOC
// 返回 m_nodeset，如果为空则返回一个空的 uint32_t 类型的 vector
std::vector<uint32_t> xmrig::RxConfig::nodeset() const
{
    if (!m_nodeset.empty()) {
        return m_nodeset;
    }

    return (m_numa && Cpu::info()->nodes() > 1) ? Cpu::info()->nodeset() : std::vector<uint32_t>();
}
// 结束条件编译指令
#endif

// 返回 modeNames 数组中索引为 m_mode 的元素
const char *xmrig::RxConfig::modeName() const
{
    return modeNames[m_mode];
}

// 返回 m_threads，如果小于等于 0，则根据 limit 和 CPU 线程数计算返回值
uint32_t xmrig::RxConfig::threads(uint32_t limit) const
{
    if (m_threads > 0) {
        return m_threads;
    }

    if (limit < 100) {
        return std::max(static_cast<uint32_t>(round(Cpu::info()->threads() * (limit / 100.0))), 1U);
    }

    return Cpu::info()->threads();
}

// 如果定义了 XMRIG_FEATURE_MSR，则执行以下代码块
#ifdef XMRIG_FEATURE_MSR
// 返回 modNames 数组中索引为 msrMod() 的元素
const char *xmrig::RxConfig::msrPresetName() const
{
    return modNames[msrMod()];
}

// 返回 m_msrPreset，如果 msrMod() 为 ICpuInfo::MSR_MOD_CUSTOM，则返回 m_msrPreset，否则返回 msrPresets[msrMod()]
const xmrig::MsrItems &xmrig::RxConfig::msrPreset() const
{
    const auto mod = msrMod();

    if (mod == ICpuInfo::MSR_MOD_CUSTOM) {
        return m_msrPreset;
    }

    return msrPresets[mod];
}

// 返回 msrMod()
uint32_t xmrig::RxConfig::msrMod() const
{
    # 如果wrmsr()函数返回false，则返回MSR_MOD_NONE
    if (!wrmsr()) {
        return ICpuInfo::MSR_MOD_NONE;
    }

    # 如果m_msrPreset不为空，则返回MSR_MOD_CUSTOM
    if (!m_msrPreset.empty()) {
        return ICpuInfo::MSR_MOD_CUSTOM;
    }

    # 返回Cpu::info()对象的msrMod()方法的返回值
    return Cpu::info()->msrMod();
}

# 读取 MSR 配置
void xmrig::RxConfig::readMSR(const rapidjson::Value &value)
{
    # 如果值是布尔类型
    if (value.IsBool()) {
        # 将值赋给 m_wrmsr
        m_wrmsr = value.GetBool();

        # 返回
        return;
    }

    # 如果值是整数类型
    if (value.IsInt()) {
        # 取值和 15 中的较小值
        const int i = std::min(value.GetInt(), 15);
        # 如果 i 大于等于 0
        if (i >= 0) {
            # 如果 CPU 厂商是 Intel
            if (Cpu::info()->vendor() == ICpuInfo::VENDOR_INTEL) {
                # 将 (0x1a4, i) 添加到 m_msrPreset
                m_msrPreset.emplace_back(0x1a4, i);
            }
        }
        else {
            # 将 m_wrmsr 设为 false
            m_wrmsr = false;
        }
    }

    # 如果值是数组类型
    if (value.IsArray()) {
        # 遍历数组中的每个元素
        for (const auto &i : value.GetArray()) {
            # 创建 MsrItem 对象
            MsrItem item(i);
            # 如果 item 是有效的
            if (item.isValid()) {
                # 将 item 添加到 m_msrPreset
                m_msrPreset.emplace_back(item);
            }
        }

        # 将 m_wrmsr 设为 m_msrPreset 是否为空
        m_wrmsr = !m_msrPreset.empty();
    }
}
#endif


# 读取模式配置
xmrig::RxConfig::Mode xmrig::RxConfig::readMode(const rapidjson::Value &value)
{
    # 如果值是无符号整数类型
    if (value.IsUint()) {
        # 将值转换为 Mode 类型，并取值和 ModeMax - 1 中的较小值
        return static_cast<Mode>(std::min(value.GetUint(), ModeMax - 1));
    }

    # 如果值是字符串类型
    if (value.IsString()) {
        # 将值转换为字符串
        auto mode = value.GetString();

        # 遍历 modeNames 数组
        for (size_t i = 0; i < modeNames.size(); i++) {
            # 如果 mode 和 modeNames[i] 相等（忽略大小写）
            if (strcasecmp(mode, modeNames[i]) == 0) {
                # 返回 Mode 类型的值
                return static_cast<Mode>(i);
            }
        }
    }

    # 返回 AutoMode
    return AutoMode;
}
```