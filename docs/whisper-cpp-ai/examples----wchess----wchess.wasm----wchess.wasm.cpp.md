# `whisper.cpp\examples\wchess\wchess.wasm\wchess.wasm.cpp`

```cpp
#include <WChess.h>
#include <emscripten.h>
#include <emscripten/bind.h>

#include <thread>

constexpr int N_THREAD = 8;

std::vector<struct whisper_context *> g_contexts(4, nullptr);

std::mutex  g_mutex;
std::thread g_worker;

std::condition_variable g_cv;

bool g_running(false);
std::vector<float> g_pcmf32;

void set_move(const std::string & move, float prob) {
    // 在主线程中调用 JavaScript 函数 setMove，传入棋步和概率
    MAIN_THREAD_EM_ASM({
        setMove(UTF8ToString($0), $1)
    }, move.c_str(), prob);
}

void set_grammar(const std::string & grammar) {
    // 在主线程中调用 JavaScript 函数 setGrammar，传入语法
    MAIN_THREAD_EM_ASM({
        setGrammar(UTF8ToString($0))
    }, grammar.c_str());
}

bool get_audio(std::vector<float> & audio) {
    // 获取互斥锁，等待条件变量通知，判断是否正在运行或音频数据非空
    std::unique_lock<std::mutex> lock(g_mutex);
    g_cv.wait(lock, [] { return !g_running || !g_pcmf32.empty(); });
    if (!g_running) return false;
    // 将音频数据移动到传入的音频向量中
    audio = std::move(g_pcmf32);
    return true;
}

void wchess_main(size_t i) {
    // 设置 Whisper 参数
    struct whisper_full_params wparams = whisper_full_default_params(whisper_sampling_strategy::WHISPER_SAMPLING_GREEDY);

    // 设置线程数为硬件支持的最小值
    wparams.n_threads        = std::min(N_THREAD, (int) std::thread::hardware_concurrency());
    wparams.offset_ms        = 0;
    wparams.translate        = false;
    wparams.no_context       = true;
    wparams.single_segment   = true;
    wparams.print_realtime   = false;
    wparams.print_progress   = false;
    wparams.print_timestamps = true;
    wparams.print_special    = false;
    wparams.no_timestamps    = true;

    wparams.max_tokens       = 32;
    wparams.audio_ctx        = 1280; // 为了提高性能，使用部分编码器上下文

    wparams.temperature      = 0.0f;
    wparams.temperature_inc  = 2.0f;
    wparams.greedy.best_of   = 1;

    wparams.beam_search.beam_size = 1;

    wparams.language         = "en";

    wparams.grammar_penalty = 100.0;
    wparams.initial_prompt = "bishop to c3, rook to d4, knight to e5, d4 d5, knight to c3, c3, queen to d4, king b1, pawn to a1, bishop to b2, knight to c3,";

    // 打印使用的线程数
    printf("command: using %d threads\n", wparams.n_threads);
    // 创建一个 WChess::callbacks 结构体实例 cb
    WChess::callbacks cb;
    // 将 get_audio 函数赋值给 cb 结构体中的 get_audio 成员
    cb.get_audio = get_audio;
    // 将 set_move 函数赋值给 cb 结构体中的 set_move 成员
    cb.set_move = set_move;
    // 将 set_grammar 函数赋值给 cb 结构体中的 set_grammar 成员
    cb.set_grammar = set_grammar;

    // 创建一个 WChess 对象，传入 g_contexts[i]、wparams、cb 和空的初始化列表，然后运行
    WChess(g_contexts[i], wparams, cb, {}).run();

    // 如果 i 小于 g_contexts 的大小
    if (i < g_contexts.size()) {
        // 释放 g_contexts[i] 的内存
        whisper_free(g_contexts[i]);
        // 将 g_contexts[i] 指向空指针
        g_contexts[i] = nullptr;
    }
}

// 绑定 JavaScript 函数到 C++ 函数
EMSCRIPTEN_BINDINGS(command) {
    // 初始化函数，从文件中加载模型
    emscripten::function("init", emscripten::optional_override([](const std::string & path_model) {
        // 遍历上下文数组
        for (size_t i = 0; i < g_contexts.size(); ++i) {
            // 如果当前上下文为空
            if (g_contexts[i] == nullptr) {
                // 从文件加载模型并初始化上下文
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
                        wchess_main(i);
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

    // 释放函数，释放指定索引的上下文
    emscripten::function("free", emscripten::optional_override([](size_t /* index */) {
        {
            // 加锁
            std::unique_lock<std::mutex> lock(g_mutex);
            // 设置运行标志为 false
            g_running = false;
        }
        // 通知等待的线程
        g_cv.notify_one();
    }));

    // 设置音频函数，设置指定索引的音频数据
    emscripten::function("set_audio", emscripten::optional_override([](size_t index, const emscripten::val & audio) {
        // 索引减一
        --index;

        // 如果索引超出上下文数组大小
        if (index >= g_contexts.size()) {
            return -1;
        }

        // 如果上下文为空
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

            // 重新设置 PCM 数据大小
            g_pcmf32.resize(n);

            // 创建内存视图并设置音频数据
            emscripten::val memoryView = audio["constructor"].new_(memory, reinterpret_cast<uintptr_t>(g_pcmf32.data()), n);
            memoryView.call<void>("set", audio);
        }
        // 通知等待的线程
        g_cv.notify_one();

        return 0;
    }));
}
```