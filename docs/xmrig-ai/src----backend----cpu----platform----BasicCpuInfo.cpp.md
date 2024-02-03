# `xmrig\src\backend\cpu\platform\BasicCpuInfo.cpp`

```cpp
/* XMRig
 * 版权声明
 */
#include <algorithm>  // 包含算法库
#include <array>  // 包含数组库
#include <cstring>  // 包含字符串库
#include <thread>  // 包含线程库

#ifdef _MSC_VER
#   include <intrin.h>  // 如果是 MSVC 编译器，包含内联汇编库
#else
#   include <cpuid.h>  // 如果不是 MSVC 编译器，包含 CPUID 库
#endif

#include "crypto/cn/CryptoNight_monero.h"  // 包含加密库
#ifdef XMRIG_VAES
#   include "crypto/cn/CryptoNight_x86_vaes.h"  // 如果支持 VAES，包含 VAES 加密库
#endif

#include "backend/cpu/platform/BasicCpuInfo.h"  // 包含 CPU 基本信息库
#include "3rdparty/rapidjson/document.h"  // 包含 rapidjson 库
#include "crypto/common/Assembly.h"  // 包含通用汇编库

#define VENDOR_ID                  (0)  // 定义厂商 ID
#define PROCESSOR_INFO             (1)  // 定义处理器信息
#define EXTENDED_FEATURES          (7)  // 定义扩展特性
#define PROCESSOR_EXT_INFO         (0x80000001)  // 定义处理器扩展信息
#define PROCESSOR_BRAND_STRING_1   (0x80000002)  // 定义处理器品牌字符串1
#define PROCESSOR_BRAND_STRING_2   (0x80000003)  // 定义处理器品牌字符串2
#define PROCESSOR_BRAND_STRING_3   (0x80000004)  // 定义处理器品牌字符串3

#define EAX_Reg  (0)  // 定义 EAX 寄存器
#define EBX_Reg  (1)  // 定义 EBX 寄存器
#define ECX_Reg  (2)  // 定义 ECX 寄存器
#define EDX_Reg  (3)  // 定义 EDX 寄存器

namespace xmrig {
  
constexpr size_t kCpuFlagsSize                                  = 15;  // 定义 CPU 标志位大小
static const std::array<const char *, kCpuFlagsSize> flagNames  = { "aes", "vaes", "avx", "avx2", "avx512f", "bmi2", "osxsave", "pdpe1gb", "sse2", "ssse3", "sse4.1", "xop", "popcnt", "cat_l3", "vm" };  // 定义 CPU 标志位名称数组
# 检查 CPU 标志位大小是否与 ICpuInfo::FLAG_MAX 相等，如果不相等则触发静态断言错误
static_assert(kCpuFlagsSize == ICpuInfo::FLAG_MAX, "kCpuFlagsSize and FLAG_MAX mismatch");

#ifdef XMRIG_FEATURE_MSR
# 如果定义了 XMRIG_FEATURE_MSR，则定义 kMsrArraySize 为 6
constexpr size_t kMsrArraySize                                  = 6;
# 定义包含 MSR 名称的静态常量数组 msrNames
static const std::array<const char *, kMsrArraySize> msrNames   = { MSR_NAMES_LIST };
# 检查 kMsrArraySize 是否与 ICpuInfo::MSR_MOD_MAX 相等，如果不相等则触发静态断言错误
static_assert(kMsrArraySize == ICpuInfo::MSR_MOD_MAX, "kMsrArraySize and MSR_MOD_MAX mismatch");
#endif

# 定义一个内联函数 cpuid，用于获取 CPU 信息
static inline void cpuid(uint32_t level, int32_t output[4])
{
    # 将 output 数组清零
    memset(output, 0, sizeof(int32_t) * 4);

    # 根据不同的编译器使用不同的 CPUID 指令
#   ifdef _MSC_VER
    __cpuidex(output, static_cast<int>(level), 0);
#   else
    __cpuid_count(level, 0, output[0], output[1], output[2], output[3]);
#   endif
}

# 定义一个函数 cpu_brand_string，用于获取 CPU 品牌字符串
static void cpu_brand_string(char out[64 + 6]) {
    # 定义一个存储 CPU 信息的数组
    int32_t cpu_info[4] = { 0 };
    # 定义一个缓冲区
    char buf[64]        = { 0 };

    # 获取 CPU 厂商 ID
    cpuid(VENDOR_ID, cpu_info);

    # 如果 CPU 支持 CPUID 指令的最大扩展功能，则获取 CPU 品牌字符串
    if (cpu_info[EAX_Reg] >= 4) {
        for (uint32_t i = 0; i < 4; i++) {
            cpuid(0x80000002 + i, cpu_info);
            memcpy(buf + (i * 16), cpu_info, sizeof(cpu_info));
        }
    }

    # 处理 CPU 品牌字符串，去除多余的空格
    size_t pos        = 0;
    const size_t size = strlen(buf);

    for (size_t i = 0; i < size; ++i) {
        if (buf[i] == ' ' && ((pos > 0 && out[pos - 1] == ' ') || pos == 0)) {
            continue;
        }

        out[pos++] = buf[i];
    }

    if (pos > 0 && out[pos - 1] == ' ') {
        out[pos - 1] = '\0';
    }
}

# 定义一个内联函数，用于检查 CPU 是否支持特定的功能
static inline bool has_feature(uint32_t level, uint32_t reg, int32_t bit)
{
    # 定义一个存储 CPU 信息的数组
    int32_t cpu_info[4] = { 0 };
    # 获取 CPU 信息
    cpuid(level, cpu_info);

    # 检查 CPU 是否支持特定的功能
    return (cpu_info[reg] & bit) != 0;
}

# 定义一个内联函数，用于获取指定位范围内的值
static inline int32_t get_masked(int32_t val, int32_t h, int32_t l)
{
    # 对输入值进行位掩码操作，获取指定位范围内的值
    val &= (0x7FFFFFFF >> (31 - (h - l))) << l;
    return val >> l;
}

# 定义一个内联函数，用于获取 XSAVE 扩展状态寄存器值
static inline uint64_t xgetbv()
{
# 根据不同的编译器使用不同的 XGETBV 指令
#ifdef _MSC_VER
    return _xgetbv(_XCR_XFEATURE_ENABLED_MASK);
#else
    uint32_t eax_reg = 0;
    uint32_t edx_reg = 0;
    __asm__ __volatile__("xgetbv": "=a"(eax_reg), "=d"(edx_reg) : "c"(0) : "cc");
    return (static_cast<uint64_t>(edx_reg) << 32) | eax_reg;
#endif
}
# 检查处理器是否支持 AVX 指令集
static inline bool has_xcr_avx()    { return (xgetbv() & 0x06) == 0x06; }
# 检查处理器是否支持 AVX512 指令集
static inline bool has_xcr_avx512() { return (xgetbv() & 0xE6) == 0xE6; }
# 检查处理器是否支持 OSXSAVE 指令集
static inline bool has_osxsave()    { return has_feature(PROCESSOR_INFO,        ECX_Reg, 1 << 27); }
# 检查处理器是否支持 AES-NI 指令集
static inline bool has_aes_ni()     { return has_feature(PROCESSOR_INFO,        ECX_Reg, 1 << 25); }
# 检查处理器是否支持 AVX 指令集
static inline bool has_avx()        { return has_feature(PROCESSOR_INFO,        ECX_Reg, 1 << 28) && has_osxsave() && has_xcr_avx(); }
# 检查处理器是否支持 AVX2 指令集
static inline bool has_avx2()       { return has_feature(EXTENDED_FEATURES,     EBX_Reg, 1 << 5) && has_osxsave() && has_xcr_avx(); }
# 检查处理器是否支持 VAES 指令集
static inline bool has_vaes()       { return has_feature(EXTENDED_FEATURES,     ECX_Reg, 1 << 9) && has_osxsave() && has_xcr_avx(); }
# 检查处理器是否支持 AVX512F 指令集
static inline bool has_avx512f()    { return has_feature(EXTENDED_FEATURES,     EBX_Reg, 1 << 16) && has_osxsave() && has_xcr_avx512(); }
# 检查处理器是否支持 BMI2 指令集
static inline bool has_bmi2()       { return has_feature(EXTENDED_FEATURES,     EBX_Reg, 1 << 8); }
# 检查处理器是否支持 PDPE1GB 指令集
static inline bool has_pdpe1gb()    { return has_feature(PROCESSOR_EXT_INFO,    EDX_Reg, 1 << 26); }
# 检查处理器是否支持 SSE2 指令集
static inline bool has_sse2()       { return has_feature(PROCESSOR_INFO,        EDX_Reg, 1 << 26); }
# 检查处理器是否支持 SSSE3 指令集
static inline bool has_ssse3()      { return has_feature(PROCESSOR_INFO,        ECX_Reg, 1 << 9); }
# 检查处理器是否支持 SSE4.1 指令集
static inline bool has_sse41()      { return has_feature(PROCESSOR_INFO,        ECX_Reg, 1 << 19); }
# 检查处理器是否支持 XOP 指令集
static inline bool has_xop()        { return has_feature(0x80000001,            ECX_Reg, 1 << 11); }
# 检查处理器是否支持 POPCNT 指令集
static inline bool has_popcnt()     { return has_feature(PROCESSOR_INFO,        ECX_Reg, 1 << 23); }
# 检查处理器是否支持 CAT L3 指令集
static inline bool has_cat_l3()     { return has_feature(EXTENDED_FEATURES,     EBX_Reg, 1 << 15) && has_feature(0x10, EBX_Reg, 1 << 1); }
# 检查处理器是否为虚拟机
static inline bool is_vm()          { return has_feature(PROCESSOR_INFO,        ECX_Reg, 1 << 31); }
# 返回处理器是否支持 AVX2 指令集
int cpu_flags_has_avx2()    { return xmrig::has_avx2(); }
// 检查 CPU 是否支持 AVX512F 指令集
int cpu_flags_has_avx512f() { return xmrig::has_avx512f(); }
// 检查 CPU 是否支持 SSE2 指令集
int cpu_flags_has_sse2()    { return xmrig::has_sse2(); }
// 检查 CPU 是否支持 SSSE3 指令集
int cpu_flags_has_ssse3()   { return xmrig::has_ssse3(); }
// 检查 CPU 是否支持 XOP 指令集
int cpu_flags_has_xop()     { return xmrig::has_xop(); }

// 创建 BasicCpuInfo 对象的构造函数
xmrig::BasicCpuInfo::BasicCpuInfo() :
    // 获取系统可用的线程数
    m_threads(std::thread::hardware_concurrency())
{
    // 获取 CPU 品牌信息
    cpu_brand_string(m_brand);

    // 设置 CPU 标志位
    m_flags.set(FLAG_AES,     has_aes_ni());
    m_flags.set(FLAG_AVX,     has_avx());
    m_flags.set(FLAG_AVX2,    has_avx2());
    m_flags.set(FLAG_VAES,    has_vaes());
    m_flags.set(FLAG_AVX512F, has_avx512f());
    m_flags.set(FLAG_BMI2,    has_bmi2());
    m_flags.set(FLAG_OSXSAVE, has_osxsave());
    m_flags.set(FLAG_PDPE1GB, has_pdpe1gb());
    m_flags.set(FLAG_SSE2,    has_sse2());
    m_flags.set(FLAG_SSSE3,   has_ssse3());
    m_flags.set(FLAG_SSE41,   has_sse41());
    m_flags.set(FLAG_XOP,     has_xop());
    m_flags.set(FLAG_POPCNT,  has_popcnt());
    m_flags.set(FLAG_CAT_L3,  has_cat_l3());
    m_flags.set(FLAG_VM,      is_vm());

    // 根据线程数调整单位数组大小
    m_units.resize(m_threads);
    for (int32_t i = 0; i < static_cast<int32_t>(m_threads); ++i) {
        m_units[i] = i;
    }

#   ifdef XMRIG_FEATURE_ASM
    }
#   endif

    // 设置 cn_sse41_enabled 和 cn_vaes_enabled 标志位
    cn_sse41_enabled = has(FLAG_SSE41);
    cn_vaes_enabled = has(FLAG_VAES);
}

// 返回后端信息
const char *xmrig::BasicCpuInfo::backend() const
{
    return "basic/1";
}

// 返回 CPU 线程数
xmrig::CpuThreads xmrig::BasicCpuInfo::threads(const Algorithm &algorithm, uint32_t) const
{
    const size_t count = std::thread::hardware_concurrency();

    if (count == 1) {
        return 1;
    }

    const auto f = algorithm.family();

#   ifdef XMRIG_ALGO_CN_LITE
    if (f == Algorithm::CN_LITE) {
        return CpuThreads(count, 1);
    }
#   endif

#   ifdef XMRIG_ALGO_CN_PICO
    if (f == Algorithm::CN_PICO) {
        return CpuThreads(count, 2);
    }
#   endif

#   ifdef XMRIG_ALGO_CN_FEMTO
    if (f == Algorithm::CN_FEMTO) {
        return CpuThreads(count, 2);
    }
#   endif

#   ifdef XMRIG_ALGO_CN_HEAVY
    # 如果算法是CN_HEAVY
    if (f == Algorithm::CN_HEAVY) {
        # 返回CPU线程数，最大为count/4，最小为1
        return CpuThreads(std::max<size_t>(count / 4, 1), 1);
    }
#   endif
#   ifdef XMRIG_ALGO_RANDOMX
    // 如果算法是 RANDOM_X
    if (f == Algorithm::RANDOM_X) {
        // 如果算法是 RX_WOW，则返回 count
        if (algorithm == Algorithm::RX_WOW) {
            return count;
        }
        // 否则返回 count 的一半或者 1 中的较大值
        return std::max<size_t>(count / 2, 1);
    }
#   endif
#   ifdef XMRIG_ALGO_ARGON2
    // 如果算法是 ARGON2，则返回 count
    if (f == Algorithm::ARGON2) {
        return count;
    }
#   endif
#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果算法是 GHOSTRIDER，则返回最大值为 count 的一半或者 1 的 CpuThreads 对象
    if (f == Algorithm::GHOSTRIDER) {
        return CpuThreads(std::max<size_t>(count / 2, 1), 8);
    }
#   endif
    // 返回最大值为 count 的一半或者 1 的 CpuThreads 对象
    return CpuThreads(std::max<size_t>(count / 2, 1), 1);
}

rapidjson::Value xmrig::BasicCpuInfo::toJSON(rapidjson::Document &doc) const
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();
    // 创建一个对象
    Value out(kObjectType);
    // 添加成员到对象中
    out.AddMember("brand",      StringRef(brand()), allocator);
    out.AddMember("family",     m_family, allocator);
    out.AddMember("model",      m_model, allocator);
    out.AddMember("stepping",   m_stepping, allocator);
    out.AddMember("proc_info",  m_procInfo, allocator);
    out.AddMember("aes",        hasAES(), allocator);
    out.AddMember("avx2",       hasAVX2(), allocator);
    out.AddMember("x64",        is64bit(), allocator); // DEPRECATED will be removed in the next major release.
    out.AddMember("64_bit",     is64bit(), allocator);
    out.AddMember("l2",         static_cast<uint64_t>(L2()), allocator);
    out.AddMember("l3",         static_cast<uint64_t>(L3()), allocator);
    out.AddMember("cores",      static_cast<uint64_t>(cores()), allocator);
    out.AddMember("threads",    static_cast<uint64_t>(threads()), allocator);
    out.AddMember("packages",   static_cast<uint64_t>(packages()), allocator);
    out.AddMember("nodes",      static_cast<uint64_t>(nodes()), allocator);
    out.AddMember("backend",    StringRef(backend()), allocator);
#   ifdef XMRIG_FEATURE_MSR
    // 如果支持 MSR，则添加对应的名称
    out.AddMember("msr",        StringRef(msrNames[msrMod()]), allocator);
#   else
    // 否则添加 "none"
    out.AddMember("msr",        "none", allocator);
#   endif
#   ifdef XMRIG_FEATURE_ASM
    # 如果定义了 XMRIG_FEATURE_ASM，则将汇编信息添加到输出对象中
    out.AddMember("assembly",   StringRef(Assembly(assembly()).toString()), allocator);
#   else
    # 如果未定义 XMRIG_FEATURE_ASM，则将 "none" 添加到输出对象中
    out.AddMember("assembly",   "none", allocator);
#   endif

#   if defined(__x86_64__) || defined(_M_AMD64)
    # 如果定义了 __x86_64__ 或 _M_AMD64，则将 "x86_64" 添加到输出对象中
    out.AddMember("arch", "x86_64", allocator);
#   else
    # 如果未定义 __x86_64__ 或 _M_AMD64，则将 "x86" 添加到输出对象中
    out.AddMember("arch", "x86", allocator);
#   endif

    # 创建一个数组对象来存储标志
    Value flags(kArrayType);

    # 遍历标志名称列表，如果标志被设置，则将其添加到 flags 数组中
    for (size_t i = 0; i < flagNames.size(); ++i) {
        if (m_flags.test(i)) {
            flags.PushBack(StringRef(flagNames[i]), allocator);
        }
    }

    # 将存储标志的数组对象添加到输出对象中
    out.AddMember("flags", flags, allocator);

    # 返回输出对象
    return out;
}
```