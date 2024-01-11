# `xmrig\src\crypto\kawpow\KPCache.cpp`

```
/* XMRig
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款
 *   它，无论是许可证的第 3 版还是
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。请参阅
 *   GNU 通用公共许可证以获取更多详细信息。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include <cinttypes>
#include <algorithm>
#include <thread>

#include "crypto/kawpow/KPCache.h"
#include "3rdparty/libethash/data_sizes.h"
#include "3rdparty/libethash/ethash_internal.h"
#include "3rdparty/libethash/ethash.h"
#include "base/io/log/Log.h"
#include "base/io/log/Tags.h"
#include "base/tools/Chrono.h"
#include "crypto/common/VirtualMemory.h"


namespace xmrig {


std::mutex KPCache::s_cacheMutex;
KPCache KPCache::s_cache;


KPCache::KPCache()
{
}


KPCache::~KPCache()
{
    delete m_memory;
}


bool KPCache::init(uint32_t epoch)
{
    // 如果 epoch 大于等于 cache_sizes 数组的大小，则返回 false
    if (epoch >= sizeof(cache_sizes) / sizeof(cache_sizes[0])) {
        return false;
    }

    // 如果当前缓存的 epoch 与传入的 epoch 相同，则返回 true
    if (m_epoch == epoch) {
        return true;
    }

    // 获取当前时间的毫秒数
    const uint64_t start_ms = Chrono::steadyMSecs();

    // 获取当前 epoch 对应的缓存大小
    const size_t size = cache_sizes[epoch];
    // 如果内存为空或内存大小小于当前缓存大小，则重新分配内存
    if (!m_memory || m_memory->size() < size) {
        delete m_memory;
        m_memory = new VirtualMemory(size, false, false, false);
    }

    // 获取当前 epoch 对应的种子哈希
    const ethash_h256_t seedhash = ethash_get_seedhash(epoch);
    // 计算缓存节点
    ethash_compute_cache_nodes(m_memory->raw(), size, &seedhash);

    // 创建 ethash_light 对象并设置缓存和缓存大小
    ethash_light cache;
    cache.cache = m_memory->raw();
    cache.cache_size = size;
    # 计算缓存中父节点的数量
    cache.num_parent_nodes = cache.cache_size / sizeof(node);
    # 计算快速取模所需的数据
    calculate_fast_mod_data(cache.num_parent_nodes, cache.reciprocal, cache.increment, cache.shift);
    
    # 计算缓存节点的数量
    const uint64_t cache_nodes = (size + sizeof(node) * 4 - 1) / sizeof(node);
    # 调整 DAG 缓存的大小
    m_DAGCache.resize(cache_nodes * (sizeof(node) / sizeof(uint32_t)));
    
    # 初始化 DAG 缓存
    {
        # 获取可用的线程数
        const uint64_t n = std::max(std::thread::hardware_concurrency(), 1U);
    
        # 创建线程数组
        std::vector<std::thread> threads;
        threads.reserve(n);
    
        # 根据线程数分配任务
        for (uint64_t i = 0; i < n; ++i) {
            const uint32_t a = (cache_nodes * i) / n;
            const uint32_t b = (cache_nodes * (i + 1)) / n;
    
            # 创建线程并执行任务
            threads.emplace_back([this, a, b, &cache]() {
                uint32_t j = a;
                # 计算 DAG 缓存的每个节点
                for (; j + 4 <= b; j += 4) ethash_calculate_dag_item4_opt(((node*)m_DAGCache.data()) + j, j, num_dataset_parents, &cache);
                for (; j < b; ++j) ethash_calculate_dag_item_opt(((node*)m_DAGCache.data()) + j, j, num_dataset_parents, &cache);
            });
        }
    
        # 等待所有线程执行完毕
        for (auto& t : threads) {
            t.join();
        }
    }
    
    # 设置缓存的大小和当前的 epoch
    m_size = size;
    m_epoch = epoch;
    
    # 输出计算缓存所花费的时间
    LOG_INFO("%s " YELLOW("KawPow") " light cache for epoch " WHITE_BOLD("%u") " calculated " BLACK_BOLD("(%" PRIu64 "ms)"), Tags::miner(), epoch, Chrono::steadyMSecs() - start_ms);
    
    # 返回计算结果
    return true;
// KPCache 类的 data() 方法，返回缓存数据的指针
void* KPCache::data() const
{
    // 如果 m_memory 不为空，则返回 m_memory 的原始指针，否则返回空指针
    return m_memory ? m_memory->raw() : nullptr;
}

// clz() 函数，计算一个无符号整数的前导零位数
static inline uint32_t clz(uint32_t a)
{
    // 如果是在 Microsoft Visual C++ 编译器下，使用 _BitScanReverse() 函数计算前导零位数
    #ifdef _MSC_VER
        unsigned long index;
        _BitScanReverse(&index, a);
        return 31 - index;
    // 否则使用 __builtin_clz() 函数计算前导零位数
    #else
        return __builtin_clz(a);
    #endif
}

// KPCache 类的 cache_size() 方法，根据 epoch 返回缓存大小
uint64_t KPCache::cache_size(uint32_t epoch)
{
    // 如果 epoch 大于等于 cache_sizes 数组的长度，则返回 0
    if (epoch >= sizeof(cache_sizes) / sizeof(cache_sizes[0])) {
        return 0;
    }

    // 返回 cache_sizes 数组中对应 epoch 的缓存大小
    return cache_sizes[epoch];
}

// KPCache 类的 dag_size() 方法，根据 epoch 返回 DAG 大小
uint64_t KPCache::dag_size(uint32_t epoch)
{
    // 如果 epoch 大于等于 dag_sizes 数组的长度，则返回 0
    if (epoch >= sizeof(dag_sizes) / sizeof(dag_sizes[0])) {
        return 0;
    }

    // 返回 dag_sizes 数组中对应 epoch 的 DAG 大小
    return dag_sizes[epoch];
}

// KPCache 类的 calculate_fast_mod_data() 方法，计算快速取模所需的数据
void KPCache::calculate_fast_mod_data(uint32_t divisor, uint32_t& reciprocal, uint32_t& increment, uint32_t& shift)
{
    // 如果除数是 2 的幂次方，则直接计算 reciprocal、increment 和 shift
    if ((divisor & (divisor - 1)) == 0) {
        reciprocal = 1;
        increment = 0;
        shift = 31U - clz(divisor);
    }
    // 否则根据除数计算 reciprocal、increment 和 shift
    else {
        shift = 63U - clz(divisor);
        const uint64_t N = 1ULL << shift;
        const uint64_t q = N / divisor;
        const uint64_t r = N - q * divisor;
        // 如果 r * 2 小于除数，则 reciprocal 为 q，increment 为 1
        if (r * 2 < divisor)
        {
            reciprocal = static_cast<uint32_t>(q);
            increment = 1;
        }
        // 否则 reciprocal 为 q+1，increment 为 0
        else
        {
            reciprocal = static_cast<uint32_t>(q + 1);
            increment = 0;
        }
    }
}

// 命名空间 xmrig 的结束标记
} // namespace xmrig
```