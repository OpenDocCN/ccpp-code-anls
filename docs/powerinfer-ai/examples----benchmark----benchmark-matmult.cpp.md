# `PowerInfer\examples\benchmark\benchmark-matmult.cpp`

```
// 包含自定义的头文件 common.h 和 ggml.h
#include "common.h"
#include "ggml.h"

// 包含标准库头文件
#include <locale.h>
#include <assert.h>
#include <math.h>
#include <cstring>
#include <cstdio>
#include <cinttypes>
#include <unordered_map>
#include <queue>
#include <string.h>
#include <cassert>
#include <fstream>
#include <string>
#include <iterator>
#include <algorithm>

// 如果编译器是 MSC_VER，则禁用警告 4244 和 4267，这两个警告是可能的数据丢失
#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
// 结束预处理指令
#endif

// 辅助函数，用于计算图形的计算
static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    // 根据图形和线程数创建计划
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);

    // 如果计划中有工作量
    if (plan.work_size > 0) {
        // 调整缓冲区大小
        buf.resize(plan.work_size);
        // 将工作数据指针指向缓冲区
        plan.work_data = buf.data();
    }

    // 执行图形计算
    ggml_graph_compute(graph, &plan);
}

// 计算张量元素之和
static float tensor_sum_elements(const ggml_tensor * tensor) {
    double sum = 0;
    // 如果张量类型为 GGML_TYPE_F32
    if (tensor->type == GGML_TYPE_F32) {
        // 遍历张量元素
        for (int j = 0; j < tensor->ne[1]; j++) {
            for (int k = 0; k < tensor->ne[0]; k++) {
                // 计算元素之和
                sum += ((float *) tensor->data)[j*tensor->ne[0] + k];
            }
// 计算张量中所有元素的总和
static float tensor_sum_elements(const ggml_tensor * tensor) {
    float sum = 0.0;
    // 遍历张量中的所有元素，累加求和
    for (int i = 0; i < tensor->ne[0] * tensor->ne[1] * tensor->ne[2]; i++) {
        sum += tensor->data[i];
    }
    return sum;
}

// 打印张量的信息和元素总和
static void tensor_dump(const ggml_tensor * tensor, const char * name) {
    // 打印张量的名称、类型、维度和元素总和
    printf("%15s: type = %i (%5s) ne = %5" PRIi64 " x %5" PRIi64 " x %5" PRIi64 ", nb = (%5zi, %5zi, %5zi) - ", name,
        tensor->type, ggml_type_name(tensor->type),
        tensor->ne[0], tensor->ne[1], tensor->ne[2], tensor->nb[0], tensor->nb[1], tensor->nb[2]);
    // 计算张量的元素总和
    float sum = tensor_sum_elements(tensor);
    // 打印张量的名称和元素总和
    printf("Sum of tensor %s is %6.2f\n", name, sum);
}

// 宏定义，用于简化调用 tensor_dump 函数
#define TENSOR_DUMP(tensor) tensor_dump(tensor, #tensor)

// 定义结构体 benchmark_params_struct，包含两个整型成员变量
struct benchmark_params_struct {
    int32_t n_threads     = 1;
    int32_t n_iterations  = 10;
};
// 打印程序的使用方法
static void print_usage(int /*argc*/, char ** argv, struct benchmark_params_struct params) {
    // 打印程序的使用方法
    fprintf(stderr, "usage: %s [options]\n", argv[0]);
    // 打印空行
    fprintf(stderr, "\n");
    // 打印选项部分的说明
    fprintf(stderr, "options:\n");
    // 打印帮助选项
    fprintf(stderr, "  -h, --help            show this help message and exit\n");
    // 打印线程数选项
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    // 打印迭代次数选项
    fprintf(stderr, "  -i N, --iter N     number of iterations to use during computation (default: %d)\n", params.n_iterations);
    // 打印空行
    fprintf(stderr, "\n");
}

int main(int argc, char ** argv)  {
    // 定义 benchmark_params_struct 结构体变量
    struct benchmark_params_struct benchmark_params;

    // 定义无效参数标志
    bool invalid_param = false;
    // 定义字符串变量 arg
    std::string arg;
    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数
        arg = argv[i];

        // 判断当前参数是否为线程数选项
        if (arg == "-t" || arg == "--threads") {
            // 判断下一个参数是否存在
            if (++i >= argc) {
        // 如果参数无效，设置标志位为true，并跳出循环
        invalid_param = true;
        break;
    }
    // 如果参数为"-t"或"--threads"
    else if (arg == "-t" || arg == "--threads") {
        // 如果下一个参数不存在，设置标志位为true，并跳出循环
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        // 将参数转换为整数并赋值给benchmark_params.n_threads
        benchmark_params.n_threads = std::stoi(argv[i]);
    } 
    // 如果参数为"-i"或"--iter"
    else if (arg == "-i" || arg == "--iter") {
        // 如果下一个参数不存在，设置标志位为true，并跳出循环
        if (++i >= argc) {
            invalid_param = true;
            break;
        }
        // 将参数转换为整数并赋值给benchmark_params.n_iterations
        benchmark_params.n_iterations = std::stoi(argv[i]);
    }  
    // 如果参数为"-h"或"--help"
    else if (arg == "-h" || arg == "--help") {
        // 打印用法信息并退出程序
        print_usage(argc, argv, benchmark_params);
        exit(0);
    }
}
// 如果存在无效参数
if (invalid_param) {
    // 打印错误信息和用法信息，并退出程序
    fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
    print_usage(argc, argv, benchmark_params);
    exit(1);
}
    // 打印构建信息
    print_build_info();
    // 打印 "Starting Test"
    printf("Starting Test\n");

    // 创建 ggml 上下文
    struct ggml_context * ctx;
    // 定义变量 sizex 和 sizey
    //const int sizex = 4096;
    //const int sizey = 11008;

    // 取消定义 VERBOSE_DEBUGGING
    #undef VERBOSE_DEBUGGING
    // 如果 VERBOSE_DEBUGGING 未定义
    #ifndef VERBOSE_DEBUGGING
        // 定义变量 sizey, sizex, sizez
        const int sizey = 4096;
        const int sizex = 11008;
        const int sizez = 128;
    // 如果 VERBOSE_DEBUGGING 已定义
    #else
        /* Working - let's increase size */
        // 定义变量 sizey, sizex, sizez
        const int sizey = 1;
        const int sizex = (8*32);
        const int sizez = 1;
    // 定义三个常量，分别表示数组的大小
    const int sizey = 1;
    const int sizex = 3*(8*32);
    const int sizez = 1;
#endif

    // 打印所需的内存大小
    //printf("Memsize required = %i\n", sizex*sizex);

    // TODO: 对所有类型或用户指定类型执行基准测试
    // 定义变量 qtype，表示数据类型为 GGML_TYPE_Q4_1
    const ggml_type qtype = GGML_TYPE_Q4_1;

    // 计算上下文所需的内存大小
    size_t ctx_size = 0;
    ctx_size += sizex*sizey*ggml_type_sizef(GGML_TYPE_F32);
    ctx_size += sizex*sizey*ggml_type_sizef(GGML_TYPE_F32);
    ctx_size += sizex*sizez*ggml_type_sizef(GGML_TYPE_F32);
    ctx_size += sizex*sizey*ggml_type_sizef(qtype);
    ctx_size += sizex*sizey*ggml_type_sizef(qtype);
    ctx_size += sizex*sizey*ggml_type_sizef(GGML_TYPE_F32); // BLAS
    ctx_size += sizex*sizey*ggml_type_sizef(GGML_TYPE_F32); // BLAS
    ctx_size += 1024*1024*16;
    # 打印分配内存的大小，以字节和 MB 为单位
    printf("Allocating Memory of size %zi bytes, %zi MB\n",ctx_size, (ctx_size/1024/1024));

    # 初始化结构体 ggml_init_params 的参数
    struct ggml_init_params params = {
        /*.mem_size   =*/ ctx_size,  # 内存大小
        /*.mem_buffer =*/ NULL,       # 内存缓冲区
        /* no_alloc   =*/ 0            # 是否分配内存
    };

    # 初始化 ggml 上下文
    ctx = ggml_init(params);
    # 如果初始化失败，打印错误信息并返回 1
    if (!ctx) {
        fprintf(stderr, "%s: ggml_init() failed\n", __func__);
        return 1;
    }

    # 打印信息，表示正在创建新的张量
    printf("Creating new tensors\n");
    # 使用 ggml_new_tensor_2d 函数创建一个新的二维张量 m11，并初始化为 1.0f
    struct ggml_tensor * m11 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, sizex, sizey);
    ggml_set_f32(m11, 1.0f);
    // 创建一个新的二维浮点型张量 m12
    struct ggml_tensor * m12 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, sizex, sizey);
    // 设置张量 m12 的值为 1.5

    // 创建一个新的二维浮点型张量 m2
    struct ggml_tensor * m2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, sizex, sizez);
    // 设置张量 m2 的值为 2.0

    // 打印测试标题
    printf("\n------ Test 1 - Matrix Mult via F32 code\n");

    // 创建一个新的张量 m11xm2，通过 ggml_mul_mat 函数进行矩阵乘法运算
    struct ggml_tensor * m11xm2 = ggml_mul_mat(ctx, m11, m2);

    // 创建一个新的计算图 gf
    struct ggml_cgraph * gf = ggml_new_graph(ctx);
    // 构建前向传播计算图，将 m11xm2 加入计算图
    ggml_build_forward_expand(gf, m11xm2);

    // 打印线程数
    printf("n_threads=%i\n", benchmark_params.n_threads);

    // 打印张量 m11 和 m2 的值
    TENSOR_DUMP(m11);
    TENSOR_DUMP(m2);
    // 创建一个存储 uint8_t 类型数据的向量
    std::vector<uint8_t> work_buffer;

    // 调用 ggml_graph_compute_helper 函数，传入 work_buffer、gf 和 benchmark_params.n_threads 作为参数
    ggml_graph_compute_helper(work_buffer, gf, benchmark_params.n_threads);

    // 打印 gf->nodes[0] 的内容
    TENSOR_DUMP(gf->nodes[0]);

    // 打印测试信息
    printf("\n------ Test 2 - Matrix Mult via %s code\n", ggml_type_name(qtype));

    // 计算 nelements 的值
    int32_t nelements = sizex*sizey;

    // 创建一个存储 int64_t 类型数据的向量，初始化为 0
    std::vector<int64_t> hist_cur(1 << 4, 0);

    // 创建一个新的二维张量 q11，并运行量化操作
    struct ggml_tensor * q11 = ggml_new_tensor_2d(ctx, qtype, sizex, sizey);
    ggml_quantize_chunk(qtype, (const float *) m11->data, q11->data, 0, nelements, hist_cur.data());

    // 创建一个新的张量 q31
    // printf("Creating new tensor q31\n");
// 使用 ggml_mul_mat 函数计算 q11 和 m2 的矩阵乘法，结果存储在 q31 中
struct ggml_tensor * q31 = ggml_mul_mat(ctx, q11, m2);

// 创建一个新的计算图对象 gf31，并将 q31 添加到计算图中
struct ggml_cgraph * gf31 = ggml_new_graph(ctx);
ggml_build_forward_expand(gf31, q31);

// 创建一个新的二维张量 q12，并对 m12 进行量化，结果存储在 q12 中
struct ggml_tensor * q12 = ggml_new_tensor_2d(ctx, qtype, sizex, sizey);
ggml_quantize_chunk(qtype, (const float *) m12->data, q12->data, 0, nelements, hist_cur.data());

// 使用 ggml_mul_mat 函数计算 q12 和 m2 的矩阵乘法，结果存储在 q32 中
struct ggml_tensor * q32 = ggml_mul_mat(ctx, q12, m2);

// 创建一个新的计算图对象 gf32，并将 q32 添加到计算图中
struct ggml_cgraph * gf32 = ggml_new_graph(ctx);
ggml_build_forward_expand(gf32, q32);

// 打印 benchmark_params.n_threads 的值
printf("n_threads=%i\n", benchmark_params.n_threads);

// 定义一个整型变量 dimx，赋值为 sizex
const int dimx = sizex;
    // 定义维度变量，用于存储矩阵的大小
    const int dimy = sizey;
    const int dimz = sizez;
    // 计算每个点乘操作所需的浮点运算次数
    long long int flops_per_dot_product = dimy + dimy;
    // 计算矩阵乘法所需的浮点运算次数
    long long int flops_per_matrix = flops_per_dot_product * dimx * dimz; ;
    // 打印矩阵乘法的浮点运算次数，以及大致的 gFLOPS（十亿次浮点运算每秒）
    printf("Matrix Multiplication of (%i,%i,%i) x (%i,%i,%i) - about %6.2f gFLOPS\n\n", sizex, sizey, 1, sizex, sizez, 1, 1.0f*flops_per_matrix / 1000 / 1000 / 1000);

    // 使用上面的 F32 结果作为量化乘法的参考
    float sum_of_F32_reference = tensor_sum_elements(gf->nodes[0]);

    // 打印迭代次数、线程数、矩阵大小、所需浮点运算次数、经过的时间和 gigaFLOPS
    printf("Iteration;NThreads; SizeX; SizeY; SizeZ; Required_FLOPS; Elapsed_u_Seconds; gigaFLOPS\n");
    printf("=====================================================================================\n");

    // 初始化 gflops_sum 变量
    double  gflops_sum = 0;
    // 循环执行 benchmark_params.n_iterations 次
    for (int i=0;i<benchmark_params.n_iterations ;i++) {
        // 记录开始时间
        long long int start = ggml_time_us();
        // 运行 ggml_graph_compute_helper 函数
        ggml_graph_compute_helper(work_buffer, gf31, benchmark_params.n_threads);
        // 计算程序运行时间并保存
        long long int stop = ggml_time_us();
        long long int usec = stop-start;
        // 计算每秒浮点运算次数并累加
        double gflops = (double)(flops_per_matrix)/usec/1000.0;
        gflops_sum += gflops;
        // 打印输出结果
        printf("%9i;%8i;%6i;%6i;%6i;%15lli;%18lli;%10.2f\n",
            i,
            benchmark_params.n_threads,
            sizex, sizey, sizez, flops_per_matrix,
            usec,gflops);

#ifdef VERBOSE_DEBUGGING
        // 如果定义了VERBOSE_DEBUGGING，则输出张量的调试信息
        TENSOR_DUMP("res",gf31.nodes[0])
#endif

        // 检查矩阵乘法结果是否在合理范围内
        // 由于量化会导致略微不同的结果，因此无法使用F32乘法的精确值
        float sum_of_Q4_result = tensor_sum_elements(gf31->nodes[0]);
        float delta = std::abs(sum_of_Q4_result - sum_of_F32_reference);
        // 允许的误差范围为F32结果的百万分之一
        float allowed_delta = (sum_of_F32_reference) / 1000 / 1000; //  Let's accept an epsilon of 10^-6
        // 如果计算结果的差值大于允许的差值，打印错误信息并退出程序
        if (delta > allowed_delta)  {
            printf("\nABORT - ERROR in Matrix Multiplication result - expected %6.2f, got %6.2f (delta %6.2f > allowed_delta %6.2f)\n",
                sum_of_F32_reference,
                sum_of_Q4_result,
                delta,
                allowed_delta
            );
            exit(0);
        }

        // 运行不同的图计算以确保覆盖 CPU 缓存行
        ggml_graph_compute_helper(work_buffer, gf32, benchmark_params.n_threads);
    }
    // 打印平均值
    printf("\n");
    printf("Average%78.2f\n",gflops_sum/((double)benchmark_params.n_iterations));
    // 打印分隔线
    printf("=====================================================================================\n");
}

```