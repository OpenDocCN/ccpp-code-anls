# xmrig源码解析 32

# `src/3rdparty/hwloc/src/cpukinds.c`

这段代码是一个C语言程序，它包括以下几个部分：

1. 包含头文件：private/autogen/config.h，private/private.h，private/debug.h，以及一个自定义的头文件（可能是其他人的或者可能是同一个作者的）。这些头文件包含定义了一些常量和宏的函数。

2. 引入硬件抽象层（HAL）头文件：hwloc.h。这个头文件可能定义了与硬件抽象层相关的函数和变量，用于与硬件抽象层交互。

3. 定义一个名为 "my_main" 的函数，它可能是程序的入口点。

4. 在 "my_main" 函数内部，使用 include 函数引入了前面定义的头文件，包括 private/autogen/config.h，private/private.h，private/debug.h，以及 hwloc.h。

5. 在 "my_main" 函数内部，又使用了 include 函数引入了一个名为 "my_action" 的函数，这个函数可能是用来处理用户输入的。

6. 在 "my_action" 函数内部，使用 include 函数引入了一个名为 "my_system" 的函数，这个函数可能是用来执行系统命令的。

7. 在所有函数内部，使用了 private/debug.h 中的函数 debug，用于输出调试信息。

8. 在所有函数内部，使用了 hwloc.h 中的函数 hal_get_device，用于获取与当前硬件平台相关的设备。

9. 没有其他明显的错误或者敏感信息，因此，这段代码的作用是允许用户在他们的硬件平台上运行一个未知的应用程序，它使用私有作者编写的代码，用于与硬件抽象层和操作系统交互。


```cpp
/*
 * Copyright © 2020-2022 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/debug.h"


/*****************
 * Basics
 */

```

这两函数是在一个名为"hwloc_topology"的结构中进行操作，主要作用是管理CPU绑定和内存分配。

函数1:hwloc_internal_cpukinds_init初始化，将hwloc_topology结构中的cpukinds设置为NULL，同时将nr_cpukinds和nr_cpukinds_allocated设置为0。

函数2:hwloc_internal_cpukinds_destroy销毁，在hwloc_topology结构中遍历所有的cpukinds，然后使用hwloc_bitmap_free函数 free(cpuset) 释放cpu集合，使用hwloc__free_infos函数 free(infos)释放infos，最后使用free函数释放topology中的cpukinds和topology中的内存。

通过这两个函数，可以方便地管理CPU绑定和内存分配，从而更加灵活地配置硬件系统。


```cpp
void
hwloc_internal_cpukinds_init(struct hwloc_topology *topology)
{
  topology->cpukinds = NULL;
  topology->nr_cpukinds = 0;
  topology->nr_cpukinds_allocated = 0;
}

void
hwloc_internal_cpukinds_destroy(struct hwloc_topology *topology)
{
  unsigned i;
  for(i=0; i<topology->nr_cpukinds; i++) {
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
    hwloc_bitmap_free(kind->cpuset);
    hwloc__free_infos(kind->infos, kind->nr_infos);
  }
  free(topology->cpukinds);
  topology->cpukinds = NULL;
  topology->nr_cpukinds = 0;
}

```

This function appears to be a part of a library that manages HWLOC (Highway Workload窃取) information, such as CPU usage and memory usage.

It appears to handle the case where there is a duplicate entry in the HWLOC database for a given set of CPU usage and memory usage data.

If it is unable to find any duplicates, it returns 0. If it finds any duplicates, it wraps the duplicates in a new HWLOC internal CPU kindness object and returns 0.

If it is unable to allocate memory for the new HWLOC internal CPU kindness object, it returns -1.

If it encounter some other issues, it will打印错误信息 and return -1.


```cpp
int
hwloc_internal_cpukinds_dup(hwloc_topology_t new, hwloc_topology_t old)
{
  struct hwloc_tma *tma = new->tma;
  struct hwloc_internal_cpukind_s *kinds;
  unsigned i;

  if (!old->nr_cpukinds)
    return 0;

  kinds = hwloc_tma_malloc(tma, old->nr_cpukinds * sizeof(*kinds));
  if (!kinds)
    return -1;
  new->cpukinds = kinds;
  new->nr_cpukinds = old->nr_cpukinds;
  memcpy(kinds, old->cpukinds, old->nr_cpukinds * sizeof(*kinds));

  for(i=0;i<old->nr_cpukinds; i++) {
    kinds[i].cpuset = hwloc_bitmap_tma_dup(tma, old->cpukinds[i].cpuset);
    if (!kinds[i].cpuset) {
      new->nr_cpukinds = i;
      goto failed;
    }
    if (hwloc__tma_dup_infos(tma,
                             &kinds[i].infos, &kinds[i].nr_infos,
                             old->cpukinds[i].infos, old->cpukinds[i].nr_infos) < 0) {
      assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
      hwloc_bitmap_free(kinds[i].cpuset);
      new->nr_cpukinds = i;
      goto failed;
    }
  }

  return 0;

 failed:
  hwloc_internal_cpukinds_destroy(new);
  return -1;
}

```

这段代码是一个名为 `hwloc_internal_cpukinds_restrict` 的函数，属于 `hwloc_topology_internal` 系列的函数。它用于限制 CPU 核心的映射，以保证在多核处理器上，每个物理核心只能与一个物理核心相对应。

函数的主要作用是检查每个物理核心是否与多个物理核心相对应，如果是，则将其从哈希表中移除并释放内存。移除的过程如下：

1. 对于每个物理核心 `kind`，首先尝试从 `topology.cpukinds` 数组中查找该核心映射的物理核心。
2. 如果找到该物理核心，则检查该物理核心是否在哈希表中，如果是，则执行以下操作：
  a. 从 `topology.cpukinds` 数组中移除该物理核心的映射。
  b. 从 `topology.infos` 数组中移除该物理核心的抽象信息。
  c. 从 `topology.nr_cpukinds` 减 1。
  d. 如果 `topology.nr_cpukinds` 已经减为 0，则执行以下操作：
   i. 从 `topology.cpukinds` 数组中移除该物理核心的映射。
   ii. 从 `topology.nr_infos` 数组中移除该物理核心的抽象信息。
   iii. 从 `topology.nr_cpukinds` 减 1。
   iv. 将 `kind` 增加 1，以继续遍历哈希表。
3. 如果 `topology.nr_cpukinds` 没有减为 0，则执行以下操作：
  a. 从 `topology.cpukinds` 数组中找到所有与物理核心相对应的物理核心，并将它们按秩排序。
  b. 使用 `hwloc_internal_cpukinds_rank` 函数对排序后的物理核心进行排序。
4. 如果哈希表中仍然存在与多个物理核心相对应的物理核心，则执行以下操作：
  a. 从 `topology.cpukinds` 数组中找到所有与物理核心相对应的物理核心，并将它们按秩排序。
  b. 使用 `hwloc_internal_cpukinds_rank` 函数对排序后的物理核心进行排序。
5. 最后，如果哈希表中仍然存在与多个物理核心相对应的物理核心，则执行以下操作：
  a. 从 `topology.cpukinds` 数组中找到所有与物理核心相对应的物理核心，并将它们按秩排序。
  b. 使用 `hwloc_internal_cpukinds_rank` 函数对排序后的物理核心进行排序。
  c. 使用 `hwloc_cpu_get_index` 函数获取与物理核心相对应的物理 CPU ID。
  d. 使用 `hwloc_create_散热器` 函数创建一个新的散热器，并将它与物理 CPU 连接。
  e. 使用 `hwloc_buspower_add` 函数增加与物理核心相对应的物理 CPU 的总功率。

通过这种限制，可以保证在多核处理器上，每个物理核心只与一个物理核心相对应，从而避免了 CPU 核心的性能问题。


```cpp
void
hwloc_internal_cpukinds_restrict(hwloc_topology_t topology)
{
  unsigned i;
  int removed = 0;
  for(i=0; i<topology->nr_cpukinds; i++) {
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
    hwloc_bitmap_and(kind->cpuset, kind->cpuset, hwloc_get_root_obj(topology)->cpuset);
    if (hwloc_bitmap_iszero(kind->cpuset)) {
      hwloc_bitmap_free(kind->cpuset);
      hwloc__free_infos(kind->infos, kind->nr_infos);
      memmove(kind, kind+1, (topology->nr_cpukinds - i - 1)*sizeof(*kind));
      i--;
      topology->nr_cpukinds--;
      removed = 1;
    }
  }
  if (removed)
    hwloc_internal_cpukinds_rank(topology);
}


```

这段代码是一个名为`hwloc__cpukind_check_duplicate_info`的函数，属于`hwloc_internal_cpukind_s`类型的数据结构。它的作用是检查给定的`name`和`value`是否与给定的`kind`中的信息重叠，如果是，则返回1，否则返回0。

该函数的具体实现是：首先定义一个名为`hwloc__cpukind_check_duplicate_info`的函数，它接收两个参数：一个`struct hwloc_internal_cpukind_s`类型的整数`kind`，以及两个字符串参数`name`和`value`。然后，该函数使用一个循环来遍历`kind`中的所有信息，检查给定的`name`和`value`是否与当前信息的重叠。如果是，函数返回1，否则返回0。

该函数可以被看作是一个辅助函数，用于在`hwloc_internal_cpukind_s`结构体中检查信息是否重叠。这个辅助函数可以提高程序的可靠性和可读性，特别是在大型应用程序中，通过减少错误传播和数据冗余。


```cpp
/********************
 * Registering
 */

static __hwloc_inline int
hwloc__cpukind_check_duplicate_info(struct hwloc_internal_cpukind_s *kind,
                                    const char *name, const char *value)
{
  unsigned i;
  for(i=0; i<kind->nr_infos; i++)
    if (!strcmp(kind->infos[i].name, name)
        && !strcmp(kind->infos[i].value, value))
      return 1;
  return 0;
}

```



This is a Java method that takes an array of `HWLOC_CPUKIND_INFO` objects and
returns a topology object. It filters the input bitmap, either
applying a hard mask to all elements or keeping the masked elements, and
returning the topology object.

The `HWLOC_CPUKIND_INFO` class represents an information about a
CPU Kind as specified by the `HWLOC_CPUKIND_FLAG_</TRAILS/><EMPTY|LOGICAL|ORDERED>`
attribute. This class has several fields, including `cpuset`, ` kinds`, and ` infos`,
which correspond to the input bitmap, the kind to compare the masked elements against,
and the information about the masked elements, respectively.

The method `bitmap_andnot` takes an input bitmap and a new bitmap, and
returns the new bitmap. It compares the elements of the input bitmap with the corresponding
elements of the new bitmap, and returns the new bitmap if any elements are present in the
input bitmap but not in the new bitmap.

The method `is_empty` takes an input bitmap and returns true if all elements
of the bitmap are zero, and false otherwise.

The method `compare_with_info` takes an `HWLOC_CPUKIND_INFO` object and
compares the masked elements of the input bitmap with the corresponding information
about the masked elements. It returns true if the masked elements match any of the masked
elements in the `HWLOC_CPUKIND_INFO` object, and false otherwise.

The method `topology_add_cpukinds` adds the number of CPU kinds defined by the `HWLOC_CPUKIND_INFO` objects in the `kinds` array to the topology object.

The method `return_topology` returns the topology object.


```cpp
static __hwloc_inline void
hwloc__cpukind_add_infos(struct hwloc_internal_cpukind_s *kind,
                         const struct hwloc_info_s *infos, unsigned nr_infos)
{
  unsigned i;
  for(i=0; i<nr_infos; i++) {
    if (hwloc__cpukind_check_duplicate_info(kind, infos[i].name, infos[i].value))
      continue;
    hwloc__add_info(&kind->infos, &kind->nr_infos, infos[i].name, infos[i].value);
  }
}

int
hwloc_internal_cpukinds_register(hwloc_topology_t topology, hwloc_cpuset_t cpuset,
                                 int forced_efficiency,
                                 const struct hwloc_info_s *infos, unsigned nr_infos,
                                 unsigned long flags)
{
  struct hwloc_internal_cpukind_s *kinds;
  unsigned i, max, bits, oldnr, newnr;

  if (hwloc_bitmap_iszero(cpuset)) {
    hwloc_bitmap_free(cpuset);
    errno = EINVAL;
    return -1;
  }

  if (flags & ~HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY) {
    errno = EINVAL;
    return -1;
  }

  /* TODO: for now, only windows provides a forced efficiency.
   * if another backend ever provides a conflicting value, the first backend value will be kept.
   * (user-provided values are not an issue, they are meant to overwrite)
   */

  /* If we have N kinds currently, we may need 2N+1 kinds after inserting the new one:
   * - each existing kind may get split into which PUs are in the new kind and which aren't.
   * - some PUs might not have been in any kind yet.
   */
  max = 2 * topology->nr_cpukinds + 1;
  /* Allocate the power-of-two above 2N+1. */
  bits = hwloc_flsl(max-1) + 1;
  max = 1U<<bits;
  /* Allocate 8 minimum to avoid multiple reallocs */
  if (max < 8)
    max = 8;

  /* Create or enlarge the array of kinds if needed */
  kinds = topology->cpukinds;
  if (max > topology->nr_cpukinds_allocated) {
    kinds = realloc(kinds, max * sizeof(*kinds));
    if (!kinds) {
      hwloc_bitmap_free(cpuset);
      return -1;
    }
    memset(&kinds[topology->nr_cpukinds_allocated], 0, (max - topology->nr_cpukinds_allocated) * sizeof(*kinds));
    topology->nr_cpukinds_allocated = max;
    topology->cpukinds = kinds;
  }

  newnr = oldnr = topology->nr_cpukinds;
  for(i=0; i<oldnr; i++) {
    int res = hwloc_bitmap_compare_inclusion(cpuset, kinds[i].cpuset);
    if (res == HWLOC_BITMAP_INTERSECTS || res == HWLOC_BITMAP_INCLUDED) {
      /* new kind with intersection of cpusets and union of infos */
      kinds[newnr].cpuset = hwloc_bitmap_alloc();
      kinds[newnr].efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN;
      kinds[newnr].forced_efficiency = forced_efficiency;
      hwloc_bitmap_and(kinds[newnr].cpuset, cpuset, kinds[i].cpuset);
      hwloc__cpukind_add_infos(&kinds[newnr], kinds[i].infos, kinds[i].nr_infos);
      hwloc__cpukind_add_infos(&kinds[newnr], infos, nr_infos);
      /* remove cpuset PUs from the existing kind that we just split */
      hwloc_bitmap_andnot(kinds[i].cpuset, kinds[i].cpuset, kinds[newnr].cpuset);
      /* clear cpuset PUs that were taken care of */
      hwloc_bitmap_andnot(cpuset, cpuset, kinds[newnr].cpuset);

      newnr++;

    } else if (res == HWLOC_BITMAP_CONTAINS
               || res == HWLOC_BITMAP_EQUAL) {
      /* append new info to existing smaller (or equal) kind */
      hwloc__cpukind_add_infos(&kinds[i], infos, nr_infos);
      if ((flags & HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY)
          || kinds[i].forced_efficiency == HWLOC_CPUKIND_EFFICIENCY_UNKNOWN)
        kinds[i].forced_efficiency = forced_efficiency;
      /* clear cpuset PUs that were taken care of */
      hwloc_bitmap_andnot(cpuset, cpuset, kinds[i].cpuset);

    } else {
      assert(res == HWLOC_BITMAP_DIFFERENT);
      /* nothing to do */
    }

    /* don't compare with anything else if already empty */
    if (hwloc_bitmap_iszero(cpuset))
      break;
  }

  /* add a final kind with remaining PUs if any */
  if (!hwloc_bitmap_iszero(cpuset)) {
    kinds[newnr].cpuset = cpuset;
    kinds[newnr].efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN;
    kinds[newnr].forced_efficiency = forced_efficiency;
    hwloc__cpukind_add_infos(&kinds[newnr], infos, nr_infos);
    newnr++;
  } else {
    hwloc_bitmap_free(cpuset);
  }

  topology->nr_cpukinds = newnr;
  return 0;
}

```

该函数是一个名为 `hwloc_cpukinds_register` 的函数，其作用是注册到 `hwloc_cpukinds` 内部数据结构中。

具体来说，该函数接收四个参数：

1. `hwloc_topology_t`：头信息，用于指定输入的 `hwloc_topology` 结构体。
2. `hwloc_cpuset_t`：输入的 `hwloc_cpuset` 结构体指针。
3. `int` 类型的 `forced_efficiency`：一个整数，表示强制效率。它的默认值是 `HWLOC_CPUKIND_EFFICIENCY_UNKNOWN`。
4. `unsigned long` 类型的 `nr_infos`：一个表示输入 `hwloc_info_s` 数组长度的整数。
5. `struct hwloc_info_s *` 类型的 `infos`：指向 `hwloc_info_s` 的指针。
6. `unsigned long` 类型的 `flags`：一个表示输入 `hwloc_cpukinds` 注册时带上标志的整数。

函数首先检查输入的 `hwloc_topology` 和 `hwloc_cpuset` 是否正确，然后检查 `forced_efficiency` 是否小于 0，如果是，则将其设为 `HWLOC_CPUKIND_EFFICIENCY_UNKNOWN`。

接着，函数调用 `hwloc_internal_cpukinds_register` 函数，并传递输入的 `hwloc_topology`、`hwloc_cpuset`、`forced_efficiency`、`infos` 和 `flags` 参数。其中，函数的最后一个参数 `HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY` 表示在注册过程中，如果 `flags` 为 `1`，则强制执行效率。

接着，函数输出一个整数，表示注册成功返回的值。如果没有错误，函数调用 `hwloc_internal_cpukinds_rank` 函数，作为最后的返回结果。


```cpp
int
hwloc_cpukinds_register(hwloc_topology_t topology, hwloc_cpuset_t _cpuset,
                        int forced_efficiency,
                        unsigned nr_infos, struct hwloc_info_s *infos,
                        unsigned long flags)
{
  hwloc_bitmap_t cpuset;
  int err;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  if (!_cpuset || hwloc_bitmap_iszero(_cpuset)) {
    errno = EINVAL;
    return -1;
  }

  cpuset = hwloc_bitmap_dup(_cpuset);
  if (!cpuset)
    return -1;

  if (forced_efficiency < 0)
    forced_efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN;

  err = hwloc_internal_cpukinds_register(topology, cpuset, forced_efficiency, infos, nr_infos, HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY);
  if (err < 0)
    return err;

  hwloc_internal_cpukinds_rank(topology);
  return 0;
}


```



该代码是一个名为 `hwloc__cpukinds_check_duplicate_rankings` 的函数，属于 `hwloc_topology` 结构体的成员函数。

该函数的作用是检查给定的 `hwloc_topology` 结构体中 `cpukinds` 成员是否存在 duplicate ranking。如果存在 duplicate ranking，函数将返回 -1，否则返回 0。

函数的具体实现包括以下几个步骤：

1. 遍历 `topology->nr_cpukinds` 次，每次对于每个 `cpukinds` 结构体，使用两个循环遍历 `topology->nr_cpukinds` 次，检查当前结构体中的 `ranking_value` 是否与另一个结构体中的 `ranking_value` 相等。

2. 如果找到任何一个结构体 `cpukinds` 的 `ranking_value` 在另一个结构体中存在，函数将返回 -1。

3. 如果没有找到 duplicate ranking，函数返回 0，表示 `cpukinds` 结构体没有 duplicate ranking。

该函数可以作为一个简单的工具函数，用于检查给定的 `hwloc_topology` 结构体中 `cpukinds` 成员是否存在 duplicate ranking，有助于确保在 rank 相关的任务中，不会存在因多个进程具有相同的偏好设置而引发的问题。


```cpp
/*********************
 * Ranking
 */

static int
hwloc__cpukinds_check_duplicate_rankings(struct hwloc_topology *topology)
{
  unsigned i,j;
  for(i=0; i<topology->nr_cpukinds; i++)
    for(j=i+1; j<topology->nr_cpukinds; j++)
      if (topology->cpukinds[i].ranking_value == topology->cpukinds[j].ranking_value)
        /* if any duplicate, fail */
        return -1;
  return 0;
}

```

这段代码是一个名为 `hwloc__cpukinds_try_rank_by_forced_efficiency` 的函数，属于 `hwloc_topology` 结构体的成员函数。

它的作用是尝试对 `hwloc_cpukinds` 结构体中的所有 `forced_efficiency` 成员进行升序排序，将其分为不同的 CPU 核心，并返回最高排序次的 CPU 核心编号，如果没有排序成功，则返回 -1。具体实现包括以下几个步骤：

1. 初始化变量 `i` 并赋值为 0，用于计数排序过程中的迭代次数。
2. 遍历 `topology->nr_cpukinds` 成员，对于每个成员，执行以下操作：
	1. 如果当前成员的 `forced_efficiency` 属性为 `HWLOC_CPUKIND_EFFICIENCY_UNKNOWN`，则输出一条日志信息并返回 `-1`，表示无法确定该成员的排名。
	2. 否则，将当前成员的 `ranking_value` 属性设置为当前成员的 `forced_efficiency` 属性，以设置该成员所属的 CPU 核心的排名。
3. 在循环结束后，调用一个名为 `hwloc__cpukinds_check_duplicate_rankings` 的函数，检查是否有成员重复排序的情况。如果没有，则该函数应该返回 `0`。如果有成员重复排序，则该函数应该返回 `-1`，表明排序失败。


```cpp
static int
hwloc__cpukinds_try_rank_by_forced_efficiency(struct hwloc_topology *topology)
{
  unsigned i;

  hwloc_debug("Trying to rank cpukinds by forced efficiency...\n");
  for(i=0; i<topology->nr_cpukinds; i++) {
    if (topology->cpukinds[i].forced_efficiency == HWLOC_CPUKIND_EFFICIENCY_UNKNOWN)
      /* if any unknown, fail */
      return -1;
    topology->cpukinds[i].ranking_value = topology->cpukinds[i].forced_efficiency;
  }

  return hwloc__cpukinds_check_duplicate_rankings(topology);
}

```

这一段代码是用来设置基频的。

首先，将最大频数设为1，因为频数不能超过2000。

然后设置基频的最大值和基准频，这样就可以计算出其他频数。

接着，遍历所有可用的CPU架构，对于每一个架构，获取该架构下所有信息的数量。

接下来，遍历这些信息，找到所有与频率相关的信息，并记录下来。

如果是基准频，直接将最大值赋为它。

如果是最大频，将最大值赋为它。

如果是架构类型为IntelAtom，那么对于每一个核心，将intel_core_type设置为1，并将基准频赋为它。

如果是架构类型为IntelCore，那么对于每一个核心，将intel_core_type设置为2，并将基准频赋为它。

最后，根据这些设置，将基频的最大值、基准频、最大频和是否有基准频设置为1，如果有则是0。


```cpp
struct hwloc_cpukinds_info_summary {
  int have_max_freq;
  int have_base_freq;
  int have_intel_core_type;
  struct hwloc_cpukind_info_summary {
    unsigned intel_core_type; /* 1 for atom, 2 for core */
    unsigned max_freq, base_freq; /* MHz, hence < 100000 */
  } * summaries;
};

static void
hwloc__cpukinds_summarize_info(struct hwloc_topology *topology,
                               struct hwloc_cpukinds_info_summary *summary)
{
  unsigned i, j;

  summary->have_max_freq = 1;
  summary->have_base_freq = 1;
  summary->have_intel_core_type = 1;

  for(i=0; i<topology->nr_cpukinds; i++) {
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
    for(j=0; j<kind->nr_infos; j++) {
      struct hwloc_info_s *info = &kind->infos[j];
      if (!strcmp(info->name, "FrequencyMaxMHz")) {
        summary->summaries[i].max_freq = atoi(info->value);
      } else if (!strcmp(info->name, "FrequencyBaseMHz")) {
        summary->summaries[i].base_freq = atoi(info->value);
      } else if (!strcmp(info->name, "CoreType")) {
        if (!strcmp(info->value, "IntelAtom"))
          summary->summaries[i].intel_core_type = 1;
        else if (!strcmp(info->value, "IntelCore"))
          summary->summaries[i].intel_core_type = 2;
      }
    }
    hwloc_debug("cpukind #%u has intel_core_type %u max_freq %u base_freq %u\n",
                i, summary->summaries[i].intel_core_type,
                summary->summaries[i].max_freq, summary->summaries[i].base_freq);
    if (!summary->summaries[i].base_freq)
      summary->have_base_freq = 0;
    if (!summary->summaries[i].max_freq)
      summary->have_max_freq = 0;
    if (!summary->summaries[i].intel_core_type)
      summary->have_intel_core_type = 0;
  }
}

```

This function appears to be responsible for ranking the cpukinds based on one of the given heuristics. It does this by first storing the `summary` object that contains information about the base frequencies and maximum frequencies of each item. It then, for each kind of cpukind, compares the ranker value to the maximum frequency of that kind, and if the maximum frequency is greater than or equal to the sum of the base frequencies for that kind, it updates the ranker value to be the maximum frequency. If the maximum frequency is less than the base frequencies, it returns -1.

It should be noted that this function also checks for duplicate rankings, and if it finds any, it returns an error.


```cpp
enum hwloc_cpukinds_ranking {
  HWLOC_CPUKINDS_RANKING_DEFAULT, /* forced + frequency on ARM, forced + coretype_frequency otherwise */
  HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY, /* default without forced */
  HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY,
  HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY, /* either coretype or frequency or both */
  HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY_STRICT, /* both coretype and frequency are required */
  HWLOC_CPUKINDS_RANKING_CORETYPE,
  HWLOC_CPUKINDS_RANKING_FREQUENCY,
  HWLOC_CPUKINDS_RANKING_FREQUENCY_MAX,
  HWLOC_CPUKINDS_RANKING_FREQUENCY_BASE,
  HWLOC_CPUKINDS_RANKING_NONE
};

static int
hwloc__cpukinds_try_rank_by_info(struct hwloc_topology *topology,
                                 enum hwloc_cpukinds_ranking heuristics,
                                 struct hwloc_cpukinds_info_summary *summary)
{
  unsigned i;

  if (HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY_STRICT == heuristics) {
    hwloc_debug("Trying to rank cpukinds by coretype+frequency_strict...\n");
    /* we need intel_core_type AND (base or max freq) for all kinds */
    if (!summary->have_intel_core_type
        || (!summary->have_max_freq && !summary->have_base_freq))
      return -1;
    /* rank first by coretype (Core>>Atom) then by frequency, base if available, max otherwise */
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      if (summary->have_base_freq)
        kind->ranking_value = (summary->summaries[i].intel_core_type << 20) + summary->summaries[i].base_freq;
      else
        kind->ranking_value = (summary->summaries[i].intel_core_type << 20) + summary->summaries[i].max_freq;
    }

  } else if (HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY == heuristics) {
    hwloc_debug("Trying to rank cpukinds by coretype+frequency...\n");
    /* we need intel_core_type OR (base or max freq) for all kinds */
    if (!summary->have_intel_core_type
        && (!summary->have_max_freq && !summary->have_base_freq))
      return -1;
    /* rank first by coretype (Core>>Atom) then by frequency, base if available, max otherwise */
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      if (summary->have_base_freq)
        kind->ranking_value = (summary->summaries[i].intel_core_type << 20) + summary->summaries[i].base_freq;
      else
        kind->ranking_value = (summary->summaries[i].intel_core_type << 20) + summary->summaries[i].max_freq;
    }

  } else if (HWLOC_CPUKINDS_RANKING_CORETYPE == heuristics) {
    hwloc_debug("Trying to rank cpukinds by coretype...\n");
    /* we need intel_core_type */
    if (!summary->have_intel_core_type)
      return -1;
    /* rank by coretype (Core>>Atom) */
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      kind->ranking_value = (summary->summaries[i].intel_core_type << 20);
    }

  } else if (HWLOC_CPUKINDS_RANKING_FREQUENCY == heuristics) {
    hwloc_debug("Trying to rank cpukinds by frequency...\n");
    /* we need base or max freq for all kinds */
    if (!summary->have_max_freq && !summary->have_base_freq)
      return -1;
    /* rank first by frequency, base if available, max otherwise */
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      if (summary->have_base_freq)
        kind->ranking_value = summary->summaries[i].base_freq;
      else
        kind->ranking_value = summary->summaries[i].max_freq;
    }

  } else if (HWLOC_CPUKINDS_RANKING_FREQUENCY_MAX == heuristics) {
    hwloc_debug("Trying to rank cpukinds by frequency max...\n");
    /* we need max freq for all kinds */
    if (!summary->have_max_freq)
      return -1;
    /* rank first by frequency, base if available, max otherwise */
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      kind->ranking_value = summary->summaries[i].max_freq;
    }

  } else if (HWLOC_CPUKINDS_RANKING_FREQUENCY_BASE == heuristics) {
    hwloc_debug("Trying to rank cpukinds by frequency base...\n");
    /* we need max freq for all kinds */
    if (!summary->have_base_freq)
      return -1;
    /* rank first by frequency, base if available, max otherwise */
    for(i=0; i<topology->nr_cpukinds; i++) {
      struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[i];
      kind->ranking_value = summary->summaries[i].base_freq;
    }

  } else assert(0);

  return hwloc__cpukinds_check_duplicate_rankings(topology);
}

```



这段代码定义了一个名为 `hwloc__cpukinds_compare_ranking_values` 的函数，用于比较两个 `hwloc_internal_cpukind_s` 结构体的大小关系，并返回一个整数表示它们的大小关系。该函数的参数为两个指向相同结构体的指针，返回值为 -1、0 或 1 中的一个。

此外，还定义了一个名为 `hwloc__cpukinds_finalize_ranking` 的函数，该函数接受一个 `hwloc_topology` 结构体作为参数，对其中的 `cpukinds` 成员进行排序，并定义了每个 `cpukind` 结构体成员的 `efficiency` 成员。


```cpp
static int hwloc__cpukinds_compare_ranking_values(const void *_a, const void *_b)
{
  const struct hwloc_internal_cpukind_s *a = _a;
  const struct hwloc_internal_cpukind_s *b = _b;
  uint64_t arv = a->ranking_value;
  uint64_t brv = b->ranking_value;
  return arv < brv ? -1 : arv > brv ? 1 : 0;
}

/* this function requires ranking values to be unique */
static void
hwloc__cpukinds_finalize_ranking(struct hwloc_topology *topology)
{
  unsigned i;
  /* sort */
  qsort(topology->cpukinds, topology->nr_cpukinds, sizeof(*topology->cpukinds), hwloc__cpukinds_compare_ranking_values);
  /* define our own efficiency between 0 and N-1 */
  for(i=0; i<topology->nr_cpukinds; i++)
    topology->cpukinds[i].efficiency = i;
}

```

This code is using a custom ranking strategy for the `HWLOC_CPUKINDS_RANKING` hint. The strategy is expected to be efficient, and the efficiency ranking is being forced.

If the `HWLOC_CPUKINDS_RANKING_NONE` hint is specified, the code will use the default heuristics and force the rankings to be inefficient.

If the `HWLOC_CPUKINDS_RANKING_CUSTOM_ORDER_WINNING_ASCENDING` hint is specified, the code will use the custom ordering strategy and force the rankings to be efficient in ascending order.

If the `HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY` hint is specified, the code will use the custom ranking strategy and force the rankings to be efficient.

If the heuristics are `HWLOC_CPUKINDS_RANKING_NONE`, `HWLOC_CPUKINDS_RANKING_CUSTOM_ORDER_WINNING_ASCENDING`, or `HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY`, the code will rank the cpukinds based on the specified hint.

If the `HWLOC_CPUKINDS_RANKING_NONE` hint is specified and the `HWLOC_CPUKINDS_RANKING_CUSTOM_ORDER_WINNING_ASCENDING` hint is also specified, the code will use the custom ordering strategy and force the rankings to be inefficient. This is a counter-intuitive behavior, as the custom ordering strategy is expected to be efficient.

If the `HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY` hint is specified and the `HWLOC_CPUKINDS_RANKING_NONE` hint is also specified, the code will use the custom ranking strategy and force the rankings to be inefficient. This is a behavior that is not expected by the API designers.

If the heuristics are `HWLOC_CPUKINDS_RANKING_NONE`, `HWLOC_CPUKINDS_RANKING_CUSTOM_ORDER_WINNING_ASCENDING`, or `HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY`, the code will rank the cpukinds based on the specified hint.


```cpp
int
hwloc_internal_cpukinds_rank(struct hwloc_topology *topology)
{
  enum hwloc_cpukinds_ranking heuristics;
  char *env;
  unsigned i;
  int err;

  if (!topology->nr_cpukinds)
    return 0;

  if (topology->nr_cpukinds == 1) {
    topology->cpukinds[0].efficiency = 0;
    return 0;
  }

  heuristics = HWLOC_CPUKINDS_RANKING_DEFAULT;
  env = getenv("HWLOC_CPUKINDS_RANKING");
  if (env) {
    if (!strcmp(env, "default"))
      heuristics = HWLOC_CPUKINDS_RANKING_DEFAULT;
    else if (!strcmp(env, "none"))
      heuristics = HWLOC_CPUKINDS_RANKING_NONE;
    else if (!strcmp(env, "coretype+frequency"))
      heuristics = HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY;
    else if (!strcmp(env, "coretype+frequency_strict"))
      heuristics = HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY_STRICT;
    else if (!strcmp(env, "coretype"))
      heuristics = HWLOC_CPUKINDS_RANKING_CORETYPE;
    else if (!strcmp(env, "frequency"))
      heuristics = HWLOC_CPUKINDS_RANKING_FREQUENCY;
    else if (!strcmp(env, "frequency_max"))
      heuristics = HWLOC_CPUKINDS_RANKING_FREQUENCY_MAX;
    else if (!strcmp(env, "frequency_base"))
      heuristics = HWLOC_CPUKINDS_RANKING_FREQUENCY_BASE;
    else if (!strcmp(env, "forced_efficiency"))
      heuristics = HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY;
    else if (!strcmp(env, "no_forced_efficiency"))
      heuristics = HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY;
    else if (HWLOC_SHOW_CRITICAL_ERRORS())
      fprintf(stderr, "hwloc: Failed to recognize HWLOC_CPUKINDS_RANKING value %s\n", env);
  }

  if (heuristics == HWLOC_CPUKINDS_RANKING_DEFAULT
      || heuristics == HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY) {
    /* default is forced_efficiency first */
    struct hwloc_cpukinds_info_summary summary;

    if (heuristics == HWLOC_CPUKINDS_RANKING_DEFAULT)
      hwloc_debug("Using default ranking strategy...\n");
    else
      hwloc_debug("Using custom ranking strategy from HWLOC_CPUKINDS_RANKING=%s\n", env);

    if (heuristics != HWLOC_CPUKINDS_RANKING_NO_FORCED_EFFICIENCY) {
      err = hwloc__cpukinds_try_rank_by_forced_efficiency(topology);
      if (!err)
        goto ready;
    }

    summary.summaries = calloc(topology->nr_cpukinds, sizeof(*summary.summaries));
    if (!summary.summaries)
      goto failed;
    hwloc__cpukinds_summarize_info(topology, &summary);

    err = hwloc__cpukinds_try_rank_by_info(topology, HWLOC_CPUKINDS_RANKING_CORETYPE_FREQUENCY, &summary);
    free(summary.summaries);
    if (!err)
      goto ready;

  } else if (heuristics == HWLOC_CPUKINDS_RANKING_FORCED_EFFICIENCY) {
    hwloc_debug("Using custom ranking strategy from HWLOC_CPUKINDS_RANKING=%s\n", env);

    err = hwloc__cpukinds_try_rank_by_forced_efficiency(topology);
    if (!err)
      goto ready;

  } else if (heuristics != HWLOC_CPUKINDS_RANKING_NONE) {
    /* custom heuristics */
    struct hwloc_cpukinds_info_summary summary;

    hwloc_debug("Using custom ranking strategy from HWLOC_CPUKINDS_RANKING=%s\n", env);

    summary.summaries = calloc(topology->nr_cpukinds, sizeof(*summary.summaries));
    if (!summary.summaries)
      goto failed;
    hwloc__cpukinds_summarize_info(topology, &summary);

    err = hwloc__cpukinds_try_rank_by_info(topology, heuristics, &summary);
    free(summary.summaries);
    if (!err)
      goto ready;
  }

 failed:
  /* failed to rank, clear efficiencies */
  for(i=0; i<topology->nr_cpukinds; i++)
    topology->cpukinds[i].efficiency = HWLOC_CPUKIND_EFFICIENCY_UNKNOWN;
  hwloc_debug("Failed to rank cpukinds.\n\n");
  return 0;

 ready:
  for(i=0; i<topology->nr_cpukinds; i++)
    hwloc_debug("cpukind #%u got ranking value %llu\n", i, (unsigned long long) topology->cpukinds[i].ranking_value);
  hwloc__cpukinds_finalize_ranking(topology);
```

这段代码的作用是检查给定的topology结构中，cpukinds数组中每个元素的效率是否为i，其中i从0到topology.nr_cpukinds-1。如果i不等于topology.nr_cpukinds中的任何元素，代码会通过errno为eINVAL错误并返回-1。如果i等于topology.nr_cpukinds中的某个元素，代码会输出调试信息并返回0。

这里使用了if语句，如果给定的flags为真，则代码会执行if语句块内的内容。if语句块中有一个输出语句，但是输出信息不会被输出，只会在调试器中显示。


```cpp
#ifdef HWLOC_DEBUG
  for(i=0; i<topology->nr_cpukinds; i++)
    assert(topology->cpukinds[i].efficiency == (int) i);
#endif
  hwloc_debug("\n");
  return 0;
}


/*****************
 * Consulting
 */

int
hwloc_cpukinds_get_nr(hwloc_topology_t topology, unsigned long flags)
{
  if (flags) {
    errno = EINVAL;
    return -1;
  }

  return topology->nr_cpukinds;
}

```

该函数的作用是获取一个 hwloc_topology 结构中的 cpukind 元素的 efficiency 属性的值，并返回该属性的值。

具体来说，函数接受一个 hwloc_topology 结构、一个 id、一个 cpuset、一个指向 efficiency 属性的指针、一个指向 infos 的指针和一个整数标志 flags。函数首先检查 flags 是否为真，如果是，则执行 hwloc_topology_cpukinds_get_info 函数，否则直接返回 -1。

如果 id 不大于 topology 结构中 cpukinds 数组的数量，则函数成功检查 id 是否存在于 cpukinds 数组中，否则返回 ENOENT。

函数使用 hwloc_bitmap_copy 函数将 cpuset 复制到 kind 指向的 cpukind 元素的cpuset中，然后根据 efficiency 属性获取其效率值。

最后，函数检查 nr_infosp 和 infosp 是否为真，如果是，则将 kind 指向的 cpukind 的 nr_infos 和 infos 复制到 *nr_infosp 和 *infosp 指向的变量中，否则不进行复制。函数最终返回 0，表示成功执行。


```cpp
int
hwloc_cpukinds_get_info(hwloc_topology_t topology,
                        unsigned id,
                        hwloc_bitmap_t cpuset,
                        int *efficiencyp,
                        unsigned *nr_infosp, struct hwloc_info_s **infosp,
                        unsigned long flags)
{
  struct hwloc_internal_cpukind_s *kind;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  if (id >= topology->nr_cpukinds) {
    errno = ENOENT;
    return -1;
  }

  kind = &topology->cpukinds[id];

  if (cpuset)
    hwloc_bitmap_copy(cpuset, kind->cpuset);

  if (efficiencyp)
    *efficiencyp = kind->efficiency;

  if (nr_infosp && infosp) {
    *nr_infosp = kind->nr_infos;
    *infosp = kind->infos;
  }
  return 0;
}

```

该函数的作用是获取给定 CPU 集的 hwloc_cpukinds 结构体，它属于 hwloc_topology_t 结构体的一个函数。

具体来说，该函数接收三个参数：

1. topology：是一个 hwloc_topology_t 结构体，它用于表示要操作的 topology 结构。
2. cpuset：是一个 hwloc_const_bitmap_t 结构体，它用于表示要操作的 CPU 集合。
3. flags：是一个unsigned long 类型的参数，用于表示要操作的指令集。如果 flags 的值为 0，那么函数的行为与不带参数的行为相同。

函数内部首先检查给定的 CPU 集合是否为空，如果是，则输出 EINVAL，表示出错。然后，函数遍历给定的 CPU 集合，检查它是否与 cpuset 中的所有元素都相等，如果是，那么函数返回该 CPU 集合的索引号。否则，函数会尝试使用索引号所对应的 hwloc_cpukinds 结构体，并检查给定的 CPU 集合是否与该结构体中的元素相等。如果两个操作都成功，则函数不会输出错误。否则，函数会输出 EXDEV，表示出错。最后，函数返 ENOENT，表示出错。


```cpp
int
hwloc_cpukinds_get_by_cpuset(hwloc_topology_t topology,
                             hwloc_const_bitmap_t cpuset,
                             unsigned long flags)
{
  unsigned id;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  if (!cpuset || hwloc_bitmap_iszero(cpuset)) {
    errno = EINVAL;
    return -1;
  }

  for(id=0; id<topology->nr_cpukinds; id++) {
    struct hwloc_internal_cpukind_s *kind = &topology->cpukinds[id];
    int res = hwloc_bitmap_compare_inclusion(cpuset, kind->cpuset);
    if (res == HWLOC_BITMAP_EQUAL || res == HWLOC_BITMAP_INCLUDED) {
      return (int) id;
    } else if (res == HWLOC_BITMAP_INTERSECTS || res == HWLOC_BITMAP_CONTAINS) {
      errno = EXDEV;
      return -1;
    }
  }

  errno = ENOENT;
  return -1;
}

```

# `src/3rdparty/hwloc/src/diff.c`

这段代码是一个 C 语言函数，名为 `hwloc_topology_diff_destroy`，属于 `hwloc_topology_diff` 系列的函数。它的作用是处理一个 `hwloc_topology_diff_t` 类型的数据结构，这个数据结构包含了不同的 topology 差异，如差异对象、属性等。

函数首先定义了一个 `next` 变量，用于存储下一个 topology 差异对象，然后进入一个 while 循环，这个循环用于遍历当前 topology 差异对象的所有属性。

在循环中，首先定义了一个 `switch` 语句，用于根据当前属性的类型来决定是否遍历。如果是默认的 `HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME`，那么就表示所有属性都不需要进行释放，直接跳过。如果是 `HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO`，那么需要释放对象的名称和旧值，并为新值赋值。如果是 `HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME`，需要释放名称和旧值，并为新值赋值。如果是 `HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO`，需要释放对象的名称和旧值，并为新值赋值。

在循环结束后，首先释放当前的 topology 差异对象，然后将 `next` 指向下一个 topology 差异对象，继续循环。这样就形成了一个完整的 topology 差异链，最后返回 0，表示操作成功。


```cpp
/*
 * Copyright © 2013-2022 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "private/private.h"
#include "private/misc.h"

int hwloc_topology_diff_destroy(hwloc_topology_diff_t diff)
{
	hwloc_topology_diff_t next;
	while (diff) {
		next = diff->generic.next;
		switch (diff->generic.type) {
		default:
			break;
		case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR:
			switch (diff->obj_attr.diff.generic.type) {
			default:
				break;
			case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME:
			case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO:
				free(diff->obj_attr.diff.string.name);
				free(diff->obj_attr.diff.string.oldvalue);
				free(diff->obj_attr.diff.string.newvalue);
				break;
			}
			break;
		}
		free(diff);
		diff = next;
	}
	return 0;
}

```



这段代码定义了一个名为 `hwloc_append_diff` 的函数，用于计算并追加 `hwloc_topology_diff_t` 结构体之间的差异。

该函数的主要作用是将 `newdiff` 结构体追加到 `firstdiffp` 和 `lastdiffp` 指向的差异链中。如果 `firstdiffp` 或 `lastdiffp` 指向的差异链为空，则将 `newdiff` 追加到差异链的末尾。如果 `firstdiffp` 或 `lastdiffp` 指向的差异链不为空，则更新 `lastdiffp` 指向为 `newdiff`，并将 `newdiff` 追加到差异链的末尾。最后，将 `newdiff` 的 `generic.next` 成员设置为 `NULL`，以确保差异链中的新节点不会被进一步处理。


```cpp
/************************
 * Computing diffs
 */

static void hwloc_append_diff(hwloc_topology_diff_t newdiff,
			      hwloc_topology_diff_t *firstdiffp,
			      hwloc_topology_diff_t *lastdiffp)
{
	if (*firstdiffp)
		(*lastdiffp)->generic.next = newdiff;
	else
		*firstdiffp = newdiff;
	*lastdiffp = newdiff;
	newdiff->generic.next = NULL;
}

```

该函数的作用是返回一个名为 "hwloc_append_diff_too_complex" 的函数的返回值，它是 "hwloc_topology_diff_append" 的别名。

该函数的主要作用是创建一个新的 "hwloc_topology_diff_t" 类型的变量 "newdiff"，并将其加入到 "firstdiffp" 和 "lastdiffp" 指向的变量中。

具体来说，函数首先检查内存是否可用，如果不可用，则返回 -1。接着，函数创建一个名为 "newdiff" 的新 "hwloc_topology_diff_t" 类型的变量，并设置其 "too_complex" 成员的值为 HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX，同时设置其 "obj_depth" 和 "obj_index" 成员的值。接着，函数调用 "hwloc_append_diff" 函数，将 "newdiff" 和 "firstdiffp" 和 "lastdiffp" 指向的值进行比较并添加到感兴趣的 "diff" 中。最后，函数返回 0，表示成功调用 "hwloc_topology_diff_append" 函数。


```cpp
static int hwloc_append_diff_too_complex(hwloc_obj_t obj1,
					 hwloc_topology_diff_t *firstdiffp,
					 hwloc_topology_diff_t *lastdiffp)
{
	hwloc_topology_diff_t newdiff;
	newdiff = malloc(sizeof(*newdiff));
	if (!newdiff)
		return -1;

	newdiff->too_complex.type = HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX;
	newdiff->too_complex.obj_depth = obj1->depth;
	newdiff->too_complex.obj_index = obj1->logical_index;
	hwloc_append_diff(newdiff, firstdiffp, lastdiffp);
	return 0;
}

```

这段代码定义了一个名为 `hwloc_append_diff_obj_attr_string` 的函数，它接受四个参数：

1. `obj`：要分析的 HWLOC 对象的引用。
2. `type`：表示要分析的 HWLOC 对象的属性类型，例如 `HWLOC_TOPOLOGY_DIFF_OBJ_ATTR`。
3. `name`：表示要分析的 HWLOC 对象的属性名称，如果已给出，则覆盖现有名称。
4. `oldvalue`：表示要比较的 HWLOC 对象的旧值，如果已给出，则覆盖现有值。
5. `newvalue`：表示要分析的 HWLOC 对象的属性新值，如果已给出，则覆盖现有值。
6. `firstdiffp`：指向第一个差异指针的 HWLOC 指针。
7. `lastdiffp`：指向最后一个差异指针的 HWLOC 指针。

函数的作用是将传入的 HWLOC 对象与已知的差异对象进行比较，并输出差异信息。比较结果将被添加到新的差异对象中，并返回差异对象的计数器。


```cpp
static int hwloc_append_diff_obj_attr_string(hwloc_obj_t obj,
					     hwloc_topology_diff_obj_attr_type_t type,
					     const char *name,
					     const char *oldvalue,
					     const char *newvalue,
					     hwloc_topology_diff_t *firstdiffp,
					     hwloc_topology_diff_t *lastdiffp)
{
	hwloc_topology_diff_t newdiff;
	newdiff = malloc(sizeof(*newdiff));
	if (!newdiff)
		return -1;

	newdiff->obj_attr.type = HWLOC_TOPOLOGY_DIFF_OBJ_ATTR;
	newdiff->obj_attr.obj_depth = obj->depth;
	newdiff->obj_attr.obj_index = obj->logical_index;
	newdiff->obj_attr.diff.string.type = type;
	newdiff->obj_attr.diff.string.name = name ? strdup(name) : NULL;
	newdiff->obj_attr.diff.string.oldvalue = oldvalue ? strdup(oldvalue) : NULL;
	newdiff->obj_attr.diff.string.newvalue = newvalue ? strdup(newvalue) : NULL;
	hwloc_append_diff(newdiff, firstdiffp, lastdiffp);
	return 0;
}

```

该函数的作用是返回一个名为 `hwloc_topology_diff_obj_attr_uint64` 的函数指针，用于将 `hwloc_topology_diff_obj_attr_uint64` 类型的数据作为第一个或最后一个 diff 对象添加到指定的 `hwloc_topology_diff_t` 结构中。

具体来说，该函数接收四个参数：

- `obj`：要添加 diff 对象的 `hwloc_obj_t` 类型的对象。
- `type`:diff 对象的属性类型，可以是 `HWLOC_TOPOLOGY_DIFF_OBJ_ATTR`。
- `idx`：属性索引，用于标识要添加的 diff 对象在对象中的位置。
- `oldvalue`：属性旧值，可以是 `hwloc_uint64_t` 类型的任意值。
- `newvalue`：属性新值，可以是 `hwloc_uint64_t` 类型的任意值。
- `firstdiffp`：指向第一个 diff 对象的指针。
- `lastdiffp`：指向最后一个 diff 对象的指针，如果有。

函数内部首先根据传入的参数检查是否能够成功分配内存空间 for newdiff，然后初始化 newdiff 的各个字段。接着，使用 hwloc_append_diff 函数将 newdiff 和 firstdiffp、lastdiffp 指向的 diff 对象添加到链表中。最后，函数返回 0，表示成功添加 diff 对象。


```cpp
static int hwloc_append_diff_obj_attr_uint64(hwloc_obj_t obj,
					     hwloc_topology_diff_obj_attr_type_t type,
					     hwloc_uint64_t idx,
					     hwloc_uint64_t oldvalue,
					     hwloc_uint64_t newvalue,
					     hwloc_topology_diff_t *firstdiffp,
					     hwloc_topology_diff_t *lastdiffp)
{
	hwloc_topology_diff_t newdiff;
	newdiff = malloc(sizeof(*newdiff));
	if (!newdiff)
		return -1;

	newdiff->obj_attr.type = HWLOC_TOPOLOGY_DIFF_OBJ_ATTR;
	newdiff->obj_attr.obj_depth = obj->depth;
	newdiff->obj_attr.obj_index = obj->logical_index;
	newdiff->obj_attr.diff.uint64.type = type;
	newdiff->obj_attr.diff.uint64.index = idx;
	newdiff->obj_attr.diff.uint64.oldvalue = oldvalue;
	newdiff->obj_attr.diff.uint64.newvalue = newvalue;
	hwloc_append_diff(newdiff, firstdiffp, lastdiffp);
	return 0;
}

```

这段代码定义了一个名为 `hwloc_diff_trees` 的函数，它接受四个参数：`topo1`、`obj1`、`topo2` 和 `obj2`，以及一个表示 diff 类型的整数 `flags`，返回一个指向第一个差异节点后者的指针 `firstdiffp` 和指向最后一个差异节点后者的指针 `lastdiffp`。

首先，函数条件检查 `obj1` 和 `obj2` 的深度是否相等，如果不相等，函数跳转到 `out_too_complex` 标签处。然后，函数条件检查 `obj1` 和 `obj2` 的类型是否相等，如果不相等，函数跳转到 `out_too_complex` 标签处。接下来，函数条件检查 `obj1` 和 `obj2` 的子类型是否相同，如果不相同，函数跳转到 `out_too_complex` 标签处。最后，函数条件检查 `obj1` 和 `obj2` 的操作系统索引是否相同，如果不相同，函数可能会有一些影响，但目前不清楚具体会怎么样。

函数的具体实现并未在给定的代码中显示，因此无法提供更多细节。


```cpp
static int
hwloc_diff_trees(hwloc_topology_t topo1, hwloc_obj_t obj1,
		 hwloc_topology_t topo2, hwloc_obj_t obj2,
		 unsigned flags,
		 hwloc_topology_diff_t *firstdiffp, hwloc_topology_diff_t *lastdiffp)
{
	unsigned i;
	int err;
	hwloc_obj_t child1, child2;

	if (obj1->depth != obj2->depth)
		goto out_too_complex;

	if (obj1->type != obj2->type)
		goto out_too_complex;
	if ((!obj1->subtype) != (!obj2->subtype)
	    || (obj1->subtype && strcmp(obj1->subtype, obj2->subtype)))
		goto out_too_complex;

	if (obj1->os_index != obj2->os_index)
		/* we could allow different os_index for non-PU non-NUMAnode objects
		 * but it's likely useless anyway */
		goto out_too_complex;

```

This is a C function that compares two HWLOC trees. The function takes in two objects, obj1 and obj2, and compares their children and their respective components, such as I/O devices and miscellaneous devices. It returns 0 if the comparison was successful, or an error if one or more of the objects were not found or if there were any discrepancies in the children of the objects. The function uses the hwloc\_diff\_trees function to compare the trees and the firstdiffp and lastdiffp functions to track changes in the children.


```cpp
#define _SETS_DIFFERENT(_set1, _set2) \
 (   ( !(_set1) != !(_set2) ) \
  || ( (_set1) && !hwloc_bitmap_isequal(_set1, _set2) ) )
#define SETS_DIFFERENT(_set, _obj1, _obj2) _SETS_DIFFERENT((_obj1)->_set, (_obj2)->_set)
	if (SETS_DIFFERENT(cpuset, obj1, obj2)
	    || SETS_DIFFERENT(complete_cpuset, obj1, obj2)
	    || SETS_DIFFERENT(nodeset, obj1, obj2)
	    || SETS_DIFFERENT(complete_nodeset, obj1, obj2))
		goto out_too_complex;

	/* no need to check logical_index, sibling_rank, symmetric_subtree,
	 * the parents did it */

	/* gp_index don't have to be strictly identical */

	if ((!obj1->name) != (!obj2->name)
	    || (obj1->name && strcmp(obj1->name, obj2->name))) {
		err = hwloc_append_diff_obj_attr_string(obj1,
						       HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME,
						       NULL,
						       obj1->name,
						       obj2->name,
						       firstdiffp, lastdiffp);
		if (err < 0)
			return err;
	}

	/* type-specific attrs */
	switch (obj1->type) {
	default:
		break;
	case HWLOC_OBJ_NUMANODE:
		if (obj1->attr->numanode.local_memory != obj2->attr->numanode.local_memory) {
			err = hwloc_append_diff_obj_attr_uint64(obj1,
								HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE,
								0,
								obj1->attr->numanode.local_memory,
								obj2->attr->numanode.local_memory,
								firstdiffp, lastdiffp);
			if (err < 0)
				return err;
		}
		/* ignore memory page_types */
		break;
	case HWLOC_OBJ_L1CACHE:
	case HWLOC_OBJ_L2CACHE:
	case HWLOC_OBJ_L3CACHE:
	case HWLOC_OBJ_L4CACHE:
	case HWLOC_OBJ_L5CACHE:
	case HWLOC_OBJ_L1ICACHE:
	case HWLOC_OBJ_L2ICACHE:
	case HWLOC_OBJ_L3ICACHE:
		if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->cache)))
			goto out_too_complex;
		break;
	case HWLOC_OBJ_GROUP:
		if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->group)))
			goto out_too_complex;
		break;
	case HWLOC_OBJ_PCI_DEVICE:
		if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->pcidev)))
			goto out_too_complex;
		break;
	case HWLOC_OBJ_BRIDGE:
		if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->bridge)))
			goto out_too_complex;
		break;
	case HWLOC_OBJ_OS_DEVICE:
		if (memcmp(obj1->attr, obj2->attr, sizeof(obj1->attr->osdev)))
			goto out_too_complex;
		break;
	}

	/* infos */
	if (obj1->infos_count != obj2->infos_count)
		goto out_too_complex;
	for(i=0; i<obj1->infos_count; i++) {
		struct hwloc_info_s *info1 = &obj1->infos[i], *info2 = &obj2->infos[i];
		if (strcmp(info1->name, info2->name))
			goto out_too_complex;
		if (strcmp(info1->value, info2->value)) {
			err = hwloc_append_diff_obj_attr_string(obj1,
								HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO,
								info1->name,
								info1->value,
								info2->value,
								firstdiffp, lastdiffp);
			if (err < 0)
				return err;
		}
	}

	/* ignore userdata */

	/* children */
	for(child1 = obj1->first_child, child2 = obj2->first_child;
	    child1 != NULL && child2 != NULL;
	    child1 = child1->next_sibling, child2 = child2->next_sibling) {
		err = hwloc_diff_trees(topo1, child1,
				       topo2, child2,
				       flags,
				       firstdiffp, lastdiffp);
		if (err < 0)
			return err;
	}
	if (child1 || child2)
		goto out_too_complex;

	/* memory children */
	for(child1 = obj1->memory_first_child, child2 = obj2->memory_first_child;
	    child1 != NULL && child2 != NULL;
	    child1 = child1->next_sibling, child2 = child2->next_sibling) {
		err = hwloc_diff_trees(topo1, child1,
				       topo2, child2,
				       flags,
				       firstdiffp, lastdiffp);
		if (err < 0)
			return err;
	}
	if (child1 || child2)
		goto out_too_complex;

	/* I/O children */
	for(child1 = obj1->io_first_child, child2 = obj2->io_first_child;
	    child1 != NULL && child2 != NULL;
	    child1 = child1->next_sibling, child2 = child2->next_sibling) {
		err = hwloc_diff_trees(topo1, child1,
				       topo2, child2,
				       flags,
				       firstdiffp, lastdiffp);
		if (err < 0)
			return err;
	}
	if (child1 || child2)
		goto out_too_complex;

	/* misc children */
	for(child1 = obj1->misc_first_child, child2 = obj2->misc_first_child;
	    child1 != NULL && child2 != NULL;
	    child1 = child1->next_sibling, child2 = child2->next_sibling) {
		err = hwloc_diff_trees(topo1, child1,
				       topo2, child2,
				       flags,
				       firstdiffp, lastdiffp);
		if (err < 0)
			return err;
	}
	if (child1 || child2)
		goto out_too_complex;

	return 0;

```

This function appears to be a part of a larger software build process, and it is used to convert an array of memory attributes from a HWLOC format to aOComplex format.

The function takes in two arguments: a pointer to an instance of the `HWLOC_MEMATTR_INITIATOR_T` struct, and a pointer to an instance of the `HWLOC_MEMATTR_S` struct.

The function first checks if the first memory attribute's flags indicate that it needs to be initialized. If it does, the function then iterates through the `nr_initiators` field of the first memory attribute, and for each initializer, it compares the value of the memory attribute to the value of the corresponding memory attribute in the second memory attribute. If the values are not equal or the initializer type is different, the function jumps to the `roottoocomplex` function.

If the first memory attribute does not need to be initialized, or if the second memory attribute does not have the same initializer as the first memory attribute, the function also jumps to the `roottoocomplex` function.

The function also checks if the second memory attribute has any initiators. If it does, the function compares the value of the first memory attribute's `initiator_value` to the value of the `noinitiator_value` of the second memory attribute. If the values are not equal, the function jumps to the `roottoocomplex` function.

Overall, this function appears to be a utility function that helps to convert memory attributes between different formats.


```cpp
out_too_complex:
	hwloc_append_diff_too_complex(obj1, firstdiffp, lastdiffp);
	return 0;
}

int hwloc_topology_diff_build(hwloc_topology_t topo1,
			      hwloc_topology_t topo2,
			      unsigned long flags,
			      hwloc_topology_diff_t *diffp)
{
	hwloc_topology_diff_t lastdiff, tmpdiff;
	struct hwloc_internal_distances_s *dist1, *dist2;
	unsigned i;
	int err;

	if (!topo1->is_loaded || !topo2->is_loaded) {
	  errno = EINVAL;
	  return -1;
	}

	if (flags != 0) {
		errno = EINVAL;
		return -1;
	}

	*diffp = NULL;
	err = hwloc_diff_trees(topo1, hwloc_get_root_obj(topo1),
			       topo2, hwloc_get_root_obj(topo2),
			       flags,
			       diffp, &lastdiff);
	if (!err) {
		tmpdiff = *diffp;
		while (tmpdiff) {
			if (tmpdiff->generic.type == HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX) {
				err = 1;
				break;
			}
			tmpdiff = tmpdiff->generic.next;
		}
	}

	if (!err) {
		if (SETS_DIFFERENT(allowed_cpuset, topo1, topo2)
		    || SETS_DIFFERENT(allowed_nodeset, topo1, topo2))
                  goto roottoocomplex;
	}

	if (!err) {
		/* distances */
		hwloc_internal_distances_refresh(topo1);
		hwloc_internal_distances_refresh(topo2);
		dist1 = topo1->first_dist;
		dist2 = topo2->first_dist;
		while (dist1 || dist2) {
			if (!!dist1 != !!dist2)
                          goto roottoocomplex;
			if (dist1->unique_type != dist2->unique_type
			    || dist1->different_types || dist2->different_types /* too lazy to support this case */
			    || dist1->nbobjs != dist2->nbobjs
			    || dist1->kind != dist2->kind
			    || memcmp(dist1->values, dist2->values, dist1->nbobjs * dist1->nbobjs * sizeof(*dist1->values)))
                          goto roottoocomplex;
			for(i=0; i<dist1->nbobjs; i++)
				/* gp_index isn't enforced above. so compare logical_index instead, which is enforced. requires distances refresh() above */
				if (dist1->objs[i]->logical_index != dist2->objs[i]->logical_index)
                                  goto roottoocomplex;
			dist1 = dist1->next;
			dist2 = dist2->next;
		}
	}

        if (!err) {
          /* memattrs */
          hwloc_internal_memattrs_refresh(topo1);
          hwloc_internal_memattrs_refresh(topo2);
          if (topo1->nr_memattrs != topo2->nr_memattrs)
            goto roottoocomplex;
          for(i=0; i<topo1->nr_memattrs; i++) {
            struct hwloc_internal_memattr_s *imattr1 = &topo1->memattrs[i], *imattr2 = &topo2->memattrs[i];
            unsigned j;
           if (strcmp(imattr1->name, imattr2->name)
                || imattr1->flags != imattr2->flags
                || imattr1->nr_targets != imattr2->nr_targets)
              goto roottoocomplex;
            if (i == HWLOC_MEMATTR_ID_CAPACITY
                || i == HWLOC_MEMATTR_ID_LOCALITY)
              /* no need to check virtual attributes, there were refreshed from other topology attributes, checked above */
              continue;
            for(j=0; j<imattr1->nr_targets; j++) {
              struct hwloc_internal_memattr_target_s *imtg1 = &imattr1->targets[j], *imtg2 = &imattr2->targets[j];
              if (imtg1->type != imtg2->type)
                goto roottoocomplex;
              if (imtg1->obj->logical_index != imtg2->obj->logical_index)
                goto roottoocomplex;
              if (imattr1->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
                unsigned k;
                for(k=0; k<imtg1->nr_initiators; k++) {
                  struct hwloc_internal_memattr_initiator_s *imi1 = &imtg1->initiators[k], *imi2 = &imtg2->initiators[k];
                  if (imi1->value != imi2->value
                      || imi1->initiator.type != imi2->initiator.type)
                    goto roottoocomplex;
                  if (imi1->initiator.type == HWLOC_LOCATION_TYPE_CPUSET) {
                    if (!hwloc_bitmap_isequal(imi1->initiator.location.cpuset, imi2->initiator.location.cpuset))
                      goto roottoocomplex;
                  } else if (imi1->initiator.type == HWLOC_LOCATION_TYPE_OBJECT) {
                    if (imi1->initiator.location.object.type != imi2->initiator.location.object.type)
                      goto roottoocomplex;
                    if (imi1->initiator.location.object.obj->logical_index != imi2->initiator.location.object.obj->logical_index)
                      goto roottoocomplex;
                  } else {
                    assert(0);
                  }
                }
              } else {
                if (imtg1->noinitiator_value != imtg2->noinitiator_value)
                  goto roottoocomplex;
              }
            }
          }
        }

	return err;

 roottoocomplex:
  hwloc_append_diff_too_complex(hwloc_get_root_obj(topo1), diffp, &lastdiff);
  return 1;
}

```

This function appears to be part of the OrphanC device driver for the Linux distribution. It is used to compare a file to a specified topology and to update the object attributes of the file to reflect any changes.

The function takes a single parameter of type `HWLOC_OBJECT_TYPE`, which specifies the object type to compare. It is then called with a maximum of three arguments:

* First, a null pointer `NULL`, which is passed to the `obj_sys_diff_io` function to avoid a null pointer dereference error.
* Second, a pointer to a `HWLOC_INFO_S结构`, which is the object information structure for the file being compared.
* Third, a pointer to a `HWLOC_DIF_OBJECT_ATTR_NAME_S结构`, which is the structure containing the name of the attribute being compared.

The function works by comparing the object information structure to the specified topology, and updating the corresponding object attribute to reflect any changes. It does this by first attempting to match the object information structure to the specified topology. If a match is found, it compares the `name` attribute of the object to the specified `name` attribute, and the `value` attribute to the specified `oldvalue` or `newvalue`.

If the comparison is successful, the function frees the old or new value, and if the object does not exist, it creates a new object with the new value.

If the comparison is not successful, the function returns `-1`, indicating an error.

The function also handles the case where the object type is not specified properly, in which case it returns `-1`.


```cpp
/********************
 * Applying diffs
 */

static int
hwloc_apply_diff_one(hwloc_topology_t topology,
		     hwloc_topology_diff_t diff,
		     unsigned long flags)
{
	int reverse = !!(flags & HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE);

	switch (diff->generic.type) {
	case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR: {
		struct hwloc_topology_diff_obj_attr_s *obj_attr = &diff->obj_attr;
		hwloc_obj_t obj = hwloc_get_obj_by_depth(topology, obj_attr->obj_depth, obj_attr->obj_index);
		if (!obj)
			return -1;

		switch (obj_attr->diff.generic.type) {
		case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE: {
			hwloc_obj_t tmpobj;
			hwloc_uint64_t oldvalue = reverse ? obj_attr->diff.uint64.newvalue : obj_attr->diff.uint64.oldvalue;
			hwloc_uint64_t newvalue = reverse ? obj_attr->diff.uint64.oldvalue : obj_attr->diff.uint64.newvalue;
			hwloc_uint64_t valuediff = newvalue - oldvalue;
			if (obj->type != HWLOC_OBJ_NUMANODE)
				return -1;
			if (obj->attr->numanode.local_memory != oldvalue)
				return -1;
			obj->attr->numanode.local_memory = newvalue;
			tmpobj = obj;
			while (tmpobj) {
				tmpobj->total_memory += valuediff;
				tmpobj = tmpobj->parent;
			}
			break;
		}
		case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME: {
			const char *oldvalue = reverse ? obj_attr->diff.string.newvalue : obj_attr->diff.string.oldvalue;
			const char *newvalue = reverse ? obj_attr->diff.string.oldvalue : obj_attr->diff.string.newvalue;
			if (!obj->name || strcmp(obj->name, oldvalue))
				return -1;
			free(obj->name);
			obj->name = strdup(newvalue);
			break;
		}
		case HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO: {
			const char *name = obj_attr->diff.string.name;
			const char *oldvalue = reverse ? obj_attr->diff.string.newvalue : obj_attr->diff.string.oldvalue;
			const char *newvalue = reverse ? obj_attr->diff.string.oldvalue : obj_attr->diff.string.newvalue;
			unsigned i;
			int found = 0;
			for(i=0; i<obj->infos_count; i++) {
				struct hwloc_info_s *info = &obj->infos[i];
				if (!strcmp(info->name, name)
				    && !strcmp(info->value, oldvalue)) {
					free(info->value);
					info->value = strdup(newvalue);
					found = 1;
					break;
				}
			}
			if (!found)
				return -1;
			break;
		}
		default:
			return -1;
		}

		break;
	}
	default:
		return -1;
	}

	return 0;
}

```

这段代码的作用是检查并应用差分 topology 到另一个 topology。如果两个 topology 之间存在差异，则会递归地应用这些差异。

具体来说，代码首先检查两个输入的 topology 是否已经加载到内存中。如果不存在，则会发生错误并返回 -1。然后，代码检查 topology 是否已经采用过共享内存，如果是，则会发生错误并返回 -1。

接下来，代码检查要应用的标志是否包括“HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE”选项。如果不是，则会发生错误并返回 -1。如果是，则不会执行任何操作。

接下来，代码将创建两个临时变量，用于存储比较和应用差异。然后，代码遍历差分 topology 的每个元素，并尝试将其应用到目标 topology。如果任何错误发生，代码会通过“goto cancel”语句跳转到取消异常处。

最后，代码返回 0，表示成功应用差异。


```cpp
int hwloc_topology_diff_apply(hwloc_topology_t topology,
			      hwloc_topology_diff_t diff,
			      unsigned long flags)
{
	hwloc_topology_diff_t tmpdiff, tmpdiff2;
	int err, nr;

	if (!topology->is_loaded) {
	  errno = EINVAL;
	  return -1;
	}
	if (topology->adopted_shmem_addr) {
	  errno = EPERM;
	  return -1;
	}

	if (flags & ~HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE) {
		errno = EINVAL;
		return -1;
	}

	tmpdiff = diff;
	nr = 0;
	while (tmpdiff) {
		nr++;
		err = hwloc_apply_diff_one(topology, tmpdiff, flags);
		if (err < 0)
			goto cancel;
		tmpdiff = tmpdiff->generic.next;
	}
	return 0;

```

这段代码是一个 C 语言函数，名为 "cancel"，其作用是取消并停止对某个 HWLOC 配置文件的差异比较。

具体来说，该函数首先创建一个名为 "tmpdiff2" 的临时差异对象，然后将差异对象 "diff" 复制到该临时差异对象中。接下来，该函数在一个 while 循环中进行操作，只要临时差异对象 "tmpdiff" 还与原始差异对象 "diff2" 不相等，就会执行该循环体内的代码。

循环体内的代码首先使用 hwloc_apply_diff_one 函数将差异对象 "diff" 应用到指定 topology 架构中，并使用 flags 参数中的 HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE flag 设置为真，表示反向应用变更。然后，代码从差异对象 "diff" 的 generic.next 成员处获取下一个差异对象，并将其复制到 "tmpdiff" 中。

最后，如果 "tmpdiff" 与 "diff2" 不相等，函数返回一个非零的错误码，并表示无法找到第一个无法应用的元素。否则，函数返回 0，即表示成功取消了差异。


```cpp
cancel:
	tmpdiff2 = tmpdiff;
	tmpdiff = diff;
	while (tmpdiff != tmpdiff2) {
		hwloc_apply_diff_one(topology, tmpdiff, flags ^ HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE);
		tmpdiff = tmpdiff->generic.next;
	}
	errno = EINVAL;
	return -nr; /* return the index (starting at 1) of the first element that couldn't be applied */
}

```