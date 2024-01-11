# `xmrig\src\3rdparty\hwloc\include\hwloc\linux-libnuma.h`

```
/*
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2017 Inria。保留所有权利。
 * 版权所有 © 2009-2010, 2012 波尔多大学
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 和 Linux libnuma 之间交互的宏。
 *
 * 使用同时使用 Linux libnuma 和 hwloc 的应用程序可能希望包含此文件，以便简化它们各自类型之间的转换。
*/

#ifndef HWLOC_LINUX_LIBNUMA_H
#define HWLOC_LINUX_LIBNUMA_H

#include "hwloc.h"

#include <numa.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_linux_libnuma_ulongs 与 Linux libnuma 无符号长整型掩码的互操作性
 *
 * 此接口有助于在 Linux libnuma 无符号长整型掩码和 hwloc cpusets 和 nodesets 之间进行转换。
 *
 * \note 拓扑 \p topology 必须与当前机器匹配。
 *
 * \note 如果内核不是 NUMA-aware（在内核配置中未设置 CONFIG_NUMA），则 libnuma 的行为是未定义的。
 * 在这种情况下，此辅助程序和 libnuma 可能不严格兼容，可以通过检查 numa_available() 是否返回 -1 来检测到这一点。
 *
 * @{
 */


/** \brief 将 hwloc CPU 集 \p cpuset 转换为无符号长整型数组 \p mask
 *
 * \p mask 是将要填充的无符号长整型数组。
 * \p maxnode 包含了可能存储在 \p mask 中的最大节点编号。
 * \p maxnode 将设置为找到的最大节点编号加一。
 *
 * 在调用 set_mempolicy、mbind、migrate_pages 或任何接受无符号长整型数组和最大节点编号作为输入参数的函数之前，可以使用此函数。
 */
static __hwloc_inline int
hwloc_cpuset_to_linux_libnuma_ulongs(hwloc_topology_t topology, hwloc_const_cpuset_t cpuset,
                    unsigned long *mask, unsigned long *maxnode)
{
  // 获取 NUMA 节点的深度
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  // 初始化最大节点数为无符号长整型的最大值
  unsigned long outmaxnode = -1;
  // 初始化节点对象为空
  hwloc_obj_t node = NULL;

  /* 将 maxnode 向上取整到下一个 ulong，并清除所有字节 */
  *maxnode = (*maxnode + 8*sizeof(*mask) - 1) & ~(8*sizeof(*mask) - 1);
  memset(mask, 0, *maxnode/8);

  // 遍历覆盖 cpuset 的下一个深度的对象
  while ((node = hwloc_get_next_obj_covering_cpuset_by_depth(topology, cpuset, depth, node)) != NULL) {
    // 如果节点的索引大于等于最大节点数，则继续下一次循环
    if (node->os_index >= *maxnode)
      continue;
    // 将节点的索引位置置为 1
    mask[node->os_index/sizeof(*mask)/8] |= 1UL << (node->os_index % (sizeof(*mask)*8));
    // 更新最大节点数
    if (outmaxnode == (unsigned long) -1 || outmaxnode < node->os_index)
      outmaxnode = node->os_index;
  }

  // 更新最大节点数
  *maxnode = outmaxnode+1;
  // 返回 0 表示成功
  return 0;
}

/** \brief 将 hwloc NUMA 节点集 nodeset 转换为无符号长整型数组 mask
 *
 * \p mask 是将要填充的无符号长整型数组
 * \p maxnode 包含可能存储在 \p mask 中的最大节点编号
 * \p maxnode 将被设置为找到的最大节点编号加一
 *
 * 在调用 set_mempolicy、mbind、migrate_pages 或任何需要无符号长整型数组和最大节点编号作为输入参数的函数之前，可以使用此函数。
 */
static __hwloc_inline int
hwloc_nodeset_to_linux_libnuma_ulongs(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset,
                      unsigned long *mask, unsigned long *maxnode)
{
  // 获取 NUMA 节点的深度
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  // 初始化最大节点数为无符号长整型的最大值
  unsigned long outmaxnode = -1;
  // 初始化节点对象为空
  hwloc_obj_t node = NULL;

  /* 将 maxnode 向上取整到下一个 ulong，并清除所有字节 */
  *maxnode = (*maxnode + 8*sizeof(*mask) - 1) & ~(8*sizeof(*mask) - 1);
  memset(mask, 0, *maxnode/8);

  // 遍历指定深度的下一个对象
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL) {
    // 如果节点的索引大于等于最大节点数，则继续下一次循环
    if (node->os_index >= *maxnode)
      continue;
    // 如果节点的索引不在 nodeset 中，则继续下一次循环
    if (!hwloc_bitmap_isset(nodeset, node->os_index))
      continue;
    // 将节点的索引位置置为 1
    mask[node->os_index/sizeof(*mask)/8] |= 1UL << (node->os_index % (sizeof(*mask)*8));
    # 如果 outmaxnode 的值等于无符号长整型的最大值，或者小于当前节点的 os_index 值
    if (outmaxnode == (unsigned long) -1 || outmaxnode < node->os_index)
      # 将当前节点的 os_index 值赋给 outmaxnode
      outmaxnode = node->os_index;
  }

  # 将 outmaxnode 的值加一赋给 maxnode
  *maxnode = outmaxnode+1;
  # 返回 0，表示执行成功
  return 0;
} // 结束函数定义

/**
 * \brief 将无符号长整型数组 \p mask 转换为 hwloc CPU 集合
 *
 * \p mask 是将要读取的无符号长整型数组。
 * \p maxnode 包含了可能在 \p mask 中读取的最大节点编号。
 *
 * 此函数可用于在调用 get_mempolicy 或任何其他将无符号长整型数组作为输出参数（可能还有最大节点编号作为输入参数）的函数之后使用。
 */
static __hwloc_inline int
hwloc_cpuset_from_linux_libnuma_ulongs(hwloc_topology_t topology, hwloc_cpuset_t cpuset,
                      const unsigned long *mask, unsigned long maxnode)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  hwloc_obj_t node = NULL;
  hwloc_bitmap_zero(cpuset);
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (node->os_index < maxnode
    && (mask[node->os_index/sizeof(*mask)/8] & (1UL << (node->os_index % (sizeof(*mask)*8)))))
      hwloc_bitmap_or(cpuset, cpuset, node->cpuset);
  return 0;
}

/**
 * \brief 将无符号长整型数组 \p mask 转换为 hwloc NUMA 节点集合
 *
 * \p mask 是将要读取的无符号长整型数组。
 * \p maxnode 包含了可能在 \p mask 中读取的最大节点编号。
 *
 * 此函数可用于在调用 get_mempolicy 或任何其他将无符号长整型数组作为输出参数（可能还有最大节点编号作为输入参数）的函数之后使用。
 */
static __hwloc_inline int
hwloc_nodeset_from_linux_libnuma_ulongs(hwloc_topology_t topology, hwloc_nodeset_t nodeset,
                    const unsigned long *mask, unsigned long maxnode)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  hwloc_obj_t node = NULL;
  hwloc_bitmap_zero(nodeset);
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (node->os_index < maxnode
    # 检查节点的操作系统索引是否在掩码数组中，并且对应位是否为1
    && (mask[node->os_index/sizeof(*mask)/8] & (1UL << (node->os_index % (sizeof(*mask)*8))))
      # 设置节点对应的位在节点集合中
      hwloc_bitmap_set(nodeset, node->os_index);
  # 返回0表示成功
  return 0;
}

/** @} */



/** \defgroup hwlocality_linux_libnuma_bitmask Interoperability with Linux libnuma bitmask
 *
 * This interface helps converting between Linux libnuma bitmasks
 * and hwloc cpusets and nodesets.
 *
 * \note Topology \p topology must match the current machine.
 *
 * \note The behavior of libnuma is undefined if the kernel is not NUMA-aware.
 * (when CONFIG_NUMA is not set in the kernel configuration).
 * This helper and libnuma may thus not be strictly compatible in this case,
 * which may be detected by checking whether numa_available() returns -1.
 *
 * @{
 */


/** \brief Convert hwloc CPU set \p cpuset into the returned libnuma bitmask
 *
 * The returned bitmask should later be freed with numa_bitmask_free.
 *
 * This function may be used before calling many numa_ functions
 * that use a struct bitmask as an input parameter.
 *
 * \return newly allocated struct bitmask.
 */
static __hwloc_inline struct bitmask *
hwloc_cpuset_to_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_const_cpuset_t cpuset) __hwloc_attribute_malloc;
static __hwloc_inline struct bitmask *
hwloc_cpuset_to_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_const_cpuset_t cpuset)
{
  // 获取 NUMA 节点的深度
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  // 初始化节点对象
  hwloc_obj_t node = NULL;
  // 分配一个 libnuma 的位掩码
  struct bitmask *bitmask = numa_allocate_cpumask();
  // 如果分配失败，则返回空指针
  if (!bitmask)
    return NULL;
  // 遍历 NUMA 节点，将包含在 cpuset 中的节点添加到位掩码中
  while ((node = hwloc_get_next_obj_covering_cpuset_by_depth(topology, cpuset, depth, node)) != NULL)
    if (node->attr->numanode.local_memory)
      numa_bitmask_setbit(bitmask, node->os_index);
  return bitmask;
}

/** \brief Convert hwloc NUMA node set \p nodeset into the returned libnuma bitmask
 *
 * The returned bitmask should later be freed with numa_bitmask_free.
 *
 * This function may be used before calling many numa_ functions
 * that use a struct bitmask as an input parameter.
 *
 * \return newly allocated struct bitmask.
 */
static __hwloc_inline struct bitmask *
# 将 hwloc 节点集转换为 Linux libnuma 位掩码
hwloc_nodeset_to_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset) __hwloc_attribute_malloc;

# 将 hwloc 节点集转换为 Linux libnuma 位掩码的内联函数
static __hwloc_inline struct bitmask *
hwloc_nodeset_to_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset)
{
  # 获取 NUMA 节点的深度
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  # 初始化节点对象
  hwloc_obj_t node = NULL;
  # 分配一个位掩码
  struct bitmask *bitmask = numa_allocate_cpumask();
  # 如果分配失败则返回空
  if (!bitmask)
    return NULL;
  # 遍历节点对象，将节点的位设置到位掩码中
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (hwloc_bitmap_isset(nodeset, node->os_index) && node->attr->numanode.local_memory)
      numa_bitmask_setbit(bitmask, node->os_index);
  # 返回位掩码
  return bitmask;
}

# 将 libnuma 位掩码转换为 hwloc CPU 集合的内联函数
static __hwloc_inline int
hwloc_cpuset_from_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_cpuset_t cpuset,
                    const struct bitmask *bitmask)
{
  # 获取 NUMA 节点的深度
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  # 初始化节点对象
  hwloc_obj_t node = NULL;
  # 将 CPU 集合清零
  hwloc_bitmap_zero(cpuset);
  # 遍历节点对象，将位掩码中的位设置到 CPU 集合中
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (numa_bitmask_isbitset(bitmask, node->os_index))
      hwloc_bitmap_or(cpuset, cpuset, node->cpuset);
  # 返回 0
  return 0;
}

# 将 libnuma 位掩码转换为 hwloc NUMA 节点集合的内联函数
static __hwloc_inline int
hwloc_nodeset_from_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_nodeset_t nodeset,
                     const struct bitmask *bitmask)
{
  # 获取 NUMA 节点的深度
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  # 初始化节点对象
  hwloc_obj_t node = NULL;
  # 将 NUMA 节点集合清零
  hwloc_bitmap_zero(nodeset);
  # 遍历节点对象，将位掩码中的位设置到 NUMA 节点集合中
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    # 检查位掩码中是否设置了指定节点的位
    if (numa_bitmask_isbitset(bitmask, node->os_index))
      # 如果设置了，则将对应的节点添加到节点集合中
      hwloc_bitmap_set(nodeset, node->os_index);
  # 返回 0，表示成功执行
  return 0;
// 结束对特定功能模块的注释
/** @} */

// 如果是 C++ 环境，则使用 extern "C" 包裹代码
#ifdef __cplusplus
} /* extern "C" */
#endif

// 结束对 HWLOC_LINUX_NUMA_H 文件的条件编译
#endif /* HWLOC_LINUX_NUMA_H */
```