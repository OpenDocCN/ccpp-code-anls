# `ggml\tests\test-xpos.c`

```cpp
#include "ggml/ggml.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>

// 定义一个函数，用于判断两个浮点数是否在给定的误差范围内相等
bool is_close(float a, float b, float epsilon) {
    return fabs(a - b) < epsilon;
}

// 主函数
int main(int argc, char ** argv) {
    // 定义常量
    const int n_threads = 1;
    const int n_embd_head = 4; // aka head_dim
    const int n_head = 1;
    const int N = 8;

    // 初始化参数结构体
    struct ggml_init_params params = {
        .mem_size   = 16*1024*1024,
        .mem_buffer = NULL,
    };

    // 初始化 GGML 上下文
    // 内存分配发生在这里
    struct ggml_context * ctx = ggml_init(params);

    // 创建新的三维张量 Q 和 K
    struct ggml_tensor * Q = ggml_new_tensor_3d(ctx, GGML_TYPE_F32, n_embd_head, n_head, N);
    struct ggml_tensor * K = ggml_new_tensor_3d(ctx, GGML_TYPE_F32, n_embd_head, n_head, N);

    // 对张量 Q 和 K 进行赋值
    for (int i = 0; i < ggml_nelements(Q); i++) {
        ((float*) Q->data)[i] = 2.0f;
        ((float*) K->data)[i] = 2.0f;
    }

    // 创建新的一维张量 KQ_pos，并对其进行赋值
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, N);
    int * data = (int *) KQ_pos->data;
    for (int i = 0; i < N; ++i) {
        data[i] = 1 + i;
    }

    // 对张量 Q 和 K 进行原位操作
    struct ggml_tensor * Qx = ggml_rope_xpos_inplace(ctx, Q, KQ_pos, n_embd_head, 512.0f, false);
    struct ggml_tensor * Kx = ggml_rope_xpos_inplace(ctx, K, KQ_pos, n_embd_head, 512.0f, true);

    // 创建新的计算图
    struct ggml_cgraph * gf = ggml_new_graph(ctx);
    // 构建前向扩展
    ggml_build_forward_expand(gf, Qx);
    ggml_build_forward_expand(gf, Kx);
    // 使用上下文计算计算图
    ggml_graph_compute_with_ctx(ctx, gf, n_threads);

    // 输出 Qx 的预期结果
    // -0.6009  2.7568  1.9782  2.0182
    // -2.6379  0.9815  1.9562  2.0361
    // -2.2457 -1.6853  1.9341  2.0538
    //  0.2043 -2.7934  1.9118  2.0712
    //  2.4550 -1.3341  1.8894  2.0884
    //  2.4430  1.3417  1.8668  2.1054
    //  0.1905  2.7739  1.8440  2.1221
    // -2.2257  1.6550  1.8212  2.1386
    for (int i = 0; i < ggml_nelements(Q); i++) {
        if (((float*) Qx->data)[i] > 0) printf(" ");
        printf("%.4f ", ((float*) Qx->data)[i]);
        if ((i+1) % n_embd_head == 0) printf("\n");
    }
    printf("\n");
}
    // 检查 Qx 数据中特定位置的值是否接近预期值
    GGML_ASSERT(is_close(((float*) Qx->data)[7 * n_embd_head + 0], -2.2257f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Qx->data)[7 * n_embd_head + 1],  1.6550f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Qx->data)[7 * n_embd_head + 2],  1.8212f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Qx->data)[7 * n_embd_head + 3],  2.1386f, 0.0001f));

    // 期望的 Kx 输出:
    // -0.6038  2.7703  1.9816  2.0216
    // -2.6639  0.9911  1.9630  2.0431
    // -2.2789 -1.7103  1.9441  2.0644
    //  0.2083 -2.8486  1.9251  2.0856
    //  2.5158 -1.3671  1.9057  2.1065
    //  2.5158  1.3816  1.8862  2.1273
    //  0.1972  2.8705  1.8665  2.1479
    // -2.3146  1.7211  1.8465  2.1684

    // 遍历 Kx 数据并打印每个值，每行结束时换行
    for (int i = 0; i < ggml_nelements(K); i++) {
        if (((float*) Kx->data)[i] > 0) printf(" ");
        printf("%.4f ", ((float*) Kx->data)[i]);
        if ((i+1) % n_embd_head == 0) printf("\n");
    }
    printf("\n");

    // 检查 Kx 数据中特定位置的值是否接近预期值
    GGML_ASSERT(is_close(((float*) Kx->data)[7 * n_embd_head + 0], -2.3146f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Kx->data)[7 * n_embd_head + 1],  1.7211f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Kx->data)[7 * n_embd_head + 2],  1.8465f, 0.0001f));
    GGML_ASSERT(is_close(((float*) Kx->data)[7 * n_embd_head + 3],  2.1684f, 0.0001f));

    // 释放上下文内存
    ggml_free(ctx);

    // 返回 0 表示成功
    return 0;
# 闭合前面的函数定义
```