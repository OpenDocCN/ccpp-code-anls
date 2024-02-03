# `PowerInfer\examples\benchmark\benchmark-matmult.cpp`

```cpp
#include "common.h"  // 包含 common.h 头文件
#include "ggml.h"  // 包含 ggml.h 头文件

#include <locale.h>  // 包含 locale.h 头文件
#include <assert.h>  // 包含 assert.h 头文件
#include <math.h>  // 包含 math.h 头文件
#include <cstring>  // 包含 cstring 头文件
#include <cstdio>  // 包含 cstdio 头文件
#include <cinttypes>  // 包含 cinttypes 头文件
#include <unordered_map>  // 包含 unordered_map 头文件
#include <queue>  // 包含 queue 头文件
#include <string.h>  // 包含 string.h 头文件
#include <cassert>  // 包含 assert 头文件
#include <fstream>  // 包含 fstream 头文件
#include <string>  // 包含 string 头文件
#include <iterator>  // 包含 iterator 头文件
#include <algorithm>  // 包含 algorithm 头文件

#if defined(_MSC_VER)
#pragma warning(disable: 4244 4267) // possible loss of data
#endif

static void ggml_graph_compute_helper(std::vector<uint8_t> & buf, ggml_cgraph * graph, int n_threads) {
    struct ggml_cplan plan = ggml_graph_plan(graph, n_threads);  // 调用 ggml_graph_plan 函数，得到计划

    if (plan.work_size > 0) {  // 如果计划中的工作大小大于 0
        buf.resize(plan.work_size);  // 调整 buf 的大小为工作大小
        plan.work_data = buf.data();  // 将工作数据指针指向 buf 的数据
    }

    ggml_graph_compute(graph, &plan);  // 调用 ggml_graph_compute 函数，计算图形
}

static float tensor_sum_elements(const ggml_tensor * tensor) {  // 计算张量元素的总和
    double sum = 0;  // 初始化总和为 0
    if (tensor->type == GGML_TYPE_F32) {  // 如果张量类型为 GGML_TYPE_F32
        for (int j = 0; j < tensor->ne[1]; j++) {  // 遍历张量的第二维
            for (int k = 0; k < tensor->ne[0]; k++) {  // 遍历张量的第一维
                sum += ((float *) tensor->data)[j*tensor->ne[0] + k];  // 计算总和
            }
        }
    }
    return sum;  // 返回总和
}

static void tensor_dump(const ggml_tensor * tensor, const char * name) {  // 打印张量信息
    printf("%15s: type = %i (%5s) ne = %5" PRIi64 " x %5" PRIi64 " x %5" PRIi64 ", nb = (%5zi, %5zi, %5zi) - ", name,
        tensor->type, ggml_type_name(tensor->type),
        tensor->ne[0], tensor->ne[1], tensor->ne[2], tensor->nb[0], tensor->nb[1], tensor->nb[2]);  // 打印张量信息
    float sum = tensor_sum_elements(tensor);  // 计算张量元素总和
    printf("Sum of tensor %s is %6.2f\n", name, sum);  // 打印张量总和
}

#define TENSOR_DUMP(tensor) tensor_dump(tensor, #tensor)  // 定义宏，用于打印张量信息

struct benchmark_params_struct {  // 定义基准参数结构体
    int32_t n_threads     = 1;  // 线程数默认为 1
    int32_t n_iterations  = 10;  // 迭代次数默认为 10
};

static void print_usage(int /*argc*/, char ** argv, struct benchmark_params_struct params) {  // 打印用法
    fprintf(stderr, "usage: %s [options]\n", argv[0]);  // 打印用法信息
    fprintf(stderr, "\n");  // 打印空行
    fprintf(stderr, "options:\n");  // 打印选项信息
    fprintf(stderr, "  -h, --help            show this help message and exit\n");  // 打印帮助信息
    # 输出到标准错误流，显示线程数的帮助信息，使用默认值作为默认参数
    fprintf(stderr, "  -t N, --threads N     number of threads to use during computation (default: %d)\n", params.n_threads);
    # 输出到标准错误流，显示迭代次数的帮助信息，使用默认值作为默认参数
    fprintf(stderr, "  -i N, --iter N     number of iterations to use during computation (default: %d)\n", params.n_iterations);
    # 输出到标准错误流，显示空行
    fprintf(stderr, "\n");
}

int main(int argc, char ** argv)  {
    // 定义结构体 benchmark_params_struct 的实例 benchmark_params
    struct benchmark_params_struct benchmark_params;

    // 初始化参数无效标志为 false
    bool invalid_param = false;
    // 定义字符串 arg
    std::string arg;
    // 遍历命令行参数
    for (int i = 1; i < argc; i++) {
        // 获取当前参数值
        arg = argv[i];

        // 判断参数是否为线程数选项
        if (arg == "-t" || arg == "--threads") {
            // 如果下一个参数不存在，则标记参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数并赋值给 benchmark_params 的 n_threads
            benchmark_params.n_threads = std::stoi(argv[i]);
        } else if (arg == "-i" || arg == "--iter") {
            // 如果下一个参数不存在，则标记参数无效并跳出循环
            if (++i >= argc) {
                invalid_param = true;
                break;
            }
            // 将下一个参数转换为整数并赋值给 benchmark_params 的 n_iterations
            benchmark_params.n_iterations = std::stoi(argv[i]);
        }  else if (arg == "-h" || arg == "--help") {
            // 打印用法信息并退出程序
            print_usage(argc, argv, benchmark_params);
            exit(0);
        }
    }
    // 如果存在无效参数，则打印错误信息并打印用法信息后退出程序
    if (invalid_param) {
        fprintf(stderr, "error: invalid parameter for argument: %s\n", arg.c_str());
        print_usage(argc, argv, benchmark_params);
        exit(1);
    }

    // 打印构建信息
    print_build_info();
    printf("Starting Test\n");

    // 创建 ggml 上下文
    struct ggml_context * ctx;
    // 定义 sizex 和 sizey 的值
    //const int sizex = 4096;
    //const int sizey = 11008;

    // 取消 VERBOSE_DEBUGGING 宏定义并定义 sizex, sizey, sizez 的值
#undef VERBOSE_DEBUGGING
#ifndef VERBOSE_DEBUGGING
    const int sizey = 4096;
    const int sizex = 11008;
    const int sizez = 128;
#else
    /* Working - let's increase size */
    const int sizey = 1;
    const int sizex = (8*32);
    const int sizez = 1;

    /*const int sizey = 1;
    const int sizex = 3*(8*32);
    const int sizez = 1;*/
#endif

    //printf("Memsize required = %i\n", sizex*sizex);

    // TODO: perform the bench for all types or for a user specified type
    // 定义 ggml_type 类型为 GGML_TYPE_Q4_1
    const ggml_type qtype = GGML_TYPE_Q4_1;

    // 计算上下文所需的内存大小
    size_t ctx_size = 0;
    ctx_size += sizex*sizey*ggml_type_sizef(GGML_TYPE_F32);
    ctx_size += sizex*sizey*ggml_type_sizef(GGML_TYPE_F32);
    ctx_size += sizex*sizez*ggml_type_sizef(GGML_TYPE_F32);
    ctx_size += sizex*sizey*ggml_type_sizef(qtype);
    ctx_size += sizex*sizey*ggml_type_sizef(qtype);
    // 计算上下文大小，包括 BLAS 计算所需的内存
    ctx_size += sizex*sizey*ggml_type_sizef(GGML_TYPE_F32); // BLAS
    ctx_size += sizex*sizey*ggml_type_sizef(GGML_TYPE_F32); // BLAS
    // 增加16MB的额外内存
    ctx_size += 1024*1024*16;

    // 打印分配的内存大小
    printf("Allocating Memory of size %zi bytes, %zi MB\n",ctx_size, (ctx_size/1024/1024));

    // 初始化参数结构体
    struct ggml_init_params params = {
        /*.mem_size   =*/ ctx_size,
        /*.mem_buffer =*/ NULL,
        /* no_alloc   =*/ 0
    };

    // 初始化上下文
    ctx = ggml_init(params);
    if (!ctx) {
        fprintf(stderr, "%s: ggml_init() failed\n", __func__);
        return 1;
    }

    // 创建新的张量
    printf("Creating new tensors\n");
    // 创建并初始化张量 m11
    struct ggml_tensor * m11 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, sizex, sizey);
    ggml_set_f32(m11, 1.0f);

    // 创建并初始化张量 m12
    struct ggml_tensor * m12 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, sizex, sizey);
    ggml_set_f32(m12, 1.5f);

    // 创建并初始化张量 m2
    struct ggml_tensor * m2 = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, sizex, sizez);
    ggml_set_f32(m2, 2.0f);

    // 打印测试信息
    printf("\n------ Test 1 - Matrix Mult via F32 code\n");
    // 创建并初始化张量 m11xm2
    struct ggml_tensor * m11xm2 = ggml_mul_mat(ctx, m11, m2);

    // 创建计算图
    struct ggml_cgraph * gf = ggml_new_graph(ctx);
    ggml_build_forward_expand(gf, m11xm2);

    // 打印线程数
    printf("n_threads=%i\n", benchmark_params.n_threads);

    // 打印张量 m11 和 m2 的信息
    TENSOR_DUMP(m11);
    TENSOR_DUMP(m2);

    // 创建工作缓冲区
    std::vector<uint8_t> work_buffer;

    // 运行计算图
    ggml_graph_compute_helper(work_buffer, gf, benchmark_params.n_threads);

    // 打印计算结果
    TENSOR_DUMP(gf->nodes[0]);

    // 打印测试信息
    printf("\n------ Test 2 - Matrix Mult via %s code\n", ggml_type_name(qtype));

    // 计算元素数量
    int32_t nelements = sizex*sizey;

    // 创建直方图
    std::vector<int64_t> hist_cur(1 << 4, 0);

    // 创建并初始化张量 q11
    struct ggml_tensor * q11 = ggml_new_tensor_2d(ctx, qtype, sizex, sizey);
    # 对输入数据进行量化处理，将结果存储在 q11 中
    ggml_quantize_chunk(qtype, (const float *) m11->data, q11->data, 0, nelements, hist_cur.data());

    # 创建一个新的张量 q31，通过将 q11 与 m2 进行矩阵乘法得到
    struct ggml_tensor * q31 = ggml_mul_mat(ctx, q11, m2);

    # 创建一个新的计算图 gf31
    struct ggml_cgraph * gf31 = ggml_new_graph(ctx);
    # 在计算图中构建前向展开
    ggml_build_forward_expand(gf31, q31);

    # 创建一个新的张量 q12，通过将 m12 与 m2 进行量化处理得到
    struct ggml_tensor * q12 = ggml_new_tensor_2d(ctx, qtype, sizex, sizey);
    ggml_quantize_chunk(qtype, (const float *) m12->data, q12->data, 0, nelements, hist_cur.data());

    # 创建一个新的张量 q32，通过将 q12 与 m2 进行矩阵乘法得到
    struct ggml_tensor * q32 = ggml_mul_mat(ctx, q12, m2);

    # 打印 benchmark_params.n_threads 的值
    printf("n_threads=%i\n", benchmark_params.n_threads);

    # 定义并初始化 dimx、dimy、dimz 三个变量
    const int dimx = sizex;
    const int dimy = sizey;
    const int dimz = sizez;
    # 计算每个点乘操作的浮点运算次数
    long long int flops_per_dot_product = dimy + dimy;
    # 计算每个矩阵乘法的浮点运算次数
    long long int flops_per_matrix = flops_per_dot_product * dimx * dimz; ;
    # 打印矩阵乘法的浮点运算次数，以及对应的 gFLOPS 值
    printf("Matrix Multiplication of (%i,%i,%i) x (%i,%i,%i) - about %6.2f gFLOPS\n\n", sizex, sizey, 1, sizex, sizez, 1, 1.0f*flops_per_matrix / 1000 / 1000 / 1000);

    # 计算 gf->nodes[0] 中所有元素的和，作为 F32 结果的参考值
    float sum_of_F32_reference = tensor_sum_elements(gf->nodes[0]);

    # 打印表头
    printf("Iteration;NThreads; SizeX; SizeY; SizeZ; Required_FLOPS; Elapsed_u_Seconds; gigaFLOPS\n");
    printf("=====================================================================================\n");

    # 初始化 gflops_sum 变量
    double  gflops_sum = 0;
    // 循环执行 benchmark_params.n_iterations 次
    for (int i=0;i<benchmark_params.n_iterations ;i++) {

        // 获取当前时间作为起始时间
        long long int start = ggml_time_us();
        // 调用 ggml_graph_compute_helper 函数进行图计算
        ggml_graph_compute_helper(work_buffer, gf31, benchmark_params.n_threads);

        // 获取当前时间作为结束时间
        long long int stop = ggml_time_us();
        // 计算执行时间
        long long int usec = stop-start;
        // 计算每秒浮点运算次数
        double gflops = (double)(flops_per_matrix)/usec/1000.0;
        // 将每次计算的每秒浮点运算次数累加到总和中
        gflops_sum += gflops;
        // 打印输出当前循环的统计信息
        printf("%9i;%8i;%6i;%6i;%6i;%15lli;%18lli;%10.2f\n",
            i,
            benchmark_params.n_threads,
            sizex, sizey, sizez, flops_per_matrix,
            usec,gflops);
#ifdef VERBOSE_DEBUGGING
        // 如果定义了 VERBOSE_DEBUGGING 宏，则输出节点 gf31.nodes[0] 的内容
        TENSOR_DUMP("res",gf31.nodes[0])
#endif

        // 检查矩阵乘法结果是否在合理范围内
        // 由于量化会略有不同，所以不能使用 F32 乘法的精确值
        float sum_of_Q4_result = tensor_sum_elements(gf31->nodes[0]);
        float delta = std::abs(sum_of_Q4_result - sum_of_F32_reference);
        float allowed_delta = (sum_of_F32_reference) / 1000 / 1000; //  接受 10^-6 的误差范围

        if (delta > allowed_delta)  {
            // 如果误差超出允许范围，则输出错误信息并退出程序
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
    // 输出平均值
    printf("\n");
    printf("Average%78.2f\n",gflops_sum/((double)benchmark_params.n_iterations));
    printf("=====================================================================================\n");
}
```