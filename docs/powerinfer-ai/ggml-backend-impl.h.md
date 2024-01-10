# `PowerInfer\ggml-backend-impl.h`

```
#pragma once

// ggml-backend internal header

#include "ggml-backend.h"

#ifdef  __cplusplus
extern "C" {
#endif

    //
    // Backend buffer
    //

    // 定义指向后端缓冲区上下文的指针类型
    typedef void * ggml_backend_buffer_context_t;

    // 定义后端缓冲区接口
    struct ggml_backend_buffer_i {
        void   (*free_buffer)   (ggml_backend_buffer_t buffer); // 释放缓冲区
        void * (*get_base)      (ggml_backend_buffer_t buffer); // 获取基地址指针
        size_t (*get_alloc_size)(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // 预分配回调
        void   (*init_tensor)   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // 分配后回调
        void   (*free_tensor)   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // 释放前回调
    };

    // 定义后端缓冲区结构
    struct ggml_backend_buffer {
        struct ggml_backend_buffer_i iface;

        ggml_backend_t                backend;
        ggml_backend_buffer_context_t context;

        size_t size;
    };

    // 初始化后端缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_buffer_init(
            struct ggml_backend                  * backend,
            struct ggml_backend_buffer_i           iface,
                   ggml_backend_buffer_context_t   context,
                   size_t                          size);

    //
    // Backend
    //

    // 定义指向后端上下文的指针类型
    typedef void * ggml_backend_context_t;
    # 定义了一个接口结构体 ggml_backend_i，包含了一系列函数指针，用于实现后端的各种操作
    struct ggml_backend_i {
        # 获取后端名称的函数指针
        const char * (*get_name)(ggml_backend_t backend);

        # 释放后端资源的函数指针
        void (*free)(ggml_backend_t backend);

        # 分配缓冲区的函数指针
        ggml_backend_buffer_t (*alloc_buffer)(ggml_backend_t backend, size_t size);

        # 获取缓冲区对齐方式的函数指针
        size_t (*get_alignment)(ggml_backend_t backend);

        # 张量数据访问的函数指针，这些函数可以是异步的，也提供了同步访问的辅助函数，会自动调用同步函数
        void (*set_tensor_async)(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
        void (*get_tensor_async)(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);
        void (*synchronize)     (ggml_backend_t backend);

        # (可选) 在不同后端之间复制张量，允许单次复制传输
        void (*cpy_tensor_from)(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst);
        void (*cpy_tensor_to)  (ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst);

        # 使用计划执行计算图的函数指针
        ggml_backend_graph_plan_t (*graph_plan_create) (ggml_backend_t backend, struct ggml_cgraph * cgraph);
        void                      (*graph_plan_free)   (ggml_backend_t backend, ggml_backend_graph_plan_t plan);
        void                      (*graph_plan_compute)(ggml_backend_t backend, ggml_backend_graph_plan_t plan);

        # 不使用计划执行计算图的函数指针
        void (*graph_compute)(ggml_backend_t backend, struct ggml_cgraph * cgraph);

        # 检查后端是否支持某个操作的函数指针
        bool (*supports_op)(ggml_backend_t backend, const struct ggml_tensor * op);
    };

    # 定义了一个后端结构体 ggml_backend，包含了接口结构体和后端上下文
    struct ggml_backend {
        struct ggml_backend_i iface;  # 接口结构体
        ggml_backend_context_t context;  # 后端上下文
    };
#ifdef  __cplusplus
}  // 如果是 C++ 环境，则结束 extern "C" 块
#endif
```