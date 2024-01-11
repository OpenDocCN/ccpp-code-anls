# `xmrig\src\core\config\ConfigTransform.cpp`

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

#include "core/config/ConfigTransform.h"  // 引入ConfigTransform.h文件
#include "base/crypto/Algorithm.h"  // 引入Algorithm.h文件
#include "base/kernel/interfaces/IConfig.h"  // 引入IConfig.h文件
#include "base/net/stratum/Pool.h"  // 引入Pool.h文件
#include "base/net/stratum/Pools.h"  // 引入Pools.h文件
#include "core/config/Config.h"  // 引入Config.h文件
#include "crypto/cn/CnHash.h"  // 引入CnHash.h文件

#ifdef XMRIG_ALGO_RANDOMX
#   include "crypto/rx/RxConfig.h"  // 如果定义了XMRIG_ALGO_RANDOMX，则引入RxConfig.h文件
#endif

#ifdef XMRIG_FEATURE_BENCHMARK
#   include "base/net/stratum/benchmark/BenchConfig.h"  // 如果定义了XMRIG_FEATURE_BENCHMARK，则引入BenchConfig.h文件
#endif

namespace xmrig
{
static const char *kAffinity    = "affinity";  // 定义常量kAffinity为"affinity"
static const char *kAsterisk    = "*";  // 定义常量kAsterisk为"*"
static const char *kEnabled     = "enabled";  // 定义常量kEnabled为"enabled"
static const char *kIntensity   = "intensity";  // 定义常量kIntensity为"intensity"
static const char *kThreads     = "threads";  // 定义常量kThreads为"threads"

static inline uint64_t intensity(uint64_t av)  // 定义内联函数intensity，参数为uint64_t类型的av
{
    switch (av) {  // 开始switch语句，根据av的值进行不同的处理
    case CnHash::AV_SINGLE:  // 如果av的值等于CnHash::AV_SINGLE
    case CnHash::AV_SINGLE_SOFT:  // 或者av的值等于CnHash::AV_SINGLE_SOFT
        return 1;  // 返回1
    case CnHash::AV_DOUBLE_SOFT:  // 如果av的值等于CnHash::AV_DOUBLE_SOFT
    case CnHash::AV_DOUBLE:  // 或者av的值等于CnHash::AV_DOUBLE
        return 2;  // 返回2
    case CnHash::AV_TRIPLE_SOFT:  // 如果av的值等于CnHash::AV_TRIPLE_SOFT
    case CnHash::AV_TRIPLE:  // 或者av的值等于CnHash::AV_TRIPLE
        return 3;  // 返回3
    case CnHash::AV_QUAD_SOFT:  // 如果av的值等于CnHash::AV_QUAD_SOFT
    case CnHash::AV_QUAD:  // 或者av的值等于CnHash::AV_QUAD
        return 4;  // 返回4
    case CnHash::AV_PENTA_SOFT:  // 如果av的值等于CnHash::AV_PENTA_SOFT
    case CnHash::AV_PENTA:  // 或者av的值等于CnHash::AV_PENTA
        return 5;  // 返回5
    default:  // 如果av的值不满足以上任何一个case
        break;  // 跳出switch语句
    }
    # 返回整数1
    return 1;
// 检查给定的 av 是否为硬件加密
static inline bool isHwAes(uint64_t av)
{
    return av == CnHash::AV_SINGLE || av == CnHash::AV_DOUBLE || (av > CnHash::AV_DOUBLE_SOFT && av < CnHash::AV_TRIPLE_SOFT);
}

// 结束 xmrig 命名空间
} // namespace xmrig

// 对配置进行最终处理
void xmrig::ConfigTransform::finalize(rapidjson::Document &doc)
{
    using namespace rapidjson;
    auto &allocator = doc.GetAllocator();

    // 调用基类的 finalize 方法
    BaseTransform::finalize(doc);

    // 如果线程数不为 0
    if (m_threads) {
        // 如果文档中没有 CPU 配置字段，则添加一个
        if (!doc.HasMember(CpuConfig::kField)) {
            doc.AddMember(StringRef(CpuConfig::kField), Value(kObjectType), allocator);
        }

        // 创建一个 profile 对象，并添加到 CPU 配置字段中
        Value profile(kObjectType);
        profile.AddMember(StringRef(kIntensity), m_intensity, allocator);
        profile.AddMember(StringRef(kThreads),   m_threads, allocator);
        profile.AddMember(StringRef(kAffinity),  m_affinity, allocator);

#       ifdef XMRIG_ALGO_KAWPOW
        // 如果定义了 XMRIG_ALGO_KAWPOW，则在 CPU 配置字段中添加 kKAWPOW 字段
        doc[CpuConfig::kField].AddMember(StringRef(Algorithm::kKAWPOW), false, doc.GetAllocator());
#       endif
        // 在 CPU 配置字段中添加一个 profile 对象
        doc[CpuConfig::kField].AddMember(StringRef(kAsterisk), profile, doc.GetAllocator());
    }

#   ifdef XMRIG_FEATURE_OPENCL
    // 如果定义了 XMRIG_FEATURE_OPENCL，则设置文档中的 Config::kOcl 为 true
    if (m_opencl) {
        set(doc, Config::kOcl, kEnabled, true);
    }
#   endif
}

// 对配置进行转换
void xmrig::ConfigTransform::transform(rapidjson::Document &doc, int key, const char *arg)
{
    // 调用基类的 transform 方法
    BaseTransform::transform(doc, key, arg);

    // 根据不同的 key 进行不同的转换操作
    switch (key) {
    case IConfig::AVKey:           /* --av */
    case IConfig::CPUPriorityKey:  /* --cpu-priority */
    case IConfig::ThreadsKey:      /* --threads */
    case IConfig::HugePageSizeKey: /* --hugepage-size */
        // 将 arg 转换为 uint64_t 类型，并进行相应的转换操作
        return transformUint64(doc, key, static_cast<uint64_t>(strtol(arg, nullptr, 10)));

    case IConfig::HugePagesKey: /* --no-huge-pages */
    case IConfig::CPUKey:       /* --no-cpu */
        // 将 key 对应的值设置为 false
        return transformBoolean(doc, key, false);
    # 当配置键为CPUAffinityKey时，解析参数中的"0x"，将其转换为无符号64位整数，否则按十进制转换为无符号64位整数
    case IConfig::CPUAffinityKey: /* --cpu-affinity */
        {
            const char *p  = strstr(arg, "0x");
            return transformUint64(doc, key, p ? strtoull(p, nullptr, 16) : strtoull(arg, nullptr, 10));
        }

    # 当配置键为CPUMaxThreadsKey时，将参数转换为十进制无符号64位整数，并设置到文档中的CpuConfig::kMaxThreadsHint字段
    case IConfig::CPUMaxThreadsKey: /* --cpu-max-threads-hint */
        return set(doc, CpuConfig::kField, CpuConfig::kMaxThreadsHint, static_cast<uint64_t>(strtol(arg, nullptr, 10)));

    # 当配置键为MemoryPoolKey时，将参数转换为十进制整数，并设置到文档中的CpuConfig::kMemoryPool字段
    case IConfig::MemoryPoolKey: /* --cpu-memory-pool */
        return set(doc, CpuConfig::kField, CpuConfig::kMemoryPool, static_cast<int64_t>(strtol(arg, nullptr, 10)));

    # 当配置键为YieldKey时，将false设置到文档中的CpuConfig::kYield字段
    case IConfig::YieldKey: /* --cpu-no-yield */
        return set(doc, CpuConfig::kField, CpuConfig::kYield, false);

    # 当配置键为PauseOnBatteryKey时，将true设置到文档中的Config::kPauseOnBattery字段
    case IConfig::PauseOnBatteryKey: /* --pause-on-battery */
        return set(doc, Config::kPauseOnBattery, true);

    # 当配置键为PauseOnActiveKey时，将参数转换为十进制无符号64位整数，并设置到文档中的Config::kPauseOnActive字段
    case IConfig::PauseOnActiveKey: /* --pause-on-active */
        return set(doc, Config::kPauseOnActive, static_cast<uint64_t>(strtol(arg, nullptr, 10)));
#   ifdef XMRIG_ALGO_ARGON2
    # 如果定义了 XMRIG_ALGO_ARGON2，则执行以下操作
    case IConfig::Argon2ImplKey: /* --argon2-impl */
        # 设置配置文档中 CPU 配置的 Argon2Impl 字段为传入的参数值
        return set(doc, CpuConfig::kField, CpuConfig::kArgon2Impl, arg);
#   endif

#   ifdef XMRIG_FEATURE_ASM
    # 如果定义了 XMRIG_FEATURE_ASM，则执行以下操作
    case IConfig::AssemblyKey: /* --asm */
        # 设置配置文档中 CPU 配置的 Asm 字段为传入的参数值
        return set(doc, CpuConfig::kField, CpuConfig::kAsm, arg);
#   endif

#   ifdef XMRIG_ALGO_RANDOMX
    # 如果定义了 XMRIG_ALGO_RANDOMX，则执行以下操作
    case IConfig::RandomXInitKey: /* --randomx-init */
        # 设置配置文档中 Rx 配置的 Init 字段为将传入的参数转换为整数类型后的值
        return set(doc, RxConfig::kField, RxConfig::kInit, static_cast<int64_t>(strtol(arg, nullptr, 10)));

#   ifdef XMRIG_FEATURE_HWLOC
    # 如果定义了 XMRIG_FEATURE_HWLOC，则执行以下操作
    case IConfig::RandomXNumaKey: /* --randomx-no-numa */
        # 设置配置文档中 Rx 配置的 NUMA 字段为 false
        return set(doc, RxConfig::kField, RxConfig::kNUMA, false);
#   endif

    case IConfig::RandomXModeKey: /* --randomx-mode */
        # 设置配置文档中 Rx 配置的 Mode 字段为传入的参数值
        return set(doc, RxConfig::kField, RxConfig::kMode, arg);

    case IConfig::RandomX1GbPagesKey: /* --randomx-1gb-pages */
        # 设置配置文档中 Rx 配置的 OneGbPages 字段为 true
        return set(doc, RxConfig::kField, RxConfig::kOneGbPages, true);

    case IConfig::RandomXWrmsrKey: /* --randomx-wrmsr */
        # 如果传入的参数为 nullptr，则设置配置文档中 Rx 配置的 Wrmsr 字段为 true，否则设置为将传入的参数转换为整数类型后的值
        if (arg == nullptr) {
            return set(doc, RxConfig::kField, RxConfig::kWrmsr, true);
        }
        return set(doc, RxConfig::kField, RxConfig::kWrmsr, static_cast<int64_t>(strtol(arg, nullptr, 10)));

    case IConfig::RandomXRdmsrKey: /* --randomx-no-rdmsr */
        # 设置配置文档中 Rx 配置的 Rdmsr 字段为 false
        return set(doc, RxConfig::kField, RxConfig::kRdmsr, false);

    case IConfig::RandomXCacheQoSKey: /* --cache-qos */
        # 设置配置文档中 Rx 配置的 CacheQoS 字段为 true
        return set(doc, RxConfig::kField, RxConfig::kCacheQoS, true);

    case IConfig::HugePagesJitKey: /* --huge-pages-jit */
        # 设置配置文档中 CPU 配置的 HugePagesJit 字段为 true
        return set(doc, CpuConfig::kField, CpuConfig::kHugePagesJit, true);
#   endif

#   ifdef XMRIG_FEATURE_OPENCL
    # 如果定义了 XMRIG_FEATURE_OPENCL，则执行以下操作
    case IConfig::OclKey: /* --opencl */
        # 将 m_opencl 设置为 true
        m_opencl = true;
        break;

    case IConfig::OclCacheKey: /* --opencl-no-cache */
        # 设置配置文档中 Ocl 配置的 cache 字段为 false
        return set(doc, Config::kOcl, "cache", false);

    case IConfig::OclLoaderKey: /* --opencl-loader */
        # 设置配置文档中 Ocl 配置的 loader 字段为传入的参数值
        return set(doc, Config::kOcl, "loader", arg);
    # 如果参数是 --opencl-devices，则设置 m_opencl 为 true，并返回设置结果
    case IConfig::OclDevicesKey: /* --opencl-devices */
        m_opencl = true;
        return set(doc, Config::kOcl, "devices-hint", arg);

    # 如果参数是 --opencl-platform
    case IConfig::OclPlatformKey: /* --opencl-platform */
        # 如果参数长度小于3，则将参数转换为整数并设置为 platform 的值
        if (strlen(arg) < 3) {
            return set(doc, Config::kOcl, "platform", static_cast<uint64_t>(strtol(arg, nullptr, 10)));
        }
        # 否则直接将参数设置为 platform 的值
        return set(doc, Config::kOcl, "platform", arg);
#   endif  // 结束条件编译指令

#   ifdef XMRIG_FEATURE_CUDA  // 如果定义了 XMRIG_FEATURE_CUDA
    case IConfig::CudaKey: /* --cuda */  // 当 key 为 CudaKey 时，设置对应的值为 true
        return set(doc, Config::kCuda, kEnabled, true);

    case IConfig::CudaLoaderKey: /* --cuda-loader */  // 当 key 为 CudaLoaderKey 时，设置对应的值为 arg
        return set(doc, Config::kCuda, "loader", arg);

    case IConfig::CudaDevicesKey: /* --cuda-devices */  // 当 key 为 CudaDevicesKey 时，设置对应的值为 true，并设置设备提示
        set(doc, Config::kCuda, kEnabled, true);
        return set(doc, Config::kCuda, "devices-hint", arg);

    case IConfig::CudaBFactorKey: /* --cuda-bfactor-hint */  // 当 key 为 CudaBFactorKey 时，设置对应的值为 arg 转换为无符号整数
        return set(doc, Config::kCuda, "bfactor-hint", static_cast<uint64_t>(strtol(arg, nullptr, 10)));

    case IConfig::CudaBSleepKey: /* --cuda-bsleep-hint */  // 当 key 为 CudaBSleepKey 时，设置对应的值为 arg 转换为无符号整数
        return set(doc, Config::kCuda, "bsleep-hint", static_cast<uint64_t>(strtol(arg, nullptr, 10)));
#   endif  // 结束条件编译指令

#   ifdef XMRIG_FEATURE_NVML  // 如果定义了 XMRIG_FEATURE_NVML
    case IConfig::NvmlKey: /* --no-nvml */  // 当 key 为 NvmlKey 时，设置对应的值为 false
        return set(doc, Config::kCuda, "nvml", false);
#   endif  // 结束条件编译指令

#   if defined(XMRIG_FEATURE_NVML) || defined (XMRIG_FEATURE_ADL)  // 如果定义了 XMRIG_FEATURE_NVML 或 XMRIG_FEATURE_ADL
    case IConfig::HealthPrintTimeKey: /* --health-print-time */  // 当 key 为 HealthPrintTimeKey 时，设置对应的值为 arg 转换为无符号整数
        return set(doc, Config::kHealthPrintTime, static_cast<uint64_t>(strtol(arg, nullptr, 10)));
#   endif  // 结束条件编译指令

#   ifdef XMRIG_FEATURE_DMI  // 如果定义了 XMRIG_FEATURE_DMI
    case IConfig::DmiKey: /* --no-dmi */  // 当 key 为 DmiKey 时，设置对应的值为 false
        return set(doc, Config::kDMI, false);
#   endif  // 结束条件编译指令

#   ifdef XMRIG_FEATURE_BENCHMARK  // 如果定义了 XMRIG_FEATURE_BENCHMARK
    case IConfig::AlgorithmKey:     /* --algo */  // 当 key 为 AlgorithmKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::BenchKey:         /* --bench */  // 当 key 为 BenchKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::StressKey:        /* --stress */  // 当 key 为 StressKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::BenchSubmitKey:   /* --submit */  // 当 key 为 BenchSubmitKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::BenchVerifyKey:   /* --verify */  // 当 key 为 BenchVerifyKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::BenchTokenKey:    /* --token */  // 当 key 为 BenchTokenKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::BenchSeedKey:     /* --seed */  // 当 key 为 BenchSeedKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::BenchHashKey:     /* --hash */  // 当 key 为 BenchHashKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::UserKey:          /* --user */  // 当 key 为 UserKey 时，调用 transformBenchmark 函数进行转换
    case IConfig::RotationKey:      /* --rotation */  // 当 key 为 RotationKey 时，调用 transformBenchmark 函数进行转换
        return transformBenchmark(doc, key, arg);
#   endif  // 结束条件编译指令

    default:  // 默认情况
        break;  // 什么也不做
    }
}


void xmrig::ConfigTransform::transformBoolean(rapidjson::Document &doc, int key, bool enable)  // 转换布尔值的函数
    # 根据不同的 key 值进行不同的操作
    switch (key) {
    # 如果 key 值为 HugePagesKey，则设置 HugePages 的配置
    case IConfig::HugePagesKey: /* --no-huge-pages */
        return set(doc, CpuConfig::kField, CpuConfig::kHugePages, enable);

    # 如果 key 值为 CPUKey，则设置 CPU 的配置
    case IConfig::CPUKey:       /* --no-cpu */
        return set(doc, CpuConfig::kField, kEnabled, enable);

    # 如果 key 值不匹配任何已知的情况，则执行默认操作
    default:
        break;
    }
// 定义一个函数，用于将 uint64_t 类型的参数转换为 rapidjson::Document 对象
void xmrig::ConfigTransform::transformUint64(rapidjson::Document &doc, int key, uint64_t arg)
{
    // 使用 rapidjson 命名空间
    using namespace rapidjson;

    // 根据 key 的值进行不同的操作
    switch (key) {
    case IConfig::CPUAffinityKey: /* --cpu-affinity */
        // 将 arg 转换为 int64_t 类型，并赋值给 m_affinity
        m_affinity = static_cast<int64_t>(arg);
        break;

    case IConfig::ThreadsKey: /* --threads */
        // 将 arg 赋值给 m_threads
        m_threads = arg;
        break;

    case IConfig::AVKey: /* --av */
        // 根据 arg 的值设置 m_intensity，并根据 isHwAes(arg) 的返回值设置 doc 中的字段
        m_intensity = intensity(arg);
        set(doc, CpuConfig::kField, CpuConfig::kHwAes, isHwAes(arg));
        break;

    case IConfig::CPUPriorityKey: /* --cpu-priority */
        // 设置 doc 中 CpuConfig::kField 的 CpuConfig::kPriority 字段为 arg
        return set(doc, CpuConfig::kField, CpuConfig::kPriority, arg);

    case IConfig::HugePageSizeKey: /* --hugepage-size */
        // 设置 doc 中 CpuConfig::kField 的 CpuConfig::kHugePages 字段为 arg
        return set(doc, CpuConfig::kField, CpuConfig::kHugePages, arg);

    default:
        break;
    }
}

// 如果定义了 XMRIG_FEATURE_BENCHMARK，则定义一个函数，用于将 char* 类型的参数转换为 rapidjson::Document 对象
void xmrig::ConfigTransform::transformBenchmark(rapidjson::Document &doc, int key, const char *arg)
{
    // 根据 key 的值进行不同的操作
    switch (key) {
    case IConfig::AlgorithmKey: /* --algo */
        // 设置 doc 中 BenchConfig::kBenchmark 的 BenchConfig::kAlgo 字段为 arg
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kAlgo, arg);

    case IConfig::BenchKey: /* --bench */
    {
        // 为基准测试设置 CPU 参数
        set(doc, CpuConfig::kField, CpuConfig::kHugePagesJit, true);
        set(doc, CpuConfig::kField, CpuConfig::kPriority, 2);
        set(doc, CpuConfig::kField, CpuConfig::kYield, false);
        // 设置 doc 中 BenchConfig::kBenchmark 的 BenchConfig::kSize 字段为 arg
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kSize, arg);
    }

    case IConfig::StressKey: /* --stress */
        // 向 doc 中 Pools::kPools 的 Pool::kUser 字段添加 BenchConfig::kBenchmark
        return add(doc, Pools::kPools, Pool::kUser, BenchConfig::kBenchmark);

    case IConfig::BenchSubmitKey: /* --submit */
        // 设置 doc 中 BenchConfig::kBenchmark 的 BenchConfig::kSubmit 字段为 true
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kSubmit, true);

    case IConfig::BenchVerifyKey: /* --verify */
        // 设置 doc 中 BenchConfig::kBenchmark 的 BenchConfig::kVerify 字段为 arg
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kVerify, arg);

    case IConfig::BenchTokenKey: /* --token */
        // 设置 doc 中 BenchConfig::kBenchmark 的 BenchConfig::kToken 字段为 arg
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kToken, arg);
    # 如果参数是 BenchSeedKey，则设置对应的配置项
    case IConfig::BenchSeedKey: /* --seed */
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kSeed, arg);

    # 如果参数是 BenchHashKey，则设置对应的配置项
    case IConfig::BenchHashKey: /* --hash */
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kHash, arg);

    # 如果参数是 UserKey，则设置对应的配置项
    case IConfig::UserKey: /* --user */
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kUser, arg);

    # 如果参数是 RotationKey，则设置对应的配置项
    case IConfig::RotationKey: /* --rotation */
        return set(doc, BenchConfig::kBenchmark, BenchConfig::kRotation, arg);

    # 如果参数不匹配以上任何一种情况，则不做任何操作
    default:
        break;
    }
# 结束一个条件编译的代码块
#endif
```