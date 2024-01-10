# `ggml\src\ggml-metal.h`

```
// Metal接口，允许使用Metal计算ggml_cgraph
//
// 这是一个完全功能的接口，为苹果设备的ggml提供了GPU支持。
// 可以为其他GPU后端（如Vulkan、CUDA、OpenCL等）创建类似的接口。
//
// 工作原理？
//
// 只要您的程序可以在CPU上创建和评估ggml_cgraph，就可以使用此接口在GPU上评估相同的图形。
// 不使用ggml_graph_compute()，而是使用ggml_metal_graph_compute()（或ggml_vulkan_graph_compute()等）。
//
// 您只需要确保在图形创建期间使用的所有内存缓冲区都通过ggml_metal_add_buffer()函数映射到设备内存。
// 在图形评估期间，此映射用于确定计算内核的参数。
//
// 设备和主机内存之间的同步（例如用于输入和输出张量）是使用ggml_metal_set_tensor()和ggml_metal_get_tensor()函数完成的。
//

#pragma once

#include "ggml.h"
#include "ggml-backend.h"

#include <stddef.h>
#include <stdbool.h>

// 可映射到设备的最大内存缓冲区
#define GGML_METAL_MAX_BUFFERS 64
#define GGML_METAL_MAX_COMMAND_BUFFERS 32

struct ggml_tensor;
struct ggml_cgraph;

#ifdef __cplusplus
extern "C" {
#endif

//
// 内部API
// 暂时暴露给用户代码
//

struct ggml_metal_context;

// 设置日志回调函数
void ggml_metal_log_set_callback(ggml_log_callback log_callback, void * user_data);

// 要使用的命令缓冲区数量
struct ggml_metal_context * ggml_metal_init(int n_cb);
void ggml_metal_free(struct ggml_metal_context * ctx);

// 主机内存分配
void * ggml_metal_host_malloc(size_t n);
void   ggml_metal_host_free  (void * data);

// 设置要使用的命令缓冲区数量
void ggml_metal_set_n_cb(struct ggml_metal_context * ctx, int n_cb);

// 创建主机内存缓冲区和设备内存缓冲区之间的映射
// - 确保在调用ggml_metal_graph_compute之前映射图中使用的所有缓冲区
// 在计算过程中使用映射来确定计算内核的参数
// 由于 Metal 永远不会访问主机内存缓冲区，因此不需要保留主机内存缓冲区
// max_size 指定张量的最大大小，并用于创建共享视图，以确保张量至少适合一个视图
bool ggml_metal_add_buffer(
        struct ggml_metal_context * ctx,
                       const char * name,
                             void * data,
                           size_t   size,
                           size_t   max_size);

// 将数据从主机内存设置到设备中
void ggml_metal_set_tensor(struct ggml_metal_context * ctx, struct ggml_tensor * t);

// 从设备中获取数据到主机内存
void ggml_metal_get_tensor(struct ggml_metal_context * ctx, struct ggml_tensor * t);

// 尝试查找可以在图中并发运行的操作
// 如果图的拓扑结构发生变化，应该再次运行它
void ggml_metal_graph_find_concurrency(struct ggml_metal_context * ctx, struct ggml_cgraph * gf, bool check_mem);

// 如果图已经被优化以进行并发调度，则返回 concur_list 的长度
int ggml_metal_if_optimized(struct ggml_metal_context * ctx);

// 输出用于 ggml_alloc 的 concur_list
int * ggml_metal_get_concur_list(struct ggml_metal_context * ctx);

// 与 ggml_graph_compute 相同，但使用 Metal
// 并行创建 gf->n_threads 个命令缓冲区
bool ggml_metal_graph_compute(struct ggml_metal_context * ctx, struct ggml_cgraph * gf);

//
// 后端 API
// 用户代码应仅使用这些函数
//

GGML_API ggml_backend_t ggml_backend_metal_init(void);

GGML_API bool ggml_backend_is_metal(ggml_backend_t backend);

GGML_API ggml_backend_buffer_t ggml_backend_metal_buffer_from_ptr(void * data, size_t size, size_t max_size);

GGML_API void ggml_backend_metal_set_n_cb(ggml_backend_t backend, int n_cb);
// 获取 Metal 后端缓冲区类型的函数声明
GGML_API ggml_backend_buffer_type_t ggml_backend_metal_buffer_type(void);

// 检查设备是否支持特定的设备族的辅助函数
// 理想情况下，用户代码应该进行这些检查
// 参考：https://developer.apple.com/metal/Metal-Feature-Set-Tables.pdf
GGML_API bool ggml_backend_metal_supports_family(ggml_backend_t backend, int family);

#ifdef __cplusplus
}
#endif
```