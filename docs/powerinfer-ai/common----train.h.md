# `PowerInfer\common\train.h`

```cpp
// Various helper functions and utilities for training

// 防止头文件重复包含
#pragma once

// 包含必要的头文件
#include <string>
#include <random>
#include <vector>

// 包含自定义的头文件
#include "ggml.h"
#include "llama.h"

// 定义最大节点数
#define LLAMA_TRAIN_MAX_NODES 16384

// 定义 mt19937 状态的字符串类型
typedef std::string mt19937_state;

// 定义训练状态结构体
struct train_state {
    struct ggml_opt_context * opt;

    uint64_t train_its;
    uint64_t train_samples;
    uint64_t train_tokens;
    uint64_t train_epochs;

    size_t        shuffle_samples_hash; // fn, sample_count, *zip(sample_begins, sample_sizes)
    mt19937_state shuffle_rng_state_current;
    mt19937_state shuffle_rng_state_next;
    size_t        shuffle_sample_count;
    size_t        shuffle_next_sample;
};

// 定义通用训练参数结构体
struct train_params_common {
    const char * fn_train_data;
    const char * fn_checkpoint_in;
    const char * fn_checkpoint_out;
    const char * pattern_fn_it;
    const char * fn_latest;

    bool print_usage;

    int save_every;

    uint32_t seed;

    int n_ctx;
    int n_threads;
    int n_batch;
    int n_gradient_accumulation;
    int n_epochs;
    int n_gpu_layers;

    bool custom_n_ctx;

    bool use_flash;
    bool use_checkpointing;

    std::string sample_start;
    bool include_sample_start;
    bool escape;
    bool overlapping_samples;
    bool fill_with_next_samples;
    bool separate_with_eos;
    bool separate_with_bos;
    bool sample_random_offsets;

    bool force_reshuffle;

    int   warmup;
    int   cos_decay_steps;
    float cos_decay_restart;
    float cos_decay_min;
    bool  enable_restart;

    int   opt_past;
    float opt_delta;
    int   opt_max_no_improvement;

    int   adam_n_iter;
    float adam_alpha;
    float adam_min_alpha;
    float adam_decay;
    int   adam_decay_min_ndim;
    float adam_beta1;
    float adam_beta2;
    float adam_gclip;
    float adam_eps_f;
};

// 定义保存训练文件的回调函数指针类型
typedef void (*save_train_files_callback)(void * data, struct train_state * train);

// 定义训练优化回调数据结构体
struct train_opt_callback_data {
    struct train_params_common * params;
    struct train_state         * train;
    # 定义一个保存训练文件的回调函数
    save_train_files_callback    save_cb;
    # 保存数据的指针
    void                       * save_data;
    # llama 上下文
    struct llama_context       * lctx;
    # 上次保存的迭代次数
    int                          last_save_iter;
    # tokens 数据
    llama_token                * tokens_data;
    # tokens 数据的大小
    size_t                       tokens_size;
    # 样本的起始位置
    size_t                     * samples_begin;
    # 样本的大小
    size_t                     * samples_size;
    # 打乱后的样本偏移
    size_t                     * shuffled_samples_offs;
    # 打乱后的样本起始位置
    size_t                     * shuffled_samples_begin;
    # 打乱后的样本大小
    size_t                     * shuffled_samples_size;
    # 样本数量
    size_t                       samples_count;
    # tokens 输入
    struct ggml_tensor         * tokens_input;
    # 目标概率
    struct ggml_tensor         * target_probs;
    # 第一次迭代
    int                          first_iter;
    # 第一次 epoch
    int                          first_epoch;
    # 上次 epoch 的迭代次数
    int                          iter_at_last_epoch;
    # 上次时间
    int64_t                      last_time;
    # 每次迭代的时间
    double                       millis_per_iter;
// 结构体定义结束

// 初始化训练状态
struct train_state * init_train_state();

// 释放训练状态
void free_train_state(struct train_state  * state);

// 获取默认的训练参数
struct train_params_common get_default_train_params_common();

// 打印通用训练用法
void print_common_train_usage(int /*argc*/, char ** argv, const struct train_params_common * params);

// 解析通用训练参数
bool consume_common_train_arg(int argc, char ** argv, int * idx, struct train_params_common * params, bool * invalid_param);

// 完成处理训练参数
void finish_processing_train_args(struct train_params_common * params);

// 初始化正态分布随机数生成器
struct random_normal_distribution  * init_random_normal_distribution (int seed, float mean, float std, float min, float max);

// 初始化均匀分布随机数生成器
struct random_uniform_distribution * init_random_uniform_distribution(int seed, float min, float max);

// 释放正态分布随机数生成器
void free_random_normal_distribution (struct random_normal_distribution  * rnd);

// 释放均匀分布随机数生成器
void free_random_uniform_distribution(struct random_uniform_distribution * rnd);

// 对张量进行正态分布随机化
struct ggml_tensor * randomize_tensor_normal (struct ggml_tensor * tensor, struct random_normal_distribution * rnd);

// 对张量进行均匀分布随机化
struct ggml_tensor * randomize_tensor_uniform(struct ggml_tensor * tensor, struct random_uniform_distribution * rnd);

// 生成在区间[0,1)内的随机浮点数
float frand();

// 生成正态分布随机浮点数
float frand_normal (struct random_normal_distribution * rnd);

// 生成均匀分布随机浮点数
float frand_uniform(struct random_uniform_distribution * rnd);

// 对整数进行范围限制
int   clamp (const int v, const int min, const int max);

// 对浮点数进行范围限制
float fclamp(const float v, const float min, const float max);

// 断言张量为1维
void assert_shape_1d(struct ggml_tensor * tensor, int64_t ne0);

// 断言张量为2维
void assert_shape_2d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1);

// 断言张量为3维
void assert_shape_3d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1, int64_t ne2);

// 断言张量为4维
void assert_shape_4d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1, int64_t ne2, int64_t ne3);
# 分词文件，将文件内容分词并返回结果
size_t tokenize_file(
        struct llama_context     * lctx,  # 指向 llama_context 结构体的指针
        const char               * filename,  # 文件名
        const std::string        & sample_start,  # 样本起始位置
        bool                       include_sample_start,  # 是否包含样本起始位置
        bool                       overlapping_samples,  # 是否允许重叠样本
        unsigned                   context_length,  # 上下文长度
        std::vector<llama_token> & out_tokens,  # 输出的 tokens
        std::vector<size_t>      & out_samples_begin,  # 输出的样本起始位置
        std::vector<size_t>      & out_samples_size);  # 输出的样本大小

# 获取示例目标批次
int64_t get_example_targets_batch(
        struct llama_context * lctx,  # 指向 llama_context 结构体的指针
        struct ggml_tensor   * tokens_input,  # 输入 tokens
        struct ggml_tensor   * target_probs,  # 目标概率
        int64_t                example_id,  # 示例 ID
        const size_t         * samples_offs,  # 样本偏移
        const size_t         * samples_begin,  # 样本起始位置
        const size_t         * samples_size,  # 样本大小
              size_t           samples_count,  # 样本数量
        const llama_token    * train_data,  # 训练数据
        size_t                 n_train_data,  # 训练数据大小
        bool                   separate_with_eos,  # 是否使用 EOS 分隔
        bool                   separate_with_bos,  # 是否使用 BOS 分隔
        bool                   fill_with_next_samples,  # 是否使用下一个样本填充
        bool                   sample_random_offsets);  # 是否使用随机偏移

# 设置 mt19937 随机数生成器的状态
void          mt19937_set_state(std::mt19937& rng, const mt19937_state& rng_state);
# 获取 mt19937 随机数生成器的状态
mt19937_state mt19937_get_state(const std::mt19937& rng);
# 将种子转换为 mt19937 随机数生成器的状态
mt19937_state mt19937_seed_to_state(unsigned seed);

# 打乱样本
mt19937_state shuffle_samples(
        const mt19937_state & rng_state,  # 随机数生成器的状态
        size_t              * shuffled_offs,  # 打乱后的偏移
        size_t              * shuffled_begins,  # 打乱后的起始位置
        size_t              * shuffled_sizes,  # 打乱后的大小
        const size_t        * begins,  # 起始位置
        const size_t        * sizes,  # 大小
        size_t                count);  # 数量

# 计算哈希值的组合
size_t hash_combine(size_t h1, size_t h2);

# 计算样本的哈希值
size_t compute_samples_hash(
    const char* fn,  # 文件名
    const size_t* samples_begin,  # 样本起始位置
    const size_t* samples_size,  # 样本大小
    size_t sample_count);  # 样本数量

# 替换字符串
std::string replace_str(const char * s, const char * needle, const char * replacement);
// 打印持续时间，单位为毫秒
void print_duration(double milliseconds);

// 计算余弦衰减值
float cosine_decay(
    int64_t step,
    int64_t decay_steps,
    float   minimum);

// 计算带重启的余弦衰减值
float cosine_decay_restart(
    int64_t step,
    int64_t decay_steps,
    float   minimum,
    float   restart_step_mult);

// 学习率调度
float learning_schedule(
    int64_t step,
    int64_t warmup_steps,
    int64_t decay_steps,
    float   learning_rate,
    float   overall_minimum,
    float   cos_decay_minimum,
    float   cos_decay_restart_step_mult,
    bool    enable_restart);

// 根据名称复制张量
void copy_tensor_by_name(struct ggml_tensor * dst, struct ggml_context * ctx, const char * name);

// 加载 GGUF 上下文的优化器上下文
void load_opt_context_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct ggml_opt_context * opt);
// 保存 GGUF 上下文的优化器上下文
void save_opt_context_gguf(struct gguf_context * fctx, struct ggml_opt_context * opt);

// 加载 GGUF 上下文的训练状态
bool load_train_state_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct train_state * train);
// 保存 GGUF 上下文的训练状态
void save_train_state_gguf(struct gguf_context * fctx, struct train_state * train);

// 获取训练文件名
std::string get_train_filename(const char * filename, const char * pattern_it, const char * latest, int64_t iteration);

// 训练优化器回调函数
void train_opt_callback(void * vdata, int accum_step, float * sched, bool * cancel);
```