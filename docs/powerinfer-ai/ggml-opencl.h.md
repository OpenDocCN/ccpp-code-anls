# `PowerInfer\ggml-opencl.h`

```
#pragma once
// 防止头文件被重复包含

#include "ggml.h"
// 包含自定义的头文件 ggml.h

#ifdef  __cplusplus
extern "C" {
#endif
// 如果是 C++ 环境，则使用 extern "C" 修饰

void ggml_cl_init(void);
// 初始化 OpenCL 环境

void   ggml_cl_mul(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 使用 OpenCL 对两个张量进行乘法运算

bool   ggml_cl_can_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 检查是否可以使用 OpenCL 对两个张量进行矩阵乘法运算

size_t ggml_cl_mul_mat_get_wsize(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst);
// 获取进行矩阵乘法运算所需的临时缓冲区大小

void   ggml_cl_mul_mat(const struct ggml_tensor * src0, const struct ggml_tensor * src1, struct ggml_tensor * dst, void * wdata, size_t wsize);
// 使用 OpenCL 对两个张量进行矩阵乘法运算

void * ggml_cl_host_malloc(size_t size);
// 在主机端分配内存

void   ggml_cl_host_free(void * ptr);
// 释放主机端内存

void ggml_cl_free_data(const struct ggml_tensor* tensor);
// 释放张量数据

# 定义一个函数ggml_cl_transform_tensor，该函数接受一个指向void类型的数据和一个指向ggml_tensor结构体的指针作为参数
void ggml_cl_transform_tensor(void * data, struct ggml_tensor * tensor);

# 如果是C++环境，则结束extern "C"语句块
#ifdef  __cplusplus
}
#endif
```