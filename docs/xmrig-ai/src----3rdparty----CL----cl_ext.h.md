# `xmrig\src\3rdparty\CL\cl_ext.h`

```
/*
 * 版权声明，允许在遵循条件的情况下使用和修改材料
 */
/*******************************************************************************
 * Copyright (c) 2008-2019 The Khronos Group Inc.
 *
 * 材料的使用和修改权限声明
 * 允许任何人免费获取此软件和/或相关文档文件（以下简称"材料"），
 * 在不受限制的情况下处理材料，包括但不限于使用、复制、修改、合并、发布、分发、许可和/或出售材料的副本，
 * 并允许材料的受益人这样做，但须遵守以下条件：
 *
 * 上述版权声明和此许可声明应包含在所有材料的副本或重要部分中。
 *
 * 对此文件的修改可能意味着它不再准确反映 Khronos 标准。
 * Khronos 规范和头文件的未修改、规范版本位于
 *    https://www.khronos.org/registry/
 *
 * 材料按"原样"提供，不附带任何形式的明示或暗示的担保，
 * 包括但不限于对适销性、特定用途的适用性和非侵权的担保。
 * 作者或版权持有人在任何情况下均不对任何索赔、损害或其他责任承担责任，
 * 无论是合同诉讼、侵权行为还是其他方式，由于材料或材料的使用或其他交易而产生。
 ******************************************************************************/

/* cl_ext.h 包含了没有外部（OpenGL、D3D）依赖的 OpenCL 扩展。 */

#ifndef __CL_EXT_H
#define __CL_EXT_H

#ifdef __cplusplus
extern "C" {
#endif

#include <CL/cl.h>

/* cl_khr_fp64 扩展 - 没有扩展 #define，因为它没有函数 */
/* 对于 OpenCL >= 120，CL_DEVICE_DOUBLE_FP_CONFIG 在 CL.h 中已定义 */

#if CL_TARGET_OPENCL_VERSION <= 110
#ifndef CL_DEVICE_DOUBLE_FP_CONFIG                       0x1032
#endif

/* cl_khr_fp16 extension - no extension #define since it has no functions  */
#define CL_DEVICE_HALF_FP_CONFIG                    0x1033

/* 内存对象销毁
 *
 * 苹果扩展，用于管理与具有CL_MEM_USE_HOST_PTR的cl_mem对象一起使用的外部分配的缓冲区
 *
 * 注册一个用户回调函数，当内存对象被删除并其资源被释放时将被调用。每次调用clSetMemObjectCallbackFn都会在与memobj关联的回调堆栈上注册指定的用户回调函数。注册的用户回调函数按照它们注册的相反顺序调用。然后调用用户回调函数，然后删除内存对象并释放其资源。这为使用memobj的应用程序（和库）提供了一种机制，用于在创建内存对象时指定的host_ptr引用的内存（用作内存对象的存储位）可以被重用或释放时通知。
 *
 * 应用程序可能不会使用传递给pfn_notify的cl_mem对象调用CL api。
 *
 * 在使用之前，请使用clGetDeviceInfo（CL_DEVICE_EXTENSIONS）检查“cl_APPLE_SetMemObjectDestructor”扩展。
 */
#define cl_APPLE_SetMemObjectDestructor 1
cl_int  CL_API_ENTRY clSetMemObjectDestructorAPPLE(  cl_mem memobj,
                                        void (* pfn_notify)(cl_mem memobj, void * user_data),
                                        void * user_data)             CL_EXT_SUFFIX__VERSION_1_0;


/* 上下文日志记录函数
 *
 * 下面的三个便利函数旨在用作clCreateContext（）的pfn_notify参数。在使用之前，请使用clGetDeviceInfo（CL_DEVICE_EXTENSIONS）检查“cl_APPLE_ContextLoggingFunctions”扩展。
 *
 * clLogMessagesToSystemLog将所有日志消息转发到Apple系统日志记录器
 */
// 定义苹果平台上的上下文日志记录函数
#define cl_APPLE_ContextLoggingFunctions 1
// 将日志消息记录到系统日志中
extern void CL_API_ENTRY clLogMessagesToSystemLogAPPLE(  const char * errstr,
                                            const void * private_info,
                                            size_t       cb,
                                            void *       user_data)  CL_EXT_SUFFIX__VERSION_1_0;

// 将日志消息记录到标准输出中
extern void CL_API_ENTRY clLogMessagesToStdoutAPPLE(   const char * errstr,
                                          const void * private_info,
                                          size_t       cb,
                                          void *       user_data)    CL_EXT_SUFFIX__VERSION_1_0;

// 将日志消息记录到标准错误输出中
extern void CL_API_ENTRY clLogMessagesToStderrAPPLE(   const char * errstr,
                                          const void * private_info,
                                          size_t       cb,
                                          void *       user_data)    CL_EXT_SUFFIX__VERSION_1_0;


/************************
* cl_khr_icd extension *
************************/
// 定义 cl_khr_icd 扩展
#define cl_khr_icd 1

// 平台信息：ICD 后缀
#define CL_PLATFORM_ICD_SUFFIX_KHR                  0x0920

// 额外的错误代码：未找到平台
#define CL_PLATFORM_NOT_FOUND_KHR                   -1001

// 获取支持 ICD 的平台 ID
extern CL_API_ENTRY cl_int CL_API_CALL
clIcdGetPlatformIDsKHR(cl_uint          num_entries,
                       cl_platform_id * platforms,
                       cl_uint *        num_platforms);

// clIcdGetPlatformIDsKHR 函数指针类型定义
typedef CL_API_ENTRY cl_int
(CL_API_CALL *clIcdGetPlatformIDsKHR_fn)(cl_uint          num_entries,
                                         cl_platform_id * platforms,
                                         cl_uint *        num_platforms);
/* cl_khr_il_program 扩展 */
#define cl_khr_il_program 1

/* 用于检索支持的中间语言的 clGetDeviceInfo 的新属性 */
#define CL_DEVICE_IL_VERSION_KHR                    0x105B

/* 用于检索程序的 IL 的 clGetProgramInfo 的新属性 */
#define CL_PROGRAM_IL_KHR                           0x1169

extern CL_API_ENTRY cl_program CL_API_CALL
clCreateProgramWithILKHR(cl_context   context,
                         const void * il,
                         size_t       length,
                         cl_int *     errcode_ret);

typedef CL_API_ENTRY cl_program
(CL_API_CALL *clCreateProgramWithILKHR_fn)(cl_context   context,
                                           const void * il,
                                           size_t       length,
                                           cl_int *     errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;

/* cl_khr_image2d_from_buffer 扩展 */
/*
 * 此扩展允许在不进行复制的情况下从 cl_mem 缓冲区创建 2D 图像。
 * 在 OpenCL 程序中，从缓冲区创建的 2D 图像关联的类型是 image2d_t。
 * 2D 图像和从缓冲区创建的 2D 图像都支持采样器和无采样器的 read_image 内置函数。
 * 同样，从缓冲区创建的 2D 图像也支持 write_image 内置函数。
 *
 * 创建 2D 图像时，客户端必须指定宽度、高度、图像格式（即通道顺序和通道数据类型），
 * 并可选地指定行间距。
 *
 * 指定的间距必须是 CL_DEVICE_IMAGE_PITCH_ALIGNMENT_KHR 像素的倍数。
 * 缓冲区的基地址必须对齐到 CL_DEVICE_IMAGE_BASE_ADDRESS_ALIGNMENT_KHR 像素。
 */
#define CL_DEVICE_IMAGE_PITCH_ALIGNMENT_KHR              0x104A
#define CL_DEVICE_IMAGE_BASE_ADDRESS_ALIGNMENT_KHR       0x104B
/*
 * cl_khr_initialize_memory extension
 */

// 定义用于初始化内存的常量
#define CL_CONTEXT_MEMORY_INITIALIZE_KHR            0x2030


/*
 * cl_khr_terminate_context extension
 */

// 定义用于终止上下文的设备能力和上下文终止的常量
#define CL_DEVICE_TERMINATE_CAPABILITY_KHR          0x2031
#define CL_CONTEXT_TERMINATE_KHR                    0x2032

// 声明终止上下文的函数指针类型和函数声明
#define cl_khr_terminate_context 1
extern CL_API_ENTRY cl_int CL_API_CALL
clTerminateContextKHR(cl_context context) CL_EXT_SUFFIX__VERSION_1_2;

typedef CL_API_ENTRY cl_int
(CL_API_CALL *clTerminateContextKHR_fn)(cl_context context) CL_EXT_SUFFIX__VERSION_1_2;


/*
 * Extension: cl_khr_spir
 *
 * This extension adds support to create an OpenCL program object from a
 * Standard Portable Intermediate Representation (SPIR) instance
 */

// 定义用于 SPIR 版本和中间二进制类型的常量
#define CL_DEVICE_SPIR_VERSIONS                     0x40E0
#define CL_PROGRAM_BINARY_TYPE_INTERMEDIATE         0x40E1


/*
 * cl_khr_create_command_queue extension
 */

// 定义用于创建命令队列的常量
#define cl_khr_create_command_queue 1

// 声明创建带属性的命令队列的函数指针类型和函数声明
typedef cl_bitfield cl_queue_properties_khr;

extern CL_API_ENTRY cl_command_queue CL_API_CALL
clCreateCommandQueueWithPropertiesKHR(cl_context context,
                                      cl_device_id device,
                                      const cl_queue_properties_khr* properties,
                                      cl_int* errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;

typedef CL_API_ENTRY cl_command_queue
(CL_API_CALL *clCreateCommandQueueWithPropertiesKHR_fn)(cl_context context,
                                                        cl_device_id device,
                                                        const cl_queue_properties_khr* properties,
                                                        cl_int* errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;
/* cl_nv_device_attribute_query extension *
******************************************/

/* 定义 NVIDIA 设备属性查询的扩展，没有函数，所以没有 #define */
#define CL_DEVICE_COMPUTE_CAPABILITY_MAJOR_NV       0x4000
#define CL_DEVICE_COMPUTE_CAPABILITY_MINOR_NV       0x4001
#define CL_DEVICE_REGISTERS_PER_BLOCK_NV            0x4002
#define CL_DEVICE_WARP_SIZE_NV                      0x4003
#define CL_DEVICE_GPU_OVERLAP_NV                    0x4004
#define CL_DEVICE_KERNEL_EXEC_TIMEOUT_NV            0x4005
#define CL_DEVICE_INTEGRATED_MEMORY_NV              0x4006


/*********************************
* cl_amd_device_attribute_query *
*********************************/

#define CL_DEVICE_PROFILING_TIMER_OFFSET_AMD        0x4036


/*********************************
* cl_arm_printf extension
*********************************/

#define CL_PRINTF_CALLBACK_ARM                      0x40B0
#define CL_PRINTF_BUFFERSIZE_ARM                    0x40B1


/***********************************
* cl_ext_device_fission extension
***********************************/
#define cl_ext_device_fission   1

/* 定义扩展设备分裂，包括释放和保留设备的函数声明 */
extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseDeviceEXT(cl_device_id device) CL_EXT_SUFFIX__VERSION_1_1;

typedef CL_API_ENTRY cl_int
(CL_API_CALL *clReleaseDeviceEXT_fn)(cl_device_id device) CL_EXT_SUFFIX__VERSION_1_1;

extern CL_API_ENTRY cl_int CL_API_CALL
clRetainDeviceEXT(cl_device_id device) CL_EXT_SUFFIX__VERSION_1_1;

typedef CL_API_ENTRY cl_int
(CL_API_CALL *clRetainDeviceEXT_fn)(cl_device_id device) CL_EXT_SUFFIX__VERSION_1_1;

typedef cl_ulong  cl_device_partition_property_ext;
extern CL_API_ENTRY cl_int CL_API_CALL
clCreateSubDevicesEXT(cl_device_id   in_device,
                      const cl_device_partition_property_ext * properties,
                      cl_uint        num_entries,
                      cl_device_id * out_devices,
                      cl_uint *      num_devices) CL_EXT_SUFFIX__VERSION_1_1;

typedef CL_API_ENTRY cl_int
# 定义一个函数指针，用于创建子设备
(CL_API_CALL * clCreateSubDevicesEXT_fn)(cl_device_id   in_device,
                                         const cl_device_partition_property_ext * properties,
                                         cl_uint        num_entries,
                                         cl_device_id * out_devices,
                                         cl_uint *      num_devices) CL_EXT_SUFFIX__VERSION_1_1;

# 定义一系列的设备分区属性常量
#define CL_DEVICE_PARTITION_EQUALLY_EXT             0x4050
#define CL_DEVICE_PARTITION_BY_COUNTS_EXT           0x4051
#define CL_DEVICE_PARTITION_BY_NAMES_EXT            0x4052
#define CL_DEVICE_PARTITION_BY_AFFINITY_DOMAIN_EXT  0x4053

# 定义 clDeviceGetInfo 的选择器常量
#define CL_DEVICE_PARENT_DEVICE_EXT                 0x4054
#define CL_DEVICE_PARTITION_TYPES_EXT               0x4055
#define CL_DEVICE_AFFINITY_DOMAINS_EXT              0x4056
#define CL_DEVICE_REFERENCE_COUNT_EXT               0x4057
#define CL_DEVICE_PARTITION_STYLE_EXT               0x4058

# 定义错误代码常量
#define CL_DEVICE_PARTITION_FAILED_EXT              -1057
#define CL_INVALID_PARTITION_COUNT_EXT              -1058
#define CL_INVALID_PARTITION_NAME_EXT               -1059

# 定义亲和性域常量
#define CL_AFFINITY_DOMAIN_L1_CACHE_EXT             0x1
#define CL_AFFINITY_DOMAIN_L2_CACHE_EXT             0x2
#define CL_AFFINITY_DOMAIN_L3_CACHE_EXT             0x3
#define CL_AFFINITY_DOMAIN_L4_CACHE_EXT             0x4
#define CL_AFFINITY_DOMAIN_NUMA_EXT                 0x10
#define CL_AFFINITY_DOMAIN_NEXT_FISSIONABLE_EXT     0x100

# 定义设备分区属性列表终止符常量
#define CL_PROPERTIES_LIST_END_EXT                  ((cl_device_partition_property_ext) 0)
#define CL_PARTITION_BY_COUNTS_LIST_END_EXT         ((cl_device_partition_property_ext) 0)
#define CL_PARTITION_BY_NAMES_LIST_END_EXT          ((cl_device_partition_property_ext) 0 - 1)
// 定义 cl_ext_migrate_memobject 扩展的标识符
#define cl_ext_migrate_memobject 1

// 定义 cl_mem_migration_flags_ext 类型
typedef cl_bitfield cl_mem_migration_flags_ext;

// 定义 CL_MIGRATE_MEM_OBJECT_HOST_EXT 常量
#define CL_MIGRATE_MEM_OBJECT_HOST_EXT              0x1

// 定义 CL_COMMAND_MIGRATE_MEM_OBJECT_EXT 常量
#define CL_COMMAND_MIGRATE_MEM_OBJECT_EXT           0x4040

// 声明 clEnqueueMigrateMemObjectEXT 函数
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueMigrateMemObjectEXT(cl_command_queue command_queue,
                             cl_uint          num_mem_objects,
                             const cl_mem *   mem_objects,
                             cl_mem_migration_flags_ext flags,
                             cl_uint          num_events_in_wait_list,
                             const cl_event * event_wait_list,
                             cl_event *       event);

// 定义 clEnqueueMigrateMemObjectEXT_fn 函数指针类型
typedef CL_API_ENTRY cl_int
(CL_API_CALL *clEnqueueMigrateMemObjectEXT_fn)(cl_command_queue command_queue,
                                               cl_uint          num_mem_objects,
                                               const cl_mem *   mem_objects,
                                               cl_mem_migration_flags_ext flags,
                                               cl_uint          num_events_in_wait_list,
                                               const cl_event * event_wait_list,
                                               cl_event *       event);


// 定义 cl_qcom_ext_host_ptr 扩展的标识符
#define cl_qcom_ext_host_ptr 1

// 定义 CL_MEM_EXT_HOST_PTR_QCOM 常量
#define CL_MEM_EXT_HOST_PTR_QCOM                  (1 << 29)

// 定义 CL_DEVICE_EXT_MEM_PADDING_IN_BYTES_QCOM 常量
#define CL_DEVICE_EXT_MEM_PADDING_IN_BYTES_QCOM   0x40A0
// 定义 CL_DEVICE_PAGE_SIZE_QCOM 常量
#define CL_DEVICE_PAGE_SIZE_QCOM                  0x40A1
// 定义 CL_IMAGE_ROW_ALIGNMENT_QCOM 常量
#define CL_IMAGE_ROW_ALIGNMENT_QCOM               0x40A2
// 定义 CL_IMAGE_SLICE_ALIGNMENT_QCOM 常量
#define CL_IMAGE_SLICE_ALIGNMENT_QCOM             0x40A3
// 定义 CL_MEM_HOST_UNCACHED_QCOM 常量
#define CL_MEM_HOST_UNCACHED_QCOM                 0x40A4
// 定义 CL_MEM_HOST_WRITEBACK_QCOM 常量
#define CL_MEM_HOST_WRITEBACK_QCOM                0x40A5
// 定义 CL_MEM_HOST_WRITETHROUGH_QCOM 常量
#define CL_MEM_HOST_WRITETHROUGH_QCOM             0x40A6
# 定义了一个名为 CL_MEM_HOST_WRITE_COMBINING_QCOM 的宏，其值为 0x40A7

# 定义了一个名为 cl_image_pitch_info_qcom 的类型别名，类型为 cl_uint

# 声明了一个名为 clGetDeviceImageInfoQCOM 的函数，用于获取设备图像信息
# 参数包括设备 ID、图像宽度、图像高度、图像格式、参数名称、参数值大小、参数值、参数值大小返回值

# 定义了一个名为 cl_mem_ext_host_ptr 的结构体，包括外部内存分配类型和主机缓存策略

# 定义了一个名为 CL_MEM_HOST_IOCOHERENT_QCOM 的宏，其值为 0x40A9，表示缓存策略指定为 io-coherence

# 定义了一个名为 CL_MEM_ION_HOST_PTR_QCOM 的宏，其值为 0x40A8

# 定义了一个名为 cl_mem_ion_host_ptr 的结构体，包括外部内存分配类型、ION 文件描述符和ION分配内存的主机指针

# 定义了一个名为 CL_MEM_ANDROID_NATIVE_BUFFER_HOST_PTR_QCOM 的宏，其值为 0x40C6
# 定义了一个结构体，用于表示 Android 原生缓冲区的内存指针
typedef struct _cl_mem_android_native_buffer_host_ptr
{
    # 外部内存分配的类型
    # 对于 Android 原生缓冲区，必须是 CL_MEM_ANDROID_NATIVE_BUFFER_HOST_PTR_QCOM
    cl_mem_ext_host_ptr  ext_host_ptr;

    # 指向 Android 原生缓冲区的虚拟指针
    void*                anb_ptr;

} cl_mem_android_native_buffer_host_ptr;


/******************************************
 * cl_img_yuv_image extension *
 ******************************************/

# 在 clCreateImage 中使用的图像格式
#define CL_NV21_IMG                                 0x40D0
#define CL_YV12_IMG                                 0x40D1


/******************************************
 * cl_img_cached_allocations extension *
 ******************************************/

# 由 clCreateBuffer 使用的标志值
#define CL_MEM_USE_UNCACHED_CPU_MEMORY_IMG          (1 << 26)
#define CL_MEM_USE_CACHED_CPU_MEMORY_IMG            (1 << 27)


/******************************************
 * cl_img_use_gralloc_ptr extension *
 ******************************************/
# 定义了 cl_img_use_gralloc_ptr
#define cl_img_use_gralloc_ptr 1

# 由 clCreateBuffer 使用的标志值
#define CL_MEM_USE_GRALLOC_PTR_IMG                  (1 << 28)

# 由 clGetEventInfo 使用的值
#define CL_COMMAND_ACQUIRE_GRALLOC_OBJECTS_IMG      0x40D2
#define CL_COMMAND_RELEASE_GRALLOC_OBJECTS_IMG      0x40D3

# clEnqueueReleaseGrallocObjectsIMG 的错误码
#define CL_GRALLOC_RESOURCE_NOT_ACQUIRED_IMG        0x40D4

# clEnqueueAcquireGrallocObjectsIMG 的声明
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueAcquireGrallocObjectsIMG(cl_command_queue      command_queue,
                                  cl_uint               num_objects,
                                  const cl_mem *        mem_objects,
                                  cl_uint               num_events_in_wait_list,
                                  const cl_event *      event_wait_list,
                                  cl_event *            event) CL_EXT_SUFFIX__VERSION_1_2;
# 声明一个名为 clEnqueueReleaseGrallocObjectsIMG 的函数，用于在 OpenCL 命令队列中释放指定的 gralloc 对象
# 参数包括命令队列、对象数量、对象数组、等待事件数量、等待事件数组、事件
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueReleaseGrallocObjectsIMG(cl_command_queue      command_queue,
                                  cl_uint               num_objects,
                                  const cl_mem *        mem_objects,
                                  cl_uint               num_events_in_wait_list,
                                  const cl_event *      event_wait_list,
                                  cl_event *            event) CL_EXT_SUFFIX__VERSION_1_2;


/*********************************
* cl_khr_subgroups extension
*********************************/
# 定义 cl_khr_subgroups 扩展，表示支持子组功能
#define cl_khr_subgroups 1

# 如果未定义 OpenCL 2.1 版本，则声明 cl_kernel_sub_group_info 类型和相关常量
#if !defined(CL_VERSION_2_1)
/* For OpenCL 2.1 and newer, cl_kernel_sub_group_info is declared in CL.h.
   In hindsight, there should have been a khr suffix on this type for
   the extension, but keeping it un-suffixed to maintain backwards
   compatibility. */
# 定义 cl_kernel_sub_group_info 类型为 cl_uint
typedef cl_uint             cl_kernel_sub_group_info;
#endif

# 定义 cl_kernel_sub_group_info 常量
# 表示可用于查询 NDRange 内核的最大子组大小
#define CL_KERNEL_MAX_SUB_GROUP_SIZE_FOR_NDRANGE_KHR    0x2033
# 表示可用于查询 NDRange 内核的子组数量
#define CL_KERNEL_SUB_GROUP_COUNT_FOR_NDRANGE_KHR       0x2034

# 声明一个名为 clGetKernelSubGroupInfoKHR 的函数，用于查询内核的子组信息
# 参数包括内核、设备、参数名、输入值大小、输入值、参数值大小、参数值、参数值大小返回值
extern CL_API_ENTRY cl_int CL_API_CALL
clGetKernelSubGroupInfoKHR(cl_kernel    in_kernel,
                           cl_device_id in_device,
                           cl_kernel_sub_group_info param_name,
                           size_t       input_value_size,
                           const void * input_value,
                           size_t       param_value_size,
                           void *       param_value,
                           size_t *     param_value_size_ret) CL_EXT_SUFFIX__VERSION_2_0_DEPRECATED;

# 定义一个名为 cl_int 的类型别名
typedef CL_API_ENTRY cl_int
// 定义一个函数指针，用于调用 clGetKernelSubGroupInfoKHR 函数
(CL_API_CALL * clGetKernelSubGroupInfoKHR_fn)(cl_kernel    in_kernel,
                                              cl_device_id in_device,
                                              cl_kernel_sub_group_info param_name,
                                              size_t       input_value_size,
                                              const void * input_value,
                                              size_t       param_value_size,
                                              void *       param_value,
                                              size_t *     param_value_size_ret) CL_EXT_SUFFIX__VERSION_2_0_DEPRECATED;

/*********************************
* cl_khr_mipmap_image extension
*********************************/

// 定义 cl_sampler_properties 常量
#define CL_SAMPLER_MIP_FILTER_MODE_KHR              0x1155
#define CL_SAMPLER_LOD_MIN_KHR                      0x1156
#define CL_SAMPLER_LOD_MAX_KHR                      0x1157

/*********************************
* cl_khr_priority_hints extension
*********************************/
// 定义 cl_khr_priority_hints 常量
#define cl_khr_priority_hints 1

// 定义 cl_queue_priority_khr 类型
typedef cl_uint  cl_queue_priority_khr;

// 定义 cl_command_queue_properties 常量
#define CL_QUEUE_PRIORITY_KHR 0x1096

// 定义 cl_queue_priority_khr 常量
#define CL_QUEUE_PRIORITY_HIGH_KHR (1<<0)
#define CL_QUEUE_PRIORITY_MED_KHR (1<<1)
#define CL_QUEUE_PRIORITY_LOW_KHR (1<<2)

/*********************************
* cl_khr_throttle_hints extension
*********************************/
// 定义 cl_khr_throttle_hints 常量
#define cl_khr_throttle_hints 1

// 定义 cl_queue_throttle_khr 类型
typedef cl_uint  cl_queue_throttle_khr;

// 定义 cl_command_queue_properties 常量
#define CL_QUEUE_THROTTLE_KHR 0x1097

// 定义 cl_queue_throttle_khr 常量
#define CL_QUEUE_THROTTLE_HIGH_KHR (1<<0)
#define CL_QUEUE_THROTTLE_MED_KHR (1<<1)
#define CL_QUEUE_THROTTLE_LOW_KHR (1<<2)

/*********************************
* cl_khr_subgroup_named_barrier
*********************************/
/* This extension define is for backwards compatibility.
   It shouldn't be required since this extension has no new functions. */
#define cl_khr_subgroup_named_barrier 1

/* cl_device_info */
#define CL_DEVICE_MAX_NAMED_BARRIER_COUNT_KHR       0x2035

/**********************************
 * cl_arm_import_memory extension *
 **********************************/
#define cl_arm_import_memory 1

typedef intptr_t cl_import_properties_arm;

/* Default and valid proporties name for cl_arm_import_memory */
#define CL_IMPORT_TYPE_ARM                        0x40B2

/* Host process memory type default value for CL_IMPORT_TYPE_ARM property */
#define CL_IMPORT_TYPE_HOST_ARM                   0x40B3

/* DMA BUF memory type value for CL_IMPORT_TYPE_ARM property */
#define CL_IMPORT_TYPE_DMA_BUF_ARM                0x40B4

/* Protected DMA BUF memory type value for CL_IMPORT_TYPE_ARM property */
#define CL_IMPORT_TYPE_PROTECTED_ARM              0x40B5

/* This extension adds a new function that allows for direct memory import into
 * OpenCL via the clImportMemoryARM function.
 *
 * Memory imported through this interface will be mapped into the device's page
 * tables directly, providing zero copy access. It will never fall back to copy
 * operations and aliased buffers.
 *
 * Types of memory supported for import are specified as additional extension
 * strings.
 *
 * This extension produces cl_mem allocations which are compatible with all other
 * users of cl_mem in the standard API.
 *
 * This extension maps pages with the same properties as the normal buffer creation
 * function clCreateBuffer.
 */
extern CL_API_ENTRY cl_mem CL_API_CALL
// 声明一个函数，用于在 ARM 架构上导入内存
clImportMemoryARM( cl_context context,  // OpenCL 上下文
                   cl_mem_flags flags,  // 内存标志
                   const cl_import_properties_arm *properties,  // ARM 共享虚拟内存的属性
                   void *memory,  // 内存指针
                   size_t size,  // 内存大小
                   cl_int *errcode_ret) CL_EXT_SUFFIX__VERSION_1_0;  // 错误码返回值

/******************************************
 * cl_arm_shared_virtual_memory extension *
 ******************************************/

// 定义 ARM 共享虚拟内存扩展的标志
#define cl_arm_shared_virtual_memory 1

/* 以下是用于 clGetDeviceInfo 的标志 */
#define CL_DEVICE_SVM_CAPABILITIES_ARM                  0x40B6  // 设备支持的 SVM 能力

/* 以下是用于 clGetMemObjectInfo 的标志 */
#define CL_MEM_USES_SVM_POINTER_ARM                     0x40B7  // 内存对象使用 SVM 指针

/* 以下是用于 clSetKernelExecInfoARM 的标志 */
#define CL_KERNEL_EXEC_INFO_SVM_PTRS_ARM                0x40B8  // 内核执行信息中 SVM 指针
#define CL_KERNEL_EXEC_INFO_SVM_FINE_GRAIN_SYSTEM_ARM   0x40B9  // 内核执行信息中 SVM 细粒度系统

/* 以下是用于 clGetEventInfo 的标志 */
#define CL_COMMAND_SVM_FREE_ARM                         0x40BA  // 释放 SVM 内存的命令
#define CL_COMMAND_SVM_MEMCPY_ARM                       0x40BB  // SVM 内存复制的命令
#define CL_COMMAND_SVM_MEMFILL_ARM                      0x40BC  // SVM 内存填充的命令
#define CL_COMMAND_SVM_MAP_ARM                          0x40BD  // SVM 内存映射的命令
#define CL_COMMAND_SVM_UNMAP_ARM                        0x40BE  // SVM 内存取消映射的命令

/* 以下是由 clGetDeviceInfo 返回的标志值，使用 CL_DEVICE_SVM_CAPABILITIES_ARM 作为 param_name */
#define CL_DEVICE_SVM_COARSE_GRAIN_BUFFER_ARM           (1 << 0)  // 设备支持 SVM 粗粒度缓冲
#define CL_DEVICE_SVM_FINE_GRAIN_BUFFER_ARM             (1 << 1)  // 设备支持 SVM 细粒度缓冲
#define CL_DEVICE_SVM_FINE_GRAIN_SYSTEM_ARM             (1 << 2)  // 设备支持 SVM 细粒度系统
#define CL_DEVICE_SVM_ATOMICS_ARM                       (1 << 3)  // 设备支持 SVM 原子操作

/* 以下是由 clSVMAllocARM 使用的标志值 */
#define CL_MEM_SVM_FINE_GRAIN_BUFFER_ARM                (1 << 10)  // SVM 细粒度缓冲
#define CL_MEM_SVM_ATOMICS_ARM                          (1 << 11)  // SVM 原子操作

// 定义 ARM 共享虚拟内存的类型
typedef cl_bitfield cl_svm_mem_flags_arm;
typedef cl_uint     cl_kernel_exec_info_arm;
typedef cl_bitfield cl_device_svm_capabilities_arm;

// 声明一个函数指针，用于在 ARM 架构上分配共享虚拟内存
extern CL_API_ENTRY void * CL_API_CALL
# 声明一个函数，用于在 ARM 平台上分配 SVM 内存
clSVMAllocARM(cl_context       context,
              cl_svm_mem_flags_arm flags,
              size_t           size,
              cl_uint          alignment) CL_EXT_SUFFIX__VERSION_1_2;

# 声明一个函数，用于在 ARM 平台上释放 SVM 内存
extern CL_API_ENTRY void CL_API_CALL
clSVMFreeARM(cl_context        context,
             void *            svm_pointer) CL_EXT_SUFFIX__VERSION_1_2;

# 声明一个函数，用于在 ARM 平台上异步释放 SVM 内存
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueSVMFreeARM(cl_command_queue  command_queue,
                    cl_uint           num_svm_pointers,
                    void *            svm_pointers[],
                    void (CL_CALLBACK * pfn_free_func)(cl_command_queue queue,
                                                       cl_uint          num_svm_pointers,
                                                       void *           svm_pointers[],
                                                       void *           user_data),
                    void *            user_data,
                    cl_uint           num_events_in_wait_list,
                    const cl_event *  event_wait_list,
                    cl_event *        event) CL_EXT_SUFFIX__VERSION_1_2;

# 声明一个函数，用于在 ARM 平台上进行 SVM 内存之间的拷贝
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueSVMMemcpyARM(cl_command_queue  command_queue,
                      cl_bool           blocking_copy,
                      void *            dst_ptr,
                      const void *      src_ptr,
                      size_t            size,
                      cl_uint           num_events_in_wait_list,
                      const cl_event *  event_wait_list,
                      cl_event *        event) CL_EXT_SUFFIX__VERSION_1_2;

# 继续声明其他函数
extern CL_API_ENTRY cl_int CL_API_CALL
# 声明一个用于在 ARM 平台上填充 SVM 内存的函数
clEnqueueSVMMemFillARM(cl_command_queue  command_queue,
                       void *            svm_ptr,
                       const void *      pattern,
                       size_t            pattern_size,
                       size_t            size,
                       cl_uint           num_events_in_wait_list,
                       const cl_event *  event_wait_list,
                       cl_event *        event) CL_EXT_SUFFIX__VERSION_1_2;

# 声明一个用于在 ARM 平台上映射 SVM 内存的函数
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueSVMMapARM(cl_command_queue  command_queue,
                   cl_bool           blocking_map,
                   cl_map_flags      flags,
                   void *            svm_ptr,
                   size_t            size,
                   cl_uint           num_events_in_wait_list,
                   const cl_event *  event_wait_list,
                   cl_event *        event) CL_EXT_SUFFIX__VERSION_1_2;

# 声明一个用于在 ARM 平台上取消映射 SVM 内存的函数
extern CL_API_ENTRY cl_int CL_API_CALL
clEnqueueSVMUnmapARM(cl_command_queue  command_queue,
                     void *            svm_ptr,
                     cl_uint           num_events_in_wait_list,
                     const cl_event *  event_wait_list,
                     cl_event *        event) CL_EXT_SUFFIX__VERSION_1_2;

# 声明一个用于设置内核参数为 SVM 指针的函数
extern CL_API_ENTRY cl_int CL_API_CALL
clSetKernelArgSVMPointerARM(cl_kernel    kernel,
                            cl_uint      arg_index,
                            const void * arg_value) CL_EXT_SUFFIX__VERSION_1_2;

# 声明一个用于设置内核执行信息的函数
extern CL_API_ENTRY cl_int CL_API_CALL
clSetKernelExecInfoARM(cl_kernel            kernel,
                       cl_kernel_exec_info_arm  param_name,
                       size_t               param_value_size,
                       const void *         param_value) CL_EXT_SUFFIX__VERSION_1_2;

# 定义 cl_arm_get_core_id 扩展
/********************************
 * cl_arm_get_core_id extension *
 ********************************/

# 如果 OpenCL 版本为 1.2，则定义 cl_arm_get_core_id 为 1
#ifdef CL_VERSION_1_2
#define cl_arm_get_core_id 1

# 用于获取设备信息属性中的核心位字段
/* Device info property for bitfield of cores present */
/* 定义 ARM 设备的计算单元位字段 */
#define CL_DEVICE_COMPUTE_UNITS_BITFIELD_ARM      0x40BF

#endif  /* CL_VERSION_1_2 */

/*********************************
* cl_arm_job_slot_selection
*********************************/

/* 定义 ARM 作业槽选择 */
#define cl_arm_job_slot_selection 1

/* cl_device_info */
/* 定义 ARM 设备作业槽 */
#define CL_DEVICE_JOB_SLOTS_ARM                   0x41E0

/* cl_command_queue_properties */
/* 定义 ARM 命令队列属性作业槽 */
#define CL_QUEUE_JOB_SLOT_ARM                     0x41E1

#ifdef __cplusplus
}
#endif


#endif /* __CL_EXT_H */
```