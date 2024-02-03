# `PowerInfer\common\train.cpp`

```cpp
// 包含头文件 train.h 和 common.h
#include "train.h"
#include "common.h"

// 包含随机数生成、字符串流、函数绑定等标准库
#include <random>
#include <sstream>
#include <functional>

// 定义随机正态分布结构体
struct random_normal_distribution {
    std::mt19937 gen;  // Mersenne Twister 19937 伪随机数生成器
    std::normal_distribution<float> rd;  // 正态分布
    float min;  // 最小值
    float max;  // 最大值
};

// 定义随机均匀分布结构体
struct random_uniform_distribution {
    std::mt19937 gen;  // Mersenne Twister 19937 伪随机数生成器
    std::uniform_real_distribution<float> rd;  // 均匀分布
};

// 初始化训练状态结构体
struct train_state  * init_train_state() {
    struct train_state * state = new struct train_state;  // 分配内存
    state->train_its     = 0;  // 初始化迭代次数
    state->train_samples = 0;  // 初始化样本数
    state->train_tokens  = 0;  // 初始化标记数
    state->train_epochs  = 0;  // 初始化训练轮数
    state->shuffle_samples_hash  = 0;  // 初始化样本哈希值
    state->shuffle_sample_count  = 0;  // 初始化样本计数
    state->shuffle_next_sample   = 0;  // 初始化下一个样本
    state->shuffle_rng_state_current = "";  // 初始化当前随机数生成器状态
    state->shuffle_rng_state_next    = "";  // 初始化下一个随机数生成器状态

    state->opt = new struct ggml_opt_context;  // 分配内存
    state->opt->ctx = NULL;  // 初始化上下文
    state->opt->params = ggml_opt_default_params(GGML_OPT_ADAM);  // 初始化参数
    state->opt->params.graph_size = LLAMA_TRAIN_MAX_NODES;  // 设置图大小
    state->opt->loss_after = 0.0f;  // 设置损失值

    return state;  // 返回训练状态结构体
}

// 释放训练状态结构体
void free_train_state(struct train_state  * state) {
    delete state->opt;  // 释放参数内存
    delete state;  // 释放训练状态内存
}

// 初始化随机正态分布结构体
struct random_normal_distribution * init_random_normal_distribution(
    int seed, float mean, float std, float min, float max
) {
    struct random_normal_distribution * rnd = (struct random_normal_distribution *) malloc(sizeof(struct random_normal_distribution));  // 分配内存
    rnd->gen = std::mt19937(seed);  // 初始化伪随机数生成器
    rnd->rd = std::normal_distribution<float>{mean, std};  // 初始化正态分布
    rnd->min = min;  // 设置最小值
    rnd->max = max;  // 设置最大值
    return rnd;  // 返回随机正态分布结构体
}

// 初始化随机均匀分布结构体
struct random_uniform_distribution * init_random_uniform_distribution(int seed, float min, float max) {
    struct random_uniform_distribution * rnd = (struct random_uniform_distribution *) malloc(sizeof(struct random_uniform_distribution));  // 分配内存
    rnd->gen = std::mt19937(seed);  // 初始化伪随机数生成器
    rnd->rd = std::uniform_real_distribution<float>{min, max};  // 初始化均匀分布
    return rnd;  // 返回随机均匀分布结构体
}

// 释放随机正态分布结构体
void free_random_normal_distribution (struct random_normal_distribution  * rnd) {
    free(rnd);  // 释放内存
}
# 释放随机均匀分布结构体所占用的内存
void free_random_uniform_distribution(struct random_uniform_distribution * rnd) {
    free(rnd);
}

# 对张量进行正态分布随机化
struct ggml_tensor * randomize_tensor_normal(struct ggml_tensor * tensor, struct random_normal_distribution * rnd) {
    # 设置比例为1.0，用于Xavier初始化
    float scale = 1.0f; // xavier
    # 根据张量的维度进行不同的处理
    switch (tensor->n_dims) {
        # 当维度为1时
        case 1:
            # 计算缩放比例
            scale /= sqrtf((float) tensor->ne[0]);
            # 遍历第一维度，生成服从正态分布的随机数并赋值给对应位置的数据
            for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0]);
                *dst = scale * frand_normal(rnd);
            }
            break;
        # 当维度为2时
        case 2:
            # 计算缩放比例
            scale /= sqrtf((float) tensor->ne[0]+tensor->ne[1]);
            # 遍历第二维度和第一维度，生成服从正态分布的随机数并赋值给对应位置的数据
            for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                    float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1]);
                    *dst = scale * frand_normal(rnd);
                }
            }
            break;
        # 当维度为3时
        case 3:
            # 计算缩放比例
            scale /= sqrtf((float) tensor->ne[0]+tensor->ne[1]);
            # 遍历第三维度、第二维度和第一维度，生成服从正态分布的随机数并赋值给对应位置的数据
            for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                    for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                        float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2]);
                        *dst = scale * frand_normal(rnd);
                    }
                }
            }
            break;
        # 当维度为4时
        case 4:
            # 计算缩放比例
            scale /= sqrtf((float) tensor->ne[0]+tensor->ne[1]);
            # 遍历第四维度、第三维度、第二维度和第一维度，生成服从正态分布的随机数并赋值给对应位置的数据
            for (int i3 = 0; i3 < tensor->ne[3]; i3++) {
                for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                    for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                        for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                            float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3]);
                            *dst = scale * frand_normal(rnd);
                        }
                    }
                }
            }
            break;
        # 默认情况下，维度不受支持，抛出错误
        default:
            die("Unsupported tensor->n_dims");
    };
    # 返回处理后的张量
    return tensor;
# 以均匀分布随机初始化张量数据
struct ggml_tensor * randomize_tensor_uniform(struct ggml_tensor * tensor, struct random_uniform_distribution * rnd) {
    # 根据张量的维度进行不同的初始化操作
    switch (tensor->n_dims) {
        # 当张量维度为1时
        case 1:
            # 遍历张量的第一维度，对每个元素赋予均匀分布的随机值
            for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0]);
                *dst = frand_uniform(rnd);
            }
            break;
        # 当张量维度为2时
        case 2:
            # 遍历张量的第二维度和第一维度，对每个元素赋予均匀分布的随机值
            for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                    float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1]);
                    *dst = frand_uniform(rnd);
                }
            }
            break;
        # 当张量维度为3时
        case 3:
            # 遍历张量的第三维度、第二维度和第一维度，对每个元素赋予均匀分布的随机值
            for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                    for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                        float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2]);
                        *dst = frand_uniform(rnd);
                    }
                }
            }
            break;
        # 当张量维度为4时
        case 4:
            # 遍历张量的第四维度、第三维度、第二维度和第一维度，对每个元素赋予均匀分布的随机值
            for (int i3 = 0; i3 < tensor->ne[3]; i3++) {
                for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                    for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                        for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                            float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3]);
                            *dst = frand_uniform(rnd);
                        }
                    }
                }
            }
            break;
        # 当张量维度不在1到4之间时
        default:
            # 报错，不支持的张量维度
            die("Unsupported tensor->n_dims");
    };
    # 返回初始化后的张量
    return tensor;
}

# 生成一个0到1之间的随机浮点数
float frand() {
    return (float)rand()/((float)(RAND_MAX) + 1.0f);
}

# 以正态分布随机初始化张量数据
float frand_normal(struct random_normal_distribution * rnd) {
    # 返回一个介于最小值和最大值之间的随机数
    return fclamp(rnd->rd(rnd->gen), rnd->min, rnd->max);
# 返回一个在指定范围内的随机浮点数
float frand_uniform(struct random_uniform_distribution * rnd) {
    return rnd->rd(rnd->gen);
}

# 将输入值限制在指定范围内，并返回结果
int clamp(const int v, const int min, const int max) {
    return ((v < min) ? (min) : (v > max) ? (max) : v);
}

# 将输入值限制在指定范围内，并返回结果
float fclamp(const float v, const float min, const float max) {
    return ((v < min) ? (min) : (v > max) ? (max) : v);
}

# 断言输入的张量是一维的，并且元素个数符合预期
void assert_shape_1d(struct ggml_tensor * tensor, int64_t ne0) {
    GGML_ASSERT(tensor->n_dims == 1);
    GGML_ASSERT(tensor->ne[0] == ne0);
}

# 断言输入的张量是二维的，并且各维度的元素个数符合预期
void assert_shape_2d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1) {
    GGML_ASSERT(tensor->n_dims == 2);
    GGML_ASSERT(tensor->ne[0] == ne0);
    GGML_ASSERT(tensor->ne[1] == ne1);
}

# 断言输入的张量是三维的，并且各维度的元素个数符合预期
void assert_shape_3d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1, int64_t ne2) {
    GGML_ASSERT(tensor->n_dims == 3);
    GGML_ASSERT(tensor->ne[0] == ne0);
    GGML_ASSERT(tensor->ne[1] == ne1);
    GGML_ASSERT(tensor->ne[2] == ne2);
}

# 断言输入的张量是四维的，并且各维度的元素个数符合预期
void assert_shape_4d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1, int64_t ne2, int64_t ne3) {
    GGML_ASSERT(tensor->n_dims == 4);
    GGML_ASSERT(tensor->ne[0] == ne0);
    GGML_ASSERT(tensor->ne[1] == ne1);
    GGML_ASSERT(tensor->ne[2] == ne2);
    GGML_ASSERT(tensor->ne[3] == ne3);
}

# 获取示例目标的批次数据
int64_t get_example_targets_batch(
    struct llama_context * lctx,
    struct ggml_tensor   * tokens_input,
    struct ggml_tensor   * target_probs,
    int64_t                example_id,
    const size_t         * samples_offs,
    const size_t         * samples_begin,
    const size_t         * samples_size,
          size_t           samples_count,
    const llama_token    * train_data,
    size_t                 n_train_data,
    bool                   separate_with_eos,
    bool                   separate_with_bos,
    bool                   fill_with_next_samples,
    bool                   sample_random_offsets
) {
    # 断言样本数量大于0
    GGML_ASSERT(samples_count > 0);
    # 断言输入的 tokens_input 张量是二维的
    GGML_ASSERT(tokens_input->n_dims  == 2);
    # 断言输入的 target_probs 张量是三维的
    GGML_ASSERT(target_probs->n_dims  == 3);
    # 获取目标概率的词汇量
    int64_t n_vocab  = target_probs->ne[0];
    # 获取输入 tokens 的数量
    int64_t n_tokens = tokens_input->ne[0];
    # 获取批次数量
    int64_t n_batch  = tokens_input->ne[1];
    # 断言目标概率的词汇量与实际词汇量相等
    GGML_ASSERT(n_vocab  == target_probs->ne[0]);
    # 断言输入 tokens 的数量与目标概率的数量相等
    GGML_ASSERT(n_tokens == target_probs->ne[1]);
    # 断言批次数量与目标概率的数量相等
    GGML_ASSERT(n_batch  == target_probs->ne[2]);

    # 初始化已使用样本数量
    int64_t used_samples = 0;

    # 将目标概率设置为 0.0
    ggml_set_f32(target_probs, 0.0f);
    # 获取开始符号
    llama_token bos = llama_token_bos(llama_get_model(lctx));
    # 获取结束符号
    llama_token eos = llama_token_eos(llama_get_model(lctx));
    # 打印函数信息和批次信息
    // printf("%s: example_id=%d n_batch=%d n_train_samples=%zu\n", __func__, example_id, n_batch, n_train_samples);
    }

    # 返回已使用样本数量
    return used_samples;
// 设置 mt19937 随机数生成器的状态
void mt19937_set_state(std::mt19937& rng, const std::string& rng_state) {
    // 创建字符串流对象，使用经典的本地化设置
    std::stringstream s_rng_state;
    s_rng_state.imbue(std::locale::classic());
    // 设置异常状态，如果发生异常则抛出 failbit
    s_rng_state.exceptions(std::stringstream::failbit);
    // 将输入的 rng_state 字符串写入字符串流
    s_rng_state.str(rng_state);
    // 从字符串流中读取数据并设置为随机数生成器的状态
    s_rng_state >> rng;
}

// 获取 mt19937 随机数生成器的状态
std::string mt19937_get_state(const std::mt19937& rng) {
    // 创建字符串流对象，使用经典的本地化设置
    std::stringstream s_rng_state;
    s_rng_state.imbue(std::locale::classic());
    // 将随机数生成器的状态写入字符串流
    s_rng_state << rng;
    // 返回字符串流中的状态数据
    return s_rng_state.str();
}

// 将种子转换为 mt19937 随机数生成器的状态
std::string mt19937_seed_to_state(unsigned seed) {
    // 使用给定的种子创建 mt19937 随机数生成器
    std::mt19937 rng(seed);
    // 返回随机数生成器的状态
    return mt19937_get_state(rng);
}

// 对样本进行洗牌
std::string shuffle_samples(
        const std::string & rng_state,
        size_t            * shuffled_offs,
        size_t            * shuffled_begins,
        size_t            * shuffled_sizes,
        const size_t      * begins,
        const size_t      * sizes,
        size_t              count) {
    // 如果样本数量为 0，则直接返回随机数生成器的状态
    if (count == 0) return rng_state;

    // 创建 mt19937 随机数生成器，并设置其状态为输入的 rng_state
    std::mt19937 rng;
    mt19937_set_state(rng, rng_state);

    // 对索引进行排序，根据每个索引的随机值
    std::vector<size_t> idcs;
    {
        std::vector<unsigned> rnd;
        idcs.resize(count);
        rnd.resize(count);
        for (unsigned i=0; i<count; ++i) {
            idcs[i] = i;
            rnd[i]  = rng();
        }

        std::sort(idcs.begin(), idcs.end(), [&rnd](size_t a, size_t b){
            // 稳定排序以保证可重现性
            return (rnd[a] == rnd[b]) ? (a < b) : (rnd[a] < rnd[b]);
        });
    }

    // 创建随机偏移量
    for (unsigned i=0; i<count; ++i) {
        shuffled_offs[i] = (size_t) ((sizes[idcs[i]] - 1) * ((double) rng() / (double) (rng.max()-1)));
    }

    // 根据排序后的索引重新排列 begins 和 sizes
    for (unsigned i=0; i<count; ++i) {
        shuffled_begins[i] = begins[idcs[i]];
    }

    for (unsigned i=0; i<count; ++i) {
        shuffled_sizes[i] = sizes[idcs[i]];
    }

    // 返回洗牌后的随机数生成器的状态
    return mt19937_get_state(rng);
}

// 合并哈希值
size_t hash_combine(size_t h1, size_t h2) {
    # 返回 h1 和 h2 左移一位后的异或结果
    return h1 ^ (h2 << 1);
// 计算样本哈希值，根据文件名、样本起始位置、样本大小和样本数量
size_t compute_samples_hash(const char* fn, const size_t* samples_begin, const size_t* samples_size, size_t sample_count) {
    // 创建字符串哈希对象
    std::hash<std::string> h_string;
    // 创建无符号长长整型哈希对象
    std::hash<unsigned long long> h_ull;
    // 使用文件名计算哈希值
    size_t h = h_string(std::string(fn));
    // 结合文件名哈希值和样本数量的哈希值
    h = hash_combine(h, h_ull((unsigned long long) sample_count));
    // 遍历样本，计算哈希值
    for (size_t i=0; i< sample_count; ++i) {
        h = hash_combine(h, h_ull((unsigned long long) samples_begin[i]));
        h = hash_combine(h, h_ull((unsigned long long) samples_size[i]));
    }
    // 返回最终哈希值
    return h;
}

// 替换字符串中的子串
std::string replace_str(const char * s, const char * needle, const char * replacement) {
    // 将输入的 C 字符串转换为 C++ 字符串
    std::string str = s;
    // 查找子串的位置
    size_t pos = str.find(needle);
    // 如果找到子串，则替换
    if (pos != std::string::npos) {
        str.replace(pos, strlen(needle), replacement);
    }
    // 返回替换后的字符串
    return str;
}

// 打印时间间隔
void print_duration(double fmillis) {
    // 如果时间小于1秒，则以毫秒为单位打印
    if (fmillis < 1000.0f) {
        printf("%.1fms", (float) fmillis);
        return;
    }
    // 计算时间间隔的天、小时、分钟和秒
    // ...
    // 打印时间间隔
    // ...
}

// 计算余弦衰减
float cosine_decay(int64_t step, int64_t decay_steps, float minimum) {
    // 如果步数大于衰减步数，则将步数设置为衰减步数
    if (step > decay_steps) {
        step = decay_steps;
    }
    // 计算余弦衰减值
    const float cosine_decay = 0.50f*(1.0f + cosf(3.14159265359f*step/decay_steps));
    // 计算最终衰减值
    const float decay = (1 - minimum)*cosine_decay + minimum;
    // 返回衰减值
    return decay;
}
// 使用余弦衰减重新启动的方式计算余弦衰减值
float cosine_decay_restart(int64_t step, int64_t decay_steps, float minimum, float restart_step_mult) {
    // 当步数大于衰减步数时，执行循环
    while (step > decay_steps) {
        // 步数减去衰减步数
        step -= decay_steps;
        // 重新计算衰减步数
        decay_steps = (int64_t) (restart_step_mult * decay_steps);
    }
    // 返回余弦衰减值
    return cosine_decay(step, decay_steps, minimum);
}

// 学习率调度函数
float learning_schedule(
    int64_t step,
    int64_t warmup_steps,
    int64_t cos_decay_steps,
    float   learning_rate,
    float   overall_minimum,
    float   cos_decay_minimum,
    float   cos_decay_restart_step_mult,
    bool    enable_restart) {

    // 计算结果
    float result =
        // 如果步数小于预热步数
        (step < warmup_steps)
            // 则返回步数除以预热步数的比例
            ? (float) step / (float) warmup_steps
            // 否则，如果启用重新启动
            : enable_restart
                // 则调用余弦衰减重新启动函数
                ? cosine_decay_restart(
                    step - warmup_steps,
                    cos_decay_steps,
                    cos_decay_minimum,
                    cos_decay_restart_step_mult)
                // 否则调用普通余弦衰减函数
                : cosine_decay(
                    step,
                    cos_decay_steps,
                    cos_decay_minimum);

    // 计算最小值
    float min = overall_minimum / learning_rate;
    // 计算最终结果
    result = min + result * (1.0f - min);
    // 返回结果
    return result;
}

// 检查两个张量是否具有相同的布局
static bool are_same_layout(struct ggml_tensor * a, struct ggml_tensor * b) {
    // 断言张量 a 和 b 非空
    GGML_ASSERT(a != NULL);
    GGML_ASSERT(b != NULL);
    // 断言张量 a 和 b 的类型相同
    GGML_ASSERT(a->type == b->type);
    // 断言张量 a 和 b 的形状相同
    GGML_ASSERT(ggml_are_same_shape(a, b));
    // 断言张量 a 和 b 是连续的
    GGML_ASSERT(ggml_is_contiguous(a) && ggml_is_contiguous(b));

    // 返回 true
    return true;
}

// 根据名称复制张量
void copy_tensor_by_name(struct ggml_tensor * dst, struct ggml_context * ctx, const char * name) {
    // 如果目标张量为空，则返回
    if (dst == NULL) {
        return;
    }
    // 根据名称获取张量
    struct ggml_tensor * t  = ggml_get_tensor(ctx, name);
    // 断言目标张量和获取的张量具有相同的布局
    GGML_ASSERT(are_same_layout(dst, t));
    // 复制获取的张量的数据到目标张量
    memcpy(dst->data, t->data, ggml_nbytes(t));

    // 如果目标张量的名称长度为0
    if (strlen(ggml_get_name(dst)) == 0) {
        // 则设置目标张量的名称为给定名称
        ggml_set_name(dst, name);
    }
}

// gguf 常量
static const char * LLM_KV_OPTIMIZER_TYPE = "optimizer.type";
static const char * LLM_KV_OPTIMIZER_TYPE_ADAM  = "adam";
# 定义字符串常量，表示优化器类型为 LBFGS
static const char * LLM_KV_OPTIMIZER_TYPE_LBFGS = "lbfgs";
# 定义字符串常量，表示优化器文件版本
static const char * LLM_KV_OPTIMIZER_FILE_VERSION = "optimizer.file_version";
# 定义字符串常量，表示优化器收敛过去计数
static const char * LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT = "optimizer.convergence_past_count";
# 定义字符串常量，表示优化器参数计数
static const char * LLM_KV_OPTIMIZER_PARAMETER_COUNT = "optimizer.parameter_count";
# 定义字符串常量，表示优化器迭代次数
static const char * LLM_KV_OPTIMIZER_ITERATION_COUNT = "optimizer.iteration_count";
# 定义字符串常量，表示优化器刚初始化
static const char * LLM_KV_OPTIMIZER_JUST_INITIALIZED = "optimizer.just_initialized";
# 定义字符串常量，表示优化器 Adam 最佳损失
static const char * LLM_KV_OPTIMIZER_ADAM_BEST_LOSS = "optimizer.adam.best_loss";
# 定义字符串常量，表示优化器 Adam 上一次损失
static const char * LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS = "optimizer.adam.previous_loss";
# 定义字符串常量，表示优化器 Adam 无改善计数
static const char * LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT = "optimizer.adam.no_improvement_count";
# 定义字符串常量，表示优化器 LBFGS 近似 Hessian 计数
static const char * LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT = "optimizer.lbfgs.approx_hessian_count";
# 定义字符串常量，表示优化器 LBFGS 最佳损失
static const char * LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS = "optimizer.lbfgs.best_loss";
# 定义字符串常量，表示优化器 LBFGS 线搜索步长
static const char * LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP = "optimizer.lbfgs.line_search_step";
# 定义字符串常量，表示优化器 LBFGS 线搜索 j
static const char * LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J = "optimizer.lbfgs.line_search_j";
# 定义字符串常量，表示优化器 LBFGS 线搜索 k
static const char * LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K = "optimizer.lbfgs.line_search_k";
# 定义字符串常量，表示优化器 LBFGS 线搜索结束
static const char * LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END = "optimizer.lbfgs.line_search_end";
# 定义字符串常量，表示优化器 LBFGS 无改善计数
static const char * LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT = "optimizer.lbfgs.no_improvement_count";

# 定义字符串常量，表示优化器 Adam 第一时刻
static const char * LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS = "optimizer.adam.first_moments";
# 定义字符串常量，表示优化器 Adam 第二时刻
static const char * LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS = "optimizer.adam.second_moments";
# 定义字符串常量，表示优化器 Adam 过去损失值
static const char * LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES = "optimizer.adam.past_loss_values";

# 定义字符串常量，表示优化器 LBFGS 当前参数
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS = "optimizer.lbfgs.current_parameters";
# 定义常量，表示优化器 LBFGS 的先前参数
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS = "optimizer.lbfgs.previous_parameters";
# 定义常量，表示优化器 LBFGS 的当前梯度
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS   = "optimizer.lbfgs.current_gradients";
# 定义常量，表示优化器 LBFGS 的先前梯度
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS  = "optimizer.lbfgs.previous_gradients";
# 定义常量，表示优化器 LBFGS 的搜索方向
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION    = "optimizer.lbfgs.search_direction";
# 定义常量，表示优化器 LBFGS 的过去损失值
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES    = "optimizer.lbfgs.past_loss_values";
# 定义常量，表示优化器 LBFGS 的内存参数 alpha
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA        = "optimizer.lbfgs.memory_alpha";
# 定义常量，表示优化器 LBFGS 的内存参数 ys
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS           = "optimizer.lbfgs.memory_ys";
# 定义常量，表示优化器 LBFGS 的内存参数 s
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S            = "optimizer.lbfgs.memory_s";
# 定义常量，表示优化器 LBFGS 的内存参数 y
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y            = "optimizer.lbfgs.memory_y";

# 定义常量，表示训练文件的版本
static const char * LLM_KV_TRAINING_FILE_VERSION         = "training.file_version";
# 定义常量，表示训练迭代次数
static const char * LLM_KV_TRAINING_ITERATION_COUNT      = "training.iteration_count";
# 定义常量，表示训练样本数量
static const char * LLM_KV_TRAINING_SAMPLE_COUNT         = "training.sample_count";
# 定义常量，表示训练标记数量
static const char * LLM_KV_TRAINING_TOKEN_COUNT          = "training.token_count";
# 定义常量，表示训练周期数量
static const char * LLM_KV_TRAINING_EPOCH_COUNT          = "training.epoch_count";
# 定义常量，表示训练样本哈希值
static const char * LLM_KV_TRAINING_SHUFFLE_SAMPLES_HASH = "training.shuffle.samples_hash";
# 定义常量，表示训练随机数生成器状态
static const char * LLM_KV_TRAINING_SHUFFLE_RNG_STATE    = "training.shuffle.rng_state";
# 定义常量，表示训练随机样本数量
static const char * LLM_KV_TRAINING_SHUFFLE_SAMPLE_COUNT = "training.shuffle.sample_count";
# 定义常量，表示训练下一个随机样本
static const char * LLM_KV_TRAINING_SHUFFLE_NEXT_SAMPLE  = "training.shuffle.next_sample";

# 定义宏，用于获取键值
#define GGUF_GET_KEY(ctx, dst, func, type, req, key) \
{ \
    # 将键值转换为字符串
    const std::string skey(key); \
    # 在上下文中查找键值的索引
    const int kid = gguf_find_key(ctx, skey.c_str()); \
    # 如果键的索引大于等于0
    if (kid >= 0) { \
        # 获取键值对的类型
        enum gguf_type ktype = gguf_get_kv_type(ctx, kid); \
        # 如果键值对的类型不等于指定的类型
        if (ktype != (type)) { \
            # 抛出错误，指明键的类型错误
            die_fmt("key %s has wrong type: %s", skey.c_str(), gguf_type_name(ktype)); \
        } \
        # 将目标变量赋值为指定函数的返回值
        (dst) = func(ctx, kid); \
    } 
    # 如果键的索引小于0且req为真
    else if (req) { \
        # 抛出错误，指明在模型中找不到指定的键
        die_fmt("key not found in model: %s", skey.c_str()); \
    } \
// 加载 gguf 上下文，初始化 gguf 上下文，确保不分配内存，否则无法读取张量数据
void load_opt_context_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct ggml_opt_context * opt) {
    // 从 gguf 上下文中获取文件版本号
    uint32_t file_version;
    GGUF_GET_KEY(fctx, file_version, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_OPTIMIZER_FILE_VERSION);
    // 断言文件版本号为0
    GGML_ASSERT(file_version == 0);

    // 从 gguf 上下文中获取收敛过去次数
    GGUF_GET_KEY(fctx, opt->params.past, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT);
    // 从 gguf 上下文中获取迭代次数
    GGUF_GET_KEY(fctx, opt->iter, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_OPTIMIZER_ITERATION_COUNT);
    // 从 gguf 上下文中获取是否刚初始化
    GGUF_GET_KEY(fctx, opt->just_initialized, gguf_get_val_bool, GGUF_TYPE_BOOL, true, LLM_KV_OPTIMIZER_JUST_INITIALIZED);

    // 从 gguf 上下文中获取参数数量
    uint64_t nx;
    GGUF_GET_KEY(fctx, nx, gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_OPTIMIZER_PARAMETER_COUNT);
    // 将参数数量转换为 size_t 类型并赋值给 opt->nx
    opt->nx = (size_t) nx;

    // 在了解优化器类型和特定参数之前，不要调用 ggml_opt_init

    // 获取优化器类型
    std::string opt_type;
    GGUF_GET_KEY(fctx, opt_type, gguf_get_val_str, GGUF_TYPE_STRING, true, LLM_KV_OPTIMIZER_TYPE);
}
    # 如果优化器类型是ADAM
    if (opt_type == LLM_KV_OPTIMIZER_TYPE_ADAM) {
        # 设置优化器参数类型为ADAM
        opt->params.type = GGML_OPT_ADAM;

        # 从上下文中获取最佳损失值，并存储到opt->adam.fx_best中
        GGUF_GET_KEY(fctx, opt->adam.fx_best,          gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, LLM_KV_OPTIMIZER_ADAM_BEST_LOSS);
        # 从上下文中获取上一次损失值，并存储到opt->adam.fx_prev中
        GGUF_GET_KEY(fctx, opt->adam.fx_prev,          gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS);
        # 从上下文中获取无改善计数，并存储到opt->adam.n_no_improvement中
        GGUF_GET_KEY(fctx, opt->adam.n_no_improvement, gguf_get_val_u32, GGUF_TYPE_UINT32,  true, LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT);

        # 初始化优化器
        ggml_opt_init(opt->ctx, opt, opt->params, opt->nx);

        # 通过名称从f_ggml_ctx中复制张量到opt->adam.m中
        copy_tensor_by_name(opt->adam.m,  f_ggml_ctx, LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS);
        # 通过名称从f_ggml_ctx中复制张量到opt->adam.v中
        copy_tensor_by_name(opt->adam.v,  f_ggml_ctx, LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS);
        # 通过名称从f_ggml_ctx中复制张量到opt->adam.pf中
        copy_tensor_by_name(opt->adam.pf, f_ggml_ctx, LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES);
    } else {
        # 如果优化器类型未知，则输出错误信息
        die("unknown optimizer type\n");
    }
void save_opt_context_gguf(struct gguf_context * fctx, struct ggml_opt_context * opt) {
    // 设置优化器文件版本号为0
    gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_FILE_VERSION, 0);
    // 设置优化器收敛过去计数为opt参数中的过去计数
    gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT, opt->params.past);
    // 设置优化器参数计数为opt参数中的nx值
    gguf_set_val_u64(fctx, LLM_KV_OPTIMIZER_PARAMETER_COUNT, (uint64_t) opt->nx);
    // 设置优化器迭代计数为opt参数中的iter值
    gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_ITERATION_COUNT, opt->iter);
    // 设置优化器刚初始化为opt参数中的just_initialized值
    gguf_set_val_bool(fctx, LLM_KV_OPTIMIZER_JUST_INITIALIZED, opt->just_initialized);
    // 结束函数
    }
}

bool load_train_state_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct train_state * train) {
    // 如果在fctx中找不到LLM_KV_TRAINING_FILE_VERSION键，则返回false
    if (gguf_find_key(fctx, LLM_KV_TRAINING_FILE_VERSION) < 0) {
        return false;
    }

    uint32_t file_version;
    // 从fctx中获取LLM_KV_TRAINING_FILE_VERSION键对应的值，存储在file_version中
    GGUF_GET_KEY(fctx, file_version,         gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_FILE_VERSION);
    // 断言file_version小于等于1
    GGML_ASSERT(file_version <= 1);

    // 如果file_version为0
    if (file_version == 0) {
        // 从fctx中获取LLM_KV_TRAINING_ITERATION_COUNT键对应的值，存储在train->train_its中
        GGUF_GET_KEY(fctx, train->train_its,     gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_ITERATION_COUNT);
        // 从fctx中获取LLM_KV_TRAINING_SAMPLE_COUNT键对应的值，存储在train->train_samples中
        GGUF_GET_KEY(fctx, train->train_samples, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_SAMPLE_COUNT);
        // 从fctx中获取LLM_KV_TRAINING_TOKEN_COUNT键对应的值，存储在train->train_tokens中
        GGUF_GET_KEY(fctx, train->train_tokens,  gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_TOKEN_COUNT);
    # 如果文件版本为1，则执行以下操作
    } else if (file_version == 1) {

        # 从上下文中获取训练迭代次数，并存储到train->train_its中
        GGUF_GET_KEY(fctx, train->train_its,     gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_TRAINING_ITERATION_COUNT);
        # 从上下文中获取训练样本数量，并存储到train->train_samples中
        GGUF_GET_KEY(fctx, train->train_samples, gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_TRAINING_SAMPLE_COUNT);
        # 从上下文中获取训练标记数量，并存储到train->train_tokens中
        GGUF_GET_KEY(fctx, train->train_tokens,  gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_TRAINING_TOKEN_COUNT);
        # 从上下文中获取训练周期数量，并存储到train->train_epochs中
        GGUF_GET_KEY(fctx, train->train_epochs,  gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_TRAINING_EPOCH_COUNT);

        # 从上下文中获取训练样本哈希值，并存储到train->shuffle_samples_hash中
        GGUF_GET_KEY(fctx, train->shuffle_samples_hash,      gguf_get_val_u64, GGUF_TYPE_UINT64, false, LLM_KV_TRAINING_SHUFFLE_SAMPLES_HASH);
        # 从上下文中获取当前随机数生成器状态，并存储到train->shuffle_rng_state_current中
        GGUF_GET_KEY(fctx, train->shuffle_rng_state_current, gguf_get_val_str, GGUF_TYPE_STRING, false, LLM_KV_TRAINING_SHUFFLE_RNG_STATE);
        # 从上下文中获取洗牌后的样本数量，并存储到train->shuffle_sample_count中
        GGUF_GET_KEY(fctx, train->shuffle_sample_count,      gguf_get_val_u64, GGUF_TYPE_UINT64, false, LLM_KV_TRAINING_SHUFFLE_SAMPLE_COUNT);
        # 从上下文中获取下一个洗牌样本的索引，并存储到train->shuffle_next_sample中
        GGUF_GET_KEY(fctx, train->shuffle_next_sample,       gguf_get_val_u64, GGUF_TYPE_UINT64, false, LLM_KV_TRAINING_SHUFFLE_NEXT_SAMPLE);
    }

    # 加载优化上下文到训练上下文中
    load_opt_context_gguf(fctx, f_ggml_ctx, train->opt);
    # 返回true
    return true;
// 保存训练状态到 gguf 上下文中
void save_train_state_gguf(struct gguf_context * fctx, struct train_state * train) {
    // 设置训练文件版本号
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_FILE_VERSION,    1);
    // 设置训练迭代次数
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_ITERATION_COUNT, train->train_its);
    // 设置训练样本数量
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_SAMPLE_COUNT,    train->train_samples);
    // 设置训练标记数量
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_TOKEN_COUNT,     train->train_tokens);
    // 设置训练周期数量
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_EPOCH_COUNT,     train->train_epochs);

    // 设置训练样本哈希值
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_SHUFFLE_SAMPLES_HASH, (uint64_t) train->shuffle_samples_hash);
    // 设置训练样本随机数生成器状态
    gguf_set_val_str(fctx, LLM_KV_TRAINING_SHUFFLE_RNG_STATE,    train->shuffle_rng_state_current.c_str());
    // 设置训练样本洗牌数量
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_SHUFFLE_SAMPLE_COUNT, (uint64_t) train->shuffle_sample_count);
    // 设置训练样本下一个洗牌样本
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_SHUFFLE_NEXT_SAMPLE,  (uint64_t) train->shuffle_next_sample);

    // 保存优化上下文到 gguf 上下文中
    save_opt_context_gguf(fctx, train->opt);
}

// llama_file 结构体
struct llama_file {
    // 使用 FILE * 以便不必重新打开文件进行内存映射
    FILE * fp;
    size_t size;

    // 构造函数，打开文件并获取文件大小
    llama_file(const char * fname, const char * mode) {
        fp = std::fopen(fname, mode);
        if (fp == NULL) {
            size = 0;
        } else {
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
        GGML_ASSERT(ret != -1); // 这不应该失败
        return (size_t) ret;
    }

    // 移动文件指针位置
    void seek(size_t offset, int whence) {
#ifdef _WIN32
        int ret = _fseeki64(fp, (__int64) offset, whence);
#else
        int ret = std::fseek(fp, (long) offset, whence);
#endif
        GGML_ASSERT(ret == 0); // 同上
    }
}
    // 从文件指针中读取指定大小的数据到指定的内存地址
    void read_raw(void * ptr, size_t size) {
        // 如果大小为0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 从文件指针中读取数据到指定内存地址，返回读取的元素个数
        std::size_t ret = std::fread(ptr, size, 1, fp);
        // 如果发生读取错误，则输出错误信息并终止程序
        if (ferror(fp)) {
            die_fmt("read error: %s", strerror(errno));
        }
        // 如果读取的元素个数不等于1，则表示意外地到达了文件末尾，终止程序
        if (ret != 1) {
            die("unexpectedly reached end of file");
        }
    }

    // 读取一个32位无符号整数
    std::uint32_t read_u32() {
        std::uint32_t ret;
        // 调用read_raw函数读取32位无符号整数
        read_raw(&ret, sizeof(ret));
        return ret;
    }

    // 读取指定长度的字符串
    std::string read_string(std::uint32_t len) {
        // 创建一个指定长度的字符向量
        std::vector<char> chars(len);
        // 调用read_raw函数读取指定长度的字符串
        read_raw(chars.data(), len);
        // 将字符向量转换为字符串并返回
        return std::string(chars.data(), len);
    }

    // 将指定大小的数据写入文件指针
    void write_raw(const void * ptr, size_t size) {
        // 如果大小为0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 将数据写入文件指针，返回写入的元素个数
        size_t ret = std::fwrite(ptr, size, 1, fp);
        // 如果写入的元素个数不等于1，则表示发生写入错误，输出错误信息并终止程序
        if (ret != 1) {
            die_fmt("write error: %s", strerror(errno));
        }
    }

    // 将32位无符号整数写入文件指针
    void write_u32(std::uint32_t val) {
        // 调用write_raw函数将32位无符号整数写入文件指针
        write_raw(&val, sizeof(val));
    }

    // 析构函数，关闭文件指针
    ~llama_file() {
        if (fp) {
            std::fclose(fp);
        }
    }
// 结束静态代码块
};

// 计算 UTF-8 编码的字符长度
static size_t utf8_len(char src) {
    // UTF-8 编码长度查找表
    const size_t lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3, 4 };
    // 获取字符的高位字节
    uint8_t highbits = static_cast<uint8_t>(src) >> 4;
    // 返回字符的 UTF-8 编码长度
    return lookup[highbits];
}

// 标记每个字节的 UTF-8 单元编号
// 返回 UTF-8 字符的数量
// 例如，当 bytes == '\x61\xD0\xB0\x62' 时，
// utf8_units 将变为 [0,0,1,0]
// utf8_nunits 将变为 [1,2,2,1]，并返回 3
// utf8_units 为零的字节是 UTF-8 字符的起始字节
static size_t mark_utf8_units(const char* bytes, int * utf8_units, int * utf8_nunits, size_t count) {
    size_t offs = 0;
    size_t count_utf8 = 0;
    while(offs < count) {
        int len = (int) utf8_len(bytes[offs]);
        for (int i=0; i<len; ++i) {
            utf8_units[offs+i]  = i;
            utf8_nunits[offs+i] = len;
        }
        offs += len;
        ++count_utf8;
    }
    return count_utf8;
}

// 对文件进行标记化处理
size_t tokenize_file(
        struct llama_context     * lctx,
        const char               * filename,
        const std::string        & sample_start,
        bool                       include_sample_start,
        bool                       overlapping_samples,
        unsigned                   context_length,
        std::vector<llama_token> & out_tokens,
        std::vector<size_t>      & out_samples_begin,
        std::vector<size_t>      & out_samples_size) {
    // 打开文件
    struct llama_file f(filename, "rb");

    // 如果文件大小为 0，则清空输出并返回
    if (f.size == 0) {
        out_tokens.clear();
        out_samples_begin.clear();
        out_samples_size.clear();
        printf("%s: warning: empty or not existing training data file '%s'\n",
            __func__, filename);
        return out_tokens.size();
    }

    // 考虑标记器可能添加的前导空白
    const int n_max_tokens_overhead = 1;

    // 创建缓冲区并读取文件内容
    std::vector<char> buf;
    buf.resize(f.size);
    f.read_raw(buf.data(), f.size);
    // 创建存储 UTF-8 单元和单元数量的向量
    std::vector<int> utf8_units;
    std::vector<int> utf8_nunits;
    // 调整向量大小以匹配输入缓冲区大小
    utf8_units.resize(buf.size());
    utf8_nunits.resize(buf.size());
    // 标记 UTF-8 单元
    mark_utf8_units(buf.data(), utf8_units.data(), utf8_nunits.data(), buf.size());

    // 如果样本起始位置为0
    if (sample_start.size() == 0) {
        // 一次性对所有数据进行标记化
        out_tokens.resize(buf.size() + n_max_tokens_overhead);

        // 对数据进行标记化
        int n_tokens = llama_tokenize(
            llama_get_model(lctx),
            buf.data(),
            (int) buf.size(),
            out_tokens.data(),
            (int) out_tokens.size(),
            false, false);
        // 如果标记化失败，重新调整输出 tokens 的大小并重试
        if (n_tokens < 0) {
            out_tokens.resize(-n_tokens);
            n_tokens = llama_tokenize(
                llama_get_model(lctx),
                buf.data(),
                (int) buf.size(),
                out_tokens.data(),
                (int) out_tokens.size(),
                false, false);
        }
        // 如果标记化成功，调整输出 tokens 的大小
        if (n_tokens >= 0) {
            out_tokens.resize(n_tokens);
        }

        // 生成所有标记位置的样本起始位置
        out_samples_begin.clear();
        out_samples_begin.push_back(0);
        out_samples_size.push_back(std::min((size_t) context_length, out_tokens.size()));
        size_t end = (out_tokens.size() >= context_length) ? (out_tokens.size() - context_length) : 0;
        for (size_t sample_begin = 1; sample_begin < end; ++sample_begin) {
            out_samples_begin.push_back(sample_begin);
            out_samples_size.push_back(context_length);
        }
    }
    // 打印函数名称和样本起始位置的总数
    printf("%s: total number of samples: %zu\n",
        __func__, out_samples_begin.size());

    // 断言样本起始位置和样本大小的数量相等
    GGML_ASSERT(out_samples_begin.size() == out_samples_size.size());

    // 返回输出 tokens 的大小
    return out_tokens.size();
}

// 根据文件名、模式、最新值和迭代次数生成训练文件名
std::string get_train_filename(const char * filename, const char * pattern_it, const char * latest, int64_t iteration) {
    // 如果迭代次数大于等于0，使用迭代次数转换为字符串，否则使用最新值
    std::string sit = (iteration >= 0) ? std::to_string(iteration) : std::string(latest);
    // 调用 replace_str 函数替换文件名中的模式为生成的字符串
    return replace_str(filename, pattern_it, sit.c_str());
}

// 获取默认的训练参数
struct train_params_common get_default_train_params_common() {
    struct train_params_common params;
    // 设置默认的训练数据文件名
    params.fn_train_data     = "shakespeare.txt";
    // 设置默认的检查点输入文件名
    params.fn_checkpoint_in  = "checkpoint.gguf";
    // 设置默认的检查点输出文件名模式
    params.fn_checkpoint_out = "checkpoint-ITERATION.gguf";
    // 设置默认的文件名模式
    params.pattern_fn_it     = "ITERATION";
    // 设置默认的最新值
    params.fn_latest         = "LATEST";

    // 设置是否打印用法信息的标志
    params.print_usage = false;

    // 设置每隔多少次迭代保存一次检查点
    params.save_every = 10;

    // 设置随机数种子
    params.seed       =   -1;

    // 设置上下文大小
    params.n_ctx      =  128;
    // 设置线程数
    params.n_threads  =    6;
    // 设置批次大小
    params.n_batch    =    8;
    // 设置梯度累积次数
    params.n_gradient_accumulation = 1;
    // 设置训练轮数
    params.n_epochs   = -1;
    // 设置 GPU 层数
    params.n_gpu_layers = 0;

    // 设置是否使用自定义上下文大小的标志
    params.custom_n_ctx = false;

    // 设置是否使用 Flash 存储的标志
    params.use_flash              = true;
    // 设置是否使用检查点的标志
    params.use_checkpointing      = true;

    // 设置采样起始位置
    params.sample_start           = "";
    // 设置是否包含采样起始位置的标志
    params.include_sample_start   = false;
    // 设置是否转义特殊字符的标志
    params.escape                 = false;
    // 设置是否重叠采样的标志
    params.overlapping_samples    = false;
    // 设置是否用下一个样本填充的标志
    params.fill_with_next_samples = false;
    // 设置是否以 EOS 分隔的标志
    params.separate_with_eos      = false;
    // 设置是否以 BOS 分隔的标志
    params.separate_with_bos      = true;
    // 设置是否随机偏移采样的标志
    params.sample_random_offsets  = false;
    // 设置是否强制重新洗牌的标志
    params.force_reshuffle        = false;

    // 设置优化器的过去步数
    params.opt_past               = 0;
    // 设置优化器的学习率
    params.opt_delta              = 1e-5f;
    // 设置优化器的最大不改善步数
    params.opt_max_no_improvement = 0;

    // 设置预热步数
    params.warmup            =  100;
    // 设置余弦退火的步数
    params.cos_decay_steps   = 1000;
    // 设置余弦退火的重启系数
    params.cos_decay_restart = 1.1f;
    // 设置余弦退火的最小值
    params.cos_decay_min     = 0.1f;
    // 设置是否启用重启的标志
    params.enable_restart    = false;

    // 设置 Adam 优化器的迭代次数
    params.adam_n_iter         = 256;
    // 设置 Adam 优化器的初始学习率
    params.adam_alpha          = 1e-3f;
    // 设置 Adam 优化器的最小学习率
    params.adam_min_alpha      = 0;
    // 设置 Adam 优化器的学习率衰减
    params.adam_decay          = 1e-1f;
    // 设置 Adam 优化器的最小维度
    params.adam_decay_min_ndim = 2;
    // 设置 Adam 优化器的一阶矩估计指数衰减率
    params.adam_beta1          = 0.9f;
    // 设置 Adam 优化器的二阶矩估计指数衰减率
    params.adam_beta2          = 0.999f;
    # 设置参数 adam_gclip 的值为 1.0f
    params.adam_gclip          = 1.0f;
    # 设置参数 adam_eps_f 的值为 0.0f
    params.adam_eps_f          = 0.0f;

    # 返回参数对象 params
    return params;
}
// 打印通用的训练参数使用说明
void print_common_train_usage(int /*argc*/, char ** /*argv*/, const struct train_params_common * params) {
    // 打印使用说明的标题
    // fprintf(stderr, "usage: %s [options]\n", argv[0]);
    // fprintf(stderr, "\n");
    // fprintf(stderr, "options:\n");
    // fprintf(stderr, "  -h, --help                 show this help message and exit\n");
    // 打印训练数据文件路径的默认值
    fprintf(stderr, "  --train-data FNAME         path from which to load training data (default '%s')\n", params->fn_train_data);
    // 打印训练检查点输入文件路径的默认值
    fprintf(stderr, "  --checkpoint-in FNAME      path from which to load training checkpoint (default '%s')\n", params->fn_checkpoint_in);
    // 打印训练检查点输出文件路径的默认值
    fprintf(stderr, "  --checkpoint-out FNAME     path to save training checkpoint (default '%s')\n", params->fn_checkpoint_out);
    // 打印输出文件名中迭代号模式的默认值
    fprintf(stderr, "  --pattern-fn-it STR        pattern in output filenames to be replaced by iteration number (default '%s')\n", params->pattern_fn_it);
    // 打印保存最新输出文件名的默认值
    fprintf(stderr, "  --fn-latest STR            string to use instead of iteration number for saving latest output (default '%s')\n", params->fn_latest);
    // 打印每N次迭代保存检查点和输出文件的默认值
    fprintf(stderr, "  --save-every N             save checkpoint and lora every N iterations. Disabled when N <= 0. (default '%d')\n", params->save_every);
    // 打印随机数生成器种子的默认值
    fprintf(stderr, "  -s SEED, --seed SEED       RNG seed (default: -1, use random seed for -1)\n");
    // 打印训练过程中使用的上下文大小的默认值
    fprintf(stderr, "  -c N, --ctx N              Context size used during training (default %d)\n", params->n_ctx);
    // 打印线程数的默认值
    fprintf(stderr, "  -t N, --threads N          Number of threads (default %d)\n", params->n_threads);
    // 打印并行批处理大小的默认值
    fprintf(stderr, "  -b N, --batch N            Parallel batch size (default %d)\n", params->n_batch);
    // 打印梯度累积步数的默认值
    fprintf(stderr, "  --grad-acc N               Number of gradient accumulation steps (simulates larger batch size of batch*gradacc) (default %d)\n", params->n_gradient_accumulation);
}
    # 输出指定格式的字符串到标准错误流，设置了起始点后的样本起始位置。如果为空，则使用每个标记位置作为样本起始点。(默认为params->sample_start.c_str())
    fprintf(stderr, "  --sample-start STR         Sets the starting point for samples after the specified pattern. If empty use every token position as sample start. (default '%s')\n", params->sample_start.c_str());
    # 输出指定格式的字符串到标准错误流，包括样本起始点在样本中。(默认关闭)
    fprintf(stderr, "  --include-sample-start     Include the sample start in the samples. (default off)\n");
    # 输出指定格式的字符串到标准错误流，处理样本起始点的转义序列(\\n, \\r, \\t, \\', \\\", \\\\)
    fprintf(stderr, "  --escape                   process sample start escapes sequences (\\n, \\r, \\t, \\', \\\", \\\\)\n");
    # 输出指定格式的字符串到标准错误流，样本可能重叠，将包括第二个和后续样本的样本起始点。关闭时，样本将在下一个样本的开头结束。(默认关闭)
    fprintf(stderr, "  --overlapping-samples      Samples my overlap, will include sample-start of second and following samples. When off, samples will end at begin of next sample. (default off)\n");
    # 输出指定格式的字符串到标准错误流，长度短于上下文长度的样本将由下一个(打乱顺序的)样本跟随。(默认关闭)
    fprintf(stderr, "  --fill-with-next-samples   Samples shorter than context length will be followed by the next (shuffled) samples. (default off)\n");
    # 输出指定格式的字符串到标准错误流，当fill-with-next-samples时，在样本之间插入结束序列标记。(默认为params->separate_with_eos ? " (default)" : "")
    fprintf(stderr, "  --separate-with-eos        When fill-with-next-samples, insert end-of-sequence token between samples.%s\n", params->separate_with_eos ? " (default)" : "");
    # 输出指定格式的字符串到标准错误流，当fill-with-next-samples时，在样本之间插入开始序列标记。(默认为params->separate_with_bos ? " (default)" : "")
    fprintf(stderr, "  --separate-with-bos        When fill-with-next-samples, insert begin-of-sequence token between samples.%s\n", params->separate_with_bos ? " (default)" : "");
    # 输出指定格式的字符串到标准错误流，当fill-with-next-samples时，不在样本之间插入结束序列标记。(默认为!params->separate_with_eos ? " (default)" : "")
    fprintf(stderr, "  --no-separate-with-eos     When fill-with-next-samples, don't insert end-of-sequence token between samples.%s\n", !params->separate_with_eos ? " (default)" : "");
    # 输出指定格式的字符串到标准错误流，当fill-with-next-samples时，不在样本之间插入开始序列标记。(默认为!params->separate_with_bos ? " (default)" : "")
    fprintf(stderr, "  --no-separate-with-bos     When fill-with-next-samples, don't insert begin-of-sequence token between samples.%s\n", !params->separate_with_bos ? " (default)" : "");
    # 输出指定格式的字符串到标准错误流，使用从随机偏移开始的样本。与fill-with-next-samples一起使用，这可能有助于训练无限文本生成。(默认为params->sample_random_offsets ? " (default)" : "")
    fprintf(stderr, "  --sample-random-offsets    Use samples beginning at random offsets. Together with fill-with-next-samples this may help for training endless text generation.%s\n", params->sample_random_offsets ? " (default)" : "");
    # 输出指定格式的字符串到标准错误流，强制在程序启动时重新洗牌数据，否则将恢复加载的检查点的洗牌。
    fprintf(stderr, "  --force-reshuffle          Force a reshuffling of data at program start, otherwise the shuffling of loaded checkpoint is resumed.\n");
    # 输出不使用闪光注意力的选项
    fprintf(stderr, "  --no-flash                 Don't use flash attention \n");
    # 输出使用闪光注意力的选项（默认）
    fprintf(stderr, "  --use-flash                Use flash attention (default)\n");
    # 输出不使用梯度检查点的选项
    fprintf(stderr, "  --no-checkpointing         Don't use gradient checkpointing\n");
    # 输出使用梯度检查点的选项（默认）
    fprintf(stderr, "  --use-checkpointing        Use gradient checkpointing (default)\n");
    # 输出仅适用于Adam优化器的热身步数选项（默认为params->warmup）
    fprintf(stderr, "  --warmup N                 Only for Adam optimizer. Number of warmup steps (default %d)\n", params->warmup);
    # 输出仅适用于Adam优化器的余弦衰减步数选项（默认为params->cos_decay_steps）
    fprintf(stderr, "  --cos-decay-steps N        Only for Adam optimizer. Number of cosine decay steps (default %d)\n", params->cos_decay_steps);
    # 输出仅适用于Adam优化器的重启后余弦衰减步数增加量选项（默认为params->cos_decay_restart）
    fprintf(stderr, "  --cos-decay-restart N      Only for Adam optimizer. Increase of cosine decay steps after restart (default %f)\n", params->cos_decay_restart);
    # 输出仅适用于Adam优化器的余弦衰减最小值选项（默认为params->cos_decay_min）
    fprintf(stderr, "  --cos-decay-min N          Only for Adam optimizer. Cosine decay minimum (default %f)\n", params->cos_decay_min);
    # 输出仅适用于Adam优化器的启用余弦衰减重启选项（默认为params->enable_restart）
    fprintf(stderr, "  --enable-restart N         Only for Adam optimizer. Enable restarts of cos-decay %s\n", params->enable_restart ? "(default)" : "");
    # 输出仅适用于Adam优化器的禁用余弦衰减重启选项（默认为!params->enable_restart）
    fprintf(stderr, "  --disable-restart N        Only for Adam optimizer. Disable restarts of cos-decay %s\n", !params->enable_restart ? "(default)" : "");
    # 输出用于跟踪优化迭代次数以进行增量收敛测试的优化迭代次数选项（默认为params->opt_past）
    fprintf(stderr, "  --opt-past N               Number of optimization iterations to track for delta convergence test. Disabled when zero. (default %d)\n", params->opt_past);
    # 输出增量收敛测试的最大增量选项（默认为params->opt_delta）
    fprintf(stderr, "  --opt-delta N              Maximum delta for delta convergence test. Disabled when <= zero. (default %f)\n", params->opt_delta);
    # 输出无改进的最大优化迭代次数选项（默认为params->opt_max_no_improvement）
    fprintf(stderr, "  --opt-max-no-improvement N Maximum number of optimization iterations with no improvement. Disabled when <= zero. (default %d)\n", params->opt_max_no_improvement);
    # 输出处理的最大周期数选项（默认为params->n_epochs）
    fprintf(stderr, "  --epochs N                 Maximum number epochs to process. (default %d)\n", params->n_epochs);
    # 输出 Adam 优化器的最大迭代次数
    fprintf(stderr, "  --adam-iter N              Maximum number of Adam optimization iterations for each batch (default %d)\n", params->adam_n_iter);
    # 输出 Adam 学习率 alpha
    fprintf(stderr, "  --adam-alpha N             Adam learning rate alpha (default %f)\n", params->adam_alpha);
    # 输出 Adam 最小学习率 alpha，包括预热阶段
    fprintf(stderr, "  --adam-min-alpha N         Adam minimum learning rate alpha - including warmup phase (default %f)\n", params->adam_min_alpha);
    # 输出 AdamW 权重衰减。大于零的值启用 AdamW 而不是常规 Adam
    fprintf(stderr, "  --adam-decay N             AdamW weight decay. Values greater zero enable AdamW instead of regular Adam. (default %f)\n", params->adam_decay);
    # 输出应用 AdamW 权重衰减的张量的最小维数。维数小于此值的张量不应用权重衰减
    fprintf(stderr, "  --adam-decay-min-ndim N    Minimum number of tensor dimensions to apply AdamW weight decay. Weight decay is not applied to tensors with less n_dims. (default %d)\n", params->adam_decay_min_ndim);
    # 输出 AdamW beta1，范围在 [0,1)。用于平滑梯度的第一时刻
    fprintf(stderr, "  --adam-beta1 N             AdamW beta1 in interval [0,1). How much to smooth the first moment of gradients. (default %f)\n", params->adam_beta1);
    # 输出 AdamW beta2，范围在 [0,1)。用于平滑梯度的第二时刻
    fprintf(stderr, "  --adam-beta2 N             AdamW beta2 in interval [0,1). How much to smooth the second moment of gradients. (default %f)\n", params->adam_beta2);
    # 输出 AdamW 梯度裁剪。当值为零时禁用
    fprintf(stderr, "  --adam-gclip N             AdamW gradient clipping. Disabled when zero. (default %f)\n", params->adam_gclip);
    # 输出 AdamW 收敛测试的 epsilon。当小于等于零时禁用
    fprintf(stderr, "  --adam-epsf N              AdamW epsilon for convergence test. Disabled when <= zero. (default %f)\n", params->adam_eps_f);
    # 输出空行
    fprintf(stderr, "\n");
# 检查是否有足够的参数来解析训练参数，如果不够则返回 false
bool consume_common_train_arg(
    int argc, char ** argv, int * idx, struct train_params_common * params, bool * invalid_param
) {
    # 获取参数索引的引用
    int& i = *idx;
    # 获取当前参数
    std::string arg = argv[i];
    # 定义参数前缀
    const std::string arg_prefix = "--";
    # 检查参数是否以前缀开头，如果是则替换下划线为破折号
    if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
        std::replace(arg.begin(), arg.end(), '_', '-');
    }
    # 检查参数类型并解析对应的值
    if (arg == "--train-data") {
        # 检查是否有足够的参数，如果不够则返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数值赋给对应的参数
        params->fn_train_data = argv[i];
    } else if (arg == "--checkpoint-in") {
        # 检查是否有足够的参数，如果不够则返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数值赋给对应的参数
        params->fn_checkpoint_in = argv[i];
    } else if (arg == "--checkpoint-out") {
        # 检查是否有足够的参数，如果不够则返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数值赋给对应的参数
        params->fn_checkpoint_out = argv[i];
    } else if (arg == "--pattern-fn-it") {
        # 检查是否有足够的参数，如果不够则返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数值赋给对应的参数
        params->pattern_fn_it = argv[i];
    } else if (arg == "--fn-latest") {
        # 检查是否有足够的参数，如果不够则返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数值赋给对应的参数
        params->fn_latest = argv[i];
    } else if (arg == "--save-every") {
        # 检查是否有足够的参数，如果不够则返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数值赋给对应的参数，将字符串转换为整数
        params->save_every = std::stoi(argv[i]);
    } else if (arg == "-s" || arg == "--seed") {
        # 检查是否有足够的参数，如果不够则返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数值赋给对应的参数，将字符串转换为整数
        params->seed = std::stoi(argv[i]);
    } else if (arg == "-c" || arg == "--ctx") {
        # 检查是否有足够的参数，如果不够则返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数值赋给对应的参数，将字符串转换为整数
        params->n_ctx = std::stoi(argv[i]);
        # 设置自定义上下文标志为 true
        params->custom_n_ctx = true;
    } else if (arg == "-t" || arg == "--threads") {
        # 如果参数是"-t"或"--threads"，则获取下一个参数作为线程数，并转换为整数保存到params->n_threads中
        if (++i >= argc) {
            # 如果没有下一个参数，则将invalid_param设置为true，并返回true
            *invalid_param = true;
            return true;
        }
        params->n_threads = std::stoi(argv[i]);
    } else if (arg == "-b" || arg == "--batch") {
        # 如果参数是"-b"或"--batch"，则获取下一个参数作为批次数，并转换为整数保存到params->n_batch中
        if (++i >= argc) {
            # 如果没有下一个参数，则将invalid_param设置为true，并返回true
            *invalid_param = true;
            return true;
        }
        params->n_batch = std::stoi(argv[i]);
    } else if (arg == "--grad-acc") {
        # 如果参数是"--grad-acc"，则获取下一个参数作为梯度累积数，并转换为整数保存到params->n_gradient_accumulation中
        if (++i >= argc) {
            # 如果没有下一个参数，则将invalid_param设置为true，并返回true
            *invalid_param = true;
            return true;
        }
        params->n_gradient_accumulation = std::max(1, std::stoi(argv[i]));
    } else if (arg == "--sample-start") {
        # 如果参数是"--sample-start"，则获取下一个参数作为样本起始值，并保存到params->sample_start中
        if (++i >= argc) {
            # 如果没有下一个参数，则将invalid_param设置为true，并返回true
            *invalid_param = true;
            return true;
        }
        params->sample_start = std::string(argv[i]);
    } else if (arg == "--escape") {
        # 如果参数是"--escape"，则将params->escape设置为true
        params->escape = true;
    } else if (arg == "--include-sample-start") {
        # 如果参数是"--include-sample-start"，则将params->include_sample_start设置为true
        params->include_sample_start = true;
    } else if (arg == "--overlapping-samples") {
        # 如果参数是"--overlapping-samples"，则将params->overlapping_samples设置为true
        params->overlapping_samples = true;
    } else if (arg == "--fill-with-next-samples") {
        # 如果参数是"--fill-with-next-samples"，则将params->fill_with_next_samples设置为true
        params->fill_with_next_samples = true;
    } else if (arg == "--separate-with-eos") {
        # 如果参数是"--separate-with-eos"，则将params->separate_with_eos设置为true
        params->separate_with_eos = true;
    } else if (arg == "--separate-with-bos") {
        # 如果参数是"--separate-with-bos"，则将params->separate_with_bos设置为true
        params->separate_with_bos = true;
    } else if (arg == "--no-separate-with-eos") {
        # 如果参数是"--no-separate-with-eos"，则将params->separate_with_eos设置为false
        params->separate_with_eos = false;
    } else if (arg == "--no-separate-with-bos") {
        # 如果参数是"--no-separate-with-bos"，则将params->separate_with_bos设置为false
        params->separate_with_bos = false;
    } else if (arg == "--sample-random-offsets") {
        # 如果参数是"--sample-random-offsets"，则将params->sample_random_offsets设置为true
        params->sample_random_offsets = true;
    } else if (arg == "--force-reshuffle") {
        # 如果参数是"--force-reshuffle"，则将params->force_reshuffle设置为true
        params->force_reshuffle = true;
    } else if (arg == "--no-flash") {
        # 如果参数是"--no-flash"，则将params->use_flash设置为false
        params->use_flash = false;
    } else if (arg == "--use-flash") {
        # 如果参数是"--use-flash"，则将params->use_flash设置为true
        params->use_flash = true;
    } else if (arg == "--no-checkpointing") {
        # 如果参数是"--no-checkpointing"，则将params->use_checkpointing设置为false
        params->use_checkpointing = false;
    } else if (arg == "--use-checkpointing") {
        // 如果参数为“--use-checkpointing”，则将参数结构体中的 use_checkpointing 设置为 true
        params->use_checkpointing = true;
    } else if (arg == "--warmup") {
        // 如果参数为“--warmup”，则将参数结构体中的 warmup 设置为后续参数的整数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->warmup = std::stoi(argv[i]);
    } else if (arg == "--cos-decay-steps") {
        // 如果参数为“--cos-decay-steps”，则将参数结构体中的 cos_decay_steps 设置为后续参数的整数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->cos_decay_steps = std::stoi(argv[i]);
    } else if (arg == "--cos-decay-restart") {
        // 如果参数为“--cos-decay-restart”，则将参数结构体中的 cos_decay_restart 设置为后续参数的浮点数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->cos_decay_restart = std::stof(argv[i]);
    } else if (arg == "--cos-decay-min") {
        // 如果参数为“--cos-decay-min”，则将参数结构体中的 cos_decay_min 设置为后续参数的浮点数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->cos_decay_min = std::stof(argv[i]);
    } else if (arg == "--enable-restart") {
        // 如果参数为“--enable-restart”，则将参数结构体中的 enable_restart 设置为 true
        params->enable_restart = true;
    } else if (arg == "--disable-restart") {
        // 如果参数为“--disable-restart”，则将参数结构体中的 enable_restart 设置为 false
        params->enable_restart = false;
    } else if (arg == "--opt-past") {
        // 如果参数为“--opt-past”，则将参数结构体中的 opt_past 设置为后续参数的整数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->opt_past = std::stoi(argv[i]);
    } else if (arg == "--opt-delta") {
        // 如果参数为“--opt-delta”，则将参数结构体中的 opt_delta 设置为后续参数的浮点数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->opt_delta = std::stof(argv[i]);
    } else if (arg == "--opt-max-no-improvement") {
        // 如果参数为“--opt-max-no-improvement”，则将参数结构体中的 opt_max_no_improvement 设置为后续参数的整数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->opt_max_no_improvement = std::stoi(argv[i]);
    } else if (arg == "--adam-epsf") {
        // 如果参数为“--adam-epsf”，则将参数结构体中的 adam_eps_f 设置为后续参数的浮点数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_eps_f = std::stof(argv[i]);
    } else if (arg == "--epochs") {
        // 如果参数为“--epochs”，则将参数结构体中的 n_epochs 设置为后续参数的整数值
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->n_epochs = std::stoi(argv[i]);
    } else if (arg == "--adam-iter") {
        // 如果参数为"--adam-iter"，则获取下一个参数作为adam_n_iter，并转换为整数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_n_iter = std::stoi(argv[i]);
    } else if (arg == "--adam-alpha") {
        // 如果参数为"--adam-alpha"，则获取下一个参数作为adam_alpha，并转换为浮点数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_alpha = std::stof(argv[i]);
    } else if (arg == "--adam-min-alpha") {
        // 如果参数为"--adam-min-alpha"，则获取下一个参数作为adam_min_alpha，并转换为浮点数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_min_alpha = std::stof(argv[i]);
    } else if (arg == "--adam-decay") {
        // 如果参数为"--adam-decay"，则获取下一个参数作为adam_decay，并转换为浮点数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_decay = std::stof(argv[i]);
    } else if (arg == "--adam-decay-min-ndim") {
        // 如果参数为"--adam-decay-min-ndim"，则获取下一个参数作为adam_decay_min_ndim，并转换为整数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_decay_min_ndim = std::stoi(argv[i]);
    } else if (arg == "--adam-beta1") {
        // 如果参数为"--adam-beta1"，则获取下一个参数作为adam_beta1，并转换为浮点数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_beta1 = std::stof(argv[i]);
    } else if (arg == "--adam-beta2") {
        // 如果参数为"--adam-beta2"，则获取下一个参数作为adam_beta2，并转换为浮点数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_beta2 = std::stof(argv[i]);
    } else if (arg == "--adam-gclip") {
        // 如果参数为"--adam-gclip"，则获取下一个参数作为adam_gclip，并转换为浮点数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_gclip = std::stof(argv[i]);
    } else if (arg == "-h" || arg == "--help") {
        // 如果参数为"-h"或"--help"，则设置print_usage为true，并返回true
        params->print_usage = true;
        return true;
    } else {
        // 其他情况返回false
        return false;
    }
    // 默认返回true
    return true;
}

void finish_processing_train_args(struct train_params_common * params) {
    // 如果设置了escape参数，对样本起始位置进行处理
    if (params->escape) {
        process_escapes(params->sample_start);
    }
}

void train_opt_callback(void * vdata, int accum_step, float * sched, bool * cancel) {
    // 将void指针类型的数据转换为train_opt_callback_data类型
    struct train_opt_callback_data * data   = (struct train_opt_callback_data *) vdata;
    // 获取训练参数
    struct train_params_common     * params = data->params;
    // 获取训练状态
    struct train_state             * train  = data->train;
    // 获取优化器上下文
    struct ggml_opt_context        * opt    = train->opt;
    // 获取批次大小
    int n_batch = params->n_batch;
    // 获取上下文大小
    int n_ctx = params->n_ctx;

    }

    // 获取使用的样本数
    int64_t used_samples = get_example_targets_batch(
        data->lctx,
        data->tokens_input,
        data->target_probs,
        train->shuffle_next_sample,
        data->shuffled_samples_offs,
        data->shuffled_samples_begin,
        data->shuffled_samples_size,
        data->samples_count,
        data->tokens_data,
        data->tokens_size,
        params->separate_with_eos,
        params->separate_with_bos,
        params->fill_with_next_samples,
        params->sample_random_offsets);

    // 更新训练样本数和下一个需要洗牌的样本位置
    train->train_samples += used_samples;
    train->shuffle_next_sample += used_samples;

    // 如果下一个需要洗牌的样本位置超过了洗牌样本总数
    if (train->shuffle_next_sample >= train->shuffle_sample_count) {
        // 增加训练轮数，并打印信息
        ++train->train_epochs;
        printf("%s: reshuffle samples. completed epochs: %llu\n", __func__, (long long unsigned) train->train_epochs);
        // 注意：当前洗牌可能会多次使用当前洗牌的一些样本
        // 更新当前和下一个洗牌的随机数状态，并重新洗牌样本
        train->shuffle_rng_state_current = train->shuffle_rng_state_next;
        train->shuffle_rng_state_next = shuffle_samples(
            train->shuffle_rng_state_current,
            data->shuffled_samples_offs,
            data->shuffled_samples_begin,
            data->shuffled_samples_size,
            data->samples_begin,
            data->samples_size,
            data->samples_count);
        train->shuffle_next_sample = 0;
    }
    # 检查是否达到了最后一个周期
    const bool last_epoch_reached = (params->n_epochs > 0 && (int64_t) train->train_epochs - data->first_epoch >= params->n_epochs);
    # 如果达到了最后一个周期
    if (last_epoch_reached) {
        # 允许在最后一个周期完成优化迭代之前取消
        if (data->iter_at_last_epoch < 0) {
            data->iter_at_last_epoch = opt->iter;
        } else if (opt->iter > data->iter_at_last_epoch) {
            # 如果当前迭代次数大于最后一个周期的迭代次数，则取消优化
            *cancel = true;
        }
    }
# 闭合前面的函数定义
```