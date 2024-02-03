# `PowerInfer\ggml-cuda.h`

```cpp
#pragma once

#include "ggml.h"  // 包含 ggml.h 头文件
#include "ggml-backend.h"  // 包含 ggml-backend.h 头文件

#ifdef GGML_USE_HIPBLAS
#define GGML_CUDA_NAME "ROCm"  // 如果使用 HIPBLAS，则定义 GGML_CUDA_NAME 为 "ROCm"
#define GGML_CUBLAS_NAME "hipBLAS"  // 如果使用 HIPBLAS，则定义 GGML_CUBLAS_NAME 为 "hipBLAS"
#else
#define GGML_CUDA_NAME "CUDA"  // 如果不使用 HIPBLAS，则定义 GGML_CUDA_NAME 为 "CUDA"
#define GGML_CUBLAS_NAME "cuBLAS"  // 如果不使用 HIPBLAS，则定义 GGML_CUBLAS_NAME 为 "cuBLAS"
#endif

#ifdef  __cplusplus
extern "C" {
#endif

#define GGML_CUDA_MAX_DEVICES       16  // 定义 GGML_CUDA_MAX_DEVICES 为 16，表示 CUDA 最大设备数

// Always success. To check if CUDA is actually loaded, use `ggml_cublas_loaded`.
GGML_API void   ggml_init_cublas(void);  // 初始化 cublas

// Returns `true` if there are available CUDA devices and cublas loads successfully; otherwise, it returns `false`.
GGML_API bool   ggml_cublas_loaded(void);  // 返回是否成功加载 CUDA 设备和 cublas

GGML_API void * ggml_cuda_host_malloc(size_t size);  // 分配 CUDA 主机内存
GGML_API void   ggml_cuda_host_free(void * ptr);  // 释放 CUDA 主机内存

GGML_API bool   ggml_cuda_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);  // 检查是否可以对矩阵进行乘法运算
GGML_API void   ggml_cuda_set_tensor_split(const float * tensor_split);  // 设置张量分割
GGML_API void   ggml_cuda_transform_tensor(void * data, struct ggml_tensor * tensor);  // 转换张量
GGML_API void   ggml_cuda_alloc_tensor(struct ggml_tensor * tensor);  // 分配张量内存
GGML_API void   ggml_cuda_free_data(struct ggml_tensor * tensor);  // 释放张量数据
GGML_API void   ggml_cuda_cpy_1d(struct ggml_tensor * dst, const struct ggml_tensor * src);  // 复制一维张量数据
GGML_API bool   debug_equal(short *a, short *b);  // 调试用，比较两个 short 数组是否相等
GGML_API void **ggml_cuda_get_data_pp(struct ggml_tensor * tensor);  // 获取张量数据指针的指针

GGML_API void   ggml_cuda_assign_buffers(struct ggml_tensor * tensor);  // 分配张量缓冲区
GGML_API void   ggml_cuda_assign_buffers_no_scratch(struct ggml_tensor * tensor);  // 分配张量缓冲区，不使用临时缓冲区
GGML_API void   ggml_cuda_assign_buffers_force_inplace(struct ggml_tensor * tensor);  // 强制就地分配张量缓冲区

GGML_API void   ggml_cuda_assign_buffers_no_alloc(struct ggml_tensor * tensor);  // 分配张量缓冲区，不分配内存
GGML_API void   ggml_cuda_assign_scratch_offset(struct ggml_tensor * tensor, size_t offset);  // 分配临时缓冲区偏移量
GGML_API void   ggml_cuda_copy_to_device(struct ggml_tensor * tensor);  // 将数据复制到设备

GGML_API void   ggml_cuda_set_main_device(int main_device);  // 设置主设备
GGML_API void   ggml_cuda_set_mul_mat_q(bool mul_mat_q);  // 设置是否进行矩阵乘法
GGML_API void   ggml_cuda_set_scratch_size(size_t scratch_size);  // 设置临时缓冲区大小
# 释放 CUDA 的临时内存
GGML_API void   ggml_cuda_free_scratch(void);

# 使用 CUDA 计算前向传播
GGML_API bool   ggml_cuda_compute_forward(struct ggml_compute_params * params, struct ggml_tensor * tensor);

# 获取 CUDA 设备数量
GGML_API int    ggml_cuda_get_device_count(void);

# 获取 CUDA 设备描述信息
GGML_API void   ggml_cuda_get_device_description(int device, char * description, size_t description_size);

# 获取 CUDA 设备的空闲内存大小
GGML_API size_t ggml_cuda_get_free_memory(int device);

# 设置 CUDA 设备的常量
GGML_API void   ggml_cuda_set_device_constants(float sparse_pred_threshold);

# 初始化 CUDA 后端
GGML_API ggml_backend_t ggml_backend_cuda_init(void); // TODO: take a list of devices to use

# 结束 C++ 的声明
#ifdef  __cplusplus
}
#endif
```