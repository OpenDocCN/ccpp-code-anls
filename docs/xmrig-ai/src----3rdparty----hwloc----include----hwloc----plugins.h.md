# `xmrig\src\3rdparty\hwloc\include\hwloc\plugins.h`

```
/*
 * 版权声明
 * 版权所有 © 2013-2022 Inria
 * 版权所有 © 2016 Cisco Systems, Inc.
 * 请查看顶层目录中的 COPYING 文件。
 */

#ifndef HWLOC_PLUGINS_H
#define HWLOC_PLUGINS_H

/** \file
 * \brief 用于构建 hwloc 插件的公共接口。
 */

// hwloc_backend 结构体的声明
struct hwloc_backend;

// 包含 hwloc.h 头文件
#include "hwloc.h"

#ifdef HWLOC_INSIDE_PLUGIN
// 为了 hwloc_plugin_check_namespace() 函数的需要
#ifdef HWLOC_HAVE_LTDL
// 如果使用 libltdl 库
#include <ltdl.h>
#else
// 如果不使用 libltdl 库
#include <dlfcn.h>
#endif
#endif

/** \defgroup hwlocality_disc_components 组件和插件：发现组件
 *
 * \note 当 ::HWLOC_COMPONENT_ABI 被修改时，这些结构和函数可能会发生变化。
 *
 * @{
 */

/** \brief 发现组件结构
 *
 * 这是主要的组件类型，负责发现。
 * 它们由通用组件注册，可以是静态构建的，也可以是插件。
 */
# 定义 hwloc_disc_component 结构体，用于表示发现组件
struct hwloc_disc_component {
  /** \brief 名称。
   * 如果此组件作为插件构建，此名称不必与插件文件名匹配。
   */
  const char *name;

  /** \brief 此组件执行的发现阶段。
   * 使用位或运算设置 ::hwloc_disc_phase_t
   */
  unsigned phases;

  /** \brief 要排除的组件阶段，作为 ::hwloc_disc_phase_t 的位或集合。
   *
   * 对于全局组件，这通常包括所有其他阶段 (\c ~UL)。
   *
   * 其他组件仅排除可能带来冲突拓扑信息的类型。MISC 组件通常不应该被排除，
   * 因为它们通常提供非主要的附加信息。
   */
  unsigned excluded_phases;

  /** \brief 实例化回调函数，用于从组件创建后端。
   * 参数 data1、data2、data3 除了具有特殊启用例程的组件之外都为 NULL，例如 hwloc_topology_set_xml()。
   */
  struct hwloc_backend * (*instantiate)(struct hwloc_topology *topology, struct hwloc_disc_component *component, unsigned excluded_phases, const void *data1, const void *data2, const void *data3);

  /** \brief 组件优先级。
   * 用于对 topology->components 进行排序，优先级高的排在前面。
   * 也用于在具有相同名称的两个组件之间进行选择。
   *
   * 通常值为
   * 50 用于本机操作系统（或平台）组件，
   * 45 用于 x86，
   * 40 用于无操作系统回退，
   * 30 用于全局组件（xml、synthetic），
   * 20 用于 pci，
   * 10 用于其他杂项组件（opencl 等）。
   */
  unsigned priority;

  /** \brief 默认情况下启用。
   * 如果未设置，则除非显式请求，否则将被禁用。
   */
  unsigned enabled_by_default;

  /** \private 用于在 topology->components 上按优先级列出组件的内部使用
   * (组件结构通常是只读的，在使用此字段排队之前，核心会复制它)
   */
  struct hwloc_disc_component * next;
};

/** @} */
/**
 * \defgroup hwlocality_disc_backends Components and Plugins: Discovery backends
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

/** \brief Discovery phase */
typedef enum hwloc_disc_phase_e {
  /** \brief xml or synthetic, platform-specific components such as bgq.
   * Discovers everything including CPU, memory, I/O and everything else.
   * A component with a Global phase usually excludes all other phases.
   * \hideinitializer */
  HWLOC_DISC_PHASE_GLOBAL = (1U<<0),

  /** \brief CPU discovery.
   * \hideinitializer */
  HWLOC_DISC_PHASE_CPU = (1U<<1),

  /** \brief Attach memory to existing CPU objects.
   * \hideinitializer */
  HWLOC_DISC_PHASE_MEMORY = (1U<<2),

  /** \brief Attach PCI devices and bridges to existing CPU objects.
   * \hideinitializer */
  HWLOC_DISC_PHASE_PCI = (1U<<3),

  /** \brief I/O discovery that requires PCI devices (OS devices such as OpenCL, CUDA, etc.).
   * \hideinitializer */
  HWLOC_DISC_PHASE_IO = (1U<<4),

  /** \brief Misc objects that gets added below anything else.
   * \hideinitializer */
  HWLOC_DISC_PHASE_MISC = (1U<<5),

  /** \brief Annotating existing objects, adding distances, etc.
   * \hideinitializer */
  HWLOC_DISC_PHASE_ANNOTATE = (1U<<6),

  /** \brief Final tweaks to a ready-to-use topology.
   * This phase runs once the topology is loaded, before it is returned to the topology.
   * Hence it may only use the main hwloc API for modifying the topology,
   * for instance by restricting it, adding info attributes, etc.
   * \hideinitializer */
  HWLOC_DISC_PHASE_TWEAK = (1U<<7)
} hwloc_disc_phase_t;

/** \brief Discovery status flags */
enum hwloc_disc_status_flag_e {
  /** \brief The sets of allowed resources were already retrieved \hideinitializer */
  HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES = (1UL<<1)
}
/** \brief 发现状态结构
 *
 * 由核心和后端用于通知在发现过程中已经/正在进行的操作
 */
struct hwloc_disc_status {
  /** \brief 当前执行的发现阶段
   * 必须与组件阶段字段中的一个阶段匹配
   */
  hwloc_disc_phase_t phase;

  /** \brief 动态排除的阶段
   * 如果组件在发现过程中决定某些阶段不再需要
   */
  unsigned excluded_phases;

  /** \brief OR'ed set of hwloc_disc_status_flag_e */
  unsigned long flags;
};

/** \brief 发现后端结构
 *
 * 后端是发现组件的实例化。
 * 当组件为拓扑启用时，其instantiate()回调会创建一个后端。
 *
 * hwloc_backend_alloc()将所有字段初始化为默认值
 * 组件可以在启用后端之前更改这些值（除了"component"和"next"）
 * 
 * 大多数后端假设拓扑的is_thissystem标志已设置，因为它们与底层操作系统通信。
 * 但是，它们仍然可以在没有is_thissystem标志的拓扑中用于调试目的。
 * 实际上，在这种情况下，它们通常会被自动禁用
 * （通过xml或合成后端排除，或者通过环境变量在更改Linux fsroot或x86 cpuid路径时排除）。
 */
struct hwloc_backend {
  /** \private Reserved for the core, set by hwloc_backend_alloc() */
  // 用于核心保留，由hwloc_backend_alloc()设置
  struct hwloc_disc_component * component;
  /** \private Reserved for the core, set by hwloc_backend_enable() */
  // 用于核心保留，由hwloc_backend_enable()设置
  struct hwloc_topology * topology;
  /** \private Reserved for the core. Set to 1 if forced through envvar, 0 otherwise. */
  // 用于核心保留。如果通过环境变量强制设置，则设置为1，否则为0。
  int envvar_forced;
  /** \private Reserved for the core. Used internally to list backends topology->backends. */
  // 用于核心保留。在内部用于列出后端topology->backends。
  struct hwloc_backend * next;

  /** \brief Discovery phases performed by this component, possibly without some of them if excluded by other components.
   * OR'ed set of ::hwloc_disc_phase_t
   */
  // 此组件执行的发现阶段，如果被其他组件排除，则可能没有其中一些。::hwloc_disc_phase_t的OR'ed集合
  unsigned phases;

  /** \brief Backend flags, currently always 0. */
  // 后端标志，当前始终为0。
  unsigned long flags;

  /** \brief Backend-specific 'is_thissystem' property.
   * Set to 0 if the backend disables the thissystem flag for this topology
   * (e.g. loading from xml or synthetic string,
   *  or using a different fsroot on Linux, or a x86 CPUID dump).
   * Set to -1 if the backend doesn't care (default).
   */
  // 后端特定的'is_thissystem'属性。
  // 如果后端禁用此拓扑的thissystem标志，则设置为0（例如，从xml或合成字符串加载，或在Linux上使用不同的fsroot，或x86 CPUID转储）。
  // 如果后端不关心（默认），则设置为-1。
  int is_thissystem;

  /** \brief Backend private data, or NULL if none. */
  // 后端私有数据，如果没有则为NULL。
  void * private_data;
  /** \brief Callback for freeing the private_data.
   * May be NULL.
   */
  // 用于释放private_data的回调函数。
  // 可能为NULL。
  void (*disable)(struct hwloc_backend *backend);

  /** \brief Main discovery callback.
   * returns -1 on error, either because it couldn't add its objects ot the existing topology,
   * or because of an actual discovery/gathering failure.
   * May be NULL.
   */
  // 主发现回调。
  // 在错误时返回-1，要么是因为它无法将其对象添加到现有的拓扑结构，要么是因为实际的发现/收集失败。
  // 可能为NULL。
  int (*discover)(struct hwloc_backend *backend, struct hwloc_disc_status *status);

  /** \brief Callback to retrieve the locality of a PCI object.
   * Called by the PCI core when attaching PCI hierarchy to CPU objects.
   * May be NULL.
   */
  // 用于检索PCI对象的位置的回调。
  // 当将PCI层次结构附加到CPU对象时，由PCI核心调用。
  // 可能为NULL。
  int (*get_pci_busid_cpuset)(struct hwloc_backend *backend, struct hwloc_pcidev_attr_s *busid, hwloc_bitmap_t cpuset);
};
/**
 * 分配一个后端结构，设置良好的默认值，初始化后端->组件和拓扑结构等。
 * 调用者将修改所需的内容，然后调用hwloc_backend_enable()。
 */
HWLOC_DECLSPEC struct hwloc_backend * hwloc_backend_alloc(struct hwloc_topology *topology, struct hwloc_disc_component *component);

/**
 * 启用先前分配并设置好的后端。
 */
HWLOC_DECLSPEC int hwloc_backend_enable(struct hwloc_backend *backend);

/** @} */

/**
 * \defgroup hwlocality_generic_components 组件和插件：通用组件
 *
 * \note 当::HWLOC_COMPONENT_ABI被修改时，这些结构和函数可能会发生变化。
 *
 * @{
 */

/**
 * 通用组件类型
 */
typedef enum hwloc_component_type_e {
  /** 数据字段必须指向一个struct hwloc_disc_component。 */
  HWLOC_COMPONENT_TYPE_DISC,

  /** 数据字段必须指向一个struct hwloc_xml_component。 */
  HWLOC_COMPONENT_TYPE_XML
} hwloc_component_type_t;

/**
 * 通用组件结构
 *
 * 通用组件结构，可以是由configure在static-components.h中静态列出的，也可以作为插件动态加载。
 */
# 定义一个结构体 hwloc_component
struct hwloc_component {
  /** \brief Component ABI version, set to ::HWLOC_COMPONENT_ABI */
  unsigned abi;  # 组件的ABI版本号，设置为 ::HWLOC_COMPONENT_ABI

  /** \brief Process-wide component initialization callback.
   *
   * This optional callback is called when the component is registered
   * to the hwloc core (after loading the plugin).
   *
   * When the component is built as a plugin, this callback
   * should call hwloc_check_plugin_namespace()
   * and return an negative error code on error.
   *
   * \p flags is always 0 for now.
   *
   * \return 0 on success, or a negative code on error.
   *
   * \note If the component uses ltdl for loading its own plugins,
   * it should load/unload them only in init() and finalize(),
   * to avoid race conditions with hwloc's use of ltdl.
   */
  int (*init)(unsigned long flags);  # 进程范围的组件初始化回调函数

  /** \brief Process-wide component termination callback.
   *
   * This optional callback is called after unregistering the component
   * from the hwloc core (before unloading the plugin).
   *
   * \p flags is always 0 for now.
   *
   * \note If the component uses ltdl for loading its own plugins,
   * it should load/unload them only in init() and finalize(),
   * to avoid race conditions with hwloc's use of ltdl.
   */
  void (*finalize)(unsigned long flags);  # 进程范围的组件终止回调函数

  /** \brief Component type */
  hwloc_component_type_t type;  # 组件类型

  /** \brief Component flags, unused for now */
  unsigned long flags;  # 组件标志，目前未使用

  /** \brief Component data, pointing to a struct hwloc_disc_component or struct hwloc_xml_component. */
  void * data;  # 组件数据，指向 struct hwloc_disc_component 或 struct hwloc_xml_component 结构体
};

/** @} */

/** \defgroup hwlocality_components_core_funcs Components and Plugins: Core functions to be used by components
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */
/**
 * 检查错误消息是否被隐藏。
 *
 * 调用者应该只在此函数返回严格小于2时打印关键错误消息
 * （例如，无效的硬件拓扑信息，无效的配置）。
 *
 * 调用者应该在此函数返回0时打印非关键错误消息
 * （例如，初始化CUDA失败）。
 *
 * 此函数默认返回1（仅显示关键错误），
 * 在lstopo中返回0（显示所有错误），
 * 或者在环境中设置了HWLOC_HIDE_ERRORS时返回其他值。
 *
 * 为了清晰起见，使用宏HWLOC_SHOW_CRITICAL_ERRORS()和HWLOC_SHOW_ALL_ERRORS()。
 */
HWLOC_DECLSPEC int hwloc_hide_errors(void);

/**
 * 显示关键错误消息的宏
 */
#define HWLOC_SHOW_CRITICAL_ERRORS() (hwloc_hide_errors() < 2)
/**
 * 显示所有错误消息的宏
 */
#define HWLOC_SHOW_ALL_ERRORS() (hwloc_hide_errors() == 0)

/**
 * 将对象添加到拓扑结构中。
 *
 * 在现有对象root下（如果为NULL，则使用拓扑根对象）插入新对象obj。
 *
 * 它根据cpusets的包含关系沿着其他对象的树进行排序，最终作为包含此对象的最小对象的子对象添加。
 *
 * 如果cpuset为空，则对象的类型（以及可能的一些属性）必须足够找到插入对象的位置。这对于具有内存但没有CPU的NUMA节点尤其重要。
 *
 * 给定对象不应该有子对象。
 *
 * 这应该只在构建层级之前调用。
 *
 * 调用者在调用此函数之前应该检查对象类型是否被过滤掉。
 *
 * 拓扑cpuset/nodesets将被扩大以包括对象集。
 *
 * reason是一个唯一的字符串，用于标识执行此插入调用的位置和原因
 * （在内部插入错误的情况下将显示该字符串）。
 *
 * 成功时返回对象。
 * 出错时返回NULL并释放obj。
 * 如果与已存在的相同对象合并，则返回另一个对象并释放obj。
 */
HWLOC_DECLSPEC hwloc_obj_t
# 在拓扑结构中插入一个对象
# 它被添加为给定父对象的最后一个子对象
# cpuset 完全被忽略，因此最好用于插入奇怪的对象，如 I/O 设备
# 当用于具有 cpusets 的“正常”子对象（从 XML 导入或复制拓扑时），调用者应确保：
# - 子对象按顺序插入
# - 子对象的 cpusets 不相交
# 给定对象可以具有正常、I/O 或 Misc 子对象，只要它们也是按顺序的
# 这些子对象必须具有有效的父对象和下一个兄弟对象指针
# 在调用此函数之前，调用者应检查对象类型是否被过滤掉
HWLOC_DECLSPEC void hwloc_insert_object_by_parent(struct hwloc_topology *topology, hwloc_obj_t parent, hwloc_obj_t obj);

# 分配并初始化给定类型和物理索引的对象
# 如果 os_index 未知或不相关，则使用 HWLOC_UNKNOWN_INDEX
HWLOC_DECLSPEC hwloc_obj_t hwloc_alloc_setup_object(hwloc_topology_t topology, hwloc_obj_type_t type, unsigned os_index);

# 通过对其所有新子对象进行 OR 运算来设置对象的 cpusets/nodesets
# 在拓扑结构中晚添加对象时使用
# 将通过对其所有新子对象进行 OR 运算来更新新对象
# 当 PCI 后端添加主机桥父对象时使用，当距离添加新的 Group 时使用
HWLOC_DECLSPEC int hwloc_obj_add_children_sets(hwloc_obj_t obj);

# 请求重新连接拓扑结构中的子对象和层级
# 可能由后端在发现过程中使用，如果它们需要完全连接的对象数组或列表
# flags 目前未使用，必须为 0
# 声明一个函数，用于重新连接拓扑结构
HWLOC_DECLSPEC int hwloc_topology_reconnect(hwloc_topology_t topology, unsigned long flags __hwloc_attribute_unused);

/** \brief 确保插件可以查找核心符号。
 *
 * 这是一个健全性检查，以避免当 libhwloc 在插件中加载时出现延迟查找失败，然后尝试加载自己的插件时出现问题。
 * 如果 libhwloc 符号在私有命名空间中，这可能会失败（并中止程序）。
 *
 * \return 成功时返回 0。
 * \return 如果无法成功加载插件，则返回 -1。调用者插件的 init() 回调也应返回负错误代码。
 *
 * 插件应在其 init() 回调中调用此函数，以避免后续崩溃，如果加载了使用延迟符号解析的上层的 hwloc（例如使用带有 RTLD_LAZY 的 dlopen 的 OpenCL 实现）。
 *
 * \note 构建系统必须定义 HWLOC_INSIDE_PLUGIN，如果且仅如果构建调用者作为插件。
 *
 * \note 此函数应保持内联，以便插件即使找不到 libhwloc 符号也可以调用它。
 */
static __hwloc_inline int
hwloc_plugin_check_namespace(const char *pluginname __hwloc_attribute_unused, const char *symbol __hwloc_attribute_unused)
{
#ifdef HWLOC_INSIDE_PLUGIN
  void *sym;
#ifdef HWLOC_HAVE_LTDL
  lt_dlhandle handle = lt_dlopen(NULL);
#else
  void *handle = dlopen(NULL, RTLD_NOW|RTLD_LOCAL);
#endif
  if (!handle)
    /* 无法检查，假设事情会正常工作 */
    return 0;
#ifdef HWLOC_HAVE_LTDL
  sym = lt_dlsym(handle, symbol);
  lt_dlclose(handle);
#else
  sym = dlsym(handle, symbol);
  dlclose(handle);
#endif
  if (!sym) {
    static int verboseenv_checked = 0;
    static int verboseenv_value = 0;
    if (!verboseenv_checked) {
      const char *verboseenv = getenv("HWLOC_PLUGINS_VERBOSE");
      verboseenv_value = verboseenv ? atoi(verboseenv) : 0;
      verboseenv_checked = 1;
    }
    # 如果 verboseenv_value 为真，则向标准错误流输出插件禁用的消息，包括插件名称和未找到的核心符号
    if (verboseenv_value)
      fprintf(stderr, "Plugin `%s' disabling itself because it cannot find the `%s' core symbol.\n",
          pluginname, symbol);
    # 返回 -1，表示插件禁用
    return -1;
  }
#endif /* HWLOC_INSIDE_PLUGIN */
  return 0;
}

/** @} */

/** \defgroup hwlocality_components_filtering Components and Plugins: Filtering objects
 *
 * \note These structures and functions may change when ::HWLOC_COMPONENT_ABI is modified.
 *
 * @{
 */

/** \brief Check whether the given PCI device classid is important.
 *
 * \return 1 if important, 0 otherwise.
 */
static __hwloc_inline int
hwloc_filter_check_pcidev_subtype_important(unsigned classid)
{
  // 将 classid 右移 8 位，获取 baseclass
  unsigned baseclass = classid >> 8;
  // 判断 baseclass 是否为重要的 PCI 设备类型
  return (baseclass == 0x03 /* PCI_BASE_CLASS_DISPLAY */
      || baseclass == 0x02 /* PCI_BASE_CLASS_NETWORK */
      || baseclass == 0x01 /* PCI_BASE_CLASS_STORAGE */
      || baseclass == 0x00 /* Unclassified, for Atos/Bull BXI */
      || baseclass == 0x0b /* PCI_BASE_CLASS_PROCESSOR */
      || classid == 0x0c04 /* PCI_CLASS_SERIAL_FIBER */
      || classid == 0x0c06 /* PCI_CLASS_SERIAL_INFINIBAND */
          || classid == 0x0502 /* PCI_CLASS_MEMORY_CXL */
          || baseclass == 0x06 /* PCI_BASE_CLASS_BRIDGE with non-PCI downstream. the core will drop the useless ones later */
      || baseclass == 0x12 /* Processing Accelerators */);
}

/** \brief Check whether the given OS device subtype is important.
 *
 * \return 1 if important, 0 otherwise.
 */
static __hwloc_inline int
hwloc_filter_check_osdev_subtype_important(hwloc_obj_osdev_type_t subtype)
{
  // 判断 OS 设备的 subtype 是否为重要的类型
  return (subtype != HWLOC_OBJ_OSDEV_DMA);
}

/** \brief Check whether a non-I/O object type should be filtered-out.
 *
 * Cannot be used for I/O objects.
 *
 * \return 1 if the object type should be kept, 0 otherwise.
 */
static __hwloc_inline int
hwloc_filter_check_keep_object_type(hwloc_topology_t topology, hwloc_obj_type_t type)
{
  // 获取指定对象类型的过滤器
  enum hwloc_type_filter_e filter = HWLOC_TYPE_FILTER_KEEP_NONE;
  hwloc_topology_get_type_filter(topology, type, &filter);
  // 断言过滤器不为 HWLOC_TYPE_FILTER_KEEP_IMPORTANT，因为该值仅用于 I/O 对象
  assert(filter != HWLOC_TYPE_FILTER_KEEP_IMPORTANT); /* IMPORTANT only used for I/O */
  // 如果过滤器为 HWLOC_TYPE_FILTER_KEEP_NONE，则返回 0，否则返回 1
  return filter == HWLOC_TYPE_FILTER_KEEP_NONE ? 0 : 1;
}
/** \brief 检查给定对象是否应该被过滤掉。
 *
 * \return 如果对象类型应该被保留，则返回1，否则返回0。
 */
static __hwloc_inline int
hwloc_filter_check_keep_object(hwloc_topology_t topology, hwloc_obj_t obj)
{
  // 获取对象的类型
  hwloc_obj_type_t type = obj->type;
  // 初始化过滤器为保留空
  enum hwloc_type_filter_e filter = HWLOC_TYPE_FILTER_KEEP_NONE;
  // 获取给定类型的过滤器
  hwloc_topology_get_type_filter(topology, type, &filter);
  // 如果过滤器为保留空，则返回0
  if (filter == HWLOC_TYPE_FILTER_KEEP_NONE)
    return 0;
  // 如果过滤器为保留重要类型
  if (filter == HWLOC_TYPE_FILTER_KEEP_IMPORTANT) {
    // 如果类型为 PCI 设备，则检查其子类型是否重要
    if (type == HWLOC_OBJ_PCI_DEVICE)
      return hwloc_filter_check_pcidev_subtype_important(obj->attr->pcidev.class_id);
    // 如果类型为操作系统设备，则检查其子类型是否重要
    if (type == HWLOC_OBJ_OS_DEVICE)
      return hwloc_filter_check_osdev_subtype_important(obj->attr->osdev.type);
  }
  // 返回1
  return 1;
}

/** @} */




/** \defgroup hwlocality_components_pcidisc 组件和插件：PCI 发现的辅助函数
 *
 * \note 当 ::HWLOC_COMPONENT_ABI 被修改时，这些结构和函数可能会发生变化。
 *
 * @{
 */

/** \brief 返回给定能力在 PCI 配置空间缓冲区中的偏移量
 *
 * 此函数需要一个256字节的配置空间。未知/不可用的字节应设置为0xff。
 */
HWLOC_DECLSPEC unsigned hwloc_pcidisc_find_cap(const unsigned char *config, unsigned cap);

/** \brief 通过读取 PCI 配置空间中的 PCI_CAP_ID_EXP 位置的偏移量来填充链路速度。
 *
 * 需要从配置空间中的偏移量开始的20字节的 EXP 能力块，用于寄存器直到链路状态。
 */
HWLOC_DECLSPEC int hwloc_pcidisc_find_linkspeed(const unsigned char *config, unsigned offset, float *linkspeed);

/** \brief 返回给定类和配置空间的 hwloc 对象类型（PCI 设备或桥）。
 *
 * 此函数需要配置的开头的16字节的通用配置头。
 */
HWLOC_DECLSPEC hwloc_obj_type_t hwloc_pcidisc_check_bridge_type(unsigned device_class, const unsigned char *config);
/** \brief 使用给定的 PCI 配置空间填充给定 PCI 桥的属性。
 *
 * 此函数需要在配置的开头有 32 字节的通用配置头。
 *
 * 如果桥字段无效，返回 -1 并销毁 /p obj。
 */
HWLOC_DECLSPEC int hwloc_pcidisc_find_bridge_buses(unsigned domain, unsigned bus, unsigned dev, unsigned func,
                           unsigned *secondary_busp, unsigned *subordinate_busp,
                           const unsigned char *config);

/** \brief 通过查看 PCI 总线 ID，在给定的 PCI 树中插入一个 PCI 对象。
 *
 * 如果 \p treep 指向 \c NULL，则在那里插入新对象。
 */
HWLOC_DECLSPEC void hwloc_pcidisc_tree_insert_by_busid(struct hwloc_obj **treep, struct hwloc_obj *obj);

/** \brief 在给定的 PCI 对象树顶部添加一些主机桥，并将它们附加到拓扑结构中。
 *
 * 其他后端可以通过使用 hwloc_pcidisc_find_by_busid() 或 hwloc_pcidisc_find_busid_parent() 查找 PCI 对象或本地性（例如附加 OS 设备）。
 */
HWLOC_DECLSPEC int hwloc_pcidisc_tree_attach(struct hwloc_topology *topology, struct hwloc_obj *tree);

/** @} */




/** \defgroup hwlocality_components_pcifind 组件和插件：在其他发现过程中查找 PCI 对象
 *
 * \note 当 ::HWLOC_COMPONENT_ABI 被修改时，这些结构和函数可能会发生变化。
 *
 * @{
 */

/** \brief 查找具有给定 PCI 总线 ID 的对象或其父对象。
 *
 * 当附加新对象（通常是 OS 设备）其本地性由 PCI 总线 ID 指定时，此函数返回要用作附加的 PCI 对象的父对象。
 *
 * 如果存在具有此总线 ID 的确切 PCI 设备，则返回该设备。
 * 否则（例如如果它被过滤掉），函数返回具有类似本地性的其他对象（例如父桥或本地 CPU Package）。
 */
# 根据 PCI 总线 ID 精确查找匹配的 PCI 设备或桥接
HWLOC_DECLSPEC struct hwloc_obj * hwloc_pci_find_by_busid(struct hwloc_topology *topology, unsigned domain, unsigned bus, unsigned dev, unsigned func);

# 这对于根据 PCI id 添加特定对象的信息非常有用。当涉及基于 PCI 位置的对象附加时，应优先使用 hwloc_pci_find_parent_by_busid()
/** \brief Find the PCI device or bridge matching a PCI bus ID exactly.
 *
 * This is useful for adding specific information about some objects
 * based on their PCI id. When it comes to attaching objects based on
 * PCI locality, hwloc_pci_find_parent_by_busid() should be preferred.
 */

# 在将其添加到拓扑结构中时，处理新距离结构的句柄
/** \brief Handle to a new distances structure during its addition to the topology. */
typedef void * hwloc_backend_distances_add_handle_t;

# 创建一个新的空距离结构
# 这与 hwloc_distances_add_create() 相同，但此变体设计用于在拓扑发现期间插入后端距离
HWLOC_DECLSPEC hwloc_backend_distances_add_handle_t
hwloc_backend_distances_add_create(hwloc_topology_t topology,
                                   const char *name, unsigned long kind,
                                   unsigned long flags);

# 指定新的空距离结构中的对象和值
# 这类似于 hwloc_distances_add_values()，但此变体设计用于在拓扑发现期间插入后端距离
# 唯一的语义差异是，objs 和 values 不会被复制，而是直接附加到拓扑结构中
# 成功后，这些数组将被传递给核心，并且调用者不应再释放这些数组
HWLOC_DECLSPEC int
# 添加距离数值到硬件拓扑结构中
hwloc_backend_distances_add_values(hwloc_topology_t topology,  # 硬件拓扑结构
                                   hwloc_backend_distances_add_handle_t handle,  # 距离句柄
                                   unsigned nbobjs,  # 对象数量
                                   hwloc_obj_t *objs,  # 对象数组
                                   hwloc_uint64_t *values,  # 距离数值数组
                                   unsigned long flags);  # 标志位

/** \brief 提交新的距离结构。
 *
 * 这类似于hwloc_distances_add_commit()，
 * 但这个变体设计用于在拓扑发现期间插入距离的后端。
 */
HWLOC_DECLSPEC int
hwloc_backend_distances_add_commit(hwloc_topology_t topology,  # 硬件拓扑结构
                                   hwloc_backend_distances_add_handle_t handle,  # 距离句柄
                                   unsigned long flags);  # 标志位

/** @} */

#endif /* HWLOC_PLUGINS_H */
```