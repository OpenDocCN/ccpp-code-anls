# xmrig源码解析 12

# `src/3rdparty/CL/cl_va_api_media_sharing_intel.h`

这段代码是一个C库，名为“墨客-示例”，它被提供给了用户许可，允许用户免费在各种方式下使用、复制、修改、编译、发布、分发、销售软件及其文档。这个库被认为遵循Khronos标准，Khronos标准是一种规范，定义了物联网设备和服务的安全和交互式通信的标准。

这个库的主要目的是提供一个简单示例，演示了如何使用C库来在支持Khronos标准的设备上发送RESTful HTTP请求，并展示了一些常见的Khronos功能，例如做什么、如何使用Khronos Stats、Khronos Certification等。


```cpp
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
```

这段代码是一个名为“*************”的文件，可能是从互联网上某处下载下来的。从代码中可以看出，它包含了一些头文件、函数声明和注释。以下是对这些内容的简要解释：

头文件：

这个文件中包含了一些头文件，它们定义了程序中需要用到的数据结构和函数。

函数声明：

这个文件中声明了一些函数，它们的目的是在程序中进行一些操作。

注释：

这个文件中有一些注释，它们描述了程序的功能、用途或者实现方式。


```cpp
/*****************************************************************************\

Copyright (c) 2013-2019 Intel Corporation All Rights Reserved.

THESE MATERIALS ARE PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL INTEL OR ITS
CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THESE
MATERIALS, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

```



这段代码定义了一个名为`cl_va_api_media_sharing_intel.h`的文件，其中包含了OpenCL的定义和声明。

具体来说，这个文件中的定义包括：

- `#include <CL/cl.h>`和`#include <CL/cl_platform.h>`包含了OpenCL的基本头文件和平台文件，以便其他依赖库正确地包含OpenCL。

- `#include <va/va.h>`包含了V8000(VA)接口文件，以便支持将OpenCL的计算和内存模型与V8000(VA)的硬件和软件接口进行交互。

- `abstract long long ml_cluster_id = 0;`定义了一个名为`ml_cluster_id`的变量，其值为`0`，意味着还没有计算节点加入过集群。

- `void init_opencl_api()`函数，该函数初始化OpenCL API，包括在计算节点上创建计算区域、创建图形管道以及创建执行计划等。

- `int create_execution_plan(int node_id, int panel_id, int num_intentions, int *仅供_data_count)`函数，该函数根据传入的节点ID、面板ID和意图数量，创建一个执行计划并返回。

- `int ml_init(int context_id, int ml_device_id, int ml_socket_id, int intel_api_version, int *num_domains)`函数，该函数用于初始化ML环境，包括设置InFO摆钟、创建内存区域、创建执行计划等。

- `void ml_shutdown(int context_id, int ml_device_id, int intel_api_version)`函数，该函数用于关闭ML环境。

- `int ml_create_device(int context_id, int ml_device_id, int *device_fd)`函数，该函数用于在ML环境中创建一个设备，并返回设备的文件描述符。

- `int ml_connect(int context_id, int device_id, int socket_id, int intel_api_version)`函数，该函数用于在ML环境中连接到图形服务器，并返回一个操作码，用于将执行计划卸载到图形服务器。

- `int ml_send_command(int context_id, int device_id, int command_id, int intel_api_version, int64 *data_count)`函数，该函数用于在ML环境中发送命令到设备，并返回命令ID和发送的数据计数器。

- `int ml_receive_command(int context_id, int device_id, int command_id, int intel_api_version, int64 *data_count)`函数，该函数用于在ML环境中接收命令，并返回接收的命令ID和接收的数据计数器。

- `int ml_create_execution_plan2(int context_id, int ml_device_id, int ml_socket_id, int num_intentions, int *仅供_data_count)`函数，该函数类似于`create_execution_plan`函数，但允许在创建执行计划时指定允许使用的数据类型。

- `int ml_run_execution_plan(int context_id, int ml_device_id, int ml_socket_id, int num_domain_args, int *domain_args, int intel_api_version, int *num_domain_results)`函数，该函数用于在ML环境中运行执行计划，并返回结果。

- `int ml_get_execution_plan2(int context_id, int ml_device_id, int ml_socket_id, int num_domain_args, int *domain_args, int intel_api_version, int *num_domain_results)`函数，该函数类似于`get_execution_plan2`函数，但允许在获取执行计划时指定允许使用的数据类型。


```cpp
File Name: cl_va_api_media_sharing_intel.h

Abstract:

Notes:

\*****************************************************************************/


#ifndef __OPENCL_CL_VA_API_MEDIA_SHARING_INTEL_H
#define __OPENCL_CL_VA_API_MEDIA_SHARING_INTEL_H

#include <CL/cl.h>
#include <CL/cl_platform.h>
#include <va/va.h>

```

这段代码是一个C++预处理指令，它通过#ifdef和#endif定义了两个头文件，然后在预处理阶段编译时检查两个声明是否为真，如果是，那么编译器会替换掉两个头文件中的所有内容，即：
```cpp
#ifdef __cplusplus
extern "C" {
#endif
```
接下来定义了一个名为cl_intel_va_api_media_sharing的宏，它的值为1。
```cpp
#define cl_intel_va_api_media_sharing 1
```
接着定义了一些错误码，它们定义了CL_INVALID_VA_API_MEDIA_ADAPTER_INTEL，CL_INVALID_VA_API_MEDIA_SURFACE_INTEL，CL_VA_API_MEDIA_SURFACE_ALREADY_ACQUIRED_INTEL，CL_VA_API_MEDIA_SURFACE_NOT_ACQUIRED_INTEL这四个宏。
```cpp
#define CL_INVALID_VA_API_MEDIA_ADAPTER_INTEL                -1098
#define CL_INVALID_VA_API_MEDIA_SURFACE_INTEL            -1099
#define CL_VA_API_MEDIA_SURFACE_ALREADY_ACQUIRED_INTEL    -1100
#define CL_VA_API_MEDIA_SURFACE_NOT_ACQUIRED_INTEL      -1101
```
最后，通过#define定义了一个名为cl_intel_va_api_media_sharing的常量，它的值为1。
```cpp
#define cl_intel_va_api_media_sharing 1
```
然后通过这个常量来定义宏，这样就可以使用宏定义中的内容了。


```cpp
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

```

这段代码定义了一系列头文件和常量，用于定义和控制与IntelVA API相关的上下文信息。

具体来说，这些头文件定义了几个寄存器或常量，用于设置或获取与IntelVA API相关的信息。例如，定义了一个名为CL_VA_API_DISPLAY_INTEL的常量，表示IntelVA API的显示ID为0x4094。

另一个头文件定义了一系列常量，用于指定IntelVA API的最佳设备设置。例如，定义了一个名为CL_PREFERRED_DEVICES_FOR_VA_API_INTEL的常量，表示使用最好的Intel设备设置来获得最好的性能。

其他头文件还定义了一些与IntelVA API相关的寄存器或常量，例如CL_CONTEXT_VA_API_DISPLAY_INTEL，用于获取或设置IntelVA API的显示ID。

最后，这些头文件还定义了一些与IntelVA API相关的函数，例如CL_VA_API_SET_INTEL，用于设置IntelVA API的显示ID。


```cpp
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

```

这段代码定义了两个宏定义：CL_COMMAND_ACQUIRE_VA_API_MEDIA_SURFACES_INTEL 和 CL_COMMAND_RELEASE_VA_API_MEDIA_SURFACES_INTEL。它们定义了两个与 VA API 相关的命令类型：获取和释放。

接下来的两行代码定义了一个名为 CL_VA_API_DEVICE_SOURCE_INTEL 的类型，它用于从VA API中获取设备ID。接下来的两行代码定义了一个名为 CL_VA_API_DEVICE_SET_INTEL 的类型，它用于设置设备ID。

最后，代码定义了一个名为 CL_INT 的函数，它接受一个名为 platform 的参数，用于指定要获取或释放的设备类型。接下来的两个函数参数：media_adapter_type 和 media_adapter，用于指定从VA API中获取或设置的媒体适配器类型和设置的设备ID。函数还接受两个名为 num_entries 和 num_devices 的参数，用于指定要返回的设备数量。函数内部使用 clGetDeviceIDsFromVA_APIMediaAdapterINTEL 函数获取设备ID，并将其存储在 devices 和 num_devices 指向的内存区域中。


```cpp
/* cl_command_type */
#define CL_COMMAND_ACQUIRE_VA_API_MEDIA_SURFACES_INTEL      0x409A
#define CL_COMMAND_RELEASE_VA_API_MEDIA_SURFACES_INTEL      0x409B

typedef cl_uint cl_va_api_device_source_intel;
typedef cl_uint cl_va_api_device_set_intel;

extern CL_API_ENTRY cl_int CL_API_CALL
clGetDeviceIDsFromVA_APIMediaAdapterINTEL(
    cl_platform_id                platform,
    cl_va_api_device_source_intel media_adapter_type,
    void*                         media_adapter,
    cl_va_api_device_set_intel    media_adapter_set,
    cl_uint                       num_entries,
    cl_device_id*                 devices,
    cl_uint*                      num_devices) CL_EXT_SUFFIX__VERSION_1_2;

```

这段代码定义了一个名为“clGetDeviceIDsFromVA_APIMediaAdapterINTEL_fn”的函数指针类型。

函数接受4个参数，分别为：
- “platform”表示输入的CL平台ID；
- “media_adapter_type”表示输入的APIMediaAdapter类型；
- “media_adapter”表示输入的媒体适配器；
- “num_entries”表示要返回的设备ID数量。

函数内部定义了一个名为“clGetDeviceIDsFromVA_APIMediaAdapterINTEL_fn”的函数，该函数接收一个APIMediaAdapter作为参数，返回一个CLDeviceID数组，其中第一个设备ID是调用者传递给函数的设备ID，后面跟着的设备ID数量由输入的num_entries参数定义。

函数定义后，需要在头文件中定义该函数，以便在程序中使用。


```cpp
typedef CL_API_ENTRY cl_int (CL_API_CALL * clGetDeviceIDsFromVA_APIMediaAdapterINTEL_fn)(
    cl_platform_id                platform,
    cl_va_api_device_source_intel media_adapter_type,
    void*                         media_adapter,
    cl_va_api_device_set_intel    media_adapter_set,
    cl_uint                       num_entries,
    cl_device_id*                 devices,
    cl_uint*                      num_devices) CL_EXT_SUFFIX__VERSION_1_2;

extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateFromVA_APIMediaSurfaceINTEL(
    cl_context                    context,
    cl_mem_flags                  flags,
    VASurfaceID*                  surface,
    cl_uint                       plane,
    cl_int*                       errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;

```

这段代码定义了一个名为“cl_mem”的CL扩展函数。

“typedef”定义了cl_mem这个名字，宣告了一个名为“clCreateFromVA_APIMediaSurfaceINTEL_fn”的函数指针。

函数指针的参数列表为：
- “cl_context”表示上下文，上下文可以提供访问CL状态的函数指针。
- “cl_mem_flags”表示内存标志，这些标志可以设置为读或写。
- “VASurfaceID*”表示一个VASurfaceID指针，可以用于输入或输出CL的媒体表面。
- “cl_uint”表示一个无符号整数，表示想要创建的媒体表面的索引或ID。
- “cl_int*”表示一个无符号整数，用于存储在等待列表中的CL事件编号。
- “CL_EXT_SUFFIX__VERSION_1_2”表示此函数的版本，有助于与其他CL库的版本兼容。

接下来定义了一个名为“cl_int”的函数，它接收命令队列，要创建的媒体表面数量，以及一个内存对象列表和一个等待列表。然后，它调用了一个内部函数“clCreateFromVA_APIMediaSurfaceINTEL”，传递所需的参数，然后返回其返回值。

最后定义了“cl_mem”函数的原型，它将上述定义的所有参数组合起来，用于在命令队列中调用“clCreateFromVA_APIMediaSurfaceINTEL”。


```cpp
typedef CL_API_ENTRY cl_mem (CL_API_CALL * clCreateFromVA_APIMediaSurfaceINTEL_fn)(
    cl_context                    context,
    cl_mem_flags                  flags,
    VASurfaceID*                  surface,
    cl_uint                       plane,
    cl_int*                       errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;

extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueAcquireVA_APIMediaSurfacesINTEL(
    cl_command_queue              command_queue,
    cl_uint                       num_objects,
    const cl_mem*                 mem_objects,
    cl_uint                       num_events_in_wait_list,
    const cl_event*               event_wait_list,
    cl_event*                     event) CL_EXT_SUFFIX__VERSION_1_2;

```

这段代码定义了一个名为`clEnqueueAcquireVA_APIMediaSurfacesINTEL_fn`的函数指针类型。该函数指针类型包含两个参数：`cl_command_queue`和`cl_uint`类型。

该函数指针所代表的函数接受一个命令队列`cl_command_queue`作为参数，然后等待指定数量的`cl_uint`类型的参数，这些参数在接下来的函数中使用。

接下来是四个函数参数：

1. `command_queue`参数，指代一个命令队列，是输入参数。
2. `num_objects`参数，是一个无符号整数，表示要管理的对象数，也是输入参数。
3. `mem_objects`参数，是一个无符号整数数组，表示要管理的内存数，也是输入参数。
4. `num_events_in_wait_list`参数，是一个无符号整数，表示正在等待的事件数，也是输入参数。
5. `event_wait_list`参数，是一个指向事件等待列表的指针，也是输入参数。
6. `event`参数，是一个事件变量，它是输出参数。

最后定义了一个外部函数`CL_API_CALL`，该函数使用该函数指针。


```cpp
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueAcquireVA_APIMediaSurfacesINTEL_fn)(
    cl_command_queue              command_queue,
    cl_uint                       num_objects,
    const cl_mem*                 mem_objects,
    cl_uint                       num_events_in_wait_list,
    const cl_event*               event_wait_list,
    cl_event*                     event) CL_EXT_SUFFIX__VERSION_1_2;

extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueReleaseVA_APIMediaSurfacesINTEL(
    cl_command_queue              command_queue,
    cl_uint                       num_objects,
    const cl_mem*                 mem_objects,
    cl_uint                       num_events_in_wait_list,
    const cl_event*               event_wait_list,
    cl_event*                     event) CL_EXT_SUFFIX__VERSION_1_2;

```

这段代码定义了一个名为“cl_int”的CL类型，该类型代表了一个名为“CL_API_ENTRY”的函数指针，该函数指针类型由一个名为“CL_API_CALL”的函数与一个名为“intEL_fn”的函数组合而成。

函数指针“intEl_fn”接收五个参数：一个命令队列、一个事件数量、一个事件等待列表、一个事件列表和一个人工选中列表（如果有的话）。然后，该函数通过命令队列将指定数量的表面（CL_MEM_OBJECT）从系统中的资源分配，并等待用户输入的信号（CL_EVENT）事件，如果该事件为“CL_EVENT_SIGNALED”。最后，函数将释放分配给它的所有表面，并返回给调用它的函数。


```cpp
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueReleaseVA_APIMediaSurfacesINTEL_fn)(
    cl_command_queue              command_queue,
    cl_uint                       num_objects,
    const cl_mem*                 mem_objects,
    cl_uint                       num_events_in_wait_list,
    const cl_event*               event_wait_list,
    cl_event*                     event) CL_EXT_SUFFIX__VERSION_1_2;

#ifdef __cplusplus
}
#endif

#endif  /* __OPENCL_CL_VA_API_MEDIA_SHARING_INTEL_H */


```

# `src/3rdparty/CL/cl_version.h`

这段代码是一个C文件，它定义了一个名为"matrix"的函数。这个函数的作用是打印出从从左到右、从上到下的一个3x3的矩阵。它的实现包括以下几个步骤：

1. 包含头文件：通过在函数的第一行添加"/倾城之誉。'、和条件运算符大括号来包含下面定义的矩阵。

2. 定义常量：在函数的第五行到第十行之间定义了4个常量连名称的"matrix"、和字符串。

3. 函数体：在函数的最后5行中实现了打印矩阵的功能。

4. 输出结果：在函数的最后10行中实现了打印结果的功能。

因此，这段代码的主要目的是定义一个名为"matrix"的函数，该函数可以打印出一个3x3的矩阵。


```cpp
/*******************************************************************************
 * Copyright (c) 2018 The Khronos Group Inc.
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
 ******************************************************************************/

```

这段代码定义了一个头文件 called "__cl-version.h"，用于定义 OpenCL 的版本号。通过检测是否定义了该头文件，代码会根据不同的条件选择不同的 OpenCL 版本号。

具体来说，如果 !defined(CL_TARGET_OPENCL_VERSION)，那么代码会输出一条消息，其中会表明 OpenCL 版本号不是定义好的。如果定义了 OpenCL 版本号，则需要根据版本号检查是否为 100、110、120、200、210 或 220 中的任何一个。如果是这些版本中的一个，则输出一条消息，否则输出另一条消息。

如果不存在这些 version，或者存在但不匹配任何一条 version，代码会输出另一条消息。在这条消息中，会给出一个指向字符串的指针，该指针包含一个格式化的字符串，用于将 OpenCL 版本号作为消息返回。该字符串的格式为 "%d (%s) %s\n"，其中 %d 表示 OpenCL 版本号，%s 表示版本描述，%s 表示错误信息。


```cpp
#ifndef __CL_VERSION_H
#define __CL_VERSION_H

/* Detect which version to target */
#if !defined(CL_TARGET_OPENCL_VERSION)
#pragma message("cl_version.h: CL_TARGET_OPENCL_VERSION is not defined. Defaulting to 220 (OpenCL 2.2)")
#define CL_TARGET_OPENCL_VERSION 220
#endif
#if CL_TARGET_OPENCL_VERSION != 100 && \
    CL_TARGET_OPENCL_VERSION != 110 && \
    CL_TARGET_OPENCL_VERSION != 120 && \
    CL_TARGET_OPENCL_VERSION != 200 && \
    CL_TARGET_OPENCL_VERSION != 210 && \
    CL_TARGET_OPENCL_VERSION != 220
#pragma message("cl_version: CL_TARGET_OPENCL_VERSION is not a valid value (100, 110, 120, 200, 210, 220). Defaulting to 220 (OpenCL 2.2)")
```

这段代码定义了一个OpenCL版本号，并根据该版本号输出一些相关的定义和注释。

具体来说，代码中定义了三个条件语句，每个语句都基于一个或多个预设条件，如果条件成立，则执行相应的代码块。这些条件语句如下：

1. `#if CL_TARGET_OPENCL_VERSION >= 220 && !defined(CL_VERSION_2_2)`

这个条件语句表示，如果CL的目标OpenCL版本号大于或等于220，但是没有定义CL_VERSION_2_2，那么将CL_VERSION_2_2的值设为1。这里使用了`!defined`这个逻辑运算符，它会检查是否定义了某个标识符，如果是则返回FALSE，否则返回TRUE。

2. `#if CL_TARGET_OPENCL_VERSION >= 210 && !defined(CL_VERSION_2_1)`

这个条件语句表示，如果CL的目标OpenCL版本号大于或等于210，但是没有定义CL_VERSION_2_1，那么将CL_VERSION_2_1的值设为1。同样使用了`!defined`这个逻辑运算符。

3. `#if CL_TARGET_OPENCL_VERSION >= 200 && !defined(CL_VERSION_2_0)`

这个条件语句表示，如果CL的目标OpenCL版本号大于或等于200，但是没有定义CL_VERSION_2_0，那么将CL_VERSION_2_0的值设为1。同样使用了`!defined`这个逻辑运算符。

最后，定义了几个宏定义，用于将上述条件语句中的版本号替换为实际定义的版本号。


```cpp
#undef CL_TARGET_OPENCL_VERSION
#define CL_TARGET_OPENCL_VERSION 220
#endif


/* OpenCL Version */
#if CL_TARGET_OPENCL_VERSION >= 220 && !defined(CL_VERSION_2_2)
#define CL_VERSION_2_2  1
#endif
#if CL_TARGET_OPENCL_VERSION >= 210 && !defined(CL_VERSION_2_1)
#define CL_VERSION_2_1  1
#endif
#if CL_TARGET_OPENCL_VERSION >= 200 && !defined(CL_VERSION_2_0)
#define CL_VERSION_2_0  1
#endif
```

这段代码是用于检查OpenCL版本是否支持某些特定的API，并根据版本号来定义对应的常量。

具体来说，代码首先检查当前OpenCL版本是否支持Cl_TARGET_OPENCL_VERSION的120及以上的版本，如果不支持，则定义CL_VERSION_1_2的常量为1。接着，再检查是否支持Cl_TARGET_OPENCL_VERSION的110及以上的版本，如果不支持，则定义CL_VERSION_1_1的常量为1。类似地，对于Cl_TARGET_OPENCL_VERSION的100及以上的版本，则定义CL_VERSION_1_0的常量为1。

此外，代码还定义了一个名为CL_USE_DEPRECATED_OPENCL_2_1_APIS的常量，用于判断当前OpenCL版本是否小于2.1版本，如果是，则定义为真，否则为假。这意味着即使当前OpenCL版本小于2.1，也可以使用Cl_DEPRECATED_OPENCL_2_1_API，但使用这个API是过时的，应该尽量避免。


```cpp
#if CL_TARGET_OPENCL_VERSION >= 120 && !defined(CL_VERSION_1_2)
#define CL_VERSION_1_2  1
#endif
#if CL_TARGET_OPENCL_VERSION >= 110 && !defined(CL_VERSION_1_1)
#define CL_VERSION_1_1  1
#endif
#if CL_TARGET_OPENCL_VERSION >= 100 && !defined(CL_VERSION_1_0)
#define CL_VERSION_1_0  1
#endif

/* Allow deprecated APIs for older OpenCL versions. */
#if CL_TARGET_OPENCL_VERSION <= 210 && !defined(CL_USE_DEPRECATED_OPENCL_2_1_APIS)
#define CL_USE_DEPRECATED_OPENCL_2_1_APIS
#endif
#if CL_TARGET_OPENCL_VERSION <= 200 && !defined(CL_USE_DEPRECATED_OPENCL_2_0_APIS)
```

这段代码是一个C语言的预处理指令，用于定义OpenCL的不同版本和API。

具体来说，代码中定义了多个宏定义：

- CL_USE_DEPRECATED_OPENCL_2_0_APIS：如果OpenCL版本2.0及更高版本，则使用定义为真，否则使用定义为假。
- CL_TARGET_OPENCL_VERSION：当前OpenCL版本。

- CL_USE_DEPRECATED_OPENCL_1_2_APIS：如果OpenCL版本1.2及更高版本，则使用定义为真，否则使用定义为假。
- CL_TARGET_OPENCL_VERSION <= 120：如果当前OpenCL版本小于或等于120，则使用定义为真，否则使用定义为假。
- !defined(CL_USE_DEPRECATED_OPENCL_1_1_APIS)：如果当前OpenCL版本1.1，则使用定义为真，否则使用定义为假。
- !defined(CL_USE_DEPRECATED_OPENCL_1_0_APIS)：如果当前OpenCL版本1.0，则使用定义为真，否则使用定义为假。

如果当前OpenCL版本小于或等于100，则使用定义为真。

如果当前OpenCL版本大于100，则使用定义为假。

定义后面还包含一个！定义，用于否定上面所有条件，即不是上述任何情况。


```cpp
#define CL_USE_DEPRECATED_OPENCL_2_0_APIS
#endif
#if CL_TARGET_OPENCL_VERSION <= 120 && !defined(CL_USE_DEPRECATED_OPENCL_1_2_APIS)
#define CL_USE_DEPRECATED_OPENCL_1_2_APIS
#endif
#if CL_TARGET_OPENCL_VERSION <= 110 && !defined(CL_USE_DEPRECATED_OPENCL_1_1_APIS)
#define CL_USE_DEPRECATED_OPENCL_1_1_APIS
#endif
#if CL_TARGET_OPENCL_VERSION <= 100 && !defined(CL_USE_DEPRECATED_OPENCL_1_0_APIS)
#define CL_USE_DEPRECATED_OPENCL_1_0_APIS
#endif

#endif  /* __CL_VERSION_H */

```

# `src/3rdparty/CL/opencl.h`

这段代码是一个C语言的函数声明，它定义了一个名为“hello”的函数，该函数接受两个整数参数“GET_INTEGER_RETURN_INT”和“SET_INTEGER_RETURN_INT”。函数的功能是接收两个整数类型的输入，计算它们的和并返回它们的和。

具体来说，这个函数接收两个整数，然后将它们相加，然后将结果存储回原来的变量。如果两个输入值相同，函数将返回相同的结果，并在调用函数时传入相同的值。

函数声明中没有包含任何错误检查，因此使用这个函数时需要确保输入参数的值是正确的。此外，函数的实现也没有进行任何实际的 checks，因此也需要确保输入参数的值是正确的。


```cpp
/*******************************************************************************
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
 ******************************************************************************/

```

这段代码是一个C/C++的预处理指令，它定义了一个头文件叫 "__openshell_api"，然后在头文件中导入了CL和CL的gl扩展。通过预处理指令，这段代码会使得编译器在编译之前检查是否定义了这个头文件，从而避免编译错误。同时，它还定义了一个头文件名称为 "__openshell_api"，以便在代码中引用它。


```cpp
/* $Revision: 11708 $ on $Date: 2010-06-13 23:36:24 -0700 (Sun, 13 Jun 2010) $ */

#ifndef __OPENCL_H
#define __OPENCL_H

#ifdef __cplusplus
extern "C" {
#endif

#include <CL/cl.h>
#include <CL/cl_gl.h>
#include <CL/cl_gl_ext.h>
#include <CL/cl_ext.h>

#ifdef __cplusplus
}
```

这段代码是一个 preprocess 指令，用于告诉编译器在编译之前需要处理一下定义。

具体来说，当源代码文件(__OPENCL_H)被包含进来的时候，编译器会检查其中是否定义了某些特定的头文件(__OPENCL_PRIVATE_HEADERS)，如果缺少，就会输出这个头文件的内容，从而避免编译错误。

这里的 #ifdef __OPENCL_H 表示判断(__OPENCL_H)是否被定义为真，如果是，那么下面的 __OPENCL_PRIVATE_HEADERS 就会被执行，否则就不会被执行。


```cpp
#endif

#endif  /* __OPENCL_H   */

```

# OpenCL<sup>TM</sup> API Headers

This repository contains C language headers for the OpenCL API.

The authoritative public repository for these headers is located at:

https://github.com/KhronosGroup/OpenCL-Headers

Issues, proposed fixes for issues, and other suggested changes should be
created using Github.

## Branch Structure

The OpenCL API headers in this repository are Unified headers and are designed
to work with all released OpenCL versions.  This differs from previous OpenCL
API headers, where version-specific API headers either existed in separate
branches, or in separate folders in a branch.

## Compiling for a Specific OpenCL Version

By default, the OpenCL API headers in this repository are for the latest
OpenCL version (currently OpenCL 2.2).  To use these API headers to target
a different OpenCL version, an application may `#define` the preprocessor
value `CL_TARGET_OPENCL_VERSION` before including the OpenCL API headers.
The `CL_TARGET_OPENCL_VERSION` is a three digit decimal value representing
the OpenCL API version.

For example, to enforce usage of no more than the OpenCL 1.2 APIs, you may
include the OpenCL API headers as follows:

```cpp
#define CL_TARGET_OPENCL_VERSION 120
#include <CL/opencl.h>
```

## Directory Structure

```cpp
README.md               This file
LICENSE                 Source license for the OpenCL API headers
CL/                     Unified OpenCL API headers tree
```

## License

See [LICENSE](LICENSE).

---

OpenCL and the OpenCL logo are trademarks of Apple Inc. used by permission by Khronos.


epee -  is a small library of helpers, wrappers, tools and and so on, used to make my life easier.


# `src/3rdparty/epee/span.h`

这段代码是一个C ++变量，它定义了一个名为"monero"的类，并指定了该类的版权限制。

该类声明了一个名为"all"的类型，该类型被保留为int类型。该类还声明了一个名为"redistribution"的函数，但没有定义任何函数体。

该代码中使用的标识符包括 " all rights reserved " 和 " The Monero Project"。前者是一个保留的版权声明，告诉任何人可以使用该代码，但必须在代码中包含原始版权 notice。后者的完整文本超出了版权声明的范围，告诉任何人可以使用该代码，但必须包含原始版权 notice 和特定授权条款。

该代码允许将该代码复制、修改或不修改地分配和使用的任何形式，前提是在分配或使用该代码时满足某些条件。条件包括：保留原始版权 notice、包含原始版权 notice 的二进制复制必须包含原始版权 notice 和特定授权条款、使用或修改该代码的人必须知道或应当知道该代码是遵循特定许可规则的。


```cpp
// Copyright (c) 2017-2020, The Monero Project
//
// All rights reserved.
//
// Redistribution and use in source and binary forms, with or without modification, are
// permitted provided that the following conditions are met:
//
// 1. Redistributions of source code must retain the above copyright notice, this list of
//    conditions and the following disclaimer.
//
// 2. Redistributions in binary form must reproduce the above copyright notice, this list
//    of conditions and the following disclaimer in the documentation and/or other
//    materials provided with the distribution.
//
// 3. Neither the name of the copyright holder nor the names of its contributors may be
```

这段代码是一个用于在未经特定书面许可的情况下推荐或促进基于该软件产生的产品的小说。它声明该软件的开发者和服务器提供该软件，但没有提供任何保证，包括 implied warranties of merchantability或 fitness for a particular purpose。它还声明，在任何情况下，开发者或服务器不对直接、间接、特殊、示例或后果性的损失承担责任，即使这些责任是根据合同、侵权或疏忽引起的。最后，它使用了 #pragma once 声明，这意味着该代码仅在调用时才解释一次。


```cpp
//    used to endorse or promote products derived from this software without specific
//    prior written permission.
//
// THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY
// EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF
// MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL
// THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
// SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
// PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
// INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT,
// STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF
// THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

#pragma once

```

This is a C++ implementation of byte span classes that allow you to perform various operations on a byte array, such as reading its contents, modifying it, and returning a subset of it. These operations are provided by two template classes, `to_byte_span` and `as_byte_span`, which take a span and return a byte span.

The `to_byte_span` template class takes a span (a named type that represents a contiguous sequence of bytes) and returns a byte span. It is used to create a byte span from a source of type T.

The `as_byte_span` template class takes a single template parameter T and returns a byte span. It is used to create a byte span from a single value of type T.

The `as_mut_byte_span` template class is similar to `as_byte_span`, but it creates a byte span in read-only mode. It is used to create a byte span from a single value of type T and perform modifications on it.

The `strspan` template class is used to create a byte span from a std::string. It is a simple implementation that is able to handle all common types of sources, including char, unsigned char, and integer types.

Note that the implementation assumes that the input types T and src are not empty and that the span is not empty. It is also assumes that the types are not larger than 8U.


```cpp
#include <algorithm>
#include <cstdint>
#include <memory>
#include <string>
#include <type_traits>

namespace epee
{
  /*!
    \brief Non-owning sequence of data. Does not deep copy

    Inspired by `gsl::span` and/or `boost::iterator_range`. This class is
    intended to be used as a parameter type for functions that need to take a
    writable or read-only sequence of data. Most common cases are `span<char>`
    and `span<std::uint8_t>`. Using as a class member is only recommended if
    clearly documented as not doing a deep-copy. C-arrays are easily convertible
    to this type.

    \note Conversion from C string literal to `span<const char>` will include
      the NULL-terminator.
    \note Never allows derived-to-base pointer conversion; an array of derived
      types is not an array of base types.
   */
  template<typename T>
  class span
  {
    template<typename U>
    static constexpr bool safe_conversion() noexcept
    {
      // Allow exact matches or `T*` -> `const T*`.
      using with_const = typename std::add_const<U>::type;
      return std::is_same<T, U>() ||
        (std::is_const<T>() && std::is_same<T, with_const>());
    }

  public:
    using value_type = T;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using iterator = pointer;
    using const_iterator = const_pointer;

    constexpr span() noexcept : ptr(nullptr), len(0) {}
    constexpr span(std::nullptr_t) noexcept : span() {}

    //! Prevent derived-to-base conversions; invalid in this context.
    template<typename U, typename = typename std::enable_if<safe_conversion<U>()>::type>
    constexpr span(U* const src_ptr, const std::size_t count) noexcept
      : ptr(src_ptr), len(count) {}

    //! Conversion from C-array. Prevents common bugs with sizeof + arrays.
    template<std::size_t N>
    constexpr span(T (&src)[N]) noexcept : span(src, N) {}

    constexpr span(const span&) noexcept = default;
    span& operator=(const span&) noexcept = default;

    /*! Try to remove `amount` elements from beginning of span.
    \return Number of elements removed. */
    std::size_t remove_prefix(std::size_t amount) noexcept
    {
        amount = std::min(len, amount);
        ptr += amount;
        len -= amount;
        return amount;
    }

    constexpr iterator begin() const noexcept { return ptr; }
    constexpr const_iterator cbegin() const noexcept { return ptr; }

    constexpr iterator end() const noexcept { return begin() + size(); }
    constexpr const_iterator cend() const noexcept { return cbegin() + size(); }

    constexpr bool empty() const noexcept { return size() == 0; }
    constexpr pointer data() const noexcept { return ptr; }
    constexpr std::size_t size() const noexcept { return len; }
    constexpr std::size_t size_bytes() const noexcept { return size() * sizeof(value_type); }

    T &operator[](size_t idx) noexcept { return ptr[idx]; }
    const T &operator[](size_t idx) const noexcept { return ptr[idx]; }

  private:
    T* ptr;
    std::size_t len;
  };

  //! \return `span<const T::value_type>` from a STL compatible `src`.
  template<typename T>
  constexpr span<const typename T::value_type> to_span(const T& src)
  {
    // compiler provides diagnostic if size() is not size_t.
    return {src.data(), src.size()};
  }

  //! \return `span<T::value_type>` from a STL compatible `src`.
  template<typename T>
  constexpr span<typename T::value_type> to_mut_span(T& src)
  {
    // compiler provides diagnostic if size() is not size_t.
    return {src.data(), src.size()};
  }

  template<typename T>
  constexpr bool has_padding() noexcept
  {
    return !std::is_standard_layout<T>() || alignof(T) != 1;
  }

  //! \return Cast data from `src` as `span<const std::uint8_t>`.
  template<typename T>
  span<const std::uint8_t> to_byte_span(const span<const T> src) noexcept
  {
    static_assert(!has_padding<T>(), "source type may have padding");
    return {reinterpret_cast<const std::uint8_t*>(src.data()), src.size_bytes()};
  }

  //! \return `span<const std::uint8_t>` which represents the bytes at `&src`.
  template<typename T>
  span<const std::uint8_t> as_byte_span(const T& src) noexcept
  {
    static_assert(!std::is_empty<T>(), "empty types will not work -> sizeof == 1");
    static_assert(!has_padding<T>(), "source type may have padding");
    return {reinterpret_cast<const std::uint8_t*>(std::addressof(src)), sizeof(T)};
  }

  //! \return `span<std::uint8_t>` which represents the bytes at `&src`.
  template<typename T>
  span<std::uint8_t> as_mut_byte_span(T& src) noexcept
  {
    static_assert(!std::is_empty<T>(), "empty types will not work -> sizeof == 1");
    static_assert(!has_padding<T>(), "source type may have padding");
    return {reinterpret_cast<std::uint8_t*>(std::addressof(src)), sizeof(T)};
  }

  //! make a span from a std::string
  template<typename T>
  span<const T> strspan(const std::string &s) noexcept
  {
    static_assert(std::is_same<T, char>() || std::is_same<T, unsigned char>() || std::is_same<T, int8_t>() || std::is_same<T, uint8_t>(), "Unexpected type");
    return {reinterpret_cast<const T*>(s.data()), s.size()};
  }
}

```

# `src/3rdparty/fmt/chrono.h`

这段代码是一个C++格式化库，支持C++11及其后续的语法。它包含了一个支持C++chrono库的包含头文件。

具体来说，这段代码包含以下内容：

1. 一个包含多个头文件的声明，其中包括`<chrono>`，`<ctime>`，`<locale>`和`<sstream>`。

2. 一个`#ifndef`和一个`#define`，分别声明了 FMT_CHRONO_H_ 和 FMT_CHRONO_H_ 这两个预处理指令。

3. 一个`#include`语句，用于引入 FMT_CHRONO_H_ 中定义的chrono库。

4. 一个`#include`语句，用于引入 FMT_CHRONO_H_ 中定义的`<ctime>`库。

5. 一个`#include`语句，用于引入 FMT_CHRONO_H_ 中定义的`<locale>`库。

6. 一个`#include`语句，用于引入 FMT_CHRONO_H_ 中定义的`<sstream>`库。

7. 一个`#include`语句，用于引入 FMT_CHRONO_H_ 中定义的`<chrono>`库。

8. 一个`#include`语句，用于引入 FMT_CHRONO_H_ 中定义的`<iostream>`库。

9. 一个`using namespace std;`声明，用于告诉编译器采用std命名空间。

这段代码的作用是定义了一个FMT_CHRONO_H_的头文件，该头文件包含了所有与chrono相关的头文件，包括`<chrono>`，`<ctime>`，`<locale>`和`<sstream>`。


```cpp
// Formatting library for C++ - chrono support
//
// Copyright (c) 2012 - present, Victor Zverovich
// All rights reserved.
//
// For the license information refer to format.h.

#ifndef FMT_CHRONO_H_
#define FMT_CHRONO_H_

#include <chrono>
#include <ctime>
#include <locale>
#include <sstream>

```

这段代码是一个C++程序，它包含两个头文件：`format.h`和`locale.h`。这两个头文件可能包含一些格式化信息和本地化支持。

然后，它引入了一个名为`FMT_BEGIN_NAMESPACE`的函数。这个函数可能是用来声明一个命名空间，让这个程序在声明变量或函数时可以正确地访问这个命名空间。

接着，它又引入了一个名为`FMT_SAFE_DURATION_CAST`的函数。这个函数可能是用来处理C++14及更高版本中的`std::chrono::duration_cast`函数，通过添加`FMT_SAFE_DURATION_CAST`宏来确保函数的行为是可预测的，不会产生未定义的结果。如果`FMT_SAFE_DURATION_CAST`宏已被定义，那么这个函数的行为可能会受到一些限制，因为有一些C++标准中，`std::chrono::duration_cast`函数是被设计为剩余的，不应该抛出异常。

最后，它还包含一个函数签名，这个函数可能是一个C++标准中的函数，但你需要提供具体的函数名称和参数才能确定它的具体行为。


```cpp
#include "format.h"
#include "locale.h"

FMT_BEGIN_NAMESPACE

// Enable safe chrono durations, unless explicitly disabled.
#ifndef FMT_SAFE_DURATION_CAST
#  define FMT_SAFE_DURATION_CAST 1
#endif
#if FMT_SAFE_DURATION_CAST

// For conversion between std::chrono::durations without undefined
// behaviour or erroneous results.
// This is a stripped down version of duration_cast, for inclusion in fmt.
// See https://github.com/pauldreik/safe_duration_cast
```

这段代码是一个C++模板类，名为`safe_duration_cast`，它实现了将一个`From`类型的数值类型转换为`To`类型（可以是基本数据类型或用户自定义数据类型）的代码。

具体来说，这段代码中定义了一个名为`lossless_integral_conversion`的函数模板，它的第一个参数是一个来自`From`类型的整数，第二个参数是一个整数类型的变量`ec`，表示舍入规则（default value）。

函数模板中包含了一些来自`numeric_limits`头文件的类型说明，以及一些来自`std`头文件的类型说明。其中，`FMT_CONSTEXPR`说明这是一个constant function；`FMT_ENABLE_IF`说明这个函数可以被当做形式化副宣梦（FMT）使用；`std::is_same`和`std::numeric_limits`类型说明用于检查两个`From`和`To`是否相等。

函数模板实现了一个if语句，根据`From`和`To`的位数进行比较，如果它们相等，则说明`From`可以被安全的嵌入到`To`中，否则需要进行进一舍弃。如果`From`是整数，则说明它是一个整数，否则需要使用`std::numeric_limits`中的`integral_duration_cast`函数来进行转换。

如果`from`不是可以被嵌入到`To`中的整数，则需要进行进一舍弃，并将`ec`设置为1，然后返回一个特殊的`To`值，表示转换失败。


```cpp
//
// Copyright Paul Dreik 2019
namespace safe_duration_cast {

template <typename To, typename From,
          FMT_ENABLE_IF(!std::is_same<From, To>::value &&
                        std::numeric_limits<From>::is_signed ==
                            std::numeric_limits<To>::is_signed)>
FMT_CONSTEXPR To lossless_integral_conversion(const From from, int& ec) {
  ec = 0;
  using F = std::numeric_limits<From>;
  using T = std::numeric_limits<To>;
  static_assert(F::is_integer, "From must be integral");
  static_assert(T::is_integer, "To must be integral");

  // A and B are both signed, or both unsigned.
  if (F::digits <= T::digits) {
    // From fits in To without any problem.
  } else {
    // From does not always fit in To, resort to a dynamic check.
    if (from < (T::min)() || from > (T::max)()) {
      // outside range.
      ec = 1;
      return {};
    }
  }
  return static_cast<To>(from);
}

```

这段代码定义了一个名为 `lossless_integral_conversion` 的模板函数，用于将一个来自 `From` 类型的数值 `from` 转换为另一个来自 `To` 类型的数值 `To` 类型的值，且不损失精度。

函数的实现包括两个步骤：

1. 检查 `from` 是否为整数类型，如果是，则将其转换为整数类型，否则允许函数失败。
2. 检查 `from` 是否为负数，如果是，则将其转换为 `ec` 等于 1 的值。否则，函数继续执行。

如果 `from` 是正数，并且 `To` 也是正数，则需要将 `from` 转换为 `To` 类型的值。为此，函数使用 `std::numeric_limits` 类型，该类型提供了对两个不同类型的数值范围的限制。函数需要确保 `from` 是一个整数，而 `To` 是一个整数。

函数的 `FMT_CONSTEXPR` 修饰符表示该函数是一个常量表达式，这意味着该函数的值不能被修改。


```cpp
/**
 * converts From to To, without loss. If the dynamic value of from
 * can't be converted to To without loss, ec is set.
 */
template <typename To, typename From,
          FMT_ENABLE_IF(!std::is_same<From, To>::value &&
                        std::numeric_limits<From>::is_signed !=
                            std::numeric_limits<To>::is_signed)>
FMT_CONSTEXPR To lossless_integral_conversion(const From from, int& ec) {
  ec = 0;
  using F = std::numeric_limits<From>;
  using T = std::numeric_limits<To>;
  static_assert(F::is_integer, "From must be integral");
  static_assert(T::is_integer, "To must be integral");

  if (detail::const_check(F::is_signed && !T::is_signed)) {
    // From may be negative, not allowed!
    if (fmt::detail::is_negative(from)) {
      ec = 1;
      return {};
    }
    // From is positive. Can it always fit in To?
    if (F::digits > T::digits &&
        from > static_cast<From>(detail::max_value<To>())) {
      ec = 1;
      return {};
    }
  }

  if (!F::is_signed && T::is_signed && F::digits >= T::digits &&
      from > static_cast<From>(detail::max_value<To>())) {
    ec = 1;
    return {};
  }
  return static_cast<To>(from);  // Lossless conversion.
}

```

这段代码定义了一个名为`lossless_integral_conversion`的函数，其模板参数为两个类型：`From`和`To`，使用了模板元编程技术。

该函数的作用是实现将一个`From`类型的数值转换为另一个`To`类型的数值，如果转换成功，则返回原始的`From`类型，否则将`ec`设置为转换失败时设置的值。

具体实现中，函数在模板定义中使用了`FMT_ENABLE_IF`语法，用于开启`std::is_same<From, To>`这个条件判断的代码块。在函数体中，首先将`ec`变量初始化为0，然后执行以下操作：

1. 如果`From`和`To`类型相同，则直接返回`from`。
2. 如果`From`和`To`类型不同，则尝试将`From`类型的值转换为`To`类型，并将结果存储在`ec`中。

最后，函数返回`ec`的值，如果转换成功，则返回原始的`From`类型。


```cpp
template <typename To, typename From,
          FMT_ENABLE_IF(std::is_same<From, To>::value)>
FMT_CONSTEXPR To lossless_integral_conversion(const From from, int& ec) {
  ec = 0;
  return from;
}  // function

// clang-format off
/**
 * converts From to To if possible, otherwise ec is set.
 *
 * input                            |    output
 * ---------------------------------|---------------
 * NaN                              | NaN
 * Inf                              | Inf
 * normal, fits in output           | converted (possibly lossy)
 * normal, does not fit in output   | ec is set
 * subnormal                        | best effort
 * -Inf                             | -Inf
 */
```

这段代码定义了一个名为`safe_float_conversion`的函数，它接受两个模板参数：来自模板类型的`From`和目标模板类型的`To`。

该函数使用`FMT_CONSTEXPR`对函数进行类型约束，确保输入参数`from`和`ec`都是整数。然后通过一些条件分支语句来检查输入参数`from`的类型是否正确，如果`from`是浮点数，则执行以下操作：

1. 如果`from`大于或等于`T.lowest()`且小于或等于`T.max()`之一，则返回`from`，否则创建一个需要类型`To`的整数并返回。
2. 如果是`from`，则不需要做任何处理，返回`static_cast<To>(from)`。
3. 如果`from`是无穷大或无穷小，则将其保留，并返回。

该函数的作用是协助用户进行安全的有符号或无符号浮点数转换，确保输入参数`from`是正确的类型，并在输入为浮点数时返回正确的结果，同时在输入为无穷大或无穷小时，返回需要类型`To`的整数。


```cpp
// clang-format on
template <typename To, typename From,
          FMT_ENABLE_IF(!std::is_same<From, To>::value)>
FMT_CONSTEXPR To safe_float_conversion(const From from, int& ec) {
  ec = 0;
  using T = std::numeric_limits<To>;
  static_assert(std::is_floating_point<From>::value, "From must be floating");
  static_assert(std::is_floating_point<To>::value, "To must be floating");

  // catch the only happy case
  if (std::isfinite(from)) {
    if (from >= T::lowest() && from <= (T::max)()) {
      return static_cast<To>(from);
    }
    // not within range.
    ec = 1;
    return {};
  }

  // nan and inf will be preserved
  return static_cast<To>(from);
}  // function

```

This is a function that converts a count of type From (e.g. a duration) to a count of type To (e.g. a rapport) in a given interval of time, represented by the duration type FromRep and the period type FromPeriod. The conversion is done by multiplying the count with a factor that is a ratio-divide between the number of periods of the From type and the denominator of the To type. The factor is defined as a struct that has a ratio of the form num/den, where num is the number of periods of the From type and den is the denominator of the To type. The function is able to handle this factor by converting the count to an intermediate type that can be converted to the desired type. If the intermediate representation cannot be found, the function will return the original count. The function is able to handle integer-valued input and the output is of the same type as the input.

The code also includes a check for the divisor of the input, to avoid overflow or underflow.


```cpp
template <typename To, typename From,
          FMT_ENABLE_IF(std::is_same<From, To>::value)>
FMT_CONSTEXPR To safe_float_conversion(const From from, int& ec) {
  ec = 0;
  static_assert(std::is_floating_point<From>::value, "From must be floating");
  return from;
}

/**
 * safe duration cast between integral durations
 */
template <typename To, typename FromRep, typename FromPeriod,
          FMT_ENABLE_IF(std::is_integral<FromRep>::value),
          FMT_ENABLE_IF(std::is_integral<typename To::rep>::value)>
To safe_duration_cast(std::chrono::duration<FromRep, FromPeriod> from,
                      int& ec) {
  using From = std::chrono::duration<FromRep, FromPeriod>;
  ec = 0;
  // the basic idea is that we need to convert from count() in the from type
  // to count() in the To type, by multiplying it with this:
  struct Factor
      : std::ratio_divide<typename From::period, typename To::period> {};

  static_assert(Factor::num > 0, "num must be positive");
  static_assert(Factor::den > 0, "den must be positive");

  // the conversion is like this: multiply from.count() with Factor::num
  // /Factor::den and convert it to To::rep, all this without
  // overflow/underflow. let's start by finding a suitable type that can hold
  // both To, From and Factor::num
  using IntermediateRep =
      typename std::common_type<typename From::rep, typename To::rep,
                                decltype(Factor::num)>::type;

  // safe conversion to IntermediateRep
  IntermediateRep count =
      lossless_integral_conversion<IntermediateRep>(from.count(), ec);
  if (ec) return {};
  // multiply with Factor::num without overflow or underflow
  if (detail::const_check(Factor::num != 1)) {
    const auto max1 = detail::max_value<IntermediateRep>() / Factor::num;
    if (count > max1) {
      ec = 1;
      return {};
    }
    const auto min1 =
        (std::numeric_limits<IntermediateRep>::min)() / Factor::num;
    if (count < min1) {
      ec = 1;
      return {};
    }
    count *= Factor::num;
  }

  if (detail::const_check(Factor::den != 1)) count /= Factor::den;
  auto tocount = lossless_integral_conversion<typename To::rep>(count, ec);
  return ec ? To() : To(tocount);
}

```

This is a function that converts a Factor::num value to the specified To::rep type, maintaining that the conversion is safe and does not overflow or underflow. The function uses intermediate representations of the types From::rep and To::rep to hold the intermediate values during the conversion process.

The function first determines the type IntermediateRep, which can hold the To, From, and Factor::num values. Then, it forces the conversion of From::rep to IntermediateRep without any overflow or underflow by宽限期检测. If the conversion succeeds, the function returns an empty value. If not, the function continues by multiplying the count by Factor::num to avoid overflow or underflow.

Finally, the function converts the IntermediateRep back to the To::rep type, ensuring that the value is safe to use. If the conversion fails or an overflow/underflow occurs, the function returns an empty value.


```cpp
/**
 * safe duration_cast between floating point durations
 */
template <typename To, typename FromRep, typename FromPeriod,
          FMT_ENABLE_IF(std::is_floating_point<FromRep>::value),
          FMT_ENABLE_IF(std::is_floating_point<typename To::rep>::value)>
To safe_duration_cast(std::chrono::duration<FromRep, FromPeriod> from,
                      int& ec) {
  using From = std::chrono::duration<FromRep, FromPeriod>;
  ec = 0;
  if (std::isnan(from.count())) {
    // nan in, gives nan out. easy.
    return To{std::numeric_limits<typename To::rep>::quiet_NaN()};
  }
  // maybe we should also check if from is denormal, and decide what to do about
  // it.

  // +-inf should be preserved.
  if (std::isinf(from.count())) {
    return To{from.count()};
  }

  // the basic idea is that we need to convert from count() in the from type
  // to count() in the To type, by multiplying it with this:
  struct Factor
      : std::ratio_divide<typename From::period, typename To::period> {};

  static_assert(Factor::num > 0, "num must be positive");
  static_assert(Factor::den > 0, "den must be positive");

  // the conversion is like this: multiply from.count() with Factor::num
  // /Factor::den and convert it to To::rep, all this without
  // overflow/underflow. let's start by finding a suitable type that can hold
  // both To, From and Factor::num
  using IntermediateRep =
      typename std::common_type<typename From::rep, typename To::rep,
                                decltype(Factor::num)>::type;

  // force conversion of From::rep -> IntermediateRep to be safe,
  // even if it will never happen be narrowing in this context.
  IntermediateRep count =
      safe_float_conversion<IntermediateRep>(from.count(), ec);
  if (ec) {
    return {};
  }

  // multiply with Factor::num without overflow or underflow
  if (Factor::num != 1) {
    constexpr auto max1 = detail::max_value<IntermediateRep>() /
                          static_cast<IntermediateRep>(Factor::num);
    if (count > max1) {
      ec = 1;
      return {};
    }
    constexpr auto min1 = std::numeric_limits<IntermediateRep>::lowest() /
                          static_cast<IntermediateRep>(Factor::num);
    if (count < min1) {
      ec = 1;
      return {};
    }
    count *= static_cast<IntermediateRep>(Factor::num);
  }

  // this can't go wrong, right? den>0 is checked earlier.
  if (Factor::den != 1) {
    using common_t = typename std::common_type<IntermediateRep, intmax_t>::type;
    count /= static_cast<common_t>(Factor::den);
  }

  // convert to the to type, safely
  using ToRep = typename To::rep;

  const ToRep tocount = safe_float_conversion<ToRep>(count, ec);
  if (ec) {
    return {};
  }
  return To{tocount};
}
}  // namespace safe_duration_cast
```

这段代码是一个C++预处理指令，用于在编译时检查函数式宏是否使用了指定的本地化函数。

该代码定义了一个名为FMT_NOMACRO的函数式宏，其含义是不要在函数式中使用前面定义的函数式参数。该宏会在编译时将该宏替换为空值，避免在编译时产生错误。

为了实现这个宏，该代码定义了三个函数式枚举类型localtime_r、localtime_s和gmtime_r，它们用于获取当前本地时间。这三个函数式枚举类型在函数式中声明，但在内部使用了null<>类型，表示它们的值是无效的，防止在函数式中使用它们。

该代码还定义了一个名为dispatcher的结构体，用于处理函数式宏。该结构体包含一个time_t类型的成员time和一个tm_type类型的成员tm，分别用于存储当前时间和当前本地时间。

localtime函数是一个内部函数，用于将当前本地时间转换为tm_type类型，这个函数使用了两个已知函数式枚举类型localtime_r和localtime_s，通过两次调用localtime_r函数来获取当前本地时间。该函数使用了一个dispatcher对象，该dispatcher对象包含time_t类型的成员time和一个tm_type类型的成员tm，用于存储当前时间和当前本地时间。该dispatcher对象的handle函数实现了对localtime_r函数的引用，以获取当前本地时间。如果当前本地时间为null<>，则调用fallback函数，该函数会尝试使用localtime_s函数获取当前本地时间，该函数会尝试获取当前本地时间并返回0，如果当前本地时间为null<>，则返回0。

该代码还定义了一个名为tm的null<>类型，用于存储当前本地时间。tm类型是tm_type类型的别名，用于存储当前本地时间的tm_type成员。

该代码通过使用函数式枚举类型和函数式函数来实现对函数式宏的检查和替换，从而可以防止在函数式中使用未定义的函数式参数。


```cpp
#endif

// Prevents expansion of a preceding token as a function-style macro.
// Usage: f FMT_NOMACRO()
#define FMT_NOMACRO

namespace detail {
inline null<> localtime_r FMT_NOMACRO(...) { return null<>(); }
inline null<> localtime_s(...) { return null<>(); }
inline null<> gmtime_r(...) { return null<>(); }
inline null<> gmtime_s(...) { return null<>(); }
}  // namespace detail

// Thread-safe replacement for std::localtime
inline std::tm localtime(std::time_t time) {
  struct dispatcher {
    std::time_t time_;
    std::tm tm_;

    dispatcher(std::time_t t) : time_(t) {}

    bool run() {
      using namespace fmt::detail;
      return handle(localtime_r(&time_, &tm_));
    }

    bool handle(std::tm* tm) { return tm != nullptr; }

    bool handle(detail::null<>) {
      using namespace fmt::detail;
      return fallback(localtime_s(&tm_, &time_));
    }

    bool fallback(int res) { return res == 0; }

```

这段代码是一个 C++ 程序，它包含一个条件分支语句和一个函数 fallback。这个条件分支语句检查 MFC 是否支持时间戳(TM)类型，如果不支持，就调用 fallback 函数。

具体来说，这段代码的作用是检查本地时间是否为 MFC 所支持的时间戳类型。如果本地时间不是 MFC 支持的时间戳类型，就调用 fallback 函数来获取本地时间。如果本地时间已经是 MFC 支持的时间戳类型，就直接返回本地时间。

此外，这段代码还定义了一个 dispatcher 类型的 lt 变量，它是一个代表 local time(本地时间)的变量。如果 lt.run() 函数返回 false，即无法获取本地时间，就会抛出 format_error 异常。

最终，这段代码返回本地时间(如果是 MFC 支持的时间戳类型)，否则抛出 format_error 异常。


```cpp
#if !FMT_MSC_VER
    bool fallback(detail::null<>) {
      using namespace fmt::detail;
      std::tm* tm = std::localtime(&time_);
      if (tm) tm_ = *tm;
      return tm != nullptr;
    }
#endif
  };
  dispatcher lt(time);
  // Too big time values may be unsupported.
  if (!lt.run()) FMT_THROW(format_error("time_t value out of range"));
  return lt.tm_;
}

```

这段代码是一个C++代码，它是一个时区处理函数，用于将给定的时间点（time_t类型）转换为本地时间（tm类型）。

tm类型是C++标准库中的一个结构体，它包含一个time_t类型的成员time，一个tm类型成员tm，以及一个布尔类型的成员run。

time_t类型是一个标准时间类型，它可以存储任何有效时间，而tm类型是tm的别名，它是一个结构体，包含time_t类型的成员time和tm类型成员tm。

函数的参数是一个time_t类型的局部时间点，它包含一个系统时间的时间戳。函数返回一个指向tm类型的本地时间的指针。

函数内部使用了一个名为dispatcher的辅助结构体，它是时间安全的，可以在多个线程之间安全地共享。dispatcher结构体包含一个time_t类型的成员time和一个tm类型成员tm，以及一个布尔类型的成员run。run成员函数是一个布尔类型，表示是否成功调用辅助函数，如果成功，则返回true，否则返回false。

函数内部还使用了一个名为handle的函数，它是dispatcher结构体的成员函数，用于将给定的time_t类型的局部时间点（tm类型）转换为本地时间（tm类型）。handle函数包含两个函数，一个使用gmtime_r函数，另一个使用fallback函数。

gmtime_r函数是一个标准库函数，它返回一个tm类型的对象，它将给定的time_t类型的局部时间点（tm类型）转换为本地时间（tm类型）。如果当前系统时间（tm类型）为null，则使用fallback函数，它返回一个null类型。

fallback函数是一个辅助函数，它是一个 fallback版本的handle函数，用于将当前系统时间（tm类型）为null的局部时间点（tm类型）转换为tm类型。这个版本的handle函数如果当前系统时间（tm类型）为null，则直接返回false。


```cpp
inline std::tm localtime(
    std::chrono::time_point<std::chrono::system_clock> time_point) {
  return localtime(std::chrono::system_clock::to_time_t(time_point));
}

// Thread-safe replacement for std::gmtime
inline std::tm gmtime(std::time_t time) {
  struct dispatcher {
    std::time_t time_;
    std::tm tm_;

    dispatcher(std::time_t t) : time_(t) {}

    bool run() {
      using namespace fmt::detail;
      return handle(gmtime_r(&time_, &tm_));
    }

    bool handle(std::tm* tm) { return tm != nullptr; }

    bool handle(detail::null<>) {
      using namespace fmt::detail;
      return fallback(gmtime_s(&tm_, &time_));
    }

    bool fallback(int res) { return res == 0; }

```

这段代码是一个 C++ 函数，名为 `gmtime`，它用于将给定的 `time_point` 转换为 `std::tm` 类型的时间类型。

首先，该代码检查是否支持 `FMT_MSC_VER` 编译器，如果不支持，则执行 `fallback` 函数。`fallback` 函数是一个真 `bool` 类型，它检查给定的 `null<>` 是否为 `null`，如果是，则返回 `true`，否则执行具体的操作并返回 `true`。

接着，该代码创建一个名为 `gt` 的 `dispatcher` 类型的对象，并调用 `gt.run()` 方法，如果执行失败，则抛出 `FMT_THROW` 异常，并返回 `null`。

最后，该代码使用 `gmtime` 函数将给定的 `time_point` 类型转换为 `std::tm` 类型的时间类型，该函数将返回 `std::tm` 类型的对象。


```cpp
#if !FMT_MSC_VER
    bool fallback(detail::null<>) {
      std::tm* tm = std::gmtime(&time_);
      if (tm) tm_ = *tm;
      return tm != nullptr;
    }
#endif
  };
  dispatcher gt(time);
  // Too big time values may be unsupported.
  if (!gt.run()) FMT_THROW(format_error("time_t value out of range"));
  return gt.tm_;
}

inline std::tm gmtime(
    std::chrono::time_point<std::chrono::system_clock> time_point) {
  return gmtime(std::chrono::system_clock::to_time_t(time_point));
}

```

这段代码定义了一个namespace detail，其中包含两个inline函数strftime和strftime，它们都接受三个参数：

1. 一个char类型的字符指针str，表示要格式化的字符串；
2. 一个size_t类型的整数count，表示要获取的字符数量；
3. 一个const char*类型的格式字符串，表示要应用的格式；
4. 一个const std::tm*类型的时间tm，表示要格式化的时间；

strftime函数采用三个模板参数：

1. 一个char类型的字符指针str，表示要格式化的字符串；
2. 一个size_t类型的整数count，表示要获取的字符数量；
3. 一个const wchar_t*类型的格式字符串，表示要应用的格式；
4. 一个const std::tm*类型的时间tm，表示要格式化的时间；

这两个函数的实现基本相同，都是一个tm类型的时间加上一个格式字符串，通过strftime函数输出的字符串就是按照格式化字符串计算出来的字符。

而formatter模板类则是一个模板，其中包含一个formatter<std::chrono::time_point<std::chrono::system_clock>, Char>，它重载了strftime和strftime函数，并且在其中的模板实参中，去掉了两个const类型的参数，也就是去掉了tm和wchar_t类型的参数，这样就避免了编译器在编译时产生一些可读性问题和未定义行为。

formatter模板类中包含了一个模板函数，用来计算给定的时间点按照给定的格式进行格式化输出，这个函数同时也包含了对localtime函数的调用，用来将time点对应的时间格式化成std::chrono::system_clock时间类型。


```cpp
namespace detail {
inline size_t strftime(char* str, size_t count, const char* format,
                       const std::tm* time) {
  return std::strftime(str, count, format, time);
}

inline size_t strftime(wchar_t* str, size_t count, const wchar_t* format,
                       const std::tm* time) {
  return std::wcsftime(str, count, format, time);
}
}  // namespace detail

template <typename Char>
struct formatter<std::chrono::time_point<std::chrono::system_clock>, Char>
    : formatter<std::tm, Char> {
  template <typename FormatContext>
  auto format(std::chrono::time_point<std::chrono::system_clock> val,
              FormatContext& ctx) -> decltype(ctx.out()) {
    std::tm time = localtime(val);
    return formatter<std::tm, Char>::format(time, ctx);
  }
};

```

这段代码定义了一个模板结构体 `formatter`，用于格式化 `std::tm` 中的日期时间字符串。

具体来说，`formatter` 模板结构体包括两个模板函数：

1. `parse`，用于解析 `std::tm` 中的日期时间字符串，返回解析后的 `true` 类型。

2. `format`，用于将 `std::tm` 中的日期时间字符串格式化为可输出格式，并输出。

这两个模板函数分别对应 `<tm, Char>` 和 `<FormatContext, Char>` 模板参数。

具体实现中，`parse` 函数从 `std::tm` 的开始位置开始遍历，直到遇到 `':'` 字符，然后将 `it` 指针移动到 `end` 指针处。在遍历过程中，如果 `it` 不等于 `ctx.end()`，则说明解析字符串可以成功，返回解析后的 `true` 类型。

`format` 函数从 `std::tm` 的开始位置开始遍历，直到遇到空字符串或者 `}'` 字符。如果当前遍历位置还有空字符串可以分配字符串，则分配字符串并将 `tm_format` 数组长度增加当前遍历位置的字符数。当 `tm_format` 数组长度达到 `std::fmt` 中的最小增长率 `MIN_GROWTH` 时，将 `buf` 数组长度设置为 `buf.capacity()` 并截断 `tm_format` 数组长度为 `std::fmt` 中的最小增长率 `MIN_GROWTH`。然后将 `buf` 数组长度设置为 `it` 指针移动后的字符数加上 `MIN_GROWTH`。最后，将 `buf` 数组长度为 `std::fmt` 中的输出字符串长度，并从 `it` 指针处开始输出字符串。

最后，由于 `std::fmt` 的输出字符串长度是固定的，因此可以通过遍历 `tm_format` 数组长度并检查是否已输出所有字符串来判断解析是否成功。


```cpp
template <typename Char> struct formatter<std::tm, Char> {
  template <typename ParseContext>
  auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    auto it = ctx.begin();
    if (it != ctx.end() && *it == ':') ++it;
    auto end = it;
    while (end != ctx.end() && *end != '}') ++end;
    tm_format.reserve(detail::to_unsigned(end - it + 1));
    tm_format.append(it, end);
    tm_format.push_back('\0');
    return end;
  }

  template <typename FormatContext>
  auto format(const std::tm& tm, FormatContext& ctx) -> decltype(ctx.out()) {
    basic_memory_buffer<Char> buf;
    size_t start = buf.size();
    for (;;) {
      size_t size = buf.capacity() - start;
      size_t count = detail::strftime(&buf[start], size, &tm_format[0], &tm);
      if (count != 0) {
        buf.resize(start + count);
        break;
      }
      if (size >= tm_format.size() * 256) {
        // If the buffer is 256 times larger than the format string, assume
        // that `strftime` gives an empty result. There doesn't seem to be a
        // better way to distinguish the two cases:
        // https://github.com/fmtlib/fmt/issues/367
        break;
      }
      const size_t MIN_GROWTH = 10;
      buf.reserve(buf.capacity() + (size > MIN_GROWTH ? size : MIN_GROWTH));
    }
    return std::copy(buf.begin(), buf.end(), ctx.out());
  }

  basic_memory_buffer<Char> tm_format;
};

```

这段代码定义了一系列模板类，名为“detail”，这些模板类可以用来计算各种物理量的单位。

每个模板类都继承自“FMT_CONSTEXPR”类型的构造函数，该构造函数提供了一个模板参数“Period”，它代表一个介于两个整数之间的浮点数。这个模板参数告诉编译器如何计算该物理量的单位。

每个模板类中包含一个名为“get_units”的函数，它是一个内部成员函数，用于计算给定物理量的单位。如果模板参数为“std::option<Period>”类型，那么这个函数的实现就是对“Period”类型的范围进行枚举，并返回相应的单位名称。否则，函数的行为是不明确的，编译器无法识别可能存在的语法错误。

该代码的作用是定义了一系列可以用来计算各种物理量单位的模板类，以及一个用于获取这些物理量单位的函数。这些模板类和函数可以被用来定义各种类型的物理量，如时间、电压、电阻等等，而函数“get_units”则可以用来获取相应的单位名称。


```cpp
namespace detail {
template <typename Period> FMT_CONSTEXPR const char* get_units() {
  return nullptr;
}
template <> FMT_CONSTEXPR const char* get_units<std::atto>() { return "as"; }
template <> FMT_CONSTEXPR const char* get_units<std::femto>() { return "fs"; }
template <> FMT_CONSTEXPR const char* get_units<std::pico>() { return "ps"; }
template <> FMT_CONSTEXPR const char* get_units<std::nano>() { return "ns"; }
template <> FMT_CONSTEXPR const char* get_units<std::micro>() { return "µs"; }
template <> FMT_CONSTEXPR const char* get_units<std::milli>() { return "ms"; }
template <> FMT_CONSTEXPR const char* get_units<std::centi>() { return "cs"; }
template <> FMT_CONSTEXPR const char* get_units<std::deci>() { return "ds"; }
template <> FMT_CONSTEXPR const char* get_units<std::ratio<1>>() { return "s"; }
template <> FMT_CONSTEXPR const char* get_units<std::deca>() { return "das"; }
template <> FMT_CONSTEXPR const char* get_units<std::hecto>() { return "hs"; }
```

这段代码定义了一个命名约定，用于在编译时根据输入的数据类型来选择正确的单位字符串。这个约定是 C++ 标准库中的一个功能，允许开发者自定义单位前缀和后缀。

具体来说，这个代码定义了一个枚举类型 numeric_system，它包含四个枚举常量：standard、alternative。standard 表示使用标准的单位前缀和后缀，alternative 表示使用 alternative 中的单位前缀和后缀。

另外，这个代码还定义了几个函数，用于获取不同数据类型的单位字符串。这些函数使用了 FMT(format-macros)库中的宏定义，FMT 是一个 C++ 格式化辅助库。

这些函数的作用是，根据输入的数据类型，选择正确的单位字符串，以便在输出时使用正确的单位前缀和后缀。这些函数可以被用来格式化输出，例如打印 "10M"、"2.5G" 等。


```cpp
template <> FMT_CONSTEXPR const char* get_units<std::kilo>() { return "ks"; }
template <> FMT_CONSTEXPR const char* get_units<std::mega>() { return "Ms"; }
template <> FMT_CONSTEXPR const char* get_units<std::giga>() { return "Gs"; }
template <> FMT_CONSTEXPR const char* get_units<std::tera>() { return "Ts"; }
template <> FMT_CONSTEXPR const char* get_units<std::peta>() { return "Ps"; }
template <> FMT_CONSTEXPR const char* get_units<std::exa>() { return "Es"; }
template <> FMT_CONSTEXPR const char* get_units<std::ratio<60>>() {
  return "m";
}
template <> FMT_CONSTEXPR const char* get_units<std::ratio<3600>>() {
  return "h";
}

enum class numeric_system {
  standard,
  // Alternative numeric system, e.g. 十二 instead of 12 in ja_JP locale.
  alternative
};

```

This is a C++ implementation of a `format_date` function. It takes a `handler` object and a `format` string as arguments. The `handler` object is used to handle the result of the formatting operation.

The function `format_date` has a case statement that covers the different representations of a date string.

If the first character of the format string is 'D', it is interpreted as a date-time format, and the function will call the `handler.on_datetime` method.

If the first character of the format string is 'E', it is interpreted as an alternatively formatted date string, and the function will call the `handler.on_loc_date` method.

If the first character of the format string is 'O', it is interpreted as an alternatively formatted date time string, and the function will call the `handler.on_loc_time` method.

If the first character of the format string is 'H', it is interpreted as a 24-hour format, and the function will call the `handler.on_24_hour` method.

If the first character of the format string is 'I', it is interpreted as a 12-hour format, and the function will call the `handler.on_12_hour` method.

If the first character of the format string is 'M', it is interpreted as a minute format, and the function will call the `handler.on_minute` method.

If the first character of the format string is 'S', it is interpreted as a second format, and the function will call the `handler.on_second` method.

If the first character of the format string is not a valid format, the function will throw an exception.

The function also has a `default` case that handles any other invalid format.

The function has a switch statement that handles the different date formats.

The `handler` object is passed by the function to the `on_text` method, which will format the date string according to the selected format.


```cpp
// Parses a put_time-like format string and invokes handler actions.
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_chrono_format(const Char* begin,
                                              const Char* end,
                                              Handler&& handler) {
  auto ptr = begin;
  while (ptr != end) {
    auto c = *ptr;
    if (c == '}') break;
    if (c != '%') {
      ++ptr;
      continue;
    }
    if (begin != ptr) handler.on_text(begin, ptr);
    ++ptr;  // consume '%'
    if (ptr == end) FMT_THROW(format_error("invalid format"));
    c = *ptr++;
    switch (c) {
    case '%':
      handler.on_text(ptr - 1, ptr);
      break;
    case 'n': {
      const Char newline[] = {'\n'};
      handler.on_text(newline, newline + 1);
      break;
    }
    case 't': {
      const Char tab[] = {'\t'};
      handler.on_text(tab, tab + 1);
      break;
    }
    // Day of the week:
    case 'a':
      handler.on_abbr_weekday();
      break;
    case 'A':
      handler.on_full_weekday();
      break;
    case 'w':
      handler.on_dec0_weekday(numeric_system::standard);
      break;
    case 'u':
      handler.on_dec1_weekday(numeric_system::standard);
      break;
    // Month:
    case 'b':
      handler.on_abbr_month();
      break;
    case 'B':
      handler.on_full_month();
      break;
    // Hour, minute, second:
    case 'H':
      handler.on_24_hour(numeric_system::standard);
      break;
    case 'I':
      handler.on_12_hour(numeric_system::standard);
      break;
    case 'M':
      handler.on_minute(numeric_system::standard);
      break;
    case 'S':
      handler.on_second(numeric_system::standard);
      break;
    // Other:
    case 'c':
      handler.on_datetime(numeric_system::standard);
      break;
    case 'x':
      handler.on_loc_date(numeric_system::standard);
      break;
    case 'X':
      handler.on_loc_time(numeric_system::standard);
      break;
    case 'D':
      handler.on_us_date();
      break;
    case 'F':
      handler.on_iso_date();
      break;
    case 'r':
      handler.on_12_hour_time();
      break;
    case 'R':
      handler.on_24_hour_time();
      break;
    case 'T':
      handler.on_iso_time();
      break;
    case 'p':
      handler.on_am_pm();
      break;
    case 'Q':
      handler.on_duration_value();
      break;
    case 'q':
      handler.on_duration_unit();
      break;
    case 'z':
      handler.on_utc_offset();
      break;
    case 'Z':
      handler.on_tz_name();
      break;
    // Alternative representation:
    case 'E': {
      if (ptr == end) FMT_THROW(format_error("invalid format"));
      c = *ptr++;
      switch (c) {
      case 'c':
        handler.on_datetime(numeric_system::alternative);
        break;
      case 'x':
        handler.on_loc_date(numeric_system::alternative);
        break;
      case 'X':
        handler.on_loc_time(numeric_system::alternative);
        break;
      default:
        FMT_THROW(format_error("invalid format"));
      }
      break;
    }
    case 'O':
      if (ptr == end) FMT_THROW(format_error("invalid format"));
      c = *ptr++;
      switch (c) {
      case 'w':
        handler.on_dec0_weekday(numeric_system::alternative);
        break;
      case 'u':
        handler.on_dec1_weekday(numeric_system::alternative);
        break;
      case 'H':
        handler.on_24_hour(numeric_system::alternative);
        break;
      case 'I':
        handler.on_12_hour(numeric_system::alternative);
        break;
      case 'M':
        handler.on_minute(numeric_system::alternative);
        break;
      case 'S':
        handler.on_second(numeric_system::alternative);
        break;
      default:
        FMT_THROW(format_error("invalid format"));
      }
      break;
    default:
      FMT_THROW(format_error("invalid format"));
    }
    begin = ptr;
  }
  if (begin != ptr) handler.on_text(begin, ptr);
  return ptr;
}

```

This is a C++ class template that defines a single member function `on_text()` which takes two parameters of type `Char` and returns void.

The `on_text()` function is not used in this class template, but it is a default implementation that reports any date/time issues to the assertion layer.

The class template also defines several other member functions of the same name with different parameter lists, each of which report any date/time issues to the assertion layer. These include `on_abbr_weekday()`, `on_full_weekday()`, `on_dec0_weekday()`, `on_dec1_weekday()`, `on_abbr_month()`, `on_full_month()`, `on_24_hour()`, `on_12_hour()`, `on_minute()`, `on_second()`, `on_datetime()`, `on_loc_date()`, `on_loc_time()`, `on_us_date()`, `on_iso_date()`, `on_12_hour_time()`, `on_24_hour_time()`, `on_iso_time()`, and `on_am_pm()`.

It is worth noting that this class template does not have any member variables, and it is recommended to add some sort of data to the template to avoid any unnecessary problems.


```cpp
struct chrono_format_checker {
  FMT_NORETURN void report_no_date() { FMT_THROW(format_error("no date")); }

  template <typename Char> void on_text(const Char*, const Char*) {}
  FMT_NORETURN void on_abbr_weekday() { report_no_date(); }
  FMT_NORETURN void on_full_weekday() { report_no_date(); }
  FMT_NORETURN void on_dec0_weekday(numeric_system) { report_no_date(); }
  FMT_NORETURN void on_dec1_weekday(numeric_system) { report_no_date(); }
  FMT_NORETURN void on_abbr_month() { report_no_date(); }
  FMT_NORETURN void on_full_month() { report_no_date(); }
  void on_24_hour(numeric_system) {}
  void on_12_hour(numeric_system) {}
  void on_minute(numeric_system) {}
  void on_second(numeric_system) {}
  FMT_NORETURN void on_datetime(numeric_system) { report_no_date(); }
  FMT_NORETURN void on_loc_date(numeric_system) { report_no_date(); }
  FMT_NORETURN void on_loc_time(numeric_system) { report_no_date(); }
  FMT_NORETURN void on_us_date() { report_no_date(); }
  FMT_NORETURN void on_iso_date() { report_no_date(); }
  void on_12_hour_time() {}
  void on_24_hour_time() {}
  void on_iso_time() {}
  void on_am_pm() {}
  void on_duration_value() {}
  void on_duration_unit() {}
  FMT_NORETURN void on_utc_offset() { report_no_date(); }
  FMT_NORETURN void on_tz_name() { report_no_date(); }
};

```

这段代码定义了两个模板别名，分别为 `isnan` 和 `isfinite`。这两个模板别名分别用于判断两个不同类型的数值是否为 `NaN` 和 `Infinity`，也就是 `NaN` 和 `Infinity` 分别是 `双精度浮点数` 和 `整数` 类型的非空值。

具体来说，`isnan` 判断参数 `T` 是否为 `NaN`，如果 `T` 为 `NaN`，则返回 `false`，否则返回 `true`。同理，`isfinite` 判断参数 `T` 是否为 `Infinity`，如果 `T` 为 `Infinity`，则返回 `true`，否则返回 `false`。

这里使用了 C++17 中的 `FMT_ENABLE_IF` 语法，用于在编译时判断模板参数是否符合某种类型要求。如果符合，则允许编译，否则报告错误。


```cpp
template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
inline bool isnan(T) {
  return false;
}
template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
inline bool isnan(T value) {
  return std::isnan(value);
}

template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
inline bool isfinite(T) {
  return true;
}
template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
inline bool isfinite(T value) {
  return std::isfinite(value);
}

```

这段代码定义了两个模板函数，`to_nonnegative_int` 和 `to_nonnegative_int_no_compiletime_dependency`，它们的作用是将一个 `T` 类型的值转换为非负整数，并检查它是否在指定范围内。

这两个函数的参数是一个 `T` 类型的值 `value` 和一个整数 `upper`，其中 `upper` 是一个整数，用于指定值的范围。函数实现中首先检查 `value` 是否大于等于 0，并且是否小于等于 `upper`，如果是，则将 `value` 转换为相应的非负整数并返回。如果不是，则输出 "invalid value"。

这里使用了 C++ 的模板元编程特性，模板的使用可以使得这段代码更易于理解和维护，同时也可以在不同的输入值上得到正确的结果，避免了写出长篇大论的代码。


```cpp
// Converts value to int and checks that it's in the range [0, upper).
template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
inline int to_nonnegative_int(T value, int upper) {
  FMT_ASSERT(value >= 0 && value <= upper, "invalid value");
  (void)upper;
  return static_cast<int>(value);
}
template <typename T, FMT_ENABLE_IF(!std::is_integral<T>::value)>
inline int to_nonnegative_int(T value, int upper) {
  FMT_ASSERT(
      std::isnan(value) || (value >= 0 && value <= static_cast<T>(upper)),
      "invalid value");
  (void)upper;
  return static_cast<int>(value);
}

```

这段代码定义了两个模板别论类型为T的函数mod，用于求一个整数型变量x对整数型变量y取模的结果。

第一个模板别论类型为T，函数采用int型参数，表示输入整数型变量x，输出整数型变量y。通过使用FMT_ENABLE_IF宏，当T为整数型时，编译器会根据INTEGRAL模板参数判断，如果是整数型则执行int型函数，否则执行float型函数。

第二个模板别论类型为T，函数采用int型参数，表示输入整数型变量x，输出浮点型变量y。通过使用FMT_ENABLE_IF宏，当T为浮点型时，编译器会根据INTEGRAL模板参数判断，如果是整数型则执行int型函数，否则执行float型函数。

同时，还定义了一个名为make_unsigned_or_unchanged的结构体类型，用于在T为整数型时将T转换为unsigned类型，否则不改变T的类型。


```cpp
template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
inline T mod(T x, int y) {
  return x % static_cast<T>(y);
}
template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
inline T mod(T x, int y) {
  return std::fmod(x, static_cast<T>(y));
}

// If T is an integral type, maps T to its unsigned counterpart, otherwise
// leaves it unchanged (unlike std::make_unsigned).
template <typename T, bool INTEGRAL = std::is_integral<T>::value>
struct make_unsigned_or_unchanged {
  using type = T;
};

```

这段代码定义了一个模板结构体 `make_unsigned_or_launchched<int, true>`，其中 `T` 是一个模板参数，`int` 是 `T` 的别名，`true` 是模板参数的约束，表示 `T` 必须是 `int` 类型的别名。

这个模板结构体中定义了一个 `using type = typename std::make_unsigned<T>::type;` 一行，表示定义了一个 `using` 别名，它依赖于 `std::make_unsigned<T>` 函数的返回类型。

接下来的 `#if FMT_SAFE_DURATION_CAST` 是一个带 `FMT_SAFE_DURATION_CAST` 前缀的函数声明，表示这个函数是一个安全的 duration 类型转换函数。函数接收两个 `std::chrono::duration<int, int>` 的参数 `from` 和 `to`，返回一个 `int` 类型的值。函数内部使用了 `safe_duration_cast` 函数，它是一个安全的 duration 类型转换函数，会尝试将 `from` 转换为 `to` 所表示的类型，如果转换失败，会抛出 `fmt_error` 异常。

最后一行 `#endif` 是 `#if FMT_SAFE_DURATION_CAST` 语句的结束，表示可以安全地使用这个函数。


```cpp
template <typename T> struct make_unsigned_or_unchanged<T, true> {
  using type = typename std::make_unsigned<T>::type;
};

#if FMT_SAFE_DURATION_CAST
// throwing version of safe_duration_cast
template <typename To, typename FromRep, typename FromPeriod>
To fmt_safe_duration_cast(std::chrono::duration<FromRep, FromPeriod> from) {
  int ec;
  To to = safe_duration_cast::safe_duration_cast<To>(from, ec);
  if (ec) FMT_THROW(format_error("cannot format duration"));
  return to;
}
#endif

```

这段代码定义了一个名为 `get_milliseconds` 的函数，它接受一个 `std::chrono::duration<Rep, Period>` 的参数 `d`。

函数的作用是将 `d` 中的时间转换为毫秒（`std::milli` 类型）并返回。首先，它检查 `d` 是否是整数类型，如果是，则使用 `std::chrono::duration<Rep, std::milli>>` 类型的函数来返回毫秒。如果不是整数类型，则将 `d` 转换为 `std::chrono::seconds` 类型，然后使用 `fmt_safe_duration_cast<std::chrono::seconds>(d_as_common)` 来获取整数类型的 `d` 中的时间，最后再将整数类型的时间转换为毫秒。

为了确保函数的安全性，使用了 `FMT_SAFE_DURATION_CAST` 函数来安全的进行时间类型转换。


```cpp
template <typename Rep, typename Period,
          FMT_ENABLE_IF(std::is_integral<Rep>::value)>
inline std::chrono::duration<Rep, std::milli> get_milliseconds(
    std::chrono::duration<Rep, Period> d) {
  // this may overflow and/or the result may not fit in the
  // target type.
#if FMT_SAFE_DURATION_CAST
  using CommonSecondsType =
      typename std::common_type<decltype(d), std::chrono::seconds>::type;
  const auto d_as_common = fmt_safe_duration_cast<CommonSecondsType>(d);
  const auto d_as_whole_seconds =
      fmt_safe_duration_cast<std::chrono::seconds>(d_as_common);
  // this conversion should be nonproblematic
  const auto diff = d_as_common - d_as_whole_seconds;
  const auto ms =
      fmt_safe_duration_cast<std::chrono::duration<Rep, std::milli>>(diff);
  return ms;
```

这段代码是一个C++函数，名为`get_milliseconds`，它接受一个`std::chrono::duration`类型的参数`d`，并返回一个`std::chrono::milliseconds`类型的值。

首先，函数检查参数`d`是否为空（`{}`），如果是，则输出一个警告信息并返回一个`std::chrono::milliseconds`类型的值，说明函数不会对空`d`进行处理。这表明了函数的文档中不会提供实际输出，因为这个函数的作用就是在开发过程中提醒你不要将`d`赋值为空。

否则，函数根据传入的`d`创建了一个`std::chrono::duration_cast<std::chrono::seconds>`类型，该类型表示`std::chrono::seconds`和`std::chrono::milliseconds`之间的转换。函数的第二个参数是一个`std::chrono::duration_cast<std::chrono::seconds>`类型的变量`s`，用于存储`d`减去`d`中`std::chrono::seconds`部分的值。

接下来，函数使用一个模板类`std::chrono::duration<typename Rep, typename Period, FMT_ENABLE_IF(std::is_floating_point<Rep>::value)>`来创建一个`std::chrono::duration<Rep, std::milli>`类型的值，其中`Rep`是类型参数，`Period`是一个用于表示周期性事件的类型，它可以从`std::chrono::duration<Rep, Period>`中减去一个`std::chrono::seconds>`类型的值。

最后，函数使用一个模板函数来计算`d`中`std::chrono::milliseconds`部分的值，这个函数将`ms`的值存储在一个`std::chrono::milliseconds`类型的变量中，其中`ms`是通过将`d.count()`乘以一个整数类型（`std::chrono::duration<Rep, Period>::num`）除以另一个整数类型（`std::chrono::duration<Rep, Period>::den`）得到的结果，再将结果乘以1000得到一个`std::chrono::milliseconds`类型的值。

综上所述，这段代码的作用是获取一个`std::chrono::duration<typename Rep, typename Period, FMT_ENABLE_IF(std::is_floating_point<Rep>::value)>`类型的`std::chrono::duration<Rep, std::milli>`类型的值，其中的`std::chrono::milliseconds`部分是由`std::chrono::duration_cast<std::chrono::seconds>(d)`计算得到的。


```cpp
#else
  auto s = std::chrono::duration_cast<std::chrono::seconds>(d);
  return std::chrono::duration_cast<std::chrono::milliseconds>(d - s);
#endif
}

template <typename Rep, typename Period,
          FMT_ENABLE_IF(std::is_floating_point<Rep>::value)>
inline std::chrono::duration<Rep, std::milli> get_milliseconds(
    std::chrono::duration<Rep, Period> d) {
  using common_type = typename std::common_type<Rep, std::intmax_t>::type;
  auto ms = mod(d.count() * static_cast<common_type>(Period::num) /
                    static_cast<common_type>(Period::den) * 1000,
                1000);
  return std::chrono::duration<Rep, std::milli>(static_cast<Rep>(ms));
}

```

这段代码定义了两个模板函数，分别是 `format_duration_value` 和 `copy_unit`。它们的作用如下：

1. `format_duration_value`：该函数用于将一个 `Rep` 类型的值（例如 `int` 类型）转换为指定精度的 `OutputIt` 类型的值。函数的实现包括以下步骤：

  a. 定义一个名为 `pr_f` 的字符数组，用于存储输入值的占位符。这些占位符将会被用来格式化输入值。

  b. 如果输入值精度高于 0，那么函数将调用 `format_to` 函数。这个函数的作用是接收一个 `OutputIt` 类型的输出流和一个字符数组（占位符），将输入值 format 并输出到指定的流中。

  c. 如果输入值精度低于 0，那么函数将调用 `is_floating_point` 函数。这个函数的作用是检查输入值是否为浮点数，如果是，则使用 `fp_f` 字符串格式化占位符。

  d. 如果输入值既不是浮点数也不是占位符，那么函数将直接输出输入值。

2. `copy_unit`：该函数用于将一个字符串类型的单位（例如 `string_view` 类型）复制到指定的 `OutputIt` 类型的输出流中。函数的实现包括以下步骤：

  a. 定义一个名为 `unit` 的字符串类型的变量，包含要复制的单位。

  b. 定义一个名为 `out` 的 `OutputIt` 类型的变量，用于存储复制后的内容。

  c. 调用 `std::copy` 函数，将 `unit` 中的内容复制到 `out` 中。

  注意：如果 `out` 也是字符串类型，则 `std::copy` 函数将会在复制之前将 `out` 中的所有内容复制到 `unit` 中。


```cpp
template <typename Char, typename Rep, typename OutputIt>
OutputIt format_duration_value(OutputIt out, Rep val, int precision) {
  const Char pr_f[] = {'{', ':', '.', '{', '}', 'f', '}', 0};
  if (precision >= 0) return format_to(out, pr_f, val, precision);
  const Char fp_f[] = {'{', ':', 'g', '}', 0};
  const Char format[] = {'{', '}', 0};
  return format_to(out, std::is_floating_point<Rep>::value ? fp_f : format,
                   val);
}
template <typename Char, typename OutputIt>
OutputIt copy_unit(string_view unit, OutputIt out, Char) {
  return std::copy(unit.begin(), unit.end(), out);
}

template <typename OutputIt>
```

这段代码定义了两个函数，`copy_unit` 和 `format_duration_unit`，它们分别用于将一个 `string_view` 类型的单位字符串复制到 `OutputIt` 类型的输出中。

`copy_unit` 函数的实现比较复杂，但可以得到一个正确的结果：将给定的 `string_view` 类型的单位字符串复制到 `OutputIt` 类型的输出中。具体的实现过程可参考上面的注释。

`format_duration_unit` 函数用于将一个 `string_view` 类型的单位字符串格式化成 `OutputIt` 类型。它首先检查给定的 `string_view` 是否包含一个有效的单位字符串，然后根据不同的单位字符串采取不同的格式化策略。如果给定的 `Period` 对象是一个 `null_pointer`，则函数的行为为未定义的，这意味着输出为未定义的 `null_pointer`。否则，函数将根据给定的 `den` 值选择正确的格式化策略，并将得到的结果返回给 `format_duration_unit`。

总的来说，这两个函数都是帮助用户将字符串单位化成 `OutputIt` 类型，尽管具体的实现有所不同。


```cpp
OutputIt copy_unit(string_view unit, OutputIt out, wchar_t) {
  // This works when wchar_t is UTF-32 because units only contain characters
  // that have the same representation in UTF-16 and UTF-32.
  utf8_to_utf16 u(unit);
  return std::copy(u.c_str(), u.c_str() + u.size(), out);
}

template <typename Char, typename Period, typename OutputIt>
OutputIt format_duration_unit(OutputIt out) {
  if (const char* unit = get_units<Period>())
    return copy_unit(string_view(unit), out, Char());
  const Char num_f[] = {'[', '{', '}', ']', 's', 0};
  if (const_check(Period::den == 1)) return format_to(out, num_f, Period::num);
  const Char num_def_f[] = {'[', '{', '}', '/', '{', '}', ']', 's', 0};
  return format_to(out, num_def_f, Period::num, Period::den);
}

```

这段代码定义了一个名为`chrono_formatter`的结构体，用于将`std::chrono`库中的`Rep`类型的时间数据格式化并输出。

该结构体包含以下成员：

1. `FormatContext&`：引用`FormatContext`模板类型，用于格式化输出。
2. `OutputIt`：引用输出流（`std::out`）类型，用于存储格式化后的时间数据。
3. `int precision`：用于表示输出数据的精度。
4. `using rep`：定义了一个名为`rep`的类型，用于表示输入数据类型。`rep`通过条件语句`std::is_integral<Rep>::value && sizeof(Rep) < sizeof(int)`判断是否为整数类型，如果不是，则使用`unsigned`类型。
5. `using seconds`：定义了一个名为`seconds`的类型，用于表示输入数据类型，它是一个`std::chrono::duration<rep, Period>`类型的别名。
6. `using milliseconds`：定义了一个名为`milliseconds`的类型，用于表示输入数据类型，它是一个`std::chrono::duration<rep, std::milli>`类型的别名。
7. `bool negative`：定义了一个名为`negative`的布尔变量，表示输入数据是否为负数。
8. `using char_type`：定义了一个名为`char_type`的类型，用于表示输出数据类型。

该结构体的定义是在`chrono_formatter`的构造函数中进行的。构造函数的第一个参数是`FormatContext`模板类型，第二个参数是输出流类型，第三个参数是输入数据类型，第四个参数是输出数据的精度。

该结构体在`chrono_formatter`的实现中还有两个辅助成员函数：`static void format_seconds()`和`static void format_milliseconds()`。这些函数分别用于将`seconds`和`milliseconds`类型的输入数据格式化并输出。


```cpp
template <typename FormatContext, typename OutputIt, typename Rep,
          typename Period>
struct chrono_formatter {
  FormatContext& context;
  OutputIt out;
  int precision;
  // rep is unsigned to avoid overflow.
  using rep =
      conditional_t<std::is_integral<Rep>::value && sizeof(Rep) < sizeof(int),
                    unsigned, typename make_unsigned_or_unchanged<Rep>::type>;
  rep val;
  using seconds = std::chrono::duration<rep>;
  seconds s;
  using milliseconds = std::chrono::duration<rep, std::milli>;
  bool negative;

  using char_type = typename FormatContext::char_type;

  explicit chrono_formatter(FormatContext& ctx, OutputIt o,
                            std::chrono::duration<Rep, Period> d)
      : context(ctx),
        out(o),
        val(static_cast<rep>(d.count())),
        negative(false) {
    if (d.count() < 0) {
      val = 0 - val;
      negative = true;
    }

    // this may overflow and/or the result may not fit in the
    // target type.
```

This appears to be a C++ program that formats a time string according to a given numeric system (e.g., ISO 8601, SQL, etc.). It includes functions for handling time zones, 24-hour and 12-hour time formats, minute and second levels, and different numeric systems.

The `on_dec0_weekday()`, `on_dec1_weekday()`, `on_abbr_month()`, `on_full_month()`, `on_datetime()`, `on_loc_date()`, `on_loc_time()`, `on_us_date()`, `on_iso_date()`, and `on_utc_offset()` functions handle various aspects of the time formatting, such as handling NaNs, ISO 8601, and SQL-specific date formats.

The `write()` function is used to write the formatted time to the console, and the `format_localized()` function is used for localization and formatting.


```cpp
#if FMT_SAFE_DURATION_CAST
    // might need checked conversion (rep!=Rep)
    auto tmpval = std::chrono::duration<rep, Period>(val);
    s = fmt_safe_duration_cast<seconds>(tmpval);
#else
    s = std::chrono::duration_cast<seconds>(
        std::chrono::duration<rep, Period>(val));
#endif
  }

  // returns true if nan or inf, writes to out.
  bool handle_nan_inf() {
    if (isfinite(val)) {
      return false;
    }
    if (isnan(val)) {
      write_nan();
      return true;
    }
    // must be +-inf
    if (val > 0) {
      write_pinf();
    } else {
      write_ninf();
    }
    return true;
  }

  Rep hour() const { return static_cast<Rep>(mod((s.count() / 3600), 24)); }

  Rep hour12() const {
    Rep hour = static_cast<Rep>(mod((s.count() / 3600), 12));
    return hour <= 0 ? 12 : hour;
  }

  Rep minute() const { return static_cast<Rep>(mod((s.count() / 60), 60)); }
  Rep second() const { return static_cast<Rep>(mod(s.count(), 60)); }

  std::tm time() const {
    auto time = std::tm();
    time.tm_hour = to_nonnegative_int(hour(), 24);
    time.tm_min = to_nonnegative_int(minute(), 60);
    time.tm_sec = to_nonnegative_int(second(), 60);
    return time;
  }

  void write_sign() {
    if (negative) {
      *out++ = '-';
      negative = false;
    }
  }

  void write(Rep value, int width) {
    write_sign();
    if (isnan(value)) return write_nan();
    uint32_or_64_or_128_t<int> n =
        to_unsigned(to_nonnegative_int(value, max_value<int>()));
    int num_digits = detail::count_digits(n);
    if (width > num_digits) out = std::fill_n(out, width - num_digits, '0');
    out = format_decimal<char_type>(out, n, num_digits).end;
  }

  void write_nan() { std::copy_n("nan", 3, out); }
  void write_pinf() { std::copy_n("inf", 3, out); }
  void write_ninf() { std::copy_n("-inf", 4, out); }

  void format_localized(const tm& time, char format, char modifier = 0) {
    if (isnan(val)) return write_nan();
    auto locale = context.locale().template get<std::locale>();
    auto& facet = std::use_facet<std::time_put<char_type>>(locale);
    std::basic_ostringstream<char_type> os;
    os.imbue(locale);
    facet.put(os, os, ' ', &time, format, modifier);
    auto str = os.str();
    std::copy(str.begin(), str.end(), out);
  }

  void on_text(const char_type* begin, const char_type* end) {
    std::copy(begin, end, out);
  }

  // These are not implemented because durations don't have date information.
  void on_abbr_weekday() {}
  void on_full_weekday() {}
  void on_dec0_weekday(numeric_system) {}
  void on_dec1_weekday(numeric_system) {}
  void on_abbr_month() {}
  void on_full_month() {}
  void on_datetime(numeric_system) {}
  void on_loc_date(numeric_system) {}
  void on_loc_time(numeric_system) {}
  void on_us_date() {}
  void on_iso_date() {}
  void on_utc_offset() {}
  void on_tz_name() {}

  void on_24_hour(numeric_system ns) {
    if (handle_nan_inf()) return;

    if (ns == numeric_system::standard) return write(hour(), 2);
    auto time = tm();
    time.tm_hour = to_nonnegative_int(hour(), 24);
    format_localized(time, 'H', 'O');
  }

  void on_12_hour(numeric_system ns) {
    if (handle_nan_inf()) return;

    if (ns == numeric_system::standard) return write(hour12(), 2);
    auto time = tm();
    time.tm_hour = to_nonnegative_int(hour12(), 12);
    format_localized(time, 'I', 'O');
  }

  void on_minute(numeric_system ns) {
    if (handle_nan_inf()) return;

    if (ns == numeric_system::standard) return write(minute(), 2);
    auto time = tm();
    time.tm_min = to_nonnegative_int(minute(), 60);
    format_localized(time, 'M', 'O');
  }

  void on_second(numeric_system ns) {
    if (handle_nan_inf()) return;

    if (ns == numeric_system::standard) {
      write(second(), 2);
```

This is a C++ program that writes a time value to a stream. The time value is stored in a `std::chrono::duration<rep, Period>` object. The program reads the time value from the input and formats it using the `std::chrono::duration<Rep, Period>` and `std::chrono::duration<Rep, Period>` types.

The program provides several overloaded functions for formatting the time value as follows:

* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`
* `std::chrono::duration<Rep, Period>` overload for formatting the time as a `std::chrono::duration<Rep, Period>`
* `std::chrono::duration<rep, Period>` overload for formatting the time as a `std::chrono::duration<rep, Period>`

The program also provides an `on_12_hour_time()`, `on_24_hour_time()`, `on_iso_time()`, `on_am_pm()`, and `on_duration_value()` functions. These functions are not used by the program but are derived from the `std::chrono::duration<rep, Period>` and `std::chrono::duration<rep, Period>` types.


```cpp
#if FMT_SAFE_DURATION_CAST
      // convert rep->Rep
      using duration_rep = std::chrono::duration<rep, Period>;
      using duration_Rep = std::chrono::duration<Rep, Period>;
      auto tmpval = fmt_safe_duration_cast<duration_Rep>(duration_rep{val});
#else
      auto tmpval = std::chrono::duration<Rep, Period>(val);
#endif
      auto ms = get_milliseconds(tmpval);
      if (ms != std::chrono::milliseconds(0)) {
        *out++ = '.';
        write(ms.count(), 3);
      }
      return;
    }
    auto time = tm();
    time.tm_sec = to_nonnegative_int(second(), 60);
    format_localized(time, 'S', 'O');
  }

  void on_12_hour_time() {
    if (handle_nan_inf()) return;
    format_localized(time(), 'r');
  }

  void on_24_hour_time() {
    if (handle_nan_inf()) {
      *out++ = ':';
      handle_nan_inf();
      return;
    }

    write(hour(), 2);
    *out++ = ':';
    write(minute(), 2);
  }

  void on_iso_time() {
    on_24_hour_time();
    *out++ = ':';
    if (handle_nan_inf()) return;
    write(second(), 2);
  }

  void on_am_pm() {
    if (handle_nan_inf()) return;
    format_localized(time(), 'p');
  }

  void on_duration_value() {
    if (handle_nan_inf()) return;
    write_sign();
    out = format_duration_value<char_type>(out, val, precision);
  }

  void on_duration_unit() {
    out = format_duration_unit<char_type, Period>(out);
  }
};
}  // namespace detail

```

`fmt` is a simple `fmt` parser that parses a `basic_format_parse_context` and returns the result.
It can parse `basic_string` and `basic_string_view` objects.
It can parse `duration` object with `detail::chrono_formatter` and `detail::format_duration_unit`
It can handle the precision of the output.
It can only handle the specific types of the input.
It is the base class for the `fmt` parser.


```cpp
template <typename Rep, typename Period, typename Char>
struct formatter<std::chrono::duration<Rep, Period>, Char> {
 private:
  basic_format_specs<Char> specs;
  int precision;
  using arg_ref_type = detail::arg_ref<Char>;
  arg_ref_type width_ref;
  arg_ref_type precision_ref;
  mutable basic_string_view<Char> format_str;
  using duration = std::chrono::duration<Rep, Period>;

  struct spec_handler {
    formatter& f;
    basic_format_parse_context<Char>& context;
    basic_string_view<Char> format_str;

    template <typename Id> FMT_CONSTEXPR arg_ref_type make_arg_ref(Id arg_id) {
      context.check_arg_id(arg_id);
      return arg_ref_type(arg_id);
    }

    FMT_CONSTEXPR arg_ref_type make_arg_ref(basic_string_view<Char> arg_id) {
      context.check_arg_id(arg_id);
      return arg_ref_type(arg_id);
    }

    FMT_CONSTEXPR arg_ref_type make_arg_ref(detail::auto_id) {
      return arg_ref_type(context.next_arg_id());
    }

    void on_error(const char* msg) { FMT_THROW(format_error(msg)); }
    void on_fill(basic_string_view<Char> fill) { f.specs.fill = fill; }
    void on_align(align_t align) { f.specs.align = align; }
    void on_width(int width) { f.specs.width = width; }
    void on_precision(int _precision) { f.precision = _precision; }
    void end_precision() {}

    template <typename Id> void on_dynamic_width(Id arg_id) {
      f.width_ref = make_arg_ref(arg_id);
    }

    template <typename Id> void on_dynamic_precision(Id arg_id) {
      f.precision_ref = make_arg_ref(arg_id);
    }
  };

  using iterator = typename basic_format_parse_context<Char>::iterator;
  struct parse_range {
    iterator begin;
    iterator end;
  };

  FMT_CONSTEXPR parse_range do_parse(basic_format_parse_context<Char>& ctx) {
    auto begin = ctx.begin(), end = ctx.end();
    if (begin == end || *begin == '}') return {begin, begin};
    spec_handler handler{*this, ctx, format_str};
    begin = detail::parse_align(begin, end, handler);
    if (begin == end) return {begin, begin};
    begin = detail::parse_width(begin, end, handler);
    if (begin == end) return {begin, begin};
    if (*begin == '.') {
      if (std::is_floating_point<Rep>::value)
        begin = detail::parse_precision(begin, end, handler);
      else
        handler.on_error("precision not allowed for this argument type");
    }
    end = parse_chrono_format(begin, end, detail::chrono_format_checker());
    return {begin, end};
  }

 public:
  formatter() : precision(-1) {}

  FMT_CONSTEXPR auto parse(basic_format_parse_context<Char>& ctx)
      -> decltype(ctx.begin()) {
    auto range = do_parse(ctx);
    format_str = basic_string_view<Char>(
        &*range.begin, detail::to_unsigned(range.end - range.begin));
    return range.end;
  }

  template <typename FormatContext>
  auto format(const duration& d, FormatContext& ctx) -> decltype(ctx.out()) {
    auto begin = format_str.begin(), end = format_str.end();
    // As a possible future optimization, we could avoid extra copying if width
    // is not specified.
    basic_memory_buffer<Char> buf;
    auto out = std::back_inserter(buf);
    detail::handle_dynamic_spec<detail::width_checker>(specs.width, width_ref,
                                                       ctx);
    detail::handle_dynamic_spec<detail::precision_checker>(precision,
                                                           precision_ref, ctx);
    if (begin == end || *begin == '}') {
      out = detail::format_duration_value<Char>(out, d.count(), precision);
      detail::format_duration_unit<Char, Period>(out);
    } else {
      detail::chrono_formatter<FormatContext, decltype(out), Rep, Period> f(
          ctx, out, d);
      f.precision = precision;
      parse_chrono_format(begin, end, f);
    }
    return detail::write(
        ctx.out(), basic_string_view<Char>(buf.data(), buf.size()), specs);
  }
};

```

这段代码是一个C++预处理指令，名为 `FMT_END_NAMESPACE`。它用于告诉编译器在定义和使用这个命名空间（namespace）时，命名空间内的所有类型和成员都将被视为外部（external）。

当我们在源代码中使用这个命名空间时，我们需要确保所有的外部类型和成员都可以被正确地访问和使用。通过声明这个命名空间作为外部，我们可以保证在程序中定义和使用这个命名空间时，编译器会 correctly地识别其内部类型和成员。

然而，需要注意的是，这个预处理指令仅仅告诉编译器在定义和使用命名空间时将其视为外部，但实际的类型和成员仍然可能需要被正确地访问和使用。因此，在使用这个命名空间时，我们需要确保所有的内部类型和成员都被正确地定义和使用。


```cpp
FMT_END_NAMESPACE

#endif  // FMT_CHRONO_H_

```