# `ggml\tests\test-conv2d.cpp`

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

// 默认的日志回调函数，将日志输出到标准错误流
static void ggml_log_callback_default(ggml_log_level level, const char * text, void * user_data) {
    (void) level;
    (void) user_data;
    fputs(text, stderr);
    fflush(stderr);
}

// 定义测试模型结构
struct test_model {
    struct ggml_tensor * a;
    struct ggml_tensor * b;
    ggml_backend_t backend = NULL;
    ggml_backend_buffer_t buffer;
    struct ggml_context * ctx;
};

// 加载模型的函数
void load_model(test_model & model, bool use_gpu = false) {
    // 创建数据
    int KW = 3, KH = 3, IC = 10, OC = 10;
    int IW = 8, IH = 6, N = 1;

    // 初始化 adata
    float * adata = new float[KW * KH * IC * OC];
    for (int i = 0; i < KW * KH * IC * OC; i++) {
        adata[i] = 2.5f;
    }

    // 将 adata 转换为 fp16 格式
    std::vector<ggml_fp16_t> hadata(KW * KH * IC * OC);
    ggml_fp32_to_fp16_row(adata, hadata.data(), KW * KH * IC * OC);

    // 初始化 bdata
    float * bdata =  new float[IW * IH * IC * N];
    for (int i = 0; i < IW * IH * IC * N; i++) {
        bdata[i] = 1.5f;
    }

    // 计算缓冲区大小
    size_t buffer_size = 0;
    {
        buffer_size += KW * KH * IC * OC * ggml_type_size(GGML_TYPE_F16); // tensor a
        buffer_size += IW * IH * IC * N  * ggml_type_size(GGML_TYPE_F32); // tensor b
        buffer_size += 1024; // overhead
    }

    // 打印信息
    printf("%s: ggml tensor size    = %d bytes\n", __func__, (int) sizeof(ggml_tensor));
    printf("%s: backend buffer size = %0.2f MB\n", __func__, (buffer_size/ 1024.f/ 1024.f));

    // 初始化参数结构
    int num_tensors = 2;
    struct ggml_init_params params {
            /*.mem_size   =*/ ggml_tensor_overhead() * num_tensors,
            /*.mem_buffer =*/ NULL,
            /*.no_alloc   =*/ true,
    };
}
    // 初始化后端
#ifdef GGML_USE_CUBLAS
    // 如果使用 GPU，则使用 CUDA 后端，并初始化 CUDA 后端
    if (use_gpu) {
        // 输出信息，表示正在使用 CUDA 后端
        fprintf(stderr, "%s: using CUDA backend\n", __func__);
        // 初始化 CUDA 后端，并将其赋值给模型的后端
        model.backend = ggml_backend_cuda_init(0);
        // 如果初始化失败，则输出错误信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_cuda_init() failed\n", __func__);
        }
    }
#endif

#ifdef GGML_USE_METAL
    // 如果使用 GPU，则使用 Metal 后端，并初始化 Metal 后端
    if (use_gpu) {
        // 输出信息，表示正在使用 Metal 后端
        fprintf(stderr, "%s: using Metal backend\n", __func__);
        // 设置 Metal 后端的日志回调函数
        ggml_metal_log_set_callback(ggml_log_callback_default, nullptr);
        // 初始化 Metal 后端，并将其赋值给模型的后端
        model.backend = ggml_backend_metal_init();
        // 如果初始化失败，则输出错误信息
        if (!model.backend) {
            fprintf(stderr, "%s: ggml_backend_metal_init() failed\n", __func__);
        }
    }
#endif

    // 如果模型的后端为空，则回退到 CPU 后端
    if(!model.backend) {
        // 初始化 CPU 后端，并将其赋值给模型的后端
        model.backend = ggml_backend_cpu_init();
    }

    // 分配模型缓冲区
    model.buffer = ggml_backend_alloc_buffer(model.backend, buffer_size);

    // 创建上下文
    model.ctx = ggml_init(params);

    // 创建张量 a
    model.a = ggml_new_tensor_4d(model.ctx, GGML_TYPE_F16,  KW, KH, IC, OC);
    // 创建张量 b
    model.b = ggml_new_tensor_4d(model.ctx, GGML_TYPE_F32, IW, IH, IC, N);

    // 创建分配器
    ggml_allocr * alloc = ggml_allocr_new_from_buffer(model.buffer);

    // 分配内存给张量 a
    ggml_allocr_alloc(alloc, model.a);

    // 将数据加载到缓冲区
    if(ggml_backend_is_cpu(model.backend)) {
        // 如果使用 CPU 后端，则使用 memcpy 将数据加载到张量 a 的数据指针中
        memcpy(model.a->data, hadata.data(), ggml_nbytes(model.a));
    } else {
        // 否则，使用后端的函数将数据加载到张量 a 的数据指针中
        ggml_backend_tensor_set(model.a, hadata.data(), 0, ggml_nbytes(model.a));
    }

    // 分配内存给张量 b
    ggml_allocr_alloc(alloc, model.b);

    // 根据后端类型加载数据到张量 b
    if(ggml_backend_is_cpu(model.backend)
#ifdef GGML_USE_METAL
                || ggml_backend_is_metal(model.backend)
#endif
    ) {
        // 如果使用 CPU 或 Metal 后端，则使用 memcpy 将数据加载到张量 b 的数据指针中
        memcpy(model.b->data, bdata, ggml_nbytes(model.b));
    } else {
        // 否则，使用后端的函数将数据加载到张量 b 的数据指针中
        ggml_backend_tensor_set(model.b, bdata, 0, ggml_nbytes(model.b));
    }

    // 释放分配器
    ggml_allocr_free(alloc);
}

// 构建计算图
struct ggml_cgraph * build_graph(const test_model& model, struct ggml_allocr * allocr) {
    // 定义一个静态变量，表示缓冲区的大小，用于存储张量和图的开销
    static size_t buf_size = ggml_tensor_overhead()*GGML_DEFAULT_GRAPH_SIZE + ggml_graph_overhead();
    // 创建一个静态向量，用于存储缓冲区的数据
    static std::vector<uint8_t> buf(buf_size);

    // 初始化参数结构体，设置内存大小、内存缓冲区和是否允许分配内存
    struct ggml_init_params params0 = {
        /*.mem_size   =*/ buf_size,
        /*.mem_buffer =*/ buf.data(),
        /*.no_alloc   =*/ true, // the tensors will be allocated later by ggml_allocr_alloc_graph()
    };

    // 创建一个临时上下文来构建图
    struct ggml_context * ctx0 = ggml_init(params0);

    // 创建一个新的计算图
    struct ggml_cgraph  * gf = ggml_new_graph(ctx0);

    // 初始化一些变量
    int s0 = 1;
    int s1 = 1;
    int p0 = 1;
    int p1 = 1;
    int d0 = 1;
    int d1 = 1;

    // 将 conv2d 拆分为基本方法进行单元测试
    // 创建一个 im2col 张量，并将其命名为 "im2col_res"，然后将其添加到图中
    struct ggml_tensor* im2col_0 = ggml_im2col(ctx0, model.a, model.b, s0, s1, p0, p1, d0, d1, true);
    ggml_set_name(im2col_0, "im2col_res");
    ggml_build_forward_expand(gf, im2col_0);

    // 重新计算以避免碎片化
    // 创建一个 conv2d_res 张量，并将其命名为 "conv2d_res"，然后将其添加到图中
    struct ggml_tensor* conv2d_res = ggml_conv_2d(ctx0, model.a, model.b, s0, s1, p0, p1, d0, d1);
    ggml_set_name(conv2d_res, "conv2d_res");
    ggml_build_forward_expand(gf, conv2d_res);

    // 释放上下文和相关资源
    ggml_free(ctx0);
    // 返回创建的计算图
    return gf;
}

// 计算图的构建函数，根据给定的模型和分配器构建计算图
struct ggml_cgraph * compute_graph(const test_model & model, struct ggml_allocr * allocr) {
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

    // 执行计算图
    ggml_backend_graph_compute(model.backend, gf);

    // 打印计算图
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

    // 用于计算的缓冲区
    ggml_backend_buffer_t buf_compute;
    struct ggml_allocr * allocr = NULL;

    {
        // 从后端创建一个测量分配器
        allocr = ggml_allocr_new_measure_from_backend(model.backend);

        // 创建最坏情况的计算图，用于估算内存使用
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

    // 初始化结果张量
    struct ggml_tensor * im2col_res = NULL;
    struct ggml_tensor * conv2d_res = NULL;

    // 遍历计算图的节点，找到指定名称的结果张量
    for(int i = 0; i < gf_res->n_nodes; i++) {
        if(strcmp(ggml_get_name(gf_res->nodes[i]), "im2col_res") == 0) {
            im2col_res = gf_res->nodes[i];
        } else if(strcmp(ggml_get_name(gf_res->nodes[i]), "conv2d_res") == 0) {
            conv2d_res = gf_res->nodes[i];
        }
    }
}
    // 为 im2col 数据分配内存空间
    uint16_t* im2col_data = new uint16_t[ggml_nelements(im2col_res)];
    // 为 conv2d 数据分配内存空间
    float* conv2d_data = new float[ggml_nelements(conv2d_res)];

    // 从 im2col_res 中获取数据并存储到 im2col_data 中
    ggml_backend_tensor_get(im2col_res, im2col_data, 0, ggml_nbytes(im2col_res));
    // 从 conv2d_res 中获取数据并存储到 conv2d_data 中
    ggml_backend_tensor_get(conv2d_res, conv2d_data, 0, ggml_nbytes(conv2d_res));

    // 定义测试用例的大小
    const int n_conv2d_test = 480;
    const int n_im2col_test = 4320;

    };

    // 打印测试开始信息
    printf("\nPerforming test:\n");

    // 初始化测试结果为通过
    bool passed = true;
    // 遍历 im2col_data 和 expected_im2col 进行比较
    for(int i = 0; i < n_conv2d_test; i++) {
        if(
            im2col_data[i] != expected_im2col[i]) {
            // 如果不相等，则测试结果为失败
            passed = false;
            break;
        }
    }

    // 打印 im2col 测试结果
    printf("ggml_im2col (%d): %s\n", (int) ggml_nelements(im2col_res), passed && (ggml_nelements(im2col_res) == n_im2col_test) ? "\033[32mPASSED\033[0m" : "\033[31mFAILED\033[0m");

    // 重新初始化测试结果为通过
    passed = true;
    // 遍历 conv2d_data 和 expected_conv2d 进行比较
    for(int i = 0; i < n_conv2d_test; i++) {
        if(conv2d_data[i] != expected_conv2d[i]) {
            // 如果不相等，则测试结果为失败
            passed = false;
            break;
        }
    }

    // 打印 conv2d 测试结果
    printf("ggml_conv2d (%d): %s\n", (int) ggml_nelements(conv2d_res), passed && (ggml_nelements(conv2d_res) == n_conv2d_test) ? "\033[32mPASSED\033[0m" : "\033[31mFAILED\033[0m");

    // 释放模型上下文
    ggml_free(model.ctx);

    // 释放模型缓冲区
    ggml_backend_buffer_free(model.buffer);
    // 释放计算缓冲区
    ggml_backend_buffer_free(buf_compute);
    // 释放模型后端
    ggml_backend_free(model.backend);
    // 返回 0 表示程序正常结束
    return 0;
# 闭合前面的函数定义
```