# `ggml\tests\test-mul-mat.cpp`

```
#include "ggml.h"
#include "ggml/ggml-alloc.h"
#include "ggml/ggml-backend.h"
// 包含所需的头文件

//#define GGML_USE_CUBLAS // uncomment this to use cuda backend, make sure build ggml lib with GGML_CUBLAS=ON
// 如果要使用 CUDA 后端，请取消注释此行，并确保使用 GGML_CUBLAS=ON 构建 ggml 库

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif
// 如果定义了 GGML_USE_CUBLAS，则包含 CUDA 相关的头文件

#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif
// 如果定义了 GGML_USE_METAL，则包含 Metal 相关的头文件

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>
// 包含标准库的头文件

static void ggml_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);
    fflush(stderr);
}
// 定义默认的日志回调函数

struct test_model {
    struct ggml_tensor * a;
    struct ggml_tensor * b;
    ggml_backend_t backend = NULL;
    ggml_backend_buffer_t buffer;
    struct ggml_context * ctx;
};
// 定义 test_model 结构体，包含指向 ggml_tensor 结构体的指针、后端类型、后端缓冲区、上下文等

void load_model(test_model & model, float* a, float* b, int M, int N, int K, bool use_gpu = false) {
    size_t buffer_size = 0;
    {
        buffer_size += (M * N) * ggml_type_size(GGML_TYPE_F32); // tensor a
        buffer_size += (N * K) * ggml_type_size(GGML_TYPE_F32); // tensor b
        buffer_size += 1024; // overhead
    }
    // 计算模型加载所需的缓冲区大小

    printf("%s: ggml tensor size    = %d bytes\n", __func__, (int) sizeof(ggml_tensor));
    printf("%s: backend buffer size = %d bytes\n", __func__, (int) buffer_size);
    // 打印 ggml tensor 和后端缓冲区的大小信息

    int num_tensors = 2;
    struct ggml_init_params params {
            /*.mem_size   =*/ ggml_tensor_overhead() * num_tensors,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
    };
    // 初始化 ggml_init_params 结构体

    // initialize the backend
#ifdef GGML_USE_CUBLAS
    if (use_gpu) {
        fprintf(stderr, "%s: using CUDA backend\n", __func__);
        model.backend = ggml_backend_cuda_init(0);
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#endif
// 如果定义了 GGML_USE_CUBLAS 并且 use_gpu 为真，则初始化 CUDA 后端
#ifdef GGML_USE_METAL
    # 如果使用 GPU，则输出使用 Metal 后端的信息
    if (use_gpu) {
        fprintf(stderr, "%s: using Metal backend\n", __func__);
        # 设置 Metal 后端的日志回调函数为默认回调函数
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        # 初始化 Metal 后端，并将返回的指针赋给 model.backend
        model.backend = ggml_backend_metal_init();
        # 如果初始化失败，则输出初始化失败的信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_metal_init() failed\n", __func__);
        }
    }
#endif
// 结束预处理指令

if(!model.backend) {
    // 如果模型的后端为空，则回退到 CPU 后端
    model.backend = ggml_backend_cpu_init();
}

// 为模型分配缓冲区
model.buffer = ggml_backend_alloc_buffer(model.backend, buffer_size);

// 创建上下文
model.ctx = ggml_init(params);

// 创建张量 A
model.a = ggml_new_tensor_2d(model.ctx, GGML_TYPE_F32, K, M);
printf("Matrix A: [%i, %i]\n", K, M);
// 创建张量 B
model.b = ggml_new_tensor_2d(model.ctx, GGML_TYPE_F32, K, N);
printf("Matrix B: [%i, %i]\n", K, N);

// 创建分配器
ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer);

// 分配内存
ggml_allocr_alloc(alloc, model.a);

// 将数据加载到缓冲区
if(ggml_backend_is_cpu(model.backend)
#ifdef GGML_USE_METAL
            || ggml_backend_is_metal(model.backend)
#endif
) {
    memcpy(model.a->data, a, ggml_nbytes(model.a));
} else {
    ggml_backend_tensor_set(model.a, a, 0, ggml_nbytes(model.a)); // CUDA 需要直接将数据复制到设备
}

// 分配内存
ggml_allocr_alloc(alloc, model.b);

if(ggml_backend_is_cpu(model.backend)
#ifdef GGML_USE_METAL
            || ggml_backend_is_metal(model.backend)
#endif
) {
    memcpy(model.b->data, b, ggml_nbytes(model.b));
} else {
    ggml_backend_tensor_set(model.b, b, 0, ggml_nbytes(model.b));  // CUDA 需要直接将数据复制到设备
}

// 释放分配器
ggml_allocr_free(alloc);
}

// 构建图形
struct ggml_cgraph * build_graph(const test_model& model, struct ggml_allocr * allocr) {
    static size_t buf_size = ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead();
    static std::vector<uint8_t> buf(buf_size);

    struct ggml_init_params params0 = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf.data(),
        /*.no_alloc   =*/ true, // 张量将在后续由 ggml_allocr_alloc_graph() 分配
    };

    // 创建一个临时上下文来构建图形
    // 初始化一个 ggml 上下文，并将返回的指针赋给 ctx0
    struct ggml_context * ctx0 = ggml_init(params0);

    // 创建一个新的计算图，并将返回的指针赋给 gf
    struct ggml_cgraph * gf = ggml_new_graph(ctx0);

    // 计算 x @ yT 的结果，并将返回的指针赋给 result
    struct ggml_tensor * result = ggml_mul_mat(ctx0, model.a, ggml_cont(ctx0, model.b));

    // 对 zT 进行转置操作，并将结果传递给 ggml_build_forward_expand 函数
    ggml_build_forward_expand(gf, ggml_cont(ctx0, ggml_transpose(ctx0, result)));

    // 释放临时上下文内存，并删除上下文
    ggml_free(ctx0);
    // 返回构建好的计算图
    return gf;
// 重置分配器以释放上一次推断期间分配的所有内存
ggml_allocr_reset(allocr);

// 构建计算图
struct ggml_cgraph * gf = build_graph(model, allocr);

// 分配张量
ggml_allocr_alloc_graph(allocr, gf);
int n_threads = 1;

// 如果后端是 CPU，则设置线程数
if (ggml_backend_is_cpu(model.backend)) {
    ggml_backend_cpu_set_n_threads(model.backend, n_threads);
}

#ifdef GGML_USE_METAL
// 如果后端是 Metal，则设置回调函数的线程数
if (ggml_backend_is_metal(model.backend)) {
    ggml_backend_metal_set_n_cb(model.backend, n_threads);
}
#endif

// 计算计算图
ggml_backend_graph_compute(model.backend, gf);

// 在这种情况下，输出张量是图中的最后一个张量
return gf->nodes[gf->n_nodes - 1];
}

// 计算两个 float16 数组的点积
static void ggml_vec_dot_f16(const int n, float * s, float * x, float * y) {
    float sumf = 0.0;
    for (int i = 0; i < n; ++i) {
        sumf += x[i] * y[i];
    }
    *s = sumf;
}

// 计算两个 float16 矩阵的乘积，并将结果存储在 float32 矩阵中
static void gemm_f16_out_f32(int m, int n, int k,
                             float * A,
                             float * B,
                             float * C,
                             const int ith, const int nth) {
    // 似乎没有什么区别
    int m0, m1, n0, n1;
    // 每个线程的分块
    if (m > n) {
        n0 = 0;
        n1 = n;

        // 目标中的总分块数
        const int np = m;

        // 每个线程的分块数
        const int dp = (np + nth - 1)/nth;

        // 该线程的分块范围
        m0 = dp*ith;
        m1 = std::min(m0 + dp, np);
    } else {
        m0 = 0;
        m1 = m;

        // 目标中的总分块数
        const int np = n;

        // 每个线程的分块数
        const int dp = (np + nth - 1)/nth;

        // 该线程的分块范围
        n0 = dp*ith;
        n1 = std::min(n0 + dp, np);
    }

    // 尝试块状分块
    int64_t blck_n = 16;
    int64_t blck_m = 16;
    // 循环遍历矩阵块的起始行和列
    for (int j = n0; j < n1; j+=blck_n) {
        for (int i = m0; i < m1; i+=blck_m) {
            // 遍历当前矩阵块的行和列
            // printf("i j k => %d %d %d\n", i, j, K);
            for (int ii = i; ii < i + blck_m && ii < m1; ii++) {
                for (int jj = j; jj < j + blck_n && jj < n1; jj++) {
                    // 调用函数计算矩阵乘法
                    ggml_vec_dot_f16(k,
                                    C + ii*n + jj,
                                    A + ii * k,
                                    B + jj * k);
                }
            }
        }
    }
// 执行 gemm_test 函数，计算矩阵乘法并打印结果
void perform_gemm_test(float* a, float* b, float* expected, int M, int N, int K) {
    // 打印提示信息
    printf("\nPerforming gemm_f16_out_f32 test:\n");

    // 创建一个大小为 M * N 的浮点数数组 gemm_out
    float* gemm_out = new float[M * N];
    // 调用 gemm_f16_out_f32 函数计算矩阵乘法
    gemm_f16_out_f32(M, N, K, a, b, gemm_out, 0, 1);

    // 打印计算结果
    for (int i = 0; i < M; i++) {
        for (int j = 0; j < N; j++) {
            printf("%.1ff,", gemm_out[i * N + j]);
        }
        printf("\n");
    }

    // 检查计算结果是否与期望结果一致
    bool passed = true;

    for(int i = 0; i < M * N; i++) {
        if(gemm_out[i] != expected[i]) {
            passed = false;
            break;
        }
    }

    // 打印测试结果
    printf("gemm_mult (%i): %s\n", (M * N), passed ? "\033[32mPASSED\033[0m" : "\033[31mFAILED\033[0m");
}

// 主函数
int main(void)
{
    // 初始化时间
    ggml_time_init();
    // 定义矩阵的维度
    const int M = 4, N = 16, K = 36;  // a conv2d expected matrix multiplication

    // 矩阵 A (4 X 36)
    float matrixA[M * K] = {
       // 省略了矩阵 A 的数据
    };

    // 矩阵 B (16 X 36)
    // 省略了矩阵 B 的数据

    // 矩阵 C (4 x 16)
    // 省略了矩阵 C 的数据
}
    // 创建一个大小为 M*N 的浮点数数组，存储期望的结果
    float expected_result[M * N] = {
        1224.0f, 1023.0f, 1158.0f,1259.0f,1359.0f,1194.0f,1535.0f,1247.0f,1185.0f,1029.0f,889.0f,1182.0f,955.0f,1179.0f,1147.0f,1048.0f,
        1216.0f, 1087.0f, 1239.0f,1361.0f,1392.0f,1260.0f,1247.0f,1563.0f,1167.0f,1052.0f,942.0f,1214.0f,1045.0f,1134.0f,1264.0f,1126.0f,
        1125.0f, 966.0f, 1079.0f,1333.0f,1287.0f,1101.0f,1185.0f,1167.0f,1368.0f,990.0f,967.0f,1121.0f,971.0f,1086.0f,1130.0f,980.0f,
        999.0f, 902.0f, 1020.0f,1056.0f,1076.0f,929.0f,1029.0f,1052.0f,990.0f,1108.0f,823.0f,989.0f,759.0f,1041.0f,1003.0f,870.0f
    };

    // 初始化一个布尔变量，用于标记测试是否通过
    bool passed = true;

    // 执行 GEMM 测试
    perform_gemm_test(matrixA, matrixB, expected_result, M, N, K);

    // 初始化测试模型
    test_model model;
    load_model(model, matrixA, matrixB, M, N, K, true);

    // 为计算分配内存缓冲区
    ggml_backend_buffer_t buf_compute; // for compute
    struct ggml_allocr * allocr = NULL;

    {
        // 从后端创建内存分配器
        allocr = ggml_allocr_new_measure_from_backend(model.backend);

        // 创建最坏情况下的图形以估算内存使用情况
        struct ggml_cgraph * gf = build_graph(model, allocr);
        size_t mem_size = ggml_allocr_alloc_graph(allocr, gf);
        ggml_allocr_free(allocr);

        // 计算所需内存
        buf_compute = ggml_backend_alloc_buffer(model.backend, mem_size);
        allocr = ggml_allocr_new_from_buffer(buf_compute);
        // 打印计算缓冲区大小
        fprintf(stderr, "%s: compute buffer size: %.4f KB\n", __func__, mem_size/1024.0);
    }

    // 计算结果
    struct ggml_tensor * result = compute(model, allocr);

    // 创建一个浮点数数组，用于存储输出数据
    float* out_data = new float[ggml_nelements(result)];

    // 从计算结果中获取数据
    ggml_backend_tensor_get(result, out_data, 0, ggml_nbytes(result));

    // 打印执行 ggml_mul_mat 测试的结果
    printf("\nPerforming ggml_mul_mat test:\n");

    // 检查测试是否通过
    passed = true;
    for(int i = 0; i < M * N; i++) {
        if(out_data[i] != expected_result[i]) {
            passed = false;
            break;
        }
    }

    // 打印输出数据
    for (int i = 0; i < M; i++) {
        for (int j = 0; j < N; j++) {
            printf("%.1f ", out_data[i * N + j]);
        }
        printf("\n");
    }
    # 打印 ggml_mul_mat 函数的执行结果，包括元素个数和是否通过测试
    printf("ggml_mul_mat (%d): %s\n", (int) ggml_nelements(result), passed && (ggml_nelements(result) == M * N) ? "\033[32mPASSED\033[0m" : "\033[31mFAILED\033[0m");

   // 释放内存
    ggml_free(model.ctx);

    # 释放模型缓冲区
    ggml_backend_buffer_free(model.buffer);
    # 释放计算缓冲区
    ggml_backend_buffer_free(buf_compute);
    # 释放模型后端资源
    ggml_backend_free(model.backend);
    # 返回 0 表示正常结束
    return 0;
# 闭合前面的函数定义
```