# xmrig源码解析 30

# `src/3rdparty/hwloc/src/bitmap.c`

这段代码是一个出自于 Inria, Université Bordeaux 和 Cisco Systems, Inc. 的开源项目 Autogen 的代码。该代码的作用是定义了一些函数，被用于生成软件配置文件(.xml)的小事件。

首先，定义了一系列来自不同版权拥有者的版权声明。

然后，引入了 Private/autogen/config.h 和 Private/config.h 头文件，这些头文件中定义了软件配置文件的格式和结构。

接着，定义了一系列常量和变量，包括最后导入的 bitmap.h 头文件，这个头文件中定义了位图，被用于在生成的配置文件中绘制软件包的位置和大小等信息。

最后，定义了一系列函数，包括私人函数 min_packages_to_include，该函数用于确定在软件包中必须包含的最小数量的软件包。private/debug.h 中定义的一些函数，如 is_option_selected 和 generate_option_specs 用于生成选项列表。

最后，通过 Private/config.h 中的函数，将生成的配置文件保存到本地或远程目录中。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2020 Inria.  All rights reserved.
 * Copyright © 2009-2011 Université Bordeaux
 * Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc/autogen/config.h"
#include "hwloc.h"
#include "private/misc.h"
#include "private/private.h"
#include "private/debug.h"
#include "hwloc/bitmap.h"

```

这段代码是一个用于分配和释放高性能内存的库，它提供了数组操作、内存映射、并行编程等功能。以下是这段代码的主要部分：

1. 标准输入输出库（stdio.h）和定义了一些全局函数（assert.h、errno.h、ctype.h）
2. 包含了一个名为hwloc的库，这是高性能内存分配和释放库。
3. 在函数声明中使用了hwloc库的一些功能，包括：
	* 包含了一个stdarg.h头文件，它定义了带参数的函数。
	* 包含了一个printf函数宏，用于输出信息。
	* 包含了一个assert函数，用于在调试时检测错误。
	* 包含了一个errno函数，用于获取当前操作系统获取的错误码。
	* 包含了一个ctype库，它定义了一些用于处理字符串函数的函数。
4. 在函数实现中，定义了一些内部函数，包括：
	* hwloc_bitmap_set_foo：用于设置hwloc库中的bitmap，并传入了两个整数，第一个是初始化大小，第二个是可变的预分配大小。
	* hwloc_core_set_ulong_fd：用于设置hwloc库中的ulong，并传入了两个整数，第一个是ulong的起始位置，第二个是ulong的长度（ulong_empty_first函数保证其中某些ulong是空）。
	* hwloc_core_get_ulong_fd：用于获取hwloc库中的ulong，并传入了一个整数，该函数使用ulong_empty_first函数来保证其中某些ulong是空。
	* hwloc_core_set_unused_ulong：用于设置hwloc库中的ulong，该函数将ulong的下一个未被分配的空间分配给该ulong。
	* hwloc_core_get_unused_ulong：用于获取hwloc库中的ulong，并返回未被分配的ulong数。
	* hwloc_core_hwloc_set_global_fd：用于设置hwloc库中的全局内存分配，并传入了一个整数，该函数使用ulong_empty_first函数来保证其中某些ulong是空。
	* hwloc_core_hwloc_get_global_fd：用于获取hwloc库中的全局内存分配，并返回分配的ulong数。
	* hwloc_core_ulong_empty_first：返回一个bool，表示当前ulong是否是空。
5. 在函数实现中，定义了一些从hwloc库中返回的函数，包括：
	* hwloc_proc_get_mem_align：用于获取内存分配的最小大小（以字节为单位）。
	* hwloc_proc_get_mem_extent：用于获取内存分配的最大大小（以字节为单位）。
	* hwloc_proc_get_mem_size：用于获取内存分配的总大小（以字节为单位）。
	* hwloc_proc_get_total_面包屑：用于获取分配的内存总量（以字节为单位）。
	* hwloc_proc_free：用于释放内存分配，并传入两个整数，第一个是ulong的起始位置，第二个是ulong的长度（ulong_empty_first函数保证其中某些ulong是空）。
	* hwloc_proc_chunk_free：用于从hwloc库中free函数，用于将ulong的起始位置和长度作为参数传递给free函数。
6. 在函数实现中，还定义了一些辅助函数，包括：
	* hwloc_assert_printf：用于在调试时输出printf函数的错误信息。
	* hwloc_assert_errno：用于在调试时获取errno函数的值。
	* hwloc_assert_size：用于在调试时输出hwloc库中函数或变量的大小。

hwloc库在Linux系统上通过/proc/


```cpp
#include <stdarg.h>
#include <stdio.h>
#include <assert.h>
#include <errno.h>
#include <ctype.h>

/*
 * possible improvements:
 * - have a way to change the initial allocation size:
 *   add hwloc_bitmap_set_foo() to changes a global here,
 *   and make the hwloc core call based on the early number of PUs
 * - make HWLOC_BITMAP_PREALLOC_BITS configurable, and detectable
 *   by parsing /proc/cpuinfo during configure on Linux.
 * - preallocate inside the bitmap structure (so that the whole structure is a cacheline for instance)
 *   and allocate a dedicated array only later when reallocating larger
 * - add a bitmap->ulongs_empty_first which guarantees that some first ulongs are empty,
 *   making tests much faster for big bitmaps since there's no need to look at first ulongs.
 *   no need for ulongs_empty_first to be exactly the max number of empty ulongs,
 *   clearing bits that were set earlier isn't very common.
 */

```

这段代码定义了一个名为"magic number"的宏，它的值为0x20091007。接下来定义了两个宏，一个是"hwloc_bitmap_preallocated_bits_"，其值为HWLOC_BITMAP_MAGIC，另一个是"hwloc_bitmap_preallocated_ulongs_"，使用了第一个宏的值，并计算出其所需的位数和分配的位组大小。

接着定义了一个名为"hwloc_bitmap_s"的结构体，其中的"ulongs_count"类型为无符号整数，表示一个位图上有效的位组数量，其计算方法为：ulongs_allocated = HWLOC_BITMAP_PREALLOC_BITS / HWLOC_BITS_PER_LONG，其中"HWLOC_BITMAP_PREALLOC_BITS"是在每个位图上预分配的位数，而"HWLOC_BITMAP_PREALLOC_ULONGS"则是计算预分配位数的整数部分。

该结构体中的"ulongs"是一个指向ulongs类型数组的指针，而"infinite"则是一个布尔值，表示该位图是否可能有超出ulongs_count位数的位。最后，该结构体定义了一个名为"magic"的整数类型，用于存储该宏的值，即0x20091007。


```cpp
/* magic number */
#define HWLOC_BITMAP_MAGIC 0x20091007

/* preallocated bits in every bitmap */
#define HWLOC_BITMAP_PREALLOC_BITS 512
#define HWLOC_BITMAP_PREALLOC_ULONGS (HWLOC_BITMAP_PREALLOC_BITS/HWLOC_BITS_PER_LONG)

/* actual opaque type internals */
struct hwloc_bitmap_s {
  unsigned ulongs_count; /* how many ulong bitmasks are valid, >= 1 */
  unsigned ulongs_allocated; /* how many ulong bitmasks are allocated, >= ulongs_count */
  unsigned long *ulongs;
  int infinite; /* set to 1 if all bits beyond ulongs are set */
#ifdef HWLOC_DEBUG
  int magic;
```

这段代码定义了一个名为HWLOC__BITMAP_CHECK的函数，该函数在debug模式下进行过度检查。它的作用类似于valgrind，但比valgrind更为强大。然而，由于它只是在调试模式下列举检查，所以其实用性可能不如valgrind。

具体来说，HWLOC__BITMAP_CHECK函数接受一个整数set和一个或多个整数作为其参数，然后对set中的整数进行操作。如果使用了整数作为索引，那么函数会检查该整数在set中是否有效，并且确保set中至少有该整数。如果使用了CPU作为索引，那么函数会检查set中对应CPU象素的值是否正确。

该函数的核心部分如下：
```cppc
do {
   assert((set)->magic == HWLOC_BITMAP_MAGIC);
   assert((set)->ulongs_count >= 1);
   assert((set)->ulongs_allocated >= (set)->ulongs_count);
} while (0)
```
在这个核心部分中，首先使用do-while循环，不断检查set中定义的整数是否正确。具体来说，首先检查set中定义的整数是否与HWLOC_BITMAP_MAGIC相等，以及set中是否至少有该整数。然后，对于使用整数作为索引的情况，检查set中对应CPU象素的值是否正确。对于使用CPU作为索引的情况，需要手动解析，这里省略。

最后，在函数定义之后，还有一个未定义的符号#endif，但通常不会对其产生影响，因为它只是一个占位符。


```cpp
#endif
};

/* overzealous check in debug-mode, not as powerful as valgrind but still useful */
#ifdef HWLOC_DEBUG
#define HWLOC__BITMAP_CHECK(set) do {				\
  assert((set)->magic == HWLOC_BITMAP_MAGIC);			\
  assert((set)->ulongs_count >= 1);				\
  assert((set)->ulongs_allocated >= (set)->ulongs_count);	\
} while (0)
#else
#define HWLOC__BITMAP_CHECK(set)
#endif

/* extract a subset from a set using an index or a cpu */
```

这段代码定义了一系列宏，用于在程序中描述硬件内存中的位图。这里简要解释一下每个宏的作用：

1. `HWLOC_SUBBITMAP_INDEX(cpu)`：定义了一个宏，名为`HWLOC_SUBBITMAP_INDEX`，它接受一个`cpu`参数。这个宏计算了将`cpu`分配给`HWLOC_BITS_PER_LONG`后，剩余的低位字节数。

2. `HWLOC_SUBBITMAP_CPU_ULBIT(cpu)`：定义了一个宏，名为`HWLOC_SUBBITMAP_CPU_ULBIT`，它接受一个`cpu`参数。这个宏计算了将`cpu`分配给`HWLOC_BITS_PER_LONG`后，剩余的高位字节数。

3. `HWLOC_SUBBITMAP_READULONG(set,x)`：定义了一个宏，名为`HWLOC_SUBBITMAP_READULONG`，它接受两个参数：`set`和`x`。这个宏读取一个给定的`ulong`位图中的`x`位置的值，并将其存储在`set->ulongs_count`中的对应位置。如果`x`是有效的，这个值将被存储在`set->ulongs`中。如果`x`无效，这个值将被设置为`HWLOC_SUBBITMAP_FULL`。

4. `HWLOC_SUBBITMAP_ZERO`：定义了一个常量，名为`HWLOC_SUBBITMAP_ZERO`，它表示一个`ulong`位图，没有有效位，即`0`。

5. `HWLOC_SUBBITMAP_FULL`：定义了一个常量，名为`HWLOC_SUBBITMAP_FULL`，它表示一个`ulong`位图，包含所有有效位，即`(~0UL)`。

6. `HWLOC_SUBBITMAP_ULBIT(bit)`：定义了一个宏，名为`HWLOC_SUBBITMAP_ULBIT`，它接受一个`bit`参数。这个宏定义了一个`ulong`位图中的位，指定为`bit`时，表示为`1`，否则为`0`。

7. `HWLOC_SUBBITMAP_CPU(cpu)`：定义了一个宏，名为`HWLOC_SUBBITMAP_CPU`，它接受一个`cpu`参数。这个宏将`cpu`的高位字节数转换为对应的`ulong`位图，以便在定义的其他宏中使用。

8. `HWLOC_SUBBITMAP_ULBIT_TO(bit)`：定义了一个宏，名为`HWLOC_SUBBITMAP_ULBIT_TO`，它接受两个参数：`bit`和`end`。这个宏将`HWLOC_SUBBITMAP_ULBIT_FROM(bit)`转换为从`begin`到`end`的等价位图，以指定`bit`为参数。

9. `HWLOC_SUBBITMAP_ULBIT_FROM(begin,end)`：定义了一个宏，名为`HWLOC_SUBBITMAP_ULBIT_FROM`，它接受两个参数：`begin`和`end`。这个宏将`HWLOC_SUBBITMAP_FULL`和`HWLOC_SUBBITMAP_ZERO`的低位字节数映射到从`begin`到`end`的等价位图，指定`ulong`为参数。


```cpp
#define HWLOC_SUBBITMAP_INDEX(cpu)		((cpu)/(HWLOC_BITS_PER_LONG))
#define HWLOC_SUBBITMAP_CPU_ULBIT(cpu)		((cpu)%(HWLOC_BITS_PER_LONG))
/* Read from a bitmap ulong without knowing whether x is valid.
 * Writers should make sure that x is valid and modify set->ulongs[x] directly.
 */
#define HWLOC_SUBBITMAP_READULONG(set,x)	((x) < (set)->ulongs_count ? (set)->ulongs[x] : (set)->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO)

/* predefined subset values */
#define HWLOC_SUBBITMAP_ZERO			0UL
#define HWLOC_SUBBITMAP_FULL			(~0UL)
#define HWLOC_SUBBITMAP_ULBIT(bit)		(1UL<<(bit))
#define HWLOC_SUBBITMAP_CPU(cpu)		HWLOC_SUBBITMAP_ULBIT(HWLOC_SUBBITMAP_CPU_ULBIT(cpu))
#define HWLOC_SUBBITMAP_ULBIT_TO(bit)		(HWLOC_SUBBITMAP_FULL>>(HWLOC_BITS_PER_LONG-1-(bit)))
#define HWLOC_SUBBITMAP_ULBIT_FROM(bit)		(HWLOC_SUBBITMAP_FULL<<(bit))
#define HWLOC_SUBBITMAP_ULBIT_FROMTO(begin,end)	(HWLOC_SUBBITMAP_ULBIT_TO(end) & HWLOC_SUBBITMAP_ULBIT_FROM(begin))

```

这段代码定义了一个名为 `hwloc_bitmap_alloc` 的函数，它的作用是返回一个指向 `struct hwloc_bitmap_s` 类型的指针。这个结构体定义了 bitmap 的相关成员，包括 bitmap 的长度、已分配的内存以及尚可分配的内存。

函数实现包括以下步骤：

1. 定义一个指向设置结构体的指针 `set`；
2. 如果 `set` 成功分配内存，设置 `set->ulongs_count` 为 1，设置 `set->ulongs_allocated` 为 `HWLOC_BITMAP_PREALLOC_ULONGS`，设置 `set->ulongs` 为已分配的内存；
3. 如果 `set->ulongs` 成功分配内存，将 `HWLOC_SUBBITMAP_ZERO` 赋值给 `set->ulongs[0]`，将 `set->infinite` 赋值设置为 0；
4. 返回设置结构体指针 `set`。


```cpp
struct hwloc_bitmap_s * hwloc_bitmap_alloc(void)
{
  struct hwloc_bitmap_s * set;

  set = malloc(sizeof(struct hwloc_bitmap_s));
  if (!set)
    return NULL;

  set->ulongs_count = 1;
  set->ulongs_allocated = HWLOC_BITMAP_PREALLOC_ULONGS;
  set->ulongs = malloc(HWLOC_BITMAP_PREALLOC_ULONGS * sizeof(unsigned long));
  if (!set->ulongs) {
    free(set);
    return NULL;
  }

  set->ulongs[0] = HWLOC_SUBBITMAP_ZERO;
  set->infinite = 0;
```

这两行代码是在定义一个名为HWLOC_DEBUG的预处理指令，如果该指令定义成功，则执行下面这两行代码。这两行代码的功能是设置一个名为set的struct hwloc_bitmap_s类型的变量为一个新的内存分配结构，该结构体定义了一个名为ulongs的联合体，ulongs包含一个名为infinite的整数类型和一个名为ulong的整数类型。

第三行代码是一个函数hwloc_bitmap_alloc_full，它的功能是返回一个名为set的struct hwloc_bitmap_s类型的指针，该指针如果是空，则表示分配成功。函数内部使用hwloc_bitmap_alloc函数为新内存分配结构分配内存，然后设置ulongs的infinite成员变量为1，表示ulongs是一个无限大小的联合体，ulong成员变量设置为HWLOC_SUBBITMAP_FULL，表示ulong指向的内存区域是一个完整的hwloc_bitmap_s结构体，该结构体包含一个无限大小的infinite数组和一个ulong类型的成员ulong。


```cpp
#ifdef HWLOC_DEBUG
  set->magic = HWLOC_BITMAP_MAGIC;
#endif
  return set;
}

struct hwloc_bitmap_s * hwloc_bitmap_alloc_full(void)
{
  struct hwloc_bitmap_s * set = hwloc_bitmap_alloc();
  if (set) {
    set->infinite = 1;
    set->ulongs[0] = HWLOC_SUBBITMAP_FULL;
  }
  return set;
}

```

这段代码定义了一个名为 `hwloc_bitmap_free` 的函数，它属于 `hwloc_bitmap_` 系列的函数，用于释放给定的 `struct hwloc_bitmap_s` 类型的数据。

函数首先检查传入的 `set` 参数是否为 `NULL`，如果是，函数返回，否则会执行后续操作。

接着，函数会检查 `set` 参数中包含的 `ulongs` 数是否为真，如果是，函数会将 `set` 对象中的 `ulongs` 数组指向的内存空间释放掉。然后，函数会释放 `set` 对象本身。

函数的另一个实现部分是一个名为 `hwloc_bitmap_enlarge` 的函数，它的作用是扩大 `set` 参数中的 `ulongs` 数，直到它包含至少 `enough_count` 个 `ulongs` 为止。然后，它将返回扩大后的 `ulongs` 数。


```cpp
void hwloc_bitmap_free(struct hwloc_bitmap_s * set)
{
  if (!set)
    return;

  HWLOC__BITMAP_CHECK(set);
#ifdef HWLOC_DEBUG
  set->magic = 0;
#endif

  free(set->ulongs);
  free(set);
}

/* enlarge until it contains at least needed_count ulongs.
 */
```

这段代码定义了一个名为 `hwloc_bitmap_enlarge_by_ulongs` 的函数，属于 `hwloc_bitmap_s` 结构体的成员函数。

这个函数的作用是动态分配一个比传入的 `set` 结构体中的 `ulongs_allocated` 大 1 的 bitmap，并将这个 bitmap 存储到 `set` 的 `ulongs` 成员中，同时将 `ulongs_allocated` 加 1。

函数的实现中，首先定义了一个名为 `tmp` 的变量，使用了 `hwloc_flsl` 函数，将 `needed_count` 减 1 的结果赋值给 `tmp`。然后，定义了一个名为 `tmpulongs` 的变量，使用了 `realloc` 函数，将 `set` 中的 `ulongs` 成员重新分配，如果分配失败则返回 -1。接着，将 `tmpulongs` 存储到了 `set` 的 `ulongs` 成员中，并将 `ulongs_allocated` 加 1。最后，函数返回 0，表示成功实现了动态扩展。


```cpp
static int
hwloc_bitmap_enlarge_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count) __hwloc_attribute_warn_unused_result;
static int
hwloc_bitmap_enlarge_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count)
{
  unsigned tmp = 1U << hwloc_flsl((unsigned long) needed_count - 1);
  if (tmp > set->ulongs_allocated) {
    unsigned long *tmpulongs;
    tmpulongs = realloc(set->ulongs, tmp * sizeof(unsigned long));
    if (!tmpulongs)
      return -1;
    set->ulongs = tmpulongs;
    set->ulongs_allocated = tmp;
  }
  return 0;
}

```

这段代码定义了一个名为 `hwloc_bitmap_realloc_by_ulongs` 的函数，它的作用是动态地重新分配一个 `hwloc_bitmap_s` 结构中的内存区域，使得该区域中的 `ulongs` 计数至少满足 `needed_count` 这一指标。同时，该函数还更新了 `set` 结构中 `ulongs_count` 变量，以便在释放内存时正确地清除它。

该函数首先检查 `set` 结构中 `ulongs_count` 是否小于等于 `needed_count`，如果是，则直接返回 0。否则，尝试使用 `hwloc_bitmap_enlarge_by_ulongs` 函数将内存区域扩张至可以容纳 `needed_count` 为止，并将 `set` 结构中的 `ulongs_count` 更新为 `needed_count`。如果这个尝试失败了，那么就设置 `ulongs_count` 为 `needed_count`，并将 `set` 结构中 `ulongs_count` 的初始值设置为 `0`，这样在释放内存时就可以正确地清除它了。

最后，该函数返回 0，表示成功释放了内存区域。


```cpp
/* enlarge until it contains at least needed_count ulongs,
 * and update new ulongs according to the infinite field.
 */
static int
hwloc_bitmap_realloc_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count) __hwloc_attribute_warn_unused_result;
static int
hwloc_bitmap_realloc_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count)
{
  unsigned i;

  HWLOC__BITMAP_CHECK(set);

  if (needed_count <= set->ulongs_count)
    return 0;

  /* realloc larger if needed */
  if (hwloc_bitmap_enlarge_by_ulongs(set, needed_count) < 0)
    return -1;

  /* fill the newly allocated subset depending on the infinite flag */
  for(i=set->ulongs_count; i<needed_count; i++)
    set->ulongs[i] = set->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO;
  set->ulongs_count = needed_count;
  return 0;
}

```

这段代码定义了两个名为`hwloc_bitmap_realloc_by_cpu_index`和`hwloc_bitmap_reset_by_ulongs`的函数，它们的作用如下：

1. `hwloc_bitmap_realloc_by_cpu_index`函数：

该函数将一个`hwloc_bitmap_s`结构体中的`set`变量传递给一个名为`cpu`的整数，然后执行以下操作：

- 计算`cpu`在`HWLOC_BITS_PER_LONG`中的偏移量(即 `((cpu)/HWLOC_BITS_PER_LONG)+1`)，然后将其作为参数传递给`hwloc_bitmap_realloc_by_ulongs`函数，作为`set`变量的新大小。
- 如果`set`中的`ulongs_count`计数器小于`cpu`的大小，则递归调用`hwloc_bitmap_realloc_by_ulongs`函数，直到`ulongs_count`计数器等于`cpu`的大小。
- 在函数内部，首先执行`hwloc_bitmap_enlarge_by_ulongs`函数，如果`enlarge_by_ulongs`函数成功，则将`set`变量中的`ulongs_count`计数器设置为`needed_count`。然后调用`hwloc_bitmap_reset_by_ulongs`函数，将其设置为所需的`ulongs_count`大小，并将`hwloc_bitmap_enlarge_by_ulongs`设置为`-1`，表示不会再次尝试调整。

2. `hwloc_bitmap_reset_by_ulongs`函数：

该函数与`hwloc_bitmap_realloc_by_cpu_index`函数正好相反，它的作用是：

- 如果`hwloc_bitmap_realloc_by_ulongs`函数成功，则执行以下操作：

- 将`set`变量中的`ulongs_count`计数器设置为所需的`hwloc_bitmap_enlarge_by_ulongs`函数返回的`ulongs_count`。
- 调用`hwloc_bitmap_reset_by_ulongs`函数，将其设置为所需的`ulongs_count`大小，并将`hwloc_bitmap_enlarge_by_ulongs`设置为`-1`，表示不会再次尝试调整。


```cpp
/* realloc until it contains at least cpu+1 bits */
#define hwloc_bitmap_realloc_by_cpu_index(set, cpu) hwloc_bitmap_realloc_by_ulongs(set, ((cpu)/HWLOC_BITS_PER_LONG)+1)

/* reset a bitmap to exactely the needed size.
 * the caller must reinitialize all ulongs and the infinite flag later.
 */
static int
hwloc_bitmap_reset_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count) __hwloc_attribute_warn_unused_result;
static int
hwloc_bitmap_reset_by_ulongs(struct hwloc_bitmap_s * set, unsigned needed_count)
{
  if (hwloc_bitmap_enlarge_by_ulongs(set, needed_count))
    return -1;
  set->ulongs_count = needed_count;
  return 0;
}

```

这段代码定义了一个名为`hwloc_bitmap_tma_dup`的函数，它的作用是将从给定的`struct hwloc_tma`结构体中调用一个名为`hwloc_bitmap_s`的指针，然后将结果保存到一个新结构体中。

函数的实现包括以下步骤：

1. 检查指针`old`是否为`NULL`，如果是，则直接返回`NULL`。
2. 如果`old`不为`NULL`，则执行以下操作：

a. 从给定的`struct hwloc_tma`结构体中获取`ulongs`成员的起始地址，以及`infinite`标志。

b. 从给定的`struct hwloc_bitmap_s`结构体中创建一个新结构体，并将其`ulongs`成员设置为新结构体中`ulongs_allocated`字段的值，即新结构体中`ulongs`成员的数量乘以每个`ulongs`成员的大小。

c. 将新结构体中的`ulongs`成员复制到新结构体中`ulongs_allocated`字段，并设置为与`old`中`ulongs`成员起始地址相同。

d. 如果`infinite`标志在`old`中为`TRUE`，则设置新结构体中`infinite`标志为`FALSE`，否则将其设置为`INFINITE`。

e. 最后，将新结构体指针返回，如果不返回，则函数结束。


```cpp
/* reset until it contains exactly cpu+1 bits (roundup to a ulong).
 * the caller must reinitialize all ulongs and the infinite flag later.
 */
#define hwloc_bitmap_reset_by_cpu_index(set, cpu) hwloc_bitmap_reset_by_ulongs(set, ((cpu)/HWLOC_BITS_PER_LONG)+1)

struct hwloc_bitmap_s * hwloc_bitmap_tma_dup(struct hwloc_tma *tma, const struct hwloc_bitmap_s * old)
{
  struct hwloc_bitmap_s * new;

  if (!old)
    return NULL;

  HWLOC__BITMAP_CHECK(old);

  new = hwloc_tma_malloc(tma, sizeof(struct hwloc_bitmap_s));
  if (!new)
    return NULL;

  new->ulongs = hwloc_tma_malloc(tma, old->ulongs_allocated * sizeof(unsigned long));
  if (!new->ulongs) {
    free(new);
    return NULL;
  }
  new->ulongs_allocated = old->ulongs_allocated;
  new->ulongs_count = old->ulongs_count;
  memcpy(new->ulongs, old->ulongs, new->ulongs_count * sizeof(unsigned long));
  new->infinite = old->infinite;
```

这段代码是一个 C 语言的函数，定义了 hwloc_bitmap_dup 和 hwloc_bitmap_copy 函数，用于实现 bitmap 的复制。

具体来说，hwloc_bitmap_dup 函数接收一个老 bitmap 和一个新 bitmap，通过调用 hwloc_bitmap_tma_dup 函数，将新 bitmap 初始化为与老 bitmap 相同的 magic 值，然后将老 bitmap 和新 bitmap 连接起来，使得新 bitmap 中的位置与老 bitmap 中的位置对应关系与相同。最后，如果初始化失败或者复制过程出现错误，函数返回 -1。

hwloc_bitmap_copy 函数与 hwloc_bitmap_dup 函数非常类似，只是返回值不同，前者返回新 bitmap，后者返回 void。在复制函数中，同样使用了 hwloc_bitmap_reset_by_ulongs 函数来初始化新 bitmap，并检查是否成功，如果初始化成功，则将老 bitmap 和新 bitmap 中的无限域值复制过去，并设置新 bitmap 的 infinite 域值为 HWLOC_INFINITY，最终返回 0。


```cpp
#ifdef HWLOC_DEBUG
  new->magic = HWLOC_BITMAP_MAGIC;
#endif
  return new;
}

struct hwloc_bitmap_s * hwloc_bitmap_dup(const struct hwloc_bitmap_s * old)
{
  return hwloc_bitmap_tma_dup(NULL, old);
}

int hwloc_bitmap_copy(struct hwloc_bitmap_s * dst, const struct hwloc_bitmap_s * src)
{
  HWLOC__BITMAP_CHECK(dst);
  HWLOC__BITMAP_CHECK(src);

  if (hwloc_bitmap_reset_by_ulongs(dst, src->ulongs_count) < 0)
    return -1;

  memcpy(dst->ulongs, src->ulongs, src->ulongs_count * sizeof(unsigned long));
  dst->infinite = src->infinite;
  return 0;
}

```

这段代码定义了一个名为HWLOC_PRIxSUBBITMAP的宏，它的含义是“%08lx”，其中%08lx表示8个字节，转换为十六进制是0x123456789abcdef。

接着定义了一个名为HWLOC_BITMAP_SUBSTRING_SIZE的宏，它的含义是“32”，表示每个子串的大小为32个字节。

然后定义了一个名为HWLOC_BITMAP_SUBSTRING_LENGTH的宏，它的含义是“(HWLOC_BITMAP_SUBSTRING_SIZE/4)”，表示每个子串的字符数为子串大小的四分之一。

接下来定义了一个名为HWLOC_BITMAP_STRING_PER_LONG的宏，它的含义是“(HWLOC_BITS_PER_LONG/HWLOC_BITMAP_SUBSTRING_SIZE)”，表示每个子串的字符数为子串大小的四分之一，其中HWLOC_BITS_PER_LONG是定义在另外的宏中的。

接着定义了一个名为hwloc_bitmap_snprintf的函数，它的含义是“将set中的值子串化并输出到buf中，输出长度为buflen，buf指针指向set中的第一个值”，其中buflen是参数，set是一个指向struct hwloc_bitmap_s结构体的指针，需要输出的子串的起始位置和长度可以从该结构体中获取。

最后在函数内部，定义了一些整型变量，用于记录子串的字节数、子串长度、字符数组大小、子串中字符的数量、子串中字符的数量、子串长度、子串的字符数组大小、子串的字符串长度等，用于计算输出子串的字节数、从子串中读取的字节数、输出子串的字符数等，以及用于输出子串的字符数组和字符串的指针。

具体的实现是通过定义宏的方式来实现的，通过定义宏的方式来定义函数的参数、函数体等，避免了函数中写入的代码冗长，同时也可以保证函数的函数体一致。


```cpp
/* Strings always use 32bit groups */
#define HWLOC_PRIxSUBBITMAP		"%08lx"
#define HWLOC_BITMAP_SUBSTRING_SIZE	32
#define HWLOC_BITMAP_SUBSTRING_LENGTH	(HWLOC_BITMAP_SUBSTRING_SIZE/4)
#define HWLOC_BITMAP_STRING_PER_LONG	(HWLOC_BITS_PER_LONG/HWLOC_BITMAP_SUBSTRING_SIZE)

int hwloc_bitmap_snprintf(char * __hwloc_restrict buf, size_t buflen, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  ssize_t size = buflen;
  char *tmp = buf;
  int res, ret = 0;
  int needcomma = 0;
  int i;
  unsigned long accum = 0;
  int accumed = 0;
```

This is a C function that appears to calculate the union of two sets of subbitmap locations. It uses the `hwloc` library to handle bitmap operations, and has a function signature with a return type of `int`.

The function takes two arguments: `set1` and `set2`, which are both passed through the `ulongs_count` field of a `ulongset` struct. `set1` should contain the subbitmap locations of the first set, while `set2` should contain the subbitmap locations of the second set.

The function returns the result of the union, or -1 if there was a problem with the input.

The function has several helper functions:

* `+= res;`: adds the `res` value from `set2` to the current accumulator.
* `if (res >= size)`: checks if the result of the union exceeds the size of the `size` field. If it does, the function sets `res` to 0.
* `tmp += res;`: adds the `res` value from `set2` to the `tmp` variable.
* `size -= res;`: subtracts the `res` value from `size` to update `size`.
* `i=(int) set->ulongs_count - 1;`: converts the `ulongs_count` field of the `set` struct to an integer and returns the `i` variable.
* `if (set->infinite)`: checks if the `infinite` field of the `set` struct is true. If it is, the function continues up to the maximum value in `ulongs_count` without filling in any empty bitmap positions.
* `else`: is the catch-all for the infinite loop that would happen if `set->infinite` is false.
* `while (i >= 0 || accum`: is the main loop that will fill the accumulator with the values from `set1` while keeping track of the index `i` with `ulongs_count` fields.
* `accum = set->ulongs_count - i;`: extracts the value from `set1` at index `i` and adds it to the accumulator.
* `if (!accum & accum_mask)`: checks if the accumulator is not empty and can print the result.
* `res = hwloc_snprintf(tmp, size, needcomma ? ",0x" HWLOC_PRIxSUBBITMAP : "0x" HWLOC_PRIxSUBBITMAP, (accum & accum_mask) >> (HWLOC_BITS_PER_LONG - HWLOC_BITMAP_SUBSTRING_SIZE));`: adds the result to the accumulator.
* `needcomma = 1;`: sets the need for a comma to insert a space.
* `res = hwloc_snprintf(tmp, size, ",");`: adds the string "," to the accumulator.
* `ret += res;`: adds the result to the `ret` variable.
* `if (ret < 0)`: checks if the result of the union is negative, and returns it.


```cpp
#if HWLOC_BITS_PER_LONG == HWLOC_BITMAP_SUBSTRING_SIZE
  const unsigned long accum_mask = ~0UL;
#else /* HWLOC_BITS_PER_LONG != HWLOC_BITMAP_SUBSTRING_SIZE */
  const unsigned long accum_mask = ((1UL << HWLOC_BITMAP_SUBSTRING_SIZE) - 1) << (HWLOC_BITS_PER_LONG - HWLOC_BITMAP_SUBSTRING_SIZE);
#endif /* HWLOC_BITS_PER_LONG != HWLOC_BITMAP_SUBSTRING_SIZE */

  HWLOC__BITMAP_CHECK(set);

  /* mark the end in case we do nothing later */
  if (buflen > 0)
    tmp[0] = '\0';

  if (set->infinite) {
    res = hwloc_snprintf(tmp, size, "0xf...f");
    needcomma = 1;
    if (res < 0)
      return -1;
    ret += res;
    if (res >= size)
      res = size>0 ? (int)size - 1 : 0;
    tmp += res;
    size -= res;
  }

  i=(int) set->ulongs_count-1;

  if (set->infinite) {
    /* ignore starting FULL since we have 0xf...f already */
    while (i>=0 && set->ulongs[i] == HWLOC_SUBBITMAP_FULL)
      i--;
  } else {
    /* ignore starting ZERO except the last one */
    while (i>=0 && set->ulongs[i] == HWLOC_SUBBITMAP_ZERO)
      i--;
  }

  while (i>=0 || accumed) {
    /* Refill accumulator */
    if (!accumed) {
      accum = set->ulongs[i--];
      accumed = HWLOC_BITS_PER_LONG;
    }

    if (accum & accum_mask) {
      /* print the whole subset if not empty */
        res = hwloc_snprintf(tmp, size, needcomma ? ",0x" HWLOC_PRIxSUBBITMAP : "0x" HWLOC_PRIxSUBBITMAP,
		     (accum & accum_mask) >> (HWLOC_BITS_PER_LONG - HWLOC_BITMAP_SUBSTRING_SIZE));
      needcomma = 1;
    } else if (i == -1 && accumed == HWLOC_BITMAP_SUBSTRING_SIZE) {
      /* print a single 0 to mark the last subset */
      res = hwloc_snprintf(tmp, size, needcomma ? ",0x0" : "0x0");
    } else if (needcomma) {
      res = hwloc_snprintf(tmp, size, ",");
    } else {
      res = 0;
    }
    if (res < 0)
      return -1;
    ret += res;

```

这段代码是一个名为 `hwloc_printf_collate` 的函数，它用于将本地二进制文件中的字符串与一个指定的子字符串进行合并，并将结果打印到控制台上。它基于两个条件判断，具体解释如下：

1. 如果 `HWLOC_BITS_PER_LONG` 与 `HWLOC_BITMAP_SUBSTRING_SIZE` 相等，那么说明子字符串在本地二进制文件中的长度与在控制台输出中的显示长度相等。此时，代码将累积两个整数 `accum` 和 `accumed`，分别指向从子字符串开始在本地二进制文件中偏移的块和子字符串在控制台输出中显示的块。
2. 如果 `HWLOC_BITS_PER_LONG` 与 `HWLOC_BITMAP_SUBSTRING_SIZE` 不相等，那么说明子字符串在本地二进制文件中的长度与在控制台输出中的显示长度不相等。此时，代码将减少子字符串在控制台输出中的显示长度，并将累计的整数 `accum` 减去显示长度，然后将显示长度与子字符串在控制台输出中的显示长度进行按位与运算，并将结果存回给 `accum`。

接下来，代码将获取子字符串在本地二进制文件中的实际长度，并将其与在控制台输出中的显示长度进行按位与运算，将结果存储回给 `ret` 变量。最后，如果 `ret` 没有被设置，并且 `HWLOC_BITS_PER_LONG` 与 `HWLOC_BITMAP_SUBSTRING_SIZE` 不相等，那么代码会将结果打印为 `0x0`。


```cpp
#if HWLOC_BITS_PER_LONG == HWLOC_BITMAP_SUBSTRING_SIZE
    accum = 0;
    accumed = 0;
#else
    accum <<= HWLOC_BITMAP_SUBSTRING_SIZE;
    accumed -= HWLOC_BITMAP_SUBSTRING_SIZE;
#endif

    if (res >= size)
      res = size>0 ? (int)size - 1 : 0;

    tmp += res;
    size -= res;
  }

  /* if didn't display anything, display 0x0 */
  if (!ret) {
    res = hwloc_snprintf(tmp, size, "0x0");
    if (res < 0)
      return -1;
    ret += res;
  }

  return ret;
}

```

这段代码定义了一个名为 `hwloc_bitmap_asprintf` 的函数，它的参数包括一个指向字符型数据的指针 `strp` 和一个指向 `hwloc_bitmap_s` 结构体的指针 `set`，函数内部根据 `set` 的值执行 bitmap 相应的操作，然后将结果字符串存储到 `buf` 指向的字符型数据中，并将结果字符串存储到 `strp` 指向的字符型数据中。函数的返回值为 -1（表示失败），如果成功，则返回结果字符串的长度。


```cpp
int hwloc_bitmap_asprintf(char ** strp, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  int len;
  char *buf;

  HWLOC__BITMAP_CHECK(set);

  len = hwloc_bitmap_snprintf(NULL, 0, set);
  buf = malloc(len+1);
  if (!buf)
    return -1;
  *strp = buf;
  return hwloc_bitmap_snprintf(buf, len+1, set);
}

```

This is a C function that appears to parse a DOM-based XML string and returns information about the parsed string, including the number of substrings and the presence of certain substrings.

The function takes a single parameter, which is a `count` variable that keeps track of the number of substrings in the parsed string. It also takes two additional parameters, `infinite` and `current`, which are passed to the `strchr()` function to check for the substrings, and a pointer to a `char*` representing the current character of the parsed string, respectively.

The function first initializes the `count` variable to 0 and sets the `infinite` variable to 0. It then enters a while loop that finds substrings in the current character and increments the `count` variable.

If the current character is a null character (`'\0'`), the function immediately sets the `infinite` variable to 0 and returns 0.

If the current character is not a null character, the function then checks if it is a digit between 0 and 9. If it is, the function returns 0. If it is not a digit, the function then uses the `strtoul()` function to convert the character to a numeric value and stores it in the `val` variable. It then updates the `count` variable by shifting the value left by the bitwise value of `(count * HWLOC_BITMAP_SUBSTRING_SIZE) % HWLOC_BITS_PER_LONG` to give it the correct number of bits for the current bitmap.

The function then checks if the current character is a comma or a closing angle-bracket. If it is a comma, the function checks if the next character is a comma or a closing angle-bracket. If it is a closing angle-bracket, the function sets the `infinite` variable to 1 and breaks out of the while loop.

The function also checks if the current character is the first character of the string, in which case the `infinite` variable is set to 1.

The function then returns 0 if the parse was successful, or -1 if the parse failed. If the parse failed, the function also sets the `infinite` variable to 0 to avoid memory corruption.


```cpp
int hwloc_bitmap_sscanf(struct hwloc_bitmap_s *set, const char * __hwloc_restrict string)
{
  const char * current = string;
  unsigned long accum = 0;
  int count=0;
  int infinite = 0;

  /* count how many substrings there are */
  count++;
  while ((current = strchr(current+1, ',')) != NULL)
    count++;

  current = string;
  if (!strncmp("0xf...f", current, 7)) {
    current += 7;
    if (*current != ',') {
      /* special case for infinite/full bitmap */
      hwloc_bitmap_fill(set);
      return 0;
    }
    current++;
    infinite = 1;
    count--;
  }

  if (hwloc_bitmap_reset_by_ulongs(set, (count + HWLOC_BITMAP_STRING_PER_LONG - 1) / HWLOC_BITMAP_STRING_PER_LONG) < 0)
    return -1;
  set->infinite = 0;

  while (*current != '\0') {
    unsigned long val;
    char *next;
    val = strtoul(current, &next, 16);

    assert(count > 0);
    count--;

    accum |= (val << ((count * HWLOC_BITMAP_SUBSTRING_SIZE) % HWLOC_BITS_PER_LONG));
    if (!(count % HWLOC_BITMAP_STRING_PER_LONG)) {
      set->ulongs[count / HWLOC_BITMAP_STRING_PER_LONG] = accum;
      accum = 0;
    }

    if (*next != ',') {
      if (*next || count > 0)
	goto failed;
      else
	break;
    }
    current = (const char*) next+1;
  }

  set->infinite = infinite; /* set at the end, to avoid spurious realloc with filled new ulongs */

  return 0;

 failed:
  /* failure to parse */
  hwloc_bitmap_zero(set);
  return -1;
}

```

该函数的作用是输出给定 bitmap 集合中每个元素的值，并可能输出一个空格分隔多个元素的值。

具体来说，函数接受一个字符指针 buf 和一个字符串长度buflen，用于存储给定的 bitmap 集合。函数首先检查给定的 bitmap 是否为空，如果是，则将字符串中的第一个空格作为结束符，否则，函数将使用 hwloc_bitmap_next 和 hwloc_bitmap_next_unset 函数遍历 bitmap 中的元素，并将它们存储在buf中。

在函数内部，有一个 while 循环，用于读取 bitmap 中的元素，并输出一个空格分隔多个元素的值，如果 end 变量为空，则退出 while 循环。在 while 循环中，首先通过 hwloc_bitmap_next 函数获取给定 bitmap 中的第一个元素，然后判断 end 变量是否为给定元素值后面的空格，如果是，则退出 while 循环。如果 end 变量不为空，则需要输出一个空格分隔多个元素的值，需要输出几个空格取决于给定的元素值。在输出元素值时，使用 hwloc_snprintf 函数将元素值插入到buf中，并使用 needcomma 变量判断是否需要输出一个空格来分隔多个元素。如果 needcomma 变量为 1，则在输出元素值时使用 ", " 分隔多个元素。

函数还使用一个 while 循环来跟踪已经输出的元素数量，如果该数量大于缓冲区大小，则返回 0。


```cpp
int hwloc_bitmap_list_snprintf(char * __hwloc_restrict buf, size_t buflen, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  int prev = -1;
  ssize_t size = buflen;
  char *tmp = buf;
  int res, ret = 0;
  int needcomma = 0;

  HWLOC__BITMAP_CHECK(set);

  /* mark the end in case we do nothing later */
  if (buflen > 0)
    tmp[0] = '\0';

  while (1) {
    int begin, end;

    begin = hwloc_bitmap_next(set, prev);
    if (begin == -1)
      break;
    end = hwloc_bitmap_next_unset(set, begin);

    if (end == begin+1) {
      res = hwloc_snprintf(tmp, size, needcomma ? ",%d" : "%d", begin);
    } else if (end == -1) {
      res = hwloc_snprintf(tmp, size, needcomma ? ",%d-" : "%d-", begin);
    } else {
      res = hwloc_snprintf(tmp, size, needcomma ? ",%d-%d" : "%d-%d", begin, end-1);
    }
    if (res < 0)
      return -1;
    ret += res;

    if (res >= size)
      res = size>0 ? (int)size - 1 : 0;

    tmp += res;
    size -= res;
    needcomma = 1;

    if (end == -1)
      break;
    else
      prev = end - 1;
  }

  return ret;
}

```

这段代码定义了一个名为 `hwloc_bitmap_list_asprintf` 的函数，它的作用是将从 `set` 结构体中取得的 `hwloc_bitmap_s` 类型的数据，以其作为参数的 `strp` 字符指针中，每三个字节复制一个 `hwloc_bitmap_s` 类型的数据，并将其复制到 `buf` 数组中。

函数首先检查 `set` 是否为 `NULL`，如果是，则表示 `set` 的所有成员都是未定义的，函数将无法完成复制操作，因此函数将返回 `-1`。否则，函数将使用 `hwloc_bitmap_list_snprintf` 函数将从 `set` 中取得的第一个 `hwloc_bitmap_s` 类型的数据开始，每三个字节复制一个该数据，并将其存储到 `buf` 数组的第一个元素中。最后，函数将 `buf` 数组的起始地址传递给 `strp` 参数，以便将复制出来的数据正确地赋值给 `strp`。


```cpp
int hwloc_bitmap_list_asprintf(char ** strp, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  int len;
  char *buf;

  HWLOC__BITMAP_CHECK(set);

  len = hwloc_bitmap_list_snprintf(NULL, 0, set);
  buf = malloc(len+1);
  if (!buf)
    return -1;
  *strp = buf;
  return hwloc_bitmap_list_snprintf(buf, len+1, set);
}

```

该函数的作用是读取一个字符串，将其转换为整数，并返回一个整数表示成功的次数。它接受一个hwloc_bitmap_s结构体类型的set和一个以字符串开始的指针。在函数内部，它首先初始化了一个变量begin和一个变量val，都初始化为-1。然后，它循环将字符串中的每个字符作为参数，并将其转换为整数。接下来，它根据参数中的字符串是否为空，以及是否已经完成了整个范围，来决定是否继续循环。如果当前字符为空，则代表已经完成了整个范围，函数将执行下一个操作。如果当前字符为'-'，则表示开始了一个新的范围。在范围内的字符，函数将其设置为当前的整数值，并检查它是否可以完成该范围。如果是，则函数将设置begin为当前的整数值，并检查它是否可以完成该范围。如果是，则函数将跳过该范围，继续循环。如果当前字符为' '或'\0'，则表示该字符是一个有效的范围结束符，函数将跳过该范围并继续循环。如果该函数在循环中遇到失败，则将函数set的hwloc_bitmap_zero函数调用，并返回-1。


```cpp
int hwloc_bitmap_list_sscanf(struct hwloc_bitmap_s *set, const char * __hwloc_restrict string)
{
  const char * current = string;
  char *next;
  long begin = -1, val;

  hwloc_bitmap_zero(set);

  while (*current != '\0') {

    /* ignore empty ranges */
    while (*current == ',' || *current == ' ')
      current++;

    val = strtoul(current, &next, 0);
    /* make sure we got at least one digit */
    if (next == current)
      goto failed;

    if (begin != -1) {
      /* finishing a range */
      if (hwloc_bitmap_set_range(set, begin, val) < 0)
        goto failed;
      begin = -1;

    } else if (*next == '-') {
      /* starting a new range */
      if (*(next+1) == '\0') {
	/* infinite range */
	if (hwloc_bitmap_set_range(set, val, -1) < 0)
	  goto failed;
        break;
      } else {
	/* normal range */
	begin = val;
      }

    } else if (*next == ',' || *next == ' ' || *next == '\0') {
      /* single digit */
      hwloc_bitmap_set(set, val);
    }

    if (*next == '\0')
      break;
    current = next+1;
  }

  return 0;

 failed:
  /* failure to parse */
  hwloc_bitmap_zero(set);
  return -1;
}

```

这段代码是一个名为 `hwloc_bitmap_taskset_snprintf` 的函数，它的作用是输出一个 `hwloc_bitmap_s` 类型的结构体中的设置，并将结果存储到给定的 `char *` 指针 `buf` 中。

函数接受两个参数：一个 `char *` 指针 `buf`，它的长度为 `buflen`；一个 `struct hwloc_bitmap_s` 类型的参数 `set`，该参数是一个指向 `hwloc_bitmap_s` 的指针。

函数内部首先定义了一些整数变量，包括 `size`、`tmp`、`res`、`ret`、`started` 和 `i`，用于跟踪输出缓冲区的剩余长度、设置中的 `ulongs_count` 值、设置中 `ulongs` 数组的当前元素值、已输出部分的长度以及设置中 `ulongs_count` 的后一个元素值等。

函数的核心部分是判断 `set` 是否为 `NULL`，如果是，则输出一个特殊字符 `0xf...f`，表示整个 `set` 中的所有元素都是 `HWLOC_SUBBITMAP_FULL`，并将 `res` 和 `ret` 累加为输入缓冲区的长度。如果不是，则执行以下步骤：

1. 如果 `set` 中的元素都是 `HWLOC_SUBBITMAP_FULL`，则输出一个字符 `HWLOC_SUBBITMAP_FULL`，然后将 `res` 和 `ret` 累加为 `size`，并将 `i` 减去 1，以跳过当前的 `ulongs_count` 元素。
2. 如果 `set` 中的元素都是 `HWLOC_SUBBITMAP_ZERO`，则输出一个字符 `HWLOC_SUBBITMAP_ZERO`，然后将 `res` 和 `ret` 累加为 `size`，并将 `i` 减去 1，以跳过当前的 `ulongs_count` 元素。
3. 如果 `set` 中的元素既有 `HWLOC_SUBBITMAP_FULL` 又有 `HWLOC_SUBBITMAP_ZERO`，则先输出一个字符 `HWLOC_SUBBITMAP_FULL`，然后输出一个字符 `HWLOC_SUBBITMAP_ZERO`，最后将 `res` 和 `ret` 累加为 `size`，并将 `i` 减去 1，以跳过当前的 `ulongs_count` 元素。注意，如果 `set` 中包含一个 `HWLOC_SUBBITMAP_FULL` 和一个 `HWLOC_SUBBITMAP_ZERO`，则只会在第一次输出 `HWLOC_SUBBITMAP_FULL`。
4. 如果 `set` 不是 `NULL`，则一直执行步骤 2-3，直到输出所有元素或者达到缓冲区末尾（即 `buflen`）。


```cpp
int hwloc_bitmap_taskset_snprintf(char * __hwloc_restrict buf, size_t buflen, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  ssize_t size = buflen;
  char *tmp = buf;
  int res, ret = 0;
  int started = 0;
  int i;

  HWLOC__BITMAP_CHECK(set);

  /* mark the end in case we do nothing later */
  if (buflen > 0)
    tmp[0] = '\0';

  if (set->infinite) {
    res = hwloc_snprintf(tmp, size, "0xf...f");
    started = 1;
    if (res < 0)
      return -1;
    ret += res;
    if (res >= size)
      res = size>0 ? (int)size - 1 : 0;
    tmp += res;
    size -= res;
  }

  i=set->ulongs_count-1;

  if (set->infinite) {
    /* ignore starting FULL since we have 0xf...f already */
    while (i>=0 && set->ulongs[i] == HWLOC_SUBBITMAP_FULL)
      i--;
  } else {
    /* ignore starting ZERO except the last one */
    while (i>=1 && set->ulongs[i] == HWLOC_SUBBITMAP_ZERO)
      i--;
  }

  while (i>=0) {
    unsigned long val = set->ulongs[i--];
    if (started) {
      /* print the whole subset */
```

这段代码是一个 C 语言函数，名为 `hwloc_format_timestamp`，它的作用是格式化一个 long 类型的时间戳（64 位）为字符串，并输出。具体实现如下：

1. 首先判断输入的 long 类型是否为 64 位，如果是，就执行以下操作：

  a. 如果已经输出过字符串，就直接输出，并将输入的 long 类型值赋给 `res`。

  b. 否则，输出字符串的前 8 位，并将输入的 long 类型值赋给 `res`。

2. 如果输入的 long 类型不是 64 位，就执行以下操作：

  a. 如果已经输出过字符串，就直接输出，并将输入的 long 类型值赋给 `res`，并将 `started` 赋为 1。

  b. 否则，输出字符串的前 16 位，并将输入的 long 类型值赋给 `res`，并将 `started` 赋为 0。

3. 如果输入的 long 类型为 -1 或没有输出任何字符串，就执行以下操作：

  a. 如果已经输出过字符串，就直接输出，并将输入的 long 类型值赋给 `res`，并将 `started` 赋为 1。

  b. 否则，输出字符串的前 16 位，并将输入的 long 类型值赋给 `res`，然后将 `res` 的值赋为 0，并将 `size` 减去 `res` 的大小。

4. 如果输出字符串的当前计数值小于 0，就返回 -1，否则，将当前计数值累加到 `ret`，并将 `res` 的值乘以 16，并将 `size` 减去 `res` 的大小。


```cpp
#if HWLOC_BITS_PER_LONG == 64
      res = hwloc_snprintf(tmp, size, "%016lx", val);
#else
      res = hwloc_snprintf(tmp, size, "%08lx", val);
#endif
    } else if (val || i == -1) {
      res = hwloc_snprintf(tmp, size, "0x%lx", val);
      started = 1;
    } else {
      res = 0;
    }
    if (res < 0)
      return -1;
    ret += res;
    if (res >= size)
      res = size>0 ? (int)size - 1 : 0;
    tmp += res;
    size -= res;
  }

  /* if didn't display anything, display 0x0 */
  if (!ret) {
    res = hwloc_snprintf(tmp, size, "0x0");
    if (res < 0)
      return -1;
    ret += res;
  }

  return ret;
}

```

这段代码定义了一个名为 `hwloc_bitmap_taskset_asprintf` 的函数，它的参数为指向字符型数组 `strp` 和一个指向 `hwloc_bitmap_s` 类型的 `set` 结构体。

这个函数的作用是将从 `set` 结构体中提取出一些字符串信息，并将它们打印成字符型数组 `strp` 中。

具体来说，函数首先检查 `set` 是否为 `NULL`，如果是，则调用 `hwloc_bitmap_taskset_snprintf` 函数打印字符串信息，并将结果存储在 `buf` 指向的字符型数组中。然后，函数将 `buf` 指向的字符型数组中的字符数组复制到 `strp` 指向的字符型数组中，并返回 `0`。

如果 `set` 不是 `NULL`，则首先调用 `hwloc_bitmap_taskset_snprintf` 函数打印 `set` 结构体中的所有字符串信息，并将其存储在 `buf` 指向的字符型数组中。然后，函数将从 `set` 结构体中提取出字符串信息后的字符数组复制到 `buf` 指向的字符型数组中，并调用 `hwloc_bitmap_taskset_snprintf` 函数将提取出的字符串信息打印成字符型数组。最后，函数将 `buf` 指向的字符型数组中的字符数组复制到 `strp` 指向的字符型数组中，并返回 `0`。


```cpp
int hwloc_bitmap_taskset_asprintf(char ** strp, const struct hwloc_bitmap_s * __hwloc_restrict set)
{
  int len;
  char *buf;

  HWLOC__BITMAP_CHECK(set);

  len = hwloc_bitmap_taskset_snprintf(NULL, 0, set);
  buf = malloc(len+1);
  if (!buf)
    return -1;
  *strp = buf;
  return hwloc_bitmap_taskset_snprintf(buf, len+1, set);
}

```

This function appears to parse an XML input file and store its contents in a bitmap. It takes as input an optional file path to the XML file and an optional array of bitmap names to use for the bitmap.

The function reads the input file and parses it into an array of bitmap names. It then initializes the bitmap with a default value or a special case for an infinite bitmap.

The function supports two cases for parsing the input:

1. An empty bitmap, which is represented by a special case for an infinite bitmap.
2. An input that contains only a limited number of characters. This case is represented by a special case for an infinite bitmap.

For each case, the function reads the input into a bitmap and updates the current position in the input file. It then continues reading the input file and parses it into the bitmap.

If the input file cannot be parsed successfully, the function returns -1. If the input is an empty bitmap, the function initializes the bitmap with all zeros and returns 0. If the input is an infinite bitmap, the function returns the current value in the bitmap.


```cpp
int hwloc_bitmap_taskset_sscanf(struct hwloc_bitmap_s *set, const char * __hwloc_restrict string)
{
  const char * current = string;
  int chars;
  int count;
  int infinite = 0;

  if (!strncmp("0xf...f", current, 7)) {
    /* infinite bitmap */
    infinite = 1;
    current += 7;
    if (*current == '\0') {
      /* special case for infinite/full bitmap */
      hwloc_bitmap_fill(set);
      return 0;
    }
  } else {
    /* finite bitmap */
    if (!strncmp("0x", current, 2))
      current += 2;
    if (*current == '\0') {
      /* special case for empty bitmap */
      hwloc_bitmap_zero(set);
      return 0;
    }
  }
  /* we know there are other characters now */

  chars = (int)strlen(current);
  count = (chars * 4 + HWLOC_BITS_PER_LONG - 1) / HWLOC_BITS_PER_LONG;

  if (hwloc_bitmap_reset_by_ulongs(set, count) < 0)
    return -1;
  set->infinite = 0;

  while (*current != '\0') {
    int tmpchars;
    char ustr[17];
    unsigned long val;
    char *next;

    tmpchars = chars % (HWLOC_BITS_PER_LONG/4);
    if (!tmpchars)
      tmpchars = (HWLOC_BITS_PER_LONG/4);

    memcpy(ustr, current, tmpchars);
    ustr[tmpchars] = '\0';
    val = strtoul(ustr, &next, 16);
    if (*next != '\0')
      goto failed;

    set->ulongs[count-1] = val;

    current += tmpchars;
    chars -= tmpchars;
    count--;
  }

  set->infinite = infinite; /* set at the end, to avoid spurious realloc with filled new ulongs */

  return 0;

 failed:
  /* failure to parse */
  hwloc_bitmap_zero(set);
  return -1;
}

```

这两段代码都是定义在名为 "hwloc_bitmap" 的结构体函数中，它们的目的是相同的作用，即在 bitmap 内部元素为 0，并设置该 bitmap 的无限值为 0。

第一段代码 "hwloc_bitmap__zero" 函数接收一个指向 bitmap 结构体的指针 set，在 bitmap 的 ulongs 数组中为 0，同时将无限值 0 设置为 set->infinite 的值，最后将无限值 0 设置为 set->ulongs_count 的值，这样就实现了 bitmap 内部元素为 0，无限值 0 的设置。

第二段代码 "hwloc_bitmap_zero" 函数与第一段代码类似，但它的作用范围更小，仅仅是在设置的 bitmap 内部，而不是整个 bitmap 中。函数接收一个指向 bitmap 结构体的指针 set，首先检查是否已经将 bitmap 中的所有元素都设置为 0，如果是，则忽略了这一步，否则执行 bitmap__zero 函数，将 bitmap 中的元素都设置为 0，最后将无限值 0 设置为 set->ulongs_count 的值，这样就实现了 bitmap 内部元素为 0，无限值 0 的设置。


```cpp
static void hwloc_bitmap__zero(struct hwloc_bitmap_s *set)
{
	unsigned i;
	for(i=0; i<set->ulongs_count; i++)
		set->ulongs[i] = HWLOC_SUBBITMAP_ZERO;
	set->infinite = 0;
}

void hwloc_bitmap_zero(struct hwloc_bitmap_s * set)
{
	HWLOC__BITMAP_CHECK(set);

	HWLOC_BUILD_ASSERT(HWLOC_BITMAP_PREALLOC_ULONGS >= 1);
	if (hwloc_bitmap_reset_by_ulongs(set, 1) < 0) {
		/* cannot fail since we preallocate some ulongs.
		 * if we ever preallocate nothing, we'll reset to 0 ulongs.
		 */
	}
	hwloc_bitmap__zero(set);
}

```



这段代码定义了两个函数，一个是 `hwloc_bitmap_fill`，另一个是 `hwloc_bitmap__fill`。这两个函数都在 `hwloc_bitmap_s` 结构中使用。

`hwloc_bitmap_fill` 函数接受一个 `struct hwloc_bitmap_s` 类型的参数 `set`，并执行以下操作：

1. 通过 `HWLOC_BUILD_ASSERT` 函数检查 `set` 是否已经定义。如果是，则继续执行以下操作：

 1. 如果 `set` 中 `ulongs_count` 成员的值小于 1，则执行以下操作：

   - `set->ulongs_count` 替换为 `ulongs_count` 成员的值。
   - 对于 `i` 从 0 到 `ulongs_count - 1` 的循环，设置 `set->ulongs[i]` 为 `HWLOC_SUBBITMAP_FULL`。
   - 将 `set->infinite` 设置为 1，以便在填充操作后通知 CPU 总线不再需要竞争条件。

2. 如果 `set` 中 `ulongs_count` 成员的值大于等于 1，则执行以下操作：

 1. 如果 `hwloc_bitmap_reset_by_ulongs` 函数在设置 `set->ulongs_count` 时失败，则执行以下操作：

   - 如果 `hwloc_bitmap_reset_by_ulongs` 函数成功，则执行以下操作：

     - 调用 `hwloc_bitmap__fill` 函数。

     - 如果 `hwloc_bitmap__fill` 函数成功填充所有 `ulongs` ，则设置 `set->ulongs_count` 为 `ulongs_count`，并将 `set->infinite` 设置为 0。

     - 如果 `hwloc_bitmap__fill` 函数失败填充任何 `ulongs` ，则执行以下操作：

       - 如果 `hwloc_bitmap_reset_by_ulongs` 函数在设置 `set->ulongs_count` 时失败，则执行以下操作：

         - `set->ulongs_count` 替换为 1。
         - `set->infinite` 设置为 1。

         - 不需要再执行 `hwloc_bitmap__fill` 函数，因为已经尝试过一次并失败了。

2. 填充完所有 `ulongs` 后，设置 `set->infinite` 为 0，以便通知 CPU 总线它已经准备好执行后续操作。

`hwloc_bitmap__fill` 函数与 `hwloc_bitmap_fill` 函数的作用是相同的，都是创建一个 `hwloc_bitmap_s` 类型的结构，并执行静态初始化。


```cpp
static void hwloc_bitmap__fill(struct hwloc_bitmap_s * set)
{
	unsigned i;
	for(i=0; i<set->ulongs_count; i++)
		set->ulongs[i] = HWLOC_SUBBITMAP_FULL;
	set->infinite = 1;
}

void hwloc_bitmap_fill(struct hwloc_bitmap_s * set)
{
	HWLOC__BITMAP_CHECK(set);

	HWLOC_BUILD_ASSERT(HWLOC_BITMAP_PREALLOC_ULONGS >= 1);
	if (hwloc_bitmap_reset_by_ulongs(set, 1) < 0) {
		/* cannot fail since we pre-allocate some ulongs.
		 * if we ever pre-allocate nothing, we'll reset to 0 ulongs.
		 */
	}
	hwloc_bitmap__fill(set);
}

```

这段代码定义了一个名为 `hwloc_bitmap_from_ulong` 的函数，它接受一个 `struct hwloc_bitmap_s` 类型的输入参数 `set` 和一个 `unsigned long` 类型的输入参数 `mask`。函数的作用是返回一个整数，表示是否成功从 `set` 中初始化了一个 `hwloc_bitmap_s` 类型的掩码 `mask`。

函数首先检查输入参数 `set` 是否已经被初始化，如果没有被初始化，函数会尝试使用 `hwloc_bitmap_reset_by_ulongs` 函数将掩码初始化为 1。如果这个函数失败，函数会让 `set` 的 `ulongs` 成员为 0，并将 `infinite` 设置为 0，这样函数就不会返回任何值。

如果 `hwloc_bitmap_reset_by_ulongs` 函数成功将掩码初始化为 1，那么函数就会创建一个新的掩码，并将其设置为输入参数 `mask`。然后，函数会将 `set` 的 `ulongs` 成员设置为掩码，并将 `infinite` 设置为 0。最后，函数返回 0，表示成功初始化了掩码。


```cpp
int hwloc_bitmap_from_ulong(struct hwloc_bitmap_s *set, unsigned long mask)
{
	HWLOC__BITMAP_CHECK(set);

	HWLOC_BUILD_ASSERT(HWLOC_BITMAP_PREALLOC_ULONGS >= 1);
	if (hwloc_bitmap_reset_by_ulongs(set, 1) < 0) {
		/* cannot fail since we pre-allocate some ulongs.
		 * if ever pre-allocate nothing, we may have to return a failure.
		 */
	}
	set->ulongs[0] = mask; /* there's always at least one ulong allocated */
	set->infinite = 0;
	return 0;
}

```

这段代码是一个名为 `hwloc_bitmap_from_ith_ulong` 的函数，它接受一个 `struct hwloc_bitmap_s` 类型的输入参数 `set`，一个 `unsigned long` 类型的输入参数 `mask`，以及一个 `unsigned i` 类型的输入参数 `i`。

函数的作用是将从 `set` 的第二个元素（索引为 `i+1`）开始的 `i` 个 `unsigned long` 类型的输入值设置为给定的 `mask`，并将 `set` 中所有元素的值都设置为 `0`。如果设置成功，函数返回 `0`，否则返回 `-1`。


```cpp
int hwloc_bitmap_from_ith_ulong(struct hwloc_bitmap_s *set, unsigned i, unsigned long mask)
{
	unsigned j;

	HWLOC__BITMAP_CHECK(set);

	if (hwloc_bitmap_reset_by_ulongs(set, i+1) < 0)
		return -1;

	set->ulongs[i] = mask;
	for(j=0; j<i; j++)
		set->ulongs[j] = HWLOC_SUBBITMAP_ZERO;
	set->infinite = 0;
	return 0;
}

```

这段代码是一个名为 `hwloc_bitmap_from_ulongs` 的函数，它接受一个 `struct hwloc_bitmap_s` 类型的输入参数 `set`，以及一个 `unsigned long` 类型的输入参数 `masks`，用于指定映射到每个 `set` 中的 `ulongs` 的子集。

函数的主要作用是初始化一个 `hwloc_bitmap_s` 类型的输出结构体 `set`，其中 `set` 的 `ulongs` 成员被初始化为输入参数 `masks` 中的每个元素，并且 `set` 的 `infinite` 标志将被设置为 `0`。

具体实现过程如下：

1. 首先检查输入参数 `set` 是否已经设置好，如果设置失败，则返回 `-1`，表示函数无法完成初始化。
2. 如果 `set` 已经设置好，那么就调用 `hwloc_bitmap_reset_by_ulongs` 函数，这个函数接受一个 `struct hwloc_bitmap_s` 类型的输入参数 `set` 和一个 `unsigned long` 类型的输入参数 `masks`，它将 `set` 中的 `ulongs` 成员设置为 `masks` 中的每个元素，并将 `set` 的 `infinite` 标志设置为 `0`。
3. 接着，程序进入一个循环，依次设置 `set` 中的 `ulongs` 成员为 `masks` 中的每个元素。
4. 循环结束后，检查 `set` 是否设置好了，如果没有设置好，则执行一些额外的操作（这里略去），并将 `set` 的 `infinite` 标志设置为 `0`，表示函数已经完成初始化。
5. 如果 `set` 设置好了，函数返回 `0`，表示初始化成功。


```cpp
int hwloc_bitmap_from_ulongs(struct hwloc_bitmap_s *set, unsigned nr, const unsigned long *masks)
{
	unsigned j;

	HWLOC__BITMAP_CHECK(set);

	if (hwloc_bitmap_reset_by_ulongs(set, nr) < 0)
		return -1;

	for(j=0; j<nr; j++)
		set->ulongs[j] = masks[j];
	set->infinite = 0;
	return 0;
}

```

这段代码定义了 hwloc_bitmap_to_ulong、hwloc_bitmap_to_ith_ulong 和 hwloc_bitmap_to_ulongs 三个函数，它们都是 hwloc_bitmap_t 结构体的成员函数。

hwloc_bitmap_to_ulong 的作用是将给定的 bitmap 中的每个元素的值转换为 long 类型，并返回该 bitmap 的第一个 long 成员。

hwloc_bitmap_to_ith_ulong 的作用是将给定的 bitmap 中的每个元素的值转换为 unsigned long 类型，并返回该 bitmap 中的对应元素的最高位(即 0)。

hwloc_bitmap_to_ulongs 的作用是将给定的 bitmap 中的每个元素的值转换为 unsigned long 类型，并返回该 bitmap 中的所有元素的最高位(即 0)。

这些函数中，都使用了 hwloc_bitmap_check 函数来检查输入是否为有效的 bitmap。另外，这些函数中的内部变量 i、masks 都使用了unsigned long 类型。


```cpp
unsigned long hwloc_bitmap_to_ulong(const struct hwloc_bitmap_s *set)
{
	HWLOC__BITMAP_CHECK(set);

	return set->ulongs[0]; /* there's always at least one ulong allocated */
}

unsigned long hwloc_bitmap_to_ith_ulong(const struct hwloc_bitmap_s *set, unsigned i)
{
	HWLOC__BITMAP_CHECK(set);

	return HWLOC_SUBBITMAP_READULONG(set, i);
}

int hwloc_bitmap_to_ulongs(const struct hwloc_bitmap_s *set, unsigned nr, unsigned long *masks)
{
	unsigned j;

	HWLOC__BITMAP_CHECK(set);

	for(j=0; j<nr; j++)
		masks[j] = HWLOC_SUBBITMAP_READULONG(set, j);
	return 0;
}

```

这两段代码是用于在 hwloc_bitmap_s 结构体中实现对位图的只使用部分元素功能的函数。具体来说：

1. int hwloc_bitmap_nr_ulongs(const struct hwloc_bitmap_s *set) - 返回一个整数，表示位图设置中可用的long数量。这个函数的作用是检查 set 结构体中所有 long 类型的成员是否都为无限（infinite），如果是，那么返回 -1，表示没有可用的long数量。

2. int hwloc_bitmap_only(struct hwloc_bitmap_s *set, unsigned cpu) - 返回一个整数，表示在给定 cpu 值上，位图设置中可用的long数量。这个函数的作用是在 set 结构体中找到对应于 cpu 的子位图，然后将该子位图中的所有long置为0，最后返回得到的long数量。注意，这个函数需要显式调用，而不是在主函数中直接调用。


```cpp
int hwloc_bitmap_nr_ulongs(const struct hwloc_bitmap_s *set)
{
	unsigned last;

	HWLOC__BITMAP_CHECK(set);

	if (set->infinite)
		return -1;

	last = hwloc_bitmap_last(set);
	return (last + HWLOC_BITS_PER_LONG)/HWLOC_BITS_PER_LONG;
}

int hwloc_bitmap_only(struct hwloc_bitmap_s * set, unsigned cpu)
{
	unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

	HWLOC__BITMAP_CHECK(set);

	if (hwloc_bitmap_reset_by_cpu_index(set, cpu) < 0)
		return -1;

	hwloc_bitmap__zero(set);
	set->ulongs[index_] |= HWLOC_SUBBITMAP_CPU(cpu);
	return 0;
}

```

这两函数的作用是管理一个多路复用（hwloc_bitmap_）结构数组，该数组包含多个子映射，每个子映射对应一个特定的CPU。多路复用结构中，有2个成员变量：set和cpu，分别表示整个多路复用的设置和当前设置的CPU。通过这两个成员变量，可以设置或获取多路复用的子映射，以达到设置或获取CPU对应子映射的目的。

具体来说，这两个函数分别实现了以下功能：
1. hwloc_bitmap_allbut：设置一个给定的多路复用结构数组中所有非CPU指定的子映射。这个函数接受一个结构体指针set和当前CPU作为参数，然后执行一系列的检查，如检查set是否为空，以及当前CPU是否在多路复用结构中，然后通过hwloc_bitmap_reset_by_cpu_index和hwloc_bitmap_fill函数，设置多路复用结构数组中的子映射。最后，函数返回0。
2. hwloc_bitmap_set：设置一个给定的多路复用结构数组中指定CPU的子映射。这个函数接受一个结构体指针set和当前CPU作为参数，然后执行一系列的检查，如检查set是否为空，以及当前CPU是否在多路复用结构中，然后通过hwloc_bitmap_realloc_by_cpu_index和hwloc_bitmap_fill函数，设置多路复用结构数组中指定CPU的子映射。最后，函数返回0。


```cpp
int hwloc_bitmap_allbut(struct hwloc_bitmap_s * set, unsigned cpu)
{
	unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

	HWLOC__BITMAP_CHECK(set);

	if (hwloc_bitmap_reset_by_cpu_index(set, cpu) < 0)
		return -1;

	hwloc_bitmap__fill(set);
	set->ulongs[index_] &= ~HWLOC_SUBBITMAP_CPU(cpu);
	return 0;
}

int hwloc_bitmap_set(struct hwloc_bitmap_s * set, unsigned cpu)
{
	unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

	HWLOC__BITMAP_CHECK(set);

	/* nothing to do if setting inside the infinite part of the bitmap */
	if (set->infinite && cpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
		return 0;

	if (hwloc_bitmap_realloc_by_cpu_index(set, cpu) < 0)
		return -1;

	set->ulongs[index_] |= HWLOC_SUBBITMAP_CPU(cpu);
	return 0;
}

```

This function appears to be a part of a larger software build, and it is intended to be used by applications that may need to modify the setting of soft Infinitum Addresses (SIA) on a Linux system.

The function appears to check whether the input 'set' is a valid input, and if it is, it performs the following actions:

1. If the input 'set' is not a valid input, the function returns -1.
2. If the input 'set' is a valid input, the function first sets the input to indicate that the infinity is set.
3. The function then checks whether the input 'set' includes the entire infinity range. If it does, the function updates the 'beginset' and 'endset' variables to exclude the already-set infinity range.
4. The function then makes sure that the 'beginset' and 'endset' variables point to valid ulongs by calling the function 'hwloc_bitmap_realloc_by_cpu_index'.
5. The function then updates the first and last ulongs of the infinity range by setting the corresponding ulong bit to 1 and updating the other ulongs to match the updated infinity range.
6. Finally, the function updates the ulongs in the middle of the infinity range by setting all of them to 1.

The function returns 0 if the input 'set' is a valid input and updates the infinity range as appropriate.


```cpp
int hwloc_bitmap_set_range(struct hwloc_bitmap_s * set, unsigned begincpu, int _endcpu)
{
	unsigned i;
	unsigned beginset,endset;
	unsigned endcpu = (unsigned) _endcpu;

	HWLOC__BITMAP_CHECK(set);

	if (endcpu < begincpu)
		return 0;
	if (set->infinite && begincpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
		/* setting only in the already-set infinite part, nothing to do */
		return 0;

	if (_endcpu == -1) {
		/* infinite range */

		/* make sure we can play with the ulong that contains begincpu */
		if (hwloc_bitmap_realloc_by_cpu_index(set, begincpu) < 0)
			return -1;

		/* update the ulong that contains begincpu */
		beginset = HWLOC_SUBBITMAP_INDEX(begincpu);
		set->ulongs[beginset] |= HWLOC_SUBBITMAP_ULBIT_FROM(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu));
		/* set ulongs after begincpu if any already allocated */
		for(i=beginset+1; i<set->ulongs_count; i++)
			set->ulongs[i] = HWLOC_SUBBITMAP_FULL;
		/* mark the infinity as set */
		set->infinite = 1;
	} else {
		/* finite range */

		/* ignore the part of the range that overlaps with the already-set infinite part */
		if (set->infinite && endcpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
			endcpu = set->ulongs_count * HWLOC_BITS_PER_LONG - 1;
		/* make sure we can play with the ulongs that contain begincpu and endcpu */
		if (hwloc_bitmap_realloc_by_cpu_index(set, endcpu) < 0)
			return -1;

		/* update first and last ulongs */
		beginset = HWLOC_SUBBITMAP_INDEX(begincpu);
		endset = HWLOC_SUBBITMAP_INDEX(endcpu);
		if (beginset == endset) {
			set->ulongs[beginset] |= HWLOC_SUBBITMAP_ULBIT_FROMTO(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu), HWLOC_SUBBITMAP_CPU_ULBIT(endcpu));
		} else {
			set->ulongs[beginset] |= HWLOC_SUBBITMAP_ULBIT_FROM(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu));
			set->ulongs[endset] |= HWLOC_SUBBITMAP_ULBIT_TO(HWLOC_SUBBITMAP_CPU_ULBIT(endcpu));
		}
		/* set ulongs in the middle of the range */
		for(i=beginset+1; i<endset; i++)
			set->ulongs[i] = HWLOC_SUBBITMAP_FULL;
	}

	return 0;
}

```

这两函数是定义在 struct hwloc_bitmap_s 结构体中的函数，用于对基于位置的 bitmap 进行设置或清除。这里定义了两个函数：

hwloc_bitmap_set_ith_ulong( )：
```cppc
这个函数接收一个 hwloc_bitmap_s 类型的 set，一个表示要设置的 bitmap 索引 i，和一个表示要设置的 bitmap 掩码 mask。
```
通过 hwloc_bitmap_realloc_by_ulongs(set, i+1) 函数，将 i+1 大小（从 0 开始）的 bitmap 内存重新分配给 set，如果分配失败或者 i+1 大于了 set.ulongs_count * HWLOC_BITS_PER_LONG，设置 Bitmap 为 0。

hwloc_bitmap_clr( )：
```cppc
这个函数接收一个 hwloc_bitmap_s 类型的 set，一个表示要清除的 bitmap 索引 cpu，和一个表示清除操作后该 bitmap 对应位置的位掩（用来输出是否被清除，true 为被清除，false 为未被清除）。
```
首先通过 hwloc_bitmap_realloc_by_cpu_index(set, cpu) 函数，将 cpu 对应的 bitmap 从 set.ulongs_count * HWLOC_BITS_PER_LONG 内存中重新分配，这个位置是无限且不在 set.ulongs_count * HWLOC_BITS_PER_LONG 内存中的。然后通过 set->ulongs[index_] &= ~HWLOC_SUBBITMAP_CPU(cpu) 函数，将 bitmap 中对应 cpu 的位掩设置为 0，并返回 0。


```cpp
int hwloc_bitmap_set_ith_ulong(struct hwloc_bitmap_s *set, unsigned i, unsigned long mask)
{
	HWLOC__BITMAP_CHECK(set);

	if (hwloc_bitmap_realloc_by_ulongs(set, i+1) < 0)
		return -1;

	set->ulongs[i] = mask;
	return 0;
}

int hwloc_bitmap_clr(struct hwloc_bitmap_s * set, unsigned cpu)
{
	unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

	HWLOC__BITMAP_CHECK(set);

	/* nothing to do if clearing inside the infinitely-unset part of the bitmap */
	if (!set->infinite && cpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
		return 0;

	if (hwloc_bitmap_realloc_by_cpu_index(set, cpu) < 0)
		return -1;

	set->ulongs[index_] &= ~HWLOC_SUBBITMAP_CPU(cpu);
	return 0;
}

```

This function appears to be a part of a system that manages data that is stored in a large number of user-level data sets. It is used to mark the infinity as unset and clear the first and last ulongs in the finite range that overlaps with the already-unset infinite part.

The function takes two arguments: `set` and `endcpu`. `set` is a pointer to a data structure that represents a user-level data set, and `endcpu` is a pointer to an index within the data set that represents the last bit of a long bitmap that has not yet been set.

The function first sets the value of the infinity parameter to 0 and then checks if the infinity parameter is already set. If it is not set, the function sets the first and last ulongs in the finite range to 0, clearing all points between them. If the infinity parameter is already set, the function sets the first and last ulongs to 0, marking the infinity as unset.

If the function makes it through the entire bitmap without setting any ulongs, it returns 0. Otherwise, it returns -1.


```cpp
int hwloc_bitmap_clr_range(struct hwloc_bitmap_s * set, unsigned begincpu, int _endcpu)
{
	unsigned i;
	unsigned beginset,endset;
	unsigned endcpu = (unsigned) _endcpu;

	HWLOC__BITMAP_CHECK(set);

	if (endcpu < begincpu)
		return 0;

	if (!set->infinite && begincpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
		/* clearing only in the already-unset infinite part, nothing to do */
		return 0;

	if (_endcpu == -1) {
		/* infinite range */

		/* make sure we can play with the ulong that contains begincpu */
		if (hwloc_bitmap_realloc_by_cpu_index(set, begincpu) < 0)
			return -1;

		/* update the ulong that contains begincpu */
		beginset = HWLOC_SUBBITMAP_INDEX(begincpu);
		set->ulongs[beginset] &= ~HWLOC_SUBBITMAP_ULBIT_FROM(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu));
		/* clear ulong after begincpu if any already allocated */
		for(i=beginset+1; i<set->ulongs_count; i++)
			set->ulongs[i] = HWLOC_SUBBITMAP_ZERO;
		/* mark the infinity as unset */
		set->infinite = 0;
	} else {
		/* finite range */

		/* ignore the part of the range that overlaps with the already-unset infinite part */
		if (!set->infinite && endcpu >= set->ulongs_count * HWLOC_BITS_PER_LONG)
			endcpu = set->ulongs_count * HWLOC_BITS_PER_LONG - 1;
		/* make sure we can play with the ulongs that contain begincpu and endcpu */
		if (hwloc_bitmap_realloc_by_cpu_index(set, endcpu) < 0)
			return -1;

		/* update first and last ulongs */
		beginset = HWLOC_SUBBITMAP_INDEX(begincpu);
		endset = HWLOC_SUBBITMAP_INDEX(endcpu);
		if (beginset == endset) {
			set->ulongs[beginset] &= ~HWLOC_SUBBITMAP_ULBIT_FROMTO(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu), HWLOC_SUBBITMAP_CPU_ULBIT(endcpu));
		} else {
			set->ulongs[beginset] &= ~HWLOC_SUBBITMAP_ULBIT_FROM(HWLOC_SUBBITMAP_CPU_ULBIT(begincpu));
			set->ulongs[endset] &= ~HWLOC_SUBBITMAP_ULBIT_TO(HWLOC_SUBBITMAP_CPU_ULBIT(endcpu));
		}
		/* clear ulongs in the middle of the range */
		for(i=beginset+1; i<endset; i++)
			set->ulongs[i] = HWLOC_SUBBITMAP_ZERO;
	}

	return 0;
}

```

这两段代码是用来判断一个 `hwloc_bitmap_s` 结构体中所有 `ulongs` 成员是否为零，并返回相应的布尔值。

具体来说，`hwloc_bitmap_isset` 函数接收一个 `const struct hwloc_bitmap_s *` 类型的参数 `set`，和一个 `unsigned int` 类型的参数 `cpu`。函数先使用 `HWLOC_SUBBITMAP_INDEX` 函数将 `cpu` 转换为对应的 `ulong` 类型，然后使用 `HWLOC_SUBBITMAP_READULONG` 函数读取 `set->ulong` 成员的值，并将其与 `HWLOC_SUBBITMAP_CPU` 函数返回的值进行按位与操作，得到一个表示 `set` 中所有 `ulongs` 成员是否为零的布尔值。最后，函数使用 `return` 语句返回该布尔值。

`hwloc_bitmap_iszero` 函数与 `hwloc_bitmap_isset` 函数相反，它接收一个 `const struct hwloc_bitmap_s *` 类型的参数 `set`，然后使用 `HWLOC_SUBBITMAP_CHECK` 函数检查 `set` 是否为真，如果是，则执行循环检查 `set->ulongs_count` 个 `ulong` 成员是否都为零，如果是，则返回 `0`，否则返回 `1`。最后，函数使用 `return` 语句返回 `1`，表示 `set` 中所有 `ulongs` 成员都为零。


```cpp
int hwloc_bitmap_isset(const struct hwloc_bitmap_s * set, unsigned cpu)
{
	unsigned index_ = HWLOC_SUBBITMAP_INDEX(cpu);

	HWLOC__BITMAP_CHECK(set);

	return (HWLOC_SUBBITMAP_READULONG(set, index_) & HWLOC_SUBBITMAP_CPU(cpu)) != 0;
}

int hwloc_bitmap_iszero(const struct hwloc_bitmap_s *set)
{
	unsigned i;

	HWLOC__BITMAP_CHECK(set);

	if (set->infinite)
		return 0;
	for(i=0; i<set->ulongs_count; i++)
		if (set->ulongs[i] != HWLOC_SUBBITMAP_ZERO)
			return 0;
	return 1;
}

```

The functions `hwloc_bitmap_compare` and `hwloc_bitmap_make_equal` compare two `hwloc_bitmap_s` structures.

`hwloc_bitmap_compare` compares the bitmap images to see if they are equal. It does this by comparing the bitmap's pixel data to each other. If the bitmap's pixel data is not equal, the function returns `-1`.

`hwloc_bitmap_make_equal` checks if the bitmap images are equal and sets the result accordingly. It does this by first checking if the bitmap's bitmap data is equal. If the bitmap data is not equal, the function returns `-1`. If the bitmap data is equal, the function returns `0`.

The function `hwloc_bitmap_get_padding` is a helper function that returns the minimum number of bits in the bitmap's padding region. It is used by `hwloc_bitmap_constraint_handle_e` to determine the minimum number of bits that can be requested from the user.

The function `hwloc_bitmap_constraint_handle_e` is a helper function that is used to handle bitmap constraints. It takes in a `hwloc_bitmap_s` structure and a `constraint_handle_e` enum parameter. It returns a bitmap constraint handle that can be used to enforce the bitmap's constraints.

The function `hwloc_get_filename_to_draw_ins` is a helper function that takes in a `const char*` parameter and returns a filename that can be used to draw a rectangle around the bitmap. It is used by `hwloc_ bitmap_draw_rectangle` and `hwloc_ bitmap_draw_external_rectangle`.


```cpp
int hwloc_bitmap_isfull(const struct hwloc_bitmap_s *set)
{
	unsigned i;

	HWLOC__BITMAP_CHECK(set);

	if (!set->infinite)
		return 0;
	for(i=0; i<set->ulongs_count; i++)
		if (set->ulongs[i] != HWLOC_SUBBITMAP_FULL)
			return 0;
	return 1;
}

int hwloc_bitmap_isequal (const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
	unsigned count1 = set1->ulongs_count;
	unsigned count2 = set2->ulongs_count;
	unsigned min_count = count1 < count2 ? count1 : count2;
	unsigned i;

	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	for(i=0; i<min_count; i++)
		if (set1->ulongs[i] != set2->ulongs[i])
			return 0;

	if (count1 != count2) {
		unsigned long w1 = set1->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO;
		unsigned long w2 = set2->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO;
		for(i=min_count; i<count1; i++) {
			if (set1->ulongs[i] != w2)
				return 0;
		}
		for(i=min_count; i<count2; i++) {
			if (set2->ulongs[i] != w1)
				return 0;
		}
	}

	if (set1->infinite != set2->infinite)
		return 0;

	return 1;
}

```

该函数是一个名为 `hwloc_bitmap_intersects` 的函数，它用于比较两个 `hwloc_bitmap_s` 结构体，并返回它们的交集数量。

函数接收两个 `hwloc_bitmap_s` 结构体作为参数，然后对它们进行比较。首先，函数检查两个 `hwloc_bitmap_s` 是否为空。如果是，函数将返回 0。然后，函数遍历两个 `hwloc_bitmap_s` 的 `ulongs_count` 成员，并检查它们是否相等。如果是，函数返回 1。

接下来，函数检查两个 `hwloc_bitmap_s` 中是否有无限循环的成员。如果是，函数将在两个 `hwloc_bitmap_s` 中查找所有无限循环的成员，并返回它们的数量。然后，函数回到函数的主部分，如果两个 `hwloc_bitmap_s` 中都有无限循环的成员，函数将返回 1。

最后，函数检查两个 `hwloc_bitmap_s` 中是否有空成员。如果是，函数将返回 1。如果两个 `hwloc_bitmap_s` 都不是空或无限循环的，函数将返回 0。


```cpp
int hwloc_bitmap_intersects (const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
	unsigned count1 = set1->ulongs_count;
	unsigned count2 = set2->ulongs_count;
	unsigned min_count = count1 < count2 ? count1 : count2;
	unsigned i;

	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	for(i=0; i<min_count; i++)
		if (set1->ulongs[i] & set2->ulongs[i])
			return 1;

	if (count1 != count2) {
		if (set2->infinite) {
			for(i=min_count; i<set1->ulongs_count; i++)
				if (set1->ulongs[i])
					return 1;
		}
		if (set1->infinite) {
			for(i=min_count; i<set2->ulongs_count; i++)
				if (set2->ulongs[i])
					return 1;
		}
	}

	if (set1->infinite && set2->infinite)
		return 1;

	return 0;
}

```

该函数的作用是检查给定的两个 bitmap 是否包含同一个子集。它接受两个 bitmap 参数，一个是子集，另一个是超集。函数返回一个整数，表示给定子集是否包含在超集内。

具体来说，函数首先计算给定的子集和超集的大小，然后对于每个子集中的元素，检查它是否属于超集中的元素。如果子集中的元素属于超集中的元素，函数返回 0，否则函数继续执行。

如果子集中存在包含无限范围的元素，函数会检查超集是否包含这些元素。如果是，函数返回 0，否则函数返回 1。


```cpp
int hwloc_bitmap_isincluded (const struct hwloc_bitmap_s *sub_set, const struct hwloc_bitmap_s *super_set)
{
	unsigned super_count = super_set->ulongs_count;
	unsigned sub_count = sub_set->ulongs_count;
	unsigned min_count = super_count < sub_count ? super_count : sub_count;
	unsigned i;

	HWLOC__BITMAP_CHECK(sub_set);
	HWLOC__BITMAP_CHECK(super_set);

	for(i=0; i<min_count; i++)
		if (super_set->ulongs[i] != (super_set->ulongs[i] | sub_set->ulongs[i]))
			return 0;

	if (super_count != sub_count) {
		if (!super_set->infinite)
			for(i=min_count; i<sub_count; i++)
				if (sub_set->ulongs[i])
					return 0;
		if (sub_set->infinite)
			for(i=min_count; i<super_count; i++)
				if (super_set->ulongs[i] != HWLOC_SUBBITMAP_FULL)
					return 0;
	}

	if (sub_set->infinite && !super_set->infinite)
		return 0;

	return 1;
}

```

This function appears to be a part of the `hwloc_bitmap_set_cvalue` function, which is a part of the `hwloc` library for Linux. It appears to be used to merge two `hwloc_bitmap` objects, `set1` and `set2`, into a single `res` bitmap.

The function takes two `hwloc_bitmap_s` arguments, `res` and `set1`, and one more `hwloc_bitmap_s` argument, `set2`. It first checks that the `res` bitmap is not already initialized, and if it is not, it initializes it by setting all its elements to 0.

Then, it loops through each element of the `set1` and `set2` bitmaps, or just the elements of `set1` if `set2` is not initialized. For each element, it compares it to the corresponding element in `res`, and if they are different, it sets the corresponding element in `res` to the current value.

If `set1` or `set2` is not initialized, the function will set all the elements in `res` to 0, and it will return -1 if the initialization fails. If both `set1` and `set2` are initialized, the function will check if the `infinite` flag of `set2` is set. If it is, the function will set all the elements in `res` to 0 and return -1 if the initialization fails. If it is not, the function will loop through each element of `set1` and set the corresponding element in `res` to the current value.

Finally, the function checks if the `infinite` flag of `set1` or `set2` is set, and if it is, it sets the `infinite` flag of `res` to `true` if it is not already set. This is used to indicate that the function has finished initializing the `res` bitmap, and it is returned as a success or failure result.


```cpp
int hwloc_bitmap_or (struct hwloc_bitmap_s *res, const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
	/* cache counts so that we can reset res even if it's also set1 or set2 */
	unsigned count1 = set1->ulongs_count;
	unsigned count2 = set2->ulongs_count;
	unsigned max_count = count1 > count2 ? count1 : count2;
	unsigned min_count = count1 + count2 - max_count;
	unsigned i;

	HWLOC__BITMAP_CHECK(res);
	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	if (hwloc_bitmap_reset_by_ulongs(res, max_count) < 0)
		return -1;

	for(i=0; i<min_count; i++)
		res->ulongs[i] = set1->ulongs[i] | set2->ulongs[i];

	if (count1 != count2) {
		if (min_count < count1) {
			if (set2->infinite) {
				res->ulongs_count = min_count;
			} else {
				for(i=min_count; i<max_count; i++)
					res->ulongs[i] = set1->ulongs[i];
			}
		} else {
			if (set1->infinite) {
				res->ulongs_count = min_count;
			} else {
				for(i=min_count; i<max_count; i++)
					res->ulongs[i] = set2->ulongs[i];
			}
		}
	}

	res->infinite = set1->infinite || set2->infinite;
	return 0;
}

```

This function appears to be a part of the `hwloc_bitmap_and` function, which performs an AND operation on two bitmaps. It takes a third bitmap as a parameter and returns an integer indicating whether the operation was successful.

The function has several checks for bitmap-related things, such as ensuring that the bitmap is valid, that the input bitmaps are not "infinite" (i.e. have the highest possible value), and that the result bitmap is not "infinite" either.

The function performs an AND operation by first resetting the result bitmap to a default value, and then looping through the input bitmaps in set1 and set2. For each input bitmap, it compares the current value to the maximum value of the bitmap (to ensure that the result does not exceed the maximum). If the current value is less than or equal to the maximum value, the function continues with the next iteration of the loop. If the current value is greater than the maximum value, the function sets the result bitmap to the maximum value and continues the loop.

After the loop is finished, the function checks whether the input bitmaps were all set to `NULL`. If not, the function returns `-1`, indicating that the operation was not successful. If all input bitmaps were set to `NULL`, the function returns 0, indicating that the operation was successful.


```cpp
int hwloc_bitmap_and (struct hwloc_bitmap_s *res, const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
	/* cache counts so that we can reset res even if it's also set1 or set2 */
	unsigned count1 = set1->ulongs_count;
	unsigned count2 = set2->ulongs_count;
	unsigned max_count = count1 > count2 ? count1 : count2;
	unsigned min_count = count1 + count2 - max_count;
	unsigned i;

	HWLOC__BITMAP_CHECK(res);
	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	if (hwloc_bitmap_reset_by_ulongs(res, max_count) < 0)
		return -1;

	for(i=0; i<min_count; i++)
		res->ulongs[i] = set1->ulongs[i] & set2->ulongs[i];

	if (count1 != count2) {
		if (min_count < count1) {
			if (set2->infinite) {
				for(i=min_count; i<max_count; i++)
					res->ulongs[i] = set1->ulongs[i];
			} else {
				res->ulongs_count = min_count;
			}
		} else {
			if (set1->infinite) {
				for(i=min_count; i<max_count; i++)
					res->ulongs[i] = set2->ulongs[i];
			} else {
				res->ulongs_count = min_count;
			}
		}
	}

	res->infinite = set1->infinite && set2->infinite;
	return 0;
}

```

This function appears to be a part of a software library, and it appears to be used to merge two bitmaps. It takes in two bitmaps, `set1` and `set2`, and a third bitmap, `res`. It is marked as "not" by the documentation, which suggests that this function may have some limitations or be used in a different context.

The function first checks the input bitmaps and their counts. If the input bitmaps are not valid, the function returns -1. Next, it resets the `res` bitmap to the same state as `set1` and `set2` bitmaps, by counting the number of unique longs between `set1` and `set2`.

Then, the function loops through the `set1` and `set2` bitmaps, and sets or resets the corresponding long at the index corresponding to the current bit in the `set1` and `set2` bitmaps.

Finally, it checks the input bitmap `res` and sets or resets it based on whether it should be used as `set1` or `set2`. If `res` should not be used, the function returns `0`.


```cpp
int hwloc_bitmap_andnot (struct hwloc_bitmap_s *res, const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
	/* cache counts so that we can reset res even if it's also set1 or set2 */
	unsigned count1 = set1->ulongs_count;
	unsigned count2 = set2->ulongs_count;
	unsigned max_count = count1 > count2 ? count1 : count2;
	unsigned min_count = count1 + count2 - max_count;
	unsigned i;

	HWLOC__BITMAP_CHECK(res);
	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	if (hwloc_bitmap_reset_by_ulongs(res, max_count) < 0)
		return -1;

	for(i=0; i<min_count; i++)
		res->ulongs[i] = set1->ulongs[i] & ~set2->ulongs[i];

	if (count1 != count2) {
		if (min_count < count1) {
			if (!set2->infinite) {
				for(i=min_count; i<max_count; i++)
					res->ulongs[i] = set1->ulongs[i];
			} else {
				res->ulongs_count = min_count;
			}
		} else {
			if (set1->infinite) {
				for(i=min_count; i<max_count; i++)
					res->ulongs[i] = ~set2->ulongs[i];
			} else {
				res->ulongs_count = min_count;
			}
		}
	}

	res->infinite = set1->infinite && !set2->infinite;
	return 0;
}

```

This function appears to be a part of the `hwloc_parser.h` header file in the `hwloc` suite. It appears to be used to combine two bitmaps, `set1` and `set2`, into a single bitmap, `res`. The bitmap is initialized with the values from both bitmaps, and then some of the values from the original bitmaps are saved as Infinity values in the new bitmap.


```cpp
int hwloc_bitmap_xor (struct hwloc_bitmap_s *res, const struct hwloc_bitmap_s *set1, const struct hwloc_bitmap_s *set2)
{
	/* cache counts so that we can reset res even if it's also set1 or set2 */
	unsigned count1 = set1->ulongs_count;
	unsigned count2 = set2->ulongs_count;
	unsigned max_count = count1 > count2 ? count1 : count2;
	unsigned min_count = count1 + count2 - max_count;
	unsigned i;

	HWLOC__BITMAP_CHECK(res);
	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	if (hwloc_bitmap_reset_by_ulongs(res, max_count) < 0)
		return -1;

	for(i=0; i<min_count; i++)
		res->ulongs[i] = set1->ulongs[i] ^ set2->ulongs[i];

	if (count1 != count2) {
		if (min_count < count1) {
			unsigned long w2 = set2->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO;
			for(i=min_count; i<max_count; i++)
				res->ulongs[i] = set1->ulongs[i] ^ w2;
		} else {
			unsigned long w1 = set1->infinite ? HWLOC_SUBBITMAP_FULL : HWLOC_SUBBITMAP_ZERO;
			for(i=min_count; i<max_count; i++)
				res->ulongs[i] = set2->ulongs[i] ^ w1;
		}
	}

	res->infinite = (!set1->infinite) != (!set2->infinite);
	return 0;
}

```

这段代码是一个名为 `hwloc_bitmap_not` 的函数，其作用是返回一个整数类型的值，表示是否成功将一个 `hwloc_bitmap_s` 结构体中的元素从另一个 `hwloc_bitmap_s` 结构体中的元素中移除。

具体来说，该函数接收两个 `hwloc_bitmap_s` 结构体作为参数，首先检查输入的 `res` 结构体是否已经被分配给另一个 `set` 结构体，如果是，则调用 `hwloc_bitmap_reset_by_ulongs` 函数将 `res` 中的元素设置为 `set` 中的元素的相反数，并设置 `res` 的 `infinite` 字段为 `~set->infinite` 的相反数（因为 `set->infinite` 的值未知，所以无法判断是否为真）。最后，函数返回 0，表示成功执行操作。


```cpp
int hwloc_bitmap_not (struct hwloc_bitmap_s *res, const struct hwloc_bitmap_s *set)
{
	unsigned count = set->ulongs_count;
	unsigned i;

	HWLOC__BITMAP_CHECK(res);
	HWLOC__BITMAP_CHECK(set);

	if (hwloc_bitmap_reset_by_ulongs(res, count) < 0)
		return -1;

	for(i=0; i<count; i++)
		res->ulongs[i] = ~set->ulongs[i];

	res->infinite = !set->infinite;
	return 0;
}

```

这段代码是一个名为 `hwloc_bitmap_first` 的函数，它是 `hwloc_bitmap_open` 函数的子函数。它的作用是返回一个整数，表示在给定的 `hwloc_bitmap_s` 结构中，第一个可用的子集的位置。

函数的实现包括以下几个步骤：

1. 检查传入的 `set` 结构是否为有效设置，如果是，则执行后续操作。
2. 遍历 `set` 中的 `ulongs_count` 成员，并检查当前成员是否为 `set` 中的第一个成员。如果是，则表示找到了第一个可用的子集，返回该子集的偏移量（即 `hwloc_ffsl` 函数的返回值减去 1，再除以 `HWLOC_BITS_PER_LONG`，再减去 1，因为一个子集中的每个元素都被认为是连续的）。
3. 如果 `set` 中没有第一个可用的子集，或者第一个子集中的元素不是连续的，则返回 `-1`，表示找不到第一个可用的子集。
4. 如果 `set` 中存在 `infinite` 成员，则返回 `set` 的 `ulongs_count` 成员乘以 `HWLOC_BITS_PER_LONG`，因为该结构中的所有成员都被认为是连续的。


```cpp
int hwloc_bitmap_first(const struct hwloc_bitmap_s * set)
{
	unsigned i;

	HWLOC__BITMAP_CHECK(set);

	for(i=0; i<set->ulongs_count; i++) {
		/* subsets are unsigned longs, use ffsl */
		unsigned long w = set->ulongs[i];
		if (w)
			return hwloc_ffsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
	}

	if (set->infinite)
		return set->ulongs_count * HWLOC_BITS_PER_LONG;

	return -1;
}

```

这段代码定义了一个名为 `hwloc_bitmap_first_unset` 的函数，它接受一个 `struct hwloc_bitmap_s` 类型的参数 `set`。

这个函数的作用是返回一个整数，表示给定 `set` 中所有 `ulongs`（即 `long` 类型）的设置值中，第一个 `ulong` 未被设置为 `0` 时的值。

以下是这个函数的实现细节：

1. 首先定义一个名为 `i` 的整数变量。
2. 调用一个名为 `set` 的 `const struct hwloc_bitmap_s *` 类型的参数所指向的函数，得到一个指向 `set` 的指针。
3. 遍历 `set` 中的 `ulongs` 数量。
4. 对于每个 `ulong`，计算它对应的 `ffsl`（即 `fattrs`）值，其中 `w` 是 `ulong` 的解集中的 `ulong` 值。
5. 如果 `w` 为 `0`，则直接返回 `i`，表示第一个未被设置为 `0` 的 `ulong`。
6. 如果 `set` 中 `infinite` 标志为 `1`，表示给定的 `ulongs` 数量将导致无限，那么函数返回 `set` 的 `ulongs_count` 乘以 `HWLOC_BITS_PER_LONG`，即设置数量。
7. 最后，如果上述步骤 5 到 6 步都未能找到第一个未被设置为 `0` 的 `ulong`，函数返回 `-1`，表示找不到匹配的 `ulong`。


```cpp
int hwloc_bitmap_first_unset(const struct hwloc_bitmap_s * set)
{
	unsigned i;

	HWLOC__BITMAP_CHECK(set);

	for(i=0; i<set->ulongs_count; i++) {
		/* subsets are unsigned longs, use ffsl */
		unsigned long w = ~set->ulongs[i];
		if (w)
			return hwloc_ffsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
	}

	if (!set->infinite)
		return set->ulongs_count * HWLOC_BITS_PER_LONG;

	return -1;
}

```

这段代码是一个名为 `hwloc_bitmap_last` 的函数，它用于返回一个整数，表示给定 `set` 结构体中的 `ulongs_count` 个无符号整数的位图的最后元素。

函数首先检查给定的 `set` 是否为空，如果是，则返回无穷。接着，函数遍历给定 `set` 中的 `ulongs_count` 个无符号整数，对于每个元素，函数首先使用 `hwloc_flsl` 函数获取其无符号整数的位图，然后将其位图的索引减去 1（因为 `set` 中的 `ulongs_count` 是从 0 开始的），并将其转换为使用无符号整数表示的整数类型。最后，函数使用这个无符号整数类型减去 1，并将其转换为使用 `HWLOC_BITS_PER_LONG` 和 `HWLOC_BITMAP_FLSL` 函数获取的整数类型。如果所有尝试过的操作都失败，函数返回无穷。


```cpp
int hwloc_bitmap_last(const struct hwloc_bitmap_s * set)
{
	int i;

	HWLOC__BITMAP_CHECK(set);

	if (set->infinite)
		return -1;

	for(i=(int)set->ulongs_count-1; i>=0; i--) {
		/* subsets are unsigned longs, use flsl */
		unsigned long w = set->ulongs[i];
		if (w)
			return hwloc_flsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
	}

	return -1;
}

```

这段代码是一个名为 `hwloc_bitmap_last_unset` 的函数，它的作用是返回一个整数，表示给定的 `set` 结构中所有不设置为 0 的最大的ulong的长度。

具体来说，函数首先检查给定的 `set` 结构是否为空或者无穷大，如果是，那么函数返回 -1。

接着，函数遍历给定的 `set` 结构中的所有ulong，对于每个ulong，它的倒数第一个元素（即它的ulong的长度减 1）的位移值（即取反后的ulong的值减 1）被用来计算ulong的值。这个值在代码中被赋值给变量 `w`，然后使用 `hwloc_flsl` 函数将其转换为long类型，再减去 1（因为每个ulong的位数是 64 位，所以还需要减去 1），最后使用 `HWLOC_BITS_PER_LONG` 计算出这个ulong的长度（即 64 位），从而得到hwloc_flsl(w) - 1 + HWLOC_BITS_PER_LONG*i的值。

如果遍历完成后仍然发现某个ulong的值为 0，那么函数返回 -1，否则函数返回上述计算得到的值。


```cpp
int hwloc_bitmap_last_unset(const struct hwloc_bitmap_s * set)
{
	int i;

	HWLOC__BITMAP_CHECK(set);

	if (!set->infinite)
		return -1;

	for(i=(int)set->ulongs_count-1; i>=0; i--) {
		/* subsets are unsigned longs, use flsl */
		unsigned long w = ~set->ulongs[i];
		if (w)
			return hwloc_flsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
	}

	return -1;
}

```

该函数为 `hwloc_bitmap_next` 函数，它用于返回在给定 `set` 结构体中，从给定 `prev_cpu` 开始，第一个子集的索引（连续的位）后面的元素（下一个子集）的索引。

函数首先定义了一个变量 `i`，它的作用是沿着 `set` 中的 `ulongs_count` 成员的索引，直到到达给定的 `prev_cpu` 为止。然后函数进行一些检查：

1. 如果 `set` 为 `null`，函数返回 `prev_cpu`。
2. 如果给定的 `prev_cpu` 不在数组中，函数返回 `-1`。
3. 函数中包含一个循环，该循环从 `set` 的 `ulongs_count` 成员的索引开始，直到到达给定的 `prev_cpu`。
4. 在循环内部，函数首先使用给定的 `prev_cpu` 去查找前一个元素（连续的位），如果找到，则将前一个元素的最低位设置为 0，从而屏蔽前一个元素。
5. 如果找到前一个元素，函数执行以下操作：
  1. 如果前一个元素在当前元素之前，函数执行以下操作：
      a. 将当前元素的最低位设置为 1，从而记录一个不同的前驱元素。
      b. 如果前一个元素和当前元素在同一列，需要将当前元素和前一个元素的列进行与操作，以确保当前元素不会被跳出当前列。
      c. 对于前一个元素，如果当前元素是该元素的最高位，函数将前一个元素的值从内存中减去，并从位置 `i` 开始继续执行。
      d. 对于前一个元素，如果当前元素不是该元素的最高位，函数将前一个元素的值从位置 `i` 开始与当前元素进行与操作，并从位置 `i` 继续执行。
      e. 如果前一个元素是该元素的最高位，函数将从内存中减去前一个元素的值，然后从位置 `i` 开始继续执行。

函数还包含一个判断 `set` 是否为 `null` 的检查，如果 `set` 为 `null`，函数将返回 `prev_cpu`。


```cpp
int hwloc_bitmap_next(const struct hwloc_bitmap_s * set, int prev_cpu)
{
	unsigned i = HWLOC_SUBBITMAP_INDEX(prev_cpu + 1);

	HWLOC__BITMAP_CHECK(set);

	if (i >= set->ulongs_count) {
		if (set->infinite)
			return prev_cpu + 1;
		else
			return -1;
	}

	for(; i<set->ulongs_count; i++) {
		/* subsets are unsigned longs, use ffsl */
		unsigned long w = set->ulongs[i];

		/* if the prev cpu is in the same word as the possible next one,
		   we need to mask out previous cpus */
		if (prev_cpu >= 0 && HWLOC_SUBBITMAP_INDEX((unsigned) prev_cpu) == i)
			w &= ~HWLOC_SUBBITMAP_ULBIT_TO(HWLOC_SUBBITMAP_CPU_ULBIT(prev_cpu));

		if (w)
			return hwloc_ffsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
	}

	if (set->infinite)
		return set->ulongs_count * HWLOC_BITS_PER_LONG;

	return -1;
}

```

这段代码是一个名为 `hwloc_bitmap_next_unset` 的函数，它用于从给定的 `hwloc_bitmap_s` 结构中检索一个子集，并返回该子集的下一个未被设置的 bit 向量。

函数的第一个参数是一个指向 `hwloc_bitmap_s` 结构的指针 `set`，第二个参数是一个整数 `prev_cpu`，用于指定要获取子集的上一颗 CPU。

函数首先定义了一个名为 `i` 的整数变量，用于存储子集中的下一个未被设置的 bit 向量在数组中的索引。然后，函数使用 `HWLOC_SUBBITMAP_INDEX` 函数获取前一颗 CPU，并检查它是否在子集中。如果是，函数将子集中的下一个未被设置的 bit 向量按位与 `HWLOC_SUBBITMAP_CPU_ULBIT` 函数，以排除前一颗 CPU。

接下来，函数使用 for 循环遍历子集中的所有未被设置的 bit 向量。在每个遍历过程中，函数首先使用 `~` 运算符获取子集中每个未被设置的 bit 向量的相反位，然后使用 `ffsl` 函数将其减 1，并计算偏移量。最后，函数使用 if 语句检查是否还有未设置的 bit 向量，如果是，函数返回子集中的 bit 向量个数乘以每个 long 的位数，并使用 `-1` 返回前一颗 CPU。如果前一颗 CPU 在子集中，函数将返回子集中 bit 向量的下一个未被设置的 bit 向量。


```cpp
int hwloc_bitmap_next_unset(const struct hwloc_bitmap_s * set, int prev_cpu)
{
	unsigned i = HWLOC_SUBBITMAP_INDEX(prev_cpu + 1);

	HWLOC__BITMAP_CHECK(set);

	if (i >= set->ulongs_count) {
		if (!set->infinite)
			return prev_cpu + 1;
		else
			return -1;
	}

	for(; i<set->ulongs_count; i++) {
		/* subsets are unsigned longs, use ffsl */
		unsigned long w = ~set->ulongs[i];

		/* if the prev cpu is in the same word as the possible next one,
		   we need to mask out previous cpus */
		if (prev_cpu >= 0 && HWLOC_SUBBITMAP_INDEX((unsigned) prev_cpu) == i)
			w &= ~HWLOC_SUBBITMAP_ULBIT_TO(HWLOC_SUBBITMAP_CPU_ULBIT(prev_cpu));

		if (w)
			return hwloc_ffsl(w) - 1 + HWLOC_BITS_PER_LONG*i;
	}

	if (!set->infinite)
		return set->ulongs_count * HWLOC_BITS_PER_LONG;

	return -1;
}

```

该函数的作用是实现一个 bitmap 数组的单例模式。它接收一个 hwloc_bitmap_s 类型的结构体数组 set，并返回一个指向单例 bitmap 数组的指针。

函数内部首先检查 set 是否为空，如果是，则直接返回一个空指针。然后，对于每个元素 in the set 中的 ulong 类型的元素，函数先执行 HWLOC_SUBBITMAP_ZERO 操作将其设置为零，然后检查是否已经找到了该元素，如果是，则说明已经找到了该元素，函数就返回。否则，函数将使用 ffsl 函数获取该元素的 ffs 值，并将结果作为该元素的 bitmap 数组中的偏移量，最后将该元素设置为 found=1 并返回。

函数还处理了无限设置的情况。如果 set 中有一个或多个无限长度的元素，函数会检查是否找到了第一个非空元素。如果是，则将无限长度的元素设置为零，并将 set 中所有元素设置为 found=0。否则，函数会尝试使用 ffsl 函数获取第一个非空元素的 ffs 值，并将结果作为第一个非空元素的 bitmap 数组中的偏移量。函数最后，函数会将空 set 中的第一个元素设置为 found=0，并将 set 中所有元素设置为 found=0。


```cpp
int hwloc_bitmap_singlify(struct hwloc_bitmap_s * set)
{
	unsigned i;
	int found = 0;

	HWLOC__BITMAP_CHECK(set);

	for(i=0; i<set->ulongs_count; i++) {
		if (found) {
			set->ulongs[i] = HWLOC_SUBBITMAP_ZERO;
			continue;
		} else {
			/* subsets are unsigned longs, use ffsl */
			unsigned long w = set->ulongs[i];
			if (w) {
				int _ffs = hwloc_ffsl(w);
				set->ulongs[i] = HWLOC_SUBBITMAP_CPU(_ffs-1);
				found = 1;
			}
		}
	}

	if (set->infinite) {
		if (found) {
			set->infinite = 0;
		} else {
			/* set the first non allocated bit */
			unsigned first = set->ulongs_count * HWLOC_BITS_PER_LONG;
			set->infinite = 0; /* do not let realloc fill the newly allocated sets */
			return hwloc_bitmap_set(set, first);
		}
	}

	return 0;
}

```

This code appears to compare the contents of two bitmaps, `set1` and `set2`, and return the difference between the counts of their elements that are set.

It does this by first checking the bitmap headers of both bitmaps, and then storing the minimum count of elements with a bit set in a temporary variable.

It then enters a loop that iterates through all elements of the bitmap, checking the values of both `set1` and `set2`.

For each element, it first checks if both bitmaps contain the element. If both bitmaps do, it compares the values of the elements and returns the difference between them. If one bitmap does contain the element but the other does not, it reverses the comparison and returns the difference.

If both bitmaps do not contain the element, it is considered to be higher and the function returns 1.

If the count of elements with a bit set in `set1` is less than the count of elements with a bit set in `set2`, it means that `set2` has more elements that are set, so the function returns -1.

If the count of elements with a bit set in `set1` is greater than the count of elements with a bit set in `set2`, it means that `set1` has more elements that are set, so the function returns 1.

Finally, it checks if either bitmap has the "infinite" flag set. If both bitmaps have this flag set, it returns -1. Otherwise, it returns 0.


```cpp
int hwloc_bitmap_compare_first(const struct hwloc_bitmap_s * set1, const struct hwloc_bitmap_s * set2)
{
	unsigned count1 = set1->ulongs_count;
	unsigned count2 = set2->ulongs_count;
	unsigned max_count = count1 > count2 ? count1 : count2;
	unsigned min_count = count1 + count2 - max_count;
	unsigned i;

	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	for(i=0; i<min_count; i++) {
		unsigned long w1 = set1->ulongs[i];
		unsigned long w2 = set2->ulongs[i];
		if (w1 || w2) {
			int _ffs1 = hwloc_ffsl(w1);
			int _ffs2 = hwloc_ffsl(w2);
			/* if both have a bit set, compare for real */
			if (_ffs1 && _ffs2)
				return _ffs1-_ffs2;
			/* one is empty, and it is considered higher, so reverse-compare them */
			return _ffs2-_ffs1;
		}
	}

	if (count1 != count2) {
		if (min_count < count2) {
			for(i=min_count; i<count2; i++) {
				unsigned long w2 = set2->ulongs[i];
				if (set1->infinite)
					return -!(w2 & 1);
				else if (w2)
					return 1;
			}
		} else {
			for(i=min_count; i<count1; i++) {
				unsigned long w1 = set1->ulongs[i];
				if (set2->infinite)
					return !(w1 & 1);
				else if (w1)
					return -1;
			}
		}
	}

	return !!set1->infinite - !!set2->infinite;
}

```

This is a function that appears to compare the Infinity flag (set1 and set2) of two CM贝叶斯网络， and return the result of the comparison.

The function takes two arguments, set1 and set2, which are both binary CM贝叶斯 networks, and returns the result of the comparison. It does this by checking the values of set1 and set2, and then counting the number of non-infinite elements in each network, as well as comparing the values of the Infinity flags.

The function first checks that the networks are not empty, and then compares the Infinity flags. If the networks are not empty and the Infinity flags are different, the function returns the result of the comparison. If the networks are empty or the Infinity flags are the same, the function returns 0.

The function also counts the number of elements in each network that are set to "infinite" (i.e. the value that corresponds to a missing data element), and uses this information to decide which Infinity flag to return. If there are more elements in one network than in the other, the function returns -1 if the count of infinite elements is greater, or 1 if the count of infinite elements is less.

Finally, the function compares the values of the Infinity flags in each network, and returns the result of the comparison. If the values are the same, the function returns 0.


```cpp
int hwloc_bitmap_compare(const struct hwloc_bitmap_s * set1, const struct hwloc_bitmap_s * set2)
{
	unsigned count1 = set1->ulongs_count;
	unsigned count2 = set2->ulongs_count;
	unsigned max_count = count1 > count2 ? count1 : count2;
	unsigned min_count = count1 + count2 - max_count;
	int i;

	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	if ((!set1->infinite) != (!set2->infinite))
		return !!set1->infinite - !!set2->infinite;

	if (count1 != count2) {
		if (min_count < count2) {
			unsigned long val1 = set1->infinite ? HWLOC_SUBBITMAP_FULL :  HWLOC_SUBBITMAP_ZERO;
			for(i=(int)max_count-1; i>=(int) min_count; i--) {
				unsigned long val2 = set2->ulongs[i];
				if (val1 == val2)
					continue;
				return val1 < val2 ? -1 : 1;
			}
		} else {
			unsigned long val2 = set2->infinite ? HWLOC_SUBBITMAP_FULL :  HWLOC_SUBBITMAP_ZERO;
			for(i=(int)max_count-1; i>=(int) min_count; i--) {
				unsigned long val1 = set1->ulongs[i];
				if (val1 == val2)
					continue;
				return val1 < val2 ? -1 : 1;
			}
		}
	}

	for(i=(int)min_count-1; i>=0; i--) {
		unsigned long val1 = set1->ulongs[i];
		unsigned long val2 = set2->ulongs[i];
		if (val1 == val2)
			continue;
		return val1 < val2 ? -1 : 1;
	}

	return 0;
}

```

这段代码是一个名为 `hwloc_bitmap_weight` 的函数，它接受一个名为 `set` 的 `hwloc_bitmap_s` 结构体参数。函数返回一个整数表示低水平集（最低水平集）中的比特映射权重。

函数首先定义了一个整数变量 `weight`，代表低水平集中的比特映射权重。然后，定义了一个 `for` 循环，该循环遍历 `set` 中的所有 `ulongs_count` 个元素。在循环中，使用 `hwloc_weight_long` 函数计算每个 `ulong` 类型的值，并将其累加到 `weight` 上。

最后，函数返回 `weight`，表示低水平集中的比特映射权重。注意，如果 `set` 中的任何元素为 `INFINITE`，函数将返回 `-1`，表示无法确定该低水平集中的比特映射权重。


```cpp
int hwloc_bitmap_weight(const struct hwloc_bitmap_s * set)
{
	int weight = 0;
	unsigned i;

	HWLOC__BITMAP_CHECK(set);

	if (set->infinite)
		return -1;

	for(i=0; i<set->ulongs_count; i++)
		weight += hwloc_weight_long(set->ulongs[i]);
	return weight;
}

```

This function appears to determine whether a given `set1` and `set2` lead to a result of `HWLOC_BITMAP_INCLUDED`, `HWLOC_BITMAP_CONTAINS`, or `HWLOC_BITMAP_DIFFERENT`. The function takes into account whether `set1` and `set2` are both infinite, and sets the result accordingly.

The function first checks if `set1` is not empty and `set2` is not infinite. If either of these conditions is not met, the function returns `HWLOC_BITMAP_DIFFERENT`. If both conditions are met, the function checks if `set2` is empty and `set1` contains a value that is not in `set2`. If this condition is not met, the function returns `HWLOC_BITMAP_INCLUDED`. If both `set2` and `set1` contain values that are not in `set2`, the function returns `HWLOC_BITMAP_CONTAINS`. If either `set1` or `set2` is infinite, the function returns `HWLOC_BITMAP_INCLUDED`, since there is no need to check for `set2` in this case.

If `set1` is infinite and `set2` is not, the function returns `HWLOC_BITMAP_CONTAINS`. If `set1` is not infinite and `set2` is infinite, the function returns `HWLOC_BITMAP_INCLUDED`. If either `set1` or `set2` is infinite, the function always returns `HWLOC_BITMAP_INCLUDED`, since there is no need to check for `set1` in this case.


```cpp
int hwloc_bitmap_compare_inclusion(const struct hwloc_bitmap_s * set1, const struct hwloc_bitmap_s * set2)
{
	unsigned max_count = set1->ulongs_count > set2->ulongs_count ? set1->ulongs_count : set2->ulongs_count;
	int result = HWLOC_BITMAP_EQUAL; /* means empty sets return equal */
	int empty1 = 1;
	int empty2 = 1;
	unsigned i;

	HWLOC__BITMAP_CHECK(set1);
	HWLOC__BITMAP_CHECK(set2);

	for(i=0; i<max_count; i++) {
	  unsigned long val1 = HWLOC_SUBBITMAP_READULONG(set1, (unsigned) i);
	  unsigned long val2 = HWLOC_SUBBITMAP_READULONG(set2, (unsigned) i);

	  if (!val1) {
	    if (!val2)
	      /* both empty, no change */
	      continue;

	    /* val1 empty, val2 not */
	    if (result == HWLOC_BITMAP_CONTAINS) {
	      if (!empty2)
		return HWLOC_BITMAP_INTERSECTS;
	      result = HWLOC_BITMAP_DIFFERENT;
	    } else if (result == HWLOC_BITMAP_EQUAL) {
	      result = HWLOC_BITMAP_INCLUDED;
	    }
	    /* no change otherwise */

	  } else if (!val2) {
	    /* val2 empty, val1 not */
	    if (result == HWLOC_BITMAP_INCLUDED) {
	      if (!empty1)
		return HWLOC_BITMAP_INTERSECTS;
	      result = HWLOC_BITMAP_DIFFERENT;
	    } else if (result == HWLOC_BITMAP_EQUAL) {
	      result = HWLOC_BITMAP_CONTAINS;
	    }
	    /* no change otherwise */

	  } else if (val1 == val2) {
	    /* equal and not empty */
	    if (result == HWLOC_BITMAP_DIFFERENT)
	      return HWLOC_BITMAP_INTERSECTS;
	    /* equal/contains/included unchanged */

	  } else if ((val1 & val2) == val1) {
	    /* included and not empty */
	    if (result == HWLOC_BITMAP_CONTAINS || result == HWLOC_BITMAP_DIFFERENT)
	      return HWLOC_BITMAP_INTERSECTS;
	    /* equal/included unchanged */
	    result = HWLOC_BITMAP_INCLUDED;

	  } else if ((val1 & val2) == val2) {
	    /* contains and not empty */
	    if (result == HWLOC_BITMAP_INCLUDED || result == HWLOC_BITMAP_DIFFERENT)
	      return HWLOC_BITMAP_INTERSECTS;
	    /* equal/contains unchanged */
	    result = HWLOC_BITMAP_CONTAINS;

	  } else if ((val1 & val2) != 0) {
	    /* intersects and not empty */
	    return HWLOC_BITMAP_INTERSECTS;

	  } else {
	    /* different and not empty */

	    /* equal/included/contains with non-empty sets means intersects */
	    if (result == HWLOC_BITMAP_EQUAL && !empty1 /* implies !empty2 */)
	      return HWLOC_BITMAP_INTERSECTS;
	    if (result == HWLOC_BITMAP_INCLUDED && !empty1)
	      return HWLOC_BITMAP_INTERSECTS;
	    if (result == HWLOC_BITMAP_CONTAINS && !empty2)
	      return HWLOC_BITMAP_INTERSECTS;
	    /* otherwise means different */
	    result = HWLOC_BITMAP_DIFFERENT;
	  }

	  empty1 &= !val1;
	  empty2 &= !val2;
	}

	if (!set1->infinite) {
	  if (set2->infinite) {
	    /* set2 infinite only */
	    if (result == HWLOC_BITMAP_CONTAINS) {
	      if (!empty2)
		return HWLOC_BITMAP_INTERSECTS;
	      result = HWLOC_BITMAP_DIFFERENT;
	    } else if (result == HWLOC_BITMAP_EQUAL) {
	      result = HWLOC_BITMAP_INCLUDED;
	    }
	    /* no change otherwise */
	  }
	} else if (!set2->infinite) {
	  /* set1 infinite only */
	  if (result == HWLOC_BITMAP_INCLUDED) {
	    if (!empty1)
	      return HWLOC_BITMAP_INTERSECTS;
	    result = HWLOC_BITMAP_DIFFERENT;
	  } else if (result == HWLOC_BITMAP_EQUAL) {
	    result = HWLOC_BITMAP_CONTAINS;
	  }
	  /* no change otherwise */
	} else {
	  /* both infinite */
	  if (result == HWLOC_BITMAP_DIFFERENT)
	    return HWLOC_BITMAP_INTERSECTS;
	  /* equal/contains/included unchanged */
	}

	return result;
}

```