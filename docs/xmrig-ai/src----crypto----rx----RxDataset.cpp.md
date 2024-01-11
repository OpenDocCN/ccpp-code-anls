# `xmrig\src\crypto\rx\RxDataset.cpp`

```
/* XMRig
 * 版权所有（c）2018-2019 tevador     <tevador@gmail.com>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以根据GNU通用公共许可证的条款重新分发或修改它，
 *   由自由软件基金会发布，无论是许可证的第3版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。有关更多详细信息，请参见
 *   GNU通用公共许可证。
 *
 *   您应该已经收到了GNU通用公共许可证的副本
 *   与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "crypto/rx/RxDataset.h"
#include "backend/cpu/Cpu.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/kernel/Platform.h"
#include "crypto/common/VirtualMemory.h"
#include "crypto/randomx/randomx.h"
#include "crypto/rx/RxAlgo.h"
#include "crypto/rx/RxCache.h"


#include <thread>
#include <uv.h>


namespace xmrig {


static void init_dataset_wrapper(randomx_dataset *dataset, randomx_cache *cache, uint32_t startItem, uint32_t itemCount, int priority)
{
    Platform::setThreadPriority(priority);

    if (Cpu::info()->hasAVX2() && (itemCount % 5)) {
        randomx_init_dataset(dataset, cache, startItem, itemCount - (itemCount % 5));
        randomx_init_dataset(dataset, cache, startItem + itemCount - 5, 5);
    }
    else {
        randomx_init_dataset(dataset, cache, startItem, itemCount);
    }
}


} // namespace xmrig


xmrig::RxDataset::RxDataset(bool hugePages, bool oneGbPages, bool cache, RxConfig::Mode mode, uint32_t node) :
    m_mode(mode),
    m_node(node)
{
    allocate(hugePages, oneGbPages);
    # 如果系统支持1GB页面，创建一个RxCache对象并将其指针存储在m_cache中
    if (isOneGbPages()) {
        m_cache = new RxCache(m_memory->raw() + VirtualMemory::align(maxSize()));

        return;
    }

    # 如果cache存在，根据hugePages和node创建一个RxCache对象并将其指针存储在m_cache中
    if (cache) {
        m_cache = new RxCache(hugePages, node);
    }
# RxDataset 类的构造函数，初始化成员变量 m_node 和 m_cache
xmrig::RxDataset::RxDataset(RxCache *cache) :
    m_node(0),
    m_cache(cache)
{
}

# RxDataset 类的析构函数，释放 m_dataset、m_cache 和 m_memory
xmrig::RxDataset::~RxDataset()
{
    # 释放 m_dataset
    randomx_release_dataset(m_dataset);
    # 释放 m_cache
    delete m_cache;
    # 释放 m_memory
    delete m_memory;
}

# 初始化数据集，根据 seed、numThreads 和 priority 进行初始化
bool xmrig::RxDataset::init(const Buffer &seed, uint32_t numThreads, int priority)
{
    # 如果 m_cache 为空，返回 false
    if (!m_cache || !m_cache->get()) {
        return false;
    }
    # 初始化 m_cache
    m_cache->init(seed);
    # 如果数据集为空，返回 true
    if (!get()) {
        return true;
    }
    # 获取数据集的项数
    const uint64_t datasetItemCount = randomx_dataset_item_count();
    # 如果线程数大于 1
    if (numThreads > 1) {
        # 创建线程数组
        std::vector<std::thread> threads;
        threads.reserve(numThreads);
        # 遍历线程数，初始化数据集
        for (uint64_t i = 0; i < numThreads; ++i) {
            const uint32_t a = (datasetItemCount * i) / numThreads;
            const uint32_t b = (datasetItemCount * (i + 1)) / numThreads;
            threads.emplace_back(init_dataset_wrapper, m_dataset, m_cache->get(), a, b - a, priority);
        }
        # 等待线程结束
        for (uint32_t i = 0; i < numThreads; ++i) {
            threads[i].join();
        }
    }
    else {
        # 初始化数据集
        init_dataset_wrapper(m_dataset, m_cache->get(), 0, datasetItemCount, priority);
    }
    # 返回 true
    return true;
}

# 返回 m_memory 是否为 HugePages
bool xmrig::RxDataset::isHugePages() const
{
    return m_memory && m_memory->isHugePages();
}

# 返回 m_memory 是否为 OneGbPages
bool xmrig::RxDataset::isOneGbPages() const
{
    return m_memory && m_memory->isOneGbPages();
}

# 返回 HugePagesInfo，如果 cache 为 true，则加上 m_cache 的 HugePagesInfo
xmrig::HugePagesInfo xmrig::RxDataset::hugePages(bool cache) const
{
    auto pages = m_memory ? m_memory->hugePages() : HugePagesInfo();
    if (cache && m_cache) {
        pages += m_cache->hugePages();
    }
    return pages;
}

# 返回数据集的大小，如果 cache 为 true，则加上 m_cache 的 maxSize
size_t xmrig::RxDataset::size(bool cache) const
{
    size_t size = 0;
    if (m_dataset) {
        size += maxSize();
    }
    if (cache && m_cache) {
        size += RxCache::maxSize();
    }
    return size;
}

# 尝试分配 Scratchpad 内存，如果失败返回 nullptr
uint8_t *xmrig::RxDataset::tryAllocateScrathpad()
{
    auto p = reinterpret_cast<uint8_t *>(raw());
    if (!p) {
        return nullptr;
    }
}
    # 获取当前的scratchpadOffset值，并将其增加RANDOMX_SCRATCHPAD_L3_MAX_SIZE
    const size_t offset = m_scratchpadOffset.fetch_add(RANDOMX_SCRATCHPAD_L3_MAX_SIZE);
    # 如果offset加上RANDOMX_SCRATCHPAD_L3_MAX_SIZE超过了scratchpadLimit的值，则返回空指针
    if (offset + RANDOMX_SCRATCHPAD_L3_MAX_SIZE > m_scratchpadLimit) {
        return nullptr;
    }
    # 返回p加上offset后的指针
    return p + offset;
# 返回数据集的原始指针，如果数据集存在则返回其内存地址，否则返回空指针
void *xmrig::RxDataset::raw() const
{
    return m_dataset ? randomx_get_dataset_memory(m_dataset) : nullptr;
}

# 设置数据集的原始指针
void xmrig::RxDataset::setRaw(const void *raw)
{
    # 如果数据集不存在，则直接返回
    if (!m_dataset) {
        return;
    }
    
    # 获取数据集的最大大小
    volatile size_t N = maxSize();
    # 将原始数据拷贝到数据集内存中
    memcpy(randomx_get_dataset_memory(m_dataset), raw, N);
}

# 分配数据集内存
void xmrig::RxDataset::allocate(bool hugePages, bool oneGbPages)
{
    # 如果模式为轻量级模式，则输出错误信息并返回
    if (m_mode == RxConfig::LightMode) {
        LOG_ERR(CLEAR "%s" RED_BOLD_S "fast RandomX mode disabled by config", Tags::randomx());
        return;
    }
    
    # 如果模式为自动模式且系统内存不足以容纳数据集和缓存，则输出错误信息并返回
    if (m_mode == RxConfig::AutoMode && uv_get_total_memory() < (maxSize() + RxCache::maxSize())) {
        LOG_ERR(CLEAR "%s" RED_BOLD_S "not enough memory for RandomX dataset", Tags::randomx());
        return;
    }
    
    # 根据数据集的最大大小和页面类型分配虚拟内存
    m_memory  = new VirtualMemory(maxSize(), hugePages, oneGbPages, false, m_node);
    
    # 如果使用了1GB页面，则设置缓存偏移和限制
    if (m_memory->isOneGbPages()) {
        m_scratchpadOffset = maxSize() + RANDOMX_CACHE_MAX_SIZE;
        m_scratchpadLimit = m_memory->capacity();
    }
    
    # 创建数据集并将其内存地址赋给m_dataset
    m_dataset = randomx_create_dataset(m_memory->raw());

#   ifdef XMRIG_OS_LINUX
    # 如果使用了1GB页面但未成功分配数据集，则输出错误信息
    if (oneGbPages && !isOneGbPages()) {
        LOG_ERR(CLEAR "%s" RED_BOLD_S "failed to allocate RandomX dataset using 1GB pages", Tags::randomx());
    }
#   endif
}
```