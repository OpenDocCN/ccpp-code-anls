# `xmrig\src\3rdparty\CL\cl_dx9_media_sharing_intel.h`

```cpp
# 版权声明，允许在遵循条件的情况下使用和修改材料
/**********************************************************************************
 * Copyright (c) 2008-2019 The Khronos Group Inc.
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
/*****************************************************************************\

Copyright (c) 2013-2019 Intel Corporation All Rights Reserved.

THESE MATERIALS ARE PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL INTEL OR ITS
#ifndef __OPENCL_CL_DX9_MEDIA_SHARING_INTEL_H
#define __OPENCL_CL_DX9_MEDIA_SHARING_INTEL_H

#include <CL/cl.h>  // 包含 OpenCL 标准头文件
#include <CL/cl_platform.h>  // 包含 OpenCL 平台相关头文件
#include <d3d9.h>  // 包含 DirectX 9 头文件
#include <dxvahd.h>  // 包含 DirectX Video Acceleration (DXVA) 头文件
#include <wtypes.h>  // 包含 Windows 数据类型头文件
#include <d3d9types.h>  // 包含 DirectX 9 类型头文件

#ifdef __cplusplus
extern "C" {
#endif

/***************************************
* cl_intel_dx9_media_sharing extension *
****************************************/

#define cl_intel_dx9_media_sharing 1  // 定义 OpenCL 扩展标识

typedef cl_uint cl_dx9_device_source_intel;  // 定义 OpenCL DirectX 9 设备来源类型
typedef cl_uint cl_dx9_device_set_intel;  // 定义 OpenCL DirectX 9 设备集类型

/* error codes */
#define CL_INVALID_DX9_DEVICE_INTEL                   -1010  // 定义 OpenCL 错误码
#define CL_INVALID_DX9_RESOURCE_INTEL                 -1011  // 定义 OpenCL 错误码
#define CL_DX9_RESOURCE_ALREADY_ACQUIRED_INTEL        -1012  // 定义 OpenCL 错误码
#define CL_DX9_RESOURCE_NOT_ACQUIRED_INTEL            -1013  // 定义 OpenCL 错误码

/* cl_dx9_device_source_intel */
#define CL_D3D9_DEVICE_INTEL                          0x4022  // 定义 OpenCL DirectX 9 设备来源
#define CL_D3D9EX_DEVICE_INTEL                        0x4070  // 定义 OpenCL DirectX 9Ex 设备来源
#define CL_DXVA_DEVICE_INTEL                          0x4071  // 定义 OpenCL DXVA 设备来源

/* cl_dx9_device_set_intel */
#define CL_PREFERRED_DEVICES_FOR_DX9_INTEL            0x4024  // 定义 OpenCL 首选 DirectX 9 设备集
#define CL_ALL_DEVICES_FOR_DX9_INTEL                  0x4025  // 定义 OpenCL 所有 DirectX 9 设备集

/* cl_context_info */
#define CL_CONTEXT_D3D9_DEVICE_INTEL                  0x4026  // 定义 OpenCL 上下文中的 DirectX 9 设备信息
#define CL_CONTEXT_D3D9EX_DEVICE_INTEL                0x4072  // 定义 OpenCL 上下文中的 DirectX 9Ex 设备信息
#define CL_CONTEXT_DXVA_DEVICE_INTEL                  0x4073  // 定义 OpenCL 上下文中的 DXVA 设备信息

/* cl_mem_info */
// 定义 OpenCL 内存对象的特定标识符，用于表示与 DirectX 9 相关的资源和共享句柄
#define CL_MEM_DX9_RESOURCE_INTEL                     0x4027
#define CL_MEM_DX9_SHARED_HANDLE_INTEL                0x4074

/* cl_image_info */
// 定义用于表示与 DirectX 9 相关的图像平面信息的标识符
#define CL_IMAGE_DX9_PLANE_INTEL                      0x4075

// 定义用于表示与 DirectX 9 相关的命令类型的标识符
#define CL_COMMAND_ACQUIRE_DX9_OBJECTS_INTEL          0x402A
#define CL_COMMAND_RELEASE_DX9_OBJECTS_INTEL          0x402B
/******************************************************************************/

// 从 DirectX 9 设备获取 OpenCL 设备 ID
extern CL_API_ENTRY cl_int CL_API_CALL
clGetDeviceIDsFromDX9INTEL(
    cl_platform_id              platform,
    cl_dx9_device_source_intel  dx9_device_source,
    void*                       dx9_object,
    cl_dx9_device_set_intel     dx9_device_set,
    cl_uint                     num_entries,
    cl_device_id*               devices,
    cl_uint*                    num_devices) CL_EXT_SUFFIX__VERSION_1_1;

// 定义用于从 DirectX 9 设备获取 OpenCL 设备 ID 的函数指针类型
typedef CL_API_ENTRY cl_int (CL_API_CALL* clGetDeviceIDsFromDX9INTEL_fn)(
    cl_platform_id              platform,
    cl_dx9_device_source_intel  dx9_device_source,
    void*                       dx9_object,
    cl_dx9_device_set_intel     dx9_device_set,
    cl_uint                     num_entries,
    cl_device_id*               devices,
    cl_uint*                    num_devices) CL_EXT_SUFFIX__VERSION_1_1;

// 从 DirectX 9 媒体表面创建 OpenCL 内存对象
extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateFromDX9MediaSurfaceINTEL(
    cl_context                  context,
    cl_mem_flags                flags,
    IDirect3DSurface9*          resource,
    HANDLE                      sharedHandle,
    UINT                        plane,
    cl_int*                     errcode_ret) CL_EXT_SUFFIX__VERSION_1_1;

// 定义用于从 DirectX 9 媒体表面创建 OpenCL 内存对象的函数指针类型
typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromDX9MediaSurfaceINTEL_fn)(
    cl_context                  context,
    cl_mem_flags                flags,
    IDirect3DSurface9*          resource,
    HANDLE                      sharedHandle,
    UINT                        plane,
    cl_int*                     errcode_ret) CL_EXT_SUFFIX__VERSION_1_1;
# 定义一个外部函数，用于在 Intel 平台上将 DX9 对象加入到命令队列中
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueAcquireDX9ObjectsINTEL(
    cl_command_queue            command_queue,  # 命令队列对象
    cl_uint                     num_objects,     # 要加入的对象数量
    const cl_mem*               mem_objects,     # 对象数组
    cl_uint                     num_events_in_wait_list,  # 等待事件列表中的事件数量
    const cl_event*             event_wait_list,  # 等待事件列表
    cl_event*                   event) CL_EXT_SUFFIX__VERSION_1_1;  # 事件对象

# 定义一个函数指针类型，用于在 Intel 平台上将 DX9 对象加入到命令队列中
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueAcquireDX9ObjectsINTEL_fn)(
    cl_command_queue            command_queue,  # 命令队列对象
    cl_uint                     num_objects,     # 要加入的对象数量
    const cl_mem*               mem_objects,     # 对象数组
    cl_uint                     num_events_in_wait_list,  # 等待事件列表中的事件数量
    const cl_event*             event_wait_list,  # 等待事件列表
    cl_event*                   event) CL_EXT_SUFFIX__VERSION_1_1;  # 事件对象

# 定义一个外部函数，用于在 Intel 平台上将 DX9 对象释放出命令队列
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueReleaseDX9ObjectsINTEL(
    cl_command_queue            command_queue,  # 命令队列对象
    cl_uint                     num_objects,     # 要释放的对象数量
    cl_mem*                     mem_objects,     # 对象数组
    cl_uint                     num_events_in_wait_list,  # 等待事件列表中的事件数量
    const cl_event*             event_wait_list,  # 等待事件列表
    cl_event*                   event) CL_EXT_SUFFIX__VERSION_1_1;  # 事件对象

# 定义一个函数指针类型，用于在 Intel 平台上将 DX9 对象释放出命令队列
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueReleaseDX9ObjectsINTEL_fn)(
    cl_command_queue            command_queue,  # 命令队列对象
    cl_uint                     num_objects,     # 要释放的对象数量
    cl_mem*                     mem_objects,     # 对象数组
    cl_uint                     num_events_in_wait_list,  # 等待事件列表中的事件数量
    const cl_event*             event_wait_list,  # 等待事件列表
    cl_event*                   event) CL_EXT_SUFFIX__VERSION_1_1;  # 事件对象

#ifdef __cplusplus
}
#endif

#endif  /* __OPENCL_CL_DX9_MEDIA_SHARING_INTEL_H */
```