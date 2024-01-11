# `xmrig\src\backend\cpu\CpuWorker.h`

```
/* XMRig
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新分发或修改
 *   根据GNU通用公共许可证的条款发布，由
 *   自由软件基金会发布的许可证的第3版，或者
 *   （在您的选择）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CPUWORKER_H
#define XMRIG_CPUWORKER_H


#include "backend/common/Worker.h"
#include "backend/common/WorkerJob.h"
#include "backend/cpu/CpuLaunchData.h"
#include "base/tools/Object.h"
#include "net/JobResult.h"


#ifdef XMRIG_ALGO_RANDOMX
class randomx_vm;
#endif


namespace xmrig {


class RxVm;


#ifdef XMRIG_ALGO_GHOSTRIDER
namespace ghostrider { struct HelperThread; }
#endif


template<size_t N>
class CpuWorker : public Worker
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(CpuWorker)

    // 构造函数，初始化CPU工作线程
    CpuWorker(size_t id, const CpuLaunchData &data);
    // 析构函数
    ~CpuWorker() override;

    // 返回线程数
    size_t threads() const override
    {
#       ifdef XMRIG_ALGO_GHOSTRIDER
        return ((m_algorithm.family() == Algorithm::GHOSTRIDER) && m_ghHelper) ? 2 : 1;
#       else
        return 1;
#       endif
    }

protected:
    // 自检函数
    bool selfTest() override;
    // 返回哈希率数据
    void hashrateData(uint64_t &hashCount, uint64_t &timeStamp, uint64_t &rawHashes) const override;
    // 启动函数
    void start() override;

    // 返回内存信息
    inline const VirtualMemory *memory() const override     { return m_memory; }
    // 返回强度
    inline size_t intensity() const override                { return N; }
    // 重写基类的 jobEarlyNotification 方法，但不做任何实际操作
    inline void jobEarlyNotification(const Job&) override   {}
private:
    // 定义内联函数，返回指定算法的哈希函数
    inline cn_hash_fun fn(const Algorithm &algorithm) const { return CnHash::fn(algorithm, m_av, m_assembly); }

#   ifdef XMRIG_ALGO_RANDOMX
    // 分配 RandomX 虚拟机的内存
    void allocateRandomX_VM();
#   endif

    // 进入下一轮挖矿
    bool nextRound();
    // 验证哈希值是否符合指定算法和参考值
    bool verify(const Algorithm &algorithm, const uint8_t *referenceValue);
    // 验证哈希值是否符合指定算法和参考值
    bool verify2(const Algorithm &algorithm, const uint8_t *referenceValue);
    // 分配 Cryptonight 上下文
    void allocateCnCtx();
    // 处理工作任务
    void consumeJob();

    // 8 字节对齐的哈希数组
    alignas(8) uint8_t m_hash[N * 32]{ 0 };
    // 算法类型
    const Algorithm m_algorithm;
    // 汇编类型
    const Assembly m_assembly;
    // 是否支持硬件 AES
    const bool m_hwAES;
    // 是否让出 CPU 时间
    const bool m_yield;
    // 算法变体
    const CnHash::AlgoVariant m_av;
    // 挖矿程序指针
    const Miner *m_miner;
    // 线程数
    const size_t m_threads;
    // Cryptonight 上下文数组
    cryptonight_ctx *m_ctx[N];
    // 虚拟内存指针
    VirtualMemory *m_memory = nullptr;
    // 工作任务
    WorkerJob<N> m_job;

#   ifdef XMRIG_ALGO_RANDOMX
    // RandomX 虚拟机指针
    randomx_vm *m_vm        = nullptr;
    // 随机种子缓冲区
    Buffer m_seed;
#   endif

#   ifdef XMRIG_ALGO_GHOSTRIDER
    // GhostRider 辅助线程指针
    ghostrider::HelperThread* m_ghHelper = nullptr;
#   endif

#   ifdef XMRIG_FEATURE_BENCHMARK
    // 基准测试大小
    uint32_t m_benchSize    = 0;
#   endif
};


// 针对单线程的特化模板，验证哈希值是否符合指定算法和参考值
template<>
bool CpuWorker<1>::verify2(const Algorithm &algorithm, const uint8_t *referenceValue);


// 显式实例化模板类
extern template class CpuWorker<1>;
extern template class CpuWorker<2>;
extern template class CpuWorker<3>;
extern template class CpuWorker<4>;
extern template class CpuWorker<5>;
extern template class CpuWorker<8>;


} // namespace xmrig


#endif /* XMRIG_CPUWORKER_H */
```