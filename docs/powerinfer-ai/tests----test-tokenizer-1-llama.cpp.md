# `PowerInfer\tests\test-tokenizer-1-llama.cpp`

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

    // 使用默认参数创建 Llama 上下文
    auto cparams = llama_context_default_params();

    // 使用模型和参数创建 Llama 上下文
    ctx = llama_new_context_with_model(model, cparams);

    // 如果上下文创建失败，输出错误信息并释放模型资源，然后返回错误码
    if (ctx == NULL) {
        fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
        llama_free_model(model);
        return 1;
    }
}

// 断言模型的词汇类型为 SPM
GGML_ASSERT(llama_vocab_type(model) == LLAMA_VOCAB_TYPE_SPM);

#ifdef _WIN32
    // 为了支持 Unicode 控制台，需要进行初始化
    console::init(false, false);
    // 在程序退出时，进行控制台的清理工作
    atexit([]() { console::cleanup(); });
#endif
// 获取词汇表的大小
const int n_vocab = llama_n_vocab(model);

// 遍历词汇表
for (int i = 0; i < n_vocab; ++i) {
    // 将单词索引转换为字符串
    std::string str = llama_detokenize_spm(ctx, std::vector<int>(1, i));
    // 对字符串进行分词
    std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
    // 将分词结果转换为字符串
    std::string check = llama_detokenize_spm(ctx, tokens);
    // 检查分词和反分词结果是否一致
    if (check != str) {
        // 输出错误信息并返回错误代码
        fprintf(stderr, "%s : error: token %d detokenizes to '%s'(%zu) but tokenization of this detokenizes to '%s'(%zu)\n",
            __func__, i, str.c_str(), str.length(), check.c_str(), check.length());
        return 2;
    }
}

// 遍历 Unicode 码点
for (uint32_t cp = 0x0000; cp < 0xffff; ++cp) {
    // 排除代理对范围的码点
    if (cp < 0xd800 || cp > 0xdfff) {
        // 将码点转换为 UTF-8 编码的字符串
        std::string str = codepoint_to_utf8(cp);
        // 对字符串进行分词
        std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
        // 将分词结果转换为字符串
        std::string check = llama_detokenize_spm(ctx, tokens);
        // 检查分词和反分词结果是否一致
        if (cp != 9601 && str != check) {
    // 循环遍历从 0x10000 到 0x0010ffff 的 Unicode 码点
    for (uint32_t cp = 0x10000; cp < 0x0010ffff; ++cp) {
        // 将 Unicode 码点转换为 UTF-8 编码的字符串
        std::string str = codepoint_to_utf8(cp);
        // 使用 llama_tokenize 函数对字符串进行分词，返回分词结果
        std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
        // 使用 llama_detokenize_spm 函数对分词结果进行反分词，返回反分词结果
        std::string check = llama_detokenize_spm(ctx, tokens);
        // 如果反分词结果与原始字符串不相等，则输出错误信息并返回错误码
        if (str != check) {
            fprintf(stderr, "%s : error: codepoint %d detokenizes to '%s'(%zu) instead of '%s'(%zu)\n",
                __func__, cp, check.c_str(), check.length(), str.c_str(), str.length());
            return 4;
        }
    }

    // 释放模型和上下文资源
    llama_free_model(model);
    llama_free(ctx);
    llama_backend_free();  // 调用 llama_backend_free() 函数，释放后端资源

    return 0;  // 返回值为 0，表示程序正常结束
}
```