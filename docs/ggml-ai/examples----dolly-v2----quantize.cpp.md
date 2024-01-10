# `ggml\examples\dolly-v2\quantize.cpp`

```
    // 包含头文件 ggml/ggml.h
#include "ggml/ggml.h"

    // 包含自定义头文件 common.h 和 common-ggml.h
#include "common.h"
#include "common-ggml.h"

    // 包含标准库头文件
#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
#include <regex>

    // 定义结构体 dollyv2_hparams，包含默认的模型超参数
struct dollyv2_hparams {
    int32_t n_vocab = 50254; // tokenizer.vocab_size
    int32_t n_ctx   = 2048;  // model.config.max_position_embeddings
    int32_t n_embd  = 2560;  // model.config.hidden_size
    int32_t n_head  = 32;    // model.config.num_attention_heads
    int32_t n_layer = 32;    // model.config.num_hidden_layers
    int32_t n_rot   = 20;    // rotary_pct[25%] * (n_embd / n_head)
    int32_t par_res = 1; // 1 = true, 0 = false
    int32_t ftype   = GGML_FTYPE_MOSTLY_F16;
};

    // 对模型进行量化
bool dollyv2_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
    gpt_vocab vocab;

    // 打印加载模型的信息
    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    // 以二进制模式打开输入文件流
    auto finp = std::ifstream(fname_inp, std::ios::binary);
    // 如果打开失败，输出错误信息并返回 false
    if (!finp) {
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__, fname_inp.c_str());
        return false;
    }

    // 以二进制模式打开输出文件流
    auto fout = std::ofstream(fname_out, std::ios::binary);
    // 如果打开失败，输出错误信息并返回 false
    if (!fout) {
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname_out.c_str());
        return false;
    }

    // 验证模型文件的魔数
    {
        uint32_t magic;
        finp.read((char *) &magic, sizeof(magic));
        // 如果魔数不匹配，输出错误信息并返回 false
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname_inp.c_str());
            return false;
        }

        // 将魔数写入输出文件流
        fout.write((char *) &magic, sizeof(magic));
    }

    // 创建 dollyv2_hparams 结构体对象
    dollyv2_hparams hparams;

    // 加载超参数
    // 从输入流中读取 hparams.n_vocab 的值，存入 hparams.n_vocab 变量中
    finp.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
    // 从输入流中读取 hparams.n_ctx 的值，存入 hparams.n_ctx 变量中
    finp.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
    // 从输入流中读取 hparams.n_embd 的值，存入 hparams.n_embd 变量中
    finp.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
    // 从输入流中读取 hparams.n_head 的值，存入 hparams.n_head 变量中
    finp.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
    // 从输入流中读取 hparams.n_layer 的值，存入 hparams.n_layer 变量中
    finp.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
    // 从输入流中读取 hparams.n_rot 的值，存入 hparams.n_rot 变量中
    finp.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
    // 从输入流中读取 hparams.par_res 的值，存入 hparams.par_res 变量中
    finp.read((char *) &hparams.par_res, sizeof(hparams.par_res));
    // 从输入流中读取 hparams.ftype 的值，存入 hparams.ftype 变量中
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
    printf("%s: par_res     = %d\n", __func__, hparams.par_res);
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
    fout.write((char *) &hparams.par_res, sizeof(hparams.par_res));
    fout.write((char *) &ftype_dst,       sizeof(ftype_dst));
    // 结束代码块
    }
    // 加载词汇表
    {
        // 定义词汇表大小
        const int32_t n_vocab = hparams.n_vocab;
    
        // 定义字符串变量 word
        std::string word;
        // 遍历词汇表
        for (int i = 0; i < n_vocab; i++) {
            // 定义变量 len，读取长度信息并写入输出文件
            uint32_t len;
            finp.read ((char *) &len, sizeof(len));
            fout.write((char *) &len, sizeof(len));
    
            // 调整 word 的大小为 len，并读取数据写入输出文件
            word.resize(len);
            finp.read ((char *) word.data(), len);
            fout.write((char *) word.data(), len);
    
            // 将 word 添加到词汇表的 token_to_id 和 id_to_token 中
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }
    
    // 定义需要量化的张量名称的正则表达式
    const std::vector<std::string> to_quant = {
        ".*weight",
    };
    
    // 如果无法对模型进行量化，则输出错误信息并返回 false
    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
        return false;
    }
    
    // 关闭输入和输出文件
    finp.close();
    fout.close();
    
    // 返回 true
    return true;
    }
// 主函数，程序入口
int main(int argc, char ** argv) {
    // 检查命令行参数数量是否正确
    if (argc != 4) {
        // 打印正确的命令行参数用法
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n", argv[0]);
        // 打印可用的类型
        ggml_print_ftypes(stderr);
        return 1;
    }

    // 初始化 f16 表格所需的参数
    {
        // 初始化参数结构体
        struct ggml_init_params params = { 0, NULL, false };
        // 初始化上下文
        struct ggml_context * ctx = ggml_init(params);
        // 释放上下文
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
        // 记录加载模型开始时间
        const int64_t t_start_us = ggml_time_us();

        // 对模型进行量化
        if (!dollyv2_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            // 如果量化失败，打印错误信息
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        // 计算量化所用时间
        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // 报告时间
    {
        // 记录主函数结束时间
        const int64_t t_main_end_us = ggml_time_us();

        // 打印量化所用时间
        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0f);
        // 打印总时间
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    return 0;
}
```