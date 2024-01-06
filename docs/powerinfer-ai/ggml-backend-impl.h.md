# `PowerInfer\ggml-backend-impl.h`

```
#pragma once
// 防止头文件被多次包含

// ggml-backend internal header
// ggml-backend 内部头文件

#include "ggml-backend.h"
// 包含 ggml-backend.h 头文件

#ifdef  __cplusplus
extern "C" {
#endif
// 如果是 C++ 代码，则使用 extern "C" 包裹

    //
    // Backend buffer
    //
    // 后端缓冲区

    typedef void * ggml_backend_buffer_context_t;
    // 定义后端缓冲区上下文类型

    struct ggml_backend_buffer_i {
        void   (*free_buffer)   (ggml_backend_buffer_t buffer);
        // 释放缓冲区的函数指针
        void * (*get_base)      (ggml_backend_buffer_t buffer); // get base pointer
        // 获取缓冲区基地址的函数指针
        size_t (*get_alloc_size)(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // pre-allocation callback
        // 预分配回调函数指针
// 定义一个函数指针 init_tensor，用于在张量分配后执行的回调函数
void   (*init_tensor)   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); 

// 定义一个函数指针 free_tensor，用于在释放张量前执行的回调函数
void   (*free_tensor)   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); 

// 定义一个结构体 ggml_backend_buffer_i，包含了初始化张量和释放张量的函数指针
struct ggml_backend_buffer_i {
    void   (*init_tensor)   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // post-allocation callback
    void   (*free_tensor)   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // pre-free callback
};

// 定义一个结构体 ggml_backend_buffer，包含了接口、后端、上下文和大小
struct ggml_backend_buffer {
    struct ggml_backend_buffer_i iface; // 接口
    ggml_backend_t                backend; // 后端
    ggml_backend_buffer_context_t context; // 上下文
    size_t size; // 大小
};

// 定义一个函数 ggml_backend_buffer_init，用于初始化后端缓冲区
GGML_API ggml_backend_buffer_t ggml_backend_buffer_init(
        struct ggml_backend                  * backend,
        struct ggml_backend_buffer_i           iface,
               ggml_backend_buffer_context_t   context,
               size_t                          size);
    // 后端
    //

    // 定义后端上下文类型
    typedef void * ggml_backend_context_t;

    // 后端接口
    struct ggml_backend_i {
        // 获取后端名称
        const char * (*get_name)(ggml_backend_t backend);

        // 释放后端资源
        void (*free)(ggml_backend_t backend);

        // 分配缓冲区
        ggml_backend_buffer_t (*alloc_buffer)(ggml_backend_t backend, size_t size);

        // 获取缓冲区对齐方式
        size_t (*get_alignment)(ggml_backend_t backend);

        // 张量数据访问
        // 这些函数可以是异步的，提供了同步访问的辅助函数，会自动调用同步
        void (*set_tensor_async)(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
        void (*get_tensor_async)(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);
    }
// 定义函数指针 synchronize，用于同步后端
void (*synchronize) (ggml_backend_t backend);

// (可选) 在不同后端之间复制张量，允许单次复制传输
void (*cpy_tensor_from)(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst);
void (*cpy_tensor_to) (ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst);

// 使用计划计算图
ggml_backend_graph_plan_t (*graph_plan_create) (ggml_backend_t backend, struct ggml_cgraph * cgraph);
void (*graph_plan_free) (ggml_backend_t backend, ggml_backend_graph_plan_t plan);
void (*graph_plan_compute)(ggml_backend_t backend, ggml_backend_graph_plan_t plan);

// 不使用计划计算图
void (*graph_compute)(ggml_backend_t backend, struct ggml_cgraph * cgraph);

// 检查后端是否支持某个操作
bool (*supports_op)(ggml_backend_t backend, const struct ggml_tensor * op);
};

// 定义后端结构体，包含接口
struct ggml_backend {
    struct ggml_backend_i iface;
// 定义一个结构体 ggml_backend_context_t，用于存储后端上下文信息
ggml_backend_context_t context;
// 结束 C++ 语言的声明
#ifdef  __cplusplus
}
#endif
```