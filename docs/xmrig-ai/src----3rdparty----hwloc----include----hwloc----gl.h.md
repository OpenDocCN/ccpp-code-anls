# `xmrig\src\3rdparty\hwloc\include\hwloc\gl.h`

```cpp
/*
 * 版权所有 © 2012 Blue Brain Project, EPFL.
 * 版权所有 © 2012-2021 Inria.
 * 请参阅顶层目录中的 COPYING 文件。
 */

/** \file
 * \brief 用于帮助 hwloc 和 OpenGL 显示之间交互的宏。
 *
 * 使用同时使用 hwloc 和 OpenGL 的应用程序可能希望包含
 * 此文件，以便获取 OpenGL 显示的拓扑信息。
 */

#ifndef HWLOC_GL_H
#define HWLOC_GL_H

#include "hwloc.h"

#include <stdio.h>
#include <string.h>


#ifdef __cplusplus
extern "C" {
#endif


/** \defgroup hwlocality_gl 与 OpenGL 显示的互操作性
 *
 * 此接口提供了一种检索有关 OpenGL 显示的拓扑信息的方法。
 *
 * 目前仅支持 NVIDIA 显示位置信息，使用 NV-CONTROL X11 扩展和 NVCtrl 库。
 *
 * @{
 */

/** \brief 获取与给定端口和设备索引的 OpenGL 显示对应的 hwloc OS 设备对象。
 *
 * \return 描述端口（服务器）为 \p port，设备（屏幕）为 \p device 的 OpenGL 显示的 hwloc OS 设备对象。
 * \return 如果找不到，则返回 \c NULL。
 *
 * 拓扑 \p topology 不一定要与当前机器匹配。例如，拓扑可以是远程主机的 XML 导入。
 * I/O 设备检测和 GL 组件必须在拓扑中启用。
 *
 * \note 相应的 PCI 设备对象可以通过查看 OS 设备的父对象（除非过滤了 PCI 设备）来获得。
 */
static __hwloc_inline hwloc_obj_t
hwloc_gl_get_display_osdev_by_port_device(hwloc_topology_t topology,
                      unsigned port, unsigned device)
{
        // 定义两个无符号整数变量x和y，初始化为-1
        unsigned x = (unsigned) -1, y = (unsigned) -1;
        // 定义一个指向hwloc对象的指针osdev，初始化为NULL
        hwloc_obj_t osdev = NULL;
        // 遍历操作系统设备，查找匹配条件的GPU设备
        while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
                // 如果是GPU设备，并且设备名称不为空，并且能够成功解析出x和y，并且port和device与给定的值相等，则返回该设备对象
                if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
                    && osdev->name
                    && sscanf(osdev->name, ":%u.%u", &x, &y) == 2
                    && port == x && device == y)
                        return osdev;
        }
    // 设置错误码为无效参数
    errno = EINVAL;
        // 返回空指针
        return NULL;
}

/** \brief 根据名称获取描述OpenGL显示的hwloc操作系统设备对象。
 *
 * \return 描述名称为\p name的OpenGL显示的hwloc操作系统设备对象，名称格式为":port.device"，例如":0.0"。
 * \return 如果找不到，则返回\c NULL。
 *
 * 拓扑结构\p topology不一定要与当前机器匹配。例如，拓扑结构可以是远程主机的XML导入。I/O设备检测和GL组件必须在拓扑结构中启用。
 *
 * \note 相应的PCI设备对象可以通过查看操作系统设备的父对象（除非PCI设备被过滤掉）来获得。
 */
static __hwloc_inline hwloc_obj_t
hwloc_gl_get_display_osdev_by_name(hwloc_topology_t topology,
                   const char *name)
{
        // 定义一个指向hwloc对象的指针osdev，初始化为NULL
        hwloc_obj_t osdev = NULL;
        // 遍历操作系统设备，查找匹配条件的GPU设备
        while ((osdev = hwloc_get_next_osdev(topology, osdev)) != NULL) {
                // 如果是GPU设备，并且设备名称不为空，并且名称与给定的名称相等，则返回该设备对象
                if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
                    && osdev->name
                    && !strcmp(name, osdev->name))
                        return osdev;
        }
    // 设置错误码为无效参数
    errno = EINVAL;
        // 返回空指针
        return NULL;
}
/**
 * @brief 根据给定的 hwloc OS 对象获取对应的 OpenGL 显示端口和设备
 *
 * 在参数 \p port 中获取对应于给定的 hwloc OS 设备对象的 OpenGL 显示端口（服务器），
 * 在参数 \p screen 中获取对应的设备（屏幕）。
 *
 * @return 如果找不到，则返回 \c -1。
 *
 * 拓扑结构 \p topology 不一定要与当前机器匹配。例如，拓扑结构可以是远程主机的 XML 导入。
 * I/O 设备检测和 GL 组件必须在拓扑结构中启用。
 */
static __hwloc_inline int
hwloc_gl_get_display_by_osdev(hwloc_topology_t topology __hwloc_attribute_unused,
                  hwloc_obj_t osdev,
                  unsigned *port, unsigned *device)
{
    unsigned x = -1, y = -1;
    if (HWLOC_OBJ_OSDEV_GPU == osdev->attr->osdev.type
        && sscanf(osdev->name, ":%u.%u", &x, &y) == 2) {
        *port = x;
        *device = y;
        return 0;
    }
    errno = EINVAL;
    return -1;
}

/** @} */

#ifdef __cplusplus
} /* extern "C" */
#endif

#endif /* HWLOC_GL_H */
```