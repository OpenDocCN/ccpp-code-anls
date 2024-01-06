# `PowerInfer\examples\simple\simple.cpp`

```
#include "common.h"
#include "llama.h"
// 引入自定义的头文件 common.h 和 llama.h

#include <cmath>
#include <cstdio>
#include <string>
#include <vector>
// 引入标准库的头文件

int main(int argc, char ** argv) {
    // 定义 gpt_params 结构体变量
    gpt_params params;

    // 如果命令行参数个数为 1 或者第一个参数的第一个字符为 '-'，则打印使用说明并返回 1
    if (argc == 1 || argv[1][0] == '-') {
        printf("usage: %s MODEL_PATH [PROMPT]\n" , argv[0]);
        return 1 ;
    }

    // 如果命令行参数个数大于等于 2，则将第二个参数赋值给 params 结构体的 model 成员
    if (argc >= 2) {
        params.model = argv[1];
    }
    // 检查命令行参数数量是否大于等于3，如果是则将第三个参数赋值给params.prompt
    if (argc >= 3) {
        params.prompt = argv[2];
    }

    // 如果params.prompt为空，则将其赋值为"Hello my name is"
    if (params.prompt.empty()) {
        params.prompt = "Hello my name is";
    }

    // 定义序列的总长度，包括提示信息
    const int n_len = 32;

    // 初始化LLM后端
    llama_backend_init(params.numa);

    // 初始化模型参数
    llama_model_params model_params = llama_model_default_params();

    // 设置模型参数中的n_gpu_layers为99，将所有层都卸载到GPU上
    // model_params.n_gpu_layers = 99; // offload all layers to the GPU
// 从文件中加载 LLAMA 模型，使用给定的参数
llama_model * model = llama_load_model_from_file(params.model.c_str(), model_params);

// 如果加载模型失败，打印错误信息并返回 1
if (model == NULL) {
    fprintf(stderr , "%s: error: unable to load model\n" , __func__);
    return 1;
}

// 初始化上下文参数
llama_context_params ctx_params = llama_context_default_params();

// 设置上下文参数的种子、上下文数量、线程数量
ctx_params.seed  = 1234;
ctx_params.n_ctx = 2048;
ctx_params.n_threads = params.n_threads;
ctx_params.n_threads_batch = params.n_threads_batch == -1 ? params.n_threads : params.n_threads_batch;

// 创建一个新的上下文，使用加载的模型和上下文参数
llama_context * ctx = llama_new_context_with_model(model, ctx_params);

// 如果创建上下文失败
if (ctx == NULL) {
// 打印错误信息，指示无法创建 llama_context
fprintf(stderr , "%s: error: failed to create the llama_context\n" , __func__);
// 返回错误代码 1
return 1;
}

// 对提示进行标记化处理
std::vector<llama_token> tokens_list;
// 使用 llama_tokenize 函数对提示进行标记化处理，并将结果存储在 tokens_list 中
tokens_list = ::llama_tokenize(ctx, params.prompt, true);

// 获取 llama_context 中的上下文数和 tokens_list 的大小，计算所需的 KV 缓存大小
const int n_ctx    = llama_n_ctx(ctx);
const int n_kv_req = tokens_list.size() + (n_len - tokens_list.size());

// 记录日志，显示 n_len、n_ctx 和 n_kv_req 的值
LOG_TEE("\n%s: n_len = %d, n_ctx = %d, n_kv_req = %d\n", __func__, n_len, n_ctx, n_kv_req);

// 确保 KV 缓存足够大，能够容纳所有提示和生成的标记
if (n_kv_req > n_ctx) {
    // 打印错误信息，指示所需的 KV 缓存大小不够
    LOG_TEE("%s: error: n_kv_req > n_ctx, the required KV cache size is not big enough\n", __func__);
    // 继续打印建议的解决方法
    LOG_TEE("%s:        either reduce n_parallel or increase n_ctx\n", __func__);
    // 返回错误代码 1
    return 1;
}
// 逐个打印提示符的标记
fprintf(stderr, "\n");

// 遍历标记列表，逐个打印标记对应的字符串
for (auto id : tokens_list) {
    fprintf(stderr, "%s", llama_token_to_piece(ctx, id).c_str());
}
// 刷新标准错误流
fflush(stderr);

// 创建一个大小为512的llama_batch对象
// 我们使用这个对象来提交用于解码的标记数据
llama_batch batch = llama_batch_init(512, 0, 1);

// 评估初始提示
for (size_t i = 0; i < tokens_list.size(); i++) {
    // 向llama_batch对象添加标记数据
    llama_batch_add(batch, tokens_list[i], i, { 0 }, false);
}
    // 设置batch中最后一个token的logits为true
    batch.logits[batch.n_tokens - 1] = true;

    // 如果llama_decode函数返回非0，表示解码失败，记录错误信息并返回1
    if (llama_decode(ctx, batch) != 0) {
        LOG_TEE("%s: llama_decode() failed\n", __func__);
        return 1;
    }

    // 主循环

    // 初始化当前token数量为batch中的token数量，解码数量为0
    int n_cur    = batch.n_tokens;
    int n_decode = 0;

    // 记录主循环开始时间
    const auto t_main_start = ggml_time_us();

    // 当当前token数量小于等于总token数量时执行循环
    while (n_cur <= n_len) {
        // 采样下一个token
        {
            // 获取模型的词汇表大小
            auto   n_vocab = llama_n_vocab(model);
// 获取当前位置的预测结果
auto * logits  = llama_get_logits_ith(ctx, batch.n_tokens - 1);

// 创建候选词列表，并预留空间
std::vector<llama_token_data> candidates;
candidates.reserve(n_vocab);

// 遍历词汇表，将每个词的信息加入候选词列表
for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
    candidates.emplace_back(llama_token_data{ token_id, logits[token_id], 0.0f });
}

// 将候选词列表转换为 llama_token_data_array 结构
llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };

// 从候选词中选择最有可能的词
const llama_token new_token_id = llama_sample_token_greedy(ctx, &candidates_p);

// 判断是否为流的结束
if (new_token_id == llama_token_eos(model) || n_cur == n_len) {
    LOG_TEE("\n");
    // 结束循环
    break;
}
// 将 llama_token_to_piece 函数返回的字符串输出到日志中
LOG_TEE("%s", llama_token_to_piece(ctx, new_token_id).c_str());
// 刷新标准输出流
fflush(stdout);

// 清空批处理对象中的数据，准备下一批数据
llama_batch_clear(batch);

// 将新的令牌添加到批处理对象中，用于下一次评估
llama_batch_add(batch, new_token_id, n_cur, { 0 }, true);

// 增加解码计数
n_decode += 1;

// 增加当前令牌的索引
n_cur += 1;

// 使用变换器模型评估当前批处理的数据
if (llama_decode(ctx, batch)) {
    // 如果评估失败，输出错误信息并返回错误代码
    fprintf(stderr, "%s : failed to eval, return code %d\n", __func__, 1);
    return 1;
}
    } 
    // 结束当前的代码块

    LOG_TEE("\n");
    // 打印一个换行符

    const auto t_main_end = ggml_time_us();
    // 获取当前时间，用于计算总运行时间

    LOG_TEE("%s: decoded %d tokens in %.2f s, speed: %.2f t/s\n",
            __func__, n_decode, (t_main_end - t_main_start) / 1000000.0f, n_decode / ((t_main_end - t_main_start) / 1000000.0f));
    // 打印解码的 token 数量，总运行时间以及速度

    llama_print_timings(ctx);
    // 打印 llama 的时间统计信息

    fprintf(stderr, "\n");
    // 打印一个换行符到标准错误流

    llama_batch_free(batch);
    // 释放批处理对象占用的内存

    llama_free(ctx);
    // 释放上下文对象占用的内存
    llama_free_model(model);
    // 释放模型对象占用的内存

    llama_backend_free();
    // 释放 llama 后端占用的资源
# 返回整数值0，结束函数的执行
```