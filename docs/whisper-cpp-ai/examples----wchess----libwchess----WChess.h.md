# `whisper.cpp\examples\wchess\libwchess\WChess.h`

```cpp
#pragma once
#include "whisper.h"
#include <string>
#include <vector>
#include <memory>

// 声明 Chessboard 类
class Chessboard;

// 声明 WChess 类
class WChess {
public:
    // 定义回调函数类型
    using CheckRunningCb = bool (*)();
    using GetAudioCb = bool (*)(std::vector<float> &);
    using SetMovesCb = void (*)(const std::string &, float);
    using SetGrammarCb = void (*)(const std::string &);
    using ClearAudioCb = void (*)();

    // 定义回调函数结构体
    struct callbacks {
        GetAudioCb get_audio = nullptr;
        SetMovesCb set_move = nullptr;
        SetGrammarCb set_grammar = nullptr;
    };

    // 定义设置结构体
    struct settings {
        int32_t vad_ms     = 2000;
        int32_t prompt_ms  = 5000;
        int32_t command_ms = 4000;
        float vad_thold    = 0.2f;
        float freq_thold   = 100.0f;
        bool print_energy  = false;
    };

    // 构造函数
    WChess(
        whisper_context * ctx,
        const whisper_full_params & wparams,
        callbacks cb,
        settings s
    );
    // 析构函数
    ~WChess();

    // 运行函数
    void run();

    // 返回棋盘状态的字符串表示
    std::string stringify_board() const;

    // 返回语法规则字符串
    std::string get_grammar() const;

private:
    // 获取音频数据
    bool get_audio(std::vector<float>& pcmf32) const;
    // 设置走棋
    void set_move(const std::string& moves, float prob) const;
    // 设置语法规则
    void set_grammar(const std::string& grammar) const;

    // 转录音频数据
    std::string transcribe(
                    const std::vector<float> & pcmf32,
                    float & logprob_min,
                    float & logprob_sum,
                    int & n_tokens,
                    int64_t & t_ms);

    whisper_context * m_ctx; // Whisper 上下文
    whisper_full_params m_wparams; // Whisper 参数
    const callbacks m_cb; // 回调函数
    const settings m_settings; // 设置
    std::unique_ptr<Chessboard> m_board; // 棋盘对象
};
```