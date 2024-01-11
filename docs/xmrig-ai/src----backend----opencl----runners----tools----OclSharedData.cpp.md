# `xmrig\src\backend\opencl\runners\tools\OclSharedData.cpp`

```
/* XMRig
 * 版权所有（c）2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有对适销性或特定用途的隐含保证。请参阅
 * GNU通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/runners/tools/OclSharedData.h"
#include "backend/opencl/wrappers/OclLib.h"
#include "base/io/log/Log.h"
#include "base/tools/Chrono.h"
#include "crypto/rx/Rx.h"
#include "crypto/rx/RxDataset.h"


#include <algorithm>
#include <cinttypes>
#include <stdexcept>
#include <thread>


// 定义一个常量，表示1GB的大小
constexpr size_t oneGiB = 1024 * 1024 * 1024;


// 创建一个OpenCL缓冲区
cl_mem xmrig::OclSharedData::createBuffer(cl_context context, size_t size, size_t &offset, size_t limit)
{
    // 使用互斥锁保护临界区
    std::lock_guard<std::mutex> lock(m_mutex);

    // 计算缓冲区的位置
    const size_t pos = offset + (size * m_offset);
    // 将缓冲区大小调整为至少1GB
    size             = std::max(size * m_threads, oneGiB);

    // 如果缓冲区大小超过限制，则返回空指针
    if (size > limit) {
        return nullptr;
    }

    // 更新偏移量
    offset = pos;
    ++m_offset;

    // 如果缓冲区为空，则创建一个新的缓冲区
    if (!m_buffer) {
        m_buffer = OclLib::createBuffer(context, CL_MEM_READ_WRITE, size);
    }

    // 保留缓冲区
    return OclLib::retain(m_buffer);
}


// 调整延迟
uint64_t xmrig::OclSharedData::adjustDelay(size_t /*id*/)
{
    // 如果线程数小于2，则返回0
    if (m_threads < 2) {
        return 0;
    }

    // 获取当前时间
    const uint64_t t0 = Chrono::steadyMSecs();
    uint64_t delay    = 0;
    {
        // 使用互斥锁保护临界区，确保线程安全
        std::lock_guard<std::mutex> lock(m_mutex);

        // 计算时间间隔
        const uint64_t dt = t0 - m_timestamp;
        // 更新时间戳
        m_timestamp = t0;

        // 完美交错是指在同一 GPU 上的 N 个线程之间以 T/N 的间隔开始
        // 如果一个线程在前一个线程之后早于 0.75*T/N 毫秒开始，延迟它以恢复完美交错
        if ((dt > 0) && (dt < m_threshold * (m_averageRunTime / m_threads))) {
            // 计算延迟时间
            delay = static_cast<uint64_t>(m_averageRunTime / m_threads - dt);
            // 更新阈值
            m_threshold = 0.75;
        }
    }

    // 如果延迟时间为 0，则直接返回
    if (delay == 0) {
        return 0;
    }

    // 如果延迟时间大于等于 400 毫秒，则将延迟时间设置为 200 毫秒
    if (delay >= 400) {
        delay = 200;
    }

    // 休眠指定的时间
    std::this_thread::sleep_for(std::chrono::milliseconds(delay));
}
#   ifdef XMRIG_INTERLEAVE_DEBUG
    #ifdef 指令用于条件编译，如果定义了 XMRIG_INTERLEAVE_DEBUG，则执行下面的代码
    LOG_WARN("Thread #%zu was paused for %" PRIu64 " ms to adjust interleaving", id, delay);
    #endif
#   endif
    #ifdef 指令结束

    return delay;
    # 返回 delay 变量的值


uint64_t xmrig::OclSharedData::resumeDelay(size_t /*id*/)
{
    # 根据线程数量判断是否需要暂停
    if (m_threads < 2) {
        return 0;
    }
    # 如果线程数量小于 2，则返回 0

    uint64_t delay = 0;
    # 定义 delay 变量并初始化为 0

    {
        constexpr const double firstThreadSpeedupCoeff = 1.25;
        # 定义第一个线程的加速系数

        std::lock_guard<std::mutex> lock(m_mutex);
        # 使用互斥锁保护临界区
        delay = static_cast<uint64_t>(m_resumeCounter * m_averageRunTime / m_threads / firstThreadSpeedupCoeff);
        # 计算暂停的延迟时间
        ++m_resumeCounter;
        # 增加 m_resumeCounter 的值
    }
    # 临界区结束

    if (delay == 0) {
        return 0;
    }
    # 如果延迟时间为 0，则返回 0

    if (delay > 1000) {
        delay = 1000;
    }
    # 如果延迟时间大于 1000，则将延迟时间设置为 1000

#   ifdef XMRIG_INTERLEAVE_DEBUG
    #ifdef 指令用于条件编译，如果定义了 XMRIG_INTERLEAVE_DEBUG，则执行下面的代码
    LOG_WARN("Thread #%zu will be paused for %" PRIu64 " ms to before resuming", id, delay);
    #endif
#   endif
    #ifdef 指令结束

    std::this_thread::sleep_for(std::chrono::milliseconds(delay));
    # 使当前线程休眠指定的时间

    return delay;
    # 返回 delay 变量的值
}


void xmrig::OclSharedData::release()
{
    # 释放 OpenCL 缓冲区
    OclLib::release(m_buffer);

#   ifdef XMRIG_ALGO_RANDOMX
    #ifdef 指令用于条件编译，如果定义了 XMRIG_ALGO_RANDOMX，则执行下面的代码
    OclLib::release(m_dataset);
    #endif
#   endif
    #ifdef 指令结束
}


void xmrig::OclSharedData::setResumeCounter(uint32_t value)
{
    # 根据线程数量判断是否需要设置 m_resumeCounter 的值
    if (m_threads < 2) {
        return;
    }
    # 如果线程数量小于 2，则直接返回

    std::lock_guard<std::mutex> lock(m_mutex);
    # 使用互斥锁保护临界区
    m_resumeCounter = value;
    # 设置 m_resumeCounter 的值
}


void xmrig::OclSharedData::setRunTime(uint64_t time)
{
    # 设置平均运行时间的权重
    constexpr double averagingBias = 0.1;

    std::lock_guard<std::mutex> lock(m_mutex);
    # 使用互斥锁保护临界区
    m_averageRunTime = m_averageRunTime * (1.0 - averagingBias) + time * averagingBias;
    # 更新平均运行时间
}


#ifdef XMRIG_ALGO_RANDOMX
cl_mem xmrig::OclSharedData::dataset() const
{
    # 如果数据集不存在，则抛出运行时异常
    if (!m_dataset) {
        throw std::runtime_error("RandomX dataset is not available");
    }

    return OclLib::retain(m_dataset);
    # 返回数据集的内存对象
}
// 创建 OpenCL 数据集
void xmrig::OclSharedData::createDataset(cl_context ctx, const Job &job, bool host)
{
    // 如果数据集已经存在，则直接返回
    if (m_dataset) {
        return;
    }

    cl_int ret = 0;

    // 如果需要在主机上创建数据集
    if (host) {
        // 从作业中获取数据集
        auto dataset = Rx::dataset(job, 0);
        // 使用 OpenCL 库创建一个可读的、使用主机指针的缓冲区
        m_dataset = OclLib::createBuffer(ctx, CL_MEM_READ_ONLY | CL_MEM_USE_HOST_PTR, RxDataset::maxSize(), dataset->raw(), &ret);
    }
    // 如果需要在设备上创建数据集
    else {
        // 使用 OpenCL 库创建一个只读的缓冲区
        m_dataset = OclLib::createBuffer(ctx, CL_MEM_READ_ONLY, RxDataset::maxSize(), nullptr, &ret);
    }
}
```