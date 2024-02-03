# `whisper.cpp\examples\quantize\quantize.cpp`

```cpp
// 包含头文件 ggml.h
#include "ggml.h"

// 包含常用头文件
#include "common.h"
#include "common-ggml.h"

// 包含必要的 C++ 标准库头文件
#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
#include <regex>

// 默认超参数结构体，定义了一些默认数值
struct whisper_hparams {
    int32_t n_vocab       = 51864;
    int32_t n_audio_ctx   = 1500;
    int32_t n_audio_state = 384;
    int32_t n_audio_head  = 6;
    int32_t n_audio_layer = 4;
    int32_t n_text_ctx    = 448;
    int32_t n_text_state  = 384;
    int32_t n_text_head   = 6;
    int32_t n_text_layer  = 4;
    int32_t n_mels        = 80;
    int32_t ftype         = 1;
};

// mel 滤波器结构体，定义了 mel 和 fft 数值以及数据向量
struct whisper_filters {
    int32_t n_mel;
    int32_t n_fft;

    std::vector<float> data;
};

// 对模型进行量化
bool whisper_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
    // 创建词汇表对象
    gpt_vocab vocab;

    // 打印加载模型信息
    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    // 打开输入文件流
    auto finp = std::ifstream(fname_inp, std::ios::binary);
    if (!finp) {
        // 如果无法打开输入文件，打印错误信息并返回 false
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__, fname_inp.c_str());
        return false;
    }

    // 打开输出文件流
    auto fout = std::ofstream(fname_out, std::ios::binary);
    if (!fout) {
        // 如果无法打开输出文件，打印错误信息并返回 false
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname_out.c_str());
        return false;
    }

    // 验证文件魔数
    {
        uint32_t magic;
        finp.read((char *) &magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            // 如果魔数不匹配，打印错误信息并返回 false
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname_inp.c_str());
            return false;
        }

        // 将魔数写入输出文件
        fout.write((char *) &magic, sizeof(magic));
    }

    // 创建超参数对象
    whisper_hparams hparams;

    // 加载超参数
    }

    // 加载 mel 滤波器
    // 定义一个名为 filters 的 whisper_filters 结构体

    // 从输入流 finp 中读取 filters 结构体的 n_mel 字段，并写入输出流 fout
    finp.read ((char *) &filters.n_mel, sizeof(filters.n_mel));
    fout.write((char *) &filters.n_mel, sizeof(filters.n_mel));

    // 从输入流 finp 中读取 filters 结构体的 n_fft 字段，并写入输出流 fout
    finp.read ((char *) &filters.n_fft, sizeof(filters.n_fft));
    fout.write((char *) &filters.n_fft, sizeof(filters.n_fft));

    // 调整 filters 结构体的 data 向量大小为 n_mel * n_fft
    filters.data.resize(filters.n_mel * filters.n_fft);

    // 从输入流 finp 中读取 filters 结构体的 data 向量数据，并写入输出流 fout
    finp.read ((char *) filters.data.data(), filters.data.size() * sizeof(float));
    fout.write((char *) filters.data.data(), filters.data.size() * sizeof(float));

    // 加载词汇表
    {
        // 定义一个 n_vocab 变量并初始化为 0
        int32_t n_vocab = 0;

        // 从输入流 finp 中读取 n_vocab 变量，并写入输出流 fout
        finp.read ((char *) &n_vocab, sizeof(n_vocab));
        fout.write((char *) &n_vocab, sizeof(n_vocab));

        // 定义一个长度为 129 的字符数组 word

        // 遍历 n_vocab 次
        for (int i = 0; i < n_vocab; i++) {
            // 定义一个长度为 32 位整数的变量 len

            // 从输入流 finp 中读取 len 变量，并写入输出流 fout
            finp.read ((char *) &len, sizeof(len));
            fout.write((char *) &len, sizeof(len));

            // 将 word 数组的第 len 个字符设置为 '\0'
            word[len] = '\0';

            // 从输入流 finp 中读取 len 长度的字符到 word 数组，并写入输出流 fout
            finp.read ((char *) word, len);
            fout.write((char *) word, len);

            // 将 word 添加到 vocab 结构体的 token_to_id 和 id_to_token 字典中
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 定义一个不需要量化的张量名称的正则表达式列表 to_skip
    const std::vector<std::string> to_skip = {
        //"encoder.*",
        "encoder.conv1.bias",
        "encoder.conv2.bias",
        "encoder.positional_embedding",
        "decoder.positional_embedding",
    };

    // 调用 ggml_common_quantize_0 函数对模型进行量化，如果失败则返回 false
    if (!ggml_common_quantize_0(finp, fout, ftype, { ".*" }, to_skip)) {
        // 打印错误信息
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
        return false;
    }

    // 关闭输入流 finp
    finp.close();
    // 关闭输出流 fout
    fout.close();

    // 返回 true
    return true;
}

// 主函数，接受命令行参数，执行模型量化
int main(int argc, char ** argv) {
    // 检查参数数量是否正确
    if (argc != 4) {
        // 打印用法信息和支持的类型
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n", argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // 初始化 f16 表格所需
    {
        // 初始化参数结构体
        struct ggml_init_params params = { 0, NULL, false };
        // 初始化 ggml 上下文
        struct ggml_context * ctx = ggml_init(params);
        // 释放 ggml 上下文
        ggml_free(ctx);
    }

    // 获取输入输出文件名
    const std::string fname_inp = argv[1];
    const std::string fname_out = argv[2];

    // 解析模型类型
    const ggml_ftype ftype = ggml_parse_ftype(argv[3]);

    // 记录主函数开始时间
    const int64_t t_main_start_us = ggml_time_us();

    int64_t t_quantize_us = 0;

    // 加载模型
    {
        const int64_t t_start_us = ggml_time_us();

        // 执行模型量化
        if (!whisper_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        // 计算量化时间
        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // 报告时间信息
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    return 0;
}
```