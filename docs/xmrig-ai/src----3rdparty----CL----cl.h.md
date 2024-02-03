# `xmrig\src\3rdparty\CL\cl.h`

```cpp
# 版权声明，允许任何人免费获取软件及相关文档，可以在不受限制的情况下处理这些材料，包括使用、复制、修改、合并、发布、分发、许可他人使用，并允许将材料提供给他人，但需要遵守以下条件
# 版权声明和许可声明必须包含在所有材料的所有副本或实质部分中
# 对此文件的修改可能意味着它不再准确反映 Khronos 标准。Khronos 规范和头文件的未修改的规范版本位于 https://www.khronos.org/registry/
# 材料按原样提供，不提供任何形式的担保，包括但不限于适销性、特定用途的适用性和非侵权性的担保。在任何情况下，作者或版权持有人对于任何索赔、损害或其他责任，无论是合同诉讼、侵权行为还是其他行为，均不承担责任，这些索赔、损害或其他责任是由于材料或材料的使用或其他交易引起的。
# 如果是 C++ 环境，则使用 extern "C" 进行声明
#ifndef __OPENCL_CL_H
#define __OPENCL_CL_H

# 包含 OpenCL 版本信息头文件
#include <CL/cl_version.h>
# 包含 OpenCL 平台信息头文件
#include <CL/cl_platform.h>

# 如果是 C++ 环境，则使用 extern "C" 进行声明
#ifdef __cplusplus
extern "C" {
#endif

/******************************************************************************/

# 定义 OpenCL 平台、设备、上下文、命令队列的数据类型
typedef struct _cl_platform_id *    cl_platform_id;
typedef struct _cl_device_id *      cl_device_id;
typedef struct _cl_context *        cl_context;
typedef struct _cl_command_queue *  cl_command_queue;
# 定义 OpenCL 中的内存对象、程序、内核、事件、采样器等类型
typedef struct _cl_mem *            cl_mem;
typedef struct _cl_program *        cl_program;
typedef struct _cl_kernel *         cl_kernel;
typedef struct _cl_event *          cl_event;
typedef struct _cl_sampler *        cl_sampler;

# 定义 OpenCL 中的布尔类型和位域类型
typedef cl_uint             cl_bool;                     /* WARNING!  Unlike cl_ types in cl_platform.h, cl_bool is not guaranteed to be the same size as the bool in kernels. */
typedef cl_ulong            cl_bitfield;
typedef cl_bitfield         cl_device_type;
typedef cl_uint             cl_platform_info;
typedef cl_uint             cl_device_info;
typedef cl_bitfield         cl_device_fp_config;
typedef cl_uint             cl_device_mem_cache_type;
typedef cl_uint             cl_device_local_mem_type;
typedef cl_bitfield         cl_device_exec_capabilities;
#ifdef CL_VERSION_2_0
typedef cl_bitfield         cl_device_svm_capabilities;
#endif
typedef cl_bitfield         cl_command_queue_properties;
#ifdef CL_VERSION_1_2
typedef intptr_t            cl_device_partition_property;
typedef cl_bitfield         cl_device_affinity_domain;
#endif

# 定义 OpenCL 中的上下文属性、上下文信息、命令队列信息等类型
typedef intptr_t            cl_context_properties;
typedef cl_uint             cl_context_info;
#ifdef CL_VERSION_2_0
typedef cl_bitfield         cl_queue_properties;
#endif
typedef cl_uint             cl_command_queue_info;
typedef cl_uint             cl_channel_order;
typedef cl_uint             cl_channel_type;
typedef cl_bitfield         cl_mem_flags;
#ifdef CL_VERSION_2_0
typedef cl_bitfield         cl_svm_mem_flags;
#endif
typedef cl_uint             cl_mem_object_type;
typedef cl_uint             cl_mem_info;
#ifdef CL_VERSION_1_2
typedef cl_bitfield         cl_mem_migration_flags;
#endif
typedef cl_uint             cl_image_info;
#ifdef CL_VERSION_1_1
typedef cl_uint             cl_buffer_create_type;
#endif
typedef cl_uint             cl_addressing_mode;
typedef cl_uint             cl_filter_mode;
typedef cl_uint             cl_sampler_info;
typedef cl_bitfield         cl_map_flags;
#ifdef CL_VERSION_2_0
// 定义 OpenCL 管道属性类型
typedef intptr_t            cl_pipe_properties;
// 定义 OpenCL 管道信息类型
typedef cl_uint             cl_pipe_info;
#endif
// 定义 OpenCL 程序信息类型
typedef cl_uint             cl_program_info;
// 定义 OpenCL 程序构建信息类型
typedef cl_uint             cl_program_build_info;
#ifdef CL_VERSION_1_2
// 定义 OpenCL 程序二进制类型
typedef cl_uint             cl_program_binary_type;
#endif
// 定义 OpenCL 构建状态类型
typedef cl_int              cl_build_status;
// 定义 OpenCL 内核信息类型
typedef cl_uint             cl_kernel_info;
#ifdef CL_VERSION_1_2
// 定义 OpenCL 内核参数信息类型
typedef cl_uint             cl_kernel_arg_info;
// 定义 OpenCL 内核参数地址限定符类型
typedef cl_uint             cl_kernel_arg_address_qualifier;
// 定义 OpenCL 内核参数访问限定符类型
typedef cl_uint             cl_kernel_arg_access_qualifier;
// 定义 OpenCL 内核参数类型限定符位域类型
typedef cl_bitfield         cl_kernel_arg_type_qualifier;
#endif
// 定义 OpenCL 内核工作组信息类型
typedef cl_uint             cl_kernel_work_group_info;
#ifdef CL_VERSION_2_1
// 定义 OpenCL 内核子组信息类型
typedef cl_uint             cl_kernel_sub_group_info;
#endif
// 定义 OpenCL 事件信息类型
typedef cl_uint             cl_event_info;
// 定义 OpenCL 命令类型
typedef cl_uint             cl_command_type;
// 定义 OpenCL 性能信息类型
typedef cl_uint             cl_profiling_info;
#ifdef CL_VERSION_2_0
// 定义 OpenCL 采样器属性类型
typedef cl_bitfield         cl_sampler_properties;
// 定义 OpenCL 内核执行信息类型
typedef cl_uint             cl_kernel_exec_info;
#endif

// 定义 OpenCL 图像格式结构体
typedef struct _cl_image_format {
    cl_channel_order        image_channel_order;
    cl_channel_type         image_channel_data_type;
} cl_image_format;

#ifdef CL_VERSION_1_2
// 定义 OpenCL 图像描述结构体
typedef struct _cl_image_desc {
    cl_mem_object_type      image_type;
    size_t                  image_width;
    size_t                  image_height;
    size_t                  image_depth;
    size_t                  image_array_size;
    size_t                  image_row_pitch;
    size_t                  image_slice_pitch;
    cl_uint                 num_mip_levels;
    cl_uint                 num_samples;
#ifdef CL_VERSION_2_0
#ifdef __GNUC__
    __extension__   /* Prevents warnings about anonymous union in -pedantic builds */
#endif
#ifdef _MSC_VER
#pragma warning( push )
#pragma warning( disable : 4201 ) /* Prevents warning about nameless struct/union in /W4 /Za builds */
#endif
    union {
#endif
      cl_mem                  buffer;
#ifdef CL_VERSION_2_0
      cl_mem                  mem_object;
    };
#ifdef _MSC_VER
#pragma warning( pop )
#endif
#endif
} cl_image_desc;

#endif

#ifdef CL_VERSION_1_1

typedef struct _cl_buffer_region {
    size_t                  origin;
    size_t                  size;
} cl_buffer_region;

#endif

这部分代码定义了一些结构体和宏，用于描述 OpenCL 中的内存对象和缓冲区区域。


/******************************************************************************/

/* Error Codes */
#define CL_SUCCESS                                  0
#define CL_DEVICE_NOT_FOUND                         -1
#define CL_DEVICE_NOT_AVAILABLE                     -2
#define CL_COMPILER_NOT_AVAILABLE                   -3
#define CL_MEM_OBJECT_ALLOCATION_FAILURE            -4
#define CL_OUT_OF_RESOURCES                         -5
#define CL_OUT_OF_HOST_MEMORY                       -6
#define CL_PROFILING_INFO_NOT_AVAILABLE             -7
#define CL_MEM_COPY_OVERLAP                         -8
#define CL_IMAGE_FORMAT_MISMATCH                    -9
#define CL_IMAGE_FORMAT_NOT_SUPPORTED               -10
#define CL_BUILD_PROGRAM_FAILURE                    -11
#define CL_MAP_FAILURE                              -12
#ifdef CL_VERSION_1_1
#define CL_MISALIGNED_SUB_BUFFER_OFFSET             -13
#define CL_EXEC_STATUS_ERROR_FOR_EVENTS_IN_WAIT_LIST -14
#endif
#ifdef CL_VERSION_1_2
#define CL_COMPILE_PROGRAM_FAILURE                  -15
#define CL_LINKER_NOT_AVAILABLE                     -16
#define CL_LINK_PROGRAM_FAILURE                     -17
#define CL_DEVICE_PARTITION_FAILED                  -18
#define CL_KERNEL_ARG_INFO_NOT_AVAILABLE            -19
#endif

#define CL_INVALID_VALUE                            -30
#define CL_INVALID_DEVICE_TYPE                      -31
#define CL_INVALID_PLATFORM                         -32
#define CL_INVALID_DEVICE                           -33
#define CL_INVALID_CONTEXT                          -34
#define CL_INVALID_QUEUE_PROPERTIES                 -35
#define CL_INVALID_COMMAND_QUEUE                    -36

这部分代码定义了一系列的错误码，用于表示 OpenCL 中可能出现的各种错误情况。
// 定义 OpenCL 错误码，表示主机指针无效
#define CL_INVALID_HOST_PTR                         -37
// 定义 OpenCL 错误码，表示内存对象无效
#define CL_INVALID_MEM_OBJECT                       -38
// 定义 OpenCL 错误码，表示图像格式描述符无效
#define CL_INVALID_IMAGE_FORMAT_DESCRIPTOR          -39
// 定义 OpenCL 错误码，表示图像大小无效
#define CL_INVALID_IMAGE_SIZE                       -40
// 定义 OpenCL 错误码，表示采样器无效
#define CL_INVALID_SAMPLER                          -41
// 定义 OpenCL 错误码，表示二进制无效
#define CL_INVALID_BINARY                           -42
// 定义 OpenCL 错误码，表示构建选项无效
#define CL_INVALID_BUILD_OPTIONS                    -43
// 定义 OpenCL 错误码，表示程序无效
#define CL_INVALID_PROGRAM                          -44
// 定义 OpenCL 错误码，表示可执行程序无效
#define CL_INVALID_PROGRAM_EXECUTABLE               -45
// 定义 OpenCL 错误码，表示内核名称无效
#define CL_INVALID_KERNEL_NAME                      -46
// 定义 OpenCL 错误码，表示内核定义无效
#define CL_INVALID_KERNEL_DEFINITION                -47
// 定义 OpenCL 错误码，表示内核无效
#define CL_INVALID_KERNEL                           -48
// 定义 OpenCL 错误码，表示参数索引无效
#define CL_INVALID_ARG_INDEX                        -49
// 定义 OpenCL 错误码，表示参数值无效
#define CL_INVALID_ARG_VALUE                        -50
// 定义 OpenCL 错误码，表示参数大小无效
#define CL_INVALID_ARG_SIZE                         -51
// 定义 OpenCL 错误码，表示内核参数无效
#define CL_INVALID_KERNEL_ARGS                      -52
// 定义 OpenCL 错误码，表示工作维度无效
#define CL_INVALID_WORK_DIMENSION                   -53
// 定义 OpenCL 错误码，表示工作组大小无效
#define CL_INVALID_WORK_GROUP_SIZE                  -54
// 定义 OpenCL 错误码，表示工作项大小无效
#define CL_INVALID_WORK_ITEM_SIZE                   -55
// 定义 OpenCL 错误码，表示全局偏移无效
#define CL_INVALID_GLOBAL_OFFSET                    -56
// 定义 OpenCL 错误码，表示事件等待列表无效
#define CL_INVALID_EVENT_WAIT_LIST                  -57
// 定义 OpenCL 错误码，表示事件无效
#define CL_INVALID_EVENT                            -58
// 定义 OpenCL 错误码，表示操作无效
#define CL_INVALID_OPERATION                        -59
// 定义 OpenCL 错误码，表示 OpenGL 对象无效
#define CL_INVALID_GL_OBJECT                        -60
// 定义 OpenCL 错误码，表示缓冲区大小无效
#define CL_INVALID_BUFFER_SIZE                      -61
// 定义 OpenCL 错误码，表示 MIP 层级无效
#define CL_INVALID_MIP_LEVEL                        -62
// 定义 OpenCL 错误码，表示全局工作大小无效
#define CL_INVALID_GLOBAL_WORK_SIZE                 -63
#ifdef CL_VERSION_1_1
// 定义 OpenCL 错误码，表示属性无效
#define CL_INVALID_PROPERTY                         -64
#endif
#ifdef CL_VERSION_1_2
// 定义 OpenCL 错误码，表示图像描述符无效
#define CL_INVALID_IMAGE_DESCRIPTOR                 -65
// 定义 OpenCL 错误码，表示编译器选项无效
#define CL_INVALID_COMPILER_OPTIONS                 -66
// 定义 OpenCL 错误码，表示链接器选项无效
#define CL_INVALID_LINKER_OPTIONS                   -67
// 定义 OpenCL 错误码，表示设备分区计数无效
#define CL_INVALID_DEVICE_PARTITION_COUNT           -68
#endif
#ifdef CL_VERSION_2_0
// 定义 OpenCL 错误码，表示管道大小无效
#define CL_INVALID_PIPE_SIZE                        -69
// 定义 OpenCL 错误码，表示设备队列无效
#define CL_INVALID_DEVICE_QUEUE                     -70
#endif
#ifdef CL_VERSION_2_2
// 如果 OpenCL 版本是 2.2，则定义无效规范 ID 为 -71
#define CL_INVALID_SPEC_ID                          -71
// 如果 OpenCL 版本是 2.2，则定义超出最大尺寸限制为 -72
#define CL_MAX_SIZE_RESTRICTION_EXCEEDED            -72
#endif


/* cl_bool */
// 定义 OpenCL 中的布尔类型 CL_FALSE 为 0
#define CL_FALSE                                    0
// 定义 OpenCL 中的布尔类型 CL_TRUE 为 1
#define CL_TRUE                                     1
#ifdef CL_VERSION_1_2
// 如果 OpenCL 版本是 1.2，则定义阻塞操作为 CL_TRUE
#define CL_BLOCKING                                 CL_TRUE
// 如果 OpenCL 版本是 1.2，则定义非阻塞操作为 CL_FALSE
#define CL_NON_BLOCKING                             CL_FALSE
#endif

/* cl_platform_info */
// 定义平台信息中的配置文件为 0x0900
#define CL_PLATFORM_PROFILE                         0x0900
// 定义平台信息中的版本号为 0x0901
#define CL_PLATFORM_VERSION                         0x0901
// 定义平台信息中的名称为 0x0902
#define CL_PLATFORM_NAME                            0x0902
// 定义平台信息中的供应商为 0x0903
#define CL_PLATFORM_VENDOR                          0x0903
// 定义平台信息中的扩展为 0x0904
#define CL_PLATFORM_EXTENSIONS                      0x0904
#ifdef CL_VERSION_2_1
// 如果 OpenCL 版本是 2.1，则定义主机计时器分辨率为 0x0905
#define CL_PLATFORM_HOST_TIMER_RESOLUTION           0x0905
#endif

/* cl_device_type - bitfield */
// 定义设备类型中的默认类型为 1 左移 0 位
#define CL_DEVICE_TYPE_DEFAULT                      (1 << 0)
// 定义设备类型中的 CPU 类型为 1 左移 1 位
#define CL_DEVICE_TYPE_CPU                          (1 << 1)
// 定义设备类型中的 GPU 类型为 1 左移 2 位
#define CL_DEVICE_TYPE_GPU                          (1 << 2)
// 定义设备类型中的加速器类型为 1 左移 3 位
#define CL_DEVICE_TYPE_ACCELERATOR                  (1 << 3)
#ifdef CL_VERSION_1_2
// 如果 OpenCL 版本是 1.2，则定义自定义设备类型为 1 左移 4 位
#define CL_DEVICE_TYPE_CUSTOM                       (1 << 4)
#endif
// 定义设备类型中的所有类型为 0xFFFFFFFF
#define CL_DEVICE_TYPE_ALL                          0xFFFFFFFF

/* cl_device_info */
// 定义设备信息中的设备类型为 0x1000
#define CL_DEVICE_TYPE                                   0x1000
// 定义设备信息中的供应商 ID 为 0x1001
#define CL_DEVICE_VENDOR_ID                              0x1001
// 定义设备信息中的最大计算单元数为 0x1002
#define CL_DEVICE_MAX_COMPUTE_UNITS                      0x1002
// 定义设备信息中的最大工作项维度为 0x1003
#define CL_DEVICE_MAX_WORK_ITEM_DIMENSIONS               0x1003
// 定义设备信息中的最大工作组大小为 0x1004
#define CL_DEVICE_MAX_WORK_GROUP_SIZE                    0x1004
// 定义设备信息中的最大工作项尺寸为 0x1005
#define CL_DEVICE_MAX_WORK_ITEM_SIZES                    0x1005
// 定义设备信息中的首选字符向量宽度为 0x1006
#define CL_DEVICE_PREFERRED_VECTOR_WIDTH_CHAR            0x1006
// 定义设备信息中的首选短整型向量宽度为 0x1007
#define CL_DEVICE_PREFERRED_VECTOR_WIDTH_SHORT           0x1007
// 定义设备信息中的首选整型向量宽度为 0x1008
#define CL_DEVICE_PREFERRED_VECTOR_WIDTH_INT             0x1008
// 定义设备信息中的首选长整型向量宽度为 0x1009
#define CL_DEVICE_PREFERRED_VECTOR_WIDTH_LONG            0x1009
// 定义设备信息中的首选浮点型向量宽度为 0x100A
#define CL_DEVICE_PREFERRED_VECTOR_WIDTH_FLOAT           0x100A
# 定义 OpenCL 设备属性的常量值
#define CL_DEVICE_PREFERRED_VECTOR_WIDTH_DOUBLE          0x100B  // 双精度浮点向量的首选宽度
#define CL_DEVICE_MAX_CLOCK_FREQUENCY                    0x100C  // 设备的最大时钟频率
#define CL_DEVICE_ADDRESS_BITS                           0x100D  // 设备的地址位数
#define CL_DEVICE_MAX_READ_IMAGE_ARGS                    0x100E  // 读取图像的最大参数个数
#define CL_DEVICE_MAX_WRITE_IMAGE_ARGS                   0x100F  // 写入图像的最大参数个数
#define CL_DEVICE_MAX_MEM_ALLOC_SIZE                     0x1010  // 单个内存分配的最大尺寸
#define CL_DEVICE_IMAGE2D_MAX_WIDTH                      0x1011  // 2D 图像的最大宽度
#define CL_DEVICE_IMAGE2D_MAX_HEIGHT                     0x1012  // 2D 图像的最大高度
#define CL_DEVICE_IMAGE3D_MAX_WIDTH                      0x1013  // 3D 图像的最大宽度
#define CL_DEVICE_IMAGE3D_MAX_HEIGHT                     0x1014  // 3D 图像的最大高度
#define CL_DEVICE_IMAGE3D_MAX_DEPTH                      0x1015  // 3D 图像的最大深度
#define CL_DEVICE_IMAGE_SUPPORT                          0x1016  // 设备是否支持图像
#define CL_DEVICE_MAX_PARAMETER_SIZE                     0x1017  // 内核参数的最大尺寸
#define CL_DEVICE_MAX_SAMPLERS                           0x1018  // 设备支持的最大采样器数量
#define CL_DEVICE_MEM_BASE_ADDR_ALIGN                    0x1019  // 内存基地址对齐要求
#define CL_DEVICE_MIN_DATA_TYPE_ALIGN_SIZE               0x101A  // 数据类型的最小对齐尺寸
#define CL_DEVICE_SINGLE_FP_CONFIG                       0x101B  // 单精度浮点运算的配置
#define CL_DEVICE_GLOBAL_MEM_CACHE_TYPE                  0x101C  // 全局内存缓存类型
#define CL_DEVICE_GLOBAL_MEM_CACHELINE_SIZE              0x101D  // 全局内存缓存行大小
#define CL_DEVICE_GLOBAL_MEM_CACHE_SIZE                  0x101E  // 全局内存缓存大小
#define CL_DEVICE_GLOBAL_MEM_SIZE                        0x101F  // 全局内存大小
#define CL_DEVICE_MAX_CONSTANT_BUFFER_SIZE               0x1020  // 常量缓冲区的最大尺寸
#define CL_DEVICE_MAX_CONSTANT_ARGS                      0x1021  // 内核中的常量参数的最大数量
#define CL_DEVICE_LOCAL_MEM_TYPE                         0x1022  // 本地内存类型
#define CL_DEVICE_LOCAL_MEM_SIZE                         0x1023  // 本地内存大小
#define CL_DEVICE_ERROR_CORRECTION_SUPPORT               0x1024  // 设备是否支持错误纠正
#define CL_DEVICE_PROFILING_TIMER_RESOLUTION             0x1025  // 设备的性能计时器分辨率
#define CL_DEVICE_ENDIAN_LITTLE                          0x1026  // 设备是否为小端字节序
#define CL_DEVICE_AVAILABLE                              0x1027  // 设备是否可用
#define CL_DEVICE_COMPILER_AVAILABLE                     0x1028  // 设备是否支持编译器
#define CL_DEVICE_EXECUTION_CAPABILITIES                 0x1029  // 设备的执行能力
# 定义 OpenCL 设备属性的常量值
#define CL_DEVICE_QUEUE_PROPERTIES                       0x102A    /* deprecated */  # 队列属性，已废弃
#ifdef CL_VERSION_2_0
#define CL_DEVICE_QUEUE_ON_HOST_PROPERTIES               0x102A  # 如果 OpenCL 版本大于等于 2.0，则定义队列在主机上的属性
#endif
#define CL_DEVICE_NAME                                   0x102B  # 设备名称
#define CL_DEVICE_VENDOR                                 0x102C  # 设备供应商
#define CL_DRIVER_VERSION                                0x102D  # 驱动程序版本
#define CL_DEVICE_PROFILE                                0x102E  # 设备的配置文件
#define CL_DEVICE_VERSION                                0x102F  # 设备版本
#define CL_DEVICE_EXTENSIONS                             0x1030  # 设备支持的扩展
#define CL_DEVICE_PLATFORM                               0x1031  # 设备所在平台
#ifdef CL_VERSION_1_2
#define CL_DEVICE_DOUBLE_FP_CONFIG                       0x1032  # 双精度浮点配置
#endif
/* 0x1033 reserved for CL_DEVICE_HALF_FP_CONFIG which is already defined in "cl_ext.h" */
#ifdef CL_VERSION_1_1
#define CL_DEVICE_PREFERRED_VECTOR_WIDTH_HALF            0x1034  # 首选半精度向量宽度
#define CL_DEVICE_HOST_UNIFIED_MEMORY                    0x1035   /* deprecated */  # 主机统一内存，已废弃
#define CL_DEVICE_NATIVE_VECTOR_WIDTH_CHAR               0x1036  # 本地字符向量宽度
#define CL_DEVICE_NATIVE_VECTOR_WIDTH_SHORT              0x1037  # 本地短整型向量宽度
#define CL_DEVICE_NATIVE_VECTOR_WIDTH_INT                0x1038  # 本地整型向量宽度
#define CL_DEVICE_NATIVE_VECTOR_WIDTH_LONG               0x1039  # 本地长整型向量宽度
#define CL_DEVICE_NATIVE_VECTOR_WIDTH_FLOAT              0x103A  # 本地浮点型向量宽度
#define CL_DEVICE_NATIVE_VECTOR_WIDTH_DOUBLE             0x103B  # 本地双精度浮点型向量宽度
#define CL_DEVICE_NATIVE_VECTOR_WIDTH_HALF               0x103C  # 本地半精度浮点型向量宽度
#define CL_DEVICE_OPENCL_C_VERSION                       0x103D  # OpenCL C 版本
#endif
#ifdef CL_VERSION_1_2
#define CL_DEVICE_LINKER_AVAILABLE                       0x103E  # 链接器是否可用
#define CL_DEVICE_BUILT_IN_KERNELS                       0x103F  # 内置的内核
#define CL_DEVICE_IMAGE_MAX_BUFFER_SIZE                  0x1040  # 图像最大缓冲区大小
#define CL_DEVICE_IMAGE_MAX_ARRAY_SIZE                   0x1041  # 图像最大数组大小
#define CL_DEVICE_PARENT_DEVICE                          0x1042  # 父设备
#define CL_DEVICE_PARTITION_MAX_SUB_DEVICES              0x1043  # 分区的最大子设备数
#define CL_DEVICE_PARTITION_PROPERTIES                   0x1044  # 分区属性
// 定义 OpenCL 设备的分区亲和域
#define CL_DEVICE_PARTITION_AFFINITY_DOMAIN              0x1045
// 定义 OpenCL 设备的分区类型
#define CL_DEVICE_PARTITION_TYPE                         0x1046
// 定义 OpenCL 设备的引用计数
#define CL_DEVICE_REFERENCE_COUNT                        0x1047
// 定义 OpenCL 设备的首选互操作用户同步
#define CL_DEVICE_PREFERRED_INTEROP_USER_SYNC            0x1048
// 定义 OpenCL 设备的 printf 缓冲区大小
#define CL_DEVICE_PRINTF_BUFFER_SIZE                     0x1049
#endif
#ifdef CL_VERSION_2_0
// 定义 OpenCL 设备的图像间距对齐
#define CL_DEVICE_IMAGE_PITCH_ALIGNMENT                  0x104A
// 定义 OpenCL 设备的图像基地址对齐
#define CL_DEVICE_IMAGE_BASE_ADDRESS_ALIGNMENT           0x104B
// 定义 OpenCL 设备的最大读写图像参数
#define CL_DEVICE_MAX_READ_WRITE_IMAGE_ARGS              0x104C
// 定义 OpenCL 设备的最大全局变量大小
#define CL_DEVICE_MAX_GLOBAL_VARIABLE_SIZE               0x104D
// 定义 OpenCL 设备的设备上队列属性
#define CL_DEVICE_QUEUE_ON_DEVICE_PROPERTIES             0x104E
// 定义 OpenCL 设备的设备上首选队列大小
#define CL_DEVICE_QUEUE_ON_DEVICE_PREFERRED_SIZE         0x104F
// 定义 OpenCL 设备的设备上最大队列大小
#define CL_DEVICE_QUEUE_ON_DEVICE_MAX_SIZE               0x1050
// 定义 OpenCL 设备的最大设备队列数
#define CL_DEVICE_MAX_ON_DEVICE_QUEUES                   0x1051
// 定义 OpenCL 设备的最大设备事件数
#define CL_DEVICE_MAX_ON_DEVICE_EVENTS                   0x1052
// 定义 OpenCL 设备的 SVM 能力
#define CL_DEVICE_SVM_CAPABILITIES                       0x1053
// 定义 OpenCL 设备的首选全局变量总大小
#define CL_DEVICE_GLOBAL_VARIABLE_PREFERRED_TOTAL_SIZE   0x1054
// 定义 OpenCL 设备的最大管道参数
#define CL_DEVICE_MAX_PIPE_ARGS                          0x1055
// 定义 OpenCL 设备的最大活动管道预留
#define CL_DEVICE_PIPE_MAX_ACTIVE_RESERVATIONS           0x1056
// 定义 OpenCL 设备的最大管道数据包大小
#define CL_DEVICE_PIPE_MAX_PACKET_SIZE                   0x1057
// 定义 OpenCL 设备的首选平台原子对齐
#define CL_DEVICE_PREFERRED_PLATFORM_ATOMIC_ALIGNMENT    0x1058
// 定义 OpenCL 设备的首选全局原子对齐
#define CL_DEVICE_PREFERRED_GLOBAL_ATOMIC_ALIGNMENT      0x1059
// 定义 OpenCL 设备的首选本地原子对齐
#define CL_DEVICE_PREFERRED_LOCAL_ATOMIC_ALIGNMENT       0x105A
#endif
#ifdef CL_VERSION_2_1
// 定义 OpenCL 设备的 IL 版本
#define CL_DEVICE_IL_VERSION                             0x105B
// 定义 OpenCL 设备的最大子组数
#define CL_DEVICE_MAX_NUM_SUB_GROUPS                     0x105C
// 定义 OpenCL 设备的子组独立前进进度
#define CL_DEVICE_SUB_GROUP_INDEPENDENT_FORWARD_PROGRESS 0x105D
#endif

/* cl_device_fp_config - 位域 */
// 定义 OpenCL 设备的浮点配置 - 无规格化数
#define CL_FP_DENORM                                (1 << 0)
// 定义 OpenCL 设备的浮点配置 - 无穷大和非数
#define CL_FP_INF_NAN                               (1 << 1)
// 定义 OpenCL 设备的浮点配置 - 四舍五入到最近
#define CL_FP_ROUND_TO_NEAREST                      (1 << 2)
// 定义 OpenCL 设备的浮点配置 - 四舍五入到零
#define CL_FP_ROUND_TO_ZERO                         (1 << 3)
#define CL_FP_ROUND_TO_INF                          (1 << 4)  // 定义浮点数舍入到无穷大的模式
#define CL_FP_FMA                                   (1 << 5)  // 定义支持浮点数乘加运算
#ifdef CL_VERSION_1_1
#define CL_FP_SOFT_FLOAT                            (1 << 6)  // 如果支持软件浮点数运算，则定义为1
#endif
#ifdef CL_VERSION_1_2
#define CL_FP_CORRECTLY_ROUNDED_DIVIDE_SQRT         (1 << 7)  // 定义支持正确舍入的除法和平方根运算
#endif

/* cl_device_mem_cache_type */
#define CL_NONE                                     0x0  // 定义无缓存
#define CL_READ_ONLY_CACHE                          0x1  // 定义只读缓存
#define CL_READ_WRITE_CACHE                         0x2  // 定义读写缓存

/* cl_device_local_mem_type */
#define CL_LOCAL                                    0x1  // 定义本地内存
#define CL_GLOBAL                                   0x2  // 定义全局内存

/* cl_device_exec_capabilities - bitfield */
#define CL_EXEC_KERNEL                              (1 << 0)  // 定义支持执行内核
#define CL_EXEC_NATIVE_KERNEL                       (1 << 1)  // 定义支持本地内核执行

/* cl_command_queue_properties - bitfield */
#define CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE      (1 << 0)  // 定义支持乱序执行模式
#define CL_QUEUE_PROFILING_ENABLE                   (1 << 1)  // 定义支持性能分析
#ifdef CL_VERSION_2_0
#define CL_QUEUE_ON_DEVICE                          (1 << 2)  // 定义支持在设备上执行命令队列
#define CL_QUEUE_ON_DEVICE_DEFAULT                  (1 << 3)  // 定义设备默认的命令队列
#endif

/* cl_context_info */
#define CL_CONTEXT_REFERENCE_COUNT                  0x1080  // 定义上下文的引用计数
#define CL_CONTEXT_DEVICES                          0x1081  // 定义上下文的设备
#define CL_CONTEXT_PROPERTIES                       0x1082  // 定义上下文的属性
#ifdef CL_VERSION_1_1
#define CL_CONTEXT_NUM_DEVICES                      0x1083  // 定义上下文的设备数量
#endif

/* cl_context_properties */
#define CL_CONTEXT_PLATFORM                         0x1084  // 定义上下文的平台
#ifdef CL_VERSION_1_2
#define CL_CONTEXT_INTEROP_USER_SYNC                0x1085  // 定义上下文的用户同步
#endif

#ifdef CL_VERSION_1_2

/* cl_device_partition_property */
#define CL_DEVICE_PARTITION_EQUALLY                 0x1086  // 定义设备分区均匀
#define CL_DEVICE_PARTITION_BY_COUNTS               0x1087  // 定义设备按数量分区
#define CL_DEVICE_PARTITION_BY_COUNTS_LIST_END      0x0  // 定义设备分区数量列表结束
#define CL_DEVICE_PARTITION_BY_AFFINITY_DOMAIN      0x1088  // 定义设备按亲和域分区

#endif

#ifdef CL_VERSION_1_2

/* cl_device_affinity_domain */
#define CL_DEVICE_AFFINITY_DOMAIN_NUMA               (1 << 0)  // 表示设备支持 NUMA 亲和性域
#define CL_DEVICE_AFFINITY_DOMAIN_L4_CACHE           (1 << 1)  // 表示设备支持 L4 缓存亲和性域
#define CL_DEVICE_AFFINITY_DOMAIN_L3_CACHE           (1 << 2)  // 表示设备支持 L3 缓存亲和性域
#define CL_DEVICE_AFFINITY_DOMAIN_L2_CACHE           (1 << 3)  // 表示设备支持 L2 缓存亲和性域
#define CL_DEVICE_AFFINITY_DOMAIN_L1_CACHE           (1 << 4)  // 表示设备支持 L1 缓存亲和性域
#define CL_DEVICE_AFFINITY_DOMAIN_NEXT_PARTITIONABLE (1 << 5)  // 表示设备支持下一个可分区的亲和性域

#endif

#ifdef CL_VERSION_2_0

/* cl_device_svm_capabilities */
#define CL_DEVICE_SVM_COARSE_GRAIN_BUFFER           (1 << 0)  // 表示设备支持粗粒度 SVM 缓冲区
#define CL_DEVICE_SVM_FINE_GRAIN_BUFFER             (1 << 1)  // 表示设备支持细粒度 SVM 缓冲区
#define CL_DEVICE_SVM_FINE_GRAIN_SYSTEM             (1 << 2)  // 表示设备支持细粒度 SVM 系统
#define CL_DEVICE_SVM_ATOMICS                       (1 << 3)  // 表示设备支持 SVM 原子操作

#endif

/* cl_command_queue_info */
#define CL_QUEUE_CONTEXT                            0x1090  // 表示命令队列的上下文信息
#define CL_QUEUE_DEVICE                             0x1091  // 表示命令队列的设备信息
#define CL_QUEUE_REFERENCE_COUNT                    0x1092  // 表示命令队列的引用计数信息
#define CL_QUEUE_PROPERTIES                         0x1093  // 表示命令队列的属性信息
#ifdef CL_VERSION_2_0
#define CL_QUEUE_SIZE                               0x1094  // 表示命令队列的大小信息
#endif
#ifdef CL_VERSION_2_1
#define CL_QUEUE_DEVICE_DEFAULT                     0x1095  // 表示命令队列的默认设备信息
#endif

/* cl_mem_flags and cl_svm_mem_flags - bitfield */
#define CL_MEM_READ_WRITE                           (1 << 0)  // 表示内存对象可读可写
#define CL_MEM_WRITE_ONLY                           (1 << 1)  // 表示内存对象只可写
#define CL_MEM_READ_ONLY                            (1 << 2)  // 表示内存对象只可读
#define CL_MEM_USE_HOST_PTR                         (1 << 3)  // 表示使用主机指针
#define CL_MEM_ALLOC_HOST_PTR                       (1 << 4)  // 表示分配主机指针
#define CL_MEM_COPY_HOST_PTR                        (1 << 5)  // 表示复制主机指针
/* reserved                                         (1 << 6)    */
#ifdef CL_VERSION_1_2
#define CL_MEM_HOST_WRITE_ONLY                      (1 << 7)  // 表示主机只可写
#define CL_MEM_HOST_READ_ONLY                       (1 << 8)  // 表示主机只可读
#define CL_MEM_HOST_NO_ACCESS                       (1 << 9)  // 表示主机无法访问
#endif
#ifdef CL_VERSION_2_0
#define CL_MEM_SVM_FINE_GRAIN_BUFFER                (1 << 10)  // 表示细粒度 SVM 缓冲区
#define CL_MEM_SVM_ATOMICS                          (1 << 11)   /* 用于 cl_svm_mem_flags */
#define CL_MEM_KERNEL_READ_AND_WRITE                (1 << 12)
#endif

#ifdef CL_VERSION_1_2

/* cl_mem_migration_flags - 位字段 */
#define CL_MIGRATE_MEM_OBJECT_HOST                  (1 << 0)
#define CL_MIGRATE_MEM_OBJECT_CONTENT_UNDEFINED     (1 << 1)

#endif

/* cl_channel_order - 通道顺序 */
#define CL_R                                        0x10B0
#define CL_A                                        0x10B1
#define CL_RG                                       0x10B2
#define CL_RA                                       0x10B3
#define CL_RGB                                      0x10B4
#define CL_RGBA                                     0x10B5
#define CL_BGRA                                     0x10B6
#define CL_ARGB                                     0x10B7
#define CL_INTENSITY                                0x10B8
#define CL_LUMINANCE                                0x10B9
#ifdef CL_VERSION_1_1
#define CL_Rx                                       0x10BA
#define CL_RGx                                      0x10BB
#define CL_RGBx                                     0x10BC
#endif
#ifdef CL_VERSION_1_2
#define CL_DEPTH                                    0x10BD
#define CL_DEPTH_STENCIL                            0x10BE
#endif
#ifdef CL_VERSION_2_0
#define CL_sRGB                                     0x10BF
#define CL_sRGBx                                    0x10C0
#define CL_sRGBA                                    0x10C1
#define CL_sBGRA                                    0x10C2
#define CL_ABGR                                     0x10C3
#endif

/* cl_channel_type - 通道类型 */
#define CL_SNORM_INT8                               0x10D0
#define CL_SNORM_INT16                              0x10D1
#define CL_UNORM_INT8                               0x10D2
#define CL_UNORM_INT16                              0x10D3
#define CL_UNORM_SHORT_565                          0x10D4
# 定义 OpenCL 中的规范化无符号短整型 5-5-5
#define CL_UNORM_SHORT_555                          0x10D5
# 定义 OpenCL 中的规范化无符号整型 10-10-10
#define CL_UNORM_INT_101010                         0x10D6
# 定义 OpenCL 中的有符号 8 位整型
#define CL_SIGNED_INT8                              0x10D7
# 定义 OpenCL 中的有符号 16 位整型
#define CL_SIGNED_INT16                             0x10D8
# 定义 OpenCL 中的有符号 32 位整型
#define CL_SIGNED_INT32                             0x10D9
# 定义 OpenCL 中的无符号 8 位整型
#define CL_UNSIGNED_INT8                            0x10DA
# 定义 OpenCL 中的无符号 16 位整型
#define CL_UNSIGNED_INT16                           0x10DB
# 定义 OpenCL 中的无符号 32 位整型
#define CL_UNSIGNED_INT32                           0x10DC
# 定义 OpenCL 中的半精度浮点数
#define CL_HALF_FLOAT                               0x10DD
# 定义 OpenCL 中的单精度浮点数
#define CL_FLOAT                                    0x10DE
#ifdef CL_VERSION_1_2
# 如果 OpenCL 版本为 1.2，则定义规范化无符号整型 24 位
#define CL_UNORM_INT24                              0x10DF
#endif
#ifdef CL_VERSION_2_1
# 如果 OpenCL 版本为 2.1，则定义规范化无符号整型 10-10-10（另一种格式）
#define CL_UNORM_INT_101010_2                       0x10E0
#endif

/* cl_mem_object_type */
# 定义 OpenCL 中的内存对象类型为缓冲区
#define CL_MEM_OBJECT_BUFFER                        0x10F0
# 定义 OpenCL 中的内存对象类型为二维图像
#define CL_MEM_OBJECT_IMAGE2D                       0x10F1
# 定义 OpenCL 中的内存对象类型为三维图像
#define CL_MEM_OBJECT_IMAGE3D                       0x10F2
#ifdef CL_VERSION_1_2
# 如果 OpenCL 版本为 1.2，则定义内存对象类型为二维图像数组
#define CL_MEM_OBJECT_IMAGE2D_ARRAY                 0x10F3
# 如果 OpenCL 版本为 1.2，则定义内存对象类型为一维图像
#define CL_MEM_OBJECT_IMAGE1D                       0x10F4
# 如果 OpenCL 版本为 1.2，则定义内存对象类型为一维图像数组
#define CL_MEM_OBJECT_IMAGE1D_ARRAY                 0x10F5
# 如果 OpenCL 版本为 1.2，则定义内存对象类型为一维图像缓冲区
#define CL_MEM_OBJECT_IMAGE1D_BUFFER                0x10F6
#endif
#ifdef CL_VERSION_2_0
# 如果 OpenCL 版本为 2.0，则定义内存对象类型为管道
#define CL_MEM_OBJECT_PIPE                          0x10F7
#endif

/* cl_mem_info */
# 定义 OpenCL 中的内存信息类型为内存类型
#define CL_MEM_TYPE                                 0x1100
# 定义 OpenCL 中的内存信息类型为内存标志
#define CL_MEM_FLAGS                                0x1101
# 定义 OpenCL 中的内存信息类型为内存大小
#define CL_MEM_SIZE                                 0x1102
# 定义 OpenCL 中的内存信息类型为主机指针
#define CL_MEM_HOST_PTR                             0x1103
# 定义 OpenCL 中的内存信息类型为映射计数
#define CL_MEM_MAP_COUNT                            0x1104
# 定义 OpenCL 中的内存信息类型为引用计数
#define CL_MEM_REFERENCE_COUNT                      0x1105
# 定义 OpenCL 中的内存信息类型为上下文
#define CL_MEM_CONTEXT                              0x1106
#ifdef CL_VERSION_1_1
# 如果 OpenCL 版本为 1.1，则定义关联内存对象
#define CL_MEM_ASSOCIATED_MEMOBJECT                 0x1107
# 如果 OpenCL 版本为 1.1，则定义内存偏移量
#define CL_MEM_OFFSET                               0x1108
#endif
#ifdef CL_VERSION_2_0
# 如果 OpenCL 版本为 2.0，则定义使用 SVM 指针的内存
#define CL_MEM_USES_SVM_POINTER                     0x1109
#endif
/* cl_image_info */
// 定义用于获取图像信息的枚举值

#define CL_IMAGE_FORMAT                             0x1110
// 图像格式

#define CL_IMAGE_ELEMENT_SIZE                       0x1111
// 图像元素大小

#define CL_IMAGE_ROW_PITCH                          0x1112
// 图像行间距

#define CL_IMAGE_SLICE_PITCH                        0x1113
// 图像切片间距

#define CL_IMAGE_WIDTH                              0x1114
// 图像宽度

#define CL_IMAGE_HEIGHT                             0x1115
// 图像高度

#define CL_IMAGE_DEPTH                              0x1116
// 图像深度

#ifdef CL_VERSION_1_2
#define CL_IMAGE_ARRAY_SIZE                         0x1117
// 图像数组大小

#define CL_IMAGE_BUFFER                             0x1118
// 图像缓冲

#define CL_IMAGE_NUM_MIP_LEVELS                     0x1119
// 图像 MIP 层级数

#define CL_IMAGE_NUM_SAMPLES                        0x111A
// 图像采样数
#endif

#ifdef CL_VERSION_2_0

/* cl_pipe_info */
#define CL_PIPE_PACKET_SIZE                         0x1120
// 管道数据包大小

#define CL_PIPE_MAX_PACKETS                         0x1121
// 管道最大数据包数

#endif

/* cl_addressing_mode */
// 定义用于获取寻址模式的枚举值
#define CL_ADDRESS_NONE                             0x1130
// 无寻址模式

#define CL_ADDRESS_CLAMP_TO_EDGE                    0x1131
// 边缘夹取寻址模式

#define CL_ADDRESS_CLAMP                            0x1132
// 夹取寻址模式

#define CL_ADDRESS_REPEAT                           0x1133
// 重复寻址模式

#ifdef CL_VERSION_1_1
#define CL_ADDRESS_MIRRORED_REPEAT                  0x1134
// 镜像重复寻址模式
#endif

/* cl_filter_mode */
// 定义用于获取过滤模式的枚举值
#define CL_FILTER_NEAREST                           0x1140
// 最近邻过滤模式

#define CL_FILTER_LINEAR                            0x1141
// 线性过滤模式

/* cl_sampler_info */
// 定义用于获取采样器信息的枚举值
#define CL_SAMPLER_REFERENCE_COUNT                  0x1150
// 采样器引用计数

#define CL_SAMPLER_CONTEXT                          0x1151
// 采样器上下文

#define CL_SAMPLER_NORMALIZED_COORDS                0x1152
// 采样器是否使用归一化坐标

#define CL_SAMPLER_ADDRESSING_MODE                  0x1153
// 采样器寻址模式

#define CL_SAMPLER_FILTER_MODE                      0x1154
// 采样器过滤模式

#ifdef CL_VERSION_2_0
/* These enumerants are for the cl_khr_mipmap_image extension.
   They have since been added to cl_ext.h with an appropriate
   KHR suffix, but are left here for backwards compatibility. */
#define CL_SAMPLER_MIP_FILTER_MODE                  0x1155
// 采样器 MIP 过滤模式
#define CL_SAMPLER_LOD_MIN                          0x1156
#define CL_SAMPLER_LOD_MAX                          0x1157
#endif

/* cl_map_flags - bitfield */
#define CL_MAP_READ                                 (1 << 0)  // 读取标志
#define CL_MAP_WRITE                                (1 << 1)  // 写入标志
#ifdef CL_VERSION_1_2
#define CL_MAP_WRITE_INVALIDATE_REGION              (1 << 2)  // 写入并使区域无效标志
#endif

/* cl_program_info */
#define CL_PROGRAM_REFERENCE_COUNT                  0x1160  // 程序引用计数
#define CL_PROGRAM_CONTEXT                          0x1161  // 程序上下文
#define CL_PROGRAM_NUM_DEVICES                      0x1162  // 设备数量
#define CL_PROGRAM_DEVICES                          0x1163  // 设备列表
#define CL_PROGRAM_SOURCE                           0x1164  // 程序源码
#define CL_PROGRAM_BINARY_SIZES                     0x1165  // 二进制大小
#define CL_PROGRAM_BINARIES                         0x1166  // 二进制数据
#ifdef CL_VERSION_1_2
#define CL_PROGRAM_NUM_KERNELS                      0x1167  // 内核数量
#define CL_PROGRAM_KERNEL_NAMES                     0x1168  // 内核名称列表
#endif
#ifdef CL_VERSION_2_1
#define CL_PROGRAM_IL                               0x1169  // 中间语言
#endif
#ifdef CL_VERSION_2_2
#define CL_PROGRAM_SCOPE_GLOBAL_CTORS_PRESENT       0x116A  // 全局构造函数是否存在
#define CL_PROGRAM_SCOPE_GLOBAL_DTORS_PRESENT       0x116B  // 全局析构函数是否存在
#endif

/* cl_program_build_info */
#define CL_PROGRAM_BUILD_STATUS                     0x1181  // 构建状态
#define CL_PROGRAM_BUILD_OPTIONS                    0x1182  // 构建选项
#define CL_PROGRAM_BUILD_LOG                        0x1183  // 构建日志
#ifdef CL_VERSION_1_2
#define CL_PROGRAM_BINARY_TYPE                      0x1184  // 二进制类型
#endif
#ifdef CL_VERSION_2_0
#define CL_PROGRAM_BUILD_GLOBAL_VARIABLE_TOTAL_SIZE 0x1185  // 全局变量总大小
#endif

#ifdef CL_VERSION_1_2

/* cl_program_binary_type */
#define CL_PROGRAM_BINARY_TYPE_NONE                 0x0  // 无
#define CL_PROGRAM_BINARY_TYPE_COMPILED_OBJECT      0x1  // 已编译对象
#define CL_PROGRAM_BINARY_TYPE_LIBRARY              0x2  // 库
#define CL_PROGRAM_BINARY_TYPE_EXECUTABLE           0x4  // 可执行文件

#endif

/* cl_build_status */
#define CL_BUILD_SUCCESS                            0  // 构建成功
#define CL_BUILD_NONE                               -1  // 无构建
// 定义 OpenCL 编译错误和编译进行中的状态码
#define CL_BUILD_ERROR                              -2
#define CL_BUILD_IN_PROGRESS                        -3

// 定义内核信息
#define CL_KERNEL_FUNCTION_NAME                     0x1190
#define CL_KERNEL_NUM_ARGS                          0x1191
#define CL_KERNEL_REFERENCE_COUNT                   0x1192
#define CL_KERNEL_CONTEXT                           0x1193
#define CL_KERNEL_PROGRAM                           0x1194
#ifdef CL_VERSION_1_2
#define CL_KERNEL_ATTRIBUTES                        0x1195
#endif
#ifdef CL_VERSION_2_1
#define CL_KERNEL_MAX_NUM_SUB_GROUPS                0x11B9
#define CL_KERNEL_COMPILE_NUM_SUB_GROUPS            0x11BA
#endif

#ifdef CL_VERSION_1_2

// 定义内核参数信息
#define CL_KERNEL_ARG_ADDRESS_QUALIFIER             0x1196
#define CL_KERNEL_ARG_ACCESS_QUALIFIER              0x1197
#define CL_KERNEL_ARG_TYPE_NAME                     0x1198
#define CL_KERNEL_ARG_TYPE_QUALIFIER                0x1199
#define CL_KERNEL_ARG_NAME                          0x119A

#endif

#ifdef CL_VERSION_1_2

// 定义内核参数地址限定符
#define CL_KERNEL_ARG_ADDRESS_GLOBAL                0x119B
#define CL_KERNEL_ARG_ADDRESS_LOCAL                 0x119C
#define CL_KERNEL_ARG_ADDRESS_CONSTANT              0x119D
#define CL_KERNEL_ARG_ADDRESS_PRIVATE               0x119E

#endif

#ifdef CL_VERSION_1_2

// 定义内核参数访问限定符
#define CL_KERNEL_ARG_ACCESS_READ_ONLY              0x11A0
#define CL_KERNEL_ARG_ACCESS_WRITE_ONLY             0x11A1
#define CL_KERNEL_ARG_ACCESS_READ_WRITE             0x11A2
#define CL_KERNEL_ARG_ACCESS_NONE                   0x11A3

#endif

#ifdef CL_VERSION_1_2

// 定义内核参数类型限定符
#define CL_KERNEL_ARG_TYPE_NONE                     0
#define CL_KERNEL_ARG_TYPE_CONST                    (1 << 0)
#define CL_KERNEL_ARG_TYPE_RESTRICT                 (1 << 1)
#define CL_KERNEL_ARG_TYPE_VOLATILE                 (1 << 2)
#ifdef CL_VERSION_2_0
// 定义 CL_KERNEL_ARG_TYPE_PIPE 常量，值为 1 左移 3 位的结果
#define CL_KERNEL_ARG_TYPE_PIPE                     (1 << 3)
#endif

#endif

// 定义 CL_KERNEL_WORK_GROUP_SIZE 常量，值为 0x11B0
#define CL_KERNEL_WORK_GROUP_SIZE                   0x11B0
// 定义 CL_KERNEL_COMPILE_WORK_GROUP_SIZE 常量，值为 0x11B1
#define CL_KERNEL_COMPILE_WORK_GROUP_SIZE           0x11B1
// 定义 CL_KERNEL_LOCAL_MEM_SIZE 常量，值为 0x11B2
#define CL_KERNEL_LOCAL_MEM_SIZE                    0x11B2
// 定义 CL_KERNEL_PREFERRED_WORK_GROUP_SIZE_MULTIPLE 常量，值为 0x11B3
#define CL_KERNEL_PREFERRED_WORK_GROUP_SIZE_MULTIPLE 0x11B3
// 定义 CL_KERNEL_PRIVATE_MEM_SIZE 常量，值为 0x11B4
#define CL_KERNEL_PRIVATE_MEM_SIZE                  0x11B4
#ifdef CL_VERSION_1_2
// 定义 CL_KERNEL_GLOBAL_WORK_SIZE 常量，值为 0x11B5
#define CL_KERNEL_GLOBAL_WORK_SIZE                  0x11B5
#endif

#ifdef CL_VERSION_2_1

// 定义 CL_KERNEL_MAX_SUB_GROUP_SIZE_FOR_NDRANGE 常量，值为 0x2033
#define CL_KERNEL_MAX_SUB_GROUP_SIZE_FOR_NDRANGE    0x2033
// 定义 CL_KERNEL_SUB_GROUP_COUNT_FOR_NDRANGE 常量，值为 0x2034
#define CL_KERNEL_SUB_GROUP_COUNT_FOR_NDRANGE       0x2034
// 定义 CL_KERNEL_LOCAL_SIZE_FOR_SUB_GROUP_COUNT 常量，值为 0x11B8
#define CL_KERNEL_LOCAL_SIZE_FOR_SUB_GROUP_COUNT    0x11B8

#endif

#ifdef CL_VERSION_2_0

// 定义 CL_KERNEL_EXEC_INFO_SVM_PTRS 常量，值为 0x11B6
#define CL_KERNEL_EXEC_INFO_SVM_PTRS                0x11B6
// 定义 CL_KERNEL_EXEC_INFO_SVM_FINE_GRAIN_SYSTEM 常量，值为 0x11B7
#define CL_KERNEL_EXEC_INFO_SVM_FINE_GRAIN_SYSTEM   0x11B7

#endif

// 定义 CL_EVENT_COMMAND_QUEUE 常量，值为 0x11D0
#define CL_EVENT_COMMAND_QUEUE                      0x11D0
// 定义 CL_EVENT_COMMAND_TYPE 常量，值为 0x11D1
#define CL_EVENT_COMMAND_TYPE                       0x11D1
// 定义 CL_EVENT_REFERENCE_COUNT 常量，值为 0x11D2
#define CL_EVENT_REFERENCE_COUNT                    0x11D2
// 定义 CL_EVENT_COMMAND_EXECUTION_STATUS 常量，值为 0x11D3
#define CL_EVENT_COMMAND_EXECUTION_STATUS           0x11D3
#ifdef CL_VERSION_1_1
// 定义 CL_EVENT_CONTEXT 常量，值为 0x11D4
#define CL_EVENT_CONTEXT                            0x11D4
#endif

// 定义不同类型的命令常量
#define CL_COMMAND_NDRANGE_KERNEL                   0x11F0
#define CL_COMMAND_TASK                             0x11F1
#define CL_COMMAND_NATIVE_KERNEL                    0x11F2
#define CL_COMMAND_READ_BUFFER                      0x11F3
#define CL_COMMAND_WRITE_BUFFER                     0x11F4
#define CL_COMMAND_COPY_BUFFER                      0x11F5
#define CL_COMMAND_READ_IMAGE                       0x11F6
#define CL_COMMAND_WRITE_IMAGE                      0x11F7
#define CL_COMMAND_COPY_IMAGE                       0x11F8
#define CL_COMMAND_COPY_IMAGE_TO_BUFFER             0x11F9
#define CL_COMMAND_COPY_BUFFER_TO_IMAGE             0x11FA
#define CL_COMMAND_MAP_BUFFER                       0x11FB
#define CL_COMMAND_MAP_IMAGE                        0x11FC
// 定义映射图像的命令

#define CL_COMMAND_UNMAP_MEM_OBJECT                 0x11FD
// 定义取消映射内存对象的命令

#define CL_COMMAND_MARKER                           0x11FE
// 定义标记命令

#define CL_COMMAND_ACQUIRE_GL_OBJECTS               0x11FF
// 定义获取 OpenGL 对象的命令

#define CL_COMMAND_RELEASE_GL_OBJECTS               0x1200
// 定义释放 OpenGL 对象的命令

#ifdef CL_VERSION_1_1
#define CL_COMMAND_READ_BUFFER_RECT                 0x1201
// 如果 OpenCL 版本为 1.1，则定义读取缓冲区矩形的命令

#define CL_COMMAND_WRITE_BUFFER_RECT                0x1202
// 如果 OpenCL 版本为 1.1，则定义写入缓冲区矩形的命令

#define CL_COMMAND_COPY_BUFFER_RECT                 0x1203
// 如果 OpenCL 版本为 1.1，则定义复制缓冲区矩形的命令

#define CL_COMMAND_USER                             0x1204
// 如果 OpenCL 版本为 1.1，则定义用户自定义命令
#endif

#ifdef CL_VERSION_1_2
#define CL_COMMAND_BARRIER                          0x1205
// 如果 OpenCL 版本为 1.2，则定义屏障命令

#define CL_COMMAND_MIGRATE_MEM_OBJECTS              0x1206
// 如果 OpenCL 版本为 1.2，则定义迁移内存对象的命令

#define CL_COMMAND_FILL_BUFFER                      0x1207
// 如果 OpenCL 版本为 1.2，则定义填充缓冲区的命令

#define CL_COMMAND_FILL_IMAGE                       0x1208
// 如果 OpenCL 版本为 1.2，则定义填充图像的命令
#endif

#ifdef CL_VERSION_2_0
#define CL_COMMAND_SVM_FREE                         0x1209
// 如果 OpenCL 版本为 2.0，则定义释放 SVM 内存的命令

#define CL_COMMAND_SVM_MEMCPY                       0x120A
// 如果 OpenCL 版本为 2.0，则定义 SVM 内存复制的命令

#define CL_COMMAND_SVM_MEMFILL                      0x120B
// 如果 OpenCL 版本为 2.0，则定义 SVM 内存填充的命令

#define CL_COMMAND_SVM_MAP                          0x120C
// 如果 OpenCL 版本为 2.0，则定义 SVM 内存映射的命令

#define CL_COMMAND_SVM_UNMAP                        0x120D
// 如果 OpenCL 版本为 2.0，则定义 SVM 内存取消映射的命令
#endif

/* command execution status */
#define CL_COMPLETE                                 0x0
// 定义命令执行完成的状态

#define CL_RUNNING                                  0x1
// 定义命令执行中的状态

#define CL_SUBMITTED                                0x2
// 定义命令已提交的状态

#define CL_QUEUED                                   0x3
// 定义命令已排队的状态

#ifdef CL_VERSION_1_1

/* cl_buffer_create_type */
#define CL_BUFFER_CREATE_TYPE_REGION                0x1220
// 如果 OpenCL 版本为 1.1，则定义缓冲区创建类型为区域
#endif

/* cl_profiling_info */
#define CL_PROFILING_COMMAND_QUEUED                 0x1280
// 定义命令排队的性能信息

#define CL_PROFILING_COMMAND_SUBMIT                 0x1281
// 定义命令提交的性能信息

#define CL_PROFILING_COMMAND_START                  0x1282
// 定义命令开始执行的性能信息

#define CL_PROFILING_COMMAND_END                    0x1283
// 定义命令执行结束的性能信息

#ifdef CL_VERSION_2_0
#define CL_PROFILING_COMMAND_COMPLETE               0x1284
// 如果 OpenCL 版本为 2.0，则定义命令执行完成的性能信息
#endif

/********************************************************************************************************/
/* 获取平台的 ID 列表 */
extern CL_API_ENTRY cl_int CL_API_CALL
clGetPlatformIDs(cl_uint          num_entries,  // 期望返回的平台 ID 数量
                 cl_platform_id * platforms,    // 存储平台 ID 的数组
                 cl_uint *        num_platforms) CL_API_SUFFIX__VERSION_1_0;  // 返回实际的平台数量

/* 获取平台信息 */
extern CL_API_ENTRY cl_int CL_API_CALL
clGetPlatformInfo(cl_platform_id   platform,  // 要查询的平台 ID
                  cl_platform_info param_name,  // 要查询的信息类型
                  size_t           param_value_size,  // 存储返回信息的缓冲区大小
                  void *           param_value,  // 存储返回信息的缓冲区
                  size_t *         param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  // 返回实际的信息大小

/* 获取设备的 ID 列表 */
extern CL_API_ENTRY cl_int CL_API_CALL
clGetDeviceIDs(cl_platform_id   platform,  // 要查询的平台 ID
               cl_device_type   device_type,  // 设备类型
               cl_uint          num_entries,  // 期望返回的设备 ID 数量
               cl_device_id *   devices,  // 存储设备 ID 的数组
               cl_uint *        num_devices) CL_API_SUFFIX__VERSION_1_0;  // 返回实际的设备数量

/* 获取设备信息 */
extern CL_API_ENTRY cl_int CL_API_CALL
clGetDeviceInfo(cl_device_id    device,  // 要查询的设备 ID
                cl_device_info  param_name,  // 要查询的信息类型
                size_t          param_value_size,  // 存储返回信息的缓冲区大小
                void *          param_value,  // 存储返回信息的缓冲区
                size_t *        param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  // 返回实际的信息大小

#ifdef CL_VERSION_1_2

/* 创建子设备 */
extern CL_API_ENTRY cl_int CL_API_CALL
clCreateSubDevices(cl_device_id                         in_device,  // 父设备 ID
                   const cl_device_partition_property * properties,  // 分区属性
                   cl_uint                              num_devices,  // 期望创建的子设备数量
                   cl_device_id *                       out_devices,  // 存储子设备 ID 的数组
                   cl_uint *                            num_devices_ret) CL_API_SUFFIX__VERSION_1_2;  // 返回实际创建的子设备数量

/* 保留设备 */
extern CL_API_ENTRY cl_int CL_API_CALL
clRetainDevice(cl_device_id device) CL_API_SUFFIX__VERSION_1_2;  // 增加设备的引用计数

/* 释放设备 */
extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseDevice(cl_device_id device) CL_API_SUFFIX__VERSION_1_2;  // 减少设备的引用计数

#endif

#ifdef CL_VERSION_2_1

extern CL_API_ENTRY cl_int CL_API_CALL
// 设置默认设备和命令队列
clSetDefaultDeviceCommandQueue(cl_context           context,
                               cl_device_id         device,
                               cl_command_queue     command_queue) CL_API_SUFFIX__VERSION_2_1;

// 获取设备和主机时间戳
extern CL_API_ENTRY cl_int CL_API_CALL
clGetDeviceAndHostTimer(cl_device_id    device,
                        cl_ulong*       device_timestamp,
                        cl_ulong*       host_timestamp) CL_API_SUFFIX__VERSION_2_1;

// 获取主机时间戳
extern CL_API_ENTRY cl_int CL_API_CALL
clGetHostTimer(cl_device_id device,
               cl_ulong *   host_timestamp) CL_API_SUFFIX__VERSION_2_2;

#endif

/* Context APIs */

// 创建 OpenCL 上下文
extern CL_API_ENTRY cl_context CL_API_CALL
clCreateContext(const cl_context_properties * properties,
                cl_uint              num_devices,
                const cl_device_id * devices,
                void (CL_CALLBACK * pfn_notify)(const char * errinfo,
                                                const void * private_info,
                                                size_t       cb,
                                                void *       user_data),
                void *               user_data,
                cl_int *             errcode_ret) CL_API_SUFFIX__VERSION_1_0;

// 根据设备类型创建 OpenCL 上下文
extern CL_API_ENTRY cl_context CL_API_CALL
clCreateContextFromType(const cl_context_properties * properties,
                        cl_device_type      device_type,
                        void (CL_CALLBACK * pfn_notify)(const char * errinfo,
                                                        const void * private_info,
                                                        size_t       cb,
                                                        void *       user_data),
                        void *              user_data,
                        cl_int *            errcode_ret) CL_API_SUFFIX__VERSION_1_0;

// 增加对上下文的引用计数
extern CL_API_ENTRY cl_int CL_API_CALL
clRetainContext(cl_context context) CL_API_SUFFIX__VERSION_1_0;

extern CL_API_ENTRY cl_int CL_API_CALL
# 释放 OpenCL 上下文
clReleaseContext(cl_context context) CL_API_SUFFIX__VERSION_1_0;

# 获取 OpenCL 上下文信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetContextInfo(cl_context         context,
                 cl_context_info    param_name,
                 size_t             param_value_size,
                 void *             param_value,
                 size_t *           param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;

/* Command Queue APIs */

# 如果 OpenCL 版本为 2.0，则创建带有属性的命令队列
#ifdef CL_VERSION_2_0
extern CL_API_ENTRY cl_command_queue CL_API_CALL
clCreateCommandQueueWithProperties(cl_context               context,
                                   cl_device_id             device,
                                   const cl_queue_properties *    properties,
                                   cl_int *                 errcode_ret) CL_API_SUFFIX__VERSION_2_0;
#endif

# 保留命令队列
extern CL_API_ENTRY cl_int CL_API_CALL
clRetainCommandQueue(cl_command_queue command_queue) CL_API_SUFFIX__VERSION_1_0;

# 释放命令队列
extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseCommandQueue(cl_command_queue command_queue) CL_API_SUFFIX__VERSION_1_0;

# 获取命令队列信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetCommandQueueInfo(cl_command_queue      command_queue,
                      cl_command_queue_info param_name,
                      size_t                param_value_size,
                      void *                param_value,
                      size_t *              param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;

/* Memory Object APIs */

# 创建缓冲区
extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateBuffer(cl_context   context,
               cl_mem_flags flags,
               size_t       size,
               void *       host_ptr,
               cl_int *     errcode_ret) CL_API_SUFFIX__VERSION_1_0;

# 如果 OpenCL 版本为 1.1，则继续添加 cl_mem 的声明
#ifdef CL_VERSION_1_1
extern CL_API_ENTRY cl_mem CL_API_CALL
# 创建一个子缓冲区，用于从父缓冲区中创建一个新的缓冲区对象
clCreateSubBuffer(cl_mem                   buffer,  # 父缓冲区对象
                  cl_mem_flags             flags,   # 缓冲区的访问权限标志
                  cl_buffer_create_type    buffer_create_type,  # 缓冲区的创建类型
                  const void *             buffer_create_info,  # 缓冲区的创建信息
                  cl_int *                 errcode_ret) CL_API_SUFFIX__VERSION_1_1;  # 错误码返回值

# 如果 OpenCL 版本为 1.2，则定义以下函数
#ifdef CL_VERSION_1_2
# 创建一个图像对象
extern CL_API_ENTRY cl_mem CL_API_CALL
clCreateImage(cl_context              context,  # 上下文对象
              cl_mem_flags            flags,  # 图像对象的访问权限标志
              const cl_image_format * image_format,  # 图像的格式描述
              const cl_image_desc *   image_desc,  # 图像的描述信息
              void *                  host_ptr,  # 主机指针
              cl_int *                errcode_ret) CL_API_SUFFIX__VERSION_1_2;  # 错误码返回值
#endif

# 如果 OpenCL 版本为 2.0，则定义以下函数
#ifdef CL_VERSION_2_0
# 创建一个管道对象
extern CL_API_ENTRY cl_mem CL_API_CALL
clCreatePipe(cl_context                 context,  # 上下文对象
             cl_mem_flags               flags,  # 管道对象的访问权限标志
             cl_uint                    pipe_packet_size,  # 管道数据包的大小
             cl_uint                    pipe_max_packets,  # 管道最大数据包数
             const cl_pipe_properties * properties,  # 管道的属性
             cl_int *                   errcode_ret) CL_API_SUFFIX__VERSION_2_0;  # 错误码返回值
#endif

# 保留内存对象
extern CL_API_ENTRY cl_int CL_API_CALL
clRetainMemObject(cl_mem memobj) CL_API_SUFFIX__VERSION_1_0;  # 内存对象

# 释放内存对象
extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseMemObject(cl_mem memobj) CL_API_SUFFIX__VERSION_1_0;  # 内存对象

# 获取支持的图像格式
extern CL_API_ENTRY cl_int CL_API_CALL
clGetSupportedImageFormats(cl_context           context,  # 上下文对象
                           cl_mem_flags         flags,  # 图像对象的访问权限标志
                           cl_mem_object_type   image_type,  # 图像对象的类型
                           cl_uint              num_entries,  # 图像格式数组的大小
                           cl_image_format *    image_formats,  # 图像格式数组
                           cl_uint *            num_image_formats) CL_API_SUFFIX__VERSION_1_0;  # 支持的图像格式数量
# 获取内存对象的信息
clGetMemObjectInfo(cl_mem           memobj,  # 内存对象
                   cl_mem_info      param_name,  # 参数名
                   size_t           param_value_size,  # 参数值的大小
                   void *           param_value,  # 参数值
                   size_t *         param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  # 返回参数值的大小

# 获取图像对象的信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetImageInfo(cl_mem           image,  # 图像对象
               cl_image_info    param_name,  # 参数名
               size_t           param_value_size,  # 参数值的大小
               void *           param_value,  # 参数值
               size_t *         param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  # 返回参数值的大小

# 如果 OpenCL 版本为 2.0，则获取管道对象的信息
#ifdef CL_VERSION_2_0
extern CL_API_ENTRY cl_int CL_API_CALL
clGetPipeInfo(cl_mem           pipe,  # 管道对象
              cl_pipe_info     param_name,  # 参数名
              size_t           param_value_size,  # 参数值的大小
              void *           param_value,  # 参数值
              size_t *         param_value_size_ret) CL_API_SUFFIX__VERSION_2_0;  # 返回参数值的大小
#endif

# 如果 OpenCL 版本为 1.1，则设置内存对象的析构回调函数
#ifdef CL_VERSION_1_1
extern CL_API_ENTRY cl_int CL_API_CALL
clSetMemObjectDestructorCallback(cl_mem memobj,  # 内存对象
                                 void (CL_CALLBACK * pfn_notify)(cl_mem memobj,  # 回调函数
                                                                 void * user_data),  # 用户数据
                                 void * user_data) CL_API_SUFFIX__VERSION_1_1;  # OpenCL 版本
#endif

# SVM 分配 API

# 如果 OpenCL 版本为 2.0，则分配 SVM 内存
#ifdef CL_VERSION_2_0
extern CL_API_ENTRY void * CL_API_CALL
clSVMAlloc(cl_context       context,  # 上下文
           cl_svm_mem_flags flags,  # SVM 内存标志
           size_t           size,  # 大小
           cl_uint          alignment) CL_API_SUFFIX__VERSION_2_0;  # 对齐方式

# 如果 OpenCL 版本为 2.0，则释放 SVM 内存
extern CL_API_ENTRY void CL_API_CALL
clSVMFree(cl_context        context,  # 上下文
          void *            svm_pointer) CL_API_SUFFIX__VERSION_2_0;  # SVM 指针
#endif

# 采样器 API

# 如果 OpenCL 版本为 2.0，则获取采样器对象
#ifdef CL_VERSION_2_0
extern CL_API_ENTRY cl_sampler CL_API_CALL
# 创建带有属性的采样器对象
clCreateSamplerWithProperties(cl_context                     context,
                              const cl_sampler_properties *  sampler_properties,
                              cl_int *                       errcode_ret) CL_API_SUFFIX__VERSION_2_0;

#endif

# 增加采样器对象的引用计数
extern CL_API_ENTRY cl_int CL_API_CALL
clRetainSampler(cl_sampler sampler) CL_API_SUFFIX__VERSION_1_0;

# 减少采样器对象的引用计数
extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseSampler(cl_sampler sampler) CL_API_SUFFIX__VERSION_1_0;

# 获取采样器对象的信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetSamplerInfo(cl_sampler         sampler,
                 cl_sampler_info    param_name,
                 size_t             param_value_size,
                 void *             param_value,
                 size_t *           param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;

# 创建使用源码的程序对象
extern CL_API_ENTRY cl_program CL_API_CALL
clCreateProgramWithSource(cl_context        context,
                          cl_uint           count,
                          const char **     strings,
                          const size_t *    lengths,
                          cl_int *          errcode_ret) CL_API_SUFFIX__VERSION_1_0;

# 创建使用二进制数据的程序对象
extern CL_API_ENTRY cl_program CL_API_CALL
clCreateProgramWithBinary(cl_context                     context,
                          cl_uint                        num_devices,
                          const cl_device_id *           device_list,
                          const size_t *                 lengths,
                          const unsigned char **         binaries,
                          cl_int *                       binary_status,
                          cl_int *                       errcode_ret) CL_API_SUFFIX__VERSION_1_0;

# 如果是 OpenCL 1.2 版本，则创建程序对象
#ifdef CL_VERSION_1_2

extern CL_API_ENTRY cl_program CL_API_CALL
# 创建一个包含内置内核的程序对象
clCreateProgramWithBuiltInKernels(cl_context            context,  # 上下文对象
                                  cl_uint               num_devices,  # 设备数量
                                  const cl_device_id *  device_list,  # 设备列表
                                  const char *          kernel_names,  # 内核名称
                                  cl_int *              errcode_ret) CL_API_SUFFIX__VERSION_1_2;  # 错误码返回值

#endif

#ifdef CL_VERSION_2_1

# 使用中间语言创建程序对象
extern CL_API_ENTRY cl_program CL_API_CALL
clCreateProgramWithIL(cl_context    context,  # 上下文对象
                     const void*    il,  # 中间语言
                     size_t         length,  # 长度
                     cl_int*        errcode_ret) CL_API_SUFFIX__VERSION_2_1;  # 错误码返回值

#endif

# 保留程序对象
extern CL_API_ENTRY cl_int CL_API_CALL
clRetainProgram(cl_program program) CL_API_SUFFIX__VERSION_1_0;  # 程序对象

# 释放程序对象
extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseProgram(cl_program program) CL_API_SUFFIX__VERSION_1_0;  # 程序对象

# 构建程序对象
extern CL_API_ENTRY cl_int CL_API_CALL
clBuildProgram(cl_program           program,  # 程序对象
               cl_uint              num_devices,  # 设备数量
               const cl_device_id * device_list,  # 设备列表
               const char *         options,  # 选项
               void (CL_CALLBACK *  pfn_notify)(cl_program program,  # 回调函数
                                                void * user_data),  # 用户数据
               void *               user_data) CL_API_SUFFIX__VERSION_1_0;  # 用户数据版本

#ifdef CL_VERSION_1_2

# 编译程序对象
extern CL_API_ENTRY cl_int CL_API_CALL
clCompileProgram(cl_program           program,  # 程序对象
                 cl_uint              num_devices,  # 设备数量
                 const cl_device_id * device_list,  # 设备列表
                 const char *         options,  # 选项
                 cl_uint              num_input_headers,  # 输入头文件数量
                 const cl_program *   input_headers,  # 输入头文件列表
                 const char **        header_include_names,  # 头文件包含名称
                 void (CL_CALLBACK *  pfn_notify)(cl_program program,  # 回调函数
                                                  void * user_data),  # 用户数据
                 void *               user_data) CL_API_SUFFIX__VERSION_1_2;  # 用户数据版本

extern CL_API_ENTRY cl_program CL_API_CALL
// 定义一个函数，用于将多个程序链接成一个程序对象
clLinkProgram(cl_context           context,  // 上下文对象
              cl_uint              num_devices,  // 设备数量
              const cl_device_id * device_list,  // 设备列表
              const char *         options,  // 选项
              cl_uint              num_input_programs,  // 输入程序数量
              const cl_program *   input_programs,  // 输入程序列表
              void (CL_CALLBACK *  pfn_notify)(cl_program program,  // 回调函数指针，用于通知程序链接完成
                                               void * user_data),  // 用户数据指针
              void *               user_data,  // 用户数据
              cl_int *             errcode_ret) CL_API_SUFFIX__VERSION_1_2;  // 错误码返回值

#endif

#ifdef CL_VERSION_2_2

// 设置程序释放回调函数
extern CL_API_ENTRY cl_int CL_API_CALL
clSetProgramReleaseCallback(cl_program          program,  // 程序对象
                            void (CL_CALLBACK * pfn_notify)(cl_program program,  // 回调函数指针，用于通知程序释放完成
                                                            void * user_data),  // 用户数据指针
                            void *              user_data) CL_API_SUFFIX__VERSION_2_2;  // 用户数据

// 设置程序特化常量
extern CL_API_ENTRY cl_int CL_API_CALL
clSetProgramSpecializationConstant(cl_program  program,  // 程序对象
                                   cl_uint     spec_id,  // 特化常量 ID
                                   size_t      spec_size,  // 特化常量大小
                                   const void* spec_value) CL_API_SUFFIX__VERSION_2_2;  // 特化常量值

#endif

#ifdef CL_VERSION_1_2

// 卸载平台编译器
extern CL_API_ENTRY cl_int CL_API_CALL
clUnloadPlatformCompiler(cl_platform_id platform) CL_API_SUFFIX__VERSION_1_2;  // 平台对象

#endif

// 获取程序信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetProgramInfo(cl_program         program,  // 程序对象
                 cl_program_info    param_name,  // 参数名
                 size_t             param_value_size,  // 参数值大小
                 void *             param_value,  // 参数值
                 size_t *           param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  // 参数值大小返回值

extern CL_API_ENTRY cl_int CL_API_CALL
# 定义了一个函数，用于获取程序构建信息
clGetProgramBuildInfo(cl_program            program,  # 程序对象
                      cl_device_id          device,   # 设备对象
                      cl_program_build_info param_name,  # 构建信息参数名
                      size_t                param_value_size,  # 参数值大小
                      void *                param_value,  # 参数值
                      size_t *              param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  # 参数值大小返回

# Kernel Object APIs

# 创建一个内核对象
extern CL_API_ENTRY cl_kernel CL_API_CALL
clCreateKernel(cl_program      program,  # 程序对象
               const char *    kernel_name,  # 内核名称
               cl_int *        errcode_ret) CL_API_SUFFIX__VERSION_1_0;  # 错误码返回

# 在程序中创建多个内核对象
extern CL_API_ENTRY cl_int CL_API_CALL
clCreateKernelsInProgram(cl_program     program,  # 程序对象
                         cl_uint        num_kernels,  # 内核数量
                         cl_kernel *    kernels,  # 内核对象数组
                         cl_uint *      num_kernels_ret) CL_API_SUFFIX__VERSION_1_0;  # 返回创建的内核数量

# 如果 OpenCL 版本为 2.1，则定义以下函数
#ifdef CL_VERSION_2_1

# 克隆一个内核对象
extern CL_API_ENTRY cl_kernel CL_API_CALL
clCloneKernel(cl_kernel     source_kernel,  # 源内核对象
              cl_int*       errcode_ret) CL_API_SUFFIX__VERSION_2_1;  # 错误码返回

#endif

# 保留一个内核对象
extern CL_API_ENTRY cl_int CL_API_CALL
clRetainKernel(cl_kernel    kernel) CL_API_SUFFIX__VERSION_1_0;  # 内核对象

# 释放一个内核对象
extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseKernel(cl_kernel   kernel) CL_API_SUFFIX__VERSION_1_0;  # 内核对象

# 设置内核对象的参数
extern CL_API_ENTRY cl_int CL_API_CALL
clSetKernelArg(cl_kernel    kernel,  # 内核对象
               cl_uint      arg_index,  # 参数索引
               size_t       arg_size,  # 参数大小
               const void * arg_value) CL_API_SUFFIX__VERSION_1_0;  # 参数值

# 如果 OpenCL 版本为 2.0，则定义以下函数
#ifdef CL_VERSION_2_0

# 设置内核对象的参数为 SVM 指针
extern CL_API_ENTRY cl_int CL_API_CALL
clSetKernelArgSVMPointer(cl_kernel    kernel,  # 内核对象
                         cl_uint      arg_index,  # 参数索引
                         const void * arg_value) CL_API_SUFFIX__VERSION_2_0;  # 参数值为 SVM 指针

extern CL_API_ENTRY cl_int CL_API_CALL
# 设置内核执行信息
clSetKernelExecInfo(cl_kernel            kernel,
                    cl_kernel_exec_info  param_name,
                    size_t               param_value_size,
                    const void *         param_value) CL_API_SUFFIX__VERSION_2_0;

#endif

# 获取内核信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetKernelInfo(cl_kernel       kernel,
                cl_kernel_info  param_name,
                size_t          param_value_size,
                void *          param_value,
                size_t *        param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;

# 获取内核参数信息
#ifdef CL_VERSION_1_2
extern CL_API_ENTRY cl_int CL_API_CALL
clGetKernelArgInfo(cl_kernel       kernel,
                   cl_uint         arg_indx,
                   cl_kernel_arg_info  param_name,
                   size_t          param_value_size,
                   void *          param_value,
                   size_t *        param_value_size_ret) CL_API_SUFFIX__VERSION_1_2;
#endif

# 获取内核工作组信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetKernelWorkGroupInfo(cl_kernel                  kernel,
                         cl_device_id               device,
                         cl_kernel_work_group_info  param_name,
                         size_t                     param_value_size,
                         void *                     param_value,
                         size_t *                   param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;

# 获取内核信息（2.1 版本）
#ifdef CL_VERSION_2_1
extern CL_API_ENTRY cl_int CL_API_CALL
# 获取内核子组信息
clGetKernelSubGroupInfo(cl_kernel                   kernel,  # 内核对象
                        cl_device_id                device,  # 设备对象
                        cl_kernel_sub_group_info    param_name,  # 子组信息参数名
                        size_t                      input_value_size,  # 输入值大小
                        const void*                 input_value,  # 输入值
                        size_t                      param_value_size,  # 参数值大小
                        void*                       param_value,  # 参数值
                        size_t*                     param_value_size_ret) CL_API_SUFFIX__VERSION_2_1;  # 参数值大小返回

#endif

/* 事件对象 API */
extern CL_API_ENTRY cl_int CL_API_CALL
clWaitForEvents(cl_uint             num_events,  # 事件数量
                const cl_event *    event_list) CL_API_SUFFIX__VERSION_1_0;  # 事件列表

extern CL_API_ENTRY cl_int CL_API_CALL
clGetEventInfo(cl_event         event,  # 事件对象
               cl_event_info    param_name,  # 参数名
               size_t           param_value_size,  # 参数值大小
               void *           param_value,  # 参数值
               size_t *         param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  # 参数值大小返回

#ifdef CL_VERSION_1_1

extern CL_API_ENTRY cl_event CL_API_CALL
clCreateUserEvent(cl_context    context,  # 上下文对象
                  cl_int *      errcode_ret) CL_API_SUFFIX__VERSION_1_1;  # 错误码返回

#endif

extern CL_API_ENTRY cl_int CL_API_CALL
clRetainEvent(cl_event event) CL_API_SUFFIX__VERSION_1_0;  # 保留事件对象

extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseEvent(cl_event event) CL_API_SUFFIX__VERSION_1_0;  # 释放事件对象

#ifdef CL_VERSION_1_1

extern CL_API_ENTRY cl_int CL_API_CALL
clSetUserEventStatus(cl_event   event,  # 事件对象
                     cl_int     execution_status) CL_API_SUFFIX__VERSION_1_1;  # 设置用户事件状态

extern CL_API_ENTRY cl_int CL_API_CALL
# 设置事件回调函数，当事件发生时调用回调函数
clSetEventCallback(cl_event    event,  # 事件对象
                   cl_int      command_exec_callback_type,  # 命令执行回调类型
                   void (CL_CALLBACK * pfn_notify)(cl_event event,  # 回调函数指针，包括事件对象、事件命令状态和用户数据
                                                   cl_int   event_command_status,
                                                   void *   user_data),
                   void *      user_data) CL_API_SUFFIX__VERSION_1_1;  # 用户数据

#endif

/* Profiling APIs */
extern CL_API_ENTRY cl_int CL_API_CALL
clGetEventProfilingInfo(cl_event            event,  # 获取事件的性能信息
                        cl_profiling_info   param_name,  # 参数名称
                        size_t              param_value_size,  # 参数值的大小
                        void *              param_value,  # 参数值
                        size_t *            param_value_size_ret) CL_API_SUFFIX__VERSION_1_0;  # 参数值的大小返回

/* Flush and Finish APIs */
extern CL_API_ENTRY cl_int CL_API_CALL
clFlush(cl_command_queue command_queue) CL_API_SUFFIX__VERSION_1_0;  # 刷新命令队列

extern CL_API_ENTRY cl_int CL_API_CALL
clFinish(cl_command_queue command_queue) CL_API_SUFFIX__VERSION_1_0;  # 完成命令队列

/* Enqueued Commands APIs */
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueReadBuffer(cl_command_queue    command_queue,  # 将缓冲区中的数据读取到主机内存中
                    cl_mem              buffer,  # 缓冲区对象
                    cl_bool             blocking_read,  # 是否阻塞读取
                    size_t              offset,  # 偏移量
                    size_t              size,  # 大小
                    void *              ptr,  # 指向主机内存的指针
                    cl_uint             num_events_in_wait_list,  # 等待事件列表中的事件数量
                    const cl_event *    event_wait_list,  # 等待事件列表
                    cl_event *          event) CL_API_SUFFIX__VERSION_1_0;  # 事件对象

#ifdef CL_VERSION_1_1

extern CL_API_ENTRY cl_int CL_API_CALL
# 从设备内存中读取数据到主机内存中的矩形区域
clEnqueueReadBufferRect(cl_command_queue    command_queue,  # 命令队列
                        cl_mem              buffer,          # 缓冲区对象
                        cl_bool             blocking_read,   # 是否阻塞读取
                        const size_t *      buffer_offset,   # 缓冲区偏移量
                        const size_t *      host_offset,     # 主机偏移量
                        const size_t *      region,          # 区域大小
                        size_t              buffer_row_pitch, # 缓冲区行间距
                        size_t              buffer_slice_pitch,  # 缓冲区切片间距
                        size_t              host_row_pitch,  # 主机行间距
                        size_t              host_slice_pitch,  # 主机切片间距
                        void *              ptr,  # 指向主机内存的指针
                        cl_uint             num_events_in_wait_list,  # 等待事件列表中的事件数量
                        const cl_event *    event_wait_list,  # 等待事件列表
                        cl_event *          event) CL_API_SUFFIX__VERSION_1_1;  # 事件对象

#endif

# 将数据从主机内存写入到设备内存
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueWriteBuffer(cl_command_queue   command_queue,  # 命令队列
                     cl_mem             buffer,  # 缓冲区对象
                     cl_bool            blocking_write,  # 是否阻塞写入
                     size_t             offset,  # 偏移量
                     size_t             size,  # 大小
                     const void *       ptr,  # 指向主机内存的指针
                     cl_uint            num_events_in_wait_list,  # 等待事件列表中的事件数量
                     const cl_event *   event_wait_list,  # 等待事件列表
                     cl_event *         event) CL_API_SUFFIX__VERSION_1_0;  # 事件对象

#ifdef CL_VERSION_1_1

extern CL_API_ENTRY cl_int CL_API_CALL
# 写入矩形区域的数据到缓冲区
clEnqueueWriteBufferRect(cl_command_queue    command_queue,  # 命令队列
                         cl_mem              buffer,          # 缓冲区
                         cl_bool             blocking_write,  # 是否阻塞写入
                         const size_t *      buffer_offset,   # 缓冲区偏移量
                         const size_t *      host_offset,     # 主机偏移量
                         const size_t *      region,          # 区域大小
                         size_t              buffer_row_pitch, # 缓冲区行间距
                         size_t              buffer_slice_pitch,  # 缓冲区切片间距
                         size_t              host_row_pitch,   # 主机行间距
                         size_t              host_slice_pitch,  # 主机切片间距
                         const void *        ptr,             # 指向数据的指针
                         cl_uint             num_events_in_wait_list,  # 等待事件列表中的事件数量
                         const cl_event *    event_wait_list,  # 等待事件列表
                         cl_event *          event) CL_API_SUFFIX__VERSION_1_1;  # 事件

#endif

#ifdef CL_VERSION_1_2

extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueFillBuffer(cl_command_queue   command_queue,  # 命令队列
                    cl_mem             buffer,         # 缓冲区
                    const void *       pattern,        # 模式数据
                    size_t             pattern_size,   # 模式数据大小
                    size_t             offset,         # 偏移量
                    size_t             size,           # 大小
                    cl_uint            num_events_in_wait_list,  # 等待事件列表中的事件数量
                    const cl_event *   event_wait_list,  # 等待事件列表
                    cl_event *         event) CL_API_SUFFIX__VERSION_1_2;  # 事件

#endif

extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueCopyBuffer(cl_command_queue    command_queue,  # 命令队列
                    cl_mem              src_buffer,      # 源缓冲区
                    cl_mem              dst_buffer,      # 目标缓冲区
                    size_t              src_offset,      # 源偏移量
                    size_t              dst_offset,      # 目标偏移量
                    size_t              size,            # 大小
                    cl_uint             num_events_in_wait_list,  # 等待事件列表中的事件数量
                    const cl_event *    event_wait_list,  # 等待事件列表
                    cl_event *          event) CL_API_SUFFIX__VERSION_1_0;  # 事件

#ifdef CL_VERSION_1_1
# 定义一个名为 clEnqueueCopyBufferRect 的函数，用于在命令队列中执行缓冲区之间的矩形拷贝操作
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueCopyBufferRect(cl_command_queue    command_queue,  # 命令队列
                        cl_mem              src_buffer,      # 源缓冲区
                        cl_mem              dst_buffer,      # 目标缓冲区
                        const size_t *      src_origin,      # 源起始位置
                        const size_t *      dst_origin,      # 目标起始位置
                        const size_t *      region,          # 拷贝的区域大小
                        size_t              src_row_pitch,   # 源缓冲区的行间距
                        size_t              src_slice_pitch, # 源缓冲区的切片间距
                        size_t              dst_row_pitch,   # 目标缓冲区的行间距
                        size_t              dst_slice_pitch, # 目标缓冲区的切片间距
                        cl_uint             num_events_in_wait_list,  # 等待事件列表中的事件数量
                        const cl_event *    event_wait_list,          # 等待的事件列表
                        cl_event *          event) CL_API_SUFFIX__VERSION_1_1;  # 事件对象

#endif

# 定义一个名为 clEnqueueReadImage 的函数，用于在命令队列中执行从图像对象中读取数据的操作
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueReadImage(cl_command_queue     command_queue,  # 命令队列
                   cl_mem               image,          # 图像对象
                   cl_bool              blocking_read,  # 是否阻塞读取
                   const size_t *       origin,         # 读取的起始位置
                   const size_t *       region,         # 读取的区域大小
                   size_t               row_pitch,      # 行间距
                   size_t               slice_pitch,    # 切片间距
                   void *               ptr,            # 存储读取数据的指针
                   cl_uint              num_events_in_wait_list,  # 等待事件列表中的事件数量
                   const cl_event *     event_wait_list,          # 等待的事件列表
                   cl_event *           event) CL_API_SUFFIX__VERSION_1_0;  # 事件对象

extern CL_API_ENTRY cl_int CL_API_CALL
# 将数据从主机内存写入图像对象
clEnqueueWriteImage(cl_command_queue    command_queue,  # 命令队列
                    cl_mem              image,           # 图像对象
                    cl_bool             blocking_write,  # 是否阻塞写入
                    const size_t *      origin,          # 写入的起始位置
                    const size_t *      region,          # 写入的区域大小
                    size_t              input_row_pitch, # 输入行间距
                    size_t              input_slice_pitch,  # 输入切片间距
                    const void *        ptr,             # 指向要写入的数据的指针
                    cl_uint             num_events_in_wait_list,  # 等待事件列表中的事件数量
                    const cl_event *    event_wait_list,  # 等待的事件列表
                    cl_event *          event) CL_API_SUFFIX__VERSION_1_0;  # 事件

#ifdef CL_VERSION_1_2

# 填充图像对象的数据
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueFillImage(cl_command_queue   command_queue,  # 命令队列
                   cl_mem             image,          # 图像对象
                   const void *       fill_color,     # 填充颜色
                   const size_t *     origin,         # 填充的起始位置
                   const size_t *     region,         # 填充的区域大小
                   cl_uint            num_events_in_wait_list,  # 等待事件列表中的事件数量
                   const cl_event *   event_wait_list,  # 等待的事件列表
                   cl_event *         event) CL_API_SUFFIX__VERSION_1_2;  # 事件

#endif

# 将一个图像对象的数据复制到另一个图像对象
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueCopyImage(cl_command_queue     command_queue,  # 命令队列
                   cl_mem               src_image,       # 源图像对象
                   cl_mem               dst_image,       # 目标图像对象
                   const size_t *       src_origin,      # 源图像的起始位置
                   const size_t *       dst_origin,      # 目标图像的起始位置
                   const size_t *       region,          # 复制的区域大小
                   cl_uint              num_events_in_wait_list,  # 等待事件列表中的事件数量
                   const cl_event *     event_wait_list,  # 等待的事件列表
                   cl_event *           event) CL_API_SUFFIX__VERSION_1_0;  # 事件

extern CL_API_ENTRY cl_int CL_API_CALL
# 将图像数据从图像对象复制到缓冲区对象
clEnqueueCopyImageToBuffer(cl_command_queue command_queue,  # 命令队列
                           cl_mem           src_image,       # 源图像对象
                           cl_mem           dst_buffer,      # 目标缓冲区对象
                           const size_t *   src_origin,      # 源图像的起始位置
                           const size_t *   region,          # 复制的区域大小
                           size_t           dst_offset,      # 目标缓冲区的偏移量
                           cl_uint          num_events_in_wait_list,  # 等待事件列表中的事件数量
                           const cl_event * event_wait_list,          # 等待的事件列表
                           cl_event *       event) CL_API_SUFFIX__VERSION_1_0;  # 事件

# 将缓冲区数据从缓冲区对象复制到图像对象
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueCopyBufferToImage(cl_command_queue command_queue,  # 命令队列
                           cl_mem           src_buffer,       # 源缓冲区对象
                           cl_mem           dst_image,        # 目标图像对象
                           size_t           src_offset,       # 源缓冲区的偏移量
                           const size_t *   dst_origin,       # 目标图像的起始位置
                           const size_t *   region,           # 复制的区域大小
                           cl_uint          num_events_in_wait_list,  # 等待事件列表中的事件数量
                           const cl_event * event_wait_list,          # 等待的事件列表
                           cl_event *       event) CL_API_SUFFIX__VERSION_1_0;  # 事件

# 将缓冲区对象映射到主机内存
extern CL_API_ENTRY void * CL_API_CALL
clEnqueueMapBuffer(cl_command_queue command_queue,  # 命令队列
                   cl_mem           buffer,          # 缓冲区对象
                   cl_bool          blocking_map,    # 是否阻塞映射
                   cl_map_flags     map_flags,       # 映射标志
                   size_t           offset,          # 偏移量
                   size_t           size,            # 映射的大小
                   cl_uint          num_events_in_wait_list,  # 等待事件列表中的事件数量
                   const cl_event * event_wait_list,          # 等待的事件列表
                   cl_event *       event,                    # 事件
                   cl_int *         errcode_ret) CL_API_SUFFIX__VERSION_1_0;  # 错误码
# 将图像对象映射到主机内存
clEnqueueMapImage(cl_command_queue  command_queue,  # 命令队列
                  cl_mem            image,           # 图像对象
                  cl_bool           blocking_map,    # 是否阻塞映射
                  cl_map_flags      map_flags,       # 映射标志
                  const size_t *    origin,          # 起始位置
                  const size_t *    region,          # 映射区域大小
                  size_t *          image_row_pitch, # 图像的行间距
                  size_t *          image_slice_pitch,  # 图像的切片间距
                  cl_uint           num_events_in_wait_list,  # 等待事件列表中的事件数量
                  const cl_event *  event_wait_list,  # 等待的事件列表
                  cl_event *        event,            # 事件
                  cl_int *          errcode_ret) CL_API_SUFFIX__VERSION_1_0;  # 错误码返回值

# 取消映射内存对象
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueUnmapMemObject(cl_command_queue command_queue,  # 命令队列
                        cl_mem           memobj,         # 内存对象
                        void *           mapped_ptr,     # 映射指针
                        cl_uint          num_events_in_wait_list,  # 等待事件列表中的事件数量
                        const cl_event * event_wait_list,  # 等待的事件列表
                        cl_event *       event) CL_API_SUFFIX__VERSION_1_0;  # 事件

# 在 OpenCL 设备之间迁移内存对象
#ifdef CL_VERSION_1_2
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueMigrateMemObjects(cl_command_queue       command_queue,  # 命令队列
                           cl_uint                num_mem_objects,  # 内存对象数量
                           const cl_mem *         mem_objects,  # 内存对象数组
                           cl_mem_migration_flags flags,  # 迁移标志
                           cl_uint                num_events_in_wait_list,  # 等待事件列表中的事件数量
                           const cl_event *       event_wait_list,  # 等待的事件列表
                           cl_event *             event) CL_API_SUFFIX__VERSION_1_2;  # 事件
#endif

extern CL_API_ENTRY cl_int CL_API_CALL
# 调用 OpenCL 函数，将 NDRange 内核添加到命令队列中以执行
clEnqueueNDRangeKernel(cl_command_queue command_queue,  # 命令队列
                       cl_kernel        kernel,         # 内核对象
                       cl_uint          work_dim,       # 工作维度
                       const size_t *   global_work_offset,  # 全局工作偏移量
                       const size_t *   global_work_size,    # 全局工作大小
                       const size_t *   local_work_size,     # 局部工作大小
                       cl_uint          num_events_in_wait_list,  # 等待事件列表中的事件数量
                       const cl_event * event_wait_list,           # 等待事件列表
                       cl_event *       event) CL_API_SUFFIX__VERSION_1_0;  # 事件

# 调用 OpenCL 函数，将本地函数添加到命令队列中以执行
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueNativeKernel(cl_command_queue  command_queue,  # 命令队列
                      void (CL_CALLBACK * user_func)(void *),  # 用户函数指针
                      void *            args,                 # 参数
                      size_t            cb_args,              # 参数大小
                      cl_uint           num_mem_objects,      # 内存对象数量
                      const cl_mem *    mem_list,             # 内存对象列表
                      const void **     args_mem_loc,         # 参数内存位置
                      cl_uint           num_events_in_wait_list,  # 等待事件列表中的事件数量
                      const cl_event *  event_wait_list,           # 等待事件列表
                      cl_event *        event) CL_API_SUFFIX__VERSION_1_0;  # 事件

# 如果 OpenCL 版本为 1.2，则定义以下函数

extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueMarkerWithWaitList(cl_command_queue  command_queue,  # 命令队列
                            cl_uint           num_events_in_wait_list,  # 等待事件列表中的事件数量
                            const cl_event *  event_wait_list,           # 等待事件列表
                            cl_event *        event) CL_API_SUFFIX__VERSION_1_2;  # 事件

extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueBarrierWithWaitList(cl_command_queue  command_queue,  # 命令队列
                             cl_uint           num_events_in_wait_list,  # 等待事件列表中的事件数量
                             const cl_event *  event_wait_list,           # 等待事件列表
                             cl_event *        event) CL_API_SUFFIX__VERSION_1_2;  # 事件

# 如果 OpenCL 版本为 2.0，则定义以下函数

extern CL_API_ENTRY cl_int CL_API_CALL
# 释放由 SVM 指针数组指向的内存
clEnqueueSVMFree(cl_command_queue  command_queue,  # 命令队列
                 cl_uint           num_svm_pointers,  # SVM 指针数组的数量
                 void *            svm_pointers[],  # SVM 指针数组
                 void (CL_CALLBACK * pfn_free_func)(cl_command_queue queue,  # 释放函数指针，用于释放内存
                                                    cl_uint          num_svm_pointers,  # SVM 指针数组的数量
                                                    void *           svm_pointers[],  # SVM 指针数组
                                                    void *           user_data),  # 用户数据
                 void *            user_data,  # 用户数据
                 cl_uint           num_events_in_wait_list,  # 等待事件列表中的事件数量
                 const cl_event *  event_wait_list,  # 等待事件列表
                 cl_event *        event) CL_API_SUFFIX__VERSION_2_0;  # 事件

# 将数据从一个 SVM 指针复制到另一个 SVM 指针
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueSVMMemcpy(cl_command_queue  command_queue,  # 命令队列
                   cl_bool           blocking_copy,  # 是否阻塞复制
                   void *            dst_ptr,  # 目标 SVM 指针
                   const void *      src_ptr,  # 源 SVM 指针
                   size_t            size,  # 复制的大小
                   cl_uint           num_events_in_wait_list,  # 等待事件列表中的事件数量
                   const cl_event *  event_wait_list,  # 等待事件列表
                   cl_event *        event) CL_API_SUFFIX__VERSION_2_0;  # 事件

# 使用指定的模式填充 SVM 指针指向的内存
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueSVMMemFill(cl_command_queue  command_queue,  # 命令队列
                    void *            svm_ptr,  # SVM 指针
                    const void *      pattern,  # 模式
                    size_t            pattern_size,  # 模式的大小
                    size_t            size,  # 填充的大小
                    cl_uint           num_events_in_wait_list,  # 等待事件列表中的事件数量
                    const cl_event *  event_wait_list,  # 等待事件列表
                    cl_event *        event) CL_API_SUFFIX__VERSION_2_0;  # 事件
# 定义了一个用于将共享虚拟内存映射到设备的 OpenCL 函数
clEnqueueSVMMap(cl_command_queue  command_queue,  # 命令队列
                cl_bool           blocking_map,   # 是否阻塞映射
                cl_map_flags      flags,          # 映射标志
                void *            svm_ptr,        # 共享虚拟内存指针
                size_t            size,           # 内存大小
                cl_uint           num_events_in_wait_list,  # 等待列表中的事件数量
                const cl_event *  event_wait_list,          # 等待的事件列表
                cl_event *        event) CL_API_SUFFIX__VERSION_2_0;  # 事件

# 定义了一个用于将共享虚拟内存从设备取消映射的 OpenCL 函数
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueSVMUnmap(cl_command_queue  command_queue,  # 命令队列
                  void *            svm_ptr,        # 共享虚拟内存指针
                  cl_uint           num_events_in_wait_list,  # 等待列表中的事件数量
                  const cl_event *  event_wait_list,          # 等待的事件列表
                  cl_event *        event) CL_API_SUFFIX__VERSION_2_0;  # 事件

#ifdef CL_VERSION_2_1

# 定义了一个用于在不同设备之间迁移共享虚拟内存的 OpenCL 函数
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueSVMMigrateMem(cl_command_queue         command_queue,  # 命令队列
                       cl_uint                  num_svm_pointers,  # 共享虚拟内存指针数量
                       const void **            svm_pointers,      # 共享虚拟内存指针数组
                       const size_t *           sizes,             # 内存大小数组
                       cl_mem_migration_flags   flags,             # 内存迁移标志
                       cl_uint                  num_events_in_wait_list,  # 等待列表中的事件数量
                       const cl_event *         event_wait_list,          # 等待的事件列表
                       cl_event *               event) CL_API_SUFFIX__VERSION_2_1;  # 事件

#endif

#ifdef CL_VERSION_1_2

/* 扩展函数访问
 *
 * 返回给定函数名的扩展函数地址，如果找不到有效的函数，则返回 NULL。客户端必须
 * 在使用或调用返回的函数地址之前检查地址是否不为 NULL。
 */
extern CL_API_ENTRY void * CL_API_CALL
clGetExtensionFunctionAddressForPlatform(cl_platform_id platform,  # 平台 ID
                                         const char *   func_name) CL_API_SUFFIX__VERSION_1_2;  # 函数名

#endif

#ifdef CL_USE_DEPRECATED_OPENCL_1_0_APIS
    /*
     *  警告：
     *     此 API 引入了可变状态到 OpenCL 实现中。它已被移除以更好地促进线程安全。1.0 版本的 API 不是线程安全的。它没有经过 OpenCL 1.1 一致性测试，因此可能无法正常工作或可能不可靠。它可能不具备性能。
     *     不建议使用此 API。使用需自担风险。
     *
     *  之前依赖此 API 的软件开发人员被指示在创建队列时设置命令队列属性。
     */
    extern CL_API_ENTRY cl_int CL_API_CALL
    clSetCommandQueueProperty(cl_command_queue              command_queue,
                              cl_command_queue_properties   properties,
                              cl_bool                       enable,
                              cl_command_queue_properties * old_properties) CL_EXT_SUFFIX__VERSION_1_0_DEPRECATED;
#endif /* CL_USE_DEPRECATED_OPENCL_1_0_APIS */

/* 声明使用 OpenCL 1.1 版本的已废弃 API */

// 创建一个二维图像对象
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_1_DEPRECATED cl_mem CL_API_CALL
clCreateImage2D(cl_context              context,  // 上下文对象
                cl_mem_flags            flags,  // 内存标识
                const cl_image_format * image_format,  // 图像格式
                size_t                  image_width,  // 图像宽度
                size_t                  image_height,  // 图像高度
                size_t                  image_row_pitch,  // 图像行间距
                void *                  host_ptr,  // 主机指针
                cl_int *                errcode_ret) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;  // 错误码返回值

// 创建一个三维图像对象
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_1_DEPRECATED cl_mem CL_API_CALL
clCreateImage3D(cl_context              context,  // 上下文对象
                cl_mem_flags            flags,  // 内存标识
                const cl_image_format * image_format,  // 图像格式
                size_t                  image_width,  // 图像宽度
                size_t                  image_height,  // 图像高度
                size_t                  image_depth,  // 图像深度
                size_t                  image_row_pitch,  // 图像行间距
                size_t                  image_slice_pitch,  // 图像层间距
                void *                  host_ptr,  // 主机指针
                cl_int *                errcode_ret) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;  // 错误码返回值

// 在命令队列中插入一个标记事件
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_1_DEPRECATED cl_int CL_API_CALL
clEnqueueMarker(cl_command_queue    command_queue,  // 命令队列
                cl_event *          event) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;  // 事件

// 在命令队列中等待事件完成
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_1_DEPRECATED cl_int CL_API_CALL
clEnqueueWaitForEvents(cl_command_queue  command_queue,  // 命令队列
                        cl_uint          num_events,  // 事件数量
                        const cl_event * event_list) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;  // 事件列表

// 在命令队列中插入一个屏障
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_1_DEPRECATED cl_int CL_API_CALL
clEnqueueBarrier(cl_command_queue command_queue) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;  // 命令队列
// 卸载 OpenCL 编译器
clUnloadCompiler(void) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;

// 获取扩展函数地址
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_1_DEPRECATED void * CL_API_CALL
clGetExtensionFunctionAddress(const char * func_name) CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED;

/* OpenCL 2.0 已废弃的 API */
// 创建命令队列
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_2_DEPRECATED cl_command_queue CL_API_CALL
clCreateCommandQueue(cl_context                     context,
                     cl_device_id                   device,
                     cl_command_queue_properties    properties,
                     cl_int *                       errcode_ret) CL_EXT_SUFFIX__VERSION_1_2_DEPRECATED;

// 创建采样器
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_2_DEPRECATED cl_sampler CL_API_CALL
clCreateSampler(cl_context          context,
                cl_bool             normalized_coords,
                cl_addressing_mode  addressing_mode,
                cl_filter_mode      filter_mode,
                cl_int *            errcode_ret) CL_EXT_SUFFIX__VERSION_1_2_DEPRECATED;

// 入队任务
extern CL_API_ENTRY CL_EXT_PREFIX__VERSION_1_2_DEPRECATED cl_int CL_API_CALL
clEnqueueTask(cl_command_queue  command_queue,
              cl_kernel         kernel,
              cl_uint           num_events_in_wait_list,
              const cl_event *  event_wait_list,
              cl_event *        event) CL_EXT_SUFFIX__VERSION_1_2_DEPRECATED;

#ifdef __cplusplus
}
#endif

#endif  /* __OPENCL_CL_H */
```