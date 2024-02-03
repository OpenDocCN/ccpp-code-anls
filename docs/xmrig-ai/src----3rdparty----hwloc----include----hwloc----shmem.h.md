# `xmrig\src\3rdparty\hwloc\include\hwloc\shmem.h`

```cpp
/*
 * 版权声明，版权归 Inria 公司所有，保留所有权利
 * 请参阅顶层目录中的 COPYING 文件
 */

/** \file
 * \brief 进程之间共享拓扑结构
 */

#ifndef HWLOC_SHMEM_H
#define HWLOC_SHMEM_H

#include "hwloc.h"

#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif


/** \defgroup hwlocality_shmem 进程之间共享拓扑结构
 *
 * 这些函数用于通过将拓扑结构复制到文件支持的共享内存缓冲区中，在进程之间共享拓扑结构。
 *
 * 主进程首先必须通过 hwloc_shmem_topology_get_length() 获取存储此拓扑结构所需的共享内存大小。
 *
 * 然后，它必须找到一个大小为该大小的虚拟内存区域，该区域在所有进程中都可用（所有进程中的虚拟地址相同）。
 * 在 Linux 上，可以通过比较每个进程中 /proc/\<pid\>/maps 中找到的空洞来实现。
 *
 * 找到后，它必须打开一个目标文件来存储缓冲区，并将其与上面获取的虚拟内存地址和长度一起传递给 hwloc_shmem_topology_write()。
 *
 * 其他进程可以通过打开相同的文件，并将其与具有完全相同虚拟内存地址和长度的 hwloc_shmem_topology_adopt() 一起采用此共享拓扑结构。
 *
 * @{
 */

/** \brief 获取存储拓扑结构所需的共享内存长度。
 *
 * 此长度（以字节为单位）必须在后续的 hwloc_shmem_topology_write() 和 hwloc_shmem_topology_adopt() 中使用。
 *
 * \note 标志 \p flags 目前未使用，必须为 0。
 */
HWLOC_DECLSPEC int hwloc_shmem_topology_get_length(hwloc_topology_t topology,
                           size_t *lengthp,
                           unsigned long flags);
/**
 * \brief 将拓扑结构复制到共享内存文件中
 *
 * 通过在虚拟内存中临时映射一个文件，并在其中分配副本来复制拓扑结构 \p topology。
 *
 * 文件描述符 \p fd 指向的文件段，从偏移量 \p fileoffset 开始，长度为 \p length（以字节为单位），
 * 将在复制期间临时映射到虚拟地址 \p mmap_address。
 *
 * 映射长度 \p length 必须先前通过 hwloc_shmem_topology_get_length() 获取，
 * 并且在此期间拓扑结构不能被修改。
 *
 * \note 标志 \p flags 目前未使用，必须为 0。
 *
 * \note 对象的用户数据指针会被复制，但指向的缓冲区不会被复制。但是调用者也可以手动在共享内存中分配它以共享它。
 *
 * \return 如果由 \p mmap_address 和 \p length 定义的虚拟内存映射在进程中不可用，则返回 -1，同时将 errno 设置为 EBUSY。
 * \return 如果 \p fileoffset、\p mmap_address 或 \p length 不是页面对齐的，则返回 -1，同时将 errno 设置为 EINVAL。
 */
HWLOC_DECLSPEC int hwloc_shmem_topology_write(hwloc_topology_t topology,
                          int fd, hwloc_uint64_t fileoffset,
                          void *mmap_address, size_t length,
                          unsigned long flags);
/**
 * 采用存储在文件中的共享内存拓扑。
 *
 * 在虚拟内存中映射文件，并采用先前使用hwloc_shmem_topology_write()存储在其中的拓扑。
 *
 * 返回的采用的拓扑在topologyp中，可以像任何拓扑一样使用。并且必须像通常一样使用hwloc_topology_destroy()来销毁。
 *
 * 但是这个拓扑是只读的。
 * 例如，它不能使用hwloc_topology_restrict()进行修改，对象的用户数据指针也不能被更改。
 *
 * 文件描述符fd指向的文件段，从偏移量fileoffset开始，长度为length（以字节为单位），将被映射到虚拟地址mmap_address。
 *
 * 文件描述符fd指向的文件，偏移量fileoffset，请求的映射虚拟地址mmap_address和长度length必须与之前传递给hwloc_shmem_topology_write()的相同。
 *
 * \note 标志flags当前未使用，必须为0。
 *
 * \note 除非创建共享拓扑的进程还将用户数据指针缓冲区放置在共享内存中，否则不应使用对象用户数据指针。
 *
 * \note 此函数负责调用hwloc_topology_abi_check()。
 *
 * \return -1，并将errno设置为EBUSY，如果由mmap_address和length定义的虚拟内存映射在进程中不可用。
 *
 * \return -1，并将errno设置为EINVAL，如果fileoffset、mmap_address或length不是页面对齐的，或者与之前传递给hwloc_shmem_topology_write()的不匹配。
 *
 * \return -1，并将errno设置为EINVAL，如果拓扑结构的布局在写入进程和采用进程之间不同。
 */
HWLOC_DECLSPEC int hwloc_shmem_topology_adopt(hwloc_topology_t *topologyp,
                          int fd, hwloc_uint64_t fileoffset,
                          void *mmap_address, size_t length,
                          unsigned long flags);
/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_SHMEM_H */
```