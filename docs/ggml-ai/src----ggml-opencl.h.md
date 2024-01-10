# `ggml\src\ggml-opencl.h`

```
#pragma once
// 防止头文件被重复包含

#include "ggml.h"
// 包含 ggml.h 头文件

#ifdef  __cplusplus
extern "C" {
#endif
// 如果是 C++ 环境，则按照 C 语言的方式进行编译

GGML_API void ggml_cl_init(void);
// 初始化 OpenCL 环境

GGML_API void   ggml_cl_mul(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 使用 OpenCL 对两个张量进行乘法运算，并将结果存储到目标张量中

GGML_API bool   ggml_cl_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 检查是否可以使用 OpenCL 对两个张量进行矩阵乘法运算

GGML_API size_t ggml_cl_mul_mat_get_wsize(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 获取进行矩阵乘法运算所需的临时缓冲区大小

GGML_API void   ggml_cl_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst, void * wdata, size_t wsize);
// 使用 OpenCL 对两个张量进行矩阵乘法运算，并将结果存储到目标张量中，同时需要提供临时缓冲区

GGML_API void * ggml_cl_host_malloc(size_t size);
// 在主机端分配内存

GGML_API void   ggml_cl_host_free(void * ptr);
// 释放主机端内存

GGML_API void ggml_cl_free_data(const struct ggml_tensor* tensor);
// 释放张量数据所占用的内存

GGML_API void ggml_cl_transform_tensor(void * data, struct ggml_tensor * tensor);
// 将数据转换为张量的形式

#ifdef  __cplusplus
}
#endif
// 如果是 C++ 环境，则结束按照 C 语言的方式进行编译
```