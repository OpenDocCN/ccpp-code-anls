# `PowerInfer\tests\test-sampling.cpp`

```
#include "ggml.h"  // 引入 ggml 库的头文件
#include "llama.h"  // 引入 llama 库的头文件

#ifdef NDEBUG  // 如果定义了 NDEBUG，则取消定义 NDEBUG
#undef NDEBUG
#endif

#include <cmath>  // 引入数学函数库
#include <numeric>  // 引入数值操作库
#include <cassert>  // 引入断言库
#include <vector>  // 引入向量库
#include <algorithm>  // 引入算法库

static void dump(const llama_token_data_array * candidates) {  // 定义一个静态函数 dump，用于打印候选项数据
    for (size_t i = 0; i < candidates->size; i++) {  // 遍历候选项数据
        printf("%d: %f (%f)\n", candidates->data[i].id, candidates->data[i].p, candidates->data[i].logit);  // 打印候选项的 id、概率和 logit 值
    }
}

#define DUMP(__candidates) do { printf("%s:%d (%s)\n", __FILE__, __LINE__, __func__); dump((__candidates)); printf("-\n"); } while(0)  // 定义宏 DUMP，用于打印文件名、行号和函数名，并调用 dump 函数打印候选项数据

static void test_top_k(const std::vector<float> & probs, const std::vector<float> & expected_probs, int k) {  // 定义一个静态函数 test_top_k，用于测试 top k 算法
    size_t n_vocab = probs.size();  // 获取概率向量的大小
    std::vector<llama_token_data> candidates;  // 定义候选项数据的向量
    candidates.reserve(n_vocab);  // 预留空间
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {  // 遍历概率向量
        float logit = log(probs[token_id]);  // 计算 logit 值
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});  // 将候选项数据加入候选项向量
    }

    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };  // 定义候选项数据数组
    llama_sample_softmax(nullptr, &candidates_p);  // 对候选项数据进行 softmax 处理
    DUMP(&candidates_p);  // 调用宏 DUMP 打印候选项数据
    llama_sample_top_k(nullptr, &candidates_p, k, 1);  // 对候选项数据进行 top k 处理
    DUMP(&candidates_p);  // 调用宏 DUMP 打印候选项数据

    GGML_ASSERT(candidates_p.size == expected_probs.size());  // 使用断言检查候选项数据的大小是否与期望的概率向量大小相同
    for (size_t i = 0; i < candidates_p.size; i++) {  // 遍历候选项数据
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-5);  // 使用断言检查候选项数据的概率与期望的概率向量的差值是否小于 1e-5
    }
}

static void test_top_p(const std::vector<float> & probs, const std::vector<float> & expected_probs, float p) {  // 定义一个静态函数 test_top_p，用于测试 top p 算法
    size_t n_vocab = probs.size();  // 获取概率向量的大小
    std::vector<llama_token_data> candidates;  // 定义候选项数据的向量
    candidates.reserve(n_vocab);  // 预留空间
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {  // 遍历概率向量
        float logit = log(probs[token_id]);  // 计算 logit 值
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});  // 将候选项数据加入候选项向量
    }

    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };  // 定义候选项数据数组
    llama_sample_softmax(nullptr, &candidates_p);  // 对候选项数据进行 softmax 处理
    # 将候选人概率的地址打印出来
    DUMP(&candidates_p);
    # 从候选人中取样，更新候选人概率
    llama_sample_top_p(nullptr, &candidates_p, p, 1);
    # 再次打印候选人概率的地址
    DUMP(&candidates_p);

    # 断言候选人概率的大小等于期望概率的大小
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    # 遍历候选人概率，断言每个概率值与期望概率值的差的绝对值小于1e-3
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-3);
    }
# 测试 TFS 算法的函数，接受概率、期望概率和 z 值作为参数
static void test_tfs(const std::vector<float> & probs, const std::vector<float> & expected_probs, float z) {
    # 获取概率向量的大小
    size_t n_vocab = probs.size();
    # 创建候选词数据的向量
    std::vector<llama_token_data> candidates;
    # 预留空间以容纳 n_vocab 个元素
    candidates.reserve(n_vocab);
    # 遍历概率向量，为每个词创建候选词数据
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {
        # 计算对数概率
        float logit = log(probs[token_id]);
        # 将候选词数据添加到候选词向量中
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});
    }

    # 创建候选词数据数组
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
    # 打印候选词数据数组
    DUMP(&candidates_p);
    # 使用 TFS 算法进行采样
    llama_sample_tail_free(nullptr, &candidates_p, z, 1);
    # 打印候选词数据数组
    DUMP(&candidates_p);

    # 断言候选词数据数组的大小与期望概率的大小相等
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    # 遍历候选词数据数组，断言每个候选词的概率与期望概率的差值小于 1e-3
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-3);
    }
}

# 测试典型采样算法的函数，接受概率、期望概率和 p 值作为参数
static void test_typical(const std::vector<float> & probs, const std::vector<float> & expected_probs, float p) {
    # 获取概率向量的大小
    size_t n_vocab = probs.size();
    # 创建候选词数据的向量
    std::vector<llama_token_data> candidates;
    # 预留空间以容纳 n_vocab 个元素
    candidates.reserve(n_vocab);
    # 遍历概率向量，为每个词创建候选词数据
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {
        # 计算对数概率
        float logit = log(probs[token_id]);
        # 将候选词数据添加到候选词向量中
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});
    }

    # 创建候选词数据数组
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
    # 打印候选词数据数组
    DUMP(&candidates_p);
    # 使用典型采样算法进行采样
    llama_sample_typical(nullptr, &candidates_p, p, 1);
    # 打印候选词数据数组
    DUMP(&candidates_p);

    # 断言候选词数据数组的大小与期望概率的大小相等
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    # 遍历候选词数据数组，断言每个候选词的概率与期望概率的差值小于 1e-3
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-3);
    }
}

# 测试重复惩罚的函数，接受概率、上一个词的标识、期望概率、重复惩罚、频率参数和存在参数作为参数
static void test_repetition_penalties(
    const std::vector<float> & probs, const std::vector<llama_token> & last_tokens,
    const std::vector<float> & expected_probs, float repeat_penalty, float alpha_frequency, float alpha_presence
) {
    # 断言概率向量和期望概率向量的大小相等
    GGML_ASSERT(probs.size() == expected_probs.size());

    # 获取概率向量的大小
    size_t n_vocab = probs.size();
    # 创建一个空的候选列表，预留 n_vocab 个空间
    std::vector<llama_token_data> candidates;
    candidates.reserve(n_vocab);
    # 遍历 token_id，将每个 token_id 对应的概率计算对数后加入候选列表
    for (llama_token token_id = 0; token_id < (llama_token)n_vocab; token_id++) {
        float logit = log(probs[token_id]);
        candidates.emplace_back(llama_token_data{token_id, logit, 0.0f});
    }

    # 创建候选列表的 C 语言结构体
    llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
    # 对候选列表进行 softmax 操作
    llama_sample_softmax(nullptr, &candidates_p);
    # 打印候选列表
    DUMP(&candidates_p);
    # 对候选列表进行重复惩罚操作
    llama_sample_repetition_penalties(nullptr, &candidates_p, (const llama_token *) last_tokens.data(), last_tokens.size(), repeat_penalty, alpha_frequency, alpha_presence);
    # 再次对候选列表进行 softmax 操作
    llama_sample_softmax(nullptr, &candidates_p);
    # 打印候选列表
    DUMP(&candidates_p);

    # 断言候选列表的大小与期望概率列表的大小相等
    GGML_ASSERT(candidates_p.size == expected_probs.size());
    # 遍历候选列表，断言每个候选的概率与期望概率的差的绝对值小于 1e-3
    for (size_t i = 0; i < candidates_p.size; i++) {
        GGML_ASSERT(fabs(candidates_p.data[i].p - expected_probs[i]) < 1e-3);
    }
# 主函数，程序的入口
int main(void) {
    # 初始化时间
    ggml_time_init();

    # 测试 top_k 函数
    test_top_k({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f}, 1);
    test_top_k({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f, 0.3f, 0.2f}, 3);

    # 测试 top_p 函数
    test_top_p({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f}, 0);
    test_top_p({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f, 0.3f}, 0.7f);
    test_top_p({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f, 0.3f, 0.2f}, 0.8f);
    test_top_p({0.1f, 0.2f, 0.3f, 0.4f}, {0.4f, 0.3f, 0.2f, 0.1f}, 1);

    # 测试 tfs 函数
    test_tfs({0.1f, 0.15f, 0.2f, 0.25f, 0.3f}, {0.3f}, 0.25f);
    test_tfs({0.1f, 0.15f, 0.2f, 0.25f, 0.3f}, {0.3f, 0.25f}, 0.75f);
    test_tfs({0.1f, 0.15f, 0.2f, 0.25f, 0.3f}, {0.3f, 0.25f}, 0.99f);

    # 测试 typical 函数
    test_typical({0.97f, 0.01f, 0.01f, 0.01f}, {0.97f}, 0.5f);
    test_typical({0.4f, 0.2f, 0.2f, 0.2f}, {0.2f, 0.2f, 0.2f}, 0.5f);

    # 测试 repetition_penalties 函数
    test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0}, {0.25f, 0.25f, 0.25f, 0.25f, 0},   50.0f, 0.0f, 0.0f);
    test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0, 1, 2}, {0.5f, 0.5f, 0, 0, 0},       50.0f, 0.0f, 0.0f);
    test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0, 1, 2, 0, 0}, {0.5f, 0.5f, 0, 0, 0}, 50.0f, 0.0f, 0.0f);

    test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0},             {0.249997f, 0.249997f, 0.249997f, 0.249997f, 0.000011f}, 1.0f, 5.0f, 5.0f);
    test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0, 1, 2},       {0.499966f, 0.499966f, 0.000023f, 0.000023f, 0.000023f}, 1.0f, 5.0f, 5.0f);
    test_repetition_penalties({0.2f, 0.2f, 0.2f, 0.2f, 0.2f}, {0, 1, 2, 0, 0}, {0.499977f, 0.499977f, 0.000023f, 0.000023f, 0.000000f}, 1.0f, 5.0f, 5.0f);

    # 输出测试结果
    printf("OK\n");

    # 返回 0 表示程序正常结束
    return 0;
}
```