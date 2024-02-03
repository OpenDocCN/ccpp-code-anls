# `PowerInfer\examples\train-text-from-scratch\train-text-from-scratch.cpp`

```cpp
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

// 如果是 MSC 编译器，禁用警告 4244 和 4267
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义张量的对齐方式
static const size_t tensor_alignment = 32;

// 定义自定义的 llama 模型超参数结构体
struct my_llama_hparams {
    uint32_t n_vocab = 32000;
    uint32_t n_ctx   = 512;
    uint32_t n_embd  = 4096;
    uint32_t n_head  = 32;
    uint32_t n_layer = 32;
    uint32_t n_rot   = 64;
    uint32_t n_ff    = 11008;

    // float f_norm_eps     = 1e-5f; // falcon
    float f_norm_rms_eps = 1e-5f; // llama

    float rope_freq_base  = 10000.0f;
    float rope_freq_scale = 1.0f;
};

// 定义自定义的 llama 层结构体
struct my_llama_layer {
    // normalization
    struct ggml_tensor * attention_norm;

    // attention
    struct ggml_tensor * wq;
    struct ggml_tensor * wk;
    struct ggml_tensor * wv;
    struct ggml_tensor * wo;

    // normalization
    struct ggml_tensor * ffn_norm;

    // ff
    struct ggml_tensor * w1;
    struct ggml_tensor * w2;
    struct ggml_tensor * w3;
};

// 定义自定义的 llama 模型结构体
struct my_llama_model {
    struct ggml_context * ctx = NULL;
    std::vector<uint8_t> data;

    my_llama_hparams hparams;

    struct ggml_tensor * tok_embeddings;

    struct ggml_tensor * norm;
    struct ggml_tensor * output;

    std::vector<my_llama_layer> layers;
};

// 定义 gguf 常量（与 gguf.py 同步）
static const char * LLM_KV_TRAINING_TYPE_TRAIN_MODEL     = "train_model";
static const char * LLM_KV_TRAINING_TYPE                 = "training.type";

static const char * LLM_KV_GENERAL_ARCHITECTURE        = "general.architecture";
static const char * LLM_KV_GENERAL_FILE_TYPE           = "general.file_type";

static const char * LLM_KV_CONTEXT_LENGTH              = "%s.context_length";
# 定义常量，表示嵌入长度的键值
static const char * LLM_KV_EMBEDDING_LENGTH            = "%s.embedding_length";
# 定义常量，表示块数量的键值
static const char * LLM_KV_BLOCK_COUNT                 = "%s.block_count";
# 定义常量，表示前馈长度的键值
static const char * LLM_KV_FEED_FORWARD_LENGTH         = "%s.feed_forward_length";
# 定义常量，表示注意力头数量的键值
static const char * LLM_KV_ATTENTION_HEAD_COUNT        = "%s.attention.head_count";
# 定义常量，表示注意力层归一化的 RMS 误差的键值
static const char * LLM_KV_ATTENTION_LAYERNORM_RMS_EPS = "%s.attention.layer_norm_rms_epsilon";
# 定义常量，表示 ROPE 维度数量的键值
static const char * LLM_KV_ROPE_DIMENSION_COUNT        = "%s.rope.dimension_count";
# 定义常量，表示 ROPE 频率基数的键值
static const char * LLM_KV_ROPE_FREQ_BASE              = "%s.rope.freq_base"; // TODO load in llama.cpp
# 定义常量，表示 ROPE 线性缩放的键值
static const char * LLM_KV_ROPE_SCALE_LINEAR           = "%s.rope.scale_linear";

# 定义常量，表示分词器模型的键值
static const char * LLM_KV_TOKENIZER_MODEL             = "tokenizer.ggml.model";
# 定义常量，表示分词器标记列表的键值
static const char * LLM_KV_TOKENIZER_LIST              = "tokenizer.ggml.tokens";
# 定义常量，表示分词器标记类型的键值
static const char * LLM_KV_TOKENIZER_TOKEN_TYPE        = "tokenizer.ggml.token_type";
# 定义常量，表示分词器分数的键值
static const char * LLM_KV_TOKENIZER_SCORES            = "tokenizer.ggml.scores";
# 定义常量，表示分词器合并的键值
static const char * LLM_KV_TOKENIZER_MERGES            = "tokenizer.ggml.merges";
# 定义常量，表示分词器起始标记 ID 的键值
static const char * LLM_KV_TOKENIZER_BOS_ID            = "tokenizer.ggml.bos_token_id";
# 定义常量，表示分词器结束标记 ID 的键值
static const char * LLM_KV_TOKENIZER_EOS_ID            = "tokenizer.ggml.eos_token_id";
# 定义常量，表示分词器未知标记 ID 的键值
static const char * LLM_KV_TOKENIZER_UNK_ID            = "tokenizer.ggml.unknown_token_id";
# 定义常量，表示分词器分隔符标记 ID 的键值
static const char * LLM_KV_TOKENIZER_SEP_ID            = "tokenizer.ggml.seperator_token_id";
# 定义常量，表示分词器填充标记 ID 的键值
static const char * LLM_KV_TOKENIZER_PAD_ID            = "tokenizer.ggml.padding_token_id";

# 定义常量，表示标记嵌入的张量名称
static const char * LLM_TENSOR_TOKEN_EMBD    = "token_embd";
# 定义常量，表示输出归一化的张量名称
static const char * LLM_TENSOR_OUTPUT_NORM   = "output_norm";
# 定义常量，表示输出的张量名称
static const char * LLM_TENSOR_OUTPUT        = "output";
# 定义常量，表示注意力归一化的张量名称
static const char * LLM_TENSOR_ATTN_NORM     = "blk.%d.attn_norm";
# 定义常量，表示注意力查询的张量名称
static const char * LLM_TENSOR_ATTN_Q        = "blk.%d.attn_q";
# 定义常量，表示注意力键的张量名称
static const char * LLM_TENSOR_ATTN_K        = "blk.%d.attn_k";
# 定义常量，表示注意力值的张量名称
static const char * LLM_TENSOR_ATTN_V        = "blk.%d.attn_v";
// 定义字符串常量，表示不同的张量名称模板
static const char * LLM_TENSOR_ATTN_OUT      = "blk.%d.attn_output";
static const char * LLM_TENSOR_FFN_NORM      = "blk.%d.ffn_norm";
static const char * LLM_TENSOR_FFN_GATE      = "blk.%d.ffn_gate";
static const char * LLM_TENSOR_FFN_DOWN      = "blk.%d.ffn_down";
static const char * LLM_TENSOR_FFN_UP        = "blk.%d.ffn_up";

// 打印模型参数的函数
static void print_params(struct my_llama_hparams * params) {
    // 打印函数名和词汇量大小
    printf("%s: n_vocab: %d\n", __func__, params->n_vocab);
    // 打印函数名和上下文大小
    printf("%s: n_ctx:   %d\n", __func__, params->n_ctx);
    // 打印函数名和嵌入维度大小
    printf("%s: n_embd:  %d\n", __func__, params->n_embd);
    // 打印函数名和头数
    printf("%s: n_head:  %d\n", __func__, params->n_head);
    // 打印函数名和前馈网络大小
    printf("%s: n_ff:    %d\n", __func__, params->n_ff);
    // 打印函数名和层数
    printf("%s: n_layer: %d\n", __func__, params->n_layer);
    // 打印函数名和旋转数
    printf("%s: n_rot:   %d\n", __func__, params->n_rot);
}

// 设置模型参数的函数
static void set_param_model(struct my_llama_model * model) {
    // 获取模型的超参数
    const auto& hparams = model->hparams;
    // 获取层数
    const uint32_t n_layer = hparams.n_layer;
    // 获取模型的上下文
    struct ggml_context* ctx = model->ctx;

    // 设置模型的参数
    ggml_set_param(ctx, model->tok_embeddings);
    ggml_set_param(ctx, model->norm);
    ggml_set_param(ctx, model->output);

    // 遍历每一层，设置每一层的参数
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

// 分配模型的函数
static void alloc_model(struct ggml_allocr * alloc, struct my_llama_model * model) {
    // 分配模型的词嵌入
    ggml_allocr_alloc(alloc, model->tok_embeddings);
    // 分配模型的归一化层
    ggml_allocr_alloc(alloc, model->norm);
    // 分配模型的输出层
    ggml_allocr_alloc(alloc, model->output);
}
    # 遍历模型的每一层
    for (uint32_t i = 0; i < model->layers.size(); ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];
        # 分配内存给当前层的注意力归一化
        ggml_allocr_alloc(alloc, layer.attention_norm);
        # 分配内存给当前层的查询权重
        ggml_allocr_alloc(alloc, layer.wq);
        # 分配内存给当前层的键权重
        ggml_allocr_alloc(alloc, layer.wk);
        # 分配内存给当前层的值权重
        ggml_allocr_alloc(alloc, layer.wv);
        # 分配内存给当前层的输出权重
        ggml_allocr_alloc(alloc, layer.wo);
        # 分配内存给当前层的前馈神经网络归一化
        ggml_allocr_alloc(alloc, layer.ffn_norm);
        # 分配内存给当前层的第一个权重
        ggml_allocr_alloc(alloc, layer.w1);
        # 分配内存给当前层的第二个权重
        ggml_allocr_alloc(alloc, layer.w2);
        # 分配内存给当前层的第三个权重
        ggml_allocr_alloc(alloc, layer.w3);
    }
    # 分配内存给模型的词嵌入梯度
    ggml_allocr_alloc(alloc, model->tok_embeddings->grad);
    # 分配内存给模型的归一化梯度
    ggml_allocr_alloc(alloc, model->norm->grad);
    # 分配内存给模型的输出梯度
    ggml_allocr_alloc(alloc, model->output->grad);
    # 再次遍历模型的每一层
    for (uint32_t i = 0; i < model->layers.size(); ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];
        # 分配内存给当前层的注意力归一化梯度
        ggml_allocr_alloc(alloc, layer.attention_norm->grad);
        # 分配内存给当前层的查询权重梯度
        ggml_allocr_alloc(alloc, layer.wq->grad);
        # 分配内存给当前层的键权重梯度
        ggml_allocr_alloc(alloc, layer.wk->grad);
        # 分配内存给当前层的值权重梯度
        ggml_allocr_alloc(alloc, layer.wv->grad);
        # 分配内存给当前层的输出权重梯度
        ggml_allocr_alloc(alloc, layer.wo->grad);
        # 分配内存给当前层的前馈神经网络归一化梯度
        ggml_allocr_alloc(alloc, layer.ffn_norm->grad);
        # 分配内存给当前层的第一个权重梯度
        ggml_allocr_alloc(alloc, layer.w1->grad);
        # 分配内存给当前层的第二个权重梯度
        ggml_allocr_alloc(alloc, layer.w2->grad);
        # 分配内存给当前层的第三个权重梯度
        ggml_allocr_alloc(alloc, layer.w3->grad);
    }
// 初始化模型结构体
static void init_model(struct my_llama_model * model) {
    // 获取模型超参数
    const auto & hparams = model->hparams;
    // 从超参数中获取嵌入维度、层数、词汇量和前馈网络维度
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;
    const uint32_t n_vocab = hparams.n_vocab;
    const uint32_t n_ff    = hparams.n_ff;

    // 初始化存储临时名称的缓冲区
    std::vector<char> tn_buf;
    tn_buf.resize(GGML_MAX_NAME);
    // 定义一个lambda函数，用于生成带有".weight"后缀的名称
    auto tn = [&tn_buf](const char * key) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", key);
        return tn_buf.data();
    };
    // 定义一个lambda函数，用于生成带有索引和".weight"后缀的名称
    auto tni = [&tn_buf](const char * key, int bid) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), key, bid);
        std::string s = tn_buf.data();
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", s.c_str());
        return tn_buf.data();
    };

    // 初始化模型张量的上下文参数
    struct ggml_init_params ctx_model_params;
    ctx_model_params.mem_size   = ggml_tensor_overhead()*2*(6 + n_layer*18);
    ctx_model_params.mem_buffer = NULL;
    ctx_model_params.no_alloc   = true;

    // 初始化模型的上下文
    struct ggml_context * ctx = ggml_init(ctx_model_params);
    model->ctx = ctx;

    // 创建模型的词嵌入张量、规范化张量和输出张量
    model->tok_embeddings = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab);
    model->norm           = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
    model->output         = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab);

    // 设置张量的名称
    ggml_set_name(model->tok_embeddings, tn(LLM_TENSOR_TOKEN_EMBD));
    ggml_set_name(model->norm,           tn(LLM_TENSOR_OUTPUT_NORM));
    ggml_set_name(model->output,         tn(LLM_TENSOR_OUTPUT));

    // 调整模型的层数
    model->layers.resize(n_layer);
}
    // 遍历每个层，初始化每个层的注意力规范化张量
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = model->layers[i];

        // 初始化每个层的注意力权重张量
        layer.attention_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        // 初始化每个层的查询权重张量
        layer.wq = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        // 初始化每个层的键权重张量
        layer.wk = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        // 初始化每个层的值权重张量
        layer.wv = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        // 初始化每个层的输出权重张量
        layer.wo = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);

        // 初始化每个层的前馈神经网络规范化张量
        layer.ffn_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        // 初始化每个层的前馈神经网络权重张量
        layer.w1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);
        layer.w2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_ff, n_embd);
        layer.w3 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);

        // 设置每个张量的名称
        ggml_set_name(layer.attention_norm, tni(LLM_TENSOR_ATTN_NORM, i));

        ggml_set_name(layer.wq,             tni(LLM_TENSOR_ATTN_Q, i));
        ggml_set_name(layer.wk,             tni(LLM_TENSOR_ATTN_K, i));
        ggml_set_name(layer.wv,             tni(LLM_TENSOR_ATTN_V, i));
        ggml_set_name(layer.wo,             tni(LLM_TENSOR_ATTN_OUT, i));

        ggml_set_name(layer.ffn_norm,       tni(LLM_TENSOR_FFN_NORM, i));

        ggml_set_name(layer.w1,             tni(LLM_TENSOR_FFN_GATE, i));
        ggml_set_name(layer.w2,             tni(LLM_TENSOR_FFN_DOWN, i));
        ggml_set_name(layer.w3,             tni(LLM_TENSOR_FFN_UP, i));
    }

    // 设置模型参数
    set_param_model(model);

    // 计算数据大小
    size_t size = 0;
    for (struct ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
        size += GGML_PAD(ggml_nbytes(t), tensor_alignment);
    }

    // 分配数据
    struct ggml_allocr * alloc = NULL;
    model->data.resize(size + tensor_alignment);
    alloc = ggml_allocr_new(model->data.data(), model->data.size(), tensor_alignment);
    alloc_model(alloc, model);
    ggml_allocr_free(alloc);
}

// 随机初始化模型参数
static void randomize_model(struct my_llama_model * model, int seed, float mean, float std, float min, float max) {
    // 获取模型的超参数
    const auto & hparams = model->hparams;

    // 获取层数
    const uint32_t n_layer = hparams.n_layer;

    // 初始化随机正态分布
    struct random_normal_distribution * rnd = init_random_normal_distribution(seed, mean, std, min, max);

    // 随机初始化模型中的张量
    randomize_tensor_normal(model->tok_embeddings, rnd);
    randomize_tensor_normal(model->norm,           rnd);
    randomize_tensor_normal(model->output,         rnd);

    // 循环遍历每一层，随机初始化张量
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = model->layers[i];
        randomize_tensor_normal(layer.attention_norm, rnd);

        randomize_tensor_normal(layer.wq, rnd);
        randomize_tensor_normal(layer.wk, rnd);
        randomize_tensor_normal(layer.wv, rnd);
        randomize_tensor_normal(layer.wo, rnd);

        randomize_tensor_normal(layer.ffn_norm, rnd);

        randomize_tensor_normal(layer.w1, rnd);
        randomize_tensor_normal(layer.w2, rnd);
        randomize_tensor_normal(layer.w3, rnd);
    }

    // 释放随机正态分布
    free_random_normal_distribution(rnd);
}

// 构建训练图
static struct ggml_tensor * llama_build_train_graphs(
        struct my_llama_model * model,
        struct ggml_allocr    * alloc,
        struct ggml_context   * ctx,
        struct ggml_cgraph    * gf,
        struct ggml_cgraph    * gb,
        struct ggml_cgraph    * gb_tmp,
        struct ggml_tensor  * * logits,
        struct ggml_tensor    * tokens_input,
        struct ggml_tensor    * targets,
        const  int              n_tokens,
        const  int              n_batch,
        const  bool             enable_flash_attn,
        const  bool             enable_checkpointing) {

    // 设置上下文的临时空间
    ggml_set_scratch(ctx, { 0, 0, nullptr, });
    const int n_past = 0;
    const int N = n_tokens;
    const auto & hparams = model->hparams;
    const int n_ctx      = hparams.n_ctx;
    const int n_vocab    = hparams.n_vocab;
    const int n_embd     = hparams.n_embd;
    const int n_layer    = hparams.n_layer;
    // 定义并初始化变量n_head，表示头的数量
    const int n_head     = hparams.n_head;
    // 定义并初始化变量n_rot，表示旋转的数量
    const int n_rot      = hparams.n_rot;
    // 定义并初始化变量n_ff，表示前馈神经网络的数量
    const int n_ff       = hparams.n_ff;
    // 定义并初始化变量f_norm_rms_eps，表示归一化的均方根误差
    const float f_norm_rms_eps  = hparams.f_norm_rms_eps;
    // 定义并初始化变量rope_freq_base，表示绳子的基础频率
    const float rope_freq_base  = hparams.rope_freq_base;
    // 定义并初始化变量rope_freq_scale，表示绳子的频率比例
    const float rope_freq_scale = hparams.rope_freq_scale;

    // 定义一个lambda函数set_name，用于设置张量的名称
    auto set_name = [](struct ggml_tensor * t, const char * n) {
        ggml_set_name(t, n);
        // 如果张量有梯度，设置梯度张量的名称
        if (t->grad) {
            ggml_format_name(t->grad, "%s->grad", n);
        }
    };

    // 创建一个1维张量KQ_pos，用于存储位置信息
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, N);
    // 分配内存给KQ_pos
    ggml_allocr_alloc(alloc, KQ_pos);
    // 如果内存不是度量，初始化KQ_pos的数据
    if (!ggml_allocr_is_measure(alloc)) {
        int * data = (int *) KQ_pos->data;
        for (int i = 0; i < N; ++i) {
            data[i] = n_past + i;
        }
    }

    // 定义一个lambda函数rope，用于处理绳子的参数
    auto rope = [ctx, KQ_pos, n_rot, n_ctx, rope_freq_base, rope_freq_scale]
                (struct ggml_tensor * t) -> struct ggml_tensor * {
        // 不捕获这些变量，以消除警告
        const int rope_mode = 0;

        return ggml_rope_custom(
            ctx, t, KQ_pos, n_rot, rope_mode, n_ctx, 0, rope_freq_base, rope_freq_scale, 0.0f, 1.0f, 0.0f, 0.0f
        );
    };

    // 设置tokens_input的名称为"tokens_input"
    set_name(tokens_input, "tokens_input");
    // 设置targets的名称为"targets"
    set_name(targets,      "targets");

    // 断言tokens_input的类型为GGML_TYPE_I32
    GGML_ASSERT(tokens_input->type == GGML_TYPE_I32);
    // 重塑tokens_input为1维张量t00，并设置名称为"t00"，并断言其形状为N*n_batch
    struct ggml_tensor * t00 = ggml_reshape_1d(ctx, tokens_input, N*n_batch);  set_name(t00, "t00"); assert_shape_1d(t00, N*n_batch);
    // 从模型的tok_embeddings中获取t00的行，设置名称为"t01"，并断言其形状为n_embd, N*n_batch
    struct ggml_tensor * t01 = ggml_get_rows(ctx, model->tok_embeddings, t00); set_name(t01, "t01"); assert_shape_2d(t01, n_embd, N*n_batch);

    // 初始化cur为t01
    struct ggml_tensor * cur = t01;

    // 创建一个存储张量的vector checkpoints，并将tokens_input、targets、t00、t01加入其中
    std::vector<struct ggml_tensor *> checkpoints;
    checkpoints.push_back(tokens_input);
    checkpoints.push_back(targets);
    checkpoints.push_back(t00);
    checkpoints.push_back(t01);

    // 初始化kv_scale为NULL
    struct ggml_tensor * kv_scale = NULL;
    # 如果不启用闪存注意力，则计算 kv_scale
    if (!enable_flash_attn) {
        kv_scale = ggml_new_f32(ctx, 1.0f/sqrtf(float(n_embd)/n_head));
    }

    # 创建并命名张量 t31，进行 RMS 标准化
    struct ggml_tensor * t31   = ggml_rms_norm          (ctx, cur, f_norm_rms_eps);                 set_name(t31, "t31");     assert_shape_2d(t31, n_embd, N*n_batch);
    # 创建并命名张量 t32，将 model->norm 重复 N*n_batch 次
    struct ggml_tensor * t32   = ggml_repeat            (ctx, model->norm, t31);                    set_name(t32, "t32");     assert_shape_2d(t32, n_embd, N*n_batch);
    # 创建并命名张量 t33，对 t32 和 t31 进行逐元素相乘
    struct ggml_tensor * t33   = ggml_mul               (ctx, t32, t31);                            set_name(t33, "t33");     assert_shape_2d(t33, n_embd, N*n_batch);
    # 创建并命名张量 t34，对 model->output 和 t33 进行矩阵相乘
    struct ggml_tensor * t34   = ggml_mul_mat           (ctx, model->output, t33);                  set_name(t34, "t34");     assert_shape_2d(t34, n_vocab, N*n_batch);
    # 创建并命名张量 t35，将 t34 重塑为三维张量
    struct ggml_tensor * t35   = ggml_reshape_3d        (ctx, t34, n_vocab, N, n_batch);            set_name(t35, "t35");     assert_shape_3d(t35, n_vocab, N, n_batch);
    # 创建并命名张量 t36，计算交叉熵损失
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

    # 如果启用检查点，则构建反向传播梯度检查点
    if (enable_checkpointing) {
        ggml_build_backward_gradient_checkpointing(ctx, gf, gb, gb_tmp, checkpoints.data(), (int) checkpoints.size());
    } else {
        # 否则，复制前向传播图到反向传播图，并构建反向传播图
        ggml_graph_cpy(gf, gb);
        ggml_build_backward_expand(ctx, gf, gb, true);
    }
    // 如果分配了内存
    if (alloc) {
        // 确保一些张量不会被重新分配，通过插入依赖于它们的新临时节点
        int n_leafs_before = gb->n_leafs;
        int n_nodes_before = gb->n_nodes;
        // 创建一个值为1.0的新张量
        struct ggml_tensor * one = ggml_new_f32(ctx, 1.0f);
        // 输出张量
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t35, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t36, one));
        // 输入梯度
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t36->grad, one));
        // KQ_pos
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, KQ_pos, one));
        // 确保 t36 的梯度数据和视图源为空
        GGML_ASSERT(t36->grad->data == NULL && t36->grad->view_src == NULL);

        // 分配 t36 的梯度
        ggml_allocr_alloc(alloc, t36->grad);

        // 在一个块中分配检查点以减少内存碎片化
        // 注意：它们将以相反的顺序被释放
        for (int i = 0; i < (int) checkpoints.size(); ++i) {
            if (checkpoints[i]->data == NULL && checkpoints[i]->view_src == NULL) {
                ggml_allocr_alloc(alloc, checkpoints[i]);
            }
        }

        // 分配图中的节点和叶子
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

    // 将 t35 赋值给 logits
    *logits = t35;
    // 返回 t36
    return t36;
// 定义宏 GGUF_GET_KEY，用于获取指定类型的键值对应的数值，并进行类型检查和错误处理
#define GGUF_GET_KEY(ctx, dst, func, type, req, key) \
do { \
    // 将键名转换为字符串类型
    const std::string skey(key); \
    // 在上下文中查找键名对应的键 ID
    const int kid = gguf_find_key(ctx, skey.c_str()); \
    // 如果找到了键 ID
    if (kid >= 0) { \
        // 获取键值对应的类型
        enum gguf_type ktype = gguf_get_kv_type(ctx, kid); \
        // 如果类型不匹配
        if (ktype != (type)) { \
            // 抛出错误，指出键的类型错误
            die_fmt("key %s has wrong type: %s", skey.c_str(), gguf_type_name(ktype)); \
        } \
        // 调用指定的函数获取键值，并赋值给目标变量
        (dst) = func(ctx, kid); \
    } else if (req) { \
        // 如果要求必须找到键，但未找到
        die_fmt("key not found in model: %s", skey.c_str()); \
    } \
} while (0)

// 定义静态函数 load_llama_model_gguf，用于加载 llama 模型的 GGUF 数据
static void load_llama_model_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model) {
    // 注意：gguf_context 必须使用 f_ggml_ctx 进行初始化，并且 no_alloc=false，否则无法读取张量数据
    std::string arch;

    // 定义存储键名的缓冲区
    std::vector<char> keybuf;
    keybuf.resize(512);
    // 定义 lambda 函数 kv，用于格式化键名
    auto kv = [&arch, &keybuf](const char * key) -> const char * {
        snprintf(keybuf.data(), keybuf.size(), key, arch.c_str());
        return keybuf.data();
    };

    // 定义存储张量名的缓冲区
    std::vector<char> tn_buf;
    tn_buf.resize(GGML_MAX_NAME);
    // 定义 lambda 函数 tn，用于格式化张量名
    auto tn = [&tn_buf](const char * key) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", key);
        return tn_buf.data();
    };
    // 定义 lambda 函数 tni，用于格式化张量名（带索引）
    auto tni = [&tn_buf](const char * key, int bid) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), key, bid);
        std::string s = tn_buf.data();
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", s.c_str());
        return tn_buf.data();
    };

    // 获取指定键名对应的字符串类型值，并赋值给 arch
    GGUF_GET_KEY(fctx, arch, gguf_get_val_str, GGUF_TYPE_STRING, true, LLM_KV_GENERAL_ARCHITECTURE);
    // 断言 arch 的值为 "llama"
    GGML_ASSERT(arch == "llama");

    // 获取指定键名对应的无符号 32 位整数类型值，并赋值给 ftype_u
    uint32_t ftype_u;
    GGUF_GET_KEY(fctx, ftype_u, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_GENERAL_FILE_TYPE);
    // 断言 ftype_u 的值为 LLAMA_FTYPE_ALL_F32

    // n_ctx 在早期的检查点文件版本中未保存，因此在此处将其设置为可选
    # 从上下文中获取模型参数中的上下文长度，并存储到模型参数中
    GGUF_GET_KEY(fctx, model->hparams.n_ctx,   gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_CONTEXT_LENGTH));

    # 从上下文中获取模型参数中的嵌入长度，并存储到模型参数中
    GGUF_GET_KEY(fctx, model->hparams.n_embd,  gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_EMBEDDING_LENGTH));
    # 从上下文中获取模型参数中的前馈神经网络长度，并存储到模型参数中
    GGUF_GET_KEY(fctx, model->hparams.n_ff,    gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_FEED_FORWARD_LENGTH));
    # 从上下文中获取模型参数中的注意力头数，并存储到模型参数中
    GGUF_GET_KEY(fctx, model->hparams.n_head,  gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_ATTENTION_HEAD_COUNT));
    # 从上下文中获取模型参数中的层数，并存储到模型参数中
    GGUF_GET_KEY(fctx, model->hparams.n_layer, gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_BLOCK_COUNT));

    # 计算模型参数中的旋转维度
    model->hparams.n_rot = model->hparams.n_embd / model->hparams.n_head;
    # 从上下文中获取模型参数中的绳索维度，并存储到模型参数中
    GGUF_GET_KEY(fctx, model->hparams.n_rot,   gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ROPE_DIMENSION_COUNT));

    # 初始化绳索频率缩放值
    float rope_freq_scale = 1.0f;
    # 从上下文中获取模型参数中的注意力层归一化 RMS 误差，并存储到模型参数中
    GGUF_GET_KEY(fctx, model->hparams.f_norm_rms_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS));
    # 从上下文中获取模型参数中的绳索频率基础值，并存储到模型参数中
    GGUF_GET_KEY(fctx, model->hparams.rope_freq_base, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_FREQ_BASE));
    # 从上下文中获取绳索频率缩放值，并存储到绳索频率缩放变量中
    GGUF_GET_KEY(fctx, rope_freq_scale, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_SCALE_LINEAR));
    # 如果绳索频率缩放值不等于1.0，则更新模型参数中的绳索频率缩放值
    if (rope_freq_scale != 1.0f) {
        model->hparams.rope_freq_scale = 1.0f / rope_freq_scale;
    }

    # 初始化模型
    init_model(model);

    # 通过名称从上下文中复制模型的 token 嵌入张量
    copy_tensor_by_name(model->tok_embeddings, f_ggml_ctx, tn(LLM_TENSOR_TOKEN_EMBD));
    # 通过名称从上下文中复制模型的归一化张量
    copy_tensor_by_name(model->norm,           f_ggml_ctx, tn(LLM_TENSOR_OUTPUT_NORM));
    # 通过名称从上下文中复制模型的输出张量
    copy_tensor_by_name(model->output,         f_ggml_ctx, tn(LLM_TENSOR_OUTPUT));
    # 遍历模型的每一层
    for (uint32_t i = 0; i < model->hparams.n_layer; ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];

        # 通过名称复制注意力规范化层的张量
        copy_tensor_by_name(layer.attention_norm, f_ggml_ctx, tni(LLM_TENSOR_ATTN_NORM, i));
        # 通过名称复制注意力查询张量
        copy_tensor_by_name(layer.wq,             f_ggml_ctx, tni(LLM_TENSOR_ATTN_Q, i));
        # 通过名称复制注意力键张量
        copy_tensor_by_name(layer.wk,             f_ggml_ctx, tni(LLM_TENSOR_ATTN_K, i));
        # 通过名称复制注意力值张量
        copy_tensor_by_name(layer.wv,             f_ggml_ctx, tni(LLM_TENSOR_ATTN_V, i));
        # 通过名称复制注意力输出张量
        copy_tensor_by_name(layer.wo,             f_ggml_ctx, tni(LLM_TENSOR_ATTN_OUT, i));
        # 通过名称复制前馈神经网络规范化层的张量
        copy_tensor_by_name(layer.ffn_norm,       f_ggml_ctx, tni(LLM_TENSOR_FFN_NORM, i));
        # 通过名称复制前馈神经网络门控张量
        copy_tensor_by_name(layer.w1,             f_ggml_ctx, tni(LLM_TENSOR_FFN_GATE, i));
        # 通过名称复制前馈神经网络下采样张量
        copy_tensor_by_name(layer.w2,             f_ggml_ctx, tni(LLM_TENSOR_FFN_DOWN, i));
        # 通过名称复制前馈神经网络上采样张量
        copy_tensor_by_name(layer.w3,             f_ggml_ctx, tni(LLM_TENSOR_FFN_UP, i));
    }
// 保存 LLAMA 模型到 GGUF 文件
static void save_llama_model_gguf(struct gguf_context * fctx, const char * fn_vocab_model, struct my_llama_model * model) {
    const char * arch = "llama"; // 定义 LLAMA 架构
    enum llama_ftype ftype = LLAMA_FTYPE_ALL_F32; // 定义 LLAMA 文件类型为所有浮点数

    std::vector<char> keybuf; // 创建字符向量 keybuf
    keybuf.resize(512); // 调整 keybuf 的大小为 512
    auto kv = [arch, &keybuf](const char * key) -> const char * { // 创建 lambda 函数 kv，用于格式化 key
        snprintf(keybuf.data(), keybuf.size(), key, arch); // 使用 snprintf 格式化 key，并将结果存储在 keybuf 中
        return keybuf.data(); // 返回格式化后的 key
    };

    // 设置 LLAMA 架构
    gguf_set_val_str(fctx, LLM_KV_GENERAL_ARCHITECTURE, arch);
    // 设置 LLAMA 文件类型
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

    // 从 vocab_model gguf 文件中复制设置词汇表
    }

    // 添加张量
    gguf_add_tensor(fctx, model->tok_embeddings);
    gguf_add_tensor(fctx, model->norm);
    gguf_add_tensor(fctx, model->output);
    # 遍历模型的每一层
    for (uint32_t i = 0; i < model->hparams.n_layer; ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];

        # 将当前层的注意力规范化张量添加到上下文中
        gguf_add_tensor(fctx, layer.attention_norm);
        # 将当前层的查询权重张量添加到上下文中
        gguf_add_tensor(fctx, layer.wq);
        # 将当前层的键权重张量添加到上下文中
        gguf_add_tensor(fctx, layer.wk);
        # 将当前层的值权重张量添加到上下文中
        gguf_add_tensor(fctx, layer.wv);
        # 将当前层的输出权重张量添加到上下文中
        gguf_add_tensor(fctx, layer.wo);
        # 将当前层的前馈神经网络规范化张量添加到上下文中
        gguf_add_tensor(fctx, layer.ffn_norm);
        # 将当前层的第一个前馈神经网络权重张量添加到上下文中
        gguf_add_tensor(fctx, layer.w1);
        # 将当前层的第二个前馈神经网络权重张量添加到上下文中
        gguf_add_tensor(fctx, layer.w2);
        # 将当前层的第三个前馈神经网络权重张量添加到上下文中
        gguf_add_tensor(fctx, layer.w3);
    }
// 保存 LLAMA 模型文件，包括词汇模型文件名和模型结构
static void save_llama_model_file(const char * filename, const char * fn_vocab_model, struct my_llama_model * model) {
    // 打印保存信息
    printf("%s: saving to %s\n", __func__, filename);
    // 初始化 GGUF 上下文
    struct gguf_context * fctx = gguf_init_empty();

    // 保存 LLAMA 模型到 GGUF 上下文
    save_llama_model_gguf(fctx, fn_vocab_model, model);

    // 写入文件
    const bool only_meta = false;
    gguf_write_to_file(fctx, filename, only_meta);
    // 释放 GGUF 上下文
    gguf_free(fctx);
}

// 从 GGUF 上下文加载检查点
static void load_checkpoint_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model, struct train_state * train) {
    // 从 GGUF 上下文加载 LLAMA 模型
    load_llama_model_gguf(fctx, f_ggml_ctx, model);
    // 如果加载训练状态成功
    if (load_train_state_gguf(fctx, f_ggml_ctx, train)) {
        // 获取训练类型
        std::string train_type = LLM_KV_TRAINING_TYPE_TRAIN_MODEL;
        GGUF_GET_KEY(fctx, train_type, gguf_get_val_str, GGUF_TYPE_STRING, false, LLM_KV_TRAINING_TYPE);
        // 断言训练类型为训练模型
        GGML_ASSERT(train_type == LLM_KV_TRAINING_TYPE_TRAIN_MODEL);
    } else {
        // 打印加载 LLAMA 模型作为检查点的信息
        printf("%s: loaded llama model as checkpoint\n", __func__);
    }
}

// 保存 GGUF 上下文到检查点文件
static void save_checkpoint_gguf(struct gguf_context * fctx, const char * fn_vocab_model, struct my_llama_model * model, struct train_state * train) {
    // 设置训练类型为训练模型
    gguf_set_val_str(fctx, LLM_KV_TRAINING_TYPE, LLM_KV_TRAINING_TYPE_TRAIN_MODEL);
    // 保存 LLAMA 模型到 GGUF 上下文
    save_llama_model_gguf(fctx, fn_vocab_model, model);
    // 保存训练状态到 GGUF 上下文
    save_train_state_gguf(fctx, train);
}

// 从文件加载检查点
static bool load_checkpoint_file(const char * filename, struct my_llama_model * model, struct train_state * train) {
    // 初始化 GGML 上下文和 GGUF 初始化参数
    struct ggml_context * f_ggml_ctx;
    struct gguf_init_params params;
    params.no_alloc = false;
    params.ctx = &f_ggml_ctx;
    // 从文件初始化 GGUF 上下文
    struct gguf_context * fctx = gguf_init_from_file(filename, params);
    // 如果初始化失败则返回 false
    if (fctx == NULL) {
        return false;
    }

    // 从 GGUF 上下文加载检查点
    load_checkpoint_gguf(fctx, f_ggml_ctx, model, train);

    return true;
}

// 保存检查点到文件
static void save_checkpoint_file(const char * filename, const char * fn_vocab_model, struct my_llama_model * model, struct train_state * train) {
    // 打印保存信息
    printf("%s: saving to %s\n", __func__, filename);
    // 初始化一个空的 gguf 上下文结构体
    struct gguf_context * fctx = gguf_init_empty();

    // 保存 gguf 上下文结构体的检查点，包括词汇模型、模型和训练数据
    save_checkpoint_gguf(fctx, fn_vocab_model, model, train);

    // 写入文件
    const bool only_meta = false;
    // 将 gguf 上下文结构体的内容写入文件，包括元数据和数据
    gguf_write_to_file(fctx, filename, only_meta);
    // 释放 gguf 上下文结构体的内存
    gguf_free(fctx);
}

# 定义一个结构体，包含训练参数的通用部分和特定部分
struct train_params {
    struct train_params_common common;

    const char * fn_vocab_model;  # 词汇模型文件名
    const char * fn_model_out;     # 输出模型文件名

    bool only_write_model;         # 是否只写模型

    int n_ctx;                     # 上下文大小
    int n_embd;                    # 嵌入大小
    int n_head;                    # 头数
    int n_layer;                   # 层数
    int n_ff;                      # 前馈大小

    float f_norm_rms_eps;          # 归一化 RMS 误差
    float rope_freq_base;          # ROPE 频率基数
    float rope_freq_scale;         # ROPE 频率比例
};

# 获取默认的训练参数
static struct train_params get_default_train_params() {
    struct train_params params;
    params.common = get_default_train_params_common();  # 获取通用训练参数
    params.fn_vocab_model    = "ggml-vic7b-uncensored-q4_0.bin";  # 设置词汇模型文件名
    params.fn_model_out      = "ggml-checkpoint-f32.bin";           # 设置输出模型文件名

    params.only_write_model = false;  # 设置是否只写模型为假

    params.n_ctx      =  128;  # 设置上下文大小
    params.n_embd     =  256;  # 设置嵌入大小
    params.n_head     =    8;  # 设置头数
    params.n_layer    =   16;  # 设置层数
    params.n_ff       =  768;  # 设置前馈大小

    params.f_norm_rms_eps  = 1e-5f;   # 设置归一化 RMS 误差
    params.rope_freq_base  = 10000.0f;  # 设置 ROPE 频率基数
    params.rope_freq_scale = 1.0f;     # 设置 ROPE 频率比例

    return params;  # 返回参数
}

# 打印训练使用说明
static void train_print_usage(int argc, char ** argv, const struct train_params * params) {
    fprintf(stderr, "usage: %s [options]\n", argv[0]);  # 打印使用说明
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help                 show this help message and exit\n");

    fprintf(stderr, "  --vocab-model FNAME        model path from which to load vocab (default '%s')\n", params->fn_vocab_model);  # 打印词汇模型文件名
    fprintf(stderr, "  --model-out FNAME          path to save ggml model (default '%s')\n", params->fn_model_out);  # 打印输出模型文件名
    fprintf(stderr, "  --only-write-model         only save llama model, don't do any training. use this if you only want to convert a checkpoint to a model.\n");  # 打印是否只写模型的说明
    fprintf(stderr, "  --embd N                   Embedding size used for new models (default %d)\n", params->n_embd);  # 打印嵌入大小
    fprintf(stderr, "  --ff N                     Feedforward size used for new models. (default %d)\n", params->n_ff);  # 打印前馈大小
    fprintf(stderr, "  --head N                   Number of heads for new models (default %d)\n", params->n_head);  # 打印头数
    # 打印参数中的层数，默认为params->n_layer
    fprintf(stderr, "  --layer N                  Number of layers for new models (default %d)\n", params->n_layer);
    # 打印参数中的 RMS-Norm epsilon 值，默认为params->f_norm_rms_eps
    fprintf(stderr, "  --norm-rms-eps F           RMS-Norm epsilon value (default %f)\n", params->f_norm_rms_eps);
    # 打印参数中的 ROPE 频率基数，默认为params->rope_freq_base
    fprintf(stderr, "  --rope-freq-base F         Frequency base for ROPE (default %f)\n", params->rope_freq_base);
    # 打印参数中的 ROPE 频率比例，默认为params->rope_freq_scale
    fprintf(stderr, "  --rope-freq-scale F        Frequency scale for ROPE (default %f)\n", params->rope_freq_scale);

    # 打印通用训练用法
    print_common_train_usage(argc, argv, &params->common);
// 解析训练参数，返回是否解析成功
static bool train_params_parse(int argc, char ** argv, struct train_params * params) {
    // 初始化参数无效标志为 false
    bool invalid_param = false;
    // 定义字符串类型的参数
    std::string arg;
    // 获取默认的训练参数
    struct train_params default_params = get_default_train_params();
    // 定义参数前缀
    const std::string arg_prefix = "--";

    // 如果存在无效参数
    if (invalid_param) {
        // 打印错误信息
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        // 打印训练使用方法
        train_print_usage(argc, argv, &default_params);
        // 退出程序
        exit(1);
    }
    // 完成处理训练参数
    finish_processing_train_args(&params->common);

    // 返回解析成功
    return true;
}

// 保存训练文件的数据结构
struct save_train_files_data {
    const char            * fn_checkpoint_out;
    const char            * fn_model_out;
    const char            * fn_vocab_model;
    const char            * pattern_fn_it;
    const char            * fn_latest;
    struct my_llama_model * model;
};

// 保存训练文件
static void save_train_files(void * vdata, struct train_state * train) {
    // 获取保存训练文件的数据
    struct save_train_files_data * data   = (struct save_train_files_data *) vdata;
    // 获取迭代次数
    int64_t iter = train->opt->iter;

    // 如果检查点输出文件名长度大于 0
    if (strlen(data->fn_checkpoint_out) > 0) {
        // 保存检查点文件
        save_checkpoint_file(get_train_filename(data->fn_checkpoint_out, data->pattern_fn_it, data->fn_latest, iter).c_str(), data->fn_vocab_model, data->model, train);
        // 保存检查点文件
        save_checkpoint_file(get_train_filename(data->fn_checkpoint_out, data->pattern_fn_it, data->fn_latest, -1  ).c_str(), data->fn_vocab_model, data->model, train);
    }
    // 如果模型输出文件名长度大于 0
    if (strlen(data->fn_model_out) > 0) {
        // 保存模型文件
        save_llama_model_file(get_train_filename(data->fn_model_out, data->pattern_fn_it, data->fn_latest, iter).c_str(), data->fn_vocab_model, data->model);
        // 保存模型文件
        save_llama_model_file(get_train_filename(data->fn_model_out, data->pattern_fn_it, data->fn_latest, -1  ).c_str(), data->fn_vocab_model, data->model);
    }
}

// 获取参数数量
static int64_t get_parameter_count(struct my_llama_model* model) {
    // 初始化参数数量为 0
    int64_t nx = 0;
    // 参数数量加上 token embeddings 的元素数量
    nx += ggml_nelements(model->tok_embeddings);
    // 参数数量加上 norm 的元素数量
    nx += ggml_nelements(model->norm);
    // 参数数量加上 output 的元素数量
    nx += ggml_nelements(model->output);
    // 遍历模型的每一层
    for (uint32_t i = 0; i < model->layers.size(); ++i) {
        // 获取当前层的引用
        auto & layer = model->layers[i];
        // 计算注意力规范化层的元素数量并累加到 nx
        nx += ggml_nelements(layer.attention_norm);
        // 计算查询权重矩阵的元素数量并累加到 nx
        nx += ggml_nelements(layer.wq);
        // 计算键权重矩阵的元素数量并累加到 nx
        nx += ggml_nelements(layer.wk);
        // 计算值权重矩阵的元素数量并累加到 nx
        nx += ggml_nelements(layer.wv);
        // 计算输出权重矩阵的元素数量并累加到 nx
        nx += ggml_nelements(layer.wo);
        // 计算前馈神经网络规范化层的元素数量并累加到 nx
        nx += ggml_nelements(layer.ffn_norm);
        // 计算前馈神经网络第一层权重矩阵的元素数量并累加到 nx
        nx += ggml_nelements(layer.w1);
        // 计算前馈神经网络第二层权重矩阵的元素数量并累加到 nx
        nx += ggml_nelements(layer.w2);
        // 计算前馈神经网络第三层权重矩阵的元素数量并累加到 nx
        nx += ggml_nelements(layer.w3);
    }
    // 返回总的元素数量
    return nx;
// 主函数，接受命令行参数并初始化训练参数结构体
int main(int argc, char ** argv) {
    // 获取默认的训练参数
    struct train_params params = get_default_train_params();

    // 解析命令行参数到训练参数结构体，如果解析失败则返回1
    if (!train_params_parse(argc, argv, &params)) {
        return 1;
    }

    // 如果参数中的随机种子为默认值，则设置为当前时间的时间戳
    if (params.common.seed == LLAMA_DEFAULT_SEED) {
        params.common.seed = time(NULL);
    }
    // 打印函数名和随机种子
    printf("%s: seed: %u\n", __func__, params.common.seed);
    // 根据随机种子初始化随机数生成器
    srand(params.common.seed);

    // 初始化模型参数，只包括词汇表
    struct llama_model_params mparams = llama_model_default_params();
    mparams.vocab_only = true;

    // 初始化上下文参数
    struct llama_context_params cparams = llama_context_default_params();

    // 从文件中加载模型
    struct llama_model * lmodel = llama_load_model_from_file(params.fn_vocab_model, mparams);
    // 使用模型初始化上下文
    struct llama_context * lctx = llama_new_context_with_model(lmodel, cparams);

    // 初始化自定义的LLAMA模型
    struct my_llama_model model;
    model.hparams.n_vocab = llama_n_vocab(lmodel);
    model.hparams.n_ctx   = params.common.n_ctx;
    model.hparams.n_embd  = params.n_embd;
    model.hparams.n_head  = params.n_head;
    model.hparams.n_layer = params.n_layer;
    model.hparams.n_ff    = params.n_ff;
    // llama.cpp要求n_rot必须等于n_embd / n_head
    model.hparams.n_rot   = model.hparams.n_embd / model.hparams.n_head;
    model.hparams.f_norm_rms_eps  = params.f_norm_rms_eps;
    model.hparams.rope_freq_base  = params.rope_freq_base;
    model.hparams.rope_freq_scale = params.rope_freq_scale;

    // 初始化训练状态
    struct train_state      * train = init_train_state();
    // 初始化优化器上下文
    struct ggml_opt_context * opt   = train->opt;

    // 从命令行设置优化器参数
    opt->params = ggml_opt_default_params(GGML_OPT_ADAM);
    opt->params.print_forward_graph     = false;
    opt->params.print_backward_graph    = false;
    opt->params.graph_size              = LLAMA_TRAIN_MAX_NODES;
    opt->params.n_threads               = params.common.n_threads;
    opt->params.past                    = params.common.opt_past;
    opt->params.delta                   = params.common.opt_delta;
    opt->params.max_no_improvement      = params.common.opt_max_no_improvement;
    // 设置梯度累积的数量
    opt->params.n_gradient_accumulation = params.common.n_gradient_accumulation;
    // 设置 Adam 优化器的迭代次数
    opt->params.adam.n_iter             = params.common.adam_n_iter;
    // 设置 Adam 优化器的调度参数
    opt->params.adam.sched              = 1.0f;
    // 设置 Adam 优化器的学习率
    opt->params.adam.alpha              = params.common.adam_alpha;
    // 设置 Adam 优化器的衰减率
    opt->params.adam.decay              = params.common.adam_decay;
    // 设置 Adam 优化器的最小维度衰减
    opt->params.adam.decay_min_ndim     = params.common.adam_decay_min_ndim;
    // 设置 Adam 优化器的 beta1 参数
    opt->params.adam.beta1              = params.common.adam_beta1;
    // 设置 Adam 优化器的 beta2 参数
    opt->params.adam.beta2              = params.common.adam_beta2;
    // 设置 Adam 优化器的梯度裁剪参数
    opt->params.adam.gclip              = params.common.adam_gclip;
    // 设置 Adam 优化器的 epsilon 参数
    opt->params.adam.eps_f              = params.common.adam_eps_f;

    // 打印初始化模型的信息
    printf("%s: init model\n", __func__);
    // 加载检查点文件，判断文件是否存在，并返回是否存在的布尔值
    bool existed = load_checkpoint_file(params.common.fn_checkpoint_in, &model, train);
    // 如果检查点文件存在
    if (existed) {
        // 如果用户提供了自定义的 n_ctx，则覆盖模型的 n_ctx
        if (params.common.custom_n_ctx) {
            model.hparams.n_ctx = params.common.n_ctx;
        }

        // 判断优化器参数 past 是否与检查点文件中的参数不同
        const bool opt_past_changed = opt->params.past != params.common.opt_past;

        // 如果 past 参数发生了改变
        if (opt_past_changed) {
            // 输出错误信息并终止程序
            die("Optimizer parameter '--opt-past N' differs from checkpoint file. To use different value train from scratch with empty input checkpoint, e.g --checkpoint-in ''. Aborting");
            // 需要丢弃先前的优化器 past 函数值统计，并使用新的形状进行优化器初始化
            // TODO
        }
    } else {
        // 初始化模型
        init_model(&model);
        // 随机初始化模型
        randomize_model(&model, params.common.seed, 0.0f, 1.0f, -1.0f, +1.0f);
        // 如果不仅仅是写模型，则进行优化器初始化
        if (!params.only_write_model) {
            ggml_opt_init(opt->ctx, opt, opt->params, get_parameter_count(&model));
        }
    }
    // 设置优化器的迭代次数为训练的迭代次数
    opt->iter = train->train_its;

    // 打印模型参数
    print_params(&model.hparams);
    // 打印总的训练迭代次数
    printf("%s: total train_iterations %llu\n", __func__, (long long unsigned) train->train_its);
    // 打印已观察到的训练样本数量
    printf("%s: seen train_samples     %llu\n", __func__, (long long unsigned) train->train_samples);
    // 打印训练标记数
    printf("%s: seen train_tokens      %llu\n", __func__, (long long unsigned) train->train_tokens);
    // 打印完成的训练周期数
    printf("%s: completed train_epochs %llu\n", __func__, (long long unsigned) train->train_epochs);
    // 打印模型大小，以字节和兆字节为单位
    printf("%s: model_size = %zu bytes (%.1f MB)\n", __func__, (ggml_used_mem(model.ctx) + model.data.size()), (float) (ggml_used_mem(model.ctx) + model.data.size()) / (1024.0f*1024.0f));

    // 如果只写模型，则保存训练文件数据，并释放资源后返回
    if (params.only_write_model) {
        save_train_files_data save_data;
        save_data.fn_checkpoint_out = "";
        save_data.fn_model_out      = params.fn_model_out;
        save_data.fn_vocab_model    = params.fn_vocab_model;
        save_data.pattern_fn_it     = params.common.pattern_fn_it;
        save_data.fn_latest         = params.common.fn_latest;
        save_data.model             = &model;

        save_train_files(&save_data, train);

        free_train_state(train);
        ggml_free(model.ctx);
        llama_free(lctx);
        llama_free_model(lmodel);
        return 0;
    }

    // 打印优化器的内存大小，以字节和兆字节为单位
    printf("%s: opt_size  = %zu bytes (%.1f MB)\n", __func__, ggml_get_mem_size(opt->ctx), (float) ggml_get_mem_size(opt->ctx) / (1024.0f*1024.0f));
    // 打印优化器的迭代次数
    printf("%s: opt iter %d\n", __func__, opt->iter);

    // 初始化变量
    int n_tokens = model.hparams.n_ctx;
    int n_vocab  = model.hparams.n_vocab;
    int n_batch  = params.common.n_batch;

    // 创建存储输入数据和计算数据的向量
    std::vector<uint8_t> mem_input_data;
    std::vector<uint8_t> mem_compute_data;

    // 初始化分配器
    ggml_allocr * alloc = NULL;

    // 初始化输入张量的上下文参数
    struct ggml_init_params ctx_input_params = {
        ggml_tensor_overhead() * 2, // mem_size
        NULL,                       // mem_buffer
        true,                       // no_alloc
    };
    // 初始化输入张量的上下文
    struct ggml_context * ctx_input = ggml_init(ctx_input_params);

    // 创建输入张量
    struct ggml_tensor * tokens_input  = ggml_new_tensor_2d(ctx_input, GGML_TYPE_I32, n_tokens, n_batch);
    // 创建一个指向三维浮点类型的目标概率张量的指针，包含 n_vocab * n_tokens * n_batch 个元素
    struct ggml_tensor * target_probs  = ggml_new_tensor_3d(ctx_input, GGML_TYPE_F32, n_vocab,  n_tokens, n_batch);

    // 计算输入张量所需的内存大小
    size_t max_input_size = GGML_PAD(ggml_nbytes(tokens_input), tensor_alignment) +
                            GGML_PAD(ggml_nbytes(target_probs), tensor_alignment) +
                            tensor_alignment;
    // 打印输入大小的信息
    printf("%s: input_size = %zu bytes (%.1f MB)\n", __func__, max_input_size, (float) max_input_size / (1024.0f*1024.0f));

    // 分配输入张量的内存
    mem_input_data.resize(max_input_size);
    alloc = ggml_allocr_new(mem_input_data.data(), mem_input_data.size(), tensor_alignment);
    ggml_allocr_alloc(alloc, tokens_input);
    ggml_allocr_alloc(alloc, target_probs);
    ggml_allocr_free(alloc);

    // 创建一个上下文，用于计算张量但不包含数据
    const size_t estimated_compute_size_wo_data = (
            2*LLAMA_TRAIN_MAX_NODES*ggml_tensor_overhead() +
            (params.common.use_checkpointing ? 3 : 2)*(GGML_OBJECT_SIZE+ggml_graph_overhead_custom(LLAMA_TRAIN_MAX_NODES, true))
    );
    struct ggml_init_params ctx_compute_params = {
        estimated_compute_size_wo_data, // mem_size
        NULL,                           // mem_buffer
        true,                           // no_alloc
    };
    struct ggml_context * ctx_compute = NULL;

    struct ggml_tensor * loss   = NULL;
    struct ggml_tensor * logits = NULL;

    struct ggml_cgraph * gf     = NULL;
    struct ggml_cgraph * gb     = NULL;
    struct ggml_cgraph * gb_tmp = NULL;

    // 计算计算张量所需的内存大小
    size_t best_compute_size = SIZE_MAX;
    enum ggml_cgraph_eval_order best_order = GGML_CGRAPH_EVAL_ORDER_COUNT;
    // 找到最佳的评估顺序
    # 遍历评估顺序的枚举值
    for (unsigned order = 0; order < (unsigned) GGML_CGRAPH_EVAL_ORDER_COUNT; ++order) {
        # 初始化计算上下文
        ctx_compute = ggml_init(ctx_compute_params);
        # 创建新的度量分配器
        alloc = ggml_allocr_new_measure(tensor_alignment);
        # 创建自定义图形对象
        gf = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
        # 设置图形对象的评估顺序
        gf->order = (enum ggml_cgraph_eval_order) order;
        # 创建新的自定义图形对象
        gb = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
        # 如果使用检查点，则创建新的自定义图形对象，否则为NULL
        gb_tmp = params.common.use_checkpointing
            ? ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true)
            : NULL;
        # 构建训练图形
        loss = llama_build_train_graphs(
            &model, alloc, ctx_compute,
            gf, gb, gb_tmp,
            &logits, tokens_input, target_probs,
            n_tokens, n_batch,
            params.common.use_flash,
            params.common.use_checkpointing
        );
        # 计算最大计算大小
        size_t max_compute_size = ggml_allocr_max_size(alloc) + tensor_alignment;
        # 如果最大计算大小小于最佳计算大小，则更新最佳计算大小和评估顺序
        if (max_compute_size < best_compute_size) {
            best_compute_size = max_compute_size;
            best_order = gf->order;
        }
        # 释放度量分配器
        ggml_allocr_free(alloc);
        # 释放计算上下文
        ggml_free(ctx_compute);
    }
    # 设置最大计算大小为最佳计算大小
    size_t max_compute_size = best_compute_size;
    # 打印计算大小
    printf("%s: compute_size = %zu bytes (%.1f MB)\n", __func__, max_compute_size, (float) max_compute_size / (1024.0f*1024.0f));
    # 打印评估顺序
    printf("%s: evaluation order = %s\n", __func__,
        (best_order == GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT) ? "LEFT_TO_RIGHT" :
        (best_order == GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT) ? "RIGHT_TO_LEFT" :
        "invalid");

    # 分配计算张量
    mem_compute_data.resize(max_compute_size);
    # 初始化计算上下文
    ctx_compute = ggml_init(ctx_compute_params);
    # 创建新的度量分配器
    alloc = ggml_allocr_new(mem_compute_data.data(), mem_compute_data.size(), tensor_alignment);
    # 创建自定义图形对象
    gf = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
    # 设置图形对象的评估顺序为最佳评估顺序
    gf->order = best_order;
    # 创建自定义图形对象
    gb = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
    # 根据参数判断是否使用检查点，如果是则创建新的图，否则为 NULL
    gb_tmp = params.common.use_checkpointing
        ? ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true)
        : NULL;
    # 构建训练图
    loss = llama_build_train_graphs(
        &model, alloc, ctx_compute,
        gf, gb, gb_tmp,
        &logits, tokens_input, target_probs,
        n_tokens, n_batch,
        params.common.use_flash,
        params.common.use_checkpointing
    );
    # 释放分配的内存
    ggml_allocr_free(alloc);

    # 初始化存储训练数据的向量
    std::vector<llama_token> train_tokens;
    std::vector<size_t> train_samples_begin;
    std::vector<size_t> train_samples_size;
    # 打印信息，标记开始对训练数据进行分词
    printf("%s: tokenize training data\n", __func__);
    # 对训练数据进行分词
    tokenize_file(lctx,
            params.common.fn_train_data,
            params.common.sample_start,
            params.common.include_sample_start,
            params.common.overlapping_samples,
            n_tokens,
            train_tokens,
            train_samples_begin,
            train_samples_size);
    # 检查训练数据的分词数量是否一致
    GGML_ASSERT(train_samples_begin.size() == train_samples_size.size());

    # 打印信息，标记训练数据的分词数量
    printf("%s: number of training tokens: %zu\n", __func__, train_tokens.size());

    # 计算训练数据的哈希值
    size_t shuffle_samples_hash = compute_samples_hash(params.common.fn_train_data, train_samples_begin.data(), train_samples_size.data(), train_samples_size.size());
    # 检查训练数据是否发生变化
    const bool changed_train_data = (shuffle_samples_hash != train->shuffle_samples_hash) || (train->shuffle_sample_count != train_samples_size.size());
    # 如果训练数据发生变化，则打印信息
    if (changed_train_data) {
        printf("%s: train data seems to have changed. restarting shuffled epoch.\n", __func__);
    }
    # 如果强制重新洗牌，则打印信息
    if (params.common.force_reshuffle) {
        printf("%s: forced reshuffling of data. restarting with newly shuffled epoch.\n", __func__);
    }
    # 如果训练数据的洗牌状态为空，或者训练数据发生改变，或者强制重新洗牌，则执行以下操作
    if ((train->shuffle_rng_state_current == "") || changed_train_data || params.common.force_reshuffle) {
        # 根据参数的种子生成随机数发生器的状态
        train->shuffle_rng_state_current = mt19937_seed_to_state(params.common.seed);
        # 记录训练样本的数量
        train->shuffle_sample_count = train_samples_size.size();
        # 初始化下一个要洗牌的样本索引
        train->shuffle_next_sample = 0;
        # 记录洗牌前的样本哈希值
        train->shuffle_samples_hash = shuffle_samples_hash;
    }
    # 初始化存储洗牌后样本偏移量的向量
    std::vector<size_t> train_shuffled_samples_offs;
    # 初始化存储洗牌后样本起始位置的向量
    std::vector<size_t> train_shuffled_samples_begin;
    # 初始化存储洗牌后样本大小的向量
    std::vector<size_t> train_shuffled_samples_size;
    # 调整存储洗牌后样本偏移量的向量的大小
    train_shuffled_samples_offs.resize(train_samples_begin.size());
    # 调整存储洗牌后样本起始位置的向量的大小
    train_shuffled_samples_begin.resize(train_samples_begin.size());
    # 调整存储洗牌后样本大小的向量的大小
    train_shuffled_samples_size.resize(train_samples_size.size());
    # 使用洗牌函数对训练样本进行洗牌，并返回下一个随机数发生器的状态
    train->shuffle_rng_state_next = shuffle_samples(
        train->shuffle_rng_state_current,
        train_shuffled_samples_offs.data(),
        train_shuffled_samples_begin.data(),
        train_shuffled_samples_size.data(),
        train_samples_begin.data(),
        train_samples_size.data(),
        train_samples_size.size());
    # 打印开始训练的提示信息
    printf("%s: begin training\n", __func__);

    # 初始化保存训练文件数据的结构体
    save_train_files_data save_data;
    # 设置检查点输出文件名
    save_data.fn_checkpoint_out = params.common.fn_checkpoint_out;
    # 设置模型输出文件名
    save_data.fn_model_out      = params.fn_model_out;
    # 设置词汇模型文件名
    save_data.fn_vocab_model    = params.fn_vocab_model;
    # 设置迭代模式文件名的模式
    save_data.pattern_fn_it     = params.common.pattern_fn_it;
    # 设置最新模型文件名
    save_data.fn_latest         = params.common.fn_latest;
    # 设置模型指针
    save_data.model             = &model;

    # 初始化训练选项回调数据的结构体
    struct train_opt_callback_data opt_cb_data;
    # 设置参数指针
    opt_cb_data.params                 = &params.common;
    # 设置训练指针
    opt_cb_data.train                  = train;
    # 设置保存回调函数指针
    opt_cb_data.save_cb                = &save_train_files;
    # 设置保存数据指针
    opt_cb_data.save_data              = &save_data;
    # 设置语言环境指针
    opt_cb_data.lctx                   = lctx;
    # 设置最后保存的迭代次数
    opt_cb_data.last_save_iter         = opt->iter;
    # 设置训练标记数据的指针
    opt_cb_data.tokens_data            = train_tokens.data();
    # 设置训练标记数据的大小
    opt_cb_data.tokens_size            = train_tokens.size();
    // 设置优化回调数据的样本起始位置
    opt_cb_data.samples_begin          = train_samples_begin.data();
    // 设置优化回调数据的样本大小
    opt_cb_data.samples_size           = train_samples_size.data();
    // 设置优化回调数据的打乱样本偏移量
    opt_cb_data.shuffled_samples_offs  = train_shuffled_samples_offs.data();
    // 设置优化回调数据的打乱样本起始位置
    opt_cb_data.shuffled_samples_begin = train_shuffled_samples_begin.data();
    // 设置优化回调数据的打乱样本大小
    opt_cb_data.shuffled_samples_size  = train_shuffled_samples_size.data();
    // 设置优化回调数据的样本数量
    opt_cb_data.samples_count          = train_samples_size.size();
    // 设置优化回调数据的输入 tokens
    opt_cb_data.tokens_input           = tokens_input;
    // 设置优化回调数据的目标概率
    opt_cb_data.target_probs           = target_probs;
    // 设置优化回调数据的首次迭代次数
    opt_cb_data.first_iter             = opt->iter;
    // 设置优化回调数据的首次训练周期
    opt_cb_data.first_epoch            = train->train_epochs;
    // 设置优化回调数据的最后一个周期的迭代次数
    opt_cb_data.iter_at_last_epoch     = -1;
    // 设置优化回调数据的最后时间
    opt_cb_data.last_time              = ggml_time_ms();
    // 设置优化回调数据的每次迭代所需的毫秒数
    opt_cb_data.millis_per_iter        = 0.0;

    // 计算工作缓冲区所需的内存大小
    size_t max_work_size = ggml_graph_plan(gb, params.common.n_threads).work_size + GGML_OBJECT_SIZE;
    printf("%s: work_size = %zu bytes (%.1f MB)\n", __func__, max_work_size, (float) max_work_size / (1024.0f*1024.0f));

    // 设置工作缓冲区的上下文
    struct ggml_init_params ctx_work_params = {
        max_work_size, // mem_size
        NULL,          // mem_buffer
        false,         // no_alloc
    };
    // 初始化工作缓冲区的上下文
    struct ggml_context * ctx_work = ggml_init(ctx_work_params);

    // 记录开始时间
    int64_t t0 = ggml_time_ms();

    // 恢复优化器并进行训练
    ggml_opt_resume_g(ctx_work, opt, loss, gf, gb, &train_opt_callback, (void *) &opt_cb_data);

    // 释放工作缓冲区的内存
    ggml_free(ctx_work);
    // 释放计算上下文的内存
    ggml_free(ctx_compute);
    // 释放输入上下文的内存
    ggml_free(ctx_input);

    // 记录结束时间
    int64_t t1 = ggml_time_ms();
    // 打印总训练时间
    printf("%s: total training time: ", __func__);
    print_duration((double) (t1 - t0));
    printf("\n");

    // 计算新的迭代次数
    int new_iters = opt->iter - opt_cb_data.last_save_iter;
    # 如果新的迭代次数大于0
    if (new_iters > 0) {
        # 更新训练迭代次数
        train->train_its     += new_iters;
        # 更新训练标记数
        train->train_tokens  += new_iters * opt->params.n_gradient_accumulation * n_batch * n_tokens;

        # 保存训练文件
        save_train_files(&save_data, train);
        # 更新最后保存的迭代次数
        opt_cb_data.last_save_iter = opt->iter;
    }

    # 如果分配了内存
    if (alloc) {
        # 释放分配的内存
        ggml_allocr_free(alloc);
    }

    # 释放优化器上下文
    ggml_free(opt->ctx);
    # 释放训练状态
    free_train_state(train);
    # 释放模型上下文
    ggml_free(model.ctx);
    # 释放 LLAMA 上下文
    llama_free(lctx);
    # 释放 LLAMA 模型
    llama_free_model(lmodel);
    # 返回0表示成功
    return 0;
# 闭合前面的函数定义
```