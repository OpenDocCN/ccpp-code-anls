# `PowerInfer\examples\gguf\gguf.cpp`

```
// 包含头文件 ggml.h 和 llama.h
#include "ggml.h"
#include "llama.h"

// 包含标准输入输出头文件
#include <cstdio>
// 包含整数类型头文件
#include <cinttypes>
// 包含字符串头文件
#include <string>
// 包含字符串流头文件
#include <sstream>
// 包含文件流头文件
#include <fstream>
// 包含向量头文件
#include <vector>

// 取消定义 MIN 和 MAX 宏
#undef MIN
#undef MAX
// 定义 MIN 宏，返回两个值中较小的一个
#define MIN(a, b) ((a) < (b) ? (a) : (b))
// 定义 MAX 宏，返回两个值中较大的一个
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 模板函数，将输入的值转换为字符串
template <typename T>
static std::string to_string(const T & val) {
    // 创建字符串流
    std::stringstream ss;
    // 将值写入字符串流
    ss << val;
    // 返回字符串流中的字符串
    return ss.str();
// 定义一个静态函数，用于将数据写入指定的文件
static bool gguf_ex_write(const std::string & fname) {
    // 初始化一个 gguf 上下文
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
    // 设置参数数组的字符串值
    gguf_set_arr_str (ctx, "some.parameter.arr.str", std::vector<const char *>{ "hello", "world", "!" }.data(), 3);

    // 初始化参数结构体
    struct ggml_init_params params = {
        /*.mem_size   =*/ 128ull*1024ull*1024ull,  // 设置内存大小为128MB
        /*.mem_buffer =*/ NULL,  // 内存缓冲区为空
        /*.no_alloc   =*/ false,  // 允许分配内存
    };

    // 初始化上下文数据
    struct ggml_context * ctx_data = ggml_init(params);

    // 定义张量的数量
    const int n_tensors = 10;

    // 张量信息
    for (int i = 0; i < n_tensors; ++i) {
        // 生成张量名称
        const std::string name = "tensor_" + to_string(i);

        // 初始化张量维度
        int64_t ne[GGML_MAX_DIMS] = { 1 };
        int32_t n_dims = rand() % GGML_MAX_DIMS + 1;

        // 遍历张量维度
        for (int j = 0; j < n_dims; ++j) {
        // 为数组 ne 赋随机值
        for (int j = 0; j < n_dims; ++j) {
            ne[j] = rand() % 10 + 1;
        }

        // 创建新的张量对象 cur
        struct ggml_tensor * cur = ggml_new_tensor(ctx_data, GGML_TYPE_F32, n_dims, ne);
        // 设置张量对象的名称
        ggml_set_name(cur, name.c_str());

        // 为张量对象的数据赋值
        {
            float * data = (float *) cur->data;
            for (int j = 0; j < ggml_nelements(cur); ++j) {
                data[j] = 100 + i;
            }
        }

        // 将张量对象添加到上下文中
        gguf_add_tensor(ctx, cur);
    }

    // 将上下文中的数据写入文件
    gguf_write_to_file(ctx, fname.c_str(), false);

    // 打印写入文件的信息
    printf("%s: wrote file '%s;\n", __func__, fname.c_str());
    释放 ctx_data 内存
    ggml_free(ctx_data);
    释放 ctx 内存
    gguf_free(ctx);

    返回 true
    return true;
}

// 仅读取张量信息
static bool gguf_ex_read_0(const std::string & fname) {
    初始化 gguf_init_params 结构体
    struct gguf_init_params params = {
        /*.no_alloc = */ false,  // 不进行内存分配
        /*.ctx      = */ NULL,   // 上下文为空
    };

    从文件中初始化 gguf_context 结构体
    struct gguf_context * ctx = gguf_init_from_file(fname.c_str(), params);

    打印版本信息
    printf("%s: version:      %d\n", __func__, gguf_get_version(ctx));
    打印对齐信息
    printf("%s: alignment:   %zu\n", __func__, gguf_get_alignment(ctx));
    打印数据偏移量
    printf("%s: data offset: %zu\n", __func__, gguf_get_data_offset(ctx));

    // kv
    // 这里缺少对于 // kv 语句的上下文，无法确定其作用
    // 获取键值对的数量
    const int n_kv = gguf_get_n_kv(ctx);

    // 打印键值对的数量
    printf("%s: n_kv: %d\n", __func__, n_kv);

    // 遍历键值对
    for (int i = 0; i < n_kv; ++i) {
        // 获取第i个键的值
        const char * key = gguf_get_key(ctx, i);

        // 打印键值对的键
        printf("%s: kv[%d]: key = %s\n", __func__, i, key);
    }

    // 查找特定键的字符串
    {
        // 要查找的键
        const char * findkey = "some.parameter.string";

        // 查找键在键值对中的索引
        const int keyidx = gguf_find_key(ctx, findkey);
        // 如果找不到，打印未找到的消息
        if (keyidx == -1) {
            printf("%s: find key: %s not found.\n", __func__, findkey);
        } else {
// 获取指定索引处的键值对，并打印相关信息
const char * key_value = gguf_get_val_str(ctx, keyidx);
printf("%s: find key: %s found, kv[%d] value = %s\n", __func__, findkey, keyidx, key_value);
}

// 获取张量信息
{
    // 获取张量的数量并打印
    const int n_tensors = gguf_get_n_tensors(ctx);
    printf("%s: n_tensors: %d\n", __func__, n_tensors);

    // 遍历每个张量，获取张量的名称和偏移量，并打印相关信息
    for (int i = 0; i < n_tensors; ++i) {
        const char * name   = gguf_get_tensor_name  (ctx, i);
        const size_t offset = gguf_get_tensor_offset(ctx, i);
        printf("%s: tensor[%d]: name = %s, offset = %zu\n", __func__, i, name, offset);
    }
}

// 释放上下文资源
gguf_free(ctx);
// 返回 true，表示函数执行成功
    return true;
}

// 读取并创建包含张量及其数据的 ggml_context
static bool gguf_ex_read_1(const std::string & fname) {
    // 初始化 ggml_context 指针
    struct ggml_context * ctx_data = NULL;

    // 初始化 gguf_init_params 结构体
    struct gguf_init_params params = {
        /*.no_alloc = */ false,  // 不禁用内存分配
        /*.ctx      = */ &ctx_data,  // 将 ggml_context 指针赋值给 ctx_data
    };

    // 从文件中初始化 gguf_context
    struct gguf_context * ctx = gguf_init_from_file(fname.c_str(), params);

    // 打印版本号
    printf("%s: version:      %d\n", __func__, gguf_get_version(ctx));
    // 打印对齐方式
    printf("%s: alignment:   %zu\n", __func__, gguf_get_alignment(ctx));
    // 打印数据偏移量
    printf("%s: data offset: %zu\n", __func__, gguf_get_data_offset(ctx));

    // kv
```

    // 获取键值对的数量
    const int n_kv = gguf_get_n_kv(ctx);

    // 打印键值对的数量
    printf("%s: n_kv: %d\n", __func__, n_kv);

    // 遍历键值对
    for (int i = 0; i < n_kv; ++i) {
        // 获取第i个键
        const char * key = gguf_get_key(ctx, i);

        // 打印键值对的键
        printf("%s: kv[%d]: key = %s\n", __func__, i, key);
    }

    // tensor info
    // 获取张量的数量
    const int n_tensors = gguf_get_n_tensors(ctx);

    // 打印张量的数量
    printf("%s: n_tensors: %d\n", __func__, n_tensors);

    // 遍历张量
    for (int i = 0; i < n_tensors; ++i) {
        // 获取第i个张量的名称
        const char * name   = gguf_get_tensor_name  (ctx, i);
// 获取张量在内存中的偏移量
const size_t offset = gguf_get_tensor_offset(ctx, i);

// 打印张量的名称和偏移量
printf("%s: tensor[%d]: name = %s, offset = %zu\n", __func__, i, name, offset);
}

// data
{
    // 获取上下文中张量的数量
    const int n_tensors = gguf_get_n_tensors(ctx);

    // 遍历每个张量
    for (int i = 0; i < n_tensors; ++i) {
        // 打印正在读取的张量数据
        printf("%s: reading tensor %d data\n", __func__, i);

        // 获取张量的名称
        const char * name = gguf_get_tensor_name(ctx, i);

        // 获取张量的结构体
        struct ggml_tensor * cur = ggml_get_tensor(ctx_data, name);

        // 打印张量的维度、名称和数据指针
        printf("%s: tensor[%d]: n_dims = %d, name = %s, data = %p\n", __func__, i, cur->n_dims, cur->name, cur->data);

        // 打印前10个元素
// 将cur->data强制转换为const float*类型的指针，并赋值给data
const float * data = (const float *) cur->data;

// 打印name和data的前10个元素
printf("%s data[:10] : ", name);
for (int j = 0; j < MIN(10, ggml_nelements(cur)); ++j) {
    printf("%f ", data[j]);
}
printf("\n\n");

// 检查data的值
{
    // 将cur->data强制转换为const float*类型的指针，并赋值给data
    const float * data = (const float *) cur->data;
    // 遍历data数组，如果发现元素值不等于100+i，则打印错误信息并返回false
    for (int j = 0; j < ggml_nelements(cur); ++j) {
        if (data[j] != 100 + i) {
            fprintf(stderr, "%s: tensor[%d]: data[%d] = %f\n", __func__, i, j, data[j]);
            return false;
        }
    }
}
    // 打印函数名和ctx_data的大小
    printf("%s: ctx_data size: %zu\n", __func__, ggml_get_mem_size(ctx_data));

    // 释放ctx_data的内存
    ggml_free(ctx_data);
    // 释放ctx的内存
    gguf_free(ctx);

    // 返回true
    return true;
}

// 主函数
int main(int argc, char ** argv) {
    // 如果参数小于3，打印使用说明并返回-1
    if (argc < 3) {
        printf("usage: %s data.gguf r|w\n", argv[0]);
        return -1;
    }

    // 将参数转换为字符串
    const std::string fname(argv[1]);
    const std::string mode (argv[2]);

    // 断言mode必须为r或w
    GGML_ASSERT((mode == "r" || mode == "w") && "mode must be r or w");
# 如果模式为写入模式
if (mode == "w") {
    # 调用 gguf_ex_write 函数写入文件，并使用断言检查是否写入成功
    GGML_ASSERT(gguf_ex_write(fname) && "failed to write gguf file");
# 如果模式为读取模式
} else if (mode == "r") {
    # 调用 gguf_ex_read_0 函数读取文件，并使用断言检查是否读取成功
    GGML_ASSERT(gguf_ex_read_0(fname) && "failed to read gguf file");
    # 调用 gguf_ex_read_1 函数读取文件，并使用断言检查是否读取成功
    GGML_ASSERT(gguf_ex_read_1(fname) && "failed to read gguf file");
}

# 返回 0 表示执行成功
return 0;
```