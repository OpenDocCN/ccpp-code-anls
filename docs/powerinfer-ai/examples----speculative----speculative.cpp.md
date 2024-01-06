# `PowerInfer\examples\speculative\speculative.cpp`

```
// 包含自定义的头文件 common.h 和 llama.h
#include "common.h"
#include "llama.h"

// 包含标准库头文件
#include <cmath>
#include <cstdio>
#include <string>
#include <vector>

// 定义宏，指定特定词汇表的最大大小差异和检查起始标记 ID
#define SPEC_VOCAB_MAX_SIZE_DIFFERENCE  100
#define SPEC_VOCAB_CHECK_START_TOKEN_ID 5

// 定义结构体 seq_draft
struct seq_draft {
    // 布尔类型成员变量，表示是否激活、是否正在草拟、是否跳过
    bool active   = false;
    bool drafting = false;
    bool skip     = false;

    // 整型成员变量，表示批次草拟的索引，目标批次的索引
    int i_batch_dft = 0;
    std::vector<int> i_batch_tgt;

    // 存储 llama_token 结构体的向量
    std::vector<llama_token> tokens;
// 定义一个结构体指针 ctx_sampling，用于存储采样上下文信息
struct llama_sampling_context * ctx_sampling;

// 主函数
int main(int argc, char ** argv) {
    // 定义参数对象 params
    gpt_params params;

    // 解析命令行参数，如果解析失败则返回1
    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果模型草稿为空，则打印错误信息并返回1
    if (params.model_draft.empty()) {
        fprintf(stderr, "%s: error: --model-draft is required\n", __func__);
        return 1;
    }

    // 定义并初始化并行草稿序列的最大数量
    const int n_seq_dft = params.n_parallel;

    // 定义并初始化接受草稿模型中标记的概率阈值
    // 从参数中获取接受概率
    const float p_accept = params.p_accept;

    // 分裂草稿分支的概率阈值（仅适用于 n_seq_dft > 1）
    const float p_split  = params.p_split;

#ifndef LOG_DISABLE_LOGS
    // 设置日志输出目标为特定的文件名生成器生成的文件
    log_set_target(log_filename_generator("speculative", "log"));
    // 记录日志开始
    LOG_TEE("Log start\n");
    // 输出命令行参数到日志
    log_dump_cmdline(argc, argv);
#endif // LOG_DISABLE_LOGS

    // 初始化 llama.cpp
    llama_backend_init(params.numa);

    // 初始化目标模型和默认模型
    llama_model * model_tgt = NULL;
    llama_model * model_dft = NULL;

    // 初始化目标上下文和默认上下文
    llama_context * ctx_tgt = NULL;
    llama_context * ctx_dft = NULL;
    // 加载目标模型
    params.logits_all = true; // 设置参数，记录所有logits
    std::tie(model_tgt, ctx_tgt) = llama_init_from_gpt_params(params); // 从参数初始化目标模型

    // 加载草稿模型
    params.model = params.model_draft; // 设置参数的模型为草稿模型
    params.n_gpu_layers = params.n_gpu_layers_draft; // 设置参数的GPU层数为草稿模型的GPU层数
    std::tie(model_dft, ctx_dft) = llama_init_from_gpt_params(params); // 从参数初始化草稿模型

    {
        const int n_vocab_tgt = llama_n_vocab(model_tgt); // 获取目标模型的词汇表大小
        const int n_vocab_dft = llama_n_vocab(model_dft); // 获取草稿模型的词汇表大小
        const int vocab_diff  = n_vocab_tgt > n_vocab_dft
            ? n_vocab_tgt - n_vocab_dft
            : n_vocab_dft - n_vocab_tgt; // 计算词汇表大小的差异

        if (vocab_diff > SPEC_VOCAB_MAX_SIZE_DIFFERENCE) { // 如果词汇表大小的差异超过最大允许差异
            fprintf(stderr, "%s: error: draft model vocab must closely match target model to use speculation but ", __func__); // 输出错误信息
            fprintf(stderr, "target vocab size %d does not match draft vocab size %d - difference %d, max allowed %d\n",
                    n_vocab_tgt, llama_n_vocab(model_dft), vocab_diff, SPEC_VOCAB_MAX_SIZE_DIFFERENCE); // 输出词汇表大小不匹配的详细信息
    // 返回值为1，表示出现错误
    return 1;
}

// 遍历特定范围内的词汇表，比较目标模型和草稿模型的词汇是否一致
for (int i = SPEC_VOCAB_CHECK_START_TOKEN_ID; i < std::min(n_vocab_tgt, n_vocab_dft); ++i) {
    // 获取目标模型和草稿模型中指定词汇的文本内容
    const char * token_text_tgt = llama_token_get_text(model_tgt, i);
    const char * token_text_dft = llama_token_get_text(model_dft, i);
    // 如果两个词汇的文本内容不一致，则输出错误信息并返回值为1，表示出现错误
    if (std::strcmp(token_text_tgt, token_text_dft) != 0) {
        fprintf(stderr, "%s: error: draft model vocab must match target model to use speculation but ", __func__);
        fprintf(stderr, "token %d content differs - target '%s', draft '%s'\n", i,
                llama_token_to_piece(ctx_tgt, i).c_str(),
                llama_token_to_piece(ctx_dft, i).c_str());
        return 1;
    }
}

// 对输入的提示进行分词处理
std::vector<llama_token> inp;
inp = ::llama_tokenize(ctx_tgt, params.prompt, true);
// 定义最大上下文大小为目标上下文的 llama_n_ctx 函数返回值
const int max_context_size = llama_n_ctx(ctx_tgt);
// 定义最大标记列表大小为最大上下文大小减去4
const int max_tokens_list_size = max_context_size - 4;

// 如果输入的标记数量大于最大标记列表大小
if ((int) inp.size() > max_tokens_list_size) {
    // 输出错误信息并返回1
    fprintf(stderr, "%s: error: prompt too long (%d tokens, max %d)\n", __func__, (int) inp.size(), max_tokens_list_size);
    return 1;
}

// 输出换行符
fprintf(stderr, "\n\n");

// 遍历输入的标记列表，输出每个标记对应的字符串
for (auto id : inp) {
    fprintf(stderr, "%s", llama_token_to_piece(ctx_tgt, id).c_str());
}

// 刷新标准错误流
fflush(stderr);

// 定义输入标记的数量
const int n_input = inp.size();

// 定义 t_enc_start 为当前时间的微秒数
const auto t_enc_start = ggml_time_us();
    // 使用两个模型对输入进行评估
    llama_decode(ctx_tgt, llama_batch_get_one( inp.data(), n_input - 1, 0, 0));
    llama_decode(ctx_tgt, llama_batch_get_one(&inp.back(), 1, n_input - 1, 0));
    llama_decode(ctx_dft, llama_batch_get_one( inp.data(), n_input, 0, 0);

    // 记录编码结束的时间
    const auto t_enc_end = ggml_time_us();

    // 两个模型应该具有相同的词汇表
    //GGML_ASSERT(n_vocab == llama_n_vocab(model_dft));

    // 每次草拟多少个标记
    int n_draft = params.n_draft;

    // 预测的标记数量
    int n_predict = 0;
    // 草拟的标记数量
    int n_drafted = 0;
    // 接受的标记数量
    int n_accept = 0;

    // 输入的过去长度（针对目标模型）
    int n_past_tgt = inp.size();
    // 输入的过去长度（针对默认模型）
    int n_past_dft = inp.size();
    // 用于确定生成结束的标志
    bool has_eos = false;

    // 目标模型的采样上下文
    struct llama_sampling_context * ctx_sampling = llama_sampling_init(params.sparams);

    // 草稿序列数据
    std::vector<seq_draft> drafts(n_seq_dft);

    // 清空目标模型的语法，草稿采样器将复制目标采样器的语法
    params.sparams.grammar.clear(); 
    // 强制使用贪婪采样和草稿模型的概率
    params.sparams.temp = -1.0f;    

    // 初始化每个草稿序列的采样上下文
    for (int s = 0; s < n_seq_dft; ++s) {
        drafts[s].ctx_sampling = llama_sampling_init(params.sparams);
    }

    // 初始化草稿和目标批次
    llama_batch batch_dft = llama_batch_init(params.n_ctx, 0, 1);
    llama_batch batch_tgt = llama_batch_init(params.n_ctx, 0, n_seq_dft);

    // 记录解码开始的时间
    const auto t_dec_start = ggml_time_us();
    // 从提示的最后一个标记中获取样本
    drafts[0].i_batch_tgt.resize(1); // 调整第一个草稿的目标批次大小为1
    drafts[0].i_batch_tgt[0] = 0; // 将第一个草稿的目标批次的第一个元素设置为0

    while (true) {
        // 打印当前草稿序列
        for (int s = 0; s < n_seq_dft; ++s) { // 遍历草稿序列
            if (!drafts[s].active) { // 如果草稿不活跃，则跳过
                continue;
            }

            const auto & tokens = drafts[s].tokens; // 获取草稿的标记

            LOG("draft %d: %s\n", s, LOG_TOKENS_TOSTR_PRETTY(ctx_dft, tokens).c_str()); // 打印草稿的序列
        }

        int i_dft  = 0; // 初始化草稿索引为0
        int s_keep = 0; // 初始化保留的草稿索引为0
// 进入循环，持续执行以下操作
while (true) {
    // 打印采样目标的信息，包括 s_keep、i_dft 和 drafts[s_keep].i_batch_tgt[i_dft] 的值
    LOG("sampling target: s_keep = %3d, i_dft = %3d, i_batch_tgt = %3d\n", s_keep, i_dft, drafts[s_keep].i_batch_tgt[i_dft]);

    // 从目标模型中进行采样
    llama_token id = llama_sampling_sample(ctx_sampling, ctx_tgt, NULL, drafts[s_keep].i_batch_tgt[i_dft]);

    // 接受采样结果
    llama_sampling_accept(ctx_sampling, ctx_tgt, id, true);

    // 将采样结果转换为字符串
    const std::string token_str = llama_token_to_piece(ctx_tgt, id);

    // 打印字符串
    printf("%s", token_str.c_str());
    // 刷新标准输出缓冲区
    fflush(stdout);

    // 如果采样结果为目标模型的结束符
    if (id == llama_token_eos(model_tgt)) {
        // 标记已经出现结束符
        has_eos = true;
    }

    // 增加预测次数
    ++n_predict;
}
// 检查目标令牌是否与任何草稿匹配
{
    // 初始化匹配标志为假
    bool matches = false;

    // 遍历所有草稿序列
    for (int s = 0; s < n_seq_dft; ++s) {
        // 如果草稿不活跃，则跳过
        if (!drafts[s].active) {
            continue;
        }

        // 如果目标令牌与草稿中的令牌匹配，则记录匹配信息
        if (i_dft < (int) drafts[s].tokens.size() && id == drafts[s].tokens[i_dft]) {
            LOG("the sampled target token matches the %dth drafted token of sequence %d (%d, '%s') - accepted\n", i_dft, s, id, token_str.c_str());

            // 保留匹配的草稿序列索引
            s_keep = s;
            // 设置匹配标志为真
            matches = true;
        } else {
            // 如果不匹配，则将草稿标记为不活跃
            drafts[s].active = false;
        }
    }
                // 如果匹配成功，则增加计数器并继续下一次循环
                if (matches) {
                    ++n_accept;
                    ++n_past_tgt;
                    ++n_past_dft;
                    ++i_dft;

                    continue;
                }
            }

            // 如果目标标记没有匹配或者我们用完了草稿标记，则记录日志
            LOG("the sampled target token (%d, '%s') did not match, or we ran out of drafted tokens\n", id, token_str.c_str());

            // TODO: 简化
            {
                // 记录日志，保留序列并更新计数器
                LOG("keeping sequence %d, n_past_tgt = %d, n_past_dft = %d\n", s_keep, n_past_tgt, n_past_dft);

                // 保留序列
                llama_kv_cache_seq_keep(ctx_dft, s_keep);
                // 复制序列
                llama_kv_cache_seq_cp  (ctx_dft, s_keep, 0, -1, -1);
                // 保留序列
                llama_kv_cache_seq_keep(ctx_dft, 0);
// 清除缓存中指定序列的键值对
llama_kv_cache_seq_rm(ctx_tgt, s_keep, n_past_tgt, -1);
// 将指定序列的键值对保留在缓存中
llama_kv_cache_seq_keep(ctx_tgt, s_keep);
// 将指定序列的键值对复制到另一个序列中
llama_kv_cache_seq_cp(ctx_tgt, s_keep, 0, -1, -1);
// 将指定序列的键值对保留在缓存中
llama_kv_cache_seq_keep(ctx_tgt, 0);

// 将草稿数组中的所有草稿设为非活跃状态
for (int s = 0; s < n_seq_dft; ++s) {
    drafts[s].active = false;
    // 清空草稿中的标记
    drafts[s].tokens.clear();
    // 清空草稿中的目标批次
    drafts[s].i_batch_tgt.clear();
}
// 将指定标识添加到第一个草稿的标记中
drafts[0].tokens.push_back(id);
// 将指定标识添加到第一个草稿的目标批次中
drafts[0].i_batch_tgt.push_back(0);

// 清空默认批次
llama_batch_clear(batch_dft);
// 向默认批次中添加指定标识和过去的数量
llama_batch_add(batch_dft, id, n_past_dft, { 0 }, true);

// 清除默认序列中指定范围的键值对
llama_kv_cache_seq_rm(ctx_dft, 0, n_past_dft, -1);
// 打印默认批次的信息
// LOG("dft batch: %s\n", LOG_BATCH_TOSTR_PRETTY(ctx_dft, batch_dft).c_str());
# 调用 llama_decode 函数，传入 ctx_dft 和 batch_dft 作为参数
llama_decode(ctx_dft, batch_dft);

# n_past_dft 自增1
++n_past_dft;

# 跳出循环
break;

# 如果 n_predict 大于 params.n_predict 或者 has_eos 为真，则跳出循环
if (n_predict > params.n_predict || has_eos) {
    break;
}

# 调用 llama_sampling_cp 函数，传入 ctx_sampling 和 drafts[0].ctx_sampling 作为参数
llama_sampling_cp(ctx_sampling, drafts[0].ctx_sampling);

# 初始化 n_seq_cur 为1
int n_seq_cur  = 1;
# 初始化 n_past_cur 为 n_past_dft
int n_past_cur = n_past_dft;

# 循环遍历 n_seq_dft 次
for (int s = 0; s < n_seq_dft; ++s) {
    # 将 drafts[s].active 设置为 false
    drafts[s].active   = false;
    # 将 drafts[s].drafting 设置为 false
    drafts[s].drafting = false;
}
        // 设置第一个草稿为激活状态
        drafts[0].active      = true;
        // 设置第一个草稿为草拟状态
        drafts[0].drafting    = true;
        // 设置第一个草稿的批次为0
        drafts[0].i_batch_dft = 0;

        // 清空目标批次的LLAMA批次
        llama_batch_clear(batch_tgt);
        // 向目标批次的LLAMA批次中添加草稿的第一个标记
        llama_batch_add  (batch_tgt, drafts[0].tokens[0], n_past_tgt, { 0 }, true);

        // 使用基于树的采样从草稿模型中对n_draft个标记进行采样
        for (int i = 0; i < n_draft; ++i) {
            // 重置草稿批次的标记数为0
            batch_dft.n_tokens = 0;

            // 将所有草稿的跳过标记重置为false
            for (int s = 0; s < n_seq_dft; ++s) {
                drafts[s].skip = false;
            }

            // 遍历所有草稿
            for (int s = 0; s < n_seq_dft; ++s) {
                // 如果草稿不在草拟状态或者被跳过，则继续下一个循环
                if (!drafts[s].drafting || drafts[s].skip) {
                    continue;
                }
// 调用 llama_sampling_sample 函数，对草稿进行采样
llama_sampling_sample(drafts[s].ctx_sampling, ctx_dft, NULL, drafts[s].i_batch_dft);

// 获取当前采样结果
const auto & cur_p = drafts[s].ctx_sampling->cur;

// 遍历当前采样结果，输出候选草稿信息
for (int k = 0; k < std::min(n_seq_dft + 3, (int) cur_p.size()); ++k) {
    LOG(" - draft candidate %3d for seq %3d, pos %3d: %6d (%8.3f) '%s'\n",
            k, s, i, cur_p[k].id, cur_p[k].p, llama_token_to_piece(ctx_dft, cur_p[k].id).c_str());
}

// 如果当前采样结果的概率小于接受概率，停止草稿
if (cur_p[0].p < p_accept) {
    LOG("stopping drafting for seq %3d, probability too low: %.3f < %.3f\n", s, cur_p[0].p, p_accept);
    drafts[s].drafting = false;
    continue;
}

// 创建包含当前序列的向量
std::vector<int> sa(1, s);

// 如果当前序列数量小于目标数量，并且当前概率高于分裂概率，尝试分裂分支
for (int f = 1; f < 8; ++f) {
    if (n_seq_cur < n_seq_dft && cur_p[f].p > p_split) {
// 打印日志，表示将序列 s 拆分成 n_seq_cur 个序列
LOG("splitting seq %3d into %3d\n", s, n_seq_cur);

// 从缓存中删除序列 n_seq_cur
llama_kv_cache_seq_rm(ctx_dft, n_seq_cur, -1, -1);
// 将序列 s 复制到新的序列 n_seq_cur
llama_kv_cache_seq_cp(ctx_dft, s, n_seq_cur, -1, -1);

// 将该分支中所有先前的标记现在也属于新的分支
for (int t = 0; t < batch_tgt.n_tokens; ++t) {
    for (int p = 0; p < batch_tgt.n_seq_id[t]; ++p) {
        if (batch_tgt.seq_id[t][p] == s) {
            batch_tgt.seq_id[t][batch_tgt.n_seq_id[t]] = n_seq_cur;
            batch_tgt.n_seq_id[t]++;
            break;
        }
    }
}

// 复制草稿状态
drafts[n_seq_cur].active   = true;
drafts[n_seq_cur].drafting = true;
drafts[n_seq_cur].skip     = true;
// 将当前草稿的标记复制到新的草稿中
drafts[n_seq_cur].tokens = drafts[s].tokens;
// 将当前草稿的批次信息复制到新的草稿中
drafts[n_seq_cur].i_batch_dft = drafts[s].i_batch_dft;
drafts[n_seq_cur].i_batch_tgt = drafts[s].i_batch_tgt;

// 复制当前草稿的上下文采样到新的草稿中
llama_sampling_cp(drafts[s].ctx_sampling, drafts[n_seq_cur].ctx_sampling);

// 将新的草稿索引添加到sa向量中
sa.push_back(n_seq_cur);

// 增加当前草稿的索引
n_seq_cur++;
// 如果当前草稿的标记已经用完，则跳出循环
} else {
    break;
}

// 为每个序列添加已选中的标记
for (int is = 0; is < (int) sa.size(); ++is) {
    // 获取当前序列的标记ID
    const llama_token id = cur_p[is].id;

    // 获取当前序列的索引
    const int s = sa[is];
// 调用llama_sampling_accept函数，将drafts[s].ctx_sampling、ctx_dft、id作为参数传入，最后一个参数为true
llama_sampling_accept(drafts[s].ctx_sampling, ctx_dft, id, true);

// 将id添加到drafts[s].tokens的末尾
drafts[s].tokens.push_back(id);

// 将唯一的被选中的token添加到目标批次中
drafts[s].i_batch_tgt.push_back(batch_tgt.n_tokens);

// 将token添加到批次中，用于使用draft模型进行批量解码
llama_batch_add(batch_tgt, id, n_past_tgt + i + 1, { s }, true);

// 将token添加到批次中，用于使用draft模型进行批量解码
drafts[s].i_batch_dft = batch_dft.n_tokens;
llama_batch_add(batch_dft, id, n_past_cur, { s }, true);

// 如果目标批次中的token数量大于n_draft，则将drafting设置为false
if (batch_tgt.n_tokens > n_draft) {
    drafts[s].drafting = false;
}
// 如果草稿中没有标记了，就结束循环
if (batch_dft.n_tokens == 0) {
    break;
}

// 在草稿模型上评估草稿的标记
llama_decode(ctx_dft, batch_dft);
++n_past_cur;
++n_drafted;

// 如果目标标记数大于草稿数，就结束循环
if (batch_tgt.n_tokens > n_draft) {
    break;
}
}

// 在草稿的标记上评估目标模型
{
    llama_kv_cache_seq_keep(ctx_tgt, 0);
    for (int s = 1; s < n_seq_dft; ++s) {
// 调用 llama_kv_cache_seq_cp 函数，将上下文 ctx_tgt 中的序列 s 复制到缓存中
llama_kv_cache_seq_cp(ctx_tgt, 0, s, -1, -1);

// 使用 LOG 函数打印目标批次的日志信息
// LOG("target batch: %s\n", LOG_BATCH_TOSTR_PRETTY(ctx_tgt, batch_tgt).c_str());

// 解码目标模型的批次数据
llama_decode(ctx_tgt, batch_tgt);
// 增加目标序列的数量
++n_past_tgt;

// 遍历草稿数组，删除每个草稿的第一个标记
for (int s = 0; s < n_seq_dft; ++s) {
    if (!drafts[s].active) {
        continue;
    }
    drafts[s].tokens.erase(drafts[s].tokens.begin());
}

// 记录解码结束的时间
auto t_dec_end = ggml_time_us();
# 打印换行符
LOG_TEE("\n\n");

# 打印编码的token数量、所花时间、速度
LOG_TEE("encoded %4d tokens in %8.3f seconds, speed: %8.3f t/s\n", n_input,   (t_enc_end - t_enc_start) / 1e6f, inp.size() / ((t_enc_end - t_enc_start) / 1e6f));
# 打印解码的token数量、所花时间、速度
LOG_TEE("decoded %4d tokens in %8.3f seconds, speed: %8.3f t/s\n", n_predict, (t_dec_end - t_dec_start) / 1e6f, n_predict  / ((t_dec_end - t_dec_start) / 1e6f));

# 打印换行符
LOG_TEE("\n");
# 打印n_draft的值
LOG_TEE("n_draft   = %d\n", n_draft);
# 打印n_predict的值
LOG_TEE("n_predict = %d\n", n_predict);
# 打印n_drafted的值
LOG_TEE("n_drafted = %d\n", n_drafted);
# 打印n_accept的值
LOG_TEE("n_accept  = %d\n", n_accept);
# 打印接受率
LOG_TEE("accept    = %.3f%%\n", 100.0f * n_accept / n_drafted);

# 打印换行符
LOG_TEE("\ndraft:\n");
# 打印ctx_dft的时间
llama_print_timings(ctx_dft);

# 打印换行符
LOG_TEE("\ntarget:\n");
# 打印ctx_tgt的时间
llama_print_timings(ctx_tgt);

# 释放ctx_sampling
llama_sampling_free(ctx_sampling);
# 遍历n_seq_dft，执行以下操作
for (int s = 0; s < n_seq_dft; ++s) {
    // 释放采样上下文
    llama_sampling_free(drafts[s].ctx_sampling);
    }

    // 释放批处理
    llama_batch_free(batch_dft);

    // 释放目标上下文
    llama_free(ctx_tgt);
    // 释放目标模型
    llama_free_model(model_tgt);

    // 释放草稿上下文
    llama_free(ctx_dft);
    // 释放草稿模型
    llama_free_model(model_dft);

    // 释放后端资源
    llama_backend_free();

    // 打印换行符
    fprintf(stderr, "\n\n");

    // 返回0表示正常结束
    return 0;
}
```