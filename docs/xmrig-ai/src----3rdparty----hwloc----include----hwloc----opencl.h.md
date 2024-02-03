# `xmrig\src\3rdparty\hwloc\include\hwloc\opencl.h`

```cpp
/*
 * 版权所有 © 2012-2021 Inria。
 * 版权所有 © 2013, 2018 Université Bordeaux。
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 和 OpenCL 接口之间交互的宏。
 *
 * 使用同时使用 hwloc 和 OpenCL 的应用程序可能希望
 * 包含此文件，以便获取 OpenCL 设备的拓扑信息。
 */

#ifndef HWLOC_OPENCL_H
#define HWLOC_OPENCL_H

#include "hwloc.h"
#include "hwloc/autogen/config.h"
#include "hwloc/helper.h"
#ifdef HWLOC_LINUX_SYS
#include "hwloc/linux.h"
#endif

#ifdef __APPLE__
#include <OpenCL/cl.h>
#else
#include <CL/cl.h>
#endif

#include <stdio.h>


#ifdef __cplusplus
extern "C" {
#endif


/* OpenCL 扩展并不总是随默认头文件一起提供，
 * 它们也不总是反映已安装实现的支持情况。
 * 尝试一切，让实现在不支持时返回错误。
 */
/* 版权所有 (c) 2008-2018 The Khronos Group Inc. */

/* 需要 "cl_amd_device_attribute_query" 设备扩展，但对 clGetDeviceInfo() 并非严格要求 */
#define HWLOC_CL_DEVICE_TOPOLOGY_AMD 0x4037
typedef union {
    struct { cl_uint type; cl_uint data[5]; } raw;
    struct { cl_uint type; cl_char unused[17]; cl_char bus; cl_char device; cl_char function; } pcie;
} hwloc_cl_device_topology_amd;
#define HWLOC_CL_DEVICE_TOPOLOGY_TYPE_PCIE_AMD 1

/* 需要 "cl_nv_device_attribute_query" 设备扩展，但对 clGetDeviceInfo() 并非严格要求 */
#define HWLOC_CL_DEVICE_PCI_BUS_ID_NV 0x4008
#define HWLOC_CL_DEVICE_PCI_SLOT_ID_NV 0x4009
#define HWLOC_CL_DEVICE_PCI_DOMAIN_ID_NV 0x400A


/** \defgroup hwlocality_opencl 与 OpenCL 的互操作性
 *
 * 此接口提供了一种获取有关 OpenCL 设备拓扑信息的方法。
 *
 * 目前只有 AMD 和 NVIDIA 的 OpenCL 实现提供了有用的设备位置信息。
 *
 * @{
 */
/** \brief Return the domain, bus and device IDs of the OpenCL device \p device.
 *
 * Device \p device must match the local machine.
 */
static __hwloc_inline int
hwloc_opencl_get_device_pci_busid(cl_device_id device,
                               unsigned *domain, unsigned *bus, unsigned *dev, unsigned *func)
{
    hwloc_cl_device_topology_amd amdtopo; // 定义一个结构体变量amdtopo，用于存储OpenCL设备的拓扑信息
    cl_uint nvbus, nvslot, nvdomain; // 定义三个无符号整型变量，用于存储设备的总线、插槽和域ID
    cl_int clret; // 定义一个整型变量，用于存储OpenCL函数调用的返回值

    clret = clGetDeviceInfo(device, HWLOC_CL_DEVICE_TOPOLOGY_AMD, sizeof(amdtopo), &amdtopo, NULL); // 调用OpenCL函数获取设备的拓扑信息
    if (CL_SUCCESS == clret
        && HWLOC_CL_DEVICE_TOPOLOGY_TYPE_PCIE_AMD == amdtopo.raw.type) { // 如果获取成功且设备类型为PCIe
        *domain = 0; /* can't do anything better */ // 将域ID设置为0
        /* cl_device_topology_amd stores bus ID in cl_char, dont convert those signed char directly to unsigned int */
        *bus = (unsigned) (unsigned char) amdtopo.pcie.bus; // 将总线ID转换为无符号整型
        *dev = (unsigned) (unsigned char) amdtopo.pcie.device; // 将设备ID转换为无符号整型
        *func = (unsigned) (unsigned char) amdtopo.pcie.function; // 将功能ID转换为无符号整型
        return 0; // 返回0表示成功
    }

    clret = clGetDeviceInfo(device, HWLOC_CL_DEVICE_PCI_BUS_ID_NV, sizeof(nvbus), &nvbus, NULL); // 调用OpenCL函数获取设备的总线ID
    if (CL_SUCCESS == clret) { // 如果获取成功
        clret = clGetDeviceInfo(device, HWLOC_CL_DEVICE_PCI_SLOT_ID_NV, sizeof(nvslot), &nvslot, NULL); // 再次调用OpenCL函数获取设备的插槽ID
        if (CL_SUCCESS == clret) { // 如果获取成功
            clret = clGetDeviceInfo(device, HWLOC_CL_DEVICE_PCI_DOMAIN_ID_NV, sizeof(nvdomain), &nvdomain, NULL); // 再次调用OpenCL函数获取设备的域ID
            if (CL_SUCCESS == clret) { /* available since CUDA 10.2 */ // 如果获取成功（CUDA 10.2及以上版本可用）
                *domain = nvdomain; // 将域ID设置为获取到的值
            } else {
                *domain = 0; // 否则将域ID设置为0
            }
            *bus = nvbus & 0xff; // 将总线ID设置为获取到的值的低8位
            /* non-documented but used in many other projects */
            *dev = nvslot >> 3; // 将设备ID设置为获取到的值右移3位
            *func = nvslot & 0x7; // 将功能ID设置为获取到的值的低3位
            return 0; // 返回0表示成功
        }
    }

    return -1; // 如果以上条件都不满足，则返回-1表示失败
}
/** \brief 获取与 OpenCL 设备 \p device 物理上接近的处理器的 CPU 集合。
 *
 * 将描述 OpenCL 设备 \p device 本地性的 CPU 集合存储在 \p set 中。
 *
 * 拓扑 \p topology 和设备 \p device 必须与本地机器匹配。
 * 不需要在拓扑中检测 I/O 设备和 OpenCL 组件。
 *
 * 该函数仅返回设备的本地性。
 * 如果需要有关设备的更多信息，应该使用操作系统对象，参见 hwloc_opencl_get_device_osdev()
 * 和 hwloc_opencl_get_device_osdev_by_index()。
 *
 * 目前，该函数仅对 Linux 上的 AMD 或 NVIDIA OpenCL 实现有意义；其他系统将简单地获得完整的 cpuset。
 */
static __hwloc_inline int
hwloc_opencl_get_device_cpuset(hwloc_topology_t topology __hwloc_attribute_unused,
                   cl_device_id device __hwloc_attribute_unused,
                   hwloc_cpuset_t set)
{
#if (defined HWLOC_LINUX_SYS)
    /* 如果我们在 Linux 上，尝试使用 AMD/NVIDIA 扩展 + sysfs 机制来获取本地 CPU */
#define HWLOC_OPENCL_DEVICE_SYSFS_PATH_MAX 128
    char path[HWLOC_OPENCL_DEVICE_SYSFS_PATH_MAX];
    unsigned pcidomain, pcibus, pcidev, pcifunc;

    if (!hwloc_topology_is_thissystem(topology)) {
        errno = EINVAL;
        return -1;
    }

    if (hwloc_opencl_get_device_pci_busid(device, &pcidomain, &pcibus, &pcidev, &pcifunc) < 0) {
        hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
        return 0;
    }

    sprintf(path, "/sys/bus/pci/devices/%04x:%02x:%02x.%01x/local_cpus", pcidomain, pcibus, pcidev, pcifunc);
    if (hwloc_linux_read_path_as_cpumask(path, set) < 0
        || hwloc_bitmap_iszero(set))
        hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#else
    /* 非 Linux 系统简单地获得完整的 cpuset */
    hwloc_bitmap_copy(set, hwloc_topology_get_complete_cpuset(topology));
#endif
  return 0;
}
/**
 * \brief 根据给定索引获取对应于 OpenCL 设备的 hwloc OS 设备对象。
 *
 * \return 描述 OpenCL 设备的 hwloc OS 设备对象，
 * 其中平台索引为 \p platform_index，
 * 设备索引为 \p device_index。
 * \return 如果没有则返回 \c NULL。
 *
 * 拓扑结构 \p topology 不一定要与当前机器匹配。
 * 例如，拓扑结构可以是远程主机的 XML 导入。
 * I/O 设备检测和 OpenCL 组件必须在拓扑结构中启用。
 *
 * \note 相应的 PCI 设备对象可以通过查看 OS 设备的父对象（除非 PCI 设备被过滤掉）来获得。
 */
static __hwloc_inline hwloc_obj_t
hwloc_opencl_get_device_osdev_by_index(hwloc_topology_t topology,
                       unsigned platform_index, unsigned device_index)
{
    unsigned x = (unsigned) -1, y = (unsigned) -1;
    hwloc_obj_t osdev = NULL;
    while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
        if (HWLOC_OBJ_OSDEV_COPROC == osdev->attr->osdev.type
                    && osdev->name
            && sscanf(osdev->name, "opencl%ud%u", &x, &y) == 2
            && platform_index == x && device_index == y)
                        return osdev;
        }
        return NULL;
}
/**
 * \brief 获取与 OpenCL 设备 \p deviceX 对应的 hwloc OS 设备对象。
 *
 * \return 与给定 OpenCL 设备 \p device 对应的 hwloc OS 设备对象。
 * \return 如果找不到，则返回 \c NULL，例如
 * 如果所需的 OpenCL 属性不可用。
 *
 * 此函数目前仅适用于支持相关 OpenCL 扩展的 AMD 和 NVIDIA OpenCL 设备。
 * 应尽可能使用 hwloc_opencl_get_device_osdev_by_index()，即当已知平台和设备索引时。
 *
 * 拓扑 \p topology 和设备 \p device 必须与本地机器匹配。
 * I/O 设备检测和 OpenCL 组件必须在拓扑中启用。
 * 如果没有启用，则仍然可以使用 hwloc_opencl_get_device_cpuset() 找到对象的位置。
 *
 * \note 如果 PCI 设备被过滤掉，此函数无法工作。
 *
 * \note 通过查看结果的父指针（除非 PCI 设备被过滤掉），可以找到相应的 hwloc PCI 设备。
 */
static __hwloc_inline hwloc_obj_t
hwloc_opencl_get_device_osdev(hwloc_topology_t topology __hwloc_attribute_unused,
                  cl_device_id device __hwloc_attribute_unused)
{
    hwloc_obj_t osdev;
    unsigned pcidomain, pcibus, pcidevice, pcifunc;

    if (hwloc_opencl_get_device_pci_busid(device, &pcidomain, &pcibus, &pcidevice, &pcifunc) < 0) {
        errno = EINVAL;
        return NULL;
    }

    osdev = NULL;
    # 循环遍历操作系统设备列表
    while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
        # 获取当前操作系统设备的父对象
        hwloc_obj_t pcidev = osdev->parent;
        # 如果操作系统设备名称不是以 "opencl" 开头，则跳过当前循环
        if (strncmp(osdev->name, "opencl", 6))
            continue;
        # 如果父对象存在，并且是 PCI 设备，并且满足指定的域、总线、设备和函数条件，则返回当前操作系统设备
        if (pcidev
            && pcidev->type == HWLOC_OBJ_PCI_DEVICE
            && pcidev->attr->pcidev.domain == pcidomain
            && pcidev->attr->pcidev.bus == pcibus
            && pcidev->attr->pcidev.dev == pcidevice
            && pcidev->attr->pcidev.func == pcifunc)
            return osdev;
        # 如果 PCI 设备被过滤掉，则需要一个信息属性来匹配
        /* if PCI are filtered out, we need a info attr to match on */
    }
    # 如果没有找到符合条件的操作系统设备，则返回空指针
    return NULL;
// 结束对特定功能模块的注释
/** @} */

// 如果是 C++ 环境，则使用 extern "C" 包裹代码
#ifdef __cplusplus
} /* extern "C" */
#endif

// 结束对 hwloc_opencl.h 文件的条件编译
#endif /* HWLOC_OPENCL_H */
```