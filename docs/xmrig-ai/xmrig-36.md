# xmrig源码解析 36

# `src/3rdparty/hwloc/src/shmem.c`

这段代码是一个C语言编写的共享内存的源代码。它的目的是在Linux系统上实现一个高性能的并行文件系统。

它包括以下几个部分：

1. 头文件包含：private/autogen/config.h,private/private.h，和hwloc.h。

2. 函数声明：包含在头文件中的函数声明，如配置文件读取、磁盘文件读写等。

3. 源文件包含：包含在头文件中的源文件，如autogen/config.c和private/private.c。

4. 内存管理函数实现：实现了磁盘文件读写和内存映射，使得多个进程可以共享同一个文件系统。

5. 包含hwloc.h：因为hwloc.h需要这个头文件。

6. 包含_getenvvar：用于获取一个envvar的值。

7. 包含unistd.h：因为unistd.h包含一些与操作系统相关的函数。

8. 包含sys/mman.h：包含mman.h，这个头文件包含与共享内存相关的函数。

9. 包含<unistd.h>：包含unistd.h，这个头文件包含与操作系统相关的函数。

该代码的作用是实现一个高性能的并行文件系统，可以让多个进程共享同一个文件系统。


```cpp
/*
 * Copyright © 2017-2020 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "hwloc/shmem.h"
#include "private/private.h"

#ifndef HWLOC_WIN_SYS

#include <sys/mman.h>
#ifdef HAVE_UNISTD_H
#include <unistd.h>
```

这段代码定义了一个名为`hwloc_shmem_header`的结构体，以及一个名为`HWLOC_SHMEM_HEADER_VERSION`的常量，表示`hwloc_shmem_header`的头部版本为1。接下来定义了一个`struct hwloc_shmem_header`的成员变量，分别表示`header_version`、`header_length`和`mmap_address`和`mmap_length`，其中`header_version`和`header_length`用于定义`hwloc_shmem_header`结构体的成员变量。然后定义了一个常量`HWLOC_SHMEM_MALLOC_ALIGN`，表示`mmap`内存分配的对齐大小为8字节。

接下来的代码是一个宏定义，其中`#define HWLOC_SHMEM_HEADER_VERSION 1`定义了一个宏，将`HWLOC_SHMEM_HEADER_VERSION`的值1作为`hwloc_shmem_header`结构体头部的版本号。

然后定义了一个名为`hwloc_shmem_malloc`的函数，它是一个私有函数，没有定义函数体，只是声明了一个返回类型为`void`的函数。由于没有定义函数体，所以无法确定该函数的具体实现。


```cpp
#endif
#include <assert.h>

#define HWLOC_SHMEM_HEADER_VERSION 1

struct hwloc_shmem_header {
  uint32_t header_version; /* sanity check */
  uint32_t header_length; /* where the actual topology starts in the file/mapping */
  uint64_t mmap_address; /* virtual address to pass to mmap */
  uint64_t mmap_length; /* length to pass to mmap (includes the header) */
};

#define HWLOC_SHMEM_MALLOC_ALIGN 8UL

static void *
```

这两段代码是针对tma_shmem_malloc函数和tma_get_length_malloc函数的定义，用于在tma_shmem结构体中分配和获取内存。

tma_shmem_malloc函数的作用是向tma_shmem结构体中分配内存，并返回该结构体的数据指针。它的参数包括两个结构体指针tma和length，分别表示待分配内存和该内存的长度。函数的具体实现是先将当前内存块的地址赋给tma->data，然后将待分配内存的长度计算出来(即在tma->data的地址上加上长度和HWLOC_SHMEM_MALLOC_ALIGN两个参数的差值，并取反)，最后返回当前内存块的地址。

tma_get_length_malloc函数的作用是从tma_shmem结构体中获取指定长度的内存，并返回该内存的地址。它的参数包括一个结构体指针tma和一个长度参数length，分别表示待分配内存和该内存的长度。函数的具体实现是先将tma->data指向的内存块的地址加上长度和HWLOC_SHMEM_MALLOC_ALIGN两个参数的差值，并取反，最后将该地址指向的内存块的长度加1并从tma->data的地址上取反，得到该内存的起始地址。


```cpp
tma_shmem_malloc(struct hwloc_tma * tma,
		 size_t length)
{
  void *current = tma->data;
  tma->data = (char*)tma->data  + ((length + HWLOC_SHMEM_MALLOC_ALIGN - 1) & ~(HWLOC_SHMEM_MALLOC_ALIGN - 1));
  return current;

}

static void *
tma_get_length_malloc(struct hwloc_tma * tma,
		      size_t length)
{
  size_t *tma_length = tma->data;
  *tma_length += (length + HWLOC_SHMEM_MALLOC_ALIGN - 1) & ~(HWLOC_SHMEM_MALLOC_ALIGN - 1);
  return malloc(length);

}

```

这段代码定义了一个名为 `hwloc_shmem_topology_get_length` 的函数，它属于 `hwloc_topology_functions` 类型。函数接收三个参数：

1. `topology`：一个 `hwloc_topology_t` 类型的整数，表示要获取其内存布局中所有设备的地址空间的顶部位置的 `hwloc_topology` 结构体。
2. `lengthp`：一个 `size_t` 类型的整数，用于存储获取到的内存布局长度。
3. `flags`：一个 `unsigned long` 类型的整数，用于指定获取内存布局时使用的选项。可选的选项包括 `HWL_TOPOLOGY_FULL_PAGE` 和 `HWL_TOPOLOGY_GROW_AREA`。

函数首先定义了一个名为 `new` 的 `hwloc_topology_t` 类型的整数，它将在函数内部用于复制 `topology` 结构体。接着定义了一个名为 `tma` 的 `struct hwloc_tma` 类型的整数，用于存储分配内存时使用的数据结构。

函数的核心部分开始执行如下步骤：

1. 如果 `flags` 中包含了 `HWL_TOPOLOGY_FULL_PAGE`，那么直接调用 `hwloc_getpagesize` 函数，将其结果存储在 `lengthp` 指向的内存空间位置，并返回 0。
2. 如果 `flags` 中包含了 `HWL_TOPOLOGY_GROW_AREA`，那么执行以下操作：

a. 计算要分配的内存空间大小。这个大小是 `pagesize` 减去 `1`，因为需要将 `pagesize` 页面大小进行 round-up，以得到一个完全有效的页面大小。

b. 调用 `hwloc__topology_dup` 函数，将 `new` 和 `topology` 结构体作为参数传入，并将 `tma` 结构体作为 `hwloc_topology_t` 类型的指针传入。如果出错，返回 `EINVAL`。

c. 调用 `hwloc_topology_destroy` 函数，传入 `new` 结构体，将其从内存中删除。

d. 计算新的内存布局长度，这个长度是 `sizeof(struct hwloc_shmem_header)`（定义在 `hwloc_shmem_topology_get_length.c` 中）和要分配的内存空间大小的结果，减去 `1`（因为要扣除页面大小）。

e. 返回 0。

函数的实现旨在帮助用户获取指定 `hwloc_topology_t` 结构体中设备的内存布局的顶部位置，并返回该位置的内存布局长度。


```cpp
int
hwloc_shmem_topology_get_length(hwloc_topology_t topology,
				size_t *lengthp,
				unsigned long flags)
{
  hwloc_topology_t new;
  struct hwloc_tma tma;
  size_t length = 0;
  unsigned long pagesize = hwloc_getpagesize(); /* round-up to full page for mmap() */
  int err;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  tma.malloc = tma_get_length_malloc;
  tma.dontfree = 0;
  tma.data = &length;

  err = hwloc__topology_dup(&new, topology, &tma);
  if (err < 0)
    return err;
  hwloc_topology_destroy(new);

  *lengthp = (sizeof(struct hwloc_shmem_header) + length + pagesize - 1) & ~(pagesize - 1);
  return 0;
}

```

It looks like you are asking for a C function that performs a number of operations on a shared memory (SHMEM) header, including copying a header to a new memory location, writing to a file, and fragmentation. Here is a possible implementation:
```cpp
#include <stdio.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/stat.h>
#include <hwloc.h>
#include <hulbert.h>

#define MAX_FILES 1024

static hwloc_topology_t topology;
static hwloc_handle_t shm_handle;
static hwloc_memattrs_t memattrs;
static hwloc_component_t components;
static int mem_fd;
static int file_fd;
static int copy_status;
static struct file_table file_table[MAX_FILES];
static int num_files;

static void copy_header(void)
{
 static char header[MAX_FILES][MAX_HEADER_SIZE];
 static int num_header_words = 0;

 for (int i = 0; i < num_files; i++) {
   static char *file_header = file_table[i];
   static int file_offset = i * file_size;
   static int header_offset = file_header[i * header_size];
   static int header_size = MAX_HEADER_SIZE - header_offset;

   if (copy_status == 0) {
     memcpy(file_header + header_offset, header, header_size);
     num_header_words += header_size;
   } else {
     int word_offset = header_offset + header_size;
     memcpy(file_header + header_offset, header + word_offset, MAX_HEADER_SIZE);
     num_header_words += MAX_HEADER_SIZE - header_offset;
   }

   file_table[i] = file_header;
   file_size[i] = file_offset + header_size;
   copy_status = 1;
 }
}

static void write_file(int file_fd, const char *filename, const char *contents)
{
 static int i;
 static int file_offset = 0;
 static int num_words = 0;
 static char *file_contents = (char *)文件的临时文件号和新文件号，每个文件号占一个字节的整数部分；

 for (int i = 0; i < num_files; i++) {
   static char *file_header = file_table[i];
   static int file_offset = i * file_size;
   static int num_words = 0;

   if (copy_status == 0) {
     file_contents[i] = '\0';
     memcpy(&file_header + file_offset, contents, MAX_HEADER_SIZE);
     num_words += MAX_HEADER_SIZE - file_offset;
   } else {
     int word_offset = file_offset + header_size;
     memcpy(&file_header + file_offset, header + word_offset, MAX_HEADER_SIZE);
     num_words += MAX_HEADER_SIZE - header_offset;
     file_contents[i] = '\0';
   }

   file_table[i] = file_header;
   file_size[i] = file_offset + num_words;
   copy_status = 0;
 }
}

static void read_file(int file_fd, const char *filename, char *buffer, int buffer_size)
{
 static int i;
 static int num_words = 0;
 static char *file_header = (char *)文件的临时文件号，每个文件号占一个字节的整数部分；
 static int header_offset = 0;
 static int num_words = 0;

 for (int i = 0; i < num_files; i++) {
   static char *file_header = file_table[i];
   static int file_offset = i * file_size;
   static int num_words = 0;

   if (copy_status == 0) {
     file_header[0] = '\0';
     memset(&file_header + file_offset, 0, MAX_HEADER_SIZE - file_offset);
     num_words += MAX_HEADER_SIZE - file_offset;
   } else {
     int word_offset = file_offset + header_size;
     memcpy(&file_header + file_offset, header + word_offset, MAX_HEADER_SIZE - header_offset);
     num_words += MAX_HEADER_SIZE - header_offset;
   }
 }

 memcpy(buffer, file_header, header_offset + num_words);
 file_table[i] = file_header;
 file_size[i] = header_offset + num_words;
 copy_status = 1;
}

static void make_memory_map(int file_fd, const char *filename)
{
 static int i;
 static int num_words = 0;
 static char *file_header = (char *)文件的临时文件号，每个文件号占一个字节的整数部分；
 static int header_offset = 0;
 static int num_words = 0;

 for (int i = 0; i < num_files; i++) {
   static char *file_header = file_table[i];
   static int file_offset = i * file_size;
   static int num_words = 0;

   if (copy_status == 0) {
     file_header[0] = '\0';
     memset(&file_header + file_offset, 0, MAX_HEADER_SIZE - file_offset);
     num_words += MAX_HEADER_SIZE - file_offset;
   } else {
     int word_offset = file_offset + header_size;
     memcpy(&file_header + file_offset, header + word_offset, MAX_HEADER_SIZE - header_offset);
     num_words += MAX_HEADER_SIZE - header_offset;



```
int
hwloc_shmem_topology_write(hwloc_topology_t topology,
			   int fd, hwloc_uint64_t fileoffset,
			   void *mmap_address, size_t length,
			   unsigned long flags)
{
  hwloc_topology_t new;
  struct hwloc_tma tma;
  struct hwloc_shmem_header header;
  void *mmap_res;
  int err;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  /* refresh old topology distances so that we don't uselessly duplicate invalid distances
   * without being able to free() them.
   */
  hwloc_internal_distances_refresh(topology);
  hwloc_internal_memattrs_refresh(topology);

  header.header_version = HWLOC_SHMEM_HEADER_VERSION;
  header.header_length = sizeof(header);
  header.mmap_address = (uintptr_t) mmap_address;
  header.mmap_length = length;

  err = lseek(fd, fileoffset, SEEK_SET);
  if (err < 0)
    return -1;

  err = write(fd, &header, sizeof(header));
  if (err != sizeof(header))
    return -1;

  err = ftruncate(fd, fileoffset + length);
  if (err < 0)
    return -1;

  mmap_res = mmap(mmap_address, length, PROT_READ|PROT_WRITE, MAP_SHARED, fd, fileoffset);
  if (mmap_res == MAP_FAILED)
    return -1;
  if (mmap_res != mmap_address) {
    munmap(mmap_res, length);
    errno = EBUSY;
    return -1;
  }

  tma.malloc = tma_shmem_malloc;
  tma.dontfree = 1;
  tma.data = (char *)mmap_res + sizeof(header);
  err = hwloc__topology_dup(&new, topology, &tma);
  if (err < 0)
    return err;
  assert((char*)new == (char*)mmap_address + sizeof(header));

  assert((char *)mmap_res <= (char *)mmap_address + length);

  /* now refresh the new distances/memattrs so that adopters can use them without refreshing the R/O shmem mapping */
  hwloc_internal_distances_refresh(new);
  hwloc_internal_memattrs_refresh(topology);

  /* topology is saved, release resources now */
  munmap(mmap_address, length);
  hwloc_components_fini();

  return 0;
}

```cpp

This is a function definition for `hwloc_components_init()`. It appears to be a part of a program that manages hardware-accelerated localhost (hwloc) components, and the function initializes the components and sets up a binding hook system for those components.

The function takes a single argument, `self`, which is the component instance to initialize. The function first checks if the component instance is already initialized and, if not, creates a new instance of the component.

The function then creates a new object for the topology object, which is the same as the old object but with some additional information. The new object is given the topology type and a pointer to a `malloc` function to dynamically allocate memory for the object.

The function also creates new objects for the `support` structure, which includes functions for discovering, mapping, and binding to hardware components. These objects are given the appropriate pointers for the old object.

Finally, the function sets up the binding hook system by calling the `hwloc_set_binding_hooks` function, passing in the instance of the component. This function is expected to be defined by the `hwloc_component_binding_hooks` function, which is passed to `hwloc_set_binding_hooks` as an argument.


```
int
hwloc_shmem_topology_adopt(hwloc_topology_t *topologyp,
			   int fd, hwloc_uint64_t fileoffset,
			   void *mmap_address, size_t length,
			   unsigned long flags)
{
  hwloc_topology_t new, old;
  struct hwloc_shmem_header header;
  void *mmap_res;
  int err;

  if (flags) {
    errno = EINVAL;
    return -1;
  }

  err = lseek(fd, fileoffset, SEEK_SET);
  if (err < 0)
    return -1;

  err = read(fd, &header, sizeof(header));
  if (err != sizeof(header))
    return -1;

  if (header.header_version != HWLOC_SHMEM_HEADER_VERSION
      || header.header_length != sizeof(header)
      || header.mmap_address != (uintptr_t) mmap_address
      || header.mmap_length != length) {
    errno = EINVAL;
    return -1;
  }

  mmap_res = mmap(mmap_address, length, PROT_READ, MAP_SHARED, fd, fileoffset);
  if (mmap_res == MAP_FAILED)
    return -1;
  if (mmap_res != mmap_address) {
    errno = EBUSY;
    goto out_with_mmap;
  }

  old = (hwloc_topology_t)((char*)mmap_address + sizeof(header));
  if (hwloc_topology_abi_check(old) < 0) {
    errno = EINVAL;
    goto out_with_mmap;
  }

  /* enforced by dup() inside shmem_topology_write() */
  assert(old->is_loaded);
  assert(old->backends == NULL);
  assert(old->get_pci_busid_cpuset_backend == NULL);

  hwloc_components_init();

  /* duplicate the topology object so that we ca change use local binding_hooks
   * (those are likely not mapped at the same location in both processes).
   */
  new = malloc(sizeof(struct hwloc_topology));
  if (!new)
    goto out_with_components;
  memcpy(new, old, sizeof(*old));
  new->tma = NULL;
  new->adopted_shmem_addr = mmap_address;
  new->adopted_shmem_length = length;
  new->topology_abi = HWLOC_TOPOLOGY_ABI;
  /* setting binding hooks will touch support arrays, so duplicate them too.
   * could avoid that by requesting a R/W mmap
   */
  new->support.discovery = malloc(sizeof(*new->support.discovery));
  new->support.cpubind = malloc(sizeof(*new->support.cpubind));
  new->support.membind = malloc(sizeof(*new->support.membind));
  new->support.misc = malloc(sizeof(*new->support.misc));
  if (!new->support.discovery || !new->support.cpubind || !new->support.membind || !new->support.misc)
    goto out_with_support;
  memcpy(new->support.discovery, old->support.discovery, sizeof(*new->support.discovery));
  memcpy(new->support.cpubind, old->support.cpubind, sizeof(*new->support.cpubind));
  memcpy(new->support.membind, old->support.membind, sizeof(*new->support.membind));
  memcpy(new->support.misc, old->support.misc, sizeof(*new->support.misc));
  hwloc_set_binding_hooks(new);
  /* clear userdata callbacks pointing to the writer process' functions */
  new->userdata_export_cb = NULL;
  new->userdata_import_cb = NULL;

```cpp

这段代码是一个C函数，名为`hwloc_remove_debug`。它定义了一个名为`hwloc_debug`的哈希表，用于存储`HWLOC_DEBUG_CHECK`环境变量。如果`HWLOC_DEBUG_CHECK`存在，则执行`hwloc_topology_check`函数。如果`HWLOC_DEBUG_CHECK`不存在，则执行一些其他的操作，包括：

1. 输出调试信息，然后创建一个新的`hwloc_topology_t`结构体，将其作为参数传递给`hwloc_topology_check`函数。
2. 创建一个新的`hwloc_topology_t`结构体，将其作为参数传递给`hwloc_topology_create`函数，并将它存储在`topologyp`指针中，并返回0。
3. 在`out_with_support`函数中，释放新的`hwloc_topology_t`结构体中与发现相关的资源，包括与CPU绑定的`discovery`，内存绑定的`cpubind`，以及与调试相关的`misc`。
4. 在`out_with_components`函数中，调用`hwloc_components_fini`函数来完成组件初始化。
5. 在`out_with_mmap`函数中，使用`munmap`函数释放内存映射，并返回-1，表示操作失败。


```
#ifndef HWLOC_DEBUG
  if (getenv("HWLOC_DEBUG_CHECK"))
#endif
    hwloc_topology_check(new);

  *topologyp = new;
  return 0;

 out_with_support:
  free(new->support.discovery);
  free(new->support.cpubind);
  free(new->support.membind);
  free(new->support.misc);
  free(new);
 out_with_components:
  hwloc_components_fini();
 out_with_mmap:
  munmap(mmap_res, length);
  return -1;
}

```cpp

这段代码定义了一个名为 `hwloc__topology_disadopt` 的函数，属于 `hwloc_topology_t` 结构体类型的辅助函数。

函数的主要作用是释放 `topology` 指向的内存，其中包括已挂载的共享内存映射、对等指针、绑定内存和一些辅助函数。

具体来说，函数在函数开始时先调用 `hwloc_components_fini`，然后释放 `topology->adopted_shmem_addr` 和 `topology->adopted_shmem_length` 所指向的内存区域，接着释放已挂载的共享内存映射，然后释放支持结构体中的所有函数指针，最后释放一些辅助函数，最后释放整个 `topology` 结构体。

需要注意的是，在 `hwloc_topology_disadopt` 函数中，使用了 `munmap` 函数来释放映射的内存，而在 `hwloc_topology_win_sys` 函数中，使用了 `system_class_desc` 函数获取了当前系统的类描述符，进而获取到了 `_HWLOC_WIN_SYS` 类的成员函数指针。


```
void
hwloc__topology_disadopt(hwloc_topology_t topology)
{
  hwloc_components_fini();
  munmap(topology->adopted_shmem_addr, topology->adopted_shmem_length);
  free(topology->support.discovery);
  free(topology->support.cpubind);
  free(topology->support.membind);
  free(topology->support.misc);
  free(topology);
}

#else /* HWLOC_WIN_SYS */

int
```cpp

这两函数函数是用于从硬件抽象层(hwloc)获取内存映射信息，然后将该信息与给定的topology标志一起写回到内存映射中。

函数`hwloc_shmem_topology_get_length`接受一个hwloc topology结构体，一个大小为`size_t`的指针和一个未定义的`unsigned long`标志，然后返回topology的长度，单位是内存单元数量。如果函数无法成功执行，返回-1。

函数`hwloc_shmem_topology_write`接受一个已经定义的hwloc topology结构体，一个文件描述符(int)，一个未定义的`unsigned long`标志，一个大小为`size_t`的内存地址和一个未定义的`size_t`长度的内存映射，然后将topology和写入的内存映射写回到内存映射中。如果函数无法成功执行，返回-1。


```
hwloc_shmem_topology_get_length(hwloc_topology_t topology __hwloc_attribute_unused,
				size_t *lengthp __hwloc_attribute_unused,
				unsigned long flags __hwloc_attribute_unused)
{
  errno = ENOSYS;
  return -1;
}

int
hwloc_shmem_topology_write(hwloc_topology_t topology __hwloc_attribute_unused,
			   int fd __hwloc_attribute_unused, hwloc_uint64_t fileoffset __hwloc_attribute_unused,
			   void *mmap_address __hwloc_attribute_unused, size_t length __hwloc_attribute_unused,
			   unsigned long flags __hwloc_attribute_unused)
{
  errno = ENOSYS;
  return -1;
}

```cpp



这两函数是用于将topology结构体中的数据拷贝到内存中。

函数hwloc_topology_adopt接收一个topology结构体指针，一个fd文件描述符，一个fileoffset文件偏移量，一个mmap_address内存映射地址，和一个length长度。函数中输出一个errno值，表示操作是否成功。

函数hwloc_topology_disadopt则是一个剩余的函数，没有使用它，因此无法从代码中得知其作用。


```
int
hwloc_shmem_topology_adopt(hwloc_topology_t *topologyp __hwloc_attribute_unused,
			   int fd __hwloc_attribute_unused, hwloc_uint64_t fileoffset __hwloc_attribute_unused,
			   void *mmap_address __hwloc_attribute_unused, size_t length __hwloc_attribute_unused,
			   unsigned long flags __hwloc_attribute_unused)
{
  errno = ENOSYS;
  return -1;
}

void
hwloc__topology_disadopt(hwloc_topology_t topology __hwloc_attribute_unused)
{
}

```cpp

这段代码是一个 preprocess 指令，它会在源代码文件被编译之前执行。它的作用是检查是否定义了名为 "HWLOC_WIN_SYS" 的符号，如果是，那么编译器会编译包含该符号的代码行。如果没有定义该符号，那么编译器不会编译该代码行，但也不会输出任何错误。

简单来说，这段代码的作用是防止源代码文件在编译时出现未定义的符号，从而保证程序在编译时正确无误。


```
#endif /* HWLOC_WIN_SYS */

```cpp

# `src/3rdparty/hwloc/src/static-components.h`

这段代码定义了一些外部结构体常量，它们都是hwloc_component类型的成员变量。

具体来说，这段代码定义了以下几个结构体常量：

- hwloc_noos_component：表示不使用libxml2的组件。
- hwloc_xml_component：表示使用libxml2的组件。
- hwloc_synthetic_component：表示使用libxml2，但是不要使用libxml2的组件。
- hwloc_xml_nolibxml_component：表示使用libxml2，但是不要使用libxml2的组件，同时不要使用NOLIBXML库。
- hwloc_windows_component：表示Windows组件。
- hwloc_x86_component：表示x86架构组件。

另外，还定义了一个名为hwloc_static_components的数组，它包含上述所有常量。

总的来说，这段代码定义了一些hwloc_component类型的成员变量，用于表示组件。其中，hwloc_noos_component表示不使用libxml2的组件，hwloc_xml_component表示使用libxml2的组件，hwloc_synthetic_component表示使用libxml2，但是不要使用libxml2的组件，hwloc_xml_nolibxml_component表示使用libxml2，但是不要使用libxml2的组件，hwloc_windows_component表示Windows组件，hwloc_x86_component表示x86架构组件。


```
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_noos_component;
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_xml_component;
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_synthetic_component;
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_xml_nolibxml_component;
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_windows_component;
HWLOC_DECLSPEC extern const struct hwloc_component hwloc_x86_component;
static const struct hwloc_component * hwloc_static_components[] = {
  &hwloc_noos_component,
  &hwloc_xml_component,
  &hwloc_synthetic_component,
  &hwloc_xml_nolibxml_component,
  &hwloc_windows_component,
  &hwloc_x86_component,
  NULL
};

```cpp

# `src/3rdparty/hwloc/src/topology-noos.c`

This is a function definition for a backend named "hwloc" that uses the hardware location information provided by the HWLOC layer.

This backend uses the underlying OS to enforce topology, but it also allows to force use of this backend when debugging with !thissystem.

It first checks if the root process has support for the hardware location information, and if not, it falls back to using the HWLOC layer. If the root process has support for the hardware location information, it then initializes the machine memory and sets the support for discovery to true.

It also creates a basic topology structure and sets the support for the machine memory to the local memory size, if it is available.

Finally, it sets the uname information for the machine, but this is only done if the machine has already been configured.

It is important to note that the code you provided is not complete and may not work as expected, because it is not the complete implementation of the HWLOC layer, and it is not sure if the code you provided is an add-on of the HWLOC layer or if it is a part of it.


```
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2019 Inria.  All rights reserved.
 * Copyright © 2009-2012, 2020 Université Bordeaux
 * Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"

static int
hwloc_look_noos(struct hwloc_backend *backend, struct hwloc_disc_status *dstatus)
{
  /*
   * This backend uses the underlying OS.
   * However we don't enforce topology->is_thissystem so that
   * we may still force use this backend when debugging with !thissystem.
   */

  struct hwloc_topology *topology = backend->topology;
  int64_t memsize;

  assert(dstatus->phase == HWLOC_DISC_PHASE_CPU);

  if (!topology->levels[0][0]->cpuset) {
    int nbprocs;
    /* Nobody (even the x86 backend) created objects yet, setup basic objects */

    nbprocs = hwloc_fallback_nbprocessors(0);
    if (nbprocs >= 1)
      topology->support.discovery->pu = 1;
    else
      nbprocs = 1;
    hwloc_alloc_root_sets(topology->levels[0][0]);
    hwloc_setup_pu_level(topology, nbprocs);
  }

  memsize = hwloc_fallback_memsize();
  if (memsize > 0)
    topology->machine_memory.local_memory = memsize;;

  hwloc_add_uname_info(topology, NULL);
  return 0;
}

```cpp

这段代码定义了一个名为 `hwloc_noos_component_instantiate` 的函数，属于 `hwloc_topology` 和 `hwloc_disc_component` 结构体的成员。

该函数接受一个 `topology` 结构体，一个 `component` 结构体，以及两个 `excluded_phases` 标志。函数内部使用 `__hwloc_attribute_unused` 修饰的参数，保留其不被使用，但不将其作为函数实参或函数返回值。

函数的作用是 instantiate（实例化）一个 `hwloc_backend` 结构体，将其存储到 `backend` 变量中。`hwloc_look_noos` 函数用于设置 `backend` 的 `discover` 成员的值，使得 `hwloc_noos` 启动时尝试使用硬件设备（如网络适配器）。

函数返回一个指向 `hwloc_backend` 结构体的指针，如果没有错误则返回 NULL。


```
static struct hwloc_backend *
hwloc_noos_component_instantiate(struct hwloc_topology *topology,
				 struct hwloc_disc_component *component,
				 unsigned excluded_phases __hwloc_attribute_unused,
				 const void *_data1 __hwloc_attribute_unused,
				 const void *_data2 __hwloc_attribute_unused,
				 const void *_data3 __hwloc_attribute_unused)
{
  struct hwloc_backend *backend;
  backend = hwloc_backend_alloc(topology, component);
  if (!backend)
    return NULL;
  backend->discover = hwloc_look_noos;
  return backend;
}

```cpp

这两行代码定义了一个静态结构体 `hwloc_disc_component`，其中包括以下字段：

1. `"no_os"`：这是一个字符串，表示该组件不是基于任何操作系统（即“no_os”）。
2. `HWLOC_DISC_PHASE_CPU`：这是一个表示CPU序位的硬件抽象，用于指定优先级。
3. `HWLOC_DISC_PHASE_GLOBAL`：这是一个表示全局范围的硬件抽象，用于指定优先级。
4. `hwloc_noos_component_instantiate`：这是一个函数指针，指向调用 `hwloc_noos_component_ instantiate` 的函数。
5. `40`：这是一个表示低于本地操作系统组件的最低优先级的值。
6. `1`：这是一个表示高于全局范围硬件抽象的最低优先级的值。
7. `NULL`：这是一个指向 NULL 的指针。

接下来的两行代码定义了一个常量结构体 `hwloc_noos_component`，其中包括：

1. `HWLOC_COMPONENT_ABI`：这是一个常量，表示该组件的硬件抽象。
2. `NULL`：这是一个指向 NULL 的指针。

这两行代码用于定义 `hwloc_disc_component`，它是 `hwloc_noos_component` 的实例化。通过这两行代码，可以调用 `hwloc_noos_component_instantiate` 函数来实例化 `hwloc_noos_component`，并将其硬件抽象设置为基于本地操作系统组件，优先级为 40，低于全局范围硬件抽象的最低优先级。


```
static struct hwloc_disc_component hwloc_noos_disc_component = {
  "no_os",
  HWLOC_DISC_PHASE_CPU,
  HWLOC_DISC_PHASE_GLOBAL,
  hwloc_noos_component_instantiate,
  40, /* lower than native OS component, higher than globals */
  1,
  NULL
};

const struct hwloc_component hwloc_noos_component = {
  HWLOC_COMPONENT_ABI,
  NULL, NULL,
  HWLOC_COMPONENT_TYPE_DISC,
  0,
  &hwloc_noos_disc_component
};

```cpp

# `src/3rdparty/hwloc/src/topology-synthetic.c`

这段代码定义了一个名为"private/config.h"的头文件，它可能是某个程序或库的私有代码。然而，从该代码中我们无法确定它的具体作用。

该头文件可能用于初始化某些配置项，如内存限制、处理器数量限制等。它包含了一系列与"private/autogen/config.h"和"private/debug.h"等相关的函数，这些函数可能会被其他私有库或用户空间程序调用。

然而，我们无法提供更多关于这段代码的上下文信息，因为缺乏足够的信息。


```
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2022 Inria.  All rights reserved.
 * Copyright © 2009-2010 Université Bordeaux
 * Copyright © 2009-2011 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

#include "private/autogen/config.h"
#include "hwloc.h"
#include "private/private.h"
#include "private/misc.h"
#include "private/debug.h"

#include <limits.h>
```cpp

这段代码定义了两个结构体：`hwloc_synthetic_attr_s` 和 `hwloc_synthetic_indexes_s`。这两个结构体用于在 CUDA 2.0 中对硬件设备属性进行缓存和查询。

具体来说，`hwloc_synthetic_attr_s` 结构体用于表示硬件设备属性的信息，包括类型、深度、缓存类型和内存大小等。这些属性在程序运行时可能会频繁被读取和写入，因此可以被缓存到硬件设备上，以避免每次都去访问操作系统内存。

`hwloc_synthetic_indexes_s` 结构体则用于表示在缓存中查找硬件设备属性时使用的索引。这个结构体包括一个指向字符串的指针、字符串的长度、一个指向数组的指针，以及一个指向下一个对象的指针。在填充topology（即创建硬件设备层次结构）时，这个结构体可以用来记录已经创建的物体层和物体，从而使得在每次查询硬件设备属性时，可以直接从已经创建好的物体层中查询，避免了每次查询都需要创建一个新的物体层。

在头文件中，首先引入了 `assert.h` 和 `strings.h`，分别用于断言和字符串操作。接着定义了 `hwloc_synthetic_attr_s` 和 `hwloc_synthetic_indexes_s` 结构体，分别用于表示硬件设备属性和查询硬件设备属性的信息。


```
#include <assert.h>
#ifdef HAVE_STRINGS_H
#include <strings.h>
#endif

struct hwloc_synthetic_attr_s {
  hwloc_obj_type_t type;
  unsigned depth; /* For caches/groups */
  hwloc_obj_cache_type_t cachetype; /* For caches */
  hwloc_uint64_t memorysize; /* For caches/memory */
};

struct hwloc_synthetic_indexes_s {
  /* the indexes= attribute before parsing */
  const char *string;
  unsigned long string_length;
  /* the array of explicit indexes after parsing */
  unsigned *array;

  /* used while filling the topology */
  unsigned next; /* id of the next object for that level */
};

```cpp

这是一段定义了硬件虚拟定位（hwloc）合成级别数据的结构体的代码。这个结构体是用来表示在硬件虚拟定位中，各个硬件资源（如CPU、GPU、内存等）的合成级别的数据结构。

这个结构体包含以下成员：

1. `arity`：合成级别数据的arity，即数据可以代表的硬件资源数量。例如，如果硬件资源有4种，则`arity`的值为4。
2. `totalwidth`：合成级别数据的总宽度，即数据中各个硬件资源的总宽度。
3. `attr`：硬件虚拟定位的属性，用于指定该结构体中硬件资源的设置选项。
4. `indexes`：硬件虚拟定位的索引，用于指定该结构体中硬件资源的设置选项。
5. `attached`：用于表示该结构体中硬件资源的状态，包括硬件资源的已绑定和未绑定状态。
6. `next`：指向下一个与该结构体中硬件资源相关的`attached`结构体的指针。

这个结构体是硬件虚拟定位合成级别数据结构的一部分，可以用于在软件中创建硬件资源对象，并返回与该对象相关的信息。


```
struct hwloc_synthetic_level_data_s {
  unsigned arity;
  unsigned long totalwidth;

  struct hwloc_synthetic_attr_s attr;
  struct hwloc_synthetic_indexes_s indexes;

  struct hwloc_synthetic_attached_s {
    struct hwloc_synthetic_attr_s attr;

    struct hwloc_synthetic_attached_s *next;
  } *attached;
};

struct hwloc_synthetic_backend_data_s {
  /* synthetic backend parameters */
  char *string;

  unsigned long numa_attached_nr;
  struct hwloc_synthetic_indexes_s numa_attached_indexes;

```cpp

This is a C implementation of the術語"interleaving"（插值）和"loops"（循環）。這是通過將一個無窮大的整數向量（nbs）通過一個固定的步長（minstep）插值到一個向量中來實現的。

該算法通過nbs和total來定義了步長和總共需要插入的步數。首先，該算法會計算出一個loops數組，其中每個元素代表一個步長和相應的nb值。接著，該算法會遍歷loops數組，計算出相應的步數和nb值，最終生成一個插值後的整數向量。

在该算法中，通過設置步長為total / nbs，保證了當 nbs 是 total 的整數倍時，插值後的所有值都落在總共中，而當 nbs 不是 total 的整數倍時，插值後的所有值都不会超出總共。此外，該算法還能處理 total 和 nbs 都是 0 的情況，分別通過免費為 non-zero 並將步長設置為 0 來實現。


```
#define HWLOC_SYNTHETIC_MAX_DEPTH 128
  struct hwloc_synthetic_level_data_s level[HWLOC_SYNTHETIC_MAX_DEPTH];
};

struct hwloc_synthetic_intlv_loop_s {
  unsigned step;
  unsigned nb;
  unsigned level_depth;
};

static void
hwloc_synthetic_process_indexes(struct hwloc_synthetic_backend_data_s *data,
				struct hwloc_synthetic_indexes_s *indexes,
				unsigned long total,
				int verbose)
{
  const char *attr = indexes->string;
  unsigned long length = indexes->string_length;
  unsigned *array = NULL;
  size_t i;

  if (!attr)
    return;

  array = calloc(total, sizeof(*array));
  if (!array) {
    if (verbose)
      fprintf(stderr, "Failed to allocate synthetic index array of size %lu\n", total);
    goto out;
  }

  i = strspn(attr, "0123456789,");
  if (i == length) {
    /* explicit array of indexes */

    for(i=0; i<total; i++) {
      const char *next;
      unsigned idx = strtoul(attr, (char **) &next, 10);
      if (next == attr) {
	if (verbose)
	  fprintf(stderr, "Failed to read synthetic index #%lu at '%s'\n", (unsigned long) i, attr);
	goto out_with_array;
      }

      array[i] = idx;
      if (i != total-1) {
	if (*next != ',') {
	  if (verbose)
	    fprintf(stderr, "Missing comma after synthetic index #%lu at '%s'\n", (unsigned long) i, attr);
	  goto out_with_array;
	}
	attr = next+1;
      } else {
	attr = next;
      }
    }
    indexes->array = array;

  } else {
    /* interleaving */
    unsigned nr_loops = 1, cur_loop;
    unsigned minstep = total;
    unsigned long nbs = 1;
    unsigned j, mul;
    const char *tmp;
    struct hwloc_synthetic_intlv_loop_s *loops;

    tmp = attr;
    while (tmp) {
      tmp = strchr(tmp, ':');
      if (!tmp || tmp >= attr+length)
	break;
      nr_loops++;
      tmp++;
    }

    /* nr_loops colon-separated fields, but we may need one more at the end */
    loops = malloc((nr_loops+1) * sizeof(*loops));
    if (!loops)
      goto out_with_array;

    if (*attr >= '0' && *attr <= '9') {
      /* interleaving as x*y:z*t:... */
      unsigned step, nb;

      tmp = attr;
      cur_loop = 0;
      while (tmp) {
	char *tmp2, *tmp3;
	step = (unsigned) strtol(tmp, &tmp2, 0);
	if (tmp2 == tmp || *tmp2 != '*') {
	  if (verbose)
	    fprintf(stderr, "Failed to read synthetic index interleaving loop '%s' without number before '*'\n", tmp);
	  free(loops);
	  goto out_with_array;
	}
	if (!step) {
	  if (verbose)
	    fprintf(stderr, "Invalid interleaving loop with step 0 at '%s'\n", tmp);
	  free(loops);
	  goto out_with_array;
	}
	tmp2++;
	nb = (unsigned) strtol(tmp2, &tmp3, 0);
	if (tmp3 == tmp2 || (*tmp3 && *tmp3 != ':' && *tmp3 != ')' && *tmp3 != ' ')) {
	  if (verbose)
	    fprintf(stderr, "Failed to read synthetic index interleaving loop '%s' without number between '*' and ':'\n", tmp);
	  free(loops);
	  goto out_with_array;
	}
	if (!nb) {
	  if (verbose)
	    fprintf(stderr, "Invalid interleaving loop with number 0 at '%s'\n", tmp2);
	  free(loops);
	  goto out_with_array;
	}
	loops[cur_loop].step = step;
	loops[cur_loop].nb = nb;
	if (step < minstep)
	  minstep = step;
	nbs *= nb;
	cur_loop++;
	if (*tmp3 == ')' || *tmp3 == ' ')
	  break;
	tmp = (const char*) (tmp3+1);
      }

    } else {
      /* interleaving as type1:type2:... */
      hwloc_obj_type_t type;
      union hwloc_obj_attr_u attrs;
      int err;

      /* find level depths for each interleaving loop */
      tmp = attr;
      cur_loop = 0;
      while (tmp) {
	err = hwloc_type_sscanf(tmp, &type, &attrs, sizeof(attrs));
	if (err < 0) {
	  if (verbose)
	    fprintf(stderr, "Failed to read synthetic index interleaving loop type '%s'\n", tmp);
	  free(loops);
	  goto out_with_array;
	}
	if (type == HWLOC_OBJ_MISC || type == HWLOC_OBJ_BRIDGE || type == HWLOC_OBJ_PCI_DEVICE || type == HWLOC_OBJ_OS_DEVICE) {
	  if (verbose)
	    fprintf(stderr, "Misc object type disallowed in synthetic index interleaving loop type '%s'\n", tmp);
	  free(loops);
	  goto out_with_array;
	}
	for(i=0; ; i++) {
	  if (!data->level[i].arity) {
	    loops[cur_loop].level_depth = (unsigned)-1;
	    break;
	  }
	  if (type != data->level[i].attr.type)
	    continue;
	  if (type == HWLOC_OBJ_GROUP
	      && attrs.group.depth != (unsigned) -1
	      && attrs.group.depth != data->level[i].attr.depth)
	    continue;
	  loops[cur_loop].level_depth = (unsigned)i;
	  break;
	}
	if (loops[cur_loop].level_depth == (unsigned)-1) {
	  if (verbose)
	    fprintf(stderr, "Failed to find level for synthetic index interleaving loop type '%s'\n",
		    tmp);
	  free(loops);
	  goto out_with_array;
	}
	tmp = strchr(tmp, ':');
	if (!tmp || tmp > attr+length)
	  break;
	tmp++;
	cur_loop++;
      }

      /* compute actual loop step/nb */
      for(cur_loop=0; cur_loop<nr_loops; cur_loop++) {
	unsigned mydepth = loops[cur_loop].level_depth;
	unsigned prevdepth = 0;
	unsigned step, nb;
	for(i=0; i<nr_loops; i++) {
	  if (loops[i].level_depth == mydepth && i != cur_loop) {
	    if (verbose)
	      fprintf(stderr, "Invalid duplicate interleaving loop type in synthetic index '%s'\n", attr);
	    free(loops);
	    goto out_with_array;
	  }
	  if (loops[i].level_depth < mydepth
	      && loops[i].level_depth > prevdepth)
	    prevdepth = loops[i].level_depth;
	}
	step = total / data->level[mydepth].totalwidth; /* number of objects below us */
	nb = data->level[mydepth].totalwidth / data->level[prevdepth].totalwidth; /* number of us within parent */

	loops[cur_loop].step = step;
	loops[cur_loop].nb = nb;
	assert(nb);
	assert(step);
	if (step < minstep)
	  minstep = step;
	nbs *= nb;
      }
    }
    assert(nbs);

    if (nbs != total) {
      /* one loop of total/nbs steps is missing, add it if it's just the smallest one */
      if (minstep == total/nbs) {
	loops[nr_loops].step = 1;
	loops[nr_loops].nb = total/nbs;
	nr_loops++;
      } else {
	if (verbose)
	  fprintf(stderr, "Invalid index interleaving total width %lu instead of %lu\n", nbs, total);
	free(loops);
	goto out_with_array;
      }
    }

    /* generate the array of indexes */
    mul = 1;
    for(i=0; i<nr_loops; i++) {
      unsigned step = loops[i].step;
      unsigned nb = loops[i].nb;
      for(j=0; j<total; j++)
	array[j] += ((j / step) % nb) * mul;
      mul *= nb;
    }

    free(loops);

    /* check that we have the right values (cannot pass total, cannot give duplicate 0) */
    for(j=0; j<total; j++) {
      if (array[j] >= total) {
	if (verbose)
	  fprintf(stderr, "Invalid index interleaving generates out-of-range index %u\n", array[j]);
	goto out_with_array;
      }
      if (!array[j] && j) {
	if (verbose)
	  fprintf(stderr, "Invalid index interleaving generates duplicate index values\n");
	goto out_with_array;
      }
    }

    indexes->array = array;
  }

  return;

 out_with_array:
  free(array);
 out:
  return;
}

```cpp

这段代码是一个函数，名为`hwloc_synthetic_parse_memory_attr`，它的作用是解析一个字符串属性，并返回对应的内存大小。输入的字符串属性包括"TB"、"TiB"、"GB"和"GiB"，分别表示1TB、1TB、1GB和1GB。函数将输入的字符串属性转换为对应的国际单位制（TB、TB、GB和GB），然后将字符串长度乘以1000得到内存大小，最后将内存大小存储在指针变量`endptr`中，并返回该指针。


```
static hwloc_uint64_t
hwloc_synthetic_parse_memory_attr(const char *attr, const char **endp)
{
  const char *endptr;
  hwloc_uint64_t size;
  size = strtoull(attr, (char **) &endptr, 0);
  if (!hwloc_strncasecmp(endptr, "TB", 2)) {
    size *= 1000ULL*1000ULL*1000ULL*1000ULL;
    endptr += 2;
  } else if (!hwloc_strncasecmp(endptr, "TiB", 3)) {
    size <<= 40;
    endptr += 3;
  } else if (!hwloc_strncasecmp(endptr, "GB", 2)) {
    size *= 1000ULL*1000ULL*1000ULL;
    endptr += 2;
  } else if (!hwloc_strncasecmp(endptr, "GiB", 3)) {
    size <<= 30;
    endptr += 3;
  } else if (!hwloc_strncasecmp(endptr, "MB", 2)) {
    size *= 1000ULL*1000ULL;
    endptr += 2;
  } else if (!hwloc_strncasecmp(endptr, "MiB", 3)) {
    size <<= 20;
    endptr += 3;
  } else if (!hwloc_strncasecmp(endptr, "kB", 2)) {
    size *= 1000ULL;
    endptr += 2;
  } else if (!hwloc_strncasecmp(endptr, "kiB", 3)) {
    size <<= 10;
    endptr += 3;
  }
  *endp = endptr;
  return size;
}

```cpp

This is a C function that parses a synthetic string that may contain a number of undeclared variables. The function takes an optional "attrs" parameter, which is the synthetic string containing the attribute names in the format of "name=value;name=value......", where "name" and "value" are the attribute names and values, respectively.

The function first checks if the synthetic string is well-formed and defines the memorysize of the last attribute value. If the string is not well-formed, an error is thrown.

The function then parses the synthetic string by checking for various attribute types. For example, if the string contains the "size=" parameter, the function will parse the "size" attribute value as a memory size. If the string contains the "memory=" parameter, the function will parse the "memory" attribute value as a memory size. If the string contains the "indexes=" parameter, the function will parse the "indexes" attribute as a string that consists of multiple elements separated by spaces or other specified delimiters.

If the string is not well-formed or if any attributes are missing, an error is thrown. The function also returns an error code if the parse fails.


```
static int
hwloc_synthetic_parse_attrs(const char *attrs, const char **next_posp,
			    struct hwloc_synthetic_attr_s *sattr,
			    struct hwloc_synthetic_indexes_s *sind,
			    int verbose)
{
  hwloc_obj_type_t type = sattr->type;
  const char *next_pos;
  hwloc_uint64_t memorysize = 0;
  const char *index_string = NULL;
  size_t index_string_length = 0;

  next_pos = (const char *) strchr(attrs, ')');
  if (!next_pos) {
    if (verbose)
      fprintf(stderr, "Missing attribute closing bracket in synthetic string doesn't have a number of objects at '%s'\n", attrs);
    errno = EINVAL;
    return -1;
  }

  while (')' != *attrs) {
    int iscache = hwloc__obj_type_is_cache(type);

    if (iscache && !strncmp("size=", attrs, 5)) {
      memorysize = hwloc_synthetic_parse_memory_attr(attrs+5, &attrs);

    } else if (!iscache && !strncmp("memory=", attrs, 7)) {
      memorysize = hwloc_synthetic_parse_memory_attr(attrs+7, &attrs);

    } else if (!strncmp("indexes=", attrs, 8)) {
      index_string = attrs+8;
      attrs += 8;
      index_string_length = strcspn(attrs, " )");
      attrs += index_string_length;

    } else {
      if (verbose)
	fprintf(stderr, "Unknown attribute at '%s'\n", attrs);
      errno = EINVAL;
      return -1;
    }

    if (' ' == *attrs)
      attrs++;
    else if (')' != *attrs) {
      if (verbose)
	fprintf(stderr, "Missing parameter separator at '%s'\n", attrs);
      errno = EINVAL;
      return -1;
    }
  }

  sattr->memorysize = memorysize;

  if (index_string) {
    if (sind->string && verbose)
      fprintf(stderr, "Overwriting duplicate indexes attribute with last occurence\n");
    sind->string = index_string;
    sind->string_length = (unsigned long)index_string_length;
  }

  *next_posp = next_pos+1;
  return 0;
}

```cpp

这段代码定义了一个名为 `hwloc_synthetic_free_levels` 的函数，属于 `hwloc_synthetic_backend_data_s` 结构的子结构。

该函数的主要作用是释放合成器中的层次结构，直到根层达到临界深度（即 `HWLOC_SYNTHETIC_MAX_DEPTH`）。具体的实现步骤如下：

1. 遍历整个层次结构，从根层开始，直到达到指定的深度 `HWLOC_SYNTHETIC_MAX_DEPTH`。
2. 对于每一个子层次结构，先获取其对应的 `struct hwloc_synthetic_level_data_s` 结构，再获取其对应的 `struct hwloc_synthetic_attached_s` 结构，接着遍历 `struct hwloc_synthetic_attached_s` 结构链表。
3. 在遍历过程中，如果发现某个 `struct hwloc_synthetic_attached_s` 结构，则先将其指向的 `struct hwloc_synthetic_level_data_s` 结构解引用，再将指向该 `struct hwloc_synthetic_attached_s` 的 `struct hwloc_synthetic_attached_s` 指针指向其下一个结构。
4. 如果当前层数的临界深度已经达到，或者当前层数为 `0`，则执行以下语句：
	* 释放整个层次结构中所有挂载的同步器。
	* 释放根层中的同步器。
	* 如果当前层数为 `0`，则跳出循环。

该函数的作用是确保在释放整个层次结构时，不会留下任何未释放的同步器，从而确保在释放树形结构时，能够正确地释放所有同步器。


```
/* frees level until arity = 0 */
static void
hwloc_synthetic_free_levels(struct hwloc_synthetic_backend_data_s *data)
{
  unsigned i;
  for(i=0; i<HWLOC_SYNTHETIC_MAX_DEPTH; i++) {
    struct hwloc_synthetic_level_data_s *curlevel = &data->level[i];
    struct hwloc_synthetic_attached_s **pprev = &curlevel->attached;
    while (*pprev) {
      struct hwloc_synthetic_attached_s *cur = *pprev;
      *pprev = cur->next;
      free(cur);
    }
    free(curlevel->indexes.array);
    if (!curlevel->arity)
      break;
  }
  free(data->numa_attached_indexes.array);
}

```cpp

This function appears to be a C language implementation of the hwloc utility, which is used to manipulate hierarchical data structures such as


```
/* Read from description a series of integers describing a symmetrical
   topology and update the hwloc_synthetic_backend_data_s accordingly.  On
   success, return zero.  */
static int
hwloc_backend_synthetic_init(struct hwloc_synthetic_backend_data_s *data,
			     const char *description)
{
  const char *pos, *next_pos;
  unsigned long item, count;
  unsigned i;
  int type_count[HWLOC_OBJ_TYPE_MAX];
  unsigned unset;
  int verbose = 0;
  const char *env = getenv("HWLOC_SYNTHETIC_VERBOSE");
  int err;
  unsigned long totalarity = 1;

  if (env)
    verbose = atoi(env);

  data->numa_attached_nr = 0;
  data->numa_attached_indexes.array = NULL;

  /* default values before we add root attributes */
  data->level[0].totalwidth = 1;
  data->level[0].attr.type = HWLOC_OBJ_MACHINE;
  data->level[0].indexes.string = NULL;
  data->level[0].indexes.array = NULL;
  data->level[0].attr.memorysize = 0;
  data->level[0].attached = NULL;
  type_count[HWLOC_OBJ_MACHINE] = 1;
  if (*description == '(') {
    err = hwloc_synthetic_parse_attrs(description+1, &description, &data->level[0].attr, &data->level[0].indexes, verbose);
    if (err < 0)
      return err;
  }

  data->numa_attached_indexes.string = NULL;
  data->numa_attached_indexes.array = NULL;

  for (pos = description, count = 1; *pos; pos = next_pos) {
    hwloc_obj_type_t type = HWLOC_OBJ_TYPE_NONE;
    union hwloc_obj_attr_u attrs;

    /* initialize parent arity to 0 so that the levels are not infinite */
    data->level[count-1].arity = 0;

    while (*pos == ' ' || *pos == '\n')
      pos++;

    if (!*pos)
      break;

    if (*pos == '[') {
      /* attached */
      struct hwloc_synthetic_attached_s *attached, **pprev;
      char *attr;

      pos++;

      if (hwloc_type_sscanf(pos, &type, &attrs, sizeof(attrs)) < 0) {
	if (verbose)
	  fprintf(stderr, "Synthetic string with unknown attached object type at '%s'\n", pos);
	errno = EINVAL;
	goto error;
      }
      if (type != HWLOC_OBJ_NUMANODE) {
	if (verbose)
	  fprintf(stderr, "Synthetic string with disallowed attached object type at '%s'\n", pos);
	errno = EINVAL;
	goto error;
      }
      data->numa_attached_nr += data->level[count-1].totalwidth;

      attached = malloc(sizeof(*attached));
      if (attached) {
	attached->attr.type = type;
	attached->attr.memorysize = 0;
	/* attached->attr.depth and .cachetype unused */
	attached->next = NULL;
	pprev = &data->level[count-1].attached;
	while (*pprev)
	  pprev = &((*pprev)->next);
	*pprev = attached;
      }

      next_pos = strchr(pos, ']');
      if (!next_pos) {
	if (verbose)
	  fprintf(stderr,"Synthetic string doesn't have a closing `]' after attached object type at '%s'\n", pos);
	errno = EINVAL;
	goto error;
      }

      attr = strchr(pos, '(');
      if (attr && attr < next_pos && attached) {
	const char *dummy;
	err = hwloc_synthetic_parse_attrs(attr+1, &dummy, &attached->attr, &data->numa_attached_indexes, verbose);
	if (err < 0)
	  goto error;
      }

      next_pos++;
      continue;
    }

    /* normal level */

    /* reset defaults */
    data->level[count].indexes.string = NULL;
    data->level[count].indexes.array = NULL;
    data->level[count].attached = NULL;

    if (*pos < '0' || *pos > '9') {
      if (hwloc_type_sscanf(pos, &type, &attrs, sizeof(attrs)) < 0) {
	if (!strncmp(pos, "Tile", 4) || !strncmp(pos, "Module", 6)) {
	  /* possible future types */
	  type = HWLOC_OBJ_GROUP;
	} else {
	  /* FIXME: allow generic "Cache" string? would require to deal with possibly duplicate cache levels */
	  if (verbose)
	    fprintf(stderr, "Synthetic string with unknown object type at '%s'\n", pos);
	  errno = EINVAL;
	  goto error;
	}
      }
      if (type == HWLOC_OBJ_MACHINE || type == HWLOC_OBJ_MISC || type == HWLOC_OBJ_BRIDGE || type == HWLOC_OBJ_PCI_DEVICE || type == HWLOC_OBJ_OS_DEVICE) {
	if (verbose)
	  fprintf(stderr, "Synthetic string with disallowed object type at '%s'\n", pos);
	errno = EINVAL;
	goto error;
      }

      next_pos = strchr(pos, ':');
      if (!next_pos) {
	if (verbose)
	  fprintf(stderr,"Synthetic string doesn't have a `:' after object type at '%s'\n", pos);
	errno = EINVAL;
	goto error;
      }
      pos = next_pos + 1;
    }

    data->level[count].attr.type = type;
    data->level[count].attr.depth = (unsigned) -1;
    data->level[count].attr.cachetype = (hwloc_obj_cache_type_t) -1;
    if (hwloc__obj_type_is_cache(type)) {
      /* these are always initialized */
      data->level[count].attr.depth = attrs.cache.depth;
      data->level[count].attr.cachetype = attrs.cache.type;
    } else if (type == HWLOC_OBJ_GROUP) {
      /* could be -1 but will be set below */
      data->level[count].attr.depth = attrs.group.depth;
    }

    /* number of normal children */
    item = strtoul(pos, (char **)&next_pos, 0);
    if (next_pos == pos) {
      if (verbose)
	fprintf(stderr,"Synthetic string doesn't have a number of objects at '%s'\n", pos);
      errno = EINVAL;
      goto error;
    }
    if (!item) {
      if (verbose)
	fprintf(stderr,"Synthetic string with disallow 0 number of objects at '%s'\n", pos);
      errno = EINVAL;
      goto error;
    }

    totalarity *= item;
    data->level[count].totalwidth = totalarity;
    data->level[count].indexes.string = NULL;
    data->level[count].indexes.array = NULL;
    data->level[count].attr.memorysize = 0;
    if (*next_pos == '(') {
      err = hwloc_synthetic_parse_attrs(next_pos+1, &next_pos, &data->level[count].attr, &data->level[count].indexes, verbose);
      if (err < 0)
	goto error;
    }

    if (count + 1 >= HWLOC_SYNTHETIC_MAX_DEPTH) {
      if (verbose)
	fprintf(stderr,"Too many synthetic levels, max %d\n", HWLOC_SYNTHETIC_MAX_DEPTH);
      errno = EINVAL;
      goto error;
    }
    if (item > UINT_MAX) {
      if (verbose)
	fprintf(stderr,"Too big arity, max %u\n", UINT_MAX);
      errno = EINVAL;
      goto error;
    }

    data->level[count-1].arity = (unsigned)item;
    count++;
  }

  if (data->level[count-1].attr.type != HWLOC_OBJ_TYPE_NONE && data->level[count-1].attr.type != HWLOC_OBJ_PU) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot use non-PU type for last level\n");
    errno = EINVAL;
    return -1;
  }
  data->level[count-1].attr.type = HWLOC_OBJ_PU;

  for(i=HWLOC_OBJ_TYPE_MIN; i<HWLOC_OBJ_TYPE_MAX; i++) {
    type_count[i] = 0;
  }
  for(i=count-1; i>0; i--) {
    hwloc_obj_type_t type = data->level[i].attr.type;
    if (type != HWLOC_OBJ_TYPE_NONE) {
      type_count[type]++;
    }
  }

  /* sanity checks */
  if (!type_count[HWLOC_OBJ_PU]) {
    if (verbose)
      fprintf(stderr, "Synthetic string missing ending number of PUs\n");
    errno = EINVAL;
    return -1;
  } else if (type_count[HWLOC_OBJ_PU] > 1) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot have several PU levels\n");
    errno = EINVAL;
    return -1;
  }
  if (type_count[HWLOC_OBJ_PACKAGE] > 1) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot have several package levels\n");
    errno = EINVAL;
    return -1;
  }
  if (type_count[HWLOC_OBJ_DIE] > 1) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot have several die levels\n");
    errno = EINVAL;
    return -1;
  }
  if (type_count[HWLOC_OBJ_NUMANODE] > 1) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot have several NUMA node levels\n");
    errno = EINVAL;
    return -1;
  }
  if (type_count[HWLOC_OBJ_NUMANODE] && data->numa_attached_nr) {
    if (verbose)
      fprintf(stderr,"Synthetic string cannot have NUMA nodes both as a level and attached\n");
    errno = EINVAL;
    return -1;
  }
  if (type_count[HWLOC_OBJ_CORE] > 1) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot have several core levels\n");
    errno = EINVAL;
    return -1;
  }

  /* deal with missing intermediate levels */
  unset = 0;
  for(i=1; i<count-1; i++) {
    if (data->level[i].attr.type == HWLOC_OBJ_TYPE_NONE)
      unset++;
  }
  if (unset && unset != count-2) {
    if (verbose)
      fprintf(stderr, "Synthetic string cannot mix unspecified and specified types for levels\n");
    errno = EINVAL;
    return -1;
  }
  if (unset) {
    /* we want in priority: numa, package, core, up to 3 caches, groups */
    unsigned _count = count;
    unsigned neednuma = 0;
    unsigned needpack = 0;
    unsigned needcore = 0;
    unsigned needcaches = 0;
    unsigned needgroups = 0;
    /* 2 levels for machine and PU */
    _count -= 2;

    neednuma = (_count >= 1 && !data->numa_attached_nr);
    _count -= neednuma;

    needpack = (_count >= 1);
    _count -= needpack;

    needcore = (_count >= 1);
    _count -= needcore;

    needcaches = (_count > 4 ? 4 : _count);
    _count -= needcaches;

    needgroups = _count;

    /* we place them in order: groups, package, numa, caches, core */
    for(i = 0; i < needgroups; i++) {
      unsigned depth = 1 + i;
      data->level[depth].attr.type = HWLOC_OBJ_GROUP;
      type_count[HWLOC_OBJ_GROUP]++;
    }
    if (needpack) {
      unsigned depth = 1 + needgroups;
      data->level[depth].attr.type = HWLOC_OBJ_PACKAGE;
      type_count[HWLOC_OBJ_PACKAGE] = 1;
    }
    if (neednuma) {
      unsigned depth = 1 + needgroups + needpack;
      data->level[depth].attr.type = HWLOC_OBJ_NUMANODE;
      type_count[HWLOC_OBJ_NUMANODE] = 1;
    }
    if (needcaches) {
      /* priority: l2, l1, l3, l1i */
      /* order: l3, l2, l1, l1i */
      unsigned l3depth = 1 + needgroups + needpack + neednuma;
      unsigned l2depth = l3depth + (needcaches >= 3);
      unsigned l1depth = l2depth + 1;
      unsigned l1idepth = l1depth + 1;
      if (needcaches >= 3) {
	data->level[l3depth].attr.type = HWLOC_OBJ_L3CACHE;
	data->level[l3depth].attr.depth = 3;
	data->level[l3depth].attr.cachetype = HWLOC_OBJ_CACHE_UNIFIED;
	type_count[HWLOC_OBJ_L3CACHE] = 1;
      }
      data->level[l2depth].attr.type = HWLOC_OBJ_L2CACHE;
      data->level[l2depth].attr.depth = 2;
      data->level[l2depth].attr.cachetype = HWLOC_OBJ_CACHE_UNIFIED;
      type_count[HWLOC_OBJ_L2CACHE] = 1;
      if (needcaches >= 2) {
	data->level[l1depth].attr.type = HWLOC_OBJ_L1CACHE;
	data->level[l1depth].attr.depth = 1;
	data->level[l1depth].attr.cachetype = HWLOC_OBJ_CACHE_DATA;
	type_count[HWLOC_OBJ_L1CACHE] = 1;
      }
      if (needcaches >= 4) {
	data->level[l1idepth].attr.type = HWLOC_OBJ_L1ICACHE;
	data->level[l1idepth].attr.depth = 1;
	data->level[l1idepth].attr.cachetype = HWLOC_OBJ_CACHE_INSTRUCTION;
	type_count[HWLOC_OBJ_L1ICACHE] = 1;
      }
    }
    if (needcore) {
      unsigned depth = 1 + needgroups + needpack + neednuma + needcaches;
      data->level[depth].attr.type = HWLOC_OBJ_CORE;
      type_count[HWLOC_OBJ_CORE] = 1;
    }
  }

  /* enforce a NUMA level */
  if (!type_count[HWLOC_OBJ_NUMANODE] && !data->numa_attached_nr) {
    /* insert a NUMA level below the automatic machine root */
    if (verbose)
      fprintf(stderr, "Inserting a NUMA level with a single object at depth 1\n");
    /* move existing levels by one */
    memmove(&data->level[2], &data->level[1], count*sizeof(struct hwloc_synthetic_level_data_s));
    data->level[1].attr.type = HWLOC_OBJ_NUMANODE;
    data->level[1].indexes.string = NULL;
    data->level[1].indexes.array = NULL;
    data->level[1].attr.memorysize = 0;
    data->level[1].totalwidth = data->level[0].totalwidth;
    /* update arity to insert a single NUMA node per parent */
    data->level[1].arity = data->level[0].arity;
    data->level[0].arity = 1;
    count++;
  }

  for (i=0; i<count; i++) {
    struct hwloc_synthetic_level_data_s *curlevel = &data->level[i];
    hwloc_obj_type_t type = curlevel->attr.type;

    if (type == HWLOC_OBJ_GROUP) {
      if (curlevel->attr.depth == (unsigned)-1)
	curlevel->attr.depth = type_count[HWLOC_OBJ_GROUP]--;

    } else if (hwloc__obj_type_is_cache(type)) {
      if (!curlevel->attr.memorysize) {
	if (1 == curlevel->attr.depth)
	  /* 32KiB in L1 */
	  curlevel->attr.memorysize = 32*1024;
	else
	  /* *4 at each level, starting from 1MiB for L2, unified */
	  curlevel->attr.memorysize = 256ULL*1024 << (2*curlevel->attr.depth);
      }

    } else if (type == HWLOC_OBJ_NUMANODE && !curlevel->attr.memorysize) {
      /* 1GiB in memory nodes. */
      curlevel->attr.memorysize = 1024*1024*1024;
    }

    hwloc_synthetic_process_indexes(data, &data->level[i].indexes, data->level[i].totalwidth, verbose);
  }

  hwloc_synthetic_process_indexes(data, &data->numa_attached_indexes, data->numa_attached_nr, verbose);

  data->string = strdup(description);
  data->level[count-1].arity = 0;
  return 0;

 error:
  hwloc_synthetic_free_levels(data);
  return -1;
}

```cpp

This code appears to be setting the attributes of an object, such as the depth and the cache types, based on the values of an object's attributes. It is using a struct called `sattr` to store the values, which has several fields, including the memory size and the cache types. The code is using a `switch` statement to compare the `sattr` field to several different object types, and then setting the appropriate attribute values based on the comparison.


```
static void
hwloc_synthetic_set_attr(struct hwloc_synthetic_attr_s *sattr,
			 hwloc_obj_t obj)
{
  switch (obj->type) {
  case HWLOC_OBJ_GROUP:
    obj->attr->group.kind = HWLOC_GROUP_KIND_SYNTHETIC;
    obj->attr->group.subkind = sattr->depth-1;
    break;
  case HWLOC_OBJ_MACHINE:
    break;
  case HWLOC_OBJ_NUMANODE:
    obj->attr->numanode.local_memory = sattr->memorysize;
    obj->attr->numanode.page_types_len = 1;
    obj->attr->numanode.page_types = malloc(sizeof(*obj->attr->numanode.page_types));
    memset(obj->attr->numanode.page_types, 0, sizeof(*obj->attr->numanode.page_types));
    obj->attr->numanode.page_types[0].size = 4096;
    obj->attr->numanode.page_types[0].count = sattr->memorysize / 4096;
    break;
  case HWLOC_OBJ_PACKAGE:
  case HWLOC_OBJ_DIE:
    break;
  case HWLOC_OBJ_L1CACHE:
  case HWLOC_OBJ_L2CACHE:
  case HWLOC_OBJ_L3CACHE:
  case HWLOC_OBJ_L4CACHE:
  case HWLOC_OBJ_L5CACHE:
  case HWLOC_OBJ_L1ICACHE:
  case HWLOC_OBJ_L2ICACHE:
  case HWLOC_OBJ_L3ICACHE:
    obj->attr->cache.depth = sattr->depth;
    obj->attr->cache.linesize = 64;
    obj->attr->cache.type = sattr->cachetype;
    obj->attr->cache.size = sattr->memorysize;
    break;
  case HWLOC_OBJ_CORE:
    break;
  case HWLOC_OBJ_PU:
    break;
  default:
    /* Should never happen */
    assert(0);
    break;
  }
}

```cpp

这段代码定义了一个名为 `hwloc_synthetic_next_index` 的函数，属于 `hwloc_synthetic_indexes_s` 结构体的成员函数。

它的作用是获取 `hwloc_synthetic_indexes_s` 结构体中下一个可用的索引，并将其返回。

具体实现中，首先通过 `next` 成员函数获取下一个未被使用的索引，如果没有下一个可用的索引，则返回 `HWLOC_UNKNOWN_INDEX`。

如果 `indexes->array` 成员或者 `hwloc__obj_type_is_cache` 函数返回 `true`，则使用缓存索引策略，否则按照普通索引策略获取下一个索引。

需要注意的是，该函数只是用于在 `hwloc_synthetic_indexes_s` 结构体中获取下一个可用的索引，具体使用时还需要根据实际需求进行相应的检查和处理。


```
static unsigned
hwloc_synthetic_next_index(struct hwloc_synthetic_indexes_s *indexes, hwloc_obj_type_t type)
{
  unsigned os_index = indexes->next++;

  if (indexes->array)
    os_index = indexes->array[os_index];
  else if (hwloc__obj_type_is_cache(type) || type == HWLOC_OBJ_GROUP)
    /* don't enforce useless os_indexes for Caches and Groups */
    os_index = HWLOC_UNKNOWN_INDEX;

  return os_index;
}

static void
```cpp

这段代码定义了一个名为 `hwloc_synthetic_insert_attached` 的函数，它属于 `hwloc_synthetic_topology_水口` 系列的成员函数。

该函数的作用是，当给定的 `topology` 对象中存在一个名为 `attached` 的 `synthetic_attached` 结构体时，将 `attached` 中的 `next` 成员点的值作为参数传递给 `hwloc_synthetic_insert_attached` 函数，并将其与 `set` 一起作为参数传递给 `hwloc_synthetic_insert_attached` 函数。

函数的具体实现包括以下几步：

1. 如果 `attached` 是一个空结构体，直接返回。
2. 如果 `attached` 的 `attr.type` 是 `HWLOC_OBJ_NUMANODE`，那么认为 `attached` 中的 `next` 成员是一个命名网络适配器（device-attached）的 `device_node_id` 值。
3. 获取 `data` 中的下一个 `numa_attached_indexes` 值，作为 `attached` 的 `next` 成员点的索引。
4. 在 `topology` 对象中，使用 `hwloc_alloc_setup_object` 函数为新创建的 `attached` 对象的属性设置，并将其 `cpu_set` 属性设置为给定的 `set` 掩码。
5. 在 `attached` 对象中设置新创建的 `nodeset` 属性，并将其设置为 `attached` 中的 `next` 成员点的 `device_node_id` 值。
6. 使用 `hwloc_synthetic_set_attr` 函数设置 `attached` 对象的 `attr` 成员，其中这个函数会根据 `attached` 中的 `next` 成员点的 `device_node_id` 值，在 `topology` 对象中查找与该 `device_node_id` 对应的 `device_node` 对象，然后设置其对应的 `device_node_属性` 成员的值。
7. 使用 `hwloc__insert_object_by_cpu_set` 函数将 `attached` 对象插入到 `topology` 中的 `synthetic_attached` 结构体数组中，并使用给定的 `set` 掩码作为插入的依据。
8. 调用 `hwloc_synthetic_insert_attached` 函数，并将 `topology`、`data` 和 `attached` 作为参数传递给该函数。


```
hwloc_synthetic_insert_attached(struct hwloc_topology *topology,
				struct hwloc_synthetic_backend_data_s *data,
				struct hwloc_synthetic_attached_s *attached,
				hwloc_bitmap_t set)
{
  hwloc_obj_t child;
  unsigned attached_os_index;

  if (!attached)
    return;

  assert(attached->attr.type == HWLOC_OBJ_NUMANODE);

  attached_os_index = hwloc_synthetic_next_index(&data->numa_attached_indexes, HWLOC_OBJ_NUMANODE);

  child = hwloc_alloc_setup_object(topology, attached->attr.type, attached_os_index);
  child->cpuset = hwloc_bitmap_dup(set);

  child->nodeset = hwloc_bitmap_alloc();
  hwloc_bitmap_set(child->nodeset, attached_os_index);

  hwloc_synthetic_set_attr(&attached->attr, child);

  hwloc__insert_object_by_cpuset(topology, NULL, child, "synthetic:attached");

  hwloc_synthetic_insert_attached(topology, data, attached->next, set);
}

```cpp

This function appears to be a higher-level API for creating synthetic power levels on a parent device. It takes a single parameter, `parent_cpuset`, which is a bitmap indicating which CPUset(s) should be included in the synthetic level.

The function first checks the type of the input `parent_cpuset` and sets up a data structure to store the level information. If the input is a normal object, the function creates a new object of the appropriate type and iterates over the children to add their own power levels. If the input is a machine object, the function creates a new object of the appropriate type for each child and iterates over the children to add their own power levels.

The function then creates a bitmap indicating the parent CPUset and, if the input is a normal object, adds the children's power levels to it. The function also creates a bitmap for the children if the input is a machine object, and sets it to include the children's power levels.

Finally, the function sets up the synthetic level data structure and inserts the level into the parent device's synthetic data structure.

Note that this function assumes that the `parent_cpuset` is always a single bitmap representing the entire parent CPUset. If `parent_cpuset` is a bitmap representing a specific CPUset, the function only sets up the synthetic level for that CPUset.


```
/*
 * Recursively build objects whose cpu start at first_cpu
 * - level gives where to look in the type, arity and id arrays
 * - the id array is used as a variable to get unique IDs for a given level.
 * - generated memory should be added to *memory_kB.
 * - generated cpus should be added to parent_cpuset.
 * - next cpu number to be used should be returned.
 */
static void
hwloc__look_synthetic(struct hwloc_topology *topology,
		      struct hwloc_synthetic_backend_data_s *data,
		      int level,
		      hwloc_bitmap_t parent_cpuset)
{
  hwloc_obj_t obj;
  unsigned i;
  struct hwloc_synthetic_level_data_s *curlevel = &data->level[level];
  hwloc_obj_type_t type = curlevel->attr.type;
  hwloc_bitmap_t set;
  unsigned os_index;

  assert(hwloc__obj_type_is_normal(type) || type == HWLOC_OBJ_NUMANODE);
  assert(type != HWLOC_OBJ_MACHINE);

  os_index = hwloc_synthetic_next_index(&curlevel->indexes, type);

  set = hwloc_bitmap_alloc();
  if (!curlevel->arity) {
    hwloc_bitmap_set(set, os_index);
  } else {
    for (i = 0; i < curlevel->arity; i++)
      hwloc__look_synthetic(topology, data, level + 1, set);
  }

  hwloc_bitmap_or(parent_cpuset, parent_cpuset, set);

  if (hwloc_filter_check_keep_object_type(topology, type)) {
    obj = hwloc_alloc_setup_object(topology, type, os_index);
    obj->cpuset = hwloc_bitmap_dup(set);

    if (type == HWLOC_OBJ_NUMANODE) {
      obj->nodeset = hwloc_bitmap_alloc();
      hwloc_bitmap_set(obj->nodeset, os_index);
    }

    hwloc_synthetic_set_attr(&curlevel->attr, obj);

    hwloc__insert_object_by_cpuset(topology, NULL, obj, "synthetic");
  }

  hwloc_synthetic_insert_attached(topology, data, curlevel->attached, set);

  hwloc_bitmap_free(set);
}

```cpp

This function appears to be adding support for synthetic data types to a HWLOC environment. It is doing this by adding a single node to the topology, which will be used to represent the synthetic data.

Here are the specific steps it is taking:

1. Setting up the root certificate for the topology.
2. Setting the default policy for the topology to allow one-dimensional data.
3. Setting the arity of the first data type to 0, meaning it has no arity.
4. Adding the node to the topology to represent the synthetic data.
5. Synthesizing the first data type according to the specified data type array.
6. Inserting the node into the topology.
7. Freeing the memory allocated for the node.
8. Returning information about the added node in the topology.

It is intended to be used in a HWLOC environment to enable the use of synthetic data types. The specific details of how it is being used are not provided.


```
static int
hwloc_look_synthetic(struct hwloc_backend *backend, struct hwloc_disc_status *dstatus)
{
  /*
   * This backend enforces !topology->is_thissystem by default.
   */

  struct hwloc_topology *topology = backend->topology;
  struct hwloc_synthetic_backend_data_s *data = backend->private_data;
  hwloc_bitmap_t cpuset = hwloc_bitmap_alloc();
  unsigned i;

  assert(dstatus->phase == HWLOC_DISC_PHASE_GLOBAL);

  assert(!topology->levels[0][0]->cpuset);

  hwloc_alloc_root_sets(topology->levels[0][0]);

  topology->support.discovery->pu = 1;
  topology->support.discovery->numa = 1; /* we add a single NUMA node if none is given */
  topology->support.discovery->numa_memory = 1; /* specified or default size */

  /* start with os_index 0 for each level */
  for (i = 0; data->level[i].arity > 0; i++)
    data->level[i].indexes.next = 0;
  data->numa_attached_indexes.next = 0;
  /* ... including the last one */
  data->level[i].indexes.next = 0;

  /* update first level type according to the synthetic type array */
  topology->levels[0][0]->type = data->level[0].attr.type;
  hwloc_synthetic_set_attr(&data->level[0].attr, topology->levels[0][0]);

  for (i = 0; i < data->level[0].arity; i++)
    hwloc__look_synthetic(topology, data, 1, cpuset);

  hwloc_synthetic_insert_attached(topology, data, data->level[0].attached, cpuset);

  hwloc_bitmap_free(cpuset);

  hwloc_obj_add_info(topology->levels[0][0], "Backend", "Synthetic");
  hwloc_obj_add_info(topology->levels[0][0], "SyntheticDescription", data->string);
  return 0;
}

```cpp

This function appears to be a part of the HWLOC library, which appears to be a library for managing the hidden HWL components of Linux systems.

The function `hwloc_synthetic_component_instantiate` takes three arguments:

* `topology`: a `hwloc_topology` object, which represents the topology of the system.
* `component`: a `struct hwloc_disc_component` object, which represents the physical device.
* `_data1`, `_data2`, and `_data3`: unused arguments that are passed to the function caller.

The function returns a pointer to a new `hwloc_synthetic_backend_data_s` structure, which contains the data needed to instantiate the component.

If the first argument is not present or is an invalid value, the function will exit with an error. If the component is not a valid device, the function will also exit with an error.

Note that the function has several helper functions that it calls, such as `hwloc_look_synthetic`, `malloc`, and `free`, which are not defined in the provided code. It is possible that these functions are part of the HWLOC library and are being used here to handle any errors or edge cases that may occur during the instantiation process.


```
static void
hwloc_synthetic_backend_disable(struct hwloc_backend *backend)
{
  struct hwloc_synthetic_backend_data_s *data = backend->private_data;
  hwloc_synthetic_free_levels(data);
  free(data->string);
  free(data);
}

static struct hwloc_backend *
hwloc_synthetic_component_instantiate(struct hwloc_topology *topology,
				      struct hwloc_disc_component *component,
				      unsigned excluded_phases __hwloc_attribute_unused,
				      const void *_data1,
				      const void *_data2 __hwloc_attribute_unused,
				      const void *_data3 __hwloc_attribute_unused)
{
  struct hwloc_backend *backend;
  struct hwloc_synthetic_backend_data_s *data;
  int err;

  if (!_data1) {
    const char *env = getenv("HWLOC_SYNTHETIC");
    if (env) {
      /* 'synthetic' was given in HWLOC_COMPONENTS without a description */
      _data1 = env;
    } else {
      errno = EINVAL;
      goto out;
    }
  }

  backend = hwloc_backend_alloc(topology, component);
  if (!backend)
    goto out;

  data = malloc(sizeof(*data));
  if (!data) {
    errno = ENOMEM;
    goto out_with_backend;
  }

  err = hwloc_backend_synthetic_init(data, (const char *) _data1);
  if (err < 0)
    goto out_with_data;

  backend->private_data = data;
  backend->discover = hwloc_look_synthetic;
  backend->disable = hwloc_synthetic_backend_disable;
  backend->is_thissystem = 0;

  return backend;

 out_with_data:
  free(data);
 out_with_backend:
  free(backend);
 out:
  return NULL;
}

```cpp

这段代码定义了一个名为 `hwloc_synthetic_disc_component` 的结构体，它是一个 `hwloc_disc_component` 的实例。

`hwloc_synthetic_disc_component` 的字面意思为 "synthetic"(即 "合成")，表示这是一个全球性的合成设备。`HWLOC_DISC_PHASE_GLOBAL` 表示该设备是全球统一的，即无论在哪里，它都使用相同的相位。

`~0` 表示这个设备可以被任何驱动器驱动，即它是一个被动的设备。

`hwloc_synthetic_component_instantiate` 是 `hwloc_synthetic_component` 的成员函数，用于创建一个新的 `hwloc_synthetic_disc_component` 实例并将其返回。

`const struct hwloc_component hwloc_synthetic_component` 定义了一个常量结构体 `hwloc_synthetic_component`，其中包含 `hwloc_component` 类型的成员变量。这些成员变量包括：

- `HWLOC_COMPONENT_ABI`：这是一个常量，表示这是一个抽象设备，它的值不被引擎或硬件平台指定。
- `NULL`：这是一个指针，指向一个 `hwloc_component` 类型的空指针，这个空指针在 `hwloc_synthetic_component_instantiate` 函数中用于设置新实例的空指针。
- `HWLOC_COMPONENT_TYPE_DISC`：这是一个整数，表示这是一个磁盘设备，它的值为 `HWLOC_COMPONENT_TYPE_DISC`。
- `0`：这是一个整数，表示这是一个被动的设备，它的值为 0。
- `NULL`：这是一个指针，指向一个 `hwloc_disc_component` 类型的空指针，这个空指针在 `hwloc_synthetic_component_instantiate` 函数中用于设置新实例的空指针。

最后，`hwloc_synthetic_component_instantiate` 函数将返回一个新的 `hwloc_synthetic_disc_component` 实例，并将其存储在 `hwloc_synthetic_component` 指向的变量中。


```
static struct hwloc_disc_component hwloc_synthetic_disc_component = {
  "synthetic",
  HWLOC_DISC_PHASE_GLOBAL,
  ~0,
  hwloc_synthetic_component_instantiate,
  30,
  1,
  NULL
};

const struct hwloc_component hwloc_synthetic_component = {
  HWLOC_COMPONENT_ABI,
  NULL, NULL,
  HWLOC_COMPONENT_TYPE_DISC,
  0,
  &hwloc_synthetic_disc_component
};

```cpp

这两段代码是针对 hwloc_synthetic_update_status() 函数进行实现的。这个函数的作用是更新一个指定同步任务的 status 变量，并根据传入的参数更新该同步任务的tmplen数组以及 status 变量。以下是这两段代码的具体解释：

1. hwloc__export_synthetic_update_status() 函数的作用是接收一个 int 类型的 status 变量，一个字符型指针 tmp 变量，以及一个指向字符型数组的 tmplen 数组。这个函数首先检查 res 变量是否小于 0，如果是，就返回 -1。然后，将 res 加到 status 变量上，并将 res 如果大于或等于 tmplen 中的一个元素，就将其设为 0。接着，将 res 减到 tmplen 数组中，并将 tmplen 数组长度减去 res。最后，函数返回 0，表示更新成功。

2. hwloc__export_synthetic_add_char() 函数的作用是接收一个 int 类型的 status 变量，一个指向字符型数组的 tmplen 数组，以及一个字符 c 变量。这个函数首先检查 tmplen 数组是否大于 1。如果是，就在 tmplen 数组的第一个元素中设置 c 变量，将第二个元素设置为 '\0'，并将 tmplen 数组长度设置为 0。接着，将 c 变量添加到 status 变量上，并将 tmplen 数组长度减少 1。最后，函数返回 0，表示更新成功。

这两段代码共同作用于 hwloc_synthetic_update_status() 函数，根据传入的参数更新同步任务的 status 变量，以及更新 tmplen 数组。


```
static __hwloc_inline int
hwloc__export_synthetic_update_status(int *ret, char **tmp, ssize_t *tmplen, int res)
{
  if (res < 0)
    return -1;
  *ret += res;
  if (res >= *tmplen)
    res = *tmplen>0 ? (int)(*tmplen) - 1 : 0;
  *tmp += res;
  *tmplen -= res;
  return 0;
}

static __hwloc_inline void
hwloc__export_synthetic_add_char(int *ret, char **tmp, ssize_t *tmplen, char c)
{
  if (*tmplen > 1) {
    (*tmp)[0] = c;
    (*tmp)[1] = '\0';
    (*tmp)++;
    (*tmplen)--;
  }
  (*ret)++;
}

```cpp

这段代码是一个简单的多线程程序，用于计算并打印多线程成绩。它主要实现了以下功能：

1. 根据传入的线程数和每个线程需要计算的圈数，合理分配计算任务。
2. 对多线程成绩进行汇总，使得每个线程的得分都可以打印出来。

代码中涉及到的函数有：

* realloc：用于重新分配内存，支持按指定索引和维度重新分配。
* hwloc_snprintf：用于打印字符串，可以将字符串填充到指定的内存空间中，并返回其有效载荷（'\0'的数量）。
* cur：用于获取当前线程的局部变量，如得分等。
* next_cousin：用于获取当前线程的邻居线程，如需要计算其他线程得分的情况。

代码中的一些关键点：

* 空间复杂度：该程序的空间复杂度为O(nlogn)，主要原因包括：计算线程数、计算每个线程需要计算的圈数、以及创建和释放包围盒等操作。
* 实际执行时间：在测试数据的情况下，该程序的实际执行时间在10几秒到20几秒之间，可能受到具体硬件环境、计算任务和运行时间的影响。


```
static int
hwloc__export_synthetic_indexes(hwloc_obj_t *level, unsigned total,
				char *buffer, size_t buflen)
{
  unsigned step = 1;
  unsigned nr_loops = 0;
  struct hwloc_synthetic_intlv_loop_s *loops = NULL, *tmploops;
  hwloc_obj_t cur;
  unsigned i, j;
  ssize_t tmplen = buflen;
  char *tmp = buffer;
  int res, ret = 0;

  /* must start with 0 */
  if (level[0]->os_index)
    goto exportall;

  while (step != total) {
    /* must be a divider of the total */
    if (total % step)
      goto exportall;

    /* look for os_index == step */
    for(i=1; i<total; i++)
      if (level[i]->os_index == step)
	break;
    if (i == total)
      goto exportall;
    for(j=2; j<total/i; j++)
      if (level[i*j]->os_index != step*j)
	break;

    nr_loops++;
    tmploops = realloc(loops, nr_loops*sizeof(*loops));
    if (!tmploops)
      goto exportall;
    loops = tmploops;
    loops[nr_loops-1].step = i;
    loops[nr_loops-1].nb = j;
    step *= j;
  }

  /* check this interleaving */
  for(i=0; i<total; i++) {
    unsigned ind = 0;
    unsigned mul = 1;
    for(j=0; j<nr_loops; j++) {
      ind += (i / loops[j].step) % loops[j].nb * mul;
      mul *= loops[j].nb;
    }
    if (level[i]->os_index != ind)
      goto exportall;
  }

  /* success, print it */
  for(j=0; j<nr_loops; j++) {
    res = hwloc_snprintf(tmp, tmplen, "%u*%u%s", loops[j].step, loops[j].nb,
			 j == nr_loops-1 ? ")" : ":");
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0) {
      free(loops);
      return -1;
    }
  }

  free(loops);
  return ret;

 exportall:
  free(loops);

  /* dump all indexes */
  cur = level[0];
  while (cur) {
    res = hwloc_snprintf(tmp, tmplen, "%u%s", cur->os_index,
			 cur->next_cousin ? "," : ")");
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
    cur = cur->next_cousin;
  }
  return ret;
}

```cpp

This function appears to be a utility function for managing the cache and memory usage in a DOM node object. It takes in a node object, and allows methods to export the cache and memory usage as a string.

The function has several helper functions that are used to accomplish this:

* `hwloc_snprintf()`: a function that takes in a format string and a buffer, and returns the result of formatting the string using the specified buffer.
* `hwloc__export_synthetic_update_status()`: a function that takes in a handle to a synchronizable object, and returns a status indicating whether the object is being updated in the current cache.
* `hwloc__export_synthetic_indexes()`: a function that takes in a handle to a synchronizable object, and returns a pointer to the CSS counter (in page units) corresponding to the given indexes.
* `topology->slevels()`: a function that returns the DOM level number corresponding to the given level index.
* `topology->level_nbobjects()`: a function that returns the total number of DOM objects for the given level index.

The function also assumes that the node object has a depth property, which is used to determine whether it is a top-level node or a node with a depth property.

Overall, this function appears to be a utility function for managing the cache and memory usage of a DOM node object.


```
static int
hwloc__export_synthetic_obj_attr(struct hwloc_topology * topology,
				 hwloc_obj_t obj,
				 char *buffer, size_t buflen)
{
  const char * separator = " ";
  const char * prefix = "(";
  char cachesize[64] = "";
  char memsize[64] = "";
  int needindexes = 0;

  if (hwloc__obj_type_is_cache(obj->type) && obj->attr->cache.size) {
    snprintf(cachesize, sizeof(cachesize), "%ssize=%llu",
	     prefix, (unsigned long long) obj->attr->cache.size);
    prefix = separator;
  }
  if (obj->type == HWLOC_OBJ_NUMANODE && obj->attr->numanode.local_memory) {
    snprintf(memsize, sizeof(memsize), "%smemory=%llu",
	     prefix, (unsigned long long) obj->attr->numanode.local_memory);
    prefix = separator;
  }
  if (!obj->logical_index /* only display indexes once per level (not for non-first NUMA children, etc.) */
      && (obj->type == HWLOC_OBJ_PU || obj->type == HWLOC_OBJ_NUMANODE)) {
    hwloc_obj_t cur = obj;
    while (cur) {
      if (cur->os_index != cur->logical_index) {
	needindexes = 1;
	break;
      }
      cur = cur->next_cousin;
    }
  }
  if (*cachesize || *memsize || needindexes) {
    ssize_t tmplen = buflen;
    char *tmp = buffer;
    int res, ret = 0;

    res = hwloc_snprintf(tmp, tmplen, "%s%s%s", cachesize, memsize, needindexes ? "" : ")");
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;

    if (needindexes) {
      unsigned total;
      hwloc_obj_t *level;

      if (obj->depth < 0) {
	assert(obj->depth == HWLOC_TYPE_DEPTH_NUMANODE);
	total = topology->slevels[HWLOC_SLEVEL_NUMANODE].nbobjs;
	level = topology->slevels[HWLOC_SLEVEL_NUMANODE].objs;
      } else {
	total = topology->level_nbobjects[obj->depth];
	level = topology->levels[obj->depth];
      }

      res = hwloc_snprintf(tmp, tmplen, "%sindexes=", prefix);
      if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
	return -1;

      res = hwloc__export_synthetic_indexes(level, total, tmp, tmplen);
      if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
	return -1;
    }
    return ret;
  } else {
    return 0;
  }
}

```cpp

This function appears to be part of the `hwloc_obj_export` function, which is responsible for exporting the properties of an HWLOC object to another process.

The function appears to accept an additional argument `topology`, which is an enumeration indicating the topology type of the HWLOC object. The function takes this argument in addition to the standard arguments for an `hwloc_obj_export` function.

The function then checks whether the `topology` argument is equal to `HWLOC_TOPOLOGY_GUI_DEPTH_SINGLE` or `HWLOC_TOPOLOGY_GUI_DEPTH_REL` and whether the `flags` argument has the `HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES` or `HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1` flag. If either of these conditions is `true`, the function uses a fallback group name and a placeholder for the group. If the `topology` argument is `HWLOC_TOPOLOGY_GUI_DEPTH_SINGLE` or `HWLOC_TOPOLOGY_GUI_DEPTH_REL`, the function uses the all-v1-compatible group name.

If the `topology` argument is neither of these values, the function uses the `%s` format string to generate a string that includes the object type and the group name. The function then returns this string, along with any additional attributes that have been set for the object. If the export of the object attributes fails, the function returns `-1`.


```
static int
hwloc__export_synthetic_obj(struct hwloc_topology * topology, unsigned long flags,
			    hwloc_obj_t obj, unsigned arity,
			    char *buffer, size_t buflen)
{
  char aritys[12] = "";
  ssize_t tmplen = buflen;
  char *tmp = buffer;
  int res, ret = 0;

  /* <type>:<arity>, except for root */
  if (arity != (unsigned)-1)
    snprintf(aritys, sizeof(aritys), ":%u", arity);
  if (hwloc__obj_type_is_cache(obj->type)
      && (flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES)) {
    /* v1 uses generic "Cache" for non-extended type name */
    res = hwloc_snprintf(tmp, tmplen, "Cache%s", aritys);

  } else if (obj->type == HWLOC_OBJ_PACKAGE
	     && (flags & (HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES
			  |HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1))) {
    /* if exporting to v1 or without extended-types, use all-v1-compatible Socket name */
    res = hwloc_snprintf(tmp, tmplen, "Socket%s", aritys);

  } else if (obj->type == HWLOC_OBJ_DIE
	     && (flags & (HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES
			  |HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1))) {
    /* if exporting to v1 or without extended-types, use all-v1-compatible Group name */
    res = hwloc_snprintf(tmp, tmplen, "Group%s", aritys);

  } else if (obj->type == HWLOC_OBJ_GROUP /* don't export group depth */
      || flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES) {
    res = hwloc_snprintf(tmp, tmplen, "%s%s", hwloc_obj_type_string(obj->type), aritys);
  } else {
    char types[64];
    hwloc_obj_type_snprintf(types, sizeof(types), obj, 1);
    res = hwloc_snprintf(tmp, tmplen, "%s%s", types, aritys);
  }
  if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
    return -1;

  if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS)) {
    /* obj attributes */
    res = hwloc__export_synthetic_obj_attr(topology, obj, tmp, tmplen);
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
  }

  return ret;
}

```cpp

This is a function that performs an export of a synthetic object for a given topology and memory hierarchy. It uses the hwloc__export\_synthetic\_obj function to export the object and updates the tmplen parameter to handle multiple inheritance. The function also checks for the need to prefix the object by adding a ':' character at the beginning.

The function takes a topology object, a set of flags, a memory child object, and an index for the maximum number of inheritance levels to search for. It recursively expends the synthetic object by adding new members and updating the tmplen parameter.

The function returns 0 on success and-1 on failure.


```
static int
hwloc__export_synthetic_memory_children(struct hwloc_topology * topology, unsigned long flags,
					hwloc_obj_t parent,
					char *buffer, size_t buflen,
					int needprefix, int verbose)
{
  hwloc_obj_t mchild;
  ssize_t tmplen = buflen;
  char *tmp = buffer;
  int res, ret = 0;

  mchild = parent->memory_first_child;
  if (!mchild)
    return 0;

  if (flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1) {
    /* v1: export a single NUMA child */
    if (parent->memory_arity > 1 || mchild->type != HWLOC_OBJ_NUMANODE) {
      /* not supported */
      if (verbose)
	fprintf(stderr, "Cannot export to synthetic v1 if multiple memory children are attached to the same location.\n");
      errno = EINVAL;
      return -1;
    }

    if (needprefix)
      hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, ' ');

    res = hwloc__export_synthetic_obj(topology, flags, mchild, 1, tmp, tmplen);
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
    return ret;
  }

  while (mchild) {
    /* FIXME: really recurse to export memcaches and numanode,
     * but it requires clever parsing of [ memcache [numa] [numa] ] during import,
     * better attaching of things to describe the hierarchy.
     */
    hwloc_obj_t numanode = mchild;
    /* only export the first NUMA node leaf of each memory child
     * FIXME: This assumes mscache aren't shared between nodes, that's true in current platforms
     */
    while (numanode && numanode->type != HWLOC_OBJ_NUMANODE) {
      assert(numanode->arity == 1);
      numanode = numanode->memory_first_child;
    }
    assert(numanode); /* there's always a numanode at the bottom of the memory tree */

    if (needprefix)
      hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, ' ');

    hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, '[');

    res = hwloc__export_synthetic_obj(topology, flags, numanode, (unsigned)-1, tmp, tmplen);
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;

    hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, ']');

    needprefix = 1;
    mchild = mchild->next_sibling;
  }

  return ret;
}

```cpp

This function appears to modify the NUMA topology of a `struct hwloc_topology` object. It does this by first counting the number of nodes in the topology, and then modifying the bitmap representing those nodes to ensure that all nodes on the same depth level have the same number of NUMA bits. This is done by first checking whether the bitmap is already zero, and if it is not, it then creates a new bitmap that is essentially a copy of the original bitmap. The new bitmap is then used to keep track of which nodes on each depth level have the same number of NUMA bits. Finally, the original bitmap is freed and the function returns 0 if the modification was successful, or -1 if it was not.


```
static int
hwloc_check_memory_symmetric(struct hwloc_topology * topology)
{
  hwloc_bitmap_t remaining_nodes;

  remaining_nodes = hwloc_bitmap_dup(hwloc_get_root_obj(topology)->nodeset);
  if (!remaining_nodes)
    /* assume asymmetric */
    return -1;

  while (!hwloc_bitmap_iszero(remaining_nodes)) {
    unsigned idx;
    hwloc_obj_t node;
    hwloc_obj_t first_parent;
    unsigned i;

    idx = hwloc_bitmap_first(remaining_nodes);
    node = hwloc_get_numanode_obj_by_os_index(topology, idx);
    assert(node);

    first_parent = node->parent;

    /* check whether all object on parent's level have same number of NUMA bits */
    for(i=0; i<hwloc_get_nbobjs_by_depth(topology, first_parent->depth); i++) {
      hwloc_obj_t parent, mchild;

      parent = hwloc_get_obj_by_depth(topology, first_parent->depth, i);
      assert(parent);

      /* must have same memory arity */
      if (parent->memory_arity != first_parent->memory_arity)
	goto out_with_bitmap;

      /* clear children NUMA bits from remaining_nodes */
      mchild = parent->memory_first_child;
      while (mchild) {
	hwloc_bitmap_clr(remaining_nodes, mchild->os_index); /* cannot use parent->nodeset, some normal children may have other NUMA nodes */
	mchild = mchild->next_sibling;
      }
    }
  }

  hwloc_bitmap_free(remaining_nodes);
  return 0;

 out_with_bitmap:
  hwloc_bitmap_free(remaining_nodes);
  return -1;
}

```cpp

This function appears to be a C language implementation of the `hwloc__export_synthetic_obj_attr` function, which is part of the hwloc library for高度可编程的硬件资源分配。

It appears to handle the case where a device tree node has a `__synthetic` attribute, which indicates that it should be generated by the library's synthetic object model.

The function takes four arguments:

* `topology`: a HWLOC topology object
* `obj`: a HWLOC object, such as a device tree node
* `tmp`: a template for the synthetic object model
* `tmplen`: the template length in bytes of the memory template
* `flags`: a set of flags indicating whether to ignore the memory and synchronize the object model updates
* `verbose`: an optional verbose flag for debug output

The function returns the result of an `hwloc__export_synthetic` call, including any additional arguments (such as the `needprefix` flag) passed to it.

If the export of the synthetic object is successful, the function returns 0. If the export fails, the function returns `-1`.

If the device tree node has a `__synthetic` attribute and `hwloc__export_synthetic_fl盂_旗` is not set, the function will export the synthetic object, updating the object's `tmp` template and the `tmplen` memory template.

Note: the `hwloc__export_synthetic_obj` function used here assumes that the synthetic object model is generated using the `hwloc__export_synthetic_obj_attr` function.


```
int
hwloc_topology_export_synthetic(struct hwloc_topology * topology,
				char *buffer, size_t buflen,
				unsigned long flags)
{
  hwloc_obj_t obj = hwloc_get_root_obj(topology);
  ssize_t tmplen = buflen;
  char *tmp = buffer;
  int res, ret = 0;
  unsigned arity;
  int needprefix = 0;
  int verbose = 0;
  const char *env = getenv("HWLOC_SYNTHETIC_VERBOSE");

  if (env)
    verbose = atoi(env);

  if (!topology->is_loaded) {
    errno = EINVAL;
    return -1;
  }

  if (flags & ~(HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES
		|HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS
		|HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1
		|HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)) {
    errno = EINVAL;
    return -1;
  }

  /* TODO: add a flag to ignore symmetric_subtree and I/Os.
   * just assume things are symmetric with the left branches of the tree.
   * but the number of objects per level may be wrong, what to do with OS index array in this case?
   * only allow ignoring symmetric_subtree if the level width remains OK?
   */

  /* TODO: add a root object by default, with a prefix such as tree=
   * so that we can backward-compatibly recognize whether there's a root or not.
   * and add a flag to disable it.
   */

  /* TODO: flag to force all indexes, not only for PU and NUMA? */

  if (!obj->symmetric_subtree) {
    if (verbose)
      fprintf(stderr, "Cannot export to synthetic unless topology is symmetric (root->symmetric_subtree must be set).\n");
    errno = EINVAL;
    return -1;
  }

  if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)
      && hwloc_check_memory_symmetric(topology) < 0) {
    if (verbose)
      fprintf(stderr, "Cannot export to synthetic unless memory is attached symmetrically.\n");
    errno = EINVAL;
    return -1;
  }

  if (flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1) {
    /* v1 requires all NUMA at the same level */
    hwloc_obj_t node;
    signed pdepth;

    node = hwloc_get_obj_by_type(topology, HWLOC_OBJ_NUMANODE, 0);
    assert(node);
    assert(hwloc__obj_type_is_normal(node->parent->type)); /* only depth-1 memory children for now */
    pdepth = node->parent->depth;

    while ((node = node->next_cousin) != NULL) {
      assert(hwloc__obj_type_is_normal(node->parent->type)); /* only depth-1 memory children for now */
      if (node->parent->depth != pdepth) {
	if (verbose)
	  fprintf(stderr, "Cannot export to synthetic v1 if memory is attached to parents at different depths.\n");
	errno = EINVAL;
	return -1;
      }
    }
  }

  /* we're good, start exporting */

  if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS)) {
    /* obj attributes */
    res = hwloc__export_synthetic_obj_attr(topology, obj, tmp, tmplen);
    if (res > 0)
      needprefix = 1;
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
  }

  if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)) {
    res = hwloc__export_synthetic_memory_children(topology, flags, obj, tmp, tmplen, needprefix, verbose);
    if (res > 0)
      needprefix = 1;
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;
  }

  arity = obj->arity;
  while (arity) {
    /* for each level */
    obj = obj->first_child;

    if (needprefix)
      hwloc__export_synthetic_add_char(&ret, &tmp, &tmplen, ' ');

    res = hwloc__export_synthetic_obj(topology, flags, obj, arity, tmp, tmplen);
    if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
      return -1;

    if (!(flags & HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)) {
      res = hwloc__export_synthetic_memory_children(topology, flags, obj, tmp, tmplen, 1, verbose);
      if (hwloc__export_synthetic_update_status(&ret, &tmp, &tmplen, res) < 0)
	return -1;
    }

    /* next level */
    needprefix = 1;
    arity = obj->arity;
  }

  return ret;
}

```