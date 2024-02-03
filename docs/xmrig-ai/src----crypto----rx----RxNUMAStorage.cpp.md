# `xmrig\src\crypto\rx\RxNUMAStorage.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以自由地重新分发和/或修改
 *   根据GNU通用公共许可证的条款，发布的版本为3或
 *   （根据您的选择）以后的版本。
 *
 *   本程序是基于希望它有用的目的分发的，
 *   但没有任何担保；甚至没有适销性或特定用途的暗示保证。
 *   有关更多详细信息，请参阅GNU通用公共许可证。
 *
 *   如果没有收到GNU通用公共许可证的副本
 *   请参阅<http://www.gnu.org/licenses/>。
 */

#include "crypto/rx/RxNUMAStorage.h"
#include "backend/cpu/Cpu.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/kernel/Platform.h"
#include "base/tools/Chrono.h"
#include "crypto/rx/RxAlgo.h"
#include "crypto/rx/RxCache.h"
#include "crypto/rx/RxDataset.h"
#include "crypto/rx/RxSeed.h"


#include <map>
#include <mutex>
#include <hwloc.h>
#include <thread>


namespace xmrig {


constexpr size_t oneMiB = 1024 * 1024;
static std::mutex mutex;


// 将线程绑定到指定的NUMA节点
static bool bindToNUMANode(uint32_t nodeId)
{
    // 获取指定NUMA节点的对象
    auto node = hwloc_get_numanode_obj_by_os_index(Cpu::info()->topology(), nodeId);
    // 如果节点不存在，则返回false
    if (!node) {
        return false;
    }

    // 如果成功将内存绑定到节点
    if (Cpu::info()->membind(node->nodeset)) {
        // 设置线程亲和性为节点的第一个CPU
        Platform::setThreadAffinity(static_cast<uint64_t>(hwloc_bitmap_first(node->cpuset)));

        return true;
    }

    return false;
}


// 打印跳过的信息
static inline void printSkipped(uint32_t nodeId, const char *reason)
{
    // 使用日志记录跳过的信息
    LOG_WARN("%s" CYAN_BOLD("#%u ") RED_BOLD("skipped") YELLOW(" (%s)"), Tags::randomx(), nodeId, reason);
}


// 打印数据集准备就绪的信息
static inline void printDatasetReady(uint32_t nodeId, uint64_t ts)
{
    # 使用LOG_INFO函数输出日志信息，包括字符串%s、无颜色的#%u、绿色加粗的"dataset ready"、黑色加粗的(%" PRIu64 " ms)，分别对应Tags::randomx()、nodeId、Chrono::steadyMSecs() - ts
    LOG_INFO("%s" CYAN_BOLD("#%u ") GREEN_BOLD("dataset ready") BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::randomx(), nodeId, Chrono::steadyMSecs() - ts);
}

// 私有类 RxNUMAStoragePrivate 的定义
class RxNUMAStoragePrivate
{
public:
    // 禁用默认的拷贝和移动构造函数
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(RxNUMAStoragePrivate)

    // 显式构造函数，接受节点集合作为参数
    inline explicit RxNUMAStoragePrivate(const std::vector<uint32_t> &nodeset) :
        m_nodeset(nodeset)
    {
        // 预留节点集合大小的空间
        m_threads.reserve(nodeset.size());
    }

    // 析构函数
    inline ~RxNUMAStoragePrivate()
    {
        // 执行线程的加入操作
        join();

        // 遍历数据集合，释放内存
        for (auto const &item : m_datasets) {
            delete item.second;
        }
    }

    // 返回是否已分配
    inline bool isAllocated() const                     { return m_allocated; }
    // 返回是否准备就绪
    inline bool isReady(const Job &job) const           { return m_ready && m_seed == job; }
    // 返回指定节点的数据集
    inline RxDataset *dataset(uint32_t nodeId) const    { return m_datasets.count(nodeId) ? m_datasets.at(nodeId) : m_datasets.at(m_nodeset.front()); }

    // 设置种子
    inline void setSeed(const RxSeed &seed)
    {
        m_ready = false;

        // 如果种子算法不同，则应用新的算法
        if (m_seed.algorithm() != seed.algorithm()) {
            RxAlgo::apply(seed.algorithm());
        }

        m_seed = seed;
    }

    // 创建数据集
    inline bool createDatasets(bool hugePages, bool oneGbPages)
    {
        // 获取当前时间戳
        const uint64_t ts = Chrono::steadyMSecs();

        // 遍历节点集合，为每个节点分配数据集
        for (uint32_t node : m_nodeset) {
            m_threads.emplace_back(allocate, this, node, hugePages, oneGbPages);
        }

        // 加入线程
        join();

        // 如果需要缓存
        if (isCacheRequired()) {
            // 分配缓存
            std::thread thread(allocateCache, this, m_nodeset.front(), hugePages);
            thread.join();

            // 如果缓存分配失败，则返回false
            if (!m_cache) {
                return false;
            }
        }

        // 如果数据集为空
        if (m_datasets.empty()) {
            // 插入新的数据集
            m_datasets.insert({ m_nodeset.front(), new RxDataset(m_cache) });

            // 输出警告信息
            LOG_WARN(CLEAR "%s" YELLOW_BOLD_S "failed to allocate RandomX datasets, switching to slow mode" BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::randomx(), Chrono::steadyMSecs() - ts);
        }
        else {
            // 如果有缓存，则设置数据集的缓存
            if (m_cache) {
                dataset(m_nodeset.front())->setCache(m_cache);
            }

            // 打印分配状态
            printAllocStatus(ts);
        }

        // 设置已分配标志为true
        m_allocated = true;

        // 返回true
        return true;
    }
    // 检查是否需要缓存，如果数据集为空则需要缓存，否则遍历数据集，如果有一个数据集需要使用1GB页面，则不需要缓存，否则需要缓存
    inline bool isCacheRequired() const
    {
        if (m_datasets.empty()) {
            return true;
        }

        for (const auto &kv : m_datasets) {
            if (kv.second->isOneGbPages()) {
                return false;
            }
        }

        return true;
    }


    // 初始化数据集，设置线程数和优先级
    inline void initDatasets(uint32_t threads, int priority)
    {
        uint64_t ts = Chrono::steadyMSecs();  // 获取当前时间戳
        uint32_t id = 0;

        // 遍历数据集，找到需要缓存的数据集
        for (const auto &kv : m_datasets) {
            if (kv.second->cache()) {
                id = kv.first;
            }
        }

        auto primary = dataset(id);  // 获取主要数据集
        primary->init(m_seed.data(), threads, priority);  // 初始化主要数据集

        printDatasetReady(id, ts);  // 打印数据集准备就绪的信息

        // 如果数据集数量大于1，则复制其他数据集，并启动线程进行复制
        if (m_datasets.size() > 1) {
            for (auto const &item : m_datasets) {
                if (item.first == id) {
                    continue;
                }

                m_threads.emplace_back(copyDataset, item.second, item.first, primary->raw());  // 复制数据集并启动线程
            }

            join();  // 等待所有线程结束
        }

        m_ready = true;  // 设置数据集准备就绪标志为true
    }


    // 获取所有数据集的大页信息并返回
    inline HugePagesInfo hugePages() const
    {
        HugePagesInfo pages;
        for (auto const &item : m_datasets) {
            pages += item.second->hugePages();  // 获取每个数据集的大页信息并累加
        }

        return pages;  // 返回所有数据集的大页信息
    }
# 分配内存，将数据存储在指定的 NUMA 节点上
private:
    static void allocate(RxNUMAStoragePrivate *d_ptr, uint32_t nodeId, bool hugePages, bool oneGbPages)
    {
        # 记录当前时间
        const uint64_t ts = Chrono::steadyMSecs();

        # 将内存绑定到指定的 NUMA 节点上
        if (!bindToNUMANode(nodeId)) {
            # 如果无法将内存绑定到 NUMA 节点上，则打印跳过的信息并返回
            printSkipped(nodeId, "can't bind memory");

            return;
        }

        # 创建一个新的 RxDataset 对象
        auto dataset = new RxDataset(hugePages, oneGbPages, false, RxConfig::FastMode, nodeId);
        # 如果无法分配数据集，则打印跳过的信息并释放内存
        if (!dataset->get()) {
            printSkipped(nodeId, "failed to allocate dataset");

            delete dataset;
            return;
        }

        # 加锁，保证线程安全
        std::lock_guard<std::mutex> lock(mutex);
        # 将数据集插入到数据集字典中
        d_ptr->m_datasets.insert({ nodeId, dataset });
        # 打印分配状态信息
        RxNUMAStoragePrivate::printAllocStatus(dataset, nodeId, ts);
    }


    # 分配缓存内存，将缓存数据存储在指定的 NUMA 节点上
    static void allocateCache(RxNUMAStoragePrivate *d_ptr, uint32_t nodeId, bool hugePages)
    {
        # 记录当前时间
        const uint64_t ts = Chrono::steadyMSecs();

        # 将缓存内存绑定到指定的 NUMA 节点上
        bindToNUMANode(nodeId);

        # 创建一个新的 RxCache 对象
        auto cache = new RxCache(hugePages, nodeId);
        # 如果无法分配缓存，则打印错误信息并返回
        if (!cache->get()) {
            delete cache;

            LOG_INFO("%s" RED_BOLD("failed to allocate RandomX memory") BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::randomx(), Chrono::steadyMSecs() - ts);

            return;
        }

        # 加锁，保证线程安全
        std::lock_guard<std::mutex> lock(mutex);
        # 将缓存对象存储在类的成员变量中
        d_ptr->m_cache = cache;
        # 打印分配状态信息
        RxNUMAStoragePrivate::printAllocStatus(cache, nodeId, ts);
    }


    # 将原始数据复制到数据集对象中
    static void copyDataset(RxDataset *dst, uint32_t nodeId, const void *raw)
    {
        # 记录当前时间
        const uint64_t ts = Chrono::steadyMSecs();

        # 将原始数据设置到数据集对象中
        dst->setRaw(raw);

        # 打印数据集准备就绪的信息
        printDatasetReady(nodeId, ts);
    }


    # 打印分配状态信息
    static void printAllocStatus(RxDataset *dataset, uint32_t nodeId, uint64_t ts)
    {
        // 获取数据集的巨大页面信息
        const auto pages = dataset->hugePages();
    
        // 打印巨大页面的分配状态
        LOG_INFO("%s" CYAN_BOLD("#%u ") GREEN_BOLD("allocated") CYAN_BOLD(" %zu MB") " huge pages %s%3.0f%%" CLEAR BLACK_BOLD(" (%" PRIu64 " ms)"),
                 Tags::randomx(),
                 nodeId,
                 pages.size / oneMiB,
                 (pages.isFullyAllocated() ? GREEN_BOLD_S : RED_BOLD_S),
                 pages.percent(),
                 Chrono::steadyMSecs() - ts
                 );
    }
    
    
    static void printAllocStatus(RxCache *cache, uint32_t nodeId, uint64_t ts)
    {
        // 获取缓存的巨大页面信息
        const auto pages = cache->hugePages();
    
        // 打印巨大页面的分配状态
        LOG_INFO("%s" CYAN_BOLD("#%u ") GREEN_BOLD("allocated") CYAN_BOLD(" %4zu MB") " huge pages %s%3.0f%%" CLEAR " %sJIT" BLACK_BOLD(" (%" PRIu64 " ms)"),
                 Tags::randomx(),
                 nodeId,
                 cache->size() / oneMiB,
                 (pages.isFullyAllocated() ? GREEN_BOLD_S : RED_BOLD_S),
                 pages.percent(),
                 cache->isJIT() ? GREEN_BOLD_S "+" : RED_BOLD_S "-",
                 Chrono::steadyMSecs() - ts
                 );
    }
    
    
    void printAllocStatus(uint64_t ts) const
    {
        // 获取巨大页面信息
        auto pages = hugePages();
    
        // 打印巨大页面的分配状态
        LOG_INFO("%s" CYAN_BOLD("-- ") GREEN_BOLD("allocated") CYAN_BOLD(" %4zu MB") " huge pages %s%3.0f%% %u/%u" CLEAR BLACK_BOLD(" (%" PRIu64 " ms)"),
                 Tags::randomx(),
                 pages.size / oneMiB,
                 (pages.isFullyAllocated() ? GREEN_BOLD_S : (pages.allocated == 0 ? RED_BOLD_S : YELLOW_BOLD_S)),
                 pages.percent(),
                 pages.allocated,
                 pages.total,
                 Chrono::steadyMSecs() - ts
                 );
    }
    
    
    inline void join()
    {
        // 等待所有线程结束
        for (auto &thread : m_threads) {
            thread.join();
        }
    
        // 清空线程列表
        m_threads.clear();
    }
    
    
    bool m_allocated        = false;
    bool m_ready            = false;
    RxCache *m_cache        = nullptr;
    RxSeed m_seed;
    
    
    注释：
    // 创建一个映射，将无符号32位整数映射到RxDataset指针
    std::map<uint32_t, RxDataset *> m_datasets;
    // 创建一个线程向量，用于存储线程对象
    std::vector<std::thread> m_threads;
    // 创建一个无符号32位整数向量，用于存储节点集合
    std::vector<uint32_t> m_nodeset;
// 命名空间 xmrig
namespace xmrig {

    // RxNUMAStorage 类的构造函数，接受一个节点集合作为参数
    xmrig::RxNUMAStorage::RxNUMAStorage(const std::vector<uint32_t> &nodeset) :
        // 创建一个 RxNUMAStoragePrivate 对象，并将其指针赋值给 d_ptr
        d_ptr(new RxNUMAStoragePrivate(nodeset))
    {
    }

    // RxNUMAStorage 类的析构函数
    xmrig::RxNUMAStorage::~RxNUMAStorage()
    {
        // 删除 d_ptr 指针指向的对象
        delete d_ptr;
    }

    // 判断是否已分配存储空间的函数
    bool xmrig::RxNUMAStorage::isAllocated() const
    {
        // 调用 d_ptr 指针指向对象的 isAllocated() 函数，并返回结果
        return d_ptr->isAllocated();
    }

    // 获取 HugePagesInfo 对象的函数
    xmrig::HugePagesInfo xmrig::RxNUMAStorage::hugePages() const
    {
        // 如果存储空间未分配，则返回一个空的 HugePagesInfo 对象
        if (!d_ptr->isAllocated()) {
            return {};
        }

        // 调用 d_ptr 指针指向对象的 hugePages() 函数，并返回结果
        return d_ptr->hugePages();
    }

    // 获取 RxDataset 对象的函数，接受一个 Job 对象和节点 ID 作为参数
    xmrig::RxDataset *xmrig::RxNUMAStorage::dataset(const Job &job, uint32_t nodeId) const
    {
        // 如果存储空间未准备好，则返回空指针
        if (!d_ptr->isReady(job)) {
            return nullptr;
        }

        // 调用 d_ptr 指针指向对象的 dataset() 函数，并返回结果
        return d_ptr->dataset(nodeId);
    }

    // 初始化函数，接受一个 RxSeed 对象、线程数、是否使用 HugePages、是否使用 1GB 页、RxConfig::Mode 对象和优先级作为参数
    void xmrig::RxNUMAStorage::init(const RxSeed &seed, uint32_t threads, bool hugePages, bool oneGbPages, RxConfig::Mode, int priority)
    {
        // 设置 d_ptr 指针指向对象的种子
        d_ptr->setSeed(seed);

        // 如果存储空间未分配且无法创建数据集，则返回
        if (!d_ptr->isAllocated() && !d_ptr->createDatasets(hugePages, oneGbPages)) {
            return;
        }

        // 初始化数据集
        d_ptr->initDatasets(threads, priority);
    }

} // namespace xmrig
```