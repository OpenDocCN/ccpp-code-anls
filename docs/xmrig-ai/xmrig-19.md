# xmrig源码解析 19

# `src/3rdparty/hwloc/include/hwloc.h`

这段代码是一个头文件，它包含了一些关于Hwloc库的声明和声明。Hwloc是一个Python库，用于高性能计算。

首先，这个头文件包含了多个知识产权声明，其中包括：

CNRS：2009年CNRS的所有权利；
Inria：2009-2022年Inria的所有权利；
Université Bordeaux：2009-2012年巴黎巴约塞大学库依胡乱纪念馆的所有权；
Cisco Systems，Inc：2009-2020年思科系统所有；

然后，这个头文件定义了一个常量，`hwloc_version`，它告诉编译器它所支持的最新版本。

接下来，这个头文件包含了一些文档string，用于在编译时生成帮助文档。其中包括：

```cpp
/**
* hwloc_version: Hwloc version
* Description: Hwloc version number, built on Dec 15, 2021
* See also: https://www.open-mpi.org/projects/hwloc/doc/
*/
const hwloc_version = "2.06";
```

最后，这个头文件包含了一些函数声明，其中包括：

```cpp
/**
* __hwloc_minimal_required()
* @jdoc
* 返回值： 0，除非当前最小HWLOC版本小于所给版本，否则为正
* 说明：如果当前HWLOC版本小于所给版本，则函数返回0，否则返回正
* 参数：
* - self：当前HWLOC实例
* - required_version：所需的最低HWLOC版本
* 返回值：0，除非当前HWLOC版本小于所给版本，否则为正
*
* __hwloc_max_required()
* @jdoc
* 返回值： 1，除非当前HWLOC版本小于所给版本，否则为0
* 说明：如果当前HWLOC版本小于所给版本，则函数返回1，否则返回0
*
* __hwloc_is_compatible()
* @jdoc
* 返回值： 1，当hwloc版本兼容于当前HWLOC实例，否则为0
* 说明：检查HWLOC版本是否兼容于当前HWLOC实例
*
* __hwloc_new_unique_id()
* @jdoc
* 返回值： 0，除非当前HWLOC实例已有指定ID，否则为正
* 说明：生成一个新的HWLOC实例指定ID
*
* __hwloc_scan_dir()
* @jdoc
* 返回值： 0，除非当前HWLOC实例已包含扫描目录，否则为正
* 说明：扫描指定目录中的HWLOC实例
*
* __hwloc_uri_prefix()
* @jdoc
* 返回值： 0，除非当前HWLOC实例已包含URI前缀，否则为正
* 说明：获取指定URI前缀的HWLOC实例
*
* __hwloc_unified_info_version()
* @jdoc
* 返回值： 2，说明：Hwloc版本2.0，同时包含文档信息和编译器信息
*
* __hwloc_version_info()
* @jdoc
* 返回值： 2，说明：Hwloc版本2.0，同时包含文档信息和编译器信息
*
* __hwloc_no_validator()
* @jdoc
* 返回值： 0，除非当前HWLOC实例已包含有效的验证者，否则为正
* 说明：检查HWLOC实例是否包含有效的验证者
*
* __hwloc_register_validation_ wtriple()
* @jdoc
* 返回值： 0，除非当前HWLOC实例已包含有效的验证者，否则为正
* 说明：设置HWLOC实例的验证者，需要提供指导
*


```
/*
 * Copyright © 2009 CNRS
 * Copyright © 2009-2022 Inria.  All rights reserved.
 * Copyright © 2009-2012 Université Bordeaux
 * Copyright © 2009-2020 Cisco Systems, Inc.  All rights reserved.
 * See COPYING in top-level directory.
 */

/*=====================================================================
 *                 PLEASE GO READ THE DOCUMENTATION!
 *         ------------------------------------------------
 *               $tarball_directory/doc/doxygen-doc/
 *                                or
 *           https://www.open-mpi.org/projects/hwloc/doc/
 *=====================================================================
 *
 * FAIR WARNING: Do NOT expect to be able to figure out all the
 * subtleties of hwloc by simply reading function prototypes and
 * constant descrptions here in this file.
 *
 * Hwloc has wonderful documentation in both PDF and HTML formats for
 * your reading pleasure.  The formal documentation explains a LOT of
 * hwloc-specific concepts, provides definitions, and discusses the
 * "big picture" for many of the things that you'll find here in this
 * header file.
 *
 * The PDF/HTML documentation was generated via Doxygen; much of what
 * you'll see in there is also here in this file.  BUT THERE IS A LOT
 * THAT IS IN THE PDF/HTML THAT IS ***NOT*** IN hwloc.h!
 *
 * There are entire paragraph-length descriptions, discussions, and
 * pretty pictures to explain subtle corner cases, provide concrete
 * examples, etc.
 *
 * Please, go read the documentation.  :-)
 *
 * Moreover there are several examples of hwloc use under doc/examples
 * in the source tree.
 *
 *=====================================================================*/

```cpp

这段代码定义了一个名为"hwloc"的API，用于访问硬件位置信息。它包含了一些定义和导出头文件，其中包含了一些与位图相关的定义和一些通用的拓扑遍历工具。

具体来说，这个API允许在应用程序中使用"hwloc"头文件中的函数和数据结构。它还定义了一些与位图相关的函数，例如：

- "get_device_tree"函数用于获取系统设备树中某个设备的节点。
- "get_device_topology"函数用于获取系统设备拓扑结构中某个节点的位置。
- "get_device_acceleration"函数用于获取设备加速度信息。
- "set_device_topology"函数用于设置系统设备拓扑结构。

此外，这个API还定义了一些通用的拓扑遍历工具，例如：

- "get_host_subnet"函数用于获取主机和子网的映射。
- "get_device_interconnect"函数用于获取设备互连信息。
- "get_device_parent"函数用于获取设备父节点。

由于该API是用于在hwloc库中使用，因此它的具体实现可能因具体的硬件平台而异。


```
/** \file
 * \brief The hwloc API.
 *
 * See hwloc/bitmap.h for bitmap specific macros.
 * See hwloc/helper.h for high-level topology traversal helpers.
 * See hwloc/inlines.h for the actual inline code of some functions below.
 * See hwloc/export.h for exporting topologies to XML or to synthetic descriptions.
 * See hwloc/distances.h for querying and modifying distances between objects.
 * See hwloc/diff.h for manipulating differences between similar topologies.
 */

#ifndef HWLOC_H
#define HWLOC_H

#include "hwloc/autogen/config.h"

```cpp

这段代码包括以下几个部分：

1. `#include <sys/types.h>`：引入了 `sys/types.h` 头文件，主要包含 `sizeof`、`min` 和 `max` 函数，用于获取和设置数据类型变量的大小。

2. `#include <stdio.h>`：引入了 `stdio.h` 头文件，用于在程序中输出信息。

3. `#include <string.h>`：引入了 `string.h` 头文件，用于处理字符串操作，包括 `strlen`、`strcpy`、`strcat` 等函数。

4. `#include <limits.h>`：引入了 `limits.h` 头文件，包含了许多常见的数学和统计概念，如 `INFINITY`、`NAN` 等。

5. `#include "hwloc/rename.h"`：引入了 `hwloc/rename.h` 头文件，可能是用于在 `hwloc` 库中进行文件操作的库。

6. `#include "hwloc/bitmap.h"`：引入了 `hwloc/bitmap.h` 头文件，可能是用于在 `hwloc` 库中创建和操作位图的库。

由于 `hwloc` 库在 `立方程序`（CUDA）中使用得比较多，而且 `hwloc/rename.h` 和 `hwloc/bitmap.h` 都和文件和位图操作有关，因此可以推测这段代码的作用可能是：为了解决在 `立方程序` 中需要使用 `hwloc` 库，但 `hwloc` 库中的一些头文件和函数需要从立方程序外的来源进行引入的问题。


```
#include <sys/types.h>
#include <stdio.h>
#include <string.h>
#include <limits.h>

/*
 * Symbol transforms
 */
#include "hwloc/rename.h"

/*
 * Bitmap definitions
 */

#include "hwloc/bitmap.h"


```cpp

这段代码定义了一个预处理指令#ifdef(__cplusplus)，如果该指令在任何地方被定义，则预处理程序会将以下代码块中的内容编译一次。这个预处理指令会在编译时将#include指示的源文件编译一次，因此这个代码块中的内容是在编译时被引入的。

如果这个预处理指令在任何地方没有被定义，则该代码块中的内容将不会被编译。

该代码中包含了一个#define指令，用于定义一个名为"hwlocality_api_version"的常量。该常量的值在每次新 release（以"X.Y.Z"格式表示）时会被更新为(X<<16)+(Y<<8)+Z。更新时，该代码块中的内容会被插入到该常量的值中。

该代码还包含了一个#ifdef指令，用于检查当前编译是否使用了当前支持的最大API版本。如果是，则该代码块中的内容会被编译一次。否则，该代码块中的内容不会被编译。

该代码中包含的代码块是一个用于在编译时检查当前API版本的说明文件。它告诉编译器当前代码库的版本，以及建议在哪些硬件上使用该代码库的API版本。该代码块中的内容只会在当前API版本被修改时才会被编译，因此使用该代码库的开发者可以在编译时查看哪些API是当前正在使用的。


```
#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_api_version API version
 * @{
 */

/** \brief Indicate at build time which hwloc API version is being used.
 *
 * This number is updated to (X<<16)+(Y<<8)+Z when a new release X.Y.Z
 * actually modifies the API.
 *
 * Users may check for available features at build time using this number
 * (see \ref faq_version_api).
 *
 * \note This should not be confused with HWLOC_VERSION, the library version.
 * Two stable releases of the same series usually have the same ::HWLOC_API_VERSION
 * even if their HWLOC_VERSION are different.
 */
```cpp

这段代码定义了一个名为 `HWLOC_API_VERSION` 的宏，其值为 0x00020800。

该宏表示在运行时会告诉开发人员所用的 hwloc API 版本，其值应该是 `HWLOC_API_VERSION`。

该代码还定义了一个名为 `HWLOC_COMPONENT_ABI` 的宏，其值为 7。

该宏表示当前组件和插件的 ABI(应用程序接口)版本，其值应该是 `HWLOC_COMPONENT_ABI`。


```
#define HWLOC_API_VERSION 0x00020800

/** \brief Indicate at runtime which hwloc API version was used at build time.
 *
 * Should be ::HWLOC_API_VERSION if running on the same version.
 */
HWLOC_DECLSPEC unsigned hwloc_get_api_version(void);

/** \brief Current component and plugin ABI version (see hwloc/plugins.h) */
#define HWLOC_COMPONENT_ABI 7

/** @} */



```cpp

这段代码定义了一个名为`hwlocality_object_sets`的组，其中包括两个成员变量：`hwloc_cpuset_t`和`hwloc_nodeset_t`。这两个成员变量都是`::hwloc_bitmap_t`的别名，代表同一个后端类型。

这个代码段解释了`hwlocality_bitmap_t`的别名，列出了使用该别名时所有可用的函数，并指出了为什么会有这两个不同的成员变量。

作者强调了这两个成员变量尽管作用相同（例如，在设置或检索对象集合时，都涉及激活或抑制单个元素），但它们在实际应用中的上下文非常不同。因此，这些名称的主要区别只是反映了它们的应用意图。


```
/** \defgroup hwlocality_object_sets Object Sets (hwloc_cpuset_t and hwloc_nodeset_t)
 *
 * Hwloc uses bitmaps to represent two distinct kinds of object sets:
 * CPU sets (::hwloc_cpuset_t) and NUMA node sets (::hwloc_nodeset_t).
 * These types are both typedefs to a common back end type
 * (::hwloc_bitmap_t), and therefore all the hwloc bitmap functions
 * are applicable to both ::hwloc_cpuset_t and ::hwloc_nodeset_t (see
 * \ref hwlocality_bitmap).
 *
 * The rationale for having two different types is that even though
 * the actions one wants to perform on these types are the same (e.g.,
 * enable and disable individual items in the set/mask), they're used
 * in very different contexts: one for specifying which processors to
 * use and one for specifying which NUMA nodes to use.  Hence, the
 * name difference is really just to reflect the intent of where the
 * type is used.
 *
 * @{
 */

```cpp

这段代码定义了两个名为`hwloc_cpuset_t`和`hwloc_const_cpuset_t`的枚举类型，它们都继承自`hwloc_bitmap_t`类型。

`hwloc_cpuset_t`是一个非可变的`hwloc_bitmap_t`，用于表示CPU的物理组，它的每个成员节点都对应一个具体的CPU物理组。它可以被任何`::hwloc_bitmap_t`(参考`hwloc/bitmap.h`)的方式咨询和修改。

`hwloc_const_cpuset_t`是一个可变的`hwloc_const_bitmap_t`，用于表示没有任何节点的CPU的物理组。它也可以被任何`::hwloc_bitmap_t`(参考`hwloc/bitmap.h`)的方式咨询和修改。

两者的成员函数都使用了`hwloc_get_pu_obj_by_os_index()`和`hwloc_get_numanode_obj_by_os_index()`函数。这些函数用于将每个成员节点转换为相应的`::hwloc_cpu_device_t`或`::hwloc_numanode_t`对象。

该代码可能有助于在不需要显式创建节点的情况下，使用操作系统提供的内存组进行绑定。


```
/** \brief A CPU set is a bitmap whose bits are set according to CPU
 * physical OS indexes.
 *
 * It may be consulted and modified with the bitmap API as any
 * ::hwloc_bitmap_t (see hwloc/bitmap.h).
 *
 * Each bit may be converted into a PU object using
 * hwloc_get_pu_obj_by_os_index().
 */
typedef hwloc_bitmap_t hwloc_cpuset_t;
/** \brief A non-modifiable ::hwloc_cpuset_t. */
typedef hwloc_const_bitmap_t hwloc_const_cpuset_t;

/** \brief A node set is a bitmap whose bits are set according to NUMA
 * memory node physical OS indexes.
 *
 * It may be consulted and modified with the bitmap API as any
 * ::hwloc_bitmap_t (see hwloc/bitmap.h).
 * Each bit may be converted into a NUMA node object using
 * hwloc_get_numanode_obj_by_os_index().
 *
 * When binding memory on a system without any NUMA node,
 * the single main memory bank is considered as NUMA node #0.
 *
 * See also \ref hwlocality_helper_nodeset_convert.
 */
```cpp

这段代码定义了两种指向不同类型节点集的类型：`hwloc_nodeset_t` 和 `hwloc_const_nodeset_t`。这些类型都是非可变的，即不能修改它们的值。

然后，没有进一步的说明或定义了什么函数或变量。因此，此代码可能在某些上下文中被用来定义或作为代码片段使用，但具体的作用和使用方法取决于具体的上下文。


```
typedef hwloc_bitmap_t hwloc_nodeset_t;
/** \brief A non-modifiable ::hwloc_nodeset_t.
 */
typedef hwloc_const_bitmap_t hwloc_const_nodeset_t;

/** @} */



/** \defgroup hwlocality_object_types Object Types
 * @{
 */

/** \brief Type of topology object.
 *
 * \note Do not rely on the ordering or completeness of the values as new ones
 * may be defined in the future!  If you need to compare types, use
 * hwloc_compare_types() instead.
 */
```cpp



This is a list of HWLOC object classes that have been defined in the `hwloc_obj_type_h.h` header file. 

The HWLOC objects are divided into several categories, including misc objects, memory-side cache objects, and die objects. 

Misc objects are objects without a specific meaning, that can be added by the application for its own use or by hwloc for miscellaneous objects such as MemoryModule (DIMMs). They have NULL CPU and node sets and may have Misc objects as children.

Memory-side cache objects are objects that exist in front of a specific NUMA node, with a special depth of `HWLOC_TYPE_DEPTH_MEMCACHE` instead of a normal depth. They have at least one NUMA node as a memory child and may have Misc objects as children.

Die objects are objects that contain multiple cores within a physical package.

The `hwloc_obj_type_t` enum defines the structure of an HWLOC object, with a指针 to the object's data and a return type of `int` that specifies the object's type.


```
typedef enum {

/** \cond */
#define HWLOC_OBJ_TYPE_MIN HWLOC_OBJ_MACHINE /* Sentinel value */
/** \endcond */

  HWLOC_OBJ_MACHINE,	/**< \brief Machine.
			  * A set of processors and memory with cache
			  * coherency.
			  *
			  * This type is always used for the root object of a topology,
			  * and never used anywhere else.
			  * Hence its parent is always \c NULL.
			  */

  HWLOC_OBJ_PACKAGE,	/**< \brief Physical package.
			  * The physical package that usually gets inserted
			  * into a socket on the motherboard.
			  * A processor package usually contains multiple cores,
			  * and possibly some dies.
			  */
  HWLOC_OBJ_CORE,	/**< \brief Core.
			  * A computation unit (may be shared by several
			  * PUs, aka logical processors).
			  */
  HWLOC_OBJ_PU,		/**< \brief Processing Unit, or (Logical) Processor.
			  * An execution unit (may share a core with some
			  * other logical processors, e.g. in the case of
			  * an SMT core).
			  *
			  * This is the smallest object representing CPU resources,
			  * it cannot have any child except Misc objects.
			  *
			  * Objects of this kind are always reported and can
			  * thus be used as fallback when others are not.
			  */

  HWLOC_OBJ_L1CACHE,	/**< \brief Level 1 Data (or Unified) Cache. */
  HWLOC_OBJ_L2CACHE,	/**< \brief Level 2 Data (or Unified) Cache. */
  HWLOC_OBJ_L3CACHE,	/**< \brief Level 3 Data (or Unified) Cache. */
  HWLOC_OBJ_L4CACHE,	/**< \brief Level 4 Data (or Unified) Cache. */
  HWLOC_OBJ_L5CACHE,	/**< \brief Level 5 Data (or Unified) Cache. */

  HWLOC_OBJ_L1ICACHE,	/**< \brief Level 1 instruction Cache (filtered out by default). */
  HWLOC_OBJ_L2ICACHE,	/**< \brief Level 2 instruction Cache (filtered out by default). */
  HWLOC_OBJ_L3ICACHE,	/**< \brief Level 3 instruction Cache (filtered out by default). */

  HWLOC_OBJ_GROUP,	/**< \brief Group objects.
			  * Objects which do not fit in the above but are
			  * detected by hwloc and are useful to take into
			  * account for affinity. For instance, some operating systems
			  * expose their arbitrary processors aggregation this
			  * way.  And hwloc may insert such objects to group
			  * NUMA nodes according to their distances.
			  * See also \ref faq_groups.
			  *
			  * These objects are removed when they do not bring
			  * any structure (see ::HWLOC_TYPE_FILTER_KEEP_STRUCTURE).
			  */

  HWLOC_OBJ_NUMANODE,	/**< \brief NUMA node.
			  * An object that contains memory that is directly
			  * and byte-accessible to the host processors.
			  * It is usually close to some cores (the corresponding objects
			  * are descendants of the NUMA node object in the hwloc tree).
			  *
			  * This is the smallest object representing Memory resources,
			  * it cannot have any child except Misc objects.
			  * However it may have Memory-side cache parents.
			  *
			  * There is always at least one such object in the topology
			  * even if the machine is not NUMA.
			  *
			  * Memory objects are not listed in the main children list,
			  * but rather in the dedicated Memory children list.
			  *
			  * NUMA nodes have a special depth ::HWLOC_TYPE_DEPTH_NUMANODE
			  * instead of a normal depth just like other objects in the
			  * main tree.
			  */

  HWLOC_OBJ_BRIDGE,	/**< \brief Bridge (filtered out by default).
			  * Any bridge (or PCI switch) that connects the host or an I/O bus,
			  * to another I/O bus.
			  *
			  * Bridges are not added to the topology unless their
			  * filtering is changed (see hwloc_topology_set_type_filter()
			  * and hwloc_topology_set_io_types_filter()).
			  *
			  * I/O objects are not listed in the main children list,
			  * but rather in the dedicated io children list.
			  * I/O objects have NULL CPU and node sets.
			  */
  HWLOC_OBJ_PCI_DEVICE,	/**< \brief PCI device (filtered out by default).
			  *
			  * PCI devices are not added to the topology unless their
			  * filtering is changed (see hwloc_topology_set_type_filter()
			  * and hwloc_topology_set_io_types_filter()).
			  *
			  * I/O objects are not listed in the main children list,
			  * but rather in the dedicated io children list.
			  * I/O objects have NULL CPU and node sets.
			  */
  HWLOC_OBJ_OS_DEVICE,	/**< \brief Operating system device (filtered out by default).
			  *
			  * OS devices are not added to the topology unless their
			  * filtering is changed (see hwloc_topology_set_type_filter()
			  * and hwloc_topology_set_io_types_filter()).
			  *
			  * I/O objects are not listed in the main children list,
			  * but rather in the dedicated io children list.
			  * I/O objects have NULL CPU and node sets.
			  */

  HWLOC_OBJ_MISC,	/**< \brief Miscellaneous objects (filtered out by default).
			  * Objects without particular meaning, that can e.g. be
			  * added by the application for its own use, or by hwloc
			  * for miscellaneous objects such as MemoryModule (DIMMs).
			  *
			  * They are not added to the topology unless their filtering
			  * is changed (see hwloc_topology_set_type_filter()).
			  *
			  * These objects are not listed in the main children list,
			  * but rather in the dedicated misc children list.
			  * Misc objects may only have Misc objects as children,
			  * and those are in the dedicated misc children list as well.
			  * Misc objects have NULL CPU and node sets.
			  */

  HWLOC_OBJ_MEMCACHE,	/**< \brief Memory-side cache (filtered out by default).
			  * A cache in front of a specific NUMA node.
			  *
			  * This object always has at least one NUMA node as a memory child.
			  *
			  * Memory objects are not listed in the main children list,
			  * but rather in the dedicated Memory children list.
			  *
			  * Memory-side cache have a special depth ::HWLOC_TYPE_DEPTH_MEMCACHE
			  * instead of a normal depth just like other objects in the
			  * main tree.
			  */

  HWLOC_OBJ_DIE,	/**< \brief Die within a physical package.
			 * A subpart of the physical package, that contains multiple cores.
			 */

  HWLOC_OBJ_TYPE_MAX    /**< \private Sentinel value */
} hwloc_obj_type_t;

```cpp

hwloc_obj_bridge_type_t - bridge type constant for hwloc_obj_bridge_register()

This type constant is used to differentiate between different types of a bridge, and is used to set the bridge to an upstream device.

HWLOC_OBJ_BRIDGE_PCI - PCI-side of a bridge

HWLOC_OBJ_BRIDGE_PARTITION_PCI - Partition of PCI-side of a bridge

HWLOC_OBJ_BRIDGE_GPU - GPU-side of a bridge

HWLOC_OBJ_BRIDGE_NET - Network-side of a bridge

HWLOC_OBJ_BRIDGE_OPENFABRICS - OpenFabrics-side of a bridge

HWLOC_OBJ_BRIDGE_DMA - DMA-side of a bridge

HWLOC_OBJ_BRIDGE_COPROC - Coprocessor-side of a bridge


```
/** \brief Cache type. */
typedef enum hwloc_obj_cache_type_e {
  HWLOC_OBJ_CACHE_UNIFIED,      /**< \brief Unified cache. */
  HWLOC_OBJ_CACHE_DATA,         /**< \brief Data cache. */
  HWLOC_OBJ_CACHE_INSTRUCTION   /**< \brief Instruction cache (filtered out by default). */
} hwloc_obj_cache_type_t;

/** \brief Type of one side (upstream or downstream) of an I/O bridge. */
typedef enum hwloc_obj_bridge_type_e {
  HWLOC_OBJ_BRIDGE_HOST,	/**< \brief Host-side of a bridge, only possible upstream. */
  HWLOC_OBJ_BRIDGE_PCI		/**< \brief PCI-side of a bridge. */
} hwloc_obj_bridge_type_t;

/** \brief Type of a OS device. */
typedef enum hwloc_obj_osdev_type_e {
  HWLOC_OBJ_OSDEV_BLOCK,	/**< \brief Operating system block device, or non-volatile memory device.
				  * For instance "sda" or "dax2.0" on Linux. */
  HWLOC_OBJ_OSDEV_GPU,		/**< \brief Operating system GPU device.
				  * For instance ":0.0" for a GL display,
				  * "card0" for a Linux DRM device. */
  HWLOC_OBJ_OSDEV_NETWORK,	/**< \brief Operating system network device.
				  * For instance the "eth0" interface on Linux. */
  HWLOC_OBJ_OSDEV_OPENFABRICS,	/**< \brief Operating system openfabrics device.
				  * For instance the "mlx4_0" InfiniBand HCA,
				  * "hfi1_0" Omni-Path interface,
				  * or "bxi0" Atos/Bull BXI HCA on Linux. */
  HWLOC_OBJ_OSDEV_DMA,		/**< \brief Operating system dma engine device.
				  * For instance the "dma0chan0" DMA channel on Linux. */
  HWLOC_OBJ_OSDEV_COPROC	/**< \brief Operating system co-processor device.
				  * For instance "opencl0d0" for a OpenCL device,
				  * "cuda0" for a CUDA device. */
} hwloc_obj_osdev_type_t;

```cpp

这段代码定义了一个名为 `compare_type` 的函数，用于比较两个对象类型 depth的大小关系。函数的参数为两个指向相同类型对象的指针 `type1` 和 `type2`。函数返回值分别表示 `type1` 对象包含 `type2` 对象、两个对象相同或者 `type1` 对象包含 `type2` 对象 时的情况。如果两个对象类型无法比较(因为它们通常不会在对方出现)，则返回 `::HWLOC_TYPE_UNORDERED`。

该函数主要用于在需要比较两个对象类型的大小时提供一种比较方法。由于对象类型通常不会包含其他类型的对象，因此该函数可以提供比按照通常机器、系统、组件、包、类等的深度更精确的比较结果。但是，这种比较方法仅供参考，不能保证实际使用中应用程序的正确性。


```
/** \brief Compare the depth of two object types
 *
 * Types shouldn't be compared as they are, since newer ones may be added in
 * the future.  This function returns less than, equal to, or greater than zero
 * respectively if \p type1 objects usually include \p type2 objects, are the
 * same as \p type2 objects, or are included in \p type2 objects. If the types
 * can not be compared (because neither is usually contained in the other),
 * ::HWLOC_TYPE_UNORDERED is returned.  Object types containing CPUs can always
 * be compared (usually, a system contains machines which contain nodes which
 * contain packages which contain caches, which contain cores, which contain
 * processors).
 *
 * \note ::HWLOC_OBJ_PU will always be the deepest,
 * while ::HWLOC_OBJ_MACHINE is always the highest.
 *
 * \note This does not mean that the actual topology will respect that order:
 * e.g. as of today cores may also contain caches, and packages may also contain
 * nodes. This is thus just to be seen as a fallback comparison method.
 */
```cpp

这段代码定义了一个名为 `hwloc_compare_types` 的函数，其参数为两种 `hwloc_obj_type_t` 类型的整数类型。这个函数返回一个整数，用于指示当这两种类型无法比较时，应返回的值。

函数定义了一个名为 `HWLOC_DECLSPEC` 的声明，说明这个函数是一个 `hwloc_attribute_const` 类型的函数，这意味着这个函数的返回值必须被声明为常量，而且这个常量的值必须是一个整数。

接下来定义了一个名为 `HWLOC_TYPE_UNORDERED` 的定义，表示当 `hwloc_compare_types` 函数返回时，如果两种类型无法比较，那么返回的值将是 `INT_MAX`。

最后定义了一个名为 `hwloc_obj_attr_u` 的 union 类型，用于存储 `hwloc_obj_type_t` 类型的属性的元数据。


```
HWLOC_DECLSPEC int hwloc_compare_types (hwloc_obj_type_t type1, hwloc_obj_type_t type2) __hwloc_attribute_const;

/** \brief Value returned by hwloc_compare_types() when types can not be compared. \hideinitializer */
#define HWLOC_TYPE_UNORDERED INT_MAX

/** @} */



/** \defgroup hwlocality_objects Object Structure and Attributes
 * @{
 */

union hwloc_obj_attr_u;

```cpp

This is a struct definition for an HWLOC node object that has a topology-aware information. It contains a pointer to an info array of a specific type (stringified), a pointer to the Info object, and a count of the number of unique NUMA nodes that contribute to this HWLOC node.

The topology information may not be always accurate, and there may be NUMA nodes contributing to this object, even if their topology information is unknown or incomplete.

The HWLOC node object is often used to represent virtualized hardware resources, and it allows for the management of these resources using a HWLOC layer.


```
/** \brief Structure of a topology object
 *
 * Applications must not modify any field except \p hwloc_obj.userdata.
 */
struct hwloc_obj {
  /* physical information */
  hwloc_obj_type_t type;		/**< \brief Type of object */
  char *subtype;			/**< \brief Subtype string to better describe the type field. */

  unsigned os_index;			/**< \brief OS-provided physical index number.
					 * It is not guaranteed unique across the entire machine,
					 * except for PUs and NUMA nodes.
					 * Set to HWLOC_UNKNOWN_INDEX if unknown or irrelevant for this object.
					 */
#define HWLOC_UNKNOWN_INDEX (unsigned)-1

  char *name;				/**< \brief Object-specific name if any.
					 * Mostly used for identifying OS devices and Misc objects where
					 * a name string is more useful than numerical indexes.
					 */

  hwloc_uint64_t total_memory; /**< \brief Total memory (in bytes) in NUMA nodes below this object. */

  union hwloc_obj_attr_u *attr;		/**< \brief Object type-specific Attributes,
					 * may be \c NULL if no attribute value was found */

  /* global position */
  int depth;				/**< \brief Vertical index in the hierarchy.
					 *
					 * For normal objects, this is the depth of the horizontal level
					 * that contains this object and its cousins of the same type.
					 * If the topology is symmetric, this is equal to the parent depth
					 * plus one, and also equal to the number of parent/child links
					 * from the root object to here.
					 *
					 * For special objects (NUMA nodes, I/O and Misc) that are not
					 * in the main tree, this is a special negative value that
					 * corresponds to their dedicated level,
					 * see hwloc_get_type_depth() and ::hwloc_get_type_depth_e.
					 * Those special values can be passed to hwloc functions such
					 * hwloc_get_nbobjs_by_depth() as usual.
					 */
  unsigned logical_index;		/**< \brief Horizontal index in the whole list of similar objects,
					 * hence guaranteed unique across the entire machine.
					 * Could be a "cousin_rank" since it's the rank within the "cousin" list below
					 * Note that this index may change when restricting the topology
					 * or when inserting a group.
					 */

  /* cousins are all objects of the same type (and depth) across the entire topology */
  struct hwloc_obj *next_cousin;	/**< \brief Next object of same type and depth */
  struct hwloc_obj *prev_cousin;	/**< \brief Previous object of same type and depth */

  /* children of the same parent are siblings, even if they may have different type and depth */
  struct hwloc_obj *parent;		/**< \brief Parent, \c NULL if root (Machine object) */
  unsigned sibling_rank;		/**< \brief Index in parent's \c children[] array. Or the index in parent's Memory, I/O or Misc children list. */
  struct hwloc_obj *next_sibling;	/**< \brief Next object below the same parent (inside the same list of children). */
  struct hwloc_obj *prev_sibling;	/**< \brief Previous object below the same parent (inside the same list of children). */
  /** @name List and array of normal children below this object (except Memory, I/O and Misc children). */
  /**@{*/
  unsigned arity;			/**< \brief Number of normal children.
					 * Memory, Misc and I/O children are not listed here
					 * but rather in their dedicated children list.
					 */
  struct hwloc_obj **children;		/**< \brief Normal children, \c children[0 .. arity -1] */
  struct hwloc_obj *first_child;	/**< \brief First normal child */
  struct hwloc_obj *last_child;		/**< \brief Last normal child */
  /**@}*/

  int symmetric_subtree;		/**< \brief Set if the subtree of normal objects below this object is symmetric,
					  * which means all normal children and their children have identical subtrees.
					  *
					  * Memory, I/O and Misc children are ignored.
					  *
					  * If set in the topology root object, lstopo may export the topology
					  * as a synthetic string.
					  */

  /** @name List of Memory children below this object. */
  /**@{*/
  unsigned memory_arity;		/**< \brief Number of Memory children.
					 * These children are listed in \p memory_first_child.
					 */
  struct hwloc_obj *memory_first_child;	/**< \brief First Memory child.
					 * NUMA nodes and Memory-side caches are listed here
					 * (\p memory_arity and \p memory_first_child)
					 * instead of in the normal children list.
					 * See also hwloc_obj_type_is_memory().
					 *
					 * A memory hierarchy starts from a normal CPU-side object
					 * (e.g. Package) and ends with NUMA nodes as leaves.
					 * There might exist some memory-side caches between them
					 * in the middle of the memory subtree.
					 */
  /**@}*/

  /** @name List of I/O children below this object. */
  /**@{*/
  unsigned io_arity;			/**< \brief Number of I/O children.
					 * These children are listed in \p io_first_child.
					 */
  struct hwloc_obj *io_first_child;	/**< \brief First I/O child.
					 * Bridges, PCI and OS devices are listed here (\p io_arity and \p io_first_child)
					 * instead of in the normal children list.
					 * See also hwloc_obj_type_is_io().
					 */
  /**@}*/

  /** @name List of Misc children below this object. */
  /**@{*/
  unsigned misc_arity;			/**< \brief Number of Misc children.
					 * These children are listed in \p misc_first_child.
					 */
  struct hwloc_obj *misc_first_child;	/**< \brief First Misc child.
					 * Misc objects are listed here (\p misc_arity and \p misc_first_child)
					 * instead of in the normal children list.
					 */
  /**@}*/

  /* cpusets and nodesets */
  hwloc_cpuset_t cpuset;		/**< \brief CPUs covered by this object
                                          *
                                          * This is the set of CPUs for which there are PU objects in the topology
                                          * under this object, i.e. which are known to be physically contained in this
                                          * object and known how (the children path between this object and the PU
                                          * objects).
                                          *
                                          * If the ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED configuration flag is set,
                                          * some of these CPUs may be online but not allowed for binding,
                                          * see hwloc_topology_get_allowed_cpuset().
                                          *
					  * \note All objects have non-NULL CPU and node sets except Misc and I/O objects.
					  *
                                          * \note Its value must not be changed, hwloc_bitmap_dup() must be used instead.
                                          */
  hwloc_cpuset_t complete_cpuset;       /**< \brief The complete CPU set of processors of this object,
                                          *
                                          * This may include not only the same as the cpuset field, but also some CPUs for
                                          * which topology information is unknown or incomplete, some offlines CPUs, and
                                          * the CPUs that are ignored when the ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED flag
                                          * is not set.
                                          * Thus no corresponding PU object may be found in the topology, because the
                                          * precise position is undefined. It is however known that it would be somewhere
                                          * under this object.
                                          *
                                          * \note Its value must not be changed, hwloc_bitmap_dup() must be used instead.
                                          */

  hwloc_nodeset_t nodeset;              /**< \brief NUMA nodes covered by this object or containing this object
                                          *
                                          * This is the set of NUMA nodes for which there are NUMA node objects in the
                                          * topology under or above this object, i.e. which are known to be physically
                                          * contained in this object or containing it and known how (the children path
                                          * between this object and the NUMA node objects).
                                          *
                                          * In the end, these nodes are those that are close to the current object.
                                          * Function hwloc_get_local_numanode_objs() may be used to list those NUMA
                                          * nodes more precisely.
                                          *
                                          * If the ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED configuration flag is set,
                                          * some of these nodes may be online but not allowed for allocation,
                                          * see hwloc_topology_get_allowed_nodeset().
                                          *
                                          * If there are no NUMA nodes in the machine, all the memory is close to this
                                          * object, so only the first bit may be set in \p nodeset.
                                          *
					  * \note All objects have non-NULL CPU and node sets except Misc and I/O objects.
					  *
                                          * \note Its value must not be changed, hwloc_bitmap_dup() must be used instead.
                                          */
  hwloc_nodeset_t complete_nodeset;     /**< \brief The complete NUMA node set of this object,
                                          *
                                          * This may include not only the same as the nodeset field, but also some NUMA
                                          * nodes for which topology information is unknown or incomplete, some offlines
                                          * nodes, and the nodes that are ignored when the ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED
                                          * flag is not set.
                                          * Thus no corresponding NUMA node object may be found in the topology, because the
                                          * precise position is undefined. It is however known that it would be
                                          * somewhere under this object.
                                          *
                                          * If there are no NUMA nodes in the machine, all the memory is close to this
                                          * object, so only the first bit is set in \p complete_nodeset.
                                          *
                                          * \note Its value must not be changed, hwloc_bitmap_dup() must be used instead.
                                          */

  struct hwloc_info_s *infos;		/**< \brief Array of stringified info type=name. */
  unsigned infos_count;			/**< \brief Size of infos array. */

  /* misc */
  void *userdata;			/**< \brief Application-given private data pointer,
					 * initialized to \c NULL, use it as you wish.
					 * See hwloc_topology_set_userdata_export_callback() in hwloc/export.h
					 * if you wish to export this field to XML. */

  hwloc_uint64_t gp_index;			/**< \brief Global persistent index.
					 * Generated by hwloc, unique across the topology (contrary to os_index)
					 * and persistent across topology changes (contrary to logical_index).
					 * Mostly used internally, but could also be used by application to identify objects.
					 */
};
```cpp

I'm sorry, but I am unable to parse the previous message as it appears to be written in some sort of code or script rather than human language. Without the context and a clear definition of the variables used in this message, I am unable to provide any meaningful response. If you have any specific questions or if there is anything else I can help you with, please provide more context and I'll do my best to assist you.


```
/**
 * \brief Convenience typedef; a pointer to a struct hwloc_obj.
 */
typedef struct hwloc_obj * hwloc_obj_t;

/** \brief Object type-specific Attributes */
union hwloc_obj_attr_u {
  /** \brief NUMA node-specific Object Attributes */
  struct hwloc_numanode_attr_s {
    hwloc_uint64_t local_memory; /**< \brief Local memory (in bytes) */
    unsigned page_types_len; /**< \brief Size of array \p page_types */
    /** \brief Array of local memory page types, \c NULL if no local memory and \p page_types is 0.
     *
     * The array is sorted by increasing \p size fields.
     * It contains \p page_types_len slots.
     */
    struct hwloc_memory_page_type_s {
      hwloc_uint64_t size;	/**< \brief Size of pages */
      hwloc_uint64_t count;	/**< \brief Number of pages of this size */
    } * page_types;
  } numanode;

  /** \brief Cache-specific Object Attributes */
  struct hwloc_cache_attr_s {
    hwloc_uint64_t size;		  /**< \brief Size of cache in bytes */
    unsigned depth;			  /**< \brief Depth of cache (e.g., L1, L2, ...etc.) */
    unsigned linesize;			  /**< \brief Cache-line size in bytes. 0 if unknown */
    int associativity;			  /**< \brief Ways of associativity,
    					    *  -1 if fully associative, 0 if unknown */
    hwloc_obj_cache_type_t type;          /**< \brief Cache type */
  } cache;
  /** \brief Group-specific Object Attributes */
  struct hwloc_group_attr_s {
    unsigned depth;			  /**< \brief Depth of group object.
					   *   It may change if intermediate Group objects are added. */
    unsigned kind;			  /**< \brief Internally-used kind of group. */
    unsigned subkind;			  /**< \brief Internally-used subkind to distinguish different levels of groups with same kind */
    unsigned char dont_merge;		  /**< \brief Flag preventing groups from being automatically merged with identical parent or children. */
  } group;
  /** \brief PCI Device specific Object Attributes */
  struct hwloc_pcidev_attr_s {
```cpp

这段代码定义了一个名为`pcidev`的结构体，用于表示PCI设备的属性。以下是它的主要部分：

```
#ifndef HWLOC_HAVE_32BITS_PCI_DOMAIN
   unsigned short domain;          /* Only 16bits PCI domains are supported by default */
#else
   unsigned int domain;          /* 32bits PCI domain support break the library ABI, hence it's disabled by default */
#endif
   unsigned char bus, dev, func;
   unsigned short class_id;
   unsigned short vendor_id, device_id, subvendor_id, subdevice_id;
   unsigned char revision;
   float linkspeed;          /* in GB/s */
 } pcidev;
 
 /**
  * Bridge specific Object Attributes
  */
 struct hwloc_bridge_attr_s {
   union {
     struct hwloc_pcidev_attr_s pci;
   };
   hwloc_obj_bridge_type_t upstream_type;
   union {
     struct {
       uint32_t pci_domain;
       uint32_t device_subclass;
       uint32_t device_subclass_extended;
       uint32_t features;
       uint32_t( alignment ) device_driver_data;
       uint32_t first_device_superclass;
       uint32_t first_device_subclass;
       uint32_t first_device_subclass_extended;
       uint32_t rpc_compatible;
       uint32_t have_domain_pci;
       uint32_t domain_support;
       uint32_t controller_scope;
       uint32_t apm_domain;
       uint32_t supported_error_models;
       uint32_t default_action;
       uint32_t pci_device_id;
       uint32_t pci_device_subclass;
       uint32_t pci_device_subclass_extended;
       uint32_t pci_domain_remaining;
       uint32_t pci_domain_expanded;
       uint32_t pci_domain;
       uint32_t device_vendor_id;
       uint32_t device_device_id;
       uint32_t device_subdevice_id;
       uint32_t device_revision;
       uint32_t device_class_id;
       uint32_t device_subclass_extended;
       uint32_t device_subclass;
       uint32_t device_driver_data2;
       uint32_t first_device_link_速度；
       uint32_t supported_domain_雷峰；
       uint32_t alignment;
       uint32_t supported_link_speed;
       uint32_t num_device_destinations;
       uint32_t num_device_interfaces;
       uint32_t num_domain_connections;
       uint32_t num_function_groups;
       uint32_t num_device_revision;
       uint32_t first_function_group;
       uint32_t first_function_device_id;
       uint32_t function_superclass;
       uint32_t function_subclass;
       uint32_t function_subclass_extended;
       uint32_t function_domain;
       uint32_t function_remaining;
       uint32_t function_expanded;
       uint32_t function_device_id;
       uint32_t function_device_subclass;
       uint32_t function_device_subclass_extended;
       uint32_t function_device_domain;
       uint32_t function_device_remaining;
       uint32_t function_device_expanded;
       uint32_t link_speed;          /**< Link speed in GB/s */
     } pci;
   };
   hwloc_obj_bridge_type_t upstream_type;
   union {
     struct {
       uint32_t pci_domain;
       uint32_t device_subclass;
       uint32_t device_subclass_extended;
       uint32_t features;
       uint32_t( alignment ) device_driver_data;
       uint32_t first_device_superclass;
       uint32_t first_device_subclass;
       uint32_t first_device_subclass_extended;
       uint32_t rpc_compatible;
       uint32_t have_domain_pci;
       uint32_t domain_support;
       uint32_t controller_scope;
       uint32_t agpm_domain;
       uint32_t supported_error_models;
       uint32_t default_action;
       uint32_t pci_device_id;
       uint32_t pci_device_subclass;
       uint32_t pci_device_subclass_extended;
       uint32_t pci_domain_remaining;
       uint32_t pci_domain_expanded;
       uint32_t pci_domain;
       uint32_t device_vendor_id;
       uint32_t device_device_id;
       uint32_t device_subdevice_id;
       uint32_t device_revision;
       uint32_t device_class_id;
       uint32_t device_subclass_extended;
       uint32_t device_subclass;
       uint32_t device_driver_data2;
       uint32_t first_device_link_speed;
       uint32_t supported_domain_雷峰；
       uint32_t alignment;
       uint32_t supported_link_speed;
       uint32_t num_device_destinations;
       uint32_t num_device_interfaces;
       uint32_t num_domain_connections;
       uint32_t num_function_groups;
       uint32_t num_function_remaining;
       uint32_t first_function_group;
       uint32_t first_function_device_id;
       uint32_t function_superclass;
       uint32_t function_subclass;
       uint32_t function_subclass_extended;
       uint32_t function_domain;
       uint32_t function_remaining;
       uint32_t function_expanded;
       uint32_t function_device_id;
       uint32_t function_device_subclass;
       uint32_t function_device_subclass_extended;
       uint32_t function_device_domain;
       uint32_t function_device_remaining;
       uint32_t function_device_expanded;
       uint32_t link_speed;          /**< Link speed in GB/s */
     } pci;
   };
   hwloc_obj_bridge_attr_t bridge_attr;
   hwloc_obj_bridge_layout_32_t bridge_layout_32;
   hwloc_obj_bridge_specs_32_t bridge_specs_32;
 } bridge_device


```cpp
#ifndef HWLOC_HAVE_32BITS_PCI_DOMAIN
    unsigned short domain; /* Only 16bits PCI domains are supported by default */
#else
    unsigned int domain; /* 32bits PCI domain support break the library ABI, hence it's disabled by default */
#endif
    unsigned char bus, dev, func;
    unsigned short class_id;
    unsigned short vendor_id, device_id, subvendor_id, subdevice_id;
    unsigned char revision;
    float linkspeed; /* in GB/s */
  } pcidev;
  /** \brief Bridge specific Object Attributes */
  struct hwloc_bridge_attr_s {
    union {
      struct hwloc_pcidev_attr_s pci;
    } upstream;
    hwloc_obj_bridge_type_t upstream_type;
    union {
      struct {
```

这段代码定义了一个名为`HWLOC_HAVE_32BITS_PCI_DOMAIN`的宏，如果该宏定义存在，则定义了一个`unsigned short`类型的变量`domain`，其值为16位PCI域。否则，定义了一个`unsigned int`类型的变量`domain`，其值为32位PCI域支持，但该支持已被禁用。接着定义了`unsigned char`类型的变量`secondary_bus`和`subordinate_bus`，以及一个`hwloc_obj_bridge_type_t`类型的变量` downstream_type`和一个`hwloc_osdev_attr_s`类型的结构体`osdev`作为`bridge`对象的属性。`osdev`结构体定义了OS设备的属性，包括设备类型和属性等。最后，在`bridge`定义中，将`domain`变量作为桥片的默认属性，同时将`osdev`结构体作为桥片的附加属性。


```cpp
#ifndef HWLOC_HAVE_32BITS_PCI_DOMAIN
	unsigned short domain; /* Only 16bits PCI domains are supported by default */
#else
	unsigned int domain; /* 32bits PCI domain support break the library ABI, hence it's disabled by default */
#endif
	unsigned char secondary_bus, subordinate_bus;
      } pci;
    } downstream;
    hwloc_obj_bridge_type_t downstream_type;
    unsigned depth;
  } bridge;
  /** \brief OS Device specific Object Attributes */
  struct hwloc_osdev_attr_s {
    hwloc_obj_osdev_type_t type;
  } osdev;
};

```

这是一个 C 语言的代码，定义了一个名为 "hwlocality_info_s" 的结构体，用于存储与 HWLOCALITY 相关的信息。该结构体包含两个成员变量：name 和 value，分别用于存储 Info 名称和值。

同时，该代码未定义任何函数，但定义了一个名为 "hwlocality_creation" 的函数组，其中包含了一些函数。这些函数可能用于在程序中创建或销毁 HWLOCALITY。


```cpp
/** \brief Object info
 *
 * \sa hwlocality_info_attr
 */
struct hwloc_info_s {
  char *name;	/**< \brief Info name */
  char *value;	/**< \brief Info value */
};

/** @} */



/** \defgroup hwlocality_creation Topology Creation and Destruction
 * @{
 */

```

这段代码定义了一个名为`hwloc_topology`的结构体，用于表示topology context(即网络拓扑结构)。

接着定义了一个名为`hwloc_topology_t`的指针类型`topologyp`，用于存储`hwloc_topology`结构体。

接着定义了一个名为`hwloc_topology_init`的函数，用于初始化`hwloc_topology`结构体，并将返回值存储在`topologyp`指向的变量中。

然后定义了一个名为`hwloc_topology_load`的函数，用于从文件中读取`hwloc_topology`结构体，并将读取的topology从文件中读取到`topologyp`指向的变量中。

最后没有定义任何其他函数或变量，说明该代码已经定义好了所有的函数和变量。


```cpp
struct hwloc_topology;
/** \brief Topology context
 *
 * To be initialized with hwloc_topology_init() and built with hwloc_topology_load().
 */
typedef struct hwloc_topology * hwloc_topology_t;

/** \brief Allocate a topology context.
 *
 * \param[out] topologyp is assigned a pointer to the new allocated context.
 *
 * \return 0 on success, -1 on error.
 */
HWLOC_DECLSPEC int hwloc_topology_init (hwloc_topology_t *topologyp);

```

这段代码是一个C语言函数，名为`hwlocality_configuration_as_topology`或`hwlocality_setsource_as_topology`，其作用是在给定的topology对象上执行hwlocality_configuration和hwlocality_setsource设置，然后返回一个0或-1的错误码。

该函数的参数`topology`是一个表示要加载的topology对象的指针，它会在初始化时通过hwloc_topology_init()函数设置。在函数实现中，该topology对象将用于后续调用hwlocality_configuration和hwlocality_setsource函数。

该函数的一个使用注意是，在函数失败时，topology将被重新初始化。这可以通过hwloc_topology_destroy()函数来实现，或者在调用hwlocality_configuration和hwlocality_setsource函数之后，根据需要重新初始化topology。


```cpp
/** \brief Build the actual topology
 *
 * Build the actual topology once initialized with hwloc_topology_init() and
 * tuned with \ref hwlocality_configuration and \ref hwlocality_setsource routines.
 * No other routine may be called earlier using this topology context.
 *
 * \param topology is the topology to be loaded with objects.
 *
 * \return 0 on success, -1 on error.
 *
 * \note On failure, the topology is reinitialized. It should be either
 * destroyed with hwloc_topology_destroy() or configured and loaded again.
 *
 * \note This function may be called only once per topology.
 *
 * \note The binding of the current thread or process may temporarily change
 * during this call but it will be restored before it returns.
 *
 * \sa hwlocality_configuration and hwlocality_setsource
 */
```

这段代码定义了两个函数，分别是`hwloc_topology_load`和`hwloc_topology_destroy`。这两个函数是用于管理topology结构体的。

`hwloc_topology_load`函数接受一个`hwloc_topology_t`类型的参数`topology`，用于保存输入的topology结构体。函数的返回值是一个整数类型，表示topology结构体是否成功加载。如果topology结构体加载成功，则返回0，否则返回非0。

`hwloc_topology_destroy`函数接受一个`hwloc_topology_t`类型的参数`topology`，用于指定要释放的topology结构体。函数的返回值是一个void类型的参数，用于指定成功或失败。如果成功释放topology结构体，则函数返回，否则返回。

这两个函数一起用于管理topology结构体，可以实现对topology结构体的复制、移动、修改等操作，使得整个topology结构在需要时可以被备份或恢复。


```cpp
HWLOC_DECLSPEC int hwloc_topology_load(hwloc_topology_t topology);

/** \brief Terminate and free a topology context
 *
 * \param topology is the topology to be freed
 */
HWLOC_DECLSPEC void hwloc_topology_destroy (hwloc_topology_t topology);

/** \brief Duplicate a topology.
 *
 * The entire topology structure as well as its objects
 * are duplicated into a new one.
 *
 * This is useful for keeping a backup while modifying a topology.
 *
 * \note Object userdata is not duplicated since hwloc does not know what it point to.
 * The objects of both old and new topologies will point to the same userdata.
 */
```

这段代码定义了一个名为 `hwloc_topology_dup` 的函数，属于 `hwloc_topology_t` 类型。

该函数接收两个参数：

1. `newtopology`：要创建的新topology结构体的指针；
2. `oldtopology`：要检查的旧topology结构体的指针。

函数的作用是验证当前topology是否与当前hwloc库兼容。在某些情况下，使用相同的topology结构可能需要在不同的库中使用，这时候就需要通过这种函数来检查两个库是否兼容。

如果两个库使用的是同一个hwloc安装，那么该函数会返回成功，否则会返回-1和错误码。


```cpp
HWLOC_DECLSPEC int hwloc_topology_dup(hwloc_topology_t *newtopology, hwloc_topology_t oldtopology);

/** \brief Verify that the topology is compatible with the current hwloc library.
 *
 * This is useful when using the same topology structure (in memory)
 * in different libraries that may use different hwloc installations
 * (for instance if one library embeds a specific version of hwloc,
 * while another library uses a default system-wide hwloc installation).
 *
 * If all libraries/programs use the same hwloc installation, this function
 * always returns success.
 *
 * \return \c 0 on success.
 *
 * \return \c -1 with \p errno set to \c EINVAL if incompatible.
 *
 * \note If sharing between processes with hwloc_shmem_topology_write(),
 * the relevant check is already performed inside hwloc_shmem_topology_adopt().
 */
```

这两段代码定义了一个名为 "hwloc_topology_abi_check" 的函数，属于 "hwloc_topology_t" 结构的函数，作用是检查传入 topology 结构是否与之前的 topology 结构一致，如果不一致，函数将打印错误并中止程序。如果两个 topology 结构相同，函数将不做任何处理。

另外，还定义了一个名为 "hwloc_topology_check" 的函数，与上述函数具有相同的函数名，但参数不同，其作用是打印错误信息并中止程序。


```cpp
HWLOC_DECLSPEC int hwloc_topology_abi_check(hwloc_topology_t topology);

/** \brief Run internal checks on a topology structure
 *
 * The program aborts if an inconsistency is detected in the given topology.
 *
 * \param topology is the topology to be checked
 *
 * \note This routine is only useful to developers.
 *
 * \note The input topology should have been previously loaded with
 * hwloc_topology_load().
 */
HWLOC_DECLSPEC void hwloc_topology_check(hwloc_topology_t topology);

```

这段代码定义了一个名为 `hwlocality_levels` 的自定义类别，提供了对对象级别的层次结构。该类别定义了对象的深度、层数和类型。

具体来说，该自定义类别定义了一个名为 `HWLOC_OBJ_PU` 的常量，表示普通用户对象（可能包括 CPU、GPU 等）。然后，通过这个常量创建了一个名为 `HWLOC_OBJ_NUMA` 的常量，表示 NUMA 节点对象。此外，还定义了一个名为 `HWLOC_OBJ_IOP` 的常量，表示 I/O 对象。最后，定义了一个名为 `HWLOC_OBJ_MISC` 的常量，表示 misc 对象。

接着，定义了一个名为 `get_object_depth` 的函数，该函数接收一个 `HWLOC_OBJ_PU` 对象作为参数，计算该对象的深度。最后，返回该对象的深度加一。

该代码的作用是提供一个自定义的类，用于计算对象层次结构中的深度。该自定义类别定义了不同类型的对象，如 CPU、GPU、NUMA、I/O 和 misc，以及它们对应的层次结构。通过这个自定义类别，可以方便地计算对象的层次结构，以及在不同层次结构之间进行通信和交互。


```cpp
/** @} */



/** \defgroup hwlocality_levels Object levels, depths and types
 * @{
 *
 * Be sure to see the figure in \ref termsanddefs that shows a
 * complete topology tree, including depths, child/sibling/cousin
 * relationships, and an example of an asymmetric topology where one
 * package has fewer caches than its peers.
 */

/** \brief Get the depth of the hierarchical tree of objects.
 *
 * This is the depth of ::HWLOC_OBJ_PU objects plus one.
 *
 * \note NUMA nodes, I/O and Misc objects are ignored when computing
 * the depth of the tree (they are placed on special levels).
 */
```

这段代码定义了一个名为 `hwloc_topology_get_depth` 的函数，它接受一个 `hwloc_topology_t` 的输入参数，并返回一个整数类型的深度。这个深度是基于系统架构中实际存在的硬件资源，如 CPU、GPU、内存等，或者是基于某些抽象类型（如 `hwloc_device_t`、`hwloc_device_prop_t`、`hwloc_device_index_t`）的深度，或者是基于某些特定的逻辑 ID（如 `hwloc_obj_group_t`、`hwloc_memory_device_t`）。

该函数的行为取决于输入的 `hwloc_topology_t` 参数，可能会返回不同的深度值。例如，如果输入的 `hwloc_device_t` 参数存在，则可能会返回 `hwloc_device_topology_get_depth()` 函数的返回值，这个函数会尝试查找与给定 `hwloc_device_t` 相关的 `hwloc_device_prop_t` 记录，以获取该设备的 `hwloc_device_index_t` 成员的深度。如果这个 `hwloc_device_prop_t` 记录不存在，则函数可能会返回一个默认值，如 `HWLOC_TYPE_DEPTH_UNKNOWN`。

对于 `hwloc_device_t` 和 `hwloc_device_prop_t` 这两个参数，函数可能会按照一定的优先级进行查找，例如，首先查找 `hwloc_device_t`，如果没有找到，则尝试查找 `hwloc_device_prop_t`，如果仍然找不到，则返回一个默认值。

如果输入的 `hwloc_topology_t` 参数中包含 `hwloc_device_t`、`hwloc_device_prop_t` 或 `hwloc_device_index_t`，则函数可能会返回一个虚拟的深度值，这个值并不是基于实际的硬件资源，也不应该被认为是一个真实的深度。

该函数可能会被用于在应用程序中获取与给定 `hwloc_topology_t` 相关的深度信息，例如，确定一个 `hwloc_device_t` 设备在系统中的层次结构、或者在多层 `hwloc_device_group_t` 中的深度等。


```cpp
HWLOC_DECLSPEC int hwloc_topology_get_depth(hwloc_topology_t __hwloc_restrict topology) __hwloc_attribute_pure;

/** \brief Returns the depth of objects of type \p type.
 *
 * If no object of this type is present on the underlying architecture, or if
 * the OS doesn't provide this kind of information, the function returns
 * ::HWLOC_TYPE_DEPTH_UNKNOWN.
 *
 * If type is absent but a similar type is acceptable, see also
 * hwloc_get_type_or_below_depth() and hwloc_get_type_or_above_depth().
 *
 * If ::HWLOC_OBJ_GROUP is given, the function may return ::HWLOC_TYPE_DEPTH_MULTIPLE
 * if multiple levels of Groups exist.
 *
 * If a NUMA node, I/O or Misc object type is given, the function returns a virtual
 * value because these objects are stored in special levels that are not CPU-related.
 * This virtual depth may be passed to other hwloc functions such as
 * hwloc_get_obj_by_depth() but it should not be considered as an actual
 * depth by the application. In particular, it should not be compared with
 * any other object depth or with the entire topology depth.
 * \sa hwloc_get_memory_parents_depth().
 *
 * \sa hwloc_type_sscanf_as_depth() for returning the depth of objects
 * whose type is given as a string.
 */
```

HWLOC_TYPE_DEPTH_MULTIPLE is a constant defined in the `hwloc_type_defs.h` header file that specifies the type of objects that exist at different depths in the topology. It is used to identify the depth of objects that have a specific type.

For example, if a group of objects with the `HWLOC_TYPE_DEPTH_MULTIPLE` type exist at different depths in the topology, it means that these objects exist at different levels of the hierarchical tree of objects.

It is important to note that the `HWLOC_TYPE_DEPTH_MULTIPLE` type is only applicable for objects that have a specific type defined in the `hwloc_type_defs.h` header file. If a custom object with a similar name exists in the topology, it should be implemented according to the defined type.


```cpp
HWLOC_DECLSPEC int hwloc_get_type_depth (hwloc_topology_t topology, hwloc_obj_type_t type);

enum hwloc_get_type_depth_e {
    HWLOC_TYPE_DEPTH_UNKNOWN = -1,    /**< \brief No object of given type exists in the topology. \hideinitializer */
    HWLOC_TYPE_DEPTH_MULTIPLE = -2,   /**< \brief Objects of given type exist at different depth in the topology (only for Groups). \hideinitializer */
    HWLOC_TYPE_DEPTH_NUMANODE = -3,   /**< \brief Virtual depth for NUMA nodes. \hideinitializer */
    HWLOC_TYPE_DEPTH_BRIDGE = -4,     /**< \brief Virtual depth for bridge object level. \hideinitializer */
    HWLOC_TYPE_DEPTH_PCI_DEVICE = -5, /**< \brief Virtual depth for PCI device object level. \hideinitializer */
    HWLOC_TYPE_DEPTH_OS_DEVICE = -6,  /**< \brief Virtual depth for software device object level. \hideinitializer */
    HWLOC_TYPE_DEPTH_MISC = -7,       /**< \brief Virtual depth for Misc object. \hideinitializer */
    HWLOC_TYPE_DEPTH_MEMCACHE = -8    /**< \brief Virtual depth for MemCache object. \hideinitializer */
};

/** \brief Return the depth of parents where memory objects are attached.
 *
 * Memory objects have virtual negative depths because they are not part of
 * the main CPU-side hierarchy of objects. This depth should not be compared
 * with other level depths.
 *
 * If all Memory objects are attached to Normal parents at the same depth,
 * this parent depth may be compared to other as usual, for instance
 * for knowing whether NUMA nodes is attached above or below Packages.
 *
 * \return The depth of Normal parents of all memory children
 * if all these parents have the same depth. For instance the depth of
 * the Package level if all NUMA nodes are attached to Package objects.
 *
 * \return ::HWLOC_TYPE_DEPTH_MULTIPLE if Normal parents of all
 * memory children do not have the same depth. For instance if some
 * NUMA nodes are attached to Packages while others are attached to
 * Groups.
 */
```

这段代码定义了一个名为hwloc_get_memory_parents_depth的函数，其属于hwloc_topology_t类型的函数。该函数返回的是具有指定类型或更少对象的底层架构中，第一个出现此类对象的深度。如果没有这种类型的对象，函数将返回通常在底层架构中发现的第一个具有该类型对象的深度。

该函数仅对正常对象类型有意义。如果给定了内存、I/O或无类别对象类型，则返回相应的虚拟深度。在函数内部，可能还会返回HWLOC_TYPE_DEPTH_MULTIPLE，就像在hwloc_get_type_depth()函数中一样。


```cpp
HWLOC_DECLSPEC int hwloc_get_memory_parents_depth (hwloc_topology_t topology);

/** \brief Returns the depth of objects of type \p type or below
 *
 * If no object of this type is present on the underlying architecture, the
 * function returns the depth of the first "present" object typically found
 * inside \p type.
 *
 * This function is only meaningful for normal object types.
 * If a memory, I/O or Misc object type is given, the corresponding virtual
 * depth is always returned (see hwloc_get_type_depth()).
 *
 * May return ::HWLOC_TYPE_DEPTH_MULTIPLE for ::HWLOC_OBJ_GROUP just like
 * hwloc_get_type_depth().
 */
```

这段代码定义了一个名为 `hwloc_get_type_or_below_depth` 的函数，属于 `__hwloc_inline` 类型。它接受两个参数，一个是 `hwloc_topology_t` 类型的 `topology`，另一个是 `hwloc_obj_type_t` 类型的 `type`。函数返回深度为 `hwloc_get_type_depth` 的函数的调用返回值的最低深度。

如果 `topology` 对应的架构中没有 object 类型为 `type` 的物体，那么函数将返回包含 `type` 的对象的深度。这个函数仅在正常对象类型有意义时有意义。如果给予的是 memory、i/o 或 misc 类型，那么相应的虚拟深度将返回，就像 `hwloc_get_type_depth` 函数一样（在同一文档中有详细说明）。函数可能还返回 `hwloc_type_depth_multiply`，与 `hwloc_get_type_depth` 函数返回的相同值，只是文档中没有明确规定。


```cpp
static __hwloc_inline int
hwloc_get_type_or_below_depth (hwloc_topology_t topology, hwloc_obj_type_t type) __hwloc_attribute_pure;

/** \brief Returns the depth of objects of type \p type or above
 *
 * If no object of this type is present on the underlying architecture, the
 * function returns the depth of the first "present" object typically
 * containing \p type.
 *
 * This function is only meaningful for normal object types.
 * If a memory, I/O or Misc object type is given, the corresponding virtual
 * depth is always returned (see hwloc_get_type_depth()).
 *
 * May return ::HWLOC_TYPE_DEPTH_MULTIPLE for ::HWLOC_OBJ_GROUP just like
 * hwloc_get_type_depth().
 */
```

这两段代码定义了两个名为 "hwloc_get_type_or_above_depth" 和 "hwloc_get_depth_type" 的函数，以及它们的实现在代码中。

第一个函数 "hwloc_get_type_or_above_depth" 接收两个参数：一个 "hwloc_topology_t" 类型的上下文对象和一个 "hwloc_obj_type_t" 类型的要获取的对象类型。函数返回 depth 对应的上下文对象的类型，或者深度为 0 时返回 -1。函数实现了一个 pure 函数，即不产生任何副作用(副作用在这里可以理解为副作用是一个整数类型)。

第二个函数 "hwloc_get_depth_type" 同样接收两个参数：一个 "hwloc_topology_t" 类型的上下文对象和一个整数类型的深度。函数返回 depth 对应的上下文对象的类型，或者是 -1(如果深度超出了上下文对象的类型范围)。函数实现了一个 pure 函数，即不产生任何副作用。

第三个函数 "hwloc_get_nbobjs_by_depth" 同样接收两个参数：一个 "hwloc_topology_t" 类型的上下文对象和一个整数类型的深度。函数返回 depth 对应的节点数，即节点在层次结构中的位置。函数实现了一个 pure 函数，即不产生任何副作用。


```cpp
static __hwloc_inline int
hwloc_get_type_or_above_depth (hwloc_topology_t topology, hwloc_obj_type_t type) __hwloc_attribute_pure;

/** \brief Returns the type of objects at depth \p depth.
 *
 * \p depth should between 0 and hwloc_topology_get_depth()-1,
 * or a virtual depth such as ::HWLOC_TYPE_DEPTH_NUMANODE.
 *
 * \return (hwloc_obj_type_t)-1 if depth \p depth does not exist.
 */
HWLOC_DECLSPEC hwloc_obj_type_t hwloc_get_depth_type (hwloc_topology_t topology, int depth) __hwloc_attribute_pure;

/** \brief Returns the width of level at depth \p depth.
 */
HWLOC_DECLSPEC unsigned hwloc_get_nbobjs_by_depth (hwloc_topology_t topology, int depth) __hwloc_attribute_pure;

```

这两段代码定义了两个函数，用于在水平布局（水平结构）中获取不同类型的对象数量。

函数1：`hwloc_get_nbobjs_by_type`
该函数接收两个参数：`topology` 和 `type`，它们都来自于 `hwloc_topology_t` 类型。函数返回两种情况下的对象数量：

1. 如果 `topology` 对象中没有任何与 `type` 相同类型的对象，那么返回 0。
2. 如果 `topology` 对象中有多个与 `type` 相同类型的对象，那么返回这些对象的 ID（对象标识符）。

函数2：`hwloc_get_root_obj`
该函数接收一个参数 `topology`，它是一个来自于 `hwloc_topology_t` 的结构体。函数返回根对象（顶级对象，即最顶层的对象）的 ID。

这两段代码共同作用于创建和返回水平布局中的对象，并允许用户根据所需的对象类型进行水平布局。


```cpp
/** \brief Returns the width of level type \p type
 *
 * If no object for that type exists, 0 is returned.
 * If there are several levels with objects of that type, -1 is returned.
 */
static __hwloc_inline int
hwloc_get_nbobjs_by_type (hwloc_topology_t topology, hwloc_obj_type_t type) __hwloc_attribute_pure;

/** \brief Returns the top-object of the topology-tree.
 *
 * Its type is ::HWLOC_OBJ_MACHINE.
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_root_obj (hwloc_topology_t topology) __hwloc_attribute_pure;

```

这段代码定义了两个函数，用于从给定的深度和逻辑索引返回 topology 对象。

第一个函数名为 `hwloc_get_obj_by_depth`，它接收一个 topology 类型和一个深度作为参数，然后返回在逻辑索引 `idx` 下的 topology 对象的引用。这个函数使用了 `__hwloc_attribute_pure` 修饰，表示函数内部计算结果时不需要考虑 `__hwloc_sup_msg` 函数。

第二个函数名为 `hwloc_get_obj_by_type`，它接收一个 topology 类型和一个类型作为参数，然后返回一个指向类型指定对象的指针，或者 `NULL`。如果类型不存在，函数返回 `NULL`，否则函数使用 `hwloc_get_obj_by_depth` 函数返回逻辑索引 `idx` 下的 topology 对象。同样地，这个函数也使用了 `__hwloc_attribute_pure` 修饰。

这两个函数可能会在某些情况下被用于搜索或遍历 topology 中的对象。


```cpp
/** \brief Returns the topology object at logical index \p idx from depth \p depth */
HWLOC_DECLSPEC hwloc_obj_t hwloc_get_obj_by_depth (hwloc_topology_t topology, int depth, unsigned idx) __hwloc_attribute_pure;

/** \brief Returns the topology object at logical index \p idx with type \p type
 *
 * If no object for that type exists, \c NULL is returned.
 * If there are several levels with objects of that type (::HWLOC_OBJ_GROUP),
 * \c NULL is returned and the caller may fallback to hwloc_get_obj_by_depth().
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_by_type (hwloc_topology_t topology, hwloc_obj_type_t type, unsigned idx) __hwloc_attribute_pure;

/** \brief Returns the next object at depth \p depth.
 *
 * If \p prev is \c NULL, return the first object at depth \p depth.
 */
```

这段代码定义了两个名为`hwloc_get_next_obj_by_depth`和`hwloc_get_next_obj_by_type`的函数，它们的实现都基于`hwloc_topology_t`和`hwloc_obj_t`这两种数据结构。

`hwloc_get_next_obj_by_depth`函数接收一个`hwloc_topology_t`表示当前对象的topology，以及一个表示当前深度层级的整数depth，函数返回下一个具有该depth层级的`hwloc_obj_t`对象的引用。函数使用`hwloc_topology_t`的`get_next_obj_by_depth`函数作为内部实现，这个函数会遍历当前对象的depth层级的所有`hwloc_obj_t`对象，找到第一个实现该深度层级的`hwloc_obj_t`对象，并返回它的引用。如果当前对象是`hwloc_null_obj_t`，函数会返回`hwloc_null_obj_t`。

`hwloc_get_next_obj_by_type`函数与`hwloc_get_next_obj_by_depth`函数类似，但它们的实现存在一定差异。`hwloc_get_next_obj_by_type`函数接收一个`hwloc_topology_t`表示当前对象的topology，以及一个表示当前对象的`hwloc_obj_type_t`类型，函数返回下一个具有该type的`hwloc_obj_t`对象的引用。函数使用`hwloc_topology_t`的`get_next_obj_by_type`函数作为内部实现，这个函数会遍历当前对象的`hwloc_obj_t`类型的所有对象，找到第一个实现该type的`hwloc_obj_t`对象，并返回它的引用。如果当前对象是`hwloc_null_obj_t`，函数会返回`hwloc_null_obj_t`。


```cpp
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_by_depth (hwloc_topology_t topology, int depth, hwloc_obj_t prev);

/** \brief Returns the next object of type \p type.
 *
 * If \p prev is \c NULL, return the first object at type \p type.  If
 * there are multiple or no depth for given type, return \c NULL and
 * let the caller fallback to hwloc_get_next_obj_by_depth().
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_by_type (hwloc_topology_t topology, hwloc_obj_type_t type,
			    hwloc_obj_t prev);

/** @} */



```

这段代码定义了一个名为hwlocality_object_strings的函数组，用于将对象类型转换为字符串表示，并返回一个常量字符串ified object type。

函数hwlocality_obj_type_string接受一个hwloc_obj_type_t类型的参数，并返回一个指向该类型的字符串常量的指针。这个函数可以用于将对象类型转换为更容易阅读和理解的人类可读形式，而不需要了解具体的对象类型和属性。

函数内部包含一个使用hwloc_type_sscanf()的更精确的输出，用于将对象的类型字符串表示转换为具体的对象类型。但是，为了使用这个更精确的输出，用户需要提供输出缓冲区。

函数还包含一个用于打印对象类型字符串的宏，hwloc_obj_type_string_print，该宏将在编译时将字符串常量打印到标准输出。

最后，函数还包含一个名为size的局部变量，用于指定在输出时需要打印的对象特定属性（如Bridge类型或OS设备类型）。如果size的值为0，则函数返回一个指向字符串的指针，而不是实际打印的字符数。


```cpp
/** \defgroup hwlocality_object_strings Converting between Object Types and Attributes, and Strings
 * @{
 */

/** \brief Return a constant stringified object type.
 *
 * This function is the basic way to convert a generic type into a string.
 * The output string may be parsed back by hwloc_type_sscanf().
 *
 * hwloc_obj_type_snprintf() may return a more precise output for a specific
 * object, but it requires the caller to provide the output buffer.
 */
HWLOC_DECLSPEC const char * hwloc_obj_type_string (hwloc_obj_type_t type) __hwloc_attribute_const;

/** \brief Stringify the type of a given topology object into a human-readable form.
 *
 * Contrary to hwloc_obj_type_string(), this function includes object-specific
 * attributes (such as the Group depth, the Bridge type, or OS device type)
 * in the output, and it requires the caller to provide the output buffer.
 *
 * The output is guaranteed to be the same for all objects of a same topology level.
 *
 * If \p verbose is 1, longer type names are used, e.g. L1Cache instead of L1.
 *
 * The output string may be parsed back by hwloc_type_sscanf().
 *
 * If \p size is 0, \p string may safely be \c NULL.
 *
 * \return the number of characters that were actually written if not truncating,
 * or that would have been written (not including the ending \\0).
 */
```

这段代码定义了一个名为HWLOC_DECLSPEC的int型变量hwloc_obj_type_snprintf，它的作用是将以字符串形式打印指定topology对象的属性，使人类更容易理解。

在函数参数中，有两个参数：
1. string：一个字符数组，用于存储要打印的属性的字符串格式。这个参数是一个指向字符数组的指针，但数组的大小必须是size_t类型的整数，可能需要在调用此函数时进行强制类型转换。
2. size：一个表示字符串长度的整数。这个参数告诉函数在打印字符串时如何处理多行字符。注意，如果size为0，则表示字符串可能太长，可能需要进行截断，但仍然可以接受。
3. obj：一个hwloc_obj_t类型的参数，指代要打印的topology对象的引用。
4. verbose：一个表示打印verbose模式下输出的int类型参数。如果这个参数为0，则表示只打印主要属性；否则，函数将输出更多的信息。

函数实现中，首先定义了一个以size为长度的字符数组，用于存储要打印的属性的字符串格式。然后，定义了一个名为obj的hwloc_obj_t类型的参数，用于存储要打印的topology对象的引用。

接着，函数内部使用size_t类型的变量size来存储字符串长度，并在打印函数体内部使用size参数来获取字符串长度。通过使用size参数中的值，可以判断整个字符串是否仅包含一个主要属性，如果是，则只打印该属性；如果不是，则打印所有主要属性。

函数返回实际打印的字符数，如果未截断字符串，则返回字符串长度减1，否则返回函数定义时给定的size参数。


```cpp
HWLOC_DECLSPEC int hwloc_obj_type_snprintf(char * __hwloc_restrict string, size_t size,
					   hwloc_obj_t obj,
					   int verbose);

/** \brief Stringify the attributes of a given topology object into a human-readable form.
 *
 * Attribute values are separated by \p separator.
 *
 * Only the major attributes are printed in non-verbose mode.
 *
 * If \p size is 0, \p string may safely be \c NULL.
 *
 * \return the number of characters that were actually written if not truncating,
 * or that would have been written (not including the ending \\0).
 */
```

这段代码定义了一个名为 `hwloc_obj_attr_snprintf` 的函数，用于从类型字符串中解析对象属性的属名和属性，并将其存储为 `hwloc_obj_t` 类型的数据结构。

这个函数接受三个参数：

1. `string`：一个字符数组，用于存储类型字符串。
2. `size_t`：一个表示字符串的大小的整数。
3. `obj`：一个 `hwloc_obj_t` 类型的对象，用于获取属性的值。
4. `separator`：一个字符串，用于分隔属性的属名和属性值之间的键值对。
5. `verbose`：一个表示是否输出调试信息的整数。

函数的作用是将类型字符串解析为 `hwloc_obj_t` 类型的数据结构，其中 `separator` 用于将属性值拆分为属性名和属性值。函数的返回值是 `0`，如果成功将类型字符串解析为 `hwloc_obj_t`，否则返回 `-1`。


```cpp
HWLOC_DECLSPEC int hwloc_obj_attr_snprintf(char * __hwloc_restrict string, size_t size,
					   hwloc_obj_t obj, const char * __hwloc_restrict separator,
					   int verbose);

/** \brief Return an object type and attributes from a type string.
 *
 * Convert strings such as "Package" or "L1iCache" into the corresponding types.
 * Matching is case-insensitive, and only the first letters are actually
 * required to match.
 *
 * The matched object type is set in \p typep (which cannot be \c NULL).
 *
 * Type-specific attributes, for instance Cache type, Cache depth, Group depth,
 * Bridge type or OS Device type may be returned in \p attrp.
 * Attributes that are not specified in the string (for instance "Group"
 * without a depth, or "L2Cache" without a cache type) are set to -1.
 *
 * \p attrp is only filled if not \c NULL and if its size specified in \p attrsize
 * is large enough. It should be at least as large as union hwloc_obj_attr_u.
 *
 * \return 0 if a type was correctly identified, otherwise -1.
 *
 * \note This function is guaranteed to match any string returned by
 * hwloc_obj_type_string() or hwloc_obj_type_snprintf().
 *
 * \note This is an extended version of the now deprecated hwloc_obj_type_sscanf().
 */
```

这段代码定义了一个名为 `hwloc_type_sscanf` 的函数，它的参数是一个字符串 `string`，它用来查找与该字符串匹配的 `hwloc_obj_type_t` 类型的实例，并将匹配的类型和其深度存储在 `typep` 和 `attrp` 两个整型变量中，其中 `attrsize` 是存储 `typep` 和 `attrp` 所占用的空间大小的整型变量。函数返回值是匹配到的 `hwloc_obj_type_t` 类型和对应的深度。

该函数的作用是将一个字符串映射到相应的 `hwloc_obj_type_t` 类型实例，其中该字符串可以是任何具有 `hwloc_obj_type_string()` 或 `hwloc_obj_type_snprintf()` 函数生成的字符串。函数将返回匹配到的 `hwloc_obj_type_t` 类型实例和对应的深度，如果 underlying 架构上没有该类型，函数将返回 `HWLOC_TYPE_DEPTH_UNKNOWN`。如果存在多个具有相同字符串的类型，函数将尝试返回 `HWLOC_TYPE_DEPTH_MULTIPLE`，但不会成功。函数的实现与 `hwloc_type_sscanf()` 函数相似，但还实现了自动 disambiguation（歧义排除）的功能，该功能可以处理字符串中可能存在多个具有相同字符的不同 `hwloc_obj_type_t` 类型的层级结构。


```cpp
HWLOC_DECLSPEC int hwloc_type_sscanf(const char *string,
				     hwloc_obj_type_t *typep,
				     union hwloc_obj_attr_u *attrp, size_t attrsize);

/** \brief Return an object type and its level depth from a type string.
 *
 * Convert strings such as "Package" or "L1iCache" into the corresponding types
 * and return in \p depthp the depth of the corresponding level in the
 * topology \p topology.
 *
 * If no object of this type is present on the underlying architecture,
 * ::HWLOC_TYPE_DEPTH_UNKNOWN is returned.
 *
 * If multiple such levels exist (for instance if giving Group without any depth),
 * the function may return ::HWLOC_TYPE_DEPTH_MULTIPLE instead.
 *
 * The matched object type is set in \p typep if \p typep is non \c NULL.
 *
 * \note This function is similar to hwloc_type_sscanf() followed
 * by hwloc_get_type_depth() but it also automatically disambiguates
 * multiple group levels etc.
 *
 * \note This function is guaranteed to match any string returned by
 * hwloc_obj_type_string() or hwloc_obj_type_snprintf().
 */
```

这段代码定义了一个名为HWLOC_DECLSPEC的int类型，用于从给定的字符串中解析出深度信息，并将其存储在hwloc_obj_type_t类型的变量typep中，存储在hwloc_topology_t类型的变量topology中，同时存储在整型变量depthp中。

具体来说，这段代码的作用是：通过传入一个字符串，获取其中的深度信息，然后将其存储在typep、topology和depthp中。这个函数可以在程序中用于搜索和提取深度信息，例如获取一个子网卡的深度、提取一个字符串中的所有深度信息等等。


```cpp
HWLOC_DECLSPEC int hwloc_type_sscanf_as_depth(const char *string,
					      hwloc_obj_type_t *typep,
					      hwloc_topology_t topology, int *depthp);

/** @} */



/** \defgroup hwlocality_info_attr Consulting and Adding Key-Value Info Attributes
 *
 * @{
 */

/** \brief Search the given key name in object infos and return the corresponding value.
 *
 * If multiple keys match the given name, only the first one is returned.
 *
 * \return \c NULL if no such key exists.
 */
```

这段代码定义了一个名为 `hwloc_obj_get_info_by_name` 的函数，属于 `__hwloc_attribute_pure` 修饰的函数，这意味着它的作用域仅限于定义它的源文件中。

该函数接收两个参数，一个是 `hwloc_obj_t` 类型的对象，另一个是字符串 `name`，函数返回一个指向该对象的指针，该对象中包含名为 `name` 的信息。如果这个名为 `name` 的信息在对象中已经存在，函数将返回该信息对象的指针。否则，函数将把输入的名称和值对添加到对象中的现有信息中。

函数内部使用 `const char *` 类型的参数来存储要添加的信息名称和值，然后对传入的 `name` 进行截断，去除任何非打印字符。接着，函数将这些信息名称和值复制到 `obj` 对象中，如果 `name` 已经存在的话，将把已有的信息与新的信息合并，否则，在 `obj` 对象中添加新的信息。

最后，函数返回 0，表示操作成功，返回 -1，表示操作失败。


```cpp
static __hwloc_inline const char *
hwloc_obj_get_info_by_name(hwloc_obj_t obj, const char *name) __hwloc_attribute_pure;

/** \brief Add the given info name and value pair to the given object.
 *
 * The info is appended to the existing info array even if another key
 * with the same name already exists.
 *
 * The input strings are copied before being added in the object infos.
 *
 * \return \c 0 on success, \c -1 on error.
 *
 * \note This function may be used to enforce object colors in the lstopo
 * graphical output by using "lstopoStyle" as a name and "Background=#rrggbb"
 * as a value. See CUSTOM COLORS in the lstopo(1) manpage for details.
 *
 * \note If \p value contains some non-printable characters, they will
 * be dropped when exporting to XML, see hwloc_topology_export_xml() in hwloc/export.h.
 */
```

This is a function that provides a query of the current CPU binding support in an operating system. It takes a single argument, which is an integer `topology`, which specifies the type of topology to use.

The function returns `-1` if the requested binding operation is not available. If the `::HWLOC_CPUBIND_STRICT` flag is passed, the function returns `-1`, indicating that the requested binding operation is not supported.

If the requested binding operation is `HWLOC_CPUBIND_STRICT`, the function will return an error code if the binding cannot be done, or the error code will be `ENOSYS` if it is not possible to bind the requested kind of object. If the `::HWLOC_CPUBIND_NOMEMBIND` flag is passed, the function will return an error code if the binding operation has side-effects on memory binding.

If the binding operation is `HWLOC_CPUBIND_THREAD`, the function will return an error code if the binding cannot be done, or the error code will be `EXDEV` if the requested CPU cannot be enforced.

If `::HWLOC_CPUBIND_STRICT` is not passed, the function may fail or the operating system may use a slightly different operation when the requested binding operation is not exactly supported.

The most portable version of this function is the following one, which just binds the current program, assuming it is single-threaded:
```cppphp
hwloc_set_cpubind(topology, set, 0);
```
If the program may be multithreaded, the following one should be preferred, to only bind the current thread:
```cppphp
hwloc_set_cpubind(topology, set, HWLOC_CPUBIND_THREAD);
```
Note:

* The `errno` variable is set to `ENOSYS` if the requested binding operation is not supported.
* When the requested binding operation is not available and the `::HWLOC_CPUBIND_STRICT` flag was passed, the function returns `-1`.
* Some example codes are available under doc/examples/ in the source tree.


```cpp
HWLOC_DECLSPEC int hwloc_obj_add_info(hwloc_obj_t obj, const char *name, const char *value);

/** @} */



/** \defgroup hwlocality_cpubinding CPU binding
 *
 * Some operating systems only support binding threads or processes to a single PU.
 * Others allow binding to larger sets such as entire Cores or Packages or
 * even random sets of individual PUs. In such operating system, the scheduler
 * is free to run the task on one of these PU, then migrate it to another PU, etc.
 * It is often useful to call hwloc_bitmap_singlify() on the target CPU set before
 * passing it to the binding function to avoid these expensive migrations.
 * See the documentation of hwloc_bitmap_singlify() for details.
 *
 * Some operating systems do not provide all hwloc-supported
 * mechanisms to bind processes, threads, etc.
 * hwloc_topology_get_support() may be used to query about the actual CPU
 * binding support in the currently used operating system.
 *
 * When the requested binding operation is not available and the
 * ::HWLOC_CPUBIND_STRICT flag was passed, the function returns -1.
 * \p errno is set to \c ENOSYS when it is not possible to bind the requested kind of object
 * processes/threads. errno is set to \c EXDEV when the requested cpuset
 * can not be enforced (e.g. some systems only allow one CPU, and some
 * other systems only allow one NUMA node).
 *
 * If ::HWLOC_CPUBIND_STRICT was not passed, the function may fail as well,
 * or the operating system may use a slightly different operation
 * (with side-effects, smaller binding set, etc.)
 * when the requested operation is not exactly supported.
 *
 * The most portable version that should be preferred over the others,
 * whenever possible, is the following one which just binds the current program,
 * assuming it is single-threaded:
 *
 * \code
 * hwloc_set_cpubind(topology, set, 0),
 * \endcode
 *
 * If the program may be multithreaded, the following one should be preferred
 * to only bind the current thread:
 *
 * \code
 * hwloc_set_cpubind(topology, set, HWLOC_CPUBIND_THREAD),
 * \endcode
 *
 * \sa Some example codes are available under doc/examples/ in the source tree.
 *
 * \note To unbind, just call the binding function with either a full cpuset or
 * a cpuset equal to the system cpuset.
 *
 * \note On some operating systems, CPU binding may have effects on memory binding, see
 * ::HWLOC_CPUBIND_NOMEMBIND
 *
 * \note Running lstopo \--top or hwloc-ps can be a very convenient tool to check
 * how binding actually happened.
 * @{
 */

```

`hwloc_cpubind_flags_t` is a枚举类型，它定义了在`hwloc_cpubind_handle()`函数中传递给操作系统的一个标志，以控制`CPU绑定`的行为。

这个枚举类型包含三个成员：

- `HWLOC_CPUBIND_STRICT`：启用严格绑定，意味着线程/进程将永远不能在任何非指定CPU上运行，即使它们正在执行其他任务并且其他CPU处于空闲状态。
- `HWLOC_CPUBIND_NOMEMBIND`：禁用内存绑定，这意味着操作系统函数在获取CPU绑定信息时不会报告任何问题，即使它们也设置了内存绑定。
- `HWLOC_CPUBIND_DEFAULT`：禁用默認激活，这意味着操作系统不会在默认情况下设置CPU绑定。

注意：

- 當操作系统不支持`HWLOC_CPUBIND_STRICT`時，`HWLOC_CPUBIND_NOMEMBIND`和`HWLOC_CPUBIND_DEFAULT`必須被明确设置，否则函数的行为不可预测。
- `HWLOC_CPUBIND_NOMEMBIND`必须在设置`HWLOC_CPUBIND_STRICT`之前设置，否则操作系统将无法识别物理内存中的位置，从而导致错误的结果。



```cpp
/** \brief Process/Thread binding flags.
 *
 * These bit flags can be used to refine the binding policy.
 *
 * The default (0) is to bind the current process, assumed to be
 * single-threaded, in a non-strict way.  This is the most portable
 * way to bind as all operating systems usually provide it.
 *
 * \note Not all systems support all kinds of binding.  See the
 * "Detailed Description" section of \ref hwlocality_cpubinding for a
 * description of errors that can occur.
 */
typedef enum {
  /** \brief Bind all threads of the current (possibly) multithreaded process.
   * \hideinitializer */
  HWLOC_CPUBIND_PROCESS = (1<<0),

  /** \brief Bind current thread of current process.
   * \hideinitializer */
  HWLOC_CPUBIND_THREAD = (1<<1),

  /** \brief Request for strict binding from the OS.
   *
   * By default, when the designated CPUs are all busy while other
   * CPUs are idle, operating systems may execute the thread/process
   * on those other CPUs instead of the designated CPUs, to let them
   * progress anyway.  Strict binding means that the thread/process
   * will _never_ execute on other CPUs than the designated CPUs, even
   * when those are busy with other tasks and other CPUs are idle.
   *
   * \note Depending on the operating system, strict binding may not
   * be possible (e.g., the OS does not implement it) or not allowed
   * (e.g., for an administrative reasons), and the function will fail
   * in that case.
   *
   * When retrieving the binding of a process, this flag checks
   * whether all its threads  actually have the same binding. If the
   * flag is not given, the binding of each thread will be
   * accumulated.
   *
   * \note This flag is meaningless when retrieving the binding of a
   * thread.
   * \hideinitializer
   */
  HWLOC_CPUBIND_STRICT = (1<<2),

  /** \brief Avoid any effect on memory binding
   *
   * On some operating systems, some CPU binding function would also
   * bind the memory on the corresponding NUMA node.  It is often not
   * a problem for the application, but if it is, setting this flag
   * will make hwloc avoid using OS functions that would also bind
   * memory.  This will however reduce the support of CPU bindings,
   * i.e. potentially return -1 with errno set to ENOSYS in some
   * cases.
   *
   * This flag is only meaningful when used with functions that set
   * the CPU binding.  It is ignored when used with functions that get
   * CPU binding information.
   * \hideinitializer
   */
  HWLOC_CPUBIND_NOMEMBIND = (1<<3)
} hwloc_cpubind_flags_t;

```

这两函数定义在 `hwloc_set_cpubind.h` 文件中。它们用于在物理上绑定当前进程或线程到指定的CPU上。

函数 `hwloc_set_cpubind` 接受一个 `hwloc_topology_t` 类型的上下文信息和一个 `hwloc_const_cpuset_t` 类型的设置，并返回一个整数。如果设置的CPU集合与当前系统中的CPU集合不匹配，或者设置其他标志不被支持，则返回 `-1`，并设置 `errno` 为 `ENOSYS`。

函数 `hwloc_get_cpubind` 同样接受一个 `hwloc_topology_t` 类型的上下文信息和一个 `hwloc_cpuset_t` 类型的设置，并返回一个整数。如果设置的CPU集合与当前系统中的CPU集合不匹配，或者设置其他标志不被支持，则返回 `-1`，并设置 `errno` 为 `EXDEV`。


```cpp
/** \brief Bind current process or thread on CPUs given in physical bitmap \p set.
 *
 * \return -1 with errno set to ENOSYS if the action is not supported
 * \return -1 with errno set to EXDEV if the binding cannot be enforced
 */
HWLOC_DECLSPEC int hwloc_set_cpubind(hwloc_topology_t topology, hwloc_const_cpuset_t set, int flags);

/** \brief Get current process or thread binding.
 *
 * The CPU-set \p set (previously allocated by the caller)
 * is filled with the list of PUs which the process or
 * thread (according to \e flags) was last bound to.
 */
HWLOC_DECLSPEC int hwloc_get_cpubind(hwloc_topology_t topology, hwloc_cpuset_t set, int flags);

```

这段代码定义了一个名为`hwloc_set_proc_cpubind`的函数，用于在给定的topology下设置进程的物理绑定（CPU集合）。该函数的第一个参数是一个指向topology的hwloc_topology_t类型的整数，第二个参数是一个hwloc_pid_t类型的整数，表示要绑定的进程ID，第三个参数是一个hwloc_const_cpuset_t类型的整数，表示要绑定的CPU集合，最后一个是用于指定flags的整数。

在函数实现中，首先检查在Linux系统上，如果没有提供pid参数，则使用提供的时间戳（tid）代替。如果使用了timestamp，则不会在non-Linux系统上使用HWLOC_CPUBIND_THREAD这个标志。

在函数实现的最后，函数返回了一个hwloc_金黄铜指针（hwloc_topology_t *）类型的物理绑定列表，其中包含绑定的所有CPU集合。


```cpp
/** \brief Bind a process \p pid on CPUs given in physical bitmap \p set.
 *
 * \note \p hwloc_pid_t is \p pid_t on Unix platforms,
 * and \p HANDLE on native Windows platforms.
 *
 * \note As a special case on Linux, if a tid (thread ID) is supplied
 * instead of a pid (process ID) and ::HWLOC_CPUBIND_THREAD is passed in flags,
 * the binding is applied to that specific thread.
 *
 * \note On non-Linux systems, ::HWLOC_CPUBIND_THREAD can not be used in \p flags.
 */
HWLOC_DECLSPEC int hwloc_set_proc_cpubind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_cpuset_t set, int flags);

/** \brief Get the current physical binding of process \p pid.
 *
 * The CPU-set \p set (previously allocated by the caller)
 * is filled with the list of PUs which the process
 * was last bound to.
 *
 * \note \p hwloc_pid_t is \p pid_t on Unix platforms,
 * and \p HANDLE on native Windows platforms.
 *
 * \note As a special case on Linux, if a tid (thread ID) is supplied
 * instead of a pid (process ID) and HWLOC_CPUBIND_THREAD is passed in flags,
 * the binding for that specific thread is returned.
 *
 * \note On non-Linux systems, HWLOC_CPUBIND_THREAD can not be used in \p flags.
 */
```

这段代码定义了一个名为 `hwloc_get_proc_cpubind` 的函数，属于 `hwloc_topology_t` 和 `hwloc_pid_t` 类型的参数。

该函数的作用是获取在给定的 topology 和 PID 下，属于给定 CPU 集合的当前物理绑定的线程。通过调用 `hwloc_set_thread_cpubind` 函数，可以设置线程的物理绑定，然后使用 `hwloc_get_proc_cpubind` 函数获取当前的物理绑定。


```cpp
HWLOC_DECLSPEC int hwloc_get_proc_cpubind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_cpuset_t set, int flags);

#ifdef hwloc_thread_t
/** \brief Bind a thread \p thread on CPUs given in physical bitmap \p set.
 *
 * \note \p hwloc_thread_t is \p pthread_t on Unix platforms,
 * and \p HANDLE on native Windows platforms.
 *
 * \note ::HWLOC_CPUBIND_PROCESS can not be used in \p flags.
 */
HWLOC_DECLSPEC int hwloc_set_thread_cpubind(hwloc_topology_t topology, hwloc_thread_t thread, hwloc_const_cpuset_t set, int flags);
#endif

#ifdef hwloc_thread_t
/** \brief Get the current physical binding of thread \p tid.
 *
 * The CPU-set \p set (previously allocated by the caller)
 * is filled with the list of PUs which the thread
 * was last bound to.
 *
 * \note \p hwloc_thread_t is \p pthread_t on Unix platforms,
 * and \p HANDLE on native Windows platforms.
 *
 * \note ::HWLOC_CPUBIND_PROCESS can not be used in \p flags.
 */
```

这段代码定义了一个名为 `hwloc_get_thread_cpubind` 的函数，属于 `hwloc_topology_t` 和 `hwloc_thread_t` 两个结构体，其作用是获取当前进程或线程在哪个物理 CPU 上。首先通过 `hwloc_topology_t` 获取当前系统的 topology，然后通过 `hwloc_thread_t` 获取当前进程或线程，最后通过设置 CPU-set `set` 以及传递给函数的几个参数（flags），获取当前进程或线程在哪个物理 CPU 上。函数的实现中，通过 `hwloc_cpuset_t` 类型的 set 存储了当前进程或线程运行时可能用到的所有物理 CPU，然后通过 `hwloc_topology_t` 结构体中的 `bind_proc` 成员判断是否需要获取整个系统的 CPU，再将结果返回。


```cpp
HWLOC_DECLSPEC int hwloc_get_thread_cpubind(hwloc_topology_t topology, hwloc_thread_t thread, hwloc_cpuset_t set, int flags);
#endif

/** \brief Get the last physical CPU where the current process or thread ran.
 *
 * The CPU-set \p set (previously allocated by the caller)
 * is filled with the list of PUs which the process or
 * thread (according to \e flags) last ran on.
 *
 * The operating system may move some tasks from one processor
 * to another at any time according to their binding,
 * so this function may return something that is already
 * outdated.
 *
 * \p flags can include either ::HWLOC_CPUBIND_PROCESS or ::HWLOC_CPUBIND_THREAD to
 * specify whether the query should be for the whole process (union of all CPUs
 * on which all threads are running), or only the current thread. If the
 * process is single-threaded, flags can be set to zero to let hwloc use
 * whichever method is available on the underlying OS.
 */
```

这段代码定义了一个名为 `hwloc_get_last_cpu_location` 的函数，属于 `hwloc_topology_t` 和 `hwloc_cpuset_t` 类的成员。这个函数接受两个参数，一个是 `topology` 是一个 `hwloc_topology_t` 类型的变量，另一个是 `set` 是一个 `hwloc_cpuset_t` 类型的变量，这两个参数用于返回一个整数，表示在给定的 `topology` 和 `set` 作用域内运行的物理 CPU 的位置。

函数的实现中，首先通过 `hwloc_topology_get_output_device` 函数获取出给定的 `topology` 作用域内的所有输出设备，并把它们存储在一个 `hwloc_device_t` 类型的变量 `devices` 中。然后，对 `set` 中的 CPU 进行遍历，检查当前遍历的 CPU 是否在之前已经返回过。如果是，则直接返回对应的 `hwloc_cpu_bind` 函数的返回值，即上一次返回的 CPU 的 ID。如果当前遍历的 CPU 没有被之前返回过，那么函数会将 `hwloc_device_t` 类型的变量 `device` 设置为当前遍历的 CPU 的设备，并将 `flags` 参数设置为 0，传递给 `hwloc_cpu_bind` 函数，函数返回这个新创建的 CPU 的 ID。

函数的说明中还提到了两个特殊情况。第一个是在 Linux 上，如果传递给函数的 `pid` 参数比 `hwloc_pid_t` 更小，则函数会尝试使用 `::HWLOC_CPUBIND_THREAD` 作为参数，尝试通过获取特定线程的 last CPU location 来返回结果。第二个是在非 Linux 系统上，如果传递给函数的 `::HWLOC_CPUBIND_THREAD` 参数不能使用，则需要手动通过 `hwloc_device_get_t Belt` 函数获取线程绑定的物理设备，然后手动通过 `hwloc_cpu_bind` 函数获取 last CPU location。


```cpp
HWLOC_DECLSPEC int hwloc_get_last_cpu_location(hwloc_topology_t topology, hwloc_cpuset_t set, int flags);

/** \brief Get the last physical CPU where a process ran.
 *
 * The CPU-set \p set (previously allocated by the caller)
 * is filled with the list of PUs which the process
 * last ran on.
 *
 * The operating system may move some tasks from one processor
 * to another at any time according to their binding,
 * so this function may return something that is already
 * outdated.
 *
 * \note \p hwloc_pid_t is \p pid_t on Unix platforms,
 * and \p HANDLE on native Windows platforms.
 *
 * \note As a special case on Linux, if a tid (thread ID) is supplied
 * instead of a pid (process ID) and ::HWLOC_CPUBIND_THREAD is passed in flags,
 * the last CPU location of that specific thread is returned.
 *
 * \note On non-Linux systems, ::HWLOC_CPUBIND_THREAD can not be used in \p flags.
 */
```

hwloc_alloc_membind_policy() function alternative

If the requested memory binding is not supported by the operating system or cannot be enforced due to the limitations of the memory binding policy, the function may fail or have unexpected side effects.

An alternative approach to achieve similar functionality is to use hwloc_cpuset_to_nodeset() or hwloc_cpuset_from_nodeset() to convert the current CPU set to/from an NUMA memory node set. This approach is generally more portable and volatile-free than trying to force the requested memory binding through memory binding.

If the hwlocality_object_sets() function or hwlocality_bitmap() function is used to create the bitmap argument, it is important to ensure that the bitmap includes all the nodes that the CPU set should bind to, as binding to a smaller subset of nodes may cause unexpected behavior.

It is also important to note that some operating systems may have different memory binding mechanisms for CPU-less NUMA memory nodes, and hwloc_cpuset_from_nodeset() or hwloc_cpuset_to_nodeset() may need to be used in certain situations.

Overall, while hwloc_alloc_membind_policy() can be a useful function for achieving memory binding in NUMA systems, it is important to consider the limitations and possible failure scenarios when using this function, and to carefully consider the best approach for achieving the desired memory binding in a given system.


```cpp
HWLOC_DECLSPEC int hwloc_get_proc_last_cpu_location(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_cpuset_t set, int flags);

/** @} */



/** \defgroup hwlocality_membinding Memory binding
 *
 * Memory binding can be done three ways:
 *
 * - explicit memory allocation thanks to hwloc_alloc_membind() and friends:
 *   the binding will have effect on the memory allocated by these functions.
 * - implicit memory binding through binding policy: hwloc_set_membind() and
 *   friends only define the current policy of the process, which will be
 *   applied to the subsequent calls to malloc() and friends.
 * - migration of existing memory ranges, thanks to hwloc_set_area_membind()
 *   and friends, which move already-allocated data.
 *
 * Not all operating systems support all three ways.
 * hwloc_topology_get_support() may be used to query about the actual memory
 * binding support in the currently used operating system.
 *
 * When the requested binding operation is not available and the
 * ::HWLOC_MEMBIND_STRICT flag was passed, the function returns -1.
 * \p errno will be set to \c ENOSYS when the system does support
 * the specified action or policy
 * (e.g., some systems only allow binding memory on a per-thread
 * basis, whereas other systems only allow binding memory for all
 * threads in a process).
 * \p errno will be set to EXDEV when the requested set can not be enforced
 * (e.g., some systems only allow binding memory to a single NUMA node).
 *
 * If ::HWLOC_MEMBIND_STRICT was not passed, the function may fail as well,
 * or the operating system may use a slightly different operation
 * (with side-effects, smaller binding set, etc.)
 * when the requested operation is not exactly supported.
 *
 * The most portable form that should be preferred over the others
 * whenever possible is as follows.
 * It allocates some memory hopefully bound to the specified set.
 * To do so, hwloc will possibly have to change the current memory
 * binding policy in order to actually get the memory bound, if the OS
 * does not provide any other way to simply allocate bound memory
 * without changing the policy for all allocations. That is the
 * difference with hwloc_alloc_membind(), which will never change the
 * current memory binding policy.
 *
 * \code
 * hwloc_alloc_membind_policy(topology, size, set,
 *                            HWLOC_MEMBIND_BIND, 0);
 * \endcode
 *
 * Each hwloc memory binding function takes a bitmap argument that
 * is a CPU set by default, or a NUMA memory node set if the flag
 * ::HWLOC_MEMBIND_BYNODESET is specified.
 * See \ref hwlocality_object_sets and \ref hwlocality_bitmap for a
 * discussion of CPU sets and NUMA memory node sets.
 * It is also possible to convert between CPU set and node set using
 * hwloc_cpuset_to_nodeset() or hwloc_cpuset_from_nodeset().
 *
 * Memory binding by CPU set cannot work for CPU-less NUMA memory nodes.
 * Binding by nodeset should therefore be preferred whenever possible.
 *
 * \sa Some example codes are available under doc/examples/ in the source tree.
 *
 * \note On some operating systems, memory binding affects the CPU
 * binding; see ::HWLOC_MEMBIND_NOCPUBIND
 * @{
 */

```

The `hwloc_topology_get_topology_nodeset()` function is used to retrieve the nodeset of a system. This nodeset can then be used by a thread to access memory. The thread may choose to touch any node in the system, as per the `hwloc_membind_policy_t` structure passed to `hwloc_membind_first()` and `hwloc_membind_bind()` functions.

The `HWLOC_MEMBIND_FIRSTTOUCH` constant is set to 1 to indicate that the touch thread should run first. If the touch thread is the first to touch a node, it will allocate memory on that node.

If the nodeset is smaller, the local node can be allocated memory on if it is in the nodeset, or it can be allocated memory from a random non-local node. If the local node is not in the nodeset, the thread will still be able to access the memory through the allocated interface.

The `HWLOC_MEMBIND_BIND` function is used to bind memory to the specified nodes.

The `HWLOC_MEMBIND_INTERLEAVE` function is used to interleave memory across multiple NUMA nodes. This can be useful when all threads distributed across the nodes will be accessing the same memory range concurrently.

The `HWLOC_MEMBIND_NEXTTOUCH` function is used to move a page to the local NUMA node of the touch thread if it needs to be moved.

The `HWLOC_MEMBIND_MIXED` constant is used to return when multiple threads or parts of a memory area have differing memory binding policies.


```cpp
/** \brief Memory binding policy.
 *
 * These constants can be used to choose the binding policy.  Only one policy can
 * be used at a time (i.e., the values cannot be OR'ed together).
 *
 * Not all systems support all kinds of binding.
 * hwloc_topology_get_support() may be used to query about the actual memory
 * binding policy support in the currently used operating system.
 * See the "Detailed Description" section of \ref hwlocality_membinding
 * for a description of errors that can occur.
 */
typedef enum {
  /** \brief Reset the memory allocation policy to the system default.
   * Depending on the operating system, this may correspond to
   * ::HWLOC_MEMBIND_FIRSTTOUCH (Linux, FreeBSD),
   * or ::HWLOC_MEMBIND_BIND (AIX, HP-UX, Solaris, Windows).
   * This policy is never returned by get membind functions.
   * The nodeset argument is ignored.
   * \hideinitializer */
  HWLOC_MEMBIND_DEFAULT =	0,

  /** \brief Allocate each memory page individually on the local NUMA
   * node of the thread that touches it.
   *
   * The given nodeset should usually be hwloc_topology_get_topology_nodeset()
   * so that the touching thread may run and allocate on any node in the system.
   *
   * On AIX, if the nodeset is smaller, pages are allocated locally (if the local
   * node is in the nodeset) or from a random non-local node (otherwise).
   * \hideinitializer */
  HWLOC_MEMBIND_FIRSTTOUCH =	1,

  /** \brief Allocate memory on the specified nodes.
   * \hideinitializer */
  HWLOC_MEMBIND_BIND =		2,

  /** \brief Allocate memory on the given nodes in an interleaved
   * / round-robin manner.  The precise layout of the memory across
   * multiple NUMA nodes is OS/system specific. Interleaving can be
   * useful when threads distributed across the specified NUMA nodes
   * will all be accessing the whole memory range concurrently, since
   * the interleave will then balance the memory references.
   * \hideinitializer */
  HWLOC_MEMBIND_INTERLEAVE =	3,

  /** \brief For each page bound with this policy, by next time
   * it is touched (and next time only), it is moved from its current
   * location to the local NUMA node of the thread where the memory
   * reference occurred (if it needs to be moved at all).
   * \hideinitializer */
  HWLOC_MEMBIND_NEXTTOUCH =	4,

  /** \brief Returned by get_membind() functions when multiple
   * threads or parts of a memory area have differing memory binding
   * policies.
   * Also returned when binding is unknown because binding hooks are empty
   * when the topology is loaded from XML without HWLOC_THISSYSTEM=1, etc.
   * \hideinitializer */
  HWLOC_MEMBIND_MIXED = -1
} hwloc_membind_policy_t;

```

`hwloc_membind_flags_t` is a union type that represents the different flags that can be passed to the `hwloc_membind()` function. These flags are intended to be set by applications and are used to control the behavior of the `hwloc_membind_()` function.

The possible values for the `hwloc_membind_flags_t` union type are:

* `1`: The flag `HWLOC_MEMBIND_PROCESS` is set, meaning that the binding process should be performed by the application.
* `2`: The flag `HWLOC_MEMBIND_STRICT` is set, meaning that the binding process should be performed by the application, and strict binding should be requested by the operating system.
* `3`: The flag `HWLOC_MEMBIND_MIGRATE` is set, meaning that memory migration should be performed by the application.
* `4`: The flag `HWLOC_MEMBIND_NOCPUBIND` is set, meaning that the application should not bind to the CPU, and should instead use the bitmap argument as a nodeset.
* `5`: The flag `HWLOC_MEMBIND_BYNODESET` is set, meaning that the application should perform memory binding by nodeset, and not by CPU set.

These flags can be combined to provide additional control over the memory binding behavior of the `hwloc_membind_()` function.


```cpp
/** \brief Memory binding flags.
 *
 * These flags can be used to refine the binding policy.
 * All flags can be logically OR'ed together with the exception of
 * ::HWLOC_MEMBIND_PROCESS and ::HWLOC_MEMBIND_THREAD;
 * these two flags are mutually exclusive.
 *
 * Not all systems support all kinds of binding.
 * hwloc_topology_get_support() may be used to query about the actual memory
 * binding support in the currently used operating system.
 * See the "Detailed Description" section of \ref hwlocality_membinding
 * for a description of errors that can occur.
 */
typedef enum {
  /** \brief Set policy for all threads of the specified (possibly
   * multithreaded) process.  This flag is mutually exclusive with
   * ::HWLOC_MEMBIND_THREAD.
   * \hideinitializer */
  HWLOC_MEMBIND_PROCESS =       (1<<0),

 /** \brief Set policy for a specific thread of the current process.
  * This flag is mutually exclusive with ::HWLOC_MEMBIND_PROCESS.
  * \hideinitializer */
  HWLOC_MEMBIND_THREAD =        (1<<1),

 /** Request strict binding from the OS.  The function will fail if
  * the binding can not be guaranteed / completely enforced.
  *
  * This flag has slightly different meanings depending on which
  * function it is used with.
  * \hideinitializer  */
  HWLOC_MEMBIND_STRICT =        (1<<2),

 /** \brief Migrate existing allocated memory.  If the memory cannot
  * be migrated and the ::HWLOC_MEMBIND_STRICT flag is passed, an error
  * will be returned.
  * \hideinitializer  */
  HWLOC_MEMBIND_MIGRATE =       (1<<3),

  /** \brief Avoid any effect on CPU binding.
   *
   * On some operating systems, some underlying memory binding
   * functions also bind the application to the corresponding CPU(s).
   * Using this flag will cause hwloc to avoid using OS functions that
   * could potentially affect CPU bindings.  Note, however, that using
   * NOCPUBIND may reduce hwloc's overall memory binding
   * support. Specifically: some of hwloc's memory binding functions
   * may fail with errno set to ENOSYS when used with NOCPUBIND.
   * \hideinitializer
   */
  HWLOC_MEMBIND_NOCPUBIND =     (1<<4),

  /** \brief Consider the bitmap argument as a nodeset.
   *
   * The bitmap argument is considered a nodeset if this flag is given,
   * or a cpuset otherwise by default.
   *
   * Memory binding by CPU set cannot work for CPU-less NUMA memory nodes.
   * Binding by nodeset should therefore be preferred whenever possible.
   * \hideinitializer
   */
  HWLOC_MEMBIND_BYNODESET =     (1<<5)
} hwloc_membind_flags_t;

```

这段代码定义了一个默认的内存绑定策略，用于指定当前进程或线程如何设置其内存绑定。它允许用户指定一个或多个目标NUMA节点。如果没有指定`::HWLOC_MEMBIND_PROCESS`或`::HWLOC_MEMBIND_THREAD`，则默认情况下当前进程或线程被认为是单线程的。

如果指定了`::HWLOC_MEMBIND_BYNODESET`，则它被视为一个节点集，否则它被视为一个节点集合。这个策略的最通用的形式是允许硬件内存布局（hwloc）根据操作系统功能可用性来设置内存绑定。

如果这个操作不能被支持，则会输出errno并设置为ENOSYS。如果不能强制执行内存绑定，则会输出errno并设置为EXDEV。


```cpp
/** \brief Set the default memory binding policy of the current
 * process or thread to prefer the NUMA node(s) specified by \p set
 *
 * If neither ::HWLOC_MEMBIND_PROCESS nor ::HWLOC_MEMBIND_THREAD is
 * specified, the current process is assumed to be single-threaded.
 * This is the most portable form as it permits hwloc to use either
 * process-based OS functions or thread-based OS functions, depending
 * on which are available.
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified, set is considered a nodeset.
 * Otherwise it's a cpuset.
 *
 * \return -1 with errno set to ENOSYS if the action is not supported
 * \return -1 with errno set to EXDEV if the binding cannot be enforced
 */
```

This is a function definition for the hwloc\_bind() function in the hwloc library.

It specifies the behavior of the function when the specified binding flag is passed.

If the binding flag is neither `::HWLOC_MEMBIND_PROCESS` nor `::HWLOC_MEMBIND_THREAD`, the function defaults to single-threaded and hwloc will use the default memory policies for all threads in the process.

If the binding flag is `::HWLOC_MEMBIND_PROCESS`, the function will check the default memory policies and nodesets for all threads in the process. If they are not identical, -1 is returned and errno is set to `EXDEV`.

If the binding flag is `::HWLOC_MEMBIND_THREAD`, the function will only be responsible for the memory management of the current thread and will return -1 if any.

If the `::HWLOC_MEMBIND_STRICT` flag is specified, hwloc will check the default memory policies and nodesets for all threads in the process. If they are not identical, -1 is returned and errno is set to `EINVAL`.

If any other flags are specified, -1 is returned and errno is set to `EINVAL`.


```cpp
HWLOC_DECLSPEC int hwloc_set_membind(hwloc_topology_t topology, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags);

/** \brief Query the default memory binding policy and physical locality of the
 * current process or thread.
 *
 * The bitmap \p set (previously allocated by the caller)
 * is filled with the process or thread memory binding.
 *
 * This function has two output parameters: \p set and \p policy.
 * The values returned in these parameters depend on both the \p flags
 * passed in and the current memory binding policies and nodesets in
 * the queried target.
 *
 * Passing the ::HWLOC_MEMBIND_PROCESS flag specifies that the query
 * target is the current policies and nodesets for all the threads in
 * the current process.  Passing ::HWLOC_MEMBIND_THREAD specifies that
 * the query target is the current policy and nodeset for only the
 * thread invoking this function.
 *
 * If neither of these flags are passed (which is the most portable
 * method), the process is assumed to be single threaded.  This allows
 * hwloc to use either process-based OS functions or thread-based OS
 * functions, depending on which are available.
 *
 * ::HWLOC_MEMBIND_STRICT is only meaningful when ::HWLOC_MEMBIND_PROCESS
 * is also specified.  In this case, hwloc will check the default
 * memory policies and nodesets for all threads in the process.  If
 * they are not identical, -1 is returned and errno is set to EXDEV.
 * If they are identical, the values are returned in \p set and \p
 * policy.
 *
 * Otherwise, if ::HWLOC_MEMBIND_PROCESS is specified (and
 * ::HWLOC_MEMBIND_STRICT is \em not specified), the default set
 * from each thread is logically OR'ed together.
 * If all threads' default policies are the same, \p policy is set to
 * that policy.  If they are different, \p policy is set to
 * ::HWLOC_MEMBIND_MIXED.
 *
 * In the ::HWLOC_MEMBIND_THREAD case (or when neither
 * ::HWLOC_MEMBIND_PROCESS or ::HWLOC_MEMBIND_THREAD is specified), there
 * is only one set and policy; they are returned in \p set and
 * \p policy, respectively.
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified, set is considered a nodeset.
 * Otherwise it's a cpuset.
 *
 * If any other flags are specified, -1 is returned and errno is set
 * to EINVAL.
 */
```

这两段代码定义了 `hwloc_get_membind` 和 `hwloc_set_proc_membind` 函数，它们属于 `hwloc_membind_policy_t` 类型。

`hwloc_get_membind` 函数的参数包括：

* `topology`：表示硬件布局的 topology 类型，如 support_level、proximity 或其他支持 NUMA 节点的布局。
* `set`：表示要设置的内存绑定集合，可以是节点集（`hwloc_bitmap_t` 类型）或节点集的掩码（`hwloc_bitmap_t` 类型）。
* `policy`：指向 `hwloc_membind_policy_t` 类型的指针，用于设置默认的内存绑定策略。这个策略是一个指针，指向一个 `hwloc_membind_policy_t` 类型的常量，这个常量可以是以下几个值：
	+ `MEMBIND_NODE_SET`：表示为指定的节点设置内存绑定。
	+ `MEMBIND_CPUSET_SET`：表示将内存绑定应用于指定的 CPU 集合。
	+ `MEMBIND_NODE_SET` 和 `MEMBIND_CPUSET_SET`：表示为指定的节点设置内存绑定，同时将指定的 CPU 集合应用于内存绑定。
* `flags`：设置内存绑定的标志，通常用于操作系统特定的需求。

`hwloc_set_proc_membind` 函数的参数包括：

* `topology`：与 `hwloc_get_membind` 函数的相同参数。
* `pid`：要设置内存绑定的进程 ID。
* `set`：表示要设置的内存绑定集合，可以是节点集（`hwloc_bitmap_t` 类型）或节点集的掩码（`hwloc_bitmap_t` 类型）。
* `policy`：指向 `hwloc_membind_policy_t` 类型的指针，用于设置默认的内存绑定策略。这个策略是一个指针，指向一个 `hwloc_membind_policy_t` 类型的常量，这个常量可以是以下几个值：
	+ `MEMBIND_NODE_SET`：表示为指定的节点设置内存绑定。
	+ `MEMBIND_CPUSET_SET`：表示将内存绑定应用于指定的 CPU 集合。
	+ `MEMBIND_NODE_SET` 和 `MEMBIND_CPUSET_SET`：表示为指定的节点设置内存绑定，同时将指定的 CPU 集合应用于内存绑定。
* `flags`：设置内存绑定的标志，通常用于操作系统特定的需求。

这两个函数的作用是：

* `hwloc_get_membind` 函数返回一个设置默认内存绑定策略的错误码，如果设置策略不支持指定的操作，则会输出错误码。
* `hwloc_set_proc_membind` 函数返回一个设置指定进程的内存绑定策略的错误码。如果设置策略不支持指定的操作，则会输出错误码。


```cpp
HWLOC_DECLSPEC int hwloc_get_membind(hwloc_topology_t topology, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags);

/** \brief Set the default memory binding policy of the specified
 * process to prefer the NUMA node(s) specified by \p set
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified, set is considered a nodeset.
 * Otherwise it's a cpuset.
 *
 * \return -1 with errno set to ENOSYS if the action is not supported
 * \return -1 with errno set to EXDEV if the binding cannot be enforced
 *
 * \note \p hwloc_pid_t is \p pid_t on Unix platforms,
 * and \p HANDLE on native Windows platforms.
 */
HWLOC_DECLSPEC int hwloc_set_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags);

```

This is a function that gets the current memory binding and returns it in two forms: `hwloc_memory_bindings()` and `hwloc_policy_path()`.

The function takes two arguments, `p_flags` and `p_policy`, which are both passed in by the caller. The `p_flags` argument is a bitmask that specifies the memory binding flags to use. The current implementation uses the following bitmask:

* If `::HWLOC_MEMBIND_PROCESS` is specified, the query target is the current policies and nodesets for all the threads in the specified process.
* If `::HWLOC_MEMBIND_STRICT` is specified, hwloc will check the default memory policies and nodesets for all threads in the specified process.
* If `::HWLOC_MEMBIND_THREAD` is specified, hwloc will assume that the query target is a single-threaded process and return an error.

The function returns a `hwloc_memory_bindings()` and a `hwloc_policy_path()` in both the cases where `::HWLOC_MEMBIND_PROCESS` is specified or `::HWLOC_MEMBIND_STRICT` is specified. If `::HWLOC_MEMBIND_THREAD` is specified, the function will return an error.

If `::HWLOC_MEMBIND_PROCESS` is specified, the function will use the process memory binding, and if `::HWLOC_MEMBIND_STRICT` is specified, the function will use the strict memory binding.

If `::HWLOC_MEMBIND_THREAD` is specified, the function will use the default memory binding for all threads in the current process.

If any other flags are specified, the function will return an error with the appropriate error code.


```cpp
/** \brief Query the default memory binding policy and physical locality of the
 * specified process.
 *
 * The bitmap \p set (previously allocated by the caller)
 * is filled with the process memory binding.
 *
 * This function has two output parameters: \p set and \p policy.
 * The values returned in these parameters depend on both the \p flags
 * passed in and the current memory binding policies and nodesets in
 * the queried target.
 *
 * Passing the ::HWLOC_MEMBIND_PROCESS flag specifies that the query
 * target is the current policies and nodesets for all the threads in
 * the specified process.  If ::HWLOC_MEMBIND_PROCESS is not specified
 * (which is the most portable method), the process is assumed to be
 * single threaded.  This allows hwloc to use either process-based OS
 * functions or thread-based OS functions, depending on which are
 * available.
 *
 * Note that it does not make sense to pass ::HWLOC_MEMBIND_THREAD to
 * this function.
 *
 * If ::HWLOC_MEMBIND_STRICT is specified, hwloc will check the default
 * memory policies and nodesets for all threads in the specified
 * process.  If they are not identical, -1 is returned and errno is
 * set to EXDEV.  If they are identical, the values are returned in \p
 * set and \p policy.
 *
 * Otherwise, \p set is set to the logical OR of all threads'
 * default set.  If all threads' default policies
 * are the same, \p policy is set to that policy.  If they are
 * different, \p policy is set to ::HWLOC_MEMBIND_MIXED.
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified, set is considered a nodeset.
 * Otherwise it's a cpuset.
 *
 * If any other flags are specified, -1 is returned and errno is set
 * to EINVAL.
 *
 * \note \p hwloc_pid_t is \p pid_t on Unix platforms,
 * and \p HANDLE on native Windows platforms.
 */
```

It appears that this is a function for setting the binding policy and mapping memory to a specific NUMA node. The function takes in information about the topology of the hardware location, the address of the memory, the length of the matching page, a set of flags indicating the memory binding policy and the nodeset to match the pages, and the memory binding policy.

If the action is not supported, the function returns -1 with errno set to EXDEV. If the binding cannot be enforced, the function also returns -1 with errno set to EXDEV.

If the hardware location is a CPU and the address is within the physical NUMA node(s), the function queries the CPUs near the physical NUMA node(s) and returns the binding policy, the set of matching pages, and the policy if specified by the `::HWLOC_MEMBIND_STRICT`, `::HWLOC_MEMBIND_MIXED`, or `::HWLOC_MEMBIND_BYNODESET` flags.


```cpp
HWLOC_DECLSPEC int hwloc_get_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags);

/** \brief Bind the already-allocated memory identified by (addr, len)
 * to the NUMA node(s) specified by \p set.
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified, set is considered a nodeset.
 * Otherwise it's a cpuset.
 *
 * \return 0 if \p len is 0.
 * \return -1 with errno set to ENOSYS if the action is not supported
 * \return -1 with errno set to EXDEV if the binding cannot be enforced
 */
HWLOC_DECLSPEC int hwloc_set_area_membind(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags);

/** \brief Query the CPUs near the physical NUMA node(s) and binding policy of
 * the memory identified by (\p addr, \p len ).
 *
 * The bitmap \p set (previously allocated by the caller)
 * is filled with the memory area binding.
 *
 * This function has two output parameters: \p set and \p policy.
 * The values returned in these parameters depend on both the \p flags
 * passed in and the memory binding policies and nodesets of the pages
 * in the address range.
 *
 * If ::HWLOC_MEMBIND_STRICT is specified, the target pages are first
 * checked to see if they all have the same memory binding policy and
 * nodeset.  If they do not, -1 is returned and errno is set to EXDEV.
 * If they are identical across all pages, the set and policy are
 * returned in \p set and \p policy, respectively.
 *
 * If ::HWLOC_MEMBIND_STRICT is not specified, the union of all NUMA
 * node(s) containing pages in the address range is calculated.
 * If all pages in the target have the same policy, it is returned in
 * \p policy.  Otherwise, \p policy is set to ::HWLOC_MEMBIND_MIXED.
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified, set is considered a nodeset.
 * Otherwise it's a cpuset.
 *
 * If any other flags are specified, -1 is returned and errno is set
 * to EINVAL.
 *
 * If \p len is 0, -1 is returned and errno is set to EINVAL.
 */
```

这段代码定义了一个名为`hwloc_get_area_membind`的函数，属于`hwloc_topology_t`类型的函数，用于根据指定的内存地址和长度，返回指定的硬件区域内存分配情况下的节点集合。

函数接受四个参数：

1. `topology`：指定的硬件区域拓扑结构，可以是`hwloc_topology_t`结构体或指针。
2. `addr`：要获取的内存地址。
3. `len`：要查找的内存长度的指针。
4. `set`：已经分配的内存区域比特map，可以是一个指向`hwloc_membind_policy_t`结构体的指针，也可以是一个指向`hwloc_topology_t`结构体的指针。如果`set`为`NULL`，则表示尚未分配内存区域。
5. `policy`：指向一个`hwloc_membind_policy_t`结构体的指针，用于在分配内存区域时执行的政策。
6. `flags`：指定使用哪些选项，如果指定，则`set`将被视为节点集，否则`set`将被视为节点集。节点集和节点集的区别在于，节点集中的每个节点都有明确的硬件位置，而节点集中只是一个节点集合。

函数首先检查`len`是否为0，如果是，函数将返回一个空节点集合。然后，函数根据`addr`和`len`计算目标内存区域在拓扑结构中的位置，并将该位置与`topology`中的节点对应。如果已分配的内存区域与计算出的位置不匹配，函数将返回一个空节点集合。

如果` flags`中指定使用`HWLOC_MEMBIND_BYNODESET`选项，则函数将`set`视为节点集，并返回一个指向节点集的指针。否则，函数将`set`视为节点集，并返回一个指向`hwloc_topology_t`结构体的指针。如果`len`为0，函数将`set`填充为`NULL`。


```cpp
HWLOC_DECLSPEC int hwloc_get_area_membind(hwloc_topology_t topology, const void *addr, size_t len, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags);

/** \brief Get the NUMA nodes where memory identified by (\p addr, \p len ) is physically allocated.
 *
 * The bitmap \p set (previously allocated by the caller)
 * is filled according to the NUMA nodes where the memory area pages
 * are physically allocated. If no page is actually allocated yet,
 * \p set may be empty.
 *
 * If pages spread to multiple nodes, it is not specified whether they spread
 * equitably, or whether most of them are on a single node, etc.
 *
 * The operating system may move memory pages from one processor
 * to another at any time according to their binding,
 * so this function may return something that is already
 * outdated.
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified in \p flags, set is
 * considered a nodeset. Otherwise it's a cpuset.
 *
 * If \p len is 0, \p set is emptied.
 */
```

这段代码定义了两个函数，分别是 `hwloc_get_area_memlocation` 和 `hwloc_alloc`。这两个函数的作用如下：

`hwloc_get_area_memlocation` 函数接收一个 `hwloc_topology_t` 类型的上下文，一个指向内存地址的指针 `addr`，以及一个指定为SET的 `size_t` 类型的内存布局设置，然后返回指定布局的物理内存位置。它使用操作系统提供的内存映射技术，将指定的物理位置映射到上下文中，并返回其全局指针。

`hwloc_alloc` 函数与 `hwloc_get_area_memlocation` 函数类似，但它在指定内存布局设置时使用了 NUMA 内存节点。它尝试从操作系统内存中分配指定长度的内存，并返回其全局指针。如果指定布局不能被强制执行，或者在尝试分配内存时出现错误，它将返回一个 `NULL` 并设置相应的错误码。


```cpp
HWLOC_DECLSPEC int hwloc_get_area_memlocation(hwloc_topology_t topology, const void *addr, size_t len, hwloc_bitmap_t set, int flags);

/** \brief Allocate some memory
 *
 * This is equivalent to malloc(), except that it tries to allocate
 * page-aligned memory from the OS.
 *
 * \note The allocated memory should be freed with hwloc_free().
 */
HWLOC_DECLSPEC void *hwloc_alloc(hwloc_topology_t topology, size_t len);

/** \brief Allocate some memory on NUMA memory nodes specified by \p set
 *
 * \return NULL with errno set to ENOSYS if the action is not supported
 * and ::HWLOC_MEMBIND_STRICT is given
 * \return NULL with errno set to EXDEV if the binding cannot be enforced
 * and ::HWLOC_MEMBIND_STRICT is given
 * \return NULL with errno set to ENOMEM if the memory allocation failed
 * even before trying to bind.
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified, set is considered a nodeset.
 * Otherwise it's a cpuset.
 *
 * \note The allocated memory should be freed with hwloc_free().
 */
```

这段代码定义了一个名为 `hwloc_alloc_membind` 的函数，它的作用是尝试在指定 NUMA 内存节点的 topology 下分配内存。这个函数接受四个参数：

1. `topology`：指定了要使用的 NUMA 内存节点的 topology，包括节点数量（具体是哪几种 NUMA 内存节点，代码中没有给出明确的定义，可以根据具体硬件环境来选择）和节点设置（可以使用 `hwloc_const_bitmap_t` 类型来指定节点设置，也可以手动指定，具体实现时可能会有所不同）。
2. `len`：指定了要分配的内存长度，单位通常是字节数或者页数等。
3. `set`：指定了要绑定的内存节点设置，可以使用 `hwloc_const_bitmap_t` 类型来指定，也可以手动指定，同样，具体实现时可能会有所不同。
4. `policy`：指定了内存绑定策略，包括按位图映射、节点映射等，具体实现时可能会有所不同。
5. `flags`：可能有其他的标志位，具体实现时可能会有所不同。

函数实现中，首先会尝试使用 `hwloc_alloc_membind()` 函数来分配内存，如果失败了，就会使用 `hwloc_set_membind()` 函数来修改当前进程或线程的内存绑定策略，然后重新分配内存。因此，这个函数可以被用来在指定 NUMA 内存节点的 topology 下分配内存，并且在失败时可以保证以后分配的内存不会使用这种 topology。


```cpp
HWLOC_DECLSPEC void *hwloc_alloc_membind(hwloc_topology_t topology, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_malloc;

/** \brief Allocate some memory on NUMA memory nodes specified by \p set
 *
 * First, try to allocate properly with hwloc_alloc_membind().
 * On failure, the current process or thread memory binding policy
 * is changed with hwloc_set_membind() before allocating memory.
 * Thus this function works in more cases, at the expense of changing
 * the current state (possibly affecting future allocations that
 * would not specify any policy).
 *
 * If ::HWLOC_MEMBIND_BYNODESET is specified, set is considered a nodeset.
 * Otherwise it's a cpuset.
 */
static __hwloc_inline void *
```



这段代码定义了两个函数：`hwloc_free()` 和 `hwloc_setsource()`。

`hwloc_free()`函数释放由`hwloc_alloc()` 或 `hwloc_alloc_membind()`分配的内存。它通过传入`topology`、`addr`和`len`参数，返回分配的内存的置灰标志。

`hwloc_setsource()`函数允许设置或取消通过`hwloc_topology_set_xml()` 或 `hwloc_topology_set_synthetic()`函数设置的源。如果这个函数没有被执行，那么它的默认设置是允许访问所有的设备。通过设置`HWLOC_XMLFILE`、`HWLOC_SYNTHETIC` 和 `HWLOC_THISYSTEM`环境变量，可以强制设置或取消设置源。


```cpp
hwloc_alloc_membind_policy(hwloc_topology_t topology, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_malloc;

/** \brief Free memory that was previously allocated by hwloc_alloc()
 * or hwloc_alloc_membind().
 */
HWLOC_DECLSPEC int hwloc_free(hwloc_topology_t topology, void *addr, size_t len);

/** @} */



/** \defgroup hwlocality_setsource Changing the Source of Topology Discovery
 *
 * If none of the functions below is called, the default is to detect all the objects
 * of the machine that the caller is allowed to access.
 *
 * This default behavior may also be modified through environment variables
 * if the application did not modify it already.
 * Setting HWLOC_XMLFILE in the environment enforces the discovery from a XML
 * file as if hwloc_topology_set_xml() had been called.
 * Setting HWLOC_SYNTHETIC enforces a synthetic topology as if
 * hwloc_topology_set_synthetic() had been called.
 *
 * Finally, HWLOC_THISSYSTEM enforces the return value of
 * hwloc_topology_is_thissystem().
 *
 * @{
 */

```

这段代码定义了一个名为`hwloc_topology_set_pid`的函数，属于`hwloc_topology_自有函数`类型。

该函数的主要作用是改变机器的topology，从默认情况下当前进程的视角，切换到另一个进程的视角。这对于在某些系统中，进程具有不同的topology（例如具有不同CPU设置的系统）非常有用。

函数的第一个参数是一个`hwloc_topology_t`类型的上下文，用于存储当前的topology。第二个参数是一个`hwloc_pid_t`类型的整数，用于指定要设置的pid。函数返回一个整数，表示设置topology后返回的errno号。-1将在不支持这个功能的目标操作系统上返回，errno号设置为ENOSYS。


```cpp
/** \brief Change which process the topology is viewed from.
 *
 * On some systems, processes may have different views of the machine, for
 * instance the set of allowed CPUs. By default, hwloc exposes the view from
 * the current process. Calling hwloc_topology_set_pid() permits to make it
 * expose the topology of the machine from the point of view of another
 * process.
 *
 * \note \p hwloc_pid_t is \p pid_t on Unix platforms,
 * and \p HANDLE on native Windows platforms.
 *
 * \note -1 is returned and errno is set to ENOSYS on platforms that do not
 * support this feature.
 */
HWLOC_DECLSPEC int hwloc_topology_set_pid(hwloc_topology_t __hwloc_restrict topology, hwloc_pid_t pid);

```

这段代码定义了一个名为`synthetic_topology_check`的函数，它的作用是检查传入的描述是否正确地描述了一个合成拓扑。如果描述正确，则返回0，否则返回-1并设置errno为EINVAL。

该函数首先通过从给定的描述中提取拓扑信息，这些信息可能包括物体类型、数据类型和拓扑层数等信息。如果这些信息正确，函数将返回0，否则返回-1并设置errno为EINVAL。

需要注意的是，尽管该函数提供了便利，但它并不会实际加载拓扑信息。要实际加载拓扑信息，需要调用`hwloc_topology_load()`函数。


```cpp
/** \brief Enable synthetic topology.
 *
 * Gather topology information from the given \p description,
 * a space-separated string of <type:number> describing
 * the object type and arity at each level.
 * All types may be omitted (space-separated string of numbers) so that
 * hwloc chooses all types according to usual topologies.
 * See also the \ref synthetic.
 *
 * Setting the environment variable HWLOC_SYNTHETIC
 * may also result in this behavior.
 *
 * If \p description was properly parsed and describes a valid topology
 * configuration, this function returns 0.
 * Otherwise -1 is returned and errno is set to EINVAL.
 *
 * Note that this function does not actually load topology
 * information; it just tells hwloc where to load it from.  You'll
 * still need to invoke hwloc_topology_load() to actually load the
 * topology information.
 *
 * \note For convenience, this backend provides empty binding hooks which just
 * return success.
 *
 * \note On success, the synthetic component replaces the previously enabled
 * component (if any), but the topology is not actually modified until
 * hwloc_topology_load().
 */
```

这段代码定义了一个名为 `hwloc_topology_set_synthetic` 的函数，属于 `hwloc_topology_t` 类型的参数 `topology` 和字符串参数 `description`。

该函数的作用是：

1. 如果 `topology` 和 `description` 参数存在，则调用 `hwloc_topology_export_xml` 函数将 XML 文件内容读取到 `topology`，然后设置环境变量 `HWLOC_XMLFILE`。
2. 如果 `topology` 和 `description` 参数不存在，则返回 `-1`，并设置 `errno` 为 `EINVAL`，表示在设置 XML 文件时出现错误。
3. 如果 `hwloc_topology_export_xml` 函数成功执行，则：
a. 如果 `topology` 存在，但是 `topology` 的 XML 文件内容中包含 `<system_service>` 标签，则执行 `hwloc_topology_load` 函数加载 XML 文件内容，并替换原来设置的组件。
b. 如果 `topology` 的 XML 文件内容中包含 `<system_component>` 标签，则表示组件已经被加载，不执行 `hwloc_topology_load` 函数。
4. 如果 `hwloc_topology_export_xml` 函数失败，则执行错误处理，并将 `errno` 设置为 `EINVAL`。


```cpp
HWLOC_DECLSPEC int hwloc_topology_set_synthetic(hwloc_topology_t __hwloc_restrict topology, const char * __hwloc_restrict description);

/** \brief Enable XML-file based topology.
 *
 * Gather topology information from the XML file given at \p xmlpath.
 * Setting the environment variable HWLOC_XMLFILE may also result in this behavior.
 * This file may have been generated earlier with hwloc_topology_export_xml() in hwloc/export.h,
 * or lstopo file.xml.
 *
 * Note that this function does not actually load topology
 * information; it just tells hwloc where to load it from.  You'll
 * still need to invoke hwloc_topology_load() to actually load the
 * topology information.
 *
 * \return -1 with errno set to EINVAL on failure to read the XML file.
 *
 * \note See also hwloc_topology_set_userdata_import_callback()
 * for importing application-specific object userdata.
 *
 * \note For convenience, this backend provides empty binding hooks which just
 * return success.  To have hwloc still actually call OS-specific hooks, the
 * ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM has to be set to assert that the loaded
 * file is really the underlying system.
 *
 * \note On success, the XML component replaces the previously enabled
 * component (if any), but the topology is not actually modified until
 * hwloc_topology_load().
 */
```

这段代码是一个用于在HWLOC中使用XML内存缓冲区设置拓扑结构的函数。它接受两个参数：hwloc_topology_t类型的topology和XML内存缓冲区的路径。

具体来说，这段代码的功能是：将给定的XML内存缓冲区中的拓扑信息读取并设置到给定的topology中。这样，用户就可以使用XML文件而不是直接从文件中读取拓扑信息。当使用这种方法时，操作系统将调用HWLOC_TOPOLOGY_SET_XMLINVALID函数来报告任何XML缓冲区错误，例如无法找到文件或错误的文件路径。

该函数的实现进一步简化了HWLOC_TOPOLOGY_SET_XMLINVALID函数的错误处理。如果尝试加载XML文件时遇到问题，将抛出EINVAL错误。调用该函数时，需要确保在调用hwloc_topology_export_xmlbuffer()函数将XML缓冲区文件输出到hwloc_topology_t类型的topology结构中，然后尝试使用上述拓扑结构。


```cpp
HWLOC_DECLSPEC int hwloc_topology_set_xml(hwloc_topology_t __hwloc_restrict topology, const char * __hwloc_restrict xmlpath);

/** \brief Enable XML based topology using a memory buffer (instead of
 * a file, as with hwloc_topology_set_xml()).
 *
 * Gather topology information from the XML memory buffer given at \p
 * buffer and of length \p size.  This buffer may have been filled
 * earlier with hwloc_topology_export_xmlbuffer() in hwloc/export.h.
 *
 * Note that this function does not actually load topology
 * information; it just tells hwloc where to load it from.  You'll
 * still need to invoke hwloc_topology_load() to actually load the
 * topology information.
 *
 * \return -1 with errno set to EINVAL on failure to read the XML buffer.
 *
 * \note See also hwloc_topology_set_userdata_import_callback()
 * for importing application-specific object userdata.
 *
 * \note For convenience, this backend provides empty binding hooks which just
 * return success.  To have hwloc still actually call OS-specific hooks, the
 * ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM has to be set to assert that the loaded
 * file is really the underlying system.
 *
 * \note On success, the XML component replaces the previously enabled
 * component (if any), but the topology is not actually modified until
 * hwloc_topology_load().
 */
```

这段代码定义了一个名为 `hwloc_topology_set_xmlbuffer` 的函数，属于 `hwloc_topology_topology` 系列的函数。该函数接受两个参数：一个 `hwloc_topology_t` 类型的整型数据 `topology`，一个指向字符数组的指针 `buffer`，以及一个表示字符串长度的整型参数 `size`。

函数的作用是设置给定的 `topology` 中的组件，通过设置 `buffer` 中存储的字符串来决定组件的显示名称，如果设置了字符串黑名单标志，则不会使用该组件。设置完组件后，该函数返回 0，表示设置成功。

该函数还定义了一个名为 `HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST` 的枚举类型，用于说明字符串黑名单标志的设置方法，包括其索引号 `1`（表示黑名单）和具体的字符串名称。


```cpp
HWLOC_DECLSPEC int hwloc_topology_set_xmlbuffer(hwloc_topology_t __hwloc_restrict topology, const char * __hwloc_restrict buffer, int size);

/** \brief Flags to be passed to hwloc_topology_set_components()
 */
enum hwloc_topology_components_flag_e {
  /** \brief Blacklist the target component from being used.
   * \hideinitializer
   */
  HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST = (1UL<<0)
};

/** \brief Prevent a discovery component from being used for a topology.
 *
 * \p name is the name of the discovery component that should not be used
 * when loading topology \p topology. The name is a string such as "cuda".
 *
 * For components with multiple phases, it may also be suffixed with the name
 * of a phase, for instance "linux:io".
 *
 * \p flags should be ::HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST.
 *
 * This may be used to avoid expensive parts of the discovery process.
 * For instance, CUDA-specific discovery may be expensive and unneeded
 * while generic I/O discovery could still be useful.
 */
```

这段代码定义了一个名为 `hwloc_topology_set_components` 的函数，属于 `hwloc_topology_t` 数据类型的函数，用于设置指定的 topology 的组件。通过这个函数，用户可以自定义 topology 的组件，例如忽略某些组件类型或者定义一个合成 topology。

函数接受三个参数：

1. `hwloc_topology_t`：这是一个用于存储 topology 信息的 `hwloc_topology_t` 数据类型变量，用于存储从 `hwloc_topology_init` 和 `hwloc_topology_load` 函数传递过来的 topology 信息。
2. `unsigned long flags`：一个无符号长整型变量，用于存储设置组件的标志。例如，如果调用 `hwloc_topology_set_components` 函数时传递了这个 `flags` 值，那么这个函数会忽略与 `flags` 相关的组件。
3. 《const char *` name`：一个字符型变量，用于存储要设置的 topology 组件的名称。如果 `name` 的值传递给了 `hwloc_topology_set_components` 函数，那么这个函数会将这个组件设置为指定的名称。

设置组件后，这个函数返回 `0`，表示成功设置组件。


```cpp
HWLOC_DECLSPEC int hwloc_topology_set_components(hwloc_topology_t __hwloc_restrict topology, unsigned long flags, const char * __hwloc_restrict name);

/** @} */



/** \defgroup hwlocality_configuration Topology Detection Configuration and Query
 *
 * Several functions can optionally be called between hwloc_topology_init() and
 * hwloc_topology_load() to configure how the detection should be performed,
 * e.g. to ignore some objects types, define a synthetic topology, etc.
 *
 * @{
 */

```

This is a list of flags for the HWLOC (Hardware Localization) tool. These flags are used to control the behavior of the tool during the discovery process.

If the `HWLOC_TOPOLOGY_FLAG_IS_THIS_SYSTEM` flag is not set, the `HWLOC_TOPOLOGY_FLAG_DON_CHANGE_BINDING` flag will be used instead. This flag disables all hardware localization discoveries that require a change of the binding, such as the process or thread binding.

If the `HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING` flag is not set, the `HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING` flag will also be used. This flag enables all hardware localization discoveries that do not require a change of the binding.

If the `HWLOC_TOPOLOGY_FLAG_NO_DISTANCES` flag is not set, the `HWLOC_TOPOLOGY_FLAG_DON_CHANGE_BINDING` flag will also be used. This flag ignores all distance information from the operating systems and from XML.

If the `HWLOC_TOPOLOGY_FLAG_NO_MEMATTRS` flag is not set, the `HWLOC_TOPOLOGY_FLAG_DON_CHANGE_BINDING` flag will also be used. This flag ignores all memory attributes information from the operating systems and from XML.

If the `HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS` flag is not set, the `HWLOC_TOPOLOGY_FLAG_DON_CHANGE_BINDING` flag will also be used. This flag ignores all CPU information from the operating systems and from XML.


```cpp
/** \brief Flags to be set onto a topology context before load.
 *
 * Flags should be given to hwloc_topology_set_flags().
 * They may also be returned by hwloc_topology_get_flags().
 */
enum hwloc_topology_flags_e {
 /** \brief Detect the whole system, ignore reservations, include disallowed objects.
   *
   * Gather all online resources, even if some were disabled by the administrator.
   * For instance, ignore Linux Cgroup/Cpusets and gather all processors and memory nodes.
   * However offline PUs and NUMA nodes are still ignored.
   *
   * When this flag is not set, PUs and NUMA nodes that are disallowed are not added to the topology.
   * Parent objects (package, core, cache, etc.) are added only if some of their children are allowed.
   * All existing PUs and NUMA nodes in the topology are allowed.
   * hwloc_topology_get_allowed_cpuset() and hwloc_topology_get_allowed_nodeset()
   * are equal to the root object cpuset and nodeset.
   *
   * When this flag is set, the actual sets of allowed PUs and NUMA nodes are given
   * by hwloc_topology_get_allowed_cpuset() and hwloc_topology_get_allowed_nodeset().
   * They may be smaller than the root object cpuset and nodeset.
   *
   * If the current topology is exported to XML and reimported later, this flag
   * should be set again in the reimported topology so that disallowed resources
   * are reimported as well.
   * \hideinitializer
   */
  HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED = (1UL<<0),

 /** \brief Assume that the selected backend provides the topology for the
   * system on which we are running.
   *
   * This forces hwloc_topology_is_thissystem() to return 1, i.e. makes hwloc assume that
   * the selected backend provides the topology for the system on which we are running,
   * even if it is not the OS-specific backend but the XML backend for instance.
   * This means making the binding functions actually call the OS-specific
   * system calls and really do binding, while the XML backend would otherwise
   * provide empty hooks just returning success.
   *
   * Setting the environment variable HWLOC_THISSYSTEM may also result in the
   * same behavior.
   *
   * This can be used for efficiency reasons to first detect the topology once,
   * save it to an XML file, and quickly reload it later through the XML
   * backend, but still having binding functions actually do bind.
   * \hideinitializer
   */
  HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM = (1UL<<1),

 /** \brief Get the set of allowed resources from the local operating system even if the topology was loaded from XML or synthetic description.
   *
   * If the topology was loaded from XML or from a synthetic string,
   * restrict it by applying the current process restrictions such as
   * Linux Cgroup/Cpuset.
   *
   * This is useful when the topology is not loaded directly from
   * the local machine (e.g. for performance reason) and it comes
   * with all resources, while the running process is restricted
   * to only parts of the machine.
   *
   * This flag is ignored unless ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM is
   * also set since the loaded topology must match the underlying machine
   * where restrictions will be gathered from.
   *
   * Setting the environment variable HWLOC_THISSYSTEM_ALLOWED_RESOURCES
   * would result in the same behavior.
   * \hideinitializer
   */
  HWLOC_TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES = (1UL<<2),

  /** \brief Import support from the imported topology.
   *
   * When importing a XML topology from a remote machine, binding is
   * disabled by default (see ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM).
   * This disabling is also marked by putting zeroes in the corresponding
   * supported feature bits reported by hwloc_topology_get_support().
   *
   * The flag ::HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT actually imports
   * support bits from the remote machine. It also sets the flag
   * \p imported_support in the struct hwloc_topology_misc_support array.
   * If the imported XML did not contain any support information
   * (exporter hwloc is too old), this flag is not set.
   *
   * Note that these supported features are only relevant for the hwloc
   * installation that actually exported the XML topology
   * (it may vary with the operating system, or with how hwloc was compiled).
   *
   * Note that setting this flag however does not enable binding for the
   * locally imported hwloc topology, it only reports what the remote
   * hwloc and machine support.
   *
   */
  HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT = (1UL<<3),

  /** \brief Do not consider resources outside of the process CPU binding.
   *
   * If the binding of the process is limited to a subset of cores,
   * ignore the other cores during discovery.
   *
   * The resulting topology is identical to what a call to hwloc_topology_restrict()
   * would generate, but this flag also prevents hwloc from ever touching other
   * resources during the discovery.
   *
   * This flag especially tells the x86 backend to never temporarily
   * rebind a thread on any excluded core. This is useful on Windows
   * because such temporary rebinding can change the process binding.
   * Another use-case is to avoid cores that would not be able to
   * perform the hwloc discovery anytime soon because they are busy
   * executing some high-priority real-time tasks.
   *
   * If process CPU binding is not supported,
   * the thread CPU binding is considered instead if supported,
   * or the flag is ignored.
   *
   * This flag requires ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM as well
   * since binding support is required.
   */
  HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING = (1UL<<4),

  /** \brief Do not consider resources outside of the process memory binding.
   *
   * If the binding of the process is limited to a subset of NUMA nodes,
   * ignore the other NUMA nodes during discovery.
   *
   * The resulting topology is identical to what a call to hwloc_topology_restrict()
   * would generate, but this flag also prevents hwloc from ever touching other
   * resources during the discovery.
   *
   * This flag is meant to be used together with
   * ::HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING when both cores
   * and NUMA nodes should be ignored outside of the process binding.
   *
   * If process memory binding is not supported,
   * the thread memory binding is considered instead if supported,
   * or the flag is ignored.
   *
   * This flag requires ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM as well
   * since binding support is required.
   */
  HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING = (1UL<<5),

  /** \brief Do not ever modify the process or thread binding during discovery.
   *
   * This flag disables all hwloc discovery steps that require a change of
   * the process or thread binding. This currently only affects the x86
   * backend which gets entirely disabled.
   *
   * This is useful when hwloc_topology_load() is called while the
   * application also creates additional threads or modifies the binding.
   *
   * This flag is also a strict way to make sure the process binding will
   * not change to due thread binding changes on Windows
   * (see ::HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING).
   */
  HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING = (1UL<<6),

  /** \brief Ignore distances.
   *
   * Ignore distance information from the operating systems (and from XML)
   * and hence do not use distances for grouping.
   */
  HWLOC_TOPOLOGY_FLAG_NO_DISTANCES = (1UL<<7),

  /** \brief Ignore memory attributes.
   *
   * Ignore memory attribues from the operating systems (and from XML).
   */
  HWLOC_TOPOLOGY_FLAG_NO_MEMATTRS = (1UL<<8),

  /** \brief Ignore CPU Kinds.
   *
   * Ignore CPU kind information from the operating systems (and from XML).
   */
  HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS = (1UL<<9)
};

```

这段代码定义了一个名为 `hwloc_topology_set_flags` 的函数，它的作用是设置一个或多个 `::hwloc_topology_flags_e` 标志到非已加载的拓扑结构中。

函数接受两个参数，一个是拓扑结构类型，另一个是设置的标志集。函数内部使用 `hwloc_topology_set_flags` 函数来设置拓扑结构中的标志，如果这个函数之前已经调用过了，那么之前的设置会被清除并替换为新的设置。

函数还提供了一个名为 `hwloc_topology_get_flags` 的函数，用于检索拓扑结构中的标志集。默认情况下，没有任何标志。

如果 `hwloc_topology_set_flags` 函数多次被调用，最近一次调用会将之前设置的标志清除并替换为新的设置，否则之前的设置不会被清除。


```cpp
/** \brief Set OR'ed flags to non-yet-loaded topology.
 *
 * Set a OR'ed set of ::hwloc_topology_flags_e onto a topology that was not yet loaded.
 *
 * If this function is called multiple times, the last invocation will erase
 * and replace the set of flags that was previously set.
 *
 * By default, no flags are set (\c 0).
 *
 * The flags set in a topology may be retrieved with hwloc_topology_get_flags().
 */
HWLOC_DECLSPEC int hwloc_topology_set_flags (hwloc_topology_t topology, unsigned long flags);

/** \brief Get OR'ed flags of a topology.
 *
 * Get the OR'ed set of ::hwloc_topology_flags_e of a topology.
 *
 * If hwloc_topology_set_flags() was not called earlier,
 * no flags are set (\c 0 is returned).
 *
 * \return the flags previously set with hwloc_topology_set_flags().
 */
```

这段代码定义了一个名为 `hwloc_topology_get_flags` 的函数，它接受一个名为 `topology` 的 `hwloc_topology_t` 类型的参数。

该函数返回一个 `unsigned long` 类型的标志，用于描述当前topology上下文是否来自当前系统。具体来说，如果topology是在系统运行时构建的，则返回1，否则返回0。

该函数还定义了一个名为 `hwloc_topology_is_thissystem` 的函数，它与 `hwloc_topology_get_flags` 函数类似，但使用了 `const` 修饰符。该函数接受一个 `hwloc_topology_t` 类型的参数，并要求该参数必须以 `__hwloc_attribute_pure` 修饰符定义。

该函数返回一个 `int` 类型的指示符，用于描述当前topology上下文是否来自当前系统。如果topology是在系统运行时构建的，则返回1，否则返回0。

该函数还定义了一个名为 `hwloc_topology_discovery_support` 的结构体，用于描述当前topology上下文中的discovery support。该结构体包含六个成员变量，分别为 `pu`、`numa`、`numa_memory`、`disallowed_pu`、`disallowed_numa` 和 `cpukind_efficiency`。这些成员变量用于描述当前topology上下文中的discovery support。


```cpp
HWLOC_DECLSPEC unsigned long hwloc_topology_get_flags (hwloc_topology_t topology);

/** \brief Does the topology context come from this system?
 *
 * \return 1 if this topology context was built using the system
 * running this program.
 * \return 0 instead (for instance if using another file-system root,
 * a XML topology file, or a synthetic topology).
 */
HWLOC_DECLSPEC int hwloc_topology_is_thissystem(hwloc_topology_t  __hwloc_restrict topology) __hwloc_attribute_pure;

/** \brief Flags describing actual discovery support for this topology. */
struct hwloc_topology_discovery_support {
  /** \brief Detecting the number of PU objects is supported. */
  unsigned char pu;
  /** \brief Detecting the number of NUMA nodes is supported. */
  unsigned char numa;
  /** \brief Detecting the amount of memory in NUMA nodes is supported. */
  unsigned char numa_memory;
  /** \brief Detecting and identifying PU objects that are not available to the current process is supported. */
  unsigned char disallowed_pu;
  /** \brief Detecting and identifying NUMA nodes that are not available to the current process is supported. */
  unsigned char disallowed_numa;
  /** \brief Detecting the efficiency of CPU kinds is supported, see \ref hwlocality_cpukinds. */
  unsigned char cpukind_efficiency;
};

```



这段代码定义了一个名为 `hwloc_topology_cpubind_support` 的结构体，用于描述当前硬件平台中 PU(进程单元)绑定支持的情况。

该结构体包含以下成员：

- `set_thisproc_cpubind`：当前进程是否支持 PU  binding。如果设置为 `1`，则表示支持当前进程的所有 PU 绑定，即使该进程的所有非连续对象都没有对应的 PU。
- `get_thisproc_cpubind`：获取当前进程是否支持 PU 绑定的值。
- `set_proc_cpubind`：进程是否支持某个特定的 PU 绑定。
- `get_proc_cpubind`：获取进程是否支持某个特定的 PU 绑定的值。
- `set_thisthread_cpubind`：当前线程是否支持 PU 绑定的值。
- `get_thisthread_cpubind`：获取当前线程是否支持 PU 绑定的值。
- `set_thread_cpubind`：某个给定线程是否支持 PU 绑定的值。
- `get_thread_cpubind`：获取某个给定线程是否支持 PU 绑定的值。
- `get_thisproc_last_cpu_location`：获取当前进程在最后一个支持整个进程绑定的 CPU 位置。
- `get_proc_last_cpu_location`：获取当前进程在最后一个支持一个进程绑定的 CPU 位置。
- `get_thisthread_last_cpu_location`：获取当前线程在最后一个支持某个给定线程绑定的 CPU 位置。

这些成员用于指定是否支持当前进程或给定线程的 PU 绑定，以及在哪些位置支持 PU 绑定。通过这些成员，用户可以确定哪些 PU 绑定是有效的，并可以获取有关支持这些绑定的信息。


```cpp
/** \brief Flags describing actual PU binding support for this topology.
 *
 * A flag may be set even if the feature isn't supported in all cases
 * (e.g. binding to random sets of non-contiguous objects).
 */
struct hwloc_topology_cpubind_support {
  /** Binding the whole current process is supported.  */
  unsigned char set_thisproc_cpubind;
  /** Getting the binding of the whole current process is supported.  */
  unsigned char get_thisproc_cpubind;
  /** Binding a whole given process is supported.  */
  unsigned char set_proc_cpubind;
  /** Getting the binding of a whole given process is supported.  */
  unsigned char get_proc_cpubind;
  /** Binding the current thread only is supported.  */
  unsigned char set_thisthread_cpubind;
  /** Getting the binding of the current thread only is supported.  */
  unsigned char get_thisthread_cpubind;
  /** Binding a given thread only is supported.  */
  unsigned char set_thread_cpubind;
  /** Getting the binding of a given thread only is supported.  */
  unsigned char get_thread_cpubind;
  /** Getting the last processors where the whole current process ran is supported */
  unsigned char get_thisproc_last_cpu_location;
  /** Getting the last processors where a whole process ran is supported */
  unsigned char get_proc_last_cpu_location;
  /** Getting the last processors where the current thread ran is supported */
  unsigned char get_thisthread_last_cpu_location;
};

```

这段代码定义了一个名为`hwloc_topology_membind_support`的结构体，用于描述硬件平台中内存绑定支持的情况。

该结构体包含了一系列与内存绑定相关的标志，包括：

* `set_thisproc_membind`：表示绑定整个当前进程；
* `get_thisproc_membind`：表示获取整个当前进程的绑定状态；
* `set_proc_membind`：表示绑定一个给定进程；
* `get_proc_membind`：表示获取一个给定进程的绑定状态；
* `set_thisthread_membind`：表示仅绑定当前线程；
* `get_thisthread_membind`：表示获取当前线程的绑定状态；
* `set_area_membind`：表示绑定一个给定内存区域；
* `get_area_membind`：表示获取一个给定内存区域的绑定状态；
* `alloc_membind`：表示分配绑定内存区域；
* `firsttouch_membind`：表示支持首次接触内存区域；
* `bind_membind`：表示支持内存绑定；
* `interleave_membind`：表示支持跨设备访问内存区域；
* `nexttouch_membind`：表示支持在给定线程首次接触内存区域之后进行内存区域操作；
* `migrate_membind`：表示支持内存迁移；
* `get_area_memlocation`：表示获取分配的内存区域在NUMA树中的位置。

这些标志的具体含义可以在附录中查看。


```cpp
/** \brief Flags describing actual memory binding support for this topology.
 *
 * A flag may be set even if the feature isn't supported in all cases
 * (e.g. binding to random sets of non-contiguous objects).
 */
struct hwloc_topology_membind_support {
  /** Binding the whole current process is supported.  */
  unsigned char set_thisproc_membind;
  /** Getting the binding of the whole current process is supported.  */
  unsigned char get_thisproc_membind;
  /** Binding a whole given process is supported.  */
  unsigned char set_proc_membind;
  /** Getting the binding of a whole given process is supported.  */
  unsigned char get_proc_membind;
  /** Binding the current thread only is supported.  */
  unsigned char set_thisthread_membind;
  /** Getting the binding of the current thread only is supported.  */
  unsigned char get_thisthread_membind;
  /** Binding a given memory area is supported. */
  unsigned char set_area_membind;
  /** Getting the binding of a given memory area is supported.  */
  unsigned char get_area_membind;
  /** Allocating a bound memory area is supported. */
  unsigned char alloc_membind;
  /** First-touch policy is supported. */
  unsigned char firsttouch_membind;
  /** Bind policy is supported. */
  unsigned char bind_membind;
  /** Interleave policy is supported. */
  unsigned char interleave_membind;
  /** Next-touch migration policy is supported. */
  unsigned char nexttouch_membind;
  /** Migration flags is supported. */
  unsigned char migrate_membind;
  /** Getting the last NUMA nodes where a memory area was allocated is supported */
  unsigned char get_area_memlocation;
};

```

这段代码定义了一个名为 `hwloc_topology_misc_support` 的结构体，描述了在topology中进行其他操作时需要支持的一些特性。

该结构体包含一个名为 `imported_support` 的无符号字节数组，用于指示在当前topology中是否已经从另一个topology中导入过支持。

另外，该结构体还包含一个名为 `discovery` 的 `hwloc_topology_discovery_support` 指针，用于指示topology对象是否支持发现新的topology，以及一个名为 `cpubind` 的 `hwloc_topology_cpubind_support` 指针，用于指示topology对象是否支持对CPU进行映射。

此外，该结构体还包括一个名为 `membind` 的 `hwloc_topology_membind_support` 指针，用于指示topology对象是否支持对内存进行映射，以及一个名为 `misc` 的 `hwloc_topology_misc_support` 指针，用于指示当前topology中包含的其他支持特性。


```cpp
/** \brief Flags describing miscellaneous features.
 */
struct hwloc_topology_misc_support {
  /** Support was imported when importing another topology, see ::HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT. */
  unsigned char imported_support;
};

/** \brief Set of flags describing actual support for this topology.
 *
 * This is retrieved with hwloc_topology_get_support() and will be valid until
 * the topology object is destroyed.  Note: the values are correct only after
 * discovery.
 */
struct hwloc_topology_support {
  struct hwloc_topology_discovery_support *discovery;
  struct hwloc_topology_cpubind_support *cpubind;
  struct hwloc_topology_membind_support *membind;
  struct hwloc_topology_misc_support *misc;
};

```

这段代码定义了一个名为"topology_support"的函数，它用于获取拓扑支持。它通过一个二进制整型数组指定每个特征是否被支持，其中设置为0表示不支持，设置为1表示支持。但需要注意的是，如果设置为1，则表示支持，但有些角落案例可能仍然失败。

这个函数可以用于报告当前拓扑支持的功能，也可以用于报告原始机器上的拓扑支持。通过调用这个函数，用户可以了解哪些功能在当前拓扑上得到了支持。如果将拓扑导出为XML格式并导入到当前机器上，函数将报告支持的功能，否则默认情况下将报告 binding 不支持。


```cpp
/** \brief Retrieve the topology support.
 *
 * Each flag indicates whether a feature is supported.
 * If set to 0, the feature is not supported.
 * If set to 1, the feature is supported, but the corresponding
 * call may still fail in some corner cases.
 *
 * These features are also listed by hwloc-info \--support
 *
 * The reported features are what the current topology supports
 * on the current machine. If the topology was exported to XML
 * from another machine and later imported here, support still
 * describes what is supported for this imported topology after
 * import. By default, binding will be reported as unsupported
 * in this case (see ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM).
 *
 * Topology flag ::HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT may be used
 * to report the supported features of the original remote machine
 * instead. If it was successfully imported, \p imported_support
 * will be set in the struct hwloc_topology_misc_support array.
 */
```

In the context of the HWLOC_OBJ_MACHINE type, the `HWLOC_TYPE_FILTER_KEEP_NONE` field specifies that the object should be ignored if it does not bring any structure to the topology. The `HWLOC_TYPE_FILTER_KEEP_STRUCTURE` field specifies that the object should be kept if it brings structure to the topology, regardless of its children. The `HWLOC_TYPE_FILTER_KEEP_IMPORTANT` field specifies that the object should be kept if it is important and it brings structure to the topology, regardless of its children.

It is important to note that if any of these fields are set to 0, it means that the object will be ignored or dropped, regardless of the context in which it is used.


```cpp
HWLOC_DECLSPEC const struct hwloc_topology_support *hwloc_topology_get_support(hwloc_topology_t __hwloc_restrict topology);

/** \brief Type filtering flags.
 *
 * By default, most objects are kept (::HWLOC_TYPE_FILTER_KEEP_ALL).
 * Instruction caches, I/O and Misc objects are ignored by default (::HWLOC_TYPE_FILTER_KEEP_NONE).
 * Die and Group levels are ignored unless they bring structure (::HWLOC_TYPE_FILTER_KEEP_STRUCTURE).
 *
 * Note that group objects are also ignored individually (without the entire level)
 * when they do not bring structure.
 */
enum hwloc_type_filter_e {
  /** \brief Keep all objects of this type.
   *
   * Cannot be set for ::HWLOC_OBJ_GROUP (groups are designed only to add more structure to the topology).
   * \hideinitializer
   */
  HWLOC_TYPE_FILTER_KEEP_ALL = 0,

  /** \brief Ignore all objects of this type.
   *
   * The bottom-level type ::HWLOC_OBJ_PU, the ::HWLOC_OBJ_NUMANODE type, and
   * the top-level type ::HWLOC_OBJ_MACHINE may not be ignored.
   * \hideinitializer
   */
  HWLOC_TYPE_FILTER_KEEP_NONE = 1,

  /** \brief Only ignore objects if their entire level does not bring any structure.
   *
   * Keep the entire level of objects if at least one of these objects adds
   * structure to the topology. An object brings structure when it has multiple
   * children and it is not the only child of its parent.
   *
   * If all objects in the level are the only child of their parent, and if none
   * of them has multiple children, the entire level is removed.
   *
   * Cannot be set for I/O and Misc objects since the topology structure does not matter there.
   * \hideinitializer
   */
  HWLOC_TYPE_FILTER_KEEP_STRUCTURE = 2,

  /** \brief Only keep likely-important objects of the given type.
   *
   * It is only useful for I/O object types.
   * For ::HWLOC_OBJ_PCI_DEVICE and ::HWLOC_OBJ_OS_DEVICE, it means that only objects
   * of major/common kinds are kept (storage, network, OpenFabrics, CUDA,
   * OpenCL, RSMI, NVML, and displays).
   * Also, only OS devices directly attached on PCI (e.g. no USB) are reported.
   * For ::HWLOC_OBJ_BRIDGE, it means that bridges are kept only if they have children.
   *
   * This flag equivalent to ::HWLOC_TYPE_FILTER_KEEP_ALL for Normal, Memory and Misc types
   * since they are likely important.
   * \hideinitializer
   */
  HWLOC_TYPE_FILTER_KEEP_IMPORTANT = 3
};

```

这段代码定义了三个函数，用于设置和获取给定对象的过滤器。

第一个函数 `hwloc_topology_set_type_filter` 接受一个 `hwloc_topology_t` 类型的顶片和一个 `hwloc_obj_type_t` 类型的类型，并返回一个 `enum hwloc_type_filter_e` 类型的整数。它用于设置给定对象类型的过滤器。

第二个函数 `hwloc_topology_get_type_filter` 与第一个函数相反，它接收一个 `hwloc_topology_t` 类型的顶片和一个 `hwloc_obj_type_t` 类型的类型和一个指向 `enum hwloc_type_filter_e` 类型的整数的指针，并返回该指针。它用于获取给定对象类型的过滤器。

第三个函数 `hwloc_topology_set_all_types_filter` 与第二个函数不同，它接受一个 `hwloc_topology_t` 类型的顶片，并返回一个 `enum hwloc_type_filter_e` 类型的整数。如果某个对象类型不支持过滤器，那么该函数将忽略该类型。

第一个函数和第三个函数都使用了 `enum hwloc_type_filter_e` 类型，该类型定义了多种不同的过滤器类型。通过 `enum hwloc_type_filter_e` 中的下标，可以访问 `enum` 中的所有定义。


```cpp
/** \brief Set the filtering for the given object type.
 */
HWLOC_DECLSPEC int hwloc_topology_set_type_filter(hwloc_topology_t topology, hwloc_obj_type_t type, enum hwloc_type_filter_e filter);

/** \brief Get the current filtering for the given object type.
 */
HWLOC_DECLSPEC int hwloc_topology_get_type_filter(hwloc_topology_t topology, hwloc_obj_type_t type, enum hwloc_type_filter_e *filter);

/** \brief Set the filtering for all object types.
 *
 * If some types do not support this filtering, they are silently ignored.
 */
HWLOC_DECLSPEC int hwloc_topology_set_all_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter);

/** \brief Set the filtering for all CPU cache object types.
 *
 * Memory-side caches are not involved since they are not CPU caches.
 */
```

这段代码定义了两个函数，分别名为`hwloc_topology_set_cache_types_filter`和`hwloc_topology_set_icache_types_filter`，它们都接受一个`hwloc_topology_t`类型的参数。这两个函数用于设置基于缓存的CPU指令高速缓存和I/O对象的过滤类型。

具体来说，这两个函数都使用了`enum hwloc_type_filter_e`类型来表示过滤类型，这个类型枚举了所有可用的过滤类型。在函数内部，使用了`hwloc_topology_set_<filter_type>()`函数形式，其中`<filter_type>`是用户自定义的缓存类型，可以将其初始化为`NULL`。

函数的作用是设置基于缓存的CPU指令高速缓存和I/O对象的过滤类型，以满足应用程序的特定需求。这两个函数都被定义为`hwloc_topology_set_<filter_type>()`的形式，其中`<filter_type>`是用户自定义的缓存类型，可以将其初始化为`NULL`。


```cpp
HWLOC_DECLSPEC int hwloc_topology_set_cache_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter);

/** \brief Set the filtering for all CPU instruction cache object types.
 *
 * Memory-side caches are not involved since they are not CPU caches.
 */
HWLOC_DECLSPEC int hwloc_topology_set_icache_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter);

/** \brief Set the filtering for all I/O object types.
 */
HWLOC_DECLSPEC int hwloc_topology_set_io_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter);

/** \brief Set the topology-specific userdata pointer.
 *
 * Each topology may store one application-given private data pointer.
 * It is initialized to \c NULL.
 * hwloc will never modify it.
 *
 * Use it as you wish, after hwloc_topology_init() and until hwloc_topolog_destroy().
 *
 * This pointer is not exported to XML.
 */
```



这段代码定义了两个函数，用于设置和获取 topology 对象的用户数据。

函数 `hwloc_topology_set_userdata` 接受一个 `hwloc_topology_t` 类型的参数 `topology`，以及一个 `const void *` 类型的参数 `userdata`。它的作用是设置指定 topology 对象下的用户数据，将 `userdata` 存储在 `topology` 指向的内存区域中。

函数 `hwloc_topology_get_userdata` 接受一个 `hwloc_topology_t` 类型的参数 `topology`，它返回指定 topology 对象下的用户数据，该用户数据是指向 `userdata` 指针的指针。

这两个函数是互相关联的，可以用来设置和获取用户数据。


```cpp
HWLOC_DECLSPEC void hwloc_topology_set_userdata(hwloc_topology_t topology, const void *userdata);

/** \brief Retrieve the topology-specific userdata pointer.
 *
 * Retrieve the application-given private data pointer that was
 * previously set with hwloc_topology_set_userdata().
 */
HWLOC_DECLSPEC void * hwloc_topology_get_userdata(hwloc_topology_t topology);

/** @} */



/** \defgroup hwlocality_tinker Modifying a loaded Topology
 * @{
 */

```

This is an enum definition that defines a set of flags for hwloc\_topology\_restrict() function. These flags include information about what should be removed from the system, such as objects that have become CPUless, nodeset, or memoryless. Some of the flags, such as HWLOC\_RESTRICT\_FLAG\_REMOVE\_CPULESS and HWLOC\_RESTRICT\_FLAG\_REMOVE\_MEMLESS, have default values that should not be used in conjunction with other flags.


```cpp
/** \brief Flags to be given to hwloc_topology_restrict(). */
enum hwloc_restrict_flags_e {
  /** \brief Remove all objects that became CPU-less.
   * By default, only objects that contain no PU and no memory are removed.
   * This flag may not be used with ::HWLOC_RESTRICT_FLAG_BYNODESET.
   * \hideinitializer
   */
  HWLOC_RESTRICT_FLAG_REMOVE_CPULESS = (1UL<<0),

  /** \brief Restrict by nodeset instead of CPU set.
   * Only keep objects whose nodeset is included or partially included in the given set.
   * This flag may not be used with ::HWLOC_RESTRICT_FLAG_REMOVE_CPULESS.
   */
  HWLOC_RESTRICT_FLAG_BYNODESET =  (1UL<<3),

  /** \brief Remove all objects that became Memory-less.
   * By default, only objects that contain no PU and no memory are removed.
   * This flag may only be used with ::HWLOC_RESTRICT_FLAG_BYNODESET.
   * \hideinitializer
   */
  HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS = (1UL<<4),

  /** \brief Move Misc objects to ancestors if their parents are removed during restriction.
   * If this flag is not set, Misc objects are removed when their parents are removed.
   * \hideinitializer
   */
  HWLOC_RESTRICT_FLAG_ADAPT_MISC = (1UL<<1),

  /** \brief Move I/O objects to ancestors if their parents are removed during restriction.
   * If this flag is not set, I/O devices and bridges are removed when their parents are removed.
   * \hideinitializer
   */
  HWLOC_RESTRICT_FLAG_ADAPT_IO = (1UL<<2)
};

```

这段代码是一个C语言函数，名为`restrict_topology`，它用于限制给定CPU集或节点集的拓扑结构。

函数接受一个整数参数`topology`，它是一个表示拓扑结构的整数类型。函数还接受一个整数参数`flags`，它是一个包含HWLOC_RESTRICT_FLAG_CPUSET和HWLOC_RESTRICT_FLAG_NODESET的OR操作结果。

函数的主要作用是检查输入的拓扑结构是否符合要求。如果`flags`中包含了`HWLOC_RESTRICT_FLAG_NODESET`，则函数将输入的拓扑结构视为节点集，否则将其视为CPU集。

如果`topology`和`flags`都正确，函数将返回0，表示拓扑结构没有被修改。如果`topology`或`flags`其中一个不正确，函数将返回一个非零错误码。

函数还实现了一个注释，指出当函数被调用时，它不会尝试将输入的拓扑结构恢复为更大的拓扑结构。


```cpp
/** \brief Restrict the topology to the given CPU set or nodeset.
 *
 * Topology \p topology is modified so as to remove all objects that
 * are not included (or partially included) in the CPU set \p set.
 * All objects CPU and node sets are restricted accordingly.
 *
 * If ::HWLOC_RESTRICT_FLAG_BYNODESET is passed in \p flags,
 * \p set is considered a nodeset instead of a CPU set.
 *
 * \p flags is a OR'ed set of ::hwloc_restrict_flags_e.
 *
 * \note This call may not be reverted by restricting back to a larger
 * set. Once dropped during restriction, objects may not be brought
 * back, except by loading another topology with hwloc_topology_load().
 *
 * \return 0 on success.
 *
 * \return -1 with errno set to EINVAL if the input set is invalid.
 * The topology is not modified in this case.
 *
 * \return -1 with errno set to ENOMEM on failure to allocate internal data.
 * The topology is reinitialized in this case. It should be either
 * destroyed with hwloc_topology_destroy() or configured and loaded again.
 */
```

这段代码是一个名为 `hwloc_topology_restrict` 的函数，属于 `hwloc_topology_t` 系列的函数，用于设置在给定的拓扑中允许使用的硬件资源。它接受一个 `hwloc_topology_t` 类型的参数 `topology`，一个 `hwloc_const_bitmap_t` 类型的参数 `set`，以及一个 `unsigned long` 类型的参数 `flags`。

函数的作用是设置或取消在给定拓扑中允许使用的硬件资源，具体设置的标志由 `HWLOC_ALLOW_FLAG_ALL`、`HWLOC_ALLOW_FLAG_LOCAL_RESTRICTIONS` 和 `HWLOC_ALLOW_FLAG_CUSTOM` 中的一个或多个共同决定。如果设置了 `HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM`，则允许从操作系统中检索可用的资源。如果设置了 `HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM` 和 `HWLOC_ALLOW_FLAG_CUSTOM`，则允许自定义设置，但必须保证设置的 `set` 中包含的硬件资源是在拓扑允许的范围内。如果设置了 `HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM` 和 `HWLOC_ALLOW_FLAG_ALL`，则允许操作系统中已知的所有硬件资源。


```cpp
HWLOC_DECLSPEC int hwloc_topology_restrict(hwloc_topology_t __hwloc_restrict topology, hwloc_const_bitmap_t set, unsigned long flags);

/** \brief Flags to be given to hwloc_topology_allow(). */
enum hwloc_allow_flags_e {
  /** \brief Mark all objects as allowed in the topology.
   *
   * \p cpuset and \p nođeset given to hwloc_topology_allow() must be \c NULL.
   * \hideinitializer */
  HWLOC_ALLOW_FLAG_ALL = (1UL<<0),

  /** \brief Only allow objects that are available to the current process.
   *
   * The topology must have ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM so that the set
   * of available resources can actually be retrieved from the operating system.
   *
   * \p cpuset and \p nođeset given to hwloc_topology_allow() must be \c NULL.
   * \hideinitializer */
  HWLOC_ALLOW_FLAG_LOCAL_RESTRICTIONS = (1UL<<1),

  /** \brief Allow a custom set of objects, given to hwloc_topology_allow() as \p cpuset and/or \p nodeset parameters.
   * \hideinitializer */
  HWLOC_ALLOW_FLAG_CUSTOM = (1UL<<2)
};

```

这段代码定义了一个名为 `allow_topology_change` 的函数，它的作用是改变topology中允许的PU和NUMA节点集。这个函数只在topology中设置了一个不允许使用 `::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED` 的标志，然后在函数内部对允许的CPU集和节点集进行了修改，使得系统可以更好地支持从其他Cgroups中导入的topology。当需要在topology中更改允许的CPU集或节点集时，应该使用 `hwloc_topology_restrict()` 函数而不是修改topology本身。


```cpp
/** \brief Change the sets of allowed PUs and NUMA nodes in the topology.
 *
 * This function only works if the ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED
 * was set on the topology. It does not modify any object, it only changes
 * the sets returned by hwloc_topology_get_allowed_cpuset() and
 * hwloc_topology_get_allowed_nodeset().
 *
 * It is notably useful when importing a topology from another process
 * running in a different Linux Cgroup.
 *
 * \p flags must be set to one flag among ::hwloc_allow_flags_e.
 *
 * \note Removing objects from a topology should rather be performed with
 * hwloc_topology_restrict().
 */
```

这段代码定义了一个名为 `hwloc_topology_allow` 的函数，属于 `hwloc_topology_t` 类型的参数有三个，分别是一个指向 `hwloc_topology_t` 类型变量的 `topology`，一个指向 `hwloc_const_cpuset_t` 类型的 `cpuset` 和一个指向 `hwloc_const_nodeset_t` 类型的 `nodeset`，还有一个无符号长整型参数 `flags`。

函数的作用是允许在一个 `hwloc_topology_t` 类型的 topology 中添加一个新的 `MISC` 对象，作为根节点或者子节点，并将它插入到指定的位置。允许在叶子节点上使用 `hwloc_const_nodeset_t` 类型的子节点集，但是叶子节点本身不能有 `cpu` 成员。函数返回一个新创建的 `MISC` 对象，或者在错误情况下返回 `NULL`。


```cpp
HWLOC_DECLSPEC int hwloc_topology_allow(hwloc_topology_t __hwloc_restrict topology, hwloc_const_cpuset_t cpuset, hwloc_const_nodeset_t nodeset, unsigned long flags);

/** \brief Add a MISC object as a leaf of the topology
 *
 * A new MISC object will be created and inserted into the topology at the
 * position given by parent. It is appended to the list of existing Misc children,
 * without ever adding any intermediate hierarchy level. This is useful for
 * annotating the topology without actually changing the hierarchy.
 *
 * \p name is supposed to be unique across all Misc objects in the topology.
 * It will be duplicated to setup the new object attributes.
 *
 * The new leaf object will not have any \p cpuset.
 *
 * \return the newly-created object
 *
 * \return \c NULL on error.
 *
 * \return \c NULL if Misc objects are filtered-out of the topology (::HWLOC_TYPE_FILTER_KEEP_NONE).
 *
 * \note If \p name contains some non-printable characters, they will
 * be dropped when exporting to XML, see hwloc_topology_export_xml() in hwloc/export.h.
 */
```

This is defining the behavior of the `hwloc_group_insert()` function in the `hwloc` library.

The function takes two arguments: `ecpuset` and `nsetset`, which must be set to a non-empty bitmap. If `ecpuset` is not set, the function falls back to using the complete topology (i.e., all objects at the same location in the topology tree).

If `ecpuset` is set, the function will insert the specified group into the topology. If the `nsets` field is not specified, the function will insert the group into the topology by default. If the `nsets` field is set, the function will insert the group into the topology by specifying the set of nodes at the specified location.

If the group is being inserted into the topology, the function will check for conflicts with the topology tree. If there are no conflicts, the function will insert the group into the topology. If there is a conflict, the function will return an error.

If the group is being inserted as part of a group of objects, the function can be used to iteratively add the group's sets to the topology. If the `hwloc_obj_add_other_obj_sets()` function is used to insert the group's sets, the function can be used to add the sets to the topology.

The function also allows for the specification of the object's subtype attribute. If it is set, it will be displayed as the subtype name in the `lstopo` library, instead of "Group". Custom name/value information can also be added with `hwloc_obj_add_info()` after insertion.

Finally, the function also allows for the specification of the `dont_merge` attribute for the group. If set to `1`, the function will prevent the core from merging this group with any other hierarchically-identical objects. This can be useful when the group itself describes an important feature that should not be exposed elsewhere in the hierarchy.


```cpp
HWLOC_DECLSPEC hwloc_obj_t hwloc_topology_insert_misc_object(hwloc_topology_t topology, hwloc_obj_t parent, const char *name);

/** \brief Allocate a Group object to insert later with hwloc_topology_insert_group_object().
 *
 * This function returns a new Group object.
 *
 * The caller should (at least) initialize its sets before inserting
 * the object in the topology. See hwloc_topology_insert_group_object().
 */
HWLOC_DECLSPEC hwloc_obj_t hwloc_topology_alloc_group_object(hwloc_topology_t topology);

/** \brief Add more structure to the topology by adding an intermediate Group
 *
 * The caller should first allocate a new Group object with hwloc_topology_alloc_group_object().
 * Then it must setup at least one of its CPU or node sets to specify
 * the final location of the Group in the topology.
 * Then the object can be passed to this function for actual insertion in the topology.
 *
 * Either the cpuset or nodeset field (or both, if compatible) must be set
 * to a non-empty bitmap. The complete_cpuset or complete_nodeset may be set
 * instead if inserting with respect to the complete topology
 * (including disallowed, offline or unknown objects).
 * If grouping several objects, hwloc_obj_add_other_obj_sets() is an easy way
 * to build the Group sets iteratively.
 * These sets cannot be larger than the current topology, or they would get
 * restricted silently.
 * The core will setup the other sets after actual insertion.
 *
 * The \p subtype object attribute may be defined (to a dynamically
 * allocated string) to display something else than "Group" as the
 * type name for this object in lstopo.
 * Custom name/value info pairs may be added with hwloc_obj_add_info() after
 * insertion.
 *
 * The group \p dont_merge attribute may be set to \c 1 to prevent
 * the hwloc core from ever merging this object with another
 * hierarchically-identical object.
 * This is useful when the Group itself describes an important feature
 * that cannot be exposed anywhere else in the hierarchy.
 *
 * The group \p kind attribute may be set to a high value such
 * as \c 0xffffffff to tell hwloc that this new Group should always
 * be discarded in favor of any existing Group with the same locality.
 *
 * \return The inserted object if it was properly inserted.
 *
 * \return An existing object if the Group was merged or discarded
 * because the topology already contained an object at the same
 * location (the Group did not add any hierarchy information).
 *
 * \return \c NULL if the insertion failed because of conflicting sets in topology tree.
 *
 * \return \c NULL if Group objects are filtered-out of the topology (::HWLOC_TYPE_FILTER_KEEP_NONE).
 *
 * \return \c NULL if the object was discarded because no set was
 * initialized in the Group before insert, or all of them were empty.
 */
```

这段代码定义了两个函数，一个是 `hwloc_obj_add_other_obj_sets`，另一个是 `hwloc_obj_refresh`。

`hwloc_obj_add_other_obj_sets` 函数接受两个参数，一个是 `dst`，表示新生成的中间对象父节点，另一个是 `src`，表示源对象。这个函数的作用是在 `hwloc_topology_alloc_group_object` 和 `hwloc_topology_insert_group_object` 之间，通过 `OR` 操作将另一个对象的集合设置为新生成的中间对象父节点的集合，并将 `src` 对象添加到新生成的集合中。

`hwloc_obj_refresh` 函数是一个内部调用，用于刷新 `hwloc_topology_topology` 结构中的内部结构，这些内部结构在 topology 修改后可能会变得无效。当调用 `hwloc_topology_topology` 结构中的其他函数之后，调用 `hwloc_obj_refresh` 函数以确保 `hwloc_topology_topology` 结构中的内部结构保持最新状态。

这两个函数共同作用于 topology 对象的刷新和维护，使 topology 对象能够正确地继承并维护其父对象的集合。


```cpp
HWLOC_DECLSPEC hwloc_obj_t hwloc_topology_insert_group_object(hwloc_topology_t topology, hwloc_obj_t group);

/** \brief Setup object cpusets/nodesets by OR'ing another object's sets.
 *
 * For each defined cpuset or nodeset in \p src, allocate the corresponding set
 * in \p dst and add \p src to it by OR'ing sets.
 *
 * This function is convenient between hwloc_topology_alloc_group_object()
 * and hwloc_topology_insert_group_object(). It builds the sets of the new Group
 * that will be inserted as a new intermediate parent of several objects.
 */
HWLOC_DECLSPEC int hwloc_obj_add_other_obj_sets(hwloc_obj_t dst, hwloc_obj_t src);

/** \brief Refresh internal structures after topology modification.
 *
 * Modifying the topology (by restricting, adding objects, modifying structures
 * such as distances or memory attributes, etc.) may cause some internal caches
 * to become invalid. These caches are automatically refreshed when accessed
 * but this refreshing is not thread-safe.
 *
 * This function is not thread-safe either, but it is a good way to end a
 * non-thread-safe phase of topology modification. Once this refresh is done,
 * multiple threads may concurrently consult the topology, objects, distances,
 * attributes, etc.
 *
 * See also \ref threadsafety
 */
```



这段代码是一个C语言定义的函数，名为"hwloc_topology_refresh"，属于"hwloc"库。它用于刷新硬件位置（HWLOC） topology，使得当 topology 发生变化时，旧的信息将不再被使用，从而达到重新初始化的目的。

具体来说，这个函数接受一个 HWLOC_TOPOLOGY_T 类型的参数，表示要更新的 topology 结构体。函数内部对 topology 结构体进行修改，然后使用 topology 参数hwloc_topology_get_current_layout() 获取当前布局，最后将修改后的 topology 结构体重新设置为当前布局，从而实现 topology 的刷新。


```cpp
HWLOC_DECLSPEC int hwloc_topology_refresh(hwloc_topology_t topology);

/** @} */



#ifdef __cplusplus
} /* extern "C" */
#endif


/* high-level helpers */
#include "hwloc/helper.h"

/* inline code of some functions above */
```



This code appears to define a memory layout for a System on a System (SOS) architecture. The SOS architecture is a hardware system that is designed to be more energy-efficient than traditional x86 systems by leveraging the power-per-core and memory-per-core properties of the processor.

The code includes several header files that are included to define the APIs and data structures that are used in the code. These include "hwloc/inlines.h", which defines inline memory functions, "hwloc/memattrs.h", which defines memory attributes for the SOS architecture, "hwloc/cpukinds.h", which defines the CPU kinds in the SOS architecture, and "hwloc/export.h", which defines the exporting of the SOS architecture to XML or synthetic documentation.

The main functionality of the code appears to be to define the memory layout for a SOS system. This includes defining the memory attributes for the SOS architecture, such as the number and type of cores, the memory hierarchy, and the memory-per-core and power-per-core properties. The code also includes definitions for the different types of CPU cores that can be used in the SOS architecture, such as the Core微体系、缓存微体系等。

此外，代码还定义了一系列距离度量（distances）以支持系统热设计。最后，代码还定义了如何将SOS架构导出为XML或图形化文档，以便于用户了解和调试SOS系统。


```cpp
#include "hwloc/inlines.h"

/* memory attributes */
#include "hwloc/memattrs.h"

/* kinds of CPU cores */
#include "hwloc/cpukinds.h"

/* exporting to XML or synthetic */
#include "hwloc/export.h"

/* distances */
#include "hwloc/distances.h"

/* topology diffs */
```

这段代码是一个C语言代码片段，它包含了两个头文件和一些 deprecated（过时）的代码。下面是这段代码的作用和一些可能的用途。

首先，它引入了两个头文件：

1. "hwloc/diff.h" 是 "hwloc" 库的头部文件，它提供了用于在 hierarchical和世界树（HWT）中计算 diff（差异）的函数和数据结构。

2. "hwloc/deprecated.h" 是 "hwloc" 库的派生头文件，它包含了 deprecated（过时）的函数和数据结构，这些函数和数据结构在代码中已被 deprecated（过时）。

接下来，定义了一个名为 "diff" 的函数，该函数可能接受两个 HWT 结构体作为其参数，计算两个 HWT 之间的差异。

最后，由于定义在 "hwloc/deprecated.h" 中的 deprecated 函数和数据结构，在文件头文件 "hwloc/diff.h" 中也被deprecated（过时），因此提醒开发人员不要在文件中使用这些过时的函数和数据结构。


```cpp
#include "hwloc/diff.h"

/* deprecated headers */
#include "hwloc/deprecated.h"

#endif /* HWLOC_H */

```