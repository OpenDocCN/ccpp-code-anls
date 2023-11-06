# xmrig源码解析 34

# `src/3rdparty/hwloc/src/memattrs.c`

这段代码是一个 C 语言源文件，其中包含了某些必要的头部文件、函数定义以及注释。让我们逐个部分地解释它的作用。

1. 包含头文件：

```cppc
#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/debug.h"
```

这四个头文件包含了配置文件相关的定义和函数，还包含了 hwloc 和 private.h，很明显，这是用于支持硬件自动编号（HWLAB）的自动编号工具链。头文件中定义了一些函数，例如：`autogenerate_config()`，`init_hwl()`，`set_hwl_private()` 等。

2. 定义全局变量：

```cppc
#define MAX_CONFIG_LINE_NUM 100
```

这个定义了一个常量 MAX_CONFIG_LINE_NUM，用于表示配置文件行数的一个最大值，用于支持一行一行地输出配置文件。

3. 引入 HWLOC：

```cppc
#include "hwloc.h"
```

这是引入 hwloc 库以便在构建自动编号配置文件时进行硬件定位。

4. 引入 Private：

```cppc
#include "private/private.h"
```

这是引入 Private 库以便在配置文件中设置私有参数。

5. 引入 Debug：

```cppc
#include "private/debug.h"
```

这是引入 Debug 库以便在配置文件输出过程中捕获和处理调试信息。

6. 定义文本：

```cppc
static HWL_CPU_TABLE *config_pwl;
static HWL_SYSTEM_TABLE *system_table;
static HWL_CLASS_TABLE *class_table;
```

这里定义了三个变量，它们都是 HWL 框架中定义的表格类型，用于在自动编号配置文件中设置硬件。

7. 初始化函数：

```cppc
void init_hwl() {
   // 在这里可以设置一些初始化配置，例如：配置文件目录、格式等。
   // 但是，根据上下文，我们可能并不需要设置任何初始化配置。
}
```

初始化函数，但是根据上下文，我们可能并不需要设置任何初始化配置。这个函数通常包含一些通用的设置，例如指定输出目录，格式等。

8. 设置 HWL_CPU_TABLE：

```cppc
static HWL_CPU_TABLE *config_pwl;
```

这里定义了一个 HWL_CPU_TABLE 类型的变量 config_pwl，用于在自动编号配置文件中设置硬件。这个变量稍后会在初始化函数中进行初始化。

9. 设置 HWL_SYSTEM_TABLE：

```cppc
static HWL_SYSTEM_TABLE *system_table;
```

这里定义了一个 HWL_SYSTEM_TABLE 类型的变量 system_table，用于在自动编号配置文件中设置硬件。这个变量稍后会在初始化函数中进行初始化。

10. 设置 HWL_CLASS_TABLE：

```cppc
static HWL_CLASS_TABLE *class_table;
```

这里定义了一个 HWL_CLASS_TABLE 类型的变量 class_table，用于在自动编号配置文件中设置硬件。这个变量稍后会在初始化函数中进行初始化。

11. 定义 MAX_CONFIG_LINE_NUM：

```cppc
#define MAX_CONFIG_LINE_NUM 100
```

这个定义了一个常量 MAX_CONFIG_LINE_NUM，用于表示配置文件行数的一个最大值，用于支持一行一行地输出配置文件。

12. 定义文本：

```cppc
static HWL_CPU_TABLE *config_pwl;
static HWL_SYSTEM_TABLE *system_table;
static HWL_CLASS_TABLE *class_table;
static HWL_API_TABLE hwl_api_table;
static HWL_HOST_TABLE *hwl_host_table;
static HWL_RUN_TABLE *hwl_run_table;
static HWL_TASK_TABLE *hwl_task_table;
static HWL_FIFO_TABLE *hwl_fifo_table;
```

这里定义了一系列 HWL 框架中的表格类型，以及一个常量 MAX_CONFIG_LINE_NUM，用于在自动编号配置文件中设置硬件。这个变量稍后会在初始化函数中进行初始化。


```cpp
/*
 * Copyright © 2020-2022 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/debug.h"


/*****************************
 * Attributes
 */

```

该代码定义了两个名为`hwloc_memattr_id_t`和`hwloc_obj_t`的枚举类型和两个函数，`hwloc__memattr_get_convenience_value`用于获取指定内存属性的便利值，`hwloc_internal_memattrs_init`用于初始化内部内存属性。以下是代码的作用：

1. `hwloc__memattr_get_convenience_value`函数用于获取指定内存属性的便利值。它接收两个参数：`hwloc_memattr_id_t`表示要获取的内存属性ID，`hwloc_obj_t`表示包含要获取的内存对象的节点。函数首先检查所要获取的内存属性ID是否为`HWLOC_MEMATTR_ID_CAPACITY`，如果是，则返回节点属性中`numanode.local_memory`的值。否则，根据ID，函数使用`hwloc_bitmap_weight`函数获取节点CPU集合的权重，并返回该权重。否则，函数输出一个 assertion，表示函数遇到了错误。函数返回0，表示函数成功返回。

2. `hwloc_internal_memattrs_init`函数用于初始化内部内存属性。它接收一个参数：`struct hwloc_topology`表示要初始化的topology结构体。函数首先初始化topology结构体中`nr_memattrs`和`memattrs`成员变量，分别为0。然后，函数调用`hwloc_memattrs_add`函数，将topology中的所有内存属性添加到`memattrs`中。函数返回`true`，表示函数成功初始化。


```cpp
static __hwloc_inline
hwloc_uint64_t hwloc__memattr_get_convenience_value(hwloc_memattr_id_t id,
                                                    hwloc_obj_t node)
{
  if (id == HWLOC_MEMATTR_ID_CAPACITY)
    return node->attr->numanode.local_memory;
  else if (id == HWLOC_MEMATTR_ID_LOCALITY)
    return hwloc_bitmap_weight(node->cpuset);
  else
    assert(0);
  return 0; /* shut up the compiler */
}

void
hwloc_internal_memattrs_init(struct hwloc_topology *topology)
{
  topology->nr_memattrs = 0;
  topology->memattrs = NULL;
}

```

这段代码定义了一个名为 "hwloc__setup_memattr" 的函数，其作用是设置 hwloc 内部内存属性。

具体来说，函数接受三个参数：

- "imattr" 是一个指向 "hwloc_internal_memattr_s" 类型的指针，用于存储设置的内存属性。
- "name" 是一个字符指针，用于存储设置的内存属性名称，函数不会对名称进行修改。
- "flags" 是第二个参数，用于设置设置的内存属性标志，具体格式见下文。
- "iflags" 是第三个参数，用于设置设置的内存属性指示符，具体格式见下文。

函数首先将 "name" 和 "flags" 存储到 "imattr" 指向的内存属性中，然后设置 "imattr" 指向的内存属性 "iflags" 和 "nr_targets" 的值，最后初始化 "imattr" 指向的内存属性 "targets" 的长度为 0。


```cpp
static void
hwloc__setup_memattr(struct hwloc_internal_memattr_s *imattr,
                     char *name,
                     unsigned long flags,
                     unsigned long iflags)
{
  imattr->name = name;
  imattr->flags = flags;
  imattr->iflags = iflags;

  imattr->nr_targets = 0;
  imattr->targets = NULL;
}

void
```

This function appears to sets up memory attributes for a system-level LRU cache. The function takes a single parameter, which is a pointer to a `hwloc` structure containing information about the cache. The structure contains an array of `hwloc__setup_memattr` functions, which are used to set the various attributes of the cache, such as the bandwidth width, latency, and flags indicating whether the cache is a primary or secondary cache.

The `hwloc__setup_memattr` function takes three arguments:

* The first argument is a `hwloc` member to be added to the cache, and is a pointer to the member's `setup_memattr` function.
* The second argument is a string指针 to use for the member's name, and is the name of the attribute to use for the member.
* The third argument is a `hwloc` member to be added to the cache, and is a pointer to the `setup_memattr` function for the member.

The `setup_memattr` function is used to set the member to the specified value or to call the `setup_memattr` function for the member. The `hwloc` structure provides a high-level interface for interacting with the cache, and the `setup_memattr` function is one of several functions provided by this structure.


```cpp
hwloc_internal_memattrs_prepare(struct hwloc_topology *topology)
{
  topology->memattrs = malloc(HWLOC_MEMATTR_ID_MAX * sizeof(*topology->memattrs));
  if (!topology->memattrs)
    return;

  hwloc__setup_memattr(&topology->memattrs[HWLOC_MEMATTR_ID_CAPACITY],
                       (char *) "Capacity",
                       HWLOC_MEMATTR_FLAG_HIGHER_FIRST,
                       HWLOC_IMATTR_FLAG_STATIC_NAME|HWLOC_IMATTR_FLAG_CONVENIENCE);

  hwloc__setup_memattr(&topology->memattrs[HWLOC_MEMATTR_ID_LOCALITY],
                       (char *) "Locality",
                       HWLOC_MEMATTR_FLAG_LOWER_FIRST,
                       HWLOC_IMATTR_FLAG_STATIC_NAME|HWLOC_IMATTR_FLAG_CONVENIENCE);

  hwloc__setup_memattr(&topology->memattrs[HWLOC_MEMATTR_ID_BANDWIDTH],
                       (char *) "Bandwidth",
                       HWLOC_MEMATTR_FLAG_HIGHER_FIRST|HWLOC_MEMATTR_FLAG_NEED_INITIATOR,
                       HWLOC_IMATTR_FLAG_STATIC_NAME);

  hwloc__setup_memattr(&topology->memattrs[HWLOC_MEMATTR_ID_READ_BANDWIDTH],
                       (char *) "ReadBandwidth",
                       HWLOC_MEMATTR_FLAG_HIGHER_FIRST|HWLOC_MEMATTR_FLAG_NEED_INITIATOR,
                       HWLOC_IMATTR_FLAG_STATIC_NAME);

  hwloc__setup_memattr(&topology->memattrs[HWLOC_MEMATTR_ID_WRITE_BANDWIDTH],
                       (char *) "WriteBandwidth",
                       HWLOC_MEMATTR_FLAG_HIGHER_FIRST|HWLOC_MEMATTR_FLAG_NEED_INITIATOR,
                       HWLOC_IMATTR_FLAG_STATIC_NAME);

  hwloc__setup_memattr(&topology->memattrs[HWLOC_MEMATTR_ID_LATENCY],
                       (char *) "Latency",
                       HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_NEED_INITIATOR,
                       HWLOC_IMATTR_FLAG_STATIC_NAME);

  hwloc__setup_memattr(&topology->memattrs[HWLOC_MEMATTR_ID_READ_LATENCY],
                       (char *) "ReadLatency",
                       HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_NEED_INITIATOR,
                       HWLOC_IMATTR_FLAG_STATIC_NAME);

  hwloc__setup_memattr(&topology->memattrs[HWLOC_MEMATTR_ID_WRITE_LATENCY],
                       (char *) "WriteLatency",
                       HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_NEED_INITIATOR,
                       HWLOC_IMATTR_FLAG_STATIC_NAME);

  topology->nr_memattrs = HWLOC_MEMATTR_ID_MAX;
}

```

这两段代码是针对HWLOC_MEMATTR_FLAG_NEED_INITIATOR标志的属性。当这个标志为真时，说明imtg是此类属性的目标，而imattr是属性的内部表示。

在hwloc__imtg_destroy函数中，首先检查imattr的标志是否为需要引导器。如果是，那么处理完initiator后，还需要遍历所有的initiators，将其指向的内存区域释放。

在hwloc__imi_destroy函数中，直接处理initiator，将其指向的内存区域释放。

注意：在hwloc_mematattr_destroy函数中，只有当imtg是此类属性时，才会执行这两段代码。


```cpp
static void
hwloc__imi_destroy(struct hwloc_internal_memattr_initiator_s *imi)
{
  if (imi->initiator.type == HWLOC_LOCATION_TYPE_CPUSET)
    hwloc_bitmap_free(imi->initiator.location.cpuset);
}

static void
hwloc__imtg_destroy(struct hwloc_internal_memattr_s *imattr,
                    struct hwloc_internal_memattr_target_s *imtg)
{
  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    /* only attributes with initiators may have something to free() in the array */
    unsigned k;
    for(k=0; k<imtg->nr_initiators; k++)
      hwloc__imi_destroy(&imtg->initiators[k]);
  }
  free(imtg->initiators);
}

```

这段代码定义了一个名为 `hwloc_internal_memattrs_destroy` 的函数，它的作用是释放内存和卸载内部内存属性。它接受一个 `struct hwloc_topology` 类型的参数 `topology`。

函数内部首先定义了一个 `id` 变量，用于跟踪在内部内存属性中要释放的每个内存分配单元的 ID。

接下来，使用一个循环遍历 `topology->nr_memattrs` 个内部内存属性，并逐个检查每个属性的目标是否已经卸载。如果是，则释放该属性的目标，并检查是否已经释放了该属性的所有内存分配单元，如果是，则释放该属性的名称。然后，根据内存分配单元是否为静态内存属性，来决定是否释放该属性的名称。最后，释放该内部内存属性，并设置 `topology->memattrs` 为 `NULL`，设置 `topology->nr_memattrs` 为 0。


```cpp
void
hwloc_internal_memattrs_destroy(struct hwloc_topology *topology)
{
  unsigned id;
  for(id=0; id<topology->nr_memattrs; id++) {
    struct hwloc_internal_memattr_s *imattr = &topology->memattrs[id];
    unsigned j;
    for(j=0; j<imattr->nr_targets; j++)
      hwloc__imtg_destroy(imattr, &imattr->targets[j]);
    free(imattr->targets);
    if (!(imattr->iflags & HWLOC_IMATTR_FLAG_STATIC_NAME))
      free(imattr->name);
  }
  free(topology->memattrs);

  topology->memattrs = NULL;
  topology->nr_memattrs = 0;
}

```

This function appears to be part of a high-level software library for managing海安玻璃门（HWLOC）的初始化和内存管理。具体来说，该函数负责初始化海安玻璃门的初始化器（initiators）并创建内存映射，以便在内存映射中引用初始化器。

在该函数中，首先检查初始化器是否已经初始化完成。如果已经完成初始化，则创建一个新的内存映射，其中包含初始化器的副本，并从初始化器中获取要映射的CPU集合。接下来，对于每个初始化器，使用相同的检查方式来确定它是要映射到硬件设备（例如CPU）还是内存中的某个位置。然后，根据它要映射到的硬件设备类型，使用相应的内存映射技术将其映射到正确的硬件设备上。最后，如果初始化器失败，则返回-1并释放内存映射。


```cpp
int
hwloc_internal_memattrs_dup(struct hwloc_topology *new, struct hwloc_topology *old)
{
  struct hwloc_tma *tma = new->tma;
  struct hwloc_internal_memattr_s *imattrs;
  hwloc_memattr_id_t id;

  /* old->nr_memattrs is always > 0 thanks to default memattrs */

  imattrs = hwloc_tma_malloc(tma, old->nr_memattrs * sizeof(*imattrs));
  if (!imattrs)
    return -1;
  new->memattrs = imattrs;
  new->nr_memattrs = old->nr_memattrs;
  memcpy(imattrs, old->memattrs, old->nr_memattrs * sizeof(*imattrs));

  for(id=0; id<old->nr_memattrs; id++) {
    struct hwloc_internal_memattr_s *oimattr = &old->memattrs[id];
    struct hwloc_internal_memattr_s *nimattr = &imattrs[id];
    unsigned j;

    assert(oimattr->name);
    nimattr->name = hwloc_tma_strdup(tma, oimattr->name);
    if (!nimattr->name) {
      assert(!tma || !tma->dontfree); /* this tma cannot fail to allocate */
      new->nr_memattrs = id;
      goto failed;
    }
    nimattr->iflags &= ~HWLOC_IMATTR_FLAG_STATIC_NAME;
    nimattr->iflags &= ~HWLOC_IMATTR_FLAG_CACHE_VALID; /* cache will need refresh */

    if (!oimattr->nr_targets)
      continue;

    nimattr->targets = hwloc_tma_malloc(tma, oimattr->nr_targets * sizeof(*nimattr->targets));
    if (!nimattr->targets) {
      free(nimattr->name);
      new->nr_memattrs = id;
      goto failed;
    }
    memcpy(nimattr->targets, oimattr->targets, oimattr->nr_targets * sizeof(*nimattr->targets));

    for(j=0; j<oimattr->nr_targets; j++) {
      struct hwloc_internal_memattr_target_s *oimtg = &oimattr->targets[j];
      struct hwloc_internal_memattr_target_s *nimtg = &nimattr->targets[j];
      unsigned k;

      nimtg->obj = NULL; /* cache will need refresh */

      if (!oimtg->nr_initiators)
        continue;

      nimtg->initiators = hwloc_tma_malloc(tma, oimtg->nr_initiators * sizeof(*nimtg->initiators));
      if (!nimtg->initiators) {
        nimattr->nr_targets = j;
        new->nr_memattrs = id+1;
        goto failed;
      }
      memcpy(nimtg->initiators, oimtg->initiators, oimtg->nr_initiators * sizeof(*nimtg->initiators));

      for(k=0; k<oimtg->nr_initiators; k++) {
        struct hwloc_internal_memattr_initiator_s *oimi = &oimtg->initiators[k];
        struct hwloc_internal_memattr_initiator_s *nimi = &nimtg->initiators[k];
        if (oimi->initiator.type == HWLOC_LOCATION_TYPE_CPUSET) {
          nimi->initiator.location.cpuset = hwloc_bitmap_tma_dup(tma, oimi->initiator.location.cpuset);
          if (!nimi->initiator.location.cpuset) {
            nimtg->nr_initiators = k;
            nimattr->nr_targets = j+1;
            new->nr_memattrs = id+1;
            goto failed;
          }
        } else if (oimi->initiator.type == HWLOC_LOCATION_TYPE_OBJECT) {
          nimi->initiator.location.object.obj = NULL; /* cache will need refresh */
        }
      }
    }
  }
  return 0;

 failed:
  hwloc_internal_memattrs_destroy(new);
  return -1;
}

```

这段代码定义了一个名为 `hwloc_memattr_get_by_name` 的函数，它用于在 `hwloc_topology_t` 结构体中查找一个给定名称的内存区域属性ID。

具体来说，函数接受三个参数：

1. `topology`：一个 `hwloc_topology_t` 类型的变量，用于存储要查找的内存区域属性的 topology 结构体。
2. `name`：一个指向字符串的指针，用于指定要查找的内存区域属性的名称。
3. `idp`：一个指向内存区域属性 ID 的指针，用于存储找到的 ID。

函数内部首先定义了一个全局变量 `id`，用于跟踪已经查找过的 ID，然后使用一个循环遍历 topology 结构体中的所有内存区域属性。在循环内部，如果找到名为给定名称的内存区域属性，就将其 ID 存储到 `idp` 指向的内存区域属性 ID 变量中，并返回 0。否则，函数返回一个负的错误码。


```cpp
int
hwloc_memattr_get_by_name(hwloc_topology_t topology,
                          const char *name,
                          hwloc_memattr_id_t *idp)
{
  unsigned id;
  for(id=0; id<topology->nr_memattrs; id++) {
    if (!strcmp(topology->memattrs[id].name, name)) {
      *idp = id;
      return 0;
    }
  }
  errno = EINVAL;
  return -1;
}

```

这两函数是用来从 HWLOC_MEMATTR_INIT 结构体中获取相关信息的。具体来说：

`hwloc_memattr_get_name`函数的作用是获取一个给定内存布局 topology 中内存属性 id 的名称，其返回值类型为 `const char *` 类型。如果 id 不在 topology 的 `nr_memattrs` 成员中，函数将返回 `-1`，并错误地抛出 `errno` 异常。

`hwloc_memattr_get_flags`函数的作用是获取给定内存布局 topology 中内存属性 id 的标志，其返回值类型为 `unsigned long` 类型。如果 id 不在 topology 的 `nr_memattrs` 成员中，函数将返回 `-1`，并错误地抛出 `errno` 异常。


```cpp
int
hwloc_memattr_get_name(hwloc_topology_t topology,
                       hwloc_memattr_id_t id,
                       const char **namep)
{
  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  *namep = topology->memattrs[id].name;
  return 0;
}

int
hwloc_memattr_get_flags(hwloc_topology_t topology,
                        hwloc_memattr_id_t id,
                        unsigned long *flagsp)
{
  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  *flagsp = topology->memattrs[id].flags;
  return 0;
}

```



This function appears to be a part of the "hwloc" library in C, which is a library for managing hardware layout of high-performance systems.

It appears to be checking whether a given memory attribute (i.e., a memory-mapped virtual memory address) is already in use by a previously created memory attribute with the same name. If it is, it returns an error code.

If the memory attribute is not already in use, it checks if the attribute exists in the memory hierarchy and assigns the memory-mapped virtual memory address to the new attribute. It also checks if the attribute already exists in the memory hierarchy, and if it does, it sets the attribute's cache valid flag.

Finally, it sets the ID of the new attribute to the memory-mapped virtual memory address and increments the total number of memory attributes in the memory hierarchy.


```cpp
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
  if (flags & ~(HWLOC_MEMATTR_FLAG_NEED_INITIATOR|HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_HIGHER_FIRST)) {
    errno = EINVAL;
    return -1;
  }
  if (!(flags & (HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_HIGHER_FIRST))) {
    errno = EINVAL;
    return -1;
  }
  if ((flags & (HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_HIGHER_FIRST))
      == (HWLOC_MEMATTR_FLAG_LOWER_FIRST|HWLOC_MEMATTR_FLAG_HIGHER_FIRST)) {
    errno = EINVAL;
    return -1;
  }

  if (!_name) {
    errno = EINVAL;
    return -1;
  }

  /* check name isn't already used */
  for(i=0; i<topology->nr_memattrs; i++) {
    if (!strcmp(_name, topology->memattrs[i].name)) {
      errno = EBUSY;
      return -1;
    }
  }

  name = strdup(_name);
  if (!name)
    return -1;

  newattrs = realloc(topology->memattrs, (topology->nr_memattrs + 1) * sizeof(*topology->memattrs));
  if (!newattrs) {
    free(name);
    return -1;
  }

  hwloc__setup_memattr(&newattrs[topology->nr_memattrs],
                       name, flags, 0);

  /* memattr valid when just created */
  newattrs[topology->nr_memattrs].iflags |= HWLOC_IMATTR_FLAG_CACHE_VALID;

  *id = topology->nr_memattrs;
  topology->nr_memattrs++;
  topology->memattrs = newattrs;
  return 0;
}


```

这段代码定义了一个名为`match_internal_location`的函数，它接收两个结构体：`struct hwloc_internal_location_s` 和 `struct hwloc_internal_memattr_initiator_s`，用于比较查询和现有的初始化器位置。

函数首先检查输入的`iloc`和`imi`的类型是否相同，如果不相同，函数返回0。然后，函数根据输入的类型来执行比较操作。

如果输入的类型是`HWLOC_LOCATION_TYPE_CPUSET`，函数会比较查询的`cpuset`和初始化器位置的`cpuset`是否相同。如果是，函数返回`hwloc_bitmap_isincluded`函数的返回值，表示查询的`cpuset`是否在初始化器位置的`cpuset`中。

如果输入的类型是`HWLOC_LOCATION_TYPE_OBJECT`，函数会比较查询的`location.type`是否与初始化器位置的`object.type`相同，以及`location.gp_index`是否与初始化器位置的`object.gp_index`相同。如果是，函数返回`true`，否则返回`false`。

如果输入的类型不是`HWLOC_LOCATION_TYPE_CPUSET`或`HWLOC_LOCATION_TYPE_OBJECT`，函数将返回`0`。


```cpp
/***************************
 * Internal Locations
 */

/* return 1 if cpuset/obj matchs the existing initiator location,
 * for instance if the cpuset of query is included in the cpuset of existing
 */
static int
match_internal_location(struct hwloc_internal_location_s *iloc,
                        struct hwloc_internal_memattr_initiator_s *imi)
{
  if (iloc->type != imi->initiator.type)
    return 0;
  switch (iloc->type) {
  case HWLOC_LOCATION_TYPE_CPUSET:
    return hwloc_bitmap_isincluded(iloc->location.cpuset, imi->initiator.location.cpuset);
  case HWLOC_LOCATION_TYPE_OBJECT:
    return iloc->location.object.type == imi->initiator.location.object.type
      && iloc->location.object.gp_index == imi->initiator.location.object.gp_index;
  default:
    return 0;
  }
}

```

该函数的作用是转换一个硬件位置（HWLOC_LOCATION_TYPE_CPUSET或HWLOC_LOCATION_TYPE_OBJECT）为内部位置（to_internal_location），并将其存储在结构体变量iloc中，同时返回转换结果。

函数接收两个参数：一个指向内部位置（struct hwloc_internal_location_s）的指针iloc和一个指向内部位置的指针location。函数首先将iloc->type设置为location->type，然后根据location->type类型进行switch判断。

如果location->type为HWLOC_LOCATION_TYPE_CPUSET，函数首先检查location->location.cpuset是否为真，并检查hwloc_bitmap_iszero(location->location.cpuset)是否为真。如果是，则函数返回0；否则，函数返回-1并输出"errno"。

如果location->type为HWLOC_LOCATION_TYPE_OBJECT，函数首先检查location->location.object是否存在。如果不存在，则函数返回-1并输出"errno"。否则，函数将iloc->location.cpuset设置为location->location.object->gp_index，并将iloc->location.object->type设置为location->location.object->type，然后返回0。

如果location->type为HWLOC_LOCATION_TYPE_UNKNOWN，函数将iloc->type设置为-1，并返回-1。


```cpp
static int
to_internal_location(struct hwloc_internal_location_s *iloc,
                     struct hwloc_location *location)
{
  iloc->type = location->type;

  switch (location->type) {
  case HWLOC_LOCATION_TYPE_CPUSET:
    if (!location->location.cpuset || hwloc_bitmap_iszero(location->location.cpuset)) {
      errno = EINVAL;
      return -1;
    }
    iloc->location.cpuset = location->location.cpuset;
    return 0;
  case HWLOC_LOCATION_TYPE_OBJECT:
    if (!location->location.object) {
      errno = EINVAL;
      return -1;
    }
    iloc->location.object.gp_index = location->location.object->gp_index;
    iloc->location.object.type = location->location.object->type;
    return 0;
  default:
    errno = EINVAL;
    return -1;
  }
}

```

该代码是一个名为 `static int from_internal_location(struct hwloc_internal_location_s *iloc, struct hwloc_location *location)` 的函数，它从 `hwloc_internal_location_s` 结构体中获取一个 `struct hwloc_location` 变量，然后根据传入的 `iloc` 结构体中 `type` 成员的值，执行相应的操作，并将结果存储到 `location` 结构体中。

具体来说，该函数的作用如下：

1. 如果 `iloc->type` 的值为 `HWLOC_LOCATION_TYPE_CPUSET`，则执行以下操作：

  a. 将 `iloc->location.cpuset` 的值存储到 `location->location.cpuset` 中；

  b. 返回 0；

2. 如果 `iloc->type` 的值为 `HWLOC_LOCATION_TYPE_OBJECT`，则执行以下操作：

  a. 如果 `location->location.object` 为 `NULL`，则执行以下操作：

     i. 返回 `-1`；

     ii. 在 `location` 结构体中设置 `location->location.object` 为 `iloc->location.object.obj`；

  b. 返回 0；

3. 如果 `iloc->type` 的值既不是 `HWLOC_LOCATION_TYPE_CPUSET` 也不是 `HWLOC_LOCATION_TYPE_OBJECT`，则返回 `-1`。

该函数通过 `hwloc_internal_location_s` 结构体中的 `type` 成员，来控制对 `struct hwloc_location` 变量的操作，根据不同的类型，函数会执行不同的操作，并将结果存储到 `location` 结构体中，以供其他函数使用。


```cpp
static int
from_internal_location(struct hwloc_internal_location_s *iloc,
                       struct hwloc_location *location)
{
  location->type = iloc->type;

  switch (iloc->type) {
  case HWLOC_LOCATION_TYPE_CPUSET:
    location->location.cpuset = iloc->location.cpuset;
    return 0;
  case HWLOC_LOCATION_TYPE_OBJECT:
    /* requires the cache to be refreshed */
    location->location.object = iloc->location.object.obj;
    if (!location->location.object)
      return -1;
    return 0;
  default:
    errno = EINVAL;
    return -1;
  }
}


```

这段代码是一个名为 `hwloc__imi_refresh` 的函数，它是 `hwloc_topology_init` 函数的实现在 `.hwloc_topology` 文件中的实现。

该函数接收两个参数：`topology` 是一个表示 HWLOC  topology 的结构体，`imi` 是一个表示 HWLOC 内部内存分配器初始化的结构体。

函数的作用是刷新给定的 HWLOC  topology 中的内存区域，使得该区域内的所有内存都为 0。以下是函数的实现细节：

1. 首先定义了一个 `hwloc__imi_refresh` 函数，它的参数 `topology` 和 `imi`。
2. 函数内部的第一个交换语句 `hwloc_bitmap_and` 接收两个参数：一个是 `imi` 中的 `initiator.location.cpuset`，另一个是 topology 中的 `levels[0][0]->cpuset`。通过与 `cpuset` 进行与操作，将两个 `cpuset` 合并，得到一个新的 `cpuset` 作为参数返回。
3. 如果两个 `cpuset` 相等（即 `hwloc_bitmap_iszero` 返回 0），则函数返回 0，否则执行以下操作：
	1. 调用 `hwloc__imi_destroy` 函数，传递 `imi` 参数，作为参数传递给 `destroy` 函数。
	2. 返回 -1，表示发生了错误。

该函数的作用是刷新给定的 HWLOC  topology 中的内存区域，使得该区域内的所有内存都为 0。这个实现与 `hwloc_topology_init` 函数的逻辑是相同的，但是使用了不同的参数类型。


```cpp
/************************
 * Refreshing
 */

static int
hwloc__imi_refresh(struct hwloc_topology *topology,
                   struct hwloc_internal_memattr_initiator_s *imi)
{
  switch (imi->initiator.type) {
  case HWLOC_LOCATION_TYPE_CPUSET: {
    hwloc_bitmap_and(imi->initiator.location.cpuset, imi->initiator.location.cpuset, topology->levels[0][0]->cpuset);
    if (hwloc_bitmap_iszero(imi->initiator.location.cpuset)) {
      hwloc__imi_destroy(imi);
      return -1;
    }
    return 0;
  }
  case HWLOC_LOCATION_TYPE_OBJECT: {
    hwloc_obj_t obj = hwloc_get_obj_by_type_and_gp_index(topology,
                                                         imi->initiator.location.object.type,
                                                         imi->initiator.location.object.gp_index);
    if (!obj) {
      hwloc__imi_destroy(imi);
      return -1;
    }
    imi->initiator.location.object.obj = obj;
    return 0;
  }
  default:
    assert(0);
  }
  return -1;
}

```

This function appears to be a part of a higher-level software library that manages node data, and it's used to cache an `IMAttribute` object that has been initialized with a `IMToken`.

It works as follows:

1. If the `IMToken` already has a `NUMA` index, the function retrieves the corresponding `NUMA` node from the topology.
2. If the `IMToken` does not have a `NUMA` index, the function retrieves the `PU` node that corresponds to the `IMToken`.
3. If the `IMToken` does not have a `PU` node, the function returns `NULL`.
4. If the `IMToken` already has a `PU` node, the function saves the `gp_index` in the `IMAttribute` object and returns `0`.
5. If the `IMAttribute` does not have a `gp_index`, the function creates a new `IMAttribute` object, sets the `gp_index` to the `imtg->gp_index`, and caches the `IMAttribute` object.
6. If the `IMToken` already has an initiator, the function checks the initiator and refills it.
7. If the `IMToken` does not have an initiator, the function waits until the `IMToken` has a `NUMA` index and creates an initiator.

Note: `hwloc_get_numanode_obj_by_os_index()` and `hwloc_get_pu_obj_by_os_index()` are not defined in the code provided, so it's unclear what they do.


```cpp
static int
hwloc__imtg_refresh(struct hwloc_topology *topology,
                    struct hwloc_internal_memattr_s *imattr,
                    struct hwloc_internal_memattr_target_s *imtg)
{
  hwloc_obj_t node;

  /* no need to refresh convenience memattrs */
  assert(!(imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE));

  /* check the target object */
  if (imtg->gp_index == (hwloc_uint64_t) -1) {
    /* only NUMA and PU may work with os_index, and only NUMA is currently used internally */
    if (imtg->type == HWLOC_OBJ_NUMANODE)
      node = hwloc_get_numanode_obj_by_os_index(topology, imtg->os_index);
    else if (imtg->type == HWLOC_OBJ_PU)
      node = hwloc_get_pu_obj_by_os_index(topology, imtg->os_index);
    else
      node = NULL;
  } else {
    node = hwloc_get_obj_by_type_and_gp_index(topology, imtg->type, imtg->gp_index);
  }
  if (!node) {
    hwloc__imtg_destroy(imattr, imtg);
    return -1;
  }

  /* save the gp_index in case it wasn't initialized yet */
  imtg->gp_index = node->gp_index;
  /* cache the object */
  imtg->obj = node;

  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    /* check the initiators */
    unsigned k, l;
    for(k=0, l=0; k<imtg->nr_initiators; k++) {
      int err = hwloc__imi_refresh(topology, &imtg->initiators[k]);
      if (err < 0)
        continue;
      if (k != l)
        memcpy(&imtg->initiators[l], &imtg->initiators[k], sizeof(*imtg->initiators));
      l++;
    }
    imtg->nr_initiators = l;
    if (!imtg->nr_initiators) {
      hwloc__imtg_destroy(imattr, imtg);
      return -1;
    }
  }
  return 0;
}

```

这段代码是一个名为 `hwloc__imattr_refresh` 的函数，它是 `hwloc_topology` 结构体内部的一个成员函数，它的作用是刷新 `hwloc_imattr_s` 结构体中的内存区域，并更新相关的属性和标志。

具体来说，这个函数接收两个参数：一个 `hwloc_topology` 结构体变量 `topology`，和一个 `hwloc_imattr_s` 结构体变量 `imattr`。函数内部首先定义了一个变量 `j`，和一个变量 `k`，用于跟踪 `imattr` 结构体中目标的变化。

接着函数内部遍历 `imattr` 结构体中所有的目标，对于每个目标，函数调用一个名为 `hwloc__imtg_refresh` 的内部函数，传递 `topology` 和 `imattr` 作为参数，并将返回值存储在 `ret` 变量中。如果 `ret` 不是 0，说明有些目标仍然有效，函数会根据目标的位置来移动这些目标，并将移动后的状态更新到 `imattr` 结构体中。

在遍历过程中，如果 `j` 不等于 `k`，说明有些目标已经被移除，函数会将那些目标的目标复制到距离较近的位置，并将 `k` 的值递增，以便在复制时可以正确地进行比较。最后，函数会将 `imattr` 结构体中的 `nr_targets` 成员变量更新为复制后目标的数量，并将 `iflags` 成员变量设置为 `HWLOC_IMATTR_FLAG_CACHE_VALID`，以指示是否可以使用缓存。


```cpp
static void
hwloc__imattr_refresh(struct hwloc_topology *topology,
                      struct hwloc_internal_memattr_s *imattr)
{
  unsigned j, k;
  for(j=0, k=0; j<imattr->nr_targets; j++) {
    int ret = hwloc__imtg_refresh(topology, imattr, &imattr->targets[j]);
    if (!ret) {
      /* target still valid, move it if some former targets were removed */
      if (j != k)
        memcpy(&imattr->targets[k], &imattr->targets[j], sizeof(*imattr->targets));
      k++;
    }
  }
  imattr->nr_targets = k;
  imattr->iflags |= HWLOC_IMATTR_FLAG_CACHE_VALID;
}

```



这两函数是 `hwloc_internal_memattrs_refresh` 和 `hwloc_internal_memattrs_need_refresh` 函数，用于管理内部内存属性。

`hwloc_internal_memattrs_refresh` 函数用于刷新内部内存属性。具体来说，它遍历 `topology->nr_memattrs` 并检查每个内存属性的内存指针是否设置了 `HWLOC_IMATTR_FLAG_CACHE_VALID` 标志。如果是，则不需要刷新缓存，否则设置 `HWLOC_IMATTR_FLAG_CONVENIENCE` 标志以表示不需要刷新该内存属性。

`hwloc_internal_memattrs_need_refresh` 函数用于通知需要刷新哪些内存属性。它遍历 `topology->nr_memattrs` 并检查每个内存属性是否设置了 `HWLOC_IMATTR_FLAG_CONVENIENCE` 标志。如果是，则不需要刷新该内存属性，否则设置 `HWLOC_IMATTR_FLAG_CACHE_VALID` 标志以表示需要刷新该内存属性。

在 `hwloc_topology` 结构中，`memattrs` 成员是一个 `struct hwloc_internal_memattr_s` 类型的数组，每个内存属性都是一个 `struct hwloc_internal_memattr_s` 类型的结构体。`hwloc_internal_memattr_s` 结构体中包含多个内存属性，例如 `iflags` 成员，包含一个或多个标记，例如 `HWLOC_IMATTR_FLAG_CACHE_VALID` 标记用于指示缓存是否有效。

这两个函数是 `hwloc_topology` 结构的一部分，用于管理内部内存属性。


```cpp
void
hwloc_internal_memattrs_refresh(struct hwloc_topology *topology)
{
  unsigned id;
  for(id=0; id<topology->nr_memattrs; id++) {
    struct hwloc_internal_memattr_s *imattr = &topology->memattrs[id];
    if (imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID)
      /* nothing to refresh */
      continue;
    hwloc__imattr_refresh(topology, imattr);
  }
}

void
hwloc_internal_memattrs_need_refresh(struct hwloc_topology *topology)
{
  unsigned id;
  for(id=0; id<topology->nr_memattrs; id++) {
    struct hwloc_internal_memattr_s *imattr = &topology->memattrs[id];
    if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE)
      /* no need to refresh convenience memattrs */
      continue;
    imattr->iflags &= ~HWLOC_IMATTR_FLAG_CACHE_VALID;
  }
}


```



This function appears to be a part of a higher-level software framework for managing hardware devices, such as the Linux kernel's pcm device driver.

It appears to be taking in an array of target-os-index values and a target_type value, and setting up a new memory region for the target object. It seems to be validating that the target object has been initialized correctly, and then initializing the memory region according to the specified target type.

If `create` is 1, the function creates the memory region and returns a pointer to it. Otherwise, it returns `NULL`.

The function also seems to be implementing a cache mechanism for storing the target object in memory, in order to avoid invalidating the cache on each initialization of the target object. The cache is initialized with the `imattr->nr_targets` target objects, and has a default value on initialization. When the target object is initialized later on, the function refreshes the cache and sets the cache value to the initialized value.


```cpp
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

  for(j=0; j<imattr->nr_targets; j++) {
    if (target_type == imattr->targets[j].type)
      if ((target_gp_index != (hwloc_uint64_t)-1 && target_gp_index == imattr->targets[j].gp_index)
          || (target_os_index != (unsigned)-1 && target_os_index == imattr->targets[j].os_index))
        return &imattr->targets[j];
  }
  if (!create)
    return NULL;

  news = realloc(imattr->targets, (imattr->nr_targets+1)*sizeof(*imattr->targets));
  if (!news)
    return NULL;
  imattr->targets = news;

  /* FIXME sort targets? by logical index at the end of load? */

  new = &news[imattr->nr_targets];
  new->type = target_type;
  new->gp_index = target_gp_index;
  new->os_index = target_os_index;

  /* cached object will be refreshed later on actual access */
  new->obj = NULL;
  imattr->iflags &= ~HWLOC_IMATTR_FLAG_CACHE_VALID;
  /* When setting a value after load(), the caller has the target object
   * (and initiator object, if not CPU set). Hence, we could avoid invalidating
   * the cache here.
   * The overhead of the imattr-wide refresh isn't high enough so far
   * to justify making the cache management more complex.
   */

  new->nr_initiators = 0;
  new->initiators = NULL;
  new->noinitiator_value = 0;
  imattr->nr_targets++;
  return new;
}

```

This function appears to be a member function of the `hwloc` library, which appears to be a Hadoop-specific logging library.

It appears to be reading through a list of `HWLOC_IMATTR_CONVENIENCE` attributes and refreshing them with the `hwloc__memattr_get_convenience_value` function if they have been previously computed.

If the `IMATTR_FLAG_CONVENIENCE` flag is set, it looks like the function will consider the convenience attributes (i.e. attributes with the `HWLOC_IMATTR_FLAG_CONVENIENCE` flag) and compute their values. It then stores the target object for each of the convenience attributes in the `targets` array.

If the `IMATTR_FLAG_NEED_INITIATOR` flag is set, the function will check if an initiator node exists for the given `imattr`. If one does not exist, it continues to the next line. If it does exist, it gets the value of the attribute from the `noinitiator_value` member function of the `IMATTR_INITIATOR_TRANSACTION` struct.

The function then continues to the next line to compute the target object for the convenience attribute.

It is difficult to fully understand the behavior of this function without more context.


```cpp
static struct hwloc_internal_memattr_initiator_s *
hwloc__memattr_get_initiator_from_location(struct hwloc_internal_memattr_s *imattr,
                                           struct hwloc_internal_memattr_target_s *imtg,
                                           struct hwloc_location *location);

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

  if (!nrp || (*nrp && !targets)) {
    errno = EINVAL;
    return -1;
  }
  max = *nrp;

  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  imattr = &topology->memattrs[id];

  if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {
    /* convenience attributes */
    for(i=0; ; i++) {
      hwloc_obj_t node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, i);
      if (!node)
        break;
      if (found<max) {
        targets[found] = node;
        if (values)
          values[found] = hwloc__memattr_get_convenience_value(id, node);
      }
      found++;
    }
    goto done;
  }

  /* normal attributes */

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    hwloc__imattr_refresh(topology, imattr);

  for(i=0; i<imattr->nr_targets; i++) {
    struct hwloc_internal_memattr_target_s *imtg = &imattr->targets[i];
    hwloc_uint64_t value = 0;

    if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
      if (initiator) {
        /* find a matching initiator */
        struct hwloc_internal_memattr_initiator_s *imi = hwloc__memattr_get_initiator_from_location(imattr, imtg, initiator);
        if (!imi)
          continue;
        value = imi->value;
      }
    } else {
      value = imtg->noinitiator_value;
    }

    if (found<max) {
      targets[found] = imtg->obj;
      if (values)
        values[found] = value;
    }
    found++;
  }

 done:
  *nrp = found;
  return 0;
}


```

这段代码定义了一个名为 `hwloc__memattr_target_get_initiator` 的函数，它是 `hwloc_internal_memattr_target_get_initiator` 函数的别名。

该函数的主要作用是获取指定 `hwloc_internal_memattr_target_s` 结构中一个名为 `imtg` 的内存属性目标的初始化器，并返回该初始化器。

具体实现过程如下：

1. 首先定义一个名为 `hwloc__memattr_initiator_s` 的结构体，用于存储内存属性的初始化器信息。
2. 在函数体内，使用一个循环来遍历所有可能的初始化器，对于每个初始化器，先检查该初始化器是否与指定位置的内存属性目标匹配，如果匹配则直接返回该初始化器，否则继续遍历。
3. 如果内存属性目标不存在，函数将返回一个指向空 `hwloc_internal_memattr_initiator_s` 类型的指针，表示没有找到匹配的初始化器。
4. 如果内存属性目标存在，函数将创建一个新的 `hwloc_internal_memattr_initiator_s` 类型的指针，并将该指针存储在 `imtg->initiators` 指向的内存属性目标上，然后增加 `imtg->nr_initiators` 计数器。
5. 最后，函数返回新的初始化器。


```cpp
/************************
 * Initiators
 */

static struct hwloc_internal_memattr_initiator_s *
hwloc__memattr_target_get_initiator(struct hwloc_internal_memattr_target_s *imtg,
                                    struct hwloc_internal_location_s *iloc,
                                    int create)
{
  struct hwloc_internal_memattr_initiator_s *news, *new;
  unsigned k;

  for(k=0; k<imtg->nr_initiators; k++) {
    struct hwloc_internal_memattr_initiator_s *imi = &imtg->initiators[k];
    if (match_internal_location(iloc, imi)) {
      return imi;
    }
  }

  if (!create)
    return NULL;

  news = realloc(imtg->initiators, (imtg->nr_initiators+1)*sizeof(*imtg->initiators));
  if (!news)
    return NULL;
  new = &news[imtg->nr_initiators];

  new->initiator = *iloc;
  if (iloc->type == HWLOC_LOCATION_TYPE_CPUSET) {
    new->initiator.location.cpuset = hwloc_bitmap_dup(iloc->location.cpuset);
    if (!new->initiator.location.cpuset)
      goto out_with_realloc;
  }

  imtg->nr_initiators++;
  imtg->initiators = news;
  return new;

 out_with_realloc:
  imtg->initiators = news;
  return NULL;
}

```

这段代码定义了一个名为 `hwloc__memattr_get_initiator_from_location` 的函数，它接收三个参数：

1. `imattr`：一个 `struct hwloc_internal_memattr_s` 类型的变量，表示内部内存属性。
2. `imtg`：一个 `struct hwloc_internal_memattr_target_s` 类型的变量，表示目标内存属性。
3. `location`：一个 `struct hwloc_location_s` 类型的变量，表示目标位置。

函数的作用是获取一个指向内存属性 `imattr` 的初始化器 `imi`，如果初始化器 `imi` 无法找到，函数将返回一个 `null` 值，否则返回该初始化器。

函数首先检查 `imattr` 的标志位是否包含 `HWLOC_MEMATTR_FLAG_NEED_INITIATOR`，如果是，则函数使用直接返回 `imi` 的值，否则会返回一个 `null` 值。

接着，函数将 `location` 作为参数传递给 `to_internal_location` 函数，该函数将 `location` 转换为相应的内部内存属性 `imtg` 的目标位置。

最后，函数再次检查 `imtg` 的目标位置 `location` 是否为 `NULL`，如果是，函数返回一个 `errno` 值为 `EINVAL` 的错误，否则返回 `imi`，即目标内存属性的初始化器。


```cpp
static struct hwloc_internal_memattr_initiator_s *
hwloc__memattr_get_initiator_from_location(struct hwloc_internal_memattr_s *imattr,
                                           struct hwloc_internal_memattr_target_s *imtg,
                                           struct hwloc_location *location)
{
  struct hwloc_internal_memattr_initiator_s *imi;
  struct hwloc_internal_location_s iloc;

  assert(imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR);

  /* use the initiator value */
  if (!location) {
    errno = EINVAL;
    return NULL;
  }

  if (to_internal_location(&iloc, location) < 0) {
    errno = EINVAL;
    return NULL;
  }

  imi = hwloc__memattr_target_get_initiator(imtg, &iloc, 0);
  if (!imi) {
    errno = EINVAL;
    return NULL;
  }

  return imi;
}

```

This function appears to be a part of a software library, and it is used to set the values of certain attributes in a `topology` struct.

The `imattr` argument is a pointer to an `HWLOC_MEMATTR_EXEC` struct, which contains information about the attributes of the `topology` struct. The `topology` struct seems to be a representation of some memory management technology, and the `imattr` field is used to access its attributes.

The function first checks if the `topology` has any convenience attributes, and if not, it sets the function return value to `-1` and returns an error code. If the `topology` does have convenience attributes, the function sets the function return value to `0` and returns no error code.

The function then checks if the `topology` has any initializers. If it does not, the function sets the function return value to `-1` and returns an error code. If it does, the function loops through the `initiators` array and sets the corresponding attribute value.

Finally, the function checks if the `topology` has any cache validating attributes, and if it does not, the function sets the function return value to `0` and returns no error code. If it does, the function sets the function return value to `imtg->nr_initiators` and returns no error code.


```cpp
int
hwloc_memattr_get_initiators(hwloc_topology_t topology,
                             hwloc_memattr_id_t id,
                             hwloc_obj_t target_node,
                             unsigned long flags,
                             unsigned *nrp, struct hwloc_location *initiators, hwloc_uint64_t *values)
{
  struct hwloc_internal_memattr_s *imattr;
  struct hwloc_internal_memattr_target_s *imtg;
  unsigned i, max;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  if (!nrp || (*nrp && !initiators)) {
    errno = EINVAL;
    return -1;
  }
  max = *nrp;

  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  imattr = &topology->memattrs[id];
  if (!(imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR)) {
    *nrp = 0;
    return 0;
  }

  /* all convenience attributes have no initiators */
  assert(!(imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE));

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    hwloc__imattr_refresh(topology, imattr);

  imtg = hwloc__memattr_get_target(imattr, target_node->type, target_node->gp_index, target_node->os_index, 0);
  if (!imtg) {
    errno = EINVAL;
    return -1;
  }

  for(i=0; i<imtg->nr_initiators && i<max; i++) {
    struct hwloc_internal_memattr_initiator_s *imi = &imtg->initiators[i];
    int err = from_internal_location(&imi->initiator, &initiators[i]);
    assert(!err);
    if (values)
      /* no need to handle capacity/locality special cases here, those are initiator-less attributes */
      values[i] = imi->value;
  }

  *nrp = imtg->nr_initiators;
  return 0;
}


```

0;
}

hwloc_errno_t hwloc_memattr_get_value(hwloc_id_t id, hwloc_topology_e e涛， hwloc_memattr_t *imattr, int *valuep) {
if (id < topology->nr_memattrs) {
if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {
*valuep = hwloc__memattr_get_convenience_value(id, target_node);
return 0;
}
}
}


```cpp
/**************************
 * Values
 */

int
hwloc_memattr_get_value(hwloc_topology_t topology,
                        hwloc_memattr_id_t id,
                        hwloc_obj_t target_node,
                        struct hwloc_location *initiator,
                        unsigned long flags,
                        hwloc_uint64_t *valuep)
{
  struct hwloc_internal_memattr_s *imattr;
  struct hwloc_internal_memattr_target_s *imtg;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  imattr = &topology->memattrs[id];

  if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {
    /* convenience attributes */
    *valuep = hwloc__memattr_get_convenience_value(id, target_node);
    return 0;
  }

  /* normal attributes */

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    hwloc__imattr_refresh(topology, imattr);

  imtg = hwloc__memattr_get_target(imattr, target_node->type, target_node->gp_index, target_node->os_index, 0);
  if (!imtg) {
    errno = EINVAL;
    return -1;
  }

  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    /* find the initiator and set its value */
    struct hwloc_internal_memattr_initiator_s *imi = hwloc__memattr_get_initiator_from_location(imattr, imtg, initiator);
    if (!imi)
      return -1;
    *valuep = imi->value;
  } else {
    /* get the no-initiator value */
    *valuep = imtg->noinitiator_value;
  }
  return 0;
}

```

This function appears to be a part of a fileet在读写时检查的代码。在函数中，首先检查是否有一个topology对象，并检查该对象的一个名为topology->nr_memattrs的属性是否已设置。如果已设置，则检查要执行的操作，这些操作包括设置一个名为imattr的内存属性为值，设置另一个名为imattr的内存属性为imattr->value，并将第三个名为imattr的内存属性设置为执行者。如果topology对象已初始化，但该对象中没有imattr对象，则会抛出EINVAL错误。如果topology对象已初始化，但是函数在尝试执行设置内存属性操作时遇到了问题，则会抛出EINVAL错误并返回-1。如果topology对象已初始化，并且imattr对象中有一个名为HWLOC_MEMATTR_FLAG_NEED_INITIATOR的标志设置为1，则检查给定的初始化者是否为空，如果是，则抛出EINVAL错误并返回-1。如果初始化者不为空，则执行设置内存属性操作。如果topology对象已初始化，并且imattr对象中有一个名为HWLOC_IMATTR_FLAG_CONVENIENCE的标志设置为1，则返回设置值，而不是值本身。如果topology对象已加载，但是尝试在加载时设置值时由于某些原因（例如某些节点可能尚未准备好）未能完成设置，则在设置值后，不会自动刷新。


```cpp
static int
hwloc__internal_memattr_set_value(hwloc_topology_t topology,
                                  hwloc_memattr_id_t id,
                                  hwloc_obj_type_t target_type,
                                  hwloc_uint64_t target_gp_index,
                                  unsigned target_os_index,
                                  struct hwloc_internal_location_s *initiator,
                                  hwloc_uint64_t value)
{
  struct hwloc_internal_memattr_s *imattr;
  struct hwloc_internal_memattr_target_s *imtg;

  if (id >= topology->nr_memattrs) {
    /* something bad happened during init */
    errno = EINVAL;
    return -1;
  }
  imattr = &topology->memattrs[id];

  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    /* check given initiator */
    if (!initiator) {
      errno = EINVAL;
      return -1;
    }
  }

  if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {
    /* convenience attributes are read-only */
    errno = EINVAL;
    return -1;
  }

  if (topology->is_loaded && !(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    /* don't refresh when adding values during load (some nodes might not be ready yet),
     * we'll refresh later
     */
    hwloc__imattr_refresh(topology, imattr);

  imtg = hwloc__memattr_get_target(imattr, target_type, target_gp_index, target_os_index, 1);
  if (!imtg)
    return -1;

  if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
    /* find/add the initiator and set its value */
    // FIXME what if cpuset is larger than an existing one ?
    struct hwloc_internal_memattr_initiator_s *imi = hwloc__memattr_target_get_initiator(imtg, initiator, 1);
    if (!imi)
      return -1;
    imi->value = value;

  } else {
    /* set the no-initiator value */
    imtg->noinitiator_value = value;
  }

  return 0;

}

```

这段代码定义了一个名为`hwloc_internal_memattr_set_value`的函数，属于`hwloc_internal_memattr_set_device`家族。这个函数的参数包括一个`hwloc_topology_t`类型的`topology`，一个`hwloc_memattr_id_t`类型的`id`，一个`hwloc_obj_type_t`类型的`target_type`，一个`hwloc_uint64_t`类型的`target_gp_index`，一个`unsigned`类型的`target_os_index`，一个`struct hwloc_internal_location_s *initiator`类型的`initiator`，和一个`hwloc_uint64_t`类型的`value`。函数的作用是设置`target_gp_index`和`target_os_index`所指定的内存区域，并返回设置后的`value`。如果`id`等于`HWLOC_MEMATTR_ID_CAPACITY`，则尝试将该内存区域设置为可用内存，否则允许该内存区域为初始值。


```cpp
int
hwloc_internal_memattr_set_value(hwloc_topology_t topology,
                                 hwloc_memattr_id_t id,
                                 hwloc_obj_type_t target_type,
                                 hwloc_uint64_t target_gp_index,
                                 unsigned target_os_index,
                                 struct hwloc_internal_location_s *initiator,
                                 hwloc_uint64_t value)
{
  assert(id != HWLOC_MEMATTR_ID_CAPACITY);
  assert(id != HWLOC_MEMATTR_ID_LOCALITY);

  return hwloc__internal_memattr_set_value(topology, id, target_type, target_gp_index, target_os_index, initiator, value);
}

```

这段代码定义了一个名为 `hwloc_memattr_set_value` 的函数，属于 `hwloc_topology_ex` 类的成员函数。它的作用是设置一个 `hwloc_memattr` 属性的值，具体的设置操作根据传递给它的 `hwloc_topology_t` 类型的参数和 `hwloc_memattr_id_t` 类型的参数进行。

函数的参数列表如下：

- `hwloc_topology_t`：表示要设置的 `hwloc_memattr` 所属的 topology 类型。
- `hwloc_memattr_id_t`：表示要设置的 `hwloc_memattr` 的ID。
- `hwloc_obj_t`：表示目标节点的 `hwloc_obj` 类型。
- `struct hwloc_location *initiator`：如果有提供初始化器，则该参数指向该初始化器。
- `unsigned long flags`：设置值时传递给 `hwloc_topology_ex` 的一些标志，具体解释见下。
- `hwloc_uint64_t value`：要设置的 `hwloc_memattr` 属性的值。

函数的实现主要步骤如下：

1. 检查传递给自己的 `hwloc_topology_t` 参数中是否提供了 `initiator`，并检查 `initiator` 是否为 `NULL`。如果 `initiator` 为 `NULL`，则表示没有提供初始化器，需要通过调用 `hwloc__internal_memattr_set_value` 函数进行初始化，并返回初始化失败的返回值。
2. 检查传递给自己的 `hwloc_memattr_id_t` 参数中指定的 `hwloc_topology_t` 参数的 `topology` 成员是否与传递给自己的 `hwloc_obj_t` 指定的 `target_node->type` 和 `target_node->gp_index` 相等。如果不相等，则表示设置的属性值无法匹配目标节点，需要返回 EINVAL。
3. 调用 `hwloc__internal_memattr_set_value` 函数，传递给自己的 `hwloc_topology_t` 参数、`hwloc_memattr_id_t` 参数、`target_node->type` 和 `target_node->gp_index` 参数、`initiator` 参数和 `value` 参数。函数的返回值表示设置的属性值是否成功。


```cpp
int
hwloc_memattr_set_value(hwloc_topology_t topology,
                        hwloc_memattr_id_t id,
                        hwloc_obj_t target_node,
                        struct hwloc_location *initiator,
                        unsigned long flags,
                        hwloc_uint64_t value)
{
  struct hwloc_internal_location_s iloc, *ilocp;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  if (initiator) {
    if (to_internal_location(&iloc, initiator) < 0) {
      errno = EINVAL;
      return -1;
    }
    ilocp = &iloc;
  } else {
    ilocp = NULL;
  }

  return hwloc__internal_memattr_set_value(topology, id, target_node->type, target_node->gp_index, target_node->os_index, ilocp, value);
}


```



这段代码是一个静态函数，名为 `hwloc__update_best_target`，它用于更新 `hwloc` 中的最佳目标。该函数的参数包括：

- `best_obj`：指向要更新的 `hwloc_obj_t` 对象的指针。
- `best_value`：保存了最佳目标值的 `hwloc_uint64_t` 类型的指针，最新的值将从这里读取。
- `found`：指示对象是否已经被查找过的标志，初始值为0。
- `new_obj`：新的目标对象的 `hwloc_obj_t` 类型的指针，新的值将从这里读取。
- `new_value`：新的目标值，从读取的 `hwloc_uint64_t` 类型的指针中读取。
- `keep_highest`：指示是否要查找最高的目标，初始值为0。

函数的作用是首先检查是否已找到了最佳目标，如果没有，就设置为新的目标。如果已找到最佳目标，则检查新的目标值是否小于或等于最佳目标值，如果是，则不执行任何操作，直接返回。否则，就更新最佳目标并设置为已找到的状态，最后返回。


```cpp
/**********************
 * Best target
 */

static void
hwloc__update_best_target(hwloc_obj_t *best_obj, hwloc_uint64_t *best_value, int *found,
                          hwloc_obj_t new_obj, hwloc_uint64_t new_value,
                          int keep_highest)
{
  if (*found) {
    if (keep_highest) {
      if (new_value <= *best_value)
        return;
    } else {
      if (new_value >= *best_value)
        return;
    }
  }

  *best_obj = new_obj;
  *best_value = new_value;
  *found = 1;
}

```

This function appears to be a part of the `hwloc` library, which appears to be a tool for managing Linux system hardware resources, such as the CPU and memory.

The function appears to be calculating the "convenience value" of an `IMMATTR` struct, which is a combination of various attributes, such as the default value, the ability to cache, the initiation time, and the lower bound for the value.

The function takes in an `IMMATTR` struct as an argument, and calculates the "convenience value" based on the values of its various attributes. It then updates the `best` target attribute with the calculated `convenience value`, and returns 0 if the calculation was successful, or -1 if it failed.

The function also handles the case where the `IMMATTR` does not have any cached value, and must calculate the "convenience value" from the "no-initiator" value.

Overall, the function seems to be a useful function for calculating the "convenience value" of an `IMMATTR` struct in the context of the `hwloc` library.


```cpp
int
hwloc_memattr_get_best_target(hwloc_topology_t topology,
                              hwloc_memattr_id_t id,
                              struct hwloc_location *initiator,
                              unsigned long flags,
                              hwloc_obj_t *bestp, hwloc_uint64_t *valuep)
{
  struct hwloc_internal_memattr_s *imattr;
  hwloc_uint64_t best_value = 0; /* shutup the compiler */
  hwloc_obj_t best = NULL;
  int found = 0;
  unsigned j;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  imattr = &topology->memattrs[id];

  if (imattr->iflags & HWLOC_IMATTR_FLAG_CONVENIENCE) {
    /* convenience attributes */
    for(j=0; ; j++) {
      hwloc_obj_t node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, j);
      hwloc_uint64_t value;
      if (!node)
        break;
      value = hwloc__memattr_get_convenience_value(id, node);
      hwloc__update_best_target(&best, &best_value, &found,
                                node, value,
                                imattr->flags & HWLOC_MEMATTR_FLAG_HIGHER_FIRST);
    }
    goto done;
  }

  /* normal attributes */

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    /* not strictly need */
    hwloc__imattr_refresh(topology, imattr);

  for(j=0; j<imattr->nr_targets; j++) {
    struct hwloc_internal_memattr_target_s *imtg = &imattr->targets[j];
    hwloc_uint64_t value;
    if (imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR) {
      /* find the initiator and set its value */
      struct hwloc_internal_memattr_initiator_s *imi = hwloc__memattr_get_initiator_from_location(imattr, imtg, initiator);
      if (!imi)
        continue;
      value = imi->value;
    } else {
      /* get the no-initiator value */
      value = imtg->noinitiator_value;
    }
    hwloc__update_best_target(&best, &best_value, &found,
                              imtg->obj, value,
                              imattr->flags & HWLOC_MEMATTR_FLAG_HIGHER_FIRST);
  }

 done:
  if (found) {
    assert(best);
    *bestp = best;
    if (valuep)
      *valuep = best_value;
    return 0;
  } else {
    errno = ENOENT;
    return -1;
  }
}

```



这段代码定义了一个名为 `hwloc__update_best_initiator` 的函数，它的作用是更新系统中最高优先级的内部位置(hwloc_internal_location_s 类型)。

该函数接受四个参数：

- `best_initiator`：要更新的最高优先级的内部位置的引用。
- `best_value`：要更新的值，它的初始值为 `hwloc_uint64_t` 类型，表示优先级最高的内部位置的值。
- `found`：一个指示符，表示是否已经发现最高优先级的内部位置。
- `new_initiator`：新的内部位置，它的类型为 `hwloc_internal_location_s`，表示新的优先级最高的内部位置。
- `new_value`：新的值，它的类型为 `hwloc_uint64_t`，表示新的最高优先级的内部位置的值。
- `keep_highest`：一个指示符，表示是否要保留已经发现的最高优先级的内部位置。

函数的主要逻辑如下：

1. 如果已经发现了最高优先级的内部位置，比较新的内部位置和旧的内部位置的大小，如果新的内部位置比旧的内部位置更大，则不做任何处理，直接返回。

2. 如果新的内部位置比旧的内部位置更大，则将其设置为新的最高优先级的内部位置，并将 `found` 指示符设置为 `1`，表示已经发现了新的最高优先级的内部位置。

3. 如果 `keep_highest` 为 `1`，则只保留已经发现的最高优先级的内部位置，并将 `found` 指示符设置为 `0`。

该函数可以用来设置系统中的最高优先级的内部位置，从而提高系统的性能和稳定性。


```cpp
/**********************
 * Best initiators
 */

static void
hwloc__update_best_initiator(struct hwloc_internal_location_s *best_initiator, hwloc_uint64_t *best_value, int *found,
                             struct hwloc_internal_location_s *new_initiator, hwloc_uint64_t new_value,
                             int keep_highest)
{
  if (*found) {
    if (keep_highest) {
      if (new_value <= *best_value)
        return;
    } else {
      if (new_value >= *best_value)
        return;
    }
  }

  *best_initiator = *new_initiator;
  *best_value = new_value;
  *found = 1;
}

```

This is a function that determines the best memory attribute initiator to use for a particular memory attribute. It takes into account several factors, including the need for an initiator, the value of the attribute, and the cache validity.

The function first checks if the user has specified flags for identifying the attribute. If they have not specified flags, it returns EINVAL. If they have specified flags, it checks if the attribute is greater than or equal to the number of memory attributes at the topology level and returns EINVAL if it is.

Next, the function checks if the attribute is already set as the best attribute. If it is, it returns the value of the attribute and the status of the attribute as a boolean value (0 for success and 1 for failure). If it is not the best attribute, it loops through all the available initiators and updates the best attribute based on the value and a higher first attribute.

If the function finds a valid attribute, it sets the best attribute to the value and the found attribute to 1. If the function finds no valid attributes, it returns ENOENT.

Finally, the function returns the value of the best attribute and the status as a boolean value.


```cpp
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

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  if (id >= topology->nr_memattrs) {
    errno = EINVAL;
    return -1;
  }
  imattr = &topology->memattrs[id];

  if (!(imattr->flags & HWLOC_MEMATTR_FLAG_NEED_INITIATOR)) {
    errno = EINVAL;
    return -1;
  }

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    /* not strictly need */
    hwloc__imattr_refresh(topology, imattr);

  imtg = hwloc__memattr_get_target(imattr, target_node->type, target_node->gp_index, target_node->os_index, 0);
  if (!imtg) {
    errno = EINVAL;
    return -1;
  }

  found = 0;
  for(i=0; i<imtg->nr_initiators; i++) {
    struct hwloc_internal_memattr_initiator_s *imi = &imtg->initiators[i];
    hwloc__update_best_initiator(&best_initiator, &best_value, &found,
                                 &imi->initiator, imi->value,
                                 imattr->flags & HWLOC_MEMATTR_FLAG_HIGHER_FIRST);
  }

  if (found) {
    if (valuep)
      *valuep = best_value;
    return from_internal_location(&best_initiator, bestp);
  } else {
    errno = ENOENT;
    return -1;
  }
}

```

这段代码定义了一个名为 `match_local_obj_cpuset` 的函数，用于匹配输入的 `cpuset` 和一个 `hwloc_obj_t` 类型的目标节点，并返回匹配成功或失败的结果。

该函数接收三个参数：

- `node`：要匹配的目标节点。
- `cpuset`：用于存储目标节点的 CPU 集合。
- `flags`：匹配标志，包括 `HWLOC_LOCAL_NUMANODE_FLAG_ALL`、
`HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY` 和 `HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY`。

函数内部首先检查 `flags` 是否包括 `HWLOC_LOCAL_NUMANODE_FLAG_ALL`，如果是，则直接返回 1，表示匹配成功。然后，分别检查 `flags` 中是否包括 `HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY` 和 `HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY`，如果是，则检查输入的 `cpuset` 是否与目标节点的 CPU 集合匹配。最后，如果所有条件都满足，则返回匹配成功。否则，函数返回失败。


```cpp
/****************************
 * Listing local nodes
 */

static __hwloc_inline int
match_local_obj_cpuset(hwloc_obj_t node, hwloc_cpuset_t cpuset, unsigned long flags)
{
  if (flags & HWLOC_LOCAL_NUMANODE_FLAG_ALL)
    return 1;
  if ((flags & HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY)
      && hwloc_bitmap_isincluded(cpuset, node->cpuset))
    return 1;
  if ((flags & HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY)
      && hwloc_bitmap_isincluded(node->cpuset, cpuset))
    return 1;
  return hwloc_bitmap_isequal(node->cpuset, cpuset);
}

```



This function appears to be a helper function for setting up a hierarchical system using the HWLOC (Highway Information and Location) conventions. It takes in information about the localities of a node and creates a set of nodes based on the specified criteria. The function returns an error code if there are any errors, or a valid set of nodes if all the localities are valid.

The function has several helper functions that it uses to accomplish its tasks. The first is `hwloc_get_obj_by_type`, which returns a handle to an object that matches a given type. The second is `hwloc_get_numanodes`, which returns the number of anonymous nodes in the current topology.

The `hwloc_locality_calculator` function is then used to calculate the localities of the nodes. This function takes in the current topology, the anonymous node flag, and the localities flag and returns a table of localities for each node.

Finally, the `hwloc_set_numanodes` function is used to mark the localities of the nodes. This function takes in the current topology and the anonymous node flag and sets the localities flag for each node to true if it should be marked as "self-hectored" or false if it should be marked as "binned".

If any of these functions fail, the `hwloc_locality_calculator` function will return an error code, and the entire function will return an error code if any of the other functions return an error.


```cpp
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

  if (flags & ~(HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY
                |HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY
                | HWLOC_LOCAL_NUMANODE_FLAG_ALL)) {
    errno = EINVAL;
    return -1;
  }

  if (!nrp || (*nrp && !nodes)) {
    errno = EINVAL;
    return -1;
  }

  if (!location) {
    if (!(flags & HWLOC_LOCAL_NUMANODE_FLAG_ALL)) {
      errno = EINVAL;
      return -1;
    }
    cpuset = NULL; /* unused */

  } else {
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

  *nrp = i;
  return 0;
}


```

这段代码定义了一个名为 `hwloc_memory_tier_s` 的结构体，用于表示硬件内存区域的等级。

这个结构体包含以下字段：

* `node`：一个 `hwloc_obj_t` 类型的成员，用于表示在 hwloc 库中使用的硬件资源名称。
* `local_bw`：一个 `uint64_t` 类型的成员，用于表示与主旋律（也称为静态存储器）相关的带宽大小。
* `type`：一个 `enum hwloc_memory_tier_type_e` 类型的成员，用于表示内存的等级。

`HWLOC_MEMORY_TIER_UNKNOWN` 表示未知的内存等级。`HWLOC_MEMORY_TIER_DRAM` 表示DRAM内存。`HWLOC_MEMORY_TIER_HBM` 表示高速缓存存储器（HBM）内存。`HWLOC_MEMORY_TIER_SPM` 表示特定目的内存（SPM），通常是HBM，我们将在之后使用带宽大小来确认。`HWLOC_MEMORY_TIER_NVM` 表示固态内存（NVM）内存。`HWLOC_MEMORY_TIER_GPU` 表示图形处理器（GPU）内存。

这个结构体可以用于在 hwloc 库中查找、设置和查询与内存相关的信息。例如，如果你想查找一个特定内存区域的带宽大小，你可以使用 `hwloc_get_obj()` 和 `hwloc_object_query()` 函数。


```cpp
/**************************************
 * Using memattrs to identify HBM/DRAM
 */

struct hwloc_memory_tier_s {
  hwloc_obj_t node;
  uint64_t local_bw;
  enum hwloc_memory_tier_type_e {
    /* warning the order is important for guess_memory_tiers() after qsort() */
    HWLOC_MEMORY_TIER_UNKNOWN,
    HWLOC_MEMORY_TIER_DRAM,
    HWLOC_MEMORY_TIER_HBM,
    HWLOC_MEMORY_TIER_SPM, /* Specific-Purpose Memory is usually HBM, we'll use BW to confirm */
    HWLOC_MEMORY_TIER_NVM,
    HWLOC_MEMORY_TIER_GPU,
  } type;
};

```

该函数是一个静态函数，名为 `compare_tiers`，作用是比较两个 `hwloc_memory_tier_s` 结构体，不包含具体的输入参数。

该函数通过以下步骤比较两个 `hwloc_memory_tier_s`：

1. 首先按照 `type` 字段对两个结构体进行排序。
2. 然后按照 `local_bw` 字段对两个结构体进行排序。

如果 `a` 的 `type` 与 `b` 的 `type` 不相同，那么函数返回 `-1`，表明 `a` 对应的 `local_bw` 更小，也就是 `a` 在 `b` 上。
如果 `a` 的 `local_bw` 小于 `b` 的 `local_bw`，那么函数返回 `1`，表明 `a` 对应的 `local_bw` 更大，也就是 `a` 在 `b` 上。
如果 `a` 和 `b` 的 `type` 相同，但 `local_bw` 不相同，那么函数返回 `0`。

总体来说，该函数是用于比较两个 `hwloc_memory_tier_s` 结构体，并根据其 `type` 和 `local_bw` 字段进行排序，返回一个整数，表示哪个结构体在另一个上。


```cpp
static int compare_tiers(const void *_a, const void *_b)
{
  const struct hwloc_memory_tier_s *a = _a, *b = _b;
  /* sort by type of tier first */
  if (a->type != b->type)
    return a->type - b->type;
  /* then by bandwidth */
  if (a->local_bw > b->local_bw)
    return -1;
  else if (a->local_bw < b->local_bw)
    return 1;
  return 0;
}

int
```


This function appears to manage the memory tiers on a system that uses the SPM HBM/GPUMemory HWLOC. It takes in a single integer argument that represents the first tier in the HWLOC hierarchy, and then sets up the HWLOC memory tiers based on that tier.

The function first checks if the first tier is HBM, and if it is, it sets all the HWLOC memory tiers to the corresponding HWLOC memory tier for HBM. If the first tier is SPM or NVM, it sets the corresponding HWLOC memory tier to the corresponding HWLOC memory tier for SPM or NVM.

If the HWLOC memory tier is marked with the subtype "GPUMemory", it sets the subtype to the corresponding subtype for HWLOC memory tiers.

It is important to note that the function assumes that the HWLOC system uses the SPM HBM/GPUMemory HWLOC, and does not handle any other memory tiers or systems that may be used.

I hope this helps! Let me know if you have any further questions.


```cpp
hwloc_internal_memattrs_guess_memory_tiers(hwloc_topology_t topology)
{
  struct hwloc_internal_memattr_s *imattr;
  struct hwloc_memory_tier_s *tiers;
  unsigned i, j, n;
  const char *env;
  int spm_is_hbm = -1; /* -1 will guess from BW, 0 no, 1 forced */
  int mark_dram = 1;
  unsigned first_spm, first_nvm;
  hwloc_uint64_t max_unknown_bw, min_spm_bw;

  env = getenv("HWLOC_MEMTIERS_GUESS");
  if (env) {
    if (!strcmp(env, "none")) {
      return 0;
    } else if (!strcmp(env, "default")) {
      /* nothing */
    } else if (!strcmp(env, "spm_is_hbm")) {
      hwloc_debug("Assuming SPM-tier is HBM, ignore bandwidth\n");
      spm_is_hbm = 1;
    } else if (HWLOC_SHOW_CRITICAL_ERRORS()) {
      fprintf(stderr, "hwloc: Failed to recognize HWLOC_MEMTIERS_GUESS value %s\n", env);
    }
  }

  imattr = &topology->memattrs[HWLOC_MEMATTR_ID_BANDWIDTH];

  if (!(imattr->iflags & HWLOC_IMATTR_FLAG_CACHE_VALID))
    hwloc__imattr_refresh(topology, imattr);

  n = hwloc_get_nbobjs_by_depth(topology, HWLOC_TYPE_DEPTH_NUMANODE);
  assert(n);

  tiers = malloc(n * sizeof(*tiers));
  if (!tiers)
    return -1;

  for(i=0; i<n; i++) {
    hwloc_obj_t node;
    const char *daxtype;
    struct hwloc_internal_location_s iloc;
    struct hwloc_internal_memattr_target_s *imtg = NULL;
    struct hwloc_internal_memattr_initiator_s *imi;

    node = hwloc_get_obj_by_depth(topology, HWLOC_TYPE_DEPTH_NUMANODE, i);
    assert(node);
    tiers[i].node = node;

    /* defaults */
    tiers[i].type = HWLOC_MEMORY_TIER_UNKNOWN;
    tiers[i].local_bw = 0; /* unknown */

    daxtype = hwloc_obj_get_info_by_name(node, "DAXType");
    /* mark NVM, SPM and GPU nodes */
    if (daxtype && !strcmp(daxtype, "NVM"))
      tiers[i].type = HWLOC_MEMORY_TIER_NVM;
    if (daxtype && !strcmp(daxtype, "SPM"))
      tiers[i].type = HWLOC_MEMORY_TIER_SPM;
    if (node->subtype && !strcmp(node->subtype, "GPUMemory"))
      tiers[i].type = HWLOC_MEMORY_TIER_GPU;

    if (spm_is_hbm == -1) {
      for(j=0; j<imattr->nr_targets; j++)
        if (imattr->targets[j].obj == node) {
          imtg = &imattr->targets[j];
          break;
        }
      if (imtg && !hwloc_bitmap_iszero(node->cpuset)) {
        iloc.type = HWLOC_LOCATION_TYPE_CPUSET;
        iloc.location.cpuset = node->cpuset;
        imi = hwloc__memattr_target_get_initiator(imtg, &iloc, 0);
        if (imi)
          tiers[i].local_bw = imi->value;
      }
    }
  }

  /* sort tiers */
  qsort(tiers, n, sizeof(*tiers), compare_tiers);
  hwloc_debug("Sorting memory tiers...\n");
  for(i=0; i<n; i++)
    hwloc_debug("  tier %u = node L#%u P#%u with tier type %d and local BW #%llu\n",
                i,
                tiers[i].node->logical_index, tiers[i].node->os_index,
                tiers[i].type, (unsigned long long) tiers[i].local_bw);

  /* now we have UNKNOWN tiers (sorted by BW), then SPM tiers (sorted by BW), then NVM, then GPU */

  /* iterate over UNKNOWN tiers, and find their BW */
  for(i=0; i<n; i++) {
    if (tiers[i].type > HWLOC_MEMORY_TIER_UNKNOWN)
      break;
  }
  first_spm = i;
  /* get max BW from first */
  if (first_spm > 0)
    max_unknown_bw = tiers[0].local_bw;
  else
    max_unknown_bw = 0;

  /* there are no DRAM or HBM tiers yet */

  /* iterate over SPM tiers, and find their BW */
  for(i=first_spm; i<n; i++) {
    if (tiers[i].type > HWLOC_MEMORY_TIER_SPM)
      break;
  }
  first_nvm = i;
  /* get min BW from last */
  if (first_nvm > first_spm)
    min_spm_bw = tiers[first_nvm-1].local_bw;
  else
    min_spm_bw = 0;

  /* FIXME: if there's more than 10% between some sets of nodes inside a tier, split it? */
  /* FIXME: if there are cpuset-intersecting nodes in same tier, abort? */

  if (spm_is_hbm == -1) {
    /* if we have BW for all SPM and UNKNOWN
     * and all SPM BW are 2x superior to all UNKNOWN BW
     */
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

  if (spm_is_hbm) {
    for(i=0; i<first_spm; i++)
      tiers[i].type = HWLOC_MEMORY_TIER_DRAM;
    for(i=first_spm; i<first_nvm; i++)
      tiers[i].type = HWLOC_MEMORY_TIER_HBM;
  }

  if (first_spm == n)
    mark_dram = 0;

    /* now apply subtypes */
  for(i=0; i<n; i++) {
    const char *type = NULL;
    if (tiers[i].node->subtype) /* don't overwrite the existing subtype */
      continue;
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
      type = "NVM";
      break;
    default:
      /* GPU memory is already marked with subtype="GPUMemory",
       * UNKNOWN doesn't deserve any subtype
       */
      break;
    }
    if (type) {
      hwloc_debug("Marking node L#%u P#%u as %s\n", tiers[i].node->logical_index, tiers[i].node->os_index, type);
      tiers[i].node->subtype = strdup(type);
    }
  }

  free(tiers);
  return 0;
}

```