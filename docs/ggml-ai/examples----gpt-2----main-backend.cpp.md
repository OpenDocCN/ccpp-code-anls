# `ggml\examples\gpt-2\main-backend.cpp`

```
#include "ggml/ggml.h"
#include "ggml/ggml-alloc.h"
#include "ggml/ggml-backend.h"

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif

#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif

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

#define GPT2_MAX_NODES 4096

static void ggml_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);  // 输出日志信息到标准错误流
    fflush(stderr);  // 刷新标准错误流
}

// default hparams (GPT-2 117M)
struct gpt2_hparams {
    int32_t n_vocab = 50257;  // 词汇表大小
    int32_t n_ctx   = 1024;   // 上下文大小
    int32_t n_embd  = 768;    // 嵌入维度
    int32_t n_head  = 12;     // 头数
    int32_t n_layer = 12;     // 层数
    int32_t ftype   = 1;      // 特征类型
    float   eps     = 1e-5f;  // epsilon值
};

struct gpt2_layer {
    // normalization
    struct ggml_tensor * ln_1_g;  // Layer Normalization 1 的缩放参数
    struct ggml_tensor * ln_1_b;  // Layer Normalization 1 的偏移参数

    struct ggml_tensor * ln_2_g;  // Layer Normalization 2 的缩放参数
    struct ggml_tensor * ln_2_b;  // Layer Normalization 2 的偏移参数

    // attention
    struct ggml_tensor * c_attn_attn_w;  // 注意力机制的权重
    struct ggml_tensor * c_attn_attn_b;  // 注意力机制的偏置

    struct ggml_tensor * c_attn_proj_w;  // 注意力机制的投影权重
    struct ggml_tensor * c_attn_proj_b;  // 注意力机制的投影偏置

    // mlp
    struct ggml_tensor * c_mlp_fc_w;  // 多层感知机的全连接层权重
    struct ggml_tensor * c_mlp_fc_b;  // 多层感知机的全连接层偏置

    struct ggml_tensor * c_mlp_proj_w;  // 多层感知机的投影层权重
    struct ggml_tensor * c_mlp_proj_b;  // 多层感知机的投影层偏置
};

struct gpt2_model {
    gpt2_hparams hparams;  // GPT-2 模型的超参数

    // normalization
    struct ggml_tensor * ln_f_g;  // 最终层的 Layer Normalization 的缩放参数
    struct ggml_tensor * ln_f_b;  // 最终层的 Layer Normalization 的偏移参数

    struct ggml_tensor * wte;     // 位置嵌入
    struct ggml_tensor * wpe;     // 词嵌入
    struct ggml_tensor * lm_head; // 语言模型头部

    std::vector<gpt2_layer> layers;  // GPT-2 模型的多层结构

    // key + value memory
    struct ggml_tensor * memory_k;  // 关键字记忆
    struct ggml_tensor * memory_v;  // 值记忆

    //
    struct ggml_context * ctx;  // 上下文

    ggml_backend_t backend = NULL;  // 后端
}
    # 定义一个名为buffer_w的ggml_backend_buffer_t类型的变量
    ggml_backend_buffer_t buffer_w;
    # 定义一个名为buffer_kv的ggml_backend_buffer_t类型的变量
    ggml_backend_buffer_t buffer_kv;
    # 定义一个名为tensors的std::map，键为string类型，值为指向ggml_tensor结构体的指针
    std::map<std::string, struct ggml_tensor *> tensors;
};

// 从文件中加载模型的权重
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab, int n_ctx, int n_gpu_layers) {
    // 打印加载模型的信息
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    // 打开文件输入流
    auto fin = std::ifstream(fname, std::ios::binary);
    // 如果无法打开文件，则打印错误信息并返回false
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // 验证文件的魔数
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        // 如果魔数不匹配，则打印错误信息并返回false
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // 加载超参数
    {
        auto & hparams = model.hparams;

        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        // 打印超参数信息
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

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

        // 读取词汇表中的单词并建立单词到索引的映射关系
        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            // 读取单词的长度
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            // 调整缓冲区大小以容纳单词
            buf.resize(len);
            // 读取单词数据
            fin.read((char *) buf.data(), len);
            // 将单词数据转换为字符串
            word.assign(buf.data(), len);

            // 建立单词到索引的映射关系
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 对于大张量，我们可以选择将数据存储为16位浮点数或量化形式
    // 以节省内存并加快计算速度
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    // 创建 ggml 上下文
    auto & ctx = model.ctx;
    {
        // 初始化参数
        size_t n_tensors = 2 + 6 + 12*model.hparams.n_layer;
        struct ggml_init_params params = {
            /*.mem_size   =*/ ggml_tensor_overhead() * n_tensors,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
        };

        // 初始化 ggml 上下文
        ctx = ggml_init(params);
        if (!ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 初始化后端
#ifdef GGML_USE_CUBLAS
    // 如果使用 CUDA 后端，并且有 GPU 层，则打印信息并初始化 CUDA 后端
    if (n_gpu_layers > 0) {
        fprintf(stderr, "%s: using CUDA backend\n", __func__);
        model.backend = ggml_backend_cuda_init(0);
        // 如果初始化失败，则打印错误信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#endif

#ifdef GGML_USE_METAL
    // 如果使用 Metal 后端，并且有 GPU 层，则打印信息并初始化 Metal 后端
    if (n_gpu_layers > 0) {
        fprintf(stderr, "%s: using Metal backend\n", __func__);
        // 设置 Metal 后端的日志回调函数
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        model.backend = ggml_backend_metal_init();
        // 如果初始化失败，则打印错误信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_metal_init() failed\n", __func__);
        }
    }
#endif

    // 如果没有选择任何后端，则回退到 CPU 后端
    if (!model.backend) {
        fprintf(stderr, "%s: using CPU backend\n", __func__);
        // 初始化 CPU 后端
        model.backend = ggml_backend_cpu_init();
    }

    // 如果初始化后端失败，则打印错误信息并返回 false
    if (!model.backend) {
        fprintf(stderr, "%s: ggml_backend_cpu_init() failed\n", __func__);
        return false;
    }

    // 创建模型的张量

    // 在后端缓冲区中分配模型张量
    model.buffer_w = ggml_backend_alloc_ctx_tensors(ctx, model.backend);

    // 打印模型张量大小和后端缓冲区大小
    printf("%s: ggml tensor size    = %d bytes\n", __func__, (int) sizeof(ggml_tensor));
    printf("%s: backend buffer size = %6.2f MB\n", __func__, ggml_backend_buffer_get_size(model.buffer_w)/(1024.0*1024.0));

    // 使用用户提供的训练上下文覆盖默认的训练上下文
    model.hparams.n_ctx = n_ctx;

    // 键 + 值内存
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;

        // 获取嵌入层的大小
        const int n_embd  = hparams.n_embd;
        // 获取层数
        const int n_layer = hparams.n_layer;
        // 获取上下文的大小
        const int n_ctx   = hparams.n_ctx;

        // 计算总的记忆单元数量
        const int n_mem      = n_layer*n_ctx;
        // 计算总的元素数量
        const int n_elements = n_embd*n_mem;

        // 创建存储记忆键的张量
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        // 创建存储记忆值的张量
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        // 计算内存大小
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        // 打印内存大小和记忆单元数量
        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);

        // 创建后端缓冲区（可以是主机内存或设备内存）
        model.buffer_kv = ggml_backend_alloc_buffer(model.backend, memory_size + 256);

        // 将张量分配到后端缓冲区中
        {
            ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer_kv);

            // 这将更新张量中的指针，使其指向缓冲区中的正确位置
            // 这是必要的，因为 ggml_context 是 .no_alloc == true
            // 请注意，缓冲区实际上可以是设备缓冲区，这取决于后端
            ggml_allocr_alloc(alloc, model.memory_k);
            ggml_allocr_alloc(alloc, model.memory_v);

            ggml_allocr_free(alloc);
        }
    }

    // 加载权重
#ifdef GGML_USE_METAL
                || ggml_backend_is_metal(model.backend)
#endif
                ) {
                // 如果使用 Metal 后端或者 CPU 后端，可以直接读取到张量中
                fin.read(reinterpret_cast<char *>(tensor->data), ggml_nbytes(tensor));
            } else {
                // 先读取到临时缓冲区，然后再复制到设备内存中
                read_buf.resize(ggml_nbytes(tensor));
                fin.read(read_buf.data(), ggml_nbytes(tensor));
                ggml_backend_tensor_set(tensor, read_buf.data(), 0, ggml_nbytes(tensor));
            }

            // GPT-2 模型将 WTE 张量共享为 LM head
            if (name == "model/wte" && has_lm_head == false) {
                //ggml_allocr_alloc(alloc, model.lm_head);
                //ggml_backend_tensor_copy(tensor, model.lm_head);
                model.lm_head = tensor;
            }

            if (name == "model/lm_head") {
                has_lm_head = true;
            }

            total_size += ggml_nbytes(tensor);
        }

        printf("%s: model size  = %8.2f MB\n", __func__, total_size/1024.0/1024.0);
    }

    fin.close();

    return true;
}

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

    // 由于使用了 ggml-alloc，这个缓冲区只需要足够容纳 ggml_tensor 和 ggml_cgraph 结构，而不需要张量数据的空间
    static size_t buf_size = ggml_tensor_overhead()*GPT2_MAX_NODES + ggml_graph_overhead_custom(GPT2_MAX_NODES, false);
    static std::vector<uint8_t> buf(buf_size);
    // 初始化参数结构体，设置内存大小和内存缓冲区
    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf.data(),
        /*.no_alloc   =*/ true, // the tensors will be allocated later by ggml_allocr_alloc_graph()
    };

    // 初始化上下文
    struct ggml_context * ctx0 = ggml_init(params);

    // 创建自定义图形
    struct ggml_cgraph  * gf = ggml_new_graph_custom(ctx0, GPT2_MAX_NODES, false);

    // 创建新的一维张量 embd
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 分配张量
    ggml_allocr_alloc(allocr, embd);

    // 如果不是在测量内存使用量，则设置张量的值
    if (!ggml_allocr_is_measure(allocr)) {
        ggml_backend_tensor_set(embd, embd_inp.data(), 0, N*ggml_element_size(embd));
    }

    // 创建新的一维张量 position
    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 分配张量
    ggml_allocr_alloc(allocr, position);
    // 如果不是在测量内存使用量，则设置张量的值
    if (!ggml_allocr_is_measure(allocr)) {
        for (int i = 0; i < N; ++i) {
            int32_t v = n_past + i;
            ggml_backend_tensor_set(position, &v, i*sizeof(int32_t), sizeof(v));
        }
    }

    // 计算 wte + wpe
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
                    inpL,
                    model.ln_f_g),
                model.ln_f_b);
    }

    // 计算 WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // 计算 logits -> probs
    //inpL = ggml_soft_max(ctx0, inpL);

    // 构建前向扩展
    ggml_build_forward_expand(gf, inpL);

    // 释放上下文
    ggml_free(ctx0);

    // 返回图形
    return gf;
// 评估变换器
//
//   - model:     模型
//   - allocr:    用于分配计算缓冲区的 ggml_allocr
//   - n_threads: 要使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    下一个标记的预测逻辑
//
bool gpt2_eval(
        const gpt2_model & model,
        struct ggml_allocr * allocr,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w) {
    const int N = embd_inp.size();  // 获取上下文中标记的数量

    const auto & hparams = model.hparams;  // 获取模型的超参数

    const int n_vocab = hparams.n_vocab;  // 获取词汇表的大小

    // 重置分配器以释放上一次推断期间分配的所有内存
    ggml_allocr_reset(allocr);

    struct ggml_cgraph * gf = gpt2_graph(model, allocr, n_past, embd_inp);  // 创建计算图

    // 分配张量
    ggml_allocr_alloc_graph(allocr, gf);

    // 设置后端选项
    if (ggml_backend_is_cpu(model.backend)) {  // 如果后端是 CPU
        ggml_backend_cpu_set_n_threads(model.backend, n_threads);  // 设置线程数
    }

#ifdef GGML_USE_METAL
    if (ggml_backend_is_metal(model.backend)) {  // 如果后端是 Metal
        ggml_backend_metal_set_n_cb(model.backend, n_threads);  // 设置回调数
    }
#endif

    // 测试
#if 0 && defined(GGML_USE_CUBLAS)
    if (ggml_backend_is_cuda(model.backend)) {  // 如果后端是 CUDA
        auto eval_callback = [](int index, struct ggml_tensor * t1, struct ggml_tensor * t2, void * user_data) {  // 创建评估回调函数
            auto tv1 = tensor_to_float(t1);  // 将张量转换为浮点数
            auto tv2 = tensor_to_float(t2);  // 将张量转换为浮点数
#if 1
            // 计算两个向量的余弦相似度
            float sim = cosine_similarity(tv1, tv2);
            // 计算第一个向量的长度
            float len1 = vec_len(tv1);
            // 计算第二个向量的长度
            float len2 = vec_len(tv2);
            // 计算长度比值
            float lenr = len1/len2;
            // 计算长度比值的绝对差
            float lenrd = std::abs(1.0f-lenr);

            // 计算两个向量的夹角
            float angle = acosf(sim)*180.0f/M_PI;

            // 如果夹角大于0.5或长度比值的绝对差大于0.05，则输出相关信息
            if (angle > 0.5f || lenrd > 0.05f) {
                printf("%3d [%15s] %s: sim = %f, a = %f, lenrd = %f\n", index, ggml_op_desc(t1), t1->name, sim, angle, lenrd);
            }
            // 断言余弦相似度大于0.90
            assert(sim > 0.90f);
#else
            // 计算两个向量的欧氏距离，并除以第一个向量的长度
            float dist = distance(tv1, tv2) / vec_len(tv1);
            // 如果距离大于0.01，则输出相关信息
            if (dist > 0.01f) {
                printf("%3d [%15s] %s: distance = %f\n", index, ggml_op_desc(t1), t1->name, dist);
            }
#endif

            // 返回 true
            return true;
        };
        // 初始化 CPU 后端
        ggml_backend_t backend_cpu = ggml_backend_cpu_init();
        // 比较图的后端
        ggml_backend_compare_graph_backend(model.backend, backend_cpu, gf, eval_callback, nullptr);
        // 释放 CPU 后端
        ggml_backend_free(backend_cpu);
        //printf("done\n");
    } else
#endif
    {
        // 运行计算
        ggml_backend_graph_compute(model.backend, gf);
    }

    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // 在这种情况下，输出张量是图中的最后一个张量
    struct ggml_tensor * inpL = gf->nodes[gf->n_nodes - 1];

    //embd_w.resize(n_vocab*N);
    //ggml_backend_tensor_get(inpL, embd_w.data(), 0, sizeof(float)*n_vocab*N);

    // 仅返回最后一个标记的结果
    embd_w.resize(n_vocab);
    ggml_backend_tensor_get(inpL, embd_w.data(), (n_vocab*(N-1))*sizeof(float), sizeof(float)*n_vocab);

    // 返回 true
    return true;
}

// 主函数
int main(int argc, char ** argv) {
    // 初始化时间
    ggml_time_init();

    // 获取主函数开始时间
    const int64_t t_main_start_us = ggml_time_us();

    // 初始化参数
    gpt_params params;
    params.model = "models/gpt-2-117M/ggml-model.bin";

    // 解析参数
    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 如果种子小于0，则设置为当前时间
    if (params.seed < 0) {
        params.seed = time(NULL);
    // 打印函数名和种子值
    printf("%s: seed = %d\n", __func__, params.seed);

    // 使用种子值创建伪随机数生成器
    std::mt19937 rng(params.seed);
    
    // 如果提示为空，则生成一个随机提示
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    // 初始化变量 t_load_us 为 0
    int64_t t_load_us = 0;

    // 初始化词汇表和 GPT2 模型
    gpt_vocab vocab;
    gpt2_model model;

    // 加载模型
    {
        // 记录加载模型的起始时间
        const int64_t t_start_us = ggml_time_us();

        // 如果加载模型失败，则打印错误信息并返回 1
        if (!gpt2_model_load(params.model, model, vocab, params.n_ctx, params.n_gpu_layers)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        // 计算加载模型所花费的时间
        t_load_us = ggml_time_us() - t_start_us;

        // 测试 GPT 分词器
        test_gpt_tokenizer(vocab, params.token_test);
    }

    // 在评估模型时保持此缓冲区有效
    ggml_backend_buffer_t buf_compute;

    // 分配计算缓冲区
    {
        // 创建一个分配器来测量内存使用情况
        allocr = ggml_allocr_new_measure_from_backend(model.backend);

        // 创建内存使用情况估计的最坏情况图形
        int n_tokens = std::min(model.hparams.n_ctx, params.n_batch);
        int n_past = model.hparams.n_ctx - n_tokens;
        struct ggml_cgraph * gf = gpt2_graph(model, allocr, n_past, std::vector<gpt_vocab::id>(n_tokens, 0));

        // 计算所需的内存
        size_t mem_size = ggml_allocr_alloc_graph(allocr, gf);

        // 使用所需的内存重新创建分配器
        ggml_allocr_free(allocr);
        buf_compute = ggml_backend_alloc_buffer(model.backend, mem_size);
        allocr = ggml_allocr_new_from_buffer(buf_compute);

        // 打印计算缓冲区的大小
        fprintf(stderr, "%s: compute buffer size: %.2f MB\n", __func__, mem_size/1024.0/1024.0);
    }

    // 初始化变量 n_past 为 0
    int n_past = 0;

    // 初始化采样时间和预测时间为 0
    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    // 初始化 logits 为一个空的浮点数向量
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
    // 从当前嵌入大小开始循环直到嵌入输入大小加上预测数量
    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // 预测
        if (embd.size() > 0) {
            // 记录开始预测的时间
            const int64_t t_start_us = ggml_time_us();

            // 如果无法成功预测，则打印错误信息并返回1
            if (!gpt2_eval(model, allocr, params.n_threads, n_past, embd, logits)) {
                printf("Failed to predict\n");
                return 1;
            }

            // 计算预测时间并累加到总预测时间
            t_predict_us += ggml_time_us() - t_start_us;
        }

        // 更新过去的数量
        n_past += embd.size();
        // 清空嵌入
        embd.clear();

        // 如果当前索引大于等于嵌入输入大小
        if (i >= embd_inp.size()) {
            // 采样下一个标记
            const int   top_k = params.top_k;
            const float top_p = params.top_p;
            const float temp  = params.temp;

            const int n_vocab = model.hparams.n_vocab;

            gpt_vocab::id id = 0;

            {
                // 记录开始采样的时间
                const int64_t t_start_sample_us = ggml_time_us();

                // 从logits中采样下一个标记
                id = gpt_sample_top_k_top_p(vocab, logits.data() + (logits.size() - n_vocab), top_k, top_p, temp, rng);

                // 计算采样时间并累加到总采样时间
                t_sample_us += ggml_time_us() - t_start_sample_us;
            }

            // 将采样的标记添加到上下文中
            embd.push_back(id);
        } else {
            // 如果执行到这里，意味着我们仍在处理输入提示
            for (size_t k = i; k < embd_inp.size(); k++) {
                embd.push_back(embd_inp[k]);
                // 如果嵌入大小达到批处理大小，则跳出循环
                if (int32_t(embd.size()) >= params.n_batch) {
                    break;
                }
            }
            // 更新索引
            i += embd.size() - 1;
        }

        // 显示文本
        for (auto id : embd) {
            printf("%s", vocab.id_to_token[id].c_str());
        }
        fflush(stdout);

        // 文本结束标记
        if (!params.ignore_eos && embd.back() == 50256) {
            break;
        }
    }

    // 报告时间
    {
        // 计算主函数结束时间
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

    // 释放模型权重缓冲区
    ggml_backend_buffer_free(model.buffer_w);
    // 释放模型键值缓冲区
    ggml_backend_buffer_free(model.buffer_kv);
    // 释放计算缓冲区
    ggml_backend_buffer_free(buf_compute);
    // 释放后端资源
    ggml_backend_free(model.backend);

    // 返回成功状态
    return 0;
}
# 闭合前面的函数定义
```