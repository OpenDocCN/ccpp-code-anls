# `xmrig\src\crypto\common\MemoryPool.cpp`

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
 * 本程序是希望它有用：
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。请参阅
 * GNU 通用公共许可证以获取更多详细信息。
 *
 * 您应该已经收到 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */


#include "crypto/common/MemoryPool.h"
#include "crypto/common/VirtualMemory.h"


#include <cassert>


namespace xmrig {


// 定义页面大小为2MB
constexpr size_t pageSize = 2 * 1024 * 1024;


} // namespace xmrig


// 构造函数，初始化内存池
xmrig::MemoryPool::MemoryPool(size_t size, bool hugePages, uint32_t node)
{
    // 如果大小为0，则返回
    if (!size) {
        return;
    }

    // 定义对齐大小为2^24
    constexpr size_t alignment = 1 << 24;

    // 创建虚拟内存对象，大小为size * pageSize + alignment
    m_memory = new VirtualMemory(size * pageSize + alignment, hugePages, false, false, node);

    // 计算对齐偏移量
    m_alignOffset = (alignment - (((size_t)m_memory->scratchpad()) % alignment)) % alignment;
}


// 析构函数，释放内存池
xmrig::MemoryPool::~MemoryPool()
{
    delete m_memory;
}
# 检查内存池是否使用了大页
bool xmrig::MemoryPool::isHugePages(uint32_t) const
{
    # 返回内存是否使用了大页
    return m_memory && m_memory->isHugePages();
}


# 从内存池中获取指定大小的内存
uint8_t *xmrig::MemoryPool::get(size_t size, uint32_t)
{
    # 断言内存大小是否是页大小的整数倍
    assert(!(size % pageSize));

    # 如果内存为空，或者剩余内存不足以分配指定大小的内存，则返回空指针
    if (!m_memory || (m_memory->size() - m_offset - m_alignOffset) < size) {
        return nullptr;
    }

    # 计算要返回的内存地址
    uint8_t *out = m_memory->scratchpad() + m_alignOffset + m_offset;

    # 更新内存偏移和引用计数
    m_offset += size;
    ++m_refs;

    # 返回分配的内存地址
    return out;
}


# 释放内存池中的内存
void xmrig::MemoryPool::release(uint32_t)
{
    # 断言引用计数大于0
    assert(m_refs > 0);

    # 如果引用计数大于0，则减少引用计数
    if (m_refs > 0) {
        --m_refs;
    }

    # 如果引用计数为0，则重置偏移量
    if (m_refs == 0) {
        m_offset = 0;
    }
}
```