# `xmrig\src\3rdparty\hwloc\include\hwloc\nvml.h`

```cpp
/*
 * 版权所有 © 2012-2021 Inria。
 * 保留所有权利。请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 与 NVIDIA 管理库之间交互的宏。
 *
 * 同时使用 hwloc 和 NVIDIA 管理库的应用程序可能希望包含此文件，以便获取 NVML 设备的拓扑信息。
 */

#ifndef HWLOC_NVML_H
#define HWLOC_NVML_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <nvml.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_nvml 与 NVIDIA 管理库的互操作性
 *
 * 该接口提供了一种获取由 NVIDIA 管理库（NVML）管理的设备拓扑信息的方式。
 *
 * @{
 */

/** \brief 获取与 NVML 设备 \p device 物理上接近的处理器的 CPU 集合。
 *
 * 将描述 NVML 设备 \p device 本地性的 CPU 集合存储在 \p set 中。
 *
 * 拓扑 \p topology 和设备 \p device 必须与本地机器匹配。
 * 不需要在拓扑中进行 I/O 设备检测和 NVML 组件。
 *
 * 该函数仅返回设备的本地性。
 * 如果需要有关设备的更多信息，应使用操作系统对象，参见 hwloc_nvml_get_device_osdev()
 * 和 hwloc_nvml_get_device_osdev_by_index()。
 *
 * 目前，该函数仅对 Linux 有意义地实现；其他系统将简单地获得完整的 CPU 集合。
 */
static __hwloc_inline int
hwloc_nvml_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
                 nvmlDevice_t device, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* 如果我们在 Linux 上，使用 sysfs 机制获取本地 CPU */
#define HWLOC_NVML_DEVICE_SYSFS_PATH_MAX 128
  char path[HWLOC_NVML_DEVICE_SYSFS_PATH_MAX];
  nvmlReturn_t nvres;
  nvmlPciInfo_t pci;

  if (!hwloc_topology_is_thissystem(topology)) {
    errno = EINVAL;
  # 返回错误代码 -1
  return -1;
}

# 获取设备的 PCI 信息
nvres = nvmlDeviceGetPciInfo(device, &pci);
if (NVML_SUCCESS != nvres) {
  # 设置错误码为无效参数，并返回错误代码 -1
  errno = EINVAL;
  return -1;
}

# 格式化路径字符串，用于获取设备的本地 CPU
sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.0/local_cpus", pci.domain, pci.bus, pci.device);
# 如果无法从路径中读取 CPU 掩码，或者掩码为空，则使用完整的 CPU 集合
if (hwloc_linux_read_path_as_cpumask(path, set) < 0
    || hwloc_bitmap_iszero(set))
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#else
  /* 非 Linux 系统直接获取完整的 cpuset */
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}

/** \brief 根据索引获取对应的 NVML 设备的 hwloc OS 设备对象。
 *
 * \return 描述索引为 \p idx 的 NVML 设备的 hwloc OS 设备对象。
 * \return 如果找不到，则返回 \c NULL。
 *
 * 拓扑结构 \p topology 不一定要匹配当前机器。例如，拓扑结构可能是远程主机的 XML 导入。
 * I/O 设备检测和 NVML 组件必须在拓扑结构中启用。
 *
 * \note 相应的 PCI 设备对象可以通过查看 OS 设备的父对象（除非 PCI 设备被过滤掉）来获得。
 */
static __hwloc_inline hwloc_obj_t
hwloc_nvml_get_device_osdev_by_index(hwloc_topology_t topology, unsigned idx)
{
    hwloc_obj_t osdev = NULL;
    while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
                if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
                    && osdev->name
            && !strncmp("nvml", osdev->name, 4)
            && atoi(osdev->name + 4) == (int) idx)
                        return osdev;
        }
        return NULL;
}

/** \brief 获取描述给定 NVML 设备 \p device 的 hwloc OS 设备对象。
 *
 * \return 描述给定 NVML 设备 \p device 的 hwloc OS 设备对象。
 * \return 如果找不到，则返回 \c NULL。
 *
 * 拓扑结构 \p topology 和设备 \p device 必须与本地机器匹配。
 * I/O 设备检测和 NVML 组件必须在拓扑结构中启用。
 * 如果没有启用，仍然可以通过 hwloc_nvml_get_device_cpuset() 来找到对象的位置。
 *
 * \note 通过查看结果的父指针（除非 PCI 设备被过滤掉），可以找到相应的 hwloc PCI 设备。
 */
static __hwloc_inline hwloc_obj_t
hwloc_nvml_get_device_osdev(hwloc_topology_t topology, nvmlDevice_t device)
{
    # 声明一个指向 hwloc_obj_t 类型的指针变量 osdev
    hwloc_obj_t osdev;
    # 声明一个 nvmlReturn_t 类型的变量 nvres
    nvmlReturn_t nvres;
    # 声明一个 nvmlPciInfo_t 类型的变量 pci
    nvmlPciInfo_t pci;
    # 声明一个长度为64的字符数组 uuid
    char uuid[64];

    # 如果当前拓扑结构不是本系统的拓扑结构，则设置错误码并返回空指针
    if (!hwloc_topology_is_thissystem(topology)) {
        errno = EINVAL;
        return NULL;
    }

    # 获取设备的 PCI 信息，如果失败则返回空指针
    nvres = nvmlDeviceGetPciInfo(device, &pci);
    if (NVML_SUCCESS != nvres)
        return NULL;

    # 获取设备的 UUID，如果失败则将 uuid 的第一个字符设置为结束符
    nvres = nvmlDeviceGetUUID(device, uuid, sizeof(uuid));
    if (NVML_SUCCESS != nvres)
        uuid[0] = '\0';

    # 初始化 osdev 为 NULL，然后开始循环
    osdev = NULL;
    while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
        # 获取当前 osdev 的父对象
        hwloc_obj_t pcidev = osdev->parent;
        # 声明一个指向常量字符的指针 info

        # 如果当前 osdev 的名称不是以 "nvml" 开头，则继续下一次循环
        if (strncmp(osdev->name, "nvml", 4))
            continue;

        # 如果 pcidev 存在，并且类型是 HWLOC_OBJ_PCI_DEVICE，并且与 pci 的 domain、bus、dev、func 相匹配，则返回 osdev
        if (pcidev
            && pcidev->type == HWLOC_OBJ_PCI_DEVICE
            && pcidev->attr->pcidev.domain == pci.domain
            && pcidev->attr->pcidev.bus == pci.bus
            && pcidev->attr->pcidev.dev == pci.device
            && pcidev->attr->pcidev.func == 0)
            return osdev;

        # 获取 osdev 的 "NVIDIAUUID" 信息，如果存在并且与 uuid 相等，则返回 osdev
        info = hwloc_obj_get_info_by_name(osdev, "NVIDIAUUID");
        if (info && !strcmp(info, uuid))
            return osdev;
    }

    # 如果循环结束仍未找到匹配的 osdev，则返回空指针
    return NULL;
// 结束对 NVML 相关函数的声明
}

/** @} */

// 结束对 NVML 相关函数的声明的注释块

#ifdef __cplusplus
// 如果是 C++ 环境
} /* extern "C" */
#endif

// 结束对 C++ 环境的处理

// 结束对 NVML 头文件的条件编译
#endif /* HWLOC_NVML_H */

// 结束对 NVML 头文件的条件编译的注释块
```