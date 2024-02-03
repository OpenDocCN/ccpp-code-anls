# `whisper.cpp\examples\common.h`

```cpp
// Various helper functions and utilities

#pragma once

#include <string>
#include <map>
#include <vector>
#include <random>
#include <thread>
#include <ctime>
#include <fstream>

#define COMMON_SAMPLE_RATE 16000

//
// GPT CLI argument parsing
//

// 结构体用于存储 GPT CLI 参数
struct gpt_params {
    int32_t seed         = -1;   // RNG seed，随机数生成器种子
    int32_t n_threads    = std::min(4, (int32_t) std::thread::hardware_concurrency()); // 线程数，默认为最小值4和硬件并发数
    int32_t n_predict    = 200;  // new tokens to predict，预测的新 token 数量
    int32_t n_parallel   = 1;    // number of parallel streams，并行流的数量
    int32_t n_batch      = 8;    // batch size for prompt processing，用于处理提示的批处理大小
    int32_t n_ctx        = 2048; // context size (this is the KV cache max size)，上下文大小（这是 KV 缓存的最大大小）
    int32_t n_gpu_layers = 0;    // number of layers to offlload to the GPU，要卸载到 GPU 的层数

    bool ignore_eos = false; // ignore EOS token when generating text，在生成文本时忽略 EOS token

    // sampling parameters，采样参数
    int32_t top_k          = 40; // top-k 参数
    float   top_p          = 0.9f; // top-p 参数
    float   temp           = 0.9f; // 温度参数
    int32_t repeat_last_n  = 64; // 重复最后 n 个 token
    float   repeat_penalty = 1.00f; // 重复惩罚

    std::string model      = "models/gpt-2-117M/ggml-model.bin"; // model path，模型路径
    std::string prompt     = ""; // 提示
    std::string token_test = ""; // token 测试

    bool    interactive      = false; // 交互模式
    int32_t interactive_port = -1; // 交互端口
};

// 解析 GPT CLI 参数
bool gpt_params_parse(int argc, char ** argv, gpt_params & params);

// 打印使用说明
void gpt_print_usage(int argc, char ** argv, const gpt_params & params);

// 生成随机提示
std::string gpt_random_prompt(std::mt19937 & rng);

//
// Vocab utils
//

// 去除字符串两端空格
std::string trim(const std::string & s);

// 替换字符串中的子串
std::string replace(
        const std::string & s,
        const std::string & from,
        const std::string & to);

// 词汇表结构体
struct gpt_vocab {
    using id    = int32_t;
    using token = std::string;

    std::map<token, id> token_to_id; // token 到 id 的映射
    std::map<id, token> id_to_token; // id 到 token 的映射
    std::vector<std::string> special_tokens; // 特殊 token

    // 添加特殊 token
    void add_special_token(const std::string & token);
};

// 简易的 JSON 解析
std::map<std::string, int32_t> json_parse(const std::string & fname);
// 将宽字符串转换为 UTF-8 字符串
std::string convert_to_utf8(const std::wstring & input);

// 将 UTF-8 字符串转换为宽字符串
std::wstring convert_to_wstring(const std::string & input);

// 将字符串拆分为单词
void gpt_split_words(std::string str, std::vector<std::string>& words);

// 将文本分割为标记
//
// 参考链接: https://github.com/openai/gpt-2/blob/a74da5d99abaaba920de8131d64da2862a8f213b/src/encoder.py#L53
//
// Python 正则表达式:
// r"""'s|'t|'re|'ve|'m|'ll|'d| ?\p{L}+| ?\p{N}+| ?[^\s\p{L}\p{N}]+|\s+(?!\S)|\s+"""
//
// C++ 正则表达式:
// R"('s|'t|'re|'ve|'m|'ll|'d| ?[[:alpha:]]+| ?[[:digit:]]+| ?[^\s[:alpha:][:digit:]]+|\s+(?!\S)|\s+)"
//
std::vector<gpt_vocab::id> gpt_tokenize(const gpt_vocab & vocab, const std::string & text);

// 测试 gpt_tokenize 的输出
//
//   - 与 huggingface tokenizer 生成的标记进行比较
//   - 测试案例基于模型的主要语言选择（在 'prompt' 目录下）
//   - 如果所有句子的标记化相同，则打印 'All tests passed.'
//   - 否则，打印句子、huggingface 标记、ggml 标记
//
void test_gpt_tokenizer(gpt_vocab & vocab, const std::string & fpath_test);

// 从 encoder.json 加载标记
bool gpt_vocab_init(const std::string & fname, gpt_vocab & vocab);

// 根据每个嵌入的概率样本下一个标记
//
//   - 仅考虑前 K 个标记
//   - 从中，仅考虑累积概率 > P 的前标记
//
// TODO: 不确定此实现是否正确
// TODO: 温度参数未实现
//
gpt_vocab::id gpt_sample_top_k_top_p(
        const gpt_vocab & vocab,
        const float * logits,
        int    top_k,
        double top_p,
        double temp,
        std::mt19937 & rng);

gpt_vocab::id gpt_sample_top_k_top_p_repeat(
        const gpt_vocab & vocab,
        const float * logits,
        const int32_t * last_n_tokens_data,
        size_t last_n_tokens_data_size,
        int    top_k,
        double top_p,
        double temp,
        int repeat_last_n,
        float repeat_penalty,
        std::mt19937 & rng);
// Audio utils
//

// 检查缓冲区是否为 WAV 音频文件
bool is_wav_buffer(const std::string buf);

// 读取 WAV 音频文件并将 PCM 数据存储到 pcmf32 中
// fname 可以是 WAV 数据的缓冲区，而不是文件名
// 音频的采样率必须等于 COMMON_SAMPLE_RATE
// 如果立体声标志被设置，并且音频有 2 个声道，则 pcmf32s 将包含 2 个声道的 PCM
bool read_wav(
        const std::string & fname,
        std::vector<float> & pcmf32,
        std::vector<std::vector<float>> & pcmf32s,
        bool stereo);

// 将 PCM 数据写入 WAV 音频文件
class wav_writer {
private:
    std::ofstream file;
    uint32_t dataSize = 0;
    std::string wav_filename;

    // 写入 WAV 文件头信息
    bool write_header(const uint32_t sample_rate,
                      const uint16_t bits_per_sample,
                      const uint16_t channels) {

        file.write("RIFF", 4);
        file.write("\0\0\0\0", 4);    // 文件大小的占位符
        file.write("WAVE", 4);
        file.write("fmt ", 4);

        const uint32_t sub_chunk_size = 16;
        const uint16_t audio_format = 1;      // PCM 格式
        const uint32_t byte_rate = sample_rate * channels * bits_per_sample / 8;
        const uint16_t block_align = channels * bits_per_sample / 8;

        file.write(reinterpret_cast<const char *>(&sub_chunk_size), 4);
        file.write(reinterpret_cast<const char *>(&audio_format), 2);
        file.write(reinterpret_cast<const char *>(&channels), 2);
        file.write(reinterpret_cast<const char *>(&sample_rate), 4);
        file.write(reinterpret_cast<const char *>(&byte_rate), 4);
        file.write(reinterpret_cast<const char *>(&block_align), 2);
        file.write(reinterpret_cast<const char *>(&bits_per_sample), 2);
        file.write("data", 4);
        file.write("\0\0\0\0", 4);    // 数据大小的占位符

        return true;
    }

    // 假设 PCM 数据已经归一化到 -1 到 1 的范围
    // 写入音频数据到文件
    bool write_audio(const float * data, size_t length) {
        // 遍历音频数据
        for (size_t i = 0; i < length; ++i) {
            // 将浮点数样本转换为16位整数样本
            const int16_t intSample = data[i] * 32767;
            // 将整数样本写入文件
            file.write(reinterpret_cast<const char *>(&intSample), sizeof(int16_t));
            // 更新数据大小
            dataSize += sizeof(int16_t);
        }
        // 如果文件已打开
        if (file.is_open()) {
            // 设置文件指针位置到文件大小字段
            file.seekp(4, std::ios::beg);
            // 计算文件大小
            uint32_t fileSize = 36 + dataSize;
            // 写入文件大小到文件
            file.write(reinterpret_cast<char *>(&fileSize), 4);
            // 设置文件指针位置到数据大小字段
            file.seekp(40, std::ios::beg);
            // 写入数据大小到文件
            file.write(reinterpret_cast<char *>(&dataSize), 4);
            // 设置文件指针位置到文件末尾
            file.seekp(0, std::ios::end);
        }
        // 返回写入结果
        return true;
    }

    // 打开 WAV 文件
    bool open_wav(const std::string & filename) {
        // 如果文件名不同于当前 WAV 文件名
        if (filename != wav_filename) {
            // 如果文件已打开，则关闭文件
            if (file.is_open()) {
                file.close();
            }
        }
        // 如果文件未打开
        if (!file.is_open()) {
            // 以二进制模式打开文件
            file.open(filename, std::ios::binary);
            // 更新 WAV 文件名
            wav_filename = filename;
            // 重置数据大小
            dataSize = 0;
        }
        // 返回文件是否打开的结果
        return file.is_open();
    }
// WAV 文件写入类，包含打开、关闭、写入等操作
public:
    bool open(const std::string & filename, // 打开 WAV 文件，指定文件名、采样率、采样位数和声道数
              const uint32_t sample_rate,
              const uint16_t bits_per_sample,
              const uint16_t channels) {

        if (open_wav(filename)) { // 如果成功打开 WAV 文件
            write_header(sample_rate, bits_per_sample, channels); // 写入 WAV 文件头信息
        } else {
            return false; // 打开失败则返回 false
        }

        return true; // 打开成功返回 true
    }

    bool close() { // 关闭 WAV 文件
        file.close(); // 关闭文件
        return true; // 返回 true
    }

    bool write(const float * data, size_t length) { // 写入音频数据
        return write_audio(data, length); // 调用写入音频数据的函数
    }

    ~wav_writer() { // WAV 文件写入类析构函数
        if (file.is_open()) { // 如果文件处于打开状态
            file.close(); // 关闭文件
        }
    }
};


// 对 PCM 音频应用高通频率滤波器
// 抑制低于截止频率的频率
void high_pass_filter(
        std::vector<float> & data, // 音频数据
        float cutoff, // 截止频率
        float sample_rate); // 采样率

// 基本的语音活动检测（VAD），使用音频能量自适应阈值
bool vad_simple(
        std::vector<float> & pcmf32, // PCM 32 位浮点音频数据
        int sample_rate, // 采样率
        int last_ms, // 上一个毫秒
        float vad_thold, // VAD 阈值
        float freq_thold, // 频率阈值
        bool verbose); // 是否输出详细信息

// 计算两个字符串之间的相似度，使用 Levenshtein 距离
float similarity(const std::string & s0, const std::string & s1);

//
// SAM 参数解析
//

struct sam_params {
    int32_t seed = -1; // 随机数生成器种子
    int32_t n_threads = std::min(4, (int32_t) std::thread::hardware_concurrency()); // 线程数

    std::string model = "models/sam-vit-b/ggml-model-f16.bin"; // 模型路径
    std::string fname_inp = "img.jpg"; // 输入文件名
    std::string fname_out = "img.out"; // 输出文件名
};

bool sam_params_parse(int argc, char ** argv, sam_params & params); // 解析 SAM 参数

void sam_print_usage(int argc, char ** argv, const sam_params & params); // 打印 SAM 使用说明
```