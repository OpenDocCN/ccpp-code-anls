# `ggml\examples\whisper\main.cpp`

```cpp
// 包含自定义头文件 common.h 和 whisper.h
#include "common.h"
#include "whisper.h"

// 包含 C++ 标准库头文件
#include <cmath>
#include <fstream>
#include <cstdio>
#include <string>
#include <thread>
#include <vector>
#include <cstring>

// 如果编译器为 MSC，禁止警告 4244 和 4267，可能会丢失数据
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 终端颜色映射，10 种颜色分组在范围 [0.0, 0.1, ..., 0.9] 中
// 最低为红色，中间为黄色，最高为绿色
const std::vector<std::string> k_colors = {
    "\033[38;5;196m", "\033[38;5;202m", "\033[38;5;208m", "\033[38;5;214m", "\033[38;5;220m",
    "\033[38;5;226m", "\033[38;5;190m", "\033[38;5;154m", "\033[38;5;118m", "\033[38;5;82m",
};

// 将整数时间戳转换为字符串表示，可选择是否使用逗号
// 500 -> 00:05.000
// 6000 -> 01:00.000
std::string to_timestamp(int64_t t, bool comma = false) {
    int64_t msec = t * 10;
    int64_t hr = msec / (1000 * 60 * 60);
    msec = msec - hr * (1000 * 60 * 60);
    int64_t min = msec / (1000 * 60);
    msec = msec - min * (1000 * 60);
    int64_t sec = msec / 1000;
    msec = msec - sec * 1000;

    char buf[32];
    // 格式化时间字符串
    snprintf(buf, sizeof(buf), "%02d:%02d:%02d%s%03d", (int) hr, (int) min, (int) sec, comma ? "," : ".", (int) msec);

    return std::string(buf);
}

// 将时间戳转换为采样点位置
int timestamp_to_sample(int64_t t, int n_samples) {
    return std::max(0, std::min((int) n_samples - 1, (int) ((t*WHISPER_SAMPLE_RATE)/100)));
}

// 辅助函数，用于替换子字符串
void replace_all(std::string & s, const std::string & search, const std::string & replace) {
    for (size_t pos = 0; ; pos += replace.length()) {
        pos = s.find(search, pos);
        if (pos == std::string::npos) break;
        s.erase(pos, search.length());
        s.insert(pos, replace);
    }
}

// 命令行参数结构体
struct whisper_params {
    int32_t n_threads    = std::min(4, (int32_t) std::thread::hardware_concurrency());
    int32_t n_processors =  1;
    int32_t offset_t_ms  =  0;
    int32_t offset_n     =  0;
    int32_t duration_ms  =  0;
    int32_t progress_step =  5;
    int32_t max_context  = -1;
    int32_t max_len      =  0;
    # 获取 WHISPER_SAMPLING_GREEDY 模式下的默认参数中的 best_of 值
    int32_t best_of      = whisper_full_default_params(WHISPER_SAMPLING_GREEDY).greedy.best_of;
    # 获取 WHISPER_SAMPLING_BEAM_SEARCH 模式下的默认参数中的 beam_size 值
    int32_t beam_size    = whisper_full_default_params(WHISPER_SAMPLING_BEAM_SEARCH).beam_search.beam_size;

    # 设置 word_thold 的值为 0.01
    float word_thold    =  0.01f;
    # 设置 entropy_thold 的值为 2.40
    float entropy_thold =  2.40f;
    # 设置 logprob_thold 的值为 -1.00
    float logprob_thold = -1.00f;

    # 初始化布尔类型变量并设置初始值
    bool speed_up        = false;
    bool debug_mode      = false;
    bool translate       = false;
    bool detect_language = false;
    bool diarize         = false;
    bool tinydiarize     = false;
    bool split_on_word   = false;
    bool no_fallback     = false;
    bool output_txt      = false;
    bool output_vtt      = false;
    bool output_srt      = false;
    bool output_wts      = false;
    bool output_csv      = false;
    bool output_jsn      = false;
    bool output_jsn_full = false;
    bool output_lrc      = false;
    bool print_special   = false;
    bool print_colors    = false;
    bool print_progress  = false;
    bool no_timestamps   = false;
    bool log_score       = false;
    bool use_gpu         = true;

    # 初始化字符串类型变量并设置初始值
    std::string language  = "en";
    std::string prompt;
    std::string font_path = "/System/Library/Fonts/Supplemental/Courier New Bold.ttf";
    std::string model     = "models/ggml-base.en.bin";

    # [TDRZ] speaker turn string
    # 设置 tdrz_speaker_turn 的值为 " [SPEAKER_TURN]"，并添加 TODO 提示
    std::string tdrz_speaker_turn = " [SPEAKER_TURN]"; // TODO: set from command line

    # 设置 openvino_encode_device 的值为 "CPU"
    std::string openvino_encode_device = "CPU";

    # 初始化字符串数组并设置初始值为空数组
    std::vector<std::string> fname_inp = {};
    std::vector<std::string> fname_out = {};
};

// 打印程序的使用方法
void whisper_print_usage(int argc, char ** argv, const whisper_params & params);

// 解析命令行参数
bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    // 解析命令行参数并设置参数对象
    }

    // 解析成功，返回 true
    return true;
}

// 打印程序的使用方法
void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params) {
    // 打印使用方法
    fprintf(stderr, "\n");
    fprintf(stderr, "usage: %s [options] file0.wav file1.wav ...\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h,        --help              [default] show this help message and exit\n");
    fprintf(stderr, "  -t N,      --threads N         [%-7d] number of threads to use during computation\n",    params.n_threads);
    fprintf(stderr, "  -p N,      --processors N      [%-7d] number of processors to use during computation\n", params.n_processors);
    fprintf(stderr, "  -ot N,     --offset-t N        [%-7d] time offset in milliseconds\n",                    params.offset_t_ms);
    fprintf(stderr, "  -on N,     --offset-n N        [%-7d] segment index offset\n",                           params.offset_n);
    fprintf(stderr, "  -d  N,     --duration N        [%-7d] duration of audio to process in milliseconds\n",   params.duration_ms);
    fprintf(stderr, "  -mc N,     --max-context N     [%-7d] maximum number of text context tokens to store\n", params.max_context);
    fprintf(stderr, "  -ml N,     --max-len N         [%-7d] maximum segment length in characters\n",           params.max_len);
    fprintf(stderr, "  -sow,      --split-on-word     [%-7s] split on word rather than on token\n",             params.split_on_word ? "true" : "false");
    fprintf(stderr, "  -bo N,     --best-of N         [%-7d] number of best candidates to keep\n",              params.best_of);
    fprintf(stderr, "  -bs N,     --beam-size N       [%-7d] beam size for beam search\n",                      params.beam_size);
    # 打印 word timestamp probability threshold 参数的帮助信息
    fprintf(stderr, "  -wt N,     --word-thold N      [%-7.2f] word timestamp probability threshold\n",         params.word_thold);
    # 打印 entropy threshold 参数的帮助信息
    fprintf(stderr, "  -et N,     --entropy-thold N   [%-7.2f] entropy threshold for decoder fail\n",           params.entropy_thold);
    # 打印 log probability threshold 参数的帮助信息
    fprintf(stderr, "  -lpt N,    --logprob-thold N   [%-7.2f] log probability threshold for decoder fail\n",   params.logprob_thold);
    # 打印 speed up audio 参数的帮助信息
    // fprintf(stderr, "  -su,       --speed-up          [%-7s] speed up audio by x2 (reduced accuracy)\n",        params.speed_up ? "true" : "false");
    # 打印 debug mode 参数的帮助信息
    fprintf(stderr, "  -debug,    --debug-mode        [%-7s] enable debug mode (eg. dump log_mel)\n",           params.debug_mode ? "true" : "false");
    # 打印 translate 参数的帮助信息
    fprintf(stderr, "  -tr,       --translate         [%-7s] translate from source language to english\n",      params.translate ? "true" : "false");
    # 打印 diarize 参数的帮助信息
    fprintf(stderr, "  -di,       --diarize           [%-7s] stereo audio diarization\n",                       params.diarize ? "true" : "false");
    # 打印 tinydiarize 参数的帮助信息
    fprintf(stderr, "  -tdrz,     --tinydiarize       [%-7s] enable tinydiarize (requires a tdrz model)\n",     params.tinydiarize ? "true" : "false");
    # 打印 no fallback 参数的帮助信息
    fprintf(stderr, "  -nf,       --no-fallback       [%-7s] do not use temperature fallback while decoding\n", params.no_fallback ? "true" : "false");
    # 打印 output txt 参数的帮助信息
    fprintf(stderr, "  -otxt,     --output-txt        [%-7s] output result in a text file\n",                   params.output_txt ? "true" : "false");
    # 打印 output vtt 参数的帮助信息
    fprintf(stderr, "  -ovtt,     --output-vtt        [%-7s] output result in a vtt file\n",                    params.output_vtt ? "true" : "false");
    # 打印 output srt 参数的帮助信息
    fprintf(stderr, "  -osrt,     --output-srt        [%-7s] output result in a srt file\n",                    params.output_srt ? "true" : "false");
    # 打印 output lrc 参数的帮助信息
    fprintf(stderr, "  -olrc,     --output-lrc        [%-7s] output result in a lrc file\n",                    params.output_lrc ? "true" : "false");
    # 输出参数信息到标准错误流，包括输出 karaoke 视频脚本的选项
    fprintf(stderr, "  -owts,     --output-words      [%-7s] output script for generating karaoke video\n",     params.output_wts ? "true" : "false");
    # 输出参数信息到标准错误流，包括 karaoke 视频所需的等宽字体路径选项
    fprintf(stderr, "  -fp,       --font-path         [%-7s] path to a monospace font for karaoke video\n",     params.font_path.c_str());
    # 输出参数信息到标准错误流，包括输出结果到 CSV 文件的选项
    fprintf(stderr, "  -ocsv,     --output-csv        [%-7s] output result in a CSV file\n",                    params.output_csv ? "true" : "false");
    # 输出参数信息到标准错误流，包括输出结果到 JSON 文件的选项
    fprintf(stderr, "  -oj,       --output-json       [%-7s] output result in a JSON file\n",                   params.output_jsn ? "true" : "false");
    # 输出参数信息到标准错误流，包括在 JSON 文件中包含更多信息的选项
    fprintf(stderr, "  -ojf,      --output-json-full  [%-7s] include more information in the JSON file\n",      params.output_jsn_full ? "true" : "false");
    # 输出参数信息到标准错误流，包括输出文件路径的选项
    fprintf(stderr, "  -of FNAME, --output-file FNAME [%-7s] output file path (without file extension)\n",      "");
    # 输出参数信息到标准错误流，包括打印特殊标记的选项
    fprintf(stderr, "  -ps,       --print-special     [%-7s] print special tokens\n",                           params.print_special ? "true" : "false");
    # 输出参数信息到标准错误流，包括打印颜色的选项
    fprintf(stderr, "  -pc,       --print-colors      [%-7s] print colors\n",                                   params.print_colors ? "true" : "false");
    # 输出参数信息到标准错误流，包括打印进度的选项
    fprintf(stderr, "  -pp,       --print-progress    [%-7s] print progress\n",                                 params.print_progress ? "true" : "false");
    # 输出参数信息到标准错误流，包括不打印时间戳的选项
    fprintf(stderr, "  -nt,       --no-timestamps     [%-7s] do not print timestamps\n",                        params.no_timestamps ? "true" : "false");
    # 输出参数信息到标准错误流，包括语言选项
    fprintf(stderr, "  -l LANG,   --language LANG     [%-7s] spoken language ('auto' for auto-detect)\n",       params.language.c_str());
    # 输出参数信息到标准错误流，包括自动检测语言后退出的选项
    fprintf(stderr, "  -dl,       --detect-language   [%-7s] exit after automatically detecting language\n",    params.detect_language ? "true" : "false");
    # 输出参数信息到标准错误流，包括初始提示选项
    fprintf(stderr, "             --prompt PROMPT     [%-7s] initial prompt\n",                                 params.prompt.c_str());
    # 打印模型路径的帮助信息，包括参数默认值
    fprintf(stderr, "  -m FNAME,  --model FNAME       [%-7s] model path\n",                                     params.model.c_str());
    # 打印输入 WAV 文件路径的帮助信息，包括参数默认值
    fprintf(stderr, "  -f FNAME,  --file FNAME        [%-7s] input WAV file path\n",                            "");
    # 打印用于编码推理的 OpenVINO 设备的帮助信息，包括参数默认值
    fprintf(stderr, "  -oved D,   --ov-e-device DNAME [%-7s] the OpenVINO device used for encode inference\n",  params.openvino_encode_device.c_str());
    # 打印是否记录最佳解码器标记的得分的帮助信息，包括参数默认值
    fprintf(stderr, "  -ls,       --log-score         [%-7s] log best decoder scores of tokens\n",              params.log_score?"true":"false");
    # 打印是否禁用 GPU 的帮助信息，包括参数默认值
    fprintf(stderr, "  -ng,       --no-gpu            [%-7s] disable GPU\n",                                    params.use_gpu ? "false" : "true");
    # 打印空行
    fprintf(stderr, "\n");
// 结构体定义，包含参数和音频数据
struct whisper_print_user_data {
    const whisper_params * params;  // 指向whisper_params结构体的指针

    const std::vector<std::vector<float>> * pcmf32s;  // 指向包含音频数据的二维向量的指针
    int progress_prev;  // 进度的前一个值
};

// 估计辨认说话者的函数
std::string estimate_diarization_speaker(std::vector<std::vector<float>> pcmf32s, int64_t t0, int64_t t1, bool id_only = false) {
    std::string speaker = "";  // 初始化说话者为空字符串
    const int64_t n_samples = pcmf32s[0].size();  // 获取音频数据的样本数

    const int64_t is0 = timestamp_to_sample(t0, n_samples);  // 将时间戳转换为样本数
    const int64_t is1 = timestamp_to_sample(t1, n_samples);  // 将时间戳转换为样本数

    double energy0 = 0.0f;  // 初始化能量值为0
    double energy1 = 0.0f;  // 初始化能量值为0

    // 计算指定时间段内的能量值
    for (int64_t j = is0; j < is1; j++) {
        energy0 += fabs(pcmf32s[0][j]);
        energy1 += fabs(pcmf32s[1][j]);
    }

    // 根据能量比较判断说话者
    if (energy0 > 1.1*energy1) {
        speaker = "0";
    } else if (energy1 > 1.1*energy0) {
        speaker = "1";
    } else {
        speaker = "?";
    }

    // 如果不仅仅是ID，添加说话者标签
    if (!id_only) {
        speaker.insert(0, "(speaker ");  // 在字符串开头插入“(speaker ”
        speaker.append(")");  // 在字符串末尾添加“)”
    }

    return speaker;  // 返回说话者
}

// 打印进度回调函数
void whisper_print_progress_callback(struct whisper_context * /*ctx*/, struct whisper_state * /*state*/, int progress, void * user_data) {
    int progress_step = ((whisper_print_user_data *) user_data)->params->progress_step;  // 获取进度步长
    int * progress_prev  = &(((whisper_print_user_data *) user_data)->progress_prev);  // 获取进度的前一个值的指针
    if (progress >= *progress_prev + progress_step) {  // 如果进度超过前一个值加上步长
        *progress_prev += progress_step;  // 更新前一个值
        fprintf(stderr, "%s: progress = %3d%%\n", __func__, progress);  // 打印进度信息
    }
}

// 打印分段回调函数
void whisper_print_segment_callback(struct whisper_context * ctx, struct whisper_state * /*state*/, int n_new, void * user_data) {
    const auto & params  = *((whisper_print_user_data *) user_data)->params;  // 获取参数
    const auto & pcmf32s = *((whisper_print_user_data *) user_data)->pcmf32s;  // 获取音频数据

    const int n_segments = whisper_full_n_segments(ctx);  // 获取分段数

    std::string speaker = "";  // 初始化说话者为空字符串

    int64_t t0 = 0;  // 初始化时间戳为0
    int64_t t1 = 0;  // 初始化时间戳为0
    // 打印最后 n_new 个片段
    const int s0 = n_segments - n_new;

    // 如果 s0 等于 0，则打印换行符
    if (s0 == 0) {
        printf("\n");
    }

    // 遍历从 s0 到 n_segments 的片段
    for (int i = s0; i < n_segments; i++) {
        // 如果不禁用时间戳或者进行说话人分离
        if (!params.no_timestamps || params.diarize) {
            t0 = whisper_full_get_segment_t0(ctx, i);
            t1 = whisper_full_get_segment_t1(ctx, i);
        }

        // 如果不禁用时间戳，则打印时间戳
        if (!params.no_timestamps) {
            printf("[%s --> %s]  ", to_timestamp(t0).c_str(), to_timestamp(t1).c_str());
        }

        // 如果进行说话人分离并且 pcmf32s 的大小为 2，则估计说话人
        if (params.diarize && pcmf32s.size() == 2) {
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        // 如果打印颜色
        if (params.print_colors) {
            // 遍历当前片段的所有标记
            for (int j = 0; j < whisper_full_n_tokens(ctx, i); ++j) {
                // 如果不打印特殊标记
                if (params.print_special == false) {
                    const whisper_token id = whisper_full_get_token_id(ctx, i, j);
                    if (id >= whisper_token_eot(ctx)) {
                        continue;
                    }
                }

                // 获取标记的文本和概率
                const char * text = whisper_full_get_token_text(ctx, i, j);
                const float  p    = whisper_full_get_token_p   (ctx, i, j);

                // 计算颜色索引
                const int col = std::max(0, std::min((int) k_colors.size() - 1, (int) (std::pow(p, 3)*float(k_colors.size()))));

                // 打印说话人、颜色、文本和重置颜色
                printf("%s%s%s%s", speaker.c_str(), k_colors[col].c_str(), text, "\033[0m");
            }
        } else {
            // 获取当前片段的文本
            const char * text = whisper_full_get_segment_text(ctx, i);

            // 打印说话人和文本
            printf("%s%s", speaker.c_str(), text);
        }

        // 如果启用 tinydiarize，并且下一个片段是新的说话人
        if (params.tinydiarize) {
            if (whisper_full_get_segment_speaker_turn_next(ctx, i)) {
                printf("%s", params.tdrz_speaker_turn.c_str());
            }
        }

        // 如果有时间戳或者说话人：每个片段换行
        if (!params.no_timestamps || params.diarize) {
            printf("\n");
        }

        // 刷新标准输出
        fflush(stdout);
    }
// 输出文本文件，包含了语音识别的结果
bool output_txt(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    // 打开文件流，准备写入结果
    std::ofstream fout(fname);
    // 如果文件流未成功打开，输出错误信息并返回false
    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    // 输出提示信息，表示正在保存结果到指定文件
    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    // 获取语音识别结果的段数
    const int n_segments = whisper_full_n_segments(ctx);
    // 遍历每个识别结果段落
    for (int i = 0; i < n_segments; ++i) {
        // 获取当前段落的文本内容
        const char * text = whisper_full_get_segment_text(ctx, i);
        // 初始化说话者为空字符串
        std::string speaker = "";

        // 如果启用了说话者分离功能，并且音频数据包含两个声道
        if (params.diarize && pcmf32s.size() == 2)
        {
            // 获取当前段落的起始时间和结束时间
            const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
            const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
            // 估计说话者并赋值给speaker变量
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        // 将说话者和文本内容写入文件
        fout << speaker << text << "\n";
    }

    // 返回true表示输出成功
    return true;
}

// 输出WebVTT格式的文件，包含了语音识别的结果
bool output_vtt(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    // 打开文件流，准备写入结果
    std::ofstream fout(fname);
    // 如果文件流未成功打开，输出错误信息并返回false
    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    // 输出提示信息，表示正在保存结果到指定文件
    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    // 写入WebVTT文件的头部信息
    fout << "WEBVTT\n\n";

    // 获取语音识别结果的段数
    const int n_segments = whisper_full_n_segments(ctx);
    # 遍历每个语音段
    for (int i = 0; i < n_segments; ++i) {
        # 获取当前语音段的文本内容
        const char * text = whisper_full_get_segment_text(ctx, i);
        # 获取当前语音段的起始时间
        const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
        # 获取当前语音段的结束时间
        const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
        # 初始化发言者为空字符串
        std::string speaker = "";

        # 如果需要说话者分离，并且音频格式为pcm32f
        if (params.diarize && pcmf32s.size() == 2)
        {
            # 估计说话者并赋值给speaker
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1, true);
            # 在speaker字符串前插入"<v Speaker"
            speaker.insert(0, "<v Speaker");
            # 在speaker字符串后追加">"
            speaker.append(">");
        }

        # 将当前语音段的起始时间和结束时间以时间戳格式输出到fout
        fout << to_timestamp(t0) << " --> " << to_timestamp(t1) << "\n";
        # 将发言者和文本内容输出到fout
        fout << speaker << text << "\n\n";
    }

    # 返回true
    return true;
}
// 输出 SRT 文件
bool output_srt(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    // 打开文件流
    std::ofstream fout(fname);
    // 如果文件流未成功打开
    if (!fout.is_open()) {
        // 输出错误信息
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        // 返回失败
        return false;
    }

    // 输出保存输出到文件的信息
    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    // 获取语音段数
    const int n_segments = whisper_full_n_segments(ctx);
    // 遍历每个语音段
    for (int i = 0; i < n_segments; ++i) {
        // 获取语音段文本
        const char * text = whisper_full_get_segment_text(ctx, i);
        // 获取语音段起始时间和结束时间
        const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
        const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
        // 初始化发言者为空字符串
        std::string speaker = "";

        // 如果需要说话者分离，并且音频数据为双声道
        if (params.diarize && pcmf32s.size() == 2)
        {
            // 估计说话者
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        // 输出语音段编号
        fout << i + 1 + params.offset_n << "\n";
        // 输出语音段起始时间和结束时间
        fout << to_timestamp(t0, true) << " --> " << to_timestamp(t1, true) << "\n";
        // 输出说话者和文本
        fout << speaker << text << "\n\n";
    }

    // 返回成功
    return true;
}

// 转义双引号和反斜杠
char *escape_double_quotes_and_backslashes(const char *str) {
    // 如果输入为空，则返回空指针
    if (str == NULL) {
        return NULL;
    }

    // 计算转义后的字符串长度
    size_t escaped_length = strlen(str) + 1;

    // 遍历字符串，计算需要转义的字符数量
    for (size_t i = 0; str[i] != '\0'; i++) {
        if (str[i] == '"' || str[i] == '\\') {
            escaped_length++;
        }
    }

    // 分配转义后的字符串内存空间
    char *escaped = (char *)calloc(escaped_length, 1); // 预先清零
    // 如果内存分配失败，则返回空指针
    if (escaped == NULL) {
        return NULL;
    }

    // 进行转义处理
    size_t pos = 0;
    for (size_t i = 0; str[i] != '\0'; i++) {
        if (str[i] == '"' || str[i] == '\\') {
            escaped[pos++] = '\\';
        }
        escaped[pos++] = str[i];
    }

    // 由于之前使用了calloc()，不需要手动置零

    // 返回转义后的字符串
    return escaped;
}

// 输出 CSV 文件
bool output_csv(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    // 打开文件流
    std::ofstream fout(fname);
    // 如果输出文件没有成功打开
    if (!fout.is_open()) {
        // 输出错误信息到标准错误流
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        // 返回 false
        return false;
    }

    // 输出信息到标准错误流，表示正在保存输出到指定文件
    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    // 获取音频文件的段数
    const int n_segments = whisper_full_n_segments(ctx);
    // 写入表头信息到输出文件
    fout << "start,end,";
    // 如果启用了说话人分离，并且音频通道数为2
    if (params.diarize && pcmf32s.size() == 2)
    {
        // 写入说话人信息到输出文件
        fout << "speaker,";
    }
    // 写入文本信息到输出文件
    fout << "text\n";

    // 遍历每个音频段
    for (int i = 0; i < n_segments; ++i) {
        // 获取当前段的文本信息
        const char * text = whisper_full_get_segment_text(ctx, i);
        // 获取当前段的起始时间和结束时间
        const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
        const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
        // 对文本信息进行转义处理
        char * text_escaped = escape_double_quotes_and_backslashes(text);

        // 需要将从whisper_full_get_segment_t{0,1}()返回的时间乘以10以获取毫秒数
        fout << 10 * t0 << "," << 10 * t1 << ",";
        // 如果启用了说话人分离，并且音频通道数为2
        if (params.diarize && pcmf32s.size() == 2)
        {
            // 估计说话人分离的说话人信息，并写入输出文件
            fout << estimate_diarization_speaker(pcmf32s, t0, t1, true) << ",";
        }
        // 写入转义后的文本信息到输出文件
        fout << "\"" << text_escaped << "\"\n";
    }

    // 返回 true
    return true;
// 输出分数到文件
bool output_score(struct whisper_context * ctx, const char * fname, const whisper_params & /*params*/, std::vector<std::vector<float>> /*pcmf32s*/) {
    // 打开文件流，准备写入输出
    std::ofstream fout(fname);
    // 打印输出文件名到标准错误流
    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);

    // 获取上下文中的分段数
    const int n_segments = whisper_full_n_segments(ctx);
    // 遍历每个分段
    for (int i = 0; i < n_segments; ++i) {
        // 获取第 i 个分段中的标记数
        const int n_tokens = whisper_full_n_tokens(ctx, i);
        // 遍历每个标记
        for (int j = 0; j < n_tokens; j++) {
            // 获取第 i 个分段中第 j 个标记的文本和概率
            auto token = whisper_full_get_token_text(ctx, i, j);
            auto probability = whisper_full_get_token_p(ctx, i, j);
            // 将标记和概率写入输出文件
            fout << token << '\t' << probability << std::endl;
        }
    }
    return true;
}

// 输出 JSON 格式的结果
bool output_json(
             struct whisper_context * ctx,
                         const char * fname,
               const whisper_params & params,
    std::vector<std::vector<float>>   pcmf32s,
                               bool   full) {
    // 打开文件流，准备写入输出
    std::ofstream fout(fname);
    int indent = 0;

    // 定义缩进函数
    auto doindent = [&]() {
        for (int i = 0; i < indent; i++) fout << "\t";
    };

    // 定义开始数组的函数
    auto start_arr = [&](const char *name) {
        doindent();
        fout << "\"" << name << "\": [\n";
        indent++;
    };

    // 定义结束数组的函数
    auto end_arr = [&](bool end) {
        indent--;
        doindent();
        fout << (end ? "]\n" : "],\n");
    };

    // 定义开始对象的函数
    auto start_obj = [&](const char *name) {
        doindent();
        if (name) {
            fout << "\"" << name << "\": {\n";
        } else {
            fout << "{\n";
        }
        indent++;
    };

    // 定义结束对象的函数
    auto end_obj = [&](bool end) {
        indent--;
        doindent();
        fout << (end ? "}\n" : "},\n");
    };

    // 定义开始值的函数
    auto start_value = [&](const char *name) {
        doindent();
        fout << "\"" << name << "\": ";
    };
    // 定义一个lambda函数，用于输出字符串类型的数值
    auto value_s = [&](const char *name, const char *val, bool end) {
        // 调用start_value函数，开始输出值
        start_value(name);
        // 对值进行转义处理，防止出现双引号和反斜杠
        char * val_escaped = escape_double_quotes_and_backslashes(val);
        // 将转义后的值输出到文件流中
        fout << "\"" << val_escaped << (end ? "\"\n" : "\",\n");
        // 释放转义后的值的内存
        free(val_escaped);
    };

    // 定义一个lambda函数，用于输出结束标记
    auto end_value = [&](bool end) {
        // 根据end参数决定是否输出换行符
        fout << (end ? "\n" : ",\n");
    };

    // 定义一个lambda函数，用于输出整数类型的数值
    auto value_i = [&](const char *name, const int64_t val, bool end) {
        // 调用start_value函数，开始输出值
        start_value(name);
        // 将整数值输出到文件流中
        fout << val;
        // 调用end_value函数，输出结束标记
        end_value(end);
    };

    // 定义一个lambda函数，用于输出浮点数类型的数值
    auto value_f = [&](const char *name, const float val, bool end) {
        // 调用start_value函数，开始输出值
        start_value(name);
        // 将浮点数值输出到文件流中
        fout << val;
        // 调用end_value函数，输出结束标记
        end_value(end);
    };

    // 定义一个lambda函数，用于输出布尔类型的数值
    auto value_b = [&](const char *name, const bool val, bool end) {
        // 调用start_value函数，开始输出值
        start_value(name);
        // 根据布尔值输出true或false到文件流中
        fout << (val ? "true" : "false");
        // 调用end_value函数，输出结束标记
        end_value(end);
    };

    // 定义一个lambda函数，用于输出时间戳范围
    auto times_o = [&](int64_t t0, int64_t t1, bool end) {
        // 调用start_obj函数，开始输出对象
        start_obj("timestamps");
        // 输出起始时间戳
        value_s("from", to_timestamp(t0, true).c_str(), false);
        // 输出结束时间戳
        value_s("to", to_timestamp(t1, true).c_str(), true);
        // 调用end_obj函数，结束输出对象
        end_obj(false);
        // 调用start_obj函数，开始输出对象
        start_obj("offsets");
        // 输出起始时间戳的10倍
        value_i("from", t0 * 10, false);
        // 输出结束时间戳的10倍
        value_i("to", t1 * 10, true);
        // 调用end_obj函数，结束输出对象
        end_obj(end);
    };

    // 如果文件流未打开，则输出错误信息并返回false
    if (!fout.is_open()) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname);
        return false;
    }

    // 输出保存输出到文件的信息
    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname);
    // 调用end_obj函数，结束输出对象
    end_obj(true);
    // 返回true
    return true;
// 生成卡拉OK视频
// 输出一个 bash 脚本，使用 ffmpeg 生成带有字幕的视频
// TODO: 字体参数调整
bool output_wts(struct whisper_context * ctx, const char * fname, const char * fname_inp, const whisper_params & params, float t_sec, std::vector<std::vector<float>> pcmf32s) {
    std::ofstream fout(fname); // 打开一个输出文件流

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname); // 输出保存输出的信息到标准错误流

    static const char * font = params.font_path.c_str(); // 获取字体路径

    std::ifstream fin(font); // 打开字体文件
    if (!fin.is_open()) { // 如果打开失败
        fprintf(stderr, "%s: font not found at '%s', please specify a monospace font with -fp\n", __func__, font); // 输出错误信息到标准错误流
        return false; // 返回 false
    }

    fout << "#!/bin/bash" << "\n"; // 写入 bash 脚本头部
    fout << "\n";

    fout << "ffmpeg -i " << fname_inp << " -f lavfi -i color=size=1200x120:duration=" << t_sec << ":rate=25:color=black -vf \""; // 写入 ffmpeg 命令

    }

    fout << "\" -c:v libx264 -pix_fmt yuv420p -y " << fname_inp << ".mp4" << "\n"; // 写入 ffmpeg 命令

    fout << "\n\n";
    fout << "echo \"Your video has been saved to " << fname_inp << ".mp4\"" << "\n"; // 输出保存视频的信息
    fout << "\n";
    fout << "echo \"  ffplay " << fname_inp << ".mp4\"\n"; // 输出播放视频的信息
    fout << "\n";

    fout.close(); // 关闭输出文件流

    fprintf(stderr, "%s: run 'source %s' to generate karaoke video\n", __func__, fname); // 输出运行脚本的信息到标准错误流

    return true; // 返回 true
}

bool output_lrc(struct whisper_context * ctx, const char * fname, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    std::ofstream fout(fname); // 打开一个输出文件流
    if (!fout.is_open()) { // 如果打开失败
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname); // 输出错误信息到标准错误流
        return false; // 返回 false
    }

    fprintf(stderr, "%s: saving output to '%s'\n", __func__, fname); // 输出保存输出的信息到标准错误流

    fout << "[by:whisper.cpp]\n"; // 写入作者信息

    const int n_segments = whisper_full_n_segments(ctx); // 获取段数
    // 遍历每个段落
    for (int i = 0; i < n_segments; ++i) {
        // 获取当前段落的文本内容
        const char * text = whisper_full_get_segment_text(ctx, i);
        // 获取当前段落的起始时间
        const int64_t t = whisper_full_get_segment_t0(ctx, i);

        // 将时间转换为毫秒
        int64_t msec = t * 10;
        // 计算分钟
        int64_t min = msec / (1000 * 60);
        msec = msec - min * (1000 * 60);
        // 计算秒
        int64_t sec = msec / 1000;
        msec = msec - sec * 1000;

        // 格式化时间为字符串
        char buf[16];
        snprintf(buf, sizeof(buf), "%02d:%02d.%02d", (int) min, (int) sec, (int) ( msec / 10));
        std::string timestamp_lrc = std::string(buf);
        std::string speaker = "";

        // 如果启用了说话人分离功能，并且音频通道数为2
        if (params.diarize && pcmf32s.size() == 2)
        {
            // 获取当前段落的起始时间和结束时间
            const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
            const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
            // 估算说话人
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        // 将时间戳、说话人和文本内容写入输出流
        fout <<  '[' << timestamp_lrc << ']' << speaker << text << "\n";
    }

    // 返回处理结果
    return true;
// 主函数，接受命令行参数
int main(int argc, char ** argv) {
    // 创建参数对象
    whisper_params params;

    // 解析命令行参数，如果解析失败则打印用法并返回错误码
    if (whisper_params_parse(argc, argv, params) == false) {
        whisper_print_usage(argc, argv, params);
        return 1;
    }

    // 如果输入文件名为空，则打印错误信息并返回错误码
    if (params.fname_inp.empty()) {
        fprintf(stderr, "error: no input files specified\n");
        whisper_print_usage(argc, argv, params);
        return 2;
    }

    // 如果语言不是自动识别且不在支持的语言列表中，则打印错误信息并返回错误码
    if (params.language != "auto" && whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // 如果同时使用了 --diarize 和 --tinydiarize，则打印错误信息并返回错误码
    if (params.diarize && params.tinydiarize) {
        fprintf(stderr, "error: cannot use both --diarize and --tinydiarize\n");
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // 初始化 whisper 上下文参数
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;

    // 从文件初始化 whisper 上下文
    struct whisper_context * ctx = whisper_init_from_file_with_params(params.model.c_str(), cparams);

    // 如果初始化失败，则打印错误信息并返回错误码
    if (ctx == nullptr) {
        fprintf(stderr, "error: failed to initialize whisper context\n");
        return 3;
    }

    // 初始化 openvino 编码器，对于没有配置 OpenVINO 的 whisper.cpp 构建，这不会产生任何影响
    whisper_ctx_init_openvino_encoder(ctx, nullptr, params.openvino_encode_device.c_str(), nullptr);

    // 打印时间信息
    whisper_print_timings(ctx);
    // 释放 whisper 上下文
    whisper_free(ctx);

    // 返回成功码
    return 0;
}
```