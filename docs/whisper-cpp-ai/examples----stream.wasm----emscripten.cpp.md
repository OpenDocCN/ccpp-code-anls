# `whisper.cpp\examples\stream.wasm\emscripten.cpp`

```cpp
// 包含头文件 ggml.h 和 whisper.h
#include "ggml.h"
#include "whisper.h"

// 包含 Emscripten 相关头文件
#include <emscripten.h>
#include <emscripten/bind.h>

// 包含 C++ 标准库头文件
#include <atomic>
#include <cmath>
#include <mutex>
#include <string>
#include <thread>
#include <vector>

// 定义常量 N_THREAD 为 8
constexpr int N_THREAD = 8;

// 创建一个包含 4 个空指针的 whisper_context 结构体指针向量
std::vector<struct whisper_context *> g_contexts(4, nullptr);

// 创建一个互斥锁对象 g_mutex
std::mutex g_mutex;
// 创建一个线程对象 g_worker
std::thread g_worker;

// 创建一个原子布尔变量 g_running，并初始化为 false
std::atomic<bool> g_running(false);

// 创建三个字符串变量 g_status、g_status_forced、g_transcribed，并初始化为空字符串
std::string g_status        = "";
std::string g_status_forced = "";
std::string g_transcribed   = "";

// 创建一个浮点数向量 g_pcmf32
std::vector<float> g_pcmf32;

// 定义函数 stream_set_status，接受一个字符串参数 status，用于设置全局状态变量 g_status
void stream_set_status(const std::string & status) {
    // 使用互斥锁保护临界区
    std::lock_guard<std::mutex> lock(g_mutex);
    // 设置全局状态变量 g_status 为传入的 status
    g_status = status;
}

// 定义函数 stream_main，接受一个大小参数 index，用于处理音频流
void stream_main(size_t index) {
    // 设置状态为 "loading data ..."
    stream_set_status("loading data ...");

    // 创建一个 whisper_full_params 结构体 wparams，并使用默认参数初始化
    struct whisper_full_params wparams = whisper_full_default_params(whisper_sampling_strategy::WHISPER_SAMPLING_GREEDY);

    // 设置参数
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
    wparams.audio_ctx        = 768; // 部分编码器上下文，以提高性能

    // 禁用温度回退
    wparams.temperature_inc  = -1.0f;

    wparams.language         = "en";

    // 打印使用的线程数
    printf("stream: using %d threads\n", wparams.n_threads);

    // 创建一个浮点数向量 pcmf32
    std::vector<float> pcmf32;

    // 获取当前线程的 whisper_context
    auto & ctx = g_contexts[index];

    // 设置 5 秒间隔的窗口样本数
    const int64_t window_samples = 5*WHISPER_SAMPLE_RATE;
    // 当程序运行时
    while (g_running) {
        // 设置流状态为等待音频
        stream_set_status("waiting for audio ...");

        {
            // 创建互斥锁
            std::unique_lock<std::mutex> lock(g_mutex);

            // 如果 g_pcmf32 中的数据小于 1024
            if (g_pcmf32.size() < 1024) {
                // 解锁互斥锁
                lock.unlock();

                // 线程休眠 10 毫秒
                std::this_thread::sleep_for(std::chrono::milliseconds(10));

                // 继续下一次循环
                continue;
            }

            // 从 g_pcmf32 中截取最后 window_samples 个数据到 pcmf32
            pcmf32 = std::vector<float>(g_pcmf32.end() - std::min((int64_t) g_pcmf32.size(), window_samples), g_pcmf32.end());
            // 清空 g_pcmf32
            g_pcmf32.clear();
        }

        {
            // 记录当前时间
            const auto t_start = std::chrono::high_resolution_clock::now();

            // 设置流状态为运行 whisper
            stream_set_status("running whisper ...");

            // 调用 whisper_full 函数进行 whisper 处理
            int ret = whisper_full(ctx, wparams, pcmf32.data(), pcmf32.size());
            // 如果处理失败，打印错误信息并跳出循环
            if (ret != 0) {
                printf("whisper_full() failed: %d\n", ret);
                break;
            }

            // 记录处理结束时间
            const auto t_end = std::chrono::high_resolution_clock::now();

            // 打印 whisper_full 函数返回值和处理时间
            printf("stream: whisper_full() returned %d in %f seconds\n", ret, std::chrono::duration<double>(t_end - t_start).count());
        }

        {
            // 存储听到的文本
            std::string text_heard;

            {
                // 获取 whisper 处理后的段落数
                const int n_segments = whisper_full_n_segments(ctx);
                // 遍历每个段落
                for (int i = n_segments - 1; i < n_segments; ++i) {
                    // 获取段落文本
                    const char * text = whisper_full_get_segment_text(ctx, i);

                    // 获取段落起始和结束时间
                    const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
                    const int64_t t1 = whisper_full_get_segment_t1(ctx, i);

                    // 打印转录的文本
                    printf("transcribed: %s\n", text);

                    // 将文本添加到 text_heard 中
                    text_heard += text;
                }
            }

            {
                // 创建互斥锁
                std::lock_guard<std::mutex> lock(g_mutex);
                // 将听到的文本存储到全局变量 g_transcribed 中
                g_transcribed = text_heard;
            }
        }
    }

    // 如果 index 小于 g_contexts 的大小
    if (index < g_contexts.size()) {
        // 释放 g_contexts[index] 的内存
        whisper_free(g_contexts[index]);
        // 将 g_contexts[index] 置为 nullptr
        g_contexts[index] = nullptr;
    }
// 定义 EMSCRIPTEN_BINDINGS，将 C++ 函数绑定到 JavaScript
EMSCRIPTEN_BINDINGS(stream) {
    // 定义 init 函数，初始化模型
    emscripten::function("init", emscripten::optional_override([](const std::string & path_model) {
        // 遍历上下文数组
        for (size_t i = 0; i < g_contexts.size(); ++i) {
            // 如果当前上下文为空
            if (g_contexts[i] == nullptr) {
                // 从文件初始化上下文
                g_contexts[i] = whisper_init_from_file_with_params(path_model.c_str(), whisper_context_default_params());
                // 如果初始化成功
                if (g_contexts[i] != nullptr) {
                    // 设置运行状态为 true
                    g_running = true;
                    // 如果工作线程可加入
                    if (g_worker.joinable()) {
                        g_worker.join();
                    }
                    // 创建新的工作线程
                    g_worker = std::thread([i]() {
                        stream_main(i);
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

    // 定义 free 函数，释放资源
    emscripten::function("free", emscripten::optional_override([](size_t index) {
        // 如果正在运行，设置运行状态为 false
        if (g_running) {
            g_running = false;
        }
    }));

    // 定义 set_audio 函数，设置音频数据
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
            // 加锁，确保线程安全
            std::lock_guard<std::mutex> lock(g_mutex);
            // 获取音频数据长度
            const int n = audio["length"].as<int>();

            // 获取内存堆和内存
            emscripten::val heap = emscripten::val::module_property("HEAPU8");
            emscripten::val memory = heap["buffer"];

            // 调整 PCM 数据大小
            g_pcmf32.resize(n);

            // 创建内存视图，设置音频数据
            emscripten::val memoryView = audio["constructor"].new_(memory, reinterpret_cast<uintptr_t>(g_pcmf32.data()), n);
            memoryView.call<void>("set", audio);
        }

        // 返回成功
        return 0;
    }));
}
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