# `PowerInfer\common\common.cpp`

```
#include "common.h" // 包含 common.h 头文件
#include "llama.h" // 包含 llama.h 头文件

#include <algorithm> // 包含算法库
#include <cassert> // 包含断言库
#include <cmath> // 包含数学库
#include <cstring> // 包含字符串库
#include <ctime> // 包含时间库
#include <fstream> // 包含文件流库
#include <iterator> // 包含迭代器库
#include <iostream> // 包含输入输出流库
#include <regex> // 包含正则表达式库
#include <sstream> // 包含字符串流库
#include <string> // 包含字符串库
#include <unordered_set> // 包含无序集合库
#include <vector> // 包含向量库
#include <cinttypes> // 包含整数类型库

#if defined(__APPLE__) && defined(__MACH__) // 如果是苹果系统和 Mach 内核
#include <sys/types.h> // 包含系统类型库
#include <sys/sysctl.h>
#endif
// 包含系统调用相关的头文件

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
// 根据操作系统的不同，包含不同的头文件

#if defined(_MSC_VER)
// 如果是使用 Microsoft Visual C++ 编译器
#pragma warning(disable: 4244 4267) // 禁止特定警告
#endif

// 获取物理核心数
int32_t get_num_physical_cores() {
#ifdef __linux__
    // 枚举线程的同级核心，条目数即为核心数
    std::unordered_set<std::string> siblings;
    for (uint32_t cpu=0; cpu < UINT32_MAX; ++cpu) {
        // 打开线程同级核心文件
        std::ifstream thread_siblings("/sys/devices/system/cpu"
            + std::to_string(cpu) + "/topology/thread_siblings");
        if (!thread_siblings.is_open()) {
            break; // 没有更多的 CPU
        }
        std::string line;
        if (std::getline(thread_siblings, line)) {
            siblings.insert(line); // 将线程同级核心添加到集合中
        }
    }
    if (!siblings.empty()) {
        return static_cast<int32_t>(siblings.size()); // 返回核心数
    }
#elif defined(__APPLE__) && defined(__MACH__)
    // 定义变量存储物理核心数
    int32_t num_physical_cores;
    // 获取变量长度
    size_t len = sizeof(num_physical_cores);
    // 通过 sysctlbyname 函数获取物理核心数
    int result = sysctlbyname("hw.perflevel0.physicalcpu", &num_physical_cores, &len, NULL, 0);
    // 如果成功获取到物理核心数，返回该值
    if (result == 0) {
        return num_physical_cores;
    }
    // 通过另一种方式获取物理核心数
    result = sysctlbyname("hw.physicalcpu", &num_physical_cores, &len, NULL, 0);
    // 如果成功获取到物理核心数，返回该值
    if (result == 0) {
        return num_physical_cores;
    }
#elif defined(_WIN32)
    // 在 Windows 平台下实现获取物理核心数的功能
    //TODO: Implement
#endif
    // 获取系统支持的线程数
    unsigned int n_threads = std::thread::hardware_concurrency();
    // 返回线程数，如果线程数大于 0，则返回线程数，否则返回 4
    return n_threads > 0 ? (n_threads <= 4 ? n_threads : n_threads / 2) : 4;
}

// 处理字符串中的转义字符
void process_escapes(std::string& input) {
    // 获取输入字符串的长度
    std::size_t input_len = input.length();
    // 初始化输出索引
    std::size_t output_idx = 0;

    // 遍历输入字符串
    for (std::size_t input_idx = 0; input_idx < input_len; ++input_idx) {
        // 判断是否为转义字符
        if (input[input_idx] == '\\' && input_idx + 1 < input_len) {
            // 处理转义字符
            switch (input[++input_idx]) {
                case 'n':  input[output_idx++] = '\n'; break;  // 换行
                case 'r':  input[output_idx++] = '\r'; break;  // 回车
                case 't':  input[output_idx++] = '\t'; break;  // 制表符
                case '\'': input[output_idx++] = '\''; break;  // 单引号
                case '\"': input[output_idx++] = '\"'; break;  // 双引号
                case '\\': input[output_idx++] = '\\'; break;  // 反斜杠
                case 'x':
                    // 处理十六进制转义字符，如 \x12
                    if (input_idx + 2 < input_len) {
                        // 获取十六进制字符
                        const char x[3] = { input[input_idx + 1], input[input_idx + 2], 0 };
                        char *err_p = nullptr;
                        // 将十六进制字符转换为长整型
                        const long val = std::strtol(x, &err_p, 16);
                        // 判断转换是否成功
                        if (err_p == x + 2) {
                            input_idx += 2;
                            // 将val转换为字符并存入input数组中，然后增加output_idx
                            input[output_idx++] = char(val);
                            // 跳出switch语句
                            break;
                        }
                    }
                    // 继续执行下面的语句
                    // 如果没有匹配的case，则执行default语句
                default:   // 将'\\'存入input数组中，然后增加output_idx
                           input[output_idx++] = '\\';
                           // 将input[input_idx]存入input数组中，然后增加output_idx
                           input[output_idx++] = input[input_idx]; 
                           // 跳出switch语句
                           break;
            }
        } else {
            // 将input[input_idx]存入input数组中，然后增加output_idx
            input[output_idx++] = input[input_idx];
        }
    }

    // 调整input数组的大小为output_idx
    input.resize(output_idx);
}

// 解析命令行参数并存入params中
bool gpt_params_parse(int argc, char ** argv, gpt_params & params) {
    bool result = true;
    try {
        // 如果gpt_params_parse_ex返回false
        if (!gpt_params_parse_ex(argc, argv, params)) {
    // 打印程序用法信息
    gpt_print_usage(argc, argv, gpt_params());
    // 退出程序
    exit(0);
}

// 捕获无效参数异常
catch (const std::invalid_argument & ex) {
    // 打印异常信息
    fprintf(stderr, "%s\n", ex.what());
    // 打印程序用法信息
    gpt_print_usage(argc, argv, gpt_params());
    // 退出程序
    exit(1);
}

// 返回解析结果
return result;
}

// 解析参数
bool gpt_params_parse_ex(int argc, char ** argv, gpt_params & params) {
    // 初始化无效参数标志
    bool invalid_param = false;
    // 定义参数字符串
    std::string arg;
    // 定义参数前缀
    const std::string arg_prefix = "--";
    // 获取采样参数
    llama_sampling_params & sparams = params.sparams;

    // 遍历参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数
        arg = argv[i];
# 如果参数以arg_prefix开头，则将参数中的下划线替换为破折号
if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
    std::replace(arg.begin(), arg.end(), '_', '-');
}

# 如果参数是"-s"或"--seed"，则将下一个参数转换为unsigned long类型并赋值给params.seed
if (arg == "-s" || arg == "--seed") {
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    params.seed = std::stoul(argv[i]);
} 
# 如果参数是"-t"或"--threads"，则将下一个参数转换为int类型并赋值给params.n_threads
# 如果n_threads小于等于0，则将其赋值为当前系统支持的线程数
else if (arg == "-t" || arg == "--threads") {
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    params.n_threads = std::stoi(argv[i]);
    if (params.n_threads <= 0) {
        params.n_threads = std::thread::hardware_concurrency();
    }
} 
# 如果参数是"-tb"或"--threads-batch"，则...
            // 如果参数数量超出范围，设置参数为无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将参数转换为整数并赋值给params.n_threads_batch，如果参数小于等于0，则使用硬件并发数
            params.n_threads_batch = std::stoi(argv[i]);
            if (params.n_threads_batch <= 0) {
                params.n_threads_batch = std::thread::hardware_concurrency();
            }
        } else if (arg == "-p" || arg == "--prompt") {
            // 如果参数数量超出范围，设置参数为无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将参数赋值给params.prompt
            params.prompt = argv[i];
        } else if (arg == "-e" || arg == "--escape") {
            // 设置params.escape为true
            params.escape = true;
        } else if (arg == "--prompt-cache") {
            // 如果参数数量超出范围，设置参数为无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
        } 
        // 如果参数是 --path-prompt-cache，则将下一个参数作为路径缓存
        params.path_prompt_cache = argv[i];
    } else if (arg == "--prompt-cache-all") {
        // 如果参数是 --prompt-cache-all，则设置 prompt_cache_all 为 true
        params.prompt_cache_all = true;
    } else if (arg == "--prompt-cache-ro") {
        // 如果参数是 --prompt-cache-ro，则设置 prompt_cache_ro 为 true
        params.prompt_cache_ro = true;
    } else if (arg == "-f" || arg == "--file") {
        // 如果参数是 -f 或 --file，则检查下一个参数是否存在
        if (++i >= argc) {
            // 如果下一个参数不存在，则设置 invalid_param 为 true 并跳出循环
            invalid_param = true;
            break;
        }
        // 打开文件流并将文件名作为参数
        std::ifstream file(argv[i]);
        // 如果文件流打开失败，则输出错误信息并设置 invalid_param 为 true 并跳出循环
        if (!file) {
            fprintf(stderr, "error: failed to open file '%s'\n", argv[i]);
            invalid_param = true;
            break;
        }
        // 将外部文件名存储在 params 中
        params.prompt_file = argv[i];
        // 将文件内容读取到 params.prompt 中
        std::copy(std::istreambuf_iterator<char>(file), std::istreambuf_iterator<char>(), back_inserter(params.prompt));
            // 如果提示不为空且最后一个字符是换行符，则移除换行符
            if (!params.prompt.empty() && params.prompt.back() == '\n') {
                params.prompt.pop_back();
            }
        } else if (arg == "-n" || arg == "--n-predict") {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数并赋值给 n_predict
            params.n_predict = std::stoi(argv[i]);
        } else if (arg == "--top-k") {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数并赋值给 top_k
            sparams.top_k = std::stoi(argv[i]);
        } else if (arg == "-c" || arg == "--ctx-size") {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
        // 如果参数是 "--ctx"，则将下一个参数转换为整数并赋值给 params.n_ctx
        if (arg == "--ctx") {
            params.n_ctx = std::stoi(argv[i]);
        } 
        // 如果参数是 "--rope-freq-base"，则将下一个参数转换为浮点数并赋值给 params.rope_freq_base
        else if (arg == "--rope-freq-base") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.rope_freq_base = std::stof(argv[i]);
        } 
        // 如果参数是 "--rope-freq-scale"，则将下一个参数转换为浮点数并赋值给 params.rope_freq_scale
        else if (arg == "--rope-freq-scale") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.rope_freq_scale = std::stof(argv[i]);
        } 
        // 如果参数是 "--rope-scaling"，则将下一个参数转换为字符串并根据不同值赋值给 params.rope_scaling_type
        else if (arg == "--rope-scaling") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            std::string value(argv[i]);
            if (value == "none")   { params.rope_scaling_type = LLAMA_ROPE_SCALING_NONE; }
        }
            else if (value == "linear") { params.rope_scaling_type = LLAMA_ROPE_SCALING_LINEAR; }
            else if (value == "yarn")   { params.rope_scaling_type = LLAMA_ROPE_SCALING_YARN; }
            else { invalid_param = true; break; }
        } else if (arg == "--rope-scale") {
            // 如果参数值为 "--rope-scale"，则将下一个参数转换为浮点数并赋值给 params.rope_freq_scale
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.rope_freq_scale = 1.0f/std::stof(argv[i]);
        } else if (arg == "--yarn-orig-ctx") {
            // 如果参数值为 "--yarn-orig-ctx"，则将下一个参数转换为整数并赋值给 params.yarn_orig_ctx
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.yarn_orig_ctx = std::stoi(argv[i]);
        } else if (arg == "--yarn-ext-factor") {
            // 如果参数值为 "--yarn-ext-factor"，则将下一个参数转换为浮点数并执行相应操作
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
        # 如果参数是 "--yarn-ext-factor"，则将下一个参数转换为浮点数并赋值给 params.yarn_ext_factor
        params.yarn_ext_factor = std::stof(argv[i]);
    } else if (arg == "--yarn-attn-factor") {
        # 如果参数是 "--yarn-attn-factor"，则检查下一个参数是否存在，如果不存在则将 invalid_param 设为 true，否则将下一个参数转换为浮点数并赋值给 params.yarn_attn_factor
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params.yarn_attn_factor = std::stof(argv[i]);
    } else if (arg == "--yarn-beta-fast") {
        # 如果参数是 "--yarn-beta-fast"，则检查下一个参数是否存在，如果不存在则将 invalid_param 设为 true，否则将下一个参数转换为浮点数并赋值给 params.yarn_beta_fast
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params.yarn_beta_fast = std::stof(argv[i]);
    } else if (arg == "--yarn-beta-slow") {
        # 如果参数是 "--yarn-beta-slow"，则检查下一个参数是否存在，如果不存在则将 invalid_param 设为 true，否则将下一个参数转换为浮点数并赋值给 params.yarn_beta_slow
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params.yarn_beta_slow = std::stof(argv[i]);
    } else if (arg == "--memory-f32") {
        // 如果参数为 "--memory-f16"，则将 params.memory_f16 设置为 false
        params.memory_f16 = false;
    } else if (arg == "--top-p") {
        // 如果参数为 "--top-p"，则将下一个参数转换为浮点数并赋值给 sparams.top_p
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        sparams.top_p = std::stof(argv[i]);
    } else if (arg == "--min-p") {
        // 如果参数为 "--min-p"，则将下一个参数转换为浮点数并赋值给 sparams.min_p
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        sparams.min_p = std::stof(argv[i]);
    } else if (arg == "--temp") {
        // 如果参数为 "--temp"，则将下一个参数转换为浮点数并赋值给 sparams.temp
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        sparams.temp = std::stof(argv[i]);
        // 确保 sparams.temp 的值不小于 0.0
        sparams.temp = std::max(sparams.temp, 0.0f);
        } else if (arg == "--tfs") {
            // 如果参数为"--tfs"，则获取下一个参数作为tfs_z的值
            if (++i >= argc) {
                // 如果没有下一个参数，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为浮点数，并赋值给tfs_z
            sparams.tfs_z = std::stof(argv[i]);
        } else if (arg == "--typical") {
            // 如果参数为"--typical"，则获取下一个参数作为typical_p的值
            if (++i >= argc) {
                // 如果没有下一个参数，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为浮点数，并赋值给typical_p
            sparams.typical_p = std::stof(argv[i]);
        } else if (arg == "--repeat-last-n") {
            // 如果参数为"--repeat-last-n"，则获取下一个参数作为penalty_last_n的值
            if (++i >= argc) {
                // 如果没有下一个参数，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数，并赋值给penalty_last_n
            sparams.penalty_last_n = std::stoi(argv[i]);
            // 将penalty_last_n与n_prev的较大值赋给n_prev
            sparams.n_prev = std::max(sparams.n_prev, sparams.penalty_last_n);
        } else if (arg == "--repeat-penalty") {
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将参数转换为浮点数，并赋值给对应的参数
            sparams.penalty_repeat = std::stof(argv[i]);
        } else if (arg == "--frequency-penalty") {
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将参数转换为浮点数，并赋值给对应的参数
            sparams.penalty_freq = std::stof(argv[i]);
        } else if (arg == "--presence-penalty") {
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将参数转换为浮点数，并赋值给对应的参数
            sparams.penalty_present = std::stof(argv[i]);
        } else if (arg == "--mirostat") {
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
            # 如果参数为"--mirostat-lr"，则将下一个参数转换为浮点数并赋值给sparams.mirostat_eta
            if (++i >= argc):
                # 如果参数不足，设置invalid_param为True并跳出循环
                invalid_param = true
                break
            sparams.mirostat_eta = std::stof(argv[i])
        # 如果参数为"--mirostat-ent"，则将下一个参数转换为浮点数并赋值给sparams.mirostat_tau
        else if (arg == "--mirostat-ent"):
            if (++i >= argc):
                # 如果参数不足，设置invalid_param为True并跳出循环
                invalid_param = true
                break
            sparams.mirostat_tau = std::stof(argv[i])
        # 如果参数为"--cfg-negative-prompt"，则将下一个参数赋值给某个变量（此处缺少具体的赋值操作）
        else if (arg == "--cfg-negative-prompt"):
            if (++i >= argc):
                # 如果参数不足，设置invalid_param为True并跳出循环
                invalid_param = true
                break
        // 如果参数是"--cfg-negative-prompt"，则将下一个参数作为负面提示设置
        sparams.cfg_negative_prompt = argv[i];
        // 如果参数是"--cfg-negative-prompt-file"
        } else if (arg == "--cfg-negative-prompt-file") {
            // 如果下一个参数不存在，则设置参数无效并退出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 打开文件流
            std::ifstream file(argv[i]);
            // 如果文件打开失败，则输出错误信息并设置参数无效
            if (!file) {
                fprintf(stderr, "error: failed to open file '%s'\n", argv[i]);
                invalid_param = true;
                break;
            }
            // 从文件流中读取内容并添加到负面提示设置中
            std::copy(std::istreambuf_iterator<char>(file), std::istreambuf_iterator<char>(), back_inserter(sparams.cfg_negative_prompt));
            // 如果负面提示设置不为空且最后一个字符是换行符，则移除换行符
            if (!sparams.cfg_negative_prompt.empty() && sparams.cfg_negative_prompt.back() == '\n') {
                sparams.cfg_negative_prompt.pop_back();
            }
        // 如果参数是"--cfg-scale"
        } else if (arg == "--cfg-scale") {
            // 如果下一个参数不存在，则设置参数无效并退出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
        } 
        # 如果参数是"-s"或者"--scale"，则将下一个参数转换为浮点数并赋值给配置参数cfg_scale
        else if (arg == "-s" || arg == "--scale") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            sparams.cfg_scale = std::stof(argv[i]);
        } 
        # 如果参数是"-b"或者"--batch-size"，则将下一个参数转换为整数并赋值给参数n_batch
        else if (arg == "-b" || arg == "--batch-size") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.n_batch = std::stoi(argv[i]);
        } 
        # 如果参数是"--keep"，则将下一个参数转换为整数并赋值给参数n_keep
        else if (arg == "--keep") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.n_keep = std::stoi(argv[i]);
        } 
        # 如果参数是"--draft"，则将下一个参数转换为整数并赋值给参数n_draft
        else if (arg == "--draft") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.n_draft = std::stoi(argv[i]);
// 如果参数为 "--chunks"，则获取下一个参数作为块数，并转换为整数保存到params.n_chunks中
} else if (arg == "--chunks") {
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    params.n_chunks = std::stoi(argv[i]);
// 如果参数为 "-np" 或 "--parallel"，则获取下一个参数作为并行数，并转换为整数保存到params.n_parallel中
} else if (arg == "-np" || arg == "--parallel") {
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    params.n_parallel = std::stoi(argv[i]);
// 如果参数为 "-ns" 或 "--sequences"，则获取下一个参数作为序列数，并转换为整数保存到params.n_sequences中
} else if (arg == "-ns" || arg == "--sequences") {
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    params.n_sequences = std::stoi(argv[i]);
// 如果参数为 "--p-accept" 或 "-pa"，则获取下一个参数作为接受概率，并转换为整数保存到params.p_accept中
} else if (arg == "--p-accept" || arg == "-pa") {
    if (++i >= argc) {
                # 检查参数是否有效，如果无效则设置标志位并跳出循环
                invalid_param = true;
                break;
            }
            # 如果参数是"--p-accept"或者"-pa"，则将下一个参数转换为浮点数并赋值给params.p_accept
            params.p_accept = std::stof(argv[i]);
        } else if (arg == "--p-split" || arg == "-ps") {
            # 如果参数是"--p-split"或者"-ps"，则将下一个参数转换为浮点数并赋值给params.p_split
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.p_split = std::stof(argv[i]);
        } else if (arg == "-m" || arg == "--model") {
            # 如果参数是"-m"或者"--model"，则将下一个参数赋值给params.model
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.model = argv[i];
        } else if (arg == "-md" || arg == "--model-draft") {
            # 如果参数是"-md"或者"--model-draft"，则将下一个参数赋值给params.model
            if (++i >= argc) {
                invalid_param = true;
                break;
        } 
        // 如果参数是"-m"或"--model"，则将下一个参数作为模型草稿文件名
        params.model_draft = argv[i];
    } else if (arg == "-a" || arg == "--alias") {
        // 如果参数是"-a"或"--alias"，则将下一个参数作为模型别名
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params.model_alias = argv[i];
    } else if (arg == "--lora") {
        // 如果参数是"--lora"，则将下一个参数作为LORA适配器的地址，并将默认权重1.0f添加到参数列表中
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        params.lora_adapter.push_back(std::make_tuple(argv[i], 1.0f));
        // 设置使用内存映射为false
        params.use_mmap = false;
    } else if (arg == "--lora-scaled") {
        // 如果参数是"--lora-scaled"，则将下一个参数作为经过缩放的LORA适配器的地址
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
// 将命令行参数赋值给变量 lora_adapter
const char * lora_adapter = argv[i];
// 如果参数数量不足，设置 invalid_param 为 true 并跳出循环
if (++i >= argc) {
    invalid_param = true;
    break;
}
// 将 lora_adapter 和 argv[i] 转换为浮点数后存入 params.lora_adapter
params.lora_adapter.push_back(std::make_tuple(lora_adapter, std::stof(argv[i])));
// 设置 params.use_mmap 为 false
params.use_mmap = false;
} else if (arg == "--lora-base") {
// 如果参数为 "--lora-base"，将下一个参数赋值给 params.lora_base
if (++i >= argc) {
    invalid_param = true;
    break;
}
params.lora_base = argv[i];
} else if (arg == "--reset-gpu-index") {
// 如果参数为 "--reset-gpu-index"，将 params.reset_gpu_index 设置为 true
params.reset_gpu_index = true;
} else if (arg == "--disable-gpu-index") {
// 如果参数为 "--disable-gpu-index"，将 params.disale_gpu_index 设置为 true
params.disale_gpu_index = true;
} else if (arg == "--mmproj") {
// 如果参数为 "--mmproj"，将下一个参数赋值给 params.mmproj
if (++i >= argc) {
    invalid_param = true;
            # 如果参数为 "--mmproj"，则将下一个参数作为 mmproj 的值
            if (arg == "--mmproj") {
                if (++i >= argc) {
                    invalid_param = true;
                    break;
                }
                params.mmproj = argv[i];
            } 
            # 如果参数为 "--image"，则将下一个参数作为 image 的值
            else if (arg == "--image") {
                if (++i >= argc) {
                    invalid_param = true;
                    break;
                }
                params.image = argv[i];
            } 
            # 如果参数为 "-i" 或 "--interactive"，则将 interactive 设为 true
            else if (arg == "-i" || arg == "--interactive") {
                params.interactive = true;
            } 
            # 如果参数为 "--embedding"，则将 embedding 设为 true
            else if (arg == "--embedding") {
                params.embedding = true;
            } 
            # 如果参数为 "--interactive-first"，则将 interactive_first 设为 true
            else if (arg == "--interactive-first") {
                params.interactive_first = true;
            } 
            # 如果参数为 "-ins" 或 "--instruct"，则将 instruct 设为 true
            else if (arg == "-ins" || arg == "--instruct") {
                params.instruct = true;
            } 
            # 如果参数为 "--infill"，则将 infill 设为 true
            else if (arg == "--infill") {
                params.infill = true;
            } 
            # 如果参数为 "--multiline-input"，则将 multiline_input 设为 true
            else if (arg == "--multiline-input") {
        // 如果参数为 "--multiline-input"，则设置参数为 true
        if (arg == "--multiline-input") {
            params.multiline_input = true;
        } 
        // 如果参数为 "--simple-io"，则设置参数为 true
        else if (arg == "--simple-io") {
            params.simple_io = true;
        } 
        // 如果参数为 "-cb" 或 "--cont-batching"，则设置参数为 true
        else if (arg == "-cb" || arg == "--cont-batching") {
            params.cont_batching = true;
        } 
        // 如果参数为 "--color"，则设置参数为 true
        else if (arg == "--color") {
            params.use_color = true;
        } 
        // 如果参数为 "--mlock"，则设置参数为 true
        else if (arg == "--mlock") {
            params.use_mlock = true;
        } 
        // 如果参数为 "--gpu-layers" 或 "-ngl" 或 "--n-gpu-layers"
        else if (arg == "--gpu-layers" || arg == "-ngl" || arg == "--n-gpu-layers") {
            // 如果下一个参数是有效的数字，则设置参数为该数字
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            #ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
            params.n_gpu_layers = std::stoi(argv[i]);
            #else
            // 如果没有 GPU offload 支持，则打印警告信息
            fprintf(stderr, "warning: not compiled with GPU offload support, --n-gpu-layers option will be ignored\n");
            fprintf(stderr, "warning: see main README.md for information on enabling GPU BLAS support\n");
            #endif
        }
// 如果参数是"--gpu-layers-draft"、"-ngld"或"--n-gpu-layers-draft"，则执行以下操作
        } else if (arg == "--gpu-layers-draft" || arg == "-ngld" || arg == "--n-gpu-layers-draft") {
            // 如果参数不足，将标记为无效参数并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 如果编译时支持 GPU offload，则将参数转换为整数并赋值给params.n_gpu_layers_draft
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
            params.n_gpu_layers_draft = std::stoi(argv[i]);
#else
            // 如果未编译支持 GPU offload，则输出警告信息
            fprintf(stderr, "warning: not compiled with GPU offload support, --n-gpu-layers-draft option will be ignored\n");
            fprintf(stderr, "warning: see main README.md for information on enabling GPU BLAS support\n");
#endif
        // 如果参数是"--main-gpu"或"-mg"，则执行以下操作
        } else if (arg == "--main-gpu" || arg == "-mg") {
            // 如果参数不足，将标记为无效参数并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 如果编译时支持 cuBLAS，则将参数转换为整数并赋值给params.main_gpu
#ifdef GGML_USE_CUBLAS
            params.main_gpu = std::stoi(argv[i]);
#else
            // 如果未编译支持 cuBLAS，则输出警告信息
            fprintf(stderr, "warning: llama.cpp was compiled without cuBLAS. It is not possible to set a main GPU.\n");
// 如果参数为 "--tensor-split" 或 "-ts"，则执行以下操作
        } else if (arg == "--tensor-split" || arg == "-ts") {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 获取下一个参数
            std::string arg_next = argv[i];

            // 通过 , 和 / 分割字符串
            const std::regex regex{R"([,/]+)"};
            std::sregex_token_iterator it{arg_next.begin(), arg_next.end(), regex, -1};
            // 将分割后的字符串存储到 split_arg 中
            std::vector<std::string> split_arg{it, {}};
            // 确保分割后的字符串数量不超过 LLAMA_MAX_DEVICES
            GGML_ASSERT(split_arg.size() <= LLAMA_MAX_DEVICES);

            // 遍历 LLAMA_MAX_DEVICES
            for (size_t i = 0; i < LLAMA_MAX_DEVICES; ++i) {
                // 如果 i 小于 split_arg 的大小，将 split_arg[i] 转换为浮点数并存储到 params.tensor_split[i] 中
                if (i < split_arg.size()) {
                    params.tensor_split[i] = std::stof(split_arg[i]);
                } else {
                    // 否则将 params.tensor_split[i] 设置为 0.0f
                    params.tensor_split[i] = 0.0f;
// 如果没有定义 GGML_USE_CUBLAS，则输出警告信息
#else
            fprintf(stderr, "warning: llama.cpp was compiled without cuBLAS. It is not possible to set a tensor split.\n");
#endif // GGML_USE_CUBLAS

// 如果参数为 "--vram-budget"，则设置 VRAM 预算
        } else if (arg == "--vram-budget") {
            // 如果参数不足，则设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 如果定义了 GGML_USE_CUBLAS，则将参数转换为浮点数并赋值给 vram_budget_gb
            params.vram_budget_gb = std::stof(argv[i]);
#else
            // 如果没有定义 GGML_USE_CUBLAS，则输出警告信息
            fprintf(stderr, "warning: PowerInfer was compiled without cuBLAS. It is not possible to set a VRAM budget.\n");
#endif

// 如果参数为 "--no-mul-mat-q" 或 "-nommq"，则禁用 mul_mat_q
        } else if (arg == "--no-mul-mat-q" || arg == "-nommq") {
#ifdef GGML_USE_CUBLAS
            // 如果定义了 GGML_USE_CUBLAS，则将 mul_mat_q 设置为 false
            params.mul_mat_q = false;
#else
            // 如果没有定义 GGML_USE_CUBLAS，则输出警告信息
            fprintf(stderr, "warning: llama.cpp was compiled without cuBLAS. Disabling mul_mat_q kernels has no effect.\n");
// 如果定义了 GGML_USE_CUBLAS，则执行相应的操作
#else // GGML_USE_CUBLAS
        // 如果参数为 "--no-mmap"，则设置参数 use_mmap 为 false
        } else if (arg == "--no-mmap") {
            params.use_mmap = false;
        // 如果参数为 "--numa"，则设置参数 numa 为 true
        } else if (arg == "--numa") {
            params.numa = true;
        // 如果参数为 "--verbose-prompt"，则设置参数 verbose_prompt 为 true
        } else if (arg == "--verbose-prompt") {
            params.verbose_prompt = true;
        // 如果参数为 "-r" 或 "--reverse-prompt"，则将下一个参数添加到 antiprompt 数组中
        } else if (arg == "-r" || arg == "--reverse-prompt") {
            // 如果下一个参数不存在，则设置 invalid_param 为 true 并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.antiprompt.push_back(argv[i]);
        // 如果参数为 "-ld" 或 "--logdir"，则将下一个参数设置为 logdir
        } else if (arg == "-ld" || arg == "--logdir") {
            // 如果下一个参数不存在，则设置 invalid_param 为 true 并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.logdir = argv[i];
        // 如果日志目录的最后一个字符不是目录分隔符，则添加目录分隔符
        if (params.logdir.back() != DIRECTORY_SEPARATOR) {
            params.logdir += DIRECTORY_SEPARATOR;
        }
        // 如果参数是"--perplexity"或者"--all-logits"，则设置params.logits_all为true
        else if (arg == "--perplexity" || arg == "--all-logits") {
            params.logits_all = true;
        }
        // 如果参数是"--ppl-stride"，则将下一个参数转换为整数并赋值给params.ppl_stride
        else if (arg == "--ppl-stride") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.ppl_stride = std::stoi(argv[i]);
        }
        // 如果参数是"--ppl-output-type"，则将下一个参数转换为整数并赋值给params.ppl_output_type
        else if (arg == "--ppl-output-type") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.ppl_output_type = std::stoi(argv[i]);
        }
        // 如果参数是"--hellaswag"，则设置params.hellaswag为true
        else if (arg == "--hellaswag") {
            params.hellaswag = true;
        }
        // 如果参数是"--hellaswag-tasks"，则...
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将字符串类型的参数转换为整数并赋值给params.hellaswag_tasks
            params.hellaswag_tasks = std::stoi(argv[i]);
        } else if (arg == "--ignore-eos") {
            # 设置params.ignore_eos为true
            params.ignore_eos = true;
        } else if (arg == "--no-penalize-nl") {
            # 设置sparams.penalize_nl为false
            sparams.penalize_nl = false;
        } else if (arg == "-l" || arg == "--logit-bias") {
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 使用stringstream将参数转换为特定格式的数据
            std::stringstream ss(argv[i]);
            llama_token key;
            char sign;
            std::string value_str;
            try {
                # 从stringstream中读取数据并进行特定格式的处理
                if (ss >> key && ss >> sign && std::getline(ss, value_str) && (sign == '+' || sign == '-')) {
                sparams.logit_bias[key] = std::stof(value_str) * ((sign == '-') ? -1.0f : 1.0f);
                // 将字符串转换为浮点数，并根据符号进行乘法运算，然后赋值给logit_bias[key]

            } else {
                throw std::exception();
                // 如果条件不满足，则抛出异常
            }
        } catch (const std::exception&) {
            invalid_param = true;
            // 捕获异常后，将invalid_param设置为true，然后跳出循环
            break;
        } else if (arg == "-h" || arg == "--help") {
            return false;
            // 如果参数为"-h"或"--help"，则返回false

        } else if (arg == "--random-prompt") {
            params.random_prompt = true;
            // 如果参数为"--random-prompt"，则将params.random_prompt设置为true
        } else if (arg == "--in-prefix-bos") {
            params.input_prefix_bos = true;
            // 如果参数为"--in-prefix-bos"，则将params.input_prefix_bos设置为true
        } else if (arg == "--in-prefix") {
            if (++i >= argc) {
                invalid_param = true;
                // 如果参数为"--in-prefix"，则检查下一个参数是否存在，若不存在则将invalid_param设置为true，然后跳出循环
                break;
            }
        # 如果参数是输入前缀，则将其赋值给params.input_prefix
        params.input_prefix = argv[i];
        # 如果参数是输入后缀，则将其赋值给params.input_suffix
        } else if (arg == "--in-suffix") {
            # 检查下一个参数是否存在，如果不存在则设置invalid_param为true并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.input_suffix = argv[i];
        # 如果参数是语法，则将其赋值给sparams.grammar
        } else if (arg == "--grammar") {
            # 检查下一个参数是否存在，如果不存在则设置invalid_param为true并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            sparams.grammar = argv[i];
        # 如果参数是语法文件，则打开文件并将内容赋值给sparams.grammar
        } else if (arg == "--grammar-file") {
            # 检查下一个参数是否存在，如果不存在则设置invalid_param为true并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            std::ifstream file(argv[i]);
            # 如果文件无法打开，则执行以下操作
            if (!file) {
// 打印错误信息，指出无法打开文件
fprintf(stderr, "error: failed to open file '%s'\n", argv[i]);
// 设置参数为无效，表示发生错误
invalid_param = true;
// 跳出循环，终止文件处理
break;
// 从文件流中复制数据到参数结构体中的语法字段
std::copy(
    std::istreambuf_iterator<char>(file),
    std::istreambuf_iterator<char>(),
    std::back_inserter(sparams.grammar)
);
// 如果日志功能未被禁用
#ifndef LOG_DISABLE_LOGS
// 解析参数以获取日志参数
} else if ( log_param_single_parse( argv[i] ) ) {
    // 什么也不做，log_param_single_parse 会自动执行其操作
    // 并返回是否找到匹配并解析
} else if ( log_param_pair_parse( /*check_but_dont_parse*/ true, argv[i] ) ) {
    // 我们有一个匹配的已知参数需要一个参数，
    // 现在我们需要检查这个 argv 后面是否还有内容
    // 并标记为无效参数或解析它
    if (++i >= argc) {
        // 如果没有参数，标记为无效参数
        invalid_param = true;
        // 如果遇到分号，跳出循环
        break;
    }
    // 如果参数不符合要求，设置参数无效标志并跳出循环
    if( !log_param_pair_parse( /*check_but_dont_parse*/ false, argv[i-1], argv[i]) ) {
        invalid_param = true;
        break;
    }
    // 解析日志参数结束
#endif // LOG_DISABLE_LOGS
    // 如果遇到未知参数，抛出异常
    } else {
        throw std::invalid_argument("error: unknown argument: " + arg);
    }
}
// 如果参数无效，抛出异常
if (invalid_param) {
    throw std::invalid_argument("error: invalid parameter for argument: " + arg);
}
// 如果参数提示缓存全部并且处于交互模式、交互模式优先或者指示模式下，抛出异常
if (params.prompt_cache_all &&
        (params.interactive || params.interactive_first ||
         params.instruct)) {
    throw std::invalid_argument("error: --prompt-cache-all not supported in interactive mode yet\n");
    }
    // 如果参数中包含转义字符，对相关字符串进行转义处理
    if (params.escape) {
        process_escapes(params.prompt);
        process_escapes(params.input_prefix);
        process_escapes(params.input_suffix);
        process_escapes(sparams.cfg_negative_prompt);
        for (auto & antiprompt : params.antiprompt) {
            process_escapes(antiprompt);
        }
    }
    // 返回 true，表示处理成功
    return true;
}

// 打印程序的使用方法
void gpt_print_usage(int /*argc*/, char ** argv, const gpt_params & params) {
    // 获取参数中的采样参数
    const llama_sampling_params & sparams = params.sparams;

    // 打印程序使用方法的说明
    printf("\n");
    printf("usage: %s [options]\n", argv[0]);
# 打印一个换行符
printf("\n");
# 打印帮助信息
printf("options:\n");
# 打印各种选项及其说明
printf("  -h, --help            show this help message and exit\n");
printf("  -i, --interactive     run in interactive mode\n");
printf("  --interactive-first   run in interactive mode and wait for input right away\n");
printf("  -ins, --instruct      run in instruction mode (use with Alpaca models)\n");
printf("  --multiline-input     allows you to write or paste multiple lines without ending each in '\\'\n");
printf("  -r PROMPT, --reverse-prompt PROMPT\n");
printf("                        halt generation at PROMPT, return control in interactive mode\n");
printf("                        (can be specified more than once for multiple prompts).\n");
printf("  --color               colorise output to distinguish prompt and user input from generations\n");
printf("  -s SEED, --seed SEED  RNG seed (default: -1, use random seed for < 0)\n");
printf("  -t N, --threads N     number of threads to use during generation (default: %d)\n", params.n_threads);
printf("  -tb N, --threads-batch N\n");
printf("                        number of threads to use during batch and prompt processing (default: same as --threads)\n");
printf("  -p PROMPT, --prompt PROMPT\n");
printf("                        prompt to start generation with (default: empty)\n");
printf("  -e, --escape          process prompt escapes sequences (\\n, \\r, \\t, \\', \\\", \\\\)\n");
printf("  --prompt-cache FNAME  file to cache prompt state for faster startup (default: none)\n");
printf("  --prompt-cache-all    if specified, saves user input and generations to cache as well.\n");
# 打印提示信息，指出在交互模式下或其他交互选项下不支持该功能
printf("                        not supported with --interactive or other interactive options\n");
# 打印提示信息，指出如果指定了该选项，则使用提示缓存但不更新它
printf("  --prompt-cache-ro     if specified, uses the prompt cache but does not update it.\n");
# 打印提示信息，指出以随机化的提示开始
printf("  --random-prompt       start with a randomized prompt.\n");
# 打印提示信息，指出在用户输入前在`--in-prefix`字符串之前添加BOS
printf("  --in-prefix-bos       prefix BOS to user inputs, preceding the `--in-prefix` string\n");
# 打印提示信息，指出用于在用户输入前添加的字符串（默认为空）
printf("  --in-prefix STRING    string to prefix user inputs with (default: empty)\n");
# 打印提示信息，指出用于在用户输入后添加的字符串（默认为空）
printf("  --in-suffix STRING    string to suffix after user inputs with (default: empty)\n");
# 打印提示信息，指出用于开始生成的提示文件
printf("  -f FNAME, --file FNAME\n");
# 打印提示信息，指出要预测的标记数（默认：%d，-1 = 无限，-2 = 直到上下文填满）
printf("  -n N, --n-predict N   number of tokens to predict (default: %d, -1 = infinity, -2 = until context filled)\n", params.n_predict);
# 打印提示信息，指出提示上下文的大小（默认：%d，0 = 从模型加载）
printf("  -c N, --ctx-size N    size of the prompt context (default: %d, 0 = loaded from model)\n", params.n_ctx);
# 打印提示信息，指出用于提示处理的批处理大小（默认：%d）
printf("  -b N, --batch-size N  batch size for prompt processing (default: %d)\n", params.n_batch);
# 打印提示信息，指出使用 top-k 抽样（默认：%d，0 = 禁用）
printf("  --top-k N             top-k sampling (default: %d, 0 = disabled)\n", sparams.top_k);
# 打印提示信息，指出使用 top-p 抽样（默认：%.1f，1.0 = 禁用）
printf("  --top-p N             top-p sampling (default: %.1f, 1.0 = disabled)\n", (double)sparams.top_p);
# 打印提示信息，指出使用 min-p 抽样（默认：%.1f，0.0 = 禁用）
printf("  --min-p N             min-p sampling (default: %.1f, 0.0 = disabled)\n", (double)sparams.min_p);
# 打印提示信息，指出使用 tail free 抽样，参数 z（默认：%.1f，1.0 = 禁用）
printf("  --tfs N               tail free sampling, parameter z (default: %.1f, 1.0 = disabled)\n", (double)sparams.tfs_z);
# 打印提示信息，指出使用局部典型抽样，参数 p（默认：%.1f，1.0 = 禁用）
printf("  --typical N           locally typical sampling, parameter p (default: %.1f, 1.0 = disabled)\n", (double)sparams.typical_p);
# 打印提示信息，指出考虑惩罚的最后 n 个标记（默认：%d，0 = 禁用，-1 = ctx_size）
printf("  --repeat-last-n N     last n tokens to consider for penalize (default: %d, 0 = disabled, -1 = ctx_size)\n", sparams.penalty_last_n);
# 打印提示信息，指出惩罚重复标记序列（默认：%.1f，1.0 = 禁用）
printf("  --repeat-penalty N    penalize repeat sequence of tokens (default: %.1f, 1.0 = disabled)\n", (double)sparams.penalty_repeat);
# 打印提示信息，指出重复 alpha 存在惩罚（默认：%.1f，0.0 = 禁用）
printf("  --presence-penalty N  repeat alpha presence penalty (default: %.1f, 0.0 = disabled)\n", (double)sparams.penalty_present);
# 打印提示信息，指出重复 alpha 频率惩罚（默认：%.1f，0.0 = 禁用）
printf("  --frequency-penalty N repeat alpha frequency penalty (default: %.1f, 0.0 = disabled)\n", (double)sparams.penalty_freq);
# 打印使用 Mirostat 抽样的选项及说明
printf("  --mirostat N          use Mirostat sampling.\n");
# 打印 Mirostat 学习率的选项及默认值
printf("  --mirostat-lr N       Mirostat learning rate, parameter eta (default: %.1f)\n", (double)sparams.mirostat_eta);
# 打印 Mirostat 目标熵的选项及默认值
printf("  --mirostat-ent N      Mirostat target entropy, parameter tau (default: %.1f)\n", (double)sparams.mirostat_tau);
# 打印修改完成中 token 出现概率的选项及说明
printf("  -l TOKEN_ID(+/-)BIAS, --logit-bias TOKEN_ID(+/-)BIAS\n");
# 打印用于限制生成的类似 BNF 的语法选项及说明
printf("  --grammar GRAMMAR     BNF-like grammar to constrain generations (see samples in grammars/ dir)\n");
# 打印从文件中读取语法的选项及说明
printf("  --grammar-file FNAME  file to read grammar from\n");
# 打印用于指导的负面提示选项及说明
printf("  --cfg-negative-prompt PROMPT\n");
# 打印从文件中读取负面提示的选项及说明
printf("  --cfg-negative-prompt-file FNAME\n");
# 打印指导强度的选项及默认值
printf("  --cfg-scale N         strength of guidance (default: %f, 1.0 = disable)\n", sparams.cfg_scale);
# 打印 RoPE 频率缩放方法的选项及说明
printf("  --rope-scaling {none,linear,yarn}\n");
# 打印 RoPE 上下文缩放因子的选项及说明
printf("  --rope-scale N        RoPE context scaling factor, expands context by a factor of N\n");
# 打印 RoPE 基础频率的选项及说明
printf("  --rope-freq-base N    RoPE base frequency, used by NTK-aware scaling (default: loaded from model)\n");
    # 打印 RoPE 频率缩放因子的说明
    printf("  --rope-freq-scale N   RoPE frequency scaling factor, expands context by a factor of 1/N\n");
    # 打印 YaRN 模型的原始上下文大小说明
    printf("  --yarn-orig-ctx N     YaRN: original context size of model (default: 0 = model training context size)\n");
    # 打印 YaRN 模型的外推混合因子说明
    printf("  --yarn-ext-factor N   YaRN: extrapolation mix factor (default: 1.0, 0.0 = full interpolation)\n");
    # 打印 YaRN 模型的注意力因子说明
    printf("  --yarn-attn-factor N  YaRN: scale sqrt(t) or attention magnitude (default: 1.0)\n");
    # 打印 YaRN 模型的高校正维度或 alpha 的默认值说明
    printf("  --yarn-beta-slow N    YaRN: high correction dim or alpha (default: %.1f)\n", params.yarn_beta_slow);
    # 打印 YaRN 模型的低校正维度或 beta 的默认值说明
    printf("  --yarn-beta-fast N    YaRN: low correction dim or beta (default: %.1f)\n", params.yarn_beta_fast);
    # 打印忽略流结束标记并继续生成的说明
    printf("  --ignore-eos          ignore end of stream token and continue generating (implies --logit-bias 2-inf)\n");
    # 打印不惩罚换行符的说明
    printf("  --no-penalize-nl      do not penalize newline token\n");
    # 打印使用 f32 而不是 f16 作为内存键值的说明
    printf("  --memory-f32          use f32 instead of f16 for memory key+value (default: disabled)\n");
    # 打印温度的默认值说明
    printf("  --temp N              temperature (default: %.1f)\n", (double)sparams.temp);
    # 打印 VRAM 预算的默认值说明
    printf("  --vram-budget N       VRAM budget in GiB (default: -1, -1 = available VRAM)\n");
    # 打印返回批处理中所有标记的对数概率的说明
    printf("  --logits-all          return logits for all tokens in the batch (default: disabled)\n");
    # 打印计算 HellaSwag 分数的说明
    printf("  --hellaswag           compute HellaSwag score over random tasks from datafile supplied with -f\n");
    # 打印计算 HellaSwag 分数时使用的任务数量的默认值说明
    printf("  --hellaswag-tasks N   number of tasks to use when computing the HellaSwag score (default: %zu)\n", params.hellaswag_tasks);
    # 打印从初始提示中保留的标记数量的默认值说明
    printf("  --keep N              number of tokens to keep from the initial prompt (default: %d, -1 = all)\n", params.n_keep);
    # 打印用于推测解码的标记数量的默认值说明
    printf("  --draft N             number of tokens to draft for speculative decoding (default: %d)\n", params.n_draft);
    # 打印要处理的最大块数的默认值说明
    printf("  --chunks N            max number of chunks to process (default: %d, -1 = all)\n", params.n_chunks);
    # 打印并行解码的默认值说明
    printf("  -np N, --parallel N   number of parallel sequences to decode (default: %d)\n", params.n_parallel);
    # 打印解码序列的默认值说明
    printf("  -ns N, --sequences N  number of sequences to decode (default: %d)\n", params.n_sequences);
    # 打印带有参数的字符串，显示 speculative decoding accept probability
    printf("  -pa N, --p-accept N   speculative decoding accept probability (default: %.1f)\n", (double)params.p_accept);
    # 打印带有参数的字符串，显示 speculative decoding split probability
    printf("  -ps N, --p-split N    speculative decoding split probability (default: %.1f)\n", (double)params.p_split);
    # 打印字符串，显示启用连续批处理
    printf("  -cb, --cont-batching  enable continuous batching (a.k.a dynamic batching) (default: disabled)\n");
    # 打印字符串，显示路径到多模态投影仪文件
    printf("  --mmproj MMPROJ_FILE  path to a multimodal projector file for LLaVA. see examples/llava/README.md\n");
    # 打印字符串，显示路径到图像文件
    printf("  --image IMAGE_FILE    path to an image file. use with multimodal models\n");
    # 如果系统支持内存锁定，则打印字符串，显示强制系统保持模型在 RAM 中
    if (llama_mlock_supported()) {
        printf("  --mlock               force system to keep model in RAM rather than swapping or compressing\n");
    }
    # 如果系统支持内存映射，则打印字符串，显示不使用内存映射模型
    if (llama_mmap_supported()) {
        printf("  --no-mmap             do not memory-map model (slower load but may reduce pageouts if not using mlock)\n");
    }
    # 打印字符串，显示尝试在一些 NUMA 系统上帮助的优化
    printf("  --numa                attempt optimizations that help on some NUMA systems\n");
    # 打印字符串，显示建议在使用 --numa 之前清除系统页缓存
    printf("                        if run without this previously, it is recommended to drop the system page cache before using this\n");
    # 打印字符串，显示链接到 GitHub 上的问题
    printf("                        see https://github.com/ggerganov/llama.cpp/issues/1437\n");
    # 如果 LLAMA 支持 GPU 卸载，则打印字符串，显示存储在 VRAM 中的层数
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
    printf("  -ngl N, --n-gpu-layers N\n");
    # 打印字符串，显示存储在 VRAM 中的草稿模型的层数
    printf("  -ngld N, --n-gpu-layers-draft N\n");
    # 打印字符串，显示张量分割
    printf("  -ts SPLIT --tensor-split SPLIT\n");
    # 打印关于在多个 GPU 上如何分割张量的说明
    printf("                        how to split tensors across multiple GPUs, comma-separated list of proportions, e.g. 3,1\n");
    # 打印关于主 GPU 的说明
    printf("  -mg i, --main-gpu i   the GPU to use for scratch and small tensors\n");
    # 如果使用了 GGML_USE_CUBLAS，则打印关于使用 GGML_CUBLAS_NAME 的说明
#ifdef GGML_USE_CUBLAS
    printf("  -nommq, --no-mul-mat-q\n");
    printf("                        use " GGML_CUBLAS_NAME " instead of custom mul_mat_q " GGML_CUDA_NAME " kernels.\n");
    printf("                        Not recommended since this is both slower and uses more VRAM.\n");
#endif // GGML_USE_CUBLAS
#endif
    # 打印关于是否打印生成前提示的说明
    printf("  --verbose-prompt      print prompt before generation\n");
    # 打印关于是否使用基本 IO 的说明
    printf("  --simple-io           use basic IO for better compatibility in subprocesses and limited consoles\n");
    # 打印关于应用 LoRA 适配器的说明
    printf("  --lora FNAME          apply LoRA adapter (implies --no-mmap)\n");
    # 打印关于应用具有用户定义缩放的 LoRA 适配器的说明
    printf("  --lora-scaled FNAME S apply LoRA adapter with user defined scaling S (implies --no-mmap)\n");
    # 打印关于可选模型的说明
    printf("  --lora-base FNAME     optional model to use as a base for the layers modified by the LoRA adapter\n");
    # 打印关于模型路径的说明
    printf("  -m FNAME, --model FNAME\n");
    printf("                        model path (default: %s)\n", params.model.c_str());
    # 打印关于草稿模型路径的说明
    printf("  -md FNAME, --model-draft FNAME\n");
    printf("                        draft model for speculative decoding (default: %s)\n", params.model.c_str());
    # 打印关于日志目录路径的说明
    printf("  -ld LOGDIR, --logdir LOGDIR\n");
    printf("                        path under which to save YAML logs (no logging if unset)\n");
    # 打印空行
    printf("\n");
#ifndef LOG_DISABLE_LOGS
    // 如果未定义 LOG_DISABLE_LOGS，则打印使用日志
    log_print_usage();
#endif // LOG_DISABLE_LOGS
}

// 获取系统信息
std::string get_system_info(const gpt_params & params) {
    std::ostringstream os;

    // 将系统信息输出到字符串流中
    os << "system_info: n_threads = " << params.n_threads;
    // 如果存在批处理线程数，则输出到字符串流中
    if (params.n_threads_batch != -1) {
        os << " (n_threads_batch = " << params.n_threads_batch << ")";
    }
    // 输出硬件并发线程数和系统信息到字符串流中
    os << " / " << std::thread::hardware_concurrency() << " | " << llama_print_system_info();

    // 将字符串流转换为字符串并返回
    return os.str();
}

// 生成随机提示
std::string gpt_random_prompt(std::mt19937 & rng) {
    // 生成一个0到9的随机数
    const int r = rng() % 10;
    // 根据随机数进行不同的操作
    switch (r) {
// 根据不同的情况返回不同的字符串
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

// 如果程序执行到这里，表示出现了不可达的情况
GGML_UNREACHABLE();
}

//
// 模型工具
//

// 从 GPT 参数中创建 Llama 模型参数
struct llama_model_params llama_model_params_from_gpt_params(const gpt_params & params) {
// 使用默认参数创建LLAMA模型参数对象
auto mparams = llama_model_default_params();

// 如果参数中指定了n_gpu_layers，则更新mparams中的n_gpu_layers
if (params.n_gpu_layers != -1) {
    mparams.n_gpu_layers = params.n_gpu_layers;
}

// 更新mparams中的其他参数
mparams.main_gpu        = params.main_gpu;
mparams.vram_budget_gb  = params.vram_budget_gb;
mparams.tensor_split    = params.tensor_split;
mparams.use_mmap        = params.use_mmap;
mparams.use_mlock       = params.use_mlock;
mparams.reset_gpu_index = params.reset_gpu_index;
mparams.disable_gpu_index = params.disale_gpu_index;

// 返回更新后的mparams
return mparams;
}

// 根据GPT参数创建LLAMA上下文参数对象
struct llama_context_params llama_context_params_from_gpt_params(const gpt_params & params) {
    // 使用默认参数创建LLAMA上下文参数对象
    auto cparams = llama_context_default_params();

    // 更新cparams中的n_ctx参数
    cparams.n_ctx             = params.n_ctx;
    # 设置C参数的批次大小为params的批次大小
    cparams.n_batch           = params.n_batch;
    # 设置C参数的线程数为params的线程数
    cparams.n_threads         = params.n_threads;
    # 如果params的批次线程数为-1，则C参数的批次线程数为params的线程数，否则为params的批次线程数
    cparams.n_threads_batch   = params.n_threads_batch == -1 ? params.n_threads : params.n_threads_batch;
    # 设置C参数的矩阵乘积为params的矩阵乘积
    cparams.mul_mat_q         = params.mul_mat_q;
    # 设置C参数的种子为params的种子
    cparams.seed              = params.seed;
    # 设置C参数的f16_kv为params的内存f16
    cparams.f16_kv            = params.memory_f16;
    # 设置C参数的所有logits为params的所有logits
    cparams.logits_all        = params.logits_all;
    # 设置C参数的嵌入为params的嵌入
    cparams.embedding         = params.embedding;
    # 设置C参数的绳索缩放类型为params的绳索缩放类型
    cparams.rope_scaling_type = params.rope_scaling_type;
    # 设置C参数的绳索频率基数为params的绳索频率基数
    cparams.rope_freq_base    = params.rope_freq_base;
    # 设置C参数的绳索频率比例为params的绳索频率比例
    cparams.rope_freq_scale   = params.rope_freq_scale;
    # 设置C参数的纱线扩展因子为params的纱线扩展因子
    cparams.yarn_ext_factor   = params.yarn_ext_factor;
    # 设置C参数的纱线注意力因子为params的纱线注意力因子
    cparams.yarn_attn_factor  = params.yarn_attn_factor;
    # 设置C参数的纱线快速beta为params的纱线快速beta
    cparams.yarn_beta_fast    = params.yarn_beta_fast;
    # 设置C参数的纱线慢速beta为params的纱线慢速beta
    cparams.yarn_beta_slow    = params.yarn_beta_slow;
    # 设置C参数的纱线原始上下文为params的纱线原始上下文

    return cparams;
}
// 清空批处理对象中的数据
void llama_batch_clear(struct llama_batch & batch) {
    // 将批处理对象中的令牌数量重置为0
    batch.n_tokens = 0;
}

// 向批处理对象中添加数据
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
    // 将序列ID的数量添加到批处理对象的序列ID数量数组中
    batch.n_seq_id[batch.n_tokens] = seq_ids.size();
    // 遍历序列ID数组，将每个序列ID添加到批处理对象的序列ID数组中
    for (size_t i = 0; i < seq_ids.size(); ++i) {
        batch.seq_id[batch.n_tokens][i] = seq_ids[i];
    }
    // 将logits值添加到批处理对象的logits数组中
    batch.logits  [batch.n_tokens] = logits;

    // 增加批处理对象中的令牌数量
    batch.n_tokens++;
}
// 从 GPT 参数初始化 LLAMA 模型和上下文
std::tuple<struct llama_model *, struct llama_context *> llama_init_from_gpt_params(gpt_params & params) {
    // 从 GPT 参数创建 LLAMA 模型参数
    auto mparams = llama_model_params_from_gpt_params(params);

    // 从文件加载 LLAMA 模型
    llama_model * model  = llama_load_model_from_file(params.model.c_str(), mparams);
    // 如果加载失败，打印错误信息并返回空指针
    if (model == NULL) {
        fprintf(stderr, "%s: error: failed to load model '%s'\n", __func__, params.model.c_str());
        return std::make_tuple(nullptr, nullptr);
    }

    // 从 GPT 参数创建 LLAMA 上下文参数
    auto cparams = llama_context_params_from_gpt_params(params);
    // 使用模型创建 LLAMA 上下文
    llama_context * lctx = llama_new_context_with_model(model, cparams);
    // 如果创建失败，打印错误信息，释放模型并返回空指针
    if (lctx == NULL) {
        fprintf(stderr, "%s: error: failed to create context with model '%s'\n", __func__, params.model.c_str());
        llama_free_model(model);
        return std::make_tuple(nullptr, nullptr);
    }

    // 遍历 params.lora_adapter 中的适配器
    for (unsigned int i = 0; i < params.lora_adapter.size(); ++i) {
        // 获取当前适配器的名称
        const std::string& lora_adapter = std::get<0>(params.lora_adapter[i]);
// 从参数中获取第i个lora适配器的缩放比例
float lora_scale = std::get<1>(params.lora_adapter[i]);
// 将lora适配器应用到模型中
int err = llama_model_apply_lora_from_file(model,
                                     lora_adapter.c_str(),
                                     lora_scale,
                                     // 如果i大于0或者params.lora_base为空，则传入NULL，否则传入params.lora_base的C风格字符串
                                     ((i > 0) || params.lora_base.empty())
                                        ? NULL
                                        : params.lora_base.c_str(),
                                     params.n_threads);
// 如果应用lora适配器出现错误
if (err != 0) {
    // 打印错误信息
    fprintf(stderr, "%s: error: failed to apply lora adapter\n", __func__);
    // 释放内存
    llama_free(lctx);
    llama_free_model(model);
    // 返回空指针元组
    return std::make_tuple(nullptr, nullptr);
}

// 如果参数中设置忽略EOS（end of sentence）
if (params.ignore_eos) {
    // 将EOS的logit偏置设置为负无穷
    params.sparams.logit_bias[llama_token_eos(model)] = -INFINITY;
}
// 打印日志，提示模型进行空运行的预热
LOG("warming up the model with an empty run\n");

// 创建一个包含开始和结束标记的临时 llma_token 向量
std::vector<llama_token> tmp = { llama_token_bos(model), llama_token_eos(model), };

// 使用 llma_batch_get_one 函数对临时向量进行解码
llama_decode(lctx, llama_batch_get_one(tmp.data(), std::min(tmp.size(), (size_t) params.n_batch), 0, 0));

// 清空 llma_kv_cache
llama_kv_cache_clear(lctx);

// 重置 llma_timing
llama_reset_timings(lctx);

// 返回模型和上下文的元组
return std::make_tuple(model, lctx);
}

//
// 词汇工具
//

// 使用 llama_context 对象进行分词
std::vector<llama_token> llama_tokenize(
  const struct llama_context * ctx,
// 调用 llama_tokenize 函数，传入上下文、文本、是否添加开头标记和特殊标记，并返回结果
std::vector<llama_token> llama_tokenize(
    const struct llama_model * model,
    const std::string & text,
    bool   add_bos,
    bool   special) {
    // 计算 tokens 的上限
    int n_tokens = text.length() + add_bos;
    // 创建一个包含 n_tokens 个元素的 llama_token 类型的向量
    std::vector<llama_token> result(n_tokens);
    // 调用 llama_tokenize 函数，传入模型、文本数据、文本长度、结果数据、结果大小、是否添加开头标记和特殊标记，并返回 tokens 的数量
    n_tokens = llama_tokenize(model, text.data(), text.length(), result.data(), result.size(), add_bos, special);
    // 如果 tokens 的数量小于 0，重新调整结果向量的大小，并再次调用 llama_tokenize 函数
    if (n_tokens < 0) {
        result.resize(-n_tokens);
        int check = llama_tokenize(model, text.data(), text.length(), result.data(), result.size(), add_bos, special);
        // 断言检查
        GGML_ASSERT(check == -n_tokens);
    } else {
// 调整结果向量的大小为指定的标记数量
result.resize(n_tokens);
}

// 将 llama_token 转换为字符串片段
std::string llama_token_to_piece(const struct llama_context * ctx, llama_token token) {
    // 创建一个包含8个元素的字符向量，并初始化为0
    std::vector<char> result(8, 0);
    // 调用 llama_token_to_piece 函数将 token 转换为字符串片段，并返回转换后的标记数量
    const int n_tokens = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
    // 如果返回的标记数量小于0，调整结果向量的大小为负的标记数量，并再次调用 llama_token_to_piece 函数
    if (n_tokens < 0) {
        result.resize(-n_tokens);
        int check = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
        // 断言检查结果是否为负的标记数量
        GGML_ASSERT(check == -n_tokens);
    } else {
        // 调整结果向量的大小为标记数量
        result.resize(n_tokens);
    }

    // 返回结果向量转换后的字符串
    return std::string(result.data(), result.size());
}

// 使用 llama_context 和标记向量进行解码
std::string llama_detokenize_spm(llama_context * ctx, const std::vector<llama_token> & tokens) {
// 获取当前上下文中的模型，并使用其创建一个 BOS 标记
const llama_token bos_id = llama_token_bos(llama_get_model(ctx));

// 定义两个字符串变量，用于存储处理过程中的片段和最终结果
std::string piece;
std::string result;

// 遍历 tokens 数组，处理每个 token
for (size_t i = 0; i < tokens.size(); ++i) {
    // 将 token 转换为对应的片段
    piece = llama_token_to_piece(ctx, tokens[i]);

    // 如果是第一个非 BOS token，则去除片段开头的空格
    if (((tokens[0] == bos_id && i == 1) || (tokens[0] != bos_id && i == 0)) && piece[0] == ' ') {
        piece = piece.substr(1);
    }

    // 将处理后的片段添加到结果字符串中
    result += piece;
}

// 返回最终的结果字符串
return result;
}
    // 声明两个字符串变量用于存储片段和结果
    std::string piece;
    std::string result;

    // 遍历 tokens 数组，将每个 token 转换为片段并拼接到结果字符串中
    for (size_t i = 0; i < tokens.size(); ++i) {
        piece = llama_token_to_piece(ctx, tokens[i]);

        result += piece;
    }

    // 注意：原始的分词器在收集片段后解码字节
    // 返回拼接后的结果字符串
    return result;
}

//
// YAML 工具函数
//

// 如果成功创建目录则返回 true，否则返回 false
bool create_directory_with_parents(const std::string & path) {
#ifdef _WIN32
    // 创建一个用于转换字符串编码的对象，从UTF-8编码转换为宽字符编码
    std::wstring_convert<std::codecvt_utf8<wchar_t>> converter;
    // 使用转换器将路径从字节转换为宽字符
    std::wstring wpath = converter.from_bytes(path);

    // 如果路径已经存在，检查它是否是一个目录
    const DWORD attributes = GetFileAttributesW(wpath.c_str());
    if ((attributes != INVALID_FILE_ATTRIBUTES) && (attributes & FILE_ATTRIBUTE_DIRECTORY)) {
        return true;
    }

    // 初始化斜杠位置
    size_t pos_slash = 0;

    // 从前到后处理路径，逐步创建目录
    while ((pos_slash = path.find('\\', pos_slash)) != std::string::npos) {
        // 获取子路径
        const std::wstring subpath = wpath.substr(0, pos_slash);
        // 将子路径转换为C风格的宽字符数组
        const wchar_t * test = subpath.c_str();

        // 创建目录
        const bool success = CreateDirectoryW(test, NULL);
        // 如果创建失败，获取错误码
        if (!success) {
            const DWORD error = GetLastError();
// 如果路径已经存在，确保它是一个目录
if (error == ERROR_ALREADY_EXISTS) {
    // 获取文件属性
    const DWORD attributes = GetFileAttributesW(subpath.c_str());
    // 如果获取属性失败或者不是一个目录，返回false
    if (attributes == INVALID_FILE_ATTRIBUTES || !(attributes & FILE_ATTRIBUTE_DIRECTORY)) {
        return false;
    }
} else {
    return false;
}

// 更新路径分隔符的位置
pos_slash += 1;
}

// 返回true
return true;
#else
// 如果路径已经存在，检查它是否是一个目录
struct stat info;
// 获取文件信息
if (stat(path.c_str(), &info) == 0) {
    // 检查是否是一个目录
    return S_ISDIR(info.st_mode);
    }

    size_t pos_slash = 1; // 跳过路径开头的斜杠，用于创建目录

    // 从前向后处理路径，逐步创建目录
    while ((pos_slash = path.find('/', pos_slash)) != std::string::npos) {
        const std::string subpath = path.substr(0, pos_slash);
        struct stat info;

        // 如果路径已经存在，确保它是一个目录
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

    // 增加斜杠位置的索引
    pos_slash += 1;
}

// 如果是 Windows 平台，返回 true
return true;
#endif // _WIN32
}

// 将浮点数向量以 YAML 格式输出到文件流
void dump_vector_float_yaml(FILE * stream, const char * prop_name, const std::vector<float> & data) {
    // 如果数据为空，输出属性名并换行
    if (data.empty()) {
        fprintf(stream, "%s:\n", prop_name);
        return;
    }

    // 输出属性名和左括号
    fprintf(stream, "%s: [", prop_name);
    // 遍历数据，输出每个元素和逗号
    for (size_t i = 0; i < data.size() - 1; ++i) {
        fprintf(stream, "%e, ", data[i]);
    }
    // 输出最后一个元素和右括号，并换行
    fprintf(stream, "%e]\n", data.back());
```

// 将整数向量以YAML格式输出到文件流中
void dump_vector_int_yaml(FILE * stream, const char * prop_name, const std::vector<int> & data) {
    // 如果数据为空，输出属性名并换行
    if (data.empty()) {
        fprintf(stream, "%s:\n", prop_name);
        return;
    }

    // 输出属性名和向量数据的起始标记
    fprintf(stream, "%s: [", prop_name);
    // 遍历整数向量，输出每个元素和逗号
    for (size_t i = 0; i < data.size() - 1; ++i) {
        fprintf(stream, "%d, ", data[i]);
    }
    // 输出最后一个元素和结束标记
    fprintf(stream, "%d]\n", data.back());
}

// 将多行字符串以YAML格式输出到文件流中
void dump_string_yaml_multiline(FILE * stream, const char * prop_name, const char * data) {
    // 将C风格字符串转换为C++风格字符串
    std::string data_str(data == NULL ? "" : data);

    // 如果字符串为空，输出属性名并换行
    if (data_str.empty()) {
        fprintf(stream, "%s:\n", prop_name);
        return;
    }
    // 初始化起始位置和找到位置
    size_t pos_start = 0;
    size_t pos_found = 0;

    // 如果数据字符串不为空且以空格开头或结尾
    if (!data_str.empty() && (std::isspace(data_str[0]) || std::isspace(data_str.back()))) {
        // 替换换行符为"\n"
        data_str = std::regex_replace(data_str, std::regex("\n"), "\\n");
        // 替换双引号为"\""
        data_str = std::regex_replace(data_str, std::regex("\""), "\\\"");
        // 在数据字符串前后加上双引号
        data_str = "\"" + data_str + "\"";
        // 将属性名和处理后的数据字符串写入流
        fprintf(stream, "%s: %s\n", prop_name, data_str.c_str());
        return;
    }

    // 如果数据字符串中不包含换行符
    if (data_str.find('\n') == std::string::npos) {
        // 将属性名和数据字符串写入流
        fprintf(stream, "%s: %s\n", prop_name, data_str.c_str());
        return;
    }

    // 将属性名写入流
    fprintf(stream, "%s: |\n", prop_name);
    // 在数据字符串中查找换行符的位置，返回值赋给pos_found，直到找不到换行符为止
    while ((pos_found = data_str.find('\n', pos_start)) != std::string::npos) {
        // 在流中打印从pos_start到pos_found之间的子字符串
        fprintf(stream, "  %s\n", data_str.substr(pos_start, pos_found-pos_start).c_str());
        // 更新pos_start的位置为pos_found加1
        pos_start = pos_found + 1;
    }
}

// 获取可排序的时间戳字符串
std::string get_sortable_timestamp() {
    using clock = std::chrono::system_clock;

    // 获取当前时间点
    const clock::time_point current_time = clock::now();
    // 将时间点转换为time_t类型
    const time_t as_time_t = clock::to_time_t(current_time);
    // 格式化时间戳，不包括纳秒部分
    char timestamp_no_ns[100];
    std::strftime(timestamp_no_ns, 100, "%Y_%m_%d-%H_%M_%S", std::localtime(&as_time_t));

    // 计算当前时间点的纳秒部分
    const int64_t ns = std::chrono::duration_cast<std::chrono::nanoseconds>(
        current_time.time_since_epoch() % 1000000000).count();
    // 格式化纳秒部分的时间戳
    char timestamp_ns[11];
    snprintf(timestamp_ns, 11, "%09" PRId64, ns);

    // 返回包含无纳秒部分时间戳和纳秒部分时间戳的字符串
    return std::string(timestamp_no_ns) + "." + std::string(timestamp_ns);
// 将非结果信息以 YAML 格式输出到指定文件流中
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
}
    # 将cpu_has_f16c的结果写入流中，如果为真则写入"true"，否则写入"false"
    fprintf(stream, "cpu_has_f16c: %s\n",        ggml_cpu_has_f16c()        ? "true" : "false");
    # 将cpu_has_fp16_va的结果写入流中，如果为真则写入"true"，否则写入"false"
    fprintf(stream, "cpu_has_fp16_va: %s\n",     ggml_cpu_has_fp16_va()     ? "true" : "false");
    # 将cpu_has_wasm_simd的结果写入流中，如果为真则写入"true"，否则写入"false"
    fprintf(stream, "cpu_has_wasm_simd: %s\n",   ggml_cpu_has_wasm_simd()   ? "true" : "false");
    # 将cpu_has_blas的结果写入流中，如果为真则写入"true"，否则写入"false"
    fprintf(stream, "cpu_has_blas: %s\n",        ggml_cpu_has_blas()        ? "true" : "false");
    # 将cpu_has_sse3的结果写入流中，如果为真则写入"true"，否则写入"false"
    fprintf(stream, "cpu_has_sse3: %s\n",        ggml_cpu_has_sse3()        ? "true" : "false");
    # 将cpu_has_vsx的结果写入流中，如果为真则写入"true"，否则写入"false"
    fprintf(stream, "cpu_has_vsx: %s\n",         ggml_cpu_has_vsx()         ? "true" : "false");

    # 如果处于非调试模式，则写入"debug: false"，否则写入"debug: true"
#ifdef NDEBUG
    fprintf(stream, "debug: false\n");
#else
    fprintf(stream, "debug: true\n");
#endif // NDEBUG

    # 将model_desc的内容写入流中
    fprintf(stream, "model_desc: %s\n", model_desc);
    # 将n_vocab的结果写入流中，同时注释说明其为最终层的输出大小，某些模型为32001
    fprintf(stream, "n_vocab: %d  # output size of the final layer, 32001 for some models\n", llama_n_vocab(llama_get_model(lctx)));

    # 如果处于优化模式，则写入"optimize: true"，否则写入"optimize: false"
#ifdef __OPTIMIZE__
    fprintf(stream, "optimize: true\n");
#else
    fprintf(stream, "optimize: false\n");
#endif // __OPTIMIZE__
// 结束条件标记，表示代码块结束
#endif // __OPTIMIZE__

// 将时间戳输出到流中
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
// 输出多行字符串参数的信息
dump_string_yaml_multiline(stream, "cfg_negative_prompt", sparams.cfg_negative_prompt.c_str());
fprintf(stream, "cfg_scale: %f # default: 1.0\n", sparams.cfg_scale);
fprintf(stream, "chunks: %d # default: -1 (unlimited)\n", params.n_chunks);
fprintf(stream, "color: %s # default: false\n", params.use_color ? "true" : "false");
fprintf(stream, "ctx_size: %d # default: 512\n", params.n_ctx);
fprintf(stream, "escape: %s # default: false\n", params.escape ? "true" : "false");
fprintf(stream, "file: # never logged, see prompt instead. Can still be specified for input.\n");
fprintf(stream, "frequency_penalty: %f # default: 0.0 \n", sparams.penalty_freq);
    # 将字符串以多行形式写入到YAML文件流中，键为"grammar"，值为sparams.grammar的C风格字符串
    dump_string_yaml_multiline(stream, "grammar", sparams.grammar.c_str());
    # 将字符串以格式化形式写入到文件流中，用于注释，不会被记录，但仍然可以作为输入指定
    fprintf(stream, "grammar-file: # never logged, see grammar instead. Can still be specified for input.\n");
    # 将字符串以格式化形式写入到文件流中，键为"hellaswag"，值为params.hellaswag的布尔值（true或false），默认为false
    fprintf(stream, "hellaswag: %s # default: false\n", params.hellaswag ? "true" : "false");
    # 将整数以格式化形式写入到文件流中，键为"hellaswag_tasks"，值为params.hellaswag_tasks的值，默认为400
    fprintf(stream, "hellaswag_tasks: %zu # default: 400\n", params.hellaswag_tasks);

    # 查找logit_bias中是否存在llama_token_eos(llama_get_model(lctx))，并将结果赋值给logit_bias_eos
    const auto logit_bias_eos = sparams.logit_bias.find(llama_token_eos(llama_get_model(lctx)));
    # 判断是否忽略EOS（End of Sentence），并将结果以格式化形式写入到文件流中，键为"ignore_eos"，默认为false
    const bool ignore_eos = logit_bias_eos != sparams.logit_bias.end() && logit_bias_eos->second == -INFINITY;
    fprintf(stream, "ignore_eos: %s # default: false\n", ignore_eos ? "true" : "false");

    # 将字符串以多行形式写入到YAML文件流中，键为"in_prefix"，值为params.input_prefix的C风格字符串
    dump_string_yaml_multiline(stream, "in_prefix", params.input_prefix.c_str());
    # 将字符串以格式化形式写入到文件流中，键为"in_prefix_bos"，值为params.input_prefix_bos的布尔值（true或false），默认为false
    fprintf(stream, "in_prefix_bos: %s # default: false\n", params.input_prefix_bos ? "true" : "false");
    # 将字符串以多行形式写入到YAML文件流中，键为"in_suffix"，值为params.input_prefix的C风格字符串
    dump_string_yaml_multiline(stream, "in_suffix", params.input_prefix.c_str());
    # 将字符串以格式化形式写入到文件流中，键为"instruct"，值为params.instruct的布尔值（true或false），默认为false
    fprintf(stream, "instruct: %s # default: false\n", params.instruct ? "true" : "false");
    # 将字符串以格式化形式写入到文件流中，键为"interactive"，值为params.interactive的布尔值（true或false），默认为false
    fprintf(stream, "interactive: %s # default: false\n", params.interactive ? "true" : "false");
    # 将字符串以格式化形式写入到文件流中，键为"interactive_first"，值为params.interactive_first的布尔值（true或false），默认为false
    fprintf(stream, "interactive_first: %s # default: false\n", params.interactive_first ? "true" : "false");
    # 将整数以格式化形式写入到文件流中，键为"keep"，值为params.n_keep的值，默认为0
    fprintf(stream, "keep: %d # default: 0\n", params.n_keep);
    # 将字符串以格式化形式写入到文件流中，键为"logdir"，值为params.logdir的C风格字符串，默认为unset (no logging)
    fprintf(stream, "logdir: %s # default: unset (no logging)\n", params.logdir.c_str());

    # 将字符串以格式化形式写入到文件流中，键为"logit_bias"，值为sparams.logit_bias的键值对
    fprintf(stream, "logit_bias:\n");
    # 遍历sparams.logit_bias中的键值对，将结果写入到文件流中
    for (std::pair<llama_token, float> lb : sparams.logit_bias) {
        // 如果忽略 EOS 并且当前 lb 的第一个元素等于 logit_bias_eos 的第一个元素，则跳过当前循环
        if (ignore_eos && lb.first == logit_bias_eos->first) {
            continue;
        }
        // 将 lb 的第一个元素和第二个元素写入流中
        fprintf(stream, "  %d: %f", lb.first, lb.second);
    }

    // 写入流中 lora:
    fprintf(stream, "lora:\n");
    // 遍历 lora_adapter 中的元组，如果第二个元素不等于 1.0f，则跳过当前循环
    for (std::tuple<std::string, float> la : params.lora_adapter) {
        if (std::get<1>(la) != 1.0f) {
            continue;
        }
        // 将第一个元素写入流中
        fprintf(stream, "  - %s\n", std::get<0>(la).c_str());
    }
    // 写入流中 lora_scaled:
    fprintf(stream, "lora_scaled:\n");
    // 遍历 lora_adapter 中的元组，如果第二个元素等于 1.0f，则跳过当前循环
    for (std::tuple<std::string, float> la : params.lora_adapter) {
        if (std::get<1>(la) == 1.0f) {
            continue;
        }
        // 将第一个元素和第二个元素写入流中
        fprintf(stream, "  - %s: %f\n", std::get<0>(la).c_str(), std::get<1>(la));
    }
# 将 params.lora_base 转换为 C 风格的字符串并写入 stream
fprintf(stream, "lora_base: %s\n", params.lora_base.c_str());
# 将 params.reset_gpu_index 转换为字符串并写入 stream
fprintf(stream, "reset_gpu_index: %s\n", params.reset_gpu_index ? "true" : "false");
# 将 params.disale_gpu_index 转换为字符串并写入 stream
fprintf(stream, "disable_gpu_index: %s\n", params.disale_gpu_index? "true": "false");
# 将 params.main_gpu 写入 stream
fprintf(stream, "main_gpu: %d # default: 0\n", params.main_gpu);
# 将 !params.memory_f16 转换为字符串并写入 stream
fprintf(stream, "memory_f32: %s # default: false\n", !params.memory_f16 ? "true" : "false");
# 将 sparams.mirostat 写入 stream
fprintf(stream, "mirostat: %d # default: 0 (disabled)\n", sparams.mirostat);
# 将 sparams.mirostat_tau 写入 stream
fprintf(stream, "mirostat_ent: %f # default: 5.0\n", sparams.mirostat_tau);
# 将 sparams.mirostat_eta 写入 stream
fprintf(stream, "mirostat_lr: %f # default: 0.1\n", sparams.mirostat_eta);
# 将 params.use_mlock 转换为字符串并写入 stream
fprintf(stream, "mlock: %s # default: false\n", params.use_mlock ? "true" : "false");
# 将 params.model 转换为 C 风格的字符串并写入 stream
fprintf(stream, "model: %s # default: models/7B/ggml-model.bin\n", params.model.c_str());
# 将 params.model_draft 转换为 C 风格的字符串并写入 stream
fprintf(stream, "model_draft: %s # default:\n", params.model_draft.c_str());
# 将 params.multiline_input 转换为字符串并写入 stream
fprintf(stream, "multiline_input: %s # default: false\n", params.multiline_input ? "true" : "false");
# 将 params.n_gpu_layers 写入 stream
fprintf(stream, "n_gpu_layers: %d # default: -1\n", params.n_gpu_layers);
# 将 params.n_predict 写入 stream
fprintf(stream, "n_predict: %d # default: -1 (unlimited)\n", params.n_predict);
# 将 sparams.n_probs 写入 stream
fprintf(stream, "n_probs: %d # only used by server binary, default: 0\n", sparams.n_probs);
# 将 !params.use_mmap 转换为字符串并写入 stream
fprintf(stream, "no_mmap: %s # default: false\n", !params.use_mmap ? "true" : "false");
# 将 !params.mul_mat_q 转换为字符串并写入 stream
fprintf(stream, "no_mul_mat_q: %s # default: false\n", !params.mul_mat_q ? "true" : "false");
# 将 !sparams.penalize_nl 转换为字符串并写入 stream
fprintf(stream, "no_penalize_nl: %s # default: false\n", !sparams.penalize_nl ? "true" : "false");
# 将 params.numa 转换为字符串并写入 stream
fprintf(stream, "numa: %s # default: false\n", params.numa ? "true" : "false");
# 将 params.ppl_output_type 写入 stream
fprintf(stream, "ppl_output_type: %d # default: 0\n", params.ppl_output_type);
    # 将 params.ppl_stride 的值写入流中，以及默认值注释
    fprintf(stream, "ppl_stride: %d # default: 0\n", params.ppl_stride);
    # 将 sparams.penalty_present 的值写入流中，以及默认值注释
    fprintf(stream, "presence_penalty: %f # default: 0.0\n", sparams.penalty_present);
    # 将 params.prompt 的值以多行形式写入流中
    dump_string_yaml_multiline(stream, "prompt", params.prompt.c_str());
    # 将 params.path_prompt_cache 的值写入流中
    fprintf(stream, "prompt_cache: %s\n", params.path_prompt_cache.c_str());
    # 将 params.prompt_cache_all 的值写入流中，以及默认值注释
    fprintf(stream, "prompt_cache_all: %s # default: false\n", params.prompt_cache_all ? "true" : "false");
    # 将 params.prompt_cache_ro 的值写入流中，以及默认值注释
    fprintf(stream, "prompt_cache_ro: %s # default: false\n", params.prompt_cache_ro ? "true" : "false");
    # 将 prompt_tokens 的值以数组形式写入流中
    dump_vector_int_yaml(stream, "prompt_tokens", prompt_tokens);
    # 将 params.random_prompt 的值写入流中，以及默认值注释
    fprintf(stream, "random_prompt: %s # default: false\n", params.random_prompt ? "true" : "false");
    # 将 sparams.penalty_repeat 的值写入流中，以及默认值注释
    fprintf(stream, "repeat_penalty: %f # default: 1.1\n", sparams.penalty_repeat);

    # 将反转后的 prompt 写入流中
    fprintf(stream, "reverse_prompt:\n");
    # 遍历 params.antiprompt，将每个字符串中的换行符替换为 "\n"，然后写入流中
    for (std::string ap : params.antiprompt) {
        size_t pos = 0;
        while ((pos = ap.find('\n', pos)) != std::string::npos) {
            ap.replace(pos, 1, "\\n");
            pos += 1;
        }
        fprintf(stream, "  - %s\n", ap.c_str());
    }
    // 将 params.rope_freq_base 写入流中，并添加注释
    fprintf(stream, "rope_freq_base: %f # default: 10000.0\n", params.rope_freq_base);
    // 将 params.rope_freq_scale 写入流中，并添加注释
    fprintf(stream, "rope_freq_scale: %f # default: 1.0\n", params.rope_freq_scale);
    // 将 params.seed 写入流中，并添加注释
    fprintf(stream, "seed: %d # default: -1 (random seed)\n", params.seed);
    // 将 params.simple_io 写入流中，并添加注释
    fprintf(stream, "simple_io: %s # default: false\n", params.simple_io ? "true" : "false");
    // 将 params.cont_batching 写入流中，并添加注释
    fprintf(stream, "cont_batching: %s # default: false\n", params.cont_batching ? "true" : "false");
    // 将 sparams.temp 写入流中，并添加注释
    fprintf(stream, "temp: %f # default: 0.8\n", sparams.temp);

    // 将 params.tensor_split 转换为 vector 后写入流中，并添加注释
    const std::vector<float> tensor_split_vector(params.tensor_split, params.tensor_split + LLAMA_MAX_DEVICES);
    dump_vector_float_yaml(stream, "tensor_split", tensor_split_vector);

    // 将 sparams.tfs_z 写入流中，并添加注释
    fprintf(stream, "tfs: %f # default: 1.0\n", sparams.tfs_z);
    // 将 params.n_threads 和硬件并发数写入流中，并添加注释
    fprintf(stream, "threads: %d # default: %d\n", params.n_threads, std::thread::hardware_concurrency());
    // 将 sparams.top_k 写入流中，并添加注释
    fprintf(stream, "top_k: %d # default: 40\n", sparams.top_k);
    // 将 sparams.top_p 写入流中，并添加注释
    fprintf(stream, "top_p: %f # default: 0.95\n", sparams.top_p);
    // 将 sparams.min_p 写入流中，并添加注释
    fprintf(stream, "min_p: %f # default: 0.0\n", sparams.min_p);
    // 将 sparams.typical_p 写入流中，并添加注释
    fprintf(stream, "typical_p: %f # default: 1.0\n", sparams.typical_p);
    // 将 params.verbose_prompt 写入流中，并添加注释
    fprintf(stream, "verbose_prompt: %s # default: false\n", params.verbose_prompt ? "true" : "false");
    // 将 params.vram_budget_gb 写入流中，并添加注释
    fprintf(stream, "vram_budget: %f # default: -1.0 (all available VRAM)\n", params.vram_budget_gb);
}
抱歉，我无法完成这个任务，因为给定的代码为空。
```