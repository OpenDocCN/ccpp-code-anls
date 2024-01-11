# `xmrig\src\3rdparty\hwloc\include\hwloc\deprecated.h`

```
/*
 * 版权声明
 * 该文件包含在 hwloc.h 中声明的内联函数的内联代码
 */

#ifndef HWLOC_DEPRECATED_H
#define HWLOC_DEPRECATED_H

#ifndef HWLOC_H
#error 请包含主要的 hwloc.h
#endif

#ifdef __cplusplus
extern "C" {
#endif

/* 与 v2.0 中 WHOLE_SYSTEM 重命名之前的向后兼容 */
#define HWLOC_TOPOLOGY_FLAG_WHOLE_SYSTEM HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED
/* 与 v1.11 中 System 移除之前的向后兼容 */
#define HWLOC_OBJ_SYSTEM HWLOC_OBJ_MACHINE
/* 与 v1.10 中 Socket->Package 重命名之前的向后兼容 */
#define HWLOC_OBJ_SOCKET HWLOC_OBJ_PACKAGE
/* 与 v1.10 中 Node->NUMANode 澄清之前的向后兼容 */
#define HWLOC_OBJ_NODE HWLOC_OBJ_NUMANODE

/** \brief 添加一个距离结构。
 *
 * 在 v2.5 中被 hwloc_distances_add_create()+hwloc_distances_add_values()+hwloc_distances_add_commit() 取代。
 */
HWLOC_DECLSPEC int hwloc_distances_add(hwloc_topology_t topology,
                       unsigned nbobjs, hwloc_obj_t *objs, hwloc_uint64_t *values,
                       unsigned long kind, unsigned long flags) __hwloc_attribute_deprecated;

/** \brief 按父对象插入一个杂项对象。
 *
 * 与 hwloc_topology_insert_misc_object() 相同。
 */
static __hwloc_inline hwloc_obj_t
hwloc_topology_insert_misc_object_by_parent(hwloc_topology_t topology, hwloc_obj_t parent, const char *name) __hwloc_attribute_deprecated;
static __hwloc_inline hwloc_obj_t
hwloc_topology_insert_misc_object_by_parent(hwloc_topology_t topology, hwloc_obj_t parent, const char *name)
{
  return hwloc_topology_insert_misc_object(topology, parent, name);
}
/** \brief Stringify the cpuset containing a set of objects.
 *
 * If \p size is 0, \p string may safely be \c NULL.
 *
 * \return the number of characters that were actually written if not truncating,
 * or that would have been written (not including the ending \\0).
 */
static __hwloc_inline int
hwloc_obj_cpuset_snprintf(char *str, size_t size, size_t nobj, struct hwloc_obj * const *objs) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_obj_cpuset_snprintf(char *str, size_t size, size_t nobj, struct hwloc_obj * const *objs)
{
  // 分配一个位图用于存储 CPU 集合
  hwloc_bitmap_t set = hwloc_bitmap_alloc();
  int res;
  unsigned i;

  // 将位图清零
  hwloc_bitmap_zero(set);
  // 遍历对象数组，将每个对象的 cpuset 合并到位图中
  for(i=0; i<nobj; i++)
    if (objs[i]->cpuset)
      hwloc_bitmap_or(set, set, objs[i]->cpuset);

  // 将位图内容格式化为字符串
  res = hwloc_bitmap_snprintf(str, size, set);
  // 释放位图
  hwloc_bitmap_free(set);
  return res;
}

/** \brief Convert a type string into a type and some attributes.
 *
 * Deprecated by hwloc_type_sscanf()
 */
static __hwloc_inline int
hwloc_obj_type_sscanf(const char *string, hwloc_obj_type_t *typep, int *depthattrp, void *typeattrp, size_t typeattrsize) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_obj_type_sscanf(const char *string, hwloc_obj_type_t *typep, int *depthattrp, void *typeattrp, size_t typeattrsize)
{
  union hwloc_obj_attr_u attr;
  // 使用 hwloc_type_sscanf() 将类型字符串转换为类型和属性
  int err = hwloc_type_sscanf(string, typep, &attr, sizeof(attr));
  if (err < 0)
    return err;
  // 如果是缓存类型的对象，获取深度和类型属性
  if (hwloc_obj_type_is_cache(*typep)) {
    if (depthattrp)
      *depthattrp = (int) attr.cache.depth;
    if (typeattrp && typeattrsize >= sizeof(hwloc_obj_cache_type_t))
      memcpy(typeattrp, &attr.cache.type, sizeof(hwloc_obj_cache_type_t));
  } else if (*typep == HWLOC_OBJ_GROUP) {
    // 如果是组类型的对象，获取深度属性
    if (depthattrp)
      *depthattrp = (int) attr.group.depth;
  }
  return 0;
}

/** \brief Set the default memory binding policy of the current
 * process or thread to prefer the NUMA node(s) specified by physical \p nodeset
 */
static __hwloc_inline int
# 设置特定节点集的内存绑定策略
static __hwloc_inline int
hwloc_set_membind_nodeset(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  # 调用 hwloc_set_membind 函数，传入节点集、策略和标志位
  return hwloc_set_membind(topology, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief 查询当前进程或线程的默认内存绑定策略和物理局部性
 */
static __hwloc_inline int
hwloc_get_membind_nodeset(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_get_membind_nodeset(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  # 调用 hwloc_get_membind 函数，传入节点集、策略和标志位
  return hwloc_get_membind(topology, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief 将指定进程的默认内存绑定策略设置为优先使用物理节点集中的 NUMA 节点
 */
static __hwloc_inline int
hwloc_set_proc_membind_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_set_proc_membind_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  # 调用 hwloc_set_proc_membind 函数，传入拓扑、进程 ID、节点集、策略和标志位
  return hwloc_set_proc_membind(topology, pid, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief 查询指定进程的默认内存绑定策略和物理局部性
 */
static __hwloc_inline int
hwloc_get_proc_membind_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
# 将已分配的内存绑定到物理节点集合中的 NUMA 节点
hwloc_get_proc_membind_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  return hwloc_get_proc_membind(topology, pid, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief 将已分配的内存（由 addr 和 len 标识）绑定到物理 nodeset 中的 NUMA 节点
 */
static __hwloc_inline int
hwloc_set_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_set_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  return hwloc_set_area_membind(topology, addr, len, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief 查询由 (addr, len) 标识的内存的物理 NUMA 节点和绑定策略
 */
static __hwloc_inline int
hwloc_get_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_get_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  return hwloc_get_area_membind(topology, addr, len, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief 在给定的物理节点集合 nodeset 上分配一些内存
 */
static __hwloc_inline void *
hwloc_alloc_membind_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_malloc __hwloc_attribute_deprecated;
static __hwloc_inline void *
hwloc_alloc_membind_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  # 在给定节点集上分配一些内存
  return hwloc_alloc_membind(topology, len, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief 在给定节点集 \p nodeset 上分配一些内存。
 */
static __hwloc_inline void *
hwloc_alloc_membind_policy_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_malloc __hwloc_attribute_deprecated;
static __hwloc_inline void *
hwloc_alloc_membind_policy_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  # 调用 hwloc_alloc_membind_policy 函数，在给定节点集上分配一些内存
  return hwloc_alloc_membind_policy(topology, len, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief 将 CPU 集转换为 NUMA 节点集，并处理非 NUMA 情况
 */
static __hwloc_inline void
hwloc_cpuset_to_nodeset_strict(hwloc_topology_t topology, hwloc_const_cpuset_t _cpuset, hwloc_nodeset_t nodeset) __hwloc_attribute_deprecated;
static __hwloc_inline void
hwloc_cpuset_to_nodeset_strict(hwloc_topology_t topology, hwloc_const_cpuset_t _cpuset, hwloc_nodeset_t nodeset)
{
  # 调用 hwloc_cpuset_to_nodeset 函数，将 CPU 集转换为 NUMA 节点集，并处理非 NUMA 情况
  hwloc_cpuset_to_nodeset(topology, _cpuset, nodeset);
}

/** \brief 将 NUMA 节点集转换为 CPU 集，并处理非 NUMA 情况
 */
static __hwloc_inline void
hwloc_cpuset_from_nodeset_strict(hwloc_topology_t topology, hwloc_cpuset_t _cpuset, hwloc_const_nodeset_t nodeset) __hwloc_attribute_deprecated;
static __hwloc_inline void
hwloc_cpuset_from_nodeset_strict(hwloc_topology_t topology, hwloc_cpuset_t _cpuset, hwloc_const_nodeset_t nodeset)
{
  # 调用 hwloc_cpuset_from_nodeset 函数，将 NUMA 节点集转换为 CPU 集，并处理非 NUMA 情况
  hwloc_cpuset_from_nodeset(topology, _cpuset, nodeset);
}

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_DEPRECATED_H */
```