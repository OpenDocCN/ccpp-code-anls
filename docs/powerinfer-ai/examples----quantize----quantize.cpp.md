# `PowerInfer\examples\quantize\quantize.cpp`

```
// 包含常用头文件和 llama.h 头文件
#include "common.h"
#include "llama.h"

// 包含 C 标准输入输出头文件和字符串处理头文件
#include <cstdio>
#include <cstring>
// 包含 vector 和 string 头文件
#include <vector>
#include <string>

// 定义 quant_option 结构体
struct quant_option {
    std::string name;  // 量化选项名称
    llama_ftype ftype;  // 量化选项类型
    std::string desc;  // 量化选项描述
};

// 定义静态的 quant_option 结构体向量 QUANT_OPTIONS
static const std::vector<struct quant_option> QUANT_OPTIONS = {
    { "Q4_0",   LLAMA_FTYPE_MOSTLY_Q4_0,   " 3.56G, +0.2166 ppl @ LLaMA-v1-7B", },  // 量化选项 Q4_0
    { "Q4_1",   LLAMA_FTYPE_MOSTLY_Q4_1,   " 3.90G, +0.1585 ppl @ LLaMA-v1-7B", },  // 量化选项 Q4_1
    { "Q5_0",   LLAMA_FTYPE_MOSTLY_Q5_0,   " 4.33G, +0.0683 ppl @ LLaMA-v1-7B", },  // 量化选项 Q5_0
    { "Q5_1",   LLAMA_FTYPE_MOSTLY_Q5_1,   " 4.70G, +0.0349 ppl @ LLaMA-v1-7B", },  // 量化选项 Q5_1
    { "Q2_K",   LLAMA_FTYPE_MOSTLY_Q2_K,   " 2.63G, +0.6717 ppl @ LLaMA-v1-7B", },  // 量化选项 Q2_K
};
    { "Q3_K",   LLAMA_FTYPE_MOSTLY_Q3_K_M, "alias for Q3_K_M" }, 
    // 定义一个键为"Q3_K"，值为LLAMA_FTYPE_MOSTLY_Q3_K_M的条目，并添加注释说明
    { "Q3_K_S", LLAMA_FTYPE_MOSTLY_Q3_K_S, " 2.75G, +0.5551 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q3_K_S"，值为LLAMA_FTYPE_MOSTLY_Q3_K_S的条目，并添加注释说明
    { "Q3_K_M", LLAMA_FTYPE_MOSTLY_Q3_K_M, " 3.07G, +0.2496 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q3_K_M"，值为LLAMA_FTYPE_MOSTLY_Q3_K_M的条目，并添加注释说明
    { "Q3_K_L", LLAMA_FTYPE_MOSTLY_Q3_K_L, " 3.35G, +0.1764 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q3_K_L"，值为LLAMA_FTYPE_MOSTLY_Q3_K_L的条目，并添加注释说明
    { "Q4_K",   LLAMA_FTYPE_MOSTLY_Q4_K_M, "alias for Q4_K_M", },
    // 定义一个键为"Q4_K"，值为LLAMA_FTYPE_MOSTLY_Q4_K_M的条目，并添加注释说明
    { "Q4_K_S", LLAMA_FTYPE_MOSTLY_Q4_K_S, " 3.59G, +0.0992 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q4_K_S"，值为LLAMA_FTYPE_MOSTLY_Q4_K_S的条目，并添加注释说明
    { "Q4_K_M", LLAMA_FTYPE_MOSTLY_Q4_K_M, " 3.80G, +0.0532 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q4_K_M"，值为LLAMA_FTYPE_MOSTLY_Q4_K_M的条目，并添加注释说明
    { "Q5_K",   LLAMA_FTYPE_MOSTLY_Q5_K_M, "alias for Q5_K_M", },
    // 定义一个键为"Q5_K"，值为LLAMA_FTYPE_MOSTLY_Q5_K_M的条目，并添加注释说明
    { "Q5_K_S", LLAMA_FTYPE_MOSTLY_Q5_K_S, " 4.33G, +0.0400 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q5_K_S"，值为LLAMA_FTYPE_MOSTLY_Q5_K_S的条目，并添加注释说明
    { "Q5_K_M", LLAMA_FTYPE_MOSTLY_Q5_K_M, " 4.45G, +0.0122 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q5_K_M"，值为LLAMA_FTYPE_MOSTLY_Q5_K_M的条目，并添加注释说明
    { "Q6_K",   LLAMA_FTYPE_MOSTLY_Q6_K,   " 5.15G, -0.0008 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q6_K"，值为LLAMA_FTYPE_MOSTLY_Q6_K的条目，并添加注释说明
    { "Q8_0",   LLAMA_FTYPE_MOSTLY_Q8_0,   " 6.70G, +0.0004 ppl @ LLaMA-v1-7B", },
    // 定义一个键为"Q8_0"，值为LLAMA_FTYPE_MOSTLY_Q8_0的条目，并添加注释说明
    { "F16",    LLAMA_FTYPE_MOSTLY_F16,    "13.00G              @ 7B", },
    // 定义一个键为"F16"，值为LLAMA_FTYPE_MOSTLY_F16的条目，并添加注释说明
    { "F32",    LLAMA_FTYPE_ALL_F32,       "26.00G              @ 7B", },
    // 定义一个键为"F32"，值为LLAMA_FTYPE_ALL_F32的条目，并添加注释说明
    // 注意：确保COPY在F32之后，以避免ftype 0匹配。
    { "COPY",   LLAMA_FTYPE_ALL_F32,       "only copy tensors, no quantizing", },
    // 定义一个键为"COPY"，值为LLAMA_FTYPE_ALL_F32的条目，并添加注释说明
};


static bool try_parse_ftype(const std::string & ftype_str_in, llama_ftype & ftype, std::string & ftype_str_out) {
// 尝试解析给定的ftype_str_in字符串，将解析结果存储在ftype和ftype_str_out中
    // 声明一个字符串变量
    std::string ftype_str;

    // 遍历输入的字符串，将每个字符转换为大写后添加到ftype_str中
    for (auto ch : ftype_str_in) {
        ftype_str.push_back(std::toupper(ch));
    }

    // 遍历QUANT_OPTIONS数组，查找与ftype_str相匹配的项
    for (auto & it : QUANT_OPTIONS) {
        if (it.name == ftype_str) {
            // 如果找到匹配的项，则设置ftype和ftype_str_out，并返回true
            ftype = it.ftype;
            ftype_str_out = it.name;
            return true;
        }
    }

    // 尝试将ftype_str转换为整数
    try {
        int ftype_int = std::stoi(ftype_str);
        // 再次遍历QUANT_OPTIONS数组，查找与ftype_int相匹配的项
        for (auto & it : QUANT_OPTIONS) {
            if (it.ftype == ftype_int) {
                // 如果找到匹配的项，则设置ftype和ftype_str_out，并返回true
                ftype = it.ftype;
                ftype_str_out = it.name;
                return true;
            }
        }
// 捕获任何异常并输出错误信息
    }
    catch (...) {
        // stoi 函数失败
    }
    // 返回 false
    return false;
}

// 用法说明
//  ./quantize [--allow-requantize] [--leave-output-tensor] [--pure] models/llama/ggml-model.gguf [models/llama/ggml-model-quant.gguf] type [nthreads]
//
// 指示该函数不会返回，即程序将在此函数内部终止
[[noreturn]]
static void usage(const char * executable) {
    // 输出程序的使用说明
    printf("usage: %s [--help] [--allow-requantize] [--leave-output-tensor] [--pure] model-f32.gguf [model-quant.gguf] type [nthreads]\n\n", executable);
    printf("  --allow-requantize: 允许对已经量化的张量进行重新量化。警告：这可能会严重降低与从16位或32位量化相比的质量\n");
    printf("  --leave-output-tensor: 将保留 output.weight 未（重新）量化。增加模型大小，但也可能增加质量，特别是在重新量化时\n");
    printf("  --pure: 禁用 k-quant 混合，并将所有张量量化为相同类型\n");
    printf("\n允许的量化类型：\n");
    for (auto & it : QUANT_OPTIONS) {
        // 如果量化类型不是 "COPY"，则输出其名称
        if (it.name != "COPY") {
        // 如果文件类型为数字，则打印文件类型
        printf("  %2d  or  ", it.ftype);
        // 如果文件类型不为数字，则打印空格
        } else {
            printf("          ");
        }
        // 打印文件名和描述
        printf("%-6s : %s\n", it.name.c_str(), it.desc.c_str());
    }
    // 退出程序
    exit(1);
}

// 主函数
int main(int argc, char ** argv) {
    // 如果参数小于3个，调用 usage 函数
    if (argc < 3) {
        usage(argv[0]);
    }

    // 初始化参数
    llama_model_quantize_params params = llama_model_quantize_default_params();

    int arg_idx = 1;

    // 遍历参数，查找以 "--" 开头的参数
    for (; arg_idx < argc && strncmp(argv[arg_idx], "--", 2) == 0; arg_idx++) {
        // 如果参数为 "--leave-output-tensor"，执行相应操作
        if (strcmp(argv[arg_idx], "--leave-output-tensor") == 0) {
        // 设置参数 quantize_output_tensor 为 false
        params.quantize_output_tensor = false;
    } else if (strcmp(argv[arg_idx], "--allow-requantize") == 0) {
        // 如果命令行参数为 "--allow-requantize"，则设置参数 allow_requantize 为 true
        params.allow_requantize = true;
    } else if (strcmp(argv[arg_idx], "--pure") == 0) {
        // 如果命令行参数为 "--pure"，则设置参数 pure 为 true
        params.pure = true;
    } else {
        // 如果命令行参数不匹配任何已知参数，则调用 usage 函数显示用法
        usage(argv[0]);
    }

    // 如果剩余的命令行参数少于 2 个，则调用 usage 函数显示用法
    if (argc - arg_idx < 2) {
        usage(argv[0]);
    }

    // 初始化 llama 后端
    llama_backend_init(false);

    // 解析命令行参数，将第一个参数作为输入文件名
    const std::string fname_inp = argv[arg_idx];
    arg_idx++;
    // 声明一个字符串变量 fname_out
    std::string fname_out;
    // 声明一个字符串变量 ftype_str
    std::string ftype_str;
    // 尝试解析文件类型参数，并将解析结果存储到 ftype 和 ftype_str 中
    if (try_parse_ftype(argv[arg_idx], params.ftype, ftype_str)) {
        // 声明一个字符串变量 fpath
        std::string fpath;
        // 查找文件名中最后一个路径分隔符的位置
        const size_t pos = fname_inp.find_last_of("/\\");
        // 如果找到路径分隔符
        if (pos != std::string::npos) {
            // 截取文件路径
            fpath = fname_inp.substr(0, pos + 1);
        }
        // 导出为 [输入路径]/ggml-model-[文件类型].gguf 的文件名
        fname_out = fpath + "ggml-model-" + ftype_str + ".gguf";
        // 增加参数索引
        arg_idx++;
        // 如果文件类型为 "COPY"，设置参数 only_copy 为 true
        if (ftype_str == "COPY") {
            params.only_copy = true;
        }
    }
    // 如果无法解析文件类型参数
    else {
        // 将输出文件名设置为参数中的文件名
        fname_out = argv[arg_idx];
        // 增加参数索引
        arg_idx++;
        // 如果参数数量小于等于参数索引，执行以下操作
        if (argc <= arg_idx) {
// 如果缺少 ftype，则输出错误信息并返回 1
fprintf(stderr, "%s: missing ftype\n", __func__);
return 1;

// 如果无法解析 ftype，则输出错误信息并返回 1
if (!try_parse_ftype(argv[arg_idx], params.ftype, ftype_str)) {
    fprintf(stderr, "%s: invalid ftype '%s'\n", __func__, argv[3]);
    return 1;
}

// 如果 ftype_str 等于 "COPY"，则将 params.only_copy 设置为 true
if (ftype_str == "COPY") {
   params.only_copy = true;
}

// 增加参数索引
arg_idx++;

// 解析 nthreads
if (argc > arg_idx) {
    try {
        // 将参数转换为整数并赋值给 params.nthread
        params.nthread = std::stoi(argv[arg_idx]);
    }
    catch (const std::exception & e) {
        // 如果转换失败，则输出错误信息
        fprintf(stderr, "%s: invalid nthread '%s' (%s)\n", __func__, argv[arg_idx], e.what());
    }
}
    // 返回整数1
    return 1;
    // 如果上面的条件不满足，则执行下面的代码
}

// 打印构建信息
print_build_info();

// 打印量化信息
fprintf(stderr, "%s: quantizing '%s' to '%s' as %s", __func__, fname_inp.c_str(), fname_out.c_str(), ftype_str.c_str());
// 如果线程数大于0，则打印使用的线程数
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
    // 如果量化模型失败，则打印错误信息并返回1
    if (llama_model_quantize(fname_inp.c_str(), fname_out.c_str(), &params)) {
        fprintf(stderr, "%s: failed to quantize model from '%s'\n", __func__, fname_inp.c_str());
        return 1;
    }

    // 计算量化时间
    t_quantize_us = llama_time_us() - t_start_us;
}

// 报告时间
{
    // 获取主函数结束时间
    const int64_t t_main_end_us = llama_time_us();

    // 打印量化时间
    printf("\n");
    printf("%s: quantize time = %8.2f ms\n", __func__, t_quantize_us/1000.0);
    // 打印总时间
    printf("%s:    total time = %8.2f ms\n", __func__, (t_main_end_us - t_main_start_us)/1000.0);
}

// 释放后端资源
llama_backend_free();

// 返回0表示成功
return 0;
这是一个代码块的结束标记，表示前面的函数或者循环的结束。
```