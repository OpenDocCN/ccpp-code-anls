# `PowerInfer\ggml-alloc.h`

```
#pragma once
// 防止头文件被重复包含

#include "ggml.h"
// 包含 ggml.h 头文件

#ifdef  __cplusplus
extern "C" {
#endif
// 如果是 C++ 代码，则使用 extern "C" 包裹

struct ggml_backend;
// 声明 ggml_backend 结构体
struct ggml_backend_buffer;
// 声明 ggml_backend_buffer 结构体

//
// Legacy API
//

typedef struct ggml_allocr * ggml_allocr_t;
// 定义 ggml_allocr_t 为 ggml_allocr 结构体指针类型

// initialize allocator for use with CPU backend only
GGML_API ggml_allocr_t ggml_allocr_new(void * data, size_t size, size_t alignment);
// 为 CPU 后端初始化分配器
GGML_API ggml_allocr_t ggml_allocr_new_measure(size_t alignment);
// 为测量初始化分配器

// initialize allocator for use with ggml-backend
GGML_API ggml_allocr_t ggml_allocr_new_from_buffer(struct ggml_backend_buffer * buffer);
// 从 ggml_backend_buffer 初始化分配器
GGML_API ggml_allocr_t ggml_allocr_new_from_backend(struct ggml_backend * backend, size_t size); // allocates an owned buffer
// 从 ggml_backend 初始化分配器，并分配一个拥有的缓冲区
GGML_API ggml_allocr_t ggml_allocr_new_measure_from_backend(struct ggml_backend * backend);
// 从 ggml_backend 测量初始化分配器

GGML_API struct ggml_backend_buffer * ggml_allocr_get_buffer(ggml_allocr_t alloc);
// 获取分配器的缓冲区

// tell the allocator to parse nodes following the order described in the list
// you should call this if your graph are optimized to execute out-of-order
GGML_API void   ggml_allocr_set_parse_seq(ggml_allocr_t alloc, const int * list, int n);
// 告诉分配器按照列表中描述的顺序解析节点，如果图被优化以无序执行，则应调用此函数

GGML_API void   ggml_allocr_free       (ggml_allocr_t alloc);
// 释放分配器
GGML_API bool   ggml_allocr_is_measure (ggml_allocr_t alloc);
// 检查分配器是否为测量
GGML_API void   ggml_allocr_reset      (ggml_allocr_t alloc);
// 重置分配器
GGML_API void   ggml_allocr_alloc      (ggml_allocr_t alloc, struct ggml_tensor * tensor);
// 分配张量
GGML_API size_t ggml_allocr_max_size   (ggml_allocr_t alloc);
// 获取分配器的最大大小

GGML_API size_t ggml_allocr_alloc_graph(ggml_allocr_t alloc, struct ggml_cgraph * graph);
// 分配图

//
// ggml-backend v2 API
//

// Seperate tensor and graph allocator objects
// This is necessary for multi-backend allocation because the graph allocator needs to use multiple tensor allocators
// The original API is kept as a wrapper around the new API

// Tensor allocator
typedef struct ggml_tallocr * ggml_tallocr_t;
// 定义 ggml_tallocr_t 为 ggml_tallocr 结构体指针类型

GGML_API ggml_tallocr_t ggml_tallocr_new(void * data, size_t size, size_t alignment);
// 初始化张量分配器
GGML_API ggml_tallocr_t ggml_tallocr_new_measure(size_t alignment);
// 初始化测量张量分配器
// 从给定的缓冲区创建一个新的图形分配器
GGML_API ggml_tallocr_t ggml_tallocr_new_from_buffer(struct ggml_backend_buffer * buffer);

// 从给定的后端和大小创建一个新的图形分配器，分配一个拥有的缓冲区
GGML_API ggml_tallocr_t ggml_tallocr_new_from_backend(struct ggml_backend * backend, size_t size);

// 从给定的后端创建一个新的测量图形分配器
GGML_API ggml_tallocr_t ggml_tallocr_new_measure_from_backend(struct ggml_backend * backend);

// 获取图形分配器的缓冲区
GGML_API struct ggml_backend_buffer * ggml_tallocr_get_buffer(ggml_tallocr_t talloc);

// 释放图形分配器
GGML_API void   ggml_tallocr_free       (ggml_tallocr_t talloc);

// 检查图形分配器是否为测量图形分配器
GGML_API bool   ggml_tallocr_is_measure (ggml_tallocr_t talloc);

// 重置图形分配器
GGML_API void   ggml_tallocr_reset      (ggml_tallocr_t talloc);

// 从图形分配器分配张量
GGML_API void   ggml_tallocr_alloc      (ggml_tallocr_t talloc, struct ggml_tensor * tensor);

// 获取图形分配器的最大大小
GGML_API size_t ggml_tallocr_max_size   (ggml_tallocr_t talloc);

// 图形分配器
typedef struct ggml_gallocr * ggml_gallocr_t;

// 创建一个新的图形分配器
GGML_API ggml_gallocr_t ggml_gallocr_new(void);

// 释放图形分配器
GGML_API void   ggml_gallocr_free(ggml_gallocr_t galloc);

// 设置解析序列
GGML_API void   ggml_gallocr_set_parse_seq(ggml_gallocr_t galloc, const int * list, int n);

// 从图形分配器分配图形
GGML_API size_t ggml_gallocr_alloc_graph(ggml_gallocr_t galloc, ggml_tallocr_t talloc, struct ggml_cgraph * graph);

// 从哈希表中的分配器分配图形
GGML_API void   ggml_gallocr_alloc_graph_n(
                    ggml_gallocr_t galloc,
                    struct ggml_cgraph * graph,
                    struct ggml_hash_set hash_set,
                    ggml_tallocr_t * hash_node_talloc);

#ifdef  __cplusplus
}
#endif
```