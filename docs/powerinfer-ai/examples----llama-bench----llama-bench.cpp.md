# `PowerInfer\examples\llama-bench\llama-bench.cpp`

```
// 包含标准库头文件
#include <algorithm>       // 提供各种算法
#include <array>           // 提供固定大小数组
#include <cassert>         // 提供断言
#include <chrono>          // 提供时间库
#include <cinttypes>       // 提供整数类型格式化
#include <clocale>         // 提供本地化库
#include <cmath>           // 提供数学函数
#include <cstdio>          // 提供 C 样式输入输出
#include <cstring>         // 提供 C 字符串操作
#include <ctime>           // 提供 C 时间库
#include <iterator>        // 提供迭代器
#include <map>             // 提供映射容器
#include <numeric>         // 提供数值算法
#include <regex>           // 提供正则表达式
#include <sstream>         // 提供字符串流
#include <string>          // 提供字符串操作
#include <vector>          // 提供动态数组

// 包含自定义头文件
#include "ggml.h"          // 自定义头文件 ggml
#include "llama.h"         // 自定义头文件 llama
// 包含自定义的头文件 common.h 和 ggml-cuda.h
#include "common.h"
#include "ggml-cuda.h"

// 获取当前时间的纳秒级别时间戳
static uint64_t get_time_ns() {
    // 使用高精度时钟获取当前时间
    using clock = std::chrono::high_resolution_clock;
    // 将时间转换为纳秒并返回
    return std::chrono::nanoseconds(clock::now().time_since_epoch()).count();
}

// 将 vector 中的值用指定的分隔符连接成字符串
template<class T>
static std::string join(const std::vector<T> & values, const std::string & delim) {
    // 创建一个字符串流
    std::ostringstream str;
    // 遍历 vector 中的值
    for (size_t i = 0; i < values.size(); i++) {
        // 将值添加到字符串流中
        str << values[i];
        // 如果不是最后一个值，添加分隔符
        if (i < values.size() - 1) {
            str << delim;
        }
    }
    // 返回连接后的字符串
    return str.str();
}
// 定义一个模板函数，用于将字符串按照指定分隔符分割成多个值，并存储在vector中
template<class T>
static std::vector<T> split(const std::string & str, char delim) {
    // 创建一个vector用于存储分割后的值
    std::vector<T> values;
    // 创建一个字符串流，用于读取输入的字符串
    std::istringstream str_stream(str);
    // 定义一个字符串变量，用于存储每次分割后的值
    std::string token;
    // 循环读取字符串流中的值，并按照指定分隔符进行分割
    while (std::getline(str_stream, token, delim)) {
        // 定义一个变量用于存储转换后的值
        T value;
        // 创建一个字符串流，用于将分割后的字符串转换成指定类型的值
        std::istringstream token_stream(token);
        // 将转换后的值存储到value中
        token_stream >> value;
        // 将转换后的值存储到vector中
        values.push_back(value);
    }
    // 返回存储分割后值的vector
    return values;
}

// 定义一个模板函数，用于计算vector中元素的平均值
template<typename T>
static T avg(const std::vector<T> & v) {
    // 如果vector为空，则返回0
    if (v.empty()) {
        return 0;
    }
    // 使用标准库函数std::accumulate对vector v中的元素进行累加求和
    T sum = std::accumulate(v.begin(), v.end(), T(0));
    // 返回vector v的平均值
    return sum / (T)v.size();
}

// 计算vector v的标准差
template<typename T>
static T stdev(const std::vector<T> & v) {
    // 如果vector v的大小小于等于1，直接返回0
    if (v.size() <= 1) {
        return 0;
    }
    // 计算vector v的平均值
    T mean = avg(v);
    // 计算vector v的平方和
    T sq_sum = std::inner_product(v.begin(), v.end(), v.begin(), T(0));
    // 计算vector v的标准差
    T stdev = std::sqrt(sq_sum / (T)(v.size() - 1) - mean * mean * (T)v.size() / (T)(v.size() - 1));
    return stdev;
}

// 获取CPU信息
static std::string get_cpu_info() {
    std::string id;
    // 如果是Linux系统
#ifdef __linux__
    // 打开/proc/cpuinfo文件
    FILE * f = fopen("/proc/cpuinfo", "r");
    // 如果文件打开成功
    if (f) {
// 定义一个字符数组，用于存储读取的每行内容
char buf[1024];
// 逐行读取文件内容，直到文件末尾
while (fgets(buf, sizeof(buf), f)) {
    // 判断当前行是否以 "model name" 开头
    if (strncmp(buf, "model name", 10) == 0) {
        // 查找当前行中的冒号位置
        char * p = strchr(buf, ':');
        // 如果找到冒号
        if (p) {
            // 指针移动到冒号后面的第一个非空白字符
            p++;
            while (std::isspace(*p)) {
                p++;
            }
            // 去除字符串末尾的空白字符
            while (std::isspace(p[strlen(p) - 1])) {
                p[strlen(p) - 1] = '\0';
            }
            // 将处理后的字符串赋值给 id，并结束循环
            id = p;
            break;
        }
    }
}
// 结束条件编译指令
#endif
// TODO: 其他平台的处理
// 返回 GPU 信息的字符串
static std::string get_gpu_info() {
    std::string id; // 创建一个空字符串用于存储 GPU 信息
#ifdef GGML_USE_CUBLAS
    int count = ggml_cuda_get_device_count(); // 获取 CUDA 设备数量
    for (int i = 0; i < count; i++) { // 遍历每个 CUDA 设备
        char buf[128]; // 创建一个缓冲区用于存储设备描述
        ggml_cuda_get_device_description(i, buf, sizeof(buf)); // 获取设备描述
        id += buf; // 将设备描述添加到 GPU 信息字符串中
        if (i < count - 1) { // 如果不是最后一个设备
            id += "/"; // 在设备描述之间添加斜杠
        }
    }
#endif
    // TODO: other backends // 其他后端的处理，暂时未实现
    return id; // 返回 GPU 信息字符串
}
// 定义枚举类型，表示输出格式
enum output_formats {CSV, JSON, MARKDOWN, SQL};

// 定义命令行参数结构体
struct cmd_params {
    std::vector<std::string> model; // 模型名称列表
    std::vector<int> n_prompt; // 提示数量列表
    std::vector<int> n_gen; // 生成数量列表
    std::vector<int> n_batch; // 批处理数量列表
    std::vector<bool> f32_kv; // 是否使用32位键值对列表
    std::vector<int> n_threads; // 线程数量列表
    std::vector<int> n_gpu_layers; // GPU层数列表
    std::vector<int> main_gpu; // 主GPU列表
    std::vector<bool> mul_mat_q; // 是否使用多个矩阵列表
    std::vector<std::array<float, LLAMA_MAX_DEVICES>> tensor_split; // 张量分割列表
    int reps; // 重复次数
    bool verbose; // 是否显示详细信息
    output_formats output_format; // 输出格式
};

// 定义命令行参数的默认值
static const cmd_params cmd_params_defaults = {
    /* model         */ {"models/7B/ggml-model-q4_0.gguf"}, // 定义模型路径
    /* n_prompt      */ {512}, // 定义提示数量
    /* n_gen         */ {128}, // 定义生成数量
    /* n_batch       */ {512}, // 定义批处理大小
    /* f32_kv        */ {false}, // 定义是否使用32位键值对
    /* n_threads     */ {get_num_physical_cores()}, // 定义线程数量
    /* n_gpu_layers  */ {99}, // 定义GPU层数
    /* main_gpu      */ {0}, // 定义主GPU
    /* mul_mat_q     */ {true}, // 定义是否使用矩阵乘法
    /* tensor_split  */ {{}}, // 定义张量分割
    /* reps          */ 5, // 定义重复次数
    /* verbose       */ false, // 定义是否冗长输出
    /* output_format */ MARKDOWN // 定义输出格式
};

static void print_usage(int /* argc */, char ** argv) {
    printf("usage: %s [options]\n", argv[0]); // 打印使用方法
    printf("\n");
    printf("options:\n"); // 打印选项
    printf("  -h, --help\n"); // 打印帮助选项
    # 打印模型文件名，默认值为 cmd_params_defaults.model，并转换为 C 字符串
    printf("  -m, --model <filename>            (default: %s)\n", join(cmd_params_defaults.model, ",").c_str());
    # 打印提示数量，默认值为 cmd_params_defaults.n_prompt，并转换为 C 字符串
    printf("  -p, --n-prompt <n>                (default: %s)\n", join(cmd_params_defaults.n_prompt, ",").c_str());
    # 打印生成数量，默认值为 cmd_params_defaults.n_gen，并转换为 C 字符串
    printf("  -n, --n-gen <n>                   (default: %s)\n", join(cmd_params_defaults.n_gen, ",").c_str());
    # 打印批处理大小，默认值为 cmd_params_defaults.n_batch，并转换为 C 字符串
    printf("  -b, --batch-size <n>              (default: %s)\n", join(cmd_params_defaults.n_batch, ",").c_str());
    # 打印内存类型，默认值为 cmd_params_defaults.f32_kv，并转换为 C 字符串
    printf("  --memory-f32 <0|1>                (default: %s)\n", join(cmd_params_defaults.f32_kv, ",").c_str());
    # 打印线程数量，默认值为 cmd_params_defaults.n_threads，并转换为 C 字符串
    printf("  -t, --threads <n>                 (default: %s)\n", join(cmd_params_defaults.n_threads, ",").c_str());
    # 打印 GPU 层数量，默认值为 cmd_params_defaults.n_gpu_layers，并转换为 C 字符串
    printf("  -ngl, --n-gpu-layers <n>          (default: %s)\n", join(cmd_params_defaults.n_gpu_layers, ",").c_str());
    # 打印主 GPU 编号，默认值为 cmd_params_defaults.main_gpu，并转换为 C 字符串
    printf("  -mg, --main-gpu <i>               (default: %s)\n", join(cmd_params_defaults.main_gpu, ",").c_str());
    # 打印是否使用矩阵乘法，默认值为 cmd_params_defaults.mul_mat_q，并转换为 C 字符串
    printf("  -mmq, --mul-mat-q <0|1>           (default: %s)\n", join(cmd_params_defaults.mul_mat_q, ",").c_str());
    # 打印张量分割方式
    printf("  -ts, --tensor_split <ts0/ts1/..>               \n");
    # 打印重复次数，默认值为 cmd_params_defaults.reps
    printf("  -r, --repetitions <n>             (default: %d)\n", cmd_params_defaults.reps);
    # 打印是否详细输出，默认值为 cmd_params_defaults.verbose
    printf("  -v, --verbose                     (default: %s)\n", cmd_params_defaults.verbose ? "1" : "0");
    # 打印空行
    printf("\n");
    # 提示多个值可以通过逗号分隔或多次指定参数来传递
    printf("Multiple values can be given for each parameter by separating them with ',' or by specifying the parameter multiple times.\n");

}

# 解析命令行参数
static cmd_params parse_cmd_params(int argc, char ** argv) {
    # 初始化参数对象
    cmd_params params;
    # 初始化字符串参数
    std::string arg;
    // 用于标记参数是否有效
    bool invalid_param = false;
    // 定义参数前缀
    const std::string arg_prefix = "--";
    // 定义参数分隔符
    const char split_delim = ',';

    // 设置参数的默认值
    params.verbose = cmd_params_defaults.verbose;
    params.output_format = cmd_params_defaults.output_format;
    params.reps = cmd_params_defaults.reps;

    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数
        arg = argv[i];
        // 如果参数以指定前缀开头，则将其中的下划线替换为破折号
        if (arg.compare(0, arg_prefix.size(), arg_prefix) == 0) {
            std::replace(arg.begin(), arg.end(), '_', '-');
        }

        // 如果参数为帮助选项，则打印用法并退出程序
        if (arg == "-h" || arg == "--help") {
            print_usage(argc, argv);
            exit(0);
        } 
        // 如果参数为模型选项，则检查下一个参数是否存在
        else if (arg == "-m" || arg == "--model") {
            if (++i >= argc) {
                // 如果下一个参数不存在，则标记参数无效
                invalid_param = true;
            // 如果参数为"-m"或"--model"，则读取下一个参数作为模型文件名，并添加到params.model中
            if (arg == "-m" || arg == "--model") {
                // 检查是否还有下一个参数，如果没有则设置invalid_param为true并跳出循环
                if (++i >= argc) {
                    invalid_param = true;
                    break;
                }
                // 将下一个参数按照分隔符分割成字符串，并添加到params.model中
                auto p = split<std::string>(argv[i], split_delim);
                params.model.insert(params.model.end(), p.begin(), p.end());
            } 
            // 如果参数为"-p"或"--n-prompt"，则读取下一个参数作为提示数量，并添加到params.n_prompt中
            else if (arg == "-p" || arg == "--n-prompt") {
                // 检查是否还有下一个参数，如果没有则设置invalid_param为true并跳出循环
                if (++i >= argc) {
                    invalid_param = true;
                    break;
                }
                // 将下一个参数转换为整数，并添加到params.n_prompt中
                auto p = split<int>(argv[i], split_delim);
                params.n_prompt.insert(params.n_prompt.end(), p.begin(), p.end());
            } 
            // 如果参数为"-n"或"--n-gen"，则读取下一个参数作为生成数量，并添加到params.n_gen中
            else if (arg == "-n" || arg == "--n-gen") {
                // 检查是否还有下一个参数，如果没有则设置invalid_param为true并跳出循环
                if (++i >= argc) {
                    invalid_param = true;
                    break;
                }
                // 将下一个参数转换为整数，并添加到params.n_gen中
                auto p = split<int>(argv[i], split_delim);
                params.n_gen.insert(params.n_gen.end(), p.begin(), p.end());
            } 
            // 如果参数为"-b"或"--batch-size"，则读取下一个参数作为批处理大小
            else if (arg == "-b" || arg == "--batch-size") {
                // 检查是否还有下一个参数，如果没有则设置invalid_param为true并跳出循环
                if (++i >= argc) {
        // 如果参数无效，设置标志并跳出循环
        invalid_param = true;
        break;
    }
    // 使用 split 函数将参数转换为整数并插入到 params.n_batch 中
    auto p = split<int>(argv[i], split_delim);
    params.n_batch.insert(params.n_batch.end(), p.begin(), p.end());
} else if (arg == "--memory-f32") {
    // 如果参数无效，设置标志并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    // 使用 split 函数将参数转换为整数并插入到 params.f32_kv 中
    auto p = split<int>(argv[i], split_delim);
    params.f32_kv.insert(params.f32_kv.end(), p.begin(), p.end());
} else if (arg == "-t" || arg == "--threads") {
    // 如果参数无效，设置标志并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    // 使用 split 函数将参数转换为整数并插入到 params.n_threads 中
    auto p = split<int>(argv[i], split_delim);
    params.n_threads.insert(params.n_threads.end(), p.begin(), p.end());
} else if (arg == "-ngl" || arg == "--n-gpu-layers") {
# 如果参数索引超出范围，设置参数无效并跳出循环
if (++i >= argc) {
    invalid_param = true;
    break;
}
# 使用 split 函数将参数转换为整数，并将结果插入到参数的 n_gpu_layers 中
auto p = split<int>(argv[i], split_delim);
params.n_gpu_layers.insert(params.n_gpu_layers.end(), p.begin(), p.end());
} else if (arg == "-mg" || arg == "--main-gpu") {
    # 如果参数索引超出范围，设置参数无效并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    # 将参数转换为整数，并赋值给 main_gpu
    params.main_gpu = split<int>(argv[i], split_delim);
} else if (arg == "-mmq" || arg == "--mul-mat-q") {
    # 如果参数索引超出范围，设置参数无效并跳出循环
    if (++i >= argc) {
        invalid_param = true;
        break;
    }
    # 使用 split 函数将参数转换为布尔值，并将结果插入到参数的 mul_mat_q 中
    auto p = split<bool>(argv[i], split_delim);
    params.mul_mat_q.insert(params.mul_mat_q.end(), p.begin(), p.end());
} else if (arg == "-ts" || arg == "--tensor-split") {
            // 检查是否超出参数数量，如果是则设置参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 对参数进行分割，按照分号和斜杠进行分割
            for (auto ts : split<std::string>(argv[i], split_delim)) {
                // 使用正则表达式定义分割规则
                const std::regex regex{R"([;/]+)"};
                // 使用正则表达式对字符串进行分割
                std::sregex_token_iterator it{ts.begin(), ts.end(), regex, -1};
                // 将分割后的结果存储到 split_arg 中
                std::vector<std::string> split_arg{it, {}};
                // 断言分割后的结果不超过最大设备数量
                GGML_ASSERT(split_arg.size() <= LLAMA_MAX_DEVICES);

                // 创建存储分割结果的数组
                std::array<float, LLAMA_MAX_DEVICES> tensor_split;
                // 遍历分割结果，将字符串转换为浮点数存储到数组中
                for (size_t i = 0; i < LLAMA_MAX_DEVICES; ++i) {
                    if (i < split_arg.size()) {
                        tensor_split[i] = std::stof(split_arg[i]);
                    } else {
                        tensor_split[i] = 0.0f;
                    }
                }
                // 将数组存储到参数对象的 tensor_split 中
                params.tensor_split.push_back(tensor_split);
        } else if (arg == "-r" || arg == "--repetitions") {
            // 如果参数是"-r"或"--repetitions"，则获取下一个参数作为重复次数
            if (++i >= argc) {
                // 如果没有下一个参数，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数并赋值给params.reps
            params.reps = std::stoi(argv[i]);
        } else if (arg == "-o" || arg == "--output") {
            // 如果参数是"-o"或"--output"，则获取下一个参数作为输出格式
            if (++i >= argc) {
                // 如果没有下一个参数，设置参数无效并跳出循环
                invalid_param = true;
                break;
            }
            // 根据下一个参数的值设置输出格式
            if (argv[i] == std::string("csv")) {
                params.output_format = CSV;
            } else if (argv[i] == std::string("json")) {
                params.output_format = JSON;
            } else if (argv[i] == std::string("md")) {
                params.output_format = MARKDOWN;
            } else if (argv[i] == std::string("sql")) {
                params.output_format = SQL;
    } else {
        // 如果参数不匹配任何已知选项，则标记为无效参数并跳出循环
        invalid_param = true;
        break;
    }
} else if (arg == "-v" || arg == "--verbose") {
    // 如果参数是"-v"或"--verbose"，则设置参数的verbose属性为true
    params.verbose = true;
} else {
    // 如果参数不匹配任何已知选项，则标记为无效参数并跳出循环
    invalid_param = true;
    break;
}
}
if (invalid_param) {
    // 如果存在无效参数，则打印错误信息，打印用法，并退出程序
    fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
    print_usage(argc, argv);
    exit(1);
}

// 设置默认值
if (params.model.empty())        { params.model = cmd_params_defaults.model; }
if (params.n_prompt.empty())     { params.n_prompt = cmd_params_defaults.n_prompt; }
    # 如果参数中的 n_gen 为空，则将其赋值为默认值 cmd_params_defaults.n_gen
    if (params.n_gen.empty())        { params.n_gen = cmd_params_defaults.n_gen; }
    # 如果参数中的 n_batch 为空，则将其赋值为默认值 cmd_params_defaults.n_batch
    if (params.n_batch.empty())      { params.n_batch = cmd_params_defaults.n_batch; }
    # 如果参数中的 f32_kv 为空，则将其赋值为默认值 cmd_params_defaults.f32_kv
    if (params.f32_kv.empty())       { params.f32_kv = cmd_params_defaults.f32_kv; }
    # 如果参数中的 n_gpu_layers 为空，则将其赋值为默认值 cmd_params_defaults.n_gpu_layers
    if (params.n_gpu_layers.empty()) { params.n_gpu_layers = cmd_params_defaults.n_gpu_layers; }
    # 如果参数中的 main_gpu 为空，则将其赋值为默认值 cmd_params_defaults.main_gpu
    if (params.main_gpu.empty())     { params.main_gpu = cmd_params_defaults.main_gpu; }
    # 如果参数中的 mul_mat_q 为空，则将其赋值为默认值 cmd_params_defaults.mul_mat_q
    if (params.mul_mat_q.empty())    { params.mul_mat_q = cmd_params_defaults.mul_mat_q; }
    # 如果参数中的 tensor_split 为空，则将其赋值为默认值 cmd_params_defaults.tensor_split
    if (params.tensor_split.empty()) { params.tensor_split = cmd_params_defaults.tensor_split; }
    # 如果参数中的 n_threads 为空，则将其赋值为默认值 cmd_params_defaults.n_threads
    if (params.n_threads.empty())    { params.n_threads = cmd_params_defaults.n_threads; }

    # 返回更新后的参数
    return params;
}

# 定义结构体 cmd_params_instance，包含模型名称、提示数量、生成数量、批次数量、是否使用 f32_kv、线程数量、GPU 层数量等参数
struct cmd_params_instance {
    std::string model;
    int n_prompt;
    int n_gen;
    int n_batch;
    bool f32_kv;
    int n_threads;
    int n_gpu_layers;
    // 定义整型变量 main_gpu，用于存储主 GPU 的编号
    int main_gpu;
    // 定义布尔型变量 mul_mat_q，用于存储是否进行矩阵乘法的标志
    bool mul_mat_q;
    // 定义长度为 LLAMA_MAX_DEVICES 的浮点数数组 tensor_split，用于存储张量分割信息

    // 将当前对象转换为 llama_model_params 结构体
    llama_model_params to_llama_mparams() const {
        // 创建一个默认的 llama_model_params 结构体
        llama_model_params mparams = llama_model_default_params();

        // 将当前对象的属性赋值给 mparams 结构体
        mparams.n_gpu_layers = n_gpu_layers;
        mparams.main_gpu = main_gpu;
        mparams.tensor_split = tensor_split.data();

        // 返回 mparams 结构体
        return mparams;
    }

    // 比较当前对象和另一个 cmd_params_instance 对象的属性是否相等
    bool equal_mparams(const cmd_params_instance & other) const {
        // 比较 model、n_gpu_layers、main_gpu 和 tensor_split 属性是否相等，返回比较结果
        return model == other.model &&
               n_gpu_layers == other.n_gpu_layers &&
               main_gpu == other.main_gpu &&
               tensor_split == other.tensor_split;
    }
// 将当前对象的参数转换为LLAMA上下文参数
llama_context_params to_llama_cparams() const {
    // 创建LLAMA上下文参数对象，并使用默认参数初始化
    llama_context_params cparams = llama_context_default_params();

    // 设置LLAMA上下文参数的上下文数量为提示数量加生成数量
    cparams.n_ctx = n_prompt + n_gen;
    // 设置LLAMA上下文参数的批处理数量为指定的批处理数量
    cparams.n_batch = n_batch;
    // 根据条件设置LLAMA上下文参数的f16_kv属性
    cparams.f16_kv = !f32_kv;
    // 根据条件设置LLAMA上下文参数的mul_mat_q属性
    cparams.mul_mat_q = mul_mat_q;

    // 返回LLAMA上下文参数对象
    return cparams;
}

// 获取整数类型的命令参数实例的向量
static std::vector<cmd_params_instance> get_cmd_params_instances_int(const cmd_params & params, int n_gen, int n_prompt) {
    // 创建命令参数实例的向量
    std::vector<cmd_params_instance> instances;

    // 遍历模型参数
    for (const auto & m : params.model)
    // 遍历GPU层参数
    for (const auto & nl : params.n_gpu_layers)
    // 遍历主GPU参数
    for (const auto & mg : params.main_gpu)
    // 遍历张量分割参数
    for (const auto & ts : params.tensor_split)
// 遍历参数中的批次数
for (const auto & nb : params.n_batch)
// 遍历参数中的f32_kv
for (const auto & fk : params.f32_kv)
// 遍历参数中的mul_mat_q
for (const auto & mmq : params.mul_mat_q)
// 遍历参数中的线程数
for (const auto & nt : params.n_threads) {
    // 创建命令参数实例
    cmd_params_instance instance = {
        /* .model        = */ m, // 模型
        /* .n_prompt     = */ n_prompt, // 提示数
        /* .n_gen        = */ n_gen, // 生成数
        /* .n_batch      = */ nb, // 批次数
        /* .f32_kv       = */ fk, // f32_kv
        /* .n_threads    = */ nt, // 线程数
        /* .n_gpu_layers = */ nl, // GPU层数
        /* .main_gpu     = */ mg, // 主GPU
        /* .mul_mat_q    = */ mmq, // mul_mat_q
        /* .tensor_split = */ ts, // 张量分割
    };
    // 将实例添加到实例列表中
    instances.push_back(instance);
}
// 返回实例列表
return instances;
// 获取命令参数的实例
static std::vector<cmd_params_instance> get_cmd_params_instances(const cmd_params & params) {
    // 创建实例向量
    std::vector<cmd_params_instance> instances;

    // 为了最小化每个模型需要重新加载的次数，按照特定顺序遍历参数
#if 1
    for (const auto & m : params.model) // 遍历模型参数
    for (const auto & nl : params.n_gpu_layers) // 遍历 GPU 层数参数
    for (const auto & mg : params.main_gpu) // 遍历主 GPU 参数
    for (const auto & ts : params.tensor_split) // 遍历张量分割参数
    for (const auto & nb : params.n_batch) // 遍历批次数参数
    for (const auto & fk : params.f32_kv) // 遍历 f32_kv 参数
    for (const auto & mmq : params.mul_mat_q) // 遍历 mul_mat_q 参数
    for (const auto & nt : params.n_threads) { // 遍历线程数参数
        for (const auto & n_prompt : params.n_prompt) { // 遍历提示数参数
            if (n_prompt == 0) { // 如果提示数为 0，则跳过
                continue;
            }
            // 创建命令参数实例
            cmd_params_instance instance = {
                /* .model        = */ m, // 模型参数
        /* .n_prompt     = */ n_prompt,  // 设置提示数量
        /* .n_gen        = */ 0,          // 设置生成数量
        /* .n_batch      = */ nb,         // 设置批处理数量
        /* .f32_kv       = */ fk,         // 设置键值对
        /* .n_threads    = */ nt,         // 设置线程数量
        /* .n_gpu_layers = */ nl,         // 设置GPU层数
        /* .main_gpu     = */ mg,         // 设置主GPU
        /* .mul_mat_q    = */ mmq,        // 设置矩阵乘法Q
        /* .tensor_split = */ ts,         // 设置张量分割
    };
    instances.push_back(instance);  // 将实例添加到实例列表中

    for (const auto & n_gen : params.n_gen) {  // 遍历生成数量参数
        if (n_gen == 0) {  // 如果生成数量为0，则跳过
            continue;
        }
        cmd_params_instance instance = {
            /* .model        = */ m,   // 设置模型
            /* .n_prompt     = */ 0,   // 设置提示数量为0
                /* .n_gen        = */ n_gen,  // 设置生成数量
                /* .n_batch      = */ nb,      // 设置批处理大小
                /* .f32_kv       = */ fk,      // 设置f32_kv
                /* .n_threads    = */ nt,      // 设置线程数量
                /* .n_gpu_layers = */ nl,      // 设置GPU层数
                /* .main_gpu     = */ mg,      // 设置主GPU
                /* .mul_mat_q    = */ mmq,     // 设置mul_mat_q
                /* .tensor_split = */ ts,      // 设置tensor_split
            };
            instances.push_back(instance);  // 将实例添加到instances数组中
        }
    }
#else
    // this ordering separates the prompt and generation tests
    // 这个顺序将提示和生成测试分开
    for (const auto & n_prompt : params.n_prompt) {  // 遍历n_prompt数组
        if (n_prompt == 0) {  // 如果n_prompt为0，则跳过
            continue;
        }
        auto instances_prompt = get_cmd_params_instances_int(params, 0, n_prompt);  // 获取n_prompt的实例
        instances.insert(instances.end(), instances_prompt.begin(), instances_prompt.end());  // 将实例添加到instances数组中
    // 结构体 test 包含了一些静态常量成员变量，分别表示构建提交、构建编号、是否支持 CUDA、是否支持 OpenCL、是否支持 Metal
    struct test {
        static const std::string build_commit; // 构建提交的字符串常量
        static const int build_number; // 构建编号的整数常量
        static const bool cuda; // 是否支持 CUDA 的布尔常量
        static const bool opencl; // 是否支持 OpenCL 的布尔常量
        static const bool metal; // 是否支持 Metal 的布尔常量
    // 声明一个静态常量布尔变量 gpu_blas
    static const bool gpu_blas;
    // 声明一个静态常量布尔变量 blas
    static const bool blas;
    // 声明一个静态常量字符串变量 cpu_info
    static const std::string cpu_info;
    // 声明一个静态常量字符串变量 gpu_info
    static const std::string gpu_info;
    // 声明一个字符串变量 model_filename
    std::string model_filename;
    // 声明一个字符串变量 model_type
    std::string model_type;
    // 声明一个64位无符号整数变量 model_size
    uint64_t model_size;
    // 声明一个64位无符号整数变量 model_n_params
    uint64_t model_n_params;
    // 声明一个整数变量 n_batch
    int n_batch;
    // 声明一个整数变量 n_threads
    int n_threads;
    // 声明一个布尔变量 f32_kv
    bool f32_kv;
    // 声明一个整数变量 n_gpu_layers
    int n_gpu_layers;
    // 声明一个整数变量 main_gpu
    int main_gpu;
    // 声明一个布尔变量 mul_mat_q
    bool mul_mat_q;
    // 声明一个包含 LLAMA_MAX_DEVICES 个元素的浮点数数组 tensor_split
    std::array<float, LLAMA_MAX_DEVICES> tensor_split;
    // 声明一个整数变量 n_prompt
    int n_prompt;
    // 声明一个整数变量 n_gen
    int n_gen;
    // 声明一个字符串变量 test_time
    std::string test_time;
    // 声明一个包含无符号64位整数的向量 samples_ns
    std::vector<uint64_t> samples_ns;
    // 定义一个测试函数，接受命令参数实例、llama模型和llama上下文作为参数
    test(const cmd_params_instance & inst, const llama_model * lmodel, const llama_context * ctx) {
        // 设置模型文件名为命令参数实例中的模型文件名
        model_filename = inst.model;
        // 定义一个长度为128的字符数组buf，用于存储模型描述信息
        char buf[128];
        // 获取llama模型的描述信息，存储在buf中
        llama_model_desc(lmodel, buf, sizeof(buf));
        // 将模型描述信息转换为字符串，存储在model_type中
        model_type = buf;
        // 获取llama模型的大小，存储在model_size中
        model_size = llama_model_size(lmodel);
        // 获取llama模型的参数数量，存储在model_n_params中
        model_n_params = llama_model_n_params(lmodel);
        // 设置批处理大小为命令参数实例中的批处理大小
        n_batch = inst.n_batch;
        // 设置线程数量为命令参数实例中的线程数量
        n_threads = inst.n_threads;
        // 设置f32_kv为命令参数实例中的f32_kv
        f32_kv = inst.f32_kv;
        // 设置GPU层数为命令参数实例中的GPU层数
        n_gpu_layers = inst.n_gpu_layers;
        // 设置主GPU为命令参数实例中的主GPU
        main_gpu = inst.main_gpu;
        // 设置mul_mat_q为命令参数实例中的mul_mat_q
        mul_mat_q = inst.mul_mat_q;
        // 设置tensor_split为命令参数实例中的tensor_split
        tensor_split = inst.tensor_split;
        // 设置提示数量为命令参数实例中的提示数量
        n_prompt = inst.n_prompt;
        // 设置生成数量为命令参数实例中的生成数量
        n_gen = inst.n_gen;
        // 获取当前时间，并按照RFC 3339日期时间格式存储在buf中
        time_t t = time(NULL);
        std::strftime(buf, sizeof(buf), "%FT%TZ", gmtime(&t));
        // 将格式化后的时间字符串存储在test_time中
        test_time = buf;
    }
    // 空函数，不做任何操作
    (void) ctx;
    }

    // 返回样本数据的平均值（单位：纳秒）
    uint64_t avg_ns() const {
        return ::avg(samples_ns);
    }

    // 返回样本数据的标准差（单位：纳秒）
    uint64_t stdev_ns() const {
        return ::stdev(samples_ns);
    }

    // 获取时间戳数据
    std::vector<double> get_ts() const {
        // 计算总的标记数
        int n_tokens = n_prompt + n_gen;
        // 创建一个空的时间戳数组
        std::vector<double> ts;
        // 将样本数据转换为时间戳数据
        std::transform(samples_ns.begin(), samples_ns.end(), std::back_inserter(ts), [n_tokens](uint64_t t) { return 1e9 * n_tokens / t; });
        // 返回时间戳数组
        return ts;
    }

    // 返回时间戳数据的平均值
    double avg_ts() const {
    // 返回时间序列的平均值
    return ::avg(get_ts());
}

// 返回时间序列的标准差
double stdev_ts() const {
    return ::stdev(get_ts());
}

// 获取后端类型
static std::string get_backend() {
    // 如果是 CUDA 后端，则返回对应的名称
    if (cuda) {
        return GGML_CUDA_NAME;
    }
    // 如果是 OpenCL 后端，则返回对应的名称
    if (opencl) {
        return "OpenCL";
    }
    // 如果是 Metal 后端，则返回对应的名称
    if (metal) {
        return "Metal";
    }
    // 如果是 GPU BLAS 后端，则返回对应的名称
    if (gpu_blas) {
        return "GPU BLAS";
    }
    // 如果存在 BLAS，则返回 "BLAS"，否则返回 "CPU"
    if (blas) {
        return "BLAS";
    }
    return "CPU";
}

// 获取字段列表的静态方法
static const std::vector<std::string> & get_fields() {
    // 定义静态常量字段列表
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
// 定义枚举类型 field_type，包括 STRING、BOOL、INT、FLOAT
enum field_type {STRING, BOOL, INT, FLOAT};

// 根据字段名获取字段类型
static field_type get_field_type(const std::string & field) {
    // 如果字段名匹配整型字段，则返回 INT 类型
    if (field == "build_number" || field == "n_batch" || field == "n_threads" ||
        field == "model_size" || field == "model_n_params" ||
        field == "n_gpu_layers" || field == "main_gpu" ||
        field == "n_prompt" || field == "n_gen" ||
        field == "avg_ns" || field == "stddev_ns") {
        return INT;
    }
    // 如果字段名匹配布尔型字段，则返回 BOOL 类型
    if (field == "cuda" || field == "opencl" || field == "metal" || field == "gpu_blas" || field == "blas" ||
        field == "f16_kv" || field == "mul_mat_q") {
        return BOOL;
    }
    // 如果字段名匹配浮点型字段，则返回 FLOAT 类型
    if (field == "avg_ts" || field == "stddev_ts") {
        return FLOAT;
    }
    // 默认返回 STRING 类型
    return STRING;
}
    // 返回一个字符串向量，包含了各种数值和构建信息
    std::vector<std::string> get_values() const {
        // 初始化一个空字符串
        std::string tensor_split_str;
        // 初始化一个最大非零值为0
        int max_nonzero = 0;
        // 遍历tensor_split数组，找到最大的非零值
        for (int i = 0; i < LLAMA_MAX_DEVICES; i++) {
            if (tensor_split[i] > 0) {
                max_nonzero = i;
            }
        }
        // 将tensor_split数组中的值转换为字符串，并拼接到tensor_split_str中
        for (int i = 0; i <= max_nonzero; i++) {
            char buf[32];
            // 将tensor_split[i]的值格式化为字符串，存储到buf中
            snprintf(buf, sizeof(buf), "%.2f", tensor_split[i]);
            // 将buf中的字符串拼接到tensor_split_str中
            tensor_split_str += buf;
            // 如果不是最后一个非零值，添加"/"分隔符
            if (i < max_nonzero) {
                tensor_split_str += "/";
            }
        }
        // 构建一个包含构建信息和数值的字符串向量
        std::vector<std::string> values = {
            build_commit, std::to_string(build_number),
            std::to_string(cuda), std::to_string(opencl), std::to_string(metal), std::to_string(gpu_blas), std::to_string(blas),
// 获取 CPU 信息、GPU 信息、模型文件名、模型类型、模型大小、模型参数数量、批处理数量、线程数量、是否使用 f32_kv、GPU 层数量、主 GPU 编号、是否进行矩阵乘法、张量分割字符串、提示数量、生成数量、测试时间、平均时间、时间标准差、平均时间戳、时间戳标准差，并将它们存储在 values 中
auto values = {
    cpu_info, gpu_info, model_filename, model_type, std::to_string(model_size), std::to_string(model_n_params),
    std::to_string(n_batch), std::to_string(n_threads), std::to_string(!f32_kv),
    std::to_string(n_gpu_layers), std::to_string(main_gpu), std::to_string(mul_mat_q), tensor_split_str,
    std::to_string(n_prompt), std::to_string(n_gen), test_time,
    std::to_string(avg_ns()), std::to_string(stdev_ns()),
    std::to_string(avg_ts()), std::to_string(stdev_ts())
};
// 返回存储了各项信息的 values
return values;
}

// 获取字段名称和对应的值，并将它们存储在 map 中
std::map<std::string, std::string> get_map() const {
    std::map<std::string, std::string> map;
    // 获取字段名称
    auto fields = get_fields();
    // 获取字段对应的值
    auto values = get_values();
    // 将字段名称和对应的值组合成键值对，并插入到 map 中
    std::transform(fields.begin(), fields.end(), values.begin(),
            std::inserter(map, map.end()), std::make_pair<const std::string &, const std::string &>);
    // 返回存储了字段名称和对应值的 map
    return map;
}
// 设置构建提交的版本号
const std::string test::build_commit = LLAMA_COMMIT;
// 设置构建的版本号
const int         test::build_number = LLAMA_BUILD_NUMBER;
// 检查是否支持 CUDA
const bool        test::cuda         = !!ggml_cpu_has_cublas();
// 检查是否支持 OpenCL
const bool        test::opencl       = !!ggml_cpu_has_clblast();
// 检查是否支持 Metal
const bool        test::metal        = !!ggml_cpu_has_metal();
// 检查是否支持 GPU BLAS
const bool        test::gpu_blas     = !!ggml_cpu_has_gpublas();
// 检查是否支持 BLAS
const bool        test::blas         = !!ggml_cpu_has_blas();
// 获取 CPU 信息
const std::string test::cpu_info     = get_cpu_info();
// 获取 GPU 信息
const std::string test::gpu_info     = get_gpu_info();

// 定义打印器结构
struct printer {
    virtual ~printer() {}

    // 输出文件指针
    FILE * fout;
    // 打印测试头部信息
    virtual void print_header(const cmd_params & params) { (void) params; }
    // 打印测试信息
    virtual void print_test(const test & t) = 0;
    // 打印测试尾部信息
    virtual void print_footer() { }
};
// 定义一个结构体 csv_printer，继承自 printer
struct csv_printer : public printer {
    // 定义一个静态方法，用于转义 CSV 中的特殊字符
    static std::string escape_csv(const std::string & field) {
        // 初始化转义后的字符串
        std::string escaped = "\"";
        // 遍历字段中的每个字符
        for (auto c : field) {
            // 如果字符是双引号，则在转义字符串中再添加一个双引号
            if (c == '"') {
                escaped += "\"";
            }
            // 添加当前字符到转义字符串中
            escaped += c;
        }
        // 在转义字符串末尾添加双引号
        escaped += "\"";
        // 返回转义后的字符串
        return escaped;
    }

    // 实现打印表头的方法
    void print_header(const cmd_params & params) override  {
        // 获取字段列表
        std::vector<std::string> fields = test::get_fields();
        // 将字段列表用逗号连接并输出到文件流中
        fprintf(fout, "%s\n", join(fields, ",").c_str());
        // 忽略参数
        (void) params;
    }

    // 实现打印测试数据的方法
    void print_test(const test & t) override {
// 从给定对象 t 中获取值并存储在 values 向量中
std::vector<std::string> values = t.get_values();
// 对 values 向量中的每个字符串应用 escape_csv 函数进行转义处理
std::transform(values.begin(), values.end(), values.begin(), escape_csv);
// 将 values 向量中的字符串用逗号连接成一个字符串，并输出到 fout 文件中
fprintf(fout, "%s\n", join(values, ",").c_str());
}

// 定义一个继承自 printer 的结构体 json_printer
struct json_printer : public printer {
    // 声明一个布尔类型的变量 first，并初始化为 true
    bool first = true;

    // 定义一个静态函数 escape_json，用于对给定的字符串进行 JSON 转义处理
    static std::string escape_json(const std::string & value) {
        // 声明一个字符串类型的变量 escaped，用于存储转义后的字符串
        std::string escaped;
        // 遍历给定字符串 value 中的每个字符
        for (auto c : value) {
            // 如果字符为双引号，则转义为 \"
            if (c == '"') {
                escaped += "\\\"";
            } 
            // 如果字符为反斜杠，则转义为 \\
            else if (c == '\\') {
                escaped += "\\\\";
            } 
            // 如果字符的 ASCII 值小于等于 0x1f，则转义为 Unicode 编码
            else  if (c <= 0x1f) {
                // 声明一个长度为 8 的字符数组 buf，用于存储转义后的 Unicode 编码
                char buf[8];
                // 将字符 c 转义为 Unicode 编码，并存储到 buf 中
                snprintf(buf, sizeof(buf), "\\u%04x", c);
                // 将 buf 中的内容添加到 escaped 中
                escaped += buf;
    } else {
        // 如果字符不需要转义，则直接添加到已转义的字符串中
        escaped += c;
    }
}
// 返回转义后的字符串
return escaped;
}

static std::string format_value(const std::string & field, const std::string & value) {
    // 根据字段类型格式化数值
    switch (test::get_field_type(field)) {
        case test::STRING:
            // 如果是字符串类型，则在值两侧添加双引号，并进行 JSON 转义
            return "\"" + escape_json(value) + "\"";
        case test::BOOL:
            // 如果是布尔类型，则根据值返回对应的 JSON 格式
            return value == "0" ? "false" : "true";
        default:
            // 其他类型直接返回值
            return value;
    }
}

void print_header(const cmd_params & params) override {
    // 在输出流中打印头部信息
    fprintf(fout, "[\n");
    // 忽略参数
    (void) params;
    }

    // 打印字段和对应的数值
    void print_fields(const std::vector<std::string> & fields, const std::vector<std::string> & values) {
        // 断言字段和数值的数量相等
        assert(fields.size() == values.size());
        // 遍历字段和数值，打印每个字段和对应的格式化数值
        for (size_t i = 0; i < fields.size(); i++) {
            fprintf(fout, "    \"%s\": %s,\n", fields.at(i).c_str(), format_value(fields.at(i), values.at(i)).c_str());
        }
    }

    // 打印测试数据
    void print_test(const test & t) override {
        // 如果是第一个测试数据，则不打印逗号
        if (first) {
            first = false;
        } else {
            fprintf(fout, ",\n");
        }
        // 打印测试数据的字段和数值
        fprintf(fout, "  {\n");
        print_fields(test::get_fields(), t.get_values());
        // 打印样本的纳秒时间戳
        fprintf(fout, "    \"samples_ns\": [ %s ],\n", join(t.samples_ns, ", ").c_str());
        // 打印样本的时间戳
        fprintf(fout, "    \"samples_ts\": [ %s ]\n", join(t.get_ts(), ", ").c_str());
        // 将字符串 "  }" 写入到输出文件中
        fprintf(fout, "  }");
        // 刷新输出缓冲区
        fflush(fout);
    }

    // 打印 Markdown 格式的表格尾部
    void print_footer() override {
        // 将字符串 "\n]\n" 写入到输出文件中
        fprintf(fout, "\n]\n");
    }
};

// Markdown 格式的打印器，继承自 printer 类
struct markdown_printer : public printer {
    // 字段名称的字符串向量
    std::vector<std::string> fields;

    // 获取字段宽度的静态方法
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
        // 如果字段为"n_gpu_layers"，返回值为3
        if (field == "n_gpu_layers") {
            return 3;
        }

        // 计算字段长度和10的最大值
        int width = std::max((int)field.length(), 10);

        // 如果字段类型为字符串，返回值为负的字段长度
        if (test::get_field_type(field) == test::STRING) {
            return -width;
        }
        // 返回字段长度
        return width;
    }

    // 获取字段的显示名称
    static std::string get_field_display_name(const std::string & field) {
        // 如果字段为"n_gpu_layers"，返回值为"ngl"
        if (field == "n_gpu_layers") {
            return "ngl";
        }
        // 如果字段为"n_threads"，返回值为"threads"
        if (field == "n_threads") {
            return "threads";
    }
    // 如果字段为 "mul_mat_q"，返回 "mmq"
    if (field == "mul_mat_q") {
        return "mmq";
    }
    // 如果字段为 "tensor_split"，返回 "ts"
    if (field == "tensor_split") {
        return "ts";
    }
    // 如果字段不是以上两种情况，直接返回字段值
    return field;
}

void print_header(const cmd_params & params) override {
    // 选择要打印的字段
    fields.push_back("model");
    fields.push_back("size");
    fields.push_back("params");
    fields.push_back("backend");
    // 检查当前测试的后端是否为 CPU 或 BLAS，如果不是则添加 "n_gpu_layers" 字段
    bool is_cpu_backend = test::get_backend() == "CPU" || test::get_backend() == "BLAS";
    if (!is_cpu_backend) {
        fields.push_back("n_gpu_layers");
    }
}
# 如果线程数大于1，或者线程数不等于默认值，或者使用CPU后端，则将字段"n_threads"加入到字段列表中
if (params.n_threads.size() > 1 || params.n_threads != cmd_params_defaults.n_threads || is_cpu_backend) {
    fields.push_back("n_threads");
}

# 如果批处理大小大于1，或者批处理大小不等于默认值，则将字段"n_batch"加入到字段列表中
if (params.n_batch.size() > 1 || params.n_batch != cmd_params_defaults.n_batch) {
    fields.push_back("n_batch");
}

# 如果f32_kv的大小大于1，或者f32_kv不等于默认值，则将字段"f16_kv"加入到字段列表中
if (params.f32_kv.size() > 1 || params.f32_kv != cmd_params_defaults.f32_kv) {
    fields.push_back("f16_kv");
}

# 如果主GPU的大小大于1，或者主GPU不等于默认值，则将字段"main_gpu"加入到字段列表中
if (params.main_gpu.size() > 1 || params.main_gpu != cmd_params_defaults.main_gpu) {
    fields.push_back("main_gpu");
}

# 如果mul_mat_q的大小大于1，或者mul_mat_q不等于默认值，则将字段"mul_mat_q"加入到字段列表中
if (params.mul_mat_q.size() > 1 || params.mul_mat_q != cmd_params_defaults.mul_mat_q) {
    fields.push_back("mul_mat_q");
}

# 如果tensor_split的大小大于1，或者tensor_split不等于默认值，则将字段"tensor_split"加入到字段列表中
if (params.tensor_split.size() > 1 || params.tensor_split != cmd_params_defaults.tensor_split) {
    fields.push_back("tensor_split");
}

# 将字段"test"加入到字段列表中
fields.push_back("test");

# 将字段"t/s"加入到字段列表中
fields.push_back("t/s");
// 在输出文件中打印竖线
fprintf(fout, "|");
// 遍历字段列表，打印每个字段的名称，并根据字段宽度对齐
for (const auto & field : fields) {
    fprintf(fout, " %*s |", get_field_width(field), get_field_display_name(field).c_str());
}
// 在输出文件中换行
fprintf(fout, "\n");
// 在输出文件中打印横线
fprintf(fout, "|");
// 遍历字段列表，根据字段宽度打印横线，用于表格分隔
for (const auto & field : fields) {
    int width = get_field_width(field);
    fprintf(fout, " %s%s |", std::string(std::abs(width) - 1, '-').c_str(), width > 0 ? ":" : "-");
}
// 在输出文件中换行
fprintf(fout, "\n");
// 打印测试数据到输出文件
void print_test(const test & t) override {
    // 获取测试数据的键值对
    std::map<std::string, std::string> vmap = t.get_map();
    // 在输出文件中打印竖线
    fprintf(fout, "|");
    // 遍历字段列表，打印每个字段的值
    for (const auto & field : fields) {
        std::string value;
// 声明一个字符数组 buf，用于存储临时字符串
char buf[128];
// 如果字段为 "model"，则将 value 设置为 t.model_type
if (field == "model") {
    value = t.model_type;
} 
// 如果字段为 "size"
else if (field == "size") {
    // 如果模型大小小于 1GB，则将 buf 格式化为 "%.2f MiB" 的字符串
    if (t.model_size < 1024*1024*1024) {
        snprintf(buf, sizeof(buf), "%.2f MiB", t.model_size / 1024.0 / 1024.0);
    } 
    // 如果模型大小大于等于 1GB，则将 buf 格式化为 "%.2f GiB" 的字符串
    else {
        snprintf(buf, sizeof(buf), "%.2f GiB", t.model_size / 1024.0 / 1024.0 / 1024.0);
    }
    // 将 value 设置为 buf 中的内容
    value = buf;
} 
// 如果字段为 "params"
else if (field == "params") {
    // 如果模型参数数量小于 1 billion，则将 buf 格式化为 "%.2f M" 的字符串
    if (t.model_n_params < 1000*1000*1000) {
        snprintf(buf, sizeof(buf), "%.2f M", t.model_n_params / 1e6);
    } 
    // 如果模型参数数量大于等于 1 billion，则将 buf 格式化为 "%.2f B" 的字符串
    else {
        snprintf(buf, sizeof(buf), "%.2f B", t.model_n_params / 1e9);
    }
    // 将 value 设置为 buf 中的内容
    value = buf;
} 
// 如果字段为 "backend"，则将 value 设置为 test::get_backend() 的返回值
else if (field == "backend") {
    value = test::get_backend();
} 
// 如果字段为 "test"
else if (field == "test") {
    // 这里缺少对 "test" 字段的处理逻辑
}
# 如果提示次数大于0且生成次数等于0，则将提示次数格式化到buf中
if (t.n_prompt > 0 && t.n_gen == 0) {
    snprintf(buf, sizeof(buf), "pp %d", t.n_prompt);
} 
# 如果生成次数大于0且提示次数等于0，则将生成次数格式化到buf中
else if (t.n_gen > 0 && t.n_prompt == 0) {
    snprintf(buf, sizeof(buf), "tg %d", t.n_gen);
} 
# 如果以上条件都不满足，则断言为假，退出程序
else {
    assert(false);
    exit(1);
}
# 将buf的值赋给value
value = buf;

# 如果字段为"t/s"
if (field == "t/s") {
    # 将t的平均时间和标准差格式化到buf中
    snprintf(buf, sizeof(buf), "%.2f ± %.2f", t.avg_ts(), t.stdev_ts());
    # 将buf的值赋给value
    value = buf;
} 
# 如果字段在vmap中存在
else if (vmap.find(field) != vmap.end()) {
    # 将vmap中对应字段的值赋给value
    value = vmap.at(field);
} 
# 如果以上条件都不满足，则断言为假，退出程序
else {
    assert(false);
    exit(1);
}

# 获取字段的宽度
int width = get_field_width(field);
            if (field == "t/s") {
                // 如果字段是 "t/s"，则增加宽度，这是一个临时的解决方案
                width += 1;
            }
            // 输出格式化字符串和值到文件
            fprintf(fout, " %*s |", width, value.c_str());
        }
        // 输出换行符
        fprintf(fout, "\n");
    }

    // 打印页脚信息
    void print_footer() override {
        // 输出构建信息到文件
        fprintf(fout, "\nbuild: %s (%d)\n", test::build_commit.c_str(), test::build_number);
    }
};

// SQL 打印器，继承自打印器
struct sql_printer : public printer {
    // 获取 SQL 字段类型
    static std::string get_sql_field_type(const std::string & field) {
        // 根据字段类型返回对应的 SQL 类型
        switch (test::get_field_type(field)) {
            case test::STRING:
                return "TEXT";
            case test::BOOL:
// 根据不同的测试类型返回对应的数据库字段类型
case test::INT:
    return "INTEGER";
case test::FLOAT:
    return "REAL";
default:
    assert(false); // 如果不是整数或浮点数类型，触发断言
    exit(1); // 退出程序，返回错误状态码

// 打印表头信息
void print_header(const cmd_params & params) override {
    std::vector<std::string> fields = test::get_fields(); // 获取测试字段列表
    fprintf(fout, "CREATE TABLE IF NOT EXISTS test (\n"); // 打印创建表的 SQL 语句
    for (size_t i = 0; i < fields.size(); i++) {
        fprintf(fout, "  %s %s%s\n", fields.at(i).c_str(), get_sql_field_type(fields.at(i)).c_str(),  i < fields.size() - 1 ? "," : ""); // 打印字段名和字段类型
    }
    fprintf(fout, ");\n"); // 结束创建表的 SQL 语句
    fprintf(fout, "\n"); // 打印空行
    (void) params; // 防止未使用参数的警告
}
// 重写 print_test 方法，输出 test 对象的信息到文件
void print_test(const test & t) override {
    // 输出 INSERT INTO test (字段列表) 
    fprintf(fout, "INSERT INTO test (%s) ", join(test::get_fields(), ", ").c_str());
    // 输出 VALUES (
    fprintf(fout, "VALUES (");
    // 获取 test 对象的值列表
    std::vector<std::string> values = t.get_values();
    // 遍历值列表，输出每个值
    for (size_t i = 0; i < values.size(); i++) {
        fprintf(fout, "'%s'%s", values.at(i).c_str(), i < values.size() - 1 ? ", " : "");
    }
    // 输出 );
    fprintf(fout, ");\n");
}

// 测试提示方法，使用 llama_context 对象和其他参数
static void test_prompt(llama_context * ctx, int n_prompt, int n_past, int n_batch, int n_threads) {
    // 创建包含 n_batch 个 llama_token_bos 对象的 tokens 向量
    std::vector<llama_token> tokens(n_batch, llama_token_bos(llama_get_model(ctx)));
    // 初始化已处理的 token 数量为 0
    int n_processed = 0;

    // 设置 llama_context 对象的线程数量
    llama_set_n_threads(ctx, n_threads, n_threads);

    // 当已处理的 token 数量小于 n_prompt 时，执行循环
    while (n_processed < n_prompt) {
        // 计算本次循环要处理的 token 数量
        int n_tokens = std::min(n_prompt - n_processed, n_batch);
// 使用llama_batch_get_one函数从tokens.data()中获取n_tokens个token，并解码
llama_decode(ctx, llama_batch_get_one(tokens.data(), n_tokens, n_past + n_processed, 0));
// 更新已处理的token数量
n_processed += n_tokens;
}

// 生成n_gen个token序列
static void test_gen(llama_context * ctx, int n_gen, int n_past, int n_threads) {
    // 获取一个开始token
    llama_token token = llama_token_bos(llama_get_model(ctx));

    // 设置线程数
    llama_set_n_threads(ctx, n_threads, n_threads);

    // 生成n_gen个token序列
    for (int i = 0; i < n_gen; i++) {
        llama_decode(ctx, llama_batch_get_one(&token, 1, n_past + i, 0));
    }
}

// 空日志回调函数
static void llama_null_log_callback(enum ggml_log_level level, const char * text, void * user_data) {
    // 忽略日志级别、文本和用户数据
    (void) level;
    (void) text;
    (void) user_data;
}
// 主函数，接受命令行参数
int main(int argc, char ** argv) {
    // 尝试设置本地化环境，以支持 markdown 中的 Unicode 字符
    setlocale(LC_CTYPE, ".UTF-8");

    // 如果处于调试模式，输出警告信息
#if !defined(NDEBUG)
    fprintf(stderr, "warning: asserts enabled, performance may be affected\n");
#endif

    // 如果是调试构建，输出警告信息
#if (defined(_MSC_VER) && defined(_DEBUG)) || (!defined(_MSC_VER) && !defined(__OPTIMIZE__))
    fprintf(stderr, "warning: debug build, performance may be affected\n");
#endif

    // 如果启用了内存或线程检测工具，输出警告信息
#if defined(__SANITIZE_ADDRESS__) || defined(__SANITIZE_THREAD__)
    fprintf(stderr, "warning: sanitizer enabled, performance may be affected\n");
#endif

    // 解析命令行参数
    cmd_params params = parse_cmd_params(argc, argv);

    // 初始化 llama.cpp
    // 如果参数中没有设置详细输出，则设置日志回调为空
    if (!params.verbose) {
        llama_log_set(llama_null_log_callback, NULL);
    }
    // 初始化 NUMA（Non-Uniform Memory Access）支持
    bool numa = false;
    llama_backend_init(numa);

    // 初始化打印机
    std::unique_ptr<printer> p;
    // 根据参数中的输出格式选择不同的打印机
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
```

在这段代码中，根据参数中的设置，选择不同的打印机进行初始化。如果参数中没有设置详细输出，则设置日志回调为空。同时，还进行了 NUMA 的初始化。
    // 结束 switch 语句
    break;
    // 默认情况下，断言为假，退出程序
    default:
        assert(false);
        exit(1);
    }
    // 将输出流设置为标准输出
    p->fout = stdout;
    // 打印参数的头部信息
    p->print_header(params);

    // 获取命令参数的实例
    std::vector<cmd_params_instance> params_instances = get_cmd_params_instances(params);

    // 初始化 llama_model 和 prev_inst
    llama_model * lmodel = nullptr;
    const cmd_params_instance * prev_inst = nullptr;

    // 遍历参数实例
    for (const auto & inst : params_instances) {
        // 在可能的情况下保持相同的模型
        if (!lmodel || !prev_inst || !inst.equal_mparams(*prev_inst)) {
            // 如果 lmodel 不为空，释放其内存
            if (lmodel) {
                llama_free_model(lmodel);
            }
// 从文件中加载 LLAMA 模型，并将其转换为 LLAMA 模型参数
lmodel = llama_load_model_from_file(inst.model.c_str(), inst.to_llama_mparams());
// 如果加载模型失败，则打印错误信息并返回 1
if (lmodel == NULL) {
    fprintf(stderr, "%s: error: failed to load model '%s'\n", __func__, inst.model.c_str());
    return 1;
}
// 将当前实例指针赋值给 prev_inst
prev_inst = &inst;

// 使用加载的模型创建 LLAMA 上下文，并将实例参数转换为 LLAMA 上下文参数
llama_context * ctx = llama_new_context_with_model(lmodel, inst.to_llama_cparams());
// 如果创建上下文失败，则打印错误信息，释放模型内存，并返回 1
if (ctx == NULL) {
    fprintf(stderr, "%s: error: failed to create context with model '%s'\n", __func__, inst.model.c_str());
    llama_free_model(lmodel);
    return 1;
}

// 使用实例、模型和上下文创建测试对象
test t(inst, lmodel, ctx);

// 清空 LLAMA 键值缓存
llama_kv_cache_clear(ctx);

// 进行预热运行
# 如果有提示数据，则进行测试
if (t.n_prompt > 0) {
    # 测试提示数据
    test_prompt(ctx, std::min(2, t.n_batch), 0, t.n_batch, t.n_threads);
}
# 如果有生成数据，则进行测试
if (t.n_gen > 0) {
    # 测试生成数据
    test_gen(ctx, 1, 0, t.n_threads);
}

# 循环执行测试
for (int i = 0; i < params.reps; i++) {
    # 清除缓存
    llama_kv_cache_clear(ctx);

    # 记录开始时间
    uint64_t t_start = get_time_ns();
    # 如果有提示数据，则进行测试
    if (t.n_prompt > 0) {
        # 测试提示数据
        test_prompt(ctx, t.n_prompt, 0, t.n_batch, t.n_threads);
    }
    # 如果有生成数据，则进行测试
    if (t.n_gen > 0) {
        # 测试生成数据
        test_gen(ctx, t.n_gen, t.n_prompt, t.n_threads);
    }
    # 计算测试时间并将结果添加到样本列表中
    uint64_t t_ns = get_time_ns() - t_start;
    t.samples_ns.push_back(t_ns);
}
    // 调用print_test方法，传入参数t
    p->print_test(t);

    // 打印时间信息
    llama_print_timings(ctx);

    // 释放上下文资源
    llama_free(ctx);
}

// 释放模型资源
llama_free_model(lmodel);

// 调用print_footer方法
p->print_footer();

// 释放后端资源
llama_backend_free();

// 返回0表示程序正常结束
return 0;
}
```