# `xmrig\src\3rdparty\CL\cl_d3d10.h`

```cpp
/*
 * 版权声明，允许在遵循条件的情况下使用和修改材料
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

/*
 * 如果未定义__OPENCL_CL_D3D10_H，则定义__OPENCL_CL_D3D10_H
 */
#ifndef __OPENCL_CL_D3D10_H
#define __OPENCL_CL_D3D10_H

/*
 * 包含相关的头文件
 */
#include <d3d10.h>
#include <CL/cl.h>
#include <CL/cl_platform.h>

#ifdef __cplusplus
extern "C" {
#endif

/******************************************************************************
 * cl_khr_d3d10_sharing                                                       */
#define cl_khr_d3d10_sharing 1
// 定义 OpenCL 与 D3D10 交互所需的数据类型
typedef cl_uint cl_d3d10_device_source_khr;
typedef cl_uint cl_d3d10_device_set_khr;

/******************************************************************************/

/* 错误代码 */
#define CL_INVALID_D3D10_DEVICE_KHR                  -1002
#define CL_INVALID_D3D10_RESOURCE_KHR                -1003
#define CL_D3D10_RESOURCE_ALREADY_ACQUIRED_KHR       -1004
#define CL_D3D10_RESOURCE_NOT_ACQUIRED_KHR           -1005

/* cl_d3d10_device_source_nv */
#define CL_D3D10_DEVICE_KHR                          0x4010
#define CL_D3D10_DXGI_ADAPTER_KHR                    0x4011

/* cl_d3d10_device_set_nv */
#define CL_PREFERRED_DEVICES_FOR_D3D10_KHR           0x4012
#define CL_ALL_DEVICES_FOR_D3D10_KHR                 0x4013

/* cl_context_info */
#define CL_CONTEXT_D3D10_DEVICE_KHR                  0x4014
#define CL_CONTEXT_D3D10_PREFER_SHARED_RESOURCES_KHR 0x402C

/* cl_mem_info */
#define CL_MEM_D3D10_RESOURCE_KHR                    0x4015

/* cl_image_info */
#define CL_IMAGE_D3D10_SUBRESOURCE_KHR               0x4016

/* cl_command_type */
#define CL_COMMAND_ACQUIRE_D3D10_OBJECTS_KHR         0x4017
#define CL_COMMAND_RELEASE_D3D10_OBJECTS_KHR         0x4018

/******************************************************************************/

// 从 D3D10 设备获取 OpenCL 设备 ID
typedef CL_API_ENTRY cl_int (CL_API_CALL *clGetDeviceIDsFromD3D10KHR_fn)(
    cl_platform_id             platform,
    cl_d3d10_device_source_khr d3d_device_source,
    void *                     d3d_object,
    cl_d3d10_device_set_khr    d3d_device_set,
    cl_uint                    num_entries,
    cl_device_id *             devices,
    cl_uint *                  num_devices) CL_API_SUFFIX__VERSION_1_0;

// 从 D3D10 缓冲区创建 OpenCL 内存对象
typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromD3D10BufferKHR_fn)(
    cl_context     context,
    cl_mem_flags   flags,
    ID3D10Buffer * resource,
    cl_int *       errcode_ret) CL_API_SUFFIX__VERSION_1_0;

// 从 D3D10 2D 纹理创建 OpenCL 内存对象
typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromD3D10Texture2DKHR_fn)(
    cl_context        context,
    # 定义一个名为cl_mem_flags的变量，表示OpenCL内存对象的标志
    cl_mem_flags      flags,
    # 定义一个名为resource的变量，表示ID3D10纹理对象的指针
    ID3D10Texture2D * resource,
    # 定义一个名为subresource的变量，表示纹理对象的子资源索引
    UINT              subresource,
    # 定义一个名为errcode_ret的变量，表示返回的错误码
    cl_int *          errcode_ret) CL_API_SUFFIX__VERSION_1_0;
// 定义一个函数指针类型，用于创建从 D3D10 纹理创建 OpenCL 内存对象
typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromD3D10Texture3DKHR_fn)(
    cl_context        context,  // OpenCL 上下文
    cl_mem_flags      flags,    // 内存标志
    ID3D10Texture3D * resource,  // D3D10 纹理资源
    UINT              subresource,  // 子资源
    cl_int *          errcode_ret) CL_API_SUFFIX__VERSION_1_0;  // 错误码返回值

// 定义一个函数指针类型，用于将 D3D10 对象加入到 OpenCL 命令队列中
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueAcquireD3D10ObjectsKHR_fn)(
    cl_command_queue command_queue,  // OpenCL 命令队列
    cl_uint          num_objects,  // 对象数量
    const cl_mem *   mem_objects,  // 内存对象数组
    cl_uint          num_events_in_wait_list,  // 等待事件列表中的事件数量
    const cl_event * event_wait_list,  // 等待事件列表
    cl_event *       event) CL_API_SUFFIX__VERSION_1_0;  // 事件

// 定义一个函数指针类型，用于将 D3D10 对象从 OpenCL 命令队列中释放
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueReleaseD3D10ObjectsKHR_fn)(
    cl_command_queue command_queue,  // OpenCL 命令队列
    cl_uint          num_objects,  // 对象数量
    const cl_mem *   mem_objects,  // 内存对象数组
    cl_uint          num_events_in_wait_list,  // 等待事件列表中的事件数量
    const cl_event * event_wait_list,  // 等待事件列表
    cl_event *       event) CL_API_SUFFIX__VERSION_1_0;  // 事件

#ifdef __cplusplus
}
#endif

#endif  /* __OPENCL_CL_D3D10_H */
```