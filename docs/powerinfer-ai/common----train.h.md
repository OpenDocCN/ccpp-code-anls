# `PowerInfer\common\train.h`

```
// Various helper functions and utilities for training
// 用于训练的各种辅助函数和实用工具

#pragma once
// 防止头文件被多次包含

#include <string>
#include <random>
#include <vector>

#include "ggml.h"
#include "llama.h"
// 包含所需的头文件

#define LLAMA_TRAIN_MAX_NODES 16384
// 定义最大节点数为16384

typedef std::string mt19937_state;
// 定义mt19937_state为std::string类型

struct train_state {
    struct ggml_opt_context * opt;
    // 定义train_state结构体，包含ggml_opt_context指针

    uint64_t train_its;
    uint64_t train_samples;
    // 定义train_its和train_samples为uint64_t类型
    // 用于存储训练数据的标记数
    uint64_t train_tokens;
    // 训练的轮数
    uint64_t train_epochs;

    // 用于存储洗牌样本的哈希值
    size_t        shuffle_samples_hash; // fn, sample_count, *zip(sample_begins, sample_sizes)
    // 当前洗牌随机数生成器的状态
    mt19937_state shuffle_rng_state_current;
    // 下一个洗牌随机数生成器的状态
    mt19937_state shuffle_rng_state_next;
    // 洗牌样本的数量
    size_t        shuffle_sample_count;
    // 下一个洗牌样本的索引
    size_t        shuffle_next_sample;
};

// 训练参数的通用结构
struct train_params_common {
    // 训练数据文件名
    const char * fn_train_data;
    // 输入的检查点文件名
    const char * fn_checkpoint_in;
    // 输出的检查点文件名
    const char * fn_checkpoint_out;
    // 迭代模式文件名
    const char * pattern_fn_it;
    // 最新的文件名
    const char * fn_latest;

    // 是否打印用法
    bool print_usage;

    // 每隔多少次保存检查点
    int save_every;
    // 用于存储随机种子的变量
    uint32_t seed;

    // 上下文的数量
    int n_ctx;
    // 线程的数量
    int n_threads;
    // 批处理的数量
    int n_batch;
    // 梯度累积的数量
    int n_gradient_accumulation;
    // 训练周期的数量
    int n_epochs;
    // GPU 层数的数量
    int n_gpu_layers;

    // 是否使用自定义上下文
    bool custom_n_ctx;

    // 是否使用快闪存
    bool use_flash;
    // 是否使用检查点
    bool use_checkpointing;

    // 样本的起始位置
    std::string sample_start;
    // 是否包含样本的起始位置
    bool include_sample_start;
    // 是否转义
    bool escape;
    // 是否重叠样本
    bool overlapping_samples;
    // 是否用下一个样本填充
    bool fill_with_next_samples;
    # 是否在每个序列之间使用EOS（End of Sequence）进行分隔
    bool separate_with_eos;
    # 是否在每个序列之间使用BOS（Beginning of Sequence）进行分隔
    bool separate_with_bos;
    # 是否在随机偏移处进行采样
    bool sample_random_offsets;

    # 是否强制重新洗牌
    bool force_reshuffle;

    # 预热步数
    int   warmup;
    # 余弦衰减步数
    int   cos_decay_steps;
    # 余弦衰减重启
    float cos_decay_restart;
    # 余弦衰减最小值
    float cos_decay_min;
    # 是否启用重启
    bool  enable_restart;

    # 优化器过去步数
    int   opt_past;
    # 优化器变化量
    float opt_delta;
    # 优化器最大不改进次数
    int   opt_max_no_improvement;

    # Adam优化器迭代次数
    int   adam_n_iter;
    # Adam优化器学习率
    float adam_alpha;
    # Adam优化器最小学习率
    float adam_min_alpha;
    # Adam优化器衰减
    float adam_decay;
// 定义整型变量adam_decay_min_ndim
int adam_decay_min_ndim;
// 定义浮点型变量adam_beta1
float adam_beta1;
// 定义浮点型变量adam_beta2
float adam_beta2;
// 定义浮点型变量adam_gclip
float adam_gclip;
// 定义浮点型变量adam_eps_f
float adam_eps_f;
// 结构体train_opt_callback_data包含训练参数、训练状态、保存回调函数、保存数据、llama上下文、最后保存的迭代次数、llama令牌数据、令牌大小、样本开始位置、样本大小
struct train_opt_callback_data {
    struct train_params_common * params;
    struct train_state         * train;
    save_train_files_callback    save_cb;
    void                       * save_data;
    struct llama_context       * lctx;
    int                          last_save_iter;
    llama_token                * tokens_data;
    size_t                       tokens_size;
    size_t                     * samples_begin;
    size_t                     * samples_size;
};
// 定义指向 size_t 类型的指针变量 shuffled_samples_offs
size_t * shuffled_samples_offs;
// 定义指向 size_t 类型的指针变量 shuffled_samples_begin
size_t * shuffled_samples_begin;
// 定义指向 size_t 类型的指针变量 shuffled_samples_size
size_t * shuffled_samples_size;
// 定义 size_t 类型的变量 samples_count
size_t samples_count;
// 定义指向 struct ggml_tensor 结构体的指针变量 tokens_input
struct ggml_tensor * tokens_input;
// 定义指向 struct ggml_tensor 结构体的指针变量 target_probs
struct ggml_tensor * target_probs;
// 定义 int 类型的变量 first_iter
int first_iter;
// 定义 int 类型的变量 first_epoch
int first_epoch;
// 定义 int 类型的变量 iter_at_last_epoch
int iter_at_last_epoch;
// 定义 int64_t 类型的变量 last_time
int64_t last_time;
// 定义 double 类型的变量 millis_per_iter
double millis_per_iter;

// 定义结构体 train_state，并声明 init_train_state 和 free_train_state 函数
struct train_state * init_train_state();
void free_train_state(struct train_state * state);

// 声明 get_default_train_params_common、print_common_train_usage 和 consume_common_train_arg 函数
struct train_params_common get_default_train_params_common();
void print_common_train_usage(int /*argc*/, char ** argv, const struct train_params_common * params);
bool consume_common_train_arg(int argc, char ** argv, int * idx, struct train_params_common * params, bool * invalid_param);
// 定义函数，用于处理训练参数的结构体
void finish_processing_train_args(struct train_params_common * params);

// 定义随机正态分布结构体
struct random_normal_distribution;
// 定义随机均匀分布结构体
struct random_uniform_distribution;

// 初始化随机正态分布对象
struct random_normal_distribution  * init_random_normal_distribution (int seed, float mean, float std, float min, float max);
// 初始化随机均匀分布对象
struct random_uniform_distribution * init_random_uniform_distribution(int seed, float min, float max);

// 释放随机正态分布对象
void free_random_normal_distribution (struct random_normal_distribution  * rnd);
// 释放随机均匀分布对象
void free_random_uniform_distribution(struct random_uniform_distribution * rnd);

// 对张量进行正态分布随机化
struct ggml_tensor * randomize_tensor_normal (struct ggml_tensor * tensor, struct random_normal_distribution * rnd);
// 对张量进行均匀分布随机化
struct ggml_tensor * randomize_tensor_uniform(struct ggml_tensor * tensor, struct random_uniform_distribution * rnd);

// 生成在区间[0,1)内的随机浮点数
float frand();
// 从随机正态分布对象中生成随机浮点数
float frand_normal (struct random_normal_distribution * rnd);
// 从随机均匀分布对象中生成随机浮点数
float frand_uniform(struct random_uniform_distribution * rnd);

// 将值限制在最小值和最大值之间
int   clamp (const int v, const int min, const int max);
// 定义一个函数，用于将输入值限制在指定范围内
float fclamp(const float v, const float min, const float max);

// 定义一个函数，用于断言输入的一维张量的形状是否符合预期
void assert_shape_1d(struct ggml_tensor * tensor, int64_t ne0);

// 定义一个函数，用于断言输入的二维张量的形状是否符合预期
void assert_shape_2d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1);

// 定义一个函数，用于断言输入的三维张量的形状是否符合预期
void assert_shape_3d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1, int64_t ne2);

// 定义一个函数，用于断言输入的四维张量的形状是否符合预期
void assert_shape_4d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1, int64_t ne2, int64_t ne3);

// 定义一个函数，用于从文件中提取标记，并返回标记的数量和位置
size_t tokenize_file(
        struct llama_context     * lctx,
        const char               * filename,
        const std::string        & sample_start,
        bool                       include_sample_start,
        bool                       overlapping_samples,
        unsigned                   context_length,
        std::vector<llama_token> & out_tokens,
        std::vector<size_t>      & out_samples_begin,
        std::vector<size_t>      & out_samples_size);

// 定义一个函数，用于获取示例目标的批次
int64_t get_example_targets_batch(
        struct llama_context * lctx,
// 定义了一系列参数和函数，用于处理神经网络训练中的样本数据和随机数生成器的状态

// 设置神经网络训练中的随机数生成器的状态
void mt19937_set_state(std::mt19937& rng, const mt19937_state& rng_state);

// 获取神经网络训练中的随机数生成器的状态
mt19937_state mt19937_get_state(const std::mt19937& rng);

// 将种子转换为随机数生成器的状态
mt19937_state mt19937_seed_to_state(unsigned seed);

// 对样本数据进行洗牌，返回洗牌后的状态
mt19937_state shuffle_samples(
// 声明一个函数，接受随机数生成器状态、洗牌偏移、洗牌开始位置、洗牌大小、开始位置、大小和计数作为参数
void shuffle_samples(
        const mt19937_state & rng_state,
        size_t              * shuffled_offs,
        size_t              * shuffled_begins,
        size_t              * shuffled_sizes,
        const size_t        * begins,
        const size_t        * sizes,
        size_t                count);

// 声明一个函数，用于组合两个哈希值
size_t hash_combine(size_t h1, size_t h2);

// 声明一个函数，用于计算样本的哈希值，接受文件名、样本开始位置、样本大小和样本数量作为参数
size_t compute_samples_hash(
    const char* fn,
    const size_t* samples_begin,
    const size_t* samples_size,
    size_t sample_count);

// 声明一个函数，用于替换字符串中的子串
std::string replace_str(const char * s, const char * needle, const char * replacement);

// 声明一个函数，用于打印时间间隔，接受毫秒数作为参数
void print_duration(double milliseconds);
# 计算余弦衰减的学习率
def cosine_decay(
    step,  # 当前步数
    decay_steps,  # 衰减步数
    minimum  # 最小学习率
    );

# 计算带重启的余弦衰减的学习率
def cosine_decay_restart(
    step,  # 当前步数
    decay_steps,  # 衰减步数
    minimum,  # 最小学习率
    restart_step_mult  # 重启步数倍增因子
    );

# 计算学习率的调度
def learning_schedule(
    step,  # 当前步数
    warmup_steps,  # 热身步数
    decay_steps,  # 衰减步数
    learning_rate,  # 初始学习率
    overall_minimum,  # 总体最小学习率
    cos_decay_minimum,  # 余弦衰减最小学习率
    cos_decay_restart_step_mult  # 余弦衰减重启步数倍增因子
    );
// 声明一个布尔类型的变量 enable_restart
bool enable_restart;

// 根据名称复制张量
void copy_tensor_by_name(struct ggml_tensor * dst, struct ggml_context * ctx, const char * name);

// 从 ggml_context 中加载优化器上下文到 gguf_context 中
void load_opt_context_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct ggml_opt_context * opt);
// 将优化器上下文保存到 gguf_context 中
void save_opt_context_gguf(struct gguf_context * fctx, struct ggml_opt_context * opt);

// 从 gguf_context 中加载训练状态到 train_state 中
bool load_train_state_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct train_state * train);
// 将训练状态保存到 gguf_context 中
void save_train_state_gguf(struct gguf_context * fctx, struct train_state * train);

// 根据文件名、模式和最新迭代次数获取训练文件名
std::string get_train_filename(const char * filename, const char * pattern_it, const char * latest, int64_t iteration);

// 训练优化回调函数
void train_opt_callback(void * vdata, int accum_step, float * sched, bool * cancel);
```