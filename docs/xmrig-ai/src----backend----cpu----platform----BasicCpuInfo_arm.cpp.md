# `xmrig\src\backend\cpu\platform\BasicCpuInfo_arm.cpp`

```cpp
/*
 * XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据GNU通用公共许可证的条款发布
 * 由自由软件基金会发布的许可证的第3版或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "base/tools/String.h"

#include <array>
#include <cstring>
#include <fstream>
#include <thread>

#if __ARM_FEATURE_CRYPTO && !defined(__APPLE__)
#   include <sys/auxv.h>
#   if !defined(XMRIG_OS_FREEBSD)
#       include <asm/hwcap.h>
#   else
#       include <stdint.h>
#       include <machine/armreg.h>
#       ifndef ID_AA64ISAR0_AES_VAL
#           define ID_AA64ISAR0_AES_VAL ID_AA64ISAR0_AES
#       endif
#   endif
#endif

#include "backend/cpu/platform/BasicCpuInfo.h"
#include "3rdparty/rapidjson/document.h"

#if defined(XMRIG_OS_UNIX)
namespace xmrig {
    // 在UNIX系统上获取ARM CPU的名称
    extern String cpu_name_arm();
} // namespace xmrig
#elif defined(XMRIG_OS_MACOS)
#   include <sys/sysctl.h>
#endif

// 构造函数，初始化CPU信息
xmrig::BasicCpuInfo::BasicCpuInfo() :
    m_threads(std::thread::hardware_concurrency())  // 获取系统支持的线程数
{
    m_units.resize(m_threads);  // 调整m_units的大小以适应线程数
    for (int32_t i = 0; i < static_cast<int32_t>(m_threads); ++i) {
        m_units[i] = i;  // 初始化m_units数组
    }

#   if (XMRIG_ARM == 8)
    memcpy(m_brand, "ARMv8", 5);  // 如果是ARMv8架构，则将m_brand设置为"ARMv8"
#   else
    memcpy(m_brand, "ARMv7", 5);  // 如果不是ARMv8架构，则将m_brand设置为"ARMv7"
#   endif

#   if __ARM_FEATURE_CRYPTO
#   if defined(__APPLE__)
    m_flags.set(FLAG_AES, true);  // 如果支持AES加密，则设置FLAG_AES为true
#   elif defined(XMRIG_OS_FREEBSD)
    # 读取特殊寄存器 id_aa64isar0_el1 的值，并赋给变量 isar0
    uint64_t isar0 = READ_SPECIALREG(id_aa64isar0_el1);
    # 根据 id_aa64isar0_el1 寄存器中 AES 的值，设置 m_flags 中的 AES 标志位
    m_flags.set(FLAG_AES, ID_AA64ISAR0_AES_VAL(isar0) >= ID_AA64ISAR0_AES_BASE);
#   else
    # 设置标志位 FLAG_AES，根据硬件辅助信息获取是否支持 AES 指令集
    m_flags.set(FLAG_AES, getauxval(AT_HWCAP) & HWCAP_AES);
#   endif
#   endif

#   if defined(XMRIG_OS_UNIX)
    # 获取 ARM 架构的 CPU 名称
    auto name = cpu_name_arm();
    # 如果名称不为空，将名称拷贝到 m_brand 中
    if (!name.isNull()) {
        strncpy(m_brand, name, sizeof(m_brand) - 1);
    }
    # 设置标志位 FLAG_PDPE1GB，根据文件是否存在判断是否支持 1GB 大页
    m_flags.set(FLAG_PDPE1GB, std::ifstream("/sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages").good());
#   elif defined(XMRIG_OS_MACOS)
    # 获取 macOS 系统下 CPU 的品牌字符串
    size_t buflen = sizeof(m_brand);
    sysctlbyname("machdep.cpu.brand_string", &m_brand, &buflen, nullptr, 0);
#   endif
}

# 返回后端信息
const char *xmrig::BasicCpuInfo::backend() const
{
    return "basic/1";
}

# 返回线程数
xmrig::CpuThreads xmrig::BasicCpuInfo::threads(const Algorithm &algorithm, uint32_t) const
{
#   ifdef XMRIG_ALGO_GHOSTRIDER
    # 如果算法族为 GHOSTRIDER，则返回线程数为当前线程数和 8 的较小值
    if (algorithm.family() == Algorithm::GHOSTRIDER) {
        return CpuThreads(threads(), 8);
    }
#   endif
    # 否则返回当前线程数
    return CpuThreads(threads());
}

# 将 CPU 信息转换为 JSON 格式
rapidjson::Value xmrig::BasicCpuInfo::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    # 创建一个 JSON 对象
    Value out(kObjectType);

    # 添加 CPU 品牌信息到 JSON 对象中
    out.AddMember("brand",      StringRef(brand()), allocator);
    # 添加是否支持 AES 指令集到 JSON 对象中
    out.AddMember("aes",        hasAES(), allocator);
    # 添加是否支持 AVX2 指令集到 JSON 对象中
    out.AddMember("avx2",       false, allocator);
    # 添加是否为 64 位系统到 JSON 对象中（已废弃，将在下一个主要版本中移除）
    out.AddMember("x64",        is64bit(), allocator);
    # 添加是否为 64 位系统到 JSON 对象中
    out.AddMember("64_bit",     is64bit(), allocator);
    # 添加 L2 缓存大小到 JSON 对象中
    out.AddMember("l2",         static_cast<uint64_t>(L2()), allocator);
    # 添加 L3 缓存大小到 JSON 对象中
    out.AddMember("l3",         static_cast<uint64_t>(L3()), allocator);
    # 添加核心数到 JSON 对象中
    out.AddMember("cores",      static_cast<uint64_t>(cores()), allocator);
    # 添加线程数到 JSON 对象中
    out.AddMember("threads",    static_cast<uint64_t>(threads()), allocator);
    # 添加处理器包数到 JSON 对象中
    out.AddMember("packages",   static_cast<uint64_t>(packages()), allocator);
    # 添加 NUMA 节点数到 JSON 对象中
    out.AddMember("nodes",      static_cast<uint64_t>(nodes()), allocator);
    # 添加后端信息到 JSON 对象中
    out.AddMember("backend",    StringRef(backend()), allocator);
    # 添加 MSR 信息到 JSON 对象中
    out.AddMember("msr",        "none", allocator);
    # 向输出对象中添加一个名为"assembly"的成员，值为"none"，使用给定的分配器
    out.AddMember("assembly",   "none", allocator);
# 如果 XMRIG_ARM 等于 8，则添加 "arch" 键值对为 "aarch64" 到输出对象中
out.AddMember("arch", "aarch64", allocator);
# 否则，添加 "arch" 键值对为 "aarch32" 到输出对象中
out.AddMember("arch", "aarch32", allocator);

# 创建一个数组对象 flags
Value flags(kArrayType);

# 如果具有 AES 功能，则向 flags 数组中添加 "aes" 元素
if (hasAES()) {
    flags.PushBack("aes", allocator);
}

# 将 flags 数组作为 "flags" 键的值添加到输出对象中
out.AddMember("flags", flags, allocator);

# 返回输出对象
return out;
}
```