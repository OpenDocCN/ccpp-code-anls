# `xmrig\src\3rdparty\CL\cl_d3d11.h`

```
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

#ifndef __OPENCL_CL_D3D11_H
#define __OPENCL_CL_D3D11_H

#include <d3d11.h>  // 包含 D3D11 头文件
#include <CL/cl.h>  // 包含 OpenCL 头文件
#include <CL/cl_platform.h>  // 包含 OpenCL 平台头文件

#ifdef __cplusplus
extern "C" {
#endif

/******************************************************************************
 * cl_khr_d3d11_sharing                                                       */
#define cl_khr_d3d11_sharing 1  // 定义 cl_khr_d3d11_sharing 宏
# 定义 OpenCL 扩展中使用的数据类型 cl_d3d11_device_source_khr 和 cl_d3d11_device_set_khr
typedef cl_uint cl_d3d11_device_source_khr;
typedef cl_uint cl_d3d11_device_set_khr;

/******************************************************************************/

/* Error Codes */
# 定义了一些错误码常量，用于表示不同的错误情况
#define CL_INVALID_D3D11_DEVICE_KHR                  -1006
#define CL_INVALID_D3D11_RESOURCE_KHR                -1007
#define CL_D3D11_RESOURCE_ALREADY_ACQUIRED_KHR       -1008
#define CL_D3D11_RESOURCE_NOT_ACQUIRED_KHR           -1009

/* cl_d3d11_device_source */
# 定义了用于标识 D3D11 设备来源的常量
#define CL_D3D11_DEVICE_KHR                          0x4019
#define CL_D3D11_DXGI_ADAPTER_KHR                    0x401A

/* cl_d3d11_device_set */
# 定义了用于标识 D3D11 设备集合的常量
#define CL_PREFERRED_DEVICES_FOR_D3D11_KHR           0x401B
#define CL_ALL_DEVICES_FOR_D3D11_KHR                 0x401C

/* cl_context_info */
# 定义了用于标识上下文信息的常量
#define CL_CONTEXT_D3D11_DEVICE_KHR                  0x401D
#define CL_CONTEXT_D3D11_PREFER_SHARED_RESOURCES_KHR 0x402D

/* cl_mem_info */
# 定义了用于标识内存对象信息的常量
#define CL_MEM_D3D11_RESOURCE_KHR                    0x401E

/* cl_image_info */
# 定义了用于标识图像对象信息的常量
#define CL_IMAGE_D3D11_SUBRESOURCE_KHR               0x401F

/* cl_command_type */
# 定义了用于标识命令类型的常量
#define CL_COMMAND_ACQUIRE_D3D11_OBJECTS_KHR         0x4020
#define CL_COMMAND_RELEASE_D3D11_OBJECTS_KHR         0x4021

/******************************************************************************/

# 定义了用于从 D3D11 设备获取设备 ID 的函数指针类型
typedef CL_API_ENTRY cl_int (CL_API_CALL *clGetDeviceIDsFromD3D11KHR_fn)(
    cl_platform_id             platform,
    cl_d3d11_device_source_khr d3d_device_source,
    void *                     d3d_object,
    cl_d3d11_device_set_khr    d3d_device_set,
    cl_uint                    num_entries,
    cl_device_id *             devices,
    cl_uint *                  num_devices) CL_API_SUFFIX__VERSION_1_2;

# 定义了用于从 D3D11 缓冲区创建内存对象的函数指针类型
typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromD3D11BufferKHR_fn)(
    cl_context     context,
    cl_mem_flags   flags,
    ID3D11Buffer * resource,
    cl_int *       errcode_ret) CL_API_SUFFIX__VERSION_1_2;

# 定义了用于从 D3D11 2D 纹理创建内存对象的函数指针类型
typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromD3D11Texture2DKHR_fn)(
    cl_context        context,
    # 定义 OpenCL 内存标志，用于描述内存对象的使用方式
    cl_mem_flags      flags,
    # 定义 DirectX 11 纹理对象指针，表示要在 OpenCL 和 DirectX 11 之间共享的纹理对象
    ID3D11Texture2D * resource,
    # 定义子资源索引，表示要访问的纹理对象的子资源索引
    UINT              subresource,
    # 定义错误码指针，用于返回函数调用的错误码
    cl_int *          errcode_ret) CL_API_SUFFIX__VERSION_1_2;
// 定义一个函数指针类型，用于创建从 D3D11 纹理创建 OpenCL 内存对象
typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromD3D11Texture3DKHR_fn)(
    cl_context        context,  // OpenCL 上下文对象
    cl_mem_flags      flags,    // 内存标志
    ID3D11Texture3D * resource,  // D3D11 纹理对象
    UINT              subresource,  // 子资源索引
    cl_int *          errcode_ret) CL_API_SUFFIX__VERSION_1_2;  // 错误码返回值

// 定义一个函数指针类型，用于将 D3D11 对象加入到 OpenCL 命令队列中
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueAcquireD3D11ObjectsKHR_fn)(
    cl_command_queue command_queue,  // OpenCL 命令队列
    cl_uint          num_objects,  // 对象数量
    const cl_mem *   mem_objects,  // 内存对象数组
    cl_uint          num_events_in_wait_list,  // 等待事件数量
    const cl_event * event_wait_list,  // 等待事件列表
    cl_event *       event) CL_API_SUFFIX__VERSION_1_2;  // 事件对象

// 定义一个函数指针类型，用于将 D3D11 对象从 OpenCL 命令队列中释放
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueReleaseD3D11ObjectsKHR_fn)(
    cl_command_queue command_queue,  // OpenCL 命令队列
    cl_uint          num_objects,  // 对象数量
    const cl_mem *   mem_objects,  // 内存对象数组
    cl_uint          num_events_in_wait_list,  // 等待事件数量
    const cl_event * event_wait_list,  // 等待事件列表
    cl_event *       event) CL_API_SUFFIX__VERSION_1_2;  // 事件对象

#ifdef __cplusplus
}
#endif

#endif  /* __OPENCL_CL_D3D11_H */
```