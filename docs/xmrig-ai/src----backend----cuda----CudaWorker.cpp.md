# `xmrig\src\backend\cuda\CudaWorker.cpp`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的GNU通用公共许可证的条款，
 *   无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，
 *   但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 *   GNU通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "backend/cuda/CudaWorker.h"  // 包含 CUDA 工作器头文件
#include "backend/common/Tags.h"  // 包含通用标签头文件
#include "backend/cuda/runners/CudaCnRunner.h"  // 包含 CUDA CnRunner 头文件
#include "backend/cuda/wrappers/CudaDevice.h"  // 包含 CUDA 设备头文件
#include "base/io/log/Log.h"  // 包含日志头文件
#include "base/tools/Alignment.h"  // 包含对齐工具头文件
#include "base/tools/Chrono.h"  // 包含计时器头文件
#include "core/Miner.h"  // 包含矿工头文件
#include "crypto/common/Nonce.h"  // 包含随机数头文件
#include "net/JobResults.h"  // 包含作业结果头文件


#ifdef XMRIG_ALGO_RANDOMX
#   include "backend/cuda/runners/CudaRxRunner.h"  // 如果定义了 XMRIG_ALGO_RANDOMX，则包含 CUDA RxRunner 头文件
#endif


#ifdef XMRIG_ALGO_KAWPOW
#   include "backend/cuda/runners/CudaKawPowRunner.h"  // 如果定义了 XMRIG_ALGO_KAWPOW，则包含 CUDA KawPowRunner 头文件
#endif


#include <cassert>  // 包含断言头文件
#include <thread>  // 包含线程头文件


namespace xmrig {


std::atomic<bool> CudaWorker::ready;  // 定义静态原子布尔类型 ready


static inline bool isReady()    { return !Nonce::isPaused() && CudaWorker::ready; }  // 内联函数 isReady，返回非暂停状态和 ready 的值


} // namespace xmrig


// 构造函数，初始化 CUDA 工作器
xmrig::CudaWorker::CudaWorker(size_t id, const CudaLaunchData &data) :
    GpuWorker(id, data.thread.affinity(), -1, data.device.index()),  // 调用基类构造函数初始化 GpuWorker
    m_algorithm(data.algorithm),  // 初始化算法
    m_miner(data.miner)  // 初始化矿工
{
    switch (m_algorithm.family()) {  // 根据算法家族进行切换
    case Algorithm::RANDOM_X:  // 如果是 RANDOM_X 算法
#       ifdef XMRIG_ALGO_RANDOMX
        m_runner = new CudaRxRunner(id, data);  // 创建 CUDA RxRunner 对象
#       endif
        break;

    case Algorithm::ARGON2:  // 如果是 ARGON2 算法
        break;
    # 如果算法是KAWPOW
// 如果定义了 XMRIG_ALGO_KAWPOW，则创建一个 CudaKawPowRunner 对象
#ifdef XMRIG_ALGO_KAWPOW
    m_runner = new CudaKawPowRunner(id, data);
#endif
    // 退出 switch 语句
    break;

// 默认情况下，创建一个 CudaCnRunner 对象
default:
    m_runner = new CudaCnRunner(id, data);
    // 退出 switch 语句
    break;
}

// 如果 m_runner 为空指针，则直接返回
if (!m_runner) {
    return;
}

// 如果初始化失败，则删除 m_runner 对象，并将其指针置为空
if (!m_runner->init()) {
    delete m_runner;
    m_runner = nullptr;
}
}

// CudaWorker 类的析构函数，释放 m_runner 对象的内存
xmrig::CudaWorker::~CudaWorker()
{
    delete m_runner;
}

// 当收到新的作业通知时，如果 m_runner 不为空，则调用其 jobEarlyNotification 方法
void xmrig::CudaWorker::jobEarlyNotification(const Job &job)
{
    if (m_runner) {
        m_runner->jobEarlyNotification(job);
    }
}

// 进行自检，判断 m_runner 是否为空指针
bool xmrig::CudaWorker::selfTest()
{
    return m_runner != nullptr;
}

// 返回 m_runner 的 roundSize，如果 m_runner 为空则返回 0
size_t xmrig::CudaWorker::intensity() const
{
    return m_runner ? m_runner->roundSize() : 0;
}

// 启动 CudaWorker 对象的工作
void xmrig::CudaWorker::start()
{
    // 循环直到 Nonce::sequence(Nonce::CUDA) 为 0
    while (Nonce::sequence(Nonce::CUDA) > 0) {
        // 如果不准备好，则等待 200 毫秒
        if (!isReady()) {
            do {
                std::this_thread::sleep_for(std::chrono::milliseconds(200));
            }
            while (!isReady() && Nonce::sequence(Nonce::CUDA) > 0);

            // 如果 Nonce::sequence(Nonce::CUDA) 为 0，则退出循环
            if (Nonce::sequence(Nonce::CUDA) == 0) {
                break;
            }

            // 如果无法获取新的作业，则返回
            if (!consumeJob()) {
                return;
            }
        }

        // 在作业未过期的情况下执行挖矿操作
        while (!Nonce::isOutdated(Nonce::CUDA, m_job.sequence())) {
            uint32_t foundNonce[16] = { 0 };
            uint32_t foundCount     = 0;

            // 调用 m_runner 的 run 方法进行挖矿
            if (!m_runner->run(readUnaligned(m_job.nonce()), &foundCount, foundNonce)) {
                return;
            }

            // 如果找到了有效的 Nonce，则提交作业结果
            if (foundCount) {
                JobResults::submit(m_job.currentJob(), foundNonce, foundCount, m_deviceIndex);
            }

            // 如果作业未过期且未进入下一轮，则标记作业完成
            if (!Nonce::isOutdated(Nonce::CUDA, m_job.sequence()) && !m_job.nextRound(1, intensity())) {
                JobResults::done(m_job.currentJob());
            }

            // 存储统计信息，并让出线程执行权
            storeStats();
            std::this_thread::yield();
        }

        // 如果无法获取新的作业，则返回
        if (!consumeJob()) {
            return;
        }
    }
}

// 获取新的作业
bool xmrig::CudaWorker::consumeJob()
{
    # 如果使用 CUDA 的 Nonce 序列为 0，则返回 false
    if (Nonce::sequence(Nonce::CUDA) == 0) {
        return false;
    }

    # 将矿工的作业、强度和 CUDA 的 Nonce 添加到作业中
    m_job.add(m_miner->job(), intensity(), Nonce::CUDA);

    # 设置运行器的当前作业和作业数据
    return m_runner->set(m_job.currentJob(), m_job.blob());
# 存储统计信息的函数
void xmrig::CudaWorker::storeStats()
{
    # 如果不准备好，直接返回
    if (!isReady()) {
        return;
    }

    # 如果运行器存在，将处理的哈希数加到总数中
    m_count += m_runner ? m_runner->processedHashes() : 0;

    # 获取当前时间戳
    const uint64_t timeStamp = Chrono::steadyMSecs();
    # 将处理的哈希数和时间戳添加到哈希率数据中
    m_hashrateData.addDataPoint(m_count, timeStamp);

    # 调用父类的存储统计信息函数
    GpuWorker::storeStats();
}
```