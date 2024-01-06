# `PowerInfer\examples\batched\batched.cpp`

```
#include "common.h" // 包含 common.h 头文件
#include "llama.h" // 包含 llama.h 头文件

#include <algorithm> // 包含算法库
#include <cmath> // 包含数学库
#include <cstdio> // 包含输入输出库
#include <string> // 包含字符串库
#include <vector> // 包含向量库

int main(int argc, char ** argv) { // 主函数，接受命令行参数
    gpt_params params; // 定义 gpt_params 结构体变量 params

    if (argc == 1 || argv[1][0] == '-') { // 如果命令行参数个数为1或者第一个参数的第一个字符为'-'，则执行以下代码
        printf("usage: %s MODEL_PATH [PROMPT] [PARALLEL] [LEN] [N_THREAD] [VRAM_BUDGET] [NGL]\n" , argv[0]); // 打印使用说明
        return 1 ; // 返回错误码1
    }

    // number of parallel batches
    int n_parallel = 1; // 定义并初始化并行批次数为1
    // 定义变量n_len，表示序列的总长度，包括提示部分
    int n_len = 32;

    // 定义变量n_gpu_layers，表示要卸载到GPU的层数
    int n_gpu_layers = 0;

    // 定义变量vram_budget，表示显存预算，单位为GiB
    double vram_budget = -1;

    // 如果命令行参数的数量大于等于2，将第二个参数赋值给params.model
    if (argc >= 2) {
        params.model = argv[1];
    }

    // 如果命令行参数的数量大于等于3，将第三个参数赋值给params.prompt
    if (argc >= 3) {
        params.prompt = argv[2];
    }

    // 如果命令行参数的数量大于等于4，将第四个参数转换为整数并赋值给n_parallel
    if (argc >= 4) {
        n_parallel = std::atoi(argv[3]);
    }
# 如果命令行参数的数量大于等于5，将第4个参数转换为整数并赋值给n_len
if (argc >= 5) {
    n_len = std::atoi(argv[4]);
}

# 如果命令行参数的数量大于等于6，将第5个参数转换为整数并赋值给params.n_threads
if (argc >= 6) {
    params.n_threads = std::atoi(argv[5]);
}

# 如果命令行参数的数量大于等于7，将第6个参数转换为浮点数并赋值给vram_budget
if (argc >= 7) {
    vram_budget = std::atof(argv[6]);
}

# 如果命令行参数的数量大于等于8，将第7个参数转换为整数并赋值给n_gpu_layers
if (argc >= 8) {
    n_gpu_layers = std::atoi(argv[7]);
}

# 打印参数的数值
printf("params: model = %s, prompt = %s, n_parallel = %d, n_len = %d, n_gpu_layers = %d, n_threads = %d, vram_budget = %.2f GiB, reset_gpu_index = true\n",
       params.model.c_str(), params.prompt.c_str(), n_parallel, n_len, n_gpu_layers, params.n_threads, vram_budget);
    // 如果参数中的提示为空，则设置默认提示为"Hello my name is"
    if (params.prompt.empty()) {
        params.prompt = "Hello my name is";
    }

    // 初始化LLM后端
    llama_backend_init(params.numa);

    // 初始化模型参数
    llama_model_params model_params = llama_model_default_params();

    // 设置GPU层数
    model_params.n_gpu_layers = n_gpu_layers;
    // 为了测试目的，总是重置GPU索引
    model_params.reset_gpu_index = true;
    // 设置VRAM预算
    model_params.vram_budget_gb = vram_budget;

    // 从文件加载模型
    llama_model * model = llama_load_model_from_file(params.model.c_str(), model_params);

    // 如果加载模型失败
    if (model == NULL) {
// 打印错误信息到标准错误流，并返回错误代码
fprintf(stderr , "%s: error: unable to load model\n" , __func__);
return 1;
}

// 对提示进行分词
std::vector<llama_token> tokens_list;
tokens_list = ::llama_tokenize(model, params.prompt, true);
const int n_kv_req = tokens_list.size() + (n_len - tokens_list.size())*n_parallel;

// 初始化上下文参数
llama_context_params ctx_params = llama_context_default_params();

ctx_params.seed  = 1234; // 设置随机种子
ctx_params.n_ctx = n_kv_req; // 设置上下文大小
ctx_params.n_batch = std::max(n_len, n_parallel); // 设置批处理大小为长度和并行度的最大值
ctx_params.n_threads       = params.n_threads; // 设置线程数
ctx_params.n_threads_batch = params.n_threads_batch == -1 ? params.n_threads : params.n_threads_batch; // 设置批处理线程数，如果未指定则与总线程数相同
// 使用给定的模型和上下文参数创建 llama_context 对象
llama_context * ctx = llama_new_context_with_model(model, ctx_params);

// 如果创建 llama_context 失败，输出错误信息并返回 1
if (ctx == NULL) {
    fprintf(stderr , "%s: error: failed to create the llama_context\n" , __func__);
    return 1;
}

// 获取 llama_context 中的上下文数量
const int n_ctx    = llama_n_ctx(ctx);

// 输出上下文数量以及其他参数信息
LOG_TEE("\n%s: n_len = %d, n_ctx = %d, n_batch = %d, n_parallel = %d, n_kv_req = %d\n", __func__, n_len, n_ctx, ctx_params.n_batch, n_parallel, n_kv_req);

// 确保 KV 缓存足够大，能够容纳所有的提示和生成的标记
if (n_kv_req > n_ctx) {
    LOG_TEE("%s: error: n_kv_req (%d) > n_ctx, the required KV cache size is not big enough\n", __func__,  n_kv_req);
    LOG_TEE("%s:        either reduce n_parallel or increase n_ctx\n", __func__);
    return 1;
}

// 逐个打印提示标记
    // 输出一个换行符
    fprintf(stderr, "\n");

    // 遍历 tokens_list 中的每个 id，并将其对应的字符串输出到 stderr
    for (auto id : tokens_list) {
        fprintf(stderr, "%s", llama_token_to_piece(ctx, id).c_str());
    }

    // 刷新 stderr 缓冲区
    fflush(stderr);

    // 创建一个 llama_batch 对象
    // 用于提交 token 数据进行解码
    llama_batch batch = llama_batch_init(std::max(tokens_list.size(), (size_t)n_parallel), 0, 1);

    // 评估初始提示
    for (size_t i = 0; i < tokens_list.size(); ++i) {
        // 向 batch 中添加 token 数据
        llama_batch_add(batch, tokens_list[i], i, { 0 }, false);
    }
    // 断言 batch 中的 token 数量与 tokens_list 中的数量相同
    GGML_ASSERT(batch.n_tokens == (int) tokens_list.size());

    // llama_decode 仅会为提示的最后一个 token 输出 logits
    batch.logits[batch.n_tokens - 1] = true;
    // 如果 llama_decode() 函数返回非零值，表示解码失败，记录日志并返回1
    if (llama_decode(ctx, batch) != 0) {
        LOG_TEE("%s: llama_decode() failed\n", __func__);
        return 1;
    }

    // 将系统 KV 缓存分配给所有并行序列
    // 这样，并行序列将“重用”提示标记，而无需复制它们
    for (int32_t i = 1; i < n_parallel; ++i) {
        llama_kv_cache_seq_cp(ctx, 0, i, 0, batch.n_tokens);
    }

    // 如果并行度大于1，记录生成序列的日志
    if (n_parallel > 1) {
        LOG_TEE("\n\n%s: generating %d sequences ...\n", __func__, n_parallel);
    }

    // 主循环

    // 我们将把并行解码的序列存储在这个向量中
    std::vector<std::string> streams(n_parallel);
    // 记录每个并行序列的最后一个标记的批次索引
    // 我们需要这个来确定从哪些logits中进行采样
    std::vector<int32_t> i_batch(n_parallel, batch.n_tokens - 1);

    int n_cur    = batch.n_tokens;  // 当前标记数
    int n_decode = 0;  // 解码的标记数

    const auto t_main_start = ggml_time_us();  // 记录主循环开始时间

    while (n_cur <= n_len) {  // 当前标记数小于等于总标记数时执行循环
        // 准备下一个批次
        llama_batch_clear(batch);

        // 为每个并行序列/流采样下一个标记
        for (int32_t i = 0; i < n_parallel; ++i) {
            if (i_batch[i] < 0) {
                // 流已经结束
                continue;
            }
// 获取模型的词汇量
auto n_vocab = llama_n_vocab(model);

// 获取第 i_batch[i] 批次的logits
auto * logits = llama_get_logits_ith(ctx, i_batch[i]);

// 创建候选词向量，预留空间
std::vector<llama_token_data> candidates;
candidates.reserve(n_vocab);

// 遍历词汇量，将每个词的信息加入候选词向量
for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
    candidates.emplace_back(llama_token_data{ token_id, logits[token_id], 0.0f });
}

// 创建候选词向量的指针数组
llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };

// 设置取样的参数
const int top_k = 40;
const float top_p = 0.9f;
const float temp = 0.4f;

// 使用 top_k 算法进行取样
llama_sample_top_k(ctx, &candidates_p, top_k, 1);

// 使用 top_p 算法进行取样
llama_sample_top_p(ctx, &candidates_p, top_p, 1);

// 使用温度参数进行取样
llama_sample_temp(ctx, &candidates_p, temp);
// 使用llama_sample_token函数从候选项中获取新的token_id
const llama_token new_token_id = llama_sample_token(ctx, &candidates_p);

// 如果是流的结束，标记流为已完成
if (new_token_id == llama_token_eos(model) || n_cur == n_len) {
    i_batch[i] = -1;
    LOG_TEE("\n");
    if (n_parallel > 1) {
        LOG_TEE("%s: stream %d finished at n_cur = %d", __func__, i, n_cur);
    }
    continue;
}

// 如果只有一个流，立即打印到标准输出
if (n_parallel == 1) {
    LOG_TEE("%s", llama_token_to_piece(ctx, new_token_id).c_str());
    fflush(stdout);
}
            }

            // 将新的令牌转换为片段，并添加到对应的流中
            streams[i] += llama_token_to_piece(ctx, new_token_id);

            // 更新批次中每个流的令牌数量
            i_batch[i] = batch.n_tokens;

            // 将新的令牌添加到下一次评估中
            llama_batch_add(batch, new_token_id, n_cur, { i }, true);

            // 增加解码计数
            n_decode += 1;
        }

        // 所有流都已完成
        if (batch.n_tokens == 0) {
            break;
        }

        // 更新当前位置
        n_cur += 1;

        // 使用变换器模型评估当前批次
    // 如果 llama_decode 函数返回非零值，打印错误信息并返回 1
    if (llama_decode(ctx, batch)) {
        fprintf(stderr, "%s : failed to eval, return code %d\n", __func__, 1);
        return 1;
    }

    // 打印换行符
    LOG_TEE("\n");

    // 如果并行数大于 1，打印换行符并遍历并行序列
    if (n_parallel > 1) {
        LOG_TEE("\n");

        for (int32_t i = 0; i < n_parallel; ++i) {
            // 打印并行序列的信息
            LOG_TEE("sequence %d:\n\n%s%s\n\n", i, params.prompt.c_str(), streams[i].c_str());
        }
    }

    // 计算主函数结束时间
    const auto t_main_end = ggml_time_us();

    // 打印解码信息
    LOG_TEE("%s: decoded %d tokens in %.2f s, speed: %.2f t/s\n",
            __func__, n_decode, (t_main_end - t_main_start) / 1000000.0f, n_decode / ((t_main_end - t_main_start) / 1000000.0f));
    # 打印程序执行时间
    llama_print_timings(ctx);
    
    # 打印错误信息到标准错误流
    fprintf(stderr, "\n");
    
    # 释放批处理对象占用的内存
    llama_batch_free(batch);
    
    # 释放上下文对象占用的内存
    llama_free(ctx);
    
    # 释放模型对象占用的内存
    llama_free_model(model);
    
    # 释放后端资源
    llama_backend_free();
    
    # 返回 0 表示程序正常结束
    return 0;
}
```