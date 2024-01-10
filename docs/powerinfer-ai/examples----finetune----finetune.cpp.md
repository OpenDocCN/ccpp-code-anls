# `PowerInfer\examples\finetune\finetune.cpp`

```
#include "ggml.h"  // 引入 ggml 库的头文件
#include "ggml-alloc.h"  // 引入 ggml-alloc 库的头文件
#include "llama.h"  // 引入 llama 库的头文件
#include "common.h"  // 引入 common 库的头文件
#include "train.h"  // 引入 train 库的头文件
#include <unordered_map>  // 引入无序映射的头文件
#include <vector>  // 引入向量的头文件
#include <cassert>  // 引入断言的头文件
#include <climits>  // 引入整数范围的头文件
#include <cstring>  // 引入字符串操作的头文件
#include <cstdarg>  // 引入可变参数的头文件
#include <ctime>  // 引入时间操作的头文件
#include <random>  // 引入随机数生成的头文件
#include <stdexcept>  // 引入异常处理的头文件
#include <algorithm>  // 引入算法的头文件
#include <string>  // 引入字符串操作的头文件

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

static const size_t tensor_alignment = 32;  // 定义静态常量 tensor_alignment 为 32

struct my_llama_hparams {  // 定义结构体 my_llama_hparams
    uint32_t n_vocab    = 32000;  // 初始化成员变量 n_vocab 为 32000
    uint32_t n_ctx      = 512;  // 初始化成员变量 n_ctx 为 512
    uint32_t n_embd     = 4096;  // 初始化成员变量 n_embd 为 4096
    uint32_t n_ff       = 11008;  // 初始化成员变量 n_ff 为 11008
    uint32_t n_head     = 32;  // 初始化成员变量 n_head 为 32
    uint32_t n_head_kv  = 32;  // 初始化成员变量 n_head_kv 为 32
    uint32_t n_layer    = 32;  // 初始化成员变量 n_layer 为 32

    // float f_norm_eps     = 1e-5f; // falcon
    float f_norm_rms_eps = 1e-5f; // llama  // 初始化成员变量 f_norm_rms_eps 为 1e-5f

    float rope_freq_base  = 10000.0f;  // 初始化成员变量 rope_freq_base 为 10000.0f
    float rope_freq_scale = 1.0f;  // 初始化成员变量 rope_freq_scale 为 1.0f

    uint32_t n_gqa() const {  // 定义成员函数 n_gqa
        return n_head/n_head_kv;  // 返回 n_head 除以 n_head_kv 的结果
    }

    uint32_t n_embd_head() const {  // 定义成员函数 n_embd_head
        return n_embd/n_head;  // 返回 n_embd 除以 n_head 的结果
    }

    uint32_t n_embd_gqa() const {  // 定义成员函数 n_embd_gqa
        return n_embd/n_gqa();  // 返回 n_embd 除以 n_gqa() 函数的结果
    }

    bool operator!=(const my_llama_hparams& other) const {  // 定义重载运算符 !=
        return memcmp(this, &other, sizeof(other));  // 返回 this 和 other 的内存比较结果
    }
};

struct my_llama_layer {  // 定义结构体 my_llama_layer
    // normalization
    struct ggml_tensor * attention_norm;  // 定义指向 ggml_tensor 结构体的指针 attention_norm

    // attention
    struct ggml_tensor * wq;  // 定义指向 ggml_tensor 结构体的指针 wq
    struct ggml_tensor * wk;  // 定义指向 ggml_tensor 结构体的指针 wk
    struct ggml_tensor * wv;  // 定义指向 ggml_tensor 结构体的指针 wv
    struct ggml_tensor * wo;  // 定义指向 ggml_tensor 结构体的指针 wo

    // normalization
    struct ggml_tensor * ffn_norm;  // 定义指向 ggml_tensor 结构体的指针 ffn_norm

    // ff
    struct ggml_tensor * w1;  // 定义指向 ggml_tensor 结构体的指针 w1
    struct ggml_tensor * w2;  // 定义指向 ggml_tensor 结构体的指针 w2
    struct ggml_tensor * w3;  // 定义指向 ggml_tensor 结构体的指针 w3
};

struct my_llama_model {  // 定义结构体 my_llama_model
    struct my_llama_hparams hparams;  // 定义 my_llama_hparams 类型的成员变量 hparams

    struct ggml_tensor * tok_embeddings;  // 定义指向 ggml_tensor 结构体的指针 tok_embeddings

    struct ggml_tensor * norm;  // 定义指向 ggml_tensor 结构体的指针 norm
    struct ggml_tensor * output;  // 定义指向 ggml_tensor 结构体的指针 output

    std::vector<my_llama_layer> layers;  // 定义 my_llama_layer 类型的向量 layers
};

struct my_llama_lora_hparams {  // 定义结构体 my_llama_lora_hparams
    uint32_t lora_r = 1;  // 初始化成员变量 lora_r 为 1
    uint32_t lora_alpha = 1;  // 初始化成员变量 lora_alpha 为 1
    uint32_t n_rank_attention_norm = 1;  // 初始化成员变量 n_rank_attention_norm 为 1
    uint32_t n_rank_wq = 4;  // 初始化成员变量 n_rank_wq 为 4
    uint32_t n_rank_wk = 4;  // 初始化成员变量 n_rank_wk 为 4
    uint32_t n_rank_wv = 4;  // 初始化成员变量 n_rank_wv 为 4
    # 定义并初始化多个 uint32_t 类型的变量，表示不同的排名
    uint32_t n_rank_wo = 4;
    uint32_t n_rank_ffn_norm = 1;
    uint32_t n_rank_w1 = 4;
    uint32_t n_rank_w2 = 4;
    uint32_t n_rank_w3 = 4;
    uint32_t n_rank_tok_embeddings = 4;
    uint32_t n_rank_norm = 1;
    uint32_t n_rank_output = 4;
    
    # 定义重载运算符"!="，用于比较两个 my_llama_lora_hparams 对象是否不相等
    bool operator!=(const my_llama_lora_hparams& other) const {
        # 使用 memcmp 函数比较当前对象和另一个对象的内存内容是否相等，返回比较结果
        return memcmp(this, &other, sizeof(other));
    }
// 定义结构体 my_llama_lora_layer，包含了多个指向 ggml_tensor 结构体的指针
struct my_llama_lora_layer {
    // normalization
    struct ggml_tensor * attention_norm_a;
    struct ggml_tensor * attention_norm_b;

    // attention
    struct ggml_tensor * wq_a;
    struct ggml_tensor * wq_b;
    struct ggml_tensor * wk_a;
    struct ggml_tensor * wk_b;
    struct ggml_tensor * wv_a;
    struct ggml_tensor * wv_b;
    struct ggml_tensor * wo_a;
    struct ggml_tensor * wo_b;

    // normalization
    struct ggml_tensor * ffn_norm_a;
    struct ggml_tensor * ffn_norm_b;

    // ff
    struct ggml_tensor * w1_a;
    struct ggml_tensor * w1_b;
    struct ggml_tensor * w2_a;
    struct ggml_tensor * w2_b;
    struct ggml_tensor * w3_a;
    struct ggml_tensor * w3_b;
};

// 定义结构体 my_llama_lora，包含了指向 ggml_context 结构体的指针、std::vector<uint8_t> 类型的数据、my_llama_lora_hparams 类型的变量、以及多个指向 ggml_tensor 结构体的指针和一个存储 my_llama_lora_layer 结构体的 vector
struct my_llama_lora {
    struct ggml_context * ctx = NULL;
    std::vector<uint8_t> data;
    my_llama_lora_hparams hparams;
    struct ggml_tensor * tok_embeddings_a;
    struct ggml_tensor * tok_embeddings_b;
    struct ggml_tensor * norm_a;
    struct ggml_tensor * norm_b;
    struct ggml_tensor * output_a;
    struct ggml_tensor * output_b;
    std::vector<my_llama_lora_layer> layers;
};

// 定义 gguf 常量
static const char * LLM_KV_TRAINING_TYPE_FINETUNE_LORA   = "finetune_lora";
static const char * LLM_KV_TRAINING_TYPE                 = "training.type";
static const char * LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD  = "training.lora.rank.token_embd";
static const char * LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM = "training.lora.rank.output_norm";
static const char * LLM_KV_TRAINING_LORA_RANK_OUTPUT      = "training.lora.rank.output";
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_NORM   = "training.lora.rank.attn_norm";
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_Q      = "training.lora.rank.attn_q";
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_K      = "training.lora.rank.attn_k";
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_V      = "training.lora.rank.attn_v";
// 定义常量字符串，表示训练中的 LoRa 模型的不同部分
static const char * LLM_KV_TRAINING_LORA_RANK_ATTN_OUT    = "training.lora.rank.attn_output";
static const char * LLM_KV_TRAINING_LORA_RANK_FFN_NORM    = "training.lora.rank.ffn_norm";
static const char * LLM_KV_TRAINING_LORA_RANK_FFN_GATE    = "training.lora.rank.ffn_gate";
static const char * LLM_KV_TRAINING_LORA_RANK_FFN_DOWN    = "training.lora.rank.ffn_down";
static const char * LLM_KV_TRAINING_LORA_RANK_FFN_UP      = "training.lora.rank.ffn_up";

// gguf 常量（与 gguf.py 同步）

// 表示通用架构的键值
static const char * LLM_KV_GENERAL_ARCHITECTURE        = "general.architecture";
// 表示通用文件类型的键值
static const char * LLM_KV_GENERAL_FILE_TYPE           = "general.file_type";

// 下面的键值包含占位符 %s，表示需要根据特定的字符串进行格式化
// 表示上下文长度的键值
static const char * LLM_KV_CONTEXT_LENGTH              = "%s.context_length";
// 表示嵌入长度的键值
static const char * LLM_KV_EMBEDDING_LENGTH            = "%s.embedding_length";
// 表示块数量的键值
static const char * LLM_KV_BLOCK_COUNT                 = "%s.block_count";
// 表示前馈长度的键值
static const char * LLM_KV_FEED_FORWARD_LENGTH         = "%s.feed_forward_length";
// 表示注意力头数量的键值
static const char * LLM_KV_ATTENTION_HEAD_COUNT        = "%s.attention.head_count";
// 表示注意力头数量（KV）的键值
static const char * LLM_KV_ATTENTION_HEAD_COUNT_KV     = "%s.attention.head_count_kv";
// 表示注意力层归一化 RMS 误差的键值
static const char * LLM_KV_ATTENTION_LAYERNORM_RMS_EPS = "%s.attention.layer_norm_rms_epsilon";
// 表示 ROPE 维度数量的键值
static const char * LLM_KV_ROPE_DIMENSION_COUNT        = "%s.rope.dimension_count";
// 表示 ROPE 频率基数的键值（TODO 在 llama.cpp 中加载）
static const char * LLM_KV_ROPE_FREQ_BASE              = "%s.rope.freq_base";
// 表示 ROPE 线性缩放的键值
static const char * LLM_KV_ROPE_SCALE_LINEAR           = "%s.rope.scale_linear";

// 表示 token 嵌入的张量名称
static const char * LLM_TENSOR_TOKEN_EMBD    = "token_embd";
// 表示输出归一化的张量名称
static const char * LLM_TENSOR_OUTPUT_NORM   = "output_norm";
// 表示输出的张量名称
static const char * LLM_TENSOR_OUTPUT        = "output";
// 表示注意力归一化的张量名称
static const char * LLM_TENSOR_ATTN_NORM     = "blk.%d.attn_norm";
// 表示注意力 Q 的张量名称
static const char * LLM_TENSOR_ATTN_Q        = "blk.%d.attn_q";
// 表示注意力 K 的张量名称
static const char * LLM_TENSOR_ATTN_K        = "blk.%d.attn_k";
// 表示注意力 V 的张量名称
static const char * LLM_TENSOR_ATTN_V        = "blk.%d.attn_v";
// 定义常量指针，表示注意力输出的张量名称
static const char * LLM_TENSOR_ATTN_OUT      = "blk.%d.attn_output";
// 定义常量指针，表示 FFN 层归一化的张量名称
static const char * LLM_TENSOR_FFN_NORM      = "blk.%d.ffn_norm";
// 定义常量指针，表示 FFN 层门控的张量名称
static const char * LLM_TENSOR_FFN_GATE      = "blk.%d.ffn_gate";
// 定义常量指针，表示 FFN 层下采样的张量名称
static const char * LLM_TENSOR_FFN_DOWN      = "blk.%d.ffn_down";
// 定义常量指针，表示 FFN 层上采样的张量名称
static const char * LLM_TENSOR_FFN_UP        = "blk.%d.ffn_up";

// 打印模型参数的函数
static void print_params(struct my_llama_hparams * params) {
    // 打印词汇量大小
    printf("%s: n_vocab:   %u\n", __func__, params->n_vocab);
    // 打印上下文大小
    printf("%s: n_ctx:     %u\n", __func__, params->n_ctx);
    // 打印嵌入维度
    printf("%s: n_embd:    %u\n", __func__, params->n_embd);
    // 打印 FFN 层大小
    printf("%s: n_ff:      %u\n", __func__, params->n_ff);
    // 打印注意力头数
    printf("%s: n_head:    %u\n", __func__, params->n_head);
    // 打印注意力头数（键值）
    printf("%s: n_head_kv: %u\n", __func__, params->n_head_kv);
    // 打印层数
    printf("%s: n_layer:   %u\n", __func__, params->n_layer);
    // 打印归一化的 RMS 误差
    printf("%s: norm_rms_eps          : %f\n", __func__, params->f_norm_rms_eps);
    // 打印 ROPE 频率基数
    printf("%s: rope_freq_base        : %f\n", __func__, params->rope_freq_base);
    // 打印 ROPE 频率缩放因子
    printf("%s: rope_freq_scale       : %f\n", __func__, params->rope_freq_scale);
}

// 打印 LORA 模型参数的函数
static void print_lora_params(struct my_llama_lora_hparams * params) {
    // 打印注意力归一化的秩
    printf("%s: n_rank_attention_norm : %u\n", __func__, params->n_rank_attention_norm);
    // 打印查询权重的秩
    printf("%s: n_rank_wq             : %u\n", __func__, params->n_rank_wq);
    // 打印键权重的秩
    printf("%s: n_rank_wk             : %u\n", __func__, params->n_rank_wk);
    // 打印值权重的秩
    printf("%s: n_rank_wv             : %u\n", __func__, params->n_rank_wv);
    // 打印输出权重的秩
    printf("%s: n_rank_wo             : %u\n", __func__, params->n_rank_wo);
    // 打印 FFN 层归一化的秩
    printf("%s: n_rank_ffn_norm       : %u\n", __func__, params->n_rank_ffn_norm);
    // 打印第一层权重的秩
    printf("%s: n_rank_w1             : %u\n", __func__, params->n_rank_w1);
    // 打印第二层权重的秩
    printf("%s: n_rank_w2             : %u\n", __func__, params->n_rank_w2);
    // 打印第三层权重的秩
    printf("%s: n_rank_w3             : %u\n", __func__, params->n_rank_w3);
    // 打印词嵌入的秩
    printf("%s: n_rank_tok_embeddings : %u\n", __func__, params->n_rank_tok_embeddings);
}
    # 打印函数名和参数中的 n_rank_norm 值
    printf("%s: n_rank_norm           : %u\n", __func__, params->n_rank_norm);
    # 打印函数名和参数中的 n_rank_output 值
    printf("%s: n_rank_output         : %u\n", __func__, params->n_rank_output);
// 定义宏 GGUF_GET_KEY，用于获取指定类型的键值对，并进行类型检查和错误处理
#define GGUF_GET_KEY(ctx, dst, func, type, req, key) \
{ \
    // 将键名转换为字符串类型
    const std::string skey(key); \
    // 在上下文中查找键的索引
    const int kid = gguf_find_key(ctx, skey.c_str()); \
    // 如果找到了键
    if (kid >= 0) { \
        // 获取键值的类型
        enum gguf_type ktype = gguf_get_kv_type(ctx, kid); \
        // 如果类型不匹配
        if (ktype != (type)) { \
            // 抛出错误，指出键的类型错误
            die_fmt("key %s has wrong type: %s", skey.c_str(), gguf_type_name(ktype)); \
        } \
        // 调用指定的函数获取键值，并赋给目标变量
        (dst) = func(ctx, kid); \
    } else if (req) { \
        // 如果要求必须找到键，但未找到
        die_fmt("key not found in model: %s", skey.c_str()); \
    } \
}

// 加载模型超参数的函数
static void load_model_hparams_gguf(struct gguf_context * ctx, struct my_llama_hparams * hparams, const char * expected_arch) {
    // 定义存储架构的字符串变量
    std::string arch;

    // 获取架构参数
    GGUF_GET_KEY(ctx, arch, gguf_get_val_str, GGUF_TYPE_STRING, true, LLM_KV_GENERAL_ARCHITECTURE);
    // 如果有期望的架构参数
    if (expected_arch != NULL) {
        // 如果实际架构与期望架构不符
        if (arch != expected_arch) {
            // 打印错误信息
            printf("%s: arch=%s expected_arch=%s\n", __func__, arch.c_str(), expected_arch);
        }
        // 断言实际架构与期望架构相同
        GGML_ASSERT(arch == expected_arch);
    }

    // 定义存储键名的缓冲区
    std::vector<char> keybuf;
    keybuf.resize(512);
    // 定义 lambda 函数，用于根据架构参数生成键名
    auto kv = [&arch, &keybuf](const char * key) -> const char * {
        snprintf(keybuf.data(), keybuf.size(), key, arch.c_str());
        return keybuf.data();
    };

    // 获取并设置模型超参数的各个值
    GGUF_GET_KEY(ctx, hparams->n_embd,         gguf_get_val_u32, GGUF_TYPE_UINT32,  true, kv(LLM_KV_EMBEDDING_LENGTH));
    GGUF_GET_KEY(ctx, hparams->n_ctx,          gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_CONTEXT_LENGTH));
    GGUF_GET_KEY(ctx, hparams->n_ff,           gguf_get_val_u32, GGUF_TYPE_UINT32,  true, kv(LLM_KV_FEED_FORWARD_LENGTH));
    GGUF_GET_KEY(ctx, hparams->n_head,         gguf_get_val_u32, GGUF_TYPE_UINT32,  true, kv(LLM_KV_ATTENTION_HEAD_COUNT));
    GGUF_GET_KEY(ctx, hparams->n_layer,        gguf_get_val_u32, GGUF_TYPE_UINT32,  true, kv(LLM_KV_BLOCK_COUNT));

    // n_head_kv 是可选的，如果未设置，则默认为 n_head
    hparams->n_head_kv = hparams->n_head;
}
    # 从上下文中获取参数值，存入hparams->n_head_kv中
    GGUF_GET_KEY(ctx, hparams->n_head_kv,      gguf_get_val_u32, GGUF_TYPE_UINT32, false, kv(LLM_KV_ATTENTION_HEAD_COUNT_KV));

    # 初始化rope_freq_scale为1.0
    float rope_freq_scale = 1.0f;
    # 从上下文中获取参数值，存入hparams->f_norm_rms_eps中
    GGUF_GET_KEY(ctx, hparams->f_norm_rms_eps, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS));
    # 从上下文中获取参数值，存入hparams->rope_freq_base中
    GGUF_GET_KEY(ctx, hparams->rope_freq_base, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_FREQ_BASE));
    # 从上下文中获取参数值，存入rope_freq_scale中
    GGUF_GET_KEY(ctx, rope_freq_scale, gguf_get_val_f32, GGUF_TYPE_FLOAT32, false, kv(LLM_KV_ROPE_SCALE_LINEAR));
    # 如果rope_freq_scale不等于1.0，则更新hparams->rope_freq_scale的值
    if (rope_freq_scale != 1.0f) {
        hparams->rope_freq_scale = 1.0f / rope_freq_scale;
    }
// 初始化模型结构体，传入输入模型、自定义模型、模型文件名和上下文数
static void init_model(struct llama_model * input, struct my_llama_model * model, const char * fn_model, uint32_t n_ctx) {
    // 获取模型的超参数
    auto & hparams = model->hparams;

    // 初始化临时缓冲区
    std::vector<char> tn_buf;
    tn_buf.resize(GGML_MAX_NAME);
    // 定义获取参数名称的 lambda 函数
    auto tn = [&tn_buf](const char * key) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", key);
        return tn_buf.data();
    };
    // 定义获取参数名称的 lambda 函数
    auto tni = [&tn_buf](const char * key, int bid) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), key, bid);
        std::string s = tn_buf.data();
        snprintf(tn_buf.data(), tn_buf.size(), "%s.weight", s.c_str());
        return tn_buf.data();
    };

    // 从 gguf 文件直接获取参数
    {
        // 初始化 gguf 参数结构体
        struct gguf_init_params params = {
            /*.no_alloc = */ false,
            /*.ctx      = */ NULL,
        };
        // 从文件初始化 gguf 上下文
        struct gguf_context * mctx = gguf_init_from_file(fn_model, params);

        // 从 gguf 上下文中加载模型的超参数
        load_model_hparams_gguf(mctx, &hparams, "llama");

        // 释放 gguf 上下文
        gguf_free(mctx);
    }
    // 设置模型的词汇量和上下文数
    hparams.n_vocab = llama_n_vocab(input);
    hparams.n_ctx = n_ctx;

    // 从 llama_model 中获取张量（可能是内存映射）
    model->tok_embeddings = llama_get_model_tensor(input, tn(LLM_TENSOR_TOKEN_EMBD));
    model->norm           = llama_get_model_tensor(input, tn(LLM_TENSOR_OUTPUT_NORM));
    model->output         = llama_get_model_tensor(input, tn(LLM_TENSOR_OUTPUT));

    // 断言张量的形状
    assert_shape_2d(model->tok_embeddings, hparams.n_embd, hparams.n_vocab);
    assert_shape_1d(model->norm,           hparams.n_embd);
    assert_shape_2d(model->output,         hparams.n_embd, hparams.n_vocab);

    // 调整模型的层数
    model->layers.resize(hparams.n_layer);
}
    # 遍历模型的每一层
    for (uint32_t i = 0; i < hparams.n_layer; ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];

        # 获取当前层的注意力规范化张量
        layer.attention_norm = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_NORM, i));
        # 获取当前层的注意力查询张量
        layer.wq             = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_Q, i));
        # 获取当前层的注意力键张量
        layer.wk             = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_K, i));
        # 获取当前层的注意力值张量
        layer.wv             = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_V, i));
        # 获取当前层的注意力输出张量
        layer.wo             = llama_get_model_tensor(input, tni(LLM_TENSOR_ATTN_OUT, i));
        # 获取当前层的前馈神经网络规范化张量
        layer.ffn_norm       = llama_get_model_tensor(input, tni(LLM_TENSOR_FFN_NORM, i));
        # 获取当前层的前馈神经网络第一层权重张量
        layer.w1             = llama_get_model_tensor(input, tni(LLM_TENSOR_FFN_GATE, i));
        # 获取当前层的前馈神经网络第二层权重张量
        layer.w2             = llama_get_model_tensor(input, tni(LLM_TENSOR_FFN_DOWN, i));
        # 获取当前层的前馈神经网络第三层权重张量
        layer.w3             = llama_get_model_tensor(input, tni(LLM_TENSOR_FFN_UP, i));

        # 断言当前层的注意力规范化张量形状为一维，长度为 hparams.n_embd
        assert_shape_1d(layer.attention_norm, hparams.n_embd);
        # 断言当前层的注意力查询张量形状为二维，大小为 hparams.n_embd x hparams.n_embd
        assert_shape_2d(layer.wq,             hparams.n_embd, hparams.n_embd);
        # 断言当前层的注意力键张量形状为二维，大小为 hparams.n_embd x hparams.n_embd_gqa()
        assert_shape_2d(layer.wk,             hparams.n_embd, hparams.n_embd_gqa());
        # 断言当前层的注意力值张量形状为二维，大小为 hparams.n_embd x hparams.n_embd_gqa()
        assert_shape_2d(layer.wv,             hparams.n_embd, hparams.n_embd_gqa());
        # 断言当前层的注意力输出张量形状为二维，大小为 hparams.n_embd x hparams.n_embd
        assert_shape_2d(layer.wo,             hparams.n_embd, hparams.n_embd);
        # 断言当前层的前馈神经网络规范化张量形状为一维，长度为 hparams.n_embd
        assert_shape_1d(layer.ffn_norm,       hparams.n_embd);
        # 断言当前层的前馈神经网络第一层权重张量形状为二维，大小为 hparams.n_embd x hparams.n_ff
        assert_shape_2d(layer.w1,             hparams.n_embd, hparams.n_ff);
        # 断言当前层的前馈神经网络第二层权重张量形状为二维，大小为 hparams.n_ff x hparams.n_embd
        assert_shape_2d(layer.w2,             hparams.n_ff,   hparams.n_embd);
        # 断言当前层的前馈神经网络第三层权重张量形状为二维，大小为 hparams.n_embd x hparams.n_ff
        assert_shape_2d(layer.w3,             hparams.n_embd, hparams.n_ff);
    }
}
// 设置 LoRa 结构体参数
static void set_param_lora(struct my_llama_lora * lora) {
    // 获取层的数量
    const uint32_t n_layer = lora->layers.size();

    // 获取 LoRa 结构体中的上下文
    struct ggml_context* ctx = lora->ctx;

    // 设置参数
    ggml_set_param(ctx, lora->tok_embeddings_a);
    ggml_set_param(ctx, lora->tok_embeddings_b);
    ggml_set_param(ctx, lora->norm_a);
    ggml_set_param(ctx, lora->norm_b);
    ggml_set_param(ctx, lora->output_a);
    ggml_set_param(ctx, lora->output_b);

    // 遍历每一层，设置参数
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = lora->layers[i];

        ggml_set_param(ctx, layer.attention_norm_a);
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
    }
}

// 分配 LoRa 结构体内存
static void alloc_lora(struct ggml_allocr * alloc, struct my_llama_lora * lora) {
    ggml_allocr_alloc(alloc, lora->tok_embeddings_a);
    ggml_allocr_alloc(alloc, lora->tok_embeddings_b);
    ggml_allocr_alloc(alloc, lora->norm_a);
    ggml_allocr_alloc(alloc, lora->norm_b);
    ggml_allocr_alloc(alloc, lora->output_a);
    ggml_allocr_alloc(alloc, lora->output_b);
    # 遍历 lora 对象的 layers 列表
    for (uint32_t i = 0; i < lora->layers.size(); ++i) {
        # 获取当前索引对应的 layer 对象的引用
        auto & layer = lora->layers[i];
        # 分配内存给 layer 对象的 attention_norm_a 属性
        ggml_allocr_alloc(alloc, layer.attention_norm_a);
        # 分配内存给 layer 对象的 attention_norm_b 属性
        ggml_allocr_alloc(alloc, layer.attention_norm_b);
        # 分配内存给 layer 对象的 wq_a 属性
        ggml_allocr_alloc(alloc, layer.wq_a);
        # 分配内存给 layer 对象的 wq_b 属性
        ggml_allocr_alloc(alloc, layer.wq_b);
        # 分配内存给 layer 对象的 wk_a 属性
        ggml_allocr_alloc(alloc, layer.wk_a);
        # 分配内存给 layer 对象的 wk_b 属性
        ggml_allocr_alloc(alloc, layer.wk_b);
        # 分配内存给 layer 对象的 wv_a 属性
        ggml_allocr_alloc(alloc, layer.wv_a);
        # 分配内存给 layer 对象的 wv_b 属性
        ggml_allocr_alloc(alloc, layer.wv_b);
        # 分配内存给 layer 对象的 wo_a 属性
        ggml_allocr_alloc(alloc, layer.wo_a);
        # 分配内存给 layer 对象的 wo_b 属性
        ggml_allocr_alloc(alloc, layer.wo_b);
        # 分配内存给 layer 对象的 ffn_norm_a 属性
        ggml_allocr_alloc(alloc, layer.ffn_norm_a);
        # 分配内存给 layer 对象的 ffn_norm_b 属性
        ggml_allocr_alloc(alloc, layer.ffn_norm_b);
        # 分配内存给 layer 对象的 w1_a 属性
        ggml_allocr_alloc(alloc, layer.w1_a);
        # 分配内存给 layer 对象的 w1_b 属性
        ggml_allocr_alloc(alloc, layer.w1_b);
        # 分配内存给 layer 对象的 w2_a 属性
        ggml_allocr_alloc(alloc, layer.w2_a);
        # 分配内存给 layer 对象的 w2_b 属性
        ggml_allocr_alloc(alloc, layer.w2_b);
        # 分配内存给 layer 对象的 w3_a 属性
        ggml_allocr_alloc(alloc, layer.w3_a);
        # 分配内存给 layer 对象的 w3_b 属性
        ggml_allocr_alloc(alloc, layer.w3_b);
    }
    # 分配内存给 lora 对象的 tok_embeddings_a 的 grad 属性
    ggml_allocr_alloc(alloc, lora->tok_embeddings_a->grad);
    # 分配内存给 lora 对象的 tok_embeddings_b 的 grad 属性
    ggml_allocr_alloc(alloc, lora->tok_embeddings_b->grad);
    # 分配内存给 lora 对象的 norm_a 的 grad 属性
    ggml_allocr_alloc(alloc, lora->norm_a->grad);
    # 分配内存给 lora 对象的 norm_b 的 grad 属性
    ggml_allocr_alloc(alloc, lora->norm_b->grad);
    # 分配内存给 lora 对象的 output_a 的 grad 属性
    ggml_allocr_alloc(alloc, lora->output_a->grad);
    # 分配内存给 lora 对象的 output_b 的 grad 属性
    ggml_allocr_alloc(alloc, lora->output_b->grad);
    # 遍历 LoRa 网络的每一层
    for (uint32_t i = 0; i < lora->layers.size(); ++i) {
        # 获取当前层的引用
        auto & layer = lora->layers[i];
        # 分配内存给注意力规范化层 A 的梯度
        ggml_allocr_alloc(alloc, layer.attention_norm_a->grad);
        # 分配内存给注意力规范化层 B 的梯度
        ggml_allocr_alloc(alloc, layer.attention_norm_b->grad);
        # 分配内存给查询权重 A 的梯度
        ggml_allocr_alloc(alloc, layer.wq_a->grad);
        # 分配内存给查询权重 B 的梯度
        ggml_allocr_alloc(alloc, layer.wq_b->grad);
        # 分配内存给键权重 A 的梯度
        ggml_allocr_alloc(alloc, layer.wk_a->grad);
        # 分配内存给键权重 B 的梯度
        ggml_allocr_alloc(alloc, layer.wk_b->grad);
        # 分配内存给数值权重 A 的梯度
        ggml_allocr_alloc(alloc, layer.wv_a->grad);
        # 分配内存给数值权重 B 的梯度
        ggml_allocr_alloc(alloc, layer.wv_b->grad);
        # 分配内存给输出权重 A 的梯度
        ggml_allocr_alloc(alloc, layer.wo_a->grad);
        # 分配内存给输出权重 B 的梯度
        ggml_allocr_alloc(alloc, layer.wo_b->grad);
        # 分配内存给前馈神经网络规范化层 A 的梯度
        ggml_allocr_alloc(alloc, layer.ffn_norm_a->grad);
        # 分配内存给前馈神经网络规范化层 B 的梯度
        ggml_allocr_alloc(alloc, layer.ffn_norm_b->grad);
        # 分配内存给第一层权重 A 的梯度
        ggml_allocr_alloc(alloc, layer.w1_a->grad);
        # 分配内存给第一层权重 B 的梯度
        ggml_allocr_alloc(alloc, layer.w1_b->grad);
        # 分配内存给第二层权重 A 的梯度
        ggml_allocr_alloc(alloc, layer.w2_a->grad);
        # 分配内存给第二层权重 B 的梯度
        ggml_allocr_alloc(alloc, layer.w2_b->grad);
        # 分配内存给第三层权重 A 的梯度
        ggml_allocr_alloc(alloc, layer.w3_a->grad);
        # 分配内存给第三层权重 B 的梯度
        ggml_allocr_alloc(alloc, layer.w3_b->grad);
    }
// 初始化 LoRa 模型，传入模型和 LoRa 结构体指针
static void init_lora(const struct my_llama_model * model, struct my_llama_lora * lora) {
    // 获取 LoRa 结构体中的超参数引用
    const auto & lparams = lora->hparams;

    // 从模型的超参数中获取各种参数值
    const uint32_t n_embd     = model->hparams.n_embd;
    const uint32_t n_embd_gqa = model->hparams.n_embd_gqa();
    const uint32_t n_layer    = model->hparams.n_layer;
    const uint32_t n_vocab    = model->hparams.n_vocab;
    const uint32_t n_ff       = model->hparams.n_ff;

    // 创建一个可变大小的字符数组用于存储字符串
    std::vector<char> tn_buf;
    tn_buf.resize(GGML_MAX_NAME);
    // 定义一个 lambda 函数用于生成带后缀的字符串
    auto tn = [&tn_buf](const char * key, const char * suffix) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), "%s%s", key, suffix);
        return tn_buf.data();
    };
    // 定义一个 lambda 函数用于生成带后缀和编号的字符串
    auto tni = [&tn_buf](const char * key, const char * suffix, int bid) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), key, bid);
        std::string s = tn_buf.data();
        snprintf(tn_buf.data(), tn_buf.size(), "%s%s", s.c_str(), suffix);
        return tn_buf.data();
    };

    // 初始化 LoRa 张量的上下文参数，包括内存大小和是否分配内存
    struct ggml_init_params ctx_lora_params;
    ctx_lora_params.mem_size   = ggml_tensor_overhead()*2*(6 + n_layer*18);
    ctx_lora_params.mem_buffer = NULL;
    ctx_lora_params.no_alloc   = true;

    // 初始化 LoRa 上下文
    struct ggml_context * ctx = ggml_init(ctx_lora_params);
    lora->ctx = ctx;

    // 创建 LoRa 张量
    lora->tok_embeddings_a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_tok_embeddings, n_embd);
    lora->tok_embeddings_b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_tok_embeddings, n_vocab);
    lora->norm_a           = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_norm, n_embd);
    lora->norm_b           = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_norm, 1);
    lora->output_a         = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_output, n_embd);
    lora->output_b         = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, lparams.n_rank_output, n_vocab);
}
    # 设置 lora->tok_embeddings_a 的名称
    ggml_set_name(lora->tok_embeddings_a, tn(LLM_TENSOR_TOKEN_EMBD,  ".weight.lora_a"));
    # 设置 lora->tok_embeddings_b 的名称
    ggml_set_name(lora->tok_embeddings_b, tn(LLM_TENSOR_TOKEN_EMBD,  ".weight.lora_b"));
    # 设置 lora->norm_a 的名称
    ggml_set_name(lora->norm_a,           tn(LLM_TENSOR_OUTPUT_NORM, ".weight.lora_a"));
    # 设置 lora->norm_b 的名称
    ggml_set_name(lora->norm_b,           tn(LLM_TENSOR_OUTPUT_NORM, ".weight.lora_b"));
    # 设置 lora->output_a 的名称
    ggml_set_name(lora->output_a,         tn(LLM_TENSOR_OUTPUT,      ".weight.lora_a"));
    # 设置 lora->output_b 的名称
    ggml_set_name(lora->output_b,         tn(LLM_TENSOR_OUTPUT,      ".weight.lora_b"));

    # 调整 lora->layers 的大小为 n_layer
    lora->layers.resize(n_layer)

    # 设置 lora 的参数
    set_param_lora(lora);

    # 测量数据大小
    size_t size = 0;
    # 遍历上下文中的张量，计算数据大小
    for (struct ggml_tensor * t = ggml_get_first_tensor(ctx); t != NULL; t = ggml_get_next_tensor(ctx, t)) {
        size += GGML_PAD(ggml_nbytes(t), tensor_alignment);
    }

    # 分配数据
    struct ggml_allocr * alloc = NULL;
    # 调整 lora->data 的大小为 size + tensor_alignment
    lora->data.resize(size + tensor_alignment);
    # 创建分配器对象，分配 lora->data 的内存
    alloc = ggml_allocr_new(lora->data.data(), lora->data.size(), tensor_alignment);
    # 为 lora 分配数据
    alloc_lora(alloc, lora);
    # 释放分配器对象
    ggml_allocr_free(alloc);
// 静态函数，用于对 LORA 结构体进行随机初始化
static void randomize_lora(struct my_llama_lora * lora, int seed, float mean, float std, float min, float max) {
    // 获取 LORA 结构体中层的数量
    const uint32_t n_layer = lora->layers.size();

    // 初始化随机正态分布对象
    struct random_normal_distribution * rnd = init_random_normal_distribution(seed, mean, std, min, max);

    // 对 LORA 结构体中的各张量进行正态分布随机初始化
    randomize_tensor_normal(lora->tok_embeddings_a, rnd);
    randomize_tensor_normal(lora->tok_embeddings_b, rnd);
    randomize_tensor_normal(lora->norm_a,           rnd);
    randomize_tensor_normal(lora->norm_b,           rnd);
    randomize_tensor_normal(lora->output_a,         rnd);
    randomize_tensor_normal(lora->output_b,         rnd);

    // 遍历 LORA 结构体中的每一层，对其中的张量进行正态分布随机初始化
    for (uint32_t i = 0; i < n_layer; ++i) {
        auto & layer = lora->layers[i];
        randomize_tensor_normal(layer.attention_norm_a, rnd);
        randomize_tensor_normal(layer.attention_norm_b, rnd);

        randomize_tensor_normal(layer.wq_a, rnd);
        randomize_tensor_normal(layer.wq_b, rnd);
        randomize_tensor_normal(layer.wk_a, rnd);
        randomize_tensor_normal(layer.wk_b, rnd);
        randomize_tensor_normal(layer.wv_a, rnd);
        randomize_tensor_normal(layer.wv_b, rnd);
        randomize_tensor_normal(layer.wo_a, rnd);
        randomize_tensor_normal(layer.wo_b, rnd);

        randomize_tensor_normal(layer.ffn_norm_a, rnd);
        randomize_tensor_normal(layer.ffn_norm_b, rnd);

        randomize_tensor_normal(layer.w1_a, rnd);
        randomize_tensor_normal(layer.w1_b, rnd);
        randomize_tensor_normal(layer.w2_a, rnd);
        randomize_tensor_normal(layer.w2_b, rnd);
        randomize_tensor_normal(layer.w3_a, rnd);
        randomize_tensor_normal(layer.w3_b, rnd);
    }

    // 释放随机正态分布对象的内存
    free_random_normal_distribution(rnd);
}
    // 构建 LORA 微调图
    static struct ggml_tensor * llama_build_lora_finetune_graphs(
        // 输入参数：模型、LORA、内存分配器、上下文、前向图、反向图、临时反向图、logits、tokens_input、targets、tokens数量、批处理数量、是否启用闪存注意力、是否启用检查点
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

    // 设置上下文的临时存储
    ggml_set_scratch(ctx, { 0, 0, nullptr, });
    // 初始化过去的 token 数量
    const int n_past = 0;
    // 初始化 N 为 token 数量
    const int N = n_tokens;
    // 获取模型的超参数
    const auto & hparams  = model->hparams;
    // 初始化上下文长度
    const int n_ctx       = hparams.n_ctx;
    // 初始化词汇表大小
    const int n_vocab     = hparams.n_vocab;
    // 初始化嵌入维度
    const int n_embd      = hparams.n_embd;
    // 初始化层数
    const int n_layer     = hparams.n_layer;
    // 初始化头数
    const int n_head      = hparams.n_head;
    // 初始化键值头数
    const int n_head_kv   = hparams.n_head_kv;
    // 初始化前馈层维度
    const int n_ff        = hparams.n_ff;
    // 初始化旋转维度
    const int n_rot       = hparams.n_embd_head();
    // 初始化头维度
    const int n_embd_head = hparams.n_embd_head();
    // 初始化 GQA 嵌入维度
    const int n_embd_gqa  = hparams.n_embd_gqa();
    // 初始化 RMS 归一化参数
    const float rms_norm_eps    = hparams.f_norm_rms_eps;
    // 初始化 ROPE 频率基数
    const float rope_freq_base  = hparams.rope_freq_base;
    // 初始化 ROPE 频率缩放
    const float rope_freq_scale = hparams.rope_freq_scale;

    // 断言层数与 LORA 层数相等
    GGML_ASSERT((size_t) n_layer == lora->layers.size());

    // 设置张量的名称
    auto set_name = [](struct ggml_tensor * t, const char * n) {
        ggml_set_name(t, n);
        if (t->grad) {
            ggml_format_name(t->grad, "%s->grad", n);
        }
    };

    // KQ_pos - 包含位置信息
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, N);
    ggml_allocr_alloc(alloc, KQ_pos);
    // 如果不是度量分配器
    if (!ggml_allocr_is_measure(alloc)) {
        int * data = (int *) KQ_pos->data;
        // 填充位置信息
        for (int i = 0; i < N; ++i) {
            data[i] = n_past + i;
        }
    // 自定义函数，用于处理 rope 参数过多的情况
    auto rope = [ctx, KQ_pos, n_rot, n_ctx, rope_freq_base, rope_freq_scale]
                (struct ggml_tensor * t) -> struct ggml_tensor * {
        // 不捕获这些变量，以消除警告
        const int rope_mode = 0;

        // 调用 ggml_rope_custom 函数处理参数
        return ggml_rope_custom(ctx,
            t, KQ_pos, n_rot, rope_mode, n_ctx, 0,
            rope_freq_base, rope_freq_scale, 0.0f, 1.0f, 0.0f, 0.0f
        );
    };

    // 设置 tokens_input 和 targets 的名称
    set_name(tokens_input, "tokens_input");
    set_name(targets,      "targets");

    // 断言 tokens_input 的类型为 GGML_TYPE_I32
    GGML_ASSERT(tokens_input->type == GGML_TYPE_I32);

    // 自定义函数，用于将两个 tensor 相加并转换为 GGML_TYPE_F32 类型
    auto add_to_f32 = [] (struct ggml_context * ctx, struct ggml_tensor * a, struct ggml_tensor * b) {
        if (ggml_is_quantized(a->type) || a->type == GGML_TYPE_F16) {
            return ggml_add_cast(ctx, a, b, GGML_TYPE_F32);
        } else if (a->type == GGML_TYPE_F32) {
            return ggml_add(ctx, a, b);
        } else {
            // 输出错误信息
            die_fmt("%s: Finetuning on tensors with type '%s' is not yet supported.\n",
                __func__, ggml_type_name(a->type));
        }
    };

    // 创建并初始化 tok_embeddings, norm, output tensor
    struct ggml_tensor * tok_embeddings = add_to_f32(ctx, model->tok_embeddings, ggml_mul_mat(ctx, lora->tok_embeddings_a, lora->tok_embeddings_b));
    struct ggml_tensor * norm           = add_to_f32(ctx, model->norm, ggml_mul_mat(ctx, lora->norm_a, lora->norm_b));
    struct ggml_tensor * output         = add_to_f32(ctx, model->output, ggml_mul_mat(ctx, lora->output_a, lora->output_b));

    // 创建并初始化 t00, t01 tensor
    struct ggml_tensor * t00 = ggml_reshape_1d(ctx, tokens_input, N*n_batch);  set_name(t00, "t00"); assert_shape_1d(t00, N*n_batch);
    struct ggml_tensor * t01 = ggml_get_rows(ctx, tok_embeddings, t00);        set_name(t01, "t01"); assert_shape_2d(t01, n_embd, N*n_batch);

    // 初始化 cur tensor
    struct ggml_tensor * cur = t01;

    // 创建一个 tensor 的 vector
    std::vector<struct ggml_tensor *> checkpoints;
    // 如果启用了检查点功能，则将输入 tokens_input、targets、t00、t01 加入检查点
    if (enable_checkpointing) {
        checkpoints.push_back(tokens_input);
        checkpoints.push_back(targets);
        checkpoints.push_back(t00);
        checkpoints.push_back(t01);
    }

    // 如果未启用闪电注意力机制，则创建 kv_scale 张量，并初始化为 1.0f/sqrtf(float(n_embd)/n_head)
    struct ggml_tensor * kv_scale = NULL;
    if (!enable_flash_attn) {
        kv_scale = ggml_new_f32(ctx, 1.0f/sqrtf(float(n_embd)/n_head));
    }

    // 创建 t31 张量，并进行 RMS 标准化操作
    struct ggml_tensor * t31   = ggml_rms_norm          (ctx, cur, rms_norm_eps);                    set_name(t31, "t31");     assert_shape_2d(t31, n_embd, N*n_batch);
    // 创建 t32 张量，并将 norm 重复拼接到 t31 上
    struct ggml_tensor * t32   = ggml_repeat            (ctx, norm, t31);                            set_name(t32, "t32");     assert_shape_2d(t32, n_embd, N*n_batch);
    // 创建 t33 张量，并将 t32 与 t31 对应位置元素相乘
    struct ggml_tensor * t33   = ggml_mul               (ctx, t32, t31);                             set_name(t33, "t33");     assert_shape_2d(t33, n_embd, N*n_batch);
    // 创建 t34 张量，并将 output 与 t33 矩阵相乘
    struct ggml_tensor * t34   = ggml_mul_mat           (ctx, output, t33);                          set_name(t34, "t34");     assert_shape_2d(t34, n_vocab, N*n_batch);
    // 创建 t35 张量，并将 t34 重塑为三维张量
    struct ggml_tensor * t35   = ggml_reshape_3d        (ctx, t34, n_vocab, N, n_batch);             set_name(t35, "t35");     assert_shape_3d(t35, n_vocab, N, n_batch);
    // 创建 t36 张量，并计算交叉熵损失
    struct ggml_tensor * t36   = ggml_cross_entropy_loss(ctx, t35, targets);                         set_name(t36, "t36");     assert_shape_1d(t36, 1);

    // 如果启用了检查点功能，则将 t31、t32、t33、t34、t35、t36 加入检查点
    if (enable_checkpointing) {
        checkpoints.push_back(t31);
        checkpoints.push_back(t32);
        checkpoints.push_back(t33);
        checkpoints.push_back(t34);
        checkpoints.push_back(t35);
        checkpoints.push_back(t36);
    }

    // 构建前向传播扩展
    ggml_build_forward_expand(gf, t36);

    // 如果启用了检查点功能，则构建反向传播梯度检查点
    if (enable_checkpointing) {
        ggml_build_backward_gradient_checkpointing(ctx, gf, gb, gb_tmp, checkpoints.data(), (int) checkpoints.size());
    } else {
        // 否则，复制前向传播图并构建反向传播扩展
        ggml_graph_cpy(gf, gb);
        ggml_build_backward_expand(ctx, gf, gb, true);
    }

    // 断言 alloc 不为空
    GGML_ASSERT(alloc != NULL);
    // 确保通过插入新的临时节点来防止重新分配一些张量
    int n_leafs_before = gb->n_leafs;
    int n_nodes_before = gb->n_nodes;
    // 创建一个值为1.0的新张量
    struct ggml_tensor * one = ggml_new_f32(ctx, 1.0f);
    // 输出张量
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t35, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t36, one));
    // 输入梯度
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, t36->grad, one));
    GGML_ASSERT(t36->grad->data == NULL && t36->grad->view_src == NULL);
    ggml_allocr_alloc(alloc, t36->grad);
    // KQ_pos
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, KQ_pos, one));

    // 确保基础模型张量的数据不能在可视化操作中使用
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, model->tok_embeddings, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, model->norm, one));
    ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, model->output, one));
    for (int il = 0; il < n_layer; ++il) {
        struct my_llama_layer & layer = model->layers[il];
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.attention_norm, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.ffn_norm, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.wq, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.wk, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.wv, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.wo, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.w1, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.w2, one));
        ggml_build_forward_expand(gb, ggml_scale_inplace(ctx, layer.w3, one));
    }

    // 在一个块中分配检查点以减少内存碎片化
    // 注意：它们将以相反的顺序被释放
    # 遍历检查点列表
    for (unsigned int i = 0; i < checkpoints.size(); ++i) {
        # 如果检查点的数据和视图源都为空
        if (checkpoints[i]->data == NULL && checkpoints[i]->view_src == NULL) {
            # 调用 ggml_allocr_alloc 函数进行分配
            ggml_allocr_alloc(alloc, checkpoints[i]);
        }
    }

    # 调用 ggml_allocr_alloc_graph 函数进行图形分配
    ggml_allocr_alloc_graph(alloc, gb);

    # 移除额外的节点和叶子
    for (int i = n_leafs_before; i < gb->n_leafs; ++i) {
        gb->leafs[i] = NULL;
    }
    for (int i = n_nodes_before; i < gb->n_nodes; ++i) {
        gb->nodes[i] = NULL;
    }
    gb->n_leafs = n_leafs_before;
    gb->n_nodes = n_nodes_before;

    # 设置 logits 指针指向 t35
    *logits = t35;
    # 返回 t36
    return t36;
}
// 从 GGUF 上下文中加载 LLAMA LORA 模型的参数
static void load_llama_lora_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model, struct my_llama_lora * lora) {
    // 注意：gguf_context 必须使用 f_ggml_ctx 进行初始化，并且 no_alloc=false，否则无法读取张量数据

    std::string arch; // 定义存储架构信息的字符串变量

    std::vector<char> keybuf; // 定义存储密钥的字符向量
    keybuf.resize(512); // 调整密钥向量的大小为 512

    GGUF_GET_KEY(fctx, arch, gguf_get_val_str, GGUF_TYPE_STRING, true, LLM_KV_GENERAL_ARCHITECTURE); // 从 GGUF 上下文中获取架构信息
    GGML_ASSERT(arch == "llama"); // 断言架构信息为 "llama"

    uint32_t ftype_u; // 定义存储文件类型的无符号整数变量
    GGUF_GET_KEY(fctx, ftype_u, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_GENERAL_FILE_TYPE); // 从 GGUF 上下文中获取文件类型
    GGML_ASSERT((enum llama_ftype) ftype_u == LLAMA_FTYPE_ALL_F32); // 断言文件类型为 LLAMA_FTYPE_ALL_F32

    struct my_llama_hparams hparams; // 定义存储 LLAMA 模型超参数的结构体变量
    load_model_hparams_gguf(fctx, &hparams, arch.c_str()); // 从 GGUF 上下文中加载模型超参数

    // 断言定义张量形状的参数必须匹配
    GGML_ASSERT(hparams.n_embd    == model->hparams.n_embd);
    GGML_ASSERT(hparams.n_ff      == model->hparams.n_ff);
    GGML_ASSERT(hparams.n_head    == model->hparams.n_head);
    GGML_ASSERT(hparams.n_head_kv == model->hparams.n_head_kv);
    GGML_ASSERT(hparams.n_layer   == model->hparams.n_layer);

    // 从 GGUF 上下文中获取 LORA 模型的参数
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_tok_embeddings, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD);
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_norm,           gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM);
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_output,         gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_OUTPUT);
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_attention_norm, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_NORM);
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_wq,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_Q);
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_wk,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_K);
    # 通过 GGUF_GET_KEY 函数获取指定参数的数值，存入 lora->hparams.n_rank_wv 中
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_wv,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_V);
    # 通过 GGUF_GET_KEY 函数获取指定参数的数值，存入 lora->hparams.n_rank_wo 中
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_wo,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_ATTN_OUT);
    # 通过 GGUF_GET_KEY 函数获取指定参数的数值，存入 lora->hparams.n_rank_ffn_norm 中
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_ffn_norm,       gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_FFN_NORM);
    # 通过 GGUF_GET_KEY 函数获取指定参数的数值，存入 lora->hparams.n_rank_w1 中
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_w1,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_FFN_GATE);
    # 通过 GGUF_GET_KEY 函数获取指定参数的数值，存入 lora->hparams.n_rank_w2 中
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_w2,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_FFN_DOWN);
    # 通过 GGUF_GET_KEY 函数获取指定参数的数值，存入 lora->hparams.n_rank_w3 中
    GGUF_GET_KEY(fctx, lora->hparams.n_rank_w3,             gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_LORA_RANK_FFN_UP);

    # 初始化 lora 模型
    init_lora(model, lora);

    # 通过 copy_tensor_by_name 函数将 f_ggml_ctx 中指定名称的张量拷贝到 lora 模型中对应的张量
    copy_tensor_by_name(lora->tok_embeddings_a, f_ggml_ctx, ggml_get_name(lora->tok_embeddings_a));
    copy_tensor_by_name(lora->tok_embeddings_b, f_ggml_ctx, ggml_get_name(lora->tok_embeddings_b));
    copy_tensor_by_name(lora->norm_a,           f_ggml_ctx, ggml_get_name(lora->norm_a));
    copy_tensor_by_name(lora->norm_b,           f_ggml_ctx, ggml_get_name(lora->norm_b));
    copy_tensor_by_name(lora->output_a,         f_ggml_ctx, ggml_get_name(lora->output_a));
    copy_tensor_by_name(lora->output_b,         f_ggml_ctx, ggml_get_name(lora->output_b));
    # 遍历 LoRa 网络的每一层
    for (uint32_t i = 0; i < lora->layers.size(); ++i) {
        # 获取当前层的引用
        auto & layer = lora->layers[i];
        # 通过名称复制注意力规范化层 A 的张量
        copy_tensor_by_name(layer.attention_norm_a, f_ggml_ctx, ggml_get_name(layer.attention_norm_a));
        # 通过名称复制注意力规范化层 B 的张量
        copy_tensor_by_name(layer.attention_norm_b, f_ggml_ctx, ggml_get_name(layer.attention_norm_b));
        # 通过名称复制查询权重 A 的张量
        copy_tensor_by_name(layer.wq_a,             f_ggml_ctx, ggml_get_name(layer.wq_a));
        # 通过名称复制查询权重 B 的张量
        copy_tensor_by_name(layer.wq_b,             f_ggml_ctx, ggml_get_name(layer.wq_b));
        # 通过名称复制键权重 A 的张量
        copy_tensor_by_name(layer.wk_a,             f_ggml_ctx, ggml_get_name(layer.wk_a));
        # 通过名称复制键权重 B 的张量
        copy_tensor_by_name(layer.wk_b,             f_ggml_ctx, ggml_get_name(layer.wk_b));
        # 通过名称复制数值权重 A 的张量
        copy_tensor_by_name(layer.wv_a,             f_ggml_ctx, ggml_get_name(layer.wv_a));
        # 通过名称复制数值权重 B 的张量
        copy_tensor_by_name(layer.wv_b,             f_ggml_ctx, ggml_get_name(layer.wv_b));
        # 通过名称复制输出权重 A 的张量
        copy_tensor_by_name(layer.wo_a,             f_ggml_ctx, ggml_get_name(layer.wo_a));
        # 通过名称复制输出权重 B 的张量
        copy_tensor_by_name(layer.wo_b,             f_ggml_ctx, ggml_get_name(layer.wo_b));
        # 通过名称复制前馈神经网络规范化层 A 的张量
        copy_tensor_by_name(layer.ffn_norm_a,       f_ggml_ctx, ggml_get_name(layer.ffn_norm_a));
        # 通过名称复制前馈神经网络规范化层 B 的张量
        copy_tensor_by_name(layer.ffn_norm_b,       f_ggml_ctx, ggml_get_name(layer.ffn_norm_b));
        # 通过名称复制第一层前馈神经网络权重 A 的张量
        copy_tensor_by_name(layer.w1_a,             f_ggml_ctx, ggml_get_name(layer.w1_a));
        # 通过名称复制第一层前馈神经网络权重 B 的张量
        copy_tensor_by_name(layer.w1_b,             f_ggml_ctx, ggml_get_name(layer.w1_b));
        # 通过名称复制第二层前馈神经网络权重 A 的张量
        copy_tensor_by_name(layer.w2_a,             f_ggml_ctx, ggml_get_name(layer.w2_a));
        # 通过名称复制第二层前馈神经网络权重 B 的张量
        copy_tensor_by_name(layer.w2_b,             f_ggml_ctx, ggml_get_name(layer.w2_b));
        # 通过名称复制第三层前馈神经网络权重 A 的张量
        copy_tensor_by_name(layer.w3_a,             f_ggml_ctx, ggml_get_name(layer.w3_a));
        # 通过名称复制第三层前馈神经网络权重 B 的张量
        copy_tensor_by_name(layer.w3_b,             f_ggml_ctx, ggml_get_name(layer.w3_b));
    }
# 保存 LLAMA 模型和 LORA 参数到 GGUF 上下文中
static void save_llama_lora_gguf(struct gguf_context * fctx, struct my_llama_model * model, struct my_llama_lora * lora) {
    # 定义 LLAMA 架构
    const char * arch = "llama";
    # 定义 LLAMA 文件类型
    enum llama_ftype ftype = LLAMA_FTYPE_ALL_F32;

    # 创建存储键的缓冲区
    std::vector<char> keybuf;
    keybuf.resize(512);
    # 定义键值对的 lambda 函数
    auto kv = [arch, &keybuf](const char * key) -> const char * {
        # 格式化键值对的键
        snprintf(keybuf.data(), keybuf.size(), key, arch);
        return keybuf.data();
    };

    # 设置 GGUF 上下文中的 LLAMA 架构
    gguf_set_val_str(fctx, LLM_KV_GENERAL_ARCHITECTURE, arch);
    # 设置 GGUF 上下文中的 LLAMA 文件类型
    gguf_set_val_u32(fctx, LLM_KV_GENERAL_FILE_TYPE, ftype);

    # 设置 GGUF 上下文中的 LLAMA 模型参数
    gguf_set_val_u32(fctx, kv(LLM_KV_CONTEXT_LENGTH),              model->hparams.n_ctx);
    gguf_set_val_u32(fctx, kv(LLM_KV_EMBEDDING_LENGTH),            model->hparams.n_embd);
    gguf_set_val_u32(fctx, kv(LLM_KV_FEED_FORWARD_LENGTH),         model->hparams.n_ff);
    gguf_set_val_u32(fctx, kv(LLM_KV_ATTENTION_HEAD_COUNT),        model->hparams.n_head);
    gguf_set_val_u32(fctx, kv(LLM_KV_ATTENTION_HEAD_COUNT_KV),     model->hparams.n_head_kv);
    gguf_set_val_u32(fctx, kv(LLM_KV_BLOCK_COUNT),                 model->hparams.n_layer);
    gguf_set_val_u32(fctx, kv(LLM_KV_ROPE_DIMENSION_COUNT),        model->hparams.n_embd_head());
    gguf_set_val_f32(fctx, kv(LLM_KV_ATTENTION_LAYERNORM_RMS_EPS), model->hparams.f_norm_rms_eps);
    gguf_set_val_f32(fctx, kv(LLM_KV_ROPE_FREQ_BASE),              model->hparams.rope_freq_base);
    gguf_set_val_f32(fctx, kv(LLM_KV_ROPE_SCALE_LINEAR),           model->hparams.rope_freq_scale);

    # 设置 GGUF 上下文中的 LORA 参数
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_TOKEN_EMBD,   lora->hparams.n_rank_tok_embeddings);
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_OUTPUT_NORM,  lora->hparams.n_rank_norm);
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_OUTPUT,       lora->hparams.n_rank_output);
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_NORM,    lora->hparams.n_rank_attention_norm);
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_Q,       lora->hparams.n_rank_wq);
    # 设置 LLM_KV_TRAINING_LORA_RANK_ATTN_K 键对应的数值为 lora->hparams.n_rank_wk
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_K,       lora->hparams.n_rank_wk);
    # 设置 LLM_KV_TRAINING_LORA_RANK_ATTN_V 键对应的数值为 lora->hparams.n_rank_wv
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_V,       lora->hparams.n_rank_wv);
    # 设置 LLM_KV_TRAINING_LORA_RANK_ATTN_OUT 键对应的数值为 lora->hparams.n_rank_wo
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_ATTN_OUT,     lora->hparams.n_rank_wo);
    # 设置 LLM_KV_TRAINING_LORA_RANK_FFN_NORM 键对应的数值为 lora->hparams.n_rank_ffn_norm
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_FFN_NORM,     lora->hparams.n_rank_ffn_norm);
    # 设置 LLM_KV_TRAINING_LORA_RANK_FFN_GATE 键对应的数值为 lora->hparams.n_rank_w1
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_FFN_GATE,     lora->hparams.n_rank_w1);
    # 设置 LLM_KV_TRAINING_LORA_RANK_FFN_DOWN 键对应的数值为 lora->hparams.n_rank_w2
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_FFN_DOWN,     lora->hparams.n_rank_w2);
    # 设置 LLM_KV_TRAINING_LORA_RANK_FFN_UP 键对应的数值为 lora->hparams.n_rank_w3
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_LORA_RANK_FFN_UP,       lora->hparams.n_rank_w3);

    # 将 lora->tok_embeddings_a 添加到 fctx 中
    gguf_add_tensor(fctx, lora->tok_embeddings_a);
    # 将 lora->tok_embeddings_b 添加到 fctx 中
    gguf_add_tensor(fctx, lora->tok_embeddings_b);
    # 将 lora->norm_a 添加到 fctx 中
    gguf_add_tensor(fctx, lora->norm_a);
    # 将 lora->norm_b 添加到 fctx 中
    gguf_add_tensor(fctx, lora->norm_b);
    # 将 lora->output_a 添加到 fctx 中
    gguf_add_tensor(fctx, lora->output_a);
    # 将 lora->output_b 添加到 fctx 中
    gguf_add_tensor(fctx, lora->output_b);

    # 遍历 lora->layers 中的每一个元素
    for (uint32_t i = 0; i < lora->layers.size(); ++i) {
        auto & layer = lora->layers[i];
        # 将 layer.attention_norm_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.attention_norm_a);
        # 将 layer.attention_norm_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.attention_norm_b);
        # 将 layer.wq_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.wq_a);
        # 将 layer.wq_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.wq_b);
        # 将 layer.wk_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.wk_a);
        # 将 layer.wk_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.wk_b);
        # 将 layer.wv_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.wv_a);
        # 将 layer.wv_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.wv_b);
        # 将 layer.wo_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.wo_a);
        # 将 layer.wo_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.wo_b);
        # 将 layer.ffn_norm_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.ffn_norm_a);
        # 将 layer.ffn_norm_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.ffn_norm_b);
        # 将 layer.w1_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.w1_a);
        # 将 layer.w1_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.w1_b);
        # 将 layer.w2_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.w2_a);
        # 将 layer.w2_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.w2_b);
        # 将 layer.w3_a 添加到 fctx 中
        gguf_add_tensor(fctx, layer.w3_a);
        # 将 layer.w3_b 添加到 fctx 中
        gguf_add_tensor(fctx, layer.w3_b);
    }
// 加载 LORA 模型的检查点数据，从 GGUF 上下文中加载
static void load_checkpoint_lora_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct my_llama_model * model, struct my_llama_lora * lora, struct train_state * train) {
    // 设置训练类型为 FINETUNE_LORA
    std::string train_type = LLM_KV_TRAINING_TYPE_FINETUNE_LORA;
    // 从 GGUF 上下文中获取训练类型的值
    GGUF_GET_KEY(fctx, train_type, gguf_get_val_str, GGUF_TYPE_STRING, false, LLM_KV_TRAINING_TYPE);
    // 断言训练类型为 FINETUNE_LORA
    GGML_ASSERT(train_type == LLM_KV_TRAINING_TYPE_FINETUNE_LORA);

    // 从 GGUF 上下文中加载训练状态
    load_train_state_gguf(fctx, f_ggml_ctx, train);
    // 从 GGUF 上下文中加载 LORA 模型
    load_llama_lora_gguf(fctx, f_ggml_ctx, model, lora);
}

// 保存 LORA 模型的检查点数据到 GGUF 上下文
static void save_checkpoint_lora_gguf(struct gguf_context * fctx, struct my_llama_model * model, struct my_llama_lora * lora, struct train_state * train) {
    // 设置训练类型为 FINETUNE_LORA
    gguf_set_val_str(fctx, LLM_KV_TRAINING_TYPE, LLM_KV_TRAINING_TYPE_FINETUNE_LORA);
    // 保存 LORA 模型的检查点数据到 GGUF 上下文
    save_llama_lora_gguf(fctx, model, lora);
    // 保存训练状态到 GGUF 上下文
    save_train_state_gguf(fctx, train);
}

// 从文件中加载 LORA 模型的检查点数据
static bool load_checkpoint_lora_file(const char * filename, struct my_llama_model * model, struct my_llama_lora * lora, struct train_state * train) {
    // 初始化 GGML 上下文
    struct ggml_context * f_ggml_ctx;
    // 初始化 GGUF 上下文参数
    struct gguf_init_params params;
    params.no_alloc = false;
    params.ctx = &f_ggml_ctx;
    // 从文件中初始化 GGUF 上下文
    struct gguf_context * fctx = gguf_init_from_file(filename, params);
    // 如果初始化失败，则返回 false
    if (fctx == NULL) {
        return false;
    }

    // 从 GGUF 上下文中加载 LORA 模型的检查点数据
    load_checkpoint_lora_gguf(fctx, f_ggml_ctx, model, lora, train);

    // 释放 GGUF 上下文
    gguf_free(fctx);
    return true;
}

// 保存 LORA 模型的检查点数据到文件
static void save_checkpoint_lora_file(const char * filename, struct my_llama_model * model, struct my_llama_lora * lora, struct train_state * train) {
    // 打印保存文件的信息
    printf("%s: saving to %s\n", __func__, filename);
    // 初始化空的 GGUF 上下文
    struct gguf_context * fctx = gguf_init_empty();

    // 保存 LORA 模型的检查点数据到 GGUF 上下文
    save_checkpoint_lora_gguf(fctx, model, lora, train);

    // 写入文件
    const bool only_meta = false;
    gguf_write_to_file(fctx, filename, only_meta);
    // 释放 GGUF 上下文
    gguf_free(fctx);
}

// 定义 llama_file 结构体
struct llama_file {
    // 使用 FILE * 以便于 mmap 时不需要重新打开文件
    FILE * fp;
    // 文件大小
    size_t size;
    # 定义一个名为 llama_file 的函数，接受文件名和模式作为参数
    llama_file(const char * fname, const char * mode) {
        # 使用标准库函数 fopen 打开文件，并将文件指针赋值给 fp
        fp = std::fopen(fname, mode);
        # 如果文件指针为空，说明文件打开失败，将 size 设为 0
        if (fp == NULL) {
            size = 0;
        } else {
            # 如果文件打开成功，将文件指针移动到文件末尾
            seek(0, SEEK_END);
            # 获取文件指针当前位置，即文件大小，并赋值给 size
            size = tell();
            # 将文件指针移动到文件开头
            seek(0, SEEK_SET);
        }
    }

    # 定义一个名为 tell 的函数，返回类型为 size_t
    size_t tell() const {
    // 根据操作系统不同，使用不同的函数获取文件指针当前位置
    #ifdef _WIN32
        __int64 ret = _ftelli64(fp);
    #else
        long ret = std::ftell(fp);
    #endif
    // 断言文件指针位置不为-1，如果为-1则输出错误信息
    GGML_ASSERT(ret != -1); // this really shouldn't fail
    // 返回文件指针位置
    return (size_t) ret;
    }

    // 根据偏移量和起始位置设置文件指针位置
    void seek(size_t offset, int whence) {
    // 根据操作系统不同，使用不同的函数设置文件指针位置
    #ifdef _WIN32
        int ret = _fseeki64(fp, (__int64) offset, whence);
    #else
        int ret = std::fseek(fp, (long) offset, whence);
    #endif
    // 断言设置文件指针位置成功，如果不成功则输出错误信息
    GGML_ASSERT(ret == 0); // same
    }

    // 从文件中读取原始数据
    void read_raw(void * ptr, size_t size) {
        // 如果要读取的数据大小为0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 从文件中读取数据到指定的内存位置
        std::size_t ret = std::fread(ptr, size, 1, fp);
        // 如果发生读取错误，则输出错误信息
        if (ferror(fp)) {
            die_fmt("read error: %s", strerror(errno));
        }
        // 如果读取的数据大小不符合预期，则输出错误信息
        if (ret != 1) {
            die("unexpectedly reached end of file");
        }
    }

    // 从文件中读取32位无符号整数
    std::uint32_t read_u32() {
        std::uint32_t ret;
        // 从文件中读取32位无符号整数
        read_raw(&ret, sizeof(ret));
        return ret;
    }

    // 从文件中读取指定长度的字符串
    std::string read_string(std::uint32_t len) {
        std::vector<char> chars(len);
        // 从文件中读取指定长度的字符串到字符数组中
        read_raw(chars.data(), len);
        return std::string(chars.data(), len);
    }

    // 向文件中写入原始数据
    void write_raw(const void * ptr, size_t size) {
        // 如果要写入的数据大小为0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 向文件中写入数据
        size_t ret = std::fwrite(ptr, size, 1, fp);
        // 如果写入的数据大小不符合预期，则输出错误信息
        if (ret != 1) {
            die_fmt("write error: %s", strerror(errno));
        }
    }

    // 向文件中写入32位无符号整数
    void write_u32(std::uint32_t val) {
        // 向文件中写入32位无符号整数
        write_raw(&val, sizeof(val));
    }

    // 析构函数，关闭文件指针
    ~llama_file() {
        if (fp) {
            std::fclose(fp);
        }
    }
};

// 写入张量数据到文件中
static void write_tensor(struct llama_file * file, struct ggml_tensor * tensor, const char * name) {
    // 如果张量为空，则写入3个0和一个特定值，然后返回
    if (tensor == NULL) {
        file->write_u32(0);
        file->write_u32(0);
        file->write_u32(GGML_TYPE_F32);
        file->seek((0-file->tell()) & 31, SEEK_CUR);
        return;
    }
    // 如果名称为空，则使用张量的名称
    if (name == NULL) {
        name = ggml_get_name(tensor);
    }
    // 获取名称的长度
    uint32_t name_len = strlen(name);
    // 获取张量的维度
    uint32_t nd = tensor->n_dims;
    # 将tensor的维度转换为uint32_t类型，并存储在数组ne中
    uint32_t ne[4] = { (uint32_t)tensor->ne[0],
                       (uint32_t)tensor->ne[1],
                       (uint32_t)tensor->ne[2],
                       (uint32_t)tensor->ne[3] };
    # 将nd写入文件
    file->write_u32(nd);
    # 将name_len写入文件
    file->write_u32(name_len);
    # 将tensor的类型写入文件
    file->write_u32(tensor->type);
    # 将ne数组中的数据以nd个sizeof(ne[0])大小的块写入文件
    file->write_raw(ne, sizeof(ne[0]) * nd);
    # 将name中的name_len个字节写入文件
    file->write_raw(name, name_len);
    # 将文件指针移动到当前位置的负值与31的按位与结果，即将文件指针移动到32字节对齐的位置
    file->seek((0-file->tell()) & 31, SEEK_CUR);
    # 将tensor的数据以ggml_nbytes(tensor)大小写入文件
    file->write_raw(tensor->data, ggml_nbytes(tensor));
}
// 将数据保存为 llama_lora 格式的文件
static void save_as_llama_lora(const char * filename, struct my_llama_lora * lora) {
    // 打印保存的文件名
    printf("%s: saving to %s\n", __func__, filename);
    // 创建 llama_file 对象，以写入二进制模式打开文件
    struct llama_file file(filename, "wb");
    // 如果文件指针为空，则返回
    if (file.fp == NULL) {
        return;
    }

    // 创建存储临时字符串的缓冲区
    std::vector<char> tn_buf;
    tn_buf.resize(GGML_MAX_NAME);

    // 定义 lambda 函数 tn，用于拼接字符串
    auto tn = [&tn_buf](const char * key, const char * suffix) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), "%s%s", key, suffix);
        return tn_buf.data();
    };

    // 定义 lambda 函数 tni，用于拼接字符串
    auto tni = [&tn_buf](const char * key, int bid, const char * suffix) -> const char * {
        snprintf(tn_buf.data(), tn_buf.size(), key, bid);
        std::string s = tn_buf.data();
        snprintf(tn_buf.data(), tn_buf.size(), "%s%s", s.c_str(), suffix);
        return tn_buf.data();
    };

    // 定义 llama_lora 文件的魔术数字
    uint32_t LLAMA_FILE_MAGIC_LORA = 0x67676C61; // 'ggla'
    // 写入魔术数字
    file.write_u32(LLAMA_FILE_MAGIC_LORA);   // magic
    file.write_u32(1); // version
    // 写入超参数
    file.write_u32(lora->hparams.lora_r);
    file.write_u32(lora->hparams.lora_alpha);
    // 写入张量数据
    write_tensor(&file, lora->tok_embeddings_a, tn(LLM_TENSOR_TOKEN_EMBD,  ".weight.loraA"));
    write_tensor(&file, lora->tok_embeddings_b, tn(LLM_TENSOR_TOKEN_EMBD,  ".weight.loraB"));
    write_tensor(&file, lora->norm_a,           tn(LLM_TENSOR_OUTPUT_NORM, ".weight.loraA"));
    write_tensor(&file, lora->norm_b,           tn(LLM_TENSOR_OUTPUT_NORM, ".weight.loraB"));
    write_tensor(&file, lora->output_a,         tn(LLM_TENSOR_OUTPUT,      ".weight.loraA"));
    write_tensor(&file, lora->output_b,         tn(LLM_TENSOR_OUTPUT,      ".weight.loraB"));
    # 遍历 LoRa 网络的每一层
    for (uint32_t i = 0; i < lora->layers.size(); ++i) {
        # 获取当前层的引用
        auto & layer = lora->layers[i];
        # 写入当前层的注意力规范化 A 权重张量
        write_tensor(&file, layer.attention_norm_a, tni(LLM_TENSOR_ATTN_NORM, i, ".weight.loraA"));
        # 写入当前层的注意力规范化 B 权重张量
        write_tensor(&file, layer.attention_norm_b, tni(LLM_TENSOR_ATTN_NORM, i, ".weight.loraB"));
        # 写入当前层的注意力查询 A 权重张量
        write_tensor(&file, layer.wq_a,             tni(LLM_TENSOR_ATTN_Q,    i, ".weight.loraA"));
        # 写入当前层的注意力查询 B 权重张量
        write_tensor(&file, layer.wq_b,             tni(LLM_TENSOR_ATTN_Q,    i, ".weight.loraB"));
        # 写入当前层的注意力键 A 权重张量
        write_tensor(&file, layer.wk_a,             tni(LLM_TENSOR_ATTN_K,    i, ".weight.loraA"));
        # 写入当前层的注意力键 B 权重张量
        write_tensor(&file, layer.wk_b,             tni(LLM_TENSOR_ATTN_K,    i, ".weight.loraB"));
        # 写入当前层的注意力值 A 权重张量
        write_tensor(&file, layer.wv_a,             tni(LLM_TENSOR_ATTN_V,    i, ".weight.loraA"));
        # 写入当前层的注意力值 B 权重张量
        write_tensor(&file, layer.wv_b,             tni(LLM_TENSOR_ATTN_V,    i, ".weight.loraB"));
        # 写入当前层的注意力输出 A 权重张量
        write_tensor(&file, layer.wo_a,             tni(LLM_TENSOR_ATTN_OUT,  i, ".weight.loraA"));
        # 写入当前层的注意力输出 B 权重张量
        write_tensor(&file, layer.wo_b,             tni(LLM_TENSOR_ATTN_OUT,  i, ".weight.loraB"));
        # 写入当前层的前馈神经网络规范化 A 权重张量
        write_tensor(&file, layer.ffn_norm_a,       tni(LLM_TENSOR_FFN_NORM,  i, ".weight.loraA"));
        # 写入当前层的前馈神经网络规范化 B 权重张量
        write_tensor(&file, layer.ffn_norm_b,       tni(LLM_TENSOR_FFN_NORM,  i, ".weight.loraB"));
        # 写入当前层的前馈神经网络第一层 A 权重张量
        write_tensor(&file, layer.w1_a,             tni(LLM_TENSOR_FFN_GATE,  i, ".weight.loraA"));
        # 写入当前层的前馈神经网络第一层 B 权重张量
        write_tensor(&file, layer.w1_b,             tni(LLM_TENSOR_FFN_GATE,  i, ".weight.loraB"));
        # 写入当前层的前馈神经网络第二层 A 权重张量
        write_tensor(&file, layer.w2_a,             tni(LLM_TENSOR_FFN_DOWN,  i, ".weight.loraA"));
        # 写入当前层的前馈神经网络第二层 B 权重张量
        write_tensor(&file, layer.w2_b,             tni(LLM_TENSOR_FFN_DOWN,  i, ".weight.loraB"));
        # 写入当前层的前馈神经网络第三层 A 权重张量
        write_tensor(&file, layer.w3_a,             tni(LLM_TENSOR_FFN_UP,    i, ".weight.loraA"));
        # 写入当前层的前馈神经网络第三层 B 权重张量
        write_tensor(&file, layer.w3_b,             tni(LLM_TENSOR_FFN_UP,    i, ".weight.loraB"));
    }
// 结构体定义，包含训练参数
struct train_params {
    // 公共训练参数
    struct train_params_common common;

    // 模型基本文件名
    const char * fn_model_base;
    // LoRa 输出文件名
    const char * fn_lora_out;

    // 是否只写入 LoRa
    bool only_write_lora;

    // 归一化 RMS 误差
    float f_norm_rms_eps;
    // Rope 频率基数
    float rope_freq_base;
    // Rope 频率缩放
    float rope_freq_scale;

    // 自定义归一化 RMS 误差
    bool custom_f_norm_rms_eps;
    // 自定义 Rope 频率基数
    bool custom_rope_freq_base;
    // 自定义 Rope 频率缩放
    bool custom_rope_freq_scale;

    // LoRa 参数 r
    int32_t lora_r;
    // LoRa 参数 alpha
    int32_t lora_alpha;
    // 自定义 LoRa 参数 alpha
    bool custom_lora_alpha;

    // 排名注意力规范化
    uint32_t n_rank_attention_norm;
    // 排名 wq
    uint32_t n_rank_wq;
    // 排名 wk
    uint32_t n_rank_wk;
    // 排名 wv
    uint32_t n_rank_wv;
    // 排名 wo
    uint32_t n_rank_wo;
    // 排名 ffn_norm
    uint32_t n_rank_ffn_norm;
    // 排名 w1
    uint32_t n_rank_w1;
    // ...（省略部分参数）

};

// 获取默认的训练参数
static struct train_params get_default_train_params() {
    // 初始化训练参数结构体
    struct train_params params;
    // 获取默认的公共训练参数
    params.common = get_default_train_params_common();
    // 模型基本文件名为空字符串
    params.fn_model_base     = "";
    // LoRa 输出文件名为默认值
    params.fn_lora_out       = "ggml-lora-ITERATION-f32.gguf";
    // ...（省略部分参数初始化）
    # 设置参数 n_rank_w2 的值为 4
    params.n_rank_w2             = 4;
    # 设置参数 n_rank_w3 的值为 4
    params.n_rank_w3             = 4;
    # 设置参数 n_rank_tok_embeddings 的值为 4
    params.n_rank_tok_embeddings = 4;
    # 设置参数 n_rank_norm 的值为 1
    params.n_rank_norm           = 1;
    # 设置参数 n_rank_output 的值为 4
    params.n_rank_output         = 4;

    # 设置参数 custom_n_rank_attention_norm 的值为 false
    params.custom_n_rank_attention_norm = false;
    # 设置参数 custom_n_rank_wq 的值为 false
    params.custom_n_rank_wq             = false;
    # 设置参数 custom_n_rank_wk 的值为 false
    params.custom_n_rank_wk             = false;
    # 设置参数 custom_n_rank_wv 的值为 false
    params.custom_n_rank_wv             = false;
    # 设置参数 custom_n_rank_wo 的值为 false
    params.custom_n_rank_wo             = false;
    # 设置参数 custom_n_rank_ffn_norm 的值为 false
    params.custom_n_rank_ffn_norm       = false;
    # 设置参数 custom_n_rank_w1 的值为 false
    params.custom_n_rank_w1             = false;
    # 设置参数 custom_n_rank_w2 的值为 false
    params.custom_n_rank_w2             = false;
    # 设置参数 custom_n_rank_w3 的值为 false
    params.custom_n_rank_w3             = false;
    # 设置参数 custom_n_rank_tok_embeddings 的值为 false
    params.custom_n_rank_tok_embeddings = false;
    # 设置参数 custom_n_rank_norm 的值为 false
    params.custom_n_rank_norm           = false;
    # 设置参数 custom_n_rank_output 的值为 false
    params.custom_n_rank_output         = false;

    # 返回参数对象
    return params;
# 打印训练程序的使用说明
static void train_print_usage(int argc, char ** argv, const struct train_params * params) {
    # 打印程序的使用方式
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help                 show this help message and exit\n");

    # 打印模型基础路径的参数说明和默认值
    fprintf(stderr, "  --model-base FNAME         model path from which to load base model (default '%s')\n", params->fn_model_base);
    # 打印保存 llama lora 路径的参数说明和默认值
    fprintf(stderr, "  --lora-out FNAME           path to save llama lora (default '%s')\n", params->fn_lora_out);
    # 打印只保存 llama lora 的参数说明
    fprintf(stderr, "  --only-write-lora          only save llama lora, don't do any training.  use this if you only want to convert a checkpoint to a lora adapter.\n");
    # 打印 RMS-Norm epsilon 值的参数说明和默认值
    fprintf(stderr, "  --norm-rms-eps F           RMS-Norm epsilon value (default %f)\n", params->f_norm_rms_eps);
    # 打印 ROPE 频率基础值的参数说明和默认值
    fprintf(stderr, "  --rope-freq-base F         Frequency base for ROPE (default %f)\n", params->rope_freq_base);
    # 打印 ROPE 频率缩放值的参数说明和默认值
    fprintf(stderr, "  --rope-freq-scale F        Frequency scale for ROPE (default %f)\n", params->rope_freq_scale);
    # 打印 LORA alpha 值的参数说明和默认值
    fprintf(stderr, "  --lora-alpha N             LORA alpha : resulting LORA scaling is alpha/r. (default %d)\n", params->lora_alpha);
    # 打印 LORA r 值的参数说明和默认值
    fprintf(stderr, "  --lora-r N                 LORA r: default rank. Also specifies resulting scaling together with lora-alpha. (default %d)\n", params->lora_r);
    # 打印用于注意力规范张量的 LORA rank 的参数说明
    fprintf(stderr, "  --rank-att-norm N          LORA rank for attention norm tensor, overrides default rank. Norm tensors should generally have rank 1.\n");
    # 打印用于前馈规范张量的 LORA rank 的参数说明
    fprintf(stderr, "  --rank-ffn-norm N          LORA rank for feed-forward norm tensor, overrides default rank. Norm tensors should generally have rank 1.\n");
    # 打印用于输出规范张量的 LORA rank 的参数说明
    fprintf(stderr, "  --rank-out-norm N          LORA rank for output norm tensor, overrides default rank. Norm tensors should generally have rank 1.\n");
    # 打印用于标记嵌入张量的 LORA rank 的参数说明
    fprintf(stderr, "  --rank-tok-embd N          LORA rank for token embeddings tensor, overrides default rank.\n");
}
    # 打印输出信息，指定 LORA 输出张量的秩，覆盖默认秩
    fprintf(stderr, "  --rank-out N               LORA rank for output tensor, overrides default rank.\n");
    # 打印输出信息，指定 LORA wq 张量的秩，覆盖默认秩
    fprintf(stderr, "  --rank-wq N                LORA rank for wq tensor, overrides default rank.\n");
    # 打印输出信息，指定 LORA wk 张量的秩，覆盖默认秩
    fprintf(stderr, "  --rank-wk N                LORA rank for wk tensor, overrides default rank.\n");
    # 打印输出信息，指定 LORA wv 张量的秩，覆盖默认秩
    fprintf(stderr, "  --rank-wv N                LORA rank for wv tensor, overrides default rank.\n");
    # 打印输出信息，指定 LORA wo 张量的秩，覆盖默认秩
    fprintf(stderr, "  --rank-wo N                LORA rank for wo tensor, overrides default rank.\n");
    # 打印输出信息，指定 LORA w1 张量的秩，覆盖默认秩
    fprintf(stderr, "  --rank-w1 N                LORA rank for w1 tensor, overrides default rank.\n");
    # 打印输出信息，指定 LORA w2 张量的秩，覆盖默认秩
    fprintf(stderr, "  --rank-w2 N                LORA rank for w2 tensor, overrides default rank.\n");
    # 打印输出信息，指定 LORA w3 张量的秩，覆盖默认秩
    fprintf(stderr, "  --rank-w3 N                LORA rank for w3 tensor, overrides default rank.\n");

    # 打印通用的训练使用信息
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

    // 如果支持 GPU 加速
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
            // 设置 GPU 层数
            params->common.n_gpu_layers = std::stoi(argv[i]);
#else
            // 如果没有编译 GPU 加速支持，则打印警告信息
            fprintf(stderr, "warning: not compiled with GPU offload support, --n-gpu-layers option will be ignored\n");
            fprintf(stderr, "warning: see main README.md for information on enabling GPU BLAS support\n");
#endif
        } else {
            // 如果参数无效，打印错误信息并退出
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            train_print_usage(argc, argv, &default_params);
            exit(1);
        }
    }
    // 如果存在无效参数，打印错误信息并退出
    if (invalid_param) {
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        train_print_usage(argc, argv, &default_params);
        exit(1);
    }
    // 完成训练参数处理
    finish_processing_train_args(&params->common);
    return true;
}

// 保存训练文件的数据结构
struct save_train_files_data {
    const char            * fn_checkpoint_out;
    const char            * fn_lora_out;
    const char            * pattern_fn_it;
    const char            * fn_latest;
    struct my_llama_model * model;
    struct my_llama_lora  * lora;
};

// 保存训练文件
static void save_train_files(void * vdata, struct train_state * train) {
    struct save_train_files_data * data   = (struct save_train_files_data *) vdata;

    int64_t iter = train->opt->iter;

    // 如果检查点输出文件名长度大于 0
    if (strlen(data->fn_checkpoint_out) > 0) {
        // 保存检查点 LORA 文件
        save_checkpoint_lora_file(get_train_filename(data->fn_checkpoint_out, data->pattern_fn_it, data->fn_latest, iter).c_str(), data->model, data->lora, train);
        save_checkpoint_lora_file(get_train_filename(data->fn_checkpoint_out, data->pattern_fn_it, data->fn_latest, -1  ).c_str(), data->model, data->lora, train);
    }
    # 如果 data->fn_lora_out 的长度大于 0
    if (strlen(data->fn_lora_out) > 0) {
        # 将 data->fn_lora_out、data->pattern_fn_it、data->fn_latest 和 iter 传入函数，保存为 llama lora 文件
        save_as_llama_lora(get_train_filename(data->fn_lora_out, data->pattern_fn_it, data->fn_latest, iter).c_str(), data->lora);
        # 将 data->fn_lora_out、data->pattern_fn_it、data->fn_latest 和 -1 传入函数，保存为 llama lora 文件
        save_as_llama_lora(get_train_filename(data->fn_lora_out, data->pattern_fn_it, data->fn_latest, -1  ).c_str(), data->lora);
    }
// 计算参数数量的函数
static int64_t get_parameter_count(struct my_llama_lora* lora) {
    // 初始化参数数量为 0
    int64_t nx = 0;
    // 计算并累加各个成员变量的参数数量
    nx += ggml_nelements(lora->tok_embeddings_a);
    nx += ggml_nelements(lora->tok_embeddings_b);
    nx += ggml_nelements(lora->norm_a);
    nx += ggml_nelements(lora->norm_b);
    nx += ggml_nelements(lora->output_a);
    nx += ggml_nelements(lora->output_b);

    // 遍历神经网络的各层，计算并累加各层的参数数量
    for (uint32_t i = 0; i < lora->layers.size(); ++i) {
        auto & layer = lora->layers[i];
        nx += ggml_nelements(layer.attention_norm_a);
        nx += ggml_nelements(layer.attention_norm_b);
        nx += ggml_nelements(layer.wq_a);
        nx += ggml_nelements(layer.wq_b);
        nx += ggml_nelements(layer.wk_a);
        nx += ggml_nelements(layer.wk_b);
        nx += ggml_nelements(layer.wv_a);
        nx += ggml_nelements(layer.wv_b);
        nx += ggml_nelements(layer.wo_a);
        nx += ggml_nelements(layer.wo_b);
        nx += ggml_nelements(layer.ffn_norm_a);
        nx += ggml_nelements(layer.ffn_norm_b);
        nx += ggml_nelements(layer.w1_a);
        nx += ggml_nelements(layer.w1_b);
        nx += ggml_nelements(layer.w2_a);
        nx += ggml_nelements(layer.w2_b);
        nx += ggml_nelements(layer.w3_a);
        nx += ggml_nelements(layer.w3_b);
    }
    // 返回总参数数量
    return nx;
}

// 主函数
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
    // 打印随机种子
    printf("%s: seed: %u\n", __func__, params.common.seed);
    // 设置随机种子
    srand(params.common.seed);

    // 获取默认的 Llama 模型参数
    struct llama_model_params llama_mparams = llama_model_default_params();
    // 设置 GPU 层数和是否仅包含词汇
    llama_mparams.n_gpu_layers = params.common.n_gpu_layers;
    llama_mparams.vocab_only = false;

    // 打印模型基础路径
    printf("%s: model base = '%s'\n", __func__, params.fn_model_base);
    // 从文件加载 Llama 模型
    struct llama_model * lmodel = llama_load_model_from_file(params.fn_model_base, llama_mparams);
}
    # 创建 llama_context_params 结构体实例，并使用默认参数初始化
    struct llama_context_params llama_cparams = llama_context_default_params();
    # 使用给定的模型和参数创建 llama_context 结构体实例
    struct llama_context * lctx = llama_new_context_with_model(lmodel, llama_cparams);

    # 创建 my_llama_model 结构体实例，并初始化
    struct my_llama_model model;
    init_model(lmodel, &model, params.fn_model_base, params.common.n_ctx);

    # 创建 my_llama_lora 结构体实例
    struct my_llama_lora lora;

    # 初始化训练状态结构体实例
    struct train_state      * train = init_train_state();
    # 获取训练状态结构体实例的优化上下文
    struct ggml_opt_context * opt   = train->opt;

    # 从命令行设置参数
    if (params.custom_f_norm_rms_eps) {
        # 如果自定义了 f_norm_rms_eps 参数，则设置模型的 f_norm_rms_eps 参数
        model.hparams.f_norm_rms_eps  = params.f_norm_rms_eps;
    }
    if (params.custom_rope_freq_base) {
        # 如果自定义了 rope_freq_base 参数，则设置模型的 rope_freq_base 参数
        model.hparams.rope_freq_base  = params.rope_freq_base;
    }
    if (params.custom_rope_freq_scale) {
        # 如果自定义了 rope_freq_scale 参数，则设置模型的 rope_freq_scale 参数
        model.hparams.rope_freq_scale = params.rope_freq_scale;
    }
    # 设置 lora 结构体实例的 lora_r 参数
    lora.hparams.lora_r                = params.lora_r;
    # 设置 lora 结构体实例的 lora_alpha 参数，如果自定义了 lora_alpha 参数，则使用自定义值，否则使用 lora_r 参数
    lora.hparams.lora_alpha            = params.custom_lora_alpha            ? params.lora_alpha            : params.lora_r;
    # 设置 n_rank_attention_norm 参数，如果自定义了 n_rank_attention_norm 参数，则使用自定义值，否则使用默认值 1
    uint32_t n_rank_attention_norm     = params.custom_n_rank_attention_norm ? params.n_rank_attention_norm : 1;
    # 设置 n_rank_wq 参数，如果自定义了 n_rank_wq 参数，则使用自定义值，否则使用 lora_r 参数
    uint32_t n_rank_wq                 = params.custom_n_rank_wq             ? params.n_rank_wq             : params.lora_r;
    # 设置 n_rank_wk 参数，如果自定义了 n_rank_wk 参数，则使用自定义值，否则使用 lora_r 参数
    uint32_t n_rank_wk                 = params.custom_n_rank_wk             ? params.n_rank_wk             : params.lora_r;
    # 设置 n_rank_wv 参数，如果自定义了 n_rank_wv 参数，则使用自定义值，否则使用 lora_r 参数
    uint32_t n_rank_wv                 = params.custom_n_rank_wv             ? params.n_rank_wv             : params.lora_r;
    # 设置 n_rank_wo 参数，如果自定义了 n_rank_wo 参数，则使用自定义值，否则使用 lora_r 参数
    uint32_t n_rank_wo                 = params.custom_n_rank_wo             ? params.n_rank_wo             : params.lora_r;
    # 设置 n_rank_ffn_norm 参数，如果自定义了 n_rank_ffn_norm 参数，则使用自定义值，否则使用默认值 1
    uint32_t n_rank_ffn_norm           = params.custom_n_rank_ffn_norm       ? params.n_rank_ffn_norm       : 1;
    # 设置 n_rank_w1 参数，如果自定义了 n_rank_w1 参数，则使用自定义值，否则使用 lora_r 参数
    uint32_t n_rank_w1                 = params.custom_n_rank_w1             ? params.n_rank_w1             : params.lora_r;
    # 设置 n_rank_w2 参数，如果自定义了 n_rank_w2 参数，则使用自定义值，否则使用 lora_r 参数
    uint32_t n_rank_w2                 = params.custom_n_rank_w2             ? params.n_rank_w2             : params.lora_r;
    // 如果自定义了 n_rank_w3 参数，则使用自定义值，否则使用默认值 params.lora_r
    uint32_t n_rank_w3                 = params.custom_n_rank_w3                 ? params.n_rank_w3                 : params.lora_r;
    // 如果自定义了 n_rank_tok_embeddings 参数，则使用自定义值，否则使用默认值 params.lora_r
    uint32_t n_rank_tok_embeddings     = params.custom_n_rank_tok_embeddings     ? params.n_rank_tok_embeddings     : params.lora_r;
    // 如果自定义了 n_rank_norm 参数，则使用自定义值，否则使用默认值 1
    uint32_t n_rank_norm               = params.custom_n_rank_norm               ? params.n_rank_norm               : 1;
    // 如果自定义了 n_rank_output 参数，则使用自定义值，否则使用默认值 params.lora_r
    uint32_t n_rank_output             = params.custom_n_rank_output             ? params.n_rank_output             : params.lora_r;
    // 设置 lora.hparams 中的 n_rank_attention_norm 参数
    lora.hparams.n_rank_attention_norm = n_rank_attention_norm;
    // 设置 lora.hparams 中的 n_rank_wq 参数
    lora.hparams.n_rank_wq             = n_rank_wq;
    // 设置 lora.hparams 中的 n_rank_wk 参数
    lora.hparams.n_rank_wk             = n_rank_wk;
    // 设置 lora.hparams 中的 n_rank_wv 参数
    lora.hparams.n_rank_wv             = n_rank_wv;
    // 设置 lora.hparams 中的 n_rank_wo 参数
    lora.hparams.n_rank_wo             = n_rank_wo;
    // 设置 lora.hparams 中的 n_rank_ffn_norm 参数
    lora.hparams.n_rank_ffn_norm       = n_rank_ffn_norm;
    // 设置 lora.hparams 中的 n_rank_w1 参数
    lora.hparams.n_rank_w1             = n_rank_w1;
    // 设置 lora.hparams 中的 n_rank_w2 参数
    lora.hparams.n_rank_w2             = n_rank_w2;
    // 设置 lora.hparams 中的 n_rank_w3 参数
    lora.hparams.n_rank_w3             = n_rank_w3;
    // 设置 lora.hparams 中的 n_rank_tok_embeddings 参数
    lora.hparams.n_rank_tok_embeddings = n_rank_tok_embeddings;
    // 设置 lora.hparams 中的 n_rank_norm 参数
    lora.hparams.n_rank_norm           = n_rank_norm;
    // 设置 lora.hparams 中的 n_rank_output 参数
    lora.hparams.n_rank_output         = n_rank_output;

    // 从命令行设置 opt 参数
    opt->params = ggml_opt_default_params(GGML_OPT_ADAM);
    // 设置是否打印前向图
    opt->params.print_forward_graph     = false;
    // 设置是否打印反向图
    opt->params.print_backward_graph    = false;
    // 设置图的大小
    opt->params.graph_size              = LLAMA_TRAIN_MAX_NODES;
    // 设置线程数
    opt->params.n_threads               = params.common.n_threads;
    // 设置过去的时间
    opt->params.past                    = params.common.opt_past;
    // 设置增量
    opt->params.delta                   = params.common.opt_delta;
    // 设置最大不改进次数
    opt->params.max_no_improvement      = params.common.opt_max_no_improvement;
    // 设置梯度累积数
    opt->params.n_gradient_accumulation = params.common.n_gradient_accumulation;
    // 设置 adam 迭代次数
    opt->params.adam.n_iter             = params.common.adam_n_iter;
    // 设置 adam 调度
    opt->params.adam.sched              = 1.0f;
    // 设置 adam 学习率
    opt->params.adam.alpha              = params.common.adam_alpha;
    # 设置opt结构体中adam参数的值
    opt->params.adam.decay              = params.common.adam_decay;
    opt->params.adam.decay_min_ndim     = params.common.adam_decay_min_ndim;
    opt->params.adam.beta1              = params.common.adam_beta1;
    opt->params.adam.beta2              = params.common.adam_beta2;
    opt->params.adam.gclip              = params.common.adam_gclip;
    opt->params.adam.eps_f              = params.common.adam_eps_f;

    # 分配内存给alloc指针
    ggml_allocr * alloc = NULL;

    # 打印初始化模型的信息
    printf("%s: init model\n", __func__);
    # 载入检查点文件，将模型和lora参数加载到内存中
    bool existed = load_checkpoint_lora_file(params.common.fn_checkpoint_in, &model, &lora, train);
    // 如果存在上一个模型
    if (existed) {
        // 用用户提供的 n_ctx 覆盖上一个 n_ctx
        if (params.common.custom_n_ctx) {
            model.hparams.n_ctx = params.common.n_ctx;
        }

        // 检查参数数量是否发生变化
        const bool opt_param_count_changed = (
           (lora.hparams.n_rank_attention_norm != n_rank_attention_norm)
        || (lora.hparams.n_rank_wq             != n_rank_wq)
        || (lora.hparams.n_rank_wk             != n_rank_wk)
        || (lora.hparams.n_rank_wv             != n_rank_wv)
        || (lora.hparams.n_rank_wo             != n_rank_wo)
        || (lora.hparams.n_rank_ffn_norm       != n_rank_ffn_norm)
        || (lora.hparams.n_rank_w1             != n_rank_w1)
        || (lora.hparams.n_rank_w2             != n_rank_w2)
        || (lora.hparams.n_rank_w3             != n_rank_w3)
        || (lora.hparams.n_rank_tok_embeddings != n_rank_tok_embeddings)
        || (lora.hparams.n_rank_norm           != n_rank_norm)
        || (lora.hparams.n_rank_output         != n_rank_output)
        );

        // 检查过去是否发生变化
        const bool opt_past_changed = opt->params.past != params.common.opt_past;

        // 如果参数数量发生变化
        if (opt_param_count_changed) {
            // 打印当前参数并终止程序
            print_lora_params(&lora.hparams);
            die("Provided rank differs from checkpoint file. To use different rank start finetune from scratch with empty input checkpoint, e.g --checkpoint-in ''. Aborting.");
            // 需要丢弃之前的优化器梯度统计，并使用新的形状进行优化器初始化
            // 待办事项
        }
        // 如果过去发生变化
        if (opt_past_changed) {
            // 终止程序
            die("Optimizer parameter '--opt-past N' differs from checkpoint file. To use different value finetune from scratch with empty input checkpoint, e.g --checkpoint-in ''. Aborting");
            // 需要丢弃之前的优化器过去函数值统计，并使用新的形状进行优化器初始化
            // 待办事项
        }
    } else { // 如果 existed == false
        // 初始化 Lora 模型和 Lora 对象
        init_lora(&model, &lora);
        // 为 Lora 对象随机生成参数
        randomize_lora(&lora, params.common.seed, 0.0f, 1.0f, -1.0f, +1.0f);
        // 如果不仅仅是写入 Lora，则初始化 ggml_opt
        if (!params.only_write_lora) {
            ggml_opt_init(opt->ctx, opt, opt->params, get_parameter_count(&lora));
        }
    }
    // 设置 opt 的迭代次数为 train 的训练次数
    opt->iter = train->train_its;

    // 打印模型参数
    print_params(&model.hparams);
    // 打印 Lora 参数
    print_lora_params(&lora.hparams);
    // 打印训练迭代次数
    printf("%s: total train_iterations %llu\n", __func__, (long long unsigned) train->train_its);
    // 打印已见训练样本数
    printf("%s: seen train_samples     %llu\n", __func__, (long long unsigned) train->train_samples);
    // 打印已见训练标记数
    printf("%s: seen train_tokens      %llu\n", __func__, (long long unsigned) train->train_tokens);
    // 打印已完成训练周期数
    printf("%s: completed train_epochs %llu\n", __func__, (long long unsigned) train->train_epochs);
    // 打印 Lora 大小
    printf("%s: lora_size = %zu bytes (%.1f MB)\n", __func__, (ggml_used_mem(lora.ctx) + lora.data.size()), (float) (ggml_used_mem(lora.ctx) + lora.data.size()) / (1024.0f*1024.0f));

    // 如果只是写入 Lora，则保存训练文件数据并释放资源后返回
    if (params.only_write_lora) {
        save_train_files_data save_data;
        save_data.fn_checkpoint_out = "";
        save_data.fn_lora_out       = params.fn_lora_out;
        save_data.pattern_fn_it     = params.common.pattern_fn_it;
        save_data.fn_latest         = params.common.fn_latest;
        save_data.model             = &model;
        save_data.lora              = &lora;

        save_train_files(&save_data, train);

        free_train_state(train);
        ggml_free(lora.ctx);
        llama_free(lctx);
        llama_free_model(lmodel);
        return 0;
    }

    // 打印 opt 大小
    printf("%s: opt_size  = %zu bytes (%.1f MB)\n", __func__, ggml_get_mem_size(opt->ctx), (float) ggml_get_mem_size(opt->ctx) / (1024.0f*1024.0f));
    // 打印 opt 迭代次数
    printf("%s: opt iter %d\n", __func__, opt->iter);

    // 初始化 n_tokens 为模型参数中的 n_ctx
    int n_tokens = model.hparams.n_ctx;
    // 初始化 n_vocab 为模型参数中的 n_vocab
    int n_vocab  = model.hparams.n_vocab;
    // 初始化 n_batch 为公共参数中的 n_batch
    int n_batch  = params.common.n_batch;

    // 初始化 mem_input_data 为 uint8_t 类型的向量
    std::vector<uint8_t> mem_input_data;
    // 初始化 mem_compute_data 为 uint8_t 类型的向量
    std::vector<uint8_t> mem_compute_data;
    // 定义输入张量的上下文，不包括数据
    struct ggml_init_params ctx_input_params = {
        ggml_tensor_overhead() * 2, // mem_size
        NULL,                       // mem_buffer
        true,                       // no_alloc
    };
    // 初始化输入张量的上下文
    struct ggml_context * ctx_input = ggml_init(ctx_input_params);

    // 创建输入张量
    struct ggml_tensor * tokens_input  = ggml_new_tensor_2d(ctx_input, GGML_TYPE_I32, n_tokens, n_batch);
    struct ggml_tensor * target_probs  = ggml_new_tensor_3d(ctx_input, GGML_TYPE_F32, n_vocab,  n_tokens, n_batch);

    // 计算输入张量所需的最大内存
    size_t max_input_size = GGML_PAD(ggml_nbytes(tokens_input), tensor_alignment) +
                            GGML_PAD(ggml_nbytes(target_probs), tensor_alignment) +
                            tensor_alignment;
    printf("%s: input_size = %zu bytes (%.1f MB)\n", __func__, max_input_size, (float) max_input_size / (1024.0f*1024.0f));

    // 分配输入张量的内存
    mem_input_data.resize(max_input_size);
    alloc = ggml_allocr_new(mem_input_data.data(), mem_input_data.size(), tensor_alignment);
    ggml_allocr_alloc(alloc, tokens_input);
    ggml_allocr_alloc(alloc, target_probs);
    ggml_allocr_free(alloc);

    // 定义计算张量的上下文，不包括数据
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
    // 声明一个指向 ggml_cgraph 结构体的指针，并初始化为 NULL
    struct ggml_cgraph * gb_tmp = NULL;

    // 用于存储计算张量所需的内存大小
    size_t best_compute_size = SIZE_MAX;
    // 用于存储最佳的评估顺序
    enum ggml_cgraph_eval_order best_order = GGML_CGRAPH_EVAL_ORDER_COUNT;
    // 寻找最佳的评估顺序
    for (unsigned order = 0; order < (unsigned) GGML_CGRAPH_EVAL_ORDER_COUNT; ++order) {
        // 初始化计算上下文
        ctx_compute = ggml_init(ctx_compute_params);
        // 创建一个用于测量的分配器
        alloc = ggml_allocr_new_measure(tensor_alignment);
        // 创建一个自定义的图形对象 gf
        gf = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
        // 设置 gf 的评估顺序
        gf->order = (enum ggml_cgraph_eval_order) order;
        // 创建一个自定义的图形对象 gb
        gb = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
        // 如果参数中使用了检查点，则创建一个临时的自定义图形对象 gb_tmp
        gb_tmp = params.common.use_checkpointing
            ? ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true)
            : NULL;
        // 构建 LORA 微调图形
        loss = llama_build_lora_finetune_graphs(
            &model, &lora, alloc, ctx_compute,
            gf, gb, gb_tmp,
            &logits, tokens_input, target_probs,
            n_tokens, n_batch,
            params.common.use_flash,
            params.common.use_checkpointing
        );
        // 计算最大计算大小
        size_t max_compute_size = ggml_allocr_max_size(alloc) + tensor_alignment;
        // 如果最大计算大小小于当前最佳计算大小，则更新最佳计算大小和最佳评估顺序
        if (max_compute_size < best_compute_size) {
            best_compute_size = max_compute_size;
            best_order = gf->order;
        }
        // 释放分配器
        ggml_allocr_free(alloc);
        // 释放计算上下文
        ggml_free(ctx_compute);
    }
    // 将最佳计算大小存储到 max_compute_size 中
    size_t max_compute_size = best_compute_size;
    // 打印计算大小
    printf("%s: compute_size = %zu bytes (%.1f MB)\n", __func__, max_compute_size, (float) max_compute_size / (1024.0f*1024.0f));
    // 打印评估顺序
    printf("%s: evaluation order = %s\n", __func__,
        (best_order == GGML_CGRAPH_EVAL_ORDER_LEFT_TO_RIGHT) ? "LEFT_TO_RIGHT" :
        (best_order == GGML_CGRAPH_EVAL_ORDER_RIGHT_TO_LEFT) ? "RIGHT_TO_LEFT" :
        "invalid");

    // 调整计算张量的内存大小
    mem_compute_data.resize(max_compute_size);
    // 初始化计算上下文
    ctx_compute = ggml_init(ctx_compute_params);
    // 使用 ggml_allocr_new 函数为计算数据分配内存，并返回分配的指针
    alloc = ggml_allocr_new(mem_compute_data.data(), mem_compute_data.size(), tensor_alignment);
    // 使用 ggml_new_graph_custom 函数创建一个新的图形对象，并设置最大节点数和是否使用默认顺序
    gf = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
    // 设置 gf 对象的顺序为最佳顺序
    gf->order = best_order;
    // 使用 ggml_new_graph_custom 函数创建一个新的图形对象，并设置最大节点数和是否使用默认顺序
    gb = ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true);
    // 如果 params.common.use_checkpointing 为真，则创建一个新的图形对象，否则为 NULL
    gb_tmp = params.common.use_checkpointing
        ? ggml_new_graph_custom(ctx_compute, LLAMA_TRAIN_MAX_NODES, true)
        : NULL;
    // 使用 llama_build_lora_finetune_graphs 函数构建 LORA 微调图形，并返回损失值
    loss = llama_build_lora_finetune_graphs(
        &model, &lora, alloc, ctx_compute,
        gf, gb, gb_tmp,
        &logits, tokens_input, target_probs,
        n_tokens, n_batch,
        params.common.use_flash,
        params.common.use_checkpointing
    );
    // 释放分配的内存
    ggml_allocr_free(alloc);

    // 对数据进行标记化
    // 创建存储标记化数据的向量
    std::vector<llama_token> train_tokens;
    // 创建存储训练样本起始位置的向量
    std::vector<size_t> train_samples_begin;
    // 创建存储训练样本大小的向量
    std::vector<size_t> train_samples_size;
    // 打印标记化训练数据的信息
    printf("%s: tokenize training data\n", __func__);
    // 调用 tokenize_file 函数对训练数据进行标记化
    tokenize_file(lctx,
            params.common.fn_train_data,
            params.common.sample_start,
            params.common.include_sample_start,
            params.common.overlapping_samples,
            n_tokens,
            train_tokens,
            train_samples_begin,
            train_samples_size);
    // 检查训练样本起始位置和大小的向量是否大小相等
    GGML_ASSERT(train_samples_begin.size() == train_samples_size.size());

    // 打印训练数据中标记化的标记数量
    printf("%s: number of training tokens: %zu\n", __func__, train_tokens.size());

    // 创建存储标记出现次数的向量，并初始化为 0
    std::vector<size_t> token_noccurs;
    token_noccurs.resize(model.hparams.n_vocab, 0);
    // 遍历训练标记并统计每个标记出现的次数
    for (unsigned int i = 0; i < train_tokens.size(); ++i) {
        ++token_noccurs[train_tokens[i]];
    }
    // 统计出现过的唯一标记的数量
    int n_unique_tokens = 0;
    for (unsigned int i = 0; i < token_noccurs.size(); ++i) {
        if (token_noccurs[i] == 0) continue;
        ++n_unique_tokens;
    }
    // 打印唯一标记的数量
    printf("%s: number of unique tokens: %d\n", __func__, n_unique_tokens);

    // 计算训练样本的哈希值
    size_t shuffle_samples_hash = compute_samples_hash(params.common.fn_train_data, train_samples_begin.data(), train_samples_size.data(), train_samples_size.size());
    // 检查训练数据是否发生变化，如果发生变化则重新开始洗牌的周期
    const bool changed_train_data = (shuffle_samples_hash != train->shuffle_samples_hash) || (train->shuffle_sample_count != train_samples_size.size());
    if (changed_train_data) {
        // 如果训练数据似乎已经发生变化，则打印提示信息
        printf("%s: train data seems to have changed. restarting shuffled epoch.\n", __func__);
    }
    // 如果强制重新洗牌参数为真，则打印提示信息
    if (params.common.force_reshuffle) {
        printf("%s: forced reshuffling of data. restarting with newly shuffled epoch.\n", __func__);
    }
    // 如果当前的洗牌状态为空，或者训练数据发生变化，或者强制重新洗牌参数为真，则执行以下操作
    if ((train->shuffle_rng_state_current == "") || changed_train_data || params.common.force_reshuffle) {
        // 根据随机种子生成当前的洗牌状态
        train->shuffle_rng_state_current = mt19937_seed_to_state(params.common.seed);
        // 设置洗牌样本数量为训练样本大小
        train->shuffle_sample_count = train_samples_size.size();
        // 设置下一个洗牌样本的索引为0
        train->shuffle_next_sample = 0;
        // 更新洗牌样本的哈希值
        train->shuffle_samples_hash = shuffle_samples_hash;
    }
    // 初始化三个存储洗牌样本偏移、起始和大小的向量
    std::vector<size_t> train_shuffled_samples_offs;
    std::vector<size_t> train_shuffled_samples_begin;
    std::vector<size_t> train_shuffled_samples_size;
    // 调整向量的大小为训练样本起始位置和大小的大小
    train_shuffled_samples_offs.resize(train_samples_begin.size());
    train_shuffled_samples_begin.resize(train_samples_begin.size());
    train_shuffled_samples_size.resize(train_samples_size.size());
    // 生成下一个洗牌状态，并更新洗牌样本的偏移、起始和大小
    train->shuffle_rng_state_next = shuffle_samples(
        train->shuffle_rng_state_current,
        train_shuffled_samples_offs.data(),
        train_shuffled_samples_begin.data(),
        train_shuffled_samples_size.data(),
        train_samples_begin.data(),
        train_samples_size.data(),
        train_samples_size.size());

    // 打印开始训练的提示信息
    printf("%s: begin training\n", __func__);

    // 初始化保存训练文件数据的结构体
    save_train_files_data save_data;
    save_data.fn_checkpoint_out = params.common.fn_checkpoint_out;
    save_data.fn_lora_out       = params.fn_lora_out;
    save_data.pattern_fn_it     = params.common.pattern_fn_it;
    save_data.fn_latest         = params.common.fn_latest;
    save_data.model             = &model;
    save_data.lora              = &lora;

    // 初始化训练选项回调数据的结构体
    struct train_opt_callback_data opt_cb_data;
    // 设置参数为公共参数
    opt_cb_data.params = &params.common;
    // 设置训练数据
    opt_cb_data.train = train;
    // 设置保存回调函数
    opt_cb_data.save_cb = &save_train_files;
    // 设置保存数据
    opt_cb_data.save_data = &save_data;
    // 设置上下文
    opt_cb_data.lctx = lctx;
    // 设置最后保存的迭代次数
    opt_cb_data.last_save_iter = opt->iter;
    // 设置训练数据的 tokens 数据
    opt_cb_data.tokens_data = train_tokens.data();
    // 设置训练数据的 tokens 大小
    opt_cb_data.tokens_size = train_tokens.size();
    // 设置训练数据的样本开始位置
    opt_cb_data.samples_begin = train_samples_begin.data();
    // 设置训练数据的样本大小
    opt_cb_data.samples_size = train_samples_size.data();
    // 设置训练数据的打乱样本偏移
    opt_cb_data.shuffled_samples_offs = train_shuffled_samples_offs.data();
    // 设置训练数据的打乱样本开始位置
    opt_cb_data.shuffled_samples_begin = train_shuffled_samples_begin.data();
    // 设置训练数据的打乱样本大小
    opt_cb_data.shuffled_samples_size = train_shuffled_samples_size.data();
    // 设置训练数据的样本数量
    opt_cb_data.samples_count = train_samples_size.size();
    // 设置 tokens 输入
    opt_cb_data.tokens_input = tokens_input;
    // 设置目标概率
    opt_cb_data.target_probs = target_probs;
    // 设置第一次迭代
    opt_cb_data.first_iter = opt->iter;
    // 设置第一次训练周期
    opt_cb_data.first_epoch = train->train_epochs;
    // 设置最后一个周期的迭代次数
    opt_cb_data.iter_at_last_epoch = -1;
    // 设置最后时间
    opt_cb_data.last_time = ggml_time_ms();
    // 设置每次迭代的毫秒数
    opt_cb_data.millis_per_iter = 0.0;

    // 测量工作缓冲区所需的内存
    size_t max_work_size = ggml_graph_plan(gb, params.common.n_threads).work_size + GGML_OBJECT_SIZE;
    printf("%s: work_size = %zu bytes (%.1f MB)\n", __func__, max_work_size, (float) max_work_size / (1024.0f*1024.0f));

    // 设置工作缓冲区的上下文
    struct ggml_init_params ctx_work_params = {
        max_work_size, // 内存大小
        NULL,          // 内存缓冲区
        false,         // 不分配
    };
    // 初始化工作上下文
    struct ggml_context * ctx_work = ggml_init(ctx_work_params);

    // 获取当前时间
    int64_t t0 = ggml_time_ms();

    // 恢复训练
    ggml_opt_resume_g(ctx_work, opt, loss, gf, gb, &train_opt_callback, (void *) &opt_cb_data);

    // 释放工作上下文
    ggml_free(ctx_work);
    // 释放计算上下文
    ggml_free(ctx_compute);
    // 释放输入上下文
    ggml_free(ctx_input);
    # 获取当前时间戳，单位为毫秒
    int64_t t1 = ggml_time_ms();
    # 打印函数名和总训练时间
    printf("%s: total training time: ", __func__);
    # 打印训练时间的持续时间
    print_duration((double) (t1 - t0));
    # 换行
    printf("\n");

    # 计算新的迭代次数
    int new_iters = opt->iter - opt_cb_data.last_save_iter;
    # 如果新的迭代次数大于0
    if (new_iters > 0) {
        # 更新训练总次数
        train->train_its     += new_iters;
        # 更新训练总标记数
        train->train_tokens  += new_iters * opt->params.n_gradient_accumulation * n_batch * n_tokens;

        # 保存训练文件
        save_train_files(&save_data, train);
        # 更新最后保存的迭代次数
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
    # 返回0表示成功
    return 0;
# 闭合前面的函数定义
```