# `whisper.cpp\examples\bench.wasm\emscripten.cpp`

```cpp
// 包含头文件 "whisper.h"
#include "whisper.h"

// 引入 Emscripten 相关头文件
#include <emscripten.h>
#include <emscripten/bind.h>

// 引入 C++ 标准库头文件
#include <cmath>
#include <string>
#include <thread>
#include <vector>

// 定义常量 N_THREAD 为 8
constexpr int N_THREAD = 8;

// TODO: get rid of this vector of contexts - bad idea in the first place
// 创建一个包含 4 个空指针的 whisper_context 结构体指针向量
std::vector<struct whisper_context *> g_contexts(4, nullptr);

// 声明全局线程对象 g_worker
std::thread g_worker;

// 定义函数 bench_main，参数为索引 index
void bench_main(size_t index) {
    // 获取可用线程数，取 N_THREAD 和硬件并发线程数的最小值
    const int n_threads = std::min(N_THREAD, (int) std::thread::hardware_concurrency());

    // 获取当前索引对应的 whisper context
    auto & ctx = g_contexts[index];

    // 打印提示信息
    fprintf(stderr, "%s: running benchmark with %d threads - please wait...\n", __func__, n_threads);

    // 获取 whisper context 中的 mel 数量
    const int n_mels = whisper_model_n_mels(ctx);

    // 设置 mel 参数
    if (int ret = whisper_set_mel(ctx, nullptr, 0, n_mels)) {
        fprintf(stderr, "error: failed to set mel: %d\n", ret);
        return;
    }

    {
        // 打印系统信息
        fprintf(stderr, "\n");
        fprintf(stderr, "system_info: n_threads = %d / %d | %s\n", n_threads, std::thread::hardware_concurrency(), whisper_print_system_info());
    }

    // 对模型进行编码
    if (int ret = whisper_encode(ctx, 0, n_threads) != 0) {
        fprintf(stderr, "error: failed to encode model: %d\n", ret);
        return;
    }

    // 打印时间信息
    whisper_print_timings(ctx);

    // 打印提交结果的提示信息
    fprintf(stderr, "\n");
    fprintf(stderr, "If you wish, you can submit these results here:\n");
    fprintf(stderr, "\n");
    fprintf(stderr, "  https://github.com/ggerganov/whisper.cpp/issues/89\n");
    fprintf(stderr, "\n");
    fprintf(stderr, "Please include the following information:\n");
    fprintf(stderr, "\n");
    fprintf(stderr, "  - CPU model\n");
    fprintf(stderr, "  - Operating system\n");
    fprintf(stderr, "  - Browser\n");
    fprintf(stderr, "\n");
}

// 绑定 bench_main 函数到 JavaScript
EMSCRIPTEN_BINDINGS(bench) {
    // 定义名为 "init" 的 JavaScript 函数，接受一个参数 path_model
    emscripten::function("init", emscripten::optional_override([](const std::string & path_model) {
        // 遍历全局变量 g_contexts 数组
        for (size_t i = 0; i < g_contexts.size(); ++i) {
            // 检查当前位置是否为空
            if (g_contexts[i] == nullptr) {
                // 从文件初始化 whisper 上下文，并使用默认参数
                g_contexts[i] = whisper_init_from_file_with_params(path_model.c_str(), whisper_context_default_params());
                // 如果初始化成功
                if (g_contexts[i] != nullptr) {
                    // 如果工作线程可加入
                    if (g_worker.joinable()) {
                        // 加入工作线程
                        g_worker.join();
                    }
                    // 创建新的线程，调用 bench_main 函数
                    g_worker = std::thread([i]() {
                        bench_main(i);
                    });
                    // 返回当前位置加 1
                    return i + 1;
                } else {
                    // 如果初始化失败，返回 0
                    return (size_t) 0;
                }
            }
        }
        // 如果所有位置都被占用，返回 0
        return (size_t) 0;
    }));

    // 定义名为 "free" 的 JavaScript 函数，接受一个参数 index
    emscripten::function("free", emscripten::optional_override([](size_t index) {
        // 检查 index 是否在 g_contexts 数组的范围内
        if (index < g_contexts.size()) {
            // 释放 g_contexts 数组中指定位置的 whisper 上下文
            whisper_free(g_contexts[index]);
            // 将指定位置的上下文设为 nullptr
            g_contexts[index] = nullptr;
        }
    }));
# 闭合之前的代码块
```