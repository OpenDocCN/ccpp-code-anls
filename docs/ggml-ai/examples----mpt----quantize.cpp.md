# `ggml\examples\mpt\quantize.cpp`

```
#include "ggml/ggml.h"
#include "common-ggml.h"
#include "common.h"
#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <regex>
#include <string>
#include <vector>

// 定义包含模型超参数的结构体
struct mpt_hparams {
    int32_t d_model      = 0;
    int32_t max_seq_len  = 0;
    int32_t n_heads      = 0;
    int32_t n_layers     = 0;
    int32_t n_vocab      = 0;
    float alibi_bias_max = 0;
    float clip_qkv       = 0;
    int32_t ftype        = 0;
};

// 对模型进行量化
bool mpt_model_quantize(const std::string & fname_inp,
                        const std::string & fname_out, ggml_ftype ftype) {
    // 打印加载模型的信息
    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    // 打开输入文件流
    auto finp = std::ifstream(fname_inp, std::ios::binary);
    if (!finp) {
        // 如果无法打开输入文件，则打印错误信息并返回false
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__,
                fname_inp.c_str());
        return false;
    }

    // 打开输出文件流
    auto fout = std::ofstream(fname_out, std::ios::binary);
    if (!fout) {
        // 如果无法打开输出文件，则打印错误信息并返回false
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__,
                fname_out.c_str());
        return false;
    }

    // 验证模型文件的魔数
    {
        uint32_t magic;
        finp.read((char *)&magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            // 如果魔数不匹配，则打印错误信息并返回false
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n",
                    __func__, fname_inp.c_str());
            return false;
        }

        // 将魔数写入输出文件
        fout.write((char *)&magic, sizeof(magic));
    }

    mpt_hparams hparams;

    // 加载超参数
    // ...

    // 加载词汇表
    {
        const int32_t n_vocab = hparams.n_vocab;

        std::string word;
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            finp.read((char *)&len, sizeof(len));
            fout.write((char *)&len, sizeof(len));

            word.resize(len);
            finp.read((char *)word.data(), len);
            fout.write((char *)word.data(), len);
        }
    }
}
    // 打印正在进行张量量化的消息，包括函数名
    printf("%s: quantizing tensors\n", __func__);

    // 要进行量化的张量名称的正则表达式列表
    const std::vector<std::string> to_quant = {
        ".*weight",
    };

    // 调用 ggml_common_quantize_0 函数进行张量量化，如果失败则打印错误消息并返回 false
    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__,
                fname_inp.c_str());
        return false;
    }

    // 关闭输入和输出文件流
    finp.close();
    fout.close();

    // 返回 true 表示量化成功
    return true;
// usage:
//  ./mpt-quantize models/mpt/ggml-model.bin
//  models/mpt/ggml-model-quant.bin type
//
// 主函数，接受命令行参数，执行模型量化
int main(int argc, char ** argv) {
    // 检查参数数量是否正确
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n",
                argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // 初始化 f16 表格所需的参数
    {
        struct ggml_init_params params = {0, NULL, false};
        // 初始化 ggml 上下文
        struct ggml_context * ctx = ggml_init(params);
        // 释放 ggml 上下文
        ggml_free(ctx);
    }

    // 读取输入和输出文件名
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
        if (!mpt_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n",
                    __func__, fname_inp.c_str());
            return 1;
        }

        // 计算量化时间
        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // 报告时间
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__,
               t_quantize_us / 1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__,
               (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    return 0;
}
```