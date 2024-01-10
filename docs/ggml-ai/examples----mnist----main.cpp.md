# `ggml\examples\mnist\main.cpp`

```
#include "ggml/ggml.h"  // 引入 ggml 库的头文件

#include "common.h"  // 引入 common.h 头文件

#include <cmath>  // 引入数学函数库
#include <cstdio>  // 引入标准输入输出库
#include <cstring>  // 引入字符串处理库
#include <ctime>  // 引入时间处理库
#include <fstream>  // 引入文件输入输出库
#include <string>  // 引入字符串库
#include <vector>  // 引入向量库
#include <algorithm>  // 引入算法库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止特定警告
#endif

// 默认超参数
struct mnist_hparams {
    int32_t n_input   = 784;  // 输入层节点数
    int32_t n_hidden  = 500;  // 隐藏层节点数
    int32_t n_classes = 10;   // 分类数
};

struct mnist_model {
    mnist_hparams hparams;  // mnist 模型的超参数

    struct ggml_tensor * fc1_weight;  // 全连接层1的权重
    struct ggml_tensor * fc1_bias;  // 全连接层1的偏置

    struct ggml_tensor * fc2_weight;  // 全连接层2的权重
    struct ggml_tensor * fc2_bias;  // 全连接层2的偏置

    struct ggml_context * ctx;  // ggml 上下文
};

// 从文件加载模型的权重
bool mnist_model_load(const std::string & fname, mnist_model & model) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());  // 打印加载模型的信息

    auto fin = std::ifstream(fname, std::ios::binary);  // 以二进制方式打开文件
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());  // 如果打开文件失败，则打印错误信息
        return false;  // 返回加载失败
    }

    // 验证魔数
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));  // 读取文件中的魔数
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());  // 如果魔数不匹配，则打印错误信息
            return false;  // 返回加载失败
        }
    }

    auto & ctx = model.ctx;  // 获取模型的上下文

    size_t ctx_size = 0;  // 上下文的大小

    {
        const auto & hparams = model.hparams;  // 获取模型的超参数

        const int n_input   = hparams.n_input;  // 输入层节点数
        const int n_hidden  = hparams.n_hidden;  // 隐藏层节点数
        const int n_classes = hparams.n_classes;  // 分类数

        ctx_size += n_input * n_hidden * ggml_type_size(GGML_TYPE_F32); // fc1 权重
        ctx_size +=           n_hidden * ggml_type_size(GGML_TYPE_F32); // fc1 偏置

        ctx_size += n_hidden * n_classes * ggml_type_size(GGML_TYPE_F32); // fc2 权重
        ctx_size +=            n_classes * ggml_type_size(GGML_TYPE_F32); // fc2 偏置

        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0));  // 打印上下文的大小
    }
}
    // 创建 ggml 上下文
    {
        // 初始化参数结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size + 1024*1024,  // 设置内存大小为 ctx_size + 1MB
            /*.mem_buffer =*/ NULL,  // 内存缓冲区为空
            /*.no_alloc   =*/ false,  // 允许分配内存
        };

        // 初始化 ggml 上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，打印错误信息并返回 false
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 读取 FC1 层 1
    {
        // 读取维度
        int32_t n_dims;
        fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));

        {
            int32_t ne_weight[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne_weight[i]), sizeof(ne_weight[i]));
            }

            // 从文件中获取 FC1 的维度，例如 768x500
            model.hparams.n_input  = ne_weight[0];
            model.hparams.n_hidden = ne_weight[1];

            // 创建 FC1 权重张量
            model.fc1_weight = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, model.hparams.n_input, model.hparams.n_hidden);
            // 从文件中读取 FC1 权重数据
            fin.read(reinterpret_cast<char *>(model.fc1_weight->data), ggml_nbytes(model.fc1_weight));
            // 设置张量名称
            ggml_set_name(model.fc1_weight, "fc1_weight");
        }

        {
            int32_t ne_bias[2] = { 1, 1 };
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne_bias[i]), sizeof(ne_bias[i]));
            }

            // 创建 FC1 偏置张量
            model.fc1_bias = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, model.hparams.n_hidden);
            // 从文件中读取 FC1 偏置数据
            fin.read(reinterpret_cast<char *>(model.fc1_bias->data), ggml_nbytes(model.fc1_bias));
            // 设置张量名称
            ggml_set_name(model.fc1_bias, "fc1_bias");

            // 仅用于测试目的，将一些参数设置为非零值
            model.fc1_bias->op_params[0] = 0xdeadbeef;
        }
    }

    // 读取 FC2 层 2
    {
        // 读取维度信息
        int32_t n_dims;
        fin.read(reinterpret_cast<char *>(&n_dims), sizeof(n_dims));

        {
            // 初始化权重数组为 {1, 1}
            int32_t ne_weight[2] = { 1, 1 };
            // 读取每个维度的权重信息
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne_weight[i]), sizeof(ne_weight[i]));
            }

            // 从文件中获取FC1层的维度，例如 10x500
            model.hparams.n_classes = ne_weight[1];

            // 创建并读取FC2层的权重数据
            model.fc2_weight = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, model.hparams.n_hidden, model.hparams.n_classes);
            fin.read(reinterpret_cast<char *>(model.fc2_weight->data), ggml_nbytes(model.fc2_weight));
            ggml_set_name(model.fc2_weight, "fc2_weight");
        }

        {
            // 初始化偏置数组为 {1, 1}
            int32_t ne_bias[2] = { 1, 1 };
            // 读取每个维度的偏置信息
            for (int i = 0; i < n_dims; ++i) {
                fin.read(reinterpret_cast<char *>(&ne_bias[i]), sizeof(ne_bias[i]));
            }

            // 创建并读取FC2层的偏置数据
            model.fc2_bias = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, model.hparams.n_classes);
            fin.read(reinterpret_cast<char *>(model.fc2_bias->data), ggml_nbytes(model.fc2_bias));
            ggml_set_name(model.fc2_bias, "fc2_bias");
        }
    }

    // 关闭文件流
    fin.close();

    // 返回true
    return true;
// 评估模型
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - digit:     784个像素值
//   - fname_cgraph: 保存计算图的文件名
//
// 返回 0 - 9 的预测结果
int mnist_eval(
        const mnist_model & model,
        const int n_threads,
        std::vector<float> digit,
        const char * fname_cgraph
        ) {

    const auto & hparams = model.hparams;

    static size_t buf_size = hparams.n_input * sizeof(float) * 32;
    static void * buf = malloc(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    // 初始化计算图上下文
    struct ggml_context * ctx0 = ggml_init(params);
    // 创建新的计算图
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 创建输入张量
    struct ggml_tensor * input = ggml_new_tensor_1d(ctx0, GGML_TYPE_F32, hparams.n_input);
    // 将像素值拷贝到输入张量中
    memcpy(input->data, digit.data(), ggml_nbytes(input));
    ggml_set_name(input, "input");

    // 全连接层1：MLP = Ax + b
    ggml_tensor * fc1 = ggml_add(ctx0, ggml_mul_mat(ctx0, model.fc1_weight, input),                model.fc1_bias);
    ggml_tensor * fc2 = ggml_add(ctx0, ggml_mul_mat(ctx0, model.fc2_weight, ggml_relu(ctx0, fc1)), model.fc2_bias);

    // Softmax
    ggml_tensor * probs = ggml_soft_max(ctx0, fc2);
    ggml_set_name(probs, "probs");

    // 构建 / 导出 / 运行计算图
    ggml_build_forward_expand(gf, probs);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 导出计算图以供以后使用
    if (fname_cgraph) {
        ggml_graph_export(gf, "mnist.ggml");
        fprintf(stderr, "%s: exported compute graph to '%s'\n", __func__, fname_cgraph);
    }

    // 获取概率数据
    const float * probs_data = ggml_get_data_f32(probs);

    // 获取最大概率对应的预测结果
    const int prediction = std::max_element(probs_data, probs_data + 10) - probs_data;

    // 释放内存
    ggml_free(ctx0);

    return prediction;
}

#ifdef __cplusplus
extern "C" {
#endif
int wasm_eval(uint8_t * digitPtr) {
    mnist_model model; // 定义一个名为model的mnist_model结构体变量
    if (!mnist_model_load("models/mnist/ggml-model-f32.bin", model)) { // 如果加载模型失败
        fprintf(stderr, "error loading model\n"); // 输出错误信息到标准错误流
        return -1; // 返回-1
    }
    std::vector<float> digit(digitPtr, digitPtr + 784); // 创建一个包含784个元素的浮点数向量digit，从digitPtr指向的内存开始
    int result = mnist_eval(model, 1, digit, nullptr); // 使用加载的模型对数字进行评估，结果存储在result中
    ggml_free(model.ctx); // 释放模型的上下文内存

    return result; // 返回评估结果
}

int wasm_random_digit(char * digitPtr) {
    auto fin = std::ifstream("models/mnist/t10k-images.idx3-ubyte", std::ios::binary); // 打开mnist测试图像文件
    if (!fin) { // 如果文件打开失败
        fprintf(stderr, "failed to open digits file\n"); // 输出错误信息到标准错误流
        return 0; // 返回0
    }
    srand(time(NULL)); // 使用当前时间初始化随机数生成器

    // Seek to a random digit: 16-byte header + 28*28 * (random 0 - 10000)
    fin.seekg(16 + 784 * (rand() % 10000)); // 定位到一个随机的数字位置：16字节的头部 + 28*28 * (0 - 10000之间的随机数)
    fin.read(digitPtr, 784); // 读取784字节的数据到digitPtr指向的内存中

    return 1; // 返回1
}

#ifdef __cplusplus
}
#endif

int main(int argc, char ** argv) {
    srand(time(NULL)); // 使用当前时间初始化随机数生成器
    ggml_time_init(); // 初始化ggml时间

    if (argc != 3) { // 如果命令行参数不等于3
        fprintf(stderr, "Usage: %s models/mnist/ggml-model-f32.bin models/mnist/t10k-images.idx3-ubyte\n", argv[0]); // 输出使用说明到标准错误流
        exit(0); // 退出程序
    }

    uint8_t buf[784]; // 定义一个包含784个元素的无符号8位整数数组buf
    mnist_model model; // 定义一个名为model的mnist_model结构体变量
    std::vector<float> digit; // 定义一个浮点数向量digit

    // load the model
    {
        const int64_t t_start_us = ggml_time_us(); // 获取当前时间的微秒数

        if (!mnist_model_load(argv[1], model)) { // 如果加载模型失败
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, "models/ggml-model-f32.bin"); // 输出错误信息到标准错误流
            return 1; // 返回1
        }

        const int64_t t_load_us = ggml_time_us() - t_start_us; // 计算加载模型所花费的时间

        fprintf(stdout, "%s: loaded model in %8.2f ms\n", __func__, t_load_us / 1000.0f); // 输出加载模型所花费的时间到标准输出流
    }

    // read a random digit from the test set
    {
        std::ifstream fin(argv[2], std::ios::binary); // 打开mnist测试图像文件
        if (!fin) { // 如果文件打开失败
            fprintf(stderr, "%s: failed to open '%s'\n", __func__, argv[2]); // 输出错误信息到标准错误流
            return 1; // 返回1
        }

        // seek to a random digit: 16-byte header + 28*28 * (random 0 - 10000)
        fin.seekg(16 + 784 * (rand() % 10000)); // 定位到一个随机的数字位置：16字节的头部 + 28*28 * (0 - 10000之间的随机数)
        fin.read((char *) &buf, sizeof(buf)); // 读取784字节的数据到buf数组中
    }
}
    // 将数字以ASCII形式呈现
    {
        // 调整数字的大小以适配缓冲区
        digit.resize(sizeof(buf));

        // 遍历28x28的像素点
        for (int row = 0; row < 28; row++) {
            for (int col = 0; col < 28; col++) {
                // 根据像素值输出相应的字符
                fprintf(stderr, "%c ", (float)buf[row*28 + col] > 230 ? '*' : '_');
                // 将像素值存入digit数组
                digit[row*28 + col] = ((float)buf[row*28 + col]);
            }

            // 换行
            fprintf(stderr, "\n");
        }

        // 输出空行
        fprintf(stderr, "\n");
    }

    // 使用mnist_eval函数对数字进行预测
    const int prediction = mnist_eval(model, 1, digit, "mnist.ggml");

    // 输出预测结果
    fprintf(stdout, "%s: predicted digit is %d\n", __func__, prediction);

    // 释放模型上下文
    ggml_free(model.ctx);

    // 返回0表示成功
    return 0;
# 闭合前面的函数定义
```