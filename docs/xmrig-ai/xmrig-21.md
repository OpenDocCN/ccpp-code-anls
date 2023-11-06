# xmrig源码解析 21

# `src/3rdparty/hwloc/include/hwloc/deprecated.h`

这段代码定义了一个名为"hwloc\_dePRECATED\_h"的函数。函数声明在hwloc.h文件的头部，并在该文件中进行了定义。

具体来说，这段代码定义了一个名为"init\_hwloc"的函数，它的函数体如下：
```cppperl
static int init_hwloc(hwloc_caller_t caller, hwloc_component_t component, hwloc_location_t location) {
   /* Use the static function to initialize hwloc */
   return 0;
}
```
在函数体中，使用了static关键字修饰了一个名为"init\_hwloc"的函数，并定义了一个名为"init\_hwloc"的函数，该函数的参数为caller、component和location，分别表示调用该函数的用户、组件和位置。函数实现了一个简单的初始化操作，将hwloc的初始状态设置为ok。

另外，在hwloc.h文件的顶部，还包含了一些函数声明和定义，如"get\_executable"，这些函数在hwloc的内部用于获取和设置hwloc的执行器。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2022 Inria.  All rights reserved.
 * Copyright © 2009-2012 Université Bordeaux
 * Copyright © 2009-2010 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/**
 * This file contains the inline code of functions declared in hwloc.h
 */

#ifndef HWLOC_DEPRECATED_H
#define HWLOC_DEPRECATED_H

```

这段代码是一个名为"hwloc.h"的头文件，它定义了一些用于定义硬件加速库（HWLOC）中特定变量和函数的宏和外部定义。以下是这段代码的作用：

1. 如果定义了"hwloc.h"，则将包含一个错误消息，要求使用"main.h"而不是"hwloc.h"进行定义。如果没有定义"hwloc.h"，则这段代码将不再产生任何错误消息。
2. 如果你在定义"hwloc.h"时使用了"#define"，则这段代码定义了一些使用"#ifdef"和"#error"预处理指令的宏，用于定义和检查定义的变量和函数。
3. 如果你在定义"hwloc.h"时使用了"#ifdef"，则这段代码定义了一些使用"#include"预处理指令的函数。这些函数将在编译时链接到特定于此特定头文件的外部库。
4. 如果你在定义"hwloc.h"时使用了"#error"，则这段代码定义了一些使用"#ifdef"和"#error"预处理指令的常量，用于在编译时检查特定错误。


```cpp
#ifndef HWLOC_H
#error Please include the main hwloc.h instead
#endif

#ifdef __cplusplus
extern "C" {
#endif

/* backward compat with v2.0 before WHOLE_SYSTEM renaming */
#define HWLOC_TOPOLOGY_FLAG_WHOLE_SYSTEM HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED
/* backward compat with v1.11 before System removal */
#define HWLOC_OBJ_SYSTEM HWLOC_OBJ_MACHINE
/* backward compat with v1.10 before Socket->Package renaming */
#define HWLOC_OBJ_SOCKET HWLOC_OBJ_PACKAGE
/* backward compat with v1.10 before Node->NUMANode clarification */
```

这段代码定义了一个名为"HWLOC_OBJ_NODE"的宏，它等价于"HWLOC_OBJ_NUMANODE"。接下来定义了一个名为"hwloc_distances_add"的函数，它的参数包括一个"topology"表示整个拓扑结构，一个"nbobjs"表示对象的数量，以及一个"values"表示距离值的指针。它的函数实现包括：

1. 如果定义在 v2.5 中，使用 __hwloc_attribute_deprecated 特性，在函数内部声明了一个函数指针 "hwloc_distances_add_create"，这个函数将调用它并使用它提供的参数。
2. 如果定义在 v2.5 或更早版本中，使用 __hwloc_attribute_deprecated 特性，在函数内部声明了一个函数指针 "hwloc_distances_add_values"，这个函数将调用它并使用它提供的参数。
3. 如果定义在 v2.5 或更早版本中，在函数内部实现了一个加法操作，将所有距离值加到给定的对象上。
4. 函数的返回值是一个整数，表示成功添加的距离值。


```cpp
#define HWLOC_OBJ_NODE HWLOC_OBJ_NUMANODE

/** \brief Add a distances structure.
 *
 * Superseded by hwloc_distances_add_create()+hwloc_distances_add_values()+hwloc_distances_add_commit()
 * in v2.5.
 */
HWLOC_DECLSPEC int hwloc_distances_add(hwloc_topology_t topology,
				       unsigned nbobjs, hwloc_obj_t *objs, hwloc_uint64_t *values,
				       unsigned long kind, unsigned long flags) __hwloc_attribute_deprecated;

/** \brief Insert a misc object by parent.
 *
 * Identical to hwloc_topology_insert_misc_object().
 */
```

这段代码定义了一个名为 `hwloc_topology_insert_misc_object_by_parent` 的函数，属于 `hwloc_topology_api` 系列的函数。它的作用是插入一个名为 `name` 的对象到指定的 topology 中，如果指定的 topology 中不存在该对象，则会创建一个新的对象并添加到 topology 中。

具体来说，代码首先定义了一个名为 `hwloc_topology_insert_misc_object_by_parent` 的函数，它接受两个参数 `topology` 和 `parent`，分别代表要插入对象的 topology 和该对象的父对象。然后，定义了一个名为 `hwloc_topology_insert_misc_object_by_parent` 的函数，它与上面那个函数等价，只是将参数 `name` 改为了 `NULL`。

这两个函数的实现主要依赖于 `hwloc_topology_insert_misc_object` 和 `hwloc_topology_insert_misc_object_by_name` 函数，它们可能来源于某个库或头文件，具体来源未给出。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_topology_insert_misc_object_by_parent(hwloc_topology_t topology, hwloc_obj_t parent, const char *name) __hwloc_attribute_deprecated;
static __hwloc_inline hwloc_obj_t
hwloc_topology_insert_misc_object_by_parent(hwloc_topology_t topology, hwloc_obj_t parent, const char *name)
{
  return hwloc_topology_insert_misc_object(topology, parent, name);
}

/** \brief Stringify the cpuset containing a set of objects.
 *
 * If \p size is 0, \p string may safely be \c NULL.
 *
 * \return the number of characters that were actually written if not truncating,
 * or that would have been written (not including the ending \\0).
 */
```

这段代码定义了一个名为 `hwloc_obj_cpuset_snprintf` 的函数，它的参数是一个字符指针 `str`，表示要输出的一串字符串，以及一个整数 `size`，表示字符串的长度。还有两个整数参数 `nobj` 和 `objs`, 分别表示要遍历的对象的个数和每个对象中可CPU集合的个数。

函数实现中，首先定义了一个名为 `set` 的 bitmap 类型变量，用于存储 CPU 集合。接着定义了一个整数变量 `res`，用于保存字符串格式化后的结果。然后，定义了一个循环，遍历每个对象，检查它是否有一个可 CPU 集合，如果有的话，将该对象的 CPU 集合与 `set` 中的所有对象相或，并将结果存储到 `res` 中。最后，通过 `snprintf` 函数将字符串和设置好的 bitmap 对象进行格式化，并将结果存储回 `res`。最后，通过调用 `hwloc_bitmap_free` 和 `hwloc_bitmap_alloc` 函数来释放和分配 bitmap 对象。


```cpp
static __hwloc_inline int
hwloc_obj_cpuset_snprintf(char *str, size_t size, size_t nobj, struct hwloc_obj * const *objs) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_obj_cpuset_snprintf(char *str, size_t size, size_t nobj, struct hwloc_obj * const *objs)
{
  hwloc_bitmap_t set = hwloc_bitmap_alloc();
  int res;
  unsigned i;

  hwloc_bitmap_zero(set);
  for(i=0; i<nobj; i++)
    if (objs[i]->cpuset)
      hwloc_bitmap_or(set, set, objs[i]->cpuset);

  res = hwloc_bitmap_snprintf(str, size, set);
  hwloc_bitmap_free(set);
  return res;
}

```

这段代码定义了一个名为 `hwloc_obj_type_sscanf` 的函数，它的功能是将一个类型字符串转换为对应类型及其属性的数据结构。这个函数的实现参照了 `hwloc_type_sscanf` 这个函数，但存在以下几个改动：

1. 删除了函数声明头中的 `__hwloc_attribute_deprecated`，因为该函数在 `hwloc_install_类别` 中被禁用。
2. 在函数内部，对 `hwloc_type_sscanf` 这个函数进行了多态封装，即如果传入的 `string` 不存在，函数会尝试使用 `hwloc_install_category` 函数尝试创建一个 `hwloc_obj_type_t` 类型的实例，并传入一个默认的 `hwloc_obj_type_t` 类型的参数。
3. 添加了一个 `err` 变量来记录 `hwloc_type_sscanf` 函数的错误，不过这个错误是在 `hwloc_obj_type_sscanf` 函数内部处理失败的，因此错误码可能有所不同。


```cpp
/** \brief Convert a type string into a type and some attributes.
 *
 * Deprecated by hwloc_type_sscanf()
 */
static __hwloc_inline int
hwloc_obj_type_sscanf(const char *string, hwloc_obj_type_t *typep, int *depthattrp, void *typeattrp, size_t typeattrsize) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_obj_type_sscanf(const char *string, hwloc_obj_type_t *typep, int *depthattrp, void *typeattrp, size_t typeattrsize)
{
  union hwloc_obj_attr_u attr;
  int err = hwloc_type_sscanf(string, typep, &attr, sizeof(attr));
  if (err < 0)
    return err;
  if (hwloc_obj_type_is_cache(*typep)) {
    if (depthattrp)
      *depthattrp = (int) attr.cache.depth;
    if (typeattrp && typeattrsize >= sizeof(hwloc_obj_cache_type_t))
      memcpy(typeattrp, &attr.cache.type, sizeof(hwloc_obj_cache_type_t));
  } else if (*typep == HWLOC_OBJ_GROUP) {
    if (depthattrp)
      *depthattrp = (int) attr.group.depth;
  }
  return 0;
}

```

hwloc_query_membind_policy(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags, int *result) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_query_membind_policy(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags, int *result)
{
 return hwloc_get_membind_policy(topology, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET, result);
}

在这段代码中，`hwloc_set_membind_nodeset`函数是一个位图上设置当前进程或线程的内存绑定策略的函数，它接受四个参数：

1. `topology`：要设置内存绑定的当前进程或线程的 `topology` 结构；
2. `nodeset`：要设置的节点的 `nodeset` 结构；
3. `policy`：内存绑定策略，可以是 `hwloc_membind_policy_t` 类型；
4. `flags`：设置的标志，包括 `HWLOC_MEMBIND_BYNODESET`，用于指示是否设置节点集的内存绑定策略；

函数返回值表示设置后的内存绑定策略。

`hwloc_query_membind_policy`函数是一个位图上查询当前进程或线程的内存绑定策略的函数，它与 `hwloc_set_membind_nodeset` 函数类似，但是是查询而非设置策略，返回值是一个整数，表示查询结果。

这两个函数共同构成了 `hwloc_topology_Membind` 类，用于在 NUMA 网络中管理进程和线程的内存绑定策略。


```cpp
/** \brief Set the default memory binding policy of the current
 * process or thread to prefer the NUMA node(s) specified by physical \p nodeset
 */
static __hwloc_inline int
hwloc_set_membind_nodeset(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_set_membind_nodeset(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  return hwloc_set_membind(topology, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief Query the default memory binding policy and physical locality of the
 * current process or thread.
 */
static __hwloc_inline int
```

这段代码定义了两个函数 hwloc_get_membind_nodeset 和 hwloc_set_proc_membind_nodeset，它们用于设置和获取内存绑定策略。

hwloc_get_membind_nodeset 函数接收三个参数：topology、nodeset 和 policy，然后返回一个整数，表示成功设置的节点数。这个函数使用了 hwloc_get_membind 函数，它会根据指定的 topology 和 nodeset 来设置内存绑定策略，并返回设置状态。如果设置成功，它将返回 0，否则返回一个负数。

hwloc_set_proc_membind_nodeset 函数与 hwloc_get_membind_nodeset 函数类似，但它的参数中还包含了一个 flags 标志，用于指示是否设置 NUMA 节点。这个函数接收四个参数：topology、pid、nodeset 和 policy，然后返回一个整数，表示成功设置的节点数。这个函数使用了 hwloc_set_proc_membind 函数，它会根据指定的 topology 和 nodeset 来设置内存绑定策略，并返回设置状态。如果设置成功，它将返回 0，否则返回一个负数。

这两个函数的原型非常相似，但它们的实现略有不同。在 hwloc_get_membind_nodeset 函数中，如果设置 nodeset 参数为 HWLOC_MEMBIND_BYNODESET 时，使用了 hwloc_MEMBIND_BYNODESET 作为旗帜，表示这是一个 bypod 策略，会将指定的 nodeset 中的所有节点都绑定到 NUMA 上。而在 hwloc_set_proc_membind_nodeset 函数中，使用了 hwloc_MEMBIND_SET_NODE 作为旗帜，表示这是一个 bynode 策略，会将指定的 nodeset 中的节点都绑定到 NUMA 上，但不会阻止子节点绑定到不同的 NUMA 上。


```cpp
hwloc_get_membind_nodeset(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_get_membind_nodeset(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  return hwloc_get_membind(topology, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief Set the default memory binding policy of the specified
 * process to prefer the NUMA node(s) specified by physical \p nodeset
 */
static __hwloc_inline int
hwloc_set_proc_membind_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_set_proc_membind_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  return hwloc_set_proc_membind(topology, pid, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

```

hwloc_bind_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
   int retval = hwloc_get_proc_membind_nodeset(topology, pid, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
   if (retval == -1) {
       return -1;
   }
   return 0;
}

```cpp

这段代码的作用是查询指定进程的默认内存绑定策略和物理定位，并返回一个整数表示查询结果。如果查询成功，函数将返回0或返回一个负数，具体取决于查询的内存绑定策略。如果查询失败，函数将返回-1。

具体来说，`hwloc_get_proc_membind_nodeset`函数接受4个参数：

- `topology`：要查询的topology类型。
- `pid`：要查询的进程ID。
- `nodeset`：要查询的节点集。
- `policy`：内存绑定策略，包括节点标记和内存标记。在这里，我们只关心节点标记，因此传递给函数的值是`hwloc_membind_policy_t`类型，其成员包括：

 - `HWLOC_MEMBIND_BIND_NODE`表示绑定内存对齐的节点，也就是内存对齐的节点集合。
 - `HWLOC_MEMBIND_BIND_CORROL`表示在内存对齐节点上执行的偏移量。
 - `HWLOC_MEMBIND_BIND_SYNC`表示同步内存对齐，即内存对齐的节点集合和内存标记的同步。
 - `HWLOC_MEMBIND_BIND_HWLOC`表示硬件内存布局。

函数首先调用`hwloc_get_proc_membind_nodeset`函数，获取查询的节点集合、内存绑定策略和初始内存对齐节点。然后，函数调用`hwloc_membind_policy_compare`函数，比较当前内存对齐策略和要查询的策略。如果当前策略与查询策略相等，则函数返回0，否则返回-1。

如果函数成功查询到节点集合，则函数返回0。否则，函数返回-1。


```
/** \brief Query the default memory binding policy and physical locality of the
 * specified process.
 */
static __hwloc_inline int
hwloc_get_proc_membind_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_get_proc_membind_nodeset(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  return hwloc_get_proc_membind(topology, pid, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief Bind the already-allocated memory identified by (addr, len)
 * to the NUMA node(s) in physical \p nodeset.
 */
static __hwloc_inline int
```cpp

这两段代码定义了两个函数，名为`hwloc_set_area_membind_nodeset`和`hwloc_get_area_membind_nodeset`，它们属于`hwloc_membind_nodeset_t`系列的函数。

它们的函数原型如下：
```c
hwloc_set_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags);
```cpp

```c
hwloc_get_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags);
```cpp

它们的实现基本上是相同的，只是在函数头和函数体中加入了一些注释。

这两个函数的主要作用是查询给定地址的物理NUMA节点和绑定策略，然后返回一个整数，表示成功查询的节点数和绑定策略。

具体来说，这两个函数接受一个`hwloc_topology_t`类型的顶先生成一个`hwloc_membind_nodeset_t`类型的变量，这个变量表示要查询的节点集，然后传给`hwloc_membind_policy_t`类型的参数，这个参数用于指定查询的节点绑定策略，包括节点绑定策略（`HWLOC_MEMBIND_BYNODESET`表示绑定到每个节点）和节点设置（`hwloc_const_nodeset_t`表示只查询节点，不设置节点）。

函数实现中，首先调用`hwloc_set_area_membind`函数，这个函数接受一个`hwloc_topology_t`类型和一个指向内存的指针`addr`和一个长度`len`，然后设置查询的节点绑定策略，并返回成功查询的节点数。

如果成功查询到节点，这两个函数就返回节点数和绑定策略，否则返回0。


```
hwloc_set_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_set_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  return hwloc_set_area_membind(topology, addr, len, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief Query the physical NUMA node(s) and binding policy of the memory
 * identified by (\p addr, \p len ).
 */
static __hwloc_inline int
hwloc_get_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags) __hwloc_attribute_deprecated;
static __hwloc_inline int
hwloc_get_area_membind_nodeset(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags)
{
  return hwloc_get_area_membind(topology, addr, len, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

```cpp

这段代码定义了两个函数 `hwloc_alloc_membind_nodeset` 和 `hwloc_alloc_membind_policy_nodeset`，它们在 `hwloc_topology_t` 类型中使用 `size_t` 表示输入参数 `len`，并使用 `hwloc_const_nodeset_t` 类型定义输入参数 `nodeset`。

这两个函数都使用了 `hwloc_membind_policy_t` 类型作为输入参数，并在函数内部使用了 `__hwloc_attribute_malloc` 和 `__hwloc_attribute_deprecated` 这两个 `hwloc_attribute_` 预处理指令。

具体来说，这两个函数的功能是：

1. `hwloc_alloc_membind_nodeset`：在指定的物理节点集上分配给定的节点集的内存，并返回分配的内存指针。如果成功，该函数将返回 `NULL`，否则返回分配的内存指针。
2. `hwloc_alloc_membind_policy_nodeset`：在指定的节点集上分配给定的策略的内存，并返回分配的内存指针。如果成功，该函数将返回 `NULL`，否则返回分配的内存指针。

注意，这两个函数都使用了 `hwloc_topology_t` 类型，这意味着它们只能在支持 `hwloc_topology_t` 的硬件设备上运行。


```
/** \brief Allocate some memory on the given physical nodeset \p nodeset
 */
static __hwloc_inline void *
hwloc_alloc_membind_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_malloc __hwloc_attribute_deprecated;
static __hwloc_inline void *
hwloc_alloc_membind_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  return hwloc_alloc_membind(topology, len, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief Allocate some memory on the given nodeset \p nodeset.
 */
static __hwloc_inline void *
hwloc_alloc_membind_policy_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_malloc __hwloc_attribute_deprecated;
static __hwloc_inline void *
```cpp

这段代码定义了一个名为 `hwloc_alloc_membind_policy_nodeset` 的函数，它的作用是接收一个硬件布局 topology、一个长度 len、一个节点集 nodeset 和一个 Membind Policy Policy，然后返回一个内存绑定策略 NUMA 节点集。

函数实现中，首先调用一个名为 `hwloc_alloc_membind_policy` 的函数，传递 topology、len、nodeset 和 Policy 参数，然后通过 `|` 运算符获取 Membind Policy 中与 HWLOC_MEMBIND_BYNODESET 相关的位，即 flags，最后将结果返回。

另外，函数中定义了一个名为 `hwloc_cpuset_to_nodeset_strict` 的函数，它的作用是将一个 CPU 集中的所有节点设置为指定的节点集，如果没有指定节点集，则直接返回。另一个名为 `hwloc_cpuset_to_nodeset_strict` 的函数与 `hwloc_cpuset_to_nodeset_strict` 类似，但是使用的是 `hwloc_topology_t` 而不是 `hwloc_cpuset_t` 作为参数，用于将 CPU 集中的节点设置为指定的节点集。


```
hwloc_alloc_membind_policy_nodeset(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags)
{
  return hwloc_alloc_membind_policy(topology, len, nodeset, policy, flags | HWLOC_MEMBIND_BYNODESET);
}

/** \brief Convert a CPU set into a NUMA node set and handle non-NUMA cases
 */
static __hwloc_inline void
hwloc_cpuset_to_nodeset_strict(hwloc_topology_t topology, hwloc_const_cpuset_t _cpuset, hwloc_nodeset_t nodeset) __hwloc_attribute_deprecated;
static __hwloc_inline void
hwloc_cpuset_to_nodeset_strict(hwloc_topology_t topology, hwloc_const_cpuset_t _cpuset, hwloc_nodeset_t nodeset)
{
  hwloc_cpuset_to_nodeset(topology, _cpuset, nodeset);
}

```cpp

这段代码定义了一个名为 `hwloc_cpuset_from_nodeset_strict` 的函数，它接受一个 `hwloc_topology_t` 类型的输入，一个 `hwloc_cpuset_t` 类型的输出，和一个 `hwloc_const_nodeset_t` 类型的输入。

函数实现了一个从给定的 `hwloc_const_nodeset_t` 类型的节点集中提取出 `hwloc_cpuset_t` 类型的输出，然后将其输入到 `hwloc_cpuset_from_nodeset_strict` 函数中。

具体实现包括两个步骤：

1. 首先，函数调用 `hwloc_cpuset_from_nodeset` 函数，将输入的 `hwloc_const_nodeset_t` 类型的节点集作为第一个参数传入，并将第二个参数留空。这个函数的作用是，将指定的节点集转换为相应的 `hwloc_cpuset_t` 类型的数据结构。

2. 接下来，函数再次调用 `hwloc_cpuset_from_nodeset` 函数，将上一步获得的 `hwloc_const_nodeset_t` 类型的节点集作为第一个参数传入，并将第二个参数设置为给定的 `hwloc_cpuset_t` 类型的数据结构。这个函数的作用是，将指定的节点集转换为相应的 `hwloc_cpuset_t` 类型的数据结构。

最后，函数使用了 `__hwloc_attribute_deprecated` 修饰符，这意味着这个函数在之后的版本中可能会被删除，并且不应该在新的代码中使用。


```
/** \brief Convert a NUMA node set into a CPU set and handle non-NUMA cases
 */
static __hwloc_inline void
hwloc_cpuset_from_nodeset_strict(hwloc_topology_t topology, hwloc_cpuset_t _cpuset, hwloc_const_nodeset_t nodeset) __hwloc_attribute_deprecated;
static __hwloc_inline void
hwloc_cpuset_from_nodeset_strict(hwloc_topology_t topology, hwloc_cpuset_t _cpuset, hwloc_const_nodeset_t nodeset)
{
  hwloc_cpuset_from_nodeset(topology, _cpuset, nodeset);
}


#ifdef __cplusplus
} /* extern "C" */
#endif


```cpp

这段代码是一个预处理指令，它用来检查一个文件中是否定义了名为 "HWLOC_DEPRECATED_H" 的头文件。如果这个头文件已经被定义了，那么预处理器会编译掉代码中包含这个头文件的部分，否则不会被编译。

这个头文件可能是一个通过 include 语句导入的外部头文件，也可能是定义在系统头文件库中的头文件。通过 include 语句导入的外部头文件，这个头文件可能会在当前源文件中被重新定义，而通过系统头文件库中的头文件，则可能已经被缓存到操作系统中，不会重复下载。


```
#endif /* HWLOC_DEPRECATED_H */

```cpp

# `src/3rdparty/hwloc/include/hwloc/diff.h`

这段代码定义了一个名为`hwloc_diff.h`的文件，其中定义了一个名为`hwloc_diff`的函数。函数的说明中包含了一行引用自`Inria`公司的版权声明，表明该函数中包含某些受版权保护的代码，需要遵守相应的授权规定。

接下来是函数体，其中定义了一个`__attribute__((weak))`的`hwloc_diff`函数，该函数的具体实现未在函数体中展开。根据该函数名称和`weak`属性，可以推测该函数可能是一个弱函数，即可能会因为类型检查失败而导致程序崩溃，因此它被声明为弱函数，以便在运行时能够尽可能地保证程序的稳定性。

最后，函数定义了一个`hwloc_diff`函数，该函数没有参数，返回类型为`void`。未定义该函数的具体实现，因此无法提供该函数的实际行为，只能通过`weak`属性的提示来告知程序员该函数可能会因为类型检查失败而导致程序崩溃。


```
/*
 * Copyright © 2013-2020 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Topology differences.
 */

#ifndef HWLOC_DIFF_H
#define HWLOC_DIFF_H

#ifndef HWLOC_H
#error Please include the main hwloc.h instead
#endif


```cpp

这段代码是一个C语言的预处理指令，它通过#ifdef和#endif来定义了一个名为`__cplusplus`的预处理目录。在这个目录下，可以定义其他源文件。

接下来的代码是一个以`extern`开头的C语言源文件，通过`#elif`和`#endif`来定义了一些关于这个源文件条件的判断。如果条件为真（即0），则不编译这个源文件；否则，编译并输出。

在`extern "C"`这一行中，定义了一些C语言的函数和变量，应该在编译时被正确地链接到。

接下来的代码是一个以`/**`开头的函数声明，定义了一个名为`hwlocality_diff`的函数。

在函数体中，定义了一些关于`Topology differences`的说明。这个函数可能会接受一个整数类型的参数，表示要比较的topology的数量。这个函数的作用是帮助用户将一个topology（也就是一个节点集）的不同之处和另一个topology（也就是一个更简单的topology，可能只有一个节点或者没有节点）进行比较，然后将它们存储为一个xml文件。

在`#ifdef`和`#elif`之间的部分是一个嵌套的`#include`，通过`__cplusplus`预处理指令将一些C语言的函数和变量定义为extern。这个`__cplusplus`指令的作用是在预处理被定义为`__cplusplus`的源文件时，包含这些定义。


```
#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif


/** \defgroup hwlocality_diff Topology differences
 *
 * Applications that manipulate many similar topologies, for instance
 * one for each node of a homogeneous cluster, may want to compress
 * topologies to reduce the memory footprint.
 *
 * This file offers a way to manipulate the difference between topologies
 * and export/import it to/from XML.
 * Compression may therefore be achieved by storing one topology
 * entirely while the others are only described by their differences
 * with the former.
 * The actual topology can be reconstructed when actually needed by
 * applying the precomputed difference to the reference topology.
 *
 * This interface targets very similar nodes.
 * Only very simple differences between topologies are actually
 * supported, for instance a change in the memory size, the name
 * of the object, or some info attribute.
 * More complex differences such as adding or removing objects cannot
 * be represented in the difference structures and therefore return
 * errors.
 * Differences between object sets or topology-wide allowed sets,
 * cannot be represented either.
 *
 * It means that there is no need to apply the difference when
 * looking at the tree organization (how many levels, how many
 * objects per level, what kind of objects, CPU and node sets, etc)
 * and when binding to objects.
 * However the difference must be applied when looking at object
 * attributes such as the name, the memory size or info attributes.
 *
 * @{
 */


```cpp

这段代码定义了一个枚举类型 hwloc_topology_diff_obj_attr_type_e，用于表示在 hwloc_topology_diff_obj 属性的不同类型。

枚举类型 hwloc_topology_diff_obj_attr_type_e 定义了三种不同的属性类型：

1. HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE，表示当属性的值属于这个类型时，会在对象 local memory 上修改。这个类型的属性通常用于描述在多线程环境中如何同步数据。

2. HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME，表示当属性的值属于这个类型时，会修改对象的名字。这个类型的属性通常用于描述在命令行界面中如何设置对象的唯一标识。

3. HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO，表示当属性的值属于这个类型时，会修改信息属性（如内存使用情况，错误消息等）。这个类型的属性通常用于描述在分布式系统中如何收集和传递性能信息。

整段代码定义了一个枚举类型 hwloc_topology_diff_obj_attr_type_t，用于表示在 hwloc_topology_diff_obj 属性的不同类型。这些枚举类型的值可以用于在程序中选择合适的属性进行不同的操作。例如，在选择同步对象的属性时，可以选择使用 HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE 类型的属性，而不需要关心同步的具体细节。


```
/** \brief Type of one object attribute difference.
 */
typedef enum hwloc_topology_diff_obj_attr_type_e {
  /** \brief The object local memory is modified.
   * The union is a hwloc_topology_diff_obj_attr_u::hwloc_topology_diff_obj_attr_uint64_s
   * (and the index field is ignored).
   */
  HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE,

  /** \brief The object name is modified.
   * The union is a hwloc_topology_diff_obj_attr_u::hwloc_topology_diff_obj_attr_string_s
   * (and the name field is ignored).
   */

  HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME,
  /** \brief the value of an info attribute is modified.
   * The union is a hwloc_topology_diff_obj_attr_u::hwloc_topology_diff_obj_attr_string_s.
   */
  HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO
} hwloc_topology_diff_obj_attr_type_t;

```cpp

这是一个描述不同HWLOC配置对象的属性的联合类型的代码。这个代码定义了三个结构体：hwloc_topology_diff_obj_attr_generic_s，hwloc_topology_diff_obj_attr_uint64_s和hwloc_topology_diff_obj_attr_string_s。

其中，hwloc_topology_diff_obj_attr_generic_s结构体表示通用属性，它有两个成员，一个是类型成员hwloc_topology_diff_obj_attr_type_t，另一个是该属性的默认值会员成员hwloc_topology_diff_obj_attr_default_value_t，它可以在联合中使用。

hwloc_topology_diff_obj_attr_uint64_s结构体表示一个整数属性，它有两个成员，一个是类型成员hwloc_topology_diff_obj_attr_type_t，另一个是成员的索引成员hwloc_uint64_s，用于存储整数属性。它的成员是hwloc_topology_diff_obj_attr_type_t和hwloc_uint64_s，分别表示属性的类型和存储整数值的成员。

hwloc_topology_diff_obj_attr_string_s结构体表示一个字符串属性，它有两个成员，一个是类型成员hwloc_topology_diff_obj_attr_type_t，另一个是成员的名称成员char *和一个用于存储新旧值的字符串成员char *。它的成员是hwloc_topology_diff_obj_attr_type_t和char *和char *，分别表示属性的类型和存储新旧值的成员。

这个代码定义了三个联合类型，每个联合类型都有自己的成员，用于存储配置对象的属性。这些属性可以用来设置或修改配置对象的属性。


```
/** \brief One object attribute difference.
 */
union hwloc_topology_diff_obj_attr_u {
  struct hwloc_topology_diff_obj_attr_generic_s {
    /* each part of the union must start with these */
    hwloc_topology_diff_obj_attr_type_t type;
  } generic;

  /** \brief Integer attribute modification with an optional index. */
  struct hwloc_topology_diff_obj_attr_uint64_s {
    /* used for storing integer attributes */
    hwloc_topology_diff_obj_attr_type_t type;
    hwloc_uint64_t index; /* not used for SIZE */
    hwloc_uint64_t oldvalue;
    hwloc_uint64_t newvalue;
  } uint64;

  /** \brief String attribute modification with an optional name */
  struct hwloc_topology_diff_obj_attr_string_s {
    /* used for storing name and info pairs */
    hwloc_topology_diff_obj_attr_type_t type;
    char *name; /* not used for NAME */
    char *oldvalue;
    char *newvalue;
  } string;
};


```cpp

这段代码定义了一个名为hwloc_topology_diff_type_t的枚举类型，用于指定一个二元组中不同元素之间的差异。

枚举类型hwloc_topology_diff_type_t中包含两个成员变量，分别为HWLOC_TOPOLOGY_DIFF_OBJ_ATTR和HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX。其中，HWLOC_TOPOLOGY_DIFF_OBJ_ATTR表示一个二元组中的一个元素被更改了，而HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX则表示另一个元素也发生了更改，但是这两个更改之间存在一个过于复杂的差异，无法通过hwloc_topology_diff_build()函数进行检查。

该枚举类型被用于描述二元组中不同元素之间的差异，可以帮助开发人员更好地理解代码中可能存在的差异情况。


```
/** \brief Type of one element of a difference list.
 */
typedef enum hwloc_topology_diff_type_e {
  /** \brief An object attribute was changed.
   * The union is a hwloc_topology_diff_u::hwloc_topology_diff_obj_attr_s.
   */
  HWLOC_TOPOLOGY_DIFF_OBJ_ATTR,

  /** \brief The difference is too complex,
   * it cannot be represented. The difference below
   * this object has not been checked.
   * hwloc_topology_diff_build() will return 1.
   *
   * The union is a hwloc_topology_diff_u::hwloc_topology_diff_too_complex_s.
   */
  HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX
} hwloc_topology_diff_type_t;

```cpp

这段代码定义了一个名为`hwloc_topology_diff_t`的指针变量，它表示两个顶级理论之间的差异列表中的一个元素。

通过观察可以发现，该变量有以下几种可能的情况：

1. `type`为`HWLOC_TOPOLOGY_DIFF_OBJ_ATTR`，表示这是一个对象属性的差异。这种情况下的差异列表中，可能包含一个或多个`next`指针，用于指向下一个属性差异的位置。
2. `type`为`HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX`，表示这是一个过于复杂的差异。在这种情况下，差异列表可能包含一个指向下一个差异计算位置的指针`next`，用于继续计算该差异。

总结来说，该代码定义了一个用于表示两个顶级理论之间的差异列表的指针变量。通过该变量，可以获取差异列表中的各种信息，如差异类型、对象属性差异、复杂差异等。


```
/** \brief One element of a difference list between two topologies.
 */
typedef union hwloc_topology_diff_u {
  struct hwloc_topology_diff_generic_s {
    /* each part of the union must start with these */
    hwloc_topology_diff_type_t type;
    union hwloc_topology_diff_u * next; /* pointer to the next element of the list, or NULL */
  } generic;

  /* A difference in an object attribute. */
  struct hwloc_topology_diff_obj_attr_s {
    hwloc_topology_diff_type_t type; /* must be ::HWLOC_TOPOLOGY_DIFF_OBJ_ATTR */
    union hwloc_topology_diff_u * next;
    /* List of attribute differences for a single object */
    int obj_depth;
    unsigned obj_index;
    union hwloc_topology_diff_obj_attr_u diff;
  } obj_attr;

  /* A difference that is too complex. */
  struct hwloc_topology_diff_too_complex_s {
    hwloc_topology_diff_type_t type; /* must be ::HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX */
    union hwloc_topology_diff_u * next;
    /* Where we had to stop computing the diff in the first topology */
    int obj_depth;
    unsigned obj_index;
  } too_complex;
} * hwloc_topology_diff_t;


```cpp

这段代码定义了一个名为`hwloc_topology_diff_diff`的函数，用于计算两个拓扑之间的差异。差异被存储为`hwloc_topology_diff_diff_t`列表，该列表的每个条目代表了一个差异条目，从`diff`开始。该函数通过同时遍历两个拓扑树来实现深度优先遍历，并在遍历过程中计算差异。如果两个对象的差异过于复杂，无法以适当的方式表示，函数将返回一个名为`::hwloc_topology_diff_diff_to_complex`的特殊差异条目，该条目将导致函数退出计算。如果函数在计算过程中遇到其他错误，例如输出目录未指定，函数将返回-1。

该函数的实现旨在为开发人员提供一种计算两个拓扑之间的差异的方法，并确保只有足够简单的差异才能得到正确计算。该函数的输出是0到1之间的整数，表示计算出的差异复杂程度。如果差异过于复杂，函数将返回一个名为`::hwloc_topology_diff_diff_to_complex`的特殊差异条目，该条目将导致函数退出计算。


```
/** \brief Compute the difference between 2 topologies.
 *
 * The difference is stored as a list of ::hwloc_topology_diff_t entries
 * starting at \p diff.
 * It is computed by doing a depth-first traversal of both topology trees
 * simultaneously.
 *
 * If the difference between 2 objects is too complex to be represented
 * (for instance if some objects have different types, or different numbers
 * of children), a special diff entry of type ::HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX
 * is queued.
 * The computation of the diff does not continue below these objects.
 * So each such diff entry means that the difference between two subtrees
 * could not be computed.
 *
 * \return 0 if the difference can be represented properly.
 *
 * \return 0 with \p diff pointing to NULL if there is no difference
 * between the topologies.
 *
 * \return 1 if the difference is too complex (see above). Some entries in
 * the list will be of type ::HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX.
 *
 * \return -1 on any other error.
 *
 * \note \p flags is currently not used. It should be 0.
 *
 * \note The output diff has to be freed with hwloc_topology_diff_destroy().
 *
 * \note The output diff can only be exported to XML or passed to
 * hwloc_topology_diff_apply() if 0 was returned, i.e. if no entry of type
 * ::HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX is listed.
 *
 * \note The output diff may be modified by removing some entries from
 * the list. The removed entries should be freed by passing them to
 * to hwloc_topology_diff_destroy() (possible as another list).
```cpp

这段代码定义了一个名为 `hwloc_topology_diff_build` 的函数，其作用是接收两个已有的 `hwloc_topology_t` 类型的数据结构 `topology` 和 `newtopology`，以及一些传递给它的标志 `flags`，然后返回一个新的 `hwloc_topology_diff_t` 类型的数据结构 `diff`，表示 topology 之间的差异。

该函数的实现包括以下几个步骤：

1. 定义了一个名为 `HWLOC_DECLSPEC` 的类型，用于声明该函数。

2. 定义了一个名为 `enum hwloc_topology_diff_apply_flags_e` 的枚举类型，用于定义 `HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE` 等常量。

3. 定义了一个名为 `apply_topology_diff` 的函数，接收一个 `hwloc_topology_diff_t` 类型的数据结构 `diff`，将其应用到指定的 topology 中，并将 `topology` 和 `newtopology` 作为参数传入。

4. 在 `hwloc_topology_diff_apply` 函数中，如果 `diff` 的大小超出了 `topology` 和 `newtopology` 所能支持的最大差分数量，那么就需要将所有之前应用的元素取消，并返回 `-N`。

5. 最后，调用 `apply_topology_diff` 函数，并将传递给该函数的标志 `flags` 作为参数传入，该函数将根据传递的标志尝试应用差异，并返回其结果。


```
*/
HWLOC_DECLSPEC int hwloc_topology_diff_build(hwloc_topology_t topology, hwloc_topology_t newtopology, unsigned long flags, hwloc_topology_diff_t *diff);

/** \brief Flags to be given to hwloc_topology_diff_apply().
 */
enum hwloc_topology_diff_apply_flags_e {
  /** \brief Apply topology diff in reverse direction.
   * \hideinitializer
   */
  HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE = (1UL<<0)
};

/** \brief Apply a topology diff to an existing topology.
 *
 * \p flags is an OR'ed set of ::hwloc_topology_diff_apply_flags_e.
 *
 * The new topology is modified in place. hwloc_topology_dup()
 * may be used to duplicate it before patching.
 *
 * If the difference cannot be applied entirely, all previous applied
 * elements are unapplied before returning.
 *
 * \return 0 on success.
 *
 * \return -N if applying the difference failed while trying
 * to apply the N-th part of the difference. For instance -1
 * is returned if the very first difference element could not
 * be applied.
 */
```cpp



这两函数定义在以 `hwloc_topology_diff_apply` 和 `hwloc_topology_diff_destroy` 为前缀的函数中。它们的主要作用是管理一个 `hwloc_topology_diff_t` 类型的数据结构列表。

具体来说，这两函数分别负责以下操作：

1. `hwloc_topology_diff_apply` 函数：该函数接收两个参数：一个 `hwloc_topology_t` 类型的顶类和一个 `hwloc_topology_diff_t` 类型的数据结构列表。该函数的主要作用是使用 `hwloc_topology_diff_t` 类型的数据结构列表对传入的 `hwloc_topology_t` 顶类进行差异比较，并将结果保存回 `hwloc_topology_diff_t` 类型的数据结构列表中。该函数有一个无返回值的类型，但函数内部使用了 `unsigned long flags` 参数，这意味着该函数将尝试将所有传递给它的 `unsigned long` 类型的标志设置为 0，以便在函数内部进行更多的操作。

2. `hwloc_topology_diff_destroy` 函数：该函数接收一个 `hwloc_topology_diff_t` 类型的数据结构列表作为参数。该函数的主要作用是将接收到的 `hwloc_topology_diff_t` 类型的数据结构列表销毁，并返回一个指向销毁的差异列表的指针。

由于这两个函数都使用了 `hwloc_topology_diff_t` 类型的数据结构列表，因此可以推断出这两个函数的主要作用是与 `hwloc_topology_diff_t` 类型的数据结构列表进行交互，而不是与具体的顶类相关。


```
HWLOC_DECLSPEC int hwloc_topology_diff_apply(hwloc_topology_t topology, hwloc_topology_diff_t diff, unsigned long flags);

/** \brief Destroy a list of topology differences.
 */
HWLOC_DECLSPEC int hwloc_topology_diff_destroy(hwloc_topology_diff_t diff);

/** \brief Load a list of topology differences from a XML file.
 *
 * If not \c NULL, \p refname will be filled with the identifier
 * string of the reference topology for the difference file,
 * if any was specified in the XML file.
 * This identifier is usually the name of the other XML file
 * that contains the reference topology.
 *
 * \note the pointer returned in refname should later be freed
 * by the caller.
 */
```cpp

这段代码定义了两个函数，用于从XML文件中导出拓扑差异列表和从XML文件中加载拓扑差异列表。

函数1：`hwloc_topology_diff_export_xml`，参数为差异图结构体`hwloc_topology_diff_t`，返回值为导出到XML文件的ID，如果XML文件不存在，返回字符指针`NULL`。结构体`hwloc_topology_diff_t`存储了计算拓扑差异所需的全部信息。

函数2：`hwloc_topology_diff_diff_load_xml`，参数为XML文件路径、差异图结构体指针`hwloc_topology_diff_t`和参考名称，返回值为差异图结构体指针。这个函数从XML文件中读取拓扑差异，并将它们存储在`hwloc_topology_diff_t`结构体中，同时将参考名称存储在`diff`参数中，以便在导出到XML文件时指定。


```
HWLOC_DECLSPEC int hwloc_topology_diff_load_xml(const char *xmlpath, hwloc_topology_diff_t *diff, char **refname);

/** \brief Export a list of topology differences to a XML file.
 *
 * If not \c NULL, \p refname defines an identifier string
 * for the reference topology which was used as a base when
 * computing this difference.
 * This identifier is usually the name of the other XML file
 * that contains the reference topology.
 * This attribute is given back when reading the diff from XML.
 */
HWLOC_DECLSPEC int hwloc_topology_diff_export_xml(hwloc_topology_diff_t diff, const char *refname, const char *xmlpath);

/** \brief Load a list of topology differences from a XML buffer.
 *
 * If not \c NULL, \p refname will be filled with the identifier
 * string of the reference topology for the difference file,
 * if any was specified in the XML file.
 * This identifier is usually the name of the other XML file
 * that contains the reference topology.
 *
 * \note the pointer returned in refname should later be freed
 * by the caller.
  */
```cpp

这段代码定义了一个名为 `hwloc_topology_diff_load_xmlbuffer` 的函数，它的作用是将从 XML 文件中读取的 topology 差异列表 export 到一个 XML 缓冲区中，并返回该 XML 缓冲区的指针。

函数的第一个参数是一个指向 XML 文件的指针 `xmlbuffer`，第二个参数是一个整数 `buflen`，用于指定生成的 XML 缓冲区的大小，第三个参数是一个指向 `hwloc_topology_diff_t` 类型的指针 `diff`，用于保存计算得到的 topology 差异列表，最后一个参数是一个指向字符串的指针 `refname`，用于指定输出文件中参考 topology 的名称。

函数首先从 `xmlbuffer` 指向的 XML 文件中读取一个长度为 `buflen` 的 topology 差异列表，然后将该列表 export 到一个以 `diff` 指向的 `hwloc_topology_diff_t` 类型的指针中，并将该列表的长度设置为 `buflen`，同时将 `diff` 的值置为 `buflen`。

最后，函数将生成的 XML 缓冲区的最后一个字符设置为 `0`，以使其长度与生成的 XML 文件中指定的字符串的长度相等，然后返回该 XML 缓冲区的指针。需要注意的是，由于 XML 文件中的内容是动态增长的，因此需要在调用 `hwloc_free_xmlbuffer` 函数时释放掉 `xmlbuffer`，以避免内存泄漏。


```
HWLOC_DECLSPEC int hwloc_topology_diff_load_xmlbuffer(const char *xmlbuffer, int buflen, hwloc_topology_diff_t *diff, char **refname);

/** \brief Export a list of topology differences to a XML buffer.
 *
 * If not \c NULL, \p refname defines an identifier string
 * for the reference topology which was used as a base when
 * computing this difference.
 * This identifier is usually the name of the other XML file
 * that contains the reference topology.
 * This attribute is given back when reading the diff from XML.
 *
 * The returned buffer ends with a \0 that is included in the returned
 * length.
 *
 * \note The XML buffer should later be freed with hwloc_free_xmlbuffer().
 */
```cpp

这段代码定义了一个名为 `hwloc_topology_diff_export_xmlbuffer` 的函数，属于 `hwloc_topology_diff_export_xmlbuffer` 函数家族。它接受两个参数，一个是 `hwloc_topology_diff_t` 类型的差异信息 `diff`，另一个是字符串引用 `refname`，以及两个指向字符数组的指针 `xmlbuffer` 和 `buflen`，分别表示存储差异信息和目标字符数组的大小。

函数的作用是将 `diff` 和 `refname` 提供的信息将到 `xmlbuffer` 指向的字符数组中，并返回该数组长度。它将被存储的字符串转换为xml格式并返回。


```
HWLOC_DECLSPEC int hwloc_topology_diff_export_xmlbuffer(hwloc_topology_diff_t diff, const char *refname, char **xmlbuffer, int *buflen);

/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_DIFF_H */

```cpp

# `src/3rdparty/hwloc/include/hwloc/distances.h`

这段代码是一个C++文件头，它定义了一个名为`HWLOC_DISTANCES_H`的文件。该文件包含了关于`HWLOC_DISTANCES`的说明。

`HWLOC_DISTANCES`是一个C++类，它提供了一种计算三维物体距离的方法。它需要通过`HWLOC_H`头文件中定义的`main`函数才能被使用。

因此，这个文件头定义了一个名为`HWLOC_DISTANCES`的类，它用于计算三维物体距离。


```
/*
 * Copyright © 2010-2022 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Object distances.
 */

#ifndef HWLOC_DISTANCES_H
#define HWLOC_DISTANCES_H

#ifndef HWLOC_H
#error Please include the main hwloc.h instead
#endif


```cpp

这段代码是一个C语言的预处理指令，用于定义一个名为"NUMALatency"的距离矩阵。这个矩阵包含了HWLOC_DISTANCES_KIND_FROM_OS或HWLOC_DISTANCES_KIND_FROM_USER类型的距离。它主要用于计算访问操作系统内存时从另一个节点核心之间的延迟。这个距离矩阵可能包含延迟信息，也可能包含用户定义的带宽，这些带宽可能是随机设置的。

具体来说，这段代码的作用是定义一个用于计算从内存中的一个对象到另一个对象之间的延迟的矩阵。这个矩阵的每个元素都被存储在一个指向对象或带宽的指针中，指针可以被修改但不能被重新分配、释放等。在使用时，指针的修改必须被谨慎操作，以避免对内存中的对象或带宽的错误访问或修改。


```
#ifdef __cplusplus
extern "C" {
#elif 0
}
#endif


/** \defgroup hwlocality_distances_get Retrieve distances between objects
 * @{
 */

/** \brief Matrix of distances between a set of objects.
 *
 * This matrix often contains latencies between NUMA nodes
 * (as reported in the System Locality Distance Information Table (SLIT)
 * in the ACPI specification), which may or may not be physically accurate.
 * It corresponds to the latency for accessing the memory of one node
 * from a core in another node.
 * The corresponding kind is ::HWLOC_DISTANCES_KIND_FROM_OS | ::HWLOC_DISTANCES_KIND_FROM_USER.
 * The name of this distances structure is "NUMALatency".
 * Others distance structures include and "XGMIBandwidth", "XGMIHops",
 * "XeLinkBandwidth" and "NVLinkBandwidth".
 *
 * The matrix may also contain bandwidths between random sets of objects,
 * possibly provided by the user, as specified in the \p kind attribute.
 *
 * Pointers \p objs and \p values should not be replaced, reallocated, freed, etc.
 * However callers are allowed to modify \p kind as well as the contents
 * of \p objs and \p values arrays.
 * For instance, if there is a single NUMA node per Package,
 * hwloc_get_obj_with_same_locality() may be used to convert between them
 * and replace NUMA nodes in the \p objs array with the corresponding Packages.
 * See also hwloc_distances_transform() for applying some transformations
 * to the structure.
 */
```cpp

这是一个结构体 `hwloc_distances_s`，表示距离矩阵中对象的描述。

该结构体包含以下成员：

1. `nbobjs`：表示距离矩阵中描述的对象的数量。

2. `objs`：是一个指向距离矩阵中对象的指针数组，这些对象没有特定的顺序，可以通过 `hwloc_distances_obj_index()` 和 `hwloc_distances_obj_pair_values()` 进行查找。

3. `kind`：是一个逻辑 OR 集合，包含了 `hwloc_distances_kind_e` 中定义的所有值。

4. `values`：是一个存储距离矩阵的单维数组，其中的元素存储了每个对象之间的距离。每个元素通过 `i` 和 `j` 计算得出，根据 `kind` 属性中的值，这些距离具有不同的含义。

该结构体的定义为离散的对象描述，通过一个距离矩阵描述了多个对象之间的距离。


```
struct hwloc_distances_s {
  unsigned nbobjs;		/**< \brief Number of objects described by the distance matrix. */
  hwloc_obj_t *objs;		/**< \brief Array of objects described by the distance matrix.
				 * These objects are not in any particular order,
				 * see hwloc_distances_obj_index() and hwloc_distances_obj_pair_values()
				 * for easy ways to find objects in this array and their corresponding values.
				 */
  unsigned long kind;		/**< \brief OR'ed set of ::hwloc_distances_kind_e. */
  hwloc_uint64_t *values;	/**< \brief Matrix of distances between objects, stored as a one-dimension array.
				 *
				 * Distance from i-th to j-th object is stored in slot i*nbobjs+j.
				 * The meaning of the value depends on the \p kind attribute.
				 */
};

```cpp

Yes, you are correct. The `hwloc_distances_kind_e` enum defines several possible values for the `HWLOC_DISTANCES_KIND_FROM_*` format, including `HWLOC_DISTANCES_KIND_FROM_OS` which indicates that the distance information comes from the operating system, `HWLOC_DISTANCES_KIND_FROM_USER` which indicates that the user provided the distance information, `HWLOC_DISTANCES_KIND_MEANS_LATENCY` which indicates that the distance values are similar to latencies between objects, and `HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH` which indicates that the distance values are similar to bandwidths between objects.


```
/** \brief Kinds of distance matrices.
 *
 * The \p kind attribute of struct hwloc_distances_s is a OR'ed set
 * of kinds.
 *
 * A kind of format HWLOC_DISTANCES_KIND_FROM_* specifies where the
 * distance information comes from, if known.
 *
 * A kind of format HWLOC_DISTANCES_KIND_MEANS_* specifies whether
 * values are latencies or bandwidths, if applicable.
 */
enum hwloc_distances_kind_e {
  /** \brief These distances were obtained from the operating system or hardware.
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_FROM_OS = (1UL<<0),
  /** \brief These distances were provided by the user.
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_FROM_USER = (1UL<<1),

  /** \brief Distance values are similar to latencies between objects.
   * Values are smaller for closer objects, hence minimal on the diagonal
   * of the matrix (distance between an object and itself).
   * It could also be the number of network hops between objects, etc.
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_MEANS_LATENCY = (1UL<<2),
  /** \brief Distance values are similar to bandwidths between objects.
   * Values are higher for closer objects, hence maximal on the diagonal
   * of the matrix (distance between an object and itself).
   * Such values are currently ignored for distance-based grouping.
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH = (1UL<<3),

  /** \brief This distances structure covers objects of different types.
   * This may apply to the "NVLinkBandwidth" structure in presence
   * of a NVSwitch or POWER processor NVLink port.
   * \hideinitializer
   */
  HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES = (1UL<<4)
};

```cpp

这段代码定义了一个名为 `retrieve_distance_matrices` 的函数，它的作用是检索距离矩阵并将其存储在 `distances` 数组中。以下是该函数的功能描述：

1. 函数接收一个名为 `flags` 的参数，它是一个二进制掩码，目前未被使用。
2. 函数接收一个名为 `kind` 的参数，它是一个整数，用于指定从哪个源获取距离矩阵。有以下可能的值：

	* 0：返回所有距离矩阵。
	* HWLOC_DISTANCES_KIND_FROM_*：返回符合条件的所有距离矩阵。
	* HWLOC_DISTANCES_KIND_MEANS_*：返回符合条件的所有距离矩阵。
3. 函数接收一个名为 `nr` 的参数，用于指定可以存储的最大距离矩阵数。
4. 函数返回一个名为 `nr` 的整数，表示实际找到的距离矩阵数，即使有一些距离矩阵无法存储，该函数仍然返回成功（值为0）。
5. 函数调用者可以比较函数返回值和调用前后的值来确认函数是否成功。
6. 函数每个返回的距离矩阵都需要由调用者释放。


```
/** \brief Retrieve distance matrices.
 *
 * Retrieve distance matrices from the topology into the \p distances array.
 *
 * \p flags is currently unused, should be \c 0.
 *
 * \p kind serves as a filter. If \c 0, all distance matrices are returned.
 * If it contains some HWLOC_DISTANCES_KIND_FROM_*, only distance matrices
 * whose kind matches one of these are returned.
 * If it contains some HWLOC_DISTANCES_KIND_MEANS_*, only distance matrices
 * whose kind matches one of these are returned.
 *
 * On input, \p nr points to the number of distance matrices that may be stored
 * in \p distances.
 * On output, \p nr points to the number of distance matrices that were actually
 * found, even if some of them couldn't be stored in \p distances.
 * Distance matrices that couldn't be stored are ignored, but the function still
 * returns success (\c 0). The caller may find out by comparing the value pointed
 * by \p nr before and after the function call.
 *
 * Each distance matrix returned in the \p distances array should be released
 * by the caller using hwloc_distances_release().
 */
```cpp

这两段代码是用于在给定的topology中获取距离矩阵的函数。

`hwloc_distances_get`函数的参数为：

* `topology`：要获取距离的topology结构，它的实现是通过`hwloc_topology_print`函数打印出来的。这个结构可能包括了所有深度为`depth`的线程bar。
* `nr`：用于存储距离的元素数量。
* `distances`：指向`hwloc_distances_s`类型的指针，用于存储每个线程bar的距离。
* `kind`：要检索的距离类型，可能是`HWLOC_DIST_KIND_SET`、`HWLOC_DIST_KIND_CPU`或`HWLOC_DIST_KIND_GPU`。
* `flags`：用于指定检索指定深度的距离，可能是0或`HWLOC_DIST_FLAG_COalesced`。

`hwloc_distances_get_by_depth`函数的参数为：

* `topology`：要获取距离的topology结构，它的实现是通过`hwloc_topology_print`函数打印出来的。这个结构可能包括了所有深度为`depth`的线程bar。
* `depth`：要检索的深度。
* `nr`：用于存储距离的元素数量。
* `distances`：指向`hwloc_distances_s`类型的指针，用于存储每个线程bar的距离。
* `kind`：要检索的距离类型，可能是`HWLOC_DIST_KIND_SET`、`HWLOC_DIST_KIND_CPU`或`HWLOC_DIST_KIND_GPU`。
* `flags`：用于指定检索指定深度的距离，可能是0或`HWLOC_DIST_FLAG_COalesced`。


```
HWLOC_DECLSPEC int
hwloc_distances_get(hwloc_topology_t topology,
		    unsigned *nr, struct hwloc_distances_s **distances,
		    unsigned long kind, unsigned long flags);

/** \brief Retrieve distance matrices for object at a specific depth in the topology.
 *
 * Identical to hwloc_distances_get() with the additional \p depth filter.
 */
HWLOC_DECLSPEC int
hwloc_distances_get_by_depth(hwloc_topology_t topology, int depth,
			     unsigned *nr, struct hwloc_distances_s **distances,
			     unsigned long kind, unsigned long flags);

/** \brief Retrieve distance matrices for object of a specific type.
 *
 * Identical to hwloc_distances_get() with the additional \p type filter.
 */
```cpp

这两段代码是用于获取“NUMALatency”距离矩阵的函数。`hwloc_distances_get_by_type`函数接受一个topology类型和一个type参数，用于获取指定name类型的距离矩阵。`hwloc_distances_get_by_name`函数接受一个名字参数和`nr`参数，用于获取与name相对应的距离矩阵。这两个函数都是属于`hwloc_distances_s`结构体的函数，用于获取预定义的距离矩阵。


```
HWLOC_DECLSPEC int
hwloc_distances_get_by_type(hwloc_topology_t topology, hwloc_obj_type_t type,
			    unsigned *nr, struct hwloc_distances_s **distances,
			    unsigned long kind, unsigned long flags);

/** \brief Retrieve a distance matrix with the given name.
 *
 * Usually only one distances structure may match a given name.
 *
 * The name of the most common structure is "NUMALatency".
 * Others include "XGMIBandwidth", "XGMIHops", "XeLinkBandwidth",
 * and "NVLinkBandwidth".
 */
HWLOC_DECLSPEC int
hwloc_distances_get_by_name(hwloc_topology_t topology, const char *name,
			    unsigned *nr, struct hwloc_distances_s **distances,
			    unsigned long flags);

```cpp

这两函数定义在`hwloc_distances.h`文件中。它们的作用如下：

1. `hwloc_distances_get_name`函数：获取距离结构中包含的描述。它接收一个`hwloc_topology_t`类型的顶部和一个`struct hwloc_distances_s`类型的结构体，然后返回一个指向字符串的指针。换句话说，这个函数返回描述距离结构物名称的指针。

2. `hwloc_distances_release`函数：释放距离矩阵结构，它接收一个`hwloc_topology_t`类型的顶部和一个`struct hwloc_distances_s`类型的结构体。如果结构体已经被使用`hwloc_distances_release_remove`函数释放，那么这个函数将不再需要执行，否则会执行。

这两个函数一起工作，确保`hwloc_distances`结构体在离开当前系统时得到正确清理。


```
/** \brief Get a description of what a distances structure contains.
 *
 * For instance "NUMALatency" for hardware-provided NUMA distances (ACPI SLIT),
 * or NULL if unknown.
 */
HWLOC_DECLSPEC const char *
hwloc_distances_get_name(hwloc_topology_t topology, struct hwloc_distances_s *distances);

/** \brief Release a distance matrix structure previously returned by hwloc_distances_get().
 *
 * \note This function is not required if the structure is removed with hwloc_distances_release_remove().
 */
HWLOC_DECLSPEC void
hwloc_distances_release(hwloc_topology_t topology, struct hwloc_distances_s *distances);

```cpp

This is a definition of the `HWLOC_DISTANCES_TRANSFORM_KINDS` constant in the `hwloc_distances_transform.h` header file.

`HWLOC_DISTANCES_TRANSFORM_KINDS` is a set of constants that specifies which of the following operations should be performed when the `hwloc_distances_transform()` function is called:

| #ferenced-constant | Description |
| --- | --- |
| `HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL` | - Remove all objects with a NULL kind. |
| `HWLOC_DISTANCES_TRANSFORM_LINKS` | - Replace bandwidth values with a number of links. All values will be either 0 (no link) or 1 (one link), with some matrices getting larger values if some pairs of peers are connected by different numbers of links. |
| `HWLOC_DISTANCES_TRANSFORM_MERGE_SWITCH_PORTS` | - Merge switches with multiple ports into a single object. This transformation applies only to NVSwitches where GPUs seem connected to different switch ports in the NVLinkBandwidth matrix. |


```
/** \brief Transformations of distances structures. */
enum hwloc_distances_transform_e {
  /** \brief Remove \c NULL objects from the distances structure.
   *
   * Every object that was replaced with \c NULL in the \p objs array
   * is removed and the \p values array is updated accordingly.
   *
   * At least \c 2 objects must remain, otherwise hwloc_distances_transform()
   * will return \c -1 with \p errno set to \c EINVAL.
   *
   * \p kind will be updated with or without ::HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES
   * according to the remaining objects.
   *
   * \hideinitializer
   */
  HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL = 0,

  /** \brief Replace bandwidth values with a number of links.
   *
   * Usually all values will be either \c 0 (no link) or \c 1 (one link).
   * However some matrices could get larger values if some pairs of
   * peers are connected by different numbers of links.
   *
   * Values on the diagonal are set to \c 0.
   *
   * This transformation only applies to bandwidth matrices.
   *
   * \hideinitializer
   */
  HWLOC_DISTANCES_TRANSFORM_LINKS = 1,

  /** \brief Merge switches with multiple ports into a single object.
   * This currently only applies to NVSwitches where GPUs seem connected to different
   * separate switch ports in the NVLinkBandwidth matrix. This transformation will
   * replace all of them with the same port connected to all GPUs.
   * Other ports are removed by applying ::HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL internally.
   * \hideinitializer
   */
  HWLOC_DISTANCES_TRANSFORM_MERGE_SWITCH_PORTS = 2,

  /** \brief Apply a transitive closure to the matrix to connect objects across switches.
   * This currently only applies to GPUs and NVSwitches in the NVLinkBandwidth matrix.
   * All pairs of GPUs will be reported as directly connected.
   * \hideinitializer
   */
  HWLOC_DISTANCES_TRANSFORM_TRANSITIVE_CLOSURE = 3
};

```cpp

这段代码定义了一个名为`transform_obj`的函数，其作用是应用给定的变换对距离结构进行修改。其定义了一个新的距离结构，与原结构具有相同的名称、类型、对象和值，然后通过`hwloc_distances_release_remove()`函数移除旧的距离结构。

在函数体中，首先检查给定的`transform`是否属于预定义的变换列表之一，如果是，则执行相应的变换操作。这些变换可以修改`objs`或`values`数组的内容。

函数内部暂时没有实现transform属性，以及设置`flags`为0（当前暂无使用该属性的规定）。

需要注意的是，距离结构中的对象可以在函数内部直接修改，而不需要使用`hwloc_distances_transform()`函数。如果需要，可以通过调用`hwloc_get_obj_with_same_locality()`函数将其转换为类似但不同类型的对象。


```
/** \brief Apply a transformation to a distances structure.
 *
 * Modify a distances structure that was previously obtained with
 * hwloc_distances_get() or one of its variants.
 *
 * This modifies the local copy of the distances structures but does
 * not modify the distances information stored inside the topology
 * (retrieved by another call to hwloc_distances_get() or exported to XML).
 * To do so, one should add a new distances structure with same
 * name, kind, objects and values (see \ref hwlocality_distances_add)
 * and then remove this old one with hwloc_distances_release_remove().
 *
 * \p transform must be one of the transformations listed
 * in ::hwloc_distances_transform_e.
 *
 * These transformations may modify the contents of the \p objs or \p values arrays.
 *
 * \p transform_attr must be \c NULL for now.
 *
 * \p flags must be \c 0 for now.
 *
 * \note Objects in distances array \p objs may be directly modified
 * in place without using hwloc_distances_transform().
 * One may use hwloc_get_obj_with_same_locality() to easily convert
 * between similar objects of different types.
 */
```cpp

这段代码定义了一个名为 `hwloc_distances_transform` 的函数，属于 `hwlocality_distances_consult` 组的成员函数。它的作用是处理 `hwloc_topology_t` 类型的数据结构，其中的 `distances` 是一个 `struct hwloc_distances_s` 类型的变量，`transform` 是一个 `enum hwloc_distances_transform_e` 类型的变量，`transform_attr` 是一个 `void *transform_attr` 类型的变量，`flags` 是一个 `unsigned long` 类型的变量。

具体来说，函数接受四个参数：

1. `topology`：一个 `hwloc_topology_t` 类型的数据结构，表示要处理的数据结构。
2. `distances`：一个 `struct hwloc_distances_s` 类型的变量，用于存储距离信息。
3. `transform`：一个 `enum hwloc_distances_transform_e` 类型的变量，表示要应用的变换类型。
4. `transform_attr`：一个 `void *transform_attr` 类型的变量，用于存储变换属性。
5. `flags`：一个 `unsigned long` 类型的变量，用于设置或清除变换标志。

函数的主要作用是执行距离矩阵的变换，将距离和方向与输入数据匹配。通过传递给函数的参数，可以控制变换的方式、属性以及索引。函数的返回值表示变换后的距离结构中的数据对。


```
HWLOC_DECLSPEC int hwloc_distances_transform(hwloc_topology_t topology, struct hwloc_distances_s *distances,
                                             enum hwloc_distances_transform_e transform,
                                             void *transform_attr,
                                             unsigned long flags);

/** @} */



/** \defgroup hwlocality_distances_consult Helpers for consulting distance matrices
 * @{
 */

/** \brief Find the index of an object in a distances structure.
 *
 * \return -1 if object \p obj is not involved in structure \p distances.
 */
```cpp

这段代码定义了一个名为 `hwloc_distances_obj_index` 的函数，它接受一个名为 `distances` 的 `hwloc_distances_s` 结构体和一个名为 `obj` 的 `hwloc_obj_t` 结构体作为参数。

该函数的作用是查找距离矩阵中两个对象的值。它首先定义了一个名为 `distances` 的结构体，它包含一个名为 `nbobjs` 的整数成员，表示距离矩阵中的对象数量。它还定义了一个名为 `objs` 的结构体，用于存储距离矩阵中的每个对象。

函数的核心部分是第二个循环，它接受一个名为 `obj` 的 `hwloc_obj_t` 结构体作为参数。在循环中，函数首先检查 `distances` 结构体中的第二个元素是否与 `obj` 结构体中的第二个元素相等。如果是，函数返回第二个元素的索引（即 `i` 的值）。否则，函数返回 `-1`，表示无法找到距离矩阵中包含 `obj` 的对象。

总的来说，该函数是用于在距离矩阵中查找两个对象的值，并返回它们的索引。


```
static __hwloc_inline int
hwloc_distances_obj_index(struct hwloc_distances_s *distances, hwloc_obj_t obj)
{
  unsigned i;
  for(i=0; i<distances->nbobjs; i++)
    if (distances->objs[i] == obj)
      return (int)i;
  return -1;
}

/** \brief Find the values between two objects in a distance matrices.
 *
 * The distance from \p obj1 to \p obj2 is stored in the value pointed by
 * \p value1to2 and reciprocally.
 *
 * \return -1 if object \p obj1 or \p obj2 is not involved in structure \p distances.
 */
```cpp

这段代码定义了一个名为 `hwloc_distances_obj_pair_values` 的函数，属于 `hwloc_distances_s` 结构体的成员函数。

该函数接收两个参数：`distances`，`obj1` 和 `obj2`，它们都是 `hwloc_obj_t` 类型的参数，并且接收一个指向 `hwloc_uint64_t` 类型变量 `value1to2` 和 `value2to1` 的指针。

函数首先检查 `i1` 和 `i2` 是否为负，如果是，则返回 -1，否则，函数将 `distances->values[i1 * distances->nbobjs + i2]` 存储到 `value1to2`，并将 `distances->values[i2 * distances->nbobjs + i1]` 存储到 `value2to1`。最后，函数返回 0。

从函数的名称来看，它似乎在计算两个 `hwloc_distances_obj_t` 的距离，并将结果存储到两个指针变量 `value1to2` 和 `value2to1` 中。


```
static __hwloc_inline int
hwloc_distances_obj_pair_values(struct hwloc_distances_s *distances,
				hwloc_obj_t obj1, hwloc_obj_t obj2,
				hwloc_uint64_t *value1to2, hwloc_uint64_t *value2to1)
{
  int i1 = hwloc_distances_obj_index(distances, obj1);
  int i2 = hwloc_distances_obj_index(distances, obj2);
  if (i1 < 0 || i2 < 0)
    return -1;
  *value1to2 = distances->values[i1 * distances->nbobjs + i2];
  *value2to1 = distances->values[i2 * distances->nbobjs + i1];
  return 0;
}

/** @} */



```cpp

这段代码定义了一个名为 `hwlocality_distances_add` 的函数，用于在给定的拓扑结构中计算两个对象之间的距离。函数接受四个参数：`topology`、`kind` 和 `nbobj`，它们分别表示要计算距离的对象的拓扑结构类型、要计算的距离对象的索引号，以及要计算的距离对象的数目。函数内部包含三个如果条件：

1. 如果调用 `hwlocality_distances_add_create` 时，函数失败，则返回 `-1`，表示添加距离失败。
2. 如果调用 `hwlocality_distances_add_values` 时，传递的参数 `handle` 无效，则返回 `-1`，表示无效的 `handle` 值。
3. 如果调用 `hwlocality_distances_add_commit` 时，传递的参数 `handle` 和 `flags` 无效，则返回 `-1`，表示无效的 `handle` 值或 `flags` 设置。

函数的主要作用是在给定的拓扑结构中计算两个对象之间的距离，并在需要时进行添加操作。


```
/** \defgroup hwlocality_distances_add Add distances between objects
 *
 * The usual way to add distances is:
 * \code
 * hwloc_distances_add_handle_t handle;
 * int err = -1;
 * handle = hwloc_distances_add_create(topology, "name", kind, 0);
 * if (handle) {
 *   err = hwloc_distances_add_values(topology, handle, nbobjs, objs, values, 0);
 *   if (!err)
 *     err = hwloc_distances_add_commit(topology, handle, flags);
 * }
 * \endcode
 * If \p err is \c 0 at the end, then addition was successful.
 *
 * @{
 */

```cpp

这段代码定义了一个名为 `hwloc_distances_add_handle_t` 的指针类型，用于处理将新距离结构添加到 topology 时的情况。

该代码中定义了一个名为 `hwloc_distances_add_kind_e` 的枚举类型，用于指定距离的类型。距离可以是或运算的组合，例如 `hwloc_distances_kind_e.HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES`。

代码还定义了一个名为 `hwloc_distances_add_flags_e` 的枚举类型，用于指定距离的标志，目前为 `0`。

接着，定义了一个名为 `hwloc_distances_add_handle_t` 的指针类型，用于处理将新距离结构添加到 topology 时的情况。该指针类型包含两个成员：一个指向距离结构的指针，另一个用于将新结构添加到 topology 的函数指针。

最后，定义了一个名为 `hwloc_distances_add_values_ex_handler_t` 的函数指针类型，用于处理将新距离结构中的值添加到 topology 的情况。


```
/** \brief Handle to a new distances structure during its addition to the topology. */
typedef void * hwloc_distances_add_handle_t;

/** \brief Create a new empty distances structure.
 *
 * Create an empty distances structure
 * to be filled with hwloc_distances_add_values()
 * and then committed with hwloc_distances_add_commit().
 *
 * Parameter \p name is optional, it may be \c NULL.
 * Otherwise, it will be copied internally and may later be freed by the caller.
 *
 * \p kind specifies the kind of distance as a OR'ed set of ::hwloc_distances_kind_e.
 * Kind ::HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES will be automatically set
 * according to objects having different types in hwloc_distances_add_values().
 *
 * \p flags must be \c 0 for now.
 *
 * \return A hwloc_distances_add_handle_t that should then be passed
 * to hwloc_distances_add_values() and hwloc_distances_add_commit().
 *
 * \return \c NULL on error.
 */
```cpp

这段代码定义了一个名为hwloc_distances_add_handle_t的结构体，用于表示距离添加 handle。该handle在使用hwloc_distances_add_create函数时指定要添加的距离对象和值。

hwloc_distances_add_create函数接受一个hwloc_topology_t类型的topology作为第一个参数，指定新创建的距离对象的名称，kind，和flags。topology是一个表示底层硬件组件的topology结构体，而kind，flags和hwloc_distances_add_handle_t都应该是long long类型的整数。

在hwloc_distances_add_create函数中，根据传入的topology和名称，在新建距离对象时执行相应的操作，然后将对象和值复制到hwloc_distances_add_handle_t结构体中，以便后续提交和承诺。

在后面的使用函数中，可以调用hwloc_distances_add_create函数来创建新的距离对象，并使用hwloc_distances_add_handle_t结构体中的函数来访问和操作距离对象。


```
HWLOC_DECLSPEC hwloc_distances_add_handle_t
hwloc_distances_add_create(hwloc_topology_t topology,
                           const char *name, unsigned long kind,
                           unsigned long flags);

/** \brief Specify the objects and values in a new empty distances structure.
 *
 * Specify the objects and values for a new distances structure
 * that was returned as a handle by hwloc_distances_add_create().
 * The structure must then be committed with hwloc_distances_add_commit().
 *
 * The number of objects is \p nbobjs and the array of objects is \p objs.
 * Distance values are stored as a one-dimension array in \p values.
 * The distance from object i to object j is in slot i*nbobjs+j.
 *
 * \p nbobjs must be at least 2.
 *
 * Arrays \p objs and \p values will be copied internally,
 * they may later be freed by the caller.
 *
 * On error, the temporary distances structure and its content are destroyed.
 *
 * \p flags must be \c 0 for now.
 *
 * \return \c 0 on success.
 * \return \c -1 on error.
 */
```cpp

这段代码定义了一个名为 `hwloc_distances_add_values` 的函数，属于 `hwloc_topology_t` 结构的成员函数。它的作用是接受一个 `hwloc_topology_t` 类型的参数 `topology`，一个 `hwloc_distances_add_handle_t` 类型的参数 `handle`，一个 `unsigned int` 类型的参数 `nbobjs`，以及一个指向 `hwloc_obj_t` 类型对象的指针 `objs` 和一个指向 `hwloc_uint64_t` 类型的指针 `values`，然后对传入的参数进行处理，并返回结果。

具体来说，这段代码的作用是：

1. 如果传递给 `hwloc_distances_add_handle_t` 类型的参数 `handle` 中的 `HWLOC_DISTANCES_ADD_FLAG_GROUP` 标志为 1，那么函数会尝试根据传递的距离信息对传入的 `topology` 和 `nbobjs` 进行分组，并使用这个分组策略来计算新的距离。
2. 如果传递给 `hwloc_distances_add_handle_t` 类型的参数 `handle` 中 `HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE` 标志为 1，那么函数会尝试对传入的距离进行分组，并在分组过程中使用一种更宽松的比较规则，以减轻对象间距离较小时计算不准确的情况。
3. 如果传递给 `hwloc_topology_t` 类型的参数 `topology` 中的 `HWLOC_GROUPING_ACCURACY` 环境变量为 `HWLOC_GROUPING_ACCURACY_NONE`，那么函数不会尝试对传入的距离进行分组，也不会使用 `HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE` 标志，而是直接对传入的距离进行计算，并返回结果。


```
HWLOC_DECLSPEC int hwloc_distances_add_values(hwloc_topology_t topology,
                                              hwloc_distances_add_handle_t handle,
                                              unsigned nbobjs, hwloc_obj_t *objs,
                                              hwloc_uint64_t *values,
                                              unsigned long flags);

/** \brief Flags for adding a new distances to a topology. */
enum hwloc_distances_add_flag_e {
  /** \brief Try to group objects based on the newly provided distance information.
   * This is ignored for distances between objects of different types.
   * \hideinitializer
   */
  HWLOC_DISTANCES_ADD_FLAG_GROUP = (1UL<<0),
  /** \brief If grouping, consider the distance values as inaccurate and relax the
   * comparisons during the grouping algorithms. The actual accuracy may be modified
   * through the HWLOC_GROUPING_ACCURACY environment variable (see \ref envvar).
   * \hideinitializer
   */
  HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE = (1UL<<1)
};

```cpp

这段代码定义了一个名为“commit_distances”的新函数，它的作用是结束现有的距离结构并将其内容插入到新结构中。通过这个函数，用户可以指定新结构中对象的配置和距离相关的行为。

具体来说，这个函数需要一个已经由 `hwloc_distances_add_create()` 函数返回的 `handle` 参数。接着，通过调用 `hwloc_distances_add_values()` 函数，传递给新结构中的属性和距离。

此外，通过传递一个可选的 `::hwloc_distances_add_flag_e` 数组，来配置函数的行为。这段代码允许用户请求对现有对象进行分组，可以根据不同的距离度量进行分组。

如果在函数执行过程中出现错误，函数会尝试释放当前内存中的距离结构和其内容，以避免内存泄漏。


```
/** \brief Commit a new distances structure.
 *
 * This function finalizes the distances structure and inserts in it the topology.
 *
 * Parameter \p handle was previously returned by hwloc_distances_add_create().
 * Then objects and values were specified with hwloc_distances_add_values().
 *
 * \p flags configures the behavior of the function using an optional OR'ed set of
 * ::hwloc_distances_add_flag_e.
 * It may be used to request the grouping of existing objects based on distances.
 *
 * On error, the temporary distances structure and its content are destroyed.
 *
 * \return \c 0 on success.
 * \return \c -1 on error.
 */
```cpp

这段代码定义了一个名为`hwloc_distances_add_commit`的函数，属于`hwloc_topology_t`和`hwloc_distances_add_handle_t`结构的实参类型。该函数的主要作用是添加新的`hwloc_distances_add_handle_t`结构，以将距离添加到指定的拓扑中。

进一步地，该函数还有两个辅助函数：`hwloc_topology_t`和`hwloc_distances_add_handle_t`。第一个辅助函数返回一个指向`hwloc_topology_t`结构体的指针，用于指定要添加距离的拓扑。第二个辅助函数是一个整数类型，用于存储添加的距离的标志，这些距离将被设置为`unsigned long`长度的`flags`参数。


```
HWLOC_DECLSPEC int hwloc_distances_add_commit(hwloc_topology_t topology,
                                              hwloc_distances_add_handle_t handle,
                                              unsigned long flags);

/** @} */



/** \defgroup hwlocality_distances_remove Remove distances between objects
 * @{
 */

/** \brief Remove all distance matrices from a topology.
 *
 * Remove all distance matrices, either provided by the user or
 * gathered through the OS.
 *
 * If these distances were used to group objects, these additional
 * Group objects are not removed from the topology.
 */
```cpp

这段代码的作用是移除topology中深度为指定值（由`hwloc_get_type_depth`函数获取）且类型为`hwloc_obj_type_t`的物体所对应的距离矩阵。具体实现是通过`hwloc_distances_remove_by_depth`函数实现的，如果topology中存在该类型的物体，该函数将返回从该物体到根节点的距离；如果topology中不存在该类型的物体，则返回0。


```
HWLOC_DECLSPEC int hwloc_distances_remove(hwloc_topology_t topology);

/** \brief Remove distance matrices for objects at a specific depth in the topology.
 *
 * Identical to hwloc_distances_remove() but only applies to one level of the topology.
 */
HWLOC_DECLSPEC int hwloc_distances_remove_by_depth(hwloc_topology_t topology, int depth);

/** \brief Remove distance matrices for objects of a specific type in the topology.
 *
 * Identical to hwloc_distances_remove() but only applies to one level of the topology.
 */
static __hwloc_inline int
hwloc_distances_remove_by_type(hwloc_topology_t topology, hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return 0;
  return hwloc_distances_remove_by_depth(topology, depth);
}

```cpp

这段代码定义了一个名为 `hwloc_distances_release_remove` 的函数，属于 `hwloc_topology_exchange` 函数家族。它实现在 `hwloc_topology_t` 结构体上的一个函数指针。

该函数的作用是释放并从topology中移除给定的距离矩阵，可以通过调用 `hwloc_distances_release()` 函数来实现。

该函数的实现没有定义，因此无法提供更多的信息。


```
/** \brief Release and remove the given distance matrice from the topology.
 *
 * This function includes a call to hwloc_distances_release().
 */
HWLOC_DECLSPEC int hwloc_distances_release_remove(hwloc_topology_t topology, struct hwloc_distances_s *distances);

/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_DISTANCES_H */

```