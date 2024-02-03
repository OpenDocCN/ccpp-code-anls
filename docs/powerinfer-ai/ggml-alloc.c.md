# `PowerInfer\ggml-alloc.c`

```cpp
#include "ggml-alloc.h"
#include "ggml-backend-impl.h"
#include "ggml.h"
#include "ggml-impl.h"
#include <assert.h>
#include <limits.h>
#include <stdarg.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MAX(a, b) ((a) > (b) ? (a) : (b))  // 定义一个宏，返回两个数中的最大值
#define MAX_FREE_BLOCKS 256  // 定义最大的空闲块数量

//#define GGML_ALLOCATOR_DEBUG  // 定义调试宏

//#define AT_PRINTF(...) fprintf(stderr, __VA_ARGS__)  // 定义一个宏，用于打印调试信息
#define AT_PRINTF(...)  // 空宏，用于屏蔽打印调试信息

// TODO: GGML_PAD ?
static size_t aligned_offset(const void * buffer, size_t offset, size_t alignment) {
    assert(alignment && !(alignment & (alignment - 1))); // power of 2
    // 计算对齐偏移量
    size_t align = (alignment - (((uintptr_t)buffer + offset) % alignment)) % alignment;
    return offset + align;
}

struct free_block {
    void * addr;
    size_t size;
};

struct ggml_tallocr {
    struct ggml_backend_buffer * buffer;
    bool buffer_owned;
    void * base;
    size_t alignment;

    int n_free_blocks;
    struct free_block free_blocks[MAX_FREE_BLOCKS];

    size_t max_size;

    bool measure;

#ifdef GGML_ALLOCATOR_DEBUG
    struct ggml_tensor * allocated_tensors[1024];
#endif
};

#ifdef GGML_ALLOCATOR_DEBUG
static void add_allocated_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    for (int i = 0; i < 1024; i++) {
        if (alloc->allocated_tensors[i] == NULL) {
            alloc->allocated_tensors[i] = tensor;
            return;
        }
    }
    GGML_ASSERT(!"out of allocated_tensors");
}
static void remove_allocated_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    for (int i = 0; i < 1024; i++) {
        if (alloc->allocated_tensors[i] == tensor ||
            (alloc->allocated_tensors[i] != NULL && alloc->allocated_tensors[i]->data == tensor->data)) {
            alloc->allocated_tensors[i] = NULL;
            return;
        }
    }
    printf("tried to free tensor %s not found\n", tensor->name);
    GGML_ASSERT(!"tensor not found");
}
#endif

// check if a tensor is allocated by this buffer
# 检查分配器是否拥有指定张量的缓冲区
static bool ggml_tallocr_is_own(ggml_tallocr_t alloc, const struct ggml_tensor * tensor) {
    return tensor->buffer == alloc->buffer;
}

# 检查张量是否是视图
static bool ggml_is_view(struct ggml_tensor * t) {
    return t->view_src != NULL;
}

# 分配张量的内存
void ggml_tallocr_alloc(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    GGML_ASSERT(!ggml_is_view(tensor)); // 视图通常从它们的源中获取数据指针
    GGML_ASSERT(tensor->data == NULL); // 避免为已经分配内存的张量分配内存

    # 获取需要分配的内存大小
    size_t size = ggml_backend_buffer_get_alloc_size(alloc->buffer, tensor);
    size = aligned_offset(NULL, size, alloc->alignment);

    AT_PRINTF("%s: allocating %s (%zu bytes) - ", __func__, tensor->name, size);

    size_t max_avail = 0;

    # 寻找除了最后一个块之外最适合的空闲块
    int best_fit_block = -1;
    size_t best_fit_size = SIZE_MAX;
    for (int i = 0; i < alloc->n_free_blocks - 1; i++) {
        struct free_block * block = &alloc->free_blocks[i];
        max_avail = MAX(max_avail, block->size);
        if (block->size >= size && block->size <= best_fit_size) {
            best_fit_block = i;
            best_fit_size = block->size;
        }
    }

    AT_PRINTF("block %d\n", best_fit_block);

    if (best_fit_block == -1) {
        # 最后一个块是我们的最后手段
        struct free_block * block = &alloc->free_blocks[alloc->n_free_blocks - 1];
        max_avail = MAX(max_avail, block->size);
        if (block->size >= size) {
            best_fit_block = alloc->n_free_blocks - 1;
        } else {
            fprintf(stderr, "%s: not enough space in the buffer (needed %zu, largest block available %zu)\n",
                    __func__, size, max_avail);
            GGML_ASSERT(!"not enough space in the buffer");
            return;
        }
    }
    struct free_block * block = &alloc->free_blocks[best_fit_block];
    void * addr = block->addr;
    block->addr = (char*)block->addr + size;
    block->size -= size;
}
    # 如果块的大小为0
    if (block->size == 0) {
        # 减少空块数量
        alloc->n_free_blocks--;
        # 从最佳匹配块开始，将后面的空块向前移动一个位置
        for (int j = best_fit_block; j < alloc->n_free_blocks; j++) {
            alloc->free_blocks[j] = alloc->free_blocks[j+1];
        }
    }

    # 将张量的数据指针指向地址
    tensor->data = addr;
    # 将张量的缓冲区指针指向分配的缓冲区
    tensor->buffer = alloc->buffer;
    # 如果不需要测量
    if (!alloc->measure) {
        # 初始化张量的缓冲区
        ggml_backend_buffer_init_tensor(alloc->buffer, tensor);
    }
#ifdef GGML_ALLOCATOR_DEBUG
    // 如果定义了 GGML_ALLOCATOR_DEBUG 宏，则执行以下代码块
    add_allocated_tensor(alloc, tensor);
    // 将分配的张量添加到分配列表中
    size_t cur_max = (char*)addr - (char*)alloc->data + size;
    // 计算当前最大的内存块大小
    if (cur_max > alloc->max_size) {
        // 如果当前最大的内存块大小大于已记录的最大大小
        printf("max_size = %.2f MB: tensors: ", cur_max / 1024.0 / 1024.0);
        // 打印当前最大内存块大小，并遍历已分配的张量
        for (int i = 0; i < 1024; i++) {
            // 遍历已分配的张量列表
            if (alloc->allocated_tensors[i]) {
                // 如果存在已分配的张量
                printf("%s (%.2f MB) ", alloc->allocated_tensors[i]->name, ggml_nbytes(alloc->allocated_tensors[i]) / 1024.0 / 1024.0);
                // 打印张量名称和占用内存大小
            }
        }
        printf("\n");
        // 打印换行符
    }
#endif

    alloc->max_size = MAX(alloc->max_size, (char*)addr - (char*)alloc->base + size);
    // 更新已记录的最大内存块大小

// this is a very naive implementation, but for our case the number of free blocks should be very small
// 这是一个非常简单的实现，但在我们的情况下，空闲块的数量应该非常小
static void ggml_tallocr_free_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    // 释放张量的函数
    if (ggml_tallocr_is_own(alloc, tensor) == false) {
        // 如果张量不是在该缓冲区中分配的
        // 这可能是因为图形分配器会尝试从不同的缓冲区释放权重和其他张量
        // 处理这种情况最简单的方法就是忽略它
        // AT_PRINTF("ignoring %s (their buffer: %p, our buffer: %p)\n", tensor->name, (void *)tensor->buffer, (void *)alloc->buffer);
        // 打印忽略的张量信息
        return;
        // 返回
    }

    void * ptr = tensor->data;
    // 获取张量数据的指针

    size_t size = ggml_backend_buffer_get_alloc_size(alloc->buffer, tensor);
    // 获取张量在缓冲区中的分配大小
    size = aligned_offset(NULL, size, alloc->alignment);
    // 根据对齐方式调整大小
    AT_PRINTF("%s: freeing %s at %p (%zu bytes) - n_free_blocks = %d\n", __func__, tensor->name, ptr, size, alloc->n_free_blocks);
    // 打印释放张量的信息

    if (!alloc->measure) {
        // 如果不需要测量
        ggml_backend_buffer_free_tensor(alloc->buffer, tensor);
        // 释放张量在缓冲区中的内存
    }

#ifdef GGML_ALLOCATOR_DEBUG
    remove_allocated_tensor(alloc, tensor);
    // 如果定义了 GGML_ALLOCATOR_DEBUG 宏，则从分配列表中移除张量
#endif

    // see if we can merge with an existing block
    // 查看是否可以与现有块合并
}
    // 遍历自由块列表中的每个自由块
    for (int i = 0; i < alloc->n_free_blocks; i++) {
        // 获取当前自由块的指针
        struct free_block * block = &alloc->free_blocks[i];
        // 检查指针是否在块的末尾
        if ((char*)block->addr + block->size == ptr) {
            // 如果是，则扩大块的大小
            block->size += size;
            // 检查是否可以与下一个块合并
            if (i < alloc->n_free_blocks - 1 && (char*)block->addr + block->size == alloc->free_blocks[i+1].addr) {
                // 如果可以，则合并块并更新自由块数量
                block->size += alloc->free_blocks[i+1].size;
                alloc->n_free_blocks--;
                // 将后续自由块向前移动
                for (int j = i+1; j < alloc->n_free_blocks; j++) {
                    alloc->free_blocks[j] = alloc->free_blocks[j+1];
                }
            }
            return;
        }
        // 检查指针是否在块的开头
        if ((char*)ptr + size == block->addr) {
            // 如果是，则更新块的地址和大小
            block->addr = ptr;
            block->size += size;
            // 检查是否可以与前一个块合并
            if (i > 0 && (char*)alloc->free_blocks[i-1].addr + alloc->free_blocks[i-1].size == block->addr) {
                // 如果可以，则合并块并更新自由块数量
                alloc->free_blocks[i-1].size += block->size;
                alloc->n_free_blocks--;
                // 将后续自由块向前移动
                for (int j = i; j < alloc->n_free_blocks; j++) {
                    alloc->free_blocks[j] = alloc->free_blocks[j+1];
                }
            }
            return;
        }
    }
    // 否则，添加一个新的块
    GGML_ASSERT(alloc->n_free_blocks < MAX_FREE_BLOCKS && "out of free blocks");
    // 将新块插入正确的位置，以保持数组按地址排序（以加快合并块的速度）
    int insert_pos = 0;
    while (insert_pos < alloc->n_free_blocks && alloc->free_blocks[insert_pos].addr < ptr) {
        insert_pos++;
    }
    // 将从插入位置开始的所有块向后移动，为新块腾出空间
    for (int i = alloc->n_free_blocks; i > insert_pos; i--) {
        alloc->free_blocks[i] = alloc->free_blocks[i-1];
    }
    // 插入新块
    # 将指针地址赋值给空闲块数组中指定位置的地址字段
    alloc->free_blocks[insert_pos].addr = ptr;
    # 将大小赋值给空闲块数组中指定位置的大小字段
    alloc->free_blocks[insert_pos].size = size;
    # 空闲块数量加一
    alloc->n_free_blocks++;
// 重置分配器，将空闲块数量设置为1
void ggml_tallocr_reset(ggml_tallocr_t alloc) {
    alloc->n_free_blocks = 1;
    // 计算对齐偏移量
    size_t align_offset = aligned_offset(alloc->base, 0, alloc->alignment);
    // 设置第一个空闲块的地址为基地址加上对齐偏移量
    alloc->free_blocks[0].addr = (char *)alloc->base + align_offset;

    // 如果是测量模式
    if (alloc->measure) {
        // 限制测量分配器的最大大小为 size_t 最大值的一半，以避免溢出
        alloc->free_blocks[0].size = SIZE_MAX/2;
    } else {
        // 否则设置第一个空闲块的大小为缓冲区大小减去对齐偏移量
        alloc->free_blocks[0].size = ggml_backend_buffer_get_size(alloc->buffer) - align_offset;
    }
}

// 创建新的分配器
ggml_tallocr_t ggml_tallocr_new(void * data, size_t size, size_t alignment) {
    // 从给定数据和大小创建 CPU 缓冲区
    struct ggml_backend_buffer * buffer = ggml_backend_cpu_buffer_from_ptr(NULL, data, size);

    // 分配内存给分配器
    ggml_tallocr_t alloc = (ggml_tallocr_t)malloc(sizeof(struct ggml_tallocr));

    // 初始化分配器的属性
    *alloc = (struct ggml_tallocr) {
        /*.buffer        = */ buffer,
        /*.buffer_owned  = */ true,
        /*.base          = */ ggml_backend_buffer_get_base(buffer),
        /*.alignment     = */ alignment,
        /*.n_free_blocks = */ 0,
        /*.free_blocks   = */ {{0}},
        /*.max_size      = */ 0,
        /*.measure       = */ false,
#ifdef GGML_ALLOCATOR_DEBUG
        /*.allocated_tensors = */ {0},
#endif
    };

    // 重置分配器
    ggml_tallocr_reset(alloc);

    return alloc;
}

// 创建新的测量模式下的分配器
ggml_tallocr_t ggml_tallocr_new_measure(size_t alignment) {
    // 创建新的分配器，初始数据地址为0x1000，大小为 size_t 最大值的一半
    ggml_tallocr_t alloc = ggml_tallocr_new((void *)0x1000, SIZE_MAX/2, alignment);
    // 设置为测量模式
    alloc->measure = true;

    return alloc;
}

// 从后端创建新的测量模式下的分配器
ggml_tallocr_t ggml_tallocr_new_measure_from_backend(struct ggml_backend * backend) {
    // 创建后端缓冲区以获取正确的张量分配大小
    ggml_backend_buffer_t buffer = ggml_backend_alloc_buffer(backend, 1);

    // 创建新的分配器
    ggml_tallocr_t alloc = ggml_tallocr_new_from_buffer(buffer);
    // 设置缓冲区所有权为真
    alloc->buffer_owned = true;
    // 设置为测量模式
    alloc->measure = true;
    // 重置分配器
    ggml_tallocr_reset(alloc);
    return alloc;
}
// 从后端创建一个新的 tallocr 对象，分配指定大小的内存
ggml_tallocr_t ggml_tallocr_new_from_backend(struct ggml_backend * backend, size_t size) {
    // 通过后端分配指定大小的缓冲区
    ggml_backend_buffer_t buffer = ggml_backend_alloc_buffer(backend, size);
    // 从缓冲区创建一个新的 tallocr 对象
    ggml_tallocr_t alloc = ggml_tallocr_new_from_buffer(buffer);
    // 设置 tallocr 对象的 buffer_owned 属性为 true
    alloc->buffer_owned = true;
    // 返回创建的 tallocr 对象
    return alloc;
}

// 从缓冲区创建一个新的 tallocr 对象
ggml_tallocr_t ggml_tallocr_new_from_buffer(struct ggml_backend_buffer * buffer) {
    // 分配内存以存储 tallocr 对象
    ggml_tallocr_t alloc = (ggml_tallocr_t)malloc(sizeof(struct ggml_tallocr));

    // 初始化 tallocr 对象的各个属性
    *alloc = (struct ggml_tallocr) {
        /*.buffer        = */ buffer,
        /*.buffer_owned  = */ false,
        /*.base          = */ ggml_backend_buffer_get_base(buffer),
        /*.alignment     = */ ggml_backend_buffer_get_alignment(buffer),
        /*.n_free_blocks = */ 0,
        /*.free_blocks   = */ {{0}},
        /*.max_size      = */ 0,
        /*.measure       = */ false,
#ifdef GGML_ALLOCATOR_DEBUG
        /*.allocated_tensors = */ {0},
#endif
    };

    // 重置 tallocr 对象
    ggml_tallocr_reset(alloc);

    // 返回创建的 tallocr 对象
    return alloc;
}

// 获取 tallocr 对象的缓冲区
struct ggml_backend_buffer * ggml_tallocr_get_buffer(ggml_tallocr_t alloc) {
    return alloc->buffer;
}

// 释放 tallocr 对象
void ggml_tallocr_free(ggml_tallocr_t alloc) {
    // 如果 tallocr 对象为空，则直接返回
    if (alloc == NULL) {
        return;
    }

    // 如果 tallocr 对象拥有缓冲区，则释放缓冲区
    if (alloc->buffer_owned) {
        ggml_backend_buffer_free(alloc->buffer);
    }
    // 释放 tallocr 对象的内存
    free(alloc);
}

// 检查 tallocr 对象是否处于测量状态
bool ggml_tallocr_is_measure(ggml_tallocr_t alloc) {
    return alloc->measure;
}

// 获取 tallocr 对象的最大大小
size_t ggml_tallocr_max_size(ggml_tallocr_t alloc) {
    return alloc->max_size;
}

// 图形分配器

// 哈希节点结构
struct hash_node {
    int n_children;
    int n_views;
};

// 图形分配器结构
struct ggml_gallocr {
    ggml_tallocr_t talloc;
    struct ggml_hash_set hash_set;
    struct hash_node * hash_values;
    size_t hash_values_size;
    ggml_tallocr_t * hash_allocs;
    int * parse_seq;
    int parse_seq_len;
};

// 创建一个新的图形分配器对象
ggml_gallocr_t ggml_gallocr_new(void) {
    // 分配内存以存储图形分配器对象
    ggml_gallocr_t galloc = (ggml_gallocr_t)malloc(sizeof(struct ggml_gallocr));
    # 初始化一个结构体 ggml_gallocr 的实例 *galloc
    *galloc = (struct ggml_gallocr) {
        # 分配器指针初始化为 NULL
        /*.talloc           = */ NULL,
        # 哈希集合初始化为 0
        /*.hash_set         = */ {0},
        # 哈希值指针初始化为 NULL
        /*.hash_values      = */ NULL,
        # 哈希值大小初始化为 0
        /*.hash_values_size = */ 0,
        # 哈希分配器指针初始化为 NULL
        /*.hash_allocs      = */ NULL,
        # 解析序列指针初始化为 NULL
        /*.parse_seq        = */ NULL,
        # 解析序列长度初始化为 0
        /*.parse_seq_len    = */ 0,
    };

    # 返回初始化后的 *galloc
    return galloc;
}

// 释放内存分配器的函数
void ggml_gallocr_free(ggml_gallocr_t galloc) {
    // 如果内存分配器为空，则直接返回
    if (galloc == NULL) {
        return;
    }

    // 释放哈希表的键数组
    if (galloc->hash_set.keys != NULL) {
        free(galloc->hash_set.keys);
    }
    // 释放哈希表的值数组
    if (galloc->hash_values != NULL) {
        free(galloc->hash_values);
    }
    // 释放哈希表的内存分配器数组
    if (galloc->hash_allocs != NULL) {
        free(galloc->hash_allocs);
    }
    // 释放解析序列数组
    if (galloc->parse_seq != NULL) {
        free(galloc->parse_seq);
    }
    // 释放内存分配器本身
    free(galloc);
}

// 设置解析序列的函数
void ggml_gallocr_set_parse_seq(ggml_gallocr_t galloc, const int * list, int n) {
    // 释放原有的解析序列数组
    free(galloc->parse_seq);
    // 重新分配内存并设置新的解析序列数组
    galloc->parse_seq = malloc(sizeof(int) * n);

    // 将传入的列表复制到解析序列数组中
    for (int i = 0; i < n; i++) {
        galloc->parse_seq[i] = list[i];
    }
    // 设置解析序列的长度
    galloc->parse_seq_len = n;
}

// 获取哈希表节点的函数
static struct hash_node * hash_get(ggml_gallocr_t galloc, struct ggml_tensor * t) {
    // 根据哈希表和张量获取哈希值并插入或查找节点
    size_t i = ggml_hash_find_or_insert(galloc->hash_set, t);
    // 返回哈希值对应的节点
    return &galloc->hash_values[i];
}

// 判断两个张量是否具有相同的布局
static bool ggml_are_same_layout(const struct ggml_tensor * a, const struct ggml_tensor * b) {
    // 判断两个张量的类型是否相同
    if (a->type != b->type) {
        return false;
    }
    // 遍历张量的维度，判断每个维度的起始和结束位置是否相同
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        if (a->ne[i] != b->ne[i]) {
            return false;
        }
        if (a->nb[i] != b->nb[i]) {
            return false;
        }
    }
    // 如果以上条件都满足，则返回 true
    return true;
}

// 判断操作是否可以原地执行
static bool ggml_op_can_inplace(enum ggml_op op) {
    // 根据操作类型判断是否可以原地执行
    switch (op) {
        case GGML_OP_SCALE:
        case GGML_OP_DIAG_MASK_ZERO:
        case GGML_OP_DIAG_MASK_INF:
        case GGML_OP_ADD:
        case GGML_OP_ADD1:
        case GGML_OP_SUB:
        case GGML_OP_MUL:
        case GGML_OP_DIV:
        case GGML_OP_SQR:
        case GGML_OP_SQRT:
        case GGML_OP_LOG:
        case GGML_OP_UNARY:
        case GGML_OP_ROPE:
        case GGML_OP_RMS_NORM:
        case GGML_OP_SOFT_MAX:
            return true;

        default:
            return false;
    }
}

// 获取节点的内存分配器
static ggml_tallocr_t node_tallocr(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    // 如果内存分配器不为空，则返回内存分配器
    if (galloc->talloc != NULL) {
        return galloc->talloc;
    # 返回galloc->hash_allocs中索引为ggml_hash_find_or_insert(galloc->hash_set, node)的元素
    return galloc->hash_allocs[ggml_hash_find_or_insert(galloc->hash_set, node)];
// 初始化视图，使用给定的内存分配器和视图结构体，更新后端标志
static void init_view(ggml_gallocr_t galloc, struct ggml_tensor * view, bool update_backend) {
    ggml_tallocr_t alloc = node_tallocr(galloc, view);

    // 打印视图的名称和来源名称
    //printf("init_view: %s from src %s\n", view->name, view->view_src->name);
    // 断言视图的来源和数据不为空
    GGML_ASSERT(view->view_src != NULL && view->view_src->data != NULL);
    // 如果需要更新后端，则将视图的后端设置为来源视图的后端
    if (update_backend) {
        view->backend = view->view_src->backend;
    }
    // 将视图的缓冲区设置为来源视图的缓冲区
    view->buffer  = view->view_src->buffer;
    // 将视图的数据设置为来源视图数据的偏移量
    view->data    = (char *)view->view_src->data + view->view_offs;

    // FIXME: 视图应该由拥有缓冲区的初始化，但目前这会破坏 CUDA 后端，因为 ggml_tensor_extra_gpu 环形缓冲区会覆盖 KV 缓存额外信息
    // 断言如果不是度量或者视图的缓冲区为空，或者视图的缓冲区后端与分配器的缓冲区后端相同
    assert(ggml_tallocr_is_measure(alloc) || !view->buffer || view->buffer->backend == alloc->buffer->backend);

    // 如果不是度量，则初始化视图的后端缓冲区
    if (!alloc->measure) {
        ggml_backend_buffer_init_tensor(alloc->buffer, view);
    }
}

// 分配节点，使用给定的内存分配器和节点结构体
static void allocate_node(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    ggml_tallocr_t alloc = node_tallocr(galloc, node);

    // 空函数体
}

// 释放节点，使用给定的内存分配器和节点结构体
static void free_node(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    ggml_tallocr_t alloc = node_tallocr(galloc, node);

    // 释放节点的内存
    ggml_tallocr_free_tensor(alloc, node);
}

// 分配图形的内存，使用给定的内存分配器和计算图结构体
static void ggml_tallocr_alloc_graph_impl(ggml_gallocr_t galloc, struct ggml_cgraph * gf) {
    const int * parse_seq     = galloc->parse_seq;
    int         parse_seq_len = galloc->parse_seq_len;

    // 计算子节点和视图的数量
}
    // 遍历图中的节点
    for (int i = 0; i < gf->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor * node = gf->nodes[i];

        // 如果当前节点是视图
        if (ggml_is_view(node)) {
            // 获取视图的源节点
            struct ggml_tensor * view_src = node->view_src;
            // 增加视图源节点的视图数量
            hash_get(galloc, view_src)->n_views += 1;
            // 如果当前节点的缓冲区为空且数据不为空
            if (node->buffer == NULL && node->data != NULL) {
                // 初始化视图
                init_view(galloc, node, true);
            }
        }

        // 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取当前源节点
            struct ggml_tensor * parent = node->src[j];
            // 如果当前源节点为空，跳出循环
            if (parent == NULL) {
                break;
            }
            // 增加当前源节点的子节点数量
            hash_get(galloc, parent)->n_children += 1;
            // 如果当前源节点是视图且缓冲区为空且数据不为空
            if (ggml_is_view(parent) && parent->buffer == NULL && parent->data != NULL) {
                // 初始化视图
                init_view(galloc, parent, true);
            }
        }
   }

    // 分配张量
    // 如果有 parse_seq，则按照列表分配节点，只在屏障处释放节点
    int last_barrier_pos = 0;
    // 计算节点数量
    int n_nodes = parse_seq_len ? parse_seq_len : gf->n_nodes;
    // 结束大括号缺失
    }
# 分配内存给图形对象
size_t ggml_gallocr_alloc_graph(ggml_gallocr_t galloc, ggml_tallocr_t talloc, struct ggml_cgraph * graph) {
    # 获取图形对象中访问过的哈希表的大小
    size_t hash_size = graph->visited_hash_table.size;

    # 检查哈希表是否已初始化并且足够大
    if (galloc->hash_set.size < hash_size) {
        # 如果哈希表的大小不够，释放旧的内存
        if (galloc->hash_set.keys != NULL) {
            free(galloc->hash_set.keys);
        }
        if (galloc->hash_values != NULL) {
            free(galloc->hash_values);
        }
        # 为哈希表的键和值分配新的内存
        galloc->hash_set.keys = malloc(sizeof(struct ggml_tensor *) * hash_size);
        galloc->hash_set.size = hash_size;
        galloc->hash_values = malloc(sizeof(struct hash_node) * hash_size);
    }

    # 重置哈希表
    memset(galloc->hash_set.keys, 0, sizeof(struct ggml_tensor *) * hash_size);
    memset(galloc->hash_values,   0, sizeof(struct hash_node) * hash_size);

    # 设置图形对象的内存分配器
    galloc->talloc = talloc;
    # 分配图形对象的内存
    ggml_tallocr_alloc_graph_impl(galloc, graph);
    galloc->talloc = NULL;

    # 获取内存分配器的最大大小
    size_t max_size = ggml_tallocr_max_size(talloc);

    # 返回最大大小
    return max_size;
}

# 分配图形对象的哈希表
void ggml_gallocr_alloc_graph_n(ggml_gallocr_t galloc, struct ggml_cgraph * graph, struct ggml_hash_set hash_set, ggml_tallocr_t * hash_node_talloc) {
    # 获取哈希表的大小
    const size_t hash_size = hash_set.size;

    # 断言哈希表的大小大于等于图形对象中节点和叶子节点的数量
    GGML_ASSERT(hash_size >= (size_t)(graph->n_nodes + graph->n_leafs));

    # 设置内存分配器为NULL
    galloc->talloc = NULL;

    # 如果哈希值为空或者哈希值的大小不够，重新分配内存
    if (galloc->hash_values == NULL || galloc->hash_values_size < hash_size) {
        free(galloc->hash_values);
        galloc->hash_values      = malloc(sizeof(struct hash_node) * hash_size);
        galloc->hash_values_size = hash_size;
    }

    # 如果哈希表的键不为空，释放内存
    if (galloc->hash_set.keys != NULL) {
        free(galloc->hash_set.keys);
    }
    # 设置哈希表
    galloc->hash_set = hash_set;

    # 重置哈希值
    memset(galloc->hash_values, 0, sizeof(struct hash_node) * hash_size);

    # 设置哈希分配器
    galloc->hash_allocs = hash_node_talloc;

    # 分配图形对象的内存
    ggml_tallocr_alloc_graph_impl(galloc, graph);

    # 移除未拥有的资源
}
    # 将 galloc 结构体中的 hash_set.keys 成员赋值为 NULL
    galloc->hash_set.keys = NULL;
    # 将 galloc 结构体中的 hash_allocs 成员赋值为 NULL
    galloc->hash_allocs = NULL;
// 定义一个结构体 ggml_allocr，包含 talloc 和 galloc 两个成员
struct ggml_allocr {
    ggml_tallocr_t talloc;
    ggml_gallocr_t galloc;
};

// 定义一个静态函数 ggml_allocr_new_impl，接受一个 tallocr_t 类型参数 talloc
static ggml_allocr_t ggml_allocr_new_impl(ggml_tallocr_t talloc) {
    // 分配内存给 alloc，并初始化为 ggml_allocr 结构体
    ggml_allocr_t alloc = (ggml_allocr_t)malloc(sizeof(struct ggml_allocr));
    *alloc = (struct ggml_allocr) {
        /*.talloc = */ talloc,  // 初始化 talloc 成员
        /*.galloc = */ ggml_gallocr_new(),  // 初始化 galloc 成员
    };
    return alloc;  // 返回初始化后的 alloc
}

// 定义函数 ggml_allocr_new，接受 data、size、alignment 三个参数
ggml_allocr_t ggml_allocr_new(void * data, size_t size, size_t alignment) {
    return ggml_allocr_new_impl(ggml_tallocr_new(data, size, alignment));  // 调用 ggml_allocr_new_impl 函数并返回结果
}

// 定义函数 ggml_allocr_new_measure，接受 alignment 参数
ggml_allocr_t ggml_allocr_new_measure(size_t alignment) {
    return ggml_allocr_new_impl(ggml_tallocr_new_measure(alignment));  // 调用 ggml_allocr_new_impl 函数并返回结果
}

// 定义函数 ggml_allocr_new_from_buffer，接受一个指向 ggml_backend_buffer 结构体的指针参数 buffer
ggml_allocr_t ggml_allocr_new_from_buffer(struct ggml_backend_buffer * buffer) {
    return ggml_allocr_new_impl(ggml_tallocr_new_from_buffer(buffer));  // 调用 ggml_allocr_new_impl 函数并返回结果
}

// 定义函数 ggml_allocr_new_from_backend，接受一个指向 ggml_backend 结构体的指针参数 backend 和 size 参数
ggml_allocr_t ggml_allocr_new_from_backend(struct ggml_backend * backend, size_t size) {
    return ggml_allocr_new_impl(ggml_tallocr_new_from_backend(backend, size));  // 调用 ggml_allocr_new_impl 函数并返回结果
}

// 定义函数 ggml_allocr_new_measure_from_backend，接受一个指向 ggml_backend 结构体的指针参数 backend
ggml_allocr_t ggml_allocr_new_measure_from_backend(struct ggml_backend * backend) {
    return ggml_allocr_new_impl(ggml_tallocr_new_measure_from_backend(backend));  // 调用 ggml_allocr_new_impl 函数并返回结果
}

// 定义函数 ggml_allocr_get_buffer，接受一个 ggml_allocr_t 类型参数 alloc
struct ggml_backend_buffer * ggml_allocr_get_buffer(ggml_allocr_t alloc) {
    return ggml_tallocr_get_buffer(alloc->talloc);  // 返回 alloc 的 talloc 成员的 buffer
}

// 定义函数 ggml_allocr_set_parse_seq，接受一个 ggml_allocr_t 类型参数 alloc，一个指向 int 的指针 list，和一个 int 类型参数 n
void ggml_allocr_set_parse_seq(ggml_allocr_t alloc, const int * list, int n) {
    ggml_gallocr_set_parse_seq(alloc->galloc, list, n);  // 调用 ggml_gallocr_set_parse_seq 函数
}

// 定义函数 ggml_allocr_free，接受一个 ggml_allocr_t 类型参数 alloc
void ggml_allocr_free(ggml_allocr_t alloc) {
    ggml_gallocr_free(alloc->galloc);  // 调用 ggml_gallocr_free 函数
    ggml_tallocr_free(alloc->talloc);  // 调用 ggml_tallocr_free 函数
    free(alloc);  // 释放 alloc 的内存
}

// 定义函数 ggml_allocr_is_measure，接受一个 ggml_allocr_t 类型参数 alloc
bool ggml_allocr_is_measure(ggml_allocr_t alloc) {
    return ggml_tallocr_is_measure(alloc->talloc);  // 返回 alloc 的 talloc 成员的是否为 measure
}

// 定义函数 ggml_allocr_reset，接受一个 ggml_allocr_t 类型参数 alloc
void ggml_allocr_reset(ggml_allocr_t alloc) {
    ggml_tallocr_reset(alloc->talloc);  // 调用 ggml_tallocr_reset 函数
}

// 定义函数 ggml_allocr_alloc，接受一个 ggml_allocr_t 类型参数 alloc 和一个指向 ggml_tensor 结构体的指针参数 tensor
void ggml_allocr_alloc(ggml_allocr_t alloc, struct ggml_tensor * tensor) {
    ggml_tallocr_alloc(alloc->talloc, tensor);  // 调用 ggml_tallocr_alloc 函数
}

// 定义函数 ggml_allocr_max_size，接受一个 ggml_allocr_t 类型参数 alloc
size_t ggml_allocr_max_size(ggml_allocr_t alloc) {
    return ggml_tallocr_max_size(alloc->talloc);  // 返回 alloc 的 talloc 成员的最大大小
}
# 定义一个函数，用于在内存中分配一个图形结构
size_t ggml_allocr_alloc_graph(ggml_allocr_t alloc, struct ggml_cgraph * graph) {
    # 调用ggml_gallocr_alloc_graph函数，使用alloc的galloc和talloc成员来分配图形结构
    return ggml_gallocr_alloc_graph(alloc->galloc, alloc->talloc, graph);
}
```