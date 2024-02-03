# `whisper.cpp\examples\server\server.cpp`

```cpp
// 包含常用头文件
#include "common.h"

// 包含自定义的whisper头文件
#include "whisper.h"
// 包含httplib库的头文件
#include "httplib.h"
// 包含json库的头文件
#include "json.hpp"

// 包含数学库的头文件
#include <cmath>
// 包含文件流库的头文件
#include <fstream>
// 包含C标准输入输出库的头文件
#include <cstdio>
// 包含字符串库的头文件
#include <string>
// 包含线程库的头文件
#include <thread>
// 包含向量库的头文件
#include <vector>
// 包含字符串操作库的头文件
#include <cstring>
// 包含字符串流库的头文件
#include <sstream>

// 如果是在MSVC编译器下，禁止警告4244和4267
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 使用httplib命名空间
using namespace httplib;
// 使用json库的ordered_json别名
using json = nlohmann::ordered_json;

// 匿名命名空间，定义终端颜色映射
const std::vector<std::string> k_colors = {
    "\033[38;5;196m", "\033[38;5;202m", "\033[38;5;208m", "\033[38;5;214m", "\033[38;5;220m",
    "\033[38;5;226m", "\033[38;5;190m", "\033[38;5;154m", "\033[38;5;118m", "\033[38;5;82m",
};

// 定义输出格式常量
const std::string json_format   = "json";
const std::string text_format   = "text";
const std::string srt_format    = "srt";
const std::string vjson_format  = "verbose_json";
const std::string vtt_format    = "vtt";

// 定义服务器参数结构体
struct server_params
{
    std::string hostname = "127.0.0.1";
    std::string public_path = "examples/server/public";
    std::string request_path = "";

    int32_t port          = 8080;
    int32_t read_timeout  = 600;
    int32_t write_timeout = 600;

    bool ffmpeg_converter = false;
};

// 定义whisper参数结构体
struct whisper_params {
    int32_t n_threads     = std::min(4, (int32_t) std::thread::hardware_concurrency());
    int32_t n_processors  = 1;
    int32_t offset_t_ms   = 0;
    int32_t offset_n      = 0;
    int32_t duration_ms   = 0;
    int32_t progress_step = 5;
    int32_t max_context   = -1;
    int32_t max_len       = 0;
    int32_t best_of       = 2;
    int32_t beam_size     = -1;

    float word_thold      =  0.01f;
    float entropy_thold   =  2.40f;
    float logprob_thold   = -1.00f;
    float temperature     =  0.00f;
    float temperature_inc =  0.20f;

    bool speed_up        = false;
    bool debug_mode      = false;
    bool translate       = false;
    bool detect_language = false;
    // 是否进行说话人分离
    bool diarize = false;
    // 是否进行小型说话人分离
    bool tinydiarize = false;
    // 是否按单词拆分
    bool split_on_word = false;
    // 是否禁用回退
    bool no_fallback = false;
    // 是否打印特殊字符
    bool print_special = false;
    // 是否打印颜色
    bool print_colors = false;
    // 是否实时打印
    bool print_realtime = false;
    // 是否打印进度
    bool print_progress = false;
    // 是否禁用时间戳
    bool no_timestamps = false;
    // 是否使用 GPU
    bool use_gpu = true;

    // 语言设置为英文
    std::string language = "en";
    // 提示信息为空
    std::string prompt = "";
    // 字体路径设置为默认路径
    std::string font_path = "/System/Library/Fonts/Supplemental/Courier New Bold.ttf";
    // 模型路径设置为默认路径
    std::string model = "models/ggml-base.en.bin";

    // 响应格式设置为 JSON 格式
    std::string response_format = json_format;

    // [TDRZ] 说话人转换字符串
    std::string tdrz_speaker_turn = " [SPEAKER_TURN]"; // TODO: set from command line

    // OpenVINO 编码设备设置为 CPU
    std::string openvino_encode_device = "CPU";
// 将时间戳转换为格式化的时间字符串，单位为毫秒，可选择是否使用逗号分隔
std::string to_timestamp(int64_t t, bool comma = false) {
    // 将时间戳转换为毫秒
    int64_t msec = t * 10;
    // 计算小时
    int64_t hr = msec / (1000 * 60 * 60);
    msec = msec - hr * (1000 * 60 * 60);
    // 计算分钟
    int64_t min = msec / (1000 * 60);
    msec = msec - min * (1000 * 60);
    // 计算秒
    int64_t sec = msec / 1000;
    msec = msec - sec * 1000;

    // 格式化时间字符串
    char buf[32];
    snprintf(buf, sizeof(buf), "%02d:%02d:%02d%s%03d", (int) hr, (int) min, (int) sec, comma ? "," : ".", (int) msec);

    return std::string(buf);
}

// 将时间戳转换为采样点索引
int timestamp_to_sample(int64_t t, int n_samples) {
    // 计算采样点索引
    return std::max(0, std::min((int) n_samples - 1, (int) ((t*WHISPER_SAMPLE_RATE)/100)));
}

// 检查文件是否存在
bool is_file_exist(const char *fileName)
{
    // 打开文件流进行检查
    std::ifstream infile(fileName);
    return infile.good();
}

// 打印程序的使用说明
void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params,
                         const server_params& sparams) {
    // 打印使用说明
    fprintf(stderr, "\n");
    fprintf(stderr, "usage: %s [options] \n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h,        --help              [default] show this help message and exit\n");
    fprintf(stderr, "  -t N,      --threads N         [%-7d] number of threads to use during computation\n",    params.n_threads);
    fprintf(stderr, "  -p N,      --processors N      [%-7d] number of processors to use during computation\n", params.n_processors);
    fprintf(stderr, "  -ot N,     --offset-t N        [%-7d] time offset in milliseconds\n",                    params.offset_t_ms);
    fprintf(stderr, "  -on N,     --offset-n N        [%-7d] segment index offset\n",                           params.offset_n);
    fprintf(stderr, "  -d  N,     --duration N        [%-7d] duration of audio to process in milliseconds\n",   params.duration_ms);
    fprintf(stderr, "  -mc N,     --max-context N     [%-7d] maximum number of text context tokens to store\n", params.max_context);
}
    // 打印最大段落长度参数的帮助信息
    fprintf(stderr, "  -ml N,     --max-len N         [%-7d] maximum segment length in characters\n",           params.max_len);
    // 打印是否在单词上拆分而不是在标记上拆分的参数的帮助信息
    fprintf(stderr, "  -sow,      --split-on-word     [%-7s] split on word rather than on token\n",             params.split_on_word ? "true" : "false");
    // 打印保留的最佳候选数参数的帮助信息
    fprintf(stderr, "  -bo N,     --best-of N         [%-7d] number of best candidates to keep\n",              params.best_of);
    // 打印束搜索的束大小参数的帮助信息
    fprintf(stderr, "  -bs N,     --beam-size N       [%-7d] beam size for beam search\n",                      params.beam_size);
    // 打印单词时间戳概率阈值参数的帮助信息
    fprintf(stderr, "  -wt N,     --word-thold N      [%-7.2f] word timestamp probability threshold\n",         params.word_thold);
    // 打印解码器失败的熵阈值参数的帮助信息
    fprintf(stderr, "  -et N,     --entropy-thold N   [%-7.2f] entropy threshold for decoder fail\n",           params.entropy_thold);
    // 打印解码器失败的对数概率阈值参数的帮助信息
    fprintf(stderr, "  -lpt N,    --logprob-thold N   [%-7.2f] log probability threshold for decoder fail\n",   params.logprob_thold);
    // 打印是否启用音频加速的帮助信息
    // fprintf(stderr, "  -su,       --speed-up          [%-7s] speed up audio by x2 (reduced accuracy)\n",        params.speed_up ? "true" : "false");
    // 打印是否启用调试模式的帮助信息
    fprintf(stderr, "  -debug,    --debug-mode        [%-7s] enable debug mode (eg. dump log_mel)\n",           params.debug_mode ? "true" : "false");
    // 打印是否从源语言翻译为英语的帮助信息
    fprintf(stderr, "  -tr,       --translate         [%-7s] translate from source language to english\n",      params.translate ? "true" : "false");
    // 打印立体声音频分离的帮助信息
    fprintf(stderr, "  -di,       --diarize           [%-7s] stereo audio diarization\n",                       params.diarize ? "true" : "false");
    // 打印是否启用小型音频分离的帮助信息
    fprintf(stderr, "  -tdrz,     --tinydiarize       [%-7s] enable tinydiarize (requires a tdrz model)\n",     params.tinydiarize ? "true" : "false");
    // 打印解码时是否使用温度回退的帮助信息
    fprintf(stderr, "  -nf,       --no-fallback       [%-7s] do not use temperature fallback while decoding\n", params.no_fallback ? "true" : "false");
    # 打印是否打印特殊标记的参数信息
    fprintf(stderr, "  -ps,       --print-special     [%-7s] print special tokens\n", params.print_special ? "true" : "false");
    # 打印是否打印颜色的参数信息
    fprintf(stderr, "  -pc,       --print-colors      [%-7s] print colors\n", params.print_colors ? "true" : "false");
    # 打印是否实时打印输出的参数信息
    fprintf(stderr, "  -pr,       --print-realtime    [%-7s] print output in realtime\n", params.print_realtime ? "true" : "false");
    # 打印是否打印进度的参数信息
    fprintf(stderr, "  -pp,       --print-progress    [%-7s] print progress\n", params.print_progress ? "true" : "false");
    # 打印是否不打印时间戳的参数信息
    fprintf(stderr, "  -nt,       --no-timestamps     [%-7s] do not print timestamps\n", params.no_timestamps ? "true" : "false");
    # 打印语言参数信息
    fprintf(stderr, "  -l LANG,   --language LANG     [%-7s] spoken language ('auto' for auto-detect)\n", params.language.c_str());
    # 打印是否检测语言的参数信息
    fprintf(stderr, "  -dl,       --detect-language   [%-7s] exit after automatically detecting language\n", params.detect_language ? "true" : "false");
    # 打印初始提示信息的参数信息
    fprintf(stderr, "             --prompt PROMPT     [%-7s] initial prompt\n", params.prompt.c_str());
    # 打印模型路径的参数信息
    fprintf(stderr, "  -m FNAME,  --model FNAME       [%-7s] model path\n", params.model.c_str());
    # 打印 OpenVINO 设备用于编码推理的参数信息
    fprintf(stderr, "  -oved D,   --ov-e-device DNAME [%-7s] the OpenVINO device used for encode inference\n", params.openvino_encode_device.c_str());
    # 服务器参数
    # 打印服务器主机名/ IP 地址的参数信息
    fprintf(stderr, "  --host HOST,                   [%-7s] Hostname/ip-adress for the server\n", sparams.hostname.c_str());
    # 打印服务器端口号的参数信息
    fprintf(stderr, "  --port PORT,                   [%-7d] Port number for the server\n", sparams.port);
    # 打印公共文件夹路径的参数信息
    fprintf(stderr, "  --public PATH,                 [%-7s] Path to the public folder\n", sparams.public_path.c_str());
    # 打印所有请求的请求路径的参数信息
    fprintf(stderr, "  --request-path PATH,           [%-7s] Request path for all requests\n", sparams.request_path.c_str());
    # 输出带有格式化的信息到标准错误流，显示是否启用音频转换为 WAV 格式，需要服务器上安装 ffmpeg
    fprintf(stderr, "  --convert,                     [%-7s] Convert audio to WAV, requires ffmpeg on the server", sparams.ffmpeg_converter ? "true" : "false");
    # 输出换行符到标准错误流
    fprintf(stderr, "\n");
}

// 解析命令行参数，填充参数结构体，返回是否成功
bool whisper_params_parse(int argc, char ** argv, whisper_params & params, server_params & sparams) {
    // 略
    return true;
}

// 存储打印用户数据的结构体
struct whisper_print_user_data {
    const whisper_params * params;

    const std::vector<std::vector<float>> * pcmf32s;
    int progress_prev;
};

// 检查系统中是否安装了 ffmpeg
void check_ffmpeg_availibility() {
    // 执行系统命令检查 ffmpeg 版本
    int result = system("ffmpeg -version");

    if (result == 0) {
        std::cout << "ffmpeg is available." << std::endl;
    } else {
        // ffmpeg 未找到
        std::cout << "ffmpeg is not found. Please ensure that ffmpeg is installed ";
        std::cout << "and that its executable is included in your system's PATH. ";
        exit(0);
    }
}

// 将文件转换为 WAV 格式
bool convert_to_wav(const std::string & temp_filename, std::string & error_resp) {
    // 构建 ffmpeg 命令
    std::ostringstream cmd_stream;
    std::string converted_filename_temp = temp_filename + "_temp.wav";
    cmd_stream << "ffmpeg -i \"" << temp_filename << "\" -ar 16000 -ac 1 -c:a pcm_s16le \"" << converted_filename_temp << "\" 2>&1";
    std::string cmd = cmd_stream.str();

    // 执行 ffmpeg 命令
    int status = std::system(cmd.c_str());
    if (status != 0) {
        error_resp = "{\"error\":\"FFmpeg conversion failed.\"}";
        return false;
    }

    // 删除原始文件
    if (remove(temp_filename.c_str()) != 0) {
        error_resp = "{\"error\":\"Failed to remove the original file.\"}";
        return false;
    }

    // 重命名临时文件以匹配原始文件名
    if (rename(converted_filename_temp.c_str(), temp_filename.c_str()) != 0) {
        error_resp = "{\"error\":\"Failed to rename the temporary file.\"}";
        return false;
    }
    return true;
}

// 估算说话者的分离
std::string estimate_diarization_speaker(std::vector<std::vector<float>> pcmf32s, int64_t t0, int64_t t1, bool id_only = false) {
    std::string speaker = "";
    const int64_t n_samples = pcmf32s[0].size();

    const int64_t is0 = timestamp_to_sample(t0, n_samples);
    const int64_t is1 = timestamp_to_sample(t1, n_samples);
    // 初始化能量值为0
    double energy0 = 0.0f;
    double energy1 = 0.0f;

    // 遍历从is0到is1的数据，计算两个通道的能量值
    for (int64_t j = is0; j < is1; j++) {
        energy0 += fabs(pcmf32s[0][j]);
        energy1 += fabs(pcmf32s[1][j]);
    }

    // 根据能量值比较判断说话者
    if (energy0 > 1.1*energy1) {
        speaker = "0";
    } else if (energy1 > 1.1*energy0) {
        speaker = "1";
    } else {
        speaker = "?";
    }

    // 如果不仅仅是ID，则在说话者前后添加标记
    if (!id_only) {
        speaker.insert(0, "(speaker ");
        speaker.append(")");
    }

    // 返回最终的说话者标识
    return speaker;
// 定义一个回调函数，用于打印进度信息
void whisper_print_progress_callback(struct whisper_context * /*ctx*/, struct whisper_state * /*state*/, int progress, void * user_data) {
    // 获取进度步长
    int progress_step = ((whisper_print_user_data *) user_data)->params->progress_step;
    // 获取上一个进度值的指针
    int * progress_prev  = &(((whisper_print_user_data *) user_data)->progress_prev);
    // 如果当前进度超过上一个进度值加上步长，则更新上一个进度值并打印进度信息
    if (progress >= *progress_prev + progress_step) {
        *progress_prev += progress_step;
        fprintf(stderr, "%s: progress = %3d%%\n", __func__, progress);
    }
}

// 定义一个回调函数，用于打印分段信息
void whisper_print_segment_callback(struct whisper_context * ctx, struct whisper_state * /*state*/, int n_new, void * user_data) {
    // 获取用户数据中的参数和 PCM 数据
    const auto & params  = *((whisper_print_user_data *) user_data)->params;
    const auto & pcmf32s = *((whisper_print_user_data *) user_data)->pcmf32s;

    // 获取总分段数
    const int n_segments = whisper_full_n_segments(ctx);

    // 初始化发言者和时间戳
    std::string speaker = "";
    int64_t t0 = 0;
    int64_t t1 = 0;

    // 打印最后 n_new 个分段
    const int s0 = n_segments - n_new;

    // 如果 s0 为 0，则打印换行符
    if (s0 == 0) {
        printf("\n");
    }
}
    // 遍历从 s0 开始到 n_segments 结束的所有段落
    for (int i = s0; i < n_segments; i++) {
        // 如果不禁用时间戳或者进行说话人分离
        if (!params.no_timestamps || params.diarize) {
            // 获取第 i 段的起始时间和结束时间
            t0 = whisper_full_get_segment_t0(ctx, i);
            t1 = whisper_full_get_segment_t1(ctx, i);
        }

        // 如果不禁用时间戳
        if (!params.no_timestamps) {
            // 打印起始时间和结束时间
            printf("[%s --> %s]  ", to_timestamp(t0).c_str(), to_timestamp(t1).c_str());
        }

        // 如果进行说话人分离并且 pcmf32s 的大小为 2
        if (params.diarize && pcmf32s.size() == 2) {
            // 估计说话人分离的说话人
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        // 如果打印颜色
        if (params.print_colors) {
            // 遍历第 i 段的所有标记
            for (int j = 0; j < whisper_full_n_tokens(ctx, i); ++j) {
                // 如果不打印特殊标记
                if (params.print_special == false) {
                    // 获取标记的 ID
                    const whisper_token id = whisper_full_get_token_id(ctx, i, j);
                    // 如果 ID 大于等于结束标记
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
            // 获取第 i 段的文本
            const char * text = whisper_full_get_segment_text(ctx, i);

            // 打印说话人和文本
            printf("%s%s", speaker.c_str(), text);
        }

        // 如果启用 tinydiarize
        if (params.tinydiarize) {
            // 如果下一个段落是说话人转换
            if (whisper_full_get_segment_speaker_turn_next(ctx, i)) {
                // 打印说话人转换标记
                printf("%s", params.tdrz_speaker_turn.c_str());
            }
        }

        // 如果有时间戳或者说话人：每个段落换行
        if (!params.no_timestamps || params.diarize) {
            printf("\n");
        }
        // 刷新标准输出
        fflush(stdout);
    }
}

// 将语音识别结果输出为字符串
std::string output_str(struct whisper_context * ctx, const whisper_params & params, std::vector<std::vector<float>> pcmf32s) {
    // 创建一个字符串流用于存储结果
    std::stringstream result;
    // 获取语音识别结果的段数
    const int n_segments = whisper_full_n_segments(ctx);
    // 遍历每个识别结果段
    for (int i = 0; i < n_segments; ++i) {
        // 获取当前段的文本内容
        const char * text = whisper_full_get_segment_text(ctx, i);
        // 初始化说话者字符串
        std::string speaker = "";

        // 如果需要说话者分离并且输入的音频数据为两个声道
        if (params.diarize && pcmf32s.size() == 2)
        {
            // 获取当前段的起始和结束时间
            const int64_t t0 = whisper_full_get_segment_t0(ctx, i);
            const int64_t t1 = whisper_full_get_segment_t1(ctx, i);
            // 估计说话者
            speaker = estimate_diarization_speaker(pcmf32s, t0, t1);
        }

        // 将说话者和文本内容添加到结果中
        result << speaker << text << "\n";
    }
    // 返回结果字符串
    return result.str();
}

// 将字符串解析为布尔值
bool parse_str_to_bool(const std::string & s) {
    // 如果字符串表示为true、1、yes或y，则返回true，否则返回false
    if (s == "true" || s == "1" || s == "yes" || s == "y") {
        return true;
    }
    return false;
}

// 从请求中获取参数
void get_req_parameters(const Request & req, whisper_params & params)
{
    // 如果请求中包含"offset_t"参数，则将其转换为整数并存储到params中
    if (req.has_file("offset_t"))
    {
        params.offset_t_ms = std::stoi(req.get_file_value("offset_t").content);
    }
    // 如果请求中包含"offset_n"参数，则将其转换为整数并存储到params中
    if (req.has_file("offset_n"))
    {
        params.offset_n = std::stoi(req.get_file_value("offset_n").content);
    }
    // 如果请求中包含"duration"参数，则将其转换为整数并存储到params中
    if (req.has_file("duration"))
    {
        params.duration_ms = std::stoi(req.get_file_value("duration").content);
    }
    // 如果请求中包含"max_context"参数，则将其转换为整数并存储到params中
    if (req.has_file("max_context"))
    {
        params.max_context = std::stoi(req.get_file_value("max_context").content);
    }
    // 如果请求中包含"max_len"参数，则将其转换为整数并存储到params中
    if (req.has_file("max_len"))
    {
        params.max_len = std::stoi(req.get_file_value("max_len").content);
    }
    // 如果请求中包含"best_of"参数，则将其转换为整数并存储到params中
    if (req.has_file("best_of"))
    {
        params.best_of = std::stoi(req.get_file_value("best_of").content);
    }
    // 如果请求中包含"beam_size"参数，则将其转换为整数并存储到params中
    if (req.has_file("beam_size"))
    {
        params.beam_size = std::stoi(req.get_file_value("beam_size").content);
    }
    // 如果请求中包含"word_thold"参数，则将其转换为浮点数并存储到params中
    if (req.has_file("word_thold"))
    {
        params.word_thold = std::stof(req.get_file_value("word_thold").content);
    }
    // 如果请求中包含"entropy_thold"参数，则继续处理
    {
        // 如果请求中包含"entropy_thold"文件，则将其内容转换为浮点数并赋值给params.entropy_thold
        params.entropy_thold = std::stof(req.get_file_value("entropy_thold").content);
    }
    // 如果请求中包含"logprob_thold"文件，则将其内容转换为浮点数并赋值给params.logprob_thold
    if (req.has_file("logprob_thold"))
    {
        params.logprob_thold = std::stof(req.get_file_value("logprob_thold").content);
    }
    // 如果请求中包含"debug_mode"文件，则将其内容解析为布尔值并赋值给params.debug_mode
    if (req.has_file("debug_mode"))
    {
        params.debug_mode = parse_str_to_bool(req.get_file_value("debug_mode").content);
    }
    // 如果请求中包含"translate"文件，则将其内容解析为布尔值并赋值给params.translate
    if (req.has_file("translate"))
    {
        params.translate = parse_str_to_bool(req.get_file_value("translate").content);
    }
    // 如果请求中包含"diarize"文件，则将其内容解析为布尔值并赋值给params.diarize
    if (req.has_file("diarize"))
    {
        params.diarize = parse_str_to_bool(req.get_file_value("diarize").content);
    }
    // 如果请求中包含"tinydiarize"文件，则将其内容解析为布尔值并赋值给params.tinydiarize
    if (req.has_file("tinydiarize"))
    {
        params.tinydiarize = parse_str_to_bool(req.get_file_value("tinydiarize").content);
    }
    // 如果请求中包含"split_on_word"文件，则将其内容解析为布尔值并赋值给params.split_on_word
    if (req.has_file("split_on_word"))
    {
        params.split_on_word = parse_str_to_bool(req.get_file_value("split_on_word").content);
    }
    // 如果请求中包含"no_timestamps"文件，则将其内容解析为布尔值并赋值给params.no_timestamps
    if (req.has_file("no_timestamps"))
    {
        params.no_timestamps = parse_str_to_bool(req.get_file_value("no_timestamps").content);
    }
    // 如果请求中包含"language"文件，则将其内容赋值给params.language
    if (req.has_file("language"))
    {
        params.language = req.get_file_value("language").content;
    }
    // 如果请求中包含"detect_language"文件，则将其内容解析为布尔值并赋值给params.detect_language
    if (req.has_file("detect_language"))
    {
        params.detect_language = parse_str_to_bool(req.get_file_value("detect_language").content);
    }
    // 如果请求中包含"prompt"文件，则将其内容赋值给params.prompt
    if (req.has_file("prompt"))
    {
        params.prompt = req.get_file_value("prompt").content;
    }
    // 如果请求中包含"response_format"文件，则将其内容赋值给params.response_format
    if (req.has_file("response_format"))
    {
        params.response_format = req.get_file_value("response_format").content;
    }
    // 如果请求中包含"temperature"文件，则将其内容转换为浮点数并赋值给params.temperature
    if (req.has_file("temperature"))
    {
        params.temperature = std::stof(req.get_file_value("temperature").content);
    }
    // 如果请求中包含"temperature_inc"文件，则将其内容转换为浮点数并赋值给params.temperature_inc
    if (req.has_file("temperature_inc"))
    {
        params.temperature_inc = std::stof(req.get_file_value("temperature_inc").content);
    }
}  // namespace

// 主函数
int main(int argc, char ** argv) {
    // 定义 whisper_params 和 server_params 对象
    whisper_params params;
    server_params sparams;

    // 创建互斥锁对象
    std::mutex whisper_mutex;

    // 解析命令行参数，如果解析失败则打印用法信息并返回错误码
    if (whisper_params_parse(argc, argv, params, sparams) == false) {
        whisper_print_usage(argc, argv, params, sparams);
        return 1;
    }

    // 检查语言参数是否合法，如果不合法则打印错误信息并返回错误码
    if (params.language != "auto" && whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params, sparams);
        exit(0);
    }

    // 检查是否同时使用了 --diarize 和 --tinydiarize 参数，如果是则打印错误信息并返回错误码
    if (params.diarize && params.tinydiarize) {
        fprintf(stderr, "error: cannot use both --diarize and --tinydiarize\n");
        whisper_print_usage(argc, argv, params, sparams);
        exit(0);
    }

    // 检查是否需要使用 ffmpeg 转换器，如果需要则检查其可用性
    if (sparams.ffmpeg_converter) {
        check_ffmpeg_availibility();
    }

    // 初始化 whisper 上下文参数
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;

    // 从文件初始化 whisper 上下文对象
    struct whisper_context * ctx = whisper_init_from_file_with_params(params.model.c_str(), cparams);

    // 如果初始化失败则打印错误信息并返回错误码
    if (ctx == nullptr) {
        fprintf(stderr, "error: failed to initialize whisper context\n");
        return 3;
    }

    // 初始化 openvino 编码器，对没有配置 OpenVINO 的 whisper.cpp 构建没有影响
    whisper_ctx_init_openvino_encoder(ctx, nullptr, params.openvino_encode_device.c_str(), nullptr);

    // 创建服务器对象
    Server svr;
    // 设置默认头部信息
    svr.set_default_headers({{"Server", "whisper.cpp"},
                             {"Access-Control-Allow-Origin", "*"},
                             {"Access-Control-Allow-Headers", "content-type"}});

    // 默认内容字符串
    std::string const default_content = R"(
    <html>
    <head>
        <title>Whisper.cpp Server</title>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width">
        <style>
        body {
            font-family: sans-serif;
        }
        form {
            display: flex;
            flex-direction: column;
            align-items: flex-start;
        }
        label {
            margin-bottom: 0.5rem;
        }
        input, select {
            margin-bottom: 1rem;
        }
        button {
            margin-top: 1rem;
        }
        </style>
    </head>
    <body>
        <h1>Whisper.cpp Server</h1>

        <h2>/inference</h2>
        <pre>
    curl 127.0.0.1:)" + std::to_string(sparams.port) + R"(/inference \
    -H "Content-Type: multipart/form-data" \
    -F file="@&lt;file-path&gt;" \  # 发送文件路径作为表单数据
    -F temperature="0.0" \  # 发送温度值作为表单数据
    -F temperature_inc="0.2" \  # 发送温度增量作为表单数据
    -F response_format="json"  # 发送响应格式作为表单数据
        </pre>

        <h2>/load</h2>
        <pre>
    curl 127.0.0.1:)" + std::to_string(sparams.port) + R"(/load \
    -H "Content-Type: multipart/form-data" \  # 设置请求头的内容类型为多部分表单数据
    // 设置模型文件路径的参数
    -F model="&lt;path-to-model-file&gt;"
        </pre>

        <div>
            <h2>Try it out</h2>
            // 创建一个表单，用于提交音频文件和其他参数
            <form action="/inference" method="POST" enctype="multipart/form-data">
                // 选择音频文件的输入框
                <label for="file">Choose an audio file:</label>
                <input type="file" id="file" name="file" accept="audio/*" required><br>

                // 温度参数输入框
                <label for="temperature">Temperature:</label>
                <input type="number" id="temperature" name="temperature" value="0.0" step="0.01" placeholder="e.g., 0.0"><br>

                // 响应格式选择框
                <label for="response_format">Response Format:</label>
                <select id="response_format" name="response_format">
                    <option value="verbose_json">Verbose JSON</option>
                    <option value="json">JSON</option>
                    <option value="text">Text</option>
                    <option value="srt">SRT</option>
                    <option value="vtt">VTT</option>
                </select><br>

                // 提交按钮
                <button type="submit">Submit</button>
            </form>
        </div>
    </body>
    </html>
    )";

    // 存储默认参数，以便在每次推理请求后重置
    whisper_params default_params = params;

    // 如果在公共路径中找不到 index.html，则调用此函数
    svr.Get(sparams.request_path + "/", [&default_content](const Request &, Response &res){
        // 返回默认内容作为响应
        res.set_content(default_content, "text/html");
        return false;
    });

    });
    // 使用 POST 方法发送请求到指定路径，并设置回调函数处理请求
    svr.Post(sparams.request_path + "/load", [&](const Request &req, Response &res){
        // 使用互斥锁保护临界区，确保线程安全
        std::lock_guard<std::mutex> lock(whisper_mutex);
        // 检查请求中是否包含名为"model"的文件
        if (!req.has_file("model"))
        {
            // 如果请求中没有'model'字段，输出错误信息并返回错误响应
            fprintf(stderr, "error: no 'model' field in the request\n");
            const std::string error_resp = "{\"error\":\"no 'model' field in the request\"}";
            res.set_content(error_resp, "application/json");
            return;
        }
        // 获取请求中名为"model"的文件内容
        std::string model = req.get_file_value("model").content;
        // 检查文件是否存在
        if (!is_file_exist(model.c_str()))
        {
            // 如果文件不存在，输出错误信息并返回错误响应
            fprintf(stderr, "error: 'model': %s not found!\n", model.c_str());
            const std::string error_resp = "{\"error\":\"model not found!\"}";
            res.set_content(error_resp, "application/json");
            return;
        }

        // 清理资源
        whisper_free(ctx);

        // 从文件初始化 whisper 上下文
        ctx = whisper_init_from_file_with_params(model.c_str(), cparams);

        // TODO 可能在此处加载先前的模型，而不是退出
        if (ctx == nullptr) {
            // 如果模型初始化失败，输出错误信息并退出程序
            fprintf(stderr, "error: model init  failed, no model loaded must exit\n");
            exit(1);
        }

        // 初始化 openvino 编码器，对没有配置 OpenVINO 的 whisper.cpp 构建没有影响
        whisper_ctx_init_openvino_encoder(ctx, nullptr, params.openvino_encode_device.c_str(), nullptr);

        // 设置成功加载模型的响应
        const std::string success = "Load was successful!";
        res.set_content(success, "application/text");

        // 检查模型是否在文件系统中
    });
    // 设置异常处理程序，当出现异常时返回500错误
    svr.set_exception_handler([](const Request &, Response &res, std::exception_ptr ep) {
        const char fmt[] = "500 Internal Server Error\n%s";
        char buf[BUFSIZ];
        try {
            // 重新抛出异常并捕获
            std::rethrow_exception(std::move(ep));
        } catch (std::exception &e) {
            // 格式化异常信息到缓冲区
            snprintf(buf, sizeof(buf), fmt, e.what());
        } catch (...) {
            snprintf(buf, sizeof(buf), fmt, "Unknown Exception");
        }
        // 设置响应内容和类型
        res.set_content(buf, "text/plain");
        res.status = 500;
    });

    // 设置错误处理程序，根据不同状态码返回不同的响应内容
    svr.set_error_handler([](const Request &req, Response &res) {
        if (res.status == 400) {
            res.set_content("Invalid request", "text/plain");
        } else if (res.status != 500) {
            res.set_content("File Not Found (" + req.path + ")", "text/plain");
            res.status = 404;
        }
    });

    // 设置读写超时时间和更改主机名和端口
    svr.set_read_timeout(sparams.read_timeout);
    svr.set_write_timeout(sparams.write_timeout);

    // 绑定主机名和端口
    if (!svr.bind_to_port(sparams.hostname, sparams.port))
    {
        fprintf(stderr, "\ncouldn't bind to server socket: hostname=%s port=%d\n\n",
                sparams.hostname.c_str(), sparams.port);
        return 1;
    }

    // 设置静态文件服务的基本目录
    svr.set_base_dir(sparams.public_path);

    // 打印服务器监听地址
    printf("\nwhisper server listening at http://%s:%d\n\n", sparams.hostname.c_str(), sparams.port);

    // 开始监听
    if (!svr.listen_after_bind())
    {
        return 1;
    }

    // 打印时间信息并释放资源
    whisper_print_timings(ctx);
    whisper_free(ctx);

    return 0;
# 闭合之前的代码块
```