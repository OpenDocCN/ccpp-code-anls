# `xmrig\src\3rdparty\CL\cl_ext_intel.h`

```
# 版权声明，允许在不受限制的情况下使用和修改材料
/*******************************************************************************
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
 ******************************************************************************/
/*****************************************************************************\

Copyright (c) 2013-2019 Intel Corporation All Rights Reserved.

THESE MATERIALS ARE PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
"AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL INTEL OR ITS
# 声明代码文件的版权和责任声明
CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THESE
MATERIALS, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

# 声明代码文件的名称和摘要
File Name: cl_ext_intel.h

Abstract:

Notes:

\*****************************************************************************/

# 如果未定义__CL_EXT_INTEL_H，则定义__CL_EXT_INTEL_H
#ifndef __CL_EXT_INTEL_H
#define __CL_EXT_INTEL_H

# 包含 OpenCL 标准头文件
#include <CL/cl.h>
#include <CL/cl_platform.h>

# 如果是 C++ 环境，则使用 C 语言的方式进行编译
#ifdef __cplusplus
extern "C" {
#endif

# 定义 cl_intel_thread_local_exec 扩展
/***************************************
* cl_intel_thread_local_exec extension *
****************************************/
#define cl_intel_thread_local_exec 1

# 定义 CL_QUEUE_THREAD_LOCAL_EXEC_ENABLE_INTEL 常量
#define CL_QUEUE_THREAD_LOCAL_EXEC_ENABLE_INTEL      (((cl_bitfield)1) << 31)

# 定义 cl_intel_device_partition_by_names 扩展
/***********************************************
* cl_intel_device_partition_by_names extension *
************************************************/
#define cl_intel_device_partition_by_names 1

# 定义 CL_DEVICE_PARTITION_BY_NAMES_INTEL 和 CL_PARTITION_BY_NAMES_LIST_END_INTEL 常量
#define CL_DEVICE_PARTITION_BY_NAMES_INTEL          0x4052
#define CL_PARTITION_BY_NAMES_LIST_END_INTEL        -1

# 定义 cl_intel_accelerator、cl_intel_motion_estimation 和 cl_intel_advanced_motion_estimation 扩展
/************************************************
* cl_intel_accelerator extension                *
* cl_intel_motion_estimation extension          *
* cl_intel_advanced_motion_estimation extension *
*************************************************/
#define cl_intel_accelerator 1
#define cl_intel_motion_estimation 1
#define cl_intel_advanced_motion_estimation 1

# 定义 cl_accelerator_intel、cl_accelerator_type_intel 和 cl_accelerator_info_intel 类型
typedef struct _cl_accelerator_intel* cl_accelerator_intel;
typedef cl_uint cl_accelerator_type_intel;
typedef cl_uint cl_accelerator_info_intel;

# 定义 _cl_motion_estimation_desc_intel 结构体
typedef struct _cl_motion_estimation_desc_intel {
    cl_uint mb_block_type;
    cl_uint subpixel_mode;
    cl_uint sad_adjust_mode;
    cl_uint search_path_type;
# 定义了一个结构体 cl_motion_estimation_desc_intel
} cl_motion_estimation_desc_intel;

# 定义了一些错误代码
#define CL_INVALID_ACCELERATOR_INTEL                              -1094
#define CL_INVALID_ACCELERATOR_TYPE_INTEL                         -1095
#define CL_INVALID_ACCELERATOR_DESCRIPTOR_INTEL                   -1096
#define CL_ACCELERATOR_TYPE_NOT_SUPPORTED_INTEL                   -1097

# 定义了加速器类型 cl_accelerator_type_intel
#define CL_ACCELERATOR_TYPE_MOTION_ESTIMATION_INTEL               0x0

# 定义了加速器信息 cl_accelerator_info_intel
#define CL_ACCELERATOR_DESCRIPTOR_INTEL                           0x4090
#define CL_ACCELERATOR_REFERENCE_COUNT_INTEL                      0x4091
#define CL_ACCELERATOR_CONTEXT_INTEL                              0x4092
#define CL_ACCELERATOR_TYPE_INTEL                                 0x4093

# 定义了运动检测描述 cl_motion_detect_desc_intel 的一些标志
#define CL_ME_MB_TYPE_16x16_INTEL                                 0x0
#define CL_ME_MB_TYPE_8x8_INTEL                                   0x1
#define CL_ME_MB_TYPE_4x4_INTEL                                   0x2

#define CL_ME_SUBPIXEL_MODE_INTEGER_INTEL                         0x0
#define CL_ME_SUBPIXEL_MODE_HPEL_INTEL                            0x1
#define CL_ME_SUBPIXEL_MODE_QPEL_INTEL                            0x2

#define CL_ME_SAD_ADJUST_MODE_NONE_INTEL                          0x0
#define CL_ME_SAD_ADJUST_MODE_HAAR_INTEL                          0x1

#define CL_ME_SEARCH_PATH_RADIUS_2_2_INTEL                        0x0
#define CL_ME_SEARCH_PATH_RADIUS_4_4_INTEL                        0x1
#define CL_ME_SEARCH_PATH_RADIUS_16_12_INTEL                      0x5

#define CL_ME_SKIP_BLOCK_TYPE_16x16_INTEL                         0x0
#define CL_ME_CHROMA_INTRA_PREDICT_ENABLED_INTEL                  0x1
#define CL_ME_LUMA_INTRA_PREDICT_ENABLED_INTEL                    0x2
#define CL_ME_SKIP_BLOCK_TYPE_8x8_INTEL                           0x4

#define CL_ME_FORWARD_INPUT_MODE_INTEL                            0x1
# 定义 Intel 后向输入模式常量
#define CL_ME_BACKWARD_INPUT_MODE_INTEL                           0x2
# 定义 Intel 双向输入模式常量
#define CL_ME_BIDIRECTION_INPUT_MODE_INTEL                        0x3

# 定义 Intel 双向权重为四分之一常量
#define CL_ME_BIDIR_WEIGHT_QUARTER_INTEL                          16
# 定义 Intel 双向权重为三分之一常量
#define CL_ME_BIDIR_WEIGHT_THIRD_INTEL                            21
# 定义 Intel 双向权重为一半常量
#define CL_ME_BIDIR_WEIGHT_HALF_INTEL                             32
# 定义 Intel 双向权重为三分之二常量
#define CL_ME_BIDIR_WEIGHT_TWO_THIRD_INTEL                        43
# 定义 Intel 双向权重为四分之三常量
#define CL_ME_BIDIR_WEIGHT_THREE_QUARTER_INTEL                    48

# 定义 Intel 运动估计代价惩罚为无常量
#define CL_ME_COST_PENALTY_NONE_INTEL                             0x0
# 定义 Intel 运动估计代价惩罚为低常量
#define CL_ME_COST_PENALTY_LOW_INTEL                              0x1
# 定义 Intel 运动估计代价惩罚为正常常量
#define CL_ME_COST_PENALTY_NORMAL_INTEL                           0x2
# 定义 Intel 运动估计代价惩罚为高常量
#define CL_ME_COST_PENALTY_HIGH_INTEL                             0x3

# 定义 Intel 运动估计代价精度为四分之一像素常量
#define CL_ME_COST_PRECISION_QPEL_INTEL                           0x0
# 定义 Intel 运动估计代价精度为半像素常量
#define CL_ME_COST_PRECISION_HPEL_INTEL                           0x1
# 定义 Intel 运动估计代价精度为像素常量
#define CL_ME_COST_PRECISION_PEL_INTEL                            0x2
# 定义 Intel 运动估计代价精度为双像素常量
#define CL_ME_COST_PRECISION_DPEL_INTEL                           0x3

# 定义 Intel 亮度预测模式为垂直常量
#define CL_ME_LUMA_PREDICTOR_MODE_VERTICAL_INTEL                  0x0
# 定义 Intel 亮度预测模式为水平常量
#define CL_ME_LUMA_PREDICTOR_MODE_HORIZONTAL_INTEL                0x1
# 定义 Intel 亮度预测模式为直流常量
#define CL_ME_LUMA_PREDICTOR_MODE_DC_INTEL                        0x2
# 定义 Intel 亮度预测模式为左下对角线常量
#define CL_ME_LUMA_PREDICTOR_MODE_DIAGONAL_DOWN_LEFT_INTEL        0x3

# 定义 Intel 亮度预测模式为右下对角线常量
#define CL_ME_LUMA_PREDICTOR_MODE_DIAGONAL_DOWN_RIGHT_INTEL       0x4
# 定义 Intel 亮度预测模式为平面常量
#define CL_ME_LUMA_PREDICTOR_MODE_PLANE_INTEL                     0x4
# 定义 Intel 亮度预测模式为右垂直常量
#define CL_ME_LUMA_PREDICTOR_MODE_VERTICAL_RIGHT_INTEL            0x5
# 定义 Intel 亮度预测模式为下水平常量
#define CL_ME_LUMA_PREDICTOR_MODE_HORIZONTAL_DOWN_INTEL           0x6
# 定义 Intel 亮度预测模式为左垂直常量
#define CL_ME_LUMA_PREDICTOR_MODE_VERTICAL_LEFT_INTEL             0x7
# 定义 Intel 亮度预测模式为上水平常量
#define CL_ME_LUMA_PREDICTOR_MODE_HORIZONTAL_UP_INTEL             0x8

# 定义 Intel 色度预测模式为直流常量
#define CL_ME_CHROMA_PREDICTOR_MODE_DC_INTEL                      0x0
# 定义 Intel 色度预测模式为水平常量
#define CL_ME_CHROMA_PREDICTOR_MODE_HORIZONTAL_INTEL              0x1
# 定义 Intel 色度预测模式为垂直常量
#define CL_ME_CHROMA_PREDICTOR_MODE_VERTICAL_INTEL                0x2
// 定义了 CHROMA_PREDICTOR_MODE_PLANE_INTEL 常量，值为 0x3
#define CL_ME_CHROMA_PREDICTOR_MODE_PLANE_INTEL                   0x3

// 定义了 ME_VERSION_INTEL 常量，值为 0x407E
#define CL_DEVICE_ME_VERSION_INTEL                                0x407E

// 定义了 ME_VERSION_LEGACY_INTEL 常量，值为 0x0
#define CL_ME_VERSION_LEGACY_INTEL                                0x0

// 定义了 ME_VERSION_ADVANCED_VER_1_INTEL 常量，值为 0x1
#define CL_ME_VERSION_ADVANCED_VER_1_INTEL                        0x1

// 定义了 ME_VERSION_ADVANCED_VER_2_INTEL 常量，值为 0x2
#define CL_ME_VERSION_ADVANCED_VER_2_INTEL                        0x2

// 创建一个名为 clCreateAcceleratorINTEL 的函数，用于创建 Intel 加速器
extern CL_API_ENTRY cl_accelerator_intel CL_API_CALL
clCreateAcceleratorINTEL(
    cl_context                   context,                     // 上下文
    cl_accelerator_type_intel    accelerator_type,            // 加速器类型
    size_t                       descriptor_size,             // 描述符大小
    const void*                  descriptor,                  // 描述符
    cl_int*                      errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;  // 错误码返回值

// 定义了一个函数指针类型 clCreateAcceleratorINTEL_fn，用于指向 clCreateAcceleratorINTEL 函数
typedef CL_API_ENTRY cl_accelerator_intel (CL_API_CALL *clCreateAcceleratorINTEL_fn)(
    cl_context                   context,
    cl_accelerator_type_intel    accelerator_type,
    size_t                       descriptor_size,
    const void*                  descriptor,
    cl_int*                      errcode_ret) CL_EXT_SUFFIX__VERSION_1_2;

// 获取 Intel 加速器的信息
extern CL_API_ENTRY cl_int CL_API_CALL
clGetAcceleratorInfoINTEL(
    cl_accelerator_intel         accelerator,                 // 加速器
    cl_accelerator_info_intel    param_name,                  // 参数名
    size_t                       param_value_size,            // 参数值大小
    void*                        param_value,                 // 参数值
    size_t*                      param_value_size_ret) CL_EXT_SUFFIX__VERSION_1_2;  // 参数值大小返回值

// 定义了一个函数指针类型 clGetAcceleratorInfoINTEL_fn，用于指向 clGetAcceleratorInfoINTEL 函数
typedef CL_API_ENTRY cl_int (CL_API_CALL *clGetAcceleratorInfoINTEL_fn)(
    cl_accelerator_intel         accelerator,
    cl_accelerator_info_intel    param_name,
    size_t                       param_value_size,
    void*                        param_value,
    size_t*                      param_value_size_ret) CL_EXT_SUFFIX__VERSION_1_2;

// 保留 Intel 加速器
extern CL_API_ENTRY cl_int CL_API_CALL
clRetainAcceleratorINTEL(
    cl_accelerator_intel         accelerator) CL_EXT_SUFFIX__VERSION_1_2;  // 加速器

// 定义了一个函数指针类型 clRetainAcceleratorINTEL_fn，用于指向 clRetainAcceleratorINTEL 函数
typedef CL_API_ENTRY cl_int (CL_API_CALL *clRetainAcceleratorINTEL_fn)(
    # 定义一个名为cl_accelerator_intel的变量，其类型为accelerator，后面的CL_EXT_SUFFIX__VERSION_1_2可能是变量的后缀或者版本信息
# 定义了一个名为 clReleaseAcceleratorINTEL 的函数，用于释放加速器资源
extern CL_API_ENTRY cl_int CL_API_CALL
clReleaseAcceleratorINTEL(
    cl_accelerator_intel         accelerator) CL_EXT_SUFFIX__VERSION_1_2;

# 定义了一个名为 clReleaseAcceleratorINTEL_fn 的函数指针类型，用于释放加速器资源
typedef CL_API_ENTRY cl_int (CL_API_CALL *clReleaseAcceleratorINTEL_fn)(
    cl_accelerator_intel         accelerator) CL_EXT_SUFFIX__VERSION_1_2;

# 定义了 cl_intel_simultaneous_sharing 扩展
#define cl_intel_simultaneous_sharing 1

# 定义了用于 cl_intel_simultaneous_sharing 扩展的常量
#define CL_DEVICE_SIMULTANEOUS_INTEROPS_INTEL            0x4104
#define CL_DEVICE_NUM_SIMULTANEOUS_INTEROPS_INTEL        0x4105

# 定义了 cl_intel_egl_image_yuv 扩展
#define cl_intel_egl_image_yuv 1

# 定义了用于 cl_intel_egl_image_yuv 扩展的常量
#define CL_EGL_YUV_PLANE_INTEL                           0x4107

# 定义了 cl_intel_packed_yuv 扩展
#define cl_intel_packed_yuv 1

# 定义了用于 cl_intel_packed_yuv 扩展的常量
#define CL_YUYV_INTEL                                    0x4076
#define CL_UYVY_INTEL                                    0x4077
#define CL_YVYU_INTEL                                    0x4078
#define CL_VYUY_INTEL                                    0x4079

# 定义了 cl_intel_required_subgroup_size 扩展
#define cl_intel_required_subgroup_size 1

# 定义了用于 cl_intel_required_subgroup_size 扩展的常量
#define CL_DEVICE_SUB_GROUP_SIZES_INTEL                  0x4108
#define CL_KERNEL_SPILL_MEM_SIZE_INTEL                   0x4109
#define CL_KERNEL_COMPILE_SUB_GROUP_SIZE_INTEL           0x410A

# 定义了 cl_intel_driver_diagnostics 扩展
#define cl_intel_driver_diagnostics 1

# 定义了用于 cl_intel_driver_diagnostics 扩展的常量
typedef cl_uint cl_diagnostics_verbose_level;
#define CL_CONTEXT_SHOW_DIAGNOSTICS_INTEL                0x4106
#define CL_CONTEXT_DIAGNOSTICS_LEVEL_ALL_INTEL           ( 0xff )
#define CL_CONTEXT_DIAGNOSTICS_LEVEL_GOOD_INTEL          ( 1 )
# 定义 Intel OpenCL 扩展中的上下文诊断级别常量
#define CL_CONTEXT_DIAGNOSTICS_LEVEL_BAD_INTEL           ( 1 << 1 )
#define CL_CONTEXT_DIAGNOSTICS_LEVEL_NEUTRAL_INTEL       ( 1 << 2 )

# 定义 cl_intel_planar_yuv 扩展中的常量
#define CL_NV12_INTEL                                       0x410E
#define CL_MEM_NO_ACCESS_INTEL                              ( 1 << 24 )
#define CL_MEM_ACCESS_FLAGS_UNRESTRICTED_INTEL              ( 1 << 25 )
#define CL_DEVICE_PLANAR_YUV_MAX_WIDTH_INTEL                0x417E
#define CL_DEVICE_PLANAR_YUV_MAX_HEIGHT_INTEL               0x417F

# 定义 cl_intel_device_side_avc_motion_estimation 扩展中的常量
#define CL_DEVICE_AVC_ME_VERSION_INTEL                      0x410B
#define CL_DEVICE_AVC_ME_SUPPORTS_TEXTURE_SAMPLER_USE_INTEL 0x410C
#define CL_DEVICE_AVC_ME_SUPPORTS_PREEMPTION_INTEL          0x410D
#define CL_AVC_ME_VERSION_0_INTEL                           0x0;  // 不支持
#define CL_AVC_ME_VERSION_1_INTEL                           0x1;  // 第一个支持的版本
#define CL_AVC_ME_MAJOR_16x16_INTEL                         0x0
#define CL_AVC_ME_MAJOR_16x8_INTEL                          0x1
#define CL_AVC_ME_MAJOR_8x16_INTEL                          0x2
#define CL_AVC_ME_MAJOR_8x8_INTEL                           0x3
#define CL_AVC_ME_MINOR_8x8_INTEL                           0x0
#define CL_AVC_ME_MINOR_8x4_INTEL                           0x1
#define CL_AVC_ME_MINOR_4x8_INTEL                           0x2
#define CL_AVC_ME_MINOR_4x4_INTEL                           0x3
#define CL_AVC_ME_MAJOR_FORWARD_INTEL                       0x0
#define CL_AVC_ME_MAJOR_BACKWARD_INTEL                      0x1
#define CL_AVC_ME_MAJOR_BIDIRECTIONAL_INTEL                 0x2
#define CL_AVC_ME_PARTITION_MASK_ALL_INTEL                  0x0
#define CL_AVC_ME_PARTITION_MASK_16x16_INTEL                0x7E
# 定义不同的 AVC ME 分区掩码，用于运动估计
#define CL_AVC_ME_PARTITION_MASK_16x8_INTEL                 0x7D
#define CL_AVC_ME_PARTITION_MASK_8x16_INTEL                 0x7B
#define CL_AVC_ME_PARTITION_MASK_8x8_INTEL                  0x77
#define CL_AVC_ME_PARTITION_MASK_8x4_INTEL                  0x6F
#define CL_AVC_ME_PARTITION_MASK_4x8_INTEL                  0x5F
#define CL_AVC_ME_PARTITION_MASK_4x4_INTEL                  0x3F

# 定义不同的 AVC ME 搜索窗口大小，用于运动估计
#define CL_AVC_ME_SEARCH_WINDOW_EXHAUSTIVE_INTEL            0x0
#define CL_AVC_ME_SEARCH_WINDOW_SMALL_INTEL                 0x1
#define CL_AVC_ME_SEARCH_WINDOW_TINY_INTEL                  0x2
#define CL_AVC_ME_SEARCH_WINDOW_EXTRA_TINY_INTEL            0x3
#define CL_AVC_ME_SEARCH_WINDOW_DIAMOND_INTEL               0x4
#define CL_AVC_ME_SEARCH_WINDOW_LARGE_DIAMOND_INTEL         0x5
#define CL_AVC_ME_SEARCH_WINDOW_RESERVED0_INTEL             0x6
#define CL_AVC_ME_SEARCH_WINDOW_RESERVED1_INTEL             0x7
#define CL_AVC_ME_SEARCH_WINDOW_CUSTOM_INTEL                0x8
#define CL_AVC_ME_SEARCH_WINDOW_16x12_RADIUS_INTEL          0x9
#define CL_AVC_ME_SEARCH_WINDOW_4x4_RADIUS_INTEL            0x2
#define CL_AVC_ME_SEARCH_WINDOW_2x2_RADIUS_INTEL            0xa

# 定义不同的 AVC ME SAD 调整模式，用于运动估计
#define CL_AVC_ME_SAD_ADJUST_MODE_NONE_INTEL                0x0
#define CL_AVC_ME_SAD_ADJUST_MODE_HAAR_INTEL                0x2

# 定义不同的 AVC ME 亚像素模式，用于运动估计
#define CL_AVC_ME_SUBPIXEL_MODE_INTEGER_INTEL               0x0
#define CL_AVC_ME_SUBPIXEL_MODE_HPEL_INTEL                  0x1
#define CL_AVC_ME_SUBPIXEL_MODE_QPEL_INTEL                  0x3

# 定义不同的 AVC ME 成本精度，用于运动估计
#define CL_AVC_ME_COST_PRECISION_QPEL_INTEL                 0x0
#define CL_AVC_ME_COST_PRECISION_HPEL_INTEL                 0x1
#define CL_AVC_ME_COST_PRECISION_PEL_INTEL                  0x2
#define CL_AVC_ME_COST_PRECISION_DPEL_INTEL                 0x3

# 定义不同的 AVC ME 双向权重，用于运动估计
#define CL_AVC_ME_BIDIR_WEIGHT_QUARTER_INTEL                0x10
#define CL_AVC_ME_BIDIR_WEIGHT_THIRD_INTEL                  0x15
#define CL_AVC_ME_BIDIR_WEIGHT_HALF_INTEL                   0x20
#define CL_AVC_ME_BIDIR_WEIGHT_TWO_THIRD_INTEL              0x2B
# 定义 AVC ME 双向加权因子为三分之四
#define CL_AVC_ME_BIDIR_WEIGHT_THREE_QUARTER_INTEL          0x30

# 定义 AVC ME 边界到达左侧
#define CL_AVC_ME_BORDER_REACHED_LEFT_INTEL                 0x0
# 定义 AVC ME 边界到达右侧
#define CL_AVC_ME_BORDER_REACHED_RIGHT_INTEL                0x2
# 定义 AVC ME 边界到达顶部
#define CL_AVC_ME_BORDER_REACHED_TOP_INTEL                  0x4
# 定义 AVC ME 边界到达底部
#define CL_AVC_ME_BORDER_REACHED_BOTTOM_INTEL               0x8

# 定义 AVC ME 跳过块分区为 16x16
#define CL_AVC_ME_SKIP_BLOCK_PARTITION_16x16_INTEL          0x0
# 定义 AVC ME 跳过块分区为 8x8
#define CL_AVC_ME_SKIP_BLOCK_PARTITION_8x8_INTEL            0x4000

# 定义 AVC ME 16x16 前向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_16x16_FORWARD_ENABLE_INTEL     ( 0x1 << 24 )
# 定义 AVC ME 16x16 后向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_16x16_BACKWARD_ENABLE_INTEL    ( 0x2 << 24 )
# 定义 AVC ME 16x16 双向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_16x16_DUAL_ENABLE_INTEL        ( 0x3 << 24 )
# 定义 AVC ME 8x8 前向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_FORWARD_ENABLE_INTEL       ( 0x55 << 24 )
# 定义 AVC ME 8x8 后向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_BACKWARD_ENABLE_INTEL      ( 0xAA << 24 )
# 定义 AVC ME 8x8 双向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_DUAL_ENABLE_INTEL          ( 0xFF << 24 )
# 定义 AVC ME 8x8_0 前向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_0_FORWARD_ENABLE_INTEL     ( 0x1 << 24 )
# 定义 AVC ME 8x8_0 后向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_0_BACKWARD_ENABLE_INTEL    ( 0x2 << 24 )
# 定义 AVC ME 8x8_1 前向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_1_FORWARD_ENABLE_INTEL     ( 0x1 << 26 )
# 定义 AVC ME 8x8_1 后向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_1_BACKWARD_ENABLE_INTEL    ( 0x2 << 26 )
# 定义 AVC ME 8x8_2 前向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_2_FORWARD_ENABLE_INTEL     ( 0x1 << 28 )
# 定义 AVC ME 8x8_2 后向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_2_BACKWARD_ENABLE_INTEL    ( 0x2 << 28 )
# 定义 AVC ME 8x8_3 前向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_3_FORWARD_ENABLE_INTEL     ( 0x1 << 30 )
# 定义 AVC ME 8x8_3 后向跳过块启用
#define CL_AVC_ME_SKIP_BLOCK_8x8_3_BACKWARD_ENABLE_INTEL    ( 0x2 << 30 )

# 定义 AVC ME 基于块的 4x4 跳过
#define CL_AVC_ME_BLOCK_BASED_SKIP_4x4_INTEL                0x00
# 定义 AVC ME 基于块的 8x8 跳过
#define CL_AVC_ME_BLOCK_BASED_SKIP_8x8_INTEL                0x80

# 定义 AVC ME 16x16 帧内预测
#define CL_AVC_ME_INTRA_16x16_INTEL                         0x0
# 定义 AVC ME 8x8 帧内预测
#define CL_AVC_ME_INTRA_8x8_INTEL                           0x1
# 定义 AVC ME 4x4 帧内预测
#define CL_AVC_ME_INTRA_4x4_INTEL                           0x2

# 定义 AVC ME 16x16 亮度帧内分区掩码
#define CL_AVC_ME_INTRA_LUMA_PARTITION_MASK_16x16_INTEL     0x6
# 定义 AVC ME 8x8 亮度帧内分区掩码
#define CL_AVC_ME_INTRA_LUMA_PARTITION_MASK_8x8_INTEL       0x5
# 定义 AVC ME（Advanced Video Coding Motion Estimation）中的帧类型和预测模式的掩码
#define CL_AVC_ME_INTRA_LUMA_PARTITION_MASK_4x4_INTEL       0x3

# 定义 AVC ME 中用于指示左侧邻居是否可用的掩码
#define CL_AVC_ME_INTRA_NEIGHBOR_LEFT_MASK_ENABLE_INTEL         0x60
# 定义 AVC ME 中用于指示上方邻居是否可用的掩码
#define CL_AVC_ME_INTRA_NEIGHBOR_UPPER_MASK_ENABLE_INTEL        0x10
# 定义 AVC ME 中用于指示右上方邻居是否可用的掩码
#define CL_AVC_ME_INTRA_NEIGHBOR_UPPER_RIGHT_MASK_ENABLE_INTEL  0x8
# 定义 AVC ME 中用于指示左上方邻居是否可用的掩码
#define CL_AVC_ME_INTRA_NEIGHBOR_UPPER_LEFT_MASK_ENABLE_INTEL   0x4

# 定义 AVC ME 中的亮度预测模式：垂直
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_VERTICAL_INTEL            0x0
# 定义 AVC ME 中的亮度预测模式：水平
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_HORIZONTAL_INTEL          0x1
# 定义 AVC ME 中的亮度预测模式：DC
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_DC_INTEL                  0x2
# 定义 AVC ME 中的亮度预测模式：左下对角线
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_DIAGONAL_DOWN_LEFT_INTEL  0x3
# 定义 AVC ME 中的亮度预测模式：右下对角线
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_DIAGONAL_DOWN_RIGHT_INTEL 0x4
# 定义 AVC ME 中的亮度预测模式：平面
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_PLANE_INTEL               0x4
# 定义 AVC ME 中的亮度预测模式：右上对角线
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_VERTICAL_RIGHT_INTEL      0x5
# 定义 AVC ME 中的亮度预测模式：下方水平
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_HORIZONTAL_DOWN_INTEL     0x6
# 定义 AVC ME 中的亮度预测模式：左上对角线
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_VERTICAL_LEFT_INTEL       0x7
# 定义 AVC ME 中的亮度预测模式：上方垂直
#define CL_AVC_ME_LUMA_PREDICTOR_MODE_HORIZONTAL_UP_INTEL       0x8
# 定义 AVC ME 中的色度预测模式：DC
#define CL_AVC_ME_CHROMA_PREDICTOR_MODE_DC_INTEL                0x0
# 定义 AVC ME 中的色度预测模式：水平
#define CL_AVC_ME_CHROMA_PREDICTOR_MODE_HORIZONTAL_INTEL        0x1
# 定义 AVC ME 中的色度预测模式：垂直
#define CL_AVC_ME_CHROMA_PREDICTOR_MODE_VERTICAL_INTEL          0x2
# 定义 AVC ME 中的色度预测模式：平面
#define CL_AVC_ME_CHROMA_PREDICTOR_MODE_PLANE_INTEL             0x3

# 定义 AVC ME 中的帧类型：向前帧
#define CL_AVC_ME_FRAME_FORWARD_INTEL                       0x1
# 定义 AVC ME 中的帧类型：向后帧
#define CL_AVC_ME_FRAME_BACKWARD_INTEL                      0x2
# 定义 AVC ME 中的帧类型：双向帧
#define CL_AVC_ME_FRAME_DUAL_INTEL                          0x3

# 定义 AVC ME 中的切片类型：预测
#define CL_AVC_ME_SLICE_TYPE_PRED_INTEL                     0x0
# 定义 AVC ME 中的切片类型：双向预测
#define CL_AVC_ME_SLICE_TYPE_BPRED_INTEL                    0x1
# 定义 AVC ME 中的切片类型：帧内预测
#define CL_AVC_ME_SLICE_TYPE_INTRA_INTEL                    0x2

# 定义 AVC ME 中的隔行扫描类型：顶部场
#define CL_AVC_ME_INTERLACED_SCAN_TOP_FIELD_INTEL           0x0
# 定义 AVC ME 中的隔行扫描类型：底部场
#define CL_AVC_ME_INTERLACED_SCAN_BOTTOM_FIELD_INTEL        0x1

# 如果是 C++ 代码，则结束 extern "C" 块
#ifdef __cplusplus
}
#endif

# 结束条件编译指令，防止重复包含
#endif /* __CL_EXT_INTEL_H */
```