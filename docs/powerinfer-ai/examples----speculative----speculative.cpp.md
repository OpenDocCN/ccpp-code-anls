# `PowerInfer\examples\speculative\speculative.cpp`

```cpp
#include "common.h"
#include "llama.h"

#include <cmath>
#include <cstdio>
#include <string>
#include <vector>

#define SPEC_VOCAB_MAX_SIZE_DIFFERENCE  100
#define SPEC_VOCAB_CHECK_START_TOKEN_ID 5

// 定义结构体 seq_draft
struct seq_draft {
    bool active   = false;  // 活跃状态，默认为 false
    bool drafting = false;  // 起草状态，默认为 false
    bool skip     = false;  // 跳过状态，默认为 false

    int i_batch_dft = 0;  // 起草批次，默认为 0
    std::vector<int> i_batch_tgt;  // 目标批次的整数向量

    std::vector<llama_token> tokens;  // 存储 llama_token 结构体的向量

    struct llama_sampling_context * ctx_sampling;  // 指向 llama_sampling_context 结构体的指针
};

// 主函数
int main(int argc, char ** argv) {
    gpt_params params;  // 定义 gpt_params 结构体变量 params

    // 解析命令行参数，如果解析失败则返回 1
    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果起草模型为空，则打印错误信息并返回 1
    if (params.model_draft.empty()) {
        fprintf(stderr, "%s: error: --model-draft is required\n", __func__);
        return 1;
    }

    // 并行起草序列的最大数量
    const int n_seq_dft = params.n_parallel;

    // 接受起草模型中标记的概率阈值
    const float p_accept = params.p_accept;

    // 分裂起草分支的概率阈值（仅适用于 n_seq_dft > 1）
    const float p_split  = params.p_split;

#ifndef LOG_DISABLE_LOGS
    // 设置日志输出目标为 "speculative.log"，并输出日志开始信息
    log_set_target(log_filename_generator("speculative", "log"));
    LOG_TEE("Log start\n");
    // 输出命令行参数信息
    log_dump_cmdline(argc, argv);
#endif // LOG_DISABLE_LOGS

    // 初始化 llama.cpp
    llama_backend_init(params.numa);

    llama_model * model_tgt = NULL;  // 目标模型指针初始化为空
    llama_model * model_dft = NULL;  // 起草模型指针初始化为空

    llama_context * ctx_tgt = NULL;  // 目标上下文指针初始化为空
    llama_context * ctx_dft = NULL;  // 起草上下文指针初始化为空

    // 加载目标模型
    params.logits_all = true;
    std::tie(model_tgt, ctx_tgt) = llama_init_from_gpt_params(params);

    // 加载起草模型
    params.model = params.model_draft;
    params.n_gpu_layers = params.n_gpu_layers_draft;
    std::tie(model_dft, ctx_dft) = llama_init_from_gpt_params(params);
}
    {
        // 获取目标模型和草稿模型的词汇表大小
        const int n_vocab_tgt = llama_n_vocab(model_tgt);
        const int n_vocab_dft = llama_n_vocab(model_dft);
        // 计算词汇表大小的差异
        const int vocab_diff  = n_vocab_tgt > n_vocab_dft
            ? n_vocab_tgt - n_vocab_dft
            : n_vocab_dft - n_vocab_tgt;

        // 如果词汇表大小差异超过指定阈值，则输出错误信息并返回
        if (vocab_diff > SPEC_VOCAB_MAX_SIZE_DIFFERENCE) {
            fprintf(stderr, "%s: error: draft model vocab must closely match target model to use speculation but ", __func__);
            fprintf(stderr, "target vocab size %d does not match draft vocab size %d - difference %d, max allowed %d\n",
                    n_vocab_tgt, llama_n_vocab(model_dft), vocab_diff, SPEC_VOCAB_MAX_SIZE_DIFFERENCE);
            return 1;
        }

        // 检查目标模型和草稿模型的词汇表是否一致
        for (int i = SPEC_VOCAB_CHECK_START_TOKEN_ID; i < std::min(n_vocab_tgt, n_vocab_dft); ++i) {
            const char * token_text_tgt = llama_token_get_text(model_tgt, i);
            const char * token_text_dft = llama_token_get_text(model_dft, i);
            // 如果词汇表不一致，则输出错误信息并返回
            if (std::strcmp(token_text_tgt, token_text_dft) != 0) {
                fprintf(stderr, "%s: error: draft model vocab must match target model to use speculation but ", __func__);
                fprintf(stderr, "token %d content differs - target '%s', draft '%s'\n", i,
                        llama_token_to_piece(ctx_tgt, i).c_str(),
                        llama_token_to_piece(ctx_dft, i).c_str());
                return 1;
            }
        }
    }

    // 对输入进行分词
    std::vector<llama_token> inp;
    inp = ::llama_tokenize(ctx_tgt, params.prompt, true);

    // 获取上下文的最大大小和最大标记列表大小
    const int max_context_size     = llama_n_ctx(ctx_tgt);
    const int max_tokens_list_size = max_context_size - 4;

    // 如果输入的标记数量超过最大标记列表大小，则输出错误信息并返回
    if ((int) inp.size() > max_tokens_list_size) {
        fprintf(stderr, "%s: error: prompt too long (%d tokens, max %d)\n", __func__, (int) inp.size(), max_tokens_list_size);
        return 1;
    }

    // 输出换行符
    fprintf(stderr, "\n\n");
    // 遍历输入的id数组，将每个id对应的token转换为字符串并输出到标准错误流
    for (auto id : inp) {
        fprintf(stderr, "%s", llama_token_to_piece(ctx_tgt, id).c_str());
    }

    // 刷新标准错误流
    fflush(stderr);

    // 获取输入数组的大小
    const int n_input = inp.size();

    // 获取当前时间，用于计算编码的时间
    const auto t_enc_start = ggml_time_us();

    // 使用目标模型和默认模型对输入进行解码
    llama_decode(ctx_tgt, llama_batch_get_one( inp.data(), n_input - 1, 0,           0));
    llama_decode(ctx_tgt, llama_batch_get_one(&inp.back(),           1, n_input - 1, 0));
    llama_decode(ctx_dft, llama_batch_get_one( inp.data(), n_input,     0,           0));

    // 获取编码结束时间
    const auto t_enc_end = ggml_time_us();

    // 两个模型应该有相同的词汇表
    //GGML_ASSERT(n_vocab == llama_n_vocab(model_dft));

    // 每次生成的token数量
    int n_draft = params.n_draft;

    // 初始化变量
    int n_predict = 0;
    int n_drafted = 0;
    int n_accept  = 0;

    // 初始化变量
    int n_past_tgt = inp.size();
    int n_past_dft = inp.size();

    // 用于确定生成的结束
    bool has_eos = false;

    // 初始化目标模型的采样上下文
    struct llama_sampling_context * ctx_sampling = llama_sampling_init(params.sparams);

    // 初始化draft序列数据
    std::vector<seq_draft> drafts(n_seq_dft);

    // 清空draft模型的语法，强制使用贪婪采样
    params.sparams.grammar.clear(); // the draft samplers will copy the target sampler's grammar
    params.sparams.temp = -1.0f;    // force greedy sampling with probs for the draft model

    // 初始化draft序列的采样上下文
    for (int s = 0; s < n_seq_dft; ++s) {
        drafts[s].ctx_sampling = llama_sampling_init(params.sparams);
    }

    // 初始化默认模型和目标模型的batch
    llama_batch batch_dft = llama_batch_init(params.n_ctx, 0, 1);
    llama_batch batch_tgt = llama_batch_init(params.n_ctx, 0, n_seq_dft);

    // 获取解码开始时间
    const auto t_dec_start = ggml_time_us();

    // 从prompt的最后一个token进行采样
    drafts[0].i_batch_tgt.resize(1);
    drafts[0].i_batch_tgt[0] = 0;

    // 获取解码结束时间
    auto t_dec_end = ggml_time_us();

    // 输出换行符到标准错误流
    LOG_TEE("\n\n");
    # 输出编码后的 token 数量和所花费的时间，以及编码速度
    LOG_TEE("encoded %4d tokens in %8.3f seconds, speed: %8.3f t/s\n", n_input,   (t_enc_end - t_enc_start) / 1e6f, inp.size() / ((t_enc_end - t_enc_start) / 1e6f));
    # 输出解码后的 token 数量和所花费的时间，以及解码速度
    LOG_TEE("decoded %4d tokens in %8.3f seconds, speed: %8.3f t/s\n", n_predict, (t_dec_end - t_dec_start) / 1e6f, n_predict  / ((t_dec_end - t_dec_start) / 1e6f));

    # 输出空行
    LOG_TEE("\n");
    # 输出草稿数量
    LOG_TEE("n_draft   = %d\n", n_draft);
    # 输出预测数量
    LOG_TEE("n_predict = %d\n", n_predict);
    # 输出已草拟数量
    LOG_TEE("n_drafted = %d\n", n_drafted);
    # 输出接受数量
    LOG_TEE("n_accept  = %d\n", n_accept);
    # 输出接受率
    LOG_TEE("accept    = %.3f%%\n", 100.0f * n_accept / n_drafted);

    # 输出草稿时间
    LOG_TEE("\ndraft:\n");
    llama_print_timings(ctx_dft);

    # 输出目标时间
    LOG_TEE("\ntarget:\n");
    llama_print_timings(ctx_tgt);

    # 释放采样上下文
    llama_sampling_free(ctx_sampling);
    # 释放每个草稿的采样上下文
    for (int s = 0; s < n_seq_dft; ++s) {
        llama_sampling_free(drafts[s].ctx_sampling);
    }

    # 释放草稿批次
    llama_batch_free(batch_dft);

    # 释放目标上下文
    llama_free(ctx_tgt);
    # 释放目标模型
    llama_free_model(model_tgt);

    # 释放草稿上下文
    llama_free(ctx_dft);
    # 释放草稿模型
    llama_free_model(model_dft);

    # 释放后端资源
    llama_backend_free();

    # 输出空行
    fprintf(stderr, "\n\n");

    # 返回 0
    return 0;
# 闭合函数或类的结束大括号
```