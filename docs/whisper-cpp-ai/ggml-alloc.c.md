# `whisper.cpp\ggml-alloc.c`

```cpp
// 包含必要的头文件
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

// 定义宏，返回两个值中的最大值
#define MAX(a, b) ((a) > (b) ? (a) : (b)
// 定义最大空闲块数
#define MAX_FREE_BLOCKS 256

// 定义宏，用于调试
//#define GGML_ALLOCATOR_DEBUG

// 定义宏，用于打印信息
//#define AT_PRINTF(...) fprintf(stderr, __VA_ARGS__)
#define AT_PRINTF(...)

// 计算对齐偏移量
static size_t aligned_offset(const void * buffer, size_t offset, size_t alignment) {
    // 断言对齐值是2的幂
    assert(alignment && !(alignment & (alignment - 1))); // power of 2
    // 计算对齐偏移量
    size_t align = (alignment - (((uintptr_t)buffer + offset) % alignment)) % alignment;
    return offset + align;
}

// 定义空闲块结构体
struct free_block {
    void * addr;
    size_t size;
};

// 定义内存分配器结构体
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
// 添加已分配的张量
static void add_allocated_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    for (int i = 0; i < 1024; i++) {
        if (alloc->allocated_tensors[i] == NULL) {
            alloc->allocated_tensors[i] = tensor;
            return;
        }
    }
    GGML_ASSERT(!"out of allocated_tensors");
}
// 移除已分配的张量
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

// 检查张量是否由此缓冲区分配
// 检查给定的张量是否使用了指定的内存分配器的缓冲区，并且如果是视图，则检查视图源是否使用了相同的缓冲区
static bool ggml_tallocr_is_own(ggml_tallocr_t alloc, const struct ggml_tensor * tensor) {
    return tensor->buffer == alloc->buffer && (!tensor->view_src || tensor->view_src->buffer == alloc->buffer);
}

// 检查给定的张量是否是视图
static bool ggml_is_view(struct ggml_tensor * t) {
    return t->view_src != NULL;
}

// 为给定的张量分配内存
void ggml_tallocr_alloc(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    GGML_ASSERT(!ggml_is_view(tensor)); // 视图通常从它们的源中获取数据指针
    GGML_ASSERT(tensor->data == NULL); // 避免为已经分配内存的张量再次分配内存

    // 获取需要分配的内存大小
    size_t size = ggml_backend_buffer_get_alloc_size(alloc->buffer, tensor);
    size = aligned_offset(NULL, size, alloc->alignment);

    AT_PRINTF("%s: allocating %s (%zu bytes) - ", __func__, tensor->name, size);

    size_t max_avail = 0;

    // 查找除了最后一个块之外最适合的空闲块
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

    if (best_fit_block == -1) {
        // 最后一个块是我们的最后手段
        struct free_block * block = &alloc->free_blocks[alloc->n_free_blocks - 1];
        max_avail = MAX(max_avail, block->size);
        if (block->size >= size) {
            best_fit_block = alloc->n_free_blocks - 1;
        } else {
            fprintf(stderr, "%s: not enough space in the buffer to allocate %s (needed %zu, largest block available %zu)\n",
                    __func__, tensor->name, size, max_avail);
            GGML_ASSERT(!"not enough space in the buffer");
            return;
        }
    }

    // 获取最适合的空闲块
    struct free_block * block = &alloc->free_blocks[best_fit_block];
    void * addr = block->addr;
}
    // 更新块的地址指针，指向下一个可用内存块的起始地址
    block->addr = (char*)block->addr + size;
    // 减去已分配的内存块大小
    block->size -= size;
    // 如果内存块大小为0，则表示该块已经被完全分配出去
    if (block->size == 0) {
        // 如果内存块为空，从空闲块列表中移除该块
        alloc->n_free_blocks--;
        // 从最佳适配块位置开始，将后续的空闲块向前移动一个位置
        for (int j = best_fit_block; j < alloc->n_free_blocks; j++) {
            alloc->free_blocks[j] = alloc->free_blocks[j+1];
        }
    }

    // 打印块的索引和地址
    AT_PRINTF("block %d, addr %p\n", best_fit_block, addr);

    // 将张量的数据指针指向分配的地址
    tensor->data = addr;
    // 将张量的缓冲区指针指向内存分配器的缓冲区
    tensor->buffer = alloc->buffer;
    // 如果不需要测量内存使用情况，则初始化张量的缓冲区
    if (!alloc->measure) {
        ggml_backend_buffer_init_tensor(alloc->buffer, tensor);
    }
#ifdef GGML_ALLOCATOR_DEBUG
    // 如果定义了 GGML_ALLOCATOR_DEBUG 宏，则执行以下代码块
    // 将分配的张量添加到分配列表中
    add_allocated_tensor(alloc, tensor);
    // 计算当前最大的内存大小
    size_t cur_max = (char*)addr - (char*)alloc->base + size;
    // 如果当前最大内存大小超过了分配器的最大大小，则输出当前内存使用情况
    if (cur_max > alloc->max_size) {
        printf("max_size = %.2f MB: tensors: ", cur_max / 1024.0 / 1024.0);
        // 遍历已分配的张量列表，输出张量名称和占用内存大小
        for (int i = 0; i < 1024; i++) {
            if (alloc->allocated_tensors[i]) {
                printf("%s (%.2f MB) ", alloc->allocated_tensors[i]->name, ggml_nbytes(alloc->allocated_tensors[i]) / 1024.0 / 1024.0);
            }
        }
        printf("\n");
    }
#endif

    // 更新分配器的最大大小
    alloc->max_size = MAX(alloc->max_size, (char*)addr - (char*)alloc->base + size);
}

// 这是一个非常简单的实现，但在我们的情况下，空闲块的数量应该非常小
static void ggml_tallocr_free_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    // 如果张量不是在当前缓冲区中分配的，则忽略
    if (ggml_tallocr_is_own(alloc, tensor) == false) {
        // 张量不是在当前缓冲区中分配的情况下的处理方式
        // 这可能发生在图形分配器尝试从不同缓冲区释放权重和其他张量的情况下
        // 最简单的处理方式是忽略它
        // AT_PRINTF("ignoring %s (their buffer: %p, our buffer: %p)\n", tensor->name, (void *)tensor->buffer, (void *)alloc->buffer);
        return;
    }

    void * ptr = tensor->data;

    // 获取张量的分配大小
    size_t size = ggml_backend_buffer_get_alloc_size(alloc->buffer, tensor);
    size = aligned_offset(NULL, size, alloc->alignment);
    // 输出释放张量的信息
    AT_PRINTF("%s: freeing %s at %p (%zu bytes) - n_free_blocks = %d\n", __func__, tensor->name, ptr, size, alloc->n_free_blocks);

#ifdef GGML_ALLOCATOR_DEBUG
    // 如果定义了 GGML_ALLOCATOR_DEBUG 宏，则执行以下代码块
    // 从分配列表中移除张量
    remove_allocated_tensor(alloc, tensor);
#endif

    // 查看是否可以与现有块合并
    // 遍历自由块数组中的每个块
    for (int i = 0; i < alloc->n_free_blocks; i++) {
        // 获取当前自由块的指针
        struct free_block * block = &alloc->free_blocks[i];
        // 检查指针是否在块的末尾
        if ((char*)block->addr + block->size == ptr) {
            // 增加块的大小
            block->size += size;
            // 检查是否可以与下一个块合并
            if (i < alloc->n_free_blocks - 1 && (char*)block->addr + block->size == alloc->free_blocks[i+1].addr) {
                // 合并块
                block->size += alloc->free_blocks[i+1].size;
                alloc->n_free_blocks--;
                // 移动后续块以填补空缺
                for (int j = i+1; j < alloc->n_free_blocks; j++) {
                    alloc->free_blocks[j] = alloc->free_blocks[j+1];
                }
            }
            return;
        }
        // 检查指针是否在块的开头
        if ((char*)ptr + size == block->addr) {
            // 更新块的地址和大小
            block->addr = ptr;
            block->size += size;
            // 检查是否可以与前一个块合并
            if (i > 0 && (char*)alloc->free_blocks[i-1].addr + alloc->free_blocks[i-1].size == block->addr) {
                // 合并块
                alloc->free_blocks[i-1].size += block->size;
                alloc->n_free_blocks--;
                // 移动后续块以填补空缺
                for (int j = i; j < alloc->n_free_blocks; j++) {
                    alloc->free_blocks[j] = alloc->free_blocks[j+1];
                }
            }
            return;
        }
    }
    // 否则，添加一个新块
    GGML_ASSERT(alloc->n_free_blocks < MAX_FREE_BLOCKS && "out of free blocks");
    // 将新块插入正确的位置，以保持数组按地址排序（以加快合并块的速度）
    int insert_pos = 0;
    while (insert_pos < alloc->n_free_blocks && alloc->free_blocks[insert_pos].addr < ptr) {
        insert_pos++;
    }
    // 移动从插入位置开始的所有块，为新块腾出空间
    for (int i = alloc->n_free_blocks; i > insert_pos; i--) {
        alloc->free_blocks[i] = alloc->free_blocks[i-1];
    }
    // 插入新块
    # 将指针 ptr 存储到 alloc 结构体的 free_blocks 数组中的指定位置的 addr 属性中
    alloc->free_blocks[insert_pos].addr = ptr;
    # 将大小 size 存储到 alloc 结构体的 free_blocks 数组中的指定位置的 size 属性中
    alloc->free_blocks[insert_pos].size = size;
    # 增加 alloc 结构体中的 n_free_blocks 属性的值，表示空闲块的数量增加了
    alloc->n_free_blocks++;
}

// 重置分配器，将空闲块数量设置为1，计算对齐偏移量，设置第一个空闲块的地址和大小
void ggml_tallocr_reset(ggml_tallocr_t alloc) {
    alloc->n_free_blocks = 1;
    size_t align_offset = aligned_offset(alloc->base, 0, alloc->alignment);
    alloc->free_blocks[0].addr = (char *)alloc->base + align_offset;

    // 如果是测量模式，则限制测量分配器的最大大小为 size_t 最大值的一半，以避免溢出
    if (alloc->measure) {
        alloc->free_blocks[0].size = SIZE_MAX/2;
    } else {
        // 否则设置第一个空闲块的大小为缓冲区大小减去对齐偏移量，并重置缓冲区
        alloc->free_blocks[0].size = ggml_backend_buffer_get_size(alloc->buffer) - align_offset;
        ggml_backend_buffer_reset(alloc->buffer);
    }
}

// 创建一个新的分配器，根据给定的数据、大小和对齐方式
ggml_tallocr_t ggml_tallocr_new(void * data, size_t size, size_t alignment) {
    // 从数据和大小创建一个 CPU 缓冲区
    struct ggml_backend_buffer * buffer = ggml_backend_cpu_buffer_from_ptr(data, size);

    // 分配内存给分配器
    ggml_tallocr_t alloc = (ggml_tallocr_t)malloc(sizeof(struct ggml_tallocr));

    // 初始化分配器的各个属性
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

// 创建一个新的测量分配器，根据给定的对齐方式
ggml_tallocr_t ggml_tallocr_new_measure(size_t alignment) {
    // 创建一个测量分配器，初始数据为 0x1000，大小为 size_t 最大值的一半
    ggml_tallocr_t alloc = ggml_tallocr_new((void *)0x1000, SIZE_MAX/2, alignment);
    alloc->measure = true;

    return alloc;
}

// 从给定的缓冲区类型创建一个新的测量分配器
ggml_tallocr_t ggml_tallocr_new_measure_from_buft(struct ggml_backend_buffer_type * buft) {
    // 创建一个后端缓冲区以获取正确的张量分配大小
    ggml_backend_buffer_t buffer = ggml_backend_buft_alloc_buffer(buft, 1);

    // 创建一个新的分配器，并设置为测量模式
    ggml_tallocr_t alloc = ggml_tallocr_new_from_buffer(buffer);
    alloc->buffer_owned = true;
    alloc->measure = true;
    ggml_tallocr_reset(alloc);
    return alloc;
}
// 从后端创建一个新的测量对象
ggml_tallocr_t ggml_tallocr_new_measure_from_backend(struct ggml_backend * backend) {
    // 从默认缓冲类型获取缓冲对象，并创建一个新的测量对象
    return ggml_tallocr_new_measure_from_buft(ggml_backend_get_default_buffer_type(backend));
}

// 从缓冲对象和指定大小创建一个新的分配对象
ggml_tallocr_t ggml_tallocr_new_from_buft(struct ggml_backend_buffer_type * buft, size_t size) {
    // 创建一个后端缓冲对象以获取正确的张量分配大小
    ggml_backend_buffer_t buffer = ggml_backend_buft_alloc_buffer(buft, size);
    // 从缓冲对象创建一个新的分配对象
    ggml_tallocr_t alloc = ggml_tallocr_new_from_buffer(buffer);
    // 设置分配对象拥有缓冲对象的标志为真
    alloc->buffer_owned = true;
    return alloc;
}

// 从后端和指定大小创建一个新的分配对象
ggml_tallocr_t ggml_tallocr_new_from_backend(struct ggml_backend * backend, size_t size) {
    // 从默认缓冲类型获取后端对象，并从缓冲类型和指定大小创建一个新的分配对象
    return ggml_tallocr_new_from_buft(ggml_backend_get_default_buffer_type(backend), size);
}

// 从缓冲对象创建一个新的分配对象
ggml_tallocr_t ggml_tallocr_new_from_buffer(struct ggml_backend_buffer * buffer) {
    // 分配内存以存储分配对象
    ggml_tallocr_t alloc = (ggml_tallocr_t)malloc(sizeof(struct ggml_tallocr));

    // 初始化分配对象的各个字段
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

    // 重置分配对象
    ggml_tallocr_reset(alloc);

    return alloc;
}

// 获取分配对象的缓冲对象
struct ggml_backend_buffer * ggml_tallocr_get_buffer(ggml_tallocr_t alloc) {
    return alloc->buffer;
}

// 释放分配对象
void ggml_tallocr_free(ggml_tallocr_t alloc) {
    // 如果分配对象为空，则直接返回
    if (alloc == NULL) {
        return;
    }

    // 如果分配对象拥有缓冲对象，则释放缓冲对象
    if (alloc->buffer_owned) {
        ggml_backend_buffer_free(alloc->buffer);
    }
    // 释放分配对象内存
    free(alloc);
}

// 检查分配对象是否为测量对象
bool ggml_tallocr_is_measure(ggml_tallocr_t alloc) {
    return alloc->measure;
}

// 获取分配对象的最大大小
size_t ggml_tallocr_max_size(ggml_tallocr_t alloc) {
    // FIXME: 与测量图中张量大小的变化可能导致分配失败
}
    // 为了避免这种情况，我们将缓冲区大小增加10%作为保留空间
    return alloc->max_size + alloc->max_size/10;
// 结构体，用于表示哈希表中的节点，包含子节点数和视图数
struct hash_node {
    int n_children; // 子节点数
    int n_views; // 视图数
};

// 图分配器结构体，包含了内存分配器、哈希集合、哈希值、哈希值大小、哈希分配器、解析序列等信息
struct ggml_gallocr {
    ggml_tallocr_t talloc; // 内存分配器
    struct ggml_hash_set hash_set; // 哈希集合
    struct hash_node * hash_values; // 哈希值数组
    size_t hash_values_size; // 哈希值数组大小
    ggml_tallocr_t * hash_allocs; // 哈希分配器数组
    int * parse_seq; // 解析序列
    int parse_seq_len; // 解析序列长度
};

// 创建新的图分配器对象
ggml_gallocr_t ggml_gallocr_new(void) {
    // 分配内存空间
    ggml_gallocr_t galloc = (ggml_gallocr_t)malloc(sizeof(struct ggml_gallocr));

    // 初始化图分配器对象的各个字段
    *galloc = (struct ggml_gallocr) {
        /*.talloc           = */ NULL,
        /*.hash_set         = */ {0},
        /*.hash_values      = */ NULL,
        /*.hash_values_size = */ 0,
        /*.hash_allocs      = */ NULL,
        /*.parse_seq        = */ NULL,
        /*.parse_seq_len    = */ 0,
    };

    return galloc;
}

// 释放图分配器对象占用的内存空间
void ggml_gallocr_free(ggml_gallocr_t galloc) {
    if (galloc == NULL) {
        return;
    }

    // 释放哈希集合的键
    if (galloc->hash_set.keys != NULL) {
        free(galloc->hash_set.keys);
    }
    // 释放哈希值数组
    if (galloc->hash_values != NULL) {
        free(galloc->hash_values);
    }
    // 释放哈希分配器数组
    if (galloc->hash_allocs != NULL) {
        free(galloc->hash_allocs);
    }
    // 释放解析序列
    if (galloc->parse_seq != NULL) {
        free(galloc->parse_seq);
    }
    // 释放图分配器对象
    free(galloc);
}

// 设置图分配器对象的解析序列
void ggml_gallocr_set_parse_seq(ggml_gallocr_t galloc, const int * list, int n) {
    // 释放原有的解析序列
    free(galloc->parse_seq);
    // 分配新的解析序列内存空间
    galloc->parse_seq = malloc(sizeof(int) * n);

    // 将传入的列表复制到解析序列中
    for (int i = 0; i < n; i++) {
        galloc->parse_seq[i] = list[i];
    }
    // 更新解析序列长度
    galloc->parse_seq_len = n;
}

// 获取哈希值对应的哈希节点
static struct hash_node * hash_get(ggml_gallocr_t galloc, struct ggml_tensor * t) {
    // 查找或插入哈希集合中的元素，并返回对应的索引
    size_t i = ggml_hash_find_or_insert(galloc->hash_set, t);
    // 返回哈希值数组中对应索引的节点
    return &galloc->hash_values[i];
}

// 检查两个张量是否具有相同的布局
static bool ggml_are_same_layout(const struct ggml_tensor * a, const struct ggml_tensor * b) {
    // 如果两个张量的类型不同，则返回false
    if (a->type != b->type) {
        return false;
    }
    # 遍历数组a和数组b的维度
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        # 如果数组a和数组b在当前维度上的元素个数不相等，则返回false
        if (a->ne[i] != b->ne[i]) {
            return false;
        }
        # 如果数组a和数组b在当前维度上的元素字节数不相等，则返回false
        if (a->nb[i] != b->nb[i]) {
            return false;
        }
    }
    # 如果所有维度上的元素个数和字节数都相等，则返回true
    return true;
// 检查给定操作是否支持原地操作
static bool ggml_op_can_inplace(enum ggml_op op) {
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

// 获取节点的分配器
static ggml_tallocr_t node_tallocr(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    if (galloc->talloc != NULL) {
        return galloc->talloc;
    }

    return galloc->hash_allocs[ggml_hash_find_or_insert(galloc->hash_set, node)];
}

// 初始化视图
static void init_view(ggml_gallocr_t galloc, struct ggml_tensor * view, bool update_backend) {
    ggml_tallocr_t alloc = node_tallocr(galloc, view);

    GGML_ASSERT(view->view_src != NULL && view->view_src->data != NULL);
    if (update_backend) {
        view->backend = view->view_src->backend;
    }
    // 视图在分配缓冲区中初始化，而不是在视图源缓冲区中
    view->buffer  = alloc->buffer;
    view->data    = (char *)view->view_src->data + view->view_offs;

    assert(ggml_tallocr_is_measure(alloc) || !view->buffer || view->buffer->buft == alloc->buffer->buft);

    if (!alloc->measure) {
        ggml_backend_buffer_init_tensor(alloc->buffer, view);
    }
}

// 分配节点
static void allocate_node(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    ggml_tallocr_t alloc = node_tallocr(galloc, node);

    // 空函数体
}

// 释放节点
static void free_node(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    ggml_tallocr_t alloc = node_tallocr(galloc, node);

    ggml_tallocr_free_tensor(alloc, node);
}

// 分配图实现
static void ggml_tallocr_alloc_graph_impl(ggml_gallocr_t galloc, struct ggml_cgraph * gf) {
    const int * parse_seq     = galloc->parse_seq;
}
    // 解析序列长度等于全局变量 galloc 的解析序列长度
    int parse_seq_len = galloc->parse_seq_len;

    // 计算子节点和视图的数量
    for (int i = 0; i < gf->n_nodes; i++) {
        // 获取当前节点
        struct ggml_tensor *node = gf->nodes[i];

        // 如果当前节点是视图
        if (ggml_is_view(node)) {
            // 获取视图的源节点
            struct ggml_tensor *view_src = node->view_src;
            // 增加视图源节点的视图数量
            hash_get(galloc, view_src)->n_views += 1;
            // 如果当前节点的缓冲区为空且数据不为空
            if (node->buffer == NULL && node->data != NULL) {
                // 视图是预先分配的张量，尚未调用 init_view() 初始化
                init_view(galloc, node, true);
            }
        }

        // 遍历当前节点的源节点
        for (int j = 0; j < GGML_MAX_SRC; j++) {
            // 获取父节点
            struct ggml_tensor *parent = node->src[j];
            // 如果父节点为空，跳出循环
            if (parent == NULL) {
                break;
            }
            // 增加父节点的子节点数量
            hash_get(galloc, parent)->n_children += 1;
            // 如果父节点是视图且缓冲区为空且数据不为空
            if (ggml_is_view(parent) && parent->buffer == NULL && parent->data != NULL) {
                // 初始化视图
                init_view(galloc, parent, true);
            }
        }
    }

    // 分配张量
    // 如果有解析序列，则按照列表分配节点，并且只在障碍处释放节点
    int last_barrier_pos = 0;
    // 节点数量等于解析序列长度或者节点数量
    int n_nodes = parse_seq_len ? parse_seq_len : gf->n_nodes;
// 分配图形资源，返回最大分配大小
size_t ggml_gallocr_alloc_graph(ggml_gallocr_t galloc, ggml_tallocr_t talloc, struct ggml_cgraph * graph) {
    // 获取图形中访问哈希表的大小
    size_t hash_size = graph->visited_hash_table.size;

    // 检查哈希表是否已初始化并足够大
    if (galloc->hash_set.size < hash_size) {
        // 释放旧的哈希表键和值
        if (galloc->hash_set.keys != NULL) {
            free(galloc->hash_set.keys);
        }
        if (galloc->hash_values != NULL) {
            free(galloc->hash_values);
        }
        // 分配新的哈希表键和值
        galloc->hash_set.keys = malloc(sizeof(struct ggml_tensor *) * hash_size);
        galloc->hash_set.size = hash_size;
        galloc->hash_values = malloc(sizeof(struct hash_node) * hash_size);
    }

    // 重置哈希表
    memset(galloc->hash_set.keys, 0, sizeof(struct ggml_tensor *) * hash_size);
    memset(galloc->hash_values, 0, sizeof(struct hash_node) * hash_size);

    galloc->talloc = talloc;
    // 分配图形资源的实现
    ggml_tallocr_alloc_graph_impl(galloc, graph);
    galloc->talloc = NULL;

    // 获取分配器的最大大小
    size_t max_size = ggml_tallocr_max_size(talloc);

    return max_size;
}

// 分配图形资源，带哈希表和哈希节点分配器
void ggml_gallocr_alloc_graph_n(ggml_gallocr_t galloc, struct ggml_cgraph * graph, struct ggml_hash_set hash_set, ggml_tallocr_t * hash_node_talloc) {
    // 获取哈希表的大小
    const size_t hash_size = hash_set.size;

    // 断言哈希表大小大于等于图形中节点和叶子节点的数量
    GGML_ASSERT(hash_size >= (size_t)(graph->n_nodes + graph->n_leafs));

    galloc->talloc = NULL;

    // 如果哈希值为空或大小不足，则重新分配哈希值
    if (galloc->hash_values == NULL || galloc->hash_values_size < hash_size) {
        free(galloc->hash_values);
        galloc->hash_values = malloc(sizeof(struct hash_node) * hash_size);
        galloc->hash_values_size = hash_size;
    }

    // 如果哈希表键不为空，则释放
    if (galloc->hash_set.keys != NULL) {
        free(galloc->hash_set.keys);
    }
    // 设置哈希表
    galloc->hash_set = hash_set;

    // 重置哈希值
    memset(galloc->hash_values, 0, sizeof(struct hash_node) * hash_size);

    galloc->hash_allocs = hash_node_talloc;

    // 分配图形资源的实现
    ggml_tallocr_alloc_graph_impl(galloc, graph);

    // 移除未拥有的资源
}
    # 将 galloc 结构体中的 hash_set.keys 成员设置为 NULL
    galloc->hash_set.keys = NULL;
    # 将 galloc 结构体中的 hash_allocs 成员设置为 NULL
    galloc->hash_allocs = NULL;
// 结构体定义，包含了两个函数指针
struct ggml_allocr {
    ggml_tallocr_t talloc;
    ggml_gallocr_t galloc;
};

// 创建 ggml_allocr_t 结构体实例的静态函数
static ggml_allocr_t ggml_allocr_new_impl(ggml_tallocr_t talloc) {
    // 分配内存空间用于存储 ggml_allocr_t 结构体实例
    ggml_allocr_t alloc = (ggml_allocr_t)malloc(sizeof(struct ggml_allocr));
    // 初始化 ggml_allocr_t 结构体实例
    *alloc = (struct ggml_allocr) {
        /*.talloc = */ talloc,  // 初始化 talloc 成员
        /*.galloc = */ ggml_gallocr_new(),  // 初始化 galloc 成员
    };
    return alloc;  // 返回初始化后的 ggml_allocr_t 结构体实例
}

// 使用 ggml_tallocr_t 实例创建 ggml_allocr_t 实例的函数
ggml_allocr_t ggml_allocr_new(void * data, size_t size, size_t alignment) {
    return ggml_allocr_new_impl(ggml_tallocr_new(data, size, alignment));  // 调用 ggml_allocr_new_impl 函数
}

// 使用 ggml_tallocr_t 实例创建 ggml_allocr_t 实例的函数
ggml_allocr_t ggml_allocr_new_measure(size_t alignment) {
    return ggml_allocr_new_impl(ggml_tallocr_new_measure(alignment));  // 调用 ggml_allocr_new_impl 函数
}

// 使用 ggml_tallocr_t 实例创建 ggml_allocr_t 实例的函数
ggml_allocr_t ggml_allocr_new_from_buffer(struct ggml_backend_buffer * buffer) {
    return ggml_allocr_new_impl(ggml_tallocr_new_from_buffer(buffer));  // 调用 ggml_allocr_new_impl 函数
}

// 使用 ggml_tallocr_t 实例创建 ggml_allocr_t 实例的函数
ggml_allocr_t ggml_allocr_new_from_backend(struct ggml_backend * backend, size_t size) {
    return ggml_allocr_new_impl(ggml_tallocr_new_from_backend(backend, size));  // 调用 ggml_allocr_new_impl 函数
}

// 使用 ggml_tallocr_t 实例创建 ggml_allocr_t 实例的函数
ggml_allocr_t ggml_allocr_new_measure_from_backend(struct ggml_backend * backend) {
    return ggml_allocr_new_impl(ggml_tallocr_new_measure_from_backend(backend));  // 调用 ggml_allocr_new_impl 函数
}

// 获取 ggml_allocr_t 实例中的 ggml_backend_buffer 结构体指针
struct ggml_backend_buffer * ggml_allocr_get_buffer(ggml_allocr_t alloc) {
    return ggml_tallocr_get_buffer(alloc->talloc);  // 返回 ggml_tallocr_t 实例中的 ggml_backend_buffer 结构体指针
}

// 设置 ggml_allocr_t 实例中的 ggml_gallocr_t 实例的解析顺序
void ggml_allocr_set_parse_seq(ggml_allocr_t alloc, const int * list, int n) {
    ggml_gallocr_set_parse_seq(alloc->galloc, list, n);  // 调用 ggml_gallocr_set_parse_seq 函数
}

// 释放 ggml_allocr_t 实例及其内部资源
void ggml_allocr_free(ggml_allocr_t alloc) {
    if (alloc == NULL) {
        return;
    }

    ggml_gallocr_free(alloc->galloc);  // 释放 ggml_gallocr_t 实例
    ggml_tallocr_free(alloc->talloc);  // 释放 ggml_tallocr_t 实例
    free(alloc);  // 释放 ggml_allocr_t 实例
}

// 判断 ggml_allocr_t 实例中的 ggml_tallocr_t 实例是否为 measure 类型
bool ggml_allocr_is_measure(ggml_allocr_t alloc) {
    return ggml_tallocr_is_measure(alloc->talloc);  // 调用 ggml_tallocr_is_measure 函数
}

// 重置 ggml_allocr_t 实例中的 ggml_tallocr_t 实例
void ggml_allocr_reset(ggml_allocr_t alloc) {
    ggml_tallocr_reset(alloc->talloc);  // 调用 ggml_tallocr_reset 函数
}

// 为 ggml_allocr_t 实例中的 ggml_tallocr_t 实例分配内存
void ggml_allocr_alloc(ggml_allocr_t alloc, struct ggml_tensor * tensor) {
    ggml_tallocr_alloc(alloc->talloc, tensor);  // 调用 ggml_tallocr_alloc 函数
}

// 获取 ggml_allocr_t 实例中的 ggml_tallocr_t 实例的最大大小
size_t ggml_allocr_max_size(ggml_allocr_t alloc) {
    # 调用 ggml_tallocr_max_size 函数，传入 alloc->talloc 参数，并返回结果
    return ggml_tallocr_max_size(alloc->talloc);
// 为给定的分配器和计算图分配内存
size_t ggml_allocr_alloc_graph(ggml_allocr_t alloc, struct ggml_cgraph * graph) {
    // 调用 ggml_gallocr_alloc_graph 函数为给定的分配器和计算图分配内存
    return ggml_gallocr_alloc_graph(alloc->galloc, alloc->talloc, graph);
}

// 实用函数

// 为一系列张量分配内存范围
static bool alloc_tensor_range(struct ggml_context * ctx,
        struct ggml_tensor * first, struct ggml_tensor * last,
        ggml_backend_buffer_type_t buft, size_t size,
        ggml_backend_buffer_t ** buffers, size_t * n_buffers) {
    // 为给定类型和大小分配缓冲区
    ggml_backend_buffer_t buffer = ggml_backend_buft_alloc_buffer(buft, size);
    // 如果分配失败
    if (buffer == NULL) {
        // 在调试模式下输出错误信息
#ifndef NDEBUG
        fprintf(stderr, "%s: failed to allocate %s buffer of size %zu\n", __func__, ggml_backend_buft_name(buft), size);
#endif
        // 释放之前分配的缓冲区
        for (size_t i = 0; i < *n_buffers; i++) {
            ggml_backend_buffer_free(*buffers[i]);
        }
        // 释放缓冲区数组
        free(*buffers);
        return false;
    }

    // 从缓冲区创建 tallocr 对象
    ggml_tallocr_t tallocr = ggml_tallocr_new_from_buffer(buffer);

    // 遍历张量范围
    for (struct ggml_tensor * t = first; t != last; t = ggml_get_next_tensor(ctx, t)) {
        // 如果张量数据为空
        if (t->data == NULL) {
            // 如果视图源为空
            if (t->view_src == NULL) {
                // 为张量分配内存
                ggml_tallocr_alloc(tallocr, t);
            } else {
                // 初始化缓冲区视图
                ggml_backend_view_init(buffer, t);
            }
        } else {
            // 如果视图源不为空
            if (t->view_src != NULL) {
                // 初始化缓冲区视图
                ggml_backend_view_init(buffer, t);
            }
        }
    }

    // 释放 tallocr 对象
    ggml_tallocr_free(tallocr);

    // 重新分配缓冲区数组，增加一个元素
    *buffers = realloc(*buffers, sizeof(ggml_backend_buffer_t) * (*n_buffers + 1));
    (*buffers)[(*n_buffers)++] = buffer;

    return true;
}

// 从给定的缓冲区类型为张量分配内存
ggml_backend_buffer_t ggml_backend_alloc_ctx_tensors_from_buft(struct ggml_context * ctx, ggml_backend_buffer_type_t buft) {
    // 确保上下文没有分配
    GGML_ASSERT(ggml_get_no_alloc(ctx) == true);

    // 获取缓冲区对齐方式和最大大小
    size_t alignment = ggml_backend_buft_get_alignment(buft);
    size_t max_size = ggml_backend_buft_get_max_size(buft);

    // 初始化缓冲区数组和数量
    ggml_backend_buffer_t * buffers = NULL;
    size_t n_buffers = 0;

    // 当前缓冲区大小
    size_t cur_buf_size = 0;
}
    // 获取上下文中的第一个张量
    struct ggml_tensor * first = ggml_get_first_tensor(ctx);
    // 遍历所有张量
    for (struct ggml_tensor * t = first; t != NULL; t = ggml_get_next_tensor(ctx, t)) {
        // 初始化当前张量的大小为0
        size_t this_size = 0;
        // 如果张量的数据和视图源都为空
        if (t->data == NULL && t->view_src == NULL) {
            // 计算当前张量需要的内存大小
            this_size = GGML_PAD(ggml_backend_buft_get_alloc_size(buft, t), alignment);
        }

        // 如果当前张量大小超过最大大小
        if (this_size > max_size) {
            // 输出错误信息并释放内存
            fprintf(stderr, "%s: tensor %s is too large to fit in a %s buffer (tensor size: %zu, max buffer size: %zu)\n",
                    __func__, t->name,
                    ggml_backend_buft_name(buft),
                    this_size, max_size);
            for (size_t i = 0; i < n_buffers; i++) {
                ggml_backend_buffer_free(buffers[i]);
            }
            free(buffers);
            return NULL;
        }

        // 如果当前缓冲区大小加上当前张量大小超过最大大小
        if ((cur_buf_size + this_size) > max_size) {
            // 在当前缓冲区中分配张量
            if (!alloc_tensor_range(ctx, first, t, buft, cur_buf_size, &buffers, &n_buffers)) {
                return NULL;
            }
            first = t;
            cur_buf_size = this_size;
        } else {
            cur_buf_size += this_size;
        }
    }

    // 分配剩余的张量
    if (cur_buf_size > 0) {
        if (!alloc_tensor_range(ctx, first, NULL, buft, cur_buf_size, &buffers, &n_buffers)) {
            return NULL;
        }
    }

    // 如果没有缓冲区被分配
    if (n_buffers == 0) {
        // 上下文中的所有张量已经被分配
#ifndef NDEBUG
        // 如果处于调试模式，输出调试信息：所有上下文中的张量已经分配
        fprintf(stderr, "%s: all tensors in the context are already allocated\n", __func__);
#endif
        // 返回空指针
        return NULL;
    }

    // 声明一个名为 buffer 的 ggml_backend_buffer_t 类型变量
    ggml_backend_buffer_t buffer;
    // 如果缓冲区数量为 1
    if (n_buffers == 1) {
        // 将第一个缓冲区赋值给 buffer
        buffer = buffers[0];
    } else {
        // 否则调用 ggml_backend_multi_buffer_alloc_buffer 函数分配多个缓冲区
        buffer = ggml_backend_multi_buffer_alloc_buffer(buffers, n_buffers);
    }
    // 释放缓冲区数组的内存
    free(buffers);
    // 返回 buffer
    return buffer;
}

// 从默认缓冲区类型中为上下文张量分配缓冲区
ggml_backend_buffer_t ggml_backend_alloc_ctx_tensors(struct ggml_context * ctx, ggml_backend_t backend) {
    // 调用 ggml_backend_get_default_buffer_type 函数获取默认缓冲区类型，并为上下文张量分配缓冲区
    return ggml_backend_alloc_ctx_tensors_from_buft(ctx, ggml_backend_get_default_buffer_type(backend));
}
```