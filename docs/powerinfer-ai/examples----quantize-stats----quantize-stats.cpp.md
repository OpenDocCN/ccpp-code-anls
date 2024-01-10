# `PowerInfer\examples\quantize-stats\quantize-stats.cpp`

```
// 定义 LLAMA_API_INTERNAL 宏
#define LLAMA_API_INTERNAL
// 引入头文件 common.h
#include "common.h"
// 引入头文件 ggml.h
#include "ggml.h"
// 引入头文件 llama.h
#include "llama.h"

// 引入 C++ 标准库头文件
#include <algorithm>
#include <cassert>
#include <cinttypes>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <map>
#include <numeric>
#include <regex>
#include <string>
#include <unordered_map>
#include <vector>
#include <thread>
#include <mutex>

// 如果编译器为 MSC_VER，则禁用警告 4244 和 4267
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义结构体 quantize_stats_params
struct quantize_stats_params {
    // 模型路径，默认为 "models/7B/ggml-model-f16.gguf"
    std::string model = "models/7B/ggml-model-f16.gguf";
    // 是否输出详细信息，默认为 false
    bool verbose = false;
    // 是否输出每层的统计信息，默认为 false
    bool per_layer_stats = false;
    // 是否打印直方图，默认为 false
    bool print_histogram = false;
    // 是否使用参考实现，默认为 false
    bool reference = false;
    // 包含的层名称列表
    std::vector<std::string> include_layers;
    // 排除的层名称列表
    std::vector<std::string> exclude_layers;
    // 包含的类型列表
    std::vector<enum ggml_type> include_types;
};

// 定义常量 HISTOGRAM_BUCKETS，值为 150
constexpr size_t HISTOGRAM_BUCKETS = 150;
// 定义常量 HISTOGRAM_RANGE，值为 0.03
constexpr double HISTOGRAM_RANGE = 0.03;

// 定义结构体 error_stats
struct error_stats {
    // 样本数量
    size_t num_samples;
    // 总误差
    double total_error;
    // 最大误差
    double max_error;
    // 误差直方图
    uint64_t error_histogram[HISTOGRAM_BUCKETS];
};

// 定义静态函数 quantize_stats_print_usage，用于打印使用说明
static void quantize_stats_print_usage(int /*argc*/, char ** argv) {
    // 创建 quantize_stats_params 对象 params
    quantize_stats_params params;
    // 打印使用说明
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    fprintf(stderr, "  -m FNAME, --model FNAME\n");
    fprintf(stderr, "                        model path (default: %s)\n", params.model.c_str());
    fprintf(stderr, "  -r, --reference\n");
    fprintf(stderr, "                        use reference implementation (default: false)\n");
    fprintf(stderr, "  -v, --verbose\n");
    fprintf(stderr, "                        verbose output (default: false)\n");
    fprintf(stderr, "  -p, --per-layer-stats\n");
    fprintf(stderr, "                        print stats per layer (default: false)\n");
    fprintf(stderr, "  --histogram\n");
}
    # 打印错误直方图的选项，默认为 false
    fprintf(stderr, "                        print error histogram (default: false)\n");
    # 包含指定层的选项，只测试与模式匹配的层
    fprintf(stderr, "  -l LAYER, --include-layer LAYER\n");
    # 排除指定层的选项，排除与模式匹配的层
    fprintf(stderr, "                        only test layers matching pattern\n");
    # 只测试给定类型的选项（q4_0, q4_1）
    fprintf(stderr, "  -t TYPE, --type TYPE\n");
    # 换行
    fprintf(stderr, "                        only test given type (q4_0, q4_1)\n");
    # 换行
    fprintf(stderr, "\n");
// 检查图层是否被命令行包含/排除
static bool layer_included(const quantize_stats_params & params, const std::string & layer) {
    // 遍历排除图层列表，如果图层名称匹配正则表达式，则返回 false
    for (const auto& excluded : params.exclude_layers) {
        if (std::regex_search(layer, std::regex(excluded))) {
            return false;
        }
    }
    // 遍历包含图层列表，如果图层名称匹配正则表达式，则返回 true
    for (const auto& included : params.include_layers) {
        if (std::regex_search(layer, std::regex(included))) {
            return true;
        }
    }
    // 如果包含图层列表为空，则返回 true
    return params.include_layers.empty();
}

// 更新错误统计信息，给定量化前后结果的向量
static void update_error_stats(int64_t nelements, const float * input, const float * output, error_stats & stats) {
    // 遍历元素，计算差值并更新统计信息
    for (int64_t i = 0; i < nelements; i++) {
        double diff = input[i] - output[i];
        stats.total_error += diff * diff;
        stats.max_error = fmax(fabs(diff), stats.max_error);
        stats.error_histogram[std::max(std::min((size_t) floor(fabs(diff) / HISTOGRAM_RANGE * HISTOGRAM_BUCKETS), HISTOGRAM_BUCKETS-1), (size_t) 0)]++;
    }
    stats.num_samples += nelements;
}

// 合并错误统计信息
static void combine_error_stats(error_stats & into, const error_stats & from) {
    into.num_samples += from.num_samples;
    into.total_error += from.total_error;
    if (from.max_error > into.max_error) into.max_error = from.max_error;
    for (size_t i=0; i<HISTOGRAM_BUCKETS; ++i) into.error_histogram[i] += from.error_histogram[i];
}

// 查找给定分位数的值
static double find_quantile(const error_stats & stats, double quantile) {
    // 计算直方图中的总和
    double sum = std::accumulate(std::begin(stats.error_histogram), std::end(stats.error_histogram), 0.0);

    double accum = 0;
    // 遍历直方图，找到累积和超过总和乘以分位数的位置
    for (size_t i = 0; i < HISTOGRAM_BUCKETS; i++) {
        accum += stats.error_histogram[i];
        if (accum >= sum*quantile) {
            return (i+1) * HISTOGRAM_RANGE / HISTOGRAM_BUCKETS;
        }
    }
    return INFINITY;
}

// 打印错误统计信息
static void print_error_stats(const std::string & name, const error_stats & stats, bool print_histogram) {
    // 计算均方根误差
    double rmse = sqrt(stats.total_error / (double) stats.num_samples);
    // 计算中位数
    double median = find_quantile(stats, .5);
    // 计算95%分位数
    double pct95 = find_quantile(stats, .95);
    // 打印结果，包括名称、均方根误差、最大误差、95%分位数、中位数
    printf("%-50s: rmse %.8f, maxerr %.8f, 95pct<%.4f, median<%.4f\n", name.c_str(), rmse, stats.max_error, pct95, median);
    // 如果需要打印误差分布直方图
    if (print_histogram) {
        // 打印误差分布直方图
        printf("Error distribution:\n");
        // 遍历直方图桶
        for (size_t i = 0; i < HISTOGRAM_BUCKETS; i++) {
            // 计算当前桶的上下界
            double lower = i * HISTOGRAM_RANGE / HISTOGRAM_BUCKETS;
            double upper = (i+1) * HISTOGRAM_RANGE / HISTOGRAM_BUCKETS;
            // 如果是最后一个桶，上界为无穷大
            if (i == HISTOGRAM_BUCKETS -1) upper = INFINITY;
            // 打印当前桶的范围和频数
            printf("[%3.4f, %3.4f): %11" PRIu64 "\n", lower, upper, stats.error_histogram[i]);
        }
    }
// 从 ggml.h 复制过来 - 验证我们是否可以将其作为一个平面数组访问
static bool tensor_is_contiguous(const struct ggml_tensor * tensor) {
    // 静态断言，确保 GGML_MAX_DIMS 等于 4，如果不是则更新此函数
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 检查张量是否是连续的
    return
        tensor->nb[0] == ggml_type_size(tensor->type) &&
        tensor->nb[1] == (tensor->nb[0]*tensor->ne[0])/ggml_blck_size(tensor->type) &&
        tensor->nb[2] == tensor->nb[1]*tensor->ne[1] &&
        tensor->nb[3] == tensor->nb[2]*tensor->ne[2];
}

// 在一个块上进行往返测试
static void test_roundtrip_on_chunk(
    const ggml_tensor * layer, int64_t offset, int64_t chunk_size, const ggml_type_traits_t & qfns, bool use_reference,
    float * input_scratch, char * quantized_scratch, float * output_scratch, error_stats & stats
) {
    // 如果层的类型是 GGML_TYPE_F16
    if (layer->type == GGML_TYPE_F16) {
        // 遍历块大小，将层中的数据复制到输入缓冲区
        for (int i = 0; i < chunk_size; i++) {
            input_scratch[i] = ggml_get_f32_1d(layer, i + offset);
        }
    } else {
        // 否则，将输入缓冲区指向层的数据
        input_scratch = ggml_get_data_f32(layer) + offset;
    }

    // 如果使用参考值
    if (use_reference) {
        // 使用参考值将输入缓冲区的数据转换为量化数据
        qfns.from_float_reference(input_scratch, quantized_scratch, chunk_size);
    } else {
        // 否则，使用量化函数将输入缓冲区的数据转换为量化数据
        qfns.from_float(input_scratch, quantized_scratch, chunk_size);
    }
    // 使用反量化函数将量化数据转换为浮点数数据
    qfns.to_float(quantized_scratch, output_scratch, chunk_size);

    // 更新错误统计信息
    update_error_stats(chunk_size, input_scratch, output_scratch, stats);
}

// 对单个层运行量化函数并更新错误统计信息
static void test_roundtrip_on_layer(
    std::string & name, bool print_layer_stats, const ggml_type_traits_t & qfns, bool use_reference,
    const ggml_tensor * layer, std::vector<float> & input_scratch, std::vector<char> & quantized_scratch,
    std::vector<float> & output_scratch, error_stats & total_error, int max_thread = 0
) {
    // 断言张量是连续的
    assert(tensor_is_contiguous(layer));
    // 初始化层错误统计信息
    error_stats layer_error {};
    // 获取层中元素的数量
    uint64_t nelements = ggml_nelements(layer);

    // 初始化输入缓冲区指针为空
    float* input_scratch_ptr = nullptr;
}
    # 如果层的类型是 GGML_TYPE_F16
    if (layer->type == GGML_TYPE_F16) {
        # 如果输入缓冲区的大小小于元素数量，则调整输入缓冲区的大小为元素数量
        if (input_scratch.size() < nelements) input_scratch.resize(nelements);
        # 将输入缓冲区的指针指向输入缓冲区的数据
        input_scratch_ptr = input_scratch.data();
    }
    # 如果量化缓冲区的大小小于 4*nelements，则调整量化缓冲区的大小为 4*nelements
    if (quantized_scratch.size() < 4*nelements) quantized_scratch.resize(4*nelements);
    # 如果输出缓冲区的大小小于元素数量，则调整输出缓冲区的大小为元素数量
    if (output_scratch.size() < nelements) output_scratch.resize(nelements);

    # 如果最大线程数小于 1，则将最大线程数设置为硬件并发数
    if (max_thread < 1) max_thread = std::thread::hardware_concurrency();
    # 设置块大小为 32*512
    int chunk_size = 32*512;
    # 计算块的数量，保证能够覆盖所有元素
    int num_chunks = (nelements + chunk_size - 1)/chunk_size;

    # 如果块的数量小于 2 或者最大线程数小于 2
    if (num_chunks < 2 || max_thread < 2) {
        # 在块上测试往返传输
        test_roundtrip_on_chunk(layer, 0, nelements, qfns, use_reference, input_scratch_ptr, quantized_scratch.data(),
                output_scratch.data(), print_layer_stats ? layer_error : total_error);
    } else {
        // 根据是否需要打印每一层的统计信息，选择相应的错误统计对象
        auto & stats = print_layer_stats ? layer_error : total_error;
        // 创建互斥锁
        std::mutex mutex;
        // 初始化计数器
        uint64_t counter = 0;
        // 定义并发执行的函数
        auto compute = [&mutex, &counter, &stats, &qfns, nelements, layer, use_reference, input_scratch_ptr,
             &quantized_scratch, &output_scratch, chunk_size] () {
            // 初始化本地错误统计对象
            error_stats local_stats {};
            // 循环执行直到所有元素都被处理
            while (true) {
                // 获取互斥锁
                std::unique_lock<std::mutex> lock(mutex);
                // 获取当前处理的起始位置，并更新计数器
                uint64_t offset = counter; counter += chunk_size;
                // 如果已经处理完所有元素，则合并本地错误统计到全局错误统计对象，并结束循环
                if (offset >= nelements) {
                    combine_error_stats(stats, local_stats);
                    break;
                }
                // 释放互斥锁
                lock.unlock();
                // 计算当前处理的数据块大小
                uint64_t chunk = offset + chunk_size < nelements ? chunk_size : nelements - offset;
                // 在数据块上执行测试往返，并更新本地错误统计
                test_roundtrip_on_chunk(layer, offset, chunk, qfns, use_reference, input_scratch_ptr + offset,
                        quantized_scratch.data() + 4*offset, output_scratch.data() + offset, local_stats);
            }
        };
        // 确定并发执行的线程数
        int nthread = std::min(num_chunks, max_thread);
        // 创建线程对象
        std::vector<std::thread> workers(nthread-1);
        // 启动并发执行的线程
        for (auto& w : workers) w = std::thread(compute);
        // 主线程也执行一次并发函数
        compute();
        // 等待所有线程执行完毕
        for (auto& w : workers) w.join();
    }

    // 如果需要打印每一层的统计信息，则打印并合并到总的错误统计对象中
    if (print_layer_stats) {
        print_error_stats(name, layer_error, false);
        combine_error_stats(total_error, layer_error);
    }
}

int main(int argc, char ** argv) {
    ggml_time_init();  // 初始化时间

    quantize_stats_params params;  // 定义参数结构体

    // 读取命令行参数

    int max_thread = 0;  // 最大线程数
    bool invalid_param = false;  // 参数是否有效
    std::string arg;  // 字符串参数
    }
    if (invalid_param) {  // 如果参数无效
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());  // 输出错误信息
        quantize_stats_print_usage(argc, argv);  // 打印用法
        return 1;  // 返回错误码
    }

    print_build_info();  // 打印构建信息

    // 加载模型
    fprintf(stderr, "Loading model\n");  // 输出加载模型信息

    const int64_t t_main_start_us = ggml_time_us();  // 获取当前时间

    llama_model * model;  // 定义模型指针
    llama_context * ctx;  // 定义上下文指针

    {
        auto mparams = llama_model_default_params();  // 获取默认模型参数
        mparams.use_mlock  = false;  // 设置模型参数

        model = llama_load_model_from_file(params.model.c_str(), mparams);  // 从文件加载模型

        if (model == NULL) {  // 如果模型为空
            fprintf(stderr, "%s: error: failed to load model '%s'\n", __func__, params.model.c_str());  // 输出错误信息
            return 1;  // 返回错误码
        }

        auto cparams = llama_context_default_params();  // 获取默认上下文参数
        cparams.n_ctx      = 256;  // 设置上下文参数
        cparams.seed       = 1;  // 设置上下文参数
        cparams.f16_kv     = false;  // 设置上下文参数

        ctx = llama_new_context_with_model(model, cparams);  // 创建带有模型的上下文

        if (ctx == NULL) {  // 如果上下文为空
            fprintf(stderr, "%s: error: failed to create context with model '%s'\n", __func__, params.model.c_str());  // 输出错误信息
            llama_free_model(model);  // 释放模型内存
            return 1;  // 返回错误码
        }
    }

    const auto &tensors = llama_internal_get_tensor_map(ctx);  // 获取张量映射

    // 检查层张量
    int included_layers = 0;  // 包含的层数
    int64_t max_nelements = 0;  // 最大元素数
    bool is_f16 = false;  // 是否为f16类型
    // 遍历tensors中的每个键值对
    for (const auto& kv_tensor : tensors) {
        // 如果params中不包含kv_tensor的键，则跳过当前循环
        if (!layer_included(params, kv_tensor.first)) {
            continue;
        }
        // 如果params中设置了verbose为true，则打印kv_tensor的类型和大小
        if (params.verbose) {
            printf("%s: type %s, size %" PRId64 "\n", kv_tensor.first.c_str(), ggml_type_name(kv_tensor.second->type), ggml_nelements(kv_tensor.second));
        }
        // 如果kv_tensor的类型为GGML_TYPE_F16，则将is_f16设置为true
        if (kv_tensor.second->type == GGML_TYPE_F16) {
            is_f16 = true;
        } 
        // 如果kv_tensor的类型不是GGML_TYPE_F32，则打印错误信息并返回1
        else if (kv_tensor.second->type != GGML_TYPE_F32) {
            fprintf(stderr, "%s: error: Quantization should be tested with a float model, "
                "this model contains already quantized layers (%s is type %d)\n", __func__, kv_tensor.first.c_str(), kv_tensor.second->type);
            llama_free(ctx);
            llama_free_model(model);
            return 1;
        }
        // 记录已包含的层数
        included_layers++;
        // 更新max_nelements为当前kv_tensor的元素个数和max_nelements中的较大值
        max_nelements = std::max(max_nelements, ggml_nelements(kv_tensor.second));
    }

    // 如果is_f16为true，则打印提示信息
    if (is_f16) {
        printf("note: source model is f16\n");
    }
    // 打印测试的层数和最大大小
    printf("testing %d layers with max size %" PRId64 "\n", included_layers, max_nelements);
    // 分配临时空间
    std::vector<float> input_scratch;
    std::vector<char> quantized_scratch;
    std::vector<float> output_scratch;

    // 循环遍历量化类型
    // 遍历 GGML_TYPE_COUNT 范围内的所有类型
    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        // 将当前索引转换为 ggml_type 类型
        const ggml_type type = (ggml_type) i;
        // 如果参数中包含需要的类型，并且当前类型不在其中，则跳过当前循环
        if (!params.include_types.empty() && std::find(params.include_types.begin(), params.include_types.end(), i) == params.include_types.end()) {
            continue;
        }
        // 获取当前类型的类型特性
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);
        // 如果当前类型支持从浮点数到量化数的转换以及从量化数到浮点数的转换
        if (qfns.from_float && qfns.to_float) {
            // 如果参数中设置了详细输出，则打印当前类型的测试信息
            if (params.verbose) {
                printf("testing %s ...\n",  ggml_type_name(type));
            }

            // 初始化全局错误统计
            error_stats global_stats {};

            // 遍历所有张量
            for (const auto& kv_tensor : tensors) {
                // 如果当前层不在参数指定的层中，则跳过当前循环
                if (!layer_included(params, kv_tensor.first)) {
                    continue;
                }
                // 如果参数中设置了详细输出，则打印当前张量的信息
                if (params.verbose) {
                    printf("  %s ...\n",  kv_tensor.first.c_str());
                }
                // 构建层名
                std::string layer_name { ggml_type_name(type) };
                layer_name += "::" + kv_tensor.first;
                // 在当前层上进行往返测试
                test_roundtrip_on_layer(
                        layer_name,
                        params.per_layer_stats,
                        qfns,
                        params.reference,
                        kv_tensor.second,
                        input_scratch,
                        quantized_scratch,
                        output_scratch,
                        global_stats,
                        max_thread
                );
            }

            // 打印当前类型的错误统计信息
            print_error_stats(ggml_type_name(type), global_stats, params.print_histogram);
        }
    }

    // 释放上下文内存
    llama_free(ctx);
    // 释放模型内存
    llama_free_model(model);
    // 报告运行时间
    {
        // 获取当前时间
        const int64_t t_main_end_us = ggml_time_us();
        // 打印总运行时间
        printf("\n");
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0);
    }

    // 返回成功
    return 0;
# 闭合之前的函数定义
```