# `ggml\examples\common.h`

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

struct gpt_params {
    int32_t seed         = -1;   // RNG seed
    int32_t n_threads    = std::min(4, (int32_t) std::thread::hardware_concurrency());
    int32_t n_predict    = 200;  // new tokens to predict
    int32_t n_parallel   = 1;    // number of parallel streams
    int32_t n_batch      = 8;    // batch size for prompt processing
    int32_t n_ctx        = 2048; // context size (this is the KV cache max size)
    int32_t n_gpu_layers = 0;    // number of layers to offlload to the GPU

    bool ignore_eos = false; // ignore EOS token when generating text

    // sampling parameters
    int32_t top_k          = 40;
    float   top_p          = 0.9f;
    float   temp           = 0.9f;
    int32_t repeat_last_n  = 64;
    float   repeat_penalty = 1.00f;

    std::string model      = "models/gpt-2-117M/ggml-model.bin"; // model path
    std::string prompt     = "";
    std::string token_test = "";

    bool    interactive      = false;
    int32_t interactive_port = -1;
};

// Function to parse command line arguments for GPT
bool gpt_params_parse(int argc, char ** argv, gpt_params & params);

// Function to print usage information for GPT
void gpt_print_usage(int argc, char ** argv, const gpt_params & params);

// Function to generate a random prompt for GPT
std::string gpt_random_prompt(std::mt19937 & rng);

//
// Vocab utils
//

// Function to trim whitespace from a string
std::string trim(const std::string & s);

// Function to replace a substring with another substring in a string
std::string replace(
        const std::string & s,
        const std::string & from,
        const std::string & to);

// Structure to handle GPT vocabulary
struct gpt_vocab {
    using id    = int32_t;
    using token = std::string;

    std::map<token, id> token_to_id; // Map to store token to id mapping
    std::map<id, token> id_to_token; // Map to store id to token mapping
    std::vector<std::string> special_tokens; // Vector to store special tokens

    // Function to add a special token to the vocabulary
    void add_special_token(const std::string & token);
};

// Function for simple JSON parsing
std::map<std::string, int32_t> json_parse(const std::string & fname);
// 将宽字符串转换为 UTF-8 编码的字符串
std::string convert_to_utf8(const std::wstring & input);

// 将字符串转换为宽字符串
std::wstring convert_to_wstring(const std::string & input);

// 将输入的字符串分割成单词，并存储到 vector 中
void gpt_split_words(std::string str, std::vector<std::string>& words);

// 使用给定的词汇表对文本进行分词，返回 token 的 id 组成的 vector
std::vector<gpt_vocab::id> gpt_tokenize(const gpt_vocab & vocab, const std::string & text);

// 测试 gpt_tokenize 函数的输出
// - 与 huggingface tokenizer 生成的 token 进行比较
// - 测试用例基于模型的主要语言（在 'prompt' 目录下）
// - 如果所有句子的 token 化结果相同，则打印 'All tests passed.'
// - 否则，打印句子、huggingface tokens、ggml tokens
void test_gpt_tokenizer(gpt_vocab & vocab, const std::string & fpath_test);

// 从 encoder.json 文件中加载词汇表
bool gpt_vocab_init(const std::string & fname, gpt_vocab & vocab);

// 根据每个嵌入的概率样本，返回下一个 token 的 id
// - 仅考虑前 K 个 token
// - 在其中，仅考虑累积概率大于 P 的 token
// - TODO: 不确定这个实现是否正确
// - TODO: 温度参数尚未实现
gpt_vocab::id gpt_sample_top_k_top_p(
        const gpt_vocab & vocab,
        const float * logits,
        int    top_k,
        double top_p,
        double temp,
        std::mt19937 & rng);

// 根据每个嵌入的概率样本，返回下一个 token 的 id，并考虑最近 N 个 token 的重复情况
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

// 读取 WAV 音频文件，并将 PCM 数据存储到 pcmf32 中
// 音频的采样率必须等于 COMMON_SAMPLE_RATE
// 如果立体声标志被设置，并且音频有 2 个声道，那么 pcmf32s 将包含 2 个声道的 PCM
bool read_wav(
        const std::string & fname,  // WAV 文件名
        std::vector<float> & pcmf32,  // 存储 PCM 数据的向量
        std::vector<std::vector<float>> & pcmf32s,  // 存储多声道 PCM 数据的向量
        bool stereo);  // 立体声标志

// 将 PCM 数据写入 WAV 音频文件
class wav_writer {
private:
    std::ofstream file;  // 文件输出流
    uint32_t dataSize = 0;  // 数据大小
    std::string wav_filename;  // WAV 文件名

    // 写入 WAV 文件头部信息
    bool write_header(const uint32_t sample_rate,  // 采样率
                      const uint16_t bits_per_sample,  // 每个样本的位数
                      const uint16_t channels) {  // 声道数

        file.write("RIFF", 4);  // 写入 RIFF 标识
        file.write("\0\0\0\0", 4);    // 文件大小的占位符
        file.write("WAVE", 4);  // 写入 WAVE 标识
        file.write("fmt ", 4);  // 写入 fmt 标识

        const uint32_t sub_chunk_size = 16;  // 子块大小
        const uint16_t audio_format = 1;      // PCM 格式
        const uint32_t byte_rate = sample_rate * channels * bits_per_sample / 8;  // 每秒的字节数
        const uint16_t block_align = channels * bits_per_sample / 8;  // 数据块的大小

        file.write(reinterpret_cast<const char *>(&sub_chunk_size), 4);  // 写入子块大小
        file.write(reinterpret_cast<const char *>(&audio_format), 2);  // 写入音频格式
        file.write(reinterpret_cast<const char *>(&channels), 2);  // 写入声道数
        file.write(reinterpret_cast<const char *>(&sample_rate), 4);  // 写入采样率
        file.write(reinterpret_cast<const char *>(&byte_rate), 4);  // 写入每秒的字节数
        file.write(reinterpret_cast<const char *>(&block_align), 2);  // 写入数据块的大小
        file.write(reinterpret_cast<const char *>(&bits_per_sample), 2);  // 写入每个样本的位数
        file.write("data", 4);  // 写入 data 标识
        file.write("\0\0\0\0", 4);    // 数据大小的占位符

        return true;
    }

    // 假设 PCM 数据被归一化到 -1 到 1 的范围内
    // 将浮点数数据写入音频文件
    bool write_audio(const float * data, size_t length) {
        // 遍历数据数组
        for (size_t i = 0; i < length; ++i) {
            // 将浮点数转换为16位整数
            const int16_t intSample = data[i] * 32767;
            // 将整数写入文件
            file.write(reinterpret_cast<const char *>(&intSample), sizeof(int16_t));
            // 更新数据大小
            dataSize += sizeof(int16_t);
        }
        // 如果文件已打开
        if (file.is_open()) {
            // 设置文件指针到文件大小位置
            file.seekp(4, std::ios::beg);
            // 计算文件大小并写入文件
            uint32_t fileSize = 36 + dataSize;
            file.write(reinterpret_cast<char *>(&fileSize), 4);
            // 设置文件指针到数据大小位置
            file.seekp(40, std::ios::beg);
            // 写入数据大小
            file.write(reinterpret_cast<char *>(&dataSize), 4);
            // 设置文件指针到文件末尾
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
// 定义一个公共函数，用于打开 WAV 文件并写入头部信息
// 参数包括文件名、采样率、每个样本的位数和声道数
bool open(const std::string & filename,
          const    uint32_t   sample_rate,
          const    uint16_t   bits_per_sample,
          const    uint16_t   channels) {

    // 如果成功打开 WAV 文件，则写入头部信息
    if (open_wav(filename)) {
        write_header(sample_rate, bits_per_sample, channels);
    } else {
        return false;
    }

    return true;
}

// 关闭 WAV 文件
bool close() {
    file.close();
    return true;
}

// 写入 PCM 音频数据
bool write(const float * data, size_t length) {
    return write_audio(data, length);
}

// WAV 文件析构函数，如果文件已打开，则关闭文件
~wav_writer() {
    if (file.is_open()) {
        file.close();
    }
}

// 对 PCM 音频应用高通频率滤波器
// 抑制低于截止频率的频率
void high_pass_filter(
        std::vector<float> & data,
        float cutoff,
        float sample_rate);

// 基本的语音活动检测（VAD），使用音频能量自适应阈值
bool vad_simple(
        std::vector<float> & pcmf32,
        int   sample_rate,
        int   last_ms,
        float vad_thold,
        float freq_thold,
        bool  verbose);

// 使用Levenshtein距离计算两个字符串之间的相似度
float similarity(const std::string & s0, const std::string & s1);

// SAM 参数解析
struct sam_params {
    int32_t seed      = -1; // 随机数生成器种子
    int32_t n_threads = std::min(4, (int32_t) std::thread::hardware_concurrency());

    std::string model     = "models/sam-vit-b/ggml-model-f16.bin"; // 模型路径
    std::string fname_inp = "img.jpg";
    std::string fname_out = "img.out";
};

// 解析 SAM 参数
bool sam_params_parse(int argc, char ** argv, sam_params & params);

// 打印 SAM 使用说明
void sam_print_usage(int argc, char ** argv, const sam_params & params);
```