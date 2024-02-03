# `PowerInfer\examples\main\main.cpp`

```cpp
// 包含自定义的头文件
#include "common.h"
#include "console.h"
#include "llama.h"

// 包含标准库头文件
#include <cassert>
#include <cinttypes>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <fstream>
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

// 根据不同的操作系统定义不同的头文件
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
#include <signal.h>
#include <unistd.h>
#elif defined (_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <windows.h>
#include <signal.h>
#endif

// 忽略特定编译器的警告
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义全局变量
static llama_context           ** g_ctx;
static llama_model             ** g_model;
static gpt_params               * g_params;
static std::vector<llama_token> * g_input_tokens;
static std::ostringstream       * g_output_ss;
static std::vector<llama_token> * g_output_tokens;
static bool is_interacting = false;

// 写入日志文件的函数
static void write_logfile(
    const llama_context * ctx, const gpt_params & params, const llama_model * model,
    const std::vector<llama_token> & input_tokens, const std::string & output,
    const std::vector<llama_token> & output_tokens
) {
    // 如果日志目录为空，则直接返回
    if (params.logdir.empty()) {
        return;
    }

    // 获取可排序的时间戳
    const std::string timestamp = get_sortable_timestamp();

    // 创建日志目录及其父目录
    const bool success = create_directory_with_parents(params.logdir);
    if (!success) {
        // 如果创建失败，则打印警告信息并返回
        fprintf(stderr, "%s: warning: failed to create logdir %s, cannot write logfile\n",
                __func__, params.logdir.c_str());
        return;
    }

    // 拼接日志文件路径
    const std::string logfile_path = params.logdir + timestamp + ".yml";
    FILE * logfile = fopen(logfile_path.c_str(), "w");

    // 如果打开日志文件失败，则打印错误信息并返回
    if (logfile == NULL) {
        fprintf(stderr, "%s: failed to open logfile %s\n", __func__, logfile_path.c_str());
        return;
    }

    // 写入日志文件的头部信息
    fprintf(logfile, "binary: main\n");
    char model_desc[128];
    llama_model_desc(model, model_desc, sizeof(model_desc));
    # 将非结果信息转储为 YAML 格式到日志文件中
    dump_non_result_info_yaml(logfile, params, ctx, timestamp, input_tokens, model_desc);
    
    # 在日志文件中输出空行
    fprintf(logfile, "\n");
    # 在日志文件中输出生成结果的标题
    fprintf(logfile, "######################\n");
    fprintf(logfile, "# Generation Results #\n");
    fprintf(logfile, "######################\n");
    # 在日志文件中输出空行
    fprintf(logfile, "\n");
    
    # 将输出内容以多行形式转储为 YAML 格式到日志文件中
    dump_string_yaml_multiline(logfile, "output", output.c_str());
    # 将输出标记以整数数组形式转储为 YAML 格式到日志文件中
    dump_vector_int_yaml(logfile, "output_tokens", output_tokens);
    
    # 将程序运行时间信息转储为 YAML 格式到日志文件中
    llama_dump_timing_info_yaml(logfile, ctx);
    # 关闭日志文件
    fclose(logfile);
// 定义信号处理函数，处理 SIGINT 信号
static void sigint_handler(int signo) {
    // 如果收到 SIGINT 信号
    if (signo == SIGINT) {
        // 如果当前不是交互状态
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

// 主函数
int main(int argc, char ** argv) {
    // 初始化参数对象
    gpt_params params;
    g_params = &params;

    // 解析命令行参数
    if (!gpt_params_parse(argc, argv, params)) {
        return 1;
    }
    // 获取采样参数对象的引用
    llama_sampling_params & sparams = params.sparams;

    // 如果日志未禁用
    #ifndef LOG_DISABLE_LOGS
    // 设置日志输出目标为日志文件
    log_set_target(log_filename_generator("main", "log"));
    // 打印日志开始信息
    LOG_TEE("Log start\n");
    // 输出命令行参数
    log_dump_cmdline(argc, argv);
    #endif // LOG_DISABLE_LOGS

    // 初始化控制台
    // 保存颜色选择以供后续使用
    // （注意：这是一个稍微尴尬的选择）
    console::init(params.simple_io, params.use_color);
    // 在程序退出时清理控制台
    atexit([]() { console::cleanup(); });

    // 如果选择输出所有 logits
    if (params.logits_all) {
        printf("\n************\n");
        printf("%s: please use the 'perplexity' tool for perplexity calculations\n", __func__);
        printf("************\n\n");
        return 0;
    }

    // 如果选择输出 embedding
    if (params.embedding) {
        printf("\n************\n");
        printf("%s: please use the 'embedding' tool for embedding calculations\n", __func__);
        printf("************\n\n");
        return 0;
    }

    // 如果上下文大小不为 0 且小于 8
    if (params.n_ctx != 0 && params.n_ctx < 8) {
        // 输出警告信息
        LOG_TEE("%s: warning: minimum context size is 8, using minimum size.\n", __func__);
        // 设置上下文大小为 8
        params.n_ctx = 8;
    }

    // 如果 RoPE 频率基数不为 0.0
    if (params.rope_freq_base != 0.0) {
        // 输出警告信息
        LOG_TEE("%s: warning: changing RoPE frequency base to %g.\n", __func__, params.rope_freq_base);
    }
}
    // 如果参数中的rope_freq_scale不等于0.0，则输出警告信息
    if (params.rope_freq_scale != 0.0) {
        LOG_TEE("%s: warning: scaling RoPE frequency by %g.\n", __func__, params.rope_freq_scale);
    }

    // 输出构建信息，包括构建号和提交信息
    LOG_TEE("%s: build = %d (%s)\n",      __func__, LLAMA_BUILD_NUMBER, LLAMA_COMMIT);
    LOG_TEE("%s: built with %s for %s\n", __func__, LLAMA_COMPILER, LLAMA_BUILD_TARGET);

    // 如果参数中的seed等于默认值LLAMA_DEFAULT_SEED，则将seed设置为当前时间
    if (params.seed == LLAMA_DEFAULT_SEED) {
        params.seed = time(NULL);
    }

    // 输出种子值
    LOG_TEE("%s: seed  = %u\n", __func__, params.seed);

    // 使用参数中的seed创建伪随机数生成器
    std::mt19937 rng(params.seed);
    // 如果参数中的random_prompt为真，则使用随机数生成器生成提示信息
    if (params.random_prompt) {
        params.prompt = gpt_random_prompt(rng);
    }

    // 输出 llama 后端初始化信息
    LOG("%s: llama backend init\n", __func__);
    // 初始化 llama 后端
    llama_backend_init(params.numa);

    llama_model * model;
    llama_context * ctx;
    llama_context * ctx_guidance = NULL;
    g_model = &model;
    g_ctx = &ctx;

    // 加载模型并应用lora适配器（如果有）
    LOG("%s: load the model and apply lora adapter, if any\n", __func__);
    // 从gpt参数中初始化模型和上下文
    std::tie(model, ctx) = llama_init_from_gpt_params(params);
    // 如果sparams中的cfg_scale大于1.0，则根据gpt参数创建上下文参数，并使用模型创建上下文
    if (sparams.cfg_scale > 1.f) {
        struct llama_context_params lparams = llama_context_params_from_gpt_params(params);
        ctx_guidance = llama_new_context_with_model(model, lparams);
    }

    // 如果模型为空，则输出错误信息并返回1
    if (model == NULL) {
        LOG_TEE("%s: error: unable to load model\n", __func__);
        return 1;
    }

    // 获取训练上下文的数量和当前上下文的数量，并输出当前上下文的数量
    const int n_ctx_train = llama_n_ctx_train(model);
    const int n_ctx = llama_n_ctx(ctx);
    LOG("n_ctx: %d\n", n_ctx);

    // 如果当前上下文的数量大于训练上下文的数量，则输出警告信息
    if (n_ctx > n_ctx_train) {
        LOG_TEE("%s: warning: model was trained on only %d context tokens (%d specified)\n",
                __func__, n_ctx_train, n_ctx);
    }

    // 打印系统信息
    {
        LOG_TEE("\n");
        LOG_TEE("%s\n", get_system_info(params).c_str());
    }

    // 设置会话路径为参数中的提示缓存路径，创建会话令牌的空向量
    std::string path_session = params.path_prompt_cache;
    std::vector<llama_token> session_tokens;
    // 如果路径不为空
    if (!path_session.empty()) {
        // 打印尝试从指定路径加载保存的会话
        LOG_TEE("%s: attempting to load saved session from '%s'\n", __func__, path_session.c_str());

        // 打开文件以检查现有会话
        FILE * fp = std::fopen(path_session.c_str(), "rb");
        // 如果文件存在
        if (fp != NULL) {
            std::fclose(fp);

            // 调整会话令牌大小
            session_tokens.resize(n_ctx);
            size_t n_token_count_out = 0;
            // 如果加载会话文件失败
            if (!llama_load_session_file(ctx, path_session.c_str(), session_tokens.data(), session_tokens.capacity(), &n_token_count_out)) {
                LOG_TEE("%s: error: failed to load session file '%s'\n", __func__, path_session.c_str());
                return 1;
            }
            // 调整会话令牌大小
            session_tokens.resize(n_token_count_out);
            // 设置随机数种子
            llama_set_rng_seed(ctx, params.seed);

            // 打印加载了包含特定令牌数量的会话
            LOG_TEE("%s: loaded a session with prompt size of %d tokens\n", __func__, (int) session_tokens.size());
        } else {
            // 打印会话文件不存在，将创建新的会话
            LOG_TEE("%s: session file does not exist, will create\n", __func__);
        }
    }

    // 检查是否需要添加bos
    const bool add_bos = llama_vocab_type(model) == LLAMA_VOCAB_TYPE_SPM;
    LOG("add_bos: %d\n", add_bos);

    // 创建嵌入输入的令牌向量
    std::vector<llama_token> embd_inp;

    // 如果是交互式的第一个或者有指令或者提示不为空或者会话令牌为空
    if (params.interactive_first || params.instruct || !params.prompt.empty() || session_tokens.empty()) {
        // 打印标记化提示
        LOG("tokenize the prompt\n");
        embd_inp = ::llama_tokenize(ctx, params.prompt, add_bos, true);
    } else {
        // 使用会话令牌
        LOG("use session tokens\n");
        embd_inp = session_tokens;
    }

    // 打印提示和令牌
    LOG("prompt: \"%s\"\n", log_tostr(params.prompt));
    LOG("tokens: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_inp).c_str());

    // 如果令牌为空，则不应该运行
    if (embd_inp.empty()) {
        embd_inp.push_back(llama_token_bos(model));
        // 打印embd_inp被认为为空，bos被添加
        LOG("embd_inp was considered empty and bos was added: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_inp).c_str());
    }

    // 标记化负面提示
    std::vector<llama_token> guidance_inp;
    int guidance_offset = 0;
    int original_prompt_len = 0;
    // 如果存在上下文指导，则记录负面提示的配置信息
    if (ctx_guidance) {
        LOG("cfg_negative_prompt: \"%s\"\n", log_tostr(sparams.cfg_negative_prompt));

        // 使用 llama_tokenize 函数对上下文指导进行分词处理
        guidance_inp = ::llama_tokenize(ctx_guidance, sparams.cfg_negative_prompt, add_bos, true);
        LOG("guidance_inp tokenized: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx_guidance, guidance_inp).c_str());

        // 使用 llama_tokenize 函数对原始上下文进行分词处理
        std::vector<llama_token> original_inp = ::llama_tokenize(ctx, params.prompt, add_bos, true);
        LOG("original_inp tokenized: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, original_inp).c_str());

        // 记录原始上下文的长度
        original_prompt_len = original_inp.size();
        // 计算上下文指导与原始上下文长度的偏移量
        guidance_offset = (int)guidance_inp.size() - original_prompt_len;
        LOG("original_prompt_len: %s", log_tostr(original_prompt_len));
        LOG("guidance_offset:     %s", log_tostr(guidance_offset));
    }

    // 如果嵌入输入的长度超过了 n_ctx - 4，则记录错误信息并返回
    if ((int) embd_inp.size() > n_ctx - 4) {
        LOG_TEE("%s: error: prompt is too long (%d tokens, max %d)\n", __func__, (int) embd_inp.size(), n_ctx - 4);
        return 1;
    }

    // 如果适用，记录关于保存会话相似性的调试信息
    size_t n_matching_session_tokens = 0;
    // 如果会话令牌不为空
    if (!session_tokens.empty()) {
        // 遍历会话令牌
        for (llama_token id : session_tokens) {
            // 如果匹配的会话令牌数量大于等于输入的 embd_inp 大小，或者当前 id 不等于 embd_inp 中的对应位置的值，则跳出循环
            if (n_matching_session_tokens >= embd_inp.size() || id != embd_inp[n_matching_session_tokens]) {
                break;
            }
            // 匹配的会话令牌数量加一
            n_matching_session_tokens++;
        }
        // 如果参数的提示为空，并且匹配的会话令牌数量等于 embd_inp 的大小
        if (params.prompt.empty() && n_matching_session_tokens == embd_inp.size()) {
            // 输出日志
            LOG_TEE("%s: using full prompt from session file\n", __func__);
        } 
        // 如果匹配的会话令牌数量大于等于 embd_inp 的大小
        else if (n_matching_session_tokens >= embd_inp.size()) {
            // 输出日志
            LOG_TEE("%s: session file has exact match for prompt!\n", __func__);
        } 
        // 如果匹配的会话令牌数量小于 embd_inp 大小的一半
        else if (n_matching_session_tokens < (embd_inp.size() / 2)) {
            // 输出警告日志
            LOG_TEE("%s: warning: session file has low similarity to prompt (%zu / %zu tokens); will mostly be reevaluated\n",
                __func__, n_matching_session_tokens, embd_inp.size());
        } 
        // 其他情况
        else {
            // 输出日志
            LOG_TEE("%s: session file matches %zu / %zu tokens of prompt\n",
                __func__, n_matching_session_tokens, embd_inp.size());
        }

        // 移除可能从上一个会话继承的“未来”令牌
        llama_kv_cache_seq_rm(ctx, -1, n_matching_session_tokens, -1);
    }

    // 输出日志，用于检查缓存日志
    LOGLN(
            "recalculate the cached logits (check): embd_inp.empty() %s, n_matching_session_tokens %zu, embd_inp.size() %zu, session_tokens.size() %zu, embd_inp.size() %zu",
            log_tostr(embd_inp.empty()), n_matching_session_tokens, embd_inp.size(), session_tokens.size(), embd_inp.size());

    // 如果我们将使用缓存来存储完整的提示，而不是达到缓存的末尾，强制重新评估最后一个令牌，以重新计算缓存的对数
    if (!embd_inp.empty() && n_matching_session_tokens == embd_inp.size() && session_tokens.size() > embd_inp.size()) {
        // 输出日志
        LOGLN("recalculate the cached logits (do): session_tokens.resize( %zu )", embd_inp.size() - 1);
        // 调整会话令牌的大小
        session_tokens.resize(embd_inp.size() - 1);
    }
    // 设置重置上下文时要保留的标记数
    if (params.n_keep < 0 || params.n_keep > (int) embd_inp.size() || params.instruct) {
        params.n_keep = (int)embd_inp.size();
    }

    // 为指令模式设置前缀和后缀
    const auto inp_pfx = ::llama_tokenize(ctx, "\n\n### Instruction:\n\n", add_bos, true);
    const auto inp_sfx = ::llama_tokenize(ctx, "\n\n### Response:\n\n",    false,   true);

    LOG("inp_pfx: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, inp_pfx).c_str());
    LOG("inp_sfx: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, inp_sfx).c_str());

    // 在指令模式下，为用户的每个输入添加前缀和后缀
    if (params.instruct) {
        params.interactive_first = true;
        params.antiprompt.push_back("### Instruction:\n\n");
    }

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
    // 如果是 Unix 系统或者苹果系统
    // 定义一个 sigaction 结构体来处理 SIGINT 信号
    struct sigaction sigint_action;
    // 设置信号处理函数为 sigint_handler
    sigint_action.sa_handler = sigint_handler;
    // 清空信号集
    sigemptyset (&sigint_action.sa_mask);
    // 设置标志为 0
    sigint_action.sa_flags = 0;
    // 指定对 SIGINT 信号的处理方式
    sigaction(SIGINT, &sigint_action, NULL);
#elif defined (_WIN32)
    // 如果是 Windows 系统
    // 定义一个 lambda 表达式来处理控制台事件
    auto console_ctrl_handler = +[](DWORD ctrl_type) -> BOOL {
        // 如果是 CTRL+C 事件，则调用 sigint_handler 处理 SIGINT 信号
        return (ctrl_type == CTRL_C_EVENT) ? (sigint_handler(SIGINT), true) : false;
    };
    // 设置控制台事件处理函数为 console_ctrl_handler
    SetConsoleCtrlHandler(reinterpret_cast<PHANDLER_ROUTINE>(console_ctrl_handler), true);
#endif

        // 打印提示信息，表示进入交互模式
        LOG_TEE("%s: interactive mode on.\n", __func__);

        // 如果存在反向提示信息，则逐个打印并解析
        if (!params.antiprompt.empty()) {
            for (const auto & antiprompt : params.antiprompt) {
                LOG_TEE("Reverse prompt: '%s'\n", antiprompt.c_str());
                // 如果设置了详细提示信息，则逐个解析并打印
                if (params.verbose_prompt) {
                    auto tmp = ::llama_tokenize(ctx, antiprompt, false, true);
                    for (int i = 0; i < (int) tmp.size(); i++) {
                        LOG_TEE("%6d -> '%s'\n", tmp[i], llama_token_to_piece(ctx, tmp[i]).c_str());
                    }
                }
            }
        }

        // 如果设置了输入前缀以BOS开头，则打印提示信息
        if (params.input_prefix_bos) {
            LOG_TEE("Input prefix with BOS\n");
        }

        // 如果存在输入前缀，则打印并解析
        if (!params.input_prefix.empty()) {
            LOG_TEE("Input prefix: '%s'\n", params.input_prefix.c_str());
            // 如果设置了详细提示信息，则逐个解析并打印
            if (params.verbose_prompt) {
                auto tmp = ::llama_tokenize(ctx, params.input_prefix, true, true);
                for (int i = 0; i < (int) tmp.size(); i++) {
                    LOG_TEE("%6d -> '%s'\n", tmp[i], llama_token_to_piece(ctx, tmp[i]).c_str());
                }
            }
        }

        // 如果存在输入后缀，则打印并解析
        if (!params.input_suffix.empty()) {
            LOG_TEE("Input suffix: '%s'\n", params.input_suffix.c_str());
            // 如果设置了详细提示信息，则逐个解析并打印
            if (params.verbose_prompt) {
                auto tmp = ::llama_tokenize(ctx, params.input_suffix, false, true);
                for (int i = 0; i < (int) tmp.size(); i++) {
                    LOG_TEE("%6d -> '%s'\n", tmp[i], llama_token_to_piece(ctx, tmp[i]).c_str());
                }
            }
        }
    }
    // 打印采样信息
    LOG_TEE("sampling: \n%s\n", llama_sampling_print(sparams).c_str());
    // 打印生成信息
    LOG_TEE("generate: n_ctx = %d, n_batch = %d, n_predict = %d, n_keep = %d\n", n_ctx, params.n_batch, params.n_predict, params.n_keep);
    // 打印空行
    LOG_TEE("\n\n");
    # 如果参数中包含交互模式
    if (params.interactive) {
        # 定义控制信息的指针
        const char *control_message;
        # 如果参数中包含多行输入
        if (params.multiline_input) {
            # 设置控制信息为多行输入的提示
            control_message = " - To return control to LLaMa, end your input with '\\'.\n"
                              " - To return control without starting a new line, end your input with '/'.\n";
        } else {
            # 设置控制信息为单行输入的提示
            control_message = " - Press Return to return control to LLaMa.\n"
                              " - To return control without starting a new line, end your input with '/'.\n"
                              " - If you want to submit another line, end your input with '\\'.\n";
        }
        # 打印日志，表示正在运行交互模式
        LOG_TEE("== Running in interactive mode. ==\n");
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__)) || defined (_WIN32)
        // 如果是 Unix 系统、苹果系统或者 Windows 系统，则打印提示信息
        LOG_TEE(       " - Press Ctrl+C to interject at any time.\n");
#endif
        // 打印控制信息
        LOG_TEE(       "%s\n", control_message);

        // 设置交互标志为参数中的交互优先标志
        is_interacting = params.interactive_first;
    }

    // 初始化变量
    bool is_antiprompt        = false;
    bool input_echo           = true;
    bool need_to_save_session = !path_session.empty() && n_matching_session_tokens < embd_inp.size();

    int n_past             = 0;
    int n_remain           = params.n_predict;
    int n_consumed         = 0;
    int n_session_consumed = 0;
    int n_past_guidance    = 0;

    // 初始化输入和输出的 token 数组
    std::vector<int>   input_tokens;  g_input_tokens  = &input_tokens;
    std::vector<int>   output_tokens; g_output_tokens = &output_tokens;
    std::ostringstream output_ss;     g_output_ss     = &output_ss;

    // 设置显示颜色为提示信息的颜色
    console::set_display(console::prompt);

    // 初始化 embd 和 embd_guidance 变量
    std::vector<llama_token> embd;
    std::vector<llama_token> embd_guidance;

    // 初始化采样上下文
    struct llama_sampling_context * ctx_sampling = llama_sampling_init(sparams);

    }

    // 如果需要保存会话信息并且参数中要求缓存所有提示信息并且不是只读模式
    if (!path_session.empty() && params.prompt_cache_all && !params.prompt_cache_ro) {
        // 打印保存最终输出到会话文件的信息
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
    // 释放后端
    llama_backend_free();

    // 如果日志未禁用，则打印日志结束信息
#ifndef LOG_DISABLE_LOGS
    LOG_TEE("Log end\n");
#endif // LOG_DISABLE_LOGS

    // 返回 0 表示正常结束
    return 0;
}
```