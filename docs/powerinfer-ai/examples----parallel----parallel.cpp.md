# `PowerInfer\examples\parallel\parallel.cpp`

```cpp
// 一个模拟具有多个客户端的服务器的基本应用程序。
// 客户端向服务器提交请求，并且这些请求会并行处理。

#include "common.h"  // 引入 common.h 头文件
#include "llama.h"  // 引入 llama.h 头文件

#include <cmath>  // 引入数学函数库
#include <cstdio>  // 引入标准输入输出库
#include <string>  // 引入字符串库
#include <vector>  // 引入向量库
#include <ctime>  // 引入时间库

// 从字符串的开头和结尾修剪空白字符
static std::string trim(const std::string & str) {
    size_t start = 0;  // 初始化字符串起始位置
    size_t end = str.size();  // 初始化字符串结束位置

    while (start < end && isspace(str[start])) {  // 循环直到字符串开头不是空白字符
        start += 1;  // 增加起始位置
    }

    while (end > start && isspace(str[end - 1])) {  // 循环直到字符串结尾不是空白字符
        end -= 1;  // 减少结束位置
    }

    return str.substr(start, end - start);  // 返回修剪后的字符串
}

static std::string k_system =  // 定义包含系统对话的字符串
R"(Transcript of a never ending dialog, where the User interacts with an Assistant.
The Assistant is helpful, kind, honest, good at writing, and never fails to answer the User's requests immediately and with precision.

User: Recommend a nice restaurant in the area.
Assistant: I recommend the restaurant "The Golden Duck". It is a 5 star restaurant with a great view of the city. The food is delicious and the service is excellent. The prices are reasonable and the portions are generous. The restaurant is located at 123 Main Street, New York, NY 10001. The phone number is (212) 555-1234. The hours are Monday through Friday from 11:00 am to 10:00 pm. The restaurant is closed on Saturdays and Sundays.
User: Who is Richard Feynman?
Assistant: Richard Feynman was an American physicist who is best known for his work in quantum mechanics and particle physics. He was awarded the Nobel Prize in Physics in 1965 for his contributions to the development of quantum electrodynamics. He was a popular lecturer and author, and he wrote several books, including "Surely You're Joking, Mr. Feynman!" and "What Do You Care What Other People Think?".
User:)";  // 定义包含系统对话的字符串

static std::vector<std::string> k_prompts = {  // 定义包含提示的字符串向量
    "What is the meaning of life?",  // 提示1
    "Tell me an interesting fact about llamas.",  // 提示2
    "What is the best way to cook a steak?",  // 提示3
    # 以下是一系列字符串，每个字符串都是一个问题或请求的描述
    "Are you familiar with the Special Theory of Relativity and can you explain it to me?",
    "Recommend some interesting books to read.",
    "What is the best way to learn a new language?",
    "How to get a job at Google?",
    "If you could have any superpower, what would it be?",
    "I want to learn how to play the piano.",
// 结构体定义，包含客户端相关信息
struct client {
    // 析构函数，释放采样上下文内存
    ~client() {
        if (ctx_sampling) {
            llama_sampling_free(ctx_sampling);
        }
    }

    // 客户端 ID
    int32_t id = 0;

    // 序列 ID
    llama_seq_id seq_id = -1;

    // 采样标记
    llama_token sampled;

    // 开始提示时间
    int64_t t_start_prompt;
    // 开始生成时间
    int64_t t_start_gen;

    // 提示数量
    int32_t n_prompt  = 0;
    // 解码数量
    int32_t n_decoded = 0;
    // 批次索引
    int32_t i_batch   = -1;

    // 输入字符串
    std::string input;
    // 提示字符串
    std::string prompt;
    // 响应字符串
    std::string response;

    // 采样上下文指针
    struct llama_sampling_context * ctx_sampling = nullptr;
};

// 打印日期时间信息
static void print_date_time() {
    // 获取当前时间
    std::time_t current_time = std::time(nullptr);
    // 转换为本地时间
    std::tm* local_time = std::localtime(&current_time);
    // 格式化时间字符串
    char buffer[80];
    strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", local_time);

    // 打印时间信息
    printf("\n\033[35mrun parameters as at %s\033[0m\n", buffer);
}

// 定义字符串分割函数
static std::vector<std::string> split_string(const std::string& input, char delimiter) {
    std::vector<std::string> tokens;
    std::istringstream stream(input);
    std::string token;
    // 使用指定分隔符分割字符串
    while (std::getline(stream, token, delimiter)) {
        tokens.push_back(token);
    }
    return tokens;
}

// 主函数
int main(int argc, char ** argv) {
    // 设置随机数种子
    srand(1234);

    // 解析命令行参数
    gpt_params params;

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 模拟的同时客户端数量
    const int32_t n_clients = params.n_parallel;

    // 模拟的请求数量
    const int32_t n_seq = params.n_sequences;

    // 是否连续批处理
    const bool cont_batching = params.cont_batching;

    // 设置日志文件名
    #ifndef LOG_DISABLE_LOGS
    log_set_target(log_filename_generator("parallel", "log"));
    LOG_TEE("Log start\n");
    log_dump_cmdline(argc, argv);
    #endif // LOG_DISABLE_LOGS

    // 初始化 llama.cpp
    llama_backend_init(params.numa);

    llama_model * model = NULL;
    llama_context * ctx = NULL;

    // 加载目标模型
    params.logits_all = true;
}
    // 从 GPT 参数初始化模型和上下文
    std::tie(model, ctx) = llama_init_from_gpt_params(params);

    // 如果参数中的提示为空，则使用内置默认值
    if (params.prompt.empty()) {
        printf("\n\033[32mNo new questions so proceed with build-in defaults.\033[0m\n");
    } else {
        // 从外部文件加载提示，将每行内容存储到 prompts 向量中
        int index = 0;
        printf("\n\033[32mNow printing the external prompt file %s\033[0m\n\n", params.prompt_file.c_str());

        std::vector<std::string> prompts = split_string(params.prompt, '\n');
        for (const auto& prompt : prompts) {
            k_prompts.resize(index + 1);
            k_prompts[index] = prompt;
            index++;
            printf("%3d prompt: %s\n", index, prompt.c_str());
        }
    }

    // 刷新标准错误流
    fprintf(stderr, "\n\n");
    fflush(stderr);

    // 获取上下文的大小
    const int n_ctx = llama_n_ctx(ctx);

    // 创建客户端对象的向量
    std::vector<client> clients(n_clients);
    for (size_t i = 0; i < clients.size(); ++i) {
        auto & client = clients[i];
        client.id = i;
        client.ctx_sampling = llama_sampling_init(params.sparams);
    }

    // 创建系统标记的向量并进行标记
    std::vector<llama_token> tokens_system;
    tokens_system = ::llama_tokenize(ctx, k_system, true);
    const int32_t n_tokens_system = tokens_system.size();

    // 初始化全局序列 ID
    llama_seq_id g_seq_id = 0;

    // 初始化批处理对象
    llama_batch batch = llama_batch_init(n_ctx, 0, 1);

    // 初始化计数器
    int32_t n_total_prompt = 0;
    int32_t n_total_gen    = 0;
    int32_t n_cache_miss   = 0;

    // 记录主程序开始时间
    const auto t_main_start = ggml_time_us();

    // 输出并记录日志
    LOG_TEE("%s: Simulating parallel requests from clients:\n", __func__);
    LOG_TEE("%s: n_parallel = %d, n_sequences = %d, cont_batching = %d, system tokens = %d\n", __func__, n_clients, n_seq, cont_batching, n_tokens_system);
    LOG_TEE("\n");
    {
        // 打印日志，表示正在评估系统提示
        LOG_TEE("%s: Evaluating the system prompt ...\n", __func__);
    
        // 遍历系统提示的所有标记
        for (int32_t i = 0; i < n_tokens_system; ++i) {
            // 将标记添加到批处理中
            llama_batch_add(batch, tokens_system[i], i, { 0 }, false);
        }
    
        // 如果 llama_decode() 返回非零值，表示解码失败
        if (llama_decode(ctx, batch) != 0) {
            LOG_TEE("%s: llama_decode() failed\n", __func__);
            return 1;
        }
    
        // 将系统 KV 缓存分配给所有并行序列
        for (int32_t i = 1; i < n_clients; ++i) {
            llama_kv_cache_seq_cp(ctx, 0, i, 0, n_tokens_system);
        }
    
        LOG_TEE("\n");
    }
    
    LOG_TEE("Processing requests ...\n\n");
    
    }
    
    // 计算主函数结束时间
    const auto t_main_end = ggml_time_us();
    
    // 打印日期和时间
    print_date_time();
    
    // 打印日志，记录并行数、序列数、批处理连续性和系统标记数
    LOG_TEE("\n%s: n_parallel = %d, n_sequences = %d, cont_batching = %d, system tokens = %d\n", __func__, n_clients, n_seq, cont_batching, n_tokens_system);
    // 如果参数中的提示文件为空，则使用内置默认值
    if (params.prompt_file.empty()) {
        params.prompt_file = "used built-in defaults";
    }
    // 打印外部提示文件
    LOG_TEE("External prompt file: \033[32m%s\033[0m\n", params.prompt_file.c_str());
    // 打印使用的模型和路径
    LOG_TEE("Model and path used:  \033[32m%s\033[0m\n\n", params.model.c_str());
    
    // 打印总提示标记数和速度
    LOG_TEE("Total prompt tokens: %6d, speed: %5.2f t/s\n", n_total_prompt, (double) (n_total_prompt              ) / (t_main_end - t_main_start) * 1e6);
    // 打印总生成标记数和速度
    LOG_TEE("Total gen tokens:    %6d, speed: %5.2f t/s\n", n_total_gen,    (double) (n_total_gen                 ) / (t_main_end - t_main_start) * 1e6);
    // 打印总速度（平均）和速度
    LOG_TEE("Total speed (AVG):   %6s  speed: %5.2f t/s\n", "",             (double) (n_total_prompt + n_total_gen) / (t_main_end - t_main_start) * 1e6);
    // 打印缓存未命中次数
    LOG_TEE("Cache misses:        %6d\n", n_cache_miss);
    
    LOG_TEE("\n");
    
    // 打印 llama 的时间统计信息
    llama_print_timings(ctx);
    
    // 释放批处理对象
    llama_batch_free(batch);
    
    // 释放 llama 上下文对象
    llama_free(ctx);
    // 释放模型对象
    llama_free_model(model);
    
    // 释放 llama 后端资源
    llama_backend_free();
    
    // 打印错误信息
    fprintf(stderr, "\n\n");
    
    // 返回成功
    return 0;
    }
# 闭合前面的函数定义
```