# `whisper.cpp\ggml-backend-impl.h`

```cpp
#pragma once

// ggml-backend internal header

#include "ggml-backend.h"

#ifdef  __cplusplus
extern "C" {
#endif

    //
    // Backend buffer
    //

    // buffer type
    typedef void * ggml_backend_buffer_type_context_t;

    // 定义了一个接口结构体 ggml_backend_buffer_type_i，包含了一系列函数指针
    struct ggml_backend_buffer_type_i {
        // 获取缓冲区类型的名称
        const char *          (*GGML_CALL get_name)        (ggml_backend_buffer_type_t buft);
        // 分配缓冲区
        ggml_backend_buffer_t (*GGML_CALL alloc_buffer)    (ggml_backend_buffer_type_t buft, size_t size);
        // 获取数据对齐方式（张量对齐）
        size_t                (*GGML_CALL get_alignment)   (ggml_backend_buffer_type_t buft);
        // 获取最大分配大小
        size_t                (*GGML_CALL get_max_size)    (ggml_backend_buffer_type_t buft);
        // 获取分配张量所需的数据大小，包括填充
        size_t                (*GGML_CALL get_alloc_size)  (ggml_backend_buffer_type_t buft, const struct ggml_tensor * tensor);
        // 检查缓冲区类型是否可被后端使用
        bool                  (*GGML_CALL supports_backend)(ggml_backend_buffer_type_t buft, ggml_backend_t backend);
        // 检查张量数据是否在主机内存中
        // 应该等同于 supports_backend(buft, ggml_backend_cpu_init())
        bool                  (*GGML_CALL is_host)         (ggml_backend_buffer_type_t buft);
    };

    // 定义了一个结构体 ggml_backend_buffer_type，包含了接口结构体和上下文
    struct ggml_backend_buffer_type {
        struct ggml_backend_buffer_type_i  iface;
        ggml_backend_buffer_type_context_t context;
    };

    // buffer
    typedef void * ggml_backend_buffer_context_t;
    // 定义一个接口结构体，包含了一系列函数指针，用于操作后端缓冲区
    struct ggml_backend_buffer_i {
        const char * (*GGML_CALL get_name)   (ggml_backend_buffer_t buffer); // 获取缓冲区的名称
        void         (*GGML_CALL free_buffer)(ggml_backend_buffer_t buffer); // 释放缓冲区
        void *       (*GGML_CALL get_base)   (ggml_backend_buffer_t buffer); // 获取缓冲区的基地址
        void         (*GGML_CALL init_tensor)(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor); // 初始化张量
        void         (*GGML_CALL set_tensor) (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size); // 设置张量数据
        void         (*GGML_CALL get_tensor) (ggml_backend_buffer_t buffer, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size); // 获取张量数据
        bool         (*GGML_CALL cpy_tensor) (ggml_backend_buffer_t buffer, const struct ggml_tensor * src, struct ggml_tensor * dst); // 复制张量
        void         (*GGML_CALL clear)      (ggml_backend_buffer_t buffer, uint8_t value); // 清空缓冲区
        void         (*GGML_CALL reset)      (ggml_backend_buffer_t buffer); // 重置缓冲区的内部状态，例如张量附加信息
    };

    // 定义一个后端缓冲区结构体，包含接口结构体、缓冲区类型、上下文、大小和使用方式等信息
    struct ggml_backend_buffer {
        struct ggml_backend_buffer_i  iface;
        ggml_backend_buffer_type_t    buft;
        ggml_backend_buffer_context_t context;
        size_t size;
        enum ggml_backend_buffer_usage usage;
    };

    // 初始化后端缓冲区
    GGML_CALL ggml_backend_buffer_t ggml_backend_buffer_init(
                   ggml_backend_buffer_type_t      buft,
            struct ggml_backend_buffer_i           iface,
                   ggml_backend_buffer_context_t   context,
                   size_t                          size);

    // 不要直接使用，使用 ggml_backend_tensor_copy 代替
    bool ggml_backend_buffer_copy_tensor(const struct ggml_tensor * src, struct ggml_tensor * dst);

    // 包含一组缓冲区的缓冲区
    # 声明一个函数，用于在多个缓冲区中分配一个缓冲区，并返回该缓冲区
    GGML_CALL ggml_backend_buffer_t ggml_backend_multi_buffer_alloc_buffer(ggml_backend_buffer_t * buffers, size_t n_buffers);
    # 声明一个函数，用于检查给定的缓冲区是否为多缓冲区
    GGML_CALL bool ggml_backend_buffer_is_multi_buffer(ggml_backend_buffer_t buffer);
    # 声明一个函数，用于设置多缓冲区的使用方式
    GGML_CALL void ggml_backend_multi_buffer_set_usage(ggml_backend_buffer_t buffer, enum ggml_backend_buffer_usage usage);
    
    //
    // Backend
    //
    
    # 定义一个类型别名，表示后端上下文
    typedef void * ggml_backend_context_t;
    // 定义了一个接口结构体 ggml_backend_i，包含了一系列函数指针，用于实现不同的功能
    struct ggml_backend_i {
        // 获取后端名称的函数指针
        const char * (*GGML_CALL get_name)(ggml_backend_t backend);

        // 释放后端资源的函数指针
        void (*GGML_CALL free)(ggml_backend_t backend);

        // 缓冲区分配相关函数指针
        ggml_backend_buffer_type_t (*GGML_CALL get_default_buffer_type)(ggml_backend_t backend);

        // （可选）异步张量数据访问相关函数指针
        void (*GGML_CALL set_tensor_async)(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
        void (*GGML_CALL get_tensor_async)(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);
        bool (*GGML_CALL cpy_tensor_async)(ggml_backend_t backend, const struct ggml_tensor * src, struct ggml_tensor * dst);

        // （可选）完成所有待处理操作的函数指针
        void (*GGML_CALL synchronize)(ggml_backend_t backend);

        // 使用计划执行计算图的函数指针
        ggml_backend_graph_plan_t (*GGML_CALL graph_plan_create) (ggml_backend_t backend, const struct ggml_cgraph * cgraph);
        void (*GGML_CALL graph_plan_free) (ggml_backend_t backend, ggml_backend_graph_plan_t plan);
        void (*GGML_CALL graph_plan_compute)(ggml_backend_t backend, ggml_backend_graph_plan_t plan);

        // 使用计划执行计算图的函数指针（异步）
        bool (*GGML_CALL graph_compute)(ggml_backend_t backend, struct ggml_cgraph * cgraph);

        // 检查后端是否支持某个操作的函数指针
        bool (*GGML_CALL supports_op)(ggml_backend_t backend, const struct ggml_tensor * op);
    };

    // 定义了一个后端结构体 ggml_backend，包含了接口结构体和上下文信息
    struct ggml_backend {
        struct ggml_backend_i iface;
        ggml_backend_context_t context;
    };

    // 后端注册相关函数声明
    typedef ggml_backend_t (*GGML_CALL ggml_backend_init_fn)(const char * params, void * user_data);
    GGML_CALL void ggml_backend_register(const char * name, ggml_backend_init_fn init_fn, ggml_backend_buffer_type_t default_buffer_type, void * user_data);
#ifdef  __cplusplus
}
#endif
``` 


#ifdef  __cplusplus
// 如果是 C++ 环境
}
#endif
```