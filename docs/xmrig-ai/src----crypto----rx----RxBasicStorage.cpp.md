# `xmrig\src\crypto\rx\RxBasicStorage.cpp`

```
/* XMRig
 * 版权所有 (c) 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，
 *   其版本由自由软件基金会发布，可以选择使用版本 3 或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，但没有任何担保；甚至没有适用于特定目的的隐含担保。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参阅 <http://www.gnu.org/licenses/>。
 */


#include "crypto/rx/RxBasicStorage.h"
#include "backend/common/Tags.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/tools/Chrono.h"
#include "crypto/rx/RxAlgo.h"
#include "crypto/rx/RxCache.h"
#include "crypto/rx/RxDataset.h"
#include "crypto/rx/RxSeed.h"


namespace xmrig {


constexpr size_t oneMiB = 1024 * 1024;


class RxBasicStoragePrivate
{
public:
    XMRIG_DISABLE_COPY_MOVE(RxBasicStoragePrivate)

    inline RxBasicStoragePrivate() = default;
    inline ~RxBasicStoragePrivate() { deleteDataset(); }

    inline bool isReady(const Job &job) const   { return m_ready && m_seed == job; }
    inline RxDataset *dataset() const           { return m_dataset; }
    inline void deleteDataset()                 { delete m_dataset; m_dataset = nullptr; }


    inline void setSeed(const RxSeed &seed)
    {
        m_ready = false;

        if (m_seed.algorithm() != seed.algorithm()) {
            RxAlgo::apply(seed.algorithm());
        }

        m_seed = seed;
    }


    inline bool createDataset(bool hugePages, bool oneGbPages, RxConfig::Mode mode)
    {
        // 获取当前时间戳
        const uint64_t ts = Chrono::steadyMSecs();
    
        // 创建一个新的RxDataset对象，并根据参数设置内存分配方式
        m_dataset = new RxDataset(hugePages, oneGbPages, true, mode, 0);
        // 如果内存分配失败，则删除数据集对象，打印错误信息，并返回false
        if (!m_dataset->cache()->get()) {
            deleteDataset();
    
            // 打印错误信息，包括失败的时间消耗
            LOG_INFO("%s" RED_BOLD("failed to allocate RandomX memory") BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::randomx(), Chrono::steadyMSecs() - ts);
    
            return false;
        }
    
        // 打印内存分配状态
        printAllocStatus(ts);
    
        // 返回true表示成功
        return true;
    }
    
    
    // 初始化数据集对象
    inline void initDataset(uint32_t threads, int priority)
    {
        // 获取当前时间戳
        const uint64_t ts = Chrono::steadyMSecs();
    
        // 调用数据集对象的init方法，传入种子数据、线程数和优先级参数
        m_ready = m_dataset->init(m_seed.data(), threads, priority);
    
        // 如果初始化成功，则打印成功信息，包括初始化的时间消耗
        if (m_ready) {
            LOG_INFO("%s" GREEN_BOLD("dataset ready") BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::randomx(), Chrono::steadyMSecs() - ts);
        }
    }
// 打印内存分配状态信息
private:
    void printAllocStatus(uint64_t ts)
    {
        // 检查数据集是否存在
        if (m_dataset->get() != nullptr) {
            // 获取巨大页面信息
            const auto pages = m_dataset->hugePages();

            // 打印分配的巨大页面信息
            LOG_INFO("%s" GREEN_BOLD("allocated") CYAN_BOLD(" %zu MB") BLACK_BOLD(" (%zu+%zu)") " huge pages %s%1.0f%% %u/%u" CLEAR " %sJIT" BLACK_BOLD(" (%" PRIu64 " ms)"),
                     Tags::randomx(),
                     pages.size / oneMiB,
                     RxDataset::maxSize() / oneMiB,
                     RxCache::maxSize() / oneMiB,
                     (pages.isFullyAllocated() ? GREEN_BOLD_S : (pages.allocated == 0 ? RED_BOLD_S : YELLOW_BOLD_S)),
                     pages.percent(),
                     pages.allocated,
                     pages.total,
                     m_dataset->cache()->isJIT() ? GREEN_BOLD_S "+" : RED_BOLD_S "-",
                     Chrono::steadyMSecs() - ts
                     );
        }
        else {
            // 如果数据集不存在，则打印警告信息
            LOG_WARN(CLEAR "%s" YELLOW_BOLD_S "failed to allocate RandomX dataset, switching to slow mode" BLACK_BOLD(" (%" PRIu64 " ms)"), Tags::randomx(), Chrono::steadyMSecs() - ts);
        }
    }

    // 初始化成员变量
    bool m_ready         = false;
    RxDataset *m_dataset = nullptr;
    RxSeed m_seed;
};


} // namespace xmrig

// 构造函数，初始化私有成员指针
xmrig::RxBasicStorage::RxBasicStorage() :
    d_ptr(new RxBasicStoragePrivate())
{
}

// 析构函数，释放私有成员指针
xmrig::RxBasicStorage::~RxBasicStorage()
{
    delete d_ptr;
}

// 检查数据集是否已分配
bool xmrig::RxBasicStorage::isAllocated() const
{
    return d_ptr->dataset() && d_ptr->dataset()->cache() && d_ptr->dataset()->cache()->get();
}

// 获取巨大页面信息
xmrig::HugePagesInfo xmrig::RxBasicStorage::hugePages() const
{
    if (!d_ptr->dataset()) {
        return {};
    }

    return d_ptr->dataset()->hugePages();
}

// 获取数据集
xmrig::RxDataset *xmrig::RxBasicStorage::dataset(const Job &job, uint32_t) const
{
    if (!d_ptr->isReady(job)) {
        return nullptr;
    }

    return d_ptr->dataset();
}
// 初始化 RxBasicStorage 对象的函数，接受一些参数
void xmrig::RxBasicStorage::init(const RxSeed &seed, uint32_t threads, bool hugePages, bool oneGbPages, RxConfig::Mode mode, int priority)
{
    // 调用 RxBasicStorage 对象的 setSeed 方法，设置种子
    d_ptr->setSeed(seed);

    // 如果数据集不存在，并且无法创建数据集，则返回
    if (!d_ptr->dataset() && !d_ptr->createDataset(hugePages, oneGbPages, mode)) {
        return;
    }

    // 初始化数据集，设置线程数和优先级
    d_ptr->initDataset(threads, priority);
}
```