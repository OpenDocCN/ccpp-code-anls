# `ggml\examples\mnist\main-cnn.cpp`

```cpp
#include "ggml/ggml.h"  // 包含 ggml 库的头文件

#include "common.h"  // 包含通用的头文件

#include <cmath>  // 包含数学函数库
#include <cstdio>  // 包含输入输出函数库
#include <cstring>  // 包含字符串处理函数库
#include <ctime>  // 包含时间函数库
#include <fstream>  // 包含文件输入输出函数库
#include <string>  // 包含字符串处理函数库
#include <vector>  // 包含向量容器函数库
#include <algorithm>  // 包含算法函数库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止特定警告
#endif

struct mnist_model {  // 定义 mnist_model 结构体
    struct ggml_tensor * conv2d_1_kernel;  // 卷积层1的核张量
    struct ggml_tensor * conv2d_1_bias;  // 卷积层1的偏置张量
    struct ggml_tensor * conv2d_2_kernel;  // 卷积层2的核张量
    struct ggml_tensor * conv2d_2_bias;  // 卷积层2的偏置张量
    struct ggml_tensor * dense_weight;  // 全连接层的权重张量
    struct ggml_tensor * dense_bias;  // 全连接层的偏置张量
    struct ggml_context * ctx;  // ggml 上下文
};

bool mnist_model_load(const std::string & fname, mnist_model & model) {  // 加载 mnist 模型
    struct gguf_init_params params = {  // 初始化 gguf 参数结构体
        /*.no_alloc   =*/ false,  // 不进行内存分配
        /*.ctx        =*/ &model.ctx,  // 上下文指针
    };
    gguf_context * ctx = gguf_init_from_file(fname.c_str(), params);  // 从文件中初始化 gguf 上下文
    if (!ctx) {  // 如果上下文为空
        fprintf(stderr, "%s: gguf_init_from_file() failed\n", __func__);  // 输出错误信息
        return false;  // 返回失败
    }
    model.conv2d_1_kernel = ggml_get_tensor(model.ctx, "kernel1");  // 获取卷积层1的核张量
    model.conv2d_1_bias = ggml_get_tensor(model.ctx, "bias1");  // 获取卷积层1的偏置张量
    model.conv2d_2_kernel = ggml_get_tensor(model.ctx, "kernel2");  // 获取卷积层2的核张量
    model.conv2d_2_bias = ggml_get_tensor(model.ctx, "bias2");  // 获取卷积层2的偏置张量
    model.dense_weight = ggml_get_tensor(model.ctx, "dense_w");  // 获取全连接层的权重张量
    model.dense_bias = ggml_get_tensor(model.ctx, "dense_b");  // 获取全连接层的偏置张量
    return true;  // 返回成功
}

int mnist_eval(
        const mnist_model & model,
        const int n_threads,
        std::vector<float> digit,
        const char * fname_cgraph
        )
{
    static size_t buf_size = 100000 * sizeof(float) * 4;  // 静态变量，缓冲区大小
    static void * buf = malloc(buf_size);  // 静态变量，分配缓冲区内存

    struct ggml_init_params params = {  // 初始化 ggml 参数结构体
        /*.mem_size   =*/ buf_size,  // 内存大小
        /*.mem_buffer =*/ buf,  // 内存缓冲区
        /*.no_alloc   =*/ false,  // 不进行内存分配
    };

    struct ggml_context * ctx0 = ggml_init(params);  // 初始化 ggml 上下文
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);  // 创建新的计算图

    struct ggml_tensor * input = ggml_new_tensor_4d(ctx0, GGML_TYPE_F32, 28, 28, 1, 1);  // 创建新的4维张量
    // 将 digit.data() 的数据拷贝到 input->data 中，拷贝的字节数由 ggml_nbytes(input) 决定
    memcpy(input->data, digit.data(), ggml_nbytes(input));
    // 设置 input 的名称为 "input"
    ggml_set_name(input, "input");
    // 使用 ctx0 上下文，对 input 进行二维卷积操作，得到 cur
    ggml_tensor * cur = ggml_conv_2d(ctx0, model.conv2d_1_kernel, input, 1, 1, 0, 0, 1, 1);
    // 对 cur 进行加法操作，加上 model.conv2d_1_bias
    cur = ggml_add(ctx0, cur, model.conv2d_1_bias);
    // 对 cur 进行激活函数操作，使用 relu
    cur = ggml_relu(ctx0, cur);
    // 输出经过 Conv2D 操作后的形状信息
    // Output shape after Conv2D: (26 26 32 1)
    cur = ggml_pool_2d(ctx0, cur, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    // 输出经过 MaxPooling2D 操作后的形状信息
    // Output shape after MaxPooling2D: (13 13 32 1)
    cur = ggml_conv_2d(ctx0, model.conv2d_2_kernel, cur, 1, 1, 0, 0, 1, 1);
    cur = ggml_add(ctx0, cur, model.conv2d_2_bias);
    cur = ggml_relu(ctx0, cur);
    // 输出经过 Conv2D 操作后的形状信息
    // Output shape after Conv2D: (11 11 64 1)
    cur = ggml_pool_2d(ctx0, cur, GGML_OP_POOL_MAX, 2, 2, 2, 2, 0, 0);
    // 输出经过 MaxPooling2D 操作后的形状信息
    // Output shape after MaxPooling2D: (5 5 64 1)
    cur = ggml_cont(ctx0, ggml_permute(ctx0, cur, 1, 2, 0, 3));
    // 输出经过 permute 操作后的形状信息
    // Output shape after permute: (64 5 5 1)
    cur = ggml_reshape_2d(ctx0, cur, 1600, 1);
    // 最终的全连接层操作
    cur = ggml_add(ctx0, ggml_mul_mat(ctx0, model.dense_weight, cur), model.dense_bias);
    // 对 cur 进行 softmax 操作，得到概率值
    ggml_tensor * probs = ggml_soft_max(ctx0, cur);
    // 设置 probs 的名称为 "probs"
    ggml_set_name(probs, "probs");

    // 构建前向传播计算图
    ggml_build_forward_expand(gf, probs);
    // 使用 ctx0 上下文，计算计算图 gf 的前向传播，使用 n_threads 个线程
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 输出计算图的 DOT 格式文件，文件名为 "mnist-cnn.dot"
    ggml_graph_dump_dot(gf, NULL, "mnist-cnn.dot");

    if (fname_cgraph) {
        // 导出计算图，用于以后使用
        // 参考 "mnist-cpu" 示例
        ggml_graph_export(gf, fname_cgraph);

        // 输出导出计算图的信息
        fprintf(stderr, "%s: exported compute graph to '%s'\n", __func__, fname_cgraph);
    }

    // 获取 probs 的数据，类型为 float
    const float * probs_data = ggml_get_data_f32(probs);
    // 获取概率值最大的索引，作为预测结果
    const int prediction = std::max_element(probs_data, probs_data + 10) - probs_data;
    // 释放 ctx0 上下文所占用的资源
    ggml_free(ctx0);
    // 返回预测结果
    return prediction;
}

int main(int argc, char ** argv) {
    // 用当前时间初始化随机数种子
    srand(time(NULL));
    // 初始化时间
    ggml_time_init();

    // 检查命令行参数数量是否为3
    if (argc != 3) {
        // 打印使用方法
        fprintf(stderr, "Usage: %s models/mnist/mnist-cnn.gguf models/mnist/t10k-images.idx3-ubyte\n", argv[0]);
        // 退出程序
        exit(0);
    }

    // 创建一个长度为784的字节数组
    uint8_t buf[784];
    // 定义 mnist_model 结构体
    mnist_model model;
    // 创建一个存储浮点数的向量
    std::vector<float> digit;

    // 加载模型
    {
        // 记录加载模型的起始时间
        const int64_t t_start_us = ggml_time_us();

        // 如果加载模型失败，则打印错误信息并返回1
        if (!mnist_model_load(argv[1], model)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, argv[1]);
            return 1;
        }

        // 计算加载模型所花费的时间
        const int64_t t_load_us = ggml_time_us() - t_start_us;

        // 打印加载模型所花费的时间
        fprintf(stdout, "%s: loaded model in %8.2f ms\n", __func__, t_load_us / 1000.0f);
    }

    // 从测试集中读取一个随机数字
    {
        // 以二进制方式打开文件
        std::ifstream fin(argv[2], std::ios::binary);
        // 如果打开文件失败，则打印错误信息并返回1
        if (!fin) {
            fprintf(stderr, "%s: failed to open '%s'\n", __func__, argv[2]);
            return 1;
        }

        // 定位到一个随机的数字：16字节的头部 + 28*28 * (0 - 10000之间的随机数)
        fin.seekg(16 + 784 * (rand() % 10000));
        // 读取数据到 buf 中
        fin.read((char *) &buf, sizeof(buf));
    }

    // 以 ASCII 形式渲染数字
    {
        // 调整 digit 的大小为 buf 的大小
        digit.resize(sizeof(buf));

        for (int row = 0; row < 28; row++) {
            for (int col = 0; col < 28; col++) {
                // 根据像素值输出 '*' 或 '_'
                fprintf(stderr, "%c ", (float)buf[row*28 + col] > 230 ? '*' : '_');
                // 将像素值归一化并存储到 digit 中
                digit[row*28 + col] = ((float)buf[row*28 + col] / 255.0f);
            }

            fprintf(stderr, "\n");
        }

        fprintf(stderr, "\n");
    }

    // 对数字进行预测
    const int prediction = mnist_eval(model, 1, digit, nullptr);
    // 打印预测的数字
    fprintf(stdout, "%s: predicted digit is %d\n", __func__, prediction);
    // 释放模型的上下文
    ggml_free(model.ctx);
    // 返回0表示程序正常结束
    return 0;
}
```