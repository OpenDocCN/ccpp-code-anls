# `xmrig\src\3rdparty\hwloc\include\hwloc\openfabrics-verbs.h`

```cpp
/*
 * 版权所有 © 2009 CNRS
 * 版权所有 © 2009-2021 Inria
 * 版权所有 © 2009-2010 Université Bordeaux
 * 版权所有 © 2009-2011 Cisco Systems, Inc.
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 和 OpenFabrics verbs 交互的宏。
 *
 * 使用同时使用 hwloc 和 OpenFabrics verbs 的应用程序可能希望包含此文件，以便获取 OpenFabrics 硬件（InfiniBand 等）的拓扑信息。
 *
 */

#ifndef HWLOC_OPENFABRICS_VERBS_H
#define HWLOC_OPENFABRICS_VERBS_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <infiniband/verbs.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_openfabrics 与 OpenFabrics 的互操作性
 *
 * 该接口提供了一种获取有关 OpenFabrics 设备（InfiniBand、Omni-Path、usNIC 等）拓扑信息的方法。
 *
 * @{
 */

/** \brief 获取与设备 \p ibdev 物理接近的处理器的 CPU 集。
 *
 * 将描述 OpenFabrics 设备 \p ibdev（InfiniBand 等）的本地性的 CPU 集存储在 \p set 中。
 *
 * 拓扑 \p topology 和设备 \p ibdev 必须与本地机器匹配。不需要在拓扑中进行 I/O 设备检测。
 *
 * 该函数仅返回设备的本地性。如果需要有关设备的更多信息，应使用操作系统对象，参见 hwloc_ibv_get_device_osdev()
 * 和 hwloc_ibv_get_device_osdev_by_name()。
 *
 * 目前，此函数仅对 Linux 有意义；其他系统将仅获得完整的 cpuset。
 */
static __hwloc_inline int
hwloc_ibv_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
                struct ibv_device *ibdev, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* 如果我们在 Linux 上，使用 verbs 提供的 sysfs 机制来获取本地 CPU */
#define HWLOC_OPENFABRICS_VERBS_SYSFS_PATH_MAX 128
  // 定义一个路径数组，用于存储路径字符串
  char path[HWLOC_OPENFABRICS_VERBS_SYSFS_PATH_MAX];

  // 检查当前拓扑是否为本地系统拓扑，如果不是则返回错误
  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return -1;
  }

  // 根据 InfiniBand 设备的名称构造路径字符串
  sprintf(path, "/sys/class/infiniband/%s/device/local_cpus",
      ibv_get_device_name(ibdev));
  // 如果无法读取路径或者路径对应的位图为空，则将完整的 CPU 集合复制给位图
  if (hwloc_linux_read_path_as_cpumask(path, set) < 0
      || hwloc_bitmap_iszero(set))
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
  // 如果不是 Linux 系统，则直接将完整的 CPU 集合复制给位图
  else
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
  // 返回成功
  return 0;
}

/** \brief Get the hwloc OS device object corresponding to the OpenFabrics
 * device named \p ibname.
 *
 * \return The hwloc OS device object describing the OpenFabrics device
 * (InfiniBand, Omni-Path, usNIC, etc) whose name is \p ibname
 * (mlx5_0, hfi1_0, usnic_0, qib0, etc).
 * \return \c NULL if none could be found.
 *
 * The name \p ibname is usually obtained from ibv_get_device_name().
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection must be enabled in the topology.
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object.
 */
static __hwloc_inline hwloc_obj_t
hwloc_ibv_get_device_osdev_by_name(hwloc_topology_t topology,
                   const char *ibname)
{
    // 初始化一个指向 OS 设备对象的指针
    hwloc_obj_t osdev = NULL;
    // 遍历拓扑中的 OS 设备对象，查找与给定名称匹配的 OpenFabrics 设备
    while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
        if (HWLOC_OBJ_OSDEV_OPENFABRICS == osdev->attr->osdev.type
            && osdev->name && !strcmp(ibname, osdev->name))
            return osdev;
    }
    // 如果找不到匹配的设备，则返回空指针
    return NULL;
}
/**
 * \brief 获取与 OpenFabrics 设备 \p ibdev 对应的 hwloc OS 设备对象。
 *
 * \return 描述 OpenFabrics 设备 \p ibdev（InfiniBand 等）的 hwloc OS 设备对象。
 * \return 如果找不到，则返回 \c NULL。
 *
 * 拓扑 \p topology 和设备 \p ibdev 必须匹配本地机器。
 * I/O 设备检测必须在拓扑中启用。
 * 如果没有启用，仍然可以使用 hwloc_ibv_get_device_cpuset() 找到对象的位置。
 *
 * \note 可以通过查看 OS 设备的父对象来获取相应的 PCI 设备对象。
 */
static __hwloc_inline hwloc_obj_t
hwloc_ibv_get_device_osdev(hwloc_topology_t topology,
               struct ibv_device *ibdev)
{
    // 如果拓扑不是本系统的，则返回错误
    if (!hwloc_topology_is_thissystem(topology)) {
        errno = EINVAL;
        return NULL;
    }
    // 通过设备名称获取与之对应的 hwloc OS 设备对象
    return hwloc_ibv_get_device_osdev_by_name(topology, ibv_get_device_name(ibdev));
}

/** @} */

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_OPENFABRICS_VERBS_H */
```