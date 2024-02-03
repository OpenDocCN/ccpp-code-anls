# `whisper.cpp\examples\talk\gpt-2.cpp`

```cpp
// 包含必要的头文件
#include "ggml.h"
#include "common-ggml.h"
#include "gpt-2.h"

// 包含 C++ 标准库头文件
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <thread>
#include <vector>
#include <regex>
#include <random>

/////////////////////// GPT-2 BEGIN /////////////////////////

// 定义默认的 GPT-2 模型超参数
struct gpt2_hparams {
    int32_t n_vocab = 50257;
    int32_t n_ctx   = 1024;
    int32_t n_embd  = 768;
    int32_t n_head  = 12;
    int32_t n_layer = 12;
    int32_t ftype   = 1;
};

// 定义 GPT-2 模型中的每个层的结构
struct gpt2_layer {
    // 归一化
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // 注意力机制
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

// 定义 GPT-2 模型结构
struct gpt2_model {
    gpt2_hparams hparams;

    // 归一化
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte;     // 位置嵌入
    struct ggml_tensor * wpe;     //    词嵌入
    struct ggml_tensor * lm_head; // 语言模型头

    std::vector<gpt2_layer> layers;

    // 键值记忆
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;
};

// 从文件加载模型的权重
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab) {
    // 打印加载模型的信息
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    // 打开二进制文件流
    auto fin = std::ifstream(fname, std::ios::binary);
    // 检查文件是否成功打开
    if (!fin) {
        // 打印错误信息
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // 验证魔数
    // 读取 magic 数值，判断是否为指定数值，若不是则输出错误信息并返回 false
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        if (magic != 0x67676d6c) {
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
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        // 输出加载的超参数值
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
    }

    // 加载词汇表
    {
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        // 检查词汇表大小是否与模型超参数中的词汇表大小相匹配，若不匹配则输出错误信息并返回 false
        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        char word[129];

        // 逐个读取词汇表中的词语及其对应的索引，并存储到词汇表中
        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            fin.read((char *) &len, sizeof(len));
            word[len] = '\0';
            fin.read((char *) word, len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 对于大张量，可以选择将数据存储为 16 位浮点数或量化形式，以节省内存并加快计算速度
    // 将模型参数中的文件类型转换为对应的 GGML 类型
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    // 如果转换后的类型为 GGML_TYPE_COUNT，则说明文件类型值无效，输出错误信息并返回 false
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    // 获取模型的上下文
    auto & ctx = model.ctx;

    // 初始化上下文大小为 0
    size_t ctx_size = 0;
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
    
        // 从超参数中获取相关参数值
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;
    
        // 计算上下文大小，包括不同部分的大小
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_g
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_b
    
        ctx_size += n_vocab*ggml_row_size(wtype, n_embd);         // wte
        ctx_size += n_ctx*ggml_row_size(GGML_TYPE_F32, n_embd); // wpe
        ctx_size += n_vocab*ggml_row_size(wtype, n_embd);         // lm_head
    
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_g
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_b
    
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_g
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_b
    
        ctx_size += n_layer*(ggml_row_size(wtype,         3*n_embd*n_embd)); // c_attn_attn_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 3*n_embd));        // c_attn_attn_b
    
        ctx_size += n_layer*(ggml_row_size(wtype,         n_embd*n_embd)); // c_attn_proj_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd));        // c_attn_proj_b
    
        ctx_size += n_layer*(ggml_row_size(wtype,         4*n_embd*n_embd)); // c_mlp_fc_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 4*n_embd));        // c_mlp_fc_b
    
        ctx_size += n_layer*(ggml_row_size(wtype,         4*n_embd*n_embd)); // c_mlp_proj_w
        ctx_size += n_layer*(ggml_row_size(GGML_TYPE_F32,   n_embd));        // c_mlp_proj_b
    
        ctx_size += n_ctx*n_layer*ggml_row_size(GGML_TYPE_F32, n_embd); // memory_k
        ctx_size += n_ctx*n_layer*ggml_row_size(GGML_TYPE_F32, n_embd); // memory_v
    
        // 计算对象开销
        ctx_size += (6 + 12*n_layer)*256; // object overhead
    
        // 打印计算得到的 ggml 上下文大小
        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size/(1024.0*1024.0);
    }
    // 创建 ggml 上下文
    {
        // 初始化参数结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,  // 上下文大小
            /*.mem_buffer =*/ NULL,      // 内存缓冲区
            /*.no_alloc   =*/ false,     // 是否分配内存
        };

        // 初始化 ggml 上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，则输出错误信息并返回 false
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

        // 获取嵌入维度、层数和上下文大小
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        // 计算内存数量和元素数量
        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        // 创建键值内存张量
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        // 计算内存大小并输出信息
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);
        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // 加载权重
    }

    // 关闭文件流并返回 true
    fin.close();

    return true;
// 评估变换器
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    预测的下一个标记的对数
//   - mem_per_token: 每个标记的内存占用
//
// TODO: 从 ggml 仓库同步最新版本
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

    static size_t buf_size = 512u*1024*1024;
    static void * buf = malloc(buf_size);

    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = 1.1*(mem_per_token*N); // 添加10%以考虑 ggml 对象的开销
        //printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__, buf_size, buf_size_new);

        // 重新分配内存
        buf_size = buf_size_new;
        buf = realloc(buf, buf_size);
        if (buf == nullptr) {
            fprintf(stderr, "%s: failed to allocate %zu bytes\n", __func__, buf_size);
            return false;
        }
    }

    // 初始化参数
    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    // 初始化 ggml 上下文
    struct ggml_context * ctx0 = ggml_init(params);
    struct ggml_cgraph gf = {};

    // 创建嵌入张量
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    // 创建位置张量
    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 遍历 N 次，将 n_past + i 的值赋给 position->data 数组中的每个元素
    for (int i = 0; i < N; ++i) {
        ((int32_t *) position->data)[i] = n_past + i;
    }

    // 计算 wte + wpe，并将结果存储在 inpL 中
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));

    }

    // 对 inpL 进行归一化处理
    {
        // [ 768, N]
        inpL = ggml_norm(ctx0, inpL, 1e-5f);

        // 对 inpL 进行线性变换
        // inpL = ln_f_g*inpL + ln_f_b
        // [ 768, N]
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    // 对 inpL 进行矩阵乘法运算，结果存储在 inpL 中
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // 运行计算
    ggml_build_forward_expand  (&gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, &gf, n_threads);

    // 将结果存储在 embd_w 中，仅保留最后一个 token 的结果
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);

    // 如果 mem_per_token 为 0，则计算每个 token 所需的内存
    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    // 打印使用的内存大小
    //printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    // 释放上下文
    ggml_free(ctx0);

    // 返回 true
    return true;
// 结构体定义，包含 GPT-2 上下文信息
struct gpt2_context {
    // 默认的提示文本
    std::string prompt_base = R"(Hello, how are you?
I'm fine, thanks. How are you?
Thanks, I'm fine too. What are you doing?
I'm just sitting here.
It's a lovely day, isn't it?
Yes, it is. I love the weather this time of year.
I wish it would rain a little bit.
Me too.
)";
    
    // 随机数生成器
    std::mt19937 rng;
    
    // GPT-2 词汇表
    gpt_vocab vocab;
    
    // GPT-2 模型
    gpt2_model model;
    
    // 线程数
    int32_t n_threads = std::min(N_THREAD, (int) std::thread::hardware_concurrency());
    
    // 采样参数
    int32_t top_k = 5;
    float top_p = 0.9f;
    float temp = 1.0f;
};

// 初始化 GPT-2 上下文
struct gpt2_context * gpt2_init(const char * path_model) {
    // 创建 GPT-2 上下文对象
    gpt2_context * ctx = new gpt2_context;
    
    // 初始化随机数生成器
    ctx->rng = std::mt19937(time(nullptr));
    
    // 加载模型
    {
        const int64_t t_start_us = ggml_time_us();
        
        // 加载模型和词汇表
        if (!gpt2_model_load(path_model, ctx->model, ctx->vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, path_model);
            delete ctx;
            return nullptr;
        }
        
        const int64_t t_load_us = ggml_time_us() - t_start_us;
        
        printf("gpt-2: model loaded in %d ms\n", (int) (t_load_us/1000));
    }
    
    return ctx;
}

// 释放 GPT-2 上下文
void gpt2_free(struct gpt2_context * ctx) {
    delete ctx;
}

// 获取默认提示文本
const char * gpt2_get_prompt(struct gpt2_context * ctx) {
    return ctx->prompt_base.c_str();
}

// 设置提示文本
void gpt2_set_prompt(struct gpt2_context * ctx, const char * prompt) {
    ctx->prompt_base = prompt;
}

// 对文本进行分词
std::vector<gpt_vocab::id> gpt2_tokenize(const gpt2_context * ctx, const char * text) {
    return ::gpt_tokenize(ctx->vocab, text);
}

// 生成文本
std::string gpt2_gen_text(gpt2_context * ctx, const char * text, int max_tokens) {
    int n_past = 0;
    
    std::vector<float> embd_w;
    
    // 对提示文本进行分词
    std::vector<gpt_vocab::id> embd_inp = ::gpt2_tokenize(ctx, text);
    // 计算预测的 token 数量，取最小值为 max_tokens 和上下文长度减去当前输入 token 数量的差值
    int n_predict = std::min(max_tokens, ctx->model.hparams.n_ctx - (int) embd_inp.size());

    // 将输入的 token 序列赋值给 embd
    std::vector<gpt_vocab::id> embd = embd_inp;

    // 每个 token 所需的内存大小
    size_t mem_per_token = 3000000;

    // 存储生成的文本结果
    std::string result;

    // 循环生成预测的 token
    for (int i = embd.size(); i < (int) embd_inp.size() + n_predict; i++) {
        // 预测下一个 token
        if (!embd.empty()) {
            // 调用 gpt2_eval 函数生成文本
            if (!gpt2_eval(ctx->model, ctx->n_threads, n_past, embd, embd_w, mem_per_token)) {
                printf("gpt-2: failed to generate text\n");
                return "";
            }
        }

        // 更新历史 token 数量
        n_past += embd.size();
        // 清空当前 token 序列
        embd.clear();

        {
            // 从概率分布中采样下一个 token
            const int   top_k = ctx->top_k;
            const float top_p = ctx->top_p;
            const float temp  = ctx->temp;

            const int n_vocab = ctx->model.hparams.n_vocab;

            // 从概率分布中采样下一个 token 的 id
            const gpt_vocab::id id = gpt_sample_top_k_top_p(ctx->vocab, embd_w.data() + (embd_w.size() - n_vocab), top_k, top_p, temp, ctx->rng);

            // 将采样得到的 token id 添加到当前 token 序列中
            embd.push_back(id);
        }

        // 将生成的 token 添加到结果中
        result += ctx->vocab.id_to_token[embd[0]];

        // 如果生成的 token 是文本结束符，则结束生成
        if (embd.back() == 50256) {
            break;
        }
    }

    // 返回生成的文本结果
    return result;
# 闭合之前的代码块
```