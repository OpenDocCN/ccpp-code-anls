# `ggml\examples\mpt\main.cpp`

```
#include "ggml/ggml.h"  // 引入 ggml 库的头文件

#include "common-ggml.h"  // 引入 ggml 通用头文件
#include "common.h"  // 引入通用头文件

#include <cmath>  // 引入数学函数库
#include <cstddef>  // 引入标准库定义的宏
#include <cstdio>  // 引入输入输出函数库
#include <cstring>  // 引入字符串处理函数库
#include <fstream>  // 引入文件输入输出流函数库
#include <cinttypes>  // 引入整数类型宏
#include <map>  // 引入映射容器库
#include <string>  // 引入字符串库
#include <utility>  // 引入实用程序库
#include <vector>  // 引入向量容器库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 可能丢失数据的警告
#endif

// 现在没有默认值
struct mpt_hparams {
    int32_t d_model      = 0;  // 模型维度
    int32_t max_seq_len  = 0;  // 最大序列长度
    int32_t n_heads      = 0;  // 头数
    int32_t n_layers     = 0;  // 层数
    int32_t n_vocab      = 0;  // 词汇量
    float alibi_bias_max = 0;  // 偏置最大值
    float clip_qkv       = 0;  // 剪辑 qkv
    int32_t ftype        = 0;  // 特征类型
    int32_t n_ctx        = 0;  // 上下文数
};

struct mpt_layer {
    // 预归一化
    struct ggml_tensor * norm_1_weight;

    // 注意力
    struct ggml_tensor * c_attn_wqkv_weight;
    struct ggml_tensor * c_attn_out_proj_weight;

    // 后归一化
    struct ggml_tensor * norm_2_weight;

    // 前馈
    struct ggml_tensor * ffn_up_proj;
    struct ggml_tensor * ffn_down_proj;
};

struct mpt_model {
    mpt_hparams hparams;

    struct ggml_tensor * wte_weight;    // 位置嵌入
    struct ggml_tensor * norm_f_weight; // 语言模型头

    std::vector<mpt_layer> layers;  // 层

    // 键 + 值 内存
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    struct ggml_context * ctx;
    std::map<std::string, struct ggml_tensor *> tensors;  // 张量
};

struct mpt_params {
    int32_t n_threads = std::min(4, (int32_t) std::thread::hardware_concurrency());  // 线程数

    int32_t seed           = -1;  // 随机数种子
    int32_t n_predict      = 200;  // 预测的新标记数
    int32_t n_batch        = 8;  // 提示处理的批处理大小
    int32_t n_ctx          = 512;  // 上下文大小

    std::string model      = "";  // 模型路径
    std::string prompt     = "";  // 提示
    std::string token_test = "";  // 标记测试

    bool    perplexity     = false;  // 困惑度

    // 采样参数
    int32_t top_k          = 0;
    float   top_p          = 1.0f;
    # 定义一个浮点型变量 temp，并赋初值 0.8
    float temp = 0.8f;
    # 定义一个 32 位整型变量 repeat_last_n，并赋初值 64
    int32_t repeat_last_n = 64;
    # 定义一个浮点型变量 repeat_penalty，并赋初值 1.02
    float repeat_penalty = 1.02f;
// 打印程序的使用方法和选项
void mpt_print_usage(int /*argc*/, char ** argv, const mpt_params & params) {
    // 打印程序的使用方法
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    // 打印各种选项的说明
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    fprintf(stderr, "  -s SEED, --seed SEED  RNG seed (default: -1)\n");
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    fprintf(stderr, "  -p PROMPT, --prompt PROMPT\n");
    fprintf(stderr, "                        prompt to start generation with (default: random)\n");
    fprintf(stderr, "  -f FNAME, --file FNAME\n");
    fprintf(stderr, "                        load prompt from a file\n");
    fprintf(stderr, "  -tt TOKEN_TEST, --token_test TOKEN_TEST\n");
    fprintf(stderr, "                        test tokenization\n");
    fprintf(stderr, "  -n N, --n_predict N   number of tokens to predict (default: %d)\n", params.n_predict);
    fprintf(stderr, "  --top_k N             top-k sampling (default: %d, 0 = n_vocab)\n", params.top_k);
    fprintf(stderr, "  --top_p N             top-p sampling (default: %.2f)\n", params.top_p);
    fprintf(stderr, "  --temp N              temperature (default: %.2f)\n", params.temp);
    fprintf(stderr, "  --repeat-last-n N     last n tokens to consider for penalize (default: %d, 0 = disabled, -1 = ctx_size)\n", params.repeat_last_n);
    fprintf(stderr, "  --repeat-penalty N    penalize repeat sequence of tokens (default: %.2f, 1.0 = disabled)\n", (double)params.repeat_penalty);
    fprintf(stderr, "  --perplexity          compute perplexity over the prompt\n");
    fprintf(stderr, "  -c N, --ctx-size N    size of the prompt context (default: %d)\n", params.n_ctx);
    fprintf(stderr, "  -b N, --batch_size N  batch size for prompt processing (default: %d)\n", params.n_batch);
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
}
    # 在标准错误流中输出带格式的字符串，显示模型路径，默认为params.model的值
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    # 在标准错误流中输出换行
    fprintf(stderr, "\n");
}

bool mpt_params_parse(int argc, char ** argv, mpt_params & params) {
    }

    return true;
}

// 从文件加载模型的权重
bool mpt_model_load(const std::string & fname, mpt_model & model, gpt_vocab & vocab) {
    // 打印加载模型的信息
    printf("%s: loading model from '%s' - please wait ...\n", __func__, fname.c_str());

    // 打开文件流
    auto fin = std::ifstream(fname, std::ios::binary);
    // 如果文件流打开失败，打印错误信息并返回false
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // 验证文件的魔数
    {
        uint32_t magic;
        fin.read((char *)&magic, sizeof(magic));
        // 如果魔数不匹配，打印错误信息并返回false
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // 加载超参数
    {
        // 获取模型的超参数引用
        auto & hparams = model.hparams;

        // 从文件中读取模型的维度信息，并存入超参数中
        fin.read((char *) &hparams.d_model,        sizeof(hparams.d_model));
        fin.read((char *) &hparams.max_seq_len,    sizeof(hparams.max_seq_len));
        fin.read((char *) &hparams.n_heads,        sizeof(hparams.n_heads));
        fin.read((char *) &hparams.n_layers,       sizeof(hparams.n_layers));
        fin.read((char *) &hparams.n_vocab,        sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.alibi_bias_max, sizeof(hparams.alibi_bias_max));
        fin.read((char *) &hparams.clip_qkv,       sizeof(hparams.clip_qkv));
        fin.read((char *) &hparams.ftype,          sizeof(hparams.ftype));

        // 根据最大序列长度和上下文长度的关系，确定上下文长度
        hparams.n_ctx = std::min(hparams.max_seq_len, hparams.n_ctx);

        // 根据读取的文件信息，打印模型的超参数信息
        printf("%s: d_model        = %d\n", __func__, hparams.d_model);
        printf("%s: max_seq_len    = %d\n", __func__, hparams.max_seq_len);
        printf("%s: n_ctx          = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_heads        = %d\n", __func__, hparams.n_heads);
        printf("%s: n_layers       = %d\n", __func__, hparams.n_layers);
        printf("%s: n_vocab        = %d\n", __func__, hparams.n_vocab);
        printf("%s: alibi_bias_max = %f\n", __func__, hparams.alibi_bias_max);
        printf("%s: clip_qkv       = %f\n", __func__, hparams.clip_qkv);
        printf("%s: ftype          = %d\n", __func__, hparams.ftype);

        // 根据读取的文件信息，计算并打印 qntvr 值
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;
        printf("%s: qntvr          = %d\n", __func__, qntvr);

        // 根据读取的文件信息，更新 ftype 值
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // 加载词汇表
    {
        // 获取词汇表大小
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

            // 将单词从 utf-8 转换
            std::wstring word_multibytes = convert_to_wstring(word);
            word.resize(word_multibytes.size());
            for (size_t w = 0; w < word_multibytes.size(); w++) {
                word[w] = uint8_t(word_multibytes[w]);
            }

            // 将单词映射到 ID，ID映射到单词
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 对于大张量，我们可以选择将数据存储为16位浮点数或量化，以节省内存并加快计算速度
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype)(model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        // 打印错误信息并返回
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n", __func__, fname.c_str(),
                model.hparams.ftype);
        return false;
    }

    // 获取模型上下文
    auto & ctx = model.ctx;

    // 初始化上下文大小
    size_t ctx_size = 0;

    // 获取模型超参数和上下文大小
    const auto & hparams = model.hparams;
    const size_t n_ctx = hparams.n_ctx;
}
    {
        // 定义嵌入大小为模型维度
        const size_t n_embd = hparams.d_model;
        // 定义层数为超参数中的层数
        const size_t n_layer = hparams.n_layers;
        // 定义词汇量大小为超参数中的词汇量大小
        const size_t n_vocab = hparams.n_vocab;

        // 计算上下文大小，加上词嵌入权重的内存大小
        ctx_size += ggml_row_size(wtype,         n_embd * n_vocab); // wte_weight
        // 加上归一化层的权重内存大小
        ctx_size += ggml_row_size(GGML_TYPE_F32, n_embd);           // norm_f_weight

        // 加上每个层的 LayerNormalization 层的权重内存大小
        ctx_size += n_layer * (ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_weight

        // 加上每个层的注意力机制的权重内存大小
        ctx_size += n_layer * (ggml_row_size(wtype, 3 * n_embd * n_embd)); // attn_Wqkv_weight
        // 加上每个层的注意力输出投影的权重内存大小
        ctx_size += n_layer * (ggml_row_size(wtype,     n_embd * n_embd)); // attn_out_proj_weight

        // 加上每个层的 LayerNormalization 层的权重内存大小
        ctx_size += n_layer * (ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_weight

        // 加上每个层的前馈神经网络的权重内存大小
        ctx_size += n_layer * (ggml_row_size(wtype, 4 * n_embd * n_embd)); // mlp_mlp_up_weight
        // 加上每个层的前馈神经网络的权重内存大小
        ctx_size += n_layer * (ggml_row_size(wtype, 4 * n_embd * n_embd)); // mlp_mlp_down_weight

        // 加上每个位置的记忆键的内存大小
        ctx_size += n_ctx * n_layer * ggml_row_size(GGML_TYPE_F16, n_embd); // memory_k
        // 加上每个位置的记忆值的内存大小
        ctx_size += n_ctx * n_layer * ggml_row_size(GGML_TYPE_F16, n_embd); // memory_v

        // 加上对象开销的内存大小
        ctx_size += (1 + 6 * n_layer) * 512; // object overhead

        // 打印 ggml 上下文大小
        printf("%s: ggml ctx size = %6.2f MB\n", __func__, ctx_size / (1024.0 * 1024.0));
    }

    // 创建 ggml 上下文
    {
        // 初始化 ggml 上下文参数
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ false,
        };

        // 初始化 ggml 上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，则打印错误信息并返回 false
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 准备权重的内存
    }

    // 键 + 值的内存
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
    
        // 获取嵌入维度和层数
        const size_t n_embd  = hparams.d_model;
        const size_t n_layer = hparams.n_layers;
    
        // 计算记忆单元数量和元素数量
        const int64_t n_mem      = n_layer * n_ctx;
        const int64_t n_elements = n_embd  * n_mem;
    
        // 为模型的记忆分配内存
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F16, n_elements);
    
        // 计算内存大小
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);
    
        // 打印内存大小和记忆单元数量
        printf("%s: memory_size = %8.2f MB, n_mem = %" PRId64 "\n", __func__, memory_size / 1024.0 / 1024.0, n_mem);
    }
    
    // 加载权重
    }
    
    // 关闭文件流
    fin.close();
    
    // 返回加载成功
    return true;
// 评估变压器
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    预测的下一个标记的logits
//
bool mpt_eval(const mpt_model & model, const int n_threads, const int n_past,
              const std::vector<gpt_vocab::id> & embd_inp, std::vector<float> & embd_w, bool logits_all, size_t & mem_per_token) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_embd  = hparams.d_model;  // 嵌入维度
    const int n_layer = hparams.n_layers;  // 层数
    const int n_head  = hparams.n_heads;  // 头数
    const int n_vocab = hparams.n_vocab;  // 词汇量大小
    const int n_ctx   = hparams.n_ctx;  // 上下文大小
    const float eps   = 1e-5f;  // 微小值

    static size_t buf_size = 256u * 1024 * 1024;  // 缓冲区大小
    static void * buf = malloc(buf_size);  // 分配缓冲区内存

    // 使用2个临时缓冲区
    // TODO: 非常巧妙的解决方案 - 以更优雅的方式重新实现
    static size_t scr0_size = 256u*1024*1024;  // 临时缓冲区0大小
    static void * scr0 = malloc(scr0_size);  // 分配临时缓冲区0内存

    static size_t scr1_size = 256u*1024*1024;  // 临时缓冲区1大小
    static void * scr1 = malloc(scr1_size);  // 分配临时缓冲区1内存

    // 如果每个标记的内存占用大于0且总共的内存占用大于缓冲区大小
    if (mem_per_token > 0 && mem_per_token * N > buf_size) {
        const size_t buf_size_new = 1.1 * (mem_per_token * N); // 增加10%以考虑ggml对象的开销
        // printf("\n%s: reallocating buffer from %zu to %zu bytes\n", __func__,
        // buf_size, buf_size_new);

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

    struct ggml_context * ctx0 = ggml_init(params);  // 初始化上下文
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);  // 创建新的图形
    // 创建一个指向 ggml_tensor 结构体的指针 embd，该结构体表示一个一维张量，数据类型为 GGML_TYPE_I32，长度为 N
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 将 embd_inp 的数据复制到 embd 的数据中，长度为 N 乘以 embd 的元素大小
    memcpy(embd->data, embd_inp.data(), N * ggml_element_size(embd));

    // 创建一个指向 ggml_tensor 结构体的指针 inpL，该结构体表示一个张量，通过从 model.wte_weight 中获取 embd 行的数据
    struct ggml_tensor * inpL = ggml_get_rows(ctx0, model.wte_weight, embd);

    }

    // 设置上下文的临时存储区，大小为 scr0_size，数据为 scr0
    ggml_set_scratch(ctx0, { 0, scr0_size, scr0, });

    // norm
    {
        // 对 inpL 进行归一化处理，eps 为归一化的参数
        inpL = ggml_norm(ctx0, inpL, eps);
        // 将 inpL 与 ln_f_g*inpL 的乘积赋值给 inpL
        inpL = ggml_mul(ctx0, ggml_repeat(ctx0, model.norm_f_weight, inpL), inpL);
    }

    // 清空上下文的临时存储区
    ggml_set_scratch(ctx0, { 0, 0, nullptr, });

    // 将 model.wte_weight 与 inpL 的矩阵乘积赋值给 inpL
    inpL = ggml_mul_mat(ctx0, model.wte_weight, inpL);

    // logits -> probs
    // 对 inpL 进行 softmax 处理
    // inpL = ggml_soft_max(ctx0, inpL);

    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 如果 logits_all 为真，则返回所有标记的结果
    // 否则，只返回最后一个标记的结果
    if (logits_all) {
        embd_w.resize(n_vocab *N);
        memcpy(embd_w.data(), (float *)ggml_get_data(inpL) , sizeof(float) * n_vocab * N);
    } else {
        embd_w.resize(n_vocab);
        memcpy(embd_w.data(), (float *)ggml_get_data(inpL) + (n_vocab * (N - 1)), sizeof(float) * n_vocab);
    }

    // 如果 mem_per_token 为 0，则计算每个标记的内存使用量
    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0) / N;
    }
    // 打印使用的内存量
    // printf("used_mem = %zu\n", ggml_used_mem(ctx0));

    // 释放上下文的内存
    ggml_free(ctx0);

    // 返回 true
    return true;
}

// 计算 softmax 函数，将 logits 转换为概率值
std::vector<float> softmax(const std::vector<float> & logits) {
    // 创建一个与 logits 大小相同的概率数组
    std::vector<float> probs(logits.size());
    // 初始化最大 logit 值为 logits 的第一个元素
    float max_logit = logits[0];
    // 遍历 logits 数组，找到最大的 logit 值
    for (float v : logits) max_logit = std::max(max_logit, v);
    // 初始化指数和为 0
    double sum_exp = 0.0;
    // 遍历 logits 数组
    for (size_t i = 0; i < logits.size(); i++) {
        // 为了数值稳定性，从当前 logit 值中减去最大 logit 值
        const float logit = logits[i] - max_logit;
        // 计算 logit 的指数值
        const float exp_logit = expf(logit);
        // 更新指数和
        sum_exp += exp_logit;
        // 将指数值存入概率数组中
        probs[i] = exp_logit;
    }
    // 将概率数组中的每个值除以指数和，得到最终的概率值
    for (size_t i = 0; i < probs.size(); i++) probs[i] /= sum_exp;
    // 返回概率数组
    return probs;
}

// 计算 perplexity 函数
int perplexity(const mpt_params & params) {
    // 初始化时间
    ggml_time_init();

    // 获取主函数开始时间
    const int64_t t_main_start_us = ggml_time_us();

    // 打印参数信息
    printf("%s: n_threads = %d\n", __func__, params.n_threads);
    printf("%s: n_batch   = %d\n", __func__, params.n_batch);
    printf("%s: n_ctx     = %d\n", __func__, params.n_ctx);
    printf("\n");

    // 初始化加载时间
    int64_t t_load_us = 0;

    // 初始化词汇表和模型
    gpt_vocab vocab;
    mpt_model model;

    // 设置模型参数
    model.hparams.n_ctx = params.n_ctx;

    // 加载模型
    {
        // 获取加载模型开始时间
        const int64_t t_start_us = ggml_time_us();

        // 如果加载模型失败，打印错误信息并返回
        if (!mpt_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        // 计算加载模型时间
        t_load_us = ggml_time_us() - t_start_us;
    }

    // 初始化预测时间
    int64_t t_predict_us = 0;

    // 初始化 logits 数组
    std::vector<float> logits;

    // 对提示进行标记化处理
    std::vector<int> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    // 打印提示中的标记数量
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());

    // 确定每个标记所需的推理内存
    size_t mem_per_token = 0;
    mpt_eval(model, params.n_threads, 0, {0, 1, 2, 3}, logits, false, mem_per_token);

    // 初始化计数
    int count   = 0;

    // 计算提示的块数
    const int n_chunk = embd_inp.size() / params.n_ctx;

    // 获取词汇表大小和批次大小
    const int n_vocab = model.hparams.n_vocab;
    const int n_batch = params.n_batch;
    // 初始化一个双精度浮点数变量nll，并赋值为0.0
    double nll = 0.0;
    // 打印计算困惑度的信息，包括计算的块数和批处理大小
    fprintf(stderr, "%s: calculating perplexity over %d chunks, batch_size=%d\n", __func__, n_chunk, n_batch);

    }

    // 报告计时信息
    {
        // 获取主函数结束时的时间戳
        const int64_t t_main_end_us = ggml_time_us();

        // 打印内存每个标记的字节数
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        // 打印加载时间
        printf("%s:     load time = %8.2f ms\n",   __func__, t_load_us / 1000.0f);
        // 打印评估时间和每个标记的平均评估时间
        printf("%s:     eval time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us / 1000.0f, t_predict_us / 1000.0f / (n_chunk * params.n_ctx));
        // 打印总时间
        printf("%s:    total time = %8.2f ms\n",   __func__, (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    // 释放模型上下文的内存
    ggml_free(model.ctx);

    // 返回0，表示成功执行
    return 0;
}

int main(int argc, char ** argv) {
    // 定义参数结构体
    mpt_params params;

    // 解析命令行参数，如果解析失败则返回1
    if (mpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果参数中包含 perplexity，则返回 perplexity 函数的结果
    if (params.perplexity) {
        return perplexity(params);
    }

    // 初始化时间
    ggml_time_init();

    // 获取程序开始执行的时间
    const int64_t t_main_start_us = ggml_time_us();

    // 如果参数中的种子小于0，则设置种子为当前时间
    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    // 如果参数中的 n_predict 小于0，则设置为0
    if (params.n_predict < 0) {
        params.n_predict = 0;
    }

    // 打印参数信息
    printf("%s: seed      = %d\n",   __func__, params.seed);
    printf("%s: n_threads = %d\n",   __func__, params.n_threads);
    printf("%s: n_batch   = %d\n",   __func__, params.n_batch);
    printf("%s: n_ctx     = %d\n",   __func__, params.n_ctx);
    printf("%s: n_predict = %d\n\n", __func__, params.n_predict);

    // 使用种子初始化随机数生成器
    std::mt19937 rng(params.seed);
    // 如果参数中的 prompt 为空，则生成一个随机的 prompt
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    // 初始化变量 t_load_us
    int64_t t_load_us = 0;

    // 定义词汇表和模型
    gpt_vocab vocab;
    mpt_model model;

    // 设置模型的上下文长度
    model.hparams.n_ctx = params.n_ctx;

    // 加载模型
    {
        // 获取加载模型的开始时间
        const int64_t t_start_us = ggml_time_us();

        // 如果加载模型失败，则打印错误信息并返回1
        if (!mpt_model_load(params.model, model, vocab)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        // 计算加载模型所用的时间
        t_load_us = ggml_time_us() - t_start_us;

        // 测试 GPT 分词器
        test_gpt_tokenizer(vocab, params.token_test);
    }

    // 如果参数中的 top_k 为0，则设置为模型词汇表的大小
    if (params.top_k == 0) {
        params.top_k = model.hparams.n_vocab;
    }

    // 如果参数中的 repeat_last_n 为-1，则设置为参数中的 n_ctx
    if (params.repeat_last_n == -1) {
        params.repeat_last_n = params.n_ctx;
    }

    // 打印参数信息
    printf("\n");
    printf("%s: temp           = %.3f\n", __func__, params.temp);
    printf("%s: top_k          = %d\n",   __func__, params.top_k);
    printf("%s: top_p          = %.3f\n", __func__, params.top_p);
    printf("%s: repeat_last_n  = %d\n",   __func__, params.repeat_last_n);
    printf("%s: repeat_penalty = %.3f\n", __func__, params.repeat_penalty);

    // 初始化变量 t_sample_us 和 t_predict_us
    int64_t t_sample_us = 0;
    int64_t t_predict_us = 0;
    // 创建一个大小为params.n_ctx的int32_t类型的vector，并将所有元素初始化为0
    std::vector<int32_t> last_n_tokens(params.n_ctx);
    std::fill(last_n_tokens.begin(), last_n_tokens.end(), 0);

    // 对提示进行标记化处理
    std::vector<int> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    // 打印换行符
    printf("\n");
    // 打印函数名和提示中的标记数量
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());

    // 遍历提示中的标记并打印它们的索引和值
    for (size_t i = 0; i < embd_inp.size(); i++) {
        printf("%s: token[%zu] = %6d\n", __func__, i, embd_inp[i]);
    }
    // 打印换行符
    printf("\n");

    // 创建两个空的vector，一个存储标记的id，一个存储logits
    std::vector<gpt_vocab::id> embd;
    std::vector<float> logits;

    // 确定每个标记所需的推理内存
    size_t mem_per_token = 0;
    mpt_eval(model, params.n_threads, 0, {0, 1, 2, 3}, logits, false, mem_per_token);

    // 初始化变量
    int n_past     = 0;
    int n_consumed = 0;
    int n_sampled  = 0;

    }

    // 报告时间
    {
        // 获取当前时间
        const int64_t t_main_end_us = ggml_time_us();

        // 打印抽样的标记数量、每个标记的内存占用、加载时间、抽样时间、评估时间和总时间
        printf("\n\n\n");
        printf("%s: sampled tokens = %8d\n", __func__, n_sampled);
        printf("%s:  mem per token = %8zu bytes\n", __func__, mem_per_token);
        printf("%s:      load time = %8.2f ms\n", __func__, t_load_us / 1000.0f);
        printf("%s:    sample time = %8.2f ms / %.2f ms per token\n", __func__, t_sample_us / 1000.0f, t_sample_us / 1000.0f / n_sampled);
        printf("%s:      eval time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us / 1000.0f, t_predict_us / 1000.0f / n_past);
        printf("%s:     total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us) / 1000.0f);
    }

    // 释放模型上下文内存
    ggml_free(model.ctx);

    // 返回0表示成功
    return 0;
# 闭合前面的函数定义
```