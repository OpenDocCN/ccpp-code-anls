# `whisper.cpp\bindings\ruby\ext\ggml-backend.c`

```cpp
// 包含头文件 ggml-backend-impl.h、ggml-alloc.h、ggml-impl.h
#include "ggml-backend-impl.h"
#include "ggml-alloc.h"
#include "ggml-impl.h"

// 包含标准库头文件
#include <assert.h>
#include <limits.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 定义宏 UNUSED 为 GGML_UNUSED
#define UNUSED GGML_UNUSED

// 定义宏 MAX(a, b) 为返回 a 和 b 中的最大值
#define MAX(a, b) ((a) > (b) ? (a) : (b)

// backend buffer

// 初始化 backend buffer
ggml_backend_buffer_t ggml_backend_buffer_init(
        struct ggml_backend                  * backend,
        struct ggml_backend_buffer_i           iface,
               ggml_backend_buffer_context_t   context,
               size_t                          size) {
    // 分配内存给 buffer
    ggml_backend_buffer_t buffer = malloc(sizeof(struct ggml_backend_buffer));

    // 断言 iface.get_base 不为空
    GGML_ASSERT(iface.get_base != NULL);

    // 初始化 buffer 结构体
    (*buffer) = (struct ggml_backend_buffer) {
        /* .interface = */ iface,
        /* .backend   = */ backend,
        /* .context   = */ context,
        /* .size      = */ size,
    };

    return buffer;
}

// 释放 backend buffer
void ggml_backend_buffer_free(ggml_backend_buffer_t buffer) {
    if (buffer == NULL) {
        return;
    }

    // 如果 iface.free_buffer 不为空，则调用其释放函数
    if (buffer->iface.free_buffer != NULL) {
        buffer->iface.free_buffer(buffer);
    }
    // 释放 buffer 内存
    free(buffer);
}

// 获取 backend buffer 的对齐方式
size_t ggml_backend_buffer_get_alignment(ggml_backend_buffer_t buffer) {
    return ggml_backend_get_alignment(buffer->backend);
}

// 获取 backend buffer 的大小
size_t ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer) {
    return buffer->size;
}

// 获取 backend buffer 的基地址
void * ggml_backend_buffer_get_base(ggml_backend_buffer_t buffer) {
    void * base = buffer->iface.get_base(buffer);

    // 断言 base 不为空，否则输出错误信息
    GGML_ASSERT(base != NULL && "backend buffer base cannot be NULL");

    return base;
}

// 获取 backend buffer 分配给 tensor 的大小
size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // 如果 iface.get_alloc_size 存在，则调用其函数，否则返回 ggml_nbytes(tensor)
    if (buffer->iface.get_alloc_size) {
        return buffer->iface.get_alloc_size(buffer, tensor);
    }
    return ggml_nbytes(tensor);
}

// 初始化 backend buffer 的 tensor
void ggml_backend_buffer_init_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // init_tensor 是可选的
    # 检查缓冲区接口是否存在初始化张量的函数
    if (buffer->iface.init_tensor) {
        # 如果存在，调用初始化张量的函数，传入缓冲区和张量作为参数
        buffer->iface.init_tensor(buffer, tensor);
    }
// 释放张量的缓冲区，如果存在释放函数则调用
void ggml_backend_buffer_free_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // 如果存在释放函数，则调用该函数释放张量
    if (buffer->iface.free_tensor) {
        buffer->iface.free_tensor(buffer, tensor);
    }
}

// 获取张量所属的后端
ggml_backend_t ggml_get_backend(const struct ggml_tensor * tensor) {
    // 如果张量的缓冲区存在，则返回其所属的后端，否则返回 NULL
    return tensor->buffer ? tensor->buffer->backend : NULL;
}

// 获取后端的名称
const char * ggml_backend_name(ggml_backend_t backend) {
    // 如果后端为 NULL，则返回字符串 "NULL"，否则调用后端的获取名称函数
    if (backend == NULL) {
        return "NULL";
    }
    return backend->iface.get_name(backend);
}

// 释放后端资源
void ggml_backend_free(ggml_backend_t backend) {
    // 如果后端为 NULL，则直接返回
    if (backend == NULL) {
        return;
    }

    // 调用后端的释放函数
    backend->iface.free(backend);
}

// 分配后端缓冲区
ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size) {
    // 调用后端的分配缓冲区函数
    return backend->iface.alloc_buffer(backend, size);
}

// 获取后端的对齐方式
size_t ggml_backend_get_alignment(ggml_backend_t backend) {
    // 调用后端的获取对齐方式函数
    return backend->iface.get_alignment(backend);
}

// 异步设置张量数据
void ggml_backend_tensor_set_async(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    // 获取张量所属的后端，并调用后端的异步设置张量数据函数
    ggml_get_backend(tensor)->iface.set_tensor_async(ggml_get_backend(tensor), tensor, data, offset, size);
}

// 异步获取张量数据
void ggml_backend_tensor_get_async(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 获取张量所属的后端，并调用后端的异步获取张量数据函数
    ggml_get_backend(tensor)->iface.get_tensor_async(ggml_get_backend(tensor), tensor, data, offset, size);
}

// 设置张量数据
void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    // 获取张量所属的后端
    ggml_backend_t backend = ggml_get_backend(tensor);

    // 断言张量数据不为空，并且后端已设置
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    GGML_ASSERT(backend != NULL && "tensor backend not set");

    // 调用后端的异步设置张量数据函数，并同步
    backend->iface.set_tensor_async(backend, tensor, data, offset, size);
    backend->iface.synchronize(backend);
}

// 获取张量数据
void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 获取张量所属的后端
    ggml_backend_t backend = ggml_get_backend(tensor);
    # 断言张量数据已经分配内存，否则输出错误信息
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    # 断言后端已经设置，否则输出错误信息
    GGML_ASSERT(backend != NULL && "tensor backend not set");

    # 调用后端接口异步获取张量数据
    backend->iface.get_tensor_async(backend, tensor, data, offset, size);
    # 同步后端操作，确保获取数据完成
    backend->iface.synchronize(backend);
// 同步后端
void ggml_backend_synchronize(ggml_backend_t backend) {
    // 调用后端接口的同步函数
    backend->iface.synchronize(backend);
}

// 创建图计划
ggml_backend_graph_plan_t ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 调用后端接口的创建图计划函数
    return backend->iface.graph_plan_create(backend, cgraph);
}

// 释放图计划
void ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    // 调用后端接口的释放图计划函数
    backend->iface.graph_plan_free(backend, plan);
}

// 计算图计划
void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    // 调用后端接口的计算图计划函数
    backend->iface.graph_plan_compute(backend, plan);
}

// 计算图
bool ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 调用后端接口的计算图函数
    return backend->iface.graph_compute(backend, cgraph);
}

// 检查后端是否支持操作
bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    // 调用后端接口的支持操作函数
    return backend->iface.supports_op(backend, op);
}

// 后端复制

// 检查两个张量是否具有相同的布局
static bool ggml_are_same_layout(const struct ggml_tensor * a, const struct ggml_tensor * b) {
    if (a->type != b->type) {
        return false;
    }
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        if (a->ne[i] != b->ne[i]) {
            return false;
        }
        if (a->nb[i] != b->nb[i]) {
            return false;
        }
    }
    return true;
}

// 复制张量
void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 打印源张量和目标张量的布局信息
    //printf("src: %s ne: [%d %d %d %d] nb: [%d %d %d %d]\n", src->name, (int)src->ne[0], (int)src->ne[1], (int)src->ne[2], (int)src->ne[3], (int)src->nb[0], (int)src->nb[1], (int)src->nb[2], (int)src->nb[3]);
    //printf("dst: %s ne: [%d %d %d %d] nb: [%d %d %d %d]\n", dst->name, (int)dst->ne[0], (int)dst->ne[1], (int)dst->ne[2], (int)dst->ne[3], (int)dst->nb[0], (int)dst->nb[1], (int)dst->nb[2], (int)dst->nb[3]);
    // 断言源张量和目标张量具有相同的布局，否则无法复制
    GGML_ASSERT(ggml_are_same_layout(src, dst) && "cannot copy tensors with different layouts");
}
    // 打印日志，复制张量的名称、源后端和目标后端，以及数据大小
    fprintf(stderr, "cpy tensor %s from %s to %s (%lu bytes)\n", src->name, ggml_backend_name(src->backend), ggml_backend_name(dst->backend), ggml_nbytes(src));

    // 如果源张量和目标张量相同，则直接返回，不进行复制操作
    if (src == dst) {
        return;
    }

    // TODO: 允许后端支持相同后端之间的复制

    // 如果目标后端支持从其他后端复制张量数据
    if (ggml_get_backend(dst)->iface.cpy_tensor_from != NULL) {
        ggml_get_backend(dst)->iface.cpy_tensor_from(ggml_get_backend(dst)->context, src, dst);
    } 
    // 如果源后端支持向其他后端复制张量数据
    else if (ggml_get_backend(src)->iface.cpy_tensor_to != NULL) {
        ggml_get_backend(src)->iface.cpy_tensor_to(ggml_get_backend(src)->context, src, dst);
    } 
    // 如果以上两种情况都不满足，则使用默认的获取和设置方法进行复制操作
    else {
        // 在从/向 CPU 复制时不应该执行到这里
        #ifndef NDEBUG
        // 打印警告日志，说明源后端和目标后端都未实现复制方法，将使用默认的获取和设置方法
        fprintf(stderr, "ggml_backend_tensor_copy: neither cpy_tensor_from nor cpy_tensor_to are implemented for backends %s and %s, falling back to get/set\n", ggml_backend_name(src->buffer->backend), ggml_backend_name(dst->buffer->backend));
        #endif
        // 获取源张量数据的大小，并分配内存
        size_t nbytes = ggml_nbytes(src);
        void * data = malloc(nbytes);
        // 获取源张量数据并设置到目标张量
        ggml_backend_tensor_get(src, data, 0, nbytes);
        ggml_backend_tensor_set(dst, data, 0, nbytes);
        // 释放内存
        free(data);
    }
// 结构体定义，用于存储 CPU 后端的上下文信息
struct ggml_backend_cpu_context {
    int n_threads; // 线程数
    void * work_data; // 工作数据
    size_t work_size; // 工作数据大小
};

// 返回 CPU 后端的名称
static const char * ggml_backend_cpu_name(ggml_backend_t backend) {
    return "CPU"; // 返回字符串 "CPU"

    UNUSED(backend); // 未使用的参数
}

// 释放 CPU 后端资源
static void ggml_backend_cpu_free(ggml_backend_t backend) {
    // 获取 CPU 上下文信息
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;
    // 释放工作数据
    free(cpu_ctx->work_data);
    // 释放 CPU 上下文信息
    free(cpu_ctx);
    // 释放后端资源
    free(backend);
}

// 获取 CPU 后端缓冲区的基地址
static void * ggml_backend_cpu_buffer_get_base(ggml_backend_buffer_t buffer) {
    return (void *)buffer->context; // 返回缓冲区的上下文信息
}

// 释放 CPU 后端缓冲区
static void ggml_backend_cpu_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    free(buffer->context); // 释放缓冲区的上下文信息
    UNUSED(buffer); // 未使用的参数
}

// CPU 后端缓冲区接口
static struct ggml_backend_buffer_i cpu_backend_buffer_i = {
    /* .free_buffer    = */ ggml_backend_cpu_buffer_free_buffer, // 释放缓冲区
    /* .get_base       = */ ggml_backend_cpu_buffer_get_base, // 获取缓冲区基地址
    /* .get_alloc_size = */ NULL, // 默认为 ggml_nbytes
    /* .init_tensor    = */ NULL, // 不需要初始化
    /* .free_tensor    = */ NULL, // 不需要清理
};

// 对于从指针创建的缓冲区，不需要调用 free
static struct ggml_backend_buffer_i cpu_backend_buffer_i_from_ptr = {
    /* .free_buffer    = */ NULL, // 指针不由缓冲区拥有，因此不需要释放
    /* .get_base       = */ ggml_backend_cpu_buffer_get_base, // 获取缓冲区基地址
    /* .get_alloc_size = */ NULL, // 默认为 ggml_nbytes
    /* .init_tensor    = */ NULL, // 不需要初始化
    /* .free_tensor    = */ NULL, // 不需要清理
};

// 张量对齐大小
static const size_t TENSOR_ALIGNMENT = 64; // 对于 AVX 512 应该足够

// 分配 CPU 后端缓冲区
static ggml_backend_buffer_t ggml_backend_cpu_alloc_buffer(ggml_backend_t backend, size_t size) {
    size += TENSOR_ALIGNMENT; // 增加对齐大小，因为 malloc 可能返回未对齐的地址
    void * data = malloc(size); // 分配内存

    GGML_ASSERT(data != NULL && "failed to allocate buffer"); // 断言内存分配成功

    return ggml_backend_buffer_init(backend, cpu_backend_buffer_i, data, size); // 初始化后端缓冲区
}
// 获取 CPU 后端的对齐方式
static size_t ggml_backend_cpu_get_alignment(ggml_backend_t backend) {
    // 返回张量的对齐方式
    return TENSOR_ALIGNMENT;
    // 未使用的参数，避免编译器警告
    UNUSED(backend);
}

// 设置异步张量数据到 CPU 后端
static void ggml_backend_cpu_set_tensor_async(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    // 检查写入数据是否超出张量边界
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor write out of bounds");
    // 检查张量是否已分配内存
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");

    // 将数据从源地址复制到目标地址
    memcpy((char *)tensor->data + offset, data, size);

    // 未使用的参数，避免编译器警告
    UNUSED(backend);
}

// 从 CPU 后端异步获取张量数据
static void ggml_backend_cpu_get_tensor_async(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 检查读取数据是否超出张量边界
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor read out of bounds");
    // 检查张量是否已分配内存
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");

    // 将数据从源地址复制到目标地址
    memcpy(data, (const char *)tensor->data + offset, size);

    // 未使用的参数，避免编译器警告
    UNUSED(backend);
}

// 同步 CPU 后端
static void ggml_backend_cpu_synchronize(ggml_backend_t backend) {
    // 未使用的参数，避免编译器警告
    UNUSED(backend);
}

// 从源张量复制数据到目标张量
static void ggml_backend_cpu_cpy_tensor_from(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 获取源张量数据并复制到目标张量
    ggml_backend_tensor_get(src, dst->data, 0, ggml_nbytes(src));

    // 未使用的参数，避免编译器警告
    UNUSED(backend);
}

// 从源张量复制数据到目标张量
static void ggml_backend_cpu_cpy_tensor_to(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 设置源张量数据到目标张量
    ggml_backend_tensor_set(dst, src->data, 0, ggml_nbytes(src));

    // 未使用的参数，避免编译器警告
    UNUSED(backend);
}

// CPU 后端计划结构体
struct ggml_backend_plan_cpu {
    struct ggml_cplan cplan;
    struct ggml_cgraph cgraph;
};

// 创建 CPU 后端图计划
static ggml_backend_graph_plan_t ggml_backend_cpu_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 获取 CPU 后端上下文
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    // 分配 CPU 计划结构体内存
    struct ggml_backend_plan_cpu * cpu_plan = malloc(sizeof(struct ggml_backend_plan_cpu));

    // 初始化 CPU 计划结构体
    cpu_plan->cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);
    cpu_plan->cgraph = *cgraph;
    # 如果 CPU 计划中的工作大小大于 0
    if (cpu_plan->cplan.work_size > 0) {
        # 分配内存以存储工作数据
        cpu_plan->cplan.work_data = malloc(cpu_plan->cplan.work_size);
    }

    # 返回 CPU 计划
    return cpu_plan;
# 释放 CPU 图计划的内存
static void ggml_backend_cpu_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 将图计划转换为 CPU 图计划结构体
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    # 释放工作数据内存
    free(cpu_plan->cplan.work_data);
    # 释放 CPU 图计划结构体内存
    free(cpu_plan);

    # 未使用的参数
    UNUSED(backend);
}

# 计算 CPU 图计划
static void ggml_backend_cpu_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 将图计划转换为 CPU 图计划结构体
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    # 执行图计算
    ggml_graph_compute(&cpu_plan->cgraph, &cpu_plan->cplan);

    # 未使用的参数
    UNUSED(backend);
}

# 计算 CPU 图
static void ggml_backend_cpu_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    # 获取 CPU 上下文
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    # 根据图计划创建计划
    struct ggml_cplan cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);

    # 如果工作数据大小超过当前工作数据大小，则重新分配内存
    if (cpu_ctx->work_size < cplan.work_size) {
        # TODO: 可能更快地释放并使用 malloc 来避免复制
        cpu_ctx->work_data = realloc(cpu_ctx->work_data, cplan.work_size);
        cpu_ctx->work_size = cplan.work_size;
    }

    # 将计划的工作数据指向 CPU 上下文的工作数据
    cplan.work_data = cpu_ctx->work_data;

    # 执行图计算
    ggml_graph_compute(cgraph, &cplan);
}

# 判断 CPU 是否支持操作
static bool ggml_backend_cpu_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    # 总是返回 true
    return true;
    # 未使用的参数
    UNUSED(backend);
    UNUSED(op);
}

# CPU 后端接口
static struct ggml_backend_i cpu_backend_i = {
    /* .get_name            = */ ggml_backend_cpu_name,
    /* .free                = */ ggml_backend_cpu_free,
    /* .alloc_buffer        = */ ggml_backend_cpu_alloc_buffer,
    /* .get_alignment       = */ ggml_backend_cpu_get_alignment,
    /* .set_tensor_async    = */ ggml_backend_cpu_set_tensor_async,
    /* .get_tensor_async    = */ ggml_backend_cpu_get_tensor_async,
    /* .synchronize         = */ ggml_backend_cpu_synchronize,
    /* .cpy_tensor_from     = */ ggml_backend_cpu_cpy_tensor_from,
    /* .cpy_tensor_to       = */ ggml_backend_cpu_cpy_tensor_to,
    /* .graph_plan_create   = */ ggml_backend_cpu_graph_plan_create,
    // 指向 CPU 后端的图计划创建函数
    /* .graph_plan_free     = */ ggml_backend_cpu_graph_plan_free,
    // 指向 CPU 后端的图计划释放函数
    /* .graph_plan_compute  = */ ggml_backend_cpu_graph_plan_compute,
    // 指向 CPU 后端的图计划计算函数
    /* .graph_compute       = */ ggml_backend_cpu_graph_compute,
    // 指向 CPU 后端的图计算函数
    /* .supports_op         = */ ggml_backend_cpu_supports_op,
    // 指向 CPU 后端的操作支持函数
// 初始化 CPU 后端，分配内存并设置默认值
ggml_backend_t ggml_backend_cpu_init(void) {
    // 分配 CPU 后端上下文内存
    struct ggml_backend_cpu_context * ctx = malloc(sizeof(struct ggml_backend_cpu_context));

    // 设置默认线程数
    ctx->n_threads = GGML_DEFAULT_N_THREADS;
    ctx->work_data = NULL;
    ctx->work_size = 0;

    // 分配 CPU 后端内存
    ggml_backend_t cpu_backend = malloc(sizeof(struct ggml_backend));

    // 初始化 CPU 后端结构体
    *cpu_backend = (struct ggml_backend) {
        /* .interface = */ cpu_backend_i,
        /* .context   = */ ctx
    };
    return cpu_backend;
}

// 检查给定后端是否为 CPU 后端
bool ggml_backend_is_cpu(ggml_backend_t backend) {
    return backend->iface.get_name == ggml_backend_cpu_name;
}

// 设置 CPU 后端的线程数
void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads) {
    GGML_ASSERT(ggml_backend_is_cpu(backend_cpu));

    // 获取 CPU 后端上下文
    struct ggml_backend_cpu_context * ctx = (struct ggml_backend_cpu_context *)backend_cpu->context;
    // 设置线程数
    ctx->n_threads = n_threads;
}

// 从指针创建 CPU 后端缓冲区
ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(ggml_backend_t backend_cpu, void * ptr, size_t size) {
    return ggml_backend_buffer_init(backend_cpu, cpu_backend_buffer_i_from_ptr, ptr, size);
}

// 调度器

// 定义最大后端数、最大分割数和最大分割输入数
#define GGML_MAX_BACKENDS 4
#define GGML_MAX_SPLITS 256
#define GGML_MAX_SPLIT_INPUTS 16

// 定义分割结构体
struct ggml_backend_sched_split {
    ggml_tallocr_t tallocr;
    int i_start;
    int i_end;
    struct ggml_tensor * inputs[GGML_MAX_SPLIT_INPUTS];
    int n_inputs;
    struct ggml_cgraph * graph;
};

// 定义调度器结构体
struct ggml_backend_sched {
    int n_backends;
    ggml_backend_t backends[GGML_MAX_BACKENDS];
    ggml_tallocr_t  tallocs[GGML_MAX_BACKENDS];

    ggml_gallocr_t galloc;

    struct ggml_hash_set    hash_set;
    ggml_tallocr_t *        node_talloc;                     // [hash_set.size]
    struct ggml_tensor * (* node_copies)[GGML_MAX_BACKENDS]; // [hash_set.size][GGML_MAX_BACKENDS]

    struct ggml_cgraph * graph;
    struct ggml_backend_sched_split splits[GGML_MAX_SPLITS];
    int n_splits;

    struct ggml_context * ctx;

    // 将 context_buffer 对齐到 GGML_MEM_ALIGN
    #ifdef _MSC_VER
    __declspec(align(GGML_MEM_ALIGN))
    #else
    #endif
    // 定义一个字符数组，用于存储一定数量的结构体 ggml_tensor 和 ggml_cgraph 的数据
    // 数组大小为 GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS*sizeof(struct ggml_tensor) + GGML_MAX_SPLITS*sizeof(struct ggml_cgraph)
    char context_buffer[GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS*sizeof(struct ggml_tensor) + GGML_MAX_SPLITS*sizeof(struct ggml_cgraph)];
// 定义一个宏，用于根据节点查找或插入哈希表中的值
#define hash_id(node) ggml_hash_find_or_insert(sched->hash_set, node)
// 定义一个宏，用于根据节点分配内存
#define node_allocr(node) sched->node_talloc[hash_id(node)]

// 检查操作是否为视图操作
static bool ggml_is_view_op(enum ggml_op op) {
    return op == GGML_OP_VIEW || op == GGML_OP_RESHAPE || op == GGML_OP_PERMUTE || op == GGML_OP_TRANSPOSE;
}

// 返回后端的优先级，数值越小表示优先级越高
static int sched_backend_prio(ggml_backend_sched_t sched, ggml_backend_t backend) {
    for (int i = 0; i < sched->n_backends; i++) {
        if (sched->backends[i] == backend) {
            return i;
        }
    }
    return INT_MAX;
}

// 返回分配器的优先级
static int sched_allocr_prio(ggml_backend_sched_t sched, ggml_tallocr_t allocr) {
    for (int i = 0; i < sched->n_backends; i++) {
        if (sched->tallocs[i] == allocr) {
            return i;
        }
    }
    return INT_MAX;
}

// 根据当前位置返回应该用于节点的后端
char causes[GGML_DEFAULT_GRAPH_SIZE*4 + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS][128]; // 调试用，可移除
static ggml_backend_t sched_backend_from_cur(ggml_backend_sched_t sched, struct ggml_tensor * node) {
    // 如果目标张量已经在缓冲区中分配了空间，我们必须假设保留在那里是至关重要的，例如 kv 缓存更新
    // 注意这不允许回退到 CPU，需要将输出张量添加到拆分中，以将数据复制回原始后端
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

    // 源
    int cur_prio = INT_MAX;
    size_t cur_size = 0;
    // 遍历节点的输入源
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        // 获取当前输入源
        const struct ggml_tensor * src = node->src[i];
        // 如果输入源为空，跳出循环
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
            // 如果输入源的优先级小于当前优先级且输入源的大小大于等于当前大小
            if (src_prio < cur_prio && src_size >= cur_size) {
                // 更新当前优先级、大小和后端
                cur_prio = src_prio;
                cur_size = src_size;
                cur_backend = src_backend;
                // 将导致变化的原因记录到causes数组中
                sprintf(causes[hash_id(node)], "1.src%d", i);
            }
        }
    }
    // 返回当前后端
    return cur_backend;
# 格式化输出文件大小，返回字符串指针
static char * fmt_size(size_t size) {
    # 静态字符数组用于存储格式化后的文件大小信息
    static char buffer[128];
    # 根据文件大小选择不同的格式化方式
    if (size >= 1024*1024) {
        # 如果文件大小大于等于1MB，以MB为单位格式化
        sprintf(buffer, "%zuM", size/1024/1024);
    } else {
        # 如果文件大小小于1MB，以KB为单位格式化
        sprintf(buffer, "%zuK", size/1024);
    }
    # 返回格式化后的文件大小信息
    return buffer;
}

# 打印调度分配情况
static void sched_print_assignments(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    # 当前分割数初始化为0
    int cur_split = 0;
    // 遍历图中的每个节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 检查是否需要在当前节点处进行分割
        if (cur_split < sched->n_splits && i == sched->splits[cur_split].i_start) {
            // 获取分割节点的后端信息
            ggml_backend_t split_backend = ggml_tallocr_get_buffer(sched->splits[cur_split].tallocr)->backend;
            // 输出分割节点信息
            fprintf(stderr, "\n## SPLIT #%d: %s # %d inputs: ", cur_split, ggml_backend_name(split_backend), sched->splits[cur_split].n_inputs);
            // 遍历分割节点的输入
            for (int j = 0; j < sched->splits[cur_split].n_inputs; j++) {
                // 输出分割节点的输入信息
                fprintf(stderr, "[%s (%5.5s)] ", sched->splits[cur_split].inputs[j]->name, fmt_size(ggml_nbytes(sched->splits[cur_split].inputs[j])));
            }
            fprintf(stderr, "\n");
            // 更新当前分割节点索引
            cur_split++;
        }
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果当前节点是视图操作，则跳过
        if (ggml_is_view_op(node->op)) {
            continue;
        }
        // 获取当前节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 获取当前节点的后端信息
        ggml_backend_t node_backend = node_allocr ? ggml_tallocr_get_buffer(node_allocr)->backend : NULL;
        // 输出当前节点信息
        fprintf(stderr, "node #%3d (%10.10s): %20.20s (%4.4s) [%4.4s %8.8s]:", i, ggml_op_name(node->op), node->name, fmt_size(ggml_nbytes(node)), node_allocr ? ggml_backend_name(node_backend) : "NULL", causes[hash_id(node)]);
        // 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * src = node->src[j];
            // 如果源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取源节点的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 获取源节点的后端信息
            ggml_backend_t src_backend = src_allocr ? ggml_tallocr_get_buffer(src_allocr)->backend : NULL;
            // 输出源节点信息
            fprintf(stderr, " %20.20s (%4.4s) [%4.4s %8.8s]", src->name, fmt_size(ggml_nbytes(src)), src_backend ? ggml_backend_name(src_backend) : "NULL", causes[hash_id(src)]);
        }
        fprintf(stderr, "\n");
    }
// 创建一个具有相同内存布局的张量副本
static struct ggml_tensor * ggml_dup_tensor_layout(struct ggml_context * ctx, const struct ggml_tensor * tensor) {
    // 复制张量
    struct ggml_tensor * dup = ggml_dup_tensor(ctx, tensor);
    // 复制张量的维度信息
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        dup->nb[i] = tensor->nb[i];
    }
    return dup;
}

// 为操作分配后端并将图拆分为可以在相同后端上计算的子图
// TODO: 合并通道
static void sched_split_graph(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    // 重置状态
    size_t hash_size = sched->hash_set.size;
    // 初始化哈希表
    memset(sched->hash_set.keys, 0, sizeof(sched->hash_set.keys[0]) * hash_size);
    memset(sched->node_talloc,   0, sizeof(sched->node_talloc[0])   * hash_size);
    memset(sched->node_copies,   0, sizeof(sched->node_copies[0])   * hash_size);
    sched->n_splits = 0;

    // 初始化参数
    struct ggml_init_params params = {
        /*.mem_size =   */ sizeof(sched->context_buffer),
        /*.mem_buffer = */ sched->context_buffer,
        /*.no_alloc =   */ true
    };

    // 如果上下文不为空，则释放上下文
    if (sched->ctx != NULL) {
        ggml_free(sched->ctx);
    }

    // 初始化上下文
    sched->ctx = ggml_init(params);

    // 第一步：为具有分配输入的操作分配后端
    for (int i = 0; i < graph->n_leafs; i++) {
        struct ggml_tensor * leaf = graph->leafs[i];
        // 如果节点已经分配，则跳过
        if (node_allocr(leaf) != NULL) {
            continue;
        }
        // 获取叶子节点的后端
        ggml_backend_t leaf_backend = ggml_get_backend(leaf);
        // 如果叶子节点没有后端并且视图源不为空，则获取视图源的后端
        if (leaf_backend == NULL && leaf->view_src != NULL) {
            leaf_backend = ggml_get_backend(leaf->view_src);
        }
        // 如果叶子节点有后端，则为节点分配内存
        if (leaf_backend != NULL) {
            node_allocr(leaf) = ggml_backend_sched_get_tallocr(sched, leaf_backend);
        }
    }
}
    // 遍历图中的每个节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 检查节点是否已经有分配的资源，如果有则跳过
        if (node_allocr(node) != NULL) {
            // 不覆盖用户分配的资源
            continue;
        }
        // 获取当前节点的调度后端
        ggml_backend_t node_backend = sched_backend_from_cur(sched, node);
        // 如果节点有调度后端，则分配资源
        if (node_backend != NULL) {
            node_allocr(node) = ggml_backend_sched_get_tallocr(sched, node_backend);
        }
    }
    //printf("PASS 1 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 2: assign backends to ops from current assignments
    // TODO:
    //  - reuse sched_backend_from_cur
    // 遍历图中的每个节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取当前节点的资源分配
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 如果节点没有资源分配
        if (node_allocr == NULL) {
            // 初始化当前优先级和大小
            int    cur_prio = INT_MAX;
            size_t cur_size = 0;
            // 遍历当前节点的所有输入节点
            for (int j = 0; j < GGML_MAX_SRC; j++) {
                struct ggml_tensor * src = node->src[j];
                // 如果输入节点为空，则跳出循环
                if (src == NULL) {
                    break;
                }
                // 获取输入节点的资源分配
                ggml_tallocr_t src_allocr = node_allocr(src);
                // 如果输入节点有资源分配
                if (src_allocr != NULL) {
                    // 获取输入节点的优先级和大小
                    int    src_prio = sched_allocr_prio(sched, src_allocr);
                    size_t src_size = ggml_nbytes(src);
                    // 如果输入节点的优先级比当前优先级小且大小大于等于当前大小，则更新资源分配
                    if (src_prio < cur_prio && src_size >= cur_size) {
                        cur_prio = src_prio;
                        cur_size = src_size;
                        node_allocr = src_allocr;
                        // 记录导致资源分配变化的原因
                        sprintf(causes[hash_id(node)], "2.src%d", j);
                    }
                }
            }
            // 如果找到合适的资源分配，则分配给当前节点
            if (node_allocr != NULL) {
                node_allocr(node) = node_allocr;
            }
        }
    }
    //printf("PASS 2 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 3: assign backends to remaining src from dst (should only be leafs)
    // 遍历图中的每个节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 为当前节点创建内存分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前源节点
            struct ggml_tensor * src = node->src[j];
            // 如果当前源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 为当前源节点创建内存分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果当前源节点的内存分配器为空，则将当前节点的内存分配器赋给当前源节点
            if (src_allocr == NULL) {
                node_allocr(src) = node_allocr;
            }
        }
    }
    //printf("PASS 3 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 4: split graph, find tensors that need to be copied
    // TODO:
    //  - when switching from a less preferred backend to a more preferred backend, check if it is possible to move the switch to an earlier point for the same cost
    // find first backend
    // 初始化当前分割点为0
    int cur_split = 0;
    // 遍历图中的每个节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果当前节点的视图源节点为空
        if (node->view_src == NULL) {
            // 将当前节点的内存分配器赋给第一个分割点
            sched->splits[0].tallocr = node_allocr(node);
            break;
        }
    }
    // 设置第一个分割点的起始索引和输入数量
    sched->splits[0].i_start = 0;
    sched->splits[0].n_inputs = 0;
    // 将第一个分割点的输入数组清零
    memset(sched->splits[0].inputs, 0, sizeof(sched->splits[0].inputs)); //HACK
    // 获取当前分割点的内存分配器
    ggml_tallocr_t cur_allocr = sched->splits[0].tallocr;
    // 获取当前后端的ID
    size_t cur_backend_id = sched_allocr_prio(sched, cur_allocr);
    }
    // 设置当前分割点的结束索引为图中节点的数量
    sched->splits[cur_split].i_end = graph->n_nodes;
    // 设置分割点数量为当前分割点加1
    sched->n_splits = cur_split + 1;

    //fprintf(stderr, "PASS 4 ASSIGNMENTS\n"); sched_print_assignments(sched, graph); fflush(stdout);
#if 1
    // 如果条件为真，则执行以下代码块
    // 对于图中的每个节点，检查其所有源节点是否具有与该节点相同的后端
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取当前节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 如果节点没有分配器，则输出错误信息
        if (node_allocr == NULL) {
            fprintf(stderr, "!!!!!!! %s has no backend\n", node->name);
        }
        // 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前源节点
            struct ggml_tensor * src = node->src[j];
            // 如果源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取当前源节点的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果源节点的分配器与当前节点的分配器不同，则输出错误信息
            if (src_allocr != node_allocr /* && src_backend != NULL */) { // 忽略目前的空值
                fprintf(stderr, "!!!! %s has backend %s, src %d (%s) has backend %s\n",
                    node->name, node_allocr ? ggml_backend_name(ggml_tallocr_get_buffer(node_allocr)->backend) : "NULL",
                    j, src->name, src_allocr ? ggml_backend_name(ggml_tallocr_get_buffer(src_allocr)->backend) : "NULL");
            }
        }
    }
#endif

    // 为每个拆分创建图的副本
    // FIXME: 避免这种复制，以某种其他方式将拆分输入传递给 ggml_gallocr_alloc_graph_n
    // 创建一个新的自定义图，包括原图节点数和拆分数*最大拆分输入数
    struct ggml_cgraph * graph_copy = ggml_new_graph_custom(sched->ctx, graph->n_nodes + sched->n_splits*GGML_MAX_SPLIT_INPUTS, false);
    // 遍历调度器中的每个分割任务
    for (int i = 0; i < sched->n_splits; i++) {
        // 获取当前分割任务的指针
        struct ggml_backend_sched_split * split = &sched->splits[i];
        // 为当前分割任务创建一个图形视图
        split->graph = ggml_graph_view(sched->ctx, graph, split->i_start, split->i_end);

        // 将输入添加到图形副本中，以便它们在分割开始时由 ggml-alloc 分配
        for (int j = 0; j < split->n_inputs; j++) {
            // 获取当前输入张量的指针
            struct ggml_tensor * input = split->inputs[j];
            // 获取当前输入张量的副本
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(input)][sched_allocr_prio(sched, split->tallocr)];
            // 将当前输入张量设置为副本的源
            input_cpy->src[0] = input;
            // 将输入张量副本添加到图形副本的节点列表中
            graph_copy->nodes[graph_copy->n_nodes++] = input_cpy;
        }

        // 将原始图形中的节点添加到图形副本中
        for (int j = split->i_start; j < split->i_end; j++) {
            graph_copy->nodes[graph_copy->n_nodes++] = graph->nodes[j];
        }
    }
    // 将图形副本设置为调度器的图形
    sched->graph = graph_copy;
# 分配计划中的任务到各个后端
static void sched_alloc_splits(ggml_backend_sched_t sched) {
    # 调用函数分配图形节点到内存
    ggml_gallocr_alloc_graph_n(
        sched->galloc,
        sched->graph,
        sched->hash_set,
        sched->node_talloc);
}

# 计算计划中的任务
static void sched_compute_splits(ggml_backend_sched_t sched) {
    # 初始化用于记录拷贝时间和计算时间的数组
    uint64_t copy_us[GGML_MAX_BACKENDS] = {0};
    uint64_t compute_us[GGML_MAX_BACKENDS] = {0};

    # 获取计划中的任务拆分信息
    struct ggml_backend_sched_split * splits = sched->splits;
}
    // 遍历调度器中的分割任务
    for (int i = 0; i < sched->n_splits; i++) {
        // 获取当前分割任务的指针
        struct ggml_backend_sched_split * split = &splits[i];
        // 获取当前分割任务对应的后端
        ggml_backend_t split_backend = ggml_tallocr_get_buffer(split->tallocr)->backend;
        // 获取当前分割任务对应的后端的 ID
        int split_backend_id = sched_backend_prio(sched, split_backend);

        // 将输入张量复制到分割任务对应的后端
        uint64_t copy_start_us = ggml_time_us();
        // 遍历当前分割任务的所有输入张量
        for (int j = 0; j < split->n_inputs; j++) {
            // 获取当前输入张量的副本
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(split->inputs[j])][sched_backend_prio(sched, split_backend)];
            // 检查当前输入张量是否有缓冲区
            if (split->inputs[j]->buffer == NULL) {
                // 如果当前输入张量既没有缓冲区也没有视图源，则输出错误信息并退出程序
                if (split->inputs[j]->view_src == NULL) {
                    fprintf(stderr, "input %s has no buffer and no view_src\n", split->inputs[j]->name);
                    exit(1);
                }
                // 获取当前输入张量的视图源，并初始化视图
                struct ggml_tensor * view = split->inputs[j];
                view->backend = view->view_src->backend;
                view->buffer  = view->view_src->buffer;
                view->data    = (char *)view->view_src->data + view->view_offs;
                ggml_backend_buffer_init_tensor(ggml_backend_sched_get_buffer(sched, view->buffer->backend), view);
            }
            // 检查当前输入张量的副本是否有缓冲区
            if (input_cpy->buffer == NULL) {
                // 如果当前输入张量的副本没有缓冲区，则输出错误信息并退出程序
                fprintf(stderr, "input_cpy %s has no buffer\n", input_cpy->name);
                exit(1);
            }
            // 断言当前输入张量的后端与副本的后端不同
            GGML_ASSERT(split->inputs[j]->buffer->backend != input_cpy->buffer->backend);
            // 断言当前输入张量的副本的后端与分割任务的后端相同
            GGML_ASSERT(input_cpy->buffer->backend == split_backend);
            // 复制当前输入张量到副本
            ggml_backend_tensor_copy(split->inputs[j], input_cpy);
        }
        // 计算复制操作的时间
        int64_t copy_end_us = ggml_time_us();
        copy_us[split_backend_id] += copy_end_us - copy_start_us;
#if 0
        // 定义一个字符数组用于存储拆分后的文件名
        char split_filename[GGML_MAX_NAME];
        // 根据给定的格式化字符串生成拆分后的文件名
        snprintf(split_filename, GGML_MAX_NAME, "split_%i_%s.dot", i, ggml_backend_name(split_backend));
        // 将拆分后的图形以 DOT 格式输出到文件中
        ggml_graph_dump_dot(split->graph, NULL, split_filename);
#endif

        // 记录计算开始时间
        uint64_t compute_start_us = ggml_time_us();
        // 使用指定后端计算拆分后的图形
        ggml_backend_graph_compute(split_backend, split->graph);
        // 计算结束时间
        uint64_t compute_end_us = ggml_time_us();
        // 计算本次计算所用时间并加入到相应后端的计算时间中
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
    // 重置每个后端的内存分配器
    for (int i = 0; i < sched->n_backends; i++) {
        ggml_tallocr_reset(sched->tallocs[i]);
    }
}

// 创建新的后端调度器
ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, int n_backends) {
    // 确保后端数量不超过最大限制
    GGML_ASSERT(n_backends <= GGML_MAX_BACKENDS);

    // 分配内存并初始化后端调度器
    struct ggml_backend_sched * sched = malloc(sizeof(struct ggml_backend_sched));
    memset(sched, 0, sizeof(struct ggml_backend_sched));

    // 输出后端调度器的大小
    fprintf(stderr, "ggml_backend_sched size: %lu KB\n", sizeof(struct ggml_backend_sched)/1024);

    // 设置后端数量和后端数组
    sched->n_backends = n_backends;
    for (int i = 0; i < n_backends; i++) {
        sched->backends[i] = backends[i];
    }

    // 创建全局内存分配器
    sched->galloc = ggml_gallocr_new();

    // 初始化每个后端的内存分配器
    for (int i = 0; i < n_backends; i++) {
        sched->tallocs[i] = ggml_tallocr_new_measure_from_backend(backends[i]);
    }

    return sched;
}

// 释放后端调度器
void ggml_backend_sched_free(ggml_backend_sched_t sched) {
    if (sched == NULL) {
        return;
    }
    // 遍历调度器中的后端数量
    for (int i = 0; i < sched->n_backends; i++) {
        // 释放调度器中第 i 个后端的内存
        ggml_tallocr_free(sched->tallocs[i]);
    }
    // 释放调度器中的全局内存
    ggml_gallocr_free(sched->galloc);
    // 释放调度器中哈希集合的键数组内存
    free(sched->hash_set.keys);
    // 释放调度器中节点内存的内存
    free(sched->node_talloc);
    // 释放调度器中节点拷贝的内存
    free(sched->node_copies);
    // 释放调度器本身的内存
    free(sched);
// 初始化测量调度器，传入调度器和测量图
void ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph) {
    // 初始化哈希表大小
    size_t hash_size = measure_graph->visited_hash_table.size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS;
    // 设置哈希表大小
    sched->hash_set.size = hash_size;
    // 分配哈希表键内存
    sched->hash_set.keys = malloc(sizeof(sched->hash_set.keys[0]) * hash_size);
    // 分配节点内存
    sched->node_talloc   = malloc(sizeof(sched->node_talloc[0])   * hash_size);
    // 分配节点拷贝内存
    sched->node_copies   = malloc(sizeof(sched->node_copies[0])   * hash_size);

    // 分割图形
    sched_split_graph(sched, measure_graph);
    // 分配拆分
    sched_alloc_splits(sched);

    // 为每个后端分配缓冲区并重置分配器
    for (int i = 0; i < sched->n_backends; i++) {
        // 获取最大尺寸
        size_t size = ggml_tallocr_max_size(sched->tallocs[i]);
        // 释放分配器
        ggml_tallocr_free(sched->tallocs[i]);
        // 从后端创建新的分配器
        sched->tallocs[i] = ggml_tallocr_new_from_backend(sched->backends[i], size);
    }

    // 重置调度器
    sched_reset(sched);
}

// 计算图形
void ggml_backend_sched_graph_compute(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    // 断言哈希表大小大于等于图形的哈希表大小加上最大拆分数乘以最大拆分输入数
    GGML_ASSERT(sched->hash_set.size >= graph->visited_hash_table.size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS);

    // 分割图形
    sched_split_graph(sched, graph);
    // 分配拆分
    sched_alloc_splits(sched);
    // 计算拆分
    sched_compute_splits(sched);
    // 重置调度器
    sched_reset(sched);
}

// 获取分配器
ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend) {
    // 获取后端索引
    int backend_index = sched_backend_prio(sched, backend);
    // 返回对应后端的分配器
    return sched->tallocs[backend_index];
}

// 获取缓冲区
ggml_backend_buffer_t ggml_backend_sched_get_buffer(ggml_backend_sched_t sched, ggml_backend_t backend) {
    // 获取后端索引
    int backend_index = sched_backend_prio(sched, backend);
    // 返回对应后端的缓冲区
    return ggml_tallocr_get_buffer(sched->tallocs[backend_index]);
}

// 设置节点后端
void ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend) {
    // 获取后端索引
    int backend_index = sched_backend_prio(sched, backend);
    // 断言后端索引在有效范围内
    GGML_ASSERT(backend_index >= 0 && backend_index < sched->n_backends);
}
    # 为节点分配资源，使用调度器中对应后端索引的资源分配器
    node_allocr(node) = sched->tallocs[backend_index];
# 闭合之前的代码块
```