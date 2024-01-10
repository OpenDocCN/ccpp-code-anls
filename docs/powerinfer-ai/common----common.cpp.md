# `PowerInfer\common\common.cpp`

```
#include "common.h"
#include "llama.h"

#include <algorithm>
#include <cassert>
#include <cmath>
#include <cstring>
#include <ctime>
#include <fstream>
#include <iterator>
#include <iostream>
#include <regex>
#include <sstream>
#include <string>
#include <unordered_set>
#include <vector>
#include <cinttypes>

#if defined(__APPLE__) && defined(__MACH__)
#include <sys/types.h>
#include <sys/sysctl.h>
#endif

#if defined(_WIN32)
#define WIN32_LEAN_AND_MEAN
#ifndef NOMINMAX
#   define NOMINMAX
#endif
#include <codecvt>
#include <locale>
#include <windows.h>
#include <fcntl.h>
#include <io.h>
#else
#include <sys/ioctl.h>
#include <sys/stat.h>
#include <unistd.h>
#endif

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 获取物理核心数
int32_t get_num_physical_cores() {
#ifdef __linux__
    // 枚举线程同级，条目数即为核心数
    std::unordered_set<std::string> siblings;
    for (uint32_t cpu=0; cpu < UINT32_MAX; ++cpu) {
        std::ifstream thread_siblings("/sys/devices/system/cpu"
            + std::to_string(cpu) + "/topology/thread_siblings");
        if (!thread_siblings.is_open()) {
            break; // 没有更多的 CPU
        }
        std::string line;
        if (std::getline(thread_siblings, line)) {
            siblings.insert(line);
        }
    }
    if (!siblings.empty()) {
        return static_cast<int32_t>(siblings.size());
    }
#elif defined(__APPLE__) && defined(__MACH__)
    int32_t num_physical_cores;
    size_t len = sizeof(num_physical_cores);
    int result = sysctlbyname("hw.perflevel0.physicalcpu", &num_physical_cores, &len, NULL, 0);
    if (result == 0) {
        return num_physical_cores;
    }
    result = sysctlbyname("hw.physicalcpu", &num_physical_cores, &len, NULL, 0);
    if (result == 0) {
        return num_physical_cores;
    }
#elif defined(_WIN32)
    //TODO: Implement
#endif
    unsigned int n_threads = std::thread::hardware_concurrency();
    # 如果线程数大于0，则判断线程数是否小于等于4，如果是则返回线程数，否则返回线程数的一半
    return n_threads > 0 ? (n_threads <= 4 ? n_threads : n_threads / 2) : 4;
}

void process_escapes(std::string& input) {
    // 获取输入字符串的长度
    std::size_t input_len = input.length();
    // 初始化输出字符串的索引
    std::size_t output_idx = 0;

    // 遍历输入字符串
    for (std::size_t input_idx = 0; input_idx < input_len; ++input_idx) {
        // 判断是否为转义字符
        if (input[input_idx] == '\\' && input_idx + 1 < input_len) {
            // 处理转义字符
            switch (input[++input_idx]) {
                case 'n':  input[output_idx++] = '\n'; break;  // 换行符
                case 'r':  input[output_idx++] = '\r'; break;  // 回车符
                case 't':  input[output_idx++] = '\t'; break;  // 制表符
                case '\'': input[output_idx++] = '\''; break;   // 单引号
                case '\"': input[output_idx++] = '\"'; break;   // 双引号
                case '\\': input[output_idx++] = '\\'; break;   // 反斜杠
                case 'x':
                    // 处理十六进制转义字符
                    if (input_idx + 2 < input_len) {
                        const char x[3] = { input[input_idx + 1], input[input_idx + 2], 0 };
                        char *err_p = nullptr;
                        // 将十六进制字符串转换为整数
                        const long val = std::strtol(x, &err_p, 16);
                        if (err_p == x + 2) {
                            input_idx += 2;
                            input[output_idx++] = char(val);
                            break;
                        }
                    }
                    // 默认情况
                default:   input[output_idx++] = '\\';
                           input[output_idx++] = input[input_idx]; break;
            }
        } else {
            input[output_idx++] = input[input_idx];
        }
    }

    // 调整输入字符串的长度
    input.resize(output_idx);
}

bool gpt_params_parse(int argc, char ** argv, gpt_params & params) {
    bool result = true;
    try {
        // 解析参数
        if (!gpt_params_parse_ex(argc, argv, params)) {
            // 打印用法信息并退出
            gpt_print_usage(argc, argv, gpt_params());
            exit(0);
        }
    }
    catch (const std::invalid_argument & ex) {
        // 捕获无效参数异常，打印错误信息并退出
        fprintf(stderr, "%s\n", ex.what());
        gpt_print_usage(argc, argv, gpt_params());
        exit(1);
    }
    return result;
}
bool gpt_params_parse_ex(int argc, char ** argv, gpt_params & params) {
    // 初始化参数无效标志
    bool invalid_param = false;
    // 初始化参数字符串
    std::string arg;
    // 初始化参数前缀字符串
    const std::string arg_prefix = "--";
    // 获取采样参数的引用
    llama_sampling_params & sparams = params.sparams;

    // 如果 LLAMA_SUPPORTS_GPU_OFFLOAD 宏被定义
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
            // 将参数转换为整数并赋值给 params.n_gpu_layers
            params.n_gpu_layers = std::stoi(argv[i]);
#else
            // 打印警告信息，表示未使用 GPU 加速支持，--n-gpu-layers 选项将被忽略
            fprintf(stderr, "warning: not compiled with GPU offload support, --n-gpu-layers option will be ignored\n");
            // 打印警告信息，提醒查看主 README.md 以获取关于启用 GPU BLAS 支持的信息
            fprintf(stderr, "warning: see main README.md for information on enabling GPU BLAS support\n");
#endif
        } else if (arg == "--gpu-layers-draft" || arg == "-ngld" || arg == "--n-gpu-layers-draft") {
            // 如果参数匹配，则执行以下代码块
            if (++i >= argc) {
                // 参数不足，设置参数无效标志
                invalid_param = true;
                // 跳出循环
                break;
            }
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
            // 将参数转换为整数并赋值给 params.n_gpu_layers_draft
            params.n_gpu_layers_draft = std::stoi(argv[i]);
#else
            // 打印警告信息，表示未使用 GPU 加速支持，--n-gpu-layers-draft 选项将被忽略
            fprintf(stderr, "warning: not compiled with GPU offload support, --n-gpu-layers-draft option will be ignored\n");
            // 打印警告信息，提醒查看主 README.md 以获取关于启用 GPU BLAS 支持的信息
            fprintf(stderr, "warning: see main README.md for information on enabling GPU BLAS support\n");
#endif
        } else if (arg == "--main-gpu" || arg == "-mg") {
            // 如果参数匹配，则执行以下代码块
            if (++i >= argc) {
                // 参数不足，设置参数无效标志
                invalid_param = true;
                // 跳出循环
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 将参数转换为整数并赋值给 params.main_gpu
            params.main_gpu = std::stoi(argv[i]);
#else
            // 打印警告信息，表示 llama.cpp 未使用 cuBLAS 编译，无法设置主 GPU
            fprintf(stderr, "warning: llama.cpp was compiled without cuBLAS. It is not possible to set a main GPU.\n");
#endif
        } else if (arg == "--tensor-split" || arg == "-ts") {
            // 如果参数匹配，则执行以下代码块
            if (++i >= argc) {
                // 参数不足，设置参数无效标志
                invalid_param = true;
                // 跳出循环
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 从命令行参数中获取下一个参数
            std::string arg_next = argv[i];

            // 通过逗号和斜杠分割字符串
            const std::regex regex{R"([,/]+)"};
            std::sregex_token_iterator it{arg_next.begin(), arg_next.end(), regex, -1};
            // 将分割后的字符串存储到 vector 中
            std::vector<std::string> split_arg{it, {}};
            // 确保分割后的字符串数量不超过 LLAMA_MAX_DEVICES
            GGML_ASSERT(split_arg.size() <= LLAMA_MAX_DEVICES);

            // 遍历分割后的字符串，将其转换为浮点数并存储到参数的 tensor_split 数组中
            for (size_t i = 0; i < LLAMA_MAX_DEVICES; ++i) {
                if (i < split_arg.size()) {
                    params.tensor_split[i] = std::stof(split_arg[i]);
                } else {
                    params.tensor_split[i] = 0.0f;
                }
            }
#else
            // 如果未使用 cuBLAS，则输出警告信息
            fprintf(stderr, "warning: llama.cpp was compiled without cuBLAS. It is not possible to set a tensor split.\n");
#endif // GGML_USE_CUBLAS
        } else if (arg == "--vram-budget") {
            // 如果下一个参数是 VRAM 预算
            if (++i >= argc) {
                // 如果没有下一个参数，则标记为无效参数并跳出循环
                invalid_param = true;
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 如果使用 cuBLAS，则将下一个参数转换为浮点数并存储到参数的 vram_budget_gb 中
            params.vram_budget_gb = std::stof(argv[i]);
#else
            // 如果未使用 cuBLAS，则输出警告信息
            fprintf(stderr, "warning: PowerInfer was compiled without cuBLAS. It is not possible to set a VRAM budget.\n");
#endif
        } else if (arg == "--no-mul-mat-q" || arg == "-nommq") {
#ifdef GGML_USE_CUBLAS
            // 如果使用 cuBLAS，则将 mul_mat_q 参数设置为 false
            params.mul_mat_q = false;
#else
            // 如果未使用 cuBLAS，则输出警告信息
            fprintf(stderr, "warning: llama.cpp was compiled without cuBLAS. Disabling mul_mat_q kernels has no effect.\n");
#ifndef LOG_DISABLE_LOGS
        // 如果日志未被禁用，则解析日志参数
        } else if ( log_param_single_parse( argv[i] ) ) {
            // 什么也不做，log_param_single_parse 自动执行其操作
            // 并返回是否找到匹配并解析
        } else if ( log_param_pair_parse( /*check_but_dont_parse*/ true, argv[i] ) ) {
            // 我们有一个需要参数的匹配已知参数，
            // 现在我们需要检查这个 argv 后面是否有任何内容
            // 并标记无效参数或解析它。
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            if( !log_param_pair_parse( /*check_but_dont_parse*/ false, argv[i-1], argv[i]) ) {
                invalid_param = true;
                break;
            }
        // 结束解析日志参数
#endif // LOG_DISABLE_LOGS
        } else {
            throw std::invalid_argument("error: unknown argument: " + arg);
        }
    }
    if (invalid_param) {
        throw std::invalid_argument("error: invalid parameter for argument: " + arg);
    }
    if (params.prompt_cache_all &&
            (params.interactive || params.interactive_first ||
             params.instruct)) {

        throw std::invalid_argument("error: --prompt-cache-all not supported in interactive mode yet\n");
    }

    if (params.escape) {
        process_escapes(params.prompt);
        process_escapes(params.input_prefix);
        process_escapes(params.input_suffix);
        process_escapes(sparams.cfg_negative_prompt);
        for (auto & antiprompt : params.antiprompt) {
            process_escapes(antiprompt);
        }
    }

    return true;
}

void gpt_print_usage(int /*argc*/, char ** argv, const gpt_params & params) {
    const llama_sampling_params & sparams = params.sparams;

    printf("\n");
    printf("usage: %s [options]\n", argv[0]);
    printf("\n");
    printf("options:\n");
    # 打印帮助信息和退出
    printf("  -h, --help            show this help message and exit\n");
    # 运行交互模式
    printf("  -i, --interactive     run in interactive mode\n");
    # 运行交互模式并立即等待输入
    printf("  --interactive-first   run in interactive mode and wait for input right away\n");
    # 运行指令模式（与 Alpaca 模型一起使用）
    printf("  -ins, --instruct      run in instruction mode (use with Alpaca models)\n");
    # 允许您写入或粘贴多行而无需在每行结尾加上'\'符号
    printf("  --multiline-input     allows you to write or paste multiple lines without ending each in '\\'\n");
    # 停止在指定提示处生成，返回交互模式下的控制（可以多次指定以指定多个提示）
    printf("  -r PROMPT, --reverse-prompt PROMPT\n");
    printf("                        halt generation at PROMPT, return control in interactive mode\n");
    printf("                        (can be specified more than once for multiple prompts).\n");
    # 为了区分提示和用户输入与生成的输出，对输出进行着色
    printf("  --color               colorise output to distinguish prompt and user input from generations\n");
    # 随机数生成器种子（默认值：-1，对于小于0的值使用随机种子）
    printf("  -s SEED, --seed SEED  RNG seed (default: -1, use random seed for < 0)\n");
    # 生成过程中使用的线程数（默认值：params.n_threads）
    printf("  -t N, --threads N     number of threads to use during generation (default: %d)\n", params.n_threads);
    # 批处理和提示处理期间使用的线程数（默认值：与--threads相同）
    printf("  -tb N, --threads-batch N\n");
    printf("                        number of threads to use during batch and prompt processing (default: same as --threads)\n");
    # 用于开始生成的提示（默认值：空）
    printf("  -p PROMPT, --prompt PROMPT\n");
    printf("                        prompt to start generation with (default: empty)\n");
    # 处理提示转义序列（\n，\r，\t，\'，\"，\\）
    printf("  -e, --escape          process prompt escapes sequences (\\n, \\r, \\t, \\', \\\", \\\\)\n");
    # 用于缓存提示状态以加快启动速度的文件（默认值：无）
    printf("  --prompt-cache FNAME  file to cache prompt state for faster startup (default: none)\n");
    # 如果指定，还将用户输入和生成结果保存到缓存中
    printf("  --prompt-cache-all    if specified, saves user input and generations to cache as well.\n");
    # 如果指定，使用提示缓存但不更新它
    printf("  --prompt-cache-ro     if specified, uses the prompt cache but does not update it.\n");
    # 以随机提示开始
    printf("  --random-prompt       start with a randomized prompt.\n");
    # 打印提示信息，指示在用户输入前添加BOS前缀，其前缀为`--in-prefix`字符串
    printf("  --in-prefix-bos       prefix BOS to user inputs, preceding the `--in-prefix` string\n");
    # 打印提示信息，指示用户输入前添加的字符串，默认为空
    printf("  --in-prefix STRING    string to prefix user inputs with (default: empty)\n");
    # 打印提示信息，指示用户输入后添加的字符串，默认为空
    printf("  --in-suffix STRING    string to suffix after user inputs with (default: empty)\n");
    # 打印提示信息，指示生成开始的提示文件
    printf("  -f FNAME, --file FNAME\n");
    printf("                        prompt file to start generation.\n");
    # 打印提示信息，指示要预测的标记数量，默认为params.n_predict
    printf("  -n N, --n-predict N   number of tokens to predict (default: %d, -1 = infinity, -2 = until context filled)\n", params.n_predict);
    # 打印提示信息，指示提示上下文的大小，默认为params.n_ctx
    printf("  -c N, --ctx-size N    size of the prompt context (default: %d, 0 = loaded from model)\n", params.n_ctx);
    # 打印提示信息，指示提示处理的批量大小，默认为params.n_batch
    printf("  -b N, --batch-size N  batch size for prompt processing (default: %d)\n", params.n_batch);
    # 打印提示信息，指示使用top-k抽样，默认为sparams.top_k
    printf("  --top-k N             top-k sampling (default: %d, 0 = disabled)\n", sparams.top_k);
    # 打印提示信息，指示使用top-p抽样，默认为sparams.top_p
    printf("  --top-p N             top-p sampling (default: %.1f, 1.0 = disabled)\n", (double)sparams.top_p);
    # 打印提示信息，指示使用min-p抽样，默认为sparams.min_p
    printf("  --min-p N             min-p sampling (default: %.1f, 0.0 = disabled)\n", (double)sparams.min_p);
    # 打印提示信息，指示使用tail free抽样，默认为sparams.tfs_z
    printf("  --tfs N               tail free sampling, parameter z (default: %.1f, 1.0 = disabled)\n", (double)sparams.tfs_z);
    # 打印提示信息，指示使用局部典型抽样，默认为sparams.typical_p
    printf("  --typical N           locally typical sampling, parameter p (default: %.1f, 1.0 = disabled)\n", (double)sparams.typical_p);
    # 打印提示信息，指示考虑惩罚的最后n个标记，默认为sparams.penalty_last_n
    printf("  --repeat-last-n N     last n tokens to consider for penalize (default: %d, 0 = disabled, -1 = ctx_size)\n", sparams.penalty_last_n);
    # 打印提示信息，指示惩罚重复标记序列，默认为sparams.penalty_repeat
    printf("  --repeat-penalty N    penalize repeat sequence of tokens (default: %.1f, 1.0 = disabled)\n", (double)sparams.penalty_repeat);
    # 打印提示信息，指示重复alpha存在惩罚，默认为sparams.penalty_present
    printf("  --presence-penalty N  repeat alpha presence penalty (default: %.1f, 0.0 = disabled)\n", (double)sparams.penalty_present);
    # 打印提示信息，指示重复alpha频率惩罚，默认为sparams.penalty_freq
    printf("  --frequency-penalty N repeat alpha frequency penalty (default: %.1f, 0.0 = disabled)\n", (double)sparams.penalty_freq);
    # 打印提示信息，指示使用Mirostat抽样
    printf("  --mirostat N          use Mirostat sampling.\n");
    # 打印提示信息，说明 Top K、Nucleus、Tail Free 和 Locally Typical 抽样器在使用时会被忽略
    printf("                        Top K, Nucleus, Tail Free and Locally Typical samplers are ignored if used.\n");
    # 打印提示信息，说明 Mirostat 参数的默认值，0 表示禁用，1 表示 Mirostat，2 表示 Mirostat 2.0
    printf("                        (default: %d, 0 = disabled, 1 = Mirostat, 2 = Mirostat 2.0)\n", sparams.mirostat);
    # 打印提示信息，说明 Mirostat 学习率的默认值
    printf("  --mirostat-lr N       Mirostat learning rate, parameter eta (default: %.1f)\n", (double)sparams.mirostat_eta);
    # 打印提示信息，说明 Mirostat 目标熵的默认值
    printf("  --mirostat-ent N      Mirostat target entropy, parameter tau (default: %.1f)\n", (double)sparams.mirostat_tau);
    # 打印提示信息，说明如何修改完成中出现的令牌的可能性
    printf("  -l TOKEN_ID(+/-)BIAS, --logit-bias TOKEN_ID(+/-)BIAS\n");
    # 打印提示信息，说明 BNF 类似的语法用于限制生成
    printf("  --grammar GRAMMAR     BNF-like grammar to constrain generations (see samples in grammars/ dir)\n");
    # 打印提示信息，说明从文件中读取语法
    printf("  --grammar-file FNAME  file to read grammar from\n");
    # 打印提示信息，说明用于指导的负面提示
    printf("  --cfg-negative-prompt PROMPT\n");
    # 打印提示信息，说明从文件中读取用于指导的负面提示
    printf("  --cfg-negative-prompt-file FNAME\n");
    # 打印提示信息，说明指导强度的默认值
    printf("  --cfg-scale N         strength of guidance (default: %f, 1.0 = disable)\n", sparams.cfg_scale);
    # 打印提示信息，说明 RoPE 频率缩放方法的默认值
    printf("  --rope-scaling {none,linear,yarn}\n");
    # 打印提示信息，说明 RoPE 上下文缩放因子的默认值
    printf("  --rope-scale N        RoPE context scaling factor, expands context by a factor of N\n");
    # 打印提示信息，说明 RoPE 基础频率的默认值
    printf("  --rope-freq-base N    RoPE base frequency, used by NTK-aware scaling (default: loaded from model)\n");
    # 打印提示信息，说明 RoPE 频率缩放因子的默认值
    printf("  --rope-freq-scale N   RoPE frequency scaling factor, expands context by a factor of 1/N\n");
    # 打印 YaRN: original context size of model 参数说明
    printf("  --yarn-orig-ctx N     YaRN: original context size of model (default: 0 = model training context size)\n");
    # 打印 YaRN: extrapolation mix factor 参数说明
    printf("  --yarn-ext-factor N   YaRN: extrapolation mix factor (default: 1.0, 0.0 = full interpolation)\n");
    # 打印 YaRN: scale sqrt(t) or attention magnitude 参数说明
    printf("  --yarn-attn-factor N  YaRN: scale sqrt(t) or attention magnitude (default: 1.0)\n");
    # 打印 YaRN: high correction dim or alpha 参数说明
    printf("  --yarn-beta-slow N    YaRN: high correction dim or alpha (default: %.1f)\n", params.yarn_beta_slow);
    # 打印 YaRN: low correction dim or beta 参数说明
    printf("  --yarn-beta-fast N    YaRN: low correction dim or beta (default: %.1f)\n", params.yarn_beta_fast);
    # 打印 ignore end of stream token and continue generating 参数说明
    printf("  --ignore-eos          ignore end of stream token and continue generating (implies --logit-bias 2-inf)\n");
    # 打印 do not penalize newline token 参数说明
    printf("  --no-penalize-nl      do not penalize newline token\n");
    # 打印 use f32 instead of f16 for memory key+value 参数说明
    printf("  --memory-f32          use f32 instead of f16 for memory key+value (default: disabled)\n");
    # 打印 temperature 参数说明
    printf("  --temp N              temperature (default: %.1f)\n", (double)sparams.temp);
    # 打印 VRAM budget in GiB 参数说明
    printf("  --vram-budget N       VRAM budget in GiB (default: -1, -1 = available VRAM)\n");
    # 打印 return logits for all tokens in the batch 参数说明
    printf("  --logits-all          return logits for all tokens in the batch (default: disabled)\n");
    # 打印 compute HellaSwag score over random tasks from datafile supplied with -f 参数说明
    printf("  --hellaswag           compute HellaSwag score over random tasks from datafile supplied with -f\n");
    # 打印 number of tasks to use when computing the HellaSwag score 参数说明
    printf("  --hellaswag-tasks N   number of tasks to use when computing the HellaSwag score (default: %zu)\n", params.hellaswag_tasks);
    # 打印 number of tokens to keep from the initial prompt 参数说明
    printf("  --keep N              number of tokens to keep from the initial prompt (default: %d, -1 = all)\n", params.n_keep);
    # 打印 number of tokens to draft for speculative decoding 参数说明
    printf("  --draft N             number of tokens to draft for speculative decoding (default: %d)\n", params.n_draft);
    # 打印 max number of chunks to process 参数说明
    printf("  --chunks N            max number of chunks to process (default: %d, -1 = all)\n", params.n_chunks);
    # 打印并格式化输出参数的默认值和说明
    printf("  -np N, --parallel N   number of parallel sequences to decode (default: %d)\n", params.n_parallel);
    printf("  -ns N, --sequences N  number of sequences to decode (default: %d)\n", params.n_sequences);
    printf("  -pa N, --p-accept N   speculative decoding accept probability (default: %.1f)\n", (double)params.p_accept);
    printf("  -ps N, --p-split N    speculative decoding split probability (default: %.1f)\n", (double)params.p_split);
    printf("  -cb, --cont-batching  enable continuous batching (a.k.a dynamic batching) (default: disabled)\n");
    printf("  --mmproj MMPROJ_FILE  path to a multimodal projector file for LLaVA. see examples/llava/README.md\n");
    printf("  --image IMAGE_FILE    path to an image file. use with multimodal models\n");
    # 如果系统支持内存锁定，则打印提示信息
    if (llama_mlock_supported()) {
        printf("  --mlock               force system to keep model in RAM rather than swapping or compressing\n");
    }
    # 如果系统支持内存映射，则打印提示信息
    if (llama_mmap_supported()) {
        printf("  --no-mmap             do not memory-map model (slower load but may reduce pageouts if not using mlock)\n");
    }
    # 打印尝试在某些 NUMA 系统上进行优化的提示信息
    printf("  --numa                attempt optimizations that help on some NUMA systems\n");
    printf("                        if run without this previously, it is recommended to drop the system page cache before using this\n");
    printf("                        see https://github.com/ggerganov/llama.cpp/issues/1437\n");
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
    // 打印GPU支持的选项
    printf("  -ngl N, --n-gpu-layers N\n");
    // 打印存储在VRAM中的层数
    printf("                        number of layers to store in VRAM\n");
    // 打印存储在VRAM中的草稿模型的层数
    printf("  -ngld N, --n-gpu-layers-draft N\n");
    printf("                        number of layers to store in VRAM for the draft model\n");
    // 打印如何在多个GPU上分割张量的选项
    printf("  -ts SPLIT --tensor-split SPLIT\n");
    printf("                        how to split tensors across multiple GPUs, comma-separated list of proportions, e.g. 3,1\n");
    // 打印用于临时和小张量的GPU
    printf("  -mg i, --main-gpu i   the GPU to use for scratch and small tensors\n");
#ifdef GGML_USE_CUBLAS
    // 打印是否使用GGML_CUBLAS_NAME而不是自定义的mul_mat_q GGML_CUDA_NAME内核的选项
    printf("  -nommq, --no-mul-mat-q\n");
    printf("                        use " GGML_CUBLAS_NAME " instead of custom mul_mat_q " GGML_CUDA_NAME " kernels.\n");
    // 打印不推荐使用GGML_CUBLAS_NAME，因为它更慢并且使用更多的VRAM
    printf("                        Not recommended since this is both slower and uses more VRAM.\n");
#endif // GGML_USE_CUBLAS
#endif
    // 打印在生成之前是否打印提示的选项
    printf("  --verbose-prompt      print prompt before generation\n");
    // 打印是否使用基本IO以在子进程和有限控制台中获得更好的兼容性的选项
    printf("  --simple-io           use basic IO for better compatibility in subprocesses and limited consoles\n");
    // 打印应用LoRA适配器的选项（意味着--no-mmap）
    printf("  --lora FNAME          apply LoRA adapter (implies --no-mmap)\n");
    // 打印应用具有用户定义缩放S的LoRA适配器的选项（意味着--no-mmap）
    printf("  --lora-scaled FNAME S apply LoRA adapter with user defined scaling S (implies --no-mmap)\n");
    // 打印用作LoRA适配器修改层基础的可选模型的选项
    printf("  --lora-base FNAME     optional model to use as a base for the layers modified by the LoRA adapter\n");
    // 打印模型路径的选项（默认值：params.model.c_str()）
    printf("  -m FNAME, --model FNAME\n");
    printf("                        model path (default: %s)\n", params.model.c_str());
    // 打印用于推测解码的草稿模型的选项（默认值：params.model.c_str()）
    printf("  -md FNAME, --model-draft FNAME\n");
    printf("                        draft model for speculative decoding (default: %s)\n", params.model.c_str());
    // 打印保存YAML日志的路径的选项（如果未设置则不记录日志）
    printf("  -ld LOGDIR, --logdir LOGDIR\n");
    printf("                        path under which to save YAML logs (no logging if unset)\n");
    // 打印空行
    printf("\n");
#ifndef LOG_DISABLE_LOGS
    // 打印日志的使用情况
    log_print_usage();
#endif // LOG_DISABLE_LOGS
}

// 获取系统信息的函数
std::string get_system_info(const gpt_params & params) {
    // 创建一个字符串流对象
    std::ostringstream os;

    // 将系统信息写入字符串流
    os << "system_info: n_threads = " << params.n_threads;
    // 如果批处理线程数不为-1，则将其写入字符串流
    if (params.n_threads_batch != -1) {
        os << " (n_threads_batch = " << params.n_threads_batch << ")";
    }
    // 将硬件并发线程数和系统信息写入字符串流
    os << " / " << std::thread::hardware_concurrency() << " | " << llama_print_system_info();

    // 返回字符串流中的内容
    return os.str();
}

// 从随机数生成器中获取随机的提示语
std::string gpt_random_prompt(std::mt19937 & rng) {
    // 生成一个0到9的随机数
    const int r = rng() % 10;
    // 根据随机数返回不同的提示语
    switch (r) {
        case 0: return "So";
        case 1: return "Once upon a time";
        case 2: return "When";
        case 3: return "The";
        case 4: return "After";
        case 5: return "If";
        case 6: return "import";
        case 7: return "He";
        case 8: return "She";
        case 9: return "They";
    }

    // 不可达的代码，用于标记不应该执行到这里的情况
    GGML_UNREACHABLE();
}

//
// 模型工具函数
//

// 从 GPT 参数转换为 Llama 模型参数
struct llama_model_params llama_model_params_from_gpt_params(const gpt_params & params) {
    // 获取默认的 Llama 模型参数
    auto mparams = llama_model_default_params();

    // 如果 GPT 参数中指定了 GPU 层数，则更新 Llama 模型参数中的 GPU 层数
    if (params.n_gpu_layers != -1) {
        mparams.n_gpu_layers = params.n_gpu_layers;
    }
    // 更新其他 Llama 模型参数
    mparams.main_gpu        = params.main_gpu;
    mparams.vram_budget_gb  = params.vram_budget_gb;
    mparams.tensor_split    = params.tensor_split;
    mparams.use_mmap        = params.use_mmap;
    mparams.use_mlock       = params.use_mlock;
    mparams.reset_gpu_index = params.reset_gpu_index;
    mparams.disable_gpu_index = params.disale_gpu_index;

    return mparams;
}

// 从 GPT 参数转换为 Llama 上下文参数
struct llama_context_params llama_context_params_from_gpt_params(const gpt_params & params) {
    // 获取默认的 Llama 上下文参数
    auto cparams = llama_context_default_params();

    // 更新 Llama 上下文参数
    cparams.n_ctx             = params.n_ctx;
    cparams.n_batch           = params.n_batch;
    cparams.n_threads         = params.n_threads;
    cparams.n_threads_batch   = params.n_threads_batch == -1 ? params.n_threads : params.n_threads_batch;
    cparams.mul_mat_q         = params.mul_mat_q;
    cparams.seed              = params.seed;
    cparams.f16_kv            = params.memory_f16;
    cparams.logits_all        = params.logits_all;
    cparams.embedding         = params.embedding;
    cparams.rope_scaling_type = params.rope_scaling_type;
    cparams.rope_freq_base    = params.rope_freq_base;
    cparams.rope_freq_scale   = params.rope_freq_scale;
    cparams.yarn_ext_factor   = params.yarn_ext_factor;
    # 将输入参数 params 中的 yarn_attn_factor 赋值给 cparams 中的 yarn_attn_factor
    cparams.yarn_attn_factor  = params.yarn_attn_factor;
    # 将输入参数 params 中的 yarn_beta_fast 赋值给 cparams 中的 yarn_beta_fast
    cparams.yarn_beta_fast    = params.yarn_beta_fast;
    # 将输入参数 params 中的 yarn_beta_slow 赋值给 cparams 中的 yarn_beta_slow
    cparams.yarn_beta_slow    = params.yarn_beta_slow;
    # 将输入参数 params 中的 yarn_orig_ctx 赋值给 cparams 中的 yarn_orig_ctx
    cparams.yarn_orig_ctx     = params.yarn_orig_ctx;

    # 返回更新后的 cparams
    return cparams;
// 清空批处理对象中的令牌数量
void llama_batch_clear(struct llama_batch & batch) {
    batch.n_tokens = 0;
}

// 向批处理对象中添加令牌及其相关信息
void llama_batch_add(
                 struct llama_batch & batch,
                        llama_token   id,
                          llama_pos   pos,
    const std::vector<llama_seq_id> & seq_ids,
                               bool   logits) {
    // 将令牌添加到批处理对象的令牌数组中
    batch.token   [batch.n_tokens] = id;
    // 将位置信息添加到批处理对象的位置数组中
    batch.pos     [batch.n_tokens] = pos,
    // 将序列 ID 数组的大小添加到批处理对象的序列 ID 数量数组中
    batch.n_seq_id[batch.n_tokens] = seq_ids.size();
    // 遍历序列 ID 数组，将每个序列 ID 添加到批处理对象的序列 ID 二维数组中
    for (size_t i = 0; i < seq_ids.size(); ++i) {
        batch.seq_id[batch.n_tokens][i] = seq_ids[i];
    }
    // 将 logits 值添加到批处理对象的 logits 数组中
    batch.logits  [batch.n_tokens] = logits;

    // 增加批处理对象的令牌数量
    batch.n_tokens++;
}

// 从 GPT 参数初始化 LLAMA 模型和上下文对象
std::tuple<struct llama_model *, struct llama_context *> llama_init_from_gpt_params(gpt_params & params) {
    // 从 GPT 参数创建 LLAMA 模型参数
    auto mparams = llama_model_params_from_gpt_params(params);

    // 从文件加载 LLAMA 模型
    llama_model * model  = llama_load_model_from_file(params.model.c_str(), mparams);
    // 如果加载失败，则输出错误信息并返回空指针
    if (model == NULL) {
        fprintf(stderr, "%s: error: failed to load model '%s'\n", __func__, params.model.c_str());
        return std::make_tuple(nullptr, nullptr);
    }

    // 从 GPT 参数创建 LLAMA 上下文参数
    auto cparams = llama_context_params_from_gpt_params(params);
    // 使用模型创建 LLAMA 上下文对象
    llama_context * lctx = llama_new_context_with_model(model, cparams);
    // 如果创建失败，则输出错误信息，释放模型内存，并返回空指针
    if (lctx == NULL) {
        fprintf(stderr, "%s: error: failed to create context with model '%s'\n", __func__, params.model.c_str());
        llama_free_model(model);
        return std::make_tuple(nullptr, nullptr);
    }
    # 遍历 lora_adapter 列表中的每个元素
    for (unsigned int i = 0; i < params.lora_adapter.size(); ++i) {
        # 获取当前元素中的第一个值作为 lora_adapter
        const std::string& lora_adapter = std::get<0>(params.lora_adapter[i]);
        # 获取当前元素中的第二个值作为 lora_scale
        float lora_scale = std::get<1>(params.lora_adapter[i]);
        # 调用 llama_model_apply_lora_from_file 函数，应用 lora 适配器到模型
        int err = llama_model_apply_lora_from_file(model,
                                             lora_adapter.c_str(),
                                             lora_scale,
                                             ((i > 0) || params.lora_base.empty())
                                                ? NULL
                                                : params.lora_base.c_str(),
                                             params.n_threads);
        # 如果应用失败，打印错误信息，释放内存，返回空指针
        if (err != 0) {
            fprintf(stderr, "%s: error: failed to apply lora adapter\n", __func__);
            llama_free(lctx);
            llama_free_model(model);
            return std::make_tuple(nullptr, nullptr);
        }
    }

    # 如果设置了忽略 EOS 标志
    if (params.ignore_eos) {
        # 将 logit_bias 中 EOS 标记对应的值设为负无穷
        params.sparams.logit_bias[llama_token_eos(model)] = -INFINITY;
    }

    # 执行模型的预热操作
    {
        LOG("warming up the model with an empty run\n");
        # 创建包含 BOS 和 EOS 标记的临时向量
        std::vector<llama_token> tmp = { llama_token_bos(model), llama_token_eos(model), };
        # 对临时向量进行解码
        llama_decode(lctx, llama_batch_get_one(tmp.data(), std::min(tmp.size(), (size_t) params.n_batch), 0, 0));
        # 清空键值缓存
        llama_kv_cache_clear(lctx);
        # 重置计时器
        llama_reset_timings(lctx);
    }

    # 返回模型和上下文的元组
    return std::make_tuple(model, lctx);
// Vocab utils
//

// 使用给定的上下文和文本进行分词，返回分词结果
std::vector<llama_token> llama_tokenize(
  const struct llama_context * ctx,
           const std::string & text,
                        bool   add_bos,
                        bool   special) {
    return llama_tokenize(llama_get_model(ctx), text, add_bos, special);
}

// 使用给定的模型和文本进行分词，返回分词结果
std::vector<llama_token> llama_tokenize(
    const struct llama_model * model,
           const std::string & text,
                        bool   add_bos,
                        bool   special) {
    // 计算最大可能的分词数量
    int n_tokens = text.length() + add_bos;
    // 创建存储分词结果的向量
    std::vector<llama_token> result(n_tokens);
    // 调用分词函数，获取实际的分词数量
    n_tokens = llama_tokenize(model, text.data(), text.length(), result.data(), result.size(), add_bos, special);
    // 处理分词数量为负数的情况
    if (n_tokens < 0) {
        // 调整存储分词结果的向量大小
        result.resize(-n_tokens);
        // 再次调用分词函数，确保分词数量为负数
        int check = llama_tokenize(model, text.data(), text.length(), result.data(), result.size(), add_bos, special);
        // 断言分词数量为负数
        GGML_ASSERT(check == -n_tokens);
    } else {
        // 调整存储分词结果的向量大小
        result.resize(n_tokens);
    }
    // 返回分词结果
    return result;
}

// 使用给定的上下文和分词结果，将分词转换为对应的文本片段
std::string llama_token_to_piece(const struct llama_context * ctx, llama_token token) {
    // 创建存储文本片段的字符向量
    std::vector<char> result(8, 0);
    // 调用将分词转换为文本片段的函数，获取文本片段的长度
    const int n_tokens = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
    // 处理文本片段长度为负数的情况
    if (n_tokens < 0) {
        // 调整存储文本片段的字符向量大小
        result.resize(-n_tokens);
        // 再次调用将分词转换为文本片段的函数，确保文本片段长度为负数
        int check = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
        // 断言文本片段长度为负数
        GGML_ASSERT(check == -n_tokens);
    } else {
        // 调整存储文本片段的字符向量大小
        result.resize(n_tokens);
    }
    // 返回文本片段
    return std::string(result.data(), result.size());
}

// 使用给定的上下文和分词结果，将分词结果反向转换为原始文本
std::string llama_detokenize_spm(llama_context * ctx, const std::vector<llama_token> & tokens) {
    // 获取开始符号的分词
    const llama_token bos_id = llama_token_bos(llama_get_model(ctx));

    // 初始化存储文本片段和结果的字符串
    std::string piece;
    std::string result;
    # 遍历 tokens 容器中的元素
    for (size_t i = 0; i < tokens.size(); ++i) {
        # 将 token 转换为对应的 piece
        piece = llama_token_to_piece(ctx, tokens[i]);

        # 如果是第一个非 BOS token，并且 piece 的第一个字符是空格，则去掉空格
        if (((tokens[0] == bos_id && i == 1) || (tokens[0] != bos_id && i == 0)) && piece[0] == ' ') {
            piece = piece.substr(1);
        }

        # 将 piece 添加到结果中
        result += piece;
    }

    # 返回最终结果
    return result;
}

// 将 BPE 标记解码为字符串
std::string llama_detokenize_bpe(llama_context * ctx, const std::vector<llama_token> & tokens) {
    std::string piece; // 用于存储单个 BPE 标记解码后的字符串
    std::string result; // 存储最终解码结果的字符串

    for (size_t i = 0; i < tokens.size(); ++i) {
        piece = llama_token_to_piece(ctx, tokens[i]); // 将 BPE 标记转换为字符串
        result += piece; // 将转换后的字符串拼接到最终结果中
    }

    // 注意：原始的标记化器在收集完片段后解码字节。
    return result; // 返回解码后的字符串
}

//
// YAML 工具函数
//

// 如果成功创建目录则返回 true，否则返回 false
bool create_directory_with_parents(const std::string & path) {
#ifdef _WIN32
    std::wstring_convert<std::codecvt_utf8<wchar_t>> converter;
    std::wstring wpath = converter.from_bytes(path); // 将 UTF-8 编码的路径转换为宽字符路径

    // 如果路径已经存在，则检查它是否是一个目录
    const DWORD attributes = GetFileAttributesW(wpath.c_str());
    if ((attributes != INVALID_FILE_ATTRIBUTES) && (attributes & FILE_ATTRIBUTE_DIRECTORY)) {
        return true;
    }

    size_t pos_slash = 0;

    // 从前到后处理路径，逐步创建目录
    while ((pos_slash = path.find('\\', pos_slash)) != std::string::npos) {
        const std::wstring subpath = wpath.substr(0, pos_slash); // 获取子路径
        const wchar_t * test = subpath.c_str(); // 获取子路径的 C 字符串表示

        const bool success = CreateDirectoryW(test, NULL); // 创建目录
        if (!success) {
            const DWORD error = GetLastError(); // 获取错误码

            // 如果路径已经存在，则确保它是一个目录
            if (error == ERROR_ALREADY_EXISTS) {
                const DWORD attributes = GetFileAttributesW(subpath.c_str());
                if (attributes == INVALID_FILE_ATTRIBUTES || !(attributes & FILE_ATTRIBUTE_DIRECTORY)) {
                    return false;
                }
            } else {
                return false;
            }
        }

        pos_slash += 1;
    }

    return true;
#else
    // 如果路径已经存在，则检查它是否是一个目录
    struct stat info;
    if (stat(path.c_str(), &info) == 0) {
        return S_ISDIR(info.st_mode);
    }
    // 设置初始斜杠位置为1，用于跳过路径开头的斜杠，以便创建目录
    size_t pos_slash = 1; 

    // 从前向后处理路径，逐步创建目录
    while ((pos_slash = path.find('/', pos_slash)) != std::string::npos) {
        // 获取当前斜杠位置之前的子路径
        const std::string subpath = path.substr(0, pos_slash);
        struct stat info;

        // 如果路径已经存在，则确保它是一个目录
        if (stat(subpath.c_str(), &info) == 0) {
            if (!S_ISDIR(info.st_mode)) {
                return false;
            }
        } else {
            // 创建父目录
            const int ret = mkdir(subpath.c_str(), 0755);
            if (ret != 0) {
                return false;
            }
        }

        // 移动到下一个斜杠位置
        pos_slash += 1;
    }

    // 如果成功创建所有目录，则返回true
    return true;
#endif // _WIN32
}

// 将浮点数向量以 YAML 格式输出到文件流
void dump_vector_float_yaml(FILE * stream, const char * prop_name, const std::vector<float> & data) {
    // 如果向量为空，输出属性名后换行
    if (data.empty()) {
        fprintf(stream, "%s:\n", prop_name);
        return;
    }

    // 输出属性名和向量起始标记
    fprintf(stream, "%s: [", prop_name);
    // 遍历向量，输出每个元素和逗号，最后输出最后一个元素和结束标记
    for (size_t i = 0; i < data.size() - 1; ++i) {
        fprintf(stream, "%e, ", data[i]);
    }
    fprintf(stream, "%e]\n", data.back());
}

// 将整数向量以 YAML 格式输出到文件流
void dump_vector_int_yaml(FILE * stream, const char * prop_name, const std::vector<int> & data) {
    // 如果向量为空，输出属性名后换行
    if (data.empty()) {
        fprintf(stream, "%s:\n", prop_name);
        return;
    }

    // 输出属性名和向量起始标记
    fprintf(stream, "%s: [", prop_name);
    // 遍历向量，输出每个元素和逗号，最后输出最后一个元素和结束标记
    for (size_t i = 0; i < data.size() - 1; ++i) {
        fprintf(stream, "%d, ", data[i]);
    }
    fprintf(stream, "%d]\n", data.back());
}

// 将多行字符串以 YAML 格式输出到文件流
void dump_string_yaml_multiline(FILE * stream, const char * prop_name, const char * data) {
    // 将 C 风格字符串转换为 C++ 字符串
    std::string data_str(data == NULL ? "" : data);

    // 如果字符串为空，输出属性名后换行
    if (data_str.empty()) {
        fprintf(stream, "%s:\n", prop_name);
        return;
    }

    size_t pos_start = 0;
    size_t pos_found = 0;

    // 如果字符串首尾有空白字符，进行转义处理并输出
    if (!data_str.empty() && (std::isspace(data_str[0]) || std::isspace(data_str.back()))) {
        data_str = std::regex_replace(data_str, std::regex("\n"), "\\n");
        data_str = std::regex_replace(data_str, std::regex("\""), "\\\"");
        data_str = "\"" + data_str + "\"";
        fprintf(stream, "%s: %s\n", prop_name, data_str.c_str());
        return;
    }

    // 如果字符串中不包含换行符，直接输出
    if (data_str.find('\n') == std::string::npos) {
        fprintf(stream, "%s: %s\n", prop_name, data_str.c_str());
        return;
    }

    // 输出属性名和多行字符串起始标记
    fprintf(stream, "%s: |\n", prop_name);
    // 遍历字符串，输出每行缩进后的内容
    while ((pos_found = data_str.find('\n', pos_start)) != std::string::npos) {
        fprintf(stream, "  %s\n", data_str.substr(pos_start, pos_found-pos_start).c_str());
        pos_start = pos_found + 1;
    }
}

// 获取可排序的时间戳字符串
std::string get_sortable_timestamp() {
    using clock = std::chrono::system_clock;

    // 获取当前时间点
    const clock::time_point current_time = clock::now();
    // 将当前时间转换为 time_t 类型
    const time_t as_time_t = clock::to_time_t(current_time);
    // 创建一个存储时间戳的字符串数组，格式为年月日-时分秒
    char timestamp_no_ns[100];
    // 格式化时间，将格式化后的时间存储到 timestamp_no_ns 中
    std::strftime(timestamp_no_ns, 100, "%Y_%m_%d-%H_%M_%S", std::localtime(&as_time_t));

    // 计算当前时间的纳秒部分
    const int64_t ns = std::chrono::duration_cast<std::chrono::nanoseconds>(
        current_time.time_since_epoch() % 1000000000).count();
    // 创建一个存储纳秒部分的字符串数组，格式为 9 位数字
    char timestamp_ns[11];
    // 将纳秒部分格式化为字符串，存储到 timestamp_ns 中
    snprintf(timestamp_ns, 11, "%09" PRId64, ns);

    // 返回格式化后的时间戳，包括秒和纳秒部分
    return std::string(timestamp_no_ns) + "." + std::string(timestamp_ns);
}
// 将非结果信息转储为 YAML 格式
void dump_non_result_info_yaml(FILE * stream, const gpt_params & params, const llama_context * lctx,
                               const std::string & timestamp, const std::vector<int> & prompt_tokens, const char * model_desc) {
    // 获取采样参数
    const llama_sampling_params & sparams = params.sparams;

    // 输出 LLAMA 的构建提交信息
    fprintf(stream, "build_commit: %s\n",        LLAMA_COMMIT);
    // 输出 LLAMA 的构建编号
    fprintf(stream, "build_number: %d\n",        LLAMA_BUILD_NUMBER);
    // 输出 CPU 是否支持 ARM FMA 指令集
    fprintf(stream, "cpu_has_arm_fma: %s\n",     ggml_cpu_has_arm_fma()     ? "true" : "false");
    // 输出 CPU 是否支持 AVX 指令集
    fprintf(stream, "cpu_has_avx: %s\n",         ggml_cpu_has_avx()         ? "true" : "false");
    // 输出 CPU 是否支持 AVX2 指令集
    fprintf(stream, "cpu_has_avx2: %s\n",        ggml_cpu_has_avx2()        ? "true" : "false");
    // 输出 CPU 是否支持 AVX512 指令集
    fprintf(stream, "cpu_has_avx512: %s\n",      ggml_cpu_has_avx512()      ? "true" : "false");
    // 输出 CPU 是否支持 AVX512 VBMI 指令集
    fprintf(stream, "cpu_has_avx512_vbmi: %s\n", ggml_cpu_has_avx512_vbmi() ? "true" : "false");
    // 输出 CPU 是否支持 AVX512 VNNI 指令集
    fprintf(stream, "cpu_has_avx512_vnni: %s\n", ggml_cpu_has_avx512_vnni() ? "true" : "false");
    // 输出 CPU 是否支持 BLAS 库
    fprintf(stream, "cpu_has_blas: %s\n",        ggml_cpu_has_blas()        ? "true" : "false");
    // 输出 CPU 是否支持 cuBLAS 库
    fprintf(stream, "cpu_has_cublas: %s\n",      ggml_cpu_has_cublas()      ? "true" : "false");
    // 输出 CPU 是否支持 clBLAS 库
    fprintf(stream, "cpu_has_clblast: %s\n",     ggml_cpu_has_clblast()     ? "true" : "false");
    // 输出 CPU 是否支持 FMA 指令集
    fprintf(stream, "cpu_has_fma: %s\n",         ggml_cpu_has_fma()         ? "true" : "false");
    // 输出 CPU 是否支持 GPU BLAS 库
    fprintf(stream, "cpu_has_gpublas: %s\n",     ggml_cpu_has_gpublas()     ? "true" : "false");
    // 输出 CPU 是否支持 NEON 指令集
    fprintf(stream, "cpu_has_neon: %s\n",        ggml_cpu_has_neon()        ? "true" : "false");
    // 输出 CPU 是否支持 F16C 指令集
    fprintf(stream, "cpu_has_f16c: %s\n",        ggml_cpu_has_f16c()        ? "true" : "false");
    // 输出 CPU 是否支持 FP16 VA 指令集
    fprintf(stream, "cpu_has_fp16_va: %s\n",     ggml_cpu_has_fp16_va()     ? "true" : "false");
    // 输出 CPU 是否支持 WASM SIMD 指令集
    fprintf(stream, "cpu_has_wasm_simd: %s\n",   ggml_cpu_has_wasm_simd()   ? "true" : "false");
    // 输出 CPU 是否支持 BLAS 库
    fprintf(stream, "cpu_has_blas: %s\n",        ggml_cpu_has_blas()        ? "true" : "false");
}
    # 将 "cpu_has_sse3: true/false" 格式的字符串写入流中，表示当前 CPU 是否支持 SSE3 指令集
    fprintf(stream, "cpu_has_sse3: %s\n",        ggml_cpu_has_sse3()        ? "true" : "false");
    # 将 "cpu_has_vsx: true/false" 格式的字符串写入流中，表示当前 CPU 是否支持 VSX 指令集
    fprintf(stream, "cpu_has_vsx: %s\n",         ggml_cpu_has_vsx()         ? "true" : "false");
#ifdef NDEBUG
    // 如果处于非调试模式，则输出“debug: false”
    fprintf(stream, "debug: false\n");
#else
    // 如果处于调试模式，则输出“debug: true”
    fprintf(stream, "debug: true\n");
#endif // NDEBUG

    // 输出模型描述
    fprintf(stream, "model_desc: %s\n", model_desc);
    // 输出词汇表大小，对于某些模型是最终层的输出大小，为32001
    fprintf(stream, "n_vocab: %d  # output size of the final layer, 32001 for some models\n", llama_n_vocab(llama_get_model(lctx)));

#ifdef __OPTIMIZE__
    // 如果处于优化模式，则输出“optimize: true”
    fprintf(stream, "optimize: true\n");
#else
    // 如果未处于优化模式，则输出“optimize: false”
    fprintf(stream, "optimize: false\n");
#endif // __OPTIMIZE__

    // 输出时间戳
    fprintf(stream, "time: %s\n", timestamp.c_str());

    // 输出分隔符和用户输入部分的标题
    fprintf(stream, "\n");
    fprintf(stream, "###############\n");
    fprintf(stream, "# User Inputs #\n");
    fprintf(stream, "###############\n");
    fprintf(stream, "\n");

    // 输出用户输入的参数信息
    fprintf(stream, "alias: %s # default: unknown\n", params.model_alias.c_str());
    fprintf(stream, "batch_size: %d # default: 512\n", params.n_batch);
    dump_string_yaml_multiline(stream, "cfg_negative_prompt", sparams.cfg_negative_prompt.c_str());
    fprintf(stream, "cfg_scale: %f # default: 1.0\n", sparams.cfg_scale);
    fprintf(stream, "chunks: %d # default: -1 (unlimited)\n", params.n_chunks);
    fprintf(stream, "color: %s # default: false\n", params.use_color ? "true" : "false");
    fprintf(stream, "ctx_size: %d # default: 512\n", params.n_ctx);
    fprintf(stream, "escape: %s # default: false\n", params.escape ? "true" : "false");
    fprintf(stream, "file: # never logged, see prompt instead. Can still be specified for input.\n");
    fprintf(stream, "frequency_penalty: %f # default: 0.0 \n", sparams.penalty_freq);
    dump_string_yaml_multiline(stream, "grammar", sparams.grammar.c_str());
    fprintf(stream, "grammar-file: # never logged, see grammar instead. Can still be specified for input.\n");
    fprintf(stream, "hellaswag: %s # default: false\n", params.hellaswag ? "true" : "false");
    fprintf(stream, "hellaswag_tasks: %zu # default: 400\n", params.hellaswag_tasks);

    // 获取特定标记的偏置值
    const auto logit_bias_eos = sparams.logit_bias.find(llama_token_eos(llama_get_model(lctx)));
    // 检查是否忽略 EOS（End of Sentence），如果 logit_bias_eos 不为空且值为负无穷，则为 true，否则为 false
    const bool ignore_eos = logit_bias_eos != sparams.logit_bias.end() && logit_bias_eos->second == -INFINITY;
    // 输出 ignore_eos 的取值，以及默认值为 false
    fprintf(stream, "ignore_eos: %s # default: false\n", ignore_eos ? "true" : "false");

    // 输出输入前缀的多行 YAML 格式字符串
    dump_string_yaml_multiline(stream, "in_prefix", params.input_prefix.c_str());
    // 输出输入前缀的 BOS（Beginning of Sentence）标志，以及默认值为 false
    fprintf(stream, "in_prefix_bos: %s # default: false\n", params.input_prefix_bos ? "true" : "false");
    // 输出输入后缀的多行 YAML 格式字符串
    dump_string_yaml_multiline(stream, "in_suffix", params.input_prefix.c_str());
    // 输出指示标志，以及默认值为 false
    fprintf(stream, "instruct: %s # default: false\n", params.instruct ? "true" : "false");
    // 输出交互标志，以及默认值为 false
    fprintf(stream, "interactive: %s # default: false\n", params.interactive ? "true" : "false");
    // 输出首次交互标志，以及默认值为 false
    fprintf(stream, "interactive_first: %s # default: false\n", params.interactive_first ? "true" : "false");
    // 输出保留数量，以及默认值为 0
    fprintf(stream, "keep: %d # default: 0\n", params.n_keep);
    // 输出日志目录，以及默认值为未设置（无日志记录）
    fprintf(stream, "logdir: %s # default: unset (no logging)\n", params.logdir.c_str());

    // 输出 logit_bias 的键值对
    fprintf(stream, "logit_bias:\n");
    for (std::pair<llama_token, float> lb : sparams.logit_bias) {
        // 如果忽略 EOS 并且当前键值对的键等于 logit_bias_eos 的键，则跳过
        if (ignore_eos && lb.first == logit_bias_eos->first) {
            continue;
        }
        fprintf(stream, "  %d: %f", lb.first, lb.second);
    }

    // 输出 lora_adapter 中比例不为 1.0 的元素
    fprintf(stream, "lora:\n");
    for (std::tuple<std::string, float> la : params.lora_adapter) {
        if (std::get<1>(la) != 1.0f) {
            continue;
        }
        fprintf(stream, "  - %s\n", std::get<0>(la).c_str());
    }

    // 输出 lora_adapter 中比例为 1.0 的元素
    fprintf(stream, "lora_scaled:\n");
    for (std::tuple<std::string, float> la : params.lora_adapter) {
        if (std::get<1>(la) == 1.0f) {
            continue;
        }
        fprintf(stream, "  - %s: %f\n", std::get<0>(la).c_str(), std::get<1>(la));
    }
    // 输出 lora_base，即 lora 的基础值
    fprintf(stream, "lora_base: %s\n", params.lora_base.c_str());
    // 输出重置 GPU 索引标志，以及默认值为 false
    fprintf(stream, "reset_gpu_index: %s\n", params.reset_gpu_index ? "true" : "false");
    // 输出禁用 GPU 索引标志，以及默认值为 false
    fprintf(stream, "disable_gpu_index: %s\n", params.disale_gpu_index? "true": "false");
    # 输出 main_gpu 参数的值到流中，如果不存在则使用默认值 0
    fprintf(stream, "main_gpu: %d # default: 0\n", params.main_gpu);
    # 输出 memory_f32 参数的值到流中，如果不存在则使用默认值 false
    fprintf(stream, "memory_f32: %s # default: false\n", !params.memory_f16 ? "true" : "false");
    # 输出 mirostat 参数的值到流中，如果不存在则使用默认值 0 (disabled)
    fprintf(stream, "mirostat: %d # default: 0 (disabled)\n", sparams.mirostat);
    # 输出 mirostat_ent 参数的值到流中，如果不存在则使用默认值 5.0
    fprintf(stream, "mirostat_ent: %f # default: 5.0\n", sparams.mirostat_tau);
    # 输出 mirostat_lr 参数的值到流中，如果不存在则使用默认值 0.1
    fprintf(stream, "mirostat_lr: %f # default: 0.1\n", sparams.mirostat_eta);
    # 输出 mlock 参数的值到流中，如果不存在则使用默认值 false
    fprintf(stream, "mlock: %s # default: false\n", params.use_mlock ? "true" : "false");
    # 输出 model 参数的值到流中，如果不存在则使用默认值 models/7B/ggml-model.bin
    fprintf(stream, "model: %s # default: models/7B/ggml-model.bin\n", params.model.c_str());
    # 输出 model_draft 参数的值到流中，如果不存在则使用默认值 空字符串
    fprintf(stream, "model_draft: %s # default:\n", params.model_draft.c_str());
    # 输出 multiline_input 参数的值到流中，如果不存在则使用默认值 false
    fprintf(stream, "multiline_input: %s # default: false\n", params.multiline_input ? "true" : "false");
    # 输出 n_gpu_layers 参数的值到流中，如果不存在则使用默认值 -1
    fprintf(stream, "n_gpu_layers: %d # default: -1\n", params.n_gpu_layers);
    # 输出 n_predict 参数的值到流中，如果不存在则使用默认值 -1 (unlimited)
    fprintf(stream, "n_predict: %d # default: -1 (unlimited)\n", params.n_predict);
    # 输出 n_probs 参数的值到流中，如果不存在则使用默认值 0，只在服务器二进制中使用
    fprintf(stream, "n_probs: %d # only used by server binary, default: 0\n", sparams.n_probs);
    # 输出 no_mmap 参数的值到流中，如果不存在则使用默认值 false
    fprintf(stream, "no_mmap: %s # default: false\n", !params.use_mmap ? "true" : "false");
    # 输出 no_mul_mat_q 参数的值到流中，如果不存在则使用默认值 false
    fprintf(stream, "no_mul_mat_q: %s # default: false\n", !params.mul_mat_q ? "true" : "false");
    # 输出 no_penalize_nl 参数的值到流中，如果不存在则使用默认值 false
    fprintf(stream, "no_penalize_nl: %s # default: false\n", !sparams.penalize_nl ? "true" : "false");
    # 输出 numa 参数的值到流中，如果不存在则使用默认值 false
    fprintf(stream, "numa: %s # default: false\n", params.numa ? "true" : "false");
    # 输出 ppl_output_type 参数的值到流中，如果不存在则使用默认值 0
    fprintf(stream, "ppl_output_type: %d # default: 0\n", params.ppl_output_type);
    # 输出 ppl_stride 参数的值到流中，如果不存在则使用默认值 0
    fprintf(stream, "ppl_stride: %d # default: 0\n", params.ppl_stride);
    # 输出 presence_penalty 参数的值到流中，如果不存在则使用默认值 0.0
    fprintf(stream, "presence_penalty: %f # default: 0.0\n", sparams.penalty_present);
    # 输出 prompt 参数的值到流中，以多行 YAML 格式输出
    dump_string_yaml_multiline(stream, "prompt", params.prompt.c_str());
    # 输出 prompt_cache 参数的值到流中
    fprintf(stream, "prompt_cache: %s\n", params.path_prompt_cache.c_str());
    # 输出 prompt_cache_all 参数的值到流中，如果不存在则使用默认值 false
    fprintf(stream, "prompt_cache_all: %s # default: false\n", params.prompt_cache_all ? "true" : "false");
    // 将 prompt_cache_ro 参数的值写入流中，如果为 true 则写入 true，否则写入 false
    fprintf(stream, "prompt_cache_ro: %s # default: false\n", params.prompt_cache_ro ? "true" : "false");
    // 将 prompt_tokens 向量以 YAML 格式写入流中
    dump_vector_int_yaml(stream, "prompt_tokens", prompt_tokens);
    // 将 random_prompt 参数的值写入流中，如果为 true 则写入 true，否则写入 false
    fprintf(stream, "random_prompt: %s # default: false\n", params.random_prompt ? "true" : "false");
    // 将 repeat_penalty 参数的值以浮点数格式写入流中
    fprintf(stream, "repeat_penalty: %f # default: 1.1\n", sparams.penalty_repeat);

    // 将 reverse_prompt 参数的值写入流中
    fprintf(stream, "reverse_prompt:\n");
    // 遍历 antiprompt 向量中的字符串，将换行符替换为 "\n"，然后写入流中
    for (std::string ap : params.antiprompt) {
        size_t pos = 0;
        while ((pos = ap.find('\n', pos)) != std::string::npos) {
            ap.replace(pos, 1, "\\n");
            pos += 1;
        }
        fprintf(stream, "  - %s\n", ap.c_str());
    }

    // 将 rope_freq_base 参数的值以浮点数格式写入流中
    fprintf(stream, "rope_freq_base: %f # default: 10000.0\n", params.rope_freq_base);
    // 将 rope_freq_scale 参数的值以浮点数格式写入流中
    fprintf(stream, "rope_freq_scale: %f # default: 1.0\n", params.rope_freq_scale);
    // 将 seed 参数的值以整数格式写入流中
    fprintf(stream, "seed: %d # default: -1 (random seed)\n", params.seed);
    // 将 simple_io 参数的值写入流中，如果为 true 则写入 true，否则写入 false
    fprintf(stream, "simple_io: %s # default: false\n", params.simple_io ? "true" : "false");
    // 将 cont_batching 参数的值写入流中，如果为 true 则写入 true，否则写入 false
    fprintf(stream, "cont_batching: %s # default: false\n", params.cont_batching ? "true" : "false");
    // 将 temp 参数的值以浮点数格式写入流中
    fprintf(stream, "temp: %f # default: 0.8\n", sparams.temp);

    // 将 tensor_split 向量以 YAML 格式写入流中
    const std::vector<float> tensor_split_vector(params.tensor_split, params.tensor_split + LLAMA_MAX_DEVICES);
    dump_vector_float_yaml(stream, "tensor_split", tensor_split_vector);

    // 将 tfs_z 参数的值以浮点数格式写入流中
    fprintf(stream, "tfs: %f # default: 1.0\n", sparams.tfs_z);
    // 将 threads 参数的值以整数格式写入流中
    fprintf(stream, "threads: %d # default: %d\n", params.n_threads, std::thread::hardware_concurrency());
    // 将 top_k 参数的值以整数格式写入流中
    fprintf(stream, "top_k: %d # default: 40\n", sparams.top_k);
    // 将 top_p 参数的值以浮点数格式写入流中
    fprintf(stream, "top_p: %f # default: 0.95\n", sparams.top_p);
    // 将 min_p 参数的值以浮点数格式写入流中
    fprintf(stream, "min_p: %f # default: 0.0\n", sparams.min_p);
    // 将 typical_p 参数的值以浮点数格式写入流中
    fprintf(stream, "typical_p: %f # default: 1.0\n", sparams.typical_p);
    // 将 verbose_prompt 参数的值写入流中，如果为 true 则写入 true，否则写入 false
    fprintf(stream, "verbose_prompt: %s # default: false\n", params.verbose_prompt ? "true" : "false");
    # 将 vram_budget_gb 的值格式化后写入流中，用于输出到文件或屏幕
    fprintf(stream, "vram_budget: %f # default: -1.0 (all available VRAM)\n", params.vram_budget_gb);
# 闭合前面的函数定义
```