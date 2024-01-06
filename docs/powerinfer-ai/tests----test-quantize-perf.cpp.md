# `PowerInfer\tests\test-quantize-perf.cpp`

```
// 在合成数据上对量化特定函数进行基准测试

#include "ggml.h" // 包含 ggml 库的头文件

#undef NDEBUG // 取消 NDEBUG 宏定义，启用断言

#include <algorithm> // 包含算法库
#include <assert.h> // 包含断言库
#include <functional> // 包含函数对象库
#include <inttypes.h> // 包含整数类型库
#include <math.h> // 包含数学库
#include <memory> // 包含内存管理库
#include <stdio.h> // 包含标准输入输出库
#include <string> // 包含字符串库
#include <vector> // 包含向量库

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // 禁用特定编译器警告
#endif

#define MAX_ALIGNMENT 64 // 定义最大对齐值为 64
// 定义常量 QK 为 32
#define QK 32
// 定义常量 WARMUP 为 5
#define WARMUP 5
// 定义常量 ITERATIONS 为 10
#define ITERATIONS 10
// 定义常量 MAX_ITERATIONS 为 100000000
#define MAX_ITERATIONS 100000000

// 定义常量 L1_SIZE 为 32*128
#define L1_SIZE 32*128
// 定义常量 L2_SIZE 为 32*2048
#define L2_SIZE 32*2048
// 定义常量 L3_SIZE 为 32*20480
#define L3_SIZE 32*20480
// 定义常量 MEM_SIZE 为 32*2048000
#define MEM_SIZE 32*2048000

// 定义结构体 quantize_perf_params
struct quantize_perf_params {
    // 包含类型的字符串向量
    std::vector<std::string> include_types;
    // 测试大小的大小向量
    std::vector<size_t> test_sizes;
    // 对齐偏移量，默认为 0
    size_t alignment_offset = 0;
    // 是否进行操作的量化行参考
    bool op_quantize_row_q_reference = false;
    // 是否进行操作的量化行
    bool op_quantize_row_q = false;
    // 是否进行操作的反量化行
    bool op_dequantize_row_q = false;
    // 是否进行操作的量化行点乘
    bool op_quantize_row_q_dot = false;
    // 是否进行操作的向量点乘
    bool op_vec_dot_q = false;
    // 迭代次数，默认为 ITERATIONS
    int64_t iterations = ITERATIONS;
// 如果定义了__x86_64__或__i386__，则包含x86intrin.h头文件
#if defined(__x86_64__) || defined(__i386__)

// 定义内联函数cpu_cycles，返回CPU周期数
#include <x86intrin.h>
inline int64_t cpu_cycles() {
// 通过检测__POPCNT__来粗略判断CPU是否支持新的指令集
#ifdef __POPCNT__
    unsigned int dummy;
    return __rdtscp(&dummy); // 使用__rdtscp函数获取CPU周期数
#else
    return __rdtsc(); // 使用__rdtsc函数获取CPU周期数
#endif
}

#else

#define cpu_cycles() 0 // 如果不是x86架构，则返回0

#endif
// 生成合成数据，给定偏移量和数量，将数据存储到指定的数组中
static void generate_data(float offset, size_t n, float * dst) {
    for (size_t i = 0; i < n; i++) {
        dst[i] = 0.1 + 2*cosf(i + offset);
    }
}

// 计算每秒传输的数据量（以GB为单位）
static float gigabytes_per_second(size_t bytes, int64_t usecs) {
    return bytes / (float) usecs * 1000000 / (1024*1024*1024);
}

// 将指针对齐到指定的偏移量
static void * align_with_offset(void * ptr, int offset) {
    size_t dummy_size = MAX_ALIGNMENT * 4;
    return (char *) std::align(MAX_ALIGNMENT, MAX_ALIGNMENT, ptr, dummy_size) + offset;
}

// 对给定函数进行基准测试，记录最小执行时间
static void benchmark_function(size_t size, size_t q_size, int64_t iterations, const std::function<float(void)> & func) {
    int64_t min_time_us = INT64_MAX;
```
    # 初始化总时间为微秒
    int64_t total_time_us = 0;
    # 初始化最小时间为最大整数
    int64_t min_time_cycles = INT64_MAX;
    # 初始化总时间为周期数
    int64_t total_time_cycles = 0;

    # 预热阶段，执行func()函数WARMUP次
    for (int i = 0; i < WARMUP; i++) {
        func();
    }

    # 迭代测试阶段，执行func()函数iterations次
    for (int i = 0; i < iterations; i++) {
        # 记录开始时间（微秒）和开始周期数
        const int64_t start_time = ggml_time_us();
        const int64_t start_cycles = cpu_cycles();

        # 执行func()函数

        func();

        # 记录结束周期数和结束时间（微秒）
        const int64_t end_cycles = cpu_cycles();
        const int64_t end_time = ggml_time_us();

        # 计算总周期数
        total_time_cycles += end_cycles - start_cycles;
        # 更新最小时间为当前时间和最小时间的较小值
        min_time_cycles = std::min(min_time_cycles, end_cycles - start_cycles);
        # 计算总时间
        total_time_us += end_time - start_time;
    // 计算最小时间差，更新min_time_us
    min_time_us = std::min(min_time_us, end_time - start_time);
    }

    // 打印最小循环次数和平均循环次数
    printf("      min cycles/%d vals   : %9.2f\n",  QK, QK * min_time_cycles / (float) size);
    printf("      avg cycles/%d vals   : %9.2f\n",  QK, QK * total_time_cycles / (float) (size * iterations));
    // 打印float32和quantized的吞吐量
    printf("      float32 throughput   : %9.2f GB/s\n",  gigabytes_per_second(4 * size * iterations, total_time_us));
    printf("      quantized throughput : %9.2f GB/s\n",  gigabytes_per_second(q_size * iterations, total_time_us));
}

// 显示用法
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
    printf("  --op OP               set test opration as quantize_row_q_reference, quantize_row_q, dequantize_row_q,\n");
```
    # 打印提示信息
    printf("                        quantize_row_q_dot, vec_dot_q (all)\n");
    printf("  --type TYPE           set test type as");
    # 遍历所有测试类型
    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        # 获取当前测试类型
        ggml_type type = (ggml_type) i;
        # 获取当前测试类型的转换函数
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);
        # 如果测试类型名称不为空，并且具有浮点数转换函数
        if (ggml_type_name(type) != NULL) {
            if (qfns.from_float && qfns.to_float) {
                # 打印当前测试类型
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
    # 初始化性能测试参数
    quantize_perf_params params {};
// 读取命令行参数
bool invalid_param = false; // 初始化参数是否有效的标志为false
std::string arg; // 声明一个字符串变量arg
for (int i = 1; i < argc; i++) { // 遍历命令行参数
    arg = argv[i]; // 获取当前参数值

    if (arg == "--size") { // 如果参数为"--size"
        if (++i >= argc) { // 如果下一个参数不存在
            invalid_param = true; // 参数无效
            break; // 跳出循环
        }
        size_t size = std::stoi(argv[i]); // 将下一个参数转换为整数类型
        if (size % 32 != 0) { // 如果size不能被32整除
            fprintf(stderr, "error: size %zu not divisible by 32\n", size); // 输出错误信息
            invalid_param = true; // 参数无效
            break; // 跳出循环
        }
        params.test_sizes.push_back(size); // 将size添加到test_sizes向量中
    }
        } else if (arg == "-3") {
            // 如果参数为"-3"，则快速选择可能适合CPU缓存的大小，并添加到测试大小列表中
            params.test_sizes.push_back(L1_SIZE);
            params.test_sizes.push_back(L2_SIZE);
            params.test_sizes.push_back(L3_SIZE);
        } else if (arg == "-4") {
            // 如果参数为"-4"，则快速选择缓存大小和内存大小，并添加到测试大小列表中
            params.test_sizes.push_back(L1_SIZE);
            params.test_sizes.push_back(L2_SIZE);
            params.test_sizes.push_back(L3_SIZE);
            params.test_sizes.push_back(MEM_SIZE);
        } else if (arg == "--op") {
            // 如果参数为"--op"，则检查下一个参数是否存在
            if (++i >= argc) {
                // 如果下一个参数不存在，则设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为字符串
            std::string op {argv[i]};
            // 如果参数为"quantize_row_q_reference"，则设置参数op_quantize_row_q_reference为true
            if (op == "quantize_row_q_reference") {
                params.op_quantize_row_q_reference = true;
            } else if (op == "quantize_row_q") {
                // 如果参数为"quantize_row_q"，则设置相应参数
        // 如果参数是 "op_quantize_row_q"，则设置参数中的 op_quantize_row_q 为 true
        if (op == "op_quantize_row_q") {
            params.op_quantize_row_q = true;
        } 
        // 如果参数是 "dequantize_row_q"，则设置参数中的 op_dequantize_row_q 为 true
        else if (op == "dequantize_row_q") {
            params.op_dequantize_row_q = true;
        } 
        // 如果参数是 "quantize_row_q_dot"，则设置参数中的 op_quantize_row_q_dot 为 true
        else if (op == "quantize_row_q_dot") {
            params.op_quantize_row_q_dot = true;
        } 
        // 如果参数是 "vec_dot_q"，则设置参数中的 op_vec_dot_q 为 true
        else if (op == "vec_dot_q") {
            params.op_vec_dot_q = true;
        } 
        // 如果参数不是以上任何一个，则设置 invalid_param 为 true 并跳出循环
        else {
            invalid_param = true;
            break;
        }
        // 如果参数是 "--type"，则将下一个参数作为类型添加到 include_types 中
        else if (arg == "--type") {
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.include_types.push_back(argv[i]);
        } 
        // 如果参数是 "--alignment-offset"，则将下一个参数作为偏移量添加到参数中
        else if (arg == "--alignment-offset") {
            if (++i >= argc) {
                invalid_param = true;
            // 如果遇到分号，跳出循环
            break;
            // 将参数转换为整数类型
            int alignment = std::stoi(argv[i]);
            // 如果对齐偏移小于0或大于最大对齐偏移，输出错误信息并设置参数为无效
            if (alignment < 0 || alignment > MAX_ALIGNMENT) {
                fprintf(stderr, "error: aligment-offset must be less than %d\n", MAX_ALIGNMENT);
                invalid_param = true;
                break;
            }
            // 将对齐偏移设置为参数值
            params.alignment_offset = alignment;
        } else if ((arg == "-i") || (arg == "--iterations")) {
            // 如果下一个参数不存在，设置参数为无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将参数转换为整数类型
            int number = std::stoi(argv[i]);
            // 如果迭代次数小于0或大于最大迭代次数，输出错误信息并设置参数为无效
            if (number < 0 || number > MAX_ITERATIONS) {
                fprintf(stderr, "error: iterations must be less than %d\n", MAX_ITERATIONS);
                invalid_param = true;
                break;
            }
    // 如果参数是迭代次数，则设置参数的迭代次数
    params.iterations = number;
    // 如果参数是"-h"或"--help"，则显示用法并返回1
    } else if ((arg == "-h") || (arg == "--help")) {
        usage(argv);
        return 1;
    // 如果参数不是已知的选项，则显示错误信息并返回1
    } else {
        fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
        return 1;
    }
    // 如果存在无效的参数，则显示错误信息并返回1
    if (invalid_param) {
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        return 1;
    }
    // 如果测试大小为空，则添加默认的测试大小
    if (params.test_sizes.empty()) {
        params.test_sizes.push_back(L1_SIZE);
    }
    // 如果没有任何操作被选中，则将所有操作都设置为true
    if (!(params.op_quantize_row_q_reference || params.op_quantize_row_q || params.op_dequantize_row_q || params.op_quantize_row_q_dot || params.op_vec_dot_q)) {
        params.op_quantize_row_q_reference = params.op_quantize_row_q = params.op_dequantize_row_q = params.op_quantize_row_q_dot = params.op_vec_dot_q = true;
    }
    # 对测试大小进行排序
    std::sort(params.test_sizes.begin(), params.test_sizes.end());
    # 找到最大的测试大小
    size_t largest = params.test_sizes.back();

    # 创建存储测试数据的向量
    std::vector<uint8_t> test_data1_v(largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_data2_v(largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_q1_v   (largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_q2_v   (largest*4 + MAX_ALIGNMENT*2);
    std::vector<uint8_t> test_out_v  (largest*4 + MAX_ALIGNMENT*2);

    # 将测试数据向量对齐并转换为浮点数指针
    float * test_data1 = (float *) align_with_offset(test_data1_v.data(), params.alignment_offset);
    float * test_data2 = (float *) align_with_offset(test_data2_v.data(), params.alignment_offset);
    float * test_q1    = (float *) align_with_offset(test_q1_v.data(),    params.alignment_offset);
    float * test_q2    = (float *) align_with_offset(test_q2_v.data(),    params.alignment_offset);
    float * test_out   = (float *) align_with_offset(test_out_v.data(),   params.alignment_offset);

    # 生成测试数据
    generate_data(0, largest, test_data1);
    generate_data(1, largest, test_data2);

    # 设置迭代次数
    int64_t iterations = params.iterations;
    // 初始化 GGML，确保浮点转换表被初始化
    // 定义初始化参数结构体，设置内存大小为 1*1024，内存缓冲区为空，不进行内存分配
    struct ggml_init_params ggml_params = {
        /* .mem_size   = */ 1*1024,
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ true,
    };
    // 初始化 GGML 上下文
    struct ggml_context * ctx = ggml_init(ggml_params);

    // 遍历 GGML 类型数量
    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        // 获取当前类型
        ggml_type type = (ggml_type) i;
        // 获取类型特性
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);
        // 如果指定了包含的类型，并且当前类型不在包含的类型列表中，则跳过当前类型
        if (!params.include_types.empty() && ggml_type_name(type) && std::find(params.include_types.begin(), params.include_types.end(), ggml_type_name(type)) == params.include_types.end()) {
            continue;
        }

        // 如果当前类型支持从浮点数到当前类型的转换以及从当前类型到浮点数的转换
        if (qfns.from_float && qfns.to_float) {
            // 打印当前类型的名称
            printf("%s\n", ggml_type_name(type));
                // 如果参数中包含 op_quantize_row_q_reference，则执行以下代码块
                if (params.op_quantize_row_q_reference) {
                    // 打印提示信息
                    printf("  quantize_row_q_reference\n");
                    // 遍历测试大小列表
                    for (size_t size : params.test_sizes) {
                        // 打印测试大小和对应的内存占用
                        printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
                        // 定义匿名函数 quantize_fn，将测试数据转换为量化数据，并返回第一个量化数据
                        auto quantize_fn = [&](void) -> float {
                            qfns.from_float_reference(test_data1, test_q1, size);
                            return test_q1[0];
                        };
                        // 计算量化后的数据大小
                        size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
                        // 执行基准测试函数，传入测试大小、量化后数据大小、迭代次数和匿名函数
                        benchmark_function(size, quantized_size, iterations, quantize_fn);
                    }
                    // 打印换行符
                    printf("\n");
                }

                // 如果参数中包含 op_quantize_row_q，则执行以下代码块
                if (params.op_quantize_row_q) {
                    // 打印提示信息
                    printf("  quantize_row_q\n");
                    // 遍历测试大小列表
                    for (size_t size : params.test_sizes) {
                        // 打印测试大小和对应的内存占用
                        printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
                        // 定义匿名函数 quantize_fn，将测试数据转换为量化数据
                        auto quantize_fn = [&](void) -> float {
                            qfns.from_float(test_data1, test_q1, size);
// 返回 test_q1 数组的第一个元素
return test_q1[0];
};

// 计算量化后的大小
size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);

// 调用 benchmark_function 函数进行基准测试
benchmark_function(size, quantized_size, iterations, quantize_fn);
}
printf("\n");
}

// 如果参数中包含 op_dequantize_row_q，则执行以下代码
if (params.op_dequantize_row_q) {
    printf("  dequantize_row_q\n");
    // 将 test_data1 数组中的数据转换为 test_q1 数组中的量化数据
    qfns.from_float(test_data1, test_q1, largest);
    // 遍历测试大小数组
    for (size_t size : params.test_sizes) {
        // 打印测试大小和对应的内存大小
        printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
        // 定义匿名函数 quantize_fn，将 test_q1 数组中的数据转换为 test_out 数组中的浮点数
        auto quantize_fn = [&](void) -> float {
            qfns.to_float(test_q1, test_out, size);
            return test_out[0];
        };
        // 计算量化后的大小
        size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
        // 调用 benchmark_function 函数进行基准测试
        benchmark_function(size, quantized_size, iterations, quantize_fn);
    }
// 如果参数 op_quantize_row_q_dot 为真，则执行以下代码块
if (params.op_quantize_row_q_dot) {
    // 打印 quantize_row_q_dot
    printf("  quantize_row_q_dot\n");
    // 遍历测试大小数组
    for (size_t size : params.test_sizes) {
        // 打印测试大小和对应的内存大小
        printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
        // 定义匿名函数 quantize_fn，该函数执行向量点积并返回结果
        auto quantize_fn = [&](void) -> float {
            // 调用 ggml_internal_get_type_traits 函数获取类型特性
            auto vdot = ggml_internal_get_type_traits(qfns.vec_dot_type);
            // 使用特性中的函数将测试数据1和测试q1进行向量点积
            vdot.from_float(test_data1, test_q1, size);
            // 返回测试q1的第一个元素
            return test_q1[0];
        };
        // 计算量化后的大小
        size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
        // 调用 benchmark_function 函数进行基准测试
        benchmark_function(size, quantized_size, iterations, quantize_fn);
    }
    // 打印换行符
    printf("\n");
}

// 如果参数 op_vec_dot_q 为真，则执行以下代码块
if (params.op_vec_dot_q) {
    // 打印 vec_dot_q
    printf("  vec_dot_q\n");
// 使用 qfns.from_float 将 test_data1 转换为 test_q1，largest 为最大值
qfns.from_float(test_data1, test_q1, largest);
// 使用 qfns.from_float 将 test_data2 转换为 test_q2，largest 为最大值
qfns.from_float(test_data2, test_q2, largest);
// 遍历测试大小数组中的每个大小
for (size_t size : params.test_sizes) {
    // 打印当前测试大小和对应的内存占用
    printf("    %zu values (%.2f MB)\n", size, 4*size/(float)(1024*1024));
    // 定义一个匿名函数 quantize_fn，用于执行量化操作
    auto quantize_fn = [&](void) -> float {
        // 定义变量 result，用于存储量化结果
        float result;
        // 使用 qfns.vec_dot 执行向量点积操作，将结果存储在 result 中
        qfns.vec_dot(size, &result, test_q1, test_q2);
        // 返回量化结果
        return result;
    };
    // 计算量化后的大小
    size_t quantized_size = size / ggml_blck_size(type) * ggml_type_size(type);
    // 执行基准测试函数 benchmark_function，传入测试大小、量化后大小、迭代次数和量化函数
    benchmark_function(size, quantized_size, iterations, quantize_fn);
}
// 打印换行符
printf("\n");
// 释放上下文内存
ggml_free(ctx);
// 返回 0，表示程序正常结束
return 0;
这是一个代码块的结束标记，表示前面的函数或者循环等代码块的结束。
```