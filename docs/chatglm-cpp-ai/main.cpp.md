# `chatglm.cpp\main.cpp`

```
#include "chatglm.h"
#include <fstream>
#include <iomanip>
#include <iostream>

#ifdef _WIN32
#include <codecvt>
#include <fcntl.h>
#include <io.h>
#include <windows.h>
#endif

// 定义枚举类型 InferenceMode，表示推理模式
enum InferenceMode {
    INFERENCE_MODE_CHAT, // 聊天模式
    INFERENCE_MODE_GENERATE, // 生成模式
};

// 将字符串转换为对应的 InferenceMode 枚举值
static inline InferenceMode to_inference_mode(const std::string &s) {
    // 创建字符串到 InferenceMode 枚举值的映射
    static std::unordered_map<std::string, InferenceMode> m{{"chat", INFERENCE_MODE_CHAT},
                                                            {"generate", INFERENCE_MODE_GENERATE}};
    // 返回对应的 InferenceMode 枚举值
    return m.at(s);
}

// 定义结构体 Args，存储程序的参数
struct Args {
    std::string model_path = "chatglm-ggml.bin"; // 模型路径，默认为 chatglm-ggml.bin
    InferenceMode mode = INFERENCE_MODE_CHAT; // 推理模式，默认为聊天模式
    bool sync = false; // 是否同步，默认为 false
    std::string prompt = "你好"; // 提示语，默认为 "你好"
    std::string system = ""; // 系统信息，默认为空字符串
    int max_length = 2048; // 最大长度，默认为 2048
    int max_new_tokens = -1; // 最大新标记数，默认为 -1
    int max_context_length = 512; // 最大上下文长度，默认为 512
    bool interactive = false; // 是否交互，默认为 false
    int top_k = 0; // top-k 参数，默认为 0
    float top_p = 0.7; // top-p 参数，默认为 0.7
    float temp = 0.95; // 温度参数，默认为 0.95
    float repeat_penalty = 1.0; // 重复惩罚参数，默认为 1.0
    int num_threads = 0; // 线程数，默认为 0
    bool verbose = false; // 是否详细输出，默认为 false
};

// 显示程序的使用方法
static void usage(const std::string &prog) {
    std::cout << "Usage: " << prog << R"( [options]
// 定义命令行参数选项
options:
  -h, --help            show this help message and exit
  -m, --model PATH      model path (default: chatglm-ggml.bin)
  --mode                inference mode chosen from {chat, generate} (default: chat)
  --sync                synchronized generation without streaming
  -p, --prompt PROMPT   prompt to start generation with (default: 你好)
  --pp, --prompt_path   path to the plain text file that stores the prompt
  -s, --system SYSTEM   system message to set the behavior of the assistant
  --sp, --system_path   path to the plain text file that stores the system message
  -i, --interactive     run in interactive mode
  -l, --max_length N    max total length including prompt and output (default: 2048)
  --max_new_tokens N    max number of tokens to generate, ignoring the number of prompt tokens
  -c, --max_context_length N
                        max context length (default: 512)
  --top_k N             top-k sampling (default: 0)
  --top_p N             top-p sampling (default: 0.7)
  --temp N              temperature (default: 0.95)
  --repeat_penalty N    penalize repeat sequence of tokens (default: 1.0, 1.0 = disabled)
  -t, --threads N       number of threads for inference
  -v, --verbose         display verbose output including config/system/performance info

// 读取文本文件内容并返回为字符串
static std::string read_text(std::string path) {
    // 打开文件流
    std::ifstream fin(path);
    // 检查文件是否成功打开
    CHATGLM_CHECK(fin) << "cannot open file " << path;
    // 创建字符串流，读取文件内容
    std::ostringstream oss;
    oss << fin.rdbuf();
    // 返回文件内容字符串
    return oss.str();
}

// 解析命令行参数并返回Args对象
static Args parse_args(const std::vector<std::string> &argv) {
    // 创建Args对象
    Args args;

    // 返回Args对象
    return args;
}

// 解析命令行参数并返回Args对象
static Args parse_args(int argc, char **argv) {
    // 创建存储命令行参数的字符串向量
    std::vector<std::string> argv_vec;
    argv_vec.reserve(argc);

    // Windows平台特定处理
#ifdef _WIN32
    // 获取命令行参数
    LPWSTR *wargs = CommandLineToArgvW(GetCommandLineW(), &argc);
    // 检查是否成功获取命令行参数
    CHATGLM_CHECK(wargs) << "failed to retrieve command line arguments";

    // 创建Unicode和UTF-8字符串转换器
    std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
    // 遍历参数列表，将每个宽字符参数转换为多字节字符并添加到参数向量中
    for (int i = 0; i < argc; i++) {
        argv_vec.emplace_back(converter.to_bytes(wargs[i]));
    }

    // 释放宽字符参数数组的内存
    LocalFree(wargs);
#ifdef _WIN32
    // 如果是在 Windows 平台下编译
    std::wstring wline;
    // 读取一行 UTF-16 编码的输入
    bool ret = !!std::getline(std::wcin, wline);
    // 创建 UTF-8 到 UTF-16 的编码转换器
    std::wstring_convert<std::codecvt_utf8_utf16<wchar_t>> converter;
    // 将 UTF-16 编码的行转换为 UTF-8 编码
    line = converter.to_bytes(wline);
    // 返回读取是否成功
    return ret;
#else
    // 如果不是在 Windows 平台下编译
    // 读取一行 UTF-8 编码的输入
    return !!std::getline(std::cin, line);
#endif

static inline void print_message(const chatglm::ChatMessage &message) {
    // 打印消息内容
    std::cout << message.content << "\n";
    // 如果消息中包含工具调用且第一个工具调用类型为代码
    if (!message.tool_calls.empty() && message.tool_calls.front().type == chatglm::ToolCallMessage::TYPE_CODE) {
        // 打印工具调用中的输入代码
        std::cout << message.tool_calls.front().code.input << "\n";
    }
}

static void chat(Args &args) {
    // 初始化时间
    ggml_time_init();
    // 记录加载开始时间
    int64_t start_load_us = ggml_time_us();
    // 创建聊天管线
    chatglm::Pipeline pipeline(args.model_path);
    // 记录加载结束时间
    int64_t end_load_us = ggml_time_us();

    // 获取模型名称
    std::string model_name = pipeline.model->config.model_type_name();

    // 创建文本流输出器
    auto text_streamer = std::make_shared<chatglm::TextStreamer>(std::cout, pipeline.tokenizer.get());
    // 创建性能流输出器
    auto perf_streamer = std::make_shared<chatglm::PerfStreamer>();
    // 创建流输出器列表
    std::vector<std::shared_ptr<chatglm::BaseStreamer>> streamers{perf_streamer};
    // 如果不是同步模式，添加文本流输出器
    if (!args.sync) {
        streamers.emplace_back(text_streamer);
    }
    // 创建流输出器组
    auto streamer = std::make_unique<chatglm::StreamerGroup>(std::move(streamers));

    // 创建生成配置
    chatglm::GenerationConfig gen_config(args.max_length, args.max_new_tokens, args.max_context_length, args.temp > 0,
                                         args.top_k, args.top_p, args.temp, args.repeat_penalty, args.num_threads);
    // 如果 verbose 标志为真，则输出系统信息
    if (args.verbose) {
        // 输出系统支持的 SIMD 指令集信息
        std::cout << "system info: | "
                  << "AVX = " << ggml_cpu_has_avx() << " | "
                  << "AVX2 = " << ggml_cpu_has_avx2() << " | "
                  << "AVX512 = " << ggml_cpu_has_avx512() << " | "
                  << "AVX512_VBMI = " << ggml_cpu_has_avx512_vbmi() << " | "
                  << "AVX512_VNNI = " << ggml_cpu_has_avx512_vnni() << " | "
                  << "FMA = " << ggml_cpu_has_fma() << " | "
                  << "NEON = " << ggml_cpu_has_neon() << " | "
                  << "ARM_FMA = " << ggml_cpu_has_arm_fma() << " | "
                  << "F16C = " << ggml_cpu_has_f16c() << " | "
                  << "FP16_VA = " << ggml_cpu_has_fp16_va() << " | "
                  << "WASM_SIMD = " << ggml_cpu_has_wasm_simd() << " | "
                  << "BLAS = " << ggml_cpu_has_blas() << " | "
                  << "SSE3 = " << ggml_cpu_has_sse3() << " | "
                  << "VSX = " << ggml_cpu_has_vsx() << " |\n";

        // 输出推理配置信息
        std::cout << "inference config: | "
                  << "max_length = " << args.max_length << " | "
                  << "max_context_length = " << args.max_context_length << " | "
                  << "top_k = " << args.top_k << " | "
                  << "top_p = " << args.top_p << " | "
                  << "temperature = " << args.temp << " | "
                  << "repetition_penalty = " << args.repeat_penalty << " | "
                  << "num_threads = " << args.num_threads << " |\n";

        // 输出加载模型信息
        std::cout << "loaded " << pipeline.model->config.model_type_name() << " model from " << args.model_path
                  << " within: " << (end_load_us - start_load_us) / 1000.f << " ms\n";

        // 输出空行
        std::cout << std::endl;
    }

    // 如果模式不是聊天推理模式且交互标志为真，则输出警告信息并将交互标志设为假
    if (args.mode != INFERENCE_MODE_CHAT && args.interactive) {
        std::cerr << "interactive demo is only supported for chat mode, falling back to non-interactive one\n";
        args.interactive = false;
    }
    // 创建一个存储系统消息的 ChatMessage 对象的向量
    std::vector<chatglm::ChatMessage> system_messages;
    // 如果系统消息不为空，则将系统消息添加到系统消息向量中
    if (!args.system.empty()) {
        system_messages.emplace_back(chatglm::ChatMessage::ROLE_SYSTEM, args.system);
    }

    // 如果没有系统消息
    } else {
        // 如果模式为 INFERENCE_MODE_CHAT
        if (args.mode == INFERENCE_MODE_CHAT) {
            // 创建一个消息向量，并将系统消息添加到其中
            std::vector<chatglm::ChatMessage> messages = system_messages;
            // 添加用户消息到消息向量中
            messages.emplace_back(chatglm::ChatMessage::ROLE_USER, args.prompt);
            // 调用 pipeline 的 chat 方法生成输出消息
            chatglm::ChatMessage output = pipeline.chat(messages, gen_config, streamer.get());
            // 如果需要同步输出，则打印输出消息
            if (args.sync) {
                print_message(output);
            }
        } else {
            // 生成输出消息并存储在 output 中
            std::string output = pipeline.generate(args.prompt, gen_config, streamer.get());
            // 如果需要同步输出，则打印输出消息
            if (args.sync) {
                std::cout << output << "\n";
            }
        }
        // 如果需要详细输出
        if (args.verbose) {
            // 打印性能流的字符串表示
            std::cout << "\n" << perf_streamer->to_string() << "\n\n";
        }
    }
}

// 主函数，程序入口
int main(int argc, char **argv) {
#ifdef _WIN32
    // 设置控制台输出编码为 UTF-8
    SetConsoleOutputCP(CP_UTF8);
    // 设置标准输入流为宽字符模式
    _setmode(_fileno(stdin), _O_WTEXT);
#endif

    try {
        // 解析命令行参数
        Args args = parse_args(argc, argv);
        // 执行聊天功能
        chat(args);
    } catch (std::exception &e) {
        // 捕获异常并输出错误信息
        std::cerr << e.what() << std::endl;
        // 退出程序并返回失败状态
        exit(EXIT_FAILURE);
    }
    // 返回成功状态
    return 0;
}
```