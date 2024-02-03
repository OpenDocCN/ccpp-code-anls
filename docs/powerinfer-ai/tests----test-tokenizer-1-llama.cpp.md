# `PowerInfer\tests\test-tokenizer-1-llama.cpp`

```cpp
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

// 主函数
int main(int argc, char **argv) {
    // 检查参数个数是否符合要求
    if (argc < 2) {
        // 打印用法信息
        fprintf(stderr, "Usage: %s <vocab-file>\n", argv[0]);
        return 1;
    }

    // 获取文件名
    const std::string fname = argv[1];

    // 打印读取词汇表的信息
    fprintf(stderr, "%s : reading vocab from: '%s'\n", __func__, fname.c_str());

    llama_model * model;
    llama_context * ctx;

    // 初始化 Llama 后端
    llama_backend_init(false);

    // 加载词汇表
    {
        // 获取默认的模型参数
        auto mparams = llama_model_default_params();

        // 设置只加载词汇表
        mparams.vocab_only = true;

        // 从文件加载模型
        model = llama_load_model_from_file(fname.c_str(), mparams);

        // 检查模型加载是否成功
        if (model == NULL) {
            fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
            return 1;
        }

        // 获取默认的上下文参数
        auto cparams = llama_context_default_params();

        // 创建带有模型的新上下文
        ctx = llama_new_context_with_model(model, cparams);

        // 检查上下文创建是否成功
        if (ctx == NULL) {
            fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
            llama_free_model(model);
            return 1;
        }
    }

    // 断言词汇表类型为 SPM
    GGML_ASSERT(llama_vocab_type(model) == LLAMA_VOCAB_TYPE_SPM);

#ifdef _WIN32
    // 为了支持 Unicode 控制台，需要进行初始化
    console::init(false, false);
    // 在程序退出时进行清理
    atexit([]() { console::cleanup(); });
#endif

    // 获取词汇表大小
    const int n_vocab = llama_n_vocab(model);
}
    // 遍历词汇表中的每个索引，将索引转换为字符串
    for (int i = 0; i < n_vocab; ++i) {
        // 使用索引生成对应的字符串
        std::string str = llama_detokenize_spm(ctx, std::vector<int>(1, i));
        // 对字符串进行分词处理，生成 token 列表
        std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
        // 将 token 列表还原为字符串
        std::string check = llama_detokenize_spm(ctx, tokens);
        // 检查还原后的字符串是否与原始字符串相同，如果不同则输出错误信息并返回 2
        if (check != str) {
            fprintf(stderr, "%s : error: token %d detokenizes to '%s'(%zu) but tokenization of this detokenizes to '%s'(%zu)\n",
                __func__, i, str.c_str(), str.length(), check.c_str(), check.length());
            return 2;
        }
    }

    // 遍历 Unicode 编码点范围
    for (uint32_t cp = 0x0000; cp < 0xffff; ++cp) {
        // 对每个编码点生成对应的 UTF-8 编码字符串
        if (cp < 0xd800 || cp > 0xdfff) {
            std::string str = codepoint_to_utf8(cp);
            // 对 UTF-8 编码字符串进行分词处理，生成 token 列表
            std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
            // 将 token 列表还原为字符串
            std::string check = llama_detokenize_spm(ctx, tokens);
            // 检查还原后的字符串是否与原始字符串相同，如果不同则输出错误信息并返回 3
            if (cp != 9601 && str != check) {
                fprintf(stderr, "%s : error: codepoint %d detokenizes to '%s'(%zu) instead of '%s'(%zu)\n",
                    __func__, cp, check.c_str(), check.length(), str.c_str(), str.length());
                return 3;
            }
        }
    }
    // 遍历 Unicode 补充编码点范围
    for (uint32_t cp = 0x10000; cp < 0x0010ffff; ++cp) {
        // 对每个编码点生成对应的 UTF-8 编码字符串
        std::string str = codepoint_to_utf8(cp);
        // 对 UTF-8 编码字符串进行分词处理，生成 token 列表
        std::vector<llama_token> tokens = llama_tokenize(ctx, str, false);
        // 将 token 列表还原为字符串
        std::string check = llama_detokenize_spm(ctx, tokens);
        // 检查还原后的字符串是否与原始字符串相同，如果不同则输出错误信息并返回 4
        if (str != check) {
            fprintf(stderr, "%s : error: codepoint %d detokenizes to '%s'(%zu) instead of '%s'(%zu)\n",
                __func__, cp, check.c_str(), check.length(), str.c_str(), str.length());
            return 4;
        }
    }

    // 释放模型资源
    llama_free_model(model);
    // 释放上下文资源
    llama_free(ctx);
    // 释放后端资源
    llama_backend_free();

    // 返回成功状态码
    return 0;
# 闭合前面的函数定义
```