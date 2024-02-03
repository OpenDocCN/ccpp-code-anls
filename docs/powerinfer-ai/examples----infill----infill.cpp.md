# `PowerInfer\examples\infill\infill.cpp`

```cpp
#include "common.h"  // 包含自定义的 common.h 头文件

#include "console.h"  // 包含自定义的 console.h 头文件
#include "llama.h"  // 包含自定义的 llama.h 头文件
#include "grammar-parser.h"  // 包含自定义的 grammar-parser.h 头文件

#include <cassert>  // 包含 C++ 标准库的 assert 头文件
#include <cinttypes>  // 包含 C++ 标准库的 cinttypes 头文件
#include <cmath>  // 包含 C++ 标准库的 cmath 头文件
#include <cstdio>  // 包含 C++ 标准库的 cstdio 头文件
#include <cstring>  // 包含 C++ 标准库的 cstring 头文件
#include <ctime>  // 包含 C++ 标准库的 ctime 头文件
#include <fstream>  // 包含 C++ 标准库的 fstream 头文件
#include <iostream>  // 包含 C++ 标准库的 iostream 头文件
#include <sstream>  // 包含 C++ 标准库的 sstream 头文件
#include <string>  // 包含 C++ 标准库的 string 头文件
#include <vector>  // 包含 C++ 标准库的 vector 头文件

#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
#include <signal.h>  // 如果是 Unix 系统，包含信号处理的头文件
#include <unistd.h>  // 如果是 Unix 系统，包含 Unix 标准库的 unistd 头文件
#elif defined (_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <windows.h>  // 如果是 Windows 系统，包含 Windows 头文件
#include <signal.h>  // 如果是 Windows 系统，包含信号处理的头文件
#endif

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止特定警告
#endif

static llama_context           ** g_ctx;  // 定义全局变量 g_ctx，指向 llama_context 指针的指针
static llama_model             ** g_model;  // 定义全局变量 g_model，指向 llama_model 指针的指针
static gpt_params               * g_params;  // 定义全局变量 g_params，指向 gpt_params 结构体的指针
static std::vector<llama_token> * g_input_tokens;  // 定义全局变量 g_input_tokens，指向存储 llama_token 的 vector 的指针
static std::ostringstream       * g_output_ss;  // 定义全局变量 g_output_ss，指向 ostringstream 对象的指针
static std::vector<llama_token> * g_output_tokens;  // 定义全局变量 g_output_tokens，指向存储 llama_token 的 vector 的指针

static bool is_interacting = false;  // 定义静态全局变量 is_interacting，并初始化为 false

static void write_logfile(  // 定义静态函数 write_logfile，用于写入日志文件
    const llama_context * ctx, const gpt_params & params, const llama_model * model,
    const std::vector<llama_token> & input_tokens, const std::string & output,
    const std::vector<llama_token> & output_tokens
) {
    if (params.logdir.empty()) {  // 如果日志目录为空，则返回
        return;
    }

    const std::string timestamp = get_sortable_timestamp();  // 获取可排序的时间戳

    const bool success = create_directory_with_parents(params.logdir);  // 创建日志目录及其父目录
    if (!success) {  // 如果创建失败，则输出警告信息并返回
        fprintf(stderr, "%s: warning: failed to create logdir %s, cannot write logfile\n",
                __func__, params.logdir.c_str());
        return;
    }

    const std::string logfile_path = params.logdir + timestamp + ".yml";  // 构建日志文件路径
    FILE * logfile = fopen(logfile_path.c_str(), "w");  // 打开日志文件

    if (logfile == NULL) {  // 如果打开失败，则输出错误信息并返回
        fprintf(stderr, "%s: failed to open logfile %s\n", __func__, logfile_path.c_str());
        return;
    }

    fprintf(logfile, "binary: infill\n");  // 写入日志文件的内容
    char model_desc[128];  // 定义存储模型描述的字符数组
    llama_model_desc(model, model_desc, sizeof(model_desc));  // 获取模型描述
    # 将非结果信息转储为 YAML 格式到日志文件中
    dump_non_result_info_yaml(logfile, params, ctx, timestamp, input_tokens, model_desc);
    
    # 在日志文件中输出空行
    fprintf(logfile, "\n");
    # 在日志文件中输出分隔符和生成结果标题
    fprintf(logfile, "######################\n");
    fprintf(logfile, "# Generation Results #\n");
    fprintf(logfile, "######################\n");
    fprintf(logfile, "\n");
    
    # 将输出内容以多行形式转储为 YAML 格式到日志文件中
    dump_string_yaml_multiline(logfile, "output", output.c_str());
    # 将输出标记以 YAML 格式转储到日志文件中
    dump_vector_int_yaml(logfile, "output_tokens", output_tokens);
    
    # 将程序运行时间信息以 YAML 格式转储到日志文件中
    llama_dump_timing_info_yaml(logfile, ctx);
    # 关闭日志文件
    fclose(logfile);
}

#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__)) || defined (_WIN32)
// 定义信号处理函数，处理 SIGINT 信号
static void sigint_handler(int signo) {
    // 如果收到 SIGINT 信号
    if (signo == SIGINT) {
        // 如果不是交互状态
        if (!is_interacting) {
            // 设置为交互状态
            is_interacting = true;
        } else {
            // 清理控制台
            console::cleanup();
            // 打印换行符
            printf("\n");
            // 打印时间信息
            llama_print_timings(*g_ctx);
            // 写入日志文件
            write_logfile(*g_ctx, *g_params, *g_model, *g_input_tokens, g_output_ss->str(), *g_output_tokens);
            // 退出程序
            _exit(130);
        }
    }
}
#endif

// 主函数
int main(int argc, char ** argv) {
    // 创建参数对象
    gpt_params params;
    // 获取采样参数的引用
    llama_sampling_params & sparams = params.sparams;
    // 设置全局参数指针
    g_params = &params;

    // 解析命令行参数，如果失败则返回 1
    if (!gpt_params_parse(argc, argv, params)) {
        return 1;
    }

    // 如果未禁用日志
#ifndef LOG_DISABLE_LOGS
    // 设置日志目标文件
    log_set_target(log_filename_generator("infill", "log"));
    // 打印日志开始信息
    LOG_TEE("Log start\n");
    // 输出命令行参数
    log_dump_cmdline(argc, argv);
#endif // LOG_DISABLE_LOGS

    // 初始化控制台
    console::init(params.simple_io, params.use_color);
    // 在程序退出时执行清理函数
    atexit([]() { console::cleanup(); });

    // 如果参数中包含 logits_all
    if (params.logits_all) {
        // 输出提示信息
        printf("\n************\n");
        printf("%s: please use the 'perplexity' tool for perplexity calculations\n", __func__);
        printf("************\n\n");

        return 0;
    }

    // 如果参数中包含 embedding
    if (params.embedding) {
        // 输出提示信息
        printf("\n************\n");
        printf("%s: please use the 'embedding' tool for embedding calculations\n", __func__);
        printf("************\n\n");

        return 0;
    }

    // 如果参数中的 n_ctx 不为 0 且小于 8
    if (params.n_ctx != 0 && params.n_ctx < 8) {
        // 输出警告信息
        LOG_TEE("%s: warning: minimum context size is 8, using minimum size.\n", __func__);
        // 将 n_ctx 设置为 8
        params.n_ctx = 8;
    }
    // 如果参数中包含 instruct
    if (params.instruct) {
        // 输出提示信息
        printf("\n************\n");
        printf("%s: please use the 'main' tool for instruct mode\n", __func__);
        printf("************\n\n");

        return 0;
    }
    // 如果参数中的 antiprompt 不为空，则输出提示信息并返回 0
    if (!params.antiprompt.empty()) {
        printf("\n************\n");
        printf("%s: please use the 'main' tool for antiprompt mode\n", __func__);
        printf("************\n\n");

        return 0;
    }
    // 如果不是交互式模式且输入前缀和后缀都为空，则输出提示信息并返回 0
    if (!params.interactive_first && (params.input_prefix.empty() && params.input_suffix.empty())) {
        printf("\n************\n");
        printf("%s: please use '--interactive_first' or specify '--in_prefix' and/or '--in_suffix'\n", __func__);
        printf("************\n\n");

        return 0;
    }
    // 如果参数中的 random_prompt 为真，则输出提示信息并返回 0
    if (params.random_prompt) {
        printf("\n************\n");
        printf("%s: please use the 'main' tool for random prompt mode\n", __func__);
        printf("************\n\n");

        return 0;
    }
    // 如果参数中的 path_prompt_cache 不为空，则输出提示信息并返回 0
    if (!params.path_prompt_cache.empty()) {
        printf("\n************\n");
        printf("%s: infill does not support prompt caching\n", __func__);
        printf("************\n\n");

        return 0;
    }

    // 如果参数中的 rope_freq_base 不为 0.0，则输出警告信息
    if (params.rope_freq_base != 0.0) {
        LOG_TEE("%s: warning: changing RoPE frequency base to %g.\n", __func__, params.rope_freq_base);
    }

    // 如果参数中的 rope_freq_scale 不为 0.0，则输出警告信息
    if (params.rope_freq_scale != 0.0) {
        LOG_TEE("%s: warning: scaling RoPE frequency by %g.\n", __func__, params.rope_freq_scale);
    }

    // 输出构建信息和编译器信息
    LOG_TEE("%s: build = %d (%s)\n",      __func__, LLAMA_BUILD_NUMBER, LLAMA_COMMIT);
    LOG_TEE("%s: built with %s for %s\n", __func__, LLAMA_COMPILER, LLAMA_BUILD_TARGET);

    // 如果参数中的种子为默认值，则将种子设置为当前时间
    if (params.seed == LLAMA_DEFAULT_SEED) {
        params.seed = time(NULL);
    }

    // 输出种子信息
    LOG_TEE("%s: seed  = %u\n", __func__, params.seed);

    // 使用种子初始化随机数生成器
    std::mt19937 rng(params.seed);

    // 输出 llama 后端初始化信息
    LOG("%s: llama backend init\n", __func__);
    llama_backend_init(params.numa);

    llama_model * model;
    llama_context * ctx;
    llama_context * ctx_guidance = NULL;
    g_model = &model;
    g_ctx = &ctx;

    // 加载模型并应用 lora 适配器（如果有）
    LOG("%s: load the model and apply lora adapter, if any\n", __func__);
    // 从 GPT 参数初始化 LLAMA 模型和上下文
    std::tie(model, ctx) = llama_init_from_gpt_params(params);
    // 如果配置比例大于1，则使用 GPT 参数创建 LLAMA 上下文参数
    if (sparams.cfg_scale > 1.f) {
        struct llama_context_params lparams = llama_context_params_from_gpt_params(params);
        ctx_guidance = llama_new_context_with_model(model, lparams);
    }

    // 如果模型为空，则记录错误信息并返回1
    if (model == NULL) {
        LOG_TEE("%s: error: unable to load model\n", __func__);
        return 1;
    }

    // 获取训练上下文的数量和当前上下文的数量，并记录当前上下文的数量
    const int n_ctx_train = llama_n_ctx_train(model);
    const int n_ctx = llama_n_ctx(ctx);
    LOG("n_ctx: %d\n", n_ctx);

    // 如果当前上下文数量大于训练上下文数量，则记录警告信息
    if (n_ctx > n_ctx_train) {
        LOG_TEE("%s: warning: model was trained on only %d context tokens (%d specified)\n",
                __func__, n_ctx_train, n_ctx);
    }

    // 打印系统信息
    {
        LOG_TEE("\n");
        LOG_TEE("%s\n", get_system_info(params).c_str());
    }
    // 判断是否需要添加 bos 标记，并记录结果
    const bool add_bos = llama_vocab_type(model) == LLAMA_VOCAB_TYPE_SPM;
    LOG("add_bos: %d\n", add_bos);

    // 判断是否需要移除输入后缀的空格，并进行相应处理
    bool suff_rm_leading_spc = params.escape;
    if (suff_rm_leading_spc && params.input_suffix.find_first_of(" ") == 0 && params.input_suffix.size() > 1) {
        params.input_suffix.erase(0, 1);
        suff_rm_leading_spc = false;
    }
    // 初始化 LLAMA 输入前缀和后缀
    std::vector<llama_token> embd_inp;
    std::vector<llama_token> inp_pfx = ::llama_tokenize(ctx, params.input_prefix, false);
    std::vector<llama_token> inp_sfx = ::llama_tokenize(ctx, params.input_suffix, false);
    const int space_token = 29871;
    // 如果需要移除后缀的空格，则进行相应处理
    if (suff_rm_leading_spc && inp_sfx[0] == space_token) {
        inp_sfx.erase(inp_sfx.begin());
    }
    // 插入 LLAMA 输入前缀和后缀的特殊标记
    inp_pfx.insert(inp_pfx.begin(), llama_token_prefix(model));
    if (add_bos) {
        inp_pfx.insert(inp_pfx.begin(), llama_token_bos(model));
    }
    inp_sfx.insert(inp_sfx.begin(), llama_token_suffix(model));
    // 组合 LLAMA 输入前缀和后缀，并添加中间标记
    embd_inp = inp_pfx;
    embd_inp.insert(embd_inp.end(), inp_sfx.begin(), inp_sfx.end());
    embd_inp.push_back(llama_token_middle(model));

    // 记录输入前缀
    LOG("prefix: \"%s\"\n", log_tostr(params.input_prefix));
    // 打印输入后缀
    LOG("suffix: \"%s\"\n", log_tostr(params.input_suffix));
    // 打印 tokens
    LOG("tokens: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_inp).c_str());

    // 如果 tokens 为空，则不应该运行
    if (embd_inp.empty()) {
        embd_inp.push_back(llama_token_bos(model));
        LOG("embd_inp was considered empty and bos was added: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_inp).c_str());
    }

    // 对负面提示进行标记化
    std::vector<llama_token> guidance_inp;
    int guidance_offset = 0;
    int original_prompt_len = 0;
    if (ctx_guidance) {
        LOG("cfg_negative_prompt: \"%s\"\n", log_tostr(sparams.cfg_negative_prompt));

        guidance_inp = ::llama_tokenize(ctx_guidance, sparams.cfg_negative_prompt, add_bos);
        LOG("guidance_inp tokenized: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx_guidance, guidance_inp).c_str());

        std::vector<llama_token> original_inp = ::llama_tokenize(ctx, params.prompt, add_bos);
        LOG("original_inp tokenized: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, original_inp).c_str());

        original_prompt_len = original_inp.size();
        guidance_offset = (int)guidance_inp.size() - original_prompt_len;
        LOG("original_prompt_len: %s", log_tostr(original_prompt_len));
        LOG("guidance_offset:     %s", log_tostr(guidance_offset));
    }

    // 如果 embd_inp 的大小大于 n_ctx - 4，则打印错误信息并返回 1
    if ((int) embd_inp.size() > n_ctx - 4) {
        LOG_TEE("%s: error: prompt is too long (%d tokens, max %d)\n", __func__, (int) embd_inp.size(), n_ctx - 4);
        return 1;
    }

    // 重置上下文时要保留的 tokens 数量
    if (params.n_keep < 0 || params.n_keep > (int) embd_inp.size()) {
        params.n_keep = (int)embd_inp.size();
    }

    // 打印输入前缀
    LOG("inp_pfx: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, inp_pfx).c_str());
    // 打印输入后缀
    LOG("inp_sfx: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, inp_sfx).c_str());

    // 如果指定了交互式开始，则启用交互模式
    if (params.interactive_first) {
        params.interactive = true;
    }
    # 如果参数中包含详细提示信息
    if (params.verbose_prompt) {
        # 输出换行符
        LOG_TEE("\n");
        # 输出函数名和提示信息
        LOG_TEE("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
        # 输出函数名和提示信息中的标记数量
        LOG_TEE("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
        # 遍历提示信息中的标记并输出它们的索引和内容
        for (int i = 0; i < (int) embd_inp.size(); i++) {
            LOG_TEE("%6d -> '%s'\n", embd_inp[i], llama_token_to_piece(ctx, embd_inp[i]).c_str());
        }

        # 如果存在上下文指导信息
        if (ctx_guidance) {
            # 输出换行符
            LOG_TEE("\n");
            # 输出函数名和负面提示信息
            LOG_TEE("%s: negative prompt: '%s'\n", __func__, sparams.cfg_negative_prompt.c_str());
            # 输出函数名和负面提示信息中的标记数量
            LOG_TEE("%s: number of tokens in negative prompt = %zu\n", __func__, guidance_inp.size());
            # 遍历负面提示信息中的标记并输出它们的索引和内容
            for (int i = 0; i < (int) guidance_inp.size(); i++) {
                LOG_TEE("%6d -> '%s'\n", guidance_inp[i], llama_token_to_piece(ctx, guidance_inp[i]).c_str());
            }
        }

        # 如果需要保留的标记数量大于0
        if (params.n_keep > 0) {
            # 输出函数名和基于 n_keep 的静态提示信息
            LOG_TEE("%s: static prompt based on n_keep: '", __func__);
            # 遍历基于 n_keep 的静态提示信息中的标记并输出它们的内容
            for (int i = 0; i < params.n_keep; i++) {
                LOG_TEE("%s", llama_token_to_piece(ctx, embd_inp[i]).c_str());
            }
            # 输出结束引号和换行符
            LOG_TEE("'\n");
        }
        # 输出换行符
        LOG_TEE("\n");
    }

    # 如果参数中包含交互式标志
    if (params.interactive) {
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
        // 如果是 UNIX 系统或者苹果系统，则设置 SIGINT 信号处理函数
        struct sigaction sigint_action;
        sigint_action.sa_handler = sigint_handler;
        sigemptyset (&sigint_action.sa_mask);
        sigint_action.sa_flags = 0;
        sigaction(SIGINT, &sigint_action, NULL);
#elif defined (_WIN32)
        // 如果是 Windows 系统，则设置控制台控制处理函数
        auto console_ctrl_handler = +[](DWORD ctrl_type) -> BOOL {
            return (ctrl_type == CTRL_C_EVENT) ? (sigint_handler(SIGINT), true) : false;
        };
        SetConsoleCtrlHandler(reinterpret_cast<PHANDLER_ROUTINE>(console_ctrl_handler), true);
#endif

        // 记录日志，表示进入交互模式
        LOG_TEE("%s: interactive mode on.\n", __func__);

        // 如果输入参数中包含 input_prefix_bos，则记录日志
        if (params.input_prefix_bos) {
            LOG_TEE("Input prefix with BOS\n");
        }

        // 如果输入参数中包含 input_prefix，则记录日志
        if (!params.input_prefix.empty()) {
            LOG_TEE("Input prefix: '%s'\n", params.input_prefix.c_str());
        }

        // 如果输入参数中包含 input_suffix，则记录日志
        if (!params.input_suffix.empty()) {
            LOG_TEE("Input suffix: '%s'\n", params.input_suffix.c_str());
        }
    }
    // 记录日志，打印采样参数
    LOG_TEE("sampling: \n%s\n", llama_sampling_print(sparams).c_str());
    // 记录日志，打印生成参数
    LOG_TEE("generate: n_ctx = %d, n_batch = %d, n_predict = %d, n_keep = %d\n", n_ctx, params.n_batch, params.n_predict, params.n_keep);
    // 记录日志，打印空行
    LOG_TEE("\n\n");

    // 记录日志，打印 Infill 模式
    LOG_TEE("\n#####  Infill mode  #####\n\n");
    // 如果输入参数中包含 infill，则打印提示信息
    if (params.infill) {
        printf("\n************\n");
        printf("no need to specify '--infill', always running infill\n");
        printf("************\n\n");
    }
    # 如果参数中包含交互式标志
    if (params.interactive) {
        # 定义控制消息变量
        const char *control_message;
        # 如果参数中包含多行输入标志
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
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__)) || defined (_WIN32)
        // 如果是 UNIX 系统、苹果系统或者 Windows 系统，则输出提示信息
        LOG_TEE(       " - Press Ctrl+C to interject at any time.\n");
#endif
        // 输出控制信息
        LOG_TEE(       "%s\n", control_message);

        // 设置交互标志为参数中的交互优先标志
        is_interacting = params.interactive_first;
    }

    // 设置输入回显为真
    bool input_echo           = true;

    // 初始化过去的输入数量为 0
    int n_past             = 0;
    // 初始化剩余预测数量为参数中的预测数量
    int n_remain           = params.n_predict;
    // 初始化已消耗数量为 0
    int n_consumed         = 0;
    // 初始化过去指导数量为 0
    int n_past_guidance    = 0;

    // 初始化输入令牌向量
    std::vector<int>   input_tokens;  g_input_tokens  = &input_tokens;
    // 初始化输出令牌向量
    std::vector<int>   output_tokens; g_output_tokens = &output_tokens;
    // 初始化输出字符串流
    std::ostringstream output_ss;     g_output_ss     = &output_ss;

    // 首先输出提示信息，设置颜色
    console::set_display(console::prompt);

    // 初始化嵌入向量
    std::vector<llama_token> embd;
    // 初始化指导嵌入向量
    std::vector<llama_token> embd_guidance;

    // 初始化采样上下文
    struct llama_sampling_context * ctx_sampling = llama_sampling_init(sparams);

    }
    // 如果不是交互模式且剩余数量小于等于 0，则输出结束标记并刷新输出
    if (!params.interactive && n_remain <= 0) {
        printf("%s", llama_token_to_piece(ctx, llama_token_eot(model)).c_str());
        fflush(stdout);
    }

    // 打印时间信息
    llama_print_timings(ctx);
    // 写入日志文件
    write_logfile(ctx, params, model, input_tokens, output_ss.str(), output_tokens);

    // 如果存在指导上下文，则释放
    if (ctx_guidance) { llama_free(ctx_guidance); }
    // 释放上下文
    llama_free(ctx);
    // 释放模型
    llama_free_model(model);

    // 释放采样上下文
    llama_sampling_free(ctx_sampling);
    // 释放后端
    llama_backend_free();

    // 如果未禁用日志，则输出日志结束信息
#ifndef LOG_DISABLE_LOGS
    LOG_TEE("Log end\n");
#endif // LOG_DISABLE_LOGS

    // 返回 0
    return 0;
}
```