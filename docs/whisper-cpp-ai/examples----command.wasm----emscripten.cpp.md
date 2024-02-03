# `whisper.cpp\examples\command.wasm\emscripten.cpp`

```cpp
// 包含所需的头文件
#include "ggml.h"
#include "common.h"
#include "whisper.h"

// 包含 Emscripten 相关的头文件
#include <emscripten.h>
#include <emscripten/bind.h>

// 包含所需的标准库头文件
#include <atomic>
#include <cmath>
#include <mutex>
#include <string>
#include <thread>
#include <vector>
#include <regex>

// 定义线程数量常量
constexpr int N_THREAD = 8;

// 全局变量，存储 whisper_context 指针的向量
std::vector<struct whisper_context *> g_contexts(4, nullptr);

// 全局互斥锁
std::mutex  g_mutex;
// 全局线程对象
std::thread g_worker;

// 全局原子布尔变量
std::atomic<bool> g_running(false);

// 全局字符串变量
std::string g_status        = "";
std::string g_status_forced = "";
std::string g_transcribed   = "";

// 全局浮点数向量
std::vector<float> g_pcmf32;

// 设置状态的函数
void command_set_status(const std::string & status) {
    // 使用互斥锁保护状态变量
    std::lock_guard<std::mutex> lock(g_mutex);
    g_status = status;
}

// 转录命令的函数
std::string command_transcribe(whisper_context * ctx, const whisper_full_params & wparams, const std::vector<float> & pcmf32, float & prob, int64_t & t_ms) {
    // 记录函数开始时间
    const auto t_start = std::chrono::high_resolution_clock::now();

    // 初始化概率和时间
    prob = 0.0f;
    t_ms = 0;

    // 调用 whisper_full 函数进行转录
    if (whisper_full(ctx, wparams, pcmf32.data(), pcmf32.size()) != 0) {
        return "";
    }

    int prob_n = 0;
    std::string result;

    // 获取转录结果
    const int n_segments = whisper_full_n_segments(ctx);
    for (int i = 0; i < n_segments; ++i) {
        const char * text = whisper_full_get_segment_text(ctx, i);

        result += text;

        const int n_tokens = whisper_full_n_tokens(ctx, i);
        for (int j = 0; j < n_tokens; ++j) {
            const auto token = whisper_full_get_token_data(ctx, i, j);

            prob += token.p;
            ++prob_n;
        }
    }

    // 计算平均概率
    if (prob_n > 0) {
        prob /= prob_n;
    }

    // 计算函数执行时间
    const auto t_end = std::chrono::high_resolution_clock::now();
    t_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t_end - t_start).count();

    return result;
}

// 获取音频数据的函数
void command_get_audio(int ms, int sample_rate, std::vector<float> & audio) {
    // 计算所需的采样数
    const int64_t n_samples = (ms * sample_rate) / 1000;

    int64_t n_take = 0;
    // 如果需要的采样数大于当前音频数据的长度
    if (n_samples > (int) g_pcmf32.size()) {
        n_take = g_pcmf32.size();
    }
    } else {
        // 如果样本数量小于等于音频数据的长度，则取样本数量为n_samples，否则取样本数量为音频数据的长度
        n_take = n_samples;
    }

    // 调整音频数据的大小为n_take
    audio.resize(n_take);
    // 将音频数据中最后n_take个元素复制到audio向量中
    std::copy(g_pcmf32.end() - n_take, g_pcmf32.end(), audio.begin());
// 定义命令处理函数，传入索引值
void command_main(size_t index) {
    // 设置命令状态为“加载数据中”
    command_set_status("loading data ...");

    // 使用默认参数创建 Whisper 全参数对象
    struct whisper_full_params wparams = whisper_full_default_params(whisper_sampling_strategy::WHISPER_SAMPLING_GREEDY);

    // 设置线程数为硬件支持的线程数和 N_THREAD 中的最小值
    wparams.n_threads        = std::min(N_THREAD, (int) std::thread::hardware_concurrency());
    wparams.offset_ms        = 0;
    wparams.translate        = false;
    wparams.no_context       = true;
    wparams.single_segment   = true;
    wparams.print_realtime   = false;
    wparams.print_progress   = false;
    wparams.print_timestamps = true;
    wparams.print_special    = false;

    // 设置最大令牌数和音频上下文
    wparams.max_tokens       = 32;
    wparams.audio_ctx        = 768; // 为了提高性能而设定的部分编码器上下文

    // 设置语言为英语
    wparams.language         = "en";

    // 打印使用的线程数
    printf("command: using %d threads\n", wparams.n_threads);

    // 初始化变量
    bool have_prompt  = false;
    bool ask_prompt   = true;
    bool print_energy = false;

    float prob0 = 0.0f;
    float prob  = 0.0f;

    std::vector<float> pcmf32_cur;
    std::vector<float> pcmf32_prompt;

    // 设置提示语
    const std::string k_prompt = "Ok Whisper, start listening for commands.";

    // Whisper 上下文
    auto & ctx = g_contexts[index];

    // 设置 VAD 时间、提示时间和命令时间
    const int32_t vad_ms     = 2000;
    const int32_t prompt_ms  = 5000;
    const int32_t command_ms = 4000;

    // 设置 VAD 和频率阈值
    const float vad_thold  = 0.1f;
    const float freq_thold = -1.0f;

    }

    // 如果索引小于上下文数组的大小
    if (index < g_contexts.size()) {
        // 释放对应索引的 Whisper 上下文
        whisper_free(g_contexts[index]);
        g_contexts[index] = nullptr;
    }
}

// 绑定命令
EMSCRIPTEN_BINDINGS(command) {
    // 初始化函数，接受一个模型路径参数，返回一个整数值
    emscripten::function("init", emscripten::optional_override([](const std::string & path_model) {
        // 遍历全局上下文数组
        for (size_t i = 0; i < g_contexts.size(); ++i) {
            // 如果当前上下文为空
            if (g_contexts[i] == nullptr) {
                // 从文件初始化上下文，并设置默认参数
                g_contexts[i] = whisper_init_from_file_with_params(path_model.c_str(), whisper_context_default_params());
                // 如果上下文初始化成功
                if (g_contexts[i] != nullptr) {
                    // 设置运行标志为真
                    g_running = true;
                    // 如果工作线程可加入
                    if (g_worker.joinable()) {
                        g_worker.join();
                    }
                    // 创建一个新线程执行 command_main 函数
                    g_worker = std::thread([i]() {
                        command_main(i);
                    });

                    // 返回当前上下文索引加一
                    return i + 1;
                } else {
                    // 如果上下文初始化失败，返回 0
                    return (size_t) 0;
                }
            }
        }

        // 如果所有上下文都已被占用，返回 0
        return (size_t) 0;
    }));

    // 释放函数，接受一个索引参数
    emscripten::function("free", emscripten::optional_override([](size_t index) {
        // 如果程序正在运行，将运行标志设为假
        if (g_running) {
            g_running = false;
        }
    }));

    // 设置音频函数，接受一个索引和音频参数
    emscripten::function("set_audio", emscripten::optional_override([](size_t index, const emscripten::val & audio) {
        // 索引减一
        --index;

        // 如果索引超出上下文数组大小，返回 -1
        if (index >= g_contexts.size()) {
            return -1;
        }

        // 如果当前上下文为空，返回 -2
        if (g_contexts[index] == nullptr) {
            return -2;
        }

        {
            // 使用互斥锁保护临界区
            std::lock_guard<std::mutex> lock(g_mutex);
            // 获取音频长度
            const int n = audio["length"].as<int>();

            // 获取内存堆和内存
            emscripten::val heap = emscripten::val::module_property("HEAPU8");
            emscripten::val memory = heap["buffer"];

            // 重新设置 PCM 数据大小
            g_pcmf32.resize(n);

            // 创建内存视图，将音频数据拷贝到 PCM 数据中
            emscripten::val memoryView = audio["constructor"].new_(memory, reinterpret_cast<uintptr_t>(g_pcmf32.data()), n);
            memoryView.call<void>("set", audio);
        }

        // 返回 0
        return 0;
    }));
    // 定义 JavaScript 函数 "get_transcribed"，返回 g_transcribed 的值
    emscripten::function("get_transcribed", emscripten::optional_override([]() {
        // 声明一个字符串变量 transcribed
        std::string transcribed;

        {
            // 使用互斥锁保护临界区，获取 g_transcribed 的值并移动给 transcribed
            std::lock_guard<std::mutex> lock(g_mutex);
            transcribed = std::move(g_transcribed);
        }

        // 返回 transcribed
        return transcribed;
    }));

    // 定义 JavaScript 函数 "get_status"，返回 g_status_forced 或 g_status 的值
    emscripten::function("get_status", emscripten::optional_override([]() {
        // 声明一个字符串变量 status
        std::string status;

        {
            // 使用互斥锁保护临界区，根据 g_status_forced 是否为空来选择返回 g_status 或 g_status_forced 的值
            std::lock_guard<std::mutex> lock(g_mutex);
            status = g_status_forced.empty() ? g_status : g_status_forced;
        }

        // 返回 status
        return status;
    }));

    // 定义 JavaScript 函数 "set_status"，设置 g_status_forced 的值
    emscripten::function("set_status", emscripten::optional_override([](const std::string & status) {
        {
            // 使用互斥锁保护临界区，设置 g_status_forced 的值为传入的 status
            std::lock_guard<std::mutex> lock(g_mutex);
            g_status_forced = status;
        }
    });
# 闭合之前的代码块
```