# `PowerInfer\ggml-mpi.h`

```
#pragma once
// 防止头文件被多次包含

struct ggml_context;
// 声明 ggml_context 结构体

struct ggml_tensor;
// 声明 ggml_tensor 结构体

struct ggml_cgraph;
// 声明 ggml_cgraph 结构体

#ifdef __cplusplus
extern "C" {
#endif
// 如果是 C++ 环境，则使用 C 语言的方式进行编译

struct ggml_mpi_context;
// 声明 ggml_mpi_context 结构体

void ggml_mpi_backend_init(void);
// 初始化 MPI 后端

void ggml_mpi_backend_free(void);
// 释放 MPI 后端资源

struct ggml_mpi_context * ggml_mpi_init(void);
// 初始化 MPI 上下文

void ggml_mpi_free(struct ggml_mpi_context * ctx);
// 释放 MPI 上下文

int ggml_mpi_rank(struct ggml_mpi_context * ctx);
// 获取当前进程在 MPI 中的排名
// 初始化 MPI 执行环境，设置初始参数
void ggml_mpi_eval_init(
        struct ggml_mpi_context * ctx_mpi,  // MPI 上下文
                            int * n_tokens,  // 令牌数量
                            int * n_past,    // 过去数量
                            int * n_threads);  // 线程数量

// 在 MPI 上下文中计算图形之前的准备工作
void ggml_mpi_graph_compute_pre(
        struct ggml_mpi_context * ctx_mpi,  // MPI 上下文
             struct ggml_cgraph * gf,       // 图形
                            int   n_layers);  // 层数

// 在 MPI 上下文中计算图形之后的收尾工作
void ggml_mpi_graph_compute_post(
        struct ggml_mpi_context * ctx_mpi,  // MPI 上下文
             struct ggml_cgraph * gf,       // 图形
                            int   n_layers);  // 层数

#ifdef __cplusplus
}
#endif
```