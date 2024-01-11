# `xmrig\src\crypto\kawpow\KPCache.h`

```
// 定义了 XMRig_KP_Cache_H 的宏，用于防止头文件被重复包含
#ifndef XMRIG_KP_CACHE_H
#define XMRIG_KP_CACHE_H

// 包含 Object.h 头文件和一些标准库头文件
#include "base/tools/Object.h"
#include <mutex>
#include <vector>

// 声明 xmrig 命名空间
namespace xmrig
{

// 声明 VirtualMemory 类
class VirtualMemory;

// 声明 KPCache 类
class KPCache
{
public:
    // 定义静态成员变量
    static constexpr size_t l1_cache_size = 16 * 1024;
    static constexpr size_t l1_cache_num_items = l1_cache_size / sizeof(uint32_t);
    static constexpr uint32_t num_dataset_parents = 512;

    // 禁用拷贝和移动构造函数
    XMRIG_DISABLE_COPY_MOVE(KPCache)

    // 声明构造函数和析构函数
    KPCache();
    ~KPCache();

    // 初始化函数
    bool init(uint32_t epoch);

    // 返回数据指针和大小的函数
    void* data() const;
    size_t size() const { return m_size; }
    uint32_t epoch() const { return m_epoch; }

    // 返回 l1_cache 的函数
    const uint32_t* l1_cache() const { return m_DAGCache.data(); }

    // 计算缓存大小和 DAG 大小的静态函数
    static uint64_t cache_size(uint32_t epoch);
    static uint64_t dag_size(uint32_t epoch);

    // 计算快速取模的静态函数
    static void calculate_fast_mod_data(uint32_t divisor, uint32_t &reciprocal, uint32_t &increment, uint32_t& shift);

    // 静态互斥量和缓存对象
    static std::mutex s_cacheMutex;
    static KPCache s_cache;

private:
    // 声明私有成员变量
    VirtualMemory* m_memory = nullptr;
    size_t m_size = 0;
    uint32_t m_epoch = 0xFFFFFFFFUL;
    std::vector<uint32_t> m_DAGCache;
};

} /* namespace xmrig */

// 结束宏定义
#endif /* XMRIG_KP_CACHE_H */
```