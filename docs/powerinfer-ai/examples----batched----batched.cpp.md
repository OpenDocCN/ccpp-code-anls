# `PowerInfer\examples\batched\batched.cpp`

```
#include "common.h"
#include "llama.h"

#include <algorithm>
#include <cmath>
#include <cstdio>
#include <string>
#include <vector>

int main(int argc, char ** argv) {
    gpt_params params;

    if (argc == 1 || argv[1][0] == '-') {
        // 如果没有提供参数或者第一个参数以'-'开头，则打印使用说明并返回1
        printf("usage: %s MODEL_PATH [PROMPT] [PARALLEL] [LEN] [N_THREAD] [VRAM_BUDGET] [NGL]\n" , argv[0]);
        return 1 ;
    }

    // number of parallel batches
    int n_parallel = 1;

    // total length of the sequences including the prompt
    int n_len = 32;

    // number of layers to offload to the GPU
    int n_gpu_layers = 0;

    // vram budget in GiB
    double vram_budget = -1;

    if (argc >= 2) {
        // 设置模型路径
        params.model = argv[1];
    }

    if (argc >= 3) {
        // 设置提示文本
        params.prompt = argv[2];
    }

    if (argc >= 4) {
        // 设置并行批次数
        n_parallel = std::atoi(argv[3]);
    }

    if (argc >= 5) {
        // 设置序列的总长度
        n_len = std::atoi(argv[4]);
    }

    if (argc >= 6) {
        // 设置线程数
        params.n_threads = std::atoi(argv[5]);
    }

    if (argc >= 7) {
        // 设置显存预算
        vram_budget = std::atof(argv[6]);
    }

    if (argc >= 8) {
        // 设置要卸载到GPU的层数
        n_gpu_layers = std::atoi(argv[7]);
    }

    // 打印参数信息
    printf("params: model = %s, prompt = %s, n_parallel = %d, n_len = %d, n_gpu_layers = %d, n_threads = %d, vram_budget = %.2f GiB, reset_gpu_index = true\n",
           params.model.c_str(), params.prompt.c_str(), n_parallel, n_len, n_gpu_layers, params.n_threads, vram_budget);

    if (params.prompt.empty()) {
        // 如果提示文本为空，则设置默认提示文本
        params.prompt = "Hello my name is";
    }

    // 初始化LLM后端
    llama_backend_init(params.numa);

    // 初始化模型参数
    llama_model_params model_params = llama_model_default_params();

    // 设置要卸载到GPU的层数
    model_params.n_gpu_layers = n_gpu_layers;
    // 为了测试目的，我们总是重置GPU索引
    model_params.reset_gpu_index = true;
    // 设置显存预算
    model_params.vram_budget_gb = vram_budget;

    // 从文件加载模型
    llama_model * model = llama_load_model_from_file(params.model.c_str(), model_params);
}
    // 如果模型为空，则打印错误信息并返回1
    if (model == NULL) {
        fprintf(stderr , "%s: error: unable to load model\n" , __func__);
        return 1;
    }

    // 对提示进行标记化处理
    std::vector<llama_token> tokens_list;
    tokens_list = ::llama_tokenize(model, params.prompt, true);
    const int n_kv_req = tokens_list.size() + (n_len - tokens_list.size())*n_parallel;

    // 初始化上下文
    llama_context_params ctx_params = llama_context_default_params();
    ctx_params.seed  = 1234;
    ctx_params.n_ctx = n_kv_req;
    ctx_params.n_batch = std::max(n_len, n_parallel);
    ctx_params.n_threads       = params.n_threads;
    ctx_params.n_threads_batch = params.n_threads_batch == -1 ? params.n_threads : params.n_threads_batch;

    // 使用模型和上下文参数创建 llama_context 对象
    llama_context * ctx = llama_new_context_with_model(model, ctx_params);

    // 如果创建 llama_context 失败，则打印错误信息并返回1
    if (ctx == NULL) {
        fprintf(stderr , "%s: error: failed to create the llama_context\n" , __func__);
        return 1;
    }

    // 获取上下文中的 token 数量
    const int n_ctx    = llama_n_ctx(ctx);

    // 打印上下文中的 token 数量等信息
    LOG_TEE("\n%s: n_len = %d, n_ctx = %d, n_batch = %d, n_parallel = %d, n_kv_req = %d\n", __func__, n_len, n_ctx, ctx_params.n_batch, n_parallel, n_kv_req);

    // 确保 KV 缓存足够大，能够容纳所有的提示和生成的 token
    if (n_kv_req > n_ctx) {
        LOG_TEE("%s: error: n_kv_req (%d) > n_ctx, the required KV cache size is not big enough\n", __func__,  n_kv_req);
        LOG_TEE("%s:        either reduce n_parallel or increase n_ctx\n", __func__);
        return 1;
    }

    // 逐个打印提示的 token
    fprintf(stderr, "\n");
    for (auto id : tokens_list) {
        fprintf(stderr, "%s", llama_token_to_piece(ctx, id).c_str());
    }
    fflush(stderr);

    // 创建一个 llama_batch 对象
    // 用于提交 token 数据进行解码
    llama_batch batch = llama_batch_init(std::max(tokens_list.size(), (size_t)n_parallel), 0, 1);

    // 评估初始提示
    // 遍历 tokens_list 中的所有元素
    for (size_t i = 0; i < tokens_list.size(); ++i) {
        // 将 tokens_list[i] 添加到 batch 中，同时指定索引和标志
        llama_batch_add(batch, tokens_list[i], i, { 0 }, false);
    }
    // 断言 batch 中的 tokens 数量等于 tokens_list 的大小
    GGML_ASSERT(batch.n_tokens == (int) tokens_list.size());

    // 设置 batch 中最后一个 token 的 logits 为 true
    // llama_decode 只会输出 prompt 的最后一个 token 的 logits
    batch.logits[batch.n_tokens - 1] = true;

    // 调用 llama_decode 进行解码，如果失败则记录日志并返回 1
    if (llama_decode(ctx, batch) != 0) {
        LOG_TEE("%s: llama_decode() failed\n", __func__);
        return 1;
    }

    // 将系统 KV 缓存分配给所有并行序列
    // 这样，并行序列将“重用” prompt tokens 而无需复制它们
    for (int32_t i = 1; i < n_parallel; ++i) {
        llama_kv_cache_seq_cp(ctx, 0, i, 0, batch.n_tokens);
    }

    // 如果并行度大于 1，则记录生成序列的日志
    if (n_parallel > 1) {
        LOG_TEE("\n\n%s: generating %d sequences ...\n", __func__, n_parallel);
    }

    // 主循环

    // 用于存储并行解码序列的向量
    std::vector<std::string> streams(n_parallel);

    // 记录每个并行序列的最后一个 token 的 batch 索引
    // 这是为了确定从哪个 logits 中进行采样
    std::vector<int32_t> i_batch(n_parallel, batch.n_tokens - 1);

    // 当前 tokens 数量和已解码 tokens 数量的初始化
    int n_cur    = batch.n_tokens;
    int n_decode = 0;

    // 记录主循环开始时间
    const auto t_main_start = ggml_time_us();

    }

    // 记录主循环结束时间
    const auto t_main_end = ggml_time_us();

    // 记录解码 tokens 数量和解码速度的日志
    LOG_TEE("%s: decoded %d tokens in %.2f s, speed: %.2f t/s\n",
            __func__, n_decode, (t_main_end - t_main_start) / 1000000.0f, n_decode / ((t_main_end - t_main_start) / 1000000.0f));

    // 打印 llama 的时间统计信息
    llama_print_timings(ctx);

    // 打印换行符
    fprintf(stderr, "\n");

    // 释放 batch 对象
    llama_batch_free(batch);

    // 释放 ctx 对象
    llama_free(ctx);
    // 释放 model 对象
    llama_free_model(model);

    // 释放 llama 后端资源
    llama_backend_free();

    // 返回 0
    return 0;
# 闭合前面的函数定义
```