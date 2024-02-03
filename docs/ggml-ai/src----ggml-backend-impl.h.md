# `ggml\src\ggml-backend-impl.h`

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

    // 定义了一个接口结构体，包含了一系列操作函数指针
    struct ggml_backend_buffer_type_i {
        ggml_backend_buffer_t (*alloc_buffer)    (ggml_backend_buffer_type_t buft, size_t size); // 分配缓冲区
        size_t                (*get_alignment)   (ggml_backend_buffer_type_t buft); // 获取张量对齐方式
        size_t                (*get_alloc_size)  (ggml_backend_buffer_type_t buft, struct ggml_tensor * tensor); // 获取分配张量所需的数据大小，包括填充
        bool                  (*supports_backend)(ggml_backend_buffer_type_t buft, ggml_backend_t backend); // 检查缓冲区类型是否可被后端使用
        // 检查张量数据是否在主机内存中
        // 应该等同于 supports_backend(buft, ggml_backend_cpu_init())
        bool                  (*is_host)         (ggml_backend_buffer_type_t buft);
    };

    // 定义了一个包含接口结构体和上下文的缓冲区类型结构体
    struct ggml_backend_buffer_type {
        struct ggml_backend_buffer_type_i  iface;
        ggml_backend_buffer_type_context_t context;
    };

    // buffer
    typedef void * ggml_backend_buffer_context_t;
    // 定义了一个接口结构体 ggml_backend_buffer_i，包含了一系列函数指针，用于操作缓冲区
    struct ggml_backend_buffer_i {
        void   (*free_buffer)    (ggml_backend_buffer_t buffer);  // 释放缓冲区
        //void     (*reset)      (ggml_backend_buffer_t buffer); // 重置缓冲区的内部状态，例如张量初始化时的张量额外信息
        void * (*get_base)       (ggml_backend_buffer_t buffer);  // 获取缓冲区的基地址
        void   (*init_tensor)    (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);  // 初始化张量
        void   (*set_tensor)     (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);  // 设置张量的数据
        void   (*get_tensor)     (ggml_backend_buffer_t buffer, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);  // 获取张量的数据
        // (optional) 在不同的缓冲区类型之间复制张量，允许单次复制传输
        void   (*cpy_tensor_from)(ggml_backend_buffer_t buffer, struct ggml_tensor * src, struct ggml_tensor * dst);  // 从源张量复制到目标张量
        void   (*cpy_tensor_to)  (ggml_backend_buffer_t buffer, struct ggml_tensor * src, struct ggml_tensor * dst);  // 从源张量复制到目标张量
        void   (*clear)          (ggml_backend_buffer_t buffer, uint8_t value);  // 清空缓冲区，填充指定值
    };

    // 定义了一个结构体 ggml_backend_buffer，包含了接口结构体、缓冲区类型、缓冲区上下文和大小
    struct ggml_backend_buffer {
        struct ggml_backend_buffer_i  iface;  // 接口结构体
        ggml_backend_buffer_type_t    buft;   // 缓冲区类型
        ggml_backend_buffer_context_t context;  // 缓冲区上下文
        size_t size;  // 缓冲区大小
    };

    // 初始化 ggml_backend_buffer_t 类型的缓冲区
    ggml_backend_buffer_t ggml_backend_buffer_init(
                   ggml_backend_buffer_type_t      buft,
            struct ggml_backend_buffer_i           iface,
                   ggml_backend_buffer_context_t   context,
                   size_t                          size);


    //
    // Backend
    //

    // 定义了一个指向 void 类型的指针 ggml_backend_context_t
    typedef void * ggml_backend_context_t;
    // 定义了一个接口结构体 ggml_backend_i，包含了一系列函数指针，用于实现后端的各种操作
    struct ggml_backend_i {
        // 获取后端名称的函数指针
        const char * (*get_name)(ggml_backend_t backend);

        // 释放后端资源的函数指针
        void (*free)(ggml_backend_t backend);

        // 获取默认缓冲区类型的函数指针
        ggml_backend_buffer_type_t (*get_default_buffer_type)(ggml_backend_t backend);

        // (可选) 异步张量数据访问的函数指针
        void (*set_tensor_async)(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
        void (*get_tensor_async)(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);

        // (可选) 异步张量复制的函数指针
        void (*cpy_tensor_from_async)(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst);
        void (*cpy_tensor_to_async)(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst);

        // 同步后端的函数指针
        void (*synchronize)(ggml_backend_t backend);

        // 使用计划执行计算图的函数指针
        ggml_backend_graph_plan_t (*graph_plan_create)(ggml_backend_t backend, struct ggml_cgraph * cgraph);
        void (*graph_plan_free)(ggml_backend_t backend, ggml_backend_graph_plan_t plan);
        void (*graph_plan_compute)(ggml_backend_t backend, ggml_backend_graph_plan_t plan);

        // 不使用计划执行计算图的函数指针
        bool (*graph_compute)(ggml_backend_t backend, struct ggml_cgraph * cgraph);

        // 检查后端是否支持某个操作的函数指针
        bool (*supports_op)(ggml_backend_t backend, const struct ggml_tensor * op);
    };

    // 定义了一个后端结构体 ggml_backend，包含了接口结构体和后端上下文
    struct ggml_backend {
        struct ggml_backend_i iface;
        ggml_backend_context_t context;
    };

    // 后端注册
    // 定义了一个函数指针类型 ggml_backend_init_fn，用于初始化后端
    typedef ggml_backend_t (*ggml_backend_init_fn)(const char * params, void * user_data);

    // 注册后端的函数，包括后端名称、初始化函数、默认缓冲区类型和用户数据
    void ggml_backend_register(const char * name, ggml_backend_init_fn init_fn, ggml_backend_buffer_type_t default_buffer_type, void * user_data);
#ifdef  __cplusplus
}  // 如果是 C++ 环境，则结束 extern "C" 块
#endif
```