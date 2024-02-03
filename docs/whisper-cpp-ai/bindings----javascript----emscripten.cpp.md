# `whisper.cpp\bindings\javascript\emscripten.cpp`

```cpp
// 这是 whisper.cpp 的 Javascript API
// 目前非常简陋
// 欢迎贡献并改进这个API
// 请查看 tests/test-whisper.js 获取示例用法

#include "whisper.h"

#include <emscripten.h>
#include <emscripten/bind.h>

#include <thread>
#include <vector>

// 定义全局的 whisper_context 结构体指针
struct whisper_context * g_context;

// 使用 EMSCRIPTEN_BINDINGS 定义 whisper 模块
EMSCRIPTEN_BINDINGS(whisper) {
    // 定义 init 函数，初始化 whisper 上下文
    emscripten::function("init", emscripten::optional_override([](const std::string & path_model) {
        // 如果上下文为空
        if (g_context == nullptr) {
            // 从文件初始化 whisper 上下文，并使用默认参数
            g_context = whisper_init_from_file_with_params(path_model.c_str(), whisper_context_default_params());
            // 如果上下文不为空，返回 true，否则返回 false
            if (g_context != nullptr) {
                return true;
            } else {
                return false;
            }
        }

        return false;
    }));

    // 定义 free 函数，释放 whisper 上下文
    emscripten::function("free", emscripten::optional_override([]() {
        // 如果上下文存在，释放上下文并将指针置为空
        if (g_context) {
            whisper_free(g_context);
            g_context = nullptr;
        }
    }));

    // 结束 EMSCRIPTEN_BINDINGS 定义
}
```