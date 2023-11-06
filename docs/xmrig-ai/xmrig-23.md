# xmrig源码解析 23

# `src/3rdparty/hwloc/include/hwloc/inlines.h`

这段代码定义了一个名为"hwloc_inlines.h"的头文件，其中包含了一些函数的内部代码。这些函数是在hwloc.h中声明的。

头文件中包含的函数声明以及必要的版权声明表明了这些函数的版权所有人，包括：

* CNRS（法国国家科学研究中心）
* Inria（IBM子公司）
* Université Bordeaux（波尔多大学）
* Cisco Systems, Inc。（思科系统公司）

此外，在头文件末尾的版权信息显示了版权所有人自某个日期起拥有的专利权。

这些函数的实现并未在hwloc.h中显示，因此无法从该文件中引用这些函数。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2018 Inria.  All rights reserved.
 * Copyright © 2009-2012 Université Bordeaux
 * Copyright © 2009-2010 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/**
 * This file contains the inline code of functions declared in hwloc.h
 */

#ifndef HWLOC_INLINES_H
#define HWLOC_INLINES_H

```

这段代码定义了一个名为`hwloc_get_type_or_below_depth`的函数，它接受一个`hwloc_topology_t`类型的顶类和一个`hwloc_obj_type_t`类型的参数。这个函数的作用是在给定的顶类和参数类型的条件下，返回一个合适的`hwloc_get_type_depth`函数的调用结果。如果给定的顶类和参数类型的深度未知，函数将返回`HWLOC_TYPE_DEPTH_UNKNOWN`。

函数体中首先定义了一个`hwloc_get_type_depth`函数，这个函数的第一个参数是一个`hwloc_topology_t`类型的顶类，第二个参数是一个`hwloc_obj_type_t`类型的参数。这个函数的作用是在给定的顶类下查找与给定参数类型的最深的已知类型，并返回该类型的深度。如果找到该类型的深度未知，函数将返回`HWLOC_TYPE_DEPTH_UNKNOWN`。

接下来定义了`hwloc_get_type_or_below_depth`函数，这个函数在`hwloc_get_type_depth`函数的基础上增加了一个`for`循环，用于查找给定顶类下的最低已知类型。这个循环的条件是`hwloc_compare_types(hwloc_get_depth_type(topology, depth), type) < 0`，如果给定的类型高于当前查找的最低类型，则返回当前查找的深度加一。最终，如果循环未找到该类型，函数将返回`HWLOC_TYPE_DEPTH_UNKNOWN`。


```cpp
#ifndef HWLOC_H
#error Please include the main hwloc.h instead
#endif

#include <stdlib.h>
#include <errno.h>


#ifdef __cplusplus
extern "C" {
#endif

static __hwloc_inline int
hwloc_get_type_or_below_depth (hwloc_topology_t topology, hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);

  if (depth != HWLOC_TYPE_DEPTH_UNKNOWN)
    return depth;

  /* find the highest existing level with type order >= */
  for(depth = hwloc_get_type_depth(topology, HWLOC_OBJ_PU); ; depth--)
    if (hwloc_compare_types(hwloc_get_depth_type(topology, depth), type) < 0)
      return depth+1;

  /* Shouldn't ever happen, as there is always a Machine level with lower order and known depth.  */
  /* abort(); */
}

```

这段代码定义了一个名为 `hwloc_get_type_or_above_depth` 的函数，它接受一个 `hwloc_topology_t` 的参数 `topology` 和一个 `hwloc_obj_type_t` 的参数 `type`。这个函数的作用是返回 `hwloc_get_type_depth(topology, type)` 失败时，能够找到的最低支持 `type` 的深度。

函数内部首先定义了一个名为 `depth` 的整数变量，用于记录当前查找的最低支持深度。然后，函数调用了 `hwloc_get_type_depth(topology, type)` 函数，获取了当前深度 `depth` 对应的类型深度。如果当前深度已经是支持的类型中的最低深度，函数就返回了当前深度。否则，函数会遍历当前深度以下所有支持 `type` 的最低深度，并返回其中最小的深度。

在遍历过程中，函数还检查了一个名为 `hwloc_compare_types` 的函数，它比较两个 `hwloc_get_depth_type` 函数返回的值，如果两个比较结果为正，则说明当前深度以下的深度比当前深度更大，函数就返回当前深度减 1。然而，这个比较函数在 `hwloc_get_type_depth` 函数中可能永远也不会用到，因此需要在函数内部进行注释，提醒程序员不要在不需要的时候引用它。


```cpp
static __hwloc_inline int
hwloc_get_type_or_above_depth (hwloc_topology_t topology, hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);

  if (depth != HWLOC_TYPE_DEPTH_UNKNOWN)
    return depth;

  /* find the lowest existing level with type order <= */
  for(depth = 0; ; depth++)
    if (hwloc_compare_types(hwloc_get_depth_type(topology, depth), type) > 0)
      return depth-1;

  /* Shouldn't ever happen, as there is always a PU level with higher order and known depth.  */
  /* abort(); */
}

```

这两段代码是用来计算 HWLOC 中的对象数量（nbobjs）的函数，其中 `hwloc_get_nbobjs_by_type` 函数接收一个 HWLOC 的 topology 和一个对象类型 type，返回该类型对象的数量，若 depth 为 depth_unknown 则返回 0，若 depth 为 depth_multiple 则返回 depth 减 1，然后调用 `hwloc_get_nbobjs_by_depth` 函数计算深度为 depth 的对象数量。 `hwloc_get_obj_by_type` 函数则接收 topology、type 和 index，返回 depth 为 depth 的对象，若 depth 为 depth_unknown 或 depth_multiple，则返回 NULL。


```cpp
static __hwloc_inline int
hwloc_get_nbobjs_by_type (hwloc_topology_t topology, hwloc_obj_type_t type)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
    return 0;
  if (depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return -1; /* FIXME: agregate nbobjs from different levels? */
  return (int) hwloc_get_nbobjs_by_depth(topology, depth);
}

static __hwloc_inline hwloc_obj_t
hwloc_get_obj_by_type (hwloc_topology_t topology, hwloc_obj_type_t type, unsigned idx)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN)
    return NULL;
  if (depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  return hwloc_get_obj_by_depth(topology, depth, idx);
}

```

这两段代码是定义了两个名为 `hwloc_get_next_obj_by_depth` 和 `hwloc_get_next_obj_by_type` 的函数，用于在给定的 `hwloc_topology_t` 结构中，根据不同的参数类型和深度，获取下一个 `hwloc_obj_t` 对象的引用。

`hwloc_get_next_obj_by_depth` 函数的作用是在给定的深度下，从根对象(也就是 `hwloc_get_obj_by_depth` 函数返回的第一个对象)开始递归获取下一个对象。它的参数包括 `topology` 表示当前对象的顶级结构体和 `depth` 表示要递归的深度。如果已经递归到深度 `depth`，那么函数将返回 `NULL`，否则返回根对象的 `next_cousin` 指针。

`hwloc_get_next_obj_by_type` 函数的作用是在给定的 `topology` 和 `type` 参数下，根据 `hwloc_get_type_depth` 函数获取了当前对象的类型深度 `depth`。如果当前对象的类型深度未知或者多个，函数将返回 `NULL`。否则，函数将调用 `hwloc_get_next_obj_by_depth` 函数获取下一个对象。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_by_depth (hwloc_topology_t topology, int depth, hwloc_obj_t prev)
{
  if (!prev)
    return hwloc_get_obj_by_depth (topology, depth, 0);
  if (prev->depth != depth)
    return NULL;
  return prev->next_cousin;
}

static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_by_type (hwloc_topology_t topology, hwloc_obj_type_t type,
			    hwloc_obj_t prev)
{
  int depth = hwloc_get_type_depth(topology, type);
  if (depth == HWLOC_TYPE_DEPTH_UNKNOWN || depth == HWLOC_TYPE_DEPTH_MULTIPLE)
    return NULL;
  return hwloc_get_next_obj_by_depth (topology, depth, prev);
}

```

这两段代码定义了两个名为 `hwloc_get_root_obj` 和 `hwloc_obj_get_info_by_name` 的函数。它们属于 `hwloc_obj` 类型的常量对象。

`hwloc_get_root_obj` 函数接收一个 `hwloc_topology_t` 类型的参数 `topology`，并返回该 `topology` 中的根对象。它通过调用 `hwloc_get_obj_by_depth` 函数并传递 `topology` 参数来获取根对象。

`hwloc_obj_get_info_by_name` 函数接收一个 `hwloc_obj_t` 类型的参数 `obj`，以及一个要查找的 `const char *` 类型的参数 `name`。它遍历 `obj` 对象中的所有信息，并检查是否存在与给定名字的相同信息。如果是，则返回该信息的相关值。如果没有找到相同的信息，函数将返回 `NULL`。

这两个函数可以被用来从根对象中获取特定信息，以及在给定名称的定义中查找信息。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_get_root_obj (hwloc_topology_t topology)
{
  return hwloc_get_obj_by_depth (topology, 0, 0);
}

static __hwloc_inline const char *
hwloc_obj_get_info_by_name(hwloc_obj_t obj, const char *name)
{
  unsigned i;
  for(i=0; i<obj->infos_count; i++) {
    struct hwloc_info_s *info = &obj->infos[i];
    if (!strcmp(info->name, name))
      return info->value;
  }
  return NULL;
}

```

这段代码定义了一个名为 `hwloc_alloc_membind_policy` 的函数，它的作用是动态分配内存绑定策略，用于指定在 HWLOC 系统中如何绑定进程的虚拟内存区域。

具体来说，该函数接收四个参数：

1. `topology`：表示输入的 HWLOC  topology 结构体，用于指定要绑定的虚拟内存区域。
2. `len`：表示要绑定的虚拟内存区域长度。
3. `set`：表示用于内存绑定的 CPU 设置。
4. `policy`：表示要绑定的虚拟内存区域策略，可以是 HWLOC_MEMBIND_FIRSTTOUCH、HWLOC_MEMBIND_DESERTE 等。
5. `flags`：表示在使用 hwloc_set_membind() 时传递给它的标志，用于指示是否强制将策略应用到当前设置。

函数首先尝试从上下文中分配内存，如果失败，则调用 hwloc_set_membind() 来设置策略，并忽略错误。如果分配成功，则创建一个新的数据，并将其设置为指定的 CPU 设置，最后返回该数据。


```cpp
static __hwloc_inline void *
hwloc_alloc_membind_policy(hwloc_topology_t topology, size_t len, hwloc_const_cpuset_t set, hwloc_membind_policy_t policy, int flags)
{
  void *p = hwloc_alloc_membind(topology, len, set, policy, flags);
  if (p)
    return p;

  if (hwloc_set_membind(topology, set, policy, flags) < 0)
    /* hwloc_set_membind() takes care of ignoring errors if non-STRICT */
    return NULL;

  p = hwloc_alloc(topology, len);
  if (p && policy != HWLOC_MEMBIND_FIRSTTOUCH)
    /* Enforce the binding by touching the data */
    memset(p, 0, len);
  return p;
}


```

这段代码的作用是定义了一个名为"__cplusplus"的标识符，并将其声明为"extern"，即外部链接。同时，该标识符将被用来检查是否有定义，如果有定义，则不会输出其定义的地址，否则输出其定义的地址。

进一步地，该代码还会定义一个名为"HWLOC_INLINES_H"的标识符，并检查其与"__cplusplus"是否相同。如果两个标识符相同，则不会输出它们的定义地址，否则输出它们的定义地址。


```cpp
#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_INLINES_H */

```

# `src/3rdparty/hwloc/include/hwloc/levelzero.h`

这段代码定义了一个名为 "hwloc\_levelzero.h" 的头文件，其中包括一些宏，用于帮助在 hwloc 和 Level Zero 之间进行交互。这些宏将提供 topology 信息，以便 Level Zero 设备。该文件可能会被用于应用程序，这些应用程序使用 hwloc 和 Level Zero。


```cpp
/*
 * Copyright © 2021 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and the oneAPI Level Zero interface.
 *
 * Applications that use both hwloc and Level Zero may want to
 * include this file so as to get topology information for L0 devices.
 */

#ifndef HWLOC_LEVELZERO_H
#define HWLOC_LEVELZERO_H

```

这段代码是一个C++程序，它包括了两个头文件：hwloc.h，hwloc/autogen/config.h。hwloc是一个硬件定位工具，它允许在多个操作系统上查找硬件资源，如CPU、GPU、PCIe设备等。这个程序的作用是编译并运行这个C++程序，以支持hwloc工具的使用。

首先，这个程序包含了两个hwloc.h和hwloc/autogen/config.h头文件。hwloc.h和hwloc/autogen/config.h是hwloc的两个主要头文件。它们允许程序使用hwloc工具，这个工具允许在多个操作系统上查找硬件资源。

然后，这个程序引入了两个library：level_zero/ze_api.h和level_zero/zes_api.h。这两个library允许程序使用level_zero库，这个库提供了对硬件资源更高级别的抽象，使得程序可以更容易地使用硬件资源。

接下来，这个程序通过#ifdef HWLOC_LINUX_SYS预处理了一个操作系统特定的事件。如果这个程序是在Linux系统上运行的，它将包含一个包含两个 Ze (/ ze 系统事件) 头文件的hwloc/linux.h头文件。然后，它将可以访问到系统级别的硬件资源，如CPU、GPU、PCIe设备等。

最后，这个程序编译并运行了生成的C++程序。生成的程序可以被运行在支持C++的平台上，如Linux、Windows等。通过运行这个程序，可以支持hwloc工具的使用，并允许程序使用level_zero库来访问硬件资源。


```cpp
#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <level_zero/ze_api.h>
#include <level_zero/zes_api.h>


#ifdef __cplusplus
extern "C" {
#endif


```

这段代码定义了一个名为 `hwlocality_levelzero` 的函数组，用于与 OneAPI Level Zero 接口进行交互。这个接口可以用来检索由 OneAPI Level Zero 管理设备的拓扑信息。

函数 `hwlocality_levelzero_get_device_osdev` 使用这个接口来获取拓扑信息，并将结果存储在 `device_osdev` 变量中。如果需要获取更多信息，可以调用 `hwloc_levelzero_get_device_osdev` 函数。

函数 `hwlocality_levelzero_get_cpu_set` 使用这个接口来获取与 Level Zero 设备物理位置相邻的逻辑处理器（CPU）的 CPU 集合。并将结果存储在 `cpu_set` 变量中。

函数 `hwlocality_levelzero_get_device_topology` 使用这个接口来获取 Level Zero 设备的拓扑信息。并将结果存储在 `device_topology` 变量中。

函数 `hwlocality_levelzero_get_device_power` 使用这个接口来获取 Level Zero 设备的力量（功率）设置。并将结果存储在 `device_power` 变量中。

函数 `hwlocality_levelzero_get_device_status` 使用这个接口来获取 Level Zero 设备的状态。并将结果存储在 `device_status` 变量中。

函数 `hwlocality_levelzero_get_device_resources` 使用这个接口来获取 Level Zero 设备上的资源。并将结果存储在 `device_resources` 变量中。

函数 `hwlocality_levelzero_get_device_temperature` 使用这个接口来获取 Level Zero 设备上的温度。并将结果存储在 `device_temperature` 变量中。

函数 `hwlocality_levelzero_get_device_movement` 使用这个接口来获取 Level Zero 设备上的运动。并将结果存储在 `device_movement` 变量中。

函数 `hwlocality_levelzero_get_device_type` 使用这个接口来获取 Level Zero 设备的类型。并将结果存储在 `device_type` 变量中。

函数 `hwlocality_levelzero_get_device_count` 使用这个接口来获取 Level Zero 设备的数量。并将结果存储在 `device_count` 变量中。

函数 `hwlocality_levelzero_get_device_power_status` 使用这个接口来获取 Level Zero 设备的力量状态。并将结果存储在 `device_power_status` 变量中。


```cpp
/** \defgroup hwlocality_levelzero Interoperability with the oneAPI Level Zero interface.
 *
 * This interface offers ways to retrieve topology information about
 * devices managed by the Level Zero API.
 *
 * @{
 */

/** \brief Get the CPU set of logical processors that are physically
 * close to the Level Zero device \p device
 *
 * Store in \p set the CPU-set describing the locality of
 * the Level Zero device \p device.
 *
 * Topology \p topology and device \p device must match the local machine.
 * The Level Zero must have been initialized with Sysman enabled
 * (ZES_ENABLE_SYSMAN=1 in the environment).
 * I/O devices detection and the Level Zero component are not needed in the
 * topology.
 *
 * The function only returns the locality of the device.
 * If more information about the device is needed, OS objects should
 * be used instead, see hwloc_levelzero_get_device_osdev().
 *
 * This function is currently only implemented in a meaningful way for
 * Linux; other systems will simply get a full cpuset.
 */
```

这段代码的作用是获取一个硬件设备（如CPU集）的CPU集合，并将它存储在一个特定的CPU集合中。这个代码针对Linux系统，在获取CPU集合时使用了两个途径：一种是直接通过系统调用获取，另一种是在Linux系统上使用sysfs机制获取。

具体来说，首先定义了一个名为`hwloc_levelzero_get_device_cpuset`的函数，它接受一个hwloc_topology_t类型的顶类图和ze_device_handle_t类型的设备句柄作为参数。这个函数首先检查要获取的设备是否在Linux系统上，如果不是，就返回-1错误。

如果设备在Linux系统上，那么就会执行以下操作：首先尝试通过`hwloc_topology_is_thissystem`函数获取系统类型，如果没有，就返回EINVAL错误。然后使用`zes_device_handle_pci_get_properties`函数获取设备的PCI属性，并使用`zes_sprintf`函数将结果打印为字符串，其中包含通过`/sys/bus/pci/devices/%04x:%02x:%02x.%01x/local_cpus`路径获取的CPU集合。接下来，使用`hwloc_linux_read_path_as_cpumask`函数将该字符串转换为CPU集合，如果失败，就返回一个包含所有CPU的掩码。最后，使用`hwloc_bitmap_iszero`函数将CPU集合与给定的硬件设备关联的CPU集合的位图复制到给定的CPU集合中。


```cpp
static __hwloc_inline int
hwloc_levelzero_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
                                  ze_device_handle_t device, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* If we're on Linux, use the sysfs mechanism to get the local cpus */
#define HWLOC_LEVELZERO_DEVICE_SYSFS_PATH_MAX 128
  char path[HWLOC_LEVELZERO_DEVICE_SYSFS_PATH_MAX];
  zes_pci_properties_t pci;
  zes_device_handle_t sdevice = device;
  ze_result_t res;

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return -1;
  }

  res = zesDevicePciGetProperties(sdevice, &pci);
  if (res != ZE_RESULT_SUCCESS) {
    errno = EINVAL;
    return -1;
  }

  sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.%01x/local_cpus",
          pci.address.domain, pci.address.bus, pci.address.device, pci.address.function);
  if (hwloc_linux_read_path_as_cpumask(path, set) < 0
      || hwloc_bitmap_iszero(set))
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
```

这段代码是一个if语句的else部分。其作用是判断给定的设备是否为Linux系统，如果是，则执行else语句块中的代码。否则，将执行topology.hwl文件的代码。

具体来说，代码将尝试从topology.hwl文件中获取描述Level Zero设备的hwloc OS设备对象。如果无法找到相应的设备对象，将返回NULL。

if语句块中的代码将在topology.hwl文件中定义的I/O设备检测和Level Zero组件启用的情况下工作。否则，它将无法保证找到相应的设备对象。


```cpp
#else
  /* Non-Linux systems simply get a full cpuset */
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}

/** \brief Get the hwloc OS device object corresponding to Level Zero device
 * \p device.
 *
 * \return The hwloc OS device object that describes the given Level Zero device \p device.
 * \return \c NULL if none could be found.
 *
 * Topology \p topology and device \p dv_ind must match the local machine.
 * I/O devices detection and the Level Zero component must be enabled in the
 * topology. If not, the locality of the object may still be found using
 * hwloc_levelzero_get_device_cpuset().
 *
 * \note The corresponding hwloc PCI device may be found by looking
 * at the result parent pointer (unless PCI devices are filtered out).
 */
```

这段代码定义了一个名为`hwloc_levelzero_get_device_osdev`的函数，它的作用是获取一个操作系统设备（device）的`osdev`对象。

函数接收两个参数：`hwloc_topology_t`表示低层硬件布局 topology，以及`ze_device_handle_t`表示一个 Zeel 设备句柄（device handle）。

函数首先检查 topology 是否表示系统（system），如果不是，则错误并返回 NULL。

接下来，函数使用 `zesDevicePciGetProperties` 函数从 device 句柄中获取设备级 Pci 属性，并使用这些属性查询 device 处的操作系统设备（osdev）。

如果 `hwloc_topology_is_thissystem` 函数返回 `true`，则 `zeros_device_osdev` 函数会尝试从系统中的所有 OSDev 对象中查找设备。

否则，函数将返回一个指向 `NULL` 的指针，表示没有找到匹配的操作系统设备。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_levelzero_get_device_osdev(hwloc_topology_t topology, ze_device_handle_t device)
{
  zes_device_handle_t sdevice = device;
  zes_pci_properties_t pci;
  ze_result_t res;
  hwloc_obj_t osdev;

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return NULL;
  }

  res = zesDevicePciGetProperties(sdevice, &pci);
  if (res != ZE_RESULT_SUCCESS) {
    /* L0 was likely initialized without sysman, don't bother */
    errno = EINVAL;
    return NULL;
  }

  osdev = NULL;
  while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
    hwloc_obj_t pcidev = osdev->parent;

    if (strncmp(osdev->name, "ze", 2))
      continue;

    if (pcidev
      && pcidev->type == HWLOC_OBJ_PCI_DEVICE
      && pcidev->attr->pcidev.domain == pci.address.domain
      && pcidev->attr->pcidev.bus == pci.address.bus
      && pcidev->attr->pcidev.dev == pci.address.device
      && pcidev->attr->pcidev.func == pci.address.function)
      return osdev;

    /* FIXME: when we'll have serialnumber, try it in case PCI is filtered-out */
  }

  return NULL;
}

```

这段代码是一个C语言的代码片段，其中包括一个头文件和两个预处理指令。

头文件：
```cpp
#include "hwloc_local_merge.h"
```
这个头文件是预处理指令，告诉编译器在编译之前要先包含这个文件。

预处理指令：
```cpp
#ifdef __cplusplus
 #define HWLOC_LEVELZERO_H
#endif
```
这个预处理指令表示这个文件是C++语言编写的，同时这个指令也是定义过的。它告诉编译器在编译之前要包含一个名为"hwloc_local_merge.h"的头文件。

这里使用了C++语言的特性，如果在C++语言环境中，这个预处理指令会告诉编译器在编译之前要先包含这个头文件。

此外，这个头文件可能定义了一些函数，由于没有给出具体的函数定义，无法判断这个头文件具体的作用。


```cpp
/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_LEVELZERO_H */

```

# `src/3rdparty/hwloc/include/hwloc/linux-libnuma.h`

这段代码定义了一个头文件 "hwloc-libnuma.h"，其中包括一些宏，用于帮助在 hwloc 和 Linux libnuma 之间进行交互。这些宏的目标是帮助应用软件更容易地从这两种系统的库中进行转换。

具体来说，这段代码定义了以下几个宏：

* export：这个宏定义了 export 函数，它是 hwloc 库中用于将 hwloc 类型转换为 libnuma 类型的一个函数。
* wgl：这个宏定义了 wgl 函数，它是 hwloc 库中用于将 hwloc 类型转换为 int 类型的一个函数。
* libnuma：这个宏定义了 libnuma 函数，它是 libnuma 库中用于进行数值操作的一个函数。
* hwloc：这个宏定义了 hwloc 函数，它是 hwloc 库中用于查找与 libnuma 接口的函数的一个函数。

由于 hwloc 和 libnuma 之间存在互操作，所以这些宏可以帮助应用软件更容易地从这两种系统的库中进行转换。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2017 Inria.  All rights reserved.
 * Copyright © 2009-2010, 2012 Université Bordeaux
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and Linux libnuma.
 *
 * Applications that use both Linux libnuma and hwloc may want to
 * include this file so as to ease conversion between their respective types.
*/

#ifndef HWLOC_LINUX_LIBNUMA_H
```

这段代码定义了一个名为 "hwlocality_linux_libnuma_ulongs" 的头文件，它包含了与 Linux libnuma 相关的接口。以下是这段代码的主要部分：

```cpp
#include "hwloc.h"

#include <numa.h>
```

这两个头文件是 hwloc 和 numa 的头文件，它们提供了用于在 hwloc 堆栈上使用 numa 功能的方法。

```cpp
#ifdef __cplusplus
extern "C" {
#endif
```

这个 "#ifdef" 是预处理指令，用于检查是否支持 `__cplusplus` 编译器。如果支持，那么它下面的内容将不会被编译，否则将会被编译。

```cpp
extern "C" {
```

这个 `extern "C"` 定义了一个名为 "hwlocality_linux_libnuma_ulongs" 的函数。它告诉编译器，接下来的内容将不会被编译，所以可以放在一起。

```cpp
/** \defgroup hwlocality_linux_libnuma_ulongs Interoperability with Linux libnuma unsigned long masks
*
* This interface helps converting between Linux libnuma unsigned long masks
* and hwloc cpusets and nodesets.
*
* \note Topology \p topology must match the current machine.
*
* \note The behavior of libnuma is undefined if the kernel is not NUMA-aware.
* (when CONFIG_NUMA is not set in the kernel configuration).
* This helper and libnuma may thus not be strictly compatible in this case,
* which may be detected by checking whether numa_available() returns -1.
*
* @{
*/
```

这个 `\defgroup` 定义了一个名为 "hwlocality_linux_libnuma_ulongs" 的组，其中包含一个名为 "hwlocality_linux_libnuma_ulongs" 的函数。这个函数帮助在 hwloc 堆栈上转换 Linux libnuma 中的 unsigned long 掩码和 hwloc 中的 cpuset 和 nodeset。

```cpp
/**
* @brief 此函数将 libnuma 中的 unsigned long 掩码转换为 hwloc 中的 cpuset 和 nodeset。
*
* @param[out] cpuset
* @param[out] nodeset
* @param[out] mask
* @param[out] data
*
* @retval libnuma 中的 unsigned long 掩码
* @retval hwloc 中的 cpuset 和 nodeset
*
* @throws[out] libnuma 中的错误
*/
```

这段代码的实现可能如下：

```cpp
int hwlocality_linux_libnuma_ulongs(hwloc_device_t device, const hwloc_radiation_domain_t topology,
                                           const uint64_t data[], size_t data_size,
                                           hwloc_cpub_t *cpuset,
                                           hwloc_nodepath_t *nodeset)
```

注意，由于 libnuma 的行为是不确定的，因此上述代码的实现可能会是错误的。


```cpp
#define HWLOC_LINUX_LIBNUMA_H

#include "hwloc.h"

#include <numa.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_linux_libnuma_ulongs Interoperability with Linux libnuma unsigned long masks
 *
 * This interface helps converting between Linux libnuma unsigned long masks
 * and hwloc cpusets and nodesets.
 *
 * \note Topology \p topology must match the current machine.
 *
 * \note The behavior of libnuma is undefined if the kernel is not NUMA-aware.
 * (when CONFIG_NUMA is not set in the kernel configuration).
 * This helper and libnuma may thus not be strictly compatible in this case,
 * which may be detected by checking whether numa_available() returns -1.
 *
 * @{
 */


```

这段代码定义了一个名为 `hwloc_cpuset_to_linux_libnuma_ulongs` 的函数，它接受一个硬件局部拓扑结构（hwloc_topology_t）和一个CPU集合（hwloc_const_cpuset_t）作为输入参数，并将CPUSET转换为在无符号长整数数组中的 mask 和最大节点数的组合，最后将结果返回。

具体来说，函数首先定义了一个名为 `outmaxnode` 的变量，用于跟踪在已找到的最大节点数之前可能的最大节点数，然后定义了一个名为 `mask` 的无符号长整数数组，用于存储已经设置的CPU集合中的每一位的掩码。接着，定义了一个名为 `depth` 的整数，用于获取拓扑结构的深度，并将其设置为 2，以确保可以正确地获取最大节点数。

在函数实现中，首先调用了 `hwloc_get_type_depth` 函数来获取拓扑结构深度，并将其存储在 `depth` 变量中。接着，定义了一个 `node` 变量，用于跟踪当前正在遍历的CPU集合中的节点。然后，使用 while 循环遍历CPUSET中的所有节点，并在遍历过程中检查当前节点是否已经遍历过所有的mask位。如果是，则说明已经遍历到了一个节点，可以跳过这个节点；否则，将当前节点的 `os_index` 存储在 `mask` 数组中，并将 `outmaxnode` 设置为当前节点在 `mask` 数组中的位置，以便在遍历过程中更新最大节点数。最后，在遍历过程中结束后，将 `outmaxnode` 加 1，以确保可以正确地获取最大节点数，并返回 0。


```cpp
/** \brief Convert hwloc CPU set \p cpuset into the array of unsigned long \p mask
 *
 * \p mask is the array of unsigned long that will be filled.
 * \p maxnode contains the maximal node number that may be stored in \p mask.
 * \p maxnode will be set to the maximal node number that was found, plus one.
 *
 * This function may be used before calling set_mempolicy, mbind, migrate_pages
 * or any other function that takes an array of unsigned long and a maximal
 * node number as input parameter.
 */
static __hwloc_inline int
hwloc_cpuset_to_linux_libnuma_ulongs(hwloc_topology_t topology, hwloc_const_cpuset_t cpuset,
				    unsigned long *mask, unsigned long *maxnode)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  unsigned long outmaxnode = -1;
  hwloc_obj_t node = NULL;

  /* round-up to the next ulong and clear all bytes */
  *maxnode = (*maxnode + 8*sizeof(*mask) - 1) & ~(8*sizeof(*mask) - 1);
  memset(mask, 0, *maxnode/8);

  while ((node = hwloc_get_next_obj_covering_cpuset_by_depth(topology, cpuset, depth, node)) != NULL) {
    if (node->os_index >= *maxnode)
      continue;
    mask[node->os_index/sizeof(*mask)/8] |= 1UL << (node->os_index % (sizeof(*mask)*8));
    if (outmaxnode == (unsigned long) -1 || outmaxnode < node->os_index)
      outmaxnode = node->os_index;
  }

  *maxnode = outmaxnode+1;
  return 0;
}

```

This is a C function that takes a hardware location topology, a nodeset (a set of node indices), and an array of unsigned long (device mask) as input, and converts it into an array of unsigned long at the given nodeset using the hardware location topology.

It works as follows:

* It sets the output mask to the bitmask of the highest node number that can be represented in the given mask.
* It iterates through all nodes, checking if the node is a part of the nodeset.
* If the node is not a part of the nodeset, it skips.
* If the node is a part of the nodeset and the corresponding bit of the mask is set, it sets the corresponding bit of the output mask.
* It keeps track of the highest node number that has been found so far.
* When all nodes have been processed, the loop continues to the next node.
* If the output mask is not set yet and a node is found to have the highest node number, the function returns 0 to indicate that the conversion was successful.

This function can be used before calling `set_mempolicy`, `mbind`, `migrate_pages`, or any other function that takes an array of unsigned long and a maximal node number as input.


```cpp
/** \brief Convert hwloc NUMA node set \p nodeset into the array of unsigned long \p mask
 *
 * \p mask is the array of unsigned long that will be filled.
 * \p maxnode contains the maximal node number that may be stored in \p mask.
 * \p maxnode will be set to the maximal node number that was found, plus one.
 *
 * This function may be used before calling set_mempolicy, mbind, migrate_pages
 * or any other function that takes an array of unsigned long and a maximal
 * node number as input parameter.
 */
static __hwloc_inline int
hwloc_nodeset_to_linux_libnuma_ulongs(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset,
				      unsigned long *mask, unsigned long *maxnode)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  unsigned long outmaxnode = -1;
  hwloc_obj_t node = NULL;

  /* round-up to the next ulong and clear all bytes */
  *maxnode = (*maxnode + 8*sizeof(*mask) - 1) & ~(8*sizeof(*mask) - 1);
  memset(mask, 0, *maxnode/8);

  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL) {
    if (node->os_index >= *maxnode)
      continue;
    if (!hwloc_bitmap_isset(nodeset, node->os_index))
      continue;
    mask[node->os_index/sizeof(*mask)/8] |= 1UL << (node->os_index % (sizeof(*mask)*8));
    if (outmaxnode == (unsigned long) -1 || outmaxnode < node->os_index)
      outmaxnode = node->os_index;
  }

  *maxnode = outmaxnode+1;
  return 0;
}

```

这段代码定义了一个名为 `hwloc_cpuset_from_linux_libnuma_ulongs` 的函数，它接受一个无符号长整型数组 `mask` 和一个最大节点数 `maxnode` 作为输入参数。函数的目的是将 `mask` 中的无符号长整型数组转换为有符号长整型数组，并将其存储在指定的 `cpuset` 中。

具体来说，函数首先通过 `hwloc_get_type_depth` 函数获取输入 `topology` 的类型深度，然后创建一个 `hwloc_obj_t` 类型的 `node` 变量，将其初始化为 `NULL`。接下来，函数使用一个循环遍历 `mask` 数组，并将指定的 `mask` 元素与输入 `mask` 中的相应元素按位与，并将结果存储在 `cpuset` 中。

最后，函数返回一个整数表示 `hwloc_cpuset_from_linux_libnuma_ulongs` 函数的执行结果，即 `0`。


```cpp
/** \brief Convert the array of unsigned long \p mask into hwloc CPU set
 *
 * \p mask is a array of unsigned long that will be read.
 * \p maxnode contains the maximal node number that may be read in \p mask.
 *
 * This function may be used after calling get_mempolicy or any other function
 * that takes an array of unsigned long as output parameter (and possibly
 * a maximal node number as input parameter).
 */
static __hwloc_inline int
hwloc_cpuset_from_linux_libnuma_ulongs(hwloc_topology_t topology, hwloc_cpuset_t cpuset,
				      const unsigned long *mask, unsigned long maxnode)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  hwloc_obj_t node = NULL;
  hwloc_bitmap_zero(cpuset);
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (node->os_index < maxnode
	&& (mask[node->os_index/sizeof(*mask)/8] & (1UL << (node->os_index % (sizeof(*mask)*8)))))
      hwloc_bitmap_or(cpuset, cpuset, node->cpuset);
  return 0;
}

```

这段代码定义了一个名为 `hwloc_nodeset_from_linux_libnuma_ulongs` 的函数，它的作用是将一个已经定义好的无符号长整数数组（ `mask`）转换为一个硬件locality NUMA 节点集合。

具体来说，这个函数的输入参数包括一个 `hwloc_topology_t` 类型的顶类和一个 `hwloc_nodeset_t` 类型的节点集，一个指向无符号长整数数组 `mask` 的指针，和一个最大节点数 `maxnode`。函数首先定义了一个名为 `depth` 的整数变量，用于获取输入 `mask` 的最大节点数。然后定义了一个名为 `node` 的整数变量和一个名为 `nodeset` 的 `hwloc_nodeset_t` 类型的变量，用于保存转换后的节点集。

接下来，函数从 `mask` 数组的第一个元素开始，遍历每一个元素，并且在遍历过程中，使用 bitmap 函数将当前节点加入节点集中。具体来说，对于每一个元素，首先判断它是否属于 `mask` 数组中当前节点的位置，如果是，则将该节点标记为 `set`，最后再将 `mask` 中对应位置的位设置为 `1`，以便在转换过程中正确地处理零填充。

函数的返回值是一个整数，表示转换是否成功。如果转换成功，则返回 0；如果转换失败，则返回 `-1`，因为 `hwloc_get_type_depth` 函数在尝试获取 `HWLOC_OBJ_NUMANODE` 类型的节点时可能会失败。


```cpp
/** \brief Convert the array of unsigned long \p mask into hwloc NUMA node set
 *
 * \p mask is a array of unsigned long that will be read.
 * \p maxnode contains the maximal node number that may be read in \p mask.
 *
 * This function may be used after calling get_mempolicy or any other function
 * that takes an array of unsigned long as output parameter (and possibly
 * a maximal node number as input parameter).
 */
static __hwloc_inline int
hwloc_nodeset_from_linux_libnuma_ulongs(hwloc_topology_t topology, hwloc_nodeset_t nodeset,
					const unsigned long *mask, unsigned long maxnode)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  hwloc_obj_t node = NULL;
  hwloc_bitmap_zero(nodeset);
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (node->os_index < maxnode
	&& (mask[node->os_index/sizeof(*mask)/8] & (1UL << (node->os_index % (sizeof(*mask)*8)))))
      hwloc_bitmap_set(nodeset, node->os_index);
  return 0;
}

```

这段代码定义了一个名为`hwlocality_linux_libnuma_bitmask`的接口，用于将Linux libnuma bitmasks和hwloc cpusets与节点集之间进行转换。

具体来说，这段代码实现了一个名为`convert_to_hwloc`的函数，该函数接收两个参数：`libnuma_bitmask`和`host_cpu_set_ bitmask`。函数先尝试使用libnuma本身的bitmasks，如果不能成功，则尝试使用操作系统提供的hwloc cpusets和节点集。

注意，在使用libnuma进行topology转换时，必须确保当前机器的topology与libnuma的topology相匹配，否则可能出现 undefined的行为。另外，libnuma的实现可能会受到操作系统和硬件平台的影响，因此在使用此函数时，需要进行充分的测试以保证其正确性。


```cpp
/** @} */



/** \defgroup hwlocality_linux_libnuma_bitmask Interoperability with Linux libnuma bitmask
 *
 * This interface helps converting between Linux libnuma bitmasks
 * and hwloc cpusets and nodesets.
 *
 * \note Topology \p topology must match the current machine.
 *
 * \note The behavior of libnuma is undefined if the kernel is not NUMA-aware.
 * (when CONFIG_NUMA is not set in the kernel configuration).
 * This helper and libnuma may thus not be strictly compatible in this case,
 * which may be detected by checking whether numa_available() returns -1.
 *
 * @{
 */


```

这段代码定义了一个名为 `hwloc_cpuset_to_linux_libnuma_bitmask` 的函数，它的功能是将指定的 `cpuset` 转换为 `libnuma` 中的位图掩码，并返回一个新的 `struct bitmask` 类型的变量。

该函数首先定义了一个名为 `hwloc_cpuset_to_linux_libnuma_bitmask` 的函数，它的参数为 `hwloc_topology_t` 类型的 topology 和 `hwloc_const_cpuset_t` 类型的 cpuset。这两个参数分别表示要转换的 CPU 集合和已知的 CPU 集合。函数返回一个 newly 被分配的 `struct bitmask` 类型的变量，如果返回值为 NULL，则说明转换失败。

接着，在另一个名为 `hwloc_cpuset_to_linux_libnuma_bitmask` 的函数中，也定义了相同的参数，但返回值类型为 `struct bitmask *`，说明该函数返回的是一个指向 `struct bitmask` 的指针类型。这两个函数的作用是相似的，都是一个将 CPU 集合转换为 bitmask 的函数，但返回类型不同，需要根据实际调用场景进行选择。


```cpp
/** \brief Convert hwloc CPU set \p cpuset into the returned libnuma bitmask
 *
 * The returned bitmask should later be freed with numa_bitmask_free.
 *
 * This function may be used before calling many numa_ functions
 * that use a struct bitmask as an input parameter.
 *
 * \return newly allocated struct bitmask.
 */
static __hwloc_inline struct bitmask *
hwloc_cpuset_to_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_const_cpuset_t cpuset) __hwloc_attribute_malloc;
static __hwloc_inline struct bitmask *
hwloc_cpuset_to_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_const_cpuset_t cpuset)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  hwloc_obj_t node = NULL;
  struct bitmask *bitmask = numa_allocate_cpumask();
  if (!bitmask)
    return NULL;
  while ((node = hwloc_get_next_obj_covering_cpuset_by_depth(topology, cpuset, depth, node)) != NULL)
    if (node->attr->numanode.local_memory)
      numa_bitmask_setbit(bitmask, node->os_index);
  return bitmask;
}

```

这段代码定义了一个名为 `hwloc_nodeset_to_linux_libnuma_bitmask` 的函数，它的作用是将从 `hwloc_topology_t` 类型的 topology 和 `hwloc_const_nodeset_t` 类型的 nodeset 转换为一个名为 `struct bitmask` 的类结构，该结构中包含一个指向 `hwloc_obj_t` 类型的 `node` 类型的指针，以及一个布尔类型的字段，表示相应的 NUMA 节点是否在本地内存中。

该函数首先定义了一个名为 `hwloc_nodeset_to_linux_libnuma_bitmask` 的函数，它的实现与上面的 `hwloc_nodeset_to_linux_libnuma_bitmask` 函数完全相同，只是将函数名称与实现函数名称进行了对应。

然后，在 `hwloc_nodeset_to_linux_libnuma_bitmask` 函数中，定义了一个名为 `numa_allocate_cpumask` 的函数，用于从系统内存中分配一个掩码，并将其返回。如果分配失败，函数返回 NULL。

接着，在 `hwloc_nodeset_to_linux_libnuma_bitmask` 函数中，定义了一个名为 `hwloc_bitmap_isset` 的函数，用于检查给定的节点是否属于指定的 `hwloc_const_nodeset_t`。

最后，在 `hwloc_nodeset_to_linux_libnuma_bitmask` 函数中，定义了一个名为 `numa_bitmask_setbit` 的函数，用于设置指定的 NUMA 节点在本地内存中的掩码位。

总之，这段代码定义了一个函数 `hwloc_nodeset_to_linux_libnuma_bitmask`，它可以将 `hwloc_topology_t` 类型的 topology 和 `hwloc_const_nodeset_t` 类型的 nodeset 转换为一个名为 `struct bitmask` 的类结构，该结构中包含一个指向 `hwloc_obj_t` 类型的 `node` 类型的指针，以及一个布尔类型的字段，表示相应的 NUMA 节点是否在本地内存中。


```cpp
/** \brief Convert hwloc NUMA node set \p nodeset into the returned libnuma bitmask
 *
 * The returned bitmask should later be freed with numa_bitmask_free.
 *
 * This function may be used before calling many numa_ functions
 * that use a struct bitmask as an input parameter.
 *
 * \return newly allocated struct bitmask.
 */
static __hwloc_inline struct bitmask *
hwloc_nodeset_to_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset) __hwloc_attribute_malloc;
static __hwloc_inline struct bitmask *
hwloc_nodeset_to_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  hwloc_obj_t node = NULL;
  struct bitmask *bitmask = numa_allocate_cpumask();
  if (!bitmask)
    return NULL;
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (hwloc_bitmap_isset(nodeset, node->os_index) && node->attr->numanode.local_memory)
      numa_bitmask_setbit(bitmask, node->os_index);
  return bitmask;
}

```

这段代码是一个高阶函数，它接收一个高阶函数（hwloc_topology_t）和一个表示特定数据结构的指针（hwloc_cpuset_t）。这个函数的目的是接收一个通过 libnuma 库定义的 bitmask，然后将其转换为特定数据结构的 cpuset。

实现中，首先定义了一个名为 hwloc_cpuset_from_linux_libnuma_bitmask 的函数，它接收两个参数：topology 和 bitmask。函数内部首先定义了一个名为 depth 的整数变量，用于获取输入 topology 的高度。接下来，定义了一个名为 node 的整数变量，用于存储当前输入 topology 中的对象。然后，定义了一个名为 cpuset 的整数变量，用于存储要复制的 cpuset。

接下来，函数内部使用 hwloc_get_type_depth 函数获取 topology 的类型，并使用 hwloc_get_next_obj_by_depth 函数获取 depth 层的一个对象。如果当前对象是 numa 数组的对象，函数会遍历该数组的所有元素，并将它们与 bitmask 中的所有元素进行 or 运算。最后，函数返回 0，表示成功地将 bitmask 转换为 cpuset。


```cpp
/** \brief Convert libnuma bitmask \p bitmask into hwloc CPU set \p cpuset
 *
 * This function may be used after calling many numa_ functions
 * that use a struct bitmask as an output parameter.
 */
static __hwloc_inline int
hwloc_cpuset_from_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_cpuset_t cpuset,
					const struct bitmask *bitmask)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  hwloc_obj_t node = NULL;
  hwloc_bitmap_zero(cpuset);
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (numa_bitmask_isbitset(bitmask, node->os_index))
      hwloc_bitmap_or(cpuset, cpuset, node->cpuset);
  return 0;
}

```

这段代码是一个C语言函数，名为`hwloc_nodeset_from_linux_libnuma_bitmask`，属于`hwloc_topology_ex`库。它接受一个`hwloc_topology_t`类型的顶部落门和一个`hwloc_nodeset_t`类型的节点集，以及一个`const struct bitmask *`类型的位图掩码。

该函数的作用是将输入的位图掩码转换为硬件定位单元（NUMA）节点集，并将其存储在指定的节点集中。这个函数可以在调用许多使用`struct bitmask`作为输出参数的`hwloc_numa_function`之后使用。

具体实现过程如下：

1. 获取输入的顶部落门和节点集。
2. 将节点集中所有操作系统索引不在此位图掩码范围内的元素设置为0。
3. 遍历节点集中的每个节点，并检查当前节点是否具有位图掩码中指定的元素。如果是，则将该节点在节点集中设置为当前节点在位图中的索引。
4. 返回0表示成功转换。


```cpp
/** \brief Convert libnuma bitmask \p bitmask into hwloc NUMA node set \p nodeset
 *
 * This function may be used after calling many numa_ functions
 * that use a struct bitmask as an output parameter.
 */
static __hwloc_inline int
hwloc_nodeset_from_linux_libnuma_bitmask(hwloc_topology_t topology, hwloc_nodeset_t nodeset,
					 const struct bitmask *bitmask)
{
  int depth = hwloc_get_type_depth(topology, HWLOC_OBJ_NUMANODE);
  hwloc_obj_t node = NULL;
  hwloc_bitmap_zero(nodeset);
  while ((node = hwloc_get_next_obj_by_depth(topology, depth, node)) != NULL)
    if (numa_bitmask_isbitset(bitmask, node->os_index))
      hwloc_bitmap_set(nodeset, node->os_index);
  return 0;
}

```

这段代码是一个C/C++语言的包含头文件，它定义了一个名为`__cplusplus`的标识符。该标识符是一个预处理指令，它告诉编译器在编译之前需要处理`__cplusplus`定义的预处理指令。

具体来说，该代码将定义一个名为`__cplusplus`的常量，该常量的值为`1`。这意味着编译器在编译任何包含`__cplusplus`定义的预处理指令之前，必须先处理`__cplusplus`定义的预处理指令。

该代码还包含一个名为`extern "C"`的定义，这意味着它告诉编译器它将定义一个C语言的函数。如果编译器在后续的编译过程中无法找到定义该函数的源文件，它将无法提供调试信息，因此`extern "C"`在这里的作用是告诉编译器它将定义一个名为`C`的函数，而不是`C++`。


```cpp
/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_LINUX_NUMA_H */

```

# `src/3rdparty/hwloc/include/hwloc/linux.h`

这段代码定义了一个名为 "hwloc_linux.h" 的头文件，其中包含了一些宏，旨在帮助在 hwloc 和 Linux 之间进行交互。

具体来说，这些宏可以用于在 hwloc 配置时生成一些在 Linux 内有效的标志和环境变量，例如平台特定的硬件设备名称、hwloc 配置文件的路径等。

因此，这段代码定义了一个用于在 hwloc 和 Linux 之间进行交互的框架，通过使用这些宏，用户可以使用一些低级别的 Linux 特性，从而更方便地使用 hwloc。


```cpp
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2021 Inria.  All rights reserved.
 * Copyright © 2009-2011 Université Bordeaux
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Macros to help interaction between hwloc and Linux.
 *
 * Applications that use hwloc on Linux may want to include this file
 * if using some low-level Linux features.
 */

#ifndef HWLOC_LINUX_H
```

这段代码是一个C/C++语言的预处理指令，它定义了一个名为"hwlocality_linux"的头部文件。通过预处理指令，源代码在编译之前被替换为以下内容：
```cppc
#include "hwloc.h"

#include <stdio.h>
```
这里定义了一个名为"hwlocality_linux"的文件头，它包含了hwloc库中的所有定义。这个头部文件是在包含hwloc库的源文件之前被定义的，因此它必须是库的依赖项。

接下来的两个包含头文件的代码块是在定义了一些库函数，它们可以被用来在程序中使用。这里我们看到了两个函数：`__attribute__((interface))` 和 `__attribute__((provisional))`。它们都被`extern "C"` 修饰，这意味着它们可能会被外部库或用户态程序使用。不过，它们的作用在定义中并未给出，因此我们需要了解库或用户态程序如何使用这些函数。

最后一个包含头文件的代码块定义了一个`extern` 函数，它的作用是在`__cplusplus` 预处理指令之前保留对C/C++源代码的访问权限。


```cpp
#define HWLOC_LINUX_H

#include "hwloc.h"

#include <stdio.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_linux Linux-specific helpers
 *
 * This includes helpers for manipulating Linux kernel cpumap files, and hwloc
 * equivalents of the Linux sched_setaffinity and sched_getaffinity system calls.
 *
 * @{
 */

```

这段代码定义了一个名为 `hwloc_linux_set_tid_cpubind` 的函数，它接受一个 `hwloc_topology_t` 的上下文，和一个 `pid_t` 的整数参数 `tid`，以及一个 `hwloc_const_cpuset_t` 的设置标志。它的作用是设置指定 `pid_t` 所对应的 `hwloc_const_cpuset_t` 中包含的 CPU集。

在函数体中，首先定义了一个名为 `hwloc_dclspec_linux_set_tid_cpubind` 的函数，它的作用与上面定义的函数相同。然后定义了一个名为 `hwloc_get_proc_cpubind` 的函数，它的作用是获取指定 `pid_t` 所对应的 `hwloc_const_cpuset_t` 中包含的 CPU集。


```cpp
/** \brief Bind a thread \p tid on cpus given in cpuset \p set
 *
 * The behavior is exactly the same as the Linux sched_setaffinity system call,
 * but uses a hwloc cpuset.
 *
 * \note This is equivalent to calling hwloc_set_proc_cpubind() with
 * HWLOC_CPUBIND_THREAD as flags.
 */
HWLOC_DECLSPEC int hwloc_linux_set_tid_cpubind(hwloc_topology_t topology, pid_t tid, hwloc_const_cpuset_t set);

/** \brief Get the current binding of thread \p tid
 *
 * The CPU-set \p set (previously allocated by the caller)
 * is filled with the list of PUs which the thread
 * was last bound to.
 *
 * The behavior is exactly the same as the Linux sched_getaffinity system call,
 * but uses a hwloc cpuset.
 *
 * \note This is equivalent to calling hwloc_get_proc_cpubind() with
 * ::HWLOC_CPUBIND_THREAD as flags.
 */
```

这两段代码是用来在 hwloc-linux 库中获取 CPU 线程在哪个物理 CPU 上运行，并返回其位置的函数。

第一段代码 `hwloc_linux_get_tid_cpubind` 接受一个 `hwloc_topology_t` 类型的上下文，一个 `pid_t` 类型的线程 ID，和一个 `hwloc_cpuset_t` 类型的设置。它返回的是线程在哪个物理 CPU 上运行，以及在该物理 CPU 上运行的 CPU 数量。

第二段代码 `hwloc_linux_get_tid_last_cpu_location` 与第一段代码类似，但返回的是一个 `hwloc_bitmap_t` 类型的设置，其中包含线程在哪个物理 CPU 上运行。这个函数可以用来从系统日志或其他类似的信息中获取线程在哪个物理 CPU 上运行，从而在编写程序或配置系统时进行一些调整。

这两段代码的作用是帮助用户更方便地获取在哪个物理 CPU 上运行的线程，以及在该物理 CPU 上运行的 CPU 数量。


```cpp
HWLOC_DECLSPEC int hwloc_linux_get_tid_cpubind(hwloc_topology_t topology, pid_t tid, hwloc_cpuset_t set);

/** \brief Get the last physical CPU where thread \p tid ran.
 *
 * The CPU-set \p set (previously allocated by the caller)
 * is filled with the PU which the thread last ran on.
 *
 * \note This is equivalent to calling hwloc_get_proc_last_cpu_location() with
 * ::HWLOC_CPUBIND_THREAD as flags.
 */
HWLOC_DECLSPEC int hwloc_linux_get_tid_last_cpu_location(hwloc_topology_t topology, pid_t tid, hwloc_bitmap_t set);

/** \brief Convert a linux kernel cpumask file \p path into a hwloc bitmap \p set.
 *
 * Might be used when reading CPU set from sysfs attributes such as topology
 * and caches for processors, or local_cpus for devices.
 *
 * \note This function ignores the HWLOC_FSROOT environment variable.
 */
```

这段代码定义了一个名为 `hwloc_linux_read_path_as_cpumask` 的函数，属于 `hwloc_linux_h` 库。它接受一个字符指针 `path`，并返回一个 `hwloc_bitmap_t` 类型的掩码，表示哪些 `hwloc_和工作区` 被设置为真。

函数的作用是读取一个路径，并将路径对应的 `hwloc_工作区` 和 `hwloc_计算区域` 设置为设置的计算区域。设置计算区域的掩码可以通过 `hwloc_linux_set_export_mask` 函数实现，但这个库函数没有提供对应的实现，因此这段代码只能通过这种形式来定义。


```cpp
HWLOC_DECLSPEC int hwloc_linux_read_path_as_cpumask(const char *path, hwloc_bitmap_t set);

/** @} */


#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_LINUX_H */

```

# `src/3rdparty/hwloc/include/hwloc/memattrs.h`

这段代码是一个C++语言的定义头文件，其中定义了一个名为`HWLOC_MEMATTR_H`的函数。该函数的主要作用是定义与`hwloc`库中`memattr`相关的一些内存节点属性。

具体来说，这个函数表明了以下几点：

1. 该函数是定义在`/path/to/your/project/including/hwloc.h`头文件中的。注意，`/path/to/your/project`应该是你项目的实际路径。

2. 函数的主要目的是为了支持`hwloc`库的编译。`hwloc`库是一个用于管理硬件加速器的工具链，它可以帮助开发者更轻松地管理硬件设备和驱动程序。这个库提供了一个高度抽象的接口，允许开发者专注于编写代码，而无需关心硬件的细节。

3. `HWLOC_MEMATTR_H`函数是一个函数，它接受了`hwloc.h`头文件作为第一个参数。这意味着，如果你在这个头文件中定义了`HWLOC_MEMATTR_H`函数，那么`hwloc.h`头文件将包含定义。

4. 该函数使用了`#ifdef`和`#define`预处理指令。`#ifdef`用于检查`hwloc.h`头文件是否已经被定义，如果是，则编译器会编译`HWLOC_MEMATTR_H`函数。`#define`则是用于定义宏，将`HWLOC_MEMATTR_H`函数的名称替换为`defined`，这样编译器就会识别`HWLOC_MEMATTR_H`函数了。

总之，这段代码定义了一个名为`HWLOC_MEMATTR_H`的函数，它用于定义与`hwloc`库中`memattr`相关的一些内存节点属性。这个函数可能会被用于编译`hwloc`库，或者在需要使用`hwloc`库的其他部分代码中中被调用。


```cpp
/*
 * Copyright © 2019-2022 Inria.  All rights reserved.
 * See COPYING in top-level directory.
 */

/** \file
 * \brief Memory node attributes.
 */

#ifndef HWLOC_MEMATTR_H
#define HWLOC_MEMATTR_H

#include "hwloc.h"

#ifdef __cplusplus
```

This is a description of memory attributes for a target memory node. These attributes are used by the CPU to determine the best memory node to access for a particular memory operation. The attributes are evaluated by the CPU to determine the best target for accessing the memory.

The attributes include:

1. Capacity: The maximum amount of memory that can be assigned to this target. This is an important attribute for programs that have a large amount of data.

2. Latency: The average memory access time. This is an important attribute for programs that have a large number of small memory accesses.

3. Bandwidth: The total amount of data that can be transferred from the target to the CPU. This is an important attribute for programs that perform large data transfers.

4. Read throughput: The amount of data that can be transferred from the target to the CPU in a single read operation.

5. Write throughput: The amount of data that can be transferred from the CPU to the target in a single write operation.

6. Errors: The number of memory errors that have occurred while accessing the memory.

These attributes are evaluated by the CPU to determine the best target for accessing the memory. The CPU will compare the attributes of different memory nodes to determine which one has the best characteristics for the current memory operation. The CPU may also apply other attributes that are specific to the current memory operation, such as the number of times the memory node has been accessed or the memory node's physical memory.

The memory attributes are also used by the hardware to determine the best memory node to access for memory operations. The hardware will compare the attributes of different memory nodes to determine which one has the best characteristics for the current memory operation. The hardware may also apply other attributes that are specific to the current memory operation, such as the number of times the memory node has been accessed or the memory node's physical memory.

The memory attributes provide information about the memory nodes that can be used by the CPU and the hardware for memory operations. The CPU and hardware use these attributes to determine the best memory node to access for memory operations, which can improve the performance of the current memory operation.


```cpp
extern "C" {
#elif 0
}
#endif

/** \defgroup hwlocality_memattrs Comparing memory node attributes for finding where to allocate on
 *
 * Platforms with heterogeneous memory require ways to decide whether
 * a buffer should be allocated on "fast" memory (such as HBM),
 * "normal" memory (DDR) or even "slow" but large-capacity memory
 * (non-volatile memory).
 * These memory nodes are called "Targets" while the CPU accessing them
 * is called the "Initiator". Access performance depends on their
 * locality (NUMA platforms) as well as the intrinsic performance
 * of the targets (heterogeneous platforms).
 *
 * The following attributes describe the performance of memory accesses
 * from an Initiator to a memory Target, for instance their latency
 * or bandwidth.
 * Initiators performing these memory accesses are usually some PUs or Cores
 * (described as a CPU set).
 * Hence a Core may choose where to allocate a memory buffer by comparing
 * the attributes of different target memory nodes nearby.
 *
 * There are also some attributes that are system-wide.
 * Their value does not depend on a specific initiator performing
 * an access.
 * The memory node Capacity is an example of such attribute without
 * initiator.
 *
 * One way to use this API is to start with a cpuset describing the Cores where
 * a program is bound. The best target NUMA node for allocating memory in this
 * program on these Cores may be obtained by passing this cpuset as an initiator
 * to hwloc_memattr_get_best_target() with the relevant memory attribute.
 * For instance, if the code is latency limited, use the Latency attribute.
 *
 * A more flexible approach consists in getting the list of local NUMA nodes
 * by passing this cpuset to hwloc_get_local_numanode_objs().
 * Attribute values for these nodes, if any, may then be obtained with
 * hwloc_memattr_get_value() and manually compared with the desired criteria.
 *
 * \sa An example is available in doc/examples/memory-attributes.c in the source tree.
 *
 * \note The API also supports specific objects as initiator,
 * but it is currently not used internally by hwloc.
 * Users may for instance use it to provide custom performance
 * values for host memory accesses performed by GPUs.
 *
 * \note The interface actually also accepts targets that are not NUMA nodes.
 * @{
 */

```

In the context of the HWLOC_MEMATTR class, the `Max` property refers to the maximum latency that can be observed for all memory access types.

So, in the example you provided, the `HWLOC_MEMATTR_ID_MAX` value is the maximum latency that can be observed for all memory access types.

If you want to optimize your memory accesses, you should try to minimize the latency by using techniques such as batching multiple accesses into a single higher-bandwidth transfer, or by using an accelerator that is optimized for your use case.


```cpp
/** \brief Memory node attributes. */
enum hwloc_memattr_id_e {
  /** \brief
   * The \"Capacity\" is returned in bytes (local_memory attribute in objects).
   *
   * Best capacity nodes are nodes with <b>higher capacity</b>.
   *
   * No initiator is involved when looking at this attribute.
   * The corresponding attribute flags are ::HWLOC_MEMATTR_FLAG_HIGHER_FIRST.
   * \hideinitializer
   */
  HWLOC_MEMATTR_ID_CAPACITY = 0,

  /** \brief
   * The \"Locality\" is returned as the number of PUs in that locality
   * (e.g. the weight of its cpuset).
   *
   * Best locality nodes are nodes with <b>smaller locality</b>
   * (nodes that are local to very few PUs).
   * Poor locality nodes are nodes with larger locality
   * (nodes that are local to the entire machine).
   *
   * No initiator is involved when looking at this attribute.
   * The corresponding attribute flags are ::HWLOC_MEMATTR_FLAG_HIGHER_FIRST.
   * \hideinitializer
   */
  HWLOC_MEMATTR_ID_LOCALITY = 1,

  /** \brief
   * The \"Bandwidth\" is returned in MiB/s, as seen from the given initiator location.
   *
   * Best bandwidth nodes are nodes with <b>higher bandwidth</b>.
   *
   * The corresponding attribute flags are ::HWLOC_MEMATTR_FLAG_HIGHER_FIRST
   * and ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR.
   *
   * This is the average bandwidth for read and write accesses. If the platform
   * provides individual read and write bandwidths but no explicit average value,
   * hwloc computes and returns the average.
   * \hideinitializer
   */
  HWLOC_MEMATTR_ID_BANDWIDTH = 2,

  /** \brief
   * The \"ReadBandwidth\" is returned in MiB/s, as seen from the given initiator location.
   *
   * Best bandwidth nodes are nodes with <b>higher bandwidth</b>.
   *
   * The corresponding attribute flags are ::HWLOC_MEMATTR_FLAG_HIGHER_FIRST
   * and ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR.
   * \hideinitializer
   */
  HWLOC_MEMATTR_ID_READ_BANDWIDTH = 4,

  /** \brief
   * The \"WriteBandwidth\" is returned in MiB/s, as seen from the given initiator location.
   *
   * Best bandwidth nodes are nodes with <b>higher bandwidth</b>.
   *
   * The corresponding attribute flags are ::HWLOC_MEMATTR_FLAG_HIGHER_FIRST
   * and ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR.
   * \hideinitializer
   */
  HWLOC_MEMATTR_ID_WRITE_BANDWIDTH = 5,

  /** \brief
   * The \"Latency\" is returned as nanoseconds, as seen from the given initiator location.
   *
   * Best latency nodes are nodes with <b>smaller latency</b>.
   *
   * The corresponding attribute flags are ::HWLOC_MEMATTR_FLAG_LOWER_FIRST
   * and ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR.
   *
   * This is the average latency for read and write accesses. If the platform
   * provides individual read and write latencies but no explicit average value,
   * hwloc computes and returns the average.
   * \hideinitializer
   */
  HWLOC_MEMATTR_ID_LATENCY = 3,

  /** \brief
   * The \"ReadLatency\" is returned as nanoseconds, as seen from the given initiator location.
   *
   * Best latency nodes are nodes with <b>smaller latency</b>.
   *
   * The corresponding attribute flags are ::HWLOC_MEMATTR_FLAG_LOWER_FIRST
   * and ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR.
   * \hideinitializer
   */
  HWLOC_MEMATTR_ID_READ_LATENCY = 6,

  /** \brief
   * The \"WriteLatency\" is returned as nanoseconds, as seen from the given initiator location.
   *
   * Best latency nodes are nodes with <b>smaller latency</b>.
   *
   * The corresponding attribute flags are ::HWLOC_MEMATTR_FLAG_LOWER_FIRST
   * and ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR.
   * \hideinitializer
   */
  HWLOC_MEMATTR_ID_WRITE_LATENCY = 7,

  /* TODO persistence? */

  HWLOC_MEMATTR_ID_MAX /**< \private Sentinel value */
};

```

这段代码定义了一个名为`hwloc_memattr_id_t`的枚举类型，用于表示内存属性标识。这个标识可能是一个已经存在的内存属性ID，也有可能是通过`hwloc_memattr_register()`函数返回的新ID。

接下来，定义了一个名为`hwloc_topology_t`的枚举类型` enum hwloc_location_type_e`，用于表示位置的类型。这个枚举类型有两个成员变量，`HWLOC_LOCATION_TYPE_CPUSET`表示位置是一个CPU集，`HWLOC_LOCATION_TYPE_OBJECT`表示位置是一个对象。

最后，定义了一个名为`hwloc_memattr_get_by_name`的函数，接受一个 topology 参数和一个 name 参数，返回给定 name 内存属性的 identifier。这个函数使用 hwloc_topology_t 中的函数`hwloc_memattr_get_by_name()`实现，这个函数将 topology 和 name 参数传递给`hwloc_memattr_get()`函数，根据上下文确定要返回的内存属性的位置类型，然后返回相应的 identifier。


```cpp
/** \brief A memory attribute identifier.
 * May be either one of ::hwloc_memattr_id_e or a new id returned by hwloc_memattr_register().
 */
typedef unsigned hwloc_memattr_id_t;

/** \brief Return the identifier of the memory attribute with the given name.
 */
HWLOC_DECLSPEC int
hwloc_memattr_get_by_name(hwloc_topology_t topology,
                          const char *name,
                          hwloc_memattr_id_t *id);


/** \brief Type of location. */
enum hwloc_location_type_e {
  /** \brief Location is given as a cpuset, in the location cpuset union field. \hideinitializer */
  HWLOC_LOCATION_TYPE_CPUSET = 1,
  /** \brief Location is given as an object, in the location object union field. \hideinitializer */
  HWLOC_LOCATION_TYPE_OBJECT = 0
};

```

这段代码定义了一个名为 `hwloc_location` 的结构体，用于表示在 HWLOC 库中要测量的属性的位置。

```cpp
/**
* Where to measure attributes from.
*/
struct hwloc_location {
 /**
  * Type of location.
  */
 enum hwloc_location_type_e type;
 
 /**
  * Actual location.
  */
 union hwloc_location_u {
   /**
    * Location as a cpuset, when the location type is
    * :HWLOC_LOCATION_TYPE_CPUSET.
    */
   hwloc_cpuset_t cpuset;
   
   /**
    * Location as an object, when the location type is
    * :HWLOC_LOCATION_TYPE_OBJECT.
    */
   hwloc_obj_t object;
 } location;
};
```

这个结构体有两个成员变量，一个是一个枚举类型 `hwloc_location_type_e`，表示位置的类型，另一个是一个联合类型 `hwloc_location_u`，表示位置的实际类型。

位置的实际类型可能包括以下几种：

* `HWLOC_LOCATION_TYPE_CPUSET`：作为 CPU 集的位置
* `HWLOC_LOCATION_TYPE_OBJECT`：作为对象的位置
* `HWLOC_LOCATION_TYPE_URL`：作为 URL 定位的位置（不常用）

对于每种位置类型，结构体中都有相应的成员函数来表示，例如 `hwloc_cpuset_t` 表示 `HWLOC_LOCATION_TYPE_CPUSET` 类型的位置，它可能包含多个 CPU 核心。


```cpp
/** \brief Where to measure attributes from. */
struct hwloc_location {
  /** \brief Type of location. */
  enum hwloc_location_type_e type;
  /** \brief Actual location. */
  union hwloc_location_u {
    /** \brief Location as a cpuset, when the location type is ::HWLOC_LOCATION_TYPE_CPUSET. */
    hwloc_cpuset_t cpuset;
    /** \brief Location as an object, when the location type is ::HWLOC_LOCATION_TYPE_OBJECT. */
    hwloc_obj_t object;
  } location;
};


/** \brief Flags for selecting target NUMA nodes. */
```

这段代码定义了一个枚举类型 hwloc_local_numanode_flag_e，它有三个成员变量，分别代表不同的 NUMA 节点本地性标志的值。

第一个成员变量 `HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY` 表示选择 NUMA 节点，其局部性大于给定的cpuset。例如，如果给定一个pu，那么它周围的节点都将被选择。

第二个成员变量 `HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY` 表示选择 NUMA 节点，其局部性小于给定的cpuset。例如，如果给定一个包含两个pu的package，那么它周围的节点都将被选择。

第三个成员变量 `HWLOC_LOCAL_NUMANODE_FLAG_ALL` 表示选择所有 NUMA 节点。如果给定了这个值，那么它将忽略 initiator 参数。

总的来说，这段代码定义了一个枚举类型，用于选择在给定情况下哪个 NUMA 节点应该被选择，然后从所有 NUMA 节点中选择所有符合特定条件的节点。


```cpp
enum hwloc_local_numanode_flag_e {
  /** \brief Select NUMA nodes whose locality is larger than the given cpuset.
   * For instance, if a single PU (or its cpuset) is given in \p initiator,
   * select all nodes close to the package that contains this PU.
   * \hideinitializer
   */
  HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY = (1UL<<0),

  /** \brief Select NUMA nodes whose locality is smaller than the given cpuset.
   * For instance, if a package (or its cpuset) is given in \p initiator,
   * also select nodes that are attached to only a half of that package.
   * \hideinitializer
   */
  HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY = (1UL<<1),

  /** \brief Select all NUMA nodes in the topology.
   * The initiator \p initiator is ignored.
   * \hideinitializer
   */
  HWLOC_LOCAL_NUMANODE_FLAG_ALL = (1UL<<2)
};

```

这段代码定义了一个名为`local_numa_nodes`的函数，它的功能是返回一个数组，该数组包含了根据输入的定位信息（`location`）选择出的局部节点。

默认情况下，该函数只选择定位在指定位置的节点。如果有其他 flag，则可以选择更多的节点。如果 `location` 被给出作为一个 explicit 对象，则使用该对象的 CPU set 来查找相应的节点。如果 `location` 不是给出一个对象，则使用其 CPU parent（即 I/O 对象所在的 CPU）来查找节点。

在输入参数中，`nr` 指向数组中可能存储的节点数量。在输出参数中，`nr` 被修改为存储的节点数量，或者在 `location` 有足够空间的情况下存储的节点数量。

需要注意的是，并非所有的 NUMA 节点都有内存属性值，因此在某些情况下，这些节点可能不会被报告为实际目标。另外，该函数中还包含一个限制，即在某些情况下，函数可能无法返回足够的节点，这种情况下 `nr` 将为负数。


```cpp
/** \brief Return an array of local NUMA nodes.
 *
 * By default only select the NUMA nodes whose locality is exactly
 * the given \p location. More nodes may be selected if additional flags
 * are given as a OR'ed set of ::hwloc_local_numanode_flag_e.
 *
 * If \p location is given as an explicit object, its CPU set is used
 * to find NUMA nodes with the corresponding locality.
 * If the object does not have a CPU set (e.g. I/O object), the CPU
 * parent (where the I/O object is attached) is used.
 *
 * On input, \p nr points to the number of nodes that may be stored
 * in the \p nodes array.
 * On output, \p nr will be changed to the number of stored nodes,
 * or the number of nodes that would have been stored if there were
 * enough room.
 *
 * \note Some of these NUMA nodes may not have any memory attribute
 * values and hence not be reported as actual targets in other functions.
 *
 * \note The number of NUMA nodes in the topology (obtained by
 * hwloc_bitmap_weight() on the root object nodeset) may be used
 * to allocate the \p nodes array.
 *
 * \note When an object CPU set is given as locality, for instance a Package,
 * and when flags contain both ::HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY
 * and ::HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY,
 * the returned array corresponds to the nodeset of that object.
 */
```

这段代码定义了一个名为 `hwloc_get_local_numanode_objs` 的函数，属于 `HWLOC_DECLSPEC` 类型。它的作用是获取指定拓扑结构中指定位置的 NUMA 节点的属性值。

具体来说，函数接受一个 `hwloc_topology_t` 的输入，一个指向 `hwloc_location` 类型的输出，一个表示节点数量和标志位的整数，以及一个指向节点对象的数组。函数返回一个 `hwloc_obj_t` 类型的属性值，这个属性值指向传递给它的 `location` 的 `node` 属性的值。

如果函数传来的输入参数 `topology` 和 `location` 不对应，或者 `nr` 参数没有被正确设置，函数的行为就不确定了，可能会导致程序崩溃或产生不可预测的结果。


```cpp
HWLOC_DECLSPEC int
hwloc_get_local_numanode_objs(hwloc_topology_t topology,
                              struct hwloc_location *location,
                              unsigned *nr,
                              hwloc_obj_t *nodes,
                              unsigned long flags);



/** \brief Return an attribute value for a specific target NUMA node.
 *
 * If the attribute does not relate to a specific initiator
 * (it does not have the flag ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR),
 * location \p initiator is ignored and may be \c NULL.
 *
 * \p flags must be \c 0 for now.
 *
 * \note The initiator \p initiator should be of type ::HWLOC_LOCATION_TYPE_CPUSET
 * when refering to accesses performed by CPU cores.
 * ::HWLOC_LOCATION_TYPE_OBJECT is currently unused internally by hwloc,
 * but users may for instance use it to provide custom information about
 * host memory accesses performed by GPUs.
 */
```

这段代码定义了一个名为 `hwloc_memattr_get_value` 的函数，它用于获取给定属性和启动器的目标节点中，与指定属性相关的 NUMA 节点的最优点。

具体来说，函数接受四个参数：

1. `topology`：要查询的 NUMA 拓扑结构。
2. `attribute`：要查询的内存属性。
3. `target_node`：目标节点。
4. `initiator`：指定启动器的 ID。
5. `flags`：用于指定与属性相关的标志，目前仅支持 `HWLOC_MEMATTR_FLAG_NEED_INITIATOR`。
6. `value`：返回的值。

函数首先定义了一个 `hwloc_memattr_id_t` 类型的整数变量 `attribute`，用于表示要查询的内存属性。接下来，定义了一个 `hwloc_location_t` 类型的整数变量 `initiator`，用于表示启动器。然后，定义了一个 `hwloc_uint64_t` 类型的变量 `flags`，用于指定与属性相关的标志。最后，定义了一个 `hwloc_uint64_t` 类型的变量 `value`，用于存储查询得到的值。

函数实现的核心部分是 `hwloc_memattr_get_value` 函数，它接收四个参数，首先通过 `topology` 获取拓扑结构，然后通过 `attribute` 获取要查询的内存属性，接着通过 `initiator` 获取启动器，最后查询指定属性对应的 NUMA 节点的最优点，并将结果存储到 `value` 变量中。

函数的实现没有显式地定义输出，因此它的作用不会在函数外部被调用。


```cpp
HWLOC_DECLSPEC int
hwloc_memattr_get_value(hwloc_topology_t topology,
                        hwloc_memattr_id_t attribute,
                        hwloc_obj_t target_node,
                        struct hwloc_location *initiator,
                        unsigned long flags,
                        hwloc_uint64_t *value);

/** \brief Return the best target NUMA node for the given attribute and initiator.
 *
 * If the attribute does not relate to a specific initiator
 * (it does not have the flag ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR),
 * location \p initiator is ignored and may be \c NULL.
 *
 * If \p value is non \c NULL, the corresponding value is returned there.
 *
 * If multiple targets have the same attribute values, only one is
 * returned (and there is no way to clarify how that one is chosen).
 * Applications that want to detect targets with identical/similar
 * values, or that want to look at values for multiple attributes,
 * should rather get all values using hwloc_memattr_get_value()
 * and manually select the target they consider the best.
 *
 * \p flags must be \c 0 for now.
 *
 * If there are no matching targets, \c -1 is returned with \p errno set to \c ENOENT;
 *
 * \note The initiator \p initiator should be of type ::HWLOC_LOCATION_TYPE_CPUSET
 * when refering to accesses performed by CPU cores.
 * ::HWLOC_LOCATION_TYPE_OBJECT is currently unused internally by hwloc,
 * but users may for instance use it to provide custom information about
 * host memory accesses performed by GPUs.
 */
```

这段代码定义了一个名为HWLOC_DECLSPEC的函数，它接受一个hwloc_topology_t类型的topology，一个hwloc_memattr_id_t类型的attribute，一个指向struct hwloc_location类型的initiator和一个unsigned long类型的flags，以及一个指向struct hwloc_obj_t类型的best_target和一个hwloc_uint64_t类型的*value。

函数的作用是返回给定属性和目标NUMA节点的最佳初始化器。如果没有与给定属性相关联的初始化器，函数将返回-1并设置errno为EINVAL。如果给定的value不是NULL，函数将返回相应的value。

如果存在多个具有相同属性的初始化器，函数将只返回其中一个，而且函数无法解释为什么只有一个初始化器被选择。对于那些希望检测具有相同或相似属性的初始化器，或者希望查看多个属性的值，应该使用hwloc_memattr_get_value()函数并手动选择被认为是最好的初始化器。

函数的返回值不应被修改或释放，因为它属于topology。


```cpp
HWLOC_DECLSPEC int
hwloc_memattr_get_best_target(hwloc_topology_t topology,
                              hwloc_memattr_id_t attribute,
                              struct hwloc_location *initiator,
                              unsigned long flags,
                              hwloc_obj_t *best_target, hwloc_uint64_t *value);

/** \brief Return the best initiator for the given attribute and target NUMA node.
 *
 * If the attribute does not relate to a specific initiator
 * (it does not have the flag ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR),
 * \c -1 is returned and \p errno is set to \c EINVAL.
 *
 * If \p value is non \c NULL, the corresponding value is returned there.
 *
 * If multiple initiators have the same attribute values, only one is
 * returned (and there is no way to clarify how that one is chosen).
 * Applications that want to detect initiators with identical/similar
 * values, or that want to look at values for multiple attributes,
 * should rather get all values using hwloc_memattr_get_value()
 * and manually select the initiator they consider the best.
 *
 * The returned initiator should not be modified or freed,
 * it belongs to the topology.
 *
 * \p flags must be \c 0 for now.
 *
 * If there are no matching initiators, \c -1 is returned with \p errno set to \c ENOENT;
 */
```

这段代码定义了一个名为HWLOC_DECLSPEC的函数，其作用是获取一个内存属性的最佳初始化器。函数的第一个参数是一个内存拓扑结构，第二个参数是一个内存属性ID，第三个参数是一个目标内存对象，第四个参数是一个标志，第五个参数是一个指向内存位置的指针，第六个参数是一个指向内存属性的输出参数。

具体来说，这段代码执行以下操作：首先，它通过调用hwloc_topology_t，设置传入的topology作为函数的一个输入参数。然后，它通过调用hwloc_memattr_id_t，将第二个输入参数attribute设置为一个内存属性ID。接着，它将第三个输入参数target设置为一个目标内存对象，第四个输入参数flags设置为一些内存属性标志，第五个输入参数best_initiator指向函数将返回的最佳初始化器的位置，最后一个输入参数value是一个输出参数，用于存储内存属性的值。

在函数实现中，它通过使用hwloc_location_t结构，将get_best_initiator函数的返回值存储在best_initiator指向的位置，然后将返回的值存储在value指向的位置。


```cpp
HWLOC_DECLSPEC int
hwloc_memattr_get_best_initiator(hwloc_topology_t topology,
                                 hwloc_memattr_id_t attribute,
                                 hwloc_obj_t target,
                                 unsigned long flags,
                                 struct hwloc_location *best_initiator, hwloc_uint64_t *value);

/** @} */


/** \defgroup hwlocality_memattrs_manage Managing memory attributes
 * @{
 */

/** \brief Return the name of a memory attribute.
 */
```

这两段代码定义了两个名为"hwloc_memattr_get_name"和"hwloc_memattr_get_flags"的函数，属于"hwloc_memattr_io_device"类别。

"hwloc_memattr_get_name"函数接受一个"hwloc_topology_t"类型的参数topology，一个"hwloc_memattr_id_t"类型的参数attribute，以及一个指向字符串的指针name。它返回给定的内存属性的名称。

"hwloc_memattr_get_flags"函数与"hwloc_memattr_get_name"类似，但返回的是给定属性的内存属性标志，即一个由多个标志位组成的OR，这些标志位由hwloc_memattr_register()函数设置。这些标志位保存在一个名为flags的整数中。

这两个函数一起组成了一个简单的内存属性获取函数，可以用于设置和获取内存属性，使程序更加灵活和可扩展。


```cpp
HWLOC_DECLSPEC int
hwloc_memattr_get_name(hwloc_topology_t topology,
                       hwloc_memattr_id_t attribute,
                       const char **name);

/** \brief Return the flags of the given attribute.
 *
 * Flags are a OR'ed set of ::hwloc_memattr_flag_e.
 */
HWLOC_DECLSPEC int
hwloc_memattr_get_flags(hwloc_topology_t topology,
                        hwloc_memattr_id_t attribute,
                        unsigned long *flags);

/** \brief Memory attribute flags.
 * Given to hwloc_memattr_register() and returned by hwloc_memattr_get_flags().
 */
```

这段代码定义了一个名为 hwloc_memattr_flag_e 的枚举类型，它定义了三个成员变量，分别表示对于给定内存属性的最佳节点，分别是有较高值还是较低值，以及需要给出 initiator 的值为多少。

具体来说，HWLOC_MEMATTR_FLAG_HIGHER_FIRST 表示对于带宽（Bandwidth）的内存属性，选择具有较高值的节点作为最佳节点；HWLOC_MEMATTR_FLAG_LOWER_FIRST 表示对于延迟（Latency）的内存属性，选择具有较低值的节点作为最佳节点；HWLOC_MEMATTR_FLAG_NEED_INITIATOR 表示对于所有内存属性，如果没有给出 initiator，则需要计算出最佳的节点。


```cpp
enum hwloc_memattr_flag_e {
  /** \brief The best nodes for this memory attribute are those with the higher values.
   * For instance Bandwidth.
   */
  HWLOC_MEMATTR_FLAG_HIGHER_FIRST = (1UL<<0),
  /** \brief The best nodes for this memory attribute are those with the lower values.
   * For instance Latency.
   */
  HWLOC_MEMATTR_FLAG_LOWER_FIRST = (1UL<<1),
  /** \brief The value returned for this memory attribute depends on the given initiator.
   * For instance Bandwidth and Latency, but not Capacity.
   */
  HWLOC_MEMATTR_FLAG_NEED_INITIATOR = (1UL<<2)
};

```

这段代码定义了一个名为 `hwloc_memattr_register` 的函数，用于注册一个新的内存属性。该函数的第一个参数是一个 `hwloc_topology_t` 类型的顶拓扑，第二个参数是一个字符指针，用于指定内存属性的名称，第三个参数是一个无符号整数，表示内存属性的标志，第四个参数是一个指向 `hwloc_memattr_id_t` 类型对象的指针，用于存储新注册的内存属性ID。

函数的作用是注册一个新的内存属性，如果该属性不定义在 `::hwloc_memattr_id_e` 中，则需要定义。该属性可以包含多个标志，如 `::HWLOC_MEMATTR_FLAG_HIGHER_FIRST` 或 `::HWLOC_MEMATTR_FLAG_LOWER_FIRST`。如果该属性是有效的，则返回新注册的内存属性ID。如果属性无效，则返回 `INVALID_MEMATTR_ID`。


```cpp
/** \brief Register a new memory attribute.
 *
 * Add a specific memory attribute that is not defined in ::hwloc_memattr_id_e.
 * Flags are a OR'ed set of ::hwloc_memattr_flag_e. It must contain at least
 * one of ::HWLOC_MEMATTR_FLAG_HIGHER_FIRST or ::HWLOC_MEMATTR_FLAG_LOWER_FIRST.
 */
HWLOC_DECLSPEC int
hwloc_memattr_register(hwloc_topology_t topology,
                       const char *name,
                       unsigned long flags,
                       hwloc_memattr_id_t *id);

/** \brief Set an attribute value for a specific target NUMA node.
 *
 * If the attribute does not relate to a specific initiator
 * (it does not have the flag ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR),
 * location \p initiator is ignored and may be \c NULL.
 *
 * The initiator will be copied into the topology,
 * the caller should free anything allocated to store the initiator,
 * for instance the cpuset.
 *
 * \p flags must be \c 0 for now.
 *
 * \note The initiator \p initiator should be of type ::HWLOC_LOCATION_TYPE_CPUSET
 * when referring to accesses performed by CPU cores.
 * ::HWLOC_LOCATION_TYPE_OBJECT is currently unused internally by hwloc,
 * but users may for instance use it to provide custom information about
 * host memory accesses performed by GPUs.
 */
```

hwloc_往反算命例函数，用于计算具有特定属性的目标 NUMA 节点。该函数的参数包括一个初始化器（可以是空）、一个表示特定属性的标志（必须是 0）、以及要计算的目标节点值。函数的实现主要针对用于工具和调试，而不是用于应用程序查询。

这个函数主要用于帮助开发者更好地理解硬件抽象层（HWLOC）中的本地内存访问。通过调用 `hwloc_get_local_numanode_objs()`，可以获取当前系统上所有具有特定属性的 NUMA 节点。然后，通过调用 `hwloc_normalize_node_value()` 来将这些节点的值转换为实际的目标节点。

这个函数的实现没有提供具体的 API 接口，但是它通过 `hwloc_normalize_node_value()` 函数将返回的节点值转换为实际的目标节点。这些目标节点将返回给调用者，以供他们进一步了解本地内存访问。


```cpp
HWLOC_DECLSPEC int
hwloc_memattr_set_value(hwloc_topology_t topology,
                        hwloc_memattr_id_t attribute,
                        hwloc_obj_t target_node,
                        struct hwloc_location *initiator,
                        unsigned long flags,
                        hwloc_uint64_t value);

/** \brief Return the target NUMA nodes that have some values for a given attribute.
 *
 * Return targets for the given attribute in the \p targets array
 * (for the given initiator if any).
 * If \p values is not \c NULL, the corresponding attribute values
 * are stored in the array it points to.
 *
 * On input, \p nr points to the number of targets that may be stored
 * in the array \p targets (and \p values).
 * On output, \p nr points to the number of targets (and values) that
 * were actually found, even if some of them couldn't be stored in the array.
 * Targets that couldn't be stored are ignored, but the function still
 * returns success (\c 0). The caller may find out by comparing the value pointed
 * by \p nr before and after the function call.
 *
 * The returned targets should not be modified or freed,
 * they belong to the topology.
 *
 * Argument \p initiator is ignored if the attribute does not relate to a specific
 * initiator (it does not have the flag ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR).
 * Otherwise \p initiator may be non \c NULL to report only targets
 * that have a value for that initiator.
 *
 * \p flags must be \c 0 for now.
 *
 * \note This function is meant for tools and debugging (listing internal information)
 * rather than for application queries. Applications should rather select useful
 * NUMA nodes with hwloc_get_local_numanode_objs() and then look at their attribute
 * values.
 *
 * \note The initiator \p initiator should be of type ::HWLOC_LOCATION_TYPE_CPUSET
 * when referring to accesses performed by CPU cores.
 * ::HWLOC_LOCATION_TYPE_OBJECT is currently unused internally by hwloc,
 * but users may for instance use it to provide custom information about
 * host memory accesses performed by GPUs.
 */
```

这段代码定义了一个名为HWLOC_DECLSPEC的函数，其作用是返回具有特定属性的目标节点，其中该属性被指定为给定的属性ID，并传递给函数的初始化器是一个指向目标节点的结构体，同时传递给函数的标志，如果有意义的话，也是一个指向目标节点的整数类型。函数将返回一个整数类型的指针，其中包含具有指定属性值的初始化器，或者如果传递给函数的标志为FLAG_NEED_INITIATOR，则返回一个包含所有具有该属性值的初始化器的指针。函数的实现对于现在来说是无效的，因为它只是定义了一个函数原型，并没有实现具体的逻辑。


```cpp
HWLOC_DECLSPEC int
hwloc_memattr_get_targets(hwloc_topology_t topology,
                          hwloc_memattr_id_t attribute,
                          struct hwloc_location *initiator,
                          unsigned long flags,
                          unsigned *nr, hwloc_obj_t *targets, hwloc_uint64_t *values);

/** \brief Return the initiators that have values for a given attribute for a specific target NUMA node.
 *
 * Return initiators for the given attribute and target node in the
 * \p initiators array.
 * If \p values is not \c NULL, the corresponding attribute values
 * are stored in the array it points to.
 *
 * On input, \p nr points to the number of initiators that may be stored
 * in the array \p initiators (and \p values).
 * On output, \p nr points to the number of initiators (and values) that
 * were actually found, even if some of them couldn't be stored in the array.
 * Initiators that couldn't be stored are ignored, but the function still
 * returns success (\c 0). The caller may find out by comparing the value pointed
 * by \p nr before and after the function call.
 *
 * The returned initiators should not be modified or freed,
 * they belong to the topology.
 *
 * \p flags must be \c 0 for now.
 *
 * If the attribute does not relate to a specific initiator
 * (it does not have the flag ::HWLOC_MEMATTR_FLAG_NEED_INITIATOR),
 * no initiator is returned.
 *
 * \note This function is meant for tools and debugging (listing internal information)
 * rather than for application queries. Applications should rather select useful
 * NUMA nodes with hwloc_get_local_numanode_objs() and then look at their attribute
 * values for some relevant initiators.
 */
```

这段代码定义了一个名为 `hwloc_memattr_get_initiators` 的函数，属于 `hwloc_memattr_api` 系列的函数。它的作用是获取指定 `hwloc_memattr_id_t` 属性的初始化器，并返回一个 `hwloc_location_t` 类型的数组，其中包含指定属性的初始化器的位置。

具体来说，函数接受四个参数：

1. `hwloc_topology_t` 类型的 `topology`：表示要查询的底层系统结构。
2. `hwloc_memattr_id_t` 类型的 `attribute`：要获取的属性ID。
3. `hwloc_obj_t` 类型的 `target_node`：目标节点，用于指定属性的位置。
4. `unsigned long flags`：用于设置或清除指定属性的标志，具体值见下表：

| 标志 | 说明 |
| --- | --- |
| L2MP_ACTIVITY | 是否激活L2内存映射活动。|
| L2MP_CACHE_ACTIVITY | 是否启用L2内存映射缓存。|
| L2MP_DISABLE_MEMORY_ mapping | 是否禁止映射内存。|
| L2MP_ENABLE_MEMORY_ mapping | 是否启用映射内存。|
| L2MP_NUM_NUMA_CHANNELS | 用于支持的NUMA通道数量。|

使用 `hwloc_topology_t` 可以帮助程序了解当前系统的拓扑结构，包括 CPU、GPU、LLC 等。`hwloc_memattr_id_t` 是一个用于标识特定属性的ID，可以在系统启动时配置，也可以在应用程序中指定。`hwloc_obj_t` 表示目标节点，可以是 `hwloc_device_t` 或 `hwloc_cpu_t` 等。

函数最后返回一个 `hwloc_location_t` 类型的数组，其中包含指定属性的初始化器的位置。这些位置用于在 `hwloc_memattr_t` 结构中查找与属性相关联的 `hwloc_location_t` 类型的对象，从而返回它们的指针。如果属性没有指定位置，函数将返回 `NULL`。


```cpp
HWLOC_DECLSPEC int
hwloc_memattr_get_initiators(hwloc_topology_t topology,
                             hwloc_memattr_id_t attribute,
                             hwloc_obj_t target_node,
                             unsigned long flags,
                             unsigned *nr, struct hwloc_location *initiators, hwloc_uint64_t *values);
/** @} */

#ifdef __cplusplus
} /* extern "C" */
#endif


#endif /* HWLOC_MEMATTR_H */

```