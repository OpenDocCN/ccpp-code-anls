# `whisper.cpp\ggml-alloc.h`

```cpp
#pragma once

#include "ggml.h"

#ifdef  __cplusplus
extern "C" {
#endif

// 声明结构体 ggml_backend
struct ggml_backend;
// 声明结构体 ggml_backend_buffer
struct ggml_backend_buffer;
// 声明结构体 ggml_backend_buffer_type

//
// Legacy API
//

// 定义 ggml_allocr_t 为 ggml_allocr 结构体指针类型
typedef struct ggml_allocr * ggml_allocr_t;

// 为 CPU 后端初始化分配器
GGML_API ggml_allocr_t ggml_allocr_new(void * data, size_t size, size_t alignment);
GGML_API ggml_allocr_t ggml_allocr_new_measure(size_t alignment);

// 为 ggml-backend 初始化分配器
GGML_API ggml_allocr_t ggml_allocr_new_from_buffer(struct ggml_backend_buffer * buffer);
GGML_API ggml_allocr_t ggml_allocr_new_from_backend(struct ggml_backend * backend, size_t size); // 分配一个拥有的缓冲区
GGML_API ggml_allocr_t ggml_allocr_new_measure_from_backend(struct ggml_backend * backend);

GGML_API struct ggml_backend_buffer * ggml_allocr_get_buffer(ggml_allocr_t alloc);

// 设置解析节点的顺序
GGML_API void   ggml_allocr_set_parse_seq(ggml_allocr_t alloc, const int * list, int n);

GGML_API void   ggml_allocr_free       (ggml_allocr_t alloc);
GGML_API bool   ggml_allocr_is_measure (ggml_allocr_t alloc);
GGML_API void   ggml_allocr_reset      (ggml_allocr_t alloc);
GGML_API void   ggml_allocr_alloc      (ggml_allocr_t alloc, struct ggml_tensor * tensor);
GGML_API size_t ggml_allocr_max_size   (ggml_allocr_t alloc);

GGML_API size_t ggml_allocr_alloc_graph(ggml_allocr_t alloc, struct ggml_cgraph * graph);

//
// ggml-backend v2 API
//

// 分离张量和图形分配器对象
// 这对于多后端分配是必要的，因为图形分配器需要使用多个张量分配器
// 原始 API 作为新 API 的包装器保留

// 张量分配器
typedef struct ggml_tallocr * ggml_tallocr_t;

GGML_API ggml_tallocr_t ggml_tallocr_new(void * data, size_t size, size_t alignment);
// 创建一个新的测量分配器，指定对齐方式
GGML_API ggml_tallocr_t ggml_tallocr_new_measure(size_t alignment);

// 从给定的缓冲区类型和大小创建一个新的分配器
GGML_API ggml_tallocr_t ggml_tallocr_new_from_buft(struct ggml_backend_buffer_type * buft, size_t size);

// 从给定的后端和大小创建一个新的分配器，分配一个拥有的缓冲区
GGML_API ggml_tallocr_t ggml_tallocr_new_from_backend(struct ggml_backend * backend, size_t size);

// 从给定的缓冲区创建一个新的分配器
GGML_API ggml_tallocr_t ggml_tallocr_new_from_buffer(struct ggml_backend_buffer * buffer);

// 从给定的缓冲区类型创建一个新的测量分配器
GGML_API ggml_tallocr_t ggml_tallocr_new_measure_from_buft(struct ggml_backend_buffer_type * buft);

// 从给定的后端创建一个新的测量分配器
GGML_API ggml_tallocr_t ggml_tallocr_new_measure_from_backend(struct ggml_backend * backend);

// 获取分配器中的缓冲区
GGML_API struct ggml_backend_buffer * ggml_tallocr_get_buffer(ggml_tallocr_t talloc);

// 释放分配器
GGML_API void   ggml_tallocr_free       (ggml_tallocr_t talloc);

// 检查分配器是否为测量分配器
GGML_API bool   ggml_tallocr_is_measure (ggml_tallocr_t talloc);

// 重置分配器
GGML_API void   ggml_tallocr_reset      (ggml_tallocr_t talloc);

// 为张量分配内存
GGML_API void   ggml_tallocr_alloc      (ggml_tallocr_t talloc, struct ggml_tensor * tensor);

// 获取分配器的最大大小
GGML_API size_t ggml_tallocr_max_size   (ggml_tallocr_t talloc);

// 创建一个新的图分配器
GGML_API ggml_gallocr_t ggml_gallocr_new(void);

// 释放图分配器
GGML_API void   ggml_gallocr_free(ggml_gallocr_t galloc);

// 设置解析顺序
GGML_API void   ggml_gallocr_set_parse_seq(ggml_gallocr_t galloc, const int * list, int n);

// 从分配器中为图分配内存
GGML_API size_t ggml_gallocr_alloc_graph(ggml_gallocr_t galloc, ggml_tallocr_t talloc, struct ggml_cgraph * graph);

// 从哈希表中的分配器分配张量
GGML_API void   ggml_gallocr_alloc_graph_n(
                    ggml_gallocr_t galloc,
                    struct ggml_cgraph * graph,
                    struct ggml_hash_set hash_set,
                    ggml_tallocr_t * hash_node_talloc);

// 从给定的上下文和缓冲区类型中为所有张量创建缓冲区
GGML_API struct ggml_backend_buffer * ggml_backend_alloc_ctx_tensors_from_buft(struct ggml_context * ctx, struct ggml_backend_buffer_type * buft);
# 定义一个名为 ggml_backend_alloc_ctx_tensors 的函数，用于在给定上下文和后端的情况下分配缓冲区
GGML_API struct ggml_backend_buffer * ggml_backend_alloc_ctx_tensors(struct ggml_context * ctx, struct ggml_backend * backend);

# 如果是 C++ 环境，则结束 extern "C" 块
#ifdef  __cplusplus
}
#endif
```