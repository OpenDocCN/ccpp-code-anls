# `ggml\examples\gpt-neox\quantize.cpp`

```cpp
    // 包含 ggml 库的头文件
#include "ggml/ggml.h"

    // 包含通用的头文件
#include "common.h"
#include "common-ggml.h"

    // 包含必要的标准库头文件
#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
#include <regex>

    // 默认的模型超参数 (StableLM 3B)
struct gpt_neox_hparams {
    int32_t n_vocab = 50257;  // 词汇量大小
    int32_t n_ctx   = 4096;   // 上下文大小
    int32_t n_embd  = 4096;   // 嵌入维度
    int32_t n_head  = 32;     // 头数
    int32_t n_layer = 16;     // 层数
    int32_t n_rot   = 32;     // 旋转数，计算公式为 0.25 * (n_embd / n_head)
    int32_t par_res = 1;      // 并行计算结果，1 代表 true，0 代表 false
    int32_t ftype   = 1;      // 文件类型
};

    // 对模型进行量化
bool gpt_neox_model_quantize(const std::string & fname_inp, const std::string & fname_out, ggml_ftype ftype) {
    gpt_vocab vocab;

    // 打印加载模型的信息
    printf("%s: loading model from '%s'\n", __func__, fname_inp.c_str());

    // 以二进制方式打开输入文件流
    auto finp = std::ifstream(fname_inp, std::ios::binary);
    if (!finp) {
        // 如果打开失败，打印错误信息并返回 false
        fprintf(stderr, "%s: failed to open '%s' for reading\n", __func__, fname_inp.c_str());
        return false;
    }

    // 以二进制方式打开输出文件流
    auto fout = std::ofstream(fname_out, std::ios::binary);
    if (!fout) {
        // 如果打开失败，打印错误信息并返回 false
        fprintf(stderr, "%s: failed to open '%s' for writing\n", __func__, fname_out.c_str());
        return false;
    }

    // 验证文件的魔数
    {
        uint32_t magic;
        finp.read((char *) &magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            // 如果魔数不匹配，打印错误信息并返回 false
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname_inp.c_str());
            return false;
        }

        // 将魔数写入输出文件流
        fout.write((char *) &magic, sizeof(magic));
    }

    gpt_neox_hparams hparams;

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
    // 从输入流中读取 hparams.par_res 的值，并将其存储到 hparams.par_res 中
    finp.read((char *) &hparams.par_res, sizeof(hparams.par_res));
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

    // load vocab
    {
        // 定义词汇表大小
        const int32_t n_vocab = hparams.n_vocab;
    
        // 定义字符串变量 word
        std::string word;
        // 遍历词汇表
        for (int i = 0; i < n_vocab; i++) {
            // 定义变量 len，读取长度信息
            uint32_t len;
            finp.read ((char *) &len, sizeof(len));
            // 将长度信息写入输出文件
            fout.write((char *) &len, sizeof(len));
    
            // 调整 word 的大小以容纳单词
            word.resize(len);
            // 读取单词数据
            finp.read ((char *) word.data(), len);
            // 将单词数据写入输出文件
            fout.write((char *) word.data(), len);
    
            // 将单词和对应的索引添加到词汇表中
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }
    
    // 定义需要量化的张量名称的正则表达式
    const std::vector<std::string> to_quant = {
        ".*weight",
    };
    
    // 调用量化函数对模型进行量化
    if (!ggml_common_quantize_0(finp, fout, ftype, to_quant, {})) {
        // 如果量化失败，输出错误信息
        fprintf(stderr, "%s: failed to quantize model '%s'\n", __func__, fname_inp.c_str());
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
// 主函数，接受命令行参数，执行模型量化操作
int main(int argc, char ** argv) {
    // 检查命令行参数数量是否正确，如果不正确则打印用法信息并返回错误码
    if (argc != 4) {
        fprintf(stderr, "usage: %s model-f32.bin model-quant.bin type\n", argv[0]);
        ggml_print_ftypes(stderr);
        return 1;
    }

    // 初始化 f16 表格所需的参数，并释放上下文
    {
        struct ggml_init_params params = { 0, NULL, false };
        struct ggml_context * ctx = ggml_init(params);
        ggml_free(ctx);
    }

    // 从命令行参数中获取输入和输出文件名
    const std::string fname_inp = argv[1];
    const std::string fname_out = argv[2];

    // 解析命令行参数中的模型类型
    const ggml_ftype ftype = ggml_parse_ftype(argv[3]);

    // 记录主函数开始时间
    const int64_t t_main_start_us = ggml_time_us();

    // 初始化量化时间
    int64_t t_quantize_us = 0;

    // 加载模型并执行量化
    {
        // 记录加载模型开始时间
        const int64_t t_start_us = ggml_time_us();

        // 调用函数加载并执行模型量化，如果失败则打印错误信息并返回错误码
        if (!gpt_neox_model_quantize(fname_inp, fname_out, ggml_ftype(ftype))) {
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        // 计算量化时间
        t_quantize_us = ggml_time_us() - t_start_us;
    }

    // 打印时间统计信息
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