# `whisper.cpp\examples\command\command.cpp`

```cpp
// 检查文件是否存在
bool file_exists(const std::string & fname) {
    // 打开文件流
    std::ifstream f(fname.c_str());
    // 返回文件流状态
    return f.good();
}

// 定义语音助手的参数结构体
struct whisper_params {
    // 线程数，默认为不超过硬件并发数的4
    int32_t n_threads  = std::min(4, (int32_t) std::thread::hardware_concurrency());
    // 提示时间（毫秒），默认为5000
    int32_t prompt_ms  = 5000;
    // 命令时间（毫秒），默认为8000
    int32_t command_ms = 8000;
    // 捕获 ID，默认为-1
    int32_t capture_id = -1;
    // 最大令牌数，默认为32
    int32_t max_tokens = 32;
    // 音频上下文，默认为0
    int32_t audio_ctx  = 0;

    // VAD 阈值，默认为0.6
    float vad_thold  = 0.6f;
    // 频率阈值，默认为100.0
    float freq_thold = 100.0f;

    // 语法惩罚，默认为100.0
    float grammar_penalty = 100.0f;

    // 语法解析状态
    grammar_parser::parse_state grammar_parsed;

    // 是否加速，默认为 false
    bool speed_up      = false;
    // 是否翻译，默认为 false
    bool translate     = false;
    // 是否打印特殊信息，默认为 false
    bool print_special = false;
    // 是否打印能量信息，默认为 false
    bool print_energy  = false;
    // 是否不显示时间戳，默认为 true
    bool no_timestamps = true;
    // 是否使用 GPU，默认为 true
    bool use_gpu       = true;

    // 语言，默认为 "en"
    std::string language  = "en";
    // 模型，默认为 "models/ggml-base.en.bin"
    std::string model     = "models/ggml-base.en.bin";
    // 输出文件名
    std::string fname_out;
    // 命令
    std::string commands;
    // 提示
    std::string prompt;
    // 上下文
    std::string context;
    // 语法
    std::string grammar;
};

// 解析命令行参数
bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    // 参数解析逻辑
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
    fprintf(stderr, "  -h,         --help           [default] show this help message and exit\n");
}
    # 打印线程数的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -t N,       --threads N      [%-7d] number of threads to use during computation\n", params.n_threads);
    # 打印提示持续时间的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -pms N,     --prompt-ms N    [%-7d] prompt duration in milliseconds\n",             params.prompt_ms);
    # 打印命令持续时间的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -cms N,     --command-ms N   [%-7d] command duration in milliseconds\n",            params.command_ms);
    # 打印捕获设备 ID 的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -c ID,      --capture ID     [%-7d] capture device ID\n",                           params.capture_id);
    # 打印每个音频块中最大令牌数的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -mt N,      --max-tokens N   [%-7d] maximum number of tokens per audio chunk\n",    params.max_tokens);
    # 打印音频上下文大小的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -ac N,      --audio-ctx N    [%-7d] audio context size (0 - all)\n",                params.audio_ctx);
    # 打印语音活动检测阈值的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -vth N,     --vad-thold N    [%-7.2f] voice activity detection threshold\n",        params.vad_thold);
    # 打印高通滤波器频率截止值的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -fth N,     --freq-thold N   [%-7.2f] high-pass frequency cutoff\n",                params.freq_thold);
    # 打印加速音频的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -su,        --speed-up       [%-7s] speed up audio by x2 (reduced accuracy)\n",     params.speed_up ? "true" : "false");
    # 打印翻译源语言到英语的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -tr,        --translate      [%-7s] translate from source language to english\n",   params.translate ? "true" : "false");
    # 打印特殊令牌的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -ps,        --print-special  [%-7s] print special tokens\n",                        params.print_special ? "true" : "false");
    # 打印声音能量的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -pe,        --print-energy   [%-7s] print sound energy (for debugging)\n",          params.print_energy ? "true" : "false");
    # 打印是否禁用 GPU 的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -ng,        --no-gpu         [%-7s] disable GPU\n",                                 params.use_gpu ? "false" : "true");
    # 打印语言设置的帮助信息，包括默认值和用户设置值
    fprintf(stderr, "  -l LANG,    --language LANG  [%-7s] spoken language\n",                             params.language.c_str());
    # 输出模型路径的帮助信息，包括参数选项和默认值
    fprintf(stderr, "  -m FNAME,   --model FNAME    [%-7s] model path\n",                                  params.model.c_str());
    # 输出文本输出文件名的帮助信息，包括参数选项和默认值
    fprintf(stderr, "  -f FNAME,   --file FNAME     [%-7s] text output file name\n",                       params.fname_out.c_str());
    # 输出包含允许命令的文本文件名的帮助信息，包括参数选项和默认值
    fprintf(stderr, "  -cmd FNAME, --commands FNAME [%-7s] text file with allowed commands\n",             params.commands.c_str());
    # 输出所需激活提示的帮助信息，包括参数选项和默认值
    fprintf(stderr, "  -p,         --prompt         [%-7s] the required activation prompt\n",              params.prompt.c_str());
    # 输出用于帮助转录的示例文本的帮助信息，包括参数选项和默认值
    fprintf(stderr, "  -ctx,       --context        [%-7s] sample text to help the transcription\n",       params.context.c_str());
    # 输出用于引导解码的 GBNF 语法的帮助信息，包括参数选项和默认值
    fprintf(stderr, "  --grammar GRAMMAR            [%-7s] GBNF grammar to guide decoding\n",              params.grammar.c_str());
    # 输出非语法标记的 logits 缩放因子的帮助信息，包括参数选项和默认值
    fprintf(stderr, "  --grammar-penalty N          [%-7.1f] scales down logits of nongrammar tokens\n",   params.grammar_penalty);
    # 输出空行
    fprintf(stderr, "\n");
// 定义一个函数，将音频数据转录为文本
std::string transcribe(
                 whisper_context * ctx, // 指向上下文的指针
            const whisper_params & params, // 包含转录参数的结构体引用
        const std::vector<float> & pcmf32, // 包含音频数据的浮点数向量引用
               const std::string & grammar_rule, // 包含语法规则的字符串引用
                           float & logprob_min, // 最小对数概率的引用
                           float & logprob_sum, // 总对数概率的引用
                             int & n_tokens, // 标记数量的引用
                         int64_t & t_ms) { // 毫秒时间的引用

    // 获取当前时间点
    const auto t_start = std::chrono::high_resolution_clock::now();

    // 初始化对数概率、标记数量和时间为零
    logprob_min = 0.0f;
    logprob_sum = 0.0f;
    n_tokens    = 0;
    t_ms = 0;

    // 使用贪婪搜索初始化转录参数
    //whisper_full_params wparams = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);
    whisper_full_params wparams = whisper_full_default_params(WHISPER_SAMPLING_BEAM_SEARCH);

    // 设置参数
    wparams.print_progress   = false;
    wparams.print_special    = params.print_special;
    wparams.print_realtime   = false;
    wparams.print_timestamps = !params.no_timestamps;
    wparams.translate        = params.translate;
    wparams.no_context       = true;
    wparams.no_timestamps    = params.no_timestamps;
    wparams.single_segment   = true;
    wparams.max_tokens       = params.max_tokens;
    wparams.language         = params.language.c_str();
    wparams.n_threads        = params.n_threads;

    wparams.audio_ctx = params.audio_ctx;
    wparams.speed_up  = params.speed_up;

    wparams.temperature     = 0.4f;
    wparams.temperature_inc = 1.0f;
    wparams.greedy.best_of  = 5;

    wparams.beam_search.beam_size = 5;

    wparams.initial_prompt = params.context.data();

    // 获取语法解析后的规则
    const auto & grammar_parsed = params.grammar_parsed;
    auto grammar_rules = grammar_parsed.c_rules();
    // 如果语法解析后的规则不为空且给定的语法规则不为空
    if (!params.grammar_parsed.rules.empty() && !grammar_rule.empty()) {
        // 如果给定的语法规则在语法解析后的符号 ID 中找不到
        if (grammar_parsed.symbol_ids.find(grammar_rule) == grammar_parsed.symbol_ids.end()) {
            // 输出警告信息，提示找不到对应的语法规则
            fprintf(stderr, "%s: warning: grammar rule '%s' not found - skipping grammar sampling\n", __func__, grammar_rule.c_str());
        } else {
            // 设置参数中的语法规则、语法规则数量、起始规则 ID 和语法惩罚值
            wparams.grammar_rules   = grammar_rules.data();
            wparams.n_grammar_rules = grammar_rules.size();
            wparams.i_start_rule    = grammar_parsed.symbol_ids.at(grammar_rule);
            wparams.grammar_penalty = params.grammar_penalty;
        }
    }

    // 调用 whisper_full 函数进行语音合成
    if (whisper_full(ctx, wparams, pcmf32.data(), pcmf32.size()) != 0) {
        // 如果语音合成失败，返回空字符串
        return "";
    }

    // 初始化结果字符串
    std::string result;

    // 获取 whisper_full 函数返回的语音段数
    const int n_segments = whisper_full_n_segments(ctx);
    // 遍历每个语音段
    for (int i = 0; i < n_segments; ++i) {
        // 获取当前语音段的文本内容
        const char * text = whisper_full_get_segment_text(ctx, i);

        // 将当前语音段的文本内容添加到结果字符串中
        result += text;

        // 获取当前语音段的 token 数量
        const int n = whisper_full_n_tokens(ctx, i);
        // 遍历当前语音段的每个 token
        for (int j = 0; j < n; ++j) {
            // 获取当前 token 的数据
            const auto token = whisper_full_get_token_data(ctx, i, j);

            // 如果当前 token 的概率大于 0，退出程序
            if(token.plog > 0.0f) exit(0);
            // 更新最小概率和概率总和，并增加 token 数量
            logprob_min = std::min(logprob_min, token.plog);
            logprob_sum += token.plog;
            ++n_tokens;
        }
    }

    // 计算程序运行时间并存储为毫秒
    const auto t_end = std::chrono::high_resolution_clock::now();
    t_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t_end - t_start).count();

    // 返回合成的结果字符串
    return result;
// 读取指定文件中的允许命令列表，并返回一个字符串向量
std::vector<std::string> read_allowed_commands(const std::string & fname) {
    // 创建一个空的字符串向量用于存储允许的命令
    std::vector<std::string> allowed_commands;

    // 打开指定文件
    std::ifstream ifs(fname);
    // 如果文件无法打开，则返回空的命令向量
    if (!ifs.is_open()) {
        return allowed_commands;
    }

    // 逐行读取文件内容
    std::string line;
    while (std::getline(ifs, line)) {
        // 去除行首尾的空格
        line = ::trim(line);
        // 如果行为空，则继续下一行
        if (line.empty()) {
            continue;
        }

        // 将行中的字符转换为小写
        std::transform(line.begin(), line.end(),line.begin(), ::tolower);
        // 将处理后的行添加到允许命令向量中
        allowed_commands.push_back(std::move(line));
    }

    // 返回允许命令向量
    return allowed_commands;
}

// 将输入文本分割为单词，并返回一个字符串向量
std::vector<std::string> get_words(const std::string &txt) {
    // 创建一个空的字符串向量用于存储单词
    std::vector<std::string> words;

    // 将输入文本转换为字符串流
    std::istringstream iss(txt);
    std::string word;
    // 逐个读取单词并添加到单词向量中
    while (iss >> word) {
        words.push_back(word);
    }

    // 返回单词向量
    return words;
}

// 命令列表模式
// 引导转录以匹配提供的命令列表中最可能的命令
int process_command_list(struct whisper_context * ctx, audio_async &audio, const whisper_params &params) {
    // 输出提示信息
    fprintf(stderr, "\n");
    fprintf(stderr, "%s: guided mode\n", __func__);

    // 读取允许的命令列表
    std::vector<std::string> allowed_commands = read_allowed_commands(params.commands);

    // 如果允许的命令列表为空，则输出错误信息并返回错误码
    if (allowed_commands.empty()) {
        fprintf(stderr, "%s: error: failed to read allowed commands from '%s'\n", __func__, params.commands.c_str());
        return 2;
    }

    // 初始化最大长度为0
    int max_len = 0;

    // 创建一个二维向量用于存储允许的令牌
    std::vector<std::vector<whisper_token>> allowed_tokens;
}
    // 遍历允许的命令列表
    for (const auto & cmd : allowed_commands) {
        // 创建一个大小为1024的令牌数组
        whisper_token tokens[1024];
        // 将一个空的令牌数组添加到允许的令牌列表中
        allowed_tokens.emplace_back();

        // 遍历当前命令的长度
        for (int l = 0; l < (int) cmd.size(); ++l) {
            // 创建一个带有空格的子字符串
            // 注意：非常重要要添加空格！
            // 原因是第一个解码的令牌也以空格开头！
            std::string ss = std::string(" ") + cmd.substr(0, l + 1);

            // 对子字符串进行令牌化
            const int n = whisper_tokenize(ctx, ss.c_str(), tokens, 1024);
            // 如果令牌化失败
            if (n < 0) {
                fprintf(stderr, "%s: error: failed to tokenize command '%s'\n", __func__, cmd.c_str());
                return 3;
            }

            // 如果令牌化成功
            if (n == 1) {
                // 将令牌添加到当前命令的令牌列表中
                allowed_tokens.back().push_back(tokens[0]);
            }
        }

        // 更新最大命令长度
        max_len = std::max(max_len, (int) cmd.size());
    }

    // 打印允许的命令及其令牌
    fprintf(stderr, "%s: allowed commands [ tokens ]:\n", __func__);
    fprintf(stderr, "\n");
    for (int i = 0; i < (int) allowed_commands.size(); ++i) {
        fprintf(stderr, "  - \033[1m%-*s\033[0m = [", max_len, allowed_commands[i].c_str());
        for (const auto & token : allowed_tokens[i]) {
            fprintf(stderr, " %5d", token);
        }
        fprintf(stderr, " ]\n");
    }

    // 创建选择提示字符串
    std::string k_prompt = "select one from the available words: ";
    for (int i = 0; i < (int) allowed_commands.size(); ++i) {
        if (i > 0) {
            k_prompt += ", ";
        }
        k_prompt += allowed_commands[i];
    }
    k_prompt += ". selected word: ";

    // 令牌化提示字符串
    std::vector<whisper_token> k_tokens;
    {
        k_tokens.resize(1024);
        const int n = whisper_tokenize(ctx, k_prompt.c_str(), k_tokens.data(), 1024);
        // 如果令牌化失败
        if (n < 0) {
            fprintf(stderr, "%s: error: failed to tokenize prompt '%s'\n", __func__, k_prompt.c_str());
            return 4;
        }
        k_tokens.resize(n);
    }

    // 打印提示字符串
    fprintf(stderr, "\n");
    fprintf(stderr, "%s: prompt: '%s'\n", __func__, k_prompt.c_str());
    // 输出函数名和 tokens 数组内容到标准错误流
    fprintf(stderr, "%s: tokens: [", __func__);
    for (const auto & token : k_tokens) {
        fprintf(stderr, " %d", token);
    }
    fprintf(stderr, " ]\n");

    // 输出空行到标准错误流
    fprintf(stderr, "\n");
    // 输出函数名和提示信息到标准错误流
    fprintf(stderr, "%s: listening for a command ...\n", __func__);
    // 输出空行到标准错误流
    fprintf(stderr, "\n");

    // 初始化一个布尔变量 is_running 为 true
    bool is_running  = true;

    // 初始化两个空的 float 向量 pcmf32_cur 和 pcmf32_prompt
    std::vector<float> pcmf32_cur;
    std::vector<float> pcmf32_prompt;

    // 主循环
    }

    // 返回 0
    return 0;
// 始终提示模式
// 在有效提示后将语音转录为文本
int always_prompt_transcription(struct whisper_context * ctx, audio_async & audio, const whisper_params & params) {
    // 标记程序是否正在运行
    bool is_running = true;
    // 标记是否需要提示
    bool ask_prompt = true;

    // 最小对数概率
    float logprob_min = 0.0f;
    // 对数概率总和
    float logprob_sum = 0.0f;
    // 标记 token 数量
    int n_tokens = 0;

    // 当前 PCM 浮点数向量
    std::vector<float> pcmf32_cur;

    // 提示文本
    const std::string k_prompt = params.prompt;

    // 提示文本的长度
    const int k_prompt_length = get_words(k_prompt).size();

    // 输出提示信息
    fprintf(stderr, "\n");
    fprintf(stderr, "%s: always-prompt mode\n", __func__);

    // 主循环
    // 当程序正在运行时执行循环
    while (is_running) {
        // 处理 Ctrl + C 信号
        is_running = sdl_poll_events();

        // 延迟执行
        std::this_thread::sleep_for(std::chrono::milliseconds(100));

        // 如果需要提示用户输入
        if (ask_prompt) {
            // 输出换行符
            fprintf(stdout, "\n");
            // 输出提示信息
            fprintf(stdout, "%s: The prompt is: '%s%s%s'\n", __func__, "\033[1m", k_prompt.c_str(), "\033[0m");
            // 输出换行符
            fprintf(stdout, "\n");

            // 将 ask_prompt 置为 false
            ask_prompt = false;
        }

        {
            // 获取音频数据
            audio.get(2000, pcmf32_cur);

            // 简单的语音活动检测
            if (::vad_simple(pcmf32_cur, WHISPER_SAMPLE_RATE, 1000, params.vad_thold, params.freq_thold, params.print_energy)) {
                // 输出检测到语音的信息
                fprintf(stdout, "%s: Speech detected! Processing ...\n", __func__);

                int64_t t_ms = 0;

                // 检测命令
                audio.get(params.command_ms, pcmf32_cur);

                // 转录音频数据
                const auto txt = ::trim(::transcribe(ctx, params, pcmf32_cur, "", logprob_min, logprob_sum, n_tokens, t_ms));

                // 获取单词列表
                const auto words = get_words(txt);

                std::string prompt;
                std::string command;

                // 将单词分为提示和命令
                for (int i = 0; i < (int) words.size(); ++i) {
                    if (i < k_prompt_length) {
                        prompt += words[i] + " ";
                    } else {
                        command += words[i] + " ";
                    }
                }

                // 计算提示与预设提示的相似度
                const float sim = similarity(prompt, k_prompt);

                // 如果相似度大于 0.7 并且命令不为空
                if ((sim > 0.7f) && (command.size() > 0)) {
                    // 输出识别到的命令信息
                    fprintf(stdout, "%s: Command '%s%s%s', (t = %d ms)\n", __func__, "\033[1m", command.c_str(), "\033[0m", (int) t_ms);
                }

                // 输出换行符
                fprintf(stdout, "\n");

                // 清空音频数据
                audio.clear();
            }
        }
    }

    // 返回 0
    return 0;
// 通用模式
// 自由地将语音转录为文本
int process_general_transcription(struct whisper_context * ctx, audio_async & audio, const whisper_params & params) {
    // 标记是否正在运行
    bool is_running  = true;
    // 标记是否有提示
    bool have_prompt = false;
    // 标记是否需要提示
    bool ask_prompt  = true;

    // 初始最小对数概率
    float logprob_min0 = 0.0f;
    // 最小对数概率
    float logprob_min  = 0.0f;

    // 初始对数概率总和
    float logprob_sum0 = 0.0f;
    // 对数概率总和
    float logprob_sum  = 0.0f;

    // 初始标记数
    int n_tokens0 = 0;
    // 标记数
    int n_tokens  = 0;

    // 当前 PCM 浮点数向量
    std::vector<float> pcmf32_cur;
    // 提示 PCM 浮点数向量
    std::vector<float> pcmf32_prompt;

    // 默认提示语句
    std::string k_prompt = "Ok Whisper, start listening for commands.";
    // 如果参数中有自定义提示语句，则使用自定义提示语句
    if (!params.prompt.empty()) {
        k_prompt = params.prompt;
    }

    // 输出空行
    fprintf(stderr, "\n");
    // 输出函数名和模式信息
    fprintf(stderr, "%s: general-purpose mode\n", __func__);

    // 主循环
    }

    // 返回 0 表示成功
    return 0;
}

// 主函数
int main(int argc, char ** argv) {
    // 初始化参数对象
    whisper_params params;

    // 解析命令行参数，如果解析失败则返回 1
    if (whisper_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 检查语言是否合法，如果不合法则输出错误信息并退出程序
    if (whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // 初始化 Whisper 上下文
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;

    // 从文件初始化 Whisper 上下文
    struct whisper_context * ctx = whisper_init_from_file_with_params(params.model.c_str(), cparams);

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
        // 输出处理信息，包括线程数、语言、任务类型和时间戳选项
        fprintf(stderr, "%s: processing, %d threads, lang = %s, task = %s, timestamps = %d ...\n",
                __func__,
                params.n_threads,
                params.language.c_str(),
                params.translate ? "translate" : "transcribe",
                params.no_timestamps ? 0 : 1);

        // 输出一个换行符
        fprintf(stderr, "\n");
    }

    // 初始化音频

    // 创建一个异步音频对象，设置缓冲时间为 30 秒
    audio_async audio(30*1000);
    // 初始化音频对象，设置捕获 ID 和采样率
    if (!audio.init(params.capture_id, WHISPER_SAMPLE_RATE)) {
        // 输出初始化失败信息
        fprintf(stderr, "%s: audio.init() failed!\n", __func__);
        return 1;
    }

    // 恢复音频
    audio.resume();

    // 等待 1 秒以避免任何缓冲噪音
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    // 清除音频缓冲
    audio.clear();

    // 初始化返回值为 0
    int  ret_val = 0;

    // 如果语法不为空
    if (!params.grammar.empty()) {
        auto & grammar = params.grammar_parsed;
        // 如果语法文件存在
        if (file_exists(params.grammar.c_str())) {
            // 从文件中读取语法
            std::ifstream ifs(params.grammar.c_str());
            const std::string txt = std::string((std::istreambuf_iterator<char>(ifs)), std::istreambuf_iterator<char>());
            grammar = grammar_parser::parse(txt.c_str());
        } else {
            // 从字符串中读取语法
            grammar = grammar_parser::parse(params.grammar.c_str());
        }

        // 如果解析错误，语法规则为空
        if (grammar.rules.empty()) {
            ret_val = 1;
        } else {
            // 输出语法信息
            fprintf(stderr, "%s: grammar:\n", __func__);
            grammar_parser::print_grammar(stderr, grammar);
            fprintf(stderr, "\n");
        }
    }
    # 如果返回值为0，则执行以下逻辑
    if (ret_val == 0) {
        # 如果参数中的命令列表不为空，则处理命令列表
        if (!params.commands.empty()) {
            ret_val = process_command_list(ctx, audio, params);
        } 
        # 如果参数中的提示不为空且语法解析规则为空，则始终提示转录
        else if (!params.prompt.empty() && params.grammar_parsed.rules.empty()) {
            ret_val = always_prompt_transcription(ctx, audio, params);
        } 
        # 否则处理一般的转录
        else {
            ret_val = process_general_transcription(ctx, audio, params);
        }
    }

    # 暂停音频播放
    audio.pause();

    # 打印时间信息
    whisper_print_timings(ctx);
    # 释放上下文资源
    whisper_free(ctx);

    # 返回处理结果值
    return ret_val;
# 闭合之前的代码块
```