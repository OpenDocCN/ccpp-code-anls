# `ggml\examples\starcoder\starcoder-mmap.cpp`

```cpp
#include "ggml/ggml.h"
// 引入 ggml 库的头文件

#include "common.h"
#include "common-ggml.h"
// 引入其他自定义的头文件

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
// 引入标准库的头文件

#if !defined(_WIN32)
// 如果不是在 Windows 平台下，则引入 mmap 相关的头文件
#include <sys/types.h>
#include <sys/mman.h>
#include <unistd.h>
#include <fcntl.h>
#else
// 如果是在 Windows 平台下，则定义 NOMINMAX，并引入 Windows 相关的头文件
#define NOMINMAX
#include <Windows.h>
#endif

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif
// 如果定义了 GGML_USE_CUBLAS，则引入 ggml-cuda 相关的头文件

#ifdef GGML_USE_CLBLAST
#include "ggml-opencl.h"
#endif
// 如果定义了 GGML_USE_CLBLAST，则引入 ggml-opencl 相关的头文件

// default hparams (GPT-2 117M)
// https://huggingface.co/bigcode/gpt_bigcode-santacoder/blob/main/config.json
// 定义默认的超参数结构体 starcoder_hparams，包括词汇量、上下文长度、嵌入维度、头数、层数、ftype 和 eps
struct starcoder_hparams {
    int32_t n_vocab = 49280;
    int32_t n_ctx   = 2048;
    int32_t n_embd  = 2048;
    int32_t n_head  = 16;
    int32_t n_layer = 24;
    int32_t ftype   = 1;
    float   eps     = 1e-5f;
};

struct starcoder_layer {
    // normalization
    // 归一化层的张量指针
    struct ggml_tensor * ln_1_g;
    struct ggml_tensor * ln_1_b;

    struct ggml_tensor * ln_2_g;
    struct ggml_tensor * ln_2_b;

    // attention
    // 注意力机制的张量指针
    struct ggml_tensor * c_attn_attn_w;
    struct ggml_tensor * c_attn_attn_b;

    struct ggml_tensor * c_attn_proj_w;
    struct ggml_tensor * c_attn_proj_b;

    // mlp
    // 多层感知机的张量指针
    struct ggml_tensor * c_mlp_fc_w;
    struct ggml_tensor * c_mlp_fc_b;

    struct ggml_tensor * c_mlp_proj_w;
    struct ggml_tensor * c_mlp_proj_b;
};

struct llama_buffer {
    uint8_t * addr = NULL;
    size_t size = 0;

    llama_buffer() = default;

    void resize(size_t len) {
        // 根据 GGML 使用的不同加速库，分别进行内存分配和初始化
#ifdef GGML_USE_METAL
        free(addr);
        int result = posix_memalign((void **) &addr, sysconf(_SC_PAGESIZE), len);
        if (result == 0) {
            memset(addr, 0, len);
        }
        else {
            addr = NULL;
        }
#else
        delete[] addr;
        addr = new uint8_t[len];
#endif
        size = len;
    }

    ~llama_buffer() {
        // 根据 GGML 使用的不同加速库，释放内存
#ifdef GGML_USE_METAL
        free(addr);
#else
        delete[] addr;
#endif
        addr = NULL;
    }

    // disable copy and move
    // 禁用拷贝和移动构造函数
    // 禁用拷贝构造函数，避免对象被拷贝
    llama_buffer(const llama_buffer&) = delete;
    // 禁用移动构造函数，避免对象被移动
    llama_buffer(llama_buffer&&) = delete;
    // 禁用拷贝赋值运算符，避免对象被拷贝赋值
    llama_buffer& operator=(const llama_buffer&) = delete;
    // 禁用移动赋值运算符，避免对象被移动赋值
    llama_buffer& operator=(llama_buffer&&) = delete;
};

// 定义一个名为 kv_cache 的结构体
struct kv_cache {
    struct ggml_tensor * k;  // 指向 ggml_tensor 结构体的指针 k
    struct ggml_tensor * v;  // 指向 ggml_tensor 结构体的指针 v

    struct ggml_context * ctx = NULL;  // 指向 ggml_context 结构体的指针 ctx，默认为 NULL

    //std::vector<uint8_t> buf;  // 注释掉的代码，使用 llama_buffer 替代
    llama_buffer buf;  // 使用 llama_buffer 类型的 buf

    int n;  // 整型变量 n
};

// 定义一个名为 starcoder_model 的结构体
struct starcoder_model {
    starcoder_hparams hparams;  // starcoder_hparams 类型的 hparams

    // normalization
    struct ggml_tensor * ln_f_g;  // 指向 ggml_tensor 结构体的指针 ln_f_g
    struct ggml_tensor * ln_f_b;  // 指向 ggml_tensor 结构体的指针 ln_f_b

    struct ggml_tensor * wte;     // 指向 ggml_tensor 结构体的指针 wte，表示位置嵌入
    struct ggml_tensor * wpe;     // 指向 ggml_tensor 结构体的指针 wpe，表示标记嵌入
    struct ggml_tensor * lm_head; // 指向 ggml_tensor 结构体的指针 lm_head，表示语言模型头部

    std::vector<starcoder_layer> layers;  // 包含 starcoder_layer 类型的向量 layers

    // key + value memory
    //struct ggml_tensor * memory_k;  // 注释掉的代码，使用 kv_cache 替代
    //struct ggml_tensor * memory_v;  // 注释掉的代码，使用 kv_cache 替代
    struct kv_cache cache;  // 使用 kv_cache 类型的 cache

    // model memory mapped file
    void * mm_addr = NULL;  // 指向 void 类型的指针 mm_addr，默认为 NULL
    uint64_t mm_length = 0;  // 64 位无符号整型变量 mm_length，默认为 0

    //
    struct ggml_context * ctx;  // 指向 ggml_context 结构体的指针 ctx
    std::map<std::string, struct ggml_tensor *> tensors;  // 包含指向 ggml_tensor 结构体的指针的字符串映射
};

// 从 PR #613 (https://github.com/ggerganov/llama.cpp/pull/613) 中获取
static void *mmap_file(const char *fname, uint64_t *mm_length) {
#if defined(_WIN32) && !defined(_POSIX_MAPPED_FILES)
    HANDLE hFile = CreateFileA(fname,  // 创建一个文件句柄
                               GENERIC_READ,  // 读取权限
                               FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE,  // 文件共享权限
                               NULL,  // 安全属性
                               OPEN_EXISTING,  // 打开已存在的文件
                               FILE_ATTRIBUTE_NORMAL | FILE_ATTRIBUTE_NOT_CONTENT_INDEXED,  // 文件属性
                               NULL);  // 模板文件句柄
    if (hFile == INVALID_HANDLE_VALUE) return 0;  // 如果文件句柄无效，则返回 0
    LARGE_INTEGER fileSize;  // 定义一个大整数类型的 fileSize
    fileSize.QuadPart = -1;  // 将 fileSize 的 QuadPart 属性设置为 -1
    GetFileSizeEx(hFile, &fileSize);  // 获取文件大小
    int64_t length = fileSize.QuadPart;  // 获取文件大小并赋值给 length
    HANDLE hMapping = CreateFileMappingA(hFile, NULL, PAGE_READONLY, 0, 0, NULL);  // 创建文件映射
    CloseHandle(hFile);  // 关闭文件句柄
    if (!hMapping) return 0;  // 如果文件映射失败，则返回 0
    void *addr = MapViewOfFile(hMapping, FILE_MAP_READ, 0, 0, 0);  // 映射文件到内存
    CloseHandle(hMapping);  // 关闭文件映射
    if (!addr) return 0;  // 如果映射失败，则返回 0
#else
    int fd = open(fname, O_RDONLY);  // 打开文件
    if (fd == -1) return 0;  // 如果文件打开失败，则返回 0
    int64_t length = lseek(fd, 0, SEEK_END);  // 获取文件大小
    // 使用 mmap 函数将文件映射到内存中的地址，NULL表示由系统自动分配地址
    // length 表示映射的长度
    // PROT_READ 表示映射区域可读
    // MAP_SHARED 表示映射区域可被其他映射区域共享
    // fd 是文件描述符，0 表示从文件的起始位置开始映射
    void *addr = mmap(NULL, length, PROT_READ, MAP_SHARED, fd, 0);
    // 关闭文件描述符
    close(fd);
    // 检查映射是否成功，如果失败则返回 0
    if (addr == MAP_FAILED) return 0;
#endif
    // 设置指针指向的内存区域的长度
    *mm_length = length;
    // 返回指向内存区域的指针
    return addr;
}

// 释放映射的文件内存
static void munmap_file(void * addr, size_t length) {
    // 如果是 Windows 平台且不支持 POSIX 映射文件，则调用 UnmapViewOfFile 函数
    #if defined(_WIN32) && !defined(_POSIX_MAPPED_FILES)
    UnmapViewOfFile(addr);
    // 否则调用 munmap 函数
    #else
    munmap(addr, length);
    #endif
}

// 从文件加载模型的权重
bool starcoder_model_load(const std::string & fname, starcoder_model & model, gpt_vocab & vocab, int32_t n_gpu_layers) {
    // 打印加载模型的信息
    printf("%s: loading model from '%s'\n", __func__, fname.c_str());

    // 打开文件流
    auto fin = std::ifstream(fname, std::ios::binary);
    // 如果文件流打开失败
    if (!fin) {
        // 打印错误信息
        fprintf(stderr, "%s: failed to open '%s'\n", __func__, fname.c_str());
        // 返回加载失败
        return false;
    }

    // 创建缓冲区
    std::vector<char> f_buf(1024*1024);
    // 设置文件流的缓冲区
    fin.rdbuf()->pubsetbuf(f_buf.data(), f_buf.size());

    // 验证魔数
    {
        uint32_t magic;
        // 读取文件中的魔数
        fin.read((char *) &magic, sizeof(magic));
        // 如果魔数不匹配
        if (magic != 0x67676d6c) {
            // 打印错误信息
            fprintf(stderr, "%s: invalid model file '%s' (bad magic)\n", __func__, fname.c_str());
            // 返回加载失败
            return false;
        }
    }

    // 加载超参数
    {
        // 获取模型的超参数引用
        auto & hparams = model.hparams;

        // 从文件中读取词汇量大小，并存入超参数中
        fin.read((char *) &hparams.n_vocab, sizeof(hparams.n_vocab));
        // 从文件中读取上下文大小，并存入超参数中
        fin.read((char *) &hparams.n_ctx,   sizeof(hparams.n_ctx));
        // 从文件中读取嵌入维度，并存入超参数中
        fin.read((char *) &hparams.n_embd,  sizeof(hparams.n_embd));
        // 从文件中读取头数，并存入超参数中
        fin.read((char *) &hparams.n_head,  sizeof(hparams.n_head));
        // 从文件中读取层数，并存入超参数中
        fin.read((char *) &hparams.n_layer, sizeof(hparams.n_layer));
        // 从文件中读取特征类型，并存入超参数中
        fin.read((char *) &hparams.ftype,   sizeof(hparams.ftype));

        // 计算特征类型的量化版本
        const int32_t qntvr = hparams.ftype / GGML_QNT_VERSION_FACTOR;

        // 打印超参数中的词汇量大小
        printf("%s: n_vocab = %d\n", __func__, hparams.n_vocab);
        // 打印超参数中的上下文大小
        printf("%s: n_ctx   = %d\n", __func__, hparams.n_ctx);
        // 打印超参数中的嵌入维度
        printf("%s: n_embd  = %d\n", __func__, hparams.n_embd);
        // 打印超参数中的头数
        printf("%s: n_head  = %d\n", __func__, hparams.n_head);
        // 打印超参数中的层数
        printf("%s: n_layer = %d\n", __func__, hparams.n_layer);
        // 打印超参数中的特征类型
        printf("%s: ftype   = %d\n", __func__, hparams.ftype);
        // 打印计算得到的特征类型的量化版本
        printf("%s: qntvr   = %d\n", __func__, qntvr);

        // 计算特征类型的余数，并存回超参数中
        hparams.ftype %= GGML_QNT_VERSION_FACTOR;
    }

    // 加载词汇表
    {
        // 读取模型文件中的词汇表大小
        int32_t n_vocab = 0;
        fin.read((char *) &n_vocab, sizeof(n_vocab));

        // 如果读取到的词汇表大小与模型参数中的词汇表大小不一致，则输出错误信息并返回 false
        if (n_vocab != model.hparams.n_vocab) {
            fprintf(stderr, "%s: invalid model file '%s' (bad vocab size %d != %d)\n",
                    __func__, fname.c_str(), n_vocab, model.hparams.n_vocab);
            return false;
        }

        // 读取词汇表中的单词并将其添加到词汇表中
        std::string word;
        std::vector<char> buf(128);

        for (int i = 0; i < n_vocab; i++) {
            // 读取单词的长度
            uint32_t len;
            fin.read((char *) &len, sizeof(len));

            // 调整缓冲区大小并读取单词数据
            buf.resize(len);
            fin.read((char *) buf.data(), len);
            word.assign(buf.data(), len);

            // 将单词添加到词汇表的映射关系中
            vocab.token_to_id[word] = i;
            vocab.id_to_token[i] = word;

            // 如果 i 小于 10，则输出单词信息
            // if (i < 10) fprintf(stderr, "%.s: vocab[%d] = '%s'\n", __func__, i, word.c_str());
        }

        // 添加 StarChat 的特殊标记到词汇表中
        for (std::string token : {
                "<|system|>",
                "<|user|>",
                "<|assistant|>",
                "<|end|>",
                }) {
            // 如果词汇表中不存在该特殊标记，则添加到词汇表中
            if (vocab.token_to_id.find(token) != vocab.token_to_id.end()) {
                vocab.add_special_token(token);
            }
        }
    }

    // 映射模型文件到内存中
    char *mm_addr = NULL;
    model.mm_addr = mmap_file(fname.c_str(), &model.mm_length);
    // 如果映射失败，则输出错误信息并返回 false
    if (model.mm_addr == NULL) {
        fprintf(stderr, "%s: failed to mmap '%s'\n", __func__, fname.c_str());
        return false;
    }
    mm_addr = (char *)model.mm_addr;
    // 输出映射文件的大小
    fprintf(stderr, "%s: ggml map size = %6.2f MB\n", __func__, model.mm_length/(1024.0*1024.0));

    // 对于大张量，可以选择将数据存储为 16 位浮点数或量化形式，以节省内存并加快计算速度
    ggml_type wtype = ggml_ftype_to_ggml_type((ggml_ftype) (model.hparams.ftype));
    }
    // 如果 wtype 等于 GGML_TYPE_COUNT
    if (wtype == GGML_TYPE_COUNT) {
        // 输出错误信息到标准错误流
        fprintf(stderr, "%s: invalid model file '%s' (bad ftype value %d)\n",
                __func__, fname.c_str(), model.hparams.ftype);
        // 返回 false
        return false;
    }

    // 获取模型的上下文
    auto & ctx = model.ctx;

    // 初始化上下文大小为 0
    size_t ctx_size = 0;

    // 创建 ggml 上下文
    {
        // 定义 ggml_init_params 结构体
        struct ggml_init_params params = {
            /*.mem_size   =*/ ctx_size,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
        };

        // 调用 ggml_init 函数创建上下文
        model.ctx = ggml_init(params);
        // 如果创建失败，输出错误信息到标准错误流并返回 false
        if (!model.ctx) {
            fprintf(stderr, "%s: ggml_init() failed\n", __func__);
            return false;
        }
    }

    // 准备权重的内存空间
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;

        // 获取超参数中的值
        const int n_embd  = hparams.n_embd;
        const int n_layer = hparams.n_layer;
        const int n_ctx   = hparams.n_ctx;

        const int n_mem      = n_layer*n_ctx;
        const int n_elements = n_embd*n_mem;

        // 调整缓冲区大小
        model.cache.buf.resize(2u*n_elements*ggml_type_size(GGML_TYPE_F16) + 2u*1024*1024);

        // 定义 ggml_init_params 结构体
        struct ggml_init_params c_params;
        c_params.mem_size   = model.cache.buf.size;
        c_params.mem_buffer = model.cache.buf.addr;
        c_params.no_alloc   = false;

        // 创建缓存上下文
        model.cache.ctx = ggml_init(c_params);

        // 如果创建失败，输出错误信息到标准错误流并返回 false
        if (!model.cache.ctx) {
            fprintf(stderr, "%s: failed to allocate memory for kv cache\n", __func__);
            return false;
        }

        // 创建 key 和 value 的张量
        model.cache.k = ggml_new_tensor_1d(model.cache.ctx, GGML_TYPE_F16, n_elements);
        model.cache.v = ggml_new_tensor_1d(model.cache.ctx, GGML_TYPE_F16, n_elements);

        // 计算内存大小并输出信息
        const size_t memory_size = ggml_nbytes(model.cache.k) + ggml_nbytes(model.cache.v);
        printf("%s: kv_cache memory size = %8.2f MB, n_mem = %d\n", __func__, memory_size/1024.0/1024.0, n_mem);
    }

    // 加载权重
    fin.close();
#ifdef GGML_USE_CUBLAS
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
        // 计算需要使用 GPU 的层数，取 n_gpu_layers 和 hparams.n_layer 中较小的一个
        const int n_gpu = std::min(n_gpu_layers, int(hparams.n_layer));

        // 打印信息，表示将有多少层的计算 offloading 到 GPU 上
        fprintf(stderr, "%s: [cublas] offloading %d layers to GPU\n", __func__, n_gpu);

        // 初始化 VRAM 总量
        size_t vram_total = 0;

        // 遍历每一层需要 offloading 到 GPU 上的操作
        for (int i = 0; i < n_gpu; ++i) {
            // 获取当前层的信息
            const auto & layer = model.layers[i];

            // 将当前层的 c_attn_attn_w 数据放到 GPU 上，并计算 VRAM 使用量
            layer.c_attn_attn_w->backend = GGML_BACKEND_GPU;
            ggml_cuda_transform_tensor((uint8_t *)layer.c_attn_attn_w->data, layer.c_attn_attn_w); vram_total += ggml_nbytes(layer.c_attn_attn_w);

            // 将当前层的 c_attn_proj_w 数据放到 GPU 上，并计算 VRAM 使用量
            layer.c_attn_proj_w->backend = GGML_BACKEND_GPU;
            ggml_cuda_transform_tensor((uint8_t *)layer.c_attn_proj_w->data, layer.c_attn_proj_w); vram_total += ggml_nbytes(layer.c_attn_proj_w);

            // 将当前层的 c_mlp_fc_w 数据放到 GPU 上，并计算 VRAM 使用量
            layer.c_mlp_fc_w->backend = GGML_BACKEND_GPU;
            ggml_cuda_transform_tensor((uint8_t *)layer.c_mlp_fc_w->data, layer.c_mlp_fc_w); vram_total += ggml_nbytes(layer.c_mlp_fc_w);

            // 将当前层的 c_mlp_proj_w 数据放到 GPU 上，并计算 VRAM 使用量
            layer.c_mlp_proj_w->backend = GGML_BACKEND_GPU;
            ggml_cuda_transform_tensor((uint8_t *)layer.c_mlp_proj_w->data, layer.c_mlp_proj_w); vram_total += ggml_nbytes(layer.c_mlp_proj_w);
        }

        // 设置 GPU 上的 scratch size 为 0，禁用 scratch
        ggml_cuda_set_scratch_size(0); // disable scratch

        // 打印信息，表示将输出层的计算 offloading 到 GPU 上
        //if (n_gpu_layers > (int) hparams.n_layer) {
        //    fprintf(stderr, "%s: [cublas] offloading output layer to GPU\n", __func__);
        //    ggml_cuda_transform_tensor(model.output); vram_total += ggml_nbytes(model.output);
        //}

        // 打印总的 VRAM 使用量
        fprintf(stderr, "%s: [cublas] total VRAM used: %zu MB\n", __func__, vram_total / 1024 / 1024);
    }
#elif defined(GGML_USE_CLBLAST)
    //From koboldcpp
    {
        // 获取模型的超参数
        const auto & hparams = model.hparams;
        // 初始化总显存大小为0
        size_t vram_total = 0;
        // 确定要使用的GPU数量，取n_gpu_layers和hparams.n_layer中较小的一个
        const int n_gpu = std::min(n_gpu_layers, int(hparams.n_layer));
        // 打印信息，指示将有多少层模型被 offloading 到 GPU 上
        fprintf(stderr, "%s: [opencl] offloading %d layers to GPU\n", __func__, n_gpu);
        // 遍历每个要 offloading 到 GPU 上的层
        for (int i = 0; i < n_gpu; ++i) {
            // 获取当前层的引用
            const auto & layer = model.layers[i];
            // 设置当前层的 c_attn_attn_w 的后端为 GPU
            layer.c_attn_attn_w->backend = GGML_BACKEND_GPU;
            // 设置当前层的 c_attn_proj_w 的后端为 GPU
            layer.c_attn_proj_w->backend = GGML_BACKEND_GPU;
            // 设置当前层的 c_mlp_fc_w 的后端为 GPU
            layer.c_mlp_fc_w->backend = GGML_BACKEND_GPU;
            // 设置当前层的 c_mlp_proj_w 的后端为 GPU
            layer.c_mlp_proj_w->backend = GGML_BACKEND_GPU;
            // 将当前层的 c_attn_attn_w 数据转换为 GPU 可用的格式，并计算显存占用
            ggml_cl_transform_tensor(layer.c_attn_attn_w->data,layer.c_attn_attn_w); vram_total += ggml_nbytes(layer.c_attn_attn_w);
            // 将当前层的 c_attn_proj_w 数据转换为 GPU 可用的格式，并计算显存占用
            ggml_cl_transform_tensor(layer.c_attn_proj_w->data,layer.c_attn_proj_w); vram_total += ggml_nbytes(layer.c_attn_proj_w);
            // 将当前层的 c_mlp_fc_w 数据转换为 GPU 可用的格式，并计算显存占用
            ggml_cl_transform_tensor(layer.c_mlp_fc_w->data,layer.c_mlp_fc_w); vram_total += ggml_nbytes(layer.c_mlp_fc_w);
            // 将当前层的 c_mlp_proj_w 数据转换为 GPU 可用的格式，并计算显存占用
            ggml_cl_transform_tensor(layer.c_mlp_proj_w->data,layer.c_mlp_proj_w); vram_total += ggml_nbytes(layer.c_mlp_proj_w);
        }
        // 打印信息，指示总共使用的显存大小
        fprintf(stderr, "%s: [opencl] total VRAM used: %zu MB\n", __func__, vram_total / 1024 / 1024);
    }
    #endif
    
    // 返回 true
    return true;
    }
// 评估变换器
//
//   - model:     模型
//   - n_threads: 使用的线程数
//   - n_past:    到目前为止的上下文大小
//   - embd_inp:  上下文中标记的嵌入
//   - embd_w:    下一个标记的预测对数
//   - mem_per_token: 每个标记的内存
//
bool starcoder_eval(
        const starcoder_model & model,
        const int n_threads,
        const int n_past,
        const std::vector<gpt_vocab::id> & embd_inp,
              std::vector<float>         & embd_w,
              size_t                     & mem_per_token) {

    const int N = int(embd_inp.size());

    const auto & hparams = model.hparams;

    auto & cache = model.cache;

    const int n_embd  = hparams.n_embd;
    const int n_layer = hparams.n_layer;
    const int n_ctx   = hparams.n_ctx;
    const int n_head  = hparams.n_head;
    const int n_vocab = hparams.n_vocab;

    // 缓冲区对于大 n_batch (256) 太小
    //static size_t buf_size = 256u*1024*1024;
    static size_t buf_size = 256u*1024*1024*2;
    static void * buf = malloc(buf_size);

    // 使用 2 个临时缓冲区
    // TODO: 非常巧妙的解决方案 - 以更优雅的方式重新实现
    static size_t scratch0_size = 256u*1024*1024*2;
    static void * scratch0 = malloc(scratch0_size);

    static size_t scratch1_size = 256u*1024*1024*2;
    static void * scratch1 = malloc(scratch1_size);

    if (mem_per_token > 0 && mem_per_token*N > buf_size) {
        const size_t buf_size_new = size_t(1.1*(mem_per_token*N)); // 添加 10% 来考虑 ggml 对象的开销
        printf("\n%s: 从 %zu 重新分配缓冲区到 %zu 字节\n", __func__, buf_size, buf_size_new);

        // 重新分配
        buf_size = buf_size_new;
        buf = realloc(buf, buf_size);
        if (buf == nullptr) {
            fprintf(stderr, "%s: 分配 %zu 字节失败\n", __func__, buf_size);
            return false;
        }
    }
}
    // 初始化参数结构体，设置内存大小、内存缓冲区和是否允许分配内存
    struct ggml_init_params params = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf,
        /*.no_alloc   =*/ false,
    };

    // 初始化上下文对象
    struct ggml_context * ctx0 = ggml_init(params);
    // 创建新的计算图对象
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 创建新的一维张量对象
    struct ggml_tensor * embd = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);

    // 将输入数据拷贝到张量的数据缓冲区
    memcpy(embd->data, embd_inp.data(), N*ggml_element_size(embd));

    // 创建新的一维张量对象
    struct ggml_tensor * position = ggml_new_tensor_1d(ctx0, GGML_TYPE_I32, N);
    // 循环设置位置张量的数据
    for (int i = 0; i < N; ++i) {
        ((int32_t *) position->data)[i] = n_past + i;
    }

    // 计算 wte + wpe
    struct ggml_tensor * inpL =
        ggml_add(ctx0,
                ggml_get_rows(ctx0, model.wte, embd),
                ggml_get_rows(ctx0, model.wpe, position));

    // 设置临时缓冲区
    ggml_set_scratch(ctx0, { 0, scratch0_size, scratch0, });

    // 归一化处理
    {
        // 归一化处理
        inpL = ggml_norm(ctx0, inpL, hparams.eps);

        // inpL = ln_f_g*inpL + ln_f_b
        inpL = ggml_add(ctx0,
                ggml_mul(ctx0,
                    ggml_repeat(ctx0, model.ln_f_g, inpL),
                    inpL),
                ggml_repeat(ctx0, model.ln_f_b, inpL));
    }

    // 清除临时缓冲区
    ggml_set_scratch(ctx0, { 0, 0, nullptr, });

    // 矩阵相乘
    inpL = ggml_mul_mat(ctx0, model.lm_head, inpL);

    // 计算概率
    //inpL = ggml_soft_max_inplace(ctx0, inpL);

    // 运行计算
    ggml_build_forward_expand(gf, inpL);
    ggml_graph_compute_with_ctx(ctx0, gf, n_threads);

    // 更新结果
    embd_w.resize(n_vocab);
    memcpy(embd_w.data(), (float *) ggml_get_data(inpL) + (n_vocab*(N-1)), sizeof(float)*n_vocab);
    # 如果每个令牌的内存占用为0，则将其设置为上下文内存占用除以令牌数量
    if (mem_per_token == 0) {
        mem_per_token = ggml_used_mem(ctx0)/N;
    }
    //printf("used_mem = %zu MB\n", ggml_used_mem(ctx0)/(1024*1024));

    # 释放上下文内存
    ggml_free(ctx0);

    # 返回 true
    return true;
}

int main(int argc, char ** argv) {
    ggml_time_init();  // 初始化时间

    const int64_t t_main_start_us = ggml_time_us();  // 获取当前时间

    gpt_params params;  // 定义参数对象
    params.model = "models/gpt-2-117M/ggml-model.bin";  // 设置模型路径

    if (gpt_params_parse(argc, argv, params) == false) {  // 解析命令行参数
        return 1;  // 如果解析失败，返回1
    }

    if (params.seed < 0) {  // 如果种子值小于0
        params.seed = int(time(NULL));  // 使用当前时间作为种子值
    }

    printf("%s: seed = %d\n", __func__, params.seed);  // 打印种子值

    std::mt19937 rng(params.seed);  // 使用种子值初始化随机数生成器
    if (params.prompt.empty()) {  // 如果提示为空
        params.prompt = gpt_random_prompt(rng);  // 生成随机提示
    }

    int64_t t_load_us = 0;  // 初始化加载时间

    gpt_vocab vocab;  // 定义词汇表对象
    starcoder_model model;  // 定义模型对象

    // load the model
    {
        const int64_t t_start_us = ggml_time_us();  // 获取加载模型前的时间

        if (!starcoder_model_load(params.model, model, vocab, params.n_gpu_layers)) {  // 加载模型
            fprintf(stderr, "%s: failed to load model from '%s'\n", __func__, params.model.c_str());  // 打印加载失败信息
            return 1;  // 返回1
        }

        t_load_us = ggml_time_us() - t_start_us;  // 计算加载模型所用时间

        test_gpt_tokenizer(vocab, params.token_test);  // 测试分词器
    }

    int n_past = 0;  // 初始化过去数量

    int64_t t_sample_us  = 0;  // 初始化采样时间
    int64_t t_predict_us = 0;  // 初始化预测时间

    std::vector<float> logits;  // 定义logits向量

    // tokenize the prompt
    std::vector<gpt_vocab::id> embd_inp = ::gpt_tokenize(vocab, params.prompt);  // 对提示进行分词

    params.n_predict = std::min(params.n_predict, model.hparams.n_ctx - (int) embd_inp.size());  // 设置预测数量

    printf("%s: prompt: '%s'\n", __func__, params.prompt.c_str());  // 打印提示
    printf("%s: number of tokens in prompt = %zu\n", __func__, embd_inp.size());  // 打印提示中的标记数量
    for (size_t i = 0; i < embd_inp.size(); i++) {  // 遍历提示中的标记
        printf("%s: token[%zu] = %6d, %s\n", __func__, i, embd_inp[i], vocab.id_to_token.at(embd_inp[i]).c_str());  // 打印每个标记的信息
    }
    printf("\n\n");  // 打印空行

    // Handle StarChat "<|end|>" token.
    gpt_vocab::id starchat_end_token = -1;  // 初始化StarChat "<|end|>"标记
    {
        const auto it = vocab.token_to_id.find("<|end|>");  // 在词汇表中查找"<|end|>"标记
        if (it != vocab.token_to_id.end()) {  // 如果找到了
            starchat_end_token = it->second;  // 设置StarChat "<|end|>"标记
        }
    }

    // submit the input prompt token-by-token
}
    // 通过减少推理过程中的内存使用，来换取一点点初始速度
    std::vector<gpt_vocab::id> embd;

    // 确定每个标记所需的推理内存
    size_t mem_per_token = 0;
    printf("Calling starcoder_eval\n");
    // 调用 starcoder_eval 函数，计算每个标记所需的推理内存
    starcoder_eval(model, params.n_threads, 0, { 0, 1, 2, 3 }, logits, mem_per_token);

    }

    // 报告时间
    {
        const int64_t t_main_end_us = ggml_time_us();

        printf("\n\n");
        // 打印每个标记所需的内存
        printf("%s: mem per token = %8zu bytes\n", __func__, mem_per_token);
        // 打印加载时间
        printf("%s:     load time = %8.2f ms\n", __func__, t_load_us/1000.0f);
        // 打印采样时间
        printf("%s:   sample time = %8.2f ms\n", __func__, t_sample_us/1000.0f);
        // 打印预测时间和每个标记的平均时间
        printf("%s:  predict time = %8.2f ms / %.2f ms per token\n", __func__, t_predict_us/1000.0f, t_predict_us/1000.0f/(n_past - embd_inp.size()));
        // 打印总时间
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0f);
    }

    // 释放模型上下文
    ggml_free(model.ctx);

    // 如果模型内存映射地址存在，则解除映射
    if (model.mm_addr) {
        munmap_file(model.mm_addr, model.mm_length);
    }

    // 返回成功
    return 0;
# 闭合前面的函数定义
```