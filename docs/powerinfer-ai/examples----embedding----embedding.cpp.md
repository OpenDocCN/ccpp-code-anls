# `PowerInfer\examples\embedding\embedding.cpp`

```
#include "common.h"
#include "llama.h"

#include <ctime>

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

int main(int argc, char ** argv) {
    gpt_params params; // 定义参数对象

    if (!gpt_params_parse(argc, argv, params)) { // 解析命令行参数到参数对象
        return 1; // 如果解析失败，返回错误码1
    }

    params.embedding = true; // 设置参数对象的嵌入标志为true

    print_build_info(); // 打印构建信息

    if (params.seed == LLAMA_DEFAULT_SEED) { // 如果参数对象的种子值为默认值
        params.seed = time(NULL); // 设置种子值为当前时间
    }

    fprintf(stderr, "%s: seed  = %u\n", __func__, params.seed); // 打印种子值

    std::mt19937 rng(params.seed); // 使用种子值初始化随机数生成器
    if (params.random_prompt) { // 如果参数对象的随机提示标志为true
        params.prompt = gpt_random_prompt(rng); // 生成随机提示
    }

    llama_backend_init(params.numa); // 初始化LLAMA后端

    llama_model * model; // 定义LLAMA模型指针
    llama_context * ctx; // 定义LLAMA上下文指针

    // 加载模型
    std::tie(model, ctx) = llama_init_from_gpt_params(params); // 从参数对象初始化LLAMA模型和上下文
    if (model == NULL) { // 如果模型为空
        fprintf(stderr, "%s: error: unable to load model\n", __func__); // 打印错误信息
        return 1; // 返回错误码1
    }

    const int n_ctx_train = llama_n_ctx_train(model); // 获取模型的训练上下文数
    const int n_ctx = llama_n_ctx(ctx); // 获取上下文的上下文数

    if (n_ctx > n_ctx_train) { // 如果上下文数大于训练上下文数
        fprintf(stderr, "%s: warning: model was trained on only %d context tokens (%d specified)\n",
                __func__, n_ctx_train, n_ctx); // 打印警告信息
    }

    // 打印系统信息
    {
        fprintf(stderr, "\n"); // 打印换行
        fprintf(stderr, "%s\n", get_system_info(params).c_str()); // 打印系统信息
    }

    int n_past = 0; // 初始化过去上下文数为0

    // 对提示进行标记化
    auto embd_inp = ::llama_tokenize(ctx, params.prompt, true); // 使用LLAMA对提示进行标记化

    if (params.verbose_prompt) { // 如果参数对象的详细提示标志为true
        fprintf(stderr, "\n"); // 打印换行
        fprintf(stderr, "%s: prompt: '%s'\n", __func__, params.prompt.c_str()); // 打印提示信息
        fprintf(stderr, "%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size()); // 打印提示中的标记数
        for (int i = 0; i < (int) embd_inp.size(); i++) { // 遍历标记
            fprintf(stderr, "%6d -> '%s'\n", embd_inp[i], llama_token_to_piece(ctx, embd_inp[i]).c_str()); // 打印标记和对应的片段
        }
        fprintf(stderr, "\n"); // 打印换行
    }
}
    # 如果输入的嵌入向量大小超过了上下文窗口的大小，则输出错误信息并返回1
    if (embd_inp.size() > (size_t)n_ctx) {
        fprintf(stderr, "%s: error: prompt is longer than the context window (%zu tokens, n_ctx = %d)\n",
                __func__, embd_inp.size(), n_ctx);
        return 1;
    }

    # 当嵌入向量不为空时，循环执行以下操作
    while (!embd_inp.empty()) {
        # 计算要处理的标记数量，取params.n_batch和embd_inp.size()的最小值
        int n_tokens = std::min(params.n_batch, (int) embd_inp.size());
        # 如果llama_decode函数执行失败，则输出错误信息并返回1
        if (llama_decode(ctx, llama_batch_get_one(embd_inp.data(), n_tokens, n_past, 0))) {
            fprintf(stderr, "%s : failed to eval\n", __func__);
            return 1;
        }
        # 更新n_past的值
        n_past += n_tokens;
        # 删除embd_inp中已处理的标记
        embd_inp.erase(embd_inp.begin(), embd_inp.begin() + n_tokens);
    }

    # 获取嵌入向量的数量
    const int n_embd = llama_n_embd(model);
    # 获取嵌入向量的指针
    const auto * embeddings = llama_get_embeddings(ctx);

    # 遍历嵌入向量并输出每个值
    for (int i = 0; i < n_embd; i++) {
        printf("%f ", embeddings[i]);
    }
    # 输出换行符
    printf("\n");

    # 打印llama的时间信息
    llama_print_timings(ctx);
    # 释放llama上下文
    llama_free(ctx);
    # 释放模型
    llama_free_model(model);

    # 释放llama后端资源
    llama_backend_free();

    # 返回0表示成功执行
    return 0;
# 闭合前面的函数定义
```