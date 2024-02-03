# `PowerInfer\common\sampling.cpp`

```cpp
// 初始化采样上下文，根据参数创建新的上下文对象
struct llama_sampling_context * llama_sampling_init(const struct llama_sampling_params & params) {
    // 分配内存创建新的上下文对象
    struct llama_sampling_context * result = new llama_sampling_context();

    // 将参数赋值给上下文对象的参数属性
    result->params  = params;
    // 将语法规则设置为空指针
    result->grammar = nullptr;

    // 如果存在语法规则，解析语法规则
    if (!params.grammar.empty()) {
        // 解析语法规则并将结果赋值给上下文对象的解析语法规则属性
        result->parsed_grammar = grammar_parser::parse(params.grammar.c_str());

        // 如果解析出错，打印错误信息并返回空指针
        if (result->parsed_grammar.rules.empty()) {
            fprintf(stderr, "%s: failed to parse grammar\n", __func__);
            return nullptr;
        }

        // 将解析后的语法规则转换为 llama_grammar_element 对象的向量
        std::vector<const llama_grammar_element *> grammar_rules(result->parsed_grammar.c_rules());

        // 根据解析后的语法规则初始化语法对象
        result->grammar = llama_grammar_init(
                grammar_rules.data(),
                grammar_rules.size(), result->parsed_grammar.symbol_ids.at("root"));
    }

    // 根据参数中的 n_prev 属性调整上下文对象的 prev 属性的大小
    result->prev.resize(params.n_prev);

    // 返回初始化后的上下文对象
    return result;
}

// 释放采样上下文对象的内存
void llama_sampling_free(struct llama_sampling_context * ctx) {
    // 如果语法对象存在，释放语法对象的内存
    if (ctx->grammar != NULL) {
        llama_grammar_free(ctx->grammar);
    }

    // 释放上下文对象的内存
    delete ctx;
}

// 重置采样上下文对象
void llama_sampling_reset(llama_sampling_context * ctx) {
    // 如果语法对象存在，释放语法对象的内存并将其设置为空指针
    if (ctx->grammar != NULL) {
        llama_grammar_free(ctx->grammar);
        ctx->grammar = NULL;
    }

    // 如果解析后的语法规则不为空，重新初始化语法对象
    if (!ctx->parsed_grammar.rules.empty()) {
        std::vector<const llama_grammar_element *> grammar_rules(ctx->parsed_grammar.c_rules());

        ctx->grammar = llama_grammar_init(
                grammar_rules.data(),
                grammar_rules.size(), ctx->parsed_grammar.symbol_ids.at("root"));
    }

    // 将 prev 属性的值填充为 0
    std::fill(ctx->prev.begin(), ctx->prev.end(), 0);
    // 清空 cur 属性的内容
    ctx->cur.clear();
}

// 复制源采样上下文对象的内容到目标采样上下文对象
void llama_sampling_cp(llama_sampling_context * src, llama_sampling_context * dst) {
    // 如果目标上下文对象的语法对象存在，释放其内存并将其设置为空指针
    if (dst->grammar) {
        llama_grammar_free(dst->grammar);
        dst->grammar = nullptr;
    }

    // 如果源上下文对象的语法对象存在，复制其内容到目标上下文对象的语法对象
    if (src->grammar) {
        dst->grammar = llama_grammar_copy(src->grammar);
    }
}
    # 将目标节点的前一个指针指向源节点的前一个指针所指向的位置
    dst->prev = src->prev;
}

// 返回上一个采样的 llamatoken
llama_token llama_sampling_last(llama_sampling_context * ctx) {
    return ctx->prev.back();
}

// 返回前 n 个采样的 llamatoken 组成的字符串
std::string llama_sampling_prev_str(llama_sampling_context * ctx_sampling, llama_context * ctx_main, int n) {
    const int size = ctx_sampling->prev.size();

    n = std::min(n, size);

    std::string result;

    for (int i = size - n; i < size; i++) {
        result += llama_token_to_piece(ctx_main, ctx_sampling->prev[i]);
    }

    return result;
}

// 返回采样参数的字符串表示
std::string llama_sampling_print(const llama_sampling_params & params) {
    char result[1024];

    snprintf(result, sizeof(result),
            "\trepeat_last_n = %d, repeat_penalty = %.3f, frequency_penalty = %.3f, presence_penalty = %.3f\n"
            "\ttop_k = %d, tfs_z = %.3f, top_p = %.3f, min_p = %.3f, typical_p = %.3f, temp = %.3f\n"
            "\tmirostat = %d, mirostat_lr = %.3f, mirostat_ent = %.3f",
            params.penalty_last_n, params.penalty_repeat, params.penalty_freq, params.penalty_present,
            params.top_k, params.tfs_z, params.top_p, params.min_p, params.typical_p, params.temp,
            params.mirostat, params.mirostat_eta, params.mirostat_tau);

    return std::string(result);
}

// 采样 llamatoken
llama_token llama_sampling_sample(
                  struct llama_sampling_context * ctx_sampling,
                  struct llama_context * ctx_main,
                  struct llama_context * ctx_cfg,
                  const int idx) {
    const llama_sampling_params & params = ctx_sampling->params;

    const int n_vocab = llama_n_vocab(llama_get_model(ctx_main));

    const float   temp            = params.temp;
    const int32_t top_k           = params.top_k <= 0 ? n_vocab : params.top_k;
    const float   top_p           = params.top_p;
    const float   min_p           = params.min_p;
    const float   tfs_z           = params.tfs_z;
    const float   typical_p       = params.typical_p;
    const int32_t penalty_last_n  = params.penalty_last_n < 0 ? params.n_prev : params.penalty_last_n;
    // 定义并初始化惩罚重复的惩罚值
    const float   penalty_repeat  = params.penalty_repeat;
    // 定义并初始化惩罚频率的惩罚值
    const float   penalty_freq    = params.penalty_freq;
    // 定义并初始化惩罚存在的惩罚值
    const float   penalty_present = params.penalty_present;
    // 定义并初始化mirostat
    const int     mirostat        = params.mirostat;
    // 定义并初始化mirostat_tau
    const float   mirostat_tau    = params.mirostat_tau;
    // 定义并初始化mirostat_eta
    const float   mirostat_eta    = params.mirostat_eta;
    // 定义并初始化是否对换行符进行惩罚
    const bool    penalize_nl     = params.penalize_nl;

    // 获取上下文采样的前一个和当前的引用
    auto & prev = ctx_sampling->prev;
    auto & cur  = ctx_sampling->cur;

    // 初始化id为0
    llama_token id = 0;

    // 获取logits指针
    float * logits = llama_get_logits_ith(ctx_main, idx);

    // 应用params.logit_bias映射
    for (auto it = params.logit_bias.begin(); it != params.logit_bias.end(); it++) {
        logits[it->first] += it->second;
    }

    // 清空cur
    cur.clear();

    // 将n_vocab个元素的llama_token_data添加到cur中
    for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
        cur.emplace_back(llama_token_data{token_id, logits[token_id], 0.0f});
    }

    // 创建llama_token_data_array结构体
    llama_token_data_array cur_p = { cur.data(), cur.size(), false };

    // 如果存在上下文配置，则使用llama_sample_classifier_free_guidance函数
    if (ctx_cfg) {
        llama_sample_classifier_free_guidance(ctx_main, &cur_p, ctx_cfg, params.cfg_scale);
    }

    // 应用惩罚
    if (!prev.empty()) {
        // 获取换行符的logit值
        const float nl_logit = logits[llama_token_nl(llama_get_model(ctx_main))];

        // 应用重复惩罚
        llama_sample_repetition_penalties(ctx_main, &cur_p,
                prev.data() + prev.size() - penalty_last_n,
                penalty_last_n, penalty_repeat, penalty_freq, penalty_present);

        // 如果不对换行符进行惩罚，则将logit值设置为nl_logit
        if (!penalize_nl) {
            for (size_t idx = 0; idx < cur_p.size; idx++) {
                if (cur_p.data[idx].id == llama_token_nl(llama_get_model(ctx_main))) {
                    cur_p.data[idx].logit = nl_logit;
                    break;
                }
            }
        }
    }

    // 如果存在采样的语法，则使用llama_sample_grammar函数
    if (ctx_sampling->grammar != NULL) {
        llama_sample_grammar(ctx_main, &cur_p, ctx_sampling->grammar);
    }

    // 如果temp小于0.0，则使用llama_sample_softmax函数进行贪婪采样，并获取id
    if (temp < 0.0) {
        llama_sample_softmax(ctx_main, &cur_p);
        id = cur_p.data[0].id;
    }
    } else if (temp == 0.0) {
        // 如果温度为0.0，则使用贪婪抽样，不需要概率
        id = llama_sample_token_greedy(ctx_main, &cur_p);
    } else {
        if (mirostat == 1) {
            // 如果mirostat为1，则设置mirostat_m为100
            const int mirostat_m = 100;
            // 使用温度进行抽样
            llama_sample_temp(ctx_main, &cur_p, temp);
            // 使用mirostat抽样token
            id = llama_sample_token_mirostat(ctx_main, &cur_p, mirostat_tau, mirostat_eta, mirostat_m, &ctx_sampling->mirostat_mu);
        } else if (mirostat == 2) {
            // 使用温度进行抽样
            llama_sample_temp(ctx_main, &cur_p, temp);
            // 使用mirostat_v2抽样token
            id = llama_sample_token_mirostat_v2(ctx_main, &cur_p, mirostat_tau, mirostat_eta, &ctx_sampling->mirostat_mu);
        } else {
            // 温度抽样
            size_t min_keep = std::max(1, params.n_probs);

            // 使用top_k进行抽样
            llama_sample_top_k    (ctx_main, &cur_p, top_k,     min_keep);
            // 使用tail_free进行抽样
            llama_sample_tail_free(ctx_main, &cur_p, tfs_z,     min_keep);
            // 使用typical进行抽样
            llama_sample_typical  (ctx_main, &cur_p, typical_p, min_keep);
            // 使用top_p进行抽样
            llama_sample_top_p    (ctx_main, &cur_p, top_p,     min_keep);
            // 使用min_p进行抽样
            llama_sample_min_p    (ctx_main, &cur_p, min_p,     min_keep);
            // 使用温度进行抽样
            llama_sample_temp     (ctx_main, &cur_p, temp);

            // 抽样token
            id = llama_sample_token(ctx_main, &cur_p);

            // 打印抽样结果
            LOG("sampled token: %5d: '%s'\n", id, llama_token_to_piece(ctx_main, id).c_str());
        }
    }

    // 返回抽样得到的token
    return id;
# 定义名为 llama_sampling_accept 的函数，接受采样上下文、主要上下文、标记ID和应用语法的布尔值作为参数
void llama_sampling_accept(
        struct llama_sampling_context * ctx_sampling,  # 采样上下文指针
        struct llama_context * ctx_main,  # 主要上下文指针
        llama_token id,  # 标记ID
        bool apply_grammar) {  # 是否应用语法的布尔值
    # 从采样上下文的前一个标记列表中删除第一个元素
    ctx_sampling->prev.erase(ctx_sampling->prev.begin());
    # 将标记ID添加到采样上下文的前一个标记列表末尾
    ctx_sampling->prev.push_back(id);

    # 如果采样上下文的语法不为空且应用语法为真
    if (ctx_sampling->grammar != NULL && apply_grammar) {
        # 调用函数 llama_grammar_accept_token，传入主要上下文、采样上下文的语法和标记ID作为参数
        llama_grammar_accept_token(ctx_main, ctx_sampling->grammar, id);
    }
}
```