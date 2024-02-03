# `xmrig\src\crypto\common\VirtualMemory.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2020 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2021 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2021 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以根据 GNU 通用公共许可证的条款重新分发或修改它，无论是许可证的第3版还是（在您的选择下）任何以后的版本。
 *
 * 本程序是基于希望它有用而分发的，但没有任何担保；甚至没有适销性或特定用途的隐含担保。有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */


#include "crypto/common/VirtualMemory.h"
#include "backend/cpu/Cpu.h"
#include "base/io/log/Log.h"
#include "crypto/common/MemoryPool.h"
#include "crypto/common/portable/mm_malloc.h"


#ifdef XMRIG_FEATURE_HWLOC
#   include "crypto/common/NUMAMemoryPool.h"
#endif


#include <cinttypes>
#include <mutex>


namespace xmrig {


// 设置默认的巨大页面大小
size_t VirtualMemory::m_hugePageSize    = VirtualMemory::kDefaultHugePageSize;
// 创建内存池指针，并初始化为nullptr
static IMemoryPool *pool                = nullptr;
// 创建互斥锁
static std::mutex mutex;


} // namespace xmrig


// 构造函数，初始化虚拟内存对象
xmrig::VirtualMemory::VirtualMemory(size_t size, bool hugePages, bool oneGbPages, bool usePool, uint32_t node, size_t alignSize) :
    // 将大小调整为巨大页面大小的倍数
    m_size(alignToHugePageSize(size)),
    // 设置节点
    m_node(node),
    // 设置容量为大小
    m_capacity(m_size)
{
    # 如果使用内存池
    if (usePool) {
        # 使用互斥锁保护临界区
        std::lock_guard<std::mutex> lock(mutex);
        # 如果使用大页并且内存池不支持大页并且成功分配大页内存，则返回
        if (hugePages && !pool->isHugePages(node) && allocateLargePagesMemory()) {
            return;
        }

        # 从内存池获取内存块
        m_scratchpad = pool->get(m_size, node);
        # 如果成功获取内存块
        if (m_scratchpad) {
            # 设置标志位，表示使用了大页
            m_flags.set(FLAG_HUGEPAGES, pool->isHugePages(node));
            # 设置标志位，表示使用了外部内存
            m_flags.set(FLAG_EXTERNAL,  true);
            # 返回
            return;
        }
    }

    # 如果使用了1GB页并且成功分配1GB页内存
    if (oneGbPages && allocateOneGbPagesMemory()) {
        # 设置容量为按1GB对齐的大小
        m_capacity = align(size, 1ULL << 30);
        # 返回
        return;
    }

    # 如果使用了大页并且成功分配大页内存
    if (hugePages && allocateLargePagesMemory()) {
        # 返回
        return;
    }

    # 使用_mm_malloc分配内存
    m_scratchpad = static_cast<uint8_t*>(_mm_malloc(m_size, alignSize));
# 虚拟内存类的析构函数
xmrig::VirtualMemory::~VirtualMemory()
{
    # 如果未分配内存，则直接返回
    if (!m_scratchpad) {
        return;
    }
    
    # 如果标记为外部内存，则使用互斥锁锁定，释放节点
    if (m_flags.test(FLAG_EXTERNAL)) {
        std::lock_guard<std::mutex> lock(mutex);
        pool->release(m_node);
    }
    # 如果是大页或者1GB页，则释放大页内存
    else if (isHugePages() || isOneGbPages()) {
        freeLargePagesMemory();
    }
    # 否则，释放普通内存
    else {
        _mm_free(m_scratchpad);
    }
}

# 返回巨大页信息
xmrig::HugePagesInfo xmrig::VirtualMemory::hugePages() const
{
    return { this };
}

# 如果未定义XMRIG_FEATURE_HWLOC，则绑定到NUMA节点的函数
#ifndef XMRIG_FEATURE_HWLOC
uint32_t xmrig::VirtualMemory::bindToNUMANode(int64_t)
{
    return 0;
}
#endif

# 销毁虚拟内存
void xmrig::VirtualMemory::destroy()
{
    delete pool;
}

# 初始化虚拟内存
void xmrig::VirtualMemory::init(size_t poolSize, size_t hugePageSize)
{
    # 如果池未初始化，则根据巨大页大小进行操作系统初始化
    if (!pool) {
        osInit(hugePageSize);
    }

#   ifdef XMRIG_FEATURE_HWLOC
    # 如果CPU节点数大于1，则根据节点数创建NUMA内存池，否则创建普通内存池
    if (Cpu::info()->nodes() > 1) {
        pool = new NUMAMemoryPool(align(poolSize, Cpu::info()->nodes()), hugePageSize > 0);
    } else
#   endif
    {
        pool = new MemoryPool(poolSize, hugePageSize > 0);
    }
}
```