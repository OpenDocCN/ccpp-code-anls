# `ggml\examples\mnist\main-mtl.cpp`

```
// 使用预生成的 MNIST 计算图在 M1 GPU 上进行推断，通过 MPS
//
// 您可以使用 "mnist" 工具生成计算图：
//
// $ ./bin/mnist ./models/mnist/ggml-model-f32.bin ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
//
// 此命令将创建 "mnist.ggml" 文件，其中包含生成的计算图。
// 现在，您可以使用 "mnist-mtl" 工具在 GPU 上重复使用计算图：
//
// $ ./bin/mnist-mtl ./models/mnist/mnist.ggml ../examples/mnist/models/mnist/t10k-images.idx3-ubyte
//

#include "ggml/ggml.h"

#include "main-mtl.h"

#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <fstream>
#include <vector>

// 评估 MNIST 计算图
//
//   - fname_cgraph: 计算图路径
//   - digit:        784 个像素值
//
// 返回 0 - 9 的预测结果
int mnist_eval(
        const char * fname_cgraph,
        std::vector<float> digit
        ) {
    // 加载计算图
    struct ggml_context * ctx_data = NULL;
    struct ggml_context * ctx_eval = NULL;

    struct ggml_cgraph * gf = ggml_graph_import(fname_cgraph, &ctx_data, &ctx_eval);

    // 分配工作上下文
    static size_t buf_size = 128ull*1024*1024; // TODO
    static void * buf = malloc(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx_work = ggml_init(params);

    // 这将分配所有 Metal 资源和内存缓冲区
    auto ctx_mtl = mnist_mtl_init(ctx_data, ctx_eval, ctx_work, gf);

    int prediction = -1;
    // 循环执行一次
    for (int i = 0; i < 1; ++i) {
        // 从图中获取名为"input"的张量
        struct ggml_tensor * input = ggml_graph_get_tensor(gf, "input");

        // 如果 i 除以 2 的余数为 0
        if (i % 2 == 0) {
            // 将 digit 的数据拷贝到 input 的数据中
            memcpy(input->data, digit.data(), ggml_nbytes(input));
        } else {
            // 否则，将 input 的数据全部置为 0
            memset(input->data, 0, ggml_nbytes(input));
        }

        // 实际推断操作在这里发生
        prediction = mnist_mtl_eval(ctx_mtl, gf);
    }

    // 释放 mnist_mtl 上下文
    mnist_mtl_free(ctx_mtl);

    // 释放工作上下文
    ggml_free(ctx_work);
    // 释放数据上下文
    ggml_free(ctx_data);
    // 释放评估上下文
    ggml_free(ctx_eval);

    // 返回推断结果
    return prediction;
}

int main(int argc, char ** argv) {
    // 生成随机种子
    srand(time(NULL));
    // 初始化时间
    ggml_time_init();

    // 检查命令行参数数量是否正确
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
        // 如果文件打开失败，则输出错误信息并返回1
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
                // 如果buf中的值大于230，则输出'*'，否则输出'_'
                fprintf(stderr, "%c ", (float)buf[row*28 + col] > 230 ? '*' : '_');
                // 将buf中的值存储到digit中
                digit[row*28 + col] = ((float)buf[row*28 + col]);
            }

            fprintf(stderr, "\n");
        }

        fprintf(stderr, "\n");
    }

    // 对数字进行预测
    const int prediction = mnist_eval(argv[1], digit);

    // 输出预测的数字
    fprintf(stdout, "%s: predicted digit is %d\n", __func__, prediction);

    return 0;
}
```