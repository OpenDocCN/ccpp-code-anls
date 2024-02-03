# `ggml\examples\gpt-2\main-batched.cpp`

```cpp
#include "ggml/ggml.h"
#include "ggml/ggml-alloc.h"
#include "ggml/ggml-backend.h"
// 包含 GGML 库的头文件

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif
// 如果使用 CUDA 加速，则包含 CUDA 相关头文件

#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif
// 如果使用 Metal 加速，则包含 Metal 相关头文件

#include "common.h"
#include "common-ggml.h"
// 包含通用的头文件

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <set>
#include <string>
#include <vector>
// 包含 C++ 标准库的头文件

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif
// 如果是在 MSVC 编译器下，则禁止特定警告

#define GPT2_MAX_NODES 4096
// 定义 GPT2 最大节点数为 4096

static void ggml_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);
    fflush(stderr);
}
// 定义默认的日志回调函数，将日志输出到标准错误流

typedef int32_t gpt2_pos;
typedef int32_t gpt2_seq_id;
// 定义 GPT2 中使用的位置和序列 ID 类型为 32 位整数

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
// 定义默认的 GPT2 参数结构体，包括词汇量、上下文长度、嵌入维度、头数、层数、特征类型和 epsilon

struct gpt2_layer {
    // normalization
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;
    // 定义 GPT2 层的归一化参数和张量

    // attention
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;
    // 定义 GPT2 层的注意力参数和张量

    // mlp
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
    // 定义 GPT2 层的多层感知机参数和张量
};

struct gpt2_kv_cell {
    gpt2_pos pos   = -1;
    gpt2_pos delta = 0;

    std::set<gpt2_seq_id> seq_id;

    bool has_seq_id(const gpt2_seq_id & id) const {
        return seq_id.find(id) != seq_id.end();
    }
};
// 定义 GPT2 键值单元结构体，包括位置、增量、序列 ID 集合和判断是否存在特定序列 ID 的方法

struct gpt2_kv_cache {
    // key + value memory
    struct ggml_tensor * k;
    struct ggml_tensor * v;
    //

    uint32_t head = 0;
    uint32_t size = 0;

    // computed before each graph build
    uint32_t n = 0;
};
// 定义 GPT2 键值缓存结构体，包括键值张量、头数、大小和计算的数量
    # 创建一个名为 cells 的 vector 对象，用于存储 gpt2_kv_cell 类型的数据
    std::vector<gpt2_kv_cell> cells;
    
    # 创建一个名为 buffer 的 ggml_backend_buffer_t 类型的对象
    ggml_backend_buffer_t buffer;
// 结构体定义，包含了 GPT2 模型的各种参数和组件
struct gpt2_model {
    gpt2_hparams hparams;  // GPT2 模型的超参数

    // 归一化
    struct ggml_tensor * ln_f_g;  // Layer Normalization 的缩放参数
    struct ggml_tensor * ln_f_b;  // Layer Normalization 的偏置参数

    struct ggml_tensor * wte;     // 位置嵌入
    struct ggml_tensor * wpe;     // 词嵌入
    struct ggml_tensor * lm_head; // 语言模型头部

    std::vector<gpt2_layer> layers;  // GPT2 模型的层

    gpt2_kv_cache kv_cache;  // GPT2 模型的键值缓存

    struct ggml_context * ctx;  // 上下文

    ggml_backend_t backend = NULL;  // 后端

    ggml_backend_buffer_t buffer_w;  // 后端缓冲区

    std::map<std::string, struct ggml_tensor *> tensors;  // 字符串到张量的映射
};

// gpt2_decode 的输入数据
// 一个 gpt2_batch 对象可以包含一个或多个序列的输入数据
// 提供的数组（例如 token、embd、pos 等）必须具有 n_tokens 大小
//
// - token  : 输入的标记 ID（当 embd 为 NULL 时使用）
// - embd   : 标记嵌入（大小为 n_embd 的浮点向量）（当 token 为 NULL 时使用）
// - pos    : 序列中各标记的位置
// - seq_id : 各标记所属的序列
// - logits : 如果为零，则不会输出相应标记的 logits
//
struct gpt2_batch {
    int32_t n_tokens = -1;  // 标记数量，默认为 -1

    gpt_vocab::id  * token  = {};  // 标记数组
    float          * embd   = {};  // 嵌入数组
    gpt2_pos       * pos    = {};  // 位置数组
    gpt2_seq_id    * seq_id = {};  // 序列 ID 数组
    int8_t         * logits = {};  // logits 数组
};

// 从文件加载模型的权重
bool gpt2_model_load(const std::string & fname, gpt2_model & model, gpt_vocab & vocab, int n_ctx, int n_gpu_layers) {
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());  // 打印加载模型的信息

    auto fin = std::ifstream(fname, std::ios::binary);  // 以二进制模式打开文件流
    if (!fin) {
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());  // 如果打开文件失败，则打印错误信息
        return false;  // 返回 false
    }

    // 验证魔数
    {
        uint32_t magic;  // 魔数
        fin.read((char *) &magic, sizeof(magic));  // 读取魔数
        if (magic != GGML_FILE_MAGIC) {  // 如果魔数不匹配
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());  // 打印错误信息
            return false;  // 返回 false
        }
    // 加载超参数
    {
        // 获取模型的超参数引用
        auto & hparams = model.hparams;

        // 从文件中读取并存储词汇量
        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        // 从文件中读取并存储上下文大小
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        // 从文件中读取并存储嵌入维度
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        // 从文件中读取并存储头数
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        // 从文件中读取并存储层数
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        // 从文件中读取并存储特征类型
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        // 计算特征类型的量化版本
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        // 打印超参数信息
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        // 计算特征类型的量化版本并更新超参数的特征类型
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // 加载词汇表
    {
        // 从文件中读取并存储词汇量
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        // 如果读取的词汇量与模型的词汇量不匹配，则输出错误信息并返回false
        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        // 读取词汇表中的单词并将其映射到对应的ID
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
    # 将模型的文件类型转换为对应的 GGML 类型
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    # 如果转换后的类型为 GGML_TYPE_COUNT，则说明文件类型无效，输出错误信息并返回 false
    if (wtype == GGML_TYPE_COUNT) {
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        return false;
    }

    # 获取模型的上下文
    auto & ctx = model.ctx;

    # 初始化缓冲区大小为 0
    size_t buffer_size = 0;
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
    
        // 从超参数中获取相关参数值
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;
        const int n_vocab = hparams.n_vocab;
    
        // 计算并累加 ln_f_g 和 ln_f_b 的内存大小
        buffer_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_g
        buffer_size += ggml_row_size(GGML_TYPE_F32, n_embd); // ln_f_b
    
        // 计算并累加 wte 和 wpe 的内存大小
        buffer_size += ggml_row_size(wtype,         n_vocab*n_embd); // wte
        buffer_size += ggml_row_size(GGML_TYPE_F32,   n_ctx*n_embd); // wpe
        buffer_size += ggml_row_size(wtype,         n_vocab*n_embd); // lm_head
    
        // 计算并累加 ln_1_g 和 ln_1_b 的内存大小
        buffer_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_g
        buffer_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_1_b
    
        // 计算并累加 ln_2_g 和 ln_2_b 的内存大小
        buffer_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_g
        buffer_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd)); // ln_2_b
    
        // 计算并累加 c_attn_attn_w 和 c_attn_attn_b 的内存大小
        buffer_size += n_layer*(ggml_row_size(wtype,         3*n_embd*n_embd)); // c_attn_attn_w
        buffer_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 3*n_embd));        // c_attn_attn_b
    
        // 计算并累加 c_attn_proj_w 和 c_attn_proj_b 的内存大小
        buffer_size += n_layer*(ggml_row_size(wtype,         n_embd*n_embd));   // c_attn_proj_w
        buffer_size += n_layer*(ggml_row_size(GGML_TYPE_F32, n_embd));          // c_attn_proj_b
    
        // 计算并累加 c_mlp_fc_w 和 c_mlp_fc_b 的内存大小
        buffer_size += n_layer*(ggml_row_size(wtype,         4*n_embd*n_embd)); // c_mlp_fc_w
        buffer_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 4*n_embd));        // c_mlp_fc_b
    
        // 计算并累加 c_mlp_proj_w 和 c_mlp_proj_b 的内存大小
        buffer_size += n_layer*(ggml_row_size(wtype,         4*n_embd*n_embd)); // c_mlp_proj_w
        buffer_size += n_layer*(ggml_row_size(GGML_TYPE_F32, 4*n_embd));        // c_mlp_proj_b
    
        // 计算并累加 alignment overhead 的内存大小
        buffer_size += (6 + 12*n_layer)*128; // alignment overhead
    
        // 打印 ggml tensor 的大小
        printf("%s: ggml tensor size    = %d bytes\n", __func__, (int) sizeof(ggml_tensor));
        // 打印 backend buffer 的大小（以 MB 为单位）
        printf("%s: backend buffer size = %6.2f MB\n", __func__, buffer_size/(1024.0*1024.0);
    }
    
    // 创建 ggml 上下文
    {
        // 计算需要的张量数量
        size_t n_tensors = 2 + 6 + 12*model.hparams.n_layer;
        // 初始化参数结构体
        struct ggml_init_params params = {
            // 内存大小为张量开销乘以张量数量
            /*.mem_size   =*/ ggml_tensor_overhead() * n_tensors,
            // 内存缓冲区为空
            /*.mem_buffer =*/ NULL,
            // 不进行内存分配
            /*.no_alloc   =*/ true,
        };

        // 使用初始化参数初始化上下文
        model.ctx = ggml_init(params);
        // 如果初始化失败，输出错误信息并返回假
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 初始化后端
#ifdef GGML_USE_CUBLAS
    // 如果使用 CUDA 后端，并且有 GPU 层，则初始化 CUDA 后端
    if (n_gpu_layers > 0) {
        // 输出信息，使用 CUDA 后端
        fprintf(stderr, "%s: using CUDA backend\n", __func__);
        // 初始化 CUDA 后端
        model.backend = ggml_backend_cuda_init(0);
        // 如果初始化失败，则输出错误信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#endif

#ifdef GGML_USE_METAL
    // 如果使用 Metal 后端，并且有 GPU 层，则初始化 Metal 后端
    if (n_gpu_layers > 0) {
        // 输出信息，使用 Metal 后端
        fprintf(stderr, "%s: using Metal backend\n", __func__);
        // 设置 Metal 后端日志回调函数
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        // 初始化 Metal 后端
        model.backend = ggml_backend_metal_init();
        // 如果初始化失败，则输出错误信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_metal_init() failed\n", __func__);
        }
    }
#endif

    // 如果没有选择任何后端，则回退到 CPU 后端
    if (!model.backend) {
        // 输出信息，使用 CPU 后端
        fprintf(stderr, "%s: using CPU backend\n", __func__);
        // 初始化 CPU 后端
        model.backend = ggml_backend_cpu_init();
    }

    // 如果初始化后端失败，则输出错误信息并返回 false
    if (!model.backend) {
        fprintf(stderr, "%s: ggml_backend_cpu_init() failed\n", __func__);
        return false;
    }

    // 分配权重缓冲区
    model.buffer_w = ggml_backend_alloc_buffer(model.backend, buffer_size);

    // 准备权重的内存

    // 用用户提供的上下文覆盖默认的训练上下文
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

        // 计算记忆单元的数量
        const int n_mem      = n_layer*n_ctx;
        // 计算元素的数量
        const int n_elements = n_embd*n_mem;

        // 为键值缓存分配内存
        model.kv_cache.k = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);
        model.kv_cache.v = ggml_new_tensor_1d(ctx, GGML_TYPE_F32, n_elements);

        // 初始化键值缓存的头和大小
        model.kv_cache.head      = 0;
        model.kv_cache.size      = n_ctx;

        // 调整键值缓存的单元格大小
        model.kv_cache.cells.resize(n_ctx);

        // 计算内存大小
        const size_t memory_size = ggml_nbytes(model.kv_cache.k) + ggml_nbytes(model.kv_cache.v);

        // 打印内存大小和记忆单元数量
        printf("%s: memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);

        // 创建后端缓冲区（可以是主机或设备内存）
        model.kv_cache.buffer = ggml_backend_alloc_buffer(model.backend, memory_size + 256);

        // 将张量分配到后端缓冲区中
        {
            ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.kv_cache.buffer);

            // 这将更新张量中的指针，指向缓冲区中的正确位置
            // 这是必要的，因为 ggml_context 是 .no_alloc == true
            // 请注意，缓冲区实际上可以是设备缓冲区，这取决于后端
            ggml_allocr_alloc(alloc, model.kv_cache.k);
            ggml_allocr_alloc(alloc, model.kv_cache.v);

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

        ggml_allocr_free(alloc);
        printf("%s: model size  = %8.2f MB\n", __func__, total_size/1024.0/1024.0);
    }

    fin.close();

    return true;
}

// 构建计算图
struct ggml_cgraph * gpt2_graph(
        const  gpt2_model  & model,
        struct ggml_allocr * allocr,
        const  gpt2_batch  & batch) {
    const auto & hparams = model.hparams;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;

    const auto & kv_cache = model.kv_cache;

    const int32_t n_tokens = batch.n_tokens;
    const int32_t n_kv     = ggml_allocr_is_measure(allocr) ? n_ctx            : kv_cache.n;
    const int32_t kv_head  = ggml_allocr_is_measure(allocr) ? n_ctx - n_tokens : kv_cache.head;
    // 由于我们使用了 ggml-alloc，所以这个缓冲区只需要足够容纳 ggml_tensor 和 ggml_cgraph 结构，但不需要张量数据
    static size_t buf_size = ggml_tensor_overhead()*GPT2_MAX_NODES + ggml_graph_overhead_custom(GPT2_MAX_NODES, false);
    static std::vector<uint8_t> buf(buf_size);

    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,  // 初始化参数：内存大小为 buf_size
        /*.mem_buffer =*/ buf.data(),  // 初始化参数：内存缓冲区为 buf 的数据
        /*.no_alloc   =*/ true, // 初始化参数：不分配内存，张量将在之后由 ggml_allocr_alloc_graph() 分配
    };

    struct ggml_context * ctx0 = ggml_init(params);  // 初始化 ggml 上下文

    struct ggml_cgraph  * gf = ggml_new_graph_custom(ctx0, GPT2_MAX_NODES, false);  // 创建自定义图形

    struct ggml_tensor * inpL;  // 输入张量
    if (batch.token) {  // 如果批处理中有 token
        struct ggml_tensor * inp_tokens = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);  // 创建一维整型张量 inp_tokens
        ggml_allocr_alloc(allocr, inp_tokens);  // 分配内存
        if (!ggml_allocr_is_measure(allocr)) {  // 如果不是测量模式
            ggml_backend_tensor_set(inp_tokens, batch.token, 0, n_tokens*ggml_element_size(inp_tokens));  // 设置张量数据
        }

        struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, n_tokens);  // 创建一维整型张量 position
        ggml_allocr_alloc(allocr, position);  // 分配内存
        if (!ggml_allocr_is_measure(allocr)) {  // 如果不是测量模式
            for (int i = 0; i < n_tokens; ++i) {
                int32_t v = batch.pos[i];
                ggml_backend_tensor_set(position, &v, i*sizeof(int32_t), sizeof(v));  // 设置张量数据
            }
        }

        // wte + wpe
        inpL =
            ggml_add(ctx0,
                    ggml_get_rows(ctx0, model.wte, inp_tokens),  // 获取 wte 的行
                    ggml_get_rows(ctx0, model.wpe, position));  // 获取 wpe 的行
    } else {  // 如果批处理中没有 token
        GGML_ASSERT(batch.embd);  // 断言批处理中有 embd

        inpL = ggml_new_tensor_2d(ctx0, GGML_TYPE_F32, n_embd, n_tokens);  // 创建二维浮点型张量 inpL

        ggml_allocr_alloc(allocr, inpL);  // 分配内存
        if (!ggml_allocr_is_measure(allocr)) {  // 如果不是测量模式
            ggml_backend_tensor_set(inpL, batch.embd, 0, n_tokens * n_embd * ggml_element_size(inpL));  // 设置张量数据
        }
    }
    // 创建一个 3 维的浮点型张量 KQ_mask，大小为 n_kv * n_tokens * 1
    struct ggml_tensor * KQ_mask = ggml_new_tensor_3d(ctx0, GGML_TYPE_F32, n_kv, n_tokens, 1);
    // 为张量设置名称
    ggml_set_name(KQ_mask, "KQ_mask");
    // 分配内存
    ggml_allocr_alloc(allocr, KQ_mask);
    // 如果不是测量模式
    if (!ggml_allocr_is_measure(allocr)) {
        // 创建一个大小为 n_kv * n_tokens 的浮点型数组 data_buf
        std::vector<float> data_buf(n_kv*n_tokens);
        // 设置负无穷大的值
        const float neg_inf_v = -INFINITY;

        // 遍历头部
        for (int h = 0; h < 1; ++h) {
            int h_offset = h*(n_kv*n_tokens);
            // 遍历 tokens
            for (int j = 0; j < n_tokens; ++j) {
                // 获取位置和序列 ID
                const gpt2_pos    pos    = batch.pos[j];
                const gpt2_seq_id seq_id = batch.seq_id[j];

                // 遍历键值对
                for (int i = 0; i < n_kv; ++i) {
                    // 如果缓存中不包含序列 ID 或者缓存中的位置大于当前位置
                    if (!kv_cache.cells[i].has_seq_id(seq_id) || kv_cache.cells[i].pos > pos) {
                        // 将对应位置的值设置为负无穷大
                        data_buf[h_offset + j*n_kv + i] = neg_inf_v;
                    }
                }
            }
        }

        // 将数据数组设置到张量中
        ggml_backend_tensor_set(KQ_mask, data_buf.data(), 0, data_buf.size() * sizeof(float));
    }

    // 归一化处理
    {
        // 对输入张量 inpL 进行归一化处理，使用参数 hparams.eps
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        // 对 inpL 进行线性变换
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    inpL,
                    model.ln_f_g),
                model.ln_f_b);
    }

    // 矩阵相乘
    // 将模型的 lm_head 与输入张量 inpL 进行矩阵相乘
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // 计算 logits -> probs
    //inpL = ggml_soft_max(ctx0, inpL);

    // 构建前向传播
    ggml_build_forward_expand(gf, inpL);

    // 释放上下文
    ggml_free(ctx0);

    // 返回结果
    return gf;
// 将指定序列ID的数据从源位置复制到目标位置
static void gpt2_kv_cache_seq_cp(
        struct gpt2_kv_cache & cache,
                 gpt2_seq_id   seq_id_src,
                 gpt2_seq_id   seq_id_dst,
                    gpt2_pos   p0,
                    gpt2_pos   p1) {
    // 如果起始位置小于0，则将其设置为0
    if (p0 < 0) p0 = 0;
    // 如果结束位置小于0，则将其设置为gpt2_pos类型的最大值
    if (p1 < 0) p1 = std::numeric_limits<gpt2_pos>::max();

    // 遍历缓存中的数据
    for (uint32_t i = 0; i < cache.size; ++i) {
        // 如果缓存中的数据包含源序列ID，并且位置在指定范围内，则将目标序列ID插入到数据中
        if (cache.cells[i].has_seq_id(seq_id_src) && cache.cells[i].pos >= p0 && cache.cells[i].pos < p1) {
            cache.cells[i].seq_id.insert(seq_id_dst);
        }
    }
}

// 初始化一个GPT-2批次数据结构
struct gpt2_batch gpt2_batch_init(int32_t n_tokens, int32_t embd) {
    gpt2_batch batch;

    // 如果embd不为0，则分配n_tokens * embd个float类型的内存空间
    if (embd) {
        batch.embd = (float *) malloc(sizeof(float) * n_tokens * embd);
    } else {
        // 否则分配n_tokens个gpt_vocab::id类型的内存空间
        batch.token = (gpt_vocab::id *) malloc(sizeof(gpt_vocab::id) * n_tokens);
    }

    // 分配n_tokens个gpt2_pos类型的内存空间
    batch.pos    = (gpt2_pos *)    malloc(sizeof(gpt2_pos)    * n_tokens);
    // 分配n_tokens个gpt2_seq_id类型的内存空间
    batch.seq_id = (gpt2_seq_id *) malloc(sizeof(gpt2_seq_id) * n_tokens);
    // 分配n_tokens个int8_t类型的内存空间
    batch.logits = (int8_t *)      malloc(sizeof(int8_t)      * n_tokens);

    return batch;
}

// 释放GPT-2批次数据结构占用的内存空间
void gpt2_batch_free(struct gpt2_batch batch) {
    // 释放token指向的内存空间
    if (batch.token)  free(batch.token);
    // 释放embd指向的内存空间
    if (batch.embd)   free(batch.embd);
    // 释放pos指向的内存空间
    if (batch.pos)    free(batch.pos);
    // 释放seq_id指向的内存空间
    if (batch.seq_id) free(batch.seq_id);
    // 释放logits指向的内存空间
    if (batch.logits) free(batch.logits);
}

// 对GPT-2模型进行解码，返回logits值
// 正数返回值并不意味着致命错误，而是警告
//   0 - 成功
// < 0 - 错误
int gpt2_decode(
        struct gpt2_model  & model,
        struct ggml_allocr * allocr,
        struct gpt2_batch    batch,
        int                  n_threads,
        std::vector<float> & logits) {
    const int32_t n_tokens = batch.n_tokens;
    const auto &  hparams  = model.hparams;
    const int     n_vocab  = hparams.n_vocab;

    // 如果n_tokens为0，则打印错误信息并返回-1
    if (n_tokens == 0) {
        printf("%s: n_tokens == 0", __func__);
        return -1;
    }

    // 断言：如果token为空且embd不为空，或者token不为空且embd为空，则抛出异常
    GGML_ASSERT((!batch.token && batch.embd) || (batch.token && !batch.embd));

    // 获取模型的键值缓存
    auto & cache = model.kv_cache;
    // 遍历 n_tokens 次，将 batch.pos[i] 赋值给 cache.cells[cache.head + i].pos
    // 将 batch.seq_id[i] 插入到 cache.cells[cache.head + i].seq_id 中
    for (int i = 0; i < n_tokens; i++) {
        cache.cells[cache.head + i].pos = batch.pos[i];
        cache.cells[cache.head + i].seq_id.insert(batch.seq_id[i]);
    }

    // 更新 cache.n 的值为 cache.head + n_tokens
    cache.n = cache.head + n_tokens;

    // 重置分配器，释放上一次推断期间分配的所有内存
    ggml_allocr_reset(allocr);

    // 使用模型、分配器和批次数据创建计算图
    struct ggml_cgraph * gf = gpt2_graph(model, allocr, batch);

    // 分配张量
    ggml_allocr_alloc_graph(allocr, gf);

    // 运行计算
    // 如果模型的后端是 CPU，则设置线程数为 n_threads
    if (ggml_backend_is_cpu(model.backend)) {
        ggml_backend_cpu_set_n_threads(model.backend, n_threads);
    }
#ifdef GGML_USE_METAL
    // 如果使用 Metal 后端，则设置线程数
    if (ggml_backend_is_metal(model.backend)) {
        ggml_backend_metal_set_n_cb(model.backend, n_threads);
    }
#endif
    // 计算图的前向传播
    ggml_backend_graph_compute(model.backend, gf);

    // 每100次迭代打印图的结构和将图导出为 DOT 格式的文件
    //if (n_past%100 == 0) {
    //    ggml_graph_print   (&gf);
    //    ggml_graph_dump_dot(&gf, NULL, "gpt-2.dot");
    //}

    // 在这种情况下，输出张量是图中的最后一个张量
    struct ggml_tensor * inpL = gf->nodes[gf->n_nodes - 1];

    if (batch.logits) {
        // 返回所有标记的 logits
        logits.resize(n_vocab*n_tokens);
        for (int32_t i = 0; i < n_tokens; i++) {
            if (batch.logits[i] == 0) {
                continue;
            }
            ggml_backend_tensor_get(inpL, logits.data() + n_vocab*i, n_vocab*i*sizeof(float), sizeof(float)*n_vocab);
        }
    } else {
        // 仅返回最后一个标记的结果
        logits.resize(n_vocab);
        ggml_backend_tensor_get(inpL, logits.data(), (n_vocab*(n_tokens-1))*sizeof(float), sizeof(float)*n_vocab);
    }

    // 更新 kv 环形缓冲区
    cache.head += n_tokens;

    // 确保 kv 缓存头指向有效索引
    if (cache.head >= cache.size) {
        printf("%s: cache.head >= cache.size\n", __func__);
        return -2;
    }

    return 0;
}

int main(int argc, char ** argv) {
    // 初始化时间
    ggml_time_init();

    const int64_t t_main_start_us = ggml_time_us();

    gpt_params params;

    if (gpt_params_parse(argc, argv, params) == false) {
        return 1;
    }

    if (params.seed < 0) {
        // 如果种子值小于0，则设置为当前时间
        params.seed = time(NULL);
    }

    printf("%s: seed = %d\n", __func__, params.seed);

    std::mt19937 rng(params.seed);
    if (params.prompt.empty()) {
        // 如果提示为空，则生成随机提示
        params.prompt = gpt_random_prompt(rng);
    }

    int64_t t_load_us = 0;

    gpt_vocab vocab;
    gpt2_model model;

    // 加载模型
    {
        // 记录开始时间
        const int64_t t_start_us = ggml_time_us();

        // 加载模型和词汇表，如果失败则打印错误信息并返回
        if (!gpt2_model_load(params.model, model, vocab, params.n_ctx, params.n_gpu_layers)) {
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());
            return 1;
        }

        // 计算加载模型所花费的时间
        t_load_us = ggml_time_us() - t_start_us;

        // 测试 GPT 分词器
        test_gpt_tokenizer(vocab, params.token_test);
    }

    // 对提示进行分词
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);

    // 在评估模型时保持此缓冲区有效
    ggml_backend_buffer_t buf_compute;

    // 并行处理的数量
    const int n_parallel = params.n_parallel;
    // 批处理的最大数量
    const int n_batch_max = std::max(embd_inp.size(), (size_t)n_parallel);

    // 创建一个 gpt2_batch 对象
    // 我们使用这个对象来提交令牌数据进行解码
    gpt2_batch batch = gpt2_batch_init(n_batch_max, 0);

    // 准备所需的内存并分配计算缓冲区
    struct ggml_allocr * allocr = NULL;
    {
        // 创建一个分配器来测量内存使用情况
        allocr = ggml_allocr_new_measure_from_backend(model.backend);

        batch.n_tokens = n_batch_max;

        // 创建内存使用情况估计的最坏情况图
        struct ggml_cgraph * gf = gpt2_graph(model, allocr, batch);

        // 计算所需的内存
        size_t mem_size = ggml_allocr_alloc_graph(allocr, gf);

        // 使用所需的内存重新创建分配器
        ggml_allocr_free(allocr);
        buf_compute = ggml_backend_alloc_buffer(model.backend, mem_size);
        allocr = ggml_allocr_new_from_buffer(buf_compute);

        // 打印计算缓冲区的大小
        fprintf(stderr, "%s: compute buffer size: %.2f MB\n", __func__, mem_size/1024.0/1024.0);
    }

    // 初始化采样时间和预测时间
    int64_t t_sample_us  = 0;
    int64_t t_predict_us = 0;

    // 存储模型输出的对数概率
    std::vector<float> logits;

    // 评估初始提示
    batch.n_tokens = embd_inp.size();
    // 遍历 batch 中的每个 token
    for (int32_t i = 0; i < batch.n_tokens; i++) {
        // 将输入的嵌入值赋给 batch 中的 token
        batch.token[i]  = embd_inp[i];
        // 将位置信息赋给 batch 中的 token
        batch.pos[i]    = i;
        // 将序列 ID 赋给 batch 中的 token
        batch.seq_id[i] = 0;
        // 将 logits 初始化为 false
        batch.logits[i] = false;
    }

    // 设置最后一个 token 的 logits 为 true，因为 gpt2_decode 只会输出最后一个 token 的 logits
    batch.logits[batch.n_tokens - 1] = true;

    // 调用 gpt2_decode 函数进行解码
    if (gpt2_decode(model, allocr, batch, params.n_threads, logits) != 0) {
        // 如果解码失败，则打印错误信息并返回
        printf("%s: gpt2_decode() failed\n", __func__);
        return 1;
    }

    // 将系统 KV 缓存分配给所有并行序列
    // 这样，并行序列将“重用”提示 token，而无需复制它们
    for (int32_t i = 1; i < n_parallel; ++i) {
        gpt2_kv_cache_seq_cp(model.kv_cache, 0, i, 0, batch.n_tokens);
    }

    // 如果并行序列数大于 1，则打印生成序列的信息
    if (n_parallel > 1) {
        printf("\n\n%s: generating %d sequences ...\n", __func__, n_parallel);
    }

    // 设置预测的 token 数量为提示 token 数量和模型上下文长度的较小值
    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());

    // 打印提示信息和前 8 个 token 的值
    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());
    printf("%s: number of tokens in prompt = %zu, first 8 tokens: ", __func__, embd_inp.size());
    for (int i = 0; i < std::min(8, (int) embd_inp.size()); i++) {
        printf("%d ", embd_inp[i]);
    }
    printf("\n\n");

    // 创建并行序列的 token 流
    std::vector<gpt_vocab::token> streams(n_parallel);

    // 记录每个并行序列的最后一个 token 的 batch 索引
    // 我们需要这个信息来确定从哪里采样 logits
    std::vector<int32_t> i_batch(n_parallel, batch.n_tokens - 1);

    // 初始化当前 token 数量、总 token 数量和已解码的 token 数量
    int n_cur     = batch.n_tokens;
    int n_len     = batch.n_tokens + params.n_predict;
    int n_decoded = 0;

    // 获取词汇表大小、top-k 值、top-p 值和温度值
    const int   n_vocab = model.hparams.n_vocab;
    const int   top_k = params.top_k;
    const float top_p = params.top_p;
    const float temp  = params.temp;
}
    // 如果并行数大于1，则执行以下代码块
    if (n_parallel > 1) {
        // 打印换行符
        printf("\n");

        // 遍历并行数，打印每个序列的提示和内容
        for (int32_t i = 0; i < n_parallel; ++i) {
            printf("sequence %d:\n\n%s%s\n\n", i, params.prompt.c_str(), streams[i].c_str());
        }
    }

    // 报告时间
    {
        // 获取主函数结束时间
        const int64_t t_main_end_us = ggml_time_us();

        // 打印加载数据的时间
        printf("\n\n");
        printf("%s:     n_decoded = %8d\n",      __func__, n_decoded);
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        printf("%s:  predict time = %8.2f ms\n", __func__, t_predict_us/1000.0f);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    // 释放批处理内存
    gpt2_batch_free(batch);
    // 释放模型上下文内存
    ggml_free(model.ctx);

    // 释放模型权重缓冲区
    ggml_backend_buffer_free(model.buffer_w);
    // 释放模型键值缓存缓冲区
    ggml_backend_buffer_free(model.kv_cache.buffer);
    // 释放计算缓冲区
    ggml_backend_buffer_free(buf_compute);
    // 释放模型后端
    ggml_backend_free(model.backend);

    // 返回0，表示正常结束
    return 0;
# 闭合前面的函数定义
```