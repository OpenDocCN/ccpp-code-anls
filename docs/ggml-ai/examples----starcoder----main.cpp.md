# `ggml\examples\starcoder\main.cpp`

```cpp
// 包含头文件 ggml/ggml.h
#include "ggml/ggml.h"

// 包含自定义的头文件 common.h 和 common-ggml.h
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

// 如果编译器为 MSC_VER，则禁用警告 4244 和 4267，可能会丢失数据
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 默认超参数 (GPT-2 117M)
// 参考链接：https://huggingface.co/bigcode/gpt_bigcode-santacoder/blob/main/config.json
struct starcoder_hparams {
    int32_t n_vocab = 49280;
    int32_t n_ctx   = 2048;
    int32_t n_embd  = 2048;
    int32_t n_head  = 16;
    int32_t n_layer = 24;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

// 定义 starcoder_layer 结构体
struct starcoder_layer {
    // 归一化
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // 注意力
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // 多层感知机
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};

// 定义 starcoder_model 结构体
struct starcoder_model {
    starcoder_hparams hparams;

    // 归一化
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte;     // 位置嵌入
    struct ggml_tensor * wpe;     //    标记嵌入
    struct ggml_tensor * lm_head; // 语言模型头部

    std::vector<starcoder_layer> layers;

    // 键 + 值 存储
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

// 从文件加载模型的权重
bool starcoder_model_load(const std::string & fname, starcoder_model & model, gpt_vocab & vocab) {
    // 打印加载模型的信息
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    // 以二进制方式打开文件流
    auto fin = std::ifstream(fname, std::ios::binary);
    // 如果文件打开失败，则输出错误信息并返回 false
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // 验证文件的魔数
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        // 如果魔数不等于预期的值，则输出错误信息并返回 false
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // 加载超参数
    {
        auto & hparams = model.hparams;

        // 依次读取并存储超参数的值
        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        // 计算并输出超参数的值
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        // 对 ftype 进行取模操作
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // 加载词汇表
    {
        // 读取模型文件中的词汇表大小
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        // 检查读取的词汇表大小是否与模型参数中的词汇表大小相匹配
        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        // 读取词汇表中的单词并构建词汇表
        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            // 读取单词长度
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            // 调整缓冲区大小以容纳单词
            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            // 将单词添加到词汇表中
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;

            // 如果需要，打印前10个单词
            // if (i < 10) fprintf(stderr, "%.s: vocab[%d] = '%s'\n", __func__, i, word.c_str());
        }

        // 添加StarChat特殊标记到词汇表中
        for (std::string token : {
                "<|system|>",
                "<|user|>",
                "<|assistant|>",
                "<|end|>",
                "<fim-prefix>",
                "<fim-middle>",
                "<fim-suffix>",
                "<fim-pad>",
                "<|end_of_turn|>"
            }) {
            if (vocab.token_to_id.find(token) != vocab.token_to_id.end()) {
                vocab.add_special_token(token);
            }
        }
    }

    // 对于大张量，我们可以选择将数据存储为16位浮点数或量化形式，以节省内存并加快计算速度
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    // 检查模型参数中的数据类型是否有效
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    // 获取模型上下文
    auto & ctx = model.ctx;

    // 初始化上下文大小
    size_t ctx_size = 0;

    }

    // 创建ggml上下文
    {
        // 初始化参数结构体，设置内存大小为ctx_size，内存缓冲区为NULL，允许分配内存
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        // 使用初始化参数结构体初始化模型上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，则输出错误信息并返回false
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 准备权重的内存
    }

    // 键 + 值内存
    {
        // 获取模型超参数
        const auto & hparams = model.hparams;

        // 获取嵌入大小、层数、上下文大小
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        // 计算内存数量和元素数量
        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        // 创建键和值的内存张量
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        // 计算内存大小并输出信息
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);
        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // 加载权重
    }

    // 关闭文件流并返回true
    fin.close();

    return true;
// 评估变换器
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    预测的下一个标记的logits
//   - mem_per_token: 每个标记的内存
//
bool starcoder_eval(
        const starcoder_model & model,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w,
              size_t                     & mem_per_token) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;
    const int n_vocab = hparams.n_vocab;

    static size_t buf_size = 256u*1024*1024;
    static void * buf = malloc(buf_size);

    // 使用2个临时缓冲区
    // TODO: 非常巧妙的解决方案 - 以更优雅的方式重新实现
    static size_t scr0_size = 256u*1024*1024;
    static void * scr0 = malloc(scr0_size);

    static size_t scr1_size = 256u*1024*1024;
    static void * scr1 = malloc(scr1_size);

    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = 1.1*(mem_per_token*N); // 添加10%以考虑ggml对象的开销
        //printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__, buf_size, buf_size_new);

        // 重新分配
        buf_size = buf_size_new;
        buf = realloc(buf, buf_size);
        if (buf == nullptr) {
            fprintf(stderr, "%s: failed to allocate %zu bytes\n", __func__, buf_size);
            return false;
        }
    }

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    struct ggml_context * ctx0 = ggml_init(params);
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);
    // 创建一个新的一维整型张量 embd，长度为 N
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 将 embd_inp 的数据复制到 embd 的数据中，长度为 N*embd 的元素大小
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    // 创建一个新的一维整型张量 position，长度为 N
    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 遍历 position 的数据，将 n_past + i 赋值给每个位置
    for (int i = 0; i < N; ++i) {
        ((int32_t *) position->data)[i] = n_past + i;
    }

    // 计算 wte + wpe，得到 inpL
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));

    }

    // 设置上下文的临时存储区域
    ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

    // 对 inpL 进行归一化处理，使用 hparams.eps 作为参数
    {
        // [ 768, N]
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // 计算 ln_f_g*inpL + ln_f_b，得到新的 inpL
        // [ 768, N]
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    // 清空上下文的临时存储区域
    ggml_set_scratch(ctx0, { 0, 0, nullptr, });

    // 计算 WTE * inpL，得到新的 inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // 将 logits 转换为概率，得到新的 inpL
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 如果 n_past 为 100 的倍数，则打印图形和将图形转储为 dot 文件
    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // 调整 embd_w 的大小为 n_vocab*N
    //memcpy(embd_w.data(), ggml_get_data(inpL), sizeof(float)*n_vocab*N);

    // 仅返回最后一个标记的结果
    embd_w.resize(n_vocab);
    // 将 (n_vocab*(N-1)) 位置开始的数据复制到 embd_w 中
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    // 如果 mem_per_token 为 0，则计算每个标记的内存使用量
    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    //printf("used_mem = %zu MB\n", ggml_used_mem(ctx0)/(1024*1024));

    // 释放上下文的内存
    ggml_free(ctx0);

    // 返回 true
    return true;
}

int main(int argc, char ** argv) {
    ggml_time_init(); // 初始化时间

    const int64_t t_main_start_us = ggml_time_us(); // 获取当前时间，单位为微秒

    gpt_params params; // 创建参数对象

    if (gpt_params_parse(argc, argv, params) == false) { // 解析命令行参数
        return 1; // 如果解析失败，返回错误码1
    }

    if (params.seed < 0) { // 如果种子值小于0
        params.seed = time(NULL); // 使用当前时间作为种子值
    }

    printf("%s: seed = %d\n", __func__, params.seed); // 打印种子值

    std::mt19937 rng(params.seed); // 创建随机数生成器对象
    if (params.prompt.empty()) { // 如果提示为空
        params.prompt = gpt_random_prompt(rng); // 生成随机提示
    }

    int64_t t_load_us = 0; // 初始化加载时间

    gpt_vocab vocab; // 创建词汇表对象
    starcoder_model model; // 创建模型对象

    // 加载模型
    {
        const int64_t t_start_us = ggml_time_us(); // 获取加载开始时间

        if (!starcoder_model_load(params.model, model, vocab)) { // 加载模型
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str()); // 打印加载失败信息
            return 1; // 返回错误码1
        }

        t_load_us = ggml_time_us() - t_start_us; // 计算加载时间

        test_gpt_tokenizer(vocab, params.token_test); // 测试分词器
    }

    if (params.repeat_last_n == -1) { // 如果重复次数为-1
        params.repeat_last_n = model.hparams.n_ctx; // 设置重复次数为上下文长度
    }
    printf("\n");
    printf("%s: temp           = %.3f\n", __func__, params.temp); // 打印温度值
    printf("%s: top_k          = %d\n",   __func__, params.top_k); // 打印top_k值
    printf("%s: top_p          = %.3f\n", __func__, params.top_p); // 打印top_p值
    printf("%s: repeat_last_n  = %d\n",   __func__, params.repeat_last_n); // 打印重复次数
    printf("%s: repeat_penalty = %.3f\n", __func__, params.repeat_penalty); // 打印重复惩罚值

    int n_past = 0; // 初始化过去数量

    int64_t t_sample_us  = 0; // 初始化采样时间
    int64_t t_predict_us = 0; // 初始化预测时间

    std::vector<float> logits; // 创建logits向量

    std::vector<int32_t> last_n_tokens(model.hparams.n_ctx); // 创建上下文长度的token向量
    std::fill(last_n_tokens.begin(), last_n_tokens.end(), 0); // 填充token向量为0

    // 分词提示
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt); // 使用词汇表对提示进行分词

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size()); // 设置预测数量为提示长度和上下文长度的较小值

    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str()); // 打印提示
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size()); // 打印提示中的token数量
    // 遍历 embd_inp 数组，打印每个元素的信息
    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6d, %s\n", __func__, i, embd_inp[i], vocab.id_to_token.at(embd_inp[i]).c_str());
    }
    printf("\n\n");

    // 处理 StarChat "<|end|>" 和 OpenCoder "<|end_of_turn>" 标记
    gpt_vocab::id starchat_end_token = -1;
    {
        // 查找 "<|end|>" 标记的 ID
        const auto it = vocab.token_to_id.find("<|end|>");
        if (it != vocab.token_to_id.end()) {
            starchat_end_token = it->second;
        } else {
            // 如果 "<|end|>" 标记不存在，则查找 "<|end_of_turn|>" 标记的 ID
            const auto eot_token_id = vocab.token_to_id.find("<|end_of_turn|>");
            if (eot_token_id != vocab.token_to_id.end()) {
              starchat_end_token = eot_token_id->second;
            }
        }
    }

    // 逐个提交输入提示的标记
    // 这样可以减少推断过程中的内存使用，但会稍微降低速度
    std::vector<gpt_vocab::id> embd;

    // 确定每个标记所需的推断内存
    size_t mem_per_token = 0;
    starcoder_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token);

    }

    // 报告时间
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    // 释放模型上下文的内存
    ggml_free(model.ctx);

    // 返回 0 表示成功
    return 0;
# 闭合前面的函数定义
```