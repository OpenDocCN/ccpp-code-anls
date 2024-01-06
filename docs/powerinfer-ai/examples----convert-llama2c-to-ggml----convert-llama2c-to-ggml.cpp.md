# `PowerInfer\examples\convert-llama2c-to-ggml\convert-llama2c-to-ggml.cpp`

```
// 包含所需的头文件
#include "ggml.h"
#include "llama.h"
#include "common.h"

// 包含所需的标准库头文件
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

// 定义 GGUF 键和张量名称的宏
#define KV_GENERAL_ARCHITECTURE          "general.architecture"
# 定义通用名称的键值对
#define KV_GENERAL_NAME                  "general.name"

# 定义分词器模型的键值对
#define KV_TOKENIZER_MODEL               "tokenizer.ggml.model"
# 定义分词器标记列表的键值对
#define KV_TOKENIZER_LIST                "tokenizer.ggml.tokens"
# 定义分词器标记类型的键值对
#define KV_TOKENIZER_TOKEN_TYPE          "tokenizer.ggml.token_type"
# 定义分词器分数的键值对
#define KV_TOKENIZER_SCORES              "tokenizer.ggml.scores"
# 定义分词器起始标记的键值对
#define KV_TOKENIZER_BOS_ID              "tokenizer.ggml.bos_token_id"
# 定义分词器结束标记的键值对
#define KV_TOKENIZER_EOS_ID              "tokenizer.ggml.eos_token_id"
# 定义分词器未知标记的键值对
#define KV_TOKENIZER_UNK_ID              "tokenizer.ggml.unknown_token_id"
# 定义分词器分隔符标记的键值对
#define KV_TOKENIZER_SEP_ID              "tokenizer.ggml.seperator_token_id"
# 定义分词器填充标记的键值对
#define KV_TOKENIZER_PAD_ID              "tokenizer.ggml.padding_token_id"
# 定义分词器 Huggingface JSON 的键值对
#define KV_TOKENIZER_HF_JSON             "tokenizer.huggingface.json"

# 定义 Llama 上下文长度的键值对
#define KV_CONTEXT_LENGTH                "llama.context_length"
# 定义 Llama 嵌入长度的键值对
#define KV_EMBEDDING_LENGTH              "llama.embedding_length"
# 定义 Llama 块数量的键值对
#define KV_BLOCK_COUNT                   "llama.block_count"
# 定义 Llama 前馈长度的键值对
#define KV_FEED_FORWARD_LENGTH           "llama.feed_forward_length"
# 定义 Llama 注意力头数量的键值对
#define KV_ATTENTION_HEAD_COUNT          "llama.attention.head_count"
# 定义 Llama 注意力头数量 KV 的键值对
#define KV_ATTENTION_HEAD_COUNT_KV       "llama.attention.head_count_kv"
# 定义 Llama 注意力层归一化 RMS 误差的键值对
#define KV_ATTENTION_LAYERNORM_RMS_EPS   "llama.attention.layer_norm_rms_epsilon"
# 定义键值对绳索维度计数的键名
#define KV_ROPE_DIMENSION_COUNT          "llama.rope.dimension_count"

# 定义各种权重文件的键名
#define TN_TOKEN_EMBD  "token_embd.weight"
#define TN_OUTPUT_NORM "output_norm.weight"
#define TN_OUTPUT      "output.weight"
#define TN_ATTN_NORM   "blk.%d.attn_norm.weight"
#define TN_ATTN_Q      "blk.%d.attn_q.weight"
#define TN_ATTN_K      "blk.%d.attn_k.weight"
#define TN_ATTN_V      "blk.%d.attn_v.weight"
#define TN_ATTN_OUTPUT "blk.%d.attn_output.weight"
#define TN_FFN_NORM    "blk.%d.ffn_norm.weight"
#define TN_FFN_GATE    "blk.%d.ffn_gate.weight"
#define TN_FFN_DOWN    "blk.%d.ffn_down.weight"
#define TN_FFN_UP      "blk.%d.ffn_up.weight"

# 如果是在 MSC 编译器下，禁用警告 4244 和 4267，这两个警告是可能的数据丢失
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

# 定义 LLAMA 文件的魔术数字为 0x67676a74u，即 'ggjt'
#define LLAMA_FILE_MAGIC_GGJT        0x67676a74u // 'ggjt'
// 定义 LLAMA 文件版本号为 3
#define LLAMA_FILE_VERSION_GGJT_V3   3

// 定义分词器名称为 "llama"，未知标记 ID 为 0，句子开头标记 ID 为 1，句子结尾标记 ID 为 2
#define TOKENIZER_NAME "llama"
#define UNKNOWN_TOKEN_ID 0
#define BOS_TOKEN_ID 1
#define EOS_TOKEN_ID 2

// llama2.c 模型结构和加载模型、分配内存等函数
typedef struct {
    int dim; // transformer 维度
    int hidden_dim; // ffn 层的隐藏层维度
    int n_layers; // 层数
    int n_heads; // 查询头的数量
    int n_kv_heads; // 键/值头的数量（可以小于查询头，因为有多个查询）
    int vocab_size; // 词汇表大小，通常为 256（字节级别）
    int seq_len; // 最大序列长度
} Config;

struct TransformerWeights {
    // 标记嵌入表
    float* token_embedding_table;    // 存储 token 的嵌入表，维度为 (词汇量大小, 嵌入维度)
    // rmsnorms 的权重
    float* rms_att_weight; // (层数, 嵌入维度) rmsnorm 权重
    float* rms_ffn_weight; // (层数, 嵌入维度)
    // matmuls 的权重
    float* wq; // (层数, 嵌入维度, 嵌入维度)
    float* wk; // (层数, 嵌入维度, 嵌入维度)
    float* wv; // (层数, 嵌入维度, 嵌入维度)
    float* wo; // (层数, 嵌入维度, 嵌入维度)
    // ffn 的权重
    float* w1; // (层数, 隐藏层维度, 嵌入维度)
    float* w2; // (层数, 嵌入维度, 隐藏层维度)
    float* w3; // (层数, 隐藏层维度, 嵌入维度)
    // 最终的 rmsnorm 权重
    float* rms_final_weight; // (嵌入维度,)
    // RoPE 相对位置嵌入的 freq_cis
    // float* freq_cis_real; // (序列长度, 嵌入维度/2)
    // float* freq_cis_imag; // (序列长度, 嵌入维度/2)
    // (可选) 用于最后一层的分类器权重
    float* wcls;
// 定义了一个析构函数，用于释放TransformerWeights对象中动态分配的内存
~TransformerWeights() {
    // 释放token_embedding_table数组的内存
    delete[] token_embedding_table;
    // 释放rms_att_weight数组的内存
    delete[] rms_att_weight;
    // 释放rms_ffn_weight数组的内存
    delete[] rms_ffn_weight;
    // 释放wq数组的内存
    delete[] wq;
    // 释放wk数组的内存
    delete[] wk;
    // 释放wv数组的内存
    delete[] wv;
    // 释放wo数组的内存
    delete[] wo;
    // 释放w1数组的内存
    delete[] w1;
    // 释放w2数组的内存
    delete[] w2;
    // 释放w3数组的内存
    delete[] w3;
    // 释放rms_final_weight数组的内存
    delete[] rms_final_weight;
    // 释放wcls数组的内存
    delete[] wcls;
}

// 定义了一个静态函数，用于动态分配TransformerWeights对象中的内存
static void malloc_weights(TransformerWeights* w, Config* p, bool shared_weights) {
    // 使用new关键字动态分配一个大小为p->vocab_size * p->dim的float数组，并将其初始化为0
    // 这里使用calloc而不是malloc是为了保持valgrind的正常运行
    w->token_embedding_table = new float[p->vocab_size * p->dim]();
# 打印分配给w->token_embedding_table的float空间的信息
printf("[%s:AK] Allocating [%d] x [%d] = [%d] float space for w->token_embedding_table\n",__func__,p->vocab_size , p->dim, p->vocab_size * p->dim);

# 分配给w->rms_att_weight的float空间，并初始化为0
w->rms_att_weight = new float[p->n_layers * p->dim]();
printf("[%s:AK] Allocating [%d] x [%d] = [%d] float space for w->rms_att_weight\n",__func__,p->n_layers, p->dim, p->n_layers * p->dim);

# 分配给w->rms_ffn_weight的float空间，并初始化为0
w->rms_ffn_weight = new float[p->n_layers * p->dim]();
printf("[%s:AK] Allocating [%d] x [%d] = [%d] float space for w->rms_ffn_weight\n",__func__,p->n_layers , p->dim, p->n_layers * p->dim);

# 分配给w->wq的float空间，并初始化为0
w->wq = new float[p->n_layers * p->dim * p->dim]();
printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->wq\n",__func__,p->n_layers, p->dim, p->dim, p->n_layers * p->dim * p->dim);

# 分配给w->wk的float空间，并初始化为0
w->wk = new float[p->n_layers * p->dim * p->dim]();
printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->wk\n",__func__,p->n_layers, p->dim, p->dim, p->n_layers * p->dim * p->dim);

# 分配给w->wv的float空间，并初始化为0
w->wv = new float[p->n_layers * p->dim * p->dim]();
printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->wv\n",__func__, p->n_layers, p->dim, p->dim, p->n_layers * p->dim * p->dim);

# 分配给w->wo的float空间，并初始化为0
w->wo = new float[p->n_layers * p->dim * p->dim]();
printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->wo\n",__func__,p->n_layers, p->dim, p->dim, p->n_layers * p->dim * p->dim);
// 为 w->w1 分配内存空间，大小为 p->n_layers * p->hidden_dim * p->dim，初始化为 0
w->w1 = new float[p->n_layers * p->hidden_dim * p->dim]();
// 打印分配内存空间的信息
printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->w1\n",__func__,p->n_layers, p->hidden_dim, p->dim, p->n_layers * p->hidden_dim * p->dim);

// 为 w->w2 分配内存空间，大小为 p->n_layers * p->hidden_dim * p->dim，初始化为 0
w->w2 = new float[p->n_layers * p->hidden_dim * p->dim]();
// 打印分配内存空间的信息
printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->w2\n",__func__,p->n_layers, p->dim, p->hidden_dim, p->n_layers * p->hidden_dim * p->dim);

// 为 w->w3 分配内存空间，大小为 p->n_layers * p->hidden_dim * p->dim，初始化为 0
w->w3 = new float[p->n_layers * p->hidden_dim * p->dim]();
// 打印分配内存空间的信息
printf("[%s:AK] Allocating [%d] x [%d] x [%d] = [%d] float space for w->w3\n",__func__,p->n_layers, p->hidden_dim, p->dim, p->n_layers * p->hidden_dim * p->dim);

// 为 w->rms_final_weight 分配内存空间，大小为 p->dim，初始化为 0
w->rms_final_weight = new float[p->dim]();
// 打印分配内存空间的信息
printf("[%s:AK] Allocating [%d] float space for w->rms_final_weight\n",__func__,p->dim);

// 如果 shared_weights 为真，则将 w->wcls 设置为 NULL，否则为 w->wcls 分配内存空间，大小为 p->vocab_size * p->dim，初始化为 0
if (shared_weights) {
    w->wcls = NULL;
} else {
    w->wcls = new float[p->vocab_size * p->dim]();
    // 打印分配内存空间的信息
    printf("[%s:AK] Allocating [%d] x [%d] = [%d] float space for w->wcls\n",__func__,p->vocab_size , p->dim, p->vocab_size * p->dim);
}
// 初始化模型权重，从文件中读取权重数据并存储到相应的权重变量中
static int checkpoint_init_weights(TransformerWeights *w, Config* p, FILE* f, bool shared_weights) {
    // 读取 token_embedding_table 数据并存储到 w->token_embedding_table 中
    if (fread(w->token_embedding_table, sizeof(float), p->vocab_size * p->dim, f) != static_cast<size_t>(p->vocab_size * p->dim)) return 1;
    // 读取 rms_att_weight 数据并存储到 w->rms_att_weight 中
    if (fread(w->rms_att_weight, sizeof(float), p->n_layers * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim)) return 1;
    // 读取 wq 数据并存储到 w->wq 中
    if (fread(w->wq, sizeof(float), p->n_layers * p->dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->dim)) return 1;
    // 读取 wk 数据并存储到 w->wk 中
    if (fread(w->wk, sizeof(float), p->n_layers * p->dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->dim)) return 1;
    // 读取 wv 数据并存储到 w->wv 中
    if (fread(w->wv, sizeof(float), p->n_layers * p->dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->dim)) return 1;
    // 读取 wo 数据并存储到 w->wo 中
    if (fread(w->wo, sizeof(float), p->n_layers * p->dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->dim)) return 1;
    // 读取 rms_ffn_weight 数据并存储到 w->rms_ffn_weight 中
    if (fread(w->rms_ffn_weight, sizeof(float), p->n_layers * p->dim, f) != static_cast<size_t>(p->n_layers * p->dim)) return 1;
    // 读取 w1 数据并存储到 w->w1 中
    if (fread(w->w1, sizeof(float), p->n_layers * p->dim * p->hidden_dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->hidden_dim)) return 1;
    // 读取 w2 数据并存储到 w->w2 中
    if (fread(w->w2, sizeof(float), p->n_layers * p->hidden_dim * p->dim, f) != static_cast<size_t>(p->n_layers * p->hidden_dim * p->dim)) return 1;
    // 读取 w3 数据并存储到 w->w3 中
    if (fread(w->w3, sizeof(float), p->n_layers * p->dim * p->hidden_dim, f) != static_cast<size_t>(p->n_layers * p->dim * p->hidden_dim)) return 1;
    // 读取 rms_final_weight 数据并存储到 w->rms_final_weight 中
    if (fread(w->rms_final_weight, sizeof(float), p->dim, f) != static_cast<size_t>(p->dim)) return 1;

    // 跳过 freq_cis_real & freq_cis_imag 数据
    int head_size = p->dim / p->n_heads;
    fseek(f, p->seq_len * head_size * sizeof(float), SEEK_CUR);

    // 如果不是共享权重，读取 wcls 数据并存储到 w->wcls 中
    if (!shared_weights && fread(w->wcls, sizeof(float), p->vocab_size * p->dim, f) != static_cast<size_t>(p->vocab_size * p->dim)) return 1;

    // 检查是否有遗漏未读取的数据
    // 获取当前文件指针的位置
    auto curr = ftell(f);
    // 将文件指针移动到文件末尾
    fseek(f, 0, SEEK_END);
    // 获取文件末尾的位置
    auto end = ftell(f);
    // 如果当前位置和文件末尾位置不相等，则输出错误信息并返回1
    if (curr != end) {
        printf("Error: failed to read the checkpoint file to the end (curr = %ld, end =  %ld)\n", curr, end);
        return 1;
    }

    // 返回0表示正常执行
    return 0;
}

static void print_sample_weights(TransformerWeights *w){
    // 打印所有变量的第一个权重值
    printf("----- Quick print of first of the weight vales of all the variables\n");
    printf("%f\n", w->token_embedding_table[0]);
    printf("%f\n", w->rms_att_weight[0]);
    printf("%f\n", w->rms_ffn_weight[0]);

    printf("%f\n", w->wq[0]);
    printf("%f\n", w->wk[0]);
    printf("%f\n", w->wv[0]);
```

    // 打印 w 结构体中的 wo[0] 元素
    printf("%f\n", w->wo[0]);
    // 打印 w 结构体中的 w1[0] 元素
    printf("%f\n", w->w1[0]);
    // 打印 w 结构体中的 w2[0] 元素
    printf("%f\n", w->w2[0]);
    // 打印 w 结构体中的 w3[0] 元素
    printf("%f\n", w->w3[0]);
    // 打印 w 结构体中的 rms_att_weight[0] 元素
    printf("%f\n", w->rms_att_weight[0]);
    // 如果 w 结构体中的 wcls 存在，则打印 wcls[0] 元素
    if (w->wcls) printf("%f\n", w->wcls[0]);
}
////////////////////////////////////////////////////////////////////////////////////////////////////////////

//////////////////////////////////////// ggml structs and functions required to load models, configs and save the model.

// 定义 llama_vocab 结构体
struct llama_vocab {
    using id    = int32_t;  // 定义 id 类型为 int32_t
    using token = std::string;  // 定义 token 类型为 std::string
    using ttype = llama_token_type;  // 定义 ttype 类型为 llama_token_type

    // 定义 token_data 结构体
    struct token_data {
        token text;  // 文本数据
        float score;  // 分数
        ttype type;  // 类型
// 定义一个结构体 my_llama_hparams，包含了模型的超参数
struct my_llama_hparams {
    // 词汇表大小
    uint32_t n_vocab = 32000;
    // 上下文大小，用户输入？
    uint32_t n_ctx   = 512;
    // 嵌入维度
    uint32_t n_embd  = 4096;
    // 前馈网络的隐藏层大小
    uint32_t n_ff    = 11008;
    // 多头注意力机制中的头数
    uint32_t n_mult  = 4;
    // 注意力机制中的头数
    uint32_t n_head  = 32;
    // 层数
    uint32_t n_layer = 32;
    // 旋转数
    uint32_t n_rot   = 64;
    // 重载不等号操作符，用于比较两个超参数对象是否相等
    bool operator!=(const my_llama_hparams& other) const {
        return memcmp(this, &other, sizeof(my_llama_hparams));
    }
};

// 定义一个结构体 token_data，用于存储 token 的相关数据
struct token_data {
    // ...
};

// 定义一个结构体 token_id_map，包含了 token 到 id 的映射和 id 到 token 的映射
struct token_id_map {
    // ...
    // token 到 id 的映射
    std::unordered_map<token, id> token_to_id;
    // id 到 token 的映射
    std::vector<token_data> id_to_token;
    // ...
};
// 定义了一个名为my_llama_layer的结构体，用于存储LLAMA模型的层信息
struct my_llama_layer {
    // 存储注意力层的归一化数据
    struct ggml_tensor * attention_norm;

    // 存储注意力层的权重矩阵
    struct ggml_tensor * wq;
    struct ggml_tensor * wk;
    struct ggml_tensor * wv;
    struct ggml_tensor * wo;

    // 存储前馈神经网络层的归一化数据
    struct ggml_tensor * ffn_norm;

    // 存储前馈神经网络层的权重矩阵
    struct ggml_tensor * w1;
    struct ggml_tensor * w2;
    struct ggml_tensor * w3;
};

// 定义了一个名为my_llama_model的结构体，用于存储LLAMA模型的整体信息
struct my_llama_model {
# 声明一个指向 ggml_context 结构体的指针，并初始化为 NULL
struct ggml_context * ctx = NULL;

# 声明一个字符串变量 name
std::string name;

# 声明一个名为 hparams 的 my_llama_hparams 结构体变量
my_llama_hparams hparams;

# 声明两个指向 ggml_tensor 结构体的指针变量
struct ggml_tensor * tok_embeddings;
struct ggml_tensor * norm;
struct ggml_tensor * output;

# 声明一个存储 my_llama_layer 结构体的向量
std::vector<my_llama_layer> layers;

# 声明三个无符号整数变量
uint32_t train_its = 0;
uint32_t train_samples = 0;
uint32_t train_tokens = 0;
};

# 声明一个名为 train_params 的结构体，包含一个指向常量字符的指针变量 fn_vocab_model
struct train_params {
    const char * fn_vocab_model;
// 定义指向 llama2c 模型文件名的指针
const char * fn_llama2c_model;
// 定义指向 llama2c 输出模型文件名的指针
const char * fn_llama2c_output_model;
// 定义指向训练数据文件名的指针
const char * fn_train_data;
// 定义指向检查点输入文件名的指针
const char * fn_checkpoint_in;
// 定义指向检查点输出文件名的指针
const char * fn_checkpoint_out;
// 定义指向模型输出文件名的指针
const char * fn_model_out;

// 定义种子值
uint32_t seed;

// 定义上下文大小
int n_ctx;
// 定义嵌入维度
int n_embd;
// 定义多头数
int n_mult;
// 定义头数
int n_head;
// 定义层数
int n_layer;
// 定义旋转最大值
int n_rotmax;

// 定义线程数
int n_threads;
// 定义批次大小
int n_batch;
// 定义样本数
int n_examples;
// 定义预测数
int n_predict;
    // 打印信息的间隔
    int print_info_interval;
    // 打印详细信息的间隔
    int print_details_interval;

    // 样本是否从换行符后开始
    bool samples_start_after_nl;
    // 是否使用 Adam 优化器
    bool use_adam;
    // 是否使用 Flash
    bool use_flash;
    // 是否使用 Scratch
    bool use_scratch;

    // 仅适用于 Adam 优化器
    // 热身步数
    int   warmup;
    // 余弦衰减步数
    int   cos_decay_steps;
    // 余弦衰减重启
    float cos_decay_restart;
    // 余弦衰减参数
    float cos_decay_alpha;

    // LBFGS 迭代次数
    int   lbfgs_n_iter;
    // Adam 迭代次数
    int   adam_n_iter;
    // Adam 学习率
    float adam_alpha;
    // Adam 衰减率
    float adam_decay;
// 定义整型变量，用于存储内存模型的大小
int mem_model_gb;
// 定义整型变量，用于存储内存计算的大小
int mem_compute_gb;
// 定义整型变量，用于存储内存计算0的大小
int mem_compute0_gb;
// 定义整型变量，用于存储内存计算1的大小
int mem_compute1_gb;
};

// 打印参数结构体中的各项参数值
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

// 初始化模型结构体中的参数
static void init_model(struct my_llama_model * model) {
    // 获取模型结构体中的参数结构体
    const auto & hparams = model->hparams;
    // 定义并初始化模型的嵌入维度、层数和词汇量大小
    const uint32_t n_embd  = hparams.n_embd;
    const uint32_t n_layer = hparams.n_layer;
    const uint32_t n_vocab = hparams.n_vocab;

    // 定义并初始化模型的前馈神经网络大小
    const uint32_t n_ff = hparams.n_ff;
    // 获取模型的上下文
    struct ggml_context * ctx = model->ctx;

    // 初始化模型的训练迭代次数、样本数和标记数
    model->train_its = 0;
    model->train_samples = 0;
    model->train_tokens = 0;

    // 为模型的词嵌入分配内存空间
    model->tok_embeddings = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab);
    printf("[%s:GG] Allocating [%d] x [%d] = [%d] float space for model->tok_embeddings\n",__func__,n_embd , n_vocab, n_embd * n_vocab);

    // 为模型的归一化参数分配内存空间
    model->norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);
    printf("[%s:GG] Allocating [%d] float space for model->norm\n",__func__,n_embd);

    // 为模型的输出层分配内存空间
    model->output = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_vocab);
    printf("[%s:GG] Allocating [%d] x[%d] = [%d] float space for model->output\n",__func__,n_embd, n_vocab, n_embd * n_vocab);
// 打印每层分配的空间，以便在循环中不重复打印
printf("[%s:GG] 为 layer.wq 分配 [%d] x[%d] = [%d] 个浮点数的空间，共 [%d] 层\n",__func__, n_embd, n_embd, n_embd * n_embd, n_layer);
printf("[%s:GG] 为 layer.wk 分配 [%d] x[%d] = [%d] 个浮点数的空间，共 [%d] 层\n",__func__, n_embd, n_embd, n_embd * n_embd, n_layer);
printf("[%s:GG] 为 layer.wv 分配 [%d] x[%d] = [%d] 个浮点数的空间，共 [%d] 层\n",__func__, n_embd, n_embd, n_embd * n_embd, n_layer);
printf("[%s:GG] 为 layer.wo 分配 [%d] x[%d] = [%d] 个浮点数的空间，共 [%d] 层\n",__func__, n_embd, n_embd, n_embd * n_embd, n_layer);

printf("[%s:GG] 为 layer.ffn_norm 分配 [%d] 个浮点数的空间，共 [%d] 层\n",__func__,n_embd, n_layer);

printf("[%s:GG] 为 layer.w1 分配 [%d] x[%d] = [%d] 个浮点数的空间，共 [%d] 层\n",__func__, n_ff, n_embd, n_embd * n_ff, n_layer);
printf("[%s:GG] 为 layer.w2 分配 [%d] x[%d] = [%d] 个浮点数的空间，共 [%d] 层\n",__func__, n_embd, n_ff, n_ff * n_embd, n_layer);
printf("[%s:GG] 为 layer.w3 分配 [%d] x[%d] = [%d] 个浮点数的空间，共 [%d] 层\n",__func__, n_ff, n_embd, n_embd * n_ff, n_layer);

// 设置模型中的权重名称
ggml_set_name(model->tok_embeddings, "tok_embeddings.weight");
ggml_set_name(model->norm,           "norm.weight");
ggml_set_name(model->output,         "output.weight");

// 调整模型中的层数
model->layers.resize(n_layer);
for (uint32_t i = 0; i < n_layer; ++i) {
    auto & layer = model->layers[i];
// 创建一个名为 layers_i 的字符串，其值为 "layers." 加上当前循环变量 i 的字符串形式
std::string layers_i = "layers." + std::to_string(i);

// 为当前层创建 attention_norm 张量，数据类型为 GGML_TYPE_F32，大小为 n_embd
layer.attention_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

// 为当前层创建 wq 张量，数据类型为 GGML_TYPE_F32，大小为 n_embd x n_embd
layer.wq = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
// 为当前层创建 wk 张量，数据类型为 GGML_TYPE_F32，大小为 n_embd x n_embd
layer.wk = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
// 为当前层创建 wv 张量，数据类型为 GGML_TYPE_F32，大小为 n_embd x n_embd
layer.wv = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);
// 为当前层创建 wo 张量，数据类型为 GGML_TYPE_F32，大小为 n_embd x n_embd
layer.wo = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_embd);

// 为当前层创建 ffn_norm 张量，数据类型为 GGML_TYPE_F32，大小为 n_embd
layer.ffn_norm = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_embd);

// 为当前层创建 w1 张量，数据类型为 GGML_TYPE_F32，大小为 n_embd x n_ff
layer.w1 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);
// 为当前层创建 w2 张量，数据类型为 GGML_TYPE_F32，大小为 n_ff x n_embd
layer.w2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_ff, n_embd);
// 为当前层创建 w3 张量，数据类型为 GGML_TYPE_F32，大小为 n_embd x n_ff
layer.w3 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, n_embd, n_ff);

// 为当前层的 attention_norm 张量设置名称
ggml_set_name(layer.attention_norm, (layers_i + ".attention_norm.weight").c_str());

// 为当前层的 wq 张量设置名称
ggml_set_name(layer.wq, (layers_i + ".attention.wq.weight").c_str());
// 为当前层的 wk 张量设置名称
ggml_set_name(layer.wk, (layers_i + ".attention.wk.weight").c_str());
// 为当前层的 wv 张量设置名称
ggml_set_name(layer.wv, (layers_i + ".attention.wv.weight").c_str());
// 设置层的名称为指定的字符串
ggml_set_name(layer.wo, (layers_i + ".attention.wo.weight").c_str());

// 设置层的名称为指定的字符串
ggml_set_name(layer.ffn_norm, (layers_i + ".ffn_norm.weight").c_str());

// 格式化层的名称为指定的字符串
ggml_format_name(layer.w1, "%s.feed_forward.w1.weight", layers_i.c_str());
ggml_format_name(layer.w2, "%s.feed_forward.w2.weight", layers_i.c_str());
ggml_format_name(layer.w3, "%s.feed_forward.w3.weight", layers_i.c_str());

// 获取二维浮点型张量的值
static float get_f32_2d(struct ggml_tensor * tensor, int64_t i0, int64_t i1) {
    float * ptr = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1]);
    return *ptr;
}

// 获取二维整型张量的值
static int32_t get_i32_2d(struct ggml_tensor * tensor, int64_t i0, int64_t i1) {
    int32_t * ptr = (int32_t *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1]);
    return *ptr;
}
# 打印指定行的概率张量数据
static void print_row(struct ggml_tensor * probs, int i) {
    # 遍历指定行的每个元素，获取并打印概率值
    for (int k = 0; k < probs->ne[0]; ++k) {
        float p = get_f32_2d(probs, k, i);
        printf(" %f", p);
    }
    # 换行
    printf("\n");
}

# 打印整个概率张量的数据
static void print_matrix(struct ggml_tensor * probs) {
    # 确保概率张量的维度为2
    assert(probs->n_dims == 2);
    # 遍历每一行，打印每个元素的概率值
    for (int i = 0; i < probs->ne[1]; ++i) {
        for (int k = 0; k < probs->ne[0]; ++k) {
            float p = get_f32_2d(probs, k, i);
            printf(" %.2f", p);
        }
        # 换行
        printf("\n");
    }
}

# 条件编译，判断是否为 GNU 编译器
#ifdef __GNUC__
#ifdef __MINGW32__
// 如果是在 MinGW32 编译环境下，使用 GNU printf 格式
__attribute__((format(gnu_printf, 1, 2)))
#else
// 否则，使用标准 printf 格式
__attribute__((format(printf, 1, 2)))
#endif
#endif
// 定义一个静态函数，用于格式化字符串
static std::string format(const char * fmt, ...) {
    // 定义两个可变参数列表
    va_list ap, ap2;
    // 初始化第一个可变参数列表
    va_start(ap, fmt);
    // 复制第一个可变参数列表到第二个
    va_copy(ap2, ap);
    // 计算格式化后的字符串长度
    int size = vsnprintf(NULL, 0, fmt, ap);
    // 断言字符串长度大于等于 0 且小于 INT_MAX
    GGML_ASSERT(size >= 0 && size < INT_MAX);
    // 创建一个字符向量，用于存储格式化后的字符串
    std::vector<char> buf(size + 1);
    // 格式化字符串并将结果存储到字符向量中
    int size2 = vsnprintf(buf.data(), size + 1, fmt, ap2);
    // 断言格式化后的字符串长度与之前计算的长度相等
    GGML_ASSERT(size2 == size);
    // 结束第二个可变参数列表
    va_end(ap2);
    // 结束第一个可变参数列表
    va_end(ap);
    // 返回格式化后的字符串
    return std::string(buf.data(), size);
}
// 定义一个结构体 llama_file，用于表示文件信息
struct llama_file {
    // 使用 FILE * 类型的指针，以便我们不必重新打开文件来进行内存映射
    FILE * fp;
    size_t size;

    // 构造函数，根据文件名和打开模式打开文件，并获取文件大小
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
            // 定位到文件开头
            seek(0, SEEK_SET);
        }
    }

    // 获取当前文件指针的位置
    size_t tell() const {
        // 根据操作系统不同调用不同的 tell 函数
#ifdef _WIN32
        __int64 ret = _ftelli64(fp);
#else
    // 获取当前文件指针的位置
    long ret = std::ftell(fp);
#endif
    // 断言当前文件指针的位置不为-1，如果为-1则输出错误信息
    GGML_ASSERT(ret != -1); // this really shouldn't fail
    // 返回当前文件指针的位置
    return (size_t) ret;
}

// 根据给定的偏移量和方式移动文件指针
void seek(size_t offset, int whence) {
#ifdef _WIN32
    // 在 Windows 平台下移动文件指针
    int ret = _fseeki64(fp, (__int64) offset, whence);
#else
    // 在其他平台下移动文件指针
    int ret = std::fseek(fp, (long) offset, whence);
#endif
    // 断言移动文件指针操作是否成功，如果不成功则输出错误信息
    GGML_ASSERT(ret == 0); // same
}

// 从文件中读取原始数据
void read_raw(void * ptr, size_t size) {
    // 如果要读取的数据大小为0，则直接返回
    if (size == 0) {
        return;
    }
    // 将错误号设置为0
    errno = 0;
// 从文件中读取数据到指定的内存位置，返回成功读取的元素个数
std::size_t ret = std::fread(ptr, size, 1, fp);
// 检查文件流是否出错，如果出错则打印错误信息并退出程序
if (ferror(fp)) {
    die_fmt("fread failed: %s", strerror(errno));
}
// 检查实际读取的元素个数是否和预期一致，如果不一致则打印错误信息并退出程序
if (ret != 1) {
    die("unexpectedly reached end of file");
}

// 读取一个 32 位无符号整数
std::uint32_t read_u32() {
    std::uint32_t ret;
    read_raw(&ret, sizeof(ret));
    return ret;
}

// 读取一个 32 位浮点数
std::float_t read_f32() {
    std::float_t ret;
    read_raw(&ret, sizeof(ret));
    return ret;
}
// 从文件中读取指定长度的字符串并返回
std::string read_string(std::uint32_t len) {
    // 创建一个字符向量，大小为指定长度
    std::vector<char> chars(len);
    // 从文件中读取指定长度的原始数据到字符向量中
    read_raw(chars.data(), len);
    // 使用字符向量中的数据创建字符串并返回
    return std::string(chars.data(), len);
}

// 析构函数，用于关闭文件指针
~llama_file() {
    // 如果文件指针存在，则关闭文件
    if (fp) {
        std::fclose(fp);
    }
}

// 静态函数，用于判断文件是否为特定类型的文件
static bool is_ggml_file(const char * filename) {
    // 创建 llama_file 对象，以只读方式打开指定文件
    llama_file file(filename, "rb");
    // 如果文件大小小于4，则不是指定类型的文件
    if (file.size < 4) {
        return false;
    }
    // 从文件中读取4个字节的数据作为魔术字符串
    std::string magic = file.read_string(4);
    // 判断魔术字符串是否与指定的魔术字符串相等
    return magic == GGUF_MAGIC;
}
// 静态函数，用于将输入的字符串中的空格替换为特定的字符串
static std::string llama_escape_whitespaces(const std::string & text) {
    // 创建一个字符串输出流
    std::ostringstream out;
    // 遍历输入的字符串
    for (char c : text) {
        // 如果字符是空格，则将特定的字符串写入输出流
        if (c == ' ') out << "\xe2\x96\x81";
        // 否则将字符本身写入输出流
        else out << c;
    }
    // 返回输出流中的字符串
    return out.str();
}

// 静态函数，用于加载词汇表
static void load_vocab(const char *filename, Config *config, struct llama_vocab *vocab) {
    // 如果输入的文件是 ggml 文件
    if (is_ggml_file(filename)) {
        // 初始化 ggml 上下文数据
        struct ggml_context * ctx_data = NULL;

        // 设置 gguf 初始化参数
        struct gguf_init_params params = {
            /*.no_alloc = */ false,  // 是否需要分配内存
            /*.ctx      = */ &ctx_data,  // 上下文数据
        };
// 从文件中初始化 gguf_context 结构体
struct gguf_context * ctx = gguf_init_from_file(filename, params);
// 断言 ctx 不为空
GGML_ASSERT(ctx != NULL);

// 查找键为 KV_TOKENIZER_MODEL 的索引
const int model_idx = gguf_find_key(ctx, KV_TOKENIZER_MODEL);
// 断言 model_idx 大于等于 0
GGML_ASSERT(model_idx >= 0);
// 获取键为 KV_TOKENIZER_MODEL 的值，并赋值给 tokenizer_name
std::string tokenizer_name = gguf_get_val_str(ctx, model_idx);
// 断言 tokenizer_name 等于 TOKENIZER_NAME
GGML_ASSERT(tokenizer_name == TOKENIZER_NAME);

// 查找键为 KV_TOKENIZER_LIST 的索引
const int token_idx = gguf_find_key(ctx, KV_TOKENIZER_LIST);
// 断言 token_idx 大于等于 0
GGML_ASSERT(token_idx >= 0);

// 查找键为 KV_TOKENIZER_SCORES 的索引
const int score_idx = gguf_find_key(ctx, KV_TOKENIZER_SCORES);
// 断言 score_idx 大于等于 0
GGML_ASSERT(score_idx >= 0);
// 获取键为 KV_TOKENIZER_SCORES 的值，并将其转换为 float 类型的指针
const float * scores = (const float * ) gguf_get_arr_data(ctx, score_idx);

// 查找键为 KV_TOKENIZER_TOKEN_TYPE 的索引
const int toktype_idx = gguf_find_key(ctx, KV_TOKENIZER_TOKEN_TYPE);
// 断言 toktype_idx 大于等于 0
GGML_ASSERT(toktype_idx >= 0);
// 获取键为 KV_TOKENIZER_TOKEN_TYPE 的值，并将其转换为 int 类型的指针
const int * toktypes = (const int * ) gguf_get_arr_data(ctx, toktype_idx);

// 获取键为 KV_TOKENIZER_LIST 的值的数组长度，并赋值给 n_vocab
const uint32_t n_vocab = gguf_get_arr_n(ctx, token_idx);
        // 调整词汇表的大小为 n_vocab
        vocab->id_to_token.resize(n_vocab);

        // 遍历词汇表
        for (uint32_t i = 0; i < n_vocab; i++) {
            // 从上下文中获取单词
            std::string word = gguf_get_arr_str(ctx, token_idx, i);

            // 将单词与对应的索引存入词汇表中
            vocab->token_to_id[word] = i;

            // 获取词汇表中索引为 i 的单词数据
            auto & token_data = vocab->id_to_token[i];
            // 将单词移动到 token_data.text 中
            token_data.text  = std::move(word);
            // 存储单词的分数
            token_data.score = scores[i];
            // 存储单词的类型
            token_data.type  = (llama_token_type) toktypes[i];
        }
        // 释放上下文数据
        ggml_free(ctx_data);
        // 释放上下文
        gguf_free(ctx);
    } else {
        // 假设是 llama2.c 词汇表
        printf("Assuming llama2.c vocabulary since %s is not a gguf file\n", filename);
        // 以只读方式打开文件
        llama_file file(filename, "rb");
        // 如果文件指针为空
        if (!file.fp) {
// 如果发生错误，使用错误号和文件名格式化错误信息并终止程序
die_fmt("%s: %s", strerror(errno), filename);
// 获取词汇表大小
const int  n_vocab = config->vocab_size;
// 读取一个无用的 uint32_t 类型数据，但未使用
/* uint32_t max_token_length =  */ file.read_u32(); // unused
// 调整词汇表的 id_to_token 大小
vocab->id_to_token.resize(n_vocab);
// 遍历词汇表 id
for (llama_vocab::id id=0; id<n_vocab; ++id) {
    // 读取一个 float_t 类型的分数
    float_t score = file.read_f32();
    // 读取一个 uint32_t 类型的长度
    uint32_t len = file.read_u32();
    // 读取指定长度的字符串
    std::string text = file.read_string(len);

    // 定义变量
    unsigned char byte_val;
    llama_vocab::ttype type = LLAMA_TOKEN_TYPE_NORMAL;
    // 根据 id 判断特殊标记
    if (id == UNKNOWN_TOKEN_ID) {
        text = "<unk>";
        type = LLAMA_TOKEN_TYPE_UNKNOWN;
    } else if (id == BOS_TOKEN_ID) {
        text = "<s>";
        type = LLAMA_TOKEN_TYPE_CONTROL;
    } else if (id == EOS_TOKEN_ID) {
        text = "</s>";
// 设置类型为控制类型
type = LLAMA_TOKEN_TYPE_CONTROL;
// 如果文本为空，则设置类型为控制类型
} else if (text.empty()) {
    type = LLAMA_TOKEN_TYPE_CONTROL;
// 如果文本不为空且符合特定格式，则设置类型为字节类型
} else if (sscanf(text.c_str(), "<0x%02hhX>", &byte_val) == 1) {
    // 字节标记的文本已经是预期的格式
    type = LLAMA_TOKEN_TYPE_BYTE;
// 否则设置类型为普通类型
} else {
    type = LLAMA_TOKEN_TYPE_NORMAL;
}
// 对文本进行空白字符转义处理
text = llama_escape_whitespaces(text);

// 将文本、分数和类型存储到词汇表中
vocab->id_to_token[id].text = text;
vocab->id_to_token[id].score = score;
vocab->id_to_token[id].type = type;
// 将文本和 ID 存储到词汇表的映射中
vocab->token_to_id.emplace(text, id);
# 定义整型变量 ct
int ct;
# 根据 gg_weights 的维度数量进行不同的处理
switch (gg_weights->n_dims){
    # 当维度数量为 1 时
    case 1:
        # 初始化 ct 为 0
        ct = 0;
        # 遍历第一维度上的元素
        for (int i0 = 0; i0 < gg_weights->ne[0]; i0++){
            # 计算指向当前元素数据的指针
            float * ptr = (float *) ((char *) gg_weights->data + i0*gg_weights->nb[0]);
            # 将 karpathy_weights 中的值赋给指针指向的数据
            *ptr = karpathy_weights[ct];
            # 更新 ct
            ct++;
        }
        break;
    # 当维度数量为 2 时
    case 2:
        # 初始化 ct 为 0
        ct = 0;
        # 遍历第二维度上的元素
        for (int i1 = 0; i1 < gg_weights->ne[1]; i1++) {
            # 遍历第一维度上的元素
            for (int i0 = 0; i0 < gg_weights->ne[0]; i0++) {
                # 计算指向当前元素数据的指针
                float * ptr = (float *) ((char *) gg_weights->data + i0*gg_weights->nb[0] + i1*gg_weights->nb[1]);
                # 将 karpathy_weights 中的值赋给指针指向的数据
                *ptr = karpathy_weights[ct];
                # 更新 ct
                ct++;
            }
        }
        break;
// 根据不同的情况进行处理
case 3:
    // 初始化计数器
    ct = 0;
    // 遍历三维数组
    for (int i2 = 0; i2 < gg_weights->ne[2]; i2++) {
        for (int i1 = 0; i1 < gg_weights->ne[1]; i1++) {
            for (int i0 = 0; i0 < gg_weights->ne[0]; i0++) {
                // 计算指针位置并赋值
                float * ptr = (float *) ((char *) gg_weights->data + i0*gg_weights->nb[0] + i1*gg_weights->nb[1] + i2*gg_weights->nb[2]);
                *ptr = karpathy_weights[ct];
                // 更新计数器
                ct++;
            }
        }
    }
    // 结束当前情况的处理
    break;
}

// 将AK权重逐个转换为GG权重
static void save_as_llama_model(
    struct llama_vocab * vocab, struct my_llama_model * model, TransformerWeights* w, const char * filename
) {
    // 将AK权重转换为GG权重，存储在模型中
    w->token_embedding_table -> model->tok_embeddings
    // 将模型中的权重数据转换为另一种数据结构
    convert_weights_ak_to_gg(model->tok_embeddings, w->token_embedding_table);
    convert_weights_ak_to_gg(model->output, w->wcls ? w->wcls : w->token_embedding_table);

    // 将模型中的权重数据转换为另一种数据结构
    convert_weights_ak_to_gg(model->norm, w->rms_final_weight);
    // 打印模型中的权重数据
    //print_row(model->norm, 0);

    // 为 rms-att-weight 进行循环操作
    int row_length = model->hparams.n_embd;
    int n_ff = model->hparams.n_ff;

    for (uint32_t i = 0; i < model->hparams.n_layer; ++i){
        auto & layer = model->layers[i];
        // 将模型中的权重数据转换为另一种数据结构
        convert_weights_ak_to_gg(layer.attention_norm, &w->rms_att_weight[i*row_length]);
        convert_weights_ak_to_gg(layer.ffn_norm      , &w->rms_ffn_weight[i*row_length]);

        // 将模型中的权重数据转换为另一种数据结构，从 3D 矩阵 layer x dim x dim 转换为 2D 矩阵 dim x dim
        convert_weights_ak_to_gg(layer.wq            , &w->wq[i*row_length*row_length]);
        convert_weights_ak_to_gg(layer.wk            , &w->wk[i*row_length*row_length]);
    // 将权重数据从ak格式转换为gg格式，并存储到相应的位置
    convert_weights_ak_to_gg(layer.wv            , &w->wv[i*row_length*row_length]);
    convert_weights_ak_to_gg(layer.wo            , &w->wo[i*row_length*row_length]);

    convert_weights_ak_to_gg(layer.w1            , &w->w1[i*row_length*n_ff]);
    convert_weights_ak_to_gg(layer.w2            , &w->w2[i*n_ff*row_length]);
    convert_weights_ak_to_gg(layer.w3            , &w->w3[i*row_length*n_ff]);
    }

    // 初始化一个gguf上下文
    struct gguf_context * ctx = gguf_init_empty();

    // 创建存储token数据的向量
    std::vector<const char*> tokens;
    std::vector<float> scores;
    std::vector<llama_token_type> token_types;
    // 遍历词汇表中的token数据，将其存储到相应的向量中
    for (const llama_vocab::token_data & token_data : vocab->id_to_token) {
        tokens.push_back(token_data.text.c_str());
        scores.push_back(token_data.score);
        token_types.push_back(token_data.type);
    }
    // 将token数据存储到gguf上下文中
    gguf_set_arr_str(ctx, KV_TOKENIZER_LIST, tokens.data(), tokens.size());
    gguf_set_arr_data(ctx, KV_TOKENIZER_SCORES, GGUF_TYPE_FLOAT32, scores.data(), scores.size());
// 使用 gguf_set_arr_data 函数设置 ctx 中的 KV_TOKENIZER_TOKEN_TYPE 键对应的值为 GGUF_TYPE_INT32 类型的数组 token_types
gguf_set_arr_data(ctx, KV_TOKENIZER_TOKEN_TYPE, GGUF_TYPE_INT32, token_types.data(), token_types.size());

// 使用 gguf_set_val_str 函数设置 ctx 中的 KV_TOKENIZER_MODEL 键对应的值为 TOKENIZER_NAME 字符串
gguf_set_val_str(ctx, KV_TOKENIZER_MODEL, TOKENIZER_NAME);

// 使用 gguf_set_val_str 函数设置 ctx 中的 KV_GENERAL_ARCHITECTURE 键对应的值为 "llama" 字符串
gguf_set_val_str(ctx, KV_GENERAL_ARCHITECTURE, "llama");
// 使用 gguf_set_val_str 函数设置 ctx 中的 KV_GENERAL_NAME 键对应的值为 "llama" 字符串
gguf_set_val_str(ctx, KV_GENERAL_NAME, "llama");

// 设置特殊标记的 ID 值
gguf_set_val_u32(ctx, KV_TOKENIZER_UNK_ID, UNKNOWN_TOKEN_ID);
gguf_set_val_u32(ctx, KV_TOKENIZER_BOS_ID, BOS_TOKEN_ID);
gguf_set_val_u32(ctx, KV_TOKENIZER_EOS_ID, EOS_TOKEN_ID);
gguf_set_val_u32(ctx, KV_TOKENIZER_SEP_ID, -1);
gguf_set_val_u32(ctx, KV_TOKENIZER_PAD_ID, -1);

// 设置模型参数的长度值
gguf_set_val_u32(ctx, KV_CONTEXT_LENGTH, model->hparams.n_ctx);
gguf_set_val_u32(ctx, KV_EMBEDDING_LENGTH, model->hparams.n_embd);
gguf_set_val_u32(ctx, KV_FEED_FORWARD_LENGTH, model->hparams.n_ff);
gguf_set_val_u32(ctx, KV_ATTENTION_HEAD_COUNT, model->hparams.n_head);
// n_head_kv 是可选的，如果没有设置则默认为 n_head
// gguf_set_val_u32(ctx, KV_ATTENTION_HEAD_COUNT_KV, ...);
    // 设置上下文中的键值对，将模型的层数存储到上下文中
    gguf_set_val_u32(ctx, KV_BLOCK_COUNT, model->hparams.n_layer);
    // 设置上下文中的键值对，将模型的旋转数存储到上下文中
    gguf_set_val_u32(ctx, KV_ROPE_DIMENSION_COUNT, model->hparams.n_rot);
    // 设置上下文中的键值对，将注意力层归一化的 RMS 误差存储到上下文中
    gguf_set_val_f32(ctx, KV_ATTENTION_LAYERNORM_RMS_EPS, 1e-5f);

    // 写入张量
    // 设置模型的 token embeddings 的名称，并将其添加到上下文中
    ggml_set_name(model->tok_embeddings, TN_TOKEN_EMBD);
    gguf_add_tensor(ctx, model->tok_embeddings);

    // 设置模型的 norm 的名称，并将其添加到上下文中
    ggml_set_name(model->norm, TN_OUTPUT_NORM);
    gguf_add_tensor(ctx, model->norm);

    // 设置模型的 output 的名称，并将其添加到上下文中
    ggml_set_name(model->output, TN_OUTPUT);
    gguf_add_tensor(ctx, model->output);

    // 遍历模型的层数，设置每一层的权重张量的名称，并将其添加到上下文中
    for (uint32_t i = 0; i < model->hparams.n_layer; ++i) {
        auto & layer = model->layers[i];

        ggml_format_name(layer.wq, TN_ATTN_Q, i);
        gguf_add_tensor(ctx, layer.wq);
    }
# 格式化注意力机制的权重名称，将其添加到工作流中
ggml_format_name(layer.wk, TN_ATTN_K, i);
# 将注意力机制的权重添加到上下文中
gguf_add_tensor(ctx, layer.wk);

# 格式化注意力机制的数值名称，将其添加到工作流中
ggml_format_name(layer.wv, TN_ATTN_V, i);
# 将注意力机制的数值添加到上下文中
gguf_add_tensor(ctx, layer.wv);

# 格式化注意力机制的输出名称，将其添加到工作流中
ggml_format_name(layer.wo, TN_ATTN_OUTPUT, i);
# 将注意力机制的输出添加到上下文中
gguf_add_tensor(ctx, layer.wo);

# 格式化注意力机制的归一化名称，将其添加到工作流中
ggml_format_name(layer.attention_norm, TN_ATTN_NORM, i);
# 将注意力机制的归一化值添加到上下文中
gguf_add_tensor(ctx, layer.attention_norm);

# 格式化前馈神经网络的门控权重名称，将其添加到工作流中
ggml_format_name(layer.w1, TN_FFN_GATE, i);
# 将前馈神经网络的门控权重添加到上下文中
gguf_add_tensor(ctx, layer.w1);

# 格式化前馈神经网络的下采样权重名称，将其添加到工作流中
ggml_format_name(layer.w2, TN_FFN_DOWN, i);
# 将前馈神经网络的下采样权重添加到上下文中
gguf_add_tensor(ctx, layer.w2);

# 格式化前馈神经网络的上采样权重名称，将其添加到工作流中
ggml_format_name(layer.w3, TN_FFN_UP, i);
# 将前馈神经网络的上采样权重添加到上下文中
gguf_add_tensor(ctx, layer.w3);
// 调用 ggml_format_name 函数，将 layer.ffn_norm 转换为 TN_FFN_NORM 格式的名称，并存储在 i 索引处
ggml_format_name(layer.ffn_norm, TN_FFN_NORM, i);
// 将 layer.ffn_norm 添加到 gguf 上下文中
gguf_add_tensor(ctx, layer.ffn_norm);
}

// 将 gguf 上下文中的数据写入文件，不追加到已存在的文件中
gguf_write_to_file(ctx, filename, false);
// 释放 gguf 上下文所占用的内存
gguf_free(ctx);
}

// 返回默认的训练参数结构体
static struct train_params get_default_train_params() {
    struct train_params params;
    // 设置默认的模型文件名
    params.fn_vocab_model    = "models/7B/ggml-model-f16.gguf";
    // 设置输出的 llama2c 模型文件名
    params.fn_llama2c_output_model = "ak_llama_model.bin";
    // 设置训练数据文件名
    params.fn_train_data     = "shakespeare.txt";
    // 设置输入的检查点文件名
    params.fn_checkpoint_in  = "checkpoint.bin";
    // 设置输出的检查点文件名
    params.fn_checkpoint_out = "checkpoint.bin";
    // 设置输出的模型文件名
    params.fn_model_out      = "ggml-checkpoint-f32.bin";

    // 设置随机数种子
    params.seed       =   -1;
    # 设置上下文的长度为128
    params.n_ctx      =  128;
    # 设置嵌入的维度为256
    params.n_embd     =  256;
    # 设置多头注意力机制中的向量维度为256
    params.n_mult     =  256;
    # 设置注意力头的数量为8
    params.n_head     =    8;
    # 设置层数为16
    params.n_layer    =   16;
    # 设置旋转最大值为64
    params.n_rotmax   =   64;

    # 设置线程数为6
    params.n_threads  =    6;
    # 设置批处理大小为8
    params.n_batch    =    8;
    # 设置样本数量为8
    params.n_examples =    8;
    # 设置预测数量为1024
    params.n_predict  = 1024;

    # 设置打印信息的间隔为1
    params.print_info_interval    = 1;
    # 设置打印详细信息的间隔为2
    params.print_details_interval = 2;

    # 设置样本开始后是否换行为false
    params.samples_start_after_nl = false;
    # 设置是否使用Adam优化器为true
    params.use_adam               = true;
    # 设置是否使用Flash为true
    params.use_flash              = true;
    # 设置是否使用Scratch为true
    params.use_scratch            = true;
    // 设置参数：预热次数为100次
    params.warmup            =  100;
    // 设置参数：余弦衰减步数为1000步
    params.cos_decay_steps   = 1000;
    // 设置参数：余弦衰减重启系数为1.1
    params.cos_decay_restart = 1.1f;
    // 设置参数：余弦衰减指数为0.0
    params.cos_decay_alpha   = 0.0f;

    // 设置参数：LBFGS 迭代次数为16次
    params.lbfgs_n_iter      = 16;
    // 设置参数：Adam 迭代次数为16次
    params.adam_n_iter       = 16;
    // 设置参数：Adam 学习率为1e-3
    params.adam_alpha        = 1e-3f;
    // 设置参数：Adam 衰减率为1e-3
    params.adam_decay        = 1e-3f;

    // 设置参数：内存模型占用为2GB
    params.mem_model_gb   = 2;
    // 设置参数：计算内存占用为24GB
    params.mem_compute_gb = 24;
    // 设置参数：计算内存初始占用为8GB
    params.mem_compute0_gb = 8;
    // 设置参数：计算内存增量占用为2GB
    params.mem_compute1_gb = 2;

    // 返回参数结构体
    return params;
}

// 打印程序用法
static void print_usage(int /*argc*/, char ** argv, const struct train_params * params) {
    // 打印使用说明
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    // 打印可用选项
    fprintf(stderr, "  -h, --help                       show this help message and exit\n");
    fprintf(stderr, "  --copy-vocab-from-model FNAME    path of gguf llama model or llama2.c vocabulary from which to copy vocab (default '%s')\n", params->fn_vocab_model);
    fprintf(stderr, "  --llama2c-model FNAME            [REQUIRED] model path from which to load Karpathy's llama2.c model\n");
    fprintf(stderr, "  --llama2c-output-model FNAME     model path to save the converted llama2.c model (default %s')\n", params->fn_llama2c_output_model);
    fprintf(stderr, "\n");
}

static bool params_parse(int argc, char ** argv, struct train_params * params) {
    bool invalid_param = false;
    bool reqd_param_found = false;
    std::string arg;
    // 获取默认的训练参数
    struct train_params default_params = get_default_train_params();
    const std::string arg_prefix = "--";

    for (int i = 1; i < argc; i++) {
        arg = argv[i];
        // 检查参数是否以"--"开头
        if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
// 使用标准库函数replace将参数arg中的"_"替换为"-"
std::replace(arg.begin(), arg.end(), '_', '-');

// 如果参数arg为"--copy-vocab-from-model"
if (arg == "--copy-vocab-from-model") {
    // 如果下一个参数存在，则将其作为params->fn_vocab_model的值
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    params->fn_vocab_model = argv[i];
} 
// 如果参数arg为"--llama2c-model"
else if (arg == "--llama2c-model") {
    // 如果下一个参数存在，则将其作为params->fn_llama2c_model的值
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    reqd_param_found = true;
    params->fn_llama2c_model = argv[i];
} 
// 如果参数arg为"--llama2c-output-model"
else if (arg == "--llama2c-output-model") {
    // 如果下一个参数存在，则将其作为params->fn_llama2c_output_model的值
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    }
    // 如果参数为 --llama2c-model，则将下一个参数作为输出模型文件名
    params->fn_llama2c_output_model = argv[i];
} else if (arg == "-h" || arg == "--help") {
    // 如果参数为 -h 或 --help，则打印帮助信息并退出程序
    print_usage(argc, argv, &default_params);
    exit(0);
} else {
    // 如果参数不是已知的选项，则打印错误信息并退出程序
    fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
    print_usage(argc, argv, &default_params);
    exit(1);
}
}
// 如果存在无效参数，则打印错误信息并退出程序
if (invalid_param) {
    fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
    print_usage(argc, argv, &default_params);
    exit(1);
}
// 如果必需参数未找到，则打印错误信息并退出程序
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
    // 如果没有找到斜杠或反斜杠，则返回整个路径
    if (pos == std::string::npos) {
        return path;
    }
    // 返回斜杠或反斜杠后面的部分作为基本名称
    return path.substr(pos + 1);
}

// 主函数
int main(int argc, char ** argv) {
    // 获取默认的训练参数
    struct train_params params = get_default_train_params();
    // 解析命令行参数，如果失败则返回1
    if (!params_parse(argc, argv, &params)) {
        return 1;
    }
    // 创建配置对象
    Config config;
    // 初始化变压器权重
    TransformerWeights weights = {};
    // 打开名为params.fn_llama2c_model的文件，以只读方式
    FILE *file = fopen(params.fn_llama2c_model, "rb");
    // 如果文件打开失败，输出错误信息并返回1
    if (!file) { printf("Unable to open the checkpoint file %s!\n", params.fn_llama2c_model); return 1; }
    // 读取配置头部信息
    if(fread(&config, sizeof(Config), 1, file) != 1) { return 1; }
    // 检查是否共享权重
    auto shared_weights = config.vocab_size > 0;
    // 将vocab_size转换为绝对值
    config.vocab_size = abs(config.vocab_size);

    // 为Transformer权重分配内存
    malloc_weights(&weights, &config, shared_weights);
    // 初始化权重
    if(checkpoint_init_weights(&weights, &config, file, shared_weights)) { return 1; }
    // 关闭文件
    fclose(file);

    // 加载词汇表
    struct llama_vocab vocab;
    load_vocab(params.fn_vocab_model, &config, &vocab);

    // 初始化llama模型
    struct my_llama_model model;
    // 设置模型的词汇表大小
    model.hparams.n_vocab = config.vocab_size; //llama_n_vocab(lctx);
    // 设置模型的上下文大小
    model.hparams.n_ctx   = params.n_ctx;
    // 设置模型的嵌入维度为配置文件中的维度
    model.hparams.n_embd  = config.dim; //params.n_embd;
    // 设置模型的前馈神经网络隐藏层维度为配置文件中的隐藏层维度
    model.hparams.n_ff    = config.hidden_dim;
    // 设置模型的多头注意力机制中的头数为32
    model.hparams.n_mult  = 32;//params.n_mult;
    // 设置模型的注意力头数为配置文件中的注意力头数
    model.hparams.n_head  = config.n_heads; //params.n_head;
    // 设置模型的层数为配置文件中的层数
    model.hparams.n_layer = config.n_layers; //params.n_layer;
    // 设置模型的旋转数为params.n_rotmax和模型的嵌入维度除以注意力头数的较小值
    model.hparams.n_rot   = std::min((uint32_t)params.n_rotmax, model.hparams.n_embd / model.hparams.n_head);
    // 打印模型参数
    print_params(&model.hparams);
    // 初始化本地上下文参数
    struct ggml_init_params lcparams;
    lcparams.mem_size   = 1024ll*1024ll*1024ll*((size_t) params.mem_model_gb);
    lcparams.mem_buffer = NULL;
    lcparams.no_alloc   = false;

    // 为模型分配本地上下文
    model.ctx = ggml_init(lcparams);

    // 初始化模型
    init_model(&model);
    // 设置模型名称为配置文件中的模型文件名
    model.name = basename(params.fn_llama2c_model);
    // 将词汇表、模型和权重保存为llama模型文件
    save_as_llama_model(&vocab, &model, &weights, params.fn_llama2c_output_model);

    // 打印保存的模型文件名和输出路径
    printf("Saving llama.c model file %s in ggml format at %s\n", params.fn_llama2c_model, params.fn_llama2c_output_model);
# 释放 ggml 模型上下文所占用的内存
ggml_free(model.ctx);
# 返回 0，表示程序执行成功
return 0;
```