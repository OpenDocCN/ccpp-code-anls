# `whisper.cpp\bindings\ruby\ext\ggml-backend.h`

```cpp
#pragma once

#include "ggml.h"
#include "ggml-alloc.h"

#ifdef  __cplusplus
extern "C" {
#endif

    //
    // Backend buffer
    //

    // 定义结构体 ggml_backend_buffer，并创建指向该结构体的指针类型 ggml_backend_buffer_t
    struct ggml_backend_buffer;
    typedef struct ggml_backend_buffer * ggml_backend_buffer_t;

    // 声明 backend buffer 相关函数
    GGML_API void   ggml_backend_buffer_free          (ggml_backend_buffer_t buffer);
    GGML_API size_t ggml_backend_buffer_get_alignment (ggml_backend_buffer_t buffer);
    GGML_API void * ggml_backend_buffer_get_base      (ggml_backend_buffer_t buffer);
    GGML_API size_t ggml_backend_buffer_get_size      (ggml_backend_buffer_t buffer);
    GGML_API size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);
    GGML_API void   ggml_backend_buffer_init_tensor   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);
    GGML_API void   ggml_backend_buffer_free_tensor   (ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);

    //
    // Backend
    //

    // 定义结构体 ggml_backend，并创建指向该结构体的指针类型 ggml_backend_t，以及指向图计划的指针类型 ggml_backend_graph_plan_t
    struct ggml_backend;
    typedef struct ggml_backend * ggml_backend_t;
    typedef void * ggml_backend_graph_plan_t;

    // 获取指定张量的后端
    GGML_API ggml_backend_t ggml_get_backend(const struct ggml_tensor * tensor);

    // 获取后端的名称
    GGML_API const char * ggml_backend_name(ggml_backend_t backend);
    // 释放后端资源
    GGML_API void         ggml_backend_free(ggml_backend_t backend);

    // 分配后端缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size);

    // 获取后端的对齐方式
    GGML_API size_t ggml_backend_get_alignment(ggml_backend_t backend);

    // 异步设置张量数据
    GGML_API void ggml_backend_tensor_set_async(      struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
    // 异步获取张量数据
    GGML_API void ggml_backend_tensor_get_async(const struct ggml_tensor * tensor,       void * data, size_t offset, size_t size);

    // 设置张量数据
    GGML_API void ggml_backend_tensor_set(      struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
    // 从给定的张量中获取数据到指定的内存地址，可以指定偏移量和大小
    GGML_API void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);
    
    // 同步后端，确保所有计算已经完成
    GGML_API void ggml_backend_synchronize(ggml_backend_t backend);
    
    // 创建后端图计划，用于执行计算图
    GGML_API ggml_backend_graph_plan_t ggml_backend_graph_plan_create (ggml_backend_t backend, struct ggml_cgraph * cgraph);
    
    // 释放后端图计划
    GGML_API void ggml_backend_graph_plan_free   (ggml_backend_t backend, ggml_backend_graph_plan_t plan);
    GGML_API void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan);
    GGML_API bool ggml_backend_graph_compute     (ggml_backend_t backend, struct ggml_cgraph * cgraph);
    GGML_API bool ggml_backend_supports_op       (ggml_backend_t backend, const struct ggml_tensor * op);
    
    // 在不同后端之间复制张量数据
    GGML_API void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst);
    
    //
    // CPU 后端
    //
    
    // 初始化 CPU 后端
    GGML_API ggml_backend_t ggml_backend_cpu_init(void);
    
    // 检查后端是否为 CPU 后端
    GGML_API bool ggml_backend_is_cpu(ggml_backend_t backend);
    GGML_API void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads);
    
    // 从现有指针创建后端缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(ggml_backend_t backend_cpu, void * ptr, size_t size);
    
    
    //
    // 后端调度器
    //
    
    // 后端调度器允许多个后端一起使用
    // 处理计算缓冲区的分配，将张量分配给后端，以及在后端之间复制张量
    // 后端的选择基于：
    // - 支持操作的后端
    // - 预先分配张量的位置（例如权重）
    // 定义一个结构体 ggml_backend_sched，用于管理后端调度器
    struct ggml_backend_sched;
    typedef struct ggml_backend_sched * ggml_backend_sched_t;

    // 初始化一个后端调度器，传入后端数组和后端数量
    GGML_API ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, int n_backends);

    // 释放后端调度器占用的内存
    GGML_API void ggml_backend_sched_free(ggml_backend_sched_t sched);

    // 从一个度量图中初始化后端缓冲区
    GGML_API void ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph);

    // 获取后端分配器，传入后端调度器和后端
    GGML_API ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend);
    
    // 获取后端缓冲区，传入后端调度器和后端
    GGML_API ggml_backend_buffer_t ggml_backend_sched_get_buffer(ggml_backend_sched_t sched, ggml_backend_t backend);
    // 设置节点的后端计算引擎
    GGML_API void ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend);

    // 在后端调度器上为图分配计算资源
    GGML_API void ggml_backend_sched_graph_compute(
            ggml_backend_sched_t sched,
            struct ggml_cgraph * graph);
#ifdef  __cplusplus
}
#endif
``` 


#ifdef  __cplusplus
// 如果是 C++ 环境
}
#endif
```