# `whisper.cpp\ggml-backend.c`

```cpp
// 包含头文件 ggml-backend-impl.h, ggml-alloc.h, ggml-impl.h
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

// 定义宏 MAX，返回两个值中的最大值
#define MAX(a, b) ((a) > (b) ? (a) : (b)

// 获取 backend buffer 类型的名称
const char * ggml_backend_buft_name(ggml_backend_buffer_type_t buft) {
    return buft->iface.get_name(buft);
}

// 分配指定大小的 backend buffer
GGML_CALL ggml_backend_buffer_t ggml_backend_buft_alloc_buffer(ggml_backend_buffer_type_t buft, size_t size) {
    return buft->iface.alloc_buffer(buft, size);
}

// 获取 backend buffer 类型的对齐方式
size_t ggml_backend_buft_get_alignment(ggml_backend_buffer_type_t buft) {
    return buft->iface.get_alignment(buft);
}

// 获取 backend buffer 类型的最大大小，如果未定义则返回 SIZE_MAX
size_t ggml_backend_buft_get_max_size(ggml_backend_buffer_type_t buft) {
    // get_max_size 是可选的，默认为 SIZE_MAX
    if (buft->iface.get_max_size) {
        return buft->iface.get_max_size(buft);
    }
    return SIZE_MAX;
}

// 获取分配给 backend buffer 的大小，如果未定义则返回 ggml_nbytes
GGML_CALL size_t ggml_backend_buft_get_alloc_size(ggml_backend_buffer_type_t buft, struct ggml_tensor * tensor) {
    // get_alloc_size 是可选的，默认为 ggml_nbytes
    if (buft->iface.get_alloc_size) {
        size_t size = buft->iface.get_alloc_size(buft, tensor);
        assert(size >= ggml_nbytes(tensor));
        return size;
    }
    return ggml_nbytes(tensor);
}

// 检查 backend buffer 类型是否支持指定的 backend
bool ggml_backend_buft_supports_backend(ggml_backend_buffer_type_t buft, ggml_backend_t backend) {
    return buft->iface.supports_backend(buft, backend);
}

// 检查 backend buffer 类型是否为主机类型
bool ggml_backend_buft_is_host(ggml_backend_buffer_type_t buft) {
    if (buft->iface.is_host) {
        return buft->iface.is_host(buft);
    }
    return false;
}

// 初始化 backend buffer
GGML_CALL ggml_backend_buffer_t ggml_backend_buffer_init(
               ggml_backend_buffer_type_t      buft,
        struct ggml_backend_buffer_i           iface,
               ggml_backend_buffer_context_t   context,
               size_t                          size) {
    // 分配内存给 buffer
    ggml_backend_buffer_t buffer = malloc(sizeof(struct ggml_backend_buffer));
    // 初始化一个 ggml_backend_buffer 结构体，并赋值给指针 buffer
    (*buffer) = (struct ggml_backend_buffer) {
        // 设置结构体字段 interface 为 iface
        /* .interface = */ iface,
        // 设置结构体字段 buft 为 buft
        /* .buft      = */ buft,
        // 设置结构体字段 context 为 context
        /* .context   = */ context,
        // 设置结构体字段 size 为 size
        /* .size      = */ size,
        // 设置结构体字段 usage 为 GGML_BACKEND_BUFFER_USAGE_ANY
        /* .usage     = */ GGML_BACKEND_BUFFER_USAGE_ANY
    };

    // 返回初始化后的 buffer 结构体指针
    return buffer;
}

// 获取缓冲区的名称
const char * ggml_backend_buffer_name(ggml_backend_buffer_t buffer) {
    return buffer->iface.get_name(buffer);
}

// 释放缓冲区
void ggml_backend_buffer_free(ggml_backend_buffer_t buffer) {
    if (buffer == NULL) {
        return;
    }

    // 如果释放缓冲区的函数存在，则调用
    if (buffer->iface.free_buffer != NULL) {
        buffer->iface.free_buffer(buffer);
    }
    // 释放缓冲区
    free(buffer);
}

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
GGML_CALL void ggml_backend_buffer_init_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // 如果初始化张量的函数存在，则调用
    if (buffer->iface.init_tensor) {
        buffer->iface.init_tensor(buffer, tensor);
    }
}

// 获取缓冲区的对齐方式
size_t ggml_backend_buffer_get_alignment (ggml_backend_buffer_t buffer) {
    return ggml_backend_buft_get_alignment(ggml_backend_buffer_get_type(buffer));
}

// 获取缓冲区的最大大小
size_t ggml_backend_buffer_get_max_size(ggml_backend_buffer_t buffer) {
    return ggml_backend_buft_get_max_size(ggml_backend_buffer_get_type(buffer));
}

// 获取缓冲区的分配大小
size_t ggml_backend_buffer_get_alloc_size(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    return ggml_backend_buft_get_alloc_size(ggml_backend_buffer_get_type(buffer), tensor);
}

// 清除缓冲区的内容
void ggml_backend_buffer_clear(ggml_backend_buffer_t buffer, uint8_t value) {
    buffer->iface.clear(buffer, value);
}

// 判断缓冲区是否在主机上
bool ggml_backend_buffer_is_host(ggml_backend_buffer_t buffer) {
    return ggml_backend_buft_is_host(ggml_backend_buffer_get_type(buffer));
}

// 设置缓冲区的用途
void ggml_backend_buffer_set_usage(ggml_backend_buffer_t buffer, enum ggml_backend_buffer_usage usage) {
    buffer->usage = usage;

    // FIXME: add a generic callback to the buffer interface
    // 如果缓冲区是多缓冲区，则设置用途
    if (ggml_backend_buffer_is_multi_buffer(buffer)) {
        ggml_backend_multi_buffer_set_usage(buffer, usage);
    }
}
// 获取缓冲区类型
ggml_backend_buffer_type_t ggml_backend_buffer_get_type(ggml_backend_buffer_t buffer) {
    return buffer->buft;
}

// 重置缓冲区
void ggml_backend_buffer_reset(ggml_backend_buffer_t buffer) {
    // 如果存在重置接口，则调用重置函数
    if (buffer->iface.reset) {
        buffer->iface.reset(buffer);
    }
}

// 复制张量数据到目标张量
bool ggml_backend_buffer_copy_tensor(const struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 获取目标缓冲区
    ggml_backend_buffer_t dst_buf = dst->view_src ? dst->view_src->buffer : dst->buffer;
    // 如果存在复制张量接口，则调用复制函数
    if (dst_buf->iface.cpy_tensor) {
        return src->buffer->iface.cpy_tensor(dst_buf, src, dst);
    }
    return false;
}

// 获取后端名称
const char * ggml_backend_name(ggml_backend_t backend) {
    // 如果后端为空，则返回字符串"NULL"
    if (backend == NULL) {
        return "NULL";
    }
    // 调用获取名称接口
    return backend->iface.get_name(backend);
}

// 释放后端资源
void ggml_backend_free(ggml_backend_t backend) {
    // 如果后端为空，则直接返回
    if (backend == NULL) {
        return;
    }
    // 调用释放资源接口
    backend->iface.free(backend);
}

// 获取默认缓冲区类型
ggml_backend_buffer_type_t ggml_backend_get_default_buffer_type(ggml_backend_t backend) {
    return backend->iface.get_default_buffer_type(backend);
}

// 分配缓冲区
ggml_backend_buffer_t ggml_backend_alloc_buffer(ggml_backend_t backend, size_t size) {
    return ggml_backend_buft_alloc_buffer(ggml_backend_get_default_buffer_type(backend), size);
}

// 获取对齐方式
size_t ggml_backend_get_alignment(ggml_backend_t backend) {
    return ggml_backend_buft_get_alignment(ggml_backend_get_default_buffer_type(backend));
}

// 获取最大尺寸
size_t ggml_backend_get_max_size(ggml_backend_t backend) {
    return ggml_backend_buft_get_max_size(ggml_backend_get_default_buffer_type(backend));
}

// 设置异步张量数据
void ggml_backend_tensor_set_async(ggml_backend_t backend, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    // 断言张量已分配内存
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    // 断言写入数据不超出张量边界
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor write out of bounds");

    // 如果不存在设置异步张量接口，则调用同步设置张量函数
    if (backend->iface.set_tensor_async == NULL) {
        ggml_backend_tensor_set(tensor, data, offset, size);
    }
}
    } else {
        # 如果条件不成立，调用后端接口的异步设置张量数据的方法
        backend->iface.set_tensor_async(backend, tensor, data, offset, size);
    }
// 异步获取张量数据，根据给定的偏移量和大小
void ggml_backend_tensor_get_async(ggml_backend_t backend, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 断言张量数据已分配
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    // 断言读取数据不超出张量边界
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor read out of bounds");

    // 如果后端接口中没有异步获取张量数据的函数，则调用同步获取函数
    if (backend->iface.get_tensor_async == NULL) {
        ggml_backend_tensor_get(tensor, data, offset, size);
    } else {
        // 调用后端接口中的异步获取张量数据的函数
        backend->iface.get_tensor_async(backend, tensor, data, offset, size);
    }
}

// 设置张量数据，根据给定的偏移量和大小
GGML_CALL void ggml_backend_tensor_set(struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    // 获取张量对应的缓冲区
    ggml_backend_buffer_t buf = tensor->view_src ? tensor->view_src->buffer : tensor->buffer;

    // 断言张量数据已分配
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    // 断言张量缓冲区已设置
    GGML_ASSERT(buf != NULL && "tensor buffer not set");
    // 断言写入数据不超出张量边界
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor write out of bounds");

    // 调用缓冲区接口中的设置张量数据的函数
    tensor->buffer->iface.set_tensor(buf, tensor, data, offset, size);
}

// 获取张量数据，根据给定的偏移量和大小
GGML_CALL void ggml_backend_tensor_get(const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    // 获取张量对应的缓冲区
    ggml_backend_buffer_t buf = tensor->view_src ? tensor->view_src->buffer : tensor->buffer;

    // 断言张量数据已分配
    GGML_ASSERT(tensor->data != NULL && "tensor not allocated");
    // 断言张量缓冲区已设置
    GGML_ASSERT(tensor->buffer != NULL && "tensor buffer not set");
    // 断言读取数据不超出张量边界
    GGML_ASSERT(offset + size <= ggml_nbytes(tensor) && "tensor read out of bounds");

    // 调用缓冲区接口中的获取张量数据的函数
    tensor->buffer->iface.get_tensor(buf, tensor, data, offset, size);
}

// 同步后端操作
void ggml_backend_synchronize(ggml_backend_t backend) {
    // 如果后端接口中没有同步函数，则直接返回
    if (backend->iface.synchronize == NULL) {
        return;
    }

    // 调用后端接口中的同步函数
    backend->iface.synchronize(backend);
}

// 创建后端图计划
ggml_backend_graph_plan_t ggml_backend_graph_plan_create(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 调用后端接口中的创建图计划函数
    return backend->iface.graph_plan_create(backend, cgraph);
}

// 释放后端图计划
void ggml_backend_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    // 调用后端接口中的释放图计划函数
    backend->iface.graph_plan_free(backend, plan);
}
// 计算图计划的后端实现函数
void ggml_backend_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    // 调用后端接口的图计划计算函数
    backend->iface.graph_plan_compute(backend, plan);
}

// 后端实现函数，计算计算图
bool ggml_backend_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 调用后端接口的图计算函数
    return backend->iface.graph_compute(backend, cgraph);
}

// 后端实现函数，检查后端是否支持某个操作
bool ggml_backend_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    // 调用后端接口的操作支持函数
    return backend->iface.supports_op(backend, op);
}

// 后端拷贝函数，检查两个张量是否具有相同的布局
static bool ggml_are_same_layout(const struct ggml_tensor * a, const struct ggml_tensor * b) {
    // 检查两个张量的类型是否相同
    if (a->type != b->type) {
        return false;
    }
    // 遍历张量的维度，检查每个维度的大小和偏移是否相同
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

// 后端张量拷贝函数
void ggml_backend_tensor_copy(struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 检查源张量和目标张量是否具有相同的布局
    GGML_ASSERT(ggml_are_same_layout(src, dst) && "cannot copy tensors with different layouts");

    // 如果源张量和目标张量相同，则直接返回
    if (src == dst) {
        return;
    }

    // 根据源张量的缓冲区类型进行不同的拷贝操作
    if (ggml_backend_buffer_is_host(src->buffer)) {
        ggml_backend_tensor_set(dst, src->data, 0, ggml_nbytes(src));
    } else if (ggml_backend_buffer_is_host(dst->buffer)) {
        ggml_backend_tensor_get(src, dst->data, 0, ggml_nbytes(src));
    } else if (!ggml_backend_buffer_copy_tensor(src, dst)) {
#ifndef NDEBUG
        // 在调试模式下输出警告信息
        fprintf(stderr, "%s: warning: slow copy from %s to %s\n", __func__, ggml_backend_buffer_name(src->buffer), ggml_backend_buffer_name(dst->buffer));
#endif
        // 分配内存并进行拷贝操作
        size_t nbytes = ggml_nbytes(src);
        void * data = malloc(nbytes);
        ggml_backend_tensor_get(src, data, 0, nbytes);
        ggml_backend_tensor_set(dst, data, 0, nbytes);
        free(data);
    }
}

// 异步后端张量拷贝函数
void ggml_backend_tensor_copy_async(ggml_backend_t backend, struct ggml_tensor * src, struct ggml_tensor * dst) {
    // 检查源张量和目标张量是否具有相同的布局
    GGML_ASSERT(ggml_are_same_layout(src, dst) && "cannot copy tensors with different layouts");
    # 如果源张量和目标张量相同，则直接返回，不进行任何操作
    if (src == dst) {
        return;
    }

    # 检查源张量和目标张量的缓冲区是否支持指定的后端
    if (ggml_backend_buft_supports_backend(src->buffer->buft, backend) && ggml_backend_buft_supports_backend(dst->buffer->buft, backend)) {
        # 如果后端支持异步张量复制操作
        if (backend->iface.cpy_tensor_async != NULL) {
            # 调用后端的异步张量复制函数，如果成功则直接返回
            if (backend->iface.cpy_tensor_async(backend, src, dst)) {
                return;
            }
        }
    }

    # 计算源张量的字节数
    size_t nbytes = ggml_nbytes(src);
    # 如果源张量的缓冲区在主机上
    if (ggml_backend_buffer_is_host(src->buffer)) {
        # 使用后端异步设置张量数据到目标张量的缓冲区中
        ggml_backend_tensor_set_async(backend, dst, src->data, 0, nbytes);
    }
    else {
        # 否则，调用后端的张量复制函数进行数据复制
        ggml_backend_tensor_copy(src, dst);
    }
// 结构体定义，用于存储后端注册信息
struct ggml_backend_reg {
    char name[128]; // 后端名称
    ggml_backend_init_fn init_fn; // 后端初始化函数指针
    ggml_backend_buffer_type_t default_buffer_type; // 默认缓冲类型
    void * user_data; // 用户数据指针
};

// 后端注册表，存储注册的后端信息
static struct ggml_backend_reg ggml_backend_registry[GGML_MAX_BACKENDS_REG];
static size_t ggml_backend_registry_count = 0; // 后端注册表计数

// CPU 后端初始化函数声明
GGML_CALL static ggml_backend_t ggml_backend_reg_cpu_init(const char * params, void * user_data);

// 后端注册表初始化函数
GGML_CALL static void ggml_backend_registry_init(void) {
    static bool initialized = false; // 标记是否已经初始化

    if (initialized) { // 如果已经初始化过，则直接返回
        return;
    }

    initialized = true; // 标记已经初始化

    // 注册 CPU 后端
    ggml_backend_register("CPU", ggml_backend_reg_cpu_init, ggml_backend_cpu_buffer_type(), NULL);

    // 在此处添加前向声明以避免包含后端头文件
#ifdef GGML_USE_CUBLAS
    extern GGML_CALL void ggml_backend_cuda_reg_devices(void);
    ggml_backend_cuda_reg_devices();
#endif

#ifdef GGML_USE_SYCL
    extern void ggml_backend_sycl_reg_devices(void);
    ggml_backend_sycl_reg_devices();
#endif

#ifdef GGML_USE_METAL
    // 注册 Metal 后端
    extern GGML_CALL ggml_backend_t ggml_backend_reg_metal_init(const char * params, void * user_data);
    extern GGML_CALL ggml_backend_buffer_type_t ggml_backend_metal_buffer_type(void);
    ggml_backend_register("Metal", ggml_backend_reg_metal_init, ggml_backend_metal_buffer_type(), NULL);
#endif

#ifdef GGML_USE_VULKAN
    extern GGML_CALL int ggml_backend_vk_reg_devices(void);
    ggml_backend_vk_reg_devices();
#endif

#ifdef GGML_USE_KOMPUTE
    extern GGML_CALL void ggml_backend_kompute_reg_devices(void);
    ggml_backend_kompute_reg_devices();
#endif
}

// 注册后端函数
GGML_CALL void ggml_backend_register(const char * name, ggml_backend_init_fn init_fn, ggml_backend_buffer_type_t default_buffer_type, void * user_data) {
    GGML_ASSERT(ggml_backend_registry_count < GGML_MAX_BACKENDS_REG); // 断言后端注册表计数小于最大注册数

    size_t id = ggml_backend_registry_count; // 获取当前注册表计数
    ggml_backend_registry[id] = (struct ggml_backend_reg) {
        /* 初始化 ggml_backend_registry 中 id 对应的结构体 */
        /* .name                = */ {0},  // 初始化 name 字段为 0
        /* .fn                  = */ init_fn,  // 初始化 fn 字段为 init_fn
        /* .default_buffer_type = */ default_buffer_type,  // 初始化 default_buffer_type 字段为 default_buffer_type
        /* .user_data           = */ user_data,  // 初始化 user_data 字段为 user_data
    };

    // 使用 snprintf 将 name 字段格式化为字符串，存储到 ggml_backend_registry[id].name 中
    snprintf(ggml_backend_registry[id].name, sizeof(ggml_backend_registry[id].name), "%s", name);
// 如果处于调试模式，输出注册后端的信息
#ifndef NDEBUG
    fprintf(stderr, "%s: registered backend %s\n", __func__, name);
#endif

// 增加注册后端计数
ggml_backend_registry_count++;
}

// 获取注册后端的数量
size_t ggml_backend_reg_get_count(void) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    return ggml_backend_registry_count;
}

// 根据名称查找注册后端的索引
size_t ggml_backend_reg_find_by_name(const char * name) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    for (size_t i = 0; i < ggml_backend_registry_count; i++) {
        // TODO: 以可移植的方式进行大小写不敏感比较
        if (strcmp(ggml_backend_registry[i].name, name) == 0) {
            return i;
        }
    }

    // 未找到
    return SIZE_MAX;
}

// 从后端参数字符串初始化后端
ggml_backend_t ggml_backend_reg_init_backend_from_str(const char * backend_str) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    const char * params = strchr(backend_str, ':');
    char backend_name[128];
    if (params == NULL) {
        snprintf(backend_name, sizeof(backend_name), "%s", backend_str);
        params = "";
    } else {
        snprintf(backend_name, sizeof(backend_name), "%.*s", (int)(params - backend_str), backend_str);
        params++;
    }

    size_t backend_i = ggml_backend_reg_find_by_name(backend_name);

    if (backend_i == SIZE_MAX) {
        fprintf(stderr, "%s: backend %s not found\n", __func__, backend_name);
        return NULL;
    }

    return ggml_backend_reg_init_backend(backend_i, params);
}

// 获取指定索引的后端名称
const char * ggml_backend_reg_get_name(size_t i) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    GGML_ASSERT(i < ggml_backend_registry_count);
    return ggml_backend_registry[i].name;
}

// 初始化指定索引的后端
ggml_backend_t ggml_backend_reg_init_backend(size_t i, const char * params) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    GGML_ASSERT(i < ggml_backend_registry_count);
    return ggml_backend_registry[i].init_fn(params, ggml_backend_registry[i].user_data);
}

// 获取指定索引的后端默认缓冲类型
ggml_backend_buffer_type_t ggml_backend_reg_get_default_buffer_type(size_t i) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    GGML_ASSERT(i < ggml_backend_registry_count);
    # 返回 ggml_backend_registry 列表中索引为 i 的元素的 default_buffer_type 属性
    return ggml_backend_registry[i].default_buffer_type;
}

// 分配注册缓冲区
ggml_backend_buffer_t ggml_backend_reg_alloc_buffer(size_t i, size_t size) {
    // 初始化后端注册表
    ggml_backend_registry_init();

    // 断言索引 i 小于注册表中的数量
    GGML_ASSERT(i < ggml_backend_registry_count);
    // 调用函数分配缓冲区
    return ggml_backend_buft_alloc_buffer(ggml_backend_registry[i].default_buffer_type, size);
}

// 后端 CPU

// 返回缓冲区名称
GGML_CALL static const char * ggml_backend_cpu_buffer_name(ggml_backend_buffer_t buffer) {
    return "CPU";

    GGML_UNUSED(buffer);
}

// 返回缓冲区基地址
GGML_CALL static void * ggml_backend_cpu_buffer_get_base(ggml_backend_buffer_t buffer) {
    return (void *)buffer->context;
}

// 释放缓冲区
GGML_CALL static void ggml_backend_cpu_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    free(buffer->context);
}

// 设置张量数据到缓冲区
GGML_CALL static void ggml_backend_cpu_buffer_set_tensor(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor, const void * data, size_t offset, size_t size) {
    memcpy((char *)tensor->data + offset, data, size);

    GGML_UNUSED(buffer);
}

// 从缓冲区获取张量数据
GGML_CALL static void ggml_backend_cpu_buffer_get_tensor(ggml_backend_buffer_t buffer, const struct ggml_tensor * tensor, void * data, size_t offset, size_t size) {
    memcpy(data, (const char *)tensor->data + offset, size);

    GGML_UNUSED(buffer);
}

// 复制张量数据到另一个张量
GGML_CALL static bool ggml_backend_cpu_buffer_cpy_tensor(ggml_backend_buffer_t buffer, const struct ggml_tensor * src, struct ggml_tensor * dst) {
    if (ggml_backend_buffer_is_host(src->buffer)) {
        memcpy(dst->data, src->data, ggml_nbytes(src));
        return true;
    }
    return false;

    GGML_UNUSED(buffer);
}

// 清空缓冲区数据
GGML_CALL static void ggml_backend_cpu_buffer_clear(ggml_backend_buffer_t buffer, uint8_t value) {
    memset(buffer->context, value, buffer->size);
}

// CPU 后端缓冲区接口
static struct ggml_backend_buffer_i cpu_backend_buffer_i = {
    /* .get_name        = */ ggml_backend_cpu_buffer_name,
    /* .free_buffer     = */ ggml_backend_cpu_buffer_free_buffer,
    /* .get_base        = */ ggml_backend_cpu_buffer_get_base,
    /* .init_tensor     = */ NULL, // 不需要初始化
    /* .set_tensor      = */ ggml_backend_cpu_buffer_set_tensor,  // 设置张量到 CPU 缓冲区
    /* .get_tensor      = */ ggml_backend_cpu_buffer_get_tensor,  // 从 CPU 缓冲区获取张量
    /* .cpy_tensor      = */ ggml_backend_cpu_buffer_cpy_tensor,  // 复制张量到 CPU 缓冲区
    /* .clear           = */ ggml_backend_cpu_buffer_clear,  // 清空 CPU 缓冲区
    /* .reset           = */ NULL,  // 重置操作为空
// 定义一个结构体，包含 CPU 后端缓冲区接口的相关函数指针
static struct ggml_backend_buffer_i cpu_backend_buffer_i_from_ptr = {
    /* .get_name        = */ ggml_backend_cpu_buffer_name, // 获取缓冲区名称的函数指针
    /* .free_buffer     = */ NULL, // 不需要释放指针，因为指针不是缓冲区的所有者
    /* .get_base        = */ ggml_backend_cpu_buffer_get_base, // 获取缓冲区基地址的函数指针
    /* .init_tensor     = */ NULL, // 不需要初始化函数
    /* .set_tensor      = */ ggml_backend_cpu_buffer_set_tensor, // 设置张量数据的函数指针
    /* .get_tensor      = */ ggml_backend_cpu_buffer_get_tensor, // 获取张量数据的函数指针
    /* .cpy_tensor      = */ ggml_backend_cpu_buffer_cpy_tensor, // 复制张量数据的函数指针
    /* .clear           = */ ggml_backend_cpu_buffer_clear, // 清空缓冲区的函数指针
    /* .reset           = */ NULL, // 重置函数指针为空
};

// 定义张量对齐的大小为 64，足够支持 AVX 512
static const size_t TENSOR_ALIGNMENT = 64;

// 获取 CPU 缓冲区类型的名称
GGML_CALL static const char * ggml_backend_cpu_buffer_type_get_name(ggml_backend_buffer_type_t buft) {
    return "CPU";

    GGML_UNUSED(buft);
}

// 分配 CPU 缓冲区
GGML_CALL static ggml_backend_buffer_t ggml_backend_cpu_buffer_type_alloc_buffer(ggml_backend_buffer_type_t buft, size_t size) {
    size += TENSOR_ALIGNMENT; // 考虑到 malloc 可能返回未对齐的地址
    void * data = malloc(size); // 分配内存空间

    GGML_ASSERT(data != NULL && "failed to allocate buffer"); // 断言内存分配成功

    return ggml_backend_buffer_init(buft, cpu_backend_buffer_i, data, size); // 初始化缓冲区
}

// 获取 CPU 缓冲区的对齐大小
GGML_CALL static size_t ggml_backend_cpu_buffer_type_get_alignment(ggml_backend_buffer_type_t buft) {
    return TENSOR_ALIGNMENT;

    GGML_UNUSED(buft);
}

// 判断 CPU 缓冲区是否支持指定的后端
GGML_CALL static bool ggml_backend_cpu_buffer_type_supports_backend(ggml_backend_buffer_type_t buft, ggml_backend_t backend) {
    return ggml_backend_is_cpu(backend); // 判断是否为 CPU 后端

    GGML_UNUSED(buft);
}

// 判断 CPU 缓冲区是否为主机缓冲区
GGML_CALL static bool ggml_backend_cpu_buffer_type_is_host(ggml_backend_buffer_type_t buft) {
    return true;

    GGML_UNUSED(buft);
}

// 获取 CPU 缓冲区类型
GGML_CALL ggml_backend_buffer_type_t ggml_backend_cpu_buffer_type(void) {
    // 定义静态结构体 ggml_backend_cpu_buffer_type，表示 CPU 缓冲区类型
    static struct ggml_backend_buffer_type ggml_backend_cpu_buffer_type = {
        // 接口部分
        /* .iface = */ {
            // 获取类型名称的函数指针
            /* .get_name         = */ ggml_backend_cpu_buffer_type_get_name,
            // 分配缓冲区的函数指针
            /* .alloc_buffer     = */ ggml_backend_cpu_buffer_type_alloc_buffer,
            // 获取对齐方式的函数指针
            /* .get_alignment    = */ ggml_backend_cpu_buffer_type_get_alignment,
            // 获取最大大小的函数指针，默认为 SIZE_MAX
            /* .get_max_size     = */ NULL,
            // 获取分配大小的函数指针，默认为 ggml_nbytes
            /* .get_alloc_size   = */ NULL,
            // 判断是否支持后端的函数指针
            /* .supports_backend = */ ggml_backend_cpu_buffer_type_supports_backend,
            // 判断是否为主机的函数指针
            /* .is_host          = */ ggml_backend_cpu_buffer_type_is_host,
        },
        // 上下文部分
        /* .context = */ NULL,
    };

    // 返回指向 ggml_backend_cpu_buffer_type 的指针
    return &ggml_backend_cpu_buffer_type;
// 如果定义了 GGML_USE_CPU_HBM，则以下代码块将被编译

// 包含 HBM 内存分配的头文件
#include <hbwmalloc.h>

// 获取 CPU_HBM 缓冲区类型的名称
GGML_CALL static const char * ggml_backend_cpu_hbm_buffer_type_get_name(ggml_backend_buffer_type_t buft) {
    return "CPU_HBM";

    GGML_UNUSED(buft);
}

// 获取 CPU_HBM 缓冲区的名称
GGML_CALL static const char * ggml_backend_cpu_hbm_buffer_get_name(ggml_backend_buffer_t buf) {
    return "CPU_HBM";

    GGML_UNUSED(buf);
}

// 释放 CPU_HBM 缓冲区的内存
GGML_CALL static void ggml_backend_cpu_hbm_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    hbw_free(buffer->context);
}

// 分配 CPU_HBM 缓冲区
GGML_CALL static ggml_backend_buffer_t ggml_backend_cpu_hbm_buffer_type_alloc_buffer(ggml_backend_buffer_type_t buft, size_t size) {
    //void * ptr = hbw_malloc(size);
    void * ptr;
    // 使用 hbw_posix_memalign 分配内存
    int result = hbw_posix_memalign(&ptr, ggml_backend_cpu_buffer_type_get_alignment(buft), size);
    // 如果分配失败，则打印错误信息并返回 NULL
    if (result != 0) {
        fprintf(stderr, "failed to allocate HBM buffer of size %zu\n", size);
        return NULL;
    }

    // 创建 CPU_HBM 缓冲区对象
    ggml_backend_buffer_t buffer = ggml_backend_cpu_buffer_from_ptr(ptr, size);
    buffer->buft = buft;
    buffer->iface.get_name = ggml_backend_cpu_hbm_buffer_get_name;
    buffer->iface.free_buffer = ggml_backend_cpu_hbm_buffer_free_buffer;

    return buffer;
}

// 获取 CPU_HBM 缓冲区类型
ggml_backend_buffer_type_t ggml_backend_cpu_hbm_buffer_type(void) {
    // 定义 CPU_HBM 缓冲区类型结构体
    static struct ggml_backend_buffer_type ggml_backend_cpu_buffer_type_hbm = {
        /* .iface    = */ {
            /* .get_name         = */ ggml_backend_cpu_hbm_buffer_type_get_name,
            /* .alloc_buffer     = */ ggml_backend_cpu_hbm_buffer_type_alloc_buffer,
            /* .get_alignment    = */ ggml_backend_cpu_buffer_type_get_alignment,
            /* .get_max_size     = */ NULL, // 默认为 SIZE_MAX
            /* .get_alloc_size   = */ NULL, // 默认为 ggml_nbytes
            /* .supports_backend = */ ggml_backend_cpu_buffer_type_supports_backend,
            /* .is_host          = */ ggml_backend_cpu_buffer_type_is_host,
        },
        /* .context  = */ NULL,
    };
    # 返回 ggml_backend_cpu_buffer_type_hbm 的值
    return &ggml_backend_cpu_buffer_type_hbm;
#endif

// 定义 CPU 后端上下文结构体
struct ggml_backend_cpu_context {
    int n_threads; // 线程数
    void * work_data; // 工作数据
    size_t work_size; // 工作数据大小
};

// 获取 CPU 后端名称
GGML_CALL static const char * ggml_backend_cpu_name(ggml_backend_t backend) {
    return "CPU"; // 返回 CPU 字符串

    GGML_UNUSED(backend);
}

// 释放 CPU 后端资源
GGML_CALL static void ggml_backend_cpu_free(ggml_backend_t backend) {
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;
    free(cpu_ctx->work_data); // 释放工作数据内存
    free(cpu_ctx); // 释放 CPU 上下文内存
    free(backend); // 释放后端内存
}

// 获取 CPU 后端默认缓冲区类型
GGML_CALL static ggml_backend_buffer_type_t ggml_backend_cpu_get_default_buffer_type(ggml_backend_t backend) {
    return ggml_backend_cpu_buffer_type(); // 返回 CPU 缓冲区类型

    GGML_UNUSED(backend);
}

// 定义 CPU 后端计划结构体
struct ggml_backend_plan_cpu {
    struct ggml_cplan cplan; // 计划
    struct ggml_cgraph cgraph; // 图
};

// 创建 CPU 后端图计划
GGML_CALL static ggml_backend_graph_plan_t ggml_backend_cpu_graph_plan_create(ggml_backend_t backend, const struct ggml_cgraph * cgraph) {
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    struct ggml_backend_plan_cpu * cpu_plan = malloc(sizeof(struct ggml_backend_plan_cpu)); // 分配内存

    cpu_plan->cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads); // 创建图计划
    cpu_plan->cgraph = *cgraph; // FIXME: deep copy

    if (cpu_plan->cplan.work_size > 0) {
        cpu_plan->cplan.work_data = malloc(cpu_plan->cplan.work_size); // 分配工作数据内存
    }

    return cpu_plan; // 返回 CPU 计划
}

// 释放 CPU 后端图计划
GGML_CALL static void ggml_backend_cpu_graph_plan_free(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    free(cpu_plan->cplan.work_data); // 释放工作数据内存
    free(cpu_plan); // 释放 CPU 计划内存

    GGML_UNUSED(backend);
}

// 计算 CPU 后端图计划
GGML_CALL static void ggml_backend_cpu_graph_plan_compute(ggml_backend_t backend, ggml_backend_graph_plan_t plan) {
    struct ggml_backend_plan_cpu * cpu_plan = (struct ggml_backend_plan_cpu *)plan;

    ggml_graph_compute(&cpu_plan->cgraph, &cpu_plan->cplan); // 计算图计划

    GGML_UNUSED(backend);
}
GGML_CALL static bool ggml_backend_cpu_graph_compute(ggml_backend_t backend, struct ggml_cgraph * cgraph) {
    // 获取 CPU 后端的上下文
    struct ggml_backend_cpu_context * cpu_ctx = (struct ggml_backend_cpu_context *)backend->context;

    // 根据计算图和线程数创建计划
    struct ggml_cplan cplan = ggml_graph_plan(cgraph, cpu_ctx->n_threads);

    // 如果工作数据大小小于计划的工作大小，则重新分配内存
    if (cpu_ctx->work_size < cplan.work_size) {
        // TODO: may be faster to free and use malloc to avoid the copy
        cpu_ctx->work_data = realloc(cpu_ctx->work_data, cplan.work_size);
        cpu_ctx->work_size = cplan.work_size;
    }

    // 将工作数据指针指向 CPU 上下文中的工作数据
    cplan.work_data = cpu_ctx->work_data;

    // 计算计划中的计算图
    ggml_graph_compute(cgraph, &cplan);
    return true;
}

GGML_CALL static bool ggml_backend_cpu_supports_op(ggml_backend_t backend, const struct ggml_tensor * op) {
    // 根据操作类型判断 CPU 后端是否支持该操作
    switch (op->op) {
        case GGML_OP_CPY:
            return op->type != GGML_TYPE_IQ2_XXS && op->type != GGML_TYPE_IQ2_XS; // missing type_traits.from_float
        case GGML_OP_MUL_MAT:
            return op->src[1]->type == GGML_TYPE_F32 || op->src[1]->type == ggml_internal_get_type_traits(op->src[0]->type).vec_dot_type;
        default:
            return true;
    }

    GGML_UNUSED(backend);
}

static struct ggml_backend_i cpu_backend_i = {
    /* .get_name                = */ ggml_backend_cpu_name,
    /* .free                    = */ ggml_backend_cpu_free,
    /* .get_default_buffer_type = */ ggml_backend_cpu_get_default_buffer_type,
    /* .set_tensor_async        = */ NULL,
    /* .get_tensor_async        = */ NULL,
    /* .cpy_tensor_async        = */ NULL,
    /* .synchronize             = */ NULL,
    /* .graph_plan_create       = */ ggml_backend_cpu_graph_plan_create,
    /* .graph_plan_free         = */ ggml_backend_cpu_graph_plan_free,
    /* .graph_plan_compute      = */ ggml_backend_cpu_graph_plan_compute,
    /* .graph_compute           = */ ggml_backend_cpu_graph_compute,
    /* .supports_op             = */ ggml_backend_cpu_supports_op,
};

ggml_backend_t ggml_backend_cpu_init(void) {
    // 分配内存以存储 ggml_backend_cpu_context 结构体的实例，并将指针赋给 ctx
    struct ggml_backend_cpu_context * ctx = malloc(sizeof(struct ggml_backend_cpu_context));

    // 设置 ctx 结构体实例中的 n_threads 字段为 GGML_DEFAULT_N_THREADS
    ctx->n_threads = GGML_DEFAULT_N_THREADS;
    // 设置 ctx 结构体实例中的 work_data 字段为 NULL
    ctx->work_data = NULL;
    // 设置 ctx 结构体实例中的 work_size 字段为 0
    ctx->work_size = 0;

    // 分配内存以存储 ggml_backend 结构体的实例，并将指针赋给 cpu_backend
    ggml_backend_t cpu_backend = malloc(sizeof(struct ggml_backend));

    // 使用结构体初始化器为 cpu_backend 结构体实例赋值
    *cpu_backend = (struct ggml_backend) {
        /* .interface = */ cpu_backend_i, // 设置 interface 字段为 cpu_backend_i
        /* .context   = */ ctx // 设置 context 字段为 ctx
    };
    // 返回指向 cpu_backend 结构体实例的指针
    return cpu_backend;
// 检查给定的后端是否为 CPU 后端
GGML_CALL bool ggml_backend_is_cpu(ggml_backend_t backend) {
    return backend && backend->iface.get_name == ggml_backend_cpu_name;
}

// 设置 CPU 后端的线程数
void ggml_backend_cpu_set_n_threads(ggml_backend_t backend_cpu, int n_threads) {
    GGML_ASSERT(ggml_backend_is_cpu(backend_cpu));

    // 获取 CPU 后端的上下文，并设置线程数
    struct ggml_backend_cpu_context * ctx = (struct ggml_backend_cpu_context *)backend_cpu->context;
    ctx->n_threads = n_threads;
}

// 从指针创建 CPU 后端缓冲区
GGML_CALL ggml_backend_buffer_t ggml_backend_cpu_buffer_from_ptr(void * ptr, size_t size) {
    return ggml_backend_buffer_init(ggml_backend_cpu_buffer_type(), cpu_backend_buffer_i_from_ptr, ptr, size);
}

// 注册 CPU 后端初始化函数
GGML_CALL static ggml_backend_t ggml_backend_reg_cpu_init(const char * params, void * user_data) {
    return ggml_backend_cpu_init();

    GGML_UNUSED(params);
    GGML_UNUSED(user_data);
}

// 多缓冲区结构
struct ggml_backend_multi_buffer_context {
    ggml_backend_buffer_t * buffers;
    size_t n_buffers;
};

typedef struct ggml_backend_multi_buffer_context * ggml_backend_multi_buffer_context_t;

// 获取多缓冲区的名称
GGML_CALL static const char * ggml_backend_multi_buffer_get_name(ggml_backend_buffer_t buffer) {
    ggml_backend_multi_buffer_context_t ctx = (ggml_backend_multi_buffer_context_t) buffer->context;

    return ctx->buffers[0]->iface.get_name(ctx->buffers[0]);
}

// 释放多缓冲区的缓冲区
GGML_CALL static void ggml_backend_multi_buffer_free_buffer(ggml_backend_buffer_t buffer) {
    ggml_backend_multi_buffer_context_t ctx = (ggml_backend_multi_buffer_context_t) buffer->context;
    for (size_t i = 0; i < ctx->n_buffers; i++) {
        ggml_backend_buffer_free(ctx->buffers[i]);
    }

    free(ctx->buffers);
    free(ctx);
}

// 清除多缓冲区的数据
GGML_CALL static void ggml_backend_multi_buffer_clear(ggml_backend_buffer_t buffer, uint8_t value) {
    ggml_backend_multi_buffer_context_t ctx = (ggml_backend_multi_buffer_context_t) buffer->context;
    for (size_t i = 0; i < ctx->n_buffers; i++) {
        ggml_backend_buffer_clear(ctx->buffers[i], value);
    }
}
// 定义一个静态结构体，包含多个指向函数的指针，用于实现多缓冲区接口
static struct ggml_backend_buffer_i ggml_backend_multi_buffer_context_interface(void) {
    // 初始化多缓冲区接口结构体
    static struct ggml_backend_buffer_i multi_backend_buffer_i = {
        /* .get_name        = */ ggml_backend_multi_buffer_get_name, // 获取缓冲区名称的函数指针
        /* .free_buffer     = */ ggml_backend_multi_buffer_free_buffer, // 释放缓冲区的函数指针
        /* .get_base        = */ NULL, // 获取缓冲区基地址的函数指针
        /* .init_tensor     = */ NULL, // 初始化张量的函数指针
        /* .set_tensor      = */ NULL, // 设置张量的函数指针
        /* .get_tensor      = */ NULL, // 获取张量的函数指针
        /* .cpy_tensor      = */ NULL, // 复制张量的函数指针
        /* .clear           = */ ggml_backend_multi_buffer_clear, // 清除缓冲区的函数指针
        /* .reset           = */ NULL, // 重置缓冲区的函数指针
    };

    return multi_backend_buffer_i; // 返回多缓冲区接口结构体
}

// 分配多个缓冲区
GGML_CALL ggml_backend_buffer_t ggml_backend_multi_buffer_alloc_buffer(ggml_backend_buffer_t * buffers, size_t n_buffers) {
    // 分配多缓冲区上下文
    ggml_backend_multi_buffer_context_t ctx = (ggml_backend_multi_buffer_context_t) malloc(sizeof(struct ggml_backend_multi_buffer_context));
    ctx->n_buffers = n_buffers;
    ctx->buffers = (ggml_backend_buffer_t *) malloc(n_buffers * sizeof(ggml_backend_buffer_t));

    size_t total_size = 0;
    for (size_t i = 0; i < n_buffers; i++) {
        ctx->buffers[i] = buffers[i];
        total_size += ggml_backend_buffer_get_size(buffers[i]);
    }

    return ggml_backend_buffer_init(buffers[0]->buft, ggml_backend_multi_buffer_context_interface(), ctx, total_size); // 初始化多缓冲区
}

// 检查缓冲区是否为多缓冲区
GGML_CALL bool ggml_backend_buffer_is_multi_buffer(ggml_backend_buffer_t buffer) {
    return buffer->iface.get_name == ggml_backend_multi_buffer_get_name; // 检查缓冲区名称是否为多缓冲区名称
}

// 设置多缓冲区的使用方式
GGML_CALL void ggml_backend_multi_buffer_set_usage(ggml_backend_buffer_t buffer, enum ggml_backend_buffer_usage usage) {
    GGML_ASSERT(ggml_backend_buffer_is_multi_buffer(buffer)); // 断言缓冲区为多缓冲区
    ggml_backend_multi_buffer_context_t ctx = (ggml_backend_multi_buffer_context_t) buffer->context;
    for (size_t i = 0; i < ctx->n_buffers; i++) {
        ggml_backend_buffer_set_usage(ctx->buffers[i], usage); // 设置多缓冲区中每个缓冲区的使用方式
    }
}

// 调度器

// 定义最大后端数和最大分割数
#define GGML_MAX_BACKENDS 16
#define GGML_MAX_SPLITS 256
// 定义最大分割输入数量为16
#define GGML_MAX_SPLIT_INPUTS 16

// 定义结构体 ggml_backend_sched_split
struct ggml_backend_sched_split {
    ggml_tallocr_t tallocr; // 内存分配器
    int i_start; // 起始索引
    int i_end; // 结束索引
    struct ggml_tensor * inputs[GGML_MAX_SPLIT_INPUTS]; // 输入张量数组
    int n_inputs; // 输入张量数量
    // 表示此分割的图形视图
    struct ggml_cgraph graph;
};

// 定义结构体 ggml_backend_sched
struct ggml_backend_sched {
    bool is_reset; // 如果调度器自上次图形分割以来已重置，则为 true

    int n_backends; // 后端数量
    ggml_backend_t backends[GGML_MAX_BACKENDS]; // 后端数组
    ggml_backend_buffer_type_t bufts[GGML_MAX_BACKENDS]; // 后端缓冲区类型数组
    ggml_tallocr_t  tallocs[GGML_MAX_BACKENDS]; // 内存分配器数组

    ggml_gallocr_t galloc; // 全局内存分配器

    // 图中节点的哈希键
    struct ggml_hash_set    hash_set;
    // 哈希值（数组大小为[hash_set.size]）
    ggml_tallocr_t *        node_talloc; // 每个节点分配的内存分配器（间接地是后端）
    struct ggml_tensor * (* node_copies)[GGML_MAX_BACKENDS]; // 每个目标后端的每个节点的副本

    // 具有修改输入的图的副本
    struct ggml_cgraph * graph;

    struct ggml_backend_sched_split splits[GGML_MAX_SPLITS]; // 分割数组
    int n_splits; // 分割数量

    struct ggml_context * ctx; // 上下文

    // 将 context_buffer 对齐到 GGML_MEM_ALIGN
    #ifdef _MSC_VER
    __declspec(align(GGML_MEM_ALIGN))
    #else
    __attribute__((aligned(GGML_MEM_ALIGN)))
    #endif
    char context_buffer[GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS*sizeof(struct ggml_tensor) + sizeof(struct ggml_cgraph)]; // 上下文缓冲区

    ggml_backend_sched_eval_callback callback_eval; // 回调函数评估
    void * callback_eval_user_data; // 回调函数评估用户数据
};

// 定义宏 hash_id(node)，查找或插入节点的哈希值
#define hash_id(node) ggml_hash_find_or_insert(sched->hash_set, node)
// 定义宏 node_allocr(node)，返回节点的内存分配器
#define node_allocr(node) sched->node_talloc[hash_id(node)]

// 判断操作是否为视图操作，是则返回 true
static bool ggml_is_view_op(enum ggml_op op) {
    return op == GGML_OP_VIEW || op == GGML_OP_RESHAPE || op == GGML_OP_PERMUTE || op == GGML_OP_TRANSPOSE;
}

// 返回后端的优先级，数值越小表示优先级越高
static int sched_backend_prio(ggml_backend_sched_t sched, ggml_backend_t backend) {
    # 遍历调度器中的后端列表
    for (int i = 0; i < sched->n_backends; i++) {
        # 如果当前后端与目标后端相同，则返回当前后端的索引
        if (sched->backends[i] == backend) {
            return i;
        }
    }
    # 如果未找到目标后端，则返回一个极大值
    return INT_MAX;
// 为给定的调度器和分配器分配优先级，返回分配器在调度器中的索引
static int sched_allocr_prio(ggml_backend_sched_t sched, ggml_tallocr_t allocr) {
    // 遍历调度器中的后端，查找是否存在与给定分配器相同的分配器
    for (int i = 0; i < sched->n_backends; i++) {
        if (sched->tallocs[i] == allocr) {
            return i;
        }
    }
    // 如果未找到相同的分配器，则返回最大整数值
    return INT_MAX;
}

// 根据缓冲区获取分配器
static ggml_tallocr_t sched_allocr_from_buffer(ggml_backend_sched_t sched, ggml_backend_buffer_t buffer) {
    // 如果缓冲区为空，则返回空
    if (buffer == NULL) {
        return NULL;
    }

    // 检查是否已经在分配器缓冲区中分配了该缓冲区（来自用户手动分配）
    for (int i = 0; i < sched->n_backends; i++) {
        if (ggml_tallocr_get_buffer(sched->tallocs[i]) == buffer) {
            return sched->tallocs[i];
        }
    }

    // 查找支持缓冲区类型的优先级最高的后端
    for (int i = 0; i < sched->n_backends; i++) {
        if (ggml_backend_buft_supports_backend(buffer->buft, sched->backends[i])) {
            return sched->tallocs[i];
        }
    }
    // 如果未找到支持的后端，则断言失败
    GGML_ASSERT(false && "tensor buffer type not supported by any backend");
}

// 根据分配器获取后端
static ggml_backend_t get_allocr_backend(ggml_backend_sched_t sched, ggml_tallocr_t allocr) {
    // 如果分配器为空，则返回空
    if (allocr == NULL) {
        return NULL;
    }
    // 遍历调度器中的后端，查找与给定分配器相对应的后端
    for (int i = 0; i < sched->n_backends; i++) {
        if (sched->tallocs[i] == allocr) {
            return sched->backends[i];
        }
    }
    // 如果未找到相应的后端，则断言失败
    GGML_UNREACHABLE();
}

#if 0
// 仅用于调试，定义一个数组用于存储导致节点的原因
static char causes[GGML_DEFAULT_GRAPH_SIZE*16 + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS][128]; // debug only
// 定义宏，用于设置导致节点的原因
#define SET_CAUSE(node, ...) sprintf(causes[hash_id(node)], __VA_ARGS__)
// 定义宏，用于获取导致节点的原因
#define GET_CAUSE(node) causes[hash_id(node)]
#else
// 如果不是调试模式，则定义 SET_CAUSE 和 GET_CAUSE 宏为空
#define SET_CAUSE(node, ...)
#define GET_CAUSE(node) ""
#endif

// 根据当前位置返回应该用于节点的后端分配器
static ggml_tallocr_t sched_allocr_from_cur(ggml_backend_sched_t sched, struct ggml_tensor * node) {
    // 为节点的缓冲区获取分配器
    ggml_tallocr_t cur_allocr = sched_allocr_from_buffer(sched, node->buffer);
    // 如果当前分配器不为空，则返回当前分配器
    if (cur_allocr != NULL) {
        SET_CAUSE(node, "1.dst");
        return cur_allocr;
    }
    // 如果节点的视图源不为空
    // 则尝试从视图源的缓冲区中分配资源
    if (node->view_src != NULL) {
        cur_allocr = sched_allocr_from_buffer(sched, node->view_src->buffer);
        // 如果成功分配资源，则返回当前分配器
        if (cur_allocr != NULL) {
            SET_CAUSE(node, "1.vsrc");
            return cur_allocr;
        }
    }
    // 将使用权重的节点分配到权重的后端
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        const struct ggml_tensor * src = node->src[i];
        // 如果源为空，则跳出循环
        if (src == NULL) {
            break;
        }
        // 如果源的缓冲区不为空且使用权重，则从缓冲区中分配资源
        if (src->buffer != NULL && src->buffer->usage == GGML_BACKEND_BUFFER_USAGE_WEIGHTS) {
            ggml_tallocr_t src_allocr = sched_allocr_from_buffer(sched, src->buffer);
            // 操作与权重始终在相同的后端上运行
            SET_CAUSE(node, "1.wgt%d", i);
            return src_allocr;
        }
    }

    // 如果以上条件都不满足，则返回空
    return NULL;
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
        // 检查是否需要在当前节点处进行拆分
        if (cur_split < sched->n_splits && i == sched->splits[cur_split].i_start) {
            // 获取拆分节点的分配器后端
            ggml_backend_t split_backend = get_allocr_backend(sched, sched->splits[cur_split].tallocr);
            // 输出拆分信息
            fprintf(stderr, "\n## SPLIT #%d: %s # %d inputs: ", cur_split, ggml_backend_name(split_backend),
                sched->splits[cur_split].n_inputs);
            // 输出拆分节点的输入信息
            for (int j = 0; j < sched->splits[cur_split].n_inputs; j++) {
                fprintf(stderr, "[%s (%5.5s)] ", sched->splits[cur_split].inputs[j]->name,
                    fmt_size(ggml_nbytes(sched->splits[cur_split].inputs[j])));
            }
            fprintf(stderr, "\n");
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
        // 获取当前节点的分配器后端
        ggml_backend_t node_backend = node_allocr ? get_allocr_backend(sched, node_allocr) : NULL; // FIXME:
        // 输出当前节点信息
        fprintf(stderr, "node #%3d (%10.10s): %20.20s (%5.5s) [%5.5s %8.8s]:", i, ggml_op_name(node->op), node->name,
            fmt_size(ggml_nbytes(node)), node_allocr ? ggml_backend_name(node_backend) : "NULL", GET_CAUSE(node));
        // 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            struct ggml_tensor * src = node->src[j];
            // 如果源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取源节点的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 获取源节点的分配器后端
            ggml_backend_t src_backend = src_allocr ? get_allocr_backend(sched, src_allocr) : NULL;
            // 输出源节点信息
            fprintf(stderr, " %20.20s (%5.5s) [%5.5s %8.8s]", src->name,
                fmt_size(ggml_nbytes(src)), src_backend ? ggml_backend_name(src_backend) : "NULL", GET_CAUSE(src));
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
    // 返回复制后的张量
    return dup;
}

// 为操作分配后端并将图拆分为可以在相同后端上计算的子图
static void sched_split_graph(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    // 重置拆分信息
    sched->n_splits = 0;
    sched->is_reset = false;

    // 初始化参数
    struct ggml_init_params params = {
        /* .mem_size =   */ sizeof(sched->context_buffer),
        /* .mem_buffer = */ sched->context_buffer,
        /* .no_alloc =   */ true
    };

    // 释放之前的上下文
    ggml_free(sched->ctx);

    // 初始化上下文
    sched->ctx = ggml_init(params);
    if (sched->ctx == NULL) {
        fprintf(stderr, "%s: failed to initialize context\n", __func__);
        GGML_ASSERT(false);
    }

    // pass 1: 为具有预分配输入的操作分配后端
    for (int i = 0; i < graph->n_leafs; i++) {
        struct ggml_tensor * leaf = graph->leafs[i];
        if (node_allocr(leaf) != NULL) {
            // 不覆盖用户分配的后端
            continue;
        }
        // 为 leaf 分配后端
        node_allocr(leaf) = sched_allocr_from_cur(sched, leaf);
    }
}
    // 遍历图中的每个节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 如果节点已经分配资源，则跳过
        if (node_allocr(node) != NULL) {
            // 不覆盖用户分配的资源
            continue;
        }
        // 为节点分配资源
        node_allocr(node) = sched_allocr_from_cur(sched, node);
        // 处理节点的输入
        // 遍历节点的输入
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前输入节点
            struct ggml_tensor * src = node->src[j];
            // 如果输入节点为空，跳出循环
            if (src == NULL) {
                break;
            }
            // 如果输入节点未分配资源，则为其分配资源
            if (node_allocr(src) == NULL) {
                node_allocr(src) = sched_allocr_from_cur(sched, src);
            }
        }
    }
#ifdef DEBUG_PASS1
    // 如果定义了 DEBUG_PASS1 宏，则输出调试信息
    fprintf(stderr, "PASS 1 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);
#endif

    // pass 2: expand current backend assignments
    // assign the same backend to adjacent nodes
    // expand gpu backends (i.e. non last prio) up and down, ignoring cpu (the lowest priority backend)
    // thus, cpu will never be used unless weights are on cpu, or there are no gpu ops between cpu ops

    // pass 2.1 expand gpu up
    {
        // 初始化当前分配器为 NULL
        ggml_tallocr_t cur_allocr = NULL;
        // 从最后一个节点向前遍历图中的节点
        for (int i = graph->n_nodes - 1; i >= 0; i--) {
            // 获取当前节点
            struct ggml_tensor * node = graph->nodes[i];
            // 如果当前节点是视图操作，则跳过
            if (ggml_is_view_op(node->op)) {
                continue;
            }
            // 获取当前节点的分配器
            ggml_tallocr_t node_allocr = node_allocr(node);
            // 如果当前节点有分配器
            if (node_allocr != NULL) {
                // 如果当前节点的分配器是最后一个优先级的后端（通常是 CPU）
                if (sched_allocr_prio(sched, node_allocr) == sched->n_backends - 1) {
                    // 跳过 CPU（最低优先级的后端）
                    cur_allocr = NULL;
                } else {
                    cur_allocr = node_allocr;
                }
            } else {
                // 将当前节点的分配器设置为当前分配器
                node_allocr(node) = cur_allocr;
                // 设置节点的原因为 "2.1"
                SET_CAUSE(node, "2.1");
            }
        }
    }

    // pass 2.2 expand gpu down
    {
        // 初始化当前分配器为 NULL
        ggml_tallocr_t cur_allocr = NULL;
        // 从第一个节点向后遍历图中的节点
        for (int i = 0; i < graph->n_nodes; i++) {
            // 获取当前节点
            struct ggml_tensor * node = graph->nodes[i];
            // 如果当前节点是视图操作，则跳过
            if (ggml_is_view_op(node->op)) {
                continue;
            }
            // 获取当前节点的分配器
            ggml_tallocr_t node_allocr = node_allocr(node);
            // 如果当前节点有分配器
            if (node_allocr != NULL) {
                // 如果当前节点的分配器是最后一个优先级的后端（通常是 CPU）
                if (sched_allocr_prio(sched, node_allocr) == sched->n_backends - 1) {
                    // 跳过 CPU（最低优先级的后端）
                    cur_allocr = NULL;
                } else {
                    cur_allocr = node_allocr;
                }
            } else {
                // 将当前节点的分配器设置为当前分配器
                node_allocr(node) = cur_allocr;
                // 设置节点的原因为 "2.2"
                SET_CAUSE(node, "2.2");
            }
        }
    }
    // pass 2.3 expand rest up
    {
        // 初始化当前分配器为 NULL
        ggml_tallocr_t cur_allocr = NULL;
        // 从最后一个节点开始向前遍历图中的节点
        for (int i = graph->n_nodes - 1; i >= 0; i--) {
            // 获取当前节点
            struct ggml_tensor * node = graph->nodes[i];
            // 如果当前节点是视图操作，则跳过
            if (ggml_is_view_op(node->op)) {
                continue;
            }
            // 获取当前节点的分配器
            ggml_tallocr_t node_allocr = node_allocr(node);
            // 如果当前节点有分配器，则更新当前分配器
            if (node_allocr != NULL) {
                cur_allocr = node_allocr;
            } else {
                // 如果当前节点没有分配器，则将当前分配器赋给当前节点的分配器，并设置原因为 "2.3"
                node_allocr(node) = cur_allocr;
                SET_CAUSE(node, "2.3");
            }
        }
    }

    // pass 2.4 expand rest down
    {
        // 初始化当前分配器为 NULL
        ggml_tallocr_t cur_allocr = NULL;
        // 从第一个节点开始向后遍历图中的节点
        for (int i = 0; i < graph->n_nodes; i++) {
            // 获取当前节点
            struct ggml_tensor * node = graph->nodes[i];
            // 如果当前节点是视图操作，则跳过
            if (ggml_is_view_op(node->op)) {
                continue;
            }
            // 获取当前节点的分配器
            ggml_tallocr_t node_allocr = node_allocr(node);
            // 如果当前节点有分配器，则更新当前分配器
            if (node_allocr != NULL) {
                cur_allocr = node_allocr;
            } else {
                // 如果当前节点没有分配器，则将当前分配器赋给当前节点的分配器，并设置原因为 "2.4"
                node_allocr(node) = cur_allocr;
                SET_CAUSE(node, "2.4");
            }
        }
    }
#ifdef DEBUG_PASS2
    // 如果定义了 DEBUG_PASS2 宏，则输出调试信息
    fprintf(stderr, "PASS 2 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);
#endif

    // pass 3: assign backends to remaining src from dst and view_src
    // 第三步：为剩余的源节点从目标节点和视图源节点分配后端
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取当前节点的分配器
        ggml_tallocr_t cur_allocr = node_allocr(node);
        // 如果当前节点的视图源节点不为空且当前分配器为空
        if (node->view_src != NULL && cur_allocr == NULL) {
            // 将当前节点的分配器设置为视图源节点的分配器
            cur_allocr = node_allocr(node) = node_allocr(node->view_src);
            // 设置当前节点的原因为 "3.vsrc"
            SET_CAUSE(node, "3.vsrc");
        }
        // 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前源节点
            struct ggml_tensor * src = node->src[j];
            // 如果当前源节点为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取当前源节点的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果当前源节点的分配器为空
            if (src_allocr == NULL) {
                // 如果当前源节点的视图源节点不为空
                if (src->view_src != NULL) {
                    // 视图节点总是与源节点相同的后端
                    // 将当前源节点的分配器设置为视图源节点的分配器
                    node_allocr(src) = node_allocr(src->view_src);
                    // 设置当前源节点的原因为 "3.vsrc"
                    SET_CAUSE(src, "3.vsrc");
                } else {
                    // 否则将当前源节点的分配器设置为当前节点的分配器
                    node_allocr(src) = cur_allocr;
                    // 设置当前源节点的原因为 "3.cur"
                    SET_CAUSE(src, "3.cur");
                }
            }
        }
    }
#ifdef DEBUG_PASS3
    // 如果定义了 DEBUG_PASS3 宏，则输出调试信息
    fprintf(stderr, "PASS 3 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);
#endif

    // pass 4: split graph, find tensors that need to be copied
    // 第四步：拆分图形，找到需要复制的张量
#if 0
                    // 检查输入是否已经在分割中
                    bool found = false;
                    for (int k = 0; k < sched->splits[cur_split].n_inputs; k++) {
                        if (sched->splits[cur_split].inputs[k] == src) {
                            found = true;
                            break;
                        }
                    }

                    if (!found) {
                        int n_inputs = sched->splits[cur_split].n_inputs++;
                        //printf("split %d input %d: %s (%s)\n", cur_split, n_inputs, src->name, ggml_backend_name(get_allocr_backend(sched, src_allocr)));
                        GGML_ASSERT(n_inputs < GGML_MAX_SPLIT_INPUTS);
                        sched->splits[cur_split].inputs[n_inputs] = src;
                    }
#endif
                }
            }
        }
        // 设置当前分割的结束节点索引为图中节点的数量
        sched->splits[cur_split].i_end = graph->n_nodes;
        // 更新分割数量
        sched->n_splits = cur_split + 1;
    }
#ifdef DEBUG_PASS4
    // 调试输出：PASS 4 ASSIGNMENTS
    fprintf(stderr, "PASS 4 ASSIGNMENTS\n"); sched_print_assignments(sched, graph);
#endif

#ifndef NDEBUG
    // 断言检查：所有源节点应该与节点具有相同的后端
    // 遍历图中的每个节点
    for (int i = 0; i < graph->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = graph->nodes[i];
        // 获取当前节点的分配器
        ggml_tallocr_t node_allocr = node_allocr(node);
        // 如果当前节点没有分配器，则输出错误信息
        if (node_allocr == NULL) {
            fprintf(stderr, "!!!!!!! %s has no backend\n", node->name);
        }
        // 如果当前节点有视图源并且其分配器与视图源的分配器不同，则输出错误信息
        if (node->view_src != NULL && node_allocr != node_allocr(node->view_src)) {
            fprintf(stderr, "!!!!!!! %s has backend %s, view_src %s has backend %s\n",
                node->name, node_allocr ? ggml_backend_name(get_allocr_backend(sched, node_allocr)) : "NULL",
                node->view_src->name, node_allocr(node->view_src) ? ggml_backend_name(get_allocr_backend(sched, node_allocr(node->view_src))) : "NULL");
        }
        // 遍历当前节点的每个输入源
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前输入源
            struct ggml_tensor * src = node->src[j];
            // 如果当前输入源为空，则跳出循环
            if (src == NULL) {
                break;
            }
            // 获取当前输入源的分配器
            ggml_tallocr_t src_allocr = node_allocr(src);
            // 如果当前输入源的分配器与当前节点的分配器不同，则输出错误信息
            if (src_allocr != node_allocr /* && src_backend != NULL */) { // ignore nulls for now
                fprintf(stderr, "!!!! %s has backend %s, src %d (%s) has backend %s\n",
                    node->name, node_allocr ? ggml_backend_name(get_allocr_backend(sched, node_allocr)) : "NULL",
                    j, src->name, src_allocr ? ggml_backend_name(get_allocr_backend(sched, src_allocr)) : "NULL");
            }
            // 如果当前输入源有视图源并且其分配器与视图源的分配器不同，则输出错误信息
            if (src->view_src != NULL && src_allocr != node_allocr(src->view_src)) {
                fprintf(stderr, "!!!!!!! [src] %s has backend %s, view_src %s has backend %s\n",
                    src->name, src_allocr ? ggml_backend_name(get_allocr_backend(sched, src_allocr)) : "NULL",
                    src->view_src->name, node_allocr(src->view_src) ? ggml_backend_name(get_allocr_backend(sched, node_allocr(src->view_src))) : "NULL");
            }
        }
    }
    // 刷新标准错误流
    fflush(stderr);
#endif

    // 为每个分割创建图的副本
    // FIXME: 避免这种复制，以某种方式将分割输入传递给 ggml_gallocr_alloc_graph_n
    // 创建一个新的图副本，节点数量为原图节点数量加上分割数量乘以最大分割输入数
    struct ggml_cgraph * graph_copy = ggml_new_graph_custom(sched->ctx, graph->n_nodes + sched->n_splits*GGML_MAX_SPLIT_INPUTS, false);
    for (int i = 0; i < sched->n_splits; i++) {
        struct ggml_backend_sched_split * split = &sched->splits[i];
        // 为每个分割创建一个视图
        split->graph = ggml_graph_view(graph, split->i_start, split->i_end);

        // 将输入添加到图副本中，以便它们在分割开始时由 ggml-alloc 分配
        for (int j = 0; j < split->n_inputs; j++) {
            struct ggml_tensor * input = split->inputs[j];
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(input)][sched_allocr_prio(sched, split->tallocr)];
            // 添加一个依赖关系到输入源，以确保在复制完成之前不会释放输入
            GGML_ASSERT(input_cpy->src[0] == NULL || input_cpy->src[0] == input);
            input_cpy->src[0] = input;
            graph_copy->nodes[graph_copy->n_nodes++] = input_cpy;
        }

        for (int j = split->i_start; j < split->i_end; j++) {
            graph_copy->nodes[graph_copy->n_nodes++] = graph->nodes[j];
        }
    }
    sched->graph = graph_copy;
}

// 分配分割
static void sched_alloc_splits(ggml_backend_sched_t sched) {
    ggml_gallocr_alloc_graph_n(
        sched->galloc,
        sched->graph,
        sched->hash_set,
        sched->node_talloc);
}

// 计算分割
static void sched_compute_splits(ggml_backend_sched_t sched) {
    // 初始化用于记录复制时间和计算时间的数组
    uint64_t copy_us[GGML_MAX_BACKENDS] = {0};
    uint64_t compute_us[GGML_MAX_BACKENDS] = {0};

    // 获取分割数组的引用
    struct ggml_backend_sched_split * splits = sched->splits;
    // 遍历调度器中的分片数量
    for (int i = 0; i < sched->n_splits; i++) {
        // 获取当前分片的指针
        struct ggml_backend_sched_split * split = &splits[i];
        // 获取当前分片对应的分配器后端
        ggml_backend_t split_backend = get_allocr_backend(sched, split->tallocr);
        // 获取当前分片对应的后端优先级
        int split_backend_id = sched_backend_prio(sched, split_backend);

        // 将输入张量复制到分片后端
        // 记录复制开始时间
        uint64_t copy_start_us = ggml_time_us();
        // 遍历当前分片的输入张量
        for (int j = 0; j < split->n_inputs; j++) {
            // 获取当前输入张量的指针
            struct ggml_tensor * input = split->inputs[j];
            // 获取当前输入张量在当前后端的副本
            struct ggml_tensor * input_cpy = sched->node_copies[hash_id(input)][split_backend_id];

            // 断言输入张量和其副本的缓冲区不为空
            GGML_ASSERT(input->buffer != NULL);
            GGML_ASSERT(input_cpy->buffer != NULL);

            // TODO: 如果在先前的分片中已经复制过，且输入没有改变，则避免这次复制
            // 这对于避免多次复制常量如 KQ_mask 和 inp_pos 非常重要
            // 异步复制输入张量到副本
            ggml_backend_tensor_copy_async(split_backend, input, input_cpy);
        }
        //ggml_backend_synchronize(split_backend); // 用于测量复制时间的必要步骤
        // 记录复制结束时间
        int64_t copy_end_us = ggml_time_us();
        // 更新当前后端的复制时间
        copy_us[split_backend_id] += copy_end_us - copy_start_us;
#if 0
        // 定义一个用于存储分割文件名的字符数组
        char split_filename[GGML_MAX_NAME];
        // 格式化生成分割文件名，包含索引和后端名称
        snprintf(split_filename, GGML_MAX_NAME, "split_%i_%s.dot", i, ggml_backend_name(split_backend));
        // 将分割图以 DOT 格式输出到文件
        ggml_graph_dump_dot(split->graph, NULL, split_filename);
#endif


        // 计算开始时间戳
        uint64_t compute_start_us = ggml_time_us();
        // 如果没有指定回调函数，则直接计算分割图
        if (!sched->callback_eval) {
            ggml_backend_graph_compute(split_backend, &split->graph);
            //ggml_backend_synchronize(split_backend); // necessary to measure compute time
        } else {
            // 类似于 ggml_backend_compare_graph_backend 函数
            for (int j0 = 0; j0 < split->graph.n_nodes; j0++) {
                struct ggml_tensor * t = split->graph.nodes[j0];

                // 检查用户是否需要来自该节点的数据
                bool need = sched->callback_eval(t, true, sched->callback_eval_user_data);

                int j1 = j0;

                // 确定可以一起计算的节点范围 [j0, j1]
                while (!need && j1 < split->graph.n_nodes - 1) {
                    t = split->graph.nodes[++j1];
                    need = sched->callback_eval(t, true, sched->callback_eval_user_data);
                }

                // 创建一个视图，包含从 j0 到 j1 的节点
                struct ggml_cgraph gv = ggml_graph_view(&split->graph, j0, j1 + 1);

                // 计算视图中的节点
                ggml_backend_graph_compute(split_backend, &gv);

                // 如果需要数据且回调函数返回 false，则跳出循环
                if (need && !sched->callback_eval(t, false, sched->callback_eval_user_data)) {
                    break;
                }

                j0 = j1;
            }
        }
        // 计算结束时间戳
        uint64_t compute_end_us = ggml_time_us();
        // 计算计算时间并累加到对应后端的计算时间中
        compute_us[split_backend_id] += compute_end_us - compute_start_us;
    }

#if 0
    // 每个后端的计时信息
    fprintf(stderr, "sched_compute_splits times (%d splits):\n", sched->n_splits);
    # 遍历调度器中的后端
    for (int i = 0; i < sched->n_backends; i++) {
        # 如果某个后端的复制时间或计算时间大于0
        if (copy_us[i] > 0 || compute_us[i] > 0) {
            # 在标准错误流中打印该后端的名称、复制时间和计算时间
            fprintf(stderr, "\t%5.5s: %lu us copy, %lu us compute\n", ggml_backend_name(sched->backends[i]), copy_us[i], compute_us[i]);
        }
    }
#endif
}

// 重置调度器状态
static void sched_reset(ggml_backend_sched_t sched) {
    // 遍历所有后端，重置内存分配器状态
    for (int i = 0; i < sched->n_backends; i++) {
        ggml_tallocr_reset(sched->tallocs[i]);
    }
    // 重置状态以准备下一次运行
    size_t hash_size = sched->hash_set.size;
    // 将哈希表的键数组清零
    memset(sched->hash_set.keys, 0, sizeof(sched->hash_set.keys[0]) * hash_size);
    // 将节点内存分配器数组清零
    memset(sched->node_talloc,   0, sizeof(sched->node_talloc[0])   * hash_size);
    // 将节点拷贝数组清零
    memset(sched->node_copies,   0, sizeof(sched->node_copies[0])   * hash_size);

    sched->is_reset = true;
}

// 创建新的后端调度器
ggml_backend_sched_t ggml_backend_sched_new(ggml_backend_t * backends, ggml_backend_buffer_type_t * bufts, int n_backends, size_t graph_size) {
    GGML_ASSERT(n_backends > 0);
    GGML_ASSERT(n_backends <= GGML_MAX_BACKENDS);

    // 分配内存以创建调度器对象
    struct ggml_backend_sched * sched = calloc(sizeof(struct ggml_backend_sched), 1);

    // 初始化哈希表
    sched->hash_set    = ggml_hash_set_new(graph_size + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS);
    // 分配节点内存分配器数组内存
    sched->node_talloc = calloc(sizeof(sched->node_talloc[0]) * sched->hash_set.size, 1);
    // 分配节点拷贝数组内存
    sched->node_copies = calloc(sizeof(sched->node_copies[0]) * sched->hash_set.size, 1);

    sched->n_backends = n_backends;
    // 初始化后端和缓冲类型
    for (int i = 0; i < n_backends; i++) {
        sched->backends[i] = backends[i];
        sched->bufts[i] = bufts ? bufts[i] : ggml_backend_get_default_buffer_type(backends[i]);
    }

    sched->galloc = ggml_gallocr_new();

    // 为每个后端初始化测量内存分配器
    for (int i = 0; i < n_backends; i++) {
        sched->tallocs[i] = ggml_tallocr_new_measure_from_buft(sched->bufts[i]);
    }

    // 重置调度器状态
    sched_reset(sched);

    return sched;
}

// 释放后端调度器
void ggml_backend_sched_free(ggml_backend_sched_t sched) {
    if (sched == NULL) {
        return;
    }
    // 释放每个后端的内存分配器
    for (int i = 0; i < sched->n_backends; i++) {
        ggml_tallocr_free(sched->tallocs[i]);
    }
    // 释放全局内存分配器
    ggml_gallocr_free(sched->galloc);
    ggml_free(sched->ctx);
    // 释放哈希表键数组内存
    free(sched->hash_set.keys);
    // 释放节点内存分配器数组内存
    free(sched->node_talloc);
    // 释放节点拷贝数组内存
    free(sched->node_copies);
}
    # 释放调度器所占用的内存空间
    free(sched);
// 初始化测量调度器，将测量图分割并分配内存
void ggml_backend_sched_init_measure(ggml_backend_sched_t sched, struct ggml_cgraph * measure_graph) {
    GGML_ASSERT(ggml_tallocr_is_measure(sched->tallocs[0])); // 断言只能初始化一次

    sched_split_graph(sched, measure_graph); // 分割测量图
    sched_alloc_splits(sched); // 分配分割

    // 分配缓冲区并重置分配器
    for (int i = 0; i < sched->n_backends; i++) {
        size_t size = ggml_tallocr_max_size(sched->tallocs[i]);
        ggml_tallocr_free(sched->tallocs[i]);
        sched->tallocs[i] = ggml_tallocr_new_from_buft(sched->bufts[i], size);
    }

    sched_reset(sched); // 重置调度器
}

// 计算图的计算调度
void ggml_backend_sched_graph_compute(ggml_backend_sched_t sched, struct ggml_cgraph * graph) {
    GGML_ASSERT((int)sched->hash_set.size >= graph->n_nodes + GGML_MAX_SPLITS*GGML_MAX_SPLIT_INPUTS);

    if (!sched->is_reset) {
        sched_reset(sched); // 如果未重置，则重置调度器
    }

    sched_split_graph(sched, graph); // 分割图
    sched_alloc_splits(sched); // 分配分割
    sched_compute_splits(sched); // 计算分割
}

// 重置调度器
void ggml_backend_sched_reset(ggml_backend_sched_t sched) {
    sched_reset(sched); // 重置调度器
}

// 设置评估回调函数
void ggml_backend_sched_set_eval_callback(ggml_backend_sched_t sched, ggml_backend_sched_eval_callback callback, void * user_data) {
    sched->callback_eval = callback;
    sched->callback_eval_user_data = user_data;
}

// 获取分割数
int ggml_backend_sched_get_n_splits(ggml_backend_sched_t sched) {
    return sched->n_splits;
}

// 获取分配器
ggml_tallocr_t ggml_backend_sched_get_tallocr(ggml_backend_sched_t sched, ggml_backend_t backend) {
    int backend_index = sched_backend_prio(sched, backend);
    GGML_ASSERT(backend_index >= 0 && backend_index < sched->n_backends);
    return sched->tallocs[backend_index];
}

// 获取缓冲区
ggml_backend_buffer_t ggml_backend_sched_get_buffer(ggml_backend_sched_t sched, ggml_backend_t backend) {
    int backend_index = sched_backend_prio(sched, backend);
    GGML_ASSERT(backend_index >= 0 && backend_index < sched->n_backends);
    return ggml_tallocr_get_buffer(sched->tallocs[backend_index]);
}
// 设置节点的后端
void ggml_backend_sched_set_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node, ggml_backend_t backend) {
    // 获取后端的索引
    int backend_index = sched_backend_prio(sched, backend);
    // 确保后端索引在有效范围内
    GGML_ASSERT(backend_index >= 0 && backend_index < sched->n_backends);
    // 设置节点的分配器为调度器中对应后端的分配器
    node_allocr(node) = sched->tallocs[backend_index];
}

// 获取节点的后端
ggml_backend_t ggml_backend_sched_get_node_backend(ggml_backend_sched_t sched, struct ggml_tensor * node) {
    // 获取节点的分配器
    ggml_tallocr_t allocr = node_allocr(node);
    // 如果分配器为空，返回空指针
    if (allocr == NULL) {
        return NULL;
    }
    // 根据分配器获取后端
    return get_allocr_backend(sched, allocr);
}

// 初始化视图
void ggml_backend_view_init(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor) {
    // 确保张量的缓冲区为空
    GGML_ASSERT(tensor->buffer == NULL);
    // 确保视图源不为空
    GGML_ASSERT(tensor->view_src != NULL);
    // 确保视图源的缓冲区和数据不为空
    GGML_ASSERT(tensor->view_src->buffer != NULL);
    GGML_ASSERT(tensor->view_src->data != NULL);

    // 设置张量的缓冲区、数据、后端，并初始化缓冲区中的张量
    tensor->buffer = buffer;
    tensor->data = (char *)tensor->view_src->data + tensor->view_offs;
    tensor->backend = tensor->view_src->backend;
    ggml_backend_buffer_init_tensor(buffer, tensor);
}

// 分配张量
void ggml_backend_tensor_alloc(ggml_backend_buffer_t buffer, struct ggml_tensor * tensor, void * addr) {
    // 确保张量的缓冲区和数据为空
    GGML_ASSERT(tensor->buffer == NULL);
    GGML_ASSERT(tensor->data == NULL);
    // 确保视图源为空
    GGML_ASSERT(tensor->view_src == NULL);
    // 确保地址在缓冲区的有效范围内
    GGML_ASSERT(addr >= ggml_backend_buffer_get_base(buffer));
    GGML_ASSERT((char *)addr + ggml_backend_buffer_get_alloc_size(buffer, tensor) <=
                (char *)ggml_backend_buffer_get_base(buffer) + ggml_backend_buffer_get_size(buffer));

    // 设置张量的缓冲区、数据，并初始化缓冲区中的张量
    tensor->buffer = buffer;
    tensor->data = addr;
    ggml_backend_buffer_init_tensor(buffer, tensor);
}

// 复制图中的张量
static struct ggml_tensor * graph_dup_tensor(struct ggml_hash_set hash_set, struct ggml_tensor ** node_copies,
    // 复制一个图节点的信息到新的图节点中
    struct ggml_context * ctx_allocated, struct ggml_context * ctx_unallocated, struct ggml_tensor * src) {

    // 断言源节点不为空
    GGML_ASSERT(src != NULL);
    // 断言源节点的数据不为空，表示图节点已经被分配
    GGML_ASSERT(src->data && "graph must be allocated");

    // 将源节点插入哈希表中，返回节点的ID
    size_t id = ggml_hash_insert(hash_set, src);
    // 如果节点已经存在于哈希表中，则返回已存在的节点的副本
    if (id == GGML_HASHTABLE_ALREADY_EXISTS) {
        return node_copies[ggml_hash_find(hash_set, src)];
    }

    // 复制源节点的布局信息到新节点中
    struct ggml_tensor * dst = ggml_dup_tensor_layout(src->data && !src->view_src ? ctx_allocated : ctx_unallocated, src);
    // 如果源节点是视图节点，则复制视图源节点信息到新节点中
    if (src->view_src != NULL) {
        dst->view_src = graph_dup_tensor(hash_set, node_copies, ctx_allocated, ctx_unallocated, src->view_src);
        dst->view_offs = src->view_offs;
    }
    // 复制源节点的操作符和参数到新节点中
    dst->op = src->op;
    memcpy(dst->op_params, src->op_params, sizeof(dst->op_params));
    // 设置新节点的名称为源节点的名称
    ggml_set_name(dst, src->name);

    // 复制源节点的输入节点到新节点中
    for (int i = 0; i < GGML_MAX_SRC; i++) {
        struct ggml_tensor * s = src->src[i];
        if (s == NULL) {
            break;
        }
        dst->src[i] = graph_dup_tensor(hash_set, node_copies, ctx_allocated, ctx_unallocated, s);
    }

    // 将新节点存储在节点副本数组中，并返回新节点
    node_copies[id] = dst;
    return dst;
// 初始化张量节点，用于复制图中的张量节点
static void graph_init_tensor(struct ggml_hash_set hash_set, struct ggml_tensor ** node_copies, bool * node_init, struct ggml_tensor * src) {
    // 查找当前张量节点在哈希表中的索引
    size_t id = ggml_hash_find(hash_set, src);
    // 如果节点已经初始化过，则直接返回
    if (node_init[id]) {
        return;
    }
    // 将节点标记为已初始化
    node_init[id] = true;

    // 获取当前节点的复制节点
    struct ggml_tensor * dst = node_copies[id];
    // 如果复制节点存在视图源，则初始化视图源节点并进行视图初始化
    if (dst->view_src != NULL) {
        graph_init_tensor(hash_set, node_copies, node_init, src->view_src);
        ggml_backend_view_init(dst->view_src->buffer, dst);
    }
    // 否则，直接复制当前节点到复制节点
    else {
        ggml_backend_tensor_copy(src, dst);
    }

    // 初始化当前节点的源节点
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

// 复制后端图
struct ggml_backend_graph_copy ggml_backend_graph_copy(ggml_backend_t backend, struct ggml_cgraph * graph) {
    // 初始化哈希表
    struct ggml_hash_set hash_set = {
        /* .size = */ graph->visited_hash_table.size,
        /* .keys = */ calloc(sizeof(hash_set.keys[0]) * graph->visited_hash_table.size, 1)
    };
    // 分配节点复制数组和节点初始化标志数组
    struct ggml_tensor ** node_copies = calloc(sizeof(node_copies[0]) * hash_set.size, 1);
    bool * node_init = calloc(sizeof(node_init[0]) * hash_set.size, 1);

    // 初始化参数
    struct ggml_init_params params = {
        /* .mem_size   = */ ggml_tensor_overhead()*hash_set.size + ggml_graph_overhead_custom(graph->size, false),
        /* .mem_buffer = */ NULL,
        /* .no_alloc   = */ true
    };

    // 初始化分配和未分配的上下文
    struct ggml_context * ctx_allocated = ggml_init(params);
    struct ggml_context * ctx_unallocated = ggml_init(params);
}
    // 检查是否分配了上下文内存，如果没有则输出错误信息并释放相关资源，返回空的图形副本
    if (ctx_allocated == NULL || ctx_unallocated == NULL) {
        fprintf(stderr, "failed to allocate context for graph copy");
        // 释放内存
        free(hash_set.keys);
        free(node_copies);
        free(node_init);
        ggml_free(ctx_allocated);
        ggml_free(ctx_unallocated);
        return (struct ggml_backend_graph_copy) {
            /* .buffer           = */ NULL,
            /* .ctx_allocated    = */ NULL,
            /* .ctx_unallocated  = */ NULL,
            /* .graph            = */ NULL,
        };
    }

    // 复制节点
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        // 复制节点
        graph_dup_tensor(hash_set, node_copies, ctx_allocated, ctx_unallocated, node);
    }

    // 分配节点
    ggml_backend_buffer_t buffer = ggml_backend_alloc_ctx_tensors(ctx_allocated, backend);
    // 如果分配失败则输出错误信息并释放相关资源，返回空的图形副本
    if (buffer == NULL) {
        fprintf(stderr, "failed to allocate buffer for graph copy");
        // 释放内存
        free(hash_set.keys);
        free(node_copies);
        free(node_init);
        ggml_free(ctx_allocated);
        ggml_free(ctx_unallocated);
        return (struct ggml_backend_graph_copy) {
            /* .buffer           = */ NULL,
            /* .ctx_allocated    = */ NULL,
            /* .ctx_unallocated  = */ NULL,
            /* .graph            = */ NULL,
        };
    }

    // 复制数据并初始化视图
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        // 复制数据并初始化视图
        graph_init_tensor(hash_set, node_copies, node_init, node);
    }

    // 构建图形副本
    struct ggml_cgraph * graph_copy = ggml_new_graph_custom(ctx_allocated, graph->size, false);
    for (int i = 0; i < graph->n_nodes; i++) {
        struct ggml_tensor * node = graph->nodes[i];
        struct ggml_tensor * node_copy = node_copies[ggml_hash_find(hash_set, node)];
        // 将复制的节点添加到图形副本中
        graph_copy->nodes[i] = node_copy;
    }
    // 将原始图的节点数量赋值给图副本的节点数量
    graph_copy->n_nodes = graph->n_nodes;

    // 释放哈希集合的键
    free(hash_set.keys);
    // 释放节点副本数组
    free(node_copies);
    // 释放节点初始化数组
    free(node_init);

    // 返回一个结构体，包含缓冲区、已分配的上下文、未分配的上下文和图副本
    return (struct ggml_backend_graph_copy) {
        /* .buffer           = */ buffer,
        /* .ctx_allocated    = */ ctx_allocated,
        /* .ctx_unallocated  = */ ctx_unallocated,
        /* .graph            = */ graph_copy,
    };
// 释放图形拷贝结构体中的缓冲区和上下文内存
void ggml_backend_graph_copy_free(struct ggml_backend_graph_copy copy) {
    // 释放缓冲区
    ggml_backend_buffer_free(copy.buffer);
    // 释放已分配的上下文内存
    ggml_free(copy.ctx_allocated);
    // 释放未分配的上下文内存
    ggml_free(copy.ctx_unallocated);
}

// 比较两个后端的图形，并执行回调函数
bool ggml_backend_compare_graph_backend(ggml_backend_t backend1, ggml_backend_t backend2, struct ggml_cgraph * graph, ggml_backend_eval_callback callback, void * user_data) {
    // 复制后端2的图形
    struct ggml_backend_graph_copy copy = ggml_backend_graph_copy(backend2, graph);
    // 如果复制的缓冲区为空，则返回false
    if (copy.buffer == NULL) {
        return false;
    }

    // 获取图形1和复制的图形2
    struct ggml_cgraph * g1 = graph;
    struct ggml_cgraph * g2 = copy.graph;

    // 断言图形1和图形2的节点数相同
    assert(g1->n_nodes == g2->n_nodes);

    // 遍历图形的节点
    for (int i = 0; i < g1->n_nodes; i++) {
        // 获取当前节点的张量
        struct ggml_tensor * t1 = g1->nodes[i];
        struct ggml_tensor * t2 = g2->nodes[i];

        // 断言当前节点的操作和布局相同
        assert(t1->op == t2->op && ggml_are_same_layout(t1, t2));

        // 创建图形视图
        struct ggml_cgraph g1v = ggml_graph_view(g1, i, i + 1);
        struct ggml_cgraph g2v = ggml_graph_view(g2, i, i + 1);

        // 计算图形1和图形2的结果
        ggml_backend_graph_compute(backend1, &g1v);
        ggml_backend_graph_compute(backend2, &g2v);

        // 如果当前节点是视图操作，则继续下一个节点
        if (ggml_is_view_op(t1->op)) {
            continue;
        }

        // 比较结果，计算均方根等
        if (!callback(i, t1, t2, user_data)) {
            break;
        }
    }

    // 释放图形拷贝结构体
    ggml_backend_graph_copy_free(copy);

    return true;
}
```