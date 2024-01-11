# `xmrig\src\backend\cpu\CpuWorker.cpp`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有暗示的保证适用于特定目的。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include <cassert>
#include <thread>
#include <mutex>


#include "backend/cpu/Cpu.h"
#include "backend/cpu/CpuWorker.h"
#include "base/tools/Alignment.h"
#include "base/tools/Chrono.h"
#include "core/config/Config.h"
#include "core/Miner.h"
#include "crypto/cn/CnCtx.h"
#include "crypto/cn/CryptoNight_test.h"
#include "crypto/cn/CryptoNight.h"
#include "crypto/common/Nonce.h"
#include "crypto/common/VirtualMemory.h"
#include "crypto/rx/Rx.h"
#include "crypto/rx/RxCache.h"
#include "crypto/rx/RxDataset.h"
#include "crypto/rx/RxVm.h"
#include "crypto/ghostrider/ghostrider.h"
#include "net/JobResults.h"


#ifdef XMRIG_ALGO_RANDOMX
#   include "crypto/randomx/randomx.h"
#endif


#ifdef XMRIG_FEATURE_BENCHMARK
#   include "backend/common/benchmark/BenchState.h"
#endif


namespace xmrig {

static constexpr uint32_t kReserveCount = 32768;


#ifdef XMRIG_ALGO_CN_HEAVY
static std::mutex cn_heavyZen3MemoryMutex;
VirtualMemory* cn_heavyZen3Memory = nullptr;
#endif

} // namespace xmrig



template<size_t N>
xmrig::CpuWorker<N>::CpuWorker(size_t id, const CpuLaunchData &data) :
    Worker(id, data.affinity, data.priority),
    m_algorithm(data.algorithm),
    # 调用 m_assembly 方法，并传入 data.assembly 参数
    m_assembly(data.assembly),
    # 调用 m_hwAES 方法，并传入 data.hwAES 参数
    m_hwAES(data.hwAES),
    # 调用 m_yield 方法，并传入 data.yield 参数
    m_yield(data.yield),
    # 调用 m_av 方法，并传入 data.av() 的返回值作为参数
    m_av(data.av()),
    # 调用 m_miner 方法，并传入 data.miner 参数
    m_miner(data.miner),
    # 调用 m_threads 方法，并传入 data.threads 参数
    m_threads(data.threads),
    # 调用 m_ctx 方法
    m_ctx()
{
#   ifdef XMRIG_ALGO_CN_HEAVY
    // 如果定义了 XMRIG_ALGO_CN_HEAVY，则进行以下操作
    // 获取 CPU 架构和型号信息
    const auto arch = Cpu::info()->arch();
    const uint32_t model = Cpu::info()->model();
    // 判断是否为 Zen3 架构的 Vermeer 或 Zen4 架构的 Raphael
    const bool is_vermeer = (arch == ICpuInfo::ARCH_ZEN3) && (model == 0x21);
    const bool is_raphael = (arch == ICpuInfo::ARCH_ZEN4) && (model == 0x61);
    // 如果满足条件，则进行以下操作
    if ((N == 1) && (m_av == CnHash::AV_SINGLE) && (m_algorithm.family() == Algorithm::CN_HEAVY) && (m_assembly != Assembly::NONE) && (is_vermeer || is_raphael)) {
        // 加锁，确保线程安全
        std::lock_guard<std::mutex> lock(cn_heavyZen3MemoryMutex);
        // 如果内存为空，则进行以下操作
        if (!cn_heavyZen3Memory) {
            // 将线程数向上取整到 8 的倍数
            const size_t num_threads = ((m_threads + 7) / 8) * 8;
            // 分配内存
            cn_heavyZen3Memory = new VirtualMemory(m_algorithm.l3() * num_threads, data.hugePages, false, false, node());
        }
        // 设置内存
        m_memory = cn_heavyZen3Memory;
    }
    // 如果不满足条件，则进行以下操作
    else
#   endif
    {
        // 分配内存
        m_memory = new VirtualMemory(m_algorithm.l3() * N, data.hugePages, false, true, node());
    }

#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果定义了 XMRIG_ALGO_GHOSTRIDER，则创建辅助线程
    m_ghHelper = ghostrider::create_helper_thread(affinity(), data.priority, data.affinities);
#   endif
}


template<size_t N>
xmrig::CpuWorker<N>::~CpuWorker()
{
#   ifdef XMRIG_ALGO_RANDOMX
    // 如果定义了 XMRIG_ALGO_RANDOMX，则销毁虚拟机
    RxVm::destroy(m_vm);
#   endif

    // 释放上下文
    CnCtx::release(m_ctx, N);

#   ifdef XMRIG_ALGO_CN_HEAVY
    // 如果定义了 XMRIG_ALGO_CN_HEAVY，则进行以下操作
    if (m_memory != cn_heavyZen3Memory)
#   endif
    {
        // 释放内存
        delete m_memory;
    }

#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果定义了 XMRIG_ALGO_GHOSTRIDER，则销毁辅助线程
    ghostrider::destroy_helper_thread(m_ghHelper);
#   endif
}


#ifdef XMRIG_ALGO_RANDOMX
template<size_t N>
void xmrig::CpuWorker<N>::allocateRandomX_VM()
{
    // 获取 RandomX 数据集
    RxDataset *dataset = Rx::dataset(m_job.currentJob(), node());

    // 如果数据集为空，则进行以下操作
    while (dataset == nullptr) {
        // 线程休眠 20 毫秒
        std::this_thread::sleep_for(std::chrono::milliseconds(20));

        // 如果 CPU 随机数序列为 0，则返回
        if (Nonce::sequence(Nonce::CPU) == 0) {
            return;
        }

        // 重新获取 RandomX 数据集
        dataset = Rx::dataset(m_job.currentJob(), node());
    }
    # 如果虚拟机对象为空
    if (!m_vm) {
        # 如果内存对象支持大页，尝试从数据集的 1GB 大页中分配临时空间，如果普通大页不可用
        uint8_t* scratchpad = m_memory->isHugePages() ? m_memory->scratchpad() : dataset->tryAllocateScrathpad();
        # 创建 RandomX 虚拟机对象，使用数据集的临时空间或者内存对象的临时空间，根据是否支持硬件 AES 加速、汇编类型和节点信息
        m_vm = RxVm::create(dataset, scratchpad ? scratchpad : m_memory->scratchpad(), !m_hwAES, m_assembly, node());
    }
    # 如果数据集为空并且当前作业的种子不等于上次的种子
    else if (!dataset->get() && (m_job.currentJob().seed() != m_seed)) {
        # 使用数据集的缓存更新 RandomX 轻量级虚拟机
        randomx_vm_set_cache(m_vm, dataset->cache()->get());
    }
    # 更新当前种子
    m_seed = m_job.currentJob().seed();
}
#endif


template<size_t N>
bool xmrig::CpuWorker<N>::selfTest()
{
#   ifdef XMRIG_ALGO_RANDOMX
    // 如果算法家族是 RANDOM_X，则返回 N 是否等于 1
    if (m_algorithm.family() == Algorithm::RANDOM_X) {
        return N == 1;
    }
#   endif

    // 分配 CN 上下文
    allocateCnCtx();

#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果算法家族是 GHOSTRIDER，则返回 (N == 8) && verify(Algorithm::GHOSTRIDER_RTM, test_output_gr)
    if (m_algorithm.family() == Algorithm::GHOSTRIDER) {
        return (N == 8) && verify(Algorithm::GHOSTRIDER_RTM, test_output_gr);
    }
#   endif

    // 如果算法家族是 CN，则进行一系列的验证，并返回结果
    if (m_algorithm.family() == Algorithm::CN) {
        const bool rc = verify(Algorithm::CN_0,      test_output_v0)   &&
                        verify(Algorithm::CN_1,      test_output_v1)   &&
                        verify(Algorithm::CN_2,      test_output_v2)   &&
                        verify(Algorithm::CN_FAST,   test_output_msr)  &&
                        verify(Algorithm::CN_XAO,    test_output_xao)  &&
                        verify(Algorithm::CN_RTO,    test_output_rto)  &&
                        verify(Algorithm::CN_HALF,   test_output_half) &&
                        verify2(Algorithm::CN_R,     test_output_r)    &&
                        verify(Algorithm::CN_RWZ,    test_output_rwz)  &&
                        verify(Algorithm::CN_ZLS,    test_output_zls)  &&
                        verify(Algorithm::CN_CCX,    test_output_ccx)  &&
                        verify(Algorithm::CN_DOUBLE, test_output_double);

        return rc;
    }

#   ifdef XMRIG_ALGO_CN_LITE
    // 如果算法家族是 CN_LITE，则返回 verify(Algorithm::CN_LITE_0, test_output_v0_lite) && verify(Algorithm::CN_LITE_1, test_output_v1_lite)
    if (m_algorithm.family() == Algorithm::CN_LITE) {
        return verify(Algorithm::CN_LITE_0,    test_output_v0_lite) &&
               verify(Algorithm::CN_LITE_1,    test_output_v1_lite);
    }
#   endif

#   ifdef XMRIG_ALGO_CN_HEAVY
    // 如果算法家族是 CN_HEAVY，则返回 verify(Algorithm::CN_HEAVY_0, test_output_v0_heavy) && verify(Algorithm::CN_HEAVY_XHV, test_output_xhv_heavy) && verify(Algorithm::CN_HEAVY_TUBE, test_output_tube_heavy)
    if (m_algorithm.family() == Algorithm::CN_HEAVY) {
        return verify(Algorithm::CN_HEAVY_0,    test_output_v0_heavy)  &&
               verify(Algorithm::CN_HEAVY_XHV,  test_output_xhv_heavy) &&
               verify(Algorithm::CN_HEAVY_TUBE, test_output_tube_heavy);
    }
#   endif

#   ifdef XMRIG_ALGO_CN_PICO
    # 如果算法家族是CN_PICO，则执行以下操作
    if (m_algorithm.family() == Algorithm::CN_PICO) {
        # 返回CN_PICO_0和CN_PICO_TLO的验证结果
        return verify(Algorithm::CN_PICO_0, test_output_pico_trtl) &&
               verify(Algorithm::CN_PICO_TLO, test_output_pico_tlo);
    }
// 结束条件判断
#   endif

// 如果算法族为 CN_FEMTO，则返回 CN_UPX2 算法的验证结果
#   ifdef XMRIG_ALGO_CN_FEMTO
    if (m_algorithm.family() == Algorithm::CN_FEMTO) {
        return verify(Algorithm::CN_UPX2, test_output_femto_upx2);
    }
#   endif

// 如果算法族为 ARGON2，则返回 AR2_CHUKWA、AR2_CHUKWA_V2、AR2_WRKZ 算法的验证结果
#   ifdef XMRIG_ALGO_ARGON2
    if (m_algorithm.family() == Algorithm::ARGON2) {
        return verify(Algorithm::AR2_CHUKWA, argon2_chukwa_test_out) &&
               verify(Algorithm::AR2_CHUKWA_V2, argon2_chukwa_v2_test_out) &&
               verify(Algorithm::AR2_WRKZ, argon2_wrkz_test_out);
    }
#   endif

// 默认返回 false
    return false;
}


// 获取哈希率数据
template<size_t N>
void xmrig::CpuWorker<N>::hashrateData(uint64_t &hashCount, uint64_t &, uint64_t &rawHashes) const
{
    // 将 m_count 赋值给 hashCount 和 rawHashes
    hashCount = m_count;
    rawHashes = m_count;
}


// 开始执行 CPU 工作
template<size_t N>
void xmrig::CpuWorker<N>::start()
{
    // 当 CPU Nonce 序列大于 0 时执行循环
    while (Nonce::sequence(Nonce::CPU) > 0) {
        // 如果 Nonce 被暂停
        if (Nonce::isPaused()) {
            // 执行循环，直到 Nonce 恢复并且 CPU Nonce 序列大于 0
            do {
                std::this_thread::sleep_for(std::chrono::milliseconds(20));
            }
            while (Nonce::isPaused() && Nonce::sequence(Nonce::CPU) > 0);

            // 如果 CPU Nonce 序列为 0，则跳出循环
            if (Nonce::sequence(Nonce::CPU) == 0) {
                break;
            }

            // 执行任务消耗
            consumeJob();
        }

//       如果定义了 XMRIG_ALGO_RANDOMX
#       ifdef XMRIG_ALGO_RANDOMX
        // 定义变量 first 为 true，声明一个 16 字节对齐的 uint64_t 数组 tempHash
        bool first = true;
        alignas(16) uint64_t tempHash[8] = {};
#       endif

        // 当 CPU Nonce 不过时且当前作业的 L3 等级与算法的 L3 等级相同时执行循环
        while (!Nonce::isOutdated(Nonce::CPU, m_job.sequence())) {
            // 获取当前作业
            const Job &job = m_job.currentJob();

            // 如果当前作业的 L3 等级与算法的 L3 等级不相同时跳出循环
            if (job.algorithm().l3() != m_algorithm.l3()) {
                break;
            }

            // 声明一个大小为 N 的 current_job_nonces 数组，读取当前作业的 Nonce
            uint32_t current_job_nonces[N];
            for (size_t i = 0; i < N; ++i) {
                current_job_nonces[i] = readUnaligned(m_job.nonce(i));
            }
# 如果定义了 XMRIG_FEATURE_BENCHMARK
            if (m_benchSize) {
                # 如果当前作业的 nonce 大于等于基准测试大小，则返回基准测试完成状态
                if (current_job_nonces[0] >= m_benchSize) {
                    return BenchState::done();
                }

                # 在单线程基准测试中，使每个哈希依赖于前一个哈希，以防止使用多个线程作弊
                if (m_threads == 1) {
                    *(uint64_t*)(m_job.blob()) ^= BenchState::data();
                }
            }
#           endif

            # 初始化变量 valid 为 true
            bool valid = true;

            # 定义长度为 64 的 miner_signature_saved 数组

#           ifdef XMRIG_ALGO_RANDOMX
            # 定义指向 m_job.blob() + m_job.nonceOffset() + m_job.nonceSize() 的指针 miner_signature_ptr
            if (job.algorithm().family() == Algorithm::RANDOM_X) {
                # 如果是第一次计算哈希
                if (first) {
                    first = false;
                    # 如果作业有矿工签名，则生成矿工签名
                    if (job.hasMinerSignature()) {
                        job.generateMinerSignature(m_job.blob(), job.size(), miner_signature_ptr);
                    }
                    # 计算第一个随机X哈希
                    randomx_calculate_hash_first(m_vm, tempHash, m_job.blob(), job.size());
                }

                # 如果不是下一轮，则跳出循环
                if (!nextRound()) {
                    break;
                }

                # 如果作业有矿工签名，则复制 miner_signature_ptr 到 miner_signature_saved
                if (job.hasMinerSignature()) {
                    memcpy(miner_signature_saved, miner_signature_ptr, sizeof(miner_signature_saved));
                    job.generateMinerSignature(m_job.blob(), job.size(), miner_signature_ptr);
                }
                # 计算下一个随机X哈希
                randomx_calculate_hash_next(m_vm, tempHash, m_job.blob(), job.size(), m_hash);
            }
            else
#           endif
            {
                # 根据作业算法家族进行不同的处理
// 如果定义了XMRIG_ALGO_GHOSTRIDER，则执行以下代码块
#ifdef XMRIG_ALGO_GHOSTRIDER
    // 根据算法类型执行不同的操作
    case Algorithm::GHOSTRIDER:
        // 如果N等于8，则调用ghostrider命名空间下的hash_octa函数
        if (N == 8) {
            ghostrider::hash_octa(m_job.blob(), job.size(), m_hash, m_ctx, m_ghHelper);
        }
        // 如果N不等于8，则将valid标记为false
        else {
            valid = false;
        }
        break;
#endif

// 如果不是以上定义的算法类型，则执行以下代码块
default:
    // 根据算法类型执行不同的操作
    fn(job.algorithm())(m_job.blob(), job.size(), m_hash, m_ctx, job.height());
    break;
}

// 如果不需要进行下一轮计算，则跳出循环
if (!nextRound()) {
    break;
};

// 如果valid为true，则执行以下代码块
if (valid) {
    // 遍历N次，每次取出m_hash中的特定位置的值
    for (size_t i = 0; i < N; ++i) {
        const uint64_t value = *reinterpret_cast<uint64_t*>(m_hash + (i * 32) + 24);

        // 如果定义了XMRIG_FEATURE_BENCHMARK，则执行以下代码块
#ifdef XMRIG_FEATURE_BENCHMARK
        if (m_benchSize) {
            // 如果当前作业的nonce小于m_benchSize，则调用BenchState::add函数
            if (current_job_nonces[i] < m_benchSize) {
                BenchState::add(value);
            }
        }
        else
#endif
        // 如果value小于作业的目标值，则调用JobResults::submit函数
        if (value < job.target()) {
            JobResults::submit(job, current_job_nonces[i], m_hash + (i * 32), job.hasMinerSignature() ? miner_signature_saved : nullptr);
        }
    }
    // 将计数器m_count增加N
    m_count += N;
}

// 如果m_yield为true，则让出当前线程的时间片
if (m_yield) {
    std::this_thread::yield();
}

// 执行完当前作业后，调用consumeJob函数
consumeJob();
}

// 模板函数CpuWorker的成员函数nextRound的实现
template<size_t N>
bool xmrig::CpuWorker<N>::nextRound()
{
    // 如果定义了XMRIG_FEATURE_BENCHMARK，则count为1，否则为kReserveCount
#ifdef XMRIG_FEATURE_BENCHMARK
    const uint32_t count = m_benchSize ? 1U : kReserveCount;
#else
    constexpr uint32_t count = kReserveCount;
#endif

    // 如果无法获取下一轮作业，则调用JobResults::done函数，并返回false
    if (!m_job.nextRound(count, 1)) {
        JobResults::done(m_job.currentJob());
        return false;
    }

    // 成功获取下一轮作业，则返回true
    return true;
}
// 验证函数，用于验证计算结果是否正确
bool xmrig::CpuWorker<N>::verify(const Algorithm &algorithm, const uint8_t *referenceValue)
{
#   ifdef XMRIG_ALGO_GHOSTRIDER
    // 如果算法为GHOSTRIDER_RTM，则进行以下操作
    if (algorithm == Algorithm::GHOSTRIDER_RTM) {
        // 初始化一个大小为N*80的blob数组，并进行赋值操作
        uint8_t blob[N * 80] = {};
        for (size_t i = 0; i < N; ++i) {
            blob[i * 80 + 0] = static_cast<uint8_t>(i);
            blob[i * 80 + 4] = 0x10;
            blob[i * 80 + 5] = 0x02;
        }

        // 初始化一个大小为N*32的hash1数组，并调用ghostrider::hash_octa函数进行计算
        uint8_t hash1[N * 32] = {};
        ghostrider::hash_octa(blob, 80, hash1, m_ctx, 0, false);

        // 对blob数组进行重新赋值操作
        for (size_t i = 0; i < N; ++i) {
            blob[i * 80 + 0] = static_cast<uint8_t>(i);
            blob[i * 80 + 4] = 0x43;
            blob[i * 80 + 5] = 0x05;
        }

        // 初始化一个大小为N*32的hash2数组，并再次调用ghostrider::hash_octa函数进行计算
        uint8_t hash2[N * 32] = {};
        ghostrider::hash_octa(blob, 80, hash2, m_ctx, 0, false);

        // 比较hash1和hash2数组的每个元素与referenceValue数组的对应元素，如果不相等则返回false
        for (size_t i = 0; i < N * 32; ++i) {
            if ((hash1[i] ^ hash2[i]) != referenceValue[i]) {
                return false;
            }
        }

        // 如果比较通过，则返回true
        return true;
    }
#   endif

    // 如果不是GHOSTRIDER_RTM算法，则调用fn函数获取对应算法的哈希函数
    cn_hash_fun func = fn(algorithm);
    // 如果获取的哈希函数为空，则返回false
    if (!func) {
        return false;
    }

    // 调用获取的哈希函数计算结果，并与referenceValue进行比较，返回比较结果
    func(test_input, 76, m_hash, m_ctx, 0);
    return memcmp(m_hash, referenceValue, sizeof m_hash) == 0;
}


// 验证函数2，用于验证计算结果是否正确
template<size_t N>
bool xmrig::CpuWorker<N>::verify2(const Algorithm &algorithm, const uint8_t *referenceValue)
{
    // 调用fn函数获取对应算法的哈希函数
    cn_hash_fun func = fn(algorithm);
    // 如果获取的哈希函数为空，则返回false
    if (!func) {
        return false;
    }

    // 遍历cn_r_test_input数组，对每个元素进行处理
    for (size_t i = 0; i < (sizeof(cn_r_test_input) / sizeof(cn_r_test_input[0])); ++i) {
        // 获取当前元素的size值
        const size_t size = cn_r_test_input[i].size;
        // 遍历N个处理单元，将数据拷贝到m_job.blob()中
        for (size_t k = 0; k < N; ++k) {
            memcpy(m_job.blob() + (k * size), cn_r_test_input[i].data, size);
        }

        // 调用获取的哈希函数计算结果，并与referenceValue进行比较，返回比较结果
        func(m_job.blob(), size, m_hash, m_ctx, cn_r_test_input[i].height);

        // 遍历N个处理单元，比较计算结果与referenceValue的对应部分，如果不相等则返回false
        for (size_t k = 0; k < N; ++k) {
            if (memcmp(m_hash + k * 32, referenceValue + i * 32, sizeof m_hash / N) != 0) {
                return false;
            }
        }
    }

    // 如果比较通过，则返回true
    return true;
}


namespace xmrig {

template<>
// 验证函数，用于验证算法和参考值是否匹配
bool CpuWorker<1>::verify2(const Algorithm &algorithm, const uint8_t *referenceValue)
{
    // 获取算法对应的哈希函数
    cn_hash_fun func = fn(algorithm);
    // 如果哈希函数不存在，则返回false
    if (!func) {
        return false;
    }

    // 遍历测试输入数组
    for (size_t i = 0; i < (sizeof(cn_r_test_input) / sizeof(cn_r_test_input[0])); ++i) {
        // 使用哈希函数计算哈希值
        func(cn_r_test_input[i].data, cn_r_test_input[i].size, m_hash, m_ctx, cn_r_test_input[i].height);

        // 比较计算得到的哈希值和参考值，如果不相等则返回false
        if (memcmp(m_hash, referenceValue + i * 32, sizeof m_hash) != 0) {
            return false;
        }
    }

    // 如果所有测试通过，则返回true
    return true;
}

// 分配CnCtx的函数
template<size_t N>
void xmrig::CpuWorker<N>::allocateCnCtx()
{
    // 如果m_ctx的第一个元素为空指针
    if (m_ctx[0] == nullptr) {
        int shift = 0;

#       ifdef XMRIG_ALGO_CN_HEAVY
        // 对于Zen3 CPU的cn-heavy优化
        if (m_memory == cn_heavyZen3Memory) {
            shift = (id() / 8) * m_algorithm.l3() * 8 + (id() % 8) * 64;
        }
#       endif

        // 创建CnCtx对象
        CnCtx::create(m_ctx, m_memory->scratchpad() + shift, m_algorithm.l3(), N);
    }
}

// 处理工作任务的函数
template<size_t N>
void xmrig::CpuWorker<N>::consumeJob()
{
    // 如果Nonce的CPU序列为0，则返回
    if (Nonce::sequence(Nonce::CPU) == 0) {
        return;
    }

    // 获取矿工的工作任务
    auto job = m_miner->job();

#   ifdef XMRIG_FEATURE_BENCHMARK
    // 获取基准测试大小
    m_benchSize          = job.benchSize();
    const uint32_t count = m_benchSize ? 1U : kReserveCount;
#   else
    // 设置保留计数
    constexpr uint32_t count = kReserveCount;
#   endif

    // 添加工作任务到队列中
    m_job.add(job, count, Nonce::CPU);

#   ifdef XMRIG_ALGO_RANDOMX
    // 如果当前工作任务的算法是RANDOM_X，则分配RandomX虚拟机
    if (m_job.currentJob().algorithm().family() == Algorithm::RANDOM_X) {
        allocateRandomX_VM();
    }
    else
#   endif
    {
        // 否则分配CnCtx
        allocateCnCtx();
    }
}

// 实例化CpuWorker类模板
namespace xmrig {
template class CpuWorker<1>;
template class CpuWorker<2>;
template class CpuWorker<3>;
template class CpuWorker<4>;
template class CpuWorker<5>;
template class CpuWorker<8>;
} // namespace xmrig
```