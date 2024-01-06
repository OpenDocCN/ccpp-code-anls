# `PowerInfer\ggml-metal.h`

```
// 一个接口，允许使用 Metal 计算 ggml_cgraph
//
// 这是一个完全功能的接口，扩展了 ggml 对苹果设备的 GPU 支持。
// 可以为其他 GPU 后端（如 Vulkan、CUDA、OpenCL 等）创建类似的接口。
//
// 它是如何工作的？
//
// 只要您的程序能够在 CPU 上创建和评估 ggml_cgraph，就可以使用此接口在 GPU 上评估相同的图形。
// 您可以使用 ggml_metal_graph_compute()（或 ggml_vulkan_graph_compute() 等）来代替使用 ggml_graph_compute()。
//
// 您只需要确保在图形创建过程中使用的所有内存缓冲区都通过 ggml_metal_add_buffer() 函数映射到设备内存。
// 在图形评估过程中，此映射用于确定计算内核的参数。
//
// 设备和主机内存之间的同步（例如用于输入和输出张量）是通过 ggml_metal_set_tensor() 和 ggml_metal_get_tensor() 函数完成的。
//
// 
#pragma once
// 包含所需的头文件
#include "ggml.h"
#include "ggml-backend.h"

// 包含必要的标准库头文件
#include <stddef.h>
#include <stdbool.h>

// 定义最大内存缓冲区和最大命令缓冲区的数量
#define GGML_METAL_MAX_BUFFERS 64
#define GGML_METAL_MAX_COMMAND_BUFFERS 32

// 定义结构体 ggml_tensor 和 ggml_cgraph
struct ggml_tensor;
struct ggml_cgraph;

#ifdef __cplusplus
extern "C" {
#endif

// 内部 API的声明
// 临时暴露给用户代码
//

// Metal 上下文结构体
struct ggml_metal_context;

// 设置日志回调函数
void ggml_metal_log_set_callback(ggml_log_callback log_callback, void * user_data);

// 初始化 Metal 上下文，指定要使用的命令缓冲区数量
struct ggml_metal_context * ggml_metal_init(int n_cb);

// 释放 Metal 上下文
void ggml_metal_free(struct ggml_metal_context * ctx);

// 主机内存分配
void * ggml_metal_host_malloc(size_t n);

// 释放主机内存
void   ggml_metal_host_free  (void * data);

// 设置要使用的命令缓冲区数量
void ggml_metal_set_n_cb(struct ggml_metal_context * ctx, int n_cb);

// 创建主机内存缓冲区和设备内存缓冲区之间的映射
// - 在调用 ggml_metal_graph_compute 之前，确保映射了图中使用的所有缓冲区
// - 映射在计算过程中用于确定计算内核的参数
// 在 Metal 中不需要保留主机内存缓冲区，因为它永远不会被 Metal 访问
// max_size 指定张量的最大大小，并用于创建共享视图，以确保张量至少适合一个视图
bool ggml_metal_add_buffer(
        struct ggml_metal_context * ctx,
                       const char * name,
                             void * data,
                           size_t   size,
                           size_t   max_size);

// 将数据从主机内存设置到设备中
void ggml_metal_set_tensor(struct ggml_metal_context * ctx, struct ggml_tensor * t);

// 将数据从设备中获取到主机内存中
void ggml_metal_get_tensor(struct ggml_metal_context * ctx, struct ggml_tensor * t);

// 尝试查找可以在图中并行运行的操作
// 如果图的拓扑结构发生变化，应该再次运行它
void ggml_metal_graph_find_concurrency(struct ggml_metal_context * ctx, struct ggml_cgraph * gf, bool check_mem);
// 如果图形已经针对并发调度进行了优化，则返回优化后的 concur_list 的长度
int ggml_metal_if_optimized(struct ggml_metal_context * ctx);

// 输出 ggml_alloc 的 concur_list
int * ggml_metal_get_concur_list(struct ggml_metal_context * ctx);

// 使用 Metal 执行 ggml_graph_compute，与 ggml_graph_compute 相同
// 在并行中创建 gf->n_threads 个命令缓冲区
void ggml_metal_graph_compute(struct ggml_metal_context * ctx, struct ggml_cgraph * gf);

//
// 后端 API
// 用户代码应仅使用这些函数
//

// 初始化 Metal 后端
GGML_API ggml_backend_t ggml_backend_metal_init(void);

// 检查后端是否为 Metal
GGML_API bool ggml_backend_is_metal(ggml_backend_t backend);
# 定义一个名为 ggml_backend_metal_set_n_cb 的函数，该函数接受一个 ggml_backend_t 类型的参数 backend 和一个整型参数 n_cb
GGML_API void ggml_backend_metal_set_n_cb(ggml_backend_t backend, int n_cb);

# 如果是 C++ 环境，则结束 extern "C" 块
#ifdef __cplusplus
}
#endif
```