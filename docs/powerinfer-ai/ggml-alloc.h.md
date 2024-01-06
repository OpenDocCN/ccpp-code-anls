# `PowerInfer\ggml-alloc.h`

```
#pragma once
// 防止头文件被多次包含

#include "ggml.h"
// 包含 ggml.h 头文件

#ifdef  __cplusplus
extern "C" {
#endif
// 如果是 C++ 环境，则使用 extern "C" 包裹

struct ggml_backend;
struct ggml_backend_buffer;
// 声明 ggml_backend 和 ggml_backend_buffer 结构体

//
// Legacy API
//

typedef struct ggml_allocr * ggml_allocr_t;
// 定义 ggml_allocr_t 类型为指向 ggml_allocr 结构体的指针

// initialize allocator for use with CPU backend only
GGML_API ggml_allocr_t ggml_allocr_new(void * data, size_t size, size_t alignment);
// 初始化用于 CPU 后端的分配器，传入数据指针、大小和对齐方式
GGML_API ggml_allocr_t ggml_allocr_new_measure(size_t alignment);
// 初始化用于测量的分配器，传入对齐方式
// 从给定的缓冲区创建一个分配器，用于 ggml-backend
GGML_API ggml_allocr_t ggml_allocr_new_from_buffer(struct ggml_backend_buffer * buffer);

// 从给定的后端创建一个分配器，分配一个拥有的缓冲区
GGML_API ggml_allocr_t ggml_allocr_new_from_backend(struct ggml_backend * backend, size_t size);

// 从给定的后端创建一个测量分配器
GGML_API ggml_allocr_t ggml_allocr_new_measure_from_backend(struct ggml_backend * backend);

// 获取分配器的缓冲区
GGML_API struct ggml_backend_buffer * ggml_allocr_get_buffer(ggml_allocr_t alloc);

// 告诉分配器按照列表中描述的顺序解析节点
// 如果您的图被优化为无序执行，应该调用这个函数
GGML_API void ggml_allocr_set_parse_seq(ggml_allocr_t alloc, const int * list, int n);

// 释放分配器
GGML_API void ggml_allocr_free(ggml_allocr_t alloc);

// 检查分配器是否是测量分配器
GGML_API bool ggml_allocr_is_measure(ggml_allocr_t alloc);

// 重置分配器
GGML_API void ggml_allocr_reset(ggml_allocr_t alloc);

// 分配张量给分配器
GGML_API void ggml_allocr_alloc(ggml_allocr_t alloc, struct ggml_tensor * tensor);

// 获取分配器的最大尺寸
GGML_API size_t ggml_allocr_max_size(ggml_allocr_t alloc);

// 分配图给分配器
GGML_API size_t ggml_allocr_alloc_graph(ggml_allocr_t alloc, struct ggml_cgraph * graph);
// ggml-backend v2 API
// 说明这部分代码是关于 ggml-backend v2 的 API

// Seperate tensor and graph allocator objects
// This is necessary for multi-backend allocation because the graph allocator needs to use multiple tensor allocators
// The original API is kept as a wrapper around the new API
// 分离张量和图形分配器对象
// 这对于多后端分配是必要的，因为图形分配器需要使用多个张量分配器
// 原始 API 作为新 API 的包装器被保留

// Tensor allocator
// 张量分配器

// 定义了 ggml_tallocr_t 结构体，表示张量分配器对象

GGML_API ggml_tallocr_t ggml_tallocr_new(void * data, size_t size, size_t alignment);
// 创建一个新的张量分配器对象，使用给定的数据、大小和对齐方式

GGML_API ggml_tallocr_t ggml_tallocr_new_measure(size_t alignment);
// 创建一个新的张量分配器对象，使用给定的对齐方式

GGML_API ggml_tallocr_t ggml_tallocr_new_from_buffer(struct ggml_backend_buffer * buffer);
// 从给定的后端缓冲区创建一个新的张量分配器对象

GGML_API ggml_tallocr_t ggml_tallocr_new_from_backend(struct ggml_backend * backend, size_t size); // allocates an owned buffer
// 从给定的后端和大小创建一个新的张量分配器对象，分配一个拥有的缓冲区

GGML_API ggml_tallocr_t ggml_tallocr_new_measure_from_backend(struct ggml_backend * backend);
// 从给定的后端创建一个新的张量分配器对象，使用测量的方式

GGML_API struct ggml_backend_buffer * ggml_tallocr_get_buffer(ggml_tallocr_t talloc);
// 获取张量分配器对象的缓冲区

GGML_API void   ggml_tallocr_free       (ggml_tallocr_t talloc);
// 释放张量分配器对象
// 检查给定的图形分配器是否是度量器
GGML_API bool   ggml_tallocr_is_measure (ggml_tallocr_t talloc);

// 重置给定的图形分配器
GGML_API void   ggml_tallocr_reset      (ggml_tallocr_t talloc);

// 从给定的图形分配器中分配给定张量的内存
GGML_API void   ggml_tallocr_alloc      (ggml_tallocr_t talloc, struct ggml_tensor * tensor);

// 获取给定的图形分配器的最大大小
GGML_API size_t ggml_tallocr_max_size   (ggml_tallocr_t talloc);


// 图形分配器
typedef struct ggml_gallocr * ggml_gallocr_t;

// 创建一个新的图形分配器
GGML_API ggml_gallocr_t ggml_gallocr_new(void);

// 释放给定的图形分配器
GGML_API void   ggml_gallocr_free(ggml_gallocr_t galloc);

// 设置解析序列的图形分配器
GGML_API void   ggml_gallocr_set_parse_seq(ggml_gallocr_t galloc, const int * list, int n);

// 从给定的图形分配器和张量分配器中分配图形
GGML_API size_t ggml_gallocr_alloc_graph(ggml_gallocr_t galloc, ggml_tallocr_t talloc, struct ggml_cgraph * graph);

// 从给定的图形分配器和哈希表中的张量分配器中分配图形
GGML_API void   ggml_gallocr_alloc_graph_n(
                    ggml_gallocr_t galloc,
                    struct ggml_cgraph * graph,
                    struct ggml_hash_set hash_set,
// 定义一个名为 ggml_tallocr_t 的结构体指针类型，命名为 hash_node_talloc
ggml_tallocr_t * hash_node_talloc;

// 如果是 C++ 环境，则结束 extern "C" 块
#ifdef  __cplusplus
}
#endif
```