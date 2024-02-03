# `xmrig\src\3rdparty\hwloc\src\topology-noos.c`

```cpp
/*
 * 版权声明
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2019 Inria
 * 版权所有 © 2009-2012, 2020 Université Bordeaux
 * 版权所有 © 2009-2011 Cisco Systems, Inc.
 * 请参阅顶层目录中的 COPYING 文件。
 */

#include "private/autogen/config.h"  // 包含自动生成的配置文件
#include "hwloc.h"  // 包含硬件定位库的头文件
#include "private/private.h"  // 包含私有的头文件

static int
hwloc_look_noos(struct hwloc_backend *backend, struct hwloc_disc_status *dstatus)
{
  /*
   * 这个后端使用底层操作系统。
   * 但是我们不强制 topology->is_thissystem，以便在调试时仍然可以强制使用此后端。
   */

  struct hwloc_topology *topology = backend->topology;  // 获取后端的拓扑结构
  int64_t memsize;  // 存储内存大小的变量

  assert(dstatus->phase == HWLOC_DISC_PHASE_CPU);  // 断言当前状态为 CPU 相关的发现阶段

  if (!topology->levels[0][0]->cpuset) {  // 如果没有创建对象
    int nbprocs;
    /* 没有人（甚至 x86 后端）创建对象，设置基本对象 */

    nbprocs = hwloc_fallback_nbprocessors(0);  // 获取处理器数量
    if (nbprocs >= 1)
      topology->support.discovery->pu = 1;  // 设置支持发现处理器单元
    else
      nbprocs = 1;
    hwloc_alloc_root_sets(topology->levels[0][0]);  // 分配根集合
    hwloc_setup_pu_level(topology, nbprocs);  // 设置处理器单元级别
  }

  memsize = hwloc_fallback_memsize();  // 获取内存大小
  if (memsize > 0)
    topology->machine_memory.local_memory = memsize;  // 设置本地内存大小

  hwloc_add_uname_info(topology, NULL);  // 添加 uname 信息
  return 0;  // 返回成功
}

static struct hwloc_backend *
hwloc_noos_component_instantiate(struct hwloc_topology *topology,
                 struct hwloc_disc_component *component,
                 unsigned excluded_phases __hwloc_attribute_unused,
                 const void *_data1 __hwloc_attribute_unused,
                 const void *_data2 __hwloc_attribute_unused,
                 const void *_data3 __hwloc_attribute_unused)
{
  struct hwloc_backend *backend;  // 定义后端结构体指针
  backend = hwloc_backend_alloc(topology, component);  // 分配后端结构体
  if (!backend)
    return NULL;  // 如果分配失败，返回空指针
  backend->discover = hwloc_look_noos;  // 设置发现函数为 hwloc_look_noos
  return backend;  // 返回后端结构体指针
}
# 定义静态结构体hwloc_noos_disc_component，表示不依赖操作系统的发现组件
static struct hwloc_disc_component hwloc_noos_disc_component = {
  "no_os",  # 组件名称为"no_os"
  HWLOC_DISC_PHASE_CPU,  # 在CPU发现阶段执行
  HWLOC_DISC_PHASE_GLOBAL,  # 在全局发现阶段执行
  hwloc_noos_component_instantiate,  # 实例化函数为hwloc_noos_component_instantiate
  40,  # 优先级为40，低于本地操作系统组件，高于全局组件
  1,  # 可用性为1
  NULL  # 无额外数据
};

# 定义hwloc_noos_component，表示不依赖操作系统的组件
const struct hwloc_component hwloc_noos_component = {
  HWLOC_COMPONENT_ABI,  # 组件的ABI版本
  NULL, NULL,  # 无需初始化或销毁函数
  HWLOC_COMPONENT_TYPE_DISC,  # 组件类型为发现组件
  0,  # 无需特定数据
  &hwloc_noos_disc_component  # 指向发现组件的指针
};
```