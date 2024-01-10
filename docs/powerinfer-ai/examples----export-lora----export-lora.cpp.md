# `PowerInfer\examples\export-lora\export-lora.cpp`

```
#include "common.h"
#include "ggml.h"
#include "ggml-alloc.h"

#include <vector>
#include <string>
#include <thread>

static const size_t tensor_alignment = 32;

struct lora_info {
    std::string filename; // 文件名
    float scale; // 缩放比例
};

struct export_lora_params {
    std::string fn_model_base; // 模型基本文件名
    std::string fn_model_out; // 输出模型文件名
    std::vector<struct lora_info> lora; // lora_info 结构体的向量
    int n_threads; // 线程数
};

struct lora_data {
    struct lora_info     info; // lora_info 结构体
    std::vector<uint8_t> data; // 无符号 8 位整数的向量
    struct ggml_context * ctx; // ggml_context 结构体指针

    uint32_t lora_r; // lora_r 变量
    uint32_t lora_alpha; // lora_alpha 变量
};

struct llama_file {
    // use FILE * so we don't have to re-open the file to mmap
    FILE * fp; // 文件指针
    size_t size; // 文件大小

    llama_file(const char * fname, const char * mode) { // 构造函数，打开文件并获取文件大小
        fp = std::fopen(fname, mode);
        if (fp == NULL) {
            size = 0;
        } else {
            seek(0, SEEK_END);
            size = tell();
            seek(0, SEEK_SET);
        }
    }

    size_t tell() const { // 获取当前文件指针位置
#ifdef _WIN32
        __int64 ret = _ftelli64(fp);
#else
        long ret = std::ftell(fp);
#endif
        GGML_ASSERT(ret != -1); // 断言，确保不会失败
        return (size_t) ret;
    }

    void seek(size_t offset, int whence) { // 移动文件指针位置
#ifdef _WIN32
        int ret = _fseeki64(fp, (__int64) offset, whence);
#else
        int ret = std::fseek(fp, (long) offset, whence);
#endif
        GGML_ASSERT(ret == 0); // 断言，确保不会失败
    }

    void read_raw(void * ptr, size_t size) { // 读取原始数据
        if (size == 0) {
            return;
        }
        errno = 0;
        std::size_t ret = std::fread(ptr, size, 1, fp);
        if (ferror(fp)) {
            die_fmt("read error: %s", strerror(errno));
        }
        if (ret != 1) {
            die("unexpectedly reached end of file");
        }
    }

    std::uint32_t read_u32() { // 读取无符号 32 位整数
        std::uint32_t ret;
        read_raw(&ret, sizeof(ret));
        return ret;
    }
    // 读取指定长度的字符串并返回
    std::string read_string(std::uint32_t len) {
        // 创建一个指定长度的字符向量
        std::vector<char> chars(len);
        // 读取指定长度的原始数据到字符向量中
        read_raw(chars.data(), len);
        // 使用字符向量中的数据创建字符串并返回
        return std::string(chars.data(), len);
    }

    // 写入指定大小的原始数据
    void write_raw(const void * ptr, size_t size) {
        // 如果大小为0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 将数据写入文件
        size_t ret = std::fwrite(ptr, size, 1, fp);
        // 如果写入不成功，则输出错误信息并终止程序
        if (ret != 1) {
            die_fmt("write error: %s", strerror(errno));
        }
    }

    // 写入32位无符号整数
    void write_u32(std::uint32_t val) {
        // 写入32位无符号整数的原始数据
        write_raw(&val, sizeof(val));
    }

    // 判断是否已到达文件末尾
    bool eof() {
        // 返回当前位置是否大于等于文件大小
        return tell() >= size;
    }

    // 析构函数，关闭文件指针
    ~llama_file() {
        // 如果文件指针存在，则关闭文件
        if (fp) {
            std::fclose(fp);
        }
    }
};

// 返回默认的导出 LoRA 参数
static struct export_lora_params get_default_export_lora_params() {
    struct export_lora_params result;
    result.fn_model_base = "";
    result.fn_model_out  = "";
    result.n_threads = GGML_DEFAULT_N_THREADS;
    return result;
}

// 打印使用说明
static void export_lora_print_usage(int /*argc*/, char ** argv, const struct export_lora_params * params) {
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help                         show this help message and exit\n");
    fprintf(stderr, "  -m FNAME, --model-base FNAME       model path from which to load base model (default '%s')\n", params->fn_model_base.c_str());
    fprintf(stderr, "  -o FNAME, --model-out FNAME        path to save exported model (default '%s')\n", params->fn_model_out.c_str());
    fprintf(stderr, "  -l FNAME, --lora FNAME             apply LoRA adapter\n");
    fprintf(stderr, "  -s FNAME S, --lora-scaled FNAME S  apply LoRA adapter with user defined scaling S\n");
    fprintf(stderr, "  -t N, --threads N                  number of threads to use during computation (default: %d)\n", params->n_threads);
}

// 解析导出 LoRA 参数
static bool export_lora_params_parse(int argc, char ** argv, struct export_lora_params * params) {
    bool invalid_param = false;
    std::string arg;
    // 获取默认的导出 LoRA 参数
    struct export_lora_params default_params = get_default_export_lora_params();
    // 参数前缀
    const std::string arg_prefix = "--";
    # 遍历命令行参数列表，从第二个参数开始
    for (int i = 1; i < argc; i++) {
        # 获取当前参数
        arg = argv[i];
        # 检查参数是否以指定前缀开头
        if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
            # 将参数中的下划线替换为破折号
            std::replace(arg.begin(), arg.end(), '_', '-');
        }

        # 检查参数是否为模型基础文件名选项
        if (arg == "-m" || arg == "--model-base") {
            # 检查下一个参数是否存在
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将下一个参数作为模型基础文件名
            params->fn_model_base = argv[i];
        } else if (arg == "-o" || arg == "--model-out") {
            # 检查下一个参数是否存在
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将下一个参数作为模型输出文件名
            params->fn_model_out = argv[i];
        } else if (arg == "-l" || arg == "--lora") {
            # 检查下一个参数是否存在
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 创建 lora_info 结构体并设置文件名和默认比例
            struct lora_info lora;
            lora.filename = argv[i];
            lora.scale = 1.0f;
            # 将 lora_info 结构体添加到参数对象的 lora 列表中
            params->lora.push_back(lora);
        } else if (arg == "-s" || arg == "--lora-scaled") {
            # 检查下一个参数是否存在
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 创建 lora_info 结构体并设置文件名
            struct lora_info lora;
            lora.filename = argv[i];
            # 检查下一个参数是否存在
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将下一个参数转换为浮点数，并设置为 lora_info 结构体的比例
            lora.scale = std::stof(argv[i]);
            # 将 lora_info 结构体添加到参数对象的 lora 列表中
            params->lora.push_back(lora);
        } else if (arg == "-t" || arg == "--threads") {
            # 检查下一个参数是否存在
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            # 将下一个参数转换为整数，并设置为线程数
            params->n_threads = std::stoi(argv[i]);
            # 如果线程数小于等于0，则设置为硬件并发数
            if (params->n_threads <= 0) {
                params->n_threads = std::thread::hardware_concurrency();
            }
        } else {
            # 输出错误信息并退出程序
            fprintf(stderr, "error: unknown argument: '%s'\n", arg.c_str());
            export_lora_print_usage(argc, argv, &default_params);
            exit(1);
        }
    }
    # 如果模型基础文件名与默认参数中的相同，则输出错误信息并打印用法，然后退出程序
    if (params->fn_model_base == default_params.fn_model_base) {
        fprintf(stderr, "error: please specify a filename for model-base.\n");
        export_lora_print_usage(argc, argv, &default_params);
        exit(1);
    }
    # 如果模型输出文件名与默认参数中的相同，则输出错误信息并打印用法，然后退出程序
    if (params->fn_model_out == default_params.fn_model_out) {
        fprintf(stderr, "error: please specify a filename for model-out.\n");
        export_lora_print_usage(argc, argv, &default_params);
        exit(1);
    }
    # 如果存在无效参数，则输出错误信息并打印用法，然后退出程序
    if (invalid_param) {
        fprintf(stderr, "error: invalid parameter for argument: '%s'\n", arg.c_str());
        export_lora_print_usage(argc, argv, &default_params);
        exit(1);
    }
    # 返回 true，表示函数执行成功
    return true;
// 释放 lora_data 结构体内存
static void free_lora(struct lora_data * lora) {
    // 检查 lora_data 结构体中的 ctx 是否为空，如果不为空则释放内存
    if (lora->ctx != NULL) {
        ggml_free(lora->ctx);
    }
    // 释放 lora_data 结构体内存
    delete lora;
}

// 加载 lora_data 结构体
static struct lora_data * load_lora(struct lora_info * info) {
    // 创建 lora_data 结构体指针，并分配内存
    struct lora_data * result = new struct lora_data;
    // 将 info 结构体的内容复制到 result 结构体中
    result->info = *info;
    // 初始化 result 结构体中的 ctx、lora_r 和 lora_alpha
    result->ctx = NULL;
    result->lora_r     = 1;
    result->lora_alpha = 1;

    // 打开 lora 文件
    struct llama_file file(info->filename.c_str(), "rb");
    // 如果文件指针为空，则输出警告信息并释放 result 结构体内存，返回空指针
    if (file.fp == NULL) {
        fprintf(stderr, "warning: Could not open lora adapter '%s'. Ignoring this adapter.\n",
            info->filename.c_str());
        free_lora(result);
        return NULL;
    }

    // 初始化 ggml_init_params 结构体
    struct ggml_init_params params_ggml;
    params_ggml.mem_size   = ggml_tensor_overhead() * GGML_DEFAULT_GRAPH_SIZE;
    params_ggml.mem_buffer = NULL;
    params_ggml.no_alloc   = true;
    // 初始化 result 结构体中的 ctx
    result->ctx = ggml_init(params_ggml);

    // 读取文件中的魔数
    uint32_t LLAMA_FILE_MAGIC_LORA = 0x67676C61; // 'ggla'
    uint32_t magic   = file.read_u32();
    // 如果魔数不符合预期，则输出错误信息
    if (magic != LLAMA_FILE_MAGIC_LORA) {
        die_fmt("unexpected lora header file magic in '%s'", info->filename.c_str());
    }
    // 读取文件中的版本号
    uint32_t version = file.read_u32();
    // 如果版本号不符合预期，则输出错误信息
    if (version != 1) {
        die_fmt("unexpected lora file version '%u' in '%s'", (unsigned) version, info->filename.c_str());
    }
    // 读取文件中的 lora_r 和 lora_alpha
    result->lora_r     = file.read_u32();
    result->lora_alpha = file.read_u32();
    // 从文件中读取张量信息
    std::vector<char> name_buf;
    std::vector<struct ggml_tensor *> tensors;
    std::vector<size_t> tensors_offset;
    size_t total_nbytes_pad = 0;
}
    // 当文件未到达末尾时执行循环
    while(!file.eof()) {
        // 初始化一个长度为4的整型数组
        int64_t ne[4]   = {1,1,1,1};
        // 读取并存储数据维度
        uint32_t n_dims  = file.read_u32();
        // 读取并存储名称长度
        uint32_t namelen = file.read_u32();
        // 读取并存储数据类型
        uint32_t type    = file.read_u32();
        // 遍历数据维度，读取并存储每个维度的大小
        for (uint32_t k = 0; k < n_dims; ++k) {
            ne[k] = (int64_t)file.read_u32();
        }
        // 清空并重新设置名称缓冲区的大小
        name_buf.clear();
        name_buf.resize(namelen + 1, '\0');
        // 读取名称数据到名称缓冲区
        file.read_raw(name_buf.data(), namelen);
        // 移动文件指针到下一个32位边界
        file.seek((0-file.tell()) & 31, SEEK_CUR);
        // 存储当前文件指针位置
        size_t offset = file.tell();
        // 创建一个新的张量对象
        struct ggml_tensor * tensor = ggml_new_tensor(result->ctx, (enum ggml_type) type, n_dims, ne);
        // 设置张量对象的名称
        ggml_set_name(tensor, name_buf.data());
        // 计算张量对象数据占用的字节数
        size_t nbytes     = ggml_nbytes(tensor);
        // 计算张量对象数据占用的字节数（包括填充）
        size_t nbytes_pad = ggml_nbytes_pad(tensor);
        // 累加填充后的总字节数
        total_nbytes_pad += nbytes_pad;
        // 将张量对象添加到张量列表中
        tensors.push_back(tensor);
        // 将当前张量对象的偏移量添加到偏移量列表中
        tensors_offset.push_back(offset);
        // 移动文件指针到下一个张量对象的位置
        file.seek(nbytes, SEEK_CUR);
    }
    // 读取张量数据
    result->data.resize(total_nbytes_pad);
    // 初始化数据偏移量
    size_t data_offset = 0;
    // 遍历张量列表
    for (size_t i = 0; i < tensors.size(); ++i) {
        // 获取当前张量对象
        struct ggml_tensor * tensor = tensors[i];
        // 获取当前张量对象的偏移量
        size_t offset     = tensors_offset[i];
        // 获取当前张量对象的数据占用字节数
        size_t nbytes     = ggml_nbytes(tensor);
        // 获取当前张量对象的数据占用字节数（包括填充）
        size_t nbytes_pad = ggml_nbytes_pad(tensor);
        // 将文件指针移动到当前张量对象的位置
        file.seek(offset, SEEK_SET);
        // 设置张量对象的数据指针
        tensor->data = result->data.data() + data_offset;
        // 读取文件中的数据到张量对象的数据指针
        file.read_raw(tensor->data, nbytes);
        // 更新数据偏移量
        data_offset += nbytes_pad;
    }
    // 返回结果
    return result;
// 构建基于 LoRa 的计算图
static struct ggml_cgraph * build_graph_lora(
    struct ggml_context * ctx,  // 上下文对象
    struct ggml_tensor * tensor,  // 输入张量
    struct ggml_tensor * lora_a,  // LoRa 张量 A
    struct ggml_tensor * lora_b,  // LoRa 张量 B
    float scaling  // 缩放因子
) {
    // 计算 LoRa 张量 A 和 B 的乘积
    struct ggml_tensor * ab = ggml_mul_mat(ctx, lora_a, lora_b);
    // 如果缩放因子不为 1.0，则对乘积张量进行缩放
    if (scaling != 1.0f) {
        ab = ggml_scale(ctx, ab, ggml_new_f32(ctx, scaling));
    }
    // 将乘积张量与输入张量相加，结果保存在 res 中
    struct ggml_tensor * res = ggml_add_inplace(ctx, tensor, ab);

    // 创建新的计算图对象
    struct ggml_cgraph * gf = ggml_new_graph(ctx);
    // 构建前向扩展计算图
    ggml_build_forward_expand (gf, res);
    // 返回构建好的计算图对象
    return gf;
}

// 应用 LoRa 计算
static bool apply_lora(struct ggml_tensor * tensor, struct lora_data * lora, int n_threads) {
    // 如果 LoRa 上下文对象为空，则返回 false
    if (lora->ctx == NULL) {
        return false;
    }
    // 获取输入张量的名称
    std::string name = ggml_get_name(tensor);
    // 构建 LoRa 张量 A 和 B 的名称
    std::string name_a = name + std::string(".loraA");
    std::string name_b = name + std::string(".loraB");
    // 根据名称获取 LoRa 张量 A 和 B
    struct ggml_tensor * lora_a = ggml_get_tensor(lora->ctx, name_a.c_str());
    struct ggml_tensor * lora_b = ggml_get_tensor(lora->ctx, name_b.c_str());
    // 如果获取的 LoRa 张量 A 或 B 为空，则返回 false
    if (lora_a == NULL || lora_b == NULL) {
        return false;
    }

    // 计算缩放因子
    float scaling = lora->info.scale * (float)lora->lora_alpha / (float)lora->lora_r;

    // 初始化参数
    struct ggml_init_params params;
    params.mem_size   = GGML_OBJECT_SIZE + ggml_graph_overhead() + ggml_tensor_overhead()*4 + GGML_MEM_ALIGN*5;
    params.mem_buffer = NULL;
    params.no_alloc   = true;
    struct ggml_context * ctx = NULL;
    struct ggml_allocr * alloc = NULL;
    struct ggml_cgraph * gf = NULL;

    // 初始化上下文对象
    ctx   = ggml_init(params);
    // 创建内存分配器
    alloc = ggml_allocr_new_measure(tensor_alignment);
    // 构建 LoRa 计算图
    gf    = build_graph_lora(ctx, tensor, lora_a, lora_b, scaling);
    // 分配计算图所需的内存
    size_t alloc_size = ggml_allocr_alloc_graph(alloc, gf);
    // 释放内存分配器
    ggml_allocr_free(alloc);
    // 释放上下文对象
    ggml_free(ctx);

    // 创建用于计算的数据缓冲区
    static std::vector<uint8_t> data_compute;
    data_compute.resize(alloc_size + tensor_alignment);

    // 重新初始化上下文对象
    ctx   = ggml_init(params);
    // 创建内存分配器，并指定数据缓冲区和对齐方式
    alloc = ggml_allocr_new(data_compute.data(), data_compute.size(), tensor_alignment);
    # 使用给定的上下文、张量、LORA参数和缩放因子构建图形
    gf    = build_graph_lora(ctx, tensor, lora_a, lora_b, scaling);
    # 将构建的图形分配给分配器
    ggml_allocr_alloc_graph(alloc, gf);
    # 释放分配器
    ggml_allocr_free(alloc);

    # 根据图形和线程数创建计划
    struct ggml_cplan cplan = ggml_graph_plan(gf, n_threads);
    # 创建静态的存储工作数据的向量
    static std::vector<uint8_t> data_work;
    # 调整工作数据的大小以适应计划中的工作大小
    data_work.resize(cplan.work_size);
    # 将工作数据指针指向向量的数据
    cplan.work_data = data_work.data();

    # 计算图形
    ggml_graph_compute(gf, &cplan);

    # 释放上下文资源
    ggml_free(ctx);
    # 返回 true
    return true;
// 导出 lora 数据到指定参数中
static void export_lora(struct export_lora_params * params) {
    // 加载所有的 lora 数据
    std::vector<struct lora_data *> loras;
    for (size_t i = 0; i < params->lora.size(); ++i) {
        // 加载 lora 数据并存入 loras 中
        struct lora_data * lora = load_lora(&params->lora[i]);
        if (lora != NULL) {
            loras.push_back(lora);
        }
    }
    // 如果没有 lora 数据，则输出警告信息
    if (loras.size() == 0) {
        fprintf(stderr, "warning: no lora adapters will be applied.\n");
    }

    // 打开输入文件
    struct llama_file fin(params->fn_model_base.c_str(), "rb");
    // 如果文件打开失败，则输出错误信息并退出程序
    if (!fin.fp) {
        die_fmt("Could not open file '%s'\n", params->fn_model_base.c_str());
    }

    // 打开基础模型 gguf，读取张量但不读取数据
    struct ggml_context * ctx_in;
    struct gguf_init_params params_gguf;
    params_gguf.no_alloc = true;
    params_gguf.ctx      = &ctx_in;
    struct gguf_context * gguf_in = gguf_init_from_file(params->fn_model_base.c_str(), params_gguf);

    // 创建新的 gguf
    struct gguf_context * gguf_out = gguf_init_empty();

    // 从基础模型复制元数据：kv 和张量
    gguf_set_kv(gguf_out, gguf_in);
    int n_tensors = gguf_get_n_tensors(gguf_in);
    for (int i=0; i < n_tensors; ++i) {
        const char * name = gguf_get_tensor_name(gguf_in, i);
        struct ggml_tensor * tensor = ggml_get_tensor(ctx_in, name);
        gguf_add_tensor(gguf_out, tensor);
    }

    // 创建输出文件
    struct llama_file fout(params->fn_model_out.c_str(), "wb");
    // 如果文件创建失败，则输出错误信息并退出程序
    if (!fout.fp) {
        die_fmt("Could not create file '%s'\n", params->fn_model_out.c_str());
    }

    // 写入 gguf 元数据
    std::vector<uint8_t> meta;
    meta.resize(gguf_get_meta_size(gguf_out));
    gguf_get_meta_data(gguf_out, meta.data());
    fout.write_raw(meta.data(), meta.size());

    // 初始化数据和填充向量
    std::vector<uint8_t> data;
    std::vector<uint8_t> padding;
}
    // 遍历所有张量
    for (int i=0; i < n_tensors; ++i) {
        // 获取张量名称
        const char * name = gguf_get_tensor_name(gguf_in, i);
        // 根据名称获取张量对象
        struct ggml_tensor * tensor = ggml_get_tensor(ctx_in, name);

        // 读取张量数据
        data.resize(ggml_nbytes(tensor)); // 调整数据容器大小
        tensor->data = data.data(); // 设置张量数据指针
        size_t offset = gguf_get_tensor_offset(gguf_in, i); // 获取张量在文件中的偏移量
        fin.seek(offset + meta.size(), SEEK_SET); // 定位到数据起始位置
        fin.read_raw(data.data(), data.size()); // 读取数据

        // 应用所有的 loras
        for (size_t k = 0; k < loras.size(); ++k) {
            apply_lora(tensor, loras[k], params->n_threads); // 应用 lora
        }

        // 写入张量数据 + 填充
        padding.clear(); // 清空填充数据
        padding.resize(GGML_PAD(data.size(), gguf_get_alignment(gguf_out)) - data.size(), 0); // 调整填充大小
        GGML_ASSERT(fout.tell() == offset + meta.size()); // 断言文件指针位置
        // fout.seek(offset + meta.size(), SEEK_SET); // 定位到数据起始位置
        fout.write_raw(data.data(), data.size()); // 写入数据
        fout.write_raw(padding.data(), padding.size()); // 写入填充数据

        // 打印进度
        if (i % 2 == 0) {
            printf(".");
        }
    }
    printf("\n");

    // 关闭 gguf
    gguf_free(gguf_out); // 释放输出 gguf
    gguf_free(gguf_in); // 释放输入 gguf

    // 释放 loras
    for (size_t i = 0; i < loras.size(); ++i) {
        free_lora(loras[i]); // 释放 lora
    }
# 主函数，程序的入口
int main(int argc, char ** argv) {
    # 创建一个结构体变量params，并初始化为默认的导出Lora参数
    struct export_lora_params params = get_default_export_lora_params();

    # 如果命令行参数解析失败，则返回1
    if (!export_lora_params_parse(argc, argv, &params)) {
        return 1;
    }

    # 调用导出Lora函数，传入参数params
    export_lora(&params);

    # 返回0，表示程序正常结束
    return 0;
}
```