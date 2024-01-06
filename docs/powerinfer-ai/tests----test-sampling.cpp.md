# `PowerInfer\tests\test-sampling.cpp`

```
// 包含头文件 ggml.h 和 llama.h
#include "ggml.h"
#include "llama.h"

// 如果定义了 NDEBUG，则取消定义
#ifdef NDEBUG
#undef NDEBUG
#endif

// 包含数学、数值计算、断言、向量和算法的标准库头文件
#include <cmath>
#include <numeric>
#include <cassert>
#include <vector>
#include <algorithm>

// 定义一个静态函数 dump，用于打印候选项数据数组的内容
static void dump(const llama_token_data_array * candidates) {
    for (size_t i = 0; i < candidates->size; i++) {
        printf("%d: %f (%f)\n", candidates->data[i].id, candidates->data[i].p, candidates->data[i].logit);
    }
}

// 定义宏 DUMP，用于在特定位置打印候选项数据数组的内容
#define DUMP(__candidates) do { printf("%s:%d (%s)\n", __FILE__, __LINE__, __func__); dump((__candidates)); printf("-\n"); } while(0)
// 定义一个函数，用于测试前 k 个概率最高的候选项
static void test_top_k(const std::vector<float> & probs, const std::vector<float> & expected_probs, int k) {
    // 获取概率向量的大小
    size_t n_vocab = probs.size();
    // 创建候选项的数据结构
    std::vector<llama_token_data> candidates;
    // 预留足够的空间
    candidates.reserve(n_vocab);
    // 遍历概率向量，将每个概率值转换为对数形式，并添加到候选项中
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {
        float logit = log(probs[token_id]);
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});
    }

    // 创建候选项数据结构
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
    // 对候选项进行 softmax 操作
    llama_sample_softmax(nullptr, &candidates_p);
    // 打印候选项数据
    DUMP(&candidates_p);
    // 从候选项中选择前 k 个概率最高的项
    llama_sample_top_k(nullptr, &candidates_p, k, 1);
    // 打印候选项数据
    DUMP(&candidates_p);

    // 断言候选项的大小与期望概率的大小相等
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    // 遍历候选项，断言每个候选项的概率与期望概率的差值小于 1e-5
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-5);
    }
}
// 定义一个静态函数，用于测试给定概率和期望概率的top_p采样
static void test_top_p(const std::vector<float> & probs, const std::vector<float> & expected_probs, float p) {
    // 获取概率向量的大小
    size_t n_vocab = probs.size();
    // 创建候选项数据的向量，并预留空间
    std::vector<llama_token_data> candidates;
    candidates.reserve(n_vocab);
    // 遍历概率向量，计算对数概率并将候选项数据添加到候选项向量中
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {
        float logit = log(probs[token_id]);
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});
    }

    // 创建候选项数据数组
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
    // 对候选项进行softmax采样
    llama_sample_softmax(nullptr, &candidates_p);
    // 打印候选项数据数组
    DUMP(&candidates_p);
    // 对候选项进行top_p采样
    llama_sample_top_p(nullptr, &candidates_p, p, 1);
    // 打印经过top_p采样后的候选项数据数组
    DUMP(&candidates_p);

    // 断言经过top_p采样后的候选项数据数组的大小与期望概率的大小相等
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    // 遍历经过top_p采样后的候选项数据数组，断言每个候选项的概率与期望概率的差值小于1e-3
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-3);
    }
}

static void test_tfs(const std::vector<float> & probs, const std::vector<float> & expected_probs, float z) {
    // 获取概率向量的大小
    size_t n_vocab = probs.size();
    // 创建候选词列表
    std::vector<llama_token_data> candidates;
    // 预留空间以容纳候选词
    candidates.reserve(n_vocab);
    // 遍历概率向量，计算对数概率并添加到候选词列表中
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {
        float logit = log(probs[token_id]);
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});
    }

    // 创建候选词数据数组
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
    // 打印候选词数据数组
    DUMP(&candidates_p);
    // 从候选词中采样尾部自由分布
    llama_sample_tail_free(nullptr, &candidates_p, z, 1);
    // 打印采样后的候选词数据数组
    DUMP(&candidates_p);

    // 断言采样后的候选词数量与期望概率数量相等
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    // 遍历采样后的候选词数据数组，断言每个候选词的概率与期望概率的差值小于1e-3
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-3);
    }
}

static void test_typical(const std::vector<float> & probs, const std::vector<float> & expected_probs, float p) {
    // 获取概率向量的大小
    size_t n_vocab = probs.size();
    // 创建候选项数据结构的向量
    std::vector<llama_token_data> candidates;
    // 预留空间以容纳概率向量的大小
    candidates.reserve(n_vocab);
    // 遍历概率向量，为每个 token_id 创建候选项数据结构
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {
        // 计算对数概率
        float logit = log(probs[token_id]);
        // 将候选项数据结构添加到候选项向量中
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});
    }

    // 创建候选项数据结构数组
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
    // 打印候选项数据结构数组
    DUMP(&candidates_p);
    // 从候选项数据结构数组中抽样 typcial 数据
    llama_sample_typical(nullptr, &candidates_p, p, 1);
    // 打印抽样后的候选项数据结构数组
    DUMP(&candidates_p);

    // 断言抽样后的候选项数据结构数组大小与期望的概率向量大小相等
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    // 遍历抽样后的候选项数据结构数组，断言每个候选项的概率与期望的概率接近
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-3);
// 测试重复惩罚函数，接受概率、上一个 token、期望概率、重复惩罚、alpha 频率、alpha 存在
static void test_repetition_penalties(
    const std::vector<float> & probs, const std::vector<llama_token> & last_tokens,
    const std::vector<float> & expected_probs, float repeat_penalty, float alpha_frequency, float alpha_presence
) {
    // 断言概率和期望概率的大小相等
    GGML_ASSERT(probs.size() == expected_probs.size());

    // 获取词汇表大小
    size_t n_vocab = probs.size();
    // 创建候选 token 数据的向量
    std::vector<llama_token_data> candidates;
    // 预留空间
    candidates.reserve(n_vocab);
    // 遍历词汇表，计算 logit 并添加到候选 token 数据中
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {
        float logit = log(probs[token_id]);
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});
    }

    // 创建候选 token 数据的数组
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
    // 对候选 token 数据进行 softmax 操作
    llama_sample_softmax(nullptr, &candidates_p);
    // 打印候选 token 数据
    DUMP(&candidates_p);
}
    // 调用 llama_sample_repetition_penalties 函数，传入参数为指针、候选项、上一个 token 的数据、上一个 token 的数量、重复惩罚、alpha 频率、alpha 存在
    llama_sample_repetition_penalties(nullptr, &candidates_p, (const llama_token *) last_tokens.data(), last_tokens.size(), repeat_penalty, alpha_frequency, alpha_presence);
    // 调用 llama_sample_softmax 函数，传入参数为指针、候选项
    llama_sample_softmax(nullptr, &candidates_p);
    // 打印 candidates_p 的内容
    DUMP(&candidates_p);

    // 断言 candidates_p 的大小等于 expected_probs 的大小
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    // 遍历 candidates_p，对比每个元素的概率值是否与 expected_probs 中的值相差小于 1e-3
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-3);
    }
}

int main(void) {
    // 初始化时间
    ggml_time_init();

    // 调用 test_top_k 函数，传入参数为两个数组和一个整数
    test_top_k({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f}, 1);
    // 调用 test_top_k 函数，传入参数为两个数组和一个整数
    test_top_k({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f, 0.3f, 0.2f}, 3);

    // 调用 test_top_p 函数，传入参数为两个数组和一个浮点数
    test_top_p({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f}, 0);
    // 调用 test_top_p 函数，传入参数为两个数组和一个浮点数
    test_top_p({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f, 0.3f}, 0.7f);
    // 调用 test_top_p 函数，传入参数为两个数组和一个浮点数
    test_top_p({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f, 0.3f, 0.2f}, 0.8f);
    // 调用 test_top_p 函数，传入参数为两个数组和一个浮点数
    test_top_p({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f, 0.3f, 0.2f, 0.1f}, 1);
# 调用 test_tfs 函数，传入参数为集合 {0.1f, 0.15f, 0.2f, 0.25f, 0.3f}，集合 {0.3f}，和浮点数 0.25f
test_tfs({0.1f, 0.15f, 0.2f, 0.25f, 0.3f}, {0.3f}, 0.25f);
# 调用 test_tfs 函数，传入参数为集合 {0.1f, 0.15f, 0.2f, 0.25f, 0.3f}，集合 {0.3f, 0.25f}，和浮点数 0.75f
test_tfs({0.1f, 0.15f, 0.2f, 0.25f, 0.3f}, {0.3f, 0.25f}, 0.75f);
# 调用 test_tfs 函数，传入参数为集合 {0.1f, 0.15f, 0.2f, 0.25f, 0.3f}，集合 {0.3f, 0.25f}，和浮点数 0.99f
test_tfs({0.1f, 0.15f, 0.2f, 0.25f, 0.3f}, {0.3f, 0.25f}, 0.99f);

# 调用 test_typical 函数，传入参数为集合 {0.97f, 0.01f, 0.01f, 0.01f}，集合 {0.97f}，和浮点数 0.5f
test_typical({0.97f, 0.01f, 0.01f, 0.01f}, {0.97f}, 0.5f);
# 调用 test_typical 函数，传入参数为集合 {0.4f, 0.2f, 0.2f, 0.2f}，集合 {0.2f, 0.2f, 0.2f}，和浮点数 0.5f
test_typical({0.4f, 0.2f, 0.2f, 0.2f}, {0.2f, 0.2f, 0.2f}, 0.5f);

# 调用 test_repetition_penalties 函数，传入参数为集合 {0.2f, 0.2f, 0.2f, 0.2f, 0.2f}，集合 {0}，集合 {0.25f, 0.25f, 0.25f, 0.25f, 0}，和三个浮点数 50.0f, 0.0f, 0.0f
test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0}, {0.25f, 0.25f, 0.25f, 0.25f, 0},   50.0f, 0.0f, 0.0f);
# 调用 test_repetition_penalties 函数，传入参数为集合 {0.2f, 0.2f, 0.2f, 0.2f, 0.2f}，集合 {0, 1, 2}，集合 {0.5f, 0.5f, 0, 0, 0}，和三个浮点数 50.0f, 0.0f, 0.0f
test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0, 1, 2}, {0.5f, 0.5f, 0, 0, 0},       50.0f, 0.0f, 0.0f);
# 调用 test_repetition_penalties 函数，传入参数为集合 {0.2f, 0.2f, 0.2f, 0.2f, 0.2f}，集合 {0, 1, 2, 0, 0}，集合 {0.5f, 0.5f, 0, 0, 0}，和三个浮点数 50.0f, 0.0f, 0.0f
test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0, 1, 2, 0, 0}, {0.5f, 0.5f, 0, 0, 0}, 50.0f, 0.0f, 0.0f);

# 调用 test_repetition_penalties 函数，传入参数为集合 {0.2f, 0.2f, 0.2f, 0.2f, 0.2f}，集合 {0}，集合 {0.249997f, 0.249997f, 0.249997f, 0.249997f, 0.000011f}，和三个浮点数 1.0f, 5.0f, 5.0f
test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0},             {0.249997f, 0.249997f, 0.249997f, 0.249997f, 0.000011f}, 1.0f, 5.0f, 5.0f);
# 调用 test_repetition_penalties 函数，传入参数为集合 {0.2f, 0.2f, 0.2f, 0.2f, 0.2f}，集合 {0, 1, 2}，集合 {0.499966f, 0.499966f, 0.000023f, 0.000023f, 0.000023f}，和三个浮点数 1.0f, 5.0f, 5.0f
test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0, 1, 2},       {0.499966f, 0.499966f, 0.000023f, 0.000023f, 0.000023f}, 1.0f, 5.0f, 5.0f);
# 调用 test_repetition_penalties 函数，传入参数为集合 {0.2f, 0.2f, 0.2f, 0.2f, 0.2f}，集合 {0, 1, 2, 0, 0}，集合 {0.499977f, 0.499977f, 0.000023f, 0.000023f, 0.000000f}，和三个浮点数 1.0f, 5.0f, 5.0f
test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0, 1, 2, 0, 0}, {0.499977f, 0.499977f, 0.000023f, 0.000023f, 0.000000f}, 1.0f, 5.0f, 5.0f);

# 打印 "OK"
printf("OK");

# 返回 0
return 0;
由于提供的代码为空，无法为其添加注释。
```