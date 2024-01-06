# `PowerInfer\examples\save-load-state\save-load-state.cpp`

```
#include "common.h" // 包含 common.h 头文件
#include "llama.h" // 包含 llama.h 头文件

#include <vector> // 包含 vector 头文件
#include <cstdio> // 包含 cstdio 头文件
#include <chrono> // 包含 chrono 头文件

int main(int argc, char ** argv) { // 主函数，接受命令行参数
    gpt_params params; // 声明 gpt_params 结构体变量 params

    params.prompt = "The quick brown fox"; // 设置 params 结构体变量 prompt 字段的值为 "The quick brown fox"

    if (!gpt_params_parse(argc, argv, params)) { // 调用 gpt_params_parse 函数解析命令行参数，如果失败则返回 1
        return 1;
    }

    print_build_info(); // 调用 print_build_info 函数打印构建信息

    if (params.n_predict < 0) { // 如果 params 结构体变量 n_predict 字段的值小于 0
        params.n_predict = 16; // 将 params 结构体变量 n_predict 字段的值设置为 16
    }

    auto n_past = 0; // 初始化变量 n_past 为 0

    std::string result0; // 声明一个空字符串变量 result0
    std::string result1; // 声明一个空字符串变量 result1

    // 初始化 llama_model 和 llama_context
    llama_model * model;
    llama_context * ctx;

    // 从 GPT 参数初始化 llama_model 和 llama_context
    std::tie(model, ctx) = llama_init_from_gpt_params(params);
    // 如果初始化失败，打印错误信息并返回 1
    if (model == nullptr || ctx == nullptr) {
        fprintf(stderr, "%s : failed to init\n", __func__);
        return 1;
    }

    // 对提示进行分词
    auto tokens = llama_tokenize(ctx, params.prompt, true);
    // 评估提示
    llama_decode(ctx, llama_batch_get_one(tokens.data(), tokens.size(), n_past, 0));
    n_past += tokens.size();

    // 将状态（rng、logits、embedding和kv_cache）保存到文件
    {
        // 创建存储状态数据的内存空间
        std::vector<uint8_t> state_mem(llama_get_state_size(ctx));

        {
            // 打开文件以便写入
            FILE *fp_write = fopen("dump_state.bin", "wb");
            // 将状态数据复制到内存中
            llama_copy_state_data(ctx, state_mem.data()); // 也可以直接复制到内存映射文件
            // 将内存中的状态数据写入文件
            fwrite(state_mem.data(), 1, state_mem.size(), fp_write);
            // 关闭文件
            fclose(fp_write);
        }
    }

    // 保存状态（最后的标记）
    const auto n_past_saved = n_past;

    // 第一次运行
    # 打印提示信息
    printf("\nfirst run: %s", params.prompt.c_str());

    # 循环预测次数
    for (auto i = 0; i < params.n_predict; i++) {
        # 获取模型的输出logits
        auto * logits = llama_get_logits(ctx);
        # 获取词汇表大小
        auto n_vocab = llama_n_vocab(model);

        # 创建候选词列表
        std::vector<llama_token_data> candidates;
        candidates.reserve(n_vocab);
        # 遍历词汇表，将每个词的信息加入候选词列表
        for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
            candidates.emplace_back(llama_token_data{token_id, logits[token_id], 0.0f});
        }
        # 将候选词列表转换为候选词数组
        llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
        # 从候选词数组中采样下一个词
        auto next_token = llama_sample_token(ctx, &candidates_p);
        # 将下一个词转换为字符串
        auto next_token_str = llama_token_to_piece(ctx, next_token);

        # 打印下一个词
        printf("%s", next_token_str.c_str());
        # 将下一个词添加到结果中
        result0 += next_token_str;

        # 解码下一个词
        if (llama_decode(ctx, llama_batch_get_one(&next_token, 1, n_past, 0))) {
            # 打印错误信息
            fprintf(stderr, "\n%s : failed to evaluate\n", __func__);
    // 释放上下文资源
    llama_free(ctx);
    // 释放模型资源
    llama_free_model(model);
    // 返回值为1
    return 1;
}
// 增加过去的步数
n_past += 1;
}

// 打印换行
printf("\n\n");

// 释放旧的上下文资源
llama_free(ctx);

// 创建新的上下文
auto * ctx2 = llama_new_context_with_model(model, llama_context_params_from_gpt_params(params));

// 打印第二次运行的提示信息
printf("\nsecond run: %s", params.prompt.c_str());

// 从文件中加载状态（rng、logits、embedding和kv_cache）
{
    // 创建存储状态的内存空间
    std::vector<uint8_t> state_mem(llama_get_state_size(ctx2));
// 打开名为 "dump_state.bin" 的文件，以只读方式
FILE * fp_read = fopen("dump_state.bin", "rb");

// 从文件中读取数据到 state_mem 中，返回实际读取的字节数
const size_t ret = fread(state_mem.data(), 1, state_mem.size(), fp_read);
if (ret != state_mem.size()) {
    // 如果读取的字节数与 state_mem 的大小不一致，则输出错误信息并释放内存后返回 1
    fprintf(stderr, "\n%s : failed to read state\n", __func__);
    llama_free(ctx2);
    llama_free_model(model);
    return 1;
}

// 将读取的状态数据设置到 ctx2 中
llama_set_state_data(ctx2, state_mem.data());

// 关闭文件流
fclose(fp_read);
}

// 恢复保存的状态（最后的标记）
n_past = n_past_saved;

// 第二次运行
    // 循环预测参数中指定次数
    for (auto i = 0; i < params.n_predict; i++) {
        // 获取模型的输出logits
        auto * logits = llama_get_logits(ctx2);
        // 获取词汇表大小
        auto n_vocab = llama_n_vocab(model);
        // 创建候选词数组
        std::vector<llama_token_data> candidates;
        candidates.reserve(n_vocab);
        // 遍历词汇表，将每个词的id、logits值和概率为0.0的候选词数据加入数组
        for (llama_token token_id = 0; token_id < n_vocab; token_id++) {
            candidates.emplace_back(llama_token_data{token_id, logits[token_id], 0.0f});
        }
        // 将候选词数组转换为候选词数据结构
        llama_token_data_array candidates_p = { candidates.data(), candidates.size(), false };
        // 从候选词中采样出下一个词的id
        auto next_token = llama_sample_token(ctx2, &candidates_p);
        // 将下一个词的id转换为对应的字符串
        auto next_token_str = llama_token_to_piece(ctx2, next_token);

        // 打印下一个词的字符串形式
        printf("%s", next_token_str.c_str());
        // 将下一个词的字符串形式添加到结果字符串中
        result1 += next_token_str;

        // 如果解码失败，则打印错误信息并返回1
        if (llama_decode(ctx2, llama_batch_get_one(&next_token, 1, n_past, 0))) {
            fprintf(stderr, "\n%s : failed to evaluate\n", __func__);
            llama_free(ctx2);
            llama_free_model(model);
            return 1;
    }
    // 增加 n_past 变量的值
    n_past += 1;
}

// 打印换行符
printf("\n");

// 释放内存
llama_free(ctx2);
llama_free_model(model);

// 如果 result0 不等于 result1，则打印错误信息并返回 1
if (result0 != result1) {
    fprintf(stderr, "\n%s : error : the 2 generations are different\n", __func__);
    return 1;
}

// 打印成功信息
fprintf(stderr, "\n%s : success\n", __func__);

// 返回 0
return 0;
}
```