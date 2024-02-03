# `xmrig\src\crypto\rx\RxQueue.cpp`

```cpp
/* XMRig
 * 版权所有（c）2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有（c）2018-2019 tevador     <tevador@gmail.com>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它
 *   该许可证由自由软件基金会发布，可以选择遵循许可证的第3版或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，但没有任何担保；甚至没有暗示的担保
 *   商品性或适用于特定目的。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "crypto/rx/RxQueue.h"  // 包含自定义头文件 RxQueue.h
#include "backend/common/interfaces/IRxListener.h"  // 包含自定义头文件 IRxListener.h
#include "base/io/Async.h"  // 包含自定义头文件 Async.h
#include "base/io/log/Log.h"  // 包含自定义头文件 Log.h
#include "base/io/log/Tags.h"  // 包含自定义头文件 Tags.h
#include "base/tools/Cvt.h"  // 包含自定义头文件 Cvt.h
#include "crypto/rx/RxBasicStorage.h"  // 包含自定义头文件 RxBasicStorage.h

#ifdef XMRIG_FEATURE_HWLOC
#   include "crypto/rx/RxNUMAStorage.h"  // 如果定义了 XMRIG_FEATURE_HWLOC，则包含自定义头文件 RxNUMAStorage.h
#endif

// 使用 xmrig 命名空间，定义 RxQueue 类的构造函数
xmrig::RxQueue::RxQueue(IRxListener *listener) :
    m_listener(listener)  // 初始化成员变量 m_listener
{
    m_async  = std::make_shared<Async>(this);  // 使用 this 创建 Async 对象的共享指针，并赋值给 m_async
    m_thread = std::thread(&RxQueue::backgroundInit, this);  // 创建新线程，调用 backgroundInit 函数
}

// 定义 RxQueue 类的析构函数
xmrig::RxQueue::~RxQueue()
{
    std::unique_lock<std::mutex> lock(m_mutex);  // 使用互斥锁 m_mutex 创建独占所有权的锁
    m_state = STATE_SHUTDOWN;  // 设置成员变量 m_state 的值为 STATE_SHUTDOWN
    lock.unlock();  // 解锁

    m_cv.notify_one();  // 通知一个等待中的线程

    m_thread.join();  // 等待线程结束

    delete m_storage;  // 释放 m_storage 指向的内存
}

// 定义 RxQueue 类的 dataset 函数，返回 RxDataset 指针
xmrig::RxDataset *xmrig::RxQueue::dataset(const Job &job, uint32_t nodeId)
{
    std::lock_guard<std::mutex> lock(m_mutex);  // 使用互斥锁 m_mutex 创建的独占所有权的锁

    if (isReadyUnsafe(job)) {  // 如果 isReadyUnsafe 函数返回 true
        return m_storage->dataset(job, nodeId);  // 返回 m_storage 指向的对象的 dataset 函数的返回值
    }

    return nullptr;  // 返回空指针
}

// 定义 RxQueue 类的 hugePages 函数，返回 HugePagesInfo 对象
xmrig::HugePagesInfo xmrig::RxQueue::hugePages()
{
    std::lock_guard<std::mutex> lock(m_mutex);  // 使用互斥锁 m_mutex 创建的独占所有权的锁
    # 如果 m_storage 存在且 m_state 等于 STATE_IDLE，则返回 m_storage 的 hugePages() 方法的结果，否则返回 HugePagesInfo() 对象
    return m_storage && m_state == STATE_IDLE ? m_storage->hugePages() : HugePagesInfo();
# 检查 RxQueue 是否准备就绪
template<typename T>
bool xmrig::RxQueue::isReady(const T &seed)
{
    # 使用互斥锁保护临界区
    std::lock_guard<std::mutex> lock(m_mutex);

    # 调用 isReadyUnsafe 函数检查是否准备就绪
    return isReadyUnsafe(seed);
}

# 将任务加入队列
void xmrig::RxQueue::enqueue(const RxSeed &seed, const std::vector<uint32_t> &nodeset, uint32_t threads, bool hugePages, bool oneGbPages, RxConfig::Mode mode, int priority)
{
    # 使用独占锁保护临界区
    std::unique_lock<std::mutex> lock(m_mutex);

    # 如果存储对象为空
    if (!m_storage) {
#       ifdef XMRIG_FEATURE_HWLOC
        # 如果节点集不为空，使用 RxNUMAStorage 创建存储对象
        if (!nodeset.empty()) {
            m_storage = new RxNUMAStorage(nodeset);
        }
        # 否则使用 RxBasicStorage 创建存储对象
        else
#       endif
        {
            m_storage = new RxBasicStorage();
        }
    }

    # 如果状态为待处理且种子与当前种子相同，则直接返回
    if (m_state == STATE_PENDING && m_seed == seed) {
        return;
    }

    # 将任务加入队列
    m_queue.emplace_back(seed, nodeset, threads, hugePages, oneGbPages, mode, priority);
    m_seed  = seed;
    m_state = STATE_PENDING;

    # 解锁
    lock.unlock();

    # 通知等待的线程
    m_cv.notify_one();
}

# 检查 RxQueue 是否准备就绪（不安全版本）
template<typename T>
bool xmrig::RxQueue::isReadyUnsafe(const T &seed) const
{
    # 检查存储对象是否存在且已分配，状态是否空闲，种子是否相同
    return m_storage != nullptr && m_storage->isAllocated() && m_state == STATE_IDLE && m_seed == seed;
}

# 后台初始化函数
void xmrig::RxQueue::backgroundInit()
{
    # 当状态不是关闭状态时，进入循环
    while (m_state != STATE_SHUTDOWN) {
        # 使用互斥锁进行加锁
        std::unique_lock<std::mutex> lock(m_mutex);

        # 如果状态是空闲状态，则等待条件变量唤醒
        if (m_state == STATE_IDLE) {
            m_cv.wait(lock, [this]{ return m_state != STATE_IDLE; });
        }

        # 如果状态不是挂起状态，则继续下一次循环
        if (m_state != STATE_PENDING) {
            continue;
        }

        # 获取队列中最后一个元素，并清空队列
        const auto item = m_queue.back();
        m_queue.clear();

        # 解锁
        lock.unlock();

        # 记录日志信息，包括数据集初始化信息
        LOG_INFO("%s" MAGENTA_BOLD("init dataset%s") " algo " WHITE_BOLD("%s (") CYAN_BOLD("%u") WHITE_BOLD(" threads)") BLACK_BOLD(" seed %s..."),
                 Tags::randomx(),
                 item.nodeset.size() > 1 ? "s" : "",
                 item.seed.algorithm().name(),
                 item.threads,
                 Cvt::toHex(item.seed.data().data(), 8).data()
                 );

        # 使用存储对象进行数据集初始化
        m_storage->init(item.seed, item.threads, item.hugePages, item.oneGbPages, item.mode, item.priority);

        # 再次加锁
        lock.lock();

        # 如果状态是关闭状态或者队列不为空，则继续下一次循环
        if (m_state == STATE_SHUTDOWN || !m_queue.empty()) {
            continue;
        }

        # 在队列中有多个元素的情况下，再次更新种子
        m_seed = item.seed;
        m_state = STATE_IDLE;
        # 发送异步信号
        m_async->send();
    }
}

# 定义了一个名为xmrig的命名空间，用于组织相关的代码
void xmrig::RxQueue::onReady()
{
    # 创建一个互斥锁对象，并使用std::unique_lock对其进行封装
    std::unique_lock<std::mutex> lock(m_mutex);
    # 检查是否存在监听器并且状态为STATE_IDLE
    const bool ready = m_listener && m_state == STATE_IDLE;
    # 解锁互斥锁
    lock.unlock();

    # 如果满足条件，则调用监听器的onDatasetReady方法
    if (ready) {
        m_listener->onDatasetReady();
    }
}

# 声明了一个名为xmrig的命名空间
namespace xmrig {

# 实例化了一个模板函数isReady，用于判断给定的Job对象是否准备就绪
template bool RxQueue::isReady(const Job &);
# 实例化了一个模板函数isReady，用于判断给定的RxSeed对象是否准备就绪
template bool RxQueue::isReady(const RxSeed &);

} // namespace xmrig
```