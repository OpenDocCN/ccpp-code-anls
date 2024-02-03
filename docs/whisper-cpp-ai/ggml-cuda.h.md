# `whisper.cpp\ggml-cuda.h`

```cpp
#pragma once

#include "ggml.h"
#include "ggml-backend.h"

#ifdef GGML_USE_HIPBLAS
#define GGML_CUDA_NAME "ROCm"
#define GGML_CUBLAS_NAME "hipBLAS"
#else
#define GGML_CUDA_NAME "CUDA"
#define GGML_CUBLAS_NAME "cuBLAS"
#endif

#ifdef  __cplusplus
extern "C" {
#endif

#define GGML_CUDA_MAX_DEVICES       16

// 初始化 cublas，总是成功。要检查 CUDA 是否真正加载，请使用 `ggml_cublas_loaded`。
GGML_API GGML_CALL void   ggml_init_cublas(void);

// 如果有可用的 CUDA 设备并且 cublas 成功加载，则返回 `true`；否则返回 `false`。
GGML_API GGML_CALL bool   ggml_cublas_loaded(void);

GGML_API GGML_CALL void * ggml_cuda_host_malloc(size_t size);
GGML_API GGML_CALL void   ggml_cuda_host_free(void * ptr);

GGML_API GGML_CALL bool   ggml_cuda_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
GGML_API GGML_CALL bool   ggml_cuda_compute_forward(struct ggml_compute_params * params, struct ggml_tensor * tensor);

GGML_API GGML_CALL int    ggml_cuda_get_device_count(void);
GGML_API GGML_CALL void   ggml_cuda_get_device_description(int device, char * description, size_t description_size);

// backend API
GGML_API GGML_CALL ggml_backend_t ggml_backend_cuda_init(int device);

GGML_API GGML_CALL bool ggml_backend_is_cuda(ggml_backend_t backend);

GGML_API GGML_CALL ggml_backend_buffer_type_t ggml_backend_cuda_buffer_type(int device);
// 将张量缓冲区按行分割到多个设备上的缓冲区类型
GGML_API GGML_CALL ggml_backend_buffer_type_t ggml_backend_cuda_split_buffer_type(const float * tensor_split);
// 用于 CPU 后端的固定主机缓冲区，用于在 CPU 和 GPU 之间更快地复制
GGML_API GGML_CALL ggml_backend_buffer_type_t ggml_backend_cuda_host_buffer_type(void);

GGML_API GGML_CALL int  ggml_backend_cuda_get_device_count(void);
GGML_API GGML_CALL void ggml_backend_cuda_get_device_description(int device, char * description, size_t description_size);
# 定义一个名为 ggml_backend_cuda_get_device_memory 的函数，用于获取指定设备的内存信息
GGML_API GGML_CALL void ggml_backend_cuda_get_device_memory(int device, size_t * free, size_t * total);

# 如果是 C++ 环境，则结束 extern "C" 块
#ifdef  __cplusplus
}
#endif
```