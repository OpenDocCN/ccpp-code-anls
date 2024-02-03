# `PowerInfer\examples\metal\metal.cpp`

```cpp
// 使用 Metal 执行静态导出的 ggml 计算图
//
// - 首先，导出一个 LLaMA 图：
//
//  $ ./bin/main -m ../models/7B/ggml-model-q4_0.gguf --export
//
// - 运行此工具以评估导出的图：
//
//  $ ./bin/metal llama.ggml
//
// 此工具的目的主要是用于调试和演示目的。
// 导出计算图的主要限制是它们的大小是静态的，这通常对实际应用程序是一个问题。
//

#include "ggml.h"
#include "ggml-metal.h"

#include <cstdio>
#include <cstring>
#include <cstdlib>

int main(int argc, char ** argv) {
    ggml_time_init();

    if (argc != 2) {
        fprintf(stderr, "Usage: %s llama.ggml\n", argv[0]);
        return -1;
    }

    const char * fname_cgraph = argv[1];

    // 加载计算图
    struct ggml_context * ctx_data = NULL;
    struct ggml_context * ctx_eval = NULL;

    struct ggml_cgraph * gf = ggml_graph_import(fname_cgraph, &ctx_data, &ctx_eval);

    // 这将分配所有 Metal 资源和内存缓冲区
    auto * ctx_metal = ggml_metal_init(1);

    const size_t max_size_data = ggml_get_max_tensor_size(ctx_data);
    const size_t max_size_eval = ggml_get_max_tensor_size(ctx_eval);
    ggml_metal_add_buffer(ctx_metal, "data", ggml_get_mem_buffer(ctx_data), ggml_get_mem_size(ctx_data), max_size_data);
    ggml_metal_add_buffer(ctx_metal, "eval", ggml_get_mem_buffer(ctx_eval), ggml_get_mem_size(ctx_eval), max_size_eval);

    // 主函数
    {
        // 获取输入张量指针，张量名为"embd"
        struct ggml_tensor * input = ggml_graph_get_tensor(gf, "embd");
        // 将输入张量的数据设置为1（BOS）
        *(int32_t *) input->data = 1; // BOS

        // 将输入张量传递给 Metal 上下文
        ggml_metal_set_tensor(ctx_metal, input);

        // 预热
        ggml_metal_graph_compute(ctx_metal, gf);

        // 迭代次数
        const int n_iter = 16;

        // 记录开始时间
        const int64_t t0 = ggml_time_us();

        // 实际推理过程发生在这里
        for (int i = 0; i < n_iter; ++i) {
            ggml_metal_graph_compute(ctx_metal, gf);
        }

        // 记录结束时间
        const int64_t t1 = ggml_time_us();

        // 输出推理时间和每个标记的平均时间
        printf("time: %.2f ms, %.2f ms/tok\n", (t1 - t0) / 1000.0, (t1 - t0) / 1000.0 / n_iter);
    }

    // 调试输出
    {
        // 获取输出张量指针，即图中最后一个节点的张量
        struct ggml_tensor * logits = gf->nodes[gf->n_nodes - 1];
        // 从 Metal 上下文中获取输出张量
        ggml_metal_get_tensor(ctx_metal, logits);

        // 获取输出张量数据的指针
        float * ptr = (float *) ggml_get_data(logits);

        // 输出 logits
        printf("logits: ");
        for (int i = 0; i < 10; i++) {
            printf("%8.4f ", ptr[i]);
        }
        printf("\n");

        // 寻找最大值和计算总和
        int imax = 0;
        double sum = 0.0;
        double vmax = -1e9;
        for (int i = 0; i < 32000; i++) {
            sum += (double) ptr[i];
            if (ptr[i] > vmax) {
                vmax = ptr[i];
                imax = i;
            }
        }
        // 输出总和、最大值的索引和值
        printf("sum: %f, imax = %d, vmax = %f\n", sum, imax, vmax);
    }

    // 释放 Metal 上下文
    ggml_metal_free(ctx_metal);

    // 释放数据上下文
    ggml_free(ctx_data);
    // 释放评估上下文
    ggml_free(ctx_eval);

    // 返回成功
    return 0;
}
# 代码块结束
```