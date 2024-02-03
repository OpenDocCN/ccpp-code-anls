# `ggml\src\ggml-cuda.h`

```cpp
#pragma once

#include "ggml.h"  // 包含 ggml 库的头文件
#include "ggml-backend.h"  // 包含 ggml 后端的头文件

#ifdef GGML_USE_HIPBLAS
#define GGML_CUDA_NAME "ROCm"  // 如果使用 HIPBLAS，则定义 CUDA 名称为 "ROCm"
#define GGML_CUBLAS_NAME "hipBLAS"  // 如果使用 HIPBLAS，则定义 CUBLAS 名称为 "hipBLAS"
#else
#define GGML_CUDA_NAME "CUDA"  // 如果不使用 HIPBLAS，则定义 CUDA 名称为 "CUDA"
#define GGML_CUBLAS_NAME "cuBLAS"  // 如果不使用 HIPBLAS，则定义 CUBLAS 名称为 "cuBLAS"
#endif

#ifdef  __cplusplus
extern "C" {
#endif

#define GGML_CUDA_MAX_DEVICES       16  // 定义 CUDA 最大设备数为 16

// 初始化 cublas，总是成功。要检查 CUDA 是否真正加载，使用 `ggml_cublas_loaded`。
GGML_API void   ggml_init_cublas(void);

// 如果有可用的 CUDA 设备并且 cublas 成功加载，则返回 `true`；否则返回 `false`。
GGML_API bool   ggml_cublas_loaded(void);

GGML_API void * ggml_cuda_host_malloc(size_t size);  // 分配 CUDA 主机内存
GGML_API void   ggml_cuda_host_free(void * ptr);  // 释放 CUDA 主机内存

GGML_API bool   ggml_cuda_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);  // 检查是否可以对矩阵进行乘法运算
GGML_API void   ggml_cuda_set_tensor_split(const float * tensor_split);  // 设置张量分割
GGML_API void   ggml_cuda_transform_tensor(void * data, struct ggml_tensor * tensor);  // 转换张量
GGML_API void   ggml_cuda_free_data(struct ggml_tensor * tensor);  // 释放张量数据

GGML_API void   ggml_cuda_assign_buffers(struct ggml_tensor * tensor);  // 分配缓冲区
GGML_API void   ggml_cuda_assign_buffers_no_scratch(struct ggml_tensor * tensor);  // 分配不带临时缓冲区的缓冲区
GGML_API void   ggml_cuda_assign_buffers_force_inplace(struct ggml_tensor * tensor);  // 强制原地分配缓冲区

GGML_API void   ggml_cuda_assign_buffers_no_alloc(struct ggml_tensor * tensor);  // 分配不分配内存的缓冲区
GGML_API void   ggml_cuda_assign_scratch_offset(struct ggml_tensor * tensor, size_t offset);  // 分配临时缓冲区偏移量
GGML_API void   ggml_cuda_copy_to_device(struct ggml_tensor * tensor);  // 将数据复制到设备

GGML_API void   ggml_cuda_set_main_device(int main_device);  // 设置主设备
GGML_API void   ggml_cuda_set_mul_mat_q(bool mul_mat_q);  // 设置矩阵乘法 Q
GGML_API void   ggml_cuda_set_scratch_size(size_t scratch_size);  // 设置临时缓冲区大小
GGML_API void   ggml_cuda_free_scratch(void);  // 释放临时缓冲区
GGML_API bool   ggml_cuda_compute_forward(struct ggml_compute_params * params, struct ggml_tensor * tensor);  // 计算前向传播

GGML_API int    ggml_cuda_get_device_count(void);  // 获取设备数量
// 获取指定设备的描述信息
GGML_API void ggml_cuda_get_device_description(int device, char * description, size_t description_size);

// CUDA 后端初始化函数，返回后端对象
GGML_API ggml_backend_t ggml_backend_cuda_init(int device);

// 判断后端对象是否为 CUDA 后端
GGML_API bool ggml_backend_is_cuda(ggml_backend_t backend);

// 获取后端对象对应的 CUDA 设备编号
GGML_API int ggml_backend_cuda_get_device(ggml_backend_t backend);

// 获取指定设备的缓冲区类型
GGML_API ggml_backend_buffer_type_t ggml_backend_cuda_buffer_type(int device);

// 获取用于 CPU 后端的固定主机缓冲区类型，用于在 CPU 和 GPU 之间进行更快的复制
GGML_API ggml_backend_buffer_type_t ggml_backend_cuda_host_buffer_type(void);

#ifdef  __cplusplus
}
#endif
```