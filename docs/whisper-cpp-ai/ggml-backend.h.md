# `whisper.cpp\ggml-backend.h`

```cpp
#pragma once

#include "ggml.h"
#include "ggml-alloc.h"

#ifdef  __cplusplus
extern "C" {
#endif

    // 定义指向 ggml_backend_buffer_type 结构体的指针类型
    typedef struct ggml_backend_buffer_type * ggml_backend_buffer_type_t;
    // 定义指向 ggml_backend_buffer 结构体的指针类型
    typedef struct ggml_backend_buffer * ggml_backend_buffer_t;
    // 定义指向 ggml_backend 结构体的指针类型
    typedef struct ggml_backend * ggml_backend_t;
    // 定义指向 ggml_backend_graph_plan 结构体的指针类型
    typedef void * ggml_backend_graph_plan_t;

    //
    // Backend buffer
    //

    // buffer type
    // 获取缓冲区类型的名称
    GGML_API           const char *          ggml_backend_buft_name            (ggml_backend_buffer_type_t buft);
    // 分配指定大小的缓冲区
    GGML_API GGML_CALL ggml_backend_buffer_t ggml_backend_buft_alloc_buffer    (ggml_backend_buffer_type_t buft, size_t size);
    // 获取缓冲区对齐方式
    GGML_API           size_t                ggml_backend_buft_get_alignment   (ggml_backend_buffer_type_t buft);
    // 获取缓冲区最大大小
    GGML_API           size_t                ggml_backend_buft_get_max_size    (ggml_backend_buffer_type_t buft);
    // 获取为给定张量分配缓冲区所需的大小
    GGML_API GGML_CALL size_t                ggml_backend_buft_get_alloc_size  (ggml_backend_buffer_type_t buft, struct ggml_tensor * tensor);
    // 检查缓冲区类型是否支持指定后端
    GGML_API           bool                  ggml_backend_buft_supports_backend(ggml_backend_buffer_type_t buft, ggml_backend_t backend);
    // 检查缓冲区类型是否在主机上
    GGML_API           bool                  ggml_backend_buft_is_host         (ggml_backend_buffer_type_t buft);

    // buffer
    // 缓冲区使用方式的枚举
    enum ggml_backend_buffer_usage {
        GGML_BACKEND_BUFFER_USAGE_ANY = 0,
        GGML_BACKEND_BUFFER_USAGE_WEIGHTS = 1,
    };

    // 获取缓冲区的名称
    GGML_API           const char *               ggml_backend_buffer_name          (ggml_backend_buffer_t buffer);
    // 释放缓冲区
    GGML_API           void                       ggml_backend_buffer_free          (ggml_backend_buffer_t buffer);
    // 获取缓冲区的基地址
    GGML_API           void *                     ggml_backend_buffer_get_base      (ggml_backend_buffer_t buffer);
    // 获取缓冲区的大小
    GGML_API           size_t                     ggml_backend_buffer_get_size      (ggml_backend_buffer_t buffer);
    // 初始化一个后端缓冲区，将其与张量关联起来
    GGML_API GGML_CALL void ggml_backend_buffer_init_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);
    
    // 获取缓冲区的对齐方式
    GGML_API size_t ggml_backend_buffer_get_alignment(ggml_backend_buffer_t buffer);
    
    // 获取缓冲区的最大大小
    GGML_API size_t ggml_backend_buffer_get_max_size(ggml_backend_buffer_t buffer);
    
    // 获取缓冲区为给定张量分配的大小
    GGML_API size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);
    
    // 清空缓冲区中的数据，填充为指定值
    GGML_API void ggml_backend_buffer_clear(ggml_backend_buffer_t buffer, uint8_t value);
    
    // 检查缓冲区是否在主机上
    GGML_API bool ggml_backend_buffer_is_host(ggml_backend_buffer_t buffer);
    
    // 设置缓冲区的用途
    GGML_API void ggml_backend_buffer_set_usage(ggml_backend_buffer_t buffer, enum ggml_backend_buffer_usage usage);
    
    // 获取缓冲区的类型
    GGML_API ggml_backend_buffer_type_t ggml_backend_buffer_get_type(ggml_backend_buffer_t buffer);
    
    // 重置缓冲区
    GGML_API void ggml_backend_buffer_reset(ggml_backend_buffer_t buffer);
    
    // 获取后端的名称
    GGML_API const char * ggml_backend_name(ggml_backend_t backend);
    
    // 释放后端资源
    GGML_API void ggml_backend_free(ggml_backend_t backend);
    
    // 获取后端的默认缓冲区类型
    GGML_API ggml_backend_buffer_type_t ggml_backend_get_default_buffer_type(ggml_backend_t backend);
    
    // 分配一个指定大小的后端缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size);
    
    // 获取后端的对齐方式
    GGML_API size_t ggml_backend_get_alignment(ggml_backend_t backend);
    
    // 获取后端的最大大小
    GGML_API size_t ggml_backend_get_max_size(ggml_backend_t backend);
    
    // 设置张量的异步数据
    GGML_API void ggml_backend_tensor_set_async(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
    // 异步从后端获取张量数据
    GGML_API void ggml_backend_tensor_get_async(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);
    
    // 设置张量数据
    GGML_API GGML_CALL void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);
    // 获取张量数据
    GGML_API GGML_CALL void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);
    
    // 同步后端
    GGML_API void ggml_backend_synchronize(ggml_backend_t backend);
    
    // 创建后端图计划
    GGML_API ggml_backend_graph_plan_t ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph);
    
    // 释放后端图计划
    GGML_API void ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan);
    // 计算后端图计划
    GGML_API void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan);
    // 计算后端图
    GGML_API bool ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph);
    // 后端是否支持操作
    GGML_API bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op);
    
    // 在不同后端之间复制张量
    GGML_API void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst);
    // 异步复制张量
    GGML_API void ggml_backend_tensor_copy_async(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst); // 自动回退到同步复制
    
    // CPU 后端
    GGML_API ggml_backend_t ggml_backend_cpu_init(void);
    
    // 判断后端是否为 CPU
    GGML_API GGML_CALL bool ggml_backend_is_cpu(ggml_backend_t backend);
    // 设置 CPU 后端线程数
    GGML_API void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads);
    
    // 从现有指针创建后端缓冲区
    GGML_API GGML_CALL ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(void * ptr, size_t size);
    
    // 获取 CPU 后端缓冲区类型
    GGML_API GGML_CALL ggml_backend_buffer_type_t ggml_backend_cpu_buffer_type(void);
#ifdef GGML_USE_CPU_HBM
    // 如果定义了 GGML_USE_CPU_HBM 宏，则声明一个函数用于获取 CPU HBM 缓冲区类型
    GGML_API ggml_backend_buffer_type_t ggml_backend_cpu_hbm_buffer_type(void);
#endif

    //
    // Backend registry
    //

    // 后端注册表是所有可用后端的注册表，允许以通用方式初始化后端

    // 获取注册的后端数量
    GGML_API size_t                     ggml_backend_reg_get_count(void);
    // 根据名称查找后端在注册表中的索引
    GGML_API size_t                     ggml_backend_reg_find_by_name(const char * name);
    // 根据字符串初始化后端，字符串格式为名称[:参数]
    GGML_API ggml_backend_t             ggml_backend_reg_init_backend_from_str(const char * backend_str);
    // 获取指定索引处后端的名称
    GGML_API const char *               ggml_backend_reg_get_name(size_t i);
    // 根据索引初始化后端，参数是后端特定的
    GGML_API ggml_backend_t             ggml_backend_reg_init_backend(size_t i, const char * params);
    // 获取指定索引处后端的默认缓冲区类型
    GGML_API ggml_backend_buffer_type_t ggml_backend_reg_get_default_buffer_type(size_t i);
    // 为指定索引处的后端分配缓冲区
    GGML_API ggml_backend_buffer_t      ggml_backend_reg_alloc_buffer(size_t i, size_t size);

    //
    // Backend scheduler
    //

    // 后端调度器允许多个后端一起使用
    // 处理计算缓冲区分配，将张量分配给后端，以及在后端之间复制张量
    // 后端的选择基于：
    // - 支持操作的后端
    // - 预先分配张量的位置（例如权重的位置）
    /*
      Example usage:

        sched = ggml_backend_sched_new({backend_gpu, backend_gpu2, backend_cpu}, num_backends);
        // sched is initialized with measure allocators and cannot be used until allocated with a measure graph

        // initialize buffers from a measure graph
        measure_graph = build_graph(sched); // use the allocr to allocate inputs as needed

        // in build_graph:
        build_graph(...) {
            // allocating tensors in a specific backend (optional, recommended: pre-allocate inputs in a different buffer)
            alloc_cpu = ggml_backend_sched_get_allocr(sched, backend_cpu);
            ggml_allocr_alloc(alloc_cpu, tensor);

            // manually assigning nodes to a backend (optional, shouldn't be needed in most cases)
            struct ggml_tensor * node = ggml_mul_mat(ctx, ...);
            ggml_backend_sched_set_node_backend(sched, node, backend_gpu);
        }

        // allocate backend buffers from measure graph
        ggml_backend_sched_init_measure(sched, measure_graph);

        // the scheduler is now ready to compute graphs

        // compute
        graph = build_graph(sched);
        ggml_backend_sched_graph_compute(sched, graph);
    */

    // Define a struct for the backend scheduler
    struct ggml_backend_sched;
    // Define a type for the backend scheduler struct pointer
    typedef struct ggml_backend_sched * ggml_backend_sched_t;

    // Define a callback function type for evaluating nodes in the scheduler
    // The callback function should return a boolean value based on whether the user wants to observe the node
    // The callback function takes a tensor, a boolean flag 'ask', and a user data pointer
    typedef bool (*ggml_backend_sched_eval_callback)(struct ggml_tensor * t, bool ask, void * user_data);

    // Initialize a backend scheduler
    // 创建一个新的后端调度器，传入后端数组、后端缓冲类型数组、后端数量和图大小
    GGML_API ggml_backend_sched_t  ggml_backend_sched_new(ggml_backend_t * backends, ggml_backend_buffer_type_t * bufts, int n_backends, size_t graph_size);
    // 释放后端调度器
    GGML_API void                  ggml_backend_sched_free(ggml_backend_sched_t sched);
    // 从度量图初始化后端缓冲区
    GGML_API void                  ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph);
    // 获取最后一个图的分割数
    GGML_API int                   ggml_backend_sched_get_n_splits(ggml_backend_sched_t sched);
    
    // 获取后端调度器的 tallocr
    GGML_API ggml_tallocr_t        ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend);
    // 获取后端调度器的缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_sched_get_buffer (ggml_backend_sched_t sched, ggml_backend_t backend);
    
    // 设置节点的后端
    GGML_API void                  ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend);
    // 获取节点的后端
    GGML_API ggml_backend_t        ggml_backend_sched_get_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node);
    
    // 在后端调度器上分配和计算图
    GGML_API void                  ggml_backend_sched_graph_compute(ggml_backend_sched_t sched, struct ggml_cgraph * graph);
    
    // 重置所有分配和分配器 - 必须在使用调度器分配输入之前调用
    GGML_API void                  ggml_backend_sched_reset(ggml_backend_sched_t sched);
    
    // 设置在图计算期间为每个结果节点调用的回调函数
    GGML_API void                  ggml_backend_sched_set_eval_callback(ggml_backend_sched_t sched, ggml_backend_sched_eval_callback callback, void * user_data);
    
    //
    // Utils
    //
    
    // 结构体，用于后端图的复制
    struct ggml_backend_graph_copy {
        ggml_backend_buffer_t buffer;
        struct ggml_context * ctx_allocated;
        struct ggml_context * ctx_unallocated;
        struct ggml_cgraph * graph;
    };
    // 复制图形到不同的后端
    GGML_API struct ggml_backend_graph_copy ggml_backend_graph_copy(ggml_backend_t backend, struct ggml_cgraph * graph);
    // 释放图形复制对象
    GGML_API void ggml_backend_graph_copy_free(struct ggml_backend_graph_copy copy);
    
    // 定义一个回调函数指针类型，用于评估后端的输出
    typedef bool (*GGML_CALL ggml_backend_eval_callback)(int node_index, struct ggml_tensor * t1, struct ggml_tensor * t2, void * user_data);
    
    // 比较两个后端的输出
    GGML_API bool ggml_backend_compare_graph_backend(ggml_backend_t backend1, ggml_backend_t backend2, struct ggml_cgraph * graph, ggml_backend_eval_callback callback, void * user_data);
    
    // 张量初始化
    GGML_API void ggml_backend_tensor_alloc(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor, void * addr);
    GGML_API void ggml_backend_view_init(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);
#ifdef  __cplusplus
}
#endif
``` 


#ifdef  __cplusplus
// 如果是 C++ 环境
}
#endif
```