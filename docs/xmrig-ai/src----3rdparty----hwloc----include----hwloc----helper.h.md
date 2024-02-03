# `xmrig\src\3rdparty\hwloc\include\hwloc\helper.h`

```cpp
/*
 * 版权声明
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2022 Inria
 * 版权所有 © 2009-2012 Université Bordeaux
 * 版权所有 © 2009-2010 Cisco Systems, Inc.
 * 请查看顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 高级 hwloc 遍历辅助函数。
 */

#ifndef HWLOC_HELPER_H
#define HWLOC_HELPER_H

#ifndef HWLOC_H
#error 请包含主要的 hwloc.h
#endif

#include <stdlib.h>
#include <errno.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_helper_find_inside 在 CPU 集合内部查找对象
 * @{
 */

/** \brief 获取包含在给定 cpuset \p set 中的第一个最大对象。
 *
 * \return 包含在 \p set 中且其父对象不在其中的第一个对象。
 *
 * 这对于通过循环获取第一个最大对象并从剩余的 CPU 集合中清除其 CPU 集合来迭代所有最大对象非常方便。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_first_largest_obj_inside_cpuset(hwloc_topology_t topology, hwloc_const_cpuset_t set)
{
  hwloc_obj_t obj = hwloc_get_root_obj(topology);
  if (!hwloc_bitmap_intersects(obj->cpuset, set))
    return NULL;
  while (!hwloc_bitmap_isincluded(obj->cpuset, set)) {
    /* 当对象相交但未被包含时，查看其子对象 */
    hwloc_obj_t child = obj->first_child;
    while (child) {
      if (hwloc_bitmap_intersects(child->cpuset, set))
    break;
      child = child->next_sibling;
    }
    if (!child)
      /* 没有子对象相交，返回它们的父对象 */
      return obj;
    /* 找到一个相交的子对象，查看其子对象 */
    obj = child;
  }
  /* obj 被包含，返回它 */
  return obj;
}

/** \brief 获取恰好覆盖给定 cpuset \p set 的最大对象集合
 *
 * \return 在 \p objs 中返回的对象数量。
 */
# 返回在给定 CPU 集合中深度为 depth 的下一个对象
# 如果 prev 为 NULL，则返回在给定 CPU 集合中深度为 depth 的第一个对象
# 下一次调用应该传入前一个返回值 prev，以获取集合中的下一个对象
# 注意：忽略 CPU 集合为空的对象（否则它们将被认为包含在任何给定集合中）
# 注意：如果给定深度的对象没有 CPU 集合（I/O 或 Misc 对象），此函数无法工作
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                       int depth, hwloc_obj_t prev)
{
  # 获取深度为 depth 的下一个对象
  hwloc_obj_t next = hwloc_get_next_obj_by_depth(topology, depth, prev);
  # 如果没有下一个对象，则返回 NULL
  if (!next)
    return NULL;
  # 循环直到找到下一个对象，其 CPU 集合不为空且包含在给定集合中
  while (next && (hwloc_bitmap_iszero(next->cpuset) || !hwloc_bitmap_isincluded(next->cpuset, set)))
    next = next->next_cousin;
  return next;
}

# 返回在给定 CPU 集合中类型为 type 的下一个对象
# 如果给定类型有多个深度或没有深度，则返回 NULL，并让调用者回退到 hwloc_get_next_obj_inside_cpuset_by_depth()
# 注意：忽略 CPU 集合为空的对象（否则它们将被认为包含在任何给定集合中）
# 注意：如果给定类型的对象没有 CPU 集合（I/O 或 Misc 对象），此函数无法工作
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                      hwloc_obj_type_t type, hwloc_obj_t prev)
{
  # 获取给定类型的深度
  int depth = hwloc_get_type_depth(topology, type);
  # 如果深度未知或有多个深度，则返回 NULL，并让调用者回退到 hwloc_get_next_obj_inside_cpuset_by_depth()
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    # 返回空值
    return NULL;
  # 根据深度获取位于给定 cpuset 中的下一个对象
  return hwloc_get_next_obj_inside_cpuset_by_depth(topology, set, depth, prev);
}  // 结束代码块

/**
 * \brief 根据深度和索引返回 CPU 集合中的第 \p idx 个对象（逻辑上）
 *
 * \note 忽略 CPU 集合为空的对象（否则它们将被认为包含在任何给定的集合中）
 *
 * \note 如果给定深度的对象没有 CPU 集合（I/O 或 Misc 对象），则此函数无法工作
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                      int depth, unsigned idx) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                      int depth, unsigned idx)
{
  // 根据深度获取对象
  hwloc_obj_t obj = hwloc_get_obj_by_depth (topology, depth, 0);
  unsigned count = 0;
  if (!obj)
    return NULL;
  while (obj) {
    // 如果对象的 CPU 集合不为空，并且包含在给定的集合中
    if (!hwloc_bitmap_iszero(obj->cpuset) && hwloc_bitmap_isincluded(obj->cpuset, set)) {
      // 如果计数等于索引，则返回对象
      if (count == idx)
    return obj;
      count++;
    }
    obj = obj->next_cousin;
  }
  return NULL;
}

/**
 * \brief 返回 CPU 集合中类型为 \p type 的第 \p idx 个对象
 *
 * 如果给定类型有多个或没有深度，返回 \c NULL，并让调用者回退到 hwloc_get_obj_inside_cpuset_by_depth()。
 *
 * \note 忽略 CPU 集合为空的对象（否则它们将被认为包含在任何给定的集合中）
 *
 * \note 如果给定类型的对象没有 CPU 集合（I/O 或 Misc 对象），则此函数无法工作
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                     hwloc_obj_type_t type, unsigned idx) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                     hwloc_obj_type_t type, unsigned idx)
{
  // 获取给定类型在拓扑结构中的深度
  int depth = hwloc_get_type_depth(topology, type);
  // 如果深度未知或者存在多个相同深度的类型，则返回空指针
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  // 返回给定深度和索引下的对象
  return hwloc_get_obj_inside_cpuset_by_depth(topology, set, depth, idx);
}

/** \brief 返回 CPU 集合中深度为 \p depth 的对象数量。
 *
 * \note 忽略 CPU 集合为空的对象
 * (否则它们将被认为包含在任何给定集合中)。
 *
 * \note 如果给定深度的对象没有 CPU 集合（I/O 或 Misc 对象），则此函数无法工作。
 */
static __hwloc_inline unsigned
hwloc_get_nbobjs_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                     int depth) __hwloc_attribute_pure;
static __hwloc_inline unsigned
hwloc_get_nbobjs_inside_cpuset_by_depth (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                     int depth)
{
  // 获取给定深度和索引下的对象
  hwloc_obj_t obj = hwloc_get_obj_by_depth (topology, depth, 0);
  unsigned count = 0;
  // 如果对象不存在，则返回 0
  if (!obj)
    return 0;
  // 遍历对象，如果对象的 CPU 集合不为空且包含在给定集合中，则计数加一
  while (obj) {
    if (!hwloc_bitmap_iszero(obj->cpuset) && hwloc_bitmap_isincluded(obj->cpuset, set))
      count++;
    obj = obj->next_cousin;
  }
  return count;
}

/** \brief 返回 CPU 集合中类型为 \p type 的对象数量。
 *
 * 如果 CPU 集合中不存在该类型的对象，则返回 0。如果 CPU 集合中存在多个具有该类型的对象，则返回 -1。
 *
 * \note 忽略 CPU 集合为空的对象
 * (否则它们将被认为包含在任何给定集合中)。
 *
 * \note 如果给定类型的对象没有 CPU 集合（I/O 对象），则此函数无法工作。
 */
static __hwloc_inline int
hwloc_get_nbobjs_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                    hwloc_obj_type_t type) __hwloc_attribute_pure;
static __hwloc_inline int
# 根据给定的拓扑结构、CPU集合和对象类型，返回该类型对象在CPU集合中的数量
hwloc_get_nbobjs_inside_cpuset_by_type (hwloc_topology_t topology, hwloc_const_cpuset_t set,
                    hwloc_obj_type_t type)
{
  # 获取指定类型对象的深度
  int depth = hwloc_get_type_depth(topology, type);
  # 如果深度未知，则返回0
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
    return 0;
  # 如果深度为多个值，则返回-1
  if (depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return -1; /* FIXME: agregate nbobjs from different levels? */
  # 返回指定深度内CPU集合中的对象数量
  return (int) hwloc_get_nbobjs_inside_cpuset_by_depth(topology, set, depth);
}

/** \brief 返回CPU集合中包含的对象的逻辑索引。
 *
 * 在逻辑顺序中，查询CPU集合中与obj相同级别的所有对象，并返回obj在其中的索引。
 * 如果集合覆盖整个拓扑结构，则这是obj的逻辑索引。
 * 否则，这类似于拓扑结构中由CPU集合set定义的部分的逻辑索引。
 *
 * \note 忽略CPU集为空的对象（否则它们将被认为包含在任何给定集合中）。
 *
 * \note 如果obj没有CPU集（I/O对象），则此函数无法工作。
 */
static __hwloc_inline int
hwloc_get_obj_index_inside_cpuset (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t set,
                   hwloc_obj_t obj) __hwloc_attribute_pure;
static __hwloc_inline int
hwloc_get_obj_index_inside_cpuset (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t set,
                   hwloc_obj_t obj)
{
  int idx = 0;
  # 如果obj的cpuset不包含在set中，则返回-1
  if (!hwloc_bitmap_isincluded(obj->cpuset, set))
    return -1;
  # 计算从obj到该级别开头的CPU集合中有多少个对象
  while ((obj = obj->prev_cousin) != NULL)
    if (!hwloc_bitmap_iszero(obj->cpuset) && hwloc_bitmap_isincluded(obj->cpuset, set))
      idx++;
  return idx;
}

/** @} */



/** \defgroup hwlocality_helper_find_covering Finding Objects covering at least CPU set
 * @{
 */
/** \brief Get the child covering at least CPU set \p set.
 *
 * \return \c NULL if no child matches or if \p set is empty.
 *
 * \note This function cannot work if parent does not have a CPU set (I/O or Misc objects).
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_child_covering_cpuset (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t set,
                hwloc_obj_t parent) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_child_covering_cpuset (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_const_cpuset_t set,
                hwloc_obj_t parent)
{
  hwloc_obj_t child; // Declare a variable to store the child object
  if (hwloc_bitmap_iszero(set)) // Check if the CPU set is empty
    return NULL; // Return NULL if the CPU set is empty
  child = parent->first_child; // Get the first child of the parent object
  while (child) { // Start a loop to iterate through the children
    if (child->cpuset && hwloc_bitmap_isincluded(set, child->cpuset)) // Check if the child's cpuset is included in the given CPU set
      return child; // Return the child if its cpuset is included in the given CPU set
    child = child->next_sibling; // Move to the next sibling
  }
  return NULL; // Return NULL if no child matches the given CPU set
}

/** \brief Get the lowest object covering at least CPU set \p set
 *
 * \return \c NULL if no object matches or if \p set is empty.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_covering_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_covering_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set)
{
  struct hwloc_obj *current = hwloc_get_root_obj(topology); // Get the root object of the topology
  if (hwloc_bitmap_iszero(set) || !hwloc_bitmap_isincluded(set, current->cpuset)) // Check if the CPU set is empty or not included in the root object's cpuset
    return NULL; // Return NULL if the CPU set is empty or not included in the root object's cpuset
  while (1) { // Start an infinite loop
    hwloc_obj_t child = hwloc_get_child_covering_cpuset(topology, set, current); // Get the child covering at least the given CPU set
    if (!child) // Check if there is no child covering the given CPU set
      return current; // Return the current object if there is no child covering the given CPU set
    current = child; // Update the current object to the child object
  }
}
/**
 * \brief 通过给定深度对象迭代，该对象至少覆盖 CPU 集合 \p set
 *
 * 如果对象 \p prev 为 \c NULL，则返回深度 \p depth 中覆盖至少部分 CPU 集合 \p set 的第一个对象。
 * 下一次调用应该将前一个返回值 \p prev 传递，以便获取覆盖 \p set 的另一个部分的下一个对象。
 *
 * \note 如果给定深度的对象没有 CPU 集合（I/O 或 Misc 对象），则此函数无法工作。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_covering_cpuset_by_depth(hwloc_topology_t topology, hwloc_const_cpuset_t set,
                        int depth, hwloc_obj_t prev)
{
  hwloc_obj_t next = hwloc_get_next_obj_by_depth(topology, depth, prev);
  if (!next)
    return NULL;
  while (next && !hwloc_bitmap_intersects(set, next->cpuset))
    next = next->next_cousin;
  return next;
}

/**
 * \brief 通过至少覆盖 CPU 集合 \p set 的相同类型对象进行迭代
 *
 * 如果对象 \p prev 为 \c NULL，则返回类型 \p type 中覆盖至少部分 CPU 集合 \p set 的第一个对象。
 * 下一次调用应该将前一个返回值 \p prev 传递，以便获取覆盖至少另一个部分 \p set 的类型 \p type 的下一个对象。
 *
 * 如果类型 \p type 没有或有多个深度，则返回 \c NULL。
 * 调用者可以针对每个深度回退到 hwloc_get_next_obj_covering_cpuset_by_depth()。
 *
 * \note 如果给定类型的对象没有 CPU 集合（I/O 或 Misc 对象），则此函数无法工作。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_covering_cpuset_by_type(hwloc_topology_t topology, hwloc_const_cpuset_t set,
                       hwloc_obj_type_t type, hwloc_obj_t prev)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  return hwloc_get_next_obj_covering_cpuset_by_depth(topology, set, depth, prev);
}

/** @} */
/**
 * @defgroup hwlocality_helper_ancestors Looking at Ancestor and Child Objects
 * @{
 *
 * Be sure to see the figure in @ref termsanddefs that shows a
 * complete topology tree, including depths, child/sibling/cousin
 * relationships, and an example of an asymmetric topology where one
 * package has fewer caches than its peers.
 */

/** \brief Returns the ancestor object of \p obj at depth \p depth.
 *
 * \note \p depth should not be the depth of PU or NUMA objects
 * since they are ancestors of no objects (except Misc or I/O).
 * This function rather expects an intermediate level depth,
 * such as the depth of Packages, Cores, or Caches.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_ancestor_obj_by_depth (hwloc_topology_t topology __hwloc_attribute_unused, int depth, hwloc_obj_t obj) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_ancestor_obj_by_depth (hwloc_topology_t topology __hwloc_attribute_unused, int depth, hwloc_obj_t obj)
{
  hwloc_obj_t ancestor = obj;
  if (obj->depth < depth)
    return NULL;
  while (ancestor && ancestor->depth > depth)
    ancestor = ancestor->parent;
  return ancestor;
}

/** \brief Returns the ancestor object of \p obj with type \p type.
 *
 * \note \p type should not be ::HWLOC_OBJ_PU or ::HWLOC_OBJ_NUMANODE
 * since these objects are ancestors of no objects (except Misc or I/O).
 * This function rather expects an intermediate object type,
 * such as ::HWLOC_OBJ_PACKAGE, ::HWLOC_OBJ_CORE, etc.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_ancestor_obj_by_type (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_type_t type, hwloc_obj_t obj) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_ancestor_obj_by_type (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_type_t type, hwloc_obj_t obj)
{
  hwloc_obj_t ancestor = obj->parent;
  while (ancestor && ancestor->type != type)
    ancestor = ancestor->parent;
  return ancestor;
}
/** \brief 返回对象 \p obj1 和 \p obj2 的共同父对象 */
static __hwloc_inline hwloc_obj_t
hwloc_get_common_ancestor_obj (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj1, hwloc_obj_t obj2) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_common_ancestor_obj (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj1, hwloc_obj_t obj2)
{
  /* 循环并不容易，因为中间的祖先可能有不同的深度，导致我们需要交替使用 obj1->parent 和 obj2->parent。
   * 而且，即使在某个时刻我们找到了相同深度的祖先，它们的祖先可能又有不同的深度。
   */
  while (obj1 != obj2) {
    while (obj1->depth > obj2->depth)
      obj1 = obj1->parent;
    while (obj2->depth > obj1->depth)
      obj2 = obj2->parent;
    if (obj1 != obj2 && obj1->depth == obj2->depth) {
      obj1 = obj1->parent;
      obj2 = obj2->parent;
    }
  }
  return obj1;
}

/** \brief 如果 \p obj 在以祖先对象 \p subtree_root 开始的子树中，则返回 true。
 *
 * \note 如果 \p obj 和 \p subtree_root 对象没有 CPU 集合（I/O 或 Misc 对象），则此函数无法工作。
 */
static __hwloc_inline int
hwloc_obj_is_in_subtree (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj, hwloc_obj_t subtree_root) __hwloc_attribute_pure;
static __hwloc_inline int
hwloc_obj_is_in_subtree (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj, hwloc_obj_t subtree_root)
{
  return obj->cpuset && subtree_root->cpuset && hwloc_bitmap_isincluded(obj->cpuset, subtree_root->cpuset);
}

/** \brief 返回下一个子对象。
 *
 * 在正常子对象列表中返回下一个子对象，
 * 然后在内存子对象列表中返回下一个子对象，
 * 然后在 I/O 子对象列表中返回下一个子对象，
 * 然后在 Misc 子对象列表中返回下一个子对象。
 *
 * 如果 \p prev 为 \c NULL，则返回第一个子对象。
 *
 * 当没有下一个子对象时返回 \c NULL。
 */
static __hwloc_inline hwloc_obj_t
# 获取下一个子对象
hwloc_get_next_child (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t parent, hwloc_obj_t prev)
{
  # 定义对象和状态变量
  hwloc_obj_t obj;
  int state = 0;
  # 如果存在上一个对象
  if (prev) {
    # 如果上一个对象是杂项类型
    if (prev->type == HWLOC_OBJ_MISC)
      state = 3;
    # 如果上一个对象是桥接、PCI设备或操作系统设备类型
    else if (prev->type == HWLOC_OBJ_BRIDGE || prev->type == HWLOC_OBJ_PCI_DEVICE || prev->type == HWLOC_OBJ_OS_DEVICE)
      state = 2;
    # 如果上一个对象是NUMA节点类型
    else if (prev->type == HWLOC_OBJ_NUMANODE)
      state = 1;
    # 获取下一个兄弟对象
    obj = prev->next_sibling;
  } else {
    # 获取父对象的第一个子对象
    obj = parent->first_child;
  }
  # 如果没有找到对象且状态为0
  if (!obj && state == 0) {
    # 获取父对象的内存子对象
    obj = parent->memory_first_child;
    state = 1;
  }
  # 如果没有找到对象且状态为1
  if (!obj && state == 1) {
    # 获取父对象的I/O子对象
    obj = parent->io_first_child;
    state = 2;
  }
  # 如果没有找到对象且状态为2
  if (!obj && state == 2) {
    # 获取父对象的杂项子对象
    obj = parent->misc_first_child;
    state = 3;
  }
  # 返回找到的对象
  return obj;
}

/** @} */

# 定义对象类型的分组
/** \defgroup hwlocality_helper_types Kinds of object Type
 * @{
 *
 * Each object type is
 * either Normal (i.e. hwloc_obj_type_is_normal() returns 1),
 * or Memory (i.e. hwloc_obj_type_is_memory() returns 1)
 * or I/O (i.e. hwloc_obj_type_is_io() returns 1)
 * or Misc (i.e. equal to ::HWLOC_OBJ_MISC).
 * It cannot be of more than one of these kinds.
 */

# 检查对象类型是否为Normal
/** \brief Check whether an object type is Normal.
 *
 * Normal objects are objects of the main CPU hierarchy
 * (Machine, Package, Core, PU, CPU caches, etc.),
 * but they are not NUMA nodes, I/O devices or Misc objects.
 *
 * They are attached to parent as Normal children,
 * not as Memory, I/O or Misc children.
 *
 * \return 1 if an object of type \p type is a Normal object, 0 otherwise.
 */
HWLOC_DECLSPEC int
hwloc_obj_type_is_normal(hwloc_obj_type_t type);

# 检查对象类型是否为I/O
/** \brief Check whether an object type is I/O.
 *
 * I/O objects are objects attached to their parents
 * in the I/O children list.
 * This current includes Bridges, PCI and OS devices.
 *
 * \return 1 if an object of type \p type is a I/O object, 0 otherwise.
 */
HWLOC_DECLSPEC int
hwloc_obj_type_is_io(hwloc_obj_type_t type);
# 检查对象类型是否为内存对象
# 内存对象是附加到其父对象的Memory子对象列表中的对象
# 目前包括NUMA节点和内存侧缓存
# 如果类型为type的对象是内存对象，则返回1，否则返回0
HWLOC_DECLSPEC int
hwloc_obj_type_is_memory(hwloc_obj_type_t type);

# 检查对象类型是否为CPU缓存（数据、统一或指令）
# 内存侧缓存不是CPU缓存
# 如果类型为type的对象是缓存，则返回1，否则返回0
HWLOC_DECLSPEC int
hwloc_obj_type_is_cache(hwloc_obj_type_t type);

# 检查对象类型是否为CPU数据或统一缓存
# 内存侧缓存不是CPU缓存
# 如果类型为type的对象是CPU数据或统一缓存，则返回1，否则返回0
HWLOC_DECLSPEC int
hwloc_obj_type_is_dcache(hwloc_obj_type_t type);

# 检查对象类型是否为CPU指令缓存
# 内存侧缓存不是CPU缓存
# 如果类型为type的对象是CPU指令缓存，则返回1，否则返回0
HWLOC_DECLSPEC int
hwloc_obj_type_is_icache(hwloc_obj_type_t type);

# 查看缓存对象
# @{
/**
 * \brief 查找与缓存级别和类型匹配的缓存对象的深度。
 *
 * 返回包含缓存对象的拓扑级别的深度，其属性与 \p cachelevel 和 \p cachetype 匹配。
 *
 * 此函数与使用相应类型（如 ::HWLOC_OBJ_L1ICACHE）调用 hwloc_get_type_depth() 完全相同，只是在查找指令缓存时也可能返回统一缓存。
 *
 * 如果没有匹配的缓存级别，则返回 ::HWLOC_TYPE_DEPTH_UNKNOWN。
 *
 * 如果 \p cachetype 是 ::HWLOC_OBJ_CACHE_UNIFIED，则返回唯一匹配的统一缓存级别的深度。
 *
 * 如果 \p cachetype 是 ::HWLOC_OBJ_CACHE_DATA 或 ::HWLOC_OBJ_CACHE_INSTRUCTION，则返回匹配的缓存或统一缓存。
 *
 * 如果 \p cachetype 是 \c -1，则忽略它，可能匹配多个级别。函数返回唯一匹配级别的深度或 ::HWLOC_TYPE_DEPTH_MULTIPLE。
 */
static __hwloc_inline int
hwloc_get_cache_type_depth (hwloc_topology_t topology,
                unsigned cachelevel, hwloc_obj_cache_type_t cachetype)
{
  int depth;
  int found = HWLOC_TYPE_DEPTH_UNKNOWN;
  for (depth=0; ; depth++) {
    hwloc_obj_t obj = hwloc_get_obj_by_depth(topology, depth, 0);
    if (!obj)
      break;
    if (!hwloc_obj_type_is_dcache(obj->type) || obj->attr->cache.depth != cachelevel)
      /* 不匹配，尝试下一个深度 */
      continue;
    if (cachetype == (hwloc_obj_cache_type_t) -1) {
      if (found != HWLOC_TYPE_DEPTH_UNKNOWN) {
    /* 第二次匹配，返回 MULTIPLE */
        return HWLOC_TYPE_DEPTH_MULTIPLE;
      }
      /* 第一次匹配，标记为找到 */
      found = depth;
      continue;
    }
    if (obj->attr->cache.type == cachetype || obj->attr->cache.type == HWLOC_OBJ_CACHE_UNIFIED)
      /* 精确匹配（统一缓存单独存在，或者我们匹配指令或数据），立即返回 */
      return depth;
  }
  /* 到达底部，返回找到的内容 */
  return found;
}
/** \brief 获取覆盖 cpuset \p set 的第一个数据（或统一）缓存
 *
 * \return 如果没有匹配的缓存，则返回 \c NULL。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_cache_covering_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_cache_covering_cpuset (hwloc_topology_t topology, hwloc_const_cpuset_t set)
{
  // 获取覆盖 cpuset 的当前对象
  hwloc_obj_t current = hwloc_get_obj_covering_cpuset(topology, set);
  while (current) {
    // 如果当前对象是数据缓存，则返回当前对象
    if (hwloc_obj_type_is_dcache(current->type))
      return current;
    // 继续向上查找父对象
    current = current->parent;
  }
  // 如果没有匹配的缓存，则返回 NULL
  return NULL;
}

/** \brief 获取对象和其他对象之间共享的第一个数据（或统一）缓存
 *
 * \return 如果没有匹配的缓存或给定了无效对象，则返回 \c NULL。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_shared_cache_covering_obj (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_shared_cache_covering_obj (hwloc_topology_t topology __hwloc_attribute_unused, hwloc_obj_t obj)
{
  // 获取对象的父对象
  hwloc_obj_t current = obj->parent;
  // 如果对象的 cpuset 为空，则返回 NULL
  if (!obj->cpuset)
    return NULL;
  while (current) {
    // 如果当前对象的 cpuset 与给定对象的 cpuset不相等，并且当前对象是数据缓存，则返回当前对象
    if (!hwloc_bitmap_isequal(current->cpuset, obj->cpuset)
        && hwloc_obj_type_is_dcache(current->type))
      return current;
    // 继续向上查找父对象
    current = current->parent;
  }
  // 如果没有匹配的缓存，则返回 NULL
  return NULL;
}

/** @} */



/** \defgroup hwlocality_helper_find_misc 查找对象，杂项帮助程序
 * @{
 *
 * 请确保查看 \ref termsanddefs 中显示完整拓扑树的图，包括深度、子/兄弟/堂兄关系，以及一个拓扑不对称的示例，其中一个 package 比其同级别的 package 拥有更少的缓存。
 */
/**
 * \brief 从 CPU 集合中移除同时多线程处理器单元（PU）。
 *
 * 对于拓扑结构中的每个核心，如果 \p cpuset 包含该核心的一些 PU，
 * 则修改 \p cpuset 以仅保留该核心的一个 PU。
 *
 * \p which 指定要保留哪个 PU。
 * PU 按物理索引顺序考虑。
 * 如果为 0，则对于每个核心，函数将保留最初在 \p cpuset 中设置的第一个 PU。
 *
 * 如果 \p which 大于在 \p cpuset 中最初设置的 PU 数量，
 * 则不会保留该核心的任何 PU。
 *
 * \note 忽略不在核心对象下的 PU
 * （例如，如果拓扑结构不包含任何核心对象）。
 * 它们中的任何一个都不会从 \p cpuset 中移除。
 */
HWLOC_DECLSPEC int hwloc_bitmap_singlify_per_core(hwloc_topology_t topology, hwloc_bitmap_t cpuset, unsigned which);

/**
 * \brief 返回具有 \p os_index 的 ::HWLOC_OBJ_PU 类型的对象。
 *
 * 此函数对于将 CPU 集合转换为其包含的 PU 对象非常有用。
 * 当检索当前绑定（例如，使用 hwloc_get_cpubind()）时，
 * 可以使用 hwloc_bitmap_foreach_begin() 迭代结果 CPU 集合的位，
 * 并使用此函数找到相应的 PU。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_pu_obj_by_os_index(hwloc_topology_t topology, unsigned os_index) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_pu_obj_by_os_index(hwloc_topology_t topology, unsigned os_index)
{
  hwloc_obj_t obj = NULL;
  while ((obj = hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_PU, obj)) != NULL)
    if (obj->os_index == os_index)
      return obj;
  return NULL;
}
/**
 * \brief 返回具有指定 \p os_index 的 ::HWLOC_OBJ_NUMANODE 类型的对象。
 *
 * 此函数用于将节点集转换为其包含的 NUMA 节点对象。
 * 在检索当前绑定（例如使用 hwloc_get_membind() 和 HWLOC_MEMBIND_BYNODESET）时，
 * 可以使用 hwloc_bitmap_foreach_begin() 遍历结果节点集的位，并使用此函数找到相应的 NUMA 节点。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_numanode_obj_by_os_index(hwloc_topology_t topology, unsigned os_index) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_numanode_obj_by_os_index(hwloc_topology_t topology, unsigned os_index)
{
  hwloc_obj_t obj = NULL;
  while ((obj = hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_NUMANODE, obj)) != NULL)
    if (obj->os_index == os_index)
      return obj;
  return NULL;
}

/**
 * \brief 对拓扑结构进行深度优先遍历，找到并排序与 \p src 处于相同深度的所有对象。
 * 在 \p objs 中报告最多 \p max 个与 \p src 最接近的物理对象。
 *
 * \return 返回在 \p objs 中返回的对象数量。
 *
 * \return 如果 \p src 是 I/O 对象，则返回 0。
 *
 * \note 此函数要求 \p src 对象具有 CPU 集。
 */
/* TODO: 是否提供迭代器？提供一种知道应该分配多少的方法？通过返回对象的总数来返回？ */
HWLOC_DECLSPEC unsigned hwloc_get_closest_objs (hwloc_topology_t topology, hwloc_obj_t src, hwloc_obj_t * __hwloc_restrict objs, unsigned max);
/** \brief 根据类型和索引查找另一个对象下面的对象。
 *
 * 从顶层系统对象开始，查找类型为 \p type1 且逻辑索引为 \p idx1 的对象。
 * 然后在该对象下方查找另一个类型为 \p type2 且逻辑索引为 \p idx2 的对象。
 * 索引是在父对象内指定的，而不是在整个系统内指定的。
 *
 * 例如，如果 type1 是 PACKAGE，idx1 是 2，type2 是 CORE，idx2 是 3，
 * 则返回第三个 package 下面的第四个 core 对象。
 *
 * \note 此函数要求这些对象具有 CPU 集。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_below_by_type (hwloc_topology_t topology,
                 hwloc_obj_type_t type1, unsigned idx1,
                 hwloc_obj_type_t type2, unsigned idx2) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_below_by_type (hwloc_topology_t topology,
                 hwloc_obj_type_t type1, unsigned idx1,
                 hwloc_obj_type_t type2, unsigned idx2)
{
  hwloc_obj_t obj;
  // 根据类型和索引查找对象
  obj = hwloc_get_obj_by_type (topology, type1, idx1);
  // 如果找不到对象，则返回空指针
  if (!obj)
    return NULL;
  // 根据类型和索引在 CPU 集内查找对象
  return hwloc_get_obj_inside_cpuset_by_type(topology, obj->cpuset, type2, idx2);
}
/**
 * \brief 根据类型和索引查找对象链中的对象。
 *
 * 这是 hwloc_get_obj_below_by_type() 的通用版本。
 *
 * 数组 typev 和 idxv 必须包含 nr 个类型和索引。
 *
 * 从顶层系统对象开始，遍历数组 typev 和 idxv。
 * 对于数组中的每个类型和逻辑索引对，查找先前找到的对象下面的给定类型的第 index 个对象。
 * 索引是在父对象内指定的，而不是在整个系统内指定的。
 *
 * 例如，如果 nr 为 3，typev 包含 NODE、PACKAGE 和 CORE，
 * idxv 包含 0、1 和 2，返回第一个 NUMA 节点下面第二个 package 下面第三个 core 对象。
 *
 * \note 此函数要求所有这些对象和根对象都有一个 CPU 集。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_below_array_by_type (hwloc_topology_t topology, int nr, hwloc_obj_type_t *typev, unsigned *idxv) __hwloc_attribute_pure;
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_below_array_by_type (hwloc_topology_t topology, int nr, hwloc_obj_type_t *typev, unsigned *idxv)
{
  // 获取根对象
  hwloc_obj_t obj = hwloc_get_root_obj(topology);
  int i;
  for(i=0; i<nr; i++) {
    // 如果对象为空，则返回空
    if (!obj)
      return NULL;
    // 根据类型和索引在对象的 cpuset 内获取对象
    obj = hwloc_get_obj_inside_cpuset_by_type(topology, obj->cpuset, typev[i], idxv[i]);
  }
  return obj;
}
# 返回具有相同本地性的不同类型的对象
#
# 如果源对象 \p src 是普通或内存类型，该函数返回具有相同 CPU 和节点集的 \p type 类型的对象，
# 位于层次结构的下方或上方。
#
# 如果源对象 \p src 是 PCI 设备或 PCI 设备内的操作系统设备，该函数可能返回该 PCI 设备，
# 或者同一 PCI 父级中的另一个操作系统设备。
# 例如，这可能对于在距离结构中转换操作系统设备（如 "nvml0" 或 "rsmi1"）为相应的 PCI 设备，
# 或者与同一物理卡对应的 CUDA 或 OpenCL 操作系统设备非常有用。
#
# 如果 \c NULL，参数 \p subtype 仅选择其 subtype 属性存在且为 \p subtype（不区分大小写）的对象，
# 例如 "OpenCL" 或 "CUDA"。
#
# 如果 \c NULL，参数 \p nameprefix 仅选择其 name 属性存在且以 \p nameprefix（不区分大小写）开头的对象，
# 例如匹配 "rsmi0" 的 "rsmi"。
#
# 如果有多个对象匹配，将返回第一个对象。
#
# 由于 PCI 本地性可能会发生变化，此函数不会遍历跨桥层次结构。
# 此函数也不能在普通/内存对象和 I/O 或 Misc 对象之间进行转换。
#
# \p flags 现在必须为 \c 0。
#
# 如果有，返回具有相同本地性，匹配 \p subtype 和 \p nameprefix 的对象。
#
# 如果找不到匹配的对象，或者源对象和目标类型不兼容，则返回 \c NULL，
# 例如在 CPU 和 I/O 对象之间进行转换时。
HWLOC_DECLSPEC hwloc_obj_t
hwloc_get_obj_with_same_locality(hwloc_topology_t topology, hwloc_obj_t src,
                                 hwloc_obj_type_t type, const char *subtype, const char *nameprefix,
                                 unsigned long flags);

# 分布拓扑结构上的项目
#
# @{
/** \brief Flags to be given to hwloc_distrib().
 */
enum hwloc_distrib_flags_e {
  /** \brief Distrib in reverse order, starting from the last objects.
   * \hideinitializer
   */
  HWLOC_DISTRIB_FLAG_REVERSE = (1UL<<0)
};

/** \brief Distribute \p n items over the topology under \p roots
 *
 * Array \p set will be filled with \p n cpusets recursively distributed
 * linearly over the topology under objects \p roots, down to depth \p until
 * (which can be INT_MAX to distribute down to the finest level).
 *
 * \p n_roots is usually 1 and \p roots only contains the topology root object
 * so as to distribute over the entire topology.
 *
 * This is typically useful when an application wants to distribute \p n
 * threads over a machine, giving each of them as much private cache as
 * possible and keeping them locally in number order.
 *
 * The caller may typically want to also call hwloc_bitmap_singlify()
 * before binding a thread so that it does not move at all.
 *
 * \p flags should be 0 or a OR'ed set of ::hwloc_distrib_flags_e.
 *
 * \note This function requires the \p roots objects to have a CPU set.
 */
static __hwloc_inline int
hwloc_distrib(hwloc_topology_t topology,
          hwloc_obj_t *roots, unsigned n_roots,
          hwloc_cpuset_t *set,
          unsigned n,
          int until, unsigned long flags)
{
  unsigned i;
  unsigned tot_weight;
  unsigned given, givenweight;
  hwloc_cpuset_t *cpusetp = set;

  if (flags & ~HWLOC_DISTRIB_FLAG_REVERSE) {
    errno = EINVAL;
    return -1;
  }

  tot_weight = 0;
  for (i = 0; i < n_roots; i++)
    tot_weight += (unsigned) hwloc_bitmap_weight(roots[i]->cpuset);

  for (i = 0, given = 0, givenweight = 0; i < n_roots; i++) {
    unsigned chunk, weight;
    hwloc_obj_t root = roots[flags & HWLOC_DISTRIB_FLAG_REVERSE ? n_roots-1-i : i];
    hwloc_cpuset_t cpuset = root->cpuset;
    while (!hwloc_obj_type_is_normal(root->type))
      /* If memory/io/misc, walk up to normal parent */
      root = root->parent;
    # 计算 cpuset 中位为 1 的个数，即权重
    weight = (unsigned) hwloc_bitmap_weight(cpuset);
    # 如果权重为 0，则跳过当前循环
    if (!weight)
      continue;
    # 给根节点分配与其权重成比例的一部分
    # 如果之前的部分被四舍五入，可能会得到稍少一点
    chunk = (( (givenweight+weight) * n  + tot_weight-1) / tot_weight)
          - ((  givenweight         * n  + tot_weight-1) / tot_weight);
    # 如果根节点没有子节点，或者分配的部分小于等于 1，或者根节点深度大于等于指定深度，则将所有内容放在这里
    if (!root->arity || chunk <= 1 || root->depth >= until) {
      # 如果分配的部分不为 0
      if (chunk) {
    # 用我们的 cpuset 填充 cpusets
    unsigned j;
    for (j=0; j < chunk; j++)
      cpusetp[j] = hwloc_bitmap_dup(cpuset);
      } else {
    # 如果没有分配到部分，将我们的 cpuset 合并到前一个
    # （第一个部分不会为空），以防这个根节点被忽略
    assert(given);
    hwloc_bitmap_or(cpusetp[-1], cpusetp[-1], cpuset);
      }
    } else {
      # 还有更多要分配，递归进入子节点
      hwloc_distrib(topology, root->children, root->arity, cpusetp, chunk, until, flags);
    }
    # 将 cpusetp 指针向后移动 chunk 个位置
    cpusetp += chunk;
    # 统计已分配的部分
    given += chunk;
    # 统计已分配的权重
    givenweight += weight;
  }

  # 返回 0，表示执行成功
  return 0;
/**
 * @defgroup hwlocality_helper_topology_sets CPU and node sets of entire topologies
 * @{
 */

/** 
 * @brief Get complete CPU set
 *
 * @return the complete CPU set of processors of the system.
 *
 * @note The returned cpuset is not newly allocated and should thus not be
 * changed or freed; hwloc_bitmap_dup() must be used to obtain a local copy.
 *
 * @note This is equivalent to retrieving the root object complete CPU-set.
 */
HWLOC_DECLSPEC hwloc_const_cpuset_t
hwloc_topology_get_complete_cpuset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** 
 * @brief Get topology CPU set
 *
 * @return the CPU set of processors of the system for which hwloc
 * provides topology information. This is equivalent to the cpuset of the
 * system object.
 *
 * @note The returned cpuset is not newly allocated and should thus not be
 * changed or freed; hwloc_bitmap_dup() must be used to obtain a local copy.
 *
 * @note This is equivalent to retrieving the root object CPU-set.
 */
HWLOC_DECLSPEC hwloc_const_cpuset_t
hwloc_topology_get_topology_cpuset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** 
 * @brief Get allowed CPU set
 *
 * @return the CPU set of allowed processors of the system.
 *
 * @note If the topology flag ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED was not set,
 * this is identical to hwloc_topology_get_topology_cpuset(), which means
 * all PUs are allowed.
 *
 * @note If ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED was set, applying
 * hwloc_bitmap_intersects() on the result of this function and on an object
 * cpuset checks whether there are allowed PUs inside that object.
 * Applying hwloc_bitmap_and() returns the list of these allowed PUs.
 *
 * @note The returned cpuset is not newly allocated and should thus not be
 * changed or freed, hwloc_bitmap_dup() must be used to obtain a local copy.
 */
HWLOC_DECLSPEC hwloc_const_cpuset_t
hwloc_topology_get_allowed_cpuset(hwloc_topology_t topology) __hwloc_attribute_pure;
/** \brief 获取完整的节点集合
 *
 * \return 系统内存的完整节点集合。
 *
 * \note 返回的节点集合不是新分配的，因此不应更改或释放；必须使用hwloc_bitmap_dup()来获得本地副本。
 *
 * \note 这相当于检索根对象的完整节点集合。
 */
HWLOC_DECLSPEC hwloc_const_nodeset_t
hwloc_topology_get_complete_nodeset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** \brief 获取拓扑节点集合
 *
 * \return 系统内存的节点集合，hwloc提供拓扑信息。这相当于系统对象的节点集合。
 *
 * \note 返回的节点集合不是新分配的，因此不应更改或释放；必须使用hwloc_bitmap_dup()来获得本地副本。
 *
 * \note 这相当于检索根对象的节点集合。
 */
HWLOC_DECLSPEC hwloc_const_nodeset_t
hwloc_topology_get_topology_nodeset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** \brief 获取允许的节点集合
 *
 * \return 系统允许的内存节点集合。
 *
 * \note 如果未设置拓扑标志::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED，则这与hwloc_topology_get_topology_nodeset()相同，这意味着所有NUMA节点都是允许的。
 *
 * \note 如果设置了::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED，对该函数的结果和对象节点集合应用hwloc_bitmap_intersects()，可以检查该对象内是否有允许的NUMA节点。
 * 应用hwloc_bitmap_and()返回这些允许的NUMA节点的列表。
 *
 * \note 返回的节点集合不是新分配的，因此不应更改或释放，必须使用hwloc_bitmap_dup()来获得本地副本。
 */
HWLOC_DECLSPEC hwloc_const_nodeset_t
hwloc_topology_get_allowed_nodeset(hwloc_topology_t topology) __hwloc_attribute_pure;

/** @} */



/** \defgroup hwlocality_helper_nodeset_convert 转换CPU集合和节点集合之间的转换
 *
 * @{
 */
/** \brief Convert a CPU set into a NUMA node set
 *
 * For each PU included in the input \p _cpuset, set the corresponding
 * local NUMA node(s) in the output \p nodeset.
 *
 * If some NUMA nodes have no CPUs at all, this function never sets their
 * indexes in the output node set, even if a full CPU set is given in input.
 *
 * Hence the entire topology CPU set is converted into the set of all nodes
 * that have some local CPUs.
 */
static __hwloc_inline int
hwloc_cpuset_to_nodeset(hwloc_topology_t topology, hwloc_const_cpuset_t _cpuset, hwloc_nodeset_t nodeset)
{
    // 获取 NUMA 节点的深度
    int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
    // 初始化 obj 为 NULL
    hwloc_obj_t obj = NULL;
    // 确保获取到了 NUMA 节点的深度
    assert(depth != HWLOC_TYPE_DEPTH_UNKNOWN);
    // 将 nodeset 清零
    hwloc_bitmap_zero(nodeset);
    // 遍历每个包含在 _cpuset 中的对象，设置相应的本地 NUMA 节点在 nodeset 中
    while ((obj = hwloc_get_next_obj_covering_cpuset_by_depth(topology, _cpuset, depth, obj)) != NULL)
        // 如果设置节点失败，返回 -1
        if (hwloc_bitmap_set(nodeset, obj->os_index) < 0)
            return -1;
    return 0;
}

/** \brief Convert a NUMA node set into a CPU set
 *
 * For each NUMA node included in the input \p nodeset, set the corresponding
 * local PUs in the output \p _cpuset.
 *
 * If some CPUs have no local NUMA nodes, this function never sets their
 * indexes in the output CPU set, even if a full node set is given in input.
 *
 * Hence the entire topology node set is converted into the set of all CPUs
 * that have some local NUMA nodes.
 */
static __hwloc_inline int
hwloc_cpuset_from_nodeset(hwloc_topology_t topology, hwloc_cpuset_t _cpuset, hwloc_const_nodeset_t nodeset)
{
    // 获取 NUMA 节点的深度
    int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
    // 初始化 obj 为 NULL
    hwloc_obj_t obj = NULL;
    // 确保获取到了 NUMA 节点的深度
    assert(depth != HWLOC_TYPE_DEPTH_UNKNOWN);
    // 将 _cpuset 清零
    hwloc_bitmap_zero(_cpuset);
    # 通过指定深度遍历拓扑结构中的每个对象
    while ((obj = hwloc_get_next_obj_by_depth(topology, depth, obj)) != NULL) {
        # 检查节点集中是否设置了指定对象的索引
        if (hwloc_bitmap_isset(nodeset, obj->os_index))
            # 不需要检查对象的 cpuset，因为层级中的对象总是有一个 cpuset
            # 将对象的 cpuset 与给定的 cpuset 求并集
            if (hwloc_bitmap_or(_cpuset, _cpuset, obj->cpuset) < 0)
                # 如果求并集失败，则返回 -1
                return -1;
    }
    # 遍历完成后返回 0
    return 0;
}

/** @} */



/** \defgroup hwlocality_advanced_io Finding I/O objects
 * @{
 */

/** \brief Get the first non-I/O ancestor object.
 *
 * Given the I/O object \p ioobj, find the smallest non-I/O ancestor
 * object. This object (normal or memory) may then be used for binding
 * because it has non-NULL CPU and node sets
 * and because its locality is the same as \p ioobj.
 *
 * \note The resulting object is usually a normal object but it could also
 * be a memory object (e.g. NUMA node) in future platforms if I/O objects
 * ever get attached to memory instead of CPUs.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_non_io_ancestor_obj(hwloc_topology_t topology __hwloc_attribute_unused,
                  hwloc_obj_t ioobj)
{
  hwloc_obj_t obj = ioobj;
  while (obj && !obj->cpuset) {
    obj = obj->parent;
  }
  return obj;
}

/** \brief Get the next PCI device in the system.
 *
 * \return the first PCI device if \p prev is \c NULL.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_pcidev(hwloc_topology_t topology, hwloc_obj_t prev)
{
  return hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_PCI_DEVICE, prev);
}

/** \brief Find the PCI device object matching the PCI bus id
 * given domain, bus device and function PCI bus id.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_pcidev_by_busid(hwloc_topology_t topology,
              unsigned domain, unsigned bus, unsigned dev, unsigned func)
{
  hwloc_obj_t obj = NULL;
  while ((obj = hwloc_get_next_pcidev(topology, obj)) != NULL) {
    if (obj->attr->pcidev.domain == domain
    && obj->attr->pcidev.bus == bus
    && obj->attr->pcidev.dev == dev
    && obj->attr->pcidev.func == func)
      return obj;
  }
  return NULL;
}

/** \brief Find the PCI device object matching the PCI bus id
 * given as a string xxxx:yy:zz.t or yy:zz.t.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_pcidev_by_busidstring(hwloc_topology_t topology, const char *busid)
{
  unsigned domain = 0; /* 默认值 */
  unsigned bus, dev, func;

  // 从 busid 中解析出总线、设备和函数号
  if (sscanf(busid, "%x:%x.%x", &bus, &dev, &func) != 3
      && sscanf(busid, "%x:%x:%x.%x", &domain, &bus, &dev, &func) != 4) {
    // 如果解析失败，设置错误码并返回空指针
    errno = EINVAL;
    return NULL;
  }

  // 根据总线、设备和函数号获取对应的 PCI 设备对象
  return hwloc_get_pcidev_by_busid(topology, domain, bus, dev, func);
}

/** \brief 获取系统中的下一个 OS 设备。
 *
 * \return 如果 \p prev 为 \c NULL，则返回第一个 OS 设备。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_osdev(hwloc_topology_t topology, hwloc_obj_t prev)
{
  // 获取下一个指定类型的对象，这里是 OS 设备
  return hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_OS_DEVICE, prev);
}

/** \brief 获取系统中的下一个桥接设备。
 *
 * \return 如果 \p prev 为 \c NULL，则返回第一个桥接设备。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_bridge(hwloc_topology_t topology, hwloc_obj_t prev)
{
  // 获取下一个指定类型的对象，这里是桥接设备
  return hwloc_get_next_obj_by_type(topology, HWLOC_OBJ_BRIDGE, prev);
}

/* \brief 检查给定的桥接设备是否覆盖给定的 PCI 总线。
 */
static __hwloc_inline int
hwloc_bridge_covers_pcibus(hwloc_obj_t bridge,
               unsigned domain, unsigned bus)
{
  // 检查桥接设备的类型和下游设备是否为 PCI 设备，并且是否覆盖给定的 PCI 总线
  return bridge->type == HWLOC_OBJ_BRIDGE
    && bridge->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI
    && bridge->attr->bridge.downstream.pci.domain == domain
    && bridge->attr->bridge.downstream.pci.secondary_bus <= bus
    && bridge->attr->bridge.downstream.pci.subordinate_bus >= bus;
}

/** @} */



#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_HELPER_H */
```