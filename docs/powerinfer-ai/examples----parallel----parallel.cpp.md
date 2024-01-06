# `PowerInfer\examples\parallel\parallel.cpp`

```
// 一个基本的应用程序，模拟具有多个客户端的服务器。
// 客户端向服务器提交请求，并且它们会并行处理。

#include "common.h"  // 包含通用的头文件
#include "llama.h"    // 包含 llama 的头文件

#include <cmath>      // 包含数学函数的头文件
#include <cstdio>     // 包含输入输出函数的头文件
#include <string>     // 包含字符串处理函数的头文件
#include <vector>     // 包含向量处理函数的头文件
#include <ctime>      // 包含时间处理函数的头文件

// 从字符串的开头和结尾修剪空白
static std::string trim(const std::string & str) {
    size_t start = 0;  // 初始化起始位置为 0
    size_t end = str.size();  // 初始化结束位置为字符串长度

    // 循环直到起始位置小于结束位置且起始位置的字符是空白字符
    while (start < end && isspace(str[start])) {
        start += 1;  // 起始位置向后移动一位
    }
    // ... (接下来的代码需要继续添加注释)
# 在字符串末尾去除空格
while (end > start && isspace(str[end - 1])) {
    end -= 1;
}

# 返回从指定位置开始到指定位置结束的子字符串
return str.substr(start, end - start);
}

# 定义一个包含多行文本的静态字符串变量
static std::string k_system =
R"(Transcript of a never ending dialog, where the User interacts with an Assistant.
The Assistant is helpful, kind, honest, good at writing, and never fails to answer the User's requests immediately and with precision.

User: Recommend a nice restaurant in the area.
User: Who is Richard Feynman?
User:)";

# 定义一个包含多个字符串的静态字符串向量
static std::vector<std::string> k_prompts = {
    "What is the meaning of life?",
    "Tell me an interesting fact about llamas.",
    "What is the best way to cook a steak?",
```
# 上述代码中的注释解释了每个语句的作用，使得代码更易于理解。
    "Are you familiar with the Special Theory of Relativity and can you explain it to me?", 
    // 询问对于相对论的了解程度，并能否解释
    "Recommend some interesting books to read.", 
    // 请求推荐一些有趣的书籍
    "What is the best way to learn a new language?", 
    // 询问学习新语言的最佳方法
    "How to get a job at Google?", 
    // 询问如何在谷歌找到工作
    "If you could have any superpower, what would it be?", 
    // 如果你能拥有超能力，你会选择什么？
    "I want to learn how to play the piano.", 
    // 表达想学习弹钢琴

};

struct client {
    ~client() {
        if (ctx_sampling) {
            llama_sampling_free(ctx_sampling);
        }
    }
    // 定义客户结构体，包含析构函数用于释放采样上下文

    int32_t id = 0;
    // 客户ID，默认为0

    llama_seq_id seq_id = -1;
    // 序列ID，默认为-1

    llama_token sampled;
    // 采样标记
// 定义起始时间变量，用于记录程序运行时间
int64_t t_start_prompt;
int64_t t_start_gen;

// 定义用于记录提示数量、解码数量和批次索引的变量
int32_t n_prompt  = 0;
int32_t n_decoded = 0;
int32_t i_batch   = -1;

// 定义存储输入、提示和响应的字符串变量
std::string input;
std::string prompt;
std::string response;

// 定义用于采样上下文的结构体指针变量，并初始化为空指针
struct llama_sampling_context * ctx_sampling = nullptr;
};

// 定义打印日期时间的函数
static void print_date_time() {
    // 获取当前时间
    std::time_t current_time = std::time(nullptr);
    // 将当前时间转换为本地时间
    std::tm* local_time = std::localtime(&current_time);
    // 定义存储日期时间的字符数组，并格式化当前时间
    char buffer[80];
    strftime(buffer, sizeof(buffer), "%Y-%m-%d %H:%M:%S", local_time);
// 打印运行参数，使用紫色字体
printf("\n\033[35mrun parameters as at %s\033[0m\n", buffer);
}

// 定义一个分割字符串的函数，将输入的字符串按照指定的分隔符分割成多个子字符串
static std::vector<std::string> split_string(const std::string& input, char delimiter) {
    std::vector<std::string> tokens; // 创建一个存储子字符串的容器
    std::istringstream stream(input); // 将输入的字符串转换成输入流
    std::string token; // 定义一个字符串变量用于存储每个子字符串
    while (std::getline(stream, token, delimiter)) { // 通过循环遍历输入流，按照分隔符将字符串分割成子字符串
        tokens.push_back(token); // 将每个子字符串存储到容器中
    }
    return tokens; // 返回存储子字符串的容器
}

int main(int argc, char ** argv) {
    srand(1234); // 设置随机数种子为1234

    gpt_params params; // 创建一个名为params的gpt_params对象
    // 如果参数解析失败，则返回1
    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 要模拟的同时“客户端”数量
    const int32_t n_clients = params.n_parallel;

    // 要模拟的请求数量
    const int32_t n_seq = params.n_sequences;

    // 当前请求完成后立即插入新请求
    const bool cont_batching = params.cont_batching;

    // 如果日志未禁用，则设置日志目标为生成的日志文件，并记录日志开始
#ifndef LOG_DISABLE_LOGS
    log_set_target(log_filename_generator("parallel", "log"));
    LOG_TEE("Log start\n");
    log_dump_cmdline(argc, argv);
#endif // LOG_DISABLE_LOGS

    // 初始化 llama.cpp
    // 初始化后端 llama
    llama_backend_init(params.numa);

    // 声明 model 和 ctx 指针
    llama_model * model = NULL;
    llama_context * ctx = NULL;

    // 加载目标模型
    params.logits_all = true;
    // 从 GPT 参数初始化 model 和 ctx
    std::tie(model, ctx) = llama_init_from_gpt_params(params);

    // 如果存在外部文件，则加载提示
    if (params.prompt.empty()) {
        // 如果没有新的问题，则使用内置默认值
        printf("\n\033[32mNo new questions so proceed with build-in defaults.\033[0m\n");
    } else {
        // 输出输入的 params.prompts 向量的每一行，并复制到 k_prompts
        int index = 0;
        // 打印外部提示文件的每一行
        printf("\n\033[32mNow printing the external prompt file %s\033[0m\n\n", params.prompt_file.c_str());

        // 将 params.prompt 按换行符分割成 prompts 向量
        std::vector<std::string> prompts = split_string(params.prompt, '\n');
        // 遍历 prompts 向量中的每个 prompt
        for (const auto& prompt : prompts) {
            // 调整 k_prompts 的大小并复制 prompt
            k_prompts.resize(index + 1);
    // 将 prompt 存储到 k_prompts 数组中的特定索引位置
    k_prompts[index] = prompt;
    // 索引自增
    index++;
    // 打印索引和 prompt 内容
    printf("%3d prompt: %s\n", index, prompt.c_str());

    // 打印空行
    fprintf(stderr, "\n\n");
    // 清空标准错误流
    fflush(stderr);

    // 获取上下文的数量
    const int n_ctx = llama_n_ctx(ctx);

    // 创建客户端对象的向量
    std::vector<client> clients(n_clients);
    // 遍历客户端向量，初始化每个客户端对象
    for (size_t i = 0; i < clients.size(); ++i) {
        auto & client = clients[i];
        client.id = i;
        client.ctx_sampling = llama_sampling_init(params.sparams);
    }

    // 创建系统令牌的向量
    std::vector<llama_token> tokens_system;
    // 为系统令牌向量赋值
    tokens_system = ::llama_tokenize(ctx, k_system, true);
    // 计算系统 tokens 的数量
    const int32_t n_tokens_system = tokens_system.size();

    // 初始化全局序列 ID
    llama_seq_id g_seq_id = 0;

    // 初始化 llama_batch 对象，用于处理输入的批次数据
    // 最大批处理大小取决于上下文的大小，以处理多个用户提供的非常长的输入提示。无论大小如何，主循环将批处理成每次最多 params.n_batch 个 tokens
    llama_batch batch = llama_batch_init(n_ctx, 0, 1);

    // 初始化计数器
    int32_t n_total_prompt = 0;
    int32_t n_total_gen    = 0;
    int32_t n_cache_miss   = 0;

    // 记录主循环开始时间
    const auto t_main_start = ggml_time_us();

    // 输出日志，模拟来自客户端的并行请求
    LOG_TEE("%s: Simulating parallel requests from clients:\n", __func__);
    LOG_TEE("%s: n_parallel = %d, n_sequences = %d, cont_batching = %d, system tokens = %d\n", __func__, n_clients, n_seq, cont_batching, n_tokens_system);
    LOG_TEE("\n");

    {
        // 输出日志，评估系统提示
        LOG_TEE("%s: Evaluating the system prompt ...\n", __func__);
// 遍历系统令牌的数量，将每个令牌添加到批处理中
for (int32_t i = 0; i < n_tokens_system; ++i) {
    llama_batch_add(batch, tokens_system[i], i, { 0 }, false);
}

// 如果解码失败，则记录错误信息并返回1
if (llama_decode(ctx, batch) != 0) {
    LOG_TEE("%s: llama_decode() failed\n", __func__);
    return 1;
}

// 将系统KV缓存分配给所有并行序列
for (int32_t i = 1; i < n_clients; ++i) {
    llama_kv_cache_seq_cp(ctx, 0, i, 0, n_tokens_system);
}

// 输出换行符
LOG_TEE("\n");
}

// 输出处理请求的信息
LOG_TEE("Processing requests ...\n\n");
    // 循环直到条件为真
    while (true) {
        // 清空批处理对象
        llama_batch_clear(batch);

        // 解码当前正在进行的序列
        for (auto & client : clients) {
            // 如果客户端的序列 ID 为 -1，则跳过
            if (client.seq_id == -1) {
                continue;
            }

            // 设置客户端的批处理索引
            client.i_batch = batch.n_tokens;

            // 向批处理对象中添加客户端的样本数据和标记数量
            llama_batch_add(batch, client.sampled, n_tokens_system + client.n_prompt + client.n_decoded, { client.id }, true);

            // 增加客户端已解码的数量
            client.n_decoded += 1;
        }

        // 如果批处理对象中的标记数量为0
        if (batch.n_tokens == 0) {
            // 所有序列已结束 - 清空整个 KV 缓存
            for (int i = 0; i < n_clients; ++i) {
                // 从上下文中移除指定范围内的序列缓存
                llama_kv_cache_seq_rm(ctx, i, n_tokens_system, -1);
        }

        // 打印日志，清空 KV 缓存
        LOG_TEE("%s: clearing the KV cache\n", __func__);

        // 如果是连续批处理或者批处理中没有令牌，则插入新的解码序列
        if (cont_batching || batch.n_tokens == 0) {
            // 遍历客户端列表
            for (auto & client : clients) {
                // 如果客户端的序列 ID 为-1，并且全局序列 ID 小于总序列数
                if (client.seq_id == -1 && g_seq_id < n_seq) {
                    // 设置客户端的序列 ID 为全局序列 ID
                    client.seq_id = g_seq_id;

                    // 设置客户端的开始提示时间为当前时间
                    client.t_start_prompt = ggml_time_us();
                    // 设置客户端的生成开始时间为0
                    client.t_start_gen    = 0;

                    // 从提示列表中随机选择一个提示作为输入
                    client.input    = k_prompts[rand() % k_prompts.size()];
                    // 将输入添加"Assistant:"作为提示
                    client.prompt   = client.input + "\nAssistant:";
                    // 将响应置为空
                    client.response = "";

                    // 重置采样上下文
                    llama_sampling_reset(client.ctx_sampling);
// 不要在系统提示前添加BOS，因为系统已经有了一个提示！
std::vector<llama_token> tokens_prompt; // 创建一个存储提示标记的向量
tokens_prompt = ::llama_tokenize(ctx, client.prompt, false); // 使用llama_tokenize函数对客户端提示进行标记化处理

for (size_t i = 0; i < tokens_prompt.size(); ++i) { // 遍历提示标记
    llama_batch_add(batch, tokens_prompt[i], i + n_tokens_system, { client.id }, false); // 将提示标记添加到批处理中
}

// 仅提取最后一个标记的logits
if (batch.n_tokens > 0) { // 如果批处理中有标记
    batch.logits[batch.n_tokens - 1] = true; // 设置最后一个标记的logits为true
}

client.n_prompt  = tokens_prompt.size(); // 设置客户端提示的标记数量
client.n_decoded = 0; // 设置客户端解码的数量为0
client.i_batch   = batch.n_tokens - 1; // 设置客户端批处理的标记数量减1

LOG_TEE("\033[31mClient %3d, seq %4d, started decoding ...\033[0m\n", client.id, client.seq_id); // 记录客户端开始解码的日志

g_seq_id += 1; // 全局序列ID加1
        // insert new requests one-by-one
        // 逐个插入新请求
        //if (cont_batching) {
        //    break;
        //}
        // 如果正在连续批处理，则跳出循环
    }
}
}

if (batch.n_tokens == 0) {
    break;
}
// 如果批处理的令牌数为0，则跳出循环

// process in chunks of params.n_batch
// 按照params.n_batch的大小进行分块处理
int32_t n_batch = params.n_batch;

for (int32_t i = 0; i < (int32_t) batch.n_tokens; i += n_batch) {
    // experiment: process in powers of 2
    // 实验：按2的幂进行处理
    //if (i + n_batch > (int32_t) batch.n_tokens && n_batch > 32) {
    //    n_batch /= 2;
    // 如果当前处理的范围超过了批处理的令牌数，并且批处理的大小大于32，则将批处理的大小减半
            // 计算当前批次中实际要处理的标记数量，取 n_batch 和 (batch.n_tokens - i) 中较小的值
            const int32_t n_tokens = std::min(n_batch, (int32_t) (batch.n_tokens - i));

            // 创建一个批次的子视图，包括要处理的标记数量、标记数组的偏移量、位置数组的偏移量、序列 ID 数组的偏移量、序列 ID 数组的指针、logits 数组的偏移量
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

            // 调用 llama_decode 函数对批次的子视图进行解码
            const int ret = llama_decode(ctx, batch_view);
            // 如果解码出错，且当前批次只有一个标记或者返回值小于 0
            if (ret != 0) {
// 如果程序执行到这里，说明键值缓存已满 - 尝试通过上下文大小增加它
LOG_TEE("%s : failed to decode the batch, n_batch = %d, ret = %d\n", __func__, n_batch, ret);
return 1;
// 打印错误日志并返回1

LOG("%s : failed to decode the batch, retrying with n_batch = %d\n", __func__, n_batch / 2);
// 打印日志，尝试使用一半的批处理大小重试

n_cache_miss += 1;
// 增加缓存未命中次数

// 使用一半的批处理大小重试以尝试在键值缓存中找到空闲槽
n_batch /= 2;
i -= n_batch;
// 更新批处理大小和索引，继续循环

LOG("%s : decoded batch of %d tokens\n", __func__, n_tokens);
// 打印日志，表示成功解码了一批指定数量的标记

for (auto & client : clients) {
    if (client.i_batch < (int) i || client.i_batch >= (int) (i + n_tokens)) {
// 遍历客户端列表，检查客户端的批处理索引是否在当前解码的标记范围内
                // 如果条件满足，则跳过本次循环，继续执行下一次循环
                continue;
                // 打印客户端的信息，包括 ID、序列号、采样情况、解码数量、批次号
                //printf("client %d, seq %d, token %d, pos %d, batch %d\n",
                //        client.id, client.seq_id, client.sampled, client.n_decoded, client.i_batch);
                // 使用采样器从上下文中获取一个 token
                const llama_token id = llama_sampling_sample(client.ctx_sampling, ctx, NULL, client.i_batch - i);
                // 接受采样得到的 token
                llama_sampling_accept(client.ctx_sampling, ctx, id, true);
                // 如果客户端解码数量为1，则开始测量生成时间，以确保所有并发客户端的提示已经被处理
                if (client.n_decoded == 1) {
                    client.t_start_gen = ggml_time_us();
                }
                // 将 token 转换为字符串，并添加到客户端的响应中
                const std::string token_str = llama_token_to_piece(ctx, id);
                client.response += token_str;
                client.sampled = id;
// 如果客户端已解码的数量大于2，并且满足以下条件之一：
// 1. 当前 token 为结束符
// 2. 预测的 token 数量大于0，并且客户端已解码的数量加上已提示的数量大于等于预测的数量
// 3. 客户端的响应中包含"User:"
// 4. 客户端的响应中包含换行符
if (client.n_decoded > 2 &&
    (id == llama_token_eos(model) ||
     (params.n_predict > 0 && client.n_decoded + client.n_prompt >= params.n_predict) ||
     client.response.find("User:") != std::string::npos ||
     client.response.find('\n') != std::string::npos)) {
    // 基本的反向提示

    // 在客户端响应中查找"User:"的位置
    const size_t pos = client.response.find("User:");
    if (pos != std::string::npos) {
        // 如果找到"User:"，则截取响应中从开头到"User:"之前的部分
        client.response = client.response.substr(0, pos);
    }

    // 仅删除序列中生成的部分，即在缓存中保留系统提示
    llama_kv_cache_seq_rm(ctx, client.id, n_tokens_system, -1);

    // 记录主要操作结束的时间
    const auto t_main_end = ggml_time_us();
}
# 打印客户端信息和输入输出内容
LOG_TEE("\033[31mClient %3d, seq %3d/%3d, prompt %4d t, response %4d t, time %5.2f s, speed %5.2f t/s, cache miss %d \033[0m \nInput:    %s\n\033[35mResponse: %s\033[0m\n\n",
        client.id, client.seq_id, n_seq, client.n_prompt, client.n_decoded,
        (t_main_end - client.t_start_prompt) / 1e6,
        (double) (client.n_prompt + client.n_decoded) / (t_main_end - client.t_start_prompt) * 1e6,
        n_cache_miss,
        ::trim(client.input).c_str(),
        ::trim(client.response).c_str());

# 更新总的提示数和生成数
n_total_prompt += client.n_prompt;
n_total_gen    += client.n_decoded;

# 重置客户端的序列号
client.seq_id = -1;

# 重置客户端的批次号
client.i_batch = -1;

# 计算主程序结束时间
const auto t_main_end = ggml_time_us();
# 打印当前日期和时间
print_date_time();

# 记录并打印并行数、序列数、连续批处理、系统标记数等参数信息
LOG_TEE("\n%s: n_parallel = %d, n_sequences = %d, cont_batching = %d, system tokens = %d\n", __func__, n_clients, n_seq, cont_batching, n_tokens_system);

# 如果外部提示文件为空，则使用内置默认值
if (params.prompt_file.empty()) {
    params.prompt_file = "used built-in defaults";
}
# 记录并打印外部提示文件的信息
LOG_TEE("External prompt file: \033[32m%s\033[0m\n", params.prompt_file.c_str());
# 记录并打印模型和路径的信息
LOG_TEE("Model and path used:  \033[32m%s\033[0m\n\n", params.model.c_str());

# 记录并打印总提示标记数和速度
LOG_TEE("Total prompt tokens: %6d, speed: %5.2f t/s\n", n_total_prompt, (double) (n_total_prompt              ) / (t_main_end - t_main_start) * 1e6);
# 记录并打印总生成标记数和速度
LOG_TEE("Total gen tokens:    %6d, speed: %5.2f t/s\n", n_total_gen,    (double) (n_total_gen                 ) / (t_main_end - t_main_start) * 1e6);
# 记录并打印总速度（平均）和速度
LOG_TEE("Total speed (AVG):   %6s  speed: %5.2f t/s\n", "",             (double) (n_total_prompt + n_total_gen) / (t_main_end - t_main_start) * 1e6);
# 记录并打印缓存未命中次数
LOG_TEE("Cache misses:        %6d\n", n_cache_miss);

# 打印空行
LOG_TEE("\n");

# 打印时间信息
llama_print_timings(ctx);

# 释放批处理内存
llama_batch_free(batch);
# 释放上下文资源
llama_free(ctx);
# 释放模型资源
llama_free_model(model);
# 释放后端资源
llama_backend_free();
# 输出换行到标准错误流
fprintf(stderr, "\n\n");
# 返回 0，表示程序正常结束
return 0;
}
```