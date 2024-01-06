# `PowerInfer\ggml-cuda.h`

```
#pragma once
// 防止头文件被多次包含

#include "ggml.h"
#include "ggml-backend.h"
// 包含其他头文件

#ifdef GGML_USE_HIPBLAS
#define GGML_CUDA_NAME "ROCm"
#define GGML_CUBLAS_NAME "hipBLAS"
#else
#define GGML_CUDA_NAME "CUDA"
#define GGML_CUBLAS_NAME "cuBLAS"
#endif
// 根据条件定义不同的宏

#ifdef  __cplusplus
extern "C" {
#endif
// 如果是 C++ 环境，则使用 C 语言的语法

#define GGML_CUDA_MAX_DEVICES       16
// 定义 CUDA 最大设备数量

// Always success. To check if CUDA is actually loaded, use `ggml_cublas_loaded`.
// 总是成功。要检查 CUDA 是否真正加载，请使用 `ggml_cublas_loaded`。
// 初始化 cuBLAS 库
GGML_API void   ggml_init_cublas(void);

// 如果有可用的 CUDA 设备并且 cublas 成功加载，则返回 true；否则返回 false
GGML_API bool   ggml_cublas_loaded(void);

// 在主机上分配 CUDA 内存
GGML_API void * ggml_cuda_host_malloc(size_t size);
// 释放主机上的 CUDA 内存
GGML_API void   ggml_cuda_host_free(void * ptr);

// 检查是否可以对两个张量进行矩阵乘法运算
GGML_API bool   ggml_cuda_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 设置张量的分割
GGML_API void   ggml_cuda_set_tensor_split(const float * tensor_split);
// 对张量进行变换
GGML_API void   ggml_cuda_transform_tensor(void * data, struct ggml_tensor * tensor);
// 分配张量的 CUDA 内存
GGML_API void   ggml_cuda_alloc_tensor(struct ggml_tensor * tensor);
// 释放张量的 CUDA 内存
GGML_API void   ggml_cuda_free_data(struct ggml_tensor * tensor);
// 在一维数组之间进行数据拷贝
GGML_API void   ggml_cuda_cpy_1d(struct ggml_tensor * dst, const struct ggml_tensor * src);
// 检查两个 short 类型数组是否相等
GGML_API bool   debug_equal(short *a, short *b);
// 获取张量数据的指针数组
GGML_API void **ggml_cuda_get_data_pp(struct ggml_tensor * tensor);

// 分配张量的 CUDA 缓冲区
GGML_API void   ggml_cuda_assign_buffers(struct ggml_tensor * tensor);
// 分配张量的 CUDA 缓冲区，不使用临时缓冲区
GGML_API void   ggml_cuda_assign_buffers_no_scratch(struct ggml_tensor * tensor);
// 强制就地分配张量的 CUDA 缓冲区
GGML_API void   ggml_cuda_assign_buffers_force_inplace(struct ggml_tensor * tensor);
// 为 CUDA 分配缓冲区，但不进行分配
GGML_API void   ggml_cuda_assign_buffers_no_alloc(struct ggml_tensor * tensor);

// 为 CUDA 分配缓冲区，并指定偏移量
GGML_API void   ggml_cuda_assign_scratch_offset(struct ggml_tensor * tensor, size_t offset);

// 将数据从主机内存复制到 CUDA 设备
GGML_API void   ggml_cuda_copy_to_device(struct ggml_tensor * tensor);

// 设置主 CUDA 设备
GGML_API void   ggml_cuda_set_main_device(int main_device);

// 设置是否在矩阵乘法中使用量化
GGML_API void   ggml_cuda_set_mul_mat_q(bool mul_mat_q);

// 设置 CUDA 临时缓冲区的大小
GGML_API void   ggml_cuda_set_scratch_size(size_t scratch_size);

// 释放 CUDA 临时缓冲区
GGML_API void   ggml_cuda_free_scratch(void);

// 在 CUDA 设备上计算前向传播
GGML_API bool   ggml_cuda_compute_forward(struct ggml_compute_params * params, struct ggml_tensor * tensor);

// 获取 CUDA 设备数量
GGML_API int    ggml_cuda_get_device_count(void);

// 获取指定 CUDA 设备的描述信息
GGML_API void   ggml_cuda_get_device_description(int device, char * description, size_t description_size);

// 获取指定 CUDA 设备的空闲内存大小
GGML_API size_t ggml_cuda_get_free_memory(int device);

// 设置 CUDA 设备的常量值，用于稀疏预测阈值
GGML_API void   ggml_cuda_set_device_constants(float sparse_pred_threshold);

// 初始化 CUDA 后端
GGML_API ggml_backend_t ggml_backend_cuda_init(void); // TODO: take a list of devices to use
#ifdef  __cplusplus
// 如果是 C++ 环境
}
#endif
// 结束 C++ 环境的条件编译
```