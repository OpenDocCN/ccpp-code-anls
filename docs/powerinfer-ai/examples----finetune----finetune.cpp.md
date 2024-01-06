# `PowerInfer\examples\finetune\finetune.cpp`

```
// 包含所需的头文件
#include "ggml.h"
#include "ggml-alloc.h"
#include "llama.h"
#include "common.h"
#include "train.h"
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

// 如果编译器为 MSC，禁用警告 4244 和 4267，避免可能的数据丢失警告
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif
// 定义张量的对齐方式为32
static const size_t tensor_alignment = 32;

// 定义模型超参数结构体
struct my_llama_hparams {
    // 词汇量大小
    uint32_t n_vocab    = 32000;
    // 上下文长度
    uint32_t n_ctx      = 512;
    // 嵌入维度
    uint32_t n_embd     = 4096;
    // 前馈网络隐藏层大小
    uint32_t n_ff       = 11008;
    // 注意力头数
    uint32_t n_head     = 32;
    // 键值注意力头数
    uint32_t n_head_kv  = 32;
    // 层数
    uint32_t n_layer    = 32;

    // RMS 归一化的 epsilon 值
    float f_norm_rms_eps = 1e-5f; // llama

    // ROPE 模块的基础频率
    float rope_freq_base  = 10000.0f;
    // ROPE 模块的频率缩放因子
    float rope_freq_scale = 1.0f;

    // 计算 GQA（Global Query-Attention）的函数
    uint32_t n_gqa() const {
        return n_head/n_head_kv;
    }

    // 返回嵌入维度除以头数的结果
    uint32_t n_embd_head() const {
        return n_embd/n_head;
    }

    // 返回嵌入维度除以问题数的结果
    uint32_t n_embd_gqa() const {
        return n_embd/n_gqa();
    }

    // 比较两个 my_llama_hparams 对象是否不相等
    bool operator!=(const my_llama_hparams& other) const {
        return memcmp(this, &other, sizeof(other));
    }
};

// my_llama_layer 结构体
struct my_llama_layer {
    // 规范化
    struct ggml_tensor * attention_norm;

    // 注意力
// 定义指向 ggml_tensor 结构体的指针变量 wq, wk, wv, wo，用于存储不同的张量数据
struct ggml_tensor * wq;
struct ggml_tensor * wk;
struct ggml_tensor * wv;
struct ggml_tensor * wo;

// 定义指向 ggml_tensor 结构体的指针变量 ffn_norm，用于存储归一化后的张量数据
struct ggml_tensor * ffn_norm;

// 定义指向 ggml_tensor 结构体的指针变量 w1, w2, w3，用于存储不同的张量数据
struct ggml_tensor * w1;
struct ggml_tensor * w2;
struct ggml_tensor * w3;

// 定义 my_llama_model 结构体，包含模型的超参数和不同的张量数据
struct my_llama_model {
    // 模型的超参数
    struct my_llama_hparams hparams;

    // 词嵌入张量
    struct ggml_tensor * tok_embeddings;

    // 归一化张量
    struct ggml_tensor * norm;
};
// 定义一个指向 ggml_tensor 结构体的指针变量 output

// 定义一个存储 my_llama_layer 结构体的向量 layers

// 定义一个结构体 my_llama_lora_hparams，包含多个 uint32_t 类型的成员变量

// 初始化 lora_r 为 1
// 初始化 lora_alpha 为 1
// 初始化 n_rank_attention_norm 为 1
// 初始化 n_rank_wq 为 4
// 初始化 n_rank_wk 为 4
// 初始化 n_rank_wv 为 4
// 初始化 n_rank_wo 为 4
// 初始化 n_rank_ffn_norm 为 1
// 初始化 n_rank_w1 为 4
// 初始化 n_rank_w2 为 4
// 初始化 n_rank_w3 为 4
// 初始化 n_rank_tok_embeddings 为 4
// 初始化 n_rank_norm 为 1
// 初始化 n_rank_output 为 4
// 定义一个不等于操作符重载函数，用于比较两个 my_llama_lora_hparams 对象是否不相等
bool operator!=(const my_llama_lora_hparams& other) const {
    // 使用 memcmp 函数比较当前对象和另一个对象的内存内容是否相等，返回比较结果
    return memcmp(this, &other, sizeof(other));
}

// 定义一个 my_llama_lora_layer 结构体，用于存储神经网络层的相关参数

// normalization
// 用于存储归一化操作的参数
struct ggml_tensor * attention_norm_a;
struct ggml_tensor * attention_norm_b;

// attention
// 用于存储注意力机制相关的参数
struct ggml_tensor * wq_a;
struct ggml_tensor * wq_b;
struct ggml_tensor * wk_a;
struct ggml_tensor * wk_b;
struct ggml_tensor * wv_a;
struct ggml_tensor * wv_b;
struct ggml_tensor * wo_a;
struct ggml_tensor * wo_b;
// 定义两个指向 ggml_tensor 结构体的指针，用于存储归一化后的数据
struct ggml_tensor * ffn_norm_a;
struct ggml_tensor * ffn_norm_b;

// 定义多个指向 ggml_tensor 结构体的指针，用于存储前馈神经网络的权重数据
struct ggml_tensor * w1_a;
struct ggml_tensor * w1_b;
struct ggml_tensor * w2_a;
struct ggml_tensor * w2_b;
struct ggml_tensor * w3_a;
struct ggml_tensor * w3_b;
};

// 定义一个结构体 my_llama_lora，包含指向 ggml_context 结构体的指针和一个存储无符号 8 位整数的向量
struct my_llama_lora {
    struct ggml_context * ctx = NULL;
    std::vector<uint8_t> data;

    // 定义一个结构体 my_llama_lora_hparams，用于存储超参数
    my_llama_lora_hparams hparams;
// 定义指向 ggml_tensor 结构体的指针变量
struct ggml_tensor * tok_embeddings_a;
struct ggml_tensor * tok_embeddings_b;

struct ggml_tensor * norm_a;
struct ggml_tensor * norm_b;
struct ggml_tensor * output_a;
struct ggml_tensor * output_b;

// 定义存储 my_llama_lora_layer 结构体的向量
std::vector<my_llama_lora_layer> layers;
};

// 定义常量字符串指针变量
static const char * LLM_KV_TRAINING_TYPE_FINETUNE_LORA   = "finetune_lora";
static const char * LLM_KV_TRAINING_TYPE                 = "training.type";

static const char * LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD  = "training.lora.rank.token_embd";
static const char * LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM = "training.lora.rank.output_norm";
static const char * LLM_KV_TRAINING_LORA_RANK_OUTPUT      = "training.lora.rank.output";
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_NORM   = "training.lora.rank.attn_norm";
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_Q      = "training.lora.rank.attn_q";
// 定义了一系列的常量，用于表示不同的键值对

// 以下是用于表示训练模型中的不同参数的常量
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_K      = "training.lora.rank.attn_k";
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_V      = "training.lora.rank.attn_v";
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_OUT    = "training.lora.rank.attn_output";
static const char * LLM_KV_TRAINING_LORA_RANK_FFN_NORM    = "training.lora.rank.ffn_norm";
static const char * LLM_KV_TRAINING_LORA_RANK_FFN_GATE    = "training.lora.rank.ffn_gate";
static const char * LLM_KV_TRAINING_LORA_RANK_FFN_DOWN    = "training.lora.rank.ffn_down";
static const char * LLM_KV_TRAINING_LORA_RANK_FFN_UP      = "training.lora.rank.ffn_up";

// 以下是用于表示通用架构和文件类型的常量
static const char * LLM_KV_GENERAL_ARCHITECTURE        = "general.architecture";
static const char * LLM_KV_GENERAL_FILE_TYPE           = "general.file_type";

// 以下是用于表示不同参数长度和数量的常量
static const char * LLM_KV_CONTEXT_LENGTH              = "%s.context_length";
static const char * LLM_KV_EMBEDDING_LENGTH            = "%s.embedding_length";
static const char * LLM_KV_BLOCK_COUNT                 = "%s.block_count";
static const char * LLM_KV_FEED_FORWARD_LENGTH         = "%s.feed_forward_length";
static const char * LLM_KV_ATTENTION_HEAD_COUNT        = "%s.attention.head_count";
static const char * LLM_KV_ATTENTION_HEAD_COUNT_KV     = "%s.attention.head_count_kv";
static const char * LLM_KV_ATTENTION_LAYERNORM_RMS_EPS = "%s.attention.layer_norm_rms_epsilon";
// 定义常量字符串，用于表示不同的维度计数、频率基数、线性比例等
static const char * LLM_KV_ROPE_DIMENSION_COUNT        = "%s.rope.dimension_count";
static const char * LLM_KV_ROPE_FREQ_BASE              = "%s.rope.freq_base"; // TODO load in llama.cpp
static const char * LLM_KV_ROPE_SCALE_LINEAR           = "%s.rope.scale_linear";

// 定义常量字符串，用于表示不同的张量名称
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

// 定义函数，用于打印参数信息
static void print_params(struct my_llama_hparams * params) {
    // 打印函数名称和词汇量大小
    printf("%s: n_vocab:   %u\n", __func__, params->n_vocab);
    // 打印函数名称和上下文大小
    printf("%s: n_ctx:     %u\n", __func__, params->n_ctx);
# 打印参数的 n_embd 值
printf("%s: n_embd:    %u\n", __func__, params->n_embd);
# 打印参数的 n_ff 值
printf("%s: n_ff:      %u\n", __func__, params->n_ff);
# 打印参数的 n_head 值
printf("%s: n_head:    %u\n", __func__, params->n_head);
# 打印参数的 n_head_kv 值
printf("%s: n_head_kv: %u\n", __func__, params->n_head_kv);
# 打印参数的 n_layer 值
printf("%s: n_layer:   %u\n", __func__, params->n_layer);
# 打印参数的 norm_rms_eps 值
printf("%s: norm_rms_eps          : %f\n", __func__, params->f_norm_rms_eps);
# 打印参数的 rope_freq_base 值
printf("%s: rope_freq_base        : %f\n", __func__, params->rope_freq_base);
# 打印参数的 rope_freq_scale 值
printf("%s: rope_freq_scale       : %f\n", __func__, params->rope_freq_scale);
# 打印参数的 n_rank_attention_norm 值
printf("%s: n_rank_attention_norm : %u\n", __func__, params->n_rank_attention_norm);
# 打印参数的 n_rank_wq 值
printf("%s: n_rank_wq             : %u\n", __func__, params->n_rank_wq);
# 打印参数的 n_rank_wk 值
printf("%s: n_rank_wk             : %u\n", __func__, params->n_rank_wk);
# 打印参数的 n_rank_wv 值
printf("%s: n_rank_wv             : %u\n", __func__, params->n_rank_wv);
# 打印参数的 n_rank_wo 值
printf("%s: n_rank_wo             : %u\n", __func__, params->n_rank_wo);
# 打印参数的 n_rank_ffn_norm 值
printf("%s: n_rank_ffn_norm       : %u\n", __func__, params->n_rank_ffn_norm);
# 打印参数的 n_rank_w1 值
printf("%s: n_rank_w1             : %u\n", __func__, params->n_rank_w1);
# 打印参数的 n_rank_w2 值
printf("%s: n_rank_w2             : %u\n", __func__, params->n_rank_w2);
# 打印参数的 n_rank_w3 值
printf("%s: n_rank_w3             : %u\n", __func__, params->n_rank_w3);
# 打印参数中的 n_rank_tok_embeddings 的值和函数名
printf("%s: n_rank_tok_embeddings : %u\n", __func__, params->n_rank_tok_embeddings);
# 打印参数中的 n_rank_norm 的值和函数名
printf("%s: n_rank_norm           : %u\n", __func__, params->n_rank_norm);
# 打印参数中的 n_rank_output 的值和函数名
printf("%s: n_rank_output         : %u\n", __func__, params->n_rank_output);
}

# 定义宏 GGUF_GET_KEY，用于获取指定键的值
#define GGUF_GET_KEY(ctx, dst, func, type, req, key) \
{ \
    # 将键转换为字符串
    const std::string skey(key); \
    # 在上下文中查找键的索引
    const int kid = gguf_find_key(ctx, skey.c_str()); \
    # 如果找到了键
    if (kid >= 0) { \
        # 获取键值的类型
        enum gguf_type ktype = gguf_get_kv_type(ctx, kid); \
        # 如果键值类型不符合预期类型
        if (ktype != (type)) { \
            # 抛出错误，指出键的类型错误
            die_fmt("key %s has wrong type: %s", skey.c_str(), gguf_type_name(ktype)); \
        } \
        # 使用指定函数获取键的值
        (dst) = func(ctx, kid); \
    } else if (req) { \
        # 如果需要找到键但未找到，抛出错误
        die_fmt("key not found in model: %s", skey.c_str()); \
    } \
}
// 加载模型超参数，将其存储在给定的 my_llama_hparams 结构体中，同时检查预期的架构
static void load_model_hparams_gguf(struct gguf_context * ctx, struct my_llama_hparams * hparams, const char * expected_arch) {
    // 声明一个字符串变量 arch
    std::string arch;

    // 从上下文中获取指定键的值，并存储在 arch 中
    GGUF_GET_KEY(ctx, arch, gguf_get_val_str, GGUF_TYPE_STRING, true, LLM_KV_GENERAL_ARCHITECTURE);
    // 如果预期架构不为空
    if (expected_arch != NULL) {
        // 如果 arch 不等于预期架构
        if (arch != expected_arch) {
            // 打印函数名、arch 和预期架构
            printf("%s: arch=%s expected_arch=%s\n", __func__, arch.c_str(), expected_arch);
        }
        // 断言 arch 等于预期架构
        GGML_ASSERT(arch == expected_arch);
    }

    // 声明一个字符向量 keybuf，大小为512
    std::vector<char> keybuf;
    keybuf.resize(512);
    // 定义一个 lambda 表达式 kv，用于构建键值
    auto kv = [&arch, &keybuf](const char * key) -> const char * {
        // 格式化键值，存储在 keybuf 中
        snprintf(keybuf.data(), keybuf.size(), key, arch.c_str());
        return keybuf.data();
    };

    // 从上下文中获取指定键的值，并存储在 hparams->n_embd 中
    GGUF_GET_KEY(ctx, hparams->n_embd, gguf_get_val_u32, GGUF_TYPE_UINT32, true, kv(LLM_KV_EMBEDDING_LENGTH));
    // 从上下文中获取指定键的值，并存储在 hparams->n_ctx 中
    GGUF_GET_KEY(ctx, hparams->n_ctx, gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_CONTEXT_LENGTH));
}
    // 从上下文中获取参数值，并将其赋给hparams->n_ff
    GGUF_GET_KEY(ctx, hparams->n_ff,           gguf_get_val_u32, GGUF_TYPE_UINT32,  true, kv(LLM_KV_FEED_FORWARD_LENGTH));
    // 从上下文中获取参数值，并将其赋给hparams->n_head
    GGUF_GET_KEY(ctx, hparams->n_head,         gguf_get_val_u32, GGUF_TYPE_UINT32,  true, kv(LLM_KV_ATTENTION_HEAD_COUNT));
    // 从上下文中获取参数值，并将其赋给hparams->n_layer
    GGUF_GET_KEY(ctx, hparams->n_layer,        gguf_get_val_u32, GGUF_TYPE_UINT32,  true, kv(LLM_KV_BLOCK_COUNT));

    // n_head_kv是可选的，默认为n_head
    hparams->n_head_kv = hparams->n_head;
    // 从上下文中获取参数值，并将其赋给hparams->n_head_kv
    GGUF_GET_KEY(ctx, hparams->n_head_kv,      gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ATTENTION_HEAD_COUNT_KV));

    // 初始化rope_freq_scale为1.0
    float rope_freq_scale = 1.0f;
    // 从上下文中获取参数值，并将其赋给hparams->f_norm_rms_eps
    GGUF_GET_KEY(ctx, hparams->f_norm_rms_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS));
    // 从上下文中获取参数值，并将其赋给hparams->rope_freq_base
    GGUF_GET_KEY(ctx, hparams->rope_freq_base, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_FREQ_BASE));
    // 从上下文中获取参数值，并将其赋给rope_freq_scale
    GGUF_GET_KEY(ctx, rope_freq_scale, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_SCALE_LINEAR));
    // 如果rope_freq_scale不等于1.0，则更新hparams->rope_freq_scale
    if (rope_freq_scale != 1.0f) {
        hparams->rope_freq_scale = 1.0f / rope_freq_scale;
    }
}

// 初始化模型
static void init_model(struct llama_model * input, struct my_llama_model * model, const char * fn_model, uint32_t n_ctx) {
    // 获取模型的超参数
    auto & hparams = model->hparams;
    // 创建一个存储字符的向量，用于存储文件名
    std::vector<char> tn_buf;
    // 调整向量大小为 GGML_MAX_NAME
    tn_buf.resize(GGML_MAX_NAME);
    // 定义一个 lambda 函数 tn，用于生成文件名
    auto tn = [&tn_buf](const char * key) -> const char * {
        // 使用 snprintf 将 key 和 ".weight" 格式化到 tn_buf 中
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", key);
        // 返回格式化后的文件名
        return tn_buf.data();
    };
    // 定义一个 lambda 函数 tni，用于生成带有参数的文件名
    auto tni = [&tn_buf](const char * key, int bid) -> const char * {
        // 使用 snprintf 将 key 和 bid 格式化到 tn_buf 中
        snprintf(tn_buf.data(), tn_buf.size(), key, bid);
        // 将 tn_buf 转换为字符串
        std::string s = tn_buf.data();
        // 使用 snprintf 将 ".weight" 格式化到 tn_buf 中
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", s.c_str());
        // 返回格式化后的文件名
        return tn_buf.data();
    };

    // 从 gguf 文件中直接获取参数
    {
        // 定义一个结构体 gguf_init_params，并初始化其中的字段
        struct gguf_init_params params = {
            /*.no_alloc = */ false,  // 设置 no_alloc 字段为 false
            /*.ctx      = */ NULL,   // 设置 ctx 字段为 NULL
        };
    // 从文件中初始化 gguf 上下文
    struct gguf_context * mctx = gguf_init_from_file(fn_model, params);

    // 从 gguf 上下文中加载模型超参数
    load_model_hparams_gguf(mctx, &hparams, "llama");

    // 释放 gguf 上下文
    gguf_free(mctx);
    }

    // 设置模型的词汇量和上下文大小
    hparams.n_vocab = llama_n_vocab(input);
    hparams.n_ctx = n_ctx;

    // 从 llama_model 中获取张量（可能是内存映射）
    model->tok_embeddings = llama_get_model_tensor(input, tn(LLM_TENSOR_TOKEN_EMBD));
    model->norm           = llama_get_model_tensor(input, tn(LLM_TENSOR_OUTPUT_NORM));
    model->output         = llama_get_model_tensor(input, tn(LLM_TENSOR_OUTPUT));

    // 确保张量的形状符合预期
    assert_shape_2d(model->tok_embeddings, hparams.n_embd, hparams.n_vocab);
    assert_shape_1d(model->norm,           hparams.n_embd);
    assert_shape_2d(model->output,         hparams.n_embd, hparams.n_vocab);

    // 调整模型的层数
    model->layers.resize(hparams.n_layer);
    for (uint32_t i = 0; i < hparams.n_layer; ++i) {
// 获取模型的第i层
auto & layer = model->layers[i];

// 获取输入中的注意力规范化张量
layer.attention_norm = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_NORM, i));
// 获取输入中的注意力查询张量
layer.wq = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_Q, i));
// 获取输入中的注意力键张量
layer.wk = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_K, i));
// 获取输入中的注意力值张量
layer.wv = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_V, i));
// 获取输入中的注意力输出张量
layer.wo = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_OUT, i));
// 获取输入中的前馈网络规范化张量
layer.ffn_norm = llama_get_model_tensor(input, tni(LLM_TENSOR_FFN_NORM, i));
// 获取输入中的前馈网络第一层权重张量
layer.w1 = llama_get_model_tensor(input, tni(LLM_TENSOR_FFN_GATE, i));
// 获取输入中的前馈网络第二层权重张量
layer.w2 = llama_get_model_tensor(input, tni(LLM_TENSOR_FFN_DOWN, i));
// 获取输入中的前馈网络第三层权重张量
layer.w3 = llama_get_model_tensor(input, tni(LLM_TENSOR_FFN_UP, i));

// 断言张量的形状为一维数组，长度为hparams.n_embd
assert_shape_1d(layer.attention_norm, hparams.n_embd);
// 断言张量的形状为二维数组，大小为hparams.n_embd x hparams.n_embd
assert_shape_2d(layer.wq, hparams.n_embd, hparams.n_embd);
// 断言张量的形状为二维数组，大小为hparams.n_embd x hparams.n_embd_gqa()
assert_shape_2d(layer.wk, hparams.n_embd, hparams.n_embd_gqa());
// 断言张量的形状为二维数组，大小为hparams.n_embd x hparams.n_embd_gqa()
assert_shape_2d(layer.wv, hparams.n_embd, hparams.n_embd_gqa());
// 断言张量的形状为二维数组，大小为hparams.n_embd x hparams.n_embd
assert_shape_2d(layer.wo, hparams.n_embd, hparams.n_embd);
// 断言张量的形状为一维数组，长度为hparams.n_embd
assert_shape_1d(layer.ffn_norm, hparams.n_embd);
// 断言张量的形状为二维数组，大小为hparams.n_embd x hparams.n_ff
assert_shape_2d(layer.w1, hparams.n_embd, hparams.n_ff);
// 断言张量的形状为二维数组，大小为hparams.n_ff x hparams.n_embd
assert_shape_2d(layer.w2, hparams.n_ff, hparams.n_embd);
// 断言 layer.w3 的形状为二维，其形状为 hparams.n_embd * hparams.n_ff
assert_shape_2d(layer.w3, hparams.n_embd, hparams.n_ff);
}

// 设置参数为 Lora 模型
static void set_param_lora(struct my_llama_lora * lora) {
    // 获取层的数量
    const uint32_t n_layer = lora->layers.size();

    // 获取 Lora 模型的上下文
    struct ggml_context* ctx = lora->ctx;

    // 设置 Lora 模型的参数
    ggml_set_param(ctx, lora->tok_embeddings_a);
    ggml_set_param(ctx, lora->tok_embeddings_b);
    ggml_set_param(ctx, lora->norm_a);
    ggml_set_param(ctx, lora->norm_b);
    ggml_set_param(ctx, lora->output_a);
    ggml_set_param(ctx, lora->output_b);

    // 遍历每一层，设置其参数
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = lora->layers[i];

        ggml_set_param(ctx, layer.attention_norm_a);
```

# 设置参数到上下文中，参数来自于层对象的属性
ggml_set_param(ctx, layer.attention_norm_b);
ggml_set_param(ctx, layer.wq_a);
ggml_set_param(ctx, layer.wq_b);
ggml_set_param(ctx, layer.wk_a);
ggml_set_param(ctx, layer.wk_b);
ggml_set_param(ctx, layer.wv_a);
ggml_set_param(ctx, layer.wv_b);
ggml_set_param(ctx, layer.wo_a);
ggml_set_param(ctx, layer.wo_b);
ggml_set_param(ctx, layer.ffn_norm_a);
ggml_set_param(ctx, layer.ffn_norm_b);
ggml_set_param(ctx, layer.w1_a);
ggml_set_param(ctx, layer.w1_b);
ggml_set_param(ctx, layer.w2_a);
ggml_set_param(ctx, layer.w2_b);
ggml_set_param(ctx, layer.w3_a);
ggml_set_param(ctx, layer.w3_b);
# 结束函数定义
// 分配内存给 Lora 模型的各个部分
static void alloc_lora(struct ggml_allocr * alloc, struct my_llama_lora * lora) {
    // 分配内存给 tok_embeddings_a
    ggml_allocr_alloc(alloc, lora->tok_embeddings_a);
    // 分配内存给 tok_embeddings_b
    ggml_allocr_alloc(alloc, lora->tok_embeddings_b);
    // 分配内存给 norm_a
    ggml_allocr_alloc(alloc, lora->norm_a);
    // 分配内存给 norm_b
    ggml_allocr_alloc(alloc, lora->norm_b);
    // 分配内存给 output_a
    ggml_allocr_alloc(alloc, lora->output_a);
    // 分配内存给 output_b
    ggml_allocr_alloc(alloc, lora->output_b);
    // 遍历 Lora 模型的每一层
    for (uint32_t i = 0; i < lora->layers.size(); ++i) {
        // 获取当前层的引用
        auto & layer = lora->layers[i];
        // 分配内存给 attention_norm_a
        ggml_allocr_alloc(alloc, layer.attention_norm_a);
        // 分配内存给 attention_norm_b
        ggml_allocr_alloc(alloc, layer.attention_norm_b);
        // 分配内存给 wq_a
        ggml_allocr_alloc(alloc, layer.wq_a);
        // 分配内存给 wq_b
        ggml_allocr_alloc(alloc, layer.wq_b);
        // 分配内存给 wk_a
        ggml_allocr_alloc(alloc, layer.wk_a);
        // 分配内存给 wk_b
        ggml_allocr_alloc(alloc, layer.wk_b);
        // 分配内存给 wv_a
        ggml_allocr_alloc(alloc, layer.wv_a);
        // 分配内存给 wv_b
        ggml_allocr_alloc(alloc, layer.wv_b);
        // 分配内存给 wo_a
        ggml_allocr_alloc(alloc, layer.wo_a);
        // 分配内存给 wo_b
        ggml_allocr_alloc(alloc, layer.wo_b);
        // 分配内存给 ffn_norm_a
        ggml_allocr_alloc(alloc, layer.ffn_norm_a);
    }
}
# 为 layer.ffn_norm_b 分配内存
ggml_allocr_alloc(alloc, layer.ffn_norm_b);
# 为 layer.w1_a 分配内存
ggml_allocr_alloc(alloc, layer.w1_a);
# 为 layer.w1_b 分配内存
ggml_allocr_alloc(alloc, layer.w1_b);
# 为 layer.w2_a 分配内存
ggml_allocr_alloc(alloc, layer.w2_a);
# 为 layer.w2_b 分配内存
ggml_allocr_alloc(alloc, layer.w2_b);
# 为 layer.w3_a 分配内存
ggml_allocr_alloc(alloc, layer.w3_a);
# 为 layer.w3_b 分配内存
ggml_allocr_alloc(alloc, layer.w3_b);
# 为 lora->tok_embeddings_a->grad 分配内存
ggml_allocr_alloc(alloc, lora->tok_embeddings_a->grad);
# 为 lora->tok_embeddings_b->grad 分配内存
ggml_allocr_alloc(alloc, lora->tok_embeddings_b->grad);
# 为 lora->norm_a->grad 分配内存
ggml_allocr_alloc(alloc, lora->norm_a->grad);
# 为 lora->norm_b->grad 分配内存
ggml_allocr_alloc(alloc, lora->norm_b->grad);
# 为 lora->output_a->grad 分配内存
ggml_allocr_alloc(alloc, lora->output_a->grad);
# 为 lora->output_b->grad 分配内存
ggml_allocr_alloc(alloc, lora->output_b->grad);
# 遍历 lora 的每个层，为每个层的相关属性分配内存
for (uint32_t i = 0; i < lora->layers.size(); ++i) {
    auto & layer = lora->layers[i];
    ggml_allocr_alloc(alloc, layer.attention_norm_a->grad);
    ggml_allocr_alloc(alloc, layer.attention_norm_b->grad);
    ggml_allocr_alloc(alloc, layer.wq_a->grad);
    ggml_allocr_alloc(alloc, layer.wq_b->grad);
}
# 调用 ggml_allocr_alloc 函数为 layer.wk_a->grad 分配内存
ggml_allocr_alloc(alloc, layer.wk_a->grad);
# 调用 ggml_allocr_alloc 函数为 layer.wk_b->grad 分配内存
ggml_allocr_alloc(alloc, layer.wk_b->grad);
# 调用 ggml_allocr_alloc 函数为 layer.wv_a->grad 分配内存
ggml_allocr_alloc(alloc, layer.wv_a->grad);
# 调用 ggml_allocr_alloc 函数为 layer.wv_b->grad 分配内存
ggml_allocr_alloc(alloc, layer.wv_b->grad);
# 调用 ggml_allocr_alloc 函数为 layer.wo_a->grad 分配内存
ggml_allocr_alloc(alloc, layer.wo_a->grad);
# 调用 ggml_allocr_alloc 函数为 layer.wo_b->grad 分配内存
ggml_allocr_alloc(alloc, layer.wo_b->grad);
# 调用 ggml_allocr_alloc 函数为 layer.ffn_norm_a->grad 分配内存
ggml_allocr_alloc(alloc, layer.ffn_norm_a->grad);
# 调用 ggml_allocr_alloc 函数为 layer.ffn_norm_b->grad 分配内存
ggml_allocr_alloc(alloc, layer.ffn_norm_b->grad);
# 调用 ggml_allocr_alloc 函数为 layer.w1_a->grad 分配内存
ggml_allocr_alloc(alloc, layer.w1_a->grad);
# 调用 ggml_allocr_alloc 函数为 layer.w1_b->grad 分配内存
ggml_allocr_alloc(alloc, layer.w1_b->grad);
# 调用 ggml_allocr_alloc 函数为 layer.w2_a->grad 分配内存
ggml_allocr_alloc(alloc, layer.w2_a->grad);
# 调用 ggml_allocr_alloc 函数为 layer.w2_b->grad 分配内存
ggml_allocr_alloc(alloc, layer.w2_b->grad);
# 调用 ggml_allocr_alloc 函数为 layer.w3_a->grad 分配内存
ggml_allocr_alloc(alloc, layer.w3_a->grad);
# 调用 ggml_allocr_alloc 函数为 layer.w3_b->grad 分配内存
ggml_allocr_alloc(alloc, layer.w3_b->grad);
# 结束 if 语句块
}
# 初始化 lora 结构体的参数为 lparams
static void init_lora(const struct my_llama_model * model, struct my_llama_lora * lora) {
    # 获取 lora 结构体的超参数
    const auto & lparams = lora->hparams;
// 定义并初始化模型的各种参数
const uint32_t n_embd     = model->hparams.n_embd; // 获取嵌入层的维度
const uint32_t n_embd_gqa = model->hparams.n_embd_gqa(); // 获取GQA嵌入层的维度
const uint32_t n_layer    = model->hparams.n_layer; // 获取模型的层数
const uint32_t n_vocab    = model->hparams.n_vocab; // 获取词汇表的大小
const uint32_t n_ff       = model->hparams.n_ff; // 获取前馈神经网络的维度

// 初始化存储名称的缓冲区
std::vector<char> tn_buf;
tn_buf.resize(GGML_MAX_NAME);

// 定义lambda函数tn，用于拼接key和suffix生成新的名称
auto tn = [&tn_buf](const char * key, const char * suffix) -> const char * {
    snprintf(tn_buf.data(), tn_buf.size(), "%s%s", key, suffix);
    return tn_buf.data();
};

// 定义lambda函数tni，用于拼接key和suffix生成新的名称，并加上bid作为后缀
auto tni = [&tn_buf](const char * key, const char * suffix, int bid) -> const char * {
    snprintf(tn_buf.data(), tn_buf.size(), key, bid);
    std::string s = tn_buf.data();
    snprintf(tn_buf.data(), tn_buf.size(), "%s%s", s.c_str(), suffix);
    return tn_buf.data();
};

// 为lora张量提供没有数据的上下文
# 定义结构体变量 ctx_lora_params，用于初始化参数
struct ggml_init_params ctx_lora_params;
# 设置内存大小为 ggml_tensor_overhead()*2*(6 + n_layer*18)
ctx_lora_params.mem_size   = ggml_tensor_overhead()*2*(6 + n_layer*18);
# 设置内存缓冲区为空
ctx_lora_params.mem_buffer = NULL;
# 设置不进行内存分配
ctx_lora_params.no_alloc   = true;

# 初始化 ggml_context 结构体指针 ctx，使用 ctx_lora_params 参数
struct ggml_context * ctx = ggml_init(ctx_lora_params);
# 将初始化的 ctx 赋值给 lora 结构体的 ctx 成员
lora->ctx = ctx;

# 创建并初始化 lora 结构体的 tok_embeddings_a 成员
lora->tok_embeddings_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_tok_embeddings, n_embd);
# 创建并初始化 lora 结构体的 tok_embeddings_b 成员
lora->tok_embeddings_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_tok_embeddings, n_vocab);
# 创建并初始化 lora 结构体的 norm_a 成员
lora->norm_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_norm, n_embd);
# 创建并初始化 lora 结构体的 norm_b 成员
lora->norm_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_norm, 1);
# 创建并初始化 lora 结构体的 output_a 成员
lora->output_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_output, n_embd);
# 创建并初始化 lora 结构体的 output_b 成员
lora->output_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_output, n_vocab);

# 设置 lora 结构体的 tok_embeddings_a 成员的名称
ggml_set_name(lora->tok_embeddings_a, tn(LLM_TENSOR_TOKEN_EMBD,  ".weight.lora_a"));
# 设置 lora 结构体的 tok_embeddings_b 成员的名称
ggml_set_name(lora->tok_embeddings_b, tn(LLM_TENSOR_TOKEN_EMBD,  ".weight.lora_b"));
# 设置 lora 结构体的 norm_a 成员的名称
ggml_set_name(lora->norm_a, tn(LLM_TENSOR_OUTPUT_NORM, ".weight.lora_a"));
# 设置 lora 结构体的 norm_b 成员的名称
ggml_set_name(lora->norm_b, tn(LLM_TENSOR_OUTPUT_NORM, ".weight.lora_b"));
# 设置 lora 结构体的 output_a 成员的名称
ggml_set_name(lora->output_a, tn(LLM_TENSOR_OUTPUT,      ".weight.lora_a"));
# 设置 lora->output_b 的名称为 ".weight.lora_b"
ggml_set_name(lora->output_b, tn(LLM_TENSOR_OUTPUT, ".weight.lora_b"));

# 调整 lora->layers 的大小为 n_layer
lora->layers.resize(n_layer)

# 遍历每一层，为每一层的不同属性创建新的张量
for (uint32_t i = 0; i < n_layer; ++i) {
    # 获取当前层的引用
    auto & layer = lora->layers[i];

    # 创建 attention_norm_a 和 attention_norm_b 张量
    layer.attention_norm_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_attention_norm, n_embd);
    layer.attention_norm_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_attention_norm, 1);

    # 创建 wq_a, wq_b, wk_a, wk_b, wv_a, wv_b, wo_a, wo_b 张量
    layer.wq_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_wq, n_embd);
    layer.wq_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_wq, n_embd);
    layer.wk_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_wk, n_embd);
    layer.wk_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_wk, n_embd_gqa);
    layer.wv_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_wv, n_embd);
    layer.wv_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_wv, n_embd_gqa);
    layer.wo_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_wo, n_embd);
    layer.wo_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_wo, n_embd);

    # 创建 ffn_norm_a 和 ffn_norm_b 张量
    layer.ffn_norm_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_ffn_norm, n_embd);
    layer.ffn_norm_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_ffn_norm, 1);
}
# 创建一个二维的浮点型张量，用于存储权重数据
layer.w1_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_w1, n_embd);
# 创建一个二维的浮点型张量，用于存储权重数据
layer.w1_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_w1, n_ff);
# 创建一个二维的浮点型张量，用于存储权重数据
layer.w2_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_w2, n_ff);
# 创建一个二维的浮点型张量，用于存储权重数据
layer.w2_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_w2, n_embd);
# 创建一个二维的浮点型张量，用于存储权重数据
layer.w3_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_w3, n_embd);
# 创建一个二维的浮点型张量，用于存储权重数据
layer.w3_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_w3, n_ff);

# 设置张量的名称
ggml_set_name(layer.attention_norm_a, tni(LLM_TENSOR_ATTN_NORM, ".weight.lora_a", i));
# 设置张量的名称
ggml_set_name(layer.attention_norm_b, tni(LLM_TENSOR_ATTN_NORM, ".weight.lora_b", i));
# 设置张量的名称
ggml_set_name(layer.wq_a,             tni(LLM_TENSOR_ATTN_Q,    ".weight.lora_a", i));
# 设置张量的名称
ggml_set_name(layer.wq_b,             tni(LLM_TENSOR_ATTN_Q,    ".weight.lora_b", i));
# 设置张量的名称
ggml_set_name(layer.wk_a,             tni(LLM_TENSOR_ATTN_K,    ".weight.lora_a", i));
# 设置张量的名称
ggml_set_name(layer.wk_b,             tni(LLM_TENSOR_ATTN_K,    ".weight.lora_b", i));
# 设置张量的名称
ggml_set_name(layer.wv_a,             tni(LLM_TENSOR_ATTN_V,    ".weight.lora_a", i));
# 设置张量的名称
ggml_set_name(layer.wv_b,             tni(LLM_TENSOR_ATTN_V,    ".weight.lora_b", i));
# 设置张量的名称
ggml_set_name(layer.wo_a,             tni(LLM_TENSOR_ATTN_OUT,  ".weight.lora_a", i));
# 设置张量的名称
ggml_set_name(layer.wo_b,             tni(LLM_TENSOR_ATTN_OUT,  ".weight.lora_b", i));
# 设置张量的名称
ggml_set_name(layer.ffn_norm_a,       tni(LLM_TENSOR_FFN_NORM,  ".weight.lora_a", i));
# 设置张量的名称
ggml_set_name(layer.ffn_norm_b,       tni(LLM_TENSOR_FFN_NORM,  ".weight.lora_b", i));
    // 设置神经网络层的权重名称
    ggml_set_name(layer.w1_a, tni(LLM_TENSOR_FFN_GATE, ".weight.lora_a", i));
    ggml_set_name(layer.w1_b, tni(LLM_TENSOR_FFN_GATE, ".weight.lora_b", i));
    ggml_set_name(layer.w2_a, tni(LLM_TENSOR_FFN_DOWN, ".weight.lora_a", i));
    ggml_set_name(layer.w2_b, tni(LLM_TENSOR_FFN_DOWN, ".weight.lora_b", i));
    ggml_set_name(layer.w3_a, tni(LLM_TENSOR_FFN_UP, ".weight.lora_a", i));
    ggml_set_name(layer.w3_b, tni(LLM_TENSOR_FFN_UP, ".weight.lora_b", i));

    // 设置 LORA 参数
    set_param_lora(lora);

    // 测量数据大小
    size_t size = 0;
    for (struct ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
        size += GGML_PAD(ggml_nbytes(t), tensor_alignment);
    }

    // 分配数据
    struct ggml_allocr * alloc = NULL;
    lora->data.resize(size + tensor_alignment);
    alloc = ggml_allocr_new(lora->data.data(), lora->data.size(), tensor_alignment);
# 分配 LORA 内存
alloc_lora(alloc, lora);
# 释放 LORA 内存
ggml_allocr_free(alloc);
}

# 随机初始化 LORA 结构体中的数据
static void randomize_lora(struct my_llama_lora * lora, int seed, float mean, float std, float min, float max) {
    # 获取 LORA 中层数
    const uint32_t n_layer = lora->layers.size();

    # 初始化随机正态分布
    struct random_normal_distribution * rnd = init_random_normal_distribution(seed, mean, std, min, max);

    # 随机初始化各个张量数据
    randomize_tensor_normal(lora->tok_embeddings_a, rnd);
    randomize_tensor_normal(lora->tok_embeddings_b, rnd);
    randomize_tensor_normal(lora->norm_a,           rnd);
    randomize_tensor_normal(lora->norm_b,           rnd);
    randomize_tensor_normal(lora->output_a,         rnd);
    randomize_tensor_normal(lora->output_b,         rnd);

    # 遍历每一层，随机初始化注意力层的张量数据
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = lora->layers[i];
        randomize_tensor_normal(layer.attention_norm_a, rnd);
        randomize_tensor_normal(layer.attention_norm_b, rnd);
# 以正态分布随机初始化权重张量
randomize_tensor_normal(layer.wq_a, rnd);
randomize_tensor_normal(layer.wq_b, rnd);
randomize_tensor_normal(layer.wk_a, rnd);
randomize_tensor_normal(layer.wk_b, rnd);
randomize_tensor_normal(layer.wv_a, rnd);
randomize_tensor_normal(layer.wv_b, rnd);
randomize_tensor_normal(layer.wo_a, rnd);
randomize_tensor_normal(layer.wo_b, rnd);

# 以正态分布随机初始化层归一化张量
randomize_tensor_normal(layer.ffn_norm_a, rnd);
randomize_tensor_normal(layer.ffn_norm_b, rnd);

# 以正态分布随机初始化前馈神经网络的权重张量
randomize_tensor_normal(layer.w1_a, rnd);
randomize_tensor_normal(layer.w1_b, rnd);
randomize_tensor_normal(layer.w2_a, rnd);
randomize_tensor_normal(layer.w2_b, rnd);
randomize_tensor_normal(layer.w3_a, rnd);
randomize_tensor_normal(layer.w3_b, rnd);
// 定义一个函数 llama_build_lora_finetune_graphs，用于构建 LORA 微调图
static struct ggml_tensor * llama_build_lora_finetune_graphs(
        // 输入参数包括模型、LORA、内存分配器、上下文、前向图、反向图、临时反向图、logits、tokens_input、targets、tokens数量、batch数量、是否启用闪存注意力、是否启用检查点
        struct my_llama_model * model,
        struct my_llama_lora  * lora,
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
    # 设置上下文环境的临时变量
    ggml_set_scratch(ctx, { 0, 0, nullptr, });
    # 定义并初始化常量 n_past
    const int n_past = 0;
    # 定义并初始化常量 N，其值为 n_tokens
    const int N = n_tokens;
    # 获取模型的超参数
    const auto & hparams  = model->hparams;
    # 获取上下文环境的大小
    const int n_ctx       = hparams.n_ctx;
    # 获取词汇表的大小
    const int n_vocab     = hparams.n_vocab;
    # 获取嵌入层的维度
    const int n_embd      = hparams.n_embd;
    # 获取层数
    const int n_layer     = hparams.n_layer;
    # 获取注意力头的数量
    const int n_head      = hparams.n_head;
    # 获取键值头的数量
    const int n_head_kv   = hparams.n_head_kv;
    # 获取前馈网络的维度
    const int n_ff        = hparams.n_ff;
    # 获取旋转的数量
    const int n_rot       = hparams.n_embd_head();
    # 获取每个注意力头的嵌入维度
    const int n_embd_head = hparams.n_embd_head();
    # 获取 GQA 嵌入的维度
    const int n_embd_gqa  = hparams.n_embd_gqa();
    # 获取 RMS 归一化的 epsilon 值
    const float rms_norm_eps    = hparams.f_norm_rms_eps;
    # 获取 ROPE 频率的基础值
    const float rope_freq_base  = hparams.rope_freq_base;
    # 获取 ROPE 频率的缩放值
    const float rope_freq_scale = hparams.rope_freq_scale;

    # 断言层数与模型的层数相等
    GGML_ASSERT((size_t) n_layer == lora->layers.size());
    // 定义一个lambda函数，用于设置张量的名称
    auto set_name = [](struct ggml_tensor * t, const char * n) {
        ggml_set_name(t, n); // 设置张量的名称
        if (t->grad) {
            ggml_format_name(t->grad, "%s->grad", n); // 如果存在梯度张量，设置梯度张量的名称
        }
    };

    // 创建一个1维整型张量KQ_pos，用于存储位置信息
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, N);
    ggml_allocr_alloc(alloc, KQ_pos); // 分配内存空间
    if (!ggml_allocr_is_measure(alloc)) { // 如果不是度量张量
        int * data = (int *) KQ_pos->data; // 获取数据指针
        for (int i = 0; i < N; ++i) {
            data[i] = n_past + i; // 设置位置信息的值
        }
    }

    // 定义一个lambda函数，用于处理rope张量的参数
    auto rope = [ctx, KQ_pos, n_rot, n_ctx, rope_freq_base, rope_freq_scale]
                (struct ggml_tensor * t) -> struct ggml_tensor * {
        // 定义一个常量 rope_mode，并赋值为 0
        const int rope_mode = 0;

        // 调用 ggml_rope_custom 函数，传入参数 t, KQ_pos, n_rot, rope_mode, n_ctx, 0, rope_freq_base, rope_freq_scale, 0.0f, 1.0f, 0.0f, 0.0f，并返回结果
        return ggml_rope_custom(ctx,
            t, KQ_pos, n_rot, rope_mode, n_ctx, 0,
            rope_freq_base, rope_freq_scale, 0.0f, 1.0f, 0.0f, 0.0f
        );
    };

    // 设置 tokens_input 的名称为 "tokens_input"
    set_name(tokens_input, "tokens_input");
    // 设置 targets 的名称为 "targets"
    set_name(targets,      "targets");

    // 断言 tokens_input 的类型为 GGML_TYPE_I32
    GGML_ASSERT(tokens_input->type == GGML_TYPE_I32);

    // 定义一个 lambda 函数 add_to_f32，接受参数 ctx, a, b
    auto add_to_f32 = [] (struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b) {
        // 如果 a 的类型是量化的或者是 GGML_TYPE_F16，则调用 ggml_add_cast 函数，将 a 转换为 GGML_TYPE_F32，并返回结果
        if (ggml_is_quantized(a->type) || a->type == GGML_TYPE_F16) {
            return ggml_add_cast(ctx, a, b, GGML_TYPE_F32);
        } 
        // 如果 a 的类型是 GGML_TYPE_F32，则调用 ggml_add 函数，对 a 和 b 进行加法操作，并返回结果
        else if (a->type == GGML_TYPE_F32) {
            return ggml_add(ctx, a, b);
        } 
        // 如果以上条件都不满足，则执行以下操作
        else {
# 输出错误信息，指出不支持对指定类型的张量进行微调
die_fmt("%s: Finetuning on tensors with type '%s' is not yet supported.\n",
    __func__, ggml_type_name(a->type));
};

# 创建 tok_embeddings 张量，并将 lora->tok_embeddings_a 和 lora->tok_embeddings_b 的乘积添加到上下文中
struct ggml_tensor * tok_embeddings = add_to_f32(ctx, model->tok_embeddings, ggml_mul_mat(ctx, lora->tok_embeddings_a, lora->tok_embeddings_b));

# 创建 norm 张量，并将 lora->norm_a 和 lora->norm_b 的乘积添加到上下文中
struct ggml_tensor * norm = add_to_f32(ctx, model->norm, ggml_mul_mat(ctx, lora->norm_a, lora->norm_b));

# 创建 output 张量，并将 lora->output_a 和 lora->output_b 的乘积添加到上下文中
struct ggml_tensor * output = add_to_f32(ctx, model->output, ggml_mul_mat(ctx, lora->output_a, lora->output_b));

# 创建 t00 张量，将 tokens_input 重塑为一维张量，并设置名称和形状
struct ggml_tensor * t00 = ggml_reshape_1d(ctx, tokens_input, N*n_batch);  set_name(t00, "t00"); assert_shape_1d(t00, N*n_batch);

# 创建 t01 张量，从 tok_embeddings 中获取 t00 中指定的行，并设置名称和形状
struct ggml_tensor * t01 = ggml_get_rows(ctx, tok_embeddings, t00);        set_name(t01, "t01"); assert_shape_2d(t01, n_embd, N*n_batch);

# 将 t01 赋值给 cur
struct ggml_tensor * cur = t01;

# 创建一个张量指针的向量 checkpoints，并在启用检查点时将 tokens_input、targets、t00 和 t01 添加到其中
std::vector<struct ggml_tensor *> checkpoints;
if (enable_checkpointing) {
    checkpoints.push_back(tokens_input);
    checkpoints.push_back(targets);
    checkpoints.push_back(t00);
    checkpoints.push_back(t01);
    }

    // 定义指向 ggml_tensor 结构体的指针 kv_scale，并初始化为 NULL
    struct ggml_tensor * kv_scale = NULL;
    // 如果不启用闪存注意力机制，则计算 kv_scale 的值
    if (!enable_flash_attn) {
        kv_scale = ggml_new_f32(ctx, 1.0f/sqrtf(float(n_embd)/n_head));
    }

    // 遍历每一层
    for (int il = 0; il < n_layer; ++il) {
        // 获取当前层的注意力层和 LORA 层
        struct my_llama_layer & layer = model->layers[il];
        struct my_llama_lora_layer & llayer = lora->layers[il];

        // 计算注意力层的归一化值
        struct ggml_tensor * attention_norm = add_to_f32(ctx, layer.attention_norm, ggml_mul_mat(ctx, llayer.attention_norm_a, llayer.attention_norm_b));
        // 计算前馈神经网络层的归一化值
        struct ggml_tensor * ffn_norm = add_to_f32(ctx, layer.ffn_norm, ggml_mul_mat(ctx, llayer.ffn_norm_a, llayer.ffn_norm_b));
        // 计算查询权重
        struct ggml_tensor * wq = add_to_f32(ctx, layer.wq, ggml_mul_mat(ctx, llayer.wq_a, llayer.wq_b));
        // 计算键权重
        struct ggml_tensor * wk = add_to_f32(ctx, layer.wk, ggml_mul_mat(ctx, llayer.wk_a, llayer.wk_b));
        // 计算值权重
        struct ggml_tensor * wv = add_to_f32(ctx, layer.wv, ggml_mul_mat(ctx, llayer.wv_a, llayer.wv_b));
        // 计算输出权重
        struct ggml_tensor * wo = add_to_f32(ctx, layer.wo, ggml_mul_mat(ctx, llayer.wo_a, llayer.wo_b));
        // 计算第一个前馈神经网络层的权重
        struct ggml_tensor * w1 = add_to_f32(ctx, layer.w1, ggml_mul_mat(ctx, llayer.w1_a, llayer.w1_b));
        // 计算第二个前馈神经网络层的权重
        struct ggml_tensor * w2 = add_to_f32(ctx, layer.w2, ggml_mul_mat(ctx, llayer.w2_a, llayer.w2_b));
        // 计算第三个前馈神经网络层的权重
        struct ggml_tensor * w3 = add_to_f32(ctx, layer.w3, ggml_mul_mat(ctx, llayer.w3_a, llayer.w3_b));
# 创建一个名为t02的张量，使用ggml_rms_norm函数对当前张量cur进行RMS标准化，设置名称为"t02"，并断言其形状为2维，大小为n_embd * N * n_batch
struct ggml_tensor * t02 = ggml_rms_norm(ctx, cur, rms_norm_eps); set_name(t02, "t02"); assert_shape_2d(t02, n_embd, N*n_batch);

# 创建一个名为t03的张量，使用ggml_repeat函数对attention_norm进行重复操作，设置名称为"t03"，并断言其形状为2维，大小为n_embd * N * n_batch
struct ggml_tensor * t03 = ggml_repeat(ctx, attention_norm, t02); set_name(t03, "t03"); assert_shape_2d(t03, n_embd, N*n_batch);

# 创建一个名为t04的张量，使用ggml_mul函数对t03和t02进行元素级乘法，设置名称为"t04"，并断言其形状为2维，大小为n_embd * N * n_batch
struct ggml_tensor * t04 = ggml_mul(ctx, t03, t02); set_name(t04, "t04"); assert_shape_2d(t04, n_embd, N*n_batch);

# 创建一个名为t05的张量，使用ggml_mul_mat函数对wq和t04进行矩阵乘法，设置名称为"t05"，并断言其形状为2维，大小为n_embd * N * n_batch
struct ggml_tensor * t05 = ggml_mul_mat(ctx, wq, t04); set_name(t05, "t05"); assert_shape_2d(t05, n_embd, N*n_batch);

# 创建一个名为t06的张量，使用ggml_reshape_4d函数对t05进行4维重塑，设置名称为"t06"，并断言其形状为4维
struct ggml_tensor * t06 = ggml_reshape_4d(ctx, t05, n_embd_head, n_head, N, n_batch); set_name(t06, "t06"); assert_shape_4d(t06, n_embd_head, n_head, N, n_batch);

# 创建一个名为t07的张量，使用rope函数对t06进行操作，设置名称为"t07"，并断言其形状为4维
struct ggml_tensor * t07 = rope(t06); set_name(t07, "t07"); assert_shape_4d(t07, n_embd_head, n_head, N, n_batch);

# 创建一个名为t08的张量，使用ggml_mul_mat函数对wk和t04进行矩阵乘法，设置名称为"t08"，并断言其形状为2维，大小为n_embd_gqa * N * n_batch
struct ggml_tensor * t08 = ggml_mul_mat(ctx, wk, t04); set_name(t08, "t08"); assert_shape_2d(t08, n_embd_gqa, N*n_batch);

# 创建一个名为t09的张量，使用ggml_reshape_4d函数对t08进行4维重塑，设置名称为"t09"，并断言其形状为4维
struct ggml_tensor * t09 = ggml_reshape_4d(ctx, t08, n_embd_head, n_head_kv, N, n_batch); set_name(t09, "t09"); assert_shape_4d(t09, n_embd_head, n_head_kv, N, n_batch);

# 创建一个名为t10的张量，使用rope函数对t09进行操作，设置名称为"t10"，并断言其形状为4维
struct ggml_tensor * t10 = rope(t09); set_name(t10, "t10"); assert_shape_4d(t10, n_embd_head, n_head_kv, N, n_batch);

# 创建一个名为t11的张量
struct ggml_tensor * t11;
# 如果wv的类型是量化的，则执行以下操作
if (ggml_is_quantized(wv->type)) {
    # 创建一个名为t11_1的张量，使用ggml_mul_mat函数对wv和t04进行矩阵乘法，设置名称为"t11_1"，并断言其形状为2维，大小为n_embd_gqa * N * n_batch
    struct ggml_tensor * t11_1 = ggml_mul_mat(ctx, wv, t04); set_name(t11_1, "t11_1"); assert_shape_2d(t11_1, n_embd_gqa, N*n_batch);
    # 创建一个名为t11_2的张量，使用ggml_transpose函数对t11_1进行转置，设置名称为"t11_2"，并断言其形状为2维，大小为N * n_batch * n_embd_gqa
    struct ggml_tensor * t11_2 = ggml_transpose(ctx, t11_1); set_name(t11_2, "t11_2"); assert_shape_2d(t11_2, N*n_batch, n_embd_gqa);
    # 创建一个名为t11的张量，使用ggml_cont函数对t11_2进行连接，设置名称为"t11"，并断言其形状为2维，大小为N * n_batch * n_embd_gqa
    t11 = ggml_cont(ctx, t11_2); set_name(t11, "t11"); assert_shape_2d(t11, N*n_batch, n_embd_gqa);
} else {
    # 创建一个名为t11的张量，使用ggml_mul_mat函数对t04和wv进行矩阵乘法，设置名称为"t11"，并断言其形状为2维，大小为N * n_batch * n_embd_gqa
    t11 = ggml_mul_mat(ctx, t04, wv); set_name(t11, "t11"); assert_shape_2d(t11, N*n_batch, n_embd_gqa);
}
# 创建一个名为 t12 的张量，通过对 t11 进行四维重塑操作
t12 = ggml_reshape_4d(ctx, t11, N, n_batch, n_embd_head, n_head_kv)
# 为张量 t12 设置名称
set_name(t12, "t12")
# 断言张量 t12 的形状为四维
assert_shape_4d(t12, N, n_batch, n_embd_head, n_head_kv)

# 创建一个名为 t13 的张量，通过对 t07 进行置换操作
t13 = ggml_permute(ctx, t07, 0, 2, 1, 3)
# 为张量 t13 设置名称
set_name(t13, "t13")
# 断言张量 t13 的形状为四维
assert_shape_4d(t13, n_embd_head, N, n_head, n_batch)

# 创建一个名为 t14 的张量，通过对 t10 进行置换操作
t14 = ggml_permute(ctx, t10, 0, 2, 1, 3)
# 为张量 t14 设置名称
set_name(t14, "t14")
# 断言张量 t14 的形状为四维
assert_shape_4d(t14, n_embd_head, N, n_head_kv, n_batch)

# 创建一个名为 t15 的张量，通过对 t12 进行置换操作
t15 = ggml_permute(ctx, t12, 0, 3, 1, 2)
# 为张量 t15 设置名称
set_name(t15, "t15")
# 断言张量 t15 的形状为四维
assert_shape_4d(t15, N, n_embd_head, n_head_kv, n_batch)

# 创建一个名为 t16 的张量
struct ggml_tensor * t16;
# 如果启用了 flash attention
if (enable_flash_attn) {
    # 使用 ggml_flash_attn 函数对 t13、t14、t15 进行操作
    t16 = ggml_flash_attn(ctx, t13, t14, t15, true)
    # 为张量 t16 设置名称
    set_name(t16, "t16")
    # 断言张量 t16 的形状为四维
    assert_shape_4d(t16, n_embd_head, N, n_head, n_batch)
} else {
    # 使用 ggml_mul_mat 函数对 t14、t13 进行操作
    struct ggml_tensor * t16_0 = ggml_mul_mat(ctx, t14, t13)
    # 为张量 t16_0 设置名称
    set_name(t16_0, "t16_0")
    # 断言张量 t16_0 的形状为四维
    assert_shape_4d(t16_0, N, N, n_head, n_batch)
    # 使用 ggml_scale_inplace 函数对 t16_0 进行操作
    struct ggml_tensor * t16_1 = ggml_scale_inplace(ctx, t16_0, kv_scale)
    # 为张量 t16_1 设置名称
    set_name(t16_1, "t16_1")
    # 断言张量 t16_1 的形状为四维
    assert_shape_4d(t16_1, N, N, n_head, n_batch)
    # 使用 ggml_diag_mask_inf_inplace 函数对 t16_1 进行操作
    struct ggml_tensor * t16_2 = ggml_diag_mask_inf_inplace(ctx, t16_1, n_past)
    # 为张量 t16_2 设置名称
    set_name(t16_2, "t16_2")
    # 断言张量 t16_2 的形状为四维
    assert_shape_4d(t16_2, N, N, n_head, n_batch)
    # 使用 ggml_soft_max_inplace 函数对 t16_2 进行操作
    struct ggml_tensor * t16_3 = ggml_soft_max_inplace(ctx, t16_2)
    # 为张量 t16_3 设置名称
    set_name(t16_3, "t16_3")
    # 断言张量 t16_3 的形状为四维
    assert_shape_4d(t16_3, N, N, n_head, n_batch)
    # 使用 ggml_mul_mat 函数对 t15、t16_3 进行操作
    t16 = ggml_mul_mat(ctx, t15, t16_3)
    # 为张量 t16 设置名称
    set_name(t16, "t16")
    # 断言张量 t16 的形状为四维
    assert_shape_4d(t16, n_embd_head, N, n_head, n_batch)
}

# 创建一个名为 t17 的张量，通过对 t16 进行置换操作
t17 = ggml_permute(ctx, t16, 0, 2, 1, 3)
# 为张量 t17 设置名称
set_name(t17, "t17")
# 断言张量 t17 的形状为四维
assert_shape_4d(t17, n_embd_head, n_head, N, n_batch)

# 创建一个名为 t18 的张量，通过对 t17 进行连接操作
t18 = ggml_cont(ctx, t17)
# 为张量 t18 设置名称
set_name(t18, "t18")
# 断言张量 t18 的形状为四维
assert_shape_4d(t18, n_embd_head, n_head, N, n_batch)

# 创建一个名为 t19 的张量，通过对 t18 进行二维重塑操作
t19 = ggml_reshape_2d(ctx, t18, n_embd, N*n_batch)
# 为张量 t19 设置名称
set_name(t19, "t19")
# 断言张量 t19 的形状为二维
assert_shape_2d(t19, n_embd, N*n_batch)

# 创建一个名为 t20 的张量，通过对 wo 和 t19 进行矩阵乘法操作
t20 = ggml_mul_mat(ctx, wo, t19)
# 为张量 t20 设置名称
set_name(t20, "t20")
# 断言张量 t20 的形状为二维
assert_shape_2d(t20, n_embd, N*n_batch)

# 创建一个名为 t21 的张量，通过对 t20 和 cur 进行加法操作
t21 = ggml_add(ctx, t20, cur)
# 为张量 t21 设置名称
set_name(t21, "t21")
# 断言张量 t21 的形状为二维
assert_shape_2d(t21, n_embd, N*n_batch)

# 创建一个名为 t22 的张量，通过对 t21 进行 RMS 归一化操作
t22 = ggml_rms_norm(ctx, t21, rms_norm_eps)
# 为张量 t22 设置名称
set_name(t22, "t22")
# 断言张量 t22 的形状为二维
assert_shape_2d(t22, n_embd, N*n_batch)
# 创建一个新的张量 t23，其值为张量 ffn_norm 重复 N*n_batch 次后的结果
# 设置张量名称为 "t23"
# 断言张量 t23 的形状为二维，行数为 n_embd，列数为 N*n_batch
struct ggml_tensor * t23 = ggml_repeat(ctx, ffn_norm, t22); set_name(t23, "t23"); assert_shape_2d(t23, n_embd, N*n_batch);

# 创建一个新的张量 t24，其值为张量 t23 与张量 t22 的逐元素相乘的结果
# 设置张量名称为 "t24"
# 断言张量 t24 的形状为二维，行数为 n_embd，列数为 N*n_batch
struct ggml_tensor * t24 = ggml_mul(ctx, t23, t22); set_name(t24, "t24"); assert_shape_2d(t24, n_embd, N*n_batch);

# 创建一个新的张量 t25，其值为张量 w3 与张量 t24 的矩阵相乘的结果
# 设置张量名称为 "t25"
# 断言张量 t25 的形状为二维，行数为 n_ff，列数为 N*n_batch
struct ggml_tensor * t25 = ggml_mul_mat(ctx, w3, t24); set_name(t25, "t25"); assert_shape_2d(t25, n_ff, N*n_batch);

# 创建一个新的张量 t26，其值为张量 w1 与张量 t24 的矩阵相乘的结果
# 设置张量名称为 "t26"
# 断言张量 t26 的形状为二维，行数为 n_ff，列数为 N*n_batch
struct ggml_tensor * t26 = ggml_mul_mat(ctx, w1, t24); set_name(t26, "t26"); assert_shape_2d(t26, n_ff, N*n_batch);

# 创建一个新的张量 t27，其值为张量 t26 的每个元素经过 sigmoid 激活函数处理后的结果
# 设置张量名称为 "t27"
# 断言张量 t27 的形状为二维，行数为 n_ff，列数为 N*n_batch
struct ggml_tensor * t27 = ggml_silu(ctx, t26); set_name(t27, "t27"); assert_shape_2d(t27, n_ff, N*n_batch);

# 创建一个新的张量 t28，其值为张量 t27 与张量 t25 的逐元素相乘的结果
# 设置张量名称为 "t28"
# 断言张量 t28 的形状为二维，行数为 n_ff，列数为 N*n_batch
struct ggml_tensor * t28 = ggml_mul(ctx, t27, t25); set_name(t28, "t28"); assert_shape_2d(t28, n_ff, N*n_batch);

# 创建一个新的张量 t29，其值为张量 w2 与张量 t28 的矩阵相乘的结果
# 设置张量名称为 "t29"
# 断言张量 t29 的形状为二维，行数为 n_embd，列数为 N*n_batch
struct ggml_tensor * t29 = ggml_mul_mat(ctx, w2, t28); set_name(t29, "t29"); assert_shape_2d(t29, n_embd, N*n_batch);

# 创建一个新的张量 t30，其值为张量 t29 与张量 t21 的逐元素相加的结果
# 设置张量名称为 "t30"
# 断言张量 t30 的形状为二维，行数为 n_embd，列数为 N*n_batch
struct ggml_tensor * t30 = ggml_add(ctx, t29, t21); set_name(t30, "t30"); assert_shape_2d(t30, n_embd, N*n_batch);

# 将当前张量 cur 设置为 t30
cur = t30;

# 如果启用了检查点功能，则将当前张量 cur 加入检查点列表
if (enable_checkpointing) {
    checkpoints.push_back(cur);
}

# 创建一个新的张量 t31，其值为张量 cur 经过 RMS 归一化处理后的结果
# 设置张量名称为 "t31"
# 断言张量 t31 的形状为二维，行数为 n_embd，列数为 N*n_batch
struct ggml_tensor * t31 = ggml_rms_norm(ctx, cur, rms_norm_eps); set_name(t31, "t31"); assert_shape_2d(t31, n_embd, N*n_batch);

# 创建一个新的张量 t32，其值为张量 norm 重复 N*n_batch 次后与张量 t31 的逐元素相乘的结果
# 设置张量名称为 "t32"
# 断言张量 t32 的形状为二维，行数为 n_embd，列数为 N*n_batch
struct ggml_tensor * t32 = ggml_repeat(ctx, norm, t31); set_name(t32, "t32"); assert_shape_2d(t32, n_embd, N*n_batch);

# 创建一个新的张量 t33，其值为张量 t32 与张量 t31 的逐元素相乘的结果
# 设置张量名称为 "t33"
# 断言张量 t33 的形状为二维，行数为 n_embd，列数为 N*n_batch
struct ggml_tensor * t33 = ggml_mul(ctx, t32, t31); set_name(t33, "t33"); assert_shape_2d(t33, n_embd, N*n_batch);

# 创建一个新的张量 t34，其值为张量 output 与张量 t33 的矩阵相乘的结果
# 设置张量名称为 "t34"
# 断言张量 t34 的形状为二维，行数为 n_vocab，列数为 N*n_batch
struct ggml_tensor * t34 = ggml_mul_mat(ctx, output, t33); set_name(t34, "t34"); assert_shape_2d(t34, n_vocab, N*n_batch);

# 创建一个新的张量 t35，其值为张量 t34 重塑为三维张量，形状为 (n_vocab, N, n_batch)
# 设置张量名称为 "t35"
# 断言张量 t35 的形状为三维，第一维长度为 n_vocab，第二维长度为 N，第三维长度为 n_batch
struct ggml_tensor * t35 = ggml_reshape_3d(ctx, t34, n_vocab, N, n_batch); set_name(t35, "t35"); assert_shape_3d(t35, n_vocab, N, n_batch);

# 创建一个新的张量 t36，其值为张量 t35 与目标张量 targets 的交叉熵损失
# 设置张量名称为 "t36"
# 断言张量 t36 的形状为一维，长度为 1
struct ggml_tensor * t36 = ggml_cross_entropy_loss(ctx, t35, targets); set_name(t36, "t36"); assert_shape_1d(t36, 1);
# 如果启用了检查点功能
if (enable_checkpointing) {
    # 将 t31 到 t36 的值依次添加到检查点列表中
    checkpoints.push_back(t31);
    checkpoints.push_back(t32);
    checkpoints.push_back(t33);
    checkpoints.push_back(t34);
    checkpoints.push_back(t35);
    checkpoints.push_back(t36);
}

# 调用 ggml_build_forward_expand 函数，传入 gf 和 t36 作为参数
ggml_build_forward_expand(gf, t36);

# 如果启用了检查点功能
if (enable_checkpointing) {
    # 调用 ggml_build_backward_gradient_checkpointing 函数，传入 ctx, gf, gb, gb_tmp, checkpoints.data(), (int) checkpoints.size() 作为参数
    ggml_build_backward_gradient_checkpointing(ctx, gf, gb, gb_tmp, checkpoints.data(), (int) checkpoints.size());
} else {
    # 如果未启用检查点功能，则调用 ggml_graph_cpy 函数，将 gf 的值复制给 gb
    ggml_graph_cpy(gf, gb);
    # 调用 ggml_build_backward_expand 函数，传入 ctx, gf, gb, true 作为参数
    ggml_build_backward_expand(ctx, gf, gb, true);
}

# 断言 alloc 不为空
GGML_ASSERT(alloc != NULL);
    // 确保一些张量不会被重新分配，通过插入依赖于它们的新临时节点
    int n_leafs_before = gb->n_leafs;  // 保存操作前叶子节点的数量
    int n_nodes_before = gb->n_nodes;  // 保存操作前节点的数量
    struct ggml_tensor * one = ggml_new_f32(ctx, 1.0f);  // 创建一个数值为1.0的张量
    // 输出张量
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t35, one));  // 对 t35 进行缩放操作
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t36, one));  // 对 t36 进行缩放操作
    // 输入梯度
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t36->grad, one));  // 对 t36 的梯度进行缩放操作
    GGML_ASSERT(t36->grad->data == NULL && t36->grad->view_src == NULL);  // 断言 t36 的梯度数据为空
    ggml_allocr_alloc(alloc, t36->grad);  // 分配 t36 的梯度数据
    // KQ_pos
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, KQ_pos, one));  // 对 KQ_pos 进行缩放操作

    // 确保基础模型张量的数据不能在可视化操作中使用
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, model->tok_embeddings, one));  // 对 model->tok_embeddings 进行缩放操作
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, model->norm, one));  // 对 model->norm 进行缩放操作
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, model->output, one));  // 对 model->output 进行缩放操作
    for (int il = 0; il < n_layer; ++il) {
        struct my_llama_layer & layer = model->layers[il];  // 遍历模型的每一层
    # 对输入的参数进行处理，并将结果传递给 ggml_build_forward_expand 函数
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.attention_norm, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.ffn_norm, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.wq, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.wk, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.wv, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.wo, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.w1, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.w2, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.w3, one));

    // 为了减少内存碎片化，将检查点一次性分配在一个块中
    // 注意：它们将以相反的顺序被释放
    for (unsigned int i = 0; i < checkpoints.size(); ++i) {
        // 如果检查点的数据和视图源都为空，则分配内存
        if (checkpoints[i]->data == NULL && checkpoints[i]->view_src == NULL) {
            ggml_allocr_alloc(alloc, checkpoints[i]);
        }
    }

    // 为图形分配内存
    ggml_allocr_alloc_graph(alloc, gb);
    // 删除额外的节点和叶子
    for (int i = n_leafs_before; i < gb->n_leafs; ++i) {
        gb->leafs[i] = NULL;
    }
    for (int i = n_nodes_before; i < gb->n_nodes; ++i) {
        gb->nodes[i] = NULL;
    }
    gb->n_leafs = n_leafs_before;
    gb->n_nodes = n_nodes_before;

    *logits = t35;
    return t36;
}

static void load_llama_lora_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model, struct my_llama_lora * lora) {
    // 注意：gguf_context 必须使用 f_ggml_ctx 进行初始化，并且 no_alloc=false，否则无法读取张量数据

    std::string arch;
// 创建一个大小为512的字符向量keybuf
std::vector<char> keybuf;
keybuf.resize(512);

// 从fctx中获取架构信息，并进行断言检查
GGUF_GET_KEY(fctx, arch, gguf_get_val_str, GGUF_TYPE_STRING, true, LLM_KV_GENERAL_ARCHITECTURE);
GGML_ASSERT(arch == "llama");

// 从fctx中获取文件类型信息，并进行断言检查
uint32_t ftype_u;
GGUF_GET_KEY(fctx, ftype_u, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_GENERAL_FILE_TYPE);
GGML_ASSERT((enum llama_ftype) ftype_u == LLAMA_FTYPE_ALL_F32);

// 创建一个名为my_llama_hparams的结构体，并从fctx中加载模型超参数
struct my_llama_hparams hparams;
load_model_hparams_gguf(fctx, &hparams, arch.c_str());

// 断言检查模型的超参数与hparams中的参数是否匹配
GGML_ASSERT(hparams.n_embd    == model->hparams.n_embd);
GGML_ASSERT(hparams.n_ff      == model->hparams.n_ff);
GGML_ASSERT(hparams.n_head    == model->hparams.n_head);
GGML_ASSERT(hparams.n_head_kv == model->hparams.n_head_kv);
GGML_ASSERT(hparams.n_layer   == model->hparams.n_layer);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_tok_embeddings, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_norm,           gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_output,         gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_OUTPUT);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_attention_norm, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_NORM);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_wq,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_Q);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_wk,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_K);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_wv,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_V);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_wo,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_OUT);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_ffn_norm,       gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_FFN_NORM);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_w1,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_FFN_GATE);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_w2,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_FFN_DOWN);
# 从上下文中获取指定键的值，并将其存储到指定的变量中
GGUF_GET_KEY(fctx, lora->hparams.n_rank_w3,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_FFN_UP);

# 初始化 LORA 模型
init_lora(model, lora);

# 通过名称将指定的张量从一个上下文复制到另一个上下文
copy_tensor_by_name(lora->tok_embeddings_a, f_ggml_ctx, ggml_get_name(lora->tok_embeddings_a));
copy_tensor_by_name(lora->tok_embeddings_b, f_ggml_ctx, ggml_get_name(lora->tok_embeddings_b));
copy_tensor_by_name(lora->norm_a,           f_ggml_ctx, ggml_get_name(lora->norm_a));
copy_tensor_by_name(lora->norm_b,           f_ggml_ctx, ggml_get_name(lora->norm_b));
copy_tensor_by_name(lora->output_a,         f_ggml_ctx, ggml_get_name(lora->output_a));
# 根据给定的名称复制张量数据到指定的上下文中
copy_tensor_by_name(lora->output_b,         f_ggml_ctx, ggml_get_name(lora->output_b));

# 遍历 Lora 模型的每一层
for (uint32_t i = 0; i < lora->layers.size(); ++i) {
    # 获取当前层的引用
    auto & layer = lora->layers[i];
    # 依次复制当前层的各个张量数据到指定的上下文中
    copy_tensor_by_name(layer.attention_norm_a, f_ggml_ctx, ggml_get_name(layer.attention_norm_a));
    copy_tensor_by_name(layer.attention_norm_b, f_ggml_ctx, ggml_get_name(layer.attention_norm_b));
    copy_tensor_by_name(layer.wq_a,             f_ggml_ctx, ggml_get_name(layer.wq_a));
    copy_tensor_by_name(layer.wq_b,             f_ggml_ctx, ggml_get_name(layer.wq_b));
    copy_tensor_by_name(layer.wk_a,             f_ggml_ctx, ggml_get_name(layer.wk_a));
    copy_tensor_by_name(layer.wk_b,             f_ggml_ctx, ggml_get_name(layer.wk_b));
    copy_tensor_by_name(layer.wv_a,             f_ggml_ctx, ggml_get_name(layer.wv_a));
    copy_tensor_by_name(layer.wv_b,             f_ggml_ctx, ggml_get_name(layer.wv_b));
    copy_tensor_by_name(layer.wo_a,             f_ggml_ctx, ggml_get_name(layer.wo_a));
    copy_tensor_by_name(layer.wo_b,             f_ggml_ctx, ggml_get_name(layer.wo_b));
    copy_tensor_by_name(layer.ffn_norm_a,       f_ggml_ctx, ggml_get_name(layer.ffn_norm_a));
    copy_tensor_by_name(layer.ffn_norm_b,       f_ggml_ctx, ggml_get_name(layer.ffn_norm_b));
    copy_tensor_by_name(layer.w1_a,             f_ggml_ctx, ggml_get_name(layer.w1_a));
    copy_tensor_by_name(layer.w1_b,             f_ggml_ctx, ggml_get_name(layer.w1_b));
    copy_tensor_by_name(layer.w2_a,             f_ggml_ctx, ggml_get_name(layer.w2_a));
    copy_tensor_by_name(layer.w2_b,             f_ggml_ctx, ggml_get_name(layer.w2_b));
}
// 通过名称复制张量数据到指定的上下文中
copy_tensor_by_name(layer.w3_a, f_ggml_ctx, ggml_get_name(layer.w3_a));
// 通过名称复制张量数据到指定的上下文中
copy_tensor_by_name(layer.w3_b, f_ggml_ctx, ggml_get_name(layer.w3_b));

// 保存LLAMA LORA GGUF结构的函数
static void save_llama_lora_gguf(struct gguf_context * fctx, struct my_llama_model * model, struct my_llama_lora * lora) {
    // 定义架构和文件类型
    const char * arch = "llama";
    enum llama_ftype ftype = LLAMA_FTYPE_ALL_F32;

    // 创建用于存储键的缓冲区
    std::vector<char> keybuf;
    keybuf.resize(512);
    // 定义一个lambda函数，用于生成键值
    auto kv = [arch, &keybuf](const char * key) -> const char * {
        snprintf(keybuf.data(), keybuf.size(), key, arch);
        return keybuf.data();
    };

    // 设置LLAMA GGUF上下文的架构和文件类型
    gguf_set_val_str(fctx, LLM_KV_GENERAL_ARCHITECTURE, arch);
    gguf_set_val_u32(fctx, LLM_KV_GENERAL_FILE_TYPE, ftype);

    // 设置LLAMA GGUF上下文的上下文长度
    gguf_set_val_u32(fctx, kv(LLM_KV_CONTEXT_LENGTH), model->hparams.n_ctx);
# 设置模型参数中的嵌入长度
gguf_set_val_u32(fctx, kv(LLM_KV_EMBEDDING_LENGTH), model->hparams.n_embd);
# 设置模型参数中的前馈神经网络长度
gguf_set_val_u32(fctx, kv(LLM_KV_FEED_FORWARD_LENGTH), model->hparams.n_ff);
# 设置模型参数中的注意力头数
gguf_set_val_u32(fctx, kv(LLM_KV_ATTENTION_HEAD_COUNT), model->hparams.n_head);
# 设置模型参数中的键值对注意力头数
gguf_set_val_u32(fctx, kv(LLM_KV_ATTENTION_HEAD_COUNT_KV), model->hparams.n_head_kv);
# 设置模型参数中的块数量
gguf_set_val_u32(fctx, kv(LLM_KV_BLOCK_COUNT), model->hparams.n_layer);
# 设置模型参数中的绳索维度数量
gguf_set_val_u32(fctx, kv(LLM_KV_ROPE_DIMENSION_COUNT), model->hparams.n_embd_head());
# 设置模型参数中的注意力层归一化的 RMS 误差
gguf_set_val_f32(fctx, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS), model->hparams.f_norm_rms_eps);
# 设置模型参数中的绳索频率基数
gguf_set_val_f32(fctx, kv(LLM_KV_ROPE_FREQ_BASE), model->hparams.rope_freq_base);
# 设置模型参数中的绳索线性缩放
gguf_set_val_f32(fctx, kv(LLM_KV_ROPE_SCALE_LINEAR), model->hparams.rope_freq_scale);

# 设置 LORA 模型参数中的排名令牌嵌入数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD, lora->hparams.n_rank_tok_embeddings);
# 设置 LORA 模型参数中的排名输出归一化数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM, lora->hparams.n_rank_norm);
# 设置 LORA 模型参数中的排名输出数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_OUTPUT, lora->hparams.n_rank_output);
# 设置 LORA 模型参数中的排名注意力归一化数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_NORM, lora->hparams.n_rank_attention_norm);
# 设置 LORA 模型参数中的排名注意力查询数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_Q, lora->hparams.n_rank_wq);
# 设置 LORA 模型参数中的排名注意力键数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_K, lora->hparams.n_rank_wk);
# 设置 LORA 模型参数中的排名注意力值数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_V, lora->hparams.n_rank_wv);
# 设置 LORA 模型参数中的排名注意力输出数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_OUT, lora->hparams.n_rank_wo);
# 设置 LORA 模型参数中的排名前馈神经网络归一化数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_FFN_NORM, lora->hparams.n_rank_ffn_norm);
# 设置 LORA 模型参数中的排名前馈神经网络门数量
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_FFN_GATE, lora->hparams.n_rank_w1);
# 设置指定上行和下行的 LoRa 排名参数
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_FFN_DOWN,     lora->hparams.n_rank_w2);
gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_FFN_UP,       lora->hparams.n_rank_w3);

# 将指定的张量添加到训练上下文中
gguf_add_tensor(fctx, lora->tok_embeddings_a);
gguf_add_tensor(fctx, lora->tok_embeddings_b);
gguf_add_tensor(fctx, lora->norm_a);
gguf_add_tensor(fctx, lora->norm_b);
gguf_add_tensor(fctx, lora->output_a);
gguf_add_tensor(fctx, lora->output_b);

# 遍历 LoRa 模型的每一层，将指定的张量添加到训练上下文中
for (uint32_t i = 0; i < lora->layers.size(); ++i) {
    auto & layer = lora->layers[i];

    gguf_add_tensor(fctx, layer.attention_norm_a);
    gguf_add_tensor(fctx, layer.attention_norm_b);
    gguf_add_tensor(fctx, layer.wq_a);
    gguf_add_tensor(fctx, layer.wq_b);
    gguf_add_tensor(fctx, layer.wk_a);
    gguf_add_tensor(fctx, layer.wk_b);
    gguf_add_tensor(fctx, layer.wv_a);
}
// 将 layer.wv_b 添加到 fctx 中
gguf_add_tensor(fctx, layer.wv_b);
// 将 layer.wo_a 添加到 fctx 中
gguf_add_tensor(fctx, layer.wo_a);
// 将 layer.wo_b 添加到 fctx 中
gguf_add_tensor(fctx, layer.wo_b);
// 将 layer.ffn_norm_a 添加到 fctx 中
gguf_add_tensor(fctx, layer.ffn_norm_a);
// 将 layer.ffn_norm_b 添加到 fctx 中
gguf_add_tensor(fctx, layer.ffn_norm_b);
// 将 layer.w1_a 添加到 fctx 中
gguf_add_tensor(fctx, layer.w1_a);
// 将 layer.w1_b 添加到 fctx 中
gguf_add_tensor(fctx, layer.w1_b);
// 将 layer.w2_a 添加到 fctx 中
gguf_add_tensor(fctx, layer.w2_a);
// 将 layer.w2_b 添加到 fctx 中
gguf_add_tensor(fctx, layer.w2_b);
// 将 layer.w3_a 添加到 fctx 中
gguf_add_tensor(fctx, layer.w3_a);
// 将 layer.w3_b 添加到 fctx 中
gguf_add_tensor(fctx, layer.w3_b);
// 结束 if 语句块
}

// 加载 LORA 模型的检查点
static void load_checkpoint_lora_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model, struct my_llama_lora * lora, struct train_state * train) {
    // 设置训练类型为 FINETUNE_LORA
    std::string train_type = LLM_KV_TRAINING_TYPE_FINETUNE_LORA;
    // 从 fctx 中获取训练类型的值
    GGUF_GET_KEY(fctx, train_type, gguf_get_val_str, GGUF_TYPE_STRING, false, LLM_KV_TRAINING_TYPE);
    // 断言训练类型为 FINETUNE_LORA
    GGML_ASSERT(train_type == LLM_KV_TRAINING_TYPE_FINETUNE_LORA);

    // 加载训练状态的 GGUF
    load_train_state_gguf(fctx, f_ggml_ctx, train);
// 从文件上下文中加载 LLAMA LORA GGUF 数据
static void load_llama_lora_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model, struct my_llama_lora * lora) {
    // 从上下文中加载 LLAMA LORA GGUF 数据
}

// 保存 LORA GGUF 检查点
static void save_checkpoint_lora_gguf(struct gguf_context * fctx, struct my_llama_model * model, struct my_llama_lora * lora, struct train_state * train) {
    // 设置训练类型为 FINETUNE LORA
    gguf_set_val_str(fctx, LLM_KV_TRAINING_TYPE, LLM_KV_TRAINING_TYPE_FINETUNE_LORA);
    // 保存 LLAMA LORA GGUF 数据
    save_llama_lora_gguf(fctx, model, lora);
    // 保存训练状态的 GGUF 数据
    save_train_state_gguf(fctx, train);
}

// 从文件中加载 LORA 文件的检查点
static bool load_checkpoint_lora_file(const char * filename, struct my_llama_model * model, struct my_llama_lora * lora, struct train_state * train) {
    // 初始化 GGUF 上下文参数
    struct ggml_context * f_ggml_ctx;
    struct gguf_init_params params;
    params.no_alloc = false;
    params.ctx = &f_ggml_ctx;
    // 从文件中初始化 GGUF 上下文
    struct gguf_context * fctx = gguf_init_from_file(filename, params);
    // 如果初始化失败，则返回 false
    if (fctx == NULL) {
        return false;
    }
    // 从 GGUF 上下文中加载 LORA 文件的检查点
    load_checkpoint_lora_gguf(fctx, f_ggml_ctx, model, lora, train);
}
// 释放 gguf_context 对象
gguf_free(fctx);
// 返回 true
return true;
}

// 保存 LORA 模型到文件
static void save_checkpoint_lora_file(const char * filename, struct my_llama_model * model, struct my_llama_lora * lora, struct train_state * train) {
    // 打印保存文件的信息
    printf("%s: saving to %s\n", __func__, filename);
    // 初始化一个空的 gguf_context 对象
    struct gguf_context * fctx = gguf_init_empty();

    // 保存 LORA 模型到 gguf_context 对象
    save_checkpoint_lora_gguf(fctx, model, lora, train);

    // 写入文件
    const bool only_meta = false;
    gguf_write_to_file(fctx, filename, only_meta);
    // 释放 gguf_context 对象
    gguf_free(fctx);
}

// 定义一个 llama_file 结构体
struct llama_file {
    // 使用 FILE * 类型，这样我们就不必重新打开文件来进行内存映射
    FILE * fp;
    // 声明一个变量 size_t 类型的变量 size
    size_t size;

    // llama_file 构造函数，打开文件并获取文件大小
    llama_file(const char * fname, const char * mode) {
        // 打开文件
        fp = std::fopen(fname, mode);
        // 如果文件打开失败，将 size 置为 0
        if (fp == NULL) {
            size = 0;
        } else {
            // 定位到文件末尾，获取文件大小
            seek(0, SEEK_END);
            size = tell();
            // 定位回文件开头
            seek(0, SEEK_SET);
        }
    }

    // tell 函数，获取当前文件指针的位置
    size_t tell() const {
        // 根据操作系统不同选择不同的获取文件指针位置的方法
#ifdef _WIN32
        __int64 ret = _ftelli64(fp);
#else
        long ret = std::ftell(fp);
#endif
        // 断言获取文件指针位置不为 -1，即不应该失败
        GGML_ASSERT(ret != -1); // this really shouldn't fail
    // 将 ret 转换为 size_t 类型并返回
    return (size_t) ret;
}

void seek(size_t offset, int whence) {
#ifdef _WIN32
    // 在 Windows 平台下使用 _fseeki64 函数进行文件定位
    int ret = _fseeki64(fp, (__int64) offset, whence);
#else
    // 在其他平台下使用 std::fseek 函数进行文件定位
    int ret = std::fseek(fp, (long) offset, whence);
#endif
    // 断言文件定位操作是否成功
    GGML_ASSERT(ret == 0); // same
}

void read_raw(void * ptr, size_t size) {
    // 如果 size 为 0，则直接返回
    if (size == 0) {
        return;
    }
    // 将 errno 置为 0
    errno = 0;
    // 从文件中读取指定大小的数据到指定的内存地址中
    std::size_t ret = std::fread(ptr, size, 1, fp);
    // 如果发生读取错误，则输出错误信息并终止程序
    if (ferror(fp)) {
        die_fmt("read error: %s", strerror(errno));
    }
    // 如果返回值不等于1，表示读取到文件末尾，抛出异常
    if (ret != 1) {
        die("unexpectedly reached end of file");
    }
}

// 读取一个32位无符号整数
std::uint32_t read_u32() {
    std::uint32_t ret;
    // 读取指定大小的原始数据到ret中
    read_raw(&ret, sizeof(ret));
    return ret;
}

// 读取指定长度的字符串
std::string read_string(std::uint32_t len) {
    // 创建一个指定长度的字符向量
    std::vector<char> chars(len);
    // 读取指定长度的原始数据到字符向量中
    read_raw(chars.data(), len);
    // 将字符向量转换为字符串并返回
    return std::string(chars.data(), len);
}

// 写入指定大小的原始数据
void write_raw(const void * ptr, size_t size) {
    // 如果大小为0，直接返回
// 如果文件指针为空，直接返回
return;
// 清空错误码
errno = 0;
// 将数据写入文件，返回写入的数据块数
size_t ret = std::fwrite(ptr, size, 1, fp);
// 如果写入的数据块数不等于1，输出错误信息
if (ret != 1) {
    die_fmt("write error: %s", strerror(errno));
}

// 写入一个无符号32位整数
void write_u32(std::uint32_t val) {
    write_raw(&val, sizeof(val));
}

// 析构函数，关闭文件指针
~llama_file() {
    if (fp) {
        std::fclose(fp);
    }
}
// 写入张量数据到文件
static void write_tensor(struct llama_file * file, struct ggml_tensor * tensor, const char * name) {
    // 如果张量为空，写入0作为标记，并返回
    if (tensor == NULL) {
        file->write_u32(0);
        file->write_u32(0);
        file->write_u32(GGML_TYPE_F32);
        file->seek((0-file->tell()) & 31, SEEK_CUR);
        return;
    }
    // 如果名称为空，使用张量的默认名称
    if (name == NULL) {
        name = ggml_get_name(tensor);
    }
    // 计算名称长度和张量维度
    uint32_t name_len = strlen(name);
    uint32_t nd = tensor->n_dims;
    uint32_t ne[4] = { (uint32_t)tensor->ne[0],
                       (uint32_t)tensor->ne[1],
                       (uint32_t)tensor->ne[2],
                       (uint32_t)tensor->ne[3] };
    // 写入张量维度和名称长度
    file->write_u32(nd);
    file->write_u32(name_len);
    // 写入张量类型
    file->write_u32(tensor->type);
    // 将数据写入文件，ne为数据，nd为数据长度
    file->write_raw(ne, sizeof(ne[0]) * nd);
    // 将名称写入文件，name为名称，name_len为名称长度
    file->write_raw(name, name_len);
    // 移动文件指针到下一个32字节对齐位置
    file->seek((0-file->tell()) & 31, SEEK_CUR);
    // 将张量数据写入文件
    file->write_raw(tensor->data, ggml_nbytes(tensor));
}

// 将数据保存为llama_lora格式
static void save_as_llama_lora(const char * filename, struct my_llama_lora * lora) {
    // 打印保存信息
    printf("%s: saving to %s\n", __func__, filename);
    // 创建llama_file对象，以写入二进制模式打开文件
    struct llama_file file(filename, "wb");
    // 如果文件指针为空，直接返回
    if (file.fp == NULL) {
        return;
    }

    // 创建存储名称的缓冲区
    std::vector<char> tn_buf;
    tn_buf.resize(GGML_MAX_NAME);

    // 定义一个lambda函数，用于拼接名称和后缀
    auto tn = [&tn_buf](const char * key, const char * suffix) -> const char * {
        // 格式化名称和后缀，存储到tn_buf中
        snprintf(tn_buf.data(), tn_buf.size(), "%s%s", key, suffix);
        // 返回拼接后的名称
        return tn_buf.data();
    };
    // 定义一个lambda函数，用于根据key、bid和suffix生成文件名
    auto tni = [&tn_buf](const char * key, int bid, const char * suffix) -> const char * {
        // 使用snprintf将key和bid格式化到tn_buf中
        snprintf(tn_buf.data(), tn_buf.size(), key, bid);
        // 将tn_buf转换为string类型
        std::string s = tn_buf.data();
        // 将s和suffix拼接到tn_buf中
        snprintf(tn_buf.data(), tn_buf.size(), "%s%s", s.c_str(), suffix);
        // 返回拼接后的文件名
        return tn_buf.data();
    };

    // 定义LLAMA_FILE_MAGIC_LORA的值为0x67676C61
    uint32_t LLAMA_FILE_MAGIC_LORA = 0x67676C61; // 'ggla'
    // 写入LLAMA_FILE_MAGIC_LORA的值到文件中
    file.write_u32(LLAMA_FILE_MAGIC_LORA);   // magic
    // 写入版本号1到文件中
    file.write_u32(1); // version
    // 写入lora的hparams中的值到文件中
    file.write_u32(lora->hparams.lora_r);
    file.write_u32(lora->hparams.lora_alpha);
    // 将lora的tok_embeddings_a、tok_embeddings_b、norm_a、norm_b写入文件中，使用tn函数生成文件名
    write_tensor(&file, lora->tok_embeddings_a, tn(LLM_TENSOR_TOKEN_EMBD,  ".weight.loraA"));
    write_tensor(&file, lora->tok_embeddings_b, tn(LLM_TENSOR_TOKEN_EMBD,  ".weight.loraB"));
    write_tensor(&file, lora->norm_a,           tn(LLM_TENSOR_OUTPUT_NORM, ".weight.loraA"));
    write_tensor(&file, lora->norm_b,           tn(LLM_TENSOR_OUTPUT_NORM, ".weight.loraB"));
# 将 lora->output_a 的数据写入文件中，文件名为 ".weight.loraA"
write_tensor(&file, lora->output_a,         tn(LLM_TENSOR_OUTPUT,      ".weight.loraA"));
# 将 lora->output_b 的数据写入文件中，文件名为 ".weight.loraB"
write_tensor(&file, lora->output_b,         tn(LLM_TENSOR_OUTPUT,      ".weight.loraB"));
# 遍历 lora->layers 中的每一层
for (uint32_t i = 0; i < lora->layers.size(); ++i) {
    # 获取当前层的引用
    auto & layer = lora->layers[i];
    # 将当前层的 attention_norm_a 数据写入文件中，文件名为 ".weight.loraA"
    write_tensor(&file, layer.attention_norm_a, tni(LLM_TENSOR_ATTN_NORM, i, ".weight.loraA"));
    # 将当前层的 attention_norm_b 数据写入文件中，文件名为 ".weight.loraB"
    write_tensor(&file, layer.attention_norm_b, tni(LLM_TENSOR_ATTN_NORM, i, ".weight.loraB"));
    # 将当前层的 wq_a 数据写入文件中，文件名为 ".weight.loraA"
    write_tensor(&file, layer.wq_a,             tni(LLM_TENSOR_ATTN_Q,    i, ".weight.loraA"));
    # 将当前层的 wq_b 数据写入文件中，文件名为 ".weight.loraB"
    write_tensor(&file, layer.wq_b,             tni(LLM_TENSOR_ATTN_Q,    i, ".weight.loraB"));
    # 将当前层的 wk_a 数据写入文件中，文件名为 ".weight.loraA"
    write_tensor(&file, layer.wk_a,             tni(LLM_TENSOR_ATTN_K,    i, ".weight.loraA"));
    # 将当前层的 wk_b 数据写入文件中，文件名为 ".weight.loraB"
    write_tensor(&file, layer.wk_b,             tni(LLM_TENSOR_ATTN_K,    i, ".weight.loraB"));
    # 将当前层的 wv_a 数据写入文件中，文件名为 ".weight.loraA"
    write_tensor(&file, layer.wv_a,             tni(LLM_TENSOR_ATTN_V,    i, ".weight.loraA"));
    # 将当前层的 wv_b 数据写入文件中，文件名为 ".weight.loraB"
    write_tensor(&file, layer.wv_b,             tni(LLM_TENSOR_ATTN_V,    i, ".weight.loraB"));
    # 将当前层的 wo_a 数据写入文件中，文件名为 ".weight.loraA"
    write_tensor(&file, layer.wo_a,             tni(LLM_TENSOR_ATTN_OUT,  i, ".weight.loraA"));
    # 将当前层的 wo_b 数据写入文件中，文件名为 ".weight.loraB"
    write_tensor(&file, layer.wo_b,             tni(LLM_TENSOR_ATTN_OUT,  i, ".weight.loraB"));
    # 将当前层的 ffn_norm_a 数据写入文件中，文件名为 ".weight.loraA"
    write_tensor(&file, layer.ffn_norm_a,       tni(LLM_TENSOR_FFN_NORM,  i, ".weight.loraA"));
    # 将当前层的 ffn_norm_b 数据写入文件中，文件名为 ".weight.loraB"
    write_tensor(&file, layer.ffn_norm_b,       tni(LLM_TENSOR_FFN_NORM,  i, ".weight.loraB"));
    # 将当前层的 w1_a 数据写入文件中，文件名为 ".weight.loraA"
    write_tensor(&file, layer.w1_a,             tni(LLM_TENSOR_FFN_GATE,  i, ".weight.loraA"));
    # 将当前层的 w1_b 数据写入文件中，文件名为 ".weight.loraB"
    write_tensor(&file, layer.w1_b,             tni(LLM_TENSOR_FFN_GATE,  i, ".weight.loraB"));
    # 将当前层的 w2_a 数据写入文件中，文件名为 ".weight.loraA"
    write_tensor(&file, layer.w2_a,             tni(LLM_TENSOR_FFN_DOWN,  i, ".weight.loraA"));
    # 将当前层的 w2_b 数据写入文件中，文件名为 ".weight.loraB"
    write_tensor(&file, layer.w2_b,             tni(LLM_TENSOR_FFN_DOWN,  i, ".weight.loraB"));
}
// 将 layer.w3_a 的数据写入文件中，文件名由 tni(LLM_TENSOR_FFN_UP, i, ".weight.loraA") 生成
write_tensor(&file, layer.w3_a, tni(LLM_TENSOR_FFN_UP, i, ".weight.loraA"));

// 将 layer.w3_b 的数据写入文件中，文件名由 tni(LLM_TENSOR_FFN_UP, i, ".weight.loraB") 生成
write_tensor(&file, layer.w3_b, tni(LLM_TENSOR_FFN_UP, i, ".weight.loraB"));
}

// 定义了一个结构体 train_params，包含了训练参数的通用部分和特定部分
struct train_params {
    struct train_params_common common; // 通用部分

    const char * fn_model_base; // 模型文件的基本名称
    const char * fn_lora_out; // LORA 输出文件的名称

    bool only_write_lora; // 是否只写入 LORA 数据

    float f_norm_rms_eps; // f_norm_rms_eps 参数
    float rope_freq_base; // rope_freq_base 参数
    float rope_freq_scale; // rope_freq_scale 参数

    bool custom_f_norm_rms_eps; // 是否使用自定义的 f_norm_rms_eps 参数
    bool custom_rope_freq_base; // 是否使用自定义的 rope_freq_base 参数
    bool custom_rope_freq_scale; // 是否使用自定义的 rope_freq_scale 参数
}
// 定义一个32位整数变量 lora_r
int32_t lora_r;
// 定义一个32位整数变量 lora_alpha
int32_t lora_alpha;
// 定义一个布尔变量 custom_lora_alpha
bool custom_lora_alpha;

// 定义一个32位无符号整数变量 n_rank_attention_norm
uint32_t n_rank_attention_norm;
// 定义一系列32位无符号整数变量 n_rank_wq, n_rank_wk, n_rank_wv, n_rank_wo, n_rank_ffn_norm, n_rank_w1, n_rank_w2, n_rank_w3, n_rank_tok_embeddings, n_rank_norm, n_rank_output
uint32_t n_rank_wq;
uint32_t n_rank_wk;
uint32_t n_rank_wv;
uint32_t n_rank_wo;
uint32_t n_rank_ffn_norm;
uint32_t n_rank_w1;
uint32_t n_rank_w2;
uint32_t n_rank_w3;
uint32_t n_rank_tok_embeddings;
uint32_t n_rank_norm;
uint32_t n_rank_output;

// 定义一系列布尔变量 custom_n_rank_attention_norm, custom_n_rank_wq
bool custom_n_rank_attention_norm;
bool custom_n_rank_wq;
// 定义布尔变量，用于标识是否使用自定义的排名权重
bool custom_n_rank_wk;
bool custom_n_rank_wv;
bool custom_n_rank_wo;
bool custom_n_rank_ffn_norm;
bool custom_n_rank_w1;
bool custom_n_rank_w2;
bool custom_n_rank_w3;
bool custom_n_rank_tok_embeddings;
bool custom_n_rank_norm;
bool custom_n_rank_output;

// 定义结构体，用于存储默认的训练参数
static struct train_params get_default_train_params() {
    struct train_params params;
    // 获取默认的通用训练参数
    params.common = get_default_train_params_common();
    // 初始化模型基础文件名为空字符串
    params.fn_model_base     = "";
    // 初始化 LORA 输出文件名
    params.fn_lora_out       = "ggml-lora-ITERATION-f32.gguf";
    // 初始化是否只写入 LORA 标志为 false
    params.only_write_lora = false;
    # 设置参数 f_norm_rms_eps 的值为 1e-5
    params.f_norm_rms_eps  = 1e-5f;
    # 设置参数 rope_freq_base 的值为 10000.0
    params.rope_freq_base  = 10000.0f;
    # 设置参数 rope_freq_scale 的值为 1.0
    params.rope_freq_scale = 1.0f;

    # 设置参数 custom_f_norm_rms_eps 为 false
    params.custom_f_norm_rms_eps  = false;
    # 设置参数 custom_rope_freq_base 为 false
    params.custom_rope_freq_base  = false;
    # 设置参数 custom_rope_freq_scale 为 false
    params.custom_rope_freq_scale = false;

    # 设置参数 lora_r 的值为 4
    params.lora_r      = 4;
    # 设置参数 lora_alpha 的值为 4
    params.lora_alpha  = 4;
    # 设置参数 custom_lora_alpha 为 false
    params.custom_lora_alpha = false;

    # 设置参数 n_rank_attention_norm 的值为 1
    params.n_rank_attention_norm = 1;
    # 设置参数 n_rank_wq 的值为 4
    params.n_rank_wq             = 4;
    # 设置参数 n_rank_wk 的值为 4
    params.n_rank_wk             = 4;
    # 设置参数 n_rank_wv 的值为 4
    params.n_rank_wv             = 4;
    # 设置参数 n_rank_wo 的值为 4
    params.n_rank_wo             = 4;
    # 设置参数 n_rank_ffn_norm 的值为 1
    params.n_rank_ffn_norm       = 1;
    # 设置参数 n_rank_w1 的值为 4
    params.n_rank_w1             = 4;
    # 设置参数 n_rank_w2 的值为 4
    params.n_rank_w2             = 4;
# 设置参数 params.n_rank_w3 的值为 4
params.n_rank_w3 = 4;
# 设置参数 params.n_rank_tok_embeddings 的值为 4
params.n_rank_tok_embeddings = 4;
# 设置参数 params.n_rank_norm 的值为 1
params.n_rank_norm = 1;
# 设置参数 params.n_rank_output 的值为 4
params.n_rank_output = 4;

# 设置参数 params.custom_n_rank_attention_norm 的值为 false
params.custom_n_rank_attention_norm = false;
# 设置参数 params.custom_n_rank_wq 的值为 false
params.custom_n_rank_wq = false;
# 设置参数 params.custom_n_rank_wk 的值为 false
params.custom_n_rank_wk = false;
# 设置参数 params.custom_n_rank_wv 的值为 false
params.custom_n_rank_wv = false;
# 设置参数 params.custom_n_rank_wo 的值为 false
params.custom_n_rank_wo = false;
# 设置参数 params.custom_n_rank_ffn_norm 的值为 false
params.custom_n_rank_ffn_norm = false;
# 设置参数 params.custom_n_rank_w1 的值为 false
params.custom_n_rank_w1 = false;
# 设置参数 params.custom_n_rank_w2 的值为 false
# 设置参数 params.custom_n_rank_w3 的值为 false
params.custom_n_rank_w3 = false;
# 设置参数 params.custom_n_rank_tok_embeddings 的值为 false
params.custom_n_rank_tok_embeddings = false;
# 设置参数 params.custom_n_rank_norm 的值为 false
params.custom_n_rank_norm = false;
# 设置参数 params.custom_n_rank_output 的值为 false
params.custom_n_rank_output = false;

# 返回参数对象 params
return params;
# 打印程序的使用方法和选项
static void train_print_usage(int argc, char ** argv, const struct train_params * params) {
    # 打印程序的使用方法
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    # 打印各种选项的说明
    fprintf(stderr, "  -h, --help                 show this help message and exit\n");
    fprintf(stderr, "  --model-base FNAME         model path from which to load base model (default '%s')\n", params->fn_model_base);
    fprintf(stderr, "  --lora-out FNAME           path to save llama lora (default '%s')\n", params->fn_lora_out);
    fprintf(stderr, "  --only-write-lora          only save llama lora, don't do any training.  use this if you only want to convert a checkpoint to a lora adapter.\n");
    fprintf(stderr, "  --norm-rms-eps F           RMS-Norm epsilon value (default %f)\n", params->f_norm_rms_eps);
    fprintf(stderr, "  --rope-freq-base F         Frequency base for ROPE (default %f)\n", params->rope_freq_base);
    fprintf(stderr, "  --rope-freq-scale F        Frequency scale for ROPE (default %f)\n", params->rope_freq_scale);
    fprintf(stderr, "  --lora-alpha N             LORA alpha : resulting LORA scaling is alpha/r. (default %d)\n", params->lora_alpha);
    fprintf(stderr, "  --lora-r N                 LORA r: default rank. Also specifies resulting scaling together with lora-alpha. (default %d)\n", params->lora_r);
    fprintf(stderr, "  --rank-att-norm N          LORA rank for attention norm tensor, overrides default rank. Norm tensors should generally have rank 1.\n");
    fprintf(stderr, "  --rank-ffn-norm N          LORA rank for feed-forward norm tensor, overrides default rank. Norm tensors should generally have rank 1.\n");
    fprintf(stderr, "  --rank-out-norm N          LORA rank for output norm tensor, overrides default rank. Norm tensors should generally have rank 1.\n");
    fprintf(stderr, "  --rank-tok-embd N          LORA rank for token embeddings tensor, overrides default rank.\n");
    fprintf(stderr, "  --rank-out N               LORA rank for output tensor, overrides default rank.\n");
    // 打印关于 LORA rank 的参数说明
    fprintf(stderr, "  --rank-wq N                LORA rank for wq tensor, overrides default rank.\n");
    fprintf(stderr, "  --rank-wk N                LORA rank for wk tensor, overrides default rank.\n");
    fprintf(stderr, "  --rank-wv N                LORA rank for wv tensor, overrides default rank.\n");
    fprintf(stderr, "  --rank-wo N                LORA rank for wo tensor, overrides default rank.\n");
    fprintf(stderr, "  --rank-w1 N                LORA rank for w1 tensor, overrides default rank.\n");
    fprintf(stderr, "  --rank-w2 N                LORA rank for w2 tensor, overrides default rank.\n");
    fprintf(stderr, "  --rank-w3 N                LORA rank for w3 tensor, overrides default rank.\n");

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
}

// 检查并处理通用的训练参数
if (consume_common_train_arg(argc, argv, &i, &params->common, &invalid_param)) {
    // 如果参数无效，则跳出循环
    if (invalid_param) {
        break;
    } 
    // 如果需要打印用法信息，则打印后退出程序
    else if (params->common.print_usage) {
        train_print_usage(argc, argv, &default_params);
        exit(0);
    }
} 
// 如果参数为"--model-base"
else if (arg == "--model-base") {
    // 如果参数不完整，则标记为无效参数并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    // 将参数赋值给模型基础文件名
    params->fn_model_base = argv[i];
} 
// 如果参数为"--lora-out"
else if (arg == "--lora-out") {
    // 如果参数不完整，则标记为无效参数并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
        } 
        # 如果参数是 "--fn-lora-out"，则将参数值赋给 fn_lora_out
        params->fn_lora_out = argv[i];
    } else if (arg == "--only-write-lora") {
        # 如果参数是 "--only-write-lora"，则将 only_write_lora 设置为 true
        params->only_write_lora = true;
    } else if (arg == "--norm-rms-eps") {
        # 如果参数是 "--norm-rms-eps"，则将下一个参数转换为浮点数并赋给 f_norm_rms_eps，同时设置 custom_f_norm_rms_eps 为 true
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params->f_norm_rms_eps = std::stof(argv[i]);
        params->custom_f_norm_rms_eps = true;
    } else if (arg == "--rope-freq-base") {
        # 如果参数是 "--rope-freq-base"，则将下一个参数转换为浮点数并赋给 rope_freq_base，同时设置 custom_rope_freq_base 为 true
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params->rope_freq_base = std::stof(argv[i]);
        params->custom_rope_freq_base = true;
    } else if (arg == "--rope-freq-scale") {
        # 如果参数是 "--rope-freq-scale"，则将下一个参数转换为浮点数并赋给 rope_freq_scale，同时设置 custom_rope_freq_scale 为 true
        if (++i >= argc) {
                # 检查参数是否有效，如果无效则设置标志位并跳出循环
                invalid_param = true;
                break;
            }
            # 将参数转换为浮点数，并赋值给params->rope_freq_scale
            params->rope_freq_scale = std::stof(argv[i]);
            # 设置自定义的rope_freq_scale标志位为true
            params->custom_rope_freq_scale = true;
        } else if (arg == "--lora-alpha") {
            # 检查参数是否有效，如果无效则设置标志位并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将参数转换为整数，并赋值给params->lora_alpha
            params->lora_alpha = std::stoi(argv[i]);
            # 设置自定义的lora_alpha标志位为true
            params->custom_lora_alpha = true;
        } else if (arg == "--lora-r") {
            # 检查参数是否有效，如果无效则设置标志位并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将参数转换为整数，并赋值给params->lora_r
            params->lora_r = std::stoi(argv[i]);
        } else if (arg == "--rank-att-norm") {
            # 检查参数是否有效，如果无效则设置标志位并跳出循环
# 检查参数是否为 "--rank-attention-norm"，如果是则执行以下操作
if (arg == "--rank-attention-norm") {
    # 检查参数是否越界
    if (++i >= argc) {
        # 参数越界，设置参数无效并跳出循环
        invalid_param = true;
        break;
    }
    # 将参数转换为整数并赋值给对应的参数变量，标记自定义参数已设置
    params->n_rank_attention_norm = std::stoi(argv[i]);
    params->custom_n_rank_attention_norm = true;
} 
# 如果参数为 "--rank-ffn-norm"，执行类似的操作
else if (arg == "--rank-ffn-norm") {
    # 检查参数是否越界
    if (++i >= argc) {
        # 参数越界，设置参数无效并跳出循环
        invalid_param = true;
        break;
    }
    # 将参数转换为整数并赋值给对应的参数变量，标记自定义参数已设置
    params->n_rank_ffn_norm = std::stoi(argv[i]);
    params->custom_n_rank_ffn_norm = true;
} 
# 如果参数为 "--rank-out-norm"，执行类似的操作
else if (arg == "--rank-out-norm") {
    # 检查参数是否越界
    if (++i >= argc) {
        # 参数越界，设置参数无效并跳出循环
        invalid_param = true;
        break;
    }
    # 将参数转换为整数并赋值给对应的参数变量，标记自定义参数已设置
    params->n_rank_norm = std::stoi(argv[i]);
    params->custom_n_rank_norm = true;
} 
# 如果参数为 "--rank-tok-embd"，执行相应操作
# 如果参数数量超出范围，设置参数无效并跳出循环
if (++i >= argc) {
    invalid_param = true;
    break;
}
# 将参数转换为整数并赋值给对应的参数变量
params->n_rank_tok_embeddings = std::stoi(argv[i]);
# 设置自定义的 n_rank_tok_embeddings 参数为 true
params->custom_n_rank_tok_embeddings = true;
# 如果参数为 "--rank-out"
} else if (arg == "--rank-out") {
    # 如果参数数量超出范围，设置参数无效并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    # 将参数转换为整数并赋值给对应的参数变量
    params->n_rank_output = std::stoi(argv[i]);
    # 设置自定义的 n_rank_output 参数为 true
    params->custom_n_rank_output = true;
# 如果参数为 "--rank-wq"
} else if (arg == "--rank-wq") {
    # 如果参数数量超出范围，设置参数无效并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    # 将参数转换为整数并赋值给对应的参数变量
    params->n_rank_wq = std::stoi(argv[i]);
    # 设置自定义的 n_rank_wq 参数为 true
    params->custom_n_rank_wq = true;
        } else if (arg == "--rank-wk") {
            // 如果参数是"--rank-wk"，则检查下一个参数是否存在，如果不存在则设置invalid_param为true并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数并赋值给params->n_rank_wk，同时设置custom_n_rank_wk为true
            params->n_rank_wk = std::stoi(argv[i]);
            params->custom_n_rank_wk = true;
        } else if (arg == "--rank-wv") {
            // 如果参数是"--rank-wv"，则检查下一个参数是否存在，如果不存在则设置invalid_param为true并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数并赋值给params->n_rank_wv，同时设置custom_n_rank_wv为true
            params->n_rank_wv = std::stoi(argv[i]);
            params->custom_n_rank_wv = true;
        } else if (arg == "--rank-wo") {
            // 如果参数是"--rank-wo"，则检查下一个参数是否存在，如果不存在则设置invalid_param为true并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数并赋值给params->n_rank_wo
            params->n_rank_wo = std::stoi(argv[i]);
        // 如果参数为"--rank-wo"，则设置custom_n_rank_wo为true
        if (arg == "--rank-wo") {
            params->custom_n_rank_wo = true;
        } 
        // 如果参数为"--rank-w1"，则获取下一个参数作为n_rank_w1的值，并设置custom_n_rank_w1为true
        else if (arg == "--rank-w1") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params->n_rank_w1 = std::stoi(argv[i]);
            params->custom_n_rank_w1 = true;
        } 
        // 如果参数为"--rank-w2"，则获取下一个参数作为n_rank_w2的值，并设置custom_n_rank_w2为true
        else if (arg == "--rank-w2") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params->n_rank_w2 = std::stoi(argv[i]);
            params->custom_n_rank_w2 = true;
        } 
        // 如果参数为"--rank-w3"，则获取下一个参数作为n_rank_w3的值，并设置custom_n_rank_w3为true
        else if (arg == "--rank-w3") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将参数转换为整数，并赋值给对应的参数变量
            params->n_rank_w3 = std::stoi(argv[i]);
            // 设置自定义的 n_rank_w3 参数为 true
            params->custom_n_rank_w3 = true;
        } else if (arg == "--gpu-layers" || arg == "-ngl" || arg == "--n-gpu-layers") {
            // 如果参数数量不足，设置 invalid_param 为 true 并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 如果编译时支持 GPU offload，则将参数转换为整数，并赋值给对应的参数变量
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
            params->common.n_gpu_layers = std::stoi(argv[i]);
#else
            // 如果不支持 GPU offload，则输出警告信息
            fprintf(stderr, "warning: not compiled with GPU offload support, --n-gpu-layers option will be ignored\n");
            fprintf(stderr, "warning: see main README.md for information on enabling GPU BLAS support\n");
#endif
        } else {
            // 输出错误信息，打印用法，并退出程序
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            train_print_usage(argc, argv, &default_params);
            exit(1);
        }
    }
    // 如果存在无效参数，则执行以下代码
    if (invalid_param) {
    // 打印错误信息，指出参数无效
    fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
    // 打印训练的用法信息
    train_print_usage(argc, argv, &default_params);
    // 退出程序，返回错误状态码
    exit(1);
    // 处理训练参数，完成训练参数的处理
    finish_processing_train_args(&params->common);
    // 返回 true，表示处理训练参数成功
    return true;
}

// 保存训练文件的数据结构
struct save_train_files_data {
    const char            * fn_checkpoint_out; // 检查点输出文件名
    const char            * fn_lora_out; // Lora 输出文件名
    const char            * pattern_fn_it; // 模式文件名
    const char            * fn_latest; // 最新文件名
    struct my_llama_model * model; // Llama 模型
    struct my_llama_lora  * lora; // Llama Lora
};

// 保存训练文件的函数
static void save_train_files(void * vdata, struct train_state * train) {
    // 将 void 指针转换为保存训练文件的数据结构指针
    struct save_train_files_data * data   = (struct save_train_files_data *) vdata;
    # 获取训练迭代次数
    int64_t iter = train->opt->iter;

    # 如果指定了保存检查点的文件名
    if (strlen(data->fn_checkpoint_out) > 0) {
        # 保存检查点文件，包括当前迭代次数的模型和数据
        save_checkpoint_lora_file(get_train_filename(data->fn_checkpoint_out, data->pattern_fn_it, data->fn_latest, iter).c_str(), data->model, data->lora, train);
        # 保存检查点文件，包括最新迭代次数的模型和数据
        save_checkpoint_lora_file(get_train_filename(data->fn_checkpoint_out, data->pattern_fn_it, data->fn_latest, -1  ).c_str(), data->model, data->lora, train);
    }
    # 如果指定了保存 LORA 数据的文件名
    if (strlen(data->fn_lora_out) > 0) {
        # 保存 LORA 数据，包括当前迭代次数的数据
        save_as_llama_lora(get_train_filename(data->fn_lora_out, data->pattern_fn_it, data->fn_latest, iter).c_str(), data->lora);
        # 保存 LORA 数据，包括最新迭代次数的数据
        save_as_llama_lora(get_train_filename(data->fn_lora_out, data->pattern_fn_it, data->fn_latest, -1  ).c_str(), data->lora);
    }
}

# 获取 LORA 模型参数的数量
static int64_t get_parameter_count(struct my_llama_lora* lora) {
    int64_t nx = 0;
    # 统计各个部分的参数数量
    nx += ggml_nelements(lora->tok_embeddings_a);
    nx += ggml_nelements(lora->tok_embeddings_b);
    nx += ggml_nelements(lora->norm_a);
    nx += ggml_nelements(lora->norm_b);
    nx += ggml_nelements(lora->output_a);
    nx += ggml_nelements(lora->output_b);
# 遍历 LoRa 网络的每一层
for (uint32_t i = 0; i < lora->layers.size(); ++i) {
    # 获取当前层的引用
    auto & layer = lora->layers[i];
    # 计算并累加 attention_norm_a 的元素数量
    nx += ggml_nelements(layer.attention_norm_a);
    # 计算并累加 attention_norm_b 的元素数量
    nx += ggml_nelements(layer.attention_norm_b);
    # 计算并累加 wq_a 的元素数量
    nx += ggml_nelements(layer.wq_a);
    # 计算并累加 wq_b 的元素数量
    nx += ggml_nelements(layer.wq_b);
    # 计算并累加 wk_a 的元素数量
    nx += ggml_nelements(layer.wk_a);
    # 计算并累加 wk_b 的元素数量
    nx += ggml_nelements(layer.wk_b);
    # 计算并累加 wv_a 的元素数量
    nx += ggml_nelements(layer.wv_a);
    # 计算并累加 wv_b 的元素数量
    nx += ggml_nelements(layer.wv_b);
    # 计算并累加 wo_a 的元素数量
    nx += ggml_nelements(layer.wo_a);
    # 计算并累加 wo_b 的元素数量
    nx += ggml_nelements(layer.wo_b);
    # 计算并累加 ffn_norm_a 的元素数量
    nx += ggml_nelements(layer.ffn_norm_a);
    # 计算并累加 ffn_norm_b 的元素数量
    nx += ggml_nelements(layer.ffn_norm_b);
    # 计算并累加 w1_a 的元素数量
    nx += ggml_nelements(layer.w1_a);
    # 计算并累加 w1_b 的元素数量
    nx += ggml_nelements(layer.w1_b);
    # 计算并累加 w2_a 的元素数量
    nx += ggml_nelements(layer.w2_a);
    # 计算并累加 w2_b 的元素数量
    nx += ggml_nelements(layer.w2_b);
    # 计算并累加 w3_a 的元素数量
    nx += ggml_nelements(layer.w3_a);
}
    // 计算 nx 的值，加上 layer.w3_b 中元素的个数
    nx += ggml_nelements(layer.w3_b);
    // 返回 nx 的值
    }
    return nx;
}

int main(int argc, char ** argv) {
    // 获取默认的训练参数
    struct train_params params = get_default_train_params();

    // 解析命令行参数，如果解析失败则返回 1
    if (!train_params_parse(argc, argv, &params)) {
        return 1;
    }

    // 如果参数中的随机种子为默认值，则设置为当前时间
    if (params.common.seed == LLAMA_DEFAULT_SEED) {
        params.common.seed = time(NULL);
    }
    // 打印随机种子的值
    printf("%s: seed: %u\n", __func__, params.common.seed);
    // 根据随机种子设置随机数生成器的种子
    srand(params.common.seed);

    // 获取默认的 LLAMA 模型参数
    struct llama_model_params llama_mparams = llama_model_default_params();
    // 设置 LLAMA 模型参数中的 GPU 层数
    llama_mparams.n_gpu_layers = params.common.n_gpu_layers;
```

    # 设置 llama_mparams 的 vocab_only 属性为 false
    llama_mparams.vocab_only = false;

    # 打印模型基础信息
    printf("%s: model base = '%s'\n", __func__, params.fn_model_base);
    
    # 从文件加载 llama_model 对象
    struct llama_model * lmodel = llama_load_model_from_file(params.fn_model_base, llama_mparams);

    # 创建 llama_context_params 对象，并设置为默认参数
    struct llama_context_params llama_cparams = llama_context_default_params();
    
    # 使用 lmodel 和 llama_cparams 创建 llama_context 对象
    struct llama_context * lctx = llama_new_context_with_model(lmodel, llama_cparams);

    # 创建 my_llama_model 对象，并初始化
    struct my_llama_model model;
    init_model(lmodel, &model, params.fn_model_base, params.common.n_ctx);

    # 创建 my_llama_lora 对象
    struct my_llama_lora lora;

    # 初始化训练状态对象 train
    struct train_state      * train = init_train_state();
    # 初始化 ggml_opt_context 对象 opt
    struct ggml_opt_context * opt   = train->opt;

    # 从命令行设置参数
    if (params.custom_f_norm_rms_eps) {
        # 如果存在自定义的 f_norm_rms_eps 参数，则设置 model.hparams.f_norm_rms_eps 为该值
        model.hparams.f_norm_rms_eps  = params.f_norm_rms_eps;
    }
# 如果自定义了绳索频率基数，则将模型的绳索频率基数设置为参数中的值
if (params.custom_rope_freq_base) {
    model.hparams.rope_freq_base  = params.rope_freq_base;
}
# 如果自定义了绳索频率比例，则将模型的绳索频率比例设置为参数中的值
if (params.custom_rope_freq_scale) {
    model.hparams.rope_freq_scale = params.rope_freq_scale;
}
# 设置 LORA 模型的参数 lora_r 为参数中的值
lora.hparams.lora_r                = params.lora_r;
# 如果自定义了 LORA 模型的参数 lora_alpha，则将其设置为参数中的值，否则设置为参数中的 lora_r 的值
lora.hparams.lora_alpha            = params.custom_lora_alpha            ? params.lora_alpha            : params.lora_r;
# 设置 n_rank_attention_norm 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 1
uint32_t n_rank_attention_norm     = params.custom_n_rank_attention_norm ? params.n_rank_attention_norm : 1;
# 设置 n_rank_wq 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_wq                 = params.custom_n_rank_wq             ? params.n_rank_wq             : params.lora_r;
# 设置 n_rank_wk 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_wk                 = params.custom_n_rank_wk             ? params.n_rank_wk             : params.lora_r;
# 设置 n_rank_wv 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_wv                 = params.custom_n_rank_wv             ? params.n_rank_wv             : params.lora_r;
# 设置 n_rank_wo 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_wo                 = params.custom_n_rank_wo             ? params.n_rank_wo             : params.lora_r;
# 设置 n_rank_ffn_norm 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 1
uint32_t n_rank_ffn_norm           = params.custom_n_rank_ffn_norm       ? params.n_rank_ffn_norm       : 1;
# 设置 n_rank_w1 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_w1                 = params.custom_n_rank_w1             ? params.n_rank_w1             : params.lora_r;
# 设置 n_rank_w2 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_w2                 = params.custom_n_rank_w2             ? params.n_rank_w2             : params.lora_r;
# 设置 n_rank_w3 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_w3                 = params.custom_n_rank_w3             ? params.n_rank_w3             : params.lora_r;
# 设置 n_rank_tok_embeddings 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_tok_embeddings     = params.custom_n_rank_tok_embeddings ? params.n_rank_tok_embeddings : params.lora_r;
# 设置 n_rank_norm 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 1
uint32_t n_rank_norm               = params.custom_n_rank_norm           ? params.n_rank_norm           : 1;
# 设置 n_rank_output 为参数中的值，如果自定义了则设置为参数中的值，否则设置为 lora_r 的值
uint32_t n_rank_output             = params.custom_n_rank_output         ? params.n_rank_output         : params.lora_r;
    # 设置 Lora 模型的注意力规范化的秩
    lora.hparams.n_rank_attention_norm = n_rank_attention_norm;
    # 设置 Lora 模型的查询权重的秩
    lora.hparams.n_rank_wq = n_rank_wq;
    # 设置 Lora 模型的键权重的秩
    lora.hparams.n_rank_wk = n_rank_wk;
    # 设置 Lora 模型的值权重的秩
    lora.hparams.n_rank_wv = n_rank_wv;
    # 设置 Lora 模型的输出权重的秩
    lora.hparams.n_rank_wo = n_rank_wo;
    # 设置 Lora 模型的前馈神经网络规范化的秩
    lora.hparams.n_rank_ffn_norm = n_rank_ffn_norm;
    # 设置 Lora 模型的第一个全连接层权重的秩
    lora.hparams.n_rank_w1 = n_rank_w1;
    # 设置 Lora 模型的第二个全连接层权重的秩
    lora.hparams.n_rank_w2 = n_rank_w2;
    # 设置 Lora 模型的第三个全连接层权重的秩
    lora.hparams.n_rank_w3 = n_rank_w3;
    # 设置 Lora 模型的 token embeddings 的秩
    lora.hparams.n_rank_tok_embeddings = n_rank_tok_embeddings;
    # 设置 Lora 模型的规范化的秩
    lora.hparams.n_rank_norm = n_rank_norm;
    # 设置 Lora 模型的输出的秩
    lora.hparams.n_rank_output = n_rank_output;

    // 从命令行设置优化器参数
    opt->params = ggml_opt_default_params(GGML_OPT_ADAM);
    // 设置是否打印前向图
    opt->params.print_forward_graph = false;
    // 设置是否打印反向图
    opt->params.print_backward_graph = false;
    // 设置图的大小
    opt->params.graph_size = LLAMA_TRAIN_MAX_NODES;
    // 设置线程数
    opt->params.n_threads = params.common.n_threads;
    // 设置过去的时间
    opt->params.past = params.common.opt_past;
    # 设置opt结构体中的delta参数为params结构体中的opt_delta参数
    opt->params.delta = params.common.opt_delta;
    # 设置opt结构体中的max_no_improvement参数为params结构体中的opt_max_no_improvement参数
    opt->params.max_no_improvement = params.common.opt_max_no_improvement;
    # 设置opt结构体中的n_gradient_accumulation参数为params结构体中的n_gradient_accumulation参数
    opt->params.n_gradient_accumulation = params.common.n_gradient_accumulation;
    # 设置opt结构体中的adam.n_iter参数为params结构体中的adam_n_iter参数
    opt->params.adam.n_iter = params.common.adam_n_iter;
    # 设置opt结构体中的adam.sched参数为1.0f
    opt->params.adam.sched = 1.0f;
    # 设置opt结构体中的adam.alpha参数为params结构体中的adam_alpha参数
    opt->params.adam.alpha = params.common.adam_alpha;
    # 设置opt结构体中的adam.decay参数为params结构体中的adam_decay参数
    opt->params.adam.decay = params.common.adam_decay;
    # 设置opt结构体中的adam.decay_min_ndim参数为params结构体中的adam_decay_min_ndim参数
    opt->params.adam.decay_min_ndim = params.common.adam_decay_min_ndim;
    # 设置opt结构体中的adam.beta1参数为params结构体中的adam_beta1参数
    opt->params.adam.beta1 = params.common.adam_beta1;
    # 设置opt结构体中的adam.beta2参数为params结构体中的adam_beta2参数
    opt->params.adam.beta2 = params.common.adam_beta2;
    # 设置opt结构体中的adam.gclip参数为params结构体中的adam_gclip参数
    opt->params.adam.gclip = params.common.adam_gclip;
    # 设置opt结构体中的adam.eps_f参数为params结构体中的adam_eps_f参数
    opt->params.adam.eps_f = params.common.adam_eps_f;

    # 初始化alloc指针为NULL
    ggml_allocr * alloc = NULL;

    # 打印初始化模型的信息
    printf("%s: init model\n", __func__);
    # 载入checkpoint_lora_file，将结果存储在model和lora中，train参数为训练标志
    bool existed = load_checkpoint_lora_file(params.common.fn_checkpoint_in, &model, &lora, train);

    if (existed) {
        # 覆盖最后的n_ctx参数为用户提供的n_ctx参数
// 如果自定义了上下文长度，将模型的上下文长度设置为自定义的长度
if (params.common.custom_n_ctx) {
    model.hparams.n_ctx = params.common.n_ctx;
}

// 检查是否有优化参数发生了变化
const bool opt_param_count_changed = (
    (lora.hparams.n_rank_attention_norm != n_rank_attention_norm)
    || (lora.hparams.n_rank_wq != n_rank_wq)
    || (lora.hparams.n_rank_wk != n_rank_wk)
    || (lora.hparams.n_rank_wv != n_rank_wv)
    || (lora.hparams.n_rank_wo != n_rank_wo)
    || (lora.hparams.n_rank_ffn_norm != n_rank_ffn_norm)
    || (lora.hparams.n_rank_w1 != n_rank_w1)
    || (lora.hparams.n_rank_w2 != n_rank_w2)
    || (lora.hparams.n_rank_w3 != n_rank_w3)
    || (lora.hparams.n_rank_tok_embeddings != n_rank_tok_embeddings)
    || (lora.hparams.n_rank_norm != n_rank_norm)
    || (lora.hparams.n_rank_output != n_rank_output)
);

// 检查过去是否发生了变化
const bool opt_past_changed = opt->params.past != params.common.opt_past;
// 如果优化参数数量发生了变化
if (opt_param_count_changed) {
    // 打印当前的 Lora 参数，并且输出错误信息，终止程序
    print_lora_params(&lora.hparams);
    die("Provided rank differs from checkpoint file. To use different rank start finetune from scratch with empty input checkpoint, e.g --checkpoint-in ''. Aborting.");
    // 需要丢弃先前的优化器梯度统计，并且使用新的形状进行 opt_init
    // TODO
}
// 如果优化器过去的值发生了变化
if (opt_past_changed) {
    // 输出错误信息，终止程序
    die("Optimizer parameter '--opt-past N' differs from checkpoint file. To use different value finetune from scratch with empty input checkpoint, e.g --checkpoint-in ''. Aborting");
    // 需要丢弃先前的优化器过去的函数值统计，并且使用新的形状进行 opt_init
    // TODO
} else { // existed == false
    // 初始化 Lora 模型
    init_lora(&model, &lora);
    // 随机初始化 Lora 模型
    randomize_lora(&lora, params.common.seed, 0.0f, 1.0f, -1.0f, +1.0f);
    // 如果不仅仅是写入 Lora 模型
    if (!params.only_write_lora) {
        // 初始化优化器
        ggml_opt_init(opt->ctx, opt, opt->params, get_parameter_count(&lora));
    }
}
// 设置迭代次数
opt->iter = train->train_its;
    // 打印模型参数
    print_params(&model.hparams);
    // 打印 LoRa 参数
    print_lora_params(&lora.hparams);
    // 打印总的训练迭代次数
    printf("%s: total train_iterations %llu\n", __func__, (long long unsigned) train->train_its);
    // 打印已观察到的训练样本数
    printf("%s: seen train_samples     %llu\n", __func__, (long long unsigned) train->train_samples);
    // 打印已观察到的训练标记数
    printf("%s: seen train_tokens      %llu\n", __func__, (long long unsigned) train->train_tokens);
    // 打印已完成的训练周期数
    printf("%s: completed train_epochs %llu\n", __func__, (long long unsigned) train->train_epochs);
    // 打印 LoRa 大小
    printf("%s: lora_size = %zu bytes (%.1f MB)\n", __func__, (ggml_used_mem(lora.ctx) + lora.data.size()), (float) (ggml_used_mem(lora.ctx) + lora.data.size()) / (1024.0f*1024.0f));

    // 如果只需要写入 LoRa 数据
    if (params.only_write_lora) {
        // 创建保存训练文件数据的结构
        save_train_files_data save_data;
        // 设置检查点输出文件名为空
        save_data.fn_checkpoint_out = "";
        // 设置 LoRa 输出文件名
        save_data.fn_lora_out       = params.fn_lora_out;
        // 设置文件名模式
        save_data.pattern_fn_it     = params.common.pattern_fn_it;
        // 设置最新文件名
        save_data.fn_latest         = params.common.fn_latest;
        // 设置模型
        save_data.model             = &model;
        // 设置 LoRa
        save_data.lora              = &lora;

        // 保存训练文件
        save_train_files(&save_data, train);
    // 释放训练状态
    free_train_state(train);
    // 释放 LoRa 上下文
    ggml_free(lora.ctx);
    // 释放 Llama 上下文
    llama_free(lctx);
    // 释放 Llama 模型
    llama_free_model(lmodel);
    // 返回 0，表示成功
    return 0;
}

// 打印优化器内存大小
printf("%s: opt_size  = %zu bytes (%.1f MB)\n", __func__, ggml_get_mem_size(opt->ctx), (float) ggml_get_mem_size(opt->ctx) / (1024.0f*1024.0f));
// 打印优化器迭代次数
printf("%s: opt iter %d\n", __func__, opt->iter);

// 初始化变量
int n_tokens = model.hparams.n_ctx;
int n_vocab  = model.hparams.n_vocab;
int n_batch  = params.common.n_batch;

// 创建存储输入数据的向量
std::vector<uint8_t> mem_input_data;
// 创建存储计算数据的向量
std::vector<uint8_t> mem_compute_data;

// 初始化输入张量的上下文参数，不包括数据
struct ggml_init_params ctx_input_params = {
    // 计算 ggml_tensor_overhead() 函数返回值的两倍，作为内存大小
    ggml_tensor_overhead() * 2, // mem_size
    // 内存缓冲区为空
    NULL,                       // mem_buffer
    // 不需要分配内存
    true,                       // no_alloc
};
// 初始化输入上下文
struct ggml_context * ctx_input = ggml_init(ctx_input_params);

// 创建输入张量
struct ggml_tensor * tokens_input  = ggml_new_tensor_2d(ctx_input, GGML_TYPE_I32, n_tokens, n_batch);
struct ggml_tensor * target_probs  = ggml_new_tensor_3d(ctx_input, GGML_TYPE_F32, n_vocab,  n_tokens, n_batch);

// 测量输入张量所需的内存
size_t max_input_size = GGML_PAD(ggml_nbytes(tokens_input), tensor_alignment) +
                        GGML_PAD(ggml_nbytes(target_probs), tensor_alignment) +
                        tensor_alignment;
// 打印输入张量所需的内存大小
printf("%s: input_size = %zu bytes (%.1f MB)\n", __func__, max_input_size, (float) max_input_size / (1024.0f*1024.0f));

// 分配输入张量所需的内存
mem_input_data.resize(max_input_size);
alloc = ggml_allocr_new(mem_input_data.data(), mem_input_data.size(), tensor_alignment);
ggml_allocr_alloc(alloc, tokens_input);
    // 调用 ggml_allocr_alloc 函数分配内存空间，并使用 target_probs 参数
    ggml_allocr_alloc(alloc, target_probs);
    // 调用 ggml_allocr_free 函数释放之前分配的内存空间
    ggml_allocr_free(alloc);

    // 计算不包含数据的张量的上估计计算大小
    const size_t estimated_compute_size_wo_data = (
            2*LLAMA_TRAIN_MAX_NODES*ggml_tensor_overhead() +
            (params.common.use_checkpointing ? 3 : 2)*(GGML_OBJECT_SIZE+ggml_graph_overhead_custom(LLAMA_TRAIN_MAX_NODES, true))
    );
    // 创建一个 ggml_init_params 结构体，用于计算张量而不分配内存
    struct ggml_init_params ctx_compute_params = {
        estimated_compute_size_wo_data, // mem_size
        NULL,                           // mem_buffer
        true,                           // no_alloc
    };
    // 初始化一个 ggml_context 结构体指针
    struct ggml_context * ctx_compute = NULL;

    // 初始化 loss 和 logits 为 NULL
    struct ggml_tensor * loss   = NULL;
    struct ggml_tensor * logits = NULL;

    // 初始化 gf 和 gb 为 NULL
    struct ggml_cgraph * gf     = NULL;
    struct ggml_cgraph * gb     = NULL;
    // 创建一个指向 ggml_cgraph 结构的指针，并初始化为 NULL
    struct ggml_cgraph * gb_tmp = NULL;

    // 用于存储计算张量所需的内存大小
    size_t best_compute_size = SIZE_MAX;
    // 用于存储最佳评估顺序
    enum ggml_cgraph_eval_order best_order = GGML_CGRAPH_EVAL_ORDER_COUNT;
    // 寻找最佳评估顺序
    for (unsigned order = 0; order < (unsigned) GGML_CGRAPH_EVAL_ORDER_COUNT; ++order) {
        // 初始化计算上下文
        ctx_compute = ggml_init(ctx_compute_params);
        // 创建一个用于测量内存分配的对象
        alloc = ggml_allocr_new_measure(tensor_alignment);
        // 创建一个自定义的图形对象
        gf = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
        // 设置图形对象的评估顺序
        gf->order = (enum ggml_cgraph_eval_order) order;
        // 创建一个自定义的图形对象
        gb = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
        // 根据参数判断是否需要创建一个临时图形对象
        gb_tmp = params.common.use_checkpointing
            ? ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true)
            : NULL;
        // 构建 LORA 微调图形
        loss = llama_build_lora_finetune_graphs(
            &model, &lora, alloc, ctx_compute,
            gf, gb, gb_tmp,
            &logits, tokens_input, target_probs,
            n_tokens, n_batch,
    // 检查是否使用闪存和检查点
    params.common.use_flash,
    params.common.use_checkpointing
);
// 计算最大计算大小
size_t max_compute_size = ggml_allocr_max_size(alloc) + tensor_alignment;
// 如果最大计算大小小于最佳计算大小，则更新最佳计算大小和最佳顺序
if (max_compute_size < best_compute_size) {
    best_compute_size = max_compute_size;
    best_order = gf->order;
}
// 释放分配的内存
ggml_allocr_free(alloc);
ggml_free(ctx_compute);
// 设置最大计算大小为最佳计算大小
size_t max_compute_size = best_compute_size;
// 打印最大计算大小
printf("%s: compute_size = %zu bytes (%.1f MB)\n", __func__, max_compute_size, (float) max_compute_size / (1024.0f*1024.0f));
// 打印评估顺序
printf("%s: evaluation order = %s\n", __func__,
    (best_order == GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT) ? "LEFT_TO_RIGHT" :
    (best_order == GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT) ? "RIGHT_TO_LEFT" :
    "invalid");

// 分配计算张量的内存
mem_compute_data.resize(max_compute_size);
    # 初始化计算上下文
    ctx_compute = ggml_init(ctx_compute_params);
    # 创建一个新的内存分配器
    alloc = ggml_allocr_new(mem_compute_data.data(), mem_compute_data.size(), tensor_alignment);
    # 创建一个新的图形对象
    gf = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
    # 设置图形对象的顺序
    gf->order = best_order;
    # 创建一个新的图形对象
    gb = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
    # 根据参数决定是否创建一个新的图形对象
    gb_tmp = params.common.use_checkpointing
        ? ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true)
        : NULL;
    # 构建 LORA 微调图形
    loss = llama_build_lora_finetune_graphs(
        &model, &lora, alloc, ctx_compute,
        gf, gb, gb_tmp,
        &logits, tokens_input, target_probs,
        n_tokens, n_batch,
        params.common.use_flash,
        params.common.use_checkpointing
    );
    # 释放内存分配器
    ggml_allocr_free(alloc);

    // 对数据进行标记化
    std::vector<llama_token> train_tokens;
    // 创建存储训练样本起始位置的向量
    std::vector<size_t> train_samples_begin;
    // 创建存储训练样本大小的向量
    std::vector<size_t> train_samples_size;
    // 打印信息，标识正在对训练数据进行分词
    printf("%s: tokenize training data\n", __func__);
    // 对训练数据进行分词，并将结果存储在train_tokens中
    tokenize_file(lctx,
            params.common.fn_train_data,
            params.common.sample_start,
            params.common.include_sample_start,
            params.common.overlapping_samples,
            n_tokens,
            train_tokens,
            train_samples_begin,
            train_samples_size);
    // 断言训练样本起始位置和大小的向量大小相等
    GGML_ASSERT(train_samples_begin.size() == train_samples_size.size());

    // 打印信息，标识训练数据的标记数量
    printf("%s: number of training tokens: %zu\n", __func__, train_tokens.size());

    // 创建存储标记出现次数的向量，并初始化为0
    std::vector<size_t> token_noccurs;
    token_noccurs.resize(model.hparams.n_vocab, 0);
    // 遍历训练标记，统计每个标记出现的次数
    for (unsigned int i = 0; i < train_tokens.size(); ++i) {
        ++token_noccurs[train_tokens[i]];
    }
    // 初始化变量，记录唯一标记的数量
    int n_unique_tokens = 0;
    // 遍历标记出现次数的数组，统计非零元素的数量
    for (unsigned int i = 0; i < token_noccurs.size(); ++i) {
        if (token_noccurs[i] == 0) continue;  // 如果标记出现次数为0，则跳过
        ++n_unique_tokens;  // 统计非零元素的数量
    }
    // 打印唯一标记的数量
    printf("%s: number of unique tokens: %d\n", __func__, n_unique_tokens);

    // 计算训练数据的哈希值
    size_t shuffle_samples_hash = compute_samples_hash(params.common.fn_train_data, train_samples_begin.data(), train_samples_size.data(), train_samples_size.size());
    // 检查训练数据是否发生变化
    const bool changed_train_data = (shuffle_samples_hash != train->shuffle_samples_hash) || (train->shuffle_sample_count != train_samples_size.size());
    if (changed_train_data) {
        printf("%s: train data seems to have changed. restarting shuffled epoch.\n", __func__);
    }
    // 检查是否强制重新洗牌数据
    if (params.common.force_reshuffle) {
        printf("%s: forced reshuffling of data. restarting with newly shuffled epoch.\n", __func__);
    }
    // 如果当前的随机数状态为空，或者训练数据发生变化，或者强制重新洗牌数据
    if ((train->shuffle_rng_state_current == "") || changed_train_data || params.common.force_reshuffle) {
        // 重新设置随机数状态、洗牌样本数量和下一个样本的索引
        train->shuffle_rng_state_current = mt19937_seed_to_state(params.common.seed);
        train->shuffle_sample_count = train_samples_size.size();
        train->shuffle_next_sample = 0;
    // 将训练数据的打乱顺序的哈希值赋给train对象的shuffle_samples_hash属性
    train->shuffle_samples_hash = shuffle_samples_hash;
    // 初始化存储打乱后样本偏移量的向量
    std::vector<size_t> train_shuffled_samples_offs;
    // 初始化存储打乱后样本起始位置的向量
    std::vector<size_t> train_shuffled_samples_begin;
    // 初始化存储打乱后样本大小的向量
    std::vector<size_t> train_shuffled_samples_size;
    // 调整存储打乱后样本偏移量的向量的大小
    train_shuffled_samples_offs.resize(train_samples_begin.size());
    // 调整存储打乱后样本起始位置的向量的大小
    train_shuffled_samples_begin.resize(train_samples_begin.size());
    // 调整存储打乱后样本大小的向量的大小
    train_shuffled_samples_size.resize(train_samples_size.size());
    // 使用shuffle_samples函数对训练数据进行打乱，并将下一次的随机数状态赋给train对象的shuffle_rng_state_next属性
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
    // 初始化保存训练文件数据的结构体
    save_train_files_data save_data;
# 将参数中的 fn_checkpoint_out 赋值给 save_data 的 fn_checkpoint_out
save_data.fn_checkpoint_out = params.common.fn_checkpoint_out;
# 将参数中的 fn_lora_out 赋值给 save_data 的 fn_lora_out
save_data.fn_lora_out       = params.fn_lora_out;
# 将参数中的 pattern_fn_it 赋值给 save_data 的 pattern_fn_it
save_data.pattern_fn_it     = params.common.pattern_fn_it;
# 将参数中的 fn_latest 赋值给 save_data 的 fn_latest
save_data.fn_latest         = params.common.fn_latest;
# 将 model 的地址赋值给 save_data 的 model
save_data.model             = &model;
# 将 lora 的地址赋值给 save_data 的 lora
save_data.lora              = &lora;

# 创建一个结构体 train_opt_callback_data 的实例 opt_cb_data
struct train_opt_callback_data opt_cb_data;
# 将参数中的 common 赋值给 opt_cb_data 的 params
opt_cb_data.params                 = &params.common;
# 将 train 赋值给 opt_cb_data 的 train
opt_cb_data.train                  = train;
# 将 save_train_files 的地址赋值给 opt_cb_data 的 save_cb
opt_cb_data.save_cb                = &save_train_files;
# 将 save_data 的地址赋值给 opt_cb_data 的 save_data
opt_cb_data.save_data              = &save_data;
# 将 lctx 赋值给 opt_cb_data 的 lctx
opt_cb_data.lctx                   = lctx;
# 将 opt 的 iter 赋值给 opt_cb_data 的 last_save_iter
opt_cb_data.last_save_iter         = opt->iter;
# 将 train_tokens 的数据赋值给 opt_cb_data 的 tokens_data
opt_cb_data.tokens_data            = train_tokens.data();
# 将 train_tokens 的大小赋值给 opt_cb_data 的 tokens_size
opt_cb_data.tokens_size            = train_tokens.size();
# 将 train_samples_begin 的数据赋值给 opt_cb_data 的 samples_begin
opt_cb_data.samples_begin          = train_samples_begin.data();
# 将 train_samples_size 的数据赋值给 opt_cb_data 的 samples_size
opt_cb_data.samples_size           = train_samples_size.data();
# 将 train_shuffled_samples_offs 的数据赋值给 opt_cb_data 的 shuffled_samples_offs
opt_cb_data.shuffled_samples_offs  = train_shuffled_samples_offs.data();
# 将 train_shuffled_samples_begin 的数据赋值给 opt_cb_data 的 shuffled_samples_begin
opt_cb_data.shuffled_samples_begin = train_shuffled_samples_begin.data();
    // 设置 opt_cb_data 结构体的 shuffled_samples_size 字段为 train_shuffled_samples_size 的数据
    opt_cb_data.shuffled_samples_size  = train_shuffled_samples_size.data();
    // 设置 opt_cb_data 结构体的 samples_count 字段为 train_samples_size 的大小
    opt_cb_data.samples_count          = train_samples_size.size();
    // 设置 opt_cb_data 结构体的 tokens_input 字段为 tokens_input
    opt_cb_data.tokens_input           = tokens_input;
    // 设置 opt_cb_data 结构体的 target_probs 字段为 target_probs
    opt_cb_data.target_probs           = target_probs;
    // 设置 opt_cb_data 结构体的 first_iter 字段为 opt 的 iter 字段
    opt_cb_data.first_iter             = opt->iter;
    // 设置 opt_cb_data 结构体的 first_epoch 字段为 train 的 train_epochs 字段
    opt_cb_data.first_epoch            = train->train_epochs;
    // 设置 opt_cb_data 结构体的 iter_at_last_epoch 字段为 -1
    opt_cb_data.iter_at_last_epoch     = -1;
    // 设置 opt_cb_data 结构体的 last_time 字段为当前时间的毫秒数
    opt_cb_data.last_time              = ggml_time_ms();
    // 设置 opt_cb_data 结构体的 millis_per_iter 字段为 0.0
    opt_cb_data.millis_per_iter        = 0.0;

    // 计算工作缓冲区所需的内存大小
    size_t max_work_size = ggml_graph_plan(gb, params.common.n_threads).work_size + GGML_OBJECT_SIZE;
    // 打印工作缓冲区的大小
    printf("%s: work_size = %zu bytes (%.1f MB)\n", __func__, max_work_size, (float) max_work_size / (1024.0f*1024.0f));

    // 初始化工作缓冲区的上下文
    struct ggml_init_params ctx_work_params = {
        max_work_size, // mem_size
        NULL,          // mem_buffer
        false,         // no_alloc
    };
    // 初始化 ggml 上下文，使用给定的参数
    struct ggml_context * ctx_work = ggml_init(ctx_work_params);

    // 获取当前时间，单位为毫秒
    int64_t t0 = ggml_time_ms();

    // 恢复训练过程，传入参数和回调函数
    ggml_opt_resume_g(ctx_work, opt, loss, gf, gb, &train_opt_callback, (void *) &opt_cb_data);

    // 释放内存，释放工作上下文
    ggml_free(ctx_work);
    ggml_free(ctx_compute);
    ggml_free(ctx_input);

    // 获取当前时间，单位为毫秒
    int64_t t1 = ggml_time_ms();
    // 打印函数名和总训练时间
    printf("%s: total training time: ", __func__);
    // 打印时间间隔
    print_duration((double) (t1 - t0));
    printf("\n");

    // 计算新的迭代次数
    int new_iters = opt->iter - opt_cb_data.last_save_iter;
    // 如果新的迭代次数大于0
    if (new_iters > 0) {
        // 更新训练次数和训练 token 数
        train->train_its     += new_iters;
        train->train_tokens  += new_iters * opt->params.n_gradient_accumulation * n_batch * n_tokens;
# 保存训练文件并更新最后保存的迭代次数
save_train_files(&save_data, train);
opt_cb_data.last_save_iter = opt->iter;
}

# 释放优化器上下文
ggml_free(opt->ctx);
# 释放训练状态
free_train_state(train);
# 释放 LORA 上下文
ggml_free(lora.ctx);
# 释放 LLAMA 上下文
llama_free(lctx);
# 释放 LLAMA 模型
llama_free_model(lmodel);
# 返回 0 表示成功
return 0;
}
```