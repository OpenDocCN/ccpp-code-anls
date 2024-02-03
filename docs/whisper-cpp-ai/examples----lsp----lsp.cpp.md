# `whisper.cpp\examples\lsp\lsp.cpp`

```cpp
#include "common.h"
#include "common-sdl.h"
#include "whisper.h"
#include "json.hpp"

#include <iostream>
#include <cassert>
#include <cstdio>
#include <string>
#include <thread>
#include <vector>
#include <deque>
#include <set>

using json = nlohmann::json;

// 定义命令行参数结构体
struct whisper_params {
    int32_t n_threads  = std::min(4, (int32_t) std::thread::hardware_concurrency());
    int32_t prompt_ms  = 5000;
    int32_t command_ms = 8000;
    int32_t capture_id = -1;
    int32_t max_tokens = 32;
    int32_t audio_ctx  = 0;

    float vad_thold    = 0.6f;
    float freq_thold   = 100.0f;

    bool speed_up      = false;
    bool translate     = false;
    bool print_special = false;
    bool print_energy  = false;
    bool use_gpu       = true;

    std::string language  = "en";
    std::string model     = "models/ggml-base.en.bin";
};

// 定义命令结构体
struct command {
    std::vector<whisper_token> tokens;
    std::string plaintext;
};

// 定义命令集结构体
struct commandset {
    std::vector<struct command> commands;
    std::vector<whisper_token> prompt_tokens;
    // TODO: Store longest command?
    // Multi-token commands should have probabilities of subsequent logits
    // given that the prior logit is correct.
    // In this case, all commands must be iterated.
    // This however, is likely highly involved as different tokens
    // almost certainly have different spoken lengths
    // It would also have performance implications equivalent to a beam search
};

// 打印程序的使用方法
void whisper_print_usage(int argc, char ** argv, const whisper_params & params);

// 解析命令行参数
bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    // 遍历命令行参数，从第二个参数开始
    for (int i = 1; i < argc; i++) {
        // 获取当前参数
        std::string arg = argv[i];

        // 判断是否为帮助选项，如果是则打印用法信息并退出程序
        if (arg == "-h" || arg == "--help") {
            whisper_print_usage(argc, argv, params);
            exit(0);
        }
        // 设置线程数参数
        else if (arg == "-t"   || arg == "--threads")       { params.n_threads     = std::stoi(argv[++i]); }
        // 设置提示间隔参数
        else if (arg == "-pms" || arg == "--prompt-ms")     { params.prompt_ms     = std::stoi(argv[++i]); }
        // 设置命令间隔参数
        else if (arg == "-cms" || arg == "--command-ms")    { params.command_ms    = std::stoi(argv[++i]); }
        // 设置捕获 ID 参数
        else if (arg == "-c"   || arg == "--capture")       { params.capture_id    = std::stoi(argv[++i]); }
        // 设置最大令牌数参数
        else if (arg == "-mt"  || arg == "--max-tokens")    { params.max_tokens    = std::stoi(argv[++i]); }
        // 设置音频上下文参数
        else if (arg == "-ac"  || arg == "--audio-ctx")     { params.audio_ctx     = std::stoi(argv[++i]); }
        // 设置 VAD 阈值参数
        else if (arg == "-vth" || arg == "--vad-thold")     { params.vad_thold     = std::stof(argv[++i]); }
        // 设置频率阈值参数
        else if (arg == "-fth" || arg == "--freq-thold")    { params.freq_thold    = std::stof(argv[++i]); }
        // 设置加速参数
        else if (arg == "-su"  || arg == "--speed-up")      { params.speed_up      = true; }
        // 设置翻译参数
        else if (arg == "-tr"  || arg == "--translate")     { params.translate     = true; }
        // 设置打印特殊信息参数
        else if (arg == "-ps"  || arg == "--print-special") { params.print_special = true; }
        // 设置打印能量参数
        else if (arg == "-pe"  || arg == "--print-energy")  { params.print_energy  = true; }
        // 设置不使用 GPU 参数
        else if (arg == "-ng"  || arg == "--no-gpu")        { params.use_gpu       = false; }
        // 设置语言参数
        else if (arg == "-l"   || arg == "--language")      { params.language      = argv[++i]; }
        // 设置模型参数
        else if (arg == "-m"   || arg == "--model")         { params.model         = argv[++i]; }
        // 未知参数，打印错误信息并退出程序
        else {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            whisper_print_usage(argc, argv, params);
            exit(0);
        }
    }

    // 返回 true 表示参数解析成功
    return true;
// 打印程序的使用方法，包括各种选项的说明和默认值
void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params) {
    // 打印空行
    fprintf(stderr, "\n");
    // 打印程序名称和使用方法
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    // 打印空行
    fprintf(stderr, "\n");
    // 打印选项部分的说明
    fprintf(stderr, "options:\n");
    // 打印各个选项的说明和默认值
    fprintf(stderr, "  -h,         --help           [default] show this help message and exit\n");
    fprintf(stderr, "  -t N,       --threads N      [%-7d] number of threads to use during computation\n", params.n_threads);
    fprintf(stderr, "  -pms N,     --prompt-ms N    [%-7d] prompt duration in milliseconds\n",             params.prompt_ms);
    fprintf(stderr, "  -cms N,     --command-ms N   [%-7d] command duration in milliseconds\n",            params.command_ms);
    fprintf(stderr, "  -c ID,      --capture ID     [%-7d] capture device ID\n",                           params.capture_id);
    fprintf(stderr, "  -mt N,      --max-tokens N   [%-7d] maximum number of tokens per audio chunk\n",    params.max_tokens);
    fprintf(stderr, "  -ac N,      --audio-ctx N    [%-7d] audio context size (0 - all)\n",                params.audio_ctx);
    fprintf(stderr, "  -vth N,     --vad-thold N    [%-7.2f] voice activity detection threshold\n",        params.vad_thold);
    fprintf(stderr, "  -fth N,     --freq-thold N   [%-7.2f] high-pass frequency cutoff\n",                params.freq_thold);
    fprintf(stderr, "  -su,        --speed-up       [%-7s] speed up audio by x2 (reduced accuracy)\n",     params.speed_up ? "true" : "false");
    fprintf(stderr, "  -tr,        --translate      [%-7s] translate from source language to english\n",   params.translate ? "true" : "false");
    fprintf(stderr, "  -ps,        --print-special  [%-7s] print special tokens\n",                        params.print_special ? "true" : "false");
    fprintf(stderr, "  -pe,        --print-energy   [%-7s] print sound energy (for debugging)\n",          params.print_energy ? "true" : "false");
}
    # 打印信息到标准错误流，显示是否禁用 GPU
    fprintf(stderr, "  -ng,        --no-gpu         [%-7s] disable GPU\n", params.use_gpu ? "false" : "true");
    # 打印信息到标准错误流，显示语言选项
    fprintf(stderr, "  -l LANG,    --language LANG  [%-7s] spoken language\n", params.language.c_str());
    # 打印信息到标准错误流，显示模型路径选项
    fprintf(stderr, "  -m FNAME,   --model FNAME    [%-7s] model path\n", params.model.c_str());
    # 打印空行到标准错误流
    fprintf(stderr, "\n");
}
// 等待语音活动检测（VAD）触发，获取音频数据
uint64_t wait_for_vad(audio_async & audio, json jparams, const whisper_params & params, uint64_t maxlength_ms, std::vector<float> & pcmf32) {
    // 使用 std::chrono 命名空间
    using namespace std::chrono;
    // 获取当前时间的毫秒数
    uint64_t time_now = time_point_cast<milliseconds>(system_clock::now()).time_since_epoch().count();
    // 记录开始时间为当前时间
    uint64_t start_time = time_now;
    // 如果参数中包含时间戳，则将开始时间设置为参数中的时间戳
    if (jparams.contains("timestamp")) {
        start_time = jparams.at("timestamp");
    }
    // 如果当前时间与开始时间的差小于 500 毫秒
    if(time_now - start_time < 500) {
        // 等待音频数据积压
        std::this_thread::sleep_for(milliseconds(500 - (time_now - start_time)));
        // 更新当前时间
        time_now = time_point_cast<milliseconds>(system_clock::now()).time_since_epoch().count();
    } else if (time_now - start_time > 1000) {
        // 获取音频数据
        audio.get(time_now-start_time, pcmf32);
        // 计算最大偏移量
        size_t max_offset = pcmf32.size() - WHISPER_SAMPLE_RATE;
        // 循环处理音频数据
        for(size_t offset=0;offset < max_offset;offset+=WHISPER_SAMPLE_RATE/10) {
            // 提取音频数据块
            std::vector<float> audio_chunk(&pcmf32[offset], &pcmf32[offset+WHISPER_SAMPLE_RATE]);
            // 进行简单的 VAD 检测
            if(::vad_simple(audio_chunk, WHISPER_SAMPLE_RATE, 1000, params.vad_thold, params.freq_thold, params.print_energy)) {
                // 调整音频数据大小
                pcmf32.resize(offset+WHISPER_SAMPLE_RATE);
                // 如果偏移量对应的时间超过最大长度，则删除开头的样本
                if (offset*1000/WHISPER_SAMPLE_RATE+1000 > maxlength_ms) {
                    // 删除开头的样本
                    pcmf32.erase(pcmf32.begin(),pcmf32.end()-(maxlength_ms*WHISPER_SAMPLE_RATE/1000));
                    // 输出信息
                    fprintf(stderr, "Shortened samples");
                }
                // 返回触发 VAD 的时间戳
                return start_time + offset*1000/WHISPER_SAMPLE_RATE+1000;
            }
        }
    }
    // 计算窗口持续时间
    size_t window_duration = std::max((uint64_t)1000, time_now-start_time);
    // 获取音频数据
    audio.get(window_duration, pcmf32);
    // 当音频流中没有检测到语音时，循环等待1000毫秒
    while (!::vad_simple(pcmf32, WHISPER_SAMPLE_RATE, 1000, params.vad_thold, params.freq_thold, params.print_energy)) {
        // 线程休眠100毫秒
        std::this_thread::sleep_for(milliseconds(100));
        // 获取当前时间的毫秒数
        time_now = time_point_cast<milliseconds>(system_clock::now()).time_since_epoch().count();
        // 计算窗口持续时间
        window_duration = std::max((uint64_t)1000,time_now-start_time);
        // 获取音频流数据
        audio.get(window_duration, pcmf32);
    }
    // 如果录音时间超过最大长度，则获取最大长度的音频数据
    if (time_now - start_time > maxlength_ms) {
        audio.get(maxlength_ms, pcmf32);
    } else {
        // 否则获取录音时间内的音频数据
        audio.get(time_now - start_time, pcmf32);
    }

    // 返回当前时间
    return time_now;
// 无指导转录函数，接收上下文、音频、参数 JSON 对象和参数结构体，返回转录结果 JSON 对象
json unguided_transcription(struct whisper_context * ctx, audio_async &audio, json jparams, const whisper_params &params) {
    // 存储提示文本的令牌和 PCM 浮点数数据
    std::vector<whisper_token> prompt_tokens;
    std::vector<float> pcmf32;
    // 等待 VAD 检测到语音，获取未处理音频时间戳
    uint64_t unprocessed_audio_timestamp = wait_for_vad(audio, jparams, params, 10000U, pcmf32);

    // 设置默认的 Whisper 参数
    whisper_full_params wparams = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);
    if (jparams.contains("prompt")) {
        // 如果参数中包含提示文本，则进行处理
        std::string prompt = jparams.at("prompt");
        prompt_tokens.resize(1024);
        // 将提示文本令牌化
        int n = whisper_tokenize(ctx, prompt.c_str(), prompt_tokens.data(), 1024);
        prompt_tokens.resize(n);

        wparams.prompt_tokens    = prompt_tokens.data();
        wparams.prompt_n_tokens  = prompt_tokens.size();
    }
    // 设置 Whisper 参数的各项属性
    wparams.print_progress   = false;
    wparams.print_special    = params.print_special;
    wparams.print_realtime   = false;
    wparams.print_timestamps = false;
    wparams.translate        = params.translate;
    wparams.no_context       = jparams.value("no_context", true);
    wparams.single_segment   = true;
    wparams.max_tokens       = params.max_tokens;
    wparams.language         = params.language.c_str();
    wparams.n_threads        = params.n_threads;

    wparams.audio_ctx        = params.audio_ctx;
    wparams.speed_up         = params.speed_up;
    wparams.suppress_non_speech_tokens = true;
    // 运行转换器和单个解码过程
    if (whisper_full(ctx, wparams, pcmf32.data(), pcmf32.size()) != 0) {
        fprintf(stderr, "%s: ERROR: whisper_full() failed\n", __func__);
        // 抛出异常
        throw json{
            {"code", -32803},
            {"message", "ERROR: whisper_full() failed"}
        };
    }
    // 获取第一个片段的文本结果
    std::string result = whisper_full_get_segment_text(ctx,0);
    // 返回转录结果 JSON 对象
    return json {
        {"transcription", result},
        {"timestamp", unprocessed_audio_timestamp}
    };
}

// 命令列表模式
// 通过提供的命令集列表，引导转录以匹配最可能的命令
json guided_transcription(struct whisper_context * ctx, audio_async &audio, const whisper_params &params, json jparams, std::vector<struct commandset> commandset_list) {
    // 从命令集列表中选择一个命令集
    struct commandset cs = commandset_list[jparams.value("commandset_index", commandset_list.size()-1)];
    // 存储音频数据的向量
    std::vector<float> pcmf32;
    // 等待语音活动检测，获取未处理的音频时间戳
    uint64_t unprocessed_audio_timestamp = wait_for_vad(audio, jparams, params, 2000U, pcmf32);

    // 打印检测到语音的消息
    fprintf(stderr, "%s: Speech detected! Processing ...\n", __func__);
    // 设置默认的 Whisper 参数
    whisper_full_params wparams = whisper_full_default_params(WHISPER_SAMPLING_GREEDY);

    wparams.print_progress   = false;
    wparams.print_special    = params.print_special;
    wparams.print_realtime   = false;
    wparams.print_timestamps = false;
    wparams.translate        = params.translate;
    wparams.no_context       = true;
    wparams.single_segment   = true;
    wparams.max_tokens       = 1;
    wparams.language         = params.language.c_str();
    wparams.n_threads        = params.n_threads;

    wparams.audio_ctx        = params.audio_ctx;
    wparams.speed_up         = params.speed_up;

    // TODO: 进行一些时间测试。过长的提示会减慢处理速度吗？
    // 设置命令集/预计算提示
    wparams.prompt_tokens    = cs.prompt_tokens.data();
    wparams.prompt_n_tokens  = cs.prompt_tokens.size();
    // TODO: 适当地作为选项暴露
    wparams.suppress_non_speech_tokens = true;

    // 运行变换器和单个解码过程
    if (whisper_full(ctx, wparams, pcmf32.data(), pcmf32.size()) != 0) {
        // 打印错误消息并抛出异常
        fprintf(stderr, "%s: ERROR: whisper_full() failed\n", __func__);
        throw json{
            {"code", -32803},
            {"message", "ERROR: whisper_full() failed"}//TODO: format string (sprintf?)
        };
    }

    // 估计命令概率
    // 注意：不是最优解
}
    {
        // 获取模型输出的logits
        const auto * logits = whisper_get_logits(ctx);
    
        // 创建一个与词汇表大小相同的概率向量，初始化为0.0
        std::vector<float> probs(whisper_n_vocab(ctx), 0.0f);
    
        // 通过softmax函数从logits计算概率
        {
            // 初始化最大值为一个很小的负数
            float max = -1e9;
            // 找到logits中的最大值
            for (int i = 0; i < (int) probs.size(); ++i) {
                max = std::max(max, logits[i]);
            }
    
            // 计算softmax的分母
            float sum = 0.0f;
            // 计算softmax的分子，并累加到分母
            for (int i = 0; i < (int) probs.size(); ++i) {
                probs[i] = expf(logits[i] - max);
                sum += probs[i];
            }
    
            // 归一化概率向量
            for (int i = 0; i < (int) probs.size(); ++i) {
                probs[i] /= sum;
            }
        }
    
        // 创建一个存储概率和索引的向量
        std::vector<std::pair<float, int>> probs_id;
    
        // 在我的测试中，最可能的令牌总是期望的
        // TODO: 在验证有效性后修剪命令集结构
        for (int i = 0; i < (int) cs.commands.size(); ++i) {
            // 将概率和索引添加到向量中
            probs_id.emplace_back(probs[cs.commands[i].tokens[0]], i);
        }
    
        // 按照概率降序排序
        {
            using pair_type = decltype(probs_id)::value_type;
            std::sort(probs_id.begin(), probs_id.end(), [](const pair_type & a, const pair_type & b) {
                    return a.first > b.first;
                    });
        }
    
        // 获取具有最高概率的命令索引
        int id = probs_id[0].second;
        // 返回JSON对象
        return json{
            {"command_index", id},
                {"command_text", cs.commands[id].plaintext},
                {"timestamp", unprocessed_audio_timestamp},
        };
    }
}

// 注册命令集，检查令牌冲突，返回命令集索引
json register_commandset(struct whisper_context * ctx, json jparams, std::vector<struct commandset> &commandset_list) {
    // 创建命令集对象
    struct commandset cs;

    // 设置提示信息
    std::string  k_prompt = " select one from the available words: ";
    // 创建令牌集合
    std::set<whisper_token> token_set;
    whisper_token tokens[32];
    // 遍历参数列表
    for (std::string s : jparams) {
        std::vector<whisper_token> token_vec;
        // 对命令进行分词
        const int n = whisper_tokenize(ctx, (" " + s).c_str(), tokens, 32);
        // 处理分词错误情况
        if (n < 0) {
            fprintf(stderr, "%s: error: failed to tokenize command '%s'\n", __func__, s.c_str());
            return 3;
        }
        // 添加第一个令牌到令牌集合
        token_vec.push_back(tokens[0]);
        // 检查是否存在重复令牌
        if (!token_set.insert(tokens[0]).second) {
            fprintf(stderr, "%s: warning: %s is a duplicate of an existing token\n", __func__, s.c_str());
            throw json{
                {"code",-31000},
                {"message", "Duplicate token in token set: " + s}
            };
        }
        // 输出错误信息，如果命令不止一个令牌
        if (n > 1) {
            fprintf(stderr, "%s: error: command is more than a single token: %s\n", __func__, s.c_str());
        }
        // 创建命令对象，添加到命令集
        struct command command = {token_vec, s};
        cs.commands.push_back(command);
        // 更新提示信息
        k_prompt += s;
    }
    // 调整提示信息格式
    k_prompt = k_prompt.substr(0,k_prompt.length()-2) + ". Selected word:";
    // 分词提示信息
    cs.prompt_tokens.resize(1024);
    int n = whisper_tokenize(ctx, k_prompt.c_str(), cs.prompt_tokens.data(), 1024);
    cs.prompt_tokens.resize(n);
    // 准备响应
    int index = commandset_list.size();
    commandset_list.push_back(cs);
    return json{{"index",index}};
}
// 查找函数，暂时未实现
json seek(struct whisper_context * /*ctx*/, audio_async & /*audio*/, json /*params*/) {
    // 抛出一个 JSON 异常，表示当前操作不支持
    throw json {
        // 设置错误码为 -32601
        {"code", -32601},
        // 设置错误信息为 "Seeking is not yet supported."
        {"message", "Seeking is not yet supported."}
    };
}
// 解析 JSON 请求并处理作业
json parse_job(const json &body, struct whisper_context * ctx, audio_async &audio, const whisper_params &params, std::vector<struct commandset> &commandset_list) {
    // 参考 JSON-RPC 规范：https://www.jsonrpc.org/specification
    // 从请求体中获取 id
    json id = body.at("id");
    try {
        // 从请求体中获取 JSON-RPC 版本号
        std::string version = body.at("jsonrpc");
        if (version != "2.0") {
            // 不支持的版本号，抛出异常
            throw json{
                {"code", -3260},
                {"message", "invalid jsonrpc version"}
            };
        }
        // 从请求体中获取方法名
        std::string method = body.at("method");
        // 初始化参数为默认值
        json jparams = json{{"dummy", "dummy"}};
        // 如果请求体中包含参数，则获取参数
        if (body.contains("params"))
            jparams = body.at("params");
        json res;
        // 输出调试信息
        fprintf(stderr, "Dispatching a job\n");
        // 根据方法名调用相应的处理函数
        if (method == "unguided")                { res = unguided_transcription(ctx, audio, jparams, params); }
        else if (method == "guided")             { res = guided_transcription(ctx, audio, params, jparams, commandset_list); }
        else if (method == "seek")               { res = seek(ctx, audio, jparams); }
        else if (method == "registerCommandset") { res = register_commandset(ctx, jparams, commandset_list); }
        else if (method == "echo")               { res = jparams; }

        // 返回处理结果
        return json{
            {"jsonrpc", "2.0"},
                {"result", res},
                {"id", id}
        };
    } catch(json ex) {
        // 处理异常情况
        return json {
            {"jsonrpc", "2.0"},
                {"error", ex},
                {"id", id}
        };
    }
}

// 处理循环
void process_loop(struct whisper_context * ctx, audio_async &audio, const whisper_params &params) {
    // 初始化作业队列和命令集列表
    std::deque<json> jobqueue;
    std::vector<struct commandset> commandset_list;
    // 进入循环，持续监听输入流
    while (true) {
        // 为了支持取消操作，如果输入流有内容或者任务队列为空，则不阻塞
        if (std::cin.rdbuf()->in_avail() > 22 || jobqueue.size() == 0) {
            // 读取 Content-Length 头部信息
            int content_length;
            if (scanf("Content-Length: %d", &content_length) != 1) {
                // 如果无法读取输入，输出错误信息并返回
                fprintf(stderr, "Could not read input: %d", std::cin.peek());
                return;
            }
            // 忽略换行符
            std::cin.ignore(2);
            if (std::cin.peek() != 13) {
                // Content-Type. jsonrpc 需要 utf8 编码
                std::cin.ignore(200,10);
            }
            std::cin.ignore(2);
            // 读取消息内容
            std::string content(content_length,'\0');
            std::cin.read(&content[0], content_length);
            // 解析 JSON 格式的消息
            json job = json::parse(content);
            // TODO: 一些消息（如取消操作）可能需要跳过队列
            if (job.is_array()) {
                // 响应也需要批量处理，稍后实现
                // for (subjob : job.begin())
                // TODO: 至少应该用不支持的错误响应
            } else {
                // 将任务加入队列
                jobqueue.push_back(job);
            }
        }
        // 确保任务队列中有任务
        assert(jobqueue.size() > 0);
        // 获取队列中的第一个任务
        json job = jobqueue.front();
        // 解析任务并获取响应
        json resp = parse_job(job, ctx, audio, params, commandset_list);
        // 如果响应不是 "unfinished"
        if (resp != "unfinished") {
            // 从队列中移除已处理的任务
            jobqueue.pop_front();
            // 发送响应
            std::string data = resp.dump(-1, ' ', false, json::error_handler_t::replace);
            fprintf(stdout, "Content-Length: %d\r\n\r\n%s\n", (int)data.length()+1, data.c_str());
            std::cout.flush();
        }
    }
// 主函数，接受命令行参数并执行程序
int main(int argc, char ** argv) {
    // 定义参数结构体
    whisper_params params;
    // 解析命令行参数，如果解析失败则返回1
    if (whisper_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 检查语言是否合法，如果不合法则输出错误信息并退出程序
    if (whisper_lang_id(params.language.c_str()) == -1) {
        fprintf(stderr, "error: unknown language '%s'\n", params.language.c_str());
        whisper_print_usage(argc, argv, params);
        exit(0);
    }

    // 初始化 whisper 上下文参数
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;
    // 从文件中初始化 whisper 上下文
    struct whisper_context * ctx = whisper_init_from_file_with_params(params.model.c_str(), cparams);
    // 初始化音频

    // 异步音频对象，设置缓冲时间为30秒
    audio_async audio(30*1000);
    // 初始化音频，如果失败则输出错误信息并返回1
    if (!audio.init(params.capture_id, WHISPER_SAMPLE_RATE)) {
        fprintf(stderr, "%s: audio.init() failed!\n", __func__);
        return 1;
    }

    // 恢复音频
    audio.resume();
    // 等待1秒钟以避免任何缓冲噪音
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    // 清空音频缓冲
    audio.clear();
    // 处理循环，传入上下文、音频和参数
    process_loop(ctx, audio, params);

    // 暂停音频
    audio.pause();
    // 打印 whisper 时间信息
    whisper_print_timings(ctx);
    // 释放 whisper 上下文
    whisper_free(ctx);

    // 返回0表示程序正常结束
    return 0;
}
```