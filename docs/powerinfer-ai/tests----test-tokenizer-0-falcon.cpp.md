# `PowerInfer\tests\test-tokenizer-0-falcon.cpp`

```
#include "llama.h"
#include "common.h"
#include "console.h"

#include <cstdio>
#include <string>
#include <map>
#include <vector>
#include <fstream>

// 生成使用 test-tokenizer-0-falcon.py 的测试数据
static const std::map<std::string, std::vector<llama_token>> & k_tests() {
    // 返回测试数据
    return _k_tests;
}

int main(int argc, char **argv) {
    // 检查参数数量是否正确
    if (argc < 2) {
        fprintf(stderr, "Usage: %s vocab-file [text-file]\n", argv[0]);
        return 1;
    }

    // 获取词汇文件名
    const std::string fname = argv[1];

    std::string fname_text;
    // 如果参数数量大于2，获取文本文件名
    if (argc > 2) {
        fname_text = argv[2];
    }

    // 打印读取词汇文件的信息
    fprintf(stderr, "%s : reading vocab from: '%s'\n", __func__, fname.c_str());

    llama_model * model;
    llama_context * ctx;

    // 初始化 LLAMA 后端
    llama_backend_init(false);

    // 加载词汇
    {
        auto mparams = llama_model_default_params();

        mparams.vocab_only = true;

        // 从文件加载词汇模型
        model = llama_load_model_from_file(fname.c_str(), mparams);

        // 检查词汇加载是否成功
        if (model == NULL) {
            fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
            return 1;
        }

        auto cparams = llama_context_default_params();

        // 创建包含词汇模型的上下文
        ctx = llama_new_context_with_model(model, cparams);

        // 检查上下文创建是否成功
        if (ctx == NULL) {
            fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
            llama_free_model(model);
            return 1;
        }
    }

    // 检查词汇类型是否为 BPE
    if (llama_vocab_type(model) != LLAMA_VOCAB_TYPE_BPE) {
        fprintf(stderr, "%s : error: vocab type is not BPE\n", __func__);
        llama_free_model(model);
        llama_free(ctx);
        return 2;
    }

#ifdef _WIN32
    // 为了支持 Unicode 控制台，需要进行初始化
    console::init(false, false);
    atexit([]() { console::cleanup(); });
#endif

    bool success = true;
}
    // 遍历测试用例集合，每个测试用例包含输入和期望输出
    for (const auto & test_kv : k_tests()) {
        // 使用 llama_tokenize 函数对输入进行分词处理，返回分词结果
        const std::vector<llama_token> res = llama_tokenize(ctx, test_kv.first, false);

        // 打印源字符串
        printf("\n");
        printf("src: '%s'\n", test_kv.first.c_str());
        // 打印分词结果
        printf("res: '%s'\n", llama_detokenize_bpe(ctx, res).c_str());
        // 打印分词结果的 token
        printf("tok: ");
        for (const auto & tok : res) {
            printf("%d ", tok);
        }
        printf("\n");

        // 检查分词结果的长度是否与期望输出的长度相同
        bool correct = res.size() == test_kv.second.size();

        // 遍历分词结果和期望输出，检查是否一致
        for (int i = 0; i < (int) res.size() && correct; ++i) {
            if (test_kv.second[i] != res[i]) {
                correct = false;
            }
        }

        // 如果分词结果与期望输出不一致，则输出错误信息，并将 success 标记为 false
        if (!correct) {
            fprintf(stderr, "%s : failed test:    '%s'\n", __func__, test_kv.first.c_str());
            fprintf(stderr, "%s : detokenized to: '%s' instead of '%s'\n", __func__,
                llama_detokenize_bpe(ctx, res).c_str(),
                llama_detokenize_bpe(ctx, test_kv.second).c_str());
            fprintf(stderr, "%s : expected tokens: ", __func__);
            for (const auto & t : test_kv.second) {
                fprintf(stderr, "%6d, ", t);
            }
            fprintf(stderr, "\n");
            fprintf(stderr, "%s : got tokens:      ", __func__);
            for (const auto & t : res) {
                fprintf(stderr, "%6d, ", t);
            }
            fprintf(stderr, "\n");

            success = false;
        }
    }
    // 如果文件名不为空
    if (!fname_text.empty()) {
        // 打印正在进行标记化的文件名
        fprintf(stderr, "%s : tokenizing: '%s'\n", __func__, fname_text.c_str());

        // 读取文件内容到字符串text中
        std::string text;
        {
            std::ifstream ifs(fname_text);
            // 如果文件无法打开，打印错误信息并返回1
            if (!ifs) {
                fprintf(stderr, "%s : error: could not open file '%s'\n", __func__, fname_text.c_str());
                return 1;
            }
            text = std::string(std::istreambuf_iterator<char>(ifs), std::istreambuf_iterator<char>());
        }

        // 打印文本大小
        fprintf(stderr, "%s : text size: %zu\n", __func__, text.size());

        // 对文本进行标记化
        const std::vector<llama_token> res = llama_tokenize(ctx, text, false);

        // 打印标记化后的token数量
        fprintf(stderr, "%s : tokens: %zu\n", __func__, res.size());

        {
            // 创建输出文件名
            const std::string fname_out = fname_text + ".tokcpp";

            // 打开输出文件
            std::ofstream ofs(fname_out);
            // 如果文件无法打开，打印错误信息并返回1
            if (!ofs) {
                fprintf(stderr, "%s : error: could not open file '%s'\n", __func__, fname_out.c_str());
                return 1;
            }

            // 将标记化后的结果写入输出文件
            for (const auto & tok : res) {
                ofs << tok << " '" << llama_detokenize_bpe(ctx, std::vector<int>{tok}) << "'" << std::endl;
            }
        }

        // 打印标记化结果写入的文件名
        fprintf(stderr, "%s : tokens written to '%s'\n", __func__, (fname_text + ".tokcpp").c_str());
    }

    // 释放模型
    llama_free_model(model);
    // 释放上下文
    llama_free(ctx);
    // 释放后端资源
    llama_backend_free();

    // 返回成功或失败的状态码
    return success ? 0 : 3;
# 闭合前面的函数定义
```