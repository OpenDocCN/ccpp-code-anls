# `xmrig\src\3rdparty\hwloc\include\hwloc\inlines.h`

```
/*
 * 版权声明
 */
 
/**
 * 该文件包含在 hwloc.h 中声明的内联函数的内联代码
 */

#ifndef HWLOC_INLINES_H
#define HWLOC_INLINES_H

#ifndef HWLOC_H
#error 请包含主要的 hwloc.h 文件
#endif

#include <stdlib.h>
#include <errno.h>


#ifdef __cplusplus
extern "C" {
#endif

static __hwloc_inline int
hwloc_get_type_or_below_depth (hwloc_topology_t topology, hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);

  if (depth != HWLOC_TYPE_DEPTH_UNKNOWN)
    return depth;

  /* 找到具有类型顺序大于等于的最高现有级别 */
  for(depth = hwloc_get_type_depth(topology, HWLOC_OBJ_PU); ; depth--)
    if (hwloc_compare_types(hwloc_get_depth_type(topology, depth), type) < 0)
      return depth+1;

  /* 不应该发生，因为始终存在一个具有较低顺序和已知深度的 Machine 级别。 */
  /* abort(); */
}

static __hwloc_inline int
hwloc_get_type_or_above_depth (hwloc_topology_t topology, hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);

  if (depth != HWLOC_TYPE_DEPTH_UNKNOWN)
    return depth;

  /* 找到具有类型顺序小于等于的最低现有级别 */
  for(depth = 0; ; depth++)
    if (hwloc_compare_types(hwloc_get_depth_type(topology, depth), type) > 0)
      return depth-1;

  /* 不应该发生，因为始终存在一个具有较高顺序和已知深度的 PU 级别。 */
  /* abort(); */
}

static __hwloc_inline int
hwloc_get_nbobjs_by_type (hwloc_topology_t topology, hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
    return 0;
  if (depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return -1; /* FIXME: agregate nbobjs from different levels? */
  return (int) hwloc_get_nbobjs_by_depth(topology, depth);
}
// 根据给定类型和索引获取相应的对象
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_by_type (hwloc_topology_t topology, hwloc_obj_type_t type, unsigned idx)
{
  // 获取给定类型的深度
  int depth = hwloc_get_type_depth(topology, type);
  // 如果深度未知，则返回空
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
    return NULL;
  // 如果深度为多个，则返回空
  if (depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  // 根据深度和索引获取对象
  return hwloc_get_obj_by_depth(topology, depth, idx);
}

// 根据深度和前一个对象获取下一个对象
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_by_depth (hwloc_topology_t topology, int depth, hwloc_obj_t prev)
{
  // 如果前一个对象为空，则根据深度和索引获取对象
  if (!prev)
    return hwloc_get_obj_by_depth (topology, depth, 0);
  // 如果前一个对象的深度不等于给定深度，则返回空
  if (prev->depth != depth)
    return NULL;
  // 返回前一个对象的下一个兄弟对象
  return prev->next_cousin;
}

// 根据类型和前一个对象获取下一个对象
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_by_type (hwloc_topology_t topology, hwloc_obj_type_t type,
                hwloc_obj_t prev)
{
  // 获取给定类型的深度
  int depth = hwloc_get_type_depth(topology, type);
  // 如果深度未知或者为多个，则返回空
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  // 根据深度和前一个对象获取下一个对象
  return hwloc_get_next_obj_by_depth (topology, depth, prev);
}

// 获取根对象
static __hwloc_inline hwloc_obj_t
hwloc_get_root_obj (hwloc_topology_t topology)
{
  // 根据深度和索引获取对象
  return hwloc_get_obj_by_depth (topology, 0, 0);
}

// 根据名称获取对象的信息
static __hwloc_inline const char *
hwloc_obj_get_info_by_name(hwloc_obj_t obj, const char *name)
{
  unsigned i;
  // 遍历对象的信息列表
  for(i=0; i<obj->infos_count; i++) {
    struct hwloc_info_s *info = &obj->infos[i];
    // 如果信息的名称匹配给定名称，则返回信息的值
    if (!strcmp(info->name, name))
      return info->value;
  }
  // 如果未找到匹配的信息，则返回空
  return NULL;
}

// 分配内存并绑定策略
static __hwloc_inline void *
hwloc_alloc_membind_policy(hwloc_topology_t topology, size_t len, hwloc_const_cpuset_t set, hwloc_membind_policy_t policy, int flags)
{
  // 分配内存并绑定策略
  void *p = hwloc_alloc_membind(topology, len, set, policy, flags);
  // 如果成功分配内存，则返回指针
  if (p)
    return p;

  // 如果分配失败，则尝试直接绑定策略
  if (hwloc_set_membind(topology, set, policy, flags) < 0)
    /* hwloc_set_membind() takes care of ignoring errors if non-STRICT */
    return NULL;

  // 如果直接绑定成功，则分配内存并返回指针
  p = hwloc_alloc(topology, len);
  // 如果分配成功且策略不是首次接触，则通过访问数据来强制绑定
  if (p && policy != HWLOC_MEMBIND_FIRSTTOUCH)
    /* Enforce the binding by touching the data */
    memset(p, 0, len);
  return p;
}

// 如果是 C++ 代码
#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_INLINES_H */
```