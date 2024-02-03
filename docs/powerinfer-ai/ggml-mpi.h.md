# `PowerInfer\ggml-mpi.h`

```cpp
#pragma once

struct ggml_context;
struct ggml_tensor;
struct ggml_cgraph;

#ifdef __cplusplus
extern "C" {
#endif

struct ggml_mpi_context;

// 初始化 MPI 后端
void ggml_mpi_backend_init(void);

// 释放 MPI 后端资源
void ggml_mpi_backend_free(void);

// 初始化 MPI 上下文
struct ggml_mpi_context * ggml_mpi_init(void);

// 释放 MPI 上下文资源
void ggml_mpi_free(struct ggml_mpi_context * ctx);

// 获取当前进程在 MPI 中的排名
int ggml_mpi_rank(struct ggml_mpi_context * ctx);

// 初始化 MPI 图计算
void ggml_mpi_eval_init(
        struct ggml_mpi_context * ctx_mpi,
                            int * n_tokens,
                            int * n_past,
                            int * n_threads);

// MPI 图计算前处理
void ggml_mpi_graph_compute_pre(
        struct ggml_mpi_context * ctx_mpi,
             struct ggml_cgraph * gf,
                            int   n_layers);

// MPI 图计算后处理
void ggml_mpi_graph_compute_post(
        struct ggml_mpi_context * ctx_mpi,
             struct ggml_cgraph * gf,
                            int   n_layers);

#ifdef __cplusplus
}
#endif
```