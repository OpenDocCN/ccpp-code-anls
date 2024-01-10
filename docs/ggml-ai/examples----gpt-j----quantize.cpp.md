# `ggml\examples\gpt-j\quantize.cpp`

```
// 包含所需的头文件
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

// 默认超参数设置为 GPT-J 6B 模型的参数
struct gptj_hparams {
    int32_t n_vocab = 50400;
    int32_t n_ctx   = 2048;
    int32_t n_embd  = 4096;
    int32_t n_head  = 16;
    int32_t n_layer = 28;
    int32_t n_rot   = 64;
    int32_t ftype   = 1;
};

// 对模型进行量化
bool gptj_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
    gpt_vocab vocab;

    // 打印加载模型的信息
    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    // 打开输入文件流
    auto finp = std::ifstream(fname_inp, std::ios::binary);
    // 如果无法打开输入文件流，则打印错误信息并返回 false
    if (!finp) {
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__, fname_inp.c_str());
        return false;
    }

    // 打开输出文件流
    auto fout = std::ofstream(fname_out, std::ios::binary);
    // 如果无法打开输出文件流，则打印错误信息并返回 false
    if (!fout) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname_out.c_str());
        return false;
    }

    // 验证模型文件的魔数
    {
        uint32_t magic;
        finp.read((char *) &magic, sizeof(magic));
        // 如果魔数不匹配，则打印错误信息并返回 false
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname_inp.c_str());
            return false;
        }
        // 将魔数写入输出文件流
        fout.write((char *) &magic, sizeof(magic));
    }

    gptj_hparams hparams;

    // 加载超参数
    // 从输入流中读取 hparams.n_vocab 的值，并将其存储到 hparams.n_vocab 中
    finp.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
    // 从输入流中读取 hparams.n_ctx 的值，并将其存储到 hparams.n_ctx 中
    finp.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
    // 从输入流中读取 hparams.n_embd 的值，并将其存储到 hparams.n_embd 中
    finp.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
    // 从输入流中读取 hparams.n_head 的值，并将其存储到 hparams.n_head 中
    finp.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
    // 从输入流中读取 hparams.n_layer 的值，并将其存储到 hparams.n_layer 中
    finp.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
    // 从输入流中读取 hparams.n_rot 的值，并将其存储到 hparams.n_rot 中
    finp.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
    // 从输入流中读取 hparams.ftype 的值，并将其存储到 hparams.ftype 中
    finp.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

    // 计算 qntvr_src 的值
    const int32_t qntvr_src =    hparams.ftype / GGML_QNT_VERSION_FACTOR;
    // 计算 ftype_dst 的值
    const int32_t ftype_dst = GGML_QNT_VERSION * GGML_QNT_VERSION_FACTOR + ftype;

    // 打印 hparams 的各个属性值
    printf("%s: n_vocab     = %d\n", __func__, hparams.n_vocab);
    printf("%s: n_ctx       = %d\n", __func__, hparams.n_ctx);
    printf("%s: n_embd      = %d\n", __func__, hparams.n_embd);
    printf("%s: n_head      = %d\n", __func__, hparams.n_head);
    printf("%s: n_layer     = %d\n", __func__, hparams.n_layer);
    printf("%s: ftype (src) = %d\n", __func__, hparams.ftype);
    printf("%s: qntvr (src) = %d\n", __func__, qntvr_src);
    printf("%s: ftype (dst) = %d\n", __func__, ftype_dst);
    printf("%s: qntvr (dst) = %d\n", __func__, GGML_QNT_VERSION);

    // 将 hparams 的各个属性值写入输出流
    fout.write((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
    fout.write((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
    fout.write((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
    fout.write((char *) &hparams.n_head,  sizeof(hparams.n_head));
    fout.write((char *) &hparams.n_layer, sizeof(hparams.n_layer));
    fout.write((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
    fout.write((char *) &ftype_dst,       sizeof(ftype_dst));
    // 结束代码块
    }

    // load vocab
    {
        // 读取模型文件中的词汇量大小
        int32_t n_vocab = 0;
        finp.read ((char *) &n_vocab, sizeof(n_vocab));
        // 将词汇量大小写入输出文件
        fout.write((char *) &n_vocab, sizeof(n_vocab));

        // 如果读取的词汇量大小与预期不符，则输出错误信息并返回false
        if (n_vocab != hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname_inp.c_str(), n_vocab, hparams.n_vocab);
            return false;
        }

        // 逐个读取词汇表中的词语和对应的长度，并写入输出文件
        std::string word;
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            finp.read ((char *) &len, sizeof(len));
            fout.write((char *) &len, sizeof(len));

            // 读取词语并写入输出文件
            word.resize(len);
            finp.read ((char *) word.data(), len);
            fout.write((char *) word.data(), len);

            // 将词语和对应的索引添加到词汇表中
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 要进行量化的张量名称的正则表达式
    const std::vector<std::string> to_quant = {
        ".*weight",
    };

    // 对模型进行量化处理，如果失败则输出错误信息并返回false
    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
        return false;
    }

    // 关闭输入和输出文件
    finp.close();
    fout.close();

    // 处理成功，返回true
    return true;
}
// 主函数，接受命令行参数，执行模型量化
int main(int argc, char ** argv) {
    // 如果参数数量不等于4，输出使用说明并返回错误码
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n", argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // 初始化 f16 表格所需的参数
    {
        struct ggml_init_params params = { 0, NULL, false };
        // 初始化上下文
        struct ggml_context * ctx = ggml_init(params);
        // 释放上下文
        ggml_free(ctx);
    }

    // 从命令行参数中获取输入和输出文件名
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
        if (!gptj_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        // 计算量化时间
        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // 输出时间统计
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    return 0;
}
```