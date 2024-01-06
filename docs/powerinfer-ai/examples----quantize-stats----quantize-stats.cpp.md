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

// 引入标准库头文件
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
// 如果是在 MSC 编译器下，禁止特定警告
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

// 定义量化统计参数结构体
struct quantize_stats_params {
    std::string model = "models/7B/ggml-model-f16.gguf"; // 默认模型路径
    bool verbose = false; // 是否输出详细信息
    bool per_layer_stats = false; // 是否输出每层的统计信息
    bool print_histogram = false; // 是否输出直方图
    bool reference = false; // 是否使用参考值
    std::vector<std::string> include_layers; // 包含的层
    std::vector<std::string> exclude_layers; // 排除的层
    std::vector<enum ggml_type> include_types; // 包含的类型
};

// 直方图桶的数量
constexpr size_t HISTOGRAM_BUCKETS = 150;
// 直方图范围
constexpr double HISTOGRAM_RANGE = 0.03;

// 错误统计结构体
struct error_stats {
    size_t num_samples; // 样本数量
    // 定义两个双精度浮点型变量，用于存储总误差和最大误差
    double total_error;
    double max_error;
    // 定义一个包含 HISTOGRAM_BUCKETS 个元素的无符号64位整型数组，用于存储误差直方图
    uint64_t error_histogram[HISTOGRAM_BUCKETS];
};

// 定义一个静态函数，用于打印使用说明
static void quantize_stats_print_usage(int /*argc*/, char ** argv) {
    // 定义一个 quantize_stats_params 结构体变量
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
    // 其他选项的说明
}
// 打印错误直方图的选项说明
fprintf(stderr, "                        print error histogram (default: false)\n");
// 包含特定层的选项说明
fprintf(stderr, "  -l LAYER, --include-layer LAYER\n");
// 排除特定层的选项说明
fprintf(stderr, "  -L LAYER, --exclude-layer LAYER\n");
// 只测试给定类型的选项说明
fprintf(stderr, "  -t TYPE, --type TYPE\n");
// 空行
fprintf(stderr, "\n");
}

// 检查命令行中是否包含/排除了特定层
static bool layer_included(const quantize_stats_params & params, const std::string & layer) {
    // 遍历排除的层列表，如果层名匹配则返回false
    for (const auto& excluded : params.exclude_layers) {
        if (std::regex_search(layer, std::regex(excluded))) {
            return false;
        }
    }
    // 遍历包含的层列表，如果层名匹配则返回true
    for (const auto& included : params.include_layers) {
        if (std::regex_search(layer, std::regex(included))) {
            return true;
        }
    }
// 检查是否参数 params 中的 include_layers 为空，如果是则返回 true，否则返回 false
}
}
return params.include_layers.empty();
}

// 根据量化前后结果的向量更新错误统计信息
static void update_error_stats(int64_t nelements, const float * input, const float * output, error_stats & stats) {
    // 遍历输入和输出向量，计算差值的平方并累加到总误差中
    for (int64_t i = 0; i < nelements; i++) {
        double diff = input[i] - output[i];
        stats.total_error += diff * diff;
        // 更新最大误差
        stats.max_error = fmax(fabs(diff), stats.max_error);
        // 更新误差直方图
        stats.error_histogram[std::max(std::min((size_t) floor(fabs(diff) / HISTOGRAM_RANGE * HISTOGRAM_BUCKETS), HISTOGRAM_BUCKETS-1), (size_t) 0)]++;
    }
    // 更新样本数量
    stats.num_samples += nelements;
}

// 合并错误统计信息
static void combine_error_stats(error_stats & into, const error_stats & from) {
    // 合并样本数量
    into.num_samples += from.num_samples;
    // 合并总误差
    into.total_error += from.total_error;
    // 如果 from 的最大误差大于 into 的最大误差，则更新 into 的最大误差
    if (from.max_error > into.max_error) into.max_error = from.max_error;
// 将from的error_histogram数组中的值累加到into的error_histogram数组中
for (size_t i=0; i<HISTOGRAM_BUCKETS; ++i) into.error_histogram[i] += from.error_histogram[i];
}

// 查找给定分位数的值
static double find_quantile(const error_stats & stats, double quantile) {
    // 计算error_histogram数组中所有值的总和
    double sum = std::accumulate(std::begin(stats.error_histogram), std::end(stats.error_histogram), 0.0);

    double accum = 0;
    // 遍历error_histogram数组
    for (size_t i = 0; i < HISTOGRAM_BUCKETS; i++) {
        accum += stats.error_histogram[i];
        // 当累积值超过总和的quantile时，返回对应的分位数值
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
    // 使用 printf 函数打印出给定格式的字符串，包括名称、均方根误差、最大误差、95% 百分位以下的值和中位数
    printf("%-50s: rmse %.8f, maxerr %.8f, 95pct<%.4f, median<%.4f\n", name.c_str(), rmse, stats.max_error, pct95, median);
    // 如果需要打印误差分布直方图
    if (print_histogram) {
        // 打印误差分布的区间和对应的频数
        printf("Error distribution:\n");
        for (size_t i = 0; i < HISTOGRAM_BUCKETS; i++) {
            // 计算当前区间的下限和上限
            double lower = i * HISTOGRAM_RANGE / HISTOGRAM_BUCKETS;
            double upper = (i+1) * HISTOGRAM_RANGE / HISTOGRAM_BUCKETS;
            // 如果是最后一个区间，上限为无穷大
            if (i == HISTOGRAM_BUCKETS -1) upper = INFINITY;
            // 打印当前区间的下限、上限和频数
            printf("[%3.4f, %3.4f): %11" PRIu64 "\n", lower, upper, stats.error_histogram[i]);
        }
    }
}

// 从 ggml.h 复制过来的函数，用于验证是否可以将张量视为一个平坦数组
static bool tensor_is_contiguous(const struct ggml_tensor * tensor) {
    // 静态断言，确保 GGML_MAX_DIMS 的值为 4
    static_assert(GGML_MAX_DIMS == 4, "GGML_MAX_DIMS is not 4 - update this function");

    // 返回张量是否是连续的
    return
        tensor->nb[0] == ggml_type_size(tensor->type) &&
        tensor->nb[1] == (tensor->nb[0]*tensor->ne[0])/ggml_blck_size(tensor->type) &&
        tensor->nb[2] == tensor->nb[1]*tensor->ne[1] &&
```

// 检查第三个维度的大小是否等于第二个维度的大小乘以第二个维度的元素个数
tensor->nb[3] == tensor->nb[2]*tensor->ne[2];

// 在给定的数据块上进行往返测试
static void test_roundtrip_on_chunk(
    const ggml_tensor * layer, int64_t offset, int64_t chunk_size, const ggml_type_traits_t & qfns, bool use_reference,
    float * input_scratch, char * quantized_scratch, float * output_scratch, error_stats & stats
) {
    // 如果层的类型是 GGML_TYPE_F16，则将数据从半精度浮点数转换为单精度浮点数
    if (layer->type == GGML_TYPE_F16) {
        for (int i = 0; i < chunk_size; i++) {
            input_scratch[i] = ggml_get_f32_1d(layer, i + offset);
        }
    } else {
        // 否则直接获取单精度浮点数数据
        input_scratch = ggml_get_data_f32(layer) + offset;
    }

    // 根据是否使用参考值，将输入数据转换为量化数据
    if (use_reference) {
        qfns.from_float_reference(input_scratch, quantized_scratch, chunk_size);
    } else {
        qfns.from_float(input_scratch, quantized_scratch, chunk_size);
    }
}
// 调用to_float函数将quantized_scratch中的数据转换为浮点数，并存储到output_scratch中，chunk_size为每次处理的数据块大小
qfns.to_float(quantized_scratch, output_scratch, chunk_size);

// 更新错误统计信息，包括输入数据、输出数据、统计信息
update_error_stats(chunk_size, input_scratch, output_scratch, stats);
}


// 对单个层运行量化函数并更新错误统计信息
static void test_roundtrip_on_layer(
    std::string & name, bool print_layer_stats, const ggml_type_traits_t & qfns, bool use_reference,
    const ggml_tensor * layer, std::vector<float> & input_scratch, std::vector<char> & quantized_scratch,
    std::vector<float> & output_scratch, error_stats & total_error, int max_thread = 0
) {
    // 断言确保tensor是连续的
    assert(tensor_is_contiguous(layer));
    // 初始化层错误统计信息
    error_stats layer_error {};
    // 获取tensor中元素的数量
    uint64_t nelements = ggml_nelements(layer);

    // 初始化输入数据的指针
    float* input_scratch_ptr = nullptr;
    // 如果层的类型是GGML_TYPE_F16，且输入数据的大小小于元素数量，则调整输入数据的大小
    if (layer->type == GGML_TYPE_F16) {
        if (input_scratch.size() < nelements) input_scratch.resize(nelements);
        input_scratch_ptr = input_scratch.data();
    // 如果量化后的临时数据大小小于4*nelements，则调整大小为4*nelements
    if (quantized_scratch.size() < 4*nelements) quantized_scratch.resize(4*nelements);
    // 如果输出临时数据大小小于nelements，则调整大小为nelements
    if (output_scratch.size() < nelements) output_scratch.resize(nelements);

    // 如果最大线程数小于1，则设置为硬件支持的线程数
    if (max_thread < 1) max_thread = std::thread::hardware_concurrency();
    // 设置每个块的大小为32*512
    int chunk_size = 32*512;
    // 计算需要处理的块数
    int num_chunks = (nelements + chunk_size - 1)/chunk_size;

    // 如果块数小于2或最大线程数小于2，则在单线程下测试往返传输
    if (num_chunks < 2 || max_thread < 2) {
        test_roundtrip_on_chunk(layer, 0, nelements, qfns, use_reference, input_scratch_ptr, quantized_scratch.data(),
                output_scratch.data(), print_layer_stats ? layer_error : total_error);
    } else {
        // 否则，使用多线程进行测试
        auto & stats = print_layer_stats ? layer_error : total_error;
        // 创建互斥锁和计数器
        std::mutex mutex;
        uint64_t counter = 0;
        // 定义计算函数
        auto compute = [&mutex, &counter, &stats, &qfns, nelements, layer, use_reference, input_scratch_ptr,
             &quantized_scratch, &output_scratch, chunk_size] () {
            error_stats local_stats {};
            // 循环处理数据
            while (true) {
                // 获取互斥锁
                std::unique_lock<std::mutex> lock(mutex);
// 设置偏移量为当前计数器的值，然后增加块大小到计数器
uint64_t offset = counter; counter += chunk_size;
// 如果偏移量大于等于元素数量，则合并错误统计并跳出循环
if (offset >= nelements) {
    combine_error_stats(stats, local_stats);
    break;
}
// 解锁
lock.unlock();
// 计算当前块的大小
uint64_t chunk = offset + chunk_size < nelements ? chunk_size : nelements - offset;
// 在多线程环境下测试当前块的往返传输
test_roundtrip_on_chunk(layer, offset, chunk, qfns, use_reference, input_scratch_ptr + offset,
                        quantized_scratch.data() + 4*offset, output_scratch.data() + offset, local_stats);
// 定义并启动多个线程
};
int nthread = std::min(num_chunks, max_thread);
std::vector<std::thread> workers(nthread-1);
for (auto& w : workers) w = std::thread(compute);
compute();
for (auto& w : workers) w.join();
}

// 如果需要打印层的统计信息，则打印错误统计
if (print_layer_stats) {
    print_error_stats(name, layer_error, false);
// 调用 combine_error_stats 函数，传入 total_error 和 layer_error 作为参数
combine_error_stats(total_error, layer_error);
}

// 主函数
int main(int argc, char ** argv) {
    // 初始化时间
    ggml_time_init();

    // 创建 quantize_stats_params 对象
    quantize_stats_params params;

    // 读取命令行参数
    int max_thread = 0; // 最大线程数
    bool invalid_param = false; // 无效参数标志
    std::string arg; // 命令行参数
    for (int i = 1; i < argc; i++) { // 遍历命令行参数
        arg = argv[i]; // 获取当前参数

        if (arg == "-h" || arg == "--help") { // 如果参数为 -h 或 --help
            quantize_stats_print_usage(argc, argv); // 调用 quantize_stats_print_usage 函数打印用法
            exit(0); // 退出程序
        } else if (arg == "-r" || arg == "--reference") {
            // 如果参数是"-r"或"--reference"，则设置参数中的参考标志为true
            params.reference = true;
        } else if (arg == "-v") {
            // 如果参数是"-v"，则设置参数中的详细信息标志为true
            params.verbose = true;
        } else if (arg == "-p" || arg == "--per-layer-stats") {
            // 如果参数是"-p"或"--per-layer-stats"，则设置参数中的每层统计标志为true
            params.per_layer_stats = true;
        } else if (arg == "--histogram") {
            // 如果参数是"--histogram"，则设置参数中的打印直方图标志为true
            params.print_histogram = true;
        } else if (arg == "-m" || arg == "--model") {
            // 如果参数是"-m"或"--model"，则检查下一个参数是否存在，如果不存在则设置无效参数标志为true，否则将下一个参数作为模型参数
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.model = argv[i];
        } else if (arg == "-l" || arg == "--include-layer") {
            // 如果参数是"-l"或"--include-layer"，则检查下一个参数是否存在，如果不存在则设置无效参数标志为true，否则将下一个参数添加到参数中的包含层列表中
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            params.include_layers.push_back(argv[i]);
// 如果参数为"-L"或"--exclude-layer"，则将下一个参数加入到排除图层的列表中
} else if (arg == "-L" || arg == "--exclude-layer") {
    // 检查下一个参数是否存在，如果不存在则标记为无效参数并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    // 将下一个参数加入到排除图层的列表中
    params.exclude_layers.push_back(argv[i]);
} else if (arg == "-t" || arg == "--type") {
    // 如果参数为"-t"或"--type"，则将下一个参数作为类型进行处理
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    int j;
    // 遍历所有类型，查找与参数匹配的类型
    for (j = 0; j < GGML_TYPE_COUNT; ++j) {
       const auto * name = ggml_type_name((ggml_type) j);
       // 如果找到匹配的类型，则将其加入到包含类型的列表中
       if (name && strcmp(argv[i], name) == 0) break;
    }
    // 如果找到匹配的类型，则将其加入到包含类型的列表中，否则输出错误信息
    if (j < GGML_TYPE_COUNT) {
        params.include_types.push_back((ggml_type) j);
    } else {
        fprintf(stderr, "error: %s not in list of types\n", argv[i]);
    }
        // 初始化参数无效标志
        invalid_param = true;
    }
} else if (arg == "-n" || arg == "--num-threads") {
    // 如果参数是"-n"或"--num-threads"
    if (++i >= argc) {
        // 如果下一个参数超出范围
        invalid_param = true;
        // 退出循环
        break;
    }
    // 将下一个参数转换为整数并赋给max_thread
    max_thread = atoi(argv[i]);
} else {
    // 如果参数不是上述两种情况，打印错误信息
    fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
    // 打印用法信息
    quantize_stats_print_usage(argc, argv);
    // 返回错误代码
    return 1;
}
// 如果存在无效参数
if (invalid_param) {
    // 打印无效参数错误信息
    fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
    // 打印用法信息
    quantize_stats_print_usage(argc, argv);
    // 返回错误代码
    return 1;
}
    // 打印构建信息
    print_build_info();

    // 加载模型
    fprintf(stderr, "Loading model\n");

    // 获取当前时间
    const int64_t t_main_start_us = ggml_time_us();
    llama_model * model;
    llama_context * ctx;

    {
        // 获取默认的模型参数
        auto mparams = llama_model_default_params();
        mparams.use_mlock  = false;

        // 从文件加载模型
        model = llama_load_model_from_file(params.model.c_str(), mparams);

        // 如果加载失败，打印错误信息并返回
        if (model == NULL) {
            fprintf(stderr, "%s: error: failed to load model '%s'\n", __func__, params.model.c_str());
            return 1;
        }
// 使用默认参数创建LLAMA上下文参数对象
auto cparams = llama_context_default_params();
// 设置上下文参数对象的上下文数为256
cparams.n_ctx = 256;
// 设置上下文参数对象的种子为1
cparams.seed = 1;
// 设置上下文参数对象的f16_kv为false
cparams.f16_kv = false;

// 使用给定模型和上下文参数对象创建LLAMA上下文对象
ctx = llama_new_context_with_model(model, cparams);

// 如果上下文对象创建失败，则输出错误信息并释放模型资源，然后返回1
if (ctx == NULL) {
    fprintf(stderr, "%s: error: failed to create context with model '%s'\n", __func__, params.model.c_str());
    llama_free_model(model);
    return 1;
}

// 获取LLAMA上下文对象中的张量映射
const auto &tensors = llama_internal_get_tensor_map(ctx);

// 检查层张量
int included_layers = 0;
int64_t max_nelements = 0;
bool is_f16 = false;
// 遍历 tensors 容器中的每个键值对
for (const auto& kv_tensor : tensors) {
    // 如果 params 中不包含 kv_tensor 的键值对，则跳过当前循环
    if (!layer_included(params, kv_tensor.first)) {
        continue;
    }
    // 如果 params 中设置了 verbose 标志，则打印输出键值对的信息
    if (params.verbose) {
        printf("%s: type %s, size %" PRId64 "\n", kv_tensor.first.c_str(), ggml_type_name(kv_tensor.second->type), ggml_nelements(kv_tensor.second));
    }
    // 如果键值对对应的值的类型为 GGML_TYPE_F16，则设置 is_f16 为 true
    if (kv_tensor.second->type == GGML_TYPE_F16) {
        is_f16 = true;
    } 
    // 如果键值对对应的值的类型不为 GGML_TYPE_F32，则输出错误信息并返回
    else if (kv_tensor.second->type != GGML_TYPE_F32) {
        fprintf(stderr, "%s: error: Quantization should be tested with a float model, "
            "this model contains already quantized layers (%s is type %d)\n", __func__, kv_tensor.first.c_str(), kv_tensor.second->type);
        llama_free(ctx);
        llama_free_model(model);
        return 1;
    }
    // 统计包含的层数量
    included_layers++;
    // 更新最大元素数量
    max_nelements = std::max(max_nelements, ggml_nelements(kv_tensor.second));
}
    // 如果是 f16 类型的模型，打印提示信息
    if (is_f16) {
        printf("note: source model is f16\n");
    }
    // 打印测试信息，包括层数和最大尺寸
    printf("testing %d layers with max size %" PRId64 "\n", included_layers, max_nelements);
    // 分配临时空间
    std::vector<float> input_scratch;
    std::vector<char> quantized_scratch;
    std::vector<float> output_scratch;

    // 循环遍历量化类型
    for (int i = 0; i < GGML_TYPE_COUNT; i++) {
        // 获取当前类型
        const ggml_type type = (ggml_type) i;
        // 如果指定了包含的类型，并且当前类型不在其中，则跳过
        if (!params.include_types.empty() && std::find(params.include_types.begin(), params.include_types.end(), i) == params.include_types.end()) {
            continue;
        }
        // 获取当前类型的量化函数
        ggml_type_traits_t qfns = ggml_internal_get_type_traits(type);
        // 如果存在从浮点数到当前类型的转换函数和从当前类型到浮点数的转换函数
        if (qfns.from_float && qfns.to_float) {
            // 如果设置了详细输出，打印当前类型的测试信息
            if (params.verbose) {
                printf("testing %s ...\n",  ggml_type_name(type));
            }
// 创建一个名为error_stats的全局统计数据结构
error_stats global_stats {};

// 遍历tensors中的每个键值对
for (const auto& kv_tensor : tensors) {
    // 如果该层不在params中指定的层中，则跳过当前循环
    if (!layer_included(params, kv_tensor.first)) {
        continue;
    }
    // 如果params中设置了verbose标志，则打印当前处理的张量名
    if (params.verbose) {
        printf("  %s ...\n",  kv_tensor.first.c_str());
    }
    // 创建一个名为layer_name的字符串，用于存储类型名称和张量名的组合
    std::string layer_name { ggml_type_name(type) };
    layer_name += "::" + kv_tensor.first;
    // 在该层上执行测试往返操作，传入相关参数和数据
    test_roundtrip_on_layer(
            layer_name,
            params.per_layer_stats,
            qfns,
            params.reference,
            kv_tensor.second,
            input_scratch,
            quantized_scratch,
    // 调用模型训练函数，传入输入数据、输出数据、全局统计数据和最大线程数
    train_model(
        input_data,
        output_scratch,
        global_stats,
        max_thread
    );
}

// 打印错误统计信息
print_error_stats(ggml_type_name(type), global_stats, params.print_histogram);
}

// 释放上下文和模型内存
llama_free(ctx);
llama_free_model(model);

// 报告时间
{
    // 获取主函数结束时间
    const int64_t t_main_end_us = ggml_time_us();

    // 打印总时间
    printf("\n");
    printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0);
}
# 返回整数值0
return 0;
```