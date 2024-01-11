# `xmrig\src\crypto\rx\Rx.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   其版本由自由软件基金会发布，可以选择遵循版本 3 或者（自行选择）之后的版本。
 *
 *   本程序是希望它有用，但没有任何担保；甚至没有适用于特定目的的隐含担保。
 *   更多详情请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本。
 *   如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "crypto/rx/Rx.h"
#include "backend/cpu/Cpu.h"
#include "backend/cpu/CpuConfig.h"
#include "backend/cpu/CpuThreads.h"
#include "crypto/rx/RxConfig.h"
#include "crypto/rx/RxQueue.h"
#include "crypto/randomx/randomx.h"
#include "crypto/randomx/aes_hash.hpp"

#ifdef XMRIG_FEATURE_MSR
#   include "crypto/rx/RxFix.h"
#   include "crypto/rx/RxMsr.h"
#endif

// 命名空间 xmrig
namespace xmrig {

// RxPrivate 类的前向声明
class RxPrivate;

// 初始化操作系统标志，默认为 false
static bool osInitialized   = false;
// RxPrivate 类的指针，默认为 nullptr
static RxPrivate *d_ptr     = nullptr;

// RxPrivate 类的定义
class RxPrivate
{
public:
    // 构造函数，接受一个 IRxListener 指针作为参数
    inline explicit RxPrivate(IRxListener *listener) : queue(listener) {}

    // RxQueue 对象
    RxQueue queue;
};

} // namespace xmrig

// 返回 HugePagesInfo 对象
xmrig::HugePagesInfo xmrig::Rx::hugePages()
{
    return d_ptr->queue.hugePages();
}

// 返回 RxDataset 指针
xmrig::RxDataset *xmrig::Rx::dataset(const Job &job, uint32_t nodeId)
{
    return d_ptr->queue.dataset(job, nodeId);
}

// 销毁 Rx 对象
void xmrig::Rx::destroy()
{
#   ifdef XMRIG_FEATURE_MSR
    RxMsr::destroy();
#   endif

    delete d_ptr;

    d_ptr = nullptr;
}

// 初始化 Rx 对象
void xmrig::Rx::init(IRxListener *listener)
{
    d_ptr = new RxPrivate(listener);
}

// 包含 blake2.h 文件
#include "crypto/randomx/blake2/blake2.h"
// 如果定义了XMRIG_FEATURE_AVX2，则包含AVX2版本的blake2b.h头文件
#if defined(XMRIG_FEATURE_AVX2)
#include "crypto/randomx/blake2/avx2/blake2b.h"
#endif

// 定义函数指针rx_blake2b_compress，指向rx_blake2b_compress_integer函数
void (*rx_blake2b_compress)(blake2b_state* S, const uint8_t * block) = rx_blake2b_compress_integer;
// 定义函数指针rx_blake2b，指向rx_blake2b_default函数
int (*rx_blake2b)(void* out, size_t outlen, const void* in, size_t inlen) = rx_blake2b_default;

// Rx类的init模板函数，接受seed、config和cpu作为参数
template<typename T>
bool xmrig::Rx::init(const T &seed, const RxConfig &config, const CpuConfig &cpu)
{
    // 获取seed的算法家族
    const auto f = seed.algorithm().family();
    // 如果算法家族不是RANDOM_X，并且定义了XMRIG_ALGO_CN_HEAVY或XMRIG_ALGO_GHOSTRIDER，则执行以下代码块
    if ((f != Algorithm::RANDOM_X)
#       ifdef XMRIG_ALGO_CN_HEAVY
        && (f != Algorithm::CN_HEAVY)
#       endif
#       ifdef XMRIG_ALGO_GHOSTRIDER
        && (f != Algorithm::GHOSTRIDER)
#       endif
        ) {
#       ifdef XMRIG_FEATURE_MSR
        // 销毁RxMsr对象
        RxMsr::destroy();
#       endif
        // 返回true
        return true;
    }

#   ifdef XMRIG_FEATURE_MSR
    // 如果RxMsr对象未初始化，则初始化RxMsr对象
    if (!RxMsr::isInitialized()) {
        RxMsr::init(config, cpu.threads().get(seed.algorithm()).data());
    }
#   endif

#   ifdef XMRIG_ALGO_CN_HEAVY
    // 如果算法家族是CN_HEAVY，则返回true
    if (f == Algorithm::CN_HEAVY) {
        return true;
    }
#   endif

#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果算法家族是GHOSTRIDER，则返回true
    if (f == Algorithm::GHOSTRIDER) {
        return true;
    }
#   endif

    // 设置随机X的内存预取模式
    randomx_set_scratchpad_prefetch_mode(config.scratchpadPrefetchMode());
    // 设置是否使用大页内存和JIT编译
    randomx_set_huge_pages_jit(cpu.isHugePagesJit());
    // 设置是否使用AVX2初始化数据集
    randomx_set_optimized_dataset_init(config.initDatasetAVX2());

    // 如果操作系统未初始化，则执行以下代码块
    if (!osInitialized) {
#       ifdef XMRIG_FIX_RYZEN
        // 设置RxFix的主循环异常帧
        RxFix::setupMainLoopExceptionFrame();
#       endif
        // 如果CPU不支持硬件AES，则选择软件AES实现
        if (!cpu.isHwAES()) {
            SelectSoftAESImpl(cpu.threads().get(seed.algorithm()).count());
        }
        // 如果CPU支持SSE4.1，则使用SSE4.1版本的blake2b_compress函数
#       if defined(XMRIG_FEATURE_SSE4_1)
        if (Cpu::info()->has(ICpuInfo::FLAG_SSE41)) {
            rx_blake2b_compress = rx_blake2b_compress_sse41;
        }
#       endif
        // 如果定义了XMRIG_FEATURE_AVX2并且CPU支持AVX2，则使用AVX2版本的blake2b函数
#if     defined(XMRIG_FEATURE_AVX2)
        if (Cpu::info()->has(ICpuInfo::FLAG_AVX2)) {
            rx_blake2b = blake2b_avx2;
        }
#       endif
        // 设置操作系统已初始化
        osInitialized = true;
    }

    // 如果准备就绪，则返回true
    if (isReady(seed)) {
        return true;
    }
}
    # 使用指针d_ptr调用queue对象的enqueue方法，将seed、config.nodeset()、config.threads(cpu.limit())、cpu.isHugePages()、config.isOneGbPages()、config.mode()、cpu.priority()作为参数传入
    d_ptr->queue.enqueue(seed, config.nodeset(), config.threads(cpu.limit()), cpu.isHugePages(), config.isOneGbPages(), config.mode(), cpu.priority());
    
    # 返回布尔值false
    return false;
// 结束命名空间 xmrig
}

// 检查 Rx 对象是否准备就绪
template<typename T>
bool xmrig::Rx::isReady(const T &seed)
{
    return d_ptr->queue.isReady(seed);
}

// 如果启用了 MSR 特性，则返回 true
#ifdef XMRIG_FEATURE_MSR
bool xmrig::Rx::isMSR()
{
    return RxMsr::isEnabled();
}
#endif

// 命名空间 xmrig
namespace xmrig {

// 实例化 Rx::init 方法，用于 RxSeed、RxConfig 和 CpuConfig 类型
template bool Rx::init(const RxSeed &seed, const RxConfig &config, const CpuConfig &cpu);
template bool Rx::isReady(const RxSeed &seed);
template bool Rx::init(const Job &seed, const RxConfig &config, const CpuConfig &cpu);
template bool Rx::isReady(const Job &seed);

} // 命名空间 xmrig
```