# `PowerInfer\examples\train-text-from-scratch\train-text-from-scratch.cpp`

```
// 包含所需的头文件
#include "ggml.h"
#include "ggml-alloc.h"
#include "common.h"
#include "train.h"
#include "llama.h"
#include <unordered_map>
#include <vector>
#include <cassert>
#include <climits>
#include <cstring>
#include <cstdarg>
#include <ctime>
#include <random>
#include <stdexcept>
#include <algorithm>
#include <string>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止警告 4244 和 4267，可能会丢失数据
#endif
// 定义张量的对齐方式为32
static const size_t tensor_alignment = 32;

// 定义LLAMA模型的超参数结构体
struct my_llama_hparams {
    uint32_t n_vocab = 32000; // 词汇量大小
    uint32_t n_ctx   = 512;   // 上下文大小
    uint32_t n_embd  = 4096;  // 嵌入维度
    uint32_t n_head  = 32;    // 头数
    uint32_t n_layer = 32;    // 层数
    uint32_t n_rot   = 64;    // 旋转数
    uint32_t n_ff    = 11008; // 前馈网络大小

    // float f_norm_eps     = 1e-5f; // falcon
    float f_norm_rms_eps = 1e-5f; // llama // 归一化RMS的epsilon值

    float rope_freq_base  = 10000.0f; // 绳索频率基数
    float rope_freq_scale = 1.0f;    // 绳索频率缩放
};

struct my_llama_layer {
// 创建一个指向 ggml_tensor 结构体的指针 attention_norm，用于存储归一化后的注意力张量

// 创建指向 ggml_tensor 结构体的指针 wq，用于存储查询张量
// 创建指向 ggml_tensor 结构体的指针 wk，用于存储键张量
// 创建指向 ggml_tensor 结构体的指针 wv，用于存储值张量
// 创建指向 ggml_tensor 结构体的指针 wo，用于存储输出张量

// 创建一个指向 ggml_tensor 结构体的指针 ffn_norm，用于存储归一化后的前馈神经网络张量

// 创建指向 ggml_tensor 结构体的指针 w1，用于存储前馈神经网络的权重张量
// 创建指向 ggml_tensor 结构体的指针 w2，用于存储前馈神经网络的权重张量
// 创建指向 ggml_tensor 结构体的指针 w3，用于存储前馈神经网络的权重张量

// 创建一个指向 ggml_context 结构体的指针 ctx，并将其初始化为 NULL
// 定义一个存储无符号8位整数的向量
std::vector<uint8_t> data;

// 定义一个名为hparams的my_llama_hparams结构体

// 定义一个指向ggml_tensor结构体的指针，名为tok_embeddings

// 定义指向ggml_tensor结构体的指针，名为norm和output

// 定义一个存储my_llama_layer结构体的向量，名为layers

// gguf常量（与gguf.py同步）
static const char * LLM_KV_TRAINING_TYPE_TRAIN_MODEL = "train_model";
static const char * LLM_KV_TRAINING_TYPE = "training.type";

static const char * LLM_KV_GENERAL_ARCHITECTURE = "general.architecture";
static const char * LLM_KV_GENERAL_FILE_TYPE = "general.file_type";

static const char * LLM_KV_CONTEXT_LENGTH = "%s.context_length";
// 定义常量字符串，用于表示不同的键值对
static const char * LLM_KV_EMBEDDING_LENGTH            = "%s.embedding_length";
static const char * LLM_KV_BLOCK_COUNT                 = "%s.block_count";
static const char * LLM_KV_FEED_FORWARD_LENGTH         = "%s.feed_forward_length";
static const char * LLM_KV_ATTENTION_HEAD_COUNT        = "%s.attention.head_count";
static const char * LLM_KV_ATTENTION_LAYERNORM_RMS_EPS = "%s.attention.layer_norm_rms_epsilon";
static const char * LLM_KV_ROPE_DIMENSION_COUNT        = "%s.rope.dimension_count";
static const char * LLM_KV_ROPE_FREQ_BASE              = "%s.rope.freq_base"; // TODO load in llama.cpp
static const char * LLM_KV_ROPE_SCALE_LINEAR           = "%s.rope.scale_linear";

static const char * LLM_KV_TOKENIZER_MODEL             = "tokenizer.ggml.model";
static const char * LLM_KV_TOKENIZER_LIST              = "tokenizer.ggml.tokens";
static const char * LLM_KV_TOKENIZER_TOKEN_TYPE        = "tokenizer.ggml.token_type";
static const char * LLM_KV_TOKENIZER_SCORES            = "tokenizer.ggml.scores";
static const char * LLM_KV_TOKENIZER_MERGES            = "tokenizer.ggml.merges";
static const char * LLM_KV_TOKENIZER_BOS_ID            = "tokenizer.ggml.bos_token_id";
static const char * LLM_KV_TOKENIZER_EOS_ID            = "tokenizer.ggml.eos_token_id";
static const char * LLM_KV_TOKENIZER_UNK_ID            = "tokenizer.ggml.unknown_token_id";
static const char * LLM_KV_TOKENIZER_SEP_ID            = "tokenizer.ggml.seperator_token_id";
static const char * LLM_KV_TOKENIZER_PAD_ID            = "tokenizer.ggml.padding_token_id";
# 定义常量，表示不同类型的张量名称
static const char * LLM_TENSOR_TOKEN_EMBD    = "token_embd";
static const char * LLM_TENSOR_OUTPUT_NORM   = "output_norm";
static const char * LLM_TENSOR_OUTPUT        = "output";
static const char * LLM_TENSOR_ATTN_NORM     = "blk.%d.attn_norm";
static const char * LLM_TENSOR_ATTN_Q        = "blk.%d.attn_q";
static const char * LLM_TENSOR_ATTN_K        = "blk.%d.attn_k";
static const char * LLM_TENSOR_ATTN_V        = "blk.%d.attn_v";
static const char * LLM_TENSOR_ATTN_OUT      = "blk.%d.attn_output";
static const char * LLM_TENSOR_FFN_NORM      = "blk.%d.ffn_norm";
static const char * LLM_TENSOR_FFN_GATE      = "blk.%d.ffn_gate";
static const char * LLM_TENSOR_FFN_DOWN      = "blk.%d.ffn_down";
static const char * LLM_TENSOR_FFN_UP        = "blk.%d.ffn_up";

# 定义函数，打印模型参数
static void print_params(struct my_llama_hparams * params) {
    # 打印参数的名称和值
    printf("%s: n_vocab: %d\n", __func__, params->n_vocab);
    printf("%s: n_ctx:   %d\n", __func__, params->n_ctx);
    printf("%s: n_embd:  %d\n", __func__, params->n_embd);
    printf("%s: n_head:  %d\n", __func__, params->n_head);
    printf("%s: n_ff:    %d\n", __func__, params->n_ff);
    printf("%s: n_layer: %d\n", __func__, params->n_layer);
// 打印函数名和参数n_rot的值
printf("%s: n_rot:   %d\n", __func__, params->n_rot);
}

// 设置模型参数
static void set_param_model(struct my_llama_model * model) {
    // 获取模型超参数
    const auto& hparams = model->hparams;

    // 获取层数
    const uint32_t n_layer = hparams.n_layer;

    // 获取模型上下文
    struct ggml_context* ctx = model->ctx;

    // 设置模型参数
    ggml_set_param(ctx, model->tok_embeddings);
    ggml_set_param(ctx, model->norm);
    ggml_set_param(ctx, model->output);

    // 遍历每一层，设置参数
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = model->layers[i];

        ggml_set_param(ctx, layer.attention_norm);
        ggml_set_param(ctx, layer.wq);
        ggml_set_param(ctx, layer.wk);
// 设置参数到上下文中
ggml_set_param(ctx, layer.wv);
ggml_set_param(ctx, layer.wo);
ggml_set_param(ctx, layer.ffn_norm);
ggml_set_param(ctx, layer.w1);
ggml_set_param(ctx, layer.w2);
ggml_set_param(ctx, layer.w3);
}

// 分配模型内存
static void alloc_model(struct ggml_allocr * alloc, struct my_llama_model * model) {
    ggml_allocr_alloc(alloc, model->tok_embeddings);
    ggml_allocr_alloc(alloc, model->norm);
    ggml_allocr_alloc(alloc, model->output);
    // 遍历模型的每一层，分配内存
    for (uint32_t i = 0; i < model->layers.size(); ++i) {
        auto & layer = model->layers[i];
        ggml_allocr_alloc(alloc, layer.attention_norm);
        ggml_allocr_alloc(alloc, layer.wq);
        ggml_allocr_alloc(alloc, layer.wk);
        ggml_allocr_alloc(alloc, layer.wv);
        ggml_allocr_alloc(alloc, layer.wo);
    }
}
# 为 layer.ffn_norm 分配内存空间
ggml_allocr_alloc(alloc, layer.ffn_norm);
# 为 layer.w1 分配内存空间
ggml_allocr_alloc(alloc, layer.w1);
# 为 layer.w2 分配内存空间
ggml_allocr_alloc(alloc, layer.w2);
# 为 layer.w3 分配内存空间
ggml_allocr_alloc(alloc, layer.w3);
# 为 model->tok_embeddings->grad 分配内存空间
ggml_allocr_alloc(alloc, model->tok_embeddings->grad);
# 为 model->norm->grad 分配内存空间
ggml_allocr_alloc(alloc, model->norm->grad);
# 为 model->output->grad 分配内存空间
ggml_allocr_alloc(alloc, model->output->grad);
# 遍历 model->layers 中的每个层，为其相关属性分配内存空间
for (uint32_t i = 0; i < model->layers.size(); ++i) {
    auto & layer = model->layers[i];
    # 为 layer.attention_norm->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.attention_norm->grad);
    # 为 layer.wq->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.wq->grad);
    # 为 layer.wk->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.wk->grad);
    # 为 layer.wv->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.wv->grad);
    # 为 layer.wo->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.wo->grad);
    # 为 layer.ffn_norm->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.ffn_norm->grad);
    # 为 layer.w1->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.w1->grad);
    # 为 layer.w2->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.w2->grad);
    # 为 layer.w3->grad 分配内存空间
    ggml_allocr_alloc(alloc, layer.w3->grad);
}
// 初始化模型结构体
static void init_model(struct my_llama_model * model) {
    // 获取模型超参数
    const auto & hparams = model->hparams;

    // 获取超参数中的各个值
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;
    const uint32_t n_vocab = hparams.n_vocab;
    const uint32_t n_ff    = hparams.n_ff;

    // 创建存储临时字符串的缓冲区
    std::vector<char> tn_buf;
    tn_buf.resize(GGML_MAX_NAME);
    // 定义一个lambda函数，用于生成带有".weight"后缀的字符串
    auto tn = [&tn_buf](const char * key) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", key);
        return tn_buf.data();
    };
    // 定义一个lambda函数，用于生成带有指定编号的字符串
    auto tni = [&tn_buf](const char * key, int bid) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), key, bid);
        std::string s = tn_buf.data();
// 使用字符串 s 创建一个新的字符串，格式为 "%s.weight"，并将结果存储在 tn_buf 中
snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", s.c_str());
// 返回 tn_buf 中的数据
return tn_buf.data();
};

// 初始化模型张量的上下文，设置内存大小和内存缓冲区
struct ggml_init_params ctx_model_params;
ctx_model_params.mem_size   = ggml_tensor_overhead()*2*(6 + n_layer*18);
ctx_model_params.mem_buffer = NULL;
ctx_model_params.no_alloc   = true;

// 初始化模型的上下文
struct ggml_context * ctx = ggml_init(ctx_model_params);
model->ctx = ctx;

// 创建模型的张量，设置张量的类型和大小
model->tok_embeddings = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab);
model->norm           = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
model->output         = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab);

// 设置模型张量的名称
ggml_set_name(model->tok_embeddings, tn(LLM_TENSOR_TOKEN_EMBD));
ggml_set_name(model->norm,           tn(LLM_TENSOR_OUTPUT_NORM));
ggml_set_name(model->output,         tn(LLM_TENSOR_OUTPUT));
    // 调整模型的层数
    model->layers.resize(n_layer);
    // 遍历每一层
    for (uint32_t i = 0; i < n_layer; ++i) {
        // 获取当前层
        auto & layer = model->layers[i];

        // 初始化当前层的注意力规范化张量
        layer.attention_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        // 初始化当前层的查询权重张量
        layer.wq = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        // 初始化当前层的键权重张量
        layer.wk = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        // 初始化当前层的值权重张量
        layer.wv = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        // 初始化当前层的输出权重张量
        layer.wo = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);

        // 初始化当前层的前馈神经网络规范化张量
        layer.ffn_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        // 初始化当前层的前馈神经网络权重张量1
        layer.w1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);
        // 初始化当前层的前馈神经网络权重张量2
        layer.w2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_ff, n_embd);
        // 初始化当前层的前馈神经网络权重张量3
        layer.w3 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);

        // 设置当前层的注意力规范化张量的名称
        ggml_set_name(layer.attention_norm, tni(LLM_TENSOR_ATTN_NORM, i));
# 设置层的权重名称，将注意力机制的查询、键、值和输出的权重名称设置为对应的张量索引
ggml_set_name(layer.wq, tni(LLM_TENSOR_ATTN_Q, i));
ggml_set_name(layer.wk, tni(LLM_TENSOR_ATTN_K, i));
ggml_set_name(layer.wv, tni(LLM_TENSOR_ATTN_V, i));
ggml_set_name(layer.wo, tni(LLM_TENSOR_ATTN_OUT, i));

# 设置层的前馈神经网络归一化层的名称
ggml_set_name(layer.ffn_norm, tni(LLM_TENSOR_FFN_NORM, i));

# 设置层的前馈神经网络的门、下采样和上采样的权重名称
ggml_set_name(layer.w1, tni(LLM_TENSOR_FFN_GATE, i));
ggml_set_name(layer.w2, tni(LLM_TENSOR_FFN_DOWN, i));
ggml_set_name(layer.w3, tni(LLM_TENSOR_FFN_UP, i));

# 设置模型参数
set_param_model(model);

# 测量数据大小
size_t size = 0;
for (struct ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
    size += GGML_PAD(ggml_nbytes(t), tensor_alignment);
}
    // 分配数据
    struct ggml_allocr * alloc = NULL; // 声明一个指向 ggml_allocr 结构体的指针，并初始化为 NULL
    model->data.resize(size + tensor_alignment); // 调整 model->data 的大小为 size + tensor_alignment
    alloc = ggml_allocr_new(model->data.data(), model->data.size(), tensor_alignment); // 使用 model->data 创建一个 ggml_allocr 对象，并赋值给 alloc
    alloc_model(alloc, model); // 分配模型数据
    ggml_allocr_free(alloc); // 释放分配的内存
}

static void randomize_model(struct my_llama_model * model, int seed, float mean, float std, float min, float max) {
    const auto & hparams = model->hparams; // 获取模型的超参数

    const uint32_t n_layer = hparams.n_layer; // 获取层数

    struct random_normal_distribution * rnd = init_random_normal_distribution(seed, mean, std, min, max); // 初始化一个随机正态分布对象

    randomize_tensor_normal(model->tok_embeddings, rnd); // 使用随机正态分布对象随机化模型的 tok_embeddings 张量
    randomize_tensor_normal(model->norm,           rnd); // 使用随机正态分布对象随机化模型的 norm 张量
    randomize_tensor_normal(model->output,         rnd); // 使用随机正态分布对象随机化模型的 output 张量

    for (uint32_t i = 0; i < n_layer; ++i) { // 遍历每一层
// 获取模型的第i层
auto & layer = model->layers[i];
// 为注意力规范化层随机初始化张量
randomize_tensor_normal(layer.attention_norm, rnd);
// 为查询权重张量随机初始化
randomize_tensor_normal(layer.wq, rnd);
// 为键权重张量随机初始化
randomize_tensor_normal(layer.wk, rnd);
// 为值权重张量随机初始化
randomize_tensor_normal(layer.wv, rnd);
// 为输出权重张量随机初始化
randomize_tensor_normal(layer.wo, rnd);
// 为前馈神经网络规范化层随机初始化张量
randomize_tensor_normal(layer.ffn_norm, rnd);
// 为前馈神经网络第一层权重张量随机初始化
randomize_tensor_normal(layer.w1, rnd);
// 为前馈神经网络第二层权重张量随机初始化
randomize_tensor_normal(layer.w2, rnd);
// 为前馈神经网络第三层权重张量随机初始化
randomize_tensor_normal(layer.w3, rnd);
// 释放随机正态分布
free_random_normal_distribution(rnd);
// 构建训练图
static struct ggml_tensor * llama_build_train_graphs(
        struct my_llama_model * model,
// 定义函数参数，包括分配器、上下文、计算图等
struct ggml_allocr * alloc,
struct ggml_context * ctx,
struct ggml_cgraph * gf,
struct ggml_cgraph * gb,
struct ggml_cgraph * gb_tmp,
struct ggml_tensor ** logits,
struct ggml_tensor * tokens_input,
struct ggml_tensor * targets,
const int n_tokens,
const int n_batch,
const bool enable_flash_attn,
const bool enable_checkpointing) {

// 设置上下文的临时存储空间
ggml_set_scratch(ctx, { 0, 0, nullptr, });
// 初始化过去的时间步数为0
const int n_past = 0;
// 设置N为tokens的数量
const int N = n_tokens;
// 获取模型的超参数
const auto & hparams = model->hparams;
// 获取上下文的大小
const int n_ctx = hparams.n_ctx;
// 获取词汇表的大小
const int n_vocab = hparams.n_vocab;
// 获取嵌入层的维度
const int n_embd = hparams.n_embd;
    // 定义变量并赋值为hparams中对应的值
    const int n_layer    = hparams.n_layer;
    const int n_head     = hparams.n_head;
    const int n_rot      = hparams.n_rot;
    const int n_ff       = hparams.n_ff;
    const float f_norm_rms_eps  = hparams.f_norm_rms_eps;
    const float rope_freq_base  = hparams.rope_freq_base;
    const float rope_freq_scale = hparams.rope_freq_scale;

    // 定义一个lambda函数，用于设置张量的名称
    auto set_name = [](struct ggml_tensor * t, const char * n) {
        ggml_set_name(t, n); // 设置张量的名称
        if (t->grad) {
            ggml_format_name(t->grad, "%s->grad", n); // 如果存在梯度张量，设置梯度张量的名称
        }
    };

    // 创建一个1维张量KQ_pos，数据类型为GGML_TYPE_I32，大小为N
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, N);
    ggml_allocr_alloc(alloc, KQ_pos); // 分配内存空间
    if (!ggml_allocr_is_measure(alloc)) { // 如果内存分配成功
        int * data = (int *) KQ_pos->data; // 获取张量的数据
// 使用循环将 n_past + i 的值赋给 data 数组中的每个元素
for (int i = 0; i < N; ++i) {
    data[i] = n_past + i;
}

// 创建一个自定义函数来处理 rope 对象，避免参数过多
auto rope = [ctx, KQ_pos, n_rot, n_ctx, rope_freq_base, rope_freq_scale]
            (struct ggml_tensor * t) -> struct ggml_tensor * {
    // 不捕获这些变量，以消除警告
    const int rope_mode = 0;

    // 调用 ggml_rope_custom 函数处理 rope 对象
    return ggml_rope_custom(
        ctx, t, KQ_pos, n_rot, rope_mode, n_ctx, 0, rope_freq_base, rope_freq_scale, 0.0f, 1.0f, 0.0f, 0.0f
    );
};

// 设置 tokens_input 对象的名称为 "tokens_input"
set_name(tokens_input, "tokens_input");
// 设置 targets 对象的名称为 "targets"
set_name(targets,      "targets");

// 检查 tokens_input 对象的类型是否为 GGML_TYPE_I32，如果不是则抛出断言错误
GGML_ASSERT(tokens_input->type == GGML_TYPE_I32);
# 创建一个指向 ggml_tensor 结构的指针 t00，该结构用于表示张量，通过 ggml_reshape_1d 函数将 tokens_input 重塑为一维张量，N*n_batch 为张量的大小
# 为 t00 设置名称为 "t00"，并断言其形状为一维
struct ggml_tensor * t00 = ggml_reshape_1d(ctx, tokens_input, N*n_batch);  set_name(t00, "t00"); assert_shape_1d(t00, N*n_batch);

# 创建一个指向 ggml_tensor 结构的指针 t01，通过 ggml_get_rows 函数从 model->tok_embeddings 中获取 t00 中指定行的数据，n_embd 为行数，N*n_batch 为列数
# 为 t01 设置名称为 "t01"，并断言其形状为二维
struct ggml_tensor * t01 = ggml_get_rows(ctx, model->tok_embeddings, t00); set_name(t01, "t01"); assert_shape_2d(t01, n_embd, N*n_batch);

# 创建一个指向 ggml_tensor 结构的指针 cur，指向 t01
struct ggml_tensor * cur = t01;

# 创建一个存储 ggml_tensor 结构指针的向量 checkpoints，并将 tokens_input、targets、t00、t01 加入其中
std::vector<struct ggml_tensor *> checkpoints;
checkpoints.push_back(tokens_input);
checkpoints.push_back(targets);
checkpoints.push_back(t00);
checkpoints.push_back(t01);

# 创建一个指向 ggml_tensor 结构的指针 kv_scale，如果 enable_flash_attn 为假，则通过 ggml_new_f32 函数创建一个值为 1.0f/sqrtf(float(n_embd)/n_head) 的张量
if (!enable_flash_attn) {
    kv_scale = ggml_new_f32(ctx, 1.0f/sqrtf(float(n_embd)/n_head));
}

# 循环执行 n_layer 次
for (int il = 0; il < n_layer; ++il) {
    # 获取 model->layers 中的第 il 层
    struct my_llama_layer & layer = model->layers[il];
    # 创建一个指向 ggml_tensor 结构的指针 t02，通过 ggml_rms_norm 函数对 cur 进行均方根归一化处理，f_norm_rms_eps 为归一化的参数
    # 为 t02 设置名称为 "t02"，并断言其形状为二维
    struct ggml_tensor * t02 = ggml_rms_norm(ctx, cur, f_norm_rms_eps); set_name(t02, "t02"); assert_shape_2d(t02, n_embd, N*n_batch);
    # 创建一个指向 ggml_tensor 结构的指针 t03，通过 ggml_repeat 函数对 layer.attention_norm 进行重复操作，t02 为重复的次数
    # 为 t03 设置名称为 "t03"，并断言其形状为二维
    struct ggml_tensor * t03 = ggml_repeat(ctx, layer.attention_norm, t02); set_name(t03, "t03"); assert_shape_2d(t03, n_embd, N*n_batch);
# 计算 t04 = t03 * t02，得到新的张量 t04
# 设置 t04 的名称为 "t04"
# 断言 t04 的形状为 2 维，形状为 (n_embd, N*n_batch)
struct ggml_tensor * t04 = ggml_mul          (ctx, t03, t02);                               
set_name(t04, "t04");     
assert_shape_2d(t04, n_embd, N*n_batch);

# 计算 t05 = layer.wq * t04，得到新的张量 t05
# 设置 t05 的名称为 "t05"
# 断言 t05 的形状为 2 维，形状为 (n_embd, N*n_batch)
struct ggml_tensor * t05 = ggml_mul_mat      (ctx, layer.wq, t04);                          
set_name(t05, "t05");     
assert_shape_2d(t05, n_embd, N*n_batch);

# 将 t05 重塑为 4 维张量 t06
# 设置 t06 的名称为 "t06"
# 断言 t06 的形状为 4 维，形状为 (n_embd/n_head, n_head, N, n_batch)
struct ggml_tensor * t06 = ggml_reshape_4d   (ctx, t05, n_embd/n_head, n_head, N, n_batch); 
set_name(t06, "t06");     
assert_shape_4d(t06, n_embd/n_head, n_head, N, n_batch);

# 对 t06 进行一些操作，得到新的张量 t07
# 设置 t07 的名称为 "t07"
# 断言 t07 的形状为 4 维，形状为 (n_embd/n_head, n_head, N, n_batch)
struct ggml_tensor * t07 = rope              (t06);                                         
set_name(t07, "t07");     
assert_shape_4d(t07, n_embd/n_head, n_head, N, n_batch);

# 计算 t08 = layer.wk * t04，得到新的张量 t08
# 设置 t08 的名称为 "t08"
# 断言 t08 的形状为 2 维，形状为 (n_embd, N*n_batch)
struct ggml_tensor * t08 = ggml_mul_mat      (ctx, layer.wk, t04);                          
set_name(t08, "t08");     
assert_shape_2d(t08, n_embd, N*n_batch);

# 将 t08 重塑为 4 维张量 t09
# 设置 t09 的名称为 "t09"
# 断言 t09 的形状为 4 维，形状为 (n_embd/n_head, n_head, N, n_batch)
struct ggml_tensor * t09 = ggml_reshape_4d   (ctx, t08, n_embd/n_head, n_head, N, n_batch); 
set_name(t09, "t09");     
assert_shape_4d(t09, n_embd/n_head, n_head, N, n_batch);

# 对 t09 进行一些操作，得到新的张量 t10
# 设置 t10 的名称为 "t10"
# 断言 t10 的形状为 4 维，形状为 (n_embd/n_head, n_head, N, n_batch)
struct ggml_tensor * t10 = rope              (t09);                                         
set_name(t10, "t10");     
assert_shape_4d(t10, n_embd/n_head, n_head, N, n_batch);

# 计算 t11 = t04 * layer.wv，得到新的张量 t11
# 设置 t11 的名称为 "t11"
# 断言 t11 的形状为 2 维，形状为 (N*n_batch, n_embd)
struct ggml_tensor * t11 = ggml_mul_mat      (ctx, t04, layer.wv);                          
set_name(t11, "t11");     
assert_shape_2d(t11, N*n_batch, n_embd);

# 将 t11 重塑为 4 维张量 t12
# 设置 t12 的名称为 "t12"
# 断言 t12 的形状为 4 维，形状为 (N, n_batch, n_embd/n_head, n_head)
struct ggml_tensor * t12 = ggml_reshape_4d   (ctx, t11, N, n_batch, n_embd/n_head, n_head); 
set_name(t12, "t12");     
assert_shape_4d(t12, N, n_batch, n_embd/n_head, n_head);

# 对 t07 进行维度置换，得到新的张量 t13
# 设置 t13 的名称为 "t13"
# 断言 t13 的形状为 4 维，形状为 (n_embd/n_head, N, n_head, n_batch)
struct ggml_tensor * t13 = ggml_permute      (ctx, t07, 0, 2, 1, 3);                        
set_name(t13, "t13");     
assert_shape_4d(t13, n_embd/n_head, N, n_head, n_batch);

# 对 t10 进行维度置换，得到新的张量 t14
# 设置 t14 的名称为 "t14"
# 断言 t14 的形状为 4 维，形状为 (n_embd/n_head, N, n_head, n_batch)
struct ggml_tensor * t14 = ggml_permute      (ctx, t10, 0, 2, 1, 3);                        
set_name(t14, "t14");     
assert_shape_4d(t14, n_embd/n_head, N, n_head, n_batch);

# 对 t12 进行维度置换，得到新的张量 t15
# 设置 t15 的名称为 "t15"
# 断言 t15 的形状为 4 维，形状为 (N, n_embd/n_head, n_head, n_batch)
struct ggml_tensor * t15 = ggml_permute      (ctx, t12, 0, 3, 1, 2);                        
set_name(t15, "t15");     
assert_shape_4d(t15, N, n_embd/n_head, n_head, n_batch);

# 定义 t16 张量
struct ggml_tensor * t16;

# 如果启用了 flash attention，则计算 t16 = ggml_flash_attn(ctx, t13, t14, t15, true)
# 设置 t16 的名称为 "t16"
# 断言 t16 的形状为 4 维，形状为 (n_embd/n_head, N, n_head, n_batch)
if (enable_flash_attn) {
    t16 = ggml_flash_attn(ctx, t13, t14, t15, true);                                        
    set_name(t16, "t16");     
    assert_shape_4d(t16, n_embd/n_head, N, n_head, n_batch);
} else {
    # 如果未启用 flash attention，则进行一系列操作得到 t16
    # 计算 t16_0 = t14 * t13
    # 设置 t16_0 的名称为 "t16_0"
    # 断言 t16_0 的形状为 4 维，形状为 (N, N, n_head, n_batch)
    struct ggml_tensor * t16_0 = ggml_mul_mat              (ctx, t14, t13);                 
    set_name(t16_0, "t16_0"); 
    assert_shape_4d(t16_0, N, N, n_head, n_batch);
    
    # 对 t16_0 进行缩放操作，得到新的张量 t16_1
    # 设置 t16_1 的名称为 "t16_1"
    # 断言 t16_1 的形状为 4 维，形状为 (N, N, n_head, n_batch)
    struct ggml_tensor * t16_1 = ggml_scale_inplace        (ctx, t16_0, kv_scale);          
    set_name(t16_1, "t16_1"); 
    assert_shape_4d(t16_1, N, N, n_head, n_batch);
    
    # 对 t16_1 进行对角线掩码操作，得到新的张量 t16_2
    # 设置 t16_2 的名称为 "t16_2"
    # 断言 t16_2 的形状为 4 维，形状为 (N, N, n_head, n_batch)
    struct ggml_tensor * t16_2 = ggml_diag_mask_inf_inplace(ctx, t16_1, n_past);            
    set_name(t16_2, "t16_2"); 
    assert_shape_4d(t16_2, N, N, n_head, n_batch);
    
    # 对 t16_2 进行 softmax 操作，得到新的张量 t16_3
    # 设置 t16_3 的名称为 "t16_3"
    # 断言 t16_3 的形状为 4 维，形状为 (N, N, n_head, n_batch)
    struct ggml_tensor * t16_3 = ggml_soft_max_inplace     (ctx, t16_2);                    
    set_name(t16_3, "t16_3"); 
    assert_shape_4d(t16_3, N, N, n_head, n_batch);
# 计算 t16，并进行命名和形状断言
t16 = ggml_mul_mat(ctx, t15, t16_3); 
set_name(t16, "t16");     
assert_shape_4d(t16, n_embd/n_head, N, n_head, n_batch);

# 对 t16 进行转置操作，并进行命名和形状断言
struct ggml_tensor * t17 = ggml_permute(ctx, t16, 0, 2, 1, 3);                        
set_name(t17, "t17");     
assert_shape_4d(t17, n_embd/n_head, n_head, N, n_batch);

# 对 t17 进行连续操作，并进行命名和形状断言
struct ggml_tensor * t18 = ggml_cont(ctx, t17);                                    
set_name(t18, "t18");     
assert_shape_4d(t18, n_embd/n_head, n_head, N, n_batch);

# 对 t18 进行二维重塑操作，并进行命名和形状断言
struct ggml_tensor * t19 = ggml_reshape_2d(ctx, t18, n_embd, N*n_batch);                 
set_name(t19, "t19");     
assert_shape_2d(t19, n_embd, N*n_batch);

# 对 t19 进行矩阵乘法操作，并进行命名和形状断言
struct ggml_tensor * t20 = ggml_mul_mat(ctx, layer.wo, t19);                          
set_name(t20, "t20");     
assert_shape_2d(t20, n_embd, N*n_batch);

# 对 t20 和 cur 进行加法操作，并进行命名和形状断言
struct ggml_tensor * t21 = ggml_add(ctx, t20, cur);                               
set_name(t21, "t21");     
assert_shape_2d(t21, n_embd, N*n_batch);

# 对 t21 进行均方根归一化操作，并进行命名和形状断言
struct ggml_tensor * t22 = ggml_rms_norm(ctx, t21, f_norm_rms_eps);                    
set_name(t22, "t22");     
assert_shape_2d(t22, n_embd, N*n_batch);

# 对 t22 进行重复操作，并进行命名和形状断言
struct ggml_tensor * t23 = ggml_repeat(ctx, layer.ffn_norm, t22);                    
set_name(t23, "t23");     
assert_shape_2d(t23, n_embd, N*n_batch);

# 对 t23 和 t22 进行乘法操作，并进行命名和形状断言
struct ggml_tensor * t24 = ggml_mul(ctx, t23, t22);                               
set_name(t24, "t24");     
assert_shape_2d(t24, n_embd, N*n_batch);

# 对 t24 和 layer.w3 进行矩阵乘法操作，并进行命名和形状断言
struct ggml_tensor * t25 = ggml_mul_mat(ctx, layer.w3, t24);                          
set_name(t25, "t25");     
assert_shape_2d(t25, n_ff, N*n_batch);

# 对 t24 和 layer.w1 进行矩阵乘法操作，并进行命名和形状断言
struct ggml_tensor * t26 = ggml_mul_mat(ctx, layer.w1, t24);                          
set_name(t26, "t26");     
assert_shape_2d(t26, n_ff, N*n_batch);

# 对 t26 进行 SiLU 激活函数操作，并进行命名和形状断言
struct ggml_tensor * t27 = ggml_silu(ctx, t26);                                    
set_name(t27, "t27");     
assert_shape_2d(t27, n_ff, N*n_batch);

# 对 t27 和 t25 进行乘法操作，并进行命名和形状断言
struct ggml_tensor * t28 = ggml_mul(ctx, t27, t25);                               
set_name(t28, "t28");     
assert_shape_2d(t28, n_ff, N*n_batch);

# 对 t28 和 layer.w2 进行矩阵乘法操作，并进行命名和形状断言
struct ggml_tensor * t29 = ggml_mul_mat(ctx, layer.w2, t28);                          
set_name(t29, "t29");     
assert_shape_2d(t29, n_embd, N*n_batch);

# 对 t29 和 t21 进行加法操作，并进行命名和形状断言
struct ggml_tensor * t30 = ggml_add(ctx, t29, t21);                               
set_name(t30, "t30");     
assert_shape_2d(t30, n_embd, N*n_batch);

# 将 t30 赋值给 cur，并将 cur 加入 checkpoints
cur = t30;
checkpoints.push_back(cur);

# 对 cur 进行均方根归一化操作，并进行命名和形状断言
struct ggml_tensor * t31 = ggml_rms_norm(ctx, cur, f_norm_rms_eps);                 
set_name(t31, "t31");     
assert_shape_2d(t31, n_embd, N*n_batch);
# 创建一个名为 t32 的张量，其值为 model->norm 重复 N*n_batch 次的结果
# 设置张量的名称为 "t32"，并断言其形状为 2 维，大小为 n_embd * N*n_batch
struct ggml_tensor * t32   = ggml_repeat            (ctx, model->norm, t31);                    set_name(t32, "t32");     assert_shape_2d(t32, n_embd, N*n_batch);

# 创建一个名为 t33 的张量，其值为 t32 与 t31 的乘积
# 设置张量的名称为 "t33"，并断言其形状为 2 维，大小为 n_embd * N*n_batch
struct ggml_tensor * t33   = ggml_mul               (ctx, t32, t31);                            set_name(t33, "t33");     assert_shape_2d(t33, n_embd, N*n_batch);

# 创建一个名为 t34 的张量，其值为 model->output 与 t33 的矩阵乘积
# 设置张量的名称为 "t34"，并断言其形状为 2 维，大小为 n_vocab * N*n_batch
struct ggml_tensor * t34   = ggml_mul_mat           (ctx, model->output, t33);                  set_name(t34, "t34");     assert_shape_2d(t34, n_vocab, N*n_batch);

# 创建一个名为 t35 的张量，将 t34 重塑为 3 维张量，形状为 n_vocab * N * n_batch
# 设置张量的名称为 "t35"，并断言其形状为 3 维，大小为 n_vocab * N * n_batch
struct ggml_tensor * t35   = ggml_reshape_3d        (ctx, t34, n_vocab, N, n_batch);            set_name(t35, "t35");     assert_shape_3d(t35, n_vocab, N, n_batch);

# 创建一个名为 t36 的张量，计算 t35 与 targets 之间的交叉熵损失
# 设置张量的名称为 "t36"，并断言其形状为 1 维，大小为 1
struct ggml_tensor * t36   = ggml_cross_entropy_loss(ctx, t35, targets);                        set_name(t36, "t36");     assert_shape_1d(t36, 1);

# 将 t31 到 t36 加入检查点列表
checkpoints.push_back(t31);
checkpoints.push_back(t32);
checkpoints.push_back(t33);
checkpoints.push_back(t34);
checkpoints.push_back(t35);
checkpoints.push_back(t36);

# 构建前向传播图
ggml_build_forward_expand(gf, t36);

# 如果启用了检查点功能，则构建反向传播梯度检查点
# 否则，将前向传播图复制到反向传播图，并构建反向传播图
if (enable_checkpointing) {
    ggml_build_backward_gradient_checkpointing(ctx, gf, gb, gb_tmp, checkpoints.data(), (int) checkpoints.size());
} else {
    ggml_graph_cpy(gf, gb);
    ggml_build_backward_expand(ctx, gf, gb, true);
    }

    if (alloc) {
        // 确保一些张量不会被重新分配，通过插入依赖于它们的新临时节点
        int n_leafs_before = gb->n_leafs;  // 记录操作前叶子节点的数量
        int n_nodes_before = gb->n_nodes;  // 记录操作前节点的数量
        struct ggml_tensor * one = ggml_new_f32(ctx, 1.0f);  // 创建一个数值为1.0的张量
        // 输出张量
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t35, one));  // 构建前向传播，对 t35 进行就地缩放
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t36, one));  // 构建前向传播，对 t36 进行就地缩放
        // 输入梯度
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t36->grad, one));  // 构建前向传播，对 t36 的梯度进行就地缩放
        // KQ_pos
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, KQ_pos, one));  // 构建前向传播，对 KQ_pos 进行就地缩放
        GGML_ASSERT(t36->grad->data == NULL && t36->grad->view_src == NULL);  // 断言 t36 的梯度数据和视图源为空

        ggml_allocr_alloc(alloc, t36->grad);  // 分配 t36 的梯度

        // 在一个块中分配检查点，以减少内存碎片化
        // 注意：它们将以相反的顺序被释放
// 遍历检查点列表，如果数据和视图源都为空，则调用ggml_allocr_alloc函数进行分配
for (int i = 0; i < (int) checkpoints.size(); ++i) {
    if (checkpoints[i]->data == NULL && checkpoints[i]->view_src == NULL) {
        ggml_allocr_alloc(alloc, checkpoints[i]);
    }
}

// 调用ggml_allocr_alloc_graph函数为图形分配内存
ggml_allocr_alloc_graph(alloc, gb);

// 移除额外的节点和叶子
for (int i = n_leafs_before; i < gb->n_leafs; ++i) {
    gb->leafs[i] = NULL;
}
for (int i = n_nodes_before; i < gb->n_nodes; ++i) {
    gb->nodes[i] = NULL;
}
gb->n_leafs = n_leafs_before;
gb->n_nodes = n_nodes_before;
    }

    *logits = t35;  // 将指针变量logits指向t35所指向的内存空间
    return t36;  // 返回变量t36的值
}

#define GGUF_GET_KEY(ctx, dst, func, type, req, key) \  // 定义宏GGUF_GET_KEY，用于获取指定类型的键值对
do { \  // 宏定义的开始
    const std::string skey(key);  // 创建一个字符串skey，其值为参数key
    const int kid = gguf_find_key(ctx, skey.c_str());  // 在上下文ctx中查找键值为skey的键，并将其索引赋值给kid
    if (kid >= 0) {  // 如果找到了键
        enum gguf_type ktype = gguf_get_kv_type(ctx, kid);  // 获取键的类型
        if (ktype != (type)) {  // 如果键的类型不是指定的type
            die_fmt("key %s has wrong type: %s", skey.c_str(), gguf_type_name(ktype));  // 输出错误信息
        } 
        (dst) = func(ctx, kid);  // 调用func函数，将键值对赋值给dst
    } else if (req) {  // 如果未找到键且req为真
        die_fmt("key not found in model: %s", skey.c_str());  // 输出错误信息
    } 
} while (0)  // 宏定义的结束
// 加载 LLAMA 模型到 GGUF 上下文中
static void load_llama_model_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model) {
    // 注意：gguf_context 必须使用 f_ggml_ctx 进行初始化，并且 no_alloc=false，否则无法读取张量数据
    std::string arch; // 定义一个字符串变量 arch

    std::vector<char> keybuf; // 定义一个字符向量 keybuf
    keybuf.resize(512); // 调整 keybuf 的大小为 512
    auto kv = [&arch, &keybuf](const char * key) -> const char * { // 定义一个 lambda 函数 kv，用于格式化 keybuf 中的键值
        snprintf(keybuf.data(), keybuf.size(), key, arch.c_str()); // 使用 snprintf 格式化 keybuf 中的键值
        return keybuf.data(); // 返回格式化后的键值
    };

    std::vector<char> tn_buf; // 定义一个字符向量 tn_buf
    tn_buf.resize(GGML_MAX_NAME); // 调整 tn_buf 的大小为 GGML_MAX_NAME
    auto tn = [&tn_buf](const char * key) -> const char * { // 定义一个 lambda 函数 tn，用于格式化 tn_buf 中的键值
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", key); // 使用 snprintf 格式化 tn_buf 中的键值
        return tn_buf.data(); // 返回格式化后的键值
    };
    auto tni = [&tn_buf](const char * key, int bid) -> const char * { // 定义一个 lambda 函数 tni，用于格式化 tn_buf 中的键值和 bid
        snprintf(tn_buf.data(), tn_buf.size(), key, bid); // 使用 snprintf 格式化 tn_buf 中的键值和 bid
// 将tn_buf的数据转换为字符串
std::string s = tn_buf.data();
// 将字符串格式化为"%s.weight"的形式，存储到tn_buf中
snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", s.c_str());
// 返回tn_buf中的数据
return tn_buf.data();
};

// 从fctx中获取arch的值，并将其转换为字符串类型
GGUF_GET_KEY(fctx, arch, gguf_get_val_str, GGUF_TYPE_STRING, true, LLM_KV_GENERAL_ARCHITECTURE);
// 断言arch的值为"llama"
GGML_ASSERT(arch == "llama");

// 从fctx中获取ftype_u的值，并将其转换为uint32_t类型
GGUF_GET_KEY(fctx, ftype_u, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_GENERAL_FILE_TYPE);
// 断言ftype_u的值为LLAMA_FTYPE_ALL_F32对应的枚举值

// 从fctx中获取model->hparams.n_ctx的值，并将其转换为uint32_t类型，如果不存在则设为可选
GGUF_GET_KEY(fctx, model->hparams.n_ctx,   gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_CONTEXT_LENGTH));

// 从fctx中获取model->hparams.n_embd的值，并将其转换为uint32_t类型
GGUF_GET_KEY(fctx, model->hparams.n_embd,  gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_EMBEDDING_LENGTH));
// 从fctx中获取model->hparams.n_ff的值，并将其转换为uint32_t类型
GGUF_GET_KEY(fctx, model->hparams.n_ff,    gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_FEED_FORWARD_LENGTH));
// 从fctx中获取model->hparams.n_head的值，并将其转换为uint32_t类型
GGUF_GET_KEY(fctx, model->hparams.n_head,  gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_ATTENTION_HEAD_COUNT));
// 从fctx中获取model->hparams.n_layer的值，并将其转换为uint32_t类型
GGUF_GET_KEY(fctx, model->hparams.n_layer, gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_BLOCK_COUNT));
    # 计算每个头的旋转数
    model->hparams.n_rot = model->hparams.n_embd / model->hparams.n_head;
    # 从配置文件中获取并设置模型的旋转数
    GGUF_GET_KEY(fctx, model->hparams.n_rot,   gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ROPE_DIMENSION_COUNT));

    # 设置绳索频率的比例为1.0
    float rope_freq_scale = 1.0f;
    # 从配置文件中获取并设置模型的层归一化的 RMS 误差
    GGUF_GET_KEY(fctx, model->hparams.f_norm_rms_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS));
    # 从配置文件中获取并设置模型的绳索基础频率
    GGUF_GET_KEY(fctx, model->hparams.rope_freq_base, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_FREQ_BASE));
    # 从配置文件中获取并设置绳索频率的比例
    GGUF_GET_KEY(fctx, rope_freq_scale, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_SCALE_LINEAR));
    # 如果绳索频率的比例不为1.0，则重新计算绳索频率的比例
    if (rope_freq_scale != 1.0f) {
        model->hparams.rope_freq_scale = 1.0f / rope_freq_scale;
    }

    # 初始化模型
    init_model(model);

    # 从上下文中复制指定名称的张量到模型中
    copy_tensor_by_name(model->tok_embeddings, f_ggml_ctx, tn(LLM_TENSOR_TOKEN_EMBD));
    copy_tensor_by_name(model->norm,           f_ggml_ctx, tn(LLM_TENSOR_OUTPUT_NORM));
    copy_tensor_by_name(model->output,         f_ggml_ctx, tn(LLM_TENSOR_OUTPUT));

    # 遍历模型的每一层
    for (uint32_t i = 0; i < model->hparams.n_layer; ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];
// 通过名称复制张量数据到指定位置
copy_tensor_by_name(layer.attention_norm, f_ggml_ctx, tni(LLM_TENSOR_ATTN_NORM, i));
copy_tensor_by_name(layer.wq,             f_ggml_ctx, tni(LLM_TENSOR_ATTN_Q, i));
copy_tensor_by_name(layer.wk,             f_ggml_ctx, tni(LLM_TENSOR_ATTN_K, i));
copy_tensor_by_name(layer.wv,             f_ggml_ctx, tni(LLM_TENSOR_ATTN_V, i));
copy_tensor_by_name(layer.wo,             f_ggml_ctx, tni(LLM_TENSOR_ATTN_OUT, i));
copy_tensor_by_name(layer.ffn_norm,       f_ggml_ctx, tni(LLM_TENSOR_FFN_NORM, i));
copy_tensor_by_name(layer.w1,             f_ggml_ctx, tni(LLM_TENSOR_FFN_GATE, i));
copy_tensor_by_name(layer.w2,             f_ggml_ctx, tni(LLM_TENSOR_FFN_DOWN, i));
copy_tensor_by_name(layer.w3,             f_ggml_ctx, tni(LLM_TENSOR_FFN_UP, i));

// 保存LLAMA模型到文件
static void save_llama_model_gguf(struct gguf_context * fctx, const char * fn_vocab_model, struct my_llama_model * model) {
    // 定义LLAMA模型的架构和数据类型
    const char * arch = "llama";
    enum llama_ftype ftype = LLAMA_FTYPE_ALL_F32;

    // 创建缓冲区用于存储键值
    std::vector<char> keybuf;
    keybuf.resize(512);
    // 定义键值对的生成函数
    auto kv = [arch, &keybuf](const char * key) -> const char * {
        // 格式化键值对
        snprintf(keybuf.data(), keybuf.size(), key, arch);
    // 返回 keybuf 的数据
    return keybuf.data();
    };

    // 设置架构
    gguf_set_val_str(fctx, LLM_KV_GENERAL_ARCHITECTURE, arch);
    gguf_set_val_u32(fctx, LLM_KV_GENERAL_FILE_TYPE, ftype);

    // 设置超参数
    gguf_set_val_u32(fctx, kv(LLM_KV_CONTEXT_LENGTH),              model->hparams.n_ctx                  );
    gguf_set_val_u32(fctx, kv(LLM_KV_EMBEDDING_LENGTH),            model->hparams.n_embd                 );
    gguf_set_val_u32(fctx, kv(LLM_KV_FEED_FORWARD_LENGTH),         model->hparams.n_ff                   );
    gguf_set_val_u32(fctx, kv(LLM_KV_ATTENTION_HEAD_COUNT),        model->hparams.n_head                 );
    gguf_set_val_u32(fctx, kv(LLM_KV_BLOCK_COUNT),                 model->hparams.n_layer                );
    gguf_set_val_u32(fctx, kv(LLM_KV_ROPE_DIMENSION_COUNT),        model->hparams.n_rot                  );

    gguf_set_val_f32(fctx, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS), model->hparams.f_norm_rms_eps         );
    gguf_set_val_f32(fctx, kv(LLM_KV_ROPE_FREQ_BASE),              model->hparams.rope_freq_base         ); // TODO load in llama.cpp
    gguf_set_val_f32(fctx, kv(LLM_KV_ROPE_SCALE_LINEAR),           1.0f / model->hparams.rope_freq_scale );

    // 通过从 vocab_model gguf 文件复制来设置词汇表
    {
        // 初始化参数结构体，设置不进行内存分配，上下文为空
        struct gguf_init_params params = {
            /*.no_alloc = */ false,  // 不进行内存分配
            /*.ctx      = */ NULL,   // 上下文为空
        };
        // 从文件中初始化上下文对象
        struct gguf_context * vctx = gguf_init_from_file(fn_vocab_model, params);

        // 查找词汇模型中分词器列表的索引
        const int token_idx = gguf_find_key(vctx, kv(LLM_KV_TOKENIZER_LIST));
        // 如果找不到，输出错误信息并退出程序
        if (token_idx == -1) {
            die("cannot find tokenizer vocab in model file");
        }
        // 获取词汇量
        const uint32_t n_vocab = gguf_get_arr_n(vctx, token_idx);

        // 查找词汇模型中分词器得分的索引
        const int score_idx = gguf_find_key(vctx, kv(LLM_KV_TOKENIZER_SCORES));
        // 如果找不到，输出错误信息并退出程序
        if (score_idx == -1) {
            die("cannot find tokenizer scores in model file");
        }

        // 获取分词器得分数据
        const float * scores = (const float * ) gguf_get_arr_data(vctx, score_idx);
    }
// 查找指定键在上下文中的索引
const int toktype_idx = gguf_find_key(vctx, kv(LLM_KV_TOKENIZER_TOKEN_TYPE));
// 如果找不到指定键的索引，抛出错误
if (toktype_idx == -1) {
    die("cannot find token type list in GGUF file");
}

// 获取指定键对应的数组数据
const int * toktypes = (const int * ) gguf_get_arr_data(vctx, toktype_idx);

// 获取指定键对应的字符串值
std::string tokenizer_name;
GGUF_GET_KEY(vctx, tokenizer_name, gguf_get_val_str, GGUF_TYPE_STRING, true, kv(LLM_KV_TOKENIZER_MODEL));

// 设置指定键对应的字符串值
gguf_set_val_str(fctx, kv(LLM_KV_TOKENIZER_MODEL), tokenizer_name.c_str());
// 设置指定键对应的数组数据
gguf_set_arr_data(fctx, kv(LLM_KV_TOKENIZER_SCORES), GGUF_TYPE_FLOAT32, scores, n_vocab);
gguf_set_arr_data(fctx, kv(LLM_KV_TOKENIZER_TOKEN_TYPE), GGUF_TYPE_INT32, toktypes, n_vocab);

// 初始化特殊标记的 ID
int32_t special_bos_id = 1;
int32_t special_eos_id = 2;
int32_t special_unk_id = 0;
int32_t special_sep_id = -1;
int32_t special_pad_id = -1;

// 如果 tokenizer_name 为 "llama"，执行以下操作
if (tokenizer_name == "llama") {
// 设置默认特殊标记的ID
special_bos_id = 1;
special_eos_id = 2;
special_unk_id = 0;
special_sep_id = -1;
special_pad_id = -1;
// 如果使用的是"gpt2"分词器
} else if (tokenizer_name == "gpt2") {
    // 读取并复制bpe合并
    const int merges_keyidx = gguf_find_key(vctx, kv(LLM_KV_TOKENIZER_MERGES));
    // 如果在模型文件中找不到分词器合并信息，则报错
    if (merges_keyidx == -1) {
        die("cannot find tokenizer merges in model file");
    }
    // 获取合并数量
    const int n_merges = gguf_get_arr_n(vctx, merges_keyidx);
    // 创建存储合并信息的vector
    std::vector<const char*> merges;
    merges.resize(n_merges);
    // 遍历合并信息，将其存入vector
    for (int i = 0; i < n_merges; i++) {
        merges[i] = gguf_get_arr_str(vctx, merges_keyidx, i);
    }
// 将 merges 数据设置到 fctx 中的 LLM_KV_TOKENIZER_MERGES 键对应的值中
gguf_set_arr_str(fctx, kv(LLM_KV_TOKENIZER_MERGES), merges.data(), n_merges);

// 设置默认的特殊标记
special_bos_id = 11;
special_eos_id = 11;
special_unk_id = -1;
special_sep_id = -1;
special_pad_id = -1;

// 如果存在未知的分词器，则输出错误信息并使用默认的分词器 llama
} else {
    fprintf(stderr, "%s: unknown tokenizer: '%s'", __func__, tokenizer_name.c_str());
    fprintf(stderr, "%s: using default tokenizer: 'llama'", __func__);
}

// 创建 tokens 数组，大小为 n_vocab
std::vector<const char*> tokens;
tokens.resize(n_vocab);

// 遍历 n_vocab，将 token_idx 中的数据设置到 tokens 数组中
for (uint32_t i = 0; i < n_vocab; i++) {
    tokens[i] = gguf_get_arr_str(vctx, token_idx, i);
}

// 将 tokens 数据设置到 fctx 中的 LLM_KV_TOKENIZER_LIST 键对应的值中
gguf_set_arr_str(fctx, kv(LLM_KV_TOKENIZER_LIST), tokens.data(), n_vocab);
# 从 vctx 中获取特殊标记的键值对，并将结果存储到 special_bos_id 中
GGUF_GET_KEY(vctx, special_bos_id, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_TOKENIZER_BOS_ID));
# 从 vctx 中获取特殊标记的键值对，并将结果存储到 special_eos_id 中
GGUF_GET_KEY(vctx, special_eos_id, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_TOKENIZER_EOS_ID));
# 从 vctx 中获取特殊标记的键值对，并将结果存储到 special_unk_id 中
GGUF_GET_KEY(vctx, special_unk_id, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_TOKENIZER_UNK_ID));
# 从 vctx 中获取特殊标记的键值对，并将结果存储到 special_sep_id 中
GGUF_GET_KEY(vctx, special_sep_id, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_TOKENIZER_SEP_ID));
# 从 vctx 中获取特殊标记的键值对，并将结果存储到 special_pad_id 中
GGUF_GET_KEY(vctx, special_pad_id, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_TOKENIZER_PAD_ID));

# 将特殊标记的键值对存储到 fctx 中
gguf_set_val_u32(fctx, kv(LLM_KV_TOKENIZER_BOS_ID), special_bos_id);
gguf_set_val_u32(fctx, kv(LLM_KV_TOKENIZER_EOS_ID), special_eos_id);
gguf_set_val_u32(fctx, kv(LLM_KV_TOKENIZER_UNK_ID), special_unk_id);
gguf_set_val_u32(fctx, kv(LLM_KV_TOKENIZER_SEP_ID), special_sep_id);
gguf_set_val_u32(fctx, kv(LLM_KV_TOKENIZER_PAD_ID), special_pad_id);

# 释放 vctx 占用的内存
gguf_free(vctx)

# 添加模型的张量到 fctx 中
gguf_add_tensor(fctx, model->tok_embeddings);
gguf_add_tensor(fctx, model->norm);
gguf_add_tensor(fctx, model->output);
# 遍历模型的层数，将每一层的张量添加到 fctx 中
for (uint32_t i = 0; i < model->hparams.n_layer; ++i) {
// 获取模型的第i层
auto & layer = model->layers[i];

// 将注意力规范化层的张量添加到文件上下文中
gguf_add_tensor(fctx, layer.attention_norm);
// 将wq张量添加到文件上下文中
gguf_add_tensor(fctx, layer.wq);
// 将wk张量添加到文件上下文中
gguf_add_tensor(fctx, layer.wk);
// 将wv张量添加到文件上下文中
gguf_add_tensor(fctx, layer.wv);
// 将wo张量添加到文件上下文中
gguf_add_tensor(fctx, layer.wo);
// 将ffn_norm张量添加到文件上下文中
gguf_add_tensor(fctx, layer.ffn_norm);
// 将w1张量添加到文件上下文中
gguf_add_tensor(fctx, layer.w1);
// 将w2张量添加到文件上下文中
gguf_add_tensor(fctx, layer.w2);
// 将w3张量添加到文件上下文中
gguf_add_tensor(fctx, layer.w3);
}

// 保存LLAMA模型文件
static void save_llama_model_file(const char * filename, const char * fn_vocab_model, struct my_llama_model * model) {
    // 打印保存信息
    printf("%s: saving to %s\n", __func__, filename);
    // 初始化空的文件上下文
    struct gguf_context * fctx = gguf_init_empty();

    // 保存LLAMA模型到文件上下文中
    save_llama_model_gguf(fctx, fn_vocab_model, model);
// 写入文件
const bool only_meta = false; // 定义布尔变量 only_meta，并赋值为 false
gguf_write_to_file(fctx, filename, only_meta); // 调用 gguf_write_to_file 函数，将 fctx、filename 和 only_meta 作为参数传入
gguf_free(fctx); // 释放 fctx 占用的内存空间

// 从 GGUF 中加载检查点
static void load_checkpoint_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model, struct train_state * train) {
    load_llama_model_gguf(fctx, f_ggml_ctx, model); // 调用 load_llama_model_gguf 函数，将 fctx、f_ggml_ctx 和 model 作为参数传入
    if (load_train_state_gguf(fctx, f_ggml_ctx, train)) { // 如果 load_train_state_gguf 函数返回 true
        std::string train_type = LLM_KV_TRAINING_TYPE_TRAIN_MODEL; // 定义字符串变量 train_type，并赋值为 LLM_KV_TRAINING_TYPE_TRAIN_MODEL
        GGUF_GET_KEY(fctx, train_type, gguf_get_val_str, GGUF_TYPE_STRING, false, LLM_KV_TRAINING_TYPE); // 调用 GGUF_GET_KEY 函数，将 fctx、train_type、gguf_get_val_str、GGUF_TYPE_STRING、false 和 LLM_KV_TRAINING_TYPE 作为参数传入
        GGML_ASSERT(train_type == LLM_KV_TRAINING_TYPE_TRAIN_MODEL); // 断言 train_type 的值等于 LLM_KV_TRAINING_TYPE_TRAIN_MODEL
    } else {
        printf("%s: loaded llama model as checkpoint\n", __func__); // 打印加载羊驼模型作为检查点的消息
    }
}

// 将检查点保存到 GGUF
static void save_checkpoint_gguf(struct gguf_context * fctx, const char * fn_vocab_model, struct my_llama_model * model, struct train_state * train) {
    gguf_set_val_str(fctx, LLM_KV_TRAINING_TYPE, LLM_KV_TRAINING_TYPE_TRAIN_MODEL); // 调用 gguf_set_val_str 函数，将 fctx、LLM_KV_TRAINING_TYPE 和 LLM_KV_TRAINING_TYPE_TRAIN_MODEL 作为参数传入
// 保存羊驼模型和词汇模型到文件
save_llama_model_gguf(fctx, fn_vocab_model, model);
// 保存训练状态到文件
save_train_state_gguf(fctx, train);
}

// 加载检查点文件
static bool load_checkpoint_file(const char * filename, struct my_llama_model * model, struct train_state * train) {
    // 创建 GGML 上下文
    struct ggml_context * f_ggml_ctx;
    // 初始化 GGUF 参数
    struct gguf_init_params params;
    params.no_alloc = false;
    params.ctx = &f_ggml_ctx;
    // 从文件中初始化 GGUF 上下文
    struct gguf_context * fctx = gguf_init_from_file(filename, params);
    // 如果初始化失败，返回 false
    if (fctx == NULL) {
        return false;
    }

    // 加载检查点文件
    load_checkpoint_gguf(fctx, f_ggml_ctx, model, train);

    return true;
}

// 保存检查点文件
static void save_checkpoint_file(const char * filename, const char * fn_vocab_model, struct my_llama_model * model, struct train_state * train) {
    # 打印保存文件的信息，包括函数名和文件名
    printf("%s: saving to %s\n", __func__, filename);
    # 初始化一个空的 gguf_context 结构体
    struct gguf_context * fctx = gguf_init_empty();

    # 保存训练检查点到 gguf 文件中
    save_checkpoint_gguf(fctx, fn_vocab_model, model, train);

    # 写入文件
    const bool only_meta = false;
    gguf_write_to_file(fctx, filename, only_meta);
    # 释放 gguf_context 结构体
    gguf_free(fctx);
}

# 定义训练参数结构体
struct train_params {
    struct train_params_common common;

    # 词汇模型文件名
    const char * fn_vocab_model;
    # 输出模型文件名
    const char * fn_model_out;

    # 是否只写入模型
    bool only_write_model;

    # 上下文数
    int n_ctx;
// 定义整型变量 n_embd, n_head, n_layer, n_ff
int n_embd;
int n_head;
int n_layer;
int n_ff;

// 定义浮点型变量 f_norm_rms_eps, rope_freq_base, rope_freq_scale
float f_norm_rms_eps;
float rope_freq_base;
float rope_freq_scale;
};

// 定义静态结构体函数 get_default_train_params，返回结构体 train_params
static struct train_params get_default_train_params() {
    // 声明结构体 train_params 变量 params，并初始化为默认值
    struct train_params params;
    params.common = get_default_train_params_common();
    params.fn_vocab_model    = "ggml-vic7b-uncensored-q4_0.bin";
    params.fn_model_out      = "ggml-checkpoint-f32.bin";

    // 初始化 only_write_model 为 false
    params.only_write_model = false;

    // 初始化 n_ctx 为 128, n_embd 为 256
    params.n_ctx      =  128;
    params.n_embd     =  256;
    # 设置参数 params 的 n_head 值为 8
    params.n_head     =    8;
    # 设置参数 params 的 n_layer 值为 16
    params.n_layer    =   16;
    # 设置参数 params 的 n_ff 值为 768
    params.n_ff       =  768;

    # 设置参数 params 的 f_norm_rms_eps 值为 1e-5f
    params.f_norm_rms_eps  = 1e-5f;
    # 设置参数 params 的 rope_freq_base 值为 10000.0f
    params.rope_freq_base  = 10000.0f;
    # 设置参数 params 的 rope_freq_scale 值为 1.0f
    params.rope_freq_scale = 1.0f;

    # 返回参数 params
    return params;
}

# 打印训练使用说明
static void train_print_usage(int argc, char ** argv, const struct train_params * params) {
    # 打印使用说明
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help                 show this help message and exit\n");

    # 打印模型路径的默认值
    fprintf(stderr, "  --vocab-model FNAME        model path from which to load vocab (default '%s')\n", params->fn_vocab_model);
    # 打印保存模型的默认路径
    fprintf(stderr, "  --model-out FNAME          path to save ggml model (default '%s')\n", params->fn_model_out);
    # 打印只保存模型而不进行训练的选项说明
    fprintf(stderr, "  --only-write-model         only save llama model, don't do any training. use this if you only want to convert a checkpoint to a model.\n");
    // 打印参数的默认值和说明
    fprintf(stderr, "  --embd N                   Embedding size used for new models (default %d)\n", params->n_embd);
    fprintf(stderr, "  --ff N                     Feedforward size used for new models. (default %d)\n", params->n_ff);
    fprintf(stderr, "  --head N                   Number of heads for new models (default %d)\n", params->n_head);
    fprintf(stderr, "  --layer N                  Number of layers for new models (default %d)\n", params->n_layer);
    fprintf(stderr, "  --norm-rms-eps F           RMS-Norm epsilon value (default %f)\n", params->f_norm_rms_eps);
    fprintf(stderr, "  --rope-freq-base F         Frequency base for ROPE (default %f)\n", params->rope_freq_base);
    fprintf(stderr, "  --rope-freq-scale F        Frequency scale for ROPE (default %f)\n", params->rope_freq_scale);

    // 打印通用训练用法
    print_common_train_usage(argc, argv, &params->common);
}

// 解析训练参数
static bool train_params_parse(int argc, char ** argv, struct train_params * params) {
    bool invalid_param = false;
    std::string arg;
    // 获取默认的训练参数
    struct train_params default_params = get_default_train_params();
    // 参数前缀
    const std::string arg_prefix = "--";

    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        arg = argv[i];
        // 检查参数是否以指定前缀开头
        if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
// 将参数中的下划线替换为破折号
std::replace(arg.begin(), arg.end(), '_', '-');

// 检查是否有常见的训练参数，如果有则处理并返回 true，同时检查是否有无效参数
if (consume_common_train_arg(argc, argv, &i, &params->common, &invalid_param)) {
    // 如果有无效参数，则跳出循环
    if (invalid_param) {
        break;
    } 
    // 如果需要打印用法，则打印用法并退出程序
    else if (params->common.print_usage) {
        train_print_usage(argc, argv, &default_params);
        exit(0);
    }
} 
// 如果参数为 "--vocab-model"，则将下一个参数作为词汇模型文件名
else if (arg == "--vocab-model") {
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    params->fn_vocab_model = argv[i];
} 
// 如果参数为 "--model-out"，则将下一个参数作为模型输出文件名
else if (arg == "--model-out") {
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
        } 
        // 如果参数是 "--fn-model-out"，则将下一个参数作为模型输出文件名
        params->fn_model_out = argv[i];
    } else if (arg == "--only-write-model") {
        // 如果参数是 "--only-write-model"，则设置只写模型的标志为 true
        params->only_write_model = true;
    } else if (arg == "--embd") {
        // 如果参数是 "--embd"，则将下一个参数作为嵌入维度，并转换为整数
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params->n_embd = std::stoi(argv[i]);
    } else if (arg == "--ff") {
        // 如果参数是 "--ff"，则将下一个参数作为前馈神经网络的维度，并转换为整数
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params->n_ff = std::stoi(argv[i]);
    } else if (arg == "--head") {
        // 如果参数是 "--head"，则将下一个参数作为头部的数量，并转换为整数
        if (++i >= argc) {
            invalid_param = true;
            break;
        } 
        // 如果参数是 "--head"，则将下一个参数转换为整数并赋值给 n_head
        params->n_head = std::stoi(argv[i]);
    } else if (arg == "--layer") {
        // 如果参数是 "--layer"，则将下一个参数转换为整数并赋值给 n_layer
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params->n_layer = std::stoi(argv[i]);
    } else if (arg == "--norm-rms-eps") {
        // 如果参数是 "--norm-rms-eps"，则将下一个参数转换为浮点数并赋值给 f_norm_rms_eps
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params->f_norm_rms_eps = std::stof(argv[i]);
    } else if (arg == "--rope-freq-base") {
        // 如果参数是 "--rope-freq-base"，则将下一个参数转换为浮点数并赋值给 rope_freq_base
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params->rope_freq_base = std::stof(argv[i]);
    } else if (arg == "--rope-freq-scale") {
        // 如果参数是"--rope-freq-scale"，则执行以下操作
        if (++i >= argc) {
            // 如果参数不完整，设置invalid_param为true，并跳出循环
            invalid_param = true;
            break;
        }
        // 将参数转换为浮点数，并赋值给params->rope_freq_scale
        params->rope_freq_scale = std::stof(argv[i]);
    } else {
        // 如果参数不是已知的任何一个选项，则输出错误信息，打印用法，并退出程序
        fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
        train_print_usage(argc, argv, &default_params);
        exit(1);
    }
}
// 如果存在无效参数，输出错误信息，打印用法，并退出程序
if (invalid_param) {
    fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
    train_print_usage(argc, argv, &default_params);
    exit(1);
}
// 完成处理训练参数的操作
finish_processing_train_args(&params->common);

// 返回true
return true;
// 定义保存训练文件数据的结构体
struct save_train_files_data {
    const char            * fn_checkpoint_out;  // 保存检查点文件的文件名
    const char            * fn_model_out;       // 保存模型文件的文件名
    const char            * fn_vocab_model;     // 保存词汇模型文件的文件名
    const char            * pattern_fn_it;      // 文件名模式
    const char            * fn_latest;          // 最新文件的文件名
    struct my_llama_model * model;              // 模型数据
};

// 保存训练文件的函数
static void save_train_files(void * vdata, struct train_state * train) {
    // 将传入的数据指针转换为保存训练文件数据的结构体指针
    struct save_train_files_data * data   = (struct save_train_files_data *) vdata;
    // 获取当前迭代次数
    int64_t iter = train->opt->iter;

    // 如果检查点文件名长度大于0
    if (strlen(data->fn_checkpoint_out) > 0) {
        // 保存检查点文件
        save_checkpoint_file(get_train_filename(data->fn_checkpoint_out, data->pattern_fn_it, data->fn_latest, iter).c_str(), data->fn_vocab_model, data->model, train);
        // 保存最新的检查点文件
        save_checkpoint_file(get_train_filename(data->fn_checkpoint_out, data->pattern_fn_it, data->fn_latest, -1  ).c_str(), data->fn_vocab_model, data->model, train);
    }
}
// 如果模型输出文件名的长度大于0
if (strlen(data->fn_model_out) > 0) {
    // 保存LLAMA模型文件，根据给定的参数生成训练文件名，并保存模型数据
    save_llama_model_file(get_train_filename(data->fn_model_out, data->pattern_fn_it, data->fn_latest, iter).c_str(), data->fn_vocab_model, data->model);
    // 保存LLAMA模型文件，根据给定的参数生成训练文件名，并保存模型数据
    save_llama_model_file(get_train_filename(data->fn_model_out, data->pattern_fn_it, data->fn_latest, -1  ).c_str(), data->fn_vocab_model, data->model);
}

// 获取模型参数数量
static int64_t get_parameter_count(struct my_llama_model* model) {
    int64_t nx = 0;
    // 计算模型的token嵌入、规范化、输出的元素数量并累加
    nx += ggml_nelements(model->tok_embeddings);
    nx += ggml_nelements(model->norm);
    nx += ggml_nelements(model->output);

    // 遍历模型的每一层
    for (uint32_t i = 0; i < model->layers.size(); ++i) {
        auto & layer = model->layers[i];
        // 计算每一层的注意力规范化、wq、wk、wv、wo、ffn_norm的元素数量并累加
        nx += ggml_nelements(layer.attention_norm);
        nx += ggml_nelements(layer.wq);
        nx += ggml_nelements(layer.wk);
        nx += ggml_nelements(layer.wv);
        nx += ggml_nelements(layer.wo);
        nx += ggml_nelements(layer.ffn_norm);
// 计算每个层的元素数量并累加
nx += ggml_nelements(layer.w1);
nx += ggml_nelements(layer.w2);
nx += ggml_nelements(layer.w3);
// 返回总的元素数量
}
return nx;
}

int main(int argc, char ** argv) {
// 获取默认的训练参数
struct train_params params = get_default_train_params();

// 解析命令行参数，如果解析失败则返回1
if (!train_params_parse(argc, argv, &params)) {
    return 1;
}

// 如果参数中的随机种子为默认值，则设置为当前时间
if (params.common.seed == LLAMA_DEFAULT_SEED) {
    params.common.seed = time(NULL);
}
// 打印随机种子
printf("%s: seed: %u\n", __func__, params.common.seed);
// 根据随机种子设置随机数生成器的种子
srand(params.common.seed);
# 使用默认参数创建 llama_model_params 结构体
struct llama_model_params mparams = llama_model_default_params();
# 设置参数 vocab_only 为 true
mparams.vocab_only = true;

# 使用默认参数创建 llama_context_params 结构体
struct llama_context_params cparams = llama_context_default_params();

# 从文件中加载模型，创建 llama_model 结构体
struct llama_model * lmodel = llama_load_model_from_file(params.fn_vocab_model, mparams);
# 使用模型创建上下文，创建 llama_context 结构体
struct llama_context * lctx = llama_new_context_with_model(lmodel, cparams);

# 创建自定义的 llama_model 结构体
struct my_llama_model model;
# 设置模型参数：词汇量
model.hparams.n_vocab = llama_n_vocab(lmodel);
# 设置模型参数：上下文大小
model.hparams.n_ctx   = params.common.n_ctx;
# 设置模型参数：嵌入维度
model.hparams.n_embd  = params.n_embd;
# 设置模型参数：头数
model.hparams.n_head  = params.n_head;
# 设置模型参数：层数
model.hparams.n_layer = params.n_layer;
# 设置模型参数：前馈网络维度
model.hparams.n_ff    = params.n_ff;
# 设置模型参数：旋转数，llama.cpp 要求 n_rot 必须等于 n_embd / n_head
model.hparams.n_rot   = model.hparams.n_embd / model.hparams.n_head;
# 设置模型参数：RMS 归一化的 epsilon
model.hparams.f_norm_rms_eps  = params.f_norm_rms_eps;
# 设置模型参数：ROPE 频率基数
model.hparams.rope_freq_base  = params.rope_freq_base;
# 设置模型参数：ROPE 频率缩放
model.hparams.rope_freq_scale = params.rope_freq_scale;
// 初始化训练状态对象 train
struct train_state * train = init_train_state();
// 从训练状态对象中获取参数上下文对象 opt
struct ggml_opt_context * opt = train->opt;

// 从命令行设置 opt 参数
opt->params = ggml_opt_default_params(GGML_OPT_ADAM);
opt->params.print_forward_graph = false;
opt->params.print_backward_graph = false;
opt->params.graph_size = LLAMA_TRAIN_MAX_NODES;
opt->params.n_threads = params.common.n_threads;
opt->params.past = params.common.opt_past;
opt->params.delta = params.common.opt_delta;
opt->params.max_no_improvement = params.common.opt_max_no_improvement;
opt->params.n_gradient_accumulation = params.common.n_gradient_accumulation;
opt->params.adam.n_iter = params.common.adam_n_iter;
opt->params.adam.sched = 1.0f;
opt->params.adam.alpha = params.common.adam_alpha;
opt->params.adam.decay = params.common.adam_decay;
opt->params.adam.decay_min_ndim = params.common.adam_decay_min_ndim;
opt->params.adam.beta1 = params.common.adam_beta1;
    # 设置 Adam 优化器的参数
    opt->params.adam.beta2              = params.common.adam_beta2;
    opt->params.adam.gclip              = params.common.adam_gclip;
    opt->params.adam.eps_f              = params.common.adam_eps_f;

    # 打印初始化模型的信息
    printf("%s: init model\n", __func__);
    # 加载检查点文件，检查文件是否存在
    bool existed = load_checkpoint_file(params.common.fn_checkpoint_in, &model, train);
    if (existed) {
        # 如果检查点文件存在，根据用户提供的 n_ctx 覆盖上一个 n_ctx
        if (params.common.custom_n_ctx) {
            model.hparams.n_ctx = params.common.n_ctx;
        }

        # 检查优化器参数是否改变
        const bool opt_past_changed = opt->params.past != params.common.opt_past;

        if (opt_past_changed) {
            # 如果优化器参数改变，输出错误信息并终止程序
            die("Optimizer parameter '--opt-past N' differs from checkpoint file. To use different value train from scratch with empty input checkpoint, e.g --checkpoint-in ''. Aborting");
            # 需要丢弃先前的优化器过去函数值统计，并使用新的形状进行优化器初始化
            # TODO
        }
    } else {
    // 初始化模型
    init_model(&model);
    // 随机化模型参数
    randomize_model(&model, params.common.seed, 0.0f, 1.0f, -1.0f, +1.0f);
    // 如果不仅仅是写模型，则初始化优化器
    if (!params.only_write_model) {
        ggml_opt_init(opt->ctx, opt, opt->params, get_parameter_count(&model));
    }
    // 设置迭代次数为训练次数
    opt->iter = train->train_its;

    // 打印模型参数
    print_params(&model.hparams);
    // 打印训练迭代次数
    printf("%s: total train_iterations %llu\n", __func__, (long long unsigned) train->train_its);
    // 打印已观察到的训练样本数
    printf("%s: seen train_samples     %llu\n", __func__, (long long unsigned) train->train_samples);
    // 打印已观察到的训练标记数
    printf("%s: seen train_tokens      %llu\n", __func__, (long long unsigned) train->train_tokens);
    // 打印已完成的训练周期数
    printf("%s: completed train_epochs %llu\n", __func__, (long long unsigned) train->train_epochs);
    // 打印模型大小
    printf("%s: model_size = %zu bytes (%.1f MB)\n", __func__, (ggml_used_mem(model.ctx) + model.data.size()), (float) (ggml_used_mem(model.ctx) + model.data.size()) / (1024.0f*1024.0f));

    // 如果只是写模型，则设置保存训练文件数据的参数
    if (params.only_write_model) {
        save_train_files_data save_data;
        save_data.fn_checkpoint_out = "";
        save_data.fn_model_out      = params.fn_model_out;
        save_data.fn_vocab_model    = params.fn_vocab_model;
    }
        # 将参数中的模式文件名模式、最新文件名、模型指针保存到save_data结构体中
        save_data.pattern_fn_it     = params.common.pattern_fn_it;
        save_data.fn_latest         = params.common.fn_latest;
        save_data.model             = &model;

        # 保存训练文件
        save_train_files(&save_data, train);

        # 释放训练状态
        free_train_state(train);
        # 释放模型上下文
        ggml_free(model.ctx);
        # 释放lctx
        llama_free(lctx);
        # 释放lmodel
        llama_free_model(lmodel);
        # 返回0表示成功
        return 0;
    }

    # 打印优化上下文的内存大小
    printf("%s: opt_size  = %zu bytes (%.1f MB)\n", __func__, ggml_get_mem_size(opt->ctx), (float) ggml_get_mem_size(opt->ctx) / (1024.0f*1024.0f));
    # 打印优化迭代次数
    printf("%s: opt iter %d\n", __func__, opt->iter);

    # 获取模型参数中的token数量、词汇表大小、批次大小
    int n_tokens = model.hparams.n_ctx;
    int n_vocab  = model.hparams.n_vocab;
    int n_batch  = params.common.n_batch;
    // 创建两个空的 uint8_t 类型的向量，用于存储输入数据和计算数据
    std::vector<uint8_t> mem_input_data;
    std::vector<uint8_t> mem_compute_data;

    // 分配器指针初始化为空
    ggml_allocr * alloc = NULL;

    // 初始化输入张量的上下文，不包括数据
    struct ggml_init_params ctx_input_params = {
        ggml_tensor_overhead() * 2, // mem_size，计算上下文所需的内存大小
        NULL,                       // mem_buffer，内存缓冲区为空
        true,                       // no_alloc，不分配内存
    };
    // 创建输入张量的上下文
    struct ggml_context * ctx_input = ggml_init(ctx_input_params);

    // 创建输入张量
    struct ggml_tensor * tokens_input  = ggml_new_tensor_2d(ctx_input, GGML_TYPE_I32, n_tokens, n_batch); // 二维整型输入张量
    struct ggml_tensor * target_probs  = ggml_new_tensor_3d(ctx_input, GGML_TYPE_F32, n_vocab,  n_tokens, n_batch); // 三维浮点型输入张量

    // 测量输入张量所需的内存大小
    size_t max_input_size = GGML_PAD(ggml_nbytes(tokens_input), tensor_alignment) + // 输入张量所需内存大小
                            GGML_PAD(ggml_nbytes(target_probs), tensor_alignment) + // 目标概率张量所需内存大小
    // 打印输入大小的信息，包括字节数和以 MB 为单位的大小
    printf("%s: input_size = %zu bytes (%.1f MB)\n", __func__, max_input_size, (float) max_input_size / (1024.0f*1024.0f));

    // 分配输入张量的内存空间
    mem_input_data.resize(max_input_size);
    // 创建内存分配器对象，用于分配输入张量的内存空间
    alloc = ggml_allocr_new(mem_input_data.data(), mem_input_data.size(), tensor_alignment);
    // 分配输入张量的内存空间
    ggml_allocr_alloc(alloc, tokens_input);
    // 分配目标概率的内存空间
    ggml_allocr_alloc(alloc, target_probs);
    // 释放内存分配器对象
    ggml_allocr_free(alloc);

    // 估算计算张量不包含数据时的内存大小
    const size_t estimated_compute_size_wo_data = (
            2*LLAMA_TRAIN_MAX_NODES*ggml_tensor_overhead() +
            (params.common.use_checkpointing ? 3 : 2)*(GGML_OBJECT_SIZE+ggml_graph_overhead_custom(LLAMA_TRAIN_MAX_NODES, true))
    );
    // 创建用于计算张量的上下文参数
    struct ggml_init_params ctx_compute_params = {
        estimated_compute_size_wo_data, // mem_size
        NULL,                           // mem_buffer
        true,                           // no_alloc
    };
    // 定义指向 ggml_context 结构体的指针，并初始化为 NULL
    struct ggml_context * ctx_compute = NULL;

    // 定义指向 ggml_tensor 结构体的指针，并初始化为 NULL
    struct ggml_tensor * loss   = NULL;
    struct ggml_tensor * logits = NULL;

    // 定义指向 ggml_cgraph 结构体的指针，并初始化为 NULL
    struct ggml_cgraph * gf     = NULL;
    struct ggml_cgraph * gb     = NULL;
    struct ggml_cgraph * gb_tmp = NULL;

    // 用于存储计算张量所需的内存大小
    size_t best_compute_size = SIZE_MAX;
    // 用于存储最佳评估顺序
    enum ggml_cgraph_eval_order best_order = GGML_CGRAPH_EVAL_ORDER_COUNT;
    
    // 寻找最佳的评估顺序
    for (unsigned order = 0; order < (unsigned) GGML_CGRAPH_EVAL_ORDER_COUNT; ++order) {
        // 初始化 ggml_context 结构体
        ctx_compute = ggml_init(ctx_compute_params);
        // 创建用于测量内存分配的对象
        alloc = ggml_allocr_new_measure(tensor_alignment);
        // 创建自定义大小的计算图
        gf = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
        // 设置计算图的评估顺序
        gf->order = (enum ggml_cgraph_eval_order) order;
        // 创建自定义大小的计算图
        gb = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
        // 临时存储用于检查点的计算图
        gb_tmp = params.common.use_checkpointing
// 调用 ggml_new_graph_custom 函数创建一个新的图形对象，如果成功则返回该对象的指针，否则返回 NULL
ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true) : NULL;
// 调用 llama_build_train_graphs 函数构建训练图形，并返回损失值
loss = llama_build_train_graphs(
    &model, alloc, ctx_compute,
    gf, gb, gb_tmp,
    &logits, tokens_input, target_probs,
    n_tokens, n_batch,
    params.common.use_flash,
    params.common.use_checkpointing
);
// 计算最大计算大小，包括分配的内存和张量对齐
size_t max_compute_size = ggml_allocr_max_size(alloc) + tensor_alignment;
// 如果最大计算大小小于最佳计算大小，则更新最佳计算大小和最佳顺序
if (max_compute_size < best_compute_size) {
    best_compute_size = max_compute_size;
    best_order = gf->order;
}
// 释放分配的内存
ggml_allocr_free(alloc);
// 释放计算上下文
ggml_free(ctx_compute);
// 将最佳计算大小赋值给 max_compute_size
size_t max_compute_size = best_compute_size;
// 打印最佳计算大小的信息
printf("%s: compute_size = %zu bytes (%.1f MB)\n", __func__, max_compute_size, (float) max_compute_size / (1024.0f*1024.0f));
    // 打印评估顺序的信息，根据最佳顺序的枚举值进行判断并打印相应的字符串
    printf("%s: evaluation order = %s\n", __func__,
        (best_order == GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT) ? "LEFT_TO_RIGHT" :
        (best_order == GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT) ? "RIGHT_TO_LEFT" :
        "invalid");

    // 分配计算张量的内存空间
    mem_compute_data.resize(max_compute_size);
    // 初始化计算上下文
    ctx_compute = ggml_init(ctx_compute_params);
    // 创建计算图对象
    alloc = ggml_allocr_new(mem_compute_data.data(), mem_compute_data.size(), tensor_alignment);
    // 创建新的计算图对象
    gf = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
    // 设置计算图对象的顺序
    gf->order = best_order;
    // 创建新的计算图对象
    gb = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
    // 根据参数判断是否创建新的计算图对象
    gb_tmp = params.common.use_checkpointing
        ? ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true)
        : NULL;
    // 构建训练图
    loss = llama_build_train_graphs(
        &model, alloc, ctx_compute,
        gf, gb, gb_tmp,
        &logits, tokens_input, target_probs,
        n_tokens, n_batch,
    # 使用 Flash 内存和检查点
    params.common.use_flash,
    params.common.use_checkpointing
);

# 释放内存
ggml_allocr_free(alloc);

# 创建用于训练的标记和样本起始位置、大小的向量
std::vector<llama_token> train_tokens;
std::vector<size_t> train_samples_begin;
std::vector<size_t> train_samples_size;

# 打印信息：标记训练数据
printf("%s: tokenize training data\n", __func__);

# 对训练数据进行标记化处理
tokenize_file(lctx,
        params.common.fn_train_data,
        params.common.sample_start,
        params.common.include_sample_start,
        params.common.overlapping_samples,
        n_tokens,
        train_tokens,
        train_samples_begin,
        train_samples_size);

# 检查训练样本的起始位置和大小是否一致
GGML_ASSERT(train_samples_begin.size() == train_samples_size.size());
    # 打印训练标记的数量
    printf("%s: number of training tokens: %zu\n", __func__, train_tokens.size());

    # 计算训练样本的哈希值
    size_t shuffle_samples_hash = compute_samples_hash(params.common.fn_train_data, train_samples_begin.data(), train_samples_size.data(), train_samples_size.size());
    # 检查训练数据是否发生了变化
    const bool changed_train_data = (shuffle_samples_hash != train->shuffle_samples_hash) || (train->shuffle_sample_count != train_samples_size.size());
    if (changed_train_data) {
        # 如果训练数据发生了变化，则重新开始洗牌的周期
        printf("%s: train data seems to have changed. restarting shuffled epoch.\n", __func__);
    }
    if (params.common.force_reshuffle) {
        # 如果强制重新洗牌数据，则重新开始新的洗牌周期
        printf("%s: forced reshuffling of data. restarting with newly shuffled epoch.\n", __func__);
    }
    if ((train->shuffle_rng_state_current == "") || changed_train_data || params.common.force_reshuffle) {
        # 如果当前的洗牌状态为空，或者训练数据发生了变化，或者强制重新洗牌数据
        # 则重新设置洗牌状态、样本数量、下一个样本索引和样本哈希值
        train->shuffle_rng_state_current = mt19937_seed_to_state(params.common.seed);
        train->shuffle_sample_count = train_samples_size.size();
        train->shuffle_next_sample = 0;
        train->shuffle_samples_hash = shuffle_samples_hash;
    }
    # 初始化训练样本的偏移量、起始位置和大小
    std::vector<size_t> train_shuffled_samples_offs;
    std::vector<size_t> train_shuffled_samples_begin;
    std::vector<size_t> train_shuffled_samples_size;
    train_shuffled_samples_offs.resize(train_samples_begin.size());
    // 调整训练样本的起始位置数组的大小
    train_shuffled_samples_begin.resize(train_samples_begin.size());
    // 调整训练样本的大小数组的大小
    train_shuffled_samples_size.resize(train_samples_size.size());
    // 设置下一次随机数生成器状态为当前状态，并对训练样本进行洗牌
    train->shuffle_rng_state_next = shuffle_samples(
        train->shuffle_rng_state_current,
        train_shuffled_samples_offs.data(),
        train_shuffled_samples_begin.data(),
        train_shuffled_samples_size.data(),
        train_samples_begin.data(),
        train_samples_size.data(),
        train_samples_size.size());
    // 打印训练开始的提示信息
    printf("%s: begin training\n", __func__);

    // 创建保存训练文件数据的对象
    save_train_files_data save_data;
    // 设置检查点输出文件名
    save_data.fn_checkpoint_out = params.common.fn_checkpoint_out;
    // 设置模型输出文件名
    save_data.fn_model_out      = params.fn_model_out;
    // 设置词汇模型文件名
    save_data.fn_vocab_model    = params.fn_vocab_model;
    // 设置迭代模式文件名的模式
    save_data.pattern_fn_it     = params.common.pattern_fn_it;
    // 设置最新文件名
    save_data.fn_latest         = params.common.fn_latest;
    // 设置模型指针
    save_data.model             = &model;
# 创建一个结构体变量opt_cb_data，用于存储训练过程中的回调数据
struct train_opt_callback_data opt_cb_data;
# 将params.common的地址赋值给opt_cb_data.params
opt_cb_data.params = &params.common;
# 将train的地址赋值给opt_cb_data.train
opt_cb_data.train = train;
# 将save_train_files的地址赋值给opt_cb_data.save_cb，用于保存训练文件
opt_cb_data.save_cb = &save_train_files;
# 将save_data的地址赋值给opt_cb_data.save_data，用于保存数据
opt_cb_data.save_data = &save_data;
# 将lctx的值赋值给opt_cb_data.lctx
opt_cb_data.lctx = lctx;
# 将opt->iter的值赋值给opt_cb_data.last_save_iter，用于保存上一次迭代的值
opt_cb_data.last_save_iter = opt->iter;
# 将train_tokens的数据地址赋值给opt_cb_data.tokens_data，用于存储训练tokens的数据
opt_cb_data.tokens_data = train_tokens.data();
# 将train_tokens的大小赋值给opt_cb_data.tokens_size，用于存储训练tokens的大小
opt_cb_data.tokens_size = train_tokens.size();
# 将train_samples_begin的数据地址赋值给opt_cb_data.samples_begin，用于存储训练样本的起始位置
opt_cb_data.samples_begin = train_samples_begin.data();
# 将train_samples_size的数据地址赋值给opt_cb_data.samples_size，用于存储训练样本的大小
opt_cb_data.samples_size = train_samples_size.data();
# 将train_shuffled_samples_offs的数据地址赋值给opt_cb_data.shuffled_samples_offs，用于存储训练样本的打乱偏移
opt_cb_data.shuffled_samples_offs = train_shuffled_samples_offs.data();
# 将train_shuffled_samples_begin的数据地址赋值给opt_cb_data.shuffled_samples_begin，用于存储训练样本的打乱起始位置
opt_cb_data.shuffled_samples_begin = train_shuffled_samples_begin.data();
# 将train_shuffled_samples_size的数据地址赋值给opt_cb_data.shuffled_samples_size，用于存储训练样本的打乱大小
opt_cb_data.shuffled_samples_size = train_shuffled_samples_size.data();
# 将train_samples_size的大小赋值给opt_cb_data.samples_count，用于存储训练样本的数量
opt_cb_data.samples_count = train_samples_size.size();
# 将tokens_input的值赋值给opt_cb_data.tokens_input，用于存储tokens的输入
opt_cb_data.tokens_input = tokens_input;
# 将target_probs的值赋值给opt_cb_data.target_probs，用于存储目标概率
opt_cb_data.target_probs = target_probs;
# 将opt->iter的值赋值给opt_cb_data.first_iter，用于存储第一次迭代的值
opt_cb_data.first_iter = opt->iter;
# 将train->train_epochs的值赋值给opt_cb_data.first_epoch，用于存储第一次训练的轮数
opt_cb_data.first_epoch = train->train_epochs;
# 将-1赋值给opt_cb_data.iter_at_last_epoch，用于存储上一个轮数的迭代值
opt_cb_data.iter_at_last_epoch = -1;
    // 设置opt_cb_data的last_time为当前时间
    opt_cb_data.last_time = ggml_time_ms();
    // 设置opt_cb_data的millis_per_iter为0.0
    opt_cb_data.millis_per_iter = 0.0;

    // 测量工作缓冲区所需的内存大小
    size_t max_work_size = ggml_graph_plan(gb, params.common.n_threads).work_size + GGML_OBJECT_SIZE;
    // 打印工作缓冲区的大小
    printf("%s: work_size = %zu bytes (%.1f MB)\n", __func__, max_work_size, (float) max_work_size / (1024.0f*1024.0f));

    // 为工作缓冲区创建上下文
    struct ggml_init_params ctx_work_params = {
        max_work_size, // mem_size
        NULL,          // mem_buffer
        false,         // no_alloc
    };
    // 初始化工作缓冲区的上下文
    struct ggml_context * ctx_work = ggml_init(ctx_work_params);

    // 记录开始时间
    int64_t t0 = ggml_time_ms();

    // 恢复优化器的状态并继续训练
    ggml_opt_resume_g(ctx_work, opt, loss, gf, gb, &train_opt_callback, (void *) &opt_cb_data);

    // 释放工作缓冲区的上下文
    ggml_free(ctx_work);
    释放计算上下文的内存
    ggml_free(ctx_compute);
    释放输入上下文的内存
    ggml_free(ctx_input);

    记录当前时间
    int64_t t1 = ggml_time_ms();
    打印总训练时间
    printf("%s: total training time: ", __func__);
    打印训练时间的持续时间
    print_duration((double) (t1 - t0));
    换行
    printf("\n");

    计算新的迭代次数
    int new_iters = opt->iter - opt_cb_data.last_save_iter;
    如果新的迭代次数大于0
    if (new_iters > 0) {
        更新训练总迭代次数
        train->train_its     += new_iters;
        更新训练总tokens数
        train->train_tokens  += new_iters * opt->params.n_gradient_accumulation * n_batch * n_tokens;

        保存训练文件
        save_train_files(&save_data, train);
        更新最后保存的迭代次数
        opt_cb_data.last_save_iter = opt->iter;
    }

    如果有分配的内存
    if (alloc) {
        释放分配的内存
        ggml_allocr_free(alloc);
    }
# 释放 opt->ctx 所占用的内存
ggml_free(opt->ctx);
# 释放 train 所占用的内存
free_train_state(train);
# 释放 model.ctx 所占用的内存
ggml_free(model.ctx);
# 释放 lctx 所占用的内存
llama_free(lctx);
# 释放 lmodel 所占用的内存
llama_free_model(lmodel);
# 返回 0，表示函数执行成功
return 0;
```