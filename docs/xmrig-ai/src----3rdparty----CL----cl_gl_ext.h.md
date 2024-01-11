# `xmrig\src\3rdparty\CL\cl_gl_ext.h`

```
/**********************************************************************************
 * 版权所有（c）2008-2019 The Khronos Group Inc.
 *
 * 在不受限制的情况下，允许任何获得本软件和/或相关文档文件（以下简称“材料”）的人
 * 复制、使用、修改、合并、发布、分发、许可和/或出售材料的副本，并允许
 * 将材料提供给材料的受益人，但须遵守以下条件：
 *
 * 所有副本或实质部分的材料中必须包含上述版权声明和本许可声明。
 *
 * 对本文件的修改可能意味着它不再准确反映
 * Khronos 标准。Khronos 的未修改的规范和头文件信息的权威版本位于
 *    https://www.khronos.org/registry/
 *
 * 材料按“原样”提供，不附带任何形式的明示或暗示的保证，
 * 包括但不限于对特定目的的适销性、适用性和非侵权性的保证。
 * 作者或版权持有人在任何情况下均不对任何索赔、损害或其他责任承担责任，
 * 无论是合同诉讼、侵权行为还是其他方式，由于材料或使用材料而产生、
 * 由此引起或与之相关。
 **********************************************************************************/

#ifndef __OPENCL_CL_GL_EXT_H
#define __OPENCL_CL_GL_EXT_H

#ifdef __cplusplus
extern "C" {
#endif

#include <CL/cl_gl.h>

/*
 *  cl_khr_gl_event extension
 */
#define CL_COMMAND_GL_FENCE_SYNC_OBJECT_KHR     0x200D

// 从 OpenGL 同步对象创建 OpenCL 事件
extern CL_API_ENTRY cl_event CL_API_CALL
clCreateEventFromGLsyncKHR(cl_context context,
                           cl_GLsync  cl_GLsync,
                           cl_int *   errcode_ret) CL_EXT_SUFFIX__VERSION_1_1;
#ifdef __cplusplus
}  // 结束 C++ 的 extern "C" 块
#endif

#endif    /* __OPENCL_CL_GL_EXT_H  */  // 结束条件编译，关闭 OPENCL_CL_GL_EXT_H 头文件的定义
```