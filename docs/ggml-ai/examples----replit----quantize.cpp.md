# `ggml\examples\replit\quantize.cpp`

```cpp
#include "ggml/ggml.h" // 引入 ggml 库的头文件

#include "common-ggml.h" // 引入 ggml 公共头文件
#include "common.h" // 引入通用头文件

#include <cassert> // 断言库
#include <cmath> // 数学函数库
#include <cstdio> // 标准输入输出库
#include <cstring> // 字符串处理库
#include <fstream> // 文件流库
#include <map> // 映射库
#include <regex> // 正则表达式库
#include <string> // 字符串库
#include <vector> // 向量库

struct mpt_hparams {
    int32_t d_model     = 0; // 模型维度
    int32_t max_seq_len = 0; // 最大序列长度
    int32_t n_heads     = 0; // 多头注意力机制中的头数
    int32_t n_layers    = 0; // 层数
    int32_t n_vocab     = 0; // 词汇量大小
    int32_t ftype       = 0; // 文件类型
};

// 量化模型
bool mpt_model_quantize(const std::string & fname_inp,
                        const std::string & fname_out, ggml_ftype ftype) {

    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str()); // 打印加载模型的信息

    auto finp = std::ifstream(fname_inp, std::ios::binary); // 以二进制模式打开输入文件流
    if (!finp) { // 如果文件流打开失败
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__,
                fname_inp.c_str()); // 打印错误信息
        return false; // 返回失败
    }

    auto fout = std::ofstream(fname_out, std::ios::binary); // 以二进制模式打开输出文件流
    if (!fout) { // 如果文件流打开失败
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__,
                fname_out.c_str()); // 打印错误信息
        return false; // 返回失败
    }

    // 验证魔数
    {
        uint32_t magic;
        finp.read((char *)&magic, sizeof(magic)); // 从输入文件流中读取魔数
        if (magic != GGML_FILE_MAGIC) { // 如果魔数不匹配
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n",
                    __func__, fname_inp.c_str()); // 打印错误信息
            return false; // 返回失败
        }

        fout.write((char *)&magic, sizeof(magic)); // 将魔数写入输出文件流
    }

    mpt_hparams hparams; // 定义模型超参数

    // 加载超参数
    // 从输入流中读取模型参数的维度
    finp.read((char *) &hparams.d_model,     sizeof(hparams.d_model));
    finp.read((char *) &hparams.max_seq_len, sizeof(hparams.max_seq_len));
    finp.read((char *) &hparams.n_heads,     sizeof(hparams.n_heads));
    finp.read((char *) &hparams.n_layers,    sizeof(hparams.n_layers));
    finp.read((char *) &hparams.n_vocab,     sizeof(hparams.n_vocab));
    finp.read((char *) &hparams.ftype,       sizeof(hparams.ftype));

    // 根据读取的文件类型计算源量化版本
    const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
    const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

    // 打印模型参数的维度信息
    printf("%s: d_model     = %d\n", __func__, hparams.d_model);
    printf("%s: max_seq_len = %d\n", __func__, hparams.max_seq_len);
    printf("%s: n_heads     = %d\n", __func__, hparams.n_heads);
    printf("%s: n_layers    = %d\n", __func__, hparams.n_layers);
    printf("%s: n_vocab     = %d\n", __func__, hparams.n_vocab);
    printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
    printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
    printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
    printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

    // 将模型参数的维度信息写入输出流
    fout.write((char *) &hparams.d_model,     sizeof(hparams.d_model));
    fout.write((char *) &hparams.max_seq_len, sizeof(hparams.max_seq_len));
    fout.write((char *) &hparams.n_heads,     sizeof(hparams.n_heads));
    fout.write((char *) &hparams.n_layers,    sizeof(hparams.n_layers));
    fout.write((char *) &hparams.n_vocab,     sizeof(hparams.n_vocab));
    fout.write((char *) &ftype_dst,           sizeof(ftype_dst));
}

// 加载词汇表
    {
        // 定义词汇表大小
        const int32_t n_vocab = hparams.n_vocab;

        // 定义字符串变量 word
        std::string word;
        // 遍历词汇表
        for (int i = 0; i < n_vocab; i++) {
            // 读取单词长度
            uint32_t len;
            finp.read((char *)&len, sizeof(len));
            // 写入单词长度
            fout.write((char *)&len, sizeof(len));

            // 调整 word 的大小以容纳单词
            word.resize(len);
            // 读取单词数据
            finp.read((char *)word.data(), len);
            // 写入单词数据
            fout.write((char *)word.data(), len);

            // 读取概率
            float prob;
            finp.read((char *)&prob, sizeof(prob));
            // 写入概率
            fout.write((char *)&prob, sizeof(prob));
        }
    }

    // 打印信息
    printf("%s: quantizing tensors\n", __func__);

    // 要进行量化的张量名称的正则表达式
    const std::vector<std::string> to_quant = {
        ".*weight",
    };

    // 调用量化函数
    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        // 打印错误信息
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__,
                fname_inp.c_str());
        // 返回失败
        return false;
    }

    // 关闭输入文件
    finp.close();
    // 关闭输出文件
    fout.close();

    // 返回成功
    return true;
}
// usage:
//  ./replit-quantize models/replit/ggml-model.bin
//  models/replit/ggml-model-quant.bin type
//
// 主函数，接受命令行参数，执行模型量化操作
int main(int argc, char ** argv) {
    // 如果参数数量不等于4，输出使用说明并返回错误码
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

    // 获取输入和输出文件名
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

        // 如果无法对模型进行量化，输出错误信息并返回错误码
        if (!mpt_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n",
                    __func__, fname_inp.c_str());
            return 1;
        }

        // 计算量化时间
        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // 输出时间信息
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