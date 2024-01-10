# `ggml\examples\mnist\main-cpu.cpp`

```
// Use a pre-generated MNIST compute graph for inference on the CPU
//
// You can generate a compute graph using the "mnist" tool:
//
// $ ./bin/mnist ./models/mnist/ggml-model-f32.bin ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
//
// This command creates the "mnist.ggml" file, which contains the generated compute graph.
// Now, you can re-use the compute graph with the "mnist-cpu" tool:
//
// $ ./bin/mnist-cpu ./models/mnist/mnist.ggml ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
//

#include "ggml/ggml.h"

#include <algorithm>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <fstream>
#include <vector>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// evaluate the MNIST compute graph
//
//   - fname_cgraph: path to the compute graph
//   - n_threads:    number of threads to use
//   - digit:        784 pixel values
//
// returns 0 - 9 prediction
int mnist_eval(
        const char * fname_cgraph,
        const int n_threads,
        std::vector<float> digit) {
    // load the compute graph
    struct ggml_context * ctx_data = NULL;  // 创建一个指向 ggml_context 结构体的指针，并初始化为 NULL
    struct ggml_context * ctx_eval = NULL;  // 创建一个指向 ggml_context 结构体的指针，并初始化为 NULL

    struct ggml_cgraph * gfi = ggml_graph_import(fname_cgraph, &ctx_data, &ctx_eval);  // 导入计算图

    // param export/import test
    GGML_ASSERT(ggml_graph_get_tensor(gfi, "fc1_bias")->op_params[0] == int(0xdeadbeef));  // 对导入的计算图进行参数导入/导出测试

    // allocate work context
    // needed during ggml_graph_compute() to allocate a work tensor
    static size_t buf_size = 128ull*1024*1024; // TODO  // 分配工作上下文所需的内存大小
    static void * buf = malloc(buf_size);  // 分配工作上下文所需的内存空间

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,  // 内存大小
        /*.mem_buffer =*/ buf,  // 内存缓冲区
        /*.no_alloc   =*/ false,  // 是否允许分配内存
    };

    struct ggml_context * ctx_work = ggml_init(params);  // 初始化工作上下文

    struct ggml_tensor * input = ggml_graph_get_tensor(gfi, "input");  // 获取输入张量
    memcpy(input->data, digit.data(), ggml_nbytes(input));  // 将输入数据拷贝到输入张量的数据中

    ggml_graph_compute_with_ctx(ctx_work, gfi, n_threads);  // 使用上下文进行计算图的计算
    # 获取指向概率数据的指针
    const float * probs_data = ggml_get_data_f32(ggml_graph_get_tensor(gfi, "probs"));

    # 计算概率数据中的最大元素的索引
    const int prediction = std::max_element(probs_data, probs_data + 10) - probs_data;

    # 释放工作上下文的内存
    ggml_free(ctx_work);
    # 释放数据上下文的内存
    ggml_free(ctx_data);
    # 释放评估上下文的内存
    ggml_free(ctx_eval);

    # 返回预测结果
    return prediction;
}

int main(int argc, char ** argv) {
    // 用当前时间初始化随机数种子
    srand(time(NULL));
    // 初始化时间
    ggml_time_init();

    // 如果参数个数不等于3，输出使用说明并退出程序
    if (argc != 3) {
        fprintf(stderr, "Usage: %s models/mnist/mnist.ggml models/mnist/t10k-images.idx3-ubyte\n", argv[0]);
        exit(0);
    }

    // 创建一个长度为784的无符号字节数组
    uint8_t buf[784];
    // 创建一个存储浮点数的向量
    std::vector<float> digit;

    // 从测试集中读取一个随机数字
    {
        // 以二进制方式打开文件
        std::ifstream fin(argv[2], std::ios::binary);
        // 如果文件打开失败，输出错误信息并返回1
        if (!fin) {
            fprintf(stderr, "%s: failed to open '%s'\n", __func__, argv[2]);
            return 1;
        }

        // 定位到一个随机的数字：16字节的头部 + 28*28 * (0 - 10000之间的随机数)
        fin.seekg(16 + 784 * (rand() % 10000));
        // 读取数据到buf中
        fin.read((char *) &buf, sizeof(buf));
    }

    // 以ASCII形式渲染数字
    {
        // 调整digit的大小为buf的大小
        digit.resize(sizeof(buf));

        for (int row = 0; row < 28; row++) {
            for (int col = 0; col < 28; col++) {
                // 如果buf中的值大于230，输出'*'，否则输出'_'
                fprintf(stderr, "%c ", (float)buf[row*28 + col] > 230 ? '*' : '_');
                // 将buf中的值存储到digit中
                digit[row*28 + col] = ((float)buf[row*28 + col]);
            }

            fprintf(stderr, "\n");
        }

        fprintf(stderr, "\n");
    }

    // 对数字进行预测
    const int prediction = mnist_eval(argv[1], 1, digit);

    // 输出预测的数字
    fprintf(stdout, "%s: predicted digit is %d\n", __func__, prediction);

    return 0;
}
```