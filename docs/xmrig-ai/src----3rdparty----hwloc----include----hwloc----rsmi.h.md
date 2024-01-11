# `xmrig\src\3rdparty\hwloc\include\hwloc\rsmi.h`

```
/*
 * 版权所有 © 2012-2021 Inria。
 * 版权所有 © 2020，Advanced Micro Devices, Inc.。
 * 由 Advanced Micro Devices 编写，
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 与 ROCm SMI 管理库交互的宏。
 *
 * 使用同时使用 hwloc 和 ROCm SMI 管理库的应用程序可能希望包含此文件，以便获取 AMD GPU 设备的拓扑信息。
 */

#ifndef HWLOC_RSMI_H
#define HWLOC_RSMI_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#include <rocm_smi/rocm_smi.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_rsmi 与 ROCm SMI 管理库的互操作性
 *
 * 该接口提供了一种获取由 ROCm SMI 管理库管理的设备拓扑信息的方法。
 *
 * @{
 */

/** \brief 获取与索引为 \p dv_ind 的 AMD GPU 设备物理上接近的逻辑处理器的 CPU 集合。
 *
 * 将描述索引为 \p dv_ind 的 AMD GPU 设备的本地性的 CPU 集合存储在 \p set 中。
 *
 * 拓扑 \p topology 和设备 \p dv_ind 必须与本地机器匹配。
 * 不需要在拓扑中进行 I/O 设备检测和 ROCm SMI 组件。
 *
 * 该函数仅返回设备的本地性。
 * 如果需要有关设备的更多信息，应使用操作系统对象，参见 hwloc_rsmi_get_device_osdev()
 * 和 hwloc_rsmi_get_device_osdev_by_index()。
 *
 * 目前，该函数仅对 Linux 有意义地实现；其他系统将简单地获得完整的 cpuset。
 */
static __hwloc_inline int
hwloc_rsmi_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
                             uint32_t dv_ind, hwloc_cpuset_t set)
{
#ifdef HWLOC_LINUX_SYS
  /* 如果我们在 Linux 上，使用 sysfs 机制获取本地 CPU */
#define HWLOC_RSMI_DEVICE_SYSFS_PATH_MAX 128
  // 定义一个常量，表示设备系统文件路径的最大长度为128

  char path[HWLOC_RSMI_DEVICE_SYSFS_PATH_MAX];
  // 声明一个字符数组，用于存储设备系统文件路径
  rsmi_status_t ret;
  // 声明一个变量，用于存储函数返回的状态值
  uint64_t bdfid = 0;
  // 声明一个64位整型变量，用于存储设备的 BDFID
  unsigned domain, device, bus;
  // 声明三个无符号整型变量，用于存储设备的域、设备和总线号

  if (!hwloc_topology_is_thissystem(topology)) {
    // 如果当前拓扑结构不是本系统的
    errno = EINVAL;
    return -1;
    // 设置错误码并返回-1
  }

  ret = rsmi_dev_pci_id_get(dv_ind, &bdfid);
  // 调用函数获取设备的 PCI ID
  if (RSMI_STATUS_SUCCESS != ret) {
    // 如果获取失败
    errno = EINVAL;
    return -1;
    // 设置错误码并返回-1
  }
  domain = (bdfid>>32) & 0xffffffff;
  bus = ((bdfid & 0xffff)>>8) & 0xff;
  device = ((bdfid & 0xff)>>3) & 0x1f;
  // 从 BDFID 中提取域、总线和设备号

  sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.0/local_cpus", domain, bus, device);
  // 格式化设备系统文件路径
  if (hwloc_linux_read_path_as_cpumask(path, set) < 0
      || hwloc_bitmap_iszero(set))
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
  // 如果读取设备的 CPU 掩码失败或者掩码为空，则将完整的 CPU 集合复制给掩码
#else
  /* Non-Linux systems simply get a full cpuset */
  hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
  // 对于非 Linux 系统，直接将完整的 CPU 集合复制给掩码
#endif
  return 0;
  // 返回0表示成功
}

/** \brief Get the hwloc OS device object corresponding to the
 * AMD GPU device whose index is \p dv_ind.
 *
 * \return The hwloc OS device object describing the AMD GPU device whose
 * index is \p dv_ind.
 * \return \c NULL if none could be found.
 *
 * The topology \p topology does not necessarily have to match the current
 * machine. For instance the topology may be an XML import of a remote host.
 * I/O devices detection and the ROCm SMI component must be enabled in the
 * topology.
 *
 * \note The corresponding PCI device object can be obtained by looking
 * at the OS device parent object (unless PCI devices are filtered out).
 */
static __hwloc_inline hwloc_obj_t
hwloc_rsmi_get_device_osdev_by_index(hwloc_topology_t topology, uint32_t dv_ind)
{
  hwloc_obj_t osdev = NULL;
  // 声明一个 hwloc OS 设备对象指针，并初始化为 NULL
  while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
    // 遍历拓扑结构中的 OS 设备对象
    if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
      && osdev->name
      && !strncmp("rsmi", osdev->name, 4)
      && atoi(osdev->name + 4) == (int) dv_ind)
      return osdev;
    // 如果是 GPU 设备且名称以 "rsmi" 开头并且索引匹配，则返回该设备对象
  }
  return NULL;
  // 如果没有找到匹配的设备对象，则返回 NULL
}
/**
 * \brief 获取对应于 AMD GPU 设备索引为 \p dv_ind 的 hwloc OS 设备对象。
 *
 * \return 描述给定 AMD GPU 的 hwloc OS 设备对象。
 * \return 如果找不到，则返回 \c NULL。
 *
 * 拓扑 \p topology 和设备 \p dv_ind 必须匹配本地机器。
 * I/O 设备检测和 ROCm SMI 组件必须在拓扑中启用。
 * 如果没有启用，仍然可以使用 hwloc_rsmi_get_device_cpuset() 找到对象的位置。
 *
 * \note 相应的 hwloc PCI 设备可以通过查看结果的父指针找到（除非 PCI 设备被过滤掉）。
 */
static __hwloc_inline hwloc_obj_t
hwloc_rsmi_get_device_osdev(hwloc_topology_t topology, uint32_t dv_ind)
{
  hwloc_obj_t osdev; // 声明 hwloc OS 设备对象
  rsmi_status_t ret; // 声明 rsmi 状态变量
  uint64_t bdfid = 0; // 初始化 bdfid 变量
  unsigned domain, device, bus, func; // 声明 domain, device, bus, func 变量
  uint64_t id; // 声明 id 变量
  char uuid[64]; // 声明 uuid 变量

  if (!hwloc_topology_is_thissystem(topology)) { // 如果拓扑不是本地系统
    errno = EINVAL; // 设置错误码为无效参数
    return NULL; // 返回空指针
  }

  ret = rsmi_dev_pci_id_get(dv_ind, &bdfid); // 获取设备的 PCI ID
  if (RSMI_STATUS_SUCCESS != ret) { // 如果获取失败
    errno = EINVAL; // 设置错误码为无效参数
    return NULL; // 返回空指针
  }
  domain = (bdfid>>32) & 0xffffffff; // 计算域
  bus = ((bdfid & 0xffff)>>8) & 0xff; // 计算总线
  device = ((bdfid & 0xff)>>3) & 0x1f; // 计算设备
  func = bdfid & 0x7; // 计算功能

  ret = rsmi_dev_unique_id_get(dv_ind, &id); // 获取设备的唯一 ID
  if (RSMI_STATUS_SUCCESS != ret) // 如果获取失败
    uuid[0] = '\0'; // 将 uuid 设置为空字符串
  else
    sprintf(uuid, "%lx", id); // 将 id 转换为十六进制字符串

  osdev = NULL; // 初始化 osdev 为空指针
  while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) { // 遍历所有 OS 设备对象
    hwloc_obj_t pcidev = osdev->parent; // 获取父对象
    const char *info; // 声明 info 变量

    if (strncmp(osdev->name, "rsmi", 4)) // 如果 OS 设备名称不是以 "rsmi" 开头
      continue; // 继续下一次循环

    if (pcidev
      && pcidev->type == HWLOC_OBJ_PCI_DEVICE
      && pcidev->attr->pcidev.domain == domain
      && pcidev->attr->pcidev.bus == bus
      && pcidev->attr->pcidev.dev == device
      && pcidev->attr->pcidev.func == func) // 如果满足 PCI 设备的条件
      return osdev; // 返回 OS 设备对象

    info = hwloc_obj_get_info_by_name(osdev, "AMDUUID"); // 获取指定名称的信息
    if (info && !strcmp(info, uuid)) // 如果信息存在且与 uuid 匹配
      return osdev; // 返回 OS 设备对象
  }

  return NULL; // 返回空指针
}

/** @} */
#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_RSMI_H */

#ifdef __cplusplus：如果是 C++ 环境
} /* extern "C" */：使用 C++ 语言编译
#endif：结束条件编译
#endif /* HWLOC_RSMI_H */：结束条件编译并指定标识符 HWLOC_RSMI_H 
```