# `xmrig\src\backend\opencl\wrappers\OclError.cpp`

```
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018-2019 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据 GNU 通用公共许可证的条款发布，由
 * 自由软件基金会发布的许可证的第3版，或者
 * （根据您的选择）任何以后的版本。
 *
 * 本程序是希望它有用，
 * 但没有任何保证；甚至没有暗示的保证
 * 适销性或特定用途的保证。详细信息请参见
 * GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见<http://www.gnu.org/licenses/>。
 */

#include "backend/opencl/wrappers/OclError.h"

// 将 OpenCL 错误码转换为字符串
const char *xmrig::OclError::toString(cl_int ret)
{
    switch(ret)
    {
    case CL_SUCCESS:
        return "CL_SUCCESS";
    case CL_DEVICE_NOT_FOUND:
        return "CL_DEVICE_NOT_FOUND";
    case CL_DEVICE_NOT_AVAILABLE:
        return "CL_DEVICE_NOT_AVAILABLE";
    case CL_COMPILER_NOT_AVAILABLE:
        return "CL_COMPILER_NOT_AVAILABLE";
    case CL_MEM_OBJECT_ALLOCATION_FAILURE:
        return "CL_MEM_OBJECT_ALLOCATION_FAILURE";
    case CL_OUT_OF_RESOURCES:
        return "CL_OUT_OF_RESOURCES";
    case CL_OUT_OF_HOST_MEMORY:
        return "CL_OUT_OF_HOST_MEMORY";
    case CL_PROFILING_INFO_NOT_AVAILABLE:
        return "CL_PROFILING_INFO_NOT_AVAILABLE";
    # 返回错误代码对应的字符串描述
    case CL_MEM_COPY_OVERLAP:
        return "CL_MEM_COPY_OVERLAP";
    # 返回错误代码对应的字符串描述
    case CL_IMAGE_FORMAT_MISMATCH:
        return "CL_IMAGE_FORMAT_MISMATCH";
    # 返回错误代码对应的字符串描述
    case CL_IMAGE_FORMAT_NOT_SUPPORTED:
        return "CL_IMAGE_FORMAT_NOT_SUPPORTED";
    # 返回错误代码对应的字符串描述
    case CL_BUILD_PROGRAM_FAILURE:
        return "CL_BUILD_PROGRAM_FAILURE";
    # 返回错误代码对应的字符串描述
    case CL_MAP_FAILURE:
        return "CL_MAP_FAILURE";
    # 返回错误代码对应的字符串描述
    case CL_MISALIGNED_SUB_BUFFER_OFFSET:
        return "CL_MISALIGNED_SUB_BUFFER_OFFSET";
    # 返回错误代码对应的字符串描述
    case CL_EXEC_STATUS_ERROR_FOR_EVENTS_IN_WAIT_LIST:
        return "CL_EXEC_STATUS_ERROR_FOR_EVENTS_IN_WAIT_LIST";
    # 返回错误代码对应的字符串描述
    case CL_COMPILE_PROGRAM_FAILURE:
        return "CL_COMPILE_PROGRAM_FAILURE";
    # 返回错误代码对应的字符串描述
    case CL_LINKER_NOT_AVAILABLE:
        return "CL_LINKER_NOT_AVAILABLE";
    # 返回错误代码对应的字符串描述
    case CL_LINK_PROGRAM_FAILURE:
        return "CL_LINK_PROGRAM_FAILURE";
    # 返回错误代码对应的字符串描述
    case CL_DEVICE_PARTITION_FAILED:
        return "CL_DEVICE_PARTITION_FAILED";
    # 返回错误代码对应的字符串描述
    case CL_KERNEL_ARG_INFO_NOT_AVAILABLE:
        return "CL_KERNEL_ARG_INFO_NOT_AVAILABLE";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_VALUE:
        return "CL_INVALID_VALUE";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_DEVICE_TYPE:
        return "CL_INVALID_DEVICE_TYPE";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_PLATFORM:
        return "CL_INVALID_PLATFORM";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_DEVICE:
        return "CL_INVALID_DEVICE";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_CONTEXT:
        return "CL_INVALID_CONTEXT";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_QUEUE_PROPERTIES:
        return "CL_INVALID_QUEUE_PROPERTIES";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_COMMAND_QUEUE:
        return "CL_INVALID_COMMAND_QUEUE";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_HOST_PTR:
        return "CL_INVALID_HOST_PTR";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_MEM_OBJECT:
        return "CL_INVALID_MEM_OBJECT";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_IMAGE_FORMAT_DESCRIPTOR:
        return "CL_INVALID_IMAGE_FORMAT_DESCRIPTOR";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_IMAGE_SIZE:
        return "CL_INVALID_IMAGE_SIZE";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_SAMPLER:
        return "CL_INVALID_SAMPLER";
    # 返回错误代码对应的字符串描述
    case CL_INVALID_BINARY:
        return "CL_INVALID_BINARY";
    # 如果发生 CL_INVALID_BUILD_OPTIONS 错误，则返回对应的错误信息
    case CL_INVALID_BUILD_OPTIONS:
        return "CL_INVALID_BUILD_OPTIONS";
    # 如果发生 CL_INVALID_PROGRAM 错误，则返回对应的错误信息
    case CL_INVALID_PROGRAM:
        return "CL_INVALID_PROGRAM";
    # 如果发生 CL_INVALID_PROGRAM_EXECUTABLE 错误，则返回对应的错误信息
    case CL_INVALID_PROGRAM_EXECUTABLE:
        return "CL_INVALID_PROGRAM_EXECUTABLE";
    # 如果发生 CL_INVALID_KERNEL_NAME 错误，则返回对应的错误信息
    case CL_INVALID_KERNEL_NAME:
        return "CL_INVALID_KERNEL_NAME";
    # 如果发生 CL_INVALID_KERNEL_DEFINITION 错误，则返回对应的错误信息
    case CL_INVALID_KERNEL_DEFINITION:
        return "CL_INVALID_KERNEL_DEFINITION";
    # 如果发生 CL_INVALID_KERNEL 错误，则返回对应的错误信息
    case CL_INVALID_KERNEL:
        return "CL_INVALID_KERNEL";
    # 如果发生 CL_INVALID_ARG_INDEX 错误，则返回对应的错误信息
    case CL_INVALID_ARG_INDEX:
        return "CL_INVALID_ARG_INDEX";
    # 如果发生 CL_INVALID_ARG_VALUE 错误，则返回对应的错误信息
    case CL_INVALID_ARG_VALUE:
        return "CL_INVALID_ARG_VALUE";
    # 如果发生 CL_INVALID_ARG_SIZE 错误，则返回对应的错误信息
    case CL_INVALID_ARG_SIZE:
        return "CL_INVALID_ARG_SIZE";
    # 如果发生 CL_INVALID_KERNEL_ARGS 错误，则返回对应的错误信息
    case CL_INVALID_KERNEL_ARGS:
        return "CL_INVALID_KERNEL_ARGS";
    # 如果发生 CL_INVALID_WORK_DIMENSION 错误，则返回对应的错误信息
    case CL_INVALID_WORK_DIMENSION:
        return "CL_INVALID_WORK_DIMENSION";
    # 如果发生 CL_INVALID_WORK_GROUP_SIZE 错误，则返回对应的错误信息
    case CL_INVALID_WORK_GROUP_SIZE:
        return "CL_INVALID_WORK_GROUP_SIZE";
    # 如果发生 CL_INVALID_WORK_ITEM_SIZE 错误，则返回对应的错误信息
    case CL_INVALID_WORK_ITEM_SIZE:
        return "CL_INVALID_WORK_ITEM_SIZE";
    # 如果发生 CL_INVALID_GLOBAL_OFFSET 错误，则返回对应的错误信息
    case CL_INVALID_GLOBAL_OFFSET:
        return "CL_INVALID_GLOBAL_OFFSET";
    # 如果发生 CL_INVALID_EVENT_WAIT_LIST 错误，则返回对应的错误信息
    case CL_INVALID_EVENT_WAIT_LIST:
        return "CL_INVALID_EVENT_WAIT_LIST";
    # 如果发生 CL_INVALID_EVENT 错误，则返回对应的错误信息
    case CL_INVALID_EVENT:
        return "CL_INVALID_EVENT";
    # 如果发生 CL_INVALID_OPERATION 错误，则返回对应的错误信息
    case CL_INVALID_OPERATION:
        return "CL_INVALID_OPERATION";
    # 如果发生 CL_INVALID_GL_OBJECT 错误，则返回对应的错误信息
    case CL_INVALID_GL_OBJECT:
        return "CL_INVALID_GL_OBJECT";
    # 如果发生 CL_INVALID_BUFFER_SIZE 错误，则返回对应的错误信息
    case CL_INVALID_BUFFER_SIZE:
        return "CL_INVALID_BUFFER_SIZE";
    # 如果发生 CL_INVALID_MIP_LEVEL 错误，则返回对应的错误信息
    case CL_INVALID_MIP_LEVEL:
        return "CL_INVALID_MIP_LEVEL";
    # 如果发生 CL_INVALID_GLOBAL_WORK_SIZE 错误，则返回对应的错误信息
    case CL_INVALID_GLOBAL_WORK_SIZE:
        return "CL_INVALID_GLOBAL_WORK_SIZE";
    # 如果发生 CL_INVALID_PROPERTY 错误，则返回对应的错误信息
    case CL_INVALID_PROPERTY:
        return "CL_INVALID_PROPERTY";
    # 如果发生 CL_INVALID_IMAGE_DESCRIPTOR 错误，则返回对应的错误信息
    case CL_INVALID_IMAGE_DESCRIPTOR:
        return "CL_INVALID_IMAGE_DESCRIPTOR";
    # 如果发生 CL_INVALID_COMPILER_OPTIONS 错误，则返回对应的错误信息
    case CL_INVALID_COMPILER_OPTIONS:
        return "CL_INVALID_COMPILER_OPTIONS";
    # 如果发生 CL_INVALID_LINKER_OPTIONS 错误，则返回对应的错误信息
    case CL_INVALID_LINKER_OPTIONS:
        return "CL_INVALID_LINKER_OPTIONS";
    # 如果发生 CL_INVALID_DEVICE_PARTITION_COUNT 错误，则返回对应的错误信息
    case CL_INVALID_DEVICE_PARTITION_COUNT:
        return "CL_INVALID_DEVICE_PARTITION_COUNT";
#ifdef CL_VERSION_2_0
    // 如果 OpenCL 版本为 2.0，则处理以下错误代码
    case CL_INVALID_PIPE_SIZE:
        // 如果错误代码为 CL_INVALID_PIPE_SIZE，则返回对应的错误信息
        return "CL_INVALID_PIPE_SIZE";
    case CL_INVALID_DEVICE_QUEUE:
        // 如果错误代码为 CL_INVALID_DEVICE_QUEUE，则返回对应的错误信息
        return "CL_INVALID_DEVICE_QUEUE";
#endif
    // 如果错误代码不在已知范围内，则返回未知错误信息
    default:
        return "UNKNOWN_ERROR";
    }
}
```