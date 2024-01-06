# `PowerInfer\examples\embedding\embedding.cpp`

```
#include "common.h"
#include "llama.h"

#include <ctime>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 主函数
int main(int argc, char ** argv) {
    // 定义参数对象
    gpt_params params;

    // 解析命令行参数，如果解析失败则返回1
    if (!gpt_params_parse(argc, argv, params)) {
        return 1;
    }

    // 设置参数中的嵌入标志为true
    params.embedding = true;

    // 打印构建信息
    print_build_info();

    # 如果参数中的种子值等于默认种子值，则将种子值设置为当前时间
    if (params.seed == LLAMA_DEFAULT_SEED) {
        params.seed = time(NULL);
    }

    # 打印种子值
    fprintf(stderr, "%s: seed  = %u\n", __func__, params.seed);

    # 使用种子值初始化随机数生成器
    std::mt19937 rng(params.seed);
    
    # 如果参数中包含随机提示的标志，则生成随机提示
    if (params.random_prompt) {
        params.prompt = gpt_random_prompt(rng);
    }

    # 初始化 LLAMA 后端
    llama_backend_init(params.numa);

    # 声明 LLAMA 模型和上下文
    llama_model * model;
    llama_context * ctx;

    # 从 GPT 参数中初始化模型
    std::tie(model, ctx) = llama_init_from_gpt_params(params);
    
    # 如果无法加载模型，则打印错误信息
    if (model == NULL) {
        fprintf(stderr, "%s: error: unable to load model\n", __func__);
    // 返回整数1
    return 1;
}

// 获取模型的训练上下文数量
const int n_ctx_train = llama_n_ctx_train(model);
// 获取当前上下文数量
const int n_ctx = llama_n_ctx(ctx);

// 如果当前上下文数量大于训练上下文数量，则打印警告信息
if (n_ctx > n_ctx_train) {
    fprintf(stderr, "%s: warning: model was trained on only %d context tokens (%d specified)\n",
            __func__, n_ctx_train, n_ctx);
}

// 打印系统信息
{
    fprintf(stderr, "\n");
    fprintf(stderr, "%s\n", get_system_info(params).c_str());
}

// 初始化过去上下文数量
int n_past = 0;

// 对提示进行标记化处理
    // 使用 llama_tokenize 函数对输入的文本进行分词处理，返回分词结果
    auto embd_inp = ::llama_tokenize(ctx, params.prompt, true);

    // 如果需要输出详细的提示信息
    if (params.verbose_prompt) {
        // 输出换行符
        fprintf(stderr, "\n");
        // 输出函数名称和输入的提示文本
        fprintf(stderr, "%s: prompt: '%s'\n", __func__, params.prompt.c_str());
        // 输出函数名称和提示文本中的分词数量
        fprintf(stderr, "%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
        // 遍历分词结果，输出每个分词的索引和内容
        for (int i = 0; i < (int) embd_inp.size(); i++) {
            fprintf(stderr, "%6d -> '%s'\n", embd_inp[i], llama_token_to_piece(ctx, embd_inp[i]).c_str());
        }
        // 输出换行符
        fprintf(stderr, "\n");
    }

    // 如果分词结果的数量超过了指定的上下文窗口大小
    if (embd_inp.size() > (size_t)n_ctx) {
        // 输出错误信息并返回错误代码
        fprintf(stderr, "%s: error: prompt is longer than the context window (%zu tokens, n_ctx = %d)\n",
                __func__, embd_inp.size(), n_ctx);
        return 1;
    }

    // 当分词结果不为空时
    while (!embd_inp.empty()) {
        // 计算需要处理的分词数量，取分词数量和指定批处理数量的最小值
        int n_tokens = std::min(params.n_batch, (int) embd_inp.size());
    // 如果 llama_decode 函数返回非零值，表示解码失败
    if (llama_decode(ctx, llama_batch_get_one(embd_inp.data(), n_tokens, n_past, 0))) {
        // 打印错误信息
        fprintf(stderr, "%s : failed to eval\n", __func__);
        // 返回 1，表示程序执行失败
        return 1;
    }
    // 更新 n_past 变量
    n_past += n_tokens;
    // 删除 embd_inp 中前 n_tokens 个元素
    embd_inp.erase(embd_inp.begin(), embd_inp.begin() + n_tokens);
    // 获取模型的嵌入维度
    const int n_embd = llama_n_embd(model);
    // 获取模型的嵌入
    const auto * embeddings = llama_get_embeddings(ctx);
    // 遍历并打印模型的嵌入
    for (int i = 0; i < n_embd; i++) {
        printf("%f ", embeddings[i]);
    }
    // 打印换行符
    printf("\n");
    // 打印 llama 的时间统计信息
    llama_print_timings(ctx);
    // 释放 llama 上下文
    llama_free(ctx);
    // 释放模型
    llama_free_model(model);
# 释放后端资源
llama_backend_free();
# 返回 0，表示程序正常结束
return 0;
```