# `PowerInfer\common\sampling.cpp`

```
// 包含 sampling.h 头文件

// 初始化采样上下文，根据参数创建一个新的采样上下文结构体
struct llama_sampling_context * llama_sampling_init(const struct llama_sampling_params & params) {
    // 为采样上下文结构体分配内存空间
    struct llama_sampling_context * result = new llama_sampling_context();

    // 将参数赋值给采样上下文的参数
    result->params  = params;
    // 将语法设置为 nullptr
    result->grammar = nullptr;

    // 如果存在语法，解析它
    if (!params.grammar.empty()) {
        // 解析语法并将结果赋值给采样上下文的解析后的语法
        result->parsed_grammar = grammar_parser::parse(params.grammar.c_str());

        // 如果解析出错，解析后的语法规则将为空
        if (result->parsed_grammar.rules.empty()) {
            // 打印错误信息
            fprintf(stderr, "%s: failed to parse grammar\n", __func__);
            // 返回空指针
            return nullptr;
        }

        // 将解析后的语法规则存储到一个向量中
        std::vector<const llama_grammar_element *> grammar_rules(result->parsed_grammar.c_rules());
// 初始化语法分析器，使用给定的语法规则和根符号
result->grammar = llama_grammar_init(
        grammar_rules.data(),
        grammar_rules.size(), result->parsed_grammar.symbol_ids.at("root"));
}

// 调整上下文中的先前结果的大小
result->prev.resize(params.n_prev);

// 返回结果
return result;
}

// 释放采样上下文中的语法对象
void llama_sampling_free(struct llama_sampling_context * ctx) {
    // 如果语法对象存在，则释放
    if (ctx->grammar != NULL) {
        llama_grammar_free(ctx->grammar);
    }

    // 释放上下文对象
    delete ctx;
}

// 重置采样上下文中的语法对象
void llama_sampling_reset(llama_sampling_context * ctx) {
    // 如果语法对象存在，则重置
    if (ctx->grammar != NULL) {
// 释放上下文中的语法规则
llama_grammar_free(ctx->grammar);
// 将上下文中的语法规则设置为 NULL
ctx->grammar = NULL;
}

// 如果解析后的语法规则不为空
if (!ctx->parsed_grammar.rules.empty()) {
    // 创建包含解析后语法规则的向量
    std::vector<const llama_grammar_element *> grammar_rules(ctx->parsed_grammar.c_rules());
    // 初始化上下文中的语法规则
    ctx->grammar = llama_grammar_init(
            grammar_rules.data(),
            grammar_rules.size(), ctx->parsed_grammar.symbol_ids.at("root"));
}

// 将上下文中的 prev 数组填充为 0
std::fill(ctx->prev.begin(), ctx->prev.end(), 0);
// 清空上下文中的 cur 容器
ctx->cur.clear();
}

// 复制采样上下文
void llama_sampling_cp(llama_sampling_context * src, llama_sampling_context * dst) {
    // 如果目标上下文中的语法规则不为空
    if (dst->grammar) {
        // 释放目标上下文中的语法规则
        llama_grammar_free(dst->grammar);
        // 将目标上下文中的语法规则设置为 nullptr
        dst->grammar = nullptr;
    }

    // 如果源上下文中存在语法信息，则复制到目标上下文中
    if (src->grammar) {
        dst->grammar = llama_grammar_copy(src->grammar);
    }

    // 将目标上下文的 prev 指针指向源上下文的 prev 指针所指向的位置
    dst->prev = src->prev;
}

// 返回上一个采样的标记
llama_token llama_sampling_last(llama_sampling_context * ctx) {
    return ctx->prev.back();
}

// 返回前 n 个采样的标记的字符串表示
std::string llama_sampling_prev_str(llama_sampling_context * ctx_sampling, llama_context * ctx_main, int n) {
    // 获取上一个采样的标记数量
    const int size = ctx_sampling->prev.size();

    // 取 n 和上一个采样的标记数量中的较小值
    n = std::min(n, size);

    // 初始化结果字符串
    std::string result;
// 从 size - n 到 size 的范围内遍历，将 llama_token_to_piece 函数返回的结果累加到 result 中
for (int i = size - n; i < size; i++) {
    result += llama_token_to_piece(ctx_main, ctx_sampling->prev[i]);
}

// 使用 snprintf 将参数 params 中的值格式化成字符串，存储到 result 中，最大长度为 1024
std::string llama_sampling_print(const llama_sampling_params & params) {
    char result[1024];

    snprintf(result, sizeof(result),
            "\trepeat_last_n = %d, repeat_penalty = %.3f, frequency_penalty = %.3f, presence_penalty = %.3f\n"
            "\ttop_k = %d, tfs_z = %.3f, top_p = %.3f, min_p = %.3f, typical_p = %.3f, temp = %.3f\n"
            "\tmirostat = %d, mirostat_lr = %.3f, mirostat_ent = %.3f",
            params.penalty_last_n, params.penalty_repeat, params.penalty_freq, params.penalty_present,
            params.top_k, params.tfs_z, params.top_p, params.min_p, params.typical_p, params.temp,
            params.mirostat, params.mirostat_eta, params.mirostat_tau);

    // 将 result 转换成 std::string 类型并返回
    return std::string(result);
}
# 定义一个函数 llama_sampling_sample，接受三个 llama_sampling_context 结构体指针参数和一个整型参数 idx
llama_token llama_sampling_sample(
                  struct llama_sampling_context * ctx_sampling,
                  struct llama_context * ctx_main,
                  struct llama_context * ctx_cfg,
                  const int idx) {
    # 从 ctx_sampling 中获取参数结构体 params
    const llama_sampling_params & params = ctx_sampling->params;

    # 从 ctx_main 中获取模型的词汇量
    const int n_vocab = llama_n_vocab(llama_get_model(ctx_main));

    # 从 params 中获取各种采样参数
    const float   temp            = params.temp;
    const int32_t top_k           = params.top_k <= 0 ? n_vocab : params.top_k;
    const float   top_p           = params.top_p;
    const float   min_p           = params.min_p;
    const float   tfs_z           = params.tfs_z;
    const float   typical_p       = params.typical_p;
    const int32_t penalty_last_n  = params.penalty_last_n < 0 ? params.n_prev : params.penalty_last_n;
    const float   penalty_repeat  = params.penalty_repeat;
    const float   penalty_freq    = params.penalty_freq;
    const float   penalty_present = params.penalty_present;
    // 定义并初始化一些常量
    const int     mirostat        = params.mirostat; // mirostat 参数
    const float   mirostat_tau    = params.mirostat_tau; // mirostat_tau 参数
    const float   mirostat_eta    = params.mirostat_eta; // mirostat_eta 参数
    const bool    penalize_nl     = params.penalize_nl; // penalize_nl 参数

    // 获取上下文采样的前一个和当前的上下文
    auto & prev = ctx_sampling->prev;
    auto & cur  = ctx_sampling->cur;

    // 初始化标识符
    llama_token id = 0;

    // 获取指向 logits 数组的指针
    float * logits = llama_get_logits_ith(ctx_main, idx);

    // 应用 params.logit_bias 映射
    for (auto it = params.logit_bias.begin(); it != params.logit_bias.end(); it++) {
        logits[it->first] += it->second;
    }

    // 清空当前上下文
    cur.clear();

    // 遍历词汇表中的标记
    for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
    // 将包含 token_id、logits[token_id]、0.0f 的 llama_token_data 结构添加到 cur 向量中
    cur.emplace_back(llama_token_data{token_id, logits[token_id], 0.0f});

    // 创建指向 cur 向量的指针，并设置其大小和是否拥有所有权
    llama_token_data_array cur_p = { cur.data(), cur.size(), false };

    // 如果存在上下文配置，则释放指导信息
    if (ctx_cfg) {
        llama_sample_classifier_free_guidance(ctx_main, &cur_p, ctx_cfg, params.cfg_scale);
    }

    // 应用惩罚
    if (!prev.empty()) {
        // 获取非线性 logit
        const float nl_logit = logits[llama_token_nl(llama_get_model(ctx_main))];

        // 对当前数据应用重复惩罚
        llama_sample_repetition_penalties(ctx_main, &cur_p,
                prev.data() + prev.size() - penalty_last_n,
                penalty_last_n, penalty_repeat, penalty_freq, penalty_present);

        // 如果不惩罚非线性 logit，则对当前数据进行遍历
        for (size_t idx = 0; idx < cur_p.size; idx++) {
            // 如果当前数据的 id 与非线性 logit 的 id 相同
            if (cur_p.data[idx].id == llama_token_nl(llama_get_model(ctx_main))) {
                    cur_p.data[idx].logit = nl_logit;
                    // 将 nl_logit 赋值给 cur_p.data[idx].logit
                    break;
                }
            }
        }
    }

    if (ctx_sampling->grammar != NULL) {
        // 如果存在语法规则，则使用 llama_sample_grammar 函数进行采样
        llama_sample_grammar(ctx_main, &cur_p, ctx_sampling->grammar);
    }

    if (temp < 0.0) {
        // 如果温度小于0，进行贪婪采样，使用 softmax 函数计算概率
        llama_sample_softmax(ctx_main, &cur_p);
        id = cur_p.data[0].id;
    } else if (temp == 0.0) {
        // 如果温度等于0，进行贪婪采样，不计算概率
        id = llama_sample_token_greedy(ctx_main, &cur_p);
    } else {
        if (mirostat == 1) {
            // 其他情况下，根据 mirostat 的值进行采样
// 定义常量 mirostat_m 为 100
const int mirostat_m = 100;
// 调用 llama_sample_temp 函数，传入参数 ctx_main, &cur_p, temp
llama_sample_temp(ctx_main, &cur_p, temp);
// 根据 mirostat_tau, mirostat_eta, mirostat_m, &ctx_sampling->mirostat_mu 调用 llama_sample_token_mirostat 函数，返回 id
id = llama_sample_token_mirostat(ctx_main, &cur_p, mirostat_tau, mirostat_eta, mirostat_m, &ctx_sampling->mirostat_mu);
} else if (mirostat == 2) {
    // 调用 llama_sample_temp 函数，传入参数 ctx_main, &cur_p, temp
    llama_sample_temp(ctx_main, &cur_p, temp);
    // 根据 mirostat_tau, mirostat_eta, &ctx_sampling->mirostat_mu 调用 llama_sample_token_mirostat_v2 函数，返回 id
    id = llama_sample_token_mirostat_v2(ctx_main, &cur_p, mirostat_tau, mirostat_eta, &ctx_sampling->mirostat_mu);
} else {
    // temperature sampling
    // 定义 min_keep 为 std::max(1, params.n_probs) 的最大值
    size_t min_keep = std::max(1, params.n_probs);
    // 调用 llama_sample_top_k 函数，传入参数 ctx_main, &cur_p, top_k, min_keep
    llama_sample_top_k    (ctx_main, &cur_p, top_k,     min_keep);
    // 调用 llama_sample_tail_free 函数，传入参数 ctx_main, &cur_p, tfs_z, min_keep
    llama_sample_tail_free(ctx_main, &cur_p, tfs_z,     min_keep);
    // 调用 llama_sample_typical 函数，传入参数 ctx_main, &cur_p, typical_p, min_keep
    llama_sample_typical  (ctx_main, &cur_p, typical_p, min_keep);
    // 调用 llama_sample_top_p 函数，传入参数 ctx_main, &cur_p, top_p, min_keep
    llama_sample_top_p    (ctx_main, &cur_p, top_p,     min_keep);
    // 调用 llama_sample_min_p 函数，传入参数 ctx_main, &cur_p, min_p, min_keep
    llama_sample_min_p    (ctx_main, &cur_p, min_p,     min_keep);
    // 调用 llama_sample_temp 函数，传入参数 ctx_main, &cur_p, temp
    llama_sample_temp     (ctx_main, &cur_p, temp);
    // 调用 llama_sample_token 函数，传入参数 ctx_main, &cur_p，返回 id
    id = llama_sample_token(ctx_main, &cur_p);
    //{
// 定义一个常量n_top，值为10
// 打印日志，显示前n_top个候选项
// 循环遍历前n_top个候选项
// 获取当前候选项的id
// 避免在禁用日志记录时出现未使用id的警告
// 打印日志，显示当前候选项的id、对应的字符串和概率
// 打印日志，显示抽样到的token的id和对应的字符串
// 结束循环

// llama_sampling_accept函数，接受抽样上下文和主上下文作为参数
// 接受一个llama_token类型的id和一个bool类型的apply_grammar参数
void update_context(Context* ctx_sampling, llama_token id, bool apply_grammar) {
    // 从ctx_sampling的prev容器中删除第一个元素
    ctx_sampling->prev.erase(ctx_sampling->prev.begin());
    // 将id添加到ctx_sampling的prev容器末尾
    ctx_sampling->prev.push_back(id);

    // 如果ctx_sampling的grammar不为空且apply_grammar为true
    if (ctx_sampling->grammar != NULL && apply_grammar) {
        // 调用llama_grammar_accept_token函数，传入ctx_main、ctx_sampling的grammar和id作为参数
        llama_grammar_accept_token(ctx_main, ctx_sampling->grammar, id);
    }
}
```