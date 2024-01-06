# `PowerInfer\examples\main\main.cpp`

```
// 包含自定义的头文件 common.h
#include "common.h"

// 包含自定义的控制台头文件 console.h
#include "console.h"

// 包含自定义的羊驼头文件 llama.h
#include "llama.h"

// 包含断言头文件
#include <cassert>

// 包含整数格式化头文件
#include <cinttypes>

// 包含数学函数头文件
#include <cmath>

// 包含 C 标准输入输出头文件
#include <cstdio>

// 包含 C 字符串操作头文件
#include <cstring>

// 包含 C 时间处理头文件
#include <ctime>

// 包含文件流处理头文件
#include <fstream>

// 包含输入输出流处理头文件
#include <iostream>

// 包含字符串流处理头文件
#include <sstream>

// 包含字符串处理头文件
#include <string>

// 包含向量容器头文件
#include <vector>

// 如果是 Unix 系统或者苹果系统，并且是 Mach 内核
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
// 包含信号处理头文件
#include <signal.h>
// 包含进程控制头文件
#include <unistd.h>
// 如果操作系统为 Windows，则定义 WIN32_LEAN_AND_MEAN，并且禁止包含 NOMINMAX
#if defined (_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <windows.h>
#include <signal.h>
#endif

// 如果使用的是 Microsoft 编译器，则禁止警告 4244 和 4267，这两个警告表示可能会丢失数据
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义全局变量
static llama_context           ** g_ctx; // 指向 llama_context 指针的指针
static llama_model             ** g_model; // 指向 llama_model 指针的指针
static gpt_params               * g_params; // 指向 gpt_params 结构体的指针
static std::vector<llama_token> * g_input_tokens; // 指向存储 llama_token 对象的向量的指针
static std::ostringstream       * g_output_ss; // 指向 ostringstream 对象的指针
static std::vector<llama_token> * g_output_tokens; // 指向存储 llama_token 对象的向量的指针
static bool is_interacting = false; // 布尔类型的全局变量，表示是否正在交互
# 写日志文件的函数，记录模型推理过程中的信息
static void write_logfile(
    # 上下文信息，参数，模型，输入token，输出文件名，输出token
    const llama_context * ctx, const gpt_params & params, const llama_model * model,
    const std::vector<llama_token> & input_tokens, const std::string & output,
    const std::vector<llama_token> & output_tokens
) {
    # 如果日志目录为空，则直接返回，不进行日志记录
    if (params.logdir.empty()) {
        return;
    }

    # 获取可排序的时间戳
    const std::string timestamp = get_sortable_timestamp();

    # 创建日志目录及其父目录，返回创建结果
    const bool success = create_directory_with_parents(params.logdir);
    # 如果创建失败，则在标准错误流中输出警告信息，并返回
    if (!success) {
        fprintf(stderr, "%s: warning: failed to create logdir %s, cannot write logfile\n",
                __func__, params.logdir.c_str());
        return;
    }
}
    # 构建日志文件路径，使用参数中的日志目录和时间戳拼接而成
    const std::string logfile_path = params.logdir + timestamp + ".yml";
    # 打开日志文件，以写入模式
    FILE * logfile = fopen(logfile_path.c_str(), "w");

    # 如果无法打开日志文件，输出错误信息并返回
    if (logfile == NULL) {
        fprintf(stderr, "%s: failed to open logfile %s\n", __func__, logfile_path.c_str());
        return;
    }

    # 在日志文件中写入指定内容
    fprintf(logfile, "binary: main");
    # 创建一个用于存储模型描述的字符数组
    char model_desc[128];
    # 获取模型描述信息并存储到 model_desc 中
    llama_model_desc(model, model_desc, sizeof(model_desc));
    # 将非结果信息以 YAML 格式写入日志文件
    dump_non_result_info_yaml(logfile, params, ctx, timestamp, input_tokens, model_desc);

    # 在日志文件中写入分隔符和标题
    fprintf(logfile, "\n");
    fprintf(logfile, "######################\n");
    fprintf(logfile, "# Generation Results #\n");
    fprintf(logfile, "######################\n");
    fprintf(logfile, "\n");

    # 将输出内容以多行的 YAML 格式写入日志文件
    dump_string_yaml_multiline(logfile, "output", output.c_str());
# 将整型向量以YAML格式写入日志文件
dump_vector_int_yaml(logfile, "output_tokens", output_tokens);

# 将LLAMA的时间信息以YAML格式写入日志文件
llama_dump_timing_info_yaml(logfile, ctx);

# 关闭日志文件
fclose(logfile);
}

# 如果是Unix、苹果操作系统或者Windows系统，定义信号处理函数
static void sigint_handler(int signo) {
    # 如果收到中断信号
    if (signo == SIGINT) {
        # 如果当前不是交互状态
        if (!is_interacting) {
            # 设置为交互状态
            is_interacting = true;
        } else {
            # 清理控制台
            console::cleanup();
            # 打印换行符
            printf("\n");
            # 打印LLAMA的时间信息
            llama_print_timings(*g_ctx);
            # 将日志写入文件
            write_logfile(*g_ctx, *g_params, *g_model, *g_input_tokens, g_output_ss->str(), *g_output_tokens);
            # 退出程序
            _exit(130);
        }
    }
}
#endif
// 结束条件，用于结束条件编译指令

int main(int argc, char ** argv) {
    gpt_params params;
    g_params = &params;

    if (!gpt_params_parse(argc, argv, params)) {
        return 1;
    }
    llama_sampling_params & sparams = params.sparams;

#ifndef LOG_DISABLE_LOGS
    // 如果未定义LOG_DISABLE_LOGS，则设置日志输出目标为特定文件
    log_set_target(log_filename_generator("main", "log"));
    // 输出日志信息
    LOG_TEE("Log start\n");
    // 输出命令行参数
    log_dump_cmdline(argc, argv);
#endif // LOG_DISABLE_LOGS

    // TODO: Dump params ?
    // 输出参数的困惑度
    //LOG("Params perplexity: %s\n", LOG_TOSTR(params.perplexity));
```

// 保存使用颜色的选择，以备后续使用
// （后续注意：这是一个稍微尴尬的选择）
console::init(params.simple_io, params.use_color);
// 在程序退出时执行清理操作
atexit([]() { console::cleanup(); });

// 如果参数中包含logits_all，则输出提示信息并返回0
if (params.logits_all) {
    printf("\n************\n");
    printf("%s: please use the 'perplexity' tool for perplexity calculations\n", __func__);
    printf("************\n\n");

    return 0;
}

// 如果参数中包含embedding，则输出提示信息并返回0
if (params.embedding) {
    printf("\n************\n");
    printf("%s: please use the 'embedding' tool for embedding calculations\n", __func__);
    printf("************\n\n");

    return 0;
}
# 如果上下文大小不为0且小于8，则打印警告信息并将上下文大小设置为8
if (params.n_ctx != 0 && params.n_ctx < 8) {
    LOG_TEE("%s: warning: minimum context size is 8, using minimum size.\n", __func__);
    params.n_ctx = 8;
}

# 如果RoPE频率基数不为0，则打印警告信息并改变RoPE频率基数为给定值
if (params.rope_freq_base != 0.0) {
    LOG_TEE("%s: warning: changing RoPE frequency base to %g.\n", __func__, params.rope_freq_base);
}

# 如果RoPE频率缩放不为0，则打印警告信息并缩放RoPE频率
if (params.rope_freq_scale != 0.0) {
    LOG_TEE("%s: warning: scaling RoPE frequency by %g.\n", __func__, params.rope_freq_scale);
}

# 打印构建信息和构建工具信息
LOG_TEE("%s: build = %d (%s)\n",      __func__, LLAMA_BUILD_NUMBER, LLAMA_COMMIT);
LOG_TEE("%s: built with %s for %s\n", __func__, LLAMA_COMPILER, LLAMA_BUILD_TARGET);

# 如果种子值为默认值，则将种子值设置为当前时间
if (params.seed == LLAMA_DEFAULT_SEED) {
    params.seed = time(NULL);
}
    // 打印日志，输出函数名和种子值
    LOG_TEE("%s: seed  = %u\n", __func__, params.seed);

    // 使用种子值初始化伪随机数生成器
    std::mt19937 rng(params.seed);
    
    // 如果需要随机提示，使用随机数生成器生成随机提示
    if (params.random_prompt) {
        params.prompt = gpt_random_prompt(rng);
    }

    // 打印日志，输出函数名
    LOG("%s: llama backend init\n", __func__);
    
    // 初始化 llama 后端
    llama_backend_init(params.numa);

    llama_model * model;
    llama_context * ctx;
    llama_context * ctx_guidance = NULL;
    g_model = &model;
    g_ctx = &ctx;

    // 加载模型并应用 lora 适配器（如果有的话）
    LOG("%s: load the model and apply lora adapter, if any\n", __func__);
    std::tie(model, ctx) = llama_init_from_gpt_params(params);
    // 如果参数中的 cfg_scale 大于 1，则根据参数创建 llama_context_params 结构体，并使用该参数创建新的上下文对象 ctx_guidance
    if (sparams.cfg_scale > 1.f) {
        struct llama_context_params lparams = llama_context_params_from_gpt_params(params);
        ctx_guidance = llama_new_context_with_model(model, lparams);
    }

    // 如果模型为空，则记录错误信息并返回 1
    if (model == NULL) {
        LOG_TEE("%s: error: unable to load model\n", __func__);
        return 1;
    }

    // 获取模型训练时的上下文数量和当前上下文的数量，并记录当前上下文的数量
    const int n_ctx_train = llama_n_ctx_train(model);
    const int n_ctx = llama_n_ctx(ctx);
    LOG("n_ctx: %d\n", n_ctx);

    // 如果当前上下文的数量大于模型训练时的上下文数量，则记录警告信息
    if (n_ctx > n_ctx_train) {
        LOG_TEE("%s: warning: model was trained on only %d context tokens (%d specified)\n",
                __func__, n_ctx_train, n_ctx);
    }

    // 打印系统信息
    {
        // 输出一个换行符
        LOG_TEE("\n");
        // 输出系统信息
        LOG_TEE("%s\n", get_system_info(params).c_str());
    }

    // 设置会话路径和会话令牌
    std::string path_session = params.path_prompt_cache;
    std::vector<llama_token> session_tokens;

    // 如果会话路径不为空
    if (!path_session.empty()) {
        // 输出尝试从指定路径加载保存的会话信息
        LOG_TEE("%s: attempting to load saved session from '%s'\n", __func__, path_session.c_str());

        // 打开文件以检查是否存在会话文件
        FILE * fp = std::fopen(path_session.c_str(), "rb");
        // 如果文件存在
        if (fp != NULL) {
            // 关闭文件
            std::fclose(fp);

            // 调整会话令牌的大小
            session_tokens.resize(n_ctx);
            size_t n_token_count_out = 0;
            // 如果加载会话文件失败
            if (!llama_load_session_file(ctx, path_session.c_str(), session_tokens.data(), session_tokens.capacity(), &n_token_count_out)) {
                // 输出加载会话文件失败的错误信息
                LOG_TEE("%s: error: failed to load session file '%s'\n", __func__, path_session.c_str());
    // 返回值为1
    return 1;
}

// 调整会话令牌的大小
session_tokens.resize(n_token_count_out);
// 设置随机数生成器的种子
llama_set_rng_seed(ctx, params.seed);

// 记录加载的会话的提示大小
LOG_TEE("%s: loaded a session with prompt size of %d tokens\n", __func__, (int) session_tokens.size());
} else {
    // 如果会话文件不存在，则创建会话
    LOG_TEE("%s: session file does not exist, will create\n", __func__);
}

// 检查是否需要添加开始符号
const bool add_bos = llama_vocab_type(model) == LLAMA_VOCAB_TYPE_SPM;
LOG("add_bos: %d\n", add_bos);

// 创建一个存储嵌入输入的向量
std::vector<llama_token> embd_inp;

// 如果需要交互、指示或者提示为空或者会话令牌为空，则进行分词处理
if (params.interactive_first || params.instruct || !params.prompt.empty() || session_tokens.empty()) {
    LOG("tokenize the prompt\n");
    embd_inp = ::llama_tokenize(ctx, params.prompt, add_bos, true);
} else {
// 输出日志信息，表示使用会话令牌
LOG("use session tokens\n");
// 将会话令牌赋值给embd_inp变量
embd_inp = session_tokens;
}

// 输出日志信息，表示打印提示信息
LOG("prompt: \"%s\"\n", log_tostr(params.prompt));
// 输出日志信息，表示打印tokens
LOG("tokens: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_inp).c_str());

// 如果embd_inp为空，则输出日志信息并添加bos令牌
// 不应该在没有任何令牌的情况下运行
if (embd_inp.empty()) {
    embd_inp.push_back(llama_token_bos(model));
    LOG("embd_inp was considered empty and bos was added: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_inp).c_str());
}

// 分词负面提示
// 定义一个名为guidance_inp的llama_token类型的向量
std::vector<llama_token> guidance_inp;
// 定义一个名为guidance_offset的整数变量并初始化为0
int guidance_offset = 0;
// 定义一个名为original_prompt_len的整数变量并初始化为0
int original_prompt_len = 0;
// 如果ctx_guidance存在，则输出日志信息并打印cfg_negative_prompt
if (ctx_guidance) {
    LOG("cfg_negative_prompt: \"%s\"\n", log_tostr(sparams.cfg_negative_prompt));
        // 使用 llama_tokenize 函数对 ctx_guidance 进行分词，使用 sparams.cfg_negative_prompt 作为负面提示，add_bos 为 true
        guidance_inp = ::llama_tokenize(ctx_guidance, sparams.cfg_negative_prompt, add_bos, true);
        // 打印 guidance_inp 的分词结果
        LOG("guidance_inp tokenized: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx_guidance, guidance_inp).c_str());

        // 使用 llama_tokenize 函数对 ctx 进行分词，使用 params.prompt 作为提示，add_bos 为 true
        std::vector<llama_token> original_inp = ::llama_tokenize(ctx, params.prompt, add_bos, true);
        // 打印 original_inp 的分词结果
        LOG("original_inp tokenized: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, original_inp).c_str());

        // 记录 original_inp 的长度
        original_prompt_len = original_inp.size();
        // 计算 guidance_inp 和 original_inp 长度的差值，作为偏移量
        guidance_offset = (int)guidance_inp.size() - original_prompt_len;
        // 打印 original_prompt_len 的值
        LOG("original_prompt_len: %s", log_tostr(original_prompt_len));
        // 打印 guidance_offset 的值
        LOG("guidance_offset:     %s", log_tostr(guidance_offset));
    }

    // 如果 embd_inp 的长度超过 n_ctx - 4，则打印错误信息并返回 1
    if ((int) embd_inp.size() > n_ctx - 4) {
        LOG_TEE("%s: error: prompt is too long (%d tokens, max %d)\n", __func__, (int) embd_inp.size(), n_ctx - 4);
        return 1;
    }

    // 如果 session_tokens 不为空，则记录匹配的会话令牌数量
    size_t n_matching_session_tokens = 0;
    if (!session_tokens.empty()) {
// 遍历会话令牌列表，查找匹配的会话令牌数量
for (llama_token id : session_tokens) {
    // 如果匹配的会话令牌数量已经达到输入的 embd_inp 大小，或者当前 id 不等于 embd_inp 中对应位置的值，则跳出循环
    if (n_matching_session_tokens >= embd_inp.size() || id != embd_inp[n_matching_session_tokens]) {
        break;
    }
    // 匹配的会话令牌数量加一
    n_matching_session_tokens++;
}

// 如果参数中的提示为空，并且匹配的会话令牌数量等于 embd_inp 的大小
if (params.prompt.empty() && n_matching_session_tokens == embd_inp.size()) {
    // 输出日志，表示使用来自会话文件的完整提示
    LOG_TEE("%s: using full prompt from session file\n", __func__);
} 
// 如果匹配的会话令牌数量大于等于 embd_inp 的大小
else if (n_matching_session_tokens >= embd_inp.size()) {
    // 输出日志，表示会话文件与提示完全匹配
    LOG_TEE("%s: session file has exact match for prompt!\n", __func__);
} 
// 如果匹配的会话令牌数量小于 embd_inp 大小的一半
else if (n_matching_session_tokens < (embd_inp.size() / 2)) {
    // 输出日志，表示会话文件与提示相似度较低，将会重新评估
    LOG_TEE("%s: warning: session file has low similarity to prompt (%zu / %zu tokens); will mostly be reevaluated\n",
        __func__, n_matching_session_tokens, embd_inp.size());
} 
// 其他情况
else {
    // 输出日志，表示会话文件与提示匹配了一部分令牌
    LOG_TEE("%s: session file matches %zu / %zu tokens of prompt\n",
        __func__, n_matching_session_tokens, embd_inp.size());
}

// 移除可能从上一个会话继承的“未来”令牌
llama_kv_cache_seq_rm(ctx, -1, n_matching_session_tokens, -1);
    }

    // 打印日志，重新计算缓存的logits（检查）：embd_inp是否为空，n_matching_session_tokens的大小，embd_inp的大小，session_tokens的大小，embd_inp的大小
    LOGLN(
            "recalculate the cached logits (check): embd_inp.empty() %s, n_matching_session_tokens %zu, embd_inp.size() %zu, session_tokens.size() %zu, embd_inp.size() %zu",
            log_tostr(embd_inp.empty()), n_matching_session_tokens, embd_inp.size(), session_tokens.size(), embd_inp.size());

    // 如果我们将使用缓存来存储完整的提示而不是达到缓存的末尾，强制重新评估最后一个标记，以重新计算缓存的logits
    if (!embd_inp.empty() && n_matching_session_tokens == embd_inp.size() && session_tokens.size() > embd_inp.size()) {
        // 打印日志，重新计算缓存的logits（执行）：session_tokens调整大小为embd_inp.size() - 1
        LOGLN("recalculate the cached logits (do): session_tokens.resize( %zu )", embd_inp.size() - 1);

        // 调整session_tokens的大小为embd_inp.size() - 1
        session_tokens.resize(embd_inp.size() - 1);
    }

    // 重置上下文时要保留的标记数
    if (params.n_keep < 0 || params.n_keep > (int) embd_inp.size() || params.instruct) {
        // 如果params.n_keep小于0或大于embd_inp的大小，或者params.instruct为真，则将params.n_keep设置为embd_inp的大小
        params.n_keep = (int)embd_inp.size();
    }

    // 指令模式的前缀和后缀
    // 使用 llama_tokenize 函数根据指定的分隔符对输入进行分词，返回结果存储在 inp_pfx 中
    const auto inp_pfx = ::llama_tokenize(ctx, "\n\n### Instruction:\n\n", add_bos, true);
    // 使用 llama_tokenize 函数根据指定的分隔符对输入进行分词，返回结果存储在 inp_sfx 中
    const auto inp_sfx = ::llama_tokenize(ctx, "\n\n### Response:\n\n",    false,   true);

    // 打印 inp_pfx 的内容
    LOG("inp_pfx: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, inp_pfx).c_str());
    // 打印 inp_sfx 的内容
    LOG("inp_sfx: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, inp_sfx).c_str());

    // 如果参数中包含指令模式，则将 interactive_first 设置为 true，并向 antiprompt 中添加指令前缀
    if (params.instruct) {
        params.interactive_first = true;
        params.antiprompt.push_back("### Instruction:\n\n");
    }

    // 如果 interactive_first 为 true，则将 interactive 设置为 true
    if (params.interactive_first) {
        params.interactive = true;
    }

    // 如果参数中包含 verbose_prompt，则打印提示信息
    if (params.verbose_prompt) {
        LOG_TEE("\n");
        LOG_TEE("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
// 打印提示中的令牌数量
LOG_TEE("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
// 遍历提示中的每个令牌，并打印其索引和对应的字符串值
for (int i = 0; i < (int) embd_inp.size(); i++) {
    LOG_TEE("%6d -> '%s'\n", embd_inp[i], llama_token_to_piece(ctx, embd_inp[i]).c_str());
}

// 如果存在上下文指导
if (ctx_guidance) {
    // 打印负面提示的内容
    LOG_TEE("\n");
    LOG_TEE("%s: negative prompt: '%s'\n", __func__, sparams.cfg_negative_prompt.c_str());
    // 打印负面提示中的令牌数量
    LOG_TEE("%s: number of tokens in negative prompt = %zu\n", __func__, guidance_inp.size());
    // 遍历负面提示中的每个令牌，并打印其索引和对应的字符串值
    for (int i = 0; i < (int) guidance_inp.size(); i++) {
        LOG_TEE("%6d -> '%s'\n", guidance_inp[i], llama_token_to_piece(ctx, guidance_inp[i]).c_str());
    }
}

// 如果需要保留的令牌数量大于0
if (params.n_keep > 0) {
    // 打印基于 n_keep 的静态提示
    LOG_TEE("%s: static prompt based on n_keep: '");
    // 遍历前 n_keep 个令牌，并打印其对应的字符串值
    for (int i = 0; i < params.n_keep; i++) {
        LOG_TEE("%s", llama_token_to_piece(ctx, embd_inp[i]).c_str());
    }
    LOG_TEE("'\n");
}
    }
    // 打印换行符
    LOG_TEE("\n");
}

// 如果参数中包含交互式标志
if (params.interactive) {
    // 如果是 UNIX 系统或者苹果系统
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
    // 设置 SIGINT 信号的处理函数为 sigint_handler
    struct sigaction sigint_action;
    sigint_action.sa_handler = sigint_handler;
    sigemptyset (&sigint_action.sa_mask);
    sigint_action.sa_flags = 0;
    sigaction(SIGINT, &sigint_action, NULL);
// 如果是 Windows 系统
#elif defined (_WIN32)
    // 设置控制台控制处理函数为 console_ctrl_handler
    auto console_ctrl_handler = +[](DWORD ctrl_type) -> BOOL {
        return (ctrl_type == CTRL_C_EVENT) ? (sigint_handler(SIGINT), true) : false;
    };
    SetConsoleCtrlHandler(reinterpret_cast<PHANDLER_ROUTINE>(console_ctrl_handler), true);
#endif

    // 打印交互模式开启的信息
    LOG_TEE("%s: interactive mode on.\n", __func__);
// 如果参数中的反向提示不为空
if (!params.antiprompt.empty()) {
    // 遍历每个反向提示
    for (const auto & antiprompt : params.antiprompt) {
        // 记录反向提示
        LOG_TEE("Reverse prompt: '%s'\n", antiprompt.c_str());
        // 如果参数中设置了详细提示信息
        if (params.verbose_prompt) {
            // 对反向提示进行分词
            auto tmp = ::llama_tokenize(ctx, antiprompt, false, true);
            // 遍历分词结果
            for (int i = 0; i < (int) tmp.size(); i++) {
                // 记录分词结果
                LOG_TEE("%6d -> '%s'\n", tmp[i], llama_token_to_piece(ctx, tmp[i]).c_str());
            }
        }
    }
}

// 如果参数中设置了输入前缀以BOS（Beginning of Sentence）开头
if (params.input_prefix_bos) {
    // 记录输入前缀包含BOS
    LOG_TEE("Input prefix with BOS\n");
}

// 如果参数中的输入前缀不为空
if (!params.input_prefix.empty()) {
    // 记录输入前缀
    LOG_TEE("Input prefix: '%s'\n", params.input_prefix.c_str());
    // 如果参数中设置了详细提示信息
    if (params.verbose_prompt) {
        // 对输入前缀进行分词
        auto tmp = ::llama_tokenize(ctx, params.input_prefix, true, true);
// 遍历 tmp 容器中的元素，打印每个元素及其对应的 llama_token_to_piece 函数返回值
for (int i = 0; i < (int) tmp.size(); i++) {
    LOG_TEE("%6d -> '%s'\n", tmp[i], llama_token_to_piece(ctx, tmp[i]).c_str());
}

// 如果输入后缀不为空，则打印输入后缀，并在参数 verbose_prompt 为真时，进行下一步操作
if (!params.input_suffix.empty()) {
    LOG_TEE("Input suffix: '%s'\n", params.input_suffix.c_str());
    if (params.verbose_prompt) {
        // 对输入后缀进行分词，并打印每个分词及其对应的 llama_token_to_piece 函数返回值
        auto tmp = ::llama_tokenize(ctx, params.input_suffix, false, true);
        for (int i = 0; i < (int) tmp.size(); i++) {
            LOG_TEE("%6d -> '%s'\n", tmp[i], llama_token_to_piece(ctx, tmp[i]).c_str());
        }
    }
}

// 打印采样结果
LOG_TEE("sampling: \n%s\n", llama_sampling_print(sparams).c_str());

// 打印生成参数的值
LOG_TEE("generate: n_ctx = %d, n_batch = %d, n_predict = %d, n_keep = %d\n", n_ctx, params.n_batch, params.n_predict, params.n_keep);

// 打印空行
LOG_TEE("\n\n");
    # 如果参数中包含交互模式
    if (params.interactive) {
        # 定义控制消息变量
        const char *control_message;
        # 如果参数中包含多行输入
        if (params.multiline_input) {
            # 设置控制消息为多行输入的提示信息
            control_message = " - To return control to LLaMa, end your input with '\\'.\n"
                              " - To return control without starting a new line, end your input with '/'.\n";
        } else {
            # 设置控制消息为单行输入的提示信息
            control_message = " - Press Return to return control to LLaMa.\n"
                              " - To return control without starting a new line, end your input with '/'.\n"
                              " - If you want to submit another line, end your input with '\\'.\n";
        }
        # 打印日志，表示正在交互模式下运行
        LOG_TEE("== Running in interactive mode. ==\n");
        # 如果是 Unix 系统、苹果系统或者 Windows 系统，打印相应的提示信息
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__)) || defined (_WIN32)
        LOG_TEE(       " - Press Ctrl+C to interject at any time.\n");
#endif
        # 打印控制消息
        LOG_TEE(       "%s\n", control_message);

        # 设置是否正在交互的标志为参数中的交互模式第一次交互
        is_interacting = params.interactive_first;
    }

    # 设置是否为反提示模式的标志为假
    bool is_antiprompt        = false;
    // 设置输入回显标志位为true
    bool input_echo = true;
    // 判断是否需要保存会话，条件是会话路径不为空并且匹配会话令牌的数量小于输入嵌入的大小
    bool need_to_save_session = !path_session.empty() && n_matching_session_tokens < embd_inp.size();

    // 初始化过去数量、剩余数量、已消耗数量、会话已消耗数量、过去指导数量
    int n_past = 0;
    int n_remain = params.n_predict;
    int n_consumed = 0;
    int n_session_consumed = 0;
    int n_past_guidance = 0;

    // 初始化输入令牌和输出令牌的向量
    std::vector<int> input_tokens;  g_input_tokens  = &input_tokens;
    std::vector<int> output_tokens; g_output_tokens = &output_tokens;
    // 初始化输出字符串流
    std::ostringstream output_ss;     g_output_ss     = &output_ss;

    // 设置显示颜色为提示
    console::set_display(console::prompt);

    // 初始化llama_token类型的向量embd和embd_guidance
    std::vector<llama_token> embd;
    std::vector<llama_token> embd_guidance;

    // 初始化llama_sampling_context结构体指针ctx_sampling
    struct llama_sampling_context * ctx_sampling = llama_sampling_init(sparams);
// 当剩余字符数不为零且不是反向提示，或者参数中包含交互模式时，执行循环
while ((n_remain != 0 && !is_antiprompt) || params.interactive) {
    // 预测
    if (!embd.empty()) {
        // 注意：这里的 n_ctx - 4 是为了匹配通过 --prompt 或 --file 处理命令行提示的逻辑，它们使用相同的值。
        int max_embd_size = n_ctx - 4;

        // 确保输入不超过上下文大小，必要时通过截断 embd 来实现
        if ((int) embd.size() > max_embd_size) {
            const int skipped_tokens = (int) embd.size() - max_embd_size;
            embd.resize(max_embd_size);

            console::set_display(console::error);
            printf("<<input too long: skipped %d token%s>>", skipped_tokens, skipped_tokens != 1 ? "s" : "");
            console::set_display(console::reset);
            fflush(stdout);
        }

        // 通过上下文交换实现无限文本生成
// 如果我们用尽了上下文：
// - 从原始提示中获取前 n_keep 个标记（通过 n_past）
// - 获取最后 (n_ctx - n_keep) 个标记的一半，并分批重新计算logits
if (n_past + (int) embd.size() + std::max<int>(0, guidance_offset) > n_ctx) {
    // 如果 n_predict 为 -2，则停止
    if (params.n_predict == -2) {
        LOG_TEE("\n\n%s: context full and n_predict == -%d => stopping\n", __func__, params.n_predict);
        break;
    }

    // 计算剩余标记数和需要丢弃的标记数
    const int n_left    = n_past - params.n_keep - 1;
    const int n_discard = n_left/2;

    // 输出日志
    LOG("context full, swapping: n_past = %d, n_left = %d, n_ctx = %d, n_keep = %d, n_discard = %d\n",
        n_past, n_left, n_ctx, params.n_keep, n_discard);

    // 从缓存中移除一定数量的标记
    llama_kv_cache_seq_rm   (ctx, 0, params.n_keep + 1            , params.n_keep + n_discard + 1);
    // 将剩余标记向前移动一定数量的位置
    llama_kv_cache_seq_shift(ctx, 0, params.n_keep + 1 + n_discard, n_past, -n_discard);

    // 更新 n_past 的值
    n_past -= n_discard;
}
                // 如果存在指导信息，则减去要丢弃的数量
                if (ctx_guidance) {
                    n_past_guidance -= n_discard;
                }

                // 打印日志，显示交换后的 n_past 和 n_past_guidance 的值
                LOG("after swap: n_past = %d, n_past_guidance = %d\n", n_past, n_past_guidance);

                // 打印日志，显示 embd 的内容
                LOG("embd: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd).c_str());

                // 打印日志，清空会话路径
                LOG("clear session path\n");
                path_session.clear();
            }

            // 尝试重用加载会话中匹配的前缀，而不是重新评估（通过 n_past）
            if (n_session_consumed < (int) session_tokens.size()) {
                size_t i = 0;
                for ( ; i < embd.size(); i++) {
                    // 如果 embd[i] 与 session_tokens[n_session_consumed] 不匹配，则调整 session_tokens 的大小并中断循环
                    if (embd[i] != session_tokens[n_session_consumed]) {
                        session_tokens.resize(n_session_consumed);
                        break;
                    }
                    // 增加过去的会话数量
                    n_past++;
                    // 增加已消耗的会话数量
                    n_session_consumed++;

                    // 如果已消耗的会话数量大于等于会话令牌的数量
                    if (n_session_consumed >= (int) session_tokens.size()) {
                        // 增加索引 i，并跳出循环
                        ++i;
                        break;
                    }
                }
                // 如果索引 i 大于 0
                if (i > 0) {
                    // 从 embd 中删除前 i 个元素
                    embd.erase(embd.begin(), embd.begin() + i);
                }
            }

            // 在批处理中评估令牌
            // embd 通常事先准备好以适应批处理，但并非总是如此
            if (ctx_guidance) {
                // 输入大小设为 0
                int input_size = 0;
                // 输入缓冲区设为 NULL
                llama_token * input_buf = NULL;
                // 检查过去的指导信息数量是否小于指导输入的大小
                if (n_past_guidance < (int) guidance_inp.size()) {
                    // 指导上下文应该具有相同的数据，但有以下修改：
                    //
                    // * 替换初始提示
                    // * 将所有内容移动 guidance_offset
                    embd_guidance = guidance_inp;
                    // 如果 embd 的起始位置加上原始提示的长度小于 embd 的结束位置
                    if (embd.begin() + original_prompt_len < embd.end()) {
                        // 在 embd_guidance 的末尾插入 embd 的起始位置加上原始提示的长度到 embd 的结束位置的数据
                        embd_guidance.insert(
                            embd_guidance.end(),
                            embd.begin() + original_prompt_len,
                            embd.end()
                        );
                    }

                    // 将 embd_guidance 的数据和大小赋值给输入缓冲区和输入大小
                    input_buf  = embd_guidance.data();
                    input_size = embd_guidance.size();

                    // 记录指导上下文的日志
                    LOG("guidance context: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_guidance).c_str());
                } else {
                    // 将 embd 的数据赋值给输入缓冲区
                    input_buf  = embd.data();
// 获取输入数据的大小
input_size = embd.size();

// 遍历输入数据，每次处理 params.n_batch 大小的数据
for (int i = 0; i < input_size; i += params.n_batch) {
    // 计算当前处理的数据大小，不超过 params.n_batch
    int n_eval = std::min(input_size - i, params.n_batch);
    // 调用 llama_batch_get_one 函数对输入数据进行处理
    if (llama_decode(ctx_guidance, llama_batch_get_one(input_buf + i, n_eval, n_past_guidance, 0))) {
        // 如果处理失败，输出错误信息并返回
        LOG_TEE("%s : failed to eval\n", __func__);
        return 1;
    }
    // 更新已处理的数据大小
    n_past_guidance += n_eval;
}

// 遍历 embd 的大小，每次处理 params.n_batch 大小的数据
for (int i = 0; i < (int) embd.size(); i += params.n_batch) {
    // 计算当前处理的数据大小
    int n_eval = (int) embd.size() - i;
    // 如果当前处理的数据大小超过 params.n_batch，则将其限制为 params.n_batch
    if (n_eval > params.n_batch) {
        n_eval = params.n_batch;
    }
}
// 打印评估的内容
LOG("eval: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd).c_str());

// 使用llama_decode函数对embd进行解码，并检查是否成功
if (llama_decode(ctx, llama_batch_get_one(&embd[i], n_eval, n_past, 0))) {
    // 如果解码失败，打印错误信息并返回1
    LOG_TEE("%s : failed to eval\n", __func__);
    return 1;
}

// 更新n_past的值
n_past += n_eval;

// 打印n_past的值
LOG("n_past = %d\n", n_past);

// 如果embd和path_session都不为空
if (!embd.empty() && !path_session.empty()) {
    // 将embd的内容添加到session_tokens中
    session_tokens.insert(session_tokens.end(), embd.begin(), embd.end());
    // 更新n_session_consumed的值为session_tokens的大小
    n_session_consumed = session_tokens.size();
}

// 清空embd和embd_guidance
embd.clear();
embd_guidance.clear();
        // 如果嵌入输入的大小小于等于已消耗的数量，并且不是交互式的
        if ((int) embd_inp.size() <= n_consumed && !is_interacting) {
            // 可选地在第一次采样时保存会话（以便下次更快地加载提示）
            if (!path_session.empty() && need_to_save_session && !params.prompt_cache_ro) {
                need_to_save_session = false;
                llama_save_session_file(ctx, path_session.c_str(), session_tokens.data(), session_tokens.size());

                LOG("saved session to %s\n", path_session.c_str());
            }

            // 从采样器中获取一个标识符
            const llama_token id = llama_sampling_sample(ctx_sampling, ctx, ctx_guidance);

            // 接受采样结果
            llama_sampling_accept(ctx_sampling, ctx, id, true);

            // 打印上一个标识符的信息
            LOG("last: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, ctx_sampling->prev).c_str());

            // 将标识符添加到嵌入列表中
            embd.push_back(id);

            // 将此输出到控制台
            input_echo = true;
// 减少剩余采样预算
--n_remain;

LOG("n_remain: %d\n", n_remain);
// 如果没有剩余的用户输入，将其转发到处理
LOG("embd_inp.size(): %d, n_consumed: %d\n", (int) embd_inp.size(), n_consumed);
while ((int) embd_inp.size() > n_consumed) {
    // 将用户输入添加到处理中
    embd.push_back(embd_inp[n_consumed]);

    // 将提示添加到采样上下文中，以便稍后应用重复惩罚
    // 对于提示，我们不应用语法规则
    llama_sampling_accept(ctx_sampling, ctx, embd_inp[n_consumed], false);

    ++n_consumed;
    if ((int) embd.size() >= params.n_batch) {
        break;
    }
}
        }

        // 显示文本
        if (input_echo) { // 如果需要回显输入
            for (auto id : embd) { // 遍历嵌入的标识符
                const std::string token_str = llama_token_to_piece(ctx, id); // 将标识符转换为字符串
                printf("%s", token_str.c_str()); // 打印字符串

                if (embd.size() > 1) { // 如果嵌入的标识符数量大于1
                    input_tokens.push_back(id); // 将标识符添加到输入标识符列表
                } else { // 如果嵌入的标识符数量为1
                    output_tokens.push_back(id); // 将标识符添加到输出标识符列表
                    output_ss << token_str; // 将字符串添加到输出流
                }
            }
            fflush(stdout); // 刷新标准输出流
        }
        // 如果没有待处理的用户输入，则将颜色重置为默认值
        if (input_echo && (int) embd_inp.size() == n_consumed) { // 如果需要回显输入且已经处理完所有输入
            console::set_display(console::reset); // 将控制台颜色重置为默认值
        }

        // 如果当前没有处理排队的输入；
        if ((int) embd_inp.size() <= n_consumed) {
            // 在最后的 n_prev 个标记中检查是否存在反向提示
            if (!params.antiprompt.empty()) {
                const int n_prev = 32;
                const std::string last_output = llama_sampling_prev_str(ctx_sampling, ctx, n_prev);

                is_antiprompt = false;
                // 检查每个反向提示是否出现在输出的末尾。
                // 如果我们不是交互式运行，反向提示可能会被标记化为一些后续字符
                // 因此我们通过稍微扩大搜索窗口来进行补偿。
                for (std::string & antiprompt : params.antiprompt) {
                    size_t extra_padding = params.interactive ? 0 : 2;
                    size_t search_start_pos = last_output.length() > static_cast<size_t>(antiprompt.length() + extra_padding)
                        ? last_output.length() - static_cast<size_t>(antiprompt.length() + extra_padding)
                        : 0;

                    if (last_output.find(antiprompt, search_start_pos) != std::string::npos) {
                // 如果参数中包含交互模式，设置交互标志位为真
                if (params.interactive) {
                    is_interacting = true;
                }
                // 设置反提示标志位为真
                is_antiprompt = true;
                // 跳出循环
                break;
            }
        }

        // 如果存在反提示，记录日志
        if (is_antiprompt) {
            LOG("found antiprompt: %s\n", last_output.c_str());
        }
    }

    // 处理交互模式下的文本结束标记
    if (llama_sampling_last(ctx_sampling) == llama_token_eos(model)) {
        // 记录日志，发现 EOS 标记
        LOG("found EOS token\n");

        // 如果参数中包含交互模式，并且反提示不为空
        if (params.interactive) {
            // 分词并注入第一个反向提示
// 从参数中获取第一个反提示，并将其转换为标记序列，插入到输入序列中
const auto first_antiprompt = ::llama_tokenize(ctx, params.antiprompt.front(), false, true);
embd_inp.insert(embd_inp.end(), first_antiprompt.begin(), first_antiprompt.end());
is_antiprompt = true;
// 设置正在交互的标志为真
is_interacting = true;
// 打印换行符
printf("\n");
// 如果参数中包含指令，则设置正在交互的标志为真
} else if (params.instruct) {
    is_interacting = true;
}

// 如果过去有交互记录，并且正在交互，则打印等待用户输入的日志
if (n_past > 0 && is_interacting) {
    LOG("waiting for user input\n");
    // 如果参数中包含指令，则打印提示符
    if (params.instruct) {
        printf("\n> ");
    }
    // 如果参数中包含输入前缀 BOS，则...
// 打印日志，表示正在添加输入前缀的 BOS 标记
LOG("adding input prefix BOS token\n");
// 将 BOS 标记添加到输入的嵌入向量中
embd_inp.push_back(llama_token_bos(model));

// 声明一个字符串变量 buffer
std::string buffer;
// 如果输入前缀不为空
if (!params.input_prefix.empty()) {
    // 打印日志，表示正在追加输入前缀
    LOG("appending input prefix: '%s'\n", params.input_prefix.c_str());
    // 打印输入前缀
    printf("%s", params.input_prefix.c_str());
}

// 设置控制台显示为用户输入模式
console::set_display(console::user_input);

// 声明一个字符串变量 line
std::string line;
// 声明一个布尔变量 another_line，并初始化为 true
bool another_line = true;
// 循环读取用户输入的每一行
do {
    // 读取用户输入的一行，并将结果存储到 line 中，同时返回是否还有另一行输入的布尔值
    another_line = console::readline(line, params.multiline_input);
    // 将读取的行添加到 buffer 中
    buffer += line;
} while (another_line);
                // 完成输入，重置颜色
                console::set_display(console::reset);

                // 只有在输入缓冲区非空的情况下才将标记添加到 embd
                // 输入空行会让用户跳过控制
                if (buffer.length() > 1) {
                    // 如果输入后缀不为空，则追加输入后缀
                    if (!params.input_suffix.empty()) {
                        LOG("appending input suffix: '%s'\n", params.input_suffix.c_str());
                        printf("%s", params.input_suffix.c_str());
                    }

                    LOG("buffer: '%s'\n", buffer.c_str());

                    const size_t original_size = embd_inp.size();

                    // 指令模式：插入指令前缀
                    if (params.instruct && !is_antiprompt) {
                        LOG("inserting instruction prefix\n");
                        n_consumed = embd_inp.size();
// 将输入前缀插入到嵌入输入中
embd_inp.insert(embd_inp.end(), inp_pfx.begin(), inp_pfx.end());

// 如果需要转义处理，则对缓冲区进行转义处理
if (params.escape) {
    process_escapes(buffer);
}

// 使用 llama_tokenize 函数对输入前缀、缓冲区和输入后缀进行分词处理
const auto line_pfx = ::llama_tokenize(ctx, params.input_prefix, false, true);
const auto line_inp = ::llama_tokenize(ctx, buffer, false, false);
const auto line_sfx = ::llama_tokenize(ctx, params.input_suffix, false, true);

// 打印输入的标记化结果
LOG("input tokens: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, line_inp).c_str());

// 将分词处理后的输入前缀、缓冲区和输入后缀插入到嵌入输入中
embd_inp.insert(embd_inp.end(), line_pfx.begin(), line_pfx.end());
embd_inp.insert(embd_inp.end(), line_inp.begin(), line_inp.end());
embd_inp.insert(embd_inp.end(), line_sfx.begin(), line_sfx.end());

// 如果是指令模式，则插入响应后缀
if (params.instruct) {
    LOG("inserting instruction suffix\n");
    embd_inp.insert(embd_inp.end(), inp_sfx.begin(), inp_sfx.end());
}
// 从原始大小开始遍历嵌入输入的元素
for (size_t i = original_size; i < embd_inp.size(); ++i) {
    // 获取嵌入输入中的令牌
    const llama_token token = embd_inp[i];
    // 将令牌添加到输出令牌列表中
    output_tokens.push_back(token);
    // 将令牌转换为字符串并添加到输出字符串流中
    output_ss << llama_token_to_piece(ctx, token);
}

// 减去输入行的大小
n_remain -= line_inp.size();
// 输出剩余数量
LOG("n_remain: %d\n", n_remain);
} else {
    // 如果是空行，则输出信息并将控制权传回
    LOG("empty line, passing control back\n");
}

// 不再回显此输入
input_echo = false;

// 如果过去存在，则重置采样上下文
if (n_past > 0) {
    if (is_interacting) {
        llama_sampling_reset(ctx_sampling);
    }
}
        // 设置交互标志为false
        is_interacting = false;
    }
}

// 文本标记结束
if (!embd.empty() && embd.back() == llama_token_eos(model) && !(params.instruct || params.interactive)) {
    LOG_TEE(" [end of text]\n");
    break;
}

// 在交互模式下，遵循最大标记数，并在达到时返回到用户输入。
// 当 n_predict == -1（无限）或 -2（停止在上下文大小）时，我们跳过此逻辑。
if (params.interactive && n_remain <= 0 && params.n_predict >= 0) {
    n_remain = params.n_predict;
    // 设置交互标志为true
    is_interacting = true;
}
}

// 如果会话路径不为空，并且 prompt_cache_all 为真，而 prompt_cache_ro 为假，则保存最终输出到会话文件
if (!path_session.empty() && params.prompt_cache_all && !params.prompt_cache_ro) {
    LOG_TEE("\n%s: saving final output to session file '%s'\n", __func__, path_session.c_str());
    // 保存会话文件
    llama_save_session_file(ctx, path_session.c_str(), session_tokens.data(), session_tokens.size());
    }

    // 打印时间信息
    llama_print_timings(ctx);
    // 写入日志文件
    write_logfile(ctx, params, model, input_tokens, output_ss.str(), output_tokens);

    // 释放指导上下文
    if (ctx_guidance) { llama_free(ctx_guidance); }
    // 释放上下文
    llama_free(ctx);
    // 释放模型
    llama_free_model(model);

    // 释放采样上下文
    llama_sampling_free(ctx_sampling);
    // 释放后端资源
    llama_backend_free();

    // 如果未禁用日志，则记录日志结束
    #ifndef LOG_DISABLE_LOGS
    LOG_TEE("Log end\n");
    #endif // LOG_DISABLE_LOGS

    // 返回 0 表示正常结束
    return 0;
}
```