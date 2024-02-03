# `PowerInfer\examples\simple\simple.cpp`

```cpp
#include "common.h"
#include "llama.h"

#include <cmath>
#include <cstdio>
#include <string>
#include <vector>

int main(int argc, char ** argv) {
    // 定义参数结构体
    gpt_params params;

    // 检查命令行参数数量，如果不符合要求则打印用法说明并返回错误码
    if (argc == 1 || argv[1][0] == '-') {
        printf("usage: %s MODEL_PATH [PROMPT]\n" , argv[0]);
        return 1 ;
    }

    // 如果命令行参数中包含模型路径，则将其赋值给params.model
    if (argc >= 2) {
        params.model = argv[1];
    }

    // 如果命令行参数中包含提示语，则将其赋值给params.prompt
    if (argc >= 3) {
        params.prompt = argv[2];
    }

    // 如果提示语为空，则设置默认提示语
    if (params.prompt.empty()) {
        params.prompt = "Hello my name is";
    }

    // 定义序列的总长度，包括提示语
    const int n_len = 32;

    // 初始化LLM后端
    llama_backend_init(params.numa);

    // 初始化模型参数
    llama_model_params model_params = llama_model_default_params();

    // 从文件加载模型
    llama_model * model = llama_load_model_from_file(params.model.c_str(), model_params);

    // 如果加载模型失败，则打印错误信息并返回错误码
    if (model == NULL) {
        fprintf(stderr , "%s: error: unable to load model\n" , __func__);
        return 1;
    }

    // 初始化上下文参数
    llama_context_params ctx_params = llama_context_default_params();

    ctx_params.seed  = 1234;
    ctx_params.n_ctx = 2048;
    ctx_params.n_threads = params.n_threads;
    ctx_params.n_threads_batch = params.n_threads_batch == -1 ? params.n_threads : params.n_threads_batch;

    // 创建上下文
    llama_context * ctx = llama_new_context_with_model(model, ctx_params);

    // 如果创建上下文失败，则打印错误信息并返回错误码
    if (ctx == NULL) {
        fprintf(stderr , "%s: error: failed to create the llama_context\n" , __func__);
        return 1;
    }

    // 对提示语进行分词
    std::vector<llama_token> tokens_list;
    tokens_list = ::llama_tokenize(ctx, params.prompt, true);

    // 获取上下文的长度和所需的键值对数量
    const int n_ctx    = llama_n_ctx(ctx);
    const int n_kv_req = tokens_list.size() + (n_len - tokens_list.size());

    // 打印上下文长度和键值对数量
    LOG_TEE("\n%s: n_len = %d, n_ctx = %d, n_kv_req = %d\n", __func__, n_len, n_ctx, n_kv_req);

    // 确保键值对缓存足够大，能够容纳所有的提示语和生成的标记
    // 如果请求的键值缓存大小大于上下文大小，则输出错误信息并返回1
    if (n_kv_req > n_ctx) {
        LOG_TEE("%s: error: n_kv_req > n_ctx, the required KV cache size is not big enough\n", __func__);
        LOG_TEE("%s:        either reduce n_parallel or increase n_ctx\n", __func__);
        return 1;
    }

    // 逐个打印提示符的标记
    fprintf(stderr, "\n");
    for (auto id : tokens_list) {
        fprintf(stderr, "%s", llama_token_to_piece(ctx, id).c_str());
    }
    fflush(stderr);

    // 创建一个大小为512的llama_batch对象
    // 我们使用这个对象来提交用于解码的标记数据
    llama_batch batch = llama_batch_init(512, 0, 1);

    // 评估初始提示
    for (size_t i = 0; i < tokens_list.size(); i++) {
        llama_batch_add(batch, tokens_list[i], i, { 0 }, false);
    }

    // llama_decode仅为提示的最后一个标记输出logits
    batch.logits[batch.n_tokens - 1] = true;

    // 如果llama_decode失败，则输出错误信息并返回1
    if (llama_decode(ctx, batch) != 0) {
        LOG_TEE("%s: llama_decode() failed\n", __func__);
        return 1;
    }

    // 主循环

    // 当前标记数
    int n_cur    = batch.n_tokens;
    // 解码数
    int n_decode = 0;

    // 记录主循环开始的时间
    const auto t_main_start = ggml_time_us();
    // 当当前处理的 token 数小于等于总 token 数时执行循环
    while (n_cur <= n_len) {
        // 采样下一个 token
        {
            // 获取词汇表大小
            auto   n_vocab = llama_n_vocab(model);
            // 获取当前 token 的 logits
            auto * logits  = llama_get_logits_ith(ctx, batch.n_tokens - 1);

            // 创建候选 token 数据的向量
            std::vector<llama_token_data> candidates;
            candidates.reserve(n_vocab);

            // 遍历词汇表，将每个 token 的数据加入候选列表
            for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
                candidates.emplace_back(llama_token_data{ token_id, logits[token_id], 0.0f });
            }

            // 创建候选 token 数据的数组
            llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };

            // 采样最有可能的 token
            const llama_token new_token_id = llama_sample_token_greedy(ctx, &candidates_p);

            // 如果是流的结束标志或者当前处理的 token 数等于总 token 数，则跳出循环
            if (new_token_id == llama_token_eos(model) || n_cur == n_len) {
                LOG_TEE("\n");

                break;
            }

            // 打印当前 token 对应的字符串
            LOG_TEE("%s", llama_token_to_piece(ctx, new_token_id).c_str());
            fflush(stdout);

            // 准备下一个 batch
            llama_batch_clear(batch);

            // 将新的 token 添加到下一个评估中
            llama_batch_add(batch, new_token_id, n_cur, { 0 }, true);

            n_decode += 1;
        }

        n_cur += 1;

        // 使用 transformer 模型评估当前 batch
        if (llama_decode(ctx, batch)) {
            fprintf(stderr, "%s : failed to eval, return code %d\n", __func__, 1);
            return 1;
        }
    }

    LOG_TEE("\n");

    // 计算主循环结束时间
    const auto t_main_end = ggml_time_us();

    // 打印解码的 token 数、总时间和速度
    LOG_TEE("%s: decoded %d tokens in %.2f s, speed: %.2f t/s\n",
            __func__, n_decode, (t_main_end - t_main_start) / 1000000.0f, n_decode / ((t_main_end - t_main_start) / 1000000.0f));

    // 打印 llama 的时间统计
    llama_print_timings(ctx);

    fprintf(stderr, "\n");

    // 释放 batch
    llama_batch_free(batch);

    // 释放上下文
    llama_free(ctx);
    // 释放模型
    llama_free_model(model);

    // 释放 llama 后端资源
    llama_backend_free();

    // 返回成功
    return 0;
# 闭合前面的函数定义
```