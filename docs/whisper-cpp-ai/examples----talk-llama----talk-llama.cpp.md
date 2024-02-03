# `whisper.cpp\examples\talk-llama\talk-llama.cpp`

```cpp
// 与 AI 进行对话

#include "common-sdl.h" // 引入 SDL 公共头文件
#include "common.h" // 引入公共头文件
#include "whisper.h" // 引入 whisper 头文件
#include "llama.h" // 引入 llama 头文件

#include <cassert> // 断言库
#include <cstdio> // C 标准 I/O 库
#include <fstream> // 文件流库
#include <regex> // 正则表达式库
#include <string> // 字符串库
#include <thread> // 线程库
#include <vector> // 向量库
#include <regex> // 正则表达式库
#include <sstream> // 字符串流库

// 将文本标记化为 llama_token 结构的向量
std::vector<llama_token> llama_tokenize(struct llama_context * ctx, const std::string & text, bool add_bos) {
    auto * model = llama_get_model(ctx);

    // 计算标记的最大数量
    int n_tokens = text.length() + add_bos;
    std::vector<llama_token> result(n_tokens);
    // 调用 llama_tokenize 函数进行标记化
    n_tokens = llama_tokenize(model, text.data(), text.length(), result.data(), result.size(), add_bos, false);
    if (n_tokens < 0) {
        result.resize(-n_tokens);
        int check = llama_tokenize(model, text.data(), text.length(), result.data(), result.size(), add_bos, false);
        GGML_ASSERT(check == -n_tokens);
    } else {
        result.resize(n_tokens);
    }
    return result;
}

// 将 llama_token 转换为字符串片段
std::string llama_token_to_piece(const struct llama_context * ctx, llama_token token) {
    std::vector<char> result(8, 0);
    const int n_tokens = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
    if (n_tokens < 0) {
        result.resize(-n_tokens);
        int check = llama_token_to_piece(llama_get_model(ctx), token, result.data(), result.size());
        GGML_ASSERT(check == -n_tokens);
    } else {
        result.resize(n_tokens);
    }

    return std::string(result.data(), result.size());
}

// 命令行参数结构体
struct whisper_params {
    int32_t n_threads  = std::min(4, (int32_t) std::thread::hardware_concurrency()); // 线程数，默认为 4 或硬件并发数的最小值
    int32_t voice_ms   = 10000; // 语音持续时间，默认为 10000 毫秒
    int32_t capture_id = -1; // 捕获 ID，默认为 -1
    int32_t max_tokens = 32; // 最大标记数，默认为 32
    int32_t audio_ctx  = 0; // 音频上下文，默认为 0
    int32_t n_gpu_layers = 999; // GPU 层数，默认为 999

    float vad_thold  = 0.6f; // VAD 阈值，默认为 0.6
    float freq_thold = 100.0f; // 频率阈值，默认为 100.0

    bool speed_up       = false; // 是否加速，默认为 false
    bool translate      = false; // 是否翻译，默认为 false
    bool print_special  = false; // 是否打印特殊信息，默认为 false
    bool print_energy   = false; // 是否打印能量信息，默认为 false
    # 定义一个布尔变量，表示是否不使用时间戳
    bool no_timestamps  = true;
    # 定义一个布尔变量，表示是否不显示详细提示信息
    bool verbose_prompt = false;
    # 定义一个布尔变量，表示是否使用 GPU
    bool use_gpu        = true;

    # 定义一个字符串变量，表示人名为 "Georgi"
    std::string person      = "Georgi";
    # 定义一个字符串变量，表示机器人名为 "LLaMA"
    std::string bot_name    = "LLaMA";
    # 定义一个字符串变量，表示唤醒命令为空
    std::string wake_cmd    = "";
    # 定义一个字符串变量，表示听到确认的命令为空
    std::string heard_ok    = "";
    # 定义一个字符串变量，表示语言为英语
    std::string language    = "en";
    # 定义一个字符串变量，表示模型路径为 "models/ggml-base.en.bin"
    std::string model_wsp   = "models/ggml-base.en.bin";
    # 定义一个字符串变量，表示模型路径为 "models/ggml-llama-7B.bin"
    std::string model_llama = "models/ggml-llama-7B.bin";
    # 定义一个字符串变量，表示说话的命令路径为 "./examples/talk-llama/speak"
    std::string speak       = "./examples/talk-llama/speak";
    # 定义一个字符串变量，表示提示信息为空
    std::string prompt      = "";
    # 定义一个字符串变量，表示输出文件名为空
    std::string fname_out;
    # 定义一个字符串变量，表示会话状态保存/加载文件路径为空
    std::string path_session = "";       // path to file for saving/loading model eval state
// 打印程序的使用方法
void whisper_print_usage(int argc, char ** argv, const whisper_params & params);

// 解析命令行参数，填充参数结构体
bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    // 解析命令行参数
    // 返回解析结果
    return true;
}

// 打印程序的使用方法
void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params) {
    // 打印使用方法
    fprintf(stderr, "\n");
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    // 打印各种选项及其默认值
    fprintf(stderr, "  -h,       --help           [default] show this help message and exit\n");
    fprintf(stderr, "  -t N,     --threads N      [%-7d] number of threads to use during computation\n", params.n_threads);
    fprintf(stderr, "  -vms N,   --voice-ms N     [%-7d] voice duration in milliseconds\n",              params.voice_ms);
    fprintf(stderr, "  -c ID,    --capture ID     [%-7d] capture device ID\n",                           params.capture_id);
    fprintf(stderr, "  -mt N,    --max-tokens N   [%-7d] maximum number of tokens per audio chunk\n",    params.max_tokens);
    fprintf(stderr, "  -ac N,    --audio-ctx N    [%-7d] audio context size (0 - all)\n",                params.audio_ctx);
    fprintf(stderr, "  -ngl N,   --n-gpu-layers N [%-7d] number of layers to store in VRAM\n",           params.n_gpu_layers);
    fprintf(stderr, "  -vth N,   --vad-thold N    [%-7.2f] voice activity detection threshold\n",        params.vad_thold);
    fprintf(stderr, "  -fth N,   --freq-thold N   [%-7.2f] high-pass frequency cutoff\n",                params.freq_thold);
    fprintf(stderr, "  -su,      --speed-up       [%-7s] speed up audio by x2 (reduced accuracy)\n",     params.speed_up ? "true" : "false");
    fprintf(stderr, "  -tr,      --translate      [%-7s] translate from source language to english\n",   params.translate ? "true" : "false");
    fprintf(stderr, "  -ps,      --print-special  [%-7s] print special tokens\n",                        params.print_special ? "true" : "false");
}
    # 打印是否打印声音能量的选项
    fprintf(stderr, "  -pe,      --print-energy   [%-7s] print sound energy (for debugging)\n",          params.print_energy ? "true" : "false");
    # 打印是否在启动时打印提示信息的选项
    fprintf(stderr, "  -vp,      --verbose-prompt [%-7s] print prompt at start\n",                       params.verbose_prompt ? "true" : "false");
    # 打印是否禁用 GPU 的选项
    fprintf(stderr, "  -ng,      --no-gpu         [%-7s] disable GPU\n",                                 params.use_gpu ? "false" : "true");
    # 打印人名用于提示选择的选项
    fprintf(stderr, "  -p NAME,  --person NAME    [%-7s] person name (for prompt selection)\n",          params.person.c_str());
    # 打印机器人名称用于显示的选项
    fprintf(stderr, "  -bn NAME, --bot-name NAME  [%-7s] bot name (to display)\n",                       params.bot_name.c_str());
    # 打印唤醒命令用于监听的选项
    fprintf(stderr, "  -w TEXT,  --wake-command T [%-7s] wake-up command to listen for\n",               params.wake_cmd.c_str());
    # 打印在生成回复之前 TTS 说的内容的选项
    fprintf(stderr, "  -ho TEXT, --heard-ok TEXT  [%-7s] said by TTS before generating reply\n",         params.heard_ok.c_str());
    # 打印口语的选项
    fprintf(stderr, "  -l LANG,  --language LANG  [%-7s] spoken language\n",                             params.language.c_str());
    # 打印耳语模型文件的选项
    fprintf(stderr, "  -mw FILE, --model-whisper  [%-7s] whisper model file\n",                          params.model_wsp.c_str());
    # 打印 llama 模型文件的选项
    fprintf(stderr, "  -ml FILE, --model-llama    [%-7s] llama model file\n",                            params.model_llama.c_str());
    # 打印 TTS 命令的选项
    fprintf(stderr, "  -s FILE,  --speak TEXT     [%-7s] command for TTS\n",                             params.speak.c_str());
    # 打印自定义提示对话框文件的选项
    fprintf(stderr, "  --prompt-file FNAME        [%-7s] file with custom prompt to start dialog\n",     "");
    # 打印缓存模型状态的文件选项（可能很大！）
    fprintf(stderr, "  --session FNAME                   file to cache model state in (may be large!) (default: none)\n");
    # 打印文本输出文件名的选项
    fprintf(stderr, "  -f FNAME, --file FNAME     [%-7s] text output file name\n",                       params.fname_out.c_str());
    # 打印空行
    fprintf(stderr, "\n");
// 定义一个函数，将音频数据转录为文本
std::string transcribe(
        whisper_context * ctx, // 指向转录上下文的指针
        const whisper_params & params, // 转录参数
        const std::vector<float> & pcmf32, // 浮点型音频数据
        const std::string prompt_text, // 提示文本
        float & prob, // 概率
        int64_t & t_ms) { // 时间（毫秒）

    // 获取当前时间
    const auto t_start = std::chrono::high_resolution_clock::now();

    // 初始化概率和时间
    prob = 0.0f;
    t_ms = 0;

    // 初始化提示文本的令牌
    std::vector<whisper_token> prompt_tokens;

    // 设置默认的转录参数
    whisper_full_params wparams = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);

    // 调整提示文本令牌的大小
    prompt_tokens.resize(1024);
    prompt_tokens.resize(whisper_tokenize(ctx, prompt_text.c_str(), prompt_tokens.data(), prompt_tokens.size()));

    // 设置转录参数
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

    // 设置提示文本令牌
    wparams.prompt_tokens    = prompt_tokens.empty() ? nullptr : prompt_tokens.data();
    wparams.prompt_n_tokens  = prompt_tokens.empty() ? 0       : prompt_tokens.size();

    // 设置音频上下文和加速度
    wparams.audio_ctx        = params.audio_ctx;
    wparams.speed_up         = params.speed_up;

    // 如果转录失败，返回空字符串
    if (whisper_full(ctx, wparams, pcmf32.data(), pcmf32.size()) != 0) {
        return "";
    }

    // 初始化概率计数和结果字符串
    int prob_n = 0;
    std::string result;

    // 获取转录结果的段数
    const int n_segments = whisper_full_n_segments(ctx);
    for (int i = 0; i < n_segments; ++i) {
        // 获取每个段的文本
        const char * text = whisper_full_get_segment_text(ctx, i);

        // 将文本添加到结果字符串中
        result += text;

        // 获取每个段的令牌数，并计算概率
        const int n_tokens = whisper_full_n_tokens(ctx, i);
        for (int j = 0; j < n_tokens; ++j) {
            const auto token = whisper_full_get_token_data(ctx, i, j);

            prob += token.p;
            ++prob_n;
        }
    }

    // 如果有令牌，计算平均概率
    if (prob_n > 0) {
        prob /= prob_n;
    }
}
    // 获取当前时间点作为结束时间
    const auto t_end = std::chrono::high_resolution_clock::now();
    // 计算从开始时间到结束时间的时间间隔，并转换为毫秒
    t_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t_end - t_start).count();

    // 返回结果
    return result;
// 结束 main 函数
}

// 根据输入的文本返回单词列表
std::vector<std::string> get_words(const std::string &txt) {
    // 创建一个空的字符串向量用于存储单词
    std::vector<std::string> words;

    // 创建一个字符串流对象，用于从输入文本中提取单词
    std::istringstream iss(txt);
    std::string word;
    // 从字符串流中逐个读取单词并添加到单词列表中
    while (iss >> word) {
        words.push_back(word);
    }

    // 返回单词列表
    return words;
}

// 定义一个名为 k_prompt_whisper 的常量字符串，包含特定格式的对话提示
const std::string k_prompt_whisper = R"(A conversation with a person called {1}.)";

// 定义一个名为 k_prompt_llama 的常量字符串，包含特定格式的对话提示
const std::string k_prompt_llama = R"(Text transcript of a never ending dialog, where {0} interacts with an AI assistant named {1}.
{1} is helpful, kind, honest, friendly, good at writing and never fails to answer {0}’s requests immediately and with details and precision.
There are no annotations like (30 seconds passed...) or (to himself), just what {0} and {1} say aloud to each other.
The transcript only includes text, it does not include markup like HTML and Markdown.
{1} responds with short and concise answers.

{0}{4} Hello, {1}!
{1}{4} Hello {0}! How may I help you today?
{0}{4} What time is it?
{1}{4} It is {2} o'clock.
{0}{4} What year is it?
{1}{4} We are in {3}.
{0}{4} What is a cat?
{1}{4} A cat is a domestic species of small carnivorous mammal. It is the only domesticated species in the family Felidae.
{0}{4} Name a color.
{1}{4} Blue
{0}{4})";

// 主函数
int main(int argc, char ** argv) {
    // 初始化 whisper_params 结构
    whisper_params params;

    // 解析命令行参数，如果解析失败则返回 1
    if (whisper_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果指定了语言并且语言不是 "auto"，且语言 ID 为 -1，则输出错误信息并退出程序
    if (params.language != "auto" && whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // 初始化 whisper 上下文参数
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;

    // 从文件初始化 whisper 上下文
    struct whisper_context * ctx_wsp = whisper_init_from_file_with_params(params.model_wsp.c_str(), cparams);

    // 初始化 llama 后端
    llama_backend_init(true);

    // 获取默认的 llama 模型参数
    auto lmparams = llama_model_default_params();
    // 如果不使用 GPU，则将 GPU 层数设置为 0
    if (!params.use_gpu) {
        lmparams.n_gpu_layers = 0;
    } else {
        // 如果不是 GPU 模式，则使用参数中的 GPU 层数
        lmparams.n_gpu_layers = params.n_gpu_layers;
    }

    // 从文件加载 LLaMA 模型
    struct llama_model * model_llama = llama_load_model_from_file(params.model_llama.c_str(), lmparams);

    // 设置 llama 上下文参数为默认值
    llama_context_params lcparams = llama_context_default_params();

    // 调整这些参数以符合您的需求
    lcparams.n_ctx      = 2048;
    lcparams.seed       = 1;
    lcparams.n_threads  = params.n_threads;

    // 创建一个带有模型的 llama 上下文
    struct llama_context * ctx_llama = llama_new_context_with_model(model_llama, lcparams);

    // 打印一些有关处理的信息
    {
        fprintf(stderr, "\n");

        // 如果不是多语言模型，忽略语言和翻译选项
        if (!whisper_is_multilingual(ctx_wsp)) {
            if (params.language != "en" || params.translate) {
                params.language = "en";
                params.translate = false;
                fprintf(stderr, "%s: WARNING: model is not multilingual, ignoring language and translation options\n", __func__);
            }
        }
        // 打印处理信息，包括线程数、语言、任务和时间戳
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
    // 如果音频初始化失败，打印错误信息并返回
    if (!audio.init(params.capture_id, WHISPER_SAMPLE_RATE)) {
        fprintf(stderr, "%s: audio.init() failed!\n", __func__);
        return 1;
    }

    // 恢复音频
    audio.resume();

    bool is_running  = true;
    bool force_speak = false;

    float prob0 = 0.0f;

    const std::string chat_symb = ":";

    std::vector<float> pcmf32_cur;
    std::vector<float> pcmf32_prompt;

    // 构建用于 LLaMA 推理的初始提示
    const std::string prompt_whisper = ::replace(k_prompt_whisper, "{1}", params.bot_name);

    // 需要有前导空格
    prompt_llama.insert(0, 1, ' ');
    // 使用参数中的 person 替换 prompt_llama 中的 {0}
    prompt_llama = ::replace(prompt_llama, "{0}", params.person);
    // 使用参数中的 bot_name 替换 prompt_llama 中的 {1}
    prompt_llama = ::replace(prompt_llama, "{1}", params.bot_name);

    {
        // 获取当前时间的字符串表示
        std::string time_str;
        {
            // 获取当前时间
            time_t t = time(0);
            // 将时间转换为本地时间
            struct tm * now = localtime(&t);
            char buf[128];
            // 格式化时间为 HH:MM 格式
            strftime(buf, sizeof(buf), "%H:%M", now);
            time_str = buf;
        }
        // 使用当前时间字符串替换 prompt_llama 中的 {2}
        prompt_llama = ::replace(prompt_llama, "{2}", time_str);
    }

    {
        // 获取当前年份的字符串表示
        std::string year_str;
        {
            // 获取当前时间
            time_t t = time(0);
            // 将时间转换为本地时间
            struct tm * now = localtime(&t);
            char buf[128];
            // 格式化时间为年份格式
            strftime(buf, sizeof(buf), "%Y", now);
            year_str = buf;
        }
        // 使用当前年份字符串替换 prompt_llama 中的 {3}
        prompt_llama = ::replace(prompt_llama, "{3}", year_str);
    }

    // 使用 chat_symb 替换 prompt_llama 中的 {4}
    prompt_llama = ::replace(prompt_llama, "{4}", chat_symb);

    // 初始化会话
    std::string path_session = params.path_session;
    std::vector<llama_token> session_tokens;
    // 对 prompt_llama 进行分词处理
    auto embd_inp = ::llama_tokenize(ctx_llama, prompt_llama, true);
    // 如果路径不为空
    if (!path_session.empty()) {
        // 打印尝试从指定路径加载保存的会话信息的消息
        fprintf(stderr, "%s: attempting to load saved session from %s\n", __func__, path_session.c_str());

        // 使用 fopen 检查是否存在会话文件
        FILE * fp = std::fopen(path_session.c_str(), "rb");
        // 如果文件存在
        if (fp != NULL) {
            std::fclose(fp);

            // 调整 session_tokens 的大小以适应上下文 llama_n_ctx(ctx_llama)
            session_tokens.resize(llama_n_ctx(ctx_llama));
            size_t n_token_count_out = 0;
            // 如果无法加载会话文件
            if (!llama_load_session_file(ctx_llama, path_session.c_str(), session_tokens.data(), session_tokens.capacity(), &n_token_count_out)) {
                // 打印加载会话文件失败的错误消息
                fprintf(stderr, "%s: error: failed to load session file '%s'\n", __func__, path_session.c_str());
                return 1;
            }
            // 调整 session_tokens 的大小以匹配加载的令牌数量
            session_tokens.resize(n_token_count_out);
            // 将加载的令牌复制到 embd_inp 中
            for (size_t i = 0; i < session_tokens.size(); i++) {
                embd_inp[i] = session_tokens[i];
            }

            // 打印加载会话的提示大小
            fprintf(stderr, "%s: loaded a session with prompt size of %d tokens\n", __func__, (int) session_tokens.size());
        } else {
            // 打印会话文件不存在的消息，将创建新的会话文件
            fprintf(stderr, "%s: session file does not exist, will create\n", __func__);
        }
    }

    // 评估初始提示

    // 打印初始化消息
    printf("\n");
    printf("%s : initializing - please wait ...\n", __func__);

    // 如果 llama_eval 返回错误
    if (llama_eval(ctx_llama, embd_inp.data(), embd_inp.size(), 0)) {
        // 打印评估失败的消息
        fprintf(stderr, "%s : failed to eval\n", __func__);
        return 1;
    }

    // 如果参数中包含详细提示信息
    if (params.verbose_prompt) {
        // 打印详细提示信息
        fprintf(stdout, "\n");
        fprintf(stdout, "%s", prompt_llama.c_str());
        fflush(stdout);
    }

    // 调试消息，关于保存的会话的相似性，如果适用
    size_t n_matching_session_tokens = 0;
    // 如果会话令牌集合非空
    if (session_tokens.size()) {
        // 遍历会话令牌集合
        for (llama_token id : session_tokens) {
            // 如果匹配的会话令牌数量大于等于输入嵌入的大小，或者当前令牌不等于输入嵌入中的令牌，则跳出循环
            if (n_matching_session_tokens >= embd_inp.size() || id != embd_inp[n_matching_session_tokens]) {
                break;
            }
            // 匹配的会话令牌数量加一
            n_matching_session_tokens++;
        }
        // 如果匹配的会话令牌数量大于等于输入嵌入的大小
        if (n_matching_session_tokens >= embd_inp.size()) {
            // 输出精确匹配的提示信息
            fprintf(stderr, "%s: session file has exact match for prompt!\n", __func__);
        } 
        // 如果匹配的会话令牌数量小于输入嵌入大小的一半
        else if (n_matching_session_tokens < (embd_inp.size() / 2)) {
            // 输出低相似度警告信息
            fprintf(stderr, "%s: warning: session file has low similarity to prompt (%zu / %zu tokens); will mostly be reevaluated\n",
                __func__, n_matching_session_tokens, embd_inp.size());
        } 
        // 其他情况
        else {
            // 输出匹配的令牌数量信息
            fprintf(stderr, "%s: session file matches %zu / %zu tokens of prompt\n",
                __func__, n_matching_session_tokens, embd_inp.size());
        }
    }

    // HACK - 因为保存会话会导致非常明显的延迟，所以如果加载的会话与提示有至少75%的相似度，则暂时跳过重新保存会话。
    // 目前仅用于加快初始提示，因此不需要完全匹配。
    // 判断是否需要保存会话
    bool need_to_save_session = !path_session.empty() && n_matching_session_tokens < (embd_inp.size() * 3 / 4);

    // 输出提示信息
    printf("%s : done! start speaking in the microphone\n", __func__);

    // 如果启用唤醒命令
    const std::string wake_cmd = params.wake_cmd;
    const int wake_cmd_length = get_words(wake_cmd).size();
    const bool use_wake_cmd = wake_cmd_length > 0;

    // 如果使用唤醒命令
    if (use_wake_cmd) {
        // 输出唤醒命令信息
        printf("%s : the wake-up command is: '%s%s%s'\n", __func__, "\033[1m", wake_cmd.c_str(), "\033[0m");
    }

    // 输出人物名称和聊天符号
    printf("\n");
    printf("%s%s", params.person.c_str(), chat_symb.c_str());
    fflush(stdout);

    // 清空音频缓冲区
    audio.clear();

    // 文本推理变量
    const int voice_id = 2;
    const int n_keep   = embd_inp.size();
    const int n_ctx    = llama_n_ctx(ctx_llama);
    // 将 n_keep 的值赋给 n_past
    int n_past = n_keep;
    // 初始化 n_prev 为 64，TODO: 根据参数设置
    int n_prev = 64;
    // 如果 path_session 不为空且 session_tokens 的大小大于0，则将 n_session_consumed 设置为 session_tokens 的大小，否则为0
    int n_session_consumed = !path_session.empty() && session_tokens.size() > 0 ? session_tokens.size() : 0;

    // 创建一个存储 llama_token 结构的向量 embd
    std::vector<llama_token> embd;

    // 反转提示语以便检测何时停止发言
    std::vector<std::string> antiprompts = {
        params.person + chat_symb,
    };

    // 主循环
    }

    // 暂停音频
    audio.pause();

    // 打印 whisper 上下文的时间信息
    whisper_print_timings(ctx_wsp);
    // 释放 whisper 上下文
    whisper_free(ctx_wsp);

    // 打印 llama 上下文的时间信息
    llama_print_timings(ctx_llama);
    // 释放 llama 上下文
    llama_free(ctx_llama);

    // 返回0表示正常结束
    return 0;
# 闭合之前的代码块
```