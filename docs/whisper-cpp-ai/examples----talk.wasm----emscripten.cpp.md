# `whisper.cpp\examples\talk.wasm\emscripten.cpp`

```cpp
// 包含必要的头文件
#include "ggml.h"
#include "gpt-2.h"
#include "whisper.h"

// 包含 Emscripten 相关的头文件
#include <emscripten.h>
#include <emscripten/bind.h>

// 包含必要的标准库头文件
#include <atomic>
#include <cmath>
#include <mutex>
#include <string>
#include <thread>
#include <vector>
#include <regex>

// 定义线程数量常量
constexpr int N_THREAD = 8;

// 定义全局变量
struct gpt2_context * g_gpt2;
std::vector<struct whisper_context *> g_contexts(4, nullptr);

std::mutex g_mutex;
std::thread g_worker;
std::atomic<bool> g_running(false);

bool g_force_speak = false;
std::string g_text_to_speak = "";
std::string g_status = "";
std::string g_status_forced = "";

std::vector<float> g_pcmf32;

// 将时间戳转换为字符串的函数
std::string to_timestamp(int64_t t) {
    int64_t sec = t/100;
    int64_t msec = t - sec*100;
    int64_t min = sec/60;
    sec = sec - min*60;

    char buf[32];
    snprintf(buf, sizeof(buf), "%02d:%02d.%03d", (int) min, (int) sec, (int) msec);

    return std::string(buf);
}

// 设置状态信息的函数
void talk_set_status(const std::string & status) {
    std::lock_guard<std::mutex> lock(g_mutex);
    g_status = status;
}

// 主要的对话处理函数
void talk_main(size_t index) {
    // 设置状态为 "loading data ..."
    talk_set_status("loading data ...");

    // 初始化 Whisper 参数
    struct whisper_full_params wparams = whisper_full_default_params(whisper_sampling_strategy::WHISPER_SAMPLING_GREEDY);

    // 设置参数值
    wparams.n_threads        = std::min(N_THREAD, (int) std::thread::hardware_concurrency());
    wparams.offset_ms        = 0;
    wparams.translate        = false;
    wparams.no_context       = true;
    wparams.single_segment   = true;
    wparams.print_realtime   = false;
    wparams.print_progress   = false;
    wparams.print_timestamps = true;
    wparams.print_special    = false;

    wparams.max_tokens       = 32;
    wparams.audio_ctx        = 768; // 部分编码器上下文以提高性能

    wparams.language         = "en";

    // 初始化 GPT-2 模型
    g_gpt2 = gpt2_init("gpt-2.bin");

    // 打印使用的线程数量
    printf("talk: using %d threads\n", wparams.n_threads);

    // 创建 PCM 数据向量
    std::vector<float> pcmf32;

    // Whisper 上下文
    auto & ctx = g_contexts[index];

    // 定义步长样本数量
    const int64_t step_samples   = 2*WHISPER_SAMPLE_RATE;
    // 定义窗口大小为9秒的采样数
    const int64_t window_samples = 9*WHISPER_SAMPLE_RATE;
    // 计算步长，单位为毫秒
    const int64_t step_ms = (step_samples*1000)/WHISPER_SAMPLE_RATE;

    // 获取当前时间
    auto t_last = std::chrono::high_resolution_clock::now();

    // 设置状态为"listening ..."
    talk_set_status("listening ...");

    // 释放 GPT2 模型资源
    gpt2_free(g_gpt2);

    // 如果索引小于上下文数组的大小
    if (index < g_contexts.size()) {
        // 释放指定索引处的上下文资源
        whisper_free(g_contexts[index]);
        // 将指定索引处的上下文指针置为空
        g_contexts[index] = nullptr;
    }
EMSCRIPTEN_BINDINGS(talk) {
    // 绑定 JavaScript 函数 "init"，初始化模型
    emscripten::function("init", emscripten::optional_override([](const std::string & path_model) {
        // 遍历上下文数组
        for (size_t i = 0; i < g_contexts.size(); ++i) {
            // 如果当前上下文为空
            if (g_contexts[i] == nullptr) {
                // 从文件初始化上下文
                g_contexts[i] = whisper_init_from_file_with_params(path_model.c_str(), whisper_context_default_params());
                // 如果初始化成功
                if (g_contexts[i] != nullptr) {
                    // 设置运行标志为 true
                    g_running = true;
                    // 如果工作线程可加入
                    if (g_worker.joinable()) {
                        g_worker.join();
                    }
                    // 创建新的工作线程
                    g_worker = std::thread([i]() {
                        talk_main(i);
                    });

                    // 返回上下文索引加一
                    return i + 1;
                } else {
                    // 返回 0 表示初始化失败
                    return (size_t) 0;
                }
            }
        }

        // 返回 0 表示初始化失败
        return (size_t) 0;
    }));

    // 绑定 JavaScript 函数 "free"，释放资源
    emscripten::function("free", emscripten::optional_override([](size_t index) {
        // 如果正在运行，设置运行标志为 false
        if (g_running) {
            g_running = false;
        }
    }));

    // 绑定 JavaScript 函数 "set_audio"，设置音频数据
    emscripten::function("set_audio", emscripten::optional_override([](size_t index, const emscripten::val & audio) {
        // 索引减一
        --index;

        // 如果索引超出上下文数组大小，返回错误码 -1
        if (index >= g_contexts.size()) {
            return -1;
        }

        // 如果上下文为空，返回错误码 -2
        if (g_contexts[index] == nullptr) {
            return -2;
        }

        {
            // 加锁
            std::lock_guard<std::mutex> lock(g_mutex);
            // 获取音频数据长度
            const int n = audio["length"].as<int>();

            // 获取内存堆和内存
            emscripten::val heap = emscripten::val::module_property("HEAPU8");
            emscripten::val memory = heap["buffer"];

            // 调整 PCM 数据大小
            g_pcmf32.resize(n);

            // 创建内存视图
            emscripten::val memoryView = audio["constructor"].new_(memory, reinterpret_cast<uintptr_t>(g_pcmf32.data()), n);
            memoryView.call<void>("set", audio);
        }

        // 返回成功
        return 0;
    }));

    // 绑定 JavaScript 函数 "force_speak"，强制发声
    emscripten::function("force_speak", emscripten::optional_override([](size_t index) {
        {
            // 加锁
            std::lock_guard<std::mutex> lock(g_mutex);
            // 设置强制发声标志为 true
            g_force_speak = true;
        }
    }));
}
    // 定义 JavaScript 函数 "get_text_context"，返回文本内容
    emscripten::function("get_text_context", emscripten::optional_override([]() {
        // 声明一个字符串变量用于存储文本内容
        std::string text_context;

        {
            // 使用互斥锁保护临界区，获取文本内容
            std::lock_guard<std::mutex> lock(g_mutex);
            text_context = gpt2_get_prompt(g_gpt2);
        }

        // 返回获取到的文本内容
        return text_context;
    }));

    // 定义 JavaScript 函数 "get_text_to_speak"，返回待朗读的文本
    emscripten::function("get_text_to_speak", emscripten::optional_override([]() {
        // 声明一个字符串变量用于存储待朗读的文本
        std::string text_to_speak;

        {
            // 使用互斥锁保护临界区，获取待朗读的文本
            std::lock_guard<std::mutex> lock(g_mutex);
            text_to_speak = std::move(g_text_to_speak);
        }

        // 返回获取到的待朗读文本
        return text_to_speak;
    }));

    // 定义 JavaScript 函数 "get_status"，返回状态信息
    emscripten::function("get_status", emscripten::optional_override([]() {
        // 声明一个字符串变量用于存储状态信息
        std::string status;

        {
            // 使用互斥锁保护临界区，获取状态信息
            std::lock_guard<std::mutex> lock(g_mutex);
            // 如果强制状态为空，则使用默认状态，否则使用强制状态
            status = g_status_forced.empty() ? g_status : g_status_forced;
        }

        // 返回获取到的状态信息
        return status;
    }));

    // 定义 JavaScript 函数 "set_status"，设置状态信息
    emscripten::function("set_status", emscripten::optional_override([](const std::string & status) {
        {
            // 使用互斥锁保护临界区，设置状态信息
            std::lock_guard<std::mutex> lock(g_mutex);
            g_status_forced = status;
        }
    }));

    // 定义 JavaScript 函数 "set_prompt"，设置提示信息
    emscripten::function("set_prompt", emscripten::optional_override([](const std::string & prompt) {
        {
            // 使用互斥锁保护临界区，设置提示信息
            std::lock_guard<std::mutex> lock(g_mutex);
            gpt2_set_prompt(g_gpt2, prompt.c_str());
        }
    });
# 闭合之前的代码块
```