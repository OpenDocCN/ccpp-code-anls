# `PowerInfer\ggml-mpi.c`

```
#include "ggml-mpi.h"  // 包含 MPI 相关的头文件

#include "ggml.h"  // 包含 ggml 库的头文件

#include <mpi.h>  // 包含 MPI 标准库的头文件

#include <stdio.h>  // 包含标准输入输出库的头文件
#include <stdlib.h>  // 包含标准库的头文件

#define MIN(a, b) ((a) < (b) ? (a) : (b))  // 定义一个取最小值的宏

#define UNUSED GGML_UNUSED  // 定义一个宏，用于标记未使用的变量

struct ggml_mpi_context {  // 定义一个结构体 ggml_mpi_context
    int rank;  // MPI 进程的排名
    int size;  // MPI 进程的数量
};

void ggml_mpi_backend_init(void) {  // MPI 后端初始化函数
    MPI_Init(NULL, NULL);  // 初始化 MPI
}

void ggml_mpi_backend_free(void) {  // MPI 后端释放函数
    MPI_Finalize();  // 终止 MPI
}

struct ggml_mpi_context * ggml_mpi_init(void) {  // MPI 初始化函数
    struct ggml_mpi_context * ctx = calloc(1, sizeof(struct ggml_mpi_context));  // 分配内存空间

    MPI_Comm_rank(MPI_COMM_WORLD, &ctx->rank);  // 获取当前进程的排名
    MPI_Comm_size(MPI_COMM_WORLD, &ctx->size);  // 获取当前进程的数量

    return ctx;  // 返回 MPI 上下文
}

void ggml_mpi_free(struct ggml_mpi_context * ctx) {  // 释放 MPI 上下文函数
    free(ctx);  // 释放内存空间
}

int ggml_mpi_rank(struct ggml_mpi_context * ctx) {  // 获取 MPI 进程排名函数
    return ctx->rank;  // 返回当前进程的排名
}

void ggml_mpi_eval_init(
        struct ggml_mpi_context * ctx_mpi,
                            int * n_tokens,
                            int * n_past,
                            int * n_threads) {  // MPI 评估初始化函数
    UNUSED(ctx_mpi);  // 标记未使用的 MPI 上下文

    // 同步工作节点参数与根节点
    MPI_Barrier(MPI_COMM_WORLD);  // 同步所有进程

    MPI_Bcast(n_tokens,  1, MPI_INT, 0, MPI_COMM_WORLD);  // 广播 n_tokens 参数
    MPI_Bcast(n_past,    1, MPI_INT, 0, MPI_COMM_WORLD);  // 广播 n_past 参数
    MPI_Bcast(n_threads, 1, MPI_INT, 0, MPI_COMM_WORLD);  // 广播 n_threads 参数
}

static int ggml_graph_get_node_idx(struct ggml_cgraph * gf, const char * name) {  // 获取图中节点索引的函数
    struct ggml_tensor * t = ggml_graph_get_tensor(gf, name);  // 获取图中指定名称的张量
    if (t == NULL) {  // 如果张量为空
        fprintf(stderr, "%s: tensor %s not found\n", __func__, name);  // 输出错误信息
        return -1;  // 返回错误码
    }

    for (int i = 0; i < gf->n_nodes; i++) {  // 遍历图中的节点
        if (gf->nodes[i] == t) {  // 如果节点等于张量
            return i;  // 返回节点索引
        }
    }

    fprintf(stderr, "%s: tensor %s not found in graph (should not happen)\n", __func__, name);  // 输出错误信息
    return -1;  // 返回错误码
}

static void ggml_mpi_tensor_send(struct ggml_tensor * t, int mpi_rank_dst) {  // MPI 发送张量的函数
    MPI_Datatype mpi_type;  // MPI 数据类型
    # 根据变量 t 的类型进行不同的处理
    switch (t->type) {
        # 如果 t 的类型是 GGML_TYPE_I32，则设置 mpi_type 为 MPI_INT32_T
        case GGML_TYPE_I32: mpi_type = MPI_INT32_T; break;
        # 如果 t 的类型是 GGML_TYPE_F32，则设置 mpi_type 为 MPI_FLOAT
        case GGML_TYPE_F32: mpi_type = MPI_FLOAT;   break;
        # 如果 t 的类型不是上述两种类型，则触发断言错误
        default: GGML_ASSERT(false && "not implemented");
    }

    # 调用 MPI_Send 函数发送数据
    const int retval = MPI_Send(t->data, ggml_nelements(t), mpi_type, mpi_rank_dst, 0, MPI_COMM_WORLD);
    # 断言发送操作是否成功
    GGML_ASSERT(retval == MPI_SUCCESS);
// 接收来自指定 MPI 进程的张量数据
static void ggml_mpi_tensor_recv(struct ggml_tensor * t, int mpi_rank_src) {
    MPI_Datatype mpi_type; // 定义 MPI 数据类型

    switch (t->type) { // 根据张量类型选择对应的 MPI 数据类型
        case GGML_TYPE_I32: mpi_type = MPI_INT32_T; break; // 整型
        case GGML_TYPE_F32: mpi_type = MPI_FLOAT;   break; // 浮点型
        default: GGML_ASSERT(false && "not implemented"); // 默认情况下报错
    }

    MPI_Status status; UNUSED(status); // 定义 MPI 状态变量

    const int retval = MPI_Recv(t->data, ggml_nelements(t), mpi_type, mpi_rank_src, MPI_ANY_TAG, MPI_COMM_WORLD, &status); // 接收 MPI 数据
    GGML_ASSERT(retval == MPI_SUCCESS); // 断言接收成功
}

// TODO: there are many improvements that can be done to this implementation
// 准备计算图的 MPI 实现，将计算图分片分配到不同的 MPI 节点上
void ggml_mpi_graph_compute_pre(
        struct ggml_mpi_context * ctx_mpi,
             struct ggml_cgraph * gf,
                            int   n_layers) {
    const int mpi_rank = ctx_mpi->rank; // 获取当前 MPI 进程的排名
    const int mpi_size = ctx_mpi->size; // 获取 MPI 进程的总数

    struct ggml_tensor * inp_tokens = ggml_graph_get_tensor(gf, "inp_tokens"); // 获取输入张量
    if (inp_tokens == NULL) { // 如果输入张量为空
        fprintf(stderr, "%s: tensor 'inp_tokens' not found\n", __func__); // 输出错误信息
        return; // 返回
    }

    struct ggml_tensor * inp0 = ggml_graph_get_tensor(gf, "layer_inp_0"); // 获取输入张量
    if (inp0 == NULL) { // 如果输入张量为空
        fprintf(stderr, "%s: tensor 'inp0' not found\n", __func__); // 输出错误信息
        return; // 返回
    }

    GGML_ASSERT(inp0 == gf->nodes[0]); // 断言输入张量等于计算图的第一个节点

    // 分配计算图到不同的 MPI 节点上进行计算
    //
    // 主节点 (0) 处理最后的层 + 计算图的剩余部分
    // 负责将输入 tokens 传递给第一个节点 (1)
    //
    // 节点 1:   [(  0) * n_per_node, (  1) * n_per_node)
    // 节点 2:   [(  1) * n_per_node, (  2) * n_per_node)
    // ...
    // 节点 n-1: [(n-2) * n_per_node, (n-1) * n_per_node)
    // 节点 0:   [(n-1) * n_per_node,            n_nodes)
    //
    # 如果当前节点的排名大于0
    if (mpi_rank > 0) {
        # 如果当前节点的排名为1
        if (mpi_rank == 1) {
            # 第一个节点（1）从主节点（0）接收输入标记
            ggml_mpi_tensor_recv(inp_tokens, 0);
        } else {
            # 接收每个节点的输入数据到“inp0”张量（即计算图中的第一个节点）
            ggml_mpi_tensor_recv(inp0, mpi_rank - 1);
        }
    } 
    # 如果当前节点的排名为0且节点数量大于1
    else if (mpi_size > 1) {
        # 节点0将输入标记发送给节点1
        ggml_mpi_tensor_send(inp_tokens, 1);

        # 从最后一个节点接收输出数据
        ggml_mpi_tensor_recv(inp0, mpi_size - 1);
    }
    {
        // 每个节点的层数
        const int n_per_node = (n_layers + (mpi_size - 1)) / mpi_size;
    
        // MPI 进程的索引
        const int mpi_idx = mpi_rank > 0 ? mpi_rank - 1 : mpi_size - 1;
    
        // 计算每个节点的起始层数和结束层数
        const int il0 =               (mpi_idx + 0) * n_per_node;
        const int il1 = MIN(n_layers, (mpi_idx + 1) * n_per_node);
    
        // 创建两个节点名称的缓冲区
        char name_l0[GGML_MAX_NAME];
        char name_l1[GGML_MAX_NAME];
    
        // 格式化节点名称
        snprintf(name_l0, sizeof(name_l0), "layer_inp_%d", il0);
        snprintf(name_l1, sizeof(name_l1), "layer_inp_%d", il1);
    
        // 获取节点索引
        const int idx_l0 =                ggml_graph_get_node_idx(gf, name_l0);
        const int idx_l1 = mpi_rank > 0 ? ggml_graph_get_node_idx(gf, name_l1) + 1 : gf->n_nodes;
    
        // 如果节点索引小于 0，则打印错误信息并返回
        if (idx_l0 < 0 || idx_l1 < 0) {
            fprintf(stderr, "%s: layer input nodes not found\n", __func__);
            return;
        }
    
        // 将输入数据附加到所有需要的节点
        // TODO: 不太好 - 应该能够在不修改计算图的情况下完成这一步（见下面的 TODO）
        for (int i = idx_l0; i < idx_l1; i++) {
            if (gf->nodes[i]->src[0] == gf->nodes[idx_l0]) {
                gf->nodes[i]->src[0] =  inp0;
            }
            if (gf->nodes[i]->src[1] == gf->nodes[idx_l0]) {
                gf->nodes[i]->src[1] =  inp0;
            }
        }
    
        // TODO: 不应该重新排列节点，应该能够执行计算图的子集
        for (int i = 1; i < idx_l1 - idx_l0; i++) {
            gf->nodes[i] = gf->nodes[idx_l0 + i];
            gf->grads[i] = gf->grads[idx_l0 + i];
        }
    
        // 第一个节点执行 "get_rows" 操作，其余节点从前一个节点获取数据
        if (mpi_idx != 0) {
            gf->nodes[0]->op = GGML_OP_NONE;
        }
    
        // 更新节点数量
        gf->n_nodes = idx_l1 - idx_l0;
    
        // 打印节点处理信息
        //fprintf(stderr, "%s: node %d: processing %d nodes [%d, %d)\n", __func__, mpi_rank, gf->n_nodes, il0, il1);
    }
// 结束 ggml_mpi_graph_compute_post 函数的定义

void ggml_mpi_graph_compute_post(
        struct ggml_mpi_context * ctx_mpi,
             struct ggml_cgraph * gf,
                            int   n_layers) {
    UNUSED(n_layers); // 标记 n_layers 未使用

    const int mpi_rank = ctx_mpi->rank; // 获取 MPI 进程的排名
    const int mpi_size = ctx_mpi->size; // 获取 MPI 进程的数量

    // 将输出数据发送给下一个节点
    if (mpi_rank > 0) { // 如果当前进程的排名大于 0
        ggml_mpi_tensor_send(gf->nodes[gf->n_nodes - 1], (mpi_rank + 1) % mpi_size); // 调用 MPI 发送数据函数
    }
}
```