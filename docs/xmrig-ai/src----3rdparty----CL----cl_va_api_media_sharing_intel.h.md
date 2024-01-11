# `xmrig\src\3rdparty\CL\cl_va_api_media_sharing_intel.h`

```
# 版权声明，允许在遵循一定条件下使用和处理材料
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
#ifndef __OPENCL_CL_VA_API_MEDIA_SHARING_INTEL_H
#define __OPENCL_CL_VA_API_MEDIA_SHARING_INTEL_H

#include <CL/cl.h>
#include <CL/cl_platform.h>
#include <va/va.h>

#ifdef __cplusplus
extern "C" {
#endif

/******************************************
* cl_intel_va_api_media_sharing extension *
*******************************************/

#define cl_intel_va_api_media_sharing 1

/* error codes */
#define CL_INVALID_VA_API_MEDIA_ADAPTER_INTEL               -1098
#define CL_INVALID_VA_API_MEDIA_SURFACE_INTEL               -1099
#define CL_VA_API_MEDIA_SURFACE_ALREADY_ACQUIRED_INTEL      -1100
#define CL_VA_API_MEDIA_SURFACE_NOT_ACQUIRED_INTEL          -1101

/* cl_va_api_device_source_intel */
#define CL_VA_API_DISPLAY_INTEL                             0x4094

/* cl_va_api_device_set_intel */
#define CL_PREFERRED_DEVICES_FOR_VA_API_INTEL               0x4095
#define CL_ALL_DEVICES_FOR_VA_API_INTEL                     0x4096

/* cl_context_info */
#define CL_CONTEXT_VA_API_DISPLAY_INTEL                     0x4097

/* cl_mem_info */
#define CL_MEM_VA_API_MEDIA_SURFACE_INTEL                   0x4098

/* cl_image_info */
#define CL_IMAGE_VA_API_PLANE_INTEL                         0x4099

/* cl_command_type */
#define CL_COMMAND_ACQUIRE_VA_API_MEDIA_SURFACES_INTEL      0x409A
#define CL_COMMAND_RELEASE_VA_API_MEDIA_SURFACES_INTEL      0x409B
// 定义 Intel VA API 设备来源的类型
typedef cl_uint cl_va_api_device_source_intel;
// 定义 Intel VA API 设备设置的类型
typedef cl_uint cl_va_api_device_set_intel;

// 从 VA API 媒体适配器获取设备 ID
extern CL_API_ENTRY cl_int CL_API_CALL
clGetDeviceIDsFromVA_APIMediaAdapterINTEL(
    cl_platform_id                platform,  // OpenCL 平台 ID
    cl_va_api_device_source_intel media_adapter_type,  // 媒体适配器类型
    void*                         media_adapter,  // 媒体适配器
    cl_va_api_device_set_intel    media_adapter_set,  // 媒体适配器设置
    cl_uint                       num_entries,  // 设备 ID 数组的大小
    cl_device_id*                 devices,  // 存储设备 ID 的数组
    cl_uint*                      num_devices) CL_EXT_SUFFIX__VERSION_1_2;  // 存储获取到的设备数量

// 定义从 VA API 媒体适配器获取设备 ID 的函数指针类型
typedef CL_API_ENTRY cl_int (CL_API_CALL * clGetDeviceIDsFromVA_APIMediaAdapterINTEL_fn)(
    cl_platform_id                platform,
    cl_va_api_device_source_intel media_adapter_type,
    void*                         media_adapter,
    cl_va_api_device_set_intel    media_adapter_set,
    cl_uint                       num_entries,
    cl_device_id*                 devices,
    cl_uint*                      num_devices) CL_EXT_SUFFIX__VERSION_1_2;

// 从 VA API 媒体表面创建 OpenCL 内存对象
extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateFromVA_APIMediaSurfaceINTEL(
    cl_context                    context,  // OpenCL 上下文
    cl_mem_flags                  flags,  // 内存标志
    VASurfaceID*                  surface,  // VA 表面 ID
    cl_uint                       plane,  // 平面数量
    cl_int*                       errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;  // 错误码

// 定义从 VA API 媒体表面创建 OpenCL 内存对象的函数指针类型
typedef CL_API_ENTRY cl_mem (CL_API_CALL * clCreateFromVA_APIMediaSurfaceINTEL_fn)(
    cl_context                    context,
    cl_mem_flags                  flags,
    VASurfaceID*                  surface,
    cl_uint                       plane,
    cl_int*                       errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;

// 将 VA API 媒体表面加入到 OpenCL 命令队列
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueAcquireVA_APIMediaSurfacesINTEL(
    cl_command_queue              command_queue,  // OpenCL 命令队列
    cl_uint                       num_objects,  // 对象数量
    const cl_mem*                 mem_objects,  // 内存对象数组
    cl_uint                       num_events_in_wait_list,  // 等待事件列表中的事件数量
    const cl_event*               event_wait_list,  // 等待事件列表
    # 定义一个指向cl_event类型指针的指针变量event，使用了CL_EXT_SUFFIX__VERSION_1_2扩展
// 定义一个函数指针类型，用于调用 clEnqueueAcquireVA_APIMediaSurfacesINTEL 函数
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueAcquireVA_APIMediaSurfacesINTEL_fn)(
    cl_command_queue              command_queue,
    cl_uint                       num_objects,
    const cl_mem*                 mem_objects,
    cl_uint                       num_events_in_wait_list,
    const cl_event*               event_wait_list,
    cl_event*                     event) CL_EXT_SUFFIX__VERSION_1_2;

// 声明 clEnqueueReleaseVA_APIMediaSurfacesINTEL 函数
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueReleaseVA_APIMediaSurfacesINTEL(
    cl_command_queue              command_queue,
    cl_uint                       num_objects,
    const cl_mem*                 mem_objects,
    cl_uint                       num_events_in_wait_list,
    const cl_event*               event_wait_list,
    cl_event*                     event) CL_EXT_SUFFIX__VERSION_1_2;

// 定义一个函数指针类型，用于调用 clEnqueueReleaseVA_APIMediaSurfacesINTEL 函数
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueReleaseVA_APIMediaSurfacesINTEL_fn)(
    cl_command_queue              command_queue,
    cl_uint                       num_objects,
    const cl_mem*                 mem_objects,
    cl_uint                       num_events_in_wait_list,
    const cl_event*               event_wait_list,
    cl_event*                     event) CL_EXT_SUFFIX__VERSION_1_2;

// 结束 C++ 的 extern "C" 声明
#ifdef __cplusplus
}
#endif

// 结束条件编译指令，关闭头文件的定义
#endif  /* __OPENCL_CL_VA_API_MEDIA_SHARING_INTEL_H */
```