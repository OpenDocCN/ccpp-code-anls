# `ggml\tests\test-quantize-fns.cpp`

```
// 用于量化特定函数的单元测试 - 量化、反量化和点积

#include "ggml.h"  // 包含自定义的头文件 ggml.h

#undef NDEBUG  // 取消 NDEBUG 宏定义
#include <assert.h>  // 包含断言库
#include <math.h>  // 包含数学库
#include <stdio.h>  // 包含标准输入输出库
#include <string>  // 包含字符串库
#include <vector>  // 包含向量库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 可能丢失数据的警告
#endif

constexpr float MAX_QUANTIZATION_REFERENCE_ERROR = 0.0001f;  // 定义最大量化参考误差
constexpr float MAX_QUANTIZATION_TOTAL_ERROR = 0.002f;  // 定义最大量化总误差
constexpr float MAX_QUANTIZATION_TOTAL_ERROR_2BITS = 0.0075f;  // 定义最大2位量化总误差
constexpr float MAX_QUANTIZATION_TOTAL_ERROR_3BITS = 0.0040f;  // 定义最大3位量化总误差
constexpr float MAX_DOT_PRODUCT_ERROR = 0.02f;  // 定义最大点积误差

static const char* RESULT_STR[] = {"ok", "FAILED"};  // 定义结果字符串数组

// 生成合成数据
static void generate_data(float offset, size_t n, float * dst) {
    for (size_t i = 0; i < n; i++) {
        dst[i] = 0.1 + 2*cosf(i + offset);  // 生成合成数据
    }
}

// 计算两个浮点数组之间的均方根误差
static float array_rmse(const float * a1, const float * a2, size_t n) {
    double sum = 0;
    for (size_t i = 0; i < n; i++) {
        double diff = a1[i] - a2[i];
        sum += diff * diff;
    }
    return sqrtf(sum) / n;  // 返回均方根误差
}

// 测试数据的总量化误差
static float total_quantization_error(ggml_type_traits_t & qfns, size_t test_size, const float * test_data) {
    std::vector<uint8_t> tmp_q(2*test_size);  // 创建临时量化数据向量
    std::vector<float> tmp_out(test_size);  // 创建临时输出数据向量

    qfns.from_float(test_data, tmp_q.data(), test_size);  // 将测试数据从浮点数转换为量化数据
    qfns.to_float(tmp_q.data(), tmp_out.data(), test_size);  // 将量化数据转换回浮点数
    return array_rmse(test_data, tmp_out.data(), test_size);  // 返回总量化误差
}

// 测试数据的参考量化误差
static float reference_quantization_error(ggml_type_traits_t & qfns, size_t test_size, const float * test_data) {
    std::vector<uint8_t> tmp_q(2*test_size);  // 创建临时量化数据向量
    std::vector<float> tmp_out(test_size);  // 创建临时输出数据向量
    std::vector<float> tmp_out_ref(test_size);  // 创建临时参考输出数据向量

    qfns.from_float(test_data, tmp_q.data(), test_size);  // 将测试数据从浮点数转换为量化数据
    qfns.to_float(tmp_q.data(), tmp_out.data(), test_size);  // 将量化数据转换回浮点数

    qfns.from_float_reference(test_data, tmp_q.data(), test_size);  // 使用参考方法将测试数据从浮点数转换为量化数据
    # 将tmp_q的数据转换为浮点数，并存储到tmp_out_ref中
    qfns.to_float(tmp_q.data(), tmp_out_ref.data(), test_size);
    # 返回tmp_out和tmp_out_ref之间的均方根误差
    return array_rmse(tmp_out.data(), tmp_out_ref.data(), test_size);
// 计算两个数组的点积，返回结果
static float dot_product(const float * a1, const float * a2, size_t test_size) {
    // 初始化和为0
    double sum = 0;
    // 遍历数组，计算点积
    for (size_t i = 0; i < test_size; i++) {
        sum += a1[i] * a2[i];
    }
    // 返回点积结果
    return sum;
}

// 计算点积误差
static float dot_product_error(
    ggml_type_traits_t & qfns, size_t test_size, const float * test_data1, const float *test_data2
) {
    // 创建临时存储量子化数据的向量
    std::vector<uint8_t> tmp_q1(2*test_size);
    std::vector<uint8_t> tmp_q2(2*test_size);

    // 获取向量点积的类型特性
    auto vdot = ggml_internal_get_type_traits(qfns.vec_dot_type);

    // 将浮点数转换为量子化数据
    qfns.from_float(test_data1, tmp_q1.data(), test_size);
    vdot.from_float(test_data2, tmp_q2.data(), test_size);

    // 初始化结果为无穷大
    float result = INFINITY;
    // 计算向量点积
    qfns.vec_dot(test_size, &result, tmp_q1.data(), tmp_q2.data());

    // 计算参考点积
    const float dot_ref = dot_product(test_data1, test_data2, test_size);

    // 返回点积误差
    return fabsf(result - dot_ref) / test_size;
}

// 主函数
int main(int argc, char * argv[]) {
    // 初始化变量
    bool verbose = false;
    const size_t test_size = 32 * 128;

    std::string arg;
    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        arg = argv[i];

        // 判断是否为-v参数
        if (arg == "-v") {
            verbose = true;
        } else {
            // 输出错误信息并返回
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            return 1;
        }
    }

    // 创建测试数据数组
    std::vector<float> test_data(test_size);
    std::vector<float> test_data2(test_size);

    // 生成测试数据
    generate_data(0.0, test_data.size(), test_data.data());
    generate_data(1.0, test_data2.size(), test_data2.data());

    // 初始化 GGML，确保浮点数转换表被初始化
    struct ggml_init_params ggml_params = {
        /* .mem_size   = */ 1*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ true,
    };
    struct ggml_context * ctx = ggml_init(ggml_params);

    // 初始化失败次数和标志
    int num_failed = 0;
    bool failed = false;
    // 遍历 GGML_TYPE_COUNT 次，i 从 0 到 GGML_TYPE_COUNT-1
    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        // 将 i 转换为 ggml_type 类型
        ggml_type type = (ggml_type) i;
        // 获取指定类型的类型特性
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);

        // 如果类型特性中的 blck_size 为 0，则跳过本次循环
        // deprecated - skip
        if (qfns.blck_size == 0) {
            continue;
        }

        // 打印正在测试的类型名称
        printf("Testing %s\n", ggml_type_name((ggml_type) i));

        // 如果类型特性中包含 from_float 和 to_float 函数
        if (qfns.from_float && qfns.to_float) {
            // 计算总量化误差
            const float total_error = total_quantization_error(qfns, test_size, test_data.data());
            // 根据类型选择最大量化误差
            const float max_quantization_error =
                type == GGML_TYPE_Q2_K ? MAX_QUANTIZATION_TOTAL_ERROR_2BITS :
                type == GGML_TYPE_Q3_K ? MAX_QUANTIZATION_TOTAL_ERROR_3BITS : MAX_QUANTIZATION_TOTAL_ERROR;
            // 判断是否测试失败
            failed = !(total_error < max_quantization_error);
            // 统计测试失败的数量
            num_failed += failed;
            // 如果测试失败或者 verbose 为真，则打印绝对量化误差
            if (failed || verbose) {
                printf("%5s absolute quantization error:    %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], total_error);
            }

            // 计算参考量化误差
            const float reference_error = reference_quantization_error(qfns, test_size, test_data.data());
            // 判断是否测试失败
            failed = !(reference_error < MAX_QUANTIZATION_REFERENCE_ERROR);
            // 统计测试失败的数量
            num_failed += failed;
            // 如果测试失败或者 verbose 为真，则打印参考实现误差
            if (failed || verbose) {
                printf("%5s reference implementation error: %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], reference_error);
            }

            // 计算点积误差
            const float vec_dot_error = dot_product_error(qfns, test_size, test_data.data(), test_data2.data());
            // 判断是否测试失败
            failed = !(vec_dot_error < MAX_DOT_PRODUCT_ERROR);
            // 统计测试失败的数量
            num_failed += failed;
            // 如果测试失败或者 verbose 为真，则打印点积误差
            if (failed || verbose) {
                printf("%5s dot product error:              %s (%f)\n", ggml_type_name(type), RESULT_STR[failed], vec_dot_error);
            }
        }
    }

    // 如果有测试失败或者 verbose 为真，则打印测试失败的数量
    if (num_failed || verbose) {
        printf("%d tests failed\n", num_failed);
    }

    // 释放上下文资源
    ggml_free(ctx);

    // 返回测试是否有失败
    return num_failed > 0;
# 闭合前面的函数定义
```