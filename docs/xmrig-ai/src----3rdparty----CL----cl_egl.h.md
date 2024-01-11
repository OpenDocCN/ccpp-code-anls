# `xmrig\src\3rdparty\CL\cl_egl.h`

```
/*
 * 版权声明
 * 2008-2019年，The Khronos Group Inc. 版权所有
 *
 * 在遵守以下条件的情况下，允许任何获得本软件和/或相关文档文件（以下简称"材料"）的人免费使用材料：
 * 无限制地处理材料，包括但不限于使用、复制、修改、合并、发布、分发、许可和/或出售材料的副本，并允许提供材料的人员这样做，但须遵守以下条件：
 * 上述版权声明和本许可声明应包含在所有材料的所有副本或重要部分中。
 *
 * 对本文件的修改可能意味着它不再准确反映 Khronos 标准。Khronos 规范和头文件的未修改、规范版本位于 https://www.khronos.org/registry/
 *
 * 材料按"原样"提供，不提供任何形式的明示或暗示保证，包括但不限于对适销性、特定用途的适用性和非侵权的保证。无论在合同、侵权行为或其他方面，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是因合同行为、侵权行为还是其他原因，与材料的使用或其他交易有关。
 */

#ifndef __OPENCL_CL_EGL_H
#define __OPENCL_CL_EGL_H

#include <CL/cl.h>

#ifdef __cplusplus
extern "C" {
#endif

/* 用于使用 clEnqueueAcquireEGLObjectsKHR 创建的事件的命令类型 */
#define CL_COMMAND_EGL_FENCE_SYNC_OBJECT_KHR  0x202F
#define CL_COMMAND_ACQUIRE_EGL_OBJECTS_KHR    0x202D
#define CL_COMMAND_RELEASE_EGL_OBJECTS_KHR    0x202E

/* clCreateFromEGLImageKHR 的错误类型 */
#define CL_INVALID_EGL_OBJECT_KHR             -1093
#define CL_EGL_RESOURCE_NOT_ACQUIRED_KHR      -1092
# 定义 EGL 资源未获取的错误码为 -1092

/* CLeglImageKHR is an opaque handle to an EGLImage */
typedef void* CLeglImageKHR;
# 定义 CLeglImageKHR 为指向 EGLImage 的不透明句柄

/* CLeglDisplayKHR is an opaque handle to an EGLDisplay */
typedef void* CLeglDisplayKHR;
# 定义 CLeglDisplayKHR 为指向 EGLDisplay 的不透明句柄

/* CLeglSyncKHR is an opaque handle to an EGLSync object */
typedef void* CLeglSyncKHR;
# 定义 CLeglSyncKHR 为指向 EGLSync 对象的不透明句柄

/* properties passed to clCreateFromEGLImageKHR */
typedef intptr_t cl_egl_image_properties_khr;
# 定义传递给 clCreateFromEGLImageKHR 的属性

#define cl_khr_egl_image 1
# 定义 cl_khr_egl_image 为 1

extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateFromEGLImageKHR(cl_context                  context,
                        CLeglDisplayKHR             egldisplay,
                        CLeglImageKHR               eglimage,
                        cl_mem_flags                flags,
                        const cl_egl_image_properties_khr * properties,
                        cl_int *                    errcode_ret) CL_API_SUFFIX__VERSION_1_0;
# 声明 clCreateFromEGLImageKHR 函数，用于从 EGLImage 创建内存对象

typedef CL_API_ENTRY cl_mem (CL_API_CALL *clCreateFromEGLImageKHR_fn)(
    cl_context                  context,
    CLeglDisplayKHR             egldisplay,
    CLeglImageKHR               eglimage,
    cl_mem_flags                flags,
    const cl_egl_image_properties_khr * properties,
    cl_int *                    errcode_ret);
# 定义 clCreateFromEGLImageKHR_fn 函数指针类型

extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueAcquireEGLObjectsKHR(cl_command_queue command_queue,
                              cl_uint          num_objects,
                              const cl_mem *   mem_objects,
                              cl_uint          num_events_in_wait_list,
                              const cl_event * event_wait_list,
                              cl_event *       event) CL_API_SUFFIX__VERSION_1_0;
# 声明 clEnqueueAcquireEGLObjectsKHR 函数，用于在命令队列中获取 EGL 对象

typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueAcquireEGLObjectsKHR_fn)(
    cl_command_queue command_queue,
    cl_uint          num_objects,
    const cl_mem *   mem_objects,
    cl_uint          num_events_in_wait_list,
    const cl_event * event_wait_list,
    cl_event *       event);
# 定义 clEnqueueAcquireEGLObjectsKHR_fn 函数指针类型
# 定义一个用于在命令队列中释放 EGL 对象的函数，返回一个整数错误码
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueReleaseEGLObjectsKHR(cl_command_queue command_queue,  # 命令队列
                              cl_uint          num_objects,     # 对象数量
                              const cl_mem *   mem_objects,     # 内存对象数组
                              cl_uint          num_events_in_wait_list,  # 等待列表中的事件数量
                              const cl_event * event_wait_list,  # 等待列表中的事件数组
                              cl_event *       event) CL_API_SUFFIX__VERSION_1_0;  # 事件对象

# 定义一个用于在命令队列中释放 EGL 对象的函数指针类型
typedef CL_API_ENTRY cl_int (CL_API_CALL *clEnqueueReleaseEGLObjectsKHR_fn)(
    cl_command_queue command_queue,  # 命令队列
    cl_uint          num_objects,     # 对象数量
    const cl_mem *   mem_objects,     # 内存对象数组
    cl_uint          num_events_in_wait_list,  # 等待列表中的事件数量
    const cl_event * event_wait_list,  # 等待列表中的事件数组
    cl_event *       event);

# 定义一个用于从 EGL 同步对象创建事件的函数，返回一个事件对象
extern CL_API_ENTRY cl_event CL_API_CALL
clCreateEventFromEGLSyncKHR(cl_context      context,  # 上下文
                            CLeglSyncKHR    sync,     # EGL 同步对象
                            CLeglDisplayKHR display,  # EGL 显示对象
                            cl_int *        errcode_ret) CL_API_SUFFIX__VERSION_1_0;  # 错误码

# 定义一个用于从 EGL 同步对象创建事件的函数指针类型
typedef CL_API_ENTRY cl_event (CL_API_CALL *clCreateEventFromEGLSyncKHR_fn)(
    cl_context      context,  # 上下文
    CLeglSyncKHR    sync,     # EGL 同步对象
    CLeglDisplayKHR display,  # EGL 显示对象
    cl_int *        errcode_ret);

#ifdef __cplusplus
}
#endif

#endif /* __OPENCL_CL_EGL_H */
```