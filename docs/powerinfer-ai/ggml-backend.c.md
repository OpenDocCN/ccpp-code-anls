# `PowerInfer\ggml-backend.c`

```cpp
// 包含所需的头文件
#include "ggml-backend-impl.h"
#include "ggml-alloc.h"
#include "ggml-impl.h"

// 包含所需的标准库头文件
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

// backend buffer

// 初始化后端缓冲区
ggml_backend_buffer_t ggml_backend_buffer_init(
        struct ggml_backend                  * backend,
        struct ggml_backend_buffer_i           iface,
               ggml_backend_buffer_context_t   context,
               size_t                          size) {
    // 分配内存以存储后端缓冲区
    ggml_backend_buffer_t buffer = malloc(sizeof(struct ggml_backend_buffer));

    // 断言接口中的 get_base 函数不为空
    GGML_ASSERT(iface.get_base != NULL);

    // 将传入的参数赋值给后端缓冲区对象
    (*buffer) = (struct ggml_backend_buffer) {
        /* .interface = */ iface,
        /* .backend   = */ backend,
        /* .context   = */ context,
        /* .size      = */ size,
    };

    return buffer;
}

// 释放后端缓冲区
void ggml_backend_buffer_free(ggml_backend_buffer_t buffer) {
    if (buffer == NULL) {
        return;
    }

    // 如果接口中的 free_buffer 函数不为空，则调用该函数释放缓冲区
    if (buffer->iface.free_buffer != NULL) {
        buffer->iface.free_buffer(buffer);
    }
    // 释放后端缓冲区的内存
    free(buffer);
}

// 获取后端缓冲区的对齐方式
size_t ggml_backend_buffer_get_alignment(ggml_backend_buffer_t buffer) {
    return ggml_backend_get_alignment(buffer->backend);
}

// 获取后端缓冲区的大小
size_t ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer) {
    return buffer->size;
}

// 获取后端缓冲区的基地址
void * ggml_backend_buffer_get_base(ggml_backend_buffer_t buffer) {
    void * base = buffer->iface.get_base(buffer);

    // 断言后端缓冲区的基地址不为空
    GGML_ASSERT(base != NULL && "backend buffer base cannot be NULL");

    return base;
}

// 获取后端缓冲区为给定张量分配的大小
size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // get_alloc_size 是可选的，如果存在则调用该函数获取分配大小，否则默认调用 ggml_nbytes 函数
    if (buffer->iface.get_alloc_size) {
        return buffer->iface.get_alloc_size(buffer, tensor);
    }
    return ggml_nbytes(tensor);
}

// 初始化后端缓冲区的张量
void ggml_backend_buffer_init_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // init_tensor 是可选的
}
    # 如果缓冲区接口中有初始化张量的函数
    if (buffer->iface.init_tensor) {
        # 调用缓冲区接口中的初始化张量函数，传入缓冲区和张量作为参数
        buffer->iface.init_tensor(buffer, tensor);
    }
// 释放张量的缓冲区，如果存在释放张量的函数，则调用该函数
void ggml_backend_buffer_free_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // 如果存在释放张量的函数，则调用该函数
    if (buffer->iface.free_tensor) {
        buffer->iface.free_tensor(buffer, tensor);
    }
}

// 获取张量的后端
ggml_backend_t ggml_get_backend(const struct ggml_tensor * tensor) {
    // 如果张量的缓冲区存在，则返回其后端，否则返回 NULL
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

// 释放后端
void ggml_backend_free(ggml_backend_t backend) {
    // 如果后端为 NULL，则直接返回，否则调用后端的释放函数
    if (backend == NULL) {
        return;
    }
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
    // 调用后端的异步设置张量数据函数
    ggml_get_backend(tensor)->iface.set_tensor_async(ggml_get_backend(tensor), tensor, data, offset, size);
}

// 异步获取张量数据
void ggml_backend_tensor_get_async(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 调用后端的异步获取张量数据函数
    ggml_get_backend(tensor)->iface.get_tensor_async(ggml_get_backend(tensor), tensor, data, offset, size);
}

// 设置张量数据
void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    // 获取张量的后端
    ggml_backend_t backend = ggml_get_backend(tensor);

    // 断言张量的数据不为 NULL，并输出错误信息
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    // 断言后端不为 NULL，并输出错误信息
    GGML_ASSERT(backend != NULL && "tensor backend not set");

    // 调用后端的异步设置张量数据函数
    backend->iface.set_tensor_async(backend, tensor, data, offset, size);
    // 调用后端的同步函数
    backend->iface.synchronize(backend);
}

// 获取张量数据
void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 获取张量的后端
    ggml_backend_t backend = ggml_get_backend(tensor);
    # 断言张量数据已经分配，如果未分配则输出错误信息
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    # 断言后端已经设置，如果未设置则输出错误信息
    GGML_ASSERT(backend != NULL && "tensor backend not set");

    # 通过后端接口异步获取张量数据
    backend->iface.get_tensor_async(backend, tensor, data, offset, size);
    # 同步后端，确保获取数据完成
    backend->iface.synchronize(backend);
// 同步后端
void ggml_backend_synchronize(ggml_backend_t backend) {
    // 调用后端接口的同步函数
    backend->iface.synchronize(backend);
}

// 创建图计划
ggml_backend_graph_plan_t ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 调用后端接口的图计划创建函数
    return backend->iface.graph_plan_create(backend, cgraph);
}

// 释放图计划
void ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    // 调用后端接口的图计划释放函数
    backend->iface.graph_plan_free(backend, plan);
}

// 计算图计划
void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    // 调用后端接口的图计划计算函数
    backend->iface.graph_plan_compute(backend, plan);
}

// 计算图
void ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 调用后端接口的图计算函数
    backend->iface.graph_compute(backend, cgraph);
}

// 判断后端是否支持操作
bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    // 调用后端接口的操作支持判断函数
    return backend->iface.supports_op(backend, op);
}

// 后端复制
// 判断两个张量是否具有相同的布局
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
    // 检查源张量和目标张量的布局是否相同
    GGML_ASSERT(ggml_are_same_layout(src, dst) && "cannot copy tensors with different layouts");
}
    // 打印消息，将张量从源张量拷贝到目标张量，包括张量名称、源后端名称、目标后端名称、张量字节数
    if (src == dst) {
        // 如果源张量和目标张量相同，则直接返回，不进行拷贝操作
        return;
    }

    // TODO: 允许后端支持从同一后端拷贝到同一后端

    // 如果目标后端支持从其他后端拷贝张量数据
    if (ggml_get_backend(dst)->iface.cpy_tensor_from != NULL) {
        // 调用目标后端的拷贝函数，将数据从源张量拷贝到目标张量
        ggml_get_backend(dst)->iface.cpy_tensor_from(ggml_get_backend(dst)->context, src, dst);
    } else if (ggml_get_backend(src)->iface.cpy_tensor_to != NULL) {
        // 如果源后端支持向其他后端拷贝张量数据
        ggml_get_backend(src)->iface.cpy_tensor_to(ggml_get_backend(src)->context, src, dst);
    } else {
        // 如果源后端和目标后端都不支持拷贝操作
        // 在调试模式下打印警告消息
        #ifndef NDEBUG
        fprintf(stderr, "ggml_backend_tensor_copy: neither cpy_tensor_from nor cpy_tensor_to are implemented for backends %s and %s, falling back to get/set\n", ggml_backend_name(src->buffer->backend), ggml_backend_name(dst->buffer->backend));
        #endif
        // 分配内存空间，拷贝源张量数据到内存中，然后再将数据设置到目标张量中
        size_t nbytes = ggml_nbytes(src);
        void * data = malloc(nbytes);
        ggml_backend_tensor_get(src, data, 0, nbytes);
        ggml_backend_tensor_set(dst, data, 0, nbytes);
        free(data);
    }
// 定义了一个结构体，用于存储后端 CPU 的上下文信息
struct ggml_backend_cpu_context {
    int n_threads; // CPU 线程数
    void * work_data; // 工作数据的指针
    size_t work_size; // 工作数据的大小
};

// 返回后端 CPU 的名称
static const char * ggml_backend_cpu_name(ggml_backend_t backend) {
    return "CPU"; // 返回字符串 "CPU"

    UNUSED(backend); // 未使用的参数
}

// 释放后端 CPU 的资源
static void ggml_backend_cpu_free(ggml_backend_t backend) {
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context; // 获取后端 CPU 上下文信息
    free(cpu_ctx->work_data); // 释放工作数据的内存
    free(cpu_ctx); // 释放后端 CPU 上下文信息的内存
    free(backend); // 释放后端 CPU 的内存
}

// 获取后端 CPU 缓冲区的基地址
static void * ggml_backend_cpu_buffer_get_base(ggml_backend_buffer_t buffer) {
    return (void *)buffer->context; // 返回缓冲区的上下文信息作为基地址
}

// 释放后端 CPU 缓冲区的内存
static void ggml_backend_cpu_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    free(buffer->context); // 释放缓冲区的上下文信息的内存
    UNUSED(buffer); // 未使用的参数
}

// 定义了后端 CPU 缓冲区的接口
static struct ggml_backend_buffer_i cpu_backend_buffer_i = {
    /* .free_buffer    = */ ggml_backend_cpu_buffer_free_buffer, // 释放缓冲区的内存
    /* .get_base       = */ ggml_backend_cpu_buffer_get_base, // 获取缓冲区的基地址
    /* .get_alloc_size = */ NULL, // 默认为 ggml_nbytes
    /* .init_tensor    = */ NULL, // 不需要初始化
    /* .free_tensor    = */ NULL, // 不需要清理
};

// 对于从指针获取的缓冲区，不需要调用 free
static struct ggml_backend_buffer_i cpu_backend_buffer_i_from_ptr = {
    /* .free_buffer    = */ NULL, // 指针不是缓冲区的所有者，因此不需要释放
    /* .get_base       = */ ggml_backend_cpu_buffer_get_base, // 获取缓冲区的基地址
    /* .get_alloc_size = */ NULL, // 默认为 ggml_nbytes
    /* .init_tensor    = */ NULL, // 不需要初始化
    /* .free_tensor    = */ NULL, // 不需要清理
};

// 定义了张量的对齐方式
static const size_t TENSOR_ALIGNMENT = 64; // 应该足够支持 AVX 512

// 分配后端 CPU 缓冲区的内存
static ggml_backend_buffer_t ggml_backend_cpu_alloc_buffer(ggml_backend_t backend, size_t size) {
    size += TENSOR_ALIGNMENT;   // 增加对齐的空间，因为 malloc 可能返回未对齐的地址
    void * data = malloc(size); // 分配内存空间

    GGML_ASSERT(data != NULL && "failed to allocate buffer"); // 断言内存分配成功

    return ggml_backend_buffer_init(backend, cpu_backend_buffer_i, data, size); // 初始化后端 CPU 缓冲区
}
# 获取 CPU 后端的对齐方式
static size_t ggml_backend_cpu_get_alignment(ggml_backend_t backend) {
    return TENSOR_ALIGNMENT;
    # 标记参数未使用，避免编译器警告
    UNUSED(backend);
}

# 设置异步张量数据到 CPU 后端
static void ggml_backend_cpu_set_tensor_async(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    # 检查写入数据是否超出张量边界
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor write out of bounds");
    # 检查张量是否已分配内存
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");

    # 将数据从源地址复制到张量数据的指定偏移位置
    memcpy((char *)tensor->data + offset, data, size);

    # 标记参数未使用，避免编译器警告
    UNUSED(backend);
}

# 从 CPU 后端异步获取张量数据
static void ggml_backend_cpu_get_tensor_async(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    # 检查读取数据是否超出张量边界
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor read out of bounds");
    # 检查张量是否已分配内存
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");

    # 将张量数据的指定偏移位置的数据复制到目标地址
    memcpy(data, (const char *)tensor->data + offset, size);

    # 标记参数未使用，避免编译器警告
    UNUSED(backend);
}

# 同步 CPU 后端
static void ggml_backend_cpu_synchronize(ggml_backend_t backend) {
    # 标记参数未使用，避免编译器警告
    UNUSED(backend);
}

# 从源张量复制数据到目标张量
static void ggml_backend_cpu_cpy_tensor_from(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    # 从源张量获取数据并复制到目标张量
    ggml_backend_tensor_get(src, dst->data, 0, ggml_nbytes(src));

    # 标记参数未使用，避免编译器警告
    UNUSED(backend);
}

# 从源张量复制数据到目标张量
static void ggml_backend_cpu_cpy_tensor_to(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    # 将源张量的数据设置到目标张量
    ggml_backend_tensor_set(dst, src->data, 0, ggml_nbytes(src));

    # 标记参数未使用，避免编译器警告
    UNUSED(backend);
}

# CPU 后端计划图创建
struct ggml_backend_plan_cpu {
    struct ggml_cplan cplan;
    struct ggml_cgraph cgraph;
};

static ggml_backend_graph_plan_t ggml_backend_cpu_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    # 获取 CPU 后端上下文
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    # 分配内存以存储 CPU 后端计划
    struct ggml_backend_plan_cpu * cpu_plan = malloc(sizeof(struct ggml_backend_plan_cpu));

    # 创建 CPU 后端计划
    cpu_plan->cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);
    cpu_plan->cgraph = *cgraph;
    # 如果 CPU 计划中的工作大小大于 0
    if (cpu_plan->cplan.work_size > 0) {
        # 分配内存空间，用于存储工作数据
        cpu_plan->cplan.work_data = malloc(cpu_plan->cplan.work_size);
    }
    # 返回 CPU 计划
    return cpu_plan;
# 释放 CPU 图计划占用的资源
static void ggml_backend_cpu_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 将传入的图计划转换为 CPU 图计划结构体
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;
    
    # 释放 CPU 图计划中的工作数据
    free(cpu_plan->cplan.work_data);
    # 释放 CPU 图计划结构体
    free(cpu_plan);

    # 标记未使用的参数
    UNUSED(backend);
}

# 计算 CPU 图计划
static void ggml_backend_cpu_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 将传入的图计划转换为 CPU 图计划结构体
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    # 调用图计算函数，传入 CPU 图计划中的图和计划
    ggml_graph_compute(&cpu_plan->cgraph, &cpu_plan->cplan);

    # 标记未使用的参数
    UNUSED(backend);
}

# 计算 CPU 图
static void ggml_backend_cpu_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    # 将传入的 backend 转换为 CPU 上下文结构体
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    # 根据传入的图计算出 CPU 计划
    struct ggml_cplan cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);

    # 如果 CPU 上下文中的工作空间大小小于计划中的工作空间大小
    if (cpu_ctx->work_size < cplan.work_size) {
        # 重新分配 CPU 上下文中的工作数据空间
        cpu_ctx->work_data = realloc(cpu_ctx->work_data, cplan.work_size);
        # 更新 CPU 上下文中的工作空间大小
        cpu_ctx->work_size = cplan.work_size;
    }

    # 将计划中的工作数据指向 CPU 上下文中的工作数据
    cplan.work_data = cpu_ctx->work_data;

    # 调用图计算函数，传入传入的图和计划
    ggml_graph_compute(cgraph, &cplan);
}

# 判断 CPU 是否支持给定的操作
static bool ggml_backend_cpu_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    # CPU 总是支持操作，返回 true
    return true;
    # 标记未使用的参数
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
    /* .graph_plan_create   = */ ggml_backend_cpu_graph_plan_create,  // 设置图计划创建函数
    /* .graph_plan_free     = */ ggml_backend_cpu_graph_plan_free,    // 设置图计划释放函数
    /* .graph_plan_compute  = */ ggml_backend_cpu_graph_plan_compute, // 设置图计划计算函数
    /* .graph_compute       = */ ggml_backend_cpu_graph_compute,      // 设置图计算函数
    /* .supports_op         = */ ggml_backend_cpu_supports_op,        // 设置操作支持函数
// 初始化 CPU 后端，分配内存并设置默认值
ggml_backend_t ggml_backend_cpu_init(void) {
    // 分配 CPU 后端上下文的内存空间
    struct ggml_backend_cpu_context * ctx = malloc(sizeof(struct ggml_backend_cpu_context));

    // 设置 CPU 后端上下文的默认线程数、工作数据和工作数据大小
    ctx->n_threads = GGML_DEFAULT_N_THREADS;
    ctx->work_data = NULL;
    ctx->work_size = 0;

    // 分配 CPU 后端的内存空间
    ggml_backend_t cpu_backend = malloc(sizeof(struct ggml_backend));

    // 初始化 CPU 后端的接口和上下文
    *cpu_backend = (struct ggml_backend) {
        /* .interface = */ cpu_backend_i,
        /* .context   = */ ctx
    };
    return cpu_backend;
}

// 判断给定的后端是否为 CPU 后端
bool ggml_backend_is_cpu(ggml_backend_t backend) {
    return backend->iface.get_name == ggml_backend_cpu_name;
}

// 设置 CPU 后端的线程数
void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads) {
    // 确保给定的后端是 CPU 后端
    GGML_ASSERT(ggml_backend_is_cpu(backend_cpu));

    // 获取 CPU 后端的上下文，并设置线程数
    struct ggml_backend_cpu_context * ctx = (struct ggml_backend_cpu_context *)backend_cpu->context;
    ctx->n_threads = n_threads;
}

// 从指针创建 CPU 后端的缓冲区
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
    # 如果不满足条件，则执行以下代码
    __attribute__((aligned(GGML_MEM_ALIGN)))
    # 对接下来的变量或数据进行特殊的对齐处理
    #endif
    # 结束条件判断
    char context_buffer[GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS*sizeof(struct ggml_tensor) + GGML_MAX_SPLITS*sizeof(struct ggml_cgraph)];
    # 声明一个字符数组，用于存储数据
// 定义宏，根据节点找到或插入哈希表中的哈希值
#define hash_id(node) ggml_hash_find_or_insert(sched->hash_set, node)
// 定义宏，根据节点分配内存
#define node_allocr(node) sched->node_talloc[hash_id(node)]

// 判断操作是否为视图操作
static bool ggml_is_view_op(enum ggml_op op) {
    return op == GGML_OP_VIEW || op == GGML_OP_RESHAPE || op == GGML_OP_PERMUTE || op == GGML_OP_TRANSPOSE;
}

// 返回后端的优先级，值越小优先级越高
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
char causes[GGML_DEFAULT_GRAPH_SIZE*4 + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS][128]; // debug, remove
static ggml_backend_t sched_backend_from_cur(ggml_backend_sched_t sched, struct ggml_tensor * node) {
    // 如果目标张量已经在缓冲区中分配了，我们必须假设它对于保持在那里是至关重要的
    // 即 kv 缓存更新
    // 注意，这不允许回退到 CPU。需要将输出张量添加到分割中，将数据复制回原始后端。
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
    # 遍历节点的输入源
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        # 获取当前输入源
        const struct ggml_tensor * src = node->src[i];
        # 如果输入源为空，则跳出循环
        if (src == NULL) {
            break;
        }
        # 获取输入源的后端类型
        ggml_backend_t src_backend = ggml_get_backend(src);
        # 如果输入源的后端类型不为空
        if (src_backend != NULL) {
            # 获取输入源的调度优先级
            int src_prio = sched_backend_prio(sched, src_backend);
            # 获取输入源的数据大小
            size_t src_size = ggml_nbytes(src);
            # 如果输入源的调度优先级小于当前优先级，并且数据大小大于等于当前大小
            if (src_prio < cur_prio && src_size >= cur_size) {
                # 更新当前优先级、大小和后端类型
                cur_prio = src_prio;
                cur_size = src_size;
                cur_backend = src_backend;
                # 格式化记录导致当前选择的输入源
                sprintf(causes[hash_id(node)], "1.src%d", i);
            }
        }
    }
    # 返回当前选择的后端类型
    return cur_backend;
# 定义一个静态函数，用于格式化文件大小
static char * fmt_size(size_t size) {
    # 定义一个静态字符数组，用于存储格式化后的文件大小
    static char buffer[128];
    # 如果文件大小大于等于1MB，则以MB为单位格式化文件大小
    if (size >= 1024*1024) {
        sprintf(buffer, "%zuM", size/1024/1024);
    } 
    # 否则以KB为单位格式化文件大小
    else {
        sprintf(buffer, "%zuK", size/1024);
    }
    # 返回格式化后的文件大小
    return buffer;
}

# 定义一个静态函数，用于打印调度分配情况
static void sched_print_assignments(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    # 初始化当前分割的值为0
    int cur_split = 0;
    // 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 如果当前节点是分割节点的起始节点
        if (cur_split < sched->n_splits && i == sched->splits[cur_split].i_start) {
            // 获取分割节点的后端类型和输入信息，并输出到标准错误流
            ggml_backend_t split_backend = ggml_tallocr_get_buffer(sched->splits[cur_split].tallocr)->backend;
            fprintf(stderr, "\n## SPLIT #%d: %s # %d inputs: ", cur_split, ggml_backend_name(split_backend), sched->splits[cur_split].n_inputs);
            for (int j = 0; j < sched->splits[cur_split].n_inputs; j++) {
                // 输出分割节点的输入信息
                fprintf(stderr, "[%s (%5.5s)] ", sched->splits[cur_split].inputs[j]->name, fmt_size(ggml_nbytes(sched->splits[cur_split].inputs[j])));
            }
            fprintf(stderr, "\n");
            cur_split++;
        }
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果当前节点是视图操作节点，则跳过
        if (ggml_is_view_op(node->op)) {
            continue;
        }
        // 获取当前节点的分配器和后端类型，并输出到标准错误流
        ggml_tallocr_t node_allocr = node_allocr(node);
        ggml_backend_t node_backend = node_allocr ? ggml_tallocr_get_buffer(node_allocr)->backend : NULL;
        fprintf(stderr, "node #%3d (%10.10s): %20.20s (%4.4s) [%4.4s %8.8s]:", i, ggml_op_name(node->op), node->name, fmt_size(ggml_nbytes(node)), node_allocr ? ggml_backend_name(node_backend) : "NULL", causes[hash_id(node)]);
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前节点的源节点
            struct ggml_tensor * src = node->src[j];
            // 如果源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取源节点的分配器和后端类型，并输出到标准错误流
            ggml_tallocr_t src_allocr = node_allocr(src);
            ggml_backend_t src_backend = src_allocr ? ggml_tallocr_get_buffer(src_allocr)->backend : NULL;
            fprintf(stderr, " %20.20s (%4.4s) [%4.4s %8.8s]", src->name, fmt_size(ggml_nbytes(src)), src_backend ? ggml_backend_name(src_backend) : "NULL", causes[hash_id(src)]);
        }
        fprintf(stderr, "\n");
    }
// 创建一个具有相同内存布局的张量副本
static struct ggml_tensor * ggml_dup_tensor_layout(struct ggml_context * ctx, const struct ggml_tensor * tensor) {
    // 复制张量
    struct ggml_tensor * dup = ggml_dup_tensor(ctx, tensor);
    // 遍历张量的维度
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        // 将副本的维度设置为与原张量相同
        dup->nb[i] = tensor->nb[i];
    }
    // 返回副本
    return dup;
}

// 为操作分配后端，并将图拆分为可以在相同后端上计算的子图
// TODO: 合并 passes
static void sched_split_graph(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    // 重置状态
    size_t hash_size = sched->hash_set.size;
    // 将哈希集合的键重置为0
    memset(sched->hash_set.keys, 0, sizeof(sched->hash_set.keys[0]) * hash_size);
    // 将节点分配的内存重置为0
    memset(sched->node_talloc,   0, sizeof(sched->node_talloc[0])   * hash_size);
    // 将节点的副本重置为0
    memset(sched->node_copies,   0, sizeof(sched->node_copies[0])   * hash_size);
    // 将拆分数重置为0
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

    // pass 1: 为具有分配输入的操作分配后端
    for (int i = 0; i < graph->n_leafs; i++) {
        // 获取叶子节点
        struct ggml_tensor * leaf = graph->leafs[i];
        // 如果节点已分配内存
        if (node_allocr(leaf) != NULL) {
            // 不覆盖用户分配
            continue;
        }
        // 获取叶子节点的后端
        ggml_backend_t leaf_backend = ggml_get_backend(leaf);
        // 如果叶子节点没有后端，并且视图源不为空
        if (leaf_backend == NULL && leaf->view_src != NULL) {
            // 获取视图源的后端
            leaf_backend = ggml_get_backend(leaf->view_src);
        }
        // 如果叶子节点有后端
        if (leaf_backend != NULL) {
            // 为节点分配内存
            node_allocr(leaf) = ggml_backend_sched_get_tallocr(sched, leaf_backend);
        }
    }
}
    // 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果节点已经有分配器，则跳过
        if (node_allocr(node) != NULL) {
            continue;
        }
        // 获取当前节点的调度后端
        ggml_backend_t node_backend = sched_backend_from_cur(sched, node);
        // 如果存在调度后端，则为节点分配分配器
        if (node_backend != NULL) {
            node_allocr(node) = ggml_backend_sched_get_tallocr(sched, node_backend);
        }
    }
    //printf("PASS 1 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 2: assign backends to ops from current assignments
    // TODO:
    //  - reuse sched_backend_from_cur
    // 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取当前节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 如果节点没有分配器
        if (node_allocr == NULL) {
            // 初始化当前优先级和大小
            int    cur_prio = INT_MAX;
            size_t cur_size = 0;
            // 遍历当前节点的源节点
            for (int j = 0; j < GGML_MAX_SRC; j++) {
                struct ggml_tensor * src = node->src[j];
                // 如果源节点为空，则跳出循环
                if (src == NULL) {
                    break;
                }
                // 获取源节点的分配器
                ggml_tallocr_t src_allocr = node_allocr(src);
                // 如果源节点有分配器
                if (src_allocr != NULL) {
                    // 获取源节点分配器的优先级和大小
                    int    src_prio = sched_allocr_prio(sched, src_allocr);
                    size_t src_size = ggml_nbytes(src);
                    // 如果源节点的优先级小于当前优先级且大小大于等于当前大小
                    if (src_prio < cur_prio && src_size >= cur_size) {
                        // 更新当前优先级、大小和分配器
                        cur_prio = src_prio;
                        cur_size = src_size;
                        node_allocr = src_allocr;
                        // 更新原因
                        sprintf(causes[hash_id(node)], "2.src%d", j);
                    }
                }
            }
            // 如果存在分配器，则为当前节点分配分配器
            if (node_allocr != NULL) {
                node_allocr(node) = node_allocr;
            }
        }
    }
    //printf("PASS 2 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 3: assign backends to remaining src from dst (should only be leafs)
    // 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 为当前节点创建分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前源节点
            struct ggml_tensor * src = node->src[j];
            // 如果当前源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 为当前源节点创建分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果当前源节点的分配器为空
            if (src_allocr == NULL) {
                // 将当前节点的分配器赋值给当前源节点的分配器
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
    // 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果当前节点的视图源节点为空
        if (node->view_src == NULL) {
            // 设置调度器的分割点分配器为当前节点的分配器
            sched->splits[0].tallocr = node_allocr(node);
            break;
        }
    }
    // 设置当前分割点的起始索引为0
    sched->splits[0].i_start = 0;
    // 设置当前分割点的输入数量为0
    sched->splits[0].n_inputs = 0;
    // 将当前分割点的输入数组清零
    memset(sched->splits[0].inputs, 0, sizeof(sched->splits[0].inputs)); //HACK
    // 获取当前分割点的分配器
    ggml_tallocr_t cur_allocr = sched->splits[0].tallocr;
    // 获取当前分割点的后端ID
    size_t cur_backend_id = sched_allocr_prio(sched, cur_allocr);
    }
    // 设置当前分割点的结束索引为图中节点的数量
    sched->splits[cur_split].i_end = graph->n_nodes;
    // 设置调度器的分割点数量为当前分割点加1
    sched->n_splits = cur_split + 1;

    //fprintf(stderr, "PASS 4 ASSIGNMENTS\n"); sched_print_assignments(sched, graph); fflush(stdout);
#if 1
    // 如果条件为真，则执行以下代码块
    // 对于图中的每个节点进行遍历
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 如果节点没有分配器，则输出错误信息
        if (node_allocr == NULL) {
            fprintf(stderr, "!!!!!!! %s has no backend\n", node->name);
        }
        // 对于节点的每个源进行遍历
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前源
            struct ggml_tensor * src = node->src[j];
            // 如果源为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取源的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果源的分配器不等于节点的分配器，则输出错误信息
            if (src_allocr != node_allocr /* && src_backend != NULL */) { // ignore nulls for now
                fprintf(stderr, "!!!! %s has backend %s, src %d (%s) has backend %s\n",
                    node->name, node_allocr ? ggml_backend_name(ggml_tallocr_get_buffer(node_allocr)->backend) : "NULL",
                    j, src->name, src_allocr ? ggml_backend_name(ggml_tallocr_get_buffer(src_allocr)->backend) : "NULL");
            }
        }
    }
#endif

    // 为每个拆分创建图的副本
    // FIXME: 避免这种复制，以某种方式将拆分输入传递给ggml_gallocr_alloc_graph_n
    // 创建一个新的图副本，包括原图节点数和拆分数*最大拆分输入数
    struct ggml_cgraph * graph_copy = ggml_new_graph_custom(sched->ctx, graph->n_nodes + sched->n_splits*GGML_MAX_SPLIT_INPUTS, false);
    // 遍历调度器的分割数量
    for (int i = 0; i < sched->n_splits; i++) {
        // 获取当前分割的指针
        struct ggml_backend_sched_split * split = &sched->splits[i];
        // 为当前分割创建图形视图
        split->graph = ggml_graph_view(sched->ctx, graph, split->i_start, split->i_end);

        // 将输入添加到图形副本中，以便它们在分割开始时由ggml-alloc分配
        for (int j = 0; j < split->n_inputs; j++) {
            // 获取当前输入的指针
            struct ggml_tensor * input = split->inputs[j];
            // 获取当前输入的副本
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(input)][sched_allocr_prio(sched, split->tallocr)];
            // 将输入的源设置为原始输入
            input_cpy->src[0] = input;
            // 将输入副本添加到图形副本的节点中
            graph_copy->nodes[graph_copy->n_nodes++] = input_cpy;
        }

        // 将原始图形中的节点添加到图形副本中
        for (int j = split->i_start; j < split->i_end; j++) {
            graph_copy->nodes[graph_copy->n_nodes++] = graph->nodes[j];
        }
    }
    // 将图形副本赋值给调度器的图形
    sched->graph = graph_copy;
# 分配调度的拆分
static void sched_alloc_splits(ggml_backend_sched_t sched) {
    # 调用 ggml_gallocr_alloc_graph_n 函数，分配图形
    ggml_gallocr_alloc_graph_n(
        sched->galloc,
        sched->graph,
        sched->hash_set,
        sched->node_talloc);
}

# 计算调度的拆分
static void sched_compute_splits(ggml_backend_sched_t sched) {
    # 初始化存储拷贝时间和计算时间的数组
    uint64_t copy_us[GGML_MAX_BACKENDS] = {0};
    uint64_t compute_us[GGML_MAX_BACKENDS] = {0};

    # 获取调度的拆分结构体指针
    struct ggml_backend_sched_split * splits = sched->splits;
}
    // 遍历调度器中的分割任务
    for (int i = 0; i < sched->n_splits; i++) {
        // 获取当前分割任务的指针
        struct ggml_backend_sched_split * split = &splits[i];
        // 获取当前分割任务对应的后端
        ggml_backend_t split_backend = ggml_tallocr_get_buffer(split->tallocr)->backend;
        // 获取当前分割任务对应的后端 ID
        int split_backend_id = sched_backend_prio(sched, split_backend);

        // 将输入张量复制到分割后端
        uint64_t copy_start_us = ggml_time_us();
        // 遍历当前分割任务的输入张量
        for (int j = 0; j < split->n_inputs; j++) {
            // 获取当前输入张量的副本
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(split->inputs[j])][sched_backend_prio(sched, split_backend)];
            // 如果当前输入张量没有缓冲区且没有视图源
            if (split->inputs[j]->buffer == NULL) {
                if (split->inputs[j]->view_src == NULL) {
                    // 输出错误信息并退出程序
                    fprintf(stderr, "input %s has no buffer and no view_src\n", split->inputs[j]->name);
                    exit(1);
                }
                // 获取当前输入张量的视图
                struct ggml_tensor * view = split->inputs[j];
                view->backend = view->view_src->backend;
                view->buffer  = view->view_src->buffer;
                view->data    = (char *)view->view_src->data + view->view_offs;
                // 初始化后端缓冲区的张量
                ggml_backend_buffer_init_tensor(ggml_backend_sched_get_buffer(sched, view->buffer->backend), view);
            }
            // 如果当前输入张量的副本没有缓冲区
            if (input_cpy->buffer == NULL) {
                // 输出错误信息并退出程序
                fprintf(stderr, "input_cpy %s has no buffer\n", input_cpy->name);
                exit(1);
            }
            // 断言当前输入张量的后端与副本的后端不同
            GGML_ASSERT(split->inputs[j]->buffer->backend != input_cpy->buffer->backend);
            // 断言副本的后端与分割后端相同
            GGML_ASSERT(input_cpy->buffer->backend == split_backend);
            // 复制输入张量到副本
            ggml_backend_tensor_copy(split->inputs[j], input_cpy);
        }
        // 计算复制操作的时间
        int64_t copy_end_us = ggml_time_us();
        copy_us[split_backend_id] += copy_end_us - copy_start_us;
// 如果条件为假，则执行以下代码块
#if 0
        // 创建一个用于存储文件名的字符数组
        char split_filename[GGML_MAX_NAME];
        // 格式化文件名字符串
        snprintf(split_filename, GGML_MAX_NAME, "split_%i_%s.dot", i, ggml_backend_name(split_backend));
        // 将分割后的图形以DOT格式写入文件
        ggml_graph_dump_dot(split->graph, NULL, split_filename);
#endif

        // 计算开始时间
        uint64_t compute_start_us = ggml_time_us();
        // 计算分割后的图形
        ggml_backend_graph_compute(split_backend, split->graph);
        // 同步分割后的图形
        // ggml_backend_synchronize(split_backend);
        // 计算结束时间
        uint64_t compute_end_us = ggml_time_us();
        // 计算并记录计算时间
        compute_us[split_backend_id] += compute_end_us - compute_start_us;
    }

// 如果条件为假，则执行以下代码块
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
        ggml_tallocr_reset(sched->tallocs[i]);
    }
}

// 创建新的后端调度器
ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, int n_backends) {
    // 断言后端数量不超过最大值
    GGML_ASSERT(n_backends <= GGML_MAX_BACKENDS);

    // 分配内存以存储后端调度器
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

    // 创建全局分配器
    sched->galloc = ggml_gallocr_new();

    // 为每个后端初始化测量分配器
    for (int i = 0; i < n_backends; i++) {
        sched->tallocs[i] = ggml_tallocr_new_measure_from_backend(backends[i]);
    }

    return sched;
}

// 释放后端调度器
void ggml_backend_sched_free(ggml_backend_sched_t sched) {
    // 如果调度器为空，则直接返回
    if (sched == NULL) {
        return;
    }
    # 遍历调度器中的后端数量
    for (int i = 0; i < sched->n_backends; i++) {
        # 释放调度器中第 i 个后端的内存
        ggml_tallocr_free(sched->tallocs[i]);
    }
    # 释放调度器中的全局内存分配器
    ggml_gallocr_free(sched->galloc);
    # 释放调度器中哈希集合的键
    free(sched->hash_set.keys);
    # 释放调度器中节点的内存
    free(sched->node_talloc);
    # 释放调度器中节点的副本内存
    free(sched->node_copies);
    # 释放调度器本身的内存
    free(sched);
// 初始化测量图的调度器，传入调度器和测量图
void ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph) {
    // 初始化哈希表的大小
    size_t hash_size = measure_graph->visited_hash_table.size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS;
    // 设置哈希表的大小
    sched->hash_set.size = hash_size;
    // 分配哈希表的键内存空间
    sched->hash_set.keys = malloc(sizeof(sched->hash_set.keys[0]) * hash_size);
    // 分配节点内存空间
    sched->node_talloc   = malloc(sizeof(sched->node_talloc[0])   * hash_size);
    // 分配节点拷贝内存空间
    sched->node_copies   = malloc(sizeof(sched->node_copies[0])   * hash_size);

    // 对调度器进行图分割
    sched_split_graph(sched, measure_graph);
    // 分配分割
    sched_alloc_splits(sched);

    // 分配缓冲区并重置分配器
    for (int i = 0; i < sched->n_backends; i++) {
        // 获取分配器的最大大小
        size_t size = ggml_tallocr_max_size(sched->tallocs[i]);
        // 释放分配器
        ggml_tallocr_free(sched->tallocs[i]);
        // 从后端创建新的分配器
        sched->tallocs[i] = ggml_tallocr_new_from_backend(sched->backends[i], size);
    }

    // 重置调度器
    sched_reset(sched);
}

// 计算图的调度器，传入调度器和图
void ggml_backend_sched_graph_compute(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    // 断言哈希表的大小大于等于图的访问哈希表大小加上最大分割数乘以最大分割输入数
    GGML_ASSERT(sched->hash_set.size >= graph->visited_hash_table.size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS);

    // 对调度器进行图分割
    sched_split_graph(sched, graph);
    // 分配分割
    sched_alloc_splits(sched);
    // 计算分割
    sched_compute_splits(sched);
    // 重置调度器
    sched_reset(sched);
}

// 获取调度器的分配器，传入调度器和后端
ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend) {
    // 获取后端的索引
    int backend_index = sched_backend_prio(sched, backend);
    // 返回调度器的分配器
    return sched->tallocs[backend_index];
}

// 获取调度器的缓冲区，传入调度器和后端
ggml_backend_buffer_t ggml_backend_sched_get_buffer(ggml_backend_sched_t sched, ggml_backend_t backend) {
    // 获取后端的索引
    int backend_index = sched_backend_prio(sched, backend);
    // 返回分配器的缓冲区
    return ggml_tallocr_get_buffer(sched->tallocs[backend_index]);
}

// 设置节点的后端，传入调度器、节点和后端
void ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend) {
    // 获取后端的索引
    int backend_index = sched_backend_prio(sched, backend);
    // 断言后端索引大于等于0且小于调度器的后端数量
    GGML_ASSERT(backend_index >= 0 && backend_index < sched->n_backends);
}
    # 为节点分配资源，使用调度器中对应后端索引的资源分配器
    node_allocr(node) = sched->tallocs[backend_index];
# 闭合前面的函数定义
```