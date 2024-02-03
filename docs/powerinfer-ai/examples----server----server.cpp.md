# `PowerInfer\examples\server\server.cpp`

```cpp
#include "common.h"
#include "llama.h"
#include "grammar-parser.h"

#include "../llava/clip.h"

#include "stb_image.h"

#ifndef NDEBUG
// 在调试模式下使服务器崩溃，否则发送 HTTP 500 错误
#define CPPHTTPLIB_NO_EXCEPTIONS 1
#endif

#include "httplib.h"
#include "json.hpp"

// 自动生成的文件（使用 ./deps.sh 更新）
#include "index.html.hpp"
#include "index.js.hpp"
#include "completion.js.hpp"
#include "json-schema-to-grammar.mjs.hpp"

#include <cstddef>
#include <thread>
#include <mutex>
#include <chrono>

#ifndef SERVER_VERBOSE
#define SERVER_VERBOSE 1
#endif

using json = nlohmann::json;

// 服务器参数结构体
struct server_params
{
    std::string hostname = "127.0.0.1";
    std::string public_path = "examples/server/public";
    int32_t port = 8080;
    int32_t read_timeout = 600;
    int32_t write_timeout = 600;
};

// 服务器是否详细输出日志
static bool server_verbose = false;

// 如果 SERVER_VERBOSE 不等于 1，则不输出详细日志
#if SERVER_VERBOSE != 1
#define LOG_VERBOSE(MSG, ...)
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

// 输出错误日志
#define LOG_ERROR(  MSG, ...) server_log("ERROR",   __func__, __LINE__, MSG, __VA_ARGS__)
// 输出警告日志
#define LOG_WARNING(MSG, ...) server_log("WARNING", __func__, __LINE__, MSG, __VA_ARGS__)
// 输出信息日志
#define LOG_INFO(   MSG, ...) server_log("INFO",    __func__, __LINE__, MSG, __VA_ARGS__)

//
// base64 工具（TODO: 将来移到 common 中）
//

// base64 字符集
static const std::string base64_chars =
             "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
             "abcdefghijklmnopqrstuvwxyz"
             "0123456789+/";
// 判断一个字符是否为 base64 编码字符
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

    int in_len = encoded_string.size();  // 获取输入字符串的长度

    uint8_t char_array_4[4];  // 创建一个长度为 4 的字符数组
    uint8_t char_array_3[3];  // 创建一个长度为 3 的字符数组

    std::vector<uint8_t> ret;  // 创建一个存储解码结果的向量

    // 循环解码输入字符串
    while (in_len-- && (encoded_string[in_] != '=') && is_base64(encoded_string[in_]))
    {
        char_array_4[i++] = encoded_string[in_]; in_++;  // 将输入字符串中的字符存入 char_array_4 中，并更新索引
        if (i == 4)  // 如果 char_array_4 已经存满 4 个字符
        {
            for (i = 0; i <4; i++)  // 遍历 char_array_4
            {
                char_array_4[i] = base64_chars.find(char_array_4[i]);  // 将 base64 字符转换为对应的索引值
            }

            // 根据 base64 解码规则，将 char_array_4 中的字符解码为 char_array_3 中的字符
            char_array_3[0] = ((char_array_4[0]      ) << 2) + ((char_array_4[1] & 0x30) >> 4);
            char_array_3[1] = ((char_array_4[1] & 0xf) << 4) + ((char_array_4[2] & 0x3c) >> 2);
            char_array_3[2] = ((char_array_4[2] & 0x3) << 6) +   char_array_4[3];

            // 将解码后的字符存入结果向量中
            for (i = 0; (i < 3); i++)
            {
                ret.push_back(char_array_3[i]);
            }
            i = 0;  // 重置索引 i
        }
    }

    // 处理剩余的字符
    if (i)
    {
        for (j = i; j <4; j++)
        {
            char_array_4[j] = 0;  // 将剩余位置填充为 0
        }

        for (j = 0; j <4; j++)
        {
            char_array_4[j] = base64_chars.find(char_array_4[j]);  // 将 base64 字符转换为对应的索引值
        }

        // 根据 base64 解码规则，将 char_array_4 中的字符解码为 char_array_3 中的字符
        char_array_3[0] = ((char_array_4[0]      ) << 2) + ((char_array_4[1] & 0x30) >> 4);
        char_array_3[1] = ((char_array_4[1] & 0xf) << 4) + ((char_array_4[2] & 0x3c) >> 2);
        char_array_3[2] = ((char_array_4[2] & 0x3) << 6) +   char_array_4[3];

        // 将解码后的字符存入结果向量中
        for (j = 0; (j < i - 1); j++)
        {
            ret.push_back(char_array_3[j]);
        }
    }

    return ret;  // 返回解码结果向量
}

//
// parallel
//

// 定义任务类型枚举
enum task_type {
    COMPLETION_TASK,  // 完成任务
    CANCEL_TASK  // 取消任务
};

// 定义任务服务器结构体
struct task_server {
    int id;  // 任务 ID
    int target_id;  // 目标 ID
    task_type type;  // 任务类型
    json data;  // 任务数据
    bool infill_mode = false;  // 是否为填充模式
    bool embedding_mode = false;  // 是否为嵌入模式
};

// 定义任务结果结构体
struct task_result {
    int id;  // 任务 ID
    bool stop;  // 是否停止
    bool error;  // 是否出错
}
    # 声明一个变量result_json，用于存储JSON格式的数据
    json result_json;
// 枚举类型，表示槽位的状态
enum slot_state
{
    IDLE, // 空闲状态
    PROCESSING, // 处理中状态
};

// 枚举类型，表示槽位的命令
enum slot_command
{
    NONE, // 无命令
    LOAD_PROMPT, // 加载提示
    RELEASE, // 释放
};

// 结构体，表示槽位的参数
struct slot_params
{
    bool stream       = true; // 是否流式处理，默认为true
    bool cache_prompt = false; // 是否缓存提示以避免重新处理所有提示，默认为false

    uint32_t seed      = -1; // 随机数生成器的种子，默认为-1
    int32_t  n_keep    =  0; // 保留初始提示中的标记数，默认为0
    int32_t  n_predict = -1; // 预测的新标记数，默认为-1

    std::vector<std::string> antiprompt; // 反向提示

    json input_prefix; // 输入前缀
    json input_suffix; // 输入后缀
};

// 结构体，表示槽位的图像信息
struct slot_image
{
    int32_t id; // 图像的ID

    bool request_encode_image = false; // 请求编码图像，默认为false
    float* image_embedding = nullptr; // 图像嵌入
    int32_t image_tokens = 0; // 图像标记数

    clip_image_u8 img_data; // 图像数据

    std::string prefix_prompt; // 图像之前的提示
};

// 结构体，表示完成标记输出及其概率
struct completion_token_output
{
    // 结构体，表示标记及其概率
    struct token_prob
    {
        llama_token tok; // 标记
        float prob; // 概率
    };

    std::vector<token_prob> probs; // 标记概率列表
    llama_token tok; // 标记
    std::string text_to_send; // 发送的文本
};

// 返回两个标记向量的公共部分的大小
static size_t common_part(const std::vector<llama_token> &a, const std::vector<llama_token> &b)
{
    size_t i;
    for (i = 0; i < a.size() && i < b.size() && a[i] == b[i]; i++)
    {
    }
    return i; // 返回公共部分的大小
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
    return str.size() >= suffix.size() &&
           0 == str.compare(str.size() - suffix.size(), suffix.size(), suffix);
}

// 查找部分停止字符串的位置
static size_t find_partial_stop_string(const std::string &stop,
                                       const std::string &text)
{
    if (!text.empty() && !stop.empty()) // 如果文本和停止字符串都不为空
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
                    // 返回文本长度减去当前部分子串的长度
                    return text.size() - char_index - 1;
                }
            }
        }
    }
    // 如果没有匹配的部分子串结尾，则返回不存在的位置
    return std::string::npos;
}
// TODO: 重用 llama_detokenize 函数

// 将 tokens 转换为字符串
template <class Iter>
static std::string tokens_to_str(llama_context *ctx, Iter begin, Iter end)
{
    // 初始化返回字符串
    std::string ret;
    // 遍历 tokens，将每个 token 转换为字符串并拼接到返回字符串中
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
        {"timestamp", time(nullptr)},
        {"level",     level},
        {"function",  function},
        {"line",      line},
        {"message",   message},
    };

    // 如果额外信息不为空，则合并到日志对象中
    if (!extra.empty())
    {
        log.merge_patch(extra);
    }

    // 将日志对象转换为字符串并输出到标准输出
    const std::string str = log.dump(-1, ' ', false, json::error_handler_t::replace);
    printf("%.*s\n", (int)str.size(), str.data());
    fflush(stdout);
}

// 格式化不完整的 UTF-8 多字节字符以便输出
static std::string tokens_to_output_formatted_string(const llama_context *ctx, const llama_token token)
{
    // 将 token 转换为字符串
    std::string out = token == -1 ? "" : llama_token_to_piece(ctx, token);
    // 如果字符串长度为 1 且第一位为 1，表示为不完整字符
    if (out.size() == 1 && (out[0] & 0x80) == 0x80)
    {
        // 将字符转换为十六进制表示
        std::stringstream ss;
        ss << std::hex << (out[0] & 0xff);
        std::string res(ss.str());
        out = "byte: \\x" + res;
    }
    // 返回格式化后的字符串
    return out;
}

// 将完成标记输出的向量转换为 JSON
static json probs_vector_to_json(const llama_context *ctx, const std::vector<completion_token_output> &probs)
{
    // 初始化输出 JSON 数组
    json out = json::array();
    // 遍历完成标记输出向量，将每个元素转换为 JSON 对象并添加到输出数组中
    for (const auto &prob : probs)
    {
        // 创建一个空的 JSON 数组
        json probs_for_token = json::array();
        // 遍历概率对象中的每个概率
        for (const auto &p : prob.probs)
        {
            // 将 token 转换为格式化的字符串
            std::string tok_str = tokens_to_output_formatted_string(ctx, p.tok);
            // 将 token 字符串和概率值添加到 probs_for_token 数组中
            probs_for_token.push_back(json
            {
                {"tok_str", tok_str},
                {"prob",    p.prob},
            });
        }
        // 将 token 转换为格式化的字符串
        std::string tok_str = tokens_to_output_formatted_string(ctx, prob.tok);
        // 将 token 字符串和概率数组添加到 out 数组中
        out.push_back(json{
            {"content", tok_str},
            {"probs",   probs_for_token},
        });
    }
    // 返回结果数组
    return out;
}

// 从 JSON 对象中获取指定键的值，如果不存在则返回默认值
template <typename T>
static T json_value(const json &body, const std::string &key, const T &default_value)
{
    // 如果 JSON 对象包含指定键并且值不为空，则返回对应值，否则返回默认值
    return body.contains(key) && !body.at(key).is_null()
        ? body.value(key, default_value)
        : default_value;
}

// 定义名为 llama_client_slot 的结构体
struct llama_client_slot
{
    int id;  // 槽位 ID
    int task_id = -1;  // 任务 ID，默认值为 -1

    struct slot_params params;  // 槽位参数

    slot_state state = IDLE;  // 槽位状态，默认为 IDLE
    slot_command command = NONE;  // 槽位命令，默认为 NONE

    int64_t t_last_used = -1;  // 上次使用时间，默认为 -1

    // 生成属性
    int32_t n_ctx       = 0;  // 每个槽位的上下文大小
    int32_t n_past      = 0;
    int32_t n_decoded   = 0;
    int32_t n_remaining = -1;
    int32_t i_batch     = -1;

    int32_t num_prompt_tokens           = 0;  // 提示标记数量
    int32_t num_prompt_tokens_processed = 0;  // 已处理的提示标记数量
    int32_t multibyte_pending           = 0;  // 多字节待处理数量

    json prompt;  // 提示 JSON 对象
    std::string generated_text;  // 生成的文本
    llama_token sampled;  // 采样的标记
    std::vector<llama_token> cache_tokens;  // 缓存标记
    std::vector<completion_token_output> generated_token_probs;  // 生成的标记概率

    bool infill = false;  // 是否填充
    bool embedding = false;  // 是否嵌入
    bool has_next_token = true;  // 是否有下一个标记
    bool truncated = false;  // 是否截断
    bool stopped_eos = false;  // 是否停止 EOS
    bool stopped_word = false;  // 是否停止单词
    bool stopped_limit = false;  // 是否停止限制

    std::string stopping_word;  // 停止的单词

    // 采样
    struct llama_sampling_params sparams;  // 采样参数
    llama_sampling_context *ctx_sampling = nullptr;  // 采样上下文

    // 多模态
    std::vector<slot_image> images;  // 图像数组

    // 统计
    size_t sent_count = 0;  // 发送计数
    size_t sent_token_probs_index = 0;  // 发送标记概率索引

    int64_t t_start_process_prompt;  // 开始处理提示时间
    int64_t t_start_genereration;  // 开始生成时间

    double t_prompt_processing; // ms  // 提示处理时间
    double t_token_generation; // ms  // 标记生成时间
    // 重置所有状态和变量
    void reset() {
        num_prompt_tokens      = 0;  // 初始化提示标记数量为0
        generated_text         = "";  // 初始化生成的文本为空字符串
        truncated              = false;  // 初始化截断标志为false
        stopped_eos            = false;  // 初始化停止标志为false
        stopped_word           = false;  // 初始化停止单词标志为false
        stopped_limit          = false;  // 初始化停止限制标志为false
        stopping_word          = "";    // 初始化停止单词为空字符串
        multibyte_pending      = 0;     // 初始化多字节挂起为0
        n_past                 = 0;     // 初始化过去数量为0
        sent_count             = 0;     // 初始化句子计数为0
        sent_token_probs_index = 0;     // 初始化句子标记概率索引为0
        infill                 = false; // 初始化填充标志为false

        generated_token_probs.clear();  // 清空生成的标记概率

        for (slot_image &img : images)  // 遍历图像列表
        {
            free(img.image_embedding);  // 释放图像嵌入
            delete[] img.img_data.data; // 删除图像数据
            img.prefix_prompt = "";      // 重置图像前缀提示
        }

        images.clear();  // 清空图像列表
        // llama_set_rng_seed(ctx, params.seed); in batched the seed matter???????
    }

    // 检查是否有预算
    bool has_budget(gpt_params &global_params) {
        n_remaining = -1;  // 初始化剩余数量为-1
        if(params.n_predict != -1)  // 如果本地参数中有预测数量
        {
            n_remaining = params.n_predict - n_decoded;  // 计算剩余数量
        }
        else if (global_params.n_predict != -1)  // 如果全局参数中有预测数量
        {
            n_remaining = global_params.n_predict - n_decoded;  // 计算剩余数量
        }
        return n_remaining > 0 || n_remaining == -1; // 返回是否有预算（剩余数量大于0或者为-1表示无限制）
    }

    // 检查状态是否可用
    bool available() const {
        return state == IDLE && command == NONE;  // 返回状态是否为IDLE且命令是否为NONE
    }

    // 检查是否正在处理
    bool is_processing() const {
        return (state == IDLE && command == LOAD_PROMPT) || state == PROCESSING;  // 返回是否处于处理状态
    }

    // 添加标记字符串
    void add_token_string(const completion_token_output &token) {
        if (command == RELEASE)  // 如果命令为RELEASE
        {
            return;  // 不执行任何操作
        }
        cache_tokens.push_back(token.tok);  // 将标记添加到缓存标记列表中
        generated_token_probs.push_back(token);  // 将标记概率添加到生成的标记概率列表中
    }

    // 释放
    void release() {
        if (state == IDLE || state == PROCESSING)  // 如果状态为IDLE或者PROCESSING
        {
            t_token_generation = (ggml_time_us() - t_start_genereration) / 1e3;  // 计算标记生成时间
            command = RELEASE;  // 设置命令为RELEASE
        }
    }
    // 返回格式化后的时间数据，以 JSON 格式返回
    json get_formated_timings() {
        // 返回 JSON 对象
        return json
        {
            // 提示符处理的标记数量
            {"prompt_n",               num_prompt_tokens_processed},
            // 提示符处理的时间（毫秒）
            {"prompt_ms",              t_prompt_processing},
            // 每个提示符处理的平均时间（毫秒）
            {"prompt_per_token_ms",    t_prompt_processing / num_prompt_tokens_processed},
            // 每秒处理的提示符数量
            {"prompt_per_second",      1e3 / t_prompt_processing * num_prompt_tokens_processed},

            // 预测结果的标记数量
            {"predicted_n",            n_decoded},
            // 预测结果的时间（毫秒）
            {"predicted_ms",           t_token_generation},
            // 每个预测结果生成的平均时间（毫秒）
            {"predicted_per_token_ms", t_token_generation / n_decoded},
            // 每秒生成的预测结果数量
            {"predicted_per_second",   1e3 / t_token_generation * n_decoded},
        };
    }

    // 打印时间数据
    void print_timings() {
        // 打印换行符
        LOG_TEE("\n");
        // 打印提示符处理时间的相关信息
        LOG_TEE("%s: prompt eval time = %10.2f ms / %5d tokens (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, t_prompt_processing, num_prompt_tokens_processed, t_prompt_processing / num_prompt_tokens_processed, 1e3 / t_prompt_processing * num_prompt_tokens_processed);
        // 打印预测结果生成时间的相关信息
        LOG_TEE("%s:        eval time = %10.2f ms / %5d runs   (%8.2f ms per token, %8.2f tokens per second)\n",
            __func__, t_token_generation, n_decoded,t_token_generation / n_decoded, 1e3 / t_token_generation * n_decoded);
        // 打印总时间
        LOG_TEE("%s:       total time = %10.2f ms\n", __func__, t_prompt_processing + t_token_generation);
    }
};

// 定义 llama_server_context 结构体
struct llama_server_context
{
    // 模型和上下文的指针，默认为空指针
    llama_model *model = nullptr;
    llama_context *ctx = nullptr;

    // 剪贴板上下文的指针，默认为空指针
    clip_ctx *clp_ctx = nullptr;

    // GPT 参数
    gpt_params params;

    // 批处理对象
    llama_batch batch;

    // 是否多模态，默认为假
    bool multimodal         = false;
    // 是否清除键值缓存，默认为真
    bool clean_kv_cache     = true;
    // 所有插槽是否空闲，默认为假
    bool all_slots_are_idle = false;

    // ID 生成器
    int32_t id_gen;
    // 所有客户端/插槽的总上下文
    int32_t n_ctx;

    // 系统提示
    bool system_need_update = false;
    // 系统提示字符串
    std::string              system_prompt;
    // 系统提示的标记
    std::vector<llama_token> system_tokens;

    // 用户名称，应该是反提示
    std::string name_user;
    // 助手名称
    std::string name_assistant;

    // 插槽/客户端
    std::vector<llama_client_slot> slots;

    // 任务队列
    std::vector<task_server> queue_tasks;
    // 结果队列
    std::vector<task_result> queue_results;
    // 任务互斥锁
    std::mutex mutex_tasks;
    // 结果互斥锁
    std::mutex mutex_results;

    // 析构函数
    ~llama_server_context()
    {
        // 如果上下文存在，则释放上下文
        if (ctx)
        {
            llama_free(ctx);
            ctx = nullptr;
        }
        // 如果模型存在，则释放模型
        if (model)
        {
            llama_free_model(model);
            model = nullptr;
        }
    }

    // 加载模型
    bool load_model(const gpt_params &params_)
    {
        // 将参数赋值给成员变量
        params = params_;
        // 如果参数中的 mmproj 不为空
        if (!params.mmproj.empty()) {
            // 设置 multimodal 为 true
            multimodal = true;
            // 输出日志信息
            LOG_TEE("Multi Modal Mode Enabled");
            // 加载 clip 模型
            clp_ctx = clip_model_load(params.mmproj.c_str(), /*verbosity=*/ 1);
            // 如果加载失败
            if(clp_ctx == nullptr) {
                // 输出错误日志信息
                LOG_ERROR("unable to load clip model", {{"model", params.mmproj}});
                // 返回 false
                return false;
            }
    
            // 如果参数中的 n_ctx 小于 2048
            if (params.n_ctx < 2048) { // request larger context for the image embedding
                // 将参数中的 n_ctx 设置为 2048
                params.n_ctx = 2048;
            }
        }
    
        // 从 GPT 参数初始化模型和上下文
        std::tie(model, ctx) = llama_init_from_gpt_params(params);
        // 如果模型为空
        if (model == nullptr)
        {
            // 输出错误日志信息
            LOG_ERROR("unable to load model", {{"model", params.model}});
            // 返回 false
            return false;
        }
    
        // 如果是 multimodal 模式
        if (multimodal) {
            // 获取 clip 模型的嵌入维度
            const int n_embd_clip = clip_n_mmproj_embd(clp_ctx);
            // 获取 LLaMA 模型的嵌入维度
            const int n_embd_llm  = llama_n_embd(model);
            // 如果 clip 模型和 LLaMA 模型的嵌入维度不相等
            if (n_embd_clip != n_embd_llm) {
                // 输出日志信息
                LOG_TEE("%s: embedding dim of the multimodal projector (%d) is not equal to that of LLaMA (%d). Make sure that you use the correct mmproj file.\n", __func__, n_embd_clip, n_embd_llm);
                // 释放上下文和模型
                llama_free(ctx);
                llama_free_model(model);
                // 返回 false
                return false;
            }
        }
    
        // 获取上下文的上下文大小
        n_ctx = llama_n_ctx(ctx);
    
        // 返回 true
        return true;
    }
    // 初始化函数，用于初始化一些变量和数据结构
    void initialize() {
        // 初始化 id_gen 为 0
        id_gen = 0;
    
        // 创建插槽
        all_slots_are_idle = true;
    
        // 计算每个插槽的上下文数量
        const int32_t n_ctx_slot = n_ctx / params.n_parallel;
    
        // 输出可用插槽信息
        LOG_TEE("Available slots:\n");
        for (int i = 0; i < params.n_parallel; i++)
        {
            // 创建插槽对象
            llama_client_slot slot;
    
            // 设置插槽的 id 和上下文数量，并重置插槽
            slot.id = i;
            slot.n_ctx = n_ctx_slot;
            slot.reset();
    
            // 输出插槽信息
            LOG_TEE(" -> Slot %i - max context: %i\n", slot.id, n_ctx_slot);
            // 将插槽对象添加到插槽列表中
            slots.push_back(slot);
        }
    
        // 初始化批处理对象
        batch = llama_batch_init(n_ctx, 0, params.n_parallel);
    
        // 清空系统提示
        system_prompt = "";
        // 清空系统标记
        system_tokens.clear();
    }
    
    // 分词函数，用于将 JSON 格式的提示转换为标记序列
    std::vector<llama_token> tokenize(const json & json_prompt, bool add_bos) const
    {
        // 如果 `add_bos` 为真，则只有在 json_prompt 是字符串，或者 json_prompt 数组的第一个元素是字符串时，才添加 BOS
        std::vector<llama_token> prompt_tokens;
    
        // 如果 json_prompt 是数组
        if (json_prompt.is_array())
        {
            bool first = true;
            // 遍历 json_prompt 数组
            for (const auto& p : json_prompt)
            {
                // 如果元素是字符串
                if (p.is_string())
                {
                    // 获取字符串
                    auto s = p.template get<std::string>();
                    std::vector<llama_token> p;
                    // 如果是第一个字符串
                    if (first)
                    {
                        // 对字符串进行分词，根据 add_bos 决定是否添加 BOS
                        p = ::llama_tokenize(ctx, s, add_bos);
                        first = false;
                    }
                    else
                    {
                        // 对字符串进行分词，不添加 BOS
                        p = ::llama_tokenize(ctx, s, false);
                    }
                    // 将分词结果添加到 prompt_tokens 中
                    prompt_tokens.insert(prompt_tokens.end(), p.begin(), p.end());
                }
                else
                {
                    // 如果不是字符串，且是第一个元素
                    if (first)
                    {
                        first = false;
                    }
                    // 将非字符串元素添加到 prompt_tokens 中
                    prompt_tokens.push_back(p.template get<llama_token>());
                }
            }
        }
        else
        {
            // 如果 json_prompt 不是数组，将其视为字符串
            auto s = json_prompt.template get<std::string>();
            // 对字符串进行分词，根据 add_bos 决定是否添加 BOS
            prompt_tokens = ::llama_tokenize(ctx, s, add_bos);
        }
    
        // 返回分词结果
        return prompt_tokens;
    }
    
    // 根据 id 获取槽位
    llama_client_slot* get_slot(int id) {
        int64_t t_last = ggml_time_us();
        llama_client_slot *last_used = nullptr;
    
        // 遍历槽位数组
        for (llama_client_slot & slot : slots)
        {
            // 如果槽位的 id 匹配，并且可用
            if (slot.id == id && slot.available())
            {
                // 返回该槽位
                return &slot;
            }
    
            // 如果槽位可用，并且上次使用时间比 t_last 小
            if (slot.available() && slot.t_last_used < t_last)
            {
                // 更新最后使用的槽位和时间
                last_used = &slot;
                t_last = slot.t_last_used;
            }
        }
    
        // 返回最后使用的槽位
        return last_used;
    }
    }
    // 清空整个 KV 缓存
    void kv_cache_clear() {
        // 调用 llama_kv_cache_clear 函数清空 KV 缓存
        llama_kv_cache_clear(ctx);
        // 将 clean_kv_cache 标记设置为 false
        clean_kv_cache = false;
    }

    // 更新系统提示信息
    void update_system_prompt() {
        // 使用 llama_tokenize 函数对系统提示信息进行分词，并存储到 system_tokens 中
        system_tokens = ::llama_tokenize(ctx, system_prompt, true);

        // 清空批处理对象
        llama_batch_clear(batch);

        // 清空 KV 缓存
        kv_cache_clear();

        // 遍历 system_tokens，将每个 token 添加到批处理对象中
        for (int i = 0; i < (int) system_tokens.size(); ++i)
        {
            llama_batch_add(batch, system_tokens[i], i, { 0 }, false);
        }

        // 如果 llama_decode 函数返回非 0 值，记录日志并返回
        if (llama_decode(ctx, batch) != 0)
        {
            LOG_TEE("%s: llama_decode() failed\n", __func__);
            return;
        }

        // 将系统 KV 缓存分配给所有并行序列
        for (int32_t i = 1; i < params.n_parallel; ++i)
        {
            llama_kv_cache_seq_cp(ctx, 0, i, 0, system_tokens.size());
        }

        // 记录日志，标记系统提示信息无需更新
        LOG_TEE("system prompt updated\n");
        system_need_update = false;
    }

    // 通知系统提示信息已更改
    void notify_system_prompt_changed() {
        // 释放所有槽位
        for (llama_client_slot &slot : slots)
        {
            slot.release();
        }

        // 标记系统提示信息需要更新
        system_need_update = true;
    }

    // 处理系统提示信息数据
    void process_system_prompt_data(const json &sys_props) {
        // 从 sys_props 中获取系统提示信息、用户名称和助手名称
        system_prompt  = sys_props.value("prompt", "");
        name_user      = sys_props.value("anti_prompt", "");
        name_assistant = sys_props.value("assistant_name", "");

        // 如果槽位数量大于 0，则通知系统提示信息已更改
        if (slots.size() > 0)
        {
            notify_system_prompt_changed();
        }
    }

    // 查找停止字符串
    static size_t find_stopping_strings(const std::string &text, const size_t last_token_size,
                                        const stop_type type, llama_client_slot &slot)
    {
        // 初始化停止位置为字符串的末尾
        size_t stop_pos = std::string::npos;

        // 遍历参数中的反提示词
        for (const std::string &word : slot.params.antiprompt)
        {
            size_t pos;
            // 如果类型为STOP_FULL
            if (type == STOP_FULL)
            {
                // 计算临时值，包括单词大小和上一个标记的大小
                const size_t tmp = word.size() + last_token_size;
                // 计算起始位置，如果文本大小大于临时值，则起始位置为文本大小减去临时值，否则为0
                const size_t from_pos = text.size() > tmp ? text.size() - tmp : 0;
                // 在文本中查找单词，从起始位置开始
                pos = text.find(word, from_pos);
            }
            else
            {
                // 调用函数在文本中查找部分停止字符串
                pos = find_partial_stop_string(word, text);
            }
            // 如果找到位置并且停止位置为字符串的末尾，或者找到位置小于停止位置
            if (pos != std::string::npos &&
                (stop_pos == std::string::npos || pos < stop_pos))
            {
                // 如果类型为STOP_FULL
                if (type == STOP_FULL)
                {
                    // 设置槽的停止单词标志为true
                    slot.stopped_word = true;
                    // 设置槽的停止单词为当前单词
                    slot.stopping_word = word;
                    // 设置槽的下一个标记标志为false
                    slot.has_next_token = false;
                }
                // 更新停止位置
                stop_pos = pos;
            }
        }

        // 返回停止位置
        return stop_pos;
    }

    }

    // 处理图片的函数
    bool process_images(llama_client_slot &slot) const
    {
        // 遍历槽位中的图片对象
        for (slot_image &img : slot.images)
        {
            // 如果图片对象不需要编码，则跳过
            if (!img.request_encode_image)
            {
                continue;
            }
            // 对图片进行预处理，将结果存储在img_res中
            clip_image_f32 img_res;
            if (!clip_image_preprocess(clp_ctx, &img.img_data, &img_res, /*pad2square =*/ true))
            {
                // 如果预处理出错，则记录错误信息并释放资源后返回false
                LOG_TEE("Error processing the given image");
                clip_free(clp_ctx);
                return false;
            }
            // 计算图片的分块数目
            img.image_tokens = clip_n_patches(clp_ctx);
            // 为图片嵌入向量分配内存空间
            img.image_embedding = (float *)malloc(clip_embd_nbytes(clp_ctx));
            // 如果内存分配失败，则记录错误信息并释放资源后返回false
            if (!img.image_embedding)
            {
                LOG_TEE("Unable to allocate memory for image embeddings\n");
                clip_free(clp_ctx);
                return false;
            }
            // 记录编码图片的信息
            LOG_TEE("slot %i - encoding image [id: %i]\n", slot.id, img.id);
            // 对图片进行编码
            if (!clip_image_encode(clp_ctx, params.n_threads, &img_res, img.image_embedding))
            {
                // 如果编码失败，则记录错误信息并返回false
                LOG_TEE("Unable to encode image\n");
                return false;
            }
            // 将图片的编码请求标记为false
            img.request_encode_image = false;
        }

        // 返回槽位中是否包含图片的信息
        return slot.images.size() > 0;
    }

    // 发送错误信息
    void send_error(int id, std::string error)
    {
        // 使用互斥锁保护共享资源
        std::lock_guard<std::mutex> lock(mutex_results);
        // 创建任务结果对象，并设置id和error属性
        task_result res;
        res.id = id;
        res.error = true;
        // 设置结果JSON对象的内容属性为错误信息
        res.result_json = { { "content", error } };
        // 将结果对象加入结果队列
        queue_results.push_back(res);
    }

    // 获取模型属性的JSON对象
    json get_model_props()
    {
        // 返回格式化后的槽位生成信息
        return get_formated_generation(slots[0]);
    }

    // 获取格式化后的生成信息
    json get_formated_generation(llama_client_slot &slot)
    {
        // 计算结束标记的偏置值
        const auto eos_bias = slot.sparams.logit_bias.find(llama_token_eos(model));
        // 检查是否忽略结束标记
        const bool ignore_eos = eos_bias != slot.sparams.logit_bias.end() &&
                                eos_bias->second < 0.0f && std::isinf(eos_bias->second);
        // 返回包含各种参数的 JSON 对象
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
            {"repeat_penalty",    slot.sparams.penalty_repeat},
            {"presence_penalty",  slot.sparams.penalty_present},
            {"frequency_penalty", slot.sparams.penalty_freq},
            {"mirostat",          slot.sparams.mirostat},
            {"mirostat_tau",      slot.sparams.mirostat_tau},
            {"mirostat_eta",      slot.sparams.mirostat_eta},
            {"penalize_nl",       slot.sparams.penalize_nl},
            {"stop",              slot.params.antiprompt},
            {"n_predict",         slot.params.n_predict},
            {"n_keep",            params.n_keep},
            {"ignore_eos",        ignore_eos},
            {"stream",            slot.params.stream},
            {"logit_bias",        slot.sparams.logit_bias},
            {"n_probs",           slot.sparams.n_probs},
            {"grammar",           slot.sparams.grammar},
        };
    }
    
    // 发送部分响应
    void send_partial_response(llama_client_slot &slot, completion_token_output tkn)
    {
        // 使用互斥锁保护共享资源
        std::lock_guard<std::mutex> lock(mutex_results);
        // 创建任务结果对象
        task_result res;
        // 设置任务结果对象的ID、错误标志和停止标志
        res.id = slot.task_id;
        res.error = false;
        res.stop = false;

        // 设置任务结果对象的JSON内容
        res.result_json = json
        {
            {"content",    tkn.text_to_send},
            {"stop",       false},
            {"slot_id",    slot.id},
            {"multimodal", multimodal}
        };

        // 如果槽参数中的概率大于0
        if (slot.sparams.n_probs > 0)
        {
            // 创建概率输出向量
            std::vector<completion_token_output> probs_output = {};
            // 获取要发送的令牌，并进行分词
            const std::vector<llama_token> to_send_toks = llama_tokenize(ctx, tkn.text_to_send, false);
            // 计算概率的起始位置和结束位置
            size_t probs_pos = std::min(slot.sent_token_probs_index, slot.generated_token_probs.size());
            size_t probs_stop_pos = std::min(slot.sent_token_probs_index + to_send_toks.size(), slot.generated_token_probs.size());
            // 如果起始位置小于结束位置
            if (probs_pos < probs_stop_pos)
            {
                // 从生成的令牌概率中获取对应位置的输出
                probs_output = std::vector<completion_token_output>(slot.generated_token_probs.begin() + probs_pos, slot.generated_token_probs.begin() + probs_stop_pos);
            }
            // 更新已发送的令牌概率的位置
            slot.sent_token_probs_index = probs_stop_pos;
            // 将概率输出向量转换为JSON格式，并添加到任务结果对象的JSON内容中
            res.result_json["completion_probabilities"] = probs_vector_to_json(ctx, probs_output);
        }

        // 将任务结果对象添加到结果队列中
        queue_results.push_back(res);
    }

    // 发送最终响应的函数
    void send_final_response(llama_client_slot &slot)
    {
        // 使用互斥锁保护共享资源
        std::lock_guard<std::mutex> lock(mutex_results);
        // 创建任务结果对象
        task_result res;
        // 设置任务结果对象的属性
        res.id = slot.task_id;
        res.error = false;
        res.stop = true;
    
        // 设置任务结果对象的 JSON 属性
        res.result_json = json
        {
            {"content",             !slot.params.stream ? slot.generated_text : ""},
            {"slot_id",             slot.id},
            {"stop",                true},
            {"model",               params.model_alias},
            {"tokens_predicted",    slot.n_decoded},
            {"tokens_evaluated",    slot.num_prompt_tokens},
            {"generation_settings", get_formated_generation(slot)},
            {"prompt",              slot.prompt},
            {"truncated",           slot.truncated},
            {"stopped_eos",         slot.stopped_eos},
            {"stopped_word",        slot.stopped_word},
            {"stopped_limit",       slot.stopped_limit},
            {"stopping_word",       slot.stopping_word},
            {"tokens_cached",       slot.n_past},
            {"timings",             slot.get_formated_timings()}
        };
    
        // 如果任务有概率信息
        if (slot.sparams.n_probs > 0)
        {
            // 创建概率信息对象
            std::vector<completion_token_output> probs = {};
            // 如果不是流式处理且任务有停止词
            if (!slot.params.stream && slot.stopped_word)
            {
                // 对停止词进行分词处理
                const std::vector<llama_token> stop_word_toks = llama_tokenize(ctx, slot.stopping_word, false);
                // 从生成的概率信息中获取除停止词外的部分
                probs = std::vector<completion_token_output>(slot.generated_token_probs.begin(), slot.generated_token_probs.end() - stop_word_toks.size());
            }
            else
            {
                // 从生成的概率信息中获取指定范围内的部分
                probs = std::vector<completion_token_output>(
                                    slot.generated_token_probs.begin(),
                                    slot.generated_token_probs.begin() + slot.sent_token_probs_index);
            }
            // 将概率信息添加到任务结果对象的 JSON 属性中
            res.result_json["completion_probabilities"] = probs_vector_to_json(ctx, probs);
        }
    
        // 将任务结果对象添加到结果队列中
        queue_results.push_back(res);
    }
    // 发送嵌入向量到客户端的函数
    void send_embedding(llama_client_slot &slot)
    {
        // 加锁，确保结果队列的线程安全
        std::lock_guard<std::mutex> lock(mutex_results);
        // 创建任务结果对象
        task_result res;
        // 设置任务结果对象的ID和错误标志
        res.id = slot.task_id;
        res.error = false;
        res.stop = true;

        // 获取嵌入向量的维度
        const int n_embd = llama_n_embd(model);
        // 如果嵌入向量被禁用
        if (!params.embedding)
        {
            // 记录警告日志
            LOG_WARNING("embedding disabled", {
                                                  {"params.embedding", params.embedding},
                                              });
            // 设置结果对象的JSON数据，嵌入向量为全零向量
            res.result_json = json
            {
                {"embedding", std::vector<float>(n_embd, 0.0f)},
            };
        }
        else
        {
            // 获取嵌入向量数据
            const float *data = llama_get_embeddings(ctx);
            // 将嵌入向量数据转换为向量
            std::vector<float> embedding(data, data + n_embd);
            // 设置结果对象的JSON数据，嵌入向量为获取的数据
            res.result_json = json
            {
                {"embedding", embedding },
            };
        }
        // 将结果对象加入结果队列
        queue_results.push_back(res);
    }

    // 请求完成的函数，创建任务并加入任务队列
    int request_completion(json data, bool infill, bool embedding)
    {
        // 加锁，确保任务队列的线程安全
        std::lock_guard<std::mutex> lock(mutex_tasks);
        // 创建任务对象
        task_server task;
        // 设置任务对象的ID、数据、填充模式和嵌入模式
        task.id = id_gen++;
        task.data = data;
        task.infill_mode = infill;
        task.embedding_mode = embedding;
        task.type = COMPLETION_TASK;
        // 将任务对象加入任务队列
        queue_tasks.push_back(task);
        // 返回任务ID
        return task.id;
    }

    // 获取下一个任务的结果
    task_result next_result(int task_id)
    {
        // 无限循环，等待5微秒
        while (true)
        {
            std::this_thread::sleep_for(std::chrono::microseconds(5));
            // 使用互斥锁锁定结果队列
            std::lock_guard<std::mutex> lock(mutex_results);

            // 如果结果队列为空，则继续循环
            if (queue_results.empty())
            {
                continue;
            }

            // 遍历结果队列，查找与任务ID匹配的结果
            for (int i = 0; i < (int) queue_results.size(); i++)
            {
                if (queue_results[i].id == task_id)
                {
                    // 将匹配的结果保存下来，并从队列中移除
                    task_result res = queue_results[i];
                    queue_results.erase(queue_results.begin() + i);
                    return res;
                }
            }
        }

        // 永远不会执行到这里
        //return task_result{-1, false, false, {}};
    }

    // 用于多图像处理
    bool ingest_images(llama_client_slot &slot, int n_batch)
    }

    // 请求取消任务
    void request_cancel(int task_id)
    {
        // 使用互斥锁锁定任务队列
        std::lock_guard<std::mutex> lock(mutex_tasks);
        // 创建一个取消任务，并加入任务队列
        task_server task;
        task.id = id_gen++;
        task.type = CANCEL_TASK;
        task.target_id = task_id;
        queue_tasks.push_back(task);
    }

    void process_tasks()
    {
        // 使用互斥锁保护临界区，确保在多线程环境下任务队列操作的原子性
        std::lock_guard<std::mutex> lock(mutex_tasks);
        // 循环处理任务队列中的任务
        while (!queue_tasks.empty())
        {
            // 获取队首任务
            task_server task = queue_tasks.front();
            // 移除队首任务
            queue_tasks.erase(queue_tasks.begin());
            // 根据任务类型进行不同的处理
            switch (task.type)
            {
                case COMPLETION_TASK: {
                    // 根据任务数据中的槽位 ID 获取槽位对象
                    llama_client_slot *slot = get_slot(json_value(task.data, "slot_id", -1));
                    // 如果槽位对象为空，记录日志并发送错误结果，然后结束当前任务处理
                    if (slot == nullptr)
                    {
                        LOG_TEE("slot unavailable\n");
                        // 发送错误结果
                        send_error(task.id, "slot unavailable");
                        return;
                    }

                    // 如果任务数据中包含系统提示信息，处理系统提示数据
                    if (task.data.contains("system_prompt"))
                    {
                        process_system_prompt_data(task.data["system_prompt"]);
                    }

                    // 重置槽位对象状态
                    slot->reset();

                    // 更新槽位对象的填充模式、嵌入模式和任务 ID
                    slot->infill = task.infill_mode;
                    slot->embedding = task.embedding_mode;
                    slot->task_id = task.id;

                    // 根据任务数据启动槽位对象，如果失败则发送错误结果
                    if (!launch_slot_with_data(slot, task.data))
                    {
                        // 发送错误结果
                        send_error(task.id, "internal_error");
                        break;
                    }
                } break;
                case CANCEL_TASK: { // 释放与任务 ID 关联的槽位
                    for (auto & slot : slots)
                    {
                        if (slot.task_id == task.target_id)
                        {
                            slot.release();
                            break;
                        }
                    }
                } break;
            }
        }
    }
};
// 打印服务器的使用说明
static void server_print_usage(const char *argv0, const gpt_params &params,
                               const server_params &sparams)
{
    // 打印程序的使用方法
    printf("usage: %s [options]\n", argv0);
    printf("\n");
    printf("options:\n");
    // 打印帮助选项
    printf("  -h, --help                show this help message and exit\n");
    // 打印详细输出选项
    printf("  -v, --verbose             verbose output (default: %s)\n", server_verbose ? "enabled" : "disabled");
    // 打印线程数选项
    printf("  -t N, --threads N         number of threads to use during computation (default: %d)\n", params.n_threads);
    // 打印批处理和提示处理时使用的线程数选项
    printf("  -tb N, --threads-batch N  number of threads to use during batch and prompt processing (default: same as --threads)\n");
    // 打印提示上下文的大小选项
    printf("  -c N, --ctx-size N        size of the prompt context (default: %d)\n", params.n_ctx);
    // 打印 RoPE 频率缩放方法选项
    printf("  --rope-scaling {none,linear,yarn}\n");
    // 打印 RoPE 基础频率选项
    printf("  --rope-freq-base N        RoPE base frequency (default: loaded from model)\n");
    // 打印 RoPE 频率缩放因子选项
    printf("  --rope-freq-scale N       RoPE frequency scaling factor, expands context by a factor of 1/N\n");
    // 打印 YaRN: 外推混合因子选项
    printf("  --yarn-ext-factor N       YaRN: extrapolation mix factor (default: 1.0, 0.0 = full interpolation)\n");
    // 打印 YaRN: 缩放 sqrt(t) 或注意力幅度选项
    printf("  --yarn-attn-factor N      YaRN: scale sqrt(t) or attention magnitude (default: 1.0)\n");
    // 打印 YaRN: 高校正维度或 alpha 选项
    printf("  --yarn-beta-slow N        YaRN: high correction dim or alpha (default: %.1f)\n", params.yarn_beta_slow);
    // 打印 YaRN: 低校正维度或 beta 选项
    printf("  --yarn-beta-fast N        YaRN: low correction dim or beta (default: %.1f)\n", params.yarn_beta_fast);
    // 打印批处理大小选项
    printf("  -b N, --batch-size N      batch size for prompt processing (default: %d)\n", params.n_batch);
    // 打印使用 f32 而不是 f16 用于内存键值的选项
    printf("  --memory-f32              use f32 instead of f16 for memory key+value (default: disabled)\n");
    // 打印不推荐使用 f32 内存的警告
    printf("                            not recommended: doubles context memory required and no measurable increase in quality\n");
    // 检查是否支持内存锁定
    if (llama_mlock_supported())
    {
        # 打印提示信息，强制系统将模型保留在内存中，而不是交换或压缩
        printf("  --mlock               force system to keep model in RAM rather than swapping or compressing\n");
    }
    # 如果系统支持内存映射
    if (llama_mmap_supported())
    {
        # 打印提示信息，不使用内存映射模型（加载速度较慢，但可能减少页面输出，如果不使用mlock）
        printf("  --no-mmap             do not memory-map model (slower load but may reduce pageouts if not using mlock)\n");
    }
    # 打印提示信息，尝试一些在某些NUMA系统上有帮助的优化
    printf("  --numa                attempt optimizations that help on some NUMA systems\n");
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
    // 打印GPU支持的选项
    printf("  -ngl N, --n-gpu-layers N\n");
    // 打印存储在VRAM中的层数
    printf("                        number of layers to store in VRAM\n");
    // 打印如何在多个GPU上分割张量的选项，以逗号分隔的比例列表，例如3,1
    printf("  -ts SPLIT --tensor-split SPLIT\n");
    // 打印要在哪个GPU上使用作为临时和小张量的GPU
    printf("                        how to split tensors across multiple GPUs, comma-separated list of proportions, e.g. 3,1\n");
    // 打印要使用的GPU
    printf("  -mg i, --main-gpu i   the GPU to use for scratch and small tensors\n");
    // 打印是否使用cuBLAS而不是自定义的mul_mat_q CUDA内核
    printf("  -nommq, --no-mul-mat-q\n");
    // 打印是否启用LoRA适配器
    printf("                        use cuBLAS instead of custom mul_mat_q CUDA kernels.\n");
    // 打印是否启用LoRA适配器
    printf("  -m FNAME, --model FNAME\n");
    // 打印模型路径
    printf("                        model path (default: %s)\n", params.model.c_str());
    // 打印设置模型的别名
    printf("  -a ALIAS, --alias ALIAS\n");
    // 打印设置模型的别名
    printf("                        set an alias for the model, will be added as `model` field in completion response\n");
    // 打印应用LoRA适配器
    printf("  --lora FNAME          apply LoRA adapter (implies --no-mmap)\n");
    // 打印可选的用作LoRA适配器修改层的基础模型
    printf("  --lora-base FNAME     optional model to use as a base for the layers modified by the LoRA adapter\n");
    // 打印要监听的IP地址
    printf("  --host                ip address to listen (default  (default: %s)\n", sparams.hostname.c_str());
    // 打印要监听的端口号
    printf("  --port PORT           port to listen (default  (default: %d)\n", sparams.port);
    // 打印要提供静态文件的路径
    printf("  --path PUBLIC_PATH    path from which to serve static files (default %s)\n", sparams.public_path.c_str());
    // 打印服务器读/写超时时间
    printf("  -to N, --timeout N    server read/write timeout in seconds (default: %d)\n", sparams.read_timeout);
    // 打印是否启用嵌入向量输出
    printf("  --embedding           enable embedding vector output (default: %s)\n", params.embedding ? "enabled" : "disabled");
    // 打印处理请求的并行槽的数量
    printf("  -np N, --parallel N   number of slots for process requests (default: %d)\n", params.n_parallel);
    // 打印是否启用连续批处理
    printf("  -cb, --cont-batching  enable continuous batching (a.k.a dynamic batching) (default: disabled)\n");
#endif
    # 打印系统提示文件的选项和说明
    printf("    -spf FNAME, --system-prompt-file FNAME\n");
    # 打印设置系统提示文件的说明
    printf("                        Set a file to load a system prompt (initial prompt of all slots), this is useful for chat applications.\n");
    # 打印VRAM预算选项和说明
    printf("  --vram-budget N       VRAM budget in GiB (default: -1, -1 = available VRAM)\n");
    # 打印LLaVA的多模态投影文件路径选项和说明
    printf("  --mmproj MMPROJ_FILE  path to a multimodal projector file for LLaVA.\n");
    # 打印空行
    printf("\n");
    // 解析服务器参数和 GPT 参数
static void server_params_parse(int argc, char **argv, server_params &sparams,
                                gpt_params &params, llama_server_context& llama)
{
    // 默认的 GPT 参数
    gpt_params default_params;
    // 默认的服务器参数
    server_params default_sparams;
    // 命令行参数
    std::string arg;
    // 无效参数标志
    bool invalid_param = false;

    // 遍历命令行参数
    for (int i = 1; i < argc; i++)
#ifdef LLAMA_SUPPORTS_GPU_OFFLOAD
            // 如果支持 GPU 卸载，则解析参数为 GPU 层数
            params.n_gpu_layers = std::stoi(argv[i]);
#else
            // 如果不支持 GPU 卸载，则输出警告信息
            LOG_WARNING("Not compiled with GPU offload support, --n-gpu-layers option will be ignored. "
                        "See main README.md for information on enabling GPU BLAS support",
                        {{"n_gpu_layers", params.n_gpu_layers}});
#endif
        }
        else if (arg == "--tensor-split" || arg == "-ts")
        {
            // 如果参数为 tensor-split 或 -ts，则解析参数为 tensor 分割
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
#ifdef GGML_USE_CUBLAS
            // 获取下一个参数
            std::string arg_next = argv[i];

            // 通过 , 和 / 分割字符串
            const std::regex regex{R"([,/]+)"};
            std::sregex_token_iterator it{arg_next.begin(), arg_next.end(), regex, -1};
            std::vector<std::string> split_arg{it, {}};
            GGML_ASSERT(split_arg.size() <= LLAMA_MAX_DEVICES);

            // 遍历设备，设置 tensor 分割参数
            for (size_t i_device = 0; i_device < LLAMA_MAX_DEVICES; ++i_device)
            {
                if (i_device < split_arg.size())
                {
                    params.tensor_split[i_device] = std::stof(split_arg[i_device]);
                }
                else
                {
                    params.tensor_split[i_device] = 0.0f;
                }
            }
#else
            // 如果没有使用 cuBLAS，则输出警告信息
            LOG_WARNING("llama.cpp was compiled without cuBLAS. It is not possible to set a tensor split.\n", {});
#endif // GGML_USE_CUBLAS
        }
        else if (arg == "--no-mul-mat-q" || arg == "-nommq")
        {
#ifdef GGML_USE_CUBLAS
            // 如果使用 cuBLAS，则设置 mul_mat_q 参数为 false
            params.mul_mat_q = false;
        // 如果编译时未使用 cuBLAS，则记录警告信息
        else
            LOG_WARNING("warning: llama.cpp was compiled without cuBLAS. Disabling mul_mat_q kernels has no effect.\n", {});
#endif // GGML_USE_CUBLAS
        }
        // 如果参数为"--main-gpu"或"-mg"
        else if (arg == "--main-gpu" || arg == "-mg")
        {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 如果编译时使用 cuBLAS，则将参数转换为整数并赋值给params.main_gpu
#ifdef GGML_USE_CUBLAS
            params.main_gpu = std::stoi(argv[i]);
#else
            // 如果未使用 cuBLAS，则记录警告信息
            LOG_WARNING("llama.cpp was compiled without cuBLAS. It is not possible to set a main GPU.", {});
#endif
        }
        // 如果参数为"--lora"
        else if (arg == "--lora")
        {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将参数转换为元组并添加到params.lora_adapter中，同时设置params.use_mmap为false
            params.lora_adapter.push_back(std::make_tuple(argv[i], 1.0f));
            params.use_mmap = false;
        }
        // 如果参数为"--lora-scaled"
        else if (arg == "--lora-scaled")
        {
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将参数转换为字符指针
            const char * lora_adapter = argv[i];
            // 如果参数不足，设置参数无效并跳出循环
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 将参数转换为元组并添加到params.lora_adapter中，同时设置params.use_mmap为false
            params.lora_adapter.push_back(std::make_tuple(lora_adapter, std::stof(argv[i]));
            params.use_mmap = false;
        }
        // 如果参数为"--lora-base"
        else if (arg == "--lora-base")
        {
            // 如果参数不足，设置参数无效并跳出循环
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
            // 如果编译时未设置SERVER_VERBOSE为1，则记录警告信息
#if SERVER_VERBOSE != 1
            LOG_WARNING("server.cpp is not built with verbose logging.", {});
#else
            // 否则设置server_verbose为true
            server_verbose = true;
#else
        }
        // 如果参数为 "--mlock"，则设置参数 use_mlock 为 true
        else if (arg == "--mlock")
        {
            params.use_mlock = true;
        }
        // 如果参数为 "--no-mmap"，则设置参数 use_mmap 为 false
        else if (arg == "--no-mmap")
        {
            params.use_mmap = false;
        }
        // 如果参数为 "--numa"，则设置参数 numa 为 true
        else if (arg == "--numa")
        {
            params.numa = true;
        }
        // 如果参数为 "--embedding"，则设置参数 embedding 为 true
        else if (arg == "--embedding")
        {
            params.embedding = true;
        }
        // 如果参数为 "-cb" 或 "--cont-batching"，则设置参数 cont_batching 为 true
        else if (arg == "-cb" || arg == "--cont-batching")
        {
            params.cont_batching = true;
        }
        // 如果参数为 "-np" 或 "--parallel"，则获取下一个参数作为并行数，存入参数 n_parallel
        else if (arg == "-np" || arg == "--parallel")
        {
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            params.n_parallel = std::stoi(argv[i]);
        } 
        // 如果参数为 "-n" 或 "--n-predict"，则获取下一个参数作为预测数，存入参数 n_predict
        else if (arg == "-n" || arg == "--n-predict")
        {
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            params.n_predict = std::stoi(argv[i]);
        } 
        // 如果参数为 "-spf" 或 "--system-prompt-file"，则读取文件内容并处理
        else if (arg == "-spf" || arg == "--system-prompt-file")
        {
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 打开文件并读取内容
            std::ifstream file(argv[i]);
            if (!file) {
                fprintf(stderr, "error: failed to open file '%s'\n", argv[i]);
                invalid_param = true;
                break;
            }
            std::string systm_content;
            // 将文件内容存入 systm_content
            std::copy(
                std::istreambuf_iterator<char>(file),
                std::istreambuf_iterator<char>(),
                std::back_inserter(systm_content)
            );
            // 处理系统提示数据
            llama.process_system_prompt_data(json::parse(systm_content));
        }
        // 如果参数为 "--vram-budget"，则获取下一个参数作为 VRAM 预算
        else if (arg == "--vram-budget")
        {
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            // 设置 VRAM 预算
#ifdef GGML_USE_CUBLAS
            params.vram_budget_gb = std::stof(argv[i]);
        // 如果编译时没有包含 cuBLAS，则输出警告信息
        #else
            fprintf(stderr, "warning: PowerInfer was compiled without cuBLAS. It is not possible to set a VRAM budget.\n");
        #endif
        }
        // 如果参数为 "--reset-gpu-index"，则将参数 reset_gpu_index 设置为 true
        else if (arg == "--reset-gpu-index")
        {
            params.reset_gpu_index = true;
        } 
        // 如果参数为 "--disable-gpu-index"，则将参数 disable_gpu_index 设置为 true
        else if (arg == "--disable-gpu-index")
        {
            params.disable_gpu_index = true;
        }
        // 如果参数为 "--mmproj"，则检查下一个参数是否存在，若存在则将参数 mmproj 设置为该值
        else if(arg == "--mmproj")
        {
            if (++i >= argc)
            {
                invalid_param = true;
                break;
            }
            params.mmproj = argv[i];
        }
        // 如果参数不匹配任何已知参数，则输出错误信息并退出程序
        else
        {
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            server_print_usage(argv[0], default_params, default_sparams);
            exit(1);
        }
    }

    // 如果存在无效参数，则输出错误信息并退出程序
    if (invalid_param)
    {
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        server_print_usage(argv[0], default_params, default_sparams);
        exit(1);
    }
}

// 格式化部分响应的 JSON 数据
static json format_partial_response(
    llama_server_context &llama, llama_client_slot *slot, const std::string &content, const std::vector<completion_token_output> &probs
) {
    // 创建 JSON 对象并设置相应的字段值
    json res = json
    {
        {"content",    content },
        {"stop",       false},
        {"slot_id",    slot->id },
        {"multimodal", llama.multimodal }
    };

    // 如果 slot 的参数中存在概率值，则将概率值转换为 JSON 格式并添加到 res 中
    if (slot->sparams.n_probs > 0)
    {
        res["completion_probabilities"] = probs_vector_to_json(llama.ctx, probs);
    }

    return res;
}

// 格式化分词器响应的 JSON 数据
static json format_tokenizer_response(const std::vector<llama_token> &tokens)
{
    // 创建 JSON 对象并设置 tokens 字段值
    return json{
        {"tokens", tokens}};
}

// 格式化去标记化响应的 JSON 数据
static json format_detokenized_response(std::string content)
{
    // 创建 JSON 对象并设置 content 字段值
    return json{
        {"content", content}};
}


// 记录服务器请求的日志信息
static void log_server_request(const httplib::Request &req, const httplib::Response &res)
{
    # 记录请求信息，包括远程地址、远程端口、状态、请求方法、路径和参数
    LOG_INFO("request", {
                            {"remote_addr", req.remote_addr},
                            {"remote_port", req.remote_port},
                            {"status", res.status},
                            {"method", req.method},
                            {"path", req.path},
                            {"params", req.params},
                        });

    # 记录详细的请求和响应信息，包括请求体和响应体
    LOG_VERBOSE("request", {
                               {"request", req.body},
                               {"response", res.body},
                           });
// 结构体，用于将 token 转换为字符串
struct token_translator
{
    llama_context * ctx; // llama 上下文指针
    std::string operator()(llama_token tok)                    const { return llama_token_to_piece(ctx, tok); } // 将 llama_token 转换为字符串
    std::string operator()(const completion_token_output &cto) const { return (*this)(cto.tok); } // 将 completion_token_output 转换为字符串
};

// 从生成的 token 概率中追加到生成的文本中
static void append_to_generated_text_from_generated_token_probs(llama_server_context &llama, llama_client_slot *slot)
{
    auto & gtps = slot->generated_token_probs; // 获取生成的 token 概率
    auto translator = token_translator{llama.ctx}; // 创建 token 转换器
    auto add_strlen = [=](size_t sum, const completion_token_output & cto) { return sum + translator(cto).size(); }; // 计算字符串长度的 lambda 函数
    const size_t len = std::accumulate(gtps.begin(), gtps.end(), size_t(0), add_strlen); // 计算生成文本的总长度
    if (slot->generated_text.capacity() < slot->generated_text.size() + len) // 如果生成文本的容量不足
    {
        slot->generated_text.reserve(slot->generated_text.size() + len); // 扩展生成文本的容量
    }
    for (const completion_token_output & cto : gtps) // 遍历生成的 token 概率
    {
        slot->generated_text += translator(cto); // 将 token 转换为字符串并追加到生成文本中
    }
}

// 主函数
int main(int argc, char **argv)
{
    // 为此示例所需的自有参数
    gpt_params params; // gpt 参数
    server_params sparams; // 服务器参数

    // 包含 llama 上下文和推理的结构体
    llama_server_context llama; // llama 服务器上下文

    server_params_parse(argc, argv, sparams, params, llama); // 解析服务器参数

    if (params.model_alias == "unknown") // 如果模型别名为 "unknown"
    {
        params.model_alias = params.model; // 将模型别名设置为模型名称
    }

    llama_backend_init(params.numa); // 初始化 llama 后端

    LOG_INFO("build info", {{"build", LLAMA_BUILD_NUMBER}, // 记录构建信息
                            {"commit", LLAMA_COMMIT}}); // 记录提交信息

    LOG_INFO("system info", {
                                {"n_threads", params.n_threads}, // 记录线程数
                                {"n_threads_batch", params.n_threads_batch}, // 记录批处理线程数
                                {"total_threads", std::thread::hardware_concurrency()}, // 记录总线程数
                                {"system_info", llama_print_system_info()}, // 记录系统信息
                            });

    // 加载模型
    if (!llama.load_model(params)) // 如果加载模型失败
    {
        return 1; // 返回错误码
    }

    llama.initialize(); // 初始化 llama
}
    # 创建一个名为svr的HTTP服务器对象
    httplib::Server svr;
    
    # 设置默认的HTTP响应头
    svr.set_default_headers({{"Server", "llama.cpp"},
                             {"Access-Control-Allow-Origin", "*"},
                             {"Access-Control-Allow-Headers", "content-type"}});
    
    # 当在public路径中找不到index.html时调用，返回index.html内容
    svr.Get("/", [](const httplib::Request &, httplib::Response &res)
            {
                res.set_content(reinterpret_cast<const char*>(&index_html), index_html_len, "text/html");
                return false;
            });
    
    # 当在public路径中找不到index.js时调用，返回index.js内容
    svr.Get("/index.js", [](const httplib::Request &, httplib::Response &res)
            {
                res.set_content(reinterpret_cast<const char *>(&index_js), index_js_len, "text/javascript");
                return false;
            });
    
    # 当在public路径中找不到completion.js时调用，返回completion.js内容
    svr.Get("/completion.js", [](const httplib::Request &, httplib::Response &res)
            {
                res.set_content(reinterpret_cast<const char*>(&completion_js), completion_js_len, "application/javascript");
                return false;
            });
    
    # 当在public路径中找不到json-schema-to-grammar.mjs时调用，返回json-schema-to-grammar.mjs内容
    svr.Get("/json-schema-to-grammar.mjs", [](const httplib::Request &, httplib::Response &res)
            {
                res.set_content(reinterpret_cast<const char*>(&json_schema_to_grammar_mjs), json_schema_to_grammar_mjs_len, "application/javascript");
                return false;
            });
    # 设置处理 "/props" 路径的回调函数，返回用户名称和助手名称的 JSON 数据
    svr.Get("/props", [&llama](const httplib::Request & /*req*/, httplib::Response &res)
            {
                # 设置响应头，允许跨域访问
                res.set_header("Access-Control-Allow-Origin", "*");
                # 创建 JSON 数据，包含用户名称和助手名称
                json data = {
                    { "user_name",      llama.name_user.c_str() },
                    { "assistant_name", llama.name_assistant.c_str() }
                };
                # 设置响应内容为 JSON 格式
                res.set_content(data.dump(), "application/json");
            });

    # 设置处理 "/model.json" 路径的回调函数，返回模型属性的 JSON 数据
    svr.Get("/model.json", [&llama](const httplib::Request &, httplib::Response &res)
            {
                # 获取助手的模型属性，并设置响应内容为 JSON 格式
                const json data = llama.get_model_props();
                return res.set_content(data.dump(), "application/json");
            });

    # 设置处理所有路径的 OPTIONS 请求的回调函数，返回空的 JSON 数据
    svr.Options(R"(/.*)", [](const httplib::Request &, httplib::Response &res)
                { return res.set_content("", "application/json"); });

    # 设置处理 "/tokenize" 路径的回调函数，接收请求体中的内容，返回分词后的 JSON 数据
    svr.Post("/tokenize", [&llama](const httplib::Request &req, httplib::Response &res)
            {
                # 解析请求体中的 JSON 数据
                const json body = json::parse(req.body);
                # 创建存储分词结果的向量
                std::vector<llama_token> tokens;
                # 如果请求体中包含 "content" 字段
                if (body.count("content") != 0)
                {
                    # 对请求体中的内容进行分词
                    tokens = llama.tokenize(body["content"], false);
                }
                # 格式化分词结果为 JSON 数据，并设置为响应内容
                const json data = format_tokenizer_response(tokens);
                return res.set_content(data.dump(), "application/json");
            });

    # 设置处理 "/detokenize" 路径的回调函数，接收请求体中的分词结果，返回合并后的文本的 JSON 数据
    svr.Post("/detokenize", [&llama](const httplib::Request &req, httplib::Response &res)
            {
                # 解析请求体中的 JSON 数据
                const json body = json::parse(req.body);
                # 创建存储合并后文本的字符串
                std::string content;
                # 如果请求体中包含 "tokens" 字段
                if (body.count("tokens") != 0)
                {
                    # 将请求体中的分词结果转换为字符串
                    const std::vector<llama_token> tokens = body["tokens"];
                    content = tokens_to_str(llama.ctx, tokens.cbegin(), tokens.cend());
                }
                # 格式化合并后的文本为 JSON 数据，并设置为响应内容
                const json data = format_detokenized_response(content);
                return res.set_content(data.dump(), "application/json");
            });
    // 定义一个路由，处理 POST 请求 "/embedding"，并传入一个 lambda 函数作为处理函数
    svr.Post("/embedding", [&llama](const httplib::Request &req, httplib::Response &res)
            {
                // 解析请求的 body 为 JSON 格式
                const json body = json::parse(req.body);
                // 定义一个 JSON 对象 prompt
                json prompt;
                // 如果 body 中包含 "content" 字段
                if (body.count("content") != 0)
                {
                    // 将 prompt 设置为 body 中的 "content" 字段的值
                    prompt = body["content"];
                }
                else
                {
                    // 否则将 prompt 设置为空字符串
                    prompt = "";
                }
                // 发送请求到 llama 服务，获取任务 ID
                const int task_id = llama.request_completion({ {"prompt", prompt}, { "n_predict", 0} }, false, true);
                // 获取任务结果
                task_result result = llama.next_result(task_id);
                // 返回结果 JSON 格式的数据
                return res.set_content(result.result_json.dump(), "application/json");
            });

    // 设置日志记录器为 log_server_request 函数
    svr.set_logger(log_server_request);

    // 设置异常处理函数
    svr.set_exception_handler([](const httplib::Request &, httplib::Response &res, std::exception_ptr ep)
            {
                // 定义格式化字符串
                const char fmt[] = "500 Internal Server Error\n%s";
                // 定义缓冲区
                char buf[BUFSIZ];
                try
                {
                    // 重新抛出异常
                    std::rethrow_exception(std::move(ep));
                }
                catch (std::exception &e)
                {
                    // 将异常信息格式化到缓冲区中
                    snprintf(buf, sizeof(buf), fmt, e.what());
                }
                catch (...)
                {
                    // 如果捕获到未知异常，将 "Unknown Exception" 格式化到缓冲区中
                    snprintf(buf, sizeof(buf), fmt, "Unknown Exception");
                }
                // 设置响应内容为缓冲区中的内容，类型为 "text/plain"
                res.set_content(buf, "text/plain");
                // 设置响应状态码为 500
                res.status = 500;
            });

    // 设置错误处理函数
    svr.set_error_handler([](const httplib::Request &, httplib::Response &res)
            {
                // 如果响应状态码为 400
                if (res.status == 400)
                {
                    // 设置响应内容为 "Invalid request"，类型为 "text/plain"
                    res.set_content("Invalid request", "text/plain");
                }
                // 如果响应状态码不为 500
                else if (res.status != 500)
                {
                    // 设置响应内容为 "File Not Found"，类型为 "text/plain"
                    res.set_content("File Not Found", "text/plain");
                    // 设置响应状态码为 404
                    res.status = 404;
                }
            });

    // 设置读取超时时间和更改主机名和端口
    svr.set_read_timeout (sparams.read_timeout);
    # 设置写入超时时间
    svr.set_write_timeout(sparams.write_timeout);

    # 如果无法绑定到服务器端口，则输出错误信息并返回1
    if (!svr.bind_to_port(sparams.hostname, sparams.port))
    {
        fprintf(stderr, "\ncouldn't bind to server socket: hostname=%s port=%d\n\n", sparams.hostname.c_str(), sparams.port);
        return 1;
    }

    # 设置静态文件服务的基础目录
    svr.set_base_dir(sparams.public_path);

    # 输出服务器监听地址信息
    LOG_TEE("\nllama server listening at http://%s:%d\n\n", sparams.hostname.c_str(), sparams.port);

    # 输出HTTP服务器监听信息
    LOG_INFO("HTTP server listening", {
                                          {"hostname", sparams.hostname},
                                          {"port", sparams.port},
                                      });

    # 在线程中运行HTTP服务器 - 详细见下面的注释
    std::thread t([&]()
            {
                # 如果绑定后无法监听，则返回1
                if (!svr.listen_after_bind())
                {
                    return 1;
                }

                return 0;
            });

    # GG: 如果将主循环放在一个线程中，当在Debug模式下进行第一个请求时会崩溃！？
    #     "Bus error: 10" - 这是在macOS上，Linux上不会崩溃
    #std::thread t2([&]()
    {
        # 运行标志设为true
        bool running = true;
        # 当运行标志为true时，更新插槽
        while (running)
        {
            running = llama.update_slots();
        }
    }
    #);

    # 等待线程t结束
    t.join();

    # 释放llama后端资源
    llama_backend_free();
    # 返回0
    return 0;
# 闭合前面的函数定义
```