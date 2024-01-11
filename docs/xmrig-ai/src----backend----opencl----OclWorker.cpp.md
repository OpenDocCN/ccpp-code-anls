# `xmrig\src\backend\opencl\OclWorker.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   无论是许可证的第 3 版还是（在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用的，但没有任何保证；甚至没有对适销性或特定用途的隐含保证。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "backend/opencl/OclWorker.h"  // 引入 OpenCL 工作者头文件
#include "backend/common/Tags.h"  // 引入通用标签头文件
#include "backend/opencl/runners/OclCnRunner.h"  // 引入 OpenCL Cn 运行器头文件
#include "backend/opencl/runners/tools/OclSharedData.h"  // 引入 OpenCL 共享数据工具头文件
#include "backend/opencl/runners/tools/OclSharedState.h"  // 引入 OpenCL 共享状态工具头文件
#include "base/io/log/Log.h"  // 引入日志记录工具头文件
#include "base/tools/Alignment.h"  // 引入对齐工具头文件
#include "base/tools/Chrono.h"  // 引入计时工具头文件
#include "core/Miner.h"  // 引入矿工核心头文件
#include "crypto/common/Nonce.h"  // 引入常见 Nonce 头文件
#include "net/JobResults.h"  // 引入作业结果头文件


#ifdef XMRIG_ALGO_RANDOMX
#   include "backend/opencl/runners/OclRxJitRunner.h"  // 如果定义了 XMRIG_ALGO_RANDOMX，则引入 OpenCL RxJit 运行器头文件
#   include "backend/opencl/runners/OclRxVmRunner.h"  // 如果定义了 XMRIG_ALGO_RANDOMX，则引入 OpenCL RxVm 运行器头文件
#endif

#ifdef XMRIG_ALGO_KAWPOW
#   include "backend/opencl/runners/OclKawPowRunner.h"  // 如果定义了 XMRIG_ALGO_KAWPOW，则引入 OpenCL KawPow 运行器头文件
#endif

#include <cassert>  // 引入断言头文件
#include <thread>  // 引入线程头文件


namespace xmrig {


std::atomic<bool> OclWorker::ready;  // 定义静态原子布尔类型 ready


static inline bool isReady()    { return !Nonce::isPaused() && OclWorker::ready; }  // 定义内联函数 isReady，用于检查是否准备就绪


static inline void printError(size_t id, const char *error)  // 定义内联函数 printError，用于打印错误信息
{
    LOG_ERR("%s" RED_S " thread " RED_BOLD("#%zu") RED_S " failed with error " RED_BOLD("%s"), ocl_tag(), id, error);  // 记录错误信息到日志
}


} // namespace xmrig



xmrig::OclWorker::OclWorker(size_t id, const OclLaunchData &data) :  // 定义 OclWorker 构造函数
    GpuWorker(id, data.affinity, -1, data.device.index()),  // 调用 GpuWorker 构造函数初始化
    # 调用 m_algorithm 函数，传入 data.algorithm 参数
    m_algorithm(data.algorithm),
    # 调用 m_miner 函数，传入 data.miner 参数
    m_miner(data.miner),
    # 调用 OclSharedState::get 函数，传入 data.device.index() 参数，将返回值赋给 m_sharedData
    m_sharedData(OclSharedState::get(data.device.index()))
{
    // 根据算法家族选择不同的处理方式
    switch (m_algorithm.family()) {
    case Algorithm::RANDOM_X:
#       ifdef XMRIG_ALGO_RANDOMX
        // 如果线程是汇编并且设备的供应商 ID 是 OCL_VENDOR_AMD，则使用 OclRxJitRunner
        if (data.thread.isAsm() && data.device.vendorId() == OCL_VENDOR_AMD) {
            m_runner = new OclRxJitRunner(id, data);
        }
        // 否则使用 OclRxVmRunner
        else {
            m_runner = new OclRxVmRunner(id, data);
        }
#       endif
        break;

    case Algorithm::ARGON2:
#       ifdef XMRIG_ALGO_ARGON2
        // 对于 ARGON2 算法，不使用任何特定的 runner
        m_runner = nullptr;
#       endif
        break;

    case Algorithm::KAWPOW:
#       ifdef XMRIG_ALGO_KAWPOW
        // 对于 KAWPOW 算法，使用 OclKawPowRunner
        m_runner = new OclKawPowRunner(id, data);
#       endif
        break;

    default:
        // 对于其他算法，使用 OclCnRunner
        m_runner = new OclCnRunner(id, data);
        break;
    }

    // 如果没有选择 runner，则直接返回
    if (!m_runner) {
        return;
    }

    // 尝试初始化和构建 runner
    try {
        m_runner->init();
        m_runner->build();
    }
    // 捕获异常并打印错误信息
    catch (std::exception &ex) {
        printError(id, ex.what());

        // 删除 runner 并将其置为 nullptr
        delete m_runner;
        m_runner = nullptr;
    }
}


// OclWorker 类的析构函数
xmrig::OclWorker::~OclWorker()
{
    // 删除 runner
    delete m_runner;
}


// 当收到新的作业通知时调用的函数
void xmrig::OclWorker::jobEarlyNotification(const Job &job)
{
    // 如果存在 runner，则调用其 jobEarlyNotification 函数
    if (m_runner) {
        m_runner->jobEarlyNotification(job);
    }
}


// 进行自检，检查是否存在 runner
bool xmrig::OclWorker::selfTest()
{
    return m_runner != nullptr;
}


// 获取 runner 的强度
size_t xmrig::OclWorker::intensity() const
{
    // 如果存在 runner，则返回其 roundSize，否则返回 0
    return m_runner ? m_runner->roundSize() : 0;
}


// 启动函数
void xmrig::OclWorker::start()
{
    // 初始化结果数组
    cl_uint results[0x100];
    # 当 Nonce::sequence(Nonce::OPENCL) 大于 0 时执行循环
    while (Nonce::sequence(Nonce::OPENCL) > 0) {
        # 如果不是准备好状态，则设置共享数据的恢复计数器为 0
        if (!isReady()) {
            m_sharedData.setResumeCounter(0);

            # 进入循环，直到准备好状态或者 Nonce::sequence(Nonce::OPENCL) 不大于 0
            do {
                std::this_thread::sleep_for(std::chrono::milliseconds(200));
            }
            while (!isReady() && Nonce::sequence(Nonce::OPENCL) > 0);

            # 如果 Nonce::sequence(Nonce::OPENCL) 等于 0，则跳出循环
            if (Nonce::sequence(Nonce::OPENCL) == 0) {
                break;
            }

            # 根据当前线程 ID 设置共享数据的恢复延迟
            m_sharedData.resumeDelay(id());

            # 如果无法消费任务，则返回
            if (!consumeJob()) {
                return;
            }
        }

        # 当 Nonce::OPENCL 的 Nonce 不过时时执行循环
        while (!Nonce::isOutdated(Nonce::OPENCL, m_job.sequence())) {
            # 调整共享数据的延迟
            m_sharedData.adjustDelay(id());

            # 获取当前时间
            const uint64_t t = Chrono::steadyMSecs();

            try {
                # 运行任务，并将结果存储在 results 中
                m_runner->run(readUnaligned(m_job.nonce()), results);
            }
            catch (std::exception &ex) {
                # 捕获异常并打印错误信息，然后返回
                printError(id(), ex.what());

                return;
            }

            # 如果结果中的某个值大于 0，则提交作业结果
            if (results[0xFF] > 0) {
                JobResults::submit(m_job.currentJob(), results, results[0xFF], m_deviceIndex);
            }

            # 如果 Nonce::OPENCL 的 Nonce 不过时且未进入下一轮，则标记作业结果为完成
            if (!Nonce::isOutdated(Nonce::OPENCL, m_job.sequence()) && !m_job.nextRound(1, intensity())) {
                JobResults::done(m_job.currentJob());
            }

            # 存储统计信息，并让出当前线程的执行权
            storeStats(t);
            std::this_thread::yield();
        }

        # 如果无法消费任务，则返回
        if (!consumeJob()) {
            return;
        }
    }
# 检查是否有可用的 OpenCL 序列号，如果没有则返回 false
bool xmrig::OclWorker::consumeJob()
{
    if (Nonce::sequence(Nonce::OPENCL) == 0) {
        return false;
    }

    # 将矿工的作业、强度和 OpenCL 序列号添加到作业队列中
    m_job.add(m_miner->job(), intensity(), Nonce::OPENCL);

    try {
        # 设置当前作业和作业数据块到运行器中
        m_runner->set(m_job.currentJob(), m_job.blob());
    }
    catch (std::exception &ex) {
        # 捕获异常并打印错误信息
        printError(id(), ex.what());

        return false;
    }

    return true;
}

# 存储统计信息，包括处理的哈希数量和时间戳
void xmrig::OclWorker::storeStats(uint64_t t)
{
    # 如果不准备好，则返回
    if (!isReady()) {
        return;
    }

    # 增加处理的哈希数量，并获取当前时间戳
    m_count += m_runner->processedHashes();
    const uint64_t timeStamp = Chrono::steadyMSecs();

    # 将哈希数量和时间戳添加到哈希率数据中
    m_hashrateData.addDataPoint(m_count, timeStamp);

    # 设置运行时间
    m_sharedData.setRunTime(timeStamp - t);

    # 调用父类的存储统计信息方法
    GpuWorker::storeStats();
}
```