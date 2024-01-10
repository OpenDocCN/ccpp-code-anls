# `PowerInfer\examples\save-load-state\save-load-state.cpp`

```
#include "common.h"
#include "llama.h"

#include <vector>
#include <cstdio>
#include <chrono>

int main(int argc, char ** argv) {
    gpt_params params;  // 定义 gpt_params 结构体变量

    params.prompt = "The quick brown fox";  // 设置 params 结构体的 prompt 字段值为 "The quick brown fox"

    if (!gpt_params_parse(argc, argv, params)) {  // 调用 gpt_params_parse 函数解析命令行参数，如果失败则返回 1
        return 1;
    }

    print_build_info();  // 调用打印构建信息的函数

    if (params.n_predict < 0) {  // 如果 params 结构体的 n_predict 字段小于 0，则将其设置为 16
        params.n_predict = 16;
    }

    auto n_past = 0;  // 定义并初始化 n_past 变量为 0

    std::string result0;  // 定义 result0 字符串变量
    std::string result1;  // 定义 result1 字符串变量

    // init
    llama_model * model;  // 定义 llama_model 指针变量 model
    llama_context * ctx;  // 定义 llama_context 指针变量 ctx

    std::tie(model, ctx) = llama_init_from_gpt_params(params);  // 调用 llama_init_from_gpt_params 函数初始化 model 和 ctx
    if (model == nullptr || ctx == nullptr) {  // 如果初始化失败，则打印错误信息并返回 1
        fprintf(stderr, "%s : failed to init\n", __func__);
        return 1;
    }

    // tokenize prompt
    auto tokens = llama_tokenize(ctx, params.prompt, true);  // 对 prompt 进行分词处理，得到 tokens

    // evaluate prompt
    llama_decode(ctx, llama_batch_get_one(tokens.data(), tokens.size(), n_past, 0));  // 对 prompt 进行评估

    n_past += tokens.size();  // 更新 n_past 变量

    // save state (rng, logits, embedding and kv_cache) to file
    {
        std::vector<uint8_t> state_mem(llama_get_state_size(ctx));  // 定义存储状态的内存空间

        {
            FILE *fp_write = fopen("dump_state.bin", "wb");  // 打开文件 dump_state.bin 用于写入
            llama_copy_state_data(ctx, state_mem.data());  // 将状态数据复制到 state_mem 中
            fwrite(state_mem.data(), 1, state_mem.size(), fp_write);  // 将 state_mem 中的数据写入文件
            fclose(fp_write);  // 关闭文件
        }
    }

    // save state (last tokens)
    const auto n_past_saved = n_past;  // 保存当前的 n_past 值

    // first run
    printf("\nfirst run: %s", params.prompt.c_str());  // 打印提示信息
    // 循环预测下一个单词，循环次数由参数 n_predict 决定
    for (auto i = 0; i < params.n_predict; i++) {
        // 获取模型的输出概率
        auto * logits = llama_get_logits(ctx);
        // 获取词汇表的大小
        auto n_vocab = llama_n_vocab(model);

        // 创建候选词数组
        std::vector<llama_token_data> candidates;
        candidates.reserve(n_vocab);
        // 遍历词汇表，将每个词的信息加入候选词数组
        for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
            candidates.emplace_back(llama_token_data{token_id, logits[token_id], 0.0f});
        }
        // 将候选词数组转换为 C 风格的结构体
        llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
        // 从候选词中采样出下一个词的 token
        auto next_token = llama_sample_token(ctx, &candidates_p);
        // 将 token 转换为对应的字符串
        auto next_token_str = llama_token_to_piece(ctx, next_token);

        // 打印下一个词的字符串
        printf("%s", next_token_str.c_str());
        // 将下一个词的字符串加入结果字符串
        result0 += next_token_str;

        // 解码下一个词，更新模型状态
        if (llama_decode(ctx, llama_batch_get_one(&next_token, 1, n_past, 0))) {
            // 如果解码失败，打印错误信息，释放内存，返回错误码
            fprintf(stderr, "\n%s : failed to evaluate\n", __func__);
            llama_free(ctx);
            llama_free_model(model);
            return 1;
        }
        // 更新过去的词数
        n_past += 1;
    }

    // 打印换行符
    printf("\n\n");

    // 释放旧的上下文
    llama_free(ctx);

    // 创建新的上下文
    auto * ctx2 = llama_new_context_with_model(model, llama_context_params_from_gpt_params(params));

    // 打印提示信息
    printf("\nsecond run: %s", params.prompt.c_str());

    // 从文件中加载状态 (rng, logits, embedding and kv_cache)
    {
        // 创建存储状态的内存
        std::vector<uint8_t> state_mem(llama_get_state_size(ctx2));

        // 打开状态文件
        FILE * fp_read = fopen("dump_state.bin", "rb");

        // 从文件中读取状态数据
        const size_t ret = fread(state_mem.data(), 1, state_mem.size(), fp_read);
        // 如果读取的数据大小不符合预期，打印错误信息，释放内存，返回错误码
        if (ret != state_mem.size()) {
            fprintf(stderr, "\n%s : failed to read state\n", __func__);
            llama_free(ctx2);
            llama_free_model(model);
            return 1;
        }

        // 设置上下文的状态数据
        llama_set_state_data(ctx2, state_mem.data());

        // 关闭文件
        fclose(fp_read);
    }

    // 恢复状态 (上次的 tokens)
    n_past = n_past_saved;

    // 第二次运行
    // 遍历 params.n_predict 次，进行预测
    for (auto i = 0; i < params.n_predict; i++) {
        // 获取模型的输出 logits
        auto * logits = llama_get_logits(ctx2);
        // 获取词汇表大小
        auto n_vocab = llama_n_vocab(model);
        // 创建候选词数组
        std::vector<llama_token_data> candidates;
        candidates.reserve(n_vocab);
        // 遍历词汇表，将每个词的信息加入候选词数组
        for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
            candidates.emplace_back(llama_token_data{token_id, logits[token_id], 0.0f});
        }
        // 将候选词数组转换为 llama_token_data_array 结构
        llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
        // 从候选词中采样出下一个词的 token
        auto next_token = llama_sample_token(ctx2, &candidates_p);
        // 将下一个词的 token 转换为字符串形式
        auto next_token_str = llama_token_to_piece(ctx2, next_token);

        // 打印下一个词的字符串形式
        printf("%s", next_token_str.c_str());
        // 将下一个词的字符串形式加入结果字符串
        result1 += next_token_str;

        // 解码下一个词
        if (llama_decode(ctx2, llama_batch_get_one(&next_token, 1, n_past, 0))) {
            // 如果解码失败，打印错误信息，释放资源，返回错误码
            fprintf(stderr, "\n%s : failed to evaluate\n", __func__);
            llama_free(ctx2);
            llama_free_model(model);
            return 1;
        }
        // 更新过去词的数量
        n_past += 1;
    }

    // 打印换行符
    printf("\n");

    // 释放资源
    llama_free(ctx2);
    llama_free_model(model);

    // 检查两次生成的结果是否相同，如果不同，打印错误信息，返回错误码
    if (result0 != result1) {
        fprintf(stderr, "\n%s : error : the 2 generations are different\n", __func__);
        return 1;
    }

    // 打印成功信息
    fprintf(stderr, "\n%s : success\n", __func__);

    // 返回成功码
    return 0;
# 闭合前面的函数定义
```