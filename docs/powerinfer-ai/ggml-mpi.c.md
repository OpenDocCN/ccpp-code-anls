# `PowerInfer\ggml-mpi.c`

```
#include "ggml-mpi.h"  // 包含 MPI 相关的头文件

#include "ggml.h"  // 包含 ggml 库的头文件

#include <mpi.h>  // 包含 MPI 标准头文件

#include <stdio.h>  // 包含标准输入输出库的头文件
#include <stdlib.h>  // 包含标准库的头文件

#define MIN(a, b) ((a) < (b) ? (a) : (b))  // 定义一个宏，返回两个数中较小的那个

#define UNUSED GGML_UNUSED  // 定义一个宏，表示未使用的变量

struct ggml_mpi_context {
    int rank;  // MPI 进程的排名
    int size;  // MPI 进程的总数
};

void ggml_mpi_backend_init(void) {
    MPI_Init(NULL, NULL);  // 初始化 MPI 环境
// 释放 MPI 资源
void ggml_mpi_backend_free(void) {
    MPI_Finalize();
}

// 初始化 MPI 上下文
struct ggml_mpi_context * ggml_mpi_init(void) {
    // 分配内存空间
    struct ggml_mpi_context * ctx = calloc(1, sizeof(struct ggml_mpi_context));
    
    // 获取当前进程在通信器中的排名
    MPI_Comm_rank(MPI_COMM_WORLD, &ctx->rank);
    // 获取通信器中的进程数
    MPI_Comm_size(MPI_COMM_WORLD, &ctx->size);

    return ctx;
}

// 释放 MPI 上下文
void ggml_mpi_free(struct ggml_mpi_context * ctx) {
    free(ctx);
}

// 获取当前进程在通信器中的排名
int ggml_mpi_rank(struct ggml_mpi_context * ctx) {
// 返回上下文中的排名
return ctx->rank;
}

// 初始化 MPI 计算环境
void ggml_mpi_eval_init(
        struct ggml_mpi_context * ctx_mpi,
                            int * n_tokens,
                            int * n_past,
                            int * n_threads) {
    UNUSED(ctx_mpi); // 未使用的参数

    // 同步工作节点参数与根节点
    MPI_Barrier(MPI_COMM_WORLD); // 同步所有节点

    // 广播 n_tokens 参数
    MPI_Bcast(n_tokens,  1, MPI_INT, 0, MPI_COMM_WORLD);
    // 广播 n_past 参数
    MPI_Bcast(n_past,    1, MPI_INT, 0, MPI_COMM_WORLD);
    // 广播 n_threads 参数
    MPI_Bcast(n_threads, 1, MPI_INT, 0, MPI_COMM_WORLD);
}

// 获取图中节点的索引
static int ggml_graph_get_node_idx(struct ggml_cgraph * gf, const char * name) {
    // 获取图中张量的指针
    struct ggml_tensor * t = ggml_graph_get_tensor(gf, name);
    // 如果输入的张量指针为空，输出错误信息并返回-1
    if (t == NULL) {
        fprintf(stderr, "%s: tensor %s not found\n", __func__, name);
        return -1;
    }

    // 遍历图中的节点，查找是否存在与输入张量相同的节点，如果存在则返回节点的索引
    for (int i = 0; i < gf->n_nodes; i++) {
        if (gf->nodes[i] == t) {
            return i;
        }
    }

    // 如果未找到与输入张量相同的节点，输出错误信息并返回-1
    fprintf(stderr, "%s: tensor %s not found in graph (should not happen)\n", __func__, name);
    return -1;
}

// 根据 MPI 协议发送张量数据
static void ggml_mpi_tensor_send(struct ggml_tensor * t, int mpi_rank_dst) {
    MPI_Datatype mpi_type;

    // 根据张量的数据类型选择相应的 MPI 数据类型
    switch (t->type) {
        case GGML_TYPE_I32: mpi_type = MPI_INT32_T; break;
// 根据数据类型选择相应的 MPI 数据类型
switch (t->type) {
    case GGML_TYPE_I32: mpi_type = MPI_INT32_T; break; // 如果数据类型为 GGML_TYPE_I32，选择 MPI_INT32_T 类型
    case GGML_TYPE_F32: mpi_type = MPI_FLOAT;   break; // 如果数据类型为 GGML_TYPE_F32，选择 MPI_FLOAT 类型
    default: GGML_ASSERT(false && "not implemented"); // 如果数据类型不是已实现的类型，抛出错误
}

// 使用 MPI_Send 函数将数据发送到指定的 MPI 进程
const int retval = MPI_Send(t->data, ggml_nelements(t), mpi_type, mpi_rank_dst, 0, MPI_COMM_WORLD);
GGML_ASSERT(retval == MPI_SUCCESS); // 检查 MPI_Send 函数是否成功执行

// 接收来自指定 MPI 进程的数据
static void ggml_mpi_tensor_recv(struct ggml_tensor * t, int mpi_rank_src) {
    MPI_Datatype mpi_type;

    switch (t->type) {
        case GGML_TYPE_I32: mpi_type = MPI_INT32_T; break; // 如果数据类型为 GGML_TYPE_I32，选择 MPI_INT32_T 类型
        case GGML_TYPE_F32: mpi_type = MPI_FLOAT;   break; // 如果数据类型为 GGML_TYPE_F32，选择 MPI_FLOAT 类型
        default: GGML_ASSERT(false && "not implemented"); // 如果数据类型不是已实现的类型，抛出错误
    }

    MPI_Status status; UNUSED(status); // 定义 MPI 状态变量并标记为未使用

    // 使用 MPI_Recv 函数接收来自指定 MPI 进程的数据
    const int retval = MPI_Recv(t->data, ggml_nelements(t), mpi_type, mpi_rank_src, MPI_ANY_TAG, MPI_COMM_WORLD, &status);
// 确保 MPI 操作返回成功
GGML_ASSERT(retval == MPI_SUCCESS);
}

// TODO: 这个实现可以进行许多改进
void ggml_mpi_graph_compute_pre(
        struct ggml_mpi_context * ctx_mpi,
             struct ggml_cgraph * gf,
                            int   n_layers) {
    // 获取 MPI 进程的排名和大小
    const int mpi_rank = ctx_mpi->rank;
    const int mpi_size = ctx_mpi->size;

    // 获取图中名为 'inp_tokens' 的张量
    struct ggml_tensor * inp_tokens = ggml_graph_get_tensor(gf, "inp_tokens");
    // 如果张量不存在，打印错误信息并返回
    if (inp_tokens == NULL) {
        fprintf(stderr, "%s: tensor 'inp_tokens' not found\n", __func__);
        return;
    }

    // 获取图中名为 'layer_inp_0' 的张量
    struct ggml_tensor * inp0 = ggml_graph_get_tensor(gf, "layer_inp_0");
    // 如果张量不存在，打印错误信息
    if (inp0 == NULL) {
        fprintf(stderr, "%s: tensor 'inp0' not found\n", __func__);
```

        return;
    }
    // 如果输入参数为 gf->nodes[0]，则断言为真，否则抛出异常
    GGML_ASSERT(inp0 == gf->nodes[0]);

    // 将计算图分布到 MPI 节点的切片中
    //
    // 主节点（0）处理最后的层 + 计算图的剩余部分
    // 并负责将输入令牌传递给第一个节点（1）
    //
    // 节点 1:   [(  0) * n_per_node, (  1) * n_per_node)
    // 节点 2:   [(  1) * n_per_node, (  2) * n_per_node)
    // ...
    // 节点 n-1: [(n-2) * n_per_node, (n-1) * n_per_node)
    // 节点 0:   [(n-1) * n_per_node,            n_nodes)
    //
    if (mpi_rank > 0) {
        if (mpi_rank == 1) {
            // 第一个节点（1）从主节点（0）接收输入令牌
            ggml_mpi_tensor_recv(inp_tokens, 0);
        } else {
            // 如果不是第一个节点，则接收每个节点的输入数据到“inp0”张量（即计算图中的第一个节点）
            ggml_mpi_tensor_recv(inp0, mpi_rank - 1);
        }
    } else if (mpi_size > 1) {
        // 节点 0 将输入标记发送到节点 1
        ggml_mpi_tensor_send(inp_tokens, 1);

        // 从最后一个节点接收输出数据
        ggml_mpi_tensor_recv(inp0, mpi_size - 1);
    }

    {
        // 每个节点的层数
        const int n_per_node = (n_layers + (mpi_size - 1)) / mpi_size;

        // MPI 索引
        const int mpi_idx = mpi_rank > 0 ? mpi_rank - 1 : mpi_size - 1;

        // 每个节点的起始层数
        const int il0 =               (mpi_idx + 0) * n_per_node;
        // 每个节点的结束层数
        const int il1 = MIN(n_layers, (mpi_idx + 1) * n_per_node);
// 定义两个字符数组，用于存储节点名称
char name_l0[GGML_MAX_NAME];
char name_l1[GGML_MAX_NAME];

// 根据 il0 和 il1 的值格式化节点名称
snprintf(name_l0, sizeof(name_l0), "layer_inp_%d", il0);
snprintf(name_l1, sizeof(name_l1), "layer_inp_%d", il1);

// 获取节点名称对应的索引
const int idx_l0 = ggml_graph_get_node_idx(gf, name_l0);
const int idx_l1 = mpi_rank > 0 ? ggml_graph_get_node_idx(gf, name_l1) + 1 : gf->n_nodes;

// 如果节点索引小于 0，则打印错误信息并返回
if (idx_l0 < 0 || idx_l1 < 0) {
    fprintf(stderr, "%s: layer input nodes not found\n", __func__);
    return;
}

// 将输入数据附加到所有需要的节点上
// TODO: 不太好 - 应该能够在不修改计算图的情况下完成这一步骤（见下面的 TODO）
for (int i = idx_l0; i < idx_l1; i++) {
    // 如果节点的第一个源节点是 idx_l0 对应的节点，则将其修改为 inp0
    if (gf->nodes[i]->src[0] == gf->nodes[idx_l0]) {
        gf->nodes[i]->src[0] =  inp0;
    }
}
        // 如果节点 i 的第二个源节点是 idx_l0 节点，则将其替换为 inp0
        if (gf->nodes[i]->src[1] == gf->nodes[idx_l0]) {
            gf->nodes[i]->src[1] =  inp0;
        }
    }

    // TODO: 与重新排列节点不同，我们应该能够执行计算图的子集
    for (int i = 1; i < idx_l1 - idx_l0; i++) {
        // 将节点 i 替换为 idx_l0 + i 的节点
        gf->nodes[i] = gf->nodes[idx_l0 + i];
        // 将梯度 i 替换为 idx_l0 + i 的梯度
        gf->grads[i] = gf->grads[idx_l0 + i];
    }

    // 第一个节点执行“get_rows”操作，其余节点从前一个节点获取数据
    if (mpi_idx != 0) {
        // 如果 mpi_idx 不等于 0，则将第一个节点的操作设置为 GGML_OP_NONE
        gf->nodes[0]->op = GGML_OP_NONE;
    }

    // 计算节点数量
    gf->n_nodes = idx_l1 - idx_l0;

    // 输出处理节点的信息
    //fprintf(stderr, "%s: node %d: processing %d nodes [%d, %d)\n", __func__, mpi_rank, gf->n_nodes, il0, il1);
// 定义一个名为 ggml_mpi_graph_compute_post 的函数，用于计算 MPI 图的后处理
void ggml_mpi_graph_compute_post(
        struct ggml_mpi_context * ctx_mpi,  // MPI 上下文
             struct ggml_cgraph * gf,       // 图结构
                            int   n_layers) {  // 层数

    UNUSED(n_layers);  // 未使用的参数 n_layers

    const int mpi_rank = ctx_mpi->rank;  // 获取当前 MPI 进程的排名
    const int mpi_size = ctx_mpi->size;  // 获取 MPI 进程的总数

    // 将输出数据发送给下一个节点
    if (mpi_rank > 0) {  // 如果当前进程的排名大于 0
        ggml_mpi_tensor_send(gf->nodes[gf->n_nodes - 1], (mpi_rank + 1) % mpi_size);  // 调用 MPI 发送张量的函数
    }
}
```