# `xmrig\src\3rdparty\CL\cl_dx9_media_sharing.h`

```
/*
 * 版权声明，允许在不受限制的情况下使用和处理材料
 */
/**********************************************************************************
 * Copyright (c) 2008-2015 The Khronos Group Inc.
 *
 * Permission is hereby granted, free of charge, to any person obtaining a
 * copy of this software and/or associated documentation files (the
 * "Materials"), to deal in the Materials without restriction, including
 * without limitation the rights to use, copy, modify, merge, publish,
 * distribute, sublicense, and/or sell copies of the Materials, and to
 * permit persons to whom the Materials are furnished to do so, subject to
 * the following conditions:
 *
 * The above copyright notice and this permission notice shall be included
 * in all copies or substantial portions of the Materials.
 *
 * MODIFICATIONS TO THIS FILE MAY MEAN IT NO LONGER ACCURATELY REFLECTS
 * KHRONOS STANDARDS. THE UNMODIFIED, NORMATIVE VERSIONS OF KHRONOS
 * SPECIFICATIONS AND HEADER INFORMATION ARE LOCATED AT
 *    https://www.khronos.org/registry/
 *
 * THE MATERIALS ARE PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 * IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
 * CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
 * TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
 * MATERIALS OR THE USE OR OTHER DEALINGS IN THE MATERIALS.
 **********************************************************************************/

/* $Revision: 11708 $ on $Date: 2010-06-13 23:36:24 -0700 (Sun, 13 Jun 2010) $ */

#ifndef __OPENCL_CL_DX9_MEDIA_SHARING_H
#define __OPENCL_CL_DX9_MEDIA_SHARING_H

#include <CL/cl.h>
#include <CL/cl_platform.h>

#ifdef __cplusplus
extern "C" {
#endif

/******************************************************************************/
/* cl_khr_dx9_media_sharing                                                   */
#define cl_khr_dx9_media_sharing 1
*/
// 定义 OpenCL 中与 DX9 媒体适配器相关的数据类型
typedef cl_uint             cl_dx9_media_adapter_type_khr;
typedef cl_uint             cl_dx9_media_adapter_set_khr;

// 如果是在 Windows 平台下编译，则包含 d3d9.h 头文件
#if defined(_WIN32)
#include <d3d9.h>
// 定义包含 IDirect3DSurface9 和 HANDLE 的结构体
typedef struct _cl_dx9_surface_info_khr
{
    IDirect3DSurface9 *resource;
    HANDLE shared_handle;
} cl_dx9_surface_info_khr;
#endif

/******************************************************************************/

/* 错误代码 */
#define CL_INVALID_DX9_MEDIA_ADAPTER_KHR                -1010
#define CL_INVALID_DX9_MEDIA_SURFACE_KHR                -1011
#define CL_DX9_MEDIA_SURFACE_ALREADY_ACQUIRED_KHR       -1012
#define CL_DX9_MEDIA_SURFACE_NOT_ACQUIRED_KHR           -1013

/* DX9 媒体适配器类型 */
#define CL_ADAPTER_D3D9_KHR                              0x2020
#define CL_ADAPTER_D3D9EX_KHR                            0x2021
#define CL_ADAPTER_DXVA_KHR                              0x2022

/* DX9 媒体适配器集合 */
#define CL_PREFERRED_DEVICES_FOR_DX9_MEDIA_ADAPTER_KHR   0x2023
#define CL_ALL_DEVICES_FOR_DX9_MEDIA_ADAPTER_KHR         0x2024

/* 上下文信息 */
#define CL_CONTEXT_ADAPTER_D3D9_KHR                      0x2025
#define CL_CONTEXT_ADAPTER_D3D9EX_KHR                    0x2026
#define CL_CONTEXT_ADAPTER_DXVA_KHR                      0x2027

/* 内存信息 */
#define CL_MEM_DX9_MEDIA_ADAPTER_TYPE_KHR                0x2028
#define CL_MEM_DX9_MEDIA_SURFACE_INFO_KHR                0x2029

/* 图像信息 */
#define CL_IMAGE_DX9_MEDIA_PLANE_KHR                     0x202A

/* 命令类型 */
#define CL_COMMAND_ACQUIRE_DX9_MEDIA_SURFACES_KHR        0x202B
#define CL_COMMAND_RELEASE_DX9_MEDIA_SURFACES_KHR        0x202C

/******************************************************************************/

// 定义函数指针类型 clGetDeviceIDsFromDX9MediaAdapterKHR_fn
typedef CL_API_ENTRY cl_int (CL_API_CALL *clGetDeviceIDsFromDX9MediaAdapterKHR_fn)(
    cl_platform_id                   platform,
    cl_uint                          num_media_adapters,
    cl_dx9_media_adapter_type_khr *  media_adapter_type,
    // 声明一个指向 void 类型的指针变量 media_adapters
    void *                           media_adapters,
    // 声明一个指向 cl_dx9_media_adapter_set_khr 类型的函数指针变量 media_adapter_set
    cl_dx9_media_adapter_set_khr     media_adapter_set,
    // 声明一个 cl_uint 类型的变量 num_entries
    cl_uint                          num_entries,
    // 声明一个指向 cl_device_id 类型的指针变量 devices
    cl_device_id *                   devices,
    // 声明一个指向 cl_uint 类型的指针变量 num_devices
    cl_uint *                        num_devices) CL_API_SUFFIX__VERSION_1_2;
    // 声明一个带有版本信息的函数声明
// 定义一个函数指针类型，用于创建从 DX9 媒体表面创建 OpenCL 内存对象
typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromDX9MediaSurfaceKHR_fn)(
    cl_context                    context,  // OpenCL 上下文
    cl_mem_flags                  flags,  // 内存标志
    cl_dx9_media_adapter_type_khr adapter_type,  // DX9 媒体适配器类型
    void *                        surface_info,  // 表面信息
    cl_uint                       plane,  // 平面
    cl_int *                      errcode_ret) CL_API_SUFFIX__VERSION_1_2;  // 错误码返回

// 定义一个函数指针类型，用于将 DX9 媒体表面的内存对象加入到命令队列中
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueAcquireDX9MediaSurfacesKHR_fn)(
    cl_command_queue command_queue,  // 命令队列
    cl_uint          num_objects,  // 对象数量
    const cl_mem *   mem_objects,  // 内存对象数组
    cl_uint          num_events_in_wait_list,  // 等待事件列表中的事件数量
    const cl_event * event_wait_list,  // 等待事件列表
    cl_event *       event) CL_API_SUFFIX__VERSION_1_2;  // 事件

// 定义一个函数指针类型，用于将 DX9 媒体表面的内存对象从命令队列中释放
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueReleaseDX9MediaSurfacesKHR_fn)(
    cl_command_queue command_queue,  // 命令队列
    cl_uint          num_objects,  // 对象数量
    const cl_mem *   mem_objects,  // 内存对象数组
    cl_uint          num_events_in_wait_list,  // 等待事件列表中的事件数量
    const cl_event * event_wait_list,  // 等待事件列表
    cl_event *       event) CL_API_SUFFIX__VERSION_1_2;  // 事件

#ifdef __cplusplus
}
#endif

#endif  /* __OPENCL_CL_DX9_MEDIA_SHARING_H */
```