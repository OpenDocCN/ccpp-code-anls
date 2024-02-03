# `PowerInfer\examples\convert-llama2c-to-ggml\convert-llama2c-to-ggml.cpp`

```cpp
// 引入所需的头文件
#include "ggml.h"
#include "llama.h"
#include "common.h"

// 引入所需的标准库头文件
#include <unordered_map>
#include <vector>
#include <cassert>
#include <climits>
#include <cstring>
#include <cstdarg>
#include <ctime>
#include <random>
#include <stdexcept>
#include <sstream>
#include <algorithm>
#include <string>

// 定义 GGUF 键和张量名称

#define KV_GENERAL_ARCHITECTURE          "general.architecture"
#define KV_GENERAL_NAME                  "general.name"

#define KV_TOKENIZER_MODEL               "tokenizer.ggml.model"
#define KV_TOKENIZER_LIST                "tokenizer.ggml.tokens"
#define KV_TOKENIZER_TOKEN_TYPE          "tokenizer.ggml.token_type"
#define KV_TOKENIZER_SCORES              "tokenizer.ggml.scores"
#define KV_TOKENIZER_BOS_ID              "tokenizer.ggml.bos_token_id"
#define KV_TOKENIZER_EOS_ID              "tokenizer.ggml.eos_token_id"
#define KV_TOKENIZER_UNK_ID              "tokenizer.ggml.unknown_token_id"
#define KV_TOKENIZER_SEP_ID              "tokenizer.ggml.seperator_token_id"
#define KV_TOKENIZER_PAD_ID              "tokenizer.ggml.padding_token_id"
#define KV_TOKENIZER_HF_JSON             "tokenizer.huggingface.json"

#define KV_CONTEXT_LENGTH                "llama.context_length"
#define KV_EMBEDDING_LENGTH              "llama.embedding_length"
#define KV_BLOCK_COUNT                   "llama.block_count"
#define KV_FEED_FORWARD_LENGTH           "llama.feed_forward_length"
#define KV_ATTENTION_HEAD_COUNT          "llama.attention.head_count"
#define KV_ATTENTION_HEAD_COUNT_KV       "llama.attention.head_count_kv"
#define KV_ATTENTION_LAYERNORM_RMS_EPS   "llama.attention.layer_norm_rms_epsilon"
#define KV_ROPE_DIMENSION_COUNT          "llama.rope.dimension_count"

#define TN_TOKEN_EMBD  "token_embd.weight"
#define TN_OUTPUT_NORM "output_norm.weight"
#define TN_OUTPUT      "output.weight"
#define TN_ATTN_NORM   "blk.%d.attn_norm.weight"
#define TN_ATTN_Q      "blk.%d.attn_q.weight"
#define TN_ATTN_K      "blk.%d.attn_k.weight"
// 定义注意力机制的权重名称模板
#define TN_ATTN_V      "blk.%d.attn_v.weight"
#define TN_ATTN_OUTPUT "blk.%d.attn_output.weight"
#define TN_FFN_NORM    "blk.%d.ffn_norm.weight"
#define TN_FFN_GATE    "blk.%d.ffn_gate.weight"
#define TN_FFN_DOWN    "blk.%d.ffn_down.weight"
#define TN_FFN_UP      "blk.%d.ffn_up.weight"

// 如果是 MSC 编译器，禁用警告 4244 和 4267，可能会丢失数据
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义 LLAMA 文件的魔术数字和版本号
#define LLAMA_FILE_MAGIC_GGJT        0x67676a74u // 'ggjt'
#define LLAMA_FILE_VERSION_GGJT_V3   3

// 定义分词器的名称和未知标记的标识符
#define TOKENIZER_NAME "llama"
#define UNKNOWN_TOKEN_ID 0
#define BOS_TOKEN_ID 1
#define EOS_TOKEN_ID 2

// 定义配置结构体
typedef struct {
    int dim; // transformer dimension
    int hidden_dim; // for ffn layers
    int n_layers; // number of layers
    int n_heads; // number of query heads
    int n_kv_heads; // number of key/value heads (can be < query heads because of multiquery)
    int vocab_size; // vocabulary size, usually 256 (byte-level)
    int seq_len; // max sequence length
} Config;

// 定义 TransformerWeights 结构体
struct TransformerWeights {
    // token embedding table
    float* token_embedding_table;    // (vocab_size, dim)
    // weights for rmsnorms
    float* rms_att_weight; // (layer, dim) rmsnorm weights
    float* rms_ffn_weight; // (layer, dim)
    // weights for matmuls
    float* wq; // (layer, dim, dim)
    float* wk; // (layer, dim, dim)
    float* wv; // (layer, dim, dim)
    float* wo; // (layer, dim, dim)
    // weights for ffn
    float* w1; // (layer, hidden_dim, dim)
    float* w2; // (layer, dim, hidden_dim)
    float* w3; // (layer, hidden_dim, dim)
    // final rmsnorm
    float* rms_final_weight; // (dim,)
    // freq_cis for RoPE relatively positional embeddings
    // float* freq_cis_real; // (seq_len, dim/2)
    // float* freq_cis_imag; // (seq_len, dim/2)
    // (optional) classifier weights for the logits, on the last layer
    float* wcls;
    # 定义析构函数，用于释放对象的内存空间
    ~TransformerWeights() {
        # 释放 token_embedding_table 数组的内存空间
        delete[] token_embedding_table;
        # 释放 rms_att_weight 数组的内存空间
        delete[] rms_att_weight;
        # 释放 rms_ffn_weight 数组的内存空间
        delete[] rms_ffn_weight;
        # 释放 wq 数组的内存空间
        delete[] wq;
        # 释放 wk 数组的内存空间
        delete[] wk;
        # 释放 wv 数组的内存空间
        delete[] wv;
        # 释放 wo 数组的内存空间
        delete[] wo;
        # 释放 w1 数组的内存空间
        delete[] w1;
        # 释放 w2 数组的内存空间
        delete[] w2;
        # 释放 w3 数组的内存空间
        delete[] w3;
        # 释放 rms_final_weight 数组的内存空间
        delete[] rms_final_weight;
        # 释放 wcls 数组的内存空间
        delete[] wcls;
    }
    // 为 TransformerWeights 结构体中的 token_embedding_table 成员分配内存空间，大小为 p->vocab_size * p->dim 个 float 类型的空间
    w->token_embedding_table = new float[p->vocab_size * p->dim]();
    // 打印分配的内存空间大小信息
    printf("[%s:AK] Allocating [%d] x [%d] = [%d] float space for w->token_embedding_table\n",__func__,p->vocab_size , p->dim, p->vocab_size * p->dim);

    // 为 TransformerWeights 结构体中的 rms_att_weight 成员分配内存空间，大小为 p->n_layers * p->dim 个 float 类型的空间
    w->rms_att_weight = new float[p->n_layers * p->dim]();
    // 打印分配的内存空间大小信息
    printf("[%s:AK] Allocating [%d] x [%d] = [%d] float space for w->rms_att_weight\n",__func__,p->n_layers, p->dim, p->n_layers * p->dim);

    // 为 TransformerWeights 结构体中的 rms_ffn_weight 成员分配内存空间，大小为 p->n_layers * p->dim 个 float 类型的空间
    w->rms_ffn_weight = new float[p->n_layers * p->dim]();
    // 打印分配的内存空间大小信息
    printf("[%s:AK] Allocating [%d] x [%d] = [%d] float space for w->rms_ffn_weight\n",__func__,p->n_layers , p->dim, p->n_layers * p->dim);

    // 为 TransformerWeights 结构体中的 wq 成员分配内存空间，大小为 p->n_layers * p->dim * p->dim 个 float 类型的空间
    w->wq = new float[p->n_layers * p->dim * p->dim]();
    // 打印分配的内存空间大小信息
    printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->wq\n",__func__,p->n_layers, p->dim, p->dim, p->n_layers * p->dim * p->dim);

    // 为 TransformerWeights 结构体中的 wk 成员分配内存空间，大小为 p->n_layers * p->dim * p->dim 个 float 类型的空间
    w->wk = new float[p->n_layers * p->dim * p->dim]();
    // 打印分配的内存空间大小信息
    printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->wk\n",__func__,p->n_layers, p->dim, p->dim, p->n_layers * p->dim * p->dim);

    // 为 TransformerWeights 结构体中的 wv 成员分配内存空间，大小为 p->n_layers * p->dim * p->dim 个 float 类型的空间
    w->wv = new float[p->n_layers * p->dim * p->dim]();
    // 打印分配的内存空间大小信息
    printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->wv\n",__func__, p->n_layers, p->dim, p->dim, p->n_layers * p->dim * p->dim);

    // 为 TransformerWeights 结构体中的 wo 成员分配内存空间，大小为 p->n_layers * p->dim * p->dim 个 float 类型的空间
    w->wo = new float[p->n_layers * p->dim * p->dim]();
    // 打印分配的内存空间大小信息
    printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->wo\n",__func__,p->n_layers, p->dim, p->dim, p->n_layers * p->dim * p->dim);

    // 为 TransformerWeights 结构体中的 w1 成员分配内存空间，大小为 p->n_layers * p->hidden_dim * p->dim 个 float 类型的空间
    w->w1 = new float[p->n_layers * p->hidden_dim * p->dim]();
    // 打印分配的内存空间大小信息
    printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->w1\n",__func__,p->n_layers, p->hidden_dim, p->dim, p->n_layers * p->hidden_dim * p->dim);

    // 为 TransformerWeights 结构体中的 w2 成员分配内存空间，大小为 p->n_layers * p->hidden_dim * p->dim 个 float 类型的空间
    w->w2 = new float[p->n_layers * p->hidden_dim * p->dim]();
    # 打印分配内存空间的信息，包括层数、维度和隐藏层维度的乘积
    printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->w2\n",__func__,p->n_layers, p->dim, p->hidden_dim, p->n_layers * p->hidden_dim * p->dim);

    # 为 w->w3 分配内存空间，并初始化为 0
    w->w3 = new float[p->n_layers * p->hidden_dim * p->dim]();
    # 打印分配内存空间的信息，包括层数、维度和隐藏层维度的乘积
    printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->w3\n",__func__,p->n_layers, p->hidden_dim, p->dim, p->n_layers * p->hidden_dim * p->dim);

    # 为 w->rms_final_weight 分配内存空间，并初始化为 0
    w->rms_final_weight = new float[p->dim]();
    # 打印分配内存空间的信息，包括维度
    printf("[%s:AK] Allocating [%d] float space for w->rms_final_weight\n",__func__,p->dim);

    # 如果共享权重，则将 w->wcls 设置为 NULL
    if (shared_weights) {
        w->wcls = NULL;
    } else {
        # 否则为 w->wcls 分配内存空间，并初始化为 0
        w->wcls = new float[p->vocab_size * p->dim]();
        # 打印分配内存空间的信息，包括词汇量大小和维度的乘积
        printf("[%s:AK] Allocating [%d] x [%d] = [%d] float space for w->wcls\n",__func__,p->vocab_size , p->dim, p->vocab_size * p->dim);
    }
// 初始化权重函数，用于从文件中读取权重数据并初始化到TransformerWeights结构体中
static int checkpoint_init_weights(TransformerWeights *w, Config* p, FILE* f, bool shared_weights) {
    // 从文件中读取token_embedding_table数据，并检查读取是否成功
    if (fread(w->token_embedding_table, sizeof(float), p->vocab_size * p->dim, f) != static_cast<size_t>(p->vocab_size * p->dim)) return 1;
    // 从文件中读取rms_att_weight数据，并检查读取是否成功
    if (fread(w->rms_att_weight, sizeof(float), p->n_layers * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim)) return 1;
    // 从文件中读取wq数据，并检查读取是否成功
    if (fread(w->wq, sizeof(float), p->n_layers * p->dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->dim)) return 1;
    // 从文件中读取wk数据，并检查读取是否成功
    if (fread(w->wk, sizeof(float), p->n_layers * p->dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->dim)) return 1;
    // 从文件中读取wv数据，并检查读取是否成功
    if (fread(w->wv, sizeof(float), p->n_layers * p->dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->dim)) return 1;
    // 从文件中读取wo数据，并检查读取是否成功
    if (fread(w->wo, sizeof(float), p->n_layers * p->dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->dim)) return 1;
    // 从文件中读取rms_ffn_weight数据，并检查读取是否成功
    if (fread(w->rms_ffn_weight, sizeof(float), p->n_layers * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim)) return 1;
    // 从文件中读取w1数据，并检查读取是否成功
    if (fread(w->w1, sizeof(float), p->n_layers * p->dim * p->hidden_dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->hidden_dim)) return 1;
    // 从文件中读取w2数据，并检查读取是否成功
    if (fread(w->w2, sizeof(float), p->n_layers * p->hidden_dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->hidden_dim * p->dim)) return 1;
    // 从文件中读取w3数据，并检查读取是否成功
    if (fread(w->w3, sizeof(float), p->n_layers * p->dim * p->hidden_dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->hidden_dim)) return 1;
    // 从文件中读取rms_final_weight数据，并检查读取是否成功
    if (fread(w->rms_final_weight, sizeof(float), p->dim, f) != static_cast<size_t>(p->dim)) return 1;

    // 跳过freq_cis_real & freq_cis_imag的读取
    int head_size = p->dim / p->n_heads;
    fseek(f, p->seq_len * head_size * sizeof(float), SEEK_CUR);

    // 如果不是共享权重，从文件中读取wcls数据，并检查读取是否成功
    if (!shared_weights && fread(w->wcls, sizeof(float), p->vocab_size * p->dim, f) != static_cast<size_t>(p->vocab_size * p->dim)) return 1;

    // 检查是否有遗漏未读取的数据
    auto curr = ftell(f);
    fseek(f, 0, SEEK_END);
    # 获取文件当前位置
    auto end = ftell(f);
    # 如果当前位置不等于文件末尾位置
    if (curr != end) {
        # 打印错误信息，显示当前位置和文件末尾位置
        printf("Error: failed to read the checkpoint file to the end (curr = %ld, end =  %ld)\n", curr, end);
        # 返回错误代码 1
        return 1;
    }
    # 返回成功代码 0
    return 0;
static void print_sample_weights(TransformerWeights *w){
    // 打印所有变量的第一个权重值的快速打印
    printf("----- Quick print of first of the weight vales of all the variables\n");
    // 打印 token_embedding_table 的第一个权重值
    printf("%f\n", w->token_embedding_table[0]);
    // 打印 rms_att_weight 的第一个权重值
    printf("%f\n", w->rms_att_weight[0]);
    // 打印 rms_ffn_weight 的第一个权重值
    printf("%f\n", w->rms_ffn_weight[0]);

    // 打印 wq 的第一个权重值
    printf("%f\n", w->wq[0]);
    // 打印 wk 的第一个权重值
    printf("%f\n", w->wk[0]);
    // 打印 wv 的第一个权重值
    printf("%f\n", w->wv[0]);
    // 打印 wo 的第一个权重值
    printf("%f\n", w->wo[0]);
    // 打印 w1 的第一个权重值
    printf("%f\n", w->w1[0]);
    // 打印 w2 的第一个权重值
    printf("%f\n", w->w2[0]);
    // 打印 w3 的第一个权重值
    printf("%f\n", w->w3[0]);
    // 再次打印 rms_att_weight 的第一个权重值
    printf("%f\n", w->rms_att_weight[0]);
    // 如果 wcls 存在，则打印其第一个权重值
    if (w->wcls) printf("%f\n", w->wcls[0]);
}
////////////////////////////////////////////////////////////////////////////////////////////////////////////

//////////////////////////////////////// ggml structs and functions required to load models, configs and save the model.

// 定义 llama_vocab 结构体
struct llama_vocab {
    using id    = int32_t;
    using token = std::string;
    using ttype = llama_token_type;

    // 定义 token_data 结构体
    struct token_data {
        token text;
        float score;
        ttype type;
    };

    // 从 token 到 id 的映射
    std::unordered_map<token, id> token_to_id;
    // id 到 token_data 的映射
    std::vector<token_data> id_to_token;
};

// 定义 my_llama_hparams 结构体
struct my_llama_hparams {
    uint32_t n_vocab = 32000;
    uint32_t n_ctx   = 512;   // this is provided as user input?
    uint32_t n_embd  = 4096;
    uint32_t n_ff    = 11008;
    uint32_t n_mult  = 4;
    uint32_t n_head  = 32;
    uint32_t n_layer = 32;
    uint32_t n_rot   = 64;
    // 重载不等于运算符
    bool operator!=(const my_llama_hparams& other) const {
        return memcmp(this, &other, sizeof(my_llama_hparams));
    }
};

// 定义 my_llama_layer 结构体
struct my_llama_layer {
    // 规范化
    struct ggml_tensor * attention_norm;

    // 注意力
    struct ggml_tensor * wq;
    struct ggml_tensor * wk;
    struct ggml_tensor * wv;
    struct ggml_tensor * wo;

    // 规范化
    struct ggml_tensor * ffn_norm;

    // 前馈
    struct ggml_tensor * w1;
    struct ggml_tensor * w2;
    struct ggml_tensor * w3;
};

// 定义 my_llama_model 结构体
struct my_llama_model {
    struct ggml_context * ctx = NULL;

    // 模型名称
    std::string name;
    # 定义名为my_llama_hparams的结构体变量hparams，用于存储模型的超参数
    my_llama_hparams hparams;
    
    # 定义指向ggml_tensor结构体的指针变量tok_embeddings，用于存储token的嵌入向量
    
    struct ggml_tensor * tok_embeddings;
    
    # 定义指向ggml_tensor结构体的指针变量norm，用于存储规范化后的数据
    # 定义指向ggml_tensor结构体的指针变量output，用于存储模型输出的数据
    
    struct ggml_tensor * norm;
    struct ggml_tensor * output;
    
    # 定义名为layers的vector容器，用于存储my_llama_layer结构体的实例
    
    std::vector<my_llama_layer> layers;
    
    # 定义train_its、train_samples、train_tokens三个无符号整型变量，用于存储训练迭代次数、样本数量和token数量
    
    uint32_t train_its = 0;
    uint32_t train_samples = 0;
    uint32_t train_tokens = 0;
};

// 定义训练参数结构体
struct train_params {
    const char * fn_vocab_model;
    const char * fn_llama2c_model;
    const char * fn_llama2c_output_model;
    const char * fn_train_data;
    const char * fn_checkpoint_in;
    const char * fn_checkpoint_out;
    const char * fn_model_out;

    uint32_t seed;

    int n_ctx;
    int n_embd;
    int n_mult;
    int n_head;
    int n_layer;
    int n_rotmax;

    int n_threads;
    int n_batch;
    int n_examples;
    int n_predict;

    int print_info_interval;
    int print_details_interval;

    bool samples_start_after_nl;
    bool use_adam;
    bool use_flash;
    bool use_scratch;

    // only adam
    int   warmup;
    int   cos_decay_steps;
    float cos_decay_restart;
    float cos_decay_alpha;

    int   lbfgs_n_iter;
    int   adam_n_iter;
    float adam_alpha;
    float adam_decay;

    int mem_model_gb;
    int mem_compute_gb;
    int mem_compute0_gb;
    int mem_compute1_gb;
};

// 打印参数信息
static void print_params(struct my_llama_hparams * params) {
    printf("%s: n_vocab: %d\n", __func__, params->n_vocab);
    printf("%s: n_ctx:   %d\n", __func__, params->n_ctx);
    printf("%s: n_embd:  %d\n", __func__, params->n_embd);
    printf("%s: n_mult:  %d\n", __func__, params->n_mult);
    printf("%s: n_head:  %d\n", __func__, params->n_head);
    printf("%s: n_ff:    %d\n", __func__, params->n_ff);
    printf("%s: n_layer: %d\n", __func__, params->n_layer);
    printf("%s: n_rot:   %d\n", __func__, params->n_rot);
}

// 初始化模型
static void init_model(struct my_llama_model * model) {
    const auto & hparams = model->hparams;

    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;
    const uint32_t n_vocab = hparams.n_vocab;

    const uint32_t n_ff = hparams.n_ff;
    struct ggml_context * ctx = model->ctx;

    model->train_its = 0;
    model->train_samples = 0;
    model->train_tokens = 0;

    // 初始化模型的词嵌入
    model->tok_embeddings = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab);
    # 打印分配给模型->tok_embeddings的float空间的信息
    printf("[%s:GG] Allocating [%d] x [%d] = [%d] float space for model->tok_embeddings\n",__func__,n_embd , n_vocab, n_embd * n_vocab);

    # 为模型->norm分配n_embd个float空间
    model->norm           = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
    printf("[%s:GG] Allocating [%d] float space for model->norm\n",__func__,n_embd);

    # 为模型->output分配n_embd * n_vocab个float空间
    model->output         = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab);
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for model->output\n",__func__,n_embd, n_vocab, n_embd * n_vocab);

    # 打印每层分配的空间，以便在for循环中不打印
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for layer.wq for [%d] layers\n",__func__, n_embd, n_embd, n_embd * n_embd, n_layer);
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for layer.wk for [%d] layers\n",__func__, n_embd, n_embd, n_embd * n_embd, n_layer);
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for layer.wv for [%d] layers\n",__func__, n_embd, n_embd, n_embd * n_embd, n_layer);
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for layer.wo for [%d] layers\n",__func__, n_embd, n_embd, n_embd * n_embd, n_layer);

    # 为每层的ffn_norm分配n_embd个float空间
    printf("[%s:GG] Allocating [%d] float space for layer.ffn_norm for [%d] layers\n",__func__,n_embd, n_layer);

    # 为每层的w1分配n_embd * n_ff个float空间
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for layer.w1 for [%d] layers\n",__func__, n_ff, n_embd, n_embd * n_ff, n_layer);
    # 为每层的w2分配n_ff * n_embd个float空间
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for layer.w2 for [%d] layers\n",__func__, n_embd, n_ff, n_ff * n_embd, n_layer);
    # 为每层的w3分配n_embd * n_ff个float空间
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for layer.w3 for [%d] layers\n",__func__, n_ff, n_embd, n_embd * n_ff, n_layer);

    # 设置模型的tok_embeddings的名称为"tok_embeddings.weight"
    ggml_set_name(model->tok_embeddings, "tok_embeddings.weight");
    # 设置模型的norm的名称为"norm.weight"
    ggml_set_name(model->norm,           "norm.weight");
    # 设置模型的output的名称为"output.weight"
    ggml_set_name(model->output,         "output.weight");

    # 调整模型的layers大小为n_layer
    model->layers.resize(n_layer);
    # 遍历每个层，i 从 0 到 n_layer-1
    for (uint32_t i = 0; i < n_layer; ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];

        # 构建层名称，格式为 "layers.i"
        std::string layers_i = "layers." + std::to_string(i);

        # 初始化当前层的 attention_norm 张量
        layer.attention_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        # 初始化当前层的 wq 张量
        layer.wq = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        # 初始化当前层的 wk 张量
        layer.wk = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        # 初始化当前层的 wv 张量
        layer.wv = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
        # 初始化当前层的 wo 张量
        layer.wo = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);

        # 初始化当前层的 ffn_norm 张量
        layer.ffn_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

        # 初始化当前层的 w1 张量
        layer.w1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);
        # 初始化当前层的 w2 张量
        layer.w2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_ff, n_embd);
        # 初始化当前层的 w3 张量
        layer.w3 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);

        # 设置当前层 attention_norm 张量的名称
        ggml_set_name(layer.attention_norm, (layers_i + ".attention_norm.weight").c_str());

        # 设置当前层 wq 张量的名称
        ggml_set_name(layer.wq, (layers_i + ".attention.wq.weight").c_str());
        # 设置当前层 wk 张量的名称
        ggml_set_name(layer.wk, (layers_i + ".attention.wk.weight").c_str());
        # 设置当前层 wv 张量的名称
        ggml_set_name(layer.wv, (layers_i + ".attention.wv.weight").c_str());
        # 设置当前层 wo 张量的名称
        ggml_set_name(layer.wo, (layers_i + ".attention.wo.weight").c_str());

        # 设置当前层 ffn_norm 张量的名称
        ggml_set_name(layer.ffn_norm, (layers_i + ".ffn_norm.weight").c_str());

        # 格式化设置当前层 w1 张量的名称
        ggml_format_name(layer.w1, "%s.feed_forward.w1.weight", layers_i.c_str());
        # 格式化设置当前层 w2 张量的名称
        ggml_format_name(layer.w2, "%s.feed_forward.w2.weight", layers_i.c_str());
        # 格式化设置当前层 w3 张量的名称
        ggml_format_name(layer.w3, "%s.feed_forward.w3.weight", layers_i.c_str());
    }
}

// 从二维张量中获取 float 类型数据
static float get_f32_2d(struct ggml_tensor * tensor, int64_t i0, int64_t i1) {
    // 计算指向数据的指针位置
    float * ptr = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1]);
    // 返回指针位置的数据
    return *ptr;
}

// 从二维张量中获取 int32_t 类型数据
static int32_t get_i32_2d(struct ggml_tensor * tensor, int64_t i0, int64_t i1) {
    // 计算指向数据的指针位置
    int32_t * ptr = (int32_t *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1]);
    // 返回指针位置的数据
    return *ptr;
}

// 打印张量中的一行数据
static void print_row(struct ggml_tensor * probs, int i) {
    // 遍历一行数据并打印
    for (int k = 0; k < probs->ne[0]; ++k) {
        float p = get_f32_2d(probs, k, i);
        printf(" %f", p);
    }
    printf("\n");
}

// 打印整个二维张量的数据
static void print_matrix(struct ggml_tensor * probs) {
    // 确保张量的维度为 2
    assert(probs->n_dims == 2);
    // 遍历整个二维张量并打印数据
    for (int i = 0; i < probs->ne[1]; ++i) {
        for (int k = 0; k < probs->ne[0]; ++k) {
            float p = get_f32_2d(probs, k, i);
            printf(" %.2f", p);
        }
        printf("\n");
    }
}

#ifdef __GNUC__
#ifdef __MINGW32__
__attribute__((format(gnu_printf, 1, 2)))
#else
__attribute__((format(printf, 1, 2)))
#endif
#endif
// 格式化字符串
static std::string format(const char * fmt, ...) {
    va_list ap, ap2;
    va_start(ap, fmt);
    va_copy(ap2, ap);
    // 获取格式化后字符串的长度
    int size = vsnprintf(NULL, 0, fmt, ap);
    GGML_ASSERT(size >= 0 && size < INT_MAX);
    // 创建缓冲区
    std::vector<char> buf(size + 1);
    // 将格式化后的字符串写入缓冲区
    int size2 = vsnprintf(buf.data(), size + 1, fmt, ap2);
    GGML_ASSERT(size2 == size);
    va_end(ap2);
    va_end(ap);
    // 返回格式化后的字符串
    return std::string(buf.data(), size);
}

// 文件结构体
struct llama_file {
    // 使用 FILE * 以便于 mmap 时不需要重新打开文件
    FILE * fp;
    size_t size;

    // 构造函数
    llama_file(const char * fname, const char * mode) {
        // 打开文件
        fp = std::fopen(fname, mode);
        // 如果文件打开失败，设置文件大小为 0
        if (fp == NULL) {
            size = 0;
        } else {
            // 定位到文件末尾，获取文件大小
            seek(0, SEEK_END);
            size = tell();
            seek(0, SEEK_SET);
        }
    }

    // 获取当前文件指针位置
    size_t tell() const {
#ifdef _WIN32
        __int64 ret = _ftelli64(fp);
#else
        long ret = std::ftell(fp);
#endif
        // 确保返回值不为-1，如果为-1则抛出断言错误
        GGML_ASSERT(ret != -1); // this really shouldn't fail
        // 返回读取的字节数
        return (size_t) ret;
    }

    void seek(size_t offset, int whence) {
#ifdef _WIN32
        // 在 Windows 平台下使用 _fseeki64 函数
        int ret = _fseeki64(fp, (__int64) offset, whence);
#else
        // 在其他平台下使用 std::fseek 函数
        int ret = std::fseek(fp, (long) offset, whence);
#endif
        // 确保返回值为0，如果不为0则抛出断言错误
        GGML_ASSERT(ret == 0); // same
    }

    void read_raw(void * ptr, size_t size) {
        // 如果 size 为0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空 errno
        errno = 0;
        // 读取数据到 ptr 指向的内存中，返回实际读取的元素个数
        std::size_t ret = std::fread(ptr, size, 1, fp);
        // 如果发生读取错误，则输出错误信息并终止程序
        if (ferror(fp)) {
            die_fmt("fread failed: %s", strerror(errno));
        }
        // 如果实际读取的元素个数不为1，则表示已经到达文件末尾，终止程序
        if (ret != 1) {
            die("unexpectedly reached end of file");
        }
    }

    std::uint32_t read_u32() {
        std::uint32_t ret;
        // 读取一个 32 位无符号整数
        read_raw(&ret, sizeof(ret));
        return ret;
    }
    std::float_t read_f32() {
        std::float_t ret;
        // 读取一个 32 位浮点数
        read_raw(&ret, sizeof(ret));
        return ret;
    }

    std::string read_string(std::uint32_t len) {
        // 读取指定长度的字符串
        std::vector<char> chars(len);
        read_raw(chars.data(), len);
        return std::string(chars.data(), len);
    }

    ~llama_file() {
        // 如果文件指针不为空，则关闭文件
        if (fp) {
            std::fclose(fp);
        }
    }
};

static bool is_ggml_file(const char * filename) {
    // 以只读方式打开文件
    llama_file file(filename, "rb");
    // 如果文件大小小于4，则不是 ggml 文件
    if (file.size < 4) {
        return false;
    }
    // 读取文件的前4个字节作为魔数
    std::string magic = file.read_string(4);
    // 判断魔数是否为 GGUF_MAGIC
    return magic == GGUF_MAGIC;
}

static std::string llama_escape_whitespaces(const std::string & text) {
    // 转义字符串中的空格
    std::ostringstream out;
    for (char c : text) {
        if (c == ' ') out << "\xe2\x96\x81";
        else out << c;
    }
    return out.str();
}

static void load_vocab(const char *filename, Config *config, struct llama_vocab *vocab) {
    # 如果文件是 GGML 文件
    if (is_ggml_file(filename)) {
        # 定义一个指向 ggml_context 结构体的指针，并初始化为 NULL
        struct ggml_context * ctx_data = NULL;

        # 初始化 gguf_init_params 结构体
        struct gguf_init_params params = {
            # 不进行内存分配
            /*.no_alloc = */ false,
            # 将 ctx_data 的地址赋给 ctx
            /*.ctx      = */ &ctx_data,
        };

        # 从文件中初始化 gguf_context 对象
        struct gguf_context * ctx = gguf_init_from_file(filename, params);
        # 断言 ctx 不为空
        GGML_ASSERT(ctx != NULL);

        # 获取模型索引
        const int model_idx = gguf_find_key(ctx, KV_TOKENIZER_MODEL);
        # 断言模型索引大于等于 0
        GGML_ASSERT(model_idx >= 0);
        # 获取分词器名称
        std::string tokenizer_name = gguf_get_val_str(ctx, model_idx);
        # 断言分词器名称与 TOKENIZER_NAME 相等
        GGML_ASSERT(tokenizer_name == TOKENIZER_NAME);

        # 获取分词器列表索引
        const int token_idx = gguf_find_key(ctx, KV_TOKENIZER_LIST);
        # 断言分词器列表索引大于等于 0
        GGML_ASSERT(token_idx >= 0);

        # 获取分词器分数索引
        const int score_idx = gguf_find_key(ctx, KV_TOKENIZER_SCORES);
        # 断言分词器分数索引大于等于 0
        GGML_ASSERT(score_idx >= 0);
        # 获取分词器分数数组
        const float * scores = (const float * ) gguf_get_arr_data(ctx, score_idx);

        # 获取分词器类型索引
        const int toktype_idx = gguf_find_key(ctx, KV_TOKENIZER_TOKEN_TYPE);
        # 断言分词器类型索引大于等于 0
        GGML_ASSERT(toktype_idx >= 0);
        # 获取分词器类型数组
        const int * toktypes = (const int * ) gguf_get_arr_data(ctx, toktype_idx);

        # 获取词汇表大小
        const uint32_t n_vocab = gguf_get_arr_n(ctx, token_idx);

        # 调整词汇表的大小
        vocab->id_to_token.resize(n_vocab);

        # 遍历词汇表
        for (uint32_t i = 0; i < n_vocab; i++) {
            # 获取词汇表中的词
            std::string word = gguf_get_arr_str(ctx, token_idx, i);

            # 将词和索引添加到词汇表的映射中
            vocab->token_to_id[word] = i;

            # 获取词汇表中索引对应的词的数据
            auto & token_data = vocab->id_to_token[i];
            # 移动词的内容
            token_data.text  = std::move(word);
            # 设置词的分数
            token_data.score = scores[i];
            # 设置词的类型
            token_data.type  = (llama_token_type) toktypes[i];
        }
        # 释放 ctx_data 指向的内存
        ggml_free(ctx_data);
        # 释放 ctx 对象
        gguf_free(ctx);
    }
    } else {
        // 假设 llama2.c 词汇表
        printf("Assuming llama2.c vocabulary since %s is not a gguf file\n", filename);
        // 以只读方式打开文件
        llama_file file(filename, "rb");
        // 如果文件指针为空，输出错误信息并退出程序
        if (!file.fp) {
            die_fmt("%s: %s", strerror(errno), filename);
        }
        // 获取词汇表大小
        const int  n_vocab = config->vocab_size;
        /* uint32_t max_token_length =  */ file.read_u32(); // 未使用
        // 调整词汇表的大小
        vocab->id_to_token.resize(n_vocab);
        // 遍历词汇表
        for (llama_vocab::id id=0; id<n_vocab; ++id) {
            // 读取浮点数类型的分数
            float_t score = file.read_f32();
            // 读取无符号32位整数类型的长度
            uint32_t len = file.read_u32();
            // 读取指定长度的字符串
            std::string text = file.read_string(len);

            unsigned char byte_val;
            llama_vocab::ttype type = LLAMA_TOKEN_TYPE_NORMAL;
            // 根据 id 判断词汇类型
            if (id == UNKNOWN_TOKEN_ID) {
                text = "<unk>";
                type = LLAMA_TOKEN_TYPE_UNKNOWN;
            } else if (id == BOS_TOKEN_ID) {
                text = "<s>";
                type = LLAMA_TOKEN_TYPE_CONTROL;
            } else if (id == EOS_TOKEN_ID) {
                text = "</s>";
                type = LLAMA_TOKEN_TYPE_CONTROL;
            } else if (text.empty()) {
                type = LLAMA_TOKEN_TYPE_CONTROL;
            } else if (sscanf(text.c_str(), "<0x%02hhX>", &byte_val) == 1) {
                // 字节标记的文本已经是预期格式
                type = LLAMA_TOKEN_TYPE_BYTE;
            } else {
                type = LLAMA_TOKEN_TYPE_NORMAL;
            }
            // 转义字符串中的空白字符
            text = llama_escape_whitespaces(text);

            // 将词汇信息存入词汇表中
            vocab->id_to_token[id].text = text;
            vocab->id_to_token[id].score = score;
            vocab->id_to_token[id].type = type;
            vocab->token_to_id.emplace(text, id);
        }
    }
// 将 AK 权重转换为 GG 权重
static void convert_weights_ak_to_gg(struct ggml_tensor * gg_weights, const float * karpathy_weights) {
    int ct;
    switch (gg_weights->n_dims){
        // 如果张量维度为 1
        case 1:
            ct = 0;
            // 遍历第一维度，将 AK 权重转换为 GG 权重
            for (int i0 = 0; i0 < gg_weights->ne[0]; i0++){
                float * ptr = (float *) ((char *) gg_weights->data + i0*gg_weights->nb[0]);
                *ptr = karpathy_weights[ct];
                ct++;
            }
            break;
        // 如果张量维度为 2
        case 2:
            ct = 0;
            // 遍历第二维度和第一维度，将 AK 权重转换为 GG 权重
            for (int i1 = 0; i1 < gg_weights->ne[1]; i1++) {
                for (int i0 = 0; i0 < gg_weights->ne[0]; i0++) {
                    float * ptr = (float *) ((char *) gg_weights->data + i0*gg_weights->nb[0] + i1*gg_weights->nb[1]);
                    *ptr = karpathy_weights[ct];
                    ct++;
                }
            }
            break;
        // 如果张量维度为 3
        case 3:
            ct = 0;
            // 遍历第三维度、第二维度和第一维度，将 AK 权重转换为 GG 权重
            for (int i2 = 0; i2 < gg_weights->ne[2]; i2++) {
                for (int i1 = 0; i1 < gg_weights->ne[1]; i1++) {
                    for (int i0 = 0; i0 < gg_weights->ne[0]; i0++) {
                        float * ptr = (float *) ((char *) gg_weights->data + i0*gg_weights->nb[0] + i1*gg_weights->nb[1] + i2*gg_weights->nb[2]);
                        *ptr = karpathy_weights[ct];
                        ct++;
                    }
                }
            }
            break;
    }
}

// 将 AK 权重转换为 GG 权重，并保存为 LLAMA 模型
static void save_as_llama_model(
    struct llama_vocab * vocab, struct my_llama_model * model, TransformerWeights* w, const char * filename
) {
    // 将 AK 权重转换为 GG 权重，保存到 LLAMA 模型中
    convert_weights_ak_to_gg(model->tok_embeddings, w->token_embedding_table);
    convert_weights_ak_to_gg(model->output, w->wcls ? w->wcls : w->token_embedding_table);
    convert_weights_ak_to_gg(model->norm, w->rms_final_weight);
    //print_row(model->norm, 0);

    // for rms-att-weight
}
    // 获取模型中的嵌入层维度
    int row_length = model->hparams.n_embd;
    // 获取模型中的前馈神经网络层维度
    int n_ff = model->hparams.n_ff;

    // 遍历模型的每一层
    for (uint32_t i = 0; i < model->hparams.n_layer; ++i){
        // 获取当前层的引用
        auto & layer = model->layers[i];
        // 将注意力规范化层的权重转换为指数加权移动平均权重
        convert_weights_ak_to_gg(layer.attention_norm, &w->rms_att_weight[i*row_length]);
        // 将前馈神经网络规范化层的权重转换为指数加权移动平均权重
        convert_weights_ak_to_gg(layer.ffn_norm      , &w->rms_ffn_weight[i*row_length]);

        // 将注意力机制中的权重矩阵从三维转换为二维
        convert_weights_ak_to_gg(layer.wq            , &w->wq[i*row_length*row_length]);
        convert_weights_ak_to_gg(layer.wk            , &w->wk[i*row_length*row_length]);
        convert_weights_ak_to_gg(layer.wv            , &w->wv[i*row_length*row_length]);
        convert_weights_ak_to_gg(layer.wo            , &w->wo[i*row_length*row_length]);

        // 将前馈神经网络中的权重矩阵从三维转换为二维
        convert_weights_ak_to_gg(layer.w1            , &w->w1[i*row_length*n_ff]);
        convert_weights_ak_to_gg(layer.w2            , &w->w2[i*n_ff*row_length]);
        convert_weights_ak_to_gg(layer.w3            , &w->w3[i*row_length*n_ff]);
    }

    // 初始化一个空的 gguf 上下文
    struct gguf_context * ctx = gguf_init_empty();

    // 创建存储 token、score 和 token 类型的容器
    std::vector<const char*> tokens;
    std::vector<float> scores;
    std::vector<llama_token_type> token_types;
    // 遍历词汇表中的每个 token 数据
    for (const llama_vocab::token_data & token_data : vocab->id_to_token) {
        // 将 token 文本添加到 tokens 容器
        tokens.push_back(token_data.text.c_str());
        // 将 token 分数添加到 scores 容器
        scores.push_back(token_data.score);
        // 将 token 类型添加到 token_types 容器
        token_types.push_back(token_data.type);
    }
    // 将 tokens 数组存储到 gguf 上下文中
    gguf_set_arr_str(ctx, KV_TOKENIZER_LIST, tokens.data(), tokens.size());
    // 将 scores 数组存储到 gguf 上下文中
    gguf_set_arr_data(ctx, KV_TOKENIZER_SCORES, GGUF_TYPE_FLOAT32, scores.data(), scores.size());
    // 将 token_types 数组存储到 gguf 上下文中
    gguf_set_arr_data(ctx, KV_TOKENIZER_TOKEN_TYPE, GGUF_TYPE_INT32, token_types.data(), token_types.size());

    // 将模型名称存储到 gguf 上下文中
    gguf_set_val_str(ctx, KV_TOKENIZER_MODEL, TOKENIZER_NAME);

    // 将通用架构类型存储到 gguf 上下文中
    gguf_set_val_str(ctx, KV_GENERAL_ARCHITECTURE, "llama");
    // 将通用名称存储到 gguf 上下文中
    gguf_set_val_str(ctx, KV_GENERAL_NAME, "llama");

    // 存储未知 token 的 ID 到 gguf 上下文中
    gguf_set_val_u32(ctx, KV_TOKENIZER_UNK_ID, UNKNOWN_TOKEN_ID);
    # 设置上下文的特殊标记值
    gguf_set_val_u32(ctx, KV_TOKENIZER_BOS_ID, BOS_TOKEN_ID);
    gguf_set_val_u32(ctx, KV_TOKENIZER_EOS_ID, EOS_TOKEN_ID);
    gguf_set_val_u32(ctx, KV_TOKENIZER_SEP_ID, -1);
    gguf_set_val_u32(ctx, KV_TOKENIZER_PAD_ID, -1);

    # 设置上下文长度、嵌入长度、前馈长度、注意力头数、块数、维度数、注意力层归一化的 RMS 误差
    gguf_set_val_u32(ctx, KV_CONTEXT_LENGTH, model->hparams.n_ctx);
    gguf_set_val_u32(ctx, KV_EMBEDDING_LENGTH, model->hparams.n_embd);
    gguf_set_val_u32(ctx, KV_FEED_FORWARD_LENGTH, model->hparams.n_ff);
    gguf_set_val_u32(ctx, KV_ATTENTION_HEAD_COUNT, model->hparams.n_head);
    # n_head_kv 是可选的，默认为 n_head
    # gguf_set_val_u32(ctx, KV_ATTENTION_HEAD_COUNT_KV, ...);
    gguf_set_val_u32(ctx, KV_BLOCK_COUNT, model->hparams.n_layer);
    gguf_set_val_u32(ctx, KV_ROPE_DIMENSION_COUNT, model->hparams.n_rot);
    gguf_set_val_f32(ctx, KV_ATTENTION_LAYERNORM_RMS_EPS, 1e-5f);

    # 写入张量
    ggml_set_name(model->tok_embeddings, TN_TOKEN_EMBD);
    gguf_add_tensor(ctx, model->tok_embeddings);

    ggml_set_name(model->norm, TN_OUTPUT_NORM);
    gguf_add_tensor(ctx, model->norm);

    ggml_set_name(model->output, TN_OUTPUT);
    gguf_add_tensor(ctx, model->output);
    # 遍历模型的每一层
    for (uint32_t i = 0; i < model->hparams.n_layer; ++i) {
        # 获取当前层的引用
        auto & layer = model->layers[i];

        # 为当前层的权重矩阵添加名称并注册到图中
        ggml_format_name(layer.wq, TN_ATTN_Q, i);
        gguf_add_tensor(ctx, layer.wq);

        # 为当前层的偏置矩阵添加名称并注册到图中
        ggml_format_name(layer.wk, TN_ATTN_K, i);
        gguf_add_tensor(ctx, layer.wk);

        # 为当前层的值矩阵添加名称并注册到图中
        ggml_format_name(layer.wv, TN_ATTN_V, i);
        gguf_add_tensor(ctx, layer.wv);

        # 为当前层的输出矩阵添加名称并注册到图中
        ggml_format_name(layer.wo, TN_ATTN_OUTPUT, i);
        gguf_add_tensor(ctx, layer.wo);

        # 为当前层的注意力规范化矩阵添加名称并注册到图中
        ggml_format_name(layer.attention_norm, TN_ATTN_NORM, i);
        gguf_add_tensor(ctx, layer.attention_norm);

        # 为当前层的第一个全连接层权重矩阵添加名称并注册到图中
        ggml_format_name(layer.w1, TN_FFN_GATE, i);
        gguf_add_tensor(ctx, layer.w1);

        # 为当前层的第二个全连接层权重矩阵添加名称并注册到图中
        ggml_format_name(layer.w2, TN_FFN_DOWN, i);
        gguf_add_tensor(ctx, layer.w2);

        # 为当前层的第三个全连接层权重矩阵添加名称并注册到图中
        ggml_format_name(layer.w3, TN_FFN_UP, i);
        gguf_add_tensor(ctx, layer.w3);

        # 为当前层的全连接层规范化矩阵添加名称并注册到图中
        ggml_format_name(layer.ffn_norm, TN_FFN_NORM, i);
        gguf_add_tensor(ctx, layer.ffn_norm);
    }

    # 将图中的信息写入文件
    gguf_write_to_file(ctx, filename, false);
    # 释放图上下文
    gguf_free(ctx);
// 返回默认的训练参数结构体
static struct train_params get_default_train_params() {
    // 初始化训练参数结构体
    struct train_params params;
    // 设置默认的词汇模型文件名
    params.fn_vocab_model    = "models/7B/ggml-model-f16.gguf";
    // 设置输出模型文件名
    params.fn_llama2c_output_model = "ak_llama_model.bin";
    // 设置训练数据文件名
    params.fn_train_data     = "shakespeare.txt";
    // 设置输入检查点文件名
    params.fn_checkpoint_in  = "checkpoint.bin";
    // 设置输出检查点文件名
    params.fn_checkpoint_out = "checkpoint.bin";
    // 设置输出模型文件名
    params.fn_model_out      = "ggml-checkpoint-f32.bin";

    // 设置随机数种子
    params.seed       =   -1;

    // 设置上下文大小
    params.n_ctx      =  128;
    // 设置嵌入维度
    params.n_embd     =  256;
    // 设置多头注意力机制中的维度
    params.n_mult     =  256;
    // 设置注意力头数
    params.n_head     =    8;
    // 设置层数
    params.n_layer    =   16;
    // 设置旋转最大值
    params.n_rotmax   =   64;

    // 设置线程数
    params.n_threads  =    6;
    // 设置批次大小
    params.n_batch    =    8;
    // 设置示例数
    params.n_examples =    8;
    // 设置预测数
    params.n_predict  = 1024;

    // 设置打印信息间隔
    params.print_info_interval    = 1;
    // 设置打印详细信息间隔
    params.print_details_interval = 2;

    // 设置是否在换行后开始采样
    params.samples_start_after_nl = false;
    // 设置是否使用 Adam 优化器
    params.use_adam               = true;
    // 设置是否使用 Flash
    params.use_flash              = true;
    // 设置是否使用 Scratch
    params.use_scratch            = true;

    // 仅使用 Adam 优化器时的参数设置
    params.warmup            =  100;
    params.cos_decay_steps   = 1000;
    params.cos_decay_restart = 1.1f;
    params.cos_decay_alpha   = 0.0f;

    // LBFGS 优化器的迭代次数
    params.lbfgs_n_iter      = 16;
    // Adam 优化器的迭代次数
    params.adam_n_iter       = 16;
    // Adam 优化器的学习率
    params.adam_alpha        = 1e-3f;
    // Adam 优化器的衰减率
    params.adam_decay        = 1e-3f;

    // 模型内存占用（GB）
    params.mem_model_gb   = 2;
    // 计算内存占用（GB）
    params.mem_compute_gb = 24;
    // 计算内存占用（GB）
    params.mem_compute0_gb = 8;
    // 计算内存占用（GB）
    params.mem_compute1_gb = 2;

    // 返回设置好的训练参数结构体
    return params;
}

// 打印使用说明
static void print_usage(int /*argc*/, char ** argv, const struct train_params * params) {
    // 打印使用说明
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help                       show this help message and exit\n");
    fprintf(stderr, "  --copy-vocab-from-model FNAME    path of gguf llama model or llama2.c vocabulary from which to copy vocab (default '%s')\n", params->fn_vocab_model);
}
    # 输出错误信息，指示需要提供的参数：模型路径
    fprintf(stderr, "  --llama2c-model FNAME            [REQUIRED] model path from which to load Karpathy's llama2.c model\n");
    # 输出错误信息，指示可选参数：保存转换后的模型路径，默认为预设值
    fprintf(stderr, "  --llama2c-output-model FNAME     model path to save the converted llama2.c model (default %s')\n", params->fn_llama2c_output_model);
    # 输出空行
    fprintf(stderr, "\n");
// 解析命令行参数，将参数值存储到结构体 train_params 中
static bool params_parse(int argc, char ** argv, struct train_params * params) {
    // 初始化标志变量
    bool invalid_param = false;
    bool reqd_param_found = false;
    // 声明字符串变量
    std::string arg;
    // 获取默认的训练参数
    struct train_params default_params = get_default_train_params();
    // 定义参数前缀
    const std::string arg_prefix = "--";

    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数
        arg = argv[i];
        // 如果参数以 arg_prefix 开头，则将下划线替换为破折号
        if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
            std::replace(arg.begin(), arg.end(), '_', '-');
        }

        // 根据参数类型进行处理
        if (arg == "--copy-vocab-from-model") {
            // 检查参数是否存在
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 存储参数值到结构体中
            params->fn_vocab_model = argv[i];
        } else if (arg == "--llama2c-model") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            reqd_param_found = true;
            params->fn_llama2c_model = argv[i];
        } else if (arg == "--llama2c-output-model") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params->fn_llama2c_output_model = argv[i];
        } else if (arg == "-h" || arg == "--help") {
            // 打印帮助信息并退出程序
            print_usage(argc, argv, &default_params);
            exit(0);
        } else {
            // 打印错误信息并退出程序
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            print_usage(argc, argv, &default_params);
            exit(1);
        }
    }
    // 检查是否存在无效参数
    if (invalid_param) {
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        print_usage(argc, argv, &default_params);
        exit(1);
    }
    // 检查是否存在必需参数
    if (!reqd_param_found){
        fprintf(stderr, "error: please specify a llama2.c .bin file to be converted with argument --llama2c-model\n");
        print_usage(argc, argv, &default_params);
        exit(1);
    }

    return true;
}

// 获取路径的基本名称
static std::string basename(const std::string &path) {
    // 查找路径中最后一个斜杠或反斜杠的位置
    size_t pos = path.find_last_of("/\\");
    # 如果找不到指定子字符串的位置，返回原始路径
    if (pos == std::string::npos) {
        return path;
    }
    # 返回从指定位置开始到字符串末尾的子字符串
    return path.substr(pos + 1);
// 主函数，接受命令行参数并执行训练
int main(int argc, char ** argv) {
    // 获取默认的训练参数
    struct train_params params = get_default_train_params();
    // 解析命令行参数，如果解析失败则返回1
    if (!params_parse(argc, argv, &params)) {
        return 1;
    }
    // 定义配置和权重对象
    Config config;
    TransformerWeights weights = {};
    {
        // 打开 llama2c 模型文件
        FILE *file = fopen(params.fn_llama2c_model, "rb");
        // 如果文件打开失败则输出错误信息并返回1
        if (!file) { printf("Unable to open the checkpoint file %s!\n", params.fn_llama2c_model); return 1; }
        // 读取配置头部信息
        if(fread(&config, sizeof(Config), 1, file) != 1) { return 1; }
        // 检查是否共享权重，如果是则更新词汇表大小
        auto shared_weights = config.vocab_size > 0;
        config.vocab_size = abs(config.vocab_size);
        // 分配权重内存并初始化权重
        malloc_weights(&weights, &config, shared_weights);
        if(checkpoint_init_weights(&weights, &config, file, shared_weights)) { return 1; }
        // 关闭文件
        fclose(file);
    }

    // 加载词汇表
    struct llama_vocab vocab;
    load_vocab(params.fn_vocab_model, &config, &vocab);

    // 初始化 llama 模型参数
    struct my_llama_model model;
    model.hparams.n_vocab = config.vocab_size; //llama_n_vocab(lctx);
    model.hparams.n_ctx   = params.n_ctx;
    model.hparams.n_embd  = config.dim; //params.n_embd;
    model.hparams.n_ff    = config.hidden_dim;
    model.hparams.n_mult  = 32;//params.n_mult;
    model.hparams.n_head  = config.n_heads; //params.n_head;
    model.hparams.n_layer = config.n_layers; //params.n_layer;
    model.hparams.n_rot   = std::min((uint32_t)params.n_rotmax, model.hparams.n_embd / model.hparams.n_head);
    // 打印模型参数
    print_params(&model.hparams);
    // 初始化 ggml 上下文
    struct ggml_init_params lcparams;
    lcparams.mem_size   = 1024ll*1024ll*1024ll*((size_t) params.mem_model_gb);
    lcparams.mem_buffer = NULL;
    lcparams.no_alloc   = false;
    model.ctx = ggml_init(lcparams);
    // 初始化模型
    init_model(&model);
    // 设置模型名称
    model.name = basename(params.fn_llama2c_model);
    // 保存模型为 llama 模型文件
    save_as_llama_model(&vocab, &model, &weights, params.fn_llama2c_output_model);
    // 输出保存模型的信息
    printf("Saving llama.c model file %s in ggml format at %s\n", params.fn_llama2c_model, params.fn_llama2c_output_model);
}
    # 释放模型上下文所占用的内存
    ggml_free(model.ctx);
    # 返回值为0，表示函数执行成功
    return 0;
# 闭合前面的函数定义
```