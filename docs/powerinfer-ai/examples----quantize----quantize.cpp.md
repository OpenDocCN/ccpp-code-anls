# `PowerInfer\examples\quantize\quantize.cpp`

```cpp
// 包含自定义头文件 common.h 和 llama.h
#include "common.h"
#include "llama.h"

// 包含标准输入输出头文件
#include <cstdio>
// 包含字符串处理头文件
#include <cstring>
// 包含向量容器头文件
#include <vector>
// 包含字符串头文件
#include <string>

// 定义结构体 quant_option
struct quant_option {
    std::string name;  // 量化选项的名称
    llama_ftype ftype; // 量化选项的类型
    std::string desc;  // 量化选项的描述
};

// 定义静态常量向量 QUANT_OPTIONS，并初始化
static const std::vector<struct quant_option> QUANT_OPTIONS = {
    { "Q4_0",   LLAMA_FTYPE_MOSTLY_Q4_0,   " 3.56G, +0.2166 ppl @ LLaMA-v1-7B", },  // 量化选项 Q4_0 的信息
    { "Q4_1",   LLAMA_FTYPE_MOSTLY_Q4_1,   " 3.90G, +0.1585 ppl @ LLaMA-v1-7B", },  // 量化选项 Q4_1 的信息
    { "Q5_0",   LLAMA_FTYPE_MOSTLY_Q5_0,   " 4.33G, +0.0683 ppl @ LLaMA-v1-7B", },  // 量化选项 Q5_0 的信息
    { "Q5_1",   LLAMA_FTYPE_MOSTLY_Q5_1,   " 4.70G, +0.0349 ppl @ LLaMA-v1-7B", },  // 量化选项 Q5_1 的信息
    { "Q2_K",   LLAMA_FTYPE_MOSTLY_Q2_K,   " 2.63G, +0.6717 ppl @ LLaMA-v1-7B", },  // 量化选项 Q2_K 的信息
    { "Q3_K",   LLAMA_FTYPE_MOSTLY_Q3_K_M, "alias for Q3_K_M" },  // 量化选项 Q3_K 的信息
    { "Q3_K_S", LLAMA_FTYPE_MOSTLY_Q3_K_S, " 2.75G, +0.5551 ppl @ LLaMA-v1-7B", },  // 量化选项 Q3_K_S 的信息
    { "Q3_K_M", LLAMA_FTYPE_MOSTLY_Q3_K_M, " 3.07G, +0.2496 ppl @ LLaMA-v1-7B", },  // 量化选项 Q3_K_M 的信息
    { "Q3_K_L", LLAMA_FTYPE_MOSTLY_Q3_K_L, " 3.35G, +0.1764 ppl @ LLaMA-v1-7B", },  // 量化选项 Q3_K_L 的信息
    { "Q4_K",   LLAMA_FTYPE_MOSTLY_Q4_K_M, "alias for Q4_K_M", },  // 量化选项 Q4_K 的信息
    { "Q4_K_S", LLAMA_FTYPE_MOSTLY_Q4_K_S, " 3.59G, +0.0992 ppl @ LLaMA-v1-7B", },  // 量化选项 Q4_K_S 的信息
    { "Q4_K_M", LLAMA_FTYPE_MOSTLY_Q4_K_M, " 3.80G, +0.0532 ppl @ LLaMA-v1-7B", },  // 量化选项 Q4_K_M 的信息
    { "Q5_K",   LLAMA_FTYPE_MOSTLY_Q5_K_M, "alias for Q5_K_M", },  // 量化选项 Q5_K 的信息
    { "Q5_K_S", LLAMA_FTYPE_MOSTLY_Q5_K_S, " 4.33G, +0.0400 ppl @ LLaMA-v1-7B", },  // 量化选项 Q5_K_S 的信息
    { "Q5_K_M", LLAMA_FTYPE_MOSTLY_Q5_K_M, " 4.45G, +0.0122 ppl @ LLaMA-v1-7B", },  // 量化选项 Q5_K_M 的信息
    { "Q6_K",   LLAMA_FTYPE_MOSTLY_Q6_K,   " 5.15G, -0.0008 ppl @ LLaMA-v1-7B", },  // 量化选项 Q6_K 的信息
    { "Q8_0",   LLAMA_FTYPE_MOSTLY_Q8_0,   " 6.70G, +0.0004 ppl @ LLaMA-v1-7B", },  // 量化选项 Q8_0 的信息
    { "F16",    LLAMA_FTYPE_MOSTLY_F16,    "13.00G              @ 7B", },  // 量化选项 F16 的信息
    { "F32",    LLAMA_FTYPE_ALL_F32,       "26.00G              @ 7B", },  // 量化选项 F32 的信息
    // 注意：确保 COPY 在 F32 之后，以避免 ftype 0 匹配
    { "COPY",   LLAMA_FTYPE_ALL_F32,       "only copy tensors, no quantizing", },  // 量化选项 COPY 的信息
};
// 尝试解析文件类型字符串，将解析结果存储在 ftype 和 ftype_str_out 中
static bool try_parse_ftype(const std::string & ftype_str_in, llama_ftype & ftype, std::string & ftype_str_out) {
    std::string ftype_str;

    // 将输入的文件类型字符串转换为大写
    for (auto ch : ftype_str_in) {
        ftype_str.push_back(std::toupper(ch));
    }
    // 遍历已定义的文件类型选项
    for (auto & it : QUANT_OPTIONS) {
        // 如果找到匹配的文件类型字符串，则更新 ftype 和 ftype_str_out，并返回 true
        if (it.name == ftype_str) {
            ftype = it.ftype;
            ftype_str_out = it.name;
            return true;
        }
    }
    // 尝试将文件类型字符串转换为整数
    try {
        int ftype_int = std::stoi(ftype_str);
        // 遍历已定义的文件类型选项
        for (auto & it : QUANT_OPTIONS) {
            // 如果找到匹配的文件类型整数，则更新 ftype 和 ftype_str_out，并返回 true
            if (it.ftype == ftype_int) {
                ftype = it.ftype;
                ftype_str_out = it.name;
                return true;
            }
        }
    }
    // 捕获异常，表示 stoi 转换失败
    catch (...) {
        // stoi failed
    }
    // 未找到匹配的文件类型，返回 false
    return false;
}

// 用法说明函数
[[noreturn]]
static void usage(const char * executable) {
    // 输出程序的用法说明
    printf("usage: %s [--help] [--allow-requantize] [--leave-output-tensor] [--pure] model-f32.gguf [model-quant.gguf] type [nthreads]\n\n", executable);
    printf("  --allow-requantize: Allows requantizing tensors that have already been quantized. Warning: This can severely reduce quality compared to quantizing from 16bit or 32bit\n");
    printf("  --leave-output-tensor: Will leave output.weight un(re)quantized. Increases model size but may also increase quality, especially when requantizing\n");
    printf("  --pure: Disable k-quant mixtures and quantize all tensors to the same type\n");
    printf("\nAllowed quantization types:\n");
    // 遍历已定义的文件类型选项，输出每种类型的描述
    for (auto & it : QUANT_OPTIONS) {
        if (it.name != "COPY") {
            printf("  %2d  or  ", it.ftype);
        } else {
            printf("          ");
        }
        printf("%-6s : %s\n", it.name.c_str(), it.desc.c_str());
    }
    // 退出程序
    exit(1);
}

// 主函数
int main(int argc, char ** argv) {
    // 如果参数少于 3 个，调用 usage 函数输出用法说明
    if (argc < 3) {
        usage(argv[0]);
    }
}
    // 初始化量化参数
    llama_model_quantize_params params = llama_model_quantize_default_params();

    // 设置命令行参数索引
    int arg_idx = 1;

    // 遍历命令行参数
    for (; arg_idx < argc && strncmp(argv[arg_idx], "--", 2) == 0; arg_idx++) {
        // 检查是否需要保留输出张量
        if (strcmp(argv[arg_idx], "--leave-output-tensor") == 0) {
            params.quantize_output_tensor = false;
        } 
        // 检查是否允许重新量化
        else if (strcmp(argv[arg_idx], "--allow-requantize") == 0) {
            params.allow_requantize = true;
        } 
        // 检查是否为纯净模式
        else if (strcmp(argv[arg_idx], "--pure") == 0) {
            params.pure = true;
        } 
        // 如果参数不匹配，则调用 usage 函数
        else {
            usage(argv[0]);
        }
    }

    // 检查参数数量是否正确
    if (argc - arg_idx < 2) {
        usage(argv[0]);
    }

    // 初始化 llama 后端
    llama_backend_init(false);

    // 解析命令行参数
    const std::string fname_inp = argv[arg_idx];
    arg_idx++;
    std::string fname_out;

    std::string ftype_str;
    // 尝试解析文件类型
    if (try_parse_ftype(argv[arg_idx], params.ftype, ftype_str)) {
        std::string fpath;
        const size_t pos = fname_inp.find_last_of("/\\");
        if (pos != std::string::npos) {
            fpath = fname_inp.substr(0, pos + 1);
        }
        // 导出为 [inp path]/ggml-model-[ftype].gguf
        fname_out = fpath + "ggml-model-" + ftype_str + ".gguf";
        arg_idx++;
        // 如果文件类型为 COPY，则设置 only_copy 为 true
        if (ftype_str == "COPY") {
            params.only_copy = true;
        }
    }
    else {
        fname_out = argv[arg_idx];
        arg_idx++;

        // 检查参数数量是否正确
        if (argc <= arg_idx) {
            fprintf(stderr, "%s: missing ftype\n", __func__);
            return 1;
        }
        // 尝试解析文件类型
        if (!try_parse_ftype(argv[arg_idx], params.ftype, ftype_str)) {
            fprintf(stderr, "%s: invalid ftype '%s'\n", __func__, argv[3]);
            return 1;
        }
        // 如果文件类型为 COPY，则设置 only_copy 为 true
        if (ftype_str == "COPY") {
           params.only_copy = true;
        }
        arg_idx++;
    }

    // 解析线程数
    // 检查命令行参数是否大于指定索引
    if (argc > arg_idx) {
        // 尝试将命令行参数转换为整数，并赋值给params.nthread
        try {
            params.nthread = std::stoi(argv[arg_idx]);
        }
        // 捕获异常，输出错误信息并返回1
        catch (const std::exception & e) {
            fprintf(stderr, "%s: invalid nthread '%s' (%s)\n", __func__, argv[arg_idx], e.what());
            return 1;
        }
    }

    // 打印构建信息
    print_build_info();

    // 打印量化信息，包括输入文件名、输出文件名、文件类型
    fprintf(stderr, "%s: quantizing '%s' to '%s' as %s", __func__, fname_inp.c_str(), fname_out.c_str(), ftype_str.c_str());
    // 如果params.nthread大于0，打印使用的线程数
    if (params.nthread > 0) {
        fprintf(stderr, " using %d threads", params.nthread);
    }
    // 打印换行符
    fprintf(stderr, "\n");

    // 记录主程序开始时间
    const int64_t t_main_start_us = llama_time_us();

    // 初始化量化时间
    int64_t t_quantize_us = 0;

    // 加载模型
    {
        // 记录加载模型开始时间
        const int64_t t_start_us = llama_time_us();

        // 调用llama_model_quantize函数进行模型量化
        if (llama_model_quantize(fname_inp.c_str(), fname_out.c_str(), &params)) {
            // 如果量化失败，输出错误信息并返回1
            fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
            return 1;
        }

        // 计算量化时间
        t_quantize_us = llama_time_us() - t_start_us;
    }

    // 报告时间
    {
        // 记录主程序结束时间
        const int64_t t_main_end_us = llama_time_us();

        // 打印量化时间和总时间
        printf("\n");
        printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0);
        printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0);
    }

    // 释放llama后端资源
    llama_backend_free();

    // 返回0
    return 0;
# 闭合前面的函数定义
```