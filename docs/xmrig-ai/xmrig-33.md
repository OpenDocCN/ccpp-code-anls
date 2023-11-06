# xmrig源码解析 33

# `src/3rdparty/hwloc/src/distances.c`

这段代码是一个C语言程序，它定义了一个名为"private/autogen/config.h"的头文件，以及两个名为"hwloc.h"和"private/debug.h"的头文件。这些头文件可能会被其他C语言程序或库使用，因为它们定义了一些全局函数和变量。

程序中还有一个名为"private/misc.h"的头文件，但是它没有被使用。

程序的入口点是一个名为"main"的函数，它包括以下语句：
```cppc
#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/debug.h"
```
这些语句可能会被其他C语言程序或库使用，因为它们定义了如何包含其他头文件和初始化调试变量。

接下来的代码会搜索一个名为"private/config.c"的文件，并将其包含在当前可执行文件中。然后，程序将开始执行"private/config.c"中定义的所有函数。

但是，在程序运行之前，开发人员需要首先确保定义了"private/config.c"，因此程序可能需要运行一些预先定义的命令来编译或构建这个程序。


```cpp
/*
 * Copyright © 2010-2022 Inria.  All rights reserved.
 * Copyright © 2011-2012 Université Bordeaux
 * Copyright © 2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/debug.h"
#include "private/misc.h"

#include <float.h>
#include <math.h>

```

hwloc_internal_distances_s* hwloc_internal_distances_create(hwloc_topology *topology, unsigned nbobjs, struct hwloc_obj *obj, uint64_t *values, unsigned long kind, unsigned nbaccuracies, float *accuracies, int needcheck) {
 struct hwloc_internal_distances_s *dist;

 dist = hwloc_make_obj(topology, topology->obj_type, obj->index);
 if (!dist)
   return NULL;

 dist->nbobjs = nbobjs;
 dist->objs = obj;
 dist->values = values;
 dist->kind = kind;
 dist->accuracies = acc;
 dist->needcheck = needcheck;

 return dist;
}

static void
hwloc_groups_by_distances_create(hwloc_topology *topology, unsigned nbobjs, struct hwloc_obj *obj, uint64_t *values, unsigned long kind, unsigned nbaccuracies, float *accuracies, int needcheck) {
 struct hwloc_internal_distances_s *dist;
 struct hwloc_obj_type_t different_types[2];
 hwloc_uint64_t values_array[2];

 dist = hwloc_internal_distances_create(topology, topology->obj_type, obj->index, values, values, kind, nbaccuracies, acc, needcheck);
 if (!dist)
   return;

 dist->nbobjs = nbobjs;
 dist->objs = obj;
 dist->values = values;
 dist->kind = kind;
 dist->accuracies = acc;
 dist->needcheck = needcheck;

 // initialize the different_types array with 2 elements (e.g. filename and file number)
 for(int i=0; i<nbobjs; i++) {
   different_types[i] = hwloc_obj_type(dist->objs[i], topology->file_types);
   values_array[i] = (uint64_t) values[i];
 }

 // initialize the values_array with the values from the input
 for(int i=0; i<nbobjs; i++) {
   values_array[i] = values[i];
 }

 dist->values_array = values_array;
 dist->kind = kind;
 dist->accuracies = acc;
 dist->needcheck = needcheck;
}

static void
hwloc_parameters_distances(hwloc_obj_t *obj,
				  uint64_t *values,
				  unsigned long kind,
				  unsigned nbaccuracies,
				  float *accuracies,
				  int needcheck) {
 struct hwloc_internal_distances_s *dist;
 unsigned i;

 if (HWLOC_DIST_TYPE_USE_OS_INDEX(dist->unique_type) && needcheck) {
   hwloc_groups_by_distances(dist->topology, nbobjs, dist->objs, values, kind, nbaccuracies, dist->accuracies, needcheck);
 } else {
   hwloc_groups_by_distances(dist->topology, nbobjs, dist->objs, values, kind, nbaccuracies, acc, needcheck);

   dist->accuracies = acc;
   dist->needcheck = needcheck;
 }

 for(i=0; i<nbobjs; i++) {
   dist->values[i] = values[i];
 }
}

static void
hwloc_prepare(hwloc_topology *topology) {
 struct hwloc_internal_distances_s *dist;
 unsigned nbobjs, i;
 hwloc_obj_type_t different_types[2];
 hwloc_uint64_t values_array[2];

 dist = hwloc_make_obj(topology, topology->obj_type, -1);
 if (!dist)
   return;

 dist->nbobjs = 0;
 dist->objs = -1;
 dist->values = -1;
 dist->kind = HWLOC_DIST_TYPE_USER;

 // initialize the different_types array with 2 elements (e.g.
 //   "file=filename.ext" and "file=filename.ext.审议")
 for(int i=0; i<nbobjs; i++)
   different_types[i] = hwloc_obj_type(dist->objs[i], topology->file_types);
 for(int i=0; i<nbobjs; i++)
   values_array[i] = (uint64_t) values[i];

 // initialize the values_array with the values from the input
 for(int i=0; i<nbobjs; i++)
   values_array[i] = values[i];

 dist->values_array = values_array;
 dist->kind = dist->kind;
 dist->accuracies = 0;
 dist->needcheck = 0;
 dist->unique_type = HWLOC_DIST_TYPE_USER;
}

static void
hwloc_print_matrix(struct hwloc_internal_distances_s *dist) {
 printf("Distances\n");
 printf("File=% 5d\n", dist->unique_type == HWLOC_DIST_TYPE_USER ? 0 : 1);
 for(int i=0; i<nbobjs; i++) {
   printf(" % 5d\n", i);
   for(int j=0; j<nbobjs; j++) {
     printf("  % 5lld", (long long) values_array[i*nbobjs + j]);
   }
   printf("\n");
 }
}

static void
hwloc_values_print(hwloc_obj_t *obj, struct hwloc_internal_distances_s *dist) {
 printf("Values\n");
 for(int i=0; i<nbobjs;


```cpp
static struct hwloc_internal_distances_s *
hwloc__internal_distances_from_public(hwloc_topology_t topology, struct hwloc_distances_s *distances);

static void
hwloc__groups_by_distances(struct hwloc_topology *topology, unsigned nbobjs, struct hwloc_obj **objs, uint64_t *values, unsigned long kind, unsigned nbaccuracies, float *accuracies, int needcheck);

static void
hwloc_internal_distances_restrict(hwloc_obj_t *objs,
				  uint64_t *indexes,
				  hwloc_obj_type_t *different_types,
				  uint64_t *values,
				  unsigned nbobjs, unsigned disappeared);

static void
hwloc_internal_distances_print_matrix(struct hwloc_internal_distances_s *dist)
{
  unsigned nbobjs = dist->nbobjs;
  hwloc_obj_t *objs = dist->objs;
  hwloc_uint64_t *values = dist->values;
  int gp = !HWLOC_DIST_TYPE_USE_OS_INDEX(dist->unique_type);
  unsigned i, j;

  fprintf(stderr, "%s", gp ? "gp_index" : "os_index");
  for(j=0; j<nbobjs; j++)
    fprintf(stderr, " % 5d", (int)(gp ? objs[j]->gp_index : objs[j]->os_index));
  fprintf(stderr, "\n");
  for(i=0; i<nbobjs; i++) {
    fprintf(stderr, "  % 5d", (int)(gp ? objs[i]->gp_index : objs[i]->os_index));
    for(j=0; j<nbobjs; j++)
      fprintf(stderr, " % 5lld", (long long) values[i*nbobjs + j]);
    fprintf(stderr, "\n");
  }
}

```

This is a C function that defines the behavior of a topology when using the `hwloc_localeswitch` function. It appears to handle the case where the user has specified a local grouping policy, and then specifies a minimum accuracy for the grouping.

The function first sets the topology's grouping flag to 1 if the user has specified a local grouping policy and 0 otherwise. It then checks the type filter for the grouping policy and sets the grouping flag accordingly.

If the user has specified a local grouping policy, the function then initializes the `hwloc_localeswitch` function and sets the accuracy flag to 1. If the user has specified a minimum accuracy for the grouping, the function sets the accuracy flag to the specified value and sets the `hwloc_localeswitch_fini` function to stop the process.

Finally, the function sets the verbosity flag for the `hwloc_localeswitch` function based on the user's specification, and sets the grouping flag to 0 if the user has specified a local grouping policy.


```cpp
/******************************************************
 * Global init, prepare, destroy, dup
 */

/* called during topology init() */
void hwloc_internal_distances_init(struct hwloc_topology *topology)
{
  topology->first_dist = topology->last_dist = NULL;
  topology->next_dist_id = 0;
}

/* called at the beginning of load() */
void hwloc_internal_distances_prepare(struct hwloc_topology *topology)
{
  char *env;
  hwloc_localeswitch_declare;

  topology->grouping = 1;
  if (topology->type_filter[HWLOC_OBJ_GROUP] == HWLOC_TYPE_FILTER_KEEP_NONE)
    topology->grouping = 0;
  env = getenv("HWLOC_GROUPING");
  if (env && !atoi(env))
    topology->grouping = 0;

  if (topology->grouping) {
    topology->grouping_next_subkind = 0;

    HWLOC_BUILD_ASSERT(sizeof(topology->grouping_accuracies)/sizeof(*topology->grouping_accuracies) == 5);
    topology->grouping_accuracies[0] = 0.0f;
    topology->grouping_accuracies[1] = 0.01f;
    topology->grouping_accuracies[2] = 0.02f;
    topology->grouping_accuracies[3] = 0.05f;
    topology->grouping_accuracies[4] = 0.1f;
    topology->grouping_nbaccuracies = 5;

    hwloc_localeswitch_init();
    env = getenv("HWLOC_GROUPING_ACCURACY");
    if (!env) {
      /* only use 0.0 */
      topology->grouping_nbaccuracies = 1;
    } else if (strcmp(env, "try")) {
      /* use the given value */
      topology->grouping_nbaccuracies = 1;
      topology->grouping_accuracies[0] = (float) atof(env);
    } /* otherwise try all values */
    hwloc_localeswitch_fini();

    topology->grouping_verbose = 0;
    env = getenv("HWLOC_GROUPING_VERBOSE");
    if (env)
      topology->grouping_verbose = atoi(env);
  }
}

```

这两段代码定义了 hwloc_internal_distances_free 和 hwloc_internal_distances_destroy 函数。

hwloc_internal_distances_free 函数用于释放 hwloc_internal_distances 结构体内部的数据成员，包括 name、different_types、indexes、objs 和 values。

hwloc_internal_distances_destroy 函数用于在 topology 结构体中释放 hwloc_internal_distances 结构体。这个函数在 topology 结构体销毁时被调用，确保所有 hwloc_internal_distances 结构体已被正确释放。

这两个函数都是 hwloc 库中的私有函数，用于管理内部距离。在 hwloc 库中，内部距离是一种用于计算布局树中组件之间的距离的机制。通过这些函数，用户可以自由地管理内部距离，以便在 hwloc 库中正确地使用组件。


```cpp
static void hwloc_internal_distances_free(struct hwloc_internal_distances_s *dist)
{
  free(dist->name);
  free(dist->different_types);
  free(dist->indexes);
  free(dist->objs);
  free(dist->values);
  free(dist);
}

/* called during topology destroy */
void hwloc_internal_distances_destroy(struct hwloc_topology * topology)
{
  struct hwloc_internal_distances_s *dist, *next = topology->first_dist;
  while ((dist = next) != NULL) {
    next = dist->next;
    hwloc_internal_distances_free(dist);
  }
  topology->first_dist = topology->last_dist = NULL;
}

```

The function `hwloc_internal_distances_free` takes a single parameter `newdist`, which is an instance of the `HWLOC_INTERNAL_DIST` structure. This function is used to free memory allocated by `hwloc_internally_distances_alloc`, or to perform any other cleanup related to the allocation.

The function first calls `hwloc_tma_malloc` function to allocate memory for the new distribution object, and then calls `memcpy` function to copy the old distribution object's data to the new object.

Then it sets the object's `kind`, `id`, and checks for duplicates. After that, it checks if the allocation failed and if so, it frees the memory and return -1.

If the allocation succeeds, it creates a new object in the same TMA, and sets its `next` and `prev` pointers to the last object and the first object in the queue, respectively. Finally, it returns 0.


```cpp
static int hwloc_internal_distances_dup_one(struct hwloc_topology *new, struct hwloc_internal_distances_s *olddist)
{
  struct hwloc_tma *tma = new->tma;
  struct hwloc_internal_distances_s *newdist;
  unsigned nbobjs = olddist->nbobjs;

  newdist = hwloc_tma_malloc(tma, sizeof(*newdist));
  if (!newdist)
    return -1;
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
  newdist->unique_type = olddist->unique_type;
  newdist->nbobjs = nbobjs;
  newdist->kind = olddist->kind;
  newdist->id = olddist->id;

  newdist->indexes = hwloc_tma_malloc(tma, nbobjs * sizeof(*newdist->indexes));
  newdist->objs = hwloc_tma_calloc(tma, nbobjs * sizeof(*newdist->objs));
  newdist->iflags = olddist->iflags & ~HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID; /* must be revalidated after dup() */
  newdist->values = hwloc_tma_malloc(tma, nbobjs*nbobjs * sizeof(*newdist->values));
  if (!newdist->indexes || !newdist->objs || !newdist->values) {
    assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
    hwloc_internal_distances_free(newdist);
    return -1;
  }

  memcpy(newdist->indexes, olddist->indexes, nbobjs * sizeof(*newdist->indexes));
  memcpy(newdist->values, olddist->values, nbobjs*nbobjs * sizeof(*newdist->values));

  newdist->next = NULL;
  newdist->prev = new->last_dist;
  if (new->last_dist)
    new->last_dist->next = newdist;
  else
    new->first_dist = newdist;
  new->last_dist = newdist;

  return 0;
}

```

这段代码是一个名为`hwloc_internal_distances_dup`的函数，其作用是复制一个名为`old`的`hwloc_topology`结构体到名为`new`的结构体中。

函数中首先定义了一个内部距离数组`distances`，该数组长度为`hwloc_topology`结构体中`distances`成员的长度。接着定义了一个指向`hwloc_internal_distances_s`类型的指针`olddist`，用于存储从`old`结构体中复制来的距离。

函数体中，首先执行了将`olddist`指向的`hwloc_internal_distances_s`类型的指针`olddist`的`next_dist_id`复制到`new.next_dist_id`中的操作。然后，使用一个for循环遍历`old`结构体中所有距离的节点，并使用`hwloc_internal_distances_dup_one`函数将它们复制到`new`结构体中。如果复制过程中出现错误，函数返回该错误。否则，函数返回0，表示复制成功。


```cpp
/* This function may be called with topology->tma set, it cannot free() or realloc() */
int hwloc_internal_distances_dup(struct hwloc_topology *new, struct hwloc_topology *old)
{
  struct hwloc_internal_distances_s *olddist;
  int err;
  new->next_dist_id = old->next_dist_id;
  for(olddist = old->first_dist; olddist; olddist = olddist->next) {
    err = hwloc_internal_distances_dup_one(new, olddist);
    if (err < 0)
      return err;
  }
  return 0;
}

/******************************************************
 * Remove distances from the topology
 */

```

这两段代码是用于在 hwloc_topology_t 结构体中移除指定深度处的距离信息。

在 hwloc_distances_remove 函数中，首先检查 topology 对象是否已加载。如果未加载，则错误并返回 -1。然后检查 topology 对象是否已采用共享内存布局，如果是，则错误并返回 -1。接着，函数调用 hwloc_internal_distances_destroy 函数来移除指定深度处的距离信息。最后，函数返回 0，表示成功移除指定深度处的距离信息。

在 hwloc_distances_remove_by_depth 函数中，首先检查 topology 对象是否已加载。如果未加载，则错误并返回 -1。然后检查 topology 对象是否已采用共享内存布局，如果是，则错误并返回 -1。接着，函数会遍历指定深度处的所有距离，并将距离对象（如果有的话）复制到 next 变量。接下来，函数会遍历下一个距离，并将其作为 prev 变量的下一跳。然后，函数会检查下一个距离的prev变量。如果上一个距离对象的next变量有效，则将 next 作为上一个距离对象的next变量。然后，函数会尝试释放距离对象，如果成功，则 hwloc_internal_distances_free 函数返回 0。


```cpp
int hwloc_distances_remove(hwloc_topology_t topology)
{
  if (!topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return -1;
  }
  hwloc_internal_distances_destroy(topology);
  return 0;
}

int hwloc_distances_remove_by_depth(hwloc_topology_t topology, int depth)
{
  struct hwloc_internal_distances_s *dist, *next;
  hwloc_obj_type_t type;

  if (!topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return -1;
  }

  /* switch back to types since we don't support groups for now */
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

```

这段代码是一个名为 `hwloc_distances_release_remove` 的函数，它是 `hwloc_topology_t` 结构体的一部分。

它的作用是处理 `hwloc_distances_s` 类型的数据，释放内存并输出结果。

具体来说，它接收一个 `hwloc_topology_t` 类型的参数 `topology` 和一个 `hwloc_distances_s` 类型的参数 `distances`。

首先，它将 `distances` 数据从 `hwloc_distances_s` 类型的数据结构中转换为 `struct hwloc_internal_distances_s` 类型的数据结构，如果转换失败，则返回 `-1`，并输出错误信息。

接着，它处理 `dist` 数据的前驱和后继指针。如果 `dist` 有前驱，则将前驱的 `next` 指针指向 `dist`，否则，设置 `topology` 的第一个距离为 `dist` 的下一个距离。

然后，它处理 `dist` 数据的后驱和前驱指针。如果 `dist` 有后驱，则将后驱的 `prev` 指针指向 `dist`，否则，设置 `topology` 的最后一个距离为 `dist` 的下一个距离。

最后，它释放 `dist` 数据，并输出结果。

由于 `hwloc_distances_s` 类型的数据结构中包含的是 `int` 类型的数据，而不是 `struct hwloc_internal_distances_s` 类型的数据结构，因此在输入参数 `topology` 和 `distances` 时需要进行类型转换。


```cpp
int hwloc_distances_release_remove(hwloc_topology_t topology,
				   struct hwloc_distances_s *distances)
{
  struct hwloc_internal_distances_s *dist = hwloc__internal_distances_from_public(topology, distances);
  if (!dist) {
    errno = EINVAL;
    return -1;
  }
  if (dist->prev)
    dist->prev->next = dist->next;
  else
    topology->first_dist = dist->next;
  if (dist->next)
    dist->next->prev = dist->prev;
  else
    topology->last_dist = dist->prev;
  hwloc_internal_distances_free(dist);
  hwloc_distances_release(topology, distances);
  return 0;
}

```



这段代码定义了一个名为 `hwloc_backend_distances_add__cancel` 的函数，它是 `hwloc_backend_distances_add` 函数的实现在 cancellation 操作上。

具体来说，这个函数接收一个 `struct hwloc_internal_distances_s` 类型的参数 `dist`，执行以下操作：

1. 释放 `dist` 指向的距离名称(名字和索引)`name` 和索引数组 `indexes`。
2. 释放 `dist` 指向的对象 `obj`。
3. 释放 `dist` 指向的距离类型数组 `different_types`。
4. 释放 `dist` 指向的值数组 `values`。
5. 释放 `dist` 指向自身，这个函数只会在 `hwloc_backend_distances_add` 函数创建 `dist` 对象之后执行一次。

同时，这个函数也重置了 `dist` 参数的值为 `NULL`，以便在之后的 `hwloc_backend_distances_add` 函数中不要再误用已经取消的距离。


```cpp
/*********************************************************
 * Backend functions for adding distances to the topology
 */

/* cancel a distances handle. only needed internally for now */
static void
hwloc_backend_distances_add__cancel(struct hwloc_internal_distances_s *dist)
{
  /* everything is set to NULL in hwloc_backend_distances_add_create() */
  free(dist->name);
  free(dist->indexes);
  free(dist->objs);
  free(dist->different_types);
  free(dist->values);
  free(dist);
}

```

这段代码定义了一个名为 `hwloc_backend_distances_add_handle_t` 的结构体，用于存储在topology中计算距离的handle。handle被用于在后续的commit操作中进行调用，允许您在以后提交时获取与调用者不同的距离信息。

在函数的实现中，首先创建一个名为`dist`的`hwloc_internal_distances_s`结构体，该结构体用于存储距离信息。如果创建失败，将打印错误并跳转到错误处理函数。

接下来，尝试从调用者传入一个名字，并将其复制到`dist`结构体的`name`成员中。然后，将`dist`结构体的`kind`成员设置为指定kind，`iflags`成员设置为`HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED`，以启用距离信息，但不会将其提交到目标系统。

最后，对于传入的名字，在`hwloc_backend_distances_add__cancel`函数中打印错误并跳转到错误处理函数，以确保在尝试取消之前，任何已经创建的距离信息都被正确地释放。

在函数调用时，传入的参数包含在`topology`参数中，例如：
```cppperl
hwloc_topology_t topology;
topology = hwloc_topology_new("my_topology");
```
您可以在之后的提交操作中使用上面定义的`hwloc_backend_distances_add_handle_t`结构体来获取计算距离信息。


```cpp
/* prepare a distances handle for later commit in the topology.
 * we duplicate the caller's name.
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
    dist->name = strdup(name); /* ignore failure */
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

```

This function appears to be used for managing object data structures that are stored in the HWLOC (Highway Workers' Local Operating System) system. It appears to be used to allocate memory for object data structures and to set some of the indexes and flags for those data structures.

The function takes in a number of parameters, including the number of object data structures to allocate memory for and a pointer to an array of indexes for those data structures. It also takes in a pointer to an array of object data structures, which is passed by value to the function caller.

The function first checks for a null pointer to the indexes array and, if one is found, frees any memory allocated for it. It then sets the `nbobjs` field of the input parameters to the specified number of object data structures and sets the `dist->kind` field to indicate that the object data structures are part of a heterogeneous distribution.

The function then sets the indexes field of the input parameters to the memory address of the index array, if one has been allocated for it. It does this by setting the indexes field of the input parameters to the `os_index` field of each object data structure, if one has been allocated for it. If the indexes field of the input parameters is not set, the function sets it to the `gp_index` field of each object data structure.

The function also sets the `unique_type` field of the input parameters to the type of the most common object data structure in the distribution, if one exists. If the most common type is `HWLOC_OBJ_TYPE_NONE`, the function sets the function to handle heterogeneous types and allocate memory for the object data structures.

The function allocates memory for the object data structures and sets the correct indexes and flags for them. It also sets the `dist->nbobjs` field to the specified number of object data structures and sets the `dist->objs` and `dist->iflags` fields to hold a pointer to the allocated memory and to indicate that the object data structures are valid, respectively.

The function returns 0 on success and -1 on failure. If the function fails, it prints an error message and frees any memory allocated for the indexes array.


```cpp
/* attach objects and values to a distances handle.
 * on success, objs and values arrays are attached and will be freed with the distances.
 * on failure, the handle is freed.
 */
int
hwloc_backend_distances_add_values(hwloc_topology_t topology __hwloc_attribute_unused,
                                   hwloc_backend_distances_add_handle_t handle,
                                   unsigned nbobjs, hwloc_obj_t *objs,
                                   hwloc_uint64_t *values,
                                   unsigned long flags)
{
  struct hwloc_internal_distances_s *dist = handle;
  hwloc_obj_type_t unique_type, *different_types = NULL;
  hwloc_uint64_t *indexes = NULL;
  unsigned i, disappeared = 0;

  if (dist->nbobjs || !(dist->iflags & HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED)) {
    /* target distances is already set */
    errno = EINVAL;
    goto err;
  }

  if (flags || nbobjs < 2 || !objs || !values) {
    errno = EINVAL;
    goto err;
  }

  /* is there any NULL object? (useful in case of problem during insert in backends) */
  for(i=0; i<nbobjs; i++)
    if (!objs[i])
      disappeared++;
  if (disappeared) {
    /* some objects are NULL */
    if (disappeared == nbobjs) {
      /* nothing left, drop the matrix */
      errno = ENOENT;
      goto err;
    }
    /* restrict the matrix */
    hwloc_internal_distances_restrict(objs, NULL, NULL, values, nbobjs, disappeared);
    nbobjs -= disappeared;
  }

  indexes = malloc(nbobjs * sizeof(*indexes));
  if (!indexes)
    goto err;

  unique_type = objs[0]->type;
  for(i=1; i<nbobjs; i++)
    if (objs[i]->type != unique_type) {
      unique_type = HWLOC_OBJ_TYPE_NONE;
      break;
    }
  if (unique_type == HWLOC_OBJ_TYPE_NONE) {
    /* heterogeneous types */
    different_types = malloc(nbobjs * sizeof(*different_types));
    if (!different_types)
      goto err_with_indexes;
    for(i=0; i<nbobjs; i++)
      different_types[i] = objs[i]->type;
  }

  dist->nbobjs = nbobjs;
  dist->objs = objs;
  dist->iflags |= HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID;
  dist->indexes = indexes;
  dist->unique_type = unique_type;
  dist->different_types = different_types;
  dist->values = values;

  if (different_types)
    dist->kind |= HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES;

  if (HWLOC_DIST_TYPE_USE_OS_INDEX(dist->unique_type)) {
      for(i=0; i<nbobjs; i++)
	dist->indexes[i] = objs[i]->os_index;
    } else {
      for(i=0; i<nbobjs; i++)
	dist->indexes[i] = objs[i]->gp_index;
    }

  return 0;

 err_with_indexes:
  free(indexes);
 err:
  hwloc_backend_distances_add__cancel(dist);
  return -1;
}

```

This function appears to be part of the `hwloc_backend_distances_add_handle_t` structure, which is a handle for the `hwloc_backend_distances_add_values_by_index` function.

It takes in a topology object and a handle for the previous operation. It checks if the handle has already been freed and if the topology object has any tags indicating that the object should not be part of a composite handle. If either condition is true, the function returns an error code.

If the handle has not been freed and the topology object does not have any tags to indicate non-committed use, the function creates a new internal distances array and sets its constructor parameters to the input values. It sets the handle's internal flag to indicate that this object should be part of a composite handle, and sets its kind to indicate that it is a heterogeneous type.

If the topology object has any tags indicating that the object should not be part of a composite handle, the function returns an error code.

If the function successfully adds the values and returns, it attaches the internal distances array and the handle to the topology object, and frees the memory分配 dynamically.


```cpp
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

  objs = malloc(nbobjs * sizeof(*objs));
  if (!objs)
    goto err;

  dist->nbobjs = nbobjs;
  dist->objs = objs;
  dist->indexes = indexes;
  dist->unique_type = unique_type;
  dist->different_types = different_types;
  dist->values = values;

  if (different_types)
    dist->kind |= HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES;

  return 0;

 err:
  hwloc_backend_distances_add__cancel(dist);
  return -1;
}

```

This code is a part of the `hwloc_internal_distances_print_matrix` function, which is a part of the `hwloc_internal_distances` function. This function is used to print the internal distances of a Direct久坐 Direct Information Level (DIL) data structure according to the configuration flags passed to it.

The function takes a single parameter, `dist`, which is an instance of the `HwlocDist` data structure that contains the internal distances for a particular data layout. The function first checks if there are any objects associated with the distance matrix and then groups them according to the configuration flags passed to it.

If the `HWLOC_DISTANCES_ADD_FLAG_GROUP` flag is set and there are no objects, the function prints the distance matrix. If the `HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE` flag is set, the function uses the `topology` object to determine the accuracy of the distances and prints the result.

If the `topology` object does not exist, the function prints an error message and returns `EINVAL`. If the `topology` object does exist and there are objects, the function groups the objects according to the configuration flags passed to it and prints the result.

If the `topology` object does not exist and there are objects, the function prints an error message and returns `EINVAL`. If the `topology` object does exist and there are objects, the function groups the objects according to the configuration flags passed to it and prints the result.

If the `topology` object does not exist, the function prints an error message and returns `EINVAL`. If the `topology` object does exist and there are objects, the function groups the objects according to the configuration flags passed to it and prints the result.


```cpp
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
    /* cannot group without objects,
     * and we don't group from XML anyway since the hwloc that generated the XML should have grouped already.
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
}

```

这是一个仅在 XML 上下文中使用的内部函数，用于计算硬件位置布局中指定名称的距离。函数接受一个硬件位置布局 topology、指定名称 name、指定唯一的对象类型 unique_type 和不同类型的数量 nbobjs，以及一个指向距离的数组长度 *indexes 和每个索引对应的价值 values。函数返回计算过程的状况，如果成功则返回 0，否则返回 -1。

函数内部使用 hwloc_backend_distances_add_create 和 hwloc_backend_distances_add_values_by_index 函数来创建和设置 handle，以及获取和设置 nbobjs，unique_type 和 different_types 变量。然后，函数使用 hwloc_backend_distances_add_commit 函数提交操作，并在完成时释放这些资源。

注意，函数没有输出，也没有定义它的头文件和函数指针。


```cpp
/* all-in-one backend function not exported to plugins, only used by XML for now */
int hwloc_internal_distances_add_by_index(hwloc_topology_t topology, const char *name,
                                          hwloc_obj_type_t unique_type, hwloc_obj_type_t *different_types, unsigned nbobjs, uint64_t *indexes, uint64_t *values,
                                          unsigned long kind, unsigned long flags)
{
  hwloc_backend_distances_add_handle_t handle;
  int err;

  handle = hwloc_backend_distances_add_create(topology, name, kind, 0);
  if (!handle)
    goto err;

  err = hwloc_backend_distances_add_values_by_index(topology, handle,
                                                    nbobjs, unique_type, different_types, indexes,
                                                    values);
  if (err < 0)
    goto err;

  /* arrays are now attached to the handle */
  indexes = NULL;
  different_types = NULL;
  values = NULL;

  err = hwloc_backend_distances_add_commit(topology, handle, flags);
  if (err < 0)
    goto err;

  return 0;

 err:
  free(indexes);
  free(different_types);
  free(values);
  return -1;
}

```

这是一段用于在操作系统（OS）底层实现（hwloc）中的内部距离（distances）增加的函数。其目的是在给定的硬件位置（hwloc_topology_t）和名称（const char *）上，统计硬件对象（hwloc_obj_t）的数量，并返回统计结果。函数支持几种输入参数：

1. `topology`：表示输入的拓扑结构。可以是 `hwloc_topology_t` 结构体或任何自定义的类似结构体。
2. `name`：表示要统计的硬件位置的名称。如果 `topology` 中的 `name` 存在，则使用这个名称来搜索硬件位置。
3. `nbobjs`：表示要统计的硬件对象的个数。这个参数用于在统计过程中减半。
4. `objs`：指向被统计的硬件对象的指针。如果有 `nbobjs` 个硬件对象，则这个参数是一个 `hwloc_obj_t` 指针，指向这个数组的第一个元素。
5. `values`：一个 `uint64_t` 类型的数组，用于存储统计结果。在没有传递给函数的硬件位置和名称的情况下，这个数组将包含所有硬件位置的名称的哈希值。
6. `kind`：表示要统计的硬件对象的类型。可以是 `hwloc_obj_kind_t` 结构体中的 `hwloc_obj_kind_addition`、`hwloc_obj_kind_connection` 等。如果不传递，则默认值为 `hwloc_obj_kind_addition`。
7. `flags`：表示函数调用时传递给 `hwloc_backend_distances_add_commit` 的标志。这些 flags 的值将影响统计结果的粒度。具体来说，这些 flag 的值可能包括：
 - `__unified_addr`：如果 `topology` 中的 `name` 与 `hwloc_topology_internal_name` 中的名称相同，则认为这个硬件位置与指定的拓扑结构在同一个位置。
 - `__device_or_device_with_controllers`：如果 `topology` 中的 `device_or_device_with_controllers` 设置为 `1`，则认为这个硬件位置是一个控制器（device or device with controllers）。

函数在统计过程中会用到两个辅助函数：

1. `hwloc_backend_distances_add_handle_t`：用于在拓扑结构中查找硬件位置并创建一个 `hwloc_backend_distances_add_handle_t` 类型的处理。
2. `hwloc_backend_distances_add_values_t`：这个类型在调用时表示要使用的统计结果的粒度。可以包含以下几个成员：
 - `nbobjs`：表示要统计的硬件对象的个数。
 - `objs`：指向被统计的硬件对象的指针。
 - `values`：一个 `uint64_t` 类型的数组，用于存储统计结果。如果没有传递给函数的硬件位置和名称，则这个数组将包含所有硬件位置的名称的哈希值。



```cpp
/* all-in-one backend function not exported to plugins, used by OS backends */
int hwloc_internal_distances_add(hwloc_topology_t topology, const char *name,
                                 unsigned nbobjs, hwloc_obj_t *objs, uint64_t *values,
                                 unsigned long kind, unsigned long flags)
{
  hwloc_backend_distances_add_handle_t handle;
  int err;

  handle = hwloc_backend_distances_add_create(topology, name, kind, 0);
  if (!handle)
    goto err;

  err = hwloc_backend_distances_add_values(topology, handle,
                                           nbobjs, objs,
                                           values,
                                           0);
  if (err < 0)
    goto err;

  /* arrays are now attached to the handle */
  objs = NULL;
  values = NULL;

  err = hwloc_backend_distances_add_commit(topology, handle, flags);
  if (err < 0)
    goto err;

  return 0;

 err:
  free(objs);
  free(values);
  return -1;
}

```

This code defines a function called `hwloc_distances_add_create`, which adds a new user API for adding distances to a HWLOC topology.

The function takes three arguments:

* `topology`: a `hwloc_topology_t` structure representing the HWLOC topology to modify.
* `name`: a string representing the name of the new distance to add.
* `kind`: a `unsigned long` representing the kind of distance to add. The allowed values are `HWLOC_DISTANCES_KIND_FROM_ALL`, `HWLOC_DISTANCES_KIND_MEANS_ALL`, and `HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES`.
* `flags`: a `unsigned long` representing any flags to apply to the added distance, such as `HWLOC_DISTANCES_ADD_FLAG_GROUP` or `HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE`.

The function returns a pointer to the new distance, if the addition is successful. If any errors occur, the function returns `NULL`.


```cpp
/********************************
 * User API for adding distances
 */

#define HWLOC_DISTANCES_KIND_FROM_ALL (HWLOC_DISTANCES_KIND_FROM_OS|HWLOC_DISTANCES_KIND_FROM_USER)
#define HWLOC_DISTANCES_KIND_MEANS_ALL (HWLOC_DISTANCES_KIND_MEANS_LATENCY|HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH)
#define HWLOC_DISTANCES_KIND_ALL (HWLOC_DISTANCES_KIND_FROM_ALL|HWLOC_DISTANCES_KIND_MEANS_ALL|HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES)
#define HWLOC_DISTANCES_ADD_FLAG_ALL (HWLOC_DISTANCES_ADD_FLAG_GROUP|HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE)

void * hwloc_distances_add_create(hwloc_topology_t topology,
                                  const char *name, unsigned long kind,
                                  unsigned long flags)
{
  if (!topology->is_loaded) {
    errno = EINVAL;
    return NULL;
  }
  if (topology->adopted_shmem_addr) {
    errno = EPERM;
    return NULL;
  }
  if ((kind & ~HWLOC_DISTANCES_KIND_ALL)
      || hwloc_weight_long(kind & HWLOC_DISTANCES_KIND_FROM_ALL) != 1
      || hwloc_weight_long(kind & HWLOC_DISTANCES_KIND_MEANS_ALL) != 1) {
    errno = EINVAL;
    return NULL;
  }

  return hwloc_backend_distances_add_create(topology, name, kind, flags);
}

```

这段代码的作用是向 HWLOC 对象树中添加距离，并返回成功或失败的结果。

具体来说，代码首先检查输入对象是否已存在，如果不存在，则错误并跳过。然后，代码将输入对象复制到内存中，并为每个对象分配一个 HWLOC 距离值。接下来，代码调用 hwloc_backend_distances_add_values 函数，将距离值添加到树中。如果该函数成功，则返回 0，否则代码将处理错误并返回 -1。

另外，如果 handle 对象已被取消，则该函数将不再调用 hwloc_backend_distances_add_values 函数，而是直接返回 -1。


```cpp
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

  /* no strict need to check for duplicates, things shouldn't break */

  for(i=1; i<nbobjs; i++)
    if (!objs[i]) {
      errno = EINVAL;
      goto out;
    }

  /* copy the input arrays and give them to the topology */
  _objs = malloc(nbobjs*sizeof(hwloc_obj_t));
  _values = malloc(nbobjs*nbobjs*sizeof(*_values));
  if (!_objs || !_values)
    goto out_with_arrays;

  memcpy(_objs, objs, nbobjs*sizeof(hwloc_obj_t));
  memcpy(_values, values, nbobjs*nbobjs*sizeof(*_values));

  err = hwloc_backend_distances_add_values(topology, handle, nbobjs, _objs, _values, flags);
  if (err < 0) {
    /* handle was canceled inside hwloc_backend_distances_add_values */
    handle = NULL;
    goto out_with_arrays;
  }

  return 0;

 out_with_arrays:
  free(_objs);
  free(_values);
 out:
  if (handle)
    hwloc_backend_distances_add__cancel(handle);
  return -1;
}

```

这段代码是一个名为 `hwloc_distances_add_commit` 的函数，属于 `hwloc_backend_distances_add` 函数家族。它的作用是接受一个 `hwloc_topology_t` 类型的数据结构（即拓扑结构）和一个 `void *handle` 类型的参数，并返回一个整数表示执行结果。

具体来说，这段代码的作用如下：

1. 如果传递给它的 `flags` 掩码中不包含 `HWLOC_DISTANCES_ADD_FLAG_ALL`，那么它将调用 `hwloc_backend_distances_add_commit` 函数，这个函数的实现见之前的回答。如果包含这个标志，那么它将直接执行 `hwloc_backend_distances_add_commit` 函数。

2. 如果 `flags` 中包含 `HWLOC_DISTANCES_ADD_FLAG_GROUP_RECONNECT`，那么它将在执行 `hwloc_backend_distances_add_commit` 函数之后调用 `hwloc_topology_reconnect` 函数。这个函数的作用是重新连接拓扑结构。

3. 如果 `flags` 中包含 `HWLOC_DISTANCES_ADD_FLAG_GROUP_CLEAR_RECONNECT`，那么它将取消之前设置的 `hwloc_backend_distances_add_commit` 函数返回的 `hwloc_backend_distances_add__c` 函数的执行。

4. 如果传递给它的 `handle` 参数是一个 `void *handle` 类型的参数，那么它将被传递给 `hwloc_backend_distances_add_commit` 函数，并且 `hwloc_topology_reconnect` 函数将被调用。如果 `handle` 是 `void *handle` 类型的指针，那么它将被传递给 `hwloc_backend_distances_add_commit` 函数，并且 `hwloc_topology_reconnect` 函数将被多次调用。


```cpp
int
hwloc_distances_add_commit(hwloc_topology_t topology,
                           void *handle,
                           unsigned long flags)
{
  int err;

  if (flags & ~HWLOC_DISTANCES_ADD_FLAG_ALL) {
    errno = EINVAL;
    goto out;
  }

  err = hwloc_backend_distances_add_commit(topology, handle, flags);
  if (err < 0) {
    /* handle was canceled inside hwloc_backend_distances_add_commit */
    handle = NULL;
    goto out;
  }

  /* in case we added some groups, see if we need to reconnect */
  hwloc_topology_reconnect(topology, 0);

  return 0;

 out:
  if (handle)
    hwloc_backend_distances_add__cancel(handle);
  return -1;
}

```

这段代码是一个名为 `hwloc_distances_add` 的函数，它是 `hwloc_topology_t` 结构体的一部分。函数的目的是在 `hwloc_topology_t` 的顶类中增加距离度量。

具体来说，这个函数接受四个参数：

1. `topology`：顶类的结构体，这里可能包含一些对象和距离度量。
2. `nbobjs`：要计算距离度的物体数量。
3. `objs`：距离度量的对象数组，每个物体都有一个距离度和一个索引。
4. `values`：距离度量的值数组，每个值都有一个时间戳和一个距离度量。
5. `kind`：允许计算的距离度量类型。可能是 `hwloc_distance_kind_bits` 中的一个。
6. `flags`：附加的标志，可能是用于设置或清除某些计算设置的。

函数首先创建一个名为 `hwloc_distances_add_create` 的函数指针，然后在 `topology` 对象上调用它。这个新创建的函数指针可能是一个函数，它接受 `topology` 和 `null` 参数，然后执行距离度量计算操作。

接下来，函数调用 `hwloc_distances_add_values` 函数，并将 `topology`、`nbobjs`、`values` 和 `kind` 作为参数传入。这个函数将计算距离度量，并将计算结果存储在 `values` 数组中。

最后，函数调用 `hwloc_distances_add_commit` 函数，并将 `topology` 和 `handle` 作为参数传入。这个函数将清除之前创建的距离度量，以便在之后的计算中使用。

如果以上所有操作成功完成，函数将返回 0，否则返回 -1。


```cpp
/* deprecated all-in-one user function */
int hwloc_distances_add(hwloc_topology_t topology,
			unsigned nbobjs, hwloc_obj_t *objs, hwloc_uint64_t *values,
			unsigned long kind, unsigned long flags)
{
  void *handle;
  int err;

  handle = hwloc_distances_add_create(topology, NULL, kind, 0);
  if (!handle)
    return -1;

  err = hwloc_distances_add_values(topology, handle, nbobjs, objs, values, 0);
  if (err < 0)
    return -1;

  err = hwloc_distances_add_commit(topology, handle, flags);
  if (err < 0)
    return -1;

  return 0;
}

```



这段代码定义了一个名为 `hwloc_internal_distances_restrict` 的函数，它的作用是刷新距离计算中对象的索引和值。

距离计算中，每个对象都有一个对应的索引值和距离值。距离计算结束后，这些对象将被重新存回内存，并且索引值和值也将被刷新。

在这段代码中， `hwloc_internal_distances_restrict` 函数接受三个参数：一个整数类型的对象数组 `objs`，一个整数类型的索引数组 `indexes`，一个不同类型的数组 `different_types`，以及一个表示对象数量和消失数量的整数 `nbobjs` 和一个表示对象数量的变化量 `disappeared`。

函数的主要部分是一个循环，它在每次循环中依次检查每个对象是否有效，并对其进行修改。具体来说，对于每个对象，函数首先检查它是否在索引数组中，如果是，则执行相应的操作。然后，函数检查该对象是否在 `different_types` 数组中，如果是，则将 `different_types` 数中的值替换为该对象的值。接下来，函数计算并更新对象索引数组中对应对象的索引值和值，并将对象重新存储回内存。最后，函数检查 `nbobjs` 和 `disappeared` 是否发生了变化，如果是，则对函数进行相应的调整。

由于具体的实现细节并不清楚，因此无法提供更多有关此代码的上下文信息。


```cpp
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
	indexes[newi] = indexes[i];
      if (different_types)
        different_types[newi] = different_types[i];
      newi++;
    }
}

```

This function appears to validate the distribution of objectives in the茂盛树状结构中。验证内容包括：检查对象的类型、获取对象索引和计算对象数量。如果对象索引存在，验证它们是否有效并更新对象数量。如果对象数量小于对象索引数量的两倍，说明对象分布无效，删除这些无效的叶子节点。在分布式分布中，对象索引是不同的，所以需要根据不同的索引类型使用不同的查找方式。最后，如果本地化索引不正确，使用内部距离限制来修复它。修复后，将通知对象索引和对象数量。


```cpp
static int
hwloc_internal_distances_refresh_one(hwloc_topology_t topology,
				     struct hwloc_internal_distances_s *dist)
{
  hwloc_obj_type_t unique_type = dist->unique_type;
  hwloc_obj_type_t *different_types = dist->different_types;
  unsigned nbobjs = dist->nbobjs;
  hwloc_obj_t *objs = dist->objs;
  uint64_t *indexes = dist->indexes;
  unsigned disappeared = 0;
  unsigned i;

  if (dist->iflags & HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID)
    return 0;

  for(i=0; i<nbobjs; i++) {
    hwloc_obj_t obj;
    /* TODO use cpuset/nodeset to find pus/numas from the root?
     * faster than traversing the entire level?
     */
    if (HWLOC_DIST_TYPE_USE_OS_INDEX(unique_type)) {
      if (unique_type == HWLOC_OBJ_PU)
	obj = hwloc_get_pu_obj_by_os_index(topology, (unsigned) indexes[i]);
      else if (unique_type == HWLOC_OBJ_NUMANODE)
	obj = hwloc_get_numanode_obj_by_os_index(topology, (unsigned) indexes[i]);
      else
	abort();
    } else {
      obj = hwloc_get_obj_by_type_and_gp_index(topology, different_types ? different_types[i] : unique_type, indexes[i]);
    }
    objs[i] = obj;
    if (!obj)
      disappeared++;
  }

  if (nbobjs-disappeared < 2)
    /* became useless, drop */
    return -1;

  if (disappeared) {
    hwloc_internal_distances_restrict(objs, dist->indexes, dist->different_types, dist->values, nbobjs, disappeared);
    dist->nbobjs -= disappeared;
  }

  dist->iflags |= HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID;
  return 0;
}

```



这段代码是一个名为 `hwloc_internal_distances_refresh` 的函数，它是 `hwloc_topology_t` 结构体的内部函数。

它的作用是刷新 topology 中的内部距离，确保所有距离都有效，即使 topology 中的某些元素无法完成分配。如果需要重新分配距离，函数将尝试通过调用 `hwloc_internal_distances_refresh_one` 来完成分配，如果没有成功，则需要手动释放距离。

对于每个距离节点，函数将检查其后续是否需要更新，如果是，则更新后续节点的距离。如果不需要更新，则更新 topology 中的第一个距离节点，并将 topology 中的第一个距离节点设置为需要更新的距离节点的后续。

如果需要重新分配距离，函数将尝试通过调用 `hwloc_internal_distances_refresh_one` 来完成分配，如果没有成功，则需要手动释放距离。如果重新分配成功，则更新后续节点的距离，并确保所有节点都有对应的后续节点。最后，函数将释放每个距离节点，并继续遍历剩余的节点。


```cpp
/* This function may be called with topology->tma set, it cannot free() or realloc() */
void
hwloc_internal_distances_refresh(hwloc_topology_t topology)
{
  struct hwloc_internal_distances_s *dist, *next;

  for(dist = topology->first_dist; dist; dist = next) {
    next = dist->next;

    if (hwloc_internal_distances_refresh_one(topology, dist) < 0) {
      assert(!topology->tma || !topology->tma->dontfree); /* this tma cannot fail to allocate */
      if (dist->prev)
	dist->prev->next = next;
      else
	topology->first_dist = next;
      if (next)
	next->prev = dist->prev;
      else
	topology->last_dist = dist->prev;
      hwloc_internal_distances_free(dist);
      continue;
    }
  }
}

```

这段代码定义了一个名为 `hwloc_internal_distances_invalidate_cached_objs` 的函数，属于 `hwloc_topology_ex` 类的成员函数。

它的作用是清除内部距离缓存，将缓存中的所有距离置为无效。这个缓存是用于在 `hwloc_topology_t` 结构中查找距离的，通过遍历 `topology` 的 `first_dist` 和 `next` 成员，可以访问到缓存中的所有距离。

函数的参数是一个 `hwloc_topology_t` 类型的整数，表示要操作的拓扑结构。返回类型是一个 `void` 类型的整数，表示函数执行成功。


```cpp
void
hwloc_internal_distances_invalidate_cached_objs(hwloc_topology_t topology)
{
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
```

这段代码定义了一个名为 `hwloc_distances_container_s` 的结构体，该结构体包含一个 `id` 和一个 `distances` 成员。

接下来，定义了两个宏：`HWLOC_DISTANCES_CONTAINER_OFFSET` 和 `HWLOC_DISTANCES_CONTAINER`。`HWLOC_DISTANCES_CONTAINER_OFFSET` 将 `(uintptr_t)(&((struct hwloc_distances_container_s*)NULL)->distances) - (uintptr_t)NULL` 计算得到一个指向 `distances` 所指结构体内部位置的指针。`HWLOC_DISTANCES_CONTAINER` 将 `((char*)_d) - HWLOC_DISTANCES_CONTAINER_OFFSET` 计算得到一个指向 `distances` 所指结构体内部位置的指针，并将其存储为 `dist` 的指针。

最后，定义了一个名为 `hwloc__internal_distances_from_public` 的函数，该函数接收一个 `topology` 结构体和一个 `distances` 结构体作为参数。首先，将 `distances` 所指的结构体指向的内存位置减去 `HWLOC_DISTANCES_CONTAINER_OFFSET`，得到一个指向 `distances` 所指结构体内部位置的指针。然后，将该指针存储为 `dist` 的指针，并返回 `dist`。


```cpp
struct hwloc_distances_container_s {
  unsigned id;
  struct hwloc_distances_s distances;
};

#define HWLOC_DISTANCES_CONTAINER_OFFSET ((uintptr_t)(&((struct hwloc_distances_container_s*)NULL)->distances) - (uintptr_t)NULL)
#define HWLOC_DISTANCES_CONTAINER(_d) (struct hwloc_distances_container_s *) ( ((char*)_d) - HWLOC_DISTANCES_CONTAINER_OFFSET )

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

```



这两段代码是针对hwloc库中的`hwloc_distances_get_name`函数和`hwloc_distances_release`函数。它们的主要作用是获取和释放hwloc库中的距离信息。

1. `hwloc_distances_get_name`函数接收一个hwloc_topology结构和一个hwloc_distances结构作为参数。它首先从topology结构中获取内部距离信息，然后通过调用`hwloc__internal_distances_from_public`函数将内部距离信息转换为public结构，最后返回该public结构的名字。

2. `hwloc_distances_release`函数同样接收一个hwloc_topology结构和一个hwloc_distances结构作为参数。它首先获取到distances结构中的值和对象，然后释放这些资源，最后将内部距离信息（如果有）返回给hwloc库的`hwloc_distances_get_name`函数。


```cpp
void
hwloc_distances_release(hwloc_topology_t topology __hwloc_attribute_unused,
			struct hwloc_distances_s *distances)
{
  struct hwloc_distances_container_s *cont = HWLOC_DISTANCES_CONTAINER(distances);
  free(distances->values);
  free(distances->objs);
  free(cont);
}

const char *
hwloc_distances_get_name(hwloc_topology_t topology, struct hwloc_distances_s *distances)
{
  struct hwloc_internal_distances_s *dist = hwloc__internal_distances_from_public(topology, distances);
  return dist ? dist->name : NULL;
}

```

该函数的作用是获取 topology 中的一个 hwloc_distances 结构体，其内部距离 topology 的距离。该函数需要从给定的 topology 和距离中提取相关信息，并返回一个指向该结构体的指针。

函数参数：

- topology：topology 是一个 hwloc_topology 类型的参数，用于指定要获取的距离的 topology。
- dist：一个 hwloc_internal_distances_s 类型的参数，用于指定要获取的距离。

函数返回：

- hwloc_distances_s：一个指向 hwloc_distances 结构体的指针，该结构体包含从 topology 和距离中提取的信息。

函数实现了一个 hwloc_distances 结构体，用于存储 topology 中的距离信息。该函数通过调用 malloc 和 free 函数来管理内存。具体来说，函数首先从 topology 和 distance 中提取相关信息，然后从 topology 和 distance 中获取该结构体所需的内存，并将其返回。


```cpp
static struct hwloc_distances_s *
hwloc_distances_get_one(hwloc_topology_t topology __hwloc_attribute_unused,
			struct hwloc_internal_distances_s *dist)
{
  struct hwloc_distances_container_s *cont;
  struct hwloc_distances_s *distances;
  unsigned nbobjs;

  cont = malloc(sizeof(*cont));
  if (!cont)
    return NULL;
  distances = &cont->distances;

  nbobjs = distances->nbobjs = dist->nbobjs;

  distances->objs = malloc(nbobjs * sizeof(hwloc_obj_t));
  if (!distances->objs)
    goto out;
  memcpy(distances->objs, dist->objs, nbobjs * sizeof(hwloc_obj_t));

  distances->values = malloc(nbobjs * nbobjs * sizeof(*distances->values));
  if (!distances->values)
    goto out_with_objs;
  memcpy(distances->values, dist->values, nbobjs*nbobjs*sizeof(*distances->values));

  distances->kind = dist->kind;

  cont->id = dist->id;
  return distances;

 out_with_objs:
  free(distances->objs);
 out:
  free(cont);
  return NULL;
}

```

This function appears to be a part of a software framework that enables the creation of custom hierarchies for HWLOC (Hardware Location Infrastructure) devices. It is used to retrieve the distances between HWLOC devices and the topology.

The function takes two arguments: `flags` and `name`. The `flags` argument is a bit field indicating whether the function should retrieve distances for all distances, or just for the distances that match a specified flag (defaulting to EINVAL). The `name` argument is a string that is compared to the name of the HWLOC device to retrieve.

The function first checks if the `flags` argument is set. If it is not set, the function returns EINVAL.

Next, the function refreshes the distances between all HWLOC devices that match the specified flag and updates the `distancesp` array with the distances.

Finally, the function iterates through the `distancesp` array and releases any distances that match the specified flag. It also increments the `nr` counter.

Note that the function only releases the distances that match the specified flag. If the `name` argument is equal to the `flags` argument, the function retrieves the distances for all distances and does not release any of them.


```cpp
static int
hwloc__distances_get(hwloc_topology_t topology,
		     const char *name, hwloc_obj_type_t type,
		     unsigned *nrp, struct hwloc_distances_s **distancesp,
		     unsigned long kind, unsigned long flags __hwloc_attribute_unused)
{
  struct hwloc_internal_distances_s *dist;
  unsigned nr = 0, i;

  /* We could return the internal arrays (as const),
   * but it would require to prevent removing distances between get() and free().
   * Not performance critical anyway.
   */

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
  hwloc_internal_distances_refresh(topology);

  for(dist = topology->first_dist; dist; dist = dist->next) {
    unsigned long kind_from = kind & HWLOC_DISTANCES_KIND_FROM_ALL;
    unsigned long kind_means = kind & HWLOC_DISTANCES_KIND_MEANS_ALL;

    if (name && (!dist->name || strcmp(name, dist->name)))
      continue;

    if (type != HWLOC_OBJ_TYPE_NONE && type != dist->unique_type)
      continue;

    if (kind_from && !(kind_from & dist->kind))
      continue;
    if (kind_means && !(kind_means & dist->kind))
      continue;

    if (nr < *nrp) {
      struct hwloc_distances_s *distances = hwloc_distances_get_one(topology, dist);
      if (!distances)
	goto error;
      distancesp[nr] = distances;
    }
    nr++;
  }

  for(i=nr; i<*nrp; i++)
    distancesp[i] = NULL;
  *nrp = nr;
  return 0;

 error:
  for(i=0; i<nr; i++)
    hwloc_distances_release(topology, distancesp[i]);
  return -1;
}

```



这两段代码是用于计算 hwloc_distances_s 结构体中距离的函数。其中，hwloc_topology_t 表示输入的 topology 结构体指针，nrp 表示指向 Distance 结构体的指针，distancesp 指向指向 hwloc_distances_s 结构体的指针，kind 表示要计算的距离类型，flags 表示传递给函数的标志。 

hwloc__distances_get 函数的实现比较复杂，但可以归纳为以下几个步骤：

1. 检查传入的参数，如果 flags 或 topology_is_loaded 为真，则认为函数内部有错误，返回 -1。

2. 如果 flags 为真，且 topology_is_loaded 为 false，则直接返回 -1，因为在这种情况下函数不会执行。

3. 如果 flags 为 false,topology_is_loaded 为 true，则调用 hwloc__distances_get 函数，传入 topology 结构体和 NULL，计算得到距离后，将结果传入回 hwloc__distances_get 函数。这里计算得到的距离类型为 topology 的 depth 类型的距离类型，如果 topology 的深度类型不存在，则返回 -1。

4. 在 hwloc__distances_get 函数中，首先检查 flags 是否为真，如果是，则调用 hwloc__distances_get 函数，如果 already 指向前一个 hwloc__distances_get 函数计算得到的距离，则直接返回前一个 hwloc__distances_get 函数的结果，否则计算新的距离，并将结果传入回 hwloc__distances_get 函数。

5. 在 hwloc__distances_get 函数中，根据 topology 的 depth 类型，调用不同的函数计算距离，如果 depth 不存在，则默认深度为 0，计算距离为 topology 本身。


```cpp
int
hwloc_distances_get(hwloc_topology_t topology,
		    unsigned *nrp, struct hwloc_distances_s **distancesp,
		    unsigned long kind, unsigned long flags)
{
  if (flags || !topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  return hwloc__distances_get(topology, NULL, HWLOC_OBJ_TYPE_NONE, nrp, distancesp, kind, flags);
}

int
hwloc_distances_get_by_depth(hwloc_topology_t topology, int depth,
			     unsigned *nrp, struct hwloc_distances_s **distancesp,
			     unsigned long kind, unsigned long flags)
{
  hwloc_obj_type_t type;

  if (flags || !topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  /* FIXME: passing the depth of a group level may return group distances at a different depth */
  type = hwloc_get_depth_type(topology, depth);
  if (type == (hwloc_obj_type_t)-1) {
    errno = EINVAL;
    return -1;
  }

  return hwloc__distances_get(topology, NULL, type, nrp, distancesp, kind, flags);
}

```

这段代码是用来从给定的topology链中获取指定name的点到目标对象的距离，并返回该距离。具体的实现过程如下：

1. 首先检查传入的topology链是否已经加载，如果是，则直接调用`hwloc__distances_get()`函数获取距离。

2. 如果topology链没有加载或者传入的name不是有效的name，则返回一个errno。

3. 对于不同类型的目标对象(如hwloc_obj_type_和hwloc_obj_type_)，需要先确定nrp参数中包含哪些距离类型，然后分别调用`hwloc__distances_get()`函数获取相应的距离。

4. 在函数内部，如果传入的topology链没有被加载，则需要执行errno为EINVAL的错误处理，并返回-1。


```cpp
int
hwloc_distances_get_by_name(hwloc_topology_t topology, const char *name,
			    unsigned *nrp, struct hwloc_distances_s **distancesp,
			    unsigned long flags)
{
  if (flags || !topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  return hwloc__distances_get(topology, name, HWLOC_OBJ_TYPE_NONE, nrp, distancesp, HWLOC_DISTANCES_KIND_ALL, flags);
}

int
hwloc_distances_get_by_type(hwloc_topology_t topology, hwloc_obj_type_t type,
			    unsigned *nrp, struct hwloc_distances_s **distancesp,
			    unsigned long kind, unsigned long flags)
{
  if (flags || !topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  return hwloc__distances_get(topology, NULL, type, nrp, distancesp, kind, flags);
}

```

这段代码定义了一个名为 `hwloc_compare_values` 的函数，用于将一组对象按照它们之间的距离分成不同的组。分组的依据是距离，可以通过设置精度（default value is 0.0f）来设置分组的距离精度。

函数接受两个参数：一个表示距离的浮点数，另一个是表示最小距离的浮点数。函数首先检查两个距离是否相等，如果相等，则返回 0，否则继续比较距离，如果距离较小的对象在距离较大的对象左边，则返回 -1，否则返回 1。

在函数内部，使用 `fabsf` 函数计算两个距离之间的差值，然后与给定的精度比较。如果差值小于精度与距离的乘积，则说明两个距离在同一个组内，返回 0。如果两个距离不在同一个组内，返回差的绝对值，表示有两个不同的距离图。


```cpp
/******************************************************
 * Grouping objects according to distances
 */

static int hwloc_compare_values(uint64_t a, uint64_t b, float accuracy)
{
  if (accuracy != 0.0f && fabsf((float)a-(float)b) < (float)a * accuracy)
    return 0;
  return a < b ? -1 : a == b ? 0 : 1;
}

/*
 * Place objects in groups if they are in a transitive graph of minimal values.
 * Return how many groups were created, or 0 if some incomplete distance graphs were found.
 */
```

This is a C function that takes a transitive graph represented as a set of group IDs and a list of group IDs and returns the last ID in the graph, which is also the number of used group IDs.

The function first initializes the graph with a single object and a maximum size for the graph. It then initializes the first ID as the first found object in the graph and the first connection as the first connection point for the first object.

The function then enters a while loop that reads the graph from the input, rescanning all connections from each new object to any other objects to find the minimal connected graph. This is done by scanning the connections from the new object to all other objects, starting from the first connection point of the new object, and keeping track of the minimal connected graph found.

The function then updates the first connection point of the first object and rescan the graph until the while loop is exited or a new object is found.

If a new object is found, the function updates the first connection point and rescan the graph until the while loop is exited or the maximum size of the graph is reached.

Finally, the function returns the last ID in the graph, which is also the number of used group IDs. If the input graph contains only one object, the function returns 0, as it does not make sense to count the ID of a single object.


```cpp
static unsigned
hwloc__find_groups_by_min_distance(unsigned nbobjs,
				   uint64_t *_values,
				   float accuracy,
				   unsigned *groupids,
				   int verbose)
{
  uint64_t min_distance = UINT64_MAX;
  unsigned groupid = 1;
  unsigned i,j,k;
  unsigned skipped = 0;

#define VALUE(i, j) _values[(i) * nbobjs + (j)]

  memset(groupids, 0, nbobjs*sizeof(*groupids));

  /* find the minimal distance */
  for(i=0; i<nbobjs; i++)
    for(j=0; j<nbobjs; j++) /* check the entire matrix, it may not be perfectly symmetric depending on the accuracy */
      if (i != j && VALUE(i, j) < min_distance) /* no accuracy here, we want the real minimal */
        min_distance = VALUE(i, j);
  hwloc_debug("  found minimal distance %llu between objects\n", (unsigned long long) min_distance);

  if (min_distance == UINT64_MAX)
    return 0;

  /* build groups of objects connected with this distance */
  for(i=0; i<nbobjs; i++) {
    unsigned size;
    unsigned firstfound;

    /* if already grouped, skip */
    if (groupids[i])
      continue;

    /* start a new group */
    groupids[i] = groupid;
    size = 1;
    firstfound = i;

    while (firstfound != (unsigned)-1) {
      /* we added new objects to the group, the first one was firstfound.
       * rescan all connections from these new objects (starting at first found) to any other objects,
       * so as to find new objects minimally-connected by transivity.
       */
      unsigned newfirstfound = (unsigned)-1;
      for(j=firstfound; j<nbobjs; j++)
	if (groupids[j] == groupid)
	  for(k=0; k<nbobjs; k++)
              if (!groupids[k] && !hwloc_compare_values(VALUE(j, k), min_distance, accuracy)) {
	      groupids[k] = groupid;
	      size++;
	      if (newfirstfound == (unsigned)-1)
		newfirstfound = k;
	      if (i == j)
		hwloc_debug("  object %u is minimally connected to %u\n", k, i);
	      else
	        hwloc_debug("  object %u is minimally connected to %u through %u\n", k, i, j);
	    }
      firstfound = newfirstfound;
    }

    if (size == 1) {
      /* cancel this useless group, ignore this object and try from the next one */
      groupids[i] = 0;
      skipped++;
      continue;
    }

    /* valid this group */
    groupid++;
    if (verbose)
      fprintf(stderr, " Found transitive graph with %u objects with minimal distance %llu accuracy %f\n",
	      size, (unsigned long long) min_distance, accuracy);
  }

  if (groupid == 2 && !skipped)
    /* we created a single group containing all objects, ignore it */
    return 0;

  /* return the last id, since it's also the number of used group ids */
  return groupid-1;
}

```

该函数的作用是检查给定的多维离散化矩阵是否满足某些最小二乘法（或者最小距离）约束，并返回一个逻辑成功或失败的结果。具体来说，该函数接受一个整数类型的变量nbobjs，一个长度为nbobjs的整数类型的数组_values，一个浮点数类型的变量accuracy和一个布尔类型的变量verbose，用于指定是否输出详细信息。

在函数内部，首先使用for循环遍历所有的对角线及其下标的元素，然后使用hwloc_compare_values函数比较当前元素与预设的最小二乘解之间的距离，如果当前元素与对角线元素之间的距离小于预设的最小距离，则说明矩阵存在，函数返回0表示成功。如果当前元素与对角线元素之间的距离大于预设的最小距离，或者矩阵存在对称性，函数将使用类似于printf函数的格式化字符串输出错误信息并返回-1，其中错误信息将包含矩阵的行列号和元素值。


```cpp
/* check that the matrix is ok */
static int
hwloc__check_grouping_matrix(unsigned nbobjs, uint64_t *_values, float accuracy, int verbose)
{
  unsigned i,j;
  for(i=0; i<nbobjs; i++) {
    for(j=i+1; j<nbobjs; j++) {
      /* should be symmetric */
      if (hwloc_compare_values(VALUE(i, j), VALUE(j, i), accuracy)) {
	if (verbose)
	  fprintf(stderr, " Distance matrix asymmetric ([%u,%u]=%llu != [%u,%u]=%llu), aborting\n",
		  i, j, (unsigned long long) VALUE(i, j), j, i, (unsigned long long) VALUE(j, i));
	return -1;
      }
      /* diagonal is smaller than everything else */
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

```

It looks like you have a Go program that reads in a list of group identifiers and group topologies, and then attempts to group the objects based on the group identifier. The program uses the `hwloc` library to perform the grouping, and in particular, it uses the `hwloc_bitmap_alloc` and `hwloc_obj_add_other_obj_sets` functions to manage the grouping process.

The program first initializes a few variables at the top: `groupsizes`, `groupids`, `groupobjs`, and `res_obj`. It then loops through each group identifier，组装出该组的所有物体，并更新 `groupsizes` 中每个组的大小。接下来，它x时分段，并在每一段中循环链表中的物体。如果当前物体属于已经组装好的组，则将属性和关系存储在该组中。最后，如果组装过程失败，则跳转到 `out_with_groups` 标签，但此时程序已经以某种程度上成功组成了物体。


```cpp
/*
 * Look at object physical distances to group them.
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
    /* don't know use to use those for grouping */
    /* TODO hwloc__find_groups_by_max_distance() for bandwidth */
    return;

  groupids = malloc(nbobjs * sizeof(*groupids));
  if (!groupids)
    return;

  for(i=0; i<nbaccuracies; i++) {
    if (verbose)
      fprintf(stderr, "Trying to group %u %s objects according to physical distances with accuracy %f\n",
	      nbobjs, hwloc_obj_type_string(objs[0]->type), accuracies[i]);
    if (needcheck && hwloc__check_grouping_matrix(nbobjs, _values, accuracies[i], verbose) < 0)
      continue;
    nbgroups = hwloc__find_groups_by_min_distance(nbobjs, _values, accuracies[i], groupids, verbose);
    if (nbgroups)
      break;
  }
  if (!nbgroups)
    goto out_with_groupids;

  groupobjs = malloc(nbgroups * sizeof(*groupobjs));
  groupsizes = malloc(nbgroups * sizeof(*groupsizes));
  groupvalues = malloc(nbgroups * nbgroups * sizeof(*groupvalues));
  if (!groupobjs || !groupsizes || !groupvalues)
    goto out_with_groups;

      /* create new Group objects and record their size */
      memset(&(groupsizes[0]), 0, sizeof(groupsizes[0]) * nbgroups);
      for(i=0; i<nbgroups; i++) {
          /* create the Group object */
          hwloc_obj_t group_obj, res_obj;
          group_obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, HWLOC_UNKNOWN_INDEX);
          group_obj->cpuset = hwloc_bitmap_alloc();
          group_obj->attr->group.kind = HWLOC_GROUP_KIND_DISTANCE;
          group_obj->attr->group.subkind = topology->grouping_next_subkind;
          for (j=0; j<nbobjs; j++)
	    if (groupids[j] == i+1) {
	      /* assemble the group sets */
	      hwloc_obj_add_other_obj_sets(group_obj, objs[j]);
              groupsizes[i]++;
            }
          hwloc_debug_1arg_bitmap("adding Group object with %u objects and cpuset %s\n",
                                  groupsizes[i], group_obj->cpuset);
          res_obj = hwloc__insert_object_by_cpuset(topology, NULL, group_obj,
                                                   (kind & HWLOC_DISTANCES_KIND_FROM_USER) ? "distances:fromuser:group" : "distances:group");
	  /* res_obj may be NULL on failure to insert. */
	  if (!res_obj)
	    failed++;
	  /* or it may be different from groupobjs if we got groups from XML import before grouping */
          groupobjs[i] = res_obj;
      }
      topology->grouping_next_subkind++;

      if (failed)
	/* don't try to group above if we got a NULL group here, just keep this incomplete level */
	goto out_with_groups;

      /* factorize values */
      memset(&(groupvalues[0]), 0, sizeof(groupvalues[0]) * nbgroups * nbgroups);
```

这段代码定义了一系列宏和变量，主要用途是生成一个表示不同组之间距离的矩阵。

宏定义中，VALUE()函数定义了一个新的变量i和j，然后从_values数组中根据i和j的值复制一个值到VALUE变量中。GROUP_VALUE()函数定义了一个新的变量i和j，然后从groupvalues数组中根据i和j的值复制一个值到GROUP_VALUE变量中。

循环结构中，使用两个嵌套的for循环，遍历objs和groups数组。在内层循环中，检查当前组中是否有一个或多个组，如果是，则执行GROUP_VALUE()函数，将i和j的值相加，并将结果相加到GROUP_VALUE变量中。在外层循环中，使用hwloc_debug()函数打印生成的距离矩阵，并使用GROUP_VALUE()函数打印每个组的值。

最后，该代码还定义了一个名为HWLOC_DEBUG的变量，用于输出调试信息。


```cpp
#undef VALUE
#define VALUE(i, j) _values[(i) * nbobjs + (j)]
#define GROUP_VALUE(i, j) groupvalues[(i) * nbgroups + (j)]
      for(i=0; i<nbobjs; i++)
	if (groupids[i])
	  for(j=0; j<nbobjs; j++)
	    if (groupids[j])
                GROUP_VALUE(groupids[i]-1, groupids[j]-1) += VALUE(i, j);
      for(i=0; i<nbgroups; i++)
          for(j=0; j<nbgroups; j++) {
              unsigned groupsize = groupsizes[i]*groupsizes[j];
              GROUP_VALUE(i, j) /= groupsize;
          }
#ifdef HWLOC_DEBUG
      hwloc_debug("%s", "generated new distance matrix between groups:\n");
      hwloc_debug("%s", "  index");
      for(j=0; j<nbgroups; j++)
	hwloc_debug(" % 5d", (int) j); /* print index because os_index is -1 for Groups */
      hwloc_debug("%s", "\n");
      for(i=0; i<nbgroups; i++) {
	hwloc_debug("  % 5d", (int) i);
	for(j=0; j<nbgroups; j++)
	  hwloc_debug(" %llu", (unsigned long long) GROUP_VALUE(i, j));
	hwloc_debug("%s", "\n");
      }
```

This function appears to be a part of the "hwloc\_distances" library, which appears to be a library for managing distances in hierarchical和世界树 data structures.

The function takes a single parameter of type hwloc\_distances\_s, which appears to be an instance of the struct type that contains the input parameters for a distance calculation.

The function appears to check for null values and update the hwloc\_obj\_type\_t variable accordingly. It also updates the nbobjs variable, which is the number of object values.

The function also updates the HWLOC\_DISTANCES\_KIND\_HETEROGENEOUS\_TYPES variable, which appears to be a combination of the current object type and a flag indicating whether the object is a node or a group node.


```cpp
#endif

      hwloc__groups_by_distances(topology, nbgroups, groupobjs, groupvalues, kind, nbaccuracies, accuracies, 0 /* no need to check generated matrix */);

 out_with_groups:
  free(groupobjs);
  free(groupsizes);
  free(groupvalues);
 out_with_groupids:
  free(groupids);
}

static int
hwloc__distances_transform_remove_null(struct hwloc_distances_s *distances)
{
  hwloc_uint64_t *values = distances->values;
  hwloc_obj_t *objs = distances->objs;
  unsigned i, nb, nbobjs = distances->nbobjs;
  hwloc_obj_type_t unique_type;

  for(i=0, nb=0; i<nbobjs; i++)
    if (objs[i])
      nb++;

  if (nb < 2) {
    errno = EINVAL;
    return -1;
  }

  if (nb == nbobjs)
    return 0;

  hwloc_internal_distances_restrict(objs, NULL, NULL, values, nbobjs, nbobjs-nb);
  distances->nbobjs = nb;

  /* update HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES for convenience */
  unique_type = objs[0]->type;
  for(i=1; i<nb; i++)
    if (objs[i]->type != unique_type) {
      unique_type = HWLOC_OBJ_TYPE_NONE;
      break;
    }
  if (unique_type == HWLOC_OBJ_TYPE_NONE)
    distances->kind |= HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES;
  else
    distances->kind &= ~HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES;

  return 0;
}

```

这段代码是一个名为 `hwloc__distances_transform_links` 的函数，属于 `hwloc_distances_s` 结构体的成员。它的作用是处理 `hwloc_distances_s` 结构体中的距离值，将其转换为距离。

具体来说，该函数的实现过程如下：

1. 检查距离值的种类是否为 `HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH`，如果不是，则函数返回一个错误码。
2. 初始化一个 `hwloc_uint64_t` 类型的变量 `divider` 和一个指向 `values` 类型的指针 `values`，同时将 `values` 初始化为 0。
3. 遍历距离值，对于每个距离值，查找其最小的正值，如果没有找到，则函数返回一个错误码。
4. 如果找到了距离值的最小正值，则将其除以 `divider`，并继续遍历。
5. 循环结束后，函数返回 0，表示处理成功。

代码中包含了一个错误检查，如果距离值不能被整除，则在尝试除以商后仍然无法整除，则会返回一个错误码。


```cpp
static int
hwloc__distances_transform_links(struct hwloc_distances_s *distances)
{
  /* FIXME: we should look for the greatest common denominator
   * but we just use the smallest positive value, that's enough for current use-cases.
   * We'll return -1 in other cases.
   */
  hwloc_uint64_t divider, *values = distances->values;
  unsigned i, nbobjs = distances->nbobjs;

  if (!(distances->kind & HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH)) {
    errno = EINVAL;
    return -1;
  }

  for(i=0; i<nbobjs; i++)
    values[i*nbobjs+i] = 0;

  /* find the smallest positive value */
  divider = 0;
  for(i=0; i<nbobjs*nbobjs; i++)
    if (values[i] && (!divider || values[i] < divider))
      divider = values[i];

  if (!divider)
    /* only zeroes? do nothing */
    return 0;

  /* check it divides all values */
  for(i=0; i<nbobjs*nbobjs; i++)
    if (values[i]%divider) {
      errno = ENOENT;
      return -1;
    }

  /* ok, now divide for real */
  for(i=0; i<nbobjs*nbobjs; i++)
    values[i] /= divider;

  return 0;
}

```

`switch_ports()` is a function that is used to switch the ports of a NVLink interface. It takes a `hwloc_topology_t` object and a `struct hwloc_distances_s` object as input arguments.

The function first checks if the port name is `NVLinkBandwidth`. If it is, the function returns an error and immediately stops execution.

If the port name is not `NVLinkBandwidth`, the function finds the first port in the `hwloc_distances_s` object and attempts to find the corresponding port in the `hwloc_topology_t` object. If the port is not found, the function returns an error and immediately stops execution.

If the port is found, the function iterates over the rest of the ports in the `hwloc_topology_t` object, checking for ports with the same name as the currently processed port. If a port with the same name is found, the function merges the values of the current port with the values of the other ports. If the port is not found, the function does not merge the values and just returns.

Finally, the function returns 0 if the process was successful and -1 if it encountered an error.


```cpp
static __hwloc_inline int is_nvswitch(hwloc_obj_t obj)
{
  return obj && obj->subtype && !strcmp(obj->subtype, "NVSwitch");
}

static int
hwloc__distances_transform_merge_switch_ports(hwloc_topology_t topology,
                                              struct hwloc_distances_s *distances)
{
  struct hwloc_internal_distances_s *dist = hwloc__internal_distances_from_public(topology, distances);
  hwloc_obj_t *objs = distances->objs;
  hwloc_uint64_t *values = distances->values;
  unsigned first, i, j, nbobjs = distances->nbobjs;

  if (strcmp(dist->name, "NVLinkBandwidth")) {
    errno = EINVAL;
    return -1;
  }

  /* find the first port */
  first = (unsigned) -1;
  for(i=0; i<nbobjs; i++)
    if (is_nvswitch(objs[i])) {
      first = i;
      break;
    }
  if (first == (unsigned)-1) {
    errno = ENOENT;
    return -1;
  }

  for(j=i+1; j<nbobjs; j++) {
    if (is_nvswitch(objs[j])) {
      /* another port, merge it */
      unsigned k;
      for(k=0; k<nbobjs; k++) {
        if (k==i || k==j)
          continue;
        values[k*nbobjs+i] += values[k*nbobjs+j];
        values[k*nbobjs+j] = 0;
        values[i*nbobjs+k] += values[j*nbobjs+k];
        values[j*nbobjs+k] = 0;
      }
      values[i*nbobjs+i] += values[j*nbobjs+j];
      values[j*nbobjs+j] = 0;
    }
    /* the caller will also call REMOVE_NULL to remove other ports */
    objs[j] = NULL;
  }

  return 0;
}

```

This function appears to be part of a library that handles NVLink bandwidth calculations. It appears to take a topology object and a array of distance objects, and returns an integer indicating whether the calculation was successful.

The function has a security warning, which indicates that the function can only be called by an app from the same app domain, and that it should not be called by an app from a different app domain or by an app running in a different sandbox.

The function first checks if the distance object being passed to it is a NVLink bandwidth distance object. If it is not, the function returns -1 and a error message is printed.

The function then iterates through the distance objects and performs some calculations on the BW values. It seems to be counting the BW values for each NVLink switch and keeping track of the total BW for each switch. It then returns 0 if the calculation was successful, or the function prints an error message if it was not successful.


```cpp
static int
hwloc__distances_transform_transitive_closure(hwloc_topology_t topology,
                                              struct hwloc_distances_s *distances)
{
  struct hwloc_internal_distances_s *dist = hwloc__internal_distances_from_public(topology, distances);
  hwloc_obj_t *objs = distances->objs;
  hwloc_uint64_t *values = distances->values;
  unsigned nbobjs = distances->nbobjs;
  unsigned i, j, k;

  if (strcmp(dist->name, "NVLinkBandwidth")) {
    errno = EINVAL;
    return -1;
  }

  for(i=0; i<nbobjs; i++) {
    hwloc_uint64_t bw_i2sw = 0;
    if (is_nvswitch(objs[i]))
      continue;
    /* count our BW to the switch */
    for(k=0; k<nbobjs; k++)
      if (is_nvswitch(objs[k]))
        bw_i2sw += values[i*nbobjs+k];

    for(j=0; j<nbobjs; j++) {
      hwloc_uint64_t bw_sw2j = 0;
      if (i == j || is_nvswitch(objs[j]))
        continue;
      /* count our BW from the switch */
      for(k=0; k<nbobjs; k++)
        if (is_nvswitch(objs[k]))
          bw_sw2j += values[k*nbobjs+j];

      /* bandwidth from i to j is now min(i2sw,sw2j) */
      values[i*nbobjs+j] = bw_i2sw > bw_sw2j ? bw_sw2j : bw_i2sw;
    }
  }

  return 0;
}

```

这段代码是一个名为 `hwloc_distances_transform` 的函数，属于 `hwloc_distances_s` 结构体类型。它的作用是执行 `hwloc_distances_transform_e` 类型的转换，并返回结果。

函数接受一个 `hwloc_topology_t` 类型的顶类和一个 `struct hwloc_distances_s` 类型的距离结构体，还有四个由用户传递的参数：`transform`、`transform_attr`、`flags` 和 `topology`。

函数内部首先检查传递的参数，如果 `transform` 为 `HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL`，则会执行 `hwloc__distances_transform_remove_null` 函数，并返回结果。如果 `transform` 为 `HWLOC_DISTANCES_TRANSFORM_LINKS`，则会执行 `hwloc__distances_transform_links` 函数，并返回结果。如果 `transform` 为 `HWLOC_DISTANCES_TRANSFORM_MERGE_SWITCH_PORTS`，则会执行以下操作：

1. 如果 `topology` 指向的顶点有多个，则执行 `hwloc__distances_transform_merge_switch_ports` 函数，并将 `distances` 中的距离与 `topology` 中的距离进行合并。
2. 如果 `transform` 为 `HWLOC_DISTANCES_TRANSFORM_TRANSITIVE_CLOSURE`，则返回 `topology` 中的距离结构体，否则会执行 `hwloc__distances_transform_transitive_closure` 函数，并返回结果。

函数的返回值是一个整数，表示转换是否成功。如果成功，则返回 0；如果失败，则返回 `EINVAL`。


```cpp
int
hwloc_distances_transform(hwloc_topology_t topology,
                          struct hwloc_distances_s *distances,
                          enum hwloc_distances_transform_e transform,
                          void *transform_attr,
                          unsigned long flags)
{
  if (flags || transform_attr) {
    errno = EINVAL;
    return -1;
  }

  switch (transform) {
  case HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL:
    return hwloc__distances_transform_remove_null(distances);
  case HWLOC_DISTANCES_TRANSFORM_LINKS:
    return hwloc__distances_transform_links(distances);
  case HWLOC_DISTANCES_TRANSFORM_MERGE_SWITCH_PORTS:
  {
    int err;
    err = hwloc__distances_transform_merge_switch_ports(topology, distances);
    if (!err)
      err = hwloc__distances_transform_remove_null(distances);
    return err;
  }
  case HWLOC_DISTANCES_TRANSFORM_TRANSITIVE_CLOSURE:
    return hwloc__distances_transform_transitive_closure(topology, distances);
  default:
    errno = EINVAL;
    return -1;
  }
}

```