# `PowerInfer\tests\test-quantize-fns.cpp`

```
// 为量化特定函数（quantize、dequantize和点积）编写单元测试

#include "ggml.h" // 包含自定义的头文件 ggml.h

#undef NDEBUG // 取消 NDEBUG 宏定义，使得 assert 生效
#include <assert.h> // 包含断言库
#include <math.h> // 包含数学函数库
#include <stdio.h> // 包含标准输入输出库
#include <string> // 包含字符串库
#include <vector> // 包含向量库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁止特定编译器警告
#endif

// 定义最大量化参考误差
constexpr float MAX_QUANTIZATION_REFERENCE_ERROR = 0.0001f;
// 定义最大量化总误差
constexpr float MAX_QUANTIZATION_TOTAL_ERROR = 0.002f;
// 定义最大量化总误差（2位量化）
constexpr float MAX_QUANTIZATION_TOTAL_ERROR_2BITS = 0.0075f;
// 定义最大量化总误差（3位量化）
constexpr float MAX_QUANTIZATION_TOTAL_ERROR_3BITS = 0.0040f;
// 定义最大点积误差
constexpr float MAX_DOT_PRODUCT_ERROR = 0.02f;
// 定义一个包含两个字符串的常量数组，用于表示结果状态
static const char* RESULT_STR[] = {"ok", "FAILED"};

// 生成合成数据
static void generate_data(float offset, size_t n, float * dst) {
    // 使用余弦函数生成合成数据
    for (size_t i = 0; i < n; i++) {
        dst[i] = 0.1 + 2*cosf(i + offset);
    }
}

// 计算两个浮点数数组之间的均方根误差（RMSE）
static float array_rmse(const float * a1, const float * a2, size_t n) {
    double sum = 0;
    // 计算两个数组之间每个元素的差的平方的和
    for (size_t i = 0; i < n; i++) {
        double diff = a1[i] - a2[i];
        sum += diff * diff;
    }
    // 返回均方根误差的值
    return sqrtf(sum) / n;
}
// 计算测试数据的总量化误差
static float total_quantization_error(ggml_type_traits_t & qfns, size_t test_size, const float * test_data) {
    // 创建临时存储量化后数据的数组
    std::vector<uint8_t> tmp_q(2*test_size);
    // 创建临时存储还原后数据的数组
    std::vector<float> tmp_out(test_size);

    // 将测试数据从浮点数转换为量化后的数据
    qfns.from_float(test_data, tmp_q.data(), test_size);
    // 将量化后的数据还原为浮点数
    qfns.to_float(tmp_q.data(), tmp_out.data(), test_size);
    // 返回测试数据与还原后数据之间的均方根误差
    return array_rmse(test_data, tmp_out.data(), test_size);
}

// 计算参考数据的总量化误差
static float reference_quantization_error(ggml_type_traits_t & qfns, size_t test_size, const float * test_data) {
    // 创建临时存储量化后数据的数组
    std::vector<uint8_t> tmp_q(2*test_size);
    // 创建临时存储还原后数据的数组
    std::vector<float> tmp_out(test_size);
    // 创建临时存储参考数据的还原后数据的数组
    std::vector<float> tmp_out_ref(test_size);

    // 将测试数据从浮点数转换为量化后的数据
    qfns.from_float(test_data, tmp_q.data(), test_size);
    // 将量化后的数据还原为浮点数
    qfns.to_float(tmp_q.data(), tmp_out.data(), test_size);
    # 使用 qfns 对象中的 from_float_reference 方法将 test_data 转换为定点表示存储在 tmp_q 中
    qfns.from_float_reference(test_data, tmp_q.data(), test_size);
    # 使用 qfns 对象中的 to_float 方法将 tmp_q 中的定点表示转换为浮点表示存储在 tmp_out_ref 中
    qfns.to_float(tmp_q.data(), tmp_out_ref.data(), test_size);

    # 返回 tmp_out 和 tmp_out_ref 之间的均方根误差
    return array_rmse(tmp_out.data(), tmp_out_ref.data(), test_size);
}

# 计算两个数组的点积
static float dot_product(const float * a1, const float * a2, size_t test_size) {
    double sum = 0;
    for (size_t i = 0; i < test_size; i++) {
        sum += a1[i] * a2[i];
    }
    return sum;
}

# 计算两个数组的点积误差
static float dot_product_error(
    ggml_type_traits_t & qfns, size_t test_size, const float * test_data1, const float *test_data2
) {
    # 创建两个大小为 2*test_size 的临时数组
    std::vector<uint8_t> tmp_q1(2*test_size);
    std::vector<uint8_t> tmp_q2(2*test_size);
    // 获取类型特征
    auto vdot = ggml_internal_get_type_traits(qfns.vec_dot_type);

    // 将测试数据1转换为浮点数，并存储到临时数组tmp_q1中
    qfns.from_float(test_data1, tmp_q1.data(), test_size);
    // 将测试数据2转换为浮点数，并存储到临时数组tmp_q2中
    vdot.from_float(test_data2, tmp_q2.data(), test_size);

    // 初始化结果为无穷大
    float result = INFINITY;
    // 计算tmp_q1和tmp_q2的点积，结果存储在result中
    qfns.vec_dot(test_size, &result, tmp_q1.data(), tmp_q2.data());

    // 计算测试数据1和测试数据2的点积作为参考值
    const float dot_ref = dot_product(test_data1, test_data2, test_size);

    // 返回结果值与参考值的绝对值除以测试大小
    return fabsf(result - dot_ref) / test_size;
}

int main(int argc, char * argv[]) {
    // 初始化verbose标志为false
    bool verbose = false;
    // 初始化测试大小为32*128
    const size_t test_size = 32 * 128;

    // 初始化字符串arg
    std::string arg;
    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
    // 从命令行参数中获取当前参数
    arg = argv[i];

    // 如果当前参数是"-v"，则设置 verbose 为 true
    if (arg == "-v") {
        verbose = true;
    } else {
        // 如果当前参数不是"-v"，则输出错误信息并返回 1
        fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
        return 1;
    }

    // 创建两个大小为 test_size 的浮点数向量
    std::vector<float> test_data(test_size);
    std::vector<float> test_data2(test_size);

    // 生成测试数据并存储到 test_data 和 test_data2 中
    generate_data(0.0, test_data.size(), test_data.data());
    generate_data(1.0, test_data2.size(), test_data2.data());

    // 初始化 GGML，确保浮点数转换表已经初始化
    struct ggml_init_params ggml_params = {
        /* .mem_size   = */ 1*1024,  // 设置内存大小为 1KB
        /* .mem_buffer = */ NULL,     // 内存缓冲区为空
    /* .no_alloc   = */ true, 
    // 设置 no_alloc 参数为 true
    };
    // 初始化 ggml_context 结构体，传入 ggml_params 参数
    struct ggml_context * ctx = ggml_init(ggml_params);

    // 初始化变量 num_failed 为 0
    int num_failed = 0;
    // 初始化变量 failed 为 false
    bool failed = false;

    // 循环遍历 GGML_TYPE_COUNT
    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        // 获取当前循环的 ggml_type
        ggml_type type = (ggml_type) i;
        // 获取当前 ggml_type 的类型特性
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);

        // 如果类型特性中的 blck_size 为 0，则跳过当前循环
        // deprecated - skip
        if (qfns.blck_size == 0) {
            continue;
        }

        // 打印当前测试的类型名称
        printf("Testing %s\n", ggml_type_name((ggml_type) i));

        // 如果类型特性中包含 from_float 和 to_float 方法
        if (qfns.from_float && qfns.to_float) {
            // 计算总量化误差
            const float total_error = total_quantization_error(qfns, test_size, test_data.data());
// 计算最大量化误差，根据不同类型选择不同的最大量化误差值
const float max_quantization_error =
    type == GGML_TYPE_Q2_K ? MAX_QUANTIZATION_TOTAL_ERROR_2BITS :
    type == GGML_TYPE_Q3_K ? MAX_QUANTIZATION_TOTAL_ERROR_3BITS : MAX_QUANTIZATION_TOTAL_ERROR;
// 判断总误差是否小于最大量化误差，如果大于则标记为失败
failed = !(total_error < max_quantization_error);
// 统计失败的数量
num_failed += failed;
// 如果失败或者需要详细输出，则打印绝对量化误差
if (failed || verbose) {
    printf("%5s absolute quantization error:    %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], total_error);
}

// 计算参考实现误差
const float reference_error = reference_quantization_error(qfns, test_size, test_data.data());
// 判断参考实现误差是否小于最大参考误差，如果大于则标记为失败
failed = !(reference_error < MAX_QUANTIZATION_REFERENCE_ERROR);
// 统计失败的数量
num_failed += failed;
// 如果失败或者需要详细输出，则打印参考实现误差
if (failed || verbose) {
    printf("%5s reference implementation error: %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], reference_error);
}

// 计算向量点积误差
const float vec_dot_error = dot_product_error(qfns, test_size, test_data.data(), test_data2.data());
// 判断向量点积误差是否小于最大点积误差，如果大于则标记为失败
failed = !(vec_dot_error < MAX_DOT_PRODUCT_ERROR);
// 统计失败的数量
num_failed += failed;
// 如果失败或者需要详细输出，则打印向量点积误差
if (failed || verbose) {
    // 打印带有格式的字符串，包括类型、错误信息和错误值
    printf("%5s dot product error:              %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], vec_dot_error);
    // 如果有测试失败或者需要详细输出
    if (num_failed || verbose) {
        // 打印测试失败的数量
        printf("%d tests failed\n", num_failed);
    }
    // 释放上下文资源
    ggml_free(ctx);
    // 返回测试是否有失败
    return num_failed > 0;
}
```