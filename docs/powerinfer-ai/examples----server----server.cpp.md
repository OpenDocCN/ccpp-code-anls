# `PowerInfer\examples\server\server.cpp`

```
// 包含自定义的头文件 common.h, llama.h, grammar-parser.h
#include "common.h"
#include "llama.h"
#include "grammar-parser.h"

// 包含 llava 库中的 clip.h 头文件
#include "../llava/clip.h"

// 包含 stb_image 库中的头文件
#include "stb_image.h"

// 在调试模式下，使服务器崩溃；否则发送 http 500 错误
#ifndef NDEBUG
#define CPPHTTPLIB_NO_EXCEPTIONS 1
#endif

// 包含 httplib 库中的头文件
#include "httplib.h"

// 包含 json 库中的头文件
#include "json.hpp"

// 包含自动生成的文件 (使用 ./deps.sh 更新)
#include "index.html.hpp"
#include "index.js.hpp"
#include "completion.js.hpp"
// 包含 json-schema-to-grammar.mjs.hpp 头文件
#include "json-schema-to-grammar.mjs.hpp"

// 包含必要的标准库头文件
#include <cstddef>
#include <thread>
#include <mutex>
#include <chrono>

// 如果未定义 SERVER_VERBOSE，则设置为默认值 1
#ifndef SERVER_VERBOSE
#define SERVER_VERBOSE 1
#endif

// 使用 nlohmann::json 命名空间
using json = nlohmann::json;

// 定义服务器参数结构体
struct server_params
{
    // 服务器主机名，默认为 "127.0.0.1"
    std::string hostname = "127.0.0.1";
    // 公共路径，默认为 "examples/server/public"
    std::string public_path = "examples/server/public";
    // 服务器端口，默认为 8080
    int32_t port = 8080;
    // 读取超时时间，默认为 600 秒
    int32_t read_timeout = 600;
    // 写入超时时间，默认为 600 秒
    int32_t write_timeout = 600;
// 定义静态变量 server_verbose，用于控制是否输出详细日志
static bool server_verbose = false;

// 如果 SERVER_VERBOSE 不等于 1，则定义 LOG_VERBOSE 为空
#if SERVER_VERBOSE != 1
#define LOG_VERBOSE(MSG, ...)
// 如果 SERVER_VERBOSE 等于 1，则定义 LOG_VERBOSE 为输出详细日志的宏
#else
#define LOG_VERBOSE(MSG, ...)                                            \
    do                                                                   \
    {                                                                    \
        if (server_verbose)                                              \
        {                                                                \
            server_log("VERBOSE", __func__, __LINE__, MSG, __VA_ARGS__); \
        }                                                                \
    } while (0)
#endif

// 定义输出错误日志的宏，调用 server_log 函数输出错误信息
#define LOG_ERROR(  MSG, ...) server_log("ERROR",   __func__, __LINE__, MSG, __VA_ARGS__)
// 定义输出警告日志的宏，调用 server_log 函数输出警告信息
#define LOG_WARNING(MSG, ...) server_log("WARNING", __func__, __LINE__, MSG, __VA_ARGS__)
// 定义输出信息日志的宏，调用 server_log 函数输出一般信息
#define LOG_INFO(   MSG, ...) server_log("INFO",    __func__, __LINE__, MSG, __VA_ARGS__)
//
// base64 utils (TODO: move to common in the future)
//

// 定义 base64 字符集
static const std::string base64_chars =
             "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
             "abcdefghijklmnopqrstuvwxyz"
             "0123456789+/";

// 判断字符是否为 base64 字符
static inline bool is_base64(uint8_t c)
{
    return (isalnum(c) || (c == '+') || (c == '/'));
}

// 对 base64 编码的字符串进行解码
static std::vector<uint8_t> base64_decode(std::string const &encoded_string)
{
    int i = 0;  // 初始化索引 i
    int j = 0;  // 初始化索引 j
    int in_ = 0;  // 初始化输入索引 in_
    // 计算输入编码字符串的长度
    int in_len = encoded_string.size();

    // 创建用于存储字符的数组
    uint8_t char_array_4[4];
    uint8_t char_array_3[3];

    // 创建用于存储解码结果的向量
    std::vector<uint8_t> ret;

    // 循环解码输入的编码字符串
    while (in_len-- && (encoded_string[in_] != '=') && is_base64(encoded_string[in_]))
    {
        // 将输入字符串中的字符存入char_array_4数组中
        char_array_4[i++] = encoded_string[in_]; in_++;
        // 如果char_array_4数组已满，进行解码
        if (i == 4)
        {
            // 将char_array_4数组中的字符转换为对应的base64编码值
            for (i = 0; i <4; i++)
            {
                char_array_4[i] = base64_chars.find(char_array_4[i]);
            }

            // 根据base64编码值计算出解码后的字符值
            char_array_3[0] = ((char_array_4[0]      ) << 2) + ((char_array_4[1] & 0x30) >> 4);
            char_array_3[1] = ((char_array_4[1] & 0xf) << 4) + ((char_array_4[2] & 0x3c) >> 2);
            // 将第四个字符的低两位左移6位，与第三个字符的值相加，得到第三个原始字符的ASCII码
            char_array_3[2] = ((char_array_4[2] & 0x3) << 6) +   char_array_4[3];

            // 将解码后的三个字符依次添加到结果数组中
            for (i = 0; (i < 3); i++)
            {
                ret.push_back(char_array_3[i]);
            }
            // 重置计数器 i
            i = 0;
        }
    }

    // 如果还有剩余的字符需要解码
    if (i)
    {
        // 将剩余的字符补零
        for (j = i; j <4; j++)
        {
            char_array_4[j] = 0;
        }

        // 将剩余的字符转换为对应的索引值
        for (j = 0; j <4; j++)
        {
            char_array_4[j] = base64_chars.find(char_array_4[j]);
        }

        // 将4个字符转换为3个字符
        char_array_3[0] = ((char_array_4[0]      ) << 2) + ((char_array_4[1] & 0x30) >> 4);
        char_array_3[1] = ((char_array_4[1] & 0xf) << 4) + ((char_array_4[2] & 0x3c) >> 2);
        char_array_3[2] = ((char_array_4[2] & 0x3) << 6) +   char_array_4[3];

        // 将转换后的字符加入到结果中
        for (j = 0; (j < i - 1); j++)
        {
            ret.push_back(char_array_3[j]);
        }
    }

    return ret;
}

//
// parallel
//

// 并行任务类型的枚举
enum task_type {
// 定义任务类型枚举，包括完成任务和取消任务
    COMPLETION_TASK,
    CANCEL_TASK
};

// 定义任务服务器结构体，包括任务ID、目标ID、任务类型、数据、填充模式和嵌入模式
struct task_server {
    int id; // 任务ID
    int target_id; // 目标ID
    task_type type; // 任务类型
    json data; // 数据
    bool infill_mode = false; // 填充模式，默认为false
    bool embedding_mode = false; // 嵌入模式，默认为false
};

// 定义任务结果结构体，包括任务ID、停止标志、错误标志、结果JSON
struct task_result {
    int id; // 任务ID
    bool stop; // 停止标志
    bool error; // 错误标志
    json result_json; // 结果JSON
};
// 定义枚举类型 slot_state，表示槽位的状态
enum slot_state
{
    IDLE, // 空闲状态
    PROCESSING, // 处理中状态
};

// 定义枚举类型 slot_command，表示槽位的命令
enum slot_command
{
    NONE, // 无命令
    LOAD_PROMPT, // 加载提示
    RELEASE, // 释放
};

// 定义结构体 slot_params，表示槽位的参数
struct slot_params
{
    bool stream       = true; // 是否使用流
    bool cache_prompt = false; // 是否缓存提示，以避免重新处理所有提示

    uint32_t seed      = -1; // 随机数生成器的种子
    int32_t  n_keep    =  0; // 从初始提示中保留的令牌数量
    int32_t  n_predict = -1; // 预测的新令牌数量，初始值为-1

    std::vector<std::string> antiprompt; // 反向提示的字符串向量

    json input_prefix; // 输入前缀
    json input_suffix; // 输入后缀
};

struct slot_image
{
    int32_t id; // 图像的ID

    bool request_encode_image = false; // 请求编码图像的布尔值，默认为false
    float* image_embedding = nullptr; // 图像嵌入
    int32_t image_tokens = 0; // 图像令牌数量

    clip_image_u8 img_data; // 图像数据

    std::string prefix_prompt; // 图像之前的提示
// 定义了一个结构体 completion_token_output，用于存储完成标记的输出和概率
struct completion_token_output
{
    // 定义了一个内部结构体 token_prob，用于存储标记和其概率
    struct token_prob
    {
        llama_token tok; // 标记
        float prob; // 概率
    };

    std::vector<token_prob> probs; // 存储标记和概率的向量
    llama_token tok; // 标记
    std::string text_to_send; // 要发送的文本
};

// 定义了一个静态函数 common_part，用于找到两个 llama_token 向量的共同部分的大小
static size_t common_part(const std::vector<llama_token> &a, const std::vector<llama_token> &b)
{
    size_t i;
    for (i = 0; i < a.size() && i < b.size() && a[i] == b[i]; i++)
    // 遍历两个向量，找到它们的共同部分的大小
// 返回一个空的字典
{
}
// 返回变量 i 的值
return i;
}

// 枚举类型，表示停止类型
enum stop_type
{
    STOP_FULL, // 完全停止
    STOP_PARTIAL, // 部分停止
};

// 判断字符串是否以指定后缀结尾
static bool ends_with(const std::string &str, const std::string &suffix)
{
    return str.size() >= suffix.size() && // 如果字符串长度大于等于后缀长度
           0 == str.compare(str.size() - suffix.size(), suffix.size(), suffix); // 并且从指定位置开始的子字符串与后缀相同，则返回真
}

// 在文本中查找部分停止字符串的位置
static size_t find_partial_stop_string(const std::string &stop,
                                       const std::string &text)
// 如果文本不为空并且停止词不为空
if (!text.empty() && !stop.empty())
{
    // 获取文本的最后一个字符
    const char text_last_char = text.back();
    // 从停止词的最后一个字符开始向前遍历
    for (int64_t char_index = stop.size() - 1; char_index >= 0; char_index--)
    {
        // 如果停止词的当前字符与文本的最后一个字符相同
        if (stop[char_index] == text_last_char)
        {
            // 获取停止词的部分子串
            const std::string current_partial = stop.substr(0, char_index + 1);
            // 如果文本以当前部分子串结尾
            if (ends_with(text, current_partial))
            {
                // 返回文本长度减去当前部分子串的长度减一
                return text.size() - char_index - 1;
            }
        }
    }
}
// 如果未找到匹配的部分子串，返回npos
return std::string::npos;
}

// TODO: 重用 llama_detokenize
template <class Iter>
// 将 tokens 转换为字符串
static std::string tokens_to_str(llama_context *ctx, Iter begin, Iter end)
{
    std::string ret;
    // 遍历 tokens，将每个 token 转换为字符串并拼接到 ret 中
    for (; begin != end; ++begin)
    {
        ret += llama_token_to_piece(ctx, *begin);
    }
    // 返回拼接后的字符串
    return ret;
}

// 记录服务器日志
static void server_log(const char *level, const char *function, int line,
                       const char *message, const nlohmann::ordered_json &extra)
{
    // 创建日志对象
    nlohmann::ordered_json log
    {
        // 添加时间戳
        {"timestamp", time(nullptr)},
        // 添加日志级别
        {"level",     level},
        // 添加函数名
        {"function",  function},
        // 添加行号
        {"line",      line},
        // 添加消息内容
        {"message",   message},
```
**注意：** 以上代码片段并不完整，缺少了一些语句和括号的闭合。在实际情况中，需要根据完整的代码来添加注释。
    };

    // 如果额外信息不为空，将其合并到日志中
    if (!extra.empty())
    {
        log.merge_patch(extra);
    }

    // 将日志转换为字符串格式
    const std::string str = log.dump(-1, ' ', false, json::error_handler_t::replace);
    // 将字符串格式的日志输出到标准输出
    printf("%.*s\n", (int)str.size(), str.data());
    fflush(stdout);
}

// 格式化不完整的 UTF-8 多字节字符以便输出
static std::string tokens_to_output_formatted_string(const llama_context *ctx, const llama_token token)
{
    // 将 token 转换为对应的字符串
    std::string out = token == -1 ? "" : llama_token_to_piece(ctx, token);
    // 如果字符串长度为1且第一个比特位为1，表示这是一个不完整的字符
    //   (长度大于1表示已经是一个已知的 token)
    if (out.size() == 1 && (out[0] & 0x80) == 0x80)
    {
// 创建一个字符串流对象
std::stringstream ss;
// 将out[0]的值转换为十六进制，并存入字符串流
ss << std::hex << (out[0] & 0xff);
// 将字符串流中的内容转换为字符串
std::string res(ss.str());
// 将"byte: \\x"和res拼接成新的字符串，存入out
out = "byte: \\x" + res;
// 返回out
}

// 将completion_token_output的vector转换为json格式
static json probs_vector_to_json(const llama_context *ctx, const std::vector<completion_token_output> &probs)
{
    // 创建一个json数组对象
    json out = json::array();
    // 遍历probs中的每个completion_token_output对象
    for (const auto &prob : probs)
    {
        // 创建一个json数组对象
        json probs_for_token = json::array();
        // 遍历prob中的每个probs对象
        for (const auto &p : prob.probs)
        {
            // 将p.tok转换为格式化字符串
            std::string tok_str = tokens_to_output_formatted_string(ctx, p.tok);
            // 将格式化后的字符串存入probs_for_token数组
            probs_for_token.push_back(json
            {
// 创建一个包含键值对 { "tok_str", tok_str } 和 { "prob", p.prob } 的 JSON 对象，并添加到数组中
// 这里的 tok_str 和 p.prob 应该是之前定义过的变量或者函数返回值
out.push_back(json{
    {"tok_str", tok_str},
    {"prob",    p.prob},
});

// 根据上下文和概率生成格式化的字符串，并添加到数组中
// 这里的 ctx 和 prob.tok 应该是之前定义过的变量
std::string tok_str = tokens_to_output_formatted_string(ctx, prob.tok);
out.push_back(json{
    {"content", tok_str},
    {"probs",   probs_for_token},
});

// 返回最终的 JSON 数组
return out;
}

// 从 JSON 对象中获取指定键的值，如果键不存在或者值为 null，则返回默认值
template <typename T>
static T json_value(const json &body, const std::string &key, const T &default_value)
{
    // 如果 JSON 对象中包含指定键并且值不为 null，则返回对应值，否则返回默认值
    return body.contains(key) && !body.at(key).is_null()
        ? body.value(key, default_value)
        : default_value;
}
// 定义了一个名为 llama_client_slot 的结构体
struct llama_client_slot
{
    int id; // 用于存储客户端槽位的ID
    int task_id = -1; // 用于存储任务ID，默认值为-1

    struct slot_params params; // 存储槽位参数的结构体变量

    slot_state state = IDLE; // 用于存储槽位状态，默认值为IDLE
    slot_command command = NONE; // 用于存储槽位命令，默认值为NONE

    // 用于确定最后一次使用的槽位
    int64_t t_last_used = -1; // 存储最后一次使用的时间，默认值为-1

    // 用于存储生成属性
    int32_t n_ctx       = 0;  // 每个槽位的上下文大小
    int32_t n_past      = 0;  // 过去的大小
    int32_t n_decoded   = 0;  // 已解码的大小
    int32_t n_remaining = -1; // 剩余大小，默认值为-1
    // 初始化批次索引为-1
    int32_t i_batch = -1;

    // 初始化提示标记数量为0
    int32_t num_prompt_tokens = 0;
    // 初始化已处理的提示标记数量为0
    int32_t num_prompt_tokens_processed = 0;
    // 初始化多字节待处理标记为0
    int32_t multibyte_pending = 0;

    // 初始化 JSON 对象 prompt
    json prompt;
    // 初始化生成的文本字符串
    std::string generated_text;
    // 初始化采样的 token
    llama_token sampled;
    // 初始化缓存 token 的向量
    std::vector<llama_token> cache_tokens;
    // 初始化生成的 token 概率的向量
    std::vector<completion_token_output> generated_token_probs;

    // 初始化填充标志为假
    bool infill = false;
    // 初始化嵌入标志为假
    bool embedding = false;
    // 初始化是否有下一个 token 的标志为真
    bool has_next_token = true;
    // 初始化是否截断的标志为假
    bool truncated = false;
    // 初始化是否停止 EOS 的标志为假
    bool stopped_eos = false;
    // 初始化是否停止单词的标志为假
    bool stopped_word = false;
    // 初始化是否停止限制的标志为假
    bool stopped_limit = false;
    // 声明一个字符串变量用于存储停用词
    std::string stopping_word;

    // 声明一个结构体变量用于存储采样参数，以及一个指向采样上下文的空指针
    struct llama_sampling_params sparams;
    llama_sampling_context *ctx_sampling = nullptr;

    // 声明一个存储图像槽的向量
    std::vector<slot_image> images;

    // 声明两个统计变量，用于记录句子数量和句子中的词概率索引
    size_t sent_count = 0;
    size_t sent_token_probs_index = 0;

    // 声明两个记录时间的变量，用于记录处理提示和生成标记的开始时间
    int64_t t_start_process_prompt;
    int64_t t_start_genereration;

    // 声明两个记录时间的变量，用于记录处理提示和生成标记的时间（以毫秒为单位）
    double t_prompt_processing; // ms
    double t_token_generation; // ms

    // 重置函数，用于重置所有变量的值
    void reset() {
# 初始化变量，设置初始值
num_prompt_tokens      = 0;  # 提示符令牌数量
generated_text         = "";  # 生成的文本
truncated              = false;  # 是否被截断
stopped_eos            = false;  # 是否停止在句尾
stopped_word           = false;  # 是否停止在单词
stopped_limit          = false;  # 是否达到停止限制
stopping_word          = "";  # 停止的单词
multibyte_pending      = 0;  # 多字节挂起
n_past                 = 0;  # 过去的数量
sent_count             = 0;  # 句子计数
sent_token_probs_index = 0;  # 句子令牌概率索引
infill                 = false;  # 是否填充

# 清空生成的令牌概率
generated_token_probs.clear();

# 遍历图像列表
for (slot_image &img : images)
{
    # 释放图像嵌入
    free(img.image_embedding);
    # 删除图像数据
    delete[] img.img_data.data;
    # 重置图像前缀提示
    img.prefix_prompt = "";
}
        }

        // 清空 images 容器
        images.clear();
        // 在批处理中种子是否重要？ llama_set_rng_seed(ctx, params.seed); 
    }

    // 检查是否有剩余预测次数
    bool has_budget(gpt_params &global_params) {
        n_remaining = -1;
        // 如果参数中指定了预测次数，则计算剩余次数
        if(params.n_predict != -1)
        {
            n_remaining = params.n_predict - n_decoded;
        }
        // 如果全局参数中指定了预测次数，则计算剩余次数
        else if (global_params.n_predict != -1)
        {
            n_remaining = global_params.n_predict - n_decoded;
        }
        // 返回剩余次数是否大于0或者为-1（无限次数）
        return n_remaining > 0 || n_remaining == -1; // no budget || limitless
    }

    // 检查是否可用
    bool available() const {
        return state == IDLE && command == NONE;
    }
    // 检查状态是否为IDLE并且命令是否为NONE，返回布尔值

    bool is_processing() const {
        return (state == IDLE && command == LOAD_PROMPT) || state == PROCESSING;
    }
    // 检查状态是否为IDLE并且命令是否为LOAD_PROMPT，或者状态是否为PROCESSING，返回布尔值

    void add_token_string(const completion_token_output &token) {
        if (command == RELEASE)
        {
            return;
        }
        // 如果命令为RELEASE，则直接返回，不执行后续操作
        cache_tokens.push_back(token.tok);
        generated_token_probs.push_back(token);
    }
    // 将token添加到cache_tokens和generated_token_probs中

    void release() {
        if (state == IDLE || state == PROCESSING)
        {
            t_token_generation = (ggml_time_us() - t_start_genereration) / 1e3;
        }
    }
    // 如果状态为IDLE或者PROCESSING，则计算t_token_generation的值
    // 设置命令为RELEASE
    command = RELEASE;
}

json get_formated_timings() {
    // 返回格式化后的时间数据
    return json
    {
        // 返回处理的提示标记数量
        {"prompt_n",               num_prompt_tokens_processed},
        // 返回处理提示的毫秒数
        {"prompt_ms",              t_prompt_processing},
        // 返回每个提示标记的平均处理时间
        {"prompt_per_token_ms",    t_prompt_processing / num_prompt_tokens_processed},
        // 返回每秒处理的提示标记数量
        {"prompt_per_second",      1e3 / t_prompt_processing * num_prompt_tokens_processed},

        // 返回解码的数量
        {"predicted_n",            n_decoded},
        // 返回生成标记的毫秒数
        {"predicted_ms",           t_token_generation},
        // 返回每个解码标记的平均生成时间
        {"predicted_per_token_ms", t_token_generation / n_decoded},
        // 返回每秒生成的解码标记数量
        {"predicted_per_second",   1e3 / t_token_generation * n_decoded},
    };
}

void print_timings() {
// 打印一个换行符
LOG_TEE("\n");
// 打印提示评估时间、处理的标记数量、每个标记的平均时间、每秒处理的标记数量
LOG_TEE("%s: prompt eval time = %10.2f ms / %5d tokens (%8.2f ms per token, %8.2f tokens per second)\n",
    __func__, t_prompt_processing, num_prompt_tokens_processed, t_prompt_processing / num_prompt_tokens_processed, 1e3 / t_prompt_processing * num_prompt_tokens_processed);
// 打印评估时间、运行次数、每个标记的平均时间、每秒生成的标记数量
LOG_TEE("%s:        eval time = %10.2f ms / %5d runs   (%8.2f ms per token, %8.2f tokens per second)\n",
    __func__, t_token_generation, n_decoded,t_token_generation / n_decoded, 1e3 / t_token_generation * n_decoded);
// 打印总时间
LOG_TEE("%s:       total time = %10.2f ms\n", __func__, t_prompt_processing + t_token_generation);
};

// 定义一个名为 llama_server_context 的结构体
struct llama_server_context
{
    // 指向 llama_model 对象的指针，默认为空
    llama_model *model = nullptr;
    // 指向 llama_context 对象的指针，默认为空
    llama_context *ctx = nullptr;
    // 指向 clip_ctx 对象的指针，默认为空
    clip_ctx *clp_ctx = nullptr;
    // 存储 gpt_params 对象
    gpt_params params;
    // 存储 llama_batch 对象
    llama_batch batch;
    // 声明并初始化布尔变量，表示是否为多模态
    bool multimodal         = false;
    // 声明并初始化布尔变量，表示是否清空键值缓存
    bool clean_kv_cache     = true;
    // 声明并初始化布尔变量，表示所有插槽是否空闲
    bool all_slots_are_idle = false;

    // 声明整型变量，表示 ID 生成器
    int32_t id_gen;
    // 声明整型变量，表示所有客户端/插槽的总上下文
    int32_t n_ctx;  

    // 系统提示
    // 声明并初始化布尔变量，表示系统是否需要更新
    bool system_need_update = false;

    // 声明字符串变量，表示系统提示
    std::string              system_prompt;
    // 声明 llama_token 类型的向量，表示系统提示的令牌
    std::vector<llama_token> system_tokens;

    // 用户名称
    // 声明字符串变量，表示用户名称（应该是反提示）
    std::string name_user;      
    // 声明字符串变量，表示助手名称
    std::string name_assistant;

    // 插槽/客户端
    // 声明 llama_client_slot 类型的向量，表示插槽
    std::vector<llama_client_slot> slots;

    // 任务队列
    // 声明 task_server 类型的向量，表示任务队列
    std::vector<task_server> queue_tasks;
    // 创建一个存储任务结果的队列
    std::vector<task_result> queue_results;
    // 创建用于保护任务队列的互斥锁
    std::mutex mutex_tasks;
    // 创建用于保护结果队列的互斥锁
    std::mutex mutex_results;

    // 析构函数，用于释放内存
    ~llama_server_context()
    {
        // 如果上下文对象存在，则释放内存并将指针置为空
        if (ctx)
        {
            llama_free(ctx);
            ctx = nullptr;
        }
        // 如果模型对象存在，则释放内存并将指针置为空
        if (model)
        {
            llama_free_model(model);
            model = nullptr;
        }
    }

    // 加载模型的方法
    bool load_model(const gpt_params &params_)
    {
        // 将参数赋值给变量params
        params = params_;
        // 如果参数中的mmproj不为空
        if (!params.mmproj.empty()) {
            // 设置多模态为true
            multimodal = true;
            // 输出日志，表示多模态模式已启用
            LOG_TEE("Multi Modal Mode Enabled");
            // 加载clip模型
            clp_ctx = clip_model_load(params.mmproj.c_str(), /*verbosity=*/ 1);
            // 如果加载失败
            if(clp_ctx == nullptr) {
                // 输出错误日志，表示无法加载clip模型
                LOG_ERROR("unable to load clip model", {{"model", params.mmproj}});
                // 返回false
                return false;
            }

            // 如果参数中的n_ctx小于2048
            if (params.n_ctx < 2048) { // request larger context for the image embedding
                // 将参数中的n_ctx设置为2048
                params.n_ctx = 2048;
            }
        }

        // 从gpt参数中初始化模型和上下文
        std::tie(model, ctx) = llama_init_from_gpt_params(params);
        // 如果模型为空
        if (model == nullptr)
        {
            // 输出错误日志，表示无法加载模型
            LOG_ERROR("unable to load model", {{"model", params.model}});
            // 返回false
            return false;
    }

    // 如果是多模态数据
    if (multimodal) {
        // 获取多模态投影器的嵌入维度
        const int n_embd_clip = clip_n_mmproj_embd(clp_ctx);
        // 获取LLaMA模型的嵌入维度
        const int n_embd_llm  = llama_n_embd(model);
        // 如果两者嵌入维度不相等
        if (n_embd_clip != n_embd_llm) {
            // 输出错误信息
            LOG_TEE("%s: embedding dim of the multimodal projector (%d) is not equal to that of LLaMA (%d). Make sure that you use the correct mmproj file.\n", __func__, n_embd_clip, n_embd_llm);
            // 释放内存
            llama_free(ctx);
            llama_free_model(model);
            // 返回false
            return false;
        }
    }

    // 获取上下文的数量
    n_ctx = llama_n_ctx(ctx);

    // 返回true
    return true;
}

// 初始化函数
void initialize() {
    // 初始化id_gen为0
    id_gen = 0;
// 创建插槽
all_slots_are_idle = true;  // 初始化所有插槽为空闲状态

const int32_t n_ctx_slot = n_ctx / params.n_parallel;  // 计算每个插槽的最大上下文数

LOG_TEE("Available slots:\n");  // 记录可用插槽信息
for (int i = 0; i < params.n_parallel; i++)  // 遍历每个并行插槽
{
    llama_client_slot slot;  // 创建插槽对象

    slot.id = i;  // 设置插槽ID
    slot.n_ctx = n_ctx_slot;  // 设置插槽的最大上下文数
    slot.reset();  // 重置插槽状态

    LOG_TEE(" -> Slot %i - max context: %i\n", slot.id, n_ctx_slot);  // 记录插槽ID和最大上下文数
    slots.push_back(slot);  // 将插槽添加到插槽列表中
}

batch = llama_batch_init(n_ctx, 0, params.n_parallel);  // 初始化批处理
        // 清空系统提示
        system_prompt = "";
        // 清空系统标记
        system_tokens.clear();
    }

    // 将 JSON 格式的提示转换为令牌序列
    std::vector<llama_token> tokenize(const json & json_prompt, bool add_bos) const
    {
        // 如果 `add_bos` 为 true，当 json_prompt 是字符串，或者 json_prompt 数组的第一个元素是字符串时，只添加 BOS
        std::vector<llama_token> prompt_tokens;

        // 如果 json_prompt 是数组
        if (json_prompt.is_array())
        {
            // 标记是否是数组的第一个元素
            bool first = true;
            // 遍历数组中的每个元素
            for (const auto& p : json_prompt)
            {
                // 如果元素是字符串
                if (p.is_string())
                {
                    // 获取字符串值
                    auto s = p.template get<std::string>();
// 声明一个名为 p 的 llama_token 类型的向量
std::vector<llama_token> p;
// 如果是第一次循环
if (first)
{
    // 调用 llama_tokenize 函数对字符串 s 进行分词，并将结果存储在 p 中，同时添加 bos
    p = ::llama_tokenize(ctx, s, add_bos);
    // 将 first 设置为 false
    first = false;
}
// 如果不是第一次循环
else
{
    // 调用 llama_tokenize 函数对字符串 s 进行分词，并将结果存储在 p 中，不添加 bos
    p = ::llama_tokenize(ctx, s, false);
}
// 将 p 中的元素添加到 prompt_tokens 的末尾
prompt_tokens.insert(prompt_tokens.end(), p.begin(), p.end());
// 如果不是第一次循环
else
{
    // 如果是第一次循环，将 first 设置为 false
    if (first)
    {
        first = false;
    }
    // 将 p 中的元素添加到 prompt_tokens 的末尾
    prompt_tokens.push_back(p.template get<llama_token>());
}
        }
        }
        else
        {
            // 如果输入的 JSON 数据不是数组类型，则尝试获取字符串类型数据
            auto s = json_prompt.template get<std::string>();
            // 使用 llama_tokenize 函数对字符串进行分词处理
            prompt_tokens = ::llama_tokenize(ctx, s, add_bos);
        }

        // 返回处理后的分词结果
        return prompt_tokens;
    }

    // 根据 ID 获取指定的槽位对象
    llama_client_slot* get_slot(int id) {
        // 获取当前时间
        int64_t t_last = ggml_time_us();
        // 初始化最后使用的槽位为空
        llama_client_slot *last_used = nullptr;

        // 遍历槽位列表
        for (llama_client_slot & slot : slots)
        {
            // 如果槽位 ID 匹配且槽位可用
            if (slot.id == id && slot.available())
            {
                // 返回匹配的槽位对象
                return &slot;
        }
        // 如果槽位可用且上次使用时间早于 t_last，则更新最后使用的槽位和时间
        if (slot.available() && slot.t_last_used < t_last)
        {
            last_used = &slot;
            t_last = slot.t_last_used;
        }
    }

    // 返回最后使用的槽位
    return last_used;
}

// 使用给定数据启动槽位
bool launch_slot_with_data(llama_client_slot* &slot, json data) {
    // 初始化默认参数和采样参数
    slot_params default_params;
    llama_sampling_params default_sparams;

    // 从给定数据中获取流、缓存提示、预测数量等参数，并赋值给槽位的参数
    slot->params.stream           = json_value(data, "stream",            false);
    slot->params.cache_prompt     = json_value(data, "cache_prompt",      false);
    slot->params.n_predict        = json_value(data, "n_predict",         default_params.n_predict);
    slot->sparams.top_k           = json_value(data, "top_k",             default_sparams.top_k);
        # 设置slot对象的top_p参数为json数据中的top_p值，如果json数据中没有top_p值，则使用默认值
        slot->sparams.top_p           = json_value(data, "top_p",             default_sparams.top_p);
        # 设置slot对象的min_p参数为json数据中的min_p值，如果json数据中没有min_p值，则使用默认值
        slot->sparams.min_p           = json_value(data, "min_p",             default_sparams.min_p);
        # 设置slot对象的tfs_z参数为json数据中的tfs_z值，如果json数据中没有tfs_z值，则使用默认值
        slot->sparams.tfs_z           = json_value(data, "tfs_z",             default_sparams.tfs_z);
        # 设置slot对象的typical_p参数为json数据中的typical_p值，如果json数据中没有typical_p值，则使用默认值
        slot->sparams.typical_p       = json_value(data, "typical_p",         default_sparams.typical_p);
        # 设置slot对象的temp参数为json数据中的temperature值，如果json数据中没有temperature值，则使用默认值
        slot->sparams.temp            = json_value(data, "temperature",       default_sparams.temp);
        # 设置slot对象的penalty_last_n参数为json数据中的repeat_last_n值，如果json数据中没有repeat_last_n值，则使用默认值
        slot->sparams.penalty_last_n  = json_value(data, "repeat_last_n",     default_sparams.penalty_last_n);
        # 设置slot对象的penalty_repeat参数为json数据中的repeat_penalty值，如果json数据中没有repeat_penalty值，则使用默认值
        slot->sparams.penalty_repeat  = json_value(data, "repeat_penalty",    default_sparams.penalty_repeat);
        # 设置slot对象的penalty_freq参数为json数据中的frequency_penalty值，如果json数据中没有frequency_penalty值，则使用默认值
        slot->sparams.penalty_freq    = json_value(data, "frequency_penalty", default_sparams.penalty_freq);
        # 设置slot对象的penalty_present参数为json数据中的presence_penalty值，如果json数据中没有presence_penalty值，则使用默认值
        slot->sparams.penalty_present = json_value(data, "presence_penalty",  default_sparams.penalty_present);
        # 设置slot对象的mirostat参数为json数据中的mirostat值，如果json数据中没有mirostat值，则使用默认值
        slot->sparams.mirostat        = json_value(data, "mirostat",          default_sparams.mirostat);
        # 设置slot对象的mirostat_tau参数为json数据中的mirostat_tau值，如果json数据中没有mirostat_tau值，则使用默认值
        slot->sparams.mirostat_tau    = json_value(data, "mirostat_tau",      default_sparams.mirostat_tau);
        # 设置slot对象的mirostat_eta参数为json数据中的mirostat_eta值，如果json数据中没有mirostat_eta值，则使用默认值
        slot->sparams.mirostat_eta    = json_value(data, "mirostat_eta",      default_sparams.mirostat_eta);
        # 设置slot对象的penalize_nl参数为json数据中的penalize_nl值，如果json数据中没有penalize_nl值，则使用默认值
        slot->sparams.penalize_nl     = json_value(data, "penalize_nl",       default_sparams.penalize_nl);
        # 设置slot对象的n_keep参数为json数据中的n_keep值
        slot->params.n_keep           = json_value(data, "n_keep",            slot->params.n_keep);
        # 设置slot对象的seed参数为json数据中的seed值，如果json数据中没有seed值，则使用默认值
        slot->params.seed             = json_value(data, "seed",              default_params.seed);
        # 设置slot对象的grammar参数为json数据中的grammar值，如果json数据中没有grammar值，则使用默认值
        slot->sparams.grammar         = json_value(data, "grammar",           default_sparams.grammar);
        # 设置slot对象的n_probs参数为json数据中的n_probs值，如果json数据中没有n_probs值，则使用默认值
        slot->sparams.n_probs         = json_value(data, "n_probs",           default_sparams.n_probs);

        // infill
        # 如果json数据中包含input_prefix，则执行以下操作
        if (data.count("input_prefix") != 0)
# 如果数据中存在键 "input_prefix"，则将其值赋给 slot 对象的 params.input_prefix，否则赋空字符串
        {
            slot->params.input_prefix = data["input_prefix"];
        }
        else
        {
            slot->params.input_prefix = "";
        }

# 如果数据中存在键 "input_suffix"，则将其值赋给 slot 对象的 params.input_suffix，否则赋空字符串
        if (data.count("input_suffix") != 0)
        {
            slot->params.input_suffix = data["input_suffix"];
        }
        else
        {
            slot->params.input_suffix = "";
        }

# 如果数据中存在键 "prompt"，则将其值赋给 slot 对象的 prompt
        if (data.count("prompt") != 0)
        {
            slot->prompt = data["prompt"];
        }
        else
        {
            // 如果条件不满足，将 slot 的 prompt 设为空字符串
            slot->prompt = "";
        }

        // 清空 logit_bias
        slot->sparams.logit_bias.clear();

        // 如果 JSON 数据中包含 ignore_eos 并且其值为 false
        if (json_value(data, "ignore_eos", false))
        {
            // 将 logit_bias 中 EOS 标记的偏置设为负无穷
            slot->sparams.logit_bias[llama_token_eos(model)] = -INFINITY;
        }

        // 查找 JSON 数据中是否包含 logit_bias
        const auto &logit_bias = data.find("logit_bias");
        // 如果找到并且其值是数组
        if (logit_bias != data.end() && logit_bias->is_array())
        {
            // 获取词汇表大小
            const int n_vocab = llama_n_vocab(model);
            // 遍历 logit_bias 数组
            for (const auto &el : *logit_bias)
            {
                // 如果元素是数组且长度为2且第一个元素是整数
// 遍历数据列表
{
    // 从列表中获取 llama_token 类型的元素
    llama_token tok = el[0].get<llama_token>();
    // 如果 tok 的值在有效范围内
    if (tok >= 0 && tok < n_vocab)
    {
        // 如果第二个元素是数字
        if (el[1].is_number())
        {
            // 将 tok 对应的偏置值设置为第二个元素的值
            slot->sparams.logit_bias[tok] = el[1].get<float>();
        }
        // 如果第二个元素是布尔类型且为假
        else if (el[1].is_boolean() && !el[1].get<bool>())
        {
            // 将 tok 对应的偏置值设置为负无穷大
            slot->sparams.logit_bias[tok] = -INFINITY;
        }
    }
}

// 清空 slot 的 antiprompt 参数
slot->params.antiprompt.clear();

// 在数据中查找 "stop" 键对应的值
const auto &stop = data.find("stop");
        # 检查是否存在名为 "stop" 的键，并且其对应的值是一个数组
        if (stop != data.end() && stop->is_array())
        {
            # 遍历数组中的每个元素
            for (const auto &word : *stop)
            {
                # 如果元素不为空
                if (!word.empty())
                {
                    # 将元素添加到 slot 对象的 params.antiprompt 数组中
                    slot->params.antiprompt.push_back(word);
                }
            }
        }

        # 如果 multimodal 为真
        if (multimodal)
        {
            # 查找名为 "image_data" 的键对应的值
            const auto &images_data = data.find("image_data");
            # 如果存在 "image_data" 键，并且其对应的值是一个数组
            if (images_data != data.end() && images_data->is_array())
            {
                # 遍历数组中的每个元素
                for (const auto &img : *images_data)
                {
                    # 获取元素中名为 "data" 的字符串值
                    std::string data_b64 = img["data"].get<std::string>();
                    # 创建一个名为 img_sl 的 slot_image 对象
                    slot_image img_sl;
// 设置图像的 id，如果图像中包含 id，则使用该 id，否则使用当前 slot 中的图像数量作为 id
img_sl.id = img.count("id") != 0 ? img["id"].get<int>() : slot->images.size();

// 定义图像的宽度、高度和通道数
int width, height, channels;

// 将 base64 编码的图像数据解码成 uint8_t 类型的图像缓冲区
std::vector<uint8_t> image_buffer = base64_decode(data_b64);

// 清空 base64 编码的图像数据
data_b64.clear();

// 使用 stb_image 库从内存中加载图像数据
auto data = stbi_load_from_memory(image_buffer.data(), image_buffer.size(), &width, &height, &channels, 3);

// 如果加载图像数据失败，则记录日志并返回 false
if (!data) {
    LOG_TEE("slot %i - failed to load image [id: %i]\n", slot->id, img_sl.id);
    return false;
}

// 记录日志，表示图像加载成功，并记录图像的分辨率
LOG_TEE("slot %i - image loaded [id: %i] resolution (%i x %i)\n", slot->id, img_sl.id, width, height);

// 设置图像数据的宽度、高度和大小
img_sl.img_data.nx = width;
img_sl.img_data.ny = height;
img_sl.img_data.size = width * height * 3;

// 分配内存并拷贝图像数据
img_sl.img_data.data = new uint8_t[width * height * 3]();
memcpy(img_sl.img_data.data, data, width * height * 3);

// 释放 stb_image 加载的图像数据
stbi_image_free(data);

// 设置请求对图像进行编码
img_sl.request_encode_image = true;

// 将图像添加到当前 slot 的图像列表中
slot->images.push_back(img_sl);
// 检查槽中的图片列表是否大于0，并且提示不是数组类型
if (slot->images.size() > 0 && !slot->prompt.is_array())
{
    // 将提示转换为字符串
    std::string prompt = slot->prompt.get<std::string>();
    size_t pos = 0, begin_prefix = 0;
    // 设置要查找的模式
    std::string pattern = "[img-";
    // 在提示中查找模式，提取图片ID和前缀
    while ((pos = prompt.find(pattern, pos)) != std::string::npos) {
        size_t end_prefix = pos;
        pos += pattern.length();
        // 查找图片ID的结束位置
        size_t end_pos = prompt.find("]", pos);
        if (end_pos != std::string::npos)
        {
            // 提取图片ID
            std::string image_id = prompt.substr(pos, end_pos - pos);
            try
            {
                // 将图片ID转换为整数
                int img_id = std::stoi(image_id);
                bool found = false;
                // 遍历槽中的图片列表，查找与图片ID匹配的图片
                for (slot_image &img : slot->images)
                {
                    if (img.id == img_id) {
# 初始化变量 found 为 true
found = true;
# 将 img 对象的前缀提示设置为 prompt 字符串中指定位置的子串
img.prefix_prompt = prompt.substr(begin_prefix, end_prefix - begin_prefix);
# 更新 begin_prefix 的值为 end_pos + 1
begin_prefix = end_pos + 1;
# 跳出循环
break;
# 如果未找到指定条件的图片，则输出错误信息并清空 slot 对象的 images 属性，返回 false
LOG_TEE("ERROR: Image with id: %i, not found.\n", img_id);
slot->images.clear();
return false;
# 捕获异常，输出错误信息并清空 slot 对象的 images 属性，返回 false
LOG_TEE("Invalid image number id in prompt\n");
slot->images.clear();
return false;
# 清空 slot 对象的 prompt 属性
slot->prompt = "";
# 将 slot 对象的 params.input_suffix 属性设置为 prompt 字符串中指定位置的子串
slot->params.input_suffix = prompt.substr(begin_prefix);
        slot->params.cache_prompt = false; // 禁用缓存提示，因为多模态不支持缓存提示
    }
}
}

if (slot->ctx_sampling != nullptr)
{
    llama_sampling_free(slot->ctx_sampling); // 释放上下文采样
}
slot->ctx_sampling = llama_sampling_init(slot->sparams); // 初始化上下文采样
slot->command = LOAD_PROMPT; // 设置命令为加载提示

all_slots_are_idle = false; // 设置所有槽位为非空闲状态

LOG_TEE("slot %i is processing [task id: %i]\n", slot->id, slot->task_id); // 记录槽位正在处理的任务信息

return true; // 返回true表示处理成功
}

void kv_cache_clear() { // 清空键值缓存
// 清空整个 KV 缓存
llama_kv_cache_clear(ctx);
// 设置清空 KV 缓存的标志为 false
clean_kv_cache = false;
}

void update_system_prompt() {
    // 使用 llama_tokenize 函数对系统提示进行分词
    system_tokens = ::llama_tokenize(ctx, system_prompt, true);

    // 清空批处理对象
    llama_batch_clear(batch);

    // 清空 KV 缓存
    kv_cache_clear();

    // 遍历系统提示的分词结果，将每个分词添加到批处理对象中
    for (int i = 0; i < (int) system_tokens.size(); ++i)
    {
        llama_batch_add(batch, system_tokens[i], i, { 0 }, false);
    }

    // 如果 llama_decode 函数返回非零值，记录错误日志
    if (llama_decode(ctx, batch) != 0)
    {
        LOG_TEE("%s: llama_decode() failed\n", __func__);
        // 如果条件不满足，直接返回
        return;
    }

    // 将系统 KV 缓存分配给所有并行序列
    for (int32_t i = 1; i < params.n_parallel; ++i)
    {
        llama_kv_cache_seq_cp(ctx, 0, i, 0, system_tokens.size());
    }

    // 记录系统提示已更新
    LOG_TEE("system prompt updated\n");
    system_need_update = false;
}

// 通知系统提示已更改
void notify_system_prompt_changed() {
    // 释放所有槽位
    for (llama_client_slot &slot : slots)
    {
        slot.release();
    }
    // 设置系统需要更新的标志为true
    system_need_update = true;
}

// 处理系统提示数据
void process_system_prompt_data(const json &sys_props) {
    // 获取系统提示信息
    system_prompt  = sys_props.value("prompt", "");
    // 获取用户名称
    name_user      = sys_props.value("anti_prompt", "");
    // 获取助手名称
    name_assistant = sys_props.value("assistant_name", "");

    // 如果槽的数量大于0
    if (slots.size() > 0)
    {
        // 通知系统提示已更改
        notify_system_prompt_changed();
    }
}

// 查找停止字符串的位置
static size_t find_stopping_strings(const std::string &text, const size_t last_token_size,
                                    const stop_type type, llama_client_slot &slot)
{
    // 初始化停止位置
    size_t stop_pos = std::string::npos;

    // 遍历槽参数中的反提示词
    for (const std::string &word : slot.params.antiprompt)
// 声明一个变量 pos 用于存储位置信息
size_t pos;
// 如果类型为 STOP_FULL
if (type == STOP_FULL)
{
    // 计算 word 的长度加上 last_token_size 的值
    const size_t tmp = word.size() + last_token_size;
    // 计算 from_pos 的值，如果 text 的长度大于 tmp，则为 text.size() - tmp，否则为 0
    const size_t from_pos = text.size() > tmp ? text.size() - tmp : 0;
    // 在 text 中查找 word，起始位置为 from_pos
    pos = text.find(word, from_pos);
}
// 如果类型不为 STOP_FULL
else
{
    // 调用 find_partial_stop_string 函数在 text 中查找部分停止字符串 word
    pos = find_partial_stop_string(word, text);
}
// 如果找到了位置并且 stop_pos 为无效位置，或者找到的位置小于 stop_pos
if (pos != std::string::npos &&
    (stop_pos == std::string::npos || pos < stop_pos))
{
    // 如果类型为 STOP_FULL
    if (type == STOP_FULL)
    {
        // 设置 slot 的 stopped_word 为 true
        slot.stopped_word = true;
        // 设置 slot 的 stopping_word 为 word
        slot.stopping_word = word;
        // 设置 slot 的 has_next_token 为 false
        slot.has_next_token = false;
        }
        // 记录被采样的令牌，用于采样过程中的重复惩罚
        const std::string token_str = llama_token_to_piece(ctx, result.tok);
        // 将被采样的令牌保存到 slot 中
        slot.sampled = result.tok;

        // 将令牌字符串添加到生成的文本中
        slot.generated_text += token_str;
        // 标记还有下一个令牌
        slot.has_next_token = true;

        // 如果有多字节待处理，则减去已处理的字节数
        if (slot.multibyte_pending > 0)
        {
            slot.multibyte_pending -= token_str.size();
        }
        else if (token_str.size() == 1)
        {
            // 如果 token_str 只包含一个字符
            const char c = token_str[0];
            // 检查是否为 2 字节字符：110xxxxx 10xxxxxx
            if ((c & 0xE0) == 0xC0)
            {
                // 设置待处理的多字节字符为 1
                slot.multibyte_pending = 1;
            }
            // 检查是否为 3 字节字符：1110xxxx 10xxxxxx 10xxxxxx
            else if ((c & 0xF0) == 0xE0)
            {
                // 设置待处理的多字节字符为 2
                slot.multibyte_pending = 2;
            }
            // 检查是否为 4 字节字符：11110xxx 10xxxxxx 10xxxxxx 10xxxxxx
            else if ((c & 0xF8) == 0xF0)
            {
                // 设置待处理的多字节字符为 3
                slot.multibyte_pending = 3;
            }
            else
        {
            // 重置多字节处理标志位为0
            slot.multibyte_pending = 0;
        }
    }

    // 如果多字节处理标志位为0
    if (slot.multibyte_pending == 0)
    {
        // 计算位置，取slot.sent_count和slot.generated_text.size()的最小值
        size_t pos = std::min(slot.sent_count, slot.generated_text.size());
        // 从生成的文本中截取子串
        const std::string str_test = slot.generated_text.substr(pos);
        // 初始化停止标志为false
        bool is_stop_full = false;
        // 查找停止字符串的位置
        size_t stop_pos = find_stopping_strings(str_test, token_str.size(), STOP_FULL, slot);
        // 如果找到停止字符串
        if (stop_pos != std::string::npos)
        {
            // 设置停止标志为true
            is_stop_full = true;
            // 从生成的文本中删除指定位置之后的内容
            slot.generated_text.erase(
                slot.generated_text.begin() + pos + stop_pos,
                slot.generated_text.end());
            // 更新位置
            pos = std::min(slot.sent_count, slot.generated_text.size());
        }
        else
            {
                // 设置 is_stop_full 为 false
                is_stop_full = false;
                // 查找停止字符串的位置
                stop_pos = find_stopping_strings(str_test, token_str.size(), STOP_PARTIAL, slot);
            }

            // 检查是否有任何要预测的标记
            if (stop_pos == std::string::npos || (!slot.has_next_token && !is_stop_full && stop_pos > 0))
            {
                // 不发送停止词在响应中
                result.text_to_send = slot.generated_text.substr(pos, std::string::npos);
                slot.sent_count += result.text_to_send.size();
                // 将标记添加到槽队列和缓存中
            }
            // 将标记字符串添加到槽中
            slot.add_token_string(result);
            // 如果参数中有流，则发送部分响应
            if (slot.params.stream)
            {
                send_partial_response(slot, result);
            }
        }
        // 如果有多字节待处理且没有下一个标记，则设置has_next_token为true
        if (slot.multibyte_pending > 0 && !slot.has_next_token)
        {
            slot.has_next_token = true;
        }

        // 检查限制条件
        if (slot.n_decoded > 2 && slot.has_next_token && !slot.has_budget(params))
        {
            // 如果解码数量大于2且有下一个标记且没有足够的预算，则设置stopped_limit为true，has_next_token为false
            slot.stopped_limit = true;
            slot.has_next_token = false;
        }

        // 如果缓存中不为空且结果的标记为EOS，则设置stopped_eos为true，has_next_token为false，并记录日志
        if (!slot.cache_tokens.empty() && result.tok == llama_token_eos(model))
        {
            slot.stopped_eos = true;
            slot.has_next_token = false;
            LOG_VERBOSE("eos token found", {});
        }

        // 记录日志，表示找到下一个标记
        LOG_VERBOSE("next token", {
    // 将结果数据存储到一个包含键值对的字典中
    slot.result_data.insert({
        {"token", result.tok},
        {"token_text", tokens_to_output_formatted_string(ctx, result.tok)},
        {"has_next_token", slot.has_next_token},
        {"n_remain", slot.n_remaining},
        {"num_tokens_predicted", slot.n_decoded},
        {"stopped_eos", slot.stopped_eos},
        {"stopped_word", slot.stopped_word},
        {"stopped_limit", slot.stopped_limit},
        {"stopping_word", slot.stopping_word},
    });

    // 返回是否还有下一个 token，用于决定是否继续处理
    return slot.has_next_token; // continue
}

// 处理图像数据
bool process_images(llama_client_slot &slot) const
{
    // 遍历槽位中的图像数据
    for (slot_image &img : slot.images)
    {
        // 如果图像不需要编码
        if (!img.request_encode_image)
        {
            // 进行处理...
                continue;
            }
            // 定义一个名为img_res的结构体变量，用于存储预处理后的图像数据
            clip_image_f32 img_res;
            // 调用clip_image_preprocess函数对图像进行预处理，将结果存储在img_res中
            if (!clip_image_preprocess(clp_ctx, &img.img_data, &img_res, /*pad2square =*/ true))
            {
                // 如果预处理出错，记录错误信息并释放资源，返回false
                LOG_TEE("Error processing the given image");
                clip_free(clp_ctx);
                return false;
            }
            // 获取图像的分块数量
            img.image_tokens = clip_n_patches(clp_ctx);
            // 为图像嵌入分配内存空间
            img.image_embedding = (float *)malloc(clip_embd_nbytes(clp_ctx));
            // 如果内存分配失败，记录错误信息并释放资源，返回false
            if (!img.image_embedding)
            {
                LOG_TEE("Unable to allocate memory for image embeddings\n");
                clip_free(clp_ctx);
                return false;
            }
            // 记录日志，表示正在对图像进行编码
            LOG_TEE("slot %i - encoding image [id: %i]\n", slot.id, img.id);
            // 调用clip_image_encode函数对图像进行编码
            if (!clip_image_encode(clp_ctx, params.n_threads, &img_res, img.image_embedding))
            {
// 记录错误信息并返回 false
LOG_TEE("Unable to encode image\n");
return false;
// 将请求编码图像的标志设为 false
img.request_encode_image = false;
// 返回插槽中是否有图像
return slot.images.size() > 0;
// 发送错误信息
void send_error(int id, std::string error)
{
    // 使用互斥锁保护共享资源
    std::lock_guard<std::mutex> lock(mutex_results);
    // 创建任务结果对象
    task_result res;
    res.id = id;
    res.error = true;
    res.result_json = { { "content", error } };
    // 将结果对象加入结果队列
    queue_results.push_back(res);
}

// 获取模型属性的 JSON 数据
json get_model_props()
# 返回格式化后的生成结果
{
    return get_formated_generation(slots[0]);
}

# 获取格式化后的生成结果
json get_formated_generation(llama_client_slot &slot)
{
    # 查找是否存在 EOS 偏置，并判断是否需要忽略 EOS
    const auto eos_bias = slot.sparams.logit_bias.find(llama_token_eos(model));
    const bool ignore_eos = eos_bias != slot.sparams.logit_bias.end() &&
                            eos_bias->second < 0.0f && std::isinf(eos_bias->second);
    # 返回包含格式化生成结果的 JSON 对象
    return json {
        {"n_ctx",             slot.n_ctx},
        {"model",             params.model_alias},
        {"seed",              slot.params.seed},
        {"temp",              slot.sparams.temp},
        {"top_k",             slot.sparams.top_k},
        {"top_p",             slot.sparams.top_p},
        {"min_p",             slot.sparams.min_p},
        {"tfs_z",             slot.sparams.tfs_z},
        {"typical_p",         slot.sparams.typical_p},
        {"repeat_last_n",     slot.sparams.penalty_last_n},
    // 创建一个包含各种参数和值的字典
    {
        {"repeat_penalty",    slot.sparams.penalty_repeat}, // 重复惩罚参数
        {"presence_penalty",  slot.sparams.penalty_present}, // 存在惩罚参数
        {"frequency_penalty", slot.sparams.penalty_freq}, // 频率惩罚参数
        {"mirostat",          slot.sparams.mirostat}, // mirostat 参数
        {"mirostat_tau",      slot.sparams.mirostat_tau}, // mirostat_tau 参数
        {"mirostat_eta",      slot.sparams.mirostat_eta}, // mirostat_eta 参数
        {"penalize_nl",       slot.sparams.penalize_nl}, // penalize_nl 参数
        {"stop",              slot.params.antiprompt}, // 停止参数
        {"n_predict",         slot.params.n_predict}, // 预测数量参数
        {"n_keep",            params.n_keep}, // 保留数量参数
        {"ignore_eos",        ignore_eos}, // 忽略 EOS 参数
        {"stream",            slot.params.stream}, // 流参数
        {"logit_bias",        slot.sparams.logit_bias}, // logit_bias 参数
        {"n_probs",           slot.sparams.n_probs}, // n_probs 参数
        {"grammar",           slot.sparams.grammar}, // 语法参数
    };
}

void send_partial_response(llama_client_slot &slot, completion_token_output tkn)
{
    // 发送部分响应
        // 使用互斥锁保护共享资源
        std::lock_guard<std::mutex> lock(mutex_results);
        // 创建任务结果对象
        task_result res;
        // 设置任务结果对象的ID
        res.id = slot.task_id;
        // 初始化错误和停止标志
        res.error = false;
        res.stop = false;

        // 设置结果的 JSON 对象
        res.result_json = json
        {
            // 设置内容字段
            {"content",    tkn.text_to_send},
            // 设置停止字段
            {"stop",       false},
            // 设置槽位ID字段
            {"slot_id",    slot.id},
            // 设置多模态字段
            {"multimodal", multimodal}
        };

        // 如果槽位的参数中包含概率值
        if (slot.sparams.n_probs > 0)
        {
            // 初始化概率输出向量
            std::vector<completion_token_output> probs_output = {};
            // 获取要发送的令牌
            const std::vector<llama_token> to_send_toks = llama_tokenize(ctx, tkn.text_to_send, false);
            // 计算概率的起始位置和结束位置
            size_t probs_pos = std::min(slot.sent_token_probs_index, slot.generated_token_probs.size());
            size_t probs_stop_pos = std::min(slot.sent_token_probs_index + to_send_toks.size(), slot.generated_token_probs.size());
        // 如果概率位置小于概率停止位置
        if (probs_pos < probs_stop_pos)
        {
            // 从生成的标记概率中创建一个子向量
            probs_output = std::vector<completion_token_output>(slot.generated_token_probs.begin() + probs_pos, slot.generated_token_probs.begin() + probs_stop_pos);
        }
        // 设置已发送标记概率的索引为停止位置
        slot.sent_token_probs_index = probs_stop_pos;
        // 将概率输出转换为 JSON 格式，并存入结果 JSON 中
        res.result_json["completion_probabilities"] = probs_vector_to_json(ctx, probs_output);
    }

    // 将结果推入结果队列
    queue_results.push_back(res);
}

// 发送最终响应
void send_final_response(llama_client_slot &slot)
{
    // 使用互斥锁保护临界区
    std::lock_guard<std::mutex> lock(mutex_results);
    // 创建任务结果对象
    task_result res;
    res.id = slot.task_id;
    res.error = false;
    res.stop = true;

    // 初始化结果 JSON
    res.result_json = json
# 创建一个包含各种参数和数据的字典
{
    "content",             # 如果没有流参数，则使用生成的文本，否则为空字符串
    "slot_id",             # 插槽的ID
    "stop",                # 停止标志
    "model",               # 参数的模型别名
    "tokens_predicted",    # 预测的标记数
    "tokens_evaluated",    # 评估的提示标记数
    "generation_settings", # 获取格式化的生成设置
    "prompt",              # 插槽的提示
    "truncated",           # 截断标志
    "stopped_eos",         # 停止的EOS标志
    "stopped_word",        # 停止的单词
    "stopped_limit",       # 停止的限制
    "stopping_word",       # 停止的单词
    "tokens_cached",       # 缓存的标记数
    "timings",             # 获取格式化的时间
};

# 如果参数的概率大于0
if (slot.sparams.n_probs > 0)
// 创建一个空的完成令牌输出向量
std::vector<completion_token_output> probs = {};
// 如果不是流式处理并且存在停止词
if (!slot.params.stream && slot.stopped_word)
{
    // 使用 llama_tokenize 函数对停止词进行分词
    const std::vector<llama_token> stop_word_toks = llama_tokenize(ctx, slot.stopping_word, false);
    // 从生成的令牌概率中创建一个新的完成令牌输出向量，去掉停止词对应的部分
    probs = std::vector<completion_token_output>(slot.generated_token_probs.begin(), slot.generated_token_probs.end() - stop_word_toks.size());
}
else
{
    // 从生成的令牌概率中创建一个新的完成令牌输出向量，取前 slot.sent_token_probs_index 个元素
    probs = std::vector<completion_token_output>(
                        slot.generated_token_probs.begin(),
                        slot.generated_token_probs.begin() + slot.sent_token_probs_index);
}
// 将完成概率向量转换为 JSON 格式，并存入结果 JSON 对象中
res.result_json["completion_probabilities"] = probs_vector_to_json(ctx, probs);
// 将结果对象添加到队列中
queue_results.push_back(res);
// 发送嵌入
void send_embedding(llama_client_slot &slot)
// 使用互斥锁保护共享资源，防止多个线程同时访问
std::lock_guard<std::mutex> lock(mutex_results);
// 创建任务结果对象
task_result res;
// 设置任务结果对象的ID为任务槽的任务ID
res.id = slot.task_id;
// 初始化错误标志为false
res.error = false;
// 初始化停止标志为true

// 获取模型的嵌入维度
const int n_embd = llama_n_embd(model);
// 如果嵌入被禁用
if (!params.embedding)
{
    // 记录警告日志，指出嵌入被禁用
    LOG_WARNING("embedding disabled", {
                                          {"params.embedding", params.embedding},
                                      });
    // 设置结果JSON为包含n_embd个0.0f的浮点数向量
    res.result_json = json
    {
        {"embedding", std::vector<float>(n_embd, 0.0f)},
    };
}
// 如果嵌入未被禁用
else
{
    // 获取上下文中的嵌入数据
    const float *data = llama_get_embeddings(ctx);
// 创建一个包含 n_embd 个元素的浮点数向量，并初始化为 data 数组的值
std::vector<float> embedding(data, data + n_embd);
// 将 embedding 向量作为值，"embedding" 作为键，创建一个 JSON 对象
res.result_json = json
{
    {"embedding", embedding },
};
// 将 res 对象添加到 queue_results 向量中
queue_results.push_back(res);
// 接收一个 JSON 数据、infill 模式和 embedding 模式作为参数，创建一个任务对象
int request_completion(json data, bool infill, bool embedding)
{
    // 使用互斥锁保护共享资源
    std::lock_guard<std::mutex> lock(mutex_tasks);
    // 创建一个任务对象，并设置其属性
    task_server task;
    task.id = id_gen++;
    task.data = data;
    task.infill_mode = infill;
    task.embedding_mode = embedding;
    task.type = COMPLETION_TASK;
    // 将任务对象添加到 queue_tasks 向量中
    queue_tasks.push_back(task);
    // 返回任务的 ID
    return task.id;
}
    }

    // 获取下一个任务的结果
    task_result next_result(int task_id)
    {
        // 循环等待结果
        while (true)
        {
            // 线程休眠5微秒
            std::this_thread::sleep_for(std::chrono::microseconds(5));
            // 加锁，保护结果队列
            std::lock_guard<std::mutex> lock(mutex_results);

            // 如果结果队列为空，则继续等待
            if (queue_results.empty())
            {
                continue;
            }

            // 遍历结果队列
            for (int i = 0; i < (int) queue_results.size(); i++)
            {
                // 如果找到对应任务的结果
                if (queue_results[i].id == task_id)
                {
                    // 获取结果并从队列中移除
                    task_result res = queue_results[i];
                    queue_results.erase(queue_results.begin() + i);
        // 返回结果
        return res;
    }
}
}

// 永远不会执行到的代码
//return task_result{-1, false, false, {}};
}

// 用于多图像处理
bool ingest_images(llama_client_slot &slot, int n_batch)
{
    int image_idx = 0;

    // 当图像索引小于槽中图像数量时执行循环
    while (image_idx < (int) slot.images.size())
    {
        // 获取当前图像
        slot_image &img = slot.images[image_idx];

        // 处理前缀提示
        for (int32_t i = 0; i < (int32_t) batch.n_tokens; i += n_batch)
{
    // 计算每个批次的标记数量，取较小值作为实际处理的标记数量
    const int32_t n_tokens = std::min(n_batch, (int32_t) (batch.n_tokens - i));
    // 创建一个批次的视图，包括标记数量、标记指针、位置指针、序列 ID 数组指针、logits 指针
    llama_batch batch_view = {
        n_tokens,
        batch.token    + i,
        nullptr,
        batch.pos      + i,
        batch.n_seq_id + i,
        batch.seq_id   + i,
        batch.logits   + i,
        0, 0, 0, // 未使用的字段
    };
    // 使用 llama_decode 函数处理批次视图
    if (llama_decode(ctx, batch_view))
    {
        // 如果处理失败，记录日志并返回 false
        LOG_TEE("%s : failed to eval\n", __func__);
        return false;
    }
}

// 使用 llm 处理图像
// 循环遍历图像的令牌，每次处理一个批次的数据
for (int i = 0; i < img.image_tokens; i += n_batch)
{
    // 计算当前批次的大小
    int n_eval = img.image_tokens - i;
    if (n_eval > n_batch)
    {
        n_eval = n_batch;
    }

    // 获取嵌入维度的大小
    const int n_embd = llama_n_embd(model);
    // 创建图像批次对象
    llama_batch batch_img = { n_eval, nullptr, (img.image_embedding + i * n_embd), nullptr, nullptr, nullptr, nullptr, slot.n_past, 1, 0, };
    // 解码图像批次对象
    if (llama_decode(ctx, batch_img))
    {
        // 如果解码失败，记录日志并返回失败
        LOG_TEE("%s : failed to eval image\n", __func__);
        return false;
    }
    // 更新已处理的图像数量
    slot.n_past += n_eval;
}
// 增加图像索引
image_idx++;

// 清空批次对象
llama_batch_clear(batch);
// 添加下一个图像的前缀
const auto json_prompt = (image_idx >= (int) slot.images.size()) ?
    slot.params.input_suffix : // 如果没有更多的图像，则处理后缀提示
    (json)(slot.images[image_idx].prefix_prompt);

// 将前缀转换为令牌序列
std::vector<llama_token> append_tokens = tokenize(json_prompt, false); // 存在下一个图像
for (int i = 0; i < (int) append_tokens.size(); ++i)
{
    // 将令牌添加到批处理中
    llama_batch_add(batch, append_tokens[i], slot.n_past, { slot.id }, true);
    slot.n_past += 1;
}

// 返回 true 表示成功
return true;
}

// 取消请求
void request_cancel(int task_id)
{
    // 获取任务锁
    std::lock_guard<std::mutex> lock(mutex_tasks);
// 声明一个名为task的task_server对象
task_server task;
// 给task对象的id属性赋值为id_gen的值，然后id_gen自增1
task.id = id_gen++;
// 给task对象的type属性赋值为CANCEL_TASK
task.type = CANCEL_TASK;
// 给task对象的target_id属性赋值为task_id
task.target_id = task_id;
// 将task对象加入到queue_tasks队列的末尾
queue_tasks.push_back(task);
// 处理任务的函数
void process_tasks()
{
    // 使用互斥锁锁住mutex_tasks
    std::lock_guard<std::mutex> lock(mutex_tasks);
    // 当队列不为空时循环处理任务
    while (!queue_tasks.empty())
    {
        // 从队列中取出第一个任务
        task_server task = queue_tasks.front();
        // 删除队列中的第一个任务
        queue_tasks.erase(queue_tasks.begin());
        // 根据任务的类型进行不同的处理
        switch (task.type)
        {
            // 如果任务类型为COMPLETION_TASK
            case COMPLETION_TASK: {
                // 根据任务的数据中的slot_id获取对应的llama_client_slot对象
                llama_client_slot *slot = get_slot(json_value(task.data, "slot_id", -1));
                // 如果获取的slot对象为空
                if (slot == nullptr)
                {
// 记录日志，提示槽位不可用
LOG_TEE("slot unavailable\n");
// 发送错误结果
send_error(task.id, "slot unavailable");
// 返回
return;

// 如果任务数据中包含系统提示
if (task.data.contains("system_prompt"))
{
    // 处理系统提示数据
    process_system_prompt_data(task.data["system_prompt"]);
}

// 重置槽位
slot->reset();

// 设置槽位的填充模式和嵌入模式
slot->infill = task.infill_mode;
slot->embedding = task.embedding_mode;
slot->task_id = task.id;

// 如果启动槽位失败
if (!launch_slot_with_data(slot, task.data))
{
    // 发送错误结果
                        send_error(task.id, "internal_error"); // 发送内部错误消息给任务
                        break; // 跳出switch语句
                    }
                } break; // 跳出switch语句
                case CANCEL_TASK: { // 取消任务
                    for (auto & slot : slots) // 遍历slots
                    {
                        if (slot.task_id == task.target_id) // 如果slot的任务id与目标id相同
                        {
                            slot.release(); // 释放slot
                            break; // 跳出循环
                        }
                    }
                } break; // 跳出switch语句
            }
        }
    }

    bool update_slots() { // 更新slots
        // attend tasks // 处理任务
        // 处理任务
        process_tasks();

        // 更新系统提示，等待直到所有插槽都处于空闲状态
        if (system_need_update && all_slots_are_idle)
        {
            // 记录日志，表示正在更新系统提示
            LOG_TEE("updating system prompt\n");
            // 更新系统提示
            update_system_prompt();
        }

        // 清空LLAMA批处理
        llama_batch_clear(batch);

        // 如果所有插槽都处于空闲状态
        if (all_slots_are_idle)
        {
            // 如果系统提示为空并且需要清除KV缓存
            if (system_prompt.empty() && clean_kv_cache)
            {
                // 记录日志，表示所有插槽都处于空闲状态且系统提示为空，清除KV缓存
                LOG_TEE("all slots are idle and system prompt is empty, clear the KV cache\n");
                // 清除KV缓存
                kv_cache_clear();
            }
            // 避免CPU始终处于100%使用率
            std::this_thread::sleep_for(std::chrono::milliseconds(5));
        }

        // 遍历所有的槽位
        for (llama_client_slot &slot : slots)
        {
            // 如果槽位正在处理并且缓存令牌数量大于等于槽位的上下文大小
            if (slot.is_processing() && slot.cache_tokens.size() >= (size_t) slot.n_ctx)
            {
                // 移动上下文
                const int n_left    = slot.n_past - slot.params.n_keep - 1;
                const int n_discard = n_left / 2;

                // 记录日志
                LOG_TEE("slot %d: context shift - n_keep = %d, n_left = %d, n_discard = %d\n", slot.id, slot.params.n_keep, n_left, n_discard);
                // 从缓存中移除一定数量的令牌
                llama_kv_cache_seq_rm   (ctx, slot.id, slot.params.n_keep + 1            , slot.params.n_keep + n_discard + 1);
                // 移动缓存中的令牌
                llama_kv_cache_seq_shift(ctx, slot.id, slot.params.n_keep + 1 + n_discard, slot.n_past, -n_discard);

                // 调整缓存中的令牌
                for (size_t i = slot.params.n_keep + 1 + n_discard; i < slot.cache_tokens.size(); i++)
                {
                    slot.cache_tokens[i - n_discard] = slot.cache_tokens[i];
                }

                // 调整缓存中的令牌大小
                slot.cache_tokens.resize(slot.cache_tokens.size() - n_discard);
                // 减去要丢弃的过去数据数量
                slot.n_past -= n_discard;

                // 标记为被截断
                slot.truncated = true;

                // 记录上下文转移的详细信息
                LOG_VERBOSE("context shift", {
                                                {"n_ctx",  n_ctx},
                                                {"n_keep", params.n_keep},
                                                {"n_left", n_left},
                                            });
            }
        }

        // 解码当前正在进行的序列
        for (auto & slot : slots)
        {
            // 释放该槽位
            if (slot.command == RELEASE)
            {
                // 将槽位状态设置为闲置
                slot.state = IDLE;
                # 将命令设置为NONE
                slot.command = NONE;
                # 更新最后使用时间
                slot.t_last_used = ggml_time_us();

                # 打印释放的插槽信息
                LOG_TEE("slot %d released (%d tokens in cache)\n", slot.id, (int) slot.cache_tokens.size());

                # 继续下一次循环
                continue;
            }

            # 如果插槽状态为IDLE，则继续下一次循环
            if (slot.state == IDLE)
            {
                continue;
            }

            # 设置批次中的tokens数量
            slot.i_batch = batch.n_tokens;

            # 向llama_batch_add函数中添加批次信息
            llama_batch_add(batch, slot.sampled, system_tokens.size() + slot.n_past, { slot.id }, true);

            # 更新插槽的解码数量和过去数量
            slot.n_decoded += 1;
            slot.n_past += 1;
        }
// 按照 params.n_batch 的大小分批处理数据
int32_t n_batch = params.n_batch;

// 将工作负载分配给插槽
if (params.cont_batching || batch.n_tokens == 0)
{
    for (auto & slot : slots)
    {
        // 检查插槽是否有提示
        const bool has_prompt = slot.prompt.is_array() || (slot.prompt.is_string() && !slot.prompt.get<std::string>().empty()) || !slot.images.empty();

        // 如果插槽处于空闲状态且命令为加载提示，并且没有提示被传递，则释放插槽并发送空响应
        if (slot.state == IDLE && slot.command == LOAD_PROMPT && !has_prompt)
        {
            slot.release();
            slot.print_timings();
            send_final_response(slot);
            continue;
        }
// 如果槽位状态为IDLE并且命令为LOAD_PROMPT，则需要处理提示
if (slot.state == IDLE && slot.command == LOAD_PROMPT)
{
    // 将槽位状态设置为PROCESSING，命令设置为NONE
    slot.state = PROCESSING;
    slot.command = NONE;
    // 创建一个存储llama_token的向量prompt_tokens
    std::vector<llama_token> prompt_tokens;
    // 记录开始处理提示的时间
    slot.t_start_process_prompt = ggml_time_us();
    slot.t_start_genereration = 0;

    // 如果槽位需要填充
    if (slot.infill)
    {
        // 初始化一个布尔变量suff_rm_leading_spc为true
        bool suff_rm_leading_spc = true;
        // 如果输入后缀的第一个字符是空格并且后缀长度大于1
        if (params.input_suffix.find_first_of(' ') == 0 && params.input_suffix.size() > 1)
        {
            // 删除输入后缀的第一个字符，将suff_rm_leading_spc设置为false
            params.input_suffix.erase(0, 1);
            suff_rm_leading_spc = false;
        }
        // 将输入前缀和后缀分别进行标记化处理
        auto prefix_tokens = tokenize(slot.params.input_prefix, false);
        auto suffix_tokens = tokenize(slot.params.input_suffix, false);
// 定义一个整型常量 space_token，并赋值为 29871，TODO: 这个数值不应该是硬编码的
if (suff_rm_leading_spc && !suffix_tokens.empty() && suffix_tokens[0] == space_token) {
    // 如果需要移除前导空格，并且后缀标记不为空，并且第一个后缀标记是空格标记，则移除第一个后缀标记
    suffix_tokens.erase(suffix_tokens.begin());
}

// 在前缀标记列表的开头插入 llama_token_prefix(model) 的返回值
prefix_tokens.insert(prefix_tokens.begin(), llama_token_prefix(model));
// 在前缀标记列表的开头插入 llama_token_bos(model) 的返回值，始终添加 BOS
prefix_tokens.insert(prefix_tokens.begin(), llama_token_bos(model));
// 在前缀标记列表的末尾插入 llama_token_suffix(model) 的返回值
prefix_tokens.insert(prefix_tokens.end(), llama_token_suffix(model));
// 在前缀标记列表的末尾插入后缀标记列表的内容
prefix_tokens.insert(prefix_tokens.end(), suffix_tokens.begin(), suffix_tokens.end());
// 在前缀标记列表的末尾添加 llama_token_middle(model) 的返回值
prefix_tokens.push_back(llama_token_middle(model));
// 将前缀标记列表赋值给提示标记列表
prompt_tokens = prefix_tokens;
}
else
{
    // 如果没有系统提示，则根据槽位提示进行标记化，并在没有系统提示时添加 BOS
    prompt_tokens = tokenize(slot.prompt, system_prompt.empty());
}

// 记录提示标记的数量
slot.num_prompt_tokens = prompt_tokens.size();

if (slot.params.n_keep < 0)
                    {
                        // 设置保留的输入提示标记数量为初始提示标记数量
                        slot.params.n_keep = slot.num_prompt_tokens;
                    }
                    // 限制保留的输入提示标记数量不超过上下文标记数量减去4
                    slot.params.n_keep = std::min(slot.n_ctx - 4, slot.params.n_keep);

                    // 如果输入提示太大，进行截断处理
                    if (slot.num_prompt_tokens >= slot.n_ctx)
                    {
                        // 计算剩余标记数量
                        const int n_left = slot.n_ctx - slot.params.n_keep;
                        // 计算每个块的大小
                        const int n_block_size = n_left / 2;
                        // 计算需要删除的块数量
                        const int erased_blocks = (slot.num_prompt_tokens - slot.params.n_keep - n_block_size) / n_block_size;

                        // 创建新的标记列表，保留部分初始标记并删除部分块
                        std::vector<llama_token> new_tokens(prompt_tokens.begin(), prompt_tokens.begin() + slot.params.n_keep);
                        new_tokens.insert(new_tokens.end(), prompt_tokens.begin() + slot.params.n_keep + erased_blocks * n_block_size, prompt_tokens.end());

                        // 记录日志，显示截断后的相关信息
                        LOG_VERBOSE("input truncated", {
                            {"n_ctx",  slot.n_ctx},
                            {"n_keep", slot.params.n_keep},
                            {"n_left", n_left},
                            {"new_tokens", tokens_to_str(ctx, new_tokens.cbegin(), new_tokens.cend())},
                    });
                    // 设置截断标志为true
                    slot.truncated = true;
                    // 更新prompt_tokens
                    prompt_tokens = new_tokens;

                    // 更新slot中的num_prompt_tokens
                    slot.num_prompt_tokens = prompt_tokens.size();
                    // 断言slot中的num_prompt_tokens小于slot中的n_ctx
                    GGML_ASSERT(slot.num_prompt_tokens < slot.n_ctx);
                }

                // 如果不缓存prompt
                if (!slot.params.cache_prompt)
                {
                    // 重置上下文采样
                    llama_sampling_reset(slot.ctx_sampling);

                    // 重置n_past为0
                    slot.n_past = 0;
                    // 更新num_prompt_tokens_processed为num_prompt_tokens
                    slot.num_prompt_tokens_processed = slot.num_prompt_tokens;
                }
                else
                {
                    // 将prompt推入采样上下文（不应用语法）
                    for (auto &token : prompt_tokens)
                    {
# 调用 llama_sampling_accept 函数，接受采样结果
llama_sampling_accept(slot.ctx_sampling, ctx, token, false);
}

# 计算 slot.n_past，即缓存中与输入 tokens 的公共部分的长度
slot.n_past = common_part(slot.cache_tokens, prompt_tokens);
# 计算需要处理的 prompt tokens 的数量
slot.num_prompt_tokens_processed = slot.num_prompt_tokens - slot.n_past;

# 打印日志，显示缓存中的 tokens 数量和需要处理的 tokens 数量
LOG_TEE("slot %d : in cache: %i tokens | to process: %i tokens\n", slot.id, slot.n_past, slot.num_prompt_tokens_processed);
}

# 打印日志，显示需要从缓存中移除的 tokens 范围
LOG_TEE("slot %d : kv cache rm - [%d, end)\n", slot.id, (int) system_tokens.size() + slot.n_past);

# 调用 llama_kv_cache_seq_rm 函数，从缓存中移除指定范围的 tokens
llama_kv_cache_seq_rm(ctx, slot.id, system_tokens.size() + slot.n_past, -1);

# 更新缓存中的 tokens 为输入的 prompt tokens
slot.cache_tokens = prompt_tokens;

# 如果缓存中的 tokens 与输入的 prompt tokens 长度相等
if (slot.n_past == slot.num_prompt_tokens)
{
    # 打印日志，提示至少需要评估一个 token 以生成 logits
    LOG_TEE("slot %d : we have to evaluate at least 1 token to generate logits\n", slot.id);
    # 减少 slot.n_past 的值，以便至少评估一个 token
    slot.n_past--;
// 记录详细信息，包括过去的数量、缓存的内容和待评估的内容
LOG_VERBOSE("prompt ingested", {
                                {"n_past", slot.n_past},
                                {"cached", tokens_to_str(ctx, slot.cache_tokens.cbegin(), slot.cache_tokens.cbegin() + slot.n_past)},
                                {"to_eval", tokens_to_str(ctx, slot.cache_tokens.cbegin() + slot.n_past, slot.cache_tokens.cend())},
                            });

// 检查是否有图片，并处理图片
const bool has_images = process_images(slot);

// 处理第一张图片的前缀
std::vector<llama_token> prefix_tokens = has_images ? tokenize(slot.images[0].prefix_prompt, true) : prompt_tokens;
for (; slot.n_past < (int) prefix_tokens.size(); ++slot.n_past)
{
   // 将前缀的 token 添加到批处理中
   llama_batch_add(batch, prefix_tokens[slot.n_past], system_tokens.size() + slot.n_past, { slot.id }, false);
}

// 如果有图片并且图片处理失败，则记录日志
if (has_images && !ingest_images(slot, n_batch))
{
    LOG_TEE("failed processing images\n");
}
// 如果批处理中没有标记，则返回假
return false;

// 仅提取最后一个标记的logits
if (batch.n_tokens > 0)
{
    batch.logits[batch.n_tokens - 1] = true;
}

// 重置已解码的槽数量和批处理中的标记索引
slot.n_decoded = 0;
slot.i_batch   = batch.n_tokens - 1;

// 如果批处理中没有标记，则所有槽都处于空闲状态，返回真
if (batch.n_tokens == 0)
{
    all_slots_are_idle = true;
    return true;
}
// 循环遍历批处理中的标记，每次增加 n_batch 个
for (int32_t i = 0; i < (int32_t) batch.n_tokens; i += n_batch)
{
    // 计算当前批处理中实际要处理的标记数量
    const int32_t n_tokens = std::min(n_batch, (int32_t) (batch.n_tokens - i));
    // 创建批处理视图
    llama_batch batch_view =
    {
        n_tokens,
        batch.token    + i,
        nullptr,
        batch.pos      + i,
        batch.n_seq_id + i,
        batch.seq_id   + i,
        batch.logits   + i,
        0, 0, 0, // 未使用的字段
    };

    // 调用 llama_decode 函数进行解码
    const int ret = llama_decode(ctx, batch_view);
    // 检查解码结果
    if (ret != 0)
    {
        // 如果批处理大小为1或者返回值小于0，则执行以下操作
{
    // 如果程序执行到这里，意味着 KV 缓存已满 - 尝试通过上下文大小增加它
    LOG_TEE("%s : failed to decode the batch, n_batch = %d, ret = %d\n", __func__, n_batch, ret);
    return false;
}

// 在 KV 缓存中找不到空闲空间，尝试使用较小的 n_batch = %d 重试
LOG_TEE("%s : failed to find free space in the KV cache, retrying with smaller n_batch = %d\n", __func__, n_batch / 2);

// 用一半的批处理大小重试，以尝试在 KV 缓存中找到空闲插槽
n_batch /= 2;
i -= n_batch;
continue;
}

for (auto & slot : slots)
{
    // 如果插槽的批次号小于 i 或者大于等于 (i + n_tokens)，则继续循环
    if (slot.i_batch < (int) i || slot.i_batch >= (int) (i + n_tokens))
    {
        continue;
    }
}
// 如果存在嵌入式的提示，则进行评估并发送嵌入式的提示
if (slot.embedding)
{
    send_embedding(slot); // 发送嵌入式的提示
    slot.release(); // 释放插槽
    slot.i_batch = -1; // 重置批次计数
    return true; // 返回true
}

completion_token_output result; // 定义完成令牌输出变量
const llama_token id = llama_sampling_sample(slot.ctx_sampling, ctx, NULL, slot.i_batch - i); // 从上下文中采样一个令牌

llama_sampling_accept(slot.ctx_sampling, ctx, id, true); // 接受采样的令牌

if (slot.n_decoded == 1) // 如果已解码的数量为1
{
    slot.t_start_genereration = ggml_time_us(); // 记录生成开始时间
    slot.t_prompt_processing = (slot.t_start_genereration - slot.t_start_process_prompt) / 1e3; // 计算提示处理时间
}
// 创建一个名为cur_p的llama_token_data_array结构体，包含当前数据指针、大小和一个布尔值
llama_token_data_array cur_p = { slot.ctx_sampling->cur.data(), slot.ctx_sampling->cur.size(), false };
// 将id赋值给result的tok成员
result.tok = id;

// 获取概率数量n_probs
const int32_t n_probs = slot.sparams.n_probs;
// 如果温度小于等于0且概率数量大于0，则对候选项进行排序
if (slot.sparams.temp <= 0 && n_probs > 0)
{
    llama_sample_softmax(ctx, &cur_p);
}

// 遍历当前数据的大小和概率数量的较小值，将id和概率值添加到result的probs成员中
for (size_t i = 0; i < std::min(cur_p.size, (size_t)n_probs); ++i)
{
    result.probs.push_back({cur_p.data[i].id, cur_p.data[i].p});
}

// 如果处理token失败，则释放slot并打印时间
if (!process_token(result, slot))
{
    slot.release();
    slot.print_timings();
}
// 发送最终响应给槽位
send_final_response(slot);
// 重置批次号为-1
slot.i_batch = -1;
// 返回true
return true;
};

// 打印服务器使用说明
static void server_print_usage(const char *argv0, const gpt_params &params,
                               const server_params &sparams)
{
    // 打印使用说明
    printf("usage: %s [options]\n", argv0);
    printf("\n");
    printf("options:\n");
    // 打印帮助信息
    printf("  -h, --help                show this help message and exit\n");
    // 打印详细输出选项
    printf("  -v, --verbose             verbose output (default: %s)\n", server_verbose ? "enabled" : "disabled");
    // 打印线程数选项
    printf("  -t N, --threads N         number of threads to use during computation (default: %d)\n", params.n_threads);
    // 打印批处理和提示处理时使用的线程数选项
    printf("  -tb N, --threads-batch N  number of threads to use during batch and prompt processing (default: same as --threads)\n");
}
# 打印提示信息，显示参数 N 的含义和默认值
printf("  -c N, --ctx-size N        size of the prompt context (default: %d)\n", params.n_ctx);
# 打印提示信息，显示 RoPE 频率缩放方法的选项
printf("  --rope-scaling {none,linear,yarn}\n");
# 打印提示信息，显示 RoPE 基础频率的含义和默认值
printf("  --rope-freq-base N        RoPE base frequency (default: loaded from model)\n");
# 打印提示信息，显示 RoPE 频率缩放因子的含义和默认值
printf("  --rope-freq-scale N       RoPE frequency scaling factor, expands context by a factor of 1/N\n");
# 打印提示信息，显示 YaRN 的外推混合因子的含义和默认值
printf("  --yarn-ext-factor N       YaRN: extrapolation mix factor (default: 1.0, 0.0 = full interpolation)\n");
# 打印提示信息，显示 YaRN 的注意力因子的含义和默认值
printf("  --yarn-attn-factor N      YaRN: scale sqrt(t) or attention magnitude (default: 1.0)\n");
# 打印提示信息，显示 YaRN 的高校正维度或 alpha 的含义和默认值
printf("  --yarn-beta-slow N        YaRN: high correction dim or alpha (default: %.1f)\n", params.yarn_beta_slow);
# 打印提示信息，显示 YaRN 的低校正维度或 beta 的含义和默认值
printf("  --yarn-beta-fast N        YaRN: low correction dim or beta (default: %.1f)\n", params.yarn_beta_fast);
# 打印提示信息，显示参数 N 的含义和默认值
printf("  -b N, --batch-size N      batch size for prompt processing (default: %d)\n", params.n_batch);
# 打印提示信息，显示是否使用 f32 代替 f16 作为内存键值对的数据类型
printf("  --memory-f32              use f32 instead of f16 for memory key+value (default: disabled)\n");
# 打印提示信息，显示不推荐使用 f32 的原因
printf("                            not recommended: doubles context memory required and no measurable increase in quality\n");
# 如果系统支持内存锁定，则打印提示信息，显示强制系统将模型保留在内存中的选项
if (llama_mlock_supported())
{
    printf("  --mlock               force system to keep model in RAM rather than swapping or compressing\n");
}
# 如果系统支持内存映射，则打印提示信息，显示不使用内存映射模型的选项
if (llama_mmap_supported())
{
    printf("  --no-mmap             do not memory-map model (slower load but may reduce pageouts if not using mlock)\n");
}
// 打印提示信息，指示在某些NUMA系统上尝试优化
printf("  --numa                attempt optimizations that help on some NUMA systems\n");
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
// 如果支持GPU卸载，则打印以下信息
printf("  -ngl N, --n-gpu-layers N\n");
printf("                        number of layers to store in VRAM\n");
printf("  -ts SPLIT --tensor-split SPLIT\n");
printf("                        how to split tensors across multiple GPUs, comma-separated list of proportions, e.g. 3,1\n");
printf("  -mg i, --main-gpu i   the GPU to use for scratch and small tensors\n");
printf("  -nommq, --no-mul-mat-q\n");
printf("                        use cuBLAS instead of custom mul_mat_q CUDA kernels.\n");
printf("                        Not recommended since this is both slower and uses more VRAM.\n");
#endif
// 打印提示信息，指示设置模型路径，默认为params.model的值
printf("  -m FNAME, --model FNAME\n");
printf("                        model path (default: %s)\n", params.model.c_str());
// 打印提示信息，指示设置模型别名，将添加为完成响应中的`model`字段
printf("  -a ALIAS, --alias ALIAS\n");
printf("                        set an alias for the model, will be added as `model` field in completion response\n");
// 打印提示信息，指示应用LoRA适配器（意味着--no-mmap）
printf("  --lora FNAME          apply LoRA adapter (implies --no-mmap)\n");
// 打印提示信息，指示可选的模型用作LoRA适配器修改的层的基础
printf("  --lora-base FNAME     optional model to use as a base for the layers modified by the LoRA adapter\n");
// 打印提示信息，指示要监听的IP地址（默认为sparams.hostname的值）
printf("  --host                ip address to listen (default  (default: %s)\n", sparams.hostname.c_str());
// 打印提示信息，指示要监听的端口号（默认为sparams.port的值）
printf("  --port PORT           port to listen (default  (default: %d)\n", sparams.port);
// 打印提示信息，指示要提供静态文件的路径（默认为sparams.public_path的值）
printf("  --path PUBLIC_PATH    path from which to serve static files (default %s)\n", sparams.public_path.c_str());
    # 打印服务器读写超时时间的帮助信息
    printf("  -to N, --timeout N    server read/write timeout in seconds (default: %d)\n", sparams.read_timeout);
    # 打印是否启用嵌入向量输出的帮助信息
    printf("  --embedding           enable embedding vector output (default: %s)\n", params.embedding ? "enabled" : "disabled");
    # 打印并行处理请求的槽位数的帮助信息
    printf("  -np N, --parallel N   number of slots for process requests (default: %d)\n", params.n_parallel);
    # 打印是否启用连续批处理的帮助信息
    printf("  -cb, --cont-batching  enable continuous batching (a.k.a dynamic batching) (default: disabled)\n");
    # 打印系统提示文件的帮助信息
    printf("    -spf FNAME, --system-prompt-file FNAME\n");
    printf("                        Set a file to load a system prompt (initial prompt of all slots), this is useful for chat applications.\n");
    # 打印VRAM预算的帮助信息
    printf("  --vram-budget N       VRAM budget in GiB (default: -1, -1 = available VRAM)\n");
    # 打印LLaVA的多模态投影文件路径的帮助信息
    printf("  --mmproj MMPROJ_FILE  path to a multimodal projector file for LLaVA.\n");
    # 打印空行
    printf("\n");
}

# 解析服务器参数
static void server_params_parse(int argc, char **argv, server_params &sparams,
                                gpt_params &params, llama_server_context& llama)
{
    # 初始化默认参数
    gpt_params default_params;
    server_params default_sparams;
    std::string arg;
    bool invalid_param = false;

    # 遍历命令行参数
    for (int i = 1; i < argc; i++)
# 遍历命令行参数
{
    # 获取当前参数
    arg = argv[i];
    # 判断当前参数是否为"--port"
    if (arg == "--port")
    {
        # 如果是"--port"，则获取下一个参数作为端口号
        if (++i >= argc)
        {
            # 如果没有下一个参数，则标记参数无效并跳出循环
            invalid_param = true;
            break;
        }
        # 将下一个参数转换为整数并赋值给端口号
        sparams.port = std::stoi(argv[i]);
    }
    # 如果当前参数不是"--port"，则判断是否为"--host"
    else if (arg == "--host")
    {
        # 如果是"--host"，则获取下一个参数作为主机名
        if (++i >= argc)
        {
            # 如果没有下一个参数，则标记参数无效并跳出循环
            invalid_param = true;
            break;
        }
        # 将下一个参数赋值给主机名
        sparams.hostname = argv[i];
    }
}
        else if (arg == "--path")
        {
            // 如果参数为"--path"，则获取下一个参数作为公共路径
            if (++i >= argc)
            {
                // 如果没有下一个参数，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            sparams.public_path = argv[i];
        }
        else if (arg == "--timeout" || arg == "-to")
        {
            // 如果参数为"--timeout"或"-to"，则获取下一个参数作为读写超时时间
            if (++i >= argc)
            {
                // 如果没有下一个参数，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            // 将参数转换为整数并设置为读写超时时间
            sparams.read_timeout = std::stoi(argv[i]);
            sparams.write_timeout = std::stoi(argv[i]);
        }
        else if (arg == "-m" || arg == "--model")
```

        {
            // 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将参数赋值给模型参数
            params.model = argv[i];
        }
        // 如果参数为"-a"或"--alias"
        else if (arg == "-a" || arg == "--alias")
        {
            // 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将参数赋值给模型别名参数
            params.model_alias = argv[i];
        }
        // 如果参数为"-h"或"--help"
        else if (arg == "-h" || arg == "--help")
        {
            // 打印帮助信息
            server_print_usage(argv[0], default_params, default_sparams);
        // 退出程序
        exit(0);
        // 如果参数是 -c、--ctx-size 或 --ctx_size
        else if (arg == "-c" || arg == "--ctx-size" || arg == "--ctx_size")
        {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将参数转换为整数并赋值给 params.n_ctx
            params.n_ctx = std::stoi(argv[i]);
        }
        // 如果参数是 --rope-scaling
        else if (arg == "--rope-scaling")
        {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将参数转换为字符串
            std::string value(argv[i]);
            // 如果参数值是 "none"，设置 params.rope_scaling_type 为 LLAMA_ROPE_SCALING_NONE
            if (value == "none")   { params.rope_scaling_type = LLAMA_ROPE_SCALING_NONE; }
        // 如果参数值为"linear"，则设置绳索缩放类型为线性
        else if (value == "linear") { params.rope_scaling_type = LLAMA_ROPE_SCALING_LINEAR; }
        // 如果参数值为"yarn"，则设置绳索缩放类型为纱线
        else if (value == "yarn")   { params.rope_scaling_type = LLAMA_ROPE_SCALING_YARN; }
        // 如果参数值不是"linear"或"yarn"，则标记参数无效并跳出循环
        else { invalid_param = true; break; }
        
        // 如果参数为"--rope-freq-base"
        else if (arg == "--rope-freq-base")
        {
            // 如果下一个参数索引超出范围，标记参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为浮点数，并赋值给绳索频率基数
            params.rope_freq_base = std::stof(argv[i]);
        }
        
        // 如果参数为"--rope-freq-scale"
        else if (arg == "--rope-freq-scale")
        {
            // 如果下一个参数索引超出范围，标记参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为浮点数，并赋值给绳索频率缩放
        # 如果参数是 "--rope-freq-scale"，则将下一个参数转换为浮点数并赋值给 params.rope_freq_scale
        if (arg == "--rope-freq-scale")
        {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.rope_freq_scale = std::stof(argv[i]);
        }
        # 如果参数是 "--yarn-ext-factor"，则将下一个参数转换为浮点数并赋值给 params.yarn_ext_factor
        else if (arg == "--yarn-ext-factor")
        {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.yarn_ext_factor = std::stof(argv[i]);
        }
        # 如果参数是 "--yarn-attn-factor"，则将下一个参数转换为浮点数并赋值给 params.yarn_attn_factor
        else if (arg == "--yarn-attn-factor")
        {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.yarn_attn_factor = std::stof(argv[i]);
        }
        # 如果参数是 "--yarn-beta-fast"，则...
# 如果参数索引超出范围，设置参数无效并跳出循环
if (++i >= argc) {
    invalid_param = true;
    break;
}
# 将参数转换为浮点数，并赋值给对应的参数
params.yarn_beta_fast = std::stof(argv[i]);
# 如果参数为"--yarn-beta-slow"
else if (arg == "--yarn-beta-slow")
{
    # 如果参数索引超出范围，设置参数无效并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    # 将参数转换为浮点数，并赋值给对应的参数
    params.yarn_beta_slow = std::stof(argv[i]);
}
# 如果参数为"--memory-f32"或"--memory_f32"
else if (arg == "--memory-f32" || arg == "--memory_f32")
{
    # 将参数memory_f16设置为false
    params.memory_f16 = false;
}
# 如果参数为"--threads"或"-t"
else if (arg == "--threads" || arg == "-t")
{
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            # 将参数转换为整数并赋值给线程数
            params.n_threads = std::stoi(argv[i]);
        }
        # 如果参数为--threads-batch或-tb
        else if (arg == "--threads-batch" || arg == "-tb")
        {
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            # 将参数转换为整数并赋值给批处理线程数
            params.n_threads_batch = std::stoi(argv[i]);
        }
        # 如果参数为-b或--batch-size
        else if (arg == "-b" || arg == "--batch-size")
        {
            # 如果参数索引超出范围，设置参数无效并跳出循环
            if (++i >= argc)
            {
                // 检查参数是否有效，如果无效则跳出循环
                invalid_param = true;
                break;
            }
            // 将参数转换为整数，并限制最大值为512
            params.n_batch = std::stoi(argv[i]);
            params.n_batch = std::min(512, params.n_batch);
        }
        // 如果参数为"--gpu-layers"、"-ngl"或"--n-gpu-layers"，则执行以下操作
        else if (arg == "--gpu-layers" || arg == "-ngl" || arg == "--n-gpu-layers")
        {
            // 如果参数不足，则将参数设置为无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 如果编译时支持 GPU 加速，则将参数转换为整数并赋值给params.n_gpu_layers
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
            params.n_gpu_layers = std::stoi(argv[i]);
#else
            // 如果未编译支持 GPU 加速，则记录警告信息
            LOG_WARNING("Not compiled with GPU offload support, --n-gpu-layers option will be ignored. "
                        "See main README.md for information on enabling GPU BLAS support",
                        {{"n_gpu_layers", params.n_gpu_layers}});
#endif
        }
        else if (arg == "--tensor-split" || arg == "-ts")
        {
            // 检查下一个参数是否存在
            if (++i >= argc)
            {
                // 如果不存在，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 获取下一个参数
            std::string arg_next = argv[i];

            // 通过逗号和斜杠分割字符串
            const std::regex regex{R"([,/]+)"};
            // 使用正则表达式分割字符串
            std::sregex_token_iterator it{arg_next.begin(), arg_next.end(), regex, -1};
            // 将分割后的字符串存储到向量中
            std::vector<std::string> split_arg{it, {}};
            // 确保分割后的字符串数量不超过最大设备数
            GGML_ASSERT(split_arg.size() <= LLAMA_MAX_DEVICES);

            // 遍历设备
            for (size_t i_device = 0; i_device < LLAMA_MAX_DEVICES; ++i_device)
            {
                // 如果当前设备小于分割后的字符串数量
                if (i_device < split_arg.size())
// 如果编译时使用了 cuBLAS，则设置参数 tensor_split[i_device] 为 split_arg[i_device] 的浮点数值
// 否则，设置参数 tensor_split[i_device] 为 0.0
{
    // 如果编译时使用了 cuBLAS，则设置参数 tensor_split[i_device] 为 split_arg[i_device] 的浮点数值
    params.tensor_split[i_device] = std::stof(split_arg[i_device]);
}
// 否则，设置参数 tensor_split[i_device] 为 0.0f
else
{
    params.tensor_split[i_device] = 0.0f;
}
// 如果未使用 cuBLAS 编译，则记录警告信息
#else
LOG_WARNING("llama.cpp was compiled without cuBLAS. It is not possible to set a tensor split.\n", {});
#endif // GGML_USE_CUBLAS
// 如果命令行参数为 "--no-mul-mat-q" 或 "-nommq"，则根据编译时是否使用 cuBLAS 设置参数 mul_mat_q
else if (arg == "--no-mul-mat-q" || arg == "-nommq")
{
#ifdef GGML_USE_CUBLAS
    // 如果编译时使用了 cuBLAS，则设置参数 mul_mat_q 为 false
    params.mul_mat_q = false;
#else
    // 否则，记录警告信息
    LOG_WARNING("warning: llama.cpp was compiled without cuBLAS. Disabling mul_mat_q kernels has no effect.\n", {});
#endif // GGML_USE_CUBLAS
}
        else if (arg == "--main-gpu" || arg == "-mg")
        {
            // 如果参数是"--main-gpu"或者"-mg"
            if (++i >= argc)
            {
                // 如果参数不完整，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 如果使用了cuBLAS，将参数转换为整数并赋值给params.main_gpu
            params.main_gpu = std::stoi(argv[i]);
#else
            // 如果没有使用cuBLAS，记录警告信息
            LOG_WARNING("llama.cpp was compiled without cuBLAS. It is not possible to set a main GPU.", {});
#endif
        }
        else if (arg == "--lora")
        {
            // 如果参数是"--lora"
            if (++i >= argc)
            {
                // 如果参数不完整，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            # 将参数和权重添加到lora_adapter的vector中
            params.lora_adapter.push_back(std::make_tuple(argv[i], 1.0f));
            # 设置use_mmap为false
            params.use_mmap = false;
        }
        # 如果参数为"--lora-scaled"
        else if (arg == "--lora-scaled")
        {
            # 如果参数不足，设置invalid_param为true并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            # 获取lora_adapter的值
            const char * lora_adapter = argv[i];
            # 如果参数不足，设置invalid_param为true并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            # 将lora_adapter和权重添加到lora_adapter的vector中
            params.lora_adapter.push_back(std::make_tuple(lora_adapter, std::stof(argv[i]));
            # 设置use_mmap为false
            params.use_mmap = false;
        }
        # 如果参数为"--lora-base"
        else if (arg == "--lora-base")
        {
            // 如果参数索引超出范围，将参数无效标志设为true，并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将参数赋值给params.lora_base
            params.lora_base = argv[i];
        }
        // 如果参数为"-v"或"--verbose"
        else if (arg == "-v" || arg == "--verbose")
        {
            // 如果SERVER_VERBOSE不等于1，记录警告日志
#if SERVER_VERBOSE != 1
            LOG_WARNING("server.cpp is not built with verbose logging.", {});
#else
            // 否则将server_verbose标志设为true
            server_verbose = true;
#endif
        }
        // 如果参数为"--mlock"
        else if (arg == "--mlock")
        {
            // 将params.use_mlock标志设为true
            params.use_mlock = true;
        }
        else if (arg == "--no-mmap")
        {
            // 如果参数为"--no-mmap"，则将params.use_mmap设置为false，表示不使用内存映射
            params.use_mmap = false;
        }
        else if (arg == "--numa")
        {
            // 如果参数为"--numa"，则将params.numa设置为true，表示启用NUMA
            params.numa = true;
        }
        else if (arg == "--embedding")
        {
            // 如果参数为"--embedding"，则将params.embedding设置为true，表示启用嵌入
            params.embedding = true;
        }
        else if (arg == "-cb" || arg == "--cont-batching")
        {
            // 如果参数为"-cb"或"--cont-batching"，则将params.cont_batching设置为true，表示启用连续批处理
            params.cont_batching = true;
        }
        else if (arg == "-np" || arg == "--parallel")
        {
            // 如果参数为"-np"或"--parallel"，则检查下一个参数是否存在，用于并行处理
            if (++i >= argc)
            {
                # 检查参数是否为无效值，如果是则设置标志位并跳出循环
                invalid_param = true;
                break;
            }
            # 将参数转换为整数并赋值给params.n_parallel
            params.n_parallel = std::stoi(argv[i]);
        } else if (arg == "-n" || arg == "--n-predict")
        {
            # 检查参数是否为无效值，如果是则设置标志位并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            # 将参数转换为整数并赋值给params.n_predict
            params.n_predict = std::stoi(argv[i]);
        } else if (arg == "-spf" || arg == "--system-prompt-file")
        {
            # 检查参数是否为无效值，如果是则设置标志位并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            # 打开文件流，准备读取文件内容
            std::ifstream file(argv[i]);
            // 如果文件打开失败，输出错误信息并设置参数为无效，然后跳出循环
            if (!file) {
                fprintf(stderr, "error: failed to open file '%s'\n", argv[i]);
                invalid_param = true;
                break;
            }
            // 创建一个字符串来存储文件内容
            std::string systm_content;
            // 将文件内容拷贝到字符串中
            std::copy(
                std::istreambuf_iterator<char>(file),
                std::istreambuf_iterator<char>(),
                std::back_inserter(systm_content)
            );
            // 处理系统提示数据
            llama.process_system_prompt_data(json::parse(systm_content));
        }
        // 如果参数是"--vram-budget"
        else if (arg == "--vram-budget")
        {
            // 如果参数不足，设置参数为无效，然后跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 如果编译时使用了 cuBLAS，则将参数转换为浮点数并赋值给 VRAM 预算
            params.vram_budget_gb = std::stof(argv[i]);
#else
            // 如果没有使用 cuBLAS 编译，则打印警告信息
            fprintf(stderr, "warning: PowerInfer was compiled without cuBLAS. It is not possible to set a VRAM budget.\n");
#endif
        }
        else if (arg == "--reset-gpu-index")
        {
            // 如果参数为 "--reset-gpu-index"，则将 reset_gpu_index 设置为 true
            params.reset_gpu_index = true;
        } 
        else if (arg == "--disable-gpu-index")
        {
            // 如果参数为 "--disable-gpu-index"，则将 disable_gpu_index 设置为 true
            params.disale_gpu_index = true;
        }
        else if(arg == "--mmproj")
        {
            // 如果参数为 "--mmproj"，则检查下一个参数是否存在
            if (++i >= argc)
            {
                // 如果下一个参数不存在，则将 invalid_param 设置为 true 并跳出循环
                invalid_param = true;
                break;
    }
    // 如果参数为 mmproj，则将其赋值给params.mmproj
    params.mmproj = argv[i];
}
else
{
    // 如果参数不是已知的，则打印错误信息并退出程序
    fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
    server_print_usage(argv[0], default_params, default_sparams);
    exit(1);
}
}

// 如果存在无效参数，则打印错误信息并退出程序
if (invalid_param)
{
    fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
    server_print_usage(argv[0], default_params, default_sparams);
    exit(1);
}
}

// 格式化部分响应的 JSON 数据
static json format_partial_response(
// 定义一个函数，接受 llama_server_context 和 llama_client_slot 对象的引用，以及 content 和 probs 参数
llama_server_context &llama, llama_client_slot *slot, const std::string &content, const std::vector<completion_token_output> &probs
) {
    // 创建一个 JSON 对象 res，包含 content、stop、slot_id 和 multimodal 字段
    json res = json
    {
        {"content",    content },
        {"stop",       false},
        {"slot_id",    slot->id },
        {"multimodal", llama.multimodal }
    };

    // 如果 slot->sparams.n_probs 大于 0，则将 completion_probabilities 字段添加到 res 中
    if (slot->sparams.n_probs > 0)
    {
        res["completion_probabilities"] = probs_vector_to_json(llama.ctx, probs);
    }

    // 返回 res
    return res;
}

// 定义一个静态函数，接受 tokens 参数
static json format_tokenizer_response(const std::vector<llama_token> &tokens)
{
# 返回一个 JSON 对象，包含名为 "tokens" 的键和 tokens 变量的值
    return json{
        {"tokens", tokens}};
}

# 格式化去标记化响应的内容，返回一个 JSON 对象，包含名为 "content" 的键和 content 变量的值
static json format_detokenized_response(std::string content)
{
    return json{
        {"content", content}};
}

# 记录服务器请求的日志信息，包括远程地址、远程端口、响应状态、请求方法、请求路径和请求参数
static void log_server_request(const httplib::Request &req, const httplib::Response &res)
{
    LOG_INFO("request", {
                            {"remote_addr", req.remote_addr},
                            {"remote_port", req.remote_port},
                            {"status", res.status},
                            {"method", req.method},
                            {"path", req.path},
                            {"params", req.params},
// 匿名函数，计算生成的文本长度
auto add_strlen = [=](size_t sum, const completion_token_output & cto) { return sum + translator(cto).size(); };
    // 计算生成文本的总长度
    const size_t len = std::accumulate(gtps.begin(), gtps.end(), size_t(0), add_strlen);
    // 如果生成文本的容量小于当前大小加上总长度，就扩展容量
    if (slot->generated_text.capacity() < slot->generated_text.size() + len)
    {
        slot->generated_text.reserve(slot->generated_text.size() + len);
    }
    // 遍历生成文本片段，将其翻译并添加到生成文本中
    for (const completion_token_output & cto : gtps)
    {
        slot->generated_text += translator(cto);
    }
}

int main(int argc, char **argv)
{
    // 为本示例所需的参数
    gpt_params params;
    server_params sparams;

    // 包含 llama 上下文和推理的结构体
    llama_server_context llama;
    // 解析服务器参数，将参数存储到相应的结构体中
    server_params_parse(argc, argv, sparams, params, llama);

    // 如果模型别名为"unknown"，则将模型别名设置为模型名称
    if (params.model_alias == "unknown")
    {
        params.model_alias = params.model;
    }

    // 初始化 LLAMA 后端，根据参数中的 NUMA 信息
    llama_backend_init(params.numa);

    // 记录构建信息和提交信息到日志
    LOG_INFO("build info", {{"build", LLAMA_BUILD_NUMBER},
                            {"commit", LLAMA_COMMIT}});

    // 记录系统信息到日志，包括线程数、批处理线程数、总线程数和系统信息
    LOG_INFO("system info", {
                                {"n_threads", params.n_threads},
                                {"n_threads_batch", params.n_threads_batch},
                                {"total_threads", std::thread::hardware_concurrency()},
                                {"system_info", llama_print_system_info()},
                            });

    // 加载模型
    // load the model
    // 如果 llama 模型加载失败，则返回 1
    if (!llama.load_model(params))
    {
        return 1;
    }

    // 初始化 llama
    llama.initialize();

    // 创建一个 HTTP 服务器对象
    httplib::Server svr;

    // 设置默认的 HTTP 头部信息
    svr.set_default_headers({{"Server", "llama.cpp"},
                             {"Access-Control-Allow-Origin", "*"},
                             {"Access-Control-Allow-Headers", "content-type"}});

    // 当在 public 路径下找不到 index.html 时调用，返回 index.html 的内容
    svr.Get("/", [](const httplib::Request &, httplib::Response &res)
            {
                res.set_content(reinterpret_cast<const char*>(&index_html), index_html_len, "text/html");
                return false;
            });
// 如果在公共路径中找不到 index.js，则调用此函数
svr.Get("/index.js", [](const httplib::Request &, httplib::Response &res)
        {
            // 设置响应内容为 index_js 的文本/javascript类型
            res.set_content(reinterpret_cast<const char *>(&index_js), index_js_len, "text/javascript");
            return false;
        });

// 如果在公共路径中找不到 index.html，则调用此函数
svr.Get("/completion.js", [](const httplib::Request &, httplib::Response &res)
        {
            // 设置响应内容为 completion_js 的application/javascript类型
            res.set_content(reinterpret_cast<const char*>(&completion_js), completion_js_len, "application/javascript");
            return false;
        });

// 如果在公共路径中找不到 index.html，则调用此函数
svr.Get("/json-schema-to-grammar.mjs", [](const httplib::Request &, httplib::Response &res)
        {
            // 设置响应内容为 json_schema_to_grammar_mjs 的application/javascript类型
            res.set_content(reinterpret_cast<const char*>(&json_schema_to_grammar_mjs), json_schema_to_grammar_mjs_len, "application/javascript");
            return false;
        });
// 设置服务器路由，当收到"/props"请求时，返回用户和助手的名称
svr.Get("/props", [&llama](const httplib::Request & /*req*/, httplib::Response &res)
        {
            // 设置响应头，允许跨域访问
            res.set_header("Access-Control-Allow-Origin", "*");
            // 创建包含用户和助手名称的 JSON 数据
            json data = {
                { "user_name",      llama.name_user.c_str() },
                { "assistant_name", llama.name_assistant.c_str() }
            };
            // 设置响应内容为 JSON 格式
            res.set_content(data.dump(), "application/json");
        });

// 设置服务器路由，当收到"/completion"的 POST 请求时，处理请求并返回结果
svr.Post("/completion", [&llama](const httplib::Request &req, httplib::Response &res)
        {
            // 解析请求中的 JSON 数据
            json data = json::parse(req.body);
            // 调用 llama 对象的 request_completion 方法，获取任务 ID
            const int task_id = llama.request_completion(data, false, false);
            // 如果请求中没有 "stream" 字段
            if (!json_value(data, "stream", false)) {
                // 定义完成文本变量
                std::string completion_text;
                // 调用 llama 对象的 next_result 方法，获取任务结果
                task_result result = llama.next_result(task_id);
                // 如果没有错误且任务已完成
                if (!result.error && result.stop) {
                    // 设置响应内容为任务结果的 JSON 格式
                    res.set_content(result.result_json.dump(-1, ' ', false, json::error_handler_t::replace), "application/json");
                    }
                    else
                    {
                        // 如果任务结果不存在，设置状态码为404，并返回结果内容
                        res.status = 404;
                        res.set_content(result.result_json["content"], "text/plain");
                        return;
                    }
                } else {
                    // 定义一个匿名函数作为chunked_content_provider，用于处理数据流
                    const auto chunked_content_provider = [task_id, &llama](size_t, httplib::DataSink & sink)
                    {
                        // 循环获取任务结果并发送数据流
                        while (true)
                        {
                            task_result result = llama.next_result(task_id);
                            // 如果没有错误，将结果转换为字符串并发送
                            if (!result.error) {
                                const std::string str =
                                "data: " +
                                result.result_json.dump(-1, ' ', false, json::error_handler_t::replace) +
                                "\n\n";
                                LOG_VERBOSE("data stream", {
                                    { "to_send", str }
                    });
                    // 如果写入失败，返回false
                    if (!sink.write(str.c_str(), str.size()))
                    {
                        return false;
                    }
                    // 如果任务被停止，跳出循环
                    if (result.stop) {
                        break;
                    } else {
                        // 否则跳出循环
                        break;
                    }
                }
                // 完成写入
                sink.done();
                // 返回true
                return true;
            };

            // 完成时取消任务
            auto on_complete = [task_id, &llama] (bool)
            {
                llama.request_cancel(task_id);
// 定义一个 POST 请求处理函数，处理 "/infill" 路径的请求
svr.Post("/infill", [&llama](const httplib::Request &req, httplib::Response &res)
{
    // 解析请求体中的 JSON 数据
    json data = json::parse(req.body);
    // 发起一个请求完成的任务，并获取任务 ID
    const int task_id = llama.request_completion(data, true, false);
    // 如果请求体中没有 "stream" 字段
    if (!json_value(data, "stream", false)) {
        // 定义一个字符串来存储完成的文本
        std::string completion_text;
        // 获取任务的结果
        task_result result = llama.next_result(task_id);
        // 如果没有错误并且任务已经完成
        if (!result.error && result.stop)
        {
            // 设置响应内容为任务结果的 JSON 数据
            res.set_content(result.result_json.dump(-1, ' ', false, json::error_handler_t::replace), "application/json");
        }
        else
        {
            // 设置响应状态码为 404
            res.status = 404;
// 设置 HTTP 响应的内容为 result.result_json["content"]，格式为纯文本
res.set_content(result.result_json["content"], "text/plain");
// 结束函数执行
return;
// 如果条件不成立，执行以下代码
} else {
    // 定义一个 chunked_content_provider 函数，该函数接受 task_id 和 llama 作为参数
    const auto chunked_content_provider = [task_id, &llama](size_t, httplib::DataSink & sink) {
        // 循环执行以下代码
        while (true)
        {
            // 从 llama 获取下一个任务结果
            task_result result = llama.next_result(task_id);
            // 如果没有错误
            if (!result.error) {
                // 构建数据流字符串
                const std::string str =
                "data: " +
                result.result_json.dump(-1, ' ', false, json::error_handler_t::replace) +
                "\n\n";
                // 记录数据流日志
                LOG_VERBOSE("data stream", {
                    { "to_send", str }
                });
                // 将数据流字符串写入到 sink 中
                if (!sink.write(str.c_str(), str.size()))
                {
                    // 如果写入失败，返回 false
                    return false;
                }
# 如果结果为停止状态，则跳出循环
if (result.stop)
{
    break;
}
# 如果结果不为停止状态，则跳出循环
else
{
    break;
}
# 完成数据传输，执行sink的done方法
sink.done();
# 返回true
return true;
};

# 完成任务后的回调函数，取消任务
auto on_complete = [task_id, &llama] (bool)
{
    llama.request_cancel(task_id);
                    };

                    // 设置分块内容提供程序为"text/event-stream"类型，使用chunked_content_provider函数，完成时调用on_complete函数
                    res.set_chunked_content_provider("text/event-stream", chunked_content_provider, on_complete);
                }
            });

    // 处理GET请求，返回模型属性的JSON数据
    svr.Get("/model.json", [&llama](const httplib::Request &, httplib::Response &res)
            {
                // 获取llama对象的模型属性数据
                const json data = llama.get_model_props();
                // 返回JSON格式的数据
                return res.set_content(data.dump(), "application/json");
            });

    // 处理OPTIONS请求，返回空的JSON数据
    svr.Options(R"(/.*)", [](const httplib::Request &, httplib::Response &res)
                { return res.set_content("", "application/json"); });

    // 处理POST请求，对内容进行分词处理
    svr.Post("/tokenize", [&llama](const httplib::Request &req, httplib::Response &res)
            {
                // 解析请求的body内容为JSON格式
                const json body = json::parse(req.body);
                // 创建llama_token对象的vector
                std::vector<llama_token> tokens;
                // 如果body中包含"content"字段
                if (body.count("content") != 0)
// 从请求体中获取内容，使用 llama.tokenize 方法对内容进行分词处理
{
    tokens = llama.tokenize(body["content"], false);
}
// 将分词处理后的结果转换为 JSON 格式的数据
const json data = format_tokenizer_response(tokens);
// 将 JSON 数据以 application/json 格式返回
return res.set_content(data.dump(), "application/json");
});

// 处理 detokenize 请求
svr.Post("/detokenize", [&llama](const httplib::Request &req, httplib::Response &res)
{
    // 从请求体中解析 JSON 数据
    const json body = json::parse(req.body);
    std::string content;
    // 如果请求体中包含 tokens 字段
    if (body.count("tokens") != 0)
    {
        // 从请求体中获取 tokens 数据
        const std::vector<llama_token> tokens = body["tokens"];
        // 使用 tokens_to_str 方法将 tokens 转换为字符串
        content = tokens_to_str(llama.ctx, tokens.cbegin(), tokens.cend());
    }

    // 将处理后的内容转换为 JSON 格式的数据
    const json data = format_detokenized_response(content);
    // 将 JSON 数据以 application/json 格式返回
    return res.set_content(data.dump(), "application/json");
});
# 使用HTTP POST方法将数据发送到"/embedding"端点
svr.Post("/embedding", [&llama](const httplib::Request &req, httplib::Response &res)
        {
            # 解析请求的JSON数据
            const json body = json::parse(req.body);
            json prompt;
            # 如果请求中包含"content"字段，则将其赋给prompt变量
            if (body.count("content") != 0)
            {
                prompt = body["content"];
            }
            # 否则将prompt变量赋为空字符串
            else
            {
                prompt = "";
            }
            # 发送请求到llama对象，获取任务ID
            const int task_id = llama.request_completion({ {"prompt", prompt}, { "n_predict", 0} }, false, true);
            # 获取任务结果
            task_result result = llama.next_result(task_id);
            # 将结果以JSON格式返回
            return res.set_content(result.result_json.dump(), "application/json");
        });

# 设置日志记录器为log_server_request
svr.set_logger(log_server_request);
    # 设置异常处理程序，当发生异常时执行以下操作
    svr.set_exception_handler([](const httplib::Request &, httplib::Response &res, std::exception_ptr ep)
            {
                # 定义格式字符串，用于构建错误信息
                const char fmt[] = "500 Internal Server Error\n%s";
                # 定义缓冲区，用于存储错误信息
                char buf[BUFSIZ];
                try
                {
                    # 重新抛出异常，获取异常信息
                    std::rethrow_exception(std::move(ep));
                }
                # 捕获特定类型的异常
                catch (std::exception &e)
                {
                    # 根据异常信息格式化错误信息
                    snprintf(buf, sizeof(buf), fmt, e.what());
                }
                # 捕获其他类型的异常
                catch (...)
                {
                    # 根据默认错误信息格式化错误信息
                    snprintf(buf, sizeof(buf), fmt, "Unknown Exception");
                }
                # 设置响应内容为错误信息
                res.set_content(buf, "text/plain");
                # 设置响应状态码为500
                res.status = 500;
            });
    // 设置错误处理程序，根据不同的状态码进行不同的处理
    svr.set_error_handler([](const httplib::Request &, httplib::Response &res)
            {
                // 如果状态码为400，返回"Invalid request"，内容类型为"text/plain"
                if (res.status == 400)
                {
                    res.set_content("Invalid request", "text/plain");
                }
                // 如果状态码不为500，返回"File Not Found"，内容类型为"text/plain"，并将状态码改为404
                else if (res.status != 500)
                {
                    res.set_content("File Not Found", "text/plain");
                    res.status = 404;
                }
            });

    // 设置读取超时时间和写入超时时间
    svr.set_read_timeout (sparams.read_timeout);
    svr.set_write_timeout(sparams.write_timeout);

    // 绑定服务器到指定的主机名和端口
    if (!svr.bind_to_port(sparams.hostname, sparams.port))
    {
        // 如果无法绑定到服务器套接字，打印错误信息
        fprintf(stderr, "\ncouldn't bind to server socket: hostname=%s port=%d\n\n", sparams.hostname.c_str(), sparams.port);
    // 返回值为1
    return 1;
    }

    // 设置静态文件服务的基本目录
    svr.set_base_dir(sparams.public_path);

    // 记录服务器监听地址和端口，使其可通过ctrl+点击跳转
    LOG_TEE("\nllama server listening at http://%s:%d\n\n", sparams.hostname.c_str(), sparams.port);

    // 记录HTTP服务器监听信息
    LOG_INFO("HTTP server listening", {
                                          {"hostname", sparams.hostname},
                                          {"port", sparams.port},
                                      });

    // 在线程中运行HTTP服务器 - 参见下面的注释
    std::thread t([&]()
            {
                // 如果绑定后无法监听，则返回1
                if (!svr.listen_after_bind())
                {
                    return 1;
// GG: 如果我将主循环放在一个线程中，当在调试模式下进行第一个请求时会崩溃！？
//     "Bus error: 10" - 这是在 macOS 上的情况，在 Linux 上不会崩溃
//std::thread t2([&]()
{
    bool running = true;
    while (running)
    {
        running = llama.update_slots();
    }
}
//);

// 等待线程 t 的结束
t.join();

// 释放 llama 的后端资源
llama_backend_free();
# 返回整数值0，结束函数的执行。
```