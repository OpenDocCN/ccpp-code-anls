# `ggml\examples\gpt-neox\main.cpp`

```cpp
#include "ggml/ggml.h"  // 引入 ggml 库的头文件

#include "common.h"  // 引入 common.h 头文件
#include "common-ggml.h"  // 引入 common-ggml.h 头文件

#include <cassert>  // 断言库
#include <cmath>  // 数学函数库
#include <cstdio>  // 标准输入输出库
#include <cstring>  // 字符串处理库
#include <cinttypes>  // 格式化输入输出库
#include <fstream>  // 文件输入输出库
#include <map>  // 映射容器库
#include <string>  // 字符串库
#include <vector>  // 动态数组库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 默认超参数 (StableLM 3B)
struct gpt_neox_hparams {
    int32_t n_vocab = 50257;  // 词汇量大小
    int32_t n_ctx   = 4096;  // 上下文大小
    int32_t n_embd  = 4096;  // 嵌入维度
    int32_t n_head  = 32;  // 头数
    int32_t n_layer = 16;  // 层数
    int32_t n_rot   = 32; // rotary_pct * (n_embd / n_head) 旋转数
    int32_t par_res = 1; // 1 = true, 0 = false  // 是否使用参数共享
    int32_t ftype   = 1;  // 特征类型
    float   eps     = 1e-5f;  // 微小值
};

struct gpt_neox_layer {
    // 预归一化
    struct ggml_tensor * ln_1_g;  // Layer Normalization 1 的缩放参数
    struct ggml_tensor * ln_1_b;  // Layer Normalization 1 的偏移参数

    // 注意力
    struct ggml_tensor * c_attn_attn_w;  // 注意力层的权重
    struct ggml_tensor * c_attn_attn_b;  // 注意力层的偏置

    struct ggml_tensor * c_attn_proj_w;  // 注意力层的投影权重
    struct ggml_tensor * c_attn_proj_b;  // 注意力层的投影偏置

    // 后归一化
    struct ggml_tensor * ln_2_g;  // Layer Normalization 2 的缩放参数
    struct ggml_tensor * ln_2_b;  // Layer Normalization 2 的偏移参数

    // 前馈神经网络
    struct ggml_tensor * c_mlp_fc_w;  // 前馈神经网络的全连接层权重
    struct ggml_tensor * c_mlp_fc_b;  // 前馈神经网络的全连接层偏置

    struct ggml_tensor * c_mlp_proj_w;  // 前馈神经网络的投影层权重
    struct ggml_tensor * c_mlp_proj_b;  // 前馈神经网络的投影层偏置
};

struct gpt_neox_model {
    gpt_neox_hparams hparams;  // 模型的超参数

    // 归一化
    struct ggml_tensor * ln_f_g;  // 最终归一化的缩放参数
    struct ggml_tensor * ln_f_b;  // 最终归一化的偏移参数

    struct ggml_tensor * wte; // 位置嵌入

    struct ggml_tensor * lmh_g; // 语言模型头
    //struct ggml_tensor * lmh_b; // 语言模型偏置

    std::vector<gpt_neox_layer> layers;  // 多层的 gpt_neox_layer

    // 键 + 值 内存
    struct ggml_tensor * memory_k;  // 键的内存
    struct ggml_tensor * memory_v;  // 值的内存

    //
    struct ggml_context * ctx;  // 上下文
    std::map<std::string, struct ggml_tensor *> tensors;  // 字符串到张量的映射
};

// 从文件加载模型的权重
bool gpt_neox_model_load(const std::string & fname, gpt_neox_model & model, gpt_vocab & vocab) {
    # 打印加载模型的提示信息，包括函数名和文件名
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str());

    # 打开给定文件名的二进制文件流
    auto fin = std::ifstream(fname, std::ios::binary);
    # 如果文件流打开失败，则打印错误信息并返回 false
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    # 验证文件的魔数
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        # 如果魔数不等于预期的值，则打印错误信息并返回 false
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    # 加载超参数
    {
        auto & hparams = model.hparams;

        # 依次读取超参数的值
        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fin.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        fin.read((char *) &hparams.par_res, sizeof(hparams.par_res));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        # 计算 qntvr 的值
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        # 打印各个超参数的值
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: n_rot   = %d\n", __func__, hparams.n_rot);
        printf("%s: par_res = %d\n", __func__, hparams.par_res);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        # 对 ftype 进行取模操作
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    # 加载词汇表
    {
        // 定义词汇表大小
        const int32_t n_vocab = model.hparams.n_vocab;

        // 定义单词和缓冲区
        std::string word;
        std::vector<char> buf(128);

        // 遍历词汇表
        for (int i = 0; i < n_vocab; i++) {
            // 读取单词长度
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            // 调整缓冲区大小并读取单词数据
            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            // 将单词和对应的 ID 存入词汇表
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 对于大张量，我们可以选择将数据存储为16位浮点数或量化形式
    // 以节省内存并加快计算速度
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        // 打印错误信息并返回
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    // 获取模型上下文
    auto & ctx = model.ctx;

    // 初始化上下文大小
    size_t ctx_size = 0;
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
    
        // 获取超参数中的各个数值
        const size_t n_embd  = hparams.n_embd;
        const size_t n_layer = hparams.n_layer;
        const size_t n_ctx   = hparams.n_ctx;
        const size_t n_vocab = hparams.n_vocab;
    
        // 计算并累加 ln_f_g 和 ln_f_b 的内存占用
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_g
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_b
    
        // 计算并累加 wte 的内存占用
        ctx_size += ggml_row_size(wtype, n_embd*n_vocab); // wte
    
        // 计算并累加 lmh_g 的内存占用
        ctx_size += ggml_row_size(wtype, n_embd*n_vocab); // lmh_g
        //ctx_size += ggml_row_size(GGML_TYPE_F32, n_vocab); // lmh_b
    
        // 计算并累加 ln_1_g 和 ln_1_b 的内存占用
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_g
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_b
    
        // 计算并累加 c_attn_attn_w 和 c_attn_attn_b 的内存占用
        ctx_size += n_layer*(ggml_row_size(wtype, 3*n_embd*n_embd)); // c_attn_attn_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 3*n_embd)); // c_attn_attn_b
    
        // 计算并累加 c_attn_proj_w 和 c_attn_proj_b 的内存占用
        ctx_size += n_layer*(ggml_row_size(wtype, n_embd*n_embd)); // c_attn_proj_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd*n_embd)); // c_attn_proj_b
    
        // 计算并累加 ln_2_g 和 ln_2_b 的内存占用
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_g
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_b
    
        // 计算并累加 c_mlp_fc_w 和 c_mlp_fc_b 的内存占用
        ctx_size += n_layer*(ggml_row_size(wtype, 4*n_embd*n_embd)); // c_mlp_fc_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 4*n_embd)); // c_mlp_fc_b
    
        // 计算并累加 c_mlp_proj_w 和 c_mlp_proj_b 的内存占用
        ctx_size += n_layer*(ggml_row_size(wtype, 4*n_embd*n_embd)); // c_mlp_proj_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // c_mlp_proj_b
    
        // 计算并累加 memory_k 和 memory_v 的内存占用
        ctx_size += n_ctx*n_layer*ggml_row_size(GGML_TYPE_F32, n_embd); // memory_k
        ctx_size += n_ctx*n_layer*ggml_row_size(GGML_TYPE_F32, n_embd); // memory_v
    
        // 计算并累加对象开销的内存占用
        ctx_size += (6 + 16*n_layer)*1024; // object overhead
    
        // 打印 ggml 上下文的内存占用情况
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

        // 使用初始化参数初始化模型上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，打印错误信息并返回false
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

        // 获取嵌入大小、层数和上下文大小
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        // 计算内存大小和元素个数
        const int64_t n_mem      = n_layer*n_ctx;
        const int64_t n_elements = n_embd*n_mem;

        // 创建键和值的内存张量
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);

        // 计算内存大小并打印信息
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);
        printf("%s: memory_size = %8.2f MB, n_mem = %" PRId64 "\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // 加载权重
    }

    // 关闭文件流
    fin.close();

    // 返回true
    return true;
// feed-forward network
// 前馈网络函数，接受一个层对象、上下文指针、输入张量和浮点数 eps 作为参数
ggml_tensor * gpt_neox_ff(
        const gpt_neox_layer & layer,  // 层对象
        ggml_context         * ctx0,   // 上下文指针
        ggml_tensor          * inp,    // 输入张量
        float                  eps) {  // 浮点数 eps
    ggml_tensor * cur = ggml_norm(ctx0, inp, eps);  // 对输入张量进行归一化处理

    cur = ggml_add(ctx0,  // 对当前张量进行加法操作
        ggml_mul(ctx0,  // 对当前张量进行乘法操作
            ggml_repeat(ctx0, layer.ln_2_g, cur),  // 对当前张量进行重复操作
            cur),  // 当前张量
        ggml_repeat(ctx0, layer.ln_2_b, cur));  // 对当前张量进行重复操作

    cur = ggml_mul_mat(ctx0,  // 对当前张量进行矩阵乘法操作
            layer.c_mlp_fc_w,  // 多层感知机全连接层权重
            cur);  // 当前张量

    cur = ggml_add(ctx0,  // 对当前张量进行加法操作
            ggml_repeat(ctx0, layer.c_mlp_fc_b, cur),  // 对当前张量进行重复操作
            cur);  // 当前张量

    // GELU activation
    // GELU 激活函数
    cur = ggml_gelu(ctx0, cur);

    // projection
    // 投影
    // cur = proj_w*cur + proj_b
    cur = ggml_mul_mat(ctx0,  // 对当前张量进行矩阵乘法操作
            layer.c_mlp_proj_w,  // 多层感知机投影层权重
            cur);  // 当前张量

    cur = ggml_add(ctx0,  // 对当前张量进行加法操作
            ggml_repeat(ctx0, layer.c_mlp_proj_b, cur),  // 对当前张量进行重复操作
            cur);  // 当前张量
    return cur;  // 返回当前张量
}

// evaluate the transformer
// 评估变压器
//
//   - model:     the model
//   - n_threads: number of threads to use
//   - n_past:    the context size so far
//   - embd_inp:  the embeddings of the tokens in the context
//   - embd_w:    the predicted logits for the next token
//
bool gpt_neox_eval(
        const gpt_neox_model & model,  // 变压器模型
        const int n_threads,  // 使用的线程数
        const int n_past,  // 到目前为止的上下文大小
        const std::vector<gpt_vocab::id> & embd_inp,  // 上下文中标记的嵌入
              std::vector<float>         & embd_w,  // 下一个标记的预测对数
              size_t                     & mem_per_token) {  // 每个标记的内存
    const int N = embd_inp.size();  // 上下文中标记的数量

    const auto & hparams = model.hparams;  // 模型的超参数

    const int n_embd  = hparams.n_embd;  // 嵌入维度
    const int n_layer = hparams.n_layer;  // 层数
    const int n_ctx   = hparams.n_ctx;  // 上下文大小
    const int n_head  = hparams.n_head;  // 头数
    const int n_vocab = hparams.n_vocab;  // 词汇量大小
    const int n_rot   = hparams.n_rot;  // 旋转数

    static size_t buf_size = 256u*1024*1024;  // 缓冲区大小
    static void * buf = malloc(buf_size);  // 分配缓冲区内存

    // use 2 scratch buffers
    // 使用 2 个临时缓冲区
    // TODO: very hacky solution - reimplement in a more elegant way
    // TODO: 非常巧妙的解决方案 - 以更优雅的方式重新实现
    static size_t scr0_size = 256u*1024*1024;  // 临时缓冲区大小
    // 分配内存并初始化静态指针 scr0
    static void * scr0 = malloc(scr0_size);

    // 分配内存并初始化静态指针 scr1
    static size_t scr1_size = 256u*1024*1024;
    static void * scr1 = malloc(scr1_size);

    // 如果每个标记的内存大于0且超过了缓冲区大小，则重新分配缓冲区内存
    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = 1.1*(mem_per_token*N); // add 10% to account for ggml object overhead
        //printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__, buf_size, buf_size_new);

        // 重新分配缓冲区内存
        buf_size = buf_size_new;
        buf = realloc(buf, buf_size);
        if (buf == nullptr) {
            fprintf(stderr, "%s: failed to allocate %zu bytes\n", __func__, buf_size);
            return false;
        }
    }

    // 初始化 ggml_init_params 结构体
    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    // 初始化 ggml_context 结构体
    struct ggml_context * ctx0 = ggml_init(params);
    // 创建新的图形对象
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 创建包含位置信息的张量对象 KQ_pos
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    int * data = (int *) KQ_pos->data;
    for (int i = 0; i < N; ++i) {
        data[i] = n_past + i;
    }

    // 创建包含输入数据的张量对象 embd
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    // 获取 wte 对应的行数据
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte, embd);

    // 设置上下文对象的临时存储区
    ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

    // 对输入数据进行归一化处理
    {
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // 对输入数据进行 ln_f_g*inpL + ln_f_b 的运算
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    // 重置上下文对象的临时存储区
    ggml_set_scratch(ctx0, { 0, 0, nullptr, });

    // 对 lm_head 进行处理
    {
        inpL = ggml_mul_mat(ctx0, model.lmh_g, inpL);

        //inpL = ggml_add(ctx0,
        //        ggml_repeat(ctx0, model.lmh_b, inpL),
        //        inpL);
    }
    // 将 logits 转换为概率值
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 如果 n_past 为 100 的倍数，则打印图形和将图形转换为 dot 格式保存
    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // 调整 embd_w 的大小为 n_vocab*N
    //memcpy(embd_w.data(), ggml_get_data(inpL), sizeof(float)*n_vocab*N);

    // 仅返回最后一个标记的结果
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    // 如果每个标记的内存占用为 0，则计算每个标记的内存占用
    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    // 打印使用的内存大小
    //printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    // 释放上下文对象
    ggml_free(ctx0);

    // 返回 true
    return true;
}

int main(int argc, char ** argv) {
    ggml_time_init(); // 初始化时间

    const int64_t t_main_start_us = ggml_time_us(); // 获取当前时间

    gpt_params params; // 定义参数对象
    params.model = "models/stablelm-base-alpha-3b/ggml-model-f16.bin"; // 设置模型路径

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

    int64_t t_load_us = 0; // 初始化加载时间

    gpt_vocab vocab; // 定义词汇表对象
    gpt_neox_model model; // 定义模型对象

    // load the model
    {
        const int64_t t_start_us = ggml_time_us(); // 获取加载模型前的时间

        if (!gpt_neox_model_load(params.model, model, vocab)) { // 加载模型
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str()); // 打印加载模型失败的错误信息
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

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size()); // 设置预测数量不超过上下文长度减去提示长度

    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size()); // 打印提示中的token数量
    for (size_t i = 0; i < embd_inp.size(); i++) { // 遍历提示中的token
        printf("%s: token[%zu] = %6d, %s\n", __func__, i, embd_inp[i], vocab.id_to_token.at(embd_inp[i]).c_str()); // 打印每个token的信息
    }
    printf("\n"); // 打印空行

    std::vector<gpt_vocab::id> embd; // 定义embd向量

    // determine the required inference memory per token:
    size_t mem_per_token = 0; // 初始化每个token所需的推理内存
    gpt_neox_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token); // 进行模型推理
}
    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // 从当前嵌入大小开始循环直到嵌入输入大小加上预测数
        // 预测
        if (embd.size() > 0) {
            // 如果嵌入大小大于0
            const int64_t t_start_us = ggml_time_us();
            // 获取当前时间

            if (!gpt_neox_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
                // 如果无法进行预测
                printf("Failed to predict\n");
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;
            // 更新预测时间
        }

        n_past += embd.size();
        // 更新过去的嵌入大小
        embd.clear();
        // 清空嵌入

        if (i >= embd_inp.size()) {
            // 如果当前索引大于等于嵌入输入大小
            // 采样下一个标记
            const int   top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp  = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;
            // 初始化标记id

            {
                const int64_t t_start_sample_us = ggml_time_us();
                // 获取当前时间

                id = gpt_sample_top_k_top_p(vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p, temp, rng);
                // 使用top-k和top-p采样方法获取标记id

                t_sample_us += ggml_time_us() - t_start_sample_us;
                // 更新采样时间
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
            // 更新索引
        }

        // 显示文本
        for (auto id : embd) {
            printf("%s", vocab.id_to_token[id].c_str());
            // 打印标记对应的文本
        }
        fflush(stdout);
        // 清空输出缓冲区

        // 文本结束标记
        if (embd.back() == 0) {
            break;
        }
        // 如果嵌入的最后一个标记是0，则跳出循环
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