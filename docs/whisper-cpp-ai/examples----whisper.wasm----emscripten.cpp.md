# `whisper.cpp\examples\whisper.wasm\emscripten.cpp`

```cpp
#include "whisper.h"

#include <emscripten.h>
#include <emscripten/bind.h>

#include <vector>
#include <thread>

// 全局变量，用于存储工作线程
std::thread g_worker;

// 全局变量，用于存储 whisper_context 指针的向量，初始大小为 4，初始值为 nullptr
std::vector<struct whisper_context *> g_contexts(4, nullptr);

// 计算大于等于 n 的最小的 2 的幂
static inline int mpow2(int n) {
    int p = 1;
    while (p <= n) p *= 2;
    return p/2;
}

// 绑定 JavaScript 函数到 C++ 函数
EMSCRIPTEN_BINDINGS(whisper) {
    // 初始化函数，可选重写
    emscripten::function("init", emscripten::optional_override([](const std::string & path_model) {
        // 如果工作线程正在运行，则等待其结束
        if (g_worker.joinable()) {
            g_worker.join();
        }

        // 遍历全局 contexts 向量
        for (size_t i = 0; i < g_contexts.size(); ++i) {
            // 如果当前位置的指针为空
            if (g_contexts[i] == nullptr) {
                // 从文件初始化 whisper_context，并使用默认参数
                g_contexts[i] = whisper_init_from_file_with_params(path_model.c_str(), whisper_context_default_params());
                // 如果初始化成功，则返回当前位置索引加一
                if (g_contexts[i] != nullptr) {
                    return i + 1;
                } else {
                    return (size_t) 0;
                }
            }
        }

        return (size_t) 0;
    }));

    // 释放函数，可选重写
    emscripten::function("free", emscripten::optional_override([](size_t index) {
        // 如果工作线程正在运行，则等待其结束
        if (g_worker.joinable()) {
            g_worker.join();
        }

        // 索引减一
        --index;

        // 如果索引在 contexts 向量范围内
        if (index < g_contexts.size()) {
            // 释放对应位置的 whisper_context 指针，并将其置为空指针
            whisper_free(g_contexts[index]);
            g_contexts[index] = nullptr;
        }
    }));

    }));
}
```