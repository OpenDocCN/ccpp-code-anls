# `PowerInfer\examples\gguf\gguf.cpp`

```
#include "ggml.h"
#include "llama.h"

#include <cstdio>
#include <cinttypes>
#include <string>
#include <sstream>
#include <fstream>
#include <vector>

// 取消之前定义的 MIN 和 MAX 宏
#undef MIN
#undef MAX
// 定义 MIN 和 MAX 宏，用于返回两个值中的最小值和最大值
#define MIN(a, b) ((a) < (b) ? (a) : (b))
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 将任意类型的值转换为字符串
template <typename T>
static std::string to_string(const T & val) {
    std::stringstream ss;
    ss << val;
    return ss.str();
}

// 写入 gguf 文件
static bool gguf_ex_write(const std::string & fname) {
    // 初始化 gguf 上下文
    struct gguf_context * ctx = gguf_init_empty();

    // 设置不同类型的参数值
    gguf_set_val_u8  (ctx, "some.parameter.uint8",    0x12);
    gguf_set_val_i8  (ctx, "some.parameter.int8",    -0x13);
    gguf_set_val_u16 (ctx, "some.parameter.uint16",   0x1234);
    gguf_set_val_i16 (ctx, "some.parameter.int16",   -0x1235);
    gguf_set_val_u32 (ctx, "some.parameter.uint32",   0x12345678);
    gguf_set_val_i32 (ctx, "some.parameter.int32",   -0x12345679);
    gguf_set_val_f32 (ctx, "some.parameter.float32",  0.123456789f);
    gguf_set_val_u64 (ctx, "some.parameter.uint64",   0x123456789abcdef0ull);
    gguf_set_val_i64 (ctx, "some.parameter.int64",   -0x123456789abcdef1ll);
    gguf_set_val_f64 (ctx, "some.parameter.float64",  0.1234567890123456789);
    gguf_set_val_bool(ctx, "some.parameter.bool",     true);
    gguf_set_val_str (ctx, "some.parameter.string",   "hello world");

    // 设置数组类型的参数值
    gguf_set_arr_data(ctx, "some.parameter.arr.i16", GGUF_TYPE_INT16,   std::vector<int16_t>{ 1, 2, 3, 4, }.data(), 4);
    gguf_set_arr_data(ctx, "some.parameter.arr.f32", GGUF_TYPE_FLOAT32, std::vector<float>{ 3.145f, 2.718f, 1.414f, }.data(), 3);
    gguf_set_arr_str (ctx, "some.parameter.arr.str",                    std::vector<const char *>{ "hello", "world", "!" }.data(), 3);

    // 初始化 ggml 上下文参数
    struct ggml_init_params params = {
        /*.mem_size   =*/ 128ull*1024ull*1024ull,
        /*.mem_buffer =*/ NULL,
        /*.no_alloc   =*/ false,
    };

    // 初始化 ggml 上下文
    struct ggml_context * ctx_data = ggml_init(params);

    const int n_tensors = 10;

    // tensor infos
    // 遍历 n_tensors 次，创建不同的张量
    for (int i = 0; i < n_tensors; ++i) {
        // 生成张量的名称，格式为 "tensor_" + i 的字符串
        const std::string name = "tensor_" + to_string(i);

        // 初始化张量每个维度的大小为 1
        int64_t ne[GGML_MAX_DIMS] = { 1 };
        // 随机生成张量的维度数量，范围为 1 到 GGML_MAX_DIMS
        int32_t n_dims = rand() % GGML_MAX_DIMS + 1;

        // 遍历张量的每个维度，随机生成维度的大小，范围为 1 到 10
        for (int j = 0; j < n_dims; ++j) {
            ne[j] = rand() % 10 + 1;
        }

        // 创建新的张量对象，数据类型为 GGML_TYPE_F32，维度数量为 n_dims，每个维度的大小为 ne
        struct ggml_tensor * cur = ggml_new_tensor(ctx_data, GGML_TYPE_F32, n_dims, ne);
        // 设置张量的名称
        ggml_set_name(cur, name.c_str());

        // 获取张量的数据指针，将每个元素的值设置为 100 + i
        {
            float * data = (float *) cur->data;
            for (int j = 0; j < ggml_nelements(cur); ++j) {
                data[j] = 100 + i;
            }
        }

        // 将创建的张量添加到上下文中
        gguf_add_tensor(ctx, cur);
    }

    // 将上下文中的数据写入到文件中，文件名为 fname
    gguf_write_to_file(ctx, fname.c_str(), false);

    // 打印函数名称和写入的文件名
    printf("%s: wrote file '%s;\n", __func__, fname.c_str());

    // 释放上下文中的数据
    ggml_free(ctx_data);
    // 释放上下文
    gguf_free(ctx);

    // 返回 true，表示函数执行成功
    return true;
// 仅仅读取张量信息
static bool gguf_ex_read_0(const std::string & fname) {
    // 初始化参数结构体，设置不分配内存，上下文为空
    struct gguf_init_params params = {
        /*.no_alloc = */ false,
        /*.ctx      = */ NULL,
    };

    // 从文件中初始化上下文
    struct gguf_context * ctx = gguf_init_from_file(fname.c_str(), params);

    // 打印版本号
    printf("%s: version:      %d\n", __func__, gguf_get_version(ctx));
    // 打印对齐方式
    printf("%s: alignment:   %zu\n", __func__, gguf_get_alignment(ctx));
    // 打印数据偏移量
    printf("%s: data offset: %zu\n", __func__, gguf_get_data_offset(ctx));

    // kv
    {
        // 获取键值对的数量
        const int n_kv = gguf_get_n_kv(ctx);

        // 打印键值对的数量
        printf("%s: n_kv: %d\n", __func__, n_kv);

        // 遍历键值对
        for (int i = 0; i < n_kv; ++i) {
            // 获取键
            const char * key = gguf_get_key(ctx, i);

            // 打印键值对的键
            printf("%s: kv[%d]: key = %s\n", __func__, i, key);
        }
    }

    // 查找键值对字符串
    {
        // 要查找的键
        const char * findkey = "some.parameter.string";

        // 查找键的索引
        const int keyidx = gguf_find_key(ctx, findkey);
        if (keyidx == -1) {
            // 如果未找到，打印未找到的信息
            printf("%s: find key: %s not found.\n", __func__, findkey);
        } else {
            // 如果找到，获取键值对的值并打印
            const char * key_value = gguf_get_val_str(ctx, keyidx);
            printf("%s: find key: %s found, kv[%d] value = %s\n", __func__, findkey, keyidx, key_value);
        }
    }

    // 张量信息
    {
        // 获取张量的数量
        const int n_tensors = gguf_get_n_tensors(ctx);

        // 打印张量的数量
        printf("%s: n_tensors: %d\n", __func__, n_tensors);

        // 遍历张量
        for (int i = 0; i < n_tensors; ++i) {
            // 获取张量的名称和偏移量
            const char * name   = gguf_get_tensor_name  (ctx, i);
            const size_t offset = gguf_get_tensor_offset(ctx, i);

            // 打印张量的名称和偏移量
            printf("%s: tensor[%d]: name = %s, offset = %zu\n", __func__, i, name, offset);
        }
    }

    // 释放上下文
    gguf_free(ctx);

    return true;
}

// 读取并创建包含张量及其数据的 ggml_context
static bool gguf_ex_read_1(const std::string & fname) {
    // 数据上下文初始化为空
    struct ggml_context * ctx_data = NULL;

    // 初始化参数结构体，设置不分配内存，上下文指向数据上下文
    struct gguf_init_params params = {
        /*.no_alloc = */ false,
        /*.ctx      = */ &ctx_data,
    };
    // 从文件名创建 gguf_context 结构体对象
    struct gguf_context * ctx = gguf_init_from_file(fname.c_str(), params);

    // 打印函数名和 gguf_context 结构体对象的版本号
    printf("%s: version:      %d\n", __func__, gguf_get_version(ctx));
    // 打印函数名和 gguf_context 结构体对象的对齐方式
    printf("%s: alignment:   %zu\n", __func__, gguf_get_alignment(ctx));
    // 打印函数名和 gguf_context 结构体对象的数据偏移量
    printf("%s: data offset: %zu\n", __func__, gguf_get_data_offset(ctx));

    // kv
    {
        // 获取 gguf_context 结构体对象中 kv 的数量
        const int n_kv = gguf_get_n_kv(ctx);

        // 打印函数名和 kv 的数量
        printf("%s: n_kv: %d\n", __func__, n_kv);

        // 遍历 kv 数组，打印每个 kv 的键值对
        for (int i = 0; i < n_kv; ++i) {
            // 获取第 i 个 kv 的键
            const char * key = gguf_get_key(ctx, i);

            // 打印函数名、kv 的索引和键
            printf("%s: kv[%d]: key = %s\n", __func__, i, key);
        }
    }

    // tensor info
    {
        // 获取 gguf_context 结构体对象中 tensor 的数量
        const int n_tensors = gguf_get_n_tensors(ctx);

        // 打印函数名和 tensor 的数量
        printf("%s: n_tensors: %d\n", __func__, n_tensors);

        // 遍历 tensor 数组，打印每个 tensor 的名称和偏移量
        for (int i = 0; i < n_tensors; ++i) {
            // 获取第 i 个 tensor 的名称
            const char * name   = gguf_get_tensor_name  (ctx, i);
            // 获取第 i 个 tensor 的偏移量
            const size_t offset = gguf_get_tensor_offset(ctx, i);

            // 打印函数名、tensor 的索引、名称和偏移量
            printf("%s: tensor[%d]: name = %s, offset = %zu\n", __func__, i, name, offset);
        }
    }

    // data
    {
        // 获取上下文中张量的数量
        const int n_tensors = gguf_get_n_tensors(ctx);

        // 遍历每个张量
        for (int i = 0; i < n_tensors; ++i) {
            // 打印正在读取张量数据的信息
            printf("%s: reading tensor %d data\n", __func__, i);

            // 获取当前张量的名称
            const char * name = gguf_get_tensor_name(ctx, i);

            // 获取当前张量的指针
            struct ggml_tensor * cur = ggml_get_tensor(ctx_data, name);

            // 打印当前张量的信息
            printf("%s: tensor[%d]: n_dims = %d, name = %s, data = %p\n", __func__, i, cur->n_dims, cur->name, cur->data);

            // 打印前10个元素的数据
            const float * data = (const float *) cur->data;

            printf("%s data[:10] : ");
            for (int j = 0; j < MIN(10, ggml_nelements(cur)); ++j) {
                printf("%f ", data[j]);
            }
            printf("\n\n");

            // 检查数据
            {
                const float * data = (const float *) cur->data;
                for (int j = 0; j < ggml_nelements(cur); ++j) {
                    // 如果数据不符合预期，打印错误信息并返回false
                    if (data[j] != 100 + i) {
                        fprintf(stderr, "%s: tensor[%d]: data[%d] = %f\n", __func__, i, j, data[j]);
                        return false;
                    }
                }
            }
        }
    }

    // 打印上下文数据的大小
    printf("%s: ctx_data size: %zu\n", __func__, ggml_get_mem_size(ctx_data));

    // 释放上下文数据
    ggml_free(ctx_data);
    gguf_free(ctx);

    // 返回true
    return true;
}
// 主函数，接受命令行参数
int main(int argc, char ** argv) {
    // 检查参数数量，如果少于3个，打印用法说明并返回错误码
    if (argc < 3) {
        printf("usage: %s data.gguf r|w\n", argv[0]);
        return -1;
    }

    // 从命令行参数中获取文件名和模式
    const std::string fname(argv[1]);
    const std::string mode (argv[2]);

    // 检查模式是否为'r'或'w'，如果不是，触发断言错误
    GGML_ASSERT((mode == "r" || mode == "w") && "mode must be r or w");

    // 如果模式为'w'，调用gguf_ex_write函数写入文件，并触发断言错误
    if (mode == "w") {
        GGML_ASSERT(gguf_ex_write(fname) && "failed to write gguf file");
    } 
    // 如果模式为'r'，分别调用gguf_ex_read_0和gguf_ex_read_1函数读取文件，并触发断言错误
    else if (mode == "r") {
        GGML_ASSERT(gguf_ex_read_0(fname) && "failed to read gguf file");
        GGML_ASSERT(gguf_ex_read_1(fname) && "failed to read gguf file");
    }

    // 返回成功码
    return 0;
}
```