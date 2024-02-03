# `ggml\examples\gpt-2\main.cpp`

```cpp
#include "ggml/ggml.h"
#include "ggml/ggml-alloc.h"
#include "ggml/ggml-backend.h"
// 包含所需的头文件

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif
// 如果定义了 GGML_USE_CUBLAS，则包含 ggml-cuda.h

#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif
// 如果定义了 GGML_USE_METAL，则包含 ggml-metal.h

#include "common.h"
#include "common-ggml.h"
// 包含通用的头文件

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
// 包含标准库的头文件

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif
// 如果定义了 _MSC_VER，则禁用警告 4244 和 4267

#define GPT2_MAX_NODES 4096
// 定义 GPT2_MAX_NODES 为 4096

static void ggml_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);
    fflush(stderr);
}
// 定义默认的日志回调函数 ggml_log_callback_default

// default hparams (GPT-2 117M)
struct gpt2_hparams {
    int32_t n_vocab = 50257;
    int32_t n_ctx   = 1024;
    int32_t n_embd  = 768;
    int32_t n_head  = 12;
    int32_t n_layer = 12;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};
// 定义默认的超参数结构体 gpt2_hparams，包括词汇量、上下文长度、嵌入维度、头数、层数、ftype 和 eps

struct gpt2_layer {
    // normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // attention
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // mlp
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};
// 定义 GPT-2 层的结构体 gpt2_layer，包括归一化、注意力和多层感知机的张量指针

struct gpt2_model {
    gpt2_hparams hparams;

    // normalization
    struct ggml_tensor * ln_f_g;
    struct ggml_tensor * ln_f_b;

    struct ggml_tensor * wte;     // position embedding
    struct ggml_tensor * wpe;     //    token embedding
    struct ggml_tensor * lm_head; // language model head

    std::vector<gpt2_layer> layers;

    // key + value memory
    struct ggml_tensor * memory_k;
    struct ggml_tensor * memory_v;

    //
    struct ggml_context * ctx;

    std::vector<ggml_backend_t> backends;
};
// 定义 GPT-2 模型的结构体 gpt2_model，包括超参数、归一化、位置嵌入、标记嵌入、语言模型头、层、键值记忆、上下文和后端
    // 创建一个存储 ggml_backend_buffer_t 类型数据的向量
    std::vector<ggml_backend_buffer_t> buffers_w;
    // 创建一个 ggml_backend_buffer_t 类型的变量
    ggml_backend_buffer_t buffer_kv;
    // 创建一个 ggml_backend_buffer_t 类型的变量
    ggml_backend_buffer_t buffer_input;

    // 创建一个存储 string 到 ggml_tensor* 的映射
    std::map<std::string, struct ggml_tensor *> tensors;

    // 创建一个 ggml_tensor* 类型的变量，用于存储输入数据
    struct ggml_tensor * embd;
    // 创建一个 ggml_tensor* 类型的变量，用于存储位置数据
    struct ggml_tensor * position;
};

// 初始化后端
void init_backends(gpt2_model & model, const gpt_params & params) {
    ggml_backend_t gpu_backend = NULL;

    // 初始化后端
#ifdef GGML_USE_CUBLAS
    if (params.n_gpu_layers > 0) {
        fprintf(stderr, "%s: using CUDA backend\n", __func__);
        // 使用 CUDA 后端
        gpu_backend = ggml_backend_cuda_init(0);
        if (!gpu_backend) {
            fprintf(stderr, "%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#endif

#ifdef GGML_USE_METAL
    if (params.n_gpu_layers > 0) {
        fprintf(stderr, "%s: using Metal backend\n", __func__);
        // 使用 Metal 后端
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        gpu_backend = ggml_backend_metal_init();
        if (!gpu_backend) {
            fprintf(stderr, "%s: ggml_backend_metal_init() failed\n", __func__);
        } else {
            ggml_backend_metal_set_n_cb(gpu_backend, params.n_threads);
        }
    }
#endif
    // 如果存在 GPU 后端，则添加到模型的后端列表中
    if (gpu_backend) {
        model.backends.push_back(gpu_backend);
    }

    // 总是将 CPU 后端作为备用添加到模型的后端列表中
    ggml_backend_t cpu_backend = ggml_backend_cpu_init();
    ggml_backend_cpu_set_n_threads(cpu_backend, params.n_threads);
    model.backends.push_back(cpu_backend);
}

// 从文件加载模型的权重
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab, const gpt_params & params) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    auto fin = std::ifstream(fname, std::ios::binary);
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        return false;
    }

    // 验证魔数
    {
        uint32_t magic;
        fin.read((char *) &magic, sizeof(magic));
        if (magic != GGML_FILE_MAGIC) {
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            return false;
        }
    }

    // 加载超参数
    {
        // 获取模型的超参数引用
        auto & hparams = model.hparams;

        // 从文件中读取并存储词汇量
        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        // 从文件中读取并存储上下文长度
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        // 从文件中读取并存储嵌入维度
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        // 从文件中读取并存储头数
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        // 从文件中读取并存储层数
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        // 从文件中读取并存储数据类型
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        // 计算量化版本
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        // 打印超参数信息
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        // 更新数据类型
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // 加载词汇表
    {
        // 读取并存储词汇表大小
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        // 检查词汇表大小是否与模型中的词汇表大小一致
        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        // 读取词汇表中的词和对应的 ID
        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;
        }
    }

    // 对于大张量，我们有选项将数据存储为16位浮点数或量化形式，以节省内存并加快计算速度
    // 将模型的文件类型转换为对应的 ggml 类型
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    // 如果文件类型无效，输出错误信息并返回 false
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    // 获取模型的上下文
    auto & ctx = model.ctx;

    // 创建 ggml 上下文
    {
        // 计算需要的张量数量
        size_t n_tensors = 3 /* input */ + 2 /* kv */ + 6 + 12*model.hparams.n_layer;
        // 初始化参数
        struct ggml_init_params params = {
            /*.mem_size   =*/ ggml_tensor_overhead() * n_tensors,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
        };

        // 初始化 ggml 上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，输出错误信息并返回 false
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 创建权重的张量

    // 将张量分配给后端
    init_backends(model, params);
    // 获取 GPU 和 CPU 后端
    ggml_backend_t backend_gpu = model.backends.front();
    ggml_backend_t backend_cpu = model.backends.back();
    // 创建张量和后端的映射
    std::map<std::string, ggml_backend_t> tensor_backends;
    {
        // 定义第一个 GPU 层的索引
        const int i_gpu_first_layer = model.hparams.n_layer - params.n_gpu_layers;
        // 遍历模型张量
        for (auto it : model.tensors) {
            // 获取张量的名称
            const std::string & name = it.first;
            // 输入张量
            if (name == "model/wte" || name == "model/wpe") {
                // 如果 GPU 层数大于模型层数，将张量放入 GPU 后端
                if (params.n_gpu_layers > model.hparams.n_layer) {
                    tensor_backends[name] = backend_gpu;
                } else {
                    // 否则将张量放入 CPU 后端
                    tensor_backends[name] = backend_cpu;
                }
            }
            // 输出张量
            if (name == "model/ln_f/g" || name == "model/ln_f/b" || name == "model/lm_head") {
                // 如果 GPU 层数大于 0，将张量放入 GPU 后端
                if (params.n_gpu_layers > 0) {
                    tensor_backends[name] = backend_gpu;
                } else {
                    // 否则将张量放入 CPU 后端
                    tensor_backends[name] = backend_cpu;
                }
            }
            // 层张量
            if (name.substr(0, 7) == "model/h") {
                // 解析层编号
                int layer = std::stoi(name.substr(7, 2));
                // 如果层编号大于等于第一个 GPU 层的索引，将张量放入 GPU 后端
                if (layer >= i_gpu_first_layer) {
                    tensor_backends[name] = backend_gpu;
                } else {
                    // 否则将张量放入 CPU 后端
                    tensor_backends[name] = backend_cpu;
                }
            }
        }
    }
    
    // 分配缓冲区
    std::map<ggml_backend_t, std::unique_ptr<ggml_allocr, decltype(&ggml_allocr_free)>> backend_buffers;
    // 遍历模型的后端列表
    for (auto backend : model.backends) {
        // 计算缓冲区的大小
        size_t size = 0;
        // 遍历模型的张量
        for (auto it : model.tensors) {
            // 如果张量对应的后端与当前后端相同
            if (tensor_backends[it.first] == backend) {
                // 计算张量占用内存大小并加上额外的512字节
                size += ggml_nbytes(it.second) + 512;
            }
        }
        // 如果缓冲区大小大于0
        if (size > 0) {
            // 打印函数名、后端名称和缓冲区大小（以MB为单位）
            printf("%s: %8s buffer size = %8.2f MB\n", __func__, ggml_backend_name(backend), size/1024.0/1024.0);
            // 分配缓冲区
            ggml_backend_buffer_t buffer = ggml_backend_alloc_buffer(backend, size);
            // 将缓冲区添加到模型的缓冲区列表中
            model.buffers_w.push_back(buffer);

            // 为缓冲区创建一个分配器以分配张量
            auto alloc = std::unique_ptr<ggml_allocr, decltype(&ggml_allocr_free)>(ggml_allocr_new_from_buffer(buffer), ggml_allocr_free);
            // 将后端和分配器的组合插入到后端缓冲区映射中
            backend_buffers.insert(std::make_pair(backend, std::move(alloc)));
        } else {
            // 如果缓冲区大小为0，则将NULL添加到模型的缓冲区列表中
            model.buffers_w.push_back(NULL);
        }
    }

    // 分配键和值的内存
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;

        // 获取嵌入层的大小
        const int n_embd  = hparams.n_embd;
        // 获取层数
        const int n_layer = hparams.n_layer;
        // 获取上下文的大小
        const int n_ctx   = hparams.n_ctx;

        // 计算记忆单元的数量
        const int n_mem      = n_layer*n_ctx;
        // 计算元素的数量
        const int n_elements = n_embd*n_mem;

        // 创建存储记忆键的张量
        model.memory_k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        // 创建存储记忆值的张量
        model.memory_v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        // 设置记忆键张量的名称
        ggml_set_name(model.memory_k, "model/memory_k");
        // 设置记忆值张量的名称
        ggml_set_name(model.memory_v, "model/memory_v");

        // 计算内存大小
        const size_t memory_size = ggml_nbytes(model.memory_k) + ggml_nbytes(model.memory_v);

        // 打印内存大小和记忆单元数量
        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);

        // 创建后端缓冲区（可以是主机内存或设备内存）
        ggml_backend_t backend_kv = params.n_gpu_layers >= hparams.n_layer/2 ? backend_gpu : backend_cpu;
        printf("%s: backend_kv = %s\n", __func__, ggml_backend_name(backend_kv));
        model.buffer_kv = ggml_backend_alloc_buffer(backend_kv, memory_size + 512*2);

        // 将张量分配到后端缓冲区中
        {
            ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer_kv);

            // 这将更新张量中的指针，指向缓冲区中的正确位置
            // 这是必要的，因为 ggml_context 是 .no_alloc == true
            // 请注意，根据后端，缓冲区实际上可以是设备缓冲区
            ggml_allocr_alloc(alloc, model.memory_k);
            ggml_allocr_alloc(alloc, model.memory_v);

            ggml_allocr_free(alloc);
        }
    }

    // 加载权重
#ifdef GGML_USE_METAL
                || ggml_backend_is_metal(backend)
#endif
                ) {
                // 如果使用 Metal 后端，或者当前后端是 Metal，则直接读取到张量中
                fin.read(reinterpret_cast<char *>(tensor->data), ggml_nbytes(tensor));
            } else {
                // 否则，先读取到临时缓冲区，然后再复制到设备内存中
                read_buf.resize(ggml_nbytes(tensor));
                fin.read(read_buf.data(), ggml_nbytes(tensor));
                ggml_backend_tensor_set(tensor, read_buf.data(), 0, ggml_nbytes(tensor));
            }

            // GPT-2 模型将 WTE 张量共享为 LM head
            if (name == "model/wte" && has_lm_head == false) {
                ggml_allocr_alloc(backend_buffers.find(tensor_backends["model/lm_head"])->second.get(), model.lm_head);
                //printf("%s: [%5.5s] %s (copied)\n", __func__, ggml_backend_name(tensor_backends["model/lm_head"]), "model/lm_head");
                ggml_backend_tensor_copy(tensor, model.lm_head);
                total_size += ggml_nbytes(model.lm_head);
            }

            if (name == "model/lm_head") {
                has_lm_head = true;
            }

            total_size += ggml_nbytes(tensor);
        }
        printf("%s: model size  = %8.2f MB\n", __func__, total_size/1024.0/1024.0);
    }

    fin.close();

    // 分配输入张量
    {
        // 创建一个一维张量，用于存储模型的embd（嵌入）数据，数据类型为32位整型，长度为模型的上下文长度
        model.embd = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, model.hparams.n_ctx);
        // 创建一个一维张量，用于存储模型的position（位置）数据，数据类型为32位整型，长度为模型的上下文长度
        model.position = ggml_new_tensor_1d(ctx, GGML_TYPE_I32, model.hparams.n_ctx);

        // 为embd张量和position张量设置名称
        ggml_set_name(model.embd, "in/embd");
        ggml_set_name(model.position, "in/position");

        // 将输入张量添加到CPU后端
        size_t input_size = ggml_nbytes(model.embd) + ggml_nbytes(model.position);

        // FIXME: 在调度实现后使用CPU后端
        ggml_backend_t backend_input = params.n_gpu_layers >= model.hparams.n_layer ? backend_gpu : backend_cpu;
        // 为模型的输入数据分配后端缓冲区
        model.buffer_input = ggml_backend_alloc_buffer(backend_input, input_size + 512*3);
        // 打印后端信息
        printf("%s: backend_in = %s (%zu bytes)\n", __func__, ggml_backend_name(backend_input), input_size);

        // 将张量分配到后端缓冲区
        ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer_input);
        ggml_allocr_alloc(alloc, model.embd);
        ggml_allocr_alloc(alloc, model.position);
        ggml_allocr_free(alloc);
    }

    // 返回true
    return true;
// 构建计算图
struct ggml_cgraph * gpt2_graph(
        const gpt2_model & model,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp) {
    const int N = embd_inp.size();  // 获取输入向量的大小

    const auto & hparams = model.hparams;  // 获取模型的超参数

    const int n_embd  = hparams.n_embd;  // 获取嵌入层的维度
    const int n_layer = hparams.n_layer;  // 获取层数
    const int n_ctx   = hparams.n_ctx;  // 获取上下文的大小
    const int n_head  = hparams.n_head;  // 获取头的数量

    // 由于我们使用了 ggml-alloc，这个缓冲区只需要足够的空间来容纳 ggml_tensor 和 ggml_cgraph 结构，但不需要张量数据
    static size_t buf_size = ggml_tensor_overhead()*GPT2_MAX_NODES + ggml_graph_overhead_custom(GPT2_MAX_NODES, false);  // 计算缓冲区大小
    static std::vector<uint8_t> buf(buf_size);  // 创建缓冲区

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,  // 内存大小
        /*.mem_buffer =*/ buf.data(),  // 内存缓冲区
        /*.no_alloc   =*/ true, // 张量将在后面由 ggml_allocr_alloc_graph() 分配，不需要立即分配
    };

    struct ggml_context * ctx0 = ggml_init(params);  // 初始化上下文

    struct ggml_cgraph  * gf = ggml_new_graph_custom(ctx0, GPT2_MAX_NODES, false);  // 创建自定义计算图

    struct ggml_tensor * embd = ggml_view_1d(ctx0, model.embd, N, 0);  // 创建嵌入层张量视图

    // TODO: 避免在只测量内存使用情况时写入张量
    // 不是关键的，只是一个小的优化
    //if (!ggml_allocr_is_measure(allocr)) {
        //ggml_backend_tensor_set(embd, embd_inp.data(), 0, N*ggml_element_size(embd));
        ggml_backend_tensor_set(model.embd, embd_inp.data(), 0, N*ggml_element_size(embd)); // FIXME: 在这里不能使用视图，因为它还没有初始化（缓冲区未设置），但我们应该
    //}
    //memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    struct ggml_tensor * position = ggml_view_1d(ctx0, model.position, N, 0);  // 创建位置张量视图
    //if (!ggml_allocr_is_measure(allocr)) {
        // 如果分配器不是度量器，则执行以下操作
        for (int i = 0; i < N; ++i) {
            // 循环遍历N次
            int32_t v = n_past + i;
            // 计算v的值
            ggml_backend_tensor_set(model.position, &v, i*sizeof(int32_t), sizeof(v)); // FIXME: same
            // 将v的值设置到model.position中
            //((int32_t *) position->data)[i] = n_past + i;
        }
    //}

    const float KQ_scale = 1.0f/sqrtf(float(model.hparams.n_embd)/model.hparams.n_head);
    // 计算KQ_scale的值

    // wte + wpe
    // 计算inpL的值
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));
    ggml_set_name(inpL, "inpL");
    ggml_set_name(inpL->src[0], "wte");
    ggml_set_name(inpL->src[1], "wpe");

    }

    // norm
    {
        // [ 768, N]
        // 对inpL进行归一化处理
        inpL = ggml_norm(ctx0, inpL, hparams.eps);
        ggml_format_name(inpL, "out_norm");

        // inpL = ln_f_g*inpL + ln_f_b
        // [ 768, N]
        // 对inpL进行线性变换
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    inpL,
                    model.ln_f_g),
                model.ln_f_b);
        ggml_format_name(inpL, "out_ln_f_b");
        ggml_format_name(inpL->src[0], "out_ln_f_g");
    }

    // inpL = WTE * inpL
    // [ 768, 50257] - model.lm_head
    // [ 768, N]     - inpL
    // 计算inpL和model.lm_head的矩阵乘法
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);
    ggml_format_name(inpL, "out_lm_head");

    // logits -> probs
    //inpL = ggml_soft_max(ctx0, inpL);

    ggml_build_forward_expand(gf, inpL);
    // 构建前向传播

    ggml_free(ctx0);
    // 释放上下文

    return gf;
    // 返回gf
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
        ggml_backend_sched_t sched,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w) {
    const int N = embd_inp.size();

    const auto & hparams = model.hparams;

    const int n_vocab = hparams.n_vocab;

    // 创建 GPT-2 图
    struct ggml_cgraph * gf = gpt2_graph(model, n_past, embd_inp);

    // 运行计算
    ggml_backend_sched_graph_compute(sched, gf);

    // 如果 n_past 是 100 的倍数，则打印图形和转储图形
    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // 在这种情况下，输出张量是图中的最后一个张量
    struct ggml_tensor * inpL = gf->nodes[gf->n_nodes - 1];

    // 调整大小并获取张量数据，仅返回最后一个标记的结果
    embd_w.resize(n_vocab);
    ggml_backend_tensor_get(inpL, embd_w.data(), (n_vocab*(N-1))*sizeof(float), sizeof(float)*n_vocab);

    return true;
}

int main(int argc, char ** argv) {
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;
    params.model = "models/gpt-2-117M/ggml-model.bin";

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        params.seed = time(NULL);
    }

    printf("%s: seed = %d\n", __func__, params.seed);

    std::mt19937 rng(params.seed);
    if (params.prompt.empty()) {
        params.prompt = gpt_random_prompt(rng);
    }

    int64_t t_load_us = 0;

    gpt_vocab vocab;
    gpt2_model model;
    // 加载模型
    {
        // 记录开始加载模型的时间
        const int64_t t_start_us = ggml_time_us();

        // 如果无法从指定路径加载模型，则输出错误信息并返回 1
        if (!gpt2_model_load(params.model, model, vocab, params)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        // 计算加载模型所花费的时间
        t_load_us = ggml_time_us() - t_start_us;

        // 测试 GPT 分词器
        test_gpt_tokenizer(vocab, params.token_test);
    }

    // 创建后端调度器
    // 调度器负责分配计算缓冲区并在不同后端之间调度计算
    ggml_backend_sched_t sched;
    {
        // 初始化调度器
        sched = ggml_backend_sched_new(model.backends.data(), model.backends.size());

        // 创建内存使用估计的最坏情况图
        int n_tokens = std::min(model.hparams.n_ctx, params.n_batch);
        int n_past = model.hparams.n_ctx - n_tokens;
        struct ggml_cgraph * gf = gpt2_graph(model, n_past, std::vector<gpt_vocab::id>(n_tokens, 0));

        // 初始化内存使用估计
        ggml_backend_sched_init_measure(sched, gf);

        // 计算所需内存
        size_t mem_size = 0;
        for (size_t i = 0; i < model.backends.size(); i++) {
            ggml_backend_buffer_t buf = ggml_backend_sched_get_buffer(sched, model.backends[i]);
            size_t size = ggml_backend_buffer_get_size(buf);
            if (size > 0) {
                mem_size += size;
                printf("%s: %8s compute buffer size = %8.2f MB\n", __func__, ggml_backend_name(model.backends[i]), size/1024.0/1024.0);
                //printf("%s: %8s compute buffer size = %zu bytes\n", __func__, ggml_backend_name(model.backends[i]), size);
            }
        }

        printf("%s: total compute buffer size: %.2f MB\n", __func__, mem_size/1024.0/1024.0);
    }

    // 初始化变量
    int n_past = 0;

    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    std::vector<float> logits;

    // 对提示进行分词
    // 使用词汇表和输入的提示文本进行分词，得到词汇表对应的 ID 列表
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    // 确定要预测的标记数量，取输入标记数量和模型上下文长度的较小值
    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    // 打印提示文本和其中的标记数量
    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
    printf("%s: number of tokens in prompt = %zu, first 8 tokens: ", __func__, embd_inp.size());
    for (int i = 0; i < std::min(8, (int) embd_inp.size()); i++) {
        printf("%d ", embd_inp[i]);
    }
    printf("\n\n");

    // 逐个提交输入的提示标记
    // 这样做可以减少推理过程中的内存使用，但会稍微降低速度
    std::vector<gpt_vocab::id> embd;
    for (size_t i = embd.size(); i < embd_inp.size() + params.n_predict; i++) {
        // 从当前嵌入大小开始循环直到嵌入输入大小加上预测数量
        // 预测
        if (embd.size() > 0) {
            // 如果嵌入大小大于0
            const int64_t t_start_us = ggml_time_us();
            // 记录开始时间

            if (!gpt2_eval(model, sched, n_past, embd, logits)) {
                // 如果无法进行预测
                printf("Failed to predict\n");
                return 1;
            }

            t_predict_us += ggml_time_us() - t_start_us;
            // 计算预测时间
        }

        n_past += embd.size();
        // 更新过去的数量
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
                // 通过top-k和top-p采样得到标记id

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
                if (int32_t(embd.size()) >= params.n_batch) {
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
        // 计算主函数结束时间
        const int64_t t_main_end_us = ggml_time_us();

        // 打印加载时间
        printf("\n\n");
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        // 打印采样时间
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        // 打印预测时间和每个标记的平均时间
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/n_past);
        // 打印总时间
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    // 释放模型上下文
    ggml_free(model.ctx);

    // 释放后端调度器
    ggml_backend_sched_free(sched);
    // 释放键值缓冲区
    ggml_backend_buffer_free(model.buffer_kv);
    // 释放所有写入缓冲区
    for (auto & buf : model.buffers_w) {
        ggml_backend_buffer_free(buf);
    }
    // 释放所有后端
    for (auto backend : model.backends) {
        ggml_backend_free(backend);
    }

    // 返回成功状态
    return 0;
}
# 闭合前面的函数定义
```