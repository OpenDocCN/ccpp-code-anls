# `xmrig\src\3rdparty\hwloc\include\hwloc\linux.h`

```
/*
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2021 Inria。保留所有权利。
 * 版权所有 © 2009-2011 波尔多大学
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 与 Linux 交互的宏。
 *
 * 在使用一些低级 Linux 特性的应用程序中，可能希望包含此文件。
 */

#ifndef HWLOC_LINUX_H
#define HWLOC_LINUX_H

#include "hwloc.h"

#include <stdio.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_linux Linux-specific helpers
 *
 * 这包括用于操作 Linux 内核 cpumap 文件的辅助程序，以及 Linux sched_setaffinity 和 sched_getaffinity 系统调用的 hwloc 等效项。
 *
 * @{
 */

/** \brief 将线程 \p tid 绑定到 cpuset \p set 中的 CPU
 *
 * 行为与 Linux sched_setaffinity 系统调用完全相同，但使用了 hwloc cpuset。
 *
 * \note 这相当于使用 HWLOC_CPUBIND_THREAD 作为标志调用 hwloc_set_proc_cpubind()。
 */
HWLOC_DECLSPEC int hwloc_linux_set_tid_cpubind(hwloc_topology_t topology, pid_t tid, hwloc_const_cpuset_t set);

/** \brief 获取线程 \p tid 的当前绑定
 *
 * CPU 集 \p set（之前由调用者分配）将填充为线程最后绑定到的 PU 列表。
 *
 * 行为与 Linux sched_getaffinity 系统调用完全相同，但使用了 hwloc cpuset。
 *
 * \note 这相当于使用 HWLOC_CPUBIND_THREAD 作为标志调用 hwloc_get_proc_cpubind()。
 */
HWLOC_DECLSPEC int hwloc_linux_get_tid_cpubind(hwloc_topology_t topology, pid_t tid, hwloc_cpuset_t set);

/** \brief 获取线程 \p tid 最后运行的物理 CPU。
 *
 * CPU 集 \p set（之前由调用者分配）将填充为线程最后运行的 PU。
 *
 * \note 这相当于使用 HWLOC_CPUBIND_THREAD 作为标志调用 hwloc_get_proc_last_cpu_location()。
 */
// 声明一个函数，用于获取指定线程在最后一个 CPU 上的位置
HWLOC_DECLSPEC int hwloc_linux_get_tid_last_cpu_location(hwloc_topology_t topology, pid_t tid, hwloc_bitmap_t set);

/** \brief 将 Linux 内核的 cpumask 文件 \p path 转换为 hwloc 位图 \p set。
 *
 * 可能在从 sysfs 属性（如处理器的拓扑和缓存，或设备的本地 CPU）中读取 CPU 集时使用。
 *
 * \note 此函数忽略 HWLOC_FSROOT 环境变量。
 */
HWLOC_DECLSPEC int hwloc_linux_read_path_as_cpumask(const char *path, hwloc_bitmap_t set);

/** @} */

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_LINUX_H */
```