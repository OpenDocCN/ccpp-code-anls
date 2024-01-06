# `PowerInfer\tests\test-tokenizer-0-llama.cpp`

```
// 包含所需的头文件
#include "llama.h"
#include "common.h"
#include "console.h"

#include <cstdio>
#include <string>
#include <map>
#include <vector>
#include <fstream>

// 生成测试用的标记化数据
static const std::map<std::string, std::vector<llama_token>> & k_tests() {
    // 静态变量，存储测试数据的映射
    static std::map<std::string, std::vector<llama_token>> _k_tests = {
        // 空字符串对应的标记化数据
        { ""                      , {  }, },
        // 单个空格对应的标记化数据
        { " "                     , {     259, }, },
        // 两个空格对应的标记化数据
        { "  "                    , {    1678, }, },
        // 三个空格对应的标记化数据
        { "   "                   , {     268, }, },
        // 制表符对应的标记化数据
        { "\t"                    , {   29871,     12, }, },
        // 换行符对应的标记化数据
        { "\n"                    , {   29871,     13, }, },
        // 制表符和换行符组合对应的标记化数据
        { "\t\n"                  , {   29871,     12,     13, }, },
# 定义一个包含字符串和对应数据的数组
{
    "Hello world", {15043, 3186},
    " Hello world", {29871, 15043, 3186},
    "Hello World", {15043, 2787},
    " Hello World", {29871, 15043, 2787},
    " Hello World!", {29871, 15043, 2787, 29991},
    "Hello, world!", {15043, 29892, 3186, 29991},
    " Hello, world!", {29871, 15043, 29892, 3186, 29991},
    " this is 🦙.cpp", {29871, 445, 338, 29871, 243, 162, 169, 156, 29889, 8223},
    "w048 7tuijk dsdfhu", {281, 29900, 29946, 29947, 29871, 29955, 9161, 13535, 18031, 2176, 6905},
    "нещо на Български", {1538, 4851, 665, 1386, 29713, 1305},
    "Hello", {15043},
    " Hello", {29871, 15043},
    "  Hello", {259, 15043},
    "   Hello", {1678, 15043},
    "    Hello", {268, 15043},
    "    Hello\n    Hello", {268, 15043, 13, 1678, 15043},
    " (", {29871, 313},
};

# 返回包含字符串和对应数据的数组
return _k_tests;
// 主函数，接受命令行参数
int main(int argc, char **argv) {
    // 检查参数数量，如果少于2个则打印用法信息并返回错误
    if (argc < 2) {
        fprintf(stderr, "Usage: %s vocab-file [text-file]\n", argv[0]);
        return 1;
    }

    // 从命令行参数中获取词汇文件名
    const std::string fname = argv[1];

    // 如果命令行参数中有第二个文件名，则获取该文件名
    std::string fname_text;
    if (argc > 2) {
        fname_text = argv[2];
    }

    // 打印正在读取词汇文件的信息
    fprintf(stderr, "%s : reading vocab from: '%s'\n", __func__, fname.c_str());

    // 声明 llama_model 和 llama_context 对象
    llama_model * model;
    llama_context * ctx;
    // 初始化后端 llama
    llama_backend_init(false);

    // 加载词汇表
    {
        // 获取默认的模型参数
        auto mparams = llama_model_default_params();

        // 设置只加载词汇表
        mparams.vocab_only = true;

        // 从文件加载模型
        model = llama_load_model_from_file(fname.c_str(), mparams);

        // 如果加载失败，输出错误信息并返回1
        if (model == NULL) {
            fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
            return 1;
        }

        // 获取默认的上下文参数
        auto cparams = llama_context_default_params();

        // 使用加载的模型创建上下文
        ctx = llama_new_context_with_model(model, cparams);

        // 如果创建上下文失败，执行以下操作
        if (ctx == NULL) {
// 打印错误信息，指示无法加载词汇表
fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
// 释放模型内存
llama_free_model(model);
// 返回错误代码1
return 1;
// 如果词汇表类型不是 SPM，则打印错误信息，释放模型内存和上下文内存，返回错误代码2
if (llama_vocab_type(model) != LLAMA_VOCAB_TYPE_SPM) {
    fprintf(stderr, "%s : error: vocab type is not SPM\n", __func__);
    llama_free_model(model);
    llama_free(ctx);
    return 2;
}
// 如果是在 Windows 系统下，则初始化控制台支持，并在程序退出时清理控制台
#ifdef _WIN32
    console::init(false, false);
    atexit([]() { console::cleanup(); });
#endif
// 设置成功标志为 true
bool success = true;
// 遍历 k_tests() 返回的键值对
for (const auto & test_kv : k_tests()) {
    // 调用 llama_tokenize 函数进行分词，分别传入 true 和 false 作为参数
    const std::vector<llama_token> res_bos   = llama_tokenize(ctx, test_kv.first, true);
    const std::vector<llama_token> res_nobos = llama_tokenize(ctx, test_kv.first, false);

    // 打印源字符串
    printf("\n");
    printf("src: '%s'\n", test_kv.first.c_str());
    // 打印使用 res_bos 分词后的结果
    printf("res: '%s'\n", llama_detokenize_spm(ctx, res_bos).c_str());
    // 打印分词后的 token
    printf("tok: ");
    for (const auto & tok : res_bos) {
        printf("%d ", tok);
    }
    printf("\n");

    // 检查分词结果是否正确
    bool correct = res_nobos.size() == test_kv.second.size() && res_bos.size() == res_nobos.size() + 1 && res_bos[0] == 1;

    // 遍历分词结果，检查是否与预期结果一致
    for (int i = 0; i < (int) res_nobos.size() && correct; ++i) {
        if (test_kv.second[i] != res_bos[i + 1]) {
            correct = false;
        }
    }
}
        // 检查测试结果是否正确，如果不正确则将 correct 设为 false
        if (test_kv.second[i] != res_nobos[i]) {
            correct = false;
        }

        // 如果结果不正确，输出测试失败的信息，包括函数名和测试键值
        if (!correct) {
            fprintf(stderr, "%s : failed test:    '%s'\n", __func__, test_kv.first.c_str());
            // 输出实际的解码结果和预期的解码结果
            fprintf(stderr, "%s : detokenized to: '%s' instead of '%s'\n", __func__,
                llama_detokenize_spm(ctx, res_nobos).c_str(),
                llama_detokenize_spm(ctx, test_kv.second).c_str());
            // 输出预期的 token
            fprintf(stderr, "%s : expected tokens: ", __func__);
            for (const auto & t : test_kv.second) {
                fprintf(stderr, "%6d, ", t);
            }
            fprintf(stderr, "\n");
            // 输出实际的 token
            fprintf(stderr, "%s : got tokens:      ", __func__);
            for (const auto & t : res_nobos) {
                fprintf(stderr, "%6d, ", t);
            }
            fprintf(stderr, "\n");
    // 初始化一个布尔变量 success，用于标记操作是否成功
    success = false;
    // 如果文件名不为空
    if (!fname_text.empty()) {
        // 打印正在进行分词操作的文件名
        fprintf(stderr, "%s : tokenizing: '%s'\n", __func__, fname_text.c_str());
        // 声明一个字符串变量 text
        std::string text;
        {
            // 以文本模式打开文件
            std::ifstream ifs(fname_text);
            // 如果文件打开失败
            if (!ifs) {
                // 打印错误信息并返回 1
                fprintf(stderr, "%s : error: could not open file '%s'\n", __func__, fname_text.c_str());
                return 1;
            }
            // 从文件流中读取数据到字符串 text
            text = std::string(std::istreambuf_iterator<char>(ifs), std::istreambuf_iterator<char>());
        }
        // 打印文本大小
        fprintf(stderr, "%s : text size: %zu\n", __func__, text.size());
    }
        // 使用 llama_tokenize 函数对文本进行分词，返回分词结果
        const std::vector<llama_token> res = llama_tokenize(ctx, text, true);

        // 打印函数名和分词结果的数量
        fprintf(stderr, "%s : tokens: %zu\n", __func__, res.size());

        // 创建输出文件名，并打开输出流
        const std::string fname_out = fname_text + ".tokcpp";
        std::ofstream ofs(fname_out);
        // 如果无法打开输出流，则打印错误信息并返回
        if (!ofs) {
            fprintf(stderr, "%s : error: could not open file '%s'\n", __func__, fname_out.c_str());
            return 1;
        }

        // 遍历分词结果，将分词结果和对应的反分词结果写入输出流
        for (const auto & tok : res) {
            ofs << tok << " '" << llama_detokenize_spm(ctx, std::vector<int>{tok}) << "'" << std::endl;
        }

        // 打印分词结果写入的文件名
        fprintf(stderr, "%s : tokens written to '%s'\n", __func__, (fname_text + ".tokcpp").c_str());
    }
# 释放模型占用的资源
llama_free_model(model);
# 释放上下文占用的资源
llama_free(ctx);
# 释放后端占用的资源
llama_backend_free();
# 根据操作结果返回相应的状态码
return success ? 0 : 3;
```