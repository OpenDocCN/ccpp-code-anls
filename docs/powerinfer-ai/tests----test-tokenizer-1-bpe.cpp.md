# `PowerInfer\tests\test-tokenizer-1-bpe.cpp`

```
#include "llama.h" 
#include "common.h" 
#include "unicode.h" 
#include "console.h"

#include <cassert> 
#include <cstdio> 
#include <cstring> 
#include <string> 
#include <codecvt> 
#include <map> 
#include <vector> 
#include <locale>

int main(int argc, char **argv) {
    // 检查命令行参数是否足够
    if (argc < 2) {
        // 打印用法信息到标准错误输出
        fprintf(stderr, "Usage: %s <vocab-file>\n", argv[0]);
        // 返回错误码
        return 1;
    }
    // 从命令行参数中获取文件名
    const std::string fname = argv[1];

    // 打印读取词汇表的信息
    fprintf(stderr, "%s : reading vocab from: '%s'\n", __func__, fname.c_str());

    // 声明指向 llama_model 和 llama_context 的指针
    llama_model * model;
    llama_context * ctx;

    // 初始化 llama 后端
    llama_backend_init(false);

    // 加载词汇表
    {
        // 获取默认的模型参数
        auto mparams = llama_model_default_params();

        // 设置只加载词汇表的选项为 true
        mparams.vocab_only = true;

        // 从文件中加载模型
        model = llama_load_model_from_file(fname.c_str(), mparams);

        // 如果加载失败，打印错误信息并返回 1
        if (model == NULL) {
            fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
            return 1;
    }

    // 使用默认参数创建 llama 上下文
    auto cparams = llama_context_default_params();

    // 使用模型和参数创建 llama 上下文
    ctx = llama_new_context_with_model(model, cparams);

    // 如果上下文创建失败，则输出错误信息并释放模型资源
    if (ctx == NULL) {
        fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
        llama_free_model(model);
        return 1;
    }
}

// 确保模型的词汇类型为 BPE
GGML_ASSERT(llama_vocab_type(model) == LLAMA_VOCAB_TYPE_BPE);

#ifdef _WIN32
    // 为了支持 Unicode 控制台，需要进行初始化
    console::init(false, false);
    // 在程序退出时进行清理
    atexit([]() { console::cleanup(); });
#endif
    // 获取模型的词汇量大小
    const int n_vocab = llama_n_vocab(model);

    // 遍历词汇量中的每个词
    for (int i = 0; i < n_vocab; ++i) {
        // 从词汇索引解码出对应的字符串
        std::string str = llama_detokenize_bpe(ctx, std::vector<int>(1, i));
        try {
            // 将字符串转换为 Unicode 码点
            auto cps = codepoints_from_utf8(str);
            // 对字符串进行分词
            std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
            // 将分词结果重新组合成字符串
            std::string check = llama_detokenize_bpe(ctx, tokens);
            // 检查重新组合的字符串是否与原始字符串相同
            if (check != str) {
                // 如果不同，输出错误信息并返回错误码
                fprintf(stderr, "%s : error: token %d detokenizes to '%s'(%zu) but tokenization of this detokenizes to '%s'(%zu)\n",
                    __func__, i, str.c_str(), str.length(), check.c_str(), check.length());
                return 2;
            }
        }
        // 捕获无效的参数异常
        catch (const std::invalid_argument &) {
            // 输出信息表示无法进行 UTF-8 转换
            fprintf(stderr, "%s : info: utf8 conversion %d '%s'\n", __func__, i, str.c_str());
        }
    }
    // 循环遍历从 0x0000 到 0xffff 的 Unicode 码点
    for (uint32_t cp = 0x0000; cp < 0xffff; ++cp) {
        // 注意：这些异常似乎是必要的，因为 GPT2 分词器不希望干扰一些 ASCII 控制字符
        if ((cp < 0x03 || cp > 0x05) && cp != 0x0b && cp != 0x11 && (cp < 0x13 || cp > 0x17) && cp != 0x19 && (cp < 0x1c || cp > 0x1e) && (cp < 0xd800 || cp > 0xdfff)) {
            // 将 Unicode 码点转换成 UTF-8 编码的字符串
            std::string str = " " + codepoint_to_utf8(cp);
            // 使用 llama_tokenize 函数对字符串进行分词
            std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
            // 使用 llama_detokenize_bpe 函数对分词结果进行合并
            std::string check = llama_detokenize_bpe(ctx, tokens);
            // 检查合并后的字符串是否与原始字符串相同，如果不同则输出错误信息并返回 3
            if (str != check) {
                fprintf(stderr, "%s : error: codepoint %x detokenizes to '%s'(%zu) instead of '%s'(%zu)\n",
                    __func__, cp, check.c_str(), check.length(), str.c_str(), str.length());
                return 3;
            }
        }
    }
    // 限制在已分配的 Unicode 平面范围内
    // for (uint32_t cp = 0x10000; cp < 0x0010ffff; ++cp) {
    // 循环遍历从 0x10000 到 0x00040000 的 Unicode 码点
    for (uint32_t cp = 0x10000; cp < 0x00040000; ++cp) {
        // 将 Unicode 码点转换成 UTF-8 编码的字符串
        std::string str = codepoint_to_utf8(cp);
        // 使用 llama_tokenize 函数对字符串进行分词
        std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
        // 使用 llama_detokenize_bpe 函数对分词结果进行合并
        std::string check = llama_detokenize_bpe(ctx, tokens);
        // 检查合并后的字符串是否与原始字符串相同，如果不同则输出错误信息并返回 3
        if (str != check) {
    // 循环遍历从 0x000e0000 到 0x0010ffff 的 Unicode 编码点
    for (uint32_t cp = 0x000e0000; cp < 0x0010ffff; ++cp) {
        // 将 Unicode 编码点转换为 UTF-8 编码的字符串
        std::string str = codepoint_to_utf8(cp);
        // 使用 llama_tokenize 函数对字符串进行分词，返回 token 列表
        std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
        // 使用 llama_detokenize_bpe 函数对 token 列表进行反分词，返回原始字符串
        std::string check = llama_detokenize_bpe(ctx, tokens);
        // 检查反分词后的字符串是否与原始字符串相同，如果不同则输出错误信息并返回 4
        if (str != check) {
            fprintf(stderr, "%s : error: codepoint %x detokenizes to '%s'(%zu) instead of '%s'(%zu)\n",
                __func__, cp, check.c_str(), check.length(), str.c_str(), str.length());
            return 4;
        }
    }
    // 释放模型和上下文资源
    llama_free_model(model);
    llama_free(ctx);
    // 释放 llama 后端资源
    llama_backend_free();
# 返回整数值0，结束函数的执行。
```