# `xmrig\src\3rdparty\hwloc\include\hwloc\rename.h`

```cpp
/*
 * 版权声明
 * 版权所有 © 2009-2011 Cisco Systems, Inc.
 * 版权所有 © 2010-2022 Inria.
 * 请参阅顶层目录中的 COPYING 文件。
 */

#ifndef HWLOC_RENAME_H
#define HWLOC_RENAME_H

#include "hwloc/autogen/config.h"

#ifdef __cplusplus
extern "C" {
#endif

/* 只有在实际重命名符号时才执行这些定义
   (即，如果我们*不*重命名，则避免尝试进行无操作定义) */

#if HWLOC_SYM_TRANSFORM

/* 使用预处理器两步来正确设置前缀。
   创建两个宏：HWLOC_NAME 和 HWLOC_NAME_CAPS 用于重命名
   事物。 */

#define HWLOC_MUNGE_NAME(a, b) HWLOC_MUNGE_NAME2(a, b)
#define HWLOC_MUNGE_NAME2(a, b) a ## b
#define HWLOC_NAME(name) HWLOC_MUNGE_NAME(HWLOC_SYM_PREFIX, hwloc_ ## name)
/* FIXME: 应该在下面是 "HWLOC_ ## name"，未更改，因为这并不重要，而且可能会破坏一些嵌入式程序的黑客 */
#define HWLOC_NAME_CAPS(name) HWLOC_MUNGE_NAME(HWLOC_SYM_PREFIX_CAPS, hwloc_ ## name)

/* 现在将所有“真实”名称定义为带有前缀的名称。这
   允许我们在整个代码库中使用真实名称 (即，“hwloc_<foo>”)；
   预处理器将在幕后调整为带有前缀的名称。 */

/* 来自 hwloc.h 的名称 */

#define hwloc_get_api_version HWLOC_NAME(get_api_version)

#define hwloc_topology HWLOC_NAME(topology)
#define hwloc_topology_t HWLOC_NAME(topology_t)

#define hwloc_cpuset_t HWLOC_NAME(cpuset_t)
#define hwloc_const_cpuset_t HWLOC_NAME(const_cpuset_t)
#define hwloc_nodeset_t HWLOC_NAME(nodeset_t)
#define hwloc_const_nodeset_t HWLOC_NAME(const_nodeset_t)

#define HWLOC_OBJ_MACHINE HWLOC_NAME_CAPS(OBJ_MACHINE)
#define HWLOC_OBJ_NUMANODE HWLOC_NAME_CAPS(OBJ_NUMANODE)
#define HWLOC_OBJ_MEMCACHE HWLOC_NAME_CAPS(OBJ_MEMCACHE)
#define HWLOC_OBJ_PACKAGE HWLOC_NAME_CAPS(OBJ_PACKAGE)
#define HWLOC_OBJ_DIE HWLOC_NAME_CAPS(OBJ_DIE)
#define HWLOC_OBJ_CORE HWLOC_NAME_CAPS(OBJ_CORE)
#define HWLOC_OBJ_PU HWLOC_NAME_CAPS(OBJ_PU)
# 定义各种硬件对象类型的宏，将对象类型转换为大写形式
#define HWLOC_OBJ_L1CACHE HWLOC_NAME_CAPS(OBJ_L1CACHE)
#define HWLOC_OBJ_L2CACHE HWLOC_NAME_CAPS(OBJ_L2CACHE)
#define HWLOC_OBJ_L3CACHE HWLOC_NAME_CAPS(OBJ_L3CACHE)
#define HWLOC_OBJ_L4CACHE HWLOC_NAME_CAPS(OBJ_L4CACHE)
#define HWLOC_OBJ_L5CACHE HWLOC_NAME_CAPS(OBJ_L5CACHE)
#define HWLOC_OBJ_L1ICACHE HWLOC_NAME_CAPS(OBJ_L1ICACHE)
#define HWLOC_OBJ_L2ICACHE HWLOC_NAME_CAPS(OBJ_L2ICACHE)
#define HWLOC_OBJ_L3ICACHE HWLOC_NAME_CAPS(OBJ_L3ICACHE)
#define HWLOC_OBJ_MISC HWLOC_NAME_CAPS(OBJ_MISC)
#define HWLOC_OBJ_GROUP HWLOC_NAME_CAPS(OBJ_GROUP)
#define HWLOC_OBJ_BRIDGE HWLOC_NAME_CAPS(OBJ_BRIDGE)
#define HWLOC_OBJ_PCI_DEVICE HWLOC_NAME_CAPS(OBJ_PCI_DEVICE)
#define HWLOC_OBJ_OS_DEVICE HWLOC_NAME_CAPS(OBJ_OS_DEVICE)
#define HWLOC_OBJ_TYPE_MAX HWLOC_NAME_CAPS(OBJ_TYPE_MAX)
#define hwloc_obj_type_t HWLOC_NAME(obj_type_t)

# 定义缓存类型的枚举和类型
#define hwloc_obj_cache_type_e HWLOC_NAME(obj_cache_type_e)
#define hwloc_obj_cache_type_t HWLOC_NAME(obj_cache_type_t)
#define HWLOC_OBJ_CACHE_UNIFIED HWLOC_NAME_CAPS(OBJ_CACHE_UNIFIED)
#define HWLOC_OBJ_CACHE_DATA HWLOC_NAME_CAPS(OBJ_CACHE_DATA)
#define HWLOC_OBJ_CACHE_INSTRUCTION HWLOC_NAME_CAPS(OBJ_CACHE_INSTRUCTION)

# 定义桥接设备类型的枚举和类型
#define hwloc_obj_bridge_type_e HWLOC_NAME(obj_bridge_type_e)
#define hwloc_obj_bridge_type_t HWLOC_NAME(obj_bridge_type_t)
#define HWLOC_OBJ_BRIDGE_HOST HWLOC_NAME_CAPS(OBJ_BRIDGE_HOST)
#define HWLOC_OBJ_BRIDGE_PCI HWLOC_NAME_CAPS(OBJ_BRIDGE_PCI)

# 定义操作系统设备类型的枚举和类型
#define hwloc_obj_osdev_type_e HWLOC_NAME(obj_osdev_type_e)
#define hwloc_obj_osdev_type_t HWLOC_NAME(obj_osdev_type_t)
#define HWLOC_OBJ_OSDEV_BLOCK HWLOC_NAME_CAPS(OBJ_OSDEV_BLOCK)
#define HWLOC_OBJ_OSDEV_GPU HWLOC_NAME_CAPS(OBJ_OSDEV_GPU)
#define HWLOC_OBJ_OSDEV_NETWORK HWLOC_NAME_CAPS(OBJ_OSDEV_NETWORK)
#define HWLOC_OBJ_OSDEV_OPENFABRICS HWLOC_NAME_CAPS(OBJ_OSDEV_OPENFABRICS)
#define HWLOC_OBJ_OSDEV_DMA HWLOC_NAME_CAPS(OBJ_OSDEV_DMA)
#define HWLOC_OBJ_OSDEV_COPROC HWLOC_NAME_CAPS(OBJ_OSDEV_COPROC)

# 定义比较硬件对象类型的宏
#define hwloc_compare_types HWLOC_NAME(compare_types)

# 定义硬件对象类型的宏
#define hwloc_obj HWLOC_NAME(obj)
# 定义 hwloc_obj_t 为 HWLOC_NAME(obj_t)
#define hwloc_obj_t HWLOC_NAME(obj_t)

# 定义 hwloc_info_s 为 HWLOC_NAME(info_s)
#define hwloc_info_s HWLOC_NAME(info_s)

# 定义 hwloc_obj_attr_u 为 HWLOC_NAME(obj_attr_u)
#define hwloc_obj_attr_u HWLOC_NAME(obj_attr_u)
# 定义 hwloc_numanode_attr_s 为 HWLOC_NAME(numanode_attr_s)
#define hwloc_numanode_attr_s HWLOC_NAME(numanode_attr_s)
# 定义 hwloc_memory_page_type_s 为 HWLOC_NAME(memory_page_type_s)
#define hwloc_memory_page_type_s HWLOC_NAME(memory_page_type_s)
# 定义 hwloc_cache_attr_s 为 HWLOC_NAME(cache_attr_s)
#define hwloc_cache_attr_s HWLOC_NAME(cache_attr_s)
# 定义 hwloc_group_attr_s 为 HWLOC_NAME(group_attr_s)
#define hwloc_group_attr_s HWLOC_NAME(group_attr_s)
# 定义 hwloc_pcidev_attr_s 为 HWLOC_NAME(pcidev_attr_s)
#define hwloc_pcidev_attr_s HWLOC_NAME(pcidev_attr_s)
# 定义 hwloc_bridge_attr_s 为 HWLOC_NAME(bridge_attr_s)
#define hwloc_bridge_attr_s HWLOC_NAME(bridge_attr_s)
# 定义 hwloc_osdev_attr_s 为 HWLOC_NAME(osdev_attr_s)
#define hwloc_osdev_attr_s HWLOC_NAME(osdev_attr_s)

# 定义 hwloc_topology_init 为 HWLOC_NAME(topology_init)
#define hwloc_topology_init HWLOC_NAME(topology_init)
# 定义 hwloc_topology_load 为 HWLOC_NAME(topology_load)
#define hwloc_topology_load HWLOC_NAME(topology_load)
# 定义 hwloc_topology_destroy 为 HWLOC_NAME(topology_destroy)
#define hwloc_topology_destroy HWLOC_NAME(topology_destroy)
# 定义 hwloc_topology_dup 为 HWLOC_NAME(topology_dup)
#define hwloc_topology_dup HWLOC_NAME(topology_dup)
# 定义 hwloc_topology_abi_check 为 HWLOC_NAME(topology_abi_check)
#define hwloc_topology_abi_check HWLOC_NAME(topology_abi_check)
# 定义 hwloc_topology_check 为 HWLOC_NAME(topology_check)
#define hwloc_topology_check HWLOC_NAME(topology_check)

# 定义 hwloc_topology_flags_e 为 HWLOC_NAME(topology_flags_e)
#define hwloc_topology_flags_e HWLOC_NAME(topology_flags_e)

# 定义 HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_WITH_DISALLOWED)
#define HWLOC_TOPOLOGY_FLAG_INCLUDE_DISALLOWED HWLOC_NAME_CAPS(TOPOLOGY_FLAG_WITH_DISALLOWED)
# 定义 HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_IS_THISSYSTEM)
#define HWLOC_TOPOLOGY_FLAG_IS_THISSYSTEM HWLOC_NAME_CAPS(TOPOLOGY_FLAG_IS_THISSYSTEM)
# 定义 HWLOC_TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES)
#define HWLOC_TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES HWLOC_NAME_CAPS(TOPOLOGY_FLAG_THISSYSTEM_ALLOWED_RESOURCES)
# 定义 HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_IMPORT_SUPPORT)
#define HWLOC_TOPOLOGY_FLAG_IMPORT_SUPPORT HWLOC_NAME_CAPS(TOPOLOGY_FLAG_IMPORT_SUPPORT)
# 定义 HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING)
#define HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING HWLOC_NAME_CAPS(TOPOLOGY_FLAG_RESTRICT_TO_CPUBINDING)
# 定义 HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING)
#define HWLOC_TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING HWLOC_NAME_CAPS(TOPOLOGY_FLAG_RESTRICT_TO_MEMBINDING)
# 定义 HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_DONT_CHANGE_BINDING)
#define HWLOC_TOPOLOGY_FLAG_DONT_CHANGE_BINDING HWLOC_NAME_CAPS(TOPOLOGY_FLAG_DONT_CHANGE_BINDING)
# 定义 HWLOC_TOPOLOGY_FLAG_NO_DISTANCES 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_NO_DISTANCES)
#define HWLOC_TOPOLOGY_FLAG_NO_DISTANCES HWLOC_NAME_CAPS(TOPOLOGY_FLAG_NO_DISTANCES)
# 定义 HWLOC_TOPOLOGY_FLAG_NO_MEMATTRS 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_NO_MEMATTRS)
#define HWLOC_TOPOLOGY_FLAG_NO_MEMATTRS HWLOC_NAME_CAPS(TOPOLOGY_FLAG_NO_MEMATTRS)
# 定义 HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS 为 HWLOC_NAME_CAPS(TOPOLOGY_FLAG_NO_CPUKINDS)
#define HWLOC_TOPOLOGY_FLAG_NO_CPUKINDS HWLOC_NAME_CAPS(TOPOLOGY_FLAG_NO_CPUKINDS)

# 定义 hwloc_topology_set_pid 为 HWLOC_NAME(topology_set_pid)
#define hwloc_topology_set_pid HWLOC_NAME(topology_set_pid)
# 定义 hwloc_topology_set_synthetic 为 HWLOC_NAME(topology_set_synthetic)
#define hwloc_topology_set_synthetic HWLOC_NAME(topology_set_synthetic)
# 定义宏，将 hwloc_topology_set_xml 重命名为 HWLOC_NAME(topology_set_xml)
#define hwloc_topology_set_xml HWLOC_NAME(topology_set_xml)
# 定义宏，将 hwloc_topology_set_xmlbuffer 重命名为 HWLOC_NAME(topology_set_xmlbuffer)
#define hwloc_topology_set_xmlbuffer HWLOC_NAME(topology_set_xmlbuffer)
# 定义宏，将 hwloc_topology_components_flag_e 重命名为 HWLOC_NAME(hwloc_topology_components_flag_e)
#define hwloc_topology_components_flag_e HWLOC_NAME(hwloc_topology_components_flag_e)
# 定义宏，将 HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST 重命名为 HWLOC_NAME_CAPS(TOPOLOGY_COMPONENTS_FLAG_BLACKLIST)
#define HWLOC_TOPOLOGY_COMPONENTS_FLAG_BLACKLIST HWLOC_NAME_CAPS(TOPOLOGY_COMPONENTS_FLAG_BLACKLIST)
# 定义宏，将 hwloc_topology_set_components 重命名为 HWLOC_NAME(topology_set_components)
#define hwloc_topology_set_components HWLOC_NAME(topology_set_components)

# 定义宏，将 hwloc_topology_set_flags 重命名为 HWLOC_NAME(topology_set_flags)
#define hwloc_topology_set_flags HWLOC_NAME(topology_set_flags)
# 定义宏，将 hwloc_topology_is_thissystem 重命名为 HWLOC_NAME(topology_is_thissystem)
#define hwloc_topology_is_thissystem HWLOC_NAME(topology_is_thissystem)
# 定义宏，将 hwloc_topology_get_flags 重命名为 HWLOC_NAME(topology_get_flags)
#define hwloc_topology_get_flags HWLOC_NAME(topology_get_flags)
# 定义宏，将 hwloc_topology_discovery_support 重命名为 HWLOC_NAME(topology_discovery_support)
#define hwloc_topology_discovery_support HWLOC_NAME(topology_discovery_support)
# 定义宏，将 hwloc_topology_cpubind_support 重命名为 HWLOC_NAME(topology_cpubind_support)
#define hwloc_topology_cpubind_support HWLOC_NAME(topology_cpubind_support)
# 定义宏，将 hwloc_topology_membind_support 重命名为 HWLOC_NAME(topology_membind_support)
#define hwloc_topology_membind_support HWLOC_NAME(topology_membind_support)
# 定义宏，将 hwloc_topology_misc_support 重命名为 HWLOC_NAME(topology_misc_support)
#define hwloc_topology_misc_support HWLOC_NAME(topology_misc_support)
# 定义宏，将 hwloc_topology_support 重命名为 HWLOC_NAME(topology_support)
#define hwloc_topology_support HWLOC_NAME(topology_support)
# 定义宏，将 hwloc_topology_get_support 重命名为 HWLOC_NAME(topology_get_support)
#define hwloc_topology_get_support HWLOC_NAME(topology_get_support)

# 定义宏，将 hwloc_type_filter_e 重命名为 HWLOC_NAME(type_filter_e)
#define hwloc_type_filter_e HWLOC_NAME(type_filter_e)
# 定义宏，将 HWLOC_TYPE_FILTER_KEEP_ALL 重命名为 HWLOC_NAME_CAPS(TYPE_FILTER_KEEP_ALL)
#define HWLOC_TYPE_FILTER_KEEP_ALL HWLOC_NAME_CAPS(TYPE_FILTER_KEEP_ALL)
# 定义宏，将 HWLOC_TYPE_FILTER_KEEP_NONE 重命名为 HWLOC_NAME_CAPS(TYPE_FILTER_KEEP_NONE)
#define HWLOC_TYPE_FILTER_KEEP_NONE HWLOC_NAME_CAPS(TYPE_FILTER_KEEP_NONE)
# 定义宏，将 HWLOC_TYPE_FILTER_KEEP_STRUCTURE 重命名为 HWLOC_NAME_CAPS(TYPE_FILTER_KEEP_STRUCTURE)
#define HWLOC_TYPE_FILTER_KEEP_STRUCTURE HWLOC_NAME_CAPS(TYPE_FILTER_KEEP_STRUCTURE)
# 定义宏，将 HWLOC_TYPE_FILTER_KEEP_IMPORTANT 重命名为 HWLOC_NAME_CAPS(TYPE_FILTER_KEEP_IMPORTANT)
#define HWLOC_TYPE_FILTER_KEEP_IMPORTANT HWLOC_NAME_CAPS(TYPE_FILTER_KEEP_IMPORTANT)
# 定义宏，将 hwloc_topology_set_type_filter 重命名为 HWLOC_NAME(topology_set_type_filter)
#define hwloc_topology_set_type_filter HWLOC_NAME(topology_set_type_filter)
# 定义宏，将 hwloc_topology_get_type_filter 重命名为 HWLOC_NAME(topology_get_type_filter)
#define hwloc_topology_get_type_filter HWLOC_NAME(topology_get_type_filter)
# 定义宏，将 hwloc_topology_set_all_types_filter 重命名为 HWLOC_NAME(topology_set_all_types_filter)
#define hwloc_topology_set_all_types_filter HWLOC_NAME(topology_set_all_types_filter)
# 定义宏，将 hwloc_topology_set_cache_types_filter 重命名为 HWLOC_NAME(topology_set_cache_types_filter)
#define hwloc_topology_set_cache_types_filter HWLOC_NAME(topology_set_cache_types_filter)
# 定义宏，将 hwloc_topology_set_icache_types_filter 重命名为 HWLOC_NAME(topology_set_icache_types_filter)
#define hwloc_topology_set_icache_types_filter HWLOC_NAME(topology_set_icache_types_filter)
# 定义宏，将 hwloc_topology_set_io_types_filter 重命名为 HWLOC_NAME(topology_set_io_types_filter)
#define hwloc_topology_set_io_types_filter HWLOC_NAME(topology_set_io_types_filter)

# 定义宏，将 hwloc_topology_set_userdata 重命名为 HWLOC_NAME(topology_set_userdata)
#define hwloc_topology_set_userdata HWLOC_NAME(topology_set_userdata)
# 定义获取用户数据的宏
#define hwloc_topology_get_userdata HWLOC_NAME(topology_get_userdata)

# 定义限制标志的枚举类型
#define hwloc_restrict_flags_e HWLOC_NAME(restrict_flags_e)
#define HWLOC_RESTRICT_FLAG_REMOVE_CPULESS HWLOC_NAME_CAPS(RESTRICT_FLAG_REMOVE_CPULESS)
#define HWLOC_RESTRICT_FLAG_BYNODESET HWLOC_NAME_CAPS(RESTRICT_FLAG_BYNODESET)
#define HWLOC_RESTRICT_FLAG_REMOVE_MEMLESS HWLOC_NAME_CAPS(RESTRICT_FLAG_REMOVE_MEMLESS)
#define HWLOC_RESTRICT_FLAG_ADAPT_MISC HWLOC_NAME_CAPS(RESTRICT_FLAG_ADAPT_MISC)
#define HWLOC_RESTRICT_FLAG_ADAPT_IO HWLOC_NAME_CAPS(RESTRICT_FLAG_ADAPT_IO)
#define hwloc_topology_restrict HWLOC_NAME(topology_restrict)

# 定义允许标志的枚举类型
#define hwloc_allow_flags_e HWLOC_NAME(allow_flags_e)
#define HWLOC_ALLOW_FLAG_ALL HWLOC_NAME_CAPS(ALLOW_FLAG_ALL)
#define HWLOC_ALLOW_FLAG_LOCAL_RESTRICTIONS HWLOC_NAME_CAPS(ALLOW_FLAG_LOCAL_RESTRICTIONS)
#define HWLOC_ALLOW_FLAG_CUSTOM HWLOC_NAME_CAPS(ALLOW_FLAG_CUSTOM)
#define hwloc_topology_allow HWLOC_NAME(topology_allow)

# 定义插入杂项对象、分配组对象、插入组对象、添加其他对象集合、刷新拓扑结构的函数
#define hwloc_topology_insert_misc_object HWLOC_NAME(topology_insert_misc_object)
#define hwloc_topology_alloc_group_object HWLOC_NAME(topology_alloc_group_object)
#define hwloc_topology_insert_group_object HWLOC_NAME(topology_insert_group_object)
#define hwloc_obj_add_other_obj_sets HWLOC_NAME(obj_add_other_obj_sets)
#define hwloc_topology_refresh HWLOC_NAME(topology_refresh)

# 定义获取拓扑结构深度、获取类型深度、获取内存父级深度的函数
#define hwloc_topology_get_depth HWLOC_NAME(topology_get_depth)
#define hwloc_get_type_depth HWLOC_NAME(get_type_depth)
#define hwloc_get_memory_parents_depth HWLOC_NAME(get_memory_parents_depth)

# 定义获取类型深度的枚举类型
#define hwloc_get_type_depth_e HWLOC_NAME(get_type_depth_e)
#define HWLOC_TYPE_DEPTH_UNKNOWN HWLOC_NAME_CAPS(TYPE_DEPTH_UNKNOWN)
#define HWLOC_TYPE_DEPTH_MULTIPLE HWLOC_NAME_CAPS(TYPE_DEPTH_MULTIPLE)
#define HWLOC_TYPE_DEPTH_BRIDGE HWLOC_NAME_CAPS(TYPE_DEPTH_BRIDGE)
#define HWLOC_TYPE_DEPTH_PCI_DEVICE HWLOC_NAME_CAPS(TYPE_DEPTH_PCI_DEVICE)
#define HWLOC_TYPE_DEPTH_OS_DEVICE HWLOC_NAME_CAPS(TYPE_DEPTH_OS_DEVICE)
#define HWLOC_TYPE_DEPTH_MISC HWLOC_NAME_CAPS(TYPE_DEPTH_MISC)
// 定义获取 NUMANODE 类型深度的宏
#define HWLOC_TYPE_DEPTH_NUMANODE HWLOC_NAME_CAPS(TYPE_DEPTH_NUMANODE)
// 定义获取 MEMCACHE 类型深度的宏
#define HWLOC_TYPE_DEPTH_MEMCACHE HWLOC_NAME_CAPS(TYPE_DEPTH_MEMCACHE)

// 定义获取深度类型的函数
#define hwloc_get_depth_type HWLOC_NAME(get_depth_type)
// 定义获取指定深度对象数量的函数
#define hwloc_get_nbobjs_by_depth HWLOC_NAME(get_nbobjs_by_depth)
// 定义获取指定类型对象数量的函数
#define hwloc_get_nbobjs_by_type HWLOC_NAME(get_nbobjs_by_type)

// 定义根据深度获取对象的函数
#define hwloc_get_obj_by_depth HWLOC_NAME(get_obj_by_depth )
// 定义根据类型获取对象的函数
#define hwloc_get_obj_by_type HWLOC_NAME(get_obj_by_type )

// 定义获取对象类型字符串的函数
#define hwloc_obj_type_string HWLOC_NAME(obj_type_string )
// 定义格式化对象类型字符串的函数
#define hwloc_obj_type_snprintf HWLOC_NAME(obj_type_snprintf )
// 定义格式化对象属性字符串的函数
#define hwloc_obj_attr_snprintf HWLOC_NAME(obj_attr_snprintf )
// 定义将字符串转换为类型的函数
#define hwloc_type_sscanf HWLOC_NAME(type_sscanf)
// 定义将字符串转换为深度的函数
#define hwloc_type_sscanf_as_depth HWLOC_NAME(type_sscanf_as_depth)

// 定义根据名称获取对象信息的函数
#define hwloc_obj_get_info_by_name HWLOC_NAME(obj_get_info_by_name)
// 定义添加对象信息的函数
#define hwloc_obj_add_info HWLOC_NAME(obj_add_info)

// 定义绑定进程的标记
#define HWLOC_CPUBIND_PROCESS HWLOC_NAME_CAPS(CPUBIND_PROCESS)
// 定义绑定线程的标记
#define HWLOC_CPUBIND_THREAD HWLOC_NAME_CAPS(CPUBIND_THREAD)
// 定义严格绑定的标记
#define HWLOC_CPUBIND_STRICT HWLOC_NAME_CAPS(CPUBIND_STRICT)
// 定义不绑定内存的标记
#define HWLOC_CPUBIND_NOMEMBIND HWLOC_NAME_CAPS(CPUBIND_NOMEMBIND)

// 定义 CPU 绑定标记类型
#define hwloc_cpubind_flags_t HWLOC_NAME(cpubind_flags_t)

// 定义设置 CPU 绑定的函数
#define hwloc_set_cpubind HWLOC_NAME(set_cpubind)
// 定义获取 CPU 绑定的函数
#define hwloc_get_cpubind HWLOC_NAME(get_cpubind)
// 定义设置进程 CPU 绑定的函数
#define hwloc_set_proc_cpubind HWLOC_NAME(set_proc_cpubind)
// 定义获取进程 CPU 绑定的函数
#define hwloc_get_proc_cpubind HWLOC_NAME(get_proc_cpubind)
// 定义设置线程 CPU 绑定的函数
#define hwloc_set_thread_cpubind HWLOC_NAME(set_thread_cpubind)
// 定义获取线程 CPU 绑定的函数
#define hwloc_get_thread_cpubind HWLOC_NAME(get_thread_cpubind)

// 定义获取最后 CPU 位置的函数
#define hwloc_get_last_cpu_location HWLOC_NAME(get_last_cpu_location)
// 定义获取进程最后 CPU 位置的函数
#define hwloc_get_proc_last_cpu_location HWLOC_NAME(get_proc_last_cpu_location)

// 定义内存绑定默认标记
#define HWLOC_MEMBIND_DEFAULT HWLOC_NAME_CAPS(MEMBIND_DEFAULT)
// 定义内存绑定首次访问标记
#define HWLOC_MEMBIND_FIRSTTOUCH HWLOC_NAME_CAPS(MEMBIND_FIRSTTOUCH)
// 定义内存绑定绑定标记
#define HWLOC_MEMBIND_BIND HWLOC_NAME_CAPS(MEMBIND_BIND)
// 定义内存绑定交错标记
#define HWLOC_MEMBIND_INTERLEAVE HWLOC_NAME_CAPS(MEMBIND_INTERLEAVE)
// 定义 HWLOC_MEMBIND_NEXTTOUCH 为 MEMBIND_NEXTTOUCH 的大写形式
#define HWLOC_MEMBIND_NEXTTOUCH HWLOC_NAME_CAPS(MEMBIND_NEXTTOUCH)
// 定义 HWLOC_MEMBIND_MIXED 为 MEMBIND_MIXED 的大写形式
#define HWLOC_MEMBIND_MIXED HWLOC_NAME_CAPS(MEMBIND_MIXED)

// 定义 hwloc_membind_policy_t 为 membind_policy_t
#define hwloc_membind_policy_t HWLOC_NAME(membind_policy_t)

// 定义 HWLOC_MEMBIND_PROCESS 为 MEMBIND_PROCESS 的大写形式
#define HWLOC_MEMBIND_PROCESS HWLOC_NAME_CAPS(MEMBIND_PROCESS)
// 定义 HWLOC_MEMBIND_THREAD 为 MEMBIND_THREAD 的大写形式
#define HWLOC_MEMBIND_THREAD HWLOC_NAME_CAPS(MEMBIND_THREAD)
// 定义 HWLOC_MEMBIND_STRICT 为 MEMBIND_STRICT 的大写形式
#define HWLOC_MEMBIND_STRICT HWLOC_NAME_CAPS(MEMBIND_STRICT)
// 定义 HWLOC_MEMBIND_MIGRATE 为 MEMBIND_MIGRATE 的大写形式
#define HWLOC_MEMBIND_MIGRATE HWLOC_NAME_CAPS(MEMBIND_MIGRATE)
// 定义 HWLOC_MEMBIND_NOCPUBIND 为 MEMBIND_NOCPUBIND 的大写形式
#define HWLOC_MEMBIND_NOCPUBIND HWLOC_NAME_CAPS(MEMBIND_NOCPUBIND)
// 定义 HWLOC_MEMBIND_BYNODESET 为 MEMBIND_BYNODESET 的大写形式
#define HWLOC_MEMBIND_BYNODESET HWLOC_NAME_CAPS(MEMBIND_BYNODESET)

// 定义 hwloc_membind_flags_t 为 membind_flags_t
#define hwloc_membind_flags_t HWLOC_NAME(membind_flags_t)

// 定义 hwloc_set_membind 为 set_membind
#define hwloc_set_membind HWLOC_NAME(set_membind)
// 定义 hwloc_get_membind 为 get_membind
#define hwloc_get_membind HWLOC_NAME(get_membind)
// 定义 hwloc_set_proc_membind 为 set_proc_membind
#define hwloc_set_proc_membind HWLOC_NAME(set_proc_membind)
// 定义 hwloc_get_proc_membind 为 get_proc_membind
#define hwloc_get_proc_membind HWLOC_NAME(get_proc_membind)
// 定义 hwloc_set_area_membind 为 set_area_membind
#define hwloc_set_area_membind HWLOC_NAME(set_area_membind)
// 定义 hwloc_get_area_membind 为 get_area_membind
#define hwloc_get_area_membind HWLOC_NAME(get_area_membind)
// 定义 hwloc_get_area_memlocation 为 get_area_memlocation
#define hwloc_get_area_memlocation HWLOC_NAME(get_area_memlocation)
// 定义 hwloc_alloc_membind 为 alloc_membind
#define hwloc_alloc_membind HWLOC_NAME(alloc_membind)
// 定义 hwloc_alloc 为 alloc
#define hwloc_alloc HWLOC_NAME(alloc)
// 定义 hwloc_free 为 free
#define hwloc_free HWLOC_NAME(free)

// 定义 hwloc_get_non_io_ancestor_obj 为 get_non_io_ancestor_obj
#define hwloc_get_non_io_ancestor_obj HWLOC_NAME(get_non_io_ancestor_obj)
// 定义 hwloc_get_next_pcidev 为 get_next_pcidev
#define hwloc_get_next_pcidev HWLOC_NAME(get_next_pcidev)
// 定义 hwloc_get_pcidev_by_busid 为 get_pcidev_by_busid
#define hwloc_get_pcidev_by_busid HWLOC_NAME(get_pcidev_by_busid)
// 定义 hwloc_get_pcidev_by_busidstring 为 get_pcidev_by_busidstring
#define hwloc_get_pcidev_by_busidstring HWLOC_NAME(get_pcidev_by_busidstring)
// 定义 hwloc_get_next_osdev 为 get_next_osdev
#define hwloc_get_next_osdev HWLOC_NAME(get_next_osdev)
// 定义 hwloc_get_next_bridge 为 get_next_bridge
#define hwloc_get_next_bridge HWLOC_NAME(get_next_bridge)
// 定义 hwloc_bridge_covers_pcibus 为 bridge_covers_pcibus
#define hwloc_bridge_covers_pcibus HWLOC_NAME(bridge_covers_pcibus)

/* hwloc/bitmap.h */

// 定义 hwloc_bitmap_s 为 bitmap_s
#define hwloc_bitmap_s HWLOC_NAME(bitmap_s)
// 定义 hwloc_bitmap_t 为 bitmap_t
#define hwloc_bitmap_t HWLOC_NAME(bitmap_t)
// 定义 hwloc_const_bitmap_t 为 const_bitmap_t
#define hwloc_const_bitmap_t HWLOC_NAME(const_bitmap_t)

// 定义 hwloc_bitmap_alloc 为 bitmap_alloc
#define hwloc_bitmap_alloc HWLOC_NAME(bitmap_alloc)
// 定义 hwloc_bitmap_alloc_full 为 bitmap_alloc_full
#define hwloc_bitmap_alloc_full HWLOC_NAME(bitmap_alloc_full)
// 定义 hwloc_bitmap_free 为 bitmap_free
#define hwloc_bitmap_free HWLOC_NAME(bitmap_free)
// 定义宏，将 hwloc_bitmap_dup 重命名为 HWLOC_NAME(bitmap_dup)
#define hwloc_bitmap_dup HWLOC_NAME(bitmap_dup)
// 定义宏，将 hwloc_bitmap_copy 重命名为 HWLOC_NAME(bitmap_copy)
#define hwloc_bitmap_copy HWLOC_NAME(bitmap_copy)
// 定义宏，将 hwloc_bitmap_snprintf 重命名为 HWLOC_NAME(bitmap_snprintf)
#define hwloc_bitmap_snprintf HWLOC_NAME(bitmap_snprintf)
// 定义宏，将 hwloc_bitmap_asprintf 重命名为 HWLOC_NAME(bitmap_asprintf)
#define hwloc_bitmap_asprintf HWLOC_NAME(bitmap_asprintf)
// 定义宏，将 hwloc_bitmap_sscanf 重命名为 HWLOC_NAME(bitmap_sscanf)
#define hwloc_bitmap_sscanf HWLOC_NAME(bitmap_sscanf)
// 定义宏，将 hwloc_bitmap_list_snprintf 重命名为 HWLOC_NAME(bitmap_list_snprintf)
#define hwloc_bitmap_list_snprintf HWLOC_NAME(bitmap_list_snprintf)
// 定义宏，将 hwloc_bitmap_list_asprintf 重命名为 HWLOC_NAME(bitmap_list_asprintf)
#define hwloc_bitmap_list_asprintf HWLOC_NAME(bitmap_list_asprintf)
// 定义宏，将 hwloc_bitmap_list_sscanf 重命名为 HWLOC_NAME(bitmap_list_sscanf)
#define hwloc_bitmap_list_sscanf HWLOC_NAME(bitmap_list_sscanf)
// 定义宏，将 hwloc_bitmap_taskset_snprintf 重命名为 HWLOC_NAME(bitmap_taskset_snprintf)
#define hwloc_bitmap_taskset_snprintf HWLOC_NAME(bitmap_taskset_snprintf)
// 定义宏，将 hwloc_bitmap_taskset_asprintf 重命名为 HWLOC_NAME(bitmap_taskset_asprintf)
#define hwloc_bitmap_taskset_asprintf HWLOC_NAME(bitmap_taskset_asprintf)
// 定义宏，将 hwloc_bitmap_taskset_sscanf 重命名为 HWLOC_NAME(bitmap_taskset_sscanf)
#define hwloc_bitmap_taskset_sscanf HWLOC_NAME(bitmap_taskset_sscanf)
// 定义宏，将 hwloc_bitmap_zero 重命名为 HWLOC_NAME(bitmap_zero)
#define hwloc_bitmap_zero HWLOC_NAME(bitmap_zero)
// 定义宏，将 hwloc_bitmap_fill 重命名为 HWLOC_NAME(bitmap_fill)
#define hwloc_bitmap_fill HWLOC_NAME(bitmap_fill)
// 定义宏，将 hwloc_bitmap_from_ulong 重命名为 HWLOC_NAME(bitmap_from_ulong)
#define hwloc_bitmap_from_ulong HWLOC_NAME(bitmap_from_ulong)
// 定义宏，将 hwloc_bitmap_from_ulongs 重命名为 HWLOC_NAME(bitmap_from_ulongs)
#define hwloc_bitmap_from_ulongs HWLOC_NAME(bitmap_from_ulongs)
// 定义宏，将 hwloc_bitmap_from_ith_ulong 重命名为 HWLOC_NAME(bitmap_from_ith_ulong)
#define hwloc_bitmap_from_ith_ulong HWLOC_NAME(bitmap_from_ith_ulong)
// 定义宏，将 hwloc_bitmap_to_ulong 重命名为 HWLOC_NAME(bitmap_to_ulong)
#define hwloc_bitmap_to_ulong HWLOC_NAME(bitmap_to_ulong)
// 定义宏，将 hwloc_bitmap_to_ith_ulong 重命名为 HWLOC_NAME(bitmap_to_ith_ulong)
#define hwloc_bitmap_to_ith_ulong HWLOC_NAME(bitmap_to_ith_ulong)
// 定义宏，将 hwloc_bitmap_to_ulongs 重命名为 HWLOC_NAME(bitmap_to_ulongs)
#define hwloc_bitmap_to_ulongs HWLOC_NAME(bitmap_to_ulongs)
// 定义宏，将 hwloc_bitmap_nr_ulongs 重命名为 HWLOC_NAME(bitmap_nr_ulongs)
#define hwloc_bitmap_nr_ulongs HWLOC_NAME(bitmap_nr_ulongs)
// 定义宏，将 hwloc_bitmap_only 重命名为 HWLOC_NAME(bitmap_only)
#define hwloc_bitmap_only HWLOC_NAME(bitmap_only)
// 定义宏，将 hwloc_bitmap_allbut 重命名为 HWLOC_NAME(bitmap_allbut)
#define hwloc_bitmap_allbut HWLOC_NAME(bitmap_allbut)
// 定义宏，将 hwloc_bitmap_set 重命名为 HWLOC_NAME(bitmap_set)
#define hwloc_bitmap_set HWLOC_NAME(bitmap_set)
// 定义宏，将 hwloc_bitmap_set_range 重命名为 HWLOC_NAME(bitmap_set_range)
#define hwloc_bitmap_set_range HWLOC_NAME(bitmap_set_range)
// 定义宏，将 hwloc_bitmap_set_ith_ulong 重命名为 HWLOC_NAME(bitmap_set_ith_ulong)
#define hwloc_bitmap_set_ith_ulong HWLOC_NAME(bitmap_set_ith_ulong)
// 定义宏，将 hwloc_bitmap_clr 重命名为 HWLOC_NAME(bitmap_clr)
#define hwloc_bitmap_clr HWLOC_NAME(bitmap_clr)
// 定义宏，将 hwloc_bitmap_clr_range 重命名为 HWLOC_NAME(bitmap_clr_range)
#define hwloc_bitmap_clr_range HWLOC_NAME(bitmap_clr_range)
// 定义宏，将 hwloc_bitmap_isset 重命名为 HWLOC_NAME(bitmap_isset)
#define hwloc_bitmap_isset HWLOC_NAME(bitmap_isset)
// 定义宏，将 hwloc_bitmap_iszero 重命名为 HWLOC_NAME(bitmap_iszero)
#define hwloc_bitmap_iszero HWLOC_NAME(bitmap_iszero)
// 定义宏，将 hwloc_bitmap_isfull 重命名为 HWLOC_NAME(bitmap_isfull)
#define hwloc_bitmap_isfull HWLOC_NAME(bitmap_isfull)
// 定义宏，将 hwloc_bitmap_isequal 重命名为 HWLOC_NAME(bitmap_isequal)
#define hwloc_bitmap_isequal HWLOC_NAME(bitmap_isequal)
// 定义宏，将 hwloc_bitmap_intersects 重命名为 HWLOC_NAME(bitmap_intersects)
#define hwloc_bitmap_intersects HWLOC_NAME(bitmap_intersects)
// 定义宏，将 hwloc_bitmap_isincluded 重命名为 HWLOC_NAME(bitmap_isincluded)
#define hwloc_bitmap_isincluded HWLOC_NAME(bitmap_isincluded)
// 定义宏，将 hwloc_bitmap_or 重命名为 HWLOC_NAME(bitmap_or)
#define hwloc_bitmap_or HWLOC_NAME(bitmap_or)
/* 定义一系列宏，用于重命名 hwloc 库中的函数 */
#define hwloc_bitmap_and HWLOC_NAME(bitmap_and)  // 重命名函数 hwloc_bitmap_and
#define hwloc_bitmap_andnot HWLOC_NAME(bitmap_andnot)  // 重命名函数 hwloc_bitmap_andnot
#define hwloc_bitmap_xor HWLOC_NAME(bitmap_xor)  // 重命名函数 hwloc_bitmap_xor
#define hwloc_bitmap_not HWLOC_NAME(bitmap_not)  // 重命名函数 hwloc_bitmap_not
#define hwloc_bitmap_first HWLOC_NAME(bitmap_first)  // 重命名函数 hwloc_bitmap_first
#define hwloc_bitmap_last HWLOC_NAME(bitmap_last)  // 重命名函数 hwloc_bitmap_last
#define hwloc_bitmap_next HWLOC_NAME(bitmap_next)  // 重命名函数 hwloc_bitmap_next
#define hwloc_bitmap_first_unset HWLOC_NAME(bitmap_first_unset)  // 重命名函数 hwloc_bitmap_first_unset
#define hwloc_bitmap_last_unset HWLOC_NAME(bitmap_last_unset)  // 重命名函数 hwloc_bitmap_last_unset
#define hwloc_bitmap_next_unset HWLOC_NAME(bitmap_next_unset)  // 重命名函数 hwloc_bitmap_next_unset
#define hwloc_bitmap_singlify HWLOC_NAME(bitmap_singlify)  // 重命名函数 hwloc_bitmap_singlify
#define hwloc_bitmap_compare_first HWLOC_NAME(bitmap_compare_first)  // 重命名函数 hwloc_bitmap_compare_first
#define hwloc_bitmap_compare HWLOC_NAME(bitmap_compare)  // 重命名函数 hwloc_bitmap_compare
#define hwloc_bitmap_weight HWLOC_NAME(bitmap_weight)  // 重命名函数 hwloc_bitmap_weight

/* 以下是 hwloc/helper.h 中的函数重命名 */
#define hwloc_get_type_or_below_depth HWLOC_NAME(get_type_or_below_depth)  // 重命名函数 hwloc_get_type_or_below_depth
#define hwloc_get_type_or_above_depth HWLOC_NAME(get_type_or_above_depth)  // 重命名函数 hwloc_get_type_or_above_depth
#define hwloc_get_root_obj HWLOC_NAME(get_root_obj)  // 重命名函数 hwloc_get_root_obj
#define hwloc_get_ancestor_obj_by_depth HWLOC_NAME(get_ancestor_obj_by_depth)  // 重命名函数 hwloc_get_ancestor_obj_by_depth
#define hwloc_get_ancestor_obj_by_type HWLOC_NAME(get_ancestor_obj_by_type)  // 重命名函数 hwloc_get_ancestor_obj_by_type
#define hwloc_get_next_obj_by_depth HWLOC_NAME(get_next_obj_by_depth)  // 重命名函数 hwloc_get_next_obj_by_depth
#define hwloc_get_next_obj_by_type HWLOC_NAME(get_next_obj_by_type)  // 重命名函数 hwloc_get_next_obj_by_type
#define hwloc_bitmap_singlify_per_core HWLOC_NAME(bitmap_singlify_by_core)  // 重命名函数 hwloc_bitmap_singlify_per_core
#define hwloc_get_pu_obj_by_os_index HWLOC_NAME(get_pu_obj_by_os_index)  // 重命名函数 hwloc_get_pu_obj_by_os_index
#define hwloc_get_numanode_obj_by_os_index HWLOC_NAME(get_numanode_obj_by_os_index)  // 重命名函数 hwloc_get_numanode_obj_by_os_index
#define hwloc_get_next_child HWLOC_NAME(get_next_child)  // 重命名函数 hwloc_get_next_child
#define hwloc_get_common_ancestor_obj HWLOC_NAME(get_common_ancestor_obj)  // 重命名函数 hwloc_get_common_ancestor_obj
#define hwloc_obj_is_in_subtree HWLOC_NAME(obj_is_in_subtree)  // 重命名函数 hwloc_obj_is_in_subtree
#define hwloc_get_first_largest_obj_inside_cpuset HWLOC_NAME(get_first_largest_obj_inside_cpuset)  // 重命名函数 hwloc_get_first_largest_obj_inside_cpuset
#define hwloc_get_largest_objs_inside_cpuset HWLOC_NAME(get_largest_objs_inside_cpuset)  // 重命名函数 hwloc_get_largest_objs_inside_cpuset
#define hwloc_get_next_obj_inside_cpuset_by_depth HWLOC_NAME(get_next_obj_inside_cpuset_by_depth)  // 重命名函数 hwloc_get_next_obj_inside_cpuset_by_depth
# 定义获取指定类型内下一个对象的宏
#define hwloc_get_next_obj_inside_cpuset_by_type HWLOC_NAME(get_next_obj_inside_cpuset_by_type)
# 定义获取指定深度内对象的宏
#define hwloc_get_obj_inside_cpuset_by_depth HWLOC_NAME(get_obj_inside_cpuset_by_depth)
# 定义获取指定类型内对象的宏
#define hwloc_get_obj_inside_cpuset_by_type HWLOC_NAME(get_obj_inside_cpuset_by_type)
# 定义获取指定深度内对象数量的宏
#define hwloc_get_nbobjs_inside_cpuset_by_depth HWLOC_NAME(get_nbobjs_inside_cpuset_by_depth)
# 定义获取指定类型内对象数量的宏
#define hwloc_get_nbobjs_inside_cpuset_by_type HWLOC_NAME(get_nbobjs_inside_cpuset_by_type)
# 定义获取指定对象在指定集合内的索引的宏
#define hwloc_get_obj_index_inside_cpuset HWLOC_NAME(get_obj_index_inside_cpuset)
# 定义获取覆盖指定集合的子对象的宏
#define hwloc_get_child_covering_cpuset HWLOC_NAME(get_child_covering_cpuset)
# 定义获取覆盖指定集合的对象的宏
#define hwloc_get_obj_covering_cpuset HWLOC_NAME(get_obj_covering_cpuset)
# 定义获取下一个覆盖指定集合的对象的深度的宏
#define hwloc_get_next_obj_covering_cpuset_by_depth HWLOC_NAME(get_next_obj_covering_cpuset_by_depth)
# 定义获取下一个覆盖指定集合的对象的类型的宏
#define hwloc_get_next_obj_covering_cpuset_by_type HWLOC_NAME(get_next_obj_covering_cpuset_by_type)
# 定义判断对象类型是否为普通类型的宏
#define hwloc_obj_type_is_normal HWLOC_NAME(obj_type_is_normal)
# 定义判断对象类型是否为内存类型的宏
#define hwloc_obj_type_is_memory HWLOC_NAME(obj_type_is_memory)
# 定义判断对象类型是否为IO类型的宏
#define hwloc_obj_type_is_io HWLOC_NAME(obj_type_is_io)
# 定义判断对象类型是否为缓存类型的宏
#define hwloc_obj_type_is_cache HWLOC_NAME(obj_type_is_cache)
# 定义判断对象类型是否为数据缓存类型的宏
#define hwloc_obj_type_is_dcache HWLOC_NAME(obj_type_is_dcache)
# 定义判断对象类型是否为指令缓存类型的宏
#define hwloc_obj_type_is_icache HWLOC_NAME(obj_type_is_icache)
# 定义获取缓存类型的深度的宏
#define hwloc_get_cache_type_depth HWLOC_NAME(get_cache_type_depth)
# 定义获取覆盖指定集合的缓存对象的宏
#define hwloc_get_cache_covering_cpuset HWLOC_NAME(get_cache_covering_cpuset)
# 定义获取覆盖指定对象的共享缓存对象的宏
#define hwloc_get_shared_cache_covering_obj HWLOC_NAME(get_shared_cache_covering_obj)
# 定义获取最接近的对象的宏
#define hwloc_get_closest_objs HWLOC_NAME(get_closest_objs)
# 定义获取指定类型下方的对象的宏
#define hwloc_get_obj_below_by_type HWLOC_NAME(get_obj_below_by_type)
# 定义获取指定类型下方的对象数组的宏
#define hwloc_get_obj_below_array_by_type HWLOC_NAME(get_obj_below_array_by_type)
# 定义获取具有相同本地性的对象的宏
#define hwloc_get_obj_with_same_locality HWLOC_NAME(get_obj_with_same_locality)
# 定义分布标志的枚举类型
#define hwloc_distrib_flags_e HWLOC_NAME(distrib_flags_e)
# 定义反向分布标志的宏
#define HWLOC_DISTRIB_FLAG_REVERSE HWLOC_NAME_CAPS(DISTRIB_FLAG_REVERSE)
# 定义分布的宏
#define hwloc_distrib HWLOC_NAME(distrib)
// 定义内存绑定策略的宏
#define hwloc_alloc_membind_policy HWLOC_NAME(alloc_membind_policy)
#define hwloc_alloc_membind_policy_nodeset HWLOC_NAME(alloc_membind_policy_nodeset)
// 获取拓扑结构中的完整 CPU 集合
#define hwloc_topology_get_complete_cpuset HWLOC_NAME(topology_get_complete_cpuset)
#define hwloc_topology_get_topology_cpuset HWLOC_NAME(topology_get_topology_cpuset)
#define hwloc_topology_get_allowed_cpuset HWLOC_NAME(topology_get_allowed_cpuset)
// 获取拓扑结构中的完整节点集合
#define hwloc_topology_get_complete_nodeset HWLOC_NAME(topology_get_complete_nodeset)
#define hwloc_topology_get_topology_nodeset HWLOC_NAME(topology_get_topology_nodeset)
#define hwloc_topology_get_allowed_nodeset HWLOC_NAME(topology_get_allowed_nodeset)
// 将 CPU 集合转换为节点集合
#define hwloc_cpuset_to_nodeset HWLOC_NAME(cpuset_to_nodeset)
// 将节点集合转换为 CPU 集合
#define hwloc_cpuset_from_nodeset HWLOC_NAME(cpuset_from_nodeset)

/* memattrs.h */

// 定义内存属性 ID 的枚举类型
#define hwloc_memattr_id_e HWLOC_NAME(memattr_id_e)
// 定义内存属性 ID 的具体值
#define HWLOC_MEMATTR_ID_CAPACITY HWLOC_NAME_CAPS(MEMATTR_ID_CAPACITY)
#define HWLOC_MEMATTR_ID_LOCALITY HWLOC_NAME_CAPS(MEMATTR_ID_LOCALITY)
#define HWLOC_MEMATTR_ID_BANDWIDTH HWLOC_NAME_CAPS(MEMATTR_ID_BANDWIDTH)
#define HWLOC_MEMATTR_ID_LATENCY HWLOC_NAME_CAPS(MEMATTR_ID_LATENCY)
#define HWLOC_MEMATTR_ID_READ_BANDWIDTH HWLOC_NAME_CAPS(MEMATTR_ID_READ_BANDWIDTH)
#define HWLOC_MEMATTR_ID_WRITE_BANDWIDTH HWLOC_NAME_CAPS(MEMATTR_ID_WRITE_BANDWIDTH)
#define HWLOC_MEMATTR_ID_READ_LATENCY HWLOC_NAME_CAPS(MEMATTR_ID_READ_LATENCY)
#define HWLOC_MEMATTR_ID_WRITE_LATENCY HWLOC_NAME_CAPS(MEMATTR_ID_WRITE_LATENCY)
#define HWLOC_MEMATTR_ID_MAX HWLOC_NAME_CAPS(MEMATTR_ID_MAX)
// 定义内存属性 ID 的类型
#define hwloc_memattr_id_t HWLOC_NAME(memattr_id_t)
// 根据名称获取内存属性 ID
#define hwloc_memattr_get_by_name HWLOC_NAME(memattr_get_by_name)
// 定义位置信息的宏
#define hwloc_location HWLOC_NAME(location)
// 定义位置信息的类型枚举
#define hwloc_location_type_e HWLOC_NAME(location_type_e)
#define HWLOC_LOCATION_TYPE_OBJECT HWLOC_NAME_CAPS(LOCATION_TYPE_OBJECT)
#define HWLOC_LOCATION_TYPE_CPUSET HWLOC_NAME_CAPS(LOCATION_TYPE_CPUSET)
// 定义位置信息的联合体
#define hwloc_location_u HWLOC_NAME(location_u)
// 定义宏，用于获取内存属性值
#define hwloc_memattr_get_value HWLOC_NAME(memattr_get_value)
// 定义宏，用于获取最佳的内存属性目标
#define hwloc_memattr_get_best_target HWLOC_NAME(memattr_get_best_target)
// 定义宏，用于获取最佳的内存属性发起者
#define hwloc_memattr_get_best_initiator HWLOC_NAME(memattr_get_best_initiator)

// 定义枚举类型，表示本地 NUMA 节点标志
#define hwloc_local_numanode_flag_e HWLOC_NAME(local_numanode_flag_e)
// 定义宏，表示较大的本地性
#define HWLOC_LOCAL_NUMANODE_FLAG_LARGER_LOCALITY HWLOC_NAME_CAPS(LOCAL_NUMANODE_FLAG_LARGER_LOCALITY)
// 定义宏，表示较小的本地性
#define HWLOC_LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY HWLOC_NAME_CAPS(LOCAL_NUMANODE_FLAG_SMALLER_LOCALITY)
// 定义宏，表示所有本地性
#define HWLOC_LOCAL_NUMANODE_FLAG_ALL HWLOC_NAME_CAPS(LOCAL_NUMANODE_FLAG_ALL)
// 定义宏，用于获取本地 NUMA 节点对象
#define hwloc_get_local_numanode_objs HWLOC_NAME(get_local_numanode_objs)

// 定义宏，用于获取内存属性名称
#define hwloc_memattr_get_name HWLOC_NAME(memattr_get_name)
// 定义宏，用于获取内存属性标志
#define hwloc_memattr_get_flags HWLOC_NAME(memattr_get_flags)
// 定义枚举类型，表示内存属性标志
#define hwloc_memattr_flag_e HWLOC_NAME(memattr_flag_e)
// 定义宏，表示较高的优先级
#define HWLOC_MEMATTR_FLAG_HIGHER_FIRST HWLOC_NAME_CAPS(MEMATTR_FLAG_HIGHER_FIRST)
// 定义宏，表示较低的优先级
#define HWLOC_MEMATTR_FLAG_LOWER_FIRST HWLOC_NAME_CAPS(MEMATTR_FLAG_LOWER_FIRST)
// 定义宏，表示需要发起者
#define HWLOC_MEMATTR_FLAG_NEED_INITIATOR HWLOC_NAME_CAPS(MEMATTR_FLAG_NEED_INITIATOR)
// 定义宏，用于注册内存属性
#define hwloc_memattr_register HWLOC_NAME(memattr_register)
// 定义宏，用于设置内存属性值
#define hwloc_memattr_set_value HWLOC_NAME(memattr_set_value)
// 定义宏，用于获取内存属性目标
#define hwloc_memattr_get_targets HWLOC_NAME(memattr_get_targets)
// 定义宏，用于获取内存属性发起者
#define hwloc_memattr_get_initiators HWLOC_NAME(memattr_get_initiators)

// cpukinds.h

// 定义宏，用于获取 CPU 种类数量
#define hwloc_cpukinds_get_nr HWLOC_NAME(cpukinds_get_nr)
// 定义宏，用于根据 CPU 集获取 CPU 种类
#define hwloc_cpukinds_get_by_cpuset HWLOC_NAME(cpukinds_get_by_cpuset)
// 定义宏，用于获取 CPU 种类信息
#define hwloc_cpukinds_get_info HWLOC_NAME(cpukinds_get_info)
// 定义宏，用于注册 CPU 种类
#define hwloc_cpukinds_register HWLOC_NAME(cpukinds_register)

// export.h

// 定义枚举类型，表示拓扑结构导出 XML 标志
#define hwloc_topology_export_xml_flags_e HWLOC_NAME(topology_export_xml_flags_e)
// 定义宏，表示拓扑结构导出 XML 标志 V1
#define HWLOC_TOPOLOGY_EXPORT_XML_FLAG_V1 HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_XML_FLAG_V1)
// 定义宏，用于导出拓扑结构到 XML 文件
#define hwloc_topology_export_xml HWLOC_NAME(topology_export_xml)
// 定义宏，用于导出拓扑结构到 XML 缓冲区
#define hwloc_topology_export_xmlbuffer HWLOC_NAME(topology_export_xmlbuffer)
// 定义宏，用于释放 XML 缓冲区
#define hwloc_free_xmlbuffer HWLOC_NAME(free_xmlbuffer)
// 定义 hwloc_topology_set_userdata_export_callback 为 HWLOC_NAME(topology_set_userdata_export_callback)
#define hwloc_topology_set_userdata_export_callback HWLOC_NAME(topology_set_userdata_export_callback)
// 定义 hwloc_export_obj_userdata 为 HWLOC_NAME(export_obj_userdata)
#define hwloc_export_obj_userdata HWLOC_NAME(export_obj_userdata)
// 定义 hwloc_export_obj_userdata_base64 为 HWLOC_NAME(export_obj_userdata_base64)
#define hwloc_export_obj_userdata_base64 HWLOC_NAME(export_obj_userdata_base64)
// 定义 hwloc_topology_set_userdata_import_callback 为 HWLOC_NAME(topology_set_userdata_import_callback)
#define hwloc_topology_set_userdata_import_callback HWLOC_NAME(topology_set_userdata_import_callback)

// 定义 hwloc_topology_export_synthetic_flags_e 为 HWLOC_NAME(topology_export_synthetic_flags_e)
#define hwloc_topology_export_synthetic_flags_e HWLOC_NAME(topology_export_synthetic_flags_e)
// 定义 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES 为 HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES)
#define HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_EXTENDED_TYPES)
// 定义 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS 为 HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS)
#define HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_SYNTHETIC_FLAG_NO_ATTRS)
// 定义 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1 为 HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1)
#define HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1 HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_SYNTHETIC_FLAG_V1)
// 定义 HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY 为 HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)
#define HWLOC_TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY HWLOC_NAME_CAPS(TOPOLOGY_EXPORT_SYNTHETIC_FLAG_IGNORE_MEMORY)
// 定义 hwloc_topology_export_synthetic 为 HWLOC_NAME(topology_export_synthetic)
#define hwloc_topology_export_synthetic HWLOC_NAME(topology_export_synthetic)

// distances.h

// 定义 hwloc_distances_s 为 HWLOC_NAME(distances_s)
#define hwloc_distances_s HWLOC_NAME(distances_s)

// 定义 hwloc_distances_kind_e 为 HWLOC_NAME(distances_kind_e)
#define hwloc_distances_kind_e HWLOC_NAME(distances_kind_e)
// 定义 HWLOC_DISTANCES_KIND_FROM_OS 为 HWLOC_NAME_CAPS(DISTANCES_KIND_FROM_OS)
#define HWLOC_DISTANCES_KIND_FROM_OS HWLOC_NAME_CAPS(DISTANCES_KIND_FROM_OS)
// 定义 HWLOC_DISTANCES_KIND_FROM_USER 为 HWLOC_NAME_CAPS(DISTANCES_KIND_FROM_USER)
#define HWLOC_DISTANCES_KIND_FROM_USER HWLOC_NAME_CAPS(DISTANCES_KIND_FROM_USER)
// 定义 HWLOC_DISTANCES_KIND_MEANS_LATENCY 为 HWLOC_NAME_CAPS(DISTANCES_KIND_MEANS_LATENCY)
#define HWLOC_DISTANCES_KIND_MEANS_LATENCY HWLOC_NAME_CAPS(DISTANCES_KIND_MEANS_LATENCY)
// 定义 HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH 为 HWLOC_NAME_CAPS(DISTANCES_KIND_MEANS_BANDWIDTH)
#define HWLOC_DISTANCES_KIND_MEANS_BANDWIDTH HWLOC_NAME_CAPS(DISTANCES_KIND_MEANS_BANDWIDTH)
// 定义 HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES 为 HWLOC_NAME_CAPS(DISTANCES_KIND_HETEROGENEOUS_TYPES)
#define HWLOC_DISTANCES_KIND_HETEROGENEOUS_TYPES HWLOC_NAME_CAPS(DISTANCES_KIND_HETEROGENEOUS_TYPES)

// 定义 hwloc_distances_get 为 HWLOC_NAME(distances_get)
#define hwloc_distances_get HWLOC_NAME(distances_get)
// 定义 hwloc_distances_get_by_depth 为 HWLOC_NAME(distances_get_by_depth)
#define hwloc_distances_get_by_depth HWLOC_NAME(distances_get_by_depth)
// 定义 hwloc_distances_get_by_type 为 HWLOC_NAME(distances_get_by_type)
#define hwloc_distances_get_by_type HWLOC_NAME(distances_get_by_type)
// 定义 hwloc_distances_get_by_name 为 HWLOC_NAME(distances_get_by_name)
#define hwloc_distances_get_by_name HWLOC_NAME(distances_get_by_name)
// 定义 hwloc_distances_get_name 为 HWLOC_NAME(distances_get_name)
#define hwloc_distances_get_name HWLOC_NAME(distances_get_name)
// 定义 hwloc_distances_release 为 HWLOC_NAME(distances_release)
#define hwloc_distances_release HWLOC_NAME(distances_release)
// 定义宏，用于获取距离对象的索引
#define hwloc_distances_obj_index HWLOC_NAME(distances_obj_index)
// 定义宏，用于获取距离对象对的值
#define hwloc_distances_obj_pair_values HWLOC_NAME(distances_pair_values)

// 定义宏，用于转换距离对象
#define hwloc_distances_transform_e HWLOC_NAME(distances_transform_e)
#define HWLOC_DISTANCES_TRANSFORM_REMOVE_NULL HWLOC_NAME_CAPS(DISTANCES_TRANSFORM_REMOVE_NULL)
#define HWLOC_DISTANCES_TRANSFORM_LINKS HWLOC_NAME_CAPS(DISTANCES_TRANSFORM_LINKS)
#define HWLOC_DISTANCES_TRANSFORM_MERGE_SWITCH_PORTS HWLOC_NAME_CAPS(DISTANCES_TRANSFORM_MERGE_SWITCH_PORTS)
#define HWLOC_DISTANCES_TRANSFORM_TRANSITIVE_CLOSURE HWLOC_NAME_CAPS(DISTANCES_TRANSFORM_TRANSITIVE_CLOSURE)
#define hwloc_distances_transform HWLOC_NAME(distances_transform)

// 定义宏，用于添加距离对象的标志
#define hwloc_distances_add_flag_e HWLOC_NAME(distances_add_flag_e)
#define HWLOC_DISTANCES_ADD_FLAG_GROUP HWLOC_NAME_CAPS(DISTANCES_ADD_FLAG_GROUP)
#define HWLOC_DISTANCES_ADD_FLAG_GROUP_INACCURATE HWLOC_NAME_CAPS(DISTANCES_ADD_FLAG_GROUP_INACCURATE)

// 定义宏，用于处理距离对象的句柄
#define hwloc_distances_add_handle_t HWLOC_NAME(distances_add_handle_t)
#define hwloc_distances_add_create HWLOC_NAME(distances_add_create)
#define hwloc_distances_add_values HWLOC_NAME(distances_add_values)
#define hwloc_distances_add_commit HWLOC_NAME(distances_add_commit)

// 定义宏，用于移除距离对象
#define hwloc_distances_remove HWLOC_NAME(distances_remove)
#define hwloc_distances_remove_by_depth HWLOC_NAME(distances_remove_by_depth)
#define hwloc_distances_remove_by_type HWLOC_NAME(distances_remove_by_type)
#define hwloc_distances_release_remove HWLOC_NAME(distances_release_remove)

// 定义宏，用于处理拓扑差异对象属性的类型
#define hwloc_topology_diff_obj_attr_type_e HWLOC_NAME(topology_diff_obj_attr_type_e)
#define hwloc_topology_diff_obj_attr_type_t HWLOC_NAME(topology_diff_obj_attr_type_t)
#define HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_SIZE HWLOC_NAME_CAPS(TOPOLOGY_DIFF_OBJ_ATTR_SIZE)
#define HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_NAME HWLOC_NAME_CAPS(TOPOLOGY_DIFF_OBJ_ATTR_NAME)
#define HWLOC_TOPOLOGY_DIFF_OBJ_ATTR_INFO HWLOC_NAME_CAPS(TOPOLOGY_DIFF_OBJ_ATTR_INFO)
// 定义一系列宏，用于重命名 hwloc 库中的一些数据结构和函数
#define hwloc_topology_diff_obj_attr_u HWLOC_NAME(topology_diff_obj_attr_u)
#define hwloc_topology_diff_obj_attr_generic_s HWLOC_NAME(topology_diff_obj_attr_generic_s)
#define hwloc_topology_diff_obj_attr_uint64_s HWLOC_NAME(topology_diff_obj_attr_uint64_s)
#define hwloc_topology_diff_obj_attr_string_s HWLOC_NAME(topology_diff_obj_attr_string_s)
#define hwloc_topology_diff_type_e HWLOC_NAME(topology_diff_type_e)
#define hwloc_topology_diff_type_t HWLOC_NAME(topology_diff_type_t)
#define HWLOC_TOPOLOGY_DIFF_OBJ_ATTR HWLOC_NAME_CAPS(TOPOLOGY_DIFF_OBJ_ATTR)
#define HWLOC_TOPOLOGY_DIFF_TOO_COMPLEX HWLOC_NAME_CAPS(TOPOLOGY_DIFF_TOO_COMPLEX)
#define hwloc_topology_diff_u HWLOC_NAME(topology_diff_u)
#define hwloc_topology_diff_t HWLOC_NAME(topology_diff_t)
#define hwloc_topology_diff_generic_s HWLOC_NAME(topology_diff_generic_s)
#define hwloc_topology_diff_obj_attr_s HWLOC_NAME(topology_diff_obj_attr_s)
#define hwloc_topology_diff_too_complex_s HWLOC_NAME(topology_diff_too_complex_s)
#define hwloc_topology_diff_build HWLOC_NAME(topology_diff_build)
#define hwloc_topology_diff_apply_flags_e HWLOC_NAME(topology_diff_apply_flags_e)
#define HWLOC_TOPOLOGY_DIFF_APPLY_REVERSE HWLOC_NAME_CAPS(TOPOLOGY_DIFF_APPLY_REVERSE)
#define hwloc_topology_diff_apply HWLOC_NAME(topology_diff_apply)
#define hwloc_topology_diff_destroy HWLOC_NAME(topology_diff_destroy)
#define hwloc_topology_diff_load_xml HWLOC_NAME(topology_diff_load_xml)
#define hwloc_topology_diff_export_xml HWLOC_NAME(topology_diff_export_xml)
#define hwloc_topology_diff_load_xmlbuffer HWLOC_NAME(topology_diff_load_xmlbuffer)
#define hwloc_topology_diff_export_xmlbuffer HWLOC_NAME(topology_diff_export_xmlbuffer)

/* shmem.h */

// 定义一系列函数的别名，用于重命名 hwloc 库中的共享内存相关函数
#define hwloc_shmem_topology_get_length HWLOC_NAME(shmem_topology_get_length)
#define hwloc_shmem_topology_write HWLOC_NAME(shmem_topology_write)
#define hwloc_shmem_topology_adopt HWLOC_NAME(shmem_topology_adopt)

/* glibc-sched.h */
// 这部分代码为空，没有需要注释的内容
/* 定义将 hwloc 中的 cpuset 转换为 glibc 调度亲和性的宏 */
#define hwloc_cpuset_to_glibc_sched_affinity HWLOC_NAME(cpuset_to_glibc_sched_affinity)
#define hwloc_cpuset_from_glibc_sched_affinity HWLOC_NAME(cpuset_from_glibc_sched_affinity)

/* linux-libnuma.h */

/* 定义将 hwloc 中的 cpuset 转换为 linux libnuma 中的无符号长整型数组的宏 */
#define hwloc_cpuset_to_linux_libnuma_ulongs HWLOC_NAME(cpuset_to_linux_libnuma_ulongs)
/* 定义将 hwloc 中的 nodeset 转换为 linux libnuma 中的无符号长整型数组的宏 */
#define hwloc_nodeset_to_linux_libnuma_ulongs HWLOC_NAME(nodeset_to_linux_libnuma_ulongs)
/* 定义将 linux libnuma 中的无符号长整型数组转换为 hwloc 中的 cpuset 的宏 */
#define hwloc_cpuset_from_linux_libnuma_ulongs HWLOC_NAME(cpuset_from_linux_libnuma_ulongs)
/* 定义将 linux libnuma 中的无符号长整型数组转换为 hwloc 中的 nodeset 的宏 */
#define hwloc_nodeset_from_linux_libnuma_ulongs HWLOC_NAME(nodeset_from_linux_libnuma_ulongs)
/* 定义将 hwloc 中的 cpuset 转换为 linux libnuma 中的位掩码的宏 */
#define hwloc_cpuset_to_linux_libnuma_bitmask HWLOC_NAME(cpuset_to_linux_libnuma_bitmask)
/* 定义将 hwloc 中的 nodeset 转换为 linux libnuma 中的位掩码的宏 */
#define hwloc_nodeset_to_linux_libnuma_bitmask HWLOC_NAME(nodeset_to_linux_libnuma_bitmask)
/* 定义将 linux libnuma 中的位掩码转换为 hwloc 中的 cpuset 的宏 */
#define hwloc_cpuset_from_linux_libnuma_bitmask HWLOC_NAME(cpuset_from_linux_libnuma_bitmask)
/* 定义将 linux libnuma 中的位掩码转换为 hwloc 中的 nodeset 的宏 */
#define hwloc_nodeset_from_linux_libnuma_bitmask HWLOC_NAME(nodeset_from_linux_libnuma_bitmask)

/* linux.h */

/* 定义设置线程 ID 的 CPU 绑定的宏 */
#define hwloc_linux_set_tid_cpubind HWLOC_NAME(linux_set_tid_cpubind)
/* 定义获取线程 ID 的 CPU 绑定的宏 */
#define hwloc_linux_get_tid_cpubind HWLOC_NAME(linux_get_tid_cpubind)
/* 定义获取线程 ID 最后的 CPU 位置的宏 */
#define hwloc_linux_get_tid_last_cpu_location HWLOC_NAME(linux_get_tid_last_cpu_location)
/* 定义将路径读取为 CPU 掩码的宏 */
#define hwloc_linux_read_path_as_cpumask HWLOC_NAME(linux_read_file_cpumask)

/* windows.h */

/* 定义获取处理器组数量的宏 */
#define hwloc_windows_get_nr_processor_groups HWLOC_NAME(windows_get_nr_processor_groups)
/* 定义获取处理器组的 CPU 集合的宏 */
#define hwloc_windows_get_processor_group_cpuset HWLOC_NAME(windows_get_processor_group_cpuset)

/* openfabrics-verbs.h */

/* 定义获取 InfiniBand 设备的 CPU 集合的宏 */
#define hwloc_ibv_get_device_cpuset HWLOC_NAME(ibv_get_device_cpuset)
/* 定义获取 InfiniBand 设备的操作系统设备的宏 */
#define hwloc_ibv_get_device_osdev HWLOC_NAME(ibv_get_device_osdev)
/* 定义根据名称获取 InfiniBand 设备的操作系统设备的宏 */
#define hwloc_ibv_get_device_osdev_by_name HWLOC_NAME(ibv_get_device_osdev_by_name)

/* opencl.h */

/* 定义获取 AMD OpenCL 设备拓扑的宏 */
#define hwloc_cl_device_topology_amd HWLOC_NAME(cl_device_topology_amd)
/* 定义获取 OpenCL 设备的 PCI 总线 ID 的宏 */
#define hwloc_opencl_get_device_pci_busid HWLOC_NAME(opencl_get_device_pci_ids)
/* 定义获取 OpenCL 设备的 CPU 集合的宏 */
#define hwloc_opencl_get_device_cpuset HWLOC_NAME(opencl_get_device_cpuset)
#define hwloc_opencl_get_device_osdev HWLOC_NAME(opencl_get_device_osdev)
#define hwloc_opencl_get_device_osdev_by_index HWLOC_NAME(opencl_get_device_osdev_by_index)

/* cuda.h */

#define hwloc_cuda_get_device_pci_ids HWLOC_NAME(cuda_get_device_pci_ids)
#define hwloc_cuda_get_device_cpuset HWLOC_NAME(cuda_get_device_cpuset)
#define hwloc_cuda_get_device_pcidev HWLOC_NAME(cuda_get_device_pcidev)
#define hwloc_cuda_get_device_osdev HWLOC_NAME(cuda_get_device_osdev)
#define hwloc_cuda_get_device_osdev_by_index HWLOC_NAME(cuda_get_device_osdev_by_index)

/* cudart.h */

#define hwloc_cudart_get_device_pci_ids HWLOC_NAME(cudart_get_device_pci_ids)
#define hwloc_cudart_get_device_cpuset HWLOC_NAME(cudart_get_device_cpuset)
#define hwloc_cudart_get_device_pcidev HWLOC_NAME(cudart_get_device_pcidev)
#define hwloc_cudart_get_device_osdev_by_index HWLOC_NAME(cudart_get_device_osdev_by_index)

/* nvml.h */

#define hwloc_nvml_get_device_cpuset HWLOC_NAME(nvml_get_device_cpuset)
#define hwloc_nvml_get_device_osdev HWLOC_NAME(nvml_get_device_osdev)
#define hwloc_nvml_get_device_osdev_by_index HWLOC_NAME(nvml_get_device_osdev_by_index)

/* rsmi.h */

#define hwloc_rsmi_get_device_cpuset HWLOC_NAME(rsmi_get_device_cpuset)
#define hwloc_rsmi_get_device_osdev HWLOC_NAME(rsmi_get_device_osdev)
#define hwloc_rsmi_get_device_osdev_by_index HWLOC_NAME(rsmi_get_device_osdev_by_index)

/* levelzero.h */

#define hwloc_levelzero_get_device_cpuset HWLOC_NAME(levelzero_get_device_cpuset)
#define hwloc_levelzero_get_device_osdev HWLOC_NAME(levelzero_get_device_osdev)

/* gl.h */

#define hwloc_gl_get_display_osdev_by_port_device HWLOC_NAME(gl_get_display_osdev_by_port_device)
#define hwloc_gl_get_display_osdev_by_name HWLOC_NAME(gl_get_display_osdev_by_name)
#define hwloc_gl_get_display_by_osdev HWLOC_NAME(gl_get_display_by_osdev)

/* hwloc/plugins.h */

#define hwloc_disc_phase_e HWLOC_NAME(disc_phase_e)
#define HWLOC_DISC_PHASE_GLOBAL HWLOC_NAME_CAPS(DISC_PHASE_GLOBAL)
# 定义 CPU 相关的发现阶段
#define HWLOC_DISC_PHASE_CPU HWLOC_NAME_CAPS(DISC_PHASE_CPU)
# 定义内存相关的发现阶段
#define HWLOC_DISC_PHASE_MEMORY HWLOC_NAME_CAPS(DISC_PHASE_MEMORY)
# 定义 PCI 相关的发现阶段
#define HWLOC_DISC_PHASE_PCI HWLOC_NAME_CAPS(DISC_PHASE_PCI)
# 定义 IO 相关的发现阶段
#define HWLOC_DISC_PHASE_IO HWLOC_NAME_CAPS(DISC_PHASE_IO)
# 定义杂项相关的发现阶段
#define HWLOC_DISC_PHASE_MISC HWLOC_NAME_CAPS(DISC_PHASE_MISC)
# 定义注释相关的发现阶段
#define HWLOC_DISC_PHASE_ANNOTATE HWLOC_NAME_CAPS(DISC_PHASE_ANNOTATE)
# 定义调整相关的发现阶段
#define HWLOC_DISC_PHASE_TWEAK HWLOC_NAME_CAPS(DISC_PHASE_TWEAK)
# 定义发现阶段类型
#define hwloc_disc_phase_t HWLOC_NAME(disc_phase_t)
# 定义发现组件
#define hwloc_disc_component HWLOC_NAME(disc_component)

# 定义发现状态标志类型
#define hwloc_disc_status_flag_e HWLOC_NAME(disc_status_flag_e)
# 定义已获取允许资源的发现状态标志
#define HWLOC_DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES HWLOC_NAME_CAPS(DISC_STATUS_FLAG_GOT_ALLOWED_RESOURCES)
# 定义发现状态
#define hwloc_disc_status HWLOC_NAME(disc_status)

# 定义后端
#define hwloc_backend HWLOC_NAME(backend)

# 分配后端
#define hwloc_backend_alloc HWLOC_NAME(backend_alloc)
# 启用后端
#define hwloc_backend_enable HWLOC_NAME(backend_enable)

# 定义组件类型
#define hwloc_component_type_e HWLOC_NAME(component_type_e)
# 定义发现组件类型
#define HWLOC_COMPONENT_TYPE_DISC HWLOC_NAME_CAPS(COMPONENT_TYPE_DISC)
# 定义 XML 组件类型
#define HWLOC_COMPONENT_TYPE_XML HWLOC_NAME_CAPS(COMPONENT_TYPE_XML)
# 定义组件类型
#define hwloc_component_type_t HWLOC_NAME(component_type_t)
# 定义组件
#define hwloc_component HWLOC_NAME(component)

# 检查插件命名空间
#define hwloc_plugin_check_namespace HWLOC_NAME(plugin_check_namespace)

# 隐藏错误
#define hwloc_hide_errors HWLOC_NAME(hide_errors)
# 通过 CPU 集合插入对象
#define hwloc__insert_object_by_cpuset HWLOC_NAME(_insert_object_by_cpuset)
# 通过父对象插入对象
#define hwloc_insert_object_by_parent HWLOC_NAME(insert_object_by_parent)
# 分配设置对象
#define hwloc_alloc_setup_object HWLOC_NAME(alloc_setup_object)
# 添加子集合到对象
#define hwloc_obj_add_children_sets HWLOC_NAME(add_children_sets)
# 重新连接拓扑结构
#define hwloc_topology_reconnect HWLOC_NAME(topology_reconnect)

# 检查 PCI 设备子类型是否重要
#define hwloc_filter_check_pcidev_subtype_important HWLOC_NAME(filter_check_pcidev_subtype_important)
# 检查操作系统设备子类型是否重要
#define hwloc_filter_check_osdev_subtype_important HWLOC_NAME(filter_check_osdev_subtype_important)
# 检查保留对象类型
#define hwloc_filter_check_keep_object_type HWLOC_NAME(filter_check_keep_object_type)
#define hwloc_filter_check_keep_object HWLOC_NAME(filter_check_keep_object)
#define hwloc_pcidisc_find_cap HWLOC_NAME(pcidisc_find_cap)
#define hwloc_pcidisc_find_linkspeed HWLOC_NAME(pcidisc_find_linkspeed)
#define hwloc_pcidisc_check_bridge_type HWLOC_NAME(pcidisc_check_bridge_type)
#define hwloc_pcidisc_find_bridge_buses HWLOC_NAME(pcidisc_find_bridge_buses)
#define hwloc_pcidisc_tree_insert_by_busid HWLOC_NAME(pcidisc_tree_insert_by_busid)
#define hwloc_pcidisc_tree_attach HWLOC_NAME(pcidisc_tree_attach)
#define hwloc_pci_find_by_busid HWLOC_NAME(pcidisc_find_by_busid)
#define hwloc_pci_find_parent_by_busid HWLOC_NAME(pcidisc_find_busid_parent)
#define hwloc_backend_distances_add_handle_t HWLOC_NAME(backend_distances_add_handle_t)
#define hwloc_backend_distances_add_create HWLOC_NAME(backend_distances_add_create)
#define hwloc_backend_distances_add_values HWLOC_NAME(backend_distances_add_values)
#define hwloc_backend_distances_add_commit HWLOC_NAME(backend_distances_add_commit)
#define hwloc_distances_add HWLOC_NAME(distances_add)
#define hwloc_topology_insert_misc_object_by_parent HWLOC_NAME(topology_insert_misc_object_by_parent)
#define hwloc_obj_cpuset_snprintf HWLOC_NAME(obj_cpuset_snprintf)
#define hwloc_obj_type_sscanf HWLOC_NAME(obj_type_sscanf)
#define hwloc_set_membind_nodeset HWLOC_NAME(set_membind_nodeset)
#define hwloc_get_membind_nodeset HWLOC_NAME(get_membind_nodeset)
#define hwloc_set_proc_membind_nodeset HWLOC_NAME(set_proc_membind_nodeset)
#define hwloc_get_proc_membind_nodeset HWLOC_NAME(get_proc_membind_nodeset)
#define hwloc_set_area_membind_nodeset HWLOC_NAME(set_area_membind_nodeset)
#define hwloc_get_area_membind_nodeset HWLOC_NAME(get_area_membind_nodeset)
#define hwloc_alloc_membind_nodeset HWLOC_NAME(alloc_membind_nodeset)
#define hwloc_cpuset_to_nodeset_strict HWLOC_NAME(cpuset_to_nodeset_strict)
#define hwloc_cpuset_from_nodeset_strict HWLOC_NAME(cpuset_from_nodeset_strict)
#define hwloc_debug_enabled HWLOC_NAME(debug_enabled)
#define hwloc_debug HWLOC_NAME(debug)

/* private/misc.h */

#ifndef HWLOC_HAVE_CORRECT_SNPRINTF
#define hwloc_snprintf HWLOC_NAME(snprintf)
#endif
#define hwloc_ffsl_manual HWLOC_NAME(ffsl_manual)
#define hwloc_ffs32 HWLOC_NAME(ffs32)
#define hwloc_ffsl_from_ffs32 HWLOC_NAME(ffsl_from_ffs32)
#define hwloc_flsl_manual HWLOC_NAME(flsl_manual)
#define hwloc_fls32 HWLOC_NAME(fls32)
#define hwloc_flsl_from_fls32 HWLOC_NAME(flsl_from_fls32)
#define hwloc_weight_long HWLOC_NAME(weight_long)
#define hwloc_strncasecmp HWLOC_NAME(strncasecmp)

#define hwloc_bitmap_compare_inclusion HWLOC_NAME(bitmap_compare_inclusion)

#define hwloc_pci_class_string HWLOC_NAME(pci_class_string)
#define hwloc_linux_pci_link_speed_from_string HWLOC_NAME(linux_pci_link_speed_from_string)

#define hwloc_cache_type_by_depth_type HWLOC_NAME(cache_type_by_depth_type)
#define hwloc__obj_type_is_normal HWLOC_NAME(_obj_type_is_normal)
#define hwloc__obj_type_is_memory HWLOC_NAME(_obj_type_is_memory)
#define hwloc__obj_type_is_io HWLOC_NAME(_obj_type_is_io)
#define hwloc__obj_type_is_special HWLOC_NAME(_obj_type_is_special)

#define hwloc__obj_type_is_cache HWLOC_NAME(_obj_type_is_cache)
#define hwloc__obj_type_is_dcache HWLOC_NAME(_obj_type_is_dcache)
#define hwloc__obj_type_is_icache HWLOC_NAME(_obj_type_is_icache)

/* private/cpuid-x86.h */

#define hwloc_have_x86_cpuid HWLOC_NAME(have_x86_cpuid)
#define hwloc_x86_cpuid HWLOC_NAME(x86_cpuid)

/* private/xml.h */

#define hwloc__xml_verbose HWLOC_NAME(_xml_verbose)

#define hwloc__xml_import_state_s HWLOC_NAME(_xml_import_state_s)
#define hwloc__xml_import_state_t HWLOC_NAME(_xml_import_state_t)
#define hwloc__xml_import_diff HWLOC_NAME(_xml_import_diff)
#define hwloc_xml_backend_data_s HWLOC_NAME(xml_backend_data_s)
#define hwloc__xml_export_state_s HWLOC_NAME(_xml_export_state_s)
#define hwloc__xml_export_state_t HWLOC_NAME(_xml_export_state_t)
/* 定义一些宏，用于重命名一些数据结构和函数名 */
#define hwloc__xml_export_data_s HWLOC_NAME(_xml_export_data_s)
#define hwloc__xml_export_topology HWLOC_NAME(_xml_export_topology)
#define hwloc__xml_export_diff HWLOC_NAME(_xml_export_diff)

#define hwloc_xml_callbacks HWLOC_NAME(xml_callbacks)
#define hwloc_xml_component HWLOC_NAME(xml_component)
#define hwloc_xml_callbacks_register HWLOC_NAME(xml_callbacks_register)
#define hwloc_xml_callbacks_reset HWLOC_NAME(xml_callbacks_reset)

#define hwloc__xml_imported_v1distances_s HWLOC_NAME(_xml_imported_v1distances_s)

/* private/components.h */

/* 定义一些宏，用于强制启用或禁用组件 */
#define hwloc_disc_component_force_enable HWLOC_NAME(disc_component_force_enable)
#define hwloc_disc_components_enable_others HWLOC_NAME(disc_components_instantiate_others)

/* 定义一些宏，用于检查系统是否支持某些后端 */
#define hwloc_backends_is_thissystem HWLOC_NAME(backends_is_thissystem)
#define hwloc_backends_find_callbacks HWLOC_NAME(backends_find_callbacks)

/* 定义一些宏，用于初始化和销毁拓扑组件 */
#define hwloc_topology_components_init HWLOC_NAME(topology_components_init)
#define hwloc_backends_disable_all HWLOC_NAME(backends_disable_all)
#define hwloc_topology_components_fini HWLOC_NAME(topology_components_fini)

/* 定义一些宏，用于初始化和销毁组件 */
#define hwloc_components_init HWLOC_NAME(components_init)
#define hwloc_components_fini HWLOC_NAME(components_fini)

/* private/internal-private.h */

/* 定义一些宏，用于指定特定的组件 */
#define hwloc_xml_component HWLOC_NAME(xml_component)
#define hwloc_synthetic_component HWLOC_NAME(synthetic_component)

#define hwloc_aix_component HWLOC_NAME(aix_component)
#define hwloc_bgq_component HWLOC_NAME(bgq_component)
#define hwloc_darwin_component HWLOC_NAME(darwin_component)
#define hwloc_freebsd_component HWLOC_NAME(freebsd_component)
#define hwloc_hpux_component HWLOC_NAME(hpux_component)
#define hwloc_linux_component HWLOC_NAME(linux_component)
#define hwloc_netbsd_component HWLOC_NAME(netbsd_component)
#define hwloc_noos_component HWLOC_NAME(noos_component)
#define hwloc_solaris_component HWLOC_NAME(solaris_component)
#define hwloc_windows_component HWLOC_NAME(windows_component)
#define hwloc_x86_component HWLOC_NAME(x86_component)  // 定义宏，将 x86_component 替换为 HWLOC_NAME(x86_component)

#define hwloc_cuda_component HWLOC_NAME(cuda_component)  // 定义宏，将 cuda_component 替换为 HWLOC_NAME(cuda_component)
#define hwloc_gl_component HWLOC_NAME(gl_component)  // 定义宏，将 gl_component 替换为 HWLOC_NAME(gl_component)
#define hwloc_levelzero_component HWLOC_NAME(levelzero_component)  // 定义宏，将 levelzero_component 替换为 HWLOC_NAME(levelzero_component)
#define hwloc_nvml_component HWLOC_NAME(nvml_component)  // 定义宏，将 nvml_component 替换为 HWLOC_NAME(nvml_component)
#define hwloc_rsmi_component HWLOC_NAME(rsmi_component)  // 定义宏，将 rsmi_component 替换为 HWLOC_NAME(rsmi_component)
#define hwloc_opencl_component HWLOC_NAME(opencl_component)  // 定义宏，将 opencl_component 替换为 HWLOC_NAME(opencl_component)
#define hwloc_pci_component HWLOC_NAME(pci_component)  // 定义宏，将 pci_component 替换为 HWLOC_NAME(pci_component)

#define hwloc_xml_libxml_component HWLOC_NAME(xml_libxml_component)  // 定义宏，将 xml_libxml_component 替换为 HWLOC_NAME(xml_libxml_component)
#define hwloc_xml_nolibxml_component HWLOC_NAME(xml_nolibxml_component)  // 定义宏，将 xml_nolibxml_component 替换为 HWLOC_NAME(xml_nolibxml_component)

/* private/private.h */

#define hwloc_internal_location_s HWLOC_NAME(internal_location_s)  // 定义宏，将 internal_location_s 替换为 HWLOC_NAME(internal_location_s)

#define hwloc_special_level_s HWLOC_NAME(special_level_s)  // 定义宏，将 special_level_s 替换为 HWLOC_NAME(special_level_s)

#define hwloc_pci_forced_locality_s HWLOC_NAME(pci_forced_locality_s)  // 定义宏，将 pci_forced_locality_s 替换为 HWLOC_NAME(pci_forced_locality_s)
#define hwloc_pci_locality_s HWLOC_NAME(pci_locality_s)  // 定义宏，将 pci_locality_s 替换为 HWLOC_NAME(pci_locality_s)

#define hwloc_topology_forced_component_s HWLOC_NAME(topology_forced_component)  // 定义宏，将 topology_forced_component_s 替换为 HWLOC_NAME(topology_forced_component)

#define hwloc_alloc_root_sets HWLOC_NAME(alloc_root_sets)  // 定义宏，将 alloc_root_sets 替换为 HWLOC_NAME(alloc_root_sets)
#define hwloc_setup_pu_level HWLOC_NAME(setup_pu_level)  // 定义宏，将 setup_pu_level 替换为 HWLOC_NAME(setup_pu_level)
#define hwloc_get_sysctlbyname HWLOC_NAME(get_sysctlbyname)  // 定义宏，将 get_sysctlbyname 替换为 HWLOC_NAME(get_sysctlbyname)
#define hwloc_get_sysctl HWLOC_NAME(get_sysctl)  // 定义宏，将 get_sysctl 替换为 HWLOC_NAME(get_sysctl)
#define hwloc_fallback_nbprocessors HWLOC_NAME(fallback_nbprocessors)  // 定义宏，将 fallback_nbprocessors 替换为 HWLOC_NAME(fallback_nbprocessors)
#define hwloc_fallback_memsize HWLOC_NAME(fallback_memsize)  // 定义宏，将 fallback_memsize 替换为 HWLOC_NAME(fallback_memsize)

#define hwloc__object_cpusets_compare_first HWLOC_NAME(_object_cpusets_compare_first)  // 定义宏，将 _object_cpusets_compare_first 替换为 HWLOC_NAME(_object_cpusets_compare_first)
#define hwloc__reorder_children HWLOC_NAME(_reorder_children)  // 定义宏，将 _reorder_children 替换为 HWLOC_NAME(_reorder_children)

#define hwloc_topology_setup_defaults HWLOC_NAME(topology_setup_defaults)  // 定义宏，将 topology_setup_defaults 替换为 HWLOC_NAME(topology_setup_defaults)
#define hwloc_topology_clear HWLOC_NAME(topology_clear)  // 定义宏，将 topology_clear 替换为 HWLOC_NAME(topology_clear)

#define hwloc__attach_memory_object HWLOC_NAME(insert_memory_object)  // 定义宏，将 insert_memory_object 替换为 HWLOC_NAME(insert_memory_object)

#define hwloc_get_obj_by_type_and_gp_index HWLOC_NAME(get_obj_by_type_and_gp_index)  // 定义宏，将 get_obj_by_type_and_gp_index 替换为 HWLOC_NAME(get_obj_by_type_and_gp_index)

#define hwloc_pci_discovery_init HWLOC_NAME(pci_discovery_init)  // 定义宏，将 pci_discovery_init 替换为 HWLOC_NAME(pci_discovery_init)
#define hwloc_pci_discovery_prepare HWLOC_NAME(pci_discovery_prepare)  // 定义宏，将 pci_discovery_prepare 替换为 HWLOC_NAME(pci_discovery_prepare)
#define hwloc_pci_discovery_exit HWLOC_NAME(pci_discovery_exit)  // 定义宏，将 pci_discovery_exit 替换为 HWLOC_NAME(pci_discovery_exit)
# 定义函数 hwloc_find_insert_io_parent_by_complete_cpuset
#define hwloc_find_insert_io_parent_by_complete_cpuset HWLOC_NAME(hwloc_find_insert_io_parent_by_complete_cpuset)

# 定义函数 hwloc__add_info
#define hwloc__add_info HWLOC_NAME(_add_info)
# 定义函数 hwloc__add_info_nodup
#define hwloc__add_info_nodup HWLOC_NAME(_add_info_nodup)
# 定义函数 hwloc__move_infos
#define hwloc__move_infos HWLOC_NAME(_move_infos)
# 定义函数 hwloc__free_infos
#define hwloc__free_infos HWLOC_NAME(_free_infos)
# 定义函数 hwloc__tma_dup_infos
#define hwloc__tma_dup_infos HWLOC_NAME(_tma_dup_infos)

# 定义变量 hwloc_binding_hooks
#define hwloc_binding_hooks HWLOC_NAME(binding_hooks)
# 定义函数 hwloc_set_native_binding_hooks
#define hwloc_set_native_binding_hooks HWLOC_NAME(set_native_binding_hooks)
# 定义函数 hwloc_set_binding_hooks
#define hwloc_set_binding_hooks HWLOC_NAME(set_binding_hooks)

# 定义函数 hwloc_set_linuxfs_hooks
#define hwloc_set_linuxfs_hooks HWLOC_NAME(set_linuxfs_hooks)
# 定义函数 hwloc_set_bgq_hooks
#define hwloc_set_bgq_hooks HWLOC_NAME(set_bgq_hooks)
# 定义函数 hwloc_set_solaris_hooks
#define hwloc_set_solaris_hooks HWLOC_NAME(set_solaris_hooks)
# 定义函数 hwloc_set_aix_hooks
#define hwloc_set_aix_hooks HWLOC_NAME(set_aix_hooks)
# 定义函数 hwloc_set_windows_hooks
#define hwloc_set_windows_hooks HWLOC_NAME(set_windows_hooks)
# 定义函数 hwloc_set_darwin_hooks
#define hwloc_set_darwin_hooks HWLOC_NAME(set_darwin_hooks)
# 定义函数 hwloc_set_freebsd_hooks
#define hwloc_set_freebsd_hooks HWLOC_NAME(set_freebsd_hooks)
# 定义函数 hwloc_set_netbsd_hooks
#define hwloc_set_netbsd_hooks HWLOC_NAME(set_netbsd_hooks)
# 定义函数 hwloc_set_hpux_hooks
#define hwloc_set_hpux_hooks HWLOC_NAME(set_hpux_hooks)

# 定义函数 hwloc_look_hardwired_fujitsu_k
#define hwloc_look_hardwired_fujitsu_k HWLOC_NAME(look_hardwired_fujitsu_k)
# 定义函数 hwloc_look_hardwired_fujitsu_fx10
#define hwloc_look_hardwired_fujitsu_fx10 HWLOC_NAME(look_hardwired_fujitsu_fx10)
# 定义函数 hwloc_look_hardwired_fujitsu_fx100
#define hwloc_look_hardwired_fujitsu_fx100 HWLOC_NAME(look_hardwired_fujitsu_fx100)

# 定义函数 hwloc_add_uname_info
#define hwloc_add_uname_info HWLOC_NAME(add_uname_info)
# 定义函数 hwloc_free_unlinked_object
#define hwloc_free_unlinked_object HWLOC_NAME(free_unlinked_object)
# 定义函数 hwloc_free_object_and_children
#define hwloc_free_object_and_children HWLOC_NAME(free_object_and_children)
# 定义函数 hwloc_free_object_siblings_and_children
#define hwloc_free_object_siblings_and_children HWLOC_NAME(free_object_siblings_and_children)

# 定义函数 hwloc_alloc_heap
#define hwloc_alloc_heap HWLOC_NAME(alloc_heap)
# 定义函数 hwloc_alloc_mmap
#define hwloc_alloc_mmap HWLOC_NAME(alloc_mmap)
# 定义函数 hwloc_free_heap
#define hwloc_free_heap HWLOC_NAME(free_heap)
# 定义函数 hwloc_free_mmap
#define hwloc_free_mmap HWLOC_NAME(free_mmap)
# 定义函数 hwloc_alloc_or_fail
#define hwloc_alloc_or_fail HWLOC_NAME(alloc_or_fail)

# 定义结构体 hwloc_internal_distances_s
#define hwloc_internal_distances_s HWLOC_NAME(internal_distances_s)
// 定义内部距离初始化函数
#define hwloc_internal_distances_init HWLOC_NAME(internal_distances_init)
// 定义内部距离准备函数
#define hwloc_internal_distances_prepare HWLOC_NAME(internal_distances_prepare)
// 定义内部距离复制函数
#define hwloc_internal_distances_dup HWLOC_NAME(internal_distances_dup)
// 定义内部距离刷新函数
#define hwloc_internal_distances_refresh HWLOC_NAME(internal_distances_refresh)
// 定义内部距离销毁函数
#define hwloc_internal_distances_destroy HWLOC_NAME(internal_distances_destroy)
// 定义内部距离添加函数
#define hwloc_internal_distances_add HWLOC_NAME(internal_distances_add)
// 定义按索引添加内部距离函数
#define hwloc_internal_distances_add_by_index HWLOC_NAME(internal_distances_add_by_index)
// 定义使缓存对象无效的内部距离函数
#define hwloc_internal_distances_invalidate_cached_objs

// 定义内部内存属性结构体
#define hwloc_internal_memattr_s HWLOC_NAME(internal_memattr_s)
// 定义内部内存属性目标结构体
#define hwloc_internal_memattr_target_s HWLOC_NAME(internal_memattr_target_s)
// 定义内部内存属性发起者结构体
#define hwloc_internal_memattr_initiator_s HWLOC_NAME(internal_memattr_initiator_s)
// 定义内部内存属性初始化函数
#define hwloc_internal_memattrs_init HWLOC_NAME(internal_memattrs_init)
// 定义内部内存属性准备函数
#define hwloc_internal_memattrs_prepare HWLOC_NAME(internal_memattrs_prepare)
// 定义内部内存属性复制函数
#define hwloc_internal_memattrs_dup HWLOC_NAME(internal_memattrs_dup)
// 定义内部内存属性销毁函数
#define hwloc_internal_memattrs_destroy HWLOC_NAME(internal_memattrs_destroy)
// 定义内部内存属性需要刷新函数
#define hwloc_internal_memattrs_need_refresh HWLOC_NAME(internal_memattrs_need_refresh)
// 定义内部内存属性刷新函数
#define hwloc_internal_memattrs_refresh HWLOC_NAME(internal_memattrs_refresh)
// 定义内部内存属性猜测内存层次函数
#define hwloc_internal_memattrs_guess_memory_tiers HWLOC_NAME(internal_memattrs_guess_memory_tiers)

// 定义内部 CPU 类型结构体
#define hwloc_internal_cpukind_s HWLOC_NAME(internal_cpukind_s)
// 定义内部 CPU 类型初始化函数
#define hwloc_internal_cpukinds_init HWLOC_NAME(internal_cpukinds_init)
// 定义内部 CPU 类型销毁函数
#define hwloc_internal_cpukinds_destroy HWLOC_NAME(internal_cpukinds_destroy)
// 定义内部 CPU 类型复制函数
#define hwloc_internal_cpukinds_dup HWLOC_NAME(internal_cpukinds_dup)
// 定义内部 CPU 类型注册函数
#define hwloc_internal_cpukinds_register HWLOC_NAME(internal_cpukinds_register)
// 定义内部 CPU 类型排名函数
#define hwloc_internal_cpukinds_rank HWLOC_NAME(internal_cpukinds_rank)
// 定义内部 CPU 类型限制函数
#define hwloc_internal_cpukinds_restrict HWLOC_NAME(internal_cpukinds_restrict)
/* 将 hwloc_encode_to_base64 宏定义为 HWLOC_NAME(encode_to_base64) */
#define hwloc_encode_to_base64 HWLOC_NAME(encode_to_base64)
/* 将 hwloc_decode_from_base64 宏定义为 HWLOC_NAME(decode_from_base64) */
#define hwloc_decode_from_base64 HWLOC_NAME(decode_from_base64)

/* 将 hwloc_progname 宏定义为 HWLOC_NAME(progname) */
#define hwloc_progname HWLOC_NAME(progname)

/* 将 hwloc__topology_disadopt 宏定义为 HWLOC_NAME(_topology_disadopt) */
#define hwloc__topology_disadopt HWLOC_NAME(_topology_disadopt)
/* 将 hwloc__topology_dup 宏定义为 HWLOC_NAME(_topology_dup) */
#define hwloc__topology_dup HWLOC_NAME(_topology_dup)

/* 将 hwloc_tma 宏定义为 HWLOC_NAME(tma) */
#define hwloc_tma HWLOC_NAME(tma)
/* 将 hwloc_tma_malloc 宏定义为 HWLOC_NAME(tma_malloc) */
#define hwloc_tma_malloc HWLOC_NAME(tma_malloc)
/* 将 hwloc_tma_calloc 宏定义为 HWLOC_NAME(tma_calloc) */
#define hwloc_tma_calloc HWLOC_NAME(tma_calloc)
/* 将 hwloc_tma_strdup 宏定义为 HWLOC_NAME(tma_strdup) */
#define hwloc_tma_strdup HWLOC_NAME(tma_strdup)
/* 将 hwloc_bitmap_tma_dup 宏定义为 HWLOC_NAME(bitmap_tma_dup) */
#define hwloc_bitmap_tma_dup HWLOC_NAME(bitmap_tma_dup)

/* private/solaris-chiptype.h */

/* 将 hwloc_solaris_chip_info_s 宏定义为 HWLOC_NAME(solaris_chip_info_s) */
#define hwloc_solaris_chip_info_s HWLOC_NAME(solaris_chip_info_s)
/* 将 hwloc_solaris_get_chip_info 宏定义为 HWLOC_NAME(solaris_get_chip_info) */
#define hwloc_solaris_get_chip_info HWLOC_NAME(solaris_get_chip_info)

/* 结束符号转换 */
#endif /* HWLOC_SYM_TRANSFORM */

/* 如果是 C++ 代码，则使用 extern "C" */
#ifdef __cplusplus
} /* extern "C" */
#endif

/* 结束 hwloc_rename.h 文件的条件编译 */
#endif /* HWLOC_RENAME_H */
```