# `whisper.cpp\examples\talk\talk.cpp`

```cpp
// 与 AI 进行对话

#include "common-sdl.h"
#include "common.h"
#include "whisper.h"
#include "gpt-2.h"

#include <cassert>
#include <cstdio>
#include <fstream>
#include <regex>
#include <string>
#include <thread>
#include <vector>
#include <regex>

// 命令行参数
struct whisper_params {
    int32_t n_threads  = std::min(4, (int32_t) std::thread::hardware_concurrency()); // 确定要使用的线程数，最小为4和硬件并发数的较小值
    int32_t voice_ms   = 10000; // 声音持续时间（毫秒）
    int32_t capture_id = -1; // 捕获设备 ID
    int32_t max_tokens = 32; // 最大 token 数
    int32_t audio_ctx  = 0; // 音频上下文

    float vad_thold    = 0.6f; // VAD 阈值
    float freq_thold   = 100.0f; // 频率阈值

    bool speed_up      = false; // 是否加速
    bool translate     = false; // 是否翻译
    bool print_special = false; // 是否打印特殊字符
    bool print_energy  = false; // 是否打印能量
    bool no_timestamps = true; // 是否不显示时间戳
    bool use_gpu       = true; // 是否使用 GPU

    std::string person    = "Santa"; // 人物
    std::string language  = "en"; // 语言
    std::string model_wsp = "models/ggml-base.en.bin"; // WSP 模型
    std::string model_gpt = "models/ggml-gpt-2-117M.bin"; // GPT 模型
    std::string speak     = "./examples/talk/speak"; // 说话
    std::string fname_out; // 输出文件名
};

void whisper_print_usage(int argc, char ** argv, const whisper_params & params);

bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    // 参数解析函数，暂时为空，需要实现
    }

    return true;
}

void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params) {
    // 打印使用说明
    fprintf(stderr, "\n");
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h,       --help          [default] show this help message and exit\n");
    fprintf(stderr, "  -t N,     --threads N     [%-7d] number of threads to use during computation\n", params.n_threads);
    fprintf(stderr, "  -vms N,   --voice-ms N    [%-7d] voice duration in milliseconds\n",              params.voice_ms);
    fprintf(stderr, "  -c ID,    --capture ID    [%-7d] capture device ID\n",                           params.capture_id);
    # 打印最大令牌数的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -mt N,    --max-tokens N  [%-7d] maximum number of tokens per audio chunk\n",    params.max_tokens);
    # 打印音频上下文大小的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -ac N,    --audio-ctx N   [%-7d] audio context size (0 - all)\n",                params.audio_ctx);
    # 打印语音活动检测阈值的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -vth N,   --vad-thold N   [%-7.2f] voice activity detection threshold\n",        params.vad_thold);
    # 打印高通滤波频率截止值的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -fth N,   --freq-thold N  [%-7.2f] high-pass frequency cutoff\n",                params.freq_thold);
    # 打印是否加速音频的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -su,      --speed-up      [%-7s] speed up audio by x2 (reduced accuracy)\n",     params.speed_up ? "true" : "false");
    # 打印是否翻译语言的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -tr,      --translate     [%-7s] translate from source language to english\n",   params.translate ? "true" : "false");
    # 打印是否打印特殊令牌的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -ps,      --print-special [%-7s] print special tokens\n",                        params.print_special ? "true" : "false");
    # 打印是否打印声音能量的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -pe,      --print-energy  [%-7s] print sound energy (for debugging)\n",          params.print_energy ? "true" : "false");
    # 打印是否禁用 GPU 的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -ng,      --no-gpu        [%-7s] disable GPU\n",                                 params.use_gpu ? "false" : "true");
    # 打印人名的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -p NAME,  --person NAME   [%-7s] person name (for prompt selection)\n",          params.person.c_str());
    # 打印语言的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -l LANG,  --language LANG [%-7s] spoken language\n",                             params.language.c_str());
    # 打印耳语模型文件的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -mw FILE, --model-whisper [%-7s] whisper model file\n",                          params.model_wsp.c_str());
    # 打印 GPT 模型文件的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -mg FILE, --model-gpt     [%-7s] gpt model file\n",                              params.model_gpt.c_str());
    # 打印 TTS 命令的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -s FILE,  --speak TEXT    [%-7s] command for TTS\n",                             params.speak.c_str());
    # 打印文本输出文件名的提示信息，包括默认值和用户设置值
    fprintf(stderr, "  -f FNAME, --file FNAME    [%-7s] text output file name\n",                       params.fname_out.c_str());
    # 在标准错误流中输出一个换行符
    fprintf(stderr, "\n");
// 定义一个函数，将音频数据转录为文本
std::string transcribe(whisper_context * ctx, const whisper_params & params, const std::vector<float> & pcmf32, float & prob, int64_t & t_ms) {
    // 记录函数开始时间
    const auto t_start = std::chrono::high_resolution_clock::now();

    // 初始化概率和时间
    prob = 0.0f;
    t_ms = 0;

    // 设置转录参数
    whisper_full_params wparams = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);
    wparams.print_progress   = false;
    wparams.print_special    = params.print_special;
    wparams.print_realtime   = false;
    wparams.print_timestamps = !params.no_timestamps;
    wparams.translate        = params.translate;
    wparams.no_context       = true;
    wparams.single_segment   = true;
    wparams.max_tokens       = params.max_tokens;
    wparams.language         = params.language.c_str();
    wparams.n_threads        = params.n_threads;
    wparams.audio_ctx        = params.audio_ctx;
    wparams.speed_up         = params.speed_up;

    // 调用转录函数
    if (whisper_full(ctx, wparams, pcmf32.data(), pcmf32.size()) != 0) {
        return "";
    }

    // 初始化概率和结果字符串
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

    // 返回转录结果
    return result;
}

// 定义一个包含对话模板的字符串常量
const std::string k_prompt =
R"(This is a dialogue between {0} (A) and a person (B). The dialogue so far is:

B: Hello {0}, how are you?
A: I'm fine, thank you.
{1}
Here is how {0} (A) continues the dialogue:

A:)";

// 主函数
int main(int argc, char ** argv) {
    // 初始化转录参数
    whisper_params params;
    // 如果解析 whisper 参数失败，则返回 1
    if (whisper_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果语言 ID 为 -1，则打印错误信息并退出程序
    if (whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // 初始化 whisper 上下文参数
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;

    // 从文件初始化 whisper 上下文
    struct whisper_context * ctx_wsp = whisper_init_from_file_with_params(params.model_wsp.c_str(), cparams);

    // 初始化 gpt 上下文
    struct gpt2_context * ctx_gpt = gpt2_init(params.model_gpt.c_str());

    // 打印一些关于处理的信息
    {
        fprintf(stderr, "\n");
        // 如果不是多语言模型，且语言不是英语或需要翻译，则忽略语言和翻译选项
        if (!whisper_is_multilingual(ctx_wsp)) {
            if (params.language != "en" || params.translate) {
                params.language = "en";
                params.translate = false;
                fprintf(stderr, "%s: WARNING: model is not multilingual, ignoring language and translation options\n", __func__);
            }
        }
        // 打印处理信息
        fprintf(stderr, "%s: processing, %d threads, lang = %s, task = %s, timestamps = %d ...\n",
                __func__,
                params.n_threads,
                params.language.c_str(),
                params.translate ? "translate" : "transcribe",
                params.no_timestamps ? 0 : 1);

        fprintf(stderr, "\n");
    }

    // 初始化音频
    audio_async audio(30*1000);
    // 如果音频初始化失败，则打印错误信息并返回 1
    if (!audio.init(params.capture_id, WHISPER_SAMPLE_RATE)) {
        fprintf(stderr, "%s: audio.init() failed!\n", __func__);
        return 1;
    }

    // 恢复音频
    audio.resume();

    int n_iter = 0;

    bool is_running  = true;
    bool force_speak = false;

    float prob0 = 0.0f;

    std::vector<float> pcmf32_cur;
    std::vector<float> pcmf32_prompt;

    // 设置 gpt2 上下文的提示为 ""
    gpt2_set_prompt(ctx_gpt, "");

    // 随机生成一个声音 ID
    const int voice_id = rand()%6;

    // 打印 gpt-2 的提示信息
    fprintf(stderr, "gpt-2: prompt:\n");
    fprintf(stderr, "========================\n\n");
    // 使用 fprintf 函数将格式化字符串输出到标准错误流，替换字符串中的 "{0}" 为 params.person，并转换为 C 风格字符串输出
    fprintf(stderr, "%s\n", ::replace(k_prompt, "{0}", params.person).c_str());
    // 输出分隔线
    fprintf(stderr, "========================\n\n");

    // 主循环结束的右括号，可能是代码块的结束
    }

    // 暂停音频播放
    audio.pause();

    // 打印 whisper 上下文的时间信息
    whisper_print_timings(ctx_wsp);
    // 释放 whisper 上下文资源
    whisper_free(ctx_wsp);

    // 返回 0 表示程序正常结束
    return 0;
# 闭合之前的代码块
```