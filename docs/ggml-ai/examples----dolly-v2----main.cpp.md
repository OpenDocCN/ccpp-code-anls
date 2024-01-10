# `ggml\examples\dolly-v2\main.cpp`

```
#include "ggml/ggml.h"
// 引入 ggml 库的头文件

#include "common.h"
#include "common-ggml.h"
// 引入通用的头文件

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <cinttypes>
#include <fstream>
#include <iostream>
#include <map>
#include <string>
#include <vector>
// 引入 C++ 标准库的头文件

#if !defined(_WIN32)
#define DOLLY_INTERACTIVE_PORT
#endif
// 如果不是在 Windows 平台下，则定义 DOLLY_INTERACTIVE_PORT

#if defined(DOLLY_INTERACTIVE_PORT)
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>
#endif
// 如果定义了 DOLLY_INTERACTIVE_PORT，则引入相关的网络编程头文件

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif
// 如果是在 MSVC 编译器下，则禁用警告 4244 和 4267

// default hparams (Dolly-V2 3B)
struct dollyv2_hparams {
    int32_t n_vocab = 50254; // tokenizer.vocab_size
    int32_t n_ctx   = 2048;  // model.config.max_position_embeddings
    int32_t n_embd  = 2560;  // model.config.hidden_size
    int32_t n_head  = 32;    // model.config.num_attention_heads
    int32_t n_layer = 32;    // model.config.num_hidden_layers
    int32_t n_rot   = 20;    // rotary_pct[25%] * (n_embd / n_head)
    int32_t par_res = 1; // 1 = true, 0 = false
    int32_t ftype   = GGML_FTYPE_MOSTLY_F16;
    float   eps     = 1e-5f;
};
// 定义了一个结构体 dollyv2_hparams，包含了默认的参数设置

const std::string INSTRUCTION_KEY = "### Instruction:";
const std::string RESPONSE_KEY    = "### Response:";
const std::string END_KEY         = "### End";
const std::string INTRO_BLURB     = "Below is an instruction that describes a task. Write a response that appropriately completes the request.";
// 定义了一些常量字符串

// dollyv2 prompt format
std::string prompt_for_generation(const std::string& instruction) {
    return INTRO_BLURB + "\n\n" + INSTRUCTION_KEY + "\n" + instruction + "\n\n" + RESPONSE_KEY + "\n";
}
// 定义了一个函数 prompt_for_generation，用于生成提示信息的格式化字符串

struct dollyv2_layer {
    // pre normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    // attention
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // post normalization
    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // ff
    # 定义指向 ggml_tensor 结构体的指针变量 c_mlp_fc_w，用于存储多层感知机全连接层的权重
    struct ggml_tensor * c_mlp_fc_w;
    # 定义指向 ggml_tensor 结构体的指针变量 c_mlp_fc_b，用于存储多层感知机全连接层的偏置
    struct ggml_tensor * c_mlp_fc_b;
    
    # 定义指向 ggml_tensor 结构体的指针变量 c_mlp_proj_w，用于存储多层感知机投影层的权重
    struct ggml_tensor * c_mlp_proj_w;
    # 定义指向 ggml_tensor 结构体的指针变量 c_mlp_proj_b，用于存储多层感知机投影层的偏置
    struct ggml_tensor * c_mlp_proj_b;
// 结构体定义，包含模型的超参数和各种张量
struct dollyv2_model {
    dollyv2_hparams hparams; // 模型的超参数

    // 归一化
    struct ggml_tensor * ln_f_g; // 归一化的缩放参数
    struct ggml_tensor * ln_f_b; // 归一化的偏置参数

    struct ggml_tensor * wte; // 位置嵌入

    struct ggml_tensor * lmh_g; // 语言模型头
    //struct ggml_tensor * lmh_b; // 语言模型偏置

    std::vector<dollyv2_layer> layers; // 模型的层

    // 键值记忆
    struct ggml_tensor * memory_k; // 键记忆
    struct ggml_tensor * memory_v; // 值记忆

    //
    struct ggml_context * ctx; // 上下文
    std::map<std::string, struct ggml_tensor *> tensors; // 张量映射
};

// 从文件加载模型的权重
bool dollyv2_model_load(const std::string & fname, dollyv2_model & model, gpt_vocab & vocab) {
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str()); // 打印加载模型的信息

    auto fin = std::ifstream(fname, std::ios::binary); // 以二进制方式打开文件
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str()); // 如果打开文件失败则打印错误信息
        return false; // 返回加载失败
    }

    // 验证魔数
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic)); // 读取文件中的魔数
        if (magic != GGML_FILE_MAGIC) { // 如果魔数不匹配
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str()); // 打印错误信息
            return false; // 返回加载失败
        }
    }

    // 加载超参数
    {
        // 获取模型的超参数引用
        auto & hparams = model.hparams;

        // 从文件中读取并存储超参数的值
        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fin.read((char *) &hparams.n_rot,   sizeof(hparams.n_rot));
        fin.read((char *) &hparams.par_res, sizeof(hparams.par_res));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        // 计算 qntvr 值
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        // 打印超参数的值
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: n_rot   = %d\n", __func__, hparams.n_rot);
        printf("%s: par_res = %d\n", __func__, hparams.par_res);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        // 计算 ftype 的余数并更新 hparams.ftype 的值
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // 加载词汇表
    {
        // 获取词汇表大小
        const int32_t n_vocab = model.hparams.n_vocab;

        // 定义变量
        std::string word;
        std::vector<char> buf(128);

        // 遍历词汇表，读取并存储词汇及其对应的 ID
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }

        // 添加特殊标记到词汇表
        vocab.add_special_token("### End");
        vocab.add_special_token("### Instruction:");
        vocab.add_special_token("### Response:");
    }
    // 对于大张量，我们有选项将数据存储为16位浮点数或量化，以节省内存并加快计算速度
    // 将模型的数据类型转换为GGML类型
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    // 如果转换后的类型为无效类型，输出错误信息并返回false
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

        // 从超参数中获取相关参数值
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;

        // 计算并累加 ln_f_g 和 ln_f_b 的内存大小
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_g
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_b

        // 计算并累加 wte 的内存大小
        ctx_size += ggml_row_size(wtype, n_embd*n_vocab); // wte

        // 计算并累加 lmh_g 的内存大小
        ctx_size += ggml_row_size(wtype, n_embd*n_vocab); // lmh_g
        //ctx_size += ggml_row_size(GGML_TYPE_F32, n_vocab); // lmh_b

        // 计算并累加 ln_1_g 和 ln_1_b 的内存大小
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_g
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_b

        // 计算并累加 c_attn_attn_w 和 c_attn_attn_b 的内存大小
        ctx_size += n_layer*(ggml_row_size(wtype,         3*n_embd*n_embd)); // c_attn_attn_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 3*n_embd));        // c_attn_attn_b

        // 计算并累加 c_attn_proj_w 和 c_attn_proj_b 的内存大小
        ctx_size += n_layer*(ggml_row_size(wtype,         n_embd*n_embd)); // c_attn_proj_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd*n_embd)); // c_attn_proj_b

        // 计算并累加 ln_2_g 和 ln_2_b 的内存大小
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_g
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_b

        // 计算并累加 c_mlp_fc_w 和 c_mlp_fc_b 的内存大小
        ctx_size += n_layer*(ggml_row_size(wtype,         4*n_embd*n_embd)); // c_mlp_fc_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 4*n_embd));        // c_mlp_fc_b

        // 计算并累加 c_mlp_proj_w 和 c_mlp_proj_b 的内存大小
        ctx_size += n_layer*(ggml_row_size(wtype,         4*n_embd*n_embd)); // c_mlp_proj_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32,   n_embd));        // c_mlp_proj_b

        // 计算并累加 memory_k 和 memory_v 的内存大小
        ctx_size += n_ctx*n_layer*ggml_row_size(GGML_TYPE_F32, n_embd); // memory_k
        ctx_size += n_ctx*n_layer*ggml_row_size(GGML_TYPE_F32, n_embd); // memory_v

        // 计算并累加对象开销的内存大小
        ctx_size += (6 + 16*n_layer)*512; // object overhead

        // 打印 ggml 上下文的内存大小
        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0));
    }

    // 创建 ggml 上下文
    {
        // 初始化参数结构体，设置内存大小为ctx_size，内存缓冲区为空，允许分配内存
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        // 使用初始化参数结构体初始化模型上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，则打印错误信息并返回false
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

        // 获取嵌入大小、层数和上下文大小
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        // 计算内存大小和元素个数
        const int64_t n_mem      = n_layer*n_ctx;
        const int64_t n_elements = n_embd*n_mem;

        // 分配键和值的内存
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
}

// 前馈网络
ggml_tensor * gpt_neox_ff(
        const dollyv2_layer & layer,
        ggml_context        * ctx0,
        ggml_tensor         * inp,
        float                 eps) {
    // 对输入进行归一化处理
    ggml_tensor * cur = ggml_norm(ctx0, inp, eps);

    // 计算 LN_2_g 与 LN_2_b 的加权和
    cur = ggml_add(ctx0,
        ggml_mul(ctx0,
            ggml_repeat(ctx0, layer.ln_2_g, cur),
            cur),
        ggml_repeat(ctx0, layer.ln_2_b, cur));

    // 计算 MLP_FC_W 与当前值的矩阵乘法
    cur = ggml_mul_mat(ctx0,
            layer.c_mlp_fc_w,
            cur);

    // 加上 MLP_FC_B
    cur = ggml_add(ctx0,
            ggml_repeat(ctx0, layer.c_mlp_fc_b, cur),
            cur);

    // GELU 激活函数
    cur = ggml_gelu(ctx0, cur);

    // 投影
    // cur = PROJ_W * cur + PROJ_B
    cur = ggml_mul_mat(ctx0,
            layer.c_mlp_proj_w,
            cur);

    cur = ggml_add(ctx0,
            ggml_repeat(ctx0, layer.c_mlp_proj_b, cur),
            cur);
    return cur;
}

// 评估变压器
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    下一个标记的预测对数
//
bool dollyv2_eval(
        const dollyv2_model & model,
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
    // 如果每个标记的内存占用大于0并且总内存占用超过缓冲区大小
    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        // 计算新的缓冲区大小，增加10%以考虑ggml对象的开销
        const size_t buf_size_new = 1.1*(mem_per_token*N); // add 10% to account for ggml object overhead
        //printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__, buf_size, buf_size_new);

        // 重新分配内存
        buf_size = buf_size_new;
        buf = realloc(buf, buf_size);
        // 如果内存分配失败，则输出错误信息并返回false
        if (buf == nullptr) {
            fprintf(stderr, "%s: failed to allocate %zu bytes\n", __func__, buf_size);
            return false;
        }
    }

    // 初始化参数结构体
    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    // 初始化ggml上下文
    struct ggml_context * ctx0 = ggml_init(params);
    // 创建新的计算图
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    // KQ_pos - 包含位置信息
    // 创建新的一维张量
    struct ggml_tensor * KQ_pos = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    int * data = (int *) KQ_pos->data;
    // 将位置信息填充到张量中
    for (int i = 0; i < N; ++i) {
        data[i] = n_past + i;
    }

    // 创建新的一维张量
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 将输入的数据复制到张量中
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    // wte
    // 获取模型wte中的指定行
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte, embd);

    }

    // norm
    {
        // 对inpL进行归一化处理
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        // 对inpL进行线性变换
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    // lm_head
    {
        // 对inpL进行矩阵乘法
        inpL = ggml_mul_mat(ctx0, model.lmh_g, inpL);

        //inpL = ggml_add(ctx0,
        //        ggml_repeat(ctx0, model.lmh_b, inpL),
        //        inpL);
    }

    // logits -> probs
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    //if (n_past%100 == 0) {
    // 打印图形数据
    // ggml_graph_print   (&gf);
    // 将图形数据转储为 DOT 格式
    // ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // 调整嵌入权重的大小
    // embd_w.resize(n_vocab*N);
    // 将输入层数据拷贝到嵌入权重中
    // memcpy(embd_w.data(), ggml_get_data(inpL), sizeof(float)*n_vocab*N);

    // 仅返回最后一个标记的结果
    // 调整嵌入权重的大小为词汇表大小
    embd_w.resize(n_vocab);
    // 将最后一个标记的数据拷贝到嵌入权重中
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    // 如果每个标记的内存占用为 0，则计算每个标记的内存占用
    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    // 打印使用的内存
    // printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    // 释放上下文内存
    ggml_free(ctx0);

    // 返回 true
    return true;
    // 执行交互式提示，生成文本输出
    std::string execute_prompt(
        // 输入模型、词汇表、提示文本、参数、随机数生成器、加载时间、采样时间、预测时间、每个标记的内存，过去的标记数，是否将响应流到控制台
        const dollyv2_model &model,
        gpt_vocab &vocab,
        const std::string &prompt,
        gpt_params &params,
        std::mt19937 &rng,
        int64_t t_load_us,
        int64_t t_sample_us,
        int64_t t_predict_us,
        size_t mem_per_token,
        int n_past,
        bool stream_response_to_cout = false) {
    // 初始化输出字符串
    std::string output = "";
    // 初始化logits向量
    std::vector<float> logits;

    // 对提示文本进行标记化
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, prompt);

    // 更新预测标记数
    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int)embd_inp.size());

    // 打印提示文本中的标记数
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());
    // 遍历提示文本中的标记并打印
    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6d, %s\n", __func__, i, embd_inp[i], vocab.id_to_token.at(embd_inp[i]).c_str());
    }
    printf("\n");

    // 初始化标记化向量
    std::vector<gpt_vocab::id> embd;

    // 执行模型评估
    dollyv2_eval(model, params.n_threads, 0, {0, 1, 2, 3}, logits, mem_per_token);

    // 获取结束标记的ID
    const int32_t end_token = vocab.token_to_id["### End"];
    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // 从当前嵌入大小开始，循环直到嵌入输入大小加上预测数量
        // 预测
        if (embd.size() > 0) {
            // 如果嵌入大小大于0
            const int64_t t_start_us = ggml_time_us();
            // 记录开始时间

            if (!dollyv2_eval(model, params.n_threads, n_past, embd, logits, mem_per_token)) {
                // 如果预测失败
                printf("Failed to predict\n");
                return output;
            }

            t_predict_us += ggml_time_us() - t_start_us;
            // 记录预测时间
        }

        n_past += embd.size();
        // 更新过去的数量
        embd.clear();
        // 清空嵌入

        if (i >= embd_inp.size()) {
            // 如果当前索引大于等于嵌入输入大小
            // 采样下一个标记
            const int top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;
            // 初始化标记id

            {
                const int64_t t_start_sample_us = ggml_time_us();
                // 记录开始采样时间

                id = gpt_sample_top_k_top_p(vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p, temp, rng);
                // 通过top-k和top-p采样得到标记id

                t_sample_us += ggml_time_us() - t_start_sample_us;
                // 记录采样时间
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
            output += vocab.id_to_token[id];
            // 将标记id转换为文本并添加到输出中
            if (stream_response_to_cout) {
                printf("%s", vocab.id_to_token[id].c_str());
                // 如果需要将响应流输出到控制台，则打印文本
            }
        }
        if (stream_response_to_cout) {
            fflush(stdout);
            // 如果需要将响应流输出到控制台，则刷新标准输出
        }

        // 文本结束标记
        if (embd.back() == 0 || (end_token > 0 && embd.back() == end_token)) {
            return output;
            // 如果嵌入的最后一个标记为0或者等于结束标记，则返回输出
        }
    }
    return output;
    // 返回输出
// 如果定义了DOLLY_INTERACTIVE_PORT，则执行以下代码
#if defined(DOLLY_INTERACTIVE_PORT)
// 设置端口的函数，参数为端口号，返回一个套接字描述符
int setup_port(const int port) {
    // 创建一个 IPv4 的流式套接字
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    // 如果创建套接字失败，则输出错误信息并返回-1
    if (sockfd < 0) {
        fprintf(stderr, "%s: Failed to create new socket\n", __func__);
        return -1;
    }

    // 初始化服务器地址结构体
    sockaddr_in servaddr;
    std::memset(&servaddr, 0, sizeof(servaddr));

    // 设置服务器地址结构体的成员
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(port);

    // 将套接字绑定到指定的端口
    if (bind(sockfd, (struct sockaddr *)&servaddr, sizeof(servaddr)) < 0) {
        fprintf(stderr, "%s: Failed to bind to port %i\n", __func__, port);
        return -1;
    }

    // 监听套接字，最大连接数为10
    if (listen(sockfd, 10) < 0) {
        fprintf(stderr, "%s: Failed to listen to socket on port %i\n", __func__, port);
        return -1;
    }
    // 返回套接字描述符
    return sockfd;
}

// 从端口读取数据的函数，参数为套接字描述符和客户端套接字描述符，返回从客户端读取的数据
std::string read_from_port(int sockfd, int clientfd) {
    // 如果客户端套接字描述符小于0，则输出错误信息并返回空字符串
    if (clientfd < 0) {
        fprintf(stderr, "%s: Failed to accept new connection\n", __func__);
        return "";
    }

    // 初始化缓冲区并清零
    char buffer[4096];
    std::memset(buffer, 0, sizeof(buffer));

    // 从客户端套接字读取数据到缓冲区，如果失败则输出错误信息
    if (read(clientfd, buffer, sizeof(buffer)) < 0) {
        fprintf(stderr, "%s: Failed to read from client\n", __func__);
    } else {
        // 输出接收到的数据并返回字符串类型的数据
        std::cout << "Received: " << buffer;
        return std::string(buffer);
    }
    // 返回空字符串
    return std::string("");
}
#endif

// 主函数
int main(int argc, char ** argv) {
    // 初始化时间
    ggml_time_init();

    // 获取主函数开始时的时间戳
    const int64_t t_main_start_us = ggml_time_us();

    // 初始化参数结构体
    gpt_params params;
    params.model = "models/dolly-v2-3b/ggml-model-f16.bin";

    // 解析命令行参数，如果失败则返回1
    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果参数中的种子值小于0，则设置为当前时间戳
    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    // 输出种子值
    printf("%s: seed = %d\n", __func__, params.seed);

    // 初始化随机数生成器
    std::mt19937 rng(params.seed);

    // 初始化时间统计变量
    int64_t t_load_us = 0;
    int64_t t_sample_us = 0;
    int64_t t_predict_us = 0;

    // 确定每个标记所需的推理内存
    size_t mem_per_token = 0;

    int n_past = 0;

    // 初始化词汇表和模型
    gpt_vocab vocab;
    dollyv2_model model;
}
    // 加载模型
    {
        // 记录加载模型的起始时间
        const int64_t t_start_us = ggml_time_us();

        // 如果无法从指定路径加载模型，则输出错误信息并返回1
        if (!dollyv2_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        // 计算加载模型所花费的时间
        t_load_us = ggml_time_us() - t_start_us;

        // 测试 GPT 分词器
        test_gpt_tokenizer(vocab, params.token_test);
    }
#if defined(DOLLY_INTERACTIVE_PORT)
    // 如果定义了DOLLY_INTERACTIVE_PORT，则创建一个套接字描述符并初始化为-1
    int sockfd = -1;
    // 如果参数中指定了交互端口，则设置套接字描述符
    if (params.interactive_port != -1) {
        sockfd = setup_port(params.interactive_port);
        // 如果设置套接字描述符失败，则返回1
        if (sockfd == -1) {
            return 1;
        }
        // 打印模型已准备好的消息
        fprintf(stdout, "Model is ready on port %i\n", params.interactive_port);
        fflush(stdout);
    }
#endif

    // 如果参数中指定了交互模式或交互端口不为-1，则进入循环
    if (params.interactive || params.interactive_port != -1) {
        while (true) {
            std::string prompt_input;
#if defined(DOLLY_INTERACTIVE_PORT)
            int clientfd = -1;
            // 如果参数中指定了交互端口，则接受客户端连接并读取输入
            if (params.interactive_port != -1) {
                sockaddr_in clientaddr;
                socklen_t clientaddrlen = sizeof(clientaddr);
                clientfd = accept(sockfd, (struct sockaddr *)&clientaddr, &clientaddrlen);
                prompt_input = read_from_port(sockfd, clientfd);
            } else
#endif
            {
                // 否则，提示用户输入问题
                printf("Please enter your quesiton:\n>");
                fflush(stdout);

                std::getline(std::cin, prompt_input);
            }

            // 如果用户输入"exit"，则退出循环
            if (strcmp(prompt_input.c_str(), "exit") == 0) {
                break;
            }

            // 生成提示并调用模型执行
            const std::string prompt = prompt_for_generation(prompt_input);
            const std::string response = execute_prompt(model, vocab, prompt, params, rng, t_load_us, t_sample_us, t_predict_us, mem_per_token, n_past, true);

#if defined(DOLLY_INTERACTIVE_PORT)
            // 如果参数中指定了交互端口，则将响应写回客户端
            if (params.interactive_port != -1) {
                if (write(clientfd, response.c_str(), response.size()) < 0) {
                    fprintf(stderr, "%s: Failed to write answer '%s' to client\n", __func__, response.c_str());
                }

                if (close(clientfd) < 0) {
                    fprintf(stderr, "%s: Failed to close client socket\n", __func__);
                }
            } else
#endif
            {
                // 否则，打印响应
                printf("%s\n\n", response.c_str());
            }
            fflush(stdout);
        }
    } else {
        // 如果参数中的提示为空，则生成一个随机的提示
        if (params.prompt.empty()) {
            params.prompt = gpt_random_prompt(rng);
        }

        // 获取用户输入的提示
        const std::string prompt = prompt_for_generation(params.prompt);
        // 执行生成
        execute_prompt(model, vocab, prompt, params, rng, t_load_us, t_sample_us, t_predict_us, mem_per_token, n_past, true);
    }

    // 报告时间
    {
        // 获取主函数结束时间
        const int64_t t_main_end_us = ggml_time_us();

        // 打印内存每个标记的消耗
        printf("\n\n");
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        // 打印加载时间
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us / 1000.0f);
        // 打印采样时间
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us / 1000.0f);
        // 打印预测时间和每个标记的平均预测时间
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us / 1000.0f, t_predict_us / 1000.0f / n_past);
        // 打印总时间
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    // 释放模型上下文
    ggml_free(model.ctx);
#if defined(DOLLY_INTERACTIVE_PORT)
    // 如果定义了DOLLY_INTERACTIVE_PORT，则执行以下代码块
    if (params.interactive_port != -1 && close(sockfd) < 0) {
        // 如果交互端口不等于-1且关闭套接字失败，则输出错误信息
        fprintf(stderr, "%s: Failed to close server socket\n", __func__);
    }
#endif
    // 返回0
    return 0;
}
```