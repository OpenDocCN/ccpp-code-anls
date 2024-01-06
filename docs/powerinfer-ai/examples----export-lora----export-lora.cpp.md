# `PowerInfer\examples\export-lora\export-lora.cpp`

```
// 包含常用的头文件和自定义的头文件
#include "common.h"
#include "ggml.h"
#include "ggml-alloc.h"

// 包含标准库中的向量、字符串和线程头文件
#include <vector>
#include <string>
#include <thread>

// 定义张量的对齐方式为32
static const size_t tensor_alignment = 32;

// 定义存储LoRa信息的结构体
struct lora_info {
    std::string filename; // LoRa文件名
    float scale; // 缩放比例
};

// 定义导出LoRa参数的结构体
struct export_lora_params {
    std::string fn_model_base; // 模型基本文件名
    std::string fn_model_out; // 输出模型文件名
    std::vector<struct lora_info> lora; // 存储LoRa信息的向量
// 定义一个结构体，用于存储 LoRa 数据的信息
struct lora_data {
    // 存储 LoRa 信息的结构体
    struct lora_info     info;
    // 存储 LoRa 数据的字节流
    std::vector<uint8_t> data;
    // 存储 GGML 上下文的指针
    struct ggml_context * ctx;

    // 存储 LoRa 的接收值
    uint32_t lora_r;
    // 存储 LoRa 的 alpha 值
    uint32_t lora_alpha;
};

// 定义一个结构体，用于表示文件的信息
struct llama_file {
    // 使用 FILE * 类型的指针，以便在 mmap 时不需要重新打开文件
    FILE * fp;
    // 文件的大小
    size_t size;

    // 构造函数，用于打开文件并初始化结构体
    llama_file(const char * fname, const char * mode) {
        // 打开文件
        fp = std::fopen(fname, mode);
        // 如果文件打开失败
        if (fp == NULL) {
    // 初始化 size 变量为 0
    size = 0;
    // 如果文件不为空，则定位到文件末尾，获取文件大小，然后重新定位到文件开头
    } else {
        seek(0, SEEK_END);
        size = tell();
        seek(0, SEEK_SET);
    }
}

// 返回当前文件指针的位置
size_t tell() const {
    // 根据操作系统不同调用不同的函数获取文件指针位置
    #ifdef _WIN32
        __int64 ret = _ftelli64(fp);
    #else
        long ret = std::ftell(fp);
    #endif
    // 断言文件指针位置不为 -1，即获取文件指针位置不应该失败
    GGML_ASSERT(ret != -1); // this really shouldn't fail
    return (size_t) ret;
}

// 移动文件指针到指定位置
void seek(size_t offset, int whence) {
    // 根据操作系统不同调用不同的函数移动文件指针
    #ifdef _WIN32
#ifdef _WIN32
        // 在 Windows 平台下使用 _fseeki64 函数定位文件指针
        int ret = _fseeki64(fp, (__int64) offset, whence);
#else
        // 在其他平台下使用 std::fseek 函数定位文件指针
        int ret = std::fseek(fp, (long) offset, whence);
#endif
        // 断言文件指针定位成功
        GGML_ASSERT(ret == 0); // 相同作用
    }

    void read_raw(void * ptr, size_t size) {
        // 如果读取大小为 0，则直接返回
        if (size == 0) {
            return;
        }
        // 清空错误标志
        errno = 0;
        // 从文件中读取指定大小的数据到指定内存地址
        std::size_t ret = std::fread(ptr, size, 1, fp);
        // 如果发生读取错误，输出错误信息
        if (ferror(fp)) {
            die_fmt("read error: %s", strerror(errno));
        }
        // 如果读取的数据大小不符合预期，输出错误信息
        if (ret != 1) {
            die("unexpectedly reached end of file");
        }
    }
    // 读取一个 32 位无符号整数
    std::uint32_t read_u32() {
        std::uint32_t ret;
        read_raw(&ret, sizeof(ret)); // 读取指定长度的原始数据
        return ret;
    }

    // 读取指定长度的字符串
    std::string read_string(std::uint32_t len) {
        std::vector<char> chars(len); // 创建指定长度的字符向量
        read_raw(chars.data(), len); // 读取指定长度的原始数据
        return std::string(chars.data(), len); // 将字符向量转换为字符串
    }

    // 写入原始数据
    void write_raw(const void * ptr, size_t size) {
        if (size == 0) {
            return;
        }
        errno = 0;
        size_t ret = std::fwrite(ptr, size, 1, fp); // 写入指定长度的原始数据
        if (ret != 1) {
    // 使用错误信息和错误码生成错误消息并终止程序
    die_fmt("write error: %s", strerror(errno));
    // 写入一个 32 位无符号整数到文件
    void write_u32(std::uint32_t val) {
        write_raw(&val, sizeof(val));
    }
    // 检查文件指针的位置是否已经到达文件末尾
    bool eof() {
        return tell() >= size;
    }
    // 析构函数，关闭文件指针
    ~llama_file() {
        if (fp) {
            std::fclose(fp);
        }
    }
    // 返回默认的导出 LORA 参数
    static struct export_lora_params get_default_export_lora_params() {
// 创建一个名为result的export_lora_params结构体实例
struct export_lora_params result;
// 将fn_model_base和fn_model_out字段设置为空字符串
result.fn_model_base = "";
result.fn_model_out  = "";
// 将n_threads字段设置为默认线程数
result.n_threads = GGML_DEFAULT_N_THREADS;
// 返回result结构体实例
return result;
}

// 打印export_lora_params结构体实例的用法信息
static void export_lora_print_usage(int /*argc*/, char ** argv, const struct export_lora_params * params) {
    // 打印程序的用法信息
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    // 打印各种选项的说明
    fprintf(stderr, "  -h, --help                         show this help message and exit\n");
    fprintf(stderr, "  -m FNAME, --model-base FNAME       model path from which to load base model (default '%s')\n", params->fn_model_base.c_str());
    fprintf(stderr, "  -o FNAME, --model-out FNAME        path to save exported model (default '%s')\n", params->fn_model_out.c_str());
    fprintf(stderr, "  -l FNAME, --lora FNAME             apply LoRA adapter\n");
    fprintf(stderr, "  -s FNAME S, --lora-scaled FNAME S  apply LoRA adapter with user defined scaling S\n");
    fprintf(stderr, "  -t N, --threads N                  number of threads to use during computation (default: %d)\n", params->n_threads);
}

// 解析命令行参数并将结果存储到export_lora_params结构体实例中
static bool export_lora_params_parse(int argc, char ** argv, struct export_lora_params * params) {
    // 定义一个布尔变量，用于标记参数是否有效
    bool invalid_param = false;
    // 定义一个字符串变量，用于存储参数
    std::string arg;
    // 获取默认的导出参数
    struct export_lora_params default_params = get_default_export_lora_params();
    // 定义一个字符串变量，用于存储参数前缀
    const std::string arg_prefix = "--";

    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数
        arg = argv[i];
        // 检查参数是否以指定前缀开头，如果是则替换其中的下划线为破折号
        if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
            std::replace(arg.begin(), arg.end(), '_', '-');
        }

        // 检查参数是否为模型基础路径的参数
        if (arg == "-m" || arg == "--model-base") {
            // 检查下一个参数是否存在
            if (++i >= argc) {
                // 如果不存在，则标记参数无效并退出循环
                invalid_param = true;
                break;
            }
            // 将下一个参数作为模型基础路径
            params->fn_model_base = argv[i];
        } else if (arg == "-o" || arg == "--model-out") {
            // 检查下一个参数是否存在
            if (++i >= argc) {
                // 如果不存在，则标记参数无效并退出循环
                invalid_param = true;
            // 如果参数是"-o"或"--output"，则将下一个参数作为输出文件名
            if (arg == "-o" || arg == "--output") {
                // 检查是否还有足够的参数
                if (++i >= argc) {
                    invalid_param = true;
                    break;
                }
                // 将下一个参数作为输出文件名
                params->fn_model_out = argv[i];
            } 
            // 如果参数是"-l"或"--lora"，则将下一个参数作为文件名，并创建一个lora_info结构体
            else if (arg == "-l" || arg == "--lora") {
                // 检查是否还有足够的参数
                if (++i >= argc) {
                    invalid_param = true;
                    break;
                }
                // 创建一个lora_info结构体，设置文件名和默认比例
                struct lora_info lora;
                lora.filename = argv[i];
                lora.scale = 1.0f;
                // 将lora_info结构体添加到params->lora向量中
                params->lora.push_back(lora);
            } 
            // 如果参数是"-s"或"--lora-scaled"，则将下一个参数作为文件名，并创建一个lora_info结构体
            else if (arg == "-s" || arg == "--lora-scaled") {
                // 检查是否还有足够的参数
                if (++i >= argc) {
                    invalid_param = true;
                    break;
                }
                // 创建一个lora_info结构体，设置文件名
                struct lora_info lora;
                lora.filename = argv[i];
                // 检查是否还有足够的参数
                if (++i >= argc) {
                    // 如果没有足够的参数，设置invalid_param为true并跳出循环
                    invalid_param = true;
                    break;
                }
            // 如果参数无效，设置标志为true并跳出循环
            invalid_param = true;
            break;
        }
        // 如果参数是"-s"或"--scale"
        if (arg == "-s" || arg == "--scale") {
            // 如果下一个参数不存在，设置标志为true并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将参数转换为浮点数，并将其添加到参数对象的lora属性中
            lora.scale = std::stof(argv[i]);
            params->lora.push_back(lora);
        } 
        // 如果参数是"-t"或"--threads"
        else if (arg == "-t" || arg == "--threads") {
            // 如果下一个参数不存在，设置标志为true并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将参数转换为整数，并将其赋值给参数对象的n_threads属性
            params->n_threads = std::stoi(argv[i]);
            // 如果线程数小于等于0，将其设置为硬件并发数
            if (params->n_threads <= 0) {
                params->n_threads = std::thread::hardware_concurrency();
            }
        } 
        // 如果参数不是以上任何一种情况
        else {
            // 输出错误信息并打印用法
            fprintf(stderr, "error: unknown argument: '%s'\n", arg.c_str());
            export_lora_print_usage(argc, argv, &default_params);
            // 退出程序
            exit(1);
        }
    }
# 检查是否指定了模型基础文件名，如果没有则输出错误信息并退出程序
if (params->fn_model_base == default_params.fn_model_base) {
    fprintf(stderr, "error: please specify a filename for model-base.\n");
    export_lora_print_usage(argc, argv, &default_params);
    exit(1);
}
# 检查是否指定了模型输出文件名，如果没有则输出错误信息并退出程序
if (params->fn_model_out == default_params.fn_model_out) {
    fprintf(stderr, "error: please specify a filename for model-out.\n");
    export_lora_print_usage(argc, argv, &default_params);
    exit(1);
}
# 检查是否存在无效参数，如果有则输出错误信息并退出程序
if (invalid_param) {
    fprintf(stderr, "error: invalid parameter for argument: '%s'\n", arg.c_str());
    export_lora_print_usage(argc, argv, &default_params);
    exit(1);
}
# 返回 true 表示执行成功
return true;
}

# 释放 lora_data 结构体占用的内存空间
static void free_lora(struct lora_data * lora) {
    // 检查 lora->ctx 是否为空，如果不为空则释放其内存
    if (lora->ctx != NULL) {
        ggml_free(lora->ctx);
    }
    // 释放 lora 对象的内存
    delete lora;
}

// 加载 lora 数据
static struct lora_data * load_lora(struct lora_info * info) {
    // 创建一个新的 lora_data 结构体对象
    struct lora_data * result = new struct lora_data;
    // 将传入的 lora_info 对象复制给 result->info
    result->info = *info;
    // 将 result->ctx 初始化为 NULL
    result->ctx = NULL;
    // 将 result->lora_r 初始化为 1
    result->lora_r     = 1;
    // 将 result->lora_alpha 初始化为 1
    result->lora_alpha = 1;

    // 使用 llama_file 打开 lora 文件
    struct llama_file file(info->filename.c_str(), "rb");
    // 如果文件指针为空，打印警告信息并释放 result 的内存，然后返回 NULL
    if (file.fp == NULL) {
        fprintf(stderr, "warning: Could not open lora adapter '%s'. Ignoring this adapter.\n",
            info->filename.c_str());
        free_lora(result);
        return NULL;
    }
    // 初始化 GGML 参数结构体
    struct ggml_init_params params_ggml;
    // 计算所需内存大小并赋值给参数结构体
    params_ggml.mem_size   = ggml_tensor_overhead() * GGML_DEFAULT_GRAPH_SIZE;
    // 初始化内存缓冲区为 NULL
    params_ggml.mem_buffer = NULL;
    // 设置不进行内存分配
    params_ggml.no_alloc   = true;
    // 使用参数结构体初始化 GGML 上下文
    result->ctx = ggml_init(params_ggml);

    // 定义并初始化文件魔数 LLAMA_FILE_MAGIC_LORA
    uint32_t LLAMA_FILE_MAGIC_LORA = 0x67676C61; // 'ggla'
    // 从文件中读取魔数
    uint32_t magic   = file.read_u32();
    // 检查读取的魔数是否与预期值相等，若不相等则输出错误信息
    if (magic != LLAMA_FILE_MAGIC_LORA) {
        die_fmt("unexpected lora header file magic in '%s'", info->filename.c_str());
    }
    // 从文件中读取版本号
    uint32_t version = file.read_u32();
    // 检查读取的版本号是否为1，若不是则输出错误信息
    if (version != 1) {
        die_fmt("unexpected lora file version '%u' in '%s'", (unsigned) version, info->filename.c_str());
    }
    // 从文件中读取 lora_r 和 lora_alpha 的值
    result->lora_r     = file.read_u32();
    result->lora_alpha = file.read_u32();
    // 从文件中读取张量信息并存储在 name_buf 中
    std::vector<char> name_buf;
    # 创建一个存储 ggml_tensor 结构体指针的向量
    std::vector<struct ggml_tensor *> tensors;
    # 创建一个存储大小为 size_t 的向量，用于存储张量的偏移量
    std::vector<size_t> tensors_offset;
    # 初始化总字节数为 0
    size_t total_nbytes_pad = 0;
    # 当文件未到达末尾时循环
    while(!file.eof()) {
        # 初始化一个长度为 4 的整型数组，元素值为 1
        int64_t ne[4]   = {1,1,1,1};
        # 读取 32 位无符号整数，表示张量的维度
        uint32_t n_dims  = file.read_u32();
        # 读取 32 位无符号整数，表示张量名称的长度
        uint32_t namelen = file.read_u32();
        # 读取 32 位无符号整数，表示张量的类型
        uint32_t type    = file.read_u32();
        # 循环读取张量的每个维度值
        for (uint32_t k = 0; k < n_dims; ++k) {
            ne[k] = (int64_t)file.read_u32();
        }
        # 清空名称缓冲区，并设置其大小为名称长度加 1，填充 '\0'
        name_buf.clear();
        name_buf.resize(namelen + 1, '\0');
        # 读取名称数据到名称缓冲区
        file.read_raw(name_buf.data(), namelen);
        # 移动文件指针到下一个 32 字节对齐的位置
        file.seek((0-file.tell()) & 31, SEEK_CUR);
        # 记录当前文件指针位置为张量的偏移量
        size_t offset = file.tell();
        # 创建一个新的 ggml_tensor 结构体指针，并初始化其属性
        struct ggml_tensor * tensor = ggml_new_tensor(result->ctx, (enum ggml_type) type, n_dims, ne);
        # 设置张量的名称
        ggml_set_name(tensor, name_buf.data());
        # 计算张量占用的字节数
        size_t nbytes     = ggml_nbytes(tensor);
        # 计算张量填充后占用的字节数
        size_t nbytes_pad = ggml_nbytes_pad(tensor);
    // 计算总的填充字节数
    total_nbytes_pad += nbytes_pad;
    // 将张量添加到张量列表中
    tensors.push_back(tensor);
    // 将偏移量添加到偏移量列表中
    tensors_offset.push_back(offset);
    // 移动文件指针到下一个张量的位置
    file.seek(nbytes, SEEK_CUR);
    // 读取张量数据
    result->data.resize(total_nbytes_pad);
    // 设置数据偏移量为0
    size_t data_offset = 0;
    // 遍历张量列表
    for (size_t i = 0; i < tensors.size(); ++i) {
        // 获取当前张量
        struct ggml_tensor * tensor = tensors[i];
        // 获取当前张量的偏移量
        size_t offset     = tensors_offset[i];
        // 获取当前张量的字节数
        size_t nbytes     = ggml_nbytes(tensor);
        // 获取当前张量的填充字节数
        size_t nbytes_pad = ggml_nbytes_pad(tensor);
        // 移动文件指针到当前张量的位置
        file.seek(offset, SEEK_SET);
        // 设置当前张量的数据指针
        tensor->data = result->data.data() + data_offset;
        // 从文件中读取张量数据
        file.read_raw(tensor->data, nbytes);
        // 更新数据偏移量
        data_offset += nbytes_pad;
    }
    // 返回结果
    return result;
}
# 构建 LoRa 图形
static struct ggml_cgraph * build_graph_lora(
    struct ggml_context * ctx,  # 上下文对象
    struct ggml_tensor * tensor,  # 张量对象
    struct ggml_tensor * lora_a,  # LoRa A 张量对象
    struct ggml_tensor * lora_b,  # LoRa B 张量对象
    float scaling  # 缩放因子
) {
    # 计算 LoRa A 和 LoRa B 的乘积
    struct ggml_tensor * ab = ggml_mul_mat(ctx, lora_a, lora_b);
    # 如果缩放因子不为1.0，则对乘积进行缩放
    if (scaling != 1.0f) {
        ab = ggml_scale(ctx, ab, ggml_new_f32(ctx, scaling));
    }
    # 将乘积与输入张量相加
    struct ggml_tensor * res = ggml_add_inplace(ctx, tensor, ab);

    # 创建新的计算图对象
    struct ggml_cgraph * gf = ggml_new_graph(ctx);
    # 构建前向扩展
    ggml_build_forward_expand (gf, res);
    # 返回计算图对象
    return gf;
}
// 应用 LORA 算法到给定的张量数据上，使用指定的线程数
static bool apply_lora(struct ggml_tensor * tensor, struct lora_data * lora, int n_threads) {
    // 如果 LORA 上下文为空，则返回 false
    if (lora->ctx == NULL) {
        return false;
    }
    // 获取张量的名称
    std::string name = ggml_get_name(tensor);
    // 构造 loraA 和 loraB 的名称
    std::string name_a = name + std::string(".loraA");
    std::string name_b = name + std::string(".loraB");
    // 获取 loraA 和 loraB 对应的张量
    struct ggml_tensor * lora_a = ggml_get_tensor(lora->ctx, name_a.c_str());
    struct ggml_tensor * lora_b = ggml_get_tensor(lora->ctx, name_b.c_str());
    // 如果 loraA 或 loraB 为空，则返回 false
    if (lora_a == NULL || lora_b == NULL) {
        return false;
    }
    // 计算缩放因子
    float scaling = lora->info.scale * (float)lora->lora_alpha / (float)lora->lora_r;
    // 初始化参数结构体
    struct ggml_init_params params;
    params.mem_size   = GGML_OBJECT_SIZE + ggml_graph_overhead() + ggml_tensor_overhead()*4 + GGML_MEM_ALIGN*5;
    params.mem_buffer = NULL;
    params.no_alloc   = true;
    // 初始化上下文为空
    struct ggml_context * ctx = NULL;
    // 声明并初始化指针变量 alloc 和 gf
    struct ggml_allocr * alloc = NULL;
    struct ggml_cgraph * gf = NULL;

    // 初始化上下文 ctx
    ctx   = ggml_init(params);
    // 创建一个新的分配器对象 alloc，用于分配内存
    alloc = ggml_allocr_new_measure(tensor_alignment);
    // 构建图形对象 gf
    gf    = build_graph_lora(ctx, tensor, lora_a, lora_b, scaling);
    // 分配内存给图形对象，并返回分配的内存大小
    size_t alloc_size = ggml_allocr_alloc_graph(alloc, gf);
    // 释放分配器对象 alloc 占用的内存
    ggml_allocr_free(alloc);
    // 释放上下文 ctx 占用的内存
    ggml_free(ctx);

    // 声明并初始化静态向量 data_compute，并设置其大小为 alloc_size + tensor_alignment
    static std::vector<uint8_t> data_compute;
    data_compute.resize(alloc_size + tensor_alignment);

    // 重新初始化上下文 ctx
    ctx   = ggml_init(params);
    // 创建一个新的分配器对象 alloc，用于分配 data_compute 的内存
    alloc = ggml_allocr_new(data_compute.data(), data_compute.size(), tensor_alignment);
    // 重新构建图形对象 gf
    gf    = build_graph_lora(ctx, tensor, lora_a, lora_b, scaling);
    // 分配内存给图形对象
    ggml_allocr_alloc_graph(alloc, gf);
    // 释放分配器对象 alloc 占用的内存
    ggml_allocr_free(alloc);

    // 创建一个图形计划对象 cplan
    struct ggml_cplan cplan = ggml_graph_plan(gf, n_threads);
    // 创建一个静态的存储 uint8_t 类型数据的向量，大小为 cplan.work_size
    static std::vector<uint8_t> data_work;
    data_work.resize(cplan.work_size);
    // 将向量的数据指针赋值给 cplan.work_data
    cplan.work_data = data_work.data();

    // 调用 ggml_graph_compute 函数进行图计算
    ggml_graph_compute(gf, &cplan);

    // 释放上下文资源
    ggml_free(ctx);
    // 返回 true
    return true;
}

// 导出 lora 数据
static void export_lora(struct export_lora_params * params) {
    // 加载所有的 lora 数据
    std::vector<struct lora_data *> loras;
    for (size_t i = 0; i < params->lora.size(); ++i) {
        // 调用 load_lora 函数加载 lora 数据
        struct lora_data * lora = load_lora(&params->lora[i]);
        // 如果加载成功，则将 lora 数据加入 loras 向量中
        if (lora != NULL) {
            loras.push_back(lora);
        }
    }
    // 如果没有加载到任何 lora 数据
    if (loras.size() == 0) {
    // 输出警告信息，表示不会应用任何 lora 适配器
    fprintf(stderr, "warning: no lora adapters will be applied.\n");

    // 打开输入文件
    // 使用 "rb" 模式打开文件，如果打开失败则输出错误信息
    struct llama_file fin(params->fn_model_base.c_str(), "rb");
    if (!fin.fp) {
        die_fmt("Could not open file '%s'\n", params->fn_model_base.c_str());
    }

    // 打开基础模型 gguf，读取张量但不读取数据
    // 初始化 gguf_init_params 结构体
    struct ggml_context * ctx_in;
    struct gguf_init_params params_gguf;
    params_gguf.no_alloc = true;
    params_gguf.ctx      = &ctx_in;
    // 从文件中初始化 gguf_context 结构体
    struct gguf_context * gguf_in = gguf_init_from_file(params->fn_model_base.c_str(), params_gguf);

    // 创建新的 gguf
    // 初始化一个空的 gguf_context 结构体
    struct gguf_context * gguf_out = gguf_init_empty();

    // 从基础模型复制元数据：键值对和张量
    // 将输入 gguf_in 的键值对复制到输出 gguf_out
    gguf_set_kv(gguf_out, gguf_in);
    // 获取输入 gguf_in 中张量的数量
    int n_tensors = gguf_get_n_tensors(gguf_in);
    // 遍历输入 gguf_in 中的张量
    for (int i=0; i < n_tensors; ++i) {
        // 获取第 i 个张量的名称
        const char * name = gguf_get_tensor_name(gguf_in, i);
        // 根据名称从输入 ctx_in 中获取张量
        struct ggml_tensor * tensor = ggml_get_tensor(ctx_in, name);
        // 将获取的张量添加到输出 gguf_out 中
        gguf_add_tensor(gguf_out, tensor);
    }

    // 创建输出文件
    struct llama_file fout(params->fn_model_out.c_str(), "wb");
    // 如果无法创建文件，则输出错误信息并退出程序
    if (!fout.fp) {
        die_fmt("Could not create file '%s'\n", params->fn_model_out.c_str());
    }

    // 写入 gguf 的元数据到输出文件
    std::vector<uint8_t> meta;
    // 调整 meta 的大小以容纳 gguf_out 的元数据
    meta.resize(gguf_get_meta_size(gguf_out));
    // 获取 gguf_out 的元数据并写入到 meta 中
    gguf_get_meta_data(gguf_out, meta.data());
    // 将 meta 中的数据写入到输出文件中
    fout.write_raw(meta.data(), meta.size());
    // 创建存储数据的向量
    std::vector<uint8_t> data;
    // 创建存储填充数据的向量
    std::vector<uint8_t> padding;
    // 遍历张量数量
    for (int i=0; i < n_tensors; ++i) {
        // 获取张量名称
        const char * name = gguf_get_tensor_name(gguf_in, i);
        // 获取张量数据
        struct ggml_tensor * tensor = ggml_get_tensor(ctx_in, name);

        // 读取张量数据
        data.resize(ggml_nbytes(tensor)); // 调整数据向量大小
        tensor->data = data.data(); // 将数据向量的指针赋给张量的数据指针
        size_t offset = gguf_get_tensor_offset(gguf_in, i); // 获取张量在文件中的偏移量
        fin.seek(offset + meta.size(), SEEK_SET); // 移动文件指针到对应位置
        fin.read_raw(data.data(), data.size()); // 读取数据到数据向量

        // 应用所有的 loras
        for (size_t k = 0; k < loras.size(); ++k) {
            apply_lora(tensor, loras[k], params->n_threads); // 对张量应用 lora
        }

        // 写入张量数据和填充数据
        padding.clear(); // 清空填充数据向量
        // 根据输出对齐方式和数据大小调整填充数据大小
        padding.resize(GGML_PAD(data.size(), gguf_get_alignment(gguf_out)) - data.size(), 0);

        // 确保输出文件指针位置正确
        GGML_ASSERT(fout.tell() == offset + meta.size());
        // 将数据写入输出文件
        fout.write_raw(data.data(), data.size());
        // 将填充数据写入输出文件
        fout.write_raw(padding.data(), padding.size());

        // 如果循环次数为偶数，打印一个点
        if (i % 2 == 0) {
            printf(".");
        }
    }
    // 打印换行
    printf("\n");

    // 释放输出 GGUF 对象
    gguf_free(gguf_out);
    // 释放输入 GGUF 对象
    gguf_free(gguf_in);

    // 释放 Loras 对象
    for (size_t i = 0; i < loras.size(); ++i) {
        free_lora(loras[i]);
// 结束 main 函数的定义
    }
}

// 主函数
int main(int argc, char ** argv) {
    // 获取默认的导出 LoRa 参数
    struct export_lora_params params = get_default_export_lora_params();

    // 解析命令行参数，如果解析失败则返回 1
    if (!export_lora_params_parse(argc, argv, &params)) {
        return 1;
    }

    // 导出 LoRa 数据
    export_lora(&params);

    // 返回 0 表示程序正常结束
    return 0;
}
```