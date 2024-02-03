# `xmrig\src\crypto\common\NUMAMemoryPool.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2018-2019 tevador     <tevador@gmail.com>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款，由
 * 自由软件基金会发布的版本 3 或
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "crypto/common/NUMAMemoryPool.h"
#include "crypto/common/VirtualMemory.h"
#include "backend/cpu/Cpu.h"
#include "crypto/common/MemoryPool.h"


#include <algorithm>


// 实现 NUMAMemoryPool 类的构造函数
xmrig::NUMAMemoryPool::NUMAMemoryPool(size_t size, bool hugePages) :
    m_hugePages(hugePages), // 初始化 m_hugePages 成员变量
    m_nodeSize(std::max<size_t>(size / Cpu::info()->nodes(), 1)), // 初始化 m_nodeSize 成员变量
    m_size(size) // 初始化 m_size 成员变量
{
}


// 实现 NUMAMemoryPool 类的析构函数
xmrig::NUMAMemoryPool::~NUMAMemoryPool()
{
    // 遍历 m_map 容器，释放内存
    for (auto kv : m_map) {
        delete kv.second;
    }
}


// 判断是否使用大页内存
bool xmrig::NUMAMemoryPool::isHugePages(uint32_t node) const
{
    // 如果 m_size 为 0，则返回 false
    if (!m_size) {
        return false;
    }

    // 调用 getOrCreate 方法获取或创建 NUMAMemoryPool 对象，并判断是否使用大页内存
    return getOrCreate(node)->isHugePages(node);
}
// 从 NUMAMemoryPool 中获取指定大小和节点的内存块
uint8_t *xmrig::NUMAMemoryPool::get(size_t size, uint32_t node)
{
    // 如果内存池大小为0，则返回空指针
    if (!m_size) {
        return nullptr;
    }

    // 获取或创建指定节点的内存池，并从中获取指定大小的内存块
    return getOrCreate(node)->get(size, node);
}


// 释放指定节点的内存池
void xmrig::NUMAMemoryPool::release(uint32_t node)
{
    // 获取指定节点的内存池
    const auto pool = get(node);
    // 如果内存池存在，则释放指定节点的内存块
    if (pool) {
        pool->release(node);
    }
}


// 获取指定节点的内存池
xmrig::IMemoryPool *xmrig::NUMAMemoryPool::get(uint32_t node) const
{
    // 如果节点存在于映射中，则返回对应的内存池，否则返回空指针
    return m_map.count(node) ? m_map.at(node) : nullptr;
}


// 获取或创建指定节点的内存池
xmrig::IMemoryPool *xmrig::NUMAMemoryPool::getOrCreate(uint32_t node) const
{
    // 获取指定节点的内存池
    auto pool = get(node);
    // 如果内存池不存在，则创建新的内存池，并将其插入映射中
    if (!pool) {
        pool = new MemoryPool(m_nodeSize, m_hugePages, node);
        m_map.insert({ node, pool });
    }

    // 返回获取或创建的内存池
    return pool;
}
```