# `ggml\tests\test-conv1d.cpp`

```cpp
#include "ggml.h"
#include "ggml/ggml-alloc.h"
#include "ggml/ggml-backend.h"

// #define GGML_USE_CUBLAS

#ifdef GGML_USE_CUBLAS
#include "ggml-cuda.h"
#endif

#ifdef GGML_USE_METAL
#include "ggml-metal.h"
#endif

#include <cassert>
#include <cmath>
#include <cstdio>
#include <cstring>
#include <fstream>
#include <map>
#include <string>
#include <vector>

static void ggml_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);  // 输出日志信息到标准错误流
    fflush(stderr);  // 刷新标准错误流
}

struct test_model {
    struct ggml_tensor * a;  // 定义指向 ggml_tensor 结构体的指针 a
    struct ggml_tensor * b;  // 定义指向 ggml_tensor 结构体的指针 b
    ggml_backend_t backend = NULL;  // 定义 ggml_backend_t 类型的变量 backend，并初始化为 NULL
    ggml_backend_buffer_t buffer;  // 定义 ggml_backend_buffer_t 类型的变量 buffer
    struct ggml_context * ctx;  // 定义指向 ggml_context 结构体的指针 ctx
};

void load_model(test_model & model, bool use_gpu = false) {
    // create data
    int K = 3, IC = 10, OC = 10;  // 定义整型变量 K, IC, OC 并初始化
    int IL = 8, N = 1;  // 定义整型变量 IL, N 并初始化

    // Initialize adata
    float * adata = new float[K * IC * OC];  // 动态分配 K * IC * OC 个 float 类型的内存，并将指针赋值给 adata
    for (int i = 0; i < K * IC * OC; i++) {  // 循环遍历 adata
        adata[i] = 4.5f;  // 将 adata[i] 的值设置为 4.5
    }

    // Convert adata to fp16 format
    std::vector<ggml_fp16_t> hadata(K * IC * OC);  // 创建包含 K * IC * OC 个 ggml_fp16_t 类型元素的向量 hadata
    ggml_fp32_to_fp16_row(adata, hadata.data(), K * IC * OC);  // 将 adata 转换为 fp16 格式，并存储到 hadata 中

    // Initialize bdata
    float * bdata =  new float[IL * IC * N];  // 动态分配 IL * IC * N 个 float 类型的内存，并将指针赋值给 bdata
    for (int i = 0; i < IL * IC * N; i++) {  // 循环遍历 bdata
        bdata[i] = 2.5f;  // 将 bdata[i] 的值设置为 2.5
    }

    size_t buffer_size = 0;  // 定义 size_t 类型的变量 buffer_size 并初始化为 0
    {
        buffer_size += K * IC * OC * ggml_type_size(GGML_TYPE_F16); // tensor a  // 计算 tensor a 的内存大小并加到 buffer_size 上
        buffer_size += IL * IC * N * ggml_type_size(GGML_TYPE_F32); // tensor b  // 计算 tensor b 的内存大小并加到 buffer_size 上
        buffer_size += 1024; // overhead  // 加上额外的 1024 字节
    }

    printf("%s: ggml tensor size    = %d bytes\n", __func__, (int) sizeof(ggml_tensor));  // 输出 ggml_tensor 结构体的大小
    printf("%s: backend buffer size = %0.2f MB\n", __func__, (buffer_size/ 1024.f/ 1024.f));  // 输出 backend buffer 的大小

    int num_tensors = 2;  // 定义整型变量 num_tensors 并初始化为 2
    struct ggml_init_params params {  // 定义 ggml_init_params 结构体变量 params
            /*.mem_size   =*/ ggml_tensor_overhead() * num_tensors,  // 设置 mem_size 字段为 ggml_tensor_overhead() * num_tensors
            /*.mem_buffer =*/ NULL,  // 设置 mem_buffer 字段为 NULL
            /*.no_alloc   =*/ true,  // 设置 no_alloc 字段为 true
    };

    // initialize the backend
#ifdef GGML_USE_CUBLAS
    # 如果使用 GPU
    if (use_gpu) {
        # 打印信息，使用 CUDA 后端
        fprintf(stderr, "%s: using CUDA backend\n", __func__);
        # 初始化 CUDA 后端，将结果赋给 model.backend
        model.backend = ggml_backend_cuda_init(0);
        # 如果初始化失败，打印错误信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#ifdef GGML_USE_METAL
    // 如果使用 GPU，并且定义了 GGML_USE_METAL，则使用 Metal 后端
    if (use_gpu) {
        // 输出信息，使用 Metal 后端
        fprintf(stderr, "%s: using Metal backend\n", __func__);
        // 设置 Metal 后端的日志回调函数
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        // 初始化 Metal 后端
        model.backend = ggml_backend_metal_init();
        // 如果初始化失败，输出错误信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_metal_init() failed\n", __func__);
        }
    }
#endif

    // 如果没有指定后端，则回退到 CPU 后端
    if(!model.backend) {
        model.backend = ggml_backend_cpu_init();
    }

    // 分配缓冲区
    model.buffer = ggml_backend_alloc_buffer(model.backend, buffer_size);

    // 创建上下文
    model.ctx = ggml_init(params);

    // 创建张量
    model.a = ggml_new_tensor_3d(model.ctx, GGML_TYPE_F16,  K, IC, OC);
    model.b = ggml_new_tensor_3d(model.ctx, GGML_TYPE_F32, IL, IC, N);

    // 创建分配器
    ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer);

    // 分配内存
    ggml_allocr_alloc(alloc, model.a);

    // 将数据加载到缓冲区
    if(ggml_backend_is_cpu(model.backend)) {
        // 如果使用 CPU 后端，则使用 memcpy 加载数据
        memcpy(model.a->data, hadata.data(), ggml_nbytes(model.a));
    } else {
        // 否则，使用后端的函数加载数据
        ggml_backend_tensor_set(model.a, hadata.data(), 0, ggml_nbytes(model.a));
    }

    // 分配内存
    ggml_allocr_alloc(alloc, model.b);

    // 根据后端类型加载数据
    if(ggml_backend_is_cpu(model.backend)
#ifdef GGML_USE_METAL
                || ggml_backend_is_metal(model.backend)
#endif
    ) {
        // 如果使用 CPU 或 Metal 后端，则使用 memcpy 加载数据
        memcpy(model.b->data, bdata, ggml_nbytes(model.b));
    } else {
        // 否则，使用后端的函数加载数据
        ggml_backend_tensor_set(model.b, bdata, 0, ggml_nbytes(model.b));
    }

    // 释放分配器
    ggml_allocr_free(alloc);
}

// 构建计算图
struct ggml_cgraph * build_graph(const test_model& model, struct ggml_allocr * allocr) {
    // 计算图缓冲区大小
    static size_t buf_size = ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead();
    // 静态缓冲区
    static std::vector<uint8_t> buf(buf_size);
    // 初始化参数结构体，设置内存大小为buf_size
    struct ggml_init_params params0 = {
        /*.mem_size   =*/ buf_size,
        // 将buf的数据作为内存缓冲区
        /*.mem_buffer =*/ buf.data(),
        // 不进行内存分配，张量将在ggml_allocr_alloc_graph()中分配
        /*.no_alloc   =*/ true, 
    };

    // 创建一个临时上下文来构建图
    struct ggml_context * ctx0 = ggml_init(params0);

    // 创建一个新的计算图
    struct ggml_cgraph  * gf = ggml_new_graph(ctx0);

    int s0 = 1;
    int p0 = 1;
    int d0 = 1;

    // 将conv1d拆分为基本方法进行单元测试
    // 创建im2col_0张量
    struct ggml_tensor* im2col_0 = ggml_im2col(ctx0, model.a, model.b, s0, 0, p0, 0, d0, 0, false);
    ggml_set_name(im2col_0, "im2col_res");
    ggml_build_forward_expand(gf, im2col_0);

    // 创建conv1d_res张量
    struct ggml_tensor* conv1d_res = ggml_conv_1d(ctx0, model.a, model.b, s0, p0, d0);
    ggml_set_name(conv1d_res, "conv1d_res");
    ggml_build_forward_expand(gf, conv1d_res);

    // 删除用于构建图的临时上下文
    ggml_free(ctx0);
    // 返回计算图
    return gf;
// 计算图的构建函数，根据给定的模型和分配器构建计算图
struct ggml_cgraph* compute_graph(const test_model & model, struct ggml_allocr * allocr) {
    // 重置分配器，释放上一次推断过程中分配的所有内存
    ggml_allocr_reset(allocr);

    // 构建计算图
    struct ggml_cgraph * gf = build_graph(model, allocr);

    // 分配张量
    ggml_allocr_alloc_graph(allocr, gf);
    int n_threads = 1;

    // 如果使用的是 CPU 后端，则设置线程数
    if (ggml_backend_is_cpu(model.backend)) {
        ggml_backend_cpu_set_n_threads(model.backend, n_threads);
    }

    // 如果使用的是 Metal 后端，则设置回调函数的线程数
#ifdef GGML_USE_METAL
    if (ggml_backend_is_metal(model.backend)) {
        ggml_backend_metal_set_n_cb(model.backend, n_threads);
    }
#endif

    // 使用后端执行计算图
    ggml_backend_graph_compute(model.backend, gf);

    // 打印计算图信息
    //ggml_graph_print(gf);

    // 返回计算图
    return gf;
}

// 主函数
int main(void)
{
    // 初始化时间
    ggml_time_init();

    // 加载模型
    test_model model;
    load_model(model, true);

    // 用于计算的后端缓冲区
    ggml_backend_buffer_t buf_compute;
    struct ggml_allocr * allocr = NULL;

    {
        // 从后端创建一个测量分配器
        allocr = ggml_allocr_new_measure_from_backend(model.backend);

        // 创建最坏情况下的计算图，用于内存使用估计
        struct ggml_cgraph * gf = build_graph(model, allocr);
        size_t mem_size = ggml_allocr_alloc_graph(allocr, gf);
        ggml_allocr_free(allocr);

        // 计算所需的内存
        buf_compute = ggml_backend_alloc_buffer(model.backend, mem_size);
        allocr = ggml_allocr_new_from_buffer(buf_compute);
        fprintf(stderr, "%s: compute buffer size: %.2f MB\n", __func__, mem_size/1024.0f/1024.0f);
    }

    // 计算图的结果
    struct ggml_cgraph * gf_res = compute_graph(model, allocr);

    // 初始化 im2col_res 和 conv1d_res
    struct ggml_tensor * im2col_res = NULL;
    struct ggml_tensor * conv1d_res = NULL;

    // 遍历计算图的节点，找到对应的结果张量
    for(int i = 0; i < gf_res->n_nodes; i++) {
        if(strcmp(ggml_get_name(gf_res->nodes[i]), "im2col_res") == 0) {
            im2col_res = gf_res->nodes[i];
        } else if(strcmp(ggml_get_name(gf_res->nodes[i]), "conv1d_res") == 0) {
            conv1d_res = gf_res->nodes[i];
        }
    }
}
    // 为 im2col 数据分配内存空间
    uint16_t* im2col_data = new uint16_t[ggml_nelements(im2col_res)];
    // 为 conv2d 数据分配内存空间
    float* conv2d_data = new float[ggml_nelements(conv1d_res)];

    // 从 im2col_res 中获取数据并存储到 im2col_data 中
    ggml_backend_tensor_get(im2col_res, im2col_data, 0, ggml_nbytes(im2col_res));
    // 从 conv1d_res 中获取数据并存储到 conv2d_data 中
    ggml_backend_tensor_get(conv1d_res, conv2d_data, 0, ggml_nbytes(conv1d_res));

    // 定义常量 n_conv1d_test 和 n_im2col_test
    const int n_conv1d_test = 80;
    const int n_im2col_test = 240;

    // 定义期望的 conv1d 数据
    float expected_conv1d[n_conv1d_test] = {
        // 省略部分数据
    };
    // 定义第一个 im2col 测试的期望数据
    uint16_t expected_im2col[n_conv1d_test] = {
        // 省略部分数据
    };

    // 打印测试信息
    printf("\nPerforming test:\n");

    // 初始化测试结果为通过
    bool passed = true;
    # 遍历 n_conv1d_test 次循环
    for(int i = 0; i < n_conv1d_test; i++) {
        # 如果 im2col_data[i] 不等于 expected_im2col[i]
        if(im2col_data[i] != expected_im2col[i]) {
            # 将 passed 置为 false，并跳出循环
            passed = false;
            break;
        }
    }

    # 打印 ggml_im2col 的测试结果
    printf("ggml_im2col (%d): %s\n", (int) ggml_nelements(im2col_res), passed && (ggml_nelements(im2col_res) == n_im2col_test) ? "\033[32mPASSED\033[0m" : "\033[31mFAILED\033[0m");

    # 将 passed 置为 true
    passed = true;
    # 再次遍历 n_conv1d_test 次循环
    for(int i = 0; i < n_conv1d_test; i++) {
        # 如果 conv2d_data[i] 不等于 expected_conv1d[i]
        if(conv2d_data[i] != expected_conv1d[i]) {
            # 将 passed 置为 false，并跳出循环
            passed = false;
            break;
        }
    }

    # 打印 ggml_conv1d 的测试结果
    printf("ggml_conv1d (%d): %s\n", (int) ggml_nelements(conv1d_res), passed && (ggml_nelements(conv1d_res) == n_conv1d_test) ? "\033[32mPASSED\033[0m" : "\033[31mFAILED\033[0m");
    
    # 释放 model.ctx
    ggml_free(model.ctx);
    # 释放 model.buffer
    ggml_backend_buffer_free(model.buffer);
    # 释放 buf_compute
    ggml_backend_buffer_free(buf_compute);
    # 释放 model.backend
    ggml_backend_free(model.backend);
    # 返回 0
    return 0;
# 闭合前面的函数定义
```