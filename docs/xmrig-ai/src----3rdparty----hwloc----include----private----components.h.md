# `xmrig\src\3rdparty\hwloc\include\private\components.h`

```cpp
/*
 * 版权声明，版权归 Inria 公司所有，保留所有权利
 * 请查看顶层目录下的 COPYING 文件
 */
 
#ifdef HWLOC_INSIDE_PLUGIN
/*
 * 这些声明仅供内部使用，对插件不可见
 * (下面的许多函数都是内部静态符号)
 */
#error 该文件不应该在插件中使用
#endif

#ifndef PRIVATE_COMPONENTS_H
#define PRIVATE_COMPONENTS_H 1

#include "hwloc/plugins.h"

struct hwloc_topology;

extern int hwloc_disc_component_force_enable(struct hwloc_topology *topology,
                         int envvar_forced, /* 1 表示通过环境变量强制，0 表示通过 API 强制 */
                         const char *name,
                         const void *data1, const void *data2, const void *data3);
extern void hwloc_disc_components_enable_others(struct hwloc_topology *topology);

/* 计算拓扑结构的 is_thissystem 标志，并基于启用的后端查找一些回调函数 */
extern void hwloc_backends_is_thissystem(struct hwloc_topology *topology);
extern void hwloc_backends_find_callbacks(struct hwloc_topology *topology);

/* 初始化拓扑结构使用的组件和后端的列表 */
extern void hwloc_topology_components_init(struct hwloc_topology *topology);
/* 禁用和销毁拓扑结构使用的所有后端 */
extern void hwloc_backends_disable_all(struct hwloc_topology *topology);
/* 清理拓扑结构使用的组件列表 */
extern void hwloc_topology_components_fini(struct hwloc_topology *topology);

/* 用于核心设置/销毁组件列表 */
extern void hwloc_components_init(void); /* 增加组件的引用计数，每个拓扑结构应该调用一次（在初始化期间） */
extern void hwloc_components_fini(void); /* 减少组件的引用计数，每个拓扑结构应该调用一次（在销毁期间） */

#endif /* PRIVATE_COMPONENTS_H */
```