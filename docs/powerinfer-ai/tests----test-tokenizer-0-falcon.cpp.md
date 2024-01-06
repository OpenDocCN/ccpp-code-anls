# `PowerInfer\tests\test-tokenizer-0-falcon.cpp`

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

// 生成测试用的标记映射
static const std::map<std::string, std::vector<llama_token>> & k_tests() {
    // 静态变量，存储测试用例的标记映射
    static std::map<std::string, std::vector<llama_token>> _k_tests = {
        // 空字符串的测试用例
        { ""                      , {  }, },
        // 包含一个空格的测试用例
        { " "                     , {     204, }, },
        // 包含两个空格的测试用例
        { "  "                    , {     258, }, },
        // 包含三个空格的测试用例
        { "   "                   , {     466, }, },
        // 包含一个制表符的测试用例
        { "\t"                    , {     192, }, },
        // 包含一个换行符的测试用例
        { "\n"                    , {     193, }, },
        // 包含一个制表符和一个换行符的测试用例
        { "\t\n"                  , {   19125, }, },
# 创建一个包含字符串和对应数据的数组
{
    "Hello world", {9856, 1079, }, 
    // 字符串 "Hello world" 对应的数据为 9856 和 1079
},
{
    " Hello world", {23090, 1079, }, 
    // 字符串 " Hello world" 对应的数据为 23090 和 1079
},
{
    "Hello World", {9856, 2889, }, 
    // 字符串 "Hello World" 对应的数据为 9856 和 2889
},
{
    " Hello World", {23090, 2889, }, 
    // 字符串 " Hello World" 对应的数据为 23090 和 2889
},
{
    " Hello World!", {23090, 2889, 12, }, 
    // 字符串 " Hello World!" 对应的数据为 23090, 2889 和 12
},
{
    "Hello, world!", {9856, 23, 1079, 12, }, 
    // 字符串 "Hello, world!" 对应的数据为 9856, 23, 1079 和 12
},
{
    " Hello, world!", {23090, 23, 1079, 12, }, 
    // 字符串 " Hello, world!" 对应的数据为 23090, 23, 1079 和 12
},
{
    " this is 🦙.cpp", {414, 304, 3346, 111, 231, 25, 29247, }, 
    // 字符串 " this is 🦙.cpp" 对应的数据为 414, 304, 3346, 111, 231, 25 和 29247
},
{
    "w048 7tuijk dsdfhu", {98, 55866, 204, 34, 16682, 7149, 36190, 6869, 11481, }, 
    // 字符串 "w048 7tuijk dsdfhu" 对应的数据为 98, 55866, 204, 34, 16682, 7149, 36190, 6869 和 11481
},
{
    "нещо на Български", {150, 133, 6207, 151, 215, 150, 134, 5052, 133, 6279, 5052, 223, 151, 216, 49679, 123, 53110, 47043, 7795, }, 
    // 字符串 "нещо на Български" 对应的数据为一系列数字
},
{
    "Hello", {9856, }, 
    // 字符串 "Hello" 对应的数据为 9856
},
{
    " Hello", {23090, }, 
    // 字符串 " Hello" 对应的数据为 23090
},
{
    "  Hello", {204, 23090, }, 
    // 字符串 "  Hello" 对应的数据为 204 和 23090
},
{
    "   Hello", {258, 23090, }, 
    // 字符串 "   Hello" 对应的数据为 258 和 23090
},
{
    "    Hello", {466, 23090, }, 
    // 字符串 "    Hello" 对应的数据为 466 和 23090
},
{
    "    Hello\n    Hello", {466, 23090, 742, 23090, }, 
    // 字符串 "    Hello\n    Hello" 对应的数据为一系列数字
},
{
    "\n =", {1212, 40, }, 
    // 字符串 "\n =" 对应的数据为 1212 和 40
},
{
    "' era", {18, 4932, }, 
    // 字符串 "' era" 对应的数据为 18 和 4932
},
};
    // 返回 _k_tests 变量的值
    return _k_tests;
}

int main(int argc, char **argv) {
    // 检查命令行参数数量，如果少于 2 个则打印用法信息并返回 1
    if (argc < 2) {
        fprintf(stderr, "Usage: %s vocab-file [text-file]\n", argv[0]);
        return 1;
    }

    // 将第一个命令行参数作为文件名保存到 fname 变量中
    const std::string fname = argv[1];

    // 如果命令行参数数量大于 2，则将第二个参数作为文件名保存到 fname_text 变量中
    std::string fname_text;
    if (argc > 2) {
        fname_text = argv[2];
    }

    // 打印读取词汇表的信息
    fprintf(stderr, "%s : reading vocab from: '%s'\n", __func__, fname.c_str());

    // 声明指向 llama_model 和 llama_context 的指针变量
    llama_model * model;
    llama_context * ctx;
// 初始化 LLAMA 后端
llama_backend_init(false);

// 加载词汇表
{
    // 获取默认的模型参数
    auto mparams = llama_model_default_params();

    // 仅加载词汇表
    mparams.vocab_only = true;

    // 从文件中加载模型
    model = llama_load_model_from_file(fname.c_str(), mparams);

    // 如果加载失败，打印错误信息并返回
    if (model == NULL) {
        fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
        return 1;
    }

    // 获取默认的上下文参数
    auto cparams = llama_context_default_params();

    // 使用加载的模型创建上下文
    ctx = llama_new_context_with_model(model, cparams);
}
// 如果上下文为空，则打印错误信息并释放模型，返回错误码1
if (ctx == NULL) {
    fprintf(stderr, "%s: error: failed to load vocab '%s'\n", __func__, fname.c_str());
    llama_free_model(model);
    return 1;
}

// 如果词汇表类型不是BPE，则打印错误信息并释放模型和上下文，返回错误码2
if (llama_vocab_type(model) != LLAMA_VOCAB_TYPE_BPE) {
    fprintf(stderr, "%s : error: vocab type is not BPE\n", __func__);
    llama_free_model(model);
    llama_free(ctx);
    return 2;
}

#ifdef _WIN32
    // 为了支持Unicode控制台，需要进行初始化和清理
    console::init(false, false);
    atexit([]() { console::cleanup(); });
#endif
    // 布尔变量，用于表示操作是否成功
    bool success = true;

    // 遍历测试集合中的每个测试用例
    for (const auto & test_kv : k_tests()) {
        // 调用 llama_tokenize 函数对测试用例进行分词，返回分词结果
        const std::vector<llama_token> res = llama_tokenize(ctx, test_kv.first, false);

        // 打印原始字符串
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

        // 检查分词结果的长度是否与预期长度相同
        bool correct = res.size() == test_kv.second.size();

        // 遍历分词结果和预期结果，检查是否一致
        for (int i = 0; i < (int) res.size() && correct; ++i) {
            if (test_kv.second[i] != res[i]) {
                correct = false;
            }
        }
        }

        // 如果测试不通过，输出测试失败的信息
        if (!correct) {
            // 输出测试失败的键值对
            fprintf(stderr, "%s : failed test:    '%s'\n", __func__, test_kv.first.c_str());
            // 输出测试失败的键值对的解码结果和期望结果
            fprintf(stderr, "%s : detokenized to: '%s' instead of '%s'\n", __func__,
                llama_detokenize_bpe(ctx, res).c_str(),
                llama_detokenize_bpe(ctx, test_kv.second).c_str());
            // 输出期望的 tokens
            fprintf(stderr, "%s : expected tokens: ", __func__);
            for (const auto & t : test_kv.second) {
                fprintf(stderr, "%6d, ", t);
            }
            fprintf(stderr, "\n");
            // 输出实际得到的 tokens
            fprintf(stderr, "%s : got tokens:      ", __func__);
            for (const auto & t : res) {
                fprintf(stderr, "%6d, ", t);
            }
            fprintf(stderr, "\n");

            // 设置测试结果为失败
            success = false;
        }
    // 如果文件名不为空
    if (!fname_text.empty()) {
        // 打印正在进行标记化的文件名
        fprintf(stderr, "%s : tokenizing: '%s'\n", __func__, fname_text.c_str());

        // 读取文件内容到字符串
        std::string text;
        {
            // 打开文件流
            std::ifstream ifs(fname_text);
            // 如果文件流无法打开，打印错误信息并返回1
            if (!ifs) {
                fprintf(stderr, "%s : error: could not open file '%s'\n", __func__, fname_text.c_str());
                return 1;
            }
            // 从文件流中读取内容到字符串
            text = std::string(std::istreambuf_iterator<char>(ifs), std::istreambuf_iterator<char>());
        }

        // 打印读取到的文本大小
        fprintf(stderr, "%s : text size: %zu\n", __func__, text.size());

        // 对文本进行标记化处理，返回标记化结果
        const std::vector<llama_token> res = llama_tokenize(ctx, text, false);

        // 打印标记化结果的数量
        fprintf(stderr, "%s : tokens: %zu\n", __func__, res.size());
    }
// 创建输出文件名
const std::string fname_out = fname_text + ".tokcpp";

// 打开输出文件流
std::ofstream ofs(fname_out);

// 检查文件流是否成功打开
if (!ofs) {
    // 如果文件流打开失败，输出错误信息并返回1
    fprintf(stderr, "%s : error: could not open file '%s'\n", __func__, fname_out.c_str());
    return 1;
}

// 遍历结果并将token写入文件
for (const auto & tok : res) {
    ofs << tok << " '" << llama_detokenize_bpe(ctx, std::vector<int>{tok}) << "'" << std::endl;
}

// 输出tokens写入的文件名
fprintf(stderr, "%s : tokens written to '%s'\n", __func__, (fname_text + ".tokcpp").c_str());

// 释放模型内存
llama_free_model(model);

// 释放上下文内存
llama_free(ctx);
# 释放后端资源
llama_backend_free();
# 根据成功与否返回不同的值，成功返回0，失败返回3
return success ? 0 : 3;
```