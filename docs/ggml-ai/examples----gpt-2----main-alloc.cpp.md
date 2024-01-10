# `ggml\examples\gpt-2\main-alloc.cpp`

```
#include "ggml/ggml.h"  // 引入 ggml 库的头文件
#include "ggml/ggml-alloc.h"  // 引入 ggml-alloc 库的头文件

#include "common.h"  // 引入 common 库的头文件
#include "common-ggml.h"  // 引入 common-ggml 库的头文件

#include <cassert>  // 引入断言库
#include <cmath>  // 引入数学库
#include <cstdio>  // 引入标准输入输出库
#include <cstring>  // 引入字符串库
#include <fstream>  // 引入文件流库
#include <map>  // 引入映射库
#include <string>  // 引入字符串库
#include <vector>  // 引入向量库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 默认超参数 (GPT-2 117M)
struct gpt2_hparams {
    int32_t n_vocab = 50257;  // 词汇量大小
    int32_t n_ctx   = 1024;  // 上下文大小
    int32_t n_embd  = 768;  // 嵌入维度
    int32_t n_head  = 12;  // 头数
    int32_t n_layer = 12;  // 层数
    int32_t ftype   = 1;  // 特征类型
    float   eps     = 1e-5f;  // 微小值
};

struct gpt2_layer {
    // 归一化
    struct ggml_tensor * ln_1_g;  // 归一化参数
    struct ggml_tensor * ln_1_b;  // 归一化偏置

    struct ggml_tensor * ln_2_g;  // 归一化参数
    struct ggml_tensor * ln_2_b;  // 归一化偏置

    // 注意力
    struct ggml_tensor * c_attn_attn_w;  // 注意力权重
    struct ggml_tensor * c_attn_attn_b;  // 注意力偏置

    struct ggml_tensor * c_attn_proj_w;  // 注意力投影权重
    struct ggml_tensor * c_attn_proj_b;  // 注意力投影偏置

    // 多层感知机
    struct ggml_tensor * c_mlp_fc_w;  // 多层感知机全连接层权重
    struct ggml_tensor * c_mlp_fc_b;  // 多层感知机全连接层偏置

    struct ggml_tensor * c_mlp_proj_w;  // 多层感知机投影层权重
    struct ggml_tensor * c_mlp_proj_b;  // 多层感知机投影层偏置
};

struct gpt2_model {
    gpt2_hparams hparams;  // GPT-2 模型的超参数

    // 归一化
    struct ggml_tensor * ln_f_g;  // 归一化参数
    struct ggml_tensor * ln_f_b;  // 归一化偏置

    struct ggml_tensor * wte;     // 位置嵌入
    struct ggml_tensor * wpe;     //    词嵌入
    struct ggml_tensor * lm_head; // 语言模型头部

    std::vector<gpt2_layer> layers;  // GPT-2 模型的多层

    // 键 + 值 存储
    struct ggml_tensor * memory_k;  // 键存储
    struct ggml_tensor * memory_v;  // 值存储

    //
    struct ggml_context * ctx;  // 上下文
    std::map<std::string, struct ggml_tensor *> tensors;  // 张量映射
};

// 从文件加载模型的权重
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());  // 打印加载模型的信息

    auto fin = std::ifstream(fname, std::ios::binary);  // 以二进制方式打开文件流
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

        // 依次读取超参数的值
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
    // 构建计算图
    struct ggml_cgraph * gpt2_graph(
            const gpt2_model & model,
            struct ggml_allocr * allocr,
            const int n_past,
            const std::vector<gpt_vocab::id> & embd_inp) {
        const int N = embd_inp.size();

        const auto & hparams = model.hparams;

        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_head  = hparams.n_head;

        // 由于我们使用了 ggml-alloc，这个缓冲区只需要足够的空间来容纳 ggml_tensor 和 ggml_cgraph 结构，而不是张量数据
        static size_t buf_size = ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead();
        static std::vector<uint8_t> buf(buf_size);

        struct ggml_init_params params = {
            /*.mem_size   =*/ buf_size,
            /*.mem_buffer =*/ buf.data(),
            /*.no_alloc   =*/ true, // 张量将在后面由 ggml_allocr_alloc_graph() 分配
        };

        struct ggml_context * ctx0 = ggml_init(params);

        struct ggml_cgraph  * gf = ggml_new_graph(ctx0);

        struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
        ggml_allocr_alloc(allocr, embd);

        // 如果我们只是测量内存使用情况，则避免写入张量
        if (!ggml_allocr_is_measure(allocr)) {
            memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));
        }

        struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
        ggml_allocr_alloc(allocr, position);
        if (!ggml_allocr_is_measure(allocr)) {
            for (int i = 0; i < N; ++i) {
                ((int32_t *) position->data)[i] = n_past + i;
            }
        }

        // wte + wpe
        struct ggml_tensor * inpL =
            ggml_add(ctx0,
                    ggml_get_rows(ctx0, model.wte, embd),
                    ggml_get_rows(ctx0, model.wpe, position));

        }

        // norm
    }
    {
        // 对输入进行规范化处理，返回规范化后的结果
        // [ 768, N]
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // 对输入进行线性变换和偏置处理
        // inpL = ln_f_g*inpL + ln_f_b
        // [ 768, N]
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    // 对输入进行矩阵乘法运算
    // inpL = WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // 将逻辑回归转换为概率
    //inpL = ggml_soft_max(ctx0, inpL);

    // 构建前向传播扩展
    ggml_build_forward_expand(gf, inpL);

    // 释放上下文资源
    ggml_free(ctx0);

    // 返回前向传播结果
    return gf;
}
// 评估变换器
//
//   - model:     模型
//   - allocr:    用于分配计算缓冲区的 ggml_allocr
//   - n_threads: 要使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    下一个标记的预测对数
//
bool gpt2_eval(
        const gpt2_model & model,
        struct ggml_allocr * allocr,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_vocab = hparams.n_vocab;

    // 重置分配器以释放上一次推断期间分配的所有内存
    ggml_allocr_reset(allocr);

    // 创建计算图
    struct ggml_cgraph * gf = gpt2_graph(model, allocr, n_past, embd_inp);

    // 分配张量
    ggml_allocr_alloc_graph(allocr, gf);

    // 运行计算
    struct ggml_cplan plan = ggml_graph_plan(gf, n_threads);
    static std::vector<uint8_t> work_buffer;
    work_buffer.resize(plan.work_size);
    plan.work_data = work_buffer.data();
    ggml_graph_compute(gf, &plan);

    // 如果 (n_past%100 == 0) {
    //     ggml_graph_print   (&gf);
    //     ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    // }

    // 在这种情况下，输出张量是图中的最后一个张量
    struct ggml_tensor * inpL = gf->nodes[gf->n_nodes - 1];

    // 仅返回最后一个标记的结果
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    return true;
}

int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "models/gpt-2-117M/ggml-model.bin";
    // 解析命令行参数，如果解析失败则返回1
    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果参数中的种子值小于0，则将其设置为当前时间
    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    // 打印种子值
    printf("%s: seed = %d\n", __func__, params.seed);

    // 使用种子值初始化随机数生成器
    std::mt19937 rng(params.seed);

    // 如果提示为空，则生成随机提示
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    // 初始化变量 t_load_us
    int64_t t_load_us = 0;

    // 初始化词汇表和模型
    gpt_vocab vocab;
    gpt2_model model;

    // 加载模型
    {
        // 记录加载模型开始时间
        const int64_t t_start_us = ggml_time_us();

        // 如果加载模型失败，则打印错误信息并返回1
        if (!gpt2_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        // 计算加载模型所用时间
        t_load_us = ggml_time_us() - t_start_us;

        // 测试 GPT 分词器
        test_gpt_tokenizer(vocab, params.token_test);
    }

    // 在评估模型时保持此缓冲区有效
    std::vector<uint8_t> compute_buffer;

    // 分配计算缓冲区
    struct ggml_allocr * allocr = NULL;
    {
        allocr = ggml_allocr_new_measure(GGML_MEM_ALIGN);

        // 创建内存使用估计的最坏情况图形
        int n_tokens = std::min(model.hparams.n_ctx, params.n_batch);
        int n_past = model.hparams.n_ctx - n_tokens;
        struct ggml_cgraph * gf = gpt2_graph(model, allocr, n_past, std::vector<gpt_vocab::id>(n_tokens, 0));

        // 计算所需内存
        size_t mem_size = ggml_allocr_alloc_graph(allocr, gf) + GGML_MEM_ALIGN;

        // 重新创建具有所需内存的分配器
        ggml_allocr_free(allocr);
        compute_buffer.resize(mem_size);
        allocr = ggml_allocr_new(compute_buffer.data(), mem_size, GGML_MEM_ALIGN);

        // 打印计算缓冲区大小
        fprintf(stderr, "%s: compute buffer size: %.2f MB\n", __func__, mem_size/1024.0/1024.0);
    }

    // 初始化变量 n_past
    int n_past = 0;

    // 初始化变量 t_sample_us 和 t_predict_us
    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    // 初始化 logits
    std::vector<float> logits;

    // 对提示进行分词
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);
    # 将 params.n_predict 设置为 params.n_predict 和 model.hparams.n_ctx - (int) embd_inp.size() 中的较小值
    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    # 打印函数名称和提示信息
    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
    # 打印提示信息中的标记数量以及前8个标记
    printf("%s: number of tokens in prompt = %zu, first 8 tokens: ", __func__, embd_inp.size());
    for (int i = 0; i < std::min(8, (int) embd_inp.size()); i++) {
        printf("%d ", embd_inp[i]);
    }
    printf("\n\n");

    # 逐个提交输入提示的标记
    # 这样可以在推断过程中减少内存使用，但会稍微降低速度
    std::vector<gpt_vocab::id> embd;
    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // 从当前嵌入大小开始循环直到达到输入嵌入大小加上预测数量
        // 预测
        if (embd.size() > 0) {
            // 如果嵌入大小大于0，则进行预测
            const int64_t t_start_us = ggml_time_us();

            // 使用GPT-2模型进行评估，获取预测结果
            if (!gpt2_eval(model, allocr, params.n_threads, n_past, embd, logits)) {
                printf("Failed to predict\n");
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;
        }

        n_past += embd.size();  // 更新过去的嵌入大小
        embd.clear();  // 清空嵌入数据

        if (i >= embd_inp.size()) {
            // 采样下一个标记
            const int   top_k = params.top_k;  // 获取top_k参数
            const float top_p = params.top_p;  // 获取top_p参数
            const float temp  = params.temp;   // 获取温度参数

            const int n_vocab = model.hparams.n_vocab;  // 获取词汇表大小

            gpt_vocab::id id = 0;  // 初始化标记id

            {
                const int64_t t_start_sample_us = ggml_time_us();

                // 使用top_k和top_p进行采样，获取下一个标记的id
                id = gpt_sample_top_k_top_p(vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p, temp, rng);

                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // 将下一个标记添加到上下文中
            embd.push_back(id);
        } else {
            // 如果在这里，意味着我们仍在处理输入提示
            for (size_t k = i; k < embd_inp.size(); k++) {
                embd.push_back(embd_inp[k]);  // 将输入嵌入添加到上下文中
                if (int32_t(embd.size()) >= params.n_batch) {
                    break;
                }
            }
            i += embd.size() - 1;  // 更新i的值
        }

        // 显示文本
        for (auto id : embd) {
            printf("%s", vocab.id_to_token[id].c_str());  // 打印标记对应的文本
        }
        fflush(stdout);  // 刷新标准输出

        // 文本结束标记
        if (embd.back() == 50256) {
            break;  // 如果遇到文本结束标记，则跳出循环
        }
    }

    // 报告时间
    {
        // 记录主函数结束的时间戳
        const int64_t t_main_end_us = ggml_time_us();

        // 打印加载时间
        printf("\n\n");
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

    // 返回成功状态码
    return 0;
# 闭合前面的函数定义
```