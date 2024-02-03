# `whisper.cpp\ggml-metal.h`

```cpp
// An interface allowing to compute ggml_cgraph with Metal
//
// This is a fully functional interface that extends ggml with GPU support for Apple devices.
// A similar interface can be created for other GPU backends (e.g. Vulkan, CUDA, OpenCL, etc.)
//
// How it works?
//
// As long as your program can create and evaluate a ggml_cgraph on the CPU, you can use this
// interface to evaluate the same graph on the GPU. Instead of using ggml_graph_compute(), you
// use ggml_metal_graph_compute() (or ggml_vulkan_graph_compute(), etc.)
//
// You only need to make sure that all memory buffers that you used during the graph creation
// are mapped to the device memory with the ggml_metal_add_buffer() function. This mapping is
// used during the graph evaluation to determine the arguments of the compute kernels.
//
// Synchronization between device and host memory (for example for input and output tensors)
// is done with the ggml_metal_set_tensor() and ggml_metal_get_tensor() functions.
//

#pragma once

#include "ggml.h"
#include "ggml-backend.h"

#include <stddef.h>
#include <stdbool.h>

// max memory buffers that can be mapped to the device
#define GGML_METAL_MAX_BUFFERS 64
#define GGML_METAL_MAX_COMMAND_BUFFERS 32

struct ggml_tensor;
struct ggml_cgraph;

#ifdef __cplusplus
extern "C" {
#endif

//
// backend API
// user-code should use only these functions
//

// 设置 Metal 后端的日志回调函数
GGML_API void ggml_backend_metal_log_set_callback(ggml_log_callback log_callback, void * user_data);

// 初始化 Metal 后端
GGML_API ggml_backend_t ggml_backend_metal_init(void);

// 检查后端是否为 Metal
GGML_API bool ggml_backend_is_metal(ggml_backend_t backend);

// 从指针创建 Metal 缓冲区
GGML_API GGML_CALL ggml_backend_buffer_t ggml_backend_metal_buffer_from_ptr(void * data, size_t size, size_t max_size);

// 设置 Metal 后端的回调数量
GGML_API void ggml_backend_metal_set_n_cb(ggml_backend_t backend, int n_cb);

// 获取 Metal 缓冲区类型
GGML_API GGML_CALL ggml_backend_buffer_type_t ggml_backend_metal_buffer_type(void);

// 检查设备是否支持特定的 family
// 理想情况下，用户代码应该进行这些检查
// 检查 Metal 后端是否支持指定的 GPU 系列
GGML_API bool ggml_backend_metal_supports_family(ggml_backend_t backend, int family);

// 在下一次调用 `ggml_backend_graph_compute` 时捕获所有提交的命令缓冲区
GGML_API void ggml_backend_metal_capture_next_compute(ggml_backend_t backend);

#ifdef __cplusplus
}
#endif
```