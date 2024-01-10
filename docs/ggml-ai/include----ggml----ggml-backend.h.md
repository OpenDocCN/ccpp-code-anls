# `ggml\include\ggml\ggml-backend.h`

```
#pragma once

#include "ggml.h"
#include "ggml-alloc.h"

#ifdef  __cplusplus
extern "C" {
#endif

    typedef struct ggml_backend_buffer_type * ggml_backend_buffer_type_t;  // 定义指向 ggml_backend_buffer_type 结构体的指针类型
    typedef struct ggml_backend_buffer * ggml_backend_buffer_t;  // 定义指向 ggml_backend_buffer 结构体的指针类型
    typedef struct ggml_backend * ggml_backend_t;  // 定义指向 ggml_backend 结构体的指针类型
    typedef void * ggml_backend_graph_plan_t;  // 定义指向 void 类型的指针类型，用于表示图计划

    //
    // Backend buffer
    //

    // buffer type
    GGML_API ggml_backend_buffer_t ggml_backend_buft_alloc_buffer(ggml_backend_buffer_type_t buft, size_t size);  // 分配指定大小的后端缓冲区
    GGML_API size_t ggml_backend_buft_get_alignment (ggml_backend_buffer_type_t buft);  // 获取缓冲区类型的对齐方式
    GGML_API size_t ggml_backend_buft_get_alloc_size(ggml_backend_buffer_type_t buft, struct ggml_tensor * tensor);  // 获取分配给缓冲区的大小
    GGML_API bool ggml_backend_buft_supports_backend(ggml_backend_buffer_type_t buft, ggml_backend_t backend);  // 检查缓冲区类型是否支持指定的后端
    GGML_API bool ggml_backend_buft_is_host         (ggml_backend_buffer_type_t buft);  // 检查缓冲区类型是否为主机缓冲区

    // buffer
    GGML_API void   ggml_backend_buffer_free          (ggml_backend_buffer_t buffer);  // 释放后端缓冲区
    GGML_API void * ggml_backend_buffer_get_base      (ggml_backend_buffer_t buffer);  // 获取后端缓冲区的基地址
    GGML_API size_t ggml_backend_buffer_get_size      (ggml_backend_buffer_t buffer);  // 获取后端缓冲区的大小
    GGML_API void   ggml_backend_buffer_init_tensor   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);  // 初始化后端缓冲区的张量
    GGML_API size_t ggml_backend_buffer_get_alignment (ggml_backend_buffer_t buffer);  // 获取后端缓冲区的对齐方式
    GGML_API size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);  // 获取分配给后端缓冲区的大小
    GGML_API void   ggml_backend_buffer_clear         (ggml_backend_buffer_t buffer, uint8_t value);  // 清空后端缓冲区
    GGML_API bool   ggml_backend_buffer_is_host       (ggml_backend_buffer_t buffer);  // 检查后端缓冲区是否为主机缓冲区
    GGML_API ggml_backend_buffer_type_t ggml_backend_buffer_type(ggml_backend_buffer_t buffer);  // 获取后端缓冲区的类型

    //
    // Backend
    //

    GGML_API const char * ggml_backend_name(ggml_backend_t backend);  // 获取后端的名称
    GGML_API void         ggml_backend_free(ggml_backend_t backend);  // 释放后端
    # 获取默认的缓冲区类型
    GGML_API ggml_backend_buffer_type_t ggml_backend_get_default_buffer_type(ggml_backend_t backend);
    # 分配指定大小的缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size);
    # 获取对齐方式
    GGML_API size_t ggml_backend_get_alignment(ggml_backend_t backend);
    
    # 异步设置张量数据
    GGML_API void ggml_backend_tensor_set_async(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
    # 异步获取张量数据
    GGML_API void ggml_backend_tensor_get_async(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);
    
    # 设置张量数据
    GGML_API void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
    # 获取张量数据
    GGML_API void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);
    
    # 同步所有操作
    GGML_API void ggml_backend_synchronize(ggml_backend_t backend);
    
    # 创建图计划
    GGML_API ggml_backend_graph_plan_t ggml_backend_graph_plan_create (ggml_backend_t backend, struct ggml_cgraph * cgraph);
    # 释放图计划
    GGML_API void ggml_backend_graph_plan_free (ggml_backend_t backend, ggml_backend_graph_plan_t plan);
    # 计算图计划
    GGML_API void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan);
    # 计算图
    GGML_API bool ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph);
    # 检查后端是否支持操作
    GGML_API bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op);
    
    # 在不同后端之间复制张量
    GGML_API void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst);
    # 异步复制张量
    GGML_API void ggml_backend_tensor_copy_async(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst); // 自动回退到同步复制
    
    # CPU 后端初始化
    GGML_API ggml_backend_t ggml_backend_cpu_init(void);
    # 检查后端是否为 CPU
    GGML_API bool ggml_backend_is_cpu(ggml_backend_t backend);
    # 设置 CPU 后端的线程数
    GGML_API void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads);
    
    # 从现有指针创建后端缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(void * ptr, size_t size);
    
    # 获取 CPU 后端缓冲区的类型
    GGML_API ggml_backend_buffer_type_t ggml_backend_cpu_buffer_type(void);
#ifdef GGML_USE_CPU_HBM
    // 如果定义了 GGML_USE_CPU_HBM，则使用 CPU HBM 缓冲区类型
    GGML_API ggml_backend_buffer_type_t ggml_backend_cpu_hbm_buffer_type(void);
#endif

    //
    // Backend registry
    //

    // 后端注册表是所有可用后端的注册表，允许以通用方式初始化后端

    // 获取注册的后端数量
    GGML_API size_t                     ggml_backend_reg_get_count(void);
    // 通过名称查找注册的后端
    GGML_API size_t                     ggml_backend_reg_find_by_name(const char * name);
    // 从字符串初始化后端，字符串格式为名称[:参数]
    GGML_API ggml_backend_t             ggml_backend_reg_init_backend_from_str(const char * backend_str);
    // 获取注册的后端名称
    GGML_API const char *               ggml_backend_reg_get_name(size_t i);
    // 初始化指定索引的后端，参数是特定于后端的
    GGML_API ggml_backend_t             ggml_backend_reg_init_backend(size_t i, const char * params);
    // 获取指定索引的后端的默认缓冲区类型
    GGML_API ggml_backend_buffer_type_t ggml_backend_reg_get_default_buffer_type(size_t i);
    // 分配指定索引的后端的缓冲区
    GGML_API ggml_backend_buffer_t      ggml_backend_reg_alloc_buffer(size_t i, size_t size);

    //
    // Backend scheduler
    //

    // 后端调度器允许多个后端一起使用
    // 处理计算缓冲区分配，将张量分配给后端，以及在后端之间复制张量
    // 后端的选择基于：
    // - 支持操作的后端
    // - 预先分配张量的位置（例如权重）
    // 定义一个结构体 ggml_backend_sched
    struct ggml_backend_sched;
    // 定义一个指向 ggml_backend_sched 结构体的指针类型 ggml_backend_sched_t
    typedef struct ggml_backend_sched * ggml_backend_sched_t;

    // 初始化一个后端调度器，传入后端数组和后端数量
    GGML_API ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, int n_backends);

    // 释放后端调度器占用的内存
    GGML_API void ggml_backend_sched_free(ggml_backend_sched_t sched);

    // 从一个度量图中初始化后端缓冲区
    GGML_API void ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph);

    // 获取后端分配器
    GGML_API ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend);
    // 获取后端缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_sched_get_buffer (ggml_backend_sched_t sched, ggml_backend_t backend);
    // 设置节点的后端调度器
    GGML_API void ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend);
    
    // 在后端调度器上分配一个图
    GGML_API void ggml_backend_sched_graph_compute(
            ggml_backend_sched_t sched,
            struct ggml_cgraph * graph);
    
    
    //
    // Utils
    //
    
    // 用于复制图到不同的后端
    struct ggml_backend_graph_copy {
        ggml_backend_buffer_t buffer;
        struct ggml_context * ctx_allocated;
        struct ggml_context * ctx_unallocated;
        struct ggml_cgraph * graph;
    };
    
    // 复制图到不同的后端
    GGML_API struct ggml_backend_graph_copy ggml_backend_graph_copy(ggml_backend_t backend, struct ggml_cgraph * graph);
    GGML_API void                           ggml_backend_graph_copy_free(struct ggml_backend_graph_copy copy);
    
    // 定义一个回调函数指针类型，用于评估两个后端的输出
    typedef bool (*ggml_backend_eval_callback)(int node_index, struct ggml_tensor * t1, struct ggml_tensor * t2, void * user_data);
    
    // 比较两个后端的图的输出
    GGML_API void ggml_backend_compare_graph_backend(ggml_backend_t backend1, ggml_backend_t backend2, struct ggml_cgraph * graph, ggml_backend_eval_callback callback, void * user_data);
    
    // 张量初始化
    GGML_API void ggml_backend_tensor_alloc(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor, void * addr);
    GGML_API void ggml_backend_view_init(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);
#ifdef  __cplusplus
}  // 如果是 C++ 环境，则结束 extern "C" 块
#endif
```