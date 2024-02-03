# `xmrig\src\backend\cpu\CpuConfig.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   它可以是许可证的第3版，也可以是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/cpu/CpuConfig.h"  // 包含CpuConfig头文件
#include "3rdparty/rapidjson/document.h"  // 包含rapidjson库的document头文件
#include "backend/cpu/CpuConfig_gen.h"  // 包含CpuConfig_gen头文件
#include "backend/cpu/Cpu.h"  // 包含Cpu头文件
#include "base/io/json/Json.h"  // 包含Json头文件

#include <algorithm>  // 包含算法头文件


namespace xmrig {  // 命名空间xmrig开始

const char *CpuConfig::kEnabled             = "enabled";  // 定义CpuConfig类的静态成员变量kEnabled
const char *CpuConfig::kField               = "cpu";  // 定义CpuConfig类的静态成员变量kField
const char *CpuConfig::kHugePages           = "huge-pages";  // 定义CpuConfig类的静态成员变量kHugePages
const char *CpuConfig::kHugePagesJit        = "huge-pages-jit";  // 定义CpuConfig类的静态成员变量kHugePagesJit
const char *CpuConfig::kHwAes               = "hw-aes";  // 定义CpuConfig类的静态成员变量kHwAes
const char *CpuConfig::kMaxThreadsHint      = "max-threads-hint";  // 定义CpuConfig类的静态成员变量kMaxThreadsHint
const char *CpuConfig::kMemoryPool          = "memory-pool";  // 定义CpuConfig类的静态成员变量kMemoryPool
const char *CpuConfig::kPriority            = "priority";  // 定义CpuConfig类的静态成员变量kPriority
const char *CpuConfig::kYield               = "yield";  // 定义CpuConfig类的静态成员变量kYield

#ifdef XMRIG_FEATURE_ASM
const char *CpuConfig::kAsm                 = "asm";  // 如果定义了XMRIG_FEATURE_ASM，则定义CpuConfig类的静态成员变量kAsm
#endif

#ifdef XMRIG_ALGO_ARGON2
const char *CpuConfig::kArgon2Impl          = "argon2-impl";  // 如果定义了XMRIG_ALGO_ARGON2，则定义CpuConfig类的静态成员变量kArgon2Impl
#endif


extern template class Threads<CpuThreads>;  // 外部模板类实例化声明


} // 命名空间xmrig结束


bool xmrig::CpuConfig::isHwAES() const  // 定义CpuConfig类的成员函数isHwAES
{
    return (m_aes == AES_AUTO ? (Cpu::info()->hasAES() ? AES_HW : AES_SOFT) : m_aes) == AES_HW;  // 返回m_aes是否为AES_HW
}
// 将 CpuConfig 对象转换为 JSON 格式
rapidjson::Value xmrig::CpuConfig::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 创建一个空的 JSON 对象
    Value obj(kObjectType);

    // 添加成员：启用状态
    obj.AddMember(StringRef(kEnabled),      m_enabled, allocator);
    // 添加成员：HugePages 大页内存设置
    obj.AddMember(StringRef(kHugePages),    m_hugePageSize == 0 || m_hugePageSize == kDefaultHugePageSizeKb ? Value(isHugePages()) : Value(static_cast<uint32_t>(m_hugePageSize)), allocator);
    // 添加成员：HugePagesJit 大页内存即时编译设置
    obj.AddMember(StringRef(kHugePagesJit), m_hugePagesJit, allocator);
    // 添加成员：HwAes 硬件加密设置
    obj.AddMember(StringRef(kHwAes),        m_aes == AES_AUTO ? Value(kNullType) : Value(m_aes == AES_HW), allocator);
    // 添加成员：优先级设置
    obj.AddMember(StringRef(kPriority),     priority() != -1 ? Value(priority()) : Value(kNullType), allocator);
    // 添加成员：内存池大小设置
    obj.AddMember(StringRef(kMemoryPool),   m_memoryPool < 1 ? Value(m_memoryPool < 0) : Value(m_memoryPool), allocator);
    // 添加成员：Yield 设置
    obj.AddMember(StringRef(kYield),        m_yield, allocator);

    // 如果线程为空，则添加最大线程数提示
    if (m_threads.isEmpty()) {
        obj.AddMember(StringRef(kMaxThreadsHint), m_limit, allocator);
    }

#   ifdef XMRIG_FEATURE_ASM
    // 添加汇编设置
    obj.AddMember(StringRef(kAsm), m_assembly.toJSON(), allocator);
#   endif

#   ifdef XMRIG_ALGO_ARGON2
    // 添加 Argon2 实现设置
    obj.AddMember(StringRef(kArgon2Impl), m_argon2Impl.toJSON(), allocator);
#   endif

    // 将线程设置转换为 JSON 格式
    m_threads.toJSON(obj, doc);

    // 返回 JSON 对象
    return obj;
}

// 获取内存池大小
size_t xmrig::CpuConfig::memPoolSize() const
{
    // 如果内存池小于 0，则返回线程数和 L3 缓存大小右移 21 位的最大值，否则返回内存池大小
    return m_memoryPool < 0 ? std::max(Cpu::info()->threads(), Cpu::info()->L3() >> 21) : m_memoryPool;
}

// 获取 CPU 启动数据
std::vector<xmrig::CpuLaunchData> xmrig::CpuConfig::get(const Miner *miner, const Algorithm &algorithm) const
{
    // 如果算法家族是 KAWPOW，则返回空向量
    if (algorithm.family() == Algorithm::KAWPOW) {
        return {};
    }

    // 初始化 CPU 启动数据向量
    std::vector<CpuLaunchData> out;
    // 获取算法对应的线程设置
    const auto &threads = m_threads.get(algorithm);

    // 如果线程为空，则返回空向量
    if (threads.isEmpty()) {
        return out;
    }

    // 获取线程数量
    const size_t count = threads.count();
    // 预留空间
    out.reserve(count);

    // 初始化亲和性向量
    std::vector<int64_t> affinities;
    affinities.reserve(count);
}
    # 遍历线程数据，将每个线程的亲和性添加到affinities向量中
    for (const auto& thread : threads.data()) {
        affinities.emplace_back(thread.affinity());
    }
    
    # 遍历线程数据，为每个线程创建一个新的对象，并将其添加到输出向量中
    for (const auto &thread : threads.data()) {
        out.emplace_back(miner, algorithm, *this, thread, count, affinities);
    }
    
    # 返回输出向量
    return out;
# 读取 CPU 配置信息
void xmrig::CpuConfig::read(const rapidjson::Value &value)
{
    # 如果值是对象
    if (value.IsObject()) {
        # 从 JSON 中获取布尔值，如果不存在则使用默认值
        m_enabled      = Json::getBool(value, kEnabled, m_enabled);
        # 从 JSON 中获取布尔值，如果不存在则使用默认值
        m_hugePagesJit = Json::getBool(value, kHugePagesJit, m_hugePagesJit);
        # 从 JSON 中获取无符号整数，如果不存在则使用默认值
        m_limit        = Json::getUint(value, kMaxThreadsHint, m_limit);
        # 从 JSON 中获取布尔值，如果不存在则使用默认值
        m_yield        = Json::getBool(value, kYield, m_yield);

        # 设置 AES 模式
        setAesMode(Json::getValue(value, kHwAes));
        # 设置大页内存
        setHugePages(Json::getValue(value, kHugePages));
        # 设置内存池
        setMemoryPool(Json::getValue(value, kMemoryPool));
        # 设置优先级
        setPriority(Json::getInt(value,  kPriority, -1));

#       ifdef XMRIG_FEATURE_ASM
        # 如果定义了 XMRIG_FEATURE_ASM，则从 JSON 中获取值
        m_assembly = Json::getValue(value, kAsm);
#       endif

#       ifdef XMRIG_ALGO_ARGON2
        # 如果定义了 XMRIG_ALGO_ARGON2，则从 JSON 中获取字符串
        m_argon2Impl = Json::getString(value, kArgon2Impl);
#       endif

        # 读取线程配置
        m_threads.read(value);

        # 生成配置
        generate();
    }
    # 如果值是布尔值
    else if (value.IsBool()) {
        # 直接设置启用状态
        m_enabled = value.GetBool();

        # 生成配置
        generate();
    }
    # 如果值不是对象也不是布尔值
    else {
        # 生成配置
        generate();
    }
}

# 生成配置
void xmrig::CpuConfig::generate()
{
    # 如果未启用或者线程配置包含通配符，则返回
    if (!isEnabled() || m_threads.has("*")) {
        return;
    }

    # 初始化计数器
    size_t count = 0;

    # 生成 CN 算法配置
    count += xmrig::generate<Algorithm::CN>(m_threads, m_limit);
    count += xmrig::generate<Algorithm::CN_LITE>(m_threads, m_limit);
    count += xmrig::generate<Algorithm::CN_HEAVY>(m_threads, m_limit);
    count += xmrig::generate<Algorithm::CN_PICO>(m_threads, m_limit);
    count += xmrig::generate<Algorithm::CN_FEMTO>(m_threads, m_limit);
    count += xmrig::generate<Algorithm::RANDOM_X>(m_threads, m_limit);
    count += xmrig::generate<Algorithm::ARGON2>(m_threads, m_limit);
    count += xmrig::generate<Algorithm::GHOSTRIDER>(m_threads, m_limit);

    # 如果有生成配置，则设置保存标志
    m_shouldSave |= count > 0;
}

# 设置 AES 模式
void xmrig::CpuConfig::setAesMode(const rapidjson::Value &value)
{
    # 如果值是布尔值
    if (value.IsBool()) {
        # 根据布尔值设置 AES 模式
        m_aes = value.GetBool() ? AES_HW : AES_SOFT;
    }
    # 如果值不是布尔值
    else {
        # 设置为自动模式
        m_aes = AES_AUTO;
    }
}

# 设置大页内存
void xmrig::CpuConfig::setHugePages(const rapidjson::Value &value)
{
    # 如果值是布尔类型
    if (value.IsBool()) {
        # 如果值为真，设置 m_hugePageSize 为默认的巨大页面大小，否则设置为0
        m_hugePageSize = value.GetBool() ? kDefaultHugePageSizeKb : 0U;
    }
    # 如果值是无符号整数类型
    else if (value.IsUint()) {
        # 获取值并存储在 size 变量中
        const uint32_t size = value.GetUint();
        # 如果 size 小于1GB页面大小，设置 m_hugePageSize 为 size，否则设置为默认的巨大页面大小
        m_hugePageSize = size < kOneGbPageSizeKb ? size : kDefaultHugePageSizeKb;
    }
# 设置内存池配置的函数，参数为 rapidjson::Value 类型的引用
void xmrig::CpuConfig::setMemoryPool(const rapidjson::Value &value)
{
    # 如果传入的值是布尔类型
    if (value.IsBool()) {
        # 如果布尔值为真，则将内存池设置为-1，否则设置为0
        m_memoryPool = value.GetBool() ? -1 : 0;
    }
    # 如果传入的值是整数类型
    else if (value.IsInt()) {
        # 将内存池设置为传入的整数值
        m_memoryPool = value.GetInt();
    }
}
```