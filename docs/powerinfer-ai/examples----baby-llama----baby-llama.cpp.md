# `PowerInfer\examples\baby-llama\baby-llama.cpp`

```cpp
// 包含头文件 ggml.h 和 train.h
#include "ggml.h"
#include "train.h"

// 包含必要的标准库头文件
#include <vector>
#include <cassert>
#include <cstdlib>
#include <cstring>
#include <random>
#include <vector>

// 如果是 MSC 编译器，禁止警告 4244 和 4267
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 如果定义了 LLAMA_DEFAULT_RMS_EPS，则使用其值作为 rms_norm_eps，否则使用默认值 5e-6f
#ifdef LLAMA_DEFAULT_RMS_EPS
constexpr float rms_norm_eps = LLAMA_DEFAULT_RMS_EPS;
#else
constexpr float rms_norm_eps = 5e-6f;
#endif

// 辅助函数，用于计算图的计算
static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    // 根据图和线程数创建计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划的工作大小大于 0，则调整缓冲区大小并将工作数据指针指向缓冲区
    if (plan.work_size > 0) {
        buf.resize(plan.work_size);
        plan.work_data = buf.data();
    }

    // 执行图的计算
    ggml_graph_compute(graph, &plan);
}

// 随机化张量的值
static struct ggml_tensor * randomize_tensor(
    struct ggml_tensor * tensor, int ndims, const int64_t ne[], float fmin, float fmax
) {
    # 根据张量的维度进行不同的初始化操作
    switch (ndims) {
        # 当维度为1时
        case 1:
            # 遍历第一维度，对张量的每个元素进行随机初始化
            for (int i0 = 0; i0 < ne[0]; i0++) {
                ((float *)tensor->data)[i0] = frand()*(fmax - fmin) + fmin;
            }
            break;
        # 当维度为2时
        case 2:
            # 遍历第二维度和第一维度，对张量的每个元素进行随机初始化
            for (int i1 = 0; i1 < ne[1]; i1++) {
                for (int i0 = 0; i0 < ne[0]; i0++) {
                    ((float *)tensor->data)[i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                }
            }
            break;
        # 当维度为3时
        case 3:
            # 遍历第三维度、第二维度和第一维度，对张量的每个元素进行随机初始化
            for (int i2 = 0; i2 < ne[2]; i2++) {
                for (int i1 = 0; i1 < ne[1]; i1++) {
                    for (int i0 = 0; i0 < ne[0]; i0++) {
                        ((float *)tensor->data)[i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                    }
                }
            }
            break;
        # 当维度为4时
        case 4:
            # 遍历第四维度、第三维度、第二维度和第一维度，对张量的每个元素进行随机初始化
            for (int i3 = 0; i3 < ne[3]; i3++) {
                for (int i2 = 0; i2 < ne[2]; i2++) {
                    for (int i1 = 0; i1 < ne[1]; i1++) {
                        for (int i0 = 0; i0 < ne[0]; i0++) {
                            ((float *)tensor->data)[i3*ne[2]*ne[1]*ne[0] + i2*ne[1]*ne[0] + i1*ne[0] + i0] = frand()*(fmax - fmin) + fmin;
                        }
                    }
                }
            }
            break;
        # 默认情况下
        default:
            # 断言，维度不应该超出范围
            assert(false);
    }

    # 返回初始化后的张量
    return tensor;
}

// 定义了名为 llama_hparams 的结构体，包含了模型的超参数
struct llama_hparams {
    uint32_t n_vocab = 32000;  // 词汇表大小，默认为32000
    uint32_t n_ctx   = 512;    // 上下文大小，默认为512，这是用户提供的输入吗？
    uint32_t n_embd  = 4096;   // 嵌入维度，默认为4096
    uint32_t n_mult  = 4;      // 嵌入维度乘数，默认为4
    uint32_t n_head  = 32;     // 头数，默认为32
    uint32_t n_layer = 32;     // 层数，默认为32
    uint32_t n_rot   = 64;     // 旋转数，默认为64

    // 重载不等于运算符，用于比较两个 llama_hparams 结构体是否相等
    bool operator!=(const llama_hparams & other) const {
        return memcmp(this, &other, sizeof(llama_hparams));
    }
};

// 定义了一个静态函数 get_n_ff，用于计算前馈网络的大小
static uint32_t get_n_ff(const struct llama_hparams* hparams) {
    const uint32_t n_ff = ((2*(4*hparams->n_embd)/3 + hparams->n_mult - 1)/hparams->n_mult)*hparams->n_mult;
    return n_ff;
}

// 定义了名为 llama_hparams_lora 的结构体，包含了带有 LORA 的模型的超参数
struct llama_hparams_lora {
    uint32_t n_vocab = 32000;  // 词汇表大小，默认为32000
    uint32_t n_ctx   = 512;    // 上下文大小，默认为512，这是用户提供的输入吗？
    uint32_t n_embd  = 4096;   // 嵌入维度，默认为4096
    uint32_t n_mult  = 4;      // 嵌入维度乘数，默认为4
    uint32_t n_head  = 32;     // 头数，默认为32
    uint32_t n_layer = 32;     // 层数，默认为32
    uint32_t n_rot   = 64;     // 旋转数，默认为64
    uint32_t n_lora  = 64;     // LORA 数，默认为64

    // 重载不等于运算符，用于比较两个 llama_hparams_lora 结构体是否相等
    bool operator!=(const llama_hparams_lora & other) const {
        return memcmp(this, &other, sizeof(llama_hparams_lora)) != 0;
    }
};

// 定义了名为 llama_layer 的结构体，包含了模型的一层的张量
struct llama_layer {
    // normalization
    struct ggml_tensor * attention_norm;  // 注意力层的归一化张量

    // attention
    struct ggml_tensor * wq;  // 查询张量
    struct ggml_tensor * wk;  // 键张量
    struct ggml_tensor * wv;  // 值张量
    struct ggml_tensor * wo;  // 输出张量

    // normalization
    struct ggml_tensor * ffn_norm;  // 前馈网络的归一化张量

    // ff
    struct ggml_tensor * w1;  // 前馈网络的权重张量1
    struct ggml_tensor * w2;  // 前馈网络的权重张量2
    struct ggml_tensor * w3;  // 前馈网络的权重张量3
};

// 定义了名为 llama_layer_lora 的结构体，包含了带有 LORA 的模型的一层的张量
struct llama_layer_lora {
    // normalization
    struct ggml_tensor * attention_norm;  // 注意力层的归一化张量

    // attention
    struct ggml_tensor * wqa;  // 查询张量A
    struct ggml_tensor * wqb;  // 查询张量B
    struct ggml_tensor * wka;  // 键张量A
    struct ggml_tensor * wkb;  // 键张量B
    struct ggml_tensor * wva;  // 值张量A
    struct ggml_tensor * wvb;  // 值张量B
    struct ggml_tensor * woa;  // 输出张量A
    struct ggml_tensor * wob;  // 输出张量B

    // normalization
    struct ggml_tensor * ffn_norm;  // 前馈网络的归一化张量

    // ff
    struct ggml_tensor * w1;  // 前馈网络的权重张量1
    struct ggml_tensor * w2;  // 前馈网络的权重张量2
    struct ggml_tensor * w3;  // 前馈网络的权重张量3
};

// 定义了名为 llama_kv_cache 的结构体，包含了模型的键值缓存
struct llama_kv_cache {
    struct ggml_context * ctx = NULL;  // 上下文，默认为空

    struct ggml_tensor * k;  // 键张量
    # 定义指向 ggml_tensor 结构体的指针变量 v
    struct ggml_tensor * v;
    
    # 注释掉 llama_ctx_buffer buf 的定义，可能是暂时不需要使用的变量
    
    # 定义整型变量 n，表示当前缓存中的令牌数量
    int n; 
// 定义了一个名为 llama_model 的结构体
struct llama_model {
    // 指向 ggml_context 结构体的指针，初始化为 NULL
    struct ggml_context * ctx = NULL;

    // 存储 llama_model 的超参数
    llama_hparams hparams;

    // 指向 ggml_tensor 结构体的指针，用于存储 token embeddings
    struct ggml_tensor * tok_embeddings;

    // 指向 ggml_tensor 结构体的指针，用于存储 normalization
    struct ggml_tensor * norm;
    // 指向 ggml_tensor 结构体的指针，用于存储输出
    struct ggml_tensor * output;

    // 存储多个 llama_layer 结构体的 vector
    std::vector<llama_layer> layers;
};

// 定义了一个名为 llama_model_lora 的结构体
struct llama_model_lora {
    // 指向 ggml_context 结构体的指针，初始化为 NULL
    struct ggml_context * ctx = NULL;

    // 存储 llama_model_lora 的超参数
    llama_hparams_lora hparams;

    // 指向 ggml_tensor 结构体的指针，用于存储 token embeddings
    struct ggml_tensor * tok_embeddings;

    // 指向 ggml_tensor 结构体的指针，用于存储 normalization
    struct ggml_tensor * norm;
    // 指向 ggml_tensor 结构体的指针，用于存储输出a
    struct ggml_tensor * outputa;
    // 指向 ggml_tensor 结构体的指针，用于存储输出b
    struct ggml_tensor * outputb;

    // 存储多个 llama_layer_lora 结构体的 vector
    std::vector<llama_layer_lora> layers;
};

// 定义了一个静态函数，用于初始化 llama_model 结构体
static void init_model(struct llama_model * model) {
    // 获取 model 的超参数
    const auto & hparams = model->hparams;

    // 获取超参数中的 n_embd、n_layer、n_vocab
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;
    const uint32_t n_vocab = hparams.n_vocab;

    // 获取超参数中的 n_ff
    const uint32_t n_ff = get_n_ff(&hparams);

    // 获取 model 的上下文
    struct ggml_context * ctx = model->ctx;

    // 初始化 model 的 tok_embeddings，用于存储 token embeddings
    model->tok_embeddings = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab); // ("tok_embeddings.weight", {n_embd, n_vocab});
    // 初始化 model 的 norm，用于存储 normalization
    model->norm           = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);          // ("norm.weight",           {n_embd});
    // 初始化 model 的 output，用于存储输出
    model->output         = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab); // ("output.weight",         {n_embd, n_vocab});

    // 调整 model 的 layers 的大小为 n_layer
    model->layers.resize(n_layer);
    // 遍历每个层，i 从 0 到 n_layer-1
    for (uint32_t i = 0; i < n_layer; ++i) {
        // 获取当前层的引用
        auto & layer = model->layers[i];

        // 为当前层的 attention_norm 属性创建一个 n_embd 大小的一维张量
        layer.attention_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd); 

        // 为当前层的 wq 属性创建一个 n_embd x n_embd 大小的二维张量
        layer.wq = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);     

        // 为当前层的 wk 属性创建一个 n_embd x n_embd 大小的二维张量
        layer.wk = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);     

        // 为当前层的 wv 属性创建一个 n_embd x n_embd 大小的二维张量
        layer.wv = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);     

        // 为当前层的 wo 属性创建一个 n_embd x n_embd 大小的二维张量
        layer.wo = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);     

        // 为当前层的 ffn_norm 属性创建一个 n_embd 大小的一维张量
        layer.ffn_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);       

        // 为当前层的 w1 属性创建一个 n_embd x n_ff 大小的二维张量
        layer.w1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);     

        // 为当前层的 w2 属性创建一个 n_ff x n_embd 大小的二维张量
        layer.w2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_ff, n_embd);     

        // 为当前层的 w3 属性创建一个 n_embd x n_ff 大小的二维张量
        layer.w3 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);     
    }
# 初始化 LoRa 模型
static void init_model_lora(struct llama_model_lora * model) {
    # 获取模型超参数
    const auto & hparams = model->hparams;

    # 从超参数中获取各种维度信息
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_mult  = hparams.n_mult;
    const uint32_t n_layer = hparams.n_layer;
    const uint32_t n_vocab = hparams.n_vocab;
    const uint32_t n_lora  = hparams.n_lora;

    # 计算全连接层的输出维度
    const uint32_t n_ff = ((2*(4*n_embd)/3 + n_mult - 1)/n_mult)*n_mult;

    # 获取模型上下文
    struct ggml_context * ctx = model->ctx;

    # 初始化模型的嵌入层权重
    model->tok_embeddings = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab); // ("tok_embeddings.weight", {n_embd, n_vocab});
    # 初始化模型的归一化层权重
    model->norm           = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);          // ("norm.weight",           {n_embd});
    # 初始化模型的输出层权重 A
    model->outputa        = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_vocab); // ("output.weight",         {n_embd, n_vocab});
    # 初始化模型的输出层权重 B
    model->outputb        = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd,  n_lora); // ("output.weight",         {n_embd, n_vocab});

    # 调整模型的层数
    model->layers.resize(n_layer);
    // 遍历每个层
    for (uint32_t i = 0; i < n_layer; ++i) {
        // 获取当前层的引用
        auto & layer = model->layers[i];

        // 为当前层的 attention_norm 属性创建一个一维张量
        layer.attention_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd); 

        // 为当前层的 wqa 属性创建一个二维张量
        layer.wqa = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_embd);    
        layer.wqb = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora);    
        layer.wka = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_embd);    
        layer.wkb = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora);    
        layer.wva = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_embd);    
        layer.wvb = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora);    
        layer.woa = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_lora, n_embd);    
        layer.wob = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_lora);    

        // 为当前层的 ffn_norm 属性创建一个一维张量
        layer.ffn_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);       

        // 为当前层的 w1、w2、w3 属性创建二维张量
        layer.w1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd,   n_ff);     
        layer.w2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32,   n_ff, n_embd);     
        layer.w3 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd,   n_ff);     
    }
# 设置模型参数的函数，接受一个指向 llama_model 结构体的指针
static void set_param_model(struct llama_model * model) {
    # 获取模型的超参数
    const auto& hparams = model->hparams;

    # 获取层的数量
    const uint32_t n_layer = hparams.n_layer;

    # 获取模型的上下文
    struct ggml_context* ctx = model->ctx;

    # 设置模型的参数
    ggml_set_param(ctx, model->tok_embeddings);
    ggml_set_param(ctx, model->norm);
    ggml_set_param(ctx, model->output);

    # 遍历每一层，设置每一层的参数
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = model->layers[i];

        ggml_set_param(ctx, layer.attention_norm);
        ggml_set_param(ctx, layer.wq);
        ggml_set_param(ctx, layer.wk);
        ggml_set_param(ctx, layer.wv);
        ggml_set_param(ctx, layer.wo);
        ggml_set_param(ctx, layer.ffn_norm);
        ggml_set_param(ctx, layer.w1);
        ggml_set_param(ctx, layer.w2);
        ggml_set_param(ctx, layer.w3);
    }
}

# 设置 LORA 模型参数的函数，接受一个指向 llama_model_lora 结构体的指针
static void set_param_model_lora(struct llama_model_lora * model) {
    # 获取模型的超参数
    const auto& hparams = model->hparams;

    # 获取层的数量
    const uint32_t n_layer = hparams.n_layer;

    # 获取模型的上下文
    struct ggml_context* ctx = model->ctx;

    # 设置模型的参数
    ggml_set_param(ctx, model->tok_embeddings);
    ggml_set_param(ctx, model->norm);
    ggml_set_param(ctx, model->outputa);
    ggml_set_param(ctx, model->outputb);

    # 遍历每一层，设置每一层的参数
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = model->layers[i];

        ggml_set_param(ctx, layer.attention_norm);
        ggml_set_param(ctx, layer.wqa);
        ggml_set_param(ctx, layer.wqb);
        ggml_set_param(ctx, layer.wka);
        ggml_set_param(ctx, layer.wkb);
        ggml_set_param(ctx, layer.wva);
        ggml_set_param(ctx, layer.wvb);
        ggml_set_param(ctx, layer.woa);
        ggml_set_param(ctx, layer.wob);
        ggml_set_param(ctx, layer.ffn_norm);
        ggml_set_param(ctx, layer.w1);
        ggml_set_param(ctx, layer.w2);
        ggml_set_param(ctx, layer.w3);
    }
}

# 随机初始化模型参数的函数，接受一个指向 llama_model 结构体的指针，以及随机数种子和参数范围
static void randomize_model(struct llama_model * model, int seed, float mean, float std, float min, float max) {
    # 获取模型的超参数
    const auto & hparams = model->hparams;

    # 获取层的数量
    const uint32_t n_layer = hparams.n_layer;
    # 初始化一个服从正态分布的随机数生成器
    struct random_normal_distribution * rnd = init_random_normal_distribution(seed, mean, std, min, max);

    # 对模型中的词嵌入张量进行正态分布随机初始化
    randomize_tensor_normal(model->tok_embeddings , rnd);
    # 对模型中的规范化张量进行正态分布随机初始化
    randomize_tensor_normal(model->norm           , rnd);
    # 对模型中的输出张量进行正态分布随机初始化
    randomize_tensor_normal(model->output         , rnd);

    # 遍历每一层神经网络
    for (uint32_t i = 0; i < n_layer; ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];
        # 对当前层的注意力规范化张量进行正态分布随机初始化
        randomize_tensor_normal(layer.attention_norm, rnd);

        # 对当前层的查询权重张量进行正态分布随机初始化
        randomize_tensor_normal(layer.wq, rnd);
        # 对当前层的键权重张量进行正态分布随机初始化
        randomize_tensor_normal(layer.wk, rnd);
        # 对当前层的值权重张量进行正态分布随机初始化
        randomize_tensor_normal(layer.wv, rnd);
        # 对当前层的输出权重张量进行正态分布随机初始化
        randomize_tensor_normal(layer.wo, rnd);

        # 对当前层的前馈神经网络规范化张量进行正态分布随机初始化
        randomize_tensor_normal(layer.ffn_norm, rnd);

        # 对当前层的第一个全连接层权重张量进行正态分布随机初始化
        randomize_tensor_normal(layer.w1, rnd);
        # 对当前层的第二个全连接层权重张量进行正态分布随机初始化
        randomize_tensor_normal(layer.w2, rnd);
        # 对当前层的第三个全连接层权重张量进行正态分布随机初始化
        randomize_tensor_normal(layer.w3, rnd);
    }

    # 释放随机数生成器占用的内存
    free_random_normal_distribution(rnd);
}

// 随机初始化模型的 Lora 参数
static void randomize_model_lora(
    struct llama_model_lora * model, int seed, float mean, float std, float min, float max
) {
    // 获取模型的超参数
    const auto & hparams = model->hparams;

    // 获取层数
    const uint32_t n_layer = hparams.n_layer;

    // 初始化正态分布随机数生成器
    struct random_normal_distribution * rnd = init_random_normal_distribution(seed, mean, std, min, max);

    // 随机初始化模型的各个张量
    randomize_tensor_normal(model->tok_embeddings, rnd);
    randomize_tensor_normal(model->norm          , rnd);
    randomize_tensor_normal(model->outputa       , rnd);
    randomize_tensor_normal(model->outputb       , rnd);

    // 遍历每一层，随机初始化各个张量
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = model->layers[i];
        randomize_tensor_normal(layer.attention_norm, rnd);

        randomize_tensor_normal(layer.wqa, rnd);
        randomize_tensor_normal(layer.wqb, rnd);
        randomize_tensor_normal(layer.wka, rnd);
        randomize_tensor_normal(layer.wkb, rnd);
        randomize_tensor_normal(layer.wva, rnd);
        randomize_tensor_normal(layer.wvb, rnd);
        randomize_tensor_normal(layer.woa, rnd);
        randomize_tensor_normal(layer.wob, rnd);

        randomize_tensor_normal(layer.ffn_norm, rnd);

        randomize_tensor_normal(layer.w1, rnd);
        randomize_tensor_normal(layer.w2, rnd);
        randomize_tensor_normal(layer.w3, rnd);
    }

    // 释放随机数生成器
    free_random_normal_distribution(rnd);
}

// 初始化键值缓存
static void init_kv_cache(struct llama_kv_cache* cache, struct llama_model * model, int n_batch) {
    // 获取模型的超参数
    const auto & hparams = model->hparams;

    // 获取上下文长度、嵌入维度、层数
    const uint32_t n_ctx   = hparams.n_ctx;
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;

    // 计算内存大小和元素个数
    const int64_t n_mem      = n_layer*n_ctx*n_batch;
    const int64_t n_elements = n_embd*n_mem;

    // 初始化缓存
    // cache.buf.resize(2u*n_elements*ggml_type_size(wtype) + 2u*MB);

    // 初始化参数
    // struct ggml_init_params params;
    // params.mem_size   = cache.buf.size;
    // params.mem_buffer = cache.buf.addr;
    // params.no_alloc   = false;
}
    # 如果缓存上下文不存在
    if (!cache->ctx) {
        # 初始化参数结构体
        struct ggml_init_params params;
        # 计算需要的内存大小，包括元素数量、元素类型大小和额外的 2MB 空间
        params.mem_size   = 2u*n_elements*ggml_type_size(GGML_TYPE_F32) + 2u*1024*1024;
        # 内存缓冲区为空
        params.mem_buffer = NULL;
        # 允许分配内存
        params.no_alloc   = false;

        # 使用初始化参数创建缓存上下文
        cache->ctx = ggml_init(params);

        # 如果缓存上下文创建失败
        if (!cache->ctx) {
            # 输出错误信息到标准错误流
            fprintf(stderr, "%s: failed to allocate memory for kv cache\n", __func__);
            # 退出程序并返回错误码 1
            exit(1);
        }
    }

    # 使用缓存上下文创建新的一维张量作为键
    cache->k = ggml_new_tensor_1d(cache->ctx, GGML_TYPE_F32, n_elements);
    # 使用缓存上下文创建新的一维张量作为值
    cache->v = ggml_new_tensor_1d(cache->ctx, GGML_TYPE_F32, n_elements);
// 初始化基于 LoRa 的键值缓存
static bool init_kv_cache_lora(struct llama_kv_cache* cache, struct llama_model_lora * model, int n_batch) {
    // 获取模型的超参数
    const auto & hparams = model->hparams;

    // 从超参数中获取相关数值
    const uint32_t n_ctx   = hparams.n_ctx;
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;

    // 计算内存大小
    const int64_t n_mem      = n_layer*n_ctx*n_batch;
    const int64_t n_elements = n_embd*n_mem;

    // 调整缓存大小
    // cache.buf.resize(2u*n_elements*ggml_type_size(wtype) + 2u*MB);

    // 初始化参数结构体
    // struct ggml_init_params params;
    // params.mem_size   = cache.buf.size;
    // params.mem_buffer = cache.buf.addr;
    // params.no_alloc   = false;
    if (!cache->ctx) {
        // 设置初始化参数
        struct ggml_init_params params;
        params.mem_size   = 2u*n_elements*ggml_type_size(GGML_TYPE_F32) + 2u*1024*1024;
        params.mem_buffer = NULL;
        params.no_alloc   = false;

        // 初始化键值缓存上下文
        cache->ctx = ggml_init(params);

        // 检查初始化是否成功
        if (!cache->ctx) {
            fprintf(stderr, "%s: failed to allocate memory for kv cache\n", __func__);
            return false;
        }
    }

    // 创建键和值的张量
    cache->k = ggml_new_tensor_1d(cache->ctx, GGML_TYPE_F32, n_elements);
    cache->v = ggml_new_tensor_1d(cache->ctx, GGML_TYPE_F32, n_elements);

    return true;
}

// 前向传播函数
static struct ggml_tensor * forward(
    struct llama_model    * model,
    struct llama_kv_cache * cache,
    struct ggml_context   * ctx0,
    struct ggml_cgraph    * gf,
    struct ggml_tensor    * tokens_input,
    const  int              n_tokens,
    const  int              n_past
) {
    // 获取 tokens 的数量
    const int N = n_tokens;

    // 获取模型的超参数和相关数值
    struct llama_kv_cache& kv_self = *cache;
    const auto & hparams = model->hparams;
    const int n_ctx   = hparams.n_ctx;
    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_head  = hparams.n_head;
    const int n_rot   = hparams.n_rot;

    // 创建 tokens 张量
    struct ggml_tensor * tokens = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 复制输入 tokens 数据到新创建的 tokens 张量中
    memcpy(tokens->data, tokens_input->data, N*ggml_element_size(tokens));
}
    // 将 kv_self.k 赋值给指针 kc
    struct ggml_tensor * kc = kv_self.k;
    // 将 kv_self.v 赋值给指针 vc
    struct ggml_tensor * vc = kv_self.v;

    // 创建一个长度为 N 的一维整型张量 KQ_pos
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    {
        // 获取 KQ_pos 的数据指针
        int * data = (int *) KQ_pos->data;
        // 遍历 KQ_pos 的数据，赋值为 n_past + i
        for (int i = 0; i < N; ++i) {
            data[i] = n_past + i;
        }
    }

    // 获取 model->tok_embeddings 中 tokens 对应的行，赋值给 inpL
    // inpL 的形状为 [n_embd,N,1,1]
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model->tok_embeddings, tokens);

    // 对 inpL 进行均方根归一化处理
    {
        // inpL 的形状为 [n_embd,N,1,1]
        inpL = ggml_rms_norm(ctx0, inpL, rms_norm_eps);

        // 将 inpL 与 model->norm 的重复值相乘，再赋值给 inpL
        // inpL 的形状为 [n_embd,N,1,1]
        inpL = ggml_mul(ctx0,
                    ggml_repeat(ctx0, model->norm, inpL),
                    inpL);

        //embeddings = inpL;
    }

    // 将 model->output 与 inpL 相乘，再赋值给 inpL
    // inpL 的形状为 [n_vocab,N,1,1]
    inpL = ggml_mul_mat(ctx0, model->output, inpL);

    // 运行计算
    ggml_build_forward_expand(gf, inpL);

    // 返回 inpL
    return inpL;
// 定义一个名为 forward_batch 的静态函数，接受多个参数
static struct ggml_tensor * forward_batch(
    struct llama_model    * model,  // 指向 llama_model 结构体的指针参数
    struct llama_kv_cache * cache,  // 指向 llama_kv_cache 结构体的指针参数
    struct ggml_context   * ctx0,    // 指向 ggml_context 结构体的指针参数
    struct ggml_cgraph    * gf,      // 指向 ggml_cgraph 结构体的指针参数
    struct ggml_tensor    * tokens_input,  // 指向 ggml_tensor 结构体的指针参数
    const  int              n_tokens,     // 整型参数
    const  int              n_past,       // 整型参数
    const  int              n_batch       // 整型参数
) {
    const int N = n_tokens;  // 声明并初始化整型常量 N，值为 n_tokens

    struct llama_kv_cache& kv_self = *cache;  // 创建引用 kv_self，指向 cache
    const auto & hparams = model->hparams;    // 创建引用 hparams，指向 model 的 hparams
    // 以下为一系列整型常量的声明和初始化
    const int n_ctx   = hparams.n_ctx;
    const int n_vocab = hparams.n_vocab;
    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_head  = hparams.n_head;
    const int n_rot   = hparams.n_rot;
    const int n_ff    = get_n_ff(&hparams);

    // 创建一个新的 ggml_tensor 对象 tokens，数据类型为 GGML_TYPE_I32，维度为 N*n_batch
    struct ggml_tensor * tokens = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N*n_batch);
    // 将 tokens_input 的数据复制到 tokens 中
    memcpy(tokens->data, tokens_input->data, ggml_element_size(tokens)*N*n_batch);

    // 创建指向 kv_self.k 和 kv_self.v 的 ggml_tensor 指针对象
    struct ggml_tensor * kc = kv_self.k;
    struct ggml_tensor * vc = kv_self.v;

    // 创建一个新的 ggml_tensor 对象 KQ_pos，数据类型为 GGML_TYPE_I32，维度为 N
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    {
        int * data = (int *) KQ_pos->data;
        // 将 n_past + i 的值赋给 KQ_pos 的数据
        for (int i = 0; i < N; ++i) {
            data[i] = n_past + i;
        }
    }

    // norm
    {
        // 对 inpL 进行均方根归一化处理
        inpL = ggml_rms_norm(ctx0, inpL, rms_norm_eps);
        assert_shape_2d(inpL, n_embd, N*n_batch);

        // 将 norm 与 inpL 相乘
        inpL = ggml_mul(ctx0,
                    ggml_repeat(ctx0, model->norm, inpL),
                    inpL);

        assert_shape_2d(inpL, n_embd, N*n_batch);
    }

    // lm_head
    // 对 inpL 进行矩阵乘法运算
    inpL = ggml_mul_mat(ctx0, model->output, inpL);
    assert_shape_2d(inpL, n_vocab, N*n_batch);
}
    {
        // 将输入数据重新整形为 [n_vocab, N, n_batch, 1] 的形状
        inpL = ggml_reshape_3d(ctx0,
                        inpL,
                        n_vocab, N, n_batch);
        // 断言输入数据的形状为 [n_vocab, N, n_batch]
        assert_shape_3d(inpL, n_vocab, N, n_batch);
    }
    
    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    
    // 返回重新整形后的输入数据
    return inpL;
static struct ggml_tensor * forward_lora(
    struct llama_model_lora * model,
    struct llama_kv_cache   * cache,
    struct ggml_context     * ctx0,
    struct ggml_cgraph      * gf,
    struct ggml_tensor      * tokens_input,
    const  int                n_tokens,
    const  int                n_past
) {
    const int N = n_tokens;  // 定义变量 N，表示 tokens 的数量

    struct llama_kv_cache& kv_self = *cache;  // 获取 cache 中的 kv_self
    const auto & hparams = model->hparams;  // 获取 model 中的 hparams

    const int n_ctx   = hparams.n_ctx;  // 获取 hparams 中的 n_ctx
    const int n_embd  = hparams.n_embd;  // 获取 hparams 中的 n_embd
    const int n_layer = hparams.n_layer;  // 获取 hparams 中的 n_layer
    const int n_head  = hparams.n_head;  // 获取 hparams 中的 n_head
    const int n_rot   = hparams.n_rot;  // 获取 hparams 中的 n_rot

    struct ggml_tensor * tokens = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);  // 创建一个 1 维的整型 tensor，表示 tokens
    memcpy(tokens->data, tokens_input->data, N*ggml_element_size(tokens));  // 将 tokens_input 的数据复制到 tokens 中

    struct ggml_tensor * kc = kv_self.k;  // 获取 kv_self 中的 k
    struct ggml_tensor * vc = kv_self.v;  // 获取 kv_self 中的 v

    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);  // 创建一个 1 维的整型 tensor，表示 KQ_pos
    {
        int * data = (int *) KQ_pos->data;  // 获取 KQ_pos 的数据
        for (int i = 0; i < N; ++i) {  // 遍历 N
            data[i] = n_past + i;  // 将 n_past + i 的值赋给 KQ_pos 的每个元素
        }
    }

    // inpL shape [n_embd,N,1,1]
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model->tok_embeddings, tokens);  // 获取 tok_embeddings 中 tokens 对应的行向量

    // norm
    {

        // inpL shape [n_embd,N,1,1]
        inpL = ggml_rms_norm(ctx0, inpL, rms_norm_eps);  // 对 inpL 进行 RMS 归一化

        // inpL = norm*inpL
        // inpL shape [n_embd,N,1,1]
        inpL = ggml_mul(ctx0,
                    ggml_repeat(ctx0, model->norm, inpL),  // 将 norm 重复到与 inpL 相同的形状
                    inpL);  // 对 inpL 进行乘法运算

        //embeddings = inpL;
    }


    // lm_head
    // inpL shape [n_vocab,N,1,1]
    inpL = ggml_mul_mat(ctx0,
                model->outputa,  // 对 outputa 进行矩阵乘法
                    ggml_mul_mat(ctx0,
                        model->outputb,  // 对 outputb 进行矩阵乘法
                        inpL));  // 对 inpL 进行矩阵乘法

    // ggml_set_scratch(ctx0, { 0, 0, nullptr, });
    // run the computation
    ggml_build_forward_expand(gf, inpL);  // 运行前向传播计算

    return inpL;  // 返回 inpL
}
# 对输入的logits进行softmax操作，计算概率值，并找出最大概率对应的样本
static void sample_softmax(struct ggml_tensor * logits, struct ggml_tensor * probs, struct ggml_tensor * best_samples) {
    # 确保logits是一个二维张量
    assert(logits->n_dims == 2);
    # 确保probs是一个二维张量
    assert(probs->n_dims == 2);
    # 确保best_samples是一个一维张量
    assert(best_samples->n_dims == 1);
    # 确保logits的第二维大小等于best_samples的大小
    assert(logits->ne[1] == best_samples->ne[0]);
    # 确保logits的第一维大小等于probs的第一维大小
    assert(logits->ne[0] == probs->ne[0]);
    # 确保logits和probs的第二维大小相等
    assert(logits->ne[1] == probs->ne[1]);
    # 遍历logits的第二维
    for (int i = 0; i < logits->ne[1]; ++i) {
        # 获取logits中第i行的最大值
        float max_logit = ggml_get_f32_1d(logits, i * logits->ne[0]);
        # 将best_samples中第i个位置的值设为0
        ggml_set_i32_1d(best_samples, i, 0);
        # 遍历logits的第一维
        for (int k = 0; k < logits->ne[0]; ++k) {
            # 获取logits中第i行第k列的值
            float logit = ggml_get_f32_1d(logits, i * logits->ne[0] + k);
            # 如果当前值大于最大值，则更新最大值，并将best_samples中第i个位置的值设为k
            if (logit > max_logit) {
                max_logit = logit;
                ggml_set_i32_1d(best_samples, i, k);
            }
        }
        # 计算概率值的总和
        float psum = 0;
        # 遍历logits的第一维
        for (int k = 0; k < logits->ne[0]; ++k) {
            # 获取logits中第i行第k列的值
            float logit = ggml_get_f32_1d(logits, i * logits->ne[0] + k);
            # 计算概率值
            float p = (logit == -INFINITY) ? 0 : expf(logit - max_logit);
            # 更新概率值到probs中
            psum += p;
            ggml_set_f32_1d(probs, i * probs->ne[0] + k, p);
        }
        # 归一化概率值
        for (int k = 0; k < logits->ne[0]; ++k) {
            float p = ggml_get_f32_1d(probs, i*probs->ne[0] + k);
            ggml_set_f32_1d(probs, i * probs->ne[0] + k, p / psum);
        }
    }
}

# 对输入的logits进行softmax操作，计算概率值，并找出最大概率对应的样本（批处理版本）
static void sample_softmax_batch(
    struct ggml_context * ctx, struct ggml_tensor * logits, struct ggml_tensor * probs,
    struct ggml_tensor * best_samples
) {
    # 确保best_samples是一个二维张量
    GGML_ASSERT(best_samples->n_dims == 2);
    # 确保logits是一个三维张量
    GGML_ASSERT(logits->n_dims == 3);
    # 确保probs是一个三维张量
    GGML_ASSERT(probs->n_dims == 3);
    # 获取样本数量
    int n_tokens = best_samples->ne[0];
    # 获取批处理数量
    int n_batch  = best_samples->ne[1];
    # 获取词汇表大小
    int n_vocab  = logits->ne[0];
    # 确保样本数量等于logits的第二维大小
    GGML_ASSERT(n_tokens == logits->ne[1]);
    # 确保批处理数量等于logits的第三维大小
    GGML_ASSERT(n_batch  == logits->ne[2]);
    # 确保词汇表大小等于probs的第一维大小
    GGML_ASSERT(n_vocab  == probs->ne[0]);
    # 确保样本数量等于probs的第二维大小
    GGML_ASSERT(n_tokens == probs->ne[1]);
    # 确保批处理数量等于probs的第三维大小
    GGML_ASSERT(n_batch  == probs->ne[2]);
    # 遍历批次中的每个元素
    for (int k = 0; k < n_batch; ++k) {
        # 从 best_samples 中获取第 k 个批次的数据视图
        struct ggml_tensor * best_samples_k = ggml_view_1d(ctx,
                                                best_samples,
                                                best_samples->ne[0],
                                                k*best_samples->nb[1]);
        # 从 logits 中获取第 k 个批次的数据视图
        struct ggml_tensor * logits_k       = ggml_view_2d(ctx,
                                                logits,
                                                logits->ne[0],
                                                logits->ne[1],
                                                logits->nb[1],
                                                k*logits->nb[2]);
        # 从 probs 中获取第 k 个批次的数据视图
        struct ggml_tensor * probs_k        = ggml_view_2d(ctx,
                                                probs,
                                                probs->ne[0],
                                                probs->ne[1],
                                                probs->nb[1],
                                                k*probs->nb[2]);
        # 对第 k 个批次的数据进行 softmax 操作
        sample_softmax(logits_k, probs_k, best_samples_k);
    }
static void print_row(struct ggml_tensor * probs, int i) {
    // 打印指定行的概率数据
    for (int k = 0; k < probs->ne[0]; ++k) {
        // 获取指定位置的概率值
        float p = ggml_get_f32_1d(probs, i*probs->ne[0] + k);
        // 打印概率值
        printf(" %.2f", p);
    }
    // 换行
    printf("\n");
}

static void print_matrix(struct ggml_tensor * probs) {
    // 断言概率数据的维度为2
    assert(probs->n_dims == 2);
    // 遍历概率矩阵，打印每一行的概率值
    for (int i = 0; i < probs->ne[1]; ++i) {
        for (int k = 0; k < probs->ne[0]; ++k) {
            // 获取指定位置的概率值
            float p = ggml_get_f32_1d(probs, i*probs->ne[0] + k);
            // 打印概率值
            printf(" %.2f", p);
        }
        // 换行
        printf("\n");
    }
}

static void print_token(int token, int n_vocab) {
    // 打印表示token的字符串，X表示当前token
    for (int k = 0; k < token; ++k) {
        printf(" ");
    }
    printf("X");
    for (int k = token+1; k < n_vocab; ++k) {
        printf(" ");
    }
    printf("\n");
}

static void print_tokens(struct ggml_tensor * tokens, int n_vocab) {
    // 打印tokens对应的字符串表示
    for (int i=0; i<tokens->ne[0]; ++i) {
        // 获取token值
        int token = ggml_get_i32_1d(tokens, i);
        // 打印token对应的字符串表示
        print_token(token, n_vocab);
    }
}

static void get_example_targets(int example_id, struct ggml_tensor * tokens_input, struct ggml_tensor * targets) {
    // 获取tokens的数量
    int n_tokens = tokens_input->ne[0];
    // 获取targets的词汇表大小
    int n_vocab = targets->ne[0];
    // 设置随机性
    float randomness = 0.0f;
    // 将targets初始化为-1.0
    ggml_set_f32(targets, -1.0f);
    // 将tokens_input的第一个位置设置为0
    ggml_set_i32_1d(tokens_input, 0, 0);
    // 遍历tokens
    for (int i=1; i<n_tokens+1; ++i) {
        // 计算x值
        float x = example_id + i * 3.14159f * 2.0f * 1.0f * 0.5f / n_tokens;
        // 计算y值
        float y = sinf(x);
        // 将y值映射到[0,1]区间
        float z = (y+1.0f)*0.5f; // scale to [0..1]
        // 添加一定的随机性
        z += (frand()-0.5f)*(randomness/n_vocab);
        // 将z值限制在[0,1]区间
        z = (z < 0.0f) ? 0.0f : (z > 1.0f) ? 1.0f : z; // clamp to [0..1]
        // 根据z值计算token值
        int token = std::max(1,std::min(1+(int)(z*(float)(n_vocab-1)), n_vocab-1));
        // 将对应位置的targets设置为1.0
        ggml_set_f32_1d(targets, (i-1)*n_vocab + token, +1.0f);
        // 如果不是最后一个token，将tokens_input对应位置设置为token
        if (i<n_tokens) {
            ggml_set_i32_1d(tokens_input, i, token);
        }
    }
}

static void get_example_targets_batch(
    # 声明一个指向 ggml_context 结构体的指针 ctx，参数包括 example_id、tokens_input 和 targets
    struct ggml_context * ctx, int example_id, struct ggml_tensor * tokens_input, struct ggml_tensor * targets
// 确保输入 tokens_input 的维度为 2
GGML_ASSERT(tokens_input->n_dims == 2);
// 确保输入 targets 的维度为 3
GGML_ASSERT(targets->n_dims == 3);
// 获取 tokens_input 的第一维大小
int n_tokens = tokens_input->ne[0];
// 获取 tokens_input 的第二维大小
int n_batch  = tokens_input->ne[1];
// 确保 n_tokens 与 targets 的第二维大小相等
GGML_ASSERT(n_tokens == targets->ne[1]);
// 确保 n_batch 与 targets 的第三维大小相等
GGML_ASSERT(n_batch  == targets->ne[2]);

// 遍历每个 batch
for (int k=0; k<n_batch; ++k) {
    // 获取 tokens_input 的第 k 列作为新的 tensor
    struct ggml_tensor * tokens_input_k = ggml_view_1d(ctx,
                                            tokens_input,
                                            tokens_input->ne[0],
                                            k*tokens_input->nb[1]);
    // 获取 targets 的第 k 个二维子张量作为新的 tensor
    struct ggml_tensor * targets_k    = ggml_view_2d(ctx,
                                            targets,
                                            targets->ne[0],
                                            targets->ne[1],
                                            targets->nb[1],
                                            k*targets->nb[2]);
    // 调用函数获取 example 的 targets
    get_example_targets(example_id*n_batch + k, tokens_input_k, targets_k);
}

// 对 tokens_input 和 targets 进行左移操作
static void lshift_examples(struct ggml_tensor * tokens_input, struct ggml_tensor * targets, int n_shift) {
    // 获取 tokens_input 的第一维大小
    int n_tokens = tokens_input->ne[0];
    // 获取 targets 的第一维大小
    int n_vocab = targets->ne[0];
    // 遍历 tokens_input 进行左移操作
    for (int i=0; i<n_tokens-n_shift; ++i) {
        // 将 tokens_input 的第 i+n_shift 个元素赋值给第 i 个元素
        ggml_set_i32_1d(tokens_input, i, ggml_get_i32_1d(tokens_input, i + n_shift));
        // 遍历 targets 进行左移操作
        for (int k=0; k<n_vocab; ++k) {
            // 将 targets 的第 (i+n_shift)*n_vocab+k 个元素赋值给第 i*n_vocab+k 个元素
            ggml_set_f32_1d(targets, i*n_vocab + k, ggml_get_f32_1d(targets, (i + n_shift)*n_vocab + k));
        }
    }
}

// 计算平方误差损失
static struct ggml_tensor * square_error_loss(
    struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b
) {
    // 返回 a 和 b 元素对应位置的平方差的和
    return ggml_sum(ctx, ggml_sqr(ctx, ggml_sub(ctx, a, b)));
}

// 计算交叉熵损失
static struct ggml_tensor * cross_entropy_loss(
    struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b
) {
    // 定义一个很小的数值 eps
    const float eps = 1e-3f;
    # 返回以下表达式的结果
        # 对上下文 ctx 中的 a 应用 softmax 函数
        # 对上下文 ctx 中的 b 应用 softmax 函数
        # 对上下文 ctx 中的 b 应用 softmax 函数后加上一个小的浮点数 eps
        # 对上下文 ctx 中的 b 应用 softmax 函数后加上一个小的浮点数 eps后取对数
        # 对上下文 ctx 中的 a 和上述结果相乘
        # 对上述结果取负数
        # 对上述结果进行行求和
        # 对上述结果取负数后求和
}

int main(int argc, char ** argv) {
    // 检查命令行参数数量是否小于1，如果是则打印使用方法并返回1
    if (argc < 1) {
        fprintf(stderr, "usage: %s\n", argv[0]);
        return 1;
    }

    // 初始化 ggml_init_params 结构体 lcparams
    struct ggml_init_params lcparams;
    lcparams.mem_size   = 1024ll*1024ll*1024ll;
    lcparams.mem_buffer = NULL;
    lcparams.no_alloc   = false;

    // 初始化 llama_model 结构体 model
    struct llama_model model;
    model.hparams.n_vocab = 8;
    model.hparams.n_ctx   = 8;
    model.hparams.n_embd  = 32;
    model.hparams.n_mult  = 2;
    model.hparams.n_head  = 8;
    model.hparams.n_layer = 1;
    model.hparams.n_rot   = std::min(16u, model.hparams.n_embd / model.hparams.n_head);

    // 初始化 model 的上下文
    model.ctx = ggml_init(lcparams);
    // 打印初始化信息
    printf("init model\n");
    // 初始化 model
    init_model(&model);
    // 设置 model 参数
    set_param_model(&model);

    // 随机初始化 model
    randomize_model(&model, 1337, 0.0f, 1.0f, -1.0f, +1.0f);

/*
    struct llama_model_lora model_lora;
    // model.hparams.n_vocab = 6;
    // model.hparams.n_ctx   = 64;
    // model.hparams.n_embd  = 128;
    // model.hparams.n_mult  = 2;
    // model.hparams.n_head  = 8;
    // model.hparams.n_layer = 6;
    // model.hparams.n_rot   = model.hparams.n_embd / model.hparams.n_head;

    // 初始化 llama_model_lora 结构体 model_lora
    model_lora.hparams.n_vocab = 16;
    model_lora.hparams.n_ctx   = 32;
    model_lora.hparams.n_embd  = 256;
    model_lora.hparams.n_mult  = 2;
    model_lora.hparams.n_head  = 16;
    model_lora.hparams.n_layer = 1;
    model_lora.hparams.n_lora  = 64;
    model_lora.hparams.n_rot   = MIN(16, model_lora.hparams.n_embd / model_lora.hparams.n_head);
    // model.hparams.n_rot   = (model.hparams.n_embd / model.hparams.n_head) / 2;

    // model.hparams.n_embd  = 32;
    // model.hparams.n_mult  = 2;
    // model.hparams.n_head  = 4;
    // model.hparams.n_layer = 8;
    // model.hparams.n_rot   = 8;

    // 初始化 model_lora 的上下文
    model_lora.ctx = ggml_init(lcparams);
    // 打印初始化信息
    printf("init model_lora\n");
    // 初始化 model_lora
    init_model_lora(&model_lora);
    # 设置参数模型为 LoRa 模型
    set_param_model_lora(&model_lora);
    # 为 LoRa 模型随机生成参数
    randomize_model_lora(&model_lora, 1337, 0.0f, 1.0f, -1.0f, +1.0f);
    // 定义一个整型变量n_batch，值为8，表示批处理的大小
    int n_batch = 8;
    // 定义一个结构体llama_kv_cache，用于自注意力机制的键值缓存
    struct llama_kv_cache kv_self;
    // 打印初始化键值缓存的消息
    printf("init_kv_cache\n");
    // 将模型的上下文赋给键值缓存的上下文
    kv_self.ctx = model.ctx;
    // 初始化键值缓存，传入键值缓存结构体指针、模型指针和批处理大小
    init_kv_cache(&kv_self, &model, n_batch);
    // 初始化计算大小为1GB的内存空间
    size_t    compute_size = 1024ll*1024ll*1024ll;
    // 为计算分配1GB大小的内存空间
    uint8_t * compute_addr = new uint8_t[compute_size];
    // 定义变量n_examples，表示示例的数量为256
    int n_examples = 256;
    // 定义变量n_tokens，表示模型的上下文大小
    int n_tokens = model.hparams.n_ctx;
    // 定义变量n_vocab，表示模型的词汇表大小
    int n_vocab  = model.hparams.n_vocab;
    // 定义一个存储uint8_t类型数据的动态数组work_buffer
    std::vector<uint8_t> work_buffer;
    {
        // 设置生成的标记数量
        int n_gen = 128;
        // 设置样本上下文
        int sample_ctx = n_tokens-n_tokens/8;

        // 打印生成的标记数量
        printf("Generating %d tokens.\n", n_gen);

        // 创建输入标记的张量
        struct ggml_tensor * tokens_input = ggml_new_tensor_1d(model.ctx, GGML_TYPE_I32, n_tokens);
        // 创建目标标记的张量
        struct ggml_tensor * targets      = ggml_new_tensor_2d(model.ctx, GGML_TYPE_F32, n_vocab, n_tokens);

        // 获取示例目标
        get_example_targets(137, tokens_input, targets);
        // 对于样本上下文之后的每个标记，将输入标记设置为词汇表大小的一半
        for (int i=sample_ctx; i<n_tokens; ++i) {
            ggml_set_i32_1d(tokens_input, i, n_vocab/2);
        }

        // 打印样本上下文之前的标记
        for (int i=0; i<sample_ctx-1; ++i) {
            print_token(ggml_get_i32_1d(tokens_input, i), n_vocab);
        }
        printf("---\n");

        // 生成新的标记
        for (int i=0; i<n_gen; ++i) {
            // 初始化参数
            struct ggml_init_params params = {
                /*.mem_size   =*/ compute_size,
                /*.mem_buffer =*/ compute_addr,
                /*.no_alloc   =*/ false,
            };
            // 初始化上下文
            struct ggml_context * ctx0 = ggml_init(params);

            // 创建计算图
            ggml_cgraph gf = {};

            int n_past = 0;
            // 前向传播获取逻辑张量
            struct ggml_tensor * logits = forward(&model, &kv_self, ctx0, &gf, tokens_input, sample_ctx, n_past);

            // 构建前向传播扩展
            ggml_build_forward_expand(&gf, logits);
            // 计算图计算
            ggml_graph_compute_helper(work_buffer, &gf, /*n_threads*/ 1);

            // 创建最佳样本的张量和概率的张量
            struct ggml_tensor * best_samples = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, sample_ctx);
            struct ggml_tensor * probs        = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_vocab, sample_ctx);

            // 采样 softmax
            sample_softmax(logits, probs, best_samples);

            // 获取标记
            int token = ggml_get_i32_1d(best_samples, sample_ctx-1);

            // 打印标记
            print_token(token, n_vocab);

            // 左移示例
            lshift_examples(tokens_input, targets, 1);
            ggml_set_i32_1d(tokens_input, 0, 0);
            ggml_set_i32_1d(tokens_input, sample_ctx-1, token);

            // 释放上下文
            ggml_free(ctx0);
        }
    }

    // 打印模型的标记嵌入矩阵
    print_matrix(model.tok_embeddings);
    // 打印 "done"，表示程序执行完成
    printf("done\n");

    // 释放 kv_self.ctx 占用的内存
    // 释放 model_lora.ctx 占用的内存
    // 释放 model.ctx 占用的内存
    ggml_free(kv_self.ctx);
    ggml_free(model_lora.ctx);
    ggml_free(model.ctx);

    // 返回 0，表示程序正常结束
    return 0;
# 闭合前面的函数定义
```