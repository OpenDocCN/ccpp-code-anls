# `whisper.cpp\examples\bench\bench.cpp`

```cpp
// 包含头文件 "whisper.h"
#include "whisper.h"

// 包含标准输入输出头文件
#include <cstdio>
// 包含字符串处理头文件
#include <cstring>
// 包含字符串处理头文件
#include <string>
// 包含线程头文件
#include <thread>

// 声明命令行参数结构体 whisper_params
struct whisper_params {
    // 线程数，默认为最小值：4 和硬件并发数的较小值
    int32_t n_threads = std::min(4, (int32_t) std::thread::hardware_concurrency());
    // 用于指定要进行基准测试的内容：0 - whisper ecoder, 1 - memcpy, 2 - ggml_mul_mat
    int32_t what = 0;

    // 模型路径，默认为 "models/ggml-base.en.bin"
    std::string model = "models/ggml-base.en.bin";

    // 是否使用 GPU，默认为 true
    bool use_gpu = true;
};

// 打印使用说明
void whisper_print_usage(int argc, char ** argv, const whisper_params & params);

// 解析命令行参数
bool whisper_params_parse(int argc, char ** argv, whisper_params & params) {
    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数
        std::string arg = argv[i];

        // 判断参数类型
        if (arg == "-h" || arg == "--help") {
            // 打印使用说明并退出程序
            whisper_print_usage(argc, argv, params);
            exit(0);
        }
        else if (arg == "-t"  || arg == "--threads") { params.n_threads = std::stoi(argv[++i]); }
        else if (arg == "-m"  || arg == "--model")   { params.model     = argv[++i]; }
        else if (arg == "-w"  || arg == "--what")    { params.what      = atoi(argv[++i]); }
        else if (arg == "-ng" || arg == "--no-gpu")  { params.use_gpu   = false; }
        else {
            // 打印错误信息并退出程序
            fprintf(stderr, "error: unknown argument: %s\n", arg.c_str());
            whisper_print_usage(argc, argv, params);
            exit(0);
        }
    }

    return true;
}

// 打印使用说明
void whisper_print_usage(int /*argc*/, char ** argv, const whisper_params & params) {
    // 打印使用说明
    fprintf(stderr, "\n");
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    fprintf(stderr, "\n");
    fprintf(stderr, "options:\n");
    fprintf(stderr, "  -h,       --help        [default] show this help message and exit\n");
    fprintf(stderr, "  -t N,     --threads N   [%-7d] number of threads to use during computation\n", params.n_threads);
    fprintf(stderr, "  -m FNAME, --model FNAME [%-7s] model path\n",                                  params.model.c_str());
    # 打印带有格式的信息到标准错误输出，显示参数 N 对应的 what 值
    fprintf(stderr, "  -w N,     --what N      [%-7d] what to benchmark:\n",                          params.what);
    # 打印带有格式的信息到标准错误输出，显示是否禁用 GPU
    fprintf(stderr, "  -ng,      --no-gpu      [%-7s] disable GPU\n",                                 params.use_gpu ? "false" : "true");
    # 打印带有格式的信息到标准错误输出，显示不同的 what 值对应的功能
    fprintf(stderr, "                           %-7s  0 - whisper\n",                                 "");
    fprintf(stderr, "                           %-7s  1 - memcpy\n",                                  "");
    fprintf(stderr, "                           %-7s  2 - ggml_mul_mat\n",                            "");
    # 打印空行到标准错误输出
    fprintf(stderr, "\n");
}

int whisper_bench_full(const whisper_params & params) {
    // 初始化 whisper

    // 定义 whisper 上下文参数结构体
    struct whisper_context_params cparams;
    cparams.use_gpu = params.use_gpu;

    // 从文件初始化 whisper 上下文
    struct whisper_context * ctx = whisper_init_from_file_with_params(params.model.c_str(), cparams);

    {
        // 输出系统信息
        fprintf(stderr, "\n");
        fprintf(stderr, "system_info: n_threads = %d / %d | %s\n", params.n_threads, std::thread::hardware_concurrency(), whisper_print_system_info());
    }

    // 如果初始化失败，输出错误信息并返回
    if (ctx == nullptr) {
        fprintf(stderr, "error: failed to initialize whisper context\n");
        return 2;
    }

    // 获取 mel 频谱的数量
    const int n_mels = whisper_model_n_mels(ctx);

    // 设置 mel 频谱
    if (int ret = whisper_set_mel(ctx, nullptr, 0, n_mels)) {
        fprintf(stderr, "error: failed to set mel: %d\n", ret);
        return 3;
    }
    // 热编码器
    if (int ret = whisper_encode(ctx, 0, params.n_threads) != 0) {
        fprintf(stderr, "error: failed to encode: %d\n", ret);
        return 4;
    }

    // 初始化 tokens 数组
    whisper_token tokens[512];
    memset(tokens, 0, sizeof(tokens));

    // 提示热度
    if (int ret = whisper_decode(ctx, tokens, 256, 0, params.n_threads) != 0) {
        fprintf(stderr, "error: failed to decode: %d\n", ret);
        return 4;
    }

    // 文本生成热度
    if (int ret = whisper_decode(ctx, tokens, 1, 256, params.n_threads) != 0) {
        fprintf(stderr, "error: failed to decode: %d\n", ret);
        return 4;
    }

    // 重置计时
    whisper_reset_timings(ctx);

    // 实际运行
    if (int ret = whisper_encode(ctx, 0, params.n_threads) != 0) {
        fprintf(stderr, "error: failed to encode: %d\n", ret);
        return 4;
    }

    // 文本生成
    for (int i = 0; i < 256; i++) {
        if (int ret = whisper_decode(ctx, tokens, 1, i, params.n_threads) != 0) {
            fprintf(stderr, "error: failed to decode: %d\n", ret);
            return 4;
        }
    }

    // 批量解码
    // 对于64个tokens进行whisper解码，每次解码5个tokens，使用0作为参数，使用params.n_threads作为线程数
    for (int i = 0; i < 64; i++) {
        // 调用whisper_decode函数进行解码，如果返回值不为0，则输出错误信息并返回4
        if (int ret = whisper_decode(ctx, tokens, 5, 0, params.n_threads) != 0) {
            fprintf(stderr, "error: failed to decode: %d\n", ret);
            return 4;
        }
    }

    // 处理提示信息
    for (int i = 0; i < 16; i++) {
        // 调用whisper_decode函数进行解码，每次解码256个tokens，使用0作为参数，使用params.n_threads作为线程数
        if (int ret = whisper_decode(ctx, tokens, 256, 0, params.n_threads) != 0) {
            // 如果返回值不为0，则输出错误信息并返回4
            fprintf(stderr, "error: failed to decode: %d\n", ret);
            return 4;
        }
    }

    // 打印whisper的时间信息
    whisper_print_timings(ctx);
    // 释放whisper的上下文
    whisper_free(ctx);

    // 输出空行
    fprintf(stderr, "\n");
    // 输出提交结果的链接
    fprintf(stderr, "If you wish, you can submit these results here:\n");
    fprintf(stderr, "\n");
    fprintf(stderr, "  https://github.com/ggerganov/whisper.cpp/issues/89\n");
    fprintf(stderr, "\n");
    fprintf(stderr, "Please include the following information:\n");
    fprintf(stderr, "\n");
    // 输出需要包含的信息
    fprintf(stderr, "  - CPU model\n");
    fprintf(stderr, "  - Operating system\n");
    fprintf(stderr, "  - Compiler\n");
    fprintf(stderr, "\n");

    // 返回0表示成功
    return 0;
// 主函数，接受命令行参数并执行相应操作
int main(int argc, char ** argv) {
    // 定义参数结构体
    whisper_params params;

    // 解析命令行参数，如果解析失败则返回1
    if (whisper_params_parse(argc, argv, params) == false) {
        return 1;
    }

    // 初始化返回值为-1
    int ret = -1;

    // 根据参数中的操作类型执行相应的操作
    switch (params.what) {
        case 0: ret = whisper_bench_full(params);                break;
        case 1: ret = whisper_bench_memcpy(params.n_threads);       break;
        case 2: ret = whisper_bench_ggml_mul_mat(params.n_threads); break;
        default: fprintf(stderr, "error: unknown benchmark: %d\n", params.what); break;
    }

    // 返回操作结果
    return ret;
}
```