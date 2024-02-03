# `xmrig\src\crypto\common\VirtualMemory.h`

```cpp
/* XMRig
 * 版权所有（c）2018-2020 tevador     <tevador@gmail.com>
 * 版权所有（c）2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有（c）2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改它
 * 根据由自由软件基金会发布的GNU通用公共许可证的条款，
 * 无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是希望它有用的分发，
 * 但没有任何保证；甚至没有暗示的保证
 * 商品性或适用于特定目的。有关更多详细信息，请参见
 * GNU通用公共许可证。
 *
 * 您应该已经收到了GNU通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_VIRTUALMEMORY_H
#define XMRIG_VIRTUALMEMORY_H


#include "base/tools/Object.h"
#include "crypto/common/HugePagesInfo.h"


#include <bitset>
#include <cstddef>
#include <cstdint>
#include <utility>


namespace xmrig {


class VirtualMemory
{
public:
    XMRIG_DISABLE_COPY_MOVE_DEFAULT(VirtualMemory)

    // 默认的巨大页面大小为2MB
    constexpr static size_t kDefaultHugePageSize    = 2U * 1024U * 1024U;
    // 1GB的大小
    constexpr static size_t kOneGiB                 = 1024U * 1024U * 1024U;

    // 构造函数，初始化虚拟内存
    VirtualMemory(size_t size, bool hugePages, bool oneGbPages, bool usePool, uint32_t node = 0, size_t alignSize = 64);
    // 析构函数，释放虚拟内存
    ~VirtualMemory();

    // 返回是否使用了巨大页面
    inline bool isHugePages() const                                 { return m_flags.test(FLAG_HUGEPAGES); }
    // 返回是否使用了1GB页面
    inline bool isOneGbPages() const                                { return m_flags.test(FLAG_1GB_PAGES); }
    // 返回虚拟内存的大小
    inline size_t size() const                                      { return m_size; }
    // 返回虚拟内存的容量
    inline size_t capacity() const                                  { return m_capacity; }
    // 返回虚拟内存的原始数据指针
    inline uint8_t *raw() const                                     { return m_scratchpad; }
    // 返回指向 scratchpad 的指针
    inline uint8_t *scratchpad() const                              { return m_scratchpad; }

    // 刷新指令缓存
    inline static void flushInstructionCache(void *p1, void *p2)    { flushInstructionCache(p1, static_cast<uint8_t*>(p2) - static_cast<uint8_t*>(p1)); }

    // 返回 HugePagesInfo 对象
    HugePagesInfo hugePages() const;

    // 检查是否支持大页
    static bool isHugepagesAvailable();
    // 检查是否支持 1GB 大页
    static bool isOneGbPagesAvailable();
    // 保护内存区域可读写
    static bool protectRW(void *p, size_t size);
    // 保护内存区域可读写执行
    static bool protectRWX(void *p, size_t size);
    // 保护内存区域可读执行
    static bool protectRX(void *p, size_t size);
    // 将内存绑定到 NUMA 节点
    static uint32_t bindToNUMANode(int64_t affinity);
    // 分配可执行内存
    static void *allocateExecutableMemory(size_t size, bool hugePages);
    // 分配大页内存
    static void *allocateLargePagesMemory(size_t size);
    // 分配 1GB 大页内存
    static void *allocateOneGbPagesMemory(size_t size);
    // 销毁内存分配器
    static void destroy();
    // 刷新指令缓存
    static void flushInstructionCache(void *p, size_t size);
    // 释放大页内存
    static void freeLargePagesMemory(void *p, size_t size);
    // 初始化内存分配器
    static void init(size_t poolSize, size_t hugePageSize);

    // 对齐到指定大小
    static inline constexpr size_t align(size_t pos, size_t align = kDefaultHugePageSize)   { return ((pos - 1) / align + 1) * align; }
    // 对齐到大页大小
    static inline size_t alignToHugePageSize(size_t pos)                                    { return align(pos, hugePageSize()); }
    // 返回大页大小
    static inline size_t hugePageSize()                                                     { return m_hugePageSize; }
// 私有成员变量和枚举类型定义
private:
    enum Flags {
        FLAG_HUGEPAGES, // 标志：启用大页
        FLAG_1GB_PAGES, // 标志：启用1GB页
        FLAG_LOCK, // 标志：锁定内存
        FLAG_EXTERNAL, // 标志：外部内存
        FLAG_MAX // 标志的最大数量
    };

    // 静态方法：初始化操作系统相关设置
    static void osInit(size_t hugePageSize);

    // 分配大页内存的方法
    bool allocateLargePagesMemory();
    // 分配1GB页内存的方法
    bool allocateOneGbPagesMemory();
    // 释放大页内存的方法
    void freeLargePagesMemory();

    // 静态成员变量：大页的大小
    static size_t m_hugePageSize;

    // 常量成员变量：内存大小、节点编号、容量
    const size_t m_size;
    const uint32_t m_node;
    size_t m_capacity;

    // 位集合：标志位
    std::bitset<FLAG_MAX> m_flags;

    // 指针：临时内存空间
    uint8_t *m_scratchpad = nullptr;
};
```