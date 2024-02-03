# `ggml\examples\gpt-j\main.cpp`

```cpp
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

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif


// default hparams (GPT-J 6B)
struct gptj_hparams {
    int32_t n_vocab = 50400;  // 默认词汇量
    int32_t n_ctx   = 2048;   // 默认上下文长度
    int32_t n_embd  = 4096;   // 默认嵌入维度
    int32_t n_head  = 16;     // 默认头数
    int32_t n_layer = 28;     // 默认层数
    int32_t n_rot   = 64;     // 默认旋转数
    int32_t ftype   = 1;      // 默认特征类型
    float   eps     = 1e-5f;  // 默认 epsilon
};

struct gptj_layer {
    // normalization
    struct ggml_tensor * ln_1_g;  // 归一化参数 g
    struct ggml_tensor * ln_1_b;  // 归一化参数 b

    // attention
    struct ggml_tensor * c_attn_q_proj_w;  // 注意力机制参数 q_proj_w
    struct ggml_tensor * c_attn_k_proj_w;  // 注意力机制参数 k_proj_w
    struct ggml_tensor * c_attn_v_proj_w;  // 注意力机制参数 v_proj_w

    struct ggml_tensor * c_attn_proj_w;     // 注意力机制参数 proj_w

    // ff
    struct ggml_tensor * c_mlp_fc_w;        // 前馈神经网络参数 fc_w
    struct ggml_tensor * c_mlp_fc_b;        // 前馈神经网络参数 fc_b

    struct ggml_tensor * c_mlp_proj_w;      // 前馈神经网络参数 proj_w
    struct ggml_tensor * c_mlp_proj_b;      // 前馈神经网络参数 proj_b
};

struct gptj_model {
    gptj_hparams hparams;  // GPT-J 模型的超参数

    // normalization
    struct ggml_tensor * ln_f_g;  // 归一化参数 g
    struct ggml_tensor * ln_f_b;  // 归一化参数 b

    struct ggml_tensor * wte;     // 位置嵌入

    struct ggml_tensor * lmh_g;   // 语言模型头部参数 g
    struct ggml_tensor * lmh_b;   // 语言模型头部参数 b

    std::vector<gptj_layer> layers;  // GPT-J 模型的层

    // key + value memory
    struct ggml_tensor * memory_k;  // 关键字记忆
    struct ggml_tensor * memory_v;  // 值记忆

    //
    struct ggml_context * ctx;  // 上下文
    std::map<std::string, struct ggml_tensor *> tensors;  // 张量映射
};

// load the model's weights from a file
bool gptj_model_load(const std::string & fname, gptj_model & model, gpt_vocab & vocab) {
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);  // 以二进制方式打开文件流
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());  // 打开文件失败时输出错误信息
        return false;
    }

    // verify magic
    // 读取文件头部的 magic 数值
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        // 检查 magic 数值是否等于预定义的 GGML_FILE_MAGIC
        if (magic != GGML_FILE_MAGIC) {
            // 如果不等于，输出错误信息并返回 false
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // 加载模型超参数
    {
        auto & hparams = model.hparams;

        // 依次读取模型超参数的值
        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fin.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        // 计算 qntvr 值
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        // 输出模型超参数的值
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: n_rot   = %d\n", __func__, hparams.n_rot);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        // 对 ftype 进行取模操作
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // 加载词汇表
    {
        // 读取模型文件中的词汇量大小
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        // 如果读取的词汇量大小与模型参数中的词汇量大小不一致，则输出错误信息并返回false
        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        // 读取词汇表中的词语，并建立词语到ID的映射关系
        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            // 读取词语的长度
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            // 调整缓冲区大小，读取词语内容
            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            // 建立词语到ID的映射关系
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 对于大张量，可以选择将数据存储为16位浮点数或量化形式，以节省内存并加快计算速度
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    // 如果转换后的数据类型无效，则输出错误信息并返回false
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    // 获取模型的上下文
    auto & ctx = model.ctx;

    // 上下文大小初始化为0
    size_t ctx_size = 0;
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
    
        // 从超参数中获取嵌入维度、层数、上下文大小和词汇表大小
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;
    
        // 计算 ln_f_g 和 ln_f_b 的上下文大小
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_g
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_b
    
        // 计算 wte 的上下文大小
        ctx_size += ggml_row_size(wtype, n_embd*n_vocab); // wte
    
        // 计算 lmh_g 和 lmh_b 的上下文大小
        ctx_size += ggml_row_size(wtype,         n_embd*n_vocab); // lmh_g
        ctx_size += ggml_row_size(GGML_TYPE_F32,        n_vocab); // lmh_b
    
        // 计算 ln_1_g 和 ln_1_b 的上下文大小
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_g
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_b
    
        // 计算注意力机制相关参数的上下文大小
        ctx_size += n_layer*(ggml_row_size(wtype, n_embd*n_embd)); // c_attn_q_proj_w
        ctx_size += n_layer*(ggml_row_size(wtype, n_embd*n_embd)); // c_attn_k_proj_w
        ctx_size += n_layer*(ggml_row_size(wtype, n_embd*n_embd)); // c_attn_v_proj_w
        ctx_size += n_layer*(ggml_row_size(wtype, n_embd*n_embd)); // c_attn_proj_w
    
        // 计算多层感知机相关参数的上下文大小
        ctx_size += n_layer*(ggml_row_size(wtype,         4*n_embd*n_embd)); // c_mlp_fc_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 4*n_embd));        // c_mlp_fc_b
        ctx_size += n_layer*(ggml_row_size(wtype,         4*n_embd*n_embd)); // c_mlp_proj_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32,   n_embd));        // c_mlp_proj_b
    
        // 计算记忆相关参数的上下文大小
        ctx_size += n_ctx*n_layer*ggml_row_size(GGML_TYPE_F16, n_embd); // memory_k
        ctx_size += n_ctx*n_layer*ggml_row_size(GGML_TYPE_F16, n_embd); // memory_v
    
        // 计算对象开销的上下文大小
        ctx_size += (5 + 10*n_layer)*512; // object overhead
    
        // 打印 ggml 上下文的大小
        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0);
    }
    
    // 创建 ggml 上下文
    {
        // 初始化参数结构体，设置内存大小为ctx_size，内存缓冲区为空，允许分配内存
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        // 使用初始化参数结构体创建ggml上下文
        model.ctx = ggml_init(params);
        // 如果ggml上下文创建失败，则打印错误信息并返回false
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 准备权重的内存
    }

    // 键值内存
    {
        // 获取模型超参数
        const auto & hparams = model.hparams;

        // 获取超参数中的嵌入大小、层数和上下文大小
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        // 计算内存数量和元素数量
        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        // 使用ggml_new_tensor_1d函数为键和值分配内存
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);

        // 计算内存大小并打印信息
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);
        printf("%s: memory_size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // 加载权重
    }

    // 关闭文件流
    fin.close();

    // 返回true
    return true;
// 评估变压器
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    下一个标记的预测对数
//
// GPT-J 模型每个输入标记大约需要 16MB 的内存。
//
bool gptj_eval(
        const gptj_model & model,
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
    const int n_rot   = hparams.n_rot;

    static size_t buf_size = 256u*1024*1024;
    static void * buf = malloc(buf_size);

    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = 1.1*(mem_per_token*N); // 添加 10% 以考虑 ggml 对象的开销
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

    // KQ_pos - 包含位置的张量
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    int * data = (int *) KQ_pos->data;
    // 使用循环将 n_past + i 的值赋给 data 数组的每个元素
    for (int i = 0; i < N; ++i) {
        data[i] = n_past + i;
    }

    // 创建一个一维整型张量 embd，长度为 N
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 将 embd_inp 的数据复制到 embd 的数据中
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    // 获取 model.wte 中 embd 对应的行，存储在 inpL 中
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte, embd);

    // 对 inpL 进行归一化处理，使用 hparams.eps 作为参数
    inpL = ggml_norm(ctx0, inpL, hparams.eps);

    // 对 inpL 进行 ln_f_g*inpL + ln_f_b 的运算
    inpL = ggml_add(ctx0,
            ggml_mul(ctx0,
                ggml_repeat(ctx0, model.ln_f_g, inpL),
                inpL),
            ggml_repeat(ctx0, model.ln_f_b, inpL));

    // 对 inpL 进行 lmh_g*inpL + lmh_b 的运算
    inpL = ggml_mul_mat(ctx0, model.lmh_g, inpL);
    inpL = ggml_add(ctx0,
            ggml_repeat(ctx0, model.lmh_b, inpL),
            inpL);

    // 将 logits 转换为概率分布，存储在 inpL 中
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 将最后一个标记对应的结果存储在 embd_w 中
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    // 如果 mem_per_token 为 0，则将 ggml_used_mem(ctx0)/N 的值赋给 mem_per_token
    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    // 打印使用的内存大小
    //printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    // 释放上下文 ctx0
    ggml_free(ctx0);

    // 返回 true
    return true;
}

int main(int argc, char ** argv) {
    ggml_time_init();  // 初始化时间

    const int64_t t_main_start_us = ggml_time_us();  // 获取当前时间

    gpt_params params;  // 定义参数对象
    params.model = "models/gpt-j-6B/ggml-model.bin";  // 设置模型路径

    if (gpt_params_parse(argc, argv, params) == false) {  // 解析命令行参数
        return 1;  // 如果解析失败，返回错误
    }

    if (params.seed < 0) {  // 如果种子值小于0
        params.seed = time(NULL);  // 使用当前时间作为种子值
    }

    printf("%s: seed = %d\n", __func__, params.seed);  // 打印种子值

    std::mt19937 rng(params.seed);  // 使用种子值初始化随机数生成器
    if (params.prompt.empty()) {  // 如果提示为空
        params.prompt = gpt_random_prompt(rng);  // 生成随机提示
    }

    int64_t t_load_us = 0;  // 初始化加载模型时间

    gpt_vocab vocab;  // 定义词汇表对象
    gptj_model model;  // 定义模型对象

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();  // 获取加载模型前的时间

        if (!gptj_model_load(params.model, model, vocab)) {  // 加载模型
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());  // 如果加载失败，打印错误信息
            return 1;  // 返回错误
        }

        t_load_us = ggml_time_us() - t_start_us;  // 计算加载模型所需时间

        test_gpt_tokenizer(vocab, params.token_test);  // 测试分词器
    }

    int n_past = 0;  // 初始化过去的数量

    int64_t t_sample_us  = 0;  // 初始化采样时间
    int64_t t_predict_us = 0;  // 初始化预测时间

    std::vector<float> logits;  // 定义logits向量

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);  // 对提示进行分词

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());  // 设置预测数量

    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());  // 打印提示中的标记数量
    printf("\n");  // 打印空行

    std::vector<gpt_vocab::id> embd;  // 定义embd向量

    // determine the required inference memory per token:
    size_t mem_per_token = 0;  // 初始化每个标记所需的推理内存
    gptj_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token);  // 进行模型评估
}
    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // 从当前嵌入大小开始循环直到嵌入输入大小加上预测数
        // 预测
        if (embd.size() > 0) {
            // 如果嵌入大小大于0
            const int64_t t_start_us = ggml_time_us();
            // 记录开始时间

            if (!gptj_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
                // 如果无法进行预测
                printf("Failed to predict\n");
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;
            // 计算预测时间
        }

        n_past += embd.size();
        // 更新过去的嵌入大小
        embd.clear();
        // 清空嵌入

        if (i >= embd_inp.size()) {
            // 如果i大于等于嵌入输入大小
            // 采样下一个标记
            const int   top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp  = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;
            // 初始化标记id

            {
                const int64_t t_start_sample_us = ggml_time_us();
                // 记录开始采样时间

                id = gpt_sample_top_k_top_p(vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p, temp, rng);
                // 通过top-k和top-p采样下一个标记

                t_sample_us += ggml_time_us() - t_start_sample_us;
                // 计算采样时间
            }

            // 将其添加到上下文
            embd.push_back(id);
            // 将标记id添加到嵌入中
        } else {
            // 如果在这里，意味着我们仍在处理输入提示
            for (size_t k = i; k < embd_inp.size(); k++) {
                embd.push_back(embd_inp[k]);
                // 将输入提示中的标记添加到嵌入中
                if (int32_t(embd.size()) > params.n_batch) {
                    break;
                }
            }
            i += embd.size() - 1;
            // 更新i的值
        }

        // 显示文本
        for (auto id : embd) {
            printf("%s", vocab.id_to_token[id].c_str());
            // 打印标记对应的文本
        }
        fflush(stdout);
        // 清空输出缓冲区

        // 文本结束标记
        if (embd.back() == 50256) {
            break;
        }
        // 如果嵌入的最后一个标记是50256，则跳出循环
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