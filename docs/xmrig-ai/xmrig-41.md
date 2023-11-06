# xmrig源码解析 41

# `src/3rdparty/hwloc/src/traversal.c`

这段代码是一个 C 语言程序，它定义了一个名为 "private/autogen/config.h" 的头文件，但并没有包含程序的其他部分。它包括以下几个头文件：

- "private/autogen/config.h"：定义了程序中定义的一些常量和函数。
- "hwloc.h"：定义了 "hwloc" 库，这个库可能用于管理硬件资源。
- "private/private.h"：定义了程序中定义的一些常量和函数。
- "private/misc.h"：定义了程序中定义的一些常量和函数。
- "private/debug.h"：定义了程序中定义的一些常量和函数。

另外，它还包括一个名为 "private/config.h" 的文件，但这个文件可能是由其他文件或同一源码文件组成的。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2021 Inria.  All rights reserved.
 * Copyright © 2009-2010, 2020 Université Bordeaux
 * Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/misc.h"
#include "private/debug.h"

#ifdef HAVE_STRINGS_H
```

这两行代码是用于在hwloc-topology结构中获取不同type类型的 depth和levels数组元素的函数。具体来说，`hwloc_get_type_depth`函数用于根据传入的topology结构和type类型，输出对应type类型的深度。如果type类型的值为超出了HWLOC_OBJ_TYPE_MAX，函数将返回HWLOC_TYPE_DEPTH_UNKNOWN。而`hwloc_get_depth_type`函数则用于根据传入的topology结构和深度，输出对应层级节点上的type类型。如果depth层级的值为超出了topology结构中levels数组元素的长度，函数将根据depth层级的数字编号来确定type类型。否则，函数将返回topology结构中levels数组元素的对应type类型。


```cpp
#include <strings.h>
#endif /* HAVE_STRINGS_H */

int
hwloc_get_type_depth (struct hwloc_topology *topology, hwloc_obj_type_t type)
{
  HWLOC_BUILD_ASSERT(HWLOC_OBJ_TYPE_MIN == 0);
  if ((unsigned) type >= HWLOC_OBJ_TYPE_MAX)
    return HWLOC_TYPE_DEPTH_UNKNOWN;
  else
    return topology->type_depth[type];
}

hwloc_obj_type_t
hwloc_get_depth_type (hwloc_topology_t topology, int depth)
{
  if ((unsigned)depth >= topology->nb_levels)
    switch (depth) {
    case HWLOC_TYPE_DEPTH_NUMANODE:
      return HWLOC_OBJ_NUMANODE;
    case HWLOC_TYPE_DEPTH_BRIDGE:
      return HWLOC_OBJ_BRIDGE;
    case HWLOC_TYPE_DEPTH_PCI_DEVICE:
      return HWLOC_OBJ_PCI_DEVICE;
    case HWLOC_TYPE_DEPTH_OS_DEVICE:
      return HWLOC_OBJ_OS_DEVICE;
    case HWLOC_TYPE_DEPTH_MISC:
      return HWLOC_OBJ_MISC;
    case HWLOC_TYPE_DEPTH_MEMCACHE:
      return HWLOC_OBJ_MEMCACHE;
    default:
      return HWLOC_OBJ_TYPE_NONE;
    }
  return topology->levels[depth][0]->type;
}

```

这段代码定义了一个名为 `hwloc_get_memory_parents_depth` 的函数，它接受一个名为 `topology` 的 `hwloc_topology_t` 类型的参数。

函数的主要作用是获取指定拓扑结构中所有内存节点的最大深度（也就是 NUMA 节点的层次结构中的父节点到根节点的距离）。这里使用了 HWLOC_TYPE_DEPTH_UNKNOWN 来表示未知的深度，但实际上它只在 depth 等于 0 或 topology 拓扑结构中未定义的深度时才会使用这个类型。在函数中，首先定义了一个名为 `depth` 的变量，它的初始值设置为 HWLOC_TYPE_DEPTH_UNKNOWN。

接下来，函数使用 `hwloc_get_obj_by_depth` 函数来获取 topology 拓扑结构中深度为 0 的内存节点，并将其存储在名为 `numa` 的变量中。如果获取成功，函数会使用一个 while 循环来遍历每个内存节点的父节点，并将其 depth 值存储在 `parent` 变量中。在遍历过程中，如果当前内存节点的 depth 与父节点的 depth 不相等，函数会返回 HWLOC_TYPE_DEPTH_MULTIPLE 来表示这是多重 NUMA 拓扑结构。

最后，函数会检查 depth 的值是否大于等于 0，如果是，函数返回这个深度作为最终结果。


```cpp
int
hwloc_get_memory_parents_depth (hwloc_topology_t topology)
{
  int depth = HWLOC_TYPE_DEPTH_UNKNOWN;
  /* memory leaves are always NUMA nodes for now, no need to check parents of other memory types */
  hwloc_obj_t numa = hwloc_get_obj_by_depth(topology, HWLOC_TYPE_DEPTH_NUMANODE, 0);
  assert(numa);
  while (numa) {
    hwloc_obj_t parent = numa->parent;
    /* walk-up the memory hierarchy */
    while (hwloc__obj_type_is_memory(parent->type))
      parent = parent->parent;

    if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
      depth = parent->depth;
    else if (depth != parent->depth)
      return HWLOC_TYPE_DEPTH_MULTIPLE;

    numa = numa->next_cousin;
  }

  assert(depth >= 0);
  return depth;
}

```

这两段代码定义了 `hwloc_get_obj_by_depth` 和 `hwloc_get_nbobjs_by_depth` 函数，它们用于在 `hwloc_topology` 结构体中获取不同的深度层中的 `nbobjs`（即构件）或 `obj`（即构件）数量。

`hwloc_get_nbobjs_by_depth` 函数接收一个 `struct hwloc_topology` 类型的 `topology` 和一个 `int` 类型的 `depth` 参数。它首先检查 `depth` 是否大于或等于 `topology` 中的 `nb_levels`，如果是，就执行以下操作：

1. 从 `topology` 中的 `slevels` 数组中，查找 `HWLOC_SLEVEL_FROM_DEPTH(depth)` 生成的 `l` 对应的 `nb_levels` 层，如果存在，就返回该层中的 `nbobjs` 数量。
2. 如果 `l` 不属于 `topology` 中的任何一层，则返回 0。

`hwloc_get_obj_by_depth` 函数与 `hwloc_get_nbobjs_by_depth` 类似，只是返回一个 `struct hwloc_obj` 类型的指针，而不是 `nbobjs` 数量。它同样接收一个 `struct hwloc_topology` 类型的 `topology` 和一个 `int` 类型的 `depth` 参数，以及一个 `unsigned` 类型的 `idx` 参数。它首先检查 `depth` 是否大于或等于 `topology` 中的 `nb_levels`，如果是，就执行以下操作：

1. 从 `topology` 中的 `level_nbobjects` 数组中，查找 `HWLOC_SLEVEL_FROM_DEPTH(depth)` 生成的 `l` 对应的 `nb_levels` 层，如果存在，就返回 `idx` 对应的 `nb_objects` 数组中的元素。
2. 如果 `l` 不属于 `topology` 中的任何一层，则返回 NULL。
3. 如果 `idx` 在 `level_nbobjects` 数组中，返回 `level_nbobjects` 数组中的元素。


```cpp
unsigned
hwloc_get_nbobjs_by_depth (struct hwloc_topology *topology, int depth)
{
  if ((unsigned)depth >= topology->nb_levels) {
    unsigned l = HWLOC_SLEVEL_FROM_DEPTH(depth);
    if (l < HWLOC_NR_SLEVELS)
      return topology->slevels[l].nbobjs;
    else
      return 0;
  }
  return topology->level_nbobjects[depth];
}

struct hwloc_obj *
hwloc_get_obj_by_depth (struct hwloc_topology *topology, int depth, unsigned idx)
{
  if ((unsigned)depth >= topology->nb_levels) {
    unsigned l = HWLOC_SLEVEL_FROM_DEPTH(depth);
    if (l < HWLOC_NR_SLEVELS)
      return idx < topology->slevels[l].nbobjs ? topology->slevels[l].objs[idx] : NULL;
    else
      return NULL;
  }
  if (idx >= topology->level_nbobjects[depth])
    return NULL;
  return topology->levels[depth][idx];
}

```



这三段代码定义了三种不同的 `hwloc_obj_type_t` 类型的判断函数，用于判断给定的 `type` 是否为正常、内存或 I/O 类型。

具体来说，`hwloc__obj_type_is_normal` 函数用于判断给定的 `type` 是否为操作系统支持的标准类型，例如 `hwloc_obj_type_t` 中的 `hwloc_obj_type_t`。如果是，函数返回 `true`，否则返回 `false`。

`hwloc__obj_type_is_memory` 函数用于判断给定的 `type` 是否为操作系统支持的内存类型，例如 `hwloc_obj_type_t` 中的 `hwloc_obj_type_t`。如果是，函数返回 `true`，否则返回 `false`。

`hwloc__obj_type_is_io` 函数用于判断给定的 `type` 是否为操作系统支持的对 I/O 类型，例如 `hwloc_obj_type_t` 中的 `hwloc_obj_type_t`。如果是，函数返回 `true`，否则返回 `false`。

这三段代码的作用是用于判断给定的 `type` 是否符合操作系统支持的标准类型，从而可以用于定义 `hwloc_obj_type_t` 类型的变量或函数等。


```cpp
int
hwloc_obj_type_is_normal(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_normal(type);
}

int
hwloc_obj_type_is_memory(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_memory(type);
}

int
hwloc_obj_type_is_io(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_io(type);
}

```



这三段代码是一个用于判断缓存器是否为有效缓存的函数，其中第一段定义了三个函数，后面两段继承了第一段函数，并添加了is_dcache和is_icache的参数。

具体来说，函数hwloc_obj_type_is_cache(hwloc_obj_type_t type)用于判断缓存器对象类型(hwloc_obj_type_t)是否为有效缓存。如果缓存器对象类型是缓存器对象类型，则直接返回缓存器对象的缓存器函数地址，即不进行任何计算。如果缓存器对象类型不是缓存器对象类型，则需要判断缓存器对象类型是否与有效缓存类型匹配。如果匹配，则返回匹配缓存器函数返回的逻辑值，否则返回缓存器函数的返回值。

第一段函数hwloc_obj_type_is_dcache(hwloc_obj_type_t type)与第二段函数hwloc_obj_type_is_icache(hwloc_obj_type_t type)的作用是类似的，只是前者的返回类型为int，而后者的返回类型也为int。


```cpp
int
hwloc_obj_type_is_cache(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_cache(type);
}

int
hwloc_obj_type_is_dcache(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_dcache(type);
}

int
hwloc_obj_type_is_icache(hwloc_obj_type_t type)
{
  return hwloc__obj_type_is_icache(type);
}

```

这两段代码是用来从给定的 topology 对象中获取特定类型和指定 GPU 索引的 hwloc_obj_t 类型的函数。

第一个函数 hwloc_get_obj_by_depth_and_gp_index() 可以获取 depth 为指定值，GPU 索引为给定 GPU 索引的 hwloc_obj_t 类型对象。该函数首先从 topology 对象中获取对应于 depth 值和指定的 GPU 索引的 hwloc_obj_t 对象。然后，如果找到具有相同 depth 值和 GPU 索引的 hwloc_obj_t 对象，则返回该对象。否则，通过循环遍历 depth 值的所有可能值，直到找到匹配的 hwloc_obj_t 对象，或者深度或 GPU 索引不明确时返回 NULL。

第二个函数 hwloc_get_obj_by_type_and_gp_index() 与此类似，但允许深度为浮动值，而不是固定的 depth 值。该函数也允许 GPU 索引为浮动值，而不是固定的 GPU 索引。函数首先从 topology 对象中获取对应于给定类型的 depth 值的最大 depth 值和对应的 hwloc_obj_type_t 类型。然后，如果找到具有相同类型的 depth 值和 GPU 索引的 hwloc_obj_t 对象，则返回该对象。否则，通过循环遍历 depth 值的所有可能值，直到找到匹配的 hwloc_obj_t 对象，或者深度或 GPU 索引不明确时返回 NULL。


```cpp
static hwloc_obj_t hwloc_get_obj_by_depth_and_gp_index(hwloc_topology_t topology, unsigned depth, uint64_t gp_index)
{
  hwloc_obj_t obj = hwloc_get_obj_by_depth(topology, depth, 0);
  while (obj) {
    if (obj->gp_index == gp_index)
      return obj;
    obj = obj->next_cousin;
  }
  return NULL;
}

hwloc_obj_t hwloc_get_obj_by_type_and_gp_index(hwloc_topology_t topology, hwloc_obj_type_t type, uint64_t gp_index)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
    return NULL;
  if (depth == HWLOC_TYPE_DEPTH_MULTIPLE) {
    for(depth=1 /* no multiple machine levels */;
	(unsigned) depth < topology->nb_levels-1 /* no multiple PU levels */;
	depth++) {
      if (hwloc_get_depth_type(topology, depth) == type) {
	hwloc_obj_t obj = hwloc_get_obj_by_depth_and_gp_index(topology, depth, gp_index);
	if (obj)
	  return obj;
      }
    }
    return NULL;
  }
  return hwloc_get_obj_by_depth_and_gp_index(topology, depth, gp_index);
}

```

这段代码的作用是计算给定顶图 topology 中给定源 obj 的最接近的 obj 的索引，并将该 obj 和它的后代 obj 一起返回。在计算过程中，代码会遍历 topology 的所有层级，当遍历到给定的源 obj 时，会对该 obj 的所有后代 obj 进行遍历，查找哪些 obj 属于给定源 obj 的后代，并将其存储到 obj 数组中。在遍历完成后，如果 obj 数组长度已经达到 max 最大长度，则停止遍历并返回 max 长度。


```cpp
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

```

这段代码定义了一个名为 `hwloc__get_largest_objs_inside_cpuset` 的函数，它接收三个参数： `current`、`set` 和 `res`，其中 `current` 是当前正在操作的 `hwloc_obj` 结构体的引用，`set` 是用于指定 CPU 集的 bitmap，`res` 是一个指向 `hwloc_obj` 结构体的指针数组，用于保存操作结果；`max` 是一个整数，用于保存 `hwloc_get_largest_objs_inside_cpuset` 函数返回的最大值。

函数的作用是：在给定的 `hwloc_const_bitmap_t` set 中找到所有与 `current` 的 CPU 集相交的 CPU 集中的元素，并将它们与 `current` 连接起来，然后返回连接好的元素的数量。如果 `max` 小于或等于 0，函数将返回 0，否则，返回 `hwloc__get_largest_objs_inside_cpuset` 函数返回的最大值，这个最大值将作为 `res` 指向的数组的下标。函数使用了 `hwloc_bitmap_dup` 和 `hwloc__get_largest_objs_inside_cpuset` 函数来实现。


```cpp
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

    /* if no more room to store remaining objects, return what we got so far */
    if (!*max)
      break;
  }

  return gotten;
}

```

这段代码定义了一个名为 `hwloc_get_largest_objs_inside_cpuset` 的函数，它接受一个 `struct hwloc_topology` 类型的参数 `topology`，一个包含 `hwloc_const_bitmap_t` 类型的参数 `set`，以及一个整数参数 `max`。

函数的作用是返回一个整数，表示在给定的 `set` 中，能够找到的最大包含在 `topology` 中的 `hwloc_obj` 数量。

函数的具体实现包括以下几个步骤：

1. 从 `topology` 的 `levels` 数组中，获取出 `set` 的第一个元素，即 `topology` 的根节点。
2. 如果 `set` 中不包含 `current` 节点的 `cpuset`，函数返回 -1，表示没有找到满足条件的最大包含数量。
3. 如果 `max` 的值小于或等于 0，函数返回 0，表示无法计算最大包含数量。
4. 使用 `hwloc__get_largest_objs_inside_cpuset` 函数，递归地搜索 `set` 中所有包含 `current` 节点的 `hwloc_obj`，并将其计数，最终返回当前节点和计数结果中的较大值。


```cpp
int
hwloc_get_largest_objs_inside_cpuset (struct hwloc_topology *topology, hwloc_const_bitmap_t set,
				      struct hwloc_obj **objs, int max)
{
  struct hwloc_obj *current = topology->levels[0][0];

  if (!hwloc_bitmap_isincluded(set, current->cpuset))
    return -1;

  if (max <= 0)
    return 0;

  return hwloc__get_largest_objs_inside_cpuset (current, set, &objs, &max);
}

```

这段代码是一个名为 `hwloc_obj_type_string` 的函数，它接受一个名为 `obj` 的 `hwloc_obj_type_t` 类型的参数，并返回一个表示 `obj` 的对象的类型字符串。

函数内部使用了一个 `switch` 语句，该语句根据 `obj` 的值来决定返回哪个字符串。`switch` 语句中的每个 `case` 分支分别对应于 `HWLOC_OBJ_MACHINE`、`HWLOC_OBJ_MISC`、`HWLOC_OBJ_GROUP`、`HWLOC_OBJ_MEMCACHE`、`HWLOC_OBJ_NUMANODE`、`HWLOC_OBJ_PACKAGE`、`HWLOC_OBJ_DIE`、`HWLOC_OBJ_L1CACHE`、`HWLOC_OBJ_L2CACHE`、`HWLOC_OBJ_L3CACHE` 和 `HWLOC_OBJ_L4CACHE` 这 13 个 `case` 分支。

如果没有找到相应的 `case` 分支，函数将返回一个名为 "Unknown" 的字符串。


```cpp
const char *
hwloc_obj_type_string (hwloc_obj_type_t obj)
{
  switch (obj)
    {
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

```

这段代码定义了一个名为 `hwloc__type_match` 的函数，它的功能是检查给定的字符串是否匹配给定的字符串类型，至少在 minmatch 个字符上。如果匹配成功，则返回匹配到的字符串的地址，否则返回 NULL。

函数接收两个参数：要匹配的输入字符串 `string` 和匹配的字符串类型 `type`，类型必须使用小写字母。函数内部包含一个循环，用于将输入字符串 `string` 和匹配的字符串类型 `type` 中的每一个字符进行比较。

函数的具体实现可以分为以下几个步骤：

1. 从输入字符串 `string` 的开始位置开始，逐个比较输入字符串 `string` 和匹配的字符串类型 `type` 中的每一个字符。
2. 如果输入字符串 `string` 的任何字符都早于匹配字符串类型中的第一个字符，那么函数返回 NULL，表示没有找到匹配的字符。
3. 如果输入字符串 `string` 的任何字符都晚于匹配字符串类型中的第一个字符，那么函数返回匹配到的字符串的地址，类型为 `type`。
4. 如果输入字符串 `string` 中的一个字符既不是匹配字符串类型中的字符，也不是匹配字符串类型中的字符数组中的字符，那么函数会继续比较，但如果已经遍历了 minmatch 个字符，那么函数就返回 NULL，表示没有找到匹配的字符。

由于在函数内部已经实现了所有的功能，所以函数的实现基本上不需要再解释了。


```cpp
/* Check if string matches the given type at least on minmatch chars.
 * On success, return the address of where matching stop, either pointing to \0 or to a suffix (digits, colon, etc)
 * On error, return NULL;
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
      /* string ends before type */
      if (i<minmatch)
	return NULL;
      else
	return s;
    }
    if (*s != *t && *s != *t + 'A' - 'a') {
      /* string is different */
      if ((*s >= 'a' && *s <= 'z') || (*s >= 'A' && *s <= 'Z') || *s == '-')
	/* valid character that doesn't match */
	return NULL;
      /* invalid character, we reached the end of the type namein string, stop matching here */
      if (i<minmatch)
	return NULL;
      else
	return s;
    }
  }

  return NULL;
}

```

This function appears to match input strings to the hwloc library, which is a library for managing hierarchical storage systems such as NFS.

It takes as input a string, which is then matched against a list of known suffixes for the different types of objects that can be represented by the hwloc library. The function returns either the index of the first matching suffix, or -1 if the input string cannot be matched to any of the known suffixes.

The function supports several different object types, including cache, group, and bridge objects. For each of these types, the function checks whether the input string matches a specific suffix, and returns the index of the first matching suffix if it does, or -1 if it does not match any of the known suffixes.

If the input string does not match any of the known suffixes, the function falls back to using the default suffix for the corresponding object type. For example, if the input string is a cache object, the default suffix is "cache".

If the input string does match a known suffix, the function also checks whether the input string matches the corresponding attribute for that object type. For example, if the input string is a cache object, the function also checks whether the input string matches the "depth" attribute for cache objects.

If the input string does not match any of the known attributes for the object type, the function returns -1.


```cpp
int
hwloc_type_sscanf(const char *string, hwloc_obj_type_t *typep,
		  union hwloc_obj_attr_u *attrp, size_t attrsize)
{
  hwloc_obj_type_t type = (hwloc_obj_type_t) -1;
  unsigned depthattr = (unsigned) -1;
  hwloc_obj_cache_type_t cachetypeattr = (hwloc_obj_cache_type_t) -1; /* unspecified */
  hwloc_obj_bridge_type_t ubtype = (hwloc_obj_bridge_type_t) -1;
  hwloc_obj_osdev_type_t ostype = (hwloc_obj_osdev_type_t) -1;
  char *end;

  /* Never match the ending \0 since we want to match things like core:2 too.
   * We'll only compare the beginning substring only made of letters and dash.
   */

  /* types without a custom depth */

  /* osdev subtype first to avoid conflicts coproc/core etc */
  if (hwloc__type_match(string, "osdev", 2)) {
    type = HWLOC_OBJ_OS_DEVICE;
  } else if (hwloc__type_match(string, "block", 4)) {
    type = HWLOC_OBJ_OS_DEVICE;
    ostype = HWLOC_OBJ_OSDEV_BLOCK;
  } else if (hwloc__type_match(string, "network", 3)) {
    type = HWLOC_OBJ_OS_DEVICE;
    ostype = HWLOC_OBJ_OSDEV_NETWORK;
  } else if (hwloc__type_match(string, "openfabrics", 7)) {
    type = HWLOC_OBJ_OS_DEVICE;
    ostype = HWLOC_OBJ_OSDEV_OPENFABRICS;
  } else if (hwloc__type_match(string, "dma", 3)) {
    type = HWLOC_OBJ_OS_DEVICE;
    ostype = HWLOC_OBJ_OSDEV_DMA;
  } else if (hwloc__type_match(string, "gpu", 3)) {
    type = HWLOC_OBJ_OS_DEVICE;
    ostype = HWLOC_OBJ_OSDEV_GPU;
  } else if (hwloc__type_match(string, "coproc", 5)
	     || hwloc__type_match(string, "co-processor", 6)) {
    type = HWLOC_OBJ_OS_DEVICE;
    ostype = HWLOC_OBJ_OSDEV_COPROC;

  } else if (hwloc__type_match(string, "machine", 2)) {
    type = HWLOC_OBJ_MACHINE;
  } else if (hwloc__type_match(string, "numanode", 2)
	     || hwloc__type_match(string, "node", 2)) { /* for convenience */
    type = HWLOC_OBJ_NUMANODE;
  } else if (hwloc__type_match(string, "memcache", 5)
	     || hwloc__type_match(string, "memory-side cache", 8)) {
    type = HWLOC_OBJ_MEMCACHE;
  } else if (hwloc__type_match(string, "package", 2)
	     || hwloc__type_match(string, "socket", 2)) { /* backward compat with v1.10 */
    type = HWLOC_OBJ_PACKAGE;
  } else if (hwloc__type_match(string, "die", 2)) {
    type = HWLOC_OBJ_DIE;
  } else if (hwloc__type_match(string, "core", 2)) {
    type = HWLOC_OBJ_CORE;
  } else if (hwloc__type_match(string, "pu", 2)) {
    type = HWLOC_OBJ_PU;
  } else if (hwloc__type_match(string, "misc", 4)) {
    type = HWLOC_OBJ_MISC;

  } else if (hwloc__type_match(string, "bridge", 4)) {
    type = HWLOC_OBJ_BRIDGE;
  } else if (hwloc__type_match(string, "hostbridge", 6)) {
    type = HWLOC_OBJ_BRIDGE;
    ubtype = HWLOC_OBJ_BRIDGE_HOST;
  } else if (hwloc__type_match(string, "pcibridge", 5)) {
    type = HWLOC_OBJ_BRIDGE;
    ubtype = HWLOC_OBJ_BRIDGE_PCI;
    /* if downstream_type can ever be non-PCI, we'll have to make strings more precise,
     * or relax the hwloc_type_sscanf test */

  } else if (hwloc__type_match(string, "pcidev", 3)) {
    type = HWLOC_OBJ_PCI_DEVICE;

  /* types with depthattr */
  } else if ((string[0] == 'l' || string[0] == 'L') && string[1] >= '0' && string[1] <= '9') {
    char *suffix;
    depthattr = strtol(string+1, &end, 10);
    if (*end == 'i' || *end == 'I') {
      if (depthattr >= 1 && depthattr <= 3) {
	type = HWLOC_OBJ_L1ICACHE + depthattr-1;
	cachetypeattr = HWLOC_OBJ_CACHE_INSTRUCTION;
	suffix = end+1;
      } else
	return -1;
    } else {
      if (depthattr >= 1 && depthattr <= 5) {
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
    /* check whether the optional suffix matches "cache" */
    if (!hwloc__type_match(suffix, "cache", 0))
      return -1;

  } else if ((end = (char *) hwloc__type_match(string, "group", 2)) != NULL) {
    type = HWLOC_OBJ_GROUP;
    if (*end >= '0' && *end <= '9') {
      depthattr = strtol(end, &end, 10);
    }

  } else
    return -1;

  *typep = type;
  if (attrp) {
    if (hwloc__obj_type_is_cache(type) && attrsize >= sizeof(attrp->cache)) {
      attrp->cache.depth = depthattr;
      attrp->cache.type = cachetypeattr;
    } else if (type == HWLOC_OBJ_GROUP && attrsize >= sizeof(attrp->group)) {
      attrp->group.depth = depthattr;
    } else if (type == HWLOC_OBJ_BRIDGE && attrsize >= sizeof(attrp->bridge)) {
      attrp->bridge.upstream_type = ubtype;
      attrp->bridge.downstream_type = HWLOC_OBJ_BRIDGE_PCI;
      /* if downstream_type can ever be non-PCI, we'll have to make strings more precise,
       * or relax the hwloc_type_sscanf test */
    } else if (type == HWLOC_OBJ_OS_DEVICE && attrsize >= sizeof(attrp->osdev)) {
      attrp->osdev.type = ostype;
    }
  }
  return 0;
}

```

这段代码定义了一个名为 `hwloc_type_sscanf_as_depth` 的函数，它接受一个字符串类型的参数 `string`，将其解析为深度类型，并返回该深度类型对应的 `hwloc_obj_type_t` 类型的对象。

函数实现中，首先定义了一个名为 `attr` 的联合体类型，包含一个 `hwloc_obj_attr_u` 成员变量 `attr` 和一个 `unsigned long long` 类型的成员变量 `depth`。

接着定义了一个名为 `hwloc_get_type_depth` 的函数，它接收一个 `hwloc_topology_t` 类型的参数 `topology` 和一个 `hwloc_obj_type_t` 类型的参数 `typep`，返回该深度类型对应的 depth。

接着，定义了一个名为 `hwloc_type_sscanf` 的函数，它接收一个字符串类型的参数 `string`，将其解析为深度类型，并返回它。

接着，定义了一个名为 `hwloc_get_type_depth` 的函数，它接收一个 `hwloc_topology_t` 类型的参数 `topology` 和一个 `hwloc_obj_type_t` 类型的参数 `typep`，返回该深度类型对应的 depth。

接着，定义了一个名为 `for` 的循环，用于在 `topology` 的 `levels` 数组中查找与 `attr.group.depth` 相等的深度。

最后，定义了 `hwloc_type_sscanf_as_depth` 函数，它接收一个字符串类型的参数 `string`，将其解析为深度类型，并将其类型的 `hwloc_obj_type_t` 类型的成员变量 `typep` 和深度类型的成员变量 `depth` 赋值给函数参数。

函数的作用是接收一个字符串类型的参数 `string`，将其解析为深度类型，并返回该深度类型对应的 `hwloc_obj_type_t` 类型的对象，同时将深度类型和 `typep` 类型的参数赋值给函数返回的数组。


```cpp
int
hwloc_type_sscanf_as_depth(const char *string, hwloc_obj_type_t *typep,
			   hwloc_topology_t topology, int *depthp)
{
  union hwloc_obj_attr_u attr;
  hwloc_obj_type_t type;
  int depth;
  int err;

  err = hwloc_type_sscanf(string, &type, &attr, sizeof(attr));
  if (err < 0)
    return err;

  depth = hwloc_get_type_depth(topology, type);
  if (type == HWLOC_OBJ_GROUP
      && depth == HWLOC_TYPE_DEPTH_MULTIPLE
      && attr.group.depth != (unsigned)-1) {
    unsigned l;
    depth = HWLOC_TYPE_DEPTH_UNKNOWN;
    for(l=0; l<topology->nb_levels; l++) {
      if (topology->levels[l][0]->type == HWLOC_OBJ_GROUP
	  && topology->levels[l][0]->attr->group.depth == attr.group.depth) {
	depth = (int)l;
	break;
      }
    }
  }

  if (typep)
    *typep = type;
  *depthp = depth;
  return 0;
}

```

This function appears to take a string and a hardware location object and return a human-readable name for the hardware location.

It does this by first looking at the `type` attribute of the `obj` object, which is a hardware location object. It then looks at the `attribute` attribute of the `obj` object and, if it is an `HWLOC_OBJ_BRIDGE` object, it tries to get the value of the `upstream_type` attribute.

If the `upstream_type` is `HWLOC_OBJ_BRIDGE_PCI`, it returns the string "PCIBridge". If it is any other value of `upstream_type`, it returns the string "HostBridge".

If the `obj` object does not have an `HWLOC_OBJ_BRIDGE` object, it checks the `type` attribute. If it is `HWLOC_OBJ_PCI_DEVICE`, it returns the string "PCI". If it is an `HWLOC_OBJ_OS_DEVICE`, it returns a verbose human-readable name based on the `type` attribute.

If it is not an `HWLOC_OBJ_OS_DEVICE`, it returns the string "". If it has any other `OSDEV` attribute, it attempts to get the value of the `type` attribute and, if it is `HWLOC_OBJ_OSDEV_BLOCK`, it returns the string "Block". If it is `HWLOC_OBJ_OSDEV_NETWORK`, it returns the string "Network". If it is `HWLOC_OBJ_OSDEV_OPENFABRICS`, it returns the string "OpenFabrics". If it is `HWLOC_OBJ_OSDEV_DMA`, it returns the string "DMA". If it is `HWLOC_OBJ_OSDEV_GPU`, it returns the string "GPU". If it is `HWLOC_OBJ_OSDEV_COPROC`, it returns the string "Co-Processor".

If the `obj` object does not have any of the above mentioned attributes, it will return the string "".


```cpp
static const char* hwloc_obj_cache_type_letter(hwloc_obj_cache_type_t type)
{
  switch (type) {
  case HWLOC_OBJ_CACHE_UNIFIED: return "";
  case HWLOC_OBJ_CACHE_DATA: return "d";
  case HWLOC_OBJ_CACHE_INSTRUCTION: return "i";
  default: return "unknown";
  }
}

int
hwloc_obj_type_snprintf(char * __hwloc_restrict string, size_t size, hwloc_obj_t obj, int verbose)
{
  hwloc_obj_type_t type = obj->type;
  switch (type) {
  case HWLOC_OBJ_MISC:
  case HWLOC_OBJ_MACHINE:
  case HWLOC_OBJ_NUMANODE:
  case HWLOC_OBJ_MEMCACHE:
  case HWLOC_OBJ_PACKAGE:
  case HWLOC_OBJ_DIE:
  case HWLOC_OBJ_CORE:
  case HWLOC_OBJ_PU:
    return hwloc_snprintf(string, size, "%s", hwloc_obj_type_string(type));
  case HWLOC_OBJ_L1CACHE:
  case HWLOC_OBJ_L2CACHE:
  case HWLOC_OBJ_L3CACHE:
  case HWLOC_OBJ_L4CACHE:
  case HWLOC_OBJ_L5CACHE:
  case HWLOC_OBJ_L1ICACHE:
  case HWLOC_OBJ_L2ICACHE:
  case HWLOC_OBJ_L3ICACHE:
    return hwloc_snprintf(string, size, "L%u%s%s", obj->attr->cache.depth,
			  hwloc_obj_cache_type_letter(obj->attr->cache.type),
			  verbose ? "Cache" : "");
  case HWLOC_OBJ_GROUP:
    if (obj->attr->group.depth != (unsigned) -1)
      return hwloc_snprintf(string, size, "%s%u", hwloc_obj_type_string(type), obj->attr->group.depth);
    else
      return hwloc_snprintf(string, size, "%s", hwloc_obj_type_string(type));
  case HWLOC_OBJ_BRIDGE:
    /* if downstream_type can ever be non-PCI, we'll have to make strings more precise,
     * or relax the hwloc_type_sscanf test */
    assert(obj->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI);
    return hwloc_snprintf(string, size, obj->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI ? "PCIBridge" : "HostBridge");
  case HWLOC_OBJ_PCI_DEVICE:
    return hwloc_snprintf(string, size, "PCI");
  case HWLOC_OBJ_OS_DEVICE:
    switch (obj->attr->osdev.type) {
    case HWLOC_OBJ_OSDEV_BLOCK: return hwloc_snprintf(string, size, "Block");
    case HWLOC_OBJ_OSDEV_NETWORK: return hwloc_snprintf(string, size, verbose ? "Network" : "Net");
    case HWLOC_OBJ_OSDEV_OPENFABRICS: return hwloc_snprintf(string, size, "OpenFabrics");
    case HWLOC_OBJ_OSDEV_DMA: return hwloc_snprintf(string, size, "DMA");
    case HWLOC_OBJ_OSDEV_GPU: return hwloc_snprintf(string, size, "GPU");
    case HWLOC_OBJ_OSDEV_COPROC: return hwloc_snprintf(string, size, verbose ? "Co-Processor" : "CoProc");
    default:
      if (size > 0)
	*string = '\0';
      return 0;
    }
    break;
  default:
    if (size > 0)
      *string = '\0';
    return 0;
  }
}

```

This is a C function that takes a `hwloc_device_t` object and returns a string that describes the device. The string is constructed by querying the `attr` member of the `hwloc_device_t` object, which is a structure that contains information about the device.

The function has several options to control the output of the string. The `--verbose` option enables verbose output, which prints the device name, class, and vendor IDs. The `--class` option specifies the class of the device, and the `--device` option specifies the device number.

The function uses several helper functions to extract the relevant information from the `attr` structure. The `hwloc_pci_class_string` function is used to convert a class ID to a class name string. The `linkspeed` function is used to convert the device's link speed to a string.

The function returns the string in the format `"Class ID: Device ID: Class Name: Device Name"`. If the device is unable to be found or the output format is invalid, the function returns -1.


```cpp
int
hwloc_obj_attr_snprintf(char * __hwloc_restrict string, size_t size, hwloc_obj_t obj, const char * separator, int verbose)
{
  const char *prefix = "";
  char *tmp = string;
  ssize_t tmplen = size;
  int ret = 0;
  int res;

  /* make sure we output at least an empty string */
  if (size)
    *string = '\0';

  /* print memory attributes */
  res = 0;
  if (verbose) {
    if (obj->type == HWLOC_OBJ_NUMANODE && obj->attr->numanode.local_memory)
      res = hwloc_snprintf(tmp, tmplen, "%slocal=%lu%s%stotal=%lu%s",
			   prefix,
			   (unsigned long) hwloc_memory_size_printf_value(obj->attr->numanode.local_memory, verbose),
			   hwloc_memory_size_printf_unit(obj->attr->numanode.local_memory, verbose),
			   separator,
			   (unsigned long) hwloc_memory_size_printf_value(obj->total_memory, verbose),
			   hwloc_memory_size_printf_unit(obj->total_memory, verbose));
    else if (obj->total_memory)
      res = hwloc_snprintf(tmp, tmplen, "%stotal=%lu%s",
			   prefix,
			   (unsigned long) hwloc_memory_size_printf_value(obj->total_memory, verbose),
			   hwloc_memory_size_printf_unit(obj->total_memory, verbose));
  } else {
    if (obj->type == HWLOC_OBJ_NUMANODE && obj->attr->numanode.local_memory)
      res = hwloc_snprintf(tmp, tmplen, "%s%lu%s",
			   prefix,
			   (unsigned long) hwloc_memory_size_printf_value(obj->attr->numanode.local_memory, verbose),
			   hwloc_memory_size_printf_unit(obj->attr->numanode.local_memory, verbose));
  }
  if (res < 0)
    return -1;
  ret += res;
  if (ret > 0)
    prefix = separator;
  if (res >= tmplen)
    res = tmplen>0 ? (int)tmplen - 1 : 0;
  tmp += res;
  tmplen -= res;

  /* printf type-specific attributes */
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
    if (verbose) {
      char assoc[32];
      if (obj->attr->cache.associativity == -1)
	snprintf(assoc, sizeof(assoc), "%sfully-associative", separator);
      else if (obj->attr->cache.associativity == 0)
	*assoc = '\0';
      else
	snprintf(assoc, sizeof(assoc), "%sways=%d", separator, obj->attr->cache.associativity);
      res = hwloc_snprintf(tmp, tmplen, "%ssize=%lu%s%slinesize=%u%s",
			   prefix,
			   (unsigned long) hwloc_memory_size_printf_value(obj->attr->cache.size, verbose),
			   hwloc_memory_size_printf_unit(obj->attr->cache.size, verbose),
			   separator, obj->attr->cache.linesize,
			   assoc);
    } else
      res = hwloc_snprintf(tmp, tmplen, "%s%lu%s",
			   prefix,
			   (unsigned long) hwloc_memory_size_printf_value(obj->attr->cache.size, verbose),
			   hwloc_memory_size_printf_unit(obj->attr->cache.size, verbose));
    break;
  case HWLOC_OBJ_BRIDGE:
    if (verbose) {
      char up[128], down[64];
      /* upstream is PCI or HOST */
      if (obj->attr->bridge.upstream_type == HWLOC_OBJ_BRIDGE_PCI) {
        char linkspeed[64]= "";
        if (obj->attr->pcidev.linkspeed)
          snprintf(linkspeed, sizeof(linkspeed), "%slink=%.2fGB/s", separator, obj->attr->pcidev.linkspeed);
	snprintf(up, sizeof(up), "busid=%04x:%02x:%02x.%01x%sid=%04x:%04x%sclass=%04x(%s)%s",
		 obj->attr->pcidev.domain, obj->attr->pcidev.bus, obj->attr->pcidev.dev, obj->attr->pcidev.func, separator,
		 obj->attr->pcidev.vendor_id, obj->attr->pcidev.device_id, separator,
		 obj->attr->pcidev.class_id, hwloc_pci_class_string(obj->attr->pcidev.class_id), linkspeed);
      } else
        *up = '\0';
      /* downstream is_PCI */
      if (obj->attr->bridge.downstream_type == HWLOC_OBJ_BRIDGE_PCI) {
        snprintf(down, sizeof(down), "buses=%04x:[%02x-%02x]",
                 obj->attr->bridge.downstream.pci.domain, obj->attr->bridge.downstream.pci.secondary_bus, obj->attr->bridge.downstream.pci.subordinate_bus);
      } else
        assert(0);
      if (*up)
	res = hwloc_snprintf(string, size, "%s%s%s", up, separator, down);
      else
	res = hwloc_snprintf(string, size, "%s", down);
    }
    break;
  case HWLOC_OBJ_PCI_DEVICE:
    if (verbose) {
      char linkspeed[64]= "";
      if (obj->attr->pcidev.linkspeed)
        snprintf(linkspeed, sizeof(linkspeed), "%slink=%.2fGB/s", separator, obj->attr->pcidev.linkspeed);
      res = hwloc_snprintf(string, size, "busid=%04x:%02x:%02x.%01x%sid=%04x:%04x%sclass=%04x(%s)%s",
			   obj->attr->pcidev.domain, obj->attr->pcidev.bus, obj->attr->pcidev.dev, obj->attr->pcidev.func, separator,
			   obj->attr->pcidev.vendor_id, obj->attr->pcidev.device_id, separator,
			   obj->attr->pcidev.class_id, hwloc_pci_class_string(obj->attr->pcidev.class_id), linkspeed);
    }
    break;
  default:
    break;
  }
  if (res < 0)
    return -1;
  ret += res;
  if (ret > 0)
    prefix = separator;
  if (res >= tmplen)
    res = tmplen>0 ? (int)tmplen - 1 : 0;
  tmp += res;
  tmplen -= res;

  /* printf infos */
  if (verbose) {
    unsigned i;
    for(i=0; i<obj->infos_count; i++) {
      struct hwloc_info_s *info = &obj->infos[i];
      const char *quote;
      if (strchr(info->value, ' '))
        quote = "\"";
      else
        quote = "";
      res = hwloc_snprintf(tmp, tmplen, "%s%s=%s%s%s",
			     prefix,
			     info->name,
			     quote, info->value, quote);
      if (res < 0)
        return -1;
      ret += res;
      if (res >= tmplen)
        res = tmplen>0 ? (int)tmplen - 1 : 0;
      tmp += res;
      tmplen -= res;
      if (ret > 0)
        prefix = separator;
    }
  }

  return ret;
}

```

这段代码是一个名为 `hwloc_bitmap_singlify_per_core` 的函数，它用于在给定的硬件布局中提取出指定的芯片集（CPUT）。

在函数中，首先定义了一个名为 `core` 的变量，它是一个 `hwloc_obj_t` 类型的指针，指向一个包含指定芯片集的辅助类结构体。

接着定义了一个名为 `which` 的无符号整数变量，用于保存需要提取的芯片组。

在下一行中，定义了一个名为 `hwloc_get_next_obj_covering_cpuset_by_type` 的函数，用于获取包含指定芯片集的辅助类结构体。这个函数接收两个参数：硬件布局类型和芯片集，然后返回下一个这样的结构体。

在下一行中，定义了一个名为 `hwloc_get_first_exclusive_pu_by_index` 的函数，用于获取芯片集中与指定索引相对应的偏移量。这个函数接收两个参数：硬件布局类型和芯片集，然后返回芯片集中的第一个偏移量。

在下一行中，定义了一个循环，用于遍历芯片集中所有与指定索引相对应的偏移量。在循环内部，首先定义了一个名为 `pu` 的变量，用于保存当前芯片集中的偏移量。

在循环内部，定义了一个名为 `do` 的循环，用于尝试从芯片集中找到与指定索引相对应的偏移量。如果找到了，则执行以下操作：

1. 尝试使用 `hwloc_bitmap_next` 函数查找与指定偏移量相对应的下一个芯片。
2. 如果找到了这个芯片，则执行以下操作：
	1. 使用 `hwloc_bitmap_isset` 函数检查当前芯片集中的芯片是否与指定芯片集中的芯片匹配。
	2. 如果匹配，则执行以下操作：
		1. 使用 `hwloc_bitmap_andnot` 函数和 `hwloc_bitmap_get_first_exclusive_pu_by_index` 函数将当前芯片集中的芯片从整个芯片集中移除，并设置指定芯片集中的偏移量为匹配的偏移量。
		2. 使用 `hwloc_bitmap_set` 函数将指定芯片集中的芯片标记为匹配的偏移量。
		3. 执行 `i` 步，索引加一。

在循环内部，如果芯片集中找不到与指定索引相对应的芯片，则执行以下操作：

1. 使用 `hwloc_bitmap_andnot` 函数和 `hwloc_bitmap_get_first_exclusive_pu_by_index` 函数将当前芯片集中的芯片从整个芯片集中移除，并设置指定芯片集中的偏移量为匹配的偏移量。
2. 使用 `hwloc_bitmap_set` 函数将指定芯片集中的芯片标记为匹配的偏移量。
3. 执行 `i` 步，索引加一。

最后，函数返回 0，表示操作成功完成。


```cpp
int hwloc_bitmap_singlify_per_core(hwloc_topology_t topology, hwloc_bitmap_t cpuset, unsigned which)
{
  hwloc_obj_t core = NULL;
  while ((core = hwloc_get_next_obj_covering_cpuset_by_type(topology, cpuset, HWLOC_OBJ_CORE, core)) != NULL) {
    /* this core has some PUs in the cpuset, find the index-th one */
    unsigned i = 0;
    int pu = -1;
    do {
      pu = hwloc_bitmap_next(core->cpuset, pu);
      if (pu == -1) {
	/* no which-th PU in cpuset and core, remove the entire core */
	hwloc_bitmap_andnot(cpuset, cpuset, core->cpuset);
	break;
      }
      if (hwloc_bitmap_isset(cpuset, pu)) {
	if (i == which) {
	  /* remove the entire core except that exact pu */
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

```



This function appears to be a part of the Linux kernel's "pcibay.h" file, which is related to the platform device driver's device tree representation.

It appears to be a utility function that walks up the device tree hierarchy from a specific platform device object (such as a PCI device) to find a specific device object (such as an OS device). The function takes a single parameter, which is a pointer to the current platform device object, and returns a pointer to the matching device object.

The function first checks if the given object is an OS device, and if it is not, it returns an error and wraps up. If the object is an OS device, it then checks if it has a specific subtype. If it does not, it returns an error and wraps up. If the object has a specific subtype, it then checks if it has a matching OS device name prefix. If it does, it returns the object. If the object does not have a matching OS device name prefix or is not an OS device, it returns an error and wraps up.

If the function successfully finds the matching device object, it returns a pointer to it. If it does not, it returns NULL.


```cpp
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
    /* normal/memory type, look for normal/memory type with same sets */
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
    return NULL;

  } else if (hwloc_obj_type_is_io(src->type)) {
    /* I/O device, look for PCI/OS in same PCI */
    hwloc_obj_t pci;

    if ((src->type != HWLOC_OBJ_OS_DEVICE && src->type != HWLOC_OBJ_PCI_DEVICE)
        || (type != HWLOC_OBJ_OS_DEVICE && type != HWLOC_OBJ_PCI_DEVICE)) {
      errno = EINVAL;
      return NULL;
    }

    /* walk up to find the container */
    pci = src;
    while (pci->type == HWLOC_OBJ_OS_DEVICE)
      pci = pci->parent;

    if (type == HWLOC_OBJ_PCI_DEVICE) {
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
      /* find a matching osdev child */
      assert(type == HWLOC_OBJ_OS_DEVICE);
      /* FIXME: won't work if we ever store osdevs in osdevs */
      hwloc_obj_t child;
      for(child = pci->io_first_child; child; child = child->next_sibling) {
        if (child->type != HWLOC_OBJ_OS_DEVICE)
          /* FIXME: should never occur currently */
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
    /* nothing for Misc */
    errno = EINVAL;
    return NULL;
  }
}

```

# `src/3rdparty/libethash/data_sizes.h`

这段代码是一个C++文件，属于cpp-ethereum项目。它通过引入ethereum库，允许用户方便地与以太坊进行交互。这个库是免费软件，遵循了GNU通用公共许可证（GPL）第3版或之后的版本。

在这段注释中，说明了这个库的创建目的是为了方便，但它不保证其质量或者能够顺利运行。这个库的发布也有一定的风险，使用时需自行承担风险。

作者建议在安装这个库时，查看GNU通用公共许可证以获得更多信息。如果你没有从这个许可证中获得任何内容，可以通过访问<http://www.gnu.org/licenses/>获取相关信息。


```cpp
/*
  This file is part of cpp-ethereum.

  cpp-ethereum is free software: you can redistribute it and/or modify
  it under the terms of the GNU General Public License as published by
  the Free Software FoundationUUU,either version 3 of the LicenseUUU,or
  (at your option) any later version.

  cpp-ethereum is distributed in the hope that it will be usefulU,
  but WITHOUT ANY WARRANTY; without even the implied warranty of
  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
  GNU General Public License for more details.

  You should have received a copy of the GNU General Public License
  along with cpp-ethereum.  If notUUU,see <http://www.gnu.org/licenses/>.
```

这段代码是一个C语言的预处理指令，名为“#pragma once”。它告诉编译器在编译之前将这段代码保存为“data_sizes.h”文件。

具体来说，这段代码的作用是告诉编译器在编译之前（而不是在编译时）检查定义是否已经被申明。这个指令有一个预处理文档，其中包含一个警告，即“不要在每次编译时重新定义已经定义的函数”。

如果这个指令允许在编译时重新定义已经定义的函数，那么编译器会在编译时产生一些警告，告诉开发人员避免这种错误。因此，这个指令通常用于确保代码在编译之前不会出现问题。


```cpp
*/

/** @file data_sizes.h
* @author Matthew Wampler-Doty <negacthulhu@gmail.com>
* @date 2015
*/

#pragma once

#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

```

这段代码是一个 Mathematica 代码，计算了约 20 年时间内，2048 个独立诚信算法（8 位二进制数）构成的有向无环图（DAG）的大小。

具体来说，这段代码首先定义了一个名为 `CacheSizeBytesInit` 的常量，表示 20 年时间内Cache中缓存用于存储每个独立诚信算法的大小（也就是 2^24 字节）。然后定义了一个名为 `CacheGrowth` 的常量，表示在计算当前Cache大小是否已经达到上限时，每次增长缓存的大小（也就是 2^17）。

接着定义了一个名为 `HashBytes` 的常量，表示每次计算独立诚信算法时，使用的哈希表的大小（也就是 64 字节）。

接下来是一个名为 `Reap` 的函数，这个函数使用了缓存算法。在 `Reap` 函数中，首先定义了一个名为 `j` 的变量，用于跟踪当前已经计算过的独立诚信算法的个数。然后进入一个循环，每次计算缓存的剩余空间是否足够再计算一个独立诚信算法。

在循环体内，首先计算出 `CacheSizeBytesInit + CacheGrowth * j` 大小，用于计算下一个独立诚信算法需要使用的缓存空间。然后将这个缓存空间大小除以 `HashBytes`，得到一个 8 位二进制数表示的缓存空间大小。如果计算得到的缓存空间大小小于当前缓存空间大小，那么就继续计算下一个独立诚信算法需要使用的缓存空间。如果计算得到的缓存空间大小大于当前缓存空间大小，那么就说明当前缓存空间已经达到了上限，需要将缓存的增长量 `CacheGrowth` 应用到当前缓存空间大小中，并将 `j` 加 1，继续计算下一个独立诚信算法需要使用的缓存空间。

在循环体内，如果计算下一个独立诚信算法需要使用的缓存空间小于当前缓存空间大小，那么就直接返回缓存空间大小，并将 `j` 加 1。如果计算下一个独立诚信算法需要使用的缓存空间大于当前缓存空间大小，那么就使用 `Reap` 函数计算出所有独立诚信算法需要使用的缓存空间，并将它们相加，最后用这个总和减去当前缓存空间大小，得到一个剩余的缓存空间大小。

最后，在 `Reap` 函数中，将计算得到的缓存空间大小作为函数的返回值，并用它来表示DAG的大小。


```cpp
#include <stdint.h>

// 2048 Epochs (~20 years) worth of tabulated DAG sizes

// Generated with the following Mathematica Code:

// GetCacheSizes[n_] := Module[{
//        CacheSizeBytesInit = 2^24,
//        CacheGrowth = 2^17,
//        HashBytes = 64,
//        j = 0},
//       Reap[
//         While[j < n,
//           Module[{i =
//             Floor[(CacheSizeBytesInit + CacheGrowth * j) / HashBytes]},
```

It appears that the memory address you provided is a loop counter for a microcontroller or microprocessor. The loop counter is a register that is used to keep track of the current value of a counter that is incremented each time the microcontroller or microprocessor executes a certain number of instructions.

The loop counter is typically used to implement an infinite loop that performs a particular task over and over again until it is stopped or reset. The number of instructions that are executed before the loop counter is reset or stopped depends on the specific implementation of the microcontroller or microprocessor.

In general, the loop counter is an important register that allows for efficient and reliable operation of the microcontroller or microprocessor.


```cpp
//             While[! PrimeQ[i], i--];
//             Sow[i*HashBytes]; j++]]]][[2]][[1]]


static const uint64_t dag_sizes[2048] = {
	1073739904U, 1082130304U, 1090514816U, 1098906752U, 1107293056U,
	1115684224U, 1124070016U, 1132461952U, 1140849536U, 1149232768U,
	1157627776U, 1166013824U, 1174404736U, 1182786944U, 1191180416U,
	1199568512U, 1207958912U, 1216345216U, 1224732032U, 1233124736U,
	1241513344U, 1249902464U, 1258290304U, 1266673792U, 1275067264U,
	1283453312U, 1291844992U, 1300234112U, 1308619904U, 1317010048U,
	1325397376U, 1333787776U, 1342176128U, 1350561664U, 1358954368U,
	1367339392U, 1375731584U, 1384118144U, 1392507008U, 1400897408U,
	1409284736U, 1417673344U, 1426062464U, 1434451072U, 1442839168U,
	1451229056U, 1459615616U, 1468006016U, 1476394112U, 1484782976U,
	1493171584U, 1501559168U, 1509948032U, 1518337664U, 1526726528U,
	1535114624U, 1543503488U, 1551892096U, 1560278656U, 1568669056U,
	1577056384U, 1585446272U, 1593831296U, 1602219392U, 1610610304U,
	1619000192U, 1627386752U, 1635773824U, 1644164224U, 1652555648U,
	1660943488U, 1669332608U, 1677721216U, 1686109312U, 1694497664U,
	1702886272U, 1711274624U, 1719661184U, 1728047744U, 1736434816U,
	1744829056U, 1753218944U, 1761606272U, 1769995904U, 1778382464U,
	1786772864U, 1795157888U, 1803550592U, 1811937664U, 1820327552U,
	1828711552U, 1837102976U, 1845488768U, 1853879936U, 1862269312U,
	1870656896U, 1879048064U, 1887431552U, 1895825024U, 1904212096U,
	1912601216U, 1920988544U, 1929379456U, 1937765504U, 1946156672U,
	1954543232U, 1962932096U, 1971321728U, 1979707264U, 1988093056U,
	1996487552U, 2004874624U, 2013262208U, 2021653888U, 2030039936U,
	2038430848U, 2046819968U, 2055208576U, 2063596672U, 2071981952U,
	2080373632U, 2088762752U, 2097149056U, 2105539712U, 2113928576U,
	2122315136U, 2130700672U, 2139092608U, 2147483264U, 2155872128U,
	2164257664U, 2172642176U, 2181035392U, 2189426048U, 2197814912U,
	2206203008U, 2214587264U, 2222979712U, 2231367808U, 2239758208U,
	2248145024U, 2256527744U, 2264922752U, 2273312128U, 2281701248U,
	2290086272U, 2298476672U, 2306867072U, 2315251072U, 2323639168U,
	2332032128U, 2340420224U, 2348808064U, 2357196416U, 2365580416U,
	2373966976U, 2382363008U, 2390748544U, 2399139968U, 2407530368U,
	2415918976U, 2424307328U, 2432695424U, 2441084288U, 2449472384U,
	2457861248U, 2466247808U, 2474637184U, 2483026816U, 2491414144U,
	2499803776U, 2508191872U, 2516582272U, 2524970368U, 2533359232U,
	2541743488U, 2550134144U, 2558525056U, 2566913408U, 2575301504U,
	2583686528U, 2592073856U, 2600467328U, 2608856192U, 2617240448U,
	2625631616U, 2634022016U, 2642407552U, 2650796416U, 2659188352U,
	2667574912U, 2675965312U, 2684352896U, 2692738688U, 2701130624U,
	2709518464U, 2717907328U, 2726293376U, 2734685056U, 2743073152U,
	2751462016U, 2759851648U, 2768232832U, 2776625536U, 2785017728U,
	2793401984U, 2801794432U, 2810182016U, 2818571648U, 2826959488U,
	2835349376U, 2843734144U, 2852121472U, 2860514432U, 2868900992U,
	2877286784U, 2885676928U, 2894069632U, 2902451584U, 2910843008U,
	2919234688U, 2927622784U, 2936011648U, 2944400768U, 2952789376U,
	2961177728U, 2969565568U, 2977951616U, 2986338944U, 2994731392U,
	3003120256U, 3011508352U, 3019895936U, 3028287104U, 3036675968U,
	3045063808U, 3053452928U, 3061837696U, 3070228352U, 3078615424U,
	3087003776U, 3095394944U, 3103782272U, 3112173184U, 3120562048U,
	3128944768U, 3137339264U, 3145725056U, 3154109312U, 3162505088U,
	3170893184U, 3179280256U, 3187669376U, 3196056704U, 3204445568U,
	3212836736U, 3221224064U, 3229612928U, 3238002304U, 3246391168U,
	3254778496U, 3263165824U, 3271556224U, 3279944576U, 3288332416U,
	3296719232U, 3305110912U, 3313500032U, 3321887104U, 3330273152U,
	3338658944U, 3347053184U, 3355440512U, 3363827072U, 3372220288U,
	3380608384U, 3388997504U, 3397384576U, 3405774208U, 3414163072U,
	3422551936U, 3430937984U, 3439328384U, 3447714176U, 3456104576U,
	3464493952U, 3472883584U, 3481268864U, 3489655168U, 3498048896U,
	3506434432U, 3514826368U, 3523213952U, 3531603584U, 3539987072U,
	3548380288U, 3556763264U, 3565157248U, 3573545344U, 3581934464U,
	3590324096U, 3598712704U, 3607098752U, 3615488384U, 3623877248U,
	3632265856U, 3640646528U, 3649043584U, 3657430144U, 3665821568U,
	3674207872U, 3682597504U, 3690984832U, 3699367808U, 3707764352U,
	3716152448U, 3724541056U, 3732925568U, 3741318016U, 3749706368U,
	3758091136U, 3766481536U, 3774872704U, 3783260032U, 3791650432U,
	3800036224U, 3808427648U, 3816815488U, 3825204608U, 3833592704U,
	3841981568U, 3850370432U, 3858755968U, 3867147904U, 3875536256U,
	3883920512U, 3892313728U, 3900702592U, 3909087872U, 3917478784U,
	3925868416U, 3934256512U, 3942645376U, 3951032192U, 3959422336U,
	3967809152U, 3976200064U, 3984588416U, 3992974976U, 4001363584U,
	4009751168U, 4018141312U, 4026530432U, 4034911616U, 4043308928U,
	4051695488U, 4060084352U, 4068472448U, 4076862848U, 4085249408U,
	4093640576U, 4102028416U, 4110413696U, 4118805632U, 4127194496U,
	4135583104U, 4143971968U, 4152360832U, 4160746112U, 4169135744U,
	4177525888U, 4185912704U, 4194303616U, 4202691968U, 4211076736U,
	4219463552U, 4227855488U, 4236246656U, 4244633728U, 4253022848U,
	4261412224U, 4269799808U, 4278184832U, 4286578048U, 4294962304U,
	4303349632U, 4311743104U, 4320130432U, 4328521088U, 4336909184U,
	4345295488U, 4353687424U, 4362073472U, 4370458496U, 4378852736U,
	4387238528U, 4395630208U, 4404019072U, 4412407424U, 4420790656U,
	4429182848U, 4437571456U, 4445962112U, 4454344064U, 4462738048U,
	4471119232U, 4479516544U, 4487904128U, 4496289664U, 4504682368U,
	4513068416U, 4521459584U, 4529846144U, 4538232704U, 4546619776U,
	4555010176U, 4563402112U, 4571790208U, 4580174464U, 4588567936U,
	4596957056U, 4605344896U, 4613734016U, 4622119808U, 4630511488U,
	4638898816U, 4647287936U, 4655675264U, 4664065664U, 4672451968U,
	4680842624U, 4689231488U, 4697620352U, 4706007424U, 4714397056U,
	4722786176U, 4731173248U, 4739562368U, 4747951744U, 4756340608U,
	4764727936U, 4773114496U, 4781504384U, 4789894784U, 4798283648U,
	4806667648U, 4815059584U, 4823449472U, 4831835776U, 4840226176U,
	4848612224U, 4857003392U, 4865391488U, 4873780096U, 4882169728U,
	4890557312U, 4898946944U, 4907333248U, 4915722368U, 4924110976U,
	4932499328U, 4940889728U, 4949276032U, 4957666432U, 4966054784U,
	4974438016U, 4982831488U, 4991221376U, 4999607168U, 5007998848U,
	5016386432U, 5024763776U, 5033164672U, 5041544576U, 5049941888U,
	5058329728U, 5066717056U, 5075107456U, 5083494272U, 5091883904U,
	5100273536U, 5108662144U, 5117048192U, 5125436032U, 5133827456U,
	5142215296U, 5150605184U, 5158993024U, 5167382144U, 5175769472U,
	5184157568U, 5192543872U, 5200936064U, 5209324928U, 5217711232U,
	5226102656U, 5234490496U, 5242877312U, 5251263872U, 5259654016U,
	5268040832U, 5276434304U, 5284819328U, 5293209728U, 5301598592U,
	5309986688U, 5318374784U, 5326764416U, 5335151488U, 5343542144U,
	5351929472U, 5360319872U, 5368706944U, 5377096576U, 5385484928U,
	5393871232U, 5402263424U, 5410650496U, 5419040384U, 5427426944U,
	5435816576U, 5444205952U, 5452594816U, 5460981376U, 5469367936U,
	5477760896U, 5486148736U, 5494536832U, 5502925952U, 5511315328U,
	5519703424U, 5528089984U, 5536481152U, 5544869504U, 5553256064U,
	5561645696U, 5570032768U, 5578423936U, 5586811264U, 5595193216U,
	5603585408U, 5611972736U, 5620366208U, 5628750464U, 5637143936U,
	5645528192U, 5653921408U, 5662310272U, 5670694784U, 5679082624U,
	5687474048U, 5695864448U, 5704251008U, 5712641408U, 5721030272U,
	5729416832U, 5737806208U, 5746194304U, 5754583936U, 5762969984U,
	5771358592U, 5779748224U, 5788137856U, 5796527488U, 5804911232U,
	5813300608U, 5821692544U, 5830082176U, 5838468992U, 5846855552U,
	5855247488U, 5863636096U, 5872024448U, 5880411008U, 5888799872U,
	5897186432U, 5905576832U, 5913966976U, 5922352768U, 5930744704U,
	5939132288U, 5947522432U, 5955911296U, 5964299392U, 5972688256U,
	5981074304U, 5989465472U, 5997851008U, 6006241408U, 6014627968U,
	6023015552U, 6031408256U, 6039796096U, 6048185216U, 6056574848U,
	6064963456U, 6073351808U, 6081736064U, 6090128768U, 6098517632U,
	6106906496U, 6115289216U, 6123680896U, 6132070016U, 6140459648U,
	6148849024U, 6157237376U, 6165624704U, 6174009728U, 6182403712U,
	6190792064U, 6199176064U, 6207569792U, 6215952256U, 6224345216U,
	6232732544U, 6241124224U, 6249510272U, 6257899136U, 6266287744U,
	6274676864U, 6283065728U, 6291454336U, 6299843456U, 6308232064U,
	6316620928U, 6325006208U, 6333395584U, 6341784704U, 6350174848U,
	6358562176U, 6366951296U, 6375337856U, 6383729536U, 6392119168U,
	6400504192U, 6408895616U, 6417283456U, 6425673344U, 6434059136U,
	6442444672U, 6450837376U, 6459223424U, 6467613056U, 6476004224U,
	6484393088U, 6492781952U, 6501170048U, 6509555072U, 6517947008U,
	6526336384U, 6534725504U, 6543112832U, 6551500672U, 6559888768U,
	6568278656U, 6576662912U, 6585055616U, 6593443456U, 6601834112U,
	6610219648U, 6618610304U, 6626999168U, 6635385472U, 6643777408U,
	6652164224U, 6660552832U, 6668941952U, 6677330048U, 6685719424U,
	6694107776U, 6702493568U, 6710882176U, 6719274112U, 6727662976U,
	6736052096U, 6744437632U, 6752825984U, 6761213824U, 6769604224U,
	6777993856U, 6786383488U, 6794770816U, 6803158144U, 6811549312U,
	6819937664U, 6828326528U, 6836706176U, 6845101696U, 6853491328U,
	6861880448U, 6870269312U, 6878655104U, 6887046272U, 6895433344U,
	6903822208U, 6912212864U, 6920596864U, 6928988288U, 6937377152U,
	6945764992U, 6954149248U, 6962544256U, 6970928768U, 6979317376U,
	6987709312U, 6996093824U, 7004487296U, 7012875392U, 7021258624U,
	7029652352U, 7038038912U, 7046427776U, 7054818944U, 7063207808U,
	7071595136U, 7079980928U, 7088372608U, 7096759424U, 7105149824U,
	7113536896U, 7121928064U, 7130315392U, 7138699648U, 7147092352U,
	7155479168U, 7163865728U, 7172249984U, 7180648064U, 7189036672U,
	7197424768U, 7205810816U, 7214196608U, 7222589824U, 7230975104U,
	7239367552U, 7247755904U, 7256145536U, 7264533376U, 7272921472U,
	7281308032U, 7289694848U, 7298088832U, 7306471808U, 7314864512U,
	7323253888U, 7331643008U, 7340029568U, 7348419712U, 7356808832U,
	7365196672U, 7373585792U, 7381973888U, 7390362752U, 7398750592U,
	7407138944U, 7415528576U, 7423915648U, 7432302208U, 7440690304U,
	7449080192U, 7457472128U, 7465860992U, 7474249088U, 7482635648U,
	7491023744U, 7499412608U, 7507803008U, 7516192384U, 7524579968U,
	7532967296U, 7541358464U, 7549745792U, 7558134656U, 7566524032U,
	7574912896U, 7583300992U, 7591690112U, 7600075136U, 7608466816U,
	7616854912U, 7625244544U, 7633629824U, 7642020992U, 7650410368U,
	7658794112U, 7667187328U, 7675574912U, 7683961984U, 7692349568U,
	7700739712U, 7709130368U, 7717519232U, 7725905536U, 7734295424U,
	7742683264U, 7751069056U, 7759457408U, 7767849088U, 7776238208U,
	7784626816U, 7793014912U, 7801405312U, 7809792128U, 7818179968U,
	7826571136U, 7834957184U, 7843347328U, 7851732352U, 7860124544U,
	7868512384U, 7876902016U, 7885287808U, 7893679744U, 7902067072U,
	7910455936U, 7918844288U, 7927230848U, 7935622784U, 7944009344U,
	7952400256U, 7960786048U, 7969176704U, 7977565312U, 7985953408U,
	7994339968U, 8002730368U, 8011119488U, 8019508096U, 8027896192U,
	8036285056U, 8044674688U, 8053062272U, 8061448832U, 8069838464U,
	8078227328U, 8086616704U, 8095006592U, 8103393664U, 8111783552U,
	8120171392U, 8128560256U, 8136949376U, 8145336704U, 8153726848U,
	8162114944U, 8170503296U, 8178891904U, 8187280768U, 8195669632U,
	8204058496U, 8212444544U, 8220834176U, 8229222272U, 8237612672U,
	8246000768U, 8254389376U, 8262775168U, 8271167104U, 8279553664U,
	8287944064U, 8296333184U, 8304715136U, 8313108352U, 8321497984U,
	8329885568U, 8338274432U, 8346663296U, 8355052928U, 8363441536U,
	8371828352U, 8380217984U, 8388606592U, 8396996224U, 8405384576U,
	8413772672U, 8422161536U, 8430549376U, 8438939008U, 8447326592U,
	8455715456U, 8464104832U, 8472492928U, 8480882048U, 8489270656U,
	8497659776U, 8506045312U, 8514434944U, 8522823808U, 8531208832U,
	8539602304U, 8547990656U, 8556378752U, 8564768384U, 8573154176U,
	8581542784U, 8589933952U, 8598322816U, 8606705024U, 8615099264U,
	8623487872U, 8631876992U, 8640264064U, 8648653952U, 8657040256U,
	8665430656U, 8673820544U, 8682209152U, 8690592128U, 8698977152U,
	8707374464U, 8715763328U, 8724151424U, 8732540032U, 8740928384U,
	8749315712U, 8757704576U, 8766089344U, 8774480768U, 8782871936U,
	8791260032U, 8799645824U, 8808034432U, 8816426368U, 8824812928U,
	8833199488U, 8841591424U, 8849976448U, 8858366336U, 8866757248U,
	8875147136U, 8883532928U, 8891923328U, 8900306816U, 8908700288U,
	8917088384U, 8925478784U, 8933867392U, 8942250368U, 8950644608U,
	8959032704U, 8967420544U, 8975809664U, 8984197504U, 8992584064U,
	9000976256U, 9009362048U, 9017752448U, 9026141312U, 9034530688U,
	9042917504U, 9051307904U, 9059694208U, 9068084864U, 9076471424U,
	9084861824U, 9093250688U, 9101638528U, 9110027648U, 9118416512U,
	9126803584U, 9135188096U, 9143581312U, 9151969664U, 9160356224U,
	9168747136U, 9177134464U, 9185525632U, 9193910144U, 9202302848U,
	9210690688U, 9219079552U, 9227465344U, 9235854464U, 9244244864U,
	9252633472U, 9261021824U, 9269411456U, 9277799296U, 9286188928U,
	9294574208U, 9302965888U, 9311351936U, 9319740032U, 9328131968U,
	9336516736U, 9344907392U, 9353296768U, 9361685888U, 9370074752U,
	9378463616U, 9386849408U, 9395239808U, 9403629184U, 9412016512U,
	9420405376U, 9428795008U, 9437181568U, 9445570688U, 9453960832U,
	9462346624U, 9470738048U, 9479121536U, 9487515008U, 9495903616U,
	9504289664U, 9512678528U, 9521067904U, 9529456256U, 9537843584U,
	9546233728U, 9554621312U, 9563011456U, 9571398784U, 9579788672U,
	9588178304U, 9596567168U, 9604954496U, 9613343104U, 9621732992U,
	9630121856U, 9638508416U, 9646898816U, 9655283584U, 9663675776U,
	9672061312U, 9680449664U, 9688840064U, 9697230464U, 9705617536U,
	9714003584U, 9722393984U, 9730772608U, 9739172224U, 9747561088U,
	9755945344U, 9764338816U, 9772726144U, 9781116544U, 9789503872U,
	9797892992U, 9806282624U, 9814670464U, 9823056512U, 9831439232U,
	9839833984U, 9848224384U, 9856613504U, 9865000576U, 9873391232U,
	9881772416U, 9890162816U, 9898556288U, 9906940544U, 9915333248U,
	9923721088U, 9932108672U, 9940496512U, 9948888448U, 9957276544U,
	9965666176U, 9974048384U, 9982441088U, 9990830464U, 9999219584U,
	10007602816U, 10015996544U, 10024385152U, 10032774016U, 10041163648U,
	10049548928U, 10057940096U, 10066329472U, 10074717824U, 10083105152U,
	10091495296U, 10099878784U, 10108272256U, 10116660608U, 10125049216U,
	10133437312U, 10141825664U, 10150213504U, 10158601088U, 10166991232U,
	10175378816U, 10183766144U, 10192157312U, 10200545408U, 10208935552U,
	10217322112U, 10225712768U, 10234099328U, 10242489472U, 10250876032U,
	10259264896U, 10267656064U, 10276042624U, 10284429184U, 10292820352U,
	10301209472U, 10309598848U, 10317987712U, 10326375296U, 10334763392U,
	10343153536U, 10351541632U, 10359930752U, 10368318592U, 10376707456U,
	10385096576U, 10393484672U, 10401867136U, 10410262144U, 10418647424U,
	10427039104U, 10435425664U, 10443810176U, 10452203648U, 10460589952U,
	10468982144U, 10477369472U, 10485759104U, 10494147712U, 10502533504U,
	10510923392U, 10519313536U, 10527702656U, 10536091264U, 10544478592U,
	10552867712U, 10561255808U, 10569642368U, 10578032768U, 10586423168U,
	10594805632U, 10603200128U, 10611588992U, 10619976064U, 10628361344U,
	10636754048U, 10645143424U, 10653531776U, 10661920384U, 10670307968U,
	10678696832U, 10687086464U, 10695475072U, 10703863168U, 10712246144U,
	10720639616U, 10729026688U, 10737414784U, 10745806208U, 10754190976U,
	10762581376U, 10770971264U, 10779356288U, 10787747456U, 10796135552U,
	10804525184U, 10812915584U, 10821301888U, 10829692288U, 10838078336U,
	10846469248U, 10854858368U, 10863247232U, 10871631488U, 10880023424U,
	10888412032U, 10896799616U, 10905188992U, 10913574016U, 10921964672U,
	10930352768U, 10938742912U, 10947132544U, 10955518592U, 10963909504U,
	10972298368U, 10980687488U, 10989074816U, 10997462912U, 11005851776U,
	11014241152U, 11022627712U, 11031017344U, 11039403904U, 11047793024U,
	11056184704U, 11064570752U, 11072960896U, 11081343872U, 11089737856U,
	11098128256U, 11106514816U, 11114904448U, 11123293568U, 11131680128U,
	11140065152U, 11148458368U, 11156845696U, 11165236864U, 11173624192U,
	11182013824U, 11190402688U, 11198790784U, 11207179136U, 11215568768U,
	11223957376U, 11232345728U, 11240734592U, 11249122688U, 11257511296U,
	11265899648U, 11274285952U, 11282675584U, 11291065472U, 11299452544U,
	11307842432U, 11316231296U, 11324616832U, 11333009024U, 11341395584U,
	11349782656U, 11358172288U, 11366560384U, 11374950016U, 11383339648U,
	11391721856U, 11400117376U, 11408504192U, 11416893568U, 11425283456U,
	11433671552U, 11442061184U, 11450444672U, 11458837888U, 11467226752U,
	11475611776U, 11484003968U, 11492392064U, 11500780672U, 11509169024U,
	11517550976U, 11525944448U, 11534335616U, 11542724224U, 11551111808U,
	11559500672U, 11567890304U, 11576277376U, 11584667008U, 11593056128U,
	11601443456U, 11609830016U, 11618221952U, 11626607488U, 11634995072U,
	11643387776U, 11651775104U, 11660161664U, 11668552576U, 11676940928U,
	11685330304U, 11693718656U, 11702106496U, 11710496128U, 11718882688U,
	11727273088U, 11735660416U, 11744050048U, 11752437376U, 11760824704U,
	11769216128U, 11777604736U, 11785991296U, 11794381952U, 11802770048U,
	11811157888U, 11819548544U, 11827932544U, 11836324736U, 11844713344U,
	11853100928U, 11861486464U, 11869879936U, 11878268032U, 11886656896U,
	11895044992U, 11903433088U, 11911822976U, 11920210816U, 11928600448U,
	11936987264U, 11945375872U, 11953761152U, 11962151296U, 11970543488U,
	11978928512U, 11987320448U, 11995708288U, 12004095104U, 12012486272U,
	12020875136U, 12029255552U, 12037652096U, 12046039168U, 12054429568U,
	12062813824U, 12071206528U, 12079594624U, 12087983744U, 12096371072U,
	12104759936U, 12113147264U, 12121534592U, 12129924992U, 12138314624U,
	12146703232U, 12155091584U, 12163481216U, 12171864704U, 12180255872U,
	12188643968U, 12197034112U, 12205424512U, 12213811328U, 12222199424U,
	12230590336U, 12238977664U, 12247365248U, 12255755392U, 12264143488U,
	12272531584U, 12280920448U, 12289309568U, 12297694592U, 12306086528U,
	12314475392U, 12322865024U, 12331253632U, 12339640448U, 12348029312U,
	12356418944U, 12364805248U, 12373196672U, 12381580928U, 12389969024U,
	12398357632U, 12406750592U, 12415138432U, 12423527552U, 12431916416U,
	12440304512U, 12448692352U, 12457081216U, 12465467776U, 12473859968U,
	12482245504U, 12490636672U, 12499025536U, 12507411584U, 12515801728U,
	12524190592U, 12532577152U, 12540966272U, 12549354368U, 12557743232U,
	12566129536U, 12574523264U, 12582911872U, 12591299456U, 12599688064U,
	12608074624U, 12616463488U, 12624845696U, 12633239936U, 12641631616U,
	12650019968U, 12658407296U, 12666795136U, 12675183232U, 12683574656U,
	12691960192U, 12700350592U, 12708740224U, 12717128576U, 12725515904U,
	12733906816U, 12742295168U, 12750680192U, 12759071872U, 12767460736U,
	12775848832U, 12784236928U, 12792626816U, 12801014656U, 12809404288U,
	12817789312U, 12826181504U, 12834568832U, 12842954624U, 12851345792U,
	12859732352U, 12868122496U, 12876512128U, 12884901248U, 12893289088U,
	12901672832U, 12910067584U, 12918455168U, 12926842496U, 12935232896U,
	12943620736U, 12952009856U, 12960396928U, 12968786816U, 12977176192U,
	12985563776U, 12993951104U, 13002341504U, 13010730368U, 13019115392U,
	13027506304U, 13035895168U, 13044272512U, 13052673152U, 13061062528U,
	13069446272U, 13077838976U, 13086227072U, 13094613632U, 13103000192U,
	13111393664U, 13119782528U, 13128157568U, 13136559232U, 13144945024U,
	13153329536U, 13161724288U, 13170111872U, 13178502784U, 13186884736U,
	13195279744U, 13203667072U, 13212057472U, 13220445824U, 13228832128U,
	13237221248U, 13245610624U, 13254000512U, 13262388352U, 13270777472U,
	13279166336U, 13287553408U, 13295943296U, 13304331904U, 13312719488U,
	13321108096U, 13329494656U, 13337885824U, 13346274944U, 13354663808U,
	13363051136U, 13371439232U, 13379825024U, 13388210816U, 13396605056U,
	13404995456U, 13413380224U, 13421771392U, 13430159744U, 13438546048U,
	13446937216U, 13455326848U, 13463708288U, 13472103808U, 13480492672U,
	13488875648U, 13497269888U, 13505657728U, 13514045312U, 13522435712U,
	13530824576U, 13539210112U, 13547599232U, 13555989376U, 13564379008U,
	13572766336U, 13581154432U, 13589544832U, 13597932928U, 13606320512U,
	13614710656U, 13623097472U, 13631477632U, 13639874944U, 13648264064U,
	13656652928U, 13665041792U, 13673430656U, 13681818496U, 13690207616U,
	13698595712U, 13706982272U, 13715373184U, 13723762048U, 13732150144U,
	13740536704U, 13748926592U, 13757316224U, 13765700992U, 13774090112U,
	13782477952U, 13790869376U, 13799259008U, 13807647872U, 13816036736U,
	13824425344U, 13832814208U, 13841202304U, 13849591424U, 13857978752U,
	13866368896U, 13874754688U, 13883145344U, 13891533184U, 13899919232U,
	13908311168U, 13916692096U, 13925085056U, 13933473152U, 13941866368U,
	13950253696U, 13958643584U, 13967032192U, 13975417216U, 13983807616U,
	13992197504U, 14000582272U, 14008973696U, 14017363072U, 14025752192U,
	14034137984U, 14042528384U, 14050918016U, 14059301504U, 14067691648U,
	14076083584U, 14084470144U, 14092852352U, 14101249664U, 14109635968U,
	14118024832U, 14126407552U, 14134804352U, 14143188608U, 14151577984U,
	14159968384U, 14168357248U, 14176741504U, 14185127296U, 14193521024U,
	14201911424U, 14210301824U, 14218685056U, 14227067264U, 14235467392U,
	14243855488U, 14252243072U, 14260630144U, 14269021568U, 14277409408U,
	14285799296U, 14294187904U, 14302571392U, 14310961792U, 14319353728U,
	14327738752U, 14336130944U, 14344518784U, 14352906368U, 14361296512U,
	14369685376U, 14378071424U, 14386462592U, 14394848128U, 14403230848U,
	14411627392U, 14420013952U, 14428402304U, 14436793472U, 14445181568U,
	14453569664U, 14461959808U, 14470347904U, 14478737024U, 14487122816U,
	14495511424U, 14503901824U, 14512291712U, 14520677504U, 14529064832U,
	14537456768U, 14545845632U, 14554234496U, 14562618496U, 14571011456U,
	14579398784U, 14587789184U, 14596172672U, 14604564608U, 14612953984U,
	14621341312U, 14629724288U, 14638120832U, 14646503296U, 14654897536U,
	14663284864U, 14671675264U, 14680061056U, 14688447616U, 14696835968U,
	14705228416U, 14713616768U, 14722003328U, 14730392192U, 14738784128U,
	14747172736U, 14755561088U, 14763947648U, 14772336512U, 14780725376U,
	14789110144U, 14797499776U, 14805892736U, 14814276992U, 14822670208U,
	14831056256U, 14839444352U, 14847836032U, 14856222848U, 14864612992U,
	14872997504U, 14881388672U, 14889775744U, 14898165376U, 14906553472U,
	14914944896U, 14923329664U, 14931721856U, 14940109696U, 14948497024U,
	14956887424U, 14965276544U, 14973663616U, 14982053248U, 14990439808U,
	14998830976U, 15007216768U, 15015605888U, 15023995264U, 15032385152U,
	15040768384U, 15049154944U, 15057549184U, 15065939072U, 15074328448U,
	15082715008U, 15091104128U, 15099493504U, 15107879296U, 15116269184U,
	15124659584U, 15133042304U, 15141431936U, 15149824384U, 15158214272U,
	15166602368U, 15174991232U, 15183378304U, 15191760512U, 15200154496U,
	15208542592U, 15216931712U, 15225323392U, 15233708416U, 15242098048U,
	15250489216U, 15258875264U, 15267265408U, 15275654528U, 15284043136U,
	15292431488U, 15300819584U, 15309208192U, 15317596544U, 15325986176U,
	15334374784U, 15342763648U, 15351151744U, 15359540608U, 15367929728U,
	15376318336U, 15384706432U, 15393092992U, 15401481856U, 15409869952U,
	15418258816U, 15426649984U, 15435037568U, 15443425664U, 15451815296U,
	15460203392U, 15468589184U, 15476979328U, 15485369216U, 15493755776U,
	15502146944U, 15510534272U, 15518924416U, 15527311232U, 15535699072U,
	15544089472U, 15552478336U, 15560866688U, 15569254528U, 15577642624U,
	15586031488U, 15594419072U, 15602809472U, 15611199104U, 15619586432U,
	15627975296U, 15636364928U, 15644753792U, 15653141888U, 15661529216U,
	15669918848U, 15678305152U, 15686696576U, 15695083136U, 15703474048U,
	15711861632U, 15720251264U, 15728636288U, 15737027456U, 15745417088U,
	15753804928U, 15762194048U, 15770582656U, 15778971008U, 15787358336U,
	15795747712U, 15804132224U, 15812523392U, 15820909696U, 15829300096U,
	15837691264U, 15846071936U, 15854466944U, 15862855808U, 15871244672U,
	15879634816U, 15888020608U, 15896409728U, 15904799104U, 15913185152U,
	15921577088U, 15929966464U, 15938354816U, 15946743424U, 15955129472U,
	15963519872U, 15971907968U, 15980296064U, 15988684928U, 15997073024U,
	16005460864U, 16013851264U, 16022241152U, 16030629248U, 16039012736U,
	16047406976U, 16055794816U, 16064181376U, 16072571264U, 16080957824U,
	16089346688U, 16097737856U, 16106125184U, 16114514816U, 16122904192U,
	16131292544U, 16139678848U, 16148066944U, 16156453504U, 16164839552U,
	16173236096U, 16181623424U, 16190012032U, 16198401152U, 16206790528U,
	16215177344U, 16223567744U, 16231956352U, 16240344704U, 16248731008U,
	16257117824U, 16265504384U, 16273898624U, 16282281856U, 16290668672U,
	16299064192U, 16307449216U, 16315842176U, 16324230016U, 16332613504U,
	16341006464U, 16349394304U, 16357783168U, 16366172288U, 16374561664U,
	16382951296U, 16391337856U, 16399726208U, 16408116352U, 16416505472U,
	16424892032U, 16433282176U, 16441668224U, 16450058624U, 16458448768U,
	16466836864U, 16475224448U, 16483613056U, 16492001408U, 16500391808U,
	16508779648U, 16517166976U, 16525555328U, 16533944192U, 16542330752U,
	16550719616U, 16559110528U, 16567497088U, 16575888512U, 16584274816U,
	16592665472U, 16601051008U, 16609442944U, 16617832064U, 16626218624U,
	16634607488U, 16642996096U, 16651385728U, 16659773824U, 16668163712U,
	16676552576U, 16684938112U, 16693328768U, 16701718144U, 16710095488U,
	16718492288U, 16726883968U, 16735272832U, 16743661184U, 16752049792U,
	16760436608U, 16768827008U, 16777214336U, 16785599104U, 16793992832U,
	16802381696U, 16810768768U, 16819151744U, 16827542656U, 16835934848U,
	16844323712U, 16852711552U, 16861101952U, 16869489536U, 16877876864U,
	16886265728U, 16894653056U, 16903044736U, 16911431296U, 16919821696U,
	16928207488U, 16936592768U, 16944987776U, 16953375616U, 16961763968U,
	16970152832U, 16978540928U, 16986929536U, 16995319168U, 17003704448U,
	17012096896U, 17020481152U, 17028870784U, 17037262208U, 17045649536U,
	17054039936U, 17062426496U, 17070814336U, 17079205504U, 17087592064U,
	17095978112U, 17104369024U, 17112759424U, 17121147776U, 17129536384U,
	17137926016U, 17146314368U, 17154700928U, 17163089792U, 17171480192U,
	17179864192U, 17188256896U, 17196644992U, 17205033856U, 17213423488U,
	17221811072U, 17230198912U, 17238588032U, 17246976896U, 17255360384U,
	17263754624U, 17272143232U, 17280530048U, 17288918912U, 17297309312U,
	17305696384U, 17314085504U, 17322475136U, 17330863744U, 17339252096U,
	17347640192U, 17356026496U, 17364413824U, 17372796544U, 17381190016U,
	17389583488U, 17397972608U, 17406360704U, 17414748544U, 17423135872U,
	17431527296U, 17439915904U, 17448303232U, 17456691584U, 17465081728U,
	17473468288U, 17481857408U, 17490247552U, 17498635904U, 17507022464U,
	17515409024U, 17523801728U, 17532189824U, 17540577664U, 17548966016U,
	17557353344U, 17565741184U, 17574131584U, 17582519168U, 17590907008U,
	17599296128U, 17607687808U, 17616076672U, 17624455808U, 17632852352U,
	17641238656U, 17649630848U, 17658018944U, 17666403968U, 17674794112U,
	17683178368U, 17691573376U, 17699962496U, 17708350592U, 17716739968U,
	17725126528U, 17733517184U, 17741898112U, 17750293888U, 17758673024U,
	17767070336U, 17775458432U, 17783848832U, 17792236928U, 17800625536U,
	17809012352U, 17817402752U, 17825785984U, 17834178944U, 17842563968U,
	17850955648U, 17859344512U, 17867732864U, 17876119424U, 17884511872U,
	17892900224U, 17901287296U, 17909677696U, 17918058112U, 17926451072U,
	17934843776U, 17943230848U, 17951609216U, 17960008576U, 17968397696U,
	17976784256U, 17985175424U, 17993564032U, 18001952128U, 18010339712U,
	18018728576U, 18027116672U, 18035503232U, 18043894144U, 18052283264U,
	18060672128U, 18069056384U, 18077449856U, 18085837184U, 18094225792U,
	18102613376U, 18111004544U, 18119388544U, 18127781248U, 18136170368U,
	18144558976U, 18152947328U, 18161336192U, 18169724288U, 18178108544U,
	18186498944U, 18194886784U, 18203275648U, 18211666048U, 18220048768U,
	18228444544U, 18236833408U, 18245220736U
};


```

这段代码是一个 Mathematica 程序，它定义了一个名为 `GetCacheSizes` 的函数。函数的定义使用了一个称为 `Module` 的 Mathematica 命令，它允许在函数内部定义一些变量和常量。

函数的主要目的是计算一个整数 `n` 的情况下，缓存空间的最大大小。为此，函数使用了一系列变量和常量，如 `DataSetSizeBytesInit`、`MixBytes`、`DataSetGrowth`、`HashBytes` 和 `CacheMultiplier`，这些常量在函数内部被初始化，并在函数外部被使用。

函数实现了一个数据集的生长机制，它会根据缓存空间的大小和计算出的哈希值来选择是否将数据写入缓存。当缓存空间不足时，函数会将当前数据集的一部分写入缓存。函数使用了陈氏算法（又称霍夫曼编码）对数据进行哈希，以允许哈希冲突。


```cpp
// Generated with the following Mathematica Code:

// GetCacheSizes[n_] := Module[{
//         DataSetSizeBytesInit = 2^30,
//         MixBytes = 128,
//         DataSetGrowth = 2^23,
//         HashBytes = 64,
//         CacheMultiplier = 1024,
//         j = 0},
//     Reap[
//       While[j < n,
//        Module[{i = Floor[(DataSetSizeBytesInit + DataSetGrowth * j) / (CacheMultiplier * HashBytes)]},
//         While[! PrimeQ[i], i--];
//         Sow[i*HashBytes]; j++]]]][[2]][[1]]

```

这是一个ASCII码字符串，表示一个28位无符号整数。每个字符表示一个8位二进制数，从0到255。可以将其转换为JavaScript数字，使用+号连接起来，例如：

"0111000001234567891011101111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111


```cpp
const uint64_t cache_sizes[2048] = {
	16776896U, 16907456U, 17039296U, 17170112U, 17301056U, 17432512U, 17563072U,
	17693888U, 17824192U, 17955904U, 18087488U, 18218176U, 18349504U, 18481088U,
	18611392U, 18742336U, 18874304U, 19004224U, 19135936U, 19267264U, 19398208U,
	19529408U, 19660096U, 19791424U, 19922752U, 20053952U, 20184896U, 20315968U,
	20446912U, 20576576U, 20709184U, 20840384U, 20971072U, 21102272U, 21233216U,
	21364544U, 21494848U, 21626816U, 21757376U, 21887552U, 22019392U, 22151104U,
	22281536U, 22412224U, 22543936U, 22675264U, 22806464U, 22935872U, 23068096U,
	23198272U, 23330752U, 23459008U, 23592512U, 23723968U, 23854912U, 23986112U,
	24116672U, 24247616U, 24378688U, 24509504U, 24640832U, 24772544U, 24903488U,
	25034432U, 25165376U, 25296704U, 25427392U, 25558592U, 25690048U, 25820096U,
	25951936U, 26081728U, 26214208U, 26345024U, 26476096U, 26606656U, 26737472U,
	26869184U, 26998208U, 27131584U, 27262528U, 27393728U, 27523904U, 27655744U,
	27786688U, 27917888U, 28049344U, 28179904U, 28311488U, 28441792U, 28573504U,
	28700864U, 28835648U, 28966208U, 29096768U, 29228608U, 29359808U, 29490752U,
	29621824U, 29752256U, 29882816U, 30014912U, 30144448U, 30273728U, 30406976U,
	30538432U, 30670784U, 30799936U, 30932672U, 31063744U, 31195072U, 31325248U,
	31456192U, 31588288U, 31719232U, 31850432U, 31981504U, 32110784U, 32243392U,
	32372672U, 32505664U, 32636608U, 32767808U, 32897344U, 33029824U, 33160768U,
	33289664U, 33423296U, 33554368U, 33683648U, 33816512U, 33947456U, 34076992U,
	34208704U, 34340032U, 34471744U, 34600256U, 34734016U, 34864576U, 34993984U,
	35127104U, 35258176U, 35386688U, 35518528U, 35650624U, 35782336U, 35910976U,
	36044608U, 36175808U, 36305728U, 36436672U, 36568384U, 36699968U, 36830656U,
	36961984U, 37093312U, 37223488U, 37355072U, 37486528U, 37617472U, 37747904U,
	37879232U, 38009792U, 38141888U, 38272448U, 38403392U, 38535104U, 38660672U,
	38795584U, 38925632U, 39059264U, 39190336U, 39320768U, 39452096U, 39581632U,
	39713984U, 39844928U, 39974848U, 40107968U, 40238144U, 40367168U, 40500032U,
	40631744U, 40762816U, 40894144U, 41023552U, 41155904U, 41286208U, 41418304U,
	41547712U, 41680448U, 41811904U, 41942848U, 42073792U, 42204992U, 42334912U,
	42467008U, 42597824U, 42729152U, 42860096U, 42991552U, 43122368U, 43253696U,
	43382848U, 43515712U, 43646912U, 43777088U, 43907648U, 44039104U, 44170432U,
	44302144U, 44433344U, 44564288U, 44694976U, 44825152U, 44956864U, 45088448U,
	45219008U, 45350464U, 45481024U, 45612608U, 45744064U, 45874496U, 46006208U,
	46136768U, 46267712U, 46399424U, 46529344U, 46660672U, 46791488U, 46923328U,
	47053504U, 47185856U, 47316928U, 47447872U, 47579072U, 47710144U, 47839936U,
	47971648U, 48103232U, 48234176U, 48365248U, 48496192U, 48627136U, 48757312U,
	48889664U, 49020736U, 49149248U, 49283008U, 49413824U, 49545152U, 49675712U,
	49807168U, 49938368U, 50069056U, 50200256U, 50331584U, 50462656U, 50593472U,
	50724032U, 50853952U, 50986048U, 51117632U, 51248576U, 51379904U, 51510848U,
	51641792U, 51773248U, 51903296U, 52035136U, 52164032U, 52297664U, 52427968U,
	52557376U, 52690112U, 52821952U, 52952896U, 53081536U, 53213504U, 53344576U,
	53475776U, 53608384U, 53738816U, 53870528U, 54000832U, 54131776U, 54263744U,
	54394688U, 54525248U, 54655936U, 54787904U, 54918592U, 55049152U, 55181248U,
	55312064U, 55442752U, 55574336U, 55705024U, 55836224U, 55967168U, 56097856U,
	56228672U, 56358592U, 56490176U, 56621888U, 56753728U, 56884928U, 57015488U,
	57146816U, 57278272U, 57409216U, 57540416U, 57671104U, 57802432U, 57933632U,
	58064576U, 58195264U, 58326976U, 58457408U, 58588864U, 58720192U, 58849984U,
	58981696U, 59113024U, 59243456U, 59375552U, 59506624U, 59637568U, 59768512U,
	59897792U, 60030016U, 60161984U, 60293056U, 60423872U, 60554432U, 60683968U,
	60817216U, 60948032U, 61079488U, 61209664U, 61341376U, 61471936U, 61602752U,
	61733696U, 61865792U, 61996736U, 62127808U, 62259136U, 62389568U, 62520512U,
	62651584U, 62781632U, 62910784U, 63045056U, 63176128U, 63307072U, 63438656U,
	63569216U, 63700928U, 63831616U, 63960896U, 64093888U, 64225088U, 64355392U,
	64486976U, 64617664U, 64748608U, 64879424U, 65009216U, 65142464U, 65273792U,
	65402816U, 65535424U, 65666752U, 65797696U, 65927744U, 66060224U, 66191296U,
	66321344U, 66453056U, 66584384U, 66715328U, 66846656U, 66977728U, 67108672U,
	67239104U, 67370432U, 67501888U, 67631296U, 67763776U, 67895104U, 68026304U,
	68157248U, 68287936U, 68419264U, 68548288U, 68681408U, 68811968U, 68942912U,
	69074624U, 69205568U, 69337024U, 69467584U, 69599168U, 69729472U, 69861184U,
	69989824U, 70122944U, 70253888U, 70385344U, 70515904U, 70647232U, 70778816U,
	70907968U, 71040832U, 71171648U, 71303104U, 71432512U, 71564992U, 71695168U,
	71826368U, 71958464U, 72089536U, 72219712U, 72350144U, 72482624U, 72613568U,
	72744512U, 72875584U, 73006144U, 73138112U, 73268672U, 73400128U, 73530944U,
	73662272U, 73793344U, 73924544U, 74055104U, 74185792U, 74316992U, 74448832U,
	74579392U, 74710976U, 74841664U, 74972864U, 75102784U, 75233344U, 75364544U,
	75497024U, 75627584U, 75759296U, 75890624U, 76021696U, 76152256U, 76283072U,
	76414144U, 76545856U, 76676672U, 76806976U, 76937792U, 77070016U, 77200832U,
	77331392U, 77462464U, 77593664U, 77725376U, 77856448U, 77987776U, 78118336U,
	78249664U, 78380992U, 78511424U, 78642496U, 78773056U, 78905152U, 79033664U,
	79166656U, 79297472U, 79429568U, 79560512U, 79690816U, 79822784U, 79953472U,
	80084672U, 80214208U, 80346944U, 80477632U, 80608576U, 80740288U, 80870848U,
	81002048U, 81133504U, 81264448U, 81395648U, 81525952U, 81657536U, 81786304U,
	81919808U, 82050112U, 82181312U, 82311616U, 82443968U, 82573376U, 82705984U,
	82835776U, 82967744U, 83096768U, 83230528U, 83359552U, 83491264U, 83622464U,
	83753536U, 83886016U, 84015296U, 84147776U, 84277184U, 84409792U, 84540608U,
	84672064U, 84803008U, 84934336U, 85065152U, 85193792U, 85326784U, 85458496U,
	85589312U, 85721024U, 85851968U, 85982656U, 86112448U, 86244416U, 86370112U,
	86506688U, 86637632U, 86769344U, 86900672U, 87031744U, 87162304U, 87293632U,
	87424576U, 87555392U, 87687104U, 87816896U, 87947968U, 88079168U, 88211264U,
	88341824U, 88473152U, 88603712U, 88735424U, 88862912U, 88996672U, 89128384U,
	89259712U, 89390272U, 89521984U, 89652544U, 89783872U, 89914816U, 90045376U,
	90177088U, 90307904U, 90438848U, 90569152U, 90700096U, 90832832U, 90963776U,
	91093696U, 91223744U, 91356992U, 91486784U, 91618496U, 91749824U, 91880384U,
	92012224U, 92143552U, 92273344U, 92405696U, 92536768U, 92666432U, 92798912U,
	92926016U, 93060544U, 93192128U, 93322816U, 93453632U, 93583936U, 93715136U,
	93845056U, 93977792U, 94109504U, 94240448U, 94371776U, 94501184U, 94632896U,
	94764224U, 94895552U, 95023424U, 95158208U, 95287744U, 95420224U, 95550016U,
	95681216U, 95811904U, 95943872U, 96075328U, 96203584U, 96337856U, 96468544U,
	96599744U, 96731072U, 96860992U, 96992576U, 97124288U, 97254848U, 97385536U,
	97517248U, 97647808U, 97779392U, 97910464U, 98041408U, 98172608U, 98303168U,
	98434496U, 98565568U, 98696768U, 98827328U, 98958784U, 99089728U, 99220928U,
	99352384U, 99482816U, 99614272U, 99745472U, 99876416U, 100007104U,
	100138048U, 100267072U, 100401088U, 100529984U, 100662592U, 100791872U,
	100925248U, 101056064U, 101187392U, 101317952U, 101449408U, 101580608U,
	101711296U, 101841728U, 101973824U, 102104896U, 102235712U, 102366016U,
	102498112U, 102628672U, 102760384U, 102890432U, 103021888U, 103153472U,
	103284032U, 103415744U, 103545152U, 103677248U, 103808576U, 103939648U,
	104070976U, 104201792U, 104332736U, 104462528U, 104594752U, 104725952U,
	104854592U, 104988608U, 105118912U, 105247808U, 105381184U, 105511232U,
	105643072U, 105774784U, 105903296U, 106037056U, 106167872U, 106298944U,
	106429504U, 106561472U, 106691392U, 106822592U, 106954304U, 107085376U,
	107216576U, 107346368U, 107478464U, 107609792U, 107739712U, 107872192U,
	108003136U, 108131392U, 108265408U, 108396224U, 108527168U, 108657344U,
	108789568U, 108920384U, 109049792U, 109182272U, 109312576U, 109444928U,
	109572928U, 109706944U, 109837888U, 109969088U, 110099648U, 110230976U,
	110362432U, 110492992U, 110624704U, 110755264U, 110886208U, 111017408U,
	111148864U, 111279296U, 111410752U, 111541952U, 111673024U, 111803456U,
	111933632U, 112066496U, 112196416U, 112328512U, 112457792U, 112590784U,
	112715968U, 112852672U, 112983616U, 113114944U, 113244224U, 113376448U,
	113505472U, 113639104U, 113770304U, 113901376U, 114031552U, 114163264U,
	114294592U, 114425536U, 114556864U, 114687424U, 114818624U, 114948544U,
	115080512U, 115212224U, 115343296U, 115473472U, 115605184U, 115736128U,
	115867072U, 115997248U, 116128576U, 116260288U, 116391488U, 116522944U,
	116652992U, 116784704U, 116915648U, 117046208U, 117178304U, 117308608U,
	117440192U, 117569728U, 117701824U, 117833024U, 117964096U, 118094656U,
	118225984U, 118357312U, 118489024U, 118617536U, 118749632U, 118882112U,
	119012416U, 119144384U, 119275328U, 119406016U, 119537344U, 119668672U,
	119798464U, 119928896U, 120061376U, 120192832U, 120321728U, 120454336U,
	120584512U, 120716608U, 120848192U, 120979136U, 121109056U, 121241408U,
	121372352U, 121502912U, 121634752U, 121764416U, 121895744U, 122027072U,
	122157632U, 122289088U, 122421184U, 122550592U, 122682944U, 122813888U,
	122945344U, 123075776U, 123207488U, 123338048U, 123468736U, 123600704U,
	123731264U, 123861952U, 123993664U, 124124608U, 124256192U, 124386368U,
	124518208U, 124649024U, 124778048U, 124911296U, 125041088U, 125173696U,
	125303744U, 125432896U, 125566912U, 125696576U, 125829056U, 125958592U,
	126090304U, 126221248U, 126352832U, 126483776U, 126615232U, 126746432U,
	126876608U, 127008704U, 127139392U, 127270336U, 127401152U, 127532224U,
	127663552U, 127794752U, 127925696U, 128055232U, 128188096U, 128319424U,
	128449856U, 128581312U, 128712256U, 128843584U, 128973632U, 129103808U,
	129236288U, 129365696U, 129498944U, 129629888U, 129760832U, 129892288U,
	130023104U, 130154048U, 130283968U, 130416448U, 130547008U, 130678336U,
	130807616U, 130939456U, 131071552U, 131202112U, 131331776U, 131464384U,
	131594048U, 131727296U, 131858368U, 131987392U, 132120256U, 132250816U,
	132382528U, 132513728U, 132644672U, 132774976U, 132905792U, 133038016U,
	133168832U, 133299392U, 133429312U, 133562048U, 133692992U, 133823296U,
	133954624U, 134086336U, 134217152U, 134348608U, 134479808U, 134607296U,
	134741056U, 134872384U, 135002944U, 135134144U, 135265472U, 135396544U,
	135527872U, 135659072U, 135787712U, 135921472U, 136052416U, 136182848U,
	136313792U, 136444864U, 136576448U, 136707904U, 136837952U, 136970048U,
	137099584U, 137232064U, 137363392U, 137494208U, 137625536U, 137755712U,
	137887424U, 138018368U, 138149824U, 138280256U, 138411584U, 138539584U,
	138672832U, 138804928U, 138936128U, 139066688U, 139196864U, 139328704U,
	139460032U, 139590208U, 139721024U, 139852864U, 139984576U, 140115776U,
	140245696U, 140376512U, 140508352U, 140640064U, 140769856U, 140902336U,
	141032768U, 141162688U, 141294016U, 141426496U, 141556544U, 141687488U,
	141819584U, 141949888U, 142080448U, 142212544U, 142342336U, 142474432U,
	142606144U, 142736192U, 142868288U, 142997824U, 143129408U, 143258944U,
	143392448U, 143523136U, 143653696U, 143785024U, 143916992U, 144045632U,
	144177856U, 144309184U, 144440768U, 144570688U, 144701888U, 144832448U,
	144965056U, 145096384U, 145227584U, 145358656U, 145489856U, 145620928U,
	145751488U, 145883072U, 146011456U, 146144704U, 146275264U, 146407232U,
	146538176U, 146668736U, 146800448U, 146931392U, 147062336U, 147193664U,
	147324224U, 147455936U, 147586624U, 147717056U, 147848768U, 147979456U,
	148110784U, 148242368U, 148373312U, 148503232U, 148635584U, 148766144U,
	148897088U, 149028416U, 149159488U, 149290688U, 149420224U, 149551552U,
	149683136U, 149814976U, 149943616U, 150076352U, 150208064U, 150338624U,
	150470464U, 150600256U, 150732224U, 150862784U, 150993088U, 151125952U,
	151254976U, 151388096U, 151519168U, 151649728U, 151778752U, 151911104U,
	152042944U, 152174144U, 152304704U, 152435648U, 152567488U, 152698816U,
	152828992U, 152960576U, 153091648U, 153222976U, 153353792U, 153484096U,
	153616192U, 153747008U, 153878336U, 154008256U, 154139968U, 154270912U,
	154402624U, 154533824U, 154663616U, 154795712U, 154926272U, 155057984U,
	155188928U, 155319872U, 155450816U, 155580608U, 155712064U, 155843392U,
	155971136U, 156106688U, 156237376U, 156367424U, 156499264U, 156630976U,
	156761536U, 156892352U, 157024064U, 157155008U, 157284416U, 157415872U,
	157545536U, 157677248U, 157810496U, 157938112U, 158071744U, 158203328U,
	158334656U, 158464832U, 158596288U, 158727616U, 158858048U, 158988992U,
	159121216U, 159252416U, 159381568U, 159513152U, 159645632U, 159776192U,
	159906496U, 160038464U, 160169536U, 160300352U, 160430656U, 160563008U,
	160693952U, 160822208U, 160956352U, 161086784U, 161217344U, 161349184U,
	161480512U, 161611456U, 161742272U, 161873216U, 162002752U, 162135872U,
	162266432U, 162397888U, 162529216U, 162660032U, 162790976U, 162922048U,
	163052096U, 163184576U, 163314752U, 163446592U, 163577408U, 163707968U,
	163839296U, 163969984U, 164100928U, 164233024U, 164364224U, 164494912U,
	164625856U, 164756672U, 164887616U, 165019072U, 165150016U, 165280064U,
	165412672U, 165543104U, 165674944U, 165805888U, 165936832U, 166067648U,
	166198336U, 166330048U, 166461248U, 166591552U, 166722496U, 166854208U,
	166985408U, 167116736U, 167246656U, 167378368U, 167508416U, 167641024U,
	167771584U, 167903168U, 168034112U, 168164032U, 168295744U, 168427456U,
	168557632U, 168688448U, 168819136U, 168951616U, 169082176U, 169213504U,
	169344832U, 169475648U, 169605952U, 169738048U, 169866304U, 169999552U,
	170131264U, 170262464U, 170393536U, 170524352U, 170655424U, 170782016U,
	170917696U, 171048896U, 171179072U, 171310784U, 171439936U, 171573184U,
	171702976U, 171835072U, 171966272U, 172097216U, 172228288U, 172359232U,
	172489664U, 172621376U, 172747712U, 172883264U, 173014208U, 173144512U,
	173275072U, 173407424U, 173539136U, 173669696U, 173800768U, 173931712U,
	174063424U, 174193472U, 174325696U, 174455744U, 174586816U, 174718912U,
	174849728U, 174977728U, 175109696U, 175242688U, 175374272U, 175504832U,
	175636288U, 175765696U, 175898432U, 176028992U, 176159936U, 176291264U,
	176422592U, 176552512U, 176684864U, 176815424U, 176946496U, 177076544U,
	177209152U, 177340096U, 177470528U, 177600704U, 177731648U, 177864256U,
	177994816U, 178126528U, 178257472U, 178387648U, 178518464U, 178650176U,
	178781888U, 178912064U, 179044288U, 179174848U, 179305024U, 179436736U,
	179568448U, 179698496U, 179830208U, 179960512U, 180092608U, 180223808U,
	180354752U, 180485696U, 180617152U, 180748096U, 180877504U, 181009984U,
	181139264U, 181272512U, 181402688U, 181532608U, 181663168U, 181795136U,
	181926592U, 182057536U, 182190016U, 182320192U, 182451904U, 182582336U,
	182713792U, 182843072U, 182976064U, 183107264U, 183237056U, 183368384U,
	183494848U, 183631424U, 183762752U, 183893824U, 184024768U, 184154816U,
	184286656U, 184417984U, 184548928U, 184680128U, 184810816U, 184941248U,
	185072704U, 185203904U, 185335616U, 185465408U, 185596352U, 185727296U,
	185859904U, 185989696U, 186121664U, 186252992U, 186383552U, 186514112U,
	186645952U, 186777152U, 186907328U, 187037504U, 187170112U, 187301824U,
	187429184U, 187562048U, 187693504U, 187825472U, 187957184U, 188087104U,
	188218304U, 188349376U, 188481344U, 188609728U, 188743616U, 188874304U,
	189005248U, 189136448U, 189265088U, 189396544U, 189528128U, 189660992U,
	189791936U, 189923264U, 190054208U, 190182848U, 190315072U, 190447424U,
	190577984U, 190709312U, 190840768U, 190971328U, 191102656U, 191233472U,
	191364032U, 191495872U, 191626816U, 191758016U, 191888192U, 192020288U,
	192148928U, 192282176U, 192413504U, 192542528U, 192674752U, 192805952U,
	192937792U, 193068608U, 193198912U, 193330496U, 193462208U, 193592384U,
	193723456U, 193854272U, 193985984U, 194116672U, 194247232U, 194379712U,
	194508352U, 194641856U, 194772544U, 194900672U, 195035072U, 195166016U,
	195296704U, 195428032U, 195558592U, 195690304U, 195818176U, 195952576U,
	196083392U, 196214336U, 196345792U, 196476736U, 196607552U, 196739008U,
	196869952U, 197000768U, 197130688U, 197262784U, 197394368U, 197523904U,
	197656384U, 197787584U, 197916608U, 198049472U, 198180544U, 198310208U,
	198442432U, 198573632U, 198705088U, 198834368U, 198967232U, 199097792U,
	199228352U, 199360192U, 199491392U, 199621696U, 199751744U, 199883968U,
	200014016U, 200146624U, 200276672U, 200408128U, 200540096U, 200671168U,
	200801984U, 200933312U, 201062464U, 201194944U, 201326144U, 201457472U,
	201588544U, 201719744U, 201850816U, 201981632U, 202111552U, 202244032U,
	202374464U, 202505152U, 202636352U, 202767808U, 202898368U, 203030336U,
	203159872U, 203292608U, 203423296U, 203553472U, 203685824U, 203816896U,
	203947712U, 204078272U, 204208192U, 204341056U, 204472256U, 204603328U,
	204733888U, 204864448U, 204996544U, 205125568U, 205258304U, 205388864U,
	205517632U, 205650112U, 205782208U, 205913536U, 206044736U, 206176192U,
	206307008U, 206434496U, 206569024U, 206700224U, 206831168U, 206961856U,
	207093056U, 207223616U, 207355328U, 207486784U, 207616832U, 207749056U,
	207879104U, 208010048U, 208141888U, 208273216U, 208404032U, 208534336U,
	208666048U, 208796864U, 208927424U, 209059264U, 209189824U, 209321792U,
	209451584U, 209582656U, 209715136U, 209845568U, 209976896U, 210106432U,
	210239296U, 210370112U, 210501568U, 210630976U, 210763712U, 210894272U,
	211024832U, 211156672U, 211287616U, 211418176U, 211549376U, 211679296U,
	211812032U, 211942592U, 212074432U, 212204864U, 212334016U, 212467648U,
	212597824U, 212727616U, 212860352U, 212991424U, 213120832U, 213253952U,
	213385024U, 213515584U, 213645632U, 213777728U, 213909184U, 214040128U,
	214170688U, 214302656U, 214433728U, 214564544U, 214695232U, 214826048U,
	214956992U, 215089088U, 215219776U, 215350592U, 215482304U, 215613248U,
	215743552U, 215874752U, 216005312U, 216137024U, 216267328U, 216399296U,
	216530752U, 216661696U, 216790592U, 216923968U, 217054528U, 217183168U,
	217316672U, 217448128U, 217579072U, 217709504U, 217838912U, 217972672U,
	218102848U, 218233024U, 218364736U, 218496832U, 218627776U, 218759104U,
	218888896U, 219021248U, 219151936U, 219281728U, 219413056U, 219545024U,
	219675968U, 219807296U, 219938624U, 220069312U, 220200128U, 220331456U,
	220461632U, 220592704U, 220725184U, 220855744U, 220987072U, 221117888U,
	221249216U, 221378368U, 221510336U, 221642048U, 221772736U, 221904832U,
	222031808U, 222166976U, 222297536U, 222428992U, 222559936U, 222690368U,
	222820672U, 222953152U, 223083968U, 223213376U, 223345984U, 223476928U,
	223608512U, 223738688U, 223869376U, 224001472U, 224132672U, 224262848U,
	224394944U, 224524864U, 224657344U, 224788288U, 224919488U, 225050432U,
	225181504U, 225312704U, 225443776U, 225574592U, 225704768U, 225834176U,
	225966784U, 226097216U, 226229824U, 226360384U, 226491712U, 226623424U,
	226754368U, 226885312U, 227015104U, 227147456U, 227278528U, 227409472U,
	227539904U, 227669696U, 227802944U, 227932352U, 228065216U, 228196288U,
	228326464U, 228457792U, 228588736U, 228720064U, 228850112U, 228981056U,
	229113152U, 229243328U, 229375936U, 229505344U, 229636928U, 229769152U,
	229894976U, 230030272U, 230162368U, 230292416U, 230424512U, 230553152U,
	230684864U, 230816704U, 230948416U, 231079616U, 231210944U, 231342016U,
	231472448U, 231603776U, 231733952U, 231866176U, 231996736U, 232127296U,
	232259392U, 232388672U, 232521664U, 232652608U, 232782272U, 232914496U,
	233043904U, 233175616U, 233306816U, 233438528U, 233569984U, 233699776U,
	233830592U, 233962688U, 234092224U, 234221888U, 234353984U, 234485312U,
	234618304U, 234749888U, 234880832U, 235011776U, 235142464U, 235274048U,
	235403456U, 235535936U, 235667392U, 235797568U, 235928768U, 236057152U,
	236190272U, 236322752U, 236453312U, 236583616U, 236715712U, 236846528U,
	236976448U, 237108544U, 237239104U, 237371072U, 237501632U, 237630784U,
	237764416U, 237895232U, 238026688U, 238157632U, 238286912U, 238419392U,
	238548032U, 238681024U, 238812608U, 238941632U, 239075008U, 239206336U,
	239335232U, 239466944U, 239599168U, 239730496U, 239861312U, 239992384U,
	240122816U, 240254656U, 240385856U, 240516928U, 240647872U, 240779072U,
	240909632U, 241040704U, 241171904U, 241302848U, 241433408U, 241565248U,
	241696192U, 241825984U, 241958848U, 242088256U, 242220224U, 242352064U,
	242481856U, 242611648U, 242744896U, 242876224U, 243005632U, 243138496U,
	243268672U, 243400384U, 243531712U, 243662656U, 243793856U, 243924544U,
	244054592U, 244187072U, 244316608U, 244448704U, 244580032U, 244710976U,
	244841536U, 244972864U, 245104448U, 245233984U, 245365312U, 245497792U,
	245628736U, 245759936U, 245889856U, 246021056U, 246152512U, 246284224U,
	246415168U, 246545344U, 246675904U, 246808384U, 246939584U, 247070144U,
	247199552U, 247331648U, 247463872U, 247593536U, 247726016U, 247857088U,
	247987648U, 248116928U, 248249536U, 248380736U, 248512064U, 248643008U,
	248773312U, 248901056U, 249036608U, 249167552U, 249298624U, 249429184U,
	249560512U, 249692096U, 249822784U, 249954112U, 250085312U, 250215488U,
	250345792U, 250478528U, 250608704U, 250739264U, 250870976U, 251002816U,
	251133632U, 251263552U, 251395136U, 251523904U, 251657792U, 251789248U,
	251919424U, 252051392U, 252182464U, 252313408U, 252444224U, 252575552U,
	252706624U, 252836032U, 252968512U, 253099712U, 253227584U, 253361728U,
	253493056U, 253623488U, 253754432U, 253885504U, 254017216U, 254148032U,
	254279488U, 254410432U, 254541376U, 254672576U, 254803264U, 254933824U,
	255065792U, 255196736U, 255326528U, 255458752U, 255589952U, 255721408U,
	255851072U, 255983296U, 256114624U, 256244416U, 256374208U, 256507712U,
	256636096U, 256768832U, 256900544U, 257031616U, 257162176U, 257294272U,
	257424448U, 257555776U, 257686976U, 257818432U, 257949632U, 258079552U,
	258211136U, 258342464U, 258473408U, 258603712U, 258734656U, 258867008U,
	258996544U, 259127744U, 259260224U, 259391296U, 259522112U, 259651904U,
	259784384U, 259915328U, 260045888U, 260175424U, 260308544U, 260438336U,
	260570944U, 260700992U, 260832448U, 260963776U, 261092672U, 261226304U,
	261356864U, 261487936U, 261619648U, 261750592U, 261879872U, 262011968U,
	262143424U, 262274752U, 262404416U, 262537024U, 262667968U, 262799296U,
	262928704U, 263061184U, 263191744U, 263322944U, 263454656U, 263585216U,
	263716672U, 263847872U, 263978944U, 264108608U, 264241088U, 264371648U,
	264501184U, 264632768U, 264764096U, 264895936U, 265024576U, 265158464U,
	265287488U, 265418432U, 265550528U, 265681216U, 265813312U, 265943488U,
	266075968U, 266206144U, 266337728U, 266468032U, 266600384U, 266731072U,
	266862272U, 266993344U, 267124288U, 267255616U, 267386432U, 267516992U,
	267648704U, 267777728U, 267910592U, 268040512U, 268172096U, 268302784U,
	268435264U, 268566208U, 268696256U, 268828096U, 268959296U, 269090368U,
	269221312U, 269352256U, 269482688U, 269614784U, 269745856U, 269876416U,
	270007616U, 270139328U, 270270272U, 270401216U, 270531904U, 270663616U,
	270791744U, 270924736U, 271056832U, 271186112U, 271317184U, 271449536U,
	271580992U, 271711936U, 271843136U, 271973056U, 272105408U, 272236352U,
	272367296U, 272498368U, 272629568U, 272759488U, 272891456U, 273022784U,
	273153856U, 273284672U, 273415616U, 273547072U, 273677632U, 273808448U,
	273937088U, 274071488U, 274200896U, 274332992U, 274463296U, 274595392U,
	274726208U, 274857536U, 274988992U, 275118656U, 275250496U, 275382208U,
	275513024U, 275643968U, 275775296U, 275906368U, 276037184U, 276167872U,
	276297664U, 276429376U, 276560576U, 276692672U, 276822976U, 276955072U,
	277085632U, 277216832U, 277347008U, 277478848U, 277609664U, 277740992U,
	277868608U, 278002624U, 278134336U, 278265536U, 278395328U, 278526784U,
	278657728U, 278789824U, 278921152U, 279052096U, 279182912U, 279313088U,
	279443776U, 279576256U, 279706048U, 279838528U, 279969728U, 280099648U,
	280230976U, 280361408U, 280493632U, 280622528U, 280755392U, 280887104U,
	281018176U, 281147968U, 281278912U, 281411392U, 281542592U, 281673152U,
	281803712U, 281935552U, 282066496U, 282197312U, 282329024U, 282458816U,
	282590272U, 282720832U, 282853184U, 282983744U, 283115072U, 283246144U,
	283377344U, 283508416U, 283639744U, 283770304U, 283901504U, 284032576U,
	284163136U, 284294848U, 284426176U, 284556992U, 284687296U, 284819264U,
	284950208U, 285081536U
};

```

这段代码是一个C语言中的预处理指令，用于判断是否支持C++�� Enabled( enabled )预处理选项。

具体来说，该代码会检查是否定义了`__cplusplus`预处理指令。如果定义了这个预处理指令，那么这段代码就会跳过编译器的中间步骤，直接进入目标代码的执行部分。如果未定义这个预处理指令，那么这段代码就会正常编译，中间步骤会按照从前往后的顺序执行。

简而言之，这段代码的作用是检查是否支持C++预处理选项，如果支持，则跳过编译器的中间步骤，否则正常编译。


```cpp
#ifdef __cplusplus
}
#endif

```