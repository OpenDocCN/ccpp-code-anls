# `xmrig\src\3rdparty\hwloc\include\hwloc\levelzero.h`

```
/*
 * 版权所有 © 2021 Inria。
 * 保留所有权利。请参阅顶层目录中的COPYING文件。
 */

/** \file
 * \brief 用于帮助 hwloc 和 oneAPI Level Zero 接口之间交互的宏。
 *
 * 使用同时使用 hwloc 和 Level Zero 的应用程序可能希望包含此文件，以便获取 L0 设备的拓扑信息。
 */

#ifndef HWLOC_LEVELZERO_H
#define HWLOC_LEVELZERO_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <level_zero/ze_api.h>
#include <level_zero/zes_api.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_levelzero 与 oneAPI Level Zero 接口的互操作性。
 *
 * 该接口提供了一种获取由 Level Zero API 管理的设备拓扑信息的方法。
 *
 * @{
 */

/** \brief 获取与 Level Zero 设备 \p device 物理上接近的逻辑处理器的 CPU 集合
 *
 * 将描述 Level Zero 设备 \p device 本地性的 CPU 集合存储在 \p set 中。
 *
 * 拓扑 \p topology 和设备 \p device 必须与本地机器匹配。
 * Level Zero 必须已启用 Sysman（环境中的 ZES_ENABLE_SYSMAN=1）。
 * 拓扑中不需要 I/O 设备检测和 Level Zero 组件。
 *
 * 该函数仅返回设备的本地性。
 * 如果需要有关设备的更多信息，应使用操作系统对象，参见 hwloc_levelzero_get_device_osdev()。
 *
 * 目前，该函数仅对 Linux 有意义地实现；其他系统将简单地获得完整的 CPU 集合。
 */
static __hwloc_inline int
hwloc_levelzero_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
                                  ze_device_handle_t device, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* 如果我们在 Linux 上，使用 sysfs 机制获取本地 CPU */
# 定义 Level Zero 设备的 sysfs 路径的最大长度为 128
#define HWLOC_LEVELZERO_DEVICE_SYSFS_PATH_MAX 128
  声明一个长度为 HWLOC_LEVELZERO_DEVICE_SYSFS_PATH_MAX 的字符数组 path
  zes_pci_properties_t 结构体变量 pci
  声明一个 Level Zero 设备句柄 sdevice，并初始化为 device
  声明一个 ze_result_t 类型的变量 res

  如果当前拓扑结构不是本系统的拓扑结构，则设置 errno 为 EINVAL 并返回 -1
  res 被赋值为 zesDevicePciGetProperties(sdevice, &pci)
  如果 res 不等于 ZE_RESULT_SUCCESS，则设置 errno 为 EINVAL 并返回 -1

  格式化字符串 path，拼接 Level Zero 设备的 sysfs 路径
  如果 hwloc_linux_read_path_as_cpumask(path, set) 返回值小于 0 或者 set 是空的，则将 set 设置为完整的 CPU 集合
#else
  /* 非 Linux 系统直接获取完整的 CPU 集合 */
  将 set 设置为完整的 CPU 集合
#endif
  返回 0
}

/** \brief 获取与 Level Zero 设备 \p device 对应的 hwloc OS 设备对象
 * \p device.
 *
 * \return 描述给定 Level Zero 设备 \p device 的 hwloc OS 设备对象。
 * \return 如果找不到，则返回 \c NULL。
 *
 * 拓扑结构 \p topology 和设备 \p dv_ind 必须与本地机器匹配。
 * I/O 设备检测和 Level Zero 组件必须在拓扑结构中启用。
 * 如果没有启用，仍然可以使用 hwloc_levelzero_get_device_cpuset() 来找到对象的位置。
 *
 * \note 相应的 hwloc PCI 设备可以通过查看结果的父指针找到（除非 PCI 设备被过滤掉）。
 */
static __hwloc_inline hwloc_obj_t
hwloc_levelzero_get_device_osdev(hwloc_topology_t topology, ze_device_handle_t device)
{
  声明一个 Level Zero 设备句柄 sdevice，并初始化为 device
  zes_pci_properties_t 结构体变量 pci
  声明一个 ze_result_t 类型的变量 res
  声明一个 hwloc_obj_t 类型的变量 osdev

  如果当前拓扑结构不是本系统的拓扑结构，则设置 errno 为 EINVAL 并返回 NULL
  res 被赋值为 zesDevicePciGetProperties(sdevice, &pci)
  如果 res 不等于 ZE_RESULT_SUCCESS，则注释 L0 可能是在没有 sysman 的情况下初始化，不要打扰
    # 设置错误码为无效参数
    errno = EINVAL;
    # 返回空指针
    return NULL;
  }

  # 初始化操作系统设备为 NULL
  osdev = NULL;
  # 遍历每个操作系统设备
  while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
    # 获取当前操作系统设备的父对象
    hwloc_obj_t pcidev = osdev->parent;

    # 如果操作系统设备名称不以 "ze" 开头，则跳过当前循环
    if (strncmp(osdev->name, "ze", 2))
      continue;

    # 如果父对象存在，并且是 PCI 设备，并且与给定的 PCI 地址匹配，则返回当前操作系统设备
    if (pcidev
      && pcidev->type == HWLOC_OBJ_PCI_DEVICE
      && pcidev->attr->pcidev.domain == pci.address.domain
      && pcidev->attr->pcidev.bus == pci.address.bus
      && pcidev->attr->pcidev.dev == pci.address.device
      && pcidev->attr->pcidev.func == pci.address.function)
      return osdev;

    # 当我们有序列号时，尝试使用它以防 PCI 被过滤掉
    # FIXME: when we'll have serialnumber, try it in case PCI is filtered-out
  }

  # 如果没有找到匹配的操作系统设备，则返回空指针
  return NULL;
// 结束对特定功能模块的注释
/** @} */

// 如果是 C++ 环境，则使用 extern "C" 包裹代码
#ifdef __cplusplus
} /* extern "C" */
#endif

// 结束对 hwloc_levelzero.h 文件的条件编译
#endif /* HWLOC_LEVELZERO_H */
```