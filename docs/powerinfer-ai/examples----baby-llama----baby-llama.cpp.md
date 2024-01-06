# `PowerInfer\examples\baby-llama\baby-llama.cpp`

```
// 包含自定义的头文件 ggml.h 和 train.h
#include "ggml.h"
#include "train.h"

// 包含标准库的头文件
#include <vector> // 向量
#include <cassert> // 断言
#include <cstdlib> // 通用工具函数
#include <cstring> // 字符串操作
#include <random> // 随机数生成
#include <vector> // 向量

// 如果是 MSC 编译器，禁用警告 4244 和 4267，可能会丢失数据
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267)
#endif

// 如果定义了 LLAMA_DEFAULT_RMS_EPS，则使用其值作为 rms_norm_eps，否则使用默认值 5e-6f
#ifdef LLAMA_DEFAULT_RMS_EPS
constexpr float rms_norm_eps = LLAMA_DEFAULT_RMS_EPS;
#else
constexpr float rms_norm_eps = 5e-6f;
#endif
// 辅助函数，用于计算图形的计算
static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    // 根据图形和线程数创建计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划中有工作量
    if (plan.work_size > 0) {
        // 调整缓冲区大小
        buf.resize(plan.work_size);
        // 将工作数据指针指向缓冲区
        plan.work_data = buf.data();
    }

    // 执行图形计算
    ggml_graph_compute(graph, &plan);
}

// 随机化张量的值
static struct ggml_tensor * randomize_tensor(
    struct ggml_tensor * tensor, int ndims, const int64_t ne[], float fmin, float fmax
) {
    // 根据张量维度进行不同的处理
    switch (ndims) {
        // 一维张量
        case 1:
            // 遍历张量的第一维度
            for (int i0 = 0; i0 < ne[0]; i0++) {
                // 为张量赋予随机值
                ((float *)tensor->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
# 根据不同的情况，对多维数组进行赋值操作
case 2:
    # 遍历第二维和第一维，对每个元素赋值
    for (int i1 = 0; i1 < ne[1]; i1++) {
        for (int i0 = 0; i0 < ne[0]; i0++) {
            # 根据随机数生成的方式，对每个元素赋值
            ((float *)tensor->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
        }
    }
    break;
case 3:
    # 遍历第三维、第二维和第一维，对每个元素赋值
    for (int i2 = 0; i2 < ne[2]; i2++) {
        for (int i1 = 0; i1 < ne[1]; i1++) {
            for (int i0 = 0; i0 < ne[0]; i0++) {
                # 根据随机数生成的方式，对每个元素赋值
                ((float *)tensor->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
            }
        }
    }
    break;
case 4:
    # 遍历第四维、第三维、第二维和第一维，对每个元素赋值
    for (int i3 = 0; i3 < ne[3]; i3++) {
        for (int i2 = 0; i2 < ne[2]; i2++) {
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    # 根据随机数生成的方式，对每个元素赋值
                    ((float *)tensor->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
        }
    }
    break;
// 循环遍历第一个维度的元素
for (int i0 = 0; i0 < ne[0]; i0++) {
    // 计算当前元素在一维数组中的索引，并赋值为随机数乘以范围加上最小值
    ((float *)tensor->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
}
// 结束第一个维度的循环

// 结构体 llama_hparams 的成员变量
struct llama_hparams {
    uint32_t n_vocab = 32000;  // 词汇量大小
    uint32_t n_ctx   = 512;    // 上下文大小，可能是用户输入的值
    uint32_t n_embd  = 4096;   // 嵌入维度
    uint32_t n_mult  = 4;      // 嵌入维度乘数
    uint32_t n_head  = 32;     // 多头注意力机制的头数
    // 定义一个名为 n_layer 的无符号 32 位整数变量，并赋值为 32
    uint32_t n_layer = 32;
    // 定义一个名为 n_rot 的无符号 32 位整数变量，并赋值为 64
    uint32_t n_rot   = 64;

    // 定义一个名为 llama_hparams 的结构体，包含不相等运算符重载函数
    bool operator!=(const llama_hparams & other) const {
        // 使用 memcmp 函数比较当前对象和另一个对象的内存内容，返回比较结果
        return memcmp(this, &other, sizeof(llama_hparams));
    }
};

// 定义一个名为 get_n_ff 的静态函数，接受一个 llama_hparams 结构体指针参数
static uint32_t get_n_ff(const struct llama_hparams* hparams) {
    // 计算 n_ff 的值并返回
    const uint32_t n_ff = ((2*(4*hparams->n_embd)/3 + hparams->n_mult - 1)/hparams->n_mult)*hparams->n_mult;
    return n_ff;
}

// 定义一个名为 llama_hparams_lora 的结构体，包含多个无符号 32 位整数变量，并赋予默认值
struct llama_hparams_lora {
    uint32_t n_vocab = 32000;
    uint32_t n_ctx   = 512;   // 这是用户输入的吗？
    uint32_t n_embd  = 4096;
    uint32_t n_mult  = 4;
    uint32_t n_head  = 32;
    uint32_t n_layer = 32;
// 定义一个名为 n_rot 的无符号 32 位整数变量，并初始化为 64
uint32_t n_rot = 64;
// 定义一个名为 n_lora 的无符号 32 位整数变量，并初始化为 64
uint32_t n_lora = 64;

// 定义一个名为 llama_hparams_lora 的结构体，包含不等于操作符重载函数
struct llama_hparams_lora {
    // 不等于操作符重载函数，用于比较两个 llama_hparams_lora 结构体对象的内存内容是否相等
    bool operator!=(const llama_hparams_lora & other) const {
        return memcmp(this, &other, sizeof(llama_hparams_lora)) != 0;
    }
};

// 定义一个名为 llama_layer 的结构体
struct llama_layer {
    // 归一化
    struct ggml_tensor * attention_norm;

    // 注意力
    struct ggml_tensor * wq;
    struct ggml_tensor * wk;
    struct ggml_tensor * wv;
    struct ggml_tensor * wo;

    // 归一化
    struct ggml_tensor * ffn_norm;
}
// 定义结构体 llama_layer_lora，包含多个指向 ggml_tensor 结构体的指针
struct llama_layer_lora {
    // normalization，指向 ggml_tensor 结构体的指针
    struct ggml_tensor * attention_norm;

    // attention，指向 ggml_tensor 结构体的指针
    struct ggml_tensor * wqa;
    struct ggml_tensor * wqb;
    struct ggml_tensor * wka;
    struct ggml_tensor * wkb;
    struct ggml_tensor * wva;
    struct ggml_tensor * wvb;
    struct ggml_tensor * woa;
    struct ggml_tensor * wob;
};
// 定义一个指向 ggml_tensor 结构体的指针变量 ffn_norm，用于存储归一化后的数据

// 定义指向 ggml_tensor 结构体的指针变量 w1, w2, w3，用于存储神经网络的权重数据

// 定义一个名为 llama_kv_cache 的结构体，用于存储键值对缓存的相关信息
struct llama_kv_cache {
    // 指向 ggml_context 结构体的指针变量 ctx，初始化为 NULL
    struct ggml_context * ctx = NULL;

    // 指向 ggml_tensor 结构体的指针变量 k, v，用于存储键值对数据
    struct ggml_tensor * k;
    struct ggml_tensor * v;

    // 用于存储缓存中当前的 token 数量
    int n; // number of tokens currently in the cache
```
// 结构体定义，用于存储模型相关的信息
struct llama_model {
    // 指向 ggml_context 结构体的指针，初始化为 NULL
    struct ggml_context * ctx = NULL;

    // 存储模型的超参数
    llama_hparams hparams;

    // 指向 ggml_tensor 结构体的指针，用于存储 token embeddings
    struct ggml_tensor * tok_embeddings;

    // 指向 ggml_tensor 结构体的指针，用于存储 normalization
    struct ggml_tensor * norm;
    // 指向 ggml_tensor 结构体的指针，用于存储模型输出
    struct ggml_tensor * output;

    // 存储模型的各个层
    std::vector<llama_layer> layers;
};

// 结构体定义，用于存储 LORA 模型相关的信息
struct llama_model_lora {
    // 指向 ggml_context 结构体的指针，初始化为 NULL
    struct ggml_context * ctx = NULL;

    // 存储 LORA 模型的超参数
    llama_hparams_lora hparams;
// 声明一个指向 ggml_tensor 结构体的指针变量 tok_embeddings

// 声明指向 ggml_tensor 结构体的指针变量 norm
// 声明指向 ggml_tensor 结构体的指针变量 outputa
// 声明指向 ggml_tensor 结构体的指针变量 outputb

// 声明一个名为 layers 的 llama_layer_lora 结构体的向量

// 初始化模型的函数，接受一个指向 llama_model 结构体的指针 model
// 从 model 中获取 hparams，并赋值给常引用 hparams
// 从 hparams 中获取 n_embd、n_layer、n_vocab，并赋值给常量 n_embd、n_layer、n_vocab
// 调用 get_n_ff 函数获取 n_ff，并赋值给常量 n_ff
// 从 model 中获取 ctx，并赋值给指向 ggml_context 结构体的指针变量 ctx
// 创建一个二维的浮点型张量，表示词嵌入矩阵，大小为 n_embd * n_vocab
model->tok_embeddings = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab); // ("tok_embeddings.weight", {n_embd, n_vocab});

// 创建一个一维的浮点型张量，表示归一化层的权重，大小为 n_embd
model->norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd); // ("norm.weight", {n_embd});

// 创建一个二维的浮点型张量，表示输出层的权重，大小为 n_embd * n_vocab
model->output = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab); // ("output.weight", {n_embd, n_vocab});

// 调整模型的层数为 n_layer
model->layers.resize(n_layer);
for (uint32_t i = 0; i < n_layer; ++i) {
    auto & layer = model->layers[i];

    // 创建一个一维的浮点型张量，表示注意力层的归一化权重，大小为 n_embd
    layer.attention_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd); // (layers_i + ".attention_norm.weight", {n_embd});

    // 创建一个二维的浮点型张量，表示注意力层的查询权重，大小为 n_embd * n_embd
    layer.wq = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd); // (layers_i + ".attention.wq.weight", {n_embd, n_embd});

    // 创建一个二维的浮点型张量，表示注意力层的键权重，大小为 n_embd * n_embd
    layer.wk = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd); // (layers_i + ".attention.wk.weight", {n_embd, n_embd});

    // 创建一个二维的浮点型张量，表示注意力层的值权重，大小为 n_embd * n_embd
    layer.wv = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd); // (layers_i + ".attention.wv.weight", {n_embd, n_embd});

    // 创建一个二维的浮点型张量，表示注意力层的输出权重，大小为 n_embd * n_embd
    layer.wo = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd); // (layers_i + ".attention.wo.weight", {n_embd, n_embd});

    // 创建一个一维的浮点型张量，表示前馈神经网络层的归一化权重，大小为 n_embd
    layer.ffn_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd); // (layers_i + ".ffn_norm.weight", {n_embd});

    // 创建一个二维的浮点型张量，表示前馈神经网络层的第一层权重，大小为 n_embd * n_ff
    layer.w1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff); // (layers_i + ".feed_forward.w1.weight", {n_embd, n_ff});
}
// 初始化模型的 Lora 结构
static void init_model_lora(struct llama_model_lora * model) {
    // 获取模型超参数
    const auto & hparams = model->hparams;

    // 获取超参数中的各个值
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_mult  = hparams.n_mult;
    const uint32_t n_layer = hparams.n_layer;
    const uint32_t n_vocab = hparams.n_vocab;
    const uint32_t n_lora  = hparams.n_lora;

    // 计算 n_ff 的值
    const uint32_t n_ff = ((2*(4*n_embd)/3 + n_mult - 1)/n_mult)*n_mult;

    // 获取模型的上下文
    struct ggml_context * ctx = model->ctx;

    // 初始化模型的 tok_embeddings 属性，创建一个 n_embd x n_vocab 的浮点型张量
    model->tok_embeddings = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab); // ("tok_embeddings.weight", {n_embd, n_vocab});
}
    // 创建一个一维的浮点型张量，表示模型的归一化权重
    model->norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd); // ("norm.weight", {n_embd});

    // 创建一个二维的浮点型张量，表示模型的输出权重
    model->outputa = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_vocab); // ("output.weight", {n_embd, n_vocab});
    model->outputb = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora); // ("output.weight", {n_embd, n_vocab});

    // 调整模型的层数
    model->layers.resize(n_layer);
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = model->layers[i];

        // 创建一维的浮点型张量，表示模型的注意力归一化权重
        layer.attention_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd); // (layers_i + ".attention_norm.weight", {n_embd});

        // 创建二维的浮点型张量，表示模型的注意力权重
        layer.wqa = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_embd); // (layers_i + ".attention.wq.weight", {n_embd, n_embd});
        layer.wqb = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora); // (layers_i + ".attention.wq.weight", {n_embd, n_embd});
        layer.wka = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_embd); // (layers_i + ".attention.wk.weight", {n_embd, n_embd});
        layer.wkb = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora); // (layers_i + ".attention.wk.weight", {n_embd, n_embd});
        layer.wva = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_embd); // (layers_i + ".attention.wv.weight", {n_embd, n_embd});
        layer.wvb = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora); // (layers_i + ".attention.wv.weight", {n_embd, n_embd});
        layer.woa = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_embd); // (layers_i + ".attention.wo.weight", {n_embd, n_embd});
        layer.wob = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora); // (layers_i + ".attention.wo.weight", {n_embd, n_embd});
    }
// 为神经网络层的正则化参数创建一个新的一维张量
layer.ffn_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd); 

// 为神经网络层的前向传播权重创建一个新的二维张量
layer.w1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff); 

// 为神经网络层的前向传播权重创建一个新的二维张量
layer.w2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_ff, n_embd); 

// 为神经网络层的前向传播权重创建一个新的二维张量
layer.w3 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff); 

// 设置模型的参数，包括词嵌入、正则化和输出
static void set_param_model(struct llama_model * model) {
    // 获取模型的超参数
    const auto& hparams = model->hparams;

    // 获取层的数量
    const uint32_t n_layer = hparams.n_layer;

    // 获取模型的上下文
    struct ggml_context* ctx = model->ctx;

    // 设置模型的词嵌入参数
    ggml_set_param(ctx, model->tok_embeddings);

    // 设置模型的正则化参数
    ggml_set_param(ctx, model->norm);

    // 设置模型的输出参数
    ggml_set_param(ctx, model->output);
    // 遍历每个层，设置参数
    for (uint32_t i = 0; i < n_layer; ++i) {
        // 获取当前层的引用
        auto & layer = model->layers[i];

        // 设置注意力规范化参数
        ggml_set_param(ctx, layer.attention_norm);
        // 设置权重参数
        ggml_set_param(ctx, layer.wq);
        ggml_set_param(ctx, layer.wk);
        ggml_set_param(ctx, layer.wv);
        ggml_set_param(ctx, layer.wo);
        // 设置前馈神经网络规范化参数
        ggml_set_param(ctx, layer.ffn_norm);
        // 设置前馈神经网络权重参数
        ggml_set_param(ctx, layer.w1);
        ggml_set_param(ctx, layer.w2);
        ggml_set_param(ctx, layer.w3);
    }
}

// 设置 Lora 模型参数
static void set_param_model_lora(struct llama_model_lora * model) {
    // 获取模型的超参数
    const auto& hparams = model->hparams;

    // 获取层数
    const uint32_t n_layer = hparams.n_layer;
// 将模型的上下文赋值给变量ctx
struct ggml_context* ctx = model->ctx;

// 设置模型的上下文参数
ggml_set_param(ctx, model->tok_embeddings);
ggml_set_param(ctx, model->norm);
ggml_set_param(ctx, model->outputa);
ggml_set_param(ctx, model->outputb);

// 遍历每一层神经网络
for (uint32_t i = 0; i < n_layer; ++i) {
    // 获取当前层的引用
    auto & layer = model->layers[i];

    // 设置当前层的注意力规范化参数
    ggml_set_param(ctx, layer.attention_norm);
    // 设置当前层的查询权重参数
    ggml_set_param(ctx, layer.wqa);
    // 设置当前层的查询偏置参数
    ggml_set_param(ctx, layer.wqb);
    // 设置当前层的键权重参数
    ggml_set_param(ctx, layer.wka);
    // 设置当前层的键偏置参数
    ggml_set_param(ctx, layer.wkb);
    // 设置当前层的值权重参数
    ggml_set_param(ctx, layer.wva);
    // 设置当前层的值偏置参数
    ggml_set_param(ctx, layer.wvb);
    // 设置当前层的输出权重参数
    ggml_set_param(ctx, layer.woa);
    // 设置当前层的输出偏置参数
    ggml_set_param(ctx, layer.wob);
    // 设置当前层的前馈神经网络规范化参数
    ggml_set_param(ctx, layer.ffn_norm);
}
// 设置模型参数为指定值
ggml_set_param(ctx, layer.w1);
ggml_set_param(ctx, layer.w2);
ggml_set_param(ctx, layer.w3);

// 随机初始化模型参数
static void randomize_model(struct llama_model * model, int seed, float mean, float std, float min, float max) {
    // 获取模型超参数
    const auto & hparams = model->hparams;

    // 获取层数
    const uint32_t n_layer = hparams.n_layer;

    // 初始化随机正态分布
    struct random_normal_distribution * rnd = init_random_normal_distribution(seed, mean, std, min, max);

    // 随机初始化模型张量
    randomize_tensor_normal(model->tok_embeddings , rnd);
    randomize_tensor_normal(model->norm           , rnd);
    randomize_tensor_normal(model->output         , rnd);

    // 循环遍历每一层，随机初始化注意力层的张量
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = model->layers[i];
        randomize_tensor_normal(layer.attention_norm, rnd);
// 以正态分布随机初始化层的权重张量 wq
randomize_tensor_normal(layer.wq, rnd);
// 以正态分布随机初始化层的权重张量 wk
randomize_tensor_normal(layer.wk, rnd);
// 以正态分布随机初始化层的权重张量 wv
randomize_tensor_normal(layer.wv, rnd);
// 以正态分布随机初始化层的权重张量 wo
randomize_tensor_normal(layer.wo, rnd);

// 以正态分布随机初始化层的归一化张量 ffn_norm
randomize_tensor_normal(layer.ffn_norm, rnd);

// 以正态分布随机初始化层的权重张量 w1
randomize_tensor_normal(layer.w1, rnd);
// 以正态分布随机初始化层的权重张量 w2
randomize_tensor_normal(layer.w2, rnd);
// 以正态分布随机初始化层的权重张量 w3
randomize_tensor_normal(layer.w3, rnd);

// 释放随机数生成器占用的内存
free_random_normal_distribution(rnd);
}

// 随机初始化 Lora 模型的参数
static void randomize_model_lora(
    struct llama_model_lora * model, int seed, float mean, float std, float min, float max
) {
    // 获取模型的超参数
    const auto & hparams = model->hparams;

    // 获取层数
    const uint32_t n_layer = hparams.n_layer;

    // 初始化一个服从正态分布的随机数生成器
    struct random_normal_distribution * rnd = init_random_normal_distribution(seed, mean, std, min, max);

    // 对模型的嵌入层进行正态分布随机化
    randomize_tensor_normal(model->tok_embeddings, rnd);
    randomize_tensor_normal(model->norm          , rnd);
    randomize_tensor_normal(model->outputa       , rnd);
    randomize_tensor_normal(model->outputb       , rnd);

    // 对每一层进行正态分布随机化
    for (uint32_t i = 0; i < n_layer; ++i) {
        // 获取当前层
        auto & layer = model->layers[i];
        // 对当前层的注意力规范化进行正态分布随机化
        randomize_tensor_normal(layer.attention_norm, rnd);

        // 对当前层的权重进行正态分布随机化
        randomize_tensor_normal(layer.wqa, rnd);
        randomize_tensor_normal(layer.wqb, rnd);
        randomize_tensor_normal(layer.wka, rnd);
        randomize_tensor_normal(layer.wkb, rnd);
        randomize_tensor_normal(layer.wva, rnd);
// 使用正态分布随机初始化神经网络层的权重和偏置
randomize_tensor_normal(layer.wvb, rnd);
randomize_tensor_normal(layer.woa, rnd);
randomize_tensor_normal(layer.wob, rnd);

// 使用正态分布随机初始化神经网络层的前馈神经网络层的归一化参数
randomize_tensor_normal(layer.ffn_norm, rnd);

// 使用正态分布随机初始化神经网络层的前馈神经网络层的权重
randomize_tensor_normal(layer.w1, rnd);
randomize_tensor_normal(layer.w2, rnd);
randomize_tensor_normal(layer.w3, rnd);
}

// 释放随机数生成器占用的内存
free_random_normal_distribution(rnd);
}

// 初始化键值缓存
static void init_kv_cache(struct llama_kv_cache* cache, struct llama_model * model, int n_batch) {
    // 获取模型的超参数
    const auto & hparams = model->hparams;

    // 获取超参数中的上下文长度、嵌入维度和层数
    const uint32_t n_ctx   = hparams.n_ctx;
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;
// 计算内存大小，n_mem为层数*上下文长度*批量大小，n_elements为嵌入大小*内存大小
const int64_t n_mem      = n_layer*n_ctx*n_batch;
const int64_t n_elements = n_embd*n_mem;

// 调整缓存大小，2倍n_elements乘以数据类型大小再加上2倍MB
// cache.buf.resize(2u*n_elements*ggml_type_size(wtype) + 2u*MB);

// 初始化参数结构体
// params.mem_size为缓存大小，params.mem_buffer为缓存地址，params.no_alloc为是否分配内存
if (!cache->ctx) {
    // 设置初始化参数
    struct ggml_init_params params;
    params.mem_size   = 2u*n_elements*ggml_type_size(GGML_TYPE_F32) + 2u*1024*1024;
    params.mem_buffer = NULL;
    params.no_alloc   = false;

    // 初始化缓存上下文
    cache->ctx = ggml_init(params);

    // 如果初始化失败，打印错误信息
    if (!cache->ctx) {
        fprintf(stderr, "%s: failed to allocate memory for kv cache\n", __func__);
    }
}
// 退出程序并返回错误代码1
exit(1);
// 初始化键值缓存的大小
cache->k = ggml_new_tensor_1d(cache->ctx, GGML_TYPE_F32, n_elements);
cache->v = ggml_new_tensor_1d(cache->ctx, GGML_TYPE_F32, n_elements);
// 初始化键值缓存的大小和参数
static bool init_kv_cache_lora(struct llama_kv_cache* cache, struct llama_model_lora * model, int n_batch) {
    // 获取模型的超参数
    const auto & hparams = model->hparams;
    // 获取上下文大小、嵌入大小和层数
    const uint32_t n_ctx   = hparams.n_ctx;
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;
    // 计算内存大小和元素个数
    const int64_t n_mem      = n_layer*n_ctx*n_batch;
    const int64_t n_elements = n_embd*n_mem;
    // 调整缓存的大小
    // cache.buf.resize(2u*n_elements*ggml_type_size(wtype) + 2u*MB);
// 定义结构体 ggml_init_params，并声明变量 params
// 设置 params 的 mem_size 为 cache.buf.size
// 设置 params 的 mem_buffer 为 cache.buf.addr
// 设置 params 的 no_alloc 为 false
if (!cache->ctx) {
    // 如果缓存的上下文为空，则执行以下操作
    // 定义结构体 ggml_init_params，并声明变量 params
    // 设置 params 的 mem_size 为 2 倍元素数量乘以 GGML_TYPE_F32 的大小再加上 2MB
    // 设置 params 的 mem_buffer 为 NULL
    // 设置 params 的 no_alloc 为 false
    // 调用 ggml_init 函数，将返回的上下文赋值给 cache->ctx
    // 如果上下文为空，则打印错误信息并返回 false
}

// 为缓存的键和值分配新的一维张量
cache->k = ggml_new_tensor_1d(cache->ctx, GGML_TYPE_F32, n_elements);
cache->v = ggml_new_tensor_1d(cache->ctx, GGML_TYPE_F32, n_elements);
// 返回 true
    return true;
}

// 前向传播函数，用于模型推断
static struct ggml_tensor * forward(
    struct llama_model    * model,  // 指向模型的指针
    struct llama_kv_cache * cache,  // 指向键值缓存的指针
    struct ggml_context   * ctx0,    // 指向上下文的指针
    struct ggml_cgraph    * gf,      // 指向计算图的指针
    struct ggml_tensor    * tokens_input,  // 输入的张量
    const  int              n_tokens,      // 输入的标记数量
    const  int              n_past         // 过去的标记数量
) {
    const int N = n_tokens;  // 将输入的标记数量赋值给 N

    struct llama_kv_cache& kv_self = *cache;  // 创建指向 cache 的引用
    const auto & hparams = model->hparams;    // 获取模型的超参数
    const int n_ctx   = hparams.n_ctx;        // 获取上下文的大小
    const int n_embd  = hparams.n_embd;       // 获取嵌入的维度
    const int n_layer = hparams.n_layer;      // 获取层数
    # 定义变量n_head和n_rot，分别为hparams中的n_head和n_rot
    const int n_head  = hparams.n_head;
    const int n_rot   = hparams.n_rot;

    # 创建一个1维整型张量tokens，长度为N，并将tokens_input的数据复制到tokens中
    struct ggml_tensor * tokens = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    memcpy(tokens->data, tokens_input->data, N*ggml_element_size(tokens));

    # 获取kv_self中的k和v张量
    struct ggml_tensor * kc = kv_self.k;
    struct ggml_tensor * vc = kv_self.v;

    # 创建一个1维整型张量KQ_pos，长度为N，并将n_past + i的值赋给每个元素
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    {
        int * data = (int *) KQ_pos->data;
        for (int i = 0; i < N; ++i) {
            data[i] = n_past + i;
        }
    }

    # 获取model->tok_embeddings中tokens对应的行，形状为[n_embd,N,1,1]
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model->tok_embeddings, tokens);
    # 遍历n_layer次
    for (int il = 0; il < n_layer; ++il) {
        // 将输入指针赋值给inpSA
        struct ggml_tensor * inpSA = inpL;

        // 声明一个指针cur
        struct ggml_tensor * cur;

        // lctx.use_buf(ctx0, 0);  // 该语句被注释掉，不起作用

        // norm
        {
            // cur的形状为[n_embd,N,1,1]
            cur = ggml_rms_norm(ctx0, inpL, rms_norm_eps);  // 对inpL进行均方根归一化，结果赋值给cur

            // cur = attention_norm*cur
            cur = ggml_mul(ctx0,
                        ggml_repeat(ctx0, model->layers[il].attention_norm, cur),  // 重复attention_norm，结果与cur相乘
                        cur);
        }

        // self-attention
        {
            // 计算Q和K，并对它们进行RoPE处理
// 定义变量 Qcur，表示当前的查询矩阵，通过对当前矩阵进行一系列操作得到
// 定义变量 Kcur，表示当前的键矩阵，通过对当前矩阵进行一系列操作得到

// 存储键和数值到内存
{
    // 计算转置的 [N, n_embd] V 矩阵
    // 定义变量 Vcur，表示当前的数值矩阵，通过对当前矩阵进行一系列操作得到

    // 定义变量 kv_self.k，表示自身键的形状
    // 定义变量 kv_self.v，表示自身数值的形状
    // 定义变量 k，表示键的形状
    // 定义变量 v，表示数值的形状
}
                    // 从 kv_self.k 中创建一个一维的 ggml_tensor 结构体指针 k
                    struct ggml_tensor * k = ggml_view_1d(ctx0, kv_self.k, N*n_embd, (ggml_element_size(kv_self.k)*n_embd)*(il*n_ctx + n_past));
                    // 从 kv_self.v 中创建一个二维的 ggml_tensor 结构体指针 v
                    struct ggml_tensor * v = ggml_view_2d(ctx0, kv_self.v, N, n_embd,
                            (   n_ctx)*ggml_element_size(kv_self.v),
                            (il*n_ctx)*ggml_element_size(kv_self.v)*n_embd + n_past*ggml_element_size(kv_self.v));

                    // 重要：将 RoPE-ed 版本的 K 存储在 KV 缓存中！
                    ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
                    ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
                } //*/

                // 将 Kcur 重塑为一维的 ggml_tensor 结构体指针，并存储在 kc 中
                kc = ggml_set_1d(ctx0, kc, ggml_reshape_1d(ctx0, Kcur, n_embd*N), (ggml_element_size(kv_self.k)*n_embd)*(il*n_ctx + n_past));
                // 将 Vcur 存储在 vc 中
                vc = ggml_set_2d(ctx0, vc, Vcur, (   n_ctx)*ggml_element_size(kv_self.v),
                        (il*n_ctx)*ggml_element_size(kv_self.v)*n_embd + n_past*ggml_element_size(kv_self.v));
            }

            // Qcur 的形状为 [n_embd/n_head, n_head, N, 1]
            // Q 的形状为 [n_embd/n_head, N, n_head, 1]
            // 创建一个新的 ggml_tensor 结构体指针 Q，通过对 Qcur 进行排列得到
            struct ggml_tensor * Q =
                ggml_permute(ctx0,
                        Qcur,
            // 创建一个新的结构体 ggml_tensor，表示经过一系列变换后的张量 K
            // 这里使用了 ggml_permute 函数对 K 进行维度重排
            struct ggml_tensor * K =
                ggml_permute(ctx0,
                        // 使用 ggml_reshape_3d 函数对 K 进行三维重塑
                        ggml_reshape_3d(ctx0,
                            // 使用 ggml_view_1d 函数对 K 进行一维视图
                            ggml_view_1d(ctx0, kc, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(kc)*n_embd),
                            n_embd/n_head, n_head, n_past + N),
                        0, 2, 1, 3);

            // 创建一个新的结构体 ggml_tensor，表示矩阵乘法 KQ
            // 这里使用了 ggml_mul_mat 函数对 K 和 Q 进行矩阵乘法
            struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

            // 创建一个新的结构体 ggml_tensor，表示经过缩放后的矩阵 KQ
            // 这里使用了 ggml_scale 函数对 KQ 进行缩放
            struct ggml_tensor * KQ_scaled =
                ggml_scale(ctx0,
                        KQ,
// 使用 ggml_new_f32 函数创建一个新的浮点数张量，值为 1.0f 除以 sqrtf(float(n_embd)/n_head)
// 这个张量的作用是对输入进行缩放
struct ggml_tensor * KQ_scaled = ggml_new_f32(ctx0, 1.0f/sqrtf(float(n_embd)/n_head));

// 使用 ggml_diag_mask_inf 函数对 KQ_scaled 进行掩码处理，返回一个新的张量 KQ_masked
// KQ_masked 的形状为 [n_past + N, N, n_head, 1]
struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ_scaled, n_past);

// 使用 ggml_soft_max 函数对 KQ_masked 进行 softmax 处理，返回一个新的张量 KQ_soft_max
// KQ_soft_max 的形状为 [n_past + N, N, n_head, 1]
struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);

// 使用 ggml_view_3d 函数对 cached V 进行分割，返回一个新的张量 V
// V 的形状为 [n_past + N, n_embd/n_head, n_head, 1]
// 其中 n_ctx*ggml_element_size(vc) 表示 V 的总长度
// n_ctx*ggml_element_size(vc)*n_embd/n_head 表示每个 head 的长度
// il*n_ctx*ggml_element_size(vc)*n_embd 表示每个 head 的偏移量
struct ggml_tensor * V =
    ggml_view_3d(ctx0, vc,
            n_past + N, n_embd/n_head, n_head,
            n_ctx*ggml_element_size(vc),
            n_ctx*ggml_element_size(vc)*n_embd/n_head,
            il*n_ctx*ggml_element_size(vc)*n_embd);
// 计算 KQV 矩阵乘以 KQ_soft_max 矩阵的结果，得到新的矩阵 KQV
struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);

// 对 KQV 矩阵进行维度置换，得到新的矩阵 KQV_merged
// KQV_merged 的形状为 [n_embd/n_head, n_head, N, 1]
struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

// 对 KQV_merged 矩阵进行连续化处理，并将其形状变换为 [n_embd, N]
// cur 的形状为 [n_embd, N, 1, 1]
cur = ggml_reshape_2d(ctx0, ggml_cont(ctx0, KQV_merged), n_embd, N);

// 使用 model->layers[il].wo 矩阵对 cur 矩阵进行矩阵乘法运算，得到新的矩阵 cur
// cur 的形状为 [n_embd, N, 1, 1]
cur = ggml_mul_mat(ctx0, model->layers[il].wo, cur);
        }

        // 使用缓冲区
        // lctx.use_buf(ctx0, 1);

        // 输入张量的形状为 [n_embd,N,1,1]
        struct ggml_tensor * inpFF = ggml_add(ctx0, cur, inpSA);

        // 前馈神经网络
        {
            // 归一化
            {
                // 当前形状为 [n_embd,N,1,1]
                cur = ggml_rms_norm(ctx0, inpFF, rms_norm_eps);

                // 当前 = ffn_norm*当前
                // 当前形状为 [n_embd,N,1,1]
                cur = ggml_mul(ctx0,
                        ggml_repeat(ctx0, model->layers[il].ffn_norm, cur),
                        cur);
            }
// 临时变量 tmp 的形状为 [n_ff, N, 1, 1]
struct ggml_tensor * tmp = ggml_mul_mat(ctx0,
        model->layers[il].w3,
        cur);

// 当前变量 cur 的形状为 [n_ff, N, 1, 1]
cur = ggml_mul_mat(ctx0,
        model->layers[il].w1,
        cur);

// 使用 SILU 激活函数
// 当前变量 cur 的形状为 [n_ff, N, 1, 1]
cur = ggml_silu(ctx0, cur);

// 当前变量 cur 的形状为 [n_ff, N, 1, 1]
cur = ggml_mul(ctx0, cur, tmp);

// 当前变量 cur 的形状为 [n_embd, N, 1, 1]
cur = ggml_mul_mat(ctx0,
        // 对当前层的权重和输入进行矩阵乘法运算
        model->layers[il].w2,
        cur);
    }

    // cur 的形状为 [n_embd,N,1,1]
    cur = ggml_add(ctx0, cur, inpFF);

    // 为下一层准备输入
    // inpL 的形状为 [n_embd,N,1,1]
    inpL = cur;
}

// 归一化
{

    // inpL 的形状为 [n_embd,N,1,1]
    inpL = ggml_rms_norm(ctx0, inpL, rms_norm_eps);

    // inpL = norm*inpL
    // inpL 的形状为 [n_embd,N,1,1]
// 使用 ggml_mul 函数对输入进行多次操作
inpL = ggml_mul(ctx0, ggml_repeat(ctx0, model->norm, inpL), inpL);

// 将结果赋值给 embeddings
// embeddings = inpL;

// lm_head
// 设置 inpL 的形状为 [n_vocab,N,1,1]
inpL = ggml_mul_mat(ctx0, model->output, inpL);

// 运行计算
ggml_build_forward_expand(gf, inpL);

// 返回 inpL
return inpL;
}

// 定义 forward_batch 函数，接受 model 和 cache 作为参数
static struct ggml_tensor * forward_batch(
    struct llama_model    * model,
    struct llama_kv_cache * cache,
# 定义一个函数，接受多个参数：ctx0, gf, tokens_input, n_tokens, n_past, n_batch
# 参数 ctx0 是指向 struct ggml_context 结构体的指针
# 参数 gf 是指向 struct ggml_cgraph 结构体的指针
# 参数 tokens_input 是指向 struct ggml_tensor 结构体的指针
# 参数 n_tokens 是一个整数
# 参数 n_past 是一个整数
# 参数 n_batch 是一个整数
) {
    # 定义一个常量 N，其值等于 n_tokens
    const int N = n_tokens;

    # 定义一个引用 kv_self，指向 struct llama_kv_cache 结构体
    struct llama_kv_cache& kv_self = *cache;
    # 定义一个常量引用 hparams，指向 model->hparams
    const auto & hparams = model->hparams;
    # 定义一系列常量，分别表示模型参数中的一些值
    const int n_ctx   = hparams.n_ctx;
    const int n_vocab = hparams.n_vocab;
    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_head  = hparams.n_head;
    const int n_rot   = hparams.n_rot;
    const int n_ff    = get_n_ff(&hparams);

    # 创建一个新的一维张量 tokens，其上下文为 ctx0，数据类型为 GGML_TYPE_I32，大小为 N*n_batch
    struct ggml_tensor * tokens = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N*n_batch);
    // 将 tokens_input->data 的数据复制到 tokens->data 中，复制的长度为 ggml_element_size(tokens)*N*n_batch
    memcpy(tokens->data, tokens_input->data, ggml_element_size(tokens)*N*n_batch);

    // 创建指向 kv_self.k 的指针 kc
    struct ggml_tensor * kc = kv_self.k;
    // 创建指向 kv_self.v 的指针 vc
    struct ggml_tensor * vc = kv_self.v;

    // 创建一个一维的 ggml_tensor KQ_pos，数据类型为 GGML_TYPE_I32，长度为 N
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    {
        // 将 KQ_pos->data 转换为 int 类型的指针
        int * data = (int *) KQ_pos->data;
        // 将 n_past + i 的值赋给 KQ_pos->data 中的每个元素
        for (int i = 0; i < N; ++i) {
            data[i] = n_past + i;
        }
    }

    // 从 model->tok_embeddings 中获取指定行的数据，存储到 inpL 中
    // inpL 的形状为 [n_embd,N*n_batch,1]
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model->tok_embeddings, tokens);
    // 断言 inpL 的形状为 n_embd * N*n_batch
    assert_shape_2d(inpL, n_embd, N*n_batch);

    // 遍历 n_layer 次
    for (int il = 0; il < n_layer; ++il) {
        // 将 inpL 赋值给 inpSA
        struct ggml_tensor * inpSA = inpL;
        // 声明一个指向 ggml_tensor 结构体的指针 cur

        // 使用 lctx 对象的 use_buf 方法，参数为 ctx0 和 0

        // norm
        {
            // cur 的形状为 [n_embd,N*n_batch,1,1]
            // 调用 ggml_rms_norm 方法，参数为 ctx0, inpL, rms_norm_eps，将结果赋给 cur
            cur = ggml_rms_norm(ctx0, inpL, rms_norm_eps);
            // 断言 cur 的形状为 2 维，大小为 n_embd, N*n_batch

            // cur = attention_norm*cur
            // 调用 ggml_mul 方法，参数为 ctx0, ggml_repeat(ctx0, model->layers[il].attention_norm, cur), cur，将结果赋给 cur
            cur = ggml_mul(ctx0,
                        ggml_repeat(ctx0, model->layers[il].attention_norm, cur),
                        cur);
            // 断言 cur 的形状为 2 维，大小为 n_embd, N*n_batch
        }

        // self-attention
        {
            // 计算 Q 和 K，并对它们进行 RoPE 处理
            // 计算 Qcur 和 Kcur 张量
            struct ggml_tensor * Qcur = ggml_rope(ctx0, ggml_reshape_4d(ctx0, ggml_mul_mat(ctx0, model->layers[il].wq, cur), n_embd/n_head, n_head, N, n_batch), KQ_pos, n_rot, 0, 0);
            struct ggml_tensor * Kcur = ggml_rope(ctx0, ggml_reshape_4d(ctx0, ggml_mul_mat(ctx0, model->layers[il].wk, cur), n_embd/n_head, n_head, N, n_batch), KQ_pos, n_rot, 0, 0);
            assert_shape_4d(Qcur, n_embd/n_head, n_head, N, n_batch);
            assert_shape_4d(Kcur, n_embd/n_head, n_head, N, n_batch);

            // 存储键和值到内存
            {
                // 计算转置后的 [N, n_embd] V 矩阵
                // wv   shape [n_embd, n_embd, 1, 1]
                // Vcur shape [N, n_embd, n_batch, 1]
                struct ggml_tensor * Vcur = ggml_cont(ctx0,
                    ggml_permute(ctx0,
                        ggml_reshape_3d(ctx0,
                            ggml_mul_mat(ctx0,
                                model->layers[il].wv,
                                cur),
                n_embd, N, n_batch), // 定义变量n_embd, N, n_batch
                1, 0, 2, 3)); // 设置参数

        assert_shape_3d(Vcur, N, n_embd, n_batch); // 确保Vcur的形状为3维

        // kv_self.k shape [n_embd * n_ctx * n_batch * n_layer]
        // kv_self.v shape [n_ctx * n_embd * n_batch * n_layer]
        // k         shape [n_embd * N, n_batch]   == kv_self.k[:,n_past:n_past+N,:,il]
        // v         shape [N, n_embd, n_batch, 1] == kv_self.v[:,n_past:n_past+N,:,il]

        /* {
            struct ggml_tensor * k = ggml_view_1d(ctx0, kv_self.k, N*n_embd, (ggml_element_size(kv_self.k)*n_embd)*(il*n_ctx + n_past));
            struct ggml_tensor * v = ggml_view_2d(ctx0, kv_self.v, N, n_embd,
                    (   n_ctx)*ggml_element_size(kv_self.v),
                    (il*n_ctx)*ggml_element_size(kv_self.v)*n_embd + n_past*ggml_element_size(kv_self.v));

            // important: storing RoPE-ed version of K in the KV cache!
            ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
            ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));
        } //*/
// 使用 ggml_set_2d 函数对 kc 进行设置，reshape_2d 函数对 Kcur 进行重塑，计算偏移量并设置 kc 的值
kc = ggml_set_2d(ctx0, kc,
                ggml_reshape_2d(ctx0, Kcur, n_embd*N, n_batch),
                ggml_element_size(kc)*n_embd*n_ctx,
                (ggml_element_size(kc)*n_embd)*(il*n_batch*n_ctx + n_past));
// 使用 ggml_set_2d 函数对 vc 进行设置，reshape_2d 函数对 Vcur 进行重塑，计算偏移量并设置 vc 的值
vc = ggml_set_2d(ctx0, vc,
                ggml_reshape_2d(ctx0, Vcur, N*n_embd, n_batch),
                ggml_element_size(vc)*n_ctx*n_embd,
                ggml_element_size(vc)*(n_past + il*n_embd*n_batch*n_ctx));
// 断言 kc 和 vc 的形状是否符合预期
assert_shape_1d(kc, n_embd * n_ctx * n_batch * n_layer);
assert_shape_1d(vc, n_embd * n_ctx * n_batch * n_layer);

// 对 Qcur 进行形状变换，将维度 0, 2, 1, 3 进行置换，得到新的 Q 张量
// Qcur 的形状为 [n_embd/n_head, n_head, N, n_batch]
// Q 的形状为 [n_embd/n_head, N, n_head, n_batch]
struct ggml_tensor * Q =
    ggml_permute(ctx0,
            Qcur,
            0, 2, 1, 3);
// 确保输入的张量 Q 的形状为 4 维
assert_shape_4d(Q, n_embd/n_head, N, n_head, n_batch);

// 创建一个新的张量 K，对输入的张量 kc 进行重塑和排列，以匹配后续的计算需求
struct ggml_tensor * K =
    ggml_permute(ctx0,
        ggml_reshape_4d(ctx0,
            ggml_view_3d(ctx0,
                kc,
                n_embd,
                (n_past + N),
                n_batch,
                n_embd*ggml_element_size(kc),
                n_ctx*n_embd*ggml_element_size(kc),
                il*n_batch*n_ctx*n_embd*ggml_element_size(kc)),
            n_embd/n_head, n_head, n_past + N, n_batch),
        0, 2, 1, 3);
// 确保新创建的张量 K 的形状为 4 维
assert_shape_4d(K, n_embd/n_head, n_past + N, n_head, n_batch);

// 执行 K 和 Q 的矩阵乘法
// 计算 K 和 Q 的乘积，得到形状为 [n_past + N, N, n_head, n_batch] 的张量 KQ
struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);
assert_shape_4d(KQ, n_past + N, N, n_head, n_batch);

// 对 KQ 进行缩放，使用公式 KQ_scaled = KQ / sqrt(n_embd/n_head)，得到形状为 [n_past + N, N, n_head, n_batch] 的张量 KQ_scaled
struct ggml_tensor * KQ_scaled =
    ggml_scale(ctx0,
            KQ,
            ggml_new_f32(ctx0, 1.0f/sqrtf(float(n_embd)/n_head)));
assert_shape_4d(KQ_scaled, n_past + N, N, n_head, n_batch);

// 对 KQ_scaled 进行掩码处理，使用函数 mask_past()，得到形状为 [n_past + N, N, n_head, n_batch] 的张量 KQ_masked
struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ_scaled, n_past);
assert_shape_4d(KQ_masked, n_past + N, N, n_head, n_batch);

// 对 KQ_masked 进行 softmax 处理，得到形状为 [n_past + N, N, n_head, n_batch] 的张量 KQ_soft_max
struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);
// 确保输入的 KQ_soft_max 张量的形状为 4 维，分别为 (n_past + N, N, n_head, n_batch)
assert_shape_4d(KQ_soft_max, n_past + N, N, n_head, n_batch);

// 将缓存的 V 张量分割成 n_head 个头部
// kv_self.v 的形状为 [n_ctx * n_embd * n_batch * n_layer]
// V 的形状为 [n_past + N, n_embd/n_head, n_head, n_batch]，等于 kv_self.v[:(n_past+N),:,:,il]
struct ggml_tensor * V = ggml_view_4d(ctx0, vc,
                    n_past + N, n_embd/n_head, n_head, n_batch,
                    ggml_element_size(vc)*n_ctx,
                    ggml_element_size(vc)*n_ctx*n_embd/n_head,
                    ggml_element_size(vc)*n_ctx*n_embd,
                    il*n_batch*n_ctx*n_embd*ggml_element_size(vc));
assert_shape_4d(V, n_past + N, n_embd/n_head, n_head, n_batch);

// 计算 KQV，形状为 [n_embd/n_head, N, n_head, n_batch]
struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);
assert_shape_4d(KQV, n_embd/n_head, N, n_head, n_batch);

// 对 KQV 进行维度置换，得到 KQV_merged，形状为 [n_embd/n_head, n_head, N, n_batch]
// 使用 ggml_permute 函数对 KQV 进行维度置换，得到 KQV_merged
struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);
// 确保 KQV_merged 的形状为 4 维
assert_shape_4d(KQV_merged, n_embd/n_head, n_head, N, n_batch);
// KQV_merged 的形状

// 对 KQV_merged 进行连续化并改变形状为 [n_embd, N*n_batch]
// cur 的形状为 [n_embd, N*n_batch]
cur = ggml_reshape_2d(ctx0, ggml_cont(ctx0, KQV_merged), n_embd, N*n_batch);
assert_shape_2d(cur, n_embd, N*n_batch);
// cur = ggml_cpy(ctx0,
//         KQV_merged,
//         ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, N));

// 投影（无偏置）
// cur 的形状为 [n_embd, N*n_batch]
cur = ggml_mul_mat(ctx0,
        model->layers[il].wo,
        cur);
assert_shape_2d(cur, n_embd, N*n_batch);
        // 使用缓冲区进行处理
        // lctx.use_buf(ctx0, 1);

        // 输入张量的形状为 [n_embd, N*n_batch, 1, 1]
        // 创建一个新的张量 inpFF，并将其添加到当前上下文中
        struct ggml_tensor * inpFF = ggml_add(ctx0, cur, inpSA);
        // 确保 inpFF 的形状为二维
        assert_shape_2d(inpFF, n_embd, N*n_batch);

        // 前馈网络
        {
            // 归一化
            {
                // 当前张量的形状为 [n_embd, N*n_batch, 1, 1]
                cur = ggml_rms_norm(ctx0, inpFF, rms_norm_eps);
                // 确保当前张量的形状为 [n_embd, N*n_batch]
                assert_shape_2d(cur, n_embd, N*n_batch);

                // cur = ffn_norm*cur
                // 当前张量的形状为 [n_embd, N*n_batch, 1, 1]
                cur = ggml_mul(ctx0,
                        ggml_repeat(ctx0, model->layers[il].ffn_norm, cur),
                        cur);
                // 确保当前张量的形状为 [n_embd, N*n_batch]
                assert_shape_2d(cur, n_embd, N*n_batch);
            }

            // 临时变量 tmp 的形状为 [n_ff, N*n_batch, 1, 1]
            struct ggml_tensor * tmp = ggml_mul_mat(ctx0,
                    model->layers[il].w3,
                    cur);
            // 断言 tmp 的形状为二维
            assert_shape_2d(tmp, n_ff, N*n_batch);

            // 当前变量 cur 的形状为 [n_ff, N*n_batch, 1, 1]
            cur = ggml_mul_mat(ctx0,
                    model->layers[il].w1,
                    cur);
            // 断言当前变量 cur 的形状为二维
            assert_shape_2d(cur, n_ff, N*n_batch);

            // 使用 SILU 激活函数
            // 当前变量 cur 的形状为 [n_ff, N*n_batch, 1, 1]
            cur = ggml_silu(ctx0, cur);
            // 断言当前变量 cur 的形状为二维
            assert_shape_2d(cur, n_ff, N*n_batch);

            // 当前变量 cur 的形状为 [n_ff, N*n_batch, 1, 1]
// 使用 ggml_mul 函数对 ctx0, cur, tmp 进行矩阵相乘操作，并将结果赋给 cur
cur = ggml_mul(ctx0, cur, tmp);
// 确保 cur 的形状为 [n_ff, N*n_batch]
assert_shape_2d(cur, n_ff, N*n_batch);

// 对 cur 进行矩阵相乘操作，使用 model->layers[il].w2，结果赋给 cur
// cur 的形状变为 [n_embd, N*n_batch]
cur = ggml_mul_mat(ctx0, model->layers[il].w2, cur);
// 确保 cur 的形状为 [n_embd, N*n_batch]

// 对 cur 和 inpFF 进行矩阵相加操作，并将结果赋给 cur
// 确保 cur 的形状为 [n_embd, N*n_batch]
cur = ggml_add(ctx0, cur, inpFF);
// 确保 cur 的形状为 [n_embd, N*n_batch]

// 将 cur 的值赋给 inpL，作为下一层的输入
// 确保 inpL 的形状为 [n_embd, N*n_batch]
inpL = cur;
// 确保 inpL 的形状为 [n_embd, N*n_batch]
// 对输入进行标准化处理
{
    // 将输入的形状调整为 [n_embd, N*n_batch, 1, 1]
    inpL = ggml_rms_norm(ctx0, inpL, rms_norm_eps);
    assert_shape_2d(inpL, n_embd, N*n_batch);

    // 将输入乘以标准化参数
    // 输入的形状为 [n_embd, N*n_batch, 1, 1]
    inpL = ggml_mul(ctx0,
                ggml_repeat(ctx0, model->norm, inpL),
                inpL);

    assert_shape_2d(inpL, n_embd, N*n_batch);

    // embeddings = inpL;
}

// lm_head
// 输入的形状为 [n_vocab, N*n_batch, 1, 1]
    # 使用 ggml_mul_mat 函数对 ctx0, model->output, inpL 进行矩阵乘法运算，并将结果赋值给 inpL
    inpL = ggml_mul_mat(ctx0, model->output, inpL);
    # 断言 inpL 的形状为二维数组，维度分别为 n_vocab, N*n_batch
    assert_shape_2d(inpL, n_vocab, N*n_batch);

    {
        # 对 inpL 进行形状重塑，使其形状变为 [n_vocab,N,n_batch,1]
        inpL = ggml_reshape_3d(ctx0,
                        inpL,
                        n_vocab, N, n_batch);
        # 断言 inpL 的形状为三维数组，维度分别为 n_vocab, N, n_batch
        assert_shape_3d(inpL, n_vocab, N, n_batch);
    }

    # 运行计算
    ggml_build_forward_expand(gf, inpL);

    # 返回 inpL
    return inpL;
}

static struct ggml_tensor * forward_lora(
    struct llama_model_lora * model,
    struct llama_kv_cache   * cache,
```
// 定义函数，接受指向结构体的指针和其他参数
void some_function(
    struct ggml_context     * ctx0,  // 指向 ggml_context 结构体的指针
    struct ggml_cgraph      * gf,    // 指向 ggml_cgraph 结构体的指针
    struct ggml_tensor      * tokens_input,  // 指向 ggml_tensor 结构体的指针
    const  int                n_tokens,      // 整型常量，表示 tokens 的数量
    const  int                n_past         // 整型常量，表示过去的数量
) {
    const int N = n_tokens;  // 定义整型常量 N，赋值为 n_tokens

    struct llama_kv_cache& kv_self = *cache;  // 定义指向 llama_kv_cache 结构体的引用 kv_self，指向 cache 指针所指向的对象
    const auto & hparams = model->hparams;    // 定义常量引用 hparams，指向 model 结构体的 hparams 成员

    const int n_ctx   = hparams.n_ctx;   // 定义整型常量 n_ctx，赋值为 hparams 结构体的 n_ctx 成员
    const int n_embd  = hparams.n_embd;  // 定义整型常量 n_embd，赋值为 hparams 结构体的 n_embd 成员
    const int n_layer = hparams.n_layer;  // 定义整型常量 n_layer，赋值为 hparams 结构体的 n_layer 成员
    const int n_head  = hparams.n_head;   // 定义整型常量 n_head，赋值为 hparams 结构体的 n_head 成员
    const int n_rot   = hparams.n_rot;    // 定义整型常量 n_rot，赋值为 hparams 结构体的 n_rot 成员

    struct ggml_tensor * tokens = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);  // 创建指向 ggml_tensor 结构体的指针 tokens，调用 ggml_new_tensor_1d 函数
    memcpy(tokens->data, tokens_input->data, N*ggml_element_size(tokens));  // 将 tokens_input 的数据复制到 tokens 的数据中
}
    # 将 kv_self.k 赋值给指针 kc
    struct ggml_tensor * kc = kv_self.k;
    # 将 kv_self.v 赋值给指针 vc
    struct ggml_tensor * vc = kv_self.v;

    # 创建一个一维的 ggml_tensor 结构体 KQ_pos，数据类型为 GGML_TYPE_I32，长度为 N
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    {
        # 将 KQ_pos 的数据指针转换为 int 类型的指针
        int * data = (int *) KQ_pos->data;
        # 遍历 KQ_pos 的数据，赋值为 n_past + i
        for (int i = 0; i < N; ++i) {
            data[i] = n_past + i;
        }
    }

    # 从 model->tok_embeddings 中获取 tokens 对应的行，赋值给 inpL
    # inpL 的形状为 [n_embd,N,1,1]
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model->tok_embeddings, tokens);
    # 遍历 n_layer 次
    for (int il = 0; il < n_layer; ++il) {
        # 将 inpL 赋值给 inpSA
        struct ggml_tensor * inpSA = inpL;

        # 声明一个指针 cur

        # 进行归一化操作
        {
            // 计算当前形状 [n_embd,N,1,1] 的值
            cur = ggml_rms_norm(ctx0, inpL, rms_norm_eps);

            // 计算 attention_norm 与 cur 的乘积，并赋值给 cur
            cur = ggml_mul(ctx0,
                        ggml_repeat(ctx0, model->layers[il].attention_norm, cur),
                        cur);
        }

        // 自注意力
        {
            // 计算 Q 和 K 并对它们进行 RoPE
            // wq   形状 [n_embd, n_embd, 1, 1]
            // wk   形状 [n_embd, n_embd, 1, 1]
            // Qcur 形状 [n_embd/n_head, n_head, N, 1]
            // Kcur 形状 [n_embd/n_head, n_head, N, 1]
            struct ggml_tensor * Qcur = ggml_rope(ctx0,
                                            ggml_reshape_3d(ctx0,
                                                ggml_mul_mat(ctx0,
                                                    model->layers[il].wqa,
```
在这里，我只能提供部分注释，因为代码片段不完整。如果您能提供完整的代码片段，我可以为您提供更全面的注释。
// 使用 ggml_mul_mat 函数计算 ctx0, model->layers[il].wqb, cur 三者的乘积
// 参数分别为 ctx0, model->layers[il].wqb, cur
// 返回结果存储在 ggml_mul_mat(ctx0, model->layers[il].wqb, cur) 中
ggml_mul_mat(ctx0, model->layers[il].wqb, cur),

// 使用 ggml_reshape_3d 函数对 ctx0, model->layers[il].wka, ggml_mul_mat(ctx0, model->layers[il].wkb, cur) 进行重塑
// 参数分别为 ctx0, model->layers[il].wka, ggml_mul_mat(ctx0, model->layers[il].wkb, cur)
// 返回结果存储在 ggml_reshape_3d(ctx0, ggml_mul_mat(ctx0, model->layers[il].wka, ggml_mul_mat(ctx0, model->layers[il].wkb, cur)), n_embd/n_head, n_head, N) 中

// 使用 ggml_rope 函数对 ctx0, ggml_reshape_3d(ctx0, ggml_mul_mat(ctx0, model->layers[il].wka, ggml_mul_mat(ctx0, model->layers[il].wkb, cur)), n_embd/n_head, n_head, N), KQ_pos, n_rot, 0, 0 进行操作
// 参数分别为 ctx0, ggml_reshape_3d(ctx0, ggml_mul_mat(ctx0, model->layers[il].wka, ggml_mul_mat(ctx0, model->layers[il].wkb, cur)), n_embd/n_head, n_head, N), KQ_pos, n_rot, 0, 0
// 返回结果存储在 ggml_rope(ctx0, ggml_reshape_3d(ctx0, ggml_mul_mat(ctx0, model->layers[il].wka, ggml_mul_mat(ctx0, model->layers[il].wkb, cur)), n_embd/n_head, n_head, N), KQ_pos, n_rot, 0, 0) 中

// 定义并初始化 Kcur 结构体指针，使用 ggml_rope 函数的返回值
struct ggml_tensor * Kcur = ggml_rope(ctx0, ggml_reshape_3d(ctx0, ggml_mul_mat(ctx0, model->layers[il].wka, ggml_mul_mat(ctx0, model->layers[il].wkb, cur)), n_embd/n_head, n_head, N), KQ_pos, n_rot, 0, 0);

// 存储键和值到内存
{
    // 计算转置的 [N, n_embd] V 矩阵
    // wv 的形状为 [n_embd, n_embd, 1, 1]
    // Vcur 的形状为 [n_embd, N, 1, 1]
}
// 创建一个指向 ggml_tensor 结构体的指针 Vcur，该结构体包含了经过一系列计算后的张量数据
struct ggml_tensor * Vcur = ggml_cont(ctx0,
                                    ggml_transpose(ctx0,
                                        ggml_reshape_2d(ctx0,
                                            ggml_mul_mat(ctx0,
                                                model->layers[il].wva,
                                                ggml_mul_mat(ctx0,
                                                    model->layers[il].wvb,
                                                    cur)),
                                            n_embd, N)));

// 以下是对不同张量的形状进行注释说明
// kv_self.k 的形状为 [n_embd * n_ctx * n_layer, 1]
// kv_self.v 的形状为 [n_embd * n_ctx * n_layer, 1]
// k 的形状为 [n_embd * N, 1]，等于 kv_self.k[:,n_past:n_past+N,il,0]
// v 的形状为 [N, n_embd, 1, 1]，等于 kv_self.v[:,n_past:n_past+N,il,0]

/* {
    // 创建一个指向 ggml_tensor 结构体的指针 k，指向 kv_self.k 中的特定数据
    struct ggml_tensor * k = ggml_view_1d(ctx0, kv_self.k, N*n_embd, (ggml_element_size(kv_self.k)*n_embd)*(il*n_ctx + n_past));
    // 创建一个指向 ggml_tensor 结构体的指针 v，指向 kv_self.v 中的特定数据
    struct ggml_tensor * v = ggml_view_2d(ctx0, kv_self.v, N, n_embd,
            (   n_ctx)*ggml_element_size(kv_self.v),
            (il*n_ctx)*ggml_element_size(kv_self.v)*n_embd + n_past*ggml_element_size(kv_self.v));
// 在 KV 缓存中存储 K 的 RoPE-ed 版本，这是一个重要步骤
ggml_build_forward_expand(gf, ggml_cpy(ctx0, Kcur, k));
ggml_build_forward_expand(gf, ggml_cpy(ctx0, Vcur, v));

// 将 Kcur 重塑为 n_embd*N 的形状，并存储在 kc 中
kc = ggml_set_1d(ctx0, kc, ggml_reshape_1d(ctx0, Kcur, n_embd*N), (ggml_element_size(kv_self.k)*n_embd)*(il*n_ctx + n_past));

// 将 Vcur 重塑为 (n_ctx)*ggml_element_size(kv_self.v) 的形状，并存储在 vc 中
vc = ggml_set_2d(ctx0, vc, Vcur, (n_ctx)*ggml_element_size(kv_self.v), (il*n_ctx)*ggml_element_size(kv_self.v)*n_embd + n_past*ggml_element_size(kv_self.v));

// 对 Qcur 进行维度置换，得到 Q
struct ggml_tensor * Q = ggml_permute(ctx0, Qcur, 0, 2, 1, 3);

// 对 kv_self.k 进行维度置换，得到 K
// 创建一个新的张量 K，通过对输入张量进行一系列操作得到
struct ggml_tensor * K =
    ggml_permute(ctx0,
        ggml_reshape_3d(ctx0,
            ggml_view_1d(ctx0, kc, (n_past + N)*n_embd, il*n_ctx*ggml_element_size(kc)*n_embd),
            n_embd/n_head, n_head, n_past + N),
        0, 2, 1, 3);

// 计算 K 和 Q 的矩阵乘法
// 得到的 KQ 张量的形状为 [n_past + N, N, n_head, 1]
struct ggml_tensor * KQ = ggml_mul_mat(ctx0, K, Q);

// 对 KQ 进行缩放，除以 sqrt(n_embd/n_head)
// 得到的 KQ_scaled 张量的形状为 [n_past + N, N, n_head, 1]
struct ggml_tensor * KQ_scaled =
    ggml_scale(ctx0,
        KQ,
        ggml_new_f32(ctx0, 1.0f/sqrtf(float(n_embd)/n_head)));

// 对 KQ_scaled 进行掩码操作，mask_past 函数的具体实现未给出
// 得到的 KQ_masked 张量的形状为 [n_past + N, N, n_head, 1]
// 使用 ggml_diag_mask_inf 函数对 KQ_scaled 进行掩码处理，得到 KQ_masked
// KQ_masked 的形状为 [n_past, N, n_head, 1]
struct ggml_tensor * KQ_masked = ggml_diag_mask_inf(ctx0, KQ_scaled, n_past);

// 使用 ggml_soft_max 函数对 KQ_masked 进行 soft_max 处理，得到 KQ_soft_max
// KQ_soft_max 的形状为 [n_past + N, N, n_head, 1]
struct ggml_tensor * KQ_soft_max = ggml_soft_max(ctx0, KQ_masked);

// 使用 ggml_view_3d 函数将 cached V 分割成 n_head 个头部
// V 的形状为 [n_past + N, n_embd/n_head, n_head, 1]
struct ggml_tensor * V =
    ggml_view_3d(ctx0, vc,
            n_past + N, n_embd/n_head, n_head,
            n_ctx*ggml_element_size(vc),
            n_ctx*ggml_element_size(vc)*n_embd/n_head,
            il*n_ctx*ggml_element_size(vc)*n_embd);

// 使用 ggml_mul_mat 函数对 V 和 KQ_soft_max 进行矩阵相乘，得到 KQV
// KQV 的形状为 [n_embd/n_head, N, n_head, 1]
struct ggml_tensor * KQV = ggml_mul_mat(ctx0, V, KQ_soft_max);

// 对 KQV 进行维度置换，得到 KQV_merged
// 创建一个新的张量 KQV_merged，通过对 KQV 进行维度重排，维度顺序为 [n_embd/n_head, n_head, N, 1]
struct ggml_tensor * KQV_merged = ggml_permute(ctx0, KQV, 0, 2, 1, 3);

// 通过对 KQV_merged 进行连续化和形状重塑，得到张量 cur，形状为 [n_embd, N]
cur = ggml_reshape_2d(ctx0, ggml_cont(ctx0, KQV_merged), n_embd, N);

// 执行投影操作（无偏置），更新 cur 的形状为 [n_embd, N]
cur = ggml_mul_mat(ctx0,
        model->layers[il].woa,
        ggml_mul_mat(ctx0,
            model->layers[il].wob,
            cur));
        // 创建一个名为inpFF的指针，指向一个ggml_tensor结构体，表示输入的前馈神经网络
        struct ggml_tensor * inpFF = ggml_add(ctx0, cur, inpSA);

        // 前馈网络
        {
            // 归一化
            {
                // 将cur的形状设为[n_embd,N,1,1]
                cur = ggml_rms_norm(ctx0, inpFF, rms_norm_eps);

                // 将cur的值更新为ffn_norm乘以cur
                // cur的形状为[n_embd,N,1,1]
                cur = ggml_mul(ctx0,
                        ggml_repeat(ctx0, model->layers[il].ffn_norm, cur),
                        cur);
            }

            // 创建一个名为tmp的指针，指向一个ggml_tensor结构体，表示临时变量
            // tmp的形状为[n_ff,N,1,1]
            struct ggml_tensor * tmp = ggml_mul_mat(ctx0,
                    model->layers[il].w3,
// 对当前数据进行矩阵乘法操作，得到新的数据
cur = ggml_mul_mat(ctx0,
        model->layers[il].w1,
        cur);

// 对当前数据进行SILU激活函数处理
cur = ggml_silu(ctx0, cur);

// 对当前数据进行乘法操作，得到新的数据
cur = ggml_mul(ctx0, cur, tmp);

// 对当前数据进行矩阵乘法操作，得到新的数据
cur = ggml_mul_mat(ctx0,
        model->layers[il].w2,
        cur);
// 对当前形状进行加法操作，inpFF为输入
cur = ggml_add(ctx0, cur, inpFF);

// 为下一层准备输入
// inpL形状为[n_embd,N,1,1]
inpL = cur;
}

// 归一化
{
    // inpL形状为[n_embd,N,1,1]
    inpL = ggml_rms_norm(ctx0, inpL, rms_norm_eps);

    // inpL = norm*inpL
    // inpL形状为[n_embd,N,1,1]
    inpL = ggml_mul(ctx0,
                ggml_repeat(ctx0, model->norm, inpL),
                inpL);
}
// embeddings = inpL;  // 这行代码被注释掉了，可能是之前的代码或者注释掉的临时代码

// lm_head  // 这行注释标明了下面代码的作用是对 lm_head 进行操作
// inpL shape [n_vocab,N,1,1]  // 这行注释说明了 inpL 的形状

// 对输入 inpL 进行一系列矩阵乘法操作
inpL = ggml_mul_mat(ctx0,
            model->outputa,
                ggml_mul_mat(ctx0,
                    model->outputb,
                    inpL));

// ggml_set_scratch(ctx0, { 0, 0, nullptr, });  // 这行代码被注释掉了，可能是之前的代码或者注释掉的临时代码
// 运行计算
ggml_build_forward_expand(gf, inpL);

// 返回处理后的输入 inpL
return inpL;
}

// 对 logits 进行 softmax 操作，得到概率分布 probs 和最佳样本 best_samples
static void sample_softmax(struct ggml_tensor * logits, struct ggml_tensor * probs, struct ggml_tensor * best_samples) {
    # 确保logits的维度为2
    assert(logits->n_dims == 2);
    # 确保probs的维度为2
    assert(probs->n_dims == 2);
    # 确保best_samples的维度为1
    assert(best_samples->n_dims == 1);
    # 确保logits的第二维与best_samples的第一维相等
    assert(logits->ne[1] == best_samples->ne[0]);
    # 确保logits的第一维与probs的第一维相等
    assert(logits->ne[0] == probs->ne[0]);
    # 确保logits的第二维与probs的第二维相等
    assert(logits->ne[1] == probs->ne[1]);
    # 遍历logits的第二维
    for (int i = 0; i < logits->ne[1]; ++i) {
        # 获取当前行最大的logit值
        float max_logit = ggml_get_f32_1d(logits, i * logits->ne[0]);
        # 初始化最大logit对应的样本索引为0
        ggml_set_i32_1d(best_samples, i, 0);
        # 遍历当前行的每个logit值
        for (int k = 0; k < logits->ne[0]; ++k) {
            # 获取当前logit值
            float logit = ggml_get_f32_1d(logits, i * logits->ne[0] + k);
            # 如果当前logit值大于最大logit值，则更新最大logit值和对应的样本索引
            if (logit > max_logit) {
                max_logit = logit;
                ggml_set_i32_1d(best_samples, i, k);
            }
        }
        # 计算softmax函数的分母
        float psum = 0;
        # 遍历当前行的每个logit值
        for (int k = 0; k < logits->ne[0]; ++k) {
            # 获取当前logit值
            float logit = ggml_get_f32_1d(logits, i * logits->ne[0] + k);
            # 计算softmax函数的分子
            float p = (logit == -INFINITY) ? 0 : expf(logit - max_logit);
        // 将概率值累加到psum中
        psum += p;
        // 将概率值存储到probs张量中的指定位置
        ggml_set_f32_1d(probs, i * probs->ne[0] + k, p);
    }
    // 对每个logits张量中的元素进行softmax操作
    for (int k = 0; k < logits->ne[0]; ++k) {
        // 获取probs张量中的概率值
        float p = ggml_get_f32_1d(probs, i*probs->ne[0] + k);
        // 将概率值除以psum，得到归一化后的概率值
        ggml_set_f32_1d(probs, i * probs->ne[0] + k, p / psum);
    }
}

// 对logits张量进行softmax采样
static void sample_softmax_batch(
    struct ggml_context * ctx, struct ggml_tensor * logits, struct ggml_tensor * probs,
    struct ggml_tensor * best_samples
) {
    // 断言best_samples张量的维度为2
    GGML_ASSERT(best_samples->n_dims == 2);
    // 断言logits张量的维度为3
    GGML_ASSERT(logits->n_dims == 3);
    // 断言probs张量的维度为3
    GGML_ASSERT(probs->n_dims == 3);
    // 获取best_samples张量的第一维大小
    int n_tokens = best_samples->ne[0];
    // 获取best_samples张量的第二维大小
    int n_batch  = best_samples->ne[1];
    // 获取logits张量的第一维大小，即词汇表大小
    int n_vocab  = logits->ne[0];
    # 确保 tokens 数量与 logits 的第二维度相等
    GGML_ASSERT(n_tokens == logits->ne[1]);
    # 确保 batch 数量与 logits 的第三维度相等
    GGML_ASSERT(n_batch  == logits->ne[2]);
    # 确保 vocab 数量与 probs 的第一维度相等
    GGML_ASSERT(n_vocab  == probs->ne[0]);
    # 确保 tokens 数量与 probs 的第二维度相等
    GGML_ASSERT(n_tokens == probs->ne[1]);
    # 确保 batch 数量与 probs 的第三维度相等
    GGML_ASSERT(n_batch  == probs->ne[2]);

    # 遍历 batch 数量
    for (int k = 0; k < n_batch; ++k) {
        # 从 best_samples 中获取第 k 个样本
        struct ggml_tensor * best_samples_k = ggml_view_1d(ctx,
                                                best_samples,
                                                best_samples->ne[0],
                                                k*best_samples->nb[1]);
        # 从 logits 中获取第 k 个 batch 的数据
        struct ggml_tensor * logits_k       = ggml_view_2d(ctx,
                                                logits,
                                                logits->ne[0],
                                                logits->ne[1],
                                                logits->nb[1],
                                                k*logits->nb[2]);
        # 从 probs 中获取第 k 个 batch 的数据
        struct ggml_tensor * probs_k        = ggml_view_2d(ctx,
                                                probs,
                                                probs->ne[0],
// 打印给定概率张量中的第 i 行数据
static void print_row(struct ggml_tensor * probs, int i) {
    for (int k = 0; k < probs->ne[0]; ++k) {
        // 获取概率张量中第 i 行第 k 列的概率值
        float p = ggml_get_f32_1d(probs, i*probs->ne[0] + k);
        // 打印概率值
        printf(" %.2f", p);
    }
    // 换行
    printf("\n");
}

// 打印给定概率张量中的所有数据
static void print_matrix(struct ggml_tensor * probs) {
    // 确保概率张量是二维的
    assert(probs->n_dims == 2);
    for (int i = 0; i < probs->ne[1]; ++i) {
        for (int k = 0; k < probs->ne[0]; ++k) {
            // 获取概率张量中第 i 行第 k 列的概率值
            float p = ggml_get_f32_1d(probs, i*probs->ne[0] + k);
// 打印一个浮点数，保留两位小数
printf(" %.2f", p);
// 打印换行符
printf("\n");
// 打印 token 个空格
for (int k = 0; k < token; ++k) {
    printf(" ");
}
// 打印字符 'X'
printf("X");
// 打印 n_vocab - token - 1 个空格
for (int k = token+1; k < n_vocab; ++k) {
    printf(" ");
}
// 打印换行符
printf("\n");
// 遍历 tokens 结构体中的第一个维度
for (int i=0; i<tokens->ne[0]; ++i) {
    // 获取 tokens 结构体中第 i 个元素的值
    int token = ggml_get_i32_1d(tokens, i);
// 打印 token 和 n_vocab 的值
print_token(token, n_vocab);
}

// 获取示例的目标值
static void get_example_targets(int example_id, struct ggml_tensor * tokens_input, struct ggml_tensor * targets) {
    // 获取 tokens_input 和 targets 的长度
    int n_tokens = tokens_input->ne[0];
    int n_vocab = targets->ne[0];
    float randomness = 0.0f;
    // 将 targets 的值全部设为 -1.0f
    ggml_set_f32(targets, -1.0f);
    // 将 tokens_input 的第一个值设为 0
    ggml_set_i32_1d(tokens_input, 0, 0);
    // 遍历 tokens_input
    for (int i=1; i<n_tokens+1; ++i) {
        // 计算 x, y, z 的值
        float x = example_id + i * 3.14159f * 2.0f * 1.0f * 0.5f / n_tokens;
        float y = sinf(x);
        float z = (y+1.0f)*0.5f; // 缩放到 [0..1] 范围内
        z += (frand()-0.5f)*(randomness/n_vocab); // 添加一定的随机性
        z = (z < 0.0f) ? 0.0f : (z > 1.0f) ? 1.0f : z; // 将 z 值限制在 [0..1] 范围内
        // 计算 token 的值
        int token = std::max(1,std::min(1+(int)(z*(float)(n_vocab-1)), n_vocab-1));
        // 将 targets 中对应位置的值设为 +1.0f
        ggml_set_f32_1d(targets, (i-1)*n_vocab + token, +1.0f);
        if (i<n_tokens) {
// 将 token 输入设置为一维数组中的值
ggml_set_i32_1d(tokens_input, i, token);
// 获取示例目标的批处理数据
static void get_example_targets_batch(
    struct ggml_context * ctx, int example_id, struct ggml_tensor * tokens_input, struct ggml_tensor * targets
) {
    // 断言 token 输入的维度为2
    GGML_ASSERT(tokens_input->n_dims == 2);
    // 断言目标的维度为3
    GGML_ASSERT(targets->n_dims == 3);
    // 获取 token 的数量
    int n_tokens = tokens_input->ne[0];
    // 获取批处理的数量
    int n_batch  = tokens_input->ne[1];
    // 断言 token 的数量与目标的第二维度相等
    GGML_ASSERT(n_tokens == targets->ne[1]);
    // 断言批处理的数量与目标的第三维度相等
    GGML_ASSERT(n_batch  == targets->ne[2]);

    // 遍历批处理的数量
    for (int k=0; k<n_batch; ++k) {
        // 获取 tokens_input_k 的视图
        struct ggml_tensor * tokens_input_k = ggml_view_1d(ctx,
                                                tokens_input,
                                                tokens_input->ne[0],
                                                k*tokens_input->nb[1]);
// 创建一个指向 targets 的二维视图，从第 k*targets->nb[2] 列开始，共 targets->ne[0] 行，targets->ne[1] 列
struct ggml_tensor * targets_k    = ggml_view_2d(ctx,
                                            targets,
                                            targets->ne[0],
                                            targets->ne[1],
                                            targets->nb[1],
                                            k*targets->nb[2]);
// 获取 example_id*n_batch + k 对应的样本的输入 tokens 和目标 targets
get_example_targets(example_id*n_batch + k, tokens_input_k, targets_k);
}

// 将 tokens_input 和 targets 中的数据向左移动 n_shift 个位置
static void lshift_examples(struct ggml_tensor * tokens_input, struct ggml_tensor * targets, int n_shift) {
    // 获取 tokens_input 的行数
    int n_tokens = tokens_input->ne[0];
    // 获取 targets 的行数
    int n_vocab = targets->ne[0];
    // 遍历 tokens_input 中的数据，将每个元素向左移动 n_shift 个位置
    for (int i=0; i<n_tokens-n_shift; ++i) {
        ggml_set_i32_1d(tokens_input, i, ggml_get_i32_1d(tokens_input, i + n_shift));
        // 遍历 targets 中的数据，将每个元素向左移动 n_shift 个位置
        for (int k=0; k<n_vocab; ++k) {
            ggml_set_f32_1d(targets, i*n_vocab + k, ggml_get_f32_1d(targets, (i + n_shift)*n_vocab + k));
        }
    }
}
// 计算平方误差损失函数
static struct ggml_tensor * square_error_loss(
    struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b
) {
    // todo: instead of a-b: a[1:]-b[:-1]  // 待办事项：不是a-b，而是a[1:]-b[:-1]
    return ggml_sum(ctx, ggml_sqr(ctx, ggml_sub(ctx, a, b)));  // 返回a和b之间的平方误差
}

// 计算交叉熵损失函数
static struct ggml_tensor * cross_entropy_loss(
    struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b
) {
    const float eps = 1e-3f;  // 定义一个小的浮点数eps
    return
        ggml_sum(ctx,  // 对计算结果进行求和
            ggml_neg(ctx,  // 对计算结果取负
                ggml_sum_rows(ctx,  // 对计算结果的行进行求和
                    ggml_mul(ctx,  // 对两个张量进行逐元素相乘
                        ggml_soft_max(ctx, a),  // 对a进行softmax操作
                        ggml_log(ctx,  // 对计算结果取对数
                            ggml_add1(ctx,  // 对计算结果加1
// 调用 ggml_soft_max 函数，传入 ctx 和 b 作为参数，返回结果
ggml_soft_max(ctx, b),
// 调用 ggml_new_f32 函数，传入 ctx 和 eps 作为参数，返回结果
ggml_new_f32(ctx, eps)))))));
}

// 主函数
int main(int argc, char ** argv) {
    // 如果参数个数小于1，输出使用方法并返回错误码
    if (argc < 1) {
        fprintf(stderr, "usage: %s\n", argv[0]);
        return 1;
    }

    // 初始化参数结构体 lcparams
    struct ggml_init_params lcparams;
    // 设置内存大小为1GB
    lcparams.mem_size   = 1024ll*1024ll*1024ll;
    // 内存缓冲区为空
    lcparams.mem_buffer = NULL;
    // 不禁用分配内存
    lcparams.no_alloc   = false;

    // 创建 llama_model 结构体 model
    struct llama_model model;
    // 设置模型参数：词汇表大小为8
    model.hparams.n_vocab = 8;
    // 设置模型参数：上下文大小为8
    model.hparams.n_ctx   = 8;
    // 设置模型参数：嵌入维度为32
    model.hparams.n_embd  = 32;
    // 设置模型超参数中的 n_mult 为 2
    model.hparams.n_mult  = 2;
    // 设置模型超参数中的 n_head 为 8
    model.hparams.n_head  = 8;
    // 设置模型超参数中的 n_layer 为 1
    model.hparams.n_layer = 1;
    // 设置模型超参数中的 n_rot 为 model.hparams.n_embd 除以 model.hparams.n_head 和 16 中的较小值
    model.hparams.n_rot   = std::min(16u, model.hparams.n_embd / model.hparams.n_head);

    // 初始化模型上下文
    model.ctx = ggml_init(lcparams);
    // 打印初始化模型的消息
    printf("init model\n");
    // 初始化模型
    init_model(&model);
    // 设置模型参数
    set_param_model(&model);
    // 随机初始化模型
    randomize_model(&model, 1337, 0.0f, 1.0f, -1.0f, +1.0f);

/*
    struct llama_model_lora model_lora;
*/
```
在这段代码中，主要是对模型的超参数进行设置和初始化，以及对模型进行随机初始化。同时，还有一段被注释掉的代码，可能是之前的实现或者备用代码。
    // 设置模型的词汇量为16
    model_lora.hparams.n_vocab = 16;
    // 设置模型的上下文长度为32
    model_lora.hparams.n_ctx   = 32;
    // 设置模型的嵌入维度为256
    model_lora.hparams.n_embd  = 256;
    // 设置模型的倍增因子为2
    model_lora.hparams.n_mult  = 2;
    // 设置模型的头数为16
    model_lora.hparams.n_head  = 16;
    // 设置模型的层数为1
    model_lora.hparams.n_layer = 1;
    // 设置模型的LORA参数为64
    model_lora.hparams.n_lora  = 64;
    // 计算并设置模型的旋转数为16和嵌入维度除以头数的最小值
    model_lora.hparams.n_rot   = MIN(16, model_lora.hparams.n_embd / model_lora.hparams.n_head);
    // 设置模型的嵌入维度为32
    // model.hparams.n_embd  = 32;
    // 设置模型的倍增因子为2
    // model.hparams.n_mult  = 2;
    // 设置模型的头数为4
    // 设置模型的层数为8
    // 设置模型的旋转数为8

    // 初始化模型的上下文
    model_lora.ctx = ggml_init(lcparams);
    // 打印初始化模型的消息
    printf("init model_lora\n");
    // 初始化模型
    init_model_lora(&model_lora);
    // 设置模型参数
    set_param_model_lora(&model_lora);
    // 为模型随机初始化参数
    randomize_model_lora(&model_lora, 1337, 0.0f, 1.0f, -1.0f, +1.0f);

    // 设置批处理大小为8
    int n_batch = 8;
    // 为自注意力机制设置键值缓存
    struct llama_kv_cache kv_self;
    // 打印初始化键值缓存的消息
    printf("init_kv_cache\n");
    // 设置键值缓存的上下文
    kv_self.ctx = model.ctx;
    // 初始化键值缓存
    init_kv_cache(&kv_self, &model, n_batch);

    // 设置计算大小为1GB
    size_t    compute_size = 1024ll*1024ll*1024ll;
    // 为计算地址分配内存空间
    uint8_t * compute_addr = new uint8_t[compute_size];

    // 设置示例数量、token数量和词汇表大小
    int n_examples = 256;
    int n_tokens = model.hparams.n_ctx;
    int n_vocab  = model.hparams.n_vocab;

    // 创建一个存储uint8_t类型数据的动态数组
    std::vector<uint8_t> work_buffer;

    // 遍历示例数量
    for (int ex=0; ex<n_examples; ++ex) {
        // 初始化参数结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ compute_size,  // 内存大小
            /*.mem_buffer =*/ compute_addr,  // 内存地址
            /*.no_alloc   =*/ false,         // 是否分配内存
        };

        // 初始化上下文
        struct ggml_context * ctx0 = ggml_init(params);

        // 创建新的2D张量
        struct ggml_tensor * after_opt_best_samples  = ggml_new_tensor_2d(ctx0, GGML_TYPE_I32, n_tokens, n_batch);
        struct ggml_tensor * after_opt_probs         = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_vocab, n_tokens, n_batch);
        struct ggml_tensor * tokens_input            = ggml_new_tensor_2d(ctx0, GGML_TYPE_I32, n_tokens, n_batch);
    }
// 创建一个三维张量 targets，数据类型为 GGML_TYPE_F32，大小为 n_vocab * n_tokens * n_batch
struct ggml_tensor * targets = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_vocab, n_tokens, n_batch);

// 初始化变量 n_past 为 0
int n_past = 0;

// 创建一个计算图对象 gf
ggml_cgraph gf = {};

// 调用函数获取样本的目标值，并存储在 targets 中
get_example_targets_batch(ctx0, 64*ex+0, tokens_input, targets);

// 调用函数进行前向传播，得到预测值 logits
struct ggml_tensor * logits = forward_batch(&model, &kv_self, ctx0, &gf, tokens_input, n_tokens, n_past, n_batch);

// 使用 square error loss 函数计算损失值 e
struct ggml_tensor * e = square_error_loss(ctx0, targets, logits);

// 构建前向传播计算图
ggml_build_forward_expand(&gf, e);

// 使用计算图对象 gf 进行计算
ggml_graph_compute_helper(work_buffer, &gf, /*n_threads*/ 1);

// 获取优化前的损失值
float error_before_opt = ggml_get_f32_1d(e, 0);

// 初始化优化参数 opt_params_lbfgs 为 LBFGS 算法的默认参数
struct ggml_opt_params opt_params_lbfgs = ggml_opt_default_params(GGML_OPT_LBFGS);
opt_params_lbfgs.print_forward_graph = false;
opt_params_lbfgs.print_backward_graph = false;
        // 设置 LBFGS 优化算法的迭代次数为16
        opt_params_lbfgs.lbfgs.n_iter = 16;
        // 使用 LBFGS 优化算法对 ctx0 进行优化，结果保存在 e 中
        ggml_opt(ctx0, opt_params_lbfgs, e);
        // 构建前向传播并扩展
        ggml_build_forward_expand(&gf, e);
        // 计算图的辅助信息，使用一个线程
        ggml_graph_compute_helper(work_buffer, &gf, /*n_threads*/ 1);

        // 获取优化后的误差值
        float error_after_opt = ggml_get_f32_1d(e, 0);

        // 每8个样本打印一次优化前后的误差值
        if (ex % 8 == 0) {
            printf("Example %d\n", (ex+1));
            printf("error_before_opt: %.2f\n", error_before_opt);
            printf("error_after_opt:  %.2f\n", error_after_opt);
        }

        // 每64个样本进行一次 softmax 批处理，并打印最佳样本
        if (ex % 64 == 0) {
            sample_softmax_batch(ctx0, logits, after_opt_probs, after_opt_best_samples);
            // 打印优化后的概率
            // printf("probabilities after optimization:\n");
            // print_matrix(after_opt_probs);
            printf("best samples after optimization:\n");
            print_tokens(after_opt_best_samples, n_vocab);
    }

    ggml_free(ctx0);
    // 释放上下文资源

}

{
    int n_gen = 128;
    // 生成的 token 数量
    int sample_ctx = n_tokens-n_tokens/8;
    // 样本上下文的大小

    printf("Generating %d tokens.\n", n_gen);
    // 打印生成 token 的数量

    struct ggml_tensor * tokens_input = ggml_new_tensor_1d(model.ctx, GGML_TYPE_I32, n_tokens);
    // 创建一个一维张量用于存储 token
    struct ggml_tensor * targets      = ggml_new_tensor_2d(model.ctx, GGML_TYPE_F32, n_vocab, n_tokens);
    // 创建一个二维张量用于存储目标

    get_example_targets(137, tokens_input, targets);
    // 获取示例目标

    for (int i=sample_ctx; i<n_tokens; ++i) {
        ggml_set_i32_1d(tokens_input, i, n_vocab/2);
        // 设置 tokens_input 中的值
    }

    for (int i=0; i<sample_ctx-1; ++i) {
        // 循环遍历样本上下文
        // 打印 tokens_input 中第 i 个元素对应的词汇，使用 n_vocab 进行解析
        print_token(ggml_get_i32_1d(tokens_input, i), n_vocab);
    }
    // 打印分隔线
    printf("---\n");
    // 遍历 n_gen 次
    for (int i=0; i<n_gen; ++i) {
        // 初始化参数结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ compute_size,  // 内存大小
            /*.mem_buffer =*/ compute_addr,  // 内存缓冲区地址
            /*.no_alloc   =*/ false,         // 是否允许分配内存
        };
        // 初始化上下文
        struct ggml_context * ctx0 = ggml_init(params);

        // 创建计算图
        ggml_cgraph gf = {};

        // 过去的数量
        int n_past = 0;
        // 计算前向传播得到 logits
        struct ggml_tensor * logits = forward(&model, &kv_self, ctx0, &gf, tokens_input, sample_ctx, n_past);

        // 构建前向传播扩展
        ggml_build_forward_expand(&gf, logits);
        // 计算图的计算辅助函数
        ggml_graph_compute_helper(work_buffer, &gf, /*n_threads*/ 1);

        // 创建新的一维张量 best_samples
        struct ggml_tensor * best_samples = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, sample_ctx);
// 创建一个二维的浮点型张量，用于存储概率值，大小为 n_vocab * sample_ctx
struct ggml_tensor * probs = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_vocab, sample_ctx);

// 对 logits 进行 softmax 操作，将结果存储到 probs 中，使用 best_samples 进行采样
sample_softmax(logits, probs, best_samples);

// 获取 best_samples 中第 sample_ctx-1 个位置的整数值，存储到 token 中
int token = ggml_get_i32_1d(best_samples, sample_ctx-1);

// 打印 probs 中第 sample_at 行的数据
print_row(probs, sample_at);

// 打印 token 的值和 n_vocab 的值
print_token(token, n_vocab);

// 将 tokens_input 和 targets 向左移动一位
lshift_examples(tokens_input, targets, 1);

// 将 tokens_input 中第 0 个位置的值设为 0
ggml_set_i32_1d(tokens_input, 0, 0);

// 将 tokens_input 中第 sample_ctx-1 个位置的值设为 token
ggml_set_i32_1d(tokens_input, sample_ctx-1, token);

// 释放上下文 ctx0
ggml_free(ctx0);

// 打印 model.tok_embeddings 矩阵
print_matrix(model.tok_embeddings);

// 打印 "done"
printf("done\n");
// 释放 kv_self.ctx 对象占用的内存
// 释放 model_lora.ctx 对象占用的内存
// 释放 model.ctx 对象占用的内存
// 返回 0，表示函数执行成功
```