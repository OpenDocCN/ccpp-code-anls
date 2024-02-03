# `whisper.cpp\examples\stream\stream.cpp`

```cpp
// 实时语音识别，从麦克风输入
//
// 一个非常快速且简陋的实现，主要作为概念验证。
//
#include "common-sdl.h"
#include "common.h"
#include "whisper.h"

#include <cassert>
#include <cstdio>
#include <string>
#include <thread>
#include <vector>
#include <fstream>


//  500 -> 00:05.000
// 6000 -> 01:00.000
// 将时间戳转换为格式化字符串，格式为 mm:ss.SSS
std::string to_timestamp(int64_t t) {
    // 计算秒数和毫秒数
    int64_t sec = t/100;
    int64_t msec = t - sec*100;
    // 计算分钟数
    int64_t min = sec/60;
    sec = sec - min*60;

    char buf[32];
    // 格式化输出时间戳字符串
    snprintf(buf, sizeof(buf), "%02d:%02d.%03d", (int) min, (int) sec, (int) msec);

    return std::string(buf);
}

// 命令行参数
struct whisper_params {
    int32_t n_threads  = std::min(4, (int32_t) std::thread::hardware_concurrency());
    int32_t step_ms    = 3000;
    int32_t length_ms  = 10000;
    int32_t keep_ms    = 200;
    int32_t capture_id = -1;
    int32_t max_tokens = 32;
    int32_t audio_ctx  = 0;

    float vad_thold    = 0.6f;
    float freq_thold   = 100.0f;

    bool speed_up      = false;
    bool translate     = false;
    bool no_fallback   = false;
    bool print_special = false;
    bool no_context    = true;
    bool no_timestamps = false;
    bool tinydiarize   = false;
    bool save_audio    = false; // save audio to wav file
    bool use_gpu       = true;

    std::string language  = "en";
    std::string model     = "models/ggml-base.en.bin";
    std::string fname_out;
};

// 打印使用说明
void whisper_print_usage(int argc, char ** argv, const whisper_params & params);

// 解析命令行参数
bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    // 解析命令行参数的逻辑
    // ...
    return true;
}

// 打印使用说明
void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params) {
    // 打印使用说明
    fprintf(stderr, "\n");
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h,       --help          [default] show this help message and exit\n");
}
    # 打印线程数选项及当前设置的线程数
    fprintf(stderr, "  -t N,     --threads N     [%-7d] number of threads to use during computation\n",    params.n_threads);
    # 打印音频步长选项及当前设置的步长
    fprintf(stderr, "            --step N        [%-7d] audio step size in milliseconds\n",                params.step_ms);
    # 打印音频长度选项及当前设置的长度
    fprintf(stderr, "            --length N      [%-7d] audio length in milliseconds\n",                   params.length_ms);
    # 打印保留音频选项及当前设置的保留时间
    fprintf(stderr, "            --keep N        [%-7d] audio to keep from previous step in ms\n",         params.keep_ms);
    # 打印捕获设备ID选项及当前设置的ID
    fprintf(stderr, "  -c ID,    --capture ID    [%-7d] capture device ID\n",                              params.capture_id);
    # 打印最大令牌数选项及当前设置的最大令牌数
    fprintf(stderr, "  -mt N,    --max-tokens N  [%-7d] maximum number of tokens per audio chunk\n",       params.max_tokens);
    # 打印音频上下文大小选项及当前设置的大小
    fprintf(stderr, "  -ac N,    --audio-ctx N   [%-7d] audio context size (0 - all)\n",                   params.audio_ctx);
    # 打印语音活动检测阈值选项及当前设置的阈值
    fprintf(stderr, "  -vth N,   --vad-thold N   [%-7.2f] voice activity detection threshold\n",           params.vad_thold);
    # 打印高通滤波频率截止选项及当前设置的频率
    fprintf(stderr, "  -fth N,   --freq-thold N  [%-7.2f] high-pass frequency cutoff\n",                   params.freq_thold);
    # 打印加速音频选项及当前设置是否加速
    fprintf(stderr, "  -su,      --speed-up      [%-7s] speed up audio by x2 (reduced accuracy)\n",        params.speed_up ? "true" : "false");
    # 打印翻译选项及当前设置是否翻译
    fprintf(stderr, "  -tr,      --translate     [%-7s] translate from source language to english\n",      params.translate ? "true" : "false");
    # 打印不使用温度回退解码选项及当前设置是否不使用
    fprintf(stderr, "  -nf,      --no-fallback   [%-7s] do not use temperature fallback while decoding\n", params.no_fallback ? "true" : "false");
    # 打印打印特殊令牌选项及当前设置是否打印
    fprintf(stderr, "  -ps,      --print-special [%-7s] print special tokens\n",                           params.print_special ? "true" : "false");
    # 打印保持上下文选项及当前设置是否不保持上下文
    fprintf(stderr, "  -kc,      --keep-context  [%-7s] keep context between audio chunks\n",              params.no_context ? "false" : "true");
    # 打印语言选项及当前设置的语言
    fprintf(stderr, "  -l LANG,  --language LANG [%-7s] spoken language\n",                                params.language.c_str());
    # 打印模型路径参数信息
    fprintf(stderr, "  -m FNAME, --model FNAME   [%-7s] model path\n", params.model.c_str());
    # 打印文本输出文件名参数信息
    fprintf(stderr, "  -f FNAME, --file FNAME    [%-7s] text output file name\n", params.fname_out.c_str());
    # 打印是否启用 tinydiarize 参数信息
    fprintf(stderr, "  -tdrz,    --tinydiarize   [%-7s] enable tinydiarize (requires a tdrz model)\n", params.tinydiarize ? "true" : "false");
    # 打印是否保存录制音频到文件参数信息
    fprintf(stderr, "  -sa,      --save-audio    [%-7s] save the recorded audio to a file\n", params.save_audio ? "true" : "false");
    # 打印是否禁用 GPU 推断参数信息
    fprintf(stderr, "  -ng,      --no-gpu        [%-7s] disable GPU inference\n", params.use_gpu ? "false" : "true");
    # 打印空行
    fprintf(stderr, "\n");
}

// 主函数，接受命令行参数并进行处理
int main(int argc, char ** argv) {
    // 定义参数结构体
    whisper_params params;

    // 解析命令行参数，如果解析失败则返回错误
    if (whisper_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 调整参数中的时间间隔和长度，确保时间间隔不大于长度
    params.keep_ms   = std::min(params.keep_ms,   params.step_ms);
    params.length_ms = std::max(params.length_ms, params.step_ms);

    // 计算采样点数
    const int n_samples_step = (1e-3*params.step_ms  )*WHISPER_SAMPLE_RATE;
    const int n_samples_len  = (1e-3*params.length_ms)*WHISPER_SAMPLE_RATE;
    const int n_samples_keep = (1e-3*params.keep_ms  )*WHISPER_SAMPLE_RATE;
    const int n_samples_30s  = (1e-3*30000.0         )*WHISPER_SAMPLE_RATE;

    // 判断是否使用 VAD（语音活动检测）
    const bool use_vad = n_samples_step <= 0; // sliding window mode uses VAD

    // 计算打印新行的步数
    const int n_new_line = !use_vad ? std::max(1, params.length_ms / params.step_ms - 1) : 1; // number of steps to print new line

    // 根据使用 VAD 的情况设置参数
    params.no_timestamps  = !use_vad;
    params.no_context    |= use_vad;
    params.max_tokens     = 0;

    // 初始化音频
    audio_async audio(params.length_ms);
    // 如果音频初始化失败，则打印错误信息并返回
    if (!audio.init(params.capture_id, WHISPER_SAMPLE_RATE)) {
        fprintf(stderr, "%s: audio.init() failed!\n", __func__);
        return 1;
    }

    // 恢复音频
    audio.resume();

    // 初始化 whisper
    if (params.language != "auto" && whisper_lang_id(params.language.c_str()) == -1){
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // 初始化 whisper 上下文
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;

    struct whisper_context * ctx = whisper_init_from_file_with_params(params.model.c_str(), cparams);

    // 初始化音频数据向量
    std::vector<float> pcmf32    (n_samples_30s, 0.0f);
    std::vector<float> pcmf32_old;
    std::vector<float> pcmf32_new(n_samples_30s, 0.0f);

    std::vector<whisper_token> prompt_tokens;

    // 打印一些关于处理的信息
    {
        // 输出一个换行符
        fprintf(stderr, "\n");
        // 检查是否为多语言模型
        if (!whisper_is_multilingual(ctx)) {
            // 如果不是多语言模型，且语言不是英语或需要翻译
            if (params.language != "en" || params.translate) {
                // 将语言设置为英语，翻译选项设置为 false
                params.language = "en";
                params.translate = false;
                // 输出警告信息
                fprintf(stderr, "%s: WARNING: model is not multilingual, ignoring language and translation options\n", __func__);
            }
        }
        // 输出处理样本信息
        fprintf(stderr, "%s: processing %d samples (step = %.1f sec / len = %.1f sec / keep = %.1f sec), %d threads, lang = %s, task = %s, timestamps = %d ...\n",
                __func__,
                n_samples_step,
                float(n_samples_step)/WHISPER_SAMPLE_RATE,
                float(n_samples_len )/WHISPER_SAMPLE_RATE,
                float(n_samples_keep)/WHISPER_SAMPLE_RATE,
                params.n_threads,
                params.language.c_str(),
                params.translate ? "translate" : "transcribe",
                params.no_timestamps ? 0 : 1);

        // 如果不使用 VAD
        if (!use_vad) {
            // 输出 n_new_line 和 no_context 信息
            fprintf(stderr, "%s: n_new_line = %d, no_context = %d\n", __func__, n_new_line, params.no_context);
        } else {
            // 输出使用 VAD 的信息
            fprintf(stderr, "%s: using VAD, will transcribe on speech activity\n", __func__);
        }

        // 输出一个换行符
        fprintf(stderr, "\n");
    }

    // 初始化迭代次数
    int n_iter = 0;

    // 设置程序运行状态为 true
    bool is_running = true;

    // 打开输出文件
    std::ofstream fout;
    if (params.fname_out.length() > 0) {
        fout.open(params.fname_out);
        // 如果无法打开输出文件，输出错误信息并返回
        if (!fout.is_open()) {
            fprintf(stderr, "%s: failed to open output file '%s'!\n", __func__, params.fname_out.c_str());
            return 1;
        }
    }

    // 初始化 WAV 文件写入器
    wav_writer wavWriter;
    // 保存 WAV 文件
    if (params.save_audio) {
        // 获取当前日期时间作为文件名
        time_t now = time(0);
        char buffer[80];
        strftime(buffer, sizeof(buffer), "%Y%m%d%H%M%S", localtime(&now));
        std::string filename = std::string(buffer) + ".wav";

        // 打开 WAV 文件
        wavWriter.open(filename, WHISPER_SAMPLE_RATE, 16, 1);
    }
    # 打印开始说话的提示信息
    printf("[Start speaking]\n");
    # 刷新标准输出流
    fflush(stdout);

    # 获取当前时间作为起始时间点
    auto t_last  = std::chrono::high_resolution_clock::now();
    const auto t_start = t_last;

    # 主音频循环
    }

    # 暂停音频播放

    # 打印时间信息
    whisper_print_timings(ctx);
    # 释放上下文资源
    whisper_free(ctx);

    # 返回0表示正常结束
    return 0;
# 闭合之前的代码块
```