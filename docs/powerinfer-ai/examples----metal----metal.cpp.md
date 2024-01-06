# `PowerInfer\examples\metal\metal.cpp`

```
// 使用 Metal 对静态导出的 ggml 计算图进行评估
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
#include <cstdlib> // 包含标准库函数的头文件

int main(int argc, char ** argv) { // 主函数，接受命令行参数
    ggml_time_init(); // 初始化时间

    if (argc != 2) { // 如果参数个数不等于2
        fprintf(stderr, "Usage: %s llama.ggml\n", argv[0]); // 输出错误信息
        return -1; // 返回错误码
    }

    const char * fname_cgraph = argv[1]; // 获取命令行参数中的文件名

    // 加载计算图
    struct ggml_context * ctx_data = NULL; // 定义指向 ggml_context 结构体的指针并初始化为 NULL
    struct ggml_context * ctx_eval = NULL; // 定义指向 ggml_context 结构体的指针并初始化为 NULL

    struct ggml_cgraph * gf = ggml_graph_import(fname_cgraph, &ctx_data, &ctx_eval); // 导入计算图

    // 分配所有 Metal 资源和内存缓冲区
    auto * ctx_metal = ggml_metal_init(1); // 初始化 Metal 资源
    // 获取数据和评估上下文的最大张量大小
    const size_t max_size_data = ggml_get_max_tensor_size(ctx_data);
    const size_t max_size_eval = ggml_get_max_tensor_size(ctx_eval);
    // 将数据上下文的内存缓冲区添加到 Metal 上下文中
    ggml_metal_add_buffer(ctx_metal, "data", ggml_get_mem_buffer(ctx_data), ggml_get_mem_size(ctx_data), max_size_data);
    // 将评估上下文的内存缓冲区添加到 Metal 上下文中
    ggml_metal_add_buffer(ctx_metal, "eval", ggml_get_mem_buffer(ctx_eval), ggml_get_mem_size(ctx_eval), max_size_eval);

    // 主函数
    {
        // 获取名为 "embd" 的张量
        struct ggml_tensor * input = ggml_graph_get_tensor(gf, "embd");
        // 将输入数据设置为 1（BOS）
        *(int32_t *) input->data = 1; // BOS

        // 将输入张量设置到 Metal 上下文中
        ggml_metal_set_tensor(ctx_metal, input);

        // 预热
        ggml_metal_graph_compute(ctx_metal, gf);

        // 迭代次数
        const int n_iter = 16;

        // 记录开始时间
        const int64_t t0 = ggml_time_us();
// 实际的推断过程发生在这里
for (int i = 0; i < n_iter; ++i) {
    ggml_metal_graph_compute(ctx_metal, gf);
}

// 记录结束时间
const int64_t t1 = ggml_time_us();

// 输出推断所花费的时间和每个标记的平均时间
printf("time: %.2f ms, %.2f ms/tok\n", (t1 - t0) / 1000.0, (t1 - t0) / 1000.0 / n_iter);
}

// 调试输出
{
    // 获取最后一个节点的输出张量
    struct ggml_tensor * logits = gf->nodes[gf->n_nodes - 1];
    ggml_metal_get_tensor(ctx_metal, logits);

    // 获取张量数据的指针
    float * ptr = (float *) ggml_get_data(logits);

    // 输出张量的值
    printf("logits: ");
    for (int i = 0; i < 10; i++) {
        printf("%8.4f ", ptr[i]);
        }
        // 打印换行
        printf("\n");
        // 初始化最大值索引和总和
        int imax = 0;
        double sum = 0.0;
        double vmax = -1e9;
        // 遍历数组，计算总和并找到最大值及其索引
        for (int i = 0; i < 32000; i++) {
            sum += (double) ptr[i];
            if (ptr[i] > vmax) {
                vmax = ptr[i];
                imax = i;
            }
        }
        // 打印总和、最大值索引和最大值
        printf("sum: %f, imax = %d, vmax = %f\n", sum, imax, vmax);
    }

    // 释放 Metal 上下文
    ggml_metal_free(ctx_metal);

    // 释放数据上下文
    ggml_free(ctx_data);
    // 释放评估上下文
    ggml_free(ctx_eval);
# 返回整数值0
return 0;
```