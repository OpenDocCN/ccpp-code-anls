# `xmrig\src\3rdparty\hwloc\include\hwloc\cudart.h`

```cpp
/*
 * 版权所有 © 2010-2021 Inria。
 * 版权所有 © 2010-2011 波尔多大学
 * 版权所有 © 2011思科系统公司。
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 与 CUDA Runtime API 交互的宏。
 *
 * 同时使用 hwloc 和 CUDA Runtime API 的应用程序可能希望包含此文件，以获取 CUDA 设备的拓扑信息。
 *
 */

#ifndef HWLOC_CUDART_H
#define HWLOC_CUDART_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <cuda.h> /* 用于 CUDA_VERSION */
#include <cuda_runtime_api.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_cudart 与 CUDA Runtime API 的互操作性
 *
 * 当使用 CUDA Runtime API 时，此接口提供了一种检索有关 CUDA 设备拓扑信息的方法。
 *
 * @{
 */

/** \brief 返回索引为 \p idx 的 CUDA 设备的域、总线和设备 ID。
 *
 * 设备索引 \p idx 必须与本地机器匹配。
 */
static __hwloc_inline int
hwloc_cudart_get_device_pci_ids(hwloc_topology_t topology __hwloc_attribute_unused,
                int idx, int *domain, int *bus, int *dev)
{
  cudaError_t cerr;
  struct cudaDeviceProp prop;

  cerr = cudaGetDeviceProperties(&prop, idx);
  if (cerr) {
    errno = ENOSYS;
    return -1;
  }

#if CUDA_VERSION >= 4000
  *domain = prop.pciDomainID;
#else
  *domain = 0;
#endif

  *bus = prop.pciBusID;
  *dev = prop.pciDeviceID;

  return 0;
}
/** \brief 获取与设备 \p idx 物理上接近的处理器的 CPU 集合。
 *
 * 将描述 CUDA 设备索引为 \p idx 的本地性的 CPU 集合存储在 \p set 中。
 *
 * 拓扑 \p topology 和设备 \p idx 必须匹配本地机器。
 * 在拓扑中不需要进行 I/O 设备检测和 CUDA 组件。
 *
 * 该函数仅返回设备的本地性。
 * 如果需要有关设备的更多信息，应该使用操作系统对象，参见 hwloc_cudart_get_device_osdev_by_index()。
 *
 * 目前，该函数只对 Linux 有意义地实现了；其他系统将简单地获得完整的 CPU 集合。
 */
static __hwloc_inline int
hwloc_cudart_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
                   int idx, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* 如果我们在 Linux 上，使用 sysfs 机制获取本地 CPU */
#define HWLOC_CUDART_DEVICE_SYSFS_PATH_MAX 128
  char path[HWLOC_CUDART_DEVICE_SYSFS_PATH_MAX];
  int domain, bus, dev;

  if (hwloc_cudart_get_device_pci_ids(topology, idx, &domain, &bus, &dev))
    return -1;

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
    return -1;
  }

  sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.0/local_cpus", (unsigned) domain, (unsigned) bus, (unsigned) dev);
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
 * \brief 根据索引获取对应的 CUDA 设备的 hwloc PCI 设备对象
 *
 * \return 描述 CUDA 设备的 hwloc PCI 设备对象，索引为 \p idx
 * \return 如果找不到，则返回 \c NULL
 *
 * 拓扑 \p topology 和设备 \p idx 必须匹配本地机器。
 * I/O 设备检测必须在拓扑 \p topology 中启用。
 * 拓扑中不需要 CUDA 组件。
 */
static __hwloc_inline hwloc_obj_t
hwloc_cudart_get_device_pcidev(hwloc_topology_t topology, int idx)
{
  int domain, bus, dev;

  if (hwloc_cudart_get_device_pci_ids(topology, idx, &domain, &bus, &dev))
    return NULL;

  return hwloc_get_pcidev_by_busid(topology, domain, bus, dev, 0);
}

/**
 * \brief 根据索引获取对应的 CUDA 设备的 hwloc OS 设备对象
 *
 * \return 描述 CUDA 设备的 hwloc OS 设备对象，索引为 \p idx
 * \return 如果找不到，则返回 \c NULL
 *
 * 拓扑 \p topology 不一定要与当前机器匹配。
 * 例如，拓扑可以是远程主机的 XML 导入。
 * I/O 设备检测和 CUDA 组件必须在拓扑 \p topology 中启用。
 * 如果没有启用，仍然可以使用 hwloc_cudart_get_device_cpuset() 找到对象的位置。
 *
 * \note 相应的 PCI 设备对象可以通过查看 OS 设备的父对象（除非过滤了 PCI 设备）来获得。
 *
 * \note 此函数与 hwloc_cuda_get_device_osdev_by_index() 相同。
 */
static __hwloc_inline hwloc_obj_t
hwloc_cudart_get_device_osdev_by_index(hwloc_topology_t topology, unsigned idx)
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

#endif /* HWLOC_CUDART_H */

#ifdef __cplusplus：如果是 C++ 环境
} /* extern "C" */：使用 C++ 语言编译
#endif：结束条件编译
#endif /* HWLOC_CUDART_H */：结束条件编译并指定标识符 HWLOC_CUDART_H 
```