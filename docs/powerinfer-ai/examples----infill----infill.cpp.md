# `PowerInfer\examples\infill\infill.cpp`

```
#include "common.h"  // 包含 common.h 头文件

#include "console.h"  // 包含 console.h 头文件
#include "llama.h"  // 包含 llama.h 头文件
#include "grammar-parser.h"  // 包含 grammar-parser.h 头文件

#include <cassert>  // 包含断言相关的头文件
#include <cinttypes>  // 包含整数类型相关的头文件
#include <cmath>  // 包含数学函数相关的头文件
#include <cstdio>  // 包含输入输出相关的头文件
#include <cstring>  // 包含字符串处理相关的头文件
#include <ctime>  // 包含时间处理相关的头文件
#include <fstream>  // 包含文件输入输出相关的头文件
#include <iostream>  // 包含标准输入输出流相关的头文件
#include <sstream>  // 包含字符串流相关的头文件
#include <string>  // 包含字符串相关的头文件
#include <vector>  // 包含向量相关的头文件

#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
#include <signal.h>  // 如果是 Unix 系统或者苹果系统，则包含信号处理相关的头文件
// 包含头文件，根据操作系统定义不同的宏
#include <unistd.h>
#elif defined (_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
#define NOMINMAX
#endif
#include <windows.h>
#include <signal.h>
#endif

// 如果使用的是 MSC 编译器，禁止特定警告
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义全局变量
static llama_context           ** g_ctx;  // 指向 llama_context 指针的指针
static llama_model             ** g_model;  // 指向 llama_model 指针的指针
static gpt_params               * g_params;  // 指向 gpt_params 结构体的指针
static std::vector<llama_token> * g_input_tokens;  // 指向存储 llama_token 对象的向量的指针
static std::ostringstream       * g_output_ss;  // 指向 ostringstream 对象的指针
static std::vector<llama_token> * g_output_tokens;  // 指向存储 llama_token 对象的向量的指针
// 定义静态变量，表示是否正在交互
static bool is_interacting = false;

// 写入日志文件，记录上下文、参数、模型、输入和输出的标记
static void write_logfile(
    const llama_context * ctx, const gpt_params & params, const llama_model * model,
    const std::vector<llama_token> & input_tokens, const std::string & output,
    const std::vector<llama_token> & output_tokens
) {
    // 如果日志目录为空，则不进行日志记录
    if (params.logdir.empty()) {
        return;
    }

    // 获取可排序的时间戳
    const std::string timestamp = get_sortable_timestamp();

    // 创建日志目录及其父目录，如果创建失败则输出警告信息并返回
    const bool success = create_directory_with_parents(params.logdir);
    if (!success) {
        fprintf(stderr, "%s: warning: failed to create logdir %s, cannot write logfile\n",
                __func__, params.logdir.c_str());
        return;
    }
# 定义日志文件路径，使用参数中的日志目录和时间戳拼接成文件名
const std::string logfile_path = params.logdir + timestamp + ".yml";
# 以写入模式打开日志文件
FILE * logfile = fopen(logfile_path.c_str(), "w");

# 如果打开文件失败，输出错误信息并返回
if (logfile == NULL) {
    fprintf(stderr, "%s: failed to open logfile %s\n", __func__, logfile_path.c_str());
    return;
}

# 向日志文件中写入内容
fprintf(logfile, "binary: infill");
# 定义字符数组存储模型描述信息
char model_desc[128];
# 获取模型描述信息并存储到字符数组中
llama_model_desc(model, model_desc, sizeof(model_desc));
# 将非结果信息以 YAML 格式写入日志文件
dump_non_result_info_yaml(logfile, params, ctx, timestamp, input_tokens, model_desc);

# 向日志文件中写入分隔符和标题
fprintf(logfile, "\n");
fprintf(logfile, "######################\n");
fprintf(logfile, "# Generation Results #\n");
fprintf(logfile, "######################\n");
fprintf(logfile, "\n");
```
# 将字符串以多行形式写入到 YAML 文件中
dump_string_yaml_multiline(logfile, "output", output.c_str());
# 将整数向量写入到 YAML 文件中
dump_vector_int_yaml(logfile, "output_tokens", output_tokens);
# 将时间信息写入到 YAML 文件中
llama_dump_timing_info_yaml(logfile, ctx);
# 关闭日志文件
fclose(logfile);
# 如果是 UNIX 系统、苹果系统或者 Windows 系统
static void sigint_handler(int signo) {
    # 如果收到 SIGINT 信号
    if (signo == SIGINT) {
        # 如果不是交互状态
        if (!is_interacting) {
            # 设置为交互状态
            is_interacting = true;
        } else {
            # 清理控制台
            console::cleanup();
            # 打印换行符
            printf("\n");
            # 打印时间信息
            llama_print_timings(*g_ctx);
            # 写入日志文件
            write_logfile(*g_ctx, *g_params, *g_model, *g_input_tokens, g_output_ss->str(), *g_output_tokens);
            # 退出程序
            _exit(130);
        }
    }
}
}
#endif
// 结束预处理指令

int main(int argc, char ** argv) {
    // 定义参数对象
    gpt_params params;
    // 获取采样参数的引用
    llama_sampling_params & sparams = params.sparams;
    // 设置全局参数指针
    g_params = &params;

    // 解析命令行参数，如果失败则返回1
    if (!gpt_params_parse(argc, argv, params)) {
        return 1;
    }

#ifndef LOG_DISABLE_LOGS
    // 设置日志输出目标
    log_set_target(log_filename_generator("infill", "log"));
    // 输出日志信息
    LOG_TEE("Log start\n");
    // 输出命令行参数
    log_dump_cmdline(argc, argv);
#endif // LOG_DISABLE_LOGS

    // 初始化控制台
    console::init(params.simple_io, params.use_color);
    // 在程序退出时执行清理操作
    atexit([]() { console::cleanup(); });
# 如果参数 logits_all 为真，则打印提示信息并返回 0
if (params.logits_all) {
    printf("\n************\n");
    printf("%s: please use the 'perplexity' tool for perplexity calculations\n", __func__);
    printf("************\n\n");
    return 0;
}

# 如果参数 embedding 为真，则打印提示信息并返回 0
if (params.embedding) {
    printf("\n************\n");
    printf("%s: please use the 'embedding' tool for embedding calculations\n", __func__);
    printf("************\n\n");
    return 0;
}

# 如果参数 n_ctx 不为 0 且小于 8，则打印警告信息并将参数 n_ctx 设置为 8
if (params.n_ctx != 0 && params.n_ctx < 8) {
    LOG_TEE("%s: warning: minimum context size is 8, using minimum size.\n", __func__);
    params.n_ctx = 8;
}
    }
    // 如果指定了instruct参数，则打印提示信息并返回0
    if (params.instruct) {
        printf("\n************\n");
        printf("%s: please use the 'main' tool for instruct mode\n", __func__);
        printf("************\n\n");

        return 0;
    }
    // 如果antiprompt参数不为空，则打印提示信息并返回0
    if (!params.antiprompt.empty()) {
        printf("\n************\n");
        printf("%s: please use the 'main' tool for antiprompt mode\n", __func__);
        printf("************\n\n");

        return 0;
    }
    // 如果不是交互式模式且未指定输入前缀和后缀，则打印提示信息并返回0
    if (!params.interactive_first && (params.input_prefix.empty() && params.input_suffix.empty())) {
        printf("\n************\n");
        printf("%s: please use '--interactive_first' or specify '--in_prefix' and/or '--in_suffix'\n", __func__);
        printf("************\n\n");
    // 如果条件成立，返回 0
    return 0;
    // 如果随机提示参数为真
    if (params.random_prompt) {
        // 打印提示信息
        printf("\n************\n");
        printf("%s: please use the 'main' tool for random prompt mode\n", __func__);
        printf("************\n\n");
        // 返回 0
        return 0;
    }
    // 如果路径提示缓存不为空
    if (!params.path_prompt_cache.empty()) {
        // 打印提示信息
        printf("\n************\n");
        printf("%s: infill does not support prompt caching\n", __func__);
        printf("************\n\n");
        // 返回 0
        return 0;
    }
    // 如果 RoPE 频率基数不为 0.0
    if (params.rope_freq_base != 0.0) {
        // 记录警告信息
        LOG_TEE("%s: warning: changing RoPE frequency base to %g.\n", __func__, params.rope_freq_base);
    }
# 如果参数中的rope_freq_scale不等于0.0，则记录警告信息
if (params.rope_freq_scale != 0.0) {
    LOG_TEE("%s: warning: scaling RoPE frequency by %g.\n", __func__, params.rope_freq_scale);
}

# 记录构建信息和编译器信息
LOG_TEE("%s: build = %d (%s)\n",      __func__, LLAMA_BUILD_NUMBER, LLAMA_COMMIT);
LOG_TEE("%s: built with %s for %s\n", __func__, LLAMA_COMPILER, LLAMA_BUILD_TARGET);

# 如果参数中的seed等于默认值LLAMA_DEFAULT_SEED，则将seed设置为当前时间
if (params.seed == LLAMA_DEFAULT_SEED) {
    params.seed = time(NULL);
}

# 记录seed值
LOG_TEE("%s: seed  = %u\n", __func__, params.seed);

# 使用seed值初始化随机数生成器
std::mt19937 rng(params.seed);

# 记录llama后端初始化信息
LOG("%s: llama backend init\n", __func__);
llama_backend_init(params.numa);

# 声明llama_model指针
llama_model * model;
// 声明指向 llama_context 结构体的指针变量 ctx 和 ctx_guidance
llama_context * ctx;
llama_context * ctx_guidance = NULL;
// 将全局变量 g_model 指向 model 变量的地址
g_model = &model;
// 将全局变量 g_ctx 指向 ctx 变量的地址
g_ctx = &ctx;

// 加载模型并应用 lora 适配器（如果有的话）
LOG("%s: load the model and apply lora adapter, if any\n", __func__);
// 调用 llama_init_from_gpt_params 函数初始化模型和上下文
std::tie(model, ctx) = llama_init_from_gpt_params(params);
// 如果 sparams.cfg_scale 大于 1.0，则根据参数创建 llama_context_params 结构体，并使用该结构体创建新的上下文 ctx_guidance
if (sparams.cfg_scale > 1.f) {
    struct llama_context_params lparams = llama_context_params_from_gpt_params(params);
    ctx_guidance = llama_new_context_with_model(model, lparams);
}

// 如果模型为空，则打印错误信息并返回 1
if (model == NULL) {
    LOG_TEE("%s: error: unable to load model\n", __func__);
    return 1;
}

// 获取训练上下文的数量并赋值给 n_ctx_train
const int n_ctx_train = llama_n_ctx_train(model);
// 获取上下文的数量并赋值给 n_ctx
const int n_ctx = llama_n_ctx(ctx);
    // 打印 n_ctx 的值
    LOG("n_ctx: %d\n", n_ctx);

    // 如果 n_ctx 大于 n_ctx_train，则打印警告信息
    if (n_ctx > n_ctx_train) {
        LOG_TEE("%s: warning: model was trained on only %d context tokens (%d specified)\n",
                __func__, n_ctx_train, n_ctx);
    }

    // 打印系统信息
    {
        LOG_TEE("\n");
        LOG_TEE("%s\n", get_system_info(params).c_str());
    }

    // 根据模型的词汇类型判断是否添加 bos
    const bool add_bos = llama_vocab_type(model) == LLAMA_VOCAB_TYPE_SPM;
    LOG("add_bos: %d\n", add_bos);

    // 根据参数判断是否需要移除输入后缀的前导空格
    bool suff_rm_leading_spc = params.escape;
    if (suff_rm_leading_spc && params.input_suffix.find_first_of(" ") == 0 && params.input_suffix.size() > 1) {
        params.input_suffix.erase(0, 1);
        suff_rm_leading_spc = false;
    }
// 创建一个存储 llama_token 类型的向量 embd_inp
std::vector<llama_token> embd_inp;
// 使用 llama_tokenize 函数将 params.input_prefix 转换为 llama_token 类型的向量
std::vector<llama_token> inp_pfx = ::llama_tokenize(ctx, params.input_prefix, false);
// 使用 llama_tokenize 函数将 params.input_suffix 转换为 llama_token 类型的向量
std::vector<llama_token> inp_sfx = ::llama_tokenize(ctx, params.input_suffix, false);
// 定义一个整型变量 space_token 并赋值为 29871
const int space_token = 29871;
// 如果 suff_rm_leading_spc 为真且 inp_sfx 的第一个元素等于 space_token，则删除 inp_sfx 的第一个元素
if (suff_rm_leading_spc && inp_sfx[0] == space_token) {
    inp_sfx.erase(inp_sfx.begin());
}
// 在 inp_pfx 的开头插入 llama_token_prefix(model)
inp_pfx.insert(inp_pfx.begin(), llama_token_prefix(model));
// 如果 add_bos 为真，则在 inp_pfx 的开头插入 llama_token_bos(model)
if (add_bos) {
    inp_pfx.insert(inp_pfx.begin(), llama_token_bos(model));
}
// 在 inp_sfx 的开头插入 llama_token_suffix(model)
inp_sfx.insert(inp_sfx.begin(), llama_token_suffix(model));
// 将 inp_pfx 的内容复制到 embd_inp
embd_inp = inp_pfx;
// 将 inp_sfx 的内容插入到 embd_inp 的末尾
embd_inp.insert(embd_inp.end(), inp_sfx.begin(), inp_sfx.end());
// 在 embd_inp 的末尾插入 llama_token_middle(model)
embd_inp.push_back(llama_token_middle(model));

// 打印 params.input_prefix 的值
LOG("prefix: \"%s\"\n", log_tostr(params.input_prefix));
// 打印 params.input_suffix 的值
LOG("suffix: \"%s\"\n", log_tostr(params.input_suffix));
// 打印 embd_inp 的内容
LOG("tokens: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_inp).c_str());
    // 如果输入的 embd_inp 为空，添加一个特殊的 token
    if (embd_inp.empty()) {
        embd_inp.push_back(llama_token_bos(model));
        LOG("embd_inp was considered empty and bos was added: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_inp).c_str());
    }

    // 对负面提示进行分词
    std::vector<llama_token> guidance_inp;
    int guidance_offset = 0;
    int original_prompt_len = 0;
    if (ctx_guidance) {
        LOG("cfg_negative_prompt: \"%s\"\n", log_tostr(sparams.cfg_negative_prompt));

        // 使用 llama_tokenize 函数对负面提示进行分词
        guidance_inp = ::llama_tokenize(ctx_guidance, sparams.cfg_negative_prompt, add_bos);
        LOG("guidance_inp tokenized: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx_guidance, guidance_inp).c_str());

        // 使用 llama_tokenize 函数对原始提示进行分词
        std::vector<llama_token> original_inp = ::llama_tokenize(ctx, params.prompt, add_bos);
        LOG("original_inp tokenized: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, original_inp).c_str());

        // 记录原始提示的长度
        original_prompt_len = original_inp.size();
    # 计算偏移量，即输入的引导文本长度减去原始提示文本长度
    guidance_offset = (int)guidance_inp.size() - original_prompt_len;
    # 打印原始提示文本长度
    LOG("original_prompt_len: %s", log_tostr(original_prompt_len));
    # 打印偏移量
    LOG("guidance_offset:     %s", log_tostr(guidance_offset));

    # 如果嵌入输入的长度超过了上下文长度减去4
    if ((int) embd_inp.size() > n_ctx - 4) {
        # 打印错误信息，提示输入的提示文本太长
        LOG_TEE("%s: error: prompt is too long (%d tokens, max %d)\n", __func__, (int) embd_inp.size(), n_ctx - 4);
        # 返回错误代码1
        return 1;
    }

    # 设置重置上下文时要保留的标记数
    if (params.n_keep < 0 || params.n_keep > (int) embd_inp.size()) {
        params.n_keep = (int)embd_inp.size();
    }

    # 打印输入前缀的标记
    LOG("inp_pfx: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, inp_pfx).c_str());
    # 打印输入后缀的标记
    LOG("inp_sfx: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, inp_sfx).c_str());

    # 如果指定了交互式开始，则启用交互模式
    # 如果参数中包含交互式优先标志，则将交互式标志设置为true
    if (params.interactive_first) {
        params.interactive = true;
    }

    # 如果参数中包含详细提示标志，则输出函数名称和提示内容
    if (params.verbose_prompt) {
        LOG_TEE("\n");
        LOG_TEE("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
        LOG_TEE("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
        # 遍历提示内容中的每个token，并输出其索引和内容
        for (int i = 0; i < (int) embd_inp.size(); i++) {
            LOG_TEE("%6d -> '%s'\n", embd_inp[i], llama_token_to_piece(ctx, embd_inp[i]).c_str());
        }

        # 如果存在上下文指导，则输出负面提示内容
        if (ctx_guidance) {
            LOG_TEE("\n");
            LOG_TEE("%s: negative prompt: '%s'\n", __func__, sparams.cfg_negative_prompt.c_str());
            LOG_TEE("%s: number of tokens in negative prompt = %zu\n", __func__, guidance_inp.size());
            # 遍历负面提示内容中的每个token，并输出其索引和内容
            for (int i = 0; i < (int) guidance_inp.size(); i++) {
                LOG_TEE("%6d -> '%s'\n", guidance_inp[i], llama_token_to_piece(ctx, guidance_inp[i]).c_str());
            }
        }
    }
// 如果保留的数量大于0，则输出基于n_keep的静态提示
if (params.n_keep > 0) {
    LOG_TEE("%s: static prompt based on n_keep: '", __func__);
    // 遍历n_keep个元素，输出对应的提示信息
    for (int i = 0; i < params.n_keep; i++) {
        LOG_TEE("%s", llama_token_to_piece(ctx, embd_inp[i]).c_str());
    }
    LOG_TEE("'\n");
}
// 输出换行符
LOG_TEE("\n");
}

// 如果是交互式模式
if (params.interactive) {
    // 如果是Unix系统或者苹果系统
    #if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__))
        // 设置SIGINT信号的处理函数为sigint_handler
        struct sigaction sigint_action;
        sigint_action.sa_handler = sigint_handler;
        sigemptyset (&sigint_action.sa_mask);
        sigint_action.sa_flags = 0;
        sigaction(SIGINT, &sigint_action, NULL);
    // 如果是Windows系统
    #elif defined (_WIN32)
        // 设置控制台控制处理程序为console_ctrl_handler
        auto console_ctrl_handler = +[](DWORD ctrl_type) -> BOOL {
// 如果控制类型为CTRL_C_EVENT，则调用sigint_handler(SIGINT)并返回true，否则返回false
return (ctrl_type == CTRL_C_EVENT) ? (sigint_handler(SIGINT), true) : false;
};
// 设置控制台控制处理程序
SetConsoleCtrlHandler(reinterpret_cast<PHANDLER_ROUTINE>(console_ctrl_handler), true);

// 记录交互模式开启的信息
LOG_TEE("%s: interactive mode on.\n", __func__);

// 如果参数中包含输入前缀BOS，则记录信息
if (params.input_prefix_bos) {
    LOG_TEE("Input prefix with BOS\n");
}

// 如果输入前缀不为空，则记录信息
if (!params.input_prefix.empty()) {
    LOG_TEE("Input prefix: '%s'\n", params.input_prefix.c_str());
}

// 如果输入后缀不为空，则记录信息
if (!params.input_suffix.empty()) {
    LOG_TEE("Input suffix: '%s'\n", params.input_suffix.c_str());
}
// 记录采样信息
LOG_TEE("sampling: \n%s\n", llama_sampling_print(sparams).c_str());
    # 打印生成的上下文数量、批次数量、预测数量和保留数量
    LOG_TEE("generate: n_ctx = %d, n_batch = %d, n_predict = %d, n_keep = %d\n", n_ctx, params.n_batch, params.n_predict, params.n_keep);
    # 打印空行
    LOG_TEE("\n\n");

    # 打印提示信息，进入填充模式
    LOG_TEE("\n#####  Infill mode  #####\n\n");
    # 如果参数中包含填充标志
    if (params.infill) {
        # 打印提示信息，无需指定'--infill'，始终运行填充
        printf("\n************\n");
        printf("no need to specify '--infill', always running infill\n");
        printf("************\n\n");
    }
    # 如果参数中包含交互标志
    if (params.interactive) {
        # 定义控制信息
        const char *control_message;
        # 如果参数中包含多行输入标志
        if (params.multiline_input) {
            control_message = " - To return control to LLaMa, end your input with '\\'.\n"
                              " - To return control without starting a new line, end your input with '/'.\n";
        } else {
            control_message = " - Press Return to return control to LLaMa.\n"
                              " - To return control without starting a new line, end your input with '/'.\n"
                              " - If you want to submit another line, end your input with '\\'.\n";
        }
        # 打印运行交互模式的提示信息
        LOG_TEE("== Running in interactive mode. ==\n");
#if defined (__unix__) || (defined (__APPLE__) && defined (__MACH__)) || defined (_WIN32)
        // 如果是在 Unix 系统、苹果系统或者 Windows 系统下，打印提示信息
        LOG_TEE(       " - Press Ctrl+C to interject at any time.\n");
#endif
        // 打印控制信息
        LOG_TEE(       "%s\n", control_message);

        // 设置交互标志为参数中的交互优先标志
        is_interacting = params.interactive_first;
    }

    // 设置输入回显为真
    bool input_echo           = true;

    // 初始化过去的输入数量为 0
    int n_past             = 0;
    // 设置剩余预测数量为参数中的预测数量
    int n_remain           = params.n_predict;
    // 初始化已消耗的数量为 0
    int n_consumed         = 0;
    // 初始化过去的指导数量为 0
    int n_past_guidance    = 0;

    // 初始化输入令牌向量
    std::vector<int>   input_tokens;  g_input_tokens  = &input_tokens;
    // 初始化输出令牌向量
    std::vector<int>   output_tokens; g_output_tokens = &output_tokens;
    // 初始化输出字符串流
    std::ostringstream output_ss;     g_output_ss     = &output_ss;

    // 我们将要做的第一件事是输出提示信息，因此相应地设置颜色
    // （这句注释可能是对下一行代码的解释）
    // 设置控制台显示模式为提示模式
    console::set_display(console::prompt);

    // 定义嵌入向量和嵌入向量指导的向量
    std::vector<llama_token> embd;
    std::vector<llama_token> embd_guidance;

    // 初始化采样上下文
    struct llama_sampling_context * ctx_sampling = llama_sampling_init(sparams);

    // 当剩余数量不为零或者参数中包含交互模式时执行循环
    while (n_remain != 0 || params.interactive) {
        // 预测
        if (!embd.empty()) {
            // 注意：这里的 n_ctx - 4 是为了匹配通过 --prompt 或 --file 处理命令行提示的逻辑，它们使用相同的值。
            int max_embd_size = n_ctx - 4;

            // 确保输入不超过上下文大小，必要时通过截断 embd 来实现
            if ((int) embd.size() > max_embd_size) {
                const int skipped_tokens = (int) embd.size() - max_embd_size;
                embd.resize(max_embd_size);

                // 设置控制台显示模式为错误模式
                console::set_display(console::error);
// 打印输入过长的提示信息，包括跳过的 token 数量
printf("<<input too long: skipped %d token%s>>", skipped_tokens, skipped_tokens != 1 ? "s" : "");
// 设置控制台显示为重置状态
console::set_display(console::reset);
// 刷新标准输出流
fflush(stdout);

// 通过上下文交换实现无限文本生成
// 如果上下文用尽：
// - 从原始提示中获取前 n_keep 个 token（通过 n_past）
// - 获取最后 (n_ctx - n_keep) 一半的 token，并分批重新计算 logits
if (n_past + (int) embd.size() + std::max<int>(0, guidance_offset) > n_ctx) {
    // 如果 n_predict 为 -2，则停止生成
    if (params.n_predict == -2) {
        LOG_TEE("\n\n%s: context full and n_predict == -%d => stopping\n", __func__, params.n_predict);
        break;
    }

    // 计算剩余的 token 数量
    const int n_left    = n_past - params.n_keep - 1;
    // 计算需要丢弃的 token 数量
    const int n_discard = n_left/2;

    // 打印上下文交换的相关信息
    LOG("context full, swapping: n_past = %d, n_left = %d, n_ctx = %d, n_keep = %d, n_discard = %d\n",
        n_past, n_left, n_ctx, params.n_keep, n_discard);
}
# 从缓存中删除指定范围的键值对
llama_kv_cache_seq_rm(ctx, 0, params.n_keep + 1, params.n_keep + n_discard + 1);
# 将缓存中的键值对序列整体向左移动指定的距离
llama_kv_cache_seq_shift(ctx, 0, params.n_keep + 1 + n_discard, n_past, -n_discard);

# 更新 n_past 的值
n_past -= n_discard;

# 如果存在 ctx_guidance，则更新 n_past_guidance 的值
if (ctx_guidance) {
    n_past_guidance -= n_discard;
}

# 打印日志，显示 n_past 和 n_past_guidance 的值
LOG("after swap: n_past = %d, n_past_guidance = %d\n", n_past, n_past_guidance);

# 打印日志，显示 embd 的内容
LOG("embd: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd).c_str());

# 如果存在 ctx_guidance，则执行以下操作
if (ctx_guidance) {
                // 初始化输入大小为0
                int input_size = 0;
                // 初始化输入缓冲区为空
                llama_token * input_buf = NULL;

                // 如果过去的指导小于指导输入的大小
                if (n_past_guidance < (int) guidance_inp.size()) {
                    // 指导上下文应该具有相同的数据，进行以下修改：
                    //
                    // * 替换初始提示
                    // * 将所有内容向后移动guidance_offset
                    embd_guidance = guidance_inp;
                    // 如果embd的起始位置加上原始提示的长度小于embd的结束位置
                    if (embd.begin() + original_prompt_len < embd.end()) {
                        // 在embd_guidance的末尾插入embd中从原始提示长度开始到结束的内容
                        embd_guidance.insert(
                            embd_guidance.end(),
                            embd.begin() + original_prompt_len,
                            embd.end()
                        );
                    }

                    // 将embd_guidance的数据指针赋值给input_buf
                    input_buf  = embd_guidance.data();
                    // 将embd_guidance的大小赋值给input_size
                    input_size = embd_guidance.size();
// 如果存在嵌入指导上下文，则记录指导上下文的信息
LOG("guidance context: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd_guidance).c_str());
// 如果不存在嵌入指导上下文，则将输入缓冲区和大小设置为嵌入数据的内容和大小
} else {
    input_buf  = embd.data();
    input_size = embd.size();
}

// 对输入数据进行批处理，每次处理params.n_batch大小的数据
for (int i = 0; i < input_size; i += params.n_batch) {
    int n_eval = std::min(input_size - i, params.n_batch);
    // 如果调用llama_decode函数失败，则记录错误信息并返回1
    if (llama_decode(ctx_guidance, llama_batch_get_one(input_buf + i, n_eval, n_past_guidance, 0))) {
        LOG_TEE("%s : failed to eval\n", __func__);
        return 1;
    }
    // 更新已处理的指导数据数量
    n_past_guidance += n_eval;
}

// 对嵌入数据进行批处理，每次处理params.n_batch大小的数据
for (int i = 0; i < (int) embd.size(); i += params.n_batch) {
    int n_eval = (int) embd.size() - i;
    // 如果剩余的数据量大于params.n_batch，则继续处理
    if (n_eval > params.n_batch) {
                // 设置评估的批次大小为参数中指定的批次大小
                n_eval = params.n_batch;
                }

                // 打印评估结果
                LOG("eval: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, embd).c_str());

                // 如果 llama_decode 函数返回非零值，表示评估失败
                if (llama_decode(ctx, llama_batch_get_one(&embd[i], n_eval, n_past, 0))) {
                    LOG_TEE("%s : failed to eval\n", __func__);
                    return 1;
                }

                // 更新已评估的数据量
                n_past += n_eval;

                // 打印已评估的数据量
                LOG("n_past = %d\n", n_past);
            }

        }

        // 清空 embd 和 embd_guidance 容器
        embd.clear();
        embd_guidance.clear();
        // 如果嵌入输入的大小小于等于已消耗的数量，并且不是交互式的
        if ((int) embd_inp.size() <= n_consumed && !is_interacting) {

            // 从采样器中获取一个 llama_token
            const llama_token id = llama_sampling_sample(ctx_sampling, ctx, ctx_guidance);

            // 在采样器中接受该 llama_token
            llama_sampling_accept(ctx_sampling, ctx, id, true);

            // 打印上一个 llama_token 的信息到日志
            LOG("last: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, ctx_sampling->prev).c_str());

            // 将获取的 llama_token 添加到 embd 中
            embd.push_back(id);

            // 设置输入回显标志为 true
            input_echo = true;

            // 减少剩余的采样预算
            --n_remain;

            // 打印剩余的采样预算到日志
            LOG("n_remain: %d\n", n_remain);
        } else {
            // 如果提示或交互中仍有一些用户输入，将其转发到处理
            LOG("embd_inp.size(): %d, n_consumed: %d\n", (int) embd_inp.size(), n_consumed);
            // 当嵌入输入的大小大于已消耗的数量时，执行循环
            while ((int) embd_inp.size() > n_consumed) {
                // 将嵌入输入中的元素添加到嵌入向量中
                embd.push_back(embd_inp[n_consumed]);

                // 将提示推送到采样上下文中，以便稍后应用重复惩罚
                // 对于提示，我们不应用语法规则
                llama_sampling_accept(ctx_sampling, ctx, embd_inp[n_consumed], false);

                // 增加已消耗的数量
                ++n_consumed;
                // 如果嵌入向量的大小达到批量大小，则跳出循环
                if ((int) embd.size() >= params.n_batch) {
                    break;
                }
            }
        }

        // 显示文本
        if (input_echo) {
            // 遍历嵌入向量中的元素
            for (auto id : embd) {
                // 将标记转换为字符串，并打印出来
                const std::string token_str = llama_token_to_piece(ctx, id);
                printf("%s", token_str.c_str());
                // 如果嵌入向量的大小大于1，则将其id添加到输入标记列表中
                if (embd.size() > 1) {
                    input_tokens.push_back(id);
                } else {
                    // 否则将其id添加到输出标记列表中，并将标记字符串添加到输出流中
                    output_tokens.push_back(id);
                    output_ss << token_str;
                }
            }
            // 刷新标准输出流
            fflush(stdout);
        }
        // 如果存在输入回显并且嵌入输入大小等于已消耗的数量，则重置颜色为默认值
        if (input_echo && (int) embd_inp.size() == n_consumed) {
            console::set_display(console::reset);
        }

        // 如果当前没有处理排队的输入；
        if ((int) embd_inp.size() <= n_consumed) {

            // 处理infill模式中的eot标记
            if ((llama_sampling_last(ctx_sampling) == llama_token_eot(model) || is_interacting) && params.interactive){
                // 如果正在交互并且不是交互模式的第一个输入
                if(is_interacting && !params.interactive_first) {
// 打印一个结束符号的标记
printf("%s", llama_token_to_piece(ctx, llama_token_eot(model)).c_str());
// 刷新标准输出流
fflush(stdout);
// 设置控制台显示为用户输入模式
console::set_display(console::user_input);
// 定义字符串变量buffer和line
std::string buffer;
std::string line;
// 定义布尔变量another_line并初始化为true
bool another_line=true;
// 通过标准输入设置一个新的前缀
do {
    // 从控制台读取一行输入，并将结果存入line中，同时更新another_line的值
    another_line = console::readline(line, params.multiline_input);
    // 将line的内容追加到buffer中
    buffer += line;
} while (another_line);
// 检查是否得到了空行，如果是，则使用旧的输入
if (!buffer.empty() && !(buffer.length() == 1 && buffer[0] == '\n')) {
    // 将buffer的内容赋值给params.input_prefix
    params.input_prefix = buffer;
}
// 清空buffer
buffer.clear();
// 通过标准输入设置一个新的后缀
                // 使用循环从控制台读取输入的多行文本，直到没有更多行为止
                do {
                    another_line = console::readline(line, params.multiline_input);
                    buffer += line;
                } while (another_line);
                // 检查是否得到了空行
                if (!buffer.empty() && !(buffer.length() == 1 && buffer[0] == '\n')) {
                    // 如果输入不为空且不仅包含换行符，则将其赋给params.input_suffix
                    params.input_suffix = buffer;
                }
                // 清空缓冲区
                buffer.clear();
                // 输入结束，重置颜色
                console::set_display(console::reset);

                if (params.escape) {
                    // 处理转义序列，对于初始提示，这是在common.cpp中加载参数时完成的，但对于交互模式，我们需要在这里完成
                    process_escapes(params.input_prefix);
                    process_escapes(params.input_suffix);
                }
                // 根据参数判断是否需要移除输入后缀的开头空格
                suff_rm_leading_spc = params.escape;
                if (suff_rm_leading_spc && params.input_suffix.find_first_of(' ') == 0 && params.input_suffix.size() > 1) {
                    // 如果需要移除开头空格，则执行移除操作
                    params.input_suffix.erase(0, 1);
// 如果需要移除后缀的前导空格，则设置标志位为false
suff_rm_leading_spc = false;
// 对输入前缀和后缀进行分词
std::vector<llama_token> inp_pfx = ::llama_tokenize(ctx, params.input_prefix, false);
std::vector<llama_token> inp_sfx = ::llama_tokenize(ctx, params.input_suffix, false);
// 如果需要移除后缀的前导空格，并且后缀的第一个词是空格，则移除后缀的第一个词
if (suff_rm_leading_spc && inp_sfx[0] == space_token) {
    inp_sfx.erase(inp_sfx.begin());
}
// 在输入前缀的最前面插入前缀标记
inp_pfx.insert(inp_pfx.begin(), llama_token_prefix(model));
// 如果需要添加开始符号，则在输入前缀的最前面再插入开始符号
if (add_bos) {
    inp_pfx.insert(inp_pfx.begin(), llama_token_bos(model));
}
// 在输入后缀的最前面插入后缀标记
inp_sfx.insert(inp_sfx.begin(), llama_token_suffix(model));
// 将输入前缀和后缀组合成嵌入输入
embd_inp = inp_pfx;
embd_inp.insert(embd_inp.end(), inp_sfx.begin(), inp_sfx.end());
embd_inp.push_back(llama_token_middle(model));
// 清空嵌入和嵌入指导
embd.clear();
embd_guidance.clear();
// 设置剩余预测数量和过去数量为0
n_remain = params.n_predict;
n_past = 0;
                // 初始化已消耗字符数为0
                n_consumed = 0;
                // 如果是交互模式，记录日志
                // LOG_TEE("took new input\n");
                // 标记为非交互状态
                is_interacting = false;
            }
            // 处理交互模式下的文本结束标记
            else if (llama_sampling_last(ctx_sampling) == llama_token_eos(model)) {
                // 记录日志
                LOG("found EOS token\n");

                // 如果是交互模式
                if (params.interactive) {
                    // 标记为交互状态
                    is_interacting = true;
                    // 输出换行符
                    printf("\n");
                    // 设置控制台显示为用户输入模式
                    console::set_display(console::user_input);
                    // 刷新标准输出
                    fflush(stdout);
               }
            }

            // 如果已经有过输入，并且是交互模式，并且不是交互参数
            if (n_past > 0 && is_interacting && !params.interactive) {
                // 记录日志
                LOG("waiting for user input\n");
// 如果输入前缀存在
if (params.input_prefix_bos) {
    // 输出日志，添加输入前缀的 BOS（Beginning of Sentence）标记
    LOG("adding input prefix BOS token\n");
    // 将 BOS 标记添加到 embd_inp 向量中
    embd_inp.push_back(llama_token_bos(model));
}

// 创建一个空字符串 buffer
std::string buffer;
// 如果输入前缀不为空
if (!params.input_prefix.empty()) {
    // 输出日志，追加输入前缀
    LOG("appending input prefix: '%s'\n", params.input_prefix.c_str());
    // 将输入前缀添加到 buffer 中
    buffer += params.input_prefix;
    // 打印 buffer 的内容
    printf("%s", buffer.c_str());
}

// 创建一个空字符串 line
std::string line;
// 创建一个布尔值变量 another_line，并初始化为 true
bool another_line = true;
// 循环读取输入的每一行
do {
    // 从控制台读取一行输入，并将结果存储在 line 中，同时检查是否还有另一行输入
    another_line = console::readline(line, params.multiline_input);
    // 将读取的行添加到 buffer 中
    buffer += line;
} while (another_line);

// 输入结束，重置颜色
// done taking input, reset color
// 设置控制台显示为默认状态
console::set_display(console::reset);

// 只有在输入缓冲区非空的情况下才将标记添加到embd中
// 输入空行会让用户将控制权传回
if (buffer.length() > 1) {
    // 如果输入后缀不为空，则追加输入后缀
    if (!params.input_suffix.empty()) {
        LOG("appending input suffix: '%s'\n", params.input_suffix.c_str());
        buffer += params.input_suffix;
        printf("%s", params.input_suffix.c_str());
    }

    LOG("buffer: '%s'\n", buffer.c_str());

    // 记录embd_inp的原始大小
    const size_t original_size = embd_inp.size();

    // 使用llama_tokenize函数对输入进行标记化
    const auto line_inp = ::llama_tokenize(ctx, buffer, false);
    LOG("input tokens: %s\n", LOG_TOKENS_TOSTR_PRETTY(ctx, line_inp).c_str());

    // 将标记化后的输入插入到embd_inp的末尾
    embd_inp.insert(embd_inp.end(), line_inp.begin(), line_inp.end());
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

// 如果过去的数量大于0
if (n_past > 0) {
    // 如果正在交互，则重置采样
    if (is_interacting) {
        llama_sampling_reset(ctx_sampling);
    }
}
        // 设置交互标志为假
        is_interacting = false;
        // 如果嵌入不为空且最后一个元素是结束符，并且不是交互模式，则跳出循环
        if (!embd.empty() && embd.back() == llama_token_eos(model) && !params.interactive) {
            break;
        }

        // 在交互模式下，遵循最大标记数，并在达到时返回到用户输入。当 n_predict == -1（无限）或 -2（停止在上下文大小）时，我们跳过此逻辑。
        if (params.interactive && n_remain <= 0 && params.n_predict >= 0) {
            n_remain = params.n_predict;
            // 设置交互标志为真
            is_interacting = true;
        }
    }
    // 如果不是交互模式且剩余标记数小于等于0，则打印结束符并刷新输出
    if (!params.interactive && n_remain <= 0) {
        printf("%s", llama_token_to_piece(ctx, llama_token_eot(model)).c_str());
        fflush(stdout);
    }
    // 打印程序执行时间
    llama_print_timings(ctx);
    // 写入日志文件
    write_logfile(ctx, params, model, input_tokens, output_ss.str(), output_tokens);

    // 如果存在指导上下文，释放内存
    if (ctx_guidance) { llama_free(ctx_guidance); }
    // 释放上下文内存
    llama_free(ctx);
    // 释放模型内存
    llama_free_model(model);

    // 释放采样上下文内存
    llama_sampling_free(ctx_sampling);
    // 释放后端内存
    llama_backend_free();

    // 如果未禁用日志，记录日志结束
#ifndef LOG_DISABLE_LOGS
    LOG_TEE("Log end\n");
#endif // LOG_DISABLE_LOGS

    // 返回 0 表示正常结束
    return 0;
}
```