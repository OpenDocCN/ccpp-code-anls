# `PowerInfer\examples\llama-bench\llama-bench.cpp`

```
// 包含必要的头文件
#include <algorithm>
#include <array>
#include <cassert>
#include <chrono>
#include <cinttypes>
#include <clocale>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <ctime>
#include <iterator>
#include <map>
#include <numeric>
#include <regex>
#include <sstream>
#include <string>
#include <vector>

// 包含自定义的头文件
#include "ggml.h"
#include "llama.h"
#include "common.h"
#include "ggml-cuda.h"

// 定义一个静态函数，用于获取当前时间的纳秒数
static uint64_t get_time_ns() {
    using clock = std::chrono::high_resolution_clock;
    return std::chrono::nanoseconds(clock::now().time_since_epoch()).count();
}

// 定义一个模板函数，用于将 vector 中的元素用指定的分隔符连接成字符串
template<class T>
static std::string join(const std::vector<T> & values, const std::string & delim) {
    std::ostringstream str;
    for (size_t i = 0; i < values.size(); i++) {
        str << values[i];
        if (i < values.size() - 1) {
            str << delim;
        }
    }
    return str.str();
}

// 定义一个模板函数，用于将字符串按指定分隔符分割成 vector
template<class T>
static std::vector<T> split(const std::string & str, char delim) {
    std::vector<T> values;
    std::istringstream str_stream(str);
    std::string token;
    while (std::getline(str_stream, token, delim)) {
        T value;
        std::istringstream token_stream(token);
        token_stream >> value;
        values.push_back(value);
    }
    return values;
}

// 定义一个模板函数，用于计算 vector 中元素的平均值
template<typename T>
static T avg(const std::vector<T> & v) {
    if (v.empty()) {
        return 0;
    }
    T sum = std::accumulate(v.begin(), v.end(), T(0));
    return sum / (T)v.size();
}

// 定义一个模板函数，用于计算 vector 中元素的标准差
template<typename T>
static T stdev(const std::vector<T> & v) {
    if (v.size() <= 1) {
        return 0;
    }
    T mean = avg(v);
    T sq_sum = std::inner_product(v.begin(), v.end(), v.begin(), T(0));
    T stdev = std::sqrt(sq_sum / (T)(v.size() - 1) - mean * mean * (T)v.size() / (T)(v.size() - 1));
    return stdev;
}

// 定义一个静态函数，用于获取 CPU 信息
static std::string get_cpu_info() {
    std::string id;
    // 如果是 Linux 系统
#ifdef __linux__
    // 打开 /proc/cpuinfo 文件
    FILE * f = fopen("/proc/cpuinfo", "r");
    # 如果文件指针存在
    if (f) {
        # 定义一个缓冲区，用于存储读取的内容
        char buf[1024];
        # 逐行读取文件内容，直到文件结束
        while (fgets(buf, sizeof(buf), f)) {
            # 如果当前行以 "model name" 开头
            if (strncmp(buf, "model name", 10) == 0) {
                # 查找当前行中的冒号位置
                char * p = strchr(buf, ':');
                # 如果找到冒号
                if (p) {
                    # 将指针移动到冒号后面的第一个非空白字符
                    p++;
                    while (std::isspace(*p)) {
                        p++;
                    }
                    # 移除字符串末尾的空白字符
                    while (std::isspace(p[strlen(p) - 1])) {
                        p[strlen(p) - 1] = '\0';
                    }
                    # 将处理后的字符串赋值给 id，并结束循环
                    id = p;
                    break;
                }
            }
        }
    }
#endif
    // TODO: other platforms
    return id;
}

static std::string get_gpu_info() {
    std::string id;
#ifdef GGML_USE_CUBLAS
    // 获取 CUDA 设备数量
    int count = ggml_cuda_get_device_count();
    // 遍历每个 CUDA 设备，获取设备描述信息并添加到 id 中
    for (int i = 0; i < count; i++) {
        char buf[128];
        ggml_cuda_get_device_description(i, buf, sizeof(buf));
        id += buf;
        // 如果不是最后一个设备，则添加分隔符 "/"
        if (i < count - 1) {
            id += "/";
        }
    }
#endif
    // TODO: other backends
    return id;
}

// command line params
enum output_formats {CSV, JSON, MARKDOWN, SQL};

struct cmd_params {
    std::vector<std::string> model;
    std::vector<int> n_prompt;
    std::vector<int> n_gen;
    std::vector<int> n_batch;
    std::vector<bool> f32_kv;
    std::vector<int> n_threads;
    std::vector<int> n_gpu_layers;
    std::vector<int> main_gpu;
    std::vector<bool> mul_mat_q;
    std::vector<std::array<float, LLAMA_MAX_DEVICES>> tensor_split;
    int reps;
    bool verbose;
    output_formats output_format;
};

// 默认的命令行参数
static const cmd_params cmd_params_defaults = {
    /* model         */ {"models/7B/ggml-model-q4_0.gguf"},
    /* n_prompt      */ {512},
    /* n_gen         */ {128},
    /* n_batch       */ {512},
    /* f32_kv        */ {false},
    /* n_threads     */ {get_num_physical_cores()},
    /* n_gpu_layers  */ {99},
    /* main_gpu      */ {0},
    /* mul_mat_q     */ {true},
    /* tensor_split  */ {{}},
    /* reps          */ 5,
    /* verbose       */ false,
    /* output_format */ MARKDOWN
};

// 打印用法信息
static void print_usage(int /* argc */, char ** argv) {
    printf("usage: %s [options]\n", argv[0]);
    printf("\n");
    printf("options:\n");
    printf("  -h, --help\n");
    printf("  -m, --model <filename>            (default: %s)\n", join(cmd_params_defaults.model, ",").c_str());
    printf("  -p, --n-prompt <n>                (default: %s)\n", join(cmd_params_defaults.n_prompt, ",").c_str());
    printf("  -n, --n-gen <n>                   (default: %s)\n", join(cmd_params_defaults.n_gen, ",").c_str());
    # 打印带有默认值的批处理大小参数
    printf("  -b, --batch-size <n>              (default: %s)\n", join(cmd_params_defaults.n_batch, ",").c_str());
    # 打印带有默认值的内存类型参数
    printf("  --memory-f32 <0|1>                (default: %s)\n", join(cmd_params_defaults.f32_kv, ",").c_str());
    # 打印带有默认值的线程数参数
    printf("  -t, --threads <n>                 (default: %s)\n", join(cmd_params_defaults.n_threads, ",").c_str());
    # 打印带有默认值的 GPU 层数参数
    printf("  -ngl, --n-gpu-layers <n>          (default: %s)\n", join(cmd_params_defaults.n_gpu_layers, ",").c_str());
    # 打印带有默认值的主 GPU 参数
    printf("  -mg, --main-gpu <i>               (default: %s)\n", join(cmd_params_defaults.main_gpu, ",").c_str());
    # 打印带有默认值的矩阵乘法参数
    printf("  -mmq, --mul-mat-q <0|1>           (default: %s)\n", join(cmd_params_defaults.mul_mat_q, ",").c_str());
    # 打印张量分割参数
    printf("  -ts, --tensor_split <ts0/ts1/..>               \n");
    # 打印带有默认值的重复次数参数
    printf("  -r, --repetitions <n>             (default: %d)\n", cmd_params_defaults.reps);
    # 打印带有默认值的输出格式参数
    printf("  -o, --output <csv|json|md|sql>    (default: %s)\n", cmd_params_defaults.output_format == CSV ? "csv" : cmd_params_defaults.output_format == JSON ? "json" : cmd_params_defaults.output_format == MARKDOWN ? "md" : "sql");
    # 打印带有默认值的详细信息参数
    printf("  -v, --verbose                     (default: %s)\n", cmd_params_defaults.verbose ? "1" : "0");
    # 打印空行
    printf("\n");
    # 提示可以通过逗号分隔或多次指定参数来给每个参数指定多个值
    printf("Multiple values can be given for each parameter by separating them with ',' or by specifying the parameter multiple times.\n");
// 解析命令行参数，返回参数结构体
static cmd_params parse_cmd_params(int argc, char ** argv) {
    // 定义命令行参数结构体
    cmd_params params;
    // 定义字符串变量和布尔变量
    std::string arg;
    bool invalid_param = false;
    // 定义参数前缀和分隔符
    const std::string arg_prefix = "--";
    const char split_delim = ',';

    // 设置默认参数
    params.verbose = cmd_params_defaults.verbose;
    params.output_format = cmd_params_defaults.output_format;
    params.reps = cmd_params_defaults.reps;

    // 检查是否有无效参数
    if (invalid_param) {
        // 输出错误信息并打印用法
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        print_usage(argc, argv);
        // 退出程序
        exit(1);
    }

    // 设置默认值
    if (params.model.empty())        { params.model = cmd_params_defaults.model; }
    if (params.n_prompt.empty())     { params.n_prompt = cmd_params_defaults.n_prompt; }
    if (params.n_gen.empty())        { params.n_gen = cmd_params_defaults.n_gen; }
    if (params.n_batch.empty())      { params.n_batch = cmd_params_defaults.n_batch; }
    if (params.f32_kv.empty())       { params.f32_kv = cmd_params_defaults.f32_kv; }
    if (params.n_gpu_layers.empty()) { params.n_gpu_layers = cmd_params_defaults.n_gpu_layers; }
    if (params.main_gpu.empty())     { params.main_gpu = cmd_params_defaults.main_gpu; }
    if (params.mul_mat_q.empty())    { params.mul_mat_q = cmd_params_defaults.mul_mat_q; }
    if (params.tensor_split.empty()) { params.tensor_split = cmd_params_defaults.tensor_split; }
    if (params.n_threads.empty())    { params.n_threads = cmd_params_defaults.n_threads; }

    // 返回参数结构体
    return params;
}

// 定义命令行参数实例结构体
struct cmd_params_instance {
    std::string model;
    int n_prompt;
    int n_gen;
    int n_batch;
    bool f32_kv;
    int n_threads;
    int n_gpu_layers;
    int main_gpu;
    bool mul_mat_q;
    std::array<float, LLAMA_MAX_DEVICES> tensor_split;
    # 将当前对象的参数转换为LLAMA模型参数并返回
    llama_model_params to_llama_mparams() const {
        # 创建LLAMA模型默认参数对象
        llama_model_params mparams = llama_model_default_params();

        # 设置LLAMA模型参数的GPU层数
        mparams.n_gpu_layers = n_gpu_layers;
        # 设置LLAMA模型参数的主GPU
        mparams.main_gpu = main_gpu;
        # 设置LLAMA模型参数的张量分割
        mparams.tensor_split = tensor_split.data();

        # 返回LLAMA模型参数
        return mparams;
    }

    # 比较当前对象的参数与另一个cmd_params_instance对象的参数是否相等
    bool equal_mparams(const cmd_params_instance & other) const {
        # 返回模型、GPU层数、主GPU和张量分割是否相等
        return model == other.model &&
               n_gpu_layers == other.n_gpu_layers &&
               main_gpu == other.main_gpu &&
               tensor_split == other.tensor_split;
    }

    # 将当前对象的上下文参数转换为LLAMA上下文参数并返回
    llama_context_params to_llama_cparams() const {
        # 创建LLAMA上下文默认参数对象
        llama_context_params cparams = llama_context_default_params();

        # 设置LLAMA上下文参数的上下文大小
        cparams.n_ctx = n_prompt + n_gen;
        # 设置LLAMA上下文参数的批次大小
        cparams.n_batch = n_batch;
        # 设置LLAMA上下文参数的f16_kv
        cparams.f16_kv = !f32_kv;
        # 设置LLAMA上下文参数的mul_mat_q
        cparams.mul_mat_q = mul_mat_q;

        # 返回LLAMA上下文参数
        return cparams;
    }
};

static std::vector<cmd_params_instance> get_cmd_params_instances_int(const cmd_params & params, int n_gen, int n_prompt) {
    std::vector<cmd_params_instance> instances;

    // 遍历参数中的模型、GPU层数、主GPU、张量分割、批处理数、f32_kv、mul_mat_q、线程数
    for (const auto & m : params.model)
    for (const auto & nl : params.n_gpu_layers)
    for (const auto & mg : params.main_gpu)
    for (const auto & ts : params.tensor_split)
    for (const auto & nb : params.n_batch)
    for (const auto & fk : params.f32_kv)
    for (const auto & mmq : params.mul_mat_q)
    for (const auto & nt : params.n_threads) {
        // 创建命令参数实例
        cmd_params_instance instance = {
            /* .model        = */ m,
            /* .n_prompt     = */ n_prompt,
            /* .n_gen        = */ n_gen,
            /* .n_batch      = */ nb,
            /* .f32_kv       = */ fk,
            /* .n_threads    = */ nt,
            /* .n_gpu_layers = */ nl,
            /* .main_gpu     = */ mg,
            /* .mul_mat_q    = */ mmq,
            /* .tensor_split = */ ts,
        };
        // 将实例添加到实例列表中
        instances.push_back(instance);
    }
    return instances;
}

static std::vector<cmd_params_instance> get_cmd_params_instances(const cmd_params & params) {
    std::vector<cmd_params_instance> instances;

#if 1
    // this ordering minimizes the number of times that each model needs to be reloaded
    // 为了最小化每个模型需要重新加载的次数，按照特定顺序遍历参数
    for (const auto & m : params.model)
    for (const auto & nl : params.n_gpu_layers)
    for (const auto & mg : params.main_gpu)
    for (const auto & ts : params.tensor_split)
    for (const auto & nb : params.n_batch)
    for (const auto & fk : params.f32_kv)
    for (const auto & mmq : params.mul_mat_q)
    // 遍历线程数参数列表
    for (const auto & nt : params.n_threads) {
        // 遍历提示数参数列表
        for (const auto & n_prompt : params.n_prompt) {
            // 如果提示数为0，则跳过当前循环
            if (n_prompt == 0) {
                continue;
            }
            // 创建命令参数实例，初始化各个字段的数值
            cmd_params_instance instance = {
                /* .model        = */ m,
                /* .n_prompt     = */ n_prompt,
                /* .n_gen        = */ 0,
                /* .n_batch      = */ nb,
                /* .f32_kv       = */ fk,
                /* .n_threads    = */ nt,
                /* .n_gpu_layers = */ nl,
                /* .main_gpu     = */ mg,
                /* .mul_mat_q    = */ mmq,
                /* .tensor_split = */ ts,
            };
            // 将实例添加到实例列表中
            instances.push_back(instance);
        }

        // 遍历生成数参数列表
        for (const auto & n_gen : params.n_gen) {
            // 如果生成数为0，则跳过当前循环
            if (n_gen == 0) {
                continue;
            }
            // 创建命令参数实例，初始化各个字段的数值
            cmd_params_instance instance = {
                /* .model        = */ m,
                /* .n_prompt     = */ 0,
                /* .n_gen        = */ n_gen,
                /* .n_batch      = */ nb,
                /* .f32_kv       = */ fk,
                /* .n_threads    = */ nt,
                /* .n_gpu_layers = */ nl,
                /* .main_gpu     = */ mg,
                /* .mul_mat_q    = */ mmq,
                /* .tensor_split = */ ts,
            };
            // 将实例添加到实例列表中
            instances.push_back(instance);
        }
    }
#else
    // 如果不是 prompt 测试，则将 prompt 数量的实例添加到 instances 中
    for (const auto & n_prompt : params.n_prompt) {
        // 如果 prompt 数量为 0，则跳过
        if (n_prompt == 0) {
            continue;
        }
        // 获取 prompt 实例参数并添加到 instances 中
        auto instances_prompt = get_cmd_params_instances_int(params, 0, n_prompt);
        instances.insert(instances.end(), instances_prompt.begin(), instances_prompt.end());
    }

    // 将 generation 测试的实例添加到 instances 中
    for (const auto & n_gen : params.n_gen) {
        // 如果 generation 数量为 0，则跳过
        if (n_gen == 0) {
            continue;
        }
        // 获取 generation 实例参数并添加到 instances 中
        auto instances_gen = get_cmd_params_instances_int(params, n_gen, 0);
        instances.insert(instances.end(), instances_gen.begin(), instances_gen.end());
    }
#endif

    // 返回 instances
    return instances;
}

// 定义 test 结构体
struct test {
    // 定义静态成员变量
    static const std::string build_commit;
    static const int build_number;
    static const bool cuda;
    static const bool opencl;
    static const bool metal;
    static const bool gpu_blas;
    static const bool blas;
    static const std::string cpu_info;
    static const std::string gpu_info;
    // 定义成员变量
    std::string model_filename;
    std::string model_type;
    uint64_t model_size;
    uint64_t model_n_params;
    int n_batch;
    int n_threads;
    bool f32_kv;
    int n_gpu_layers;
    int main_gpu;
    bool mul_mat_q;
    std::array<float, LLAMA_MAX_DEVICES> tensor_split;
    int n_prompt;
    int n_gen;
    std::string test_time;
    std::vector<uint64_t> samples_ns;
}
    // 用于测试的函数，接受命令参数实例、llama_model指针和llama_context指针作为参数
    test(const cmd_params_instance & inst, const llama_model * lmodel, const llama_context * ctx) {
        // 设置模型文件名
        model_filename = inst.model;
        // 创建一个缓冲区，用于存储模型描述
        char buf[128];
        // 获取模型描述信息
        llama_model_desc(lmodel, buf, sizeof(buf));
        // 将模型描述信息转换为字符串
        model_type = buf;
        // 获取模型大小
        model_size = llama_model_size(lmodel);
        // 获取模型参数数量
        model_n_params = llama_model_n_params(lmodel);
        // 设置批处理大小
        n_batch = inst.n_batch;
        // 设置线程数量
        n_threads = inst.n_threads;
        // 设置f32_kv
        f32_kv = inst.f32_kv;
        // 设置GPU层数
        n_gpu_layers = inst.n_gpu_layers;
        // 设置主GPU
        main_gpu = inst.main_gpu;
        // 设置mul_mat_q
        mul_mat_q = inst.mul_mat_q;
        // 设置tensor_split
        tensor_split = inst.tensor_split;
        // 设置提示数量
        n_prompt = inst.n_prompt;
        // 设置生成数量
        n_gen = inst.n_gen;
        // 获取当前时间并按RFC 3339日期时间格式存储在buf中
        time_t t = time(NULL);
        std::strftime(buf, sizeof(buf), "%FT%TZ", gmtime(&t));
        test_time = buf;

        // 忽略ctx参数
        (void) ctx;
    }

    // 返回样本时间的平均值
    uint64_t avg_ns() const {
        return ::avg(samples_ns);
    }

    // 返回样本时间的标准差
    uint64_t stdev_ns() const {
        return ::stdev(samples_ns);
    }

    // 获取时间序列
    std::vector<double> get_ts() const {
        // 计算提示数量和生成数量
        int n_tokens = n_prompt + n_gen;
        // 创建一个空的时间序列
        std::vector<double> ts;
        // 将样本时间转换为时间序列
        std::transform(samples_ns.begin(), samples_ns.end(), std::back_inserter(ts), [n_tokens](uint64_t t) { return 1e9 * n_tokens / t; });
        return ts;
    }

    // 返回时间序列的平均值
    double avg_ts() const {
        return ::avg(get_ts());
    }

    // 返回时间序列的标准差
    double stdev_ts() const {
        return ::stdev(get_ts());
    }

    // 获取后端类型
    static std::string get_backend() {
        if (cuda) {
            return GGML_CUDA_NAME;
        }
        if (opencl) {
            return "OpenCL";
        }
        if (metal) {
            return "Metal";
        }
        if (gpu_blas) {
            return "GPU BLAS";
        }
        if (blas) {
            return "BLAS";
        }
        return "CPU";
    }
    // 返回字段列表的引用
    static const std::vector<std::string> & get_fields() {
        // 定义并初始化字段列表
        static const std::vector<std::string> fields = {
            "build_commit", "build_number",
            "cuda", "opencl", "metal", "gpu_blas", "blas",
            "cpu_info", "gpu_info",
            "model_filename", "model_type", "model_size", "model_n_params",
            "n_batch", "n_threads", "f16_kv",
            "n_gpu_layers", "main_gpu", "mul_mat_q", "tensor_split",
            "n_prompt", "n_gen", "test_time",
            "avg_ns", "stddev_ns",
            "avg_ts", "stddev_ts"
        };
        // 返回字段列表
        return fields;
    }

    // 定义字段类型枚举
    enum field_type {STRING, BOOL, INT, FLOAT};

    // 获取字段类型
    static field_type get_field_type(const std::string & field) {
        // 检查字段类型并返回相应的枚举值
        if (field == "build_number" || field == "n_batch" || field == "n_threads" ||
            field == "model_size" || field == "model_n_params" ||
            field == "n_gpu_layers" || field == "main_gpu" ||
            field == "n_prompt" || field == "n_gen" ||
            field == "avg_ns" || field == "stddev_ns") {
            return INT;
        }
        if (field == "cuda" || field == "opencl" || field == "metal" || field == "gpu_blas" || field == "blas" ||
            field == "f16_kv" || field == "mul_mat_q") {
            return BOOL;
        }
        if (field == "avg_ts" || field == "stddev_ts") {
            return FLOAT;
        }
        // 默认返回字符串类型
        return STRING;
    }
    // 返回一个包含各种数值的字符串向量
    std::vector<std::string> get_values() const {
        // 初始化一个空字符串
        std::string tensor_split_str;
        // 初始化一个最大非零值为0
        int max_nonzero = 0;
        // 遍历tensor_split数组
        for (int i = 0; i < LLAMA_MAX_DEVICES; i++) {
            // 如果tensor_split[i]大于0
            if (tensor_split[i] > 0) {
                // 更新最大非零值
                max_nonzero = i;
            }
        }
        // 遍历tensor_split数组，将其转换为字符串
        for (int i = 0; i <= max_nonzero; i++) {
            // 初始化一个字符数组
            char buf[32];
            // 将tensor_split[i]格式化为字符串并存储到buf中
            snprintf(buf, sizeof(buf), "%.2f", tensor_split[i]);
            // 将buf中的内容添加到tensor_split_str中
            tensor_split_str += buf;
            // 如果不是最后一个元素，添加"/"分隔符
            if (i < max_nonzero) {
                tensor_split_str += "/";
            }
        }
        // 初始化一个包含各种数值的字符串向量
        std::vector<std::string> values = {
            build_commit, std::to_string(build_number),
            std::to_string(cuda), std::to_string(opencl), std::to_string(metal), std::to_string(gpu_blas), std::to_string(blas),
            cpu_info, gpu_info,
            model_filename, model_type, std::to_string(model_size), std::to_string(model_n_params),
            std::to_string(n_batch), std::to_string(n_threads), std::to_string(!f32_kv),
            std::to_string(n_gpu_layers), std::to_string(main_gpu), std::to_string(mul_mat_q), tensor_split_str,
            std::to_string(n_prompt), std::to_string(n_gen), test_time,
            std::to_string(avg_ns()), std::to_string(stdev_ns()),
            std::to_string(avg_ts()), std::to_string(stdev_ts())
        };
        // 返回包含各种数值的字符串向量
        return values;
    }

    // 返回一个包含字段和值的映射
    std::map<std::string, std::string> get_map() const {
        // 初始化一个空映射
        std::map<std::string, std::string> map;
        // 获取字段列表
        auto fields = get_fields();
        // 获取值列表
        auto values = get_values();
        // 将字段和值转换为映射并插入到map中
        std::transform(fields.begin(), fields.end(), values.begin(),
                std::inserter(map, map.end()), std::make_pair<const std::string &, const std::string &>);
        // 返回包含字段和值的映射
        return map;
    }
};

// 定义 test 结构体的静态成员变量，表示构建的提交和构建号
const std::string test::build_commit = LLAMA_COMMIT;
const int         test::build_number = LLAMA_BUILD_NUMBER;
// 根据硬件是否支持 CUDA、OpenCL、Metal、GPU BLAS、BLAS 分别赋值
const bool        test::cuda         = !!ggml_cpu_has_cublas();
const bool        test::opencl       = !!ggml_cpu_has_clblast();
const bool        test::metal        = !!ggml_cpu_has_metal();
const bool        test::gpu_blas     = !!ggml_cpu_has_gpublas();
const bool        test::blas         = !!ggml_cpu_has_blas();
// 获取 CPU 和 GPU 的信息
const std::string test::cpu_info     = get_cpu_info();
const std::string test::gpu_info     = get_gpu_info();

// 定义 printer 结构体
struct printer {
    virtual ~printer() {}

    FILE * fout;
    // 打印测试结果的头部信息
    virtual void print_header(const cmd_params & params) { (void) params; }
    // 打印单个测试的结果
    virtual void print_test(const test & t) = 0;
    // 打印测试结果的尾部信息
    virtual void print_footer() { }
};

// 定义 csv_printer 结构体，继承自 printer
struct csv_printer : public printer {
    // 转义 CSV 字段中的特殊字符
    static std::string escape_csv(const std::string & field) {
        std::string escaped = "\"";
        for (auto c : field) {
            if (c == '"') {
                escaped += "\"";
            }
            escaped += c;
        }
        escaped += "\"";
        return escaped;
    }

    // 打印测试结果的头部信息
    void print_header(const cmd_params & params) override  {
        std::vector<std::string> fields = test::get_fields();
        fprintf(fout, "%s\n", join(fields, ",").c_str());
        (void) params;
    }

    // 打印单个测试的结果
    void print_test(const test & t) override {
        std::vector<std::string> values = t.get_values();
        std::transform(values.begin(), values.end(), values.begin(), escape_csv);
        fprintf(fout, "%s\n", join(values, ",").c_str());
    }
};

// 定义 json_printer 结构体，继承自 printer
struct json_printer : public printer {
    bool first = true;
    // 将输入的字符串进行 JSON 转义处理
    static std::string escape_json(const std::string & value) {
        std::string escaped;
        // 遍历输入字符串的每个字符
        for (auto c : value) {
            // 如果字符是双引号，则转义为 \"
            if (c == '"') {
                escaped += "\\\"";
            } 
            // 如果字符是反斜杠，则转义为 \\
            else if (c == '\\') {
                escaped += "\\\\";
            } 
            // 如果字符是控制字符，则转义为 Unicode 编码
            else  if (c <= 0x1f) {
                char buf[8];
                snprintf(buf, sizeof(buf), "\\u%04x", c);
                escaped += buf;
            } 
            // 其他情况直接添加字符
            else {
                escaped += c;
            }
        }
        return escaped;
    }

    // 根据字段类型格式化字段值
    static std::string format_value(const std::string & field, const std::string & value) {
        switch (test::get_field_type(field)) {
            // 如果字段类型是字符串，则在值两侧加上双引号，并进行 JSON 转义处理
            case test::STRING:
                return "\"" + escape_json(value) + "\"";
            // 如果字段类型是布尔值，则返回 true 或 false
            case test::BOOL:
                return value == "0" ? "false" : "true";
            // 其他情况直接返回值
            default:
                return value;
        }
    }

    // 打印输出的头部信息
    void print_header(const cmd_params & params) override {
        // 输出左方括号
        fprintf(fout, "[\n");
        // 忽略参数
        (void) params;
    }

    // 打印字段和对应的值
    void print_fields(const std::vector<std::string> & fields, const std::vector<std::string> & values) {
        // 断言字段和值的数量相同
        assert(fields.size() == values.size());
        // 遍历字段和值，输出格式化后的字段和值
        for (size_t i = 0; i < fields.size(); i++) {
            fprintf(fout, "    \"%s\": %s,\n", fields.at(i).c_str(), format_value(fields.at(i), values.at(i)).c_str());
        }
    }

    // 打印测试信息
    void print_test(const test & t) override {
        // 如果是第一个测试，则不输出逗号
        if (first) {
            first = false;
        } else {
            fprintf(fout, ",\n");
        }
        // 输出测试信息的开始部分
        fprintf(fout, "  {\n");
        // 输出字段和值
        print_fields(test::get_fields(), t.get_values());
        // 输出样本的纳秒时间戳
        fprintf(fout, "    \"samples_ns\": [ %s ],\n", join(t.samples_ns, ", ").c_str());
        // 输出样本的时间戳
        fprintf(fout, "    \"samples_ts\": [ %s ]\n", join(t.get_ts(), ", ").c_str());
        // 输出测试信息的结束部分
        fprintf(fout, "  }");
        // 刷新输出缓冲区
        fflush(fout);
    }

    // 打印输出的尾部信息
    void print_footer() override {
        // 输出右方括号
        fprintf(fout, "\n]\n");
    }
// 结构体 markdown_printer 继承自 printer 类
struct markdown_printer : public printer {
    // 字段 fields 用于存储字符串向量
    std::vector<std::string> fields;

    // 静态方法，根据字段名返回字段宽度
    static int get_field_width(const std::string & field) {
        // 如果字段是 "model"，返回 -30
        if (field == "model") {
            return -30;
        }
        // 如果字段是 "t/s"，返回 16
        if (field == "t/s") {
            return 16;
        }
        // 如果字段是 "size" 或 "params"，返回 10
        if (field == "size" || field == "params") {
            return 10;
        }
        // 如果字段是 "n_gpu_layers"，返回 3
        if (field == "n_gpu_layers") {
            return 3;
        }

        // 计算字段名的长度和 10 的最大值
        int width = std::max((int)field.length(), 10);

        // 如果字段类型是字符串，返回负值的宽度
        if (test::get_field_type(field) == test::STRING) {
            return -width;
        }
        // 否则返回字段名的宽度
        return width;
    }

    // 静态方法，根据字段名返回字段显示名称
    static std::string get_field_display_name(const std::string & field) {
        // 如果字段是 "n_gpu_layers"，返回 "ngl"
        if (field == "n_gpu_layers") {
            return "ngl";
        }
        // 如果字段是 "n_threads"，返回 "threads"
        if (field == "n_threads") {
            return "threads";
        }
        // 如果字段是 "mul_mat_q"，返回 "mmq"
        if (field == "mul_mat_q") {
            return "mmq";
        }
        // 如果字段是 "tensor_split"，返回 "ts"
        if (field == "tensor_split") {
            return "ts";
        }
        // 否则返回字段名
        return field;
    }
    // 重写父类的打印头部信息的方法
    void print_header(const cmd_params & params) override {
        // 选择要打印的字段
        fields.push_back("model");
        fields.push_back("size");
        fields.push_back("params");
        fields.push_back("backend");
        // 检查当前是否是 CPU 后端，如果不是则添加 n_gpu_layers 字段
        bool is_cpu_backend = test::get_backend() == "CPU" || test::get_backend() == "BLAS";
        if (!is_cpu_backend) {
            fields.push_back("n_gpu_layers");
        }
        // 如果线程数大于 1，或者不等于默认值，或者是 CPU 后端，则添加 n_threads 字段
        if (params.n_threads.size() > 1 || params.n_threads != cmd_params_defaults.n_threads || is_cpu_backend) {
            fields.push_back("n_threads");
        }
        // 如果批次数大于 1，或者不等于默认值，则添加 n_batch 字段
        if (params.n_batch.size() > 1 || params.n_batch != cmd_params_defaults.n_batch) {
            fields.push_back("n_batch");
        }
        // 如果 f32_kv 大于 1，或者不等于默认值，则添加 f16_kv 字段
        if (params.f32_kv.size() > 1 || params.f32_kv != cmd_params_defaults.f32_kv) {
            fields.push_back("f16_kv");
        }
        // 如果 main_gpu 大于 1，或者不等于默认值，则添加 main_gpu 字段
        if (params.main_gpu.size() > 1 || params.main_gpu != cmd_params_defaults.main_gpu) {
            fields.push_back("main_gpu");
        }
        // 如果 mul_mat_q 大于 1，或者不等于默认值，则添加 mul_mat_q 字段
        if (params.mul_mat_q.size() > 1 || params.mul_mat_q != cmd_params_defaults.mul_mat_q) {
            fields.push_back("mul_mat_q");
        }
        // 如果 tensor_split 大于 1，或者不等于默认值，则添加 tensor_split 字段
        if (params.tensor_split.size() > 1 || params.tensor_split != cmd_params_defaults.tensor_split) {
            fields.push_back("tensor_split");
        }
        // 添加 test 字段和 t/s 字段
        fields.push_back("test");
        fields.push_back("t/s");

        // 打印表格头部分隔线
        fprintf(fout, "|");
        for (const auto & field : fields) {
            fprintf(fout, " %*s |", get_field_width(field), get_field_display_name(field).c_str());
        }
        fprintf(fout, "\n");
        // 打印表格头部分隔线下方的线条
        fprintf(fout, "|");
        for (const auto & field : fields) {
            int width = get_field_width(field);
            fprintf(fout, " %s%s |", std::string(std::abs(width) - 1, '-').c_str(), width > 0 ? ":" : "-");
        }
        fprintf(fout, "\n");
    }
    // 重写基类的打印页脚函数
    void print_footer() override {
        // 在输出流中打印构建信息，包括构建提交和构建编号
        fprintf(fout, "\nbuild: %s (%d)\n", test::build_commit.c_str(), test::build_number);
    }
};

// 定义一个结构体 sql_printer，继承自 printer
struct sql_printer : public printer {
    // 静态方法，根据字段名获取 SQL 字段类型
    static std::string get_sql_field_type(const std::string & field) {
        // 根据字段类型返回对应的 SQL 类型
        switch (test::get_field_type(field)) {
            case test::STRING:
                return "TEXT";
            case test::BOOL:
            case test::INT:
                return "INTEGER";
            case test::FLOAT:
                return "REAL";
            default:
                // 断言，如果出现未知字段类型，则终止程序
                assert(false);
                exit(1);
        }
    }

    // 重写父类的方法，打印表头
    void print_header(const cmd_params & params) override {
        // 获取字段列表
        std::vector<std::string> fields = test::get_fields();
        // 打印创建表的 SQL 语句
        fprintf(fout, "CREATE TABLE IF NOT EXISTS test (\n");
        // 遍历字段列表，打印字段名和类型
        for (size_t i = 0; i < fields.size(); i++) {
            fprintf(fout, "  %s %s%s\n", fields.at(i).c_str(), get_sql_field_type(fields.at(i)).c_str(),  i < fields.size() - 1 ? "," : "");
        }
        // 打印结束符
        fprintf(fout, ");\n");
        fprintf(fout, "\n");
        (void) params;
    }

    // 重写父类的方法，打印测试数据
    void print_test(const test & t) override {
        // 打印插入数据的 SQL 语句
        fprintf(fout, "INSERT INTO test (%s) ", join(test::get_fields(), ", ").c_str());
        fprintf(fout, "VALUES (");
        // 获取数值列表，打印每个数值
        std::vector<std::string> values = t.get_values();
        for (size_t i = 0; i < values.size(); i++) {
            fprintf(fout, "'%s'%s", values.at(i).c_str(), i < values.size() - 1 ? ", " : "");
        }
        // 打印结束符
        fprintf(fout, ");\n");
    }
};

// 静态方法，测试提示
static void test_prompt(llama_context * ctx, int n_prompt, int n_past, int n_batch, int n_threads) {
    // 创建令牌列表
    std::vector<llama_token> tokens(n_batch, llama_token_bos(llama_get_model(ctx)));
    int n_processed = 0;

    // 设置线程数
    llama_set_n_threads(ctx, n_threads, n_threads);

    // 循环处理提示
    while (n_processed < n_prompt) {
        int n_tokens = std::min(n_prompt - n_processed, n_batch);
        // 解码
        llama_decode(ctx, llama_batch_get_one(tokens.data(), n_tokens, n_past + n_processed, 0));
        n_processed += n_tokens;
    }
}

// 静态方法，生成测试数据
static void test_gen(llama_context * ctx, int n_gen, int n_past, int n_threads) {
    # 从上下文中获取模型，并生成一个令牌
    llama_token token = llama_token_bos(llama_get_model(ctx));

    # 设置LLAMA上下文中的线程数
    llama_set_n_threads(ctx, n_threads, n_threads);

    # 循环执行生成操作
    for (int i = 0; i < n_gen; i++) {
        # 从令牌中获取一个批次的数据，并解码
        llama_decode(ctx, llama_batch_get_one(&token, 1, n_past + i, 0));
    }
}

// 定义一个静态函数，用于处理空日志回调
static void llama_null_log_callback(enum ggml_log_level level, const char * text, void * user_data) {
    // 忽略日志级别、文本和用户数据，不做任何处理
    (void) level;
    (void) text;
    (void) user_data;
}

// 主函数
int main(int argc, char ** argv) {
    // 尝试设置区域设置以支持 markdown 中的 Unicode 字符
    setlocale(LC_CTYPE, ".UTF-8");

    // 如果未定义 NDEBUG，则输出警告信息
#if !defined(NDEBUG)
    fprintf(stderr, "warning: asserts enabled, performance may be affected\n");
#endif

    // 如果是 debug build，则输出警告信息
#if (defined(_MSC_VER) && defined(_DEBUG)) || (!defined(_MSC_VER) && !defined(__OPTIMIZE__))
    fprintf(stderr, "warning: debug build, performance may be affected\n");
#endif

    // 如果启用了 sanitizer，则输出警告信息
#if defined(__SANITIZE_ADDRESS__) || defined(__SANITIZE_THREAD__)
    fprintf(stderr, "warning: sanitizer enabled, performance may be affected\n");
#endif

    // 解析命令行参数
    cmd_params params = parse_cmd_params(argc, argv);

    // 如果不需要详细输出，则设置空日志回调
    if (!params.verbose) {
        llama_log_set(llama_null_log_callback, NULL);
    }

    // 初始化 llama.cpp
    bool numa = false;
    llama_backend_init(numa);

    // 初始化打印机
    std::unique_ptr<printer> p;
    switch (params.output_format) {
        case CSV:
            p.reset(new csv_printer());
            break;
        case JSON:
            p.reset(new json_printer());
            break;
        case MARKDOWN:
            p.reset(new markdown_printer());
            break;
        case SQL:
            p.reset(new sql_printer());
            break;
        default:
            assert(false);  // 断言，如果执行到这里说明出现了意外情况
            exit(1);  // 退出程序
    }
    p->fout = stdout;  // 设置打印机的输出流为标准输出
    p->print_header(params);  // 打印头部信息

    // 获取命令行参数的实例
    std::vector<cmd_params_instance> params_instances = get_cmd_params_instances(params);

    // 初始化 llama_model 和 prev_inst
    llama_model * lmodel = nullptr;
    const cmd_params_instance * prev_inst = nullptr;
    for (const auto & inst : params_instances) {
        // 遍历参数实例列表
        // 如果模型为空，或者前一个实例为空，或者当前实例的模型参数与前一个实例不相等
        if (!lmodel || !prev_inst || !inst.equal_mparams(*prev_inst)) {
            // 如果模型不为空，释放模型
            if (lmodel) {
                llama_free_model(lmodel);
            }
            // 从文件加载模型，并根据实例参数创建LLAMA模型
            lmodel = llama_load_model_from_file(inst.model.c_str(), inst.to_llama_mparams());
            // 如果加载模型失败，输出错误信息并返回1
            if (lmodel == NULL) {
                fprintf(stderr, "%s: error: failed to load model '%s'\n", __func__, inst.model.c_str());
                return 1;
            }
            // 更新前一个实例为当前实例
            prev_inst = &inst;
        }

        // 根据模型和实例参数创建LLAMA上下文
        llama_context * ctx = llama_new_context_with_model(lmodel, inst.to_llama_cparams());
        // 如果创建上下文失败，输出错误信息，释放模型，并返回1
        if (ctx == NULL) {
            fprintf(stderr, "%s: error: failed to create context with model '%s'\n", __func__, inst.model.c_str());
            llama_free_model(lmodel);
            return 1;
        }

        // 创建测试对象
        test t(inst, lmodel, ctx);

        // 清空LLAMA键值缓存
        llama_kv_cache_clear(ctx);

        // 预热运行
        if (t.n_prompt > 0) {
            test_prompt(ctx, std::min(2, t.n_batch), 0, t.n_batch, t.n_threads);
        }
        if (t.n_gen > 0) {
            test_gen(ctx, 1, 0, t.n_threads);
        }

        // 运行多次测试
        for (int i = 0; i < params.reps; i++) {
            // 清空LLAMA键值缓存
            llama_kv_cache_clear(ctx);

            // 记录测试开始时间
            uint64_t t_start = get_time_ns();
            // 运行测试
            if (t.n_prompt > 0) {
                test_prompt(ctx, t.n_prompt, 0, t.n_batch, t.n_threads);
            }
            if (t.n_gen > 0) {
                test_gen(ctx, t.n_gen, t.n_prompt, t.n_threads);
            }
            // 计算测试时间并添加到样本时间列表
            uint64_t t_ns = get_time_ns() - t_start;
            t.samples_ns.push_back(t_ns);
        }

        // 打印测试结果
        p->print_test(t);

        // 打印LLAMA计时信息
        llama_print_timings(ctx);

        // 释放LLAMA上下文
        llama_free(ctx);
    }

    // 释放LLAMA模型
    llama_free_model(lmodel);

    // 打印页脚信息
    p->print_footer();

    // 释放LLAMA后端资源
    llama_backend_free();

    // 返回0
    return 0;
# 闭合前面的函数定义
```