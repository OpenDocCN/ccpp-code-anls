# `whisper.cpp\ggml-opencl.h`

```cpp
#pragma once

#include "ggml.h"
#include "ggml-backend.h"

#ifdef  __cplusplus
extern "C" {
#endif

// 初始化 OpenCL 环境
GGML_API void ggml_cl_init(void);

// 矩阵相乘操作
GGML_API void   ggml_cl_mul(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 矩阵相加操作
GGML_API void   ggml_cl_add(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 检查是否可以进行矩阵相乘操作
GGML_API bool   ggml_cl_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, const struct ggml_tensor * dst);
// 获取矩阵相乘结果的大小
GGML_API size_t ggml_cl_mul_mat_get_wsize(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 执行矩阵相乘操作
GGML_API void   ggml_cl_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst, void * wdata, size_t wsize);

// 释放 tensor 数据
GGML_API void ggml_cl_free_data(const struct ggml_tensor* tensor);

// 转换 tensor 数据
GGML_API void ggml_cl_transform_tensor(void * data, struct ggml_tensor * tensor);

// backend API

// 初始化 OpenCL 后端
// GGML_API ggml_backend_t ggml_backend_opencl_init(void);

// 检查是否为 OpenCL 后端
// GGML_API bool ggml_backend_is_opencl(ggml_backend_t backend);

// 获取 OpenCL 缓冲区类型
GGML_API ggml_backend_buffer_type_t ggml_backend_opencl_buffer_type(void);
// 获取 OpenCL 主机缓冲区类型
// GGML_API ggml_backend_buffer_type_t ggml_backend_opencl_host_buffer_type(void);

#ifdef  __cplusplus
}
#endif
```