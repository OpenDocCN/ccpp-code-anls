# `PowerInfer\ggml-backend.h`

```
#pragma once
// 防止头文件被重复包含

#include "ggml.h"
#include "ggml-alloc.h"
// 包含其他头文件

#ifdef  __cplusplus
extern "C" {
#endif
// 如果是 C++ 编译器，则采用 C 的方式进行编译

    //
    // Backend buffer
    //

    struct ggml_backend_buffer;
    typedef struct ggml_backend_buffer * ggml_backend_buffer_t;
    // 定义了一个结构体和一个指向该结构体的指针类型

    // backend buffer functions
    GGML_API void   ggml_backend_buffer_free          (ggml_backend_buffer_t buffer);
    // 释放后端缓冲区
    GGML_API size_t ggml_backend_buffer_get_alignment (ggml_backend_buffer_t buffer);
    // 获取后端缓冲区的对齐方式
    GGML_API void * ggml_backend_buffer_get_base      (ggml_backend_buffer_t buffer);
    // 获取后端缓冲区的基地址
// 获取缓冲区的大小
GGML_API size_t ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer);

// 获取缓冲区分配的大小
GGML_API size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);

// 初始化缓冲区的张量
GGML_API void ggml_backend_buffer_init_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);

// 释放缓冲区的张量
GGML_API void ggml_backend_buffer_free_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor);

//
// 后端
//

// 后端结构体
struct ggml_backend;
typedef struct ggml_backend * ggml_backend_t;
typedef void * ggml_backend_graph_plan_t;

// 获取张量的后端
GGML_API ggml_backend_t ggml_get_backend(const struct ggml_tensor * tensor);

// 获取后端的名称
GGML_API const char * ggml_backend_name(ggml_backend_t backend);

// 释放后端
GGML_API void ggml_backend_free(ggml_backend_t backend);

// 分配后端的缓冲区
GGML_API ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size);
# 获取后端对齐方式的大小
GGML_API size_t ggml_backend_get_alignment(ggml_backend_t backend);

# 异步设置张量数据
GGML_API void ggml_backend_tensor_set_async(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);

# 异步获取张量数据
GGML_API void ggml_backend_tensor_get_async(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);

# 设置张量数据
GGML_API void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size);

# 获取张量数据
GGML_API void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size);

# 同步后端
GGML_API void ggml_backend_synchronize(ggml_backend_t backend);

# 创建图计划
GGML_API ggml_backend_graph_plan_t ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph);

# 释放图计划
GGML_API void ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan);

# 计算图计划
GGML_API void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan);

# 计算图
GGML_API void ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph);

# 后端是否支持操作
GGML_API bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op);

# 在不同后端之间复制张量
GGML_API void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst);
    // CPU后端

    // 初始化CPU后端
    GGML_API ggml_backend_t ggml_backend_cpu_init(void);

    // 检查给定的后端是否为CPU后端
    GGML_API bool ggml_backend_is_cpu(ggml_backend_t backend);

    // 设置CPU后端的线程数
    GGML_API void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads);

    // 从现有指针创建后端缓冲区
    GGML_API ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(ggml_backend_t backend_cpu, void * ptr, size_t size);


    // 后端调度器

    // 后端调度器允许多个后端一起使用
    // 处理计算缓冲区分配，张量分配给后端，以及在后端之间复制张量
    // 后端的选择基于：
    // - 支持操作的后端
    // - 预先分配张量的位置（例如权重）

    /*
      示例用法：

        sched = ggml_backend_sched_new({backend_gpu, backend_gpu2, backend_cpu}, num_backends);
        // sched 使用测量分配器进行初始化，直到使用测量图分配为止

        // 从测量图中初始化缓冲区
        measure_graph = build_graph(sched); // 使用 allocr 根据需要分配输入

        // 在 build_graph 中：
        build_graph(...) {
            // 在特定后端分配张量（可选，建议：在不同的缓冲区中预先分配输入）
            alloc_cpu = ggml_backend_sched_get_allocr(sched, backend_cpu);
            ggml_allocr_alloc(alloc_cpu, tensor);

            // 手动将节点分配给后端（可选，在大多数情况下不应该需要）
            struct ggml_tensor * node = ggml_mul_mat(ctx, ...);
            ggml_backend_sched_set_node_backend(sched, node, backend_gpu);
        }

        // 从测量图中分配后端缓冲区
        ggml_backend_sched_init_measure(sched, measure_graph);

        // 调度器现在已准备好计算图形

        // 计算
        graph = build_graph(sched);
        ggml_backend_sched_graph_compute(sched, graph);
    */

    // 定义一个后端调度器结构
    struct ggml_backend_sched;
    typedef struct ggml_backend_sched * ggml_backend_sched_t;

    // 初始化一个后端调度器
    GGML_API ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, int n_backends);

    // 释放后端调度器
    GGML_API void ggml_backend_sched_free(ggml_backend_sched_t sched);
// 从一个测量图中初始化后端缓冲区
GGML_API void ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph);

// 获取后端调度器的 tallocr
GGML_API ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend);

// 获取后端调度器的缓冲区
GGML_API ggml_backend_buffer_t ggml_backend_sched_get_buffer (ggml_backend_sched_t sched, ggml_backend_t backend);

// 设置节点的后端
GGML_API void ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend);

// 在后端调度器上分配一个图
GGML_API void ggml_backend_sched_graph_compute(
        ggml_backend_sched_t sched,
        struct ggml_cgraph * graph);

#ifdef  __cplusplus
}
#endif
```