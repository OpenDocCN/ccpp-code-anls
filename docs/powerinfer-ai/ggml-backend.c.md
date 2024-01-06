# `PowerInfer\ggml-backend.c`

```
// 包含所需的头文件
#include "ggml-backend-impl.h"
#include "ggml-alloc.h"
#include "ggml-impl.h"

#include <assert.h>
#include <limits.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 定义宏，用于标记未使用的变量
#define UNUSED GGML_UNUSED

// 定义宏，用于返回两个值中的最大值
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 定义后端缓冲区结构的初始化函数
ggml_backend_buffer_t ggml_backend_buffer_init(
        struct ggml_backend                  * backend,
        struct ggml_backend_buffer_i           iface,
# 定义一个函数，用于创建一个包含指定上下文和大小的后端缓冲区
ggml_backend_buffer_t create_buffer(ggml_backend_interface_t iface, ggml_backend_t backend, ggml_backend_buffer_context_t context, size_t size) {
    # 分配内存以存储后端缓冲区的结构体
    ggml_backend_buffer_t buffer = malloc(sizeof(struct ggml_backend_buffer));

    # 确保接口的基本功能不为空
    GGML_ASSERT(iface.get_base != NULL);

    # 将结构体的字段初始化为指定的值
    (*buffer) = (struct ggml_backend_buffer) {
        /* .interface = */ iface,
        /* .backend   = */ backend,
        /* .context   = */ context,
        /* .size      = */ size,
    };

    # 返回创建的后端缓冲区
    return buffer;
}

# 定义一个函数，用于释放后端缓冲区的内存
void free_buffer(ggml_backend_buffer_t buffer) {
    # 如果缓冲区为空，则直接返回，不执行任何操作
    if (buffer == NULL) {
        return;
    }
}
# 如果缓冲区的接口中有释放缓冲区的函数，则调用该函数释放缓冲区
if (buffer->iface.free_buffer != NULL) {
    buffer->iface.free_buffer(buffer);
}
# 释放缓冲区的内存
free(buffer);
}

# 获取缓冲区的对齐方式
size_t ggml_backend_buffer_get_alignment(ggml_backend_buffer_t buffer) {
    return ggml_backend_get_alignment(buffer->backend);
}

# 获取缓冲区的大小
size_t ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer) {
    return buffer->size;
}

# 获取缓冲区的基地址
void * ggml_backend_buffer_get_base(ggml_backend_buffer_t buffer) {
    # 获取缓冲区的基地址
    void * base = buffer->iface.get_base(buffer);
    # 断言缓冲区的基地址不为空
    GGML_ASSERT(base != NULL && "backend buffer base cannot be NULL");
    // 返回缓冲区的基地址
    return base;
}

// 获取分配大小，如果未定义则默认为 ggml_nbytes
size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // 如果存在 get_alloc_size 函数，则调用该函数返回分配大小
    if (buffer->iface.get_alloc_size) {
        return buffer->iface.get_alloc_size(buffer, tensor);
    }
    // 否则返回 ggml_nbytes 函数返回的大小
    return ggml_nbytes(tensor);
}

// 初始化张量，如果未定义则不执行任何操作
void ggml_backend_buffer_init_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // 如果存在 init_tensor 函数，则调用该函数初始化张量
    if (buffer->iface.init_tensor) {
        buffer->iface.init_tensor(buffer, tensor);
    }
}

// 释放张量，如果未定义则不执行任何操作
void ggml_backend_buffer_free_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // free_tensor 是可选的
// 如果缓冲区的接口中有释放张量的函数，则调用该函数释放张量
if (buffer->iface.free_tensor) {
    buffer->iface.free_tensor(buffer, tensor);
}

// 获取张量对应的后端
ggml_backend_t ggml_get_backend(const struct ggml_tensor * tensor) {
    // 如果张量有缓冲区，则返回缓冲区的后端，否则返回 NULL
    return tensor->buffer ? tensor->buffer->backend : NULL;
}

// 获取后端的名称
const char * ggml_backend_name(ggml_backend_t backend) {
    // 如果后端为 NULL，则返回字符串 "NULL"，否则调用后端接口中的获取名称函数
    if (backend == NULL) {
        return "NULL";
    }
    return backend->iface.get_name(backend);
}

// 释放后端
void ggml_backend_free(ggml_backend_t backend) {
    // 如果后端为 NULL，则不执行任何操作
    if (backend == NULL) {
        return;
    }
    // 如果没有分配内存，则直接返回

    backend->iface.free(backend);
}
// 释放后端资源

ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size) {
    return backend->iface.alloc_buffer(backend, size);
}
// 分配后端缓冲区

size_t ggml_backend_get_alignment(ggml_backend_t backend) {
    return backend->iface.get_alignment(backend);
}
// 获取后端对齐方式

void ggml_backend_tensor_set_async(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    ggml_get_backend(tensor)->iface.set_tensor_async(ggml_get_backend(tensor), tensor, data, offset, size);
}
// 设置异步张量数据

void ggml_backend_tensor_get_async(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    ggml_get_backend(tensor)->iface.get_tensor_async(ggml_get_backend(tensor), tensor, data, offset, size);
}
// 获取异步张量数据
// 设置张量数据，异步操作
void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    // 获取张量的后端
    ggml_backend_t backend = ggml_get_backend(tensor);

    // 断言张量数据不为空
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    // 断言后端不为空
    GGML_ASSERT(backend != NULL && "tensor backend not set");

    // 调用后端接口设置张量数据，异步操作
    backend->iface.set_tensor_async(backend, tensor, data, offset, size);
    // 同步后端操作
    backend->iface.synchronize(backend);
}

// 获取张量数据，异步操作
void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 获取张量的后端
    ggml_backend_t backend = ggml_get_backend(tensor);

    // 断言张量数据不为空
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    // 断言后端不为空
    GGML_ASSERT(backend != NULL && "tensor backend not set");

    // 调用后端接口获取张量数据，异步操作
    backend->iface.get_tensor_async(backend, tensor, data, offset, size);
    // 同步后端操作
    backend->iface.synchronize(backend);
}
// 同步后端数据
void ggml_backend_synchronize(ggml_backend_t backend) {
    backend->iface.synchronize(backend);
}

// 创建图计划
ggml_backend_graph_plan_t ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    return backend->iface.graph_plan_create(backend, cgraph);
}

// 释放图计划
void ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    backend->iface.graph_plan_free(backend, plan);
}

// 计算图计划
void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    backend->iface.graph_plan_compute(backend, plan);
}

// 计算图
void ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    backend->iface.graph_compute(backend, cgraph);
}
// 检查后端是否支持给定的操作
bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    // 调用后端接口的 supports_op 方法来检查是否支持给定的操作
    return backend->iface.supports_op(backend, op);
}

// 备份后端

// 检查两个张量是否具有相同的布局
static bool ggml_are_same_layout(const struct ggml_tensor * a, const struct ggml_tensor * b) {
    // 如果张量类型不同，则返回 false
    if (a->type != b->type) {
        return false;
    }
    // 遍历张量的维度
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        // 如果张量 a 和张量 b 在第 i 维的元素数量不同，则返回 false
        if (a->ne[i] != b->ne[i]) {
            return false;
        }
        // 如果张量 a 和张量 b 在第 i 维的边界数量不同，则返回 false
        if (a->nb[i] != b->nb[i]) {
            return false;
        }
    }
}
// 返回 true
    return true;
}

// 从源张量复制数据到目标张量
void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 检查源张量和目标张量是否具有相同的布局，如果不同则报错
    GGML_ASSERT(ggml_are_same_layout(src, dst) && "cannot copy tensors with different layouts");

    // 如果源张量和目标张量相同，则不进行复制操作，直接返回
    if (src == dst) {
        return;
    }

    // TODO: 允许后端支持相同后端的复制操作

    // 如果目标张量的后端支持从源张量复制数据，则调用该函数进行复制
    if (ggml_get_backend(dst)->iface.cpy_tensor_from != NULL) {
        ggml_get_backend(dst)->iface.cpy_tensor_from(ggml_get_backend(dst)->context, src, dst);
    } 
    // 如果源张量的后端支持向目标张量复制数据，则调用该函数进行复制
    else if (ggml_get_backend(src)->iface.cpy_tensor_to != NULL) {
        ggml_get_backend(src)->iface.cpy_tensor_to(ggml_get_backend(src)->context, src, dst);
    } 
    // 否则，当从/向 CPU 复制数据时，不应该执行到这里
    else {
        // shouldn't be hit when copying from/to CPU
// 如果处于调试模式，则执行以下代码
#ifndef NDEBUG
#endif

// 获取源张量的字节数
size_t nbytes = ggml_nbytes(src);

// 分配内存空间，大小为源张量的字节数
void * data = malloc(nbytes);

// 从源张量中获取数据，存储到分配的内存空间中
ggml_backend_tensor_get(src, data, 0, nbytes);

// 将数据设置到目标张量中
ggml_backend_tensor_set(dst, data, 0, nbytes);

// 释放分配的内存空间
free(data);
}

// CPU后端

// CPU后端的上下文结构体
struct ggml_backend_cpu_context {
    int n_threads; // 线程数
    void * work_data; // 工作数据
    size_t work_size; // 工作数据大小
};

// 返回CPU后端的名称
static const char * ggml_backend_cpu_name(ggml_backend_t backend) {
    return "CPU";
}
// 释放 CPU 后端资源
static void ggml_backend_cpu_free(ggml_backend_t backend) {
    // 获取 CPU 上下文
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;
    // 释放工作数据
    free(cpu_ctx->work_data);
    // 释放 CPU 上下文
    free(cpu_ctx);
    // 释放后端资源
    free(backend);
}

// 获取 CPU 缓冲区基地址
static void * ggml_backend_cpu_buffer_get_base(ggml_backend_buffer_t buffer) {
    return (void *)buffer->context;
}

// 释放 CPU 缓冲区
static void ggml_backend_cpu_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    // 释放缓冲区上下文
    free(buffer->context);
    UNUSED(buffer);
}
// 定义一个静态的结构体变量，包含了一系列函数指针，用于处理 CPU 后端的缓冲区操作
static struct ggml_backend_buffer_i cpu_backend_buffer_i = {
    /* .free_buffer    = */ ggml_backend_cpu_buffer_free_buffer, // 释放缓冲区的函数指针
    /* .get_base       = */ ggml_backend_cpu_buffer_get_base, // 获取缓冲区基地址的函数指针
    /* .get_alloc_size = */ NULL, // 默认使用 ggml_nbytes 函数来获取分配大小
    /* .init_tensor    = */ NULL, // 不需要初始化
    /* .free_tensor    = */ NULL, // 不需要清理
};

// 对于从指针获取的缓冲区，不需要调用 free
static struct ggml_backend_buffer_i cpu_backend_buffer_i_from_ptr = {
    /* .free_buffer    = */ NULL, // 指针不是缓冲区的所有者，因此不需要释放
    /* .get_base       = */ ggml_backend_cpu_buffer_get_base, // 获取缓冲区基地址的函数指针
    /* .get_alloc_size = */ NULL, // 默认使用 ggml_nbytes 函数来获取分配大小
    /* .init_tensor    = */ NULL, // 不需要初始化
    /* .free_tensor    = */ NULL, // 不需要清理
};

// 定义一个常量，表示张量的对齐方式为 64
static const size_t TENSOR_ALIGNMENT = 64; // 应该足够支持 AVX 512

// 定义一个函数，用于在 CPU 后端分配缓冲区
static ggml_backend_buffer_t ggml_backend_cpu_alloc_buffer(ggml_backend_t backend, size_t size) {
    size += TENSOR_ALIGNMENT;   // 增加 size 的值，以确保分配的内存地址是对齐的
    void * data = malloc(size); // 使用 malloc 分配内存空间，存储地址赋给 data 变量
    GGML_ASSERT(data != NULL && "failed to allocate buffer"); // 断言，确保内存分配成功

    return ggml_backend_buffer_init(backend, cpu_backend_buffer_i, data, size); // 返回初始化后的缓冲区

}

static size_t ggml_backend_cpu_get_alignment(ggml_backend_t backend) {
    return TENSOR_ALIGNMENT; // 返回张量的对齐值
    UNUSED(backend); // 未使用的参数

}

static void ggml_backend_cpu_set_tensor_async(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor write out of bounds"); // 断言，确保写入的数据不超出张量的范围
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated"); // 断言，确保张量已经分配内存

    memcpy((char *)tensor->data + offset, data, size); // 将数据从 data 复制到张量的指定位置

    UNUSED(backend); // 未使用的参数
// 异步从 CPU 后端获取张量数据
static void ggml_backend_cpu_get_tensor_async(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 确保读取的数据不超出张量的范围
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor read out of bounds");
    // 确保张量已经分配内存
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");

    // 从张量数据中复制指定大小的数据到目标地址
    memcpy(data, (const char *)tensor->data + offset, size);

    // 标记参数未使用
    UNUSED(backend);
}

// 同步 CPU 后端
static void ggml_backend_cpu_synchronize(ggml_backend_t backend) {
    // 标记参数未使用
    UNUSED(backend);
}

// 从源张量复制数据到目标张量
static void ggml_backend_cpu_cpy_tensor_from(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 从后端获取源张量的数据，并复制到目标张量的数据中
    ggml_backend_tensor_get(src, dst->data, 0, ggml_nbytes(src));

    // 标记参数未使用
    UNUSED(backend);
}
// 将源张量的数据复制到目标张量中
static void ggml_backend_cpu_cpy_tensor_to(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 设置目标张量的数据为源张量的数据
    ggml_backend_tensor_set(dst, src->data, 0, ggml_nbytes(src));

    // 未使用的参数，可以忽略
    UNUSED(backend);
}

// CPU 后端计划结构
struct ggml_backend_plan_cpu {
    struct ggml_cplan cplan; // 计划
    struct ggml_cgraph cgraph; // 图
};

// 创建 CPU 后端图计划
static ggml_backend_graph_plan_t ggml_backend_cpu_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 获取 CPU 上下文
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    // 分配内存以存储 CPU 计划
    struct ggml_backend_plan_cpu * cpu_plan = malloc(sizeof(struct ggml_backend_plan_cpu));

    // 创建 CPU 计划
    cpu_plan->cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);
    cpu_plan->cgraph = *cgraph; // 复制图

# 如果 CPU 计划的工作大小大于 0，则分配内存给工作数据
if (cpu_plan->cplan.work_size > 0) {
    cpu_plan->cplan.work_data = malloc(cpu_plan->cplan.work_size);
}

# 释放 CPU 计划的工作数据和 CPU 计划本身的内存
return cpu_plan;
}

# 释放 CPU 图计划的内存
static void ggml_backend_cpu_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 将图计划转换为 CPU 图计划
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    # 释放 CPU 计划的工作数据和 CPU 计划本身的内存
    free(cpu_plan->cplan.work_data);
    free(cpu_plan);

    # 标记参数未使用
    UNUSED(backend);
}

# 计算 CPU 图计划
static void ggml_backend_cpu_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 将图计划转换为 CPU 图计划
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    # 调用函数计算 CPU 图计划
    ggml_graph_compute(&cpu_plan->cgraph, &cpu_plan->cplan);
// 未使用参数backend
UNUSED(backend);
}

// 计算 CPU 图形的计算
static void ggml_backend_cpu_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 将 backend 转换为 CPU 上下文
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    // 计划 CPU 上下文的计划
    struct ggml_cplan cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);

    // 如果工作大小小于计划的工作大小，则重新分配内存
    if (cpu_ctx->work_size < cplan.work_size) {
        // TODO: 可能更快的方法是释放内存并使用 malloc 来避免复制
        cpu_ctx->work_data = realloc(cpu_ctx->work_data, cplan.work_size);
        cpu_ctx->work_size = cplan.work_size;
    }

    // 将计划的工作数据设置为 CPU 上下文的工作数据
    cplan.work_data = cpu_ctx->work_data;

    // 计算图形的计划
    ggml_graph_compute(cgraph, &cplan);
}
// 检查 CPU 后端是否支持给定的操作
static bool ggml_backend_cpu_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    return true;
    // 忽略参数，始终返回 true
    UNUSED(backend);
    UNUSED(op);
}

// CPU 后端接口
static struct ggml_backend_i cpu_backend_i = {
    /* .get_name            = */ ggml_backend_cpu_name, // 获取 CPU 后端名称的函数指针
    /* .free                = */ ggml_backend_cpu_free, // 释放 CPU 后端资源的函数指针
    /* .alloc_buffer        = */ ggml_backend_cpu_alloc_buffer, // 分配 CPU 后端缓冲区的函数指针
    /* .get_alignment       = */ ggml_backend_cpu_get_alignment, // 获取 CPU 后端对齐方式的函数指针
    /* .set_tensor_async    = */ ggml_backend_cpu_set_tensor_async, // 异步设置 CPU 后端张量的函数指针
    /* .get_tensor_async    = */ ggml_backend_cpu_get_tensor_async, // 异步获取 CPU 后端张量的函数指针
    /* .synchronize         = */ ggml_backend_cpu_synchronize, // 同步 CPU 后端的函数指针
    /* .cpy_tensor_from     = */ ggml_backend_cpu_cpy_tensor_from, // 从其他后端复制张量到 CPU 后端的函数指针
    /* .cpy_tensor_to       = */ ggml_backend_cpu_cpy_tensor_to, // 从 CPU 后端复制张量到其他后端的函数指针
    /* .graph_plan_create   = */ ggml_backend_cpu_graph_plan_create, // 创建 CPU 后端图计划的函数指针
    /* .graph_plan_free     = */ ggml_backend_cpu_graph_plan_free, // 释放 CPU 后端图计划的函数指针
    /* .graph_plan_compute  = */ ggml_backend_cpu_graph_plan_compute, // 执行 CPU 后端图计划的函数指针
    /* .graph_compute       = */ ggml_backend_cpu_graph_compute, // 执行 CPU 后端图的函数指针
/* .supports_op         = */ ggml_backend_cpu_supports_op,
};  // 结束了一个结构体的初始化

ggml_backend_t ggml_backend_cpu_init(void) {
    // 为 CPU 上下文分配内存
    struct ggml_backend_cpu_context * ctx = malloc(sizeof(struct ggml_backend_cpu_context));

    // 初始化 CPU 上下文的线程数、工作数据和工作大小
    ctx->n_threads = GGML_DEFAULT_N_THREADS;
    ctx->work_data = NULL;
    ctx->work_size = 0;

    // 为 CPU 后端分配内存
    ggml_backend_t cpu_backend = malloc(sizeof(struct ggml_backend));

    // 初始化 CPU 后端的接口和上下文
    *cpu_backend = (struct ggml_backend) {
        /* .interface = */ cpu_backend_i,  // 设置接口
        /* .context   = */ ctx  // 设置上下文
    };
    return cpu_backend;  // 返回 CPU 后端
}

// 检查给定的后端是否为 CPU 后端
bool ggml_backend_is_cpu(ggml_backend_t backend) {
// 检查后端接口的名称是否与给定的名称相匹配
return backend->iface.get_name == ggml_backend_cpu_name;

// 设置 CPU 后端的线程数
void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads) {
    // 断言给定的后端是 CPU 后端
    GGML_ASSERT(ggml_backend_is_cpu(backend_cpu));

    // 获取 CPU 后端的上下文，并设置线程数
    struct ggml_backend_cpu_context * ctx = (struct ggml_backend_cpu_context *)backend_cpu->context;
    ctx->n_threads = n_threads;
}

// 从指针创建 CPU 后端的缓冲区
ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(ggml_backend_t backend_cpu, void * ptr, size_t size) {
    return ggml_backend_buffer_init(backend_cpu, cpu_backend_buffer_i_from_ptr, ptr, size);
}

// 定义最大后端数量、最大分割数量和最大分割输入数量
#define GGML_MAX_BACKENDS 4
#define GGML_MAX_SPLITS 256
#define GGML_MAX_SPLIT_INPUTS 16
// 定义了一个结构体 ggml_backend_sched_split，用于表示后端调度的分割信息
struct ggml_backend_sched_split {
    ggml_tallocr_t tallocr;  // 分配器
    int i_start;  // 分割的起始位置
    int i_end;  // 分割的结束位置
    struct ggml_tensor * inputs[GGML_MAX_SPLIT_INPUTS];  // 输入张量数组
    int n_inputs;  // 输入张量的数量
    struct ggml_cgraph * graph;  // 计算图
};

// 定义了一个结构体 ggml_backend_sched，用于表示后端调度的信息
struct ggml_backend_sched {
    int n_backends;  // 后端数量
    ggml_backend_t backends[GGML_MAX_BACKENDS];  // 后端数组
    ggml_tallocr_t  tallocs[GGML_MAX_BACKENDS];  // 分配器数组

    ggml_gallocr_t galloc;  // 全局分配器

    struct ggml_hash_set    hash_set;  // 哈希集合
    ggml_tallocr_t *        node_talloc;  // 节点分配器数组，大小为 hash_set.size
    struct ggml_tensor * (* node_copies)[GGML_MAX_BACKENDS];  // 节点拷贝数组，大小为 hash_set.size * GGML_MAX_BACKENDS
}
// 定义指向 ggml_cgraph 结构体的指针变量 graph
struct ggml_cgraph * graph;
// 定义包含 GGML_MAX_SPLITS 个 ggml_backend_sched_split 结构体的数组变量 splits
struct ggml_backend_sched_split splits[GGML_MAX_SPLITS];
// 定义整型变量 n_splits
int n_splits;

// 定义指向 ggml_context 结构体的指针变量 ctx
struct ggml_context * ctx;

// 定义字符数组 context_buffer，长度为 GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS*sizeof(struct ggml_tensor) + GGML_MAX_SPLITS*sizeof(struct ggml_cgraph)，并按照 GGML_MEM_ALIGN 对齐
#ifdef _MSC_VER
__declspec(align(GGML_MEM_ALIGN))
#else
__attribute__((aligned(GGML_MEM_ALIGN)))
#endif
char context_buffer[GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS*sizeof(struct ggml_tensor) + GGML_MAX_SPLITS*sizeof(struct ggml_cgraph)];

// 定义宏 hash_id，用于查找或插入节点的哈希值
#define hash_id(node) ggml_hash_find_or_insert(sched->hash_set, node)
// 定义宏 node_allocr，用于分配节点
#define node_allocr(node) sched->node_talloc[hash_id(node)]

// 定义函数 ggml_is_view_op，用于判断操作是否为视图操作
static bool ggml_is_view_op(enum ggml_op op) {
    return op == GGML_OP_VIEW || op == GGML_OP_RESHAPE || op == GGML_OP_PERMUTE || op == GGML_OP_TRANSPOSE;
}
// 返回后端的优先级，值越小优先级越高
static int sched_backend_prio(ggml_backend_sched_t sched, ggml_backend_t backend) {
    // 遍历后端调度器中的所有后端
    for (int i = 0; i < sched->n_backends; i++) {
        // 如果找到指定的后端，返回其索引作为优先级
        if (sched->backends[i] == backend) {
            return i;
        }
    }
    // 如果未找到指定的后端，返回最大整数作为优先级
    return INT_MAX;
}

// 返回分配器的优先级，值越小优先级越高
static int sched_allocr_prio(ggml_backend_sched_t sched, ggml_tallocr_t allocr) {
    // 遍历后端调度器中的所有分配器
    for (int i = 0; i < sched->n_backends; i++) {
        // 如果找到指定的分配器，返回其索引作为优先级
        if (sched->tallocs[i] == allocr) {
            return i;
        }
    }
    // 如果未找到指定的分配器，返回最大整数作为优先级
    return INT_MAX;
}
// 定义一个数组用于存储导致节点选择的后端，用于调试，可以移除
char causes[GGML_DEFAULT_GRAPH_SIZE*4 + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS][128]; 

// 根据当前位置返回应该使用的后端
static ggml_backend_t sched_backend_from_cur(ggml_backend_sched_t sched, struct ggml_tensor * node) {
    // 如果目标张量已经在缓冲区中分配了，我们必须假设它对于保持在那里是至关重要的
    // 例如，kv 缓存更新
    // 注意，这不允许回退到 CPU。需要将输出张量添加到分割中，以将数据复制回原始后端。
    // 目标张量
    ggml_backend_t cur_backend = ggml_get_backend(node);
    if (cur_backend != NULL) {
        sprintf(causes[hash_id(node)], "1.dst");
        return cur_backend;
    }

    // 视图源
    if (node->view_src != NULL && ggml_get_backend(node->view_src) != NULL) {
        sprintf(causes[hash_id(node)], "1.vsrc");
        return ggml_get_backend(node->view_src);
    }
}
    // 初始化当前优先级为最大值，当前大小为0
    int cur_prio = INT_MAX;
    size_t cur_size = 0;

    // 遍历节点的输入源
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        // 获取当前输入源
        const struct ggml_tensor * src = node->src[i];
        // 如果输入源为空，则跳出循环
        if (src == NULL) {
            break;
        }
        // 获取输入源的后端
        ggml_backend_t src_backend = ggml_get_backend(src);
        // 如果输入源的后端不为空
        if (src_backend != NULL) {
            // 获取输入源的调度优先级
            int src_prio = sched_backend_prio(sched, src_backend);
            // 获取输入源的大小
            size_t src_size = ggml_nbytes(src);
            // 如果输入源的优先级低于当前优先级，并且大小大于等于当前大小
            if (src_prio < cur_prio && src_size >= cur_size) {
                // 更新当前优先级和大小
                cur_prio = src_prio;
                cur_size = src_size;
                // 更新当前后端为输入源的后端
                cur_backend = src_backend;
                // 格式化记录当前原因
                sprintf(causes[hash_id(node)], "1.src%d", i);
            }
        }
    }
    }
    return cur_backend;
}

// 格式化输出文件大小
static char * fmt_size(size_t size) {
    static char buffer[128];
    if (size >= 1024*1024) {
        sprintf(buffer, "%zuM", size/1024/1024);
    } else {
        sprintf(buffer, "%zuK", size/1024);
    }
    return buffer;
}

// 打印调度分配情况
static void sched_print_assignments(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    int cur_split = 0;
    for (int i = 0; i < graph->n_nodes; i++) {
        if (cur_split < sched->n_splits && i == sched->splits[cur_split].i_start) {
            // 获取分配的后端
            ggml_backend_t split_backend = ggml_tallocr_get_buffer(sched->splits[cur_split].tallocr)->backend;
            // 打印分配情况
            fprintf(stderr, "\n## SPLIT #%d: %s # %d inputs: ", cur_split, ggml_backend_name(split_backend), sched->splits[cur_split].n_inputs);
// 遍历当前分割的输入，打印输入的名称和数据大小
for (int j = 0; j < sched->splits[cur_split].n_inputs; j++) {
    fprintf(stderr, "[%s (%5.5s)] ", sched->splits[cur_split].inputs[j]->name, fmt_size(ggml_nbytes(sched->splits[cur_split].inputs[j])));
}
// 打印换行符
fprintf(stderr, "\n");
// 增加当前分割的索引
cur_split++;
// 获取当前节点
struct ggml_tensor * node = graph->nodes[i];
// 如果当前节点是视图操作，则跳过
if (ggml_is_view_op(node->op)) {
    continue;
}
// 获取当前节点的分配器
ggml_tallocr_t node_allocr = node_allocr(node);
// 获取当前节点的后端
ggml_backend_t node_backend = node_allocr ? ggml_tallocr_get_buffer(node_allocr)->backend : NULL;
// 遍历当前节点的源
for (int j = 0; j < GGML_MAX_SRC; j++) {
    struct ggml_tensor * src = node->src[j];
    // 如果源为空，则跳出循环
    if (src == NULL) {
        break;
    }
    // 获取源的分配器
    ggml_tallocr_t src_allocr = node_allocr(src);
    // 获取源的后端
    ggml_backend_t src_backend = src_allocr ? ggml_tallocr_get_buffer(src_allocr)->backend : NULL;
    // 打印源的名称、数据大小、后端名称和引起的哈希值
    fprintf(stderr, " %20.20s (%4.4s) [%4.4s %8.8s]", src->name, fmt_size(ggml_nbytes(src)), src_backend ? ggml_backend_name(src_backend) : "NULL", causes[hash_id(src)]);
}
// 关闭大括号，结束一个代码块
        }
        // 输出换行符到标准错误流
        fprintf(stderr, "\n");
    }
}

// 创建一个具有相同内存布局的张量副本
static struct ggml_tensor * ggml_dup_tensor_layout(struct ggml_context * ctx, const struct ggml_tensor * tensor) {
    // 创建张量的副本
    struct ggml_tensor * dup = ggml_dup_tensor(ctx, tensor);
    // 遍历张量的维度，复制维度大小
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        dup->nb[i] = tensor->nb[i];
    }
    // 返回副本
    return dup;
}

// 为操作分配后端，并将图拆分为可以在相同后端上计算的子图
// 待办事项：合并通行证
static void sched_split_graph(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    // 重置状态
    size_t hash_size = sched->hash_set.size;
    // 将哈希集合的键数组清零
    memset(sched->hash_set.keys, 0, sizeof(sched->hash_set.keys[0]) * hash_size);
    // 用0填充sched->node_talloc数组，数组长度为hash_size
    memset(sched->node_talloc,   0, sizeof(sched->node_talloc[0])   * hash_size);
    // 用0填充sched->node_copies数组，数组长度为hash_size
    memset(sched->node_copies,   0, sizeof(sched->node_copies[0])   * hash_size);
    // 初始化sched->n_splits为0
    sched->n_splits = 0;

    // 初始化ggml_init_params结构体
    struct ggml_init_params params = {
        /*.mem_size =   */ sizeof(sched->context_buffer),
        /*.mem_buffer = */ sched->context_buffer,
        /*.no_alloc =   */ true
    };

    // 如果sched->ctx不为空，则释放其内存
    if (sched->ctx != NULL) {
        ggml_free(sched->ctx);
    }

    // 初始化sched->ctx，调用ggml_init函数
    sched->ctx = ggml_init(params);

    // pass 1: assign backends to ops with allocated inputs
    // 遍历图的叶子节点
    for (int i = 0; i < graph->n_leafs; i++) {
        // 获取当前叶子节点
        struct ggml_tensor * leaf = graph->leafs[i];
        // 如果叶子节点的分配器不为空
        if (node_allocr(leaf) != NULL) {
    // 不要覆盖用户的赋值
    // 继续下一次循环
    continue;
}

// 获取叶子节点的后端
ggml_backend_t leaf_backend = ggml_get_backend(leaf);

// 如果叶子节点的后端为空并且视图源不为空，则获取视图源的后端
if (leaf_backend == NULL && leaf->view_src != NULL) {
    leaf_backend = ggml_get_backend(leaf->view_src);
}

// 如果叶子节点的后端不为空，则为节点分配内存
if (leaf_backend != NULL) {
    node_allocr(leaf) = ggml_backend_sched_get_tallocr(sched, leaf_backend);
}

// 遍历图中的节点
for (int i = 0; i < graph->n_nodes; i++) {
    struct ggml_tensor * node = graph->nodes[i];
    
    // 如果节点已经分配了内存，则继续下一次循环
    if (node_allocr(node) != NULL) {
        continue;
    }
    
    // 获取节点的后端
    ggml_backend_t node_backend = sched_backend_from_cur(sched, node);
    
    // 如果节点的后端不为空，则进行相应的操作
    if (node_backend != NULL) {
    // 为节点分配内存分配器
    node_allocr(node) = ggml_backend_sched_get_tallocr(sched, node_backend);
    }
}

// pass 2: assign backends to ops from current assignments
// 第二步：根据当前分配情况为操作分配后端
// TODO:
//  - reuse sched_backend_from_cur
//  - 重用 sched_backend_from_cur
for (int i = 0; i < graph->n_nodes; i++) {
    // 遍历图中的节点
    struct ggml_tensor * node = graph->nodes[i];
    // 获取节点的内存分配器
    ggml_tallocr_t node_allocr = node_allocr(node);
    // 如果节点的内存分配器为空
    if (node_allocr == NULL) {
        // 初始化当前优先级和大小
        int    cur_prio = INT_MAX;
        size_t cur_size = 0;
        // 遍历节点的源
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * src = node->src[j];
            // 如果源为空，跳出循环
            if (src == NULL) {
                break;
            }
            // 获取源的内存分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
    // 如果源分配器不为空
    if (src_allocr != NULL) {
        // 获取源分配器的优先级
        int src_prio = sched_allocr_prio(sched, src_allocr);
        // 获取源节点的大小
        size_t src_size = ggml_nbytes(src);
        // 如果源节点的优先级小于当前优先级并且源节点的大小大于等于当前大小
        if (src_prio < cur_prio && src_size >= cur_size) {
            // 更新当前优先级和大小
            cur_prio = src_prio;
            cur_size = src_size;
            // 更新节点的分配器
            node_allocr = src_allocr;
            // 格式化原因字符串
            sprintf(causes[hash_id(node)], "2.src%d", j);
        }
    }
    // 如果节点的分配器不为空
    if (node_allocr != NULL) {
        // 分配节点的分配器
        node_allocr(node) = node_allocr;
    }
}
//printf("PASS 2 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

// pass 3: assign backends to remaining src from dst (should only be leafs)
// 第三步：将后端分配给剩余的源节点（应该只有叶子节点）
for (int i = 0; i < graph->n_nodes; i++) {
    // 获取图中第 i 个节点的指针
    struct ggml_tensor * node = graph->nodes[i];
    // 获取节点的内存分配器
    ggml_tallocr_t node_allocr = node_allocr(node);
    // 遍历节点的输入
    for (int j = 0; j < GGML_MAX_SRC; j++) {
        // 获取节点的第 j 个输入张量的指针
        struct ggml_tensor * src = node->src[j];
        // 如果输入为空，则跳出循环
        if (src == NULL) {
            break;
        }
        // 获取输入张量的内存分配器
        ggml_tallocr_t src_allocr = node_allocr(src);
        // 如果输入张量的内存分配器为空，则将节点的内存分配器赋给输入张量的内存分配器
        if (src_allocr == NULL) {
            node_allocr(src) = node_allocr;
        }
    }
    //printf("PASS 3 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 4: split graph, find tensors that need to be copied
    // TODO:
    //  - when switching from a less preferred backend to a more preferred backend, check if it is possible to move the switch to an earlier point for the same cost
    // find first backend
    // 初始化当前分割点为 0
    int cur_split = 0;
    // 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果当前节点的视图源为空
        if (node->view_src == NULL) {
            // 为调度器的第一个分割设置节点的分配器
            sched->splits[0].tallocr = node_allocr(node);
            // 跳出循环
            break;
        }
    }
    // 设置调度器的第一个分割的起始索引和输入数量
    sched->splits[0].i_start = 0;
    sched->splits[0].n_inputs = 0;
    // 将调度器的第一个分割的输入数组清零
    memset(sched->splits[0].inputs, 0, sizeof(sched->splits[0].inputs)); //HACK
    // 获取当前分割的分配器和后端 ID
    ggml_tallocr_t cur_allocr = sched->splits[0].tallocr;
    size_t cur_backend_id = sched_allocr_prio(sched, cur_allocr);
    // 再次遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果当前节点是视图操作，则跳过
        if (ggml_is_view_op(node->op)) {
            continue;
        }
        // 获取当前节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 如果节点的分配器与当前分配器不同
        if (node_allocr != cur_allocr) {
            // 设置当前分割的结束索引为当前索引
            sched->splits[cur_split].i_end = i;
            // 增加当前分割索引
            cur_split++;
            // 断言当前分割索引小于最大分割数
            GGML_ASSERT(cur_split < GGML_MAX_SPLITS);
            // 设置新的分割的分配器
            sched->splits[cur_split].tallocr = node_allocr;
            // 设置新的分割的起始索引为当前索引
            sched->splits[cur_split].i_start = i;
            // 设置新的分割的输入数量为0
            sched->splits[cur_split].n_inputs = 0;
            // 将新的分割的输入数组清零
            memset(sched->splits[cur_split].inputs, 0, sizeof(sched->splits[cur_split].inputs)); //HACK
            // 更新当前分配器为节点的分配器
            cur_allocr = node_allocr;
            // 更新当前后端ID为节点分配器的优先级
            cur_backend_id = sched_allocr_prio(sched, cur_allocr);
        }

        // 查找不在同一后端的输入
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取节点的第j个输入
            struct ggml_tensor * src = node->src[j];
            // 如果输入为空，跳出循环
            if (src == NULL) {
                break;
            }
            // 获取输入的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
// 如果源节点的分配器与当前节点的分配器不同
if (src_allocr != node_allocr) {
    // 增加当前分裂的输入数量
    int n_inputs = sched->splits[cur_split].n_inputs++;
    // 断言输入数量小于最大输入数量
    GGML_ASSERT(n_inputs < GGML_MAX_SPLIT_INPUTS);
    // 将源节点添加到当前分裂的输入中
    sched->splits[cur_split].inputs[n_inputs] = (struct ggml_tensor *)src;

    // 创建源节点的副本
    size_t id = hash_id(src);
    // 如果当前节点的副本在当前后端中不存在
    if (sched->node_copies[id][cur_backend_id] == NULL) {
        // 复制源节点的张量布局
        struct ggml_tensor * tensor_copy = ggml_dup_tensor_layout(sched->ctx, src);
        // 将源节点的副本存储在当前后端中
        sched->node_copies[id][cur_backend_id] = tensor_copy;
        // 设置副本的分配器
        node_allocr(tensor_copy) = cur_allocr;
        // 获取当前分配器对应的后端
        ggml_backend_t backend = ggml_tallocr_get_buffer(cur_allocr)->backend;
        // 为副本设置格式名称
        ggml_format_name(tensor_copy, "%s#%s", ggml_backend_name(backend), src->name);
    }
    // 将当前节点的源节点替换为副本
    node->src[j] = sched->node_copies[id][cur_backend_id];
}
// 更新分裂的结束索引和分裂数量
sched->splits[cur_split].i_end = graph->n_nodes;
sched->n_splits = cur_split + 1;
    // 打印调试信息，输出当前分配情况和图形结构
    //fprintf(stderr, "PASS 4 ASSIGNMENTS\n"); sched_print_assignments(sched, graph); fflush(stdout);

#if 1
    // 检查：所有源节点应该与节点具有相同的后端
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 如果节点没有分配器，则输出错误信息
        if (node_allocr == NULL) {
            fprintf(stderr, "!!!!!!! %s has no backend\n", node->name);
        }
        // 遍历节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前源节点
            struct ggml_tensor * src = node->src[j];
            // 如果源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取源节点的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果源节点的分配器与节点的分配器不同，则输出错误信息
            if (src_allocr != node_allocr /* && src_backend != NULL */) { // ignore nulls for now
                fprintf(stderr, "!!!! %s has backend %s, src %d (%s) has backend %s\n",
                    node->name, node_allocr ? ggml_backend_name(ggml_tallocr_get_buffer(node_allocr)->backend) : "NULL",
// 为每个分割创建图的副本
// FIXME: 避免这种复制，以某种其他方式将分割输入传递给ggml_gallocr_alloc_graph_n
struct ggml_cgraph * graph_copy = ggml_new_graph_custom(sched->ctx, graph->n_nodes + sched->n_splits*GGML_MAX_SPLIT_INPUTS, false);
for (int i = 0; i < sched->n_splits; i++) {
    struct ggml_backend_sched_split * split = &sched->splits[i];
    split->graph = ggml_graph_view(sched->ctx, graph, split->i_start, split->i_end);

    // 将输入添加到图的副本中，以便它们在分割开始时由ggml-alloc分配
    for (int j = 0; j < split->n_inputs; j++) {
        struct ggml_tensor * input = split->inputs[j];
        struct ggml_tensor * input_cpy = sched->node_copies[hash_id(input)][sched_allocr_prio(sched, split->tallocr)];
        input_cpy->src[0] = input;
        graph_copy->nodes[graph_copy->n_nodes++] = input_cpy;
    }
}
// 遍历从 i_start 到 i_end 的节点，将其复制到新的图中的节点数组中
for (int j = split->i_start; j < split->i_end; j++) {
    graph_copy->nodes[graph_copy->n_nodes++] = graph->nodes[j];
}

// 将新图赋值给调度器的图
sched->graph = graph_copy;
}

// 分配调度器的图的拆分
static void sched_alloc_splits(ggml_backend_sched_t sched) {
    ggml_gallocr_alloc_graph_n(
        sched->galloc,
        sched->graph,
        sched->hash_set,
        sched->node_talloc);
}

// 计算调度器的图的拆分
static void sched_compute_splits(ggml_backend_sched_t sched) {
    // 初始化用于复制和计算的时间数组
    uint64_t copy_us[GGML_MAX_BACKENDS] = {0};
    uint64_t compute_us[GGML_MAX_BACKENDS] = {0};
    // 获取调度器中的分割结构体数组
    struct ggml_backend_sched_split * splits = sched->splits;

    // 遍历分割结构体数组
    for (int i = 0; i < sched->n_splits; i++) {
        // 获取当前分割结构体
        struct ggml_backend_sched_split * split = &splits[i];
        // 获取分割结构体对应的后端
        ggml_backend_t split_backend = ggml_tallocr_get_buffer(split->tallocr)->backend;
        // 获取分割结构体对应的后端 ID
        int split_backend_id = sched_backend_prio(sched, split_backend);

        // 将输入张量复制到分割后的后端
        uint64_t copy_start_us = ggml_time_us();
        // 遍历分割结构体的输入张量
        for (int j = 0; j < split->n_inputs; j++) {
            // 获取输入张量的副本
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(split->inputs[j])][sched_backend_prio(sched, split_backend)];
            // 如果输入张量的缓冲区为空
            if (split->inputs[j]->buffer == NULL) {
                // 如果输入张量的视图源为空
                if (split->inputs[j]->view_src == NULL) {
                    // 输出错误信息并退出程序
                    fprintf(stderr, "input %s has no buffer and no view_src\n", split->inputs[j]->name);
                    exit(1);
                }
                // 获取输入张量的视图
                struct ggml_tensor * view = split->inputs[j];
                // 设置视图的后端为视图源的后端
                view->backend = view->view_src->backend;
                // 设置视图的缓冲区为视图源的缓冲区
                view->buffer  = view->view_src->buffer;
                // 设置视图的数据为视图源数据的偏移
                view->data    = (char *)view->view_src->data + view->view_offs;
// 初始化一个张量，将调度器中视图的缓冲区传递给后端缓冲区
ggml_backend_buffer_init_tensor(ggml_backend_sched_get_buffer(sched, view->buffer->backend), view);
// 如果输入的拷贝没有缓冲区，则输出错误信息并退出程序
if (input_cpy->buffer == NULL) {
    fprintf(stderr, "input_cpy %s has no buffer\n", input_cpy->name);
    exit(1);
}
// 断言拆分的输入张量的后端与输入拷贝的后端不同
GGML_ASSERT(split->inputs[j]->buffer->backend != input_cpy->buffer->backend);
// 断言输入拷贝的后端与拆分的后端相同
GGML_ASSERT(input_cpy->buffer->backend == split_backend);
// 将输入拷贝的数据复制到拆分的输入张量中
ggml_backend_tensor_copy(split->inputs[j], input_cpy);
// 记录拷贝操作结束的时间
int64_t copy_end_us = ggml_time_us();
copy_us[split_backend_id] += copy_end_us - copy_start_us;

#if 0
// 创建一个文件名，用于保存拆分图的可视化表示
char split_filename[GGML_MAX_NAME];
snprintf(split_filename, GGML_MAX_NAME, "split_%i_%s.dot", i, ggml_backend_name(split_backend));
ggml_graph_dump_dot(split->graph, NULL, split_filename);
#endif
// 计算开始时间，以微秒为单位
uint64_t compute_start_us = ggml_time_us();
// 使用指定的后端计算图形
ggml_backend_graph_compute(split_backend, split->graph);
// 同步后端
// ggml_backend_synchronize(split_backend);
// 计算结束时间，以微秒为单位
uint64_t compute_end_us = ggml_time_us();
// 更新计算时间
compute_us[split_backend_id] += compute_end_us - compute_start_us;
}

#if 0
// 每个后端的计时
fprintf(stderr, "sched_compute_splits times (%d splits):\n", sched->n_splits);
for (int i = 0; i < sched->n_backends; i++) {
    if (copy_us[i] > 0 || compute_us[i] > 0) {
        fprintf(stderr, "\t%5.5s: %lu us copy, %lu us compute\n", ggml_backend_name(sched->backends[i]), copy_us[i], compute_us[i]);
    }
}
#endif
}

// 重置调度器
static void sched_reset(ggml_backend_sched_t sched) {
    for (int i = 0; i < sched->n_backends; i++) {
    // 重置调度器中每个线程的内存分配器
    ggml_tallocr_reset(sched->tallocs[i]);
}

// 创建一个新的后端调度器
ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, int n_backends) {
    // 确保后端数量不超过最大限制
    GGML_ASSERT(n_backends <= GGML_MAX_BACKENDS);

    // 分配内存给后端调度器
    struct ggml_backend_sched * sched = malloc(sizeof(struct ggml_backend_sched));
    // 将分配的内存清零
    memset(sched, 0, sizeof(struct ggml_backend_sched));

    // 打印后端调度器的大小
    fprintf(stderr, "ggml_backend_sched size: %lu KB\n", sizeof(struct ggml_backend_sched)/1024);

    // 设置后端数量
    sched->n_backends = n_backends;
    // 将传入的后端数组复制到调度器中
    for (int i = 0; i < n_backends; i++) {
        sched->backends[i] = backends[i];
    }

    // 初始化全局内存分配器
    sched->galloc = ggml_gallocr_new();

    // 为每个后端初始化测量内存分配
# 遍历 n_backends 数组，为每个元素创建一个新的 ggml_tallocr 对象，并将其存储在 sched->tallocs 数组中
for (int i = 0; i < n_backends; i++) {
    sched->tallocs[i] = ggml_tallocr_new_measure_from_backend(backends[i]);
}

# 释放 ggml_backend_sched_t 对象占用的内存
void ggml_backend_sched_free(ggml_backend_sched_t sched) {
    # 如果 sched 为 NULL，则直接返回，不进行任何操作
    if (sched == NULL) {
        return;
    }
    # 遍历 sched->n_backends 数组，释放每个 ggml_tallocr 对象占用的内存
    for (int i = 0; i < sched->n_backends; i++) {
        ggml_tallocr_free(sched->tallocs[i]);
    }
    # 释放 sched->galloc 占用的内存
    ggml_gallocr_free(sched->galloc);
    # 释放 sched->hash_set.keys 占用的内存
    free(sched->hash_set.keys);
    # 释放 sched->node_talloc 占用的内存
    free(sched->node_talloc);
    # 释放 sched->node_copies 占用的内存
    free(sched->node_copies);
    # 释放 sched 对象占用的内存
    free(sched);
}
// 初始化测量调度器，传入调度器和测量图
void ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph) {
    // 初始化哈希表的大小
    size_t hash_size = measure_graph->visited_hash_table.size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS;
    // 分配哈希表的键数组内存空间
    sched->hash_set.size = hash_size;
    sched->hash_set.keys = malloc(sizeof(sched->hash_set.keys[0]) * hash_size);
    // 分配节点内存空间
    sched->node_talloc   = malloc(sizeof(sched->node_talloc[0])   * hash_size);
    sched->node_copies   = malloc(sizeof(sched->node_copies[0])   * hash_size);

    // 对测量图进行分割
    sched_split_graph(sched, measure_graph);
    // 分配分割的内存空间
    sched_alloc_splits(sched);

    // 分配缓冲区并重置分配器
    for (int i = 0; i < sched->n_backends; i++) {
        // 获取每个后端的最大分配大小
        size_t size = ggml_tallocr_max_size(sched->tallocs[i]);
        // 释放当前分配器的内存
        ggml_tallocr_free(sched->tallocs[i]);
        // 从后端创建新的分配器
        sched->tallocs[i] = ggml_tallocr_new_from_backend(sched->backends[i], size);
    }

    // 重置调度器
    sched_reset(sched);
}
// 计算图的计算调度，确保调度哈希集合的大小大于等于图的访问哈希表大小加上最大分割数乘以最大分割输入数
void ggml_backend_sched_graph_compute(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    GGML_ASSERT(sched->hash_set.size >= graph->visited_hash_table.size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS);

    // 对图进行分割调度
    sched_split_graph(sched, graph);
    // 分配分割
    sched_alloc_splits(sched);
    // 计算分割
    sched_compute_splits(sched);
    // 重置调度
    sched_reset(sched);
}

// 获取调度器的内存分配器
ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend) {
    // 获取后端的索引
    int backend_index = sched_backend_prio(sched, backend);
    // 返回对应后端的内存分配器
    return sched->tallocs[backend_index];
}

// 获取调度器的缓冲区
ggml_backend_buffer_t ggml_backend_sched_get_buffer(ggml_backend_sched_t sched, ggml_backend_t backend) {
    // 获取后端的索引
    int backend_index = sched_backend_prio(sched, backend);
    // 返回对应后端的缓冲区
    return ggml_tallocr_get_buffer(sched->tallocs[backend_index]);
}
# 设置节点的后端计算引擎
void ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend) {
    # 获取后端计算引擎的索引
    int backend_index = sched_backend_prio(sched, backend);
    # 断言后端计算引擎索引在有效范围内
    GGML_ASSERT(backend_index >= 0 && backend_index < sched->n_backends);
    # 为节点分配后端计算引擎的内存分配器
    node_allocr(node) = sched->tallocs[backend_index];
}
```