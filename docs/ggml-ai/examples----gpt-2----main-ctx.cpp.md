# `ggml\examples\gpt-2\main-ctx.cpp`

```
#include "ggml/ggml.h"  // 引入 ggml 库的头文件

#include "common.h"  // 引入 common.h 头文件
#include "common-ggml.h"  // 引入 common-ggml.h 头文件

#include <cassert>  // 断言库
#include <cmath>  // 数学库
#include <cstdio>  // 标准输入输出库
#include <cstring>  // 字符串库
#include <fstream>  // 文件流库
#include <map>  // 映射库
#include <string>  // 字符串库
#include <vector>  // 向量库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 默认超参数 (GPT-2 117M)
struct gpt2_hparams {
    int32_t n_vocab = 50257;  // 词汇量
    int32_t n_ctx   = 1024;  // 上下文长度
    int32_t n_embd  = 768;  // 嵌入维度
    int32_t n_head  = 12;  // 头数
    int32_t n_layer = 12;  // 层数
    int32_t ftype   = 1;  // 特征类型
    float   eps     = 1e-5f;  // epsilon
};

struct gpt2_layer {
    // 归一化
    struct ggml_tensor * ln_1_g;  // Layer Normalization 1 的缩放参数
    struct ggml_tensor * ln_1_b;  // Layer Normalization 1 的偏移参数

    struct ggml_tensor * ln_2_g;  // Layer Normalization 2 的缩放参数
    struct ggml_tensor * ln_2_b;  // Layer Normalization 2 的偏移参数

    // 注意力
    struct ggml_tensor * c_attn_attn_w;  // 注意力层的权重
    struct ggml_tensor * c_attn_attn_b;  // 注意力层的偏置

    struct ggml_tensor * c_attn_proj_w;  // 注意力层的投影权重
    struct ggml_tensor * c_attn_proj_b;  // 注意力层的投影偏置

    // 多层感知机
    struct ggml_tensor * c_mlp_fc_w;  // 多层感知机的全连接层权重
    struct ggml_tensor * c_mlp_fc_b;  // 多层感知机的全连接层偏置

    struct ggml_tensor * c_mlp_proj_w;  // 多层感知机的投影层权重
    struct ggml_tensor * c_mlp_proj_b;  // 多层感知机的投影层偏置
};

struct gpt2_model {
    gpt2_hparams hparams;  // GPT-2 模型的超参数

    // 归一化
    struct ggml_tensor * ln_f_g;  // 最终归一化层的缩放参数
    struct ggml_tensor * ln_f_b;  // 最终归一化层的偏移参数

    struct ggml_tensor * wte;     // 位置嵌入
    struct ggml_tensor * wpe;     //    词嵌入
    struct ggml_tensor * lm_head; // 语言模型头部

    std::vector<gpt2_layer> layers;  // GPT-2 模型的多层结构

    // 键 + 值 存储
    struct ggml_tensor * memory_k;  // 键的存储
    struct ggml_tensor * memory_v;  // 值的存储

    //
    struct ggml_context * ctx;  // 上下文
    std::map<std::string, struct ggml_tensor *> tensors;  // 张量映射
};

// 从文件加载模型的权重
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());  // 打印加载模型的信息

    auto fin = std::ifstream(fname, std::ios::binary);  // 以二进制模式打开文件流
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());  // 如果打开失败，则打印错误信息
        return false;  // 返回加载失败
    }
    // 验证文件的魔数
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));  // 从文件流中读取魔数
        if (magic != GGML_FILE_MAGIC) {  // 如果魔数不等于预设的值
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());  // 输出错误信息
            return false;  // 返回 false
        }
    }

    // 加载超参数
    {
        auto & hparams = model.hparams;  // 获取模型的超参数引用

        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));  // 从文件流中读取词汇量
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));    // 从文件流中读取上下文大小
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));   // 从文件流中读取嵌入维度
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));   // 从文件流中读取头数
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));  // 从文件流中读取层数
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));    // 从文件流中读取特征类型

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;  // 计算量化版本号

        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);  // 输出词汇量
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);    // 输出上下文大小
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);   // 输出嵌入维度
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);   // 输出头数
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);  // 输出层数
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);    // 输出特征类型
        printf("%s: qntvr   = %d\n", __func__, qntvr);            // 输出量化版本号

        hparams.ftype %= GGML_QNT_VERSION_FACTOR;  // 对特征类型进行取模操作
    }

    // 加载词汇表
    {
        // 读取模型文件中的词汇量大小
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        // 检查读取的词汇量是否与模型参数中的词汇量大小相匹配
        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        // 读取词汇表中的单词并建立单词到ID的映射关系
        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            // 读取单词的长度
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            // 调整缓冲区大小以容纳单词
            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            // 建立单词到ID的映射关系
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 为大型张量准备内存，可以选择使用16位浮点数或量化数据以节省内存并加快计算速度
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    // 获取模型的上下文
    auto & ctx = model.ctx;

    // 初始化上下文的大小
    size_t ctx_size = 0;

    }

    // 创建ggml上下文
    {
        // 设置ggml初始化参数
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        // 初始化ggml上下文
        model.ctx = ggml_init(params);
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 为权重准备内存
    }

    // 键值内存
    {
        // 从模型中获取超参数
        const auto & hparams = model.hparams;

        // 获取嵌入层大小、层数和上下文大小
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        // 计算记忆单元数量和元素数量
        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        // 创建记忆键和值的张量
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        // 计算内存大小
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        // 打印内存大小和记忆单元数量
        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // 加载权重
    }

    // 关闭文件流
    fin.close();

    // 返回加载成功
    return true;
// 评估变换器
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    下一个标记的预测对数
//   - mem_per_token: 每个标记的内存占用量

bool gpt2_eval(
        const gpt2_model & model,
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

    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = 1.1*(mem_per_token*N); // 增加10%以考虑 ggml 对象的开销
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

    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    for (int i = 0; i < N; ++i) {
        ((int32_t *) position->data)[i] = n_past + i;
    }

    // 计算输入的词嵌入和位置嵌入的和
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));

    }

    // 归一化
    {
        // [ 768, N]
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        // [ 768, N]
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    // 计算最终的输出
    // inpL = WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // 计算概率
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    //embd_w.resize(n_vocab*N);
    //memcpy(embd_w.data(), ggml_get_data(inpL), sizeof(float)*n_vocab*N);

    // 仅返回最后一个标记的结果
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    //printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    ggml_free(ctx0);

    return true;
}

int main(int argc, char ** argv) {
    ggml_time_init(); // 初始化时间

    const int64_t t_main_start_us = ggml_time_us(); // 获取当前时间

    gpt_params params; // 定义参数对象
    params.model = "models/gpt-2-117M/ggml-model.bin"; // 设置模型路径

    if (gpt_params_parse(argc, argv, params) == false) { // 解析命令行参数
        return 1; // 如果解析失败，返回错误码
    }

    if (params.seed < 0) { // 如果种子值小于0
        params.seed = time(NULL); // 使用当前时间作为种子值
    }

    printf("%s: seed = %d\n", __func__, params.seed); // 打印种子值

    std::mt19937 rng(params.seed); // 使用种子值初始化随机数生成器
    if (params.prompt.empty()) { // 如果提示为空
        params.prompt = gpt_random_prompt(rng); // 生成随机提示
    }

    int64_t t_load_us = 0; // 初始化加载模型时间

    gpt_vocab vocab; // 定义词汇表对象
    gpt2_model model; // 定义模型对象

    // load the model
    {
        const int64_t t_start_us = ggml_time_us(); // 获取加载模型前的时间

        if (!gpt2_model_load(params.model, model, vocab)) { // 加载模型
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str()); // 打印加载模型失败信息
            return 1; // 返回错误码
        }

        t_load_us = ggml_time_us() - t_start_us; // 计算加载模型所用时间

        test_gpt_tokenizer(vocab, params.token_test); // 测试分词器
    }

    int n_past = 0; // 初始化过去的数量

    int64_t t_sample_us  = 0; // 初始化采样时间
    int64_t t_predict_us = 0; // 初始化预测时间

    std::vector<float> logits; // 定义logits向量

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt); // 对提示进行分词

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size()); // 设置预测数量

    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str()); // 打印提示
    printf("%s: number of tokens in prompt = %zu, first 8 tokens: ", __func__, embd_inp.size()); // 打印提示中的token数量
    for (int i = 0; i < std::min(8, (int) embd_inp.size()); i++) { // 遍历前8个token
        printf("%d ", embd_inp[i]); // 打印每个token
    }
    printf("\n\n"); // 换行

    // submit the input prompt token-by-token
    // this reduces the memory usage during inference, at the cost of a bit of speed at the beginning
    std::vector<gpt_vocab::id> embd; // 定义embd向量

    // determine the required inference memory per token:
    size_t mem_per_token = 0; // 初始化每个token所需的推理内存
    gpt2_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token); // 进行模型评估
    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // 从当前嵌入大小开始循环直到输入嵌入大小加上预测数量
        // 预测
        if (embd.size() > 0) {
            // 如果嵌入大小大于0，则进行预测
            const int64_t t_start_us = ggml_time_us();

            // 调用gpt2_eval函数进行预测
            if (!gpt2_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
                printf("Failed to predict\n");
                return 1;
            }

            // 计算预测时间
            t_predict_us += ggml_time_us() - t_start_us;
        }

        // 更新过去的数量
        n_past += embd.size();
        // 清空嵌入
        embd.clear();

        if (i >= embd_inp.size()) {
            // 如果i大于等于输入嵌入大小，则进行下一个标记的采样
            // 设置参数
            const int   top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp  = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;

            {
                const int64_t t_start_sample_us = ggml_time_us();

                // 从logits中进行top-k和top-p采样
                id = gpt_sample_top_k_top_p(vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p, temp, rng);

                // 计算采样时间
                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // 将采样结果添加到上下文中
            embd.push_back(id);
        } else {
            // 如果在这里，意味着我们仍在处理输入提示
            for (size_t k = i; k < embd_inp.size(); k++) {
                embd.push_back(embd_inp[k]);
                if (int32_t(embd.size()) >= params.n_batch) {
                    break;
                }
            }
            i += embd.size() - 1;
        }

        // 显示文本
        for (auto id : embd) {
            printf("%s", vocab.id_to_token[id].c_str());
        }
        fflush(stdout);

        // 文本结束标记
        if (embd.back() == 50256) {
            break;
        }
    }

    // 报告时间
    {
        // 计算主函数结束时间的微秒数
        const int64_t t_main_end_us = ggml_time_us();

        // 打印内存每个标记的字节数
        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        // 打印加载时间
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        // 打印采样时间
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        // 打印预测时间和每个标记的平均预测时间
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);
        // 打印总时间
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    // 释放模型上下文
    ggml_free(model.ctx);

    // 返回 0 表示成功
    return 0;
# 闭合前面的函数定义
```