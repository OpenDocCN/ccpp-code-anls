# `ggml\src\ggml-backend.c`

```cpp
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

// 定义一个宏，用于返回两个值中的较大值
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// backend buffer type

// 分配一个指定类型和大小的缓冲区
ggml_backend_buffer_t ggml_backend_buft_alloc_buffer(ggml_backend_buffer_type_t buft, size_t size) {
    return buft->iface.alloc_buffer(buft, size);
}

// 获取缓冲区的对齐方式
size_t ggml_backend_buft_get_alignment(ggml_backend_buffer_type_t buft) {
    return buft->iface.get_alignment(buft);
}

// 获取分配给缓冲区的大小
size_t ggml_backend_buft_get_alloc_size(ggml_backend_buffer_type_t buft, struct ggml_tensor * tensor) {
    // 如果 get_alloc_size 存在，则调用它，否则调用 ggml_nbytes
    if (buft->iface.get_alloc_size) {
        return buft->iface.get_alloc_size(buft, tensor);
    }
    return ggml_nbytes(tensor);
}

// 检查缓冲区类型是否支持指定的后端
bool ggml_backend_buft_supports_backend(ggml_backend_buffer_type_t buft, ggml_backend_t backend) {
    return buft->iface.supports_backend(buft, backend);
}

// 检查缓冲区类型是否为主机缓冲区
bool ggml_backend_buft_is_host(ggml_backend_buffer_type_t buft) {
    if (buft->iface.is_host) {
        return buft->iface.is_host(buft);
    }
    return false;
}

// backend buffer

// 初始化后端缓冲区
ggml_backend_buffer_t ggml_backend_buffer_init(
               ggml_backend_buffer_type_t      buft,
        struct ggml_backend_buffer_i           iface,
               ggml_backend_buffer_context_t   context,
               size_t                          size) {
    // 分配内存给缓冲区
    ggml_backend_buffer_t buffer = malloc(sizeof(struct ggml_backend_buffer));

    // 断言 get_base 函数不为空
    GGML_ASSERT(iface.get_base != NULL);

    // 初始化缓冲区
    (*buffer) = (struct ggml_backend_buffer) {
        /* .interface = */ iface,
        /* .buft      = */ buft,
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

    // 如果 free_buffer 函数不为空，则调用它
    if (buffer->iface.free_buffer != NULL) {
        buffer->iface.free_buffer(buffer);
    }
}
    # 释放内存缓冲区
    free(buffer);
// 获取缓冲区的大小
size_t ggml_backend_buffer_get_size(ggml_backend_buffer_t buffer) {
    return buffer->size;
}

// 获取缓冲区的基地址
void * ggml_backend_buffer_get_base(ggml_backend_buffer_t buffer) {
    void * base = buffer->iface.get_base(buffer);

    // 断言缓冲区的基地址不为空
    GGML_ASSERT(base != NULL && "backend buffer base cannot be NULL");

    return base;
}

// 初始化张量
void ggml_backend_buffer_init_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // 如果存在初始化张量的接口，则调用初始化张量的函数
    if (buffer->iface.init_tensor) {
        buffer->iface.init_tensor(buffer, tensor);
    }
}

// 获取缓冲区的对齐方式
size_t ggml_backend_buffer_get_alignment (ggml_backend_buffer_t buffer) {
    return ggml_backend_buft_get_alignment(ggml_backend_buffer_type(buffer));
}

// 获取缓冲区分配的大小
size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    return ggml_backend_buft_get_alloc_size(ggml_backend_buffer_type(buffer), tensor);
}

// 清空缓冲区
void ggml_backend_buffer_clear(ggml_backend_buffer_t buffer, uint8_t value) {
    buffer->iface.clear(buffer, value);
}

// 判断缓冲区是否在主机上
bool ggml_backend_buffer_is_host(ggml_backend_buffer_t buffer) {
    return ggml_backend_buft_is_host(ggml_backend_buffer_type(buffer));
}

// 获取缓冲区的类型
ggml_backend_buffer_type_t ggml_backend_buffer_type(ggml_backend_buffer_t buffer) {
    return buffer->buft;
}

// 获取后端的名称
const char * ggml_backend_name(ggml_backend_t backend) {
    if (backend == NULL) {
        return "NULL";
    }
    return backend->iface.get_name(backend);
}

// 释放后端资源
void ggml_backend_free(ggml_backend_t backend) {
    if (backend == NULL) {
        return;
    }

    backend->iface.free(backend);
}

// 获取默认的缓冲区类型
ggml_backend_buffer_type_t ggml_backend_get_default_buffer_type(ggml_backend_t backend) {
    return backend->iface.get_default_buffer_type(backend);
}

// 分配缓冲区
ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size) {
    return ggml_backend_buft_alloc_buffer(ggml_backend_get_default_buffer_type(backend), size);
}

// 获取后端的对齐方式
size_t ggml_backend_get_alignment(ggml_backend_t backend) {
    # 调用 ggml_backend_get_default_buffer_type 函数获取默认的缓冲区类型，并将其作为参数传递给 ggml_backend_buft_get_alignment 函数，返回对齐方式
    return ggml_backend_buft_get_alignment(ggml_backend_get_default_buffer_type(backend));
# 设置异步方式将数据写入张量
void ggml_backend_tensor_set_async(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    # 断言张量数据不为空，如果为空则输出错误信息"tensor not allocated"
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    # 断言写入数据的偏移量加上数据大小不超过张量的总字节数，如果超过则输出错误信息"tensor write out of bounds"
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor write out of bounds");

    # 调用后端接口的异步设置张量数据的方法
    backend->iface.set_tensor_async(backend, tensor, data, offset, size);
}

# 设置异步方式从张量中读取数据
void ggml_backend_tensor_get_async(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    # 断言张量数据不为空，如果为空则输出错误信息"tensor not allocated"
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    # 断言读取数据的偏移量加上数据大小不超过张量的总字节数，如果超过则输出错误信息"tensor read out of bounds"
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor read out of bounds");

    # 调用后端接口的异步获取张量数据的方法
    backend->iface.get_tensor_async(backend, tensor, data, offset, size);
}

# 设置同步方式将数据写入张量
void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    # 断言张量数据不为空，如果为空则输出错误信息"tensor not allocated"
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    # 断言张量缓冲区不为空，如果为空则输出错误信息"tensor buffer not set"
    GGML_ASSERT(tensor->buffer != NULL && "tensor buffer not set");
    # 断言写入数据的偏移量加上数据大小不超过张量的总字节数，如果超过则输出错误信息"tensor write out of bounds"
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor write out of bounds");

    # 调用张量缓冲区接口的设置张量数据的方法
    tensor->buffer->iface.set_tensor(tensor->buffer, tensor, data, offset, size);
}

# 设置同步方式从张量中读取数据
void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    # 断言张量数据不为空，如果为空则输出错误信息"tensor not allocated"
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    # 断言张量缓冲区不为空，如果为空则输出错误信息"tensor buffer not set"
    GGML_ASSERT(tensor->buffer != NULL && "tensor buffer not set");
    # 断言读取数据的偏移量加上数据大小不超过张量的总字节数，如果超过则输出错误信息"tensor read out of bounds"
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor read out of bounds");

    # 调用张量缓冲区接口的获取张量数据的方法
    tensor->buffer->iface.get_tensor(tensor->buffer, tensor, data, offset, size);
}

# 同步后端操作
void ggml_backend_synchronize(ggml_backend_t backend) {
    # 如果后端接口的同步方法为空，则直接返回
    if (backend->iface.synchronize == NULL) {
        return;
    }

    # 调用后端接口的同步方法
    backend->iface.synchronize(backend);
}

# 创建后端图计划
ggml_backend_graph_plan_t ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    # 调用后端接口的创建图计划方法
    return backend->iface.graph_plan_create(backend, cgraph);
}

# 释放后端图计划
void ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 通过接口指针调用图计划释放函数，释放后端和计划
    backend->iface.graph_plan_free(backend, plan);
// 计算图计划的后端计算函数，调用后端接口的图计划计算函数
void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    backend->iface.graph_plan_compute(backend, plan);

    // TODO: optional sync
    ggml_backend_synchronize(backend);
}

// 使用后端接口的图计算函数计算计算图
bool ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    if (!backend->iface.graph_compute(backend, cgraph)) {
        return false;
    }

    // TODO: optional sync
    ggml_backend_synchronize(backend);
    return true;
}

// 检查后端是否支持给定的操作
bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
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

// 张量复制函数
void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst) {
    //printf("src: %s ne: [%d %d %d %d] nb: [%d %d %d %d]\n", src->name, (int)src->ne[0], (int)src->ne[1], (int)src->ne[2], (int)src->ne[3], (int)src->nb[0], (int)src->nb[1], (int)src->nb[2], (int)src->nb[3]);
    //printf("dst: %s ne: [%d %d %d %d] nb: [%d %d %d %d]\n", dst->name, (int)dst->ne[0], (int)dst->ne[1], (int)dst->ne[2], (int)dst->ne[3], (int)dst->nb[0], (int)dst->nb[1], (int)dst->nb[2], (int)dst->nb[3]);
    GGML_ASSERT(ggml_are_same_layout(src, dst) && "cannot copy tensors with different layouts");

    // fprintf(stderr, "cpy tensor %s from %s to %s (%lu bytes)\n", src->name, ggml_backend_name(src->backend), ggml_backend_name(dst->backend), ggml_nbytes(src));

    if (src == dst) {
        return;
    }

    // TODO: allow backends to support copy to/from same backend
}
    # 如果目标缓冲区的接口实现了从另一个张量复制数据的方法
    if (dst->buffer->iface.cpy_tensor_from != NULL) {
        # 调用目标缓冲区的接口方法，从源张量复制数据到目标张量
        dst->buffer->iface.cpy_tensor_from(dst->buffer, src, dst);
    } 
    # 如果源缓冲区的接口实现了向另一个张量复制数据的方法
    else if (src->buffer->iface.cpy_tensor_to != NULL) {
        # 调用源缓冲区的接口方法，从源张量复制数据到目标张量
        src->buffer->iface.cpy_tensor_to(src->buffer, src, dst);
    } 
    # 如果以上两种情况都不满足
    else {
        // 在从/向 CPU 复制数据时不应该执行到这里
        #ifndef NDEBUG
        # 打印调试信息，说明未实现从源张量复制数据到目标张量的方法
        fprintf(stderr, "ggml_backend_tensor_copy: neither cpy_tensor_from nor cpy_tensor_to "
                        "are implemented for %s and %s, falling back to get/set\n", src->name, dst->name);
        #endif
        # 计算源张量数据的字节数
        size_t nbytes = ggml_nbytes(src);
        # 分配内存空间，用于存储源张量的数据
        void * data = malloc(nbytes);
        # 从源张量获取数据
        ggml_backend_tensor_get(src, data, 0, nbytes);
        # 将数据设置到目标张量
        ggml_backend_tensor_set(dst, data, 0, nbytes);
        # 释放内存空间
        free(data);
    }
// 后端注册表
#define GGML_MAX_BACKENDS_REG 16

// 定义后端注册表结构
struct ggml_backend_reg {
    char name[128]; // 后端名称
    ggml_backend_init_fn init_fn; // 后端初始化函数
    ggml_backend_buffer_type_t default_buffer_type; // 默认缓冲类型
    void * user_data; // 用户数据
};

// 静态后端注册表数组
static struct ggml_backend_reg ggml_backend_registry[GGML_MAX_BACKENDS_REG];
// 后端注册表计数
static size_t ggml_backend_registry_count = 0;

// 声明 CPU 后端初始化函数
static ggml_backend_t ggml_backend_reg_cpu_init(const char * params, void * user_data);

// 后端注册表初始化函数
static void ggml_backend_registry_init(void) {
    static bool initialized = false;

    if (initialized) {
        return;
    }

    initialized = true;

    // 注册 CPU 后端
    ggml_backend_register("CPU", ggml_backend_reg_cpu_init, ggml_backend_cpu_buffer_type(), NULL);

    // 在此处添加前向声明以避免包含后端头文件
#ifdef GGML_USE_CUBLAS
    extern void ggml_backend_cuda_reg_devices();
    ggml_backend_cuda_reg_devices();
#endif

#ifdef GGML_USE_METAL
    // 声明 Metal 后端初始化函数和缓冲类型函数，并注册 Metal 后端
    extern ggml_backend_t ggml_backend_reg_metal_init(const char * params, void * user_data);
    extern ggml_backend_buffer_type_t ggml_backend_metal_buffer_type();
    ggml_backend_register("Metal", ggml_backend_reg_metal_init, ggml_backend_metal_buffer_type(), NULL);
#endif
}

// 后端注册函数
void ggml_backend_register(const char * name, ggml_backend_init_fn init_fn, ggml_backend_buffer_type_t default_buffer_type, void * user_data) {
    GGML_ASSERT(ggml_backend_registry_count < GGML_MAX_BACKENDS_REG);

    size_t id = ggml_backend_registry_count;

    // 将后端信息填入注册表中
    ggml_backend_registry[id] = (struct ggml_backend_reg) {
        /* .name                = */ {0},
        /* .fn                  = */ init_fn,
        /* .default_buffer_type = */ default_buffer_type,
        /* .user_data           = */ user_data,
    };

    // 将后端名称复制到注册表中
    snprintf(ggml_backend_registry[id].name, sizeof(ggml_backend_registry[id].name), "%s", name);

#ifndef NDEBUG
    // 在调试模式下输出注册后端信息
    fprintf(stderr, "%s: registered backend %s\n", __func__, name);
#endif

    // 增加注册表计数
    ggml_backend_registry_count++;
}

// 获取注册后端数量
size_t ggml_backend_reg_get_count(void) {
    # 初始化后端注册表
    ggml_backend_registry_init();
    
    # 返回注册的后端数量
    return ggml_backend_registry_count;
// 根据名称查找注册的后端，返回其索引
size_t ggml_backend_reg_find_by_name(const char * name) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    // 遍历注册表，查找与给定名称匹配的后端
    for (size_t i = 0; i < ggml_backend_registry_count; i++) {
        // TODO: 在可移植的方式下不区分大小写
        if (strcmp(ggml_backend_registry[i].name, name) == 0) {
            return i;
        }
    }

    // 未找到匹配的后端
    return SIZE_MAX;
}

// 从后端参数字符串初始化后端
ggml_backend_t ggml_backend_reg_init_backend_from_str(const char * backend_str) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    // 查找参数字符串中的冒号位置
    const char * params = strchr(backend_str, ':');
    char backend_name[128];
    if (params == NULL) {
        // 如果没有冒号，则整个字符串为后端名称，参数为空字符串
        snprintf(backend_name, sizeof(backend_name), "%s", backend_str);
        params = "";
    } else {
        // 如果有冒号，则冒号之前的部分为后端名称，冒号之后的部分为参数
        snprintf(backend_name, sizeof(backend_name), "%.*s", (int)(params - backend_str), backend_str);
        params++;
    }

    // 根据名称查找注册的后端
    size_t backend_i = ggml_backend_reg_find_by_name(backend_name);

    // 如果未找到匹配的后端，则输出错误信息并返回空指针
    if (backend_i == SIZE_MAX) {
        fprintf(stderr, "%s: backend %s not found\n", __func__, backend_name);
        return NULL;
    }

    // 根据索引和参数初始化后端
    return ggml_backend_reg_init_backend(backend_i, params);
}

// 获取指定索引处的后端名称
const char * ggml_backend_reg_get_name(size_t i) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    // 确保索引在合法范围内
    GGML_ASSERT(i < ggml_backend_registry_count);
    return ggml_backend_registry[i].name;
}

// 根据索引和参数初始化后端
ggml_backend_t ggml_backend_reg_init_backend(size_t i, const char * params) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    // 确保索引在合法范围内
    GGML_ASSERT(i < ggml_backend_registry_count);
    return ggml_backend_registry[i].init_fn(params, ggml_backend_registry[i].user_data);
}

// 获取指定索引处的后端默认缓冲类型
ggml_backend_buffer_type_t ggml_backend_reg_get_default_buffer_type(size_t i) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    // 确保索引在合法范围内
    GGML_ASSERT(i < ggml_backend_registry_count);
    return ggml_backend_registry[i].default_buffer_type;
}

// 为指定索引处的后端分配缓冲区
ggml_backend_buffer_t ggml_backend_reg_alloc_buffer(size_t i, size_t size) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    // 确保索引在合法范围内
    GGML_ASSERT(i < ggml_backend_registry_count);
    # 调用 ggml_backend_buft_alloc_buffer 函数，分配指定大小的缓冲区，并返回结果
    return ggml_backend_buft_alloc_buffer(ggml_backend_registry[i].default_buffer_type, size);
// backend CPU

// 获取缓冲区的基地址
static void * ggml_backend_cpu_buffer_get_base(ggml_backend_buffer_t buffer) {
    return (void *)buffer->context;
}

// 释放缓冲区的内存
static void ggml_backend_cpu_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    free(buffer->context);
}

// 设置张量数据到缓冲区
static void ggml_backend_cpu_buffer_set_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    memcpy((char *)tensor->data + offset, data, size);

    GGML_UNUSED(buffer);
}

// 从缓冲区获取张量数据
static void ggml_backend_cpu_buffer_get_tensor(ggml_backend_buffer_t buffer, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    memcpy(data, (const char *)tensor->data + offset, size);

    GGML_UNUSED(buffer);
}

// 从源张量复制数据到目标张量
static void ggml_backend_cpu_buffer_cpy_tensor_from(ggml_backend_buffer_t buffer, struct ggml_tensor * src, struct ggml_tensor * dst) {
    ggml_backend_tensor_get(src, dst->data, 0, ggml_nbytes(src));

    GGML_UNUSED(buffer);
}

// 从源张量复制数据到目标张量
static void ggml_backend_cpu_buffer_cpy_tensor_to(ggml_backend_buffer_t buffer, struct ggml_tensor * src, struct ggml_tensor * dst) {
    ggml_backend_tensor_set(dst, src->data, 0, ggml_nbytes(src));

    GGML_UNUSED(buffer);
}

// 清空缓冲区，填充指定值
static void ggml_backend_cpu_buffer_clear(ggml_backend_buffer_t buffer, uint8_t value) {
    memset(buffer->context, value, buffer->size);
}

// 定义 CPU 后端缓冲区接口
static struct ggml_backend_buffer_i cpu_backend_buffer_i = {
    /* .free_buffer     = */ ggml_backend_cpu_buffer_free_buffer,
    /* .get_base        = */ ggml_backend_cpu_buffer_get_base,
    /* .init_tensor     = */ NULL, // 不需要初始化
    /* .set_tensor      = */ ggml_backend_cpu_buffer_set_tensor,
    /* .get_tensor      = */ ggml_backend_cpu_buffer_get_tensor,
    /* .cpy_tensor_from = */ ggml_backend_cpu_buffer_cpy_tensor_from,
    /* .cpy_tensor_to   = */ ggml_backend_cpu_buffer_cpy_tensor_to,
    /* .clear           = */ ggml_backend_cpu_buffer_clear,
};

// 对于从指针获取的缓冲区，不调用 free
static struct ggml_backend_buffer_i cpu_backend_buffer_i_from_ptr = {
    /* .free_buffer     = */ NULL, // ptr is not owned by the buffer, so it does not need to be freed
    /* .get_base        = */ ggml_backend_cpu_buffer_get_base, // 获取 CPU 缓冲区的基地址
    /* .init_tensor     = */ NULL, // 不需要初始化
    /* .set_tensor      = */ ggml_backend_cpu_buffer_set_tensor, // 设置 CPU 缓冲区的张量
    /* .get_tensor      = */ ggml_backend_cpu_buffer_get_tensor, // 获取 CPU 缓冲区的张量
    /* .cpy_tensor_from = */ ggml_backend_cpu_buffer_cpy_tensor_from, // 从其他缓冲区复制张量到 CPU 缓冲区
    /* .cpy_tensor_to   = */ ggml_backend_cpu_buffer_cpy_tensor_to, // 从 CPU 缓冲区复制张量到其他缓冲区
    /* .clear           = */ ggml_backend_cpu_buffer_clear, // 清空 CPU 缓冲区
};

static const size_t TENSOR_ALIGNMENT = 64; // 应该足够支持 AVX 512

static ggml_backend_buffer_t ggml_backend_cpu_buffer_type_alloc_buffer(ggml_backend_buffer_type_t buft, size_t size) {
    size += TENSOR_ALIGNMENT;   // malloc 可能返回未对齐的地址
    void * data = malloc(size); // TODO: maybe use GGML_ALIGNED_MALLOC? // 分配内存空间

    GGML_ASSERT(data != NULL && "failed to allocate buffer"); // 断言内存分配是否成功

    return ggml_backend_buffer_init(buft, cpu_backend_buffer_i, data, size); // 初始化 CPU 缓冲区
}

static size_t ggml_backend_cpu_buffer_type_get_alignment(ggml_backend_buffer_type_t buft) {
    return TENSOR_ALIGNMENT; // 返回张量对齐的大小

    GGML_UNUSED(buft); // 忽略未使用的参数
}

static bool ggml_backend_cpu_buffer_type_supports_backend(ggml_backend_buffer_type_t buft, ggml_backend_t backend) {
    return ggml_backend_is_cpu(backend); // 判断是否支持 CPU 后端

    GGML_UNUSED(buft); // 忽略未使用的参数
}

static bool ggml_backend_cpu_buffer_type_is_host(ggml_backend_buffer_type_t buft) {
    return true; // 判断是否是主机缓冲区

    GGML_UNUSED(buft); // 忽略未使用的参数
}

ggml_backend_buffer_type_t ggml_backend_cpu_buffer_type(void) {
    // 定义静态结构体变量 ggml_backend_cpu_buffer_type，表示 CPU 缓冲区类型
    static struct ggml_backend_buffer_type ggml_backend_cpu_buffer_type = {
        // 接口部分
        /* .iface = */ {
            // 分配缓冲区的函数指针
            /* .alloc_buffer     = */ ggml_backend_cpu_buffer_type_alloc_buffer,
            // 获取对齐方式的函数指针
            /* .get_alignment    = */ ggml_backend_cpu_buffer_type_get_alignment,
            // 获取分配大小的函数指针，默认为 ggml_nbytes
            /* .get_alloc_size   = */ NULL, // defaults to ggml_nbytes
            // 判断是否支持后端的函数指针
            /* .supports_backend = */ ggml_backend_cpu_buffer_type_supports_backend,
            // 判断是否为主机的函数指针
            /* .is_host          = */ ggml_backend_cpu_buffer_type_is_host,
        },
        // 上下文部分，暂时为空
        /* .context = */ NULL,
    };

    // 返回指向 ggml_backend_cpu_buffer_type 的指针
    return &ggml_backend_cpu_buffer_type;
#ifdef GGML_USE_CPU_HBM
// 如果定义了 GGML_USE_CPU_HBM，则包含 HBM 相关的头文件

#include <hbwmalloc.h>
// 包含 HBM 内存分配相关的头文件

static void ggml_backend_cpu_hbm_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    // 释放 HBM 缓冲区
    hbw_free(buffer->context);
}

static ggml_backend_buffer_t ggml_backend_cpu_hbm_buffer_type_alloc_buffer(ggml_backend_buffer_type_t buft, size_t size) {
    // 分配 HBM 缓冲区
    void * ptr;
    int result = hbw_posix_memalign(&ptr, ggml_backend_cpu_buffer_type_get_alignment(buft), size);
    if (result != 0) {
        // 如果分配失败，则输出错误信息并返回空指针
        fprintf(stderr, "failed to allocate HBM buffer of size %zu\n", size);
        return NULL;
    }

    // FIXME: this is a hack to avoid having to implement a new buffer type
    // 这是一个 hack，避免实现新的缓冲区类型
    ggml_backend_buffer_t buffer = ggml_backend_cpu_buffer_from_ptr(ptr, size);
    buffer->buft = buft;
    buffer->iface.free_buffer = ggml_backend_cpu_hbm_buffer_free_buffer;

    return buffer;
}

ggml_backend_buffer_type_t ggml_backend_cpu_hbm_buffer_type() {
    // 返回 CPU HBM 缓冲区类型
    static struct ggml_backend_buffer_type ggml_backend_cpu_buffer_type_hbm = {
        /* .iface    = */ {
            /* .alloc_buffer     = */ ggml_backend_cpu_hbm_buffer_type_alloc_buffer,
            /* .get_alignment    = */ ggml_backend_cpu_buffer_type_get_alignment,
            /* .get_alloc_size   = */ NULL, // defaults to ggml_nbytes
            /* .supports_backend = */ ggml_backend_cpu_buffer_type_supports_backend,
            /* .is_host          = */ ggml_backend_cpu_buffer_type_is_host,
        },
        /* .context  = */ NULL,
    };

    return &ggml_backend_cpu_buffer_type_hbm;
}
#endif
// 结束条件编译指令

struct ggml_backend_cpu_context {
    int n_threads;
    void * work_data;
    size_t work_size;
};
// 定义 CPU 后端上下文结构体

static const char * ggml_backend_cpu_name(ggml_backend_t backend) {
    // 返回 CPU 后端名称
    return "CPU";

    GGML_UNUSED(backend);
}

static void ggml_backend_cpu_free(ggml_backend_t backend) {
    // 释放 CPU 后端资源
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;
    free(cpu_ctx->work_data);
    free(cpu_ctx);
    free(backend);
}
# 获取默认的 CPU 缓冲区类型
static ggml_backend_buffer_type_t ggml_backend_cpu_get_default_buffer_type(ggml_backend_t backend) {
    # 调用 ggml_backend_cpu_buffer_type 函数获取默认的 CPU 缓冲区类型
    return ggml_backend_cpu_buffer_type();

    # 标记参数 backend 未使用
    GGML_UNUSED(backend);
}

# 创建 CPU 图计划
static ggml_backend_graph_plan_t ggml_backend_cpu_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    # 将 backend 的上下文转换为 ggml_backend_cpu_context 类型
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    # 分配内存以存储 ggml_backend_plan_cpu 结构体
    struct ggml_backend_plan_cpu * cpu_plan = malloc(sizeof(struct ggml_backend_plan_cpu));

    # 使用 ggml_graph_plan 函数创建 cplan
    cpu_plan->cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);
    # 将 cgraph 的内容复制到 cpu_plan->cgraph 中（深拷贝）
    cpu_plan->cgraph = *cgraph; // FIXME: deep copy

    # 如果 cplan 的工作大小大于 0，则分配内存以存储工作数据
    if (cpu_plan->cplan.work_size > 0) {
        cpu_plan->cplan.work_data = malloc(cpu_plan->cplan.work_size);
    }

    # 返回 cpu_plan
    return cpu_plan;
}

# 释放 CPU 图计划
static void ggml_backend_cpu_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 将 plan 转换为 ggml_backend_plan_cpu 类型
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    # 释放工作数据的内存
    free(cpu_plan->cplan.work_data);
    # 释放 cpu_plan 的内存
    free(cpu_plan);

    # 标记参数 backend 未使用
    GGML_UNUSED(backend);
}

# 计算 CPU 图计划
static void ggml_backend_cpu_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    # 将 plan 转换为 ggml_backend_plan_cpu 类型
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    # 调用 ggml_graph_compute 函数计算 cgraph 和 cplan
    ggml_graph_compute(&cpu_plan->cgraph, &cpu_plan->cplan);

    # 标记参数 backend 未使用
    GGML_UNUSED(backend);
}

# 计算 CPU 图计算
static bool ggml_backend_cpu_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    # 将 backend 的上下文转换为 ggml_backend_cpu_context 类型
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    # 使用 ggml_graph_plan 函数创建 cplan
    struct ggml_cplan cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);

    # 如果 cpu_ctx 的工作大小小于 cplan 的工作大小，则重新分配内存
    if (cpu_ctx->work_size < cplan.work_size) {
        # 重新分配内存以存储工作数据，并更新工作大小
        cpu_ctx->work_data = realloc(cpu_ctx->work_data, cplan.work_size);
        cpu_ctx->work_size = cplan.work_size;
    }

    # 将 cplan 的工作数据指向 cpu_ctx 的工作数据
    cplan.work_data = cpu_ctx->work_data;
    # 调用 ggml_graph_compute 函数，传入 cgraph 和 cplan 参数，计算图形
    ggml_graph_compute(cgraph, &cplan);
    # 返回 true，表示计算成功
    return true;
static bool ggml_backend_cpu_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    // 检查后端是否支持给定的操作类型
    switch (op->op) {
        // 如果是矩阵乘法操作
        case GGML_OP_MUL_MAT:
            // 返回操作的第二个输入张量类型是否为 GGML_TYPE_F32 或者与第一个输入张量类型对应的点乘类型
            return op->src[1]->type == GGML_TYPE_F32 || op->src[1]->type == ggml_internal_get_type_traits(op->src[0]->type).vec_dot_type;
        // 对于其他操作类型，默认返回 true
        default:
            return true;
    }

    // 忽略未使用的参数
    GGML_UNUSED(backend);
}

static struct ggml_backend_i cpu_backend_i = {
    /* .get_name                = */ ggml_backend_cpu_name,  // 获取后端名称的函数
    /* .free                    = */ ggml_backend_cpu_free,  // 释放后端资源的函数
    /* .get_default_buffer_type = */ ggml_backend_cpu_get_default_buffer_type,  // 获取默认缓冲区类型的函数
    /* .set_tensor_async        = */ NULL,  // 设置异步张量的函数
    /* .get_tensor_async        = */ NULL,  // 获取异步张量的函数
    /* .cpy_tensor_from_async   = */ NULL,  // 从异步张量复制数据的函数
    /* .cpy_tensor_to_async     = */ NULL,  // 复制数据到异步张量的函数
    /* .synchronize             = */ NULL,  // 同步函数
    /* .graph_plan_create       = */ ggml_backend_cpu_graph_plan_create,  // 创建图计划的函数
    /* .graph_plan_free         = */ ggml_backend_cpu_graph_plan_free,  // 释放图计划资源的函数
    /* .graph_plan_compute      = */ ggml_backend_cpu_graph_plan_compute,  // 执行图计划的函数
    /* .graph_compute           = */ ggml_backend_cpu_graph_compute,  // 执行图计算的函数
    /* .supports_op             = */ ggml_backend_cpu_supports_op,  // 检查后端是否支持给定操作类型的函数
};

ggml_backend_t ggml_backend_cpu_init(void) {
    // 分配 CPU 后端上下文的内存
    struct ggml_backend_cpu_context * ctx = malloc(sizeof(struct ggml_backend_cpu_context));

    // 初始化 CPU 后端上下文
    ctx->n_threads = GGML_DEFAULT_N_THREADS;
    ctx->work_data = NULL;
    ctx->work_size = 0;

    // 分配 CPU 后端的内存
    ggml_backend_t cpu_backend = malloc(sizeof(struct ggml_backend));

    // 初始化 CPU 后端
    *cpu_backend = (struct ggml_backend) {
        /* .interface = */ cpu_backend_i,  // 后端接口
        /* .context   = */ ctx  // 后端上下文
    };
    return cpu_backend;  // 返回 CPU 后端
}

bool ggml_backend_is_cpu(ggml_backend_t backend) {
    // 检查给定的后端是否为 CPU 后端
    return backend->iface.get_name == ggml_backend_cpu_name;
}

void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads) {
    // 断言给定的后端为 CPU 后端
    GGML_ASSERT(ggml_backend_is_cpu(backend_cpu));

    // 获取 CPU 后端上下文
    struct ggml_backend_cpu_context * ctx = (struct ggml_backend_cpu_context *)backend_cpu->context;
}
    # 将结构体 ctx 中的 n_threads 成员赋值为变量 n_threads 的值
    ctx->n_threads = n_threads;
// 从指针和大小创建一个 CPU 缓冲区对象
ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(void * ptr, size_t size) {
    return ggml_backend_buffer_init(ggml_backend_cpu_buffer_type(), cpu_backend_buffer_i_from_ptr, ptr, size);
}

// 初始化 CPU 后端注册函数，返回 CPU 后端对象
static ggml_backend_t ggml_backend_reg_cpu_init(const char * params, void * user_data) {
    return ggml_backend_cpu_init();

    // 忽略未使用的参数
    GGML_UNUSED(params);
    GGML_UNUSED(user_data);
}

// 调度器

// 定义最大后端数量
#define GGML_MAX_BACKENDS 4
// 定义最大分割数量
#define GGML_MAX_SPLITS 256
// 定义最大分割输入数量
#define GGML_MAX_SPLIT_INPUTS 16

// 定义分割结构体
struct ggml_backend_sched_split {
    ggml_tallocr_t tallocr;
    int i_start;
    int i_end;
    struct ggml_tensor * inputs[GGML_MAX_SPLIT_INPUTS];
    int n_inputs;
    struct ggml_cgraph graph;
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
    __attribute__((aligned(GGML_MEM_ALIGN)))
    #endif
    char context_buffer[GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS*sizeof(struct ggml_tensor) + sizeof(struct ggml_cgraph)];
};

// 定义哈希 ID 宏
#define hash_id(node) ggml_hash_find_or_insert(sched->hash_set, node)
// 定义节点分配器宏
#define node_allocr(node) sched->node_talloc[hash_id(node)]

// 判断操作是否为视图操作，返回布尔值
static bool ggml_is_view_op(enum ggml_op op) {
    return op == GGML_OP_VIEW || op == GGML_OP_RESHAPE || op == GGML_OP_PERMUTE || op == GGML_OP_TRANSPOSE;
}

// 返回后端的优先级，数值越小优先级越高
static int sched_backend_prio(ggml_backend_sched_t sched, ggml_backend_t backend) {
    # 遍历调度器中的后端列表
    for (int i = 0; i < sched->n_backends; i++) {
        # 如果当前后端与指定的后端相同，则返回当前索引
        if (sched->backends[i] == backend) {
            return i;
        }
    }
    # 如果未找到匹配的后端，则返回最大整数值
    return INT_MAX;
// 分配给定分配器的优先级，返回其索引值
static int sched_allocr_prio(ggml_backend_sched_t sched, ggml_tallocr_t allocr) {
    // 遍历后端数组，查找给定分配器的索引值
    for (int i = 0; i < sched->n_backends; i++) {
        if (sched->tallocs[i] == allocr) {
            return i;
        }
    }
    // 如果未找到，返回最大整数值
    return INT_MAX;
}

// 获取支持缓冲区类型的最高优先级后端
static ggml_backend_t get_buffer_backend(ggml_backend_sched_t sched, ggml_backend_buffer_t buffer) {
    // 如果缓冲区为空，返回空指针
    if (buffer == NULL) {
        return NULL;
    }
    // 查找支持缓冲区类型的最高优先级后端
    for (int i = 0; i < sched->n_backends; i++) {
        if (ggml_backend_buft_supports_backend(buffer->buft, sched->backends[i])) {
            return sched->backends[i];
        }
    }
    // 如果未找到支持的后端，抛出断言错误
    GGML_ASSERT(false && "tensor buffer type not supported by any backend");
}

// 获取支持分配器的最高优先级后端
static ggml_backend_t get_allocr_backend(ggml_backend_sched_t sched, ggml_tallocr_t allocr) {
    // 如果分配器为空，返回空指针
    if (allocr == NULL) {
        return NULL;
    }
    // 查找支持分配器的最高优先级后端
    for (int i = 0; i < sched->n_backends; i++) {
        if (sched->tallocs[i] == allocr) {
            return sched->backends[i];
        }
    }
    // 如果未找到支持的后端，抛出不可达错误
    GGML_UNREACHABLE();
}

// 定义 SET_CAUSE 和 GET_CAUSE 宏
#if 0
static char causes[GGML_DEFAULT_GRAPH_SIZE*8 + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS][128]; // debug, remove
#define SET_CAUSE(node, ...) sprintf(causes[hash_id(node)], __VA_ARGS__)
#define GET_CAUSE(node) causes[hash_id(node)]
#else
#define SET_CAUSE(node, ...)
#define GET_CAUSE(node) ""
#endif

// 根据当前位置返回应该用于节点的后端
static ggml_backend_t sched_backend_from_cur(ggml_backend_sched_t sched, struct ggml_tensor * node) {
    // 如果目标张量已经分配在缓冲区中，必须假设保持在那里是关键的
    // 即 kv 缓存更新
    // 注意，这不允许回退到 CPU。需要将输出张量添加到拆分中，将数据复制回原始后端。
    // 目标张量
    ggml_backend_t cur_backend = get_buffer_backend(sched, node->buffer);
    // 如果当前后端不为空，则设置错误原因并返回当前后端
    if (cur_backend != NULL) {
        SET_CAUSE(node, "1.dst");
        return cur_backend;
    }

    // 如果节点的视图源不为空，并且获取缓冲区后端不为空，则设置错误原因并返回获取缓冲区后端
    if (node->view_src != NULL && get_buffer_backend(sched, node->view_src->buffer) != NULL) {
        SET_CAUSE(node, "1.vsrc");
        return get_buffer_backend(sched, node->view_src->buffer);
    }

    // 初始化当前优先级和当前大小
    int cur_prio = INT_MAX;
    size_t cur_size = 0;

    // 遍历节点的源数组
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        const struct ggml_tensor * src = node->src[i];
        // 如果源为空，则跳出循环
        if (src == NULL) {
            break;
        }
        // 获取缓冲区后端
        ggml_backend_t src_backend = get_buffer_backend(sched, src->buffer);
        // 如果缓冲区后端不为空
        if (src_backend != NULL) {
            // 获取源的优先级和大小
            int src_prio = sched_backend_prio(sched, src_backend);
            size_t src_size = ggml_nbytes(src);
            // 如果源的优先级小于当前优先级并且大小大于等于当前大小
            if (src_prio < cur_prio && src_size >= cur_size) {
                // 更新当前优先级、大小和后端，并设置错误原因
                cur_prio = src_prio;
                cur_size = src_size;
                cur_backend = src_backend;
                SET_CAUSE(node, "1.src%d", i);
            }
        }
    }
    // 返回当前后端
    return cur_backend;
# 定义一个静态函数，用于格式化文件大小
static char * fmt_size(size_t size) {
    # 定义一个静态字符数组作为缓冲区
    static char buffer[128];
    # 如果文件大小大于等于1MB，则格式化为M为单位的字符串
    if (size >= 1024*1024) {
        sprintf(buffer, "%zuM", size/1024/1024);
    } 
    # 否则格式化为K为单位的字符串
    else {
        sprintf(buffer, "%zuK", size/1024);
    }
    # 返回格式化后的字符串
    return buffer;
}

# 定义一个静态函数，用于打印调度分配
static void sched_print_assignments(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    # 初始化当前分割值为0
    int cur_split = 0;
    # 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        # 如果当前节点是分割节点的起始节点
        if (cur_split < sched->n_splits && i == sched->splits[cur_split].i_start) {
            # 获取分割节点的分配器后端
            ggml_backend_t split_backend = get_allocr_backend(sched, sched->splits[cur_split].tallocr);
            # 输出分割节点信息
            fprintf(stderr, "\n## SPLIT #%d: %s # %d inputs: ", cur_split, ggml_backend_name(split_backend),
                sched->splits[cur_split].n_inputs);
            # 遍历分割节点的输入
            for (int j = 0; j < sched->splits[cur_split].n_inputs; j++) {
                # 输出分割节点的输入信息
                fprintf(stderr, "[%s (%5.5s)] ", sched->splits[cur_split].inputs[j]->name,
                    fmt_size(ggml_nbytes(sched->splits[cur_split].inputs[j])));
            }
            # 输出换行符
            fprintf(stderr, "\n");
            # 更新当前分割节点索引
            cur_split++;
        }
        # 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        # 如果当前节点是视图操作节点，则跳过
        if (ggml_is_view_op(node->op)) {
            continue;
        }
        # 获取当前节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        # 获取当前节点的后端
        ggml_backend_t node_backend = node_allocr ? get_allocr_backend(sched, node_allocr) : NULL; // FIXME:
        # 输出当前节点信息
        fprintf(stderr, "node #%3d (%10.10s): %20.20s (%4.4s) [%4.4s %8.8s]:", i, ggml_op_name(node->op), node->name,
            fmt_size(ggml_nbytes(node)), node_allocr ? ggml_backend_name(node_backend) : "NULL", GET_CAUSE(node));
        # 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * src = node->src[j];
            # 如果源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            # 获取源节点的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            # 获取源节点的后端
            ggml_backend_t src_backend = src_allocr ? get_allocr_backend(sched, src_allocr) : NULL;
            # 输出源节点信息
            fprintf(stderr, " %20.20s (%4.4s) [%4.4s %8.8s]", src->name,
                fmt_size(ggml_nbytes(src)), src_backend ? ggml_backend_name(src_backend) : "NULL", GET_CAUSE(src));
        }
        # 输出换行符
        fprintf(stderr, "\n");
    }
// 创建一个具有相同内存布局的张量副本
static struct ggml_tensor * ggml_dup_tensor_layout(struct ggml_context * ctx, const struct ggml_tensor * tensor) {
    // 复制张量
    struct ggml_tensor * dup = ggml_dup_tensor(ctx, tensor);
    // 遍历张量的维度
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        // 将副本的维度设置为原张量的维度
        dup->nb[i] = tensor->nb[i];
    }
    // 返回副本
    return dup;
}

// 为操作分配后端，并将图拆分为可以在相同后端上计算的子图
// TODO: 合并通道
static void sched_split_graph(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    // 重置状态
    size_t hash_size = sched->hash_set.size;
    memset(sched->hash_set.keys, 0, sizeof(sched->hash_set.keys[0]) * hash_size);
    memset(sched->node_talloc,   0, sizeof(sched->node_talloc[0])   * hash_size);
    memset(sched->node_copies,   0, sizeof(sched->node_copies[0])   * hash_size);
    sched->n_splits = 0;

    // 初始化参数
    struct ggml_init_params params = {
        /* .mem_size =   */ sizeof(sched->context_buffer),
        /* .mem_buffer = */ sched->context_buffer,
        /* .no_alloc =   */ true
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
        if (node_allocr(leaf) != NULL) {
            // 不覆盖用户分配
            continue;
        }
        ggml_backend_t leaf_backend = get_buffer_backend(sched, leaf->buffer);
        if (leaf_backend == NULL && leaf->view_src != NULL) {
            leaf_backend = get_buffer_backend(sched, leaf->view_src->buffer);
        }
        if (leaf_backend != NULL) {
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
                    // 获取源节点的优先级和大小
                    int    src_prio = sched_allocr_prio(sched, src_allocr);
                    size_t src_size = ggml_nbytes(src);
                    // 如果源节点的优先级小于当前优先级且大小大于等于当前大小
                    if (src_prio < cur_prio && src_size >= cur_size) {
                        // 更新当前优先级、大小和分配器
                        cur_prio = src_prio;
                        cur_size = src_size;
                        node_allocr = src_allocr;
                        SET_CAUSE(node, "2.src%d", j);
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
        // 遍历当前节点的输入节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前输入节点
            struct ggml_tensor * src = node->src[j];
            // 如果输入节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 为当前输入节点创建分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果输入节点的分配器为空
            if (src_allocr == NULL) {
                // 将当前节点的分配器赋值给输入节点的分配器
                node_allocr(src) = node_allocr;
            }
        }
    }
    //printf("PASS 3 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);

    // pass 4: split graph, find tensors that need to be copied
    // TODO:
    //  - when switching from a less preferred backend to a more preferred backend, check if it is possible to move the switch to an earlier point for the same cost
    // find first backend
    // 初始化当前分割点的索引
    int cur_split = 0;
    // 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果当前节点的视图源为空
        if (node->view_src == NULL) {
            // 设置调度器的分割点的分配器为当前节点的分配器
            sched->splits[0].tallocr = node_allocr(node);
            // 跳出循环
            break;
        }
    }
    // 设置当前分割点的起始索引
    sched->splits[0].i_start = 0;
    // 设置当前分割点的输入数量
    sched->splits[0].n_inputs = 0;
    // 将当前分割点的输入数组清零
    memset(sched->splits[0].inputs, 0, sizeof(sched->splits[0].inputs)); //HACK
    // 获取当前分割点的分配器
    ggml_tallocr_t cur_allocr = sched->splits[0].tallocr;
    // 获取当前分割点的后端 ID
    size_t cur_backend_id = sched_allocr_prio(sched, cur_allocr);
    // 遍历图中的节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];

        // 如果当前节点是视图操作，则跳过
        if (ggml_is_view_op(node->op)) {
            continue;
        }

        // 获取当前节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);

        // 如果当前节点的分配器与上一个节点的分配器不同
        if (node_allocr != cur_allocr) {
            // 更新上一个分割点的结束索引
            sched->splits[cur_split].i_end = i;
            cur_split++;
            GGML_ASSERT(cur_split < GGML_MAX_SPLITS);
            // 更新当前分割点的信息
            sched->splits[cur_split].tallocr = node_allocr;
            sched->splits[cur_split].i_start = i;
            sched->splits[cur_split].n_inputs = 0;
            // 将当前分割点的输入数组清零
            memset(sched->splits[cur_split].inputs, 0, sizeof(sched->splits[cur_split].inputs)); //HACK
            cur_allocr = node_allocr;
            // 根据当前分配器获取后端 ID
            cur_backend_id = sched_allocr_prio(sched, cur_allocr);
        }

        // 查找不在同一后端的输入
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前节点的输入
            struct ggml_tensor * src = node->src[j];
            // 如果输入为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取输入的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果输入的分配器与当前节点的分配器不同
            if (src_allocr != node_allocr) {
                // 更新当前分割点的输入数组
                int n_inputs = sched->splits[cur_split].n_inputs++;
                GGML_ASSERT(n_inputs < GGML_MAX_SPLIT_INPUTS);
                sched->splits[cur_split].inputs[n_inputs] = (struct ggml_tensor *)src;

                // 创建输入的副本
                size_t id = hash_id(src);
                if (sched->node_copies[id][cur_backend_id] == NULL) {
                    // 复制输入的张量布局
                    struct ggml_tensor * tensor_copy = ggml_dup_tensor_layout(sched->ctx, src);
                    sched->node_copies[id][cur_backend_id] = tensor_copy;
                    // 更新副本的分配器
                    node_allocr(tensor_copy) = cur_allocr;
                    // 获取分配器对应的后端
                    ggml_backend_t backend = get_allocr_backend(sched, cur_allocr);
                    // 格式化副本的名称
                    ggml_format_name(tensor_copy, "%s#%s", ggml_backend_name(backend), src->name);
                }
                // 更新当前节点的输入为副本
                node->src[j] = sched->node_copies[id][cur_backend_id];
            }
        }
    }
    // 设置当前分割的结束节点为图中的节点数
    sched->splits[cur_split].i_end = graph->n_nodes;
    // 设置分割的数量为当前分割数加一
    sched->n_splits = cur_split + 1;

    // 打印调试信息，输出当前分配情况
    //fprintf(stderr, "PASS 4 ASSIGNMENTS\n"); sched_print_assignments(sched, graph); fflush(stdout);
#if 1
    // 如果条件为真，则执行以下代码块
    // 对于图中的每个节点进行遍历
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取当前节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 如果当前节点没有分配器，则输出错误信息
        if (node_allocr == NULL) {
            fprintf(stderr, "!!!!!!! %s has no backend\n", node->name);
        }
        // 对当前节点的每个源进行遍历
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前源节点
            struct ggml_tensor * src = node->src[j];
            // 如果当前源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取当前源节点的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果当前源节点的分配器与当前节点的分配器不同，则输出错误信息
            if (src_allocr != node_allocr /* && src_backend != NULL */) { // ignore nulls for now
                fprintf(stderr, "!!!! %s has backend %s, src %d (%s) has backend %s\n",
                    node->name, node_allocr ? ggml_backend_name(get_allocr_backend(sched, node_allocr)) : "NULL",
                    j, src->name, src_allocr ? ggml_backend_name(get_allocr_backend(sched, src_allocr)) : "NULL");
            }
        }
    }
#endif

    // 为每个拆分创建图的副本
    // FIXME: 避免进行此复制，以某种其他方式将拆分输入传递给 ggml_gallocr_alloc_graph_n
    // 创建一个新的图副本，该副本包含原图节点数加上拆分数乘以最大拆分输入数的节点数
    struct ggml_cgraph * graph_copy = ggml_new_graph_custom(sched->ctx, graph->n_nodes + sched->n_splits*GGML_MAX_SPLIT_INPUTS, false);
    // 遍历调度器的分割数量
    for (int i = 0; i < sched->n_splits; i++) {
        // 获取当前分割的指针
        struct ggml_backend_sched_split * split = &sched->splits[i];
        // 为当前分割设置图形视图
        split->graph = ggml_graph_view(graph, split->i_start, split->i_end);

        // 将输入添加到图形副本中，以便它们在分割开始时由 ggml-alloc 分配
        for (int j = 0; j < split->n_inputs; j++) {
            // 获取当前输入的指针
            struct ggml_tensor * input = split->inputs[j];
            // 获取当前输入的副本
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(input)][sched_allocr_prio(sched, split->tallocr)];
            // 设置副本的源为当前输入
            input_cpy->src[0] = input;
            // 将副本添加到图形副本的节点中
            graph_copy->nodes[graph_copy->n_nodes++] = input_cpy;
        }

        // 遍历当前分割的起始索引到结束索引之间的节点
        for (int j = split->i_start; j < split->i_end; j++) {
            // 将原始图形的节点添加到图形副本的节点中
            graph_copy->nodes[graph_copy->n_nodes++] = graph->nodes[j];
        }
    }
    // 将图形副本设置为调度器的图形
    sched->graph = graph_copy;
# 分配调度所需的资源
static void sched_alloc_splits(ggml_backend_sched_t sched) {
    # 为图形分配资源
    ggml_gallocr_alloc_graph_n(
        sched->galloc,
        sched->graph,
        sched->hash_set,
        sched->node_talloc);
}

# 计算分配的资源
static void sched_compute_splits(ggml_backend_sched_t sched) {
    # 初始化用于复制和计算的时间数组
    uint64_t copy_us[GGML_MAX_BACKENDS] = {0};
    uint64_t compute_us[GGML_MAX_BACKENDS] = {0};

    # 获取调度的分割
    struct ggml_backend_sched_split * splits = sched->splits;

    # 遍历所有分割
    for (int i = 0; i < sched->n_splits; i++) {
        # 获取当前分割
        struct ggml_backend_sched_split * split = &splits[i];
        # 获取分割的后端
        ggml_backend_t split_backend = get_allocr_backend(sched, split->tallocr);
        int split_backend_id = sched_backend_prio(sched, split_backend);

        # 复制输入张量到分割的后端
        uint64_t copy_start_us = ggml_time_us();
        for (int j = 0; j < split->n_inputs; j++) {
            # 获取当前输入张量
            struct ggml_tensor * input = split->inputs[j];
            # 获取当前输入张量的副本
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(input)][sched_backend_prio(sched, split_backend)];
            # 如果输入张量没有缓冲区且没有视图源，则报错并退出
            if (input->buffer == NULL) {
                if (input->view_src == NULL) {
                    fprintf(stderr, "input %s has no buffer and no view_src\n", input->name);
                    exit(1);
                }
                # 可能需要使用调度缓冲区
                ggml_backend_view_init(input->view_src->buffer, input);
            }
            # 如果输入张量的副本没有缓冲区，则报错并退出
            if (input_cpy->buffer == NULL) {
                fprintf(stderr, "input_cpy %s has no buffer\n", input_cpy->name);
                exit(1);
            }
            # 复制输入张量到副本
            ggml_backend_tensor_copy(input, input_cpy);
        }
        # 计算复制时间
        int64_t copy_end_us = ggml_time_us();
        copy_us[split_backend_id] += copy_end_us - copy_start_us;
#if 0
        // 创建一个文件名字符串，格式为"split_序号_后端名称.dot"
        char split_filename[GGML_MAX_NAME];
        snprintf(split_filename, GGML_MAX_NAME, "split_%i_%s.dot", i, ggml_backend_name(split_backend));
        // 将分割后的图形以DOT格式写入文件
        ggml_graph_dump_dot(split->graph, NULL, split_filename);
#endif

        // 计算开始时间
        uint64_t compute_start_us = ggml_time_us();
        // 使用指定后端计算分割后的图形
        ggml_backend_graph_compute(split_backend, &split->graph);
        // 计算结束时间
        uint64_t compute_end_us = ggml_time_us();
        // 计算时间差，加入到对应后端的计算时间中
        compute_us[split_backend_id] += compute_end_us - compute_start_us;
    }

#if 0
    // 每个后端的计时信息
    fprintf(stderr, "sched_compute_splits times (%d splits):\n", sched->n_splits);
    for (int i = 0; i < sched->n_backends; i++) {
        if (copy_us[i] > 0 || compute_us[i] > 0) {
            // 输出每个后端的复制和计算时间
            fprintf(stderr, "\t%5.5s: %lu us copy, %lu us compute\n", ggml_backend_name(sched->backends[i]), copy_us[i], compute_us[i]);
        }
    }
#endif
}

static void sched_reset(ggml_backend_sched_t sched) {
    // 重置每个后端的分配器
    for (int i = 0; i < sched->n_backends; i++) {
        ggml_tallocr_reset(sched->tallocs[i]);
    }
}

ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, int n_backends) {
    // 确保后端数量不超过最大值
    GGML_ASSERT(n_backends <= GGML_MAX_BACKENDS);

    // 分配后端调度器内存
    struct ggml_backend_sched * sched = malloc(sizeof(struct ggml_backend_sched));
    // 将内存清零
    memset(sched, 0, sizeof(struct ggml_backend_sched));

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

void ggml_backend_sched_free(ggml_backend_sched_t sched) {
    if (sched == NULL) {
        return;
    }
    // 释放每个后端的测量分配器
    for (int i = 0; i < sched->n_backends; i++) {
        ggml_tallocr_free(sched->tallocs[i]);
    }
    // 释放全局分配器
    ggml_gallocr_free(sched->galloc);
    // 释放后端调度器内存
    free(sched->hash_set.keys);
    # 释放调度器中节点分配的内存
    free(sched->node_talloc);
    # 释放调度器中节点副本分配的内存
    free(sched->node_copies);
    # 释放调度器对象的内存
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
}

// 初始化视图的后端缓冲区
void ggml_backend_view_init(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    GGML_ASSERT(tensor->buffer == NULL); // 确保张量的缓冲区为空
    //GGML_ASSERT(tensor->data == NULL); // 预先分配张量的视图可能已经设置了数据，但仍需要初始化
    GGML_ASSERT(tensor->view_src != NULL); // 确保张量的视图源不为空
    GGML_ASSERT(tensor->view_src->buffer != NULL); // 确保张量的视图源的缓冲区不为空
    GGML_ASSERT(tensor->view_src->data != NULL); // 确保张量的视图源的数据不为空

    tensor->buffer = buffer; // 设置张量的缓冲区
    tensor->data = (char *)tensor->view_src->data + tensor->view_offs; // 设置张量的数据
    tensor->backend = tensor->view_src->backend; // 设置张量的后端
    ggml_backend_buffer_init_tensor(buffer, tensor); // 初始化缓冲区的张量
}

// 分配张量的后端缓冲区
void ggml_backend_tensor_alloc(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor, void * addr) {
    GGML_ASSERT(tensor->buffer == NULL); // 确保张量的缓冲区为空
    GGML_ASSERT(tensor->data == NULL); // 确保张量的数据为空
    GGML_ASSERT(tensor->view_src == NULL); // 确保张量的视图源为空
    GGML_ASSERT(addr >= ggml_backend_buffer_get_base(buffer)); // 确保地址大于等于缓冲区的基地址
    GGML_ASSERT((char *)addr + ggml_backend_buffer_get_alloc_size(buffer, tensor) <=
                (char *)ggml_backend_buffer_get_base(buffer) + ggml_backend_buffer_get_size(buffer)); // 确保地址加上分配大小不超过缓冲区的大小

    tensor->buffer = buffer; // 设置张量的缓冲区
    tensor->data = addr; // 设置张量的数据
    ggml_backend_buffer_init_tensor(buffer, tensor); // 初始化缓冲区的张量
}

// 复制张量的图形
static struct ggml_tensor * graph_dup_tensor(struct ggml_hash_set hash_set, struct ggml_tensor ** node_copies,
    struct ggml_context * ctx_allocated, struct ggml_context * ctx_unallocated, struct ggml_tensor * src) {
    GGML_ASSERT(src != NULL); // 确保源张量不为空
    GGML_ASSERT(src->data && "graph must be allocated"); // 确保图形已分配

    size_t id = ggml_hash_insert(hash_set, src); // 将源张量插入哈希集合
    if (id == GGML_HASHTABLE_ALREADY_EXISTS) {
        return node_copies[ggml_hash_find(hash_set, src)]; // 如果已存在，则返回节点副本
    }

    struct ggml_tensor * dst = ggml_dup_tensor_layout(src->data && !src->view_src ? ctx_allocated : ctx_unallocated, src); // 复制张量的布局
    if (src->view_src != NULL) {
        dst->view_src = graph_dup_tensor(hash_set, node_copies, ctx_allocated, ctx_unallocated, src->view_src); // 复制张量的视图源
        dst->view_offs = src->view_offs; // 设置目标张量的视图偏移量
    }
}
    // 将源操作符赋值给目标操作符
    dst->op = src->op;
    // 将源操作参数的内容复制到目标操作参数中
    memcpy(dst->op_params, src->op_params, sizeof(dst->op_params));
    // 设置目标节点的名称为源节点的名称
    ggml_set_name(dst, src->name);

    // 复制源节点
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        // 获取源节点的第i个输入张量
        struct ggml_tensor * s = src->src[i];
        // 如果输入张量为空，则跳出循环
        if (s == NULL) {
            break;
        }
        // 复制输入张量并赋值给目标节点的第i个输入张量
        dst->src[i] = graph_dup_tensor(hash_set, node_copies, ctx_allocated, ctx_unallocated, s);
    }

    // 将目标节点添加到节点副本的哈希表中
    node_copies[id] = dst;
    // 返回目标节点
    return dst;
// 初始化张量图中的节点，确保每个节点只被初始化一次
static void graph_init_tensor(struct ggml_hash_set hash_set, struct ggml_tensor ** node_copies, bool * node_init, struct ggml_tensor * src) {
    // 查找节点在哈希集合中的索引
    size_t id = ggml_hash_find(hash_set, src);
    // 如果节点已经被初始化，则直接返回
    if (node_init[id]) {
        return;
    }
    // 标记节点已经被初始化
    node_init[id] = true;

    // 获取节点的副本
    struct ggml_tensor * dst = node_copies[id];
    // 如果节点的视图源不为空，则初始化视图
    if (dst->view_src != NULL) {
        ggml_backend_view_init(dst->view_src->buffer, dst);
    }
    // 否则，复制数据到目标节点
    else {
        ggml_backend_tensor_copy(src, dst);
    }

    // 初始化节点的源节点
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        struct ggml_tensor * s = src->src[i];
        // 如果源节点为空，则跳出循环
        if (s == NULL) {
            break;
        }
        // 递归初始化源节点
        graph_init_tensor(hash_set, node_copies, node_init, s);
    }
}

// 复制图的后端表示
struct ggml_backend_graph_copy ggml_backend_graph_copy(ggml_backend_t backend, struct ggml_cgraph * graph) {
    // 初始化哈希集合
    struct ggml_hash_set hash_set = {
        /* .size = */ graph->visited_hash_table.size,
        /* .keys = */ calloc(sizeof(hash_set.keys[0]) * graph->visited_hash_table.size, 1)
    };
    // 分配节点副本数组
    struct ggml_tensor ** node_copies = calloc(sizeof(node_copies[0]) * hash_set.size, 1);
    // 初始化节点是否被初始化的标志数组
    bool * node_init = calloc(sizeof(node_init[0]) * hash_set.size, 1);

    // 初始化参数
    struct ggml_init_params params = {
        /* .mem_size   = */ ggml_tensor_overhead()*hash_set.size + ggml_graph_overhead_custom(graph->size, false),
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ true
    };

    // 分配两个上下文
    struct ggml_context * ctx_allocated = ggml_init(params);
    struct ggml_context * ctx_unallocated = ggml_init(params);

    // 复制节点
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        graph_dup_tensor(hash_set, node_copies, ctx_allocated, ctx_unallocated, node);
    }

    // 分配节点的内存
    ggml_backend_buffer_t buffer = ggml_backend_alloc_ctx_tensors(ctx_allocated, backend);

    // 复制数据并初始化视图
    // 遍历图中的节点，对每个节点进行初始化
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 初始化节点的拷贝并加入哈希集合
        graph_init_tensor(hash_set, node_copies, node_init, node);
    }

    // 构建图的拷贝
    struct ggml_cgraph * graph_copy = ggml_new_graph_custom(ctx_allocated, graph->size, false);
    // 遍历原图的节点，将节点的拷贝加入到图的拷贝中
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取原图的节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取节点的拷贝
        struct ggml_tensor * node_copy = node_copies[ggml_hash_find(hash_set, node)];
        // 将节点的拷贝加入到图的拷贝中
        graph_copy->nodes[i] = node_copy;
    }
    // 设置图的拷贝的节点数量
    graph_copy->n_nodes = graph->n_nodes;

    // 释放哈希集合的键数组
    free(hash_set.keys);
    // 释放节点拷贝数组
    free(node_copies);
    // 释放节点初始化数组
    free(node_init);

    // 返回图的拷贝及相关信息
    return (struct ggml_backend_graph_copy) {
        /* .buffer           = */ buffer,
        /* .ctx_allocated    = */ ctx_allocated,
        /* .ctx_unallocated  = */ ctx_unallocated,
        /* .graph            = */ graph_copy,
    };
// 释放图形拷贝的资源
void ggml_backend_graph_copy_free(struct ggml_backend_graph_copy copy) {
    // 释放缓冲区资源
    ggml_backend_buffer_free(copy.buffer);
    // 释放上下文已分配资源
    ggml_free(copy.ctx_allocated);
    // 释放上下文未分配资源
    ggml_free(copy.ctx_unallocated);
}

// 比较两个后端的图形
void ggml_backend_compare_graph_backend(ggml_backend_t backend1, ggml_backend_t backend2, struct ggml_cgraph * graph, ggml_backend_eval_callback callback, void * user_data) {
    // 复制后端2的图形
    struct ggml_backend_graph_copy copy = ggml_backend_graph_copy(backend2, graph);
    // 获取图形指针
    struct ggml_cgraph * g1 = graph;
    struct ggml_cgraph * g2 = copy.graph;

    // 断言两个图形的节点数相同
    assert(g1->n_nodes == g2->n_nodes);

    // 遍历图形的节点
    for (int i = 0; i < g1->n_nodes; i++) {
        //printf("eval %d/%d\n", i, g1->n_nodes);
        // 获取节点的张量
        struct ggml_tensor * t1 = g1->nodes[i];
        struct ggml_tensor * t2 = g2->nodes[i];

        // 断言两个张量的操作和布局相同
        assert(t1->op == t2->op && ggml_are_same_layout(t1, t2));

        // 创建图形视图
        struct ggml_cgraph g1v = ggml_graph_view(g1, i, i + 1);
        struct ggml_cgraph g2v = ggml_graph_view(g2, i, i + 1);

        // 计算图形的计算结果
        ggml_backend_graph_compute(backend1, &g1v);
        ggml_backend_graph_compute(backend2, &g2v);

        // 如果是视图操作，则继续下一次循环
        if (ggml_is_view_op(t1->op)) {
            continue;
        }

        // 比较结果，计算均方根等
        if (!callback(i, t1, t2, user_data)) {
            break;
        }
    }

    // 释放图形拷贝的资源
    ggml_backend_graph_copy_free(copy);
}
```