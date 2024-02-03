# `xmrig\src\3rdparty\hwloc\include\private\private.h`

```cpp
/*
 * 版权声明，版权归 CNRS 所有
 * 版权声明，版权归 Inria 2009-2022 所有
 * 版权声明，版权归 Université Bordeaux 2009-2012, 2020 所有
 * 版权声明，版权归 Cisco Systems, Inc. 2009-2011 所有
 *
 * 请参阅顶层目录中的 COPYING 文件。
 */

/* 内部类型和辅助函数。 */


#ifdef HWLOC_INSIDE_PLUGIN
/*
 * 这些声明仅供内部使用，插件无法访问
 * (下面的许多函数是内部静态符号)。
 */
#error 该文件不应在插件中使用
#endif


#ifndef HWLOC_PRIVATE_H
#define HWLOC_PRIVATE_H

#include "private/autogen/config.h"
#include "hwloc.h"
#include "hwloc/bitmap.h"
#include "private/components.h"
#include "private/misc.h"

#include <sys/types.h>
#ifdef HAVE_UNISTD_H
#include <unistd.h>
#endif
#ifdef HAVE_STDINT_H
#include <stdint.h>
#endif
#ifdef HAVE_SYS_UTSNAME_H
#include <sys/utsname.h>
#endif
#include <string.h>

#define HWLOC_TOPOLOGY_ABI 0x20400 /* struct topology 布局的版本 */

struct hwloc_internal_location_s {
  enum hwloc_location_type_e type;
  union {
    struct {
      hwloc_obj_t obj; /* 在刷新之间缓存 */
      uint64_t gp_index;
      hwloc_obj_type_t type;
    } object; /* 如果 type == HWLOC_LOCATION_TYPE_OBJECT */
    hwloc_cpuset_t cpuset; /* 如果 type == HWLOC_LOCATION_TYPE_CPUSET */
  } location;
};

/*****************************************************
 * 警告：
 * 对此结构体（及其子结构体）的下面更改
 * 应该导致 HWLOC_TOPOLOGY_ABI 的增加。
 *****************************************************/
  # 定义了一个名为 hwloc_topology 的结构体，用于表示硬件拓扑信息
struct hwloc_topology {
  unsigned topology_abi;  # 用于表示拓扑信息的 ABI 版本号

  unsigned nb_levels;  # 水平层级的数量
  unsigned nb_levels_allocated;  # 分配并清零的水平层级数量，包括 level_nbobjects 和以下的层级
  unsigned *level_nbobjects;  # 每个水平层级上的对象数量
  struct hwloc_obj ***levels;  # 直接访问层级的指针，levels[l = 0 .. nblevels-1][0..level_nbobjects[l]]
  unsigned long flags;  # 标志位
  int type_depth[HWLOC_OBJ_TYPE_MAX];  # 每种对象类型的深度
  enum hwloc_type_filter_e type_filter[HWLOC_OBJ_TYPE_MAX];  # 每种对象类型的过滤器
  int is_thissystem;  # 表示是否是当前系统的拓扑信息
  int is_loaded;  # 表示拓扑信息是否已加载
  int modified;  # 如果最近添加/删除了对象，则大于 0，表示需要重新连接
  hwloc_pid_t pid;  # 查看拓扑信息的进程 ID，对于自身为 0
  void *userdata;  # 用户数据
  uint64_t next_gp_index;  # 下一个全局索引

  void *adopted_shmem_addr;  # 采用的共享内存地址
  size_t adopted_shmem_length;  # 采用的共享内存长度

  # 定义了 6 个特殊层级的宏
#define HWLOC_NR_SLEVELS 6
#define HWLOC_SLEVEL_NUMANODE 0
#define HWLOC_SLEVEL_BRIDGE 1
#define HWLOC_SLEVEL_PCIDEV 2
#define HWLOC_SLEVEL_OSDEV 3
#define HWLOC_SLEVEL_MISC 4
#define HWLOC_SLEVEL_MEMCACHE 5
  /* order must match negative depth, it's asserted in setup_defaults() */
#define HWLOC_SLEVEL_FROM_DEPTH(x) (HWLOC_TYPE_DEPTH_NUMANODE-(x))
#define HWLOC_SLEVEL_TO_DEPTH(x) (HWLOC_TYPE_DEPTH_NUMANODE-(x))
  struct hwloc_special_level_s {
    unsigned nbobjs;  # 对象数量
    struct hwloc_obj **objs;  # 对象数组
    struct hwloc_obj *first, *last;  # 在构建 objs 数组之前临时使用的对象
  } slevels[HWLOC_NR_SLEVELS];  # 特殊层级数组

  hwloc_bitmap_t allowed_cpuset;  # 允许的 CPU 集合
  hwloc_bitmap_t allowed_nodeset;  # 允许的节点集合

  struct hwloc_binding_hooks {
    /* These are actually rather OS hooks since some of them are not about binding */
    int (*set_thisproc_cpubind)(hwloc_topology_t topology, hwloc_const_cpuset_t set, int flags);
    // 定义函数指针，用于获取当前进程的 CPU 绑定信息
    int (*get_thisproc_cpubind)(hwloc_topology_t topology, hwloc_cpuset_t set, int flags);
    // 定义函数指针，用于设置当前线程的 CPU 绑定信息
    int (*set_thisthread_cpubind)(hwloc_topology_t topology, hwloc_const_cpuset_t set, int flags);
    // 定义函数指针，用于获取当前线程的 CPU 绑定信息
    int (*get_thisthread_cpubind)(hwloc_topology_t topology, hwloc_cpuset_t set, int flags);
    // 定义函数指针，用于设置指定进程的 CPU 绑定信息
    int (*set_proc_cpubind)(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_cpuset_t set, int flags);
    // 定义函数指针，用于获取指定进程的 CPU 绑定信息
    int (*get_proc_cpubind)(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_cpuset_t set, int flags);
#ifdef hwloc_thread_t
    # 如果定义了 hwloc_thread_t，则声明设置线程 CPU 亲和性的函数指针
    int (*set_thread_cpubind)(hwloc_topology_t topology, hwloc_thread_t tid, hwloc_const_cpuset_t set, int flags);
    # 如果定义了 hwloc_thread_t，则声明获取线程 CPU 亲和性的函数指针
    int (*get_thread_cpubind)(hwloc_topology_t topology, hwloc_thread_t tid, hwloc_cpuset_t set, int flags);
#endif

    # 声明获取当前进程最后一个 CPU 位置的函数指针
    int (*get_thisproc_last_cpu_location)(hwloc_topology_t topology, hwloc_cpuset_t set, int flags);
    # 声明获取当前线程最后一个 CPU 位置的函数指针
    int (*get_thisthread_last_cpu_location)(hwloc_topology_t topology, hwloc_cpuset_t set, int flags);
    # 声明获取指定进程最后一个 CPU 位置的函数指针
    int (*get_proc_last_cpu_location)(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_cpuset_t set, int flags);

    # 声明设置当前进程内存绑定的函数指针
    int (*set_thisproc_membind)(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags);
    # 声明获取当前进程内存绑定的函数指针
    int (*get_thisproc_membind)(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags);
    # 声明设置当前线程内存绑定的函数指针
    int (*set_thisthread_membind)(hwloc_topology_t topology, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags);
    # 声明获取当前线程内存绑定的函数指针
    int (*get_thisthread_membind)(hwloc_topology_t topology, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags);
    # 声明设置指定进程内存绑定的函数指针
    int (*set_proc_membind)(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags);
    # 声明获取指定进程内存绑定的函数指针
    int (*get_proc_membind)(hwloc_topology_t topology, hwloc_pid_t pid, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags);
    # 声明设置指定内存区域内存绑定的函数指针
    int (*set_area_membind)(hwloc_topology_t topology, const void *addr, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags);
    # 声明获取指定内存区域内存绑定的函数指针
    int (*get_area_membind)(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, hwloc_membind_policy_t * policy, int flags);
    # 声明获取指定内存区域内存位置的函数指针
    int (*get_area_memlocation)(hwloc_topology_t topology, const void *addr, size_t len, hwloc_nodeset_t nodeset, int flags);
    # 分配内存的函数指针，返回的指针类型与 alloc_membind 相同，以便可以使用 free_membind
    void *(*alloc)(hwloc_topology_t topology, size_t len);
  /* 定义一个指向函数的指针，用于在特定节点上分配内存并绑定 */
  void *(*alloc_membind)(hwloc_topology_t topology, size_t len, hwloc_const_nodeset_t nodeset, hwloc_membind_policy_t policy, int flags);
  /* 定义一个指向函数的指针，用于在特定节点上释放内存绑定 */
  int (*free_membind)(hwloc_topology_t topology, void *addr, size_t len);

  /* 定义一个指向函数的指针，用于获取允许使用的资源 */
  int (*get_allowed_resources)(hwloc_topology_t topology);
} binding_hooks;

/* 定义一个结构体，用于存储拓扑支持信息 */
struct hwloc_topology_support support;

/* 定义一个指向函数的指针，用于导出用户数据 */
void (*userdata_export_cb)(void *reserved, struct hwloc_topology *topology, struct hwloc_obj *obj);
/* 定义一个指向函数的指针，用于导入用户数据 */
void (*userdata_import_cb)(struct hwloc_topology *topology, struct hwloc_obj *obj, const char *name, const void *buffer, size_t length);
/* 用于标记用户数据是否已解码 */
int userdata_not_decoded;

/* 定义一个内部结构体，用于存储距离信息 */
struct hwloc_internal_distances_s {
  char *name; /* 距离名称，需要一个API来从用户处设置 */

  unsigned id; /* 用于匹配公共距离结构的容器ID字段，不会导出到XML，会在_add()期间重新生成 */

  /* 如果所有对象具有相同类型，则different_types为NULL，unique_type有效。
   * 否则，unique_type为HWLOC_OBJ_TYPE_NONE，different_types包含各个对象类型。
   */
  hwloc_obj_type_t unique_type;
  hwloc_obj_type_t *different_types;

  /* 如果我们支持组，添加union hwloc_obj_attr_u */
  unsigned nbobjs;
  uint64_t *indexes; /* 在将它们转换为对象之前，存储OS或GP索引的数组。
          * 仅用于覆盖PUs或NUMAnodes的距离的索引。
          */
    # 定义宏，用于判断是否使用操作系统索引
    # 如果类型是处理器或NUMA节点，则使用操作系统索引
#define HWLOC_DIST_TYPE_USE_OS_INDEX(_type) ((_type) == HWLOC_OBJ_PU || (_type == HWLOC_OBJ_NUMANODE))
    # 距离矩阵，按照上面的索引/对象数组排序
    # 从 i 到 j 的距离存储在位置 i*nbnodes+j
    uint64_t *values;
    # 类型
    unsigned long kind;

    # 内部距离标志，如果下面的对象数组有效
#define HWLOC_INTERNAL_DIST_FLAG_OBJS_VALID (1U<<0)
    # 内部距离标志，如果距离尚未提交到列表中
#define HWLOC_INTERNAL_DIST_FLAG_NOT_COMMITTED (1U<<1)
    unsigned iflags;

    # 对象当前按物理索引顺序存储
    # 对象数组
    hwloc_obj_t *objs; 
    # 前一个距离对象和后一个距离对象
    struct hwloc_internal_distances_s *prev, *next;
  } *first_dist, *last_dist;
  # 下一个距离对象的ID
  unsigned next_dist_id;

  # 内存属性
  # 内存属性结构体
  unsigned nr_memattrs;
  struct hwloc_internal_memattr_s {
    # 内存属性信息
    # 名称
    char *name; 
    # 标志
    unsigned long flags;
    # 内部标志
    # 名称无需释放
#define HWLOC_IMATTR_FLAG_STATIC_NAME (1U<<0)
    # 缓存有效
#define HWLOC_IMATTR_FLAG_CACHE_VALID (1U<<1)
    # 便利属性，报告来自非内存属性的值（只读，没有实际存储的目标）
#define HWLOC_IMATTR_FLAG_CONVENIENCE (1U<<2)
    unsigned iflags;

    # 值数组
    unsigned nr_targets;
    # 内存属性目标结构体
    struct hwloc_internal_memattr_target_s {
      # 目标对象
      hwloc_obj_t obj; 
      # 类型
      hwloc_obj_type_t type;
      # 操作系统索引
      unsigned os_index; 
      # 全局索引
      hwloc_uint64_t gp_index;

      # 如果没有发起者，则使用该属性值
      hwloc_uint64_t noinitiator_value;
      # 否则使用发起者
      unsigned nr_initiators;
      # 内存属性发起者结构体
      struct hwloc_internal_memattr_initiator_s {
        # 发起者位置
        struct hwloc_internal_location_s initiator;
        # 值
        hwloc_uint64_t value;
      } *initiators;
  // 定义指向 cpukind 结构体的指针数组
  } *targets;
  // 定义指向 memattrs 结构体的指针数组
  } *memattrs;

  // 定义 hybridcpus
  // 记录 CPU 种类的数量
  unsigned nr_cpukinds;
  // 记录已分配的 CPU 种类数量
  unsigned nr_cpukinds_allocated;
  // 定义内部的 cpukind 结构体
  struct hwloc_internal_cpukind_s {
    // CPU 核心的位图
    hwloc_cpuset_t cpuset;
#define HWLOC_CPUKIND_EFFICIENCY_UNKNOWN -1
    // 定义一个名为 HWLOC_CPUKIND_EFFICIENCY_UNKNOWN 的宏，值为 -1
    int efficiency;
    // 效率变量
    int forced_efficiency; /* returned by the hardware or OS if any */
    // 强制效率变量，如果有的话由硬件或操作系统返回
    hwloc_uint64_t ranking_value; /* internal value for ranking */
    // 用于排序的内部数值
    unsigned nr_infos;
    // 信息数量
    struct hwloc_info_s *infos;
    // 信息结构体指针
  } *cpukinds;

  int grouping;
  // 分组变量
  int grouping_verbose;
  // 分组详细变量
  unsigned grouping_nbaccuracies;
  // 分组准确性数量
  float grouping_accuracies[5];
  // 分组准确性数组
  unsigned grouping_next_subkind;
  // 下一个子类分组

  /* list of enabled backends. */
  struct hwloc_backend * backends;
  // 启用的后端列表
  struct hwloc_backend * get_pci_busid_cpuset_backend; /* first backend that provides get_pci_busid_cpuset() callback */
  // 提供 get_pci_busid_cpuset() 回调的第一个后端
  unsigned backend_phases;
  // 后端阶段数量
  unsigned backend_excluded_phases;
  // 排除的后端阶段数量

  /* memory allocator for topology objects */
  struct hwloc_tma * tma;
  // 拓扑对象的内存分配器

/*****************************************************
 * WARNING:
 * changes above in this structure (and its children)
 * should cause a bump of HWLOC_TOPOLOGY_ABI.
 *****************************************************/

  /*
   * temporary variables during discovery
   */

  /* machine-wide memory.
   * temporarily stored there by OSes that only provide this without NUMA information,
   * and actually used later by the core.
   */
  struct hwloc_numanode_attr_s machine_memory;
  // 机器范围内存
  // 由仅提供此信息而不提供 NUMA 信息的操作系统临时存储在那里
  // 后来由核心实际使用

  /* pci stuff */
  int pci_has_forced_locality;
  // PCI 强制本地性
  unsigned pci_forced_locality_nr;
  // PCI 强制本地性数量
  struct hwloc_pci_forced_locality_s {
    unsigned domain;
    unsigned bus_first, bus_last;
    hwloc_bitmap_t cpuset;
  } * pci_forced_locality;
  // PCI 强制本地性结构体指针
  hwloc_uint64_t pci_locality_quirks;
  // PCI 本地性特性

  /* component blacklisting */
  unsigned nr_blacklisted_components;
  // 组件黑名单数量
  struct hwloc_topology_forced_component_s {
    struct hwloc_disc_component *component;
    unsigned phases;
  } *blacklisted_components;
  // 强制组件结构体指针

  /* FIXME: keep until topo destroy and reuse for finding specific buses */
  struct hwloc_pci_locality_s {
    unsigned domain;
    unsigned bus_min;
    unsigned bus_max;
    hwloc_bitmap_t cpuset;
    hwloc_obj_t parent;
  // PCI 本地性结构体
    # 定义指向 hwloc_pci_locality_s 结构体的指针 prev 和 next
    struct hwloc_pci_locality_s *prev, *next;
    # 定义指向 hwloc_pci_locality_s 结构体的指针 first_pci_locality 和 last_pci_locality
    } *first_pci_locality, *last_pci_locality;
# 外部函数声明，用于分配根节点的集合
extern void hwloc_alloc_root_sets(hwloc_obj_t root);
# 外部函数声明，用于设置处理器单元级别
extern void hwloc_setup_pu_level(struct hwloc_topology *topology, unsigned nb_pus);
# 外部函数声明，通过名称获取系统控制信息
extern int hwloc_get_sysctlbyname(const char *name, int64_t *n);
# 外部函数声明，通过名称数组获取系统控制信息
extern int hwloc_get_sysctl(int name[], unsigned namelen, int64_t *n);

# 返回操作系统中的 CPU 数量（仅在本系统有效）
#define HWLOC_FALLBACK_NBPROCESSORS_INCLUDE_OFFLINE 1  # 默认情况下，尝试仅获取在线 CPU
extern int hwloc_fallback_nbprocessors(unsigned flags);
# 返回操作系统中的内存大小（仅在本系统有效）
extern int64_t hwloc_fallback_memsize(void);

# 内部函数声明，用于比较对象的 CPU 集合
extern int hwloc__object_cpusets_compare_first(hwloc_obj_t obj1, hwloc_obj_t obj2);
# 内部函数声明，用于重新排序子对象
extern void hwloc__reorder_children(hwloc_obj_t parent);

# 外部函数声明，用于设置拓扑结构的默认值
extern void hwloc_topology_setup_defaults(struct hwloc_topology *topology);
# 外部函数声明，用于清除拓扑结构
extern void hwloc_topology_clear(struct hwloc_topology *topology);

# 外部函数声明，用于将内存对象作为普通父对象的内存子对象插入
extern struct hwloc_obj * hwloc__attach_memory_object(struct hwloc_topology *topology, hwloc_obj_t parent,
                                                      hwloc_obj_t obj, const char *reason);

# 外部函数声明，通过类型和全局索引获取对象
extern hwloc_obj_t hwloc_get_obj_by_type_and_gp_index(hwloc_topology_t topology, hwloc_obj_type_t type, uint64_t gp_index);

# 外部函数声明，用于 PCI 设备的发现初始化
extern void hwloc_pci_discovery_init(struct hwloc_topology *topology);
# 外部函数声明，用于 PCI 设备的发现准备
extern void hwloc_pci_discovery_prepare(struct hwloc_topology *topology);
# 外部函数声明，用于 PCI 设备的发现退出
extern void hwloc_pci_discovery_exit(struct hwloc_topology *topology);

# 外部函数声明，通过完整 CPU 集合查找或插入 IO 父对象
extern hwloc_obj_t hwloc_find_insert_io_parent_by_complete_cpuset(struct hwloc_topology *topology, hwloc_cpuset_t cpuset);

# 外部函数声明，用于添加信息
extern int hwloc__add_info(struct hwloc_info_s **infosp, unsigned *countp, const char *name, const char *value);
# 添加一个新的信息项到信息数组中，如果已存在同名信息项则不添加
extern int hwloc__add_info_nodup(struct hwloc_info_s **infosp, unsigned *countp, const char *name, const char *value, int replace);
# 将源信息数组中的信息移动到目标信息数组中
extern int hwloc__move_infos(struct hwloc_info_s **dst_infosp, unsigned *dst_countp, struct hwloc_info_s **src_infosp, unsigned *src_countp);
# 复制源信息数组到目标信息数组，使用指定的内存分配器
extern int hwloc__tma_dup_infos(struct hwloc_tma *tma, struct hwloc_info_s **dst_infosp, unsigned *dst_countp, struct hwloc_info_s *src_infos, unsigned src_count);
# 释放信息数组
extern void hwloc__free_infos(struct hwloc_info_s *infos, unsigned count);

# 设置本地操作系统绑定钩子
extern void hwloc_set_native_binding_hooks(struct hwloc_binding_hooks *hooks, struct hwloc_topology_support *support);
# 设置本地操作系统绑定钩子（如果是本系统），否则设置虚拟的绑定钩子
extern void hwloc_set_binding_hooks(struct hwloc_topology *topology);

# 如果是 Linux 系统，设置 Linux 文件系统绑定钩子
extern void hwloc_set_linuxfs_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);

# 如果是 BGQ 系统，设置 BGQ 绑定钩子
extern void hwloc_set_bgq_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);

# 如果是 Solaris 系统，设置 Solaris 绑定钩子
extern void hwloc_set_solaris_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);

# 如果是 AIX 系统，设置 AIX 绑定钩子
extern void hwloc_set_aix_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);

# 如果是 Windows 系统，设置 Windows 绑定钩子
extern void hwloc_set_windows_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);

# 如果是 Darwin 系统，设置 Darwin 绑定钩子
extern void hwloc_set_darwin_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);

# 如果是 FreeBSD 系统，设置 FreeBSD 绑定钩子
extern void hwloc_set_freebsd_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);
#ifdef HWLOC_FREEBSD_SYS
// 如果是 FreeBSD 系统，设置 FreeBSD 特定的绑定钩子和拓扑支持
extern void hwloc_set_freebsd_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);
#endif /* HWLOC_FREEBSD_SYS */

#ifdef HWLOC_NETBSD_SYS
// 如果是 NetBSD 系统，设置 NetBSD 特定的绑定钩子和拓扑支持
extern void hwloc_set_netbsd_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);
#endif /* HWLOC_NETBSD_SYS */

#ifdef HWLOC_HPUX_SYS
// 如果是 HP-UX 系统，设置 HP-UX 特定的绑定钩子和拓扑支持
extern void hwloc_set_hpux_hooks(struct hwloc_binding_hooks *binding_hooks, struct hwloc_topology_support *support);
#endif /* HWLOC_HPUX_SYS */

// 在拓扑对象信息数组中插入与 uname 相关的名称/值
// 如果 cached_uname 不为 NULL，则使用它作为 struct utsname，而不是重新调用 uname
// 任何以 \0 开头的字段都会被忽略
extern void hwloc_add_uname_info(struct hwloc_topology *topology, void *cached_uname);

// 释放未连接到父对象且没有子对象的 obj 及其属性
extern void hwloc_free_unlinked_object(hwloc_obj_t obj);

// 释放 obj 及其子对象，假设它没有连接到父对象
extern void hwloc_free_object_and_children(hwloc_obj_t obj);

// 释放 obj、其下一个兄弟对象及其子对象，假设它们没有连接到父对象
extern void hwloc_free_object_siblings_and_children(hwloc_obj_t obj);

// 用于 alloc 字段，用于获取可以通过 free() 释放的分配数据
void *hwloc_alloc_heap(hwloc_topology_t topology, size_t len);

// 用于 alloc 字段，用于获取可以通过 munmap() 释放的分配数据
void *hwloc_alloc_mmap(hwloc_topology_t topology, size_t len);

// 用于 free_membind 字段，用于使用 free() 释放数据
int hwloc_free_heap(hwloc_topology_t topology, void *addr, size_t len);

// 用于 free_membind 字段，用于使用 munmap() 释放数据
int hwloc_free_mmap(hwloc_topology_t topology, void *addr, size_t len);
/* 根据是否请求了严格模式，分配未绑定的内存或者失败 */
static __hwloc_inline void *
hwloc_alloc_or_fail(hwloc_topology_t topology, size_t len, int flags)
{
  // 如果请求了严格模式，则返回空指针
  if (flags & HWLOC_MEMBIND_STRICT)
    return NULL;
  // 否则调用hwloc_alloc函数分配内存
  return hwloc_alloc(topology, len);
}

// 初始化距离信息
extern void hwloc_internal_distances_init(hwloc_topology_t topology);
// 准备距离信息
extern void hwloc_internal_distances_prepare(hwloc_topology_t topology);
// 销毁距离信息
extern void hwloc_internal_distances_destroy(hwloc_topology_t topology);
// 复制距离信息
extern int hwloc_internal_distances_dup(hwloc_topology_t new, hwloc_topology_t old);
// 刷新距离信息
extern void hwloc_internal_distances_refresh(hwloc_topology_t topology);
// 使缓存对象的距离信息无效
extern void hwloc_internal_distances_invalidate_cached_objs(hwloc_topology_t topology);

/* 这些distances_add()函数比hwloc/plugins.h中的函数更高级
 * 但它们可能会在将来更改，因此不导出给插件 */
extern int hwloc_internal_distances_add_by_index(hwloc_topology_t topology, const char *name, hwloc_obj_type_t unique_type, hwloc_obj_type_t *different_types, unsigned nbobjs, uint64_t *indexes, uint64_t *values, unsigned long kind, unsigned long flags);
extern int hwloc_internal_distances_add(hwloc_topology_t topology, const char *name, unsigned nbobjs, hwloc_obj_t *objs, uint64_t *values, unsigned long kind, unsigned long flags);

// 初始化内存属性
extern void hwloc_internal_memattrs_init(hwloc_topology_t topology);
// 准备内存属性
extern void hwloc_internal_memattrs_prepare(hwloc_topology_t topology);
// 销毁内存属性
extern void hwloc_internal_memattrs_destroy(hwloc_topology_t topology);
// 需要刷新内存属性
extern void hwloc_internal_memattrs_need_refresh(hwloc_topology_t topology);
// 刷新内存属性
extern void hwloc_internal_memattrs_refresh(hwloc_topology_t topology);
// 复制内存属性
extern int hwloc_internal_memattrs_dup(hwloc_topology_t new, hwloc_topology_t old);
# 声明一个函数，用于设置内存属性的值
extern int hwloc_internal_memattr_set_value(hwloc_topology_t topology, hwloc_memattr_id_t id, hwloc_obj_type_t target_type, hwloc_uint64_t target_gp_index, unsigned target_os_index, struct hwloc_internal_location_s *initiator, hwloc_uint64_t value);
# 声明一个函数，用于猜测内存层次结构
extern int hwloc_internal_memattrs_guess_memory_tiers(hwloc_topology_t topology);

# 初始化 CPU 种类
extern void hwloc_internal_cpukinds_init(hwloc_topology_t topology);
# 获取 CPU 种类的排名
extern int hwloc_internal_cpukinds_rank(hwloc_topology_t topology);
# 销毁 CPU 种类
extern void hwloc_internal_cpukinds_destroy(hwloc_topology_t topology);
# 复制 CPU 种类
extern int hwloc_internal_cpukinds_dup(hwloc_topology_t new, hwloc_topology_t old);
# 注册 CPU 种类
#define HWLOC_CPUKINDS_REGISTER_FLAG_OVERWRITE_FORCED_EFFICIENCY (1<<0)
extern int hwloc_internal_cpukinds_register(hwloc_topology_t topology, hwloc_cpuset_t cpuset, int forced_efficiency, const struct hwloc_info_s *infos, unsigned nr_infos, unsigned long flags);
# 限制 CPU 种类
extern void hwloc_internal_cpukinds_restrict(hwloc_topology_t topology);

# 将源缓冲区编码为目标缓冲区
# targsize 必须至少为 4*((srclength+2)/3)+1
# 目标缓冲区将以 0 结尾
extern int hwloc_encode_to_base64(const char *src, size_t srclength, char *target, size_t targsize);
# 将源缓冲区解码为目标缓冲区
# 源缓冲区以 0 结尾
# targsize 必须至少为 srclength*3/4+1（不包括 \0）
# 但只有 srclength*3/4 个字符将是有意义的
# （下一个字符可能在解码过程中部分写入，但应该被忽略）
extern int hwloc_decode_from_base64(char const *src, char *target, size_t targsize);

# 在某些系统上，snprintf 返回写入数据的大小，而不是实际所需的大小
# 有时它在截断时返回 -1
# 有时它不喜欢空输出缓冲区
# hwloc_snprintf 表现正常，但在绝大多数平台上有点过度，所以除非真的需要，不要启用它
#ifdef HWLOC_HAVE_CORRECT_SNPRINTF
#define hwloc_snprintf snprintf
#else
extern int hwloc_snprintf(char *str, size_t size, const char *format, ...) __hwloc_attribute_format(printf, 3, 4);
#endif

/* 如果 HWLOC_HAVE_CORRECT_SNPRINTF 被定义，则使用系统的 snprintf 函数 */
#define hwloc_snprintf snprintf
/* 如果 HWLOC_HAVE_CORRECT_SNPRINTF 没有被定义，则声明 hwloc_snprintf 函数 */
extern int hwloc_snprintf(char *str, size_t size, const char *format, ...) __hwloc_attribute_format(printf, 3, 4);

/* 返回当前运行程序的名称，如果支持的话 */
extern char * hwloc_progname(struct hwloc_topology *topology);

/* 返回当前运行程序的名称，如果支持的话，返回的字符串需要由调用者释放 */
extern char * hwloc_progname(struct hwloc_topology *topology);

/* obj->attr->group.kind 内部值的定义 */
/* 当合并两个组时，核心将保留最小的值，这就是为什么用户给定的种类排在前面的原因 */
/* 首先是用户给定的组，应该尽可能保留 */
#define HWLOC_GROUP_KIND_USER                0    /* 用户给定的，用户可能也会使用子种类 */
#define HWLOC_GROUP_KIND_SYNTHETIC            10    /* 子种类是合成描述中的组深度 */
/* 然后是硬件特定的组 */
#define HWLOC_GROUP_KIND_INTEL_KNL_SUBNUMA_CLUSTER    100    /* 没有子种类 */
#define HWLOC_GROUP_KIND_INTEL_EXTTOPOENUM_UNKNOWN    101    /* 子种类是未知级别 */
#define HWLOC_GROUP_KIND_INTEL_MODULE            102    /* 没有子种类 */
#define HWLOC_GROUP_KIND_INTEL_TILE            103    /* 没有子种类 */
#define HWLOC_GROUP_KIND_INTEL_DIE            104    /* 没有子种类 */
#define HWLOC_GROUP_KIND_S390_BOOK            110    /* 子种类 0 是 book，子种类 1 是 drawer（book 的组） */
#define HWLOC_GROUP_KIND_AMD_COMPUTE_UNIT        120    /* 没有子种类 */
/* 然后是操作系统特定的组 */
#define HWLOC_GROUP_KIND_SOLARIS_PG_HW_PERF        200    /* 子种类是组的宽度 */
#define HWLOC_GROUP_KIND_AIX_SDL_UNKNOWN        210    /* 子种类是 SDL 级别 */
#define HWLOC_GROUP_KIND_WINDOWS_PROCESSOR_GROUP    220    /* 没有子种类 */
#define HWLOC_GROUP_KIND_WINDOWS_RELATIONSHIP_UNKNOWN    221    /* 没有子种类 */
#define HWLOC_GROUP_KIND_LINUX_CLUSTER                  222     /* 没有子种类 */
/* 距离组 */
#define HWLOC_GROUP_KIND_DISTANCE            900    /* 子种类是在基于距离的分组期间添加这些组的轮次 */
/* finally, hwloc-specific groups required to insert something else, should disappear as soon as possible */
#define HWLOC_GROUP_KIND_IO                1000    /* no subkind */
#define HWLOC_GROUP_KIND_MEMORY                1001    /* no subkind */

/* memory allocator for topology objects */
struct hwloc_tma {
  void * (*malloc)(struct hwloc_tma *, size_t);  // 定义了一个函数指针，用于分配内存
  void *data;  // 指向数据的指针
  int dontfree; /* when set, free() or realloc() cannot be used, and tma->malloc() cannot fail */  // 当设置时，不能使用 free() 或 realloc()，并且 tma->malloc() 不能失败
};

static __hwloc_inline void *
hwloc_tma_malloc(struct hwloc_tma *tma,
         size_t size)
{
  if (tma) {
    return tma->malloc(tma, size);  // 如果 tma 存在，则调用 tma->malloc() 分配内存
  } else {
    return malloc(size);  // 否则调用标准的 malloc() 分配内存
  }
}

static __hwloc_inline void *
hwloc_tma_calloc(struct hwloc_tma *tma,
         size_t size)
{
  char *ptr = hwloc_tma_malloc(tma, size);  // 调用 hwloc_tma_malloc() 分配内存
  if (ptr)
    memset(ptr, 0, size);  // 如果分配成功，则将内存清零
  return ptr;  // 返回分配的内存指针
}

static __hwloc_inline char *
hwloc_tma_strdup(struct hwloc_tma *tma,
         const char *src)
{
  size_t len = strlen(src);  // 获取字符串的长度
  char *ptr = hwloc_tma_malloc(tma, len+1);  // 调用 hwloc_tma_malloc() 分配内存
  if (ptr)
    memcpy(ptr, src, len+1);  // 如果分配成功，则将字符串复制到新分配的内存中
  return ptr;  // 返回分配的内存指针
}

/* bitmap allocator to be used inside hwloc */
extern hwloc_bitmap_t hwloc_bitmap_tma_dup(struct hwloc_tma *tma, hwloc_const_bitmap_t old);  // 用于在 hwloc 内部使用的位图分配器

extern int hwloc__topology_dup(hwloc_topology_t *newp, hwloc_topology_t old, struct hwloc_tma *tma);  // 复制拓扑结构
extern void hwloc__topology_disadopt(hwloc_topology_t  topology);  // 取消拓扑结构的采用

#endif /* HWLOC_PRIVATE_H */
```