# `ggml\examples\whisper\quantize.cpp`

```
#include "ggml.h"

#include "common.h"
#include "common-ggml.h"

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
#include <regex>

// 默认超参数（Whisper tiny）
struct whisper_hparams {
    int32_t n_vocab       = 51864;  // 词汇量大小
    int32_t n_audio_ctx   = 1500;   // 音频上下文大小
    int32_t n_audio_state = 384;    // 音频状态大小
    int32_t n_audio_head  = 6;      // 音频头数
    int32_t n_audio_layer = 4;      // 音频层数
    int32_t n_text_ctx    = 448;    // 文本上下文大小
    int32_t n_text_state  = 384;    // 文本状态大小
    int32_t n_text_head   = 6;      // 文本头数
    int32_t n_text_layer  = 4;      // 文本层数
    int32_t n_mels        = 80;     // 梅尔频率数量
    int32_t ftype         = 1;      // 文件类型
};

struct whisper_filters {
    int32_t n_mel;  // 梅尔频率数量
    int32_t n_fft;  // FFT 点数

    std::vector<float> data;  // 滤波器数据
};

// 量化模型
bool whisper_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
    gpt_vocab vocab;

    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());  // 打印加载模型的信息

    auto finp = std::ifstream(fname_inp, std::ios::binary);  // 以二进制模式打开输入文件流
    if (!finp) {
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__, fname_inp.c_str());  // 打印打开文件失败的信息
        return false;  // 返回失败
    }

    auto fout = std::ofstream(fname_out, std::ios::binary);  // 以二进制模式打开输出文件流
    if (!fout) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname_out.c_str());  // 打印打开文件失败的信息
        return false;  // 返回失败
    }

    // 验证魔数
    {
        uint32_t magic;
        finp.read((char *) &magic, sizeof(magic));  // 读取文件中的魔数
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname_inp.c_str());  // 打印无效模型文件的信息
            return false;  // 返回失败
        }

        fout.write((char *) &magic, sizeof(magic));  // 写入魔数到输出文件流
    }

    whisper_hparams hparams;  // 创建超参数对象

    // 加载超参数
    }

    // 加载梅尔滤波器
    // 定义一个名为 filters 的 whisper_filters 结构体实例
    whisper_filters filters;

    // 从输入流 finp 中读取 filters.n_mel 的大小，并写入输出流 fout
    finp.read ((char *) &filters.n_mel, sizeof(filters.n_mel));
    fout.write((char *) &filters.n_mel, sizeof(filters.n_mel));

    // 从输入流 finp 中读取 filters.n_fft 的大小，并写入输出流 fout
    finp.read ((char *) &filters.n_fft, sizeof(filters.n_fft));
    fout.write((char *) &filters.n_fft, sizeof(filters.n_fft));

    // 调整 filters.data 的大小为 filters.n_mel * filters.n_fft
    filters.data.resize(filters.n_mel * filters.n_fft);

    // 从输入流 finp 中读取 filters.data 的数据，并写入输出流 fout
    finp.read ((char *) filters.data.data(), filters.data.size() * sizeof(float));
    fout.write((char *) filters.data.data(), filters.data.size() * sizeof(float));

    // 加载词汇表
    {
        // 定义一个名为 n_vocab 的整型变量，并初始化为 0
        int32_t n_vocab = 0;

        // 从输入流 finp 中读取 n_vocab 的大小，并写入输出流 fout
        finp.read ((char *) &n_vocab, sizeof(n_vocab));
        fout.write((char *) &n_vocab, sizeof(n_vocab));

        // 定义一个名为 word 的字符数组，长度为 129
        char word[129];

        // 遍历 n_vocab 次
        for (int i = 0; i < n_vocab; i++) {
            // 定义一个名为 len 的无符号整型变量
            uint32_t len;

            // 从输入流 finp 中读取 len 的大小，并写入输出流 fout
            finp.read ((char *) &len, sizeof(len));
            fout.write((char *) &len, sizeof(len));

            // 将 word[len] 设置为 '\0'
            word[len] = '\0';

            // 从输入流 finp 中读取 len 长度的 word，并写入输出流 fout
            finp.read ((char *) word, len);
            fout.write((char *) word, len);

            // 将 word 作为键，i 作为值存入 vocab.token_to_id
            vocab.token_to_id[word] = i;
            // 将 i 作为键，word 作为值存入 vocab.id_to_token
            vocab.id_to_token[i] = word;
        }
    }

    // 定义一个名为 to_skip 的字符串向量，包含特定的正则表达式
    const std::vector<std::string> to_skip = {
        //"encoder.*",
        "encoder.conv1.bias",
        "encoder.conv2.bias",
        "encoder.positional_embedding",
        "decoder.positional_embedding",
    };

    // 如果无法对模型进行量化，则输出错误信息并返回 false
    if (!ggml_common_quantize_0(finp, fout, ftype, { ".*" }, to_skip)) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
        return false;
    }

    // 关闭输入流 finp
    finp.close();
    // 关闭输出流 fout
    fout.close();

    // 返回 true
    return true;
// 主函数，接受命令行参数
int main(int argc, char ** argv) {
    // 如果参数数量不等于4，打印使用说明并返回错误码1
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n", argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // 初始化f16表格所需的参数
    {
        // 初始化参数结构体
        struct ggml_init_params params = { 0, NULL, false };
        // 初始化上下文
        struct ggml_context * ctx = ggml_init(params);
        // 释放上下文
        ggml_free(ctx);
    }

    // 定义输入文件名和输出文件名
    const std::string fname_inp = argv[1];
    const std::string fname_out = argv[2];

    // 解析模型类型
    const ggml_ftype ftype = ggml_parse_ftype(argv[3]);

    // 记录主函数开始时间
    const int64_t t_main_start_us = ggml_time_us();

    // 记录量化时间
    int64_t t_quantize_us = 0;

    // 加载模型
    {
        // 记录加载模型开始时间
        const int64_t t_start_us = ggml_time_us();

        // 如果无法量化模型，打印错误信息并返回错误码1
        if (!whisper_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        // 计算量化时间
        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // 报告时间
    {
        // 记录主函数结束时间
        const int64_t t_main_end_us = ggml_time_us();

        // 打印量化时间和总时间
        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    // 返回成功码
    return 0;
}
```