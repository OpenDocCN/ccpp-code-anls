# xmrig源码解析 38

# `src/3rdparty/hwloc/src/topology-x86.c`

这段代码是一个用于在 FreeBSD 和 NetBSD 等 BSD 发行版中补充硬件 topology 信息的后端程序。当操作系统无法向用户空间应用程序提供足够的信息时，它将使用特殊的后端程序来获取这些信息。

目前，仅 FreeBSD 和 NetBSD 通过 ` topdown` 配置来使用此后端程序，而 Linux 等其他操作系统则使用其自己的方式来获取各种硬件 topology 信息。

尽管这个后端程序在 BSD 发行版中很有用，但请注意，这个后端程序仅在操作系统无法提供足够信息时才使用，因此在某些情况下，用户空间应用程序可能需要手动配置这些信息以获得更好的性能。


```cpp
/*
 * Copyright © 2010-2022 Inria.  All rights reserved.
 * Copyright © 2010-2013 Université Bordeaux
 * Copyright © 2010-2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 *
 *
 * This backend is only used when the operating system does not export
 * the necessary hardware topology information to user-space applications.
 * Currently, FreeBSD and NetBSD only add PUs and then fallback to this
 * backend for CPU/Cache discovery.
 *
 * Other backends such as Linux have their own way to retrieve various
 * pieces of hardware topology information from the operating system
 * on various architectures, without having to use this x86-specific code.
 * But this backend is still used after them to annotate some objects with
 * additional details (CPU info in Package, Inclusiveness in Caches).
 */

```

这段代码的作用是定义了一个名为 "private/autogen/config.h" 的头文件，它可能是为了支持软件自动生成的功能而设计的。

接下来，它引入了 "hwloc.h" 和 "private/private.h" 头文件，这两个头文件可能是在其他源代码中定义的头部声明。

然后，它引入了 "private/debug.h" 和 "private/misc.h" 头文件，这两个头文件很可能是为了提供调试信息和实用程序而设计的。

接着，它引入了 "private/cpuid-x86.h" 头文件，这个头文件可能是在处理 CPU 类型为 x86 的机器上进行特定操作时使用的。

接下来，它引入了两个标准输入输出头文件 "sys/types.h" 和 "dirent.h"，可能是在导入其他库时需要使用的。

最后，它还引入了 "private/valgrind-cache.h" 和 "private/valgrind-printf.h" 头文件，这两个头文件可能是为了支持 Valgrind 工具链而设计的。


```cpp
#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/debug.h"
#include "private/misc.h"
#include "private/cpuid-x86.h"

#include <sys/types.h>
#ifdef HAVE_DIRENT_H
#include <dirent.h>
#endif
#ifdef HAVE_VALGRIND_VALGRIND_H
#include <valgrind/valgrind.h>
#endif

```



这段代码定义了一个硬件ID生成器结构体 `hwloc_x86_backend_data_s`，其中包含以下字段：

- `nbprocs`：当前进程数，定义为无符号整数类型 `unsigned` 并将其声明为 `hwloc_x86_backend_data_s` 的成员。
- `apicid_set`：一个硬件ID设置位图，用于设置APIC(Advanced Programmable Interrupt Controller)ID。
- `apicid_unique`：一个整数类型字段，用于存储唯一ID。
- `src_cpuiddump_path`：一个字符指针，用于存储CPU ID dump文件的路径。
- `is_knl`：一个布尔类型字段，用于存储是否是KNL(K探索器)模式。

接下来，定义了一个 `cpuiddump` 结构体，其中包含以下字段：

- `nr`:CPU ID的数量，定义为无符号整数类型 `unsigned` 并将其声明为 `cpuiddump` 的成员。
- `entries`：一个指向 `cpuiddump_entry` 的指针，用于存储CPU ID dump文件中的每个条目。

`cpuiddump_entry` 是一个结构体，其中包含以下字段：

- `ineax`：输入CPU ID中`eax`寄存器的值。
- `inebx`：输入CPU ID中`ebx`寄存器的值。
- `ineecx`：输入CPU ID中`ecx`寄存器的值。
- `ineedx`：输入CPU ID中`edx`寄存器的值。
- `outeax`：输出CPU ID中`eax`寄存器的值。
- `outebx`：输出CPU ID中`ebx`寄存器的值。
- `outecx`：输出CPU ID中`ecx`寄存器的值。
- `outedx`：输出CPU ID中`edx`寄存器的值。

整条代码定义了一个 `hwloc_x86_backend_data_s` 结构体，用于管理 CPU ID  dump 文件，以及一个 `cpuiddump` 结构体，用于管理 CPU ID dump 文件中的每个条目。


```cpp
struct hwloc_x86_backend_data_s {
  unsigned nbprocs;
  hwloc_bitmap_t apicid_set;
  int apicid_unique;
  char *src_cpuiddump_path;
  int is_knl;
};

/************************************
 * Management of cpuid dump as input
 */

struct cpuiddump {
  unsigned nr;
  struct cpuiddump_entry {
    unsigned inmask; /* which of ine[abcd]x are set on input */
    unsigned ineax;
    unsigned inebx;
    unsigned inecx;
    unsigned inedx;
    unsigned outeax;
    unsigned outebx;
    unsigned outecx;
    unsigned outedx;
  } *entries;
};

```

This is a C function that reads a file containing a PUD (Page-User-Data) table and returns a pointer to the associated CPUID (Controller Identifier). The PUD table is assumed to be a 9-byte binary file that contains a single 32-bit integer, which represents the page number of a virtual machine.

The function takes a file name as an argument, opens the file, and reads the PUD table from the file. It then creates an array of 9 elements of the `cpuiddump_entry` structure, which will be used to store the readed data from the PUD table.

The function first reads the file header (32-bit integer at the beginning), which indicates the file format, the number of PUD entries in the file, and the page number of the virtual machine.

The while loop then reads each byte of the PUD table, starting with the first byte (which is the page number), and stores the readed value in the appropriate index of the `cpuiddump_entry` structure.

The function uses a 9-byte pointer, `cur`, to keep track of the current position in the file, since each byte of the PUD table is 9 bytes long. The `cur` pointer is updated each time the while loop reads a byte of the PUD table.

The function also checks if the end of the file is reached (9th byte is a null character), and if it is, the function frees the memory allocated for the `cpuiddump_entry` structure and returns a NULL pointer.

The function has several errors:

* The file name must be passed as an argument.
* The PUD table file should be read in binary mode.
* The PUD table should have only one page, and the page number should be a 32-bit integer.

If these errors occur, the function will print a message and return a NULL pointer.


```cpp
static void
cpuiddump_free(struct cpuiddump *cpuiddump)
{
  if (cpuiddump->nr)
    free(cpuiddump->entries);
  free(cpuiddump);
}

static struct cpuiddump *
cpuiddump_read(const char *dirpath, unsigned idx)
{
  struct cpuiddump *cpuiddump;
  struct cpuiddump_entry *cur;
  size_t filenamelen;
  char *filename;
  FILE *file;
  char line[128];
  unsigned nr;

  cpuiddump = malloc(sizeof(*cpuiddump));
  if (!cpuiddump) {
    fprintf(stderr, "Failed to allocate cpuiddump for PU #%u, ignoring cpuiddump.\n", idx);
    goto out;
  }

  filenamelen = strlen(dirpath) + 15;
  filename = malloc(filenamelen);
  if (!filename)
    goto out_with_dump;
  snprintf(filename, filenamelen, "%s/pu%u", dirpath, idx);
  file = fopen(filename, "r");
  if (!file) {
    fprintf(stderr, "Could not read dumped cpuid file %s, ignoring cpuiddump.\n", filename);
    goto out_with_filename;
  }

  nr = 0;
  while (fgets(line, sizeof(line), file))
    nr++;
  cpuiddump->entries = malloc(nr * sizeof(struct cpuiddump_entry));
  if (!cpuiddump->entries) {
    fprintf(stderr, "Failed to allocate %u cpuiddump entries for PU #%u, ignoring cpuiddump.\n", nr, idx);
    goto out_with_file;
  }

  fseek(file, 0, SEEK_SET);
  cur = &cpuiddump->entries[0];
  nr = 0;
  while (fgets(line, sizeof(line), file)) {
    if (*line == '#')
      continue;
    if (sscanf(line, "%x %x %x %x %x => %x %x %x %x",
	      &cur->inmask,
	      &cur->ineax, &cur->inebx, &cur->inecx, &cur->inedx,
	      &cur->outeax, &cur->outebx, &cur->outecx, &cur->outedx) == 9) {
      cur++;
      nr++;
    }
  }

  cpuiddump->nr = nr;
  fclose(file);
  free(filename);
  return cpuiddump;

 out_with_file:
  fclose(file);
 out_with_filename:
  free(filename);
 out_with_dump:
  free(cpuiddump);
 out:
  return NULL;
}

```

这段代码是一个名为`cpuiddump_find_by_input`的函数，属于`cpuiddump`结构体的一部分。它的作用是检查给定的输入是否存在于`cpuiddump`结构体中，如果不存在，则输出一个错误并返回一个整数（通常是0）。

该函数接受四个参数：一个指向`unsigned`类型变量的指针`eax`，一个指向`unsigned`类型变量的指针`ebx`，一个指向`unsigned`类型变量的指针`ecx`，和一个指向`struct cpuiddump`类型的指针`cpuiddump`。

函数内部首先定义了一个循环，该循环遍历`cpuiddump`结构体中的所有条目。对于每个条目，函数首先检查给定的输入是否存在于当前条目中。如果是，则跳过继续循环；如果不是，则执行循环体中的代码，将正确的值存储到`eax`，`ebx`，`ecx`和`edx`中，然后返回。

如果循环结束后仍然没有找到输入，函数将输出一个错误并返回一个整数（通常是0）。


```cpp
static void
cpuiddump_find_by_input(unsigned *eax, unsigned *ebx, unsigned *ecx, unsigned *edx, struct cpuiddump *cpuiddump)
{
  unsigned i;

  for(i=0; i<cpuiddump->nr; i++) {
    struct cpuiddump_entry *entry = &cpuiddump->entries[i];
    if ((entry->inmask & 0x1) && *eax != entry->ineax)
      continue;
    if ((entry->inmask & 0x2) && *ebx != entry->inebx)
      continue;
    if ((entry->inmask & 0x4) && *ecx != entry->inecx)
      continue;
    if ((entry->inmask & 0x8) && *edx != entry->inedx)
      continue;
    *eax = entry->outeax;
    *ebx = entry->outebx;
    *ecx = entry->outecx;
    *edx = entry->outedx;
    return;
  }

  fprintf(stderr, "Couldn't find %x,%x,%x,%x in dumped cpuid, returning 0s.\n",
	  *eax, *ebx, *ecx, *edx);
  *eax = 0;
  *ebx = 0;
  *ecx = 0;
  *edx = 0;
}

```

这段代码定义了一个名为 `cpuid_or_from_dump` 的函数，它接受四个指针参数：`eax`、`ebx`、`ecx` 和 `edx`，分别表示处理器、扩展描述符和寄存器堆。同时，它还接受一个指向 `src_cpuiddump` 的指针参数。

函数的作用是执行从 CPU 内存或从系统内存中获取 CPUID 信息。如果指针 `src_cpuiddump` 存在，则先从该内存区域获取信息；否则，通过 `hwloc_x86_cpuid` 函数从物理内存中获取信息。

函数的实现主要取决于输入的 `src_cpuiddump` 是否为 `NULL`。如果为 `NULL`，则函数将直接从物理内存中获取信息；否则，函数将从 CPU 内存中获取信息。


```cpp
static void cpuid_or_from_dump(unsigned *eax, unsigned *ebx, unsigned *ecx, unsigned *edx, struct cpuiddump *src_cpuiddump)
{
  if (src_cpuiddump) {
    cpuiddump_find_by_input(eax, ebx, ecx, edx, src_cpuiddump);
  } else {
    hwloc_x86_cpuid(eax, ebx, ecx, edx);
  }
}

/*******************************
 * Core detection routines and structures
 */

enum hwloc_x86_disc_flags {
  HWLOC_X86_DISC_FLAG_FULL = (1<<0), /* discover everything instead of only annotating */
  HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES = (1<<1) /* use AMD topoext numanode information */
};

```

这段代码定义了几个宏，用于检查芯片中的 top 功能是否启用，并定义了一个 struct 类型的缓存信息结构体，用于描述缓存的详细信息。

具体来说，has_topoext(features) 宏用于检查芯片的 top 功能是否被启用，其中 features 是一个包含 23 位的二进制数，表示芯片支持的所有 top 功能。通过将 22 位与 1 进行与运算，获取出当前芯片是否支持 top 功能。

has_x2apic(features) 宏用于检查芯片中的 x2a 功能是否被启用，其中 features 是一个包含 21位的二进制数，表示芯片支持的所有 x2a 功能。通过将 21 位与 1 进行与运算，获取出当前芯片是否支持 x2a 功能。

has_hybrid(features) 宏用于检查芯片中的 hybrid 功能是否被启用，其中 features 是一个包含 15 位二进制数，表示芯片支持的所有 hybrid 功能。通过将 15 位与 1 进行与运算，获取出当前芯片是否支持 hybrid 功能。

struct cacheinfo 定义了一个缓存信息结构体，其中包含当前芯片支持的所有哈希表(hwloc_obj_cache_type_t)的详细信息，包括类型、级别、共享线程数量、缓存 ID、行大小、行部分、是否包含该缓存、设置、大小等。哈希表中的每个键都包含一个指向记录的指针，用于获取该缓存的详细信息。


```cpp
#define has_topoext(features) ((features)[6] & (1 << 22))
#define has_x2apic(features) ((features)[4] & (1 << 21))
#define has_hybrid(features) ((features)[18] & (1 << 15))

struct cacheinfo {
  hwloc_obj_cache_type_t type;
  unsigned level;
  unsigned nbthreads_sharing;
  unsigned cacheid;

  unsigned linesize;
  unsigned linepart;
  int inclusive;
  int ways;
  unsigned sets;
  unsigned long size;
};

```

这是一个结构体 `procinfo`，它定义了处理器的一些基本信息。下面是这个结构的详细解释：

```cpp
struct procinfo {
 // 基本信息
 unsigned present;
 unsigned apicid;

 // 定义为包装类型号
 #define PKG     0
 #define CORE     1
 #define NODE     2
 #define UNIT     3
 #define TILE     4
 #define MODULE   5
 #define DIE      6
 #define HWLOC_X86_PROCINFO_ID_NR 7

 // 定义为高速缓存类型号
 #define NODE0
 #define NODE1
 #define NODE2
 #define NODE3

 // 定义为处理器家族号
 unsigned cpustepping;

 // 定义为1表示有hyperus为主机，2表示有hyperus客户端
 unsigned hybridcoretype;
 unsigned hybridnativemodel;

 // 定义为高速缓存器配置
 unsigned numcaches;
 struct cacheinfo *cache;

 // 定义为系统型号
 char cpuvendor[13];

 // 定义为处理器型号
 char cpumodel[3*4*4+1];

 // 定义为处理器时钟速度
 unsigned cpufamilynumber;

 // 定义为1表示具有1个芯片的系统，2表示具有2个芯片的系统
 unsigned hybridpass400;
 unsigned hybridpass400_256;

 // 定义为是时候升级到Hyper-爱你的模式
 unsigned int.stype;

 // 定义为当前处理器和芯片的修订版
 unsigned int.stype_reg;

 // 定义为内存缓存类型
 enum {
   FLAT,
   RING,
   LOCK,
   EQUIP,
   OVERLAY
 } mem_cache_type;

 // 定义为对齐类型
 enum {
   NONE,
   LIT,
   SAT,
   REL,
   JC,
   REQ,
   FIX,
   NUM
 } sync_by_value;

 // 定义为缺省缓存类型
 struct cacheinfo {
   char name;
   unsigned int size;
   struct cacheinfo *next;
 };

 // 定义为其他高速缓存器配置
 struct cacheinfo高速缓存器配置；
};
```

尽管 `procinfo` 定义了多个变量，但它们都被隐藏在一个大括号 `{}` 中，这表明该结构体实际上是一个模板。模板实际上定义了类型 `procinfo`，该类型具有一个 `unsigned` 类型的 `present` 成员和一个 `unsigned` 类型的 `apicid` 成员。

具体来说，`procinfo` 模板定义了一个 `struct` 类型的 `procinfo` 变量，其中包含 `unsigned` 类型的 `ids` 成员一个整数数组，每个元素都被命名为 `hwloc_x86_procinfo_id_nr`，这个成员属于 `HWLOC_X86_PROCINFO_ID_NR` 类型。

`procinfo` 模板还定义了一个 `struct` 类型的 `cacheinfo` 变量，其中包含 `unsigned` 类型的 `numcaches` 成员，一个整数数组 `ids` 成员，每个元素都被命名为 `hwloc_x86_procinfo_id_name`，这个成员属于 `HWLOC_X86_PROCINFO_ID_NAME` 类型。

另外，`procinfo` 模板定义了一个 `unsigned` 类型的 `hybridcoretype` 成员和一个 `unsigned` 类型的 `hybridnativemodel` 成员，这两个成员都被定义为 `enum` 类型。

`procinfo` 模板还定义了几个 `unsigned` 类型的成员，包括 `present` 的定义域 `unsigned`，`apicid` 的定义域 `unsigned`，`的高速缓存器类型 `高速缓存器类型`，`缓存类型 `缓存类型`，`缓存级 `缓存级` 和 `当机时钟频率`。

最后，`procinfo` 模板定义了一个 `char` 类型的 `


```cpp
struct procinfo {
  unsigned present;
  unsigned apicid;
#define PKG 0
#define CORE 1
#define NODE 2
#define UNIT 3
#define TILE 4
#define MODULE 5
#define DIE 6
#define HWLOC_X86_PROCINFO_ID_NR 7
  unsigned ids[HWLOC_X86_PROCINFO_ID_NR];
  unsigned *otherids;
  unsigned levels;
  unsigned numcaches;
  struct cacheinfo *cache;
  char cpuvendor[13];
  char cpumodel[3*4*4+1];
  unsigned cpustepping;
  unsigned cpumodelnumber;
  unsigned cpufamilynumber;

  unsigned hybridcoretype;
  unsigned hybridnativemodel;
};

```

This function appears to be managing a cache, which is a type of data structure that is used to store frequently accessed data. The function takes in a single parameter, which is an instance of the `Info` struct that represents the cache. The `Info` struct contains several other functions that are used to manage the cache, such as `get_cache_line_size` and `cache_config`.

The function first checks if the cache is full, and if it is not, it attempts to allocate memory for the cache using the `realloc` function. If the allocation fails, the function returns and the cache remains in its current state.

If the cache is full, the function then initializes the cache configuration, including the number of threads sharing the cache, the cache's line size, and the cache's ways. The `ways` parameter is set based on the value of the `cpuid` field, which is a 32-bit unsigned integer that is present in the `Info` struct.

Finally, the function sets the cache's status and returns a pointer to the cache's memory location.


```cpp
enum cpuid_type {
  intel,
  amd,
  zhaoxin,
  hygon,
  unknown
};

/* AMD legacy cache information from specific CPUID 0x80000005-6 leaves */
static void setup__amd_cache_legacy(struct procinfo *infos, unsigned level, hwloc_obj_cache_type_t type, unsigned nbthreads_sharing, unsigned cpuid)
{
  struct cacheinfo *cache, *tmpcaches;
  unsigned cachenum;
  unsigned long size = 0;

  if (level == 1)
    size = ((cpuid >> 24)) << 10;
  else if (level == 2)
    size = ((cpuid >> 16)) << 10;
  else if (level == 3)
    size = ((cpuid >> 18)) << 19;
  if (!size)
    return;

  tmpcaches = realloc(infos->cache, (infos->numcaches+1)*sizeof(*infos->cache));
  if (!tmpcaches)
    /* failed to allocated, ignore that cache */
    return;
  infos->cache = tmpcaches;
  cachenum = infos->numcaches++;

  cache = &infos->cache[cachenum];

  cache->type = type;
  cache->level = level;
  cache->nbthreads_sharing = nbthreads_sharing;
  cache->linesize = cpuid & 0xff;
  cache->linepart = 0;
  cache->inclusive = 0; /* old AMD (K8-K10) supposed to have exclusive caches */

  if (level == 1) {
    cache->ways = (cpuid >> 16) & 0xff;
    if (cache->ways == 0xff)
      /* Fully associative */
      cache->ways = -1;
  } else {
    static const unsigned ways_tab[] = { 0, 1, 2, 0, 4, 0, 8, 0, 16, 0, 32, 48, 64, 96, 128, -1 };
    unsigned ways = (cpuid >> 12) & 0xf;
    cache->ways = ways_tab[ways];
  }
  cache->size = size;
  cache->sets = 0;

  hwloc_debug("cache L%u t%u linesize %u ways %d size %luKB\n", cache->level, cache->nbthreads_sharing, cache->linesize, cache->ways, cache->size >> 10);
}

```

这段代码定义了一个名为`read_amd_caches_legacy`的函数，它的作用是读取CPUID 0x80000005至0x80000007 inclusive的遗留缓存信息。

函数接受三个参数：

1. `infos`：一个结构体，包含了CPUID的名称、ID和版本等信息。
2. `src_cpuiddump`：一个指向CPUID断言的指针，这个指针包含了从CPUID 0x80000005到0x80000007 inclusive的访问控制。
3. `legacy_max_log_proc`：一个无符号整数，用于指示在函数中记录的最大日志进程ID。

函数的核心部分如下：
```cppperl
static void read_amd_caches_legacy(struct procinfo *infos, struct cpuiddump *src_cpuiddump, unsigned legacy_max_log_proc)
{
 unsigned eax, ebx, ecx, edx;

 eax = 0x80000005;
 cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
 setup__amd_cache_legacy(infos, 1, HWLOC_OBJ_CACHE_DATA, 1, ecx);    /* private L1d */
 setup__amd_cache_legacy(infos, 1, HWLOC_OBJ_CACHE_INSTRUCTION, 1, edx);    /* private L1i */

 eax = 0x80000006;
 cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
 if (ecx & 0xf000)
   /* This is actually supported on Intel but LinePerTag isn't returned in bits 8-11.
    * Could be useful if some Intels (at least before Core micro-architecture)
    * support this leaf without leaf 0x4.
    */
   setup__amd_cache_legacy(infos, 2, HWLOC_OBJ_CACHE_UNIFIED, 1, ecx); /* private L2u */
 if (edx & 0xf000)
   setup__amd_cache_legacy(infos, 3, HWLOC_OBJ_CACHE_UNIFIED, legacy_max_log_proc, edx); /* package-wide L3u */
}
```
函数首先定义了`read_amd_caches_legacy`函数，然后在函数内部通过`cpuid_or_from_dump`函数从CPUID 0x80000005获取了遗留缓存信息，并进行了相关的设置。

设置的内容如下：

* 在`setup__amd_cache_legacy`函数中，我们通过`ecx`和`edx`变量获取了源CPUID的值，然后将其传给`cpuid_or_from_dump`函数。这个函数允许我们设置无符号指定的最高位，具体用法如下：
```cppc
cpuid_or_from_dump(x, &eax, y, &eax, ...)
```
其中，`x`是一个无符号整数，`y`是另一个无符号整数，它们一起构成了一个无符号整数。这个函数的第一个参数是`x`，第二个参数是`eax`，用于存放传来的值。第三个参数和第四个参数同样是`y`和`eax`，但用于存放`eax`的残余部分。第五个参数用于指定叶子0x4是否执行设置。

* 在`setup__amd_cache_legacy`函数中，我们通过`HWLOC_OBJ_CACHE_DATA`、`HWLOC_OBJ_CACHE_INSTRUCTION`和`legacy_max_log_proc`参数设置私有L1d、私有L1i和package-wide L3u。这里，我们使用`eax`作为叶子的索引，然后通过`cpuid_or_from_dump`函数将其赋值为最高位。由于`HWLOC_OBJ_CACHE_DATA`，`HWLOC_OBJ_CACHE_INSTRUCTION`和`legacy_max_log_proc`是`unsigned`类型，所以这里我们可以将其隐式初始化。
* 接下来，我们通过`cpuid_or_from_dump`函数获取了CPUID 0x80000006最高位（也就是最低位），并设置了一系列私有缓存。这里，我们同样使用`eax`作为叶子的索引，然后通过`cpuid_or_from_dump`函数将其赋值为最高位。同样，由于`HWLOC_OBJ_CACHE_UNIFIED`是`unsigned`类型，所以这里我们将其隐式初始化。
* 最后，在`if`语句中，我们检查`ecx`的最高位是否为1。如果是，我们就执行设置。这里的设置对象包括：
	+ 私有L1d、私有L1i和package-wide L3u。
	+ `HWLOC_OBJ_CACHE_DATA`。
	+ `HWLOC_OBJ_CACHE_INSTRUCTION`。
	+ `legacy_max_log_proc`。


```cpp
/* AMD legacy cache information from CPUID 0x80000005-6 leaves */
static void read_amd_caches_legacy(struct procinfo *infos, struct cpuiddump *src_cpuiddump, unsigned legacy_max_log_proc)
{
  unsigned eax, ebx, ecx, edx;

  eax = 0x80000005;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  setup__amd_cache_legacy(infos, 1, HWLOC_OBJ_CACHE_DATA, 1, ecx); /* private L1d */
  setup__amd_cache_legacy(infos, 1, HWLOC_OBJ_CACHE_INSTRUCTION, 1, edx); /* private L1i */

  eax = 0x80000006;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  if (ecx & 0xf000)
    /* This is actually supported on Intel but LinePerTag isn't returned in bits 8-11.
     * Could be useful if some Intels (at least before Core micro-architecture)
     * support this leaf without leaf 0x4.
     */
    setup__amd_cache_legacy(infos, 2, HWLOC_OBJ_CACHE_UNIFIED, 1, ecx); /* private L2u */
  if (edx & 0xf000)
    setup__amd_cache_legacy(infos, 3, HWLOC_OBJ_CACHE_UNIFIED, legacy_max_log_proc, edx); /* package-wide L3u */
}

```

This is a function that initializes a cache object in硬件-accelerated


```cpp
/* AMD caches from CPUID 0x8000001d leaf (topoext) */
static void read_amd_caches_topoext(struct procinfo *infos, struct cpuiddump *src_cpuiddump)
{
  unsigned eax, ebx, ecx, edx;
  unsigned cachenum;
  struct cacheinfo *cache;

  /* the code below doesn't want any other cache yet */
  assert(!infos->numcaches);

  for (cachenum = 0; ; cachenum++) {
    eax = 0x8000001d;
    ecx = cachenum;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    if ((eax & 0x1f) == 0)
      break;
    infos->numcaches++;
  }

  cache = infos->cache = malloc(infos->numcaches * sizeof(*infos->cache));
  if (cache) {
    for (cachenum = 0; ; cachenum++) {
      unsigned long linesize, linepart, ways, sets;
      eax = 0x8000001d;
      ecx = cachenum;
      cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);

      if ((eax & 0x1f) == 0)
	break;
      switch (eax & 0x1f) {
      case 1: cache->type = HWLOC_OBJ_CACHE_DATA; break;
      case 2: cache->type = HWLOC_OBJ_CACHE_INSTRUCTION; break;
      default: cache->type = HWLOC_OBJ_CACHE_UNIFIED; break;
      }

      cache->level = (eax >> 5) & 0x7;
      /* Note: actually number of cores */
      cache->nbthreads_sharing = ((eax >> 14) &  0xfff) + 1;

      cache->linesize = linesize = (ebx & 0xfff) + 1;
      cache->linepart = linepart = ((ebx >> 12) & 0x3ff) + 1;
      ways = ((ebx >> 22) & 0x3ff) + 1;

      if (eax & (1 << 9))
	/* Fully associative */
	cache->ways = -1;
      else
	cache->ways = ways;
      cache->sets = sets = ecx + 1;
      cache->size = linesize * linepart * ways * sets;
      cache->inclusive = edx & 0x2;

      hwloc_debug("cache %u L%u%c t%u linesize %lu linepart %lu ways %lu sets %lu, size %luKB\n",
		  cachenum, cache->level,
		  cache->type == HWLOC_OBJ_CACHE_DATA ? 'd' : cache->type == HWLOC_OBJ_CACHE_INSTRUCTION ? 'i' : 'u',
		  cache->nbthreads_sharing, linesize, linepart, ways, sets, cache->size >> 10);

      cache++;
    }
  } else {
    infos->numcaches = 0;
  }
}

```

This is a function that performs a performance counter for a banker's algorithm that uses multi-threading. The counter includes various metadata such as the number of cache lines, the number of threads sharing the cache, the cache type, and the cache line size.

The function takes an integer value `eax` as an input and performs various calculations based on that value. The integer value `eax` is the extractor for the most significant 32 bits, and `eax & 0x1f` extracts the least significant 32 bits.

The function checks whether the current thread is assigned to a banker's algorithm thread. If it is, the function performs a banker's algorithm update and stores the result in the `ecx` register.

The function also performs a performance counter for the banker's algorithm based on the number of cache lines, the number of threads sharing the cache, the cache type, and the cache line size. The counter includes support for both data and instruction cache lines.

The function returns the performance counter in a hexadecimal format.


```cpp
/* Intel cache info from CPUID 0x04 leaf */
static void read_intel_caches(struct hwloc_x86_backend_data_s *data, struct procinfo *infos, struct cpuiddump *src_cpuiddump)
{
  unsigned level;
  struct cacheinfo *tmpcaches;
  unsigned eax, ebx, ecx, edx;
  unsigned oldnumcaches = infos->numcaches; /* in case we got caches above */
  unsigned cachenum;
  struct cacheinfo *cache;

  for (cachenum = 0; ; cachenum++) {
    eax = 0x04;
    ecx = cachenum;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);

    hwloc_debug("cache %u type %u\n", cachenum, eax & 0x1f);
    if ((eax & 0x1f) == 0)
      break;
    level = (eax >> 5) & 0x7;
    if (data->is_knl && level == 3)
      /* KNL reports wrong L3 information (size always 0, cpuset always the entire machine, ignore it */
      break;
    infos->numcaches++;
  }

  tmpcaches = realloc(infos->cache, infos->numcaches * sizeof(*infos->cache));
  if (!tmpcaches) {
    infos->numcaches = oldnumcaches;
  } else {
    infos->cache = tmpcaches;
    cache = &infos->cache[oldnumcaches];

    for (cachenum = 0; ; cachenum++) {
      unsigned long linesize, linepart, ways, sets;
      eax = 0x04;
      ecx = cachenum;
      cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);

      if ((eax & 0x1f) == 0)
	break;
      level = (eax >> 5) & 0x7;
      if (data->is_knl && level == 3)
	/* KNL reports wrong L3 information (size always 0, cpuset always the entire machine, ignore it */
	break;
      switch (eax & 0x1f) {
      case 1: cache->type = HWLOC_OBJ_CACHE_DATA; break;
      case 2: cache->type = HWLOC_OBJ_CACHE_INSTRUCTION; break;
      default: cache->type = HWLOC_OBJ_CACHE_UNIFIED; break;
      }

      cache->level = level;
      cache->nbthreads_sharing = ((eax >> 14) & 0xfff) + 1;

      cache->linesize = linesize = (ebx & 0xfff) + 1;
      cache->linepart = linepart = ((ebx >> 12) & 0x3ff) + 1;
      ways = ((ebx >> 22) & 0x3ff) + 1;
      if (eax & (1 << 9))
        /* Fully associative */
        cache->ways = -1;
      else
        cache->ways = ways;
      cache->sets = sets = ecx + 1;
      cache->size = linesize * linepart * ways * sets;
      cache->inclusive = edx & 0x2;

      hwloc_debug("cache %u L%u%c t%u linesize %lu linepart %lu ways %lu sets %lu, size %luKB\n",
		  cachenum, cache->level,
		  cache->type == HWLOC_OBJ_CACHE_DATA ? 'd' : cache->type == HWLOC_OBJ_CACHE_INSTRUCTION ? 'i' : 'u',
		  cache->nbthreads_sharing, linesize, linepart, ways, sets, cache->size >> 10);
      cache++;
    }
  }
}

```

This function appears to be part of the AMD总公司 的开发文档，描述了如何从 CPU ID 内存泄漏中提取出旧 CPU 的一些信息，例如核心 ID、线程 ID 等。

具体来说，函数首先通过 `cpuid_or_from_dump` 函数从传入的 `src_cpuuddump` 结构中提取出 CPU ID、核心 ID 等信息，然后根据提取出的信息计算出最大核心 ID、最大线程 ID 等参数，最后将这些参数用于 `hwloc_attribute_unused` 标记的 `threadid` 字段中。

函数输出的参数包括：

- `infos`：包含 CPU ID 和其他一些信息的结构体
- `src_cpuuddump`：包含源 CPU ID 和其他信息的结构体，这个结构体可能是从 `cpuid_setup` 函数中传递过来的
- `max_nbcores`：旧 CPU 最多支持的核数
- `max_nbthreads`：旧 CPU 最多支持的线程数
- `coreidsize`：旧 CPU 每个核心占用的字节数
- `hwloc_attribute_unused`：保留字段，用于标记在输出中不被使用的值

函数还包含以下输出：

- `coreidsize`：从 `src_cpuuddump` 中提取出的旧 CPU 核心 ID
- `hwloc_debug`：用于输出调试信息的函数，输出格式如下：
```cppperl
<printf format="%u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' - %u ' -


```
/* AMD core/thread info from CPUID 0x80000008 leaf */
static void read_amd_cores_legacy(struct procinfo *infos, struct cpuiddump *src_cpuiddump)
{
  unsigned eax, ebx, ecx, edx;
  unsigned max_nbcores;
  unsigned max_nbthreads;
  unsigned coreidsize;
  unsigned logprocid;
  unsigned threadid __hwloc_attribute_unused;

  eax = 0x80000008;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);

  coreidsize = (ecx >> 12) & 0xf;
  hwloc_debug("core ID size: %u\n", coreidsize);
  if (!coreidsize) {
    max_nbcores = (ecx & 0xff) + 1;
  } else
    max_nbcores = 1 << coreidsize;
  hwloc_debug("Thus max # of cores: %u\n", max_nbcores);

  /* No multithreaded AMD for this old CPUID leaf */
  max_nbthreads = 1 ;
  hwloc_debug("and max # of threads: %u\n", max_nbthreads);

  /* legacy_max_log_proc is deprecated, it can be smaller than max_nbcores,
   * which is the maximum number of cores that the processor could theoretically support
   * (see "Multiple Core Calculation" in the AMD CPUID specification).
   * Recompute packageid/coreid accordingly.
   */
  infos->ids[PKG] = infos->apicid / max_nbcores;
  logprocid = infos->apicid % max_nbcores;
  infos->ids[CORE] = logprocid / max_nbthreads;
  threadid = logprocid % max_nbthreads;
  hwloc_debug("this is thread %u of core %u\n", threadid, infos->ids[CORE]);
}

```cpp

This is a section of the code of the "topoext" driver that appears to define the behavior of the "awesome" driver when it is used to determine the number of nodes and threads per core in a multi-core processor.

The code starts by defining a number of constants and variables, such as the number of nodes per process, the number of threads per core, and the topoext ID.

The next step is to check if the multi-core processor is "topoext"-compatible, which seems to involve checking the value of the "cpufamilynumber" field in the "topoext"-compatible information structure. If the value is within the range of 0x15 to 0x19, the next step is to assume that the "nodes_per_proc" field is 0, which may indicate that the processor is not topoext-compatible.

If the multi-core processor is topoext-compatible, the next step is to calculate the number of threads per core by taking the difference between the "nodes_per_proc" field and the "infos->cpufamilynumber" field, and then adding 1.

Finally, the code defines two hwloc\_debug() calls to print information about the topoext driver, including the number of nodes per process, the number of threads per core, and the topoext ID.

If the multi-core processor is not topoext-compatible, the code defines a number of variables and constants that are used in the calculation of the number of nodes per process and the number of threads per core.

The topcode "topoext" driver is a "raw岗尼" driver, which means that it is a simplified version of the "raw岗尼" driver that does not perform any actual work, but instead provides a hook to the "raw岗尼" driver that allows it to print information about the multi-core processor.


```
/* AMD unit/node from CPUID 0x8000001e leaf (topoext) */
static void read_amd_cores_topoext(struct procinfo *infos, unsigned long flags, struct cpuiddump *src_cpuiddump)
{
  unsigned apic_id, nodes_per_proc = 0;
  unsigned eax, ebx, ecx, edx;

  eax = 0x8000001e;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  infos->apicid = apic_id = eax;

  if (flags & HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES) {
    if (infos->cpufamilynumber == 0x16) {
      /* ecx is reserved */
      infos->ids[NODE] = 0;
      nodes_per_proc = 1;
    } else {
      /* AMD other families or Hygon family 18h */
      infos->ids[NODE] = ecx & 0xff;
      nodes_per_proc = ((ecx >> 8) & 7) + 1;
    }
    if ((infos->cpufamilynumber == 0x15 && nodes_per_proc > 2)
	|| ((infos->cpufamilynumber == 0x17 || infos->cpufamilynumber == 0x18) && nodes_per_proc > 4)
        || (infos->cpufamilynumber == 0x19 && nodes_per_proc > 1)) {
      hwloc_debug("warning: undefined nodes_per_proc value %u, assuming it means %u\n", nodes_per_proc, nodes_per_proc);
    }
  }

  if (infos->cpufamilynumber <= 0x16) { /* topoext appeared in 0x15 and compute-units were only used in 0x15 and 0x16 */
    unsigned cores_per_unit;
    /* coreid was obtained from read_amd_cores_legacy() earlier */
    infos->ids[UNIT] = ebx & 0xff;
    cores_per_unit = ((ebx >> 8) & 0xff) + 1;
    hwloc_debug("topoext %08x, %u nodes, node %u, %u cores in unit %u\n", apic_id, nodes_per_proc, infos->ids[NODE], cores_per_unit, infos->ids[UNIT]);
    /* coreid and unitid are package-wide (core 0-15 and unit 0-7 on 16-core 2-NUMAnode processor).
     * The Linux kernel reduces theses to NUMA-node-wide (by applying %core_per_node and %unit_per node respectively).
     * It's not clear if we should do this as well.
     */
  } else {
    unsigned threads_per_core;
    infos->ids[CORE] = ebx & 0xff;
    threads_per_core = ((ebx >> 8) & 0xff) + 1;
    hwloc_debug("topoext %08x, %u nodes, node %u, %u threads in core %u\n", apic_id, nodes_per_proc, infos->ids[NODE], threads_per_core, infos->ids[CORE]);
  }
}

```cpp

This is a function provided by the "apic\_packageshift" and "apic\_shift" APIs. It determines the next shift of the APIButtonF行政部门负责的出错信息。函数的第一个参数是"apic\_packageshift"函数的输出，第二个参数是"apic\_shift"函数的输出。函数首先输出"nextshift"和"num"的值，然后输出"nextshift"和"apic\_id"的值。根据"apic\_type"的值选择执行下列操作：

- 如果"apic\_type"是1，那么执行"threadid = id"，即获取当前核心的线程ID，然后将此值赋给"threadid"变量。接着，尝试将"apic\_number"设置为实际的线程数，然后尝试将"apic\_shift"设置为实际的线程数。
- 如果"apic\_type"是2，那么执行"infos->ids[CORE] = id"，将此值设置为当前核心的ID。接着，将"apic\_number"设置为实际的线程数，然后将"apic\_shift"设置为实际的线程数。
- 如果"apic\_type"是3，那么执行"infos->ids[MODULE] = id"，将此值设置为当前模块的ID。接着，将"apic\_number"设置为实际的线程数，然后将"apic\_shift"设置为实际的线程数。
- 如果"apic\_type"是4，那么执行"infos->ids[TILE] = id"，将此值设置为当前芯片的ID。接着，将"apic\_number"设置为实际的线程数，然后将"apic\_shift"设置为实际的线程数。
- 如果"apic\_type"是5，那么执行"infos->ids[DIE] = id"，将此值设置为当前芯片的ID。接着，将"apic\_number"设置为实际的线程数，然后将"apic\_shift"设置为实际的线程数。
- 如果"apic\_type"的值不匹配上述任何一种，那么执行错误。


```
/* Intel core/thread or even die/module/tile from CPUID 0x0b or 0x1f leaves (v1 and v2 extended topology enumeration) */
static void read_intel_cores_exttopoenum(struct procinfo *infos, unsigned leaf, struct cpuiddump *src_cpuiddump)
{
  unsigned level, apic_nextshift, apic_number, apic_type, apic_id = 0, apic_shift = 0, id;
  unsigned threadid __hwloc_attribute_unused = 0; /* shut-up compiler */
  unsigned eax, ebx, ecx = 0, edx;
  int apic_packageshift = 0;

  for (level = 0; ; level++) {
    ecx = level;
    eax = leaf;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    if (!eax && !ebx)
      break;
    apic_packageshift = eax & 0x1f;
  }

  if (level) {
    infos->otherids = malloc(level * sizeof(*infos->otherids));
    if (infos->otherids) {
      infos->levels = level;
      for (level = 0; ; level++) {
	ecx = level;
	eax = leaf;
	cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
	if (!eax && !ebx)
	  break;
	apic_nextshift = eax & 0x1f;
	apic_number = ebx & 0xffff;
	apic_type = (ecx & 0xff00) >> 8;
	apic_id = edx;
	id = (apic_id >> apic_shift) & ((1 << (apic_packageshift - apic_shift)) - 1);
	hwloc_debug("x2APIC %08x %u: nextshift %u num %2u type %u id %2u\n", apic_id, level, apic_nextshift, apic_number, apic_type, id);
	infos->apicid = apic_id;
	infos->otherids[level] = UINT_MAX;
	switch (apic_type) {
	case 1:
	  threadid = id;
	  /* apic_number is the actual number of threads per core */
	  break;
	case 2:
	  infos->ids[CORE] = id;
	  /* apic_number is the actual number of threads per die */
	  break;
	case 3:
	  infos->ids[MODULE] = id;
	  /* apic_number is the actual number of threads per tile */
	  break;
	case 4:
	  infos->ids[TILE] = id;
	  /* apic_number is the actual number of threads per die */
	  break;
	case 5:
	  infos->ids[DIE] = id;
	  /* apic_number is the actual number of threads per package */
	  break;
	default:
	  hwloc_debug("x2APIC %u: unknown type %u\n", level, apic_type);
	  infos->otherids[level] = apic_id >> apic_shift;
	  break;
	}
	apic_shift = apic_nextshift;
      }
      infos->apicid = apic_id;
      infos->ids[PKG] = apic_id >> apic_shift;
      hwloc_debug("x2APIC remainder: %u\n", infos->ids[PKG]);
      hwloc_debug("this is thread %u of core %u\n", threadid, infos->ids[CORE]);
    }
  }
}

```cpp

This code appears to be part of an Apple FaithlessHWL driver for the AMD Ryzen Ryzh旁边路笔记本。 It appears to be setting up the cache hierarchy for the APIC (Application Programming Interface) functionality, which is a system-level interface for communicating with the processor.

The code appears to be setting up a cache for each APIC ID, based on the number of threads sharing the APIC (2 for each ID, minimum) and the number of threads per package (2 per previous package). The cache ID is calculated by taking the remainder of the APIC ID divided by the number of threads per package, and then taking the result of that result divided by 2.

If the APIC ID is from a specific range of values, the code checks if the corresponding number of threads per package is within that range, and if so, sets the cache ID to that number.

If the APIC ID is not from a specific range of values, the code sets the cache ID to a default value of 0.

It appears to be handling the case where multiple APIC IDs share the same cache, by setting the cache ID to a value that is appropriate for sharing.


```
/* Fetch information from the processor itself thanks to cpuid and store it in
 * infos for summarize to analyze them globally */
static void look_proc(struct hwloc_backend *backend, struct procinfo *infos, unsigned long flags, unsigned highest_cpuid, unsigned highest_ext_cpuid, unsigned *features, enum cpuid_type cpuid_type, struct cpuiddump *src_cpuiddump)
{
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  unsigned eax, ebx, ecx = 0, edx;
  unsigned cachenum;
  struct cacheinfo *cache;
  unsigned regs[4];
  unsigned legacy_max_log_proc; /* not valid on Intel processors with > 256 threads, or when cpuid 0x80000008 is supported */
  unsigned legacy_log_proc_id;
  unsigned _model, _extendedmodel, _family, _extendedfamily;

  infos->present = 1;

  /* Get apicid, legacy_max_log_proc, packageid, legacy_log_proc_id from cpuid 0x01 */
  eax = 0x01;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  infos->apicid = ebx >> 24;
  if (edx & (1 << 28)) {
    legacy_max_log_proc = 1 << hwloc_flsl(((ebx >> 16) & 0xff) - 1);
  } else {
    hwloc_debug("HTT bit not set in CPUID 0x01.edx, assuming legacy_max_log_proc = 1\n");
    legacy_max_log_proc = 1;
  }

  hwloc_debug("APIC ID 0x%02x legacy_max_log_proc %u\n", infos->apicid, legacy_max_log_proc);
  infos->ids[PKG] = infos->apicid / legacy_max_log_proc;
  legacy_log_proc_id = infos->apicid % legacy_max_log_proc;
  hwloc_debug("phys %u legacy thread %u\n", infos->ids[PKG], legacy_log_proc_id);

  /* Get cpu model/family/stepping numbers from same cpuid */
  _model          = (eax>>4) & 0xf;
  _extendedmodel  = (eax>>16) & 0xf;
  _family         = (eax>>8) & 0xf;
  _extendedfamily = (eax>>20) & 0xff;
  if ((cpuid_type == intel || cpuid_type == amd || cpuid_type == hygon) && _family == 0xf) {
    infos->cpufamilynumber = _family + _extendedfamily;
  } else {
    infos->cpufamilynumber = _family;
  }
  if ((cpuid_type == intel && (_family == 0x6 || _family == 0xf))
      || ((cpuid_type == amd || cpuid_type == hygon) && _family == 0xf)
      || (cpuid_type == zhaoxin && (_family == 0x6 || _family == 0x7))) {
    infos->cpumodelnumber = _model + (_extendedmodel << 4);
  } else {
    infos->cpumodelnumber = _model;
  }
  infos->cpustepping = eax & 0xf;

  if (cpuid_type == intel && infos->cpufamilynumber == 0x6 &&
      (infos->cpumodelnumber == 0x57 || infos->cpumodelnumber == 0x85))
    data->is_knl = 1; /* KNM is the same as KNL */

  /* Get cpu vendor string from cpuid 0x00 */
  memset(regs, 0, sizeof(regs));
  regs[0] = 0;
  cpuid_or_from_dump(&regs[0], &regs[1], &regs[3], &regs[2], src_cpuiddump);
  memcpy(infos->cpuvendor, regs+1, 4*3);
  /* infos was calloc'ed, already ends with \0 */

  /* Get cpu model string from cpuid 0x80000002-4 */
  if (highest_ext_cpuid >= 0x80000004) {
    memset(regs, 0, sizeof(regs));
    regs[0] = 0x80000002;
    cpuid_or_from_dump(&regs[0], &regs[1], &regs[2], &regs[3], src_cpuiddump);
    memcpy(infos->cpumodel, regs, 4*4);
    regs[0] = 0x80000003;
    cpuid_or_from_dump(&regs[0], &regs[1], &regs[2], &regs[3], src_cpuiddump);
    memcpy(infos->cpumodel + 4*4, regs, 4*4);
    regs[0] = 0x80000004;
    cpuid_or_from_dump(&regs[0], &regs[1], &regs[2], &regs[3], src_cpuiddump);
    memcpy(infos->cpumodel + 4*4*2, regs, 4*4);
    /* infos was calloc'ed, already ends with \0 */
  }

  if ((cpuid_type != amd && cpuid_type != hygon) && highest_cpuid >= 0x04) {
    /* Get core/thread information from first cache reported by cpuid 0x04
     * (not supported on AMD)
     */
    eax = 0x04;
    ecx = 0;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    if ((eax & 0x1f) != 0) {
      /* cache looks valid */
      unsigned max_nbcores;
      unsigned max_nbthreads;
      unsigned threadid __hwloc_attribute_unused;
      hwloc_debug("Trying to get core/thread IDs from 0x04...\n");
      max_nbcores = ((eax >> 26) & 0x3f) + 1;
      hwloc_debug("found %u cores max\n", max_nbcores);
      /* some VMs (e.g. issue#525) don't report valid information, check things before dividing by 0. */
      if (!max_nbcores) {
        hwloc_debug("cannot detect core/thread IDs from 0x04 without a valid max of cores\n");
      } else {
        max_nbthreads = legacy_max_log_proc / max_nbcores;
        hwloc_debug("found %u threads max\n", max_nbthreads);
        if (!max_nbthreads) {
          hwloc_debug("cannot detect core/thread IDs from 0x04 without a valid max of threads\n");
        } else {
          threadid = legacy_log_proc_id % max_nbthreads;
          infos->ids[CORE] = legacy_log_proc_id / max_nbthreads;
          hwloc_debug("this is thread %u of core %u\n", threadid, infos->ids[CORE]);
        }
      }
    }
  }

  if (highest_cpuid >= 0x1a && has_hybrid(features)) {
    /* Get hybrid cpu information from cpuid 0x1a */
    eax = 0x1a;
    ecx = 0;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    infos->hybridcoretype = eax >> 24;
    infos->hybridnativemodel = eax & 0xffffff;
  }

  /*********************************************************************************
   * Get the hierarchy of thread, core, die, package, etc. from CPU-specific leaves
   */

  if (cpuid_type != intel && cpuid_type != zhaoxin && highest_ext_cpuid >= 0x80000008 && !has_x2apic(features)) {
    /* Get core/thread information from cpuid 0x80000008
     * (not supported on Intel)
     * We could ignore this codepath when x2apic is supported, but we may need
     * nodeids if HWLOC_X86_TOPOEXT_NUMANODES is set.
     */
    read_amd_cores_legacy(infos, src_cpuiddump);
  }

  if (cpuid_type != intel && cpuid_type != zhaoxin && has_topoext(features)) {
    /* Get apicid, nodeid, unitid/coreid from cpuid 0x8000001e (AMD topology extension).
     * Requires read_amd_cores_legacy() for coreid on family 0x15-16.
     *
     * Only needed when x2apic supported if NUMA nodes are needed.
     */
    read_amd_cores_topoext(infos, flags, src_cpuiddump);
  }

  if ((cpuid_type == intel) && highest_cpuid >= 0x1f) {
    /* Get package/die/module/tile/core/thread information from cpuid 0x1f
     * (Intel v2 Extended Topology Enumeration)
     */
    read_intel_cores_exttopoenum(infos, 0x1f, src_cpuiddump);

  } else if ((cpuid_type == intel || cpuid_type == amd || cpuid_type == zhaoxin)
	     && highest_cpuid >= 0x0b && has_x2apic(features)) {
    /* Get package/core/thread information from cpuid 0x0b
     * (Intel v1 Extended Topology Enumeration)
     */
    read_intel_cores_exttopoenum(infos, 0x0b, src_cpuiddump);
  }

  /**************************************
   * Get caches from CPU-specific leaves
   */

  infos->numcaches = 0;
  infos->cache = NULL;

  if (cpuid_type != intel && cpuid_type != zhaoxin && has_topoext(features)) {
    /* Get cache information from cpuid 0x8000001d (AMD topology extension) */
    read_amd_caches_topoext(infos, src_cpuiddump);

  } else if (cpuid_type != intel && cpuid_type != zhaoxin && highest_ext_cpuid >= 0x80000006) {
    /* If there's no topoext,
     * get cache information from cpuid 0x80000005 and 0x80000006.
     * (not supported on Intel)
     * It looks like we cannot have 0x80000005 without 0x80000006.
     */
    read_amd_caches_legacy(infos, src_cpuiddump, legacy_max_log_proc);
  }

  if ((cpuid_type != amd && cpuid_type != hygon) && highest_cpuid >= 0x04) {
    /* Get cache information from cpuid 0x04
     * (not supported on AMD)
     */
    read_intel_caches(data, infos, src_cpuiddump);
  }

  /* Now that we have all info, compute cacheids and apply quirks */
  for (cachenum = 0; cachenum < infos->numcaches; cachenum++) {
    cache = &infos->cache[cachenum];

    /* default cacheid value */
    cache->cacheid = infos->apicid / cache->nbthreads_sharing;

    if (cpuid_type == intel) {
      /* round nbthreads_sharing to nearest power of two to build a mask (for clearing lower bits) */
      unsigned bits = hwloc_flsl(cache->nbthreads_sharing-1);
      unsigned mask = ~((1U<<bits) - 1);
      cache->cacheid = infos->apicid & mask;

    } else if (cpuid_type == amd) {
      /* AMD quirks */
      if (infos->cpufamilynumber >= 0x17 && cache->level == 3) {
	/* AMD family 0x19 always shares L3 between 16 APIC ids (8 HT cores).
         * while Family 0x17 shares between 8 APIC ids (4 HT cores).
         * But many models have less APIC ids enabled and reported in nbthreads_sharing.
         * It means we must round-up nbthreads_sharing to the nearest power of 2
         * before computing cacheid.
	 */
        unsigned nbapics_sharing = cache->nbthreads_sharing;
        if (nbapics_sharing & (nbapics_sharing-1))
          /* not a power of two, round-up */
          nbapics_sharing = 1U<<(1+hwloc_ffsl(nbapics_sharing));

	cache->cacheid = infos->apicid / nbapics_sharing;

      } else if (infos->cpufamilynumber== 0x10 && infos->cpumodelnumber == 0x9
	  && cache->level == 3
	  && (cache->ways == -1 || (cache->ways % 2 == 0)) && cache->nbthreads_sharing >= 8) {
	/* Fix AMD family 0x10 model 0x9 (Magny-Cours) with 8 or 12 cores.
	 * The L3 (and its associativity) is actually split into two halves).
	 */
	if (cache->nbthreads_sharing == 16)
	  cache->nbthreads_sharing = 12; /* nbthreads_sharing is a power of 2 but the processor actually has 8 or 12 cores */
	cache->nbthreads_sharing /= 2;
	cache->size /= 2;
	if (cache->ways != -1)
	  cache->ways /= 2;
	/* AMD Magny-Cours 12-cores processor reserve APIC ids as AAAAAABBBBBB....
	 * among first L3 (A), second L3 (B), and unexisting cores (.).
	 * On multi-socket servers, L3 in non-first sockets may have APIC id ranges
	 * such as [16-21] that are not aligned on multiple of nbthreads_sharing (6).
	 * That means, we can't just compare apicid/nbthreads_sharing to identify siblings.
	 */
	cache->cacheid = (infos->apicid % legacy_max_log_proc) / cache->nbthreads_sharing /* cacheid within the package */
	  + 2 * (infos->apicid / legacy_max_log_proc); /* add 2 caches per previous package */

      } else if (infos->cpufamilynumber == 0x15
		 && (infos->cpumodelnumber == 0x1 /* Bulldozer */ || infos->cpumodelnumber == 0x2 /* Piledriver */)
		 && cache->level == 3 && cache->nbthreads_sharing == 6) {
	/* AMD Bulldozer and Piledriver 12-core processors have same APIC ids as Magny-Cours above,
	 * but we can't merge the checks because the original nbthreads_sharing must be exactly 6 here.
	 */
	cache->cacheid = (infos->apicid % legacy_max_log_proc) / cache->nbthreads_sharing /* cacheid within the package */
	  + 2 * (infos->apicid / legacy_max_log_proc); /* add 2 cache per previous package */
      }
    } else if (cpuid_type == hygon) {
      if (infos->cpufamilynumber == 0x18
	  && cache->level == 3 && cache->nbthreads_sharing == 6) {
        /* Hygon family 0x18 always shares L3 between 8 APIC ids,
         * even when only 6 APIC ids are enabled and reported in nbthreads_sharing
         * (on 24-core CPUs).
         */
        cache->cacheid = infos->apicid / 8;
      }
    }
  }

  if (hwloc_bitmap_isset(data->apicid_set, infos->apicid))
    data->apicid_unique = 0;
  else
    hwloc_bitmap_set(data->apicid_set, infos->apicid);
}

```cpp

这段代码定义了一个名为 `hwloc_x86_add_cpuinfos` 的函数，属于 `hwloc_x86_element` 类的成员函数。

它的作用是添加 CPU 信息，例如 CPU 架构、CPU 家族和模型，以及 CPU 步长。它接受一个 `hwloc_obj_t` 类型的对象和一个 `struct procinfo` 类型的参数，并允许在系统中重新分配这些信息。

具体来说，代码首先检查 `info` 参数中是否包含 CPU 架构信息，如果是，函数将添加 CPU 家族和模型信息。接下来，函数将获取 CPU 家族和模型的编号，并允许用户重新分配它们。然后，如果 `info` 中包含 CPU 步长信息，函数将添加 CPU 步长信息。最后，如果 `info` 中包含 CPU 架构信息，函数将添加 CPU 步长信息。


```
static void
hwloc_x86_add_cpuinfos(hwloc_obj_t obj, struct procinfo *info, int replace)
{
  char number[12];
  if (info->cpuvendor[0])
    hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUVendor", info->cpuvendor, replace);
  snprintf(number, sizeof(number), "%u", info->cpufamilynumber);
  hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUFamilyNumber", number, replace);
  snprintf(number, sizeof(number), "%u", info->cpumodelnumber);
  hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUModelNumber", number, replace);
  if (info->cpumodel[0]) {
    const char *c = info->cpumodel;
    while (*c == ' ')
      c++;
    hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUModel", c, replace);
  }
  snprintf(number, sizeof(number), "%u", info->cpustepping);
  hwloc__add_info_nodup(&obj->infos, &obj->infos_count, "CPUStepping", number, replace);
}

```cpp

This is a C function that creates a `Topology` object with a specific `unsigned type` and a specified `const char *subtype`.

It starts by creating a bitmap object with the given `unsigned kind` ofcpuset, and then it goes through all the infos that have a `unsigned type` and sets the corresponding bitmap on the given cpuset.

Then, it creates an object that represents the topology group and sets its attributes, including the `dont_merge` flag.

Finally, it inserts the object into the topology, and logs the information about the newly added object.

It should be noted that this function assumes that the topology has already been initialized and that the `unsigned type` and `const char *subtype` are passed as arguments.


```
static void
hwloc_x86_add_groups(hwloc_topology_t topology,
		     struct procinfo *infos,
		     unsigned nbprocs,
		     hwloc_bitmap_t remaining_cpuset,
		     unsigned type,
		     const char *subtype,
		     unsigned kind,
		     int dont_merge)
{
  hwloc_bitmap_t obj_cpuset;
  hwloc_obj_t obj;
  unsigned i, j;

  while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
    unsigned packageid = infos[i].ids[PKG];
    unsigned id = infos[i].ids[type];

    if (id == (unsigned)-1) {
      hwloc_bitmap_clr(remaining_cpuset, i);
      continue;
    }

    obj_cpuset = hwloc_bitmap_alloc();
    for (j = i; j < nbprocs; j++) {
      if (infos[j].ids[type] == (unsigned) -1) {
	hwloc_bitmap_clr(remaining_cpuset, j);
	continue;
      }

      if (infos[j].ids[PKG] == packageid && infos[j].ids[type] == id) {
	hwloc_bitmap_set(obj_cpuset, j);
	hwloc_bitmap_clr(remaining_cpuset, j);
      }
    }

    obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, id);
    obj->cpuset = obj_cpuset;
    obj->subtype = strdup(subtype);
    obj->attr->group.kind = kind;
    obj->attr->group.dont_merge = dont_merge;
    hwloc_debug_2args_bitmap("os %s %u has cpuset %s\n",
			     subtype, id, obj_cpuset);
    hwloc__insert_object_by_cpuset(topology, NULL, obj, "x86:group");
  }
}

```cpp

这段代码的作用是分析存储在`infos`中的信息，并相应地构建和注释顶级别。它主要做了以下几件事情：

1. 根据`flags`设置，输出或编辑以下信息：`present`（CPUIDs在进程中的当前数量）、`apicid`（CPUID的Apicaller ID）、`remaining_cpuset`（除了当前正在运行的CPU，其余CPU的集合，可能是从硬件描述符中获取的，也可能手动设置）、`level`（顶层的CPU级别，可能是静态或动态）。

2. 计算`nbprocs`（进程数）并输出。

3. 根据`flags`设置，判断是否需要同步所有CPU。如果是，那么将`fulldiscovery`（强制同步，设置为1）设置为1，并执行以下操作：

  a. 使用`hwloc_bitmap_alloc`函数，分配一个大小为`nbprocs`的已完成CPU集合，并将其存储在`complete_cpuset`中。

  b. 初始化一个布尔变量`gotnuma`为0，并设置`fulldiscovery`为1。

  c. 执行遍历，将`remaining_cpuset`中指定的CPU分配给进程，并将`gotnuma`设置为1。

  d. 如果`fulldiscovery`为1，`hwloc_bitmap_flush`函数被调用，将`complete_cpuset`中的信息保存到`infos`中，以便在分析过程中重新使用。

4. 输出信息。


```
/* Analyse information stored in infos, and build/annotate topology levels accordingly */
static void summarize(struct hwloc_backend *backend, struct procinfo *infos, unsigned long flags)
{
  struct hwloc_topology *topology = backend->topology;
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  unsigned nbprocs = data->nbprocs;
  hwloc_bitmap_t complete_cpuset = hwloc_bitmap_alloc();
  unsigned i, j, l, level;
  int one = -1;
  hwloc_bitmap_t remaining_cpuset;
  int gotnuma = 0;
  int fulldiscovery = (flags & HWLOC_X86_DISC_FLAG_FULL);

#ifdef HWLOC_DEBUG
  hwloc_debug("\nSummary of x86 CPUID topology:\n");
  for(i=0; i<nbprocs; i++) {
    hwloc_debug("PU %u present=%u apicid=%u on PKG %d CORE %d DIE %d NODE %d\n",
                i, infos[i].present, infos[i].apicid,
                infos[i].ids[PKG], infos[i].ids[CORE], infos[i].ids[DIE], infos[i].ids[NODE]);
  }
  hwloc_debug("\n");
```cpp

This is a function that sets the cache properties of a Cache object. The cache properties include the depth, size, linesize, associativity, and type of the cache. The function takes a list of CacheInformations as an input and sets the corresponding cache properties for each cache in the list. It also sets the cache CPU set and logs the cache information if the logging of the kernel is enabled.

The function first sets the depth of the cache to the level specified by the input. It then sets the size and linesize of the cache based on the size and linesize of the corresponding CacheInformations. It also sets the associativity and type of the cache based on the input.

If the function is called on a Linux system, it checks for any kernel bugs that might cause IDs to be missing. If the x86 backend is being used, the function logs the cache information even if the kernel is not expected to be bug-free.

If the function is called on a resctrl system, it sets the cache attributes to the default values specified by resctrl. It also sets the cache_cpuset to the same value as the topology->get_id() function, which is the current CPU set for the current node.

Finally, the function logs the cache information if the resctrl logging is enabled, and inserts the cache object into the topology object tree using the hwloc__insert_object_by_cpuset function.


```
#endif

  for (i = 0; i < nbprocs; i++)
    if (infos[i].present) {
      hwloc_bitmap_set(complete_cpuset, i);
      one = i;
    }

  if (one == -1) {
    hwloc_bitmap_free(complete_cpuset);
    return;
  }

  remaining_cpuset = hwloc_bitmap_alloc();

  /* Ideally, when fulldiscovery=0, we could add any object that doesn't exist yet.
   * But what if the x86 and the native backends disagree because one is buggy? Which one to trust?
   * We only add missing caches, and annotate other existing objects for now.
   */

  if (hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_PACKAGE)) {
    /* Look for packages */
    hwloc_obj_t package;

    hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
    while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
      if (fulldiscovery) {
	unsigned packageid = infos[i].ids[PKG];
	hwloc_bitmap_t package_cpuset = hwloc_bitmap_alloc();

	for (j = i; j < nbprocs; j++) {
	  if (infos[j].ids[PKG] == packageid) {
	    hwloc_bitmap_set(package_cpuset, j);
	    hwloc_bitmap_clr(remaining_cpuset, j);
	  }
	}
	package = hwloc_alloc_setup_object(topology, HWLOC_OBJ_PACKAGE, packageid);
	package->cpuset = package_cpuset;

	hwloc_x86_add_cpuinfos(package, &infos[i], 0);

	hwloc_debug_1arg_bitmap("os package %u has cpuset %s\n",
				packageid, package_cpuset);
	hwloc__insert_object_by_cpuset(topology, NULL, package, "x86:package");

      } else {
	/* Annotate packages previously-existing packages */
	hwloc_bitmap_t set = hwloc_bitmap_alloc();
	hwloc_bitmap_set(set, i);
	package = hwloc_get_next_obj_covering_cpuset_by_type(topology, set, HWLOC_OBJ_PACKAGE, NULL);
	hwloc_bitmap_free(set);
	if (package) {
	  /* Found package above that PU, annotate if no such attribute yet */
	  hwloc_x86_add_cpuinfos(package, &infos[i], 1);
	  hwloc_bitmap_andnot(remaining_cpuset, remaining_cpuset, package->cpuset);
	} else {
	  /* No package, annotate the root object */
	  hwloc_x86_add_cpuinfos(hwloc_get_root_obj(topology), &infos[i], 1);
	  break;
	}
      }
    }
  }

  /* Look for Numa nodes inside packages (cannot be filtered-out) */
  if (fulldiscovery && (flags & HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES)) {
    hwloc_bitmap_t node_cpuset;
    hwloc_obj_t node;

    /* FIXME: if there's memory inside the root object, divide it into NUMA nodes? */

    hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
    while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
      unsigned packageid = infos[i].ids[PKG];
      unsigned nodeid = infos[i].ids[NODE];

      if (nodeid == (unsigned)-1) {
        hwloc_bitmap_clr(remaining_cpuset, i);
	continue;
      }

      node_cpuset = hwloc_bitmap_alloc();
      for (j = i; j < nbprocs; j++) {
	if (infos[j].ids[NODE] == (unsigned) -1) {
	  hwloc_bitmap_clr(remaining_cpuset, j);
	  continue;
	}

        if (infos[j].ids[PKG] == packageid && infos[j].ids[NODE] == nodeid) {
          hwloc_bitmap_set(node_cpuset, j);
          hwloc_bitmap_clr(remaining_cpuset, j);
        }
      }
      node = hwloc_alloc_setup_object(topology, HWLOC_OBJ_NUMANODE, nodeid);
      node->cpuset = node_cpuset;
      node->nodeset = hwloc_bitmap_alloc();
      hwloc_bitmap_set(node->nodeset, nodeid);
      hwloc_debug_1arg_bitmap("os node %u has cpuset %s\n",
          nodeid, node_cpuset);
      hwloc__insert_object_by_cpuset(topology, NULL, node, "x86:numa");
      gotnuma++;
    }
  }

  if (hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_GROUP)) {
    if (fulldiscovery) {
      /* Look for AMD Compute units inside packages */
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      hwloc_x86_add_groups(topology, infos, nbprocs, remaining_cpuset,
			   UNIT, "Compute Unit",
			   HWLOC_GROUP_KIND_AMD_COMPUTE_UNIT, 0);
      /* Look for Intel Modules inside packages */
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      hwloc_x86_add_groups(topology, infos, nbprocs, remaining_cpuset,
			   MODULE, "Module",
			   HWLOC_GROUP_KIND_INTEL_MODULE, 0);
      /* Look for Intel Tiles inside packages */
      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      hwloc_x86_add_groups(topology, infos, nbprocs, remaining_cpuset,
			   TILE, "Tile",
			   HWLOC_GROUP_KIND_INTEL_TILE, 0);

      /* Look for unknown objects */
      if (infos[one].otherids) {
	for (level = infos[one].levels-1; level <= infos[one].levels-1; level--) {
	  if (infos[one].otherids[level] != UINT_MAX) {
	    hwloc_bitmap_t unknown_cpuset;
	    hwloc_obj_t unknown_obj;

	    hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
	    while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
	      unsigned unknownid = infos[i].otherids[level];

	      unknown_cpuset = hwloc_bitmap_alloc();
	      for (j = i; j < nbprocs; j++) {
		if (infos[j].otherids[level] == unknownid) {
		  hwloc_bitmap_set(unknown_cpuset, j);
		  hwloc_bitmap_clr(remaining_cpuset, j);
		}
	      }
	      unknown_obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_GROUP, unknownid);
	      unknown_obj->cpuset = unknown_cpuset;
	      unknown_obj->attr->group.kind = HWLOC_GROUP_KIND_INTEL_EXTTOPOENUM_UNKNOWN;
	      unknown_obj->attr->group.subkind = level;
	      hwloc_debug_2args_bitmap("os unknown%u %u has cpuset %s\n",
				       level, unknownid, unknown_cpuset);
	      hwloc__insert_object_by_cpuset(topology, NULL, unknown_obj, "x86:group:unknown");
	    }
	  }
	}
      }
    }
  }

  if (hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_DIE)) {
    /* Look for Intel Dies inside packages */
    if (fulldiscovery) {
      hwloc_bitmap_t die_cpuset;
      hwloc_obj_t die;

      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
	unsigned packageid = infos[i].ids[PKG];
	unsigned dieid = infos[i].ids[DIE];

	if (dieid == (unsigned) -1) {
	  hwloc_bitmap_clr(remaining_cpuset, i);
	  continue;
	}

	die_cpuset = hwloc_bitmap_alloc();
	for (j = i; j < nbprocs; j++) {
	  if (infos[j].ids[DIE] == (unsigned) -1) {
	    hwloc_bitmap_clr(remaining_cpuset, j);
	    continue;
	  }

	  if (infos[j].ids[PKG] == packageid && infos[j].ids[DIE] == dieid) {
	    hwloc_bitmap_set(die_cpuset, j);
	    hwloc_bitmap_clr(remaining_cpuset, j);
	  }
	}
	die = hwloc_alloc_setup_object(topology, HWLOC_OBJ_DIE, dieid);
	die->cpuset = die_cpuset;
	hwloc_debug_1arg_bitmap("os die %u has cpuset %s\n",
				dieid, die_cpuset);
	hwloc__insert_object_by_cpuset(topology, NULL, die, "x86:die");
      }
    }
  }

  if (hwloc_filter_check_keep_object_type(topology, HWLOC_OBJ_CORE)) {
    /* Look for cores */
    if (fulldiscovery) {
      hwloc_bitmap_t core_cpuset;
      hwloc_obj_t core;

      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
	unsigned packageid = infos[i].ids[PKG];
	unsigned nodeid = infos[i].ids[NODE];
	unsigned coreid = infos[i].ids[CORE];

	if (coreid == (unsigned) -1) {
	  hwloc_bitmap_clr(remaining_cpuset, i);
	  continue;
	}

	core_cpuset = hwloc_bitmap_alloc();
	for (j = i; j < nbprocs; j++) {
	  if (infos[j].ids[CORE] == (unsigned) -1) {
	    hwloc_bitmap_clr(remaining_cpuset, j);
	    continue;
	  }

	  if (infos[j].ids[PKG] == packageid && infos[j].ids[NODE] == nodeid && infos[j].ids[CORE] == coreid) {
	    hwloc_bitmap_set(core_cpuset, j);
	    hwloc_bitmap_clr(remaining_cpuset, j);
	  }
	}
	core = hwloc_alloc_setup_object(topology, HWLOC_OBJ_CORE, coreid);
	core->cpuset = core_cpuset;
	hwloc_debug_1arg_bitmap("os core %u has cpuset %s\n",
				coreid, core_cpuset);
	hwloc__insert_object_by_cpuset(topology, NULL, core, "x86:core");
      }
    }
  }

  /* Look for PUs (cannot be filtered-out) */
  if (fulldiscovery) {
    hwloc_debug("%s", "\n\n * CPU cpusets *\n\n");
    for (i=0; i<nbprocs; i++)
      if(infos[i].present) { /* Only add present PU. We don't know if others actually exist */
       struct hwloc_obj *obj = hwloc_alloc_setup_object(topology, HWLOC_OBJ_PU, i);
       obj->cpuset = hwloc_bitmap_alloc();
       hwloc_bitmap_only(obj->cpuset, i);
       hwloc_debug_1arg_bitmap("PU %u has cpuset %s\n", i, obj->cpuset);
       hwloc__insert_object_by_cpuset(topology, NULL, obj, "x86:pu");
     }
  }

  /* Look for caches */
  /* First find max level */
  level = 0;
  for (i = 0; i < nbprocs; i++)
    for (j = 0; j < infos[i].numcaches; j++)
      if (infos[i].cache[j].level > level)
        level = infos[i].cache[j].level;
  while (level > 0) {
    hwloc_obj_cache_type_t type;
    HWLOC_BUILD_ASSERT(HWLOC_OBJ_CACHE_DATA == HWLOC_OBJ_CACHE_UNIFIED+1);
    HWLOC_BUILD_ASSERT(HWLOC_OBJ_CACHE_INSTRUCTION == HWLOC_OBJ_CACHE_DATA+1);
    for (type = HWLOC_OBJ_CACHE_UNIFIED; type <= HWLOC_OBJ_CACHE_INSTRUCTION; type++) {
      /* Look for caches of that type at level level */
      hwloc_obj_type_t otype;
      hwloc_obj_t cache;

      otype = hwloc_cache_type_by_depth_type(level, type);
      if (otype == HWLOC_OBJ_TYPE_NONE)
	continue;
      if (!hwloc_filter_check_keep_object_type(topology, otype))
	continue;

      hwloc_bitmap_copy(remaining_cpuset, complete_cpuset);
      while ((i = hwloc_bitmap_first(remaining_cpuset)) != (unsigned) -1) {
	hwloc_bitmap_t puset;

	for (l = 0; l < infos[i].numcaches; l++) {
	  if (infos[i].cache[l].level == level && infos[i].cache[l].type == type)
	    break;
	}
	if (l == infos[i].numcaches) {
	  /* no cache Llevel of that type in i */
	  hwloc_bitmap_clr(remaining_cpuset, i);
	  continue;
	}

	puset = hwloc_bitmap_alloc();
	hwloc_bitmap_set(puset, i);
	cache = hwloc_get_next_obj_covering_cpuset_by_type(topology, puset, otype, NULL);
	hwloc_bitmap_free(puset);

	if (cache) {
	  /* Found cache above that PU, annotate if no such attribute yet */
	  if (!hwloc_obj_get_info_by_name(cache, "Inclusive"))
	    hwloc_obj_add_info(cache, "Inclusive", infos[i].cache[l].inclusive ? "1" : "0");
	  hwloc_bitmap_andnot(remaining_cpuset, remaining_cpuset, cache->cpuset);
	} else {
	  /* Add the missing cache */
	  hwloc_bitmap_t cache_cpuset;
	  unsigned packageid = infos[i].ids[PKG];
	  unsigned cacheid = infos[i].cache[l].cacheid;
	  /* Now look for others sharing it */
	  cache_cpuset = hwloc_bitmap_alloc();
	  for (j = i; j < nbprocs; j++) {
	    unsigned l2;
	    for (l2 = 0; l2 < infos[j].numcaches; l2++) {
	      if (infos[j].cache[l2].level == level && infos[j].cache[l2].type == type)
		break;
	    }
	    if (l2 == infos[j].numcaches) {
	      /* no cache Llevel of that type in j */
	      hwloc_bitmap_clr(remaining_cpuset, j);
	      continue;
	    }
	    if (infos[j].ids[PKG] == packageid && infos[j].cache[l2].cacheid == cacheid) {
	      hwloc_bitmap_set(cache_cpuset, j);
	      hwloc_bitmap_clr(remaining_cpuset, j);
	    }
	  }
	  cache = hwloc_alloc_setup_object(topology, otype, HWLOC_UNKNOWN_INDEX);
          /* We don't specify the os_index of caches because we want to be
           * 100% sure they are identical to what the Linux kernel reports
           * (so that things like resctrl work).
           * However, vendor/model-specific quirks in the x86 code above
           * make this difficult.
           *
           * Caveat: if the x86 backend is used on Linux to avoid kernel bugs,
           * IDs won't be available to resctrl users. But resctrl heavily
           * relies on the kernel x86 discovery being non-buggy anyway.
           *
           * TODO: make this optional? or only disable it on Linux?
           */
	  cache->attr->cache.depth = level;
	  cache->attr->cache.size = infos[i].cache[l].size;
	  cache->attr->cache.linesize = infos[i].cache[l].linesize;
	  cache->attr->cache.associativity = infos[i].cache[l].ways;
	  cache->attr->cache.type = infos[i].cache[l].type;
	  cache->cpuset = cache_cpuset;
	  hwloc_obj_add_info(cache, "Inclusive", infos[i].cache[l].inclusive ? "1" : "0");
	  hwloc_debug_2args_bitmap("os L%u cache %u has cpuset %s\n",
				   level, cacheid, cache_cpuset);
	  hwloc__insert_object_by_cpuset(topology, NULL, cache, "x86:cache");
	}
      }
    }
    level--;
  }

  /* FIXME: if KNL and L2 disabled, add tiles instead of L2 */

  hwloc_bitmap_free(remaining_cpuset);
  hwloc_bitmap_free(complete_cpuset);

  if (gotnuma)
    topology->support.discovery->numa = 1;
}

```cpp

This function appears to be part of a system that manages hybrid information for CPUinds (e.g. information about the process and the cores). It takes a bitmap of information and allows the caller to specify which types of hybrid information to display.

The function first checks which cores have a specific hybrid core type and sets the appropriate bitmap for that coreset. It then checks which processors have the IntelAtom set and sets the bitmap for that coreset. Finally, it checks which processors have the IntelCore set and sets the bitmap for that coreset.

If the bitmap for any of the cores is not empty, the function registers the coreset for that processor and the corresponding bitmap with the IntelAtom or IntelCore set. It also, if the bitmap is not empty, it checks if it contains any information for the APIC ID unique. If it does not, the function does nothing and returns success. If it does, the function registers the coreset for that processor and returns success.


```
static int
look_procs(struct hwloc_backend *backend, struct procinfo *infos, unsigned long flags,
	   unsigned highest_cpuid, unsigned highest_ext_cpuid, unsigned *features, enum cpuid_type cpuid_type,
	   int (*get_cpubind)(hwloc_topology_t topology, hwloc_cpuset_t set, int flags),
	   int (*set_cpubind)(hwloc_topology_t topology, hwloc_const_cpuset_t set, int flags),
           hwloc_bitmap_t restrict_set)
{
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  struct hwloc_topology *topology = backend->topology;
  unsigned nbprocs = data->nbprocs;
  hwloc_bitmap_t orig_cpuset = NULL;
  hwloc_bitmap_t set = NULL;
  unsigned i;

  if (!data->src_cpuiddump_path) {
    orig_cpuset = hwloc_bitmap_alloc();
    if (get_cpubind(topology, orig_cpuset, HWLOC_CPUBIND_STRICT)) {
      hwloc_bitmap_free(orig_cpuset);
      return -1;
    }
    set = hwloc_bitmap_alloc();
  }

  for (i = 0; i < nbprocs; i++) {
    struct cpuiddump *src_cpuiddump = NULL;

    if (restrict_set && !hwloc_bitmap_isset(restrict_set, i)) {
      /* skip this CPU outside of the binding mask */
      continue;
    }

    if (data->src_cpuiddump_path) {
      src_cpuiddump = cpuiddump_read(data->src_cpuiddump_path, i);
      if (!src_cpuiddump)
	continue;
    } else {
      hwloc_bitmap_only(set, i);
      hwloc_debug("binding to CPU%u\n", i);
      if (set_cpubind(topology, set, HWLOC_CPUBIND_STRICT)) {
	hwloc_debug("could not bind to CPU%u: %s\n", i, strerror(errno));
	continue;
      }
    }

    look_proc(backend, &infos[i], flags, highest_cpuid, highest_ext_cpuid, features, cpuid_type, src_cpuiddump);

    if (data->src_cpuiddump_path) {
      cpuiddump_free(src_cpuiddump);
    }
  }

  if (!data->src_cpuiddump_path) {
    set_cpubind(topology, orig_cpuset, 0);
    hwloc_bitmap_free(set);
    hwloc_bitmap_free(orig_cpuset);
  }

  if (data->apicid_unique) {
    summarize(backend, infos, flags);

    if (has_hybrid(features) && !(topology->flags & HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS)) {
      /* use hybrid info for cpukinds */
      hwloc_bitmap_t atomset = hwloc_bitmap_alloc();
      hwloc_bitmap_t coreset = hwloc_bitmap_alloc();
      for(i=0; i<nbprocs; i++) {
        if (infos[i].hybridcoretype == 0x20)
          hwloc_bitmap_set(atomset, i);
        else if (infos[i].hybridcoretype == 0x40)
          hwloc_bitmap_set(coreset, i);
      }
      /* register IntelAtom set if any */
      if (!hwloc_bitmap_iszero(atomset)) {
        struct hwloc_info_s infoattr;
        infoattr.name = (char *) "CoreType";
        infoattr.value = (char *) "IntelAtom";
        hwloc_internal_cpukinds_register(topology, atomset, HWLOC_CPUKIND_EFFICIENCY_UNKNOWN, &infoattr, 1, 0);
        /* the cpuset is given to the callee */
      } else {
        hwloc_bitmap_free(atomset);
      }
      /* register IntelCore set if any */
      if (!hwloc_bitmap_iszero(coreset)) {
        struct hwloc_info_s infoattr;
        infoattr.name = (char *) "CoreType";
        infoattr.value = (char *) "IntelCore";
        hwloc_internal_cpukinds_register(topology, coreset, HWLOC_CPUKIND_EFFICIENCY_UNKNOWN, &infoattr, 1, 0);
        /* the cpuset is given to the callee */
      } else {
        hwloc_bitmap_free(coreset);
      }
    }
  }
  /* if !data->apicid_unique, do nothing and return success, so that the caller does nothing either */

  return 0;
}

```cpp

这段代码定义了两个函数：`hwloc_x86_os_state_save` 和 `hwloc_x86_os_state_restore`。这两个函数用于在 `hwloc_x86_os_state_t` 结构体中保存和恢复 `CPU ID` 数据。

`hwloc_x86_os_state_save` 函数用于在发现过程中，将所有可用 CPU ID 设置为初始值。它的实现包括：

1. 如果传入了 `src_cpuiddump` 作为参数，那么不做任何操作。
2. 否则，获取当前所有 CPU 的 ID，并将它们设置为 0。
3. 将第一个 CPU 设置为不可用，第二个 CPU 设置为可用的状态。

`hwloc_x86_os_state_restore` 函数用于将保存的 CPU ID 数据还原到 `hwloc_x86_os_state_t` 结构体中。它的实现包括：

1. 如果传入了 `src_cpuiddump` 作为参数，那么不做任何操作。
2. 否则，将所有 CPU 设置为可用的状态，并将第一个 CPU 设置为不可用的状态。

这两个函数的实现是为了在 `hwloc_x86_os_state_t` 结构体中保存和恢复 `CPU ID` 数据，以便在发现过程中，以及在之后的调度过程中，正确地设置 CPU。


```
#if defined HWLOC_FREEBSD_SYS && defined HAVE_CPUSET_SETID
#include <sys/param.h>
#include <sys/cpuset.h>
typedef cpusetid_t hwloc_x86_os_state_t;
static void hwloc_x86_os_state_save(hwloc_x86_os_state_t *state, struct cpuiddump *src_cpuiddump)
{
  if (!src_cpuiddump) {
    /* temporary make all cpus available during discovery */
    cpuset_getid(CPU_LEVEL_CPUSET, CPU_WHICH_PID, -1, state);
    cpuset_setid(CPU_WHICH_PID, -1, 0);
  }
}
static void hwloc_x86_os_state_restore(hwloc_x86_os_state_t *state, struct cpuiddump *src_cpuiddump)
{
  if (!src_cpuiddump) {
    /* restore initial cpuset */
    cpuset_setid(CPU_WHICH_PID, -1, *state);
  }
}
```cpp

这段代码定义了两个名为`hwloc_x86_os_state_t`的别名，用于表示硬件ID的`hwloc_x86_os_state_t`类型的变量。这两个别名都有一个指向`struct cpuiddump`类型的`src_cpuiddump`参数和一个指向`hwloc_x86_os_state_t`类型的`state`参数。

这两个函数分别名为`hwloc_x86_os_state_save`和`hwloc_x86_os_state_restore`，用于在`hwloc_x86_os_state_t`类型的变量被使用时保存或恢复`state`变量的值，同时保存或恢复`src_cpuiddump`参数的值。

函数内部使用宏定义来根据`HWLOC_FREEBSD_SYS`和`HAVE_CPUSET_SETID`两个条件是否定义来定义`hwloc_x86_os_state_t`的别名。如果未定义这两个条件，则定义`hwloc_x86_os_state_t`为`void *`类型，表示一个空指针类型。


```
#else /* !defined HWLOC_FREEBSD_SYS || !defined HAVE_CPUSET_SETID */
typedef void * hwloc_x86_os_state_t;
static void hwloc_x86_os_state_save(hwloc_x86_os_state_t *state __hwloc_attribute_unused, struct cpuiddump *src_cpuiddump __hwloc_attribute_unused) { }
static void hwloc_x86_os_state_restore(hwloc_x86_os_state_t *state __hwloc_attribute_unused, struct cpuiddump *src_cpuiddump __hwloc_attribute_unused) { }
#endif /* !defined HWLOC_FREEBSD_SYS || !defined HAVE_CPUSET_SETID */

/* GenuineIntel */
#define INTEL_EBX ('G' | ('e'<<8) | ('n'<<16) | ('u'<<24))
#define INTEL_EDX ('i' | ('n'<<8) | ('e'<<16) | ('I'<<24))
#define INTEL_ECX ('n' | ('t'<<8) | ('e'<<16) | ('l'<<24))

/* AuthenticAMD */
#define AMD_EBX ('A' | ('u'<<8) | ('t'<<16) | ('h'<<24))
#define AMD_EDX ('e' | ('n'<<8) | ('t'<<16) | ('i'<<24))
#define AMD_ECX ('c' | ('A'<<8) | ('M'<<16) | ('D'<<24))

```cpp

这段代码定义了三个宏，分别代表 CentaurHauls，Shanghai 和 Zhaoxin。它们定义了三个字符串类型的变量，分别代表这三个地点的外围键。这些键被划分为不同的位置，使用了不同的高斯格式。通过这些宏，可以很方便地根据需要对外围键进行操作。


```
/* HYGON "HygonGenuine" */
#define HYGON_EBX ('H' | ('y'<<8) | ('g'<<16) | ('o'<<24))
#define HYGON_EDX ('n' | ('G'<<8) | ('e'<<16) | ('n'<<24))
#define HYGON_ECX ('u' | ('i'<<8) | ('n'<<16) | ('e'<<24))

/* (Zhaoxin) CentaurHauls */
#define ZX_EBX ('C' | ('e'<<8) | ('n'<<16) | ('t'<<24))
#define ZX_EDX ('a' | ('u'<<8) | ('r'<<16) | ('H'<<24))
#define ZX_ECX ('a' | ('u'<<8) | ('l'<<16) | ('s'<<24))
/* (Zhaoxin) Shanghai */
#define SH_EBX (' ' | (' '<<8) | ('S'<<16) | ('h'<<24))
#define SH_EDX ('a' | ('n'<<8) | ('g'<<16) | ('h'<<24))
#define SH_ECX ('a' | ('i'<<8) | (' '<<16) | (' '<<24))

/* fake cpubind for when nbprocs=1 and no binding support */
```cpp

这段代码的作用是检测 CPU 型号并设置相应的硬件支持。首先，获取当前 CPU 的最高扩展 CPU ID，并将其存储在 ecx 变量中。然后，通过调用函数 `cpuid_or_from_dump()`，将当前 CPU 的 ID 存储到 eax 变量中，并将 eax 赋值为 0x80000001，表示这是一个 Intel 处理器。接下来，根据当前 CPU 的类型，设置不同的硬件支持。最后，将最高扩展 CPU ID 存储回 ecx 变量中，并执行一些硬件相关的操作。


```
static int fake_get_cpubind(hwloc_topology_t topology __hwloc_attribute_unused,
			    hwloc_cpuset_t set __hwloc_attribute_unused,
			    int flags __hwloc_attribute_unused)
{
  return 0;
}
static int fake_set_cpubind(hwloc_topology_t topology __hwloc_attribute_unused,
			    hwloc_const_cpuset_t set __hwloc_attribute_unused,
			    int flags __hwloc_attribute_unused)
{
  return 0;
}

static
int hwloc_look_x86(struct hwloc_backend *backend, unsigned long flags)
{
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  struct hwloc_topology *topology = backend->topology;
  unsigned nbprocs = data->nbprocs;
  unsigned eax, ebx, ecx = 0, edx;
  unsigned i;
  unsigned highest_cpuid;
  unsigned highest_ext_cpuid;
  /* This stores cpuid features with the same indexing as Linux */
  unsigned features[19] = { 0 };
  struct procinfo *infos = NULL;
  enum cpuid_type cpuid_type = unknown;
  hwloc_x86_os_state_t os_state;
  struct hwloc_binding_hooks hooks;
  struct hwloc_topology_support support;
  struct hwloc_topology_membind_support memsupport __hwloc_attribute_unused;
  int (*get_cpubind)(hwloc_topology_t topology, hwloc_cpuset_t set, int flags) = NULL;
  int (*set_cpubind)(hwloc_topology_t topology, hwloc_const_cpuset_t set, int flags) = NULL;
  hwloc_bitmap_t restrict_set = NULL;
  struct cpuiddump *src_cpuiddump = NULL;
  int ret = -1;

  /* check if binding works */
  memset(&hooks, 0, sizeof(hooks));
  support.membind = &memsupport;
  /* We could just copy the main hooks (except in some corner cases),
   * but the current overhead is negligible, so just always reget them.
   */
  hwloc_set_native_binding_hooks(&hooks, &support);
  /* in theory, those are only needed if !data->src_cpuiddump_path || HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_BINDING
   * but that's the vast majority of cases anyway, and the overhead is very small.
   */

  if (data->src_cpuiddump_path) {
    /* Just read cpuid from the dump (implies !topology->is_thissystem by default) */
    src_cpuiddump = cpuiddump_read(data->src_cpuiddump_path, 0);
    if (!src_cpuiddump)
      goto out;

  } else {
    /* Using real hardware.
     * However we don't enforce topology->is_thissystem so that
     * we may still force use this backend when debugging with !thissystem.
     */

    if (hooks.get_thisthread_cpubind && hooks.set_thisthread_cpubind) {
      get_cpubind = hooks.get_thisthread_cpubind;
      set_cpubind = hooks.set_thisthread_cpubind;
    } else if (hooks.get_thisproc_cpubind && hooks.set_thisproc_cpubind) {
      /* FIXME: if called by a multithreaded program, we will restore the original process binding
       * for each thread instead of their own original thread binding.
       * See issue #158.
       */
      get_cpubind = hooks.get_thisproc_cpubind;
      set_cpubind = hooks.set_thisproc_cpubind;
    } else {
      /* we need binding support if there are multiple PUs */
      if (nbprocs > 1)
	goto out;
      get_cpubind = fake_get_cpubind;
      set_cpubind = fake_set_cpubind;
    }
  }

  if (topology->flags & HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING) {
    restrict_set = hwloc_bitmap_alloc();
    if (!restrict_set)
      goto out;
    if (hooks.get_thisproc_cpubind)
      hooks.get_thisproc_cpubind(topology, restrict_set, 0);
    else if (hooks.get_thisthread_cpubind)
      hooks.get_thisthread_cpubind(topology, restrict_set, 0);
    if (hwloc_bitmap_iszero(restrict_set)) {
      hwloc_bitmap_free(restrict_set);
      restrict_set = NULL;
    }
  }

  if (!src_cpuiddump && !hwloc_have_x86_cpuid())
    goto out;

  infos = calloc(nbprocs, sizeof(struct procinfo));
  if (NULL == infos)
    goto out;
  for (i = 0; i < nbprocs; i++) {
    infos[i].ids[PKG] = (unsigned) -1;
    infos[i].ids[CORE] = (unsigned) -1;
    infos[i].ids[NODE] = (unsigned) -1;
    infos[i].ids[UNIT] = (unsigned) -1;
    infos[i].ids[TILE] = (unsigned) -1;
    infos[i].ids[MODULE] = (unsigned) -1;
    infos[i].ids[DIE] = (unsigned) -1;
  }

  eax = 0x00;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  highest_cpuid = eax;
  if (ebx == INTEL_EBX && ecx == INTEL_ECX && edx == INTEL_EDX)
    cpuid_type = intel;
  else if (ebx == AMD_EBX && ecx == AMD_ECX && edx == AMD_EDX)
    cpuid_type = amd;
  else if ((ebx == ZX_EBX && ecx == ZX_ECX && edx == ZX_EDX)
	   || (ebx == SH_EBX && ecx == SH_ECX && edx == SH_EDX))
    cpuid_type = zhaoxin;
  else if (ebx == HYGON_EBX && ecx == HYGON_ECX && edx == HYGON_EDX)
    cpuid_type = hygon;

  hwloc_debug("highest cpuid %x, cpuid type %u\n", highest_cpuid, cpuid_type);
  if (highest_cpuid < 0x01) {
      goto out_with_infos;
  }

  eax = 0x01;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  features[0] = edx;
  features[4] = ecx;

  eax = 0x80000000;
  cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
  highest_ext_cpuid = eax;

  hwloc_debug("highest extended cpuid %x\n", highest_ext_cpuid);

  if (highest_cpuid >= 0x7) {
    eax = 0x7;
    ecx = 0;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    features[9] = ebx;
    features[18] = edx;
  }

  if (cpuid_type != intel && highest_ext_cpuid >= 0x80000001) {
    eax = 0x80000001;
    cpuid_or_from_dump(&eax, &ebx, &ecx, &edx, src_cpuiddump);
    features[1] = edx;
    features[6] = ecx;
  }

  hwloc_x86_os_state_save(&os_state, src_cpuiddump);

  ret = look_procs(backend, infos, flags,
		   highest_cpuid, highest_ext_cpuid, features, cpuid_type,
		   get_cpubind, set_cpubind, restrict_set);
  if (!ret)
    /* success, we're done */
    goto out_with_os_state;

  if (nbprocs == 1) {
    /* only one processor, no need to bind */
    look_proc(backend, &infos[0], flags, highest_cpuid, highest_ext_cpuid, features, cpuid_type, src_cpuiddump);
    summarize(backend, infos, flags);
    ret = 0;
  }

```cpp

这段代码定义了一个名为`out_with_os_state`的函数，它的作用是保存输出进程的`OSState`信息，如果已经保存了信息，则执行以下操作：

1. 如果已经保存了`OSState`信息，则执行以下操作：
  1. 从`infos`数组中遍历每个进程的`OSState`信息，并免费内存：
     ```javascript
     for (i = 0; i < nbprocs; i++) {
       free(infos[i].cache);
       free(infos[i].otherids);
     }
     free(infos);
     ```cpp

2. 如果`infos`数组为空，则执行以下操作：
   ```javascript
   for (i = 0; i < nbprocs; i++) {
     free(infos[i].cache);
     free(infos[i].otherids);
   }
   free(infos);
   ```cpp

3. 从`restrict_set`输出区域分配内存，并将`src_cpuiddump`作为参数传递给`cpuiddump_free`函数，同时返回`ret`表示OSState的恢复成功。


```
out_with_os_state:
  hwloc_x86_os_state_restore(&os_state, src_cpuiddump);

out_with_infos:
  if (NULL != infos) {
    for (i = 0; i < nbprocs; i++) {
      free(infos[i].cache);
      free(infos[i].otherids);
    }
    free(infos);
  }

out:
  hwloc_bitmap_free(restrict_set);
  if (src_cpuiddump)
    cpuiddump_free(src_cpuiddump);
  return ret;
}

```cpp

这段代码是一个名为 `hwloc_x86_discover` 的函数，属于 `hwloc_backend` 结构体的成员函数。它的作用是发现并返回一个 `hwloc_topology` 结构体，其中包含用于 `hwloc_disc_status` 结构体的数据。

函数首先定义了几个变量，包括 `data` 指针、`topology` 指针和 `flags` 变量。然后进行一系列条件判断，包括检查 `dstatus` 结构体中 `phase` 是否为 `HWLOC_DISC_PHASE_CPU`，如果是，就表示当前过程只在此核心上运行，就不会改变绑定状态，然后判断一些条件，如 `topology` 结构体中是否包含 `HWLOC_TOPOLOGY_FLAG_DON_CHANGE_BINDING`，如果包含则不执行更改，否则就需要更改绑定状态。接着检查是否有 `HWLOC_X86_TOPOEXT_NUMANODES` 环境变量，如果有，则设置 `flags` 变量为 `HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES` 的一半，否则不需要更改。

最后，函数使用 `ret` 变量返回一个 `hwloc_topology` 结构体，其中包含用于 `hwloc_disc_status` 结构体的数据。


```
static int
hwloc_x86_discover(struct hwloc_backend *backend, struct hwloc_disc_status *dstatus)
{
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  struct hwloc_topology *topology = backend->topology;
  unsigned long flags = 0;
  int alreadypus = 0;
  int ret;

  assert(dstatus->phase == HWLOC_DISC_PHASE_CPU);

  if (topology->flags & HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING) {
    /* TODO: Things would work if there's a single PU, no need to rebind */
    return 0;
  }

  if (getenv("HWLOC_X86_TOPOEXT_NUMANODES")) {
    flags |= HWLOC_X86_DISC_FLAG_TOPOEXT_NUMANODES;
  }

```cpp

This is a program that performs a DiscoverX operation. DiscoverX is a software development kit (SDK) that enables developers to discover and analyze high-performance systems with the HWLOC (硬件布局) information.

The program seems to be using Valgrind to run the DiscoverX operation, as it mentions that it cannot work under Valgrind, disabling it. It then checks if the system is running Valgrind and if it is, it disable the DiscoverX operation. May be it is running some other Valgrind based tools or configurations that丁自居， in that case, it might not work under Valgrind.

Then it checks if the system has any active CPUs, and if so, it creates a new topology level for the DiscoverX operation. If there is no active CPUs, it initializes a new topology level with a single process group for the default system.

It then seems to be performing some sort of topology discovery, but it is not clear what it is actually doing. It mentions that it may be able to gather more information about the system if it is running a different discover tool like hwloc-gather-cpuid, but it is not clear if it is using that tool.

It also seems to be using the HWLOC_CPUID_PATH to dump the CPUIDs of the system, which is a path to the hardware layout of the system. It then reloads the CPUIDs with the information it gathered, and then it seems to be doing some sort of processing with the CPUIDs, but it is not clear what it is doing.

Overall, it seems to be using a combination of Valgrind, topology information, and CPUID dumping to perform some sort of DiscoverX operation, but it is not clear what it is actually doing.


```
#if HAVE_DECL_RUNNING_ON_VALGRIND
  if (RUNNING_ON_VALGRIND && !data->src_cpuiddump_path) {
    fprintf(stderr, "hwloc x86 backend cannot work under Valgrind, disabling.\n"
	    "May be reenabled by dumping CPUIDs with hwloc-gather-cpuid\n"
	    "and reloading them under Valgrind with HWLOC_CPUID_PATH.\n");
    return 0;
  }
#endif

  if (data->src_cpuiddump_path) {
    assert(data->nbprocs > 0); /* enforced by hwloc_x86_component_instantiate() */
    topology->support.discovery->pu = 1;
  } else {
    int nbprocs = hwloc_fallback_nbprocessors(HWLOC_FALLBACK_NBPROCESSORS_INCLUDE_OFFLINE);
    if (nbprocs >= 1)
      topology->support.discovery->pu = 1;
    else
      nbprocs = 1;
    data->nbprocs = (unsigned) nbprocs;
  }

  if (topology->levels[0][0]->cpuset) {
    /* somebody else discovered things, reconnect levels so that we can look at them */
    hwloc_topology_reconnect(topology, 0);
    if (topology->nb_levels == 2 && topology->level_nbobjects[1] == data->nbprocs) {
      /* only PUs were discovered, as much as we would, complete the topology with everything else */
      alreadypus = 1;
      goto fulldiscovery;
    }

    /* several object types were added, we can't easily complete, just do partial discovery */
    ret = hwloc_look_x86(backend, flags);
    if (ret)
      hwloc_obj_add_info(topology->levels[0][0], "Backend", "x86");
    return 0;
  } else {
    /* topology is empty, initialize it */
    hwloc_alloc_root_sets(topology->levels[0][0]);
  }

```cpp

这段代码是用来设置一个基于x86架构的系统中的硬件资源。它的主要作用是检查在尝试使用hwloc_look_x86函数时是否失败，如果失败，则创建PPU（并行处理器单元）。

具体来说，代码首先检查本地硬件是否支持x86架构，如果硬件不支持，则尝试使用alreadypus变量来判断是否已经创建了PPU。如果没有创建PPU，则创建一个并行处理器单元，并将其添加到topology中。

接下来，代码会检查是否已经设置好了CPU ID dump文件路径，如果没有设置，则会手动设置CPU架构信息。如果已经设置好了，则检查是否支持x86架构，如果是，则使用hwloc_obj_add_info函数添加一个名为"Backend"的子系统，其值为"x86"。

最后，代码使用hwloc_add_uname_info函数添加一个名为"Architecture"的子系统，如果已经设置好了系统为x86_64架构，则使用alreadypus变量来设置，否则需要手动设置。


```
fulldiscovery:
  if (hwloc_look_x86(backend, flags | HWLOC_X86_DISC_FLAG_FULL) < 0) {
    /* if failed, create PUs */
    if (!alreadypus)
      hwloc_setup_pu_level(topology, data->nbprocs);
  }

  hwloc_obj_add_info(topology->levels[0][0], "Backend", "x86");

  if (!data->src_cpuiddump_path) { /* CPUID dump works for both x86 and x86_64 */
#ifdef HAVE_UNAME
    hwloc_add_uname_info(topology, NULL); /* we already know is_thissystem() is true */
#else
    /* uname isn't available, manually setup the "Architecture" info */
#ifdef HWLOC_X86_64_ARCH
    hwloc_obj_add_info(topology->levels[0][0], "Architecture", "x86_64");
```cpp

This is a C program that searches for evidence of undumped CPUs in a directory. The program first checks if the input file is a valid Linux header and then searches for lines containing the string "pu". If a valid header is found, the program searches for lines containing the string "pu" followed by a space. If such a line is found, the program checks if the line starts with the string "Architecture: x86". If it does, the program prints a message and then proceeds to the next step. If it doesn't, the program prints a message and then jumps to the next step. This step is repeated for each entry in the directory. The program also checks if the set of PUs contains any virtual CPUUs that are not contiguous and prints a message if it does. If the program finishes all the checks and finds no evidence of undumped CPUs, it returns 0. Otherwise, it returns -1.


```
#else
    hwloc_obj_add_info(topology->levels[0][0], "Architecture", "x86");
#endif
#endif
  }

  return 1;
}

static int
hwloc_x86_check_cpuiddump_input(const char *src_cpuiddump_path, hwloc_bitmap_t set)
{

#if !(defined HWLOC_WIN_SYS && !defined __MINGW32__ && !defined __CYGWIN__) /* needs a lot of work */
  struct dirent *dirent;
  DIR *dir;
  char *path;
  FILE *file;
  char line [32];

  dir = opendir(src_cpuiddump_path);
  if (!dir) 
    return -1;

  path = malloc(strlen(src_cpuiddump_path) + strlen("/hwloc-cpuid-info") + 1);
  if (!path)
    goto out_with_dir;
  sprintf(path, "%s/hwloc-cpuid-info", src_cpuiddump_path);
  file = fopen(path, "r");
  if (!file) {
    fprintf(stderr, "Couldn't open dumped cpuid summary %s\n", path);
    goto out_with_path;
  }
  if (!fgets(line, sizeof(line), file)) {
    fprintf(stderr, "Found read dumped cpuid summary in %s\n", path);
    fclose(file);
    goto out_with_path;
  }
  fclose(file);
  if (strcmp(line, "Architecture: x86\n")) {
    fprintf(stderr, "Found non-x86 dumped cpuid summary in %s: %s\n", path, line);
    goto out_with_path;
  }
  free(path);

  while ((dirent = readdir(dir)) != NULL) {
    if (!strncmp(dirent->d_name, "pu", 2)) {
      char *end;
      unsigned long idx = strtoul(dirent->d_name+2, &end, 10);
      if (!*end)
	hwloc_bitmap_set(set, idx);
      else
	fprintf(stderr, "Ignoring invalid dirent `%s' in dumped cpuid directory `%s'\n",
		dirent->d_name, src_cpuiddump_path);
    }
  }
  closedir(dir);

  if (hwloc_bitmap_iszero(set)) {
    fprintf(stderr, "Did not find any valid pu%%u entry in dumped cpuid directory `%s'\n",
	    src_cpuiddump_path);
    return -1;
  } else if (hwloc_bitmap_last(set) != hwloc_bitmap_weight(set) - 1) {
    /* The x86 backends enforces contigous set of PUs starting at 0 so far */
    fprintf(stderr, "Found non-contigous pu%%u range in dumped cpuid directory `%s'\n",
	    src_cpuiddump_path);
    return -1;
  }

  return 0;

 out_with_path:
  free(path);
 out_with_dir:
  closedir(dir);
```cpp

This is a C function that creates an instance of the HWLOC_CPUID_POSIX backend. The backend is responsible for managing the hardware resources related to the CPU IDs.

Here's how you can use it:

1. First, you need to get a C context for this backend. You can do this by calling `hwloc_backend_init()` which will return a handle to the backend.
2. Next, you need to allocate memory for the data structure that will hold the information about the CPU IDs. You can do this by calling `malloc()` and passing it the memory size of the data structure.
3. Then, you need to set up the data structure that will hold the information about the CPU IDs. You can do this by initializing all the fields of the data structure to zero.
4. After that, you need to set the source CPU ID dump directory if it has not already been set.
5. Finally, you can return the handle to the HWLOC_CPUID_POSIX backend.

Note: This function assumes that the input data is a valid instance of the HWLOC_CPUID_POSIX data structure and that the HWLOC_CPUID_POSIX backend has been properly initialized.


```
#endif /* HWLOC_WIN_SYS & !__MINGW32__ needs a lot of work */
  return -1;
}

static void
hwloc_x86_backend_disable(struct hwloc_backend *backend)
{
  struct hwloc_x86_backend_data_s *data = backend->private_data;
  hwloc_bitmap_free(data->apicid_set);
  free(data->src_cpuiddump_path);
  free(data);
}

static struct hwloc_backend *
hwloc_x86_component_instantiate(struct hwloc_topology *topology,
				struct hwloc_disc_component *component,
				unsigned excluded_phases __hwloc_attribute_unused,
				const void *_data1 __hwloc_attribute_unused,
				const void *_data2 __hwloc_attribute_unused,
				const void *_data3 __hwloc_attribute_unused)
{
  struct hwloc_backend *backend;
  struct hwloc_x86_backend_data_s *data;
  const char *src_cpuiddump_path;

  backend = hwloc_backend_alloc(topology, component);
  if (!backend)
    goto out;

  data = malloc(sizeof(*data));
  if (!data) {
    errno = ENOMEM;
    goto out_with_backend;
  }

  backend->private_data = data;
  backend->discover = hwloc_x86_discover;
  backend->disable = hwloc_x86_backend_disable;

  /* default values */
  data->is_knl = 0;
  data->apicid_set = hwloc_bitmap_alloc();
  data->apicid_unique = 1;
  data->src_cpuiddump_path = NULL;

  src_cpuiddump_path = getenv("HWLOC_CPUID_PATH");
  if (src_cpuiddump_path) {
    hwloc_bitmap_t set = hwloc_bitmap_alloc();
    if (!hwloc_x86_check_cpuiddump_input(src_cpuiddump_path, set)) {
      backend->is_thissystem = 0;
      data->src_cpuiddump_path = strdup(src_cpuiddump_path);
      assert(!hwloc_bitmap_iszero(set)); /* enforced by hwloc_x86_check_cpuiddump_input() */
      data->nbprocs = hwloc_bitmap_weight(set);
    } else {
      fprintf(stderr, "Ignoring dumped cpuid directory.\n");
    }
    hwloc_bitmap_free(set);
  }

  return backend;

 out_with_backend:
  free(backend);
 out:
  return NULL;
}

```cpp

这段代码定义了一个 `hwloc_disc_component` 结构体，该结构体包含了在本地硬件设备上执行的任务。

具体来说，`hwloc_disc_component` 结构体包含了以下成员：

1. `"x86"`：这是一个字符串，表示组件的类型为 x86 类型的本地硬件设备。
2. `HWLOC_DISC_PHASE_CPU`：这是一个表示 CPU 执行阶段的枚举类型，用于指定在本地硬件设备上如何使用 CPU。
3. `HWLOC_DISC_PHASE_GLOBAL`：这也是一个表示 CPU 执行阶段的枚举类型，用于指定在本地硬件设备上如何使用全局 CPU 执行阶段。
4. `hwloc_x86_component_instantiate`：这是一个函数指针，指向组件的实例化函数。
5. `45`：这是一个整数，表示在本地硬件设备上，这个组件的执行优先级，值范围为 `between_native_and_no_os` 类型的值。
6. `1`：这是一个整数，表示在 `hwloc_x86_disc_component` 中，这个组件的 `eager_start` 标志。
7. `NULL`：这是一个指向 `NULL` 的指针，用于保存 `hwloc_x86_disc_component` 的 `null_generate` 函数的指针。

`hwloc_disc_component` 的实例化函数在定义时被调用，并在 `hwloc_x86_disc_component` 的 `null_generate` 函数中执行。如果没有执行 `hwloc_disc_component` 的 `null_generate` 函数，组件将无法分配内存，因此其行为将取决于定义的硬件设备。


```
static struct hwloc_disc_component hwloc_x86_disc_component = {
  "x86",
  HWLOC_DISC_PHASE_CPU,
  HWLOC_DISC_PHASE_GLOBAL,
  hwloc_x86_component_instantiate,
  45, /* between native and no_os */
  1,
  NULL
};

const struct hwloc_component hwloc_x86_component = {
  HWLOC_COMPONENT_ABI,
  NULL, NULL,
  HWLOC_COMPONENT_TYPE_DISC,
  0,
  &hwloc_x86_disc_component
};

```