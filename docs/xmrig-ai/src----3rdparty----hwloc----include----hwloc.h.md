# `xmrig\src\3rdparty\hwloc\include\hwloc.h`

```
/*
 * 版权 © 2009 CNRS
 * 版权 © 2009-2022 Inria。保留所有权利。
 * 版权 © 2009-2012 波尔多大学
 * 版权 © 2009-2020思科系统公司。保留所有权利。
 * 请参阅顶层目录中的 COPYING 文件。
 */

/*=====================================================================
 *                 请务必阅读文档！
 *         ------------------------------------------------
 *               $tarball_directory/doc/doxygen-doc/
 *                                或
 *           https://www.open-mpi.org/projects/hwloc/doc/
 *=====================================================================
 *
 * 警告：不要指望仅通过阅读此文件中的函数原型和常量描述来弄清楚hwloc的所有微妙之处。
 *
 * Hwloc有精彩的文档，以PDF和HTML格式提供，供您阅读。正式文档解释了许多hwloc特定的概念，提供定义，并讨论了您在此头文件中找到的许多内容的“大局观”。
 *
 * PDF/HTML文档是通过Doxygen生成的；您在其中看到的许多内容也在此文件中。但是PDF/HTML中有很多内容是***不在***hwloc.h中的！
 *
 * 整个段落长度的描述、讨论和漂亮的图片用于解释微妙的边缘情况，提供具体示例等等。
 *
 * 请务必阅读文档。 :-)
 *
 * 此外，在源代码树的doc/examples下有几个hwloc使用示例。
 *
 *=====================================================================*/
/**
 * \file
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
#define HWLOC_API_VERSION 0x00020800

/** \brief Indicate at runtime which hwloc API version was used at build time.
 *
 * Should be ::HWLOC_API_VERSION if running on the same version.
 */
HWLOC_DECLSPEC unsigned hwloc_get_api_version(void);

/** \brief Current component and plugin ABI version (see hwloc/plugins.h) */
#define HWLOC_COMPONENT_ABI 7

/** @} */
/**
 * \defgroup hwlocality_object_sets Object Sets (hwloc_cpuset_t and hwloc_nodeset_t)
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
typedef hwloc_bitmap_t hwloc_nodeset_t;
/** \brief A non-modifiable ::hwloc_nodeset_t.
 */
typedef hwloc_const_bitmap_t hwloc_const_nodeset_t;

/** @} */
/**
 * @defgroup hwlocality_object_types Object Types
 * @{
 */

/** \brief Type of topology object.
 *
 * \note Do not rely on the ordering or completeness of the values as new ones
 * may be defined in the future!  If you need to compare types, use
 * hwloc_compare_types() instead.
 */
typedef enum {

/** \cond */
#define HWLOC_OBJ_TYPE_MIN HWLOC_OBJ_MACHINE /* Sentinel value */
} hwloc_obj_type_t;

/** \brief Cache type. */
typedef enum hwloc_obj_cache_type_e {
  HWLOC_OBJ_CACHE_UNIFIED,      /**< \brief Unified cache. */
  HWLOC_OBJ_CACHE_DATA,         /**< \brief Data cache. */
  HWLOC_OBJ_CACHE_INSTRUCTION   /**< \brief Instruction cache (filtered out by default). */
} hwloc_obj_cache_type_t;

/** \brief Type of one side (upstream or downstream) of an I/O bridge. */
typedef enum hwloc_obj_bridge_type_e {
  HWLOC_OBJ_BRIDGE_HOST,    /**< \brief Host-side of a bridge, only possible upstream. */
  HWLOC_OBJ_BRIDGE_PCI        /**< \brief PCI-side of a bridge. */
} hwloc_obj_bridge_type_t;

/** \brief Type of a OS device. */
typedef enum hwloc_obj_osdev_type_e {
  HWLOC_OBJ_OSDEV_BLOCK,    /**< \brief 操作系统块设备，或非易失性存储设备。
                  * 例如在 Linux 上的 "sda" 或 "dax2.0"。 */
  HWLOC_OBJ_OSDEV_GPU,        /**< \brief 操作系统 GPU 设备。
                  * 例如 GL 显示的 ":0.0"，
                  * Linux DRM 设备的 "card0"。 */
  HWLOC_OBJ_OSDEV_NETWORK,    /**< \brief 操作系统网络设备。
                  * 例如 Linux 上的 "eth0" 接口。 */
  HWLOC_OBJ_OSDEV_OPENFABRICS,    /**< \brief 操作系统开放式互连设备。
                  * 例如 InfiniBand HCA 的 "mlx4_0"，
                  * Omni-Path 接口的 "hfi1_0"，
                  * 或 Linux 上的 Atos/Bull BXI HCA 的 "bxi0"。 */
  HWLOC_OBJ_OSDEV_DMA,        /**< \brief 操作系统 DMA 引擎设备。
                  * 例如 Linux 上的 "dma0chan0" DMA 通道。 */
  HWLOC_OBJ_OSDEV_COPROC    /**< \brief 操作系统协处理器设备。
                  * 例如 OpenCL 设备的 "opencl0d0"，
                  * CUDA 设备的 "cuda0"。 */
} hwloc_obj_osdev_type_t;
/**
 * \brief 比较两个对象类型的深度
 *
 * 不应该直接比较类型，因为将来可能会添加新的类型。该函数返回小于、等于或大于零，分别表示 \p type1 对象通常包含 \p type2 对象、与 \p type2 对象相同，或者被 \p type2 对象包含。如果类型无法比较（因为两者都不包含在对方内），则返回 ::HWLOC_TYPE_UNORDERED。包含 CPU 的对象类型总是可以比较的（通常，系统包含机器，机器包含节点，节点包含封装，封装包含缓存，缓存包含核心，核心包含处理器）。
 *
 * \note ::HWLOC_OBJ_PU 总是最深层的，而 ::HWLOC_OBJ_MACHINE 总是最高层的。
 *
 * \note 这并不意味着实际拓扑将遵循该顺序：例如，截至今天，核心也可能包含缓存，封装也可能包含节点。因此，这只是作为一种备用比较方法。
 */
HWLOC_DECLSPEC int hwloc_compare_types (hwloc_obj_type_t type1, hwloc_obj_type_t type2) __hwloc_attribute_const;

/** \brief 当类型无法比较时，hwloc_compare_types() 返回的值。 \hideinitializer */
#define HWLOC_TYPE_UNORDERED INT_MAX

/** @} */

/** \defgroup hwlocality_objects 对象结构和属性
 * @{
 */

union hwloc_obj_attr_u;

/** \brief 拓扑对象的结构
 *
 * 应用程序不能修改任何字段，除了 \p hwloc_obj.userdata。
 */
struct hwloc_obj {
  /* 物理信息 */
  hwloc_obj_type_t type;        /**< \brief 对象类型 */
  char *subtype;            /**< \brief 子类型字符串，用于更好地描述类型字段。 */

  unsigned os_index;            /**< \brief 操作系统提供的物理索引号。
                     * 不能保证在整个机器上是唯一的，
                     * 除了处理器核心和 NUMA 节点。
                     * 如果未知或对该对象无关，则设置为 HWLOC_UNKNOWN_INDEX。
                     */
};
/**
 * \brief 方便的类型定义；指向 struct hwloc_obj 的指针。
 */
typedef struct hwloc_obj * hwloc_obj_t;

/** \brief 对象类型特定的属性 */
union hwloc_obj_attr_u {
  /** \brief NUMA 节点特定的对象属性 */
  struct hwloc_numanode_attr_s {
    hwloc_uint64_t local_memory; /**< \brief 本地内存（以字节为单位） */
    unsigned page_types_len; /**< \brief 数组 \p page_types 的大小 */
    /** \brief 本地内存页面类型的数组，如果没有本地内存则为 \c NULL，\p page_types 为 0。
     *
     * 数组按照递增的 \p size 字段进行排序。
     * 它包含 \p page_types_len 个槽位。
     */
    struct hwloc_memory_page_type_s {
      hwloc_uint64_t size;    /**< \brief 页面大小 */
      hwloc_uint64_t count;    /**< \brief 此大小的页面数 */
    } * page_types;
  } numanode;

  /** \brief 缓存特定的对象属性 */
  struct hwloc_cache_attr_s {
    hwloc_uint64_t size;          /**< \brief 缓存大小（以字节为单位） */
    unsigned depth;              /**< \brief 缓存深度（例如，L1、L2 等） */
    unsigned linesize;              /**< \brief 缓存行大小（以字节为单位）。如果未知，则为 0 */
    int associativity;              /**< \brief 关联度的方式，
                            * -1 表示完全关联，0 表示未知 */
    hwloc_obj_cache_type_t type;          /**< \brief 缓存类型 */
  } cache;
  /** \brief 组特定的对象属性 */
  struct hwloc_group_attr_s {
    # 组对象的深度
    unsigned depth;              /**< \brief Depth of group object.
                       *   It may change if intermediate Group objects are added. */
    # 组对象的内部使用种类
    unsigned kind;              /**< \brief Internally-used kind of group. */
    # 用于区分相同种类组的不同级别的内部使用子种类
    unsigned subkind;              /**< \brief Internally-used subkind to distinguish different levels of groups with same kind */
    # 防止组自动与相同父级或子级合并的标志
    unsigned char dont_merge;          /**< \brief Flag preventing groups from being automatically merged with identical parent or children. */
  } group;
  /** \brief PCI Device specific Object Attributes */
  # PCI设备特定的对象属性
  struct hwloc_pcidev_attr_s {
#ifndef HWLOC_HAVE_32BITS_PCI_DOMAIN
    // 如果没有定义 HWLOC_HAVE_32BITS_PCI_DOMAIN，则使用 unsigned short 类型的 domain
    unsigned short domain; /* Only 16bits PCI domains are supported by default */
#else
    // 如果定义了 HWLOC_HAVE_32BITS_PCI_DOMAIN，则使用 unsigned int 类型的 domain
    unsigned int domain; /* 32bits PCI domain support break the library ABI, hence it's disabled by default */
#endif
    // 定义其他的变量
    unsigned char bus, dev, func;
    unsigned short class_id;
    unsigned short vendor_id, device_id, subvendor_id, subdevice_id;
    unsigned char revision;
    float linkspeed; /* in GB/s */
  } pcidev;
  /** \brief Bridge specific Object Attributes */
  // 定义桥接设备的属性
  struct hwloc_bridge_attr_s {
    union {
      // 上游属性
      struct hwloc_pcidev_attr_s pci;
    } upstream;
    // 上游类型
    hwloc_obj_bridge_type_t upstream_type;
    union {
      // 下游属性
      struct {
#ifndef HWLOC_HAVE_32BITS_PCI_DOMAIN
    unsigned short domain; /* Only 16bits PCI domains are supported by default */
#else
    unsigned int domain; /* 32bits PCI domain support break the library ABI, hence it's disabled by default */
#endif
    unsigned char secondary_bus, subordinate_bus;
      } pci;
    } downstream;
    // 下游类型
    hwloc_obj_bridge_type_t downstream_type;
    // 深度
    unsigned depth;
  } bridge;
  /** \brief OS Device specific Object Attributes */
  // 操作系统设备的属性
  struct hwloc_osdev_attr_s {
    // 设备类型
    hwloc_obj_osdev_type_t type;
  } osdev;
};

/** \brief Object info
 *
 * \sa hwlocality_info_attr
 */
// 对象信息
struct hwloc_info_s {
  char *name;    /**< \brief Info name */
  char *value;    /**< \brief Info value */
};

/** @} */



/** \defgroup hwlocality_creation Topology Creation and Destruction
 * @{
 */

// 拓扑结构
struct hwloc_topology;
/** \brief Topology context
 *
 * To be initialized with hwloc_topology_init() and built with hwloc_topology_load().
 */
// 拓扑上下文
typedef struct hwloc_topology * hwloc_topology_t;

/** \brief Allocate a topology context.
 *
 * \param[out] topologyp is assigned a pointer to the new allocated context.
 *
 * \return 0 on success, -1 on error.
 */
// 分配拓扑上下文
HWLOC_DECLSPEC int hwloc_topology_init (hwloc_topology_t *topologyp);
# 构建实际的拓扑结构
#
# 一旦使用 hwloc_topology_init() 进行初始化，并使用 hwlocality_configuration 和 hwlocality_setsource 进行调优，
# 就可以构建实际的拓扑结构。在此之前不能使用任何其他例程来操作拓扑结构。
#
# 参数 topology 是要加载对象的拓扑结构。
#
# 成功返回 0，失败返回 -1。
#
# 失败时，拓扑结构将被重新初始化。应该使用 hwloc_topology_destroy() 销毁它，或者重新配置和加载。
#
# 这个函数只能对每个拓扑结构调用一次。
#
# 在此调用期间，当前线程或进程的绑定可能会暂时改变，但在返回之前会恢复。
#
# 参见 hwlocality_configuration 和 hwlocality_setsource
HWLOC_DECLSPEC int hwloc_topology_load(hwloc_topology_t topology);

# 终止和释放拓扑结构上下文
#
# 参数 topology 是要释放的拓扑结构
HWLOC_DECLSPEC void hwloc_topology_destroy (hwloc_topology_t topology);

# 复制一个拓扑结构
#
# 整个拓扑结构以及其对象都被复制到一个新的拓扑结构中。
#
# 这对于在修改拓扑结构时保留备份很有用。
#
# 注意：对象的用户数据不会被复制，因为 hwloc 不知道它指向什么。新旧拓扑结构的对象将指向相同的用户数据。
HWLOC_DECLSPEC int hwloc_topology_dup(hwloc_topology_t *newtopology, hwloc_topology_t oldtopology);
# 验证拓扑结构是否与当前的 hwloc 库兼容
# 当在不同的库中使用相同的拓扑结构（内存中）时很有用，这些库可能使用不同的 hwloc 安装
# （例如，如果一个库嵌入了特定版本的 hwloc，而另一个库使用默认的系统范围内的 hwloc 安装）。
# 如果所有的库/程序使用相同的 hwloc 安装，这个函数总是返回成功。
# 返回 0 表示成功。
# 返回 -1 并设置 errno 为 EINVAL 表示不兼容。
# 注意：如果使用 hwloc_shmem_topology_write() 在进程之间共享，相关的检查已经在 hwloc_shmem_topology_adopt() 中执行。
HWLOC_DECLSPEC int hwloc_topology_abi_check(hwloc_topology_t topology);

# 在拓扑结构上运行内部检查
# 如果在给定的拓扑结构中检测到不一致，程序将中止。
# 参数 topology 是要检查的拓扑结构
# 注意：这个例程只对开发人员有用。
# 注意：输入的拓扑结构应该先用 hwloc_topology_load() 加载。
HWLOC_DECLSPEC void hwloc_topology_check(hwloc_topology_t topology);

# 获取对象的层次树的深度
# 这是 ::HWLOC_OBJ_PU 对象的深度加一。
# 注意：在计算树的深度时，NUMA 节点、I/O 和 Misc 对象会被忽略（它们被放置在特殊的层级）。
HWLOC_DECLSPEC int hwloc_topology_get_depth(hwloc_topology_t __hwloc_restrict topology) __hwloc_attribute_pure;
/** \brief 返回类型为 \p type 的对象的深度。
 *
 * 如果在底层架构上不存在此类型的对象，或者操作系统不提供这种信息，函数将返回 ::HWLOC_TYPE_DEPTH_UNKNOWN。
 *
 * 如果类型不存在但类似类型可接受，也可以参考 hwloc_get_type_or_below_depth() 和 hwloc_get_type_or_above_depth()。
 *
 * 如果给定 ::HWLOC_OBJ_GROUP，函数可能返回 ::HWLOC_TYPE_DEPTH_MULTIPLE，如果拓扑中存在多个级别的组。
 *
 * 如果给定 NUMA 节点、I/O 或 Misc 对象类型，函数将返回虚拟值，因为这些对象存储在与 CPU 无关的特殊级别中。
 * 这个虚拟深度可以传递给其他 hwloc 函数，比如 hwloc_get_obj_by_depth()，但应用程序不应将其视为实际深度。
 * 特别是，它不应与任何其他对象深度或整个拓扑深度进行比较。
 * \sa hwloc_get_memory_parents_depth()。
 *
 * \sa hwloc_type_sscanf_as_depth() 用于返回给定类型为字符串时对象的深度。
 */
HWLOC_DECLSPEC int hwloc_get_type_depth (hwloc_topology_t topology, hwloc_obj_type_t type);

enum hwloc_get_type_depth_e {
    HWLOC_TYPE_DEPTH_UNKNOWN = -1,    /**< \brief 拓扑中不存在给定类型的对象。 \hideinitializer */
    HWLOC_TYPE_DEPTH_MULTIPLE = -2,   /**< \brief 拓扑中给定类型的对象存在于不同的深度（仅适用于组）。 \hideinitializer */
    HWLOC_TYPE_DEPTH_NUMANODE = -3,   /**< \brief NUMA 节点的虚拟深度。 \hideinitializer */
    HWLOC_TYPE_DEPTH_BRIDGE = -4,     /**< \brief 桥接对象级别的虚拟深度。 \hideinitializer */
    HWLOC_TYPE_DEPTH_PCI_DEVICE = -5, /**< \brief PCI 设备对象级别的虚拟深度。 \hideinitializer */
    HWLOC_TYPE_DEPTH_OS_DEVICE = -6,  /**< \brief 软件设备对象级别的虚拟深度。 \hideinitializer */
    # 定义 HWLOC_TYPE_DEPTH_MISC 常量，表示 Misc 对象的虚拟深度
    HWLOC_TYPE_DEPTH_MISC = -7,       /**< \brief Virtual depth for Misc object. \hideinitializer */
    # 定义 HWLOC_TYPE_DEPTH_MEMCACHE 常量，表示 MemCache 对象的虚拟深度
    HWLOC_TYPE_DEPTH_MEMCACHE = -8    /**< \brief Virtual depth for MemCache object. \hideinitializer */
};

/** \brief 返回内存对象附加的父级深度。
 *
 * 内存对象具有虚拟负深度，因为它们不是主 CPU 端对象层次结构的一部分。此深度不应与其他级别深度进行比较。
 *
 * 如果所有内存对象都附加到相同深度的普通父级，则可以像通常一样将此父级深度与其他父级深度进行比较，例如，用于了解 NUMA 节点是在包装器上方还是下方附加。
 *
 * \return 如果所有这些父级具有相同深度，则返回所有内存子级的普通父级的深度。例如，如果所有 NUMA 节点都附加到 Package 对象，则返回 Package 级的深度。
 *
 * \return 如果所有内存子级的普通父级深度不相同，则返回 ::HWLOC_TYPE_DEPTH_MULTIPLE。例如，如果一些 NUMA 节点附加到 Packages，而其他一些附加到 Groups，则返回 ::HWLOC_TYPE_DEPTH_MULTIPLE。
 */
HWLOC_DECLSPEC int hwloc_get_memory_parents_depth (hwloc_topology_t topology);

/** \brief 返回类型为 \p type 或以下对象的深度
 *
 * 如果底层架构上不存在此类型的对象，则函数返回通常在 \p type 内找到的第一个“存在”对象的深度。
 *
 * 此函数仅适用于普通对象类型。
 * 如果给定了内存、I/O 或 Misc 对象类型，则始终返回相应的虚拟深度（参见 hwloc_get_type_depth()）。
 *
 * 可能会像 hwloc_get_type_depth() 一样，对于 ::HWLOC_OBJ_GROUP，返回 ::HWLOC_TYPE_DEPTH_MULTIPLE。
 */
static __hwloc_inline int
hwloc_get_type_or_below_depth (hwloc_topology_t topology, hwloc_obj_type_t type) __hwloc_attribute_pure;
/** \brief 返回类型为 \p type 或以上的对象的深度
 *
 * 如果在底层架构中不存在此类型的对象，则函数返回通常包含 \p type 的第一个“存在”对象的深度。
 *
 * 此函数仅适用于普通对象类型。
 * 如果给定了内存、I/O 或其他类型的对象类型，则始终返回相应的虚拟深度（参见 hwloc_get_type_depth()）。
 *
 * 可能会像 hwloc_get_type_depth() 一样，对于 ::HWLOC_OBJ_GROUP，返回 ::HWLOC_TYPE_DEPTH_MULTIPLE。
 */
static __hwloc_inline int
hwloc_get_type_or_above_depth (hwloc_topology_t topology, hwloc_obj_type_t type) __hwloc_attribute_pure;

/** \brief 返回深度为 \p depth 的对象的类型。
 *
 * \p depth 应该在 0 和 hwloc_topology_get_depth()-1 之间，
 * 或者是虚拟深度，如 ::HWLOC_TYPE_DEPTH_NUMANODE。
 *
 * 如果深度 \p depth 不存在，则返回 (hwloc_obj_type_t)-1。
 */
HWLOC_DECLSPEC hwloc_obj_type_t hwloc_get_depth_type (hwloc_topology_t topology, int depth) __hwloc_attribute_pure;

/** \brief 返回深度为 \p depth 的级别的宽度。
 */
HWLOC_DECLSPEC unsigned hwloc_get_nbobjs_by_depth (hwloc_topology_t topology, int depth) __hwloc_attribute_pure;

/** \brief 返回类型为 \p type 的级别的宽度
 *
 * 如果不存在该类型的对象，则返回 0。
 * 如果存在多个具有该类型对象的级别，则返回 -1。
 */
static __hwloc_inline int
hwloc_get_nbobjs_by_type (hwloc_topology_t topology, hwloc_obj_type_t type) __hwloc_attribute_pure;

/** \brief 返回拓扑树的顶层对象。
 *
 * 其类型为 ::HWLOC_OBJ_MACHINE。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_root_obj (hwloc_topology_t topology) __hwloc_attribute_pure;

/** \brief 返回深度为 \p depth，逻辑索引为 \p idx 的拓扑对象 */
HWLOC_DECLSPEC hwloc_obj_t hwloc_get_obj_by_depth (hwloc_topology_t topology, int depth, unsigned idx) __hwloc_attribute_pure;
/** \brief 返回逻辑索引 \p idx 处类型为 \p type 的拓扑对象
 *
 * 如果不存在该类型的对象，则返回 \c NULL。
 * 如果存在多个层级具有该类型的对象 (::HWLOC_OBJ_GROUP)，则返回 \c NULL，并且调用者可以回退到 hwloc_get_obj_by_depth()。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_obj_by_type (hwloc_topology_t topology, hwloc_obj_type_t type, unsigned idx) __hwloc_attribute_pure;

/** \brief 返回深度为 \p depth 的下一个对象。
 *
 * 如果 \p prev 为 \c NULL，则返回深度为 \p depth 的第一个对象。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_by_depth (hwloc_topology_t topology, int depth, hwloc_obj_t prev);

/** \brief 返回类型为 \p type 的下一个对象。
 *
 * 如果 \p prev 为 \c NULL，则返回类型为 \p type 的第一个对象。如果给定类型有多个或没有深度，则返回 \c NULL，并让调用者回退到 hwloc_get_next_obj_by_depth()。
 */
static __hwloc_inline hwloc_obj_t
hwloc_get_next_obj_by_type (hwloc_topology_t topology, hwloc_obj_type_t type,
                hwloc_obj_t prev);

/** @} */



/** \defgroup hwlocality_object_strings 将对象类型和属性转换为字符串
 * @{
 */

/** \brief 返回一个常量字符串化的对象类型。
 *
 * 这个函数是将通用类型转换为字符串的基本方法。
 * 输出字符串可以通过 hwloc_type_sscanf() 进行解析。
 *
 * hwloc_obj_type_snprintf() 可能会为特定对象返回更精确的输出，但它需要调用者提供输出缓冲区。
 */
HWLOC_DECLSPEC const char * hwloc_obj_type_string (hwloc_obj_type_t type) __hwloc_attribute_const;
/**
 * \brief 将给定拓扑对象的类型转换为人类可读的形式。
 *
 * 与hwloc_obj_type_string()相反，此函数在输出中包含对象特定属性（如组深度、桥类型或OS设备类型），并要求调用者提供输出缓冲区。
 *
 * 输出对于同一拓扑级别的所有对象是保证相同的。
 *
 * 如果verbose为1，则使用更长的类型名称，例如L1Cache而不是L1。
 *
 * 输出字符串可以通过hwloc_type_sscanf()进行解析。
 *
 * 如果size为0，则string可以安全地为NULL。
 *
 * \return 如果没有截断，则实际写入的字符数，或者将要写入的字符数（不包括结尾\0）。
 */
HWLOC_DECLSPEC int hwloc_obj_type_snprintf(char * __hwloc_restrict string, size_t size,
                       hwloc_obj_t obj,
                       int verbose);

/**
 * \brief 将给定拓扑对象的属性转换为人类可读的形式。
 *
 * 属性值由separator分隔。
 *
 * 非详细模式下只打印主要属性。
 *
 * 如果size为0，则string可以安全地为NULL。
 *
 * \return 如果没有截断，则实际写入的字符数，或者将要写入的字符数（不包括结尾\0）。
 */
HWLOC_DECLSPEC int hwloc_obj_attr_snprintf(char * __hwloc_restrict string, size_t size,
                       hwloc_obj_t obj, const char * __hwloc_restrict separator,
                       int verbose);
/**
 * @brief 从类型字符串中返回对象类型和属性。
 *
 * 将字符串（例如"Package"或"L1iCache"）转换为相应的类型。
 * 匹配不区分大小写，实际上只需要匹配第一个字母。
 *
 * 匹配的对象类型设置在 typep 中（不能为 NULL）。
 *
 * 类型特定的属性，例如缓存类型、缓存深度、组深度、桥类型或操作系统设备类型，可以在 attrp 中返回。
 * 未在字符串中指定的属性（例如没有深度的"Group"，或没有缓存类型的"L2Cache"）设置为-1。
 *
 * 只有在 attrp 不为 NULL 且其大小在 attrsize 中指定的足够大时，才会填充 attrp。
 * 它应至少与 union hwloc_obj_attr_u 一样大。
 *
 * 如果正确识别了类型，则返回0，否则返回-1。
 *
 * 注意：此函数保证匹配由 hwloc_obj_type_string() 或 hwloc_obj_type_snprintf() 返回的任何字符串。
 *
 * 注意：这是现在已弃用的 hwloc_obj_type_sscanf() 的扩展版本。
 */
HWLOC_DECLSPEC int hwloc_type_sscanf(const char *string,
                     hwloc_obj_type_t *typep,
                     union hwloc_obj_attr_u *attrp, size_t attrsize);
/**
 * \brief 从类型字符串中返回对象类型和其级别深度。
 *
 * 将诸如"Package"或"L1iCache"之类的字符串转换为相应的类型，并在\p depthp中返回相应级别在拓扑\p topology中的深度。
 *
 * 如果底层架构中不存在此类型的对象，则返回::HWLOC_TYPE_DEPTH_UNKNOWN。
 *
 * 如果存在多个这样的级别（例如如果给定Group而没有任何深度），则该函数可能返回::HWLOC_TYPE_DEPTH_MULTIPLE。
 *
 * 如果\p typep非\c NULL，则匹配的对象类型将设置在\p typep中。
 *
 * \note 此函数类似于hwloc_type_sscanf()后跟hwloc_get_type_depth()，但它还自动消除多个组级别等。
 *
 * \note 此函数保证匹配由hwloc_obj_type_string()或hwloc_obj_type_snprintf()返回的任何字符串。
 */
HWLOC_DECLSPEC int hwloc_type_sscanf_as_depth(const char *string,
                          hwloc_obj_type_t *typep,
                          hwloc_topology_t topology, int *depthp);

/** @} */

/** \defgroup hwlocality_info_attr 咨询和添加键-值信息属性
 *
 * @{
 */

/** \brief 在对象信息中搜索给定的键名，并返回相应的值。
 *
 * 如果多个键与给定名称匹配，则只返回第一个。
 *
 * \return 如果不存在这样的键，则返回\c NULL。
 */
static __hwloc_inline const char *
hwloc_obj_get_info_by_name(hwloc_obj_t obj, const char *name) __hwloc_attribute_pure;
/**
 * \brief 将给定的信息名称和值对添加到给定对象中。
 *
 * 即使已经存在具有相同名称的键，信息也会被追加到现有的信息数组中。
 *
 * 输入字符串在添加到对象信息之前会被复制。
 *
 * \return 成功返回 \c 0，出错返回 \c -1。
 *
 * \note 可以使用此函数通过将 "lstopoStyle" 作为名称和 "Background=#rrggbb" 作为值来强制在 lstopo 图形输出中设置对象颜色。有关详细信息，请参阅 lstopo(1) 手册中的 CUSTOM COLORS 部分。
 *
 * \note 如果 \p value 包含一些不可打印的字符，在导出到 XML 时，它们将被丢弃，请参阅 hwloc_topology_export_xml() 中的 hwloc/export.h。
 */
HWLOC_DECLSPEC int hwloc_obj_add_info(hwloc_obj_t obj, const char *name, const char *value);

/** @} */

/**
 * \brief 处理/线程绑定标志。
 *
 * 这些位标志可用于细化绑定策略。
 *
 * 默认值（0）是绑定当前进程，假定为单线程，以非严格方式绑定。这是最具可移植性的绑定方式，因为所有操作系统通常都提供它。
 *
 * \note 并非所有系统都支持所有类型的绑定。请参阅 \ref hwlocality_cpubinding 的 "Detailed Description" 部分，了解可能发生的错误描述。
 */
} hwloc_cpubind_flags_t;

/**
 * \brief 在物理位图 \p set 中给定的 CPU 上绑定当前进程或线程。
 *
 * \return 如果不支持该操作，则返回 -1，并将 errno 设置为 ENOSYS
 * \return 如果无法强制执行绑定，则返回 -1，并将 errno 设置为 EXDEV
 */
HWLOC_DECLSPEC int hwloc_set_cpubind(hwloc_topology_t topology, hwloc_const_cpuset_t set, int flags);

/**
 * \brief 获取当前进程或线程的绑定。
 *
 * CPU 集 \p set（之前由调用者分配）将填充为进程或线程（根据 \e flags）最后绑定到的 PU 列表。
 */
HWLOC_DECLSPEC int hwloc_get_cpubind(hwloc_topology_t topology, hwloc_cpuset_t set, int flags);
/** \brief 绑定进程 \p pid 到物理位图 \p set 中的 CPU 上。
 *
 * \note 在 Unix 平台上，\p hwloc_pid_t 是 \p pid_t，
 * 在本机 Windows 平台上，\p hwloc_pid_t 是 \p HANDLE。
 *
 * \note 在 Linux 上的一个特殊情况是，如果提供了 tid（线程 ID）而不是 pid（进程 ID），
 * 并且在 flags 中传递了 ::HWLOC_CPUBIND_THREAD，那么绑定将应用于该特定线程。
 *
 * \note 在非 Linux 系统上，::HWLOC_CPUBIND_THREAD 不能在 \p flags 中使用。
 */
HWLOC_DECLSPEC int hwloc_set_proc_cpubind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_cpuset_t set, int flags);

/** \brief 获取进程 \p pid 的当前物理绑定。
 *
 * CPU 集合 \p set（之前由调用者分配）将填充该进程最后绑定到的 PU 列表。
 *
 * \note 在 Unix 平台上，\p hwloc_pid_t 是 \p pid_t，
 * 在本机 Windows 平台上，\p hwloc_pid_t 是 \p HANDLE。
 *
 * \note 在 Linux 上的一个特殊情况是，如果提供了 tid（线程 ID）而不是 pid（进程 ID），
 * 并且在 flags 中传递了 HWLOC_CPUBIND_THREAD，那么将返回该特定线程的绑定。
 *
 * \note 在非 Linux 系统上，HWLOC_CPUBIND_THREAD 不能在 \p flags 中使用。
 */
HWLOC_DECLSPEC int hwloc_get_proc_cpubind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_cpuset_t set, int flags);

#ifdef hwloc_thread_t
/** \brief 绑定线程 \p thread 到物理位图 \p set 中的 CPU 上。
 *
 * \note 在 Unix 平台上，\p hwloc_thread_t 是 \p pthread_t，
 * 在本机 Windows 平台上，\p hwloc_thread_t 是 \p HANDLE。
 *
 * \note ::HWLOC_CPUBIND_PROCESS 不能在 \p flags 中使用。
 */
HWLOC_DECLSPEC int hwloc_set_thread_cpubind(hwloc_topology_t topology, hwloc_thread_t thread, hwloc_const_cpuset_t set, int flags);
#endif

#ifdef hwloc_thread_t
/** \brief 获取线程 \p tid 的当前物理绑定。
 *
 * CPU 集合 \p set（之前由调用者分配）将填充线程最后绑定到的处理器单元（PU）列表。
 *
 * \note \p hwloc_thread_t 在 Unix 平台上是 \p pthread_t，在本机 Windows 平台上是 \p HANDLE。
 *
 * \note 不能在 \p flags 中使用 ::HWLOC_CPUBIND_PROCESS。
 */
HWLOC_DECLSPEC int hwloc_get_thread_cpubind(hwloc_topology_t topology, hwloc_thread_t thread, hwloc_cpuset_t set, int flags);
#endif

/** \brief 获取当前进程或线程上次运行的最后一个物理 CPU。
 *
 * CPU 集合 \p set（之前由调用者分配）将填充进程或线程（根据 \e flags）上次运行的处理器单元（PU）列表。
 *
 * 操作系统可能会根据它们的绑定将一些任务从一个处理器移动到另一个处理器，因此该函数可能返回已经过时的内容。
 *
 * \p flags 可以包括 ::HWLOC_CPUBIND_PROCESS 或 ::HWLOC_CPUBIND_THREAD，以指定查询是针对整个进程（所有线程运行的所有 CPU 的并集）还是仅针对当前线程。如果进程是单线程的，则可以将 flags 设置为零，以让 hwloc 使用底层操作系统上可用的任何方法。
 */
HWLOC_DECLSPEC int hwloc_get_last_cpu_location(hwloc_topology_t topology, hwloc_cpuset_t set, int flags);
/**
 * 获取进程最后运行的物理 CPU。
 *
 * 参数 \p set（由调用者预先分配）将填充进程最后运行的处理器单元（PU）列表。
 *
 * 操作系统可能会根据它们的绑定随时将一些任务从一个处理器移动到另一个处理器，因此该函数可能返回已经过时的信息。
 *
 * \note 在 Unix 平台上，\p hwloc_pid_t 是 \p pid_t，在本机 Windows 平台上是 \p HANDLE。
 *
 * \note 在 Linux 上的一个特殊情况是，如果提供了 tid（线程 ID）而不是 pid（进程 ID），并且在 flags 中传递了 ::HWLOC_CPUBIND_THREAD，则返回该特定线程的最后 CPU 位置。
 *
 * \note 在非 Linux 系统上，无法在 \p flags 中使用 ::HWLOC_CPUBIND_THREAD。
 */
HWLOC_DECLSPEC int hwloc_get_proc_last_cpu_location(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_cpuset_t set, int flags);

/** @} */

/** \brief 内存绑定策略。
 *
 * 这些常量可用于选择绑定策略。一次只能使用一种策略（即，值不能进行 OR 运算）。
 *
 * 并非所有系统都支持所有类型的绑定。
 * 可以使用 hwloc_topology_get_support() 查询当前操作系统中实际的内存绑定策略支持情况。
 * 有关可能发生的错误的描述，请参阅 \ref hwlocality_membinding 的 "详细说明" 部分。
 */
} hwloc_membind_policy_t;
/**
 * 内存绑定标志。
 *
 * 这些标志可用于细化绑定策略。
 * 所有标志都可以与逻辑 OR 运算符一起使用，除了
 * ::HWLOC_MEMBIND_PROCESS 和 ::HWLOC_MEMBIND_THREAD；
 * 这两个标志是互斥的。
 *
 * 并非所有系统都支持所有类型的绑定。
 * 可以使用 hwloc_topology_get_support() 查询当前操作系统中实际的内存绑定支持情况。
 * 有关可能发生的错误的描述，请参阅 \ref hwlocality_membinding 的“详细说明”部分。
 */
typedef enum {
  /** \brief 设置指定（可能是多线程的）进程的所有线程的策略。此标志与 ::HWLOC_MEMBIND_THREAD 互斥。
   * \hideinitializer */
  HWLOC_MEMBIND_PROCESS =       (1<<0),

 /** \brief 设置当前进程的特定线程的策略。此标志与 ::HWLOC_MEMBIND_PROCESS 互斥。
  * \hideinitializer */
  HWLOC_MEMBIND_THREAD =        (1<<1),

 /** 请求严格绑定。如果绑定不能保证/完全执行，函数将失败。
  *
  * 此标志在使用不同函数时具有略微不同的含义。
  * \hideinitializer  */
  HWLOC_MEMBIND_STRICT =        (1<<2),

 /** \brief 迁移已分配的内存。如果内存无法迁移并且传递了 ::HWLOC_MEMBIND_STRICT 标志，将返回错误。
  * \hideinitializer  */
  HWLOC_MEMBIND_MIGRATE =       (1<<3),

  /** \brief 避免对 CPU 绑定产生任何影响。
   *
   * 在某些操作系统上，一些底层内存绑定函数还会将应用程序绑定到相应的 CPU(s)。
   * 使用此标志将导致 hwloc 避免使用可能会影响 CPU 绑定的 OS 函数。但是请注意，使用 NOCPUBIND 可能会减少 hwloc 的整体内存绑定支持。具体来说：在使用 NOCPUBIND 时，一些 hwloc 的内存绑定函数可能会因 errno 设置为 ENOSYS 而失败。
   * \hideinitializer
   */
  HWLOC_MEMBIND_NOCPUBIND =     (1<<4),

  /** \brief 将位图参数视为节点集。
   *
   * 如果传递了此标志，则位图参数被视为节点集；否则，默认情况下被视为 CPU 集。
   *
   * 对于无 CPU 的 NUMA 内存节点，使用 CPU 集进行内存绑定是行不通的。因此，应尽可能优先使用节点集进行绑定。
   * \hideinitializer
   */
  HWLOC_MEMBIND_BYNODESET =     (1<<5)
} hwloc_membind_flags_t;
/**
 * \brief 将当前进程或线程的默认内存绑定策略设置为优先使用指定的NUMA节点
 *
 * 如果未指定 ::HWLOC_MEMBIND_PROCESS 或 ::HWLOC_MEMBIND_THREAD，则假定当前进程是单线程的。
 * 这是最具可移植性的形式，因为它允许hwloc使用基于进程的操作系统函数或基于线程的操作系统函数，具体取决于哪种函数可用。
 *
 * 如果指定了 ::HWLOC_MEMBIND_BYNODESET，则set被视为节点集。否则它是一个cpuset。
 *
 * \return 如果不支持该操作，则返回-1，并将errno设置为ENOSYS
 * \return 如果无法强制执行绑定，则返回-1，并将errno设置为EXDEV
 */
HWLOC_DECLSPEC int hwloc_set_membind(hwloc_topology_t topology, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags);

/**
 * \brief 将指定进程的默认内存绑定策略设置为优先使用指定的NUMA节点
 *
 * 如果指定了 ::HWLOC_MEMBIND_BYNODESET，则set被视为节点集。否则它是一个cpuset。
 *
 * \return 如果不支持该操作，则返回-1，并将errno设置为ENOSYS
 * \return 如果无法强制执行绑定，则返回-1，并将errno设置为EXDEV
 *
 * \note 在Unix平台上，hwloc_pid_t是pid_t，在本机Windows平台上是HANDLE。
 */
HWLOC_DECLSPEC int hwloc_set_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags);
/**
 * 查询指定进程的默认内存绑定策略和物理局部性。
 *
 * 位图 \p set（由调用者预先分配）将填充进程的内存绑定。
 *
 * 此函数有两个输出参数：\p set 和 \p policy。
 * 这些参数中返回的值取决于传入的 \p flags 和查询目标中当前的内存绑定策略和节点集。
 *
 * 传递 ::HWLOC_MEMBIND_PROCESS 标志指定查询目标是指定进程中所有线程的当前策略和节点集。
 * 如果未指定 ::HWLOC_MEMBIND_PROCESS（这是最可移植的方法），则假定进程是单线程的。
 * 这允许 hwloc 使用基于进程的操作系统函数或基于线程的操作系统函数，具体取决于哪个可用。
 *
 * 请注意，向此函数传递 ::HWLOC_MEMBIND_THREAD 是没有意义的。
 *
 * 如果指定了 ::HWLOC_MEMBIND_STRICT，则 hwloc 将检查指定进程中所有线程的默认内存策略和节点集。
 * 如果它们不相同，则返回 -1，并将 errno 设置为 EXDEV。
 * 如果它们相同，则将值返回到 \p set 和 \p policy 中。
 *
 * 否则，\p set 被设置为所有线程默认集合的逻辑 OR。
 * 如果所有线程的默认策略相同，则将 \p policy 设置为该策略。
 * 如果它们不同，则将 \p policy 设置为 ::HWLOC_MEMBIND_MIXED。
 *
 * 如果指定了 ::HWLOC_MEMBIND_BYNODESET，则将 set 视为节点集。
 * 否则它是一个 cpuset。
 *
 * 如果指定了任何其他标志，则返回 -1，并将 errno 设置为 EINVAL。
 *
 * \note 在 Unix 平台上，\p hwloc_pid_t 是 \p pid_t，在本机 Windows 平台上是 \p HANDLE。
 */
HWLOC_DECLSPEC int hwloc_get_proc_membind(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags);
/**
 * \brief 将已分配的内存（由addr和len标识）绑定到指定的NUMA节点
 * 
 * 如果指定了::HWLOC_MEMBIND_BYNODESET，set被视为节点集。否则它是一个cpuset。
 * 
 * \return 如果len为0，则返回0。
 * \return 如果不支持该操作，则返回-1，并将errno设置为ENOSYS。
 * \return 如果无法强制执行绑定，则返回-1，并将errno设置为EXDEV。
 */
HWLOC_DECLSPEC int hwloc_set_area_membind(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags);

/**
 * \brief 查询物理NUMA节点附近的CPU和内存（由addr和len标识）的绑定策略
 * 
 * 由调用者预先分配的位图set将填充内存区域的绑定情况。
 * 
 * 该函数有两个输出参数：set和policy。这些参数中返回的值取决于传入的flags和地址范围内页面的内存绑定策略和节点集。
 * 
 * 如果指定了::HWLOC_MEMBIND_STRICT，首先检查目标页面是否都具有相同的内存绑定策略和节点集。如果它们不相同，则返回-1，并将errno设置为EXDEV。如果它们在所有页面上都相同，则将set和policy分别返回到set和policy中。
 * 
 * 如果未指定::HWLOC_MEMBIND_STRICT，则计算地址范围内包含页面的所有NUMA节点的并集。如果目标中的所有页面都具有相同的策略，则将其返回到policy中。否则，将policy设置为::HWLOC_MEMBIND_MIXED。
 * 
 * 如果指定了::HWLOC_MEMBIND_BYNODESET，set被视为节点集。否则它是一个cpuset。
 * 
 * 如果指定了任何其他标志，则返回-1，并将errno设置为EINVAL。
 * 
 * 如果len为0，则返回-1，并将errno设置为EINVAL。
 */
# 获取内存绑定的 NUMA 节点
HWLOC_DECLSPEC int hwloc_get_area_membind(hwloc_topology_t topology, const void *addr, size_t len, hwloc_bitmap_t set, hwloc_membind_policy_t * policy, int flags);

/** \brief 获取内存区域的物理分配位置的 NUMA 节点
 *
 * 根据内存区域的地址和长度，填充位图 \p set，表示内存区域页面所在的 NUMA 节点。
 * 如果没有页面被分配，\p set 可能为空。
 *
 * 如果页面分布在多个节点上，不保证它们分布均匀，或者大部分页面在单个节点上等。
 *
 * 操作系统可能根据绑定将内存页面从一个处理器移动到另一个处理器，因此该函数可能返回已经过时的信息。
 *
 * 如果在 \p flags 中指定了 ::HWLOC_MEMBIND_BYNODESET，set 被视为节点集。否则它是一个 CPU 集。
 *
 * 如果 \p len 为 0，\p set 被清空。
 */
HWLOC_DECLSPEC int hwloc_get_area_memlocation(hwloc_topology_t topology, const void *addr, size_t len, hwloc_bitmap_t set, int flags);

/** \brief 分配一些内存
 *
 * 这相当于 malloc()，不同之处在于它尝试从操作系统分配页面对齐的内存。
 *
 * \note 分配的内存应该使用 hwloc_free() 释放。
 */
HWLOC_DECLSPEC void *hwloc_alloc(hwloc_topology_t topology, size_t len);

/** \brief 在由 \p set 指定的 NUMA 内存节点上分配一些内存
 *
 * 如果操作不支持并且给定了 ::HWLOC_MEMBIND_STRICT，则返回 NULL 并将 errno 设置为 ENOSYS。
 * 如果绑定无法强制执行并且给定了 ::HWLOC_MEMBIND_STRICT，则返回 NULL 并将 errno 设置为 EXDEV。
 * 如果内存分配失败甚至在尝试绑定之前，则返回 NULL 并将 errno 设置为 ENOMEM。
 *
 * 如果在 \p flags 中指定了 ::HWLOC_MEMBIND_BYNODESET，则 set 被视为节点集。否则它是一个 CPU 集。
 *
 * \note 分配的内存应该使用 hwloc_free() 释放。
 */
# 分配内存并将其绑定到指定的 NUMA 内存节点
HWLOC_DECLSPEC void *hwloc_alloc_membind(hwloc_topology_t topology, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_malloc;

/** \brief 在由 \p set 指定的 NUMA 内存节点上分配一些内存
 *
 * 首先，尝试使用 hwloc_alloc_membind() 正确分配内存。
 * 失败时，会在分配内存之前使用 hwloc_set_membind() 更改当前进程或线程的内存绑定策略。
 * 因此，这个函数在更多情况下都能工作，但会改变当前状态（可能会影响未指定任何策略的未来分配）。
 *
 * 如果指定了 ::HWLOC_MEMBIND_BYNODESET，则将 set 视为节点集。
 * 否则它是一个 cpuset。
 */
static __hwloc_inline void *
hwloc_alloc_membind_policy(hwloc_topology_t topology, size_t len, hwloc_const_bitmap_t set, hwloc_membind_policy_t policy, int flags) __hwloc_attribute_malloc;

/** \brief 释放先前由 hwloc_alloc() 或 hwloc_alloc_membind() 分配的内存
 */
HWLOC_DECLSPEC int hwloc_free(hwloc_topology_t topology, void *addr, size_t len);

/** @} */



/** \defgroup hwlocality_setsource 更改拓扑发现的来源
 *
 * 如果没有调用下面的任何函数，则默认情况下是检测调用者被允许访问的机器的所有对象。
 *
 * 如果应用程序尚未修改默认行为，则还可以通过环境变量进行修改。
 * 在环境中设置 HWLOC_XMLFILE 强制从 XML 文件中发现拓扑，就好像调用了 hwloc_topology_set_xml() 一样。
 * 设置 HWLOC_SYNTHETIC 强制使用合成拓扑，就好像调用了 hwloc_topology_set_synthetic() 一样。
 *
 * 最后，HWLOC_THISSYSTEM 强制返回 hwloc_topology_is_thissystem() 的返回值。
 *
 * @{
 */
# 更改拓扑视图的进程
#
# 在某些系统上，进程可能对机器有不同的视图，例如允许的 CPU 集合。默认情况下，hwloc 显示当前进程的视图。
# 调用 hwloc_topology_set_pid() 允许它显示来自另一个进程视图的机器拓扑。
#
# 注意：在 Unix 平台上，hwloc_pid_t 是 pid_t，在本机 Windows 平台上是 HANDLE。
#
# 注意：在不支持此功能的平台上，返回 -1 并将 errno 设置为 ENOSYS。
HWLOC_DECLSPEC int hwloc_topology_set_pid(hwloc_topology_t __hwloc_restrict topology, hwloc_pid_t pid);

# 启用合成拓扑
#
# 从给定的描述中收集拓扑信息，描述是一个空格分隔的字符串，描述每个级别的对象类型和度。
# 所有类型都可以省略（空格分隔的数字字符串），以便 hwloc 根据通常的拓扑选择所有类型。
# 也可以设置环境变量 HWLOC_SYNTHETIC 以获得此行为。
#
# 如果描述被正确解析并描述了有效的拓扑配置，则此函数返回 0。
# 否则返回 -1 并将 errno 设置为 EINVAL。
#
# 注意，此函数实际上并不加载拓扑信息；它只告诉 hwloc 从哪里加载它。您仍然需要调用 hwloc_topology_load() 来实际加载拓扑信息。
#
# 注意：为了方便起见，此后端提供了空绑定钩子，它只返回成功。
#
# 注意：成功后，合成组件将替换先前启用的组件（如果有），但拓扑直到 hwloc_topology_load() 才会实际修改。
HWLOC_DECLSPEC int hwloc_topology_set_synthetic(hwloc_topology_t __hwloc_restrict topology, const char * __hwloc_restrict description);
/**
 * \brief 启用基于 XML 文件的拓扑结构。
 *
 * 从给定的 \p xmlpath 参数中获取 XML 文件中的拓扑信息。
 * 设置环境变量 HWLOC_XMLFILE 也可以实现这种行为。
 * 这个文件可以是之前使用 hwloc/export.h 中的 hwloc_topology_export_xml() 或者 lstopo file.xml 生成的。
 *
 * 注意，这个函数实际上并不加载拓扑信息；它只是告诉 hwloc 从哪里加载拓扑信息。
 * 你仍然需要调用 hwloc_topology_load() 来实际加载拓扑信息。
 *
 * \return 失败时返回 -1，并设置 errno 为 EINVAL。
 *
 * \note 还可以参考 hwloc_topology_set_userdata_import_callback() 来导入特定于应用程序的对象用户数据。
 *
 * \note 为了方便起见，这个后端提供了空的绑定钩子，只是返回成功。
 * 要让 hwloc 仍然调用特定于操作系统的钩子，必须设置 ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM 来断言加载的文件确实是底层系统。
 *
 * \note 成功时，XML 组件会替换先前启用的组件（如果有），但拓扑结构实际上直到 hwloc_topology_load() 才会被修改。
 */
HWLOC_DECLSPEC int hwloc_topology_set_xml(hwloc_topology_t __hwloc_restrict topology, const char * __hwloc_restrict xmlpath);
# 启用基于 XML 的拓扑结构，使用内存缓冲区（而不是文件，类似于 hwloc_topology_set_xml()）。
# 从给定的内存缓冲区 buffer 中收集拓扑信息，长度为 size。该缓冲区可以在之前使用 hwloc_topology_export_xmlbuffer() 在 hwloc/export.h 中填充。
# 注意，这个函数实际上并不加载拓扑信息；它只是告诉 hwloc 从哪里加载。你仍然需要调用 hwloc_topology_load() 来实际加载拓扑信息。
# 在失败时返回 -1，并设置 errno 为 EINVAL，表示无法读取 XML 缓冲区。
# 请参见 hwloc_topology_set_userdata_import_callback() 以导入特定于应用程序的对象用户数据。
# 为了方便起见，这个后端提供了空的绑定钩子，只是返回成功。为了让 hwloc 实际调用特定于操作系统的钩子，必须设置 ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM 来断言加载的文件确实是底层系统。
# 成功时，XML 组件会替换先前启用的组件（如果有），但拓扑结构直到 hwloc_topology_load() 才会实际修改。
HWLOC_DECLSPEC int hwloc_topology_set_xmlbuffer(hwloc_topology_t __hwloc_restrict topology, const char * __hwloc_restrict buffer, int size);

# 传递给 hwloc_topology_set_components() 的标志。
enum hwloc_topology_components_flag_e {
  # 从使用中黑名单目标组件。
  HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST = (1UL<<0)
};
/**
 * \brief 阻止发现组件用于拓扑结构。
 *
 * \p name 是不应该在加载拓扑结构 \p topology 时使用的发现组件的名称。名称是一个字符串，比如 "cuda"。
 *
 * 对于具有多个阶段的组件，还可以在后面加上阶段的名称，例如 "linux:io"。
 *
 * \p flags 应该是 ::HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST。
 *
 * 这可以用于避免发现过程中的昂贵部分。
 * 例如，CUDA 特定的发现可能是昂贵且不需要的，而通用 I/O 发现仍然可能是有用的。
 */
HWLOC_DECLSPEC int hwloc_topology_set_components(hwloc_topology_t __hwloc_restrict topology, unsigned long flags, const char * __hwloc_restrict name);

/** @} */

/** \defgroup hwlocality_configuration 拓扑检测配置和查询
 *
 * 在 hwloc_topology_init() 和 hwloc_topology_load() 之间可以选择调用几个函数来配置检测的方式，
 * 例如忽略某些对象类型，定义一个合成的拓扑结构等。
 *
 * @{
 */

/** \brief 要设置到未加载的拓扑上的标志。
 *
 * 标志应该通过 hwloc_topology_set_flags() 设置。
 * 它们也可以通过 hwloc_topology_get_flags() 返回。
 */
};

/** \brief 设置 OR 运算后的标志到尚未加载的拓扑上。
 *
 * 将一个 OR 运算后的 ::hwloc_topology_flags_e 集合设置到尚未加载的拓扑上。
 *
 * 如果多次调用此函数，最后一次调用将擦除并替换先前设置的标志集合。
 *
 * 默认情况下，没有设置任何标志（\c 0）。
 *
 * 可以使用 hwloc_topology_get_flags() 获取拓扑中设置的标志。
 */
HWLOC_DECLSPEC int hwloc_topology_set_flags (hwloc_topology_t topology, unsigned long flags);
# 获取拓扑结构的 OR 运算标志
#
# 获取拓扑结构的 ::hwloc_topology_flags_e 的 OR 运算集合。
#
# 如果之前没有调用 hwloc_topology_set_flags()，则不设置任何标志（返回 \c 0）。
#
# \return 之前使用 hwloc_topology_set_flags() 设置的标志。
HWLOC_DECLSPEC unsigned long hwloc_topology_get_flags (hwloc_topology_t topology);

# 拓扑结构上下文是否来自该系统？
#
# 如果此拓扑上下文是使用运行此程序的系统构建的，则返回 1。
# 否则返回 0（例如，如果使用另一个文件系统根目录、XML 拓扑文件或合成拓扑）。
#
# \return 如果此拓扑上下文是使用运行此程序的系统构建的，则返回 1。
# 否则返回 0（例如，如果使用另一个文件系统根目录、XML 拓扑文件或合成拓扑）。
HWLOC_DECLSPEC int hwloc_topology_is_thissystem(hwloc_topology_t  __hwloc_restrict topology) __hwloc_attribute_pure;

# 描述此拓扑的实际发现支持的标志。
struct hwloc_topology_discovery_support {
  # 检测 PU 对象的数量是否受支持。
  unsigned char pu;
  # 检测 NUMA 节点的数量是否受支持。
  unsigned char numa;
  # 检测 NUMA 节点中的内存量是否受支持。
  unsigned char numa_memory;
  # 检测和识别对当前进程不可用的 PU 对象是否受支持。
  unsigned char disallowed_pu;
  # 检测和识别对当前进程不可用的 NUMA 节点是否受支持。
  unsigned char disallowed_numa;
  # 支持检测 CPU 种类的效率，参见 \ref hwlocality_cpukinds。
  unsigned char cpukind_efficiency;
};

# 描述此拓扑的实际 PU 绑定支持的标志。
#
# 即使在所有情况下不支持该功能（例如，绑定到不连续对象的随机集合），也可能设置标志。
# 定义了一个结构体，用于描述处理器绑定的支持情况
struct hwloc_topology_cpubind_support {
  /** 是否支持绑定整个当前进程 */
  unsigned char set_thisproc_cpubind;
  /** 是否支持获取整个当前进程的绑定情况 */
  unsigned char get_thisproc_cpubind;
  /** 是否支持绑定整个指定进程 */
  unsigned char set_proc_cpubind;
  /** 是否支持获取整个指定进程的绑定情况 */
  unsigned char get_proc_cpubind;
  /** 是否支持绑定当前线程 */
  unsigned char set_thisthread_cpubind;
  /** 是否支持获取当前线程的绑定情况 */
  unsigned char get_thisthread_cpubind;
  /** 是否支持绑定指定线程 */
  unsigned char set_thread_cpubind;
  /** 是否支持获取指定线程的绑定情况 */
  unsigned char get_thread_cpubind;
  /** 是否支持获取整个当前进程最后运行的处理器 */
  unsigned char get_thisproc_last_cpu_location;
  /** 是否支持获取整个指定进程最后运行的处理器 */
  unsigned char get_proc_last_cpu_location;
  /** 是否支持获取当前线程最后运行的处理器 */
  unsigned char get_thisthread_last_cpu_location;
};

/** \brief 描述此拓扑结构的实际内存绑定支持的标志。
 *
 * 即使在某些情况下不支持该功能，也可能设置标志（例如绑定到不连续对象的随机集合）。
 */
# 定义了一个结构体，用于描述内存绑定支持的情况
struct hwloc_topology_membind_support {
  /** Binding the whole current process is supported.  */
  unsigned char set_thisproc_membind;  // 是否支持将整个当前进程绑定到内存
  /** Getting the binding of the whole current process is supported.  */
  unsigned char get_thisproc_membind;  // 是否支持获取整个当前进程的内存绑定
  /** Binding a whole given process is supported.  */
  unsigned char set_proc_membind;  // 是否支持将给定进程绑定到内存
  /** Getting the binding of a whole given process is supported.  */
  unsigned char get_proc_membind;  // 是否支持获取给定进程的内存绑定
  /** Binding the current thread only is supported.  */
  unsigned char set_thisthread_membind;  // 是否支持将当前线程绑定到内存
  /** Getting the binding of the current thread only is supported.  */
  unsigned char get_thisthread_membind;  // 是否支持获取当前线程的内存绑定
  /** Binding a given memory area is supported. */
  unsigned char set_area_membind;  // 是否支持将给定内存区域绑定到内存
  /** Getting the binding of a given memory area is supported.  */
  unsigned char get_area_membind;  // 是否支持获取给定内存区域的内存绑定
  /** Allocating a bound memory area is supported. */
  unsigned char alloc_membind;  // 是否支持分配绑定的内存区域
  /** First-touch policy is supported. */
  unsigned char firsttouch_membind;  // 是否支持首次访问策略
  /** Bind policy is supported. */
  unsigned char bind_membind;  // 是否支持绑定策略
  /** Interleave policy is supported. */
  unsigned char interleave_membind;  // 是否支持交错策略
  /** Next-touch migration policy is supported. */
  unsigned char nexttouch_membind;  // 是否支持下次访问迁移策略
  /** Migration flags is supported. */
  unsigned char migrate_membind;  // 是否支持迁移标志
  /** Getting the last NUMA nodes where a memory area was allocated is supported */
  unsigned char get_area_memlocation;  // 是否支持获取分配内存区域的最后NUMA节点
};

/** \brief Flags describing miscellaneous features.
 */
struct hwloc_topology_misc_support {
  /** Support was imported when importing another topology, see ::HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT. */
  unsigned char imported_support;  // 是否在导入另一个拓扑结构时支持导入
};

/** \brief Set of flags describing actual support for this topology.
 *
 * This is retrieved with hwloc_topology_get_support() and will be valid until
 * the topology object is destroyed.  Note: the values are correct only after
 * discovery.
 */
# 定义了一个结构体，包含了用于描述拓扑支持的各种特性的指针
struct hwloc_topology_support {
  struct hwloc_topology_discovery_support *discovery;  // 用于描述拓扑发现支持的指针
  struct hwloc_topology_cpubind_support *cpubind;  // 用于描述 CPU 绑定支持的指针
  struct hwloc_topology_membind_support *membind;  // 用于描述内存绑定支持的指针
  struct hwloc_topology_misc_support *misc;  // 用于描述其他支持的指针
};

/** \brief Retrieve the topology support.
 *
 * 检索拓扑支持的信息
 * 每个标志指示一个特性是否受支持。
 * 如果设置为 0，则该特性不受支持。
 * 如果设置为 1，则该特性受支持，但相应的调用在某些极端情况下仍可能失败。
 *
 * 这些特性也可以通过 hwloc-info \--support 列出
 *
 * 报告的特性是当前拓扑在当前机器上支持的。
 * 如果拓扑是从另一台机器导出并在此处导入的，则支持仍描述了导入后此导入拓扑支持的内容。
 * 在这种情况下，默认情况下，绑定将报告为不受支持（参见::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM）。
 *
 * 拓扑标志::HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT 可用于报告原始远程机器的支持特性。
 * 如果成功导入，\p imported_support 将在 struct hwloc_topology_misc_support 数组中设置。
 */
HWLOC_DECLSPEC const struct hwloc_topology_support *hwloc_topology_get_support(hwloc_topology_t __hwloc_restrict topology);

/** \brief Type filtering flags.
 *
 * 类型过滤标志。
 * 默认情况下，大多数对象都保留（::HWLOC_TYPE_FILTER_KEEP_ALL）。
 * 指令缓存、I/O 和其他对象默认情况下被忽略（::HWLOC_TYPE_FILTER_KEEP_NONE）。
 * Die 和 Group 级别默认情况下被忽略，除非它们带来结构（::HWLOC_TYPE_FILTER_KEEP_STRUCTURE）。
 *
 * 请注意，当组对象不带来结构时，它们也会被单独忽略（而不是整个级别）。
 */
# 定义枚举类型 hwloc_type_filter_e，用于表示对象类型过滤器
enum hwloc_type_filter_e {
  /** \brief 保留该类型的所有对象。
   *
   * 不能用于 ::HWLOC_OBJ_GROUP（组仅用于为拓扑结构添加更多结构）。
   * \hideinitializer
   */
  HWLOC_TYPE_FILTER_KEEP_ALL = 0,

  /** \brief 忽略该类型的所有对象。
   *
   * 不能忽略底层类型 ::HWLOC_OBJ_PU，::HWLOC_OBJ_NUMANODE 类型和
   * 顶层类型 ::HWLOC_OBJ_MACHINE。
   * \hideinitializer
   */
  HWLOC_TYPE_FILTER_KEEP_NONE = 1,

  /** \brief 仅在整个级别不提供任何结构时忽略对象。
   *
   * 如果级别中的至少一个对象为拓扑结构添加了结构，则保留整个级别的对象。
   * 当一个对象有多个子对象并且它不是其父对象的唯一子对象时，该对象添加了结构。
   *
   * 如果级别中的所有对象都是其父对象的唯一子对象，并且没有一个对象有多个子对象，则删除整个级别。
   *
   * 不能用于 I/O 和 Misc 对象，因为在那里拓扑结构并不重要。
   * \hideinitializer
   */
  HWLOC_TYPE_FILTER_KEEP_STRUCTURE = 2,

  /** \brief 仅保留给定类型的可能重要对象。
   *
   * 仅对 I/O 对象类型有用。
   * 对于 ::HWLOC_OBJ_PCI_DEVICE 和 ::HWLOC_OBJ_OS_DEVICE，意味着仅保留主要/常见类型的对象（存储、网络、OpenFabrics、CUDA、OpenCL、RSMI、NVML 和显示）。
   * 此外，仅报告直接连接在 PCI 上的 OS 设备（例如，没有 USB）。
   * 对于 ::HWLOC_OBJ_BRIDGE，意味着仅在其具有子对象时保留桥接器。
   *
   * 对于 Normal、Memory 和 Misc 类型，此标志相当于 ::HWLOC_TYPE_FILTER_KEEP_ALL，因为它们可能很重要。
   * \hideinitializer
   */
  HWLOC_TYPE_FILTER_KEEP_IMPORTANT = 3
};

/** \brief 为给定对象类型设置过滤器。
 */
# 设置给定对象类型的过滤器
HWLOC_DECLSPEC int hwloc_topology_set_type_filter(hwloc_topology_t topology, hwloc_obj_type_t type, enum hwloc_type_filter_e filter);

/** \brief 获取给定对象类型的当前过滤器设置
 */
HWLOC_DECLSPEC int hwloc_topology_get_type_filter(hwloc_topology_t topology, hwloc_obj_type_t type, enum hwloc_type_filter_e *filter);

/** \brief 设置所有对象类型的过滤器
 *
 * 如果某些类型不支持此过滤器，则会被静默忽略。
 */
HWLOC_DECLSPEC int hwloc_topology_set_all_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter);

/** \brief 设置所有 CPU 缓存对象类型的过滤器
 *
 * 内存侧缓存不涉及，因为它们不是 CPU 缓存。
 */
HWLOC_DECLSPEC int hwloc_topology_set_cache_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter);

/** \brief 设置所有 CPU 指令缓存对象类型的过滤器
 *
 * 内存侧缓存不涉及，因为它们不是 CPU 缓存。
 */
HWLOC_DECLSPEC int hwloc_topology_set_icache_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter);

/** \brief 设置所有 I/O 对象类型的过滤器
 */
HWLOC_DECLSPEC int hwloc_topology_set_io_types_filter(hwloc_topology_t topology, enum hwloc_type_filter_e filter);

/** \brief 设置拓扑特定的用户数据指针
 *
 * 每个拓扑可能存储一个应用程序提供的私有数据指针。
 * 它被初始化为 \c NULL。
 * hwloc 永远不会修改它。
 *
 * 在 hwloc_topology_init() 之后和 hwloc_topolog_destroy() 之前使用它。
 *
 * 此指针不会导出到 XML。
 */
HWLOC_DECLSPEC void hwloc_topology_set_userdata(hwloc_topology_t topology, const void *userdata);

/** \brief 检索拓扑特定的用户数据指针
 *
 * 检索先前使用 hwloc_topology_set_userdata() 设置的应用程序提供的私有数据指针。
 */
HWLOC_DECLSPEC void * hwloc_topology_get_userdata(hwloc_topology_t topology);

/** @} */
# 定义枚举类型，用于指定给 hwloc_topology_restrict() 函数的标志
enum hwloc_restrict_flags_e {
  # 移除所有变为无 CPU 的对象
  # 默认情况下，只移除不包含处理器和内存的对象
  # 不能与 ::HWLOC_RESTRICT_FLAG_BYNODESET 一起使用
  HWLOC_RESTRICT_FLAG_REMOVE_CPULESS = (1UL<<0),

  # 通过节点集合而不是 CPU 集合进行限制
  # 仅保留节点集合包含或部分包含在给定集合中的对象
  # 不能与 ::HWLOC_RESTRICT_FLAG_REMOVE_CPULESS 一起使用
  HWLOC_RESTRICT_FLAG_BYNODESET =  (1UL<<3),

  # 移除所有变为无内存的对象
  # 默认情况下，只移除不包含处理器和内存的对象
  # 只能与 ::HWLOC_RESTRICT_FLAG_BYNODESET 一起使用
  HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS = (1UL<<4),

  # 如果在限制期间删除了父对象，则将 Misc 对象移动到祖先对象
  # 如果未设置此标志，则在删除父对象时删除 Misc 对象
  HWLOC_RESTRICT_FLAG_ADAPT_MISC = (1UL<<1),

  # 如果在限制期间删除了父对象，则将 I/O 对象移动到祖先对象
  # 如果未设置此标志，则在删除父对象时删除 I/O 设备和桥接
  HWLOC_RESTRICT_FLAG_ADAPT_IO = (1UL<<2)
};
# 限制拓扑结构只包含给定的 CPU 集合或节点集合
# 修改拓扑结构，移除所有不包含在 CPU 集合或节点集合中的对象，同时限制所有对象的 CPU 和节点集合
# 如果 flags 参数中包含 HWLOC_RESTRICT_FLAG_BYNODESET，则 set 被视为节点集合而不是 CPU 集合
# flags 是 hwloc_restrict_flags_e 枚举类型的按位或结果
# 注意：此调用可能无法通过将限制恢复到更大的集合来撤销。一旦在限制期间被丢弃，对象就无法恢复，除非通过 hwloc_topology_load() 加载另一个拓扑结构
# 成功返回 0
# 如果输入集合无效，则返回 -1，并将 errno 设置为 EINVAL。在这种情况下，拓扑结构不会被修改
# 如果内部数据分配失败，则返回 -1，并将 errno 设置为 ENOMEM。在这种情况下，拓扑结构将被重新初始化。应该使用 hwloc_topology_destroy() 销毁它，或者重新配置和加载
HWLOC_DECLSPEC int hwloc_topology_restrict(hwloc_topology_t __hwloc_restrict topology, hwloc_const_bitmap_t set, unsigned long flags);

# 用于传递给 hwloc_topology_allow() 的标志
# 定义枚举类型 hwloc_allow_flags_e，表示允许的标志
enum hwloc_allow_flags_e {
  # 标记拓扑中的所有对象都是允许的
  # 调用 hwloc_topology_allow() 时，cpuset 和 nodeset 必须为 NULL
  # 隐藏初始化
  HWLOC_ALLOW_FLAG_ALL = (1UL<<0),

  # 仅允许当前进程可用的对象
  # 拓扑必须具有 ::HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM，以便实际从操作系统中检索可用资源集
  # 调用 hwloc_topology_allow() 时，cpuset 和 nodeset 必须为 NULL
  # 隐藏初始化
  HWLOC_ALLOW_FLAG_LOCAL_RESTRICTIONS = (1UL<<1),

  # 允许自定义对象集，作为 cpuset 和/或 nodeset 参数传递给 hwloc_topology_allow()
  # 隐藏初始化
  HWLOC_ALLOW_FLAG_CUSTOM = (1UL<<2)
};

# 更改拓扑中允许的处理器和 NUMA 节点集
# 仅当拓扑设置了 ::HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED 时，此函数才有效
# 它不修改任何对象，只是更改由 hwloc_topology_get_allowed_cpuset() 和 hwloc_topology_get_allowed_nodeset() 返回的集合
# 在从运行在不同 Linux Cgroup 中的另一个进程导入拓扑时特别有用
# flags 必须设置为 ::hwloc_allow_flags_e 中的一个标志
# 注意：应该使用 hwloc_topology_restrict() 来从拓扑中移除对象
HWLOC_DECLSPEC int hwloc_topology_allow(hwloc_topology_t __hwloc_restrict topology, hwloc_const_cpuset_t cpuset, hwloc_const_nodeset_t nodeset, unsigned long flags);
/**
 * \brief 将一个 MISC 对象作为拓扑的叶子节点添加
 *
 * 在给定的父节点位置创建一个新的 MISC 对象，并将其插入到拓扑中。它被追加到现有 Misc 子节点的列表中，而不会添加任何中间层级。这对于注释拓扑而不实际改变层次结构非常有用。
 *
 * \p name 应该在拓扑中所有 Misc 对象中是唯一的。它将被复制以设置新对象的属性。
 *
 * 新的叶子对象将不具有任何 \p cpuset。
 *
 * \return 新创建的对象
 *
 * \return 在错误时返回 \c NULL。
 *
 * \return 如果 Misc 对象被过滤出拓扑（::HWLOC_TYPE_FILTER_KEEP_NONE），则返回 \c NULL。
 *
 * \note 如果 \p name 包含一些不可打印的字符，在导出到 XML 时，它们将被丢弃，请参见 hwloc_topology_export_xml() 在 hwloc/export.h 中。
 */
HWLOC_DECLSPEC hwloc_obj_t hwloc_topology_insert_misc_object(hwloc_topology_t topology, hwloc_obj_t parent, const char *name);

/**
 * \brief 分配一个 Group 对象，以便稍后插入到 hwloc_topology_insert_group_object() 中。
 *
 * 此函数返回一个新的 Group 对象。
 *
 * 调用者应该（至少）在插入对象到拓扑之前初始化其集合。参见 hwloc_topology_insert_group_object()。
 */
HWLOC_DECLSPEC hwloc_obj_t hwloc_topology_alloc_group_object(hwloc_topology_t topology);

/**
 * \brief 通过对另一个对象的集合进行 OR 运算来设置对象的 cpusets/nodesets。
 *
 * 对于 \p src 中定义的每个 cpuset 或 nodeset，在 \p dst 中分配相应的集合，并通过 OR 运算将 \p src 添加到其中。
 *
 * 这个函数在 hwloc_topology_alloc_group_object() 和 hwloc_topology_insert_group_object() 之间很方便。它构建了将作为多个对象的新中间父级插入的新 Group 的集合。
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
HWLOC_DECLSPEC int hwloc_topology_refresh(hwloc_topology_t topology);



#ifdef __cplusplus
} /* extern "C" */
#endif



/* high-level helpers */
#include "hwloc/helper.h"



/* inline code of some functions above */
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
#include "hwloc/diff.h"



/* deprecated headers */
#include "hwloc/deprecated.h"



#endif /* HWLOC_H */
```