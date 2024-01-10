# `PowerInfer\ggml-opencl.h`

```
#pragma once

#include "ggml.h"

#ifdef  __cplusplus
extern "C" {
#endif

// 初始化 OpenCL 环境
void ggml_cl_init(void);

// 矩阵相乘
void   ggml_cl_mul(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);

// 检查是否可以进行矩阵相乘
bool   ggml_cl_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);

// 获取矩阵相乘所需的临时缓冲区大小
size_t ggml_cl_mul_mat_get_wsize(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);

// 矩阵相乘
void   ggml_cl_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst, void * wdata, size_t wsize);

// 在主机上分配内存
void * ggml_cl_host_malloc(size_t size);

// 释放在主机上分配的内存
void   ggml_cl_host_free(void * ptr);

// 释放张量数据
void ggml_cl_free_data(const struct ggml_tensor* tensor);

// 转换张量数据
void ggml_cl_transform_tensor(void * data, struct ggml_tensor * tensor);

#ifdef  __cplusplus
}
#endif
```