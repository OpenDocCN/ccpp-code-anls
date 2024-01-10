# `ggml\examples\starcoder\quantize.cpp`

```
#include "ggml/ggml.h"

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

// default hparams (GPT-2 117M)
struct starcoder_hparams {
    int32_t n_vocab = 49280;  // 默认词汇量
    int32_t n_ctx   = 2048;   // 默认上下文大小
    int32_t n_embd  = 2048;   // 默认嵌入维度
    int32_t n_head  = 16;     // 默认头数
    int32_t n_layer = 24;     // 默认层数
    int32_t ftype   = 1;      // 默认文件类型
};

// quantize a model
bool starcoder_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
    gpt_vocab vocab;

    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());  // 打印加载模型的信息

    auto finp = std::ifstream(fname_inp, std::ios::binary);  // 以二进制方式打开输入文件流
    if (!finp) {
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__, fname_inp.c_str());  // 如果打开失败则打印错误信息
        return false;  // 返回失败
    }

    auto fout = std::ofstream(fname_out, std::ios::binary);  // 以二进制方式打开输出文件流
    if (!fout) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname_out.c_str());  // 如果打开失败则打印错误信息
        return false;  // 返回失败
    }

    // verify magic
    {
        uint32_t magic;
        finp.read((char *) &magic, sizeof(magic));  // 从输入文件流中读取魔数
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname_inp.c_str());  // 如果魔数不匹配则打印错误信息
            return false;  // 返回失败
        }

        fout.write((char *) &magic, sizeof(magic));  // 将魔数写入输出文件流
    }

    starcoder_hparams hparams;  // 创建模型超参数对象

    // load hparams
    {
        // 从输入流中读取词汇量，存入hparams.n_vocab中
        finp.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        // 从输入流中读取上下文大小，存入hparams.n_ctx中
        finp.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        // 从输入流中读取嵌入维度，存入hparams.n_embd中
        finp.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        // 从输入流中读取头数，存入hparams.n_head中
        finp.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        // 从输入流中读取层数，存入hparams.n_layer中
        finp.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        // 从输入流中读取特征类型，存入hparams.ftype中
        finp.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        // 计算特征类型对应的量化版本号
        const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
        // 计算目标特征类型
        const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

        // 打印各参数的值
        printf("%s: n_vocab     = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx       = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd      = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head      = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer     = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
        printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
        printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

        // 将各参数的值写入输出流
        fout.write((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fout.write((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fout.write((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fout.write((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fout.write((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fout.write((char *) &ftype_dst,       sizeof(ftype_dst));
    }

    // load vocab
    {
        // 读取模型文件中的词汇表大小
        int32_t n_vocab = 0;
        finp.read ((char *) &n_vocab, sizeof(n_vocab));
        // 将词汇表大小写入输出文件
        fout.write((char *) &n_vocab, sizeof(n_vocab));

        // 如果读取的词汇表大小与期望的大小不一致，则输出错误信息并返回false
        if (n_vocab != hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname_inp.c_str(), n_vocab, hparams.n_vocab);
            return false;
        }

        // 逐个读取词汇表中的词和对应的长度，并写入输出文件
        std::string word;
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            finp.read ((char *) &len, sizeof(len));
            fout.write((char *) &len, sizeof(len));

            // 读取词的内容并写入输出文件
            word.resize(len);
            finp.read ((char *) word.data(), len);
            fout.write((char *) word.data(), len);

            // 将词和对应的索引添加到词汇表中
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 要进行量化的张量名称的正则表达式
    const std::vector<std::string> to_quant = {
        "model/wte",
        "model/lm_head",
        "model/h.*/attn/c_attn/w",
        "model/h.*/attn/c_proj/w",
        "model/h.*/mlp/c_fc/w",
        "model/h.*/mlp/c_proj/w",
    };

    // 调用量化函数对模型进行量化，如果失败则输出错误信息并返回false
    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
        return false;
    }

    // 关闭输入和输出文件
    finp.close();
    fout.close();

    // 返回true表示处理成功
    return true;
}
// 主函数，接受命令行参数，执行模型量化
int main(int argc, char ** argv) {
    // 检查命令行参数数量是否正确，如果不正确则打印用法信息并返回错误码
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n", argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // 初始化 f16 表格所需的参数
    {
        struct ggml_init_params params = { 0, NULL, false };
        // 初始化 ggml 上下文
        struct ggml_context * ctx = ggml_init(params);
        // 释放 ggml 上下文
        ggml_free(ctx);
    }

    // 从命令行参数中获取输入和输出文件名
    const std::string fname_inp = argv[1];
    const std::string fname_out = argv[2];

    // 解析命令行参数中的模型类型
    const ggml_ftype ftype = ggml_parse_ftype(argv[3]);

    // 记录主函数开始时间
    const int64_t t_main_start_us = ggml_time_us();

    int64_t t_quantize_us = 0;

    // 加载模型
    {
        // 记录加载模型开始时间
        const int64_t t_start_us = ggml_time_us();

        // 调用模型量化函数，如果失败则打印错误信息并返回错误码
        if (!starcoder_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        // 计算模型加载时间
        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // 报告时间
    {
        // 记录主函数结束时间
        const int64_t t_main_end_us = ggml_time_us();

        // 打印模型量化时间和总时间
        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    // 返回成功码
    return 0;
}
```