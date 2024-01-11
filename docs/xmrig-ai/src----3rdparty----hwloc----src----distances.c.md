# `xmrig\src\3rdparty\hwloc\src\distances.c`

```
/*
 * 版权声明
 * 版权所有 © 2010-2022 Inria
 * 版权所有 © 2011-2012 Université Bordeaux
 * 版权所有 © 2011 Cisco Systems, Inc.
 * 请查看顶层目录中的 COPYING 文件。
 */

#include "private/autogen/config.h"  // 包含自动生成的配置文件
#include "hwloc.h"  // 包含公共的 hwloc 头文件
#include "private/private.h"  // 包含私有的 hwloc 头文件
#include "private/debug.h"  // 包含私有的调试头文件
#include "private/misc.h"  // 包含私有的杂项头文件

#include <float.h>  // 包含浮点数的头文件
#include <math.h>  // 包含数学函数的头文件

static struct hwloc_internal_distances_s *
hwloc__internal_distances_from_public(hwloc_topology_t topology, struct hwloc_distances_s *distances);
// 从公共的距离结构转换为内部的距离结构

static void
hwloc__groups_by_distances(struct hwloc_topology *topology, unsigned nbobjs, struct hwloc_obj **objs, uint64_t *values, unsigned long kind, unsigned nbaccuracies, float *accuracies, int needcheck);
// 根据距离分组对象

static void
hwloc_internal_distances_restrict(hwloc_obj_t *objs,
                  uint64_t *indexes,
                  hwloc_obj_type_t *different_types,
                  uint64_t *values,
                  unsigned nbobjs, unsigned disappeared);
// 限制距离对象

static void
hwloc_internal_distances_print_matrix(struct hwloc_internal_distances_s *dist)
{
  unsigned nbobjs = dist->nbobjs;
  hwloc_obj_t *objs = dist->objs;
  hwloc_uint64_t *values = dist->values;
  int gp = !HWLOC_DIST_TYPE_USE_OS_INDEX(dist->unique_type);
  unsigned i, j;

  fprintf(stderr, "%s", gp ? "gp_index" : "os_index");  // 输出 gp_index 或 os_index
  for(j=0; j<nbobjs; j++)
    fprintf(stderr, " % 5d", (int)(gp ? objs[j]->gp_index : objs[j]->os_index));  // 输出对象的 gp_index 或 os_index
  fprintf(stderr, "\n");
  for(i=0; i<nbobjs; i++) {
    fprintf(stderr, "  % 5d", (int)(gp ? objs[i]->gp_index : objs[i]->os_index));  // 输出对象的 gp_index 或 os_index
    for(j=0; j<nbobjs; j++)
      fprintf(stderr, " % 5lld", (long long) values[i*nbobjs + j]);  // 输出距离值
    fprintf(stderr, "\n");
  }
}

/******************************************************
 * 全局初始化、准备、销毁、复制
 */

/* 在拓扑初始化期间调用 */
void hwloc_internal_distances_init(struct hwloc_topology *topology)
{
  topology->first_dist = topology->last_dist = NULL;  // 初始化距离链表
  topology->next_dist_id = 0;  // 初始化下一个距离的 ID
}
/* 在load()函数开始时调用 */
void hwloc_internal_distances_prepare(struct hwloc_topology *topology)
{
  char *env;
  hwloc_localeswitch_declare;

  // 设置默认为启用分组
  topology->grouping = 1;
  // 如果类型过滤器中不包含HWLOC_OBJ_GROUP，则禁用分组
  if (topology->type_filter[HWLOC_OBJ_GROUP] == HWLOC_TYPE_FILTER_KEEP_NONE)
    topology->grouping = 0;
  // 从环境变量HWLOC_GROUPING中获取分组设置，若为0则禁用分组
  env = getenv("HWLOC_GROUPING");
  if (env && !atoi(env))
    topology->grouping = 0;

  // 如果启用分组
  if (topology->grouping) {
    // 初始化下一个子类型
    topology->grouping_next_subkind = 0;

    // 设置分组精度数组
    HWLOC_BUILD_ASSERT(sizeof(topology->grouping_accuracies)/sizeof(*topology->grouping_accuracies) == 5);
    topology->grouping_accuracies[0] = 0.0f;
    topology->grouping_accuracies[1] = 0.01f;
    topology->grouping_accuracies[2] = 0.02f;
    topology->grouping_accuracies[3] = 0.05f;
    topology->grouping_accuracies[4] = 0.1f;
    topology->grouping_nbaccuracies = 5;

    // 初始化本地化开关
    hwloc_localeswitch_init();
    // 从环境变量HWLOC_GROUPING_ACCURACY中获取分组精度设置
    env = getenv("HWLOC_GROUPING_ACCURACY");
    if (!env) {
      /* 只使用0.0的精度 */
      topology->grouping_nbaccuracies = 1;
    } else if (strcmp(env, "try")) {
      /* 使用给定的值作为精度 */
      topology->grouping_nbaccuracies = 1;
      topology->grouping_accuracies[0] = (float) atof(env);
    } /* 否则尝试所有值 */
    hwloc_localeswitch_fini();

    // 设置分组详细信息的默认值
    topology->grouping_verbose = 0;
    // 从环境变量HWLOC_GROUPING_VERBOSE中获取分组详细信息设置
    env = getenv("HWLOC_GROUPING_VERBOSE");
    if (env)
      topology->grouping_verbose = atoi(env);
  }
}

// 释放内部距离结构
static void hwloc_internal_distances_free(struct hwloc_internal_distances_s *dist)
{
  free(dist->name);
  free(dist->different_types);
  free(dist->indexes);
  free(dist->objs);
  free(dist->values);
  free(dist);
}

/* 在拓扑销毁时调用 */
void hwloc_internal_distances_destroy(struct hwloc_topology * topology)
{
  struct hwloc_internal_distances_s *dist, *next = topology->first_dist;
  // 释放所有内部距离结构
  while ((dist = next) != NULL) {
    next = dist->next;
    hwloc_internal_distances_free(dist);
  }
  topology->first_dist = topology->last_dist = NULL;
}
static int hwloc_internal_distances_dup_one(struct hwloc_topology *new, struct hwloc_internal_distances_s *olddist)
{
  // 获取新拓扑结构的内存分配器
  struct hwloc_tma *tma = new->tma;
  // 新的距离结构
  struct hwloc_internal_distances_s *newdist;
  // 旧距离结构中对象的数量
  unsigned nbobjs = olddist->nbobjs;

  // 分配新的距离结构
  newdist = hwloc_tma_malloc(tma, sizeof(*newdist));
  if (!newdist)
    return -1;
  // 复制旧距离结构的名称
  if (olddist->name) {
    newdist->name = hwloc_tma_strdup(tma, olddist->name);
    if (!newdist->name) {
      assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
      hwloc_internal_distances_free(newdist);
      return -1;
    }
  } else {
    newdist->name = NULL;
  }

  // 复制旧距离结构的不同类型标记
  if (olddist->different_types) {
    newdist->different_types = hwloc_tma_malloc(tma, nbobjs * sizeof(*newdist->different_types));
    if (!newdist->different_types) {
      assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
      hwloc_internal_distances_free(newdist);
      return -1;
    }
    memcpy(newdist->different_types, olddist->different_types, nbobjs * sizeof(*newdist->different_types));
  } else
    newdist->different_types = NULL;
  // 复制旧距离结构的唯一类型
  newdist->unique_type = olddist->unique_type;
  // 复制旧距离结构的对象数量
  newdist->nbobjs = nbobjs;
  // 复制旧距离结构的类型
  newdist->kind = olddist->kind;
  // 复制旧距离结构的ID
  newdist->id = olddist->id;

  // 分配新距离结构的索引
  newdist->indexes = hwloc_tma_malloc(tma, nbobjs * sizeof(*newdist->indexes));
  // 分配新距离结构的对象
  newdist->objs = hwloc_tma_calloc(tma, nbobjs * sizeof(*newdist->objs));
  // 重置新距离结构的标记
  newdist->iflags = olddist->iflags & ~HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID; /* must be revalidated after dup() */
  // 分配新距离结构的值
  newdist->values = hwloc_tma_malloc(tma, nbobjs*nbobjs * sizeof(*newdist->values));
  if (!newdist->indexes || !newdist->objs || !newdist->values) {
    assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
    hwloc_internal_distances_free(newdist);
  # 返回错误代码 -1
  return -1;
  # 复制旧距离对象的索引数据到新距离对象
  memcpy(newdist->indexes, olddist->indexes, nbobjs * sizeof(*newdist->indexes));
  # 复制旧距离对象的数值数据到新距离对象
  memcpy(newdist->values, olddist->values, nbobjs*nbobjs * sizeof(*newdist->values));
  # 设置新距离对象的下一个指针为 NULL
  newdist->next = NULL;
  # 设置新距离对象的前一个指针为当前最后一个距离对象
  newdist->prev = new->last_dist;
  # 如果当前最后一个距离对象存在，则将其下一个指针指向新距离对象，否则将新距离对象设置为第一个距离对象
  if (new->last_dist)
    new->last_dist->next = newdist;
  else
    new->first_dist = newdist;
  # 将新距离对象设置为当前最后一个距离对象
  new->last_dist = newdist;
  # 返回成功代码 0
  return 0;
/* 
   复制旧拓扑结构中的距离信息到新的拓扑结构中
   如果 topology->tma 设置了，不能使用 free() 或 realloc()
*/
int hwloc_internal_distances_dup(struct hwloc_topology *new, struct hwloc_topology *old)
{
  struct hwloc_internal_distances_s *olddist;
  int err;
  // 复制旧拓扑结构中的下一个距离 ID 到新的拓扑结构中
  new->next_dist_id = old->next_dist_id;
  // 遍历旧拓扑结构中的距离信息，复制到新的拓扑结构中
  for(olddist = old->first_dist; olddist; olddist = olddist->next) {
    err = hwloc_internal_distances_dup_one(new, olddist);
    if (err < 0)
      return err;
  }
  return 0;
}

/******************************************************
 * 从拓扑结构中移除距离信息
 */

int hwloc_distances_remove(hwloc_topology_t topology)
{
  // 如果拓扑结构未加载，则返回错误
  if (!topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }
  // 如果拓扑结构采用了共享内存地址，则返回错误
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return -1;
  }
  // 销毁拓扑结构中的距离信息
  hwloc_internal_distances_destroy(topology);
  return 0;
}

// 通过深度从拓扑结构中移除距离信息
int hwloc_distances_remove_by_depth(hwloc_topology_t topology, int depth)
{
  struct hwloc_internal_distances_s *dist, *next;
  hwloc_obj_type_t type;

  // 如果拓扑结构未加载，则返回错误
  if (!topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }
  // 如果拓扑结构采用了共享内存地址，则返回错误
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return -1;
  }

  // 切换回类型，因为目前不支持组
  type = hwloc_get_depth_type(topology, depth);
  if (type == (hwloc_obj_type_t)-1) {
    errno = EINVAL;
    return -1;
  }

  next = topology->first_dist;
  while ((dist = next) != NULL) {
    next = dist->next;
    if (dist->unique_type == type) {
      if (next)
    next->prev = dist->prev;
      else
    topology->last_dist = dist->prev;
      if (dist->prev)
    dist->prev->next = dist->next;
      else
    topology->first_dist = dist->next;
      hwloc_internal_distances_free(dist);
    }
  }

  return 0;
}

// 释放并移除拓扑结构中的距离信息
int hwloc_distances_release_remove(hwloc_topology_t topology, struct hwloc_distances_s *distances)
{
  struct hwloc_internal_distances_s *dist = hwloc__internal_distances_from_public(topology, distances);
  if (!dist) {
    errno = EINVAL;
  }
}
    # 返回错误代码 -1
    return -1;
  }
  # 如果距离结构体的前一个指针存在，则将其指向下一个指针
  if (dist->prev)
    dist->prev->next = dist->next;
  # 否则，将拓扑结构体的第一个距离结构体指针指向下一个指针
  else
    topology->first_dist = dist->next;
  # 如果距离结构体的下一个指针存在，则将其指向前一个指针
  if (dist->next)
    dist->next->prev = dist->prev;
  # 否则，将拓扑结构体的最后一个距离结构体指针指向前一个指针
  else
    topology->last_dist = dist->prev;
  # 释放距离结构体
  hwloc_internal_distances_free(dist);
  # 释放距离结构体列表
  hwloc_distances_release(topology, distances);
  # 返回成功代码 0
  return 0;
/*********************************************************
 * 用于向拓扑结构添加距离的后端函数
 */

/* 取消一个距离句柄。目前只在内部需要 */
static void
hwloc_backend_distances_add__cancel(struct hwloc_internal_distances_s *dist)
{
  /* 在 hwloc_backend_distances_add_create() 中所有内容都被设置为 NULL */
  free(dist->name);
  free(dist->indexes);
  free(dist->objs);
  free(dist->different_types);
  free(dist->values);
  free(dist);
}

/* 为后续在拓扑结构中提交距离做准备。
 * 我们复制调用者的名称。
 */
hwloc_backend_distances_add_handle_t
hwloc_backend_distances_add_create(hwloc_topology_t topology,
                                   const char *name, unsigned long kind, unsigned long flags)
{
  struct hwloc_internal_distances_s *dist;

  if (flags) {
    errno = EINVAL;
    goto err;
  }

  dist = calloc(1, sizeof(*dist));
  if (!dist)
    goto err;

  if (name) {
    dist->name = strdup(name); /* 忽略失败 */
    if (!dist->name)
      goto err_with_dist;
  }

  dist->kind = kind;
  dist->iflags = HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED;

  dist->unique_type = HWLOC_OBJ_TYPE_NONE;
  dist->different_types = NULL;
  dist->nbobjs = 0;
  dist->indexes = NULL;
  dist->objs = NULL;
  dist->values = NULL;

  dist->id = topology->next_dist_id++;
  return dist;

 err_with_dist:
  hwloc_backend_distances_add__cancel(dist);
 err:
  return NULL;
}

/* 将对象和值附加到一个距离句柄。
 * 成功时，对象和值数组被附加，并将随距离一起释放。
 * 失败时，句柄被释放。
 */
int
hwloc_backend_distances_add_values(hwloc_topology_t topology __hwloc_attribute_unused,
                                   hwloc_backend_distances_add_handle_t handle,
                                   unsigned nbobjs, hwloc_obj_t *objs,
                                   hwloc_uint64_t *values,
                                   unsigned long flags)
  # 将传入的句柄转换为结构体指针
  struct hwloc_internal_distances_s *dist = handle;
  # 声明变量
  hwloc_obj_type_t unique_type, *different_types = NULL;
  hwloc_uint64_t *indexes = NULL;
  unsigned i, disappeared = 0;

  # 如果距离对象的数量不为零，或者标志位中包含 HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED，则返回无效参数错误
  if (dist->nbobjs || !(dist->iflags & HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED)) {
    errno = EINVAL;
    goto err;
  }

  # 如果标志位不为零，对象数量小于2，或者对象数组为空，或者值数组为空，则返回无效参数错误
  if (flags || nbobjs < 2 || !objs || !values) {
    errno = EINVAL;
    goto err;
  }

  # 检查是否存在空对象，统计空对象数量
  for(i=0; i<nbobjs; i++)
    if (!objs[i])
      disappeared++;
  # 如果存在空对象
  if (disappeared) {
    # 如果所有对象都为空，则返回无此文件或目录错误
    if (disappeared == nbobjs) {
      errno = ENOENT;
      goto err;
    }
    # 限制矩阵范围，去除空对象
    hwloc_internal_distances_restrict(objs, NULL, NULL, values, nbobjs, disappeared);
    nbobjs -= disappeared;
  }

  # 分配索引数组内存
  indexes = malloc(nbobjs * sizeof(*indexes));
  if (!indexes)
    goto err;

  # 检查对象类型是否一致
  unique_type = objs[0]->type;
  for(i=1; i<nbobjs; i++)
    if (objs[i]->type != unique_type) {
      unique_type = HWLOC_OBJ_TYPE_NONE;
      break;
    }
  # 如果对象类型不一致，分配不同类型数组内存
  if (unique_type == HWLOC_OBJ_TYPE_NONE) {
    different_types = malloc(nbobjs * sizeof(*different_types));
    if (!different_types)
      goto err_with_indexes;
    for(i=0; i<nbobjs; i++)
      different_types[i] = objs[i]->type;
  }

  # 设置距离对象的数量、对象数组、标志位、索引数组、唯一类型、不同类型数组、值数组
  dist->nbobjs = nbobjs;
  dist->objs = objs;
  dist->iflags |= HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID;
  dist->indexes = indexes;
  dist->unique_type = unique_type;
  dist->different_types = different_types;
  dist->values = values;

  # 如果存在不同类型的对象，设置标志位
  if (different_types)
    dist->kind |= HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES;

  # 根据唯一类型的不同情况，设置索引数组
  if (HWLOC_DIST_TYPE_USE_OS_INDEX(dist->unique_type)) {
      for(i=0; i<nbobjs; i++)
    dist->indexes[i] = objs[i]->os_index;
    } else {
      for(i=0; i<nbobjs; i++)
    dist->indexes[i] = objs[i]->gp_index;
    }
    # 返回错误码 0
  return 0;

 err_with_indexes:
  # 释放索引内存
  free(indexes);
 err:
  # 取消添加距离信息
  hwloc_backend_distances_add__cancel(dist);
  # 返回错误码 -1
  return -1;
/* attach objects and values to a distance handle.
 * on success, objs and values arrays are attached and will be freed with the distances.
 * on failure, the handle is freed.
 */
static int
hwloc_backend_distances_add_values_by_index(hwloc_topology_t topology __hwloc_attribute_unused,
                                            hwloc_backend_distances_add_handle_t handle,
                                            unsigned nbobjs, hwloc_obj_type_t unique_type, hwloc_obj_type_t *different_types, hwloc_uint64_t *indexes,
                                            hwloc_uint64_t *values)
{
  struct hwloc_internal_distances_s *dist = handle;
  hwloc_obj_t *objs;

  if (dist->nbobjs || !(dist->iflags & HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED)) {
    /* target distances is already set */
    errno = EINVAL;
    goto err;
  }
  if (nbobjs < 2 || !indexes || !values || (unique_type == HWLOC_OBJ_TYPE_NONE && !different_types)) {
    errno = EINVAL;
    goto err;
  }

  objs = malloc(nbobjs * sizeof(*objs));  // 分配存储对象的数组内存空间
  if (!objs)  // 如果分配内存失败
    goto err;

  dist->nbobjs = nbobjs;  // 设置对象数量
  dist->objs = objs;  // 将对象数组赋值给距离结构体
  dist->indexes = indexes;  // 将索引数组赋值给距离结构体
  dist->unique_type = unique_type;  // 设置唯一类型
  dist->different_types = different_types;  // 设置不同类型数组
  dist->values = values;  // 设置值数组

  if (different_types)  // 如果存在不同类型数组
    dist->kind |= HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES;  // 设置距离类型为异构类型

  return 0;  // 成功返回

 err:
  hwloc_backend_distances_add__cancel(dist);  // 取消添加距离处理
  return -1;  // 返回失败
}

/* commit a distances handle.
 * on failure, the handle is freed with its objects and values arrays.
 */
int
hwloc_backend_distances_add_commit(hwloc_topology_t topology,
                                   hwloc_backend_distances_add_handle_t handle,
                                   unsigned long flags)
{
  struct hwloc_internal_distances_s *dist = handle;

  if (!dist->nbobjs || !(dist->iflags & HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED)) {
    /* target distances not ready for commit */
    errno = EINVAL;
    goto err;
  }

  if ((flags & HWLOC_DISTANCES_ADD_FLAG_GROUP) && !dist->objs) {
    // 如果设置了分组标志并且对象数组为空
    /* 无法在没有对象的情况下进行分组，
     * 而且我们也不会从 XML 中进行分组，因为生成 XML 的 hwloc 应该已经进行了分组。
     */
    errno = EINVAL;
    goto err;
  }

  if (topology->grouping && (flags & HWLOC_DISTANCES_ADD_FLAG_GROUP) && !dist->different_types) {
    float full_accuracy = 0.f;
    float *accuracies;
    unsigned nbaccuracies;

    if (flags & HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE) {
      accuracies = topology->grouping_accuracies;
      nbaccuracies = topology->grouping_nbaccuracies;
    } else {
      accuracies = &full_accuracy;
      nbaccuracies = 1;
    }

    if (topology->grouping_verbose) {
      fprintf(stderr, "Trying to group objects using distance matrix:\n");
      hwloc_internal_distances_print_matrix(dist);
    }

    hwloc__groups_by_distances(topology, dist->nbobjs, dist->objs, dist->values,
                   dist->kind, nbaccuracies, accuracies, 1 /* check the first matrix */);
  }

  if (topology->last_dist)
    topology->last_dist->next = dist;
  else
    topology->first_dist = dist;
  dist->prev = topology->last_dist;
  dist->next = NULL;
  topology->last_dist = dist;

  dist->iflags &= ~HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED;
  return 0;

 err:
  hwloc_backend_distances_add__cancel(dist);
  return -1;
/* all-in-one backend function not exported to plugins, only used by XML for now */
// 定义一个内部函数，用于向拓扑结构中添加距离信息，目前仅被 XML 使用
int hwloc_internal_distances_add_by_index(hwloc_topology_t topology, const char *name,
                                          hwloc_obj_type_t unique_type, hwloc_obj_type_t *different_types, unsigned nbobjs, uint64_t *indexes, uint64_t *values,
                                          unsigned long kind, unsigned long flags)
{
  // 声明一个距离信息添加句柄
  hwloc_backend_distances_add_handle_t handle;
  // 声明一个错误码
  int err;

  // 创建一个距离信息添加句柄
  handle = hwloc_backend_distances_add_create(topology, name, kind, 0);
  // 如果句柄创建失败，则跳转到错误处理
  if (!handle)
    goto err;

  // 向句柄中添加指定索引的数值
  err = hwloc_backend_distances_add_values_by_index(topology, handle,
                                                    nbobjs, unique_type, different_types, indexes,
                                                    values);
  // 如果添加失败，则跳转到错误处理
  if (err < 0)
    goto err;

  /* arrays are now attached to the handle */
  // 数组现在已经附加到句柄上
  indexes = NULL;
  different_types = NULL;
  values = NULL;

  // 提交距离信息的添加
  err = hwloc_backend_distances_add_commit(topology, handle, flags);
  // 如果提交失败，则跳转到错误处理
  if (err < 0)
    goto err;

  // 返回成功
  return 0;

 err:
  // 释放内存并返回错误
  free(indexes);
  free(different_types);
  free(values);
  return -1;
}

/* all-in-one backend function not exported to plugins, used by OS backends */
// 定义一个内部函数，用于向拓扑结构中添加距离信息，被操作系统后端使用
int hwloc_internal_distances_add(hwloc_topology_t topology, const char *name,
                                 unsigned nbobjs, hwloc_obj_t *objs, uint64_t *values,
                                 unsigned long kind, unsigned long flags)
{
  // 声明一个距离信息添加句柄
  hwloc_backend_distances_add_handle_t handle;
  // 声明一个错误码
  int err;

  // 创建一个距离信息添加句柄
  handle = hwloc_backend_distances_add_create(topology, name, kind, 0);
  // 如果句柄创建失败，则跳转到错误处理
  if (!handle)
    goto err;

  // 向句柄中添加数值
  err = hwloc_backend_distances_add_values(topology, handle,
                                           nbobjs, objs,
                                           values,
                                           0);
  // 如果添加失败，则跳转到错误处理
  if (err < 0)
    // 跳转到错误处理标签
    goto err;

  /* arrays are now attached to the handle */
  // 现在数组已经附加到句柄上

  // 为硬件拓扑结构添加距离信息并提交
  err = hwloc_backend_distances_add_commit(topology, handle, flags);
  // 如果出现错误则跳转到错误处理标签
  if (err < 0)
    goto err;

  // 返回成功
  return 0;

 err:
  // 释放内存并返回错误
  free(objs);
  free(values);
  return -1;
/********************************
 * 用户API用于添加距离
 */

// 定义距离种类，从操作系统和用户处获取
#define HWLOC_DISTANCES_KIND_FROM_ALL (HWLOC_DISTANCES_KIND_FROM_OS|HWLOC_DISTANCES_KIND_FROM_USER)
// 定义距离种类，包括延迟和带宽
#define HWLOC_DISTANCES_KIND_MEANS_ALL (HWLOC_DISTANCES_KIND_MEANS_LATENCY|HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH)
// 定义所有距离种类
#define HWLOC_DISTANCES_KIND_ALL (HWLOC_DISTANCES_KIND_FROM_ALL|HWLOC_DISTANCES_KIND_MEANS_ALL|HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES)
// 定义添加距离的标志
#define HWLOC_DISTANCES_ADD_FLAG_ALL (HWLOC_DISTANCES_ADD_FLAG_GROUP|HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE)

// 创建距离对象
void * hwloc_distances_add_create(hwloc_topology_t topology,
                                  const char *name, unsigned long kind,
                                  unsigned long flags)
{
  // 如果拓扑未加载，则返回错误
  if (!topology->is_loaded) {
    errno = EINVAL;
    return NULL;
  }
  // 如果拓扑已采用共享内存地址，则返回错误
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return NULL;
  }
  // 如果距离种类不合法，则返回错误
  if ((kind & ~HWLOC_DISTANCES_KIND_ALL)
      || hwloc_weight_long(kind & HWLOC_DISTANCES_KIND_FROM_ALL) != 1
      || hwloc_weight_long(kind & HWLOC_DISTANCES_KIND_MEANS_ALL) != 1) {
    errno = EINVAL;
    return NULL;
  }

  // 调用后端函数创建距离对象
  return hwloc_backend_distances_add_create(topology, name, kind, flags);
}

// 添加距离数值
int hwloc_distances_add_values(hwloc_topology_t topology,
                               void *handle,
                               unsigned nbobjs, hwloc_obj_t *objs,
                               hwloc_uint64_t *values,
                               unsigned long flags)
{
  unsigned i;
  uint64_t *_values;
  hwloc_obj_t *_objs;
  int err;

  // 检查是否有重复的对象
  // 如果有空对象，则返回错误
  for(i=1; i<nbobjs; i++)
    if (!objs[i]) {
      errno = EINVAL;
      goto out;
    }

  // 复制输入数组并将其提供给拓扑
  _objs = malloc(nbobjs*sizeof(hwloc_obj_t));
  _values = malloc(nbobjs*nbobjs*sizeof(*_values));
  if (!_objs || !_values)
  # 跳转到 out_with_arrays 标签处
  goto out_with_arrays;

  # 将 objs 数组中的数据复制到 _objs 数组中
  memcpy(_objs, objs, nbobjs*sizeof(hwloc_obj_t));
  # 将 values 数组中的数据复制到 _values 数组中
  memcpy(_values, values, nbobjs*nbobjs*sizeof(*_values));

  # 调用 hwloc_backend_distances_add_values 函数，将数据添加到拓扑结构中
  err = hwloc_backend_distances_add_values(topology, handle, nbobjs, _objs, _values, flags);
  # 如果出现错误
  if (err < 0) {
    # 在 hwloc_backend_distances_add_values 函数内部取消了 handle
    handle = NULL;
    # 跳转到 out_with_arrays 标签处
    goto out_with_arrays;
  }

  # 返回成功
  return 0;

 out_with_arrays:
  # 释放 _objs 数组的内存
  free(_objs);
  # 释放 _values 数组的内存
  free(_values);
 out:
  # 如果 handle 存在，则取消 handle
  if (handle)
    hwloc_backend_distances_add__cancel(handle);
  # 返回失败
  return -1;
}

int
hwloc_distances_add_commit(hwloc_topology_t topology,
                           void *handle,
                           unsigned long flags)
{
  int err;

  // 检查标志位是否超出范围
  if (flags & ~HWLOC_DISTANCES_ADD_FLAG_ALL) {
    errno = EINVAL;
    goto out;
  }

  // 调用底层函数进行距离添加和提交
  err = hwloc_backend_distances_add_commit(topology, handle, flags);
  if (err < 0) {
    /* handle was canceled inside hwloc_backend_distances_add_commit */
    // 如果在底层函数中取消了句柄，则将句柄置为空
    handle = NULL;
    goto out;
  }

  /* in case we added some groups, see if we need to reconnect */
  // 如果添加了一些组，则检查是否需要重新连接拓扑结构
  hwloc_topology_reconnect(topology, 0);

  return 0;

 out:
  // 如果句柄存在，则取消距离添加
  if (handle)
    hwloc_backend_distances_add__cancel(handle);
  return -1;
}

/* deprecated all-in-one user function */
// 添加距离的过时的一体化用户函数
int hwloc_distances_add(hwloc_topology_t topology,
            unsigned nbobjs, hwloc_obj_t *objs, hwloc_uint64_t *values,
            unsigned long kind, unsigned long flags)
{
  void *handle;
  int err;

  // 创建距离句柄
  handle = hwloc_distances_add_create(topology, NULL, kind, 0);
  if (!handle)
    return -1;

  // 添加距离值
  err = hwloc_distances_add_values(topology, handle, nbobjs, objs, values, 0);
  if (err < 0)
    return -1;

  // 提交距离添加
  err = hwloc_distances_add_commit(topology, handle, flags);
  if (err < 0)
    return -1;

  return 0;
}

/******************************************************
 * Refresh objects in distances
 */

static void
hwloc_internal_distances_restrict(hwloc_obj_t *objs,
                  uint64_t *indexes,
                                  hwloc_obj_type_t *different_types,
                  uint64_t *values,
                  unsigned nbobjs, unsigned disappeared)
{
  unsigned i, newi;
  unsigned j, newj;

  // 重新整理距离对象
  for(i=0, newi=0; i<nbobjs; i++)
    if (objs[i]) {
      for(j=0, newj=0; j<nbobjs; j++)
    if (objs[j]) {
      values[newi*(nbobjs-disappeared)+newj] = values[i*nbobjs+j];
      newj++;
    }
      newi++;
    }

  for(i=0, newi=0; i<nbobjs; i++)
    if (objs[i]) {
      objs[newi] = objs[i];
      if (indexes)
    # 将索引 i 处的值赋给索引 newi 处
    indexes[newi] = indexes[i];
    # 如果存在不同类型的标志，将索引 i 处的值赋给索引 newi 处
    if (different_types)
        different_types[newi] = different_types[i];
    # newi 自增1
    newi++;
static int
hwloc_internal_distances_refresh_one(hwloc_topology_t topology,
                     struct hwloc_internal_distances_s *dist)
{
  // 保存距离结构中的唯一类型
  hwloc_obj_type_t unique_type = dist->unique_type;
  // 保存距离结构中的不同类型数组
  hwloc_obj_type_t *different_types = dist->different_types;
  // 保存距离结构中的对象数量
  unsigned nbobjs = dist->nbobjs;
  // 保存距离结构中的对象数组
  hwloc_obj_t *objs = dist->objs;
  // 保存距离结构中的索引数组
  uint64_t *indexes = dist->indexes;
  // 记录消失的对象数量
  unsigned disappeared = 0;
  unsigned i;

  // 如果对象数组已经有效，则直接返回
  if (dist->iflags & HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID)
    return 0;

  // 遍历对象数组
  for(i=0; i<nbobjs; i++) {
    hwloc_obj_t obj;
    // TODO 使用 cpuset/nodeset 从根节点查找 pus/numas？
    // 比遍历整个层级更快？
    if (HWLOC_DIST_TYPE_USE_OS_INDEX(unique_type)) {
      // 如果唯一类型是 PU，则根据 OS 索引获取 PU 对象
      if (unique_type == HWLOC_OBJ_PU)
    obj = hwloc_get_pu_obj_by_os_index(topology, (unsigned) indexes[i]);
      // 如果唯一类型是 NUMANODE，则根据 OS 索引获取 NUMANODE 对象
      else if (unique_type == HWLOC_OBJ_NUMANODE)
    obj = hwloc_get_numanode_obj_by_os_index(topology, (unsigned) indexes[i]);
      else
    abort();
    } else {
      // 否则根据类型和全局索引获取对象
      obj = hwloc_get_obj_by_type_and_gp_index(topology, different_types ? different_types[i] : unique_type, indexes[i]);
    }
    // 将获取到的对象保存到对象数组中
    objs[i] = obj;
    // 如果对象为空，则消失的对象数量加一
    if (!obj)
      disappeared++;
  }

  // 如果消失的对象数量大于等于对象数量减一，则返回-1
  if (nbobjs-disappeared < 2)
    /* became useless, drop */
    return -1;

  // 如果有消失的对象，则限制距离结构中的对象和值
  if (disappeared) {
    hwloc_internal_distances_restrict(objs, dist->indexes, dist->different_types, dist->values, nbobjs, disappeared);
    dist->nbobjs -= disappeared;
  }

  // 标记对象数组有效
  dist->iflags |= HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID;
  return 0;
}

/* This function may be called with topology->tma set, it cannot free() or realloc() */
void
hwloc_internal_distances_refresh(hwloc_topology_t topology)
{
  struct hwloc_internal_distances_s *dist, *next;

  // 遍历距离结构链表
  for(dist = topology->first_dist; dist; dist = next) {
    next = dist->next;

    // 如果刷新距离结构中的对象失败，则删除该距离结构
    if (hwloc_internal_distances_refresh_one(topology, dist) < 0) {
      assert(!topology->tma || !topology->tma->dontfree); /* this tma cannot fail to allocate */
      if (dist->prev)
    dist->prev->next = next;
      else
    # 将 topology 结构体中的 first_dist 指针指向 next
    topology->first_dist = next;
    # 如果 next 存在
    if (next)
        # 将 next 的 prev 指针指向 dist 的 prev
        next->prev = dist->prev;
    # 如果 next 不存在
    else
        # 将 topology 结构体中的 last_dist 指针指向 dist 的 prev
        topology->last_dist = dist->prev;
    # 释放 dist 指针指向的内存空间
    hwloc_internal_distances_free(dist);
    # 继续下一次循环
    continue;
}

void
hwloc_internal_distances_invalidate_cached_objs(hwloc_topology_t topology)
{
  // 使缓存的对象无效
  struct hwloc_internal_distances_s *dist;
  for(dist = topology->first_dist; dist; dist = dist->next)
    dist->iflags &= ~HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID;
}

/******************************************************
 * User API for getting distances
 */

/* what we actually allocate for user queries, even if we only
 * return the distances part of it.
 */
// 用户查询实际分配的内存结构
struct hwloc_distances_container_s {
  unsigned id;
  struct hwloc_distances_s distances;
};

// 计算距离容器结构的偏移量
#define HWLOC_DISTANCES_CONTAINER_OFFSET ((uintptr_t)(&((struct hwloc_distances_container_s*)NULL)->distances) - (uintptr_t)NULL)
#define HWLOC_DISTANCES_CONTAINER(_d) (struct hwloc_distances_container_s *) ( ((char*)_d) - HWLOC_DISTANCES_CONTAINER_OFFSET )

// 从公共结构获取内部距离结构
static struct hwloc_internal_distances_s *
hwloc__internal_distances_from_public(hwloc_topology_t topology, struct hwloc_distances_s *distances)
{
  struct hwloc_distances_container_s *cont = HWLOC_DISTANCES_CONTAINER(distances);
  struct hwloc_internal_distances_s *dist;
  for(dist = topology->first_dist; dist; dist = dist->next)
    if (dist->id == cont->id)
      return dist;
  return NULL;
}

void
hwloc_distances_release(hwloc_topology_t topology __hwloc_attribute_unused,
            struct hwloc_distances_s *distances)
{
  // 释放距离结构的内存
  struct hwloc_distances_container_s *cont = HWLOC_DISTANCES_CONTAINER(distances);
  free(distances->values);
  free(distances->objs);
  free(cont);
}

// 获取距离的名称
const char *
hwloc_distances_get_name(hwloc_topology_t topology, struct hwloc_distances_s *distances)
{
  struct hwloc_internal_distances_s *dist = hwloc__internal_distances_from_public(topology, distances);
  return dist ? dist->name : NULL;
}

static struct hwloc_distances_s *
hwloc_distances_get_one(hwloc_topology_t topology __hwloc_attribute_unused,
            struct hwloc_internal_distances_s *dist)
{
  // 定义指向距离容器的指针
  struct hwloc_distances_container_s *cont;
  // 定义指向距离结构的指针
  struct hwloc_distances_s *distances;
  // 定义无符号整数变量
  unsigned nbobjs;

  // 分配内存给距离容器
  cont = malloc(sizeof(*cont));
  // 如果内存分配失败，则返回空指针
  if (!cont)
    return NULL;
  // 将距离结构指针指向距离容器内的距离结构
  distances = &cont->distances;

  // 将距离对象数量设置为距离结构中的对象数量
  nbobjs = distances->nbobjs = dist->nbobjs;

  // 分配内存给距离结构中的对象数组
  distances->objs = malloc(nbobjs * sizeof(hwloc_obj_t));
  // 如果内存分配失败，则跳转到标签 out
  if (!distances->objs)
    goto out;
  // 将距离结构中的对象数组复制为输入距离结构中的对象数组
  memcpy(distances->objs, dist->objs, nbobjs * sizeof(hwloc_obj_t));

  // 分配内存给距离结构中的值数组
  distances->values = malloc(nbobjs * nbobjs * sizeof(*distances->values));
  // 如果内存分配失败，则跳转到标签 out_with_objs
  if (!distances->values)
    goto out_with_objs;
  // 将距离结构中的值数组复制为输入距离结构中的值数组
  memcpy(distances->values, dist->values, nbobjs*nbobjs*sizeof(*distances->values));

  // 将距离结构中的类型设置为输入距离结构中的类型
  distances->kind = dist->kind;

  // 将距离容器中的ID设置为输入距离结构中的ID
  cont->id = dist->id;
  // 返回距离结构指针
  return distances;

 out_with_objs:
  // 释放距离结构中的对象数组内存
  free(distances->objs);
 out:
  // 释放距离容器内存
  free(cont);
  // 返回空指针
  return NULL;
}

static int
hwloc__distances_get(hwloc_topology_t topology,
             const char *name, hwloc_obj_type_t type,
             unsigned *nrp, struct hwloc_distances_s **distancesp,
             unsigned long kind, unsigned long flags __hwloc_attribute_unused)
{
  // 定义指向内部距离结构的指针
  struct hwloc_internal_distances_s *dist;
  // 定义无符号整数变量
  unsigned nr = 0, i;

  /* We could return the internal arrays (as const),
   * but it would require to prevent removing distances between get() and free().
   * Not performance critical anyway.
   */

  // 如果标志不为0，则设置错误码并返回-1
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  /* we could refresh only the distances that match, but we won't have many distances anyway,
   * so performance is totally negligible.
   *
   * This is also useful in multithreaded apps that modify the topology.
   * They can call any valid hwloc_distances_get() to force a refresh after
   * changing the topology, so that future concurrent get() won't cause
   * concurrent refresh().
   */
  // 刷新内部距离结构
  hwloc_internal_distances_refresh(topology);

  // 遍历拓扑结构中的距离结构
  for(dist = topology->first_dist; dist; dist = dist->next) {
    // 从输入的类型中获取距离结构的类型
    unsigned long kind_from = kind & HWLOC_DISTANCES_KIND_FROM_ALL;
    // 从输入的类型中获取距离结构的含义
    unsigned long kind_means = kind & HWLOC_DISTANCES_KIND_MEANS_ALL;
    # 如果给定的名称不为空且（分布对象的名称为空或者给定的名称与分布对象的名称不同），则继续循环
    if (name && (!dist->name || strcmp(name, dist->name)))
      continue;

    # 如果给定的类型不是无类型且不等于分布对象的唯一类型，则继续循环
    if (type != HWLOC_OBJ_TYPE_NONE && type != dist->unique_type)
      continue;

    # 如果给定的种类包含位并且给定的种类位与分布对象的种类位不匹配，则继续循环
    if (kind_from && !(kind_from & dist->kind))
      continue;
    # 如果给定的种类含义包含位并且给定的种类含义位与分布对象的种类位不匹配，则继续循环
    if (kind_means && !(kind_means & dist->kind))
      continue;

    # 如果当前索引小于指向结果数组长度的指针所指向的值
    if (nr < *nrp) {
      # 获取一个拓扑结构中的距离对象
      struct hwloc_distances_s *distances = hwloc_distances_get_one(topology, dist);
      # 如果获取失败，则跳转到错误处理
      if (!distances)
        goto error;
      # 将获取到的距离对象存储到结果数组中
      distancesp[nr] = distances;
    }
    # 增加索引值
    nr++;
  }

  # 将剩余的结果数组元素置为空
  for(i=nr; i<*nrp; i++)
    distancesp[i] = NULL;
  # 更新结果数组长度的指针所指向的值
  *nrp = nr;
  # 返回成功
  return 0;

 error:
  # 在发生错误时释放已经获取的距离对象
  for(i=0; i<nr; i++)
    hwloc_distances_release(topology, distancesp[i]);
  # 返回错误
  return -1;
}
// 获取距离信息
int
hwloc_distances_get(hwloc_topology_t topology,
            unsigned *nrp, struct hwloc_distances_s **distancesp,
            unsigned long kind, unsigned long flags)
{
  // 如果标志不为0或拓扑未加载，则设置错误码并返回-1
  if (flags || !topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  // 调用内部函数获取距离信息
  return hwloc__distances_get(topology, NULL, HWLOC_OBJ_TYPE_NONE, nrp, distancesp, kind, flags);
}

// 根据深度获取距离信息
int
hwloc_distances_get_by_depth(hwloc_topology_t topology, int depth,
                 unsigned *nrp, struct hwloc_distances_s **distancesp,
                 unsigned long kind, unsigned long flags)
{
  hwloc_obj_type_t type;

  // 如果标志不为0或拓扑未加载，则设置错误码并返回-1
  if (flags || !topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  /* FIXME: passing the depth of a group level may return group distances at a different depth */
  // 获取深度对应的对象类型
  type = hwloc_get_depth_type(topology, depth);
  // 如果获取失败，则设置错误码并返回-1
  if (type == (hwloc_obj_type_t)-1) {
    errno = EINVAL;
    return -1;
  }

  // 调用内部函数获取距离信息
  return hwloc__distances_get(topology, NULL, type, nrp, distancesp, kind, flags);
}

// 根据名称获取距离信息
int
hwloc_distances_get_by_name(hwloc_topology_t topology, const char *name,
                unsigned *nrp, struct hwloc_distances_s **distancesp,
                unsigned long flags)
{
  // 如果标志不为0或拓扑未加载，则设置错误码并返回-1
  if (flags || !topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  // 调用内部函数获取距离信息
  return hwloc__distances_get(topology, name, HWLOC_OBJ_TYPE_NONE, nrp, distancesp, HWLOC_DISTANCES_KIND_ALL, flags);
}

// 根据类型获取距离信息
int
hwloc_distances_get_by_type(hwloc_topology_t topology, hwloc_obj_type_t type,
                unsigned *nrp, struct hwloc_distances_s **distancesp,
                unsigned long kind, unsigned long flags)
{
  // 如果标志不为0或拓扑未加载，则设置错误码并返回-1
  if (flags || !topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  // 调用内部函数获取距离信息
  return hwloc__distances_get(topology, NULL, type, nrp, distancesp, kind, flags);
}

/******************************************************
 * 根据距离对对象进行分组
 */

// 比较值的函数
static int hwloc_compare_values(uint64_t a, uint64_t b, float accuracy)
{
  // 如果精度不为0且两个值的差的绝对值小于其中一个值乘以精度，则返回1
  if (accuracy != 0.0f && fabsf((float)a-(float)b) < (float)a * accuracy)
    # 返回整数0
    return 0;
  # 如果a小于b，则返回-1；如果a等于b，则返回0；如果a大于b，则返回1
  return a < b ? -1 : a == b ? 0 : 1;
}
/*
 * 如果对象在最小值的传递图中，则将对象放入组中。
 * 返回创建了多少个组，如果找到了一些不完整的距离图，则返回0。
 */
static unsigned
hwloc__find_groups_by_min_distance(unsigned nbobjs,
                   uint64_t *_values,
                   float accuracy,
                   unsigned *groupids,
                   int verbose)
{
  uint64_t min_distance = UINT64_MAX;  // 初始化最小距离为最大可能值
  unsigned groupid = 1;  // 初始化组ID为1
  unsigned i,j,k;  // 定义循环变量
  unsigned skipped = 0;  // 初始化跳过计数为0

#define VALUE(i, j) _values[(i) * nbobjs + (j)]  // 定义宏，用于访问二维数组中的元素

  memset(groupids, 0, nbobjs*sizeof(*groupids));  // 将groupids数组初始化为0

  /* 找到最小距离 */
  for(i=0; i<nbobjs; i++)
    for(j=0; j<nbobjs; j++) /* 检查整个矩阵，根据精度可能不完全对称 */
      if (i != j && VALUE(i, j) < min_distance) /* 这里不考虑精度，我们想要真正的最小值 */
        min_distance = VALUE(i, j);  // 更新最小距离
  hwloc_debug("  found minimal distance %llu between objects\n", (unsigned long long) min_distance);  // 输出找到的最小距离

  if (min_distance == UINT64_MAX)  // 如果最小距离仍然是最大可能值
    return 0;  // 返回0

  /* 用这个距离构建连接的对象组 */
  for(i=0; i<nbobjs; i++) {
    unsigned size;
    unsigned firstfound;

    /* 如果已经分组，则跳过 */
    if (groupids[i])
      continue;

    /* 开始一个新的组 */
    groupids[i] = groupid;  // 将当前对象标记为组ID
    size = 1;  // 组大小为1
    firstfound = i;  // 记录第一个找到的对象

    while (firstfound != (unsigned)-1) {
      /* 我们向组中添加了新对象，第一个对象是firstfound。
       * 重新扫描所有从这些新对象开始的连接（从firstfound开始）到任何其他对象，
       * 以便通过传递性找到新的最小连接的对象。
       */
      unsigned newfirstfound = (unsigned)-1;
      for(j=firstfound; j<nbobjs; j++)
    # 如果当前对象的组ID等于给定的组ID
    if (groupids[j] == groupid)
      # 遍历所有对象
      for(k=0; k<nbobjs; k++)
              # 如果当前对象的组ID为0且与对象j的值的最小距离与精度匹配
              if (!groupids[k] && !hwloc_compare_values(VALUE(j, k), min_distance, accuracy)) {
          # 将当前对象的组ID设置为给定的组ID
          groupids[k] = groupid;
          # 组大小加一
          size++;
          # 如果newfirstfound为无符号整数的最大值
          if (newfirstfound == (unsigned)-1)
        newfirstfound = k;
          # 如果i等于j
          if (i == j)
        # 输出调试信息
        hwloc_debug("  object %u is minimally connected to %u\n", k, i);
          else
            # 输出调试信息
            hwloc_debug("  object %u is minimally connected to %u through %u\n", k, i, j);
        }
      # 将newfirstfound赋值给firstfound
      firstfound = newfirstfound;
    }

    # 如果组大小为1
    if (size == 1) {
      # 取消这个无用的组，忽略这个对象并从下一个对象开始尝试
      groupids[i] = 0;
      # 跳过计数加一
      skipped++;
      # 继续下一次循环
      continue;
    }

    # 验证这个组
    groupid++;
    # 如果详细输出为真
    if (verbose)
      # 输出信息到标准错误流
      fprintf(stderr, " Found transitive graph with %u objects with minimal distance %llu accuracy %f\n",
          size, (unsigned long long) min_distance, accuracy);
  }

  # 如果组ID等于2且跳过计数为0
  if (groupid == 2 && !skipped)
    # 创建一个包含所有对象的单一组，忽略它
    return 0;

  # 返回最后一个ID，因为它也是使用的组ID的数量
  return groupid-1;
}

/* 检查矩阵是否正常 */
static int
hwloc__check_grouping_matrix(unsigned nbobjs, uint64_t *_values, float accuracy, int verbose)
{
  unsigned i,j;
  for(i=0; i<nbobjs; i++) {
    for(j=i+1; j<nbobjs; j++) {
      /* 应该是对称的 */
      if (hwloc_compare_values(VALUE(i, j), VALUE(j, i), accuracy)) {
    if (verbose)
      fprintf(stderr, " Distance matrix asymmetric ([%u,%u]=%llu != [%u,%u]=%llu), aborting\n",
          i, j, (unsigned long long) VALUE(i, j), j, i, (unsigned long long) VALUE(j, i));
    return -1;
      }
      /* 对角线上的值应该小于其他值 */
      if (hwloc_compare_values(VALUE(i, j), VALUE(i, i), accuracy) <= 0) {
    if (verbose)
      fprintf(stderr, " Distance to self not strictly minimal ([%u,%u]=%llu <= [%u,%u]=%llu), aborting\n",
          i, j, (unsigned long long) VALUE(i, j), i, i, (unsigned long long) VALUE(i, i));
    return -1;
      }
    }
  }
  return 0;
}

/*
 * 根据物理距离对对象进行分组。
 */
static void
hwloc__groups_by_distances(struct hwloc_topology *topology,
               unsigned nbobjs,
               struct hwloc_obj **objs,
               uint64_t *_values,
               unsigned long kind,
               unsigned nbaccuracies,
               float *accuracies,
               int needcheck)
{
  unsigned *groupids;
  unsigned nbgroups = 0;
  unsigned i,j;
  int verbose = topology->grouping_verbose;
  hwloc_obj_t *groupobjs;
  unsigned * groupsizes;
  uint64_t *groupvalues;
  unsigned failed = 0;

  if (nbobjs <= 2)
      return;

  if (!(kind & HWLOC_DISTANCES_KIND_MEANS_LATENCY))
    /* 不知道如何使用这些进行分组 */
    /* TODO hwloc__find_groups_by_max_distance() for bandwidth */
    return;

  groupids = malloc(nbobjs * sizeof(*groupids));
  if (!groupids)
    return;

  for(i=0; i<nbaccuracies; i++) {
    # 如果 verbose 为真，则输出尝试根据物理距离和给定精度对对象进行分组的信息
    if (verbose)
      fprintf(stderr, "Trying to group %u %s objects according to physical distances with accuracy %f\n",
          nbobjs, hwloc_obj_type_string(objs[0]->type), accuracies[i]);
    # 如果需要检查并且检查失败，则继续下一次循环
    if (needcheck && hwloc__check_grouping_matrix(nbobjs, _values, accuracies[i], verbose) < 0)
      continue;
    # 根据最小距离找到分组，返回分组数目
    nbgroups = hwloc__find_groups_by_min_distance(nbobjs, _values, accuracies[i], groupids, verbose);
    # 如果存在分组，则跳出循环
    if (nbgroups)
      break;
  }
  # 如果没有分组，则跳转到 out_with_groupids 标签处
  if (!nbgroups)
    goto out_with_groupids;

  # 分配内存给 groupobjs 数组
  groupobjs = malloc(nbgroups * sizeof(*groupobjs));
  # 分配内存给 groupsizes 数组
  groupsizes = malloc(nbgroups * sizeof(*groupsizes));
  # 分配内存给 groupvalues 数组
  groupvalues = malloc(nbgroups * nbgroups * sizeof(*groupvalues));
  # 如果内存分配失败，则执行相应的操作
  if (!groupobjs || !groupsizes || !groupvalues)
    # 跳转到带有组的结束标签
    goto out_with_groups;

      # 创建新的 Group 对象并记录它们的大小
      memset(&(groupsizes[0]), 0, sizeof(groupsizes[0]) * nbgroups);
      for(i=0; i<nbgroups; i++) {
          # 创建 Group 对象
          hwloc_obj_t group_obj, res_obj;
          group_obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
          group_obj->cpuset = hwloc_bitmap_alloc();
          group_obj->attr->group.kind = HWLOC_GROUP_KIND_DISTANCE;
          group_obj->attr->group.subkind = topology->grouping_next_subkind;
          for (j=0; j<nbobjs; j++)
        if (groupids[j] == i+1) {
          # 组装组集
          hwloc_obj_add_other_obj_sets(group_obj, objs[j]);
              groupsizes[i]++;
            }
          hwloc_debug_1arg_bitmap("adding Group object with %u objects and cpuset %s\n",
                                  groupsizes[i], group_obj->cpuset);
          res_obj = hwloc__insert_object_by_cpuset(topology, NULL, group_obj,
                                                   (kind & HWLOC_DISTANCES_KIND_FROM_USER) ? "distances:fromuser:group" : "distances:group");
      # 如果插入失败，res_obj 可能为 NULL
      if (!res_obj)
        failed++;
      # 或者如果我们在分组之前从 XML 导入了组，则 res_obj 可能与 groupobjs 不同
          groupobjs[i] = res_obj;
      }
      topology->grouping_next_subkind++;

      if (failed)
    # 如果这里得到了一个 NULL 组，则不要尝试在上面进行分组，只需保持这个不完整的级别
    goto out_with_groups;

      # 因子化值
      memset(&(groupvalues[0]), 0, sizeof(groupvalues[0]) * nbgroups * nbgroups);
#undef VALUE
#define VALUE(i, j) _values[(i) * nbobjs + (j)]  // 定义宏，用于计算二维数组中的值

#define GROUP_VALUE(i, j) groupvalues[(i) * nbgroups + (j)]  // 定义宏，用于计算二维数组中的值

for(i=0; i<nbobjs; i++)  // 循环遍历 nbobjs 次
    if (groupids[i])  // 如果 groupids[i] 不为 0
      for(j=0; j<nbobjs; j++)  // 再次循环遍历 nbobjs 次
        if (groupids[j])  // 如果 groupids[j] 不为 0
                GROUP_VALUE(groupids[i]-1, groupids[j]-1) += VALUE(i, j);  // 计算并累加值

for(i=0; i<nbgroups; i++)  // 循环遍历 nbgroups 次
    for(j=0; j<nbgroups; j++) {  // 再次循环遍历 nbgroups 次
        unsigned groupsize = groupsizes[i]*groupsizes[j];  // 计算 groupsize
        GROUP_VALUE(i, j) /= groupsize;  // 除以 groupsize
    }

#ifdef HWLOC_DEBUG
hwloc_debug("%s", "generated new distance matrix between groups:\n");  // 输出调试信息
hwloc_debug("%s", "  index");  // 输出调试信息
for(j=0; j<nbgroups; j++)
    hwloc_debug(" % 5d", (int) j); /* print index because os_index is -1 for Groups */  // 输出调试信息
hwloc_debug("%s", "\n");  // 输出调试信息
for(i=0; i<nbgroups; i++) {
    hwloc_debug("  % 5d", (int) i);  // 输出调试信息
    for(j=0; j<nbgroups; j++)
        hwloc_debug(" %llu", (unsigned long long) GROUP_VALUE(i, j));  // 输出调试信息
    hwloc_debug("%s", "\n");  // 输出调试信息
}
#endif

hwloc__groups_by_distances(topology, nbgroups, groupobjs, groupvalues, kind, nbaccuracies, accuracies, 0 /* no need to check generated matrix */);  // 调用函数

out_with_groups:
  free(groupobjs);  // 释放内存
  free(groupsizes);  // 释放内存
  free(groupvalues);  // 释放内存
out_with_groupids:
  free(groupids);  // 释放内存
}

static int
hwloc__distances_transform_remove_null(struct hwloc_distances_s *distances)
{
  hwloc_uint64_t *values = distances->values;  // 获取 distances 结构体中的 values
  hwloc_obj_t *objs = distances->objs;  // 获取 distances 结构体中的 objs
  unsigned i, nb, nbobjs = distances->nbobjs;  // 定义变量并获取 distances 结构体中的 nbobjs
  hwloc_obj_type_t unique_type;  // 定义变量

  for(i=0, nb=0; i<nbobjs; i++)  // 循环遍历 nbobjs 次
    if (objs[i])  // 如果 objs[i] 不为空
      nb++;  // nb 自增

  if (nb < 2) {  // 如果 nb 小于 2
    errno = EINVAL;  // 设置错误码
    return -1;  // 返回 -1
  }

  if (nb == nbobjs)  // 如果 nb 等于 nbobjs
    return 0;  // 返回 0

  hwloc_internal_distances_restrict(objs, NULL, NULL, values, nbobjs, nbobjs-nb);  // 调用函数
  distances->nbobjs = nb;  // 更新 distances 结构体中的 nbobjs

  /* update HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES for convenience */
  unique_type = objs[0]->type;  // 获取类型
  for(i=1; i<nb; i++)  // 循环遍历 nb 次
    if (objs[i]->type != unique_type) {  // 如果类型不相同
      unique_type = HWLOC_OBJ_TYPE_NONE;  // 设置为 HWLOC_OBJ_TYPE_NONE
      break;  // 跳出循环
    }
  # 如果唯一类型是 HWLOC_OBJ_TYPE_NONE，则将距离对象的类型设置为异构类型
  if (unique_type == HWLOC_OBJ_TYPE_NONE)
    distances->kind |= HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES;
  # 否则，将距离对象的类型设置为非异构类型
  else
    distances->kind &= ~HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES;

  # 返回 0，表示成功
  return 0;
}

static int
hwloc__distances_transform_links(struct hwloc_distances_s *distances)
{
  /* FIXME: we should look for the greatest common denominator
   * but we just use the smallest positive value, that's enough for current use-cases.
   * We'll return -1 in other cases.
   */
  // 定义变量divider和values，并初始化为distances的values
  hwloc_uint64_t divider, *values = distances->values;
  // 定义变量i和nbobjs，并初始化为distances的nbobjs
  unsigned i, nbobjs = distances->nbobjs;

  // 如果distances的kind不包含HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH，则设置errno为EINVAL并返回-1
  if (!(distances->kind & HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH)) {
    errno = EINVAL;
    return -1;
  }

  // 将对角线上的值设置为0
  for(i=0; i<nbobjs; i++)
    values[i*nbobjs+i] = 0;

  // 找到最小的正值
  divider = 0;
  for(i=0; i<nbobjs*nbobjs; i++)
    if (values[i] && (!divider || values[i] < divider))
      divider = values[i];

  // 如果divider为0，则返回0
  if (!divider)
    /* only zeroes? do nothing */
    return 0;

  // 检查divider是否能整除所有的值
  for(i=0; i<nbobjs*nbobjs; i++)
    if (values[i]%divider) {
      errno = ENOENT;
      return -1;
    }

  // 将所有的值除以divider
  for(i=0; i<nbobjs*nbobjs; i++)
    values[i] /= divider;

  return 0;
}

static __hwloc_inline int is_nvswitch(hwloc_obj_t obj)
{
  // 如果obj存在且subtype为"NVSwitch"，则返回1，否则返回0
  return obj && obj->subtype && !strcmp(obj->subtype, "NVSwitch");
}

static int
hwloc__distances_transform_merge_switch_ports(hwloc_topology_t topology,
                                              struct hwloc_distances_s *distances)
{
  // 从公共distances创建内部distances
  struct hwloc_internal_distances_s *dist = hwloc__internal_distances_from_public(topology, distances);
  // 定义objs和values，并初始化为distances的objs和values
  hwloc_obj_t *objs = distances->objs;
  hwloc_uint64_t *values = distances->values;
  // 定义变量first、i、j，并初始化为distances的nbobjs
  unsigned first, i, j, nbobjs = distances->nbobjs;

  // 如果dist的name不是"NVLinkBandwidth"，则设置errno为EINVAL并返回-1
  if (strcmp(dist->name, "NVLinkBandwidth")) {
    errno = EINVAL;
    return -1;
  }

  // 找到第一个端口
  first = (unsigned) -1;
  for(i=0; i<nbobjs; i++)
    if (is_nvswitch(objs[i])) {
      first = i;
      break;
    }
  // 如果没有找到端口，则设置errno为ENOENT并返回-1
  if (first == (unsigned)-1) {
    errno = ENOENT;
    return -1;
  }

  // 遍历端口，合并switch的端口
  for(j=i+1; j<nbobjs; j++) {
    # 如果当前对象是 nvswitch
    if (is_nvswitch(objs[j])) {
      # 合并另一个端口
      unsigned k;
      # 遍历所有对象
      for(k=0; k<nbobjs; k++) {
        # 如果是当前对象或者另一个对象，则跳过
        if (k==i || k==j)
          continue;
        # 将另一个对象的值加到当前对象的值上，并将另一个对象的值设为0
        values[k*nbobjs+i] += values[k*nbobjs+j];
        values[k*nbobjs+j] = 0;
        values[i*nbobjs+k] += values[j*nbobjs+k];
        values[j*nbobjs+k] = 0;
      }
      # 将另一个对象的值加到当前对象的值上，并将另一个对象的值设为0
      values[i*nbobjs+i] += values[j*nbobjs+j];
      values[j*nbobjs+j] = 0;
    }
    # 调用者也会调用 REMOVE_NULL 来移除其他端口
    objs[j] = NULL;
  }

  # 返回0表示成功
  return 0;
}
// 对距离矩阵进行传递闭包操作
static int
hwloc__distances_transform_transitive_closure(hwloc_topology_t topology,
                                              struct hwloc_distances_s *distances)
{
  // 从公共距离结构体转换为内部距离结构体
  struct hwloc_internal_distances_s *dist = hwloc__internal_distances_from_public(topology, distances);
  // 获取对象数组和数值数组
  hwloc_obj_t *objs = distances->objs;
  hwloc_uint64_t *values = distances->values;
  unsigned nbobjs = distances->nbobjs;
  unsigned i, j, k;

  // 如果距离矩阵的名称不是"NVLinkBandwidth"，则返回错误
  if (strcmp(dist->name, "NVLinkBandwidth")) {
    errno = EINVAL;
    return -1;
  }

  // 遍历对象数组
  for(i=0; i<nbobjs; i++) {
    hwloc_uint64_t bw_i2sw = 0;
    // 如果是 NVSwitch 对象，则跳过
    if (is_nvswitch(objs[i]))
      continue;
    // 计算到交换机的带宽
    for(k=0; k<nbobjs; k++)
      if (is_nvswitch(objs[k]))
        bw_i2sw += values[i*nbobjs+k];

    // 遍历对象数组
    for(j=0; j<nbobjs; j++) {
      hwloc_uint64_t bw_sw2j = 0;
      // 如果是 NVSwitch 对象或者是同一个对象，则跳过
      if (i == j || is_nvswitch(objs[j]))
        continue;
      // 计算从交换机到目标对象的带宽
      for(k=0; k<nbobjs; k++)
        if (is_nvswitch(objs[k]))
          bw_sw2j += values[k*nbobjs+j];

      // 计算从 i 到 j 的带宽，取最小值
      values[i*nbobjs+j] = bw_i2sw > bw_sw2j ? bw_sw2j : bw_i2sw;
    }
  }

  return 0;
}

// 对距离矩阵进行转换操作
int
hwloc_distances_transform(hwloc_topology_t topology,
                          struct hwloc_distances_s *distances,
                          enum hwloc_distances_transform_e transform,
                          void *transform_attr,
                          unsigned long flags)
{
  // 如果标志或者转换属性不为空，则返回错误
  if (flags || transform_attr) {
    errno = EINVAL;
    return -1;
  }

  // 根据不同的转换类型进行相应的操作
  switch (transform) {
  case HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL:
    return hwloc__distances_transform_remove_null(distances);
  case HWLOC_DISTANCES_TRANSFORM_LINKS:
    return hwloc__distances_transform_links(distances);
  case HWLOC_DISTANCES_TRANSFORM_MERGE_SWITCH_PORTS:
  {
    int err;
    // 合并交换机端口
    err = hwloc__distances_transform_merge_switch_ports(topology, distances);
    if (!err)
      // 移除空值
      err = hwloc__distances_transform_remove_null(distances);
    # 返回错误值
    return err;
  }
  # 如果操作类型为计算传递闭包
  case HWLOC_DISTANCES_TRANSFORM_TRANSITIVE_CLOSURE:
    # 调用函数计算传递闭包
    return hwloc__distances_transform_transitive_closure(topology, distances);
  # 如果操作类型不是上述两种情况
  default:
    # 设置错误码为无效参数
    errno = EINVAL;
    # 返回错误值
    return -1;
  }
# 闭合前面的函数定义
```