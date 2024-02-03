# `xmrig\src\3rdparty\hwloc\include\hwloc\windows.h`

```cpp
/*
 * 版权所有 © 2021 Inria。保留所有权利。
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 与 Windows 交互的宏。
 *
 * 在 Windows 上使用 hwloc 的应用程序可能希望包含此文件以获取特定于 Windows 的 hwloc 功能。
 */

#ifndef HWLOC_WINDOWS_H
#define HWLOC_WINDOWS_H

#include "hwloc.h"


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_windows Windows-specific helpers
 *
 * 这些函数查询 Windows 处理器组。
 * 这些组将操作系统分成最多 64 个相邻 PU 的虚拟集合。
 * 线程和进程只能绑定在单个组内。
 * 虽然 Windows 处理器组可能在 hwloc 层次结构中显示为 hwloc 组，但它们通常也会合并到现有的 hwloc 对象中，例如 NUMA 节点或 Packages。
 * 此 API 提供有关 Windows 处理器组的显式信息，以便应用程序知道绑定到大量 PU 可能会失败，因为它跨越多个 Windows 处理器组。
 *
 * @{
 */


/** \brief 获取 Windows 处理器组的数量
 *
 * \p flags 现在必须为 0。
 *
 * \return 成功时至少为 \c 1。
 * \return 错误时返回 -1，例如如果拓扑与当前系统不匹配（例如通过 XML 从另一台机器加载）。
 */
HWLOC_DECLSPEC int hwloc_windows_get_nr_processor_groups(hwloc_topology_t topology, unsigned long flags);

/** \brief 获取 Windows 处理器组的 CPU 集合。
 *
 * 获取包含在由 \p pg_index 指定的处理器组中的 PU 集合。
 * \p pg_index 必须在 0 和 hwloc_windows_get_nr_processor_groups() 返回值减 1 之间。
 *
 * \p flags 现在必须为 0。
 *
 * \return 成功时返回 \c 0。
 * \return 错误时返回 \c -1，例如如果 \p pg_index 无效，或者如果拓扑与当前系统不匹配（例如通过 XML 从另一台机器加载）。
 */
# 声明一个函数，用于获取指定处理器组的 CPU 集合
HWLOC_DECLSPEC int hwloc_windows_get_processor_group_cpuset(hwloc_topology_t topology, unsigned pg_index, hwloc_cpuset_t cpuset, unsigned long flags);

/** @} */

# 如果是 C++ 环境，则使用 extern "C" 包裹代码
#ifdef __cplusplus
} /* extern "C" */
#endif

# 结束条件编译，确保头文件只被包含一次
#endif /* HWLOC_WINDOWS_H */
```