# `xmrig\src\3rdparty\hwloc\src\traversal.c`

```cpp
/*
 * 版权声明
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2021 Inria
 * 版权所有 © 2009-2010, 2020 Université Bordeaux
 * 版权所有 © 2009-2011 Cisco Systems, Inc.
 * 请参阅顶层目录中的 COPYING 文件。
 */

#include "private/autogen/config.h" // 包含自动生成的配置文件
#include "hwloc.h" // 包含 hwloc 头文件
#include "private/private.h" // 包含私有头文件
#include "private/misc.h" // 包含私有杂项头文件
#include "private/debug.h" // 包含私有调试头文件

#ifdef HAVE_STRINGS_H
#include <strings.h> // 如果有 strings.h 头文件，则包含它
#endif /* HAVE_STRINGS_H */

int
hwloc_get_type_depth (struct hwloc_topology *topology, hwloc_obj_type_t type)
{
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_TYPE_MIN == 0); // 断言 HWLOC_OBJ_TYPE_MIN 等于 0
  if ((unsigned) type >= HWLOC_OBJ_TYPE_MAX) // 如果 type 大于等于 HWLOC_OBJ_TYPE_MAX
    return HWLOC_TYPE_DEPTH_UNKNOWN; // 返回 HWLOC_TYPE_DEPTH_UNKNOWN
  else
    return topology->type_depth[type]; // 返回 topology 中 type 对应的深度
}

hwloc_obj_type_t
hwloc_get_depth_type (hwloc_topology_t topology, int depth)
{
  if ((unsigned)depth >= topology->nb_levels) // 如果 depth 大于等于 topology 中的层级数
    switch (depth) { // 开始 switch 语句
    case HWLOC_TYPE_DEPTH_NUMANODE:
      return HWLOC_OBJ_NUMANODE; // 返回 HWLOC_OBJ_NUMANODE
    case HWLOC_TYPE_DEPTH_BRIDGE:
      return HWLOC_OBJ_BRIDGE; // 返回 HWLOC_OBJ_BRIDGE
    case HWLOC_TYPE_DEPTH_PCI_DEVICE:
      return HWLOC_OBJ_PCI_DEVICE; // 返回 HWLOC_OBJ_PCI_DEVICE
    case HWLOC_TYPE_DEPTH_OS_DEVICE:
      return HWLOC_OBJ_OS_DEVICE; // 返回 HWLOC_OBJ_OS_DEVICE
    case HWLOC_TYPE_DEPTH_MISC:
      return HWLOC_OBJ_MISC; // 返回 HWLOC_OBJ_MISC
    case HWLOC_TYPE_DEPTH_MEMCACHE:
      return HWLOC_OBJ_MEMCACHE; // 返回 HWLOC_OBJ_MEMCACHE
    default:
      return HWLOC_OBJ_TYPE_NONE; // 返回 HWLOC_OBJ_TYPE_NONE
    }
  return topology->levels[depth][0]->type; // 返回 topology 中指定深度的第一个对象的类型
}

int
hwloc_get_memory_parents_depth (hwloc_topology_t topology)
{
  int depth = HWLOC_TYPE_DEPTH_UNKNOWN; // 初始化深度为 HWLOC_TYPE_DEPTH_UNKNOWN
  /* memory leaves are always NUMA nodes for now, no need to check parents of other memory types */
  hwloc_obj_t numa = hwloc_get_obj_by_depth(topology, HWLOC_TYPE_DEPTH_NUMANODE, 0); // 获取 NUMA 节点对象
  assert(numa); // 断言 NUMA 节点对象存在
  while (numa) { // 开始 while 循环
    hwloc_obj_t parent = numa->parent; // 获取 NUMA 节点对象的父对象
    /* walk-up the memory hierarchy */
    while (hwloc__obj_type_is_memory(parent->type)) // 如果父对象是内存类型
      parent = parent->parent; // 继续向上查找父对象

    if (depth == HWLOC_TYPE_DEPTH_UNKNOWN) // 如果深度仍然是 HWLOC_TYPE_DEPTH_UNKNOWN
      depth = parent->depth; // 将深度设置为父对象的深度
    # 如果深度不等于父节点的深度，则返回深度多个值
    else if (depth != parent->depth)
      return HWLOC_TYPE_DEPTH_MULTIPLE;

    # 移动到下一个同级节点
    numa = numa->next_cousin;
  }

  # 断言深度大于等于0
  assert(depth >= 0);
  # 返回深度
  return depth;
}

unsigned
hwloc_get_nbobjs_by_depth (struct hwloc_topology *topology, int depth)
{
  // 如果深度超出拓扑结构的层级数，返回0
  if ((unsigned)depth >= topology->nb_levels) {
    // 将深度转换为特殊层级的索引
    unsigned l = HWLOC_SLEVEL_FROM_DEPTH(depth);
    // 如果特殊层级索引小于特殊层级数，返回该特殊层级的对象数
    if (l < HWLOC_NR_SLEVELS)
      return topology->slevels[l].nbobjs;
    else
      return 0;
  }
  // 返回指定深度的对象数
  return topology->level_nbobjects[depth];
}

struct hwloc_obj *
hwloc_get_obj_by_depth (struct hwloc_topology *topology, int depth, unsigned idx)
{
  // 如果深度超出拓扑结构的层级数
  if ((unsigned)depth >= topology->nb_levels) {
    // 将深度转换为特殊层级的索引
    unsigned l = HWLOC_SLEVEL_FROM_DEPTH(depth);
    // 如果特殊层级索引小于特殊层级数，返回该特殊层级的指定索引的对象，否则返回空指针
    if (l < HWLOC_NR_SLEVELS)
      return idx < topology->slevels[l].nbobjs ? topology->slevels[l].objs[idx] : NULL;
    else
      return NULL;
  }
  // 如果指定索引超出指定深度的对象数，返回空指针，否则返回指定深度和索引的对象
  if (idx >= topology->level_nbobjects[depth])
    return NULL;
  return topology->levels[depth][idx];
}

// 判断对象类型是否为普通类型
int
hwloc_obj_type_is_normal(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_normal(type);
}

// 判断对象类型是否为内存类型
int
hwloc_obj_type_is_memory(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_memory(type);
}

// 判断对象类型是否为IO类型
int
hwloc_obj_type_is_io(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_io(type);
}

// 判断对象类型是否为缓存类型
int
hwloc_obj_type_is_cache(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_cache(type);
}

// 判断对象类型是否为数据缓存类型
int
hwloc_obj_type_is_dcache(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_dcache(type);
}

// 判断对象类型是否为指令缓存类型
int
hwloc_obj_type_is_icache(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_icache(type);
}

// 根据深度和全局索引获取对象
static hwloc_obj_t hwloc_get_obj_by_depth_and_gp_index(hwloc_topology_t topology, unsigned depth, uint64_t gp_index)
{
  // 获取指定深度和索引的对象
  hwloc_obj_t obj = hwloc_get_obj_by_depth(topology, depth, 0);
  // 遍历对象链表，直到找到指定全局索引的对象或遍历完链表
  while (obj) {
    if (obj->gp_index == gp_index)
      return obj;
    obj = obj->next_cousin;
  }
  return NULL;
}

// 根据类型和全局索引获取对象
hwloc_obj_t hwloc_get_obj_by_type_and_gp_index(hwloc_topology_t topology, hwloc_obj_type_t type, uint64_t gp_index)
{
  // 获取指定类型的深度
  int depth = hwloc_get_type_depth(topology, type);
  // 如果深度未知，返回空指针
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
    return NULL;
  // 如果深度为多个层级，返回空指针
  if (depth == HWLOC_TYPE_DEPTH_MULTIPLE) {
    for(depth=1 /* no multiple machine levels */;
    # 检查深度是否小于拓扑结构中的层级数减一，即没有多个处理器单元级别
    (unsigned) depth < topology->nb_levels-1 /* no multiple PU levels */;
    # 增加深度计数
    depth++) {
      # 如果深度对应的类型与给定类型相同
      if (hwloc_get_depth_type(topology, depth) == type) {
    # 获取指定深度和全局索引的对象
    hwloc_obj_t obj = hwloc_get_obj_by_depth_and_gp_index(topology, depth, gp_index);
    # 如果对象存在，则返回该对象
    if (obj)
      return obj;
      }
    }
    # 如果循环结束仍未找到对象，则返回空
    return NULL;
  }
  # 如果深度超出范围，则返回指定深度和全局索引的对象
  return hwloc_get_obj_by_depth_and_gp_index(topology, depth, gp_index);
}

unsigned hwloc_get_closest_objs (struct hwloc_topology *topology, struct hwloc_obj *src, struct hwloc_obj **objs, unsigned max)
{
  struct hwloc_obj *parent, *nextparent, **src_objs;
  unsigned i,src_nbobjects;
  unsigned stored = 0;

  if (!src->cpuset)
    return 0;

  src_nbobjects = topology->level_nbobjects[src->depth];
  src_objs = topology->levels[src->depth];

  parent = src;
  while (stored < max) {
    while (1) {
      nextparent = parent->parent;
      if (!nextparent)
    goto out;
      if (!hwloc_bitmap_isequal(parent->cpuset, nextparent->cpuset))
    break;
      parent = nextparent;
    }

    /* traverse src's objects and find those that are in nextparent and were not in parent */
    for(i=0; i<src_nbobjects; i++) {
      if (hwloc_bitmap_isincluded(src_objs[i]->cpuset, nextparent->cpuset)
      && !hwloc_bitmap_isincluded(src_objs[i]->cpuset, parent->cpuset)) {
    objs[stored++] = src_objs[i];
    if (stored == max)
      goto out;
      }
    }
    parent = nextparent;
  }

 out:
  return stored;
}

static int
hwloc__get_largest_objs_inside_cpuset (struct hwloc_obj *current, hwloc_const_bitmap_t set,
                       struct hwloc_obj ***res, int *max)
{
  int gotten = 0;
  unsigned i;

  /* the caller must ensure this */
  if (*max <= 0)
    return 0;

  if (hwloc_bitmap_isequal(current->cpuset, set)) {
    **res = current;
    (*res)++;
    (*max)--;
    return 1;
  }

  for (i=0; i<current->arity; i++) {
    hwloc_bitmap_t subset;
    int ret;

    /* split out the cpuset part corresponding to this child and see if there's anything to do */
    if (!hwloc_bitmap_intersects(set,current->children[i]->cpuset))
      continue;

    subset = hwloc_bitmap_dup(set);
    hwloc_bitmap_and(subset, subset, current->children[i]->cpuset);
    ret = hwloc__get_largest_objs_inside_cpuset (current->children[i], subset, res, max);
    gotten += ret;
    hwloc_bitmap_free(subset);
    /* 如果没有足够的空间来存储剩余的对象，就返回目前为止得到的内容 */
    if (!*max)
      break;
  }
  // 返回已获取的内容
  return gotten;
}

int
hwloc_get_largest_objs_inside_cpuset (struct hwloc_topology *topology, hwloc_const_bitmap_t set,
                      struct hwloc_obj **objs, int max)
{
  // 获取拓扑结构中的第一个对象
  struct hwloc_obj *current = topology->levels[0][0];

  // 如果给定的集合不包含当前对象的 cpuset，则返回 -1
  if (!hwloc_bitmap_isincluded(set, current->cpuset))
    return -1;

  // 如果 max 小于等于 0，则返回 0
  if (max <= 0)
    return 0;

  // 调用内部函数，获取给定集合中最大的对象
  return hwloc__get_largest_objs_inside_cpuset (current, set, &objs, &max);
}

// 返回对象类型的字符串表示
const char *
hwloc_obj_type_string (hwloc_obj_type_t obj)
{
  switch (obj)
    {
    // 根据对象类型返回对应的字符串
    case HWLOC_OBJ_MACHINE: return "Machine";
    case HWLOC_OBJ_MISC: return "Misc";
    case HWLOC_OBJ_GROUP: return "Group";
    case HWLOC_OBJ_MEMCACHE: return "MemCache";
    case HWLOC_OBJ_NUMANODE: return "NUMANode";
    case HWLOC_OBJ_PACKAGE: return "Package";
    case HWLOC_OBJ_DIE: return "Die";
    case HWLOC_OBJ_L1CACHE: return "L1Cache";
    case HWLOC_OBJ_L2CACHE: return "L2Cache";
    case HWLOC_OBJ_L3CACHE: return "L3Cache";
    case HWLOC_OBJ_L4CACHE: return "L4Cache";
    case HWLOC_OBJ_L5CACHE: return "L5Cache";
    case HWLOC_OBJ_L1ICACHE: return "L1iCache";
    case HWLOC_OBJ_L2ICACHE: return "L2iCache";
    case HWLOC_OBJ_L3ICACHE: return "L3iCache";
    case HWLOC_OBJ_CORE: return "Core";
    case HWLOC_OBJ_BRIDGE: return "Bridge";
    case HWLOC_OBJ_PCI_DEVICE: return "PCIDev";
    case HWLOC_OBJ_OS_DEVICE: return "OSDev";
    case HWLOC_OBJ_PU: return "PU";
    default: return "Unknown";
    }
}

/* 检查字符串是否至少在 minmatch 个字符上与给定类型匹配。
 * 成功时，返回匹配停止的地址，指向 \0 或后缀（数字、冒号等）
 * 失败时，返回 NULL
 */
static __hwloc_inline const char *
hwloc__type_match(const char *string,
          const char *type, /* type must be lowercase */
          size_t minmatch)
{
  const char *s, *t;
  unsigned i;
  for(i=0, s=string, t=type; ; i++, s++, t++) {
    if (!*s) {
      /* 字符串在类型之前结束 */
      if (i<minmatch)
    return NULL;
      else
    return s;
    }
    # 如果字符串s和t的当前字符不相等，并且s的当前字符不是t的当前字符加上'A'减去'a'的结果
    if (*s != *t && *s != *t + 'A' - 'a') {
      /* 字符串不同 */
      # 如果s的当前字符是小写字母、大写字母或者短横线
      if ((*s >= 'a' && *s <= 'z') || (*s >= 'A' && *s <= 'Z') || *s == '-')
    /* 有效字符，但不匹配 */
    # 返回空指针
    return NULL;
      /* 无效字符，我们已经到达字符串中类型名称的末尾，停止匹配 */
      # 如果i小于最小匹配长度
      if (i<minmatch)
    # 返回空指针
    return NULL;
      else
    # 返回s的当前位置
    return s;
    }
  }

  # 返回空指针
  return NULL;
  // 定义一个名为 hwloc_type_sscanf 的函数，接受一个字符串、一个指向 hwloc_obj_type_t 类型的指针、一个指向 union hwloc_obj_attr_u 类型的指针和一个表示 attrp 的大小的参数
int
hwloc_type_sscanf(const char *string, hwloc_obj_type_t *typep,
          union hwloc_obj_attr_u *attrp, size_t attrsize)
{
  // 初始化 type 为 -1
  hwloc_obj_type_t type = (hwloc_obj_type_t) -1;
  // 初始化 depthattr 为 -1
  unsigned depthattr = (unsigned) -1;
  // 初始化 cachetypeattr 为未指定
  hwloc_obj_cache_type_t cachetypeattr = (hwloc_obj_cache_type_t) -1; /* unspecified */
  // 初始化 ubtype 为 -1
  hwloc_obj_bridge_type_t ubtype = (hwloc_obj_bridge_type_t) -1;
  // 初始化 ostype 为 -1
  hwloc_obj_osdev_type_t ostype = (hwloc_obj_osdev_type_t) -1;
  // 定义一个指向字符的指针 end
  char *end;

  /* Never match the ending \0 since we want to match things like core:2 too.
   * We'll only compare the beginning substring only made of letters and dash.
   */

  /* types without a custom depth */

  // 如果字符串匹配 "osdev"，则将 type 设置为 HWLOC_OBJ_OS_DEVICE
  if (hwloc__type_match(string, "osdev", 2)) {
    type = HWLOC_OBJ_OS_DEVICE;
  } else if (hwloc__type_match(string, "block", 4)) {
    type = HWLOC_OBJ_OS_DEVICE;
    // 将 ostype 设置为 HWLOC_OBJ_OSDEV_BLOCK
    ostype = HWLOC_OBJ_OSDEV_BLOCK;
  } else if (hwloc__type_match(string, "network", 3)) {
    type = HWLOC_OBJ_OS_DEVICE;
    // 将 ostype 设置为 HWLOC_OBJ_OSDEV_NETWORK
    ostype = HWLOC_OBJ_OSDEV_NETWORK;
  } else if (hwloc__type_match(string, "openfabrics", 7)) {
    type = HWLOC_OBJ_OS_DEVICE;
    // 将 ostype 设置为 HWLOC_OBJ_OSDEV_OPENFABRICS
    ostype = HWLOC_OBJ_OSDEV_OPENFABRICS;
  } else if (hwloc__type_match(string, "dma", 3)) {
    type = HWLOC_OBJ_OS_DEVICE;
    // 将 ostype 设置为 HWLOC_OBJ_OSDEV_DMA
    ostype = HWLOC_OBJ_OSDEV_DMA;
  } else if (hwloc__type_match(string, "gpu", 3)) {
    type = HWLOC_OBJ_OS_DEVICE;
    // 将 ostype 设置为 HWLOC_OBJ_OSDEV_GPU
    ostype = HWLOC_OBJ_OSDEV_GPU;
  } else if (hwloc__type_match(string, "coproc", 5)
         || hwloc__type_match(string, "co-processor", 6)) {
    type = HWLOC_OBJ_OS_DEVICE;
    // 将 ostype 设置为 HWLOC_OBJ_OSDEV_COPROC
    ostype = HWLOC_OBJ_OSDEV_COPROC;

  } else if (hwloc__type_match(string, "machine", 2)) {
    // 将 type 设置为 HWLOC_OBJ_MACHINE
    type = HWLOC_OBJ_MACHINE;
  } else if (hwloc__type_match(string, "numanode", 2)
         || hwloc__type_match(string, "node", 2)) { /* for convenience */
    // 将 type 设置为 HWLOC_OBJ_NUMANODE
    type = HWLOC_OBJ_NUMANODE;
  } else if (hwloc__type_match(string, "memcache", 5)
         || hwloc__type_match(string, "memory-side cache", 8)) {
    # 设置类型为内存缓存
    type = HWLOC_OBJ_MEMCACHE;
  } else if (hwloc__type_match(string, "package", 2)
         || hwloc__type_match(string, "socket", 2)) { # 与 v1.10 向后兼容
    # 设置类型为处理器包
    type = HWLOC_OBJ_PACKAGE;
  } else if (hwloc__type_match(string, "die", 2)) {
    # 设置类型为处理器核心组
    type = HWLOC_OBJ_DIE;
  } else if (hwloc__type_match(string, "core", 2)) {
    # 设置类型为处理器核心
    type = HWLOC_OBJ_CORE;
  } else if (hwloc__type_match(string, "pu", 2)) {
    # 设置类型为处理器单元
    type = HWLOC_OBJ_PU;
  } else if (hwloc__type_match(string, "misc", 4)) {
    # 设置类型为其他未知对象
    type = HWLOC_OBJ_MISC;

  } else if (hwloc__type_match(string, "bridge", 4)) {
    # 设置类型为桥接设备
    type = HWLOC_OBJ_BRIDGE;
  } else if (hwloc__type_match(string, "hostbridge", 6)) {
    # 设置类型为桥接设备，子类型为主机桥
    type = HWLOC_OBJ_BRIDGE;
    ubtype = HWLOC_OBJ_BRIDGE_HOST;
  } else if (hwloc__type_match(string, "pcibridge", 5)) {
    # 设置类型为桥接设备，子类型为 PCI 桥
    type = HWLOC_OBJ_BRIDGE;
    ubtype = HWLOC_OBJ_BRIDGE_PCI;
    # 如果 downstream_type 可能不是 PCI，则需要使字符串更精确，或者放宽 hwloc_type_sscanf 测试

  } else if (hwloc__type_match(string, "pcidev", 3)) {
    # 设置类型为 PCI 设备
    type = HWLOC_OBJ_PCI_DEVICE;

  # 具有深度属性的类型
  } else if ((string[0] == 'l' || string[0] == 'L') && string[1] >= '0' && string[1] <= '9') {
    char *suffix;
    # 从字符串中提取深度属性
    depthattr = strtol(string+1, &end, 10);
    if (*end == 'i' || *end == 'I') {
      if (depthattr >= 1 && depthattr <= 3) {
    # 设置类型为 L1 指令缓存
    type = HWLOC_OBJ_L1ICACHE + depthattr-1;
    cachetypeattr = HWLOC_OBJ_CACHE_INSTRUCTION;
    suffix = end+1;
      } else
    return -1;
    } else {
      if (depthattr >= 1 && depthattr <= 5) {
    # 设置类型为 L1 数据缓存或统一缓存
    type = HWLOC_OBJ_L1CACHE + depthattr-1;
    if (*end == 'd' || *end == 'D') {
      cachetypeattr = HWLOC_OBJ_CACHE_DATA;
      suffix = end+1;
    } else if (*end == 'u' || *end == 'U') {
      cachetypeattr = HWLOC_OBJ_CACHE_UNIFIED;
      suffix = end+1;
    } else {
      cachetypeattr = HWLOC_OBJ_CACHE_UNIFIED;
      suffix = end;
    }
      } else
    return -1;
    }
    # 检查可选后缀是否匹配 "cache"
    # 如果后缀不是"cache"，则返回-1
    if (!hwloc__type_match(suffix, "cache", 0))
      return -1;

  # 如果字符串中包含"group"，则执行以下操作
  } else if ((end = (char *) hwloc__type_match(string, "group", 2)) != NULL) {
    # 设置类型为HWLOC_OBJ_GROUP
    type = HWLOC_OBJ_GROUP;
    # 如果end指向的字符是数字，则将其转换为整数并赋给depthattr
    if (*end >= '0' && *end <= '9') {
      depthattr = strtol(end, &end, 10);
    }

  # 如果以上条件都不满足，则返回-1
  } else
    return -1;

  # 将type赋给typep
  *typep = type;
  # 如果attrp不为空，则执行以下操作
  if (attrp) {
    # 如果类型是缓存并且attrsize大于等于attrp->cache的大小，则设置缓存的深度和类型
    if (hwloc__obj_type_is_cache(type) && attrsize >= sizeof(attrp->cache)) {
      attrp->cache.depth = depthattr;
      attrp->cache.type = cachetypeattr;
    # 如果类型是组并且attrsize大于等于attrp->group的大小，则设置组的深度
    } else if (type == HWLOC_OBJ_GROUP && attrsize >= sizeof(attrp->group)) {
      attrp->group.depth = depthattr;
    # 如果类型是桥接设备并且attrsize大于等于attrp->bridge的大小，则设置上游和下游类型
    } else if (type == HWLOC_OBJ_BRIDGE && attrsize >= sizeof(attrp->bridge)) {
      attrp->bridge.upstream_type = ubtype;
      attrp->bridge.downstream_type = HWLOC_OBJ_BRIDGE_PCI;
      # 如果下游类型可以是非PCI类型，需要更精确地设置字符串，或者放宽hwloc_type_sscanf测试的条件
    # 如果类型是操作系统设备并且attrsize大于等于attrp->osdev的大小，则设置操作系统设备的类型
    } else if (type == HWLOC_OBJ_OS_DEVICE && attrsize >= sizeof(attrp->osdev)) {
      attrp->osdev.type = ostype;
    }
  }
  # 返回0
  return 0;
}
// 定义一个函数，根据字符串解析出对应的对象类型和深度
int
hwloc_type_sscanf_as_depth(const char *string, hwloc_obj_type_t *typep,
               hwloc_topology_t topology, int *depthp)
{
  union hwloc_obj_attr_u attr; // 定义一个对象属性联合体
  hwloc_obj_type_t type; // 定义一个对象类型变量
  int depth; // 定义一个深度变量
  int err; // 定义一个错误码变量

  // 调用hwloc_type_sscanf函数，解析字符串，获取对象类型和属性
  err = hwloc_type_sscanf(string, &type, &attr, sizeof(attr));
  if (err < 0) // 如果解析失败，返回错误码
    return err;

  // 获取对象类型的深度
  depth = hwloc_get_type_depth(topology, type);
  // 如果对象类型是组(Group)并且深度是多重的，并且属性中的深度不是-1
  if (type == HWLOC_OBJ_GROUP
      && depth == HWLOC_TYPE_DEPTH_MULTIPLE
      && attr.group.depth != (unsigned)-1) {
    unsigned l; // 定义一个无符号整数变量
    depth = HWLOC_TYPE_DEPTH_UNKNOWN; // 将深度设置为未知
    for(l=0; l<topology->nb_levels; l++) { // 遍历拓扑结构中的层级
      if (topology->levels[l][0]->type == HWLOC_OBJ_GROUP
      && topology->levels[l][0]->attr->group.depth == attr.group.depth) {
    depth = (int)l; // 如果找到匹配的深度，将深度设置为l
    break;
      }
    }
  }

  if (typep) // 如果对象类型指针不为空
    *typep = type; // 将对象类型赋值给指针指向的变量
  *depthp = depth; // 将深度赋值给深度指针指向的变量
  return 0; // 返回成功
}

// 定义一个静态函数，根据缓存类型返回对应的字母
static const char* hwloc_obj_cache_type_letter(hwloc_obj_cache_type_t type)
{
  switch (type) { // 根据缓存类型进行切换
  case HWLOC_OBJ_CACHE_UNIFIED: return ""; // 统一缓存返回空字符串
  case HWLOC_OBJ_CACHE_DATA: return "d"; // 数据缓存返回d
  case HWLOC_OBJ_CACHE_INSTRUCTION: return "i"; // 指令缓存返回i
  default: return "unknown"; // 其他情况返回unknown
  }
}

// 定义一个函数，根据对象类型格式化输出字符串
int
hwloc_obj_type_snprintf(char * __hwloc_restrict string, size_t size, hwloc_obj_t obj, int verbose)
{
  hwloc_obj_type_t type = obj->type; // 获取对象的类型
  switch (type) { // 根据对象类型进行切换
  case HWLOC_OBJ_MISC: // 杂项类型
  case HWLOC_OBJ_MACHINE: // 机器类型
  case HWLOC_OBJ_NUMANODE: // NUMA节点类型
  case HWLOC_OBJ_MEMCACHE: // 内存缓存类型
  case HWLOC_OBJ_PACKAGE: // 处理器包类型
  case HWLOC_OBJ_DIE: // 处理器内核类型
  case HWLOC_OBJ_CORE: // 处理器核心类型
  case HWLOC_OBJ_PU: // 处理器类型
    return hwloc_snprintf(string, size, "%s", hwloc_obj_type_string(type)); // 格式化输出对象类型字符串
  case HWLOC_OBJ_L1CACHE: // 一级缓存类型
  case HWLOC_OBJ_L2CACHE: // 二级缓存类型
  case HWLOC_OBJ_L3CACHE: // 三级缓存类型
  case HWLOC_OBJ_L4CACHE: // 四级缓存类型
  case HWLOC_OBJ_L5CACHE: // 五级缓存类型
  case HWLOC_OBJ_L1ICACHE: // 一级指令缓存类型
  case HWLOC_OBJ_L2ICACHE: // 二级指令缓存类型
  case HWLOC_OBJ_L3ICACHE: // 三级指令缓存类型
    return hwloc_snprintf(string, size, "L%u%s%s", obj->attr->cache.depth,
              hwloc_obj_cache_type_letter(obj->attr->cache.type),
              verbose ? "Cache" : ""); // 格式化输出缓存类型字符串
  case HWLOC_OBJ_GROUP: // 组类型
    # 如果对象的组深度不是无符号整数的最大值，则返回对象类型字符串和组深度的格式化字符串
    if (obj->attr->group.depth != (unsigned) -1)
      return hwloc_snprintf(string, size, "%s%u", hwloc_obj_type_string(type), obj->attr->group.depth);
    # 否则，返回对象类型字符串的格式化字符串
    else
      return hwloc_snprintf(string, size, "%s", hwloc_obj_type_string(type));
  case HWLOC_OBJ_BRIDGE:
    # 断言下游类型为 PCI 类型
    assert(obj->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI);
    # 返回桥接设备的类型字符串
    return hwloc_snprintf(string, size, obj->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI ? "PCIBridge" : "HostBridge");
  case HWLOC_OBJ_PCI_DEVICE:
    # 返回 PCI 设备的类型字符串
    return hwloc_snprintf(string, size, "PCI");
  case HWLOC_OBJ_OS_DEVICE:
    # 根据 OS 设备的类型返回相应的类型字符串
    switch (obj->attr->osdev.type) {
    case HWLOC_OBJ_OSDEV_BLOCK: return hwloc_snprintf(string, size, "Block");
    case HWLOC_OBJ_OSDEV_NETWORK: return hwloc_snprintf(string, size, verbose ? "Network" : "Net");
    case HWLOC_OBJ_OSDEV_OPENFABRICS: return hwloc_snprintf(string, size, "OpenFabrics");
    case HWLOC_OBJ_OSDEV_DMA: return hwloc_snprintf(string, size, "DMA");
    case HWLOC_OBJ_OSDEV_GPU: return hwloc_snprintf(string, size, "GPU");
    case HWLOC_OBJ_OSDEV_COPROC: return hwloc_snprintf(string, size, verbose ? "Co-Processor" : "CoProc");
    default:
      # 如果字符串大小大于0，则将字符串置为空字符，返回0
      if (size > 0)
        *string = '\0';
      return 0;
    }
    break;
  default:
    # 如果字符串大小大于0，则将字符串置为空字符，返回0
    if (size > 0)
      *string = '\0';
    return 0;
  }
}

int
hwloc_obj_attr_snprintf(char * __hwloc_restrict string, size_t size, hwloc_obj_t obj, const char * separator, int verbose)
{
  const char *prefix = "";  // 定义一个空字符串作为前缀
  char *tmp = string;  // 定义一个指向输出字符串的指针
  ssize_t tmplen = size;  // 定义输出字符串的长度
  int ret = 0;  // 初始化返回值为0
  int res;  // 定义一个用于存储输出结果的变量

  /* make sure we output at least an empty string */
  if (size)  // 如果输出字符串长度不为0
    *string = '\0';  // 将输出字符串的第一个字符设为'\0'，即空字符串

  /* print memory attributes */
  res = 0;  // 初始化输出结果为0
  if (verbose) {  // 如果 verbose 为真
    if (obj->type == HWLOC_OBJ_NUMANODE && obj->attr->numanode.local_memory)  // 如果对象类型为 NUMANODE 并且有本地内存
      res = hwloc_snprintf(tmp, tmplen, "%slocal=%lu%s%stotal=%lu%s",
               prefix,
               (unsigned long) hwloc_memory_size_printf_value(obj->attr->numanode.local_memory, verbose),
               hwloc_memory_size_printf_unit(obj->attr->numanode.local_memory, verbose),
               separator,
               (unsigned long) hwloc_memory_size_printf_value(obj->total_memory, verbose),
               hwloc_memory_size_printf_unit(obj->total_memory, verbose));  // 格式化输出本地内存和总内存
    else if (obj->total_memory)  // 如果有总内存
      res = hwloc_snprintf(tmp, tmplen, "%stotal=%lu%s",
               prefix,
               (unsigned long) hwloc_memory_size_printf_value(obj->total_memory, verbose),
               hwloc_memory_size_printf_unit(obj->total_memory, verbose));  // 格式化输出总内存
  } else {  // 如果 verbose 为假
    if (obj->type == HWLOC_OBJ_NUMANODE && obj->attr->numanode.local_memory)  // 如果对象类型为 NUMANODE 并且有本地内存
      res = hwloc_snprintf(tmp, tmplen, "%s%lu%s",
               prefix,
               (unsigned long) hwloc_memory_size_printf_value(obj->attr->numanode.local_memory, verbose),
               hwloc_memory_size_printf_unit(obj->attr->numanode.local_memory, verbose));  // 格式化输出本地内存
  }
  if (res < 0)  // 如果输出结果小于0
    return -1;  // 返回-1
  ret += res;  // 累加输出结果到返回值
  if (ret > 0)  // 如果返回值大于0
    prefix = separator;  // 将前缀设为分隔符
  if (res >= tmplen)  // 如果输出结果大于等于输出字符串长度
    # 计算 res 的值，如果 tmplen 大于 0，则 res 为 tmplen - 1，否则为 0
    res = tmplen>0 ? (int)tmplen - 1 : 0;
    # 将 tmp 增加 res
    tmp += res;
    # tmplen 减去 res
    tmplen -= res;

    # 打印特定类型的属性
    res = 0;
    switch (obj->type) {
    case HWLOC_OBJ_L1CACHE:
    case HWLOC_OBJ_L2CACHE:
    case HWLOC_OBJ_L3CACHE:
    case HWLOC_OBJ_L4CACHE:
    case HWLOC_OBJ_L5CACHE:
    case HWLOC_OBJ_L1ICACHE:
    case HWLOC_OBJ_L2ICACHE:
    case HWLOC_OBJ_L3ICACHE:
    case HWLOC_OBJ_MEMCACHE:
        # 如果 verbose 为真
        if (verbose) {
            # 定义一个长度为 32 的字符数组 assoc
            char assoc[32];
            # 如果 obj->attr->cache.associativity 为 -1，则将 "fully-associative" 存入 assoc
            if (obj->attr->cache.associativity == -1)
                snprintf(assoc, sizeof(assoc), "%sfully-associative", separator);
            # 如果 obj->attr->cache.associativity 为 0，则将 '\0' 存入 assoc
            else if (obj->attr->cache.associativity == 0)
                *assoc = '\0';
            # 否则将 "ways=xxx" 存入 assoc，其中 xxx 为 obj->attr->cache.associativity
            else
                snprintf(assoc, sizeof(assoc), "%sways=%d", separator, obj->attr->cache.associativity);
            # 将特定格式的字符串存入 tmp 中，并将结果存入 res
            res = hwloc_snprintf(tmp, tmplen, "%ssize=%lu%s%slinesize=%u%s",
                        prefix,
                        (unsigned long) hwloc_memory_size_printf_value(obj->attr->cache.size, verbose),
                        hwloc_memory_size_printf_unit(obj->attr->cache.size, verbose),
                        separator, obj->attr->cache.linesize,
                        assoc);
        } 
        # 如果 verbose 为假
        else {
            # 将特定格式的字符串存入 tmp 中，并将结果存入 res
            res = hwloc_snprintf(tmp, tmplen, "%s%lu%s",
                        prefix,
                        (unsigned long) hwloc_memory_size_printf_value(obj->attr->cache.size, verbose),
                        hwloc_memory_size_printf_unit(obj->attr->cache.size, verbose));
        }
        break;
    case HWLOC_OBJ_BRIDGE:
        # 如果 verbose 为真
        if (verbose) {
            # 定义长度为 128 和 64 的字符数组 up 和 down
            char up[128], down[64];
            # 如果 obj->attr->bridge.upstream_type 为 HWLOC_OBJ_BRIDGE_PCI
            if (obj->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI) {
                # 定义一个长度为 64 的字符数组 linkspeed，并根据 obj->attr->pcidev.linkspeed 存入相应的字符串
                char linkspeed[64]= "";
                if (obj->attr->pcidev.linkspeed)
                    snprintf(linkspeed, sizeof(linkspeed), "%slink=%.2fGB/s", separator, obj->attr->pcidev.linkspeed);
    // 使用 snprintf 格式化输出字符串，包括 busid、id、class 等信息
    snprintf(up, sizeof(up), "busid=%04x:%02x:%02x.%01x%sid=%04x:%04x%sclass=%04x(%s)%s",
         obj->attr->pcidev.domain, obj->attr->pcidev.bus, obj->attr->pcidev.dev, obj->attr->pcidev.func, separator,
         obj->attr->pcidev.vendor_id, obj->attr->pcidev.device_id, separator,
         obj->attr->pcidev.class_id, hwloc_pci_class_string(obj->attr->pcidev.class_id), linkspeed);
      } else
        *up = '\0';
      // 如果 downstream 是 PCI 类型的桥接设备，格式化输出 buses 信息
      if (obj->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI) {
        snprintf(down, sizeof(down), "buses=%04x:[%02x-%02x]",
                 obj->attr->bridge.downstream.pci.domain, obj->attr->bridge.downstream.pci.secondary_bus, obj->attr->bridge.downstream.pci.subordinate_bus);
      } else
        assert(0); // 如果不是 PCI 类型的桥接设备，断言失败
      // 根据 up 是否为空，选择合适的格式化输出结果
      if (*up)
    res = hwloc_snprintf(string, size, "%s%s%s", up, separator, down);
      else
    res = hwloc_snprintf(string, size, "%s", down);
    }
    break;
  case HWLOC_OBJ_PCI_DEVICE:
    // 如果是 PCI 设备，根据 verbose 是否为真，格式化输出相应信息
    if (verbose) {
      char linkspeed[64]= "";
      // 如果 linkspeed 存在，格式化输出 link 信息
      if (obj->attr->pcidev.linkspeed)
        snprintf(linkspeed, sizeof(linkspeed), "%slink=%.2fGB/s", separator, obj->attr->pcidev.linkspeed);
      // 格式化输出 busid、id、class 等信息
      res = hwloc_snprintf(string, size, "busid=%04x:%02x:%02x.%01x%sid=%04x:%04x%sclass=%04x(%s)%s",
               obj->attr->pcidev.domain, obj->attr->pcidev.bus, obj->attr->pcidev.dev, obj->attr->pcidev.func, separator,
               obj->attr->pcidev.vendor_id, obj->attr->pcidev.device_id, separator,
               obj->attr->pcidev.class_id, hwloc_pci_class_string(obj->attr->pcidev.class_id), linkspeed);
    }
    break;
  default:
    break;
  }
  // 如果 res 小于 0，返回 -1
  if (res < 0)
    return -1;
  // ret 累加 res
  ret += res;
  // 如果 ret 大于 0，prefix 赋值为 separator
  if (ret > 0)
    prefix = separator;
  // 如果 res 大于等于 tmplen，res 赋值为 tmplen 减 1
  if (res >= tmplen)
    res = tmplen>0 ? (int)tmplen - 1 : 0;
  tmp += res;
  tmplen -= res;

  // 如果 verbose 为真，打印信息
  if (verbose) {
    unsigned i;
    # 遍历对象的信息列表
    for(i=0; i<obj->infos_count; i++) {
      # 获取当前信息的指针
      struct hwloc_info_s *info = &obj->infos[i];
      # 定义引号变量
      const char *quote;
      # 如果信息值中包含空格，则使用双引号
      if (strchr(info->value, ' '))
        quote = "\"";
      # 否则不使用引号
      else
        quote = "";
      # 将信息名和值格式化成字符串，添加到临时缓冲区
      res = hwloc_snprintf(tmp, tmplen, "%s%s=%s%s%s",
                 prefix,
                 info->name,
                 quote, info->value, quote);
      # 如果格式化失败，返回错误
      if (res < 0)
        return -1;
      # 更新返回值
      ret += res;
      # 如果格式化长度超过缓冲区长度，截断字符串
      if (res >= tmplen)
        res = tmplen>0 ? (int)tmplen - 1 : 0;
      # 更新临时缓冲区指针和长度
      tmp += res;
      tmplen -= res;
      # 如果返回值大于0，更新前缀
      if (ret > 0)
        prefix = separator;
    }
  }
  # 返回格式化后的字符串长度
  return ret;
}

int hwloc_bitmap_singlify_per_core(hwloc_topology_t topology, hwloc_bitmap_t cpuset, unsigned which)
{
  // 从拓扑结构中获取下一个覆盖给定 cpuset 的核心对象
  hwloc_obj_t core = NULL;
  while ((core = hwloc_get_next_obj_covering_cpuset_by_type(topology, cpuset, HWLOC_OBJ_CORE, core)) != NULL) {
    /* 此核心在 cpuset 中有一些处理器单元，找到第 which 个 */
    unsigned i = 0;
    int pu = -1;
    do {
      pu = hwloc_bitmap_next(core->cpuset, pu);
      if (pu == -1) {
    /* 在 cpuset 和核心中没有第 which 个处理器单元，移除整个核心 */
    hwloc_bitmap_andnot(cpuset, cpuset, core->cpuset);
    break;
      }
      if (hwloc_bitmap_isset(cpuset, pu)) {
    if (i == which) {
      /* 移除整个核心，除了确切的处理器单元 */
      hwloc_bitmap_andnot(cpuset, cpuset, core->cpuset);
      hwloc_bitmap_set(cpuset, pu);
      break;
    }
    i++;
      }
    } while (1);
  }
  return 0;
}

hwloc_obj_t
hwloc_get_obj_with_same_locality(hwloc_topology_t topology, hwloc_obj_t src,
                                 hwloc_obj_type_t type, const char *subtype, const char *nameprefix,
                                 unsigned long flags)
{
  if (flags) {
    errno = EINVAL;
    return NULL;
  }

  if (hwloc_obj_type_is_normal(src->type) || hwloc_obj_type_is_memory(src->type)) {
    /* 普通/内存类型，查找具有相同集合的普通/内存类型 */
    hwloc_obj_t obj;

    if (!hwloc_obj_type_is_normal(type) && !hwloc_obj_type_is_memory(type)) {
      errno = EINVAL;
      return NULL;
    }

    obj = NULL;
    while ((obj = hwloc_get_next_obj_by_type(topology, type, obj)) != NULL) {
      if (!hwloc_bitmap_isequal(src->cpuset, obj->cpuset)
          || !hwloc_bitmap_isequal(src->nodeset, obj->nodeset))
        continue;
      if (subtype && (!obj->subtype || strcasecmp(subtype, obj->subtype)))
        continue;
      if (nameprefix && (!obj->name || hwloc_strncasecmp(nameprefix, obj->name, strlen(nameprefix))))
        continue;
      return obj;
    }
    errno = ENOENT;
  }
  return NULL;
}
    // 如果输入为空，则返回空指针
    return NULL;

  } else if (hwloc_obj_type_is_io(src->type)) {
    // 如果是 I/O 设备，则在同一 PCI 中查找 PCI/OS
    hwloc_obj_t pci;

    // 如果源和目标类型不是 OS 设备或 PCI 设备，则返回无效参数错误
    if ((src->type != HWLOC_OBJ_OS_DEVICE && src->type != HWLOC_OBJ_PCI_DEVICE)
        || (type != HWLOC_OBJ_OS_DEVICE && type != HWLOC_OBJ_PCI_DEVICE)) {
      errno = EINVAL;
      return NULL;
    }

    // 向上遍历以找到容器
    pci = src;
    while (pci->type == HWLOC_OBJ_OS_DEVICE)
      pci = pci->parent;

    if (type == HWLOC_OBJ_PCI_DEVICE) {
      // 如果目标类型是 PCI 设备
      if (pci->type != HWLOC_OBJ_PCI_DEVICE) {
        errno = ENOENT;
        return NULL;
      }
      if (subtype && (!pci->subtype || strcasecmp(subtype, pci->subtype))) {
        errno = ENOENT;
        return NULL;
      }
      if (nameprefix && (!pci->name || hwloc_strncasecmp(nameprefix, pci->name, strlen(nameprefix)))) {
        errno = ENOENT;
        return NULL;
      }
      return pci;

    } else {
      // 寻找匹配的 osdev 子节点
      assert(type == HWLOC_OBJ_OS_DEVICE);
      // 如果将 osdev 存储在 osdev 中，此处将无法工作
      hwloc_obj_t child;
      for(child = pci->io_first_child; child; child = child->next_sibling) {
        if (child->type != HWLOC_OBJ_OS_DEVICE)
          // 目前不应该发生此情况
          continue;
        if (subtype && (!child->subtype || strcasecmp(subtype, child->subtype)))
          continue;
        if (nameprefix && (!child->name || hwloc_strncasecmp(nameprefix, child->name, strlen(nameprefix))))
          continue;
        return child;
      }
    }
    errno = ENOENT;
    return NULL;

  } else {
    // 对于 Misc 类型，不做任何处理
    errno = EINVAL;
    return NULL;
  }
# 闭合前面的函数定义
```