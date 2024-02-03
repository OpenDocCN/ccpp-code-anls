# `ggml\tests\test-quantize-perf.cpp`

```cpp
// 在合成数据上对量化特定函数进行基准测试

#include "ggml.h"  // 包含自定义的头文件 ggml.h

#undef NDEBUG  // 取消 NDEBUG 宏定义
#include <algorithm>  // 包含算法库
#include <assert.h>  // 包含断言库
#include <functional>  // 包含函数库
#include <inttypes.h>  // 包含整数类型库
#include <math.h>  // 包含数学库
#include <memory>  // 包含内存库
#include <stdio.h>  // 包含标准输入输出库
#include <string>  // 包含字符串库
#include <vector>  // 包含向量库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 可能丢失数据的警告
#endif

#define MAX_ALIGNMENT 64  // 定义最大对齐值为 64
#define QK 32  // 定义 QK 为 32
#define WARMUP 5  // 定义预热次数为 5
#define ITERATIONS 10  // 定义迭代次数为 10
#define MAX_ITERATIONS 100000000  // 定义最大迭代次数为 100000000

#define L1_SIZE      32*128  // 定义 L1 缓存大小为 32*128
#define L2_SIZE     32*2048  // 定义 L2 缓存大小为 32*2048
#define L3_SIZE    32*20480  // 定义 L3 缓存大小为 32*20480
#define MEM_SIZE 32*2048000  // 定义内存大小为 32*2048000

struct quantize_perf_params {
    std::vector<std::string> include_types;  // 包含类型的字符串向量
    std::vector<size_t> test_sizes;  // 测试大小的大小_t型向量
    size_t alignment_offset = 0;  // 对齐偏移量默认为 0
    bool op_quantize_row_q_reference = false;  // op_quantize_row_q_reference 默认为 false
    bool op_quantize_row_q = false;  // op_quantize_row_q 默认为 false
    bool op_dequantize_row_q = false;  // op_dequantize_row_q 默认为 false
    bool op_quantize_row_q_dot = false;  // op_quantize_row_q_dot 默认为 false
    bool op_vec_dot_q = false;  // op_vec_dot_q 默认为 false
    int64_t iterations = ITERATIONS;  // 迭代次数默认为 ITERATIONS
};

#if defined(__x86_64__) || defined(__i386__)

#include <x86intrin.h>
inline int64_t cpu_cycles() {
// 粗略检测新的 CPU
#ifdef __POPCNT__
    unsigned int dummy;
    return __rdtscp(&dummy);
#else
    return __rdtsc();
#endif
}

#else

#define cpu_cycles() 0

#endif


// 生成合成数据
static void generate_data(float offset, size_t n, float * dst) {
    for (size_t i = 0; i < n; i++) {
        dst[i] = 0.1 + 2*cosf(i + offset);
    }
}

// 计算每秒的吉比字节数
static float gigabytes_per_second(size_t bytes, int64_t usecs) {
    return bytes / (float) usecs * 1000000 / (1024*1024*1024);
}

// 对齐指针并添加偏移量
static void * align_with_offset(void * ptr, int offset) {
    size_t dummy_size = MAX_ALIGNMENT * 4;
    return (char *) std::align(MAX_ALIGNMENT, MAX_ALIGNMENT, ptr, dummy_size) + offset;
}

// 对函数进行基准测试
static void benchmark_function(size_t size, size_t q_size, int64_t iterations, const std::function<float(void)> & func) {
    int64_t min_time_us = INT64_MAX;  // 最小时间初始化为最大整数
    int64_t total_time_us = 0;  // 总时间初始化为 0
    int64_t min_time_cycles = INT64_MAX;  // 最小时间周期初始化为最大整数
    // 初始化总时间周期为0
    int64_t total_time_cycles = 0;

    // 进行预热，调用func()函数WARMUP次
    for (int i = 0; i < WARMUP; i++) {
        func();
    }

    // 进行实际测试，调用func()函数iterations次
    for (int i = 0; i < iterations; i++) {
        // 记录开始时间和开始CPU周期数
        const int64_t start_time = ggml_time_us();
        const int64_t start_cycles = cpu_cycles();

        // 调用func()函数

        func();

        // 记录结束CPU周期数和结束时间
        const int64_t end_cycles = cpu_cycles();
        const int64_t end_time = ggml_time_us();

        // 计算总CPU周期数和最小CPU周期数
        total_time_cycles += end_cycles - start_cycles;
        min_time_cycles = std::min(min_time_cycles, end_cycles - start_cycles);

        // 计算总时间和最小时间
        total_time_us += end_time - start_time;
        min_time_us = std::min(min_time_us, end_time - start_time);
    }

    // 输出最小CPU周期数
    printf("      min cycles/%d vals   : %9.2f\n",  QK, QK * min_time_cycles / (float) size);
    // 输出平均CPU周期数
    printf("      avg cycles/%d vals   : %9.2f\n",  QK, QK * total_time_cycles / (float) (size * iterations));
    // 输出float32的吞吐量
    printf("      float32 throughput   : %9.2f GB/s\n",  gigabytes_per_second(4 * size * iterations, total_time_us));
    // 输出量化的吞吐量
    printf("      quantized throughput : %9.2f GB/s\n",  gigabytes_per_second(q_size * iterations, total_time_us));
}

// 显示用法信息
static void usage(char * argv[]) {
    printf("Benchmark quantization specific functions on synthetic data\n");
    printf("\n");
    printf("usage: %s [options]\n", argv[0]);
    printf("\n");
    printf("options: (default)\n");
    printf("  -h, --help            show this help message and exit\n");
    printf("  --size SIZE           set test size, divisible by 32 (L1_SIZE:%d)\n", L1_SIZE);
    printf("  -3                    use size as L1, L2, L3 sizes (L1:%d L2:%d L3:%d)\n", L1_SIZE, L2_SIZE, L3_SIZE);
    printf("  -4                    use size as L1, L2, L3, MEM sizes (L1:%d L2:%d L3:%d MEM:%d)\n", L1_SIZE, L2_SIZE, L3_SIZE, MEM_SIZE);
    printf("  --op OP               set test operation as quantize_row_q_reference, quantize_row_q, dequantize_row_q,\n");
    printf("                        quantize_row_q_dot, vec_dot_q (all)\n");
    printf("  --type TYPE           set test type as");
    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        ggml_type type = (ggml_type) i;
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);
        if (ggml_type_name(type) != NULL) {
            if (qfns.from_float && qfns.to_float) {
                printf(" %s", ggml_type_name(type));
            }
        }
    }
    printf(" (all)\n");
    printf("  --alignment-offset OFFSET\n");
    printf("                        set alignment offset as OFFSET (0)\n");
    printf("  -i NUM, --iterations NUM\n");
    printf("                        set test iteration number (%d)\n", ITERATIONS);
}

int main(int argc, char * argv[]) {
    quantize_perf_params params {};

    // 读取命令行参数

    bool invalid_param = false;
    std::string arg;
    }
    if (invalid_param) {
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        return 1;
    }

    if (params.test_sizes.empty()) {
        params.test_sizes.push_back(L1_SIZE);
    }
    // 如果没有设置任何量化或者点积操作，则将所有这些操作都设置为 true
    if (!(params.op_quantize_row_q_reference || params.op_quantize_row_q || params.op_dequantize_row_q || params.op_quantize_row_q_dot || params.op_vec_dot_q)) {
        params.op_quantize_row_q_reference = params.op_quantize_row_q = params.op_dequantize_row_q = params.op_quantize_row_q_dot = params.op_vec_dot_q = true;
    }

    // 对测试大小进行排序
    std::sort(params.test_sizes.begin(), params.test_sizes.end());
    // 获取最大的测试大小
    size_t largest = params.test_sizes.back();

    // 初始化用于测试的数据向量
    std::vector<uint8_t> test_data1_v(largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_data2_v(largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_q1_v   (largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_q2_v   (largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_out_v  (largest*4 + MAX_ALIGNMENT*2);

    // 将测试数据向量进行对齐
    float * test_data1 = (float *) align_with_offset(test_data1_v.data(), params.alignment_offset);
    float * test_data2 = (float *) align_with_offset(test_data2_v.data(), params.alignment_offset);
    float * test_q1    = (float *) align_with_offset(test_q1_v.data(),    params.alignment_offset);
    float * test_q2    = (float *) align_with_offset(test_q2_v.data(),    params.alignment_offset);
    float * test_out   = (float *) align_with_offset(test_out_v.data(),   params.alignment_offset);

    // 生成测试数据
    generate_data(0, largest, test_data1);
    generate_data(1, largest, test_data2);

    // 获取迭代次数
    int64_t iterations = params.iterations;

    // 初始化 GGML，确保浮点转换表被初始化
    struct ggml_init_params ggml_params = {
        /* .mem_size   = */ 1*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ true,
    };
    struct ggml_context * ctx = ggml_init(ggml_params);

    // 释放 GGML 上下文
    ggml_free(ctx);

    // 返回 0
    return 0;
# 闭合前面的函数定义
```