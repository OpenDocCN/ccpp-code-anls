# `PowerInfer\common\train.cpp`

```
// 包含头文件 train.h 和 common.h
#include "train.h"
#include "common.h"

// 包含随机数生成、字符串流和函数式编程相关的头文件
#include <random>
#include <sstream>
#include <functional>

// 定义随机正态分布结构体
struct random_normal_distribution {
    std::mt19937 gen; // Mersenne Twister 19937 伪随机数生成器
    std::normal_distribution<float> rd; // 正态分布随机数生成器
    float min; // 最小值
    float max; // 最大值
};

// 定义随机均匀分布结构体
struct random_uniform_distribution {
    std::mt19937 gen; // Mersenne Twister 19937 伪随机数生成器
    std::uniform_real_distribution<float> rd; // 均匀分布随机数生成器
};

// 初始化训练状态结构体
struct train_state  * init_train_state() {
// 创建一个名为 state 的指向 train_state 结构体的指针
struct train_state * state = new struct train_state;
// 初始化 train_state 结构体中的各个成员变量
state->train_its = 0;
state->train_samples = 0;
state->train_tokens = 0;
state->train_epochs = 0;
state->shuffle_samples_hash = 0;
state->shuffle_sample_count = 0;
state->shuffle_next_sample = 0;
state->shuffle_rng_state_current = "";
state->shuffle_rng_state_next = "";

// 为 state 结构体中的 opt 成员变量分配内存，并初始化其成员变量
state->opt = new struct ggml_opt_context;
state->opt->ctx = NULL;
state->opt->params = ggml_opt_default_params(GGML_OPT_ADAM);
state->opt->params.graph_size = LLAMA_TRAIN_MAX_NODES;
state->opt->loss_after = 0.0f;

// 返回初始化后的 state 结构体指针
return state;
// 释放训练状态结构体的内存
void free_train_state(struct train_state  * state) {
    // 释放状态结构体中的 opt 指针所指向的内存
    delete state->opt;
    // 释放状态结构体的内存
    delete state;
}

// 初始化正态分布随机数生成器
struct random_normal_distribution * init_random_normal_distribution(
    int seed, float mean, float std, float min, float max
) {
    // 分配内存给正态分布随机数生成器结构体
    struct random_normal_distribution * rnd = (struct random_normal_distribution *) malloc(sizeof(struct random_normal_distribution));
    // 使用给定的种子初始化随机数生成器
    rnd->gen = std::mt19937(seed);
    // 初始化正态分布随机数生成器
    rnd->rd = std::normal_distribution<float>{mean, std};
    // 设置最小值和最大值
    rnd->min = min;
    rnd->max = max;
    // 返回初始化后的正态分布随机数生成器
    return rnd;
}

// 初始化均匀分布随机数生成器
struct random_uniform_distribution * init_random_uniform_distribution(int seed, float min, float max) {
    // 分配内存给均匀分布随机数生成器结构体
    struct random_uniform_distribution * rnd = (struct random_uniform_distribution *) malloc(sizeof(struct random_uniform_distribution));
    // 使用给定的种子初始化随机数生成器
    rnd->gen = std::mt19937(seed);
    // 初始化均匀分布随机数生成器
    rnd->rd = std::uniform_real_distribution<float>{min, max};
    // 返回初始化后的均匀分布随机数生成器
    return rnd;
}
// 返回随机数
return rnd;
}

// 释放正态分布随机数结构体
void free_random_normal_distribution (struct random_normal_distribution  * rnd) {
    free(rnd);
}

// 释放均匀分布随机数结构体
void free_random_uniform_distribution(struct random_uniform_distribution * rnd) {
    free(rnd);
}

// 对张量进行正态分布随机化
struct ggml_tensor * randomize_tensor_normal(struct ggml_tensor * tensor, struct random_normal_distribution * rnd) {
    float scale = 1.0f; // xavier
    switch (tensor->n_dims) {
        case 1:
            scale /= sqrtf((float) tensor->ne[0]);
            for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0]);
                *dst = scale * frand_normal(rnd);
            }
```

        // 结束当前的 switch 语句
        break;
        // 如果 tensor 的维度是 2
        case 2:
            // 根据 tensor 的维度调整比例
            scale /= sqrtf((float) tensor->ne[0]+tensor->ne[1]);
            // 遍历 tensor 的第二维和第一维
            for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                    // 计算目标地址
                    float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1]);
                    // 生成服从正态分布的随机数，并乘以比例，存入目标地址
                    *dst = scale * frand_normal(rnd);
                }
            }
            // 结束当前的 switch 语句
            break;
        // 如果 tensor 的维度是 3
        case 3:
            // 根据 tensor 的维度调整比例
            scale /= sqrtf((float) tensor->ne[0]+tensor->ne[1]);
            // 遍历 tensor 的第三维、第二维和第一维
            for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                    for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                        // 计算目标地址
                        float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2]);
                        // 生成服从正态分布的随机数，并乘以比例，存入目标地址
                        *dst = scale * frand_normal(rnd);
                    }
                }
            }
# 根据不同的情况进行处理
switch (tensor->n_dims) {
    # 如果 n_dims 为 0，直接返回 tensor
    case 0:
        return tensor;
    # 如果 n_dims 为 1，将 scale 缩小为 tensor->ne[0] 和 tensor->ne[1] 的平方根的倒数
    case 1:
        scale /= sqrtf((float) tensor->ne[0]+tensor->ne[1]);
        # 遍历 tensor->ne[3]、tensor->ne[2]、tensor->ne[1]、tensor->ne[0]，分别对应四维数组的索引
        for (int i3 = 0; i3 < tensor->ne[3]; i3++) {
            for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                    for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                        # 计算数据在数组中的偏移量，然后将随机数乘以 scale 存入数组中
                        float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3]);
                        *dst = scale * frand_normal(rnd);
                    }
                }
            }
        }
        break;
    # 如果 n_dims 不是 0 或 1，输出错误信息
    default:
        die("Unsupported tensor->n_dims");
};
# 返回处理后的 tensor
return tensor;
# 使用均匀分布的随机数填充张量数据
struct ggml_tensor * randomize_tensor_uniform(struct ggml_tensor * tensor, struct random_uniform_distribution * rnd) {
    # 根据张量的维度进行不同的填充操作
    switch (tensor->n_dims) {
        # 对于一维张量
        case 1:
            # 遍历一维张量的元素，填充随机数
            for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                # 计算当前元素在数据中的偏移位置
                float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0]);
                # 填充随机数
                *dst = frand_uniform(rnd);
            }
            break;
        # 对于二维张量
        case 2:
            # 遍历二维张量的元素，填充随机数
            for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                    # 计算当前元素在数据中的偏移位置
                    float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1]);
                    # 填充随机数
                    *dst = frand_uniform(rnd);
                }
            }
            break;
        # 对于三维张量
        case 3:
            # 遍历三维张量的元素，填充随机数
            for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                    for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                        # 计算当前元素在数据中的偏移位置
// 根据不同维度的张量，遍历每个元素并赋予随机数值
switch (tensor->n_dims) {
    // 当张量维度为3时
    case 3:
        // 遍历第三维度
        for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
            // 遍历第二维度
            for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                // 遍历第一维度
                for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                    // 计算元素在内存中的偏移位置，并将随机数值赋给该位置
                    float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2]);
                    *dst = frand_uniform(rnd);
                }
            }
        }
        break;
    // 当张量维度为4时
    case 4:
        // 遍历第四维度
        for (int i3 = 0; i3 < tensor->ne[3]; i3++) {
            // 遍历第三维度
            for (int i2 = 0; i2 < tensor->ne[2]; i2++) {
                // 遍历第二维度
                for (int i1 = 0; i1 < tensor->ne[1]; i1++) {
                    // 遍历第一维度
                    for (int i0 = 0; i0 < tensor->ne[0]; i0++) {
                        // 计算元素在内存中的偏移位置，并将随机数值赋给该位置
                        float * dst = (float *) ((char *) tensor->data + i0*tensor->nb[0] + i1*tensor->nb[1] + i2*tensor->nb[2] + i3*tensor->nb[3]);
                        *dst = frand_uniform(rnd);
                    }
                }
            }
        }
        break;
    // 当张量维度不为3或4时
    default:
        // 报错，不支持的张量维度
        die("Unsupported tensor->n_dims");
}
    };
    return tensor;
}

// 生成一个 0 到 1 之间的随机浮点数
float frand() {
    return (float)rand()/((float)(RAND_MAX) + 1.0f);
}

// 生成一个服从正态分布的随机浮点数
float frand_normal(struct random_normal_distribution * rnd) {
    return fclamp(rnd->rd(rnd->gen), rnd->min, rnd->max);
}

// 生成一个服从均匀分布的随机浮点数
float frand_uniform(struct random_uniform_distribution * rnd) {
    return rnd->rd(rnd->gen);
}

// 将一个值限制在最小值和最大值之间
int clamp(const int v, const int min, const int max) {
    return ((v < min) ? (min) : (v > max) ? (max) : v);
}
# 限制浮点数的取值范围在[min, max]之间
float fclamp(const float v, const float min, const float max) {
    return ((v < min) ? (min) : (v > max) ? (max) : v);
}

# 断言一维张量的形状是否符合预期
void assert_shape_1d(struct ggml_tensor * tensor, int64_t ne0) {
    GGML_ASSERT(tensor->n_dims == 1);  # 断言张量的维度是否为1
    GGML_ASSERT(tensor->ne[0] == ne0);  # 断言张量的第一个维度大小是否为ne0
}

# 断言二维张量的形状是否符合预期
void assert_shape_2d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1) {
    GGML_ASSERT(tensor->n_dims == 2);  # 断言张量的维度是否为2
    GGML_ASSERT(tensor->ne[0] == ne0);  # 断言张量的第一个维度大小是否为ne0
    GGML_ASSERT(tensor->ne[1] == ne1);  # 断言张量的第二个维度大小是否为ne1
}

# 断言三维张量的形状是否符合预期
void assert_shape_3d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1, int64_t ne2) {
    GGML_ASSERT(tensor->n_dims == 3);  # 断言张量的维度是否为3
    GGML_ASSERT(tensor->ne[0] == ne0);  # 断言张量的第一个维度大小是否为ne0
    GGML_ASSERT(tensor->ne[1] == ne1);  # 断言张量的第二个维度大小是否为ne1
    GGML_ASSERT(tensor->ne[2] == ne2);  # 断言张量的第三个维度大小是否为ne2
}
// 断言张量的维度为4
void assert_shape_4d(struct ggml_tensor * tensor, int64_t ne0, int64_t ne1, int64_t ne2, int64_t ne3) {
    GGML_ASSERT(tensor->n_dims == 4);
    // 断言张量的各维度大小符合预期
    GGML_ASSERT(tensor->ne[0] == ne0);
    GGML_ASSERT(tensor->ne[1] == ne1);
    GGML_ASSERT(tensor->ne[2] == ne2);
    GGML_ASSERT(tensor->ne[3] == ne3);
}

// 获取示例目标批次
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
```

# 定义函数，参数包括训练数据数量、是否以结束符分隔、是否以开始符分隔、是否用下一个样本填充、是否随机抽样偏移量
void generate_training_data(
    size_t                 n_train_data,  # 训练数据数量
    bool                   separate_with_eos,  # 是否以结束符分隔
    bool                   separate_with_bos,  # 是否以开始符分隔
    bool                   fill_with_next_samples,  # 是否用下一个样本填充
    bool                   sample_random_offsets  # 是否随机抽样偏移量
) {
    GGML_ASSERT(samples_count > 0);  # 断言样本数量大于0
    GGML_ASSERT(tokens_input->n_dims  == 2);  # 断言输入tokens的维度为2
    GGML_ASSERT(target_probs->n_dims  == 3);  # 断言目标概率的维度为3
    int64_t n_vocab  = target_probs->ne[0];  # 获取目标概率的第一维大小
    int64_t n_tokens = tokens_input->ne[0];  # 获取输入tokens的第一维大小
    int64_t n_batch  = tokens_input->ne[1];  # 获取输入tokens的第二维大小
    GGML_ASSERT(n_vocab  == target_probs->ne[0]);  # 断言目标概率的第一维大小与n_vocab相等
    GGML_ASSERT(n_tokens == target_probs->ne[1]);  # 断言输入tokens的第一维大小与n_tokens相等
    GGML_ASSERT(n_batch  == target_probs->ne[2]);  # 断言输入tokens的第二维大小与n_batch相等

    int64_t used_samples = 0;  # 初始化已使用样本数量为0

    ggml_set_f32(target_probs, 0.0f);  # 将目标概率初始化为0
    llama_token bos = llama_token_bos(llama_get_model(lctx));  # 获取开始符
}
    // 使用 llama_get_model(lctx) 获取模型，然后使用 llama_token_eos() 函数获取 EOS 标记
    llama_token eos = llama_token_eos(llama_get_model(lctx));
    // 打印函数名、example_id、n_batch、n_train_samples 等信息
    // printf("%s: example_id=%d n_batch=%d n_train_samples=%zu\n", __func__, example_id, n_batch, n_train_samples);
    for (int k=0; k<n_batch; ++k) {
        // 打印函数名和批次号
        // printf("%s: batch %d\n", __func__, k);
        // 计算样本索引、偏移、起始位置和大小
        size_t sample_idx   = (example_id + used_samples) % samples_count;
        size_t sample_offs  = sample_random_offsets ? samples_offs[sample_idx] : 0;
        size_t sample_begin = samples_begin[sample_idx];
        size_t sample_size  = samples_size[sample_idx];
        ++used_samples;

        // 打印函数名和样本索引
        // printf("%s: sample_idx=%zu sample=%zu\n", __func__, sample_idx, sample);
        // 断言样本起始位置+大小-1小于训练数据大小
        GGML_ASSERT(sample_begin+sample_size-1 < n_train_data);

        // 在 tokens_input 中设置第 k 个位置的值为 bos
        ggml_set_i32_nd(tokens_input, 0, k, 0, 0, bos);
        // 初始化样本分隔符的标志
        bool sample_separation_eos = !separate_with_eos;
        bool sample_separation_bos = !separate_with_bos;
        for (int64_t i=0; i<n_tokens; ++i) {
            // 初始化 token 为 eos
            llama_token token = eos;
            // 如果样本偏移大于等于样本大小并且需要填充下一个样本
            if (sample_offs >= sample_size && fill_with_next_samples) {
                // 如果不需要使用样本分隔符 eos
                if (!sample_separation_eos) {
// 插入 EOS 标记以分隔样本
sample_separation_eos = true;
// 如果没有 BOS 标记，则插入 BOS 标记以分隔样本
} else if (!sample_separation_bos) {
    sample_separation_bos = true;
    token = bos;
// 样本分隔已完成，继续下一个样本
} else {
    sample_separation_eos = !separate_with_eos;
    sample_separation_bos = !separate_with_bos;
    sample_offs  = 0;
    sample_idx   = (example_id + used_samples) % samples_count;
    sample_begin = samples_begin[sample_idx];
    sample_size  = samples_size[sample_idx];
    ++used_samples;
}
// 注意：这里没有 else-if
// 如果样本偏移小于样本大小，则获取训练数据中的标记，并将其限制在 0 到 n_vocab-1 之间
if (sample_offs < sample_size) {
    token = clamp(train_data[sample_begin+sample_offs], 0, (llama_token) (n_vocab - 1));
    // 增加样本偏移量
    ++sample_offs;
    // 设置目标概率数组中指定位置的值为+1.0f
    ggml_set_f32_nd(target_probs,  token, (int) i, (int) k, 0, +1.0f);
    // 如果 i+1<n_tokens，则将 tokens_input 中指定位置的值设为 token
    if (i+1<n_tokens) {
        ggml_set_i32_nd(tokens_input, (int) (i + 1), (int) k, 0, 0, token);
    }
}

// 返回使用的样本数
return used_samples;
}

// 设置 mt19937 随机数生成器的状态
void mt19937_set_state(std::mt19937& rng, const std::string& rng_state) {
    // 创建字符串流 s_rng_state
    std::stringstream s_rng_state;
    // 设置 s_rng_state 使用经典的本地化设置
    s_rng_state.imbue(std::locale::classic());
    // 设置 s_rng_state 抛出异常时只抛出 failbit 异常
    s_rng_state.exceptions(std::stringstream::failbit);
    // 将 rng_state 内容写入 s_rng_state
    s_rng_state.str(rng_state);
    // 从 s_rng_state 读取内容并设置为 rng 的状态
    s_rng_state >> rng;
}
// 将 mt19937 随机数生成器的状态转换为字符串
std::string mt19937_get_state(const std::mt19937& rng) {
    // 创建一个字符串流对象
    std::stringstream s_rng_state;
    // 设置字符串流的区域设置为经典区域
    s_rng_state.imbue(std::locale::classic());
    // 将随机数生成器的状态写入字符串流
    s_rng_state << rng;
    // 返回字符串流中的内容
    return s_rng_state.str();
}

// 将种子转换为 mt19937 随机数生成器的状态字符串
std::string mt19937_seed_to_state(unsigned seed) {
    // 使用给定的种子创建 mt19937 随机数生成器
    std::mt19937 rng(seed);
    // 调用 mt19937_get_state 函数获取随机数生成器的状态字符串并返回
    return mt19937_get_state(rng);
}

// 对样本进行洗牌，并返回洗牌后的状态字符串
std::string shuffle_samples(
        const std::string & rng_state,  // 随机数生成器的状态字符串
        size_t            * shuffled_offs,  // 洗牌后的偏移量
        size_t            * shuffled_begins,  // 洗牌后的开始位置
        size_t            * shuffled_sizes,  // 洗牌后的大小
        const size_t      * begins,  // 原始样本的开始位置
        const size_t      * sizes,  // 原始样本的大小
        size_t              count) {  // 样本数量
    // 如果计数为0，则直接返回随机数生成器的状态
    if (count == 0) return rng_state;

    // 创建一个名为rng的Mersenne Twister伪随机数生成器对象
    std::mt19937 rng;
    // 使用给定的状态设置Mersenne Twister伪随机数生成器对象的状态
    mt19937_set_state(rng, rng_state);

    // 为每个索引按照随机值排序索引
    std::vector<size_t> idcs;
    {
        // 创建一个存储随机数的向量rnd，并设置其大小为count
        std::vector<unsigned> rnd;
        // 设置idcs向量的大小为count
        idcs.resize(count);
        rnd.resize(count);
        // 遍历count次，为每个索引设置对应的值，并将随机数存储在rnd向量中
        for (unsigned i=0; i<count; ++i) {
            idcs[i] = i;
            rnd[i]  = rng();
        }

        // 使用lambda表达式对idcs向量进行排序，保证排序的稳定性
        std::sort(idcs.begin(), idcs.end(), [&rnd](size_t a, size_t b){
            // 如果随机数相等，则按照索引大小排序；否则按照随机数大小排序
            return (rnd[a] == rnd[b]) ? (a < b) : (rnd[a] < rnd[b]);
        });
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

    // 返回随机数生成器的状态
    return mt19937_get_state(rng);
}

// 计算两个哈希值的组合
size_t hash_combine(size_t h1, size_t h2) {
    // 计算两个哈希值的异或结果
    return h1 ^ (h2 << 1);
}

// 计算样本哈希值
size_t compute_samples_hash(const char* fn, const size_t* samples_begin, const size_t* samples_size, size_t sample_count) {
    // 创建字符串哈希函数对象
    std::hash<std::string> h_string;
    // 创建无符号长长整型哈希函数对象
    std::hash<unsigned long long> h_ull;
    // 计算文件名的哈希值
    size_t h = h_string(std::string(fn));
    // 将文件名哈希值和样本数量的哈希值组合
    h = hash_combine(h, h_ull((unsigned long long) sample_count));
    // 遍历样本，计算哈希值并组合
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
    // 如果找到子串
    if (pos != std::string::npos) {
        // 替换子串
        str.replace(pos, strlen(needle), replacement);
// 打印时间间隔，以毫秒为单位
void print_duration(double fmillis) {
    // 如果时间间隔小于1秒，以毫秒为单位打印
    if (fmillis < 1000.0f) {
        printf("%.1fms", (float) fmillis);
        return;
    }
    // 定义常量，表示1秒、1分钟、1小时和1天的毫秒数
    const int64_t one_sec  = 1000;
    const int64_t one_min  = one_sec  * 60;
    const int64_t one_hour = one_min  * 60;
    const int64_t one_day  = one_hour * 24;

    // 将浮点数时间间隔转换为整数毫秒数
    int64_t millis  = (int64_t) fmillis;
    // 计算时间间隔包含的天数、小时数、分钟数和秒数
    int64_t days    = millis/one_day;
    int64_t hours   = (millis - days*one_day)/one_hour;
    int64_t minutes = (millis - days*one_day - hours*one_hour)/one_min;
    int64_t seconds = (millis - days*one_day - hours*one_hour - minutes*one_min)/one_sec;
// 如果天数大于0，则打印天数，使用(long long int)进行类型转换
if (days > 0) {
    printf("%lldd ", (long long int) days);
}
// 打印时分秒
printf("%02lld:%02lld:%02lld", (long long int) hours, (long long int) minutes, (long long int) seconds);
}

// 计算余弦衰减
float cosine_decay(int64_t step, int64_t decay_steps, float minimum) {
    // 如果步数大于衰减步数，则将步数设置为衰减步数
    if (step > decay_steps) {
        step = decay_steps;
    }
    // 计算余弦衰减值
    const float cosine_decay = 0.50f*(1.0f + cosf(3.14159265359f*step/decay_steps));
    // 计算衰减值
    const float decay = (1 - minimum)*cosine_decay + minimum;
    return decay;
}

// 计算带重启的余弦衰减
float cosine_decay_restart(int64_t step, int64_t decay_steps, float minimum, float restart_step_mult) {
    // 当步数大于衰减步数时，进行重启
    while (step > decay_steps) {
        step -= decay_steps;
        decay_steps = (int64_t) (restart_step_mult * decay_steps);
```

    }
    return cosine_decay(step, decay_steps, minimum);
}

float learning_schedule(
    int64_t step,  // 当前训练步数
    int64_t warmup_steps,  // 热身步数
    int64_t cos_decay_steps,  // 余弦衰减步数
    float   learning_rate,  // 初始学习率
    float   overall_minimum,  // 总体最小学习率
    float   cos_decay_minimum,  // 余弦衰减最小学习率
    float   cos_decay_restart_step_mult,  // 余弦衰减重启步数倍增
    bool    enable_restart) {  // 是否启用重启

    float result =
        (step < warmup_steps)  // 如果当前步数小于热身步数
            ? (float) step / (float) warmup_steps  // 则返回当前步数除以热身步数的比例
            : enable_restart  // 否则如果启用重启
                ? cosine_decay_restart(  // 则使用余弦衰减重启函数
                    step - warmup_steps,  // 传入当前步数减去热身步数
// 定义一个函数，实现余弦衰减学习率
float cosine_decay(
    int step,
    int cos_decay_steps,
    float cos_decay_minimum,
    float cos_decay_restart_step_mult
) {
    // 使用余弦衰减函数计算学习率
    float result = cosine_decay(
        step,
        cos_decay_steps,
        cos_decay_minimum,
        cos_decay_restart_step_mult
    );
    
    // 计算最小学习率
    float min = overall_minimum / learning_rate;
    // 对结果进行调整
    result = min + result * (1.0f - min);
    // 返回调整后的结果
    return result;
}

// 检查两个张量是否具有相同的布局
static bool are_same_layout(struct ggml_tensor * a, struct ggml_tensor * b) {
    // 断言张量 a 和 b 不为空
    GGML_ASSERT(a != NULL);
    GGML_ASSERT(b != NULL);
    // 断言张量 a 和 b 的类型相同
    GGML_ASSERT(a->type == b->type);
    // 断言张量 a 和 b 的形状相同
    GGML_ASSERT(ggml_are_same_shape(a, b));
    // 断言张量 a 和 b 是连续的
    GGML_ASSERT(ggml_is_contiguous(a) && ggml_is_contiguous(b));
}
// 返回 true，表示操作成功
    return true;
}

// 根据名称复制张量数据
void copy_tensor_by_name(struct ggml_tensor * dst, struct ggml_context * ctx, const char * name) {
    // 如果目标张量为空，则直接返回
    if (dst == NULL) {
        return;
    }
    // 根据名称从上下文中获取张量
    struct ggml_tensor * t  = ggml_get_tensor(ctx, name);
    // 断言目标张量和获取的张量具有相同的布局
    GGML_ASSERT(are_same_layout(dst, t));
    // 将获取的张量数据复制到目标张量中
    memcpy(dst->data, t->data, ggml_nbytes(t));

    // 如果目标张量的名称长度为0，则将名称设置为给定的名称
    if (strlen(ggml_get_name(dst)) == 0) {
        ggml_set_name(dst, name);
    }
}

// gguf 常量
// 优化器类型的键值
static const char * LLM_KV_OPTIMIZER_TYPE = "optimizer.type";
// Adam 优化器类型的键值
static const char * LLM_KV_OPTIMIZER_TYPE_ADAM  = "adam";
// LBFGS 优化器类型的键值
static const char * LLM_KV_OPTIMIZER_TYPE_LBFGS = "lbfgs";
# 定义了一系列的常量，用于存储优化器的相关参数和数据
static const char * LLM_KV_OPTIMIZER_FILE_VERSION               = "optimizer.file_version";
static const char * LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT     = "optimizer.convergence_past_count";
static const char * LLM_KV_OPTIMIZER_PARAMETER_COUNT            = "optimizer.parameter_count";
static const char * LLM_KV_OPTIMIZER_ITERATION_COUNT            = "optimizer.iteration_count";
static const char * LLM_KV_OPTIMIZER_JUST_INITIALIZED           = "optimizer.just_initialized";
static const char * LLM_KV_OPTIMIZER_ADAM_BEST_LOSS             = "optimizer.adam.best_loss";
static const char * LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS         = "optimizer.adam.previous_loss";
static const char * LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT  = "optimizer.adam.no_improvement_count";
static const char * LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT = "optimizer.lbfgs.approx_hessian_count";
static const char * LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS            = "optimizer.lbfgs.best_loss";
static const char * LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP     = "optimizer.lbfgs.line_search_step";
static const char * LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J        = "optimizer.lbfgs.line_search_j";
static const char * LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K        = "optimizer.lbfgs.line_search_k";
static const char * LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END      = "optimizer.lbfgs.line_search_end";
static const char * LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT = "optimizer.lbfgs.no_improvement_count";

static const char * LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS    = "optimizer.adam.first_moments";
static const char * LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS   = "optimizer.adam.second_moments";
static const char * LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES = "optimizer.adam.past_loss_values";
# 定义了一系列的字符串常量，用于表示不同的张量优化器参数和训练键值对
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS  = "optimizer.lbfgs.current_parameters";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS = "optimizer.lbfgs.previous_parameters";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS   = "optimizer.lbfgs.current_gradients";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS  = "optimizer.lbfgs.previous_gradients";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION    = "optimizer.lbfgs.search_direction";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES    = "optimizer.lbfgs.past_loss_values";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA        = "optimizer.lbfgs.memory_alpha";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS           = "optimizer.lbfgs.memory_ys";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S            = "optimizer.lbfgs.memory_s";
static const char * LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y            = "optimizer.lbfgs.memory_y";

static const char * LLM_KV_TRAINING_FILE_VERSION         = "training.file_version";
static const char * LLM_KV_TRAINING_ITERATION_COUNT      = "training.iteration_count";
static const char * LLM_KV_TRAINING_SAMPLE_COUNT         = "training.sample_count";
static const char * LLM_KV_TRAINING_TOKEN_COUNT          = "training.token_count";
static const char * LLM_KV_TRAINING_EPOCH_COUNT          = "training.epoch_count";
static const char * LLM_KV_TRAINING_SHUFFLE_SAMPLES_HASH = "training.shuffle.samples_hash";
static const char * LLM_KV_TRAINING_SHUFFLE_RNG_STATE    = "training.shuffle.rng_state";
static const char * LLM_KV_TRAINING_SHUFFLE_SAMPLE_COUNT = "training.shuffle.sample_count";
static const char * LLM_KV_TRAINING_SHUFFLE_NEXT_SAMPLE  = "training.shuffle.next_sample";
// 宏定义，用于从上下文中获取指定类型的键值对，并进行相应的处理
#define GGUF_GET_KEY(ctx, dst, func, type, req, key) \
{ \
    // 将键值转换为字符串
    const std::string skey(key); \
    // 在上下文中查找键的索引
    const int kid = gguf_find_key(ctx, skey.c_str()); \
    // 如果找到了键
    if (kid >= 0) { \
        // 获取键值对应的类型
        enum gguf_type ktype = gguf_get_kv_type(ctx, kid); \
        // 如果类型不匹配
        if (ktype != (type)) { \
            // 抛出错误信息
            die_fmt("key %s has wrong type: %s", skey.c_str(), gguf_type_name(ktype)); \
        } \
        // 调用指定的函数处理键值对
        (dst) = func(ctx, kid); \
    } else if (req) { \
        // 如果需要找到键但未找到，则抛出错误信息
        die_fmt("key not found in model: %s", skey.c_str()); \
    } \
}

// 加载 gguf 上下文
void load_opt_context_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct ggml_opt_context * opt) {
    // 注意：gguf_context 必须使用 f_ggml_ctx 进行初始化，并且 no_alloc=false，否则无法读取张量数据

    // 定义变量 file_version
    uint32_t file_version;
// 从 fctx 中获取 file_version 的值，使用 gguf_get_val_u32 函数将其转换为无符号32位整数，存储在 opt->params.past 中
GGUF_GET_KEY(fctx, file_version, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_OPTIMIZER_FILE_VERSION);
// 断言 file_version 的值为 0
GGML_ASSERT(file_version == 0);

// 从 fctx 中获取 opt->params.past 的值，使用 gguf_get_val_u32 函数将其转换为无符号32位整数，存储在 opt->params.past 中
GGUF_GET_KEY(fctx, opt->params.past, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT);
// 从 fctx 中获取 opt->iter 的值，使用 gguf_get_val_u32 函数将其转换为无符号32位整数，存储在 opt->iter 中
GGUF_GET_KEY(fctx, opt->iter, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_OPTIMIZER_ITERATION_COUNT);
// 从 fctx 中获取 opt->just_initialized 的值，使用 gguf_get_val_bool 函数将其转换为布尔值，存储在 opt->just_initialized 中
GGUF_GET_KEY(fctx, opt->just_initialized, gguf_get_val_bool, GGUF_TYPE_BOOL, true, LLM_KV_OPTIMIZER_JUST_INITIALIZED);

// 从 fctx 中获取 nx 的值，使用 gguf_get_val_u64 函数将其转换为无符号64位整数，存储在 nx 中
uint64_t nx;
GGUF_GET_KEY(fctx, nx, gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_OPTIMIZER_PARAMETER_COUNT);
// 将 nx 转换为 size_t 类型，存储在 opt->nx 中
opt->nx = (size_t) nx;

// 从 fctx 中获取 opt_type 的值，使用 gguf_get_val_str 函数将其转换为字符串，存储在 opt_type 中
std::string opt_type;
GGUF_GET_KEY(fctx, opt_type, gguf_get_val_str, GGUF_TYPE_STRING, true, LLM_KV_OPTIMIZER_TYPE);
// 如果 opt_type 的值为 LLM_KV_OPTIMIZER_TYPE_ADAM，则执行以下操作
if (opt_type == LLM_KV_OPTIMIZER_TYPE_ADAM) {
    // 将 opt->params.type 设置为 GGML_OPT_ADAM
    opt->params.type = GGML_OPT_ADAM;

    // 从 fctx 中获取 opt->adam.fx_best 的值，使用 gguf_get_val_f32 函数将其转换为单精度浮点数，存储在 opt->adam.fx_best 中
    GGUF_GET_KEY(fctx, opt->adam.fx_best, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, LLM_KV_OPTIMIZER_ADAM_BEST_LOSS);
    // 从 fctx 中获取 opt->adam.fx_prev 的值，使用 gguf_get_val_f32 函数将其转换为单精度浮点数，存储在 opt->adam.fx_prev 中
    GGUF_GET_KEY(fctx, opt->adam.fx_prev, gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS);
}
# 从上下文中获取优化器参数，存储到相应的变量中
GGUF_GET_KEY(fctx, opt->adam.n_no_improvement, gguf_get_val_u32, GGUF_TYPE_UINT32,  true, LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT);

# 初始化优化器参数
ggml_opt_init(opt->ctx, opt, opt->params, opt->nx);

# 通过名称复制张量数据到优化器参数中
copy_tensor_by_name(opt->adam.m,  f_ggml_ctx, LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS);
copy_tensor_by_name(opt->adam.v,  f_ggml_ctx, LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS);
copy_tensor_by_name(opt->adam.pf, f_ggml_ctx, LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES);

# 如果优化器类型为LBFGS
if (opt_type == LLM_KV_OPTIMIZER_TYPE_LBFGS):
    # 设置参数类型为LBFGS
    opt->params.type = GGML_OPT_LBFGS;

    # 从上下文中获取LBFGS优化器参数，并存储到相应的变量中
    GGUF_GET_KEY(fctx, opt->params.lbfgs.m,         gguf_get_val_u32, GGUF_TYPE_UINT32,  true, LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT);
    GGUF_GET_KEY(fctx, opt->lbfgs.fx_best,          gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS);
    GGUF_GET_KEY(fctx, opt->lbfgs.step,             gguf_get_val_f32, GGUF_TYPE_FLOAT32, true, LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP);
    GGUF_GET_KEY(fctx, opt->lbfgs.j,                gguf_get_val_i32, GGUF_TYPE_INT32,   true, LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J);
    GGUF_GET_KEY(fctx, opt->lbfgs.k,                gguf_get_val_i32, GGUF_TYPE_INT32,   true, LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K);
    GGUF_GET_KEY(fctx, opt->lbfgs.end,              gguf_get_val_i32, GGUF_TYPE_INT32,   true, LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END);
    GGUF_GET_KEY(fctx, opt->lbfgs.n_no_improvement, gguf_get_val_u32, GGUF_TYPE_UINT32,  true, LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT);

    # 初始化LBFGS优化器参数
    ggml_opt_init(opt->ctx, opt, opt->params, opt->nx);
// 通过名称复制张量到优化器上下文中的LBFGS参数
copy_tensor_by_name(opt->lbfgs.x,    f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS);
copy_tensor_by_name(opt->lbfgs.xp,   f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS);
copy_tensor_by_name(opt->lbfgs.g,    f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS);
copy_tensor_by_name(opt->lbfgs.gp,   f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS);
copy_tensor_by_name(opt->lbfgs.d,    f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION);
copy_tensor_by_name(opt->lbfgs.pf,   f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES);
copy_tensor_by_name(opt->lbfgs.lmal, f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA);
copy_tensor_by_name(opt->lbfgs.lmys, f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS);
copy_tensor_by_name(opt->lbfgs.lms,  f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S);
copy_tensor_by_name(opt->lbfgs.lmy,  f_ggml_ctx, LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y);
// 如果优化器类型未知，则输出错误信息
} else {
    die("unknown optimizer type\n");
}

// 保存GGUF上下文中的优化器上下文
void save_opt_context_gguf(struct gguf_context * fctx, struct ggml_opt_context * opt) {
    // 设置优化器文件版本
    gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_FILE_VERSION, 0);
    // 设置优化器收敛过去计数
    gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_CONVERGENCE_PAST_COUNT, opt->params.past);
    // 设置优化器参数数量
    gguf_set_val_u64(fctx, LLM_KV_OPTIMIZER_PARAMETER_COUNT, (uint64_t) opt->nx);
    // 设置优化器迭代次数
    gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_ITERATION_COUNT, opt->iter);
# 设置布尔类型的参数值
gguf_set_val_bool(fctx, LLM_KV_OPTIMIZER_JUST_INITIALIZED, opt->just_initialized);

# 根据参数类型进行不同的操作
switch (opt->params.type) {
    # 如果参数类型为 GGML_OPT_ADAM
    case GGML_OPT_ADAM:
        # 设置字符串类型的参数值
        gguf_set_val_str(fctx, LLM_KV_OPTIMIZER_TYPE, LLM_KV_OPTIMIZER_TYPE_ADAM);
        # 设置浮点数类型的参数值
        gguf_set_val_f32(fctx, LLM_KV_OPTIMIZER_ADAM_BEST_LOSS, opt->adam.fx_best);
        gguf_set_val_f32(fctx, LLM_KV_OPTIMIZER_ADAM_PREVIOUS_LOSS, opt->adam.fx_prev);
        gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_ADAM_NO_IMPROVEMENT_COUNT, opt->adam.n_no_improvement);

        # 设置张量的名称
        ggml_set_name(opt->adam.m, LLM_TENSOR_OPTIMIZER_ADAM_FIRST_MOMENTS);
        ggml_set_name(opt->adam.v, LLM_TENSOR_OPTIMIZER_ADAM_SECOND_MOMENTS);
        if (opt->adam.pf) {
            ggml_set_name(opt->adam.pf, LLM_TENSOR_OPTIMIZER_ADAM_PAST_LOSS_VALUES);
        }

        # 添加张量到上下文中
        gguf_add_tensor(fctx, opt->adam.m);
        gguf_add_tensor(fctx, opt->adam.v);
        if (opt->adam.pf) {
            gguf_add_tensor(fctx, opt->adam.pf);
        // 根据不同的优化器类型设置相应的参数
        case GGML_OPT_LBFGS:
            {
                // 设置优化器类型为LBFGS
                gguf_set_val_str(fctx, LLM_KV_OPTIMIZER_TYPE, LLM_KV_OPTIMIZER_TYPE_LBFGS);
                // 设置LBFGS的参数：近似Hessian矩阵的数量
                gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_LBFGS_APPROX_HESSIAN_COUNT, opt->params.lbfgs.m);
                // 设置LBFGS的参数：最佳损失值
                gguf_set_val_f32(fctx, LLM_KV_OPTIMIZER_LBFGS_BEST_LOSS,            opt->lbfgs.fx_best);
                // 设置LBFGS的参数：线搜索步长
                gguf_set_val_f32(fctx, LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_STEP,     opt->lbfgs.step);
                // 设置LBFGS的参数：线搜索迭代次数j
                gguf_set_val_i32(fctx, LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_J,        opt->lbfgs.j);
                // 设置LBFGS的参数：线搜索迭代次数k
                gguf_set_val_i32(fctx, LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_K,        opt->lbfgs.k);
                // 设置LBFGS的参数：线搜索结束标志
                gguf_set_val_i32(fctx, LLM_KV_OPTIMIZER_LBFGS_LINE_SEARCH_END,      opt->lbfgs.end);
                // 设置LBFGS的参数：无改善的次数
                gguf_set_val_u32(fctx, LLM_KV_OPTIMIZER_LBFGS_NO_IMPROVEMENT_COUNT, opt->lbfgs.n_no_improvement);

                // 设置LBFGS的参数：当前参数
                ggml_set_name(opt->lbfgs.x,    LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_PARAMETERS);
                // 设置LBFGS的参数：上一次的参数
                ggml_set_name(opt->lbfgs.xp,   LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_PARAMETERS);
                // 设置LBFGS的参数：当前梯度
                ggml_set_name(opt->lbfgs.g,    LLM_TENSOR_OPTIMIZER_LBFGS_CURRENT_GRADIENTS);
                // 设置LBFGS的参数：上一次的梯度
                ggml_set_name(opt->lbfgs.gp,   LLM_TENSOR_OPTIMIZER_LBFGS_PREVIOUS_GRADIENTS);
                // 设置LBFGS的参数：搜索方向
                ggml_set_name(opt->lbfgs.d,    LLM_TENSOR_OPTIMIZER_LBFGS_SEARCH_DIRECTION);
                // 如果有过去的损失值，设置LBFGS的参数：过去的损失值
                if (opt->lbfgs.pf) {
                    ggml_set_name(opt->lbfgs.pf, LLM_TENSOR_OPTIMIZER_LBFGS_PAST_LOSS_VALUES);
# 设置 lbfgs 对象的内存参数名称
ggml_set_name(opt->lbfgs.lmal, LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_ALPHA);
ggml_set_name(opt->lbfgs.lmys, LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_YS);
ggml_set_name(opt->lbfgs.lms,  LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_S);
ggml_set_name(opt->lbfgs.lmy,  LLM_TENSOR_OPTIMIZER_LBFGS_MEMORY_Y);

# 将 lbfgs 对象中的张量添加到上下文中
gguf_add_tensor(fctx, opt->lbfgs.x);
gguf_add_tensor(fctx, opt->lbfgs.xp);
gguf_add_tensor(fctx, opt->lbfgs.g);
gguf_add_tensor(fctx, opt->lbfgs.gp);
gguf_add_tensor(fctx, opt->lbfgs.d);
if (opt->lbfgs.pf) {
    gguf_add_tensor(fctx, opt->lbfgs.pf);
}
gguf_add_tensor(fctx, opt->lbfgs.lmal);
gguf_add_tensor(fctx, opt->lbfgs.lmys);
gguf_add_tensor(fctx, opt->lbfgs.lms);
gguf_add_tensor(fctx, opt->lbfgs.lmy);
// 加载训练状态的函数，接受 gguf_context 结构体指针、ggml_context 结构体指针和 train_state 结构体指针作为参数
bool load_train_state_gguf(struct gguf_context * fctx, struct ggml_context * f_ggml_ctx, struct train_state * train) {
    // 如果在 fctx 中找不到 LLM_KV_TRAINING_FILE_VERSION 键，则返回 false
    if (gguf_find_key(fctx, LLM_KV_TRAINING_FILE_VERSION) < 0) {
        return false;
    }

    // 声明并初始化 file_version 变量，获取 LLM_KV_TRAINING_FILE_VERSION 键对应的值
    uint32_t file_version;
    GGUF_GET_KEY(fctx, file_version,         gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_FILE_VERSION);
    // 断言 file_version 的值小于等于 1
    GGML_ASSERT(file_version <= 1);

    // 如果 file_version 的值为 0，则执行以下代码块
    if (file_version == 0) {
        // 获取训练迭代次数、训练样本数和训练标记数，并存储到 train 结构体中
        GGUF_GET_KEY(fctx, train->train_its,     gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_ITERATION_COUNT);
        GGUF_GET_KEY(fctx, train->train_samples, gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_SAMPLE_COUNT);
        GGUF_GET_KEY(fctx, train->train_tokens,  gguf_get_val_u32, GGUF_TYPE_UINT32, true, LLM_KV_TRAINING_TOKEN_COUNT);

    // 如果 file_version 的值为 1，则执行以下代码块
    } else if (file_version == 1) {
        // 获取训练迭代次数，并存储到 train 结构体中
        GGUF_GET_KEY(fctx, train->train_its,     gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_TRAINING_ITERATION_COUNT);
# 从上下文中获取训练样本数，并存储到上下文中
GGUF_GET_KEY(fctx, train->train_samples, gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_TRAINING_SAMPLE_COUNT);
# 从上下文中获取训练标记数，并存储到上下文中
GGUF_GET_KEY(fctx, train->train_tokens,  gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_TRAINING_TOKEN_COUNT);
# 从上下文中获取训练周期数，并存储到上下文中
GGUF_GET_KEY(fctx, train->train_epochs,  gguf_get_val_u64, GGUF_TYPE_UINT64, true, LLM_KV_TRAINING_EPOCH_COUNT);

# 从上下文中获取训练样本洗牌哈希值，并存储到上下文中
GGUF_GET_KEY(fctx, train->shuffle_samples_hash,      gguf_get_val_u64, GGUF_TYPE_UINT64, false, LLM_KV_TRAINING_SHUFFLE_SAMPLES_HASH);
# 从上下文中获取当前训练样本洗牌随机数生成器状态，并存储到上下文中
GGUF_GET_KEY(fctx, train->shuffle_rng_state_current, gguf_get_val_str, GGUF_TYPE_STRING, false, LLM_KV_TRAINING_SHUFFLE_RNG_STATE);
# 从上下文中获取训练样本洗牌数量，并存储到上下文中
GGUF_GET_KEY(fctx, train->shuffle_sample_count,      gguf_get_val_u64, GGUF_TYPE_UINT64, false, LLM_KV_TRAINING_SHUFFLE_SAMPLE_COUNT);
# 从上下文中获取下一个训练样本洗牌索引，并存储到上下文中
GGUF_GET_KEY(fctx, train->shuffle_next_sample,       gguf_get_val_u64, GGUF_TYPE_UINT64, false, LLM_KV_TRAINING_SHUFFLE_NEXT_SAMPLE);

# 从上下文中加载优化器上下文
load_opt_context_gguf(fctx, f_ggml_ctx, train->opt);
# 返回 true
return true;
}

# 存储训练状态到上下文中
void save_train_state_gguf(struct gguf_context * fctx, struct train_state * train) {
    # 存储训练文件版本号到上下文中
    gguf_set_val_u32(fctx, LLM_KV_TRAINING_FILE_VERSION,    1);
    # 存储训练迭代次数到上下文中
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_ITERATION_COUNT, train->train_its);
    # 存储训练样本数到上下文中
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_SAMPLE_COUNT,    train->train_samples);
    # 存储训练标记数到上下文中
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_TOKEN_COUNT,     train->train_tokens);
    # 存储训练周期数到上下文中
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_EPOCH_COUNT,     train->train_epochs);
    // 设置训练数据的随机采样哈希值
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_SHUFFLE_SAMPLES_HASH, (uint64_t) train->shuffle_samples_hash);
    // 设置训练数据的随机数生成器状态
    gguf_set_val_str(fctx, LLM_KV_TRAINING_SHUFFLE_RNG_STATE,    train->shuffle_rng_state_current.c_str());
    // 设置训练数据的随机采样数量
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_SHUFFLE_SAMPLE_COUNT, (uint64_t) train->shuffle_sample_count);
    // 设置训练数据的下一个随机采样
    gguf_set_val_u64(fctx, LLM_KV_TRAINING_SHUFFLE_NEXT_SAMPLE,  (uint64_t) train->shuffle_next_sample);

    // 保存优化上下文到 GGUF
    save_opt_context_gguf(fctx, train->opt);
}


struct llama_file {
    // 使用 FILE * 以便不必重新打开文件进行内存映射
    FILE * fp;
    // 文件大小
    size_t size;

    // 构造函数，打开文件并初始化文件大小
    llama_file(const char * fname, const char * mode) {
        // 打开文件
        fp = std::fopen(fname, mode);
        // 如果文件打开失败，设置文件大小为 0
        if (fp == NULL) {
            size = 0;
        } else {
    // 将文件指针移动到文件末尾
    seek(0, SEEK_END);
    // 获取文件指针当前位置，即文件大小
    size = tell();
    // 将文件指针移动到文件开头
    seek(0, SEEK_SET);
}

// 返回文件指针当前位置
size_t tell() const {
    // 根据操作系统不同，使用不同的函数获取文件指针当前位置
#ifdef _WIN32
    __int64 ret = _ftelli64(fp);
#else
    long ret = std::ftell(fp);
#endif
    // 断言文件指针当前位置不为-1，即获取位置操作不应该失败
    GGML_ASSERT(ret != -1); // this really shouldn't fail
    return (size_t) ret;
}

// 将文件指针移动到指定位置
void seek(size_t offset, int whence) {
    // 根据操作系统不同，使用不同的函数移动文件指针
#ifdef _WIN32
    int ret = _fseeki64(fp, (__int64) offset, whence);
#else
        // 将文件指针移动到指定位置
        int ret = std::fseek(fp, (long) offset, whence);
#endif
        // 断言文件指针移动操作是否成功
        GGML_ASSERT(ret == 0); // 相同
    }

    // 从文件中读取原始数据
    void read_raw(void * ptr, size_t size) {
        // 如果要读取的数据大小为0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 从文件中读取指定大小的数据到指定位置
        std::size_t ret = std::fread(ptr, size, 1, fp);
        // 检查是否发生了读取错误
        if (ferror(fp)) {
            // 如果有错误，输出错误信息并终止程序
            die_fmt("read error: %s", strerror(errno));
        }
        // 检查是否成功读取了指定大小的数据
        if (ret != 1) {
            // 如果没有成功读取，输出错误信息并终止程序
            die("unexpectedly reached end of file");
        }
    }

    // 从文件中读取32位无符号整数
    std::uint32_t read_u32() {
        // 读取一个无符号32位整数
        std::uint32_t ret;
        read_raw(&ret, sizeof(ret));
        return ret;
    }

    // 读取指定长度的字符串
    std::string read_string(std::uint32_t len) {
        // 创建一个字符向量，用于存储读取的字符
        std::vector<char> chars(len);
        // 读取指定长度的字符到字符向量中
        read_raw(chars.data(), len);
        // 将字符向量转换为字符串并返回
        return std::string(chars.data(), len);
    }

    // 写入指定大小的原始数据
    void write_raw(const void * ptr, size_t size) {
        // 如果大小为0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 将数据写入文件
        size_t ret = std::fwrite(ptr, size, 1, fp);
        // 如果写入不成功，则输出错误信息并终止程序
        if (ret != 1) {
            die_fmt("write error: %s", strerror(errno));
        }
    }

    // 写入一个 32 位无符号整数
    void write_u32(std::uint32_t val) {
        // 调用 write_raw 函数写入指定长度的数据
        write_raw(&val, sizeof(val));
    }

    // 析构函数，用于释放资源
    ~llama_file() {
        // 如果文件指针存在，则关闭文件
        if (fp) {
            std::fclose(fp);
        }
    }
};

// 计算 UTF-8 编码的字符长度
static size_t utf8_len(char src) {
    // UTF-8 编码的字符长度查找表
    const size_t lookup[] = { 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 3, 4 };
    // 获取字符的高位字节
    uint8_t highbits = static_cast<uint8_t>(src) >> 4;
    // 返回对应的 UTF-8 编码长度
    return lookup[highbits];
}

// 为每个字节标记其 UTF-8 单元编号
// 返回 utf8 字符的数量。
// 例如，当 bytes == '\x61\xD0\xB0\x62' 时，
// utf8_units 将变为 [0,0,1,0]，
// utf8_nunits 将变为 [1,2,2,1]，并返回 3。
// utf8_units 为零的字节是 utf8 字符的起始位置。
static size_t mark_utf8_units(const char* bytes, int * utf8_units, int * utf8_nunits, size_t count) {
    size_t offs = 0; // 偏移量
    size_t count_utf8 = 0; // utf8 字符的数量
    while(offs < count) { // 循环直到偏移量小于 count
        int len = (int) utf8_len(bytes[offs]); // 获取当前字节的 utf8 长度
        for (int i=0; i<len; ++i) { // 遍历当前 utf8 字符的每个字节
            utf8_units[offs+i]  = i; // 设置 utf8_units 数组中的值
            utf8_nunits[offs+i] = len; // 设置 utf8_nunits 数组中的值
        }
        offs += len; // 更新偏移量
        ++count_utf8; // 增加 utf8 字符数量
    }
    return count_utf8; // 返回 utf8 字符数量
}
# 从文件中分词，生成标记和样本的起始位置和大小
size_t tokenize_file(
        struct llama_context     * lctx,  # 指向 llama_context 结构体的指针
        const char               * filename,  # 文件名
        const std::string        & sample_start,  # 样本起始位置
        bool                       include_sample_start,  # 是否包含样本起始位置
        bool                       overlapping_samples,  # 是否允许重叠样本
        unsigned                   context_length,  # 上下文长度
        std::vector<llama_token> & out_tokens,  # 输出的标记
        std::vector<size_t>      & out_samples_begin,  # 输出的样本起始位置
        std::vector<size_t>      & out_samples_size) {  # 输出的样本大小
    # 打开文件
    struct llama_file f(filename, "rb");
    # 如果文件大小为0
    if (f.size == 0) {
        # 清空输出的标记和样本信息
        out_tokens.clear();
        out_samples_begin.clear();
        out_samples_size.clear();
        # 打印警告信息
        printf("%s: warning: empty or not existing training data file '%s'\n",
            __func__, filename);
        # 返回输出标记的大小
        return out_tokens.size();
    }
    // 考虑到分词器可能会添加的前导空白，例如'\t'会被llama spm分词器标记为[29871, 12]
    const int n_max_tokens_overhead = 1; // 定义最大标记开销

    std::vector<char> buf; // 创建一个字符向量buf
    buf.resize(f.size); // 调整buf的大小为f.size

    f.read_raw(buf.data(), f.size); // 从文件f中读取原始数据到buf中

    std::vector<int> utf8_units; // 创建一个整数向量utf8_units
    std::vector<int> utf8_nunits; // 创建一个整数向量utf8_nunits
    utf8_units.resize(buf.size()); // 调整utf8_units的大小为buf的大小
    utf8_nunits.resize(buf.size()); // 调整utf8_nunits的大小为buf的大小
    mark_utf8_units(buf.data(), utf8_units.data(), utf8_nunits.data(), buf.size()); // 标记UTF-8单位

    if (sample_start.size() == 0) { // 如果sample_start的大小为0
        // 一次性对所有数据进行标记
        out_tokens.resize(buf.size() + n_max_tokens_overhead); // 调整out_tokens的大小为buf.size() + n_max_tokens_overhead
# 调用 llama_tokenize 函数对输入的数据进行分词处理，将结果存储在 out_tokens 中
int n_tokens = llama_tokenize(
    llama_get_model(lctx),  # 获取模型
    buf.data(),  # 输入数据的指针
    (int) buf.size(),  # 输入数据的大小
    out_tokens.data(),  # 输出 tokens 的指针
    (int) out_tokens.size(),  # 输出 tokens 的大小
    false, false  # 其他参数
);
# 如果分词处理返回的结果小于 0，说明出现了错误，重新调整 out_tokens 的大小并再次进行分词处理
if (n_tokens < 0) {
    out_tokens.resize(-n_tokens);  # 调整 out_tokens 的大小
    n_tokens = llama_tokenize(
        llama_get_model(lctx),  # 获取模型
        buf.data(),  # 输入数据的指针
        (int) buf.size(),  # 输入数据的大小
        out_tokens.data(),  # 输出 tokens 的指针
        (int) out_tokens.size(),  # 输出 tokens 的大小
        false, false  # 其他参数
    );
}
# 如果分词处理返回的结果大于等于 0，说明处理成功，调整 out_tokens 的大小为 n_tokens
if (n_tokens >= 0) {
    out_tokens.resize(n_tokens);  # 调整 out_tokens 的大小
}
        // 清空开始样本的位置列表
        out_samples_begin.clear();
        // 将开始样本的位置设为0
        out_samples_begin.push_back(0);
        // 将开始样本的大小设为上下文长度和输出标记数的最小值
        out_samples_size.push_back(std::min((size_t) context_length, out_tokens.size()));
        // 计算结束位置
        size_t end = (out_tokens.size() >= context_length) ? (out_tokens.size() - context_length) : 0;
        // 遍历生成所有可能的样本开始位置
        for (size_t sample_begin = 1; sample_begin < end; ++sample_begin) {
            out_samples_begin.push_back(sample_begin);
            out_samples_size.push_back(context_length);
        }
    } else {
        // 将数据转换为字符串
        std::string data_str(buf.data(), buf.size());
        // 清空开始样本的位置列表
        out_samples_begin.clear();
        // 清空开始样本的大小列表
        out_samples_size.clear();
        // 清空输出标记列表
        out_tokens.clear();

        // 寻找所有样本开始模式的位置
        size_t sample_begin = data_str.find(sample_start, 0);
        while (sample_begin != std::string::npos) {
// 将sample_begin添加到out_samples_begin的末尾
out_samples_begin.push_back(sample_begin);
// 设置搜索起点为sample_start的末尾
const size_t search_start = sample_begin + sample_start.size();
// 在data_str中从search_start位置开始搜索sample_start，更新sample_begin的值
sample_begin = data_str.find(sample_start, search_start);
// 如果out_samples_begin的大小为0，说明未找到sample_start，输出警告信息并将0添加到out_samples_begin中
if (out_samples_begin.size() == 0) {
    printf("%s: warning: sample start pattern '%s' not found. inserting single sample at data begin\n",
        __func__, sample_start.c_str());
    out_samples_begin.push_back(0);
}

// 将out_samples_size的大小设置为out_samples_begin的大小，初始值为0
out_samples_size.resize(out_samples_begin.size(), 0);

// 创建一个存储字符的向量buf_sample和存储llama_token的向量tok_sample
std::vector<char>        buf_sample;
std::vector<llama_token> tok_sample;

// 设置sample_begin_offset的值，如果include_sample_start为true，则为0，否则为sample_start的大小
const size_t sample_begin_offset = (include_sample_start ? 0 : sample_start.size());
// 初始化found_too_big_sample、found_too_small_sample、found_empty_sample和found_min_sample_size的值
size_t found_too_big_sample   = 0;
size_t found_too_small_sample = 0;
size_t found_empty_sample     = 0;
size_t found_min_sample_size  = SIZE_MAX;
        // 初始化找到的最大样本大小为0
        size_t found_max_sample_size  = 0;

        // 初始化最大标记文本大小为0，获取词汇表大小
        size_t max_token_text_size = 0;
        int n_vocab = llama_n_vocab(llama_get_model(lctx));
        // 遍历词汇表，获取最大标记文本大小
        for (llama_token token=0; token < n_vocab; ++token) {
            max_token_text_size = std::max(
                max_token_text_size,
                strlen(llama_token_get_text(llama_get_model(lctx), token)));
        }

        // 上下文字节长度的上限
        // 具有此字节长度的字符串应始终标记为至少context_length个标记
        size_t context_byte_len = max_token_text_size*context_length;

        // 遍历输出样本的起始位置
        for (unsigned i=0; i<out_samples_begin.size(); ++i) {
            // 从模式位置确定样本的起始和结束
            size_t sample_begin = out_samples_begin[i] + sample_begin_offset;
            size_t sample_end   = overlapping_samples
                                    ? std::min(
                                        data_str.size(),
            // 如果样本结束位置在 utf8 单元的范围内，并且 utf8 单元大于 0
            if (sample_end < utf8_units.size() && utf8_units[sample_end] > 0) {
                // 样本结束位置在 utf8 字符的中间。
                // 将样本结束位置移动到下一个 utf8 字符的开始位置。
                sample_end += utf8_nunits[sample_end] - utf8_units[sample_end];
            }
            // 计算样本大小
            size_t sample_size = sample_end - sample_begin;
            // 如果样本大小为 0
            if (sample_size == 0) {
                // 增加空样本计数
                ++found_empty_sample;
            }

            // 如果样本大小大于 0
            if (sample_size > 0) {
                // llama_tokenize 需要以零结尾的字符串，
                // 将样本复制到缓冲区并以零结尾。
                buf_sample.resize(sample_size);
                memcpy(buf_sample.data(), data_str.data() + sample_begin, sample_size);
                // 打印样本数据
                // printf("sample: '%s'\n", buf_sample.data());

                // 对样本进行标记化处理
                tok_sample.resize(buf_sample.size() + n_max_tokens_overhead);
                // 调用llama_tokenize函数对样本进行标记化处理，返回标记数量
                int n_tokens = llama_tokenize(llama_get_model(lctx),
                    buf_sample.data(),
                    (int) buf_sample.size(),
                    tok_sample.data(),
                    (int) tok_sample.size(),
                    false, false);
                // 如果标记数量小于0，重新调整tok_sample大小，并再次调用llama_tokenize函数
                if (n_tokens < 0) {
                    tok_sample.resize(-n_tokens);
                    n_tokens = llama_tokenize(llama_get_model(lctx),
                        buf_sample.data(),
                        (int) buf_sample.size(),
                        tok_sample.data(),
                        (int) tok_sample.size(),
                        false, false);
                    // 断言标记数量大于等于0
                    GGML_ASSERT(n_tokens >= 0);
                }
// 确保 n_tokens 不超过 tok_sample 的大小
GGML_ASSERT(n_tokens <= (int) tok_sample.size());

// 如果 n_tokens 大于上下文长度，则增加找到过大样本的计数
if ((size_t) n_tokens > context_length) {
    ++found_too_big_sample;
} 
// 如果 n_tokens 小于上下文长度，则增加找到过小样本的计数
else if ((size_t) n_tokens < context_length) {
    ++found_too_small_sample;
}
// 更新找到的最大样本大小
found_max_sample_size = std::max(found_max_sample_size, (size_t) n_tokens);
// 更新找到的最小样本大小
found_min_sample_size = std::min(found_min_sample_size, (size_t) n_tokens);

// 写出 tokens，样本的起始位置和大小
// 用 token 的起始位置覆盖字符串的起始位置
out_samples_begin[i] = out_tokens.size();
out_samples_size[i] = (size_t) n_tokens;
out_tokens.insert(out_tokens.end(), tok_sample.begin(), tok_sample.begin() + n_tokens);
// 如果没有找到 token，则样本的起始位置为当前 tokens 的大小，大小为 0
} else {
    out_samples_begin[i] = out_tokens.size();
    out_samples_size[i] = 0;
}
    }
    // 如果发现超过上下文长度的样本数量大于0，则打印警告信息
    if (found_too_big_sample > 0) {
        printf("%s: warning: found %zu samples (max length %zu) that exceed context length of %u. samples will be cut off.\n",
            __func__, found_too_big_sample, found_max_sample_size, context_length);
    }

    // 如果发现小于上下文长度的样本数量大于0，则打印警告信息
    if (found_too_small_sample > 0) {
        printf("%s: warning: found %zu samples (min length %zu) that are shorter than context length of %u.\n",
            __func__, found_too_small_sample, found_min_sample_size, context_length);
    }

    // 如果发现空样本，则打印警告信息
    if (found_empty_sample) {
        printf("%s: warning: found %zu empty samples.\n",
            __func__, found_empty_sample);
    }
}
// 打印总样本数量
printf("%s: total number of samples: %zu\n",
    __func__, out_samples_begin.size());

// 断言输出样本的开始和大小的数量相等
GGML_ASSERT(out_samples_begin.size() == out_samples_size.size());
// 返回 out_tokens 的大小
return out_tokens.size();
}

// 根据文件名、模式、最新值和迭代次数获取训练文件名
std::string get_train_filename(const char * filename, const char * pattern_it, const char * latest, int64_t iteration) {
    // 如果迭代次数大于等于0，使用迭代次数转换为字符串，否则使用最新值
    std::string sit = (iteration >= 0) ? std::to_string(iteration) : std::string(latest);
    // 替换文件名中的模式为实际值
    return replace_str(filename, pattern_it, sit.c_str());
}

// 获取默认的训练参数
struct train_params_common get_default_train_params_common() {
    struct train_params_common params;
    // 设置默认的训练数据文件名、检查点输入文件名、检查点输出文件名、模式、最新值
    params.fn_train_data     = "shakespeare.txt";
    params.fn_checkpoint_in  = "checkpoint.gguf";
    params.fn_checkpoint_out = "checkpoint-ITERATION.gguf";
    params.pattern_fn_it     = "ITERATION";
    params.fn_latest         = "LATEST";

    // 设置是否打印用法信息
    params.print_usage = false;

    // 设置每隔多少次保存一次检查点
    params.save_every = 10;
# 设置种子为-1
params.seed = -1;

# 设置上下文大小为128
params.n_ctx = 128;
# 设置线程数为6
params.n_threads = 6;
# 设置批次大小为8
params.n_batch = 8;
# 设置梯度累积次数为1
params.n_gradient_accumulation = 1;
# 设置训练轮数为-1
params.n_epochs = -1;
# 设置GPU层数为0
params.n_gpu_layers = 0;

# 禁用自定义上下文大小
params.custom_n_ctx = false;

# 启用flash
params.use_flash = true;
# 启用检查点
params.use_checkpointing = true;

# 设置样本起始位置为空字符串
params.sample_start = "";
# 禁用包含样本起始位置
params.include_sample_start = false;
# 禁用转义
params.escape = false;
# 禁用重叠样本
params.overlapping_samples = false;
# 禁用使用下一个样本填充
params.fill_with_next_samples = false;
    # 设置是否在每个序列之间添加结束符号
    params.separate_with_eos      = false;
    # 设置是否在每个序列之间添加开始符号
    params.separate_with_bos      = true;
    # 设置是否在训练时随机采样偏移量
    params.sample_random_offsets  = false;
    # 设置是否强制重新洗牌
    params.force_reshuffle        = false;

    # 设置优化器的过去梯度的权重
    params.opt_past               = 0;
    # 设置优化器的梯度变化的阈值
    params.opt_delta              = 1e-5f;
    # 设置优化器的最大不改善次数
    params.opt_max_no_improvement = 0;

    # 设置训练的预热步数
    params.warmup            =  100;
    # 设置余弦退火的步数
    params.cos_decay_steps   = 1000;
    # 设置余弦退火的重启倍数
    params.cos_decay_restart = 1.1f;
    # 设置余弦退火的最小值
    params.cos_decay_min     = 0.1f;
    # 设置是否启用余弦退火的重启
    params.enable_restart    = false;

    # 设置Adam优化器的迭代次数
    params.adam_n_iter         = 256;
    # 设置Adam优化器的学习率
    params.adam_alpha          = 1e-3f;
    # 设置Adam优化器的最小学习率
    params.adam_min_alpha      = 0;
    # 设置Adam优化器的学习率衰减
    params.adam_decay          = 1e-1f;
    # 设置Adam优化器的最小维度
    params.adam_decay_min_ndim = 2;
    // 设置参数 adam_beta1 的值为 0.9
    params.adam_beta1          = 0.9f;
    // 设置参数 adam_beta2 的值为 0.999
    params.adam_beta2          = 0.999f;
    // 设置参数 adam_gclip 的值为 1.0
    params.adam_gclip          = 1.0f;
    // 设置参数 adam_eps_f 的值为 0.0
    params.adam_eps_f          = 0.0f;

    // 返回参数结构体
    return params;
}

// 打印通用训练用法
void print_common_train_usage(int /*argc*/, char ** /*argv*/, const struct train_params_common * params) {
    // 打印使用方法
    // fprintf(stderr, "usage: %s [options]\n", argv[0]);
    // fprintf(stderr, "\n");
    // fprintf(stderr, "options:\n");
    // fprintf(stderr, "  -h, --help                 show this help message and exit\n");
    // 打印训练数据路径
    fprintf(stderr, "  --train-data FNAME         path from which to load training data (default '%s')\n", params->fn_train_data);
    // 打印训练检查点输入路径
    fprintf(stderr, "  --checkpoint-in FNAME      path from which to load training checkpoint (default '%s')\n", params->fn_checkpoint_in);
    // 打印训练检查点输出路径
    fprintf(stderr, "  --checkpoint-out FNAME     path to save training checkpoint (default '%s')\n", params->fn_checkpoint_out);
    // 打印输出文件名中迭代号的替换模式
    fprintf(stderr, "  --pattern-fn-it STR        pattern in output filenames to be replaced by iteration number (default '%s')\n", params->pattern_fn_it);
    // 打印保存最新输出时使用的字符串
    fprintf(stderr, "  --fn-latest STR            string to use instead of iteration number for saving latest output (default '%s')\n", params->fn_latest);
    // 打印每N次迭代保存检查点和输出文件
    fprintf(stderr, "  --save-every N             save checkpoint and lora every N iterations. Disabled when N <= 0. (default '%d')\n", params->save_every);
    // 打印随机数种子
    fprintf(stderr, "  -s SEED, --seed SEED       RNG seed (default: -1, use random seed for -1)\n");
    # 打印参数说明和默认值到标准错误输出
    fprintf(stderr, "  -c N, --ctx N              Context size used during training (default %d)\n", params->n_ctx);
    fprintf(stderr, "  -t N, --threads N          Number of threads (default %d)\n", params->n_threads);
    fprintf(stderr, "  -b N, --batch N            Parallel batch size (default %d)\n", params->n_batch);
    fprintf(stderr, "  --grad-acc N               Number of gradient accumulation steps (simulates larger batch size of batch*gradacc) (default %d)\n", params->n_gradient_accumulation);
    fprintf(stderr, "  --include-sample-start     Include the sample start in the samples. (default off)\n");
    fprintf(stderr, "  --escape                   process sample start escapes sequences (\\n, \\r, \\t, \\', \\\", \\\\)\n");
    fprintf(stderr, "  --overlapping-samples      Samples my overlap, will include sample-start of second and following samples. When off, samples will end at begin of next sample. (default off)\n");
    fprintf(stderr, "  --fill-with-next-samples   Samples shorter than context length will be followed by the next (shuffled) samples. (default off)\n");
    fprintf(stderr, "  --separate-with-eos        When fill-with-next-samples, insert end-of-sequence token between samples.%s\n", params->separate_with_eos ? " (default)" : "");
    fprintf(stderr, "  --separate-with-bos        When fill-with-next-samples, insert begin-of-sequence token between samples.%s\n", params->separate_with_bos ? " (default)" : "");
    fprintf(stderr, "  --no-separate-with-eos     When fill-with-next-samples, don't insert end-of-sequence token between samples.%s\n", !params->separate_with_eos ? " (default)" : "");
    fprintf(stderr, "  --no-separate-with-bos     When fill-with-next-samples, don't insert begin-of-sequence token between samples.%s\n", !params->separate_with_bos ? " (default)" : "");
    fprintf(stderr, "  --force-reshuffle          Force a reshuffling of data at program start, otherwise the shuffling of loaded checkpoint is resumed.\n");
    fprintf(stderr, "  --no-flash                 Don't use flash attention \n");
    fprintf(stderr, "  --use-flash                Use flash attention (default)\n");
    fprintf(stderr, "  --no-checkpointing         Don't use gradient checkpointing\n");
    fprintf(stderr, "  --use-checkpointing        Use gradient checkpointing (default)\n");
    fprintf(stderr, "  --warmup N                 Only for Adam optimizer. Number of warmup steps (default %d)\n", params->warmup);
    fprintf(stderr, "  --cos-decay-steps N        Only for Adam optimizer. Number of cosine decay steps (default %d)\n", params->cos_decay_steps);
    fprintf(stderr, "  --cos-decay-restart N      Only for Adam optimizer. Increase of cosine decay steps after restart (default %f)\n", params->cos_decay_restart);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --cos-decay-min N          Only for Adam optimizer. Cosine decay minimum (default %f)\n", params->cos_decay_min);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --enable-restart N         Only for Adam optimizer. Enable restarts of cos-decay %s\n", params->enable_restart ? "(default)" : "");
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --disable-restart N        Only for Adam optimizer. Disable restarts of cos-decay %s\n", !params->enable_restart ? "(default)" : "");
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --opt-past N               Number of optimization iterations to track for delta convergence test. Disabled when zero. (default %d)\n", params->opt_past);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --opt-delta N              Maximum delta for delta convergence test. Disabled when <= zero. (default %f)\n", params->opt_delta);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --opt-max-no-improvement N Maximum number of optimization iterations with no improvement. Disabled when <= zero. (default %d)\n", params->opt_max_no_improvement);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --epochs N                 Maximum number epochs to process. (default %d)\n", params->n_epochs);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --adam-iter N              Maximum number of Adam optimization iterations for each batch (default %d)\n", params->adam_n_iter);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --adam-alpha N             Adam learning rate alpha (default %f)\n", params->adam_alpha);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --adam-min-alpha N         Adam minimum learning rate alpha - including warmup phase (default %f)\n", params->adam_min_alpha);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --adam-decay N             AdamW weight decay. Values greater zero enable AdamW instead of regular Adam. (default %f)\n", params->adam_decay);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --adam-beta1 N             AdamW beta1 in interval [0,1). How much to smooth the first moment of gradients. (default %f)\n", params->adam_beta1);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --adam-beta2 N             AdamW beta2 in interval [0,1). How much to smooth the second moment of gradients. (default %f)\n", params->adam_beta2);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --adam-gclip N             AdamW gradient clipping. Disabled when zero. (default %f)\n", params->adam_gclip);
    # 打印参数信息到标准错误流，用于控制台输出
    fprintf(stderr, "  --adam-epsf N              AdamW epsilon for convergence test. Disabled when <= zero. (default %f)\n", params->adam_eps_f);
    # 打印空行到标准错误流，用于控制台输出
    fprintf(stderr, "\n");
}

# 消费通用的训练参数
bool consume_common_train_arg(
    int argc, char ** argv, int * idx, struct train_params_common * params, bool * invalid_param
) {
    // 获取参数索引的引用
    int& i = *idx;
    // 获取参数值
    std::string arg = argv[i];
    // 定义参数前缀
    const std::string arg_prefix = "--";
    // 检查参数是否以"--"开头
    if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
        // 将参数中的"_"替换为"-"
        std::replace(arg.begin(), arg.end(), '_', '-');
    }
    // 检查参数是否为"--train-data"
    if (arg == "--train-data") {
        // 检查下一个参数是否存在
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        // 将下一个参数赋值给训练数据文件名
        params->fn_train_data = argv[i];
    } else if (arg == "--checkpoint-in") {
        // 检查下一个参数是否存在
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        // 将下一个参数赋值给输入检查点文件名
        params->fn_checkpoint_in = argv[i];
    } else if (arg == "--checkpoint-out") {
        # 如果参数索引超出范围，设置无效参数标志并返回true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数后的值赋给对应的参数变量
        params->fn_checkpoint_out = argv[i];
    } else if (arg == "--pattern-fn-it") {
        # 如果参数索引超出范围，设置无效参数标志并返回true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数后的值赋给对应的参数变量
        params->pattern_fn_it = argv[i];
    } else if (arg == "--fn-latest") {
        # 如果参数索引超出范围，设置无效参数标志并返回true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数后的值赋给对应的参数变量
        params->fn_latest = argv[i];
    } else if (arg == "--save-every") {
        # 如果参数索引超出范围，设置无效参数标志并返回true
        if (++i >= argc) {
            *invalid_param = true;
        # 如果参数为 true，则返回 true
        return true;
        # 如果参数为 save_every，则将其转换为整数并赋值给 params->save_every
        params->save_every = std::stoi(argv[i]);
    # 如果参数为 -s 或 --seed
    } else if (arg == "-s" || arg == "--seed") {
        # 如果参数不足，则将 invalid_param 设为 true 并返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数转换为整数并赋值给 params->seed
        params->seed = std::stoi(argv[i]);
    # 如果参数为 -c 或 --ctx
    } else if (arg == "-c" || arg == "--ctx") {
        # 如果参数不足，则将 invalid_param 设为 true 并返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        # 将参数转换为整数并赋值给 params->n_ctx，同时将 custom_n_ctx 设为 true
        params->n_ctx = std::stoi(argv[i]);
        params->custom_n_ctx = true;
    # 如果参数为 -t 或 --threads
    } else if (arg == "-t" || arg == "--threads") {
        # 如果参数不足，则将 invalid_param 设为 true 并返回 true
        if (++i >= argc) {
            *invalid_param = true;
            return true;
    } 
    // 如果参数是 -t 或 --threads，则将下一个参数转换为整数并赋值给 n_threads
    else if (arg == "-t" || arg == "--threads") {
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->n_threads = std::stoi(argv[i]);
    } 
    // 如果参数是 -b 或 --batch，则将下一个参数转换为整数并赋值给 n_batch
    else if (arg == "-b" || arg == "--batch") {
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->n_batch = std::stoi(argv[i]);
    } 
    // 如果参数是 --grad-acc，则将下一个参数转换为整数并赋值给 n_gradient_accumulation
    else if (arg == "--grad-acc") {
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->n_gradient_accumulation = std::max(1, std::stoi(argv[i]));
    } 
    // 如果参数是 --sample-start，则将下一个参数转换为字符串并赋值给 sample_start
    else if (arg == "--sample-start") {
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->sample_start = std::string(argv[i]);
    } else if (arg == "--escape") {
        // 如果参数为"--escape"，则将params->escape设置为true
        params->escape = true;
    } else if (arg == "--include-sample-start") {
        // 如果参数为"--include-sample-start"，则将params->include_sample_start设置为true
        params->include_sample_start = true;
    } else if (arg == "--overlapping-samples") {
        // 如果参数为"--overlapping-samples"，则将params->overlapping_samples设置为true
        params->overlapping_samples = true;
    } else if (arg == "--fill-with-next-samples") {
        // 如果参数为"--fill-with-next-samples"，则将params->fill_with_next_samples设置为true
        params->fill_with_next_samples = true;
    } else if (arg == "--separate-with-eos") {
        // 如果参数为"--separate-with-eos"，则将params->separate_with_eos设置为true
        params->separate_with_eos = true;
    } else if (arg == "--separate-with-bos") {
        // 如果参数为"--separate-with-bos"，则将params->separate_with_bos设置为true
        params->separate_with_bos = true;
    } else if (arg == "--no-separate-with-eos") {
        // 如果参数为"--no-separate-with-eos"，则将params->separate_with_eos设置为false
        params->separate_with_eos = false;
    } else if (arg == "--no-separate-with-bos") {
        // 如果参数为"--no-separate-with-bos"，则将params->separate_with_bos设置为false
        params->separate_with_bos = false;
    } else if (arg == "--sample-random-offsets") {
        // 如果参数为"--sample-random-offsets"，则将params->sample_random_offsets设置为true
        params->sample_random_offsets = true;
    } else if (arg == "--force-reshuffle") {
        // 如果参数为"--force-reshuffle"，则将params->force_reshuffle设置为true
        params->force_reshuffle = true;
    } else if (arg == "--no-flash") {
        // 如果参数为"--no-flash"，则设置使用闪存为false
        params->use_flash = false;
    } else if (arg == "--use-flash") {
        // 如果参数为"--use-flash"，则设置使用闪存为true
        params->use_flash = true;
    } else if (arg == "--no-checkpointing") {
        // 如果参数为"--no-checkpointing"，则设置使用检查点为false
        params->use_checkpointing = false;
    } else if (arg == "--use-checkpointing") {
        // 如果参数为"--use-checkpointing"，则设置使用检查点为true
        params->use_checkpointing = true;
    } else if (arg == "--warmup") {
        // 如果参数为"--warmup"，则获取下一个参数作为热身次数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->warmup = std::stoi(argv[i]);
    } else if (arg == "--cos-decay-steps") {
        // 如果参数为"--cos-decay-steps"，则获取下一个参数作为余弦衰减步数
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->cos_decay_steps = std::stoi(argv[i]);
    } else if (arg == "--cos-decay-restart") {
        // 如果参数为 "--cos-decay-restart"，则获取下一个参数作为参数值，并转换为浮点数，赋值给参数对象的 cos_decay_restart
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->cos_decay_restart = std::stof(argv[i]);
    } else if (arg == "--cos-decay-min") {
        // 如果参数为 "--cos-decay-min"，则获取下一个参数作为参数值，并转换为浮点数，赋值给参数对象的 cos_decay_min
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->cos_decay_min = std::stof(argv[i]);
    } else if (arg == "--enable-restart") {
        // 如果参数为 "--enable-restart"，则将参数对象的 enable_restart 设置为 true
        params->enable_restart = true;
    } else if (arg == "--disable-restart") {
        // 如果参数为 "--disable-restart"，则将参数对象的 enable_restart 设置为 false
        params->enable_restart = false;
    } else if (arg == "--opt-past") {
        // 如果参数为 "--opt-past"，则获取下一个参数作为参数值，并转换为浮点数，赋值给参数对象的 opt_past
        if (++i >= argc) {
            *invalid_param = true;
            return true;
    } else if (arg == "--opt-past") {
        // 如果参数为--opt-past，则将下一个参数转换为整数并赋值给opt_past
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->opt_past = std::stoi(argv[i]);
    } else if (arg == "--opt-delta") {
        // 如果参数为--opt-delta，则将下一个参数转换为浮点数并赋值给opt_delta
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->opt_delta = std::stof(argv[i]);
    } else if (arg == "--opt-max-no-improvement") {
        // 如果参数为--opt-max-no-improvement，则将下一个参数转换为整数并赋值给opt_max_no_improvement
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->opt_max_no_improvement = std::stoi(argv[i]);
    } else if (arg == "--adam-epsf") {
        // 如果参数为--adam-epsf，则将下一个参数转换为浮点数并赋值给adam_eps_f
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_eps_f = std::stof(argv[i]);
    } else if (arg == "--epochs") {
        // 如果参数是 "--epochs"，则获取下一个参数作为训练轮数，并转换为整数赋值给params->n_epochs
        if (++i >= argc) {
            // 如果没有足够的参数，则设置invalid_param为true，并返回true
            *invalid_param = true;
            return true;
        }
        params->n_epochs = std::stoi(argv[i]);
    } else if (arg == "--adam-iter") {
        // 如果参数是 "--adam-iter"，则获取下一个参数作为Adam优化器的迭代次数，并转换为整数赋值给params->adam_n_iter
        if (++i >= argc) {
            // 如果没有足够的参数，则设置invalid_param为true，并返回true
            *invalid_param = true;
            return true;
        }
        params->adam_n_iter = std::stoi(argv[i]);
    } else if (arg == "--adam-alpha") {
        // 如果参数是 "--adam-alpha"，则获取下一个参数作为Adam优化器的学习率，并转换为浮点数赋值给params->adam_alpha
        if (++i >= argc) {
            // 如果没有足够的参数，则设置invalid_param为true，并返回true
            *invalid_param = true;
            return true;
        }
        params->adam_alpha = std::stof(argv[i]);
    } else if (arg == "--adam-min-alpha") {
        // 如果参数是 "--adam-min-alpha"，则获取下一个参数作为Adam优化器的最小学习率，并转换为浮点数赋值给params->adam_min_alpha
        if (++i >= argc) {
// 检查参数是否有效，如果无效则设置标志位并返回true
*invalid_param = true;
return true;
// 如果参数为--adam-alpha，则将下一个参数转换为浮点数并赋值给params->adam_alpha
params->adam_alpha = std::stof(argv[i]);
// 如果参数为--adam-decay，则将下一个参数转换为浮点数并赋值给params->adam_decay
} else if (arg == "--adam-decay") {
    if (++i >= argc) {
        *invalid_param = true;
        return true;
    }
    params->adam_decay = std::stof(argv[i]);
// 如果参数为--adam-decay-min-ndim，则将下一个参数转换为整数并赋值给params->adam_decay_min_ndim
} else if (arg == "--adam-decay-min-ndim") {
    if (++i >= argc) {
        *invalid_param = true;
        return true;
    }
    params->adam_decay_min_ndim = std::stoi(argv[i]);
// 如果参数为--adam-beta1，则将下一个参数转换为浮点数并赋值给params->adam_beta1
} else if (arg == "--adam-beta1") {
    if (++i >= argc) {
        *invalid_param = true;
        return true;
    } 
    // 如果参数是"--adam-beta1"，则将下一个参数转换为浮点数并赋值给adam_beta1
    else if (arg == "--adam-beta1") {
        // 检查下一个参数是否存在
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_beta1 = std::stof(argv[i]);
    } 
    // 如果参数是"--adam-beta2"，则将下一个参数转换为浮点数并赋值给adam_beta2
    else if (arg == "--adam-beta2") {
        // 检查下一个参数是否存在
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_beta2 = std::stof(argv[i]);
    } 
    // 如果参数是"--adam-gclip"，则将下一个参数转换为浮点数并赋值给adam_gclip
    else if (arg == "--adam-gclip") {
        // 检查下一个参数是否存在
        if (++i >= argc) {
            *invalid_param = true;
            return true;
        }
        params->adam_gclip = std::stof(argv[i]);
    } 
    // 如果参数是"-h"或"--help"，则将print_usage设置为true
    else if (arg == "-h" || arg == "--help") {
        params->print_usage = true;
        return true;
    } 
    // 如果参数不匹配任何已知参数，则返回false
    else {
        return false;
    }
    // 返回 true
    return true;
}

// 完成处理训练参数
void finish_processing_train_args(struct train_params_common * params) {
    // 如果参数中包含转义字符，处理转义字符
    if (params->escape) {
        process_escapes(params->sample_start);
    }
}

// 训练选项回调函数
void train_opt_callback(void * vdata, int accum_step, float * sched, bool * cancel) {
    // 将 void 指针转换为训练选项回调数据结构
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

    // 如果累积步数为 0
    if (accum_step == 0) {
        // 计算当前时间
        int64_t now = ggml_time_ms();
        // 如果当前时间大于上次记录的时间，并且迭代次数大于第一次迭代
        if (now > data->last_time && opt->iter > data->first_iter) {
            // 计算时间差
            double dt = (double) (now - data->last_time);
            // 如果每次迭代的时间为0，则将时间差赋值给每次迭代的时间
            if (data->millis_per_iter == 0.0) {
                data->millis_per_iter = dt;
            } else {
                // 否则根据一定的比例更新每次迭代的时间
                const double gain = 0.7;
                data->millis_per_iter = data->millis_per_iter*(1.0-gain) + dt*gain;
            }
        }

        // 计算剩余时间
        double remaining_millis = 0.0;
        if (data->millis_per_iter > 0.0) {
            const int n_iter = params->adam_n_iter;
            const int done_iter = opt->iter - data->first_iter;
            const int remaining_iter = n_iter - done_iter;
            remaining_millis = remaining_iter * data->millis_per_iter;
        }

        // 文件保存
        // 判断是否需要保存文件
        const bool save_now = (params->save_every > 0) && (opt->iter - data->last_save_iter >= params->save_every);
        // 如果需要保存当前状态
        if (save_now) {
            // 计算新的迭代次数
            int new_iters = opt->iter - data->last_save_iter;
            // 更新训练总迭代次数和总词数
            train->train_its    += new_iters;
            train->train_tokens += new_iters * opt->params.n_gradient_accumulation * n_batch * n_ctx;

            // 如果有保存回调函数，则调用保存回调函数
            if (data->save_cb) {
                data->save_cb(data->save_data, train);
            }

            // 更新上次保存的迭代次数
            data->last_save_iter = opt->iter;
        }

        // 通过在保存后测量 last_time 来排除文件保存对时间测量的影响
        data->last_time = ggml_time_ms();

        // 更新学习率调度
        *sched = learning_schedule(
            opt->iter,
            params->warmup,
            params->cos_decay_steps,
            params->adam_alpha,
        // 计算并设置impr_plot的值，用于绘制图表
        int impr_plot = -(int)(1 + (opt->loss_before - opt->loss_after) * 10.0f + 0.5f);
        // 如果impr_plot大于0，则将其设置为0
        if (impr_plot > 0) impr_plot = 0;
        // 如果opt->loss_before或opt->loss_after为NaN，则将impr_plot设置为0
        if (std::isnan(opt->loss_before) || std::isnan(opt->loss_after)) impr_plot = 0;
        // 打印函数名、迭代次数、样本数量、调度值、损失值
        printf("%s: iter=%6d sample=%zu/%zu sched=%f loss=%f",
            __func__, opt->iter, std::min(1+train->shuffle_next_sample, train->shuffle_sample_count), train->shuffle_sample_count,
            *sched, opt->loss_after);

        // 如果每次迭代的时间大于0，则打印迭代时间和剩余时间
        if (data->millis_per_iter > 0) {
            printf(" dt=");
            print_duration(data->millis_per_iter);
            printf(" eta=");
            print_duration(remaining_millis);
        }
        // 计算优化前后损失的改善值
        float improvement = opt->loss_before - opt->loss_after;
        // 设置绘图比例
        const float plot_scale = 10.0f;
        // 根据改善值和绘图比例计算进度条长度
        int bar_len = (int)(1 + improvement*plot_scale + 0.5);
        // 打印进度条起始符号
        printf(" |");
        // 根据进度条长度打印对应数量的横线
        for (int i=0; i<bar_len; ++i) {
            printf("-");
        }
        // 打印进度条结束符号
        printf(">");
        // 换行
        printf("\n");
    }

    // 获取示例目标批次的使用样本数
    int64_t used_samples = get_example_targets_batch(
        data->lctx,
        data->tokens_input,
        data->target_probs,
        train->shuffle_next_sample,
        data->shuffled_samples_offs,
        data->shuffled_samples_begin,
        data->shuffled_samples_size,
        data->samples_count,
    // 将数据中的 tokens_data 和 tokens_size 赋值给 train 对象
    data->tokens_data,
    data->tokens_size,
    // 将参数中的 separate_with_eos、separate_with_bos、fill_with_next_samples、sample_random_offsets 赋值给 train 对象
    params->separate_with_eos,
    params->separate_with_bos,
    params->fill_with_next_samples,
    params->sample_random_offsets);

    // 更新 train 对象中的 train_samples 和 shuffle_next_sample
    train->train_samples += used_samples;
    train->shuffle_next_sample += used_samples;

    // 如果 shuffle_next_sample 大于等于 shuffle_sample_count，则增加 train_epochs，并打印信息
    if (train->shuffle_next_sample >= train->shuffle_sample_count) {
        ++train->train_epochs;
        printf("%s: reshuffle samples. completed epochs: %llu\n", __func__, (long long unsigned) train->train_epochs);
        // 注意：当前的洗牌可能会多次使用相同的样本
        // 更新 shuffle_rng_state_current 和 shuffle_rng_state_next，并调用 shuffle_samples 函数重新洗牌
        train->shuffle_rng_state_current = train->shuffle_rng_state_next;
        train->shuffle_rng_state_next = shuffle_samples(
            train->shuffle_rng_state_current,
            data->shuffled_samples_offs,
            data->shuffled_samples_begin,
            data->shuffled_samples_size,
    // 设置训练数据的起始样本位置
    data->samples_begin,
    // 设置训练数据的样本大小
    data->samples_size,
    // 设置训练数据的样本数量
    data->samples_count);
    // 重置下一个要被洗牌的样本的索引
    train->shuffle_next_sample = 0;
}

// 检查是否达到了最后一个周期
const bool last_epoch_reached = (params->n_epochs > 0 && (int64_t) train->train_epochs - data->first_epoch >= params->n_epochs);
if (last_epoch_reached) {
    // 在取消之前允许完成最后一个周期的优化迭代
    if (data->iter_at_last_epoch < 0) {
        // 设置最后一个周期的迭代次数
        data->iter_at_last_epoch = opt->iter;
    } else if (opt->iter > data->iter_at_last_epoch) {
        // 如果当前迭代次数大于最后一个周期的迭代次数，则取消训练
        *cancel = true;
    }
}
```