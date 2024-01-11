# `xmrig\src\3rdparty\hwloc\src\memattrs.c`

```
/*
 * 版权所有 © 2020-2022 Inria。
 * 保留所有权利。请参阅顶层目录中的COPYING文件。
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/debug.h"


/*****************************
 * 属性
 */

static __hwloc_inline
hwloc_uint64_t hwloc__memattr_get_convenience_value(hwloc_memattr_id_t id,
                                                    hwloc_obj_t node)
{
  // 如果属性ID为HWLOC_MEMATTR_ID_CAPACITY，则返回节点的本地内存
  if (id == HWLOC_MEMATTR_ID_CAPACITY)
    return node->attr->numanode.local_memory;
  // 如果属性ID为HWLOC_MEMATTR_ID_LOCALITY，则返回节点cpuset的位数
  else if (id == HWLOC_MEMATTR_ID_LOCALITY)
    return hwloc_bitmap_weight(node->cpuset);
  // 否则断言失败
  else
    assert(0);
  return 0; /* 关闭编译器警告 */
}

void
hwloc_internal_memattrs_init(struct hwloc_topology *topology)
{
  // 初始化内存属性数量为0
  topology->nr_memattrs = 0;
  // 内存属性指针为空
  topology->memattrs = NULL;
}

static void
hwloc__setup_memattr(struct hwloc_internal_memattr_s *imattr,
                     char *name,
                     unsigned long flags,
                     unsigned long iflags)
{
  // 设置内存属性的名称、标志和内部标志
  imattr->name = name;
  imattr->flags = flags;
  imattr->iflags = iflags;

  // 初始化目标数量为0，目标指针为空
  imattr->nr_targets = 0;
  imattr->targets = NULL;
}

void
hwloc_internal_memattrs_prepare(struct hwloc_topology *topology)
{
  // 为内存属性分配内存
  topology->memattrs = malloc(HWLOC_MEMATTR_ID_MAX * sizeof(*topology->memattrs));
  // 如果分配失败，则...
  if (!topology->memattrs)
}

static void
hwloc__imi_destroy(struct hwloc_internal_memattr_initiator_s *imi)
{
  // 如果initiator的类型为HWLOC_LOCATION_TYPE_CPUSET，则释放cpuset
  if (imi->initiator.type == HWLOC_LOCATION_TYPE_CPUSET)
    hwloc_bitmap_free(imi->initiator.location.cpuset);
}

static void
hwloc__imtg_destroy(struct hwloc_internal_memattr_s *imattr,
                    struct hwloc_internal_memattr_target_s *imtg)
{
  // 如果属性标志包含HWLOC_MEMATTR_FLAG_NEED_INITIATOR
  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    // 遍历initiators数组，释放资源
    unsigned k;
    for(k=0; k<imtg->nr_initiators; k++)
      hwloc__imi_destroy(&imtg->initiators[k]);
  }
  // 释放initiators数组
  free(imtg->initiators);
}

void
hwloc_internal_memattrs_destroy(struct hwloc_topology *topology)
{
  // 定义无符号整数 id
  unsigned id;
  // 遍历内存属性数组
  for(id=0; id<topology->nr_memattrs; id++) {
    // 获取当前内存属性
    struct hwloc_internal_memattr_s *imattr = &topology->memattrs[id];
    // 定义无符号整数 j
    unsigned j;
    // 遍历当前内存属性的目标数组
    for(j=0; j<imattr->nr_targets; j++)
      // 销毁当前内存属性的目标
      hwloc__imtg_destroy(imattr, &imattr->targets[j]);
    // 释放当前内存属性的目标数组
    free(imattr->targets);
    // 如果当前内存属性的标志不包含 HWLOC_IMATTR_FLAG_STATIC_NAME
    if (!(imattr->iflags & HWLOC_IMATTR_FLAG_STATIC_NAME))
      // 释放当前内存属性的名称
      free(imattr->name);
  }
  // 释放内存属性数组
  free(topology->memattrs);

  // 将内存属性数组指针置为空
  topology->memattrs = NULL;
  // 将内存属性数量置为 0
  topology->nr_memattrs = 0;
}

// 复制内存属性
int
hwloc_internal_memattrs_dup(struct hwloc_topology *new, struct hwloc_topology *old)
{
  // 获取新拓扑结构的内存分配器
  struct hwloc_tma *tma = new->tma;
  // 定义内部内存属性数组
  struct hwloc_internal_memattr_s *imattrs;
  // 定义内存属性 id
  hwloc_memattr_id_t id;

  /* old->nr_memattrs is always > 0 thanks to default memattrs */

  // 分配新拓扑结构的内存属性数组
  imattrs = hwloc_tma_malloc(tma, old->nr_memattrs * sizeof(*imattrs));
  // 如果分配失败，返回 -1
  if (!imattrs)
    return -1;
  // 将新拓扑结构的内存属性数组指向分配的内存
  new->memattrs = imattrs;
  // 设置新拓扑结构的内存属性数量
  new->nr_memattrs = old->nr_memattrs;
  // 复制旧拓扑结构的内存属性到新拓扑结构
  memcpy(imattrs, old->memattrs, old->nr_memattrs * sizeof(*imattrs));

  // 遍历旧拓扑结构的内存属性
  for(id=0; id<old->nr_memattrs; id++) {
    // 获取旧拓扑结构的当前内存属性
    struct hwloc_internal_memattr_s *oimattr = &old->memattrs[id];
    // 获取新拓扑结构的当前内存属性
    struct hwloc_internal_memattr_s *nimattr = &imattrs[id];
    // 定义无符号整数 j
    unsigned j;

    // 断言当前内存属性的名称存在
    assert(oimattr->name);
    // 复制当前内存属性的名称到新拓扑结构的内存分配器
    nimattr->name = hwloc_tma_strdup(tma, oimattr->name);
    // 如果名称复制失败
    if (!nimattr->name) {
      // 断言内存分配器存在且不会失败
      assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
      // 设置新拓扑结构的内存属性数量
      new->nr_memattrs = id;
      // 跳转到失败标签
      goto failed;
    }
    // 清除当前内存属性的标志中的 HWLOC_IMATTR_FLAG_STATIC_NAME
    nimattr->iflags &= ~HWLOC_IMATTR_FLAG_STATIC_NAME;
    // 清除当前内存属性的标志中的 HWLOC_IMATTR_FLAG_CACHE_VALID
    nimattr->iflags &= ~HWLOC_IMATTR_FLAG_CACHE_VALID; /* cache will need refresh */

    // 如果当前内存属性没有目标
    if (!oimattr->nr_targets)
      // 继续下一次循环
      continue;

    // 分配新拓扑结构的当前内存属性的目标数组
    nimattr->targets = hwloc_tma_malloc(tma, oimattr->nr_targets * sizeof(*nimattr->targets));
    // 如果分配失败
    if (!nimattr->targets) {
      // 释放当前内存属性的名称
      free(nimattr->name);
      // 设置新拓扑结构的内存属性数量
      new->nr_memattrs = id;
      // 跳转到失败标签
      goto failed;
    }
    // 复制当前内存属性的目标到新拓扑结构的当前内存属性的目标数组
    memcpy(nimattr->targets, oimattr->targets, oimattr->nr_targets * sizeof(*nimattr->targets));
    # 遍历目标的数量
    for(j=0; j<oimattr->nr_targets; j++) {
      # 获取旧目标和新目标的指针
      struct hwloc_internal_memattr_target_s *oimtg = &oimattr->targets[j];
      struct hwloc_internal_memattr_target_s *nimtg = &nimattr->targets[j];
      unsigned k;

      # 将新目标的对象设置为NULL，缓存需要刷新
      nimtg->obj = NULL; /* cache will need refresh */

      # 如果旧目标的发起者数量为0，则继续下一次循环
      if (!oimtg->nr_initiators)
        continue;

      # 为新目标的发起者分配内存
      nimtg->initiators = hwloc_tma_malloc(tma, oimtg->nr_initiators * sizeof(*nimtg->initiators));
      # 如果内存分配失败，则更新目标数量和新的内存属性数量，然后跳转到失败标签
      if (!nimtg->initiators) {
        nimattr->nr_targets = j;
        new->nr_memattrs = id+1;
        goto failed;
      }
      # 复制旧目标的发起者到新目标的发起者
      memcpy(nimtg->initiators, oimtg->initiators, oimtg->nr_initiators * sizeof(*nimtg->initiators));

      # 遍历发起者的数量
      for(k=0; k<oimtg->nr_initiators; k++) {
        # 获取旧发起者和新发起者的指针
        struct hwloc_internal_memattr_initiator_s *oimi = &oimtg->initiators[k];
        struct hwloc_internal_memattr_initiator_s *nimi = &nimtg->initiators[k];
        # 如果发起者的类型是HWLOC_LOCATION_TYPE_CPUSET，则复制发起者的位置位图
        if (oimi->initiator.type == HWLOC_LOCATION_TYPE_CPUSET) {
          nimi->initiator.location.cpuset = hwloc_bitmap_tma_dup(tma, oimi->initiator.location.cpuset);
          # 如果位图复制失败，则更新发起者数量和目标数量，然后跳转到失败标签
          if (!nimi->initiator.location.cpuset) {
            nimtg->nr_initiators = k;
            nimattr->nr_targets = j+1;
            new->nr_memattrs = id+1;
            goto failed;
          }
        } 
        # 如果发起者的类型是HWLOC_LOCATION_TYPE_OBJECT，则将发起者的对象设置为NULL，缓存需要刷新
        else if (oimi->initiator.type == HWLOC_LOCATION_TYPE_OBJECT) {
          nimi->initiator.location.object.obj = NULL; /* cache will need refresh */
        }
      }
    }
  }
  # 成功返回0
  return 0;

 failed:
  # 销毁新的内存属性
  hwloc_internal_memattrs_destroy(new);
  # 返回失败标志-1
  return -1;
}

int
hwloc_memattr_get_by_name(hwloc_topology_t topology,
                          const char *name,
                          hwloc_memattr_id_t *idp)
{
  unsigned id;
  // 遍历内存属性列表，查找指定名称的内存属性
  for(id=0; id<topology->nr_memattrs; id++) {
    if (!strcmp(topology->memattrs[id].name, name)) {
      *idp = id;
      return 0;
    }
  }
  // 如果未找到指定名称的内存属性，设置错误码并返回-1
  errno = EINVAL;
  return -1;
}

int
hwloc_memattr_get_name(hwloc_topology_t topology,
                       hwloc_memattr_id_t id,
                       const char **namep)
{
  // 检查内存属性 ID 是否有效
  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  // 获取指定 ID 的内存属性名称
  *namep = topology->memattrs[id].name;
  return 0;
}

int
hwloc_memattr_get_flags(hwloc_topology_t topology,
                        hwloc_memattr_id_t id,
                        unsigned long *flagsp)
{
  // 检查内存属性 ID 是否有效
  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  // 获取指定 ID 的内存属性标志
  *flagsp = topology->memattrs[id].flags;
  return 0;
}

int
hwloc_memattr_register(hwloc_topology_t topology,
                       const char *_name,
                       unsigned long flags,
                       hwloc_memattr_id_t *id)
{
  struct hwloc_internal_memattr_s *newattrs;
  char *name;
  unsigned i;

  /* check flags */
  // 检查标志位是否有效
  if (flags & ~(HWLOC_MEMATTR_FLAG_NEED_INITIATOR|HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_HIGHER_FIRST)) {
    errno = EINVAL;
    return -1;
  }
  // 检查标志位是否设置了合法的排序方式
  if (!(flags & (HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_HIGHER_FIRST))) {
    errno = EINVAL;
    return -1;
  }
  // 检查标志位是否同时设置了两种排序方式
  if ((flags & (HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_HIGHER_FIRST))
      == (HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_HIGHER_FIRST)) {
    errno = EINVAL;
    return -1;
  }

  // 检查名称是否为空
  if (!_name) {
    errno = EINVAL;
    return -1;
  }

  // 检查名称是否已经被使用
  for(i=0; i<topology->nr_memattrs; i++) {
    if (!strcmp(_name, topology->memattrs[i].name)) {
      errno = EBUSY;
      return -1;
    }
  }

  // 复制名称字符串
  name = strdup(_name);
  if (!name)
    # 返回错误代码 -1
    return -1;

  # 重新分配内存以扩展topology->memattrs数组的大小
  newattrs = realloc(topology->memattrs, (topology->nr_memattrs + 1) * sizeof(*topology->memattrs));
  # 如果内存分配失败，则释放name指针并返回错误代码 -1
  if (!newattrs) {
    free(name);
    return -1;
  }

  # 初始化新的内存属性
  hwloc__setup_memattr(&newattrs[topology->nr_memattrs],
                       name, flags, 0);

  # 设置内存属性标记为有效
  # 内存属性在创建时是有效的
  newattrs[topology->nr_memattrs].iflags |= HWLOC_IMATTR_FLAG_CACHE_VALID;

  # 将新的内存属性的索引赋值给id
  *id = topology->nr_memattrs;
  # 增加内存属性的数量
  topology->nr_memattrs++;
  # 更新topology->memattrs指针
  topology->memattrs = newattrs;
  # 返回成功代码 0
  return 0;
/***************************
 * Internal Locations
 */

/* 如果 cpuset/obj 匹配现有的发起者位置，则返回1，
 * 例如，如果查询的 cpuset 包含在现有的 cpuset 中
 */
static int
match_internal_location(struct hwloc_internal_location_s *iloc,
                        struct hwloc_internal_memattr_initiator_s *imi)
{
  // 如果位置类型不匹配，则返回0
  if (iloc->type != imi->initiator.type)
    return 0;
  // 根据位置类型进行匹配
  switch (iloc->type) {
  case HWLOC_LOCATION_TYPE_CPUSET:
    // 检查 cpuset 是否包含在现有的 cpuset 中
    return hwloc_bitmap_isincluded(iloc->location.cpuset, imi->initiator.location.cpuset);
  case HWLOC_LOCATION_TYPE_OBJECT:
    // 检查对象类型和 gp_index 是否匹配
    return iloc->location.object.type == imi->initiator.location.object.type
      && iloc->location.object.gp_index == imi->initiator.location.object.gp_index;
  default:
    return 0;
  }
}

static int
to_internal_location(struct hwloc_internal_location_s *iloc,
                     struct hwloc_location *location)
{
  // 设置位置类型
  iloc->type = location->type;

  switch (location->type) {
  case HWLOC_LOCATION_TYPE_CPUSET:
    // 如果 cpuset 为空或者为零，则返回错误
    if (!location->location.cpuset || hwloc_bitmap_iszero(location->location.cpuset)) {
      errno = EINVAL;
      return -1;
    }
    // 设置 cpuset
    iloc->location.cpuset = location->location.cpuset;
    return 0;
  case HWLOC_LOCATION_TYPE_OBJECT:
    // 如果对象为空，则返回错误
    if (!location->location.object) {
      errno = EINVAL;
      return -1;
    }
    // 设置对象的 gp_index 和类型
    iloc->location.object.gp_index = location->location.object->gp_index;
    iloc->location.object.type = location->location.object->type;
    return 0;
  default:
    errno = EINVAL;
    return -1;
  }
}

static int
from_internal_location(struct hwloc_internal_location_s *iloc,
                       struct hwloc_location *location)
{
  // 设置位置类型
  location->type = iloc->type;

  switch (iloc->type) {
  case HWLOC_LOCATION_TYPE_CPUSET:
    // 设置 cpuset
    location->location.cpuset = iloc->location.cpuset;
    return 0;
  case HWLOC_LOCATION_TYPE_OBJECT:
    /* 需要刷新缓存 */
    location->location.object = iloc->location.object.obj;
    # 如果位置对象为空，则返回-1
    if (!location->location.object)
      return -1;
    # 返回0表示成功
    return 0;
  # 默认情况下
  default:
    # 设置错误码为无效参数
    errno = EINVAL;
    # 返回-1表示失败
    return -1;
  }
}
/************************
 * Refreshing
 */

static int
hwloc__imi_refresh(struct hwloc_topology *topology,
                   struct hwloc_internal_memattr_initiator_s *imi)
{
  switch (imi->initiator.type) {
  case HWLOC_LOCATION_TYPE_CPUSET: {
    // 如果初始化器类型是 CPUSET，则将其与拥有的第一个核心集合进行按位与操作
    hwloc_bitmap_and(imi->initiator.location.cpuset, imi->initiator.location.cpuset, topology->levels[0][0]->cpuset);
    // 如果结果集合为空，则销毁初始化器并返回错误
    if (hwloc_bitmap_iszero(imi->initiator.location.cpuset)) {
      hwloc__imi_destroy(imi);
      return -1;
    }
    return 0;
  }
  case HWLOC_LOCATION_TYPE_OBJECT: {
    // 如果初始化器类型是 OBJECT，则根据类型和全局索引获取对象
    hwloc_obj_t obj = hwloc_get_obj_by_type_and_gp_index(topology,
                                                         imi->initiator.location.object.type,
                                                         imi->initiator.location.object.gp_index);
    // 如果对象不存在，则销毁初始化器并返回错误
    if (!obj) {
      hwloc__imi_destroy(imi);
      return -1;
    }
    // 将对象存储在初始化器的位置中，并返回成功
    imi->initiator.location.object.obj = obj;
    return 0;
  }
  default:
    // 如果初始化器类型不是 CPUSET 或 OBJECT，则断言失败
    assert(0);
  }
  // 默认返回错误
  return -1;
}

static int
hwloc__imtg_refresh(struct hwloc_topology *topology,
                    struct hwloc_internal_memattr_s *imattr,
                    struct hwloc_internal_memattr_target_s *imtg)
{
  hwloc_obj_t node;

  /* no need to refresh convenience memattrs */
  // 不需要刷新便利内存属性

  /* check the target object */
  // 检查目标对象
  if (imtg->gp_index == (hwloc_uint64_t) -1) {
    // 如果全局索引为-1，则根据操作系统索引获取 NUMA 或 PU 对象
    if (imtg->type == HWLOC_OBJ_NUMANODE)
      node = hwloc_get_numanode_obj_by_os_index(topology, imtg->os_index);
    else if (imtg->type == HWLOC_OBJ_PU)
      node = hwloc_get_pu_obj_by_os_index(topology, imtg->os_index);
    else
      node = NULL;
  } else {
    // 否则根据类型和全局索引获取对象
    node = hwloc_get_obj_by_type_and_gp_index(topology, imtg->type, imtg->gp_index);
  }
  // 如果对象不存在，则销毁目标并返回错误
  if (!node) {
    hwloc__imtg_destroy(imattr, imtg);
    # 返回错误代码 -1
    return -1;
  }

  # 如果 gp_index 尚未初始化，则保存 gp_index
  imtg->gp_index = node->gp_index;
  # 缓存对象
  imtg->obj = node;

  # 如果需要初始化发起者
  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    # 检查发起者
    unsigned k, l;
    for(k=0, l=0; k<imtg->nr_initiators; k++) {
      # 刷新发起者
      int err = hwloc__imi_refresh(topology, &imtg->initiators[k]);
      if (err < 0)
        continue;
      if (k != l)
        # 复制发起者信息
        memcpy(&imtg->initiators[l], &imtg->initiators[k], sizeof(*imtg->initiators));
      l++;
    }
    # 更新发起者数量
    imtg->nr_initiators = l;
    # 如果没有发起者，则销毁 imtg 并返回错误代码 -1
    if (!imtg->nr_initiators) {
      hwloc__imtg_destroy(imattr, imtg);
      return -1;
    }
  }
  # 返回成功代码 0
  return 0;
}

static void
hwloc__imattr_refresh(struct hwloc_topology *topology,
                      struct hwloc_internal_memattr_s *imattr)
{
  unsigned j, k;
  // 遍历内存属性对象的目标列表
  for(j=0, k=0; j<imattr->nr_targets; j++) {
    // 刷新内存属性对象的目标
    int ret = hwloc__imtg_refresh(topology, imattr, &imattr->targets[j]);
    // 如果目标仍然有效，将其移动到新的位置
    if (!ret) {
      if (j != k)
        memcpy(&imattr->targets[k], &imattr->targets[j], sizeof(*imattr->targets));
      k++;
    }
  }
  // 更新内存属性对象的目标数量
  imattr->nr_targets = k;
  // 设置内存属性对象的缓存有效标志
  imattr->iflags |= HWLOC_IMATTR_FLAG_CACHE_VALID;
}

void
hwloc_internal_memattrs_refresh(struct hwloc_topology *topology)
{
  unsigned id;
  // 遍历拓扑结构中的内存属性对象
  for(id=0; id<topology->nr_memattrs; id++) {
    struct hwloc_internal_memattr_s *imattr = &topology->memattrs[id];
    // 如果内存属性对象的缓存有效标志已经设置，则无需刷新
    if (imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID)
      continue;
    // 刷新内存属性对象
    hwloc__imattr_refresh(topology, imattr);
  }
}

void
hwloc_internal_memattrs_need_refresh(struct hwloc_topology *topology)
{
  unsigned id;
  // 遍历拓扑结构中的内存属性对象
  for(id=0; id<topology->nr_memattrs; id++) {
    struct hwloc_internal_memattr_s *imattr = &topology->memattrs[id];
    // 如果内存属性对象的方便标志已经设置，则无需刷新
    if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE)
      continue;
    // 清除内存属性对象的缓存有效标志
    imattr->iflags &= ~HWLOC_IMATTR_FLAG_CACHE_VALID;
  }
}


/********************************
 * Targets
 */

static struct hwloc_internal_memattr_target_s *
hwloc__memattr_get_target(struct hwloc_internal_memattr_s *imattr,
                          hwloc_obj_type_t target_type,
                          hwloc_uint64_t target_gp_index,
                          unsigned target_os_index,
                          int create)
{
  struct hwloc_internal_memattr_target_s *news, *new;
  unsigned j;

  // 遍历内存属性对象的目标列表
  for(j=0; j<imattr->nr_targets; j++) {
  # 如果目标类型与当前遍历到的目标类型相同
  if (target_type == imattr->targets[j].type)
    # 如果目标全局索引不是-1且等于当前遍历到的目标的全局索引，或者目标操作系统索引不是-1且等于当前遍历到的目标的操作系统索引
    if ((target_gp_index != (hwloc_uint64_t)-1 && target_gp_index == imattr->targets[j].gp_index)
        || (target_os_index != (unsigned)-1 && target_os_index == imattr->targets[j].os_index))
      # 返回当前遍历到的目标
      return &imattr->targets[j];
  # 如果不需要创建新的目标，则返回空
  if (!create)
    return NULL;

  # 重新分配内存以扩展目标数组
  news = realloc(imattr->targets, (imattr->nr_targets+1)*sizeof(*imattr->targets));
  # 如果分配内存失败，则返回空
  if (!news)
    return NULL;
  imattr->targets = news;

  # 可能需要对目标进行排序？在加载结束时按逻辑索引排序？
  /* FIXME sort targets? by logical index at the end of load? */

  # 获取新的目标指针，并设置其类型、全局索引和操作系统索引
  new = &news[imattr->nr_targets];
  new->type = target_type;
  new->gp_index = target_gp_index;
  new->os_index = target_os_index;

  # 缓存对象将在实际访问时刷新
  new->obj = NULL;
  imattr->iflags &= ~HWLOC_IMATTR_FLAG_CACHE_VALID;
  /* When setting a value after load(), the caller has the target object
   * (and initiator object, if not CPU set). Hence, we could avoid invalidating
   * the cache here.
   * The overhead of the imattr-wide refresh isn't high enough so far
   * to justify making the cache management more complex.
   */

  # 初始化新目标的发起者数量和发起者数组
  new->nr_initiators = 0;
  new->initiators = NULL;
  new->noinitiator_value = 0;
  # 增加目标数量
  imattr->nr_targets++;
  # 返回新的目标指针
  return new;
}

static struct hwloc_internal_memattr_initiator_s *
hwloc__memattr_get_initiator_from_location(struct hwloc_internal_memattr_s *imattr,
                                           struct hwloc_internal_memattr_target_s *imtg,
                                           struct hwloc_location *location);
// 从给定的内存属性和目标位置获取对应的内存属性发起者

int
hwloc_memattr_get_targets(hwloc_topology_t topology,
                          hwloc_memattr_id_t id,
                          struct hwloc_location *initiator,
                          unsigned long flags,
                          unsigned *nrp, hwloc_obj_t *targets, hwloc_uint64_t *values)
{
  struct hwloc_internal_memattr_s *imattr;
  unsigned i, found = 0, max;

  if (flags) {
    errno = EINVAL;
    return -1;
  }
  // 如果标志不为0，设置错误码并返回-1
  if (!nrp || (*nrp && !targets)) {
    errno = EINVAL;
    return -1;
  }
  // 如果nrp为空或者(nr不为0且targets为空)，设置错误码并返回-1
  max = *nrp;

  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  // 如果id大于等于拓扑结构中的内存属性数量，设置错误码并返回-1
  imattr = &topology->memattrs[id];

  if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {
    /* convenience attributes */
    // 如果是便利属性
    for(i=0; ; i++) {
      hwloc_obj_t node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, i);
      // 获取指定类型的对象
      if (!node)
        break;
      if (found<max) {
        targets[found] = node;
        // 将对象添加到目标数组中
        if (values)
          values[found] = hwloc__memattr_get_convenience_value(id, node);
        // 如果值数组不为空，获取便利值并添加到值数组中
      }
      found++;
    }
    goto done;
  }

  /* normal attributes */

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    hwloc__imattr_refresh(topology, imattr);
  // 如果缓存无效，刷新内存属性

  for(i=0; i<imattr->nr_targets; i++) {
    struct hwloc_internal_memattr_target_s *imtg = &imattr->targets[i];
    hwloc_uint64_t value = 0;

    if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
      if (initiator) {
        /* find a matching initiator */
        // 寻找匹配的内存属性发起者
        struct hwloc_internal_memattr_initiator_s *imi = hwloc__memattr_get_initiator_from_location(imattr, imtg, initiator);
        // 从位置获取对应的内存属性发起者
        if (!imi)
          continue;
        value = imi->value;
      }
      // 如果没有找到匹配的内存属性发起者，继续下一次循环
    } else {
      // 如果条件不成立，将值设置为imtg->noinitiator_value
      value = imtg->noinitiator_value;
    }

    // 如果找到的目标数量小于最大值
    if (found<max) {
      // 将imtg->obj赋值给targets数组中的对应位置
      targets[found] = imtg->obj;
      // 如果values存在，将value赋值给values数组中的对应位置
      if (values)
        values[found] = value;
    }
    // 增加找到的目标数量
    found++;
  }

 done:
  // 将found的值赋给nrp指向的变量
  *nrp = found;
  // 返回0表示执行成功
  return 0;
/************************
 * Initiators
 */

// 从内存属性目标中获取初始化器
static struct hwloc_internal_memattr_initiator_s *
hwloc__memattr_target_get_initiator(struct hwloc_internal_memattr_target_s *imtg,
                                    struct hwloc_internal_location_s *iloc,
                                    int create)
{
  // 定义新的初始化器指针
  struct hwloc_internal_memattr_initiator_s *news, *new;
  unsigned k;

  // 遍历内存属性目标的初始化器列表
  for(k=0; k<imtg->nr_initiators; k++) {
    struct hwloc_internal_memattr_initiator_s *imi = &imtg->initiators[k];
    // 如果找到匹配的内部位置，返回该初始化器
    if (match_internal_location(iloc, imi)) {
      return imi;
    }
  }

  // 如果不需要创建新的初始化器，直接返回空指针
  if (!create)
    return NULL;

  // 重新分配内存以扩展初始化器列表
  news = realloc(imtg->initiators, (imtg->nr_initiators+1)*sizeof(*imtg->initiators));
  if (!news)
    return NULL;
  new = &news[imtg->nr_initiators];

  // 将新的位置信息复制到初始化器中
  new->initiator = *iloc;
  // 如果位置类型是 CPUSET，则复制 CPUSET 信息
  if (iloc->type == HWLOC_LOCATION_TYPE_CPUSET) {
    new->initiator.location.cpuset = hwloc_bitmap_dup(iloc->location.cpuset);
    if (!new->initiator.location.cpuset)
      goto out_with_realloc;
  }

  // 增加初始化器数量并更新初始化器列表
  imtg->nr_initiators++;
  imtg->initiators = news;
  return new;

 out_with_realloc:
  // 释放重新分配的内存并返回空指针
  imtg->initiators = news;
  return NULL;
}

// 从位置信息中获取初始化器
static struct hwloc_internal_memattr_initiator_s *
hwloc__memattr_get_initiator_from_location(struct hwloc_internal_memattr_s *imattr,
                                           struct hwloc_internal_memattr_target_s *imtg,
                                           struct hwloc_location *location)
{
  struct hwloc_internal_memattr_initiator_s *imi;
  struct hwloc_internal_location_s iloc;

  // 断言内存属性标志需要初始化器
  assert(imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR);

  // 如果位置信息为空，返回错误
  if (!location) {
    errno = EINVAL;
    return NULL;
  }

  // 将外部位置信息转换为内部位置信息
  if (to_internal_location(&iloc, location) < 0) {
    errno = EINVAL;
    return NULL;
  }

  // 从内存属性目标中获取初始化器
  imi = hwloc__memattr_target_get_initiator(imtg, &iloc, 0);
  if (!imi) {
    errno = EINVAL;
    return NULL;
  }

  return imi;
}

// ...
# 获取内存属性的初始化器
def hwloc_memattr_get_initiators(hwloc_topology_t topology,
                                 hwloc_memattr_id_t id,
                                 hwloc_obj_t target_node,
                                 unsigned long flags,
                                 unsigned *nrp, struct hwloc_location *initiators, hwloc_uint64_t *values)
{
  # 定义内部变量
  struct hwloc_internal_memattr_s *imattr;
  struct hwloc_internal_memattr_target_s *imtg;
  unsigned i, max;

  # 如果标志不为零，设置错误码并返回-1
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  # 如果 nrp 为空或者 nrp 不为空但 initiators 为空，设置错误码并返回-1
  if (!nrp || (*nrp && !initiators)) {
    errno = EINVAL;
    return -1;
  }
  max = *nrp;

  # 如果 id 大于等于内存属性的数量，设置错误码并返回-1
  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  imattr = &topology->memattrs[id];
  # 如果内存属性标志不包含需要初始化器的标志，将 nrp 设置为0并返回0
  if (!(imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR)) {
    *nrp = 0;
    return 0;
  }

  # 断言方便属性没有初始化器
  assert(!(imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE));

  # 如果方便属性标志无效，刷新内存属性
  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    hwloc__imattr_refresh(topology, imattr);

  # 获取内存属性的目标
  imtg = hwloc__memattr_get_target(imattr, target_node->type, target_node->gp_index, target_node->os_index, 0);
  # 如果目标为空，设置错误码并返回-1
  if (!imtg) {
    errno = EINVAL;
    return -1;
  }

  # 遍历初始化器并赋值
  for(i=0; i<imtg->nr_initiators && i<max; i++) {
    struct hwloc_internal_memattr_initiator_s *imi = &imtg->initiators[i];
    int err = from_internal_location(&imi->initiator, &initiators[i]);
    assert(!err);
    if (values)
      # 不需要在这里处理容量/本地性特殊情况，这些是无初始化器的属性
      values[i] = imi->value;
  }

  # 将 nrp 设置为初始化器数量并返回0
  *nrp = imtg->nr_initiators;
  return 0;
}

# 获取内存属性的值
int
hwloc_memattr_get_value(hwloc_topology_t topology,
                        hwloc_memattr_id_t id,
                        hwloc_obj_t target_node,
                        struct hwloc_location *initiator,
                        unsigned long flags,
                        hwloc_uint64_t *valuep)
{
  // 定义指向内存属性结构体的指针
  struct hwloc_internal_memattr_s *imattr;
  // 定义指向内存属性目标结构体的指针
  struct hwloc_internal_memattr_target_s *imtg;

  // 如果标志不为零，则返回无效参数错误
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  // 如果给定的 id 超出了内存属性的数量，则返回无效参数错误
  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  // 将 imattr 指针指向对应 id 的内存属性
  imattr = &topology->memattrs[id];

  // 如果内存属性标志包含 HWLOC_IMATTR_FLAG_CONVENIENCE
  if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {
    /* 便利属性 */
    // 获取便利属性的值，并赋给 valuep
    *valuep = hwloc__memattr_get_convenience_value(id, target_node);
    return 0;
  }

  // 如果不是便利属性
  /* 普通属性 */

  // 如果内存属性标志不包含 HWLOC_IMATTR_FLAG_CACHE_VALID
  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    // 刷新内存属性
    hwloc__imattr_refresh(topology, imattr);

  // 获取内存属性目标
  imtg = hwloc__memattr_get_target(imattr, target_node->type, target_node->gp_index, target_node->os_index, 0);
  // 如果获取失败，则返回无效参数错误
  if (!imtg) {
    errno = EINVAL;
    return -1;
  }

  // 如果内存属性标志包含 HWLOC_MEMATTR_FLAG_NEED_INITIATOR
  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    /* 查找发起者并设置其值 */
    // 从位置获取发起者，并将其值赋给 valuep
    struct hwloc_internal_memattr_initiator_s *imi = hwloc__memattr_get_initiator_from_location(imattr, imtg, initiator);
    if (!imi)
      return -1;
    *valuep = imi->value;
  } else {
    /* 获取无发起者值 */
    // 将无发起者值赋给 valuep
    *valuep = imtg->noinitiator_value;
  }
  return 0;
}

// 设置内部内存属性值
static int
hwloc__internal_memattr_set_value(hwloc_topology_t topology,
                                  hwloc_memattr_id_t id,
                                  hwloc_obj_type_t target_type,
                                  hwloc_uint64_t target_gp_index,
                                  unsigned target_os_index,
                                  struct hwloc_internal_location_s *initiator,
                                  hwloc_uint64_t value)
{
  // 定义指向内存属性结构体的指针
  struct hwloc_internal_memattr_s *imattr;
  // 定义指向内存属性目标结构体的指针
  struct hwloc_internal_memattr_target_s *imtg;

  // 如果给定的 id 超出了内存属性的数量，则返回无效参数错误
  if (id >= topology->nr_memattrs) {
    /* 在初始化过程中发生了错误 */
    errno = EINVAL;
    return -1;
  }
  // 将 imattr 指针指向对应 id 的内存属性
  imattr = &topology->memattrs[id];

  // 如果内存属性标志包含 HWLOC_MEMATTR_FLAG_NEED_INITIATOR
  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    /* 检查给定的发起者 */
    // 如果发起者为空，则返回无效参数错误
    if (!initiator) {
      errno = EINVAL;
      return -1;
  }
}

if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {
  /* 如果属性标记包含方便属性，则属性是只读的 */
  errno = EINVAL;
  return -1;
}

if (topology->is_loaded && !(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
  /* 在加载过程中添加值时不刷新（某些节点可能尚未准备好），稍后会刷新 */
  hwloc__imattr_refresh(topology, imattr);

imtg = hwloc__memattr_get_target(imattr, target_type, target_gp_index, target_os_index, 1);
if (!imtg)
  return -1;

if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
  /* 查找/添加发起者并设置其值 */
  // FIXME 如果 cpuset 大于现有的 cpuset 怎么办？
  struct hwloc_internal_memattr_initiator_s *imi = hwloc__memattr_target_get_initiator(imtg, initiator, 1);
  if (!imi)
    return -1;
  imi->value = value;

} else {
  /* 设置无发起者值 */
  imtg->noinitiator_value = value;
}

return 0;
# 设置内部内存属性的值
int
hwloc_internal_memattr_set_value(hwloc_topology_t topology,  # 设置内存属性的拓扑结构
                                 hwloc_memattr_id_t id,  # 内存属性的ID
                                 hwloc_obj_type_t target_type,  # 目标对象的类型
                                 hwloc_uint64_t target_gp_index,  # 目标对象的全局索引
                                 unsigned target_os_index,  # 目标对象的操作系统索引
                                 struct hwloc_internal_location_s *initiator,  # 内存属性设置的发起者
                                 hwloc_uint64_t value)  # 要设置的值
{
  assert(id != HWLOC_MEMATTR_ID_CAPACITY);  # 断言内存属性ID不等于容量
  assert(id != HWLOC_MEMATTR_ID_LOCALITY);  # 断言内存属性ID不等于局部性

  return hwloc__internal_memattr_set_value(topology, id, target_type, target_gp_index, target_os_index, initiator, value);  # 调用内部函数设置内存属性的值
}

# 设置内存属性的值
int
hwloc_memattr_set_value(hwloc_topology_t topology,  # 设置内存属性的拓扑结构
                        hwloc_memattr_id_t id,  # 内存属性的ID
                        hwloc_obj_t target_node,  # 目标节点对象
                        struct hwloc_location *initiator,  # 内存属性设置的发起者
                        unsigned long flags,  # 标志位
                        hwloc_uint64_t value)  # 要设置的值
{
  struct hwloc_internal_location_s iloc, *ilocp;

  if (flags) {  # 如果标志位不为0
    errno = EINVAL;  # 设置错误码为无效参数
    return -1;  # 返回-1
  }

  if (initiator) {  # 如果有发起者
    if (to_internal_location(&iloc, initiator) < 0) {  # 如果转换为内部位置失败
      errno = EINVAL;  # 设置错误码为无效参数
      return -1;  # 返回-1
    }
    ilocp = &iloc;  # 设置内部位置指针
  } else {
    ilocp = NULL;  # 否则设置内部位置指针为空
  }

  return hwloc__internal_memattr_set_value(topology, id, target_node->type, target_node->gp_index, target_node->os_index, ilocp, value);  # 调用内部函数设置内存属性的值
}

# 更新最佳目标
static void
hwloc__update_best_target(hwloc_obj_t *best_obj, hwloc_uint64_t *best_value, int *found,
                          hwloc_obj_t new_obj, hwloc_uint64_t new_value,
                          int keep_highest)
{
  if (*found) {  # 如果已经找到
    if (keep_highest) {  # 如果保持最高值
      if (new_value <= *best_value)  # 如果新值小于等于最佳值
        return;  # 返回
    } else {
      if (new_value >= *best_value)  # 如果新值大于等于最佳值
        return;  # 返回
    }
  }

  *best_obj = new_obj;  # 更新最佳对象
  *best_value = new_value;  # 更新最佳值
  *found = 1;  # 设置已找到标志为1
}

# 返回结果
int
def hwloc_memattr_get_best_target(hwloc_topology_t topology,
                                  hwloc_memattr_id_t id,
                                  struct hwloc_location *initiator,
                                  unsigned long flags,
                                  hwloc_obj_t *bestp, hwloc_uint64_t *valuep)
{
  struct hwloc_internal_memattr_s *imattr;  # 定义内部内存属性结构体指针
  hwloc_uint64_t best_value = 0;  # 初始化最佳值为0
  hwloc_obj_t best = NULL;  # 初始化最佳对象为空
  int found = 0;  # 初始化找到标志为0
  unsigned j;  # 定义循环变量j

  if (flags) {  # 如果标志不为0
    errno = EINVAL;  # 设置错误码为无效参数
    return -1;  # 返回-1
  }

  if (id >= topology->nr_memattrs) {  # 如果id大于等于拓扑结构中的内存属性数量
    errno = EINVAL;  # 设置错误码为无效参数
    return -1;  # 返回-1
  }
  imattr = &topology->memattrs[id];  # 获取对应id的内存属性

  if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {  # 如果内存属性标志包含方便属性标志
    /* convenience attributes */  # 注释：方便属性
    for(j=0; ; j++) {  # 循环遍历
      hwloc_obj_t node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, j);  # 获取NUMANODE对象
      hwloc_uint64_t value;  # 定义值变量
      if (!node)  # 如果节点为空
        break;  # 退出循环
      value = hwloc__memattr_get_convenience_value(id, node);  # 获取方便属性值
      hwloc__update_best_target(&best, &best_value, &found,  # 更新最佳目标
                                node, value,
                                imattr->flags & HWLOC_MEMATTR_FLAG_HIGHER_FIRST);
    }
    goto done;  # 跳转到done标签
  }

  /* normal attributes */  # 注释：普通属性

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))  # 如果缓存无效标志未设置
    /* not strictly need */  # 注释：不是严格需要
    hwloc__imattr_refresh(topology, imattr);  # 刷新内存属性

  for(j=0; j<imattr->nr_targets; j++) {  # 循环遍历目标
    struct hwloc_internal_memattr_target_s *imtg = &imattr->targets[j];  # 获取目标结构体
    hwloc_uint64_t value;  # 定义值变量
    if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {  # 如果需要发起者
      /* find the initiator and set its value */  # 注释：找到发起者并设置其值
      struct hwloc_internal_memattr_initiator_s *imi = hwloc__memattr_get_initiator_from_location(imattr, imtg, initiator);  # 获取发起者
      if (!imi)  # 如果发起者为空
        continue;  # 继续下一次循环
      value = imi->value;  # 设置值为发起者的值
    } else {
      /* get the no-initiator value */  # 注释：获取无发起者值
      value = imtg->noinitiator_value;  # 获取无发起者值
    }
    # 更新最佳目标和最佳值，根据给定的对象、数值和标志位
    hwloc__update_best_target(&best, &best_value, &found,
                              imtg->obj, value,
                              imattr->flags & HWLOC_MEMATTR_FLAG_HIGHER_FIRST);
  }

 done:
  # 如果找到了最佳目标
  if (found) {
    # 断言最佳目标存在
    assert(best);
    # 将最佳目标赋值给bestp指向的变量
    *bestp = best;
    # 如果valuep不为空，将最佳值赋值给valuep指向的变量
    if (valuep)
      *valuep = best_value;
    # 返回0表示成功
    return 0;
  } else {
    # 如果未找到最佳目标，设置errno为ENOENT
    errno = ENOENT;
    # 返回-1表示失败
    return -1;
  }
}

/**********************
 * Best initiators
 */

// 更新最佳发起者的函数，根据新的发起者和值更新最佳发起者和值
static void
hwloc__update_best_initiator(struct hwloc_internal_location_s *best_initiator, hwloc_uint64_t *best_value, int *found,
                             struct hwloc_internal_location_s *new_initiator, hwloc_uint64_t new_value,
                             int keep_highest)
{
  // 如果已经找到最佳发起者
  if (*found) {
    // 如果需要保留最高值
    if (keep_highest) {
      // 如果新值小于等于当前最佳值，则不更新
      if (new_value <= *best_value)
        return;
    } else {
      // 如果新值大于等于当前最佳值，则不更新
      if (new_value >= *best_value)
        return;
    }
  }

  // 更新最佳发起者和值
  *best_initiator = *new_initiator;
  *best_value = new_value;
  *found = 1;
}

// 获取最佳发起者的函数
int
hwloc_memattr_get_best_initiator(hwloc_topology_t topology,
                                 hwloc_memattr_id_t id,
                                 hwloc_obj_t target_node,
                                 unsigned long flags,
                                 struct hwloc_location *bestp, hwloc_uint64_t *valuep)
{
  struct hwloc_internal_memattr_s *imattr;
  struct hwloc_internal_memattr_target_s *imtg;
  struct hwloc_internal_location_s best_initiator;
  hwloc_uint64_t best_value;
  int found;
  unsigned i;

  // 如果有标志位，则返回错误
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  // 如果 id 超出内存属性数量，则返回错误
  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  imattr = &topology->memattrs[id];

  // 如果内存属性不需要发起者，则返回错误
  if (!(imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR)) {
    errno = EINVAL;
    return -1;
  }

  // 如果内存属性缓存无效，则刷新缓存
  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    /* not strictly need */
    hwloc__imattr_refresh(topology, imattr);

  // 获取内存属性的目标
  imtg = hwloc__memattr_get_target(imattr, target_node->type, target_node->gp_index, target_node->os_index, 0);
  if (!imtg) {
    errno = EINVAL;
    return -1;
  }

  found = 0;
  for(i=0; i<imtg->nr_initiators; i++) {
    struct hwloc_internal_memattr_initiator_s *imi = &imtg->initiators[i];
    # 调用函数更新最佳发起者，传入最佳发起者指针、最佳数值指针、是否找到标志指针、发起者、数值、以及内存属性标志
    hwloc__update_best_initiator(&best_initiator, &best_value, &found,
                                 &imi->initiator, imi->value,
                                 imattr->flags & HWLOC_MEMATTR_FLAG_HIGHER_FIRST);
  }

  # 如果找到最佳发起者
  if (found) {
    # 如果数值指针存在，则将最佳数值赋给该指针指向的变量
    if (valuep)
      *valuep = best_value;
    # 返回最佳发起者的内部位置
    return from_internal_location(&best_initiator, bestp);
  } else {
    # 如果未找到最佳发起者，则设置错误码为 ENOENT
    errno = ENOENT;
    # 返回 -1
    return -1;
  }
}

/****************************
 * Listing local nodes
 */

static __hwloc_inline int
match_local_obj_cpuset(hwloc_obj_t node, hwloc_cpuset_t cpuset, unsigned long flags)
{
  // 如果标志包含 HWLOC_LOCAL_NUMANODE_FLAG_ALL，则返回1
  if (flags & HWLOC_LOCAL_NUMANODE_FLAG_ALL)
    return 1;
  // 如果标志包含 HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY，并且节点的 cpuset 包含在给定的 cpuset 中，则返回1
  if ((flags & HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY)
      && hwloc_bitmap_isincluded(cpuset, node->cpuset))
    return 1;
  // 如果标志包含 HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY，并且节点的 cpuset 被给定的 cpuset 包含，则返回1
  if ((flags & HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY)
      && hwloc_bitmap_isincluded(node->cpuset, cpuset))
    return 1;
  // 如果节点的 cpuset 等于给定的 cpuset，则返回1
  return hwloc_bitmap_isequal(node->cpuset, cpuset);
}

// 获取本地 NUMA 节点对象
int
hwloc_get_local_numanode_objs(hwloc_topology_t topology,
                              struct hwloc_location *location,
                              unsigned *nrp,
                              hwloc_obj_t *nodes,
                              unsigned long flags)
{
  hwloc_cpuset_t cpuset;
  hwloc_obj_t node;
  unsigned i;

  // 如果 flags 包含未知标志位，则设置 errno 并返回 -1
  if (flags & ~(HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY
                |HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY
                | HWLOC_LOCAL_NUMANODE_FLAG_ALL)) {
    errno = EINVAL;
    return -1;
  }

  // 如果 nrp 为空指针，或者 nrp 不为空且 nodes 为空指针，则设置 errno 并返回 -1
  if (!nrp || (*nrp && !nodes)) {
    errno = EINVAL;
    return -1;
  }

  // 如果 location 为空指针，并且 flags 不包含 HWLOC_LOCAL_NUMANODE_FLAG_ALL，则设置 errno 并返回 -1
  if (!location) {
    if (!(flags & HWLOC_LOCAL_NUMANODE_FLAG_ALL)) {
      errno = EINVAL;
      return -1;
    }
    cpuset = NULL; /* unused */

  } else {
    // 根据 location 的类型设置 cpuset
    if (location->type == HWLOC_LOCATION_TYPE_CPUSET) {
      cpuset = location->location.cpuset;
    } else if (location->type == HWLOC_LOCATION_TYPE_OBJECT) {
      hwloc_obj_t obj = location->location.object;
      while (!obj->cpuset)
        obj = obj->parent;
      cpuset = obj->cpuset;
    } else {
      errno = EINVAL;
      return -1;
    }
  }

  // 遍历 NUMA 节点对象，匹配本地对象的 cpuset 和标志
  i = 0;
  for(node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, 0);
      node;
      node = node->next_cousin) {
    if (!match_local_obj_cpuset(node, cpuset, flags))
      continue;
    if (i < *nrp)
      nodes[i] = node;
    i++;
  }

  // 更新 nrp 的值为 i
  *nrp = i;
  return 0;
}
/**************************************
 * 使用 memattrs 来识别 HBM/DRAM
 */

struct hwloc_memory_tier_s {
  hwloc_obj_t node; // 用于表示 NUMA 节点
  uint64_t local_bw; // 本地带宽
  enum hwloc_memory_tier_type_e { // 内存层级类型枚举
    /* 警告：顺序对于 qsort() 后的 guess_memory_tiers() 很重要 */
    HWLOC_MEMORY_TIER_UNKNOWN, // 未知类型
    HWLOC_MEMORY_TIER_DRAM, // DRAM 类型
    HWLOC_MEMORY_TIER_HBM, // HBM 类型
    HWLOC_MEMORY_TIER_SPM, // SPM 类型，通常是 HBM，我们将使用带宽来确认
    HWLOC_MEMORY_TIER_NVM, // NVM 类型
    HWLOC_MEMORY_TIER_GPU, // GPU 类型
  } type; // 内存层级类型
};

static int compare_tiers(const void *_a, const void *_b)
{
  const struct hwloc_memory_tier_s *a = _a, *b = _b;
  /* 首先按层级类型排序 */
  if (a->type != b->type)
    return a->type - b->type;
  /* 然后按带宽排序 */
  if (a->local_bw > b->local_bw)
    return -1;
  else if (a->local_bw < b->local_bw)
    return 1;
  return 0;
}

int
hwloc_internal_memattrs_guess_memory_tiers(hwloc_topology_t topology)
{
  struct hwloc_internal_memattr_s *imattr;
  struct hwloc_memory_tier_s *tiers;
  unsigned i, j, n;
  const char *env;
  int spm_is_hbm = -1; /* -1 表示从带宽猜测，0 表示不是，1 表示强制 */
  int mark_dram = 1;
  unsigned first_spm, first_nvm;
  hwloc_uint64_t max_unknown_bw, min_spm_bw;

  env = getenv("HWLOC_MEMTIERS_GUESS"); // 获取环境变量 HWLOC_MEMTIERS_GUESS 的值
  if (env) {
    if (!strcmp(env, "none")) { // 如果值为 "none"，则返回 0
      return 0;
    } else if (!strcmp(env, "default")) {
      /* 什么也不做 */
    } else if (!strcmp(env, "spm_is_hbm")) { // 如果值为 "spm_is_hbm"，则假设 SPM-tier 是 HBM，忽略带宽
      hwloc_debug("Assuming SPM-tier is HBM, ignore bandwidth\n");
      spm_is_hbm = 1;
    } else if (HWLOC_SHOW_CRITICAL_ERRORS()) {
      fprintf(stderr, "hwloc: Failed to recognize HWLOC_MEMTIERS_GUESS value %s\n", env);
    }
  }

  imattr = &topology->memattrs[HWLOC_MEMATTR_ID_BANDWIDTH]; // 获取带宽属性

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID)) // 如果带宽属性的缓存无效，则刷新
    hwloc__imattr_refresh(topology, imattr);

  n = hwloc_get_nbobjs_by_depth(topology, HWLOC_TYPE_DEPTH_NUMANODE); // 获取 NUMA 节点的数量
  assert(n);

  tiers = malloc(n * sizeof(*tiers)); // 分配内存空间用于存储内存层级信息
  if (!tiers)
    # 返回错误代码 -1
    return -1;

  # 遍历每个 NUMA 节点
  for(i=0; i<n; i++) {
    # 获取 NUMA 节点对象
    hwloc_obj_t node;
    # DAX 类型
    const char *daxtype;
    # 内部位置结构
    struct hwloc_internal_location_s iloc;
    # 内存属性目标结构
    struct hwloc_internal_memattr_target_s *imtg = NULL;
    # 内存属性发起者结构
    struct hwloc_internal_memattr_initiator_s *imi;

    # 获取指定深度的 NUMA 节点对象
    node = hwloc_get_obj_by_depth(topology, HWLOC_TYPE_DEPTH_NUMANODE, i);
    # 断言节点对象存在
    assert(node);
    # 将节点对象存入 tiers 数组
    tiers[i].node = node;

    # 默认值
    tiers[i].type = HWLOC_MEMORY_TIER_UNKNOWN;
    tiers[i].local_bw = 0; # 未知

    # 获取节点的 DAXType 信息
    daxtype = hwloc_obj_get_info_by_name(node, "DAXType");
    # 标记 NVM、SPM 和 GPU 节点
    if (daxtype && !strcmp(daxtype, "NVM"))
      tiers[i].type = HWLOC_MEMORY_TIER_NVM;
    if (daxtype && !strcmp(daxtype, "SPM"))
      tiers[i].type = HWLOC_MEMORY_TIER_SPM;
    if (node->subtype && !strcmp(node->subtype, "GPUMemory"))
      tiers[i].type = HWLOC_MEMORY_TIER_GPU;

    # 如果 spm_is_hbm 等于 -1
    if (spm_is_hbm == -1) {
      # 遍历内存属性的目标
      for(j=0; j<imattr->nr_targets; j++)
        if (imattr->targets[j].obj == node) {
          imtg = &imattr->targets[j];
          break;
        }
      # 如果 imtg 存在且节点的 cpuset 不为空
      if (imtg && !hwloc_bitmap_iszero(node->cpuset)) {
        iloc.type = HWLOC_LOCATION_TYPE_CPUSET;
        iloc.location.cpuset = node->cpuset;
        # 获取内存属性的发起者
        imi = hwloc__memattr_target_get_initiator(imtg, &iloc, 0);
        # 如果 imi 存在
        if (imi)
          tiers[i].local_bw = imi->value;
      }
    }
  }

  # 对 tiers 数组进行排序
  qsort(tiers, n, sizeof(*tiers), compare_tiers);
  hwloc_debug("Sorting memory tiers...\n");
  # 输出排序后的 tiers 信息
  for(i=0; i<n; i++)
    hwloc_debug("  tier %u = node L#%u P#%u with tier type %d and local BW #%llu\n",
                i,
                tiers[i].node->logical_index, tiers[i].node->os_index,
                tiers[i].type, (unsigned long long) tiers[i].local_bw);

  # 现在有未知的 tiers（按 BW 排序），然后是 SPM tiers（按 BW 排序），然后是 NVM，最后是 GPU

  # 遍历未知的 tiers，并找到它们的 BW
  for(i=0; i<n; i++) {
    # 如果当前层级的类型大于未知内存层级，则跳出循环
    if (tiers[i].type > HWLOC_MEMORY_TIER_UNKNOWN)
      break;
  }
  # 记录第一个SPM层级的索引
  first_spm = i;
  # 从第一个SPM层级获取最大带宽
  if (first_spm > 0)
    max_unknown_bw = tiers[0].local_bw;
  else
    max_unknown_bw = 0;

  # 没有DRAM或HBM层级

  # 遍历SPM层级，并找到它们的带宽
  for(i=first_spm; i<n; i++) {
    if (tiers[i].type > HWLOC_MEMORY_TIER_SPM)
      break;
  }
  # 记录第一个NVM层级的索引
  first_nvm = i;
  # 从最后一个SPM层级获取最小带宽
  if (first_nvm > first_spm)
    min_spm_bw = tiers[first_nvm-1].local_bw;
  else
    min_spm_bw = 0;

  # 如果SPM和未知内存层级之间的带宽相差超过10%，则拆分？
  # 如果同一层级中存在交叉的节点，则中止？

  if (spm_is_hbm == -1) {
    # 如果我们有所有SPM和未知内存层级的带宽
    # 并且所有SPM带宽都是所有未知带宽的2倍
    # 则假设SPM代表HBM，!SPM代表DRAM，因为带宽非常不同
    hwloc_debug("UNKNOWN-memory-tier max bandwidth %llu\n", (unsigned long long) max_unknown_bw);
    hwloc_debug("SPM-memory-tier min bandwidth %llu\n", (unsigned long long) min_spm_bw);
    if (max_unknown_bw > 0 && min_spm_bw > 0 && max_unknown_bw*2 < min_spm_bw) {
      hwloc_debug("assuming SPM means HBM and !SPM means DRAM since bandwidths are very different\n");
      spm_is_hbm = 1;
    } else {
      hwloc_debug("cannot assume SPM means HBM\n");
      spm_is_hbm = 0;
    }
  }

  # 如果SPM代表HBM
  for(i=0; i<first_spm; i++)
    tiers[i].type = HWLOC_MEMORY_TIER_DRAM;
  for(i=first_spm; i<first_nvm; i++)
    tiers[i].type = HWLOC_MEMORY_TIER_HBM;

  # 如果没有SPM层级
  if (first_spm == n)
    mark_dram = 0;

  # 现在应用子类型
  for(i=0; i<n; i++) {
    const char *type = NULL;
    # 如果当前层级已经有子类型，则跳过
    if (tiers[i].node->subtype) /* don't overwrite the existing subtype */
      continue;
    # 根据层级类型设置相应的子类型
    switch (tiers[i].type) {
    case HWLOC_MEMORY_TIER_DRAM:
      if (mark_dram)
        type = "DRAM";
      break;
    case HWLOC_MEMORY_TIER_HBM:
      type = "HBM";
      break;
    case HWLOC_MEMORY_TIER_SPM:
      type = "SPM";
      break;
    case HWLOC_MEMORY_TIER_NVM:
      # 如果内存层级是NVM，则将type设置为"NVM"
      type = "NVM";
      # 跳出switch语句
      break;
    default:
      /* GPU memory is already marked with subtype="GPUMemory",
       * UNKNOWN doesn't deserve any subtype
       */
      # 默认情况下，GPU内存已经标记为subtype="GPUMemory"，UNKNOWN不需要任何子类型
      # 跳出switch语句
      break;
    }
    # 如果type不为空
    if (type) {
      # 输出调试信息，标记节点L#%u P#%u为%s
      hwloc_debug("Marking node L#%u P#%u as %s\n", tiers[i].node->logical_index, tiers[i].node->os_index, type);
      # 为节点的subtype属性分配内存并复制type的值
      tiers[i].node->subtype = strdup(type);
    }
  }
  # 释放tiers数组的内存
  free(tiers);
  # 返回0，表示成功
  return 0;
# 闭合前面的函数定义
```