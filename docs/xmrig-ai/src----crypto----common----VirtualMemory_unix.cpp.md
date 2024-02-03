# `xmrig\src\crypto\common\VirtualMemory_unix.cpp`

```cpp
/* XMRig
 * 版权所有 (c) 2018-2020 tevador     <tevador@gmail.com>
 * 版权所有 (c) 2018-2023 SChernykh   <https://github.com/SChernykh>
 * 版权所有 (c) 2016-2023 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改它
 *   根据 GNU 通用公共许可证的条款发布，由
 *   自由软件基金会发布的版本 3 或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用，
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "crypto/common/VirtualMemory.h"
#include "backend/cpu/Cpu.h"
#include "crypto/common/portable/mm_malloc.h"


#include <cmath>
#include <cstdlib>
#include <fstream>
#include <sys/mman.h>


#ifdef XMRIG_OS_APPLE
#   include <libkern/OSCacheControl.h>
#   include <mach/vm_statistics.h>
#   include <pthread.h>
#   include <TargetConditionals.h>
#   ifdef XMRIG_ARM
#       define MEXTRA MAP_JIT
#   else
#       define MEXTRA 0
#   endif
#else
#   define MEXTRA 0
#endif


#ifdef XMRIG_OS_LINUX
#   include "crypto/common/LinuxMemory.h"
#endif


#ifndef MAP_HUGE_SHIFT
#   define MAP_HUGE_SHIFT 26
#endif


#ifndef MAP_HUGE_MASK
#   define MAP_HUGE_MASK 0x3f
#endif

#ifdef XMRIG_OS_FREEBSD
#   ifndef MAP_ALIGNED_SUPER
#       define MAP_ALIGNED_SUPER 0
#   endif
#   ifndef MAP_PREFAULT_READ
#       define MAP_PREFAULT_READ 0
#   endif
#endif


#ifdef XMRIG_SECURE_JIT
#   define SECURE_PROT_EXEC 0
#else
#   define SECURE_PROT_EXEC PROT_EXEC
#endif


#if defined(XMRIG_OS_LINUX) || (!defined(XMRIG_OS_APPLE) && !defined(XMRIG_OS_FREEBSD))
static inline int hugePagesFlag(size_t size)
    # 将 size 取对数并转换为整数，然后与 MAP_HUGE_MASK 按位与，再左移 MAP_HUGE_SHIFT 位
    return (static_cast<int>(log2(size)) & MAP_HUGE_MASK) << MAP_HUGE_SHIFT;
}
#endif


bool xmrig::VirtualMemory::isHugepagesAvailable()
{
#   ifdef XMRIG_OS_LINUX
    // 检查是否存在大页内存，通过检查两个文件是否存在
    return std::ifstream("/proc/sys/vm/nr_hugepages").good() || std::ifstream("/sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages").good();
#   elif defined(XMRIG_OS_MACOS) && defined(XMRIG_ARM)
    // 在 macOS 上不支持大页内存
    return false;
#   else
    // 其他系统默认支持大页内存
    return true;
#   endif
}


bool xmrig::VirtualMemory::isOneGbPagesAvailable()
{
#   ifdef XMRIG_OS_LINUX
    // 检查是否存在 1GB 页内存，通过检查 CPU 信息
    return Cpu::info()->hasOneGbPages();
#   else
    // 其他系统不支持 1GB 页内存
    return false;
#   endif
}


bool xmrig::VirtualMemory::protectRW(void *p, size_t size)
{
#   if defined(XMRIG_OS_APPLE) && defined(XMRIG_ARM)
    // 在苹果 ARM 架构上关闭 JIT 写保护
    pthread_jit_write_protect_np(false);
    return true;
#   else
    // 其他系统调用 mprotect 函数设置内存区域可读可写
    return mprotect(p, size, PROT_READ | PROT_WRITE) == 0;
#   endif
}


bool xmrig::VirtualMemory::protectRWX(void *p, size_t size)
{
    // 调用 mprotect 函数设置内存区域可读可写可执行
    return mprotect(p, size, PROT_READ | PROT_WRITE | PROT_EXEC) == 0;
}


bool xmrig::VirtualMemory::protectRX(void *p, size_t size)
{
    bool result = true;

#   if defined(XMRIG_OS_APPLE) && defined(XMRIG_ARM)
    // 在苹果 ARM 架构上开启 JIT 写保护
    pthread_jit_write_protect_np(true);
#   else
    // 其他系统调用 mprotect 函数设置内存区域可读可执行
    result = (mprotect(p, size, PROT_READ | PROT_EXEC) == 0);
#   endif

#   if defined(XMRIG_ARM)
    // 刷新指令缓存
    flushInstructionCache(p, size);
#   endif

    return result;
}


void *xmrig::VirtualMemory::allocateExecutableMemory(size_t size, bool hugePages)
{
#   if defined(XMRIG_OS_APPLE)
    // 在苹果系统上使用 mmap 函数分配可执行内存
    void *mem = mmap(0, size, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANON | MEXTRA, -1, 0);
#   ifdef XMRIG_ARM
    // 在 ARM 架构上关闭 JIT 写保护
    pthread_jit_write_protect_np(false);
#   endif
#   elif defined(XMRIG_OS_FREEBSD)
    void *mem = nullptr;

    if (hugePages) {
        // 在 FreeBSD 上使用 mmap 函数分配大页内存
        mem = mmap(0, size, PROT_READ | PROT_WRITE | SECURE_PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS | MAP_ALIGNED_SUPER | MAP_PREFAULT_READ, -1, 0);
    }

    if (!mem) {
        // 在 FreeBSD 上使用 mmap 函数分配普通内存
        mem = mmap(0, size, PROT_READ | PROT_WRITE | SECURE_PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    }

#   else
    // 其他系统暂未实现
    void *mem = nullptr;
    # 如果 hugePages 为真，则使用大页内存映射
    if (hugePages) {
        mem = mmap(0, align(size), PROT_READ | PROT_WRITE | SECURE_PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS | MAP_POPULATE | hugePagesFlag(hugePageSize()), -1, 0);
    }
    
    # 如果内存映射失败，则使用普通大小的内存映射
    if (!mem) {
        mem = mmap(0, size, PROT_READ | PROT_WRITE | SECURE_PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    }
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#   endif
#  
    # 如果存在临时内存区域
    if (m_scratchpad) {
        # 设置标志位，表示使用大页
        m_flags.set(FLAG_HUGEPAGES, true);

        # 提前告知内核，该内存区域将以随机顺序访问，并且需要预取
        madvise(m_scratchpad, m_size, MADV_RANDOM | MADV_WILLNEED);

        # 锁定内存区域，防止被交换出去
        if (mlock(m_scratchpad, m_size) == 0) {
            # 设置标志位，表示内存已被锁定
            m_flags.set(FLAG_LOCK, true);
        }

        # 返回操作成功
        return true;
    }

    # 返回操作失败
    return false;
# 释放大页内存
void xmrig::VirtualMemory::freeLargePagesMemory()
{
    # 如果标志位中包含锁定标志
    if (m_flags.test(FLAG_LOCK)) {
        # 解锁内存
        munlock(m_scratchpad, m_size);
    }
    # 释放大页内存
    freeLargePagesMemory(m_scratchpad, m_size);
}
```