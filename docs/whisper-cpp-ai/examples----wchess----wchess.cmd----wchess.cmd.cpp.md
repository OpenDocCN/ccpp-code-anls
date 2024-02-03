# `whisper.cpp\examples\wchess\wchess.cmd\wchess.cmd.cpp`

```cpp
// 命令行语音辅助国际象棋
//
// 通过麦克风说出国际象棋移动命令。
// 移动将被转换为棋盘位置。
//
//

#include "WChess.h"
#include "common-sdl.h"
#include <iostream>

#include <memory>
#include <thread>

// 命令行参数
struct whisper_params {
    int32_t n_threads  = std::min(4, (int32_t) std::thread::hardware_concurrency());
    int32_t prompt_ms  = 5000;
    int32_t command_ms = 8000;
    int32_t capture_id = -1;
    int32_t max_tokens = 32;
    int32_t audio_ctx  = 0;

    float vad_thold  = 0.6f;
    float freq_thold = 100.0f;

    float grammar_penalty = 100.0f;

    bool speed_up      = false;
    bool translate     = false;
    bool print_special = false;
    bool print_energy  = false;
    bool no_timestamps = true;
    bool use_gpu       = true;

    std::string language  = "en";
    std::string model     = "models/ggml-base.en.bin";
    std::string fname_out;
    std::string commands;
    std::string prompt;
    std::string context;
    std::string grammar;
};

// 打印使用说明
void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params) {
    fprintf(stderr, "\n");
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h,         --help           [default] show this help message and exit\n");
    fprintf(stderr, "  -t N,       --threads N      [%-7d] number of threads to use during computation\n", params.n_threads);
    fprintf(stderr, "  -pms N,     --prompt-ms N    [%-7d] prompt duration in milliseconds\n",             params.prompt_ms);
    fprintf(stderr, "  -cms N,     --command-ms N   [%-7d] command duration in milliseconds\n",            params.command_ms);
    fprintf(stderr, "  -c ID,      --capture ID     [%-7d] capture device ID\n",                           params.capture_id);
    # 打印最大令牌数的参数说明和当前值
    fprintf(stderr, "  -mt N,      --max-tokens N   [%-7d] maximum number of tokens per audio chunk\n",    params.max_tokens);
    # 打印音频上下文大小的参数说明和当前值
    fprintf(stderr, "  -ac N,      --audio-ctx N    [%-7d] audio context size (0 - all)\n",                params.audio_ctx);
    # 打印语音活动检测阈值的参数说明和当前值
    fprintf(stderr, "  -vth N,     --vad-thold N    [%-7.2f] voice activity detection threshold\n",        params.vad_thold);
    # 打印高通滤波频率截止值的参数说明和当前值
    fprintf(stderr, "  -fth N,     --freq-thold N   [%-7.2f] high-pass frequency cutoff\n",                params.freq_thold);
    # 打印是否加速音频的参数说明和当前值
    fprintf(stderr, "  -su,        --speed-up       [%-7s] speed up audio by x2 (reduced accuracy)\n",     params.speed_up ? "true" : "false");
    # 打印是否翻译的参数说明和当前值
    fprintf(stderr, "  -tr,        --translate      [%-7s] translate from source language to english\n",   params.translate ? "true" : "false");
    # 打印是否打印特殊标记的参数说明和当前值
    fprintf(stderr, "  -ps,        --print-special  [%-7s] print special tokens\n",                        params.print_special ? "true" : "false");
    # 打印是否打印声音能量的参数说明和当前值
    fprintf(stderr, "  -pe,        --print-energy   [%-7s] print sound energy (for debugging)\n",          params.print_energy ? "true" : "false");
    # 打印是否禁用 GPU 的参数说明和当前值
    fprintf(stderr, "  -ng,        --no-gpu         [%-7s] disable GPU\n",                                 params.use_gpu ? "false" : "true");
    # 打印口语语言的参数说明和当前值
    fprintf(stderr, "  -l LANG,    --language LANG  [%-7s] spoken language\n",                             params.language.c_str());
    # 打印模型路径的参数说明和当前值
    fprintf(stderr, "  -m FNAME,   --model FNAME    [%-7s] model path\n",                                  params.model.c_str());
    # 打印文本输出文件名的参数说明和当前值
    fprintf(stderr, "  -f FNAME,   --file FNAME     [%-7s] text output file name\n",                       params.fname_out.c_str());
    # 打印允许命令的文本文件名的参数说明和当前值
    fprintf(stderr, "  -cmd FNAME, --commands FNAME [%-7s] text file with allowed commands\n",             params.commands.c_str());
    # 打印所需激活提示的参数说明和当前值
    fprintf(stderr, "  -p,         --prompt         [%-7s] the required activation prompt\n",              params.prompt.c_str());
    # 输出带有参数值的文本到标准错误流，用于帮助转录
    fprintf(stderr, "  -ctx,       --context        [%-7s] sample text to help the transcription\n",       params.context.c_str());
    # 输出带有参数值的文本到标准错误流，用于缩小非语法标记的对数值
    fprintf(stderr, "  --grammar-penalty N          [%-7.1f] scales down logits of nongrammar tokens\n",   params.grammar_penalty);
    # 输出换行符到标准错误流
    fprintf(stderr, "\n");
}

// 解析命令行参数，填充参数结构体
bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    // 待实现
    return true;
}

// 全局变量，存储棋盘状态
std::unique_ptr<WChess> g_wchess;
// 全局变量，记录走棋步数
int g_moveCount = 0;
// 设置走棋步骤
void set_move(const std::string & move, float) {
    // 如果走棋步骤非空
    if (!move.empty()) {
        // 增加走棋步数
        g_moveCount++;
        // 打印走棋步骤
        fprintf(stdout, "Move: %s\n\n", move.c_str());
    }
    else 
        // 打印走棋步骤被拒绝
        fprintf(stdout, "Move rejected\n\n");
    // 打印棋盘状态
    fprintf(stdout, "%s\n", g_wchess->stringify_board().c_str());
    // 打印轮到哪方走棋
    fprintf(stdout, "%s\n", g_moveCount ? "White's turn" : "Black's turn");
}

// 全局变量，异步音频对象
audio_async g_audio(30*1000);
// 全局变量，标识是否正在监听
bool g_listening = false;
// 全局变量，存储音频数据
std::vector<float> g_pcmf32;

// 读取输入
bool read_input() {
    std::string input;
    while (true) {
        // 提示用户输入指令
        fprintf(stdout, "[(l)isten/(p)ause/(q)uit]: ");
        std::cin >> input;
        fprintf(stdout, "\n");
        // 如果用户输入退出指令
        if (input[0] == 'q') {
            fprintf(stdout, "Quitting\n");
            return false;
        }
        // 如果用户输入监听指令
        if (input[0] == 'l') {
            // 如果当前未在监听
            if (!g_listening) {
                fprintf(stdout, "Listening\n");
                g_listening = true;
                g_pcmf32.clear();
                g_audio.resume();
                g_audio.clear();
            }
            else 
                // 如果当前正在监听
                fprintf(stdout, "Still listening\n");
            return true;
        }
        else {
            // 如果当前正在监听
            if (g_listening) {
                g_listening = false;
                g_audio.get(0, g_pcmf32);
                g_audio.pause();
                fprintf(stdout, "Processing\n");
            }
            else 
                // 如果当前未在监听
                fprintf(stdout, "Not listening\n");
            return true;
        }
    }
    return true;
}

// 获取音频数据
bool get_audio(std::vector<float> & pcmf32_cur) {
    // 如果读取输入失败，则返回false
    if (!read_input()) return false;
    // 如果音频数据非空，则移动数据到传入参数中
    if (!g_pcmf32.empty()) pcmf32_cur = std::move(g_pcmf32);
    else pcmf32_cur.clear();
    return true;
}

// 主函数
int main(int argc, char ** argv) {
    // 初始化参数结构体
    whisper_params params;

    // 如果解析参数失败，则返回1
    if (whisper_params_parse(argc, argv, params) == false) {
        return 1;
    }
    // 检查语言是否合法，如果不合法则输出错误信息并退出程序
    if (whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // 初始化 whisper

    // 设置 whisper 上下文参数
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;

    // 从文件中初始化 whisper 上下文
    struct whisper_context * ctx = whisper_init_from_file_with_params(params.model.c_str(), cparams);
    if (!ctx) {
        fprintf(stderr, "%s: whisper_init_from_file_with_params() failed!\n", __func__);
        return 1;
    }

    // 初始化音频

    // 初始化音频捕获
    if (!g_audio.init(params.capture_id, WHISPER_SAMPLE_RATE)) {
        fprintf(stderr, "%s: audio.init() failed!\n", __func__);
        return 1;
    }

    // 设置 whisper 完整参数
    struct whisper_full_params wparams = whisper_full_default_params(whisper_sampling_strategy::WHISPER_SAMPLING_GREEDY);
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
    wparams.audio_ctx        = 768; // 为了提高性能而设置的部分编码器上下文

    wparams.temperature     = 0.0f;
    wparams.temperature_inc = 2.0f;
    wparams.greedy.best_of  = 1;

    wparams.beam_search.beam_size = 1;

    wparams.language         = "en";

    wparams.grammar_penalty = 100.0;

    wparams.initial_prompt = params.context.data();

    // 设置 WChess 回调函数
    WChess::callbacks cb;
    cb.get_audio = get_audio;
    cb.set_move = set_move;

    // 设置 WChess 参数
    WChess::settings s;
    s.vad_ms = 2000;
    s.prompt_ms = params.prompt_ms;
    s.command_ms = params.command_ms;
    s.vad_thold = params.vad_thold;
    s.freq_thold = params.freq_thold;
    s.print_energy = params.print_energy;

    // 重置 WChess 对象并运行
    g_wchess.reset(new WChess(ctx, wparams, cb, s));
    set_move("start", 0);
    g_wchess->run();
    # 打印程序运行时间信息
    whisper_print_timings(ctx);
    # 释放上下文资源
    whisper_free(ctx);

    # 返回成功状态码
    return 0;
# 闭合之前的代码块
```