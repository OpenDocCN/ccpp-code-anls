# `xmrig\src\3rdparty\hwloc\include\hwloc\cuda.h`

```cpp
/*
 * 版权所有 © 2010-2021 Inria。
 * 版权所有 © 2010-2011 Université Bordeaux
 * 版权所有 © 2011 Cisco Systems, Inc.。
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 与 CUDA 驱动程序 API 交互的宏。
 *
 * 使用 hwloc 和 CUDA 驱动程序 API 的应用程序可能希望包含此文件，以便获取 CUDA 设备的拓扑信息。
 *
 */

#ifndef HWLOC_CUDA_H
#define HWLOC_CUDA_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <cuda.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_cuda 与 CUDA 驱动程序 API 的互操作性
 *
 * 当使用 CUDA 驱动程序 API 时，此接口提供了一种检索有关 CUDA 设备的拓扑信息的方法。
 *
 * @{
 */

/** \brief 返回 CUDA 设备 \p cudevice 的域、总线和设备 ID。
 *
 * 设备 \p cudevice 必须与本地机器匹配。
 */
static __hwloc_inline int
hwloc_cuda_get_device_pci_ids(hwloc_topology_t topology __hwloc_attribute_unused,
                  CUdevice cudevice, int *domain, int *bus, int *dev)
{
  CUresult cres;

#if CUDA_VERSION >= 4000
  cres = cuDeviceGetAttribute(domain, CU_DEVICE_ATTRIBUTE_PCI_DOMAIN_ID, cudevice);
  if (cres != CUDA_SUCCESS) {
    errno = ENOSYS;
    return -1;
  }
#else
  *domain = 0;
#endif
  cres = cuDeviceGetAttribute(bus, CU_DEVICE_ATTRIBUTE_PCI_BUS_ID, cudevice);
  if (cres != CUDA_SUCCESS) {
    errno = ENOSYS;
    return -1;
  }
  cres = cuDeviceGetAttribute(dev, CU_DEVICE_ATTRIBUTE_PCI_DEVICE_ID, cudevice);
  if (cres != CUDA_SUCCESS) {
    errno = ENOSYS;
    return -1;
  }

  return 0;
}
/** \brief 获取与设备 \p cudevice 物理上接近的处理器的 CPU 集合。
 *
 * 将描述 CUDA 设备 \p cudevice 的本地性存储在 \p set 中的 CPU 集合中。
 *
 * 拓扑 \p topology 和设备 \p cudevice 必须与本地机器匹配。
 * I/O 设备检测和 CUDA 组件在拓扑中不需要。
 *
 * 该函数仅返回设备的本地性。
 * 如果需要有关设备的更多信息，应使用操作系统对象，参见 hwloc_cuda_get_device_osdev()
 * 和 hwloc_cuda_get_device_osdev_by_index()。
 *
 * 目前，该函数仅对 Linux 有意义地实现；
 * 其他系统将简单地获得完整的 CPU 集合。
 */
static __hwloc_inline int
hwloc_cuda_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
                 CUdevice cudevice, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* 如果我们在 Linux 上，使用 sysfs 机制来获取本地 CPU */
#define HWLOC_CUDA_DEVICE_SYSFS_PATH_MAX 128
  char path[HWLOC_CUDA_DEVICE_SYSFS_PATH_MAX];
  int domainid, busid, deviceid;

  if (hwloc_cuda_get_device_pci_ids(topology, cudevice, &domainid, &busid, &deviceid))
    return -1;

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return -1;
  }

  sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.0/local_cpus", domainid, busid, deviceid);
  if (hwloc_linux_read_path_as_cpumask(path, set) < 0
      || hwloc_bitmap_iszero(set))
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#else
  /* 非 Linux 系统简单地获得完整的 CPU 集合 */
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}
/**
 * \brief 获取与 CUDA 设备 \p cudevice 对应的 hwloc PCI 设备对象。
 *
 * \return 描述 CUDA 设备 \p cudevice 的 hwloc PCI 设备对象。
 * \return 如果找不到，则返回 \c NULL。
 *
 * 拓扑 \p topology 和设备 \p cudevice 必须匹配本地机器。
 * I/O 设备检测必须在拓扑 \p topology 中启用。
 * 拓扑中不需要 CUDA 组件。
 */
static __hwloc_inline hwloc_obj_t
hwloc_cuda_get_device_pcidev(hwloc_topology_t topology, CUdevice cudevice)
{
  int domain, bus, dev;

  if (hwloc_cuda_get_device_pci_ids(topology, cudevice, &domain, &bus, &dev))
    return NULL;

  return hwloc_get_pcidev_by_busid(topology, domain, bus, dev, 0);
}

/**
 * \brief 获取与 CUDA 设备 \p cudevice 对应的 hwloc OS 设备对象。
 *
 * \return 描述给定 CUDA 设备 \p cudevice 的 hwloc OS 设备对象。
 * \return 如果找不到，则返回 \c NULL。
 *
 * 拓扑 \p topology 和设备 \p cudevice 必须匹配本地机器。
 * I/O 设备检测和 CUDA 组件必须在拓扑 \p topology 中启用。
 * 如果没有启用，仍然可以使用 hwloc_cuda_get_device_cpuset() 来找到对象的位置。
 *
 * \note 如果 PCI 设备被过滤掉，此函数无法工作。
 *
 * \note 通过查看结果的父指针，可以找到相应的 hwloc PCI 设备（除非 PCI 设备被过滤掉）。
 */
static __hwloc_inline hwloc_obj_t
hwloc_cuda_get_device_osdev(hwloc_topology_t topology, CUdevice cudevice)
{
    hwloc_obj_t osdev = NULL;
    int domain, bus, dev;

    if (hwloc_cuda_get_device_pci_ids(topology, cudevice, &domain, &bus, &dev))
        return NULL;

    osdev = NULL;
    # 循环遍历下一个操作系统设备
    while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
        # 获取当前操作系统设备的父对象
        hwloc_obj_t pcidev = osdev->parent;
        # 如果操作系统设备的名称不是以 "cuda" 开头，则跳过当前循环
        if (strncmp(osdev->name, "cuda", 4))
            continue;
        # 如果存在父对象，并且父对象是 PCI 设备，并且满足特定的域、总线、设备和函数条件，则返回当前操作系统设备
        if (pcidev
            && pcidev->type == HWLOC_OBJ_PCI_DEVICE
            && (int) pcidev->attr->pcidev.domain == domain
            && (int) pcidev->attr->pcidev.bus == bus
            && (int) pcidev->attr->pcidev.dev == dev
            && pcidev->attr->pcidev.func == 0)
            return osdev;
        # 如果 PCI 被过滤掉，需要一个信息属性来匹配
        /* if PCI are filtered out, we need a info attr to match on */
    }

    # 如果没有找到符合条件的操作系统设备，则返回空
    return NULL;
/**
 * \brief 根据索引获取对应的 CUDA 设备的 hwloc OS 设备对象
 *
 * \param topology 描述 CUDA 设备的 hwloc 拓扑结构
 * \param idx CUDA 设备的索引
 * \return 描述 CUDA 设备的 hwloc OS 设备对象
 * \return 如果找不到则返回 \c NULL
 *
 * 拓扑结构 \p topology 不一定要匹配当前机器。例如，拓扑结构可以是远程主机的 XML 导入。
 * I/O 设备检测和 CUDA 组件必须在拓扑结构中启用。
 *
 * \note 相应的 PCI 设备对象可以通过查看 OS 设备的父对象（除非过滤了 PCI 设备）来获得。
 *
 * \note 此函数与 hwloc_cudart_get_device_osdev_by_index() 函数相同。
 */
static __hwloc_inline hwloc_obj_t
hwloc_cuda_get_device_osdev_by_index(hwloc_topology_t topology, unsigned idx)
{
    hwloc_obj_t osdev = NULL;
    while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
        if (HWLOC_OBJ_OSDEV_COPROC == osdev->attr->osdev.type
            && osdev->name
            && !strncmp("cuda", osdev->name, 4)
            && atoi(osdev->name + 4) == (int) idx)
            return osdev;
    }
    return NULL;
}

/** @} */

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_CUDA_H */
```