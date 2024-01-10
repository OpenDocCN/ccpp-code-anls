# `PowerInfer\common\sampling.h`

```
#pragma once

#include "llama.h"  // 包含 llama.h 头文件

#include "grammar-parser.h"  // 包含 grammar-parser.h 头文件

#include <string>  // 包含 string 头文件
#include <vector>  // 包含 vector 头文件
#include <unordered_map>  // 包含 unordered_map 头文件

// 采样参数
typedef struct llama_sampling_params {
    int32_t n_prev            = 64;    // 记忆的前 n 个标记数
    int32_t n_probs           = 0;     // 如果大于 0，则输出前 n_probs 个标记的概率
    int32_t top_k             = 40;    // <= 0 使用词汇表大小
    float   top_p             = 0.95f; // 1.0 = 禁用
    float   min_p             = 0.05f; // 0.0 = 禁用
    float   tfs_z             = 1.00f; // 1.0 = 禁用
    float   typical_p         = 1.00f; // 1.0 = 禁用
    float   temp              = 0.80f; // 1.0 = 禁用
    int32_t penalty_last_n    = 64;    // 最后 n 个标记进行惩罚 (0 = 禁用惩罚, -1 = 上下文大小)
    float   penalty_repeat    = 1.10f; // 1.0 = 禁用
    float   penalty_freq      = 0.00f; // 0.0 = 禁用
    float   penalty_present   = 0.00f; // 0.0 = 禁用
    int32_t mirostat          = 0;     // 0 = 禁用, 1 = mirostat, 2 = mirostat 2.0
    float   mirostat_tau      = 5.00f; // 目标熵
    float   mirostat_eta      = 0.10f; // 学习率
    bool    penalize_nl       = true;  // 将换行符视为可重复的标记

    std::string grammar;  // 可选的类似 BNF 的语法，用于约束采样

    // 无分类器指导
    // https://arxiv.org/abs/2306.17806
    std::string cfg_negative_prompt; // 用于辅助指导的字符串
    float       cfg_scale     = 1.f; // 指导的强度

    std::unordered_map<llama_token, float> logit_bias; // 特定标记的 logit 偏置
} llama_sampling_params;

// 通用采样器上下文
// TODO: 移动到 llama.h
struct llama_sampling_context {
    // 将用于采样的参数
    llama_sampling_params params;

    // mirostat 采样器状态
    float mirostat_mu;

    llama_grammar * grammar;

    // 内部
    grammar_parser::parse_state parsed_grammar;
    // 用环形缓冲替换，待实现
    std::vector<llama_token>      prev;  // 用于存储前一个状态的 token
    std::vector<llama_token_data> cur;   // 用于存储当前状态的 token 数据
// 结构体定义结束

#include "common.h"

// 创建一个新的采样上下文实例
struct llama_sampling_context * llama_sampling_init(const struct llama_sampling_params & params);

// 释放采样上下文
void llama_sampling_free(struct llama_sampling_context * ctx);

// 重置采样器上下文
// - 清除先前的标记
// - 重置语法
void llama_sampling_reset(llama_sampling_context * ctx);

// 复制采样器上下文
void llama_sampling_cp(llama_sampling_context * src, llama_sampling_context * dst);

// 获取最后采样的标记
llama_token llama_sampling_last(llama_sampling_context * ctx);

// 获取最后采样标记的字符串表示
std::string llama_sampling_prev_str(llama_sampling_context * ctx_sampling, llama_context * ctx_main, int n);

// 将采样参数打印成字符串
std::string llama_sampling_print(const llama_sampling_params & params);

// 这是一个常见的采样函数，用于方便起见在示例中使用
// 它可以作为实现自己的采样函数的起点
// 注意：当使用多个序列时，调用者有责任在序列结束时调用llama_sampling_reset
//
// 必需参数：
//  - ctx_main：用于采样的上下文
//  - ctx_sampling：特定于采样的上下文
//
// 可选参数：
//  - ctx_cfg：用于无分类器指导的上下文
//  - idx：从llama_get_logits_ith(ctx, idx)中采样
//
// 返回：
//  - token：采样的标记
//  - candidates：候选标记的向量
//
llama_token llama_sampling_sample(
        struct llama_sampling_context * ctx_sampling,
        struct llama_context * ctx_main,
        struct llama_context * ctx_cfg,
        int idx = 0);

void llama_sampling_accept(
        struct llama_sampling_context * ctx_sampling,
        struct llama_context * ctx_main,
        llama_token id,
        bool apply_grammar);
```