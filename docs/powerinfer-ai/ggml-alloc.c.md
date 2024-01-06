# `PowerInfer\ggml-alloc.c`

```
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

#define MAX(a, b) ((a) > (b) ? (a) : (b)  // 定义一个宏，返回两个数中较大的那个

#define MAX_FREE_BLOCKS 256  // 定义最大空闲块数

//#define GGML_ALLOCATOR_DEBUG  // 定义 GGML_ALLOCATOR_DEBUG 宏

//#define AT_PRINTF(...) fprintf(stderr, __VA_ARGS__)  // 定义 AT_PRINTF 宏，用于输出调试信息到标准错误流
#define AT_PRINTF(...)  // 空的 AT_PRINTF 宏，不做任何操作

// TODO: GGML_PAD ?  // 待完成：GGML_PAD 是什么？
// 计算给定偏移量和对齐方式下的对齐偏移量
static size_t aligned_offset(const void * buffer, size_t offset, size_t alignment) {
    // 断言对齐方式为非零且为2的幂次方
    assert(alignment && !(alignment & (alignment - 1))); // power of 2
    // 计算对齐偏移量
    size_t align = (alignment - (((uintptr_t)buffer + offset) % alignment)) % alignment;
    return offset + align;
}

// 定义空闲块的结构体
struct free_block {
    void * addr; // 空闲块的地址
    size_t size; // 空闲块的大小
};

// 定义内存分配器的结构体
struct ggml_tallocr {
    struct ggml_backend_buffer * buffer; // 内存分配器的缓冲区
    bool buffer_owned; // 缓冲区是否由内存分配器拥有
    void * base; // 内存分配器的基地址
    size_t alignment; // 内存分配器的对齐方式

    int n_free_blocks; // 空闲块的数量
    struct free_block free_blocks[MAX_FREE_BLOCKS]; // 空闲块的数组
    // 定义一个变量 max_size，用于存储最大尺寸
    size_t max_size;

    // 定义一个变量 measure，用于表示是否进行测量
    bool measure;

    // 在调试模式下，定义一个数组 allocated_tensors，用于存储分配的张量指针
#ifdef GGML_ALLOCATOR_DEBUG
    struct ggml_tensor * allocated_tensors[1024];
#endif
};

// 在调试模式下，定义一个函数 add_allocated_tensor，用于向 allocated_tensors 数组中添加分配的张量指针
static void add_allocated_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    // 遍历 allocated_tensors 数组
    for (int i = 0; i < 1024; i++) {
        // 如果数组中有空位，将张量指针添加进去并返回
        if (alloc->allocated_tensors[i] == NULL) {
            alloc->allocated_tensors[i] = tensor;
            return;
        }
    }
    // 如果数组已满，抛出错误信息
    GGML_ASSERT(!"out of allocated_tensors");
}

// 在调试模式下，定义一个函数 remove_allocated_tensor，用于从 allocated_tensors 数组中移除指定的张量指针
static void remove_allocated_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
// 循环遍历alloc->allocated_tensors数组，查找是否存在与给定tensor相同的指针或者数据指针相同的元素
for (int i = 0; i < 1024; i++) {
    if (alloc->allocated_tensors[i] == tensor ||
        (alloc->allocated_tensors[i] != NULL && alloc->allocated_tensors[i]->data == tensor->data)) {
        // 如果找到相同的tensor或者数据指针相同的元素，将其置为NULL并返回
        alloc->allocated_tensors[i] = NULL;
        return;
    }
}
// 如果未找到相同的tensor或者数据指针相同的元素，打印错误信息并终止程序
printf("tried to free tensor %s not found\n", tensor->name);
GGML_ASSERT(!"tensor not found");
}

// 检查一个tensor是否由该缓冲区分配
static bool ggml_tallocr_is_own(ggml_tallocr_t alloc, const struct ggml_tensor * tensor) {
    return tensor->buffer == alloc->buffer;
}

// 检查一个tensor是否是视图
static bool ggml_is_view(struct ggml_tensor * t) {
    return t->view_src != NULL;
}
// 分配内存给张量
void ggml_tallocr_alloc(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    // 断言张量不是视图，通常视图会从其源中获取数据指针
    GGML_ASSERT(!ggml_is_view(tensor));
    // 断言张量的数据指针为空，避免为已经分配内存的张量再次分配内存
    GGML_ASSERT(tensor->data == NULL);

    // 获取需要分配的内存大小
    size_t size = ggml_backend_buffer_get_alloc_size(alloc->buffer, tensor);
    // 对齐内存大小
    size = aligned_offset(NULL, size, alloc->alignment);

    // 打印分配信息
    AT_PRINTF("%s: allocating %s (%zu bytes) - ", __func__, tensor->name, size);

    size_t max_avail = 0;

    // 寻找除最后一个块之外最适合的空闲块
    int best_fit_block = -1;
    size_t best_fit_size = SIZE_MAX;
    for (int i = 0; i < alloc->n_free_blocks - 1; i++) {
        struct free_block * block = &alloc->free_blocks[i];
        max_avail = MAX(max_avail, block->size);
        if (block->size >= size && block->size <= best_fit_size) {
            best_fit_block = i;
    // 用于记录最佳匹配的块的大小
    best_fit_size = block->size;
    // 遍历所有空闲块，找到最佳匹配的块
    for (int i = 0; i < alloc->n_free_blocks; i++) {
        // 如果当前块的大小恰好等于需要的大小，则直接选择该块
        if (alloc->free_blocks[i].size == size) {
            best_fit_block = i;
            break;
        }
        // 如果当前块的大小大于需要的大小，并且比之前记录的最佳匹配块的大小更小，则更新最佳匹配块的索引和大小
        else if (alloc->free_blocks[i].size > size && alloc->free_blocks[i].size < best_fit_size) {
            best_fit_block = i;
            best_fit_size = alloc->free_blocks[i].size;
        }
    }

    // 打印最佳匹配块的索引
    AT_PRINTF("block %d\n", best_fit_block);

    // 如果没有找到最佳匹配块
    if (best_fit_block == -1) {
        // 最后一个块作为最后的选择
        struct free_block * block = &alloc->free_blocks[alloc->n_free_blocks - 1];
        // 更新最大可用空间
        max_avail = MAX(max_avail, block->size);
        // 如果最后一个块的大小足够容纳需要的大小，则选择该块
        if (block->size >= size) {
            best_fit_block = alloc->n_free_blocks - 1;
        } else {
            // 如果最后一个块的大小也不够，则打印错误信息并终止程序
            fprintf(stderr, "%s: not enough space in the buffer (needed %zu, largest block available %zu)\n",
                    __func__, size, max_avail);
            GGML_ASSERT(!"not enough space in the buffer");
            return;
        }
    }
    // 获取最佳匹配块的指针
    struct free_block * block = &alloc->free_blocks[best_fit_block];
    // 将指针 addr 指向的内存地址赋值给 block->addr
    void * addr = block->addr;
    // 将 block->addr 指向的内存地址向后偏移 size 个字节
    block->addr = (char*)block->addr + size;
    // 减去已分配的内存大小
    block->size -= size;
    // 如果块的大小为0，则移除该块
    if (block->size == 0) {
        // 减少空闲块的数量
        alloc->n_free_blocks--;
        // 从最佳匹配块开始，将后面的空闲块向前移动一个位置
        for (int j = best_fit_block; j < alloc->n_free_blocks; j++) {
            alloc->free_blocks[j] = alloc->free_blocks[j+1];
        }
    }

    // 将 addr 指向的内存地址赋值给 tensor->data
    tensor->data = addr;
    // 将 alloc->buffer 赋值给 tensor->buffer
    tensor->buffer = alloc->buffer;
    // 如果不需要测量内存分配性能
    if (!alloc->measure) {
        // 初始化张量的缓冲区
        ggml_backend_buffer_init_tensor(alloc->buffer, tensor);
    }

    // 如果定义了 GGML_ALLOCATOR_DEBUG
    #ifdef GGML_ALLOCATOR_DEBUG
    // 将分配的张量添加到已分配的张量列表中
    add_allocated_tensor(alloc, tensor);
    // 计算当前最大的内存地址
    size_t cur_max = (char*)addr - (char*)alloc->data + size;
    // 如果当前最大值超过了分配器的最大大小
    if (cur_max > alloc->max_size) {
        // 打印最大大小，并遍历已分配的张量
        printf("max_size = %.2f MB: tensors: ", cur_max / 1024.0 / 1024.0);
        for (int i = 0; i < 1024; i++) {
            // 如果张量已分配，则打印张量名和大小
            if (alloc->allocated_tensors[i]) {
                printf("%s (%.2f MB) ", alloc->allocated_tensors[i]->name, ggml_nbytes(alloc->allocated_tensors[i]) / 1024.0 / 1024.0);
            }
        }
        // 打印换行符
        printf("\n");
    }
#endif

    // 更新分配器的最大大小
    alloc->max_size = MAX(alloc->max_size, (char*)addr - (char*)alloc->base + size);
}

// 这是一个非常简单的实现，但对于我们的情况，空闲块的数量应该非常小
static void ggml_tallocr_free_tensor(ggml_tallocr_t alloc, struct ggml_tensor * tensor) {
    // 如果张量不是在这个缓冲区中分配的
    if (ggml_tallocr_is_own(alloc, tensor) == false) {
        // 张量不是在这个缓冲区中分配的情况可能发生，因为图形分配器会尝试从不同的缓冲区释放权重和其他张量
        // 处理这种情况最简单的方法就是忽略它
    // 如果 tensor 的 buffer 指针和 alloc 的 buffer 指针不一样，就忽略这个 tensor
    // 返回空值
    return;
    // 获取 tensor 的数据指针
    void * ptr = tensor->data;
    // 获取 tensor 占用的内存大小
    size_t size = ggml_backend_buffer_get_alloc_size(alloc->buffer, tensor);
    // 对齐内存大小
    size = aligned_offset(NULL, size, alloc->alignment);
    // 打印释放内存的信息
    AT_PRINTF("%s: freeing %s at %p (%zu bytes) - n_free_blocks = %d\n", __func__, tensor->name, ptr, size, alloc->n_free_blocks);
    // 如果不是测量模式，释放 tensor 占用的内存
    if (!alloc->measure) {
        ggml_backend_buffer_free_tensor(alloc->buffer, tensor);
    }
    // 在调试模式下，移除已分配的 tensor
#ifdef GGML_ALLOCATOR_DEBUG
    remove_allocated_tensor(alloc, tensor);
#endif
    // 查看是否可以与现有的块合并
    for (int i = 0; i < alloc->n_free_blocks; i++) {
        // 获取指向第 i 个空闲块的指针
        struct free_block * block = &alloc->free_blocks[i];
        // 检查指针是否在块的末尾
        if ((char*)block->addr + block->size == ptr) {
            block->size += size;
            // 检查是否可以与下一个块合并
            if (i < alloc->n_free_blocks - 1 && (char*)block->addr + block->size == alloc->free_blocks[i+1].addr) {
                block->size += alloc->free_blocks[i+1].size;
                alloc->n_free_blocks--;
                for (int j = i+1; j < alloc->n_free_blocks; j++) {
                    alloc->free_blocks[j] = alloc->free_blocks[j+1];
                }
            }
            return;
        }
        // 检查指针是否在块的开头
        if ((char*)ptr + size == block->addr) {
            block->addr = ptr;
            block->size += size;
            // 检查是否可以与前一个块合并
            if (i > 0 && (char*)alloc->free_blocks[i-1].addr + alloc->free_blocks[i-1].size == block->addr) {
                // 如果找到了可以合并的空闲块
                alloc->free_blocks[i-1].size += block->size;
                // 减少空闲块数量
                alloc->n_free_blocks--;
                // 将后面的空闲块向前移动，填补被合并的空闲块
                for (int j = i; j < alloc->n_free_blocks; j++) {
                    alloc->free_blocks[j] = alloc->free_blocks[j+1];
                }
            }
            // 返回，结束函数
            return;
        }
    }
    // 否则，添加一个新的空闲块
    GGML_ASSERT(alloc->n_free_blocks < MAX_FREE_BLOCKS && "out of free blocks");
    // 将新的空闲块插入到正确的位置，以保持数组按地址排序（以加快合并块的速度）
    int insert_pos = 0;
    while (insert_pos < alloc->n_free_blocks && alloc->free_blocks[insert_pos].addr < ptr) {
        insert_pos++;
    }
    // 将从insert_pos开始的所有块向后移动，为新块腾出位置
    for (int i = alloc->n_free_blocks; i > insert_pos; i--) {
        alloc->free_blocks[i] = alloc->free_blocks[i-1];
    }
// 插入新的块
alloc->free_blocks[insert_pos].addr = ptr; // 将指针地址插入到空闲块数组的指定位置
alloc->free_blocks[insert_pos].size = size; // 将块的大小插入到空闲块数组的指定位置
alloc->n_free_blocks++; // 空闲块数量加一
}

void ggml_tallocr_reset(ggml_tallocr_t alloc) {
    alloc->n_free_blocks = 1; // 重置空闲块数量为1
    size_t align_offset = aligned_offset(alloc->base, 0, alloc->alignment); // 计算对齐偏移量
    alloc->free_blocks[0].addr = (char *)alloc->base + align_offset; // 设置第一个空闲块的地址

    if (alloc->measure) {
        alloc->free_blocks[0].size = SIZE_MAX/2; // 如果是测量模式，限制测量分配器的最大大小为 size_t 最大值的一半，以避免溢出
    } else {
        alloc->free_blocks[0].size = ggml_backend_buffer_get_size(alloc->buffer) - align_offset; // 否则，设置第一个空闲块的大小为缓冲区大小减去对齐偏移量
    }
}

ggml_tallocr_t ggml_tallocr_new(void * data, size_t size, size_t alignment) {
    struct ggml_backend_buffer * buffer = ggml_backend_cpu_buffer_from_ptr(NULL, data, size); // 从给定的数据和大小创建一个 CPU 缓冲区
# 分配内存空间，大小为 ggml_tallocr 结构体的大小
ggml_tallocr_t alloc = (ggml_tallocr_t)malloc(sizeof(struct ggml_tallocr));

# 初始化分配的内存空间，设置结构体的各个字段的值
*alloc = (struct ggml_tallocr) {
    # 设置 buffer 字段的值为 buffer
    /*.buffer        = */ buffer,
    # 设置 buffer_owned 字段的值为 true
    /*.buffer_owned  = */ true,
    # 设置 base 字段的值为 ggml_backend_buffer_get_base(buffer) 的返回值
    /*.base          = */ ggml_backend_buffer_get_base(buffer),
    # 设置 alignment 字段的值为 alignment
    /*.alignment     = */ alignment,
    # 设置 n_free_blocks 字段的值为 0
    /*.n_free_blocks = */ 0,
    # 设置 free_blocks 字段的值为 {{0}}
    /*.free_blocks   = */ {{0}},
    # 设置 max_size 字段的值为 0
    /*.max_size      = */ 0,
    # 设置 measure 字段的值为 false
    /*.measure       = */ false,
    # 如果定义了 GGML_ALLOCATOR_DEBUG 宏，则设置 allocated_tensors 字段的值为 {0}
    #ifdef GGML_ALLOCATOR_DEBUG
    /*.allocated_tensors = */ {0},
    #endif
};

# 重置分配的内存空间
ggml_tallocr_reset(alloc);

# 返回分配的内存空间
return alloc;
// 创建一个新的测量分配器，指定对齐方式
ggml_tallocr_t ggml_tallocr_new_measure(size_t alignment) {
    // 使用指定的地址和大小创建一个新的分配器
    ggml_tallocr_t alloc = ggml_tallocr_new((void *)0x1000, SIZE_MAX/2, alignment);
    // 设置分配器为测量模式
    alloc->measure = true;

    return alloc;
}

// 从后端创建一个新的测量分配器
ggml_tallocr_t ggml_tallocr_new_measure_from_backend(struct ggml_backend * backend) {
    // 创建一个后端缓冲区以获取正确的张量分配大小
    ggml_backend_buffer_t buffer = ggml_backend_alloc_buffer(backend, 1);

    // TODO: 将分配器初始化移动到一个通用的ggml_tallocr_new_impl函数
    // 使用后端缓冲区创建一个新的分配器
    ggml_tallocr_t alloc = ggml_tallocr_new_from_buffer(buffer);
    // 设置分配器拥有缓冲区
    alloc->buffer_owned = true;
    // 设置分配器为测量模式
    alloc->measure = true;
    // 重置分配器
    ggml_tallocr_reset(alloc);
    return alloc;
}
# 从后端创建一个新的 tallocr 对象，分配指定大小的内存
ggml_tallocr_t ggml_tallocr_new_from_backend(struct ggml_backend * backend, size_t size) {
    # 从后端分配指定大小的缓冲区
    ggml_backend_buffer_t buffer = ggml_backend_alloc_buffer(backend, size);
    # 从缓冲区创建一个新的 tallocr 对象
    ggml_tallocr_t alloc = ggml_tallocr_new_from_buffer(buffer);
    # 设置 tallocr 对象的 buffer_owned 属性为 true
    alloc->buffer_owned = true;
    # 返回 tallocr 对象
    return alloc;
}

# 从缓冲区创建一个新的 tallocr 对象
ggml_tallocr_t ggml_tallocr_new_from_buffer(struct ggml_backend_buffer * buffer) {
    # 分配内存以存储 tallocr 对象
    ggml_tallocr_t alloc = (ggml_tallocr_t)malloc(sizeof(struct ggml_tallocr));

    # 初始化 tallocr 对象的属性
    *alloc = (struct ggml_tallocr) {
        # 设置 buffer 属性为传入的 buffer
        /*.buffer        = */ buffer,
        # 设置 buffer_owned 属性为 false
        /*.buffer_owned  = */ false,
        # 设置 base 属性为 buffer 的基地址
        /*.base          = */ ggml_backend_buffer_get_base(buffer),
        # 设置 alignment 属性为 buffer 的对齐方式
        /*.alignment     = */ ggml_backend_buffer_get_alignment(buffer),
        # 设置 n_free_blocks 属性为 0
        /*.n_free_blocks = */ 0,
        # 设置 free_blocks 属性为全 0 数组
        /*.free_blocks   = */ {{0}},
        # 设置 max_size 属性为 0
        /*.max_size      = */ 0,
        # 设置 measure 属性为 false
        /*.measure       = */ false,
#ifdef GGML_ALLOCATOR_DEBUG
        /*.allocated_tensors = */ {0},  // 如果定义了 GGML_ALLOCATOR_DEBUG，则初始化 allocated_tensors 为 0
#endif
    };

    ggml_tallocr_reset(alloc);  // 重置分配器状态

    return alloc;  // 返回分配器

}

struct ggml_backend_buffer * ggml_tallocr_get_buffer(ggml_tallocr_t alloc) {
    return alloc->buffer;  // 返回分配器中的缓冲区
}

void ggml_tallocr_free(ggml_tallocr_t alloc) {
    if (alloc == NULL) {  // 如果分配器为空，则直接返回
        return;
    }

    if (alloc->buffer_owned) {  // 如果分配器拥有缓冲区
// 释放分配器的缓冲区
ggml_backend_buffer_free(alloc->buffer);
}

// 释放分配器
free(alloc);
}

// 检查分配器是否为测量
bool ggml_tallocr_is_measure(ggml_tallocr_t alloc) {
    return alloc->measure;
}

// 获取分配器的最大大小
size_t ggml_tallocr_max_size(ggml_tallocr_t alloc) {
    return alloc->max_size;
}

// 图形分配器

// 哈希节点结构
struct hash_node {
    int n_children; // 子节点数量
    int n_views; // 视图数量
};
# 定义结构体 ggml_gallocr，包含了一些成员变量
struct ggml_gallocr {
    ggml_tallocr_t talloc;  // 用于分配内存的对象
    struct ggml_hash_set hash_set;  // 哈希集合
    struct hash_node * hash_values;  // 哈希值
    size_t hash_values_size;  // 哈希值的大小
    ggml_tallocr_t * hash_allocs;  // 用于分配哈希值内存的对象
    int * parse_seq;  // 解析序列
    int parse_seq_len;  // 解析序列的长度
};

# 定义函数 ggml_gallocr_new，用于创建 ggml_gallocr_t 类型的对象
ggml_gallocr_t ggml_gallocr_new(void) {
    # 为 ggml_gallocr_t 类型的对象分配内存
    ggml_gallocr_t galloc = (ggml_gallocr_t)malloc(sizeof(struct ggml_gallocr));

    # 初始化分配的内存
    *galloc = (struct ggml_gallocr) {
        /*.talloc           = */ NULL,  // talloc 初始化为 NULL
        /*.hash_set         = */ {0},  // hash_set 初始化为 0
        /*.hash_values      = */ NULL,  // hash_values 初始化为 NULL
        /*.hash_values_size = */ 0,  // hash_values_size 初始化为 0
        /*.hash_allocs      = */ NULL,  // hash_allocs 初始化为 NULL
        /*.parse_seq        = */ NULL,  // parse_seq 初始化为 NULL
    /* 分配器的序列长度设置为0 */
    .parse_seq_len = 0,
};

/* 释放分配器 */
ggml_gallocr_t ggml_gallocr_alloc() {
    /* 分配一个新的分配器对象 */
    ggml_gallocr_t galloc = malloc(sizeof(struct ggml_gallocr_s));
    if (galloc == NULL) {
        return NULL;
    }

    return galloc;
}

/* 释放分配器对象 */
void ggml_gallocr_free(ggml_gallocr_t galloc) {
    if (galloc == NULL) {
        return;
    }

    /* 如果分配器对象中的键数组不为空，则释放内存 */
    if (galloc->hash_set.keys != NULL) {
        free(galloc->hash_set.keys);
    }
    /* 如果分配器对象中的值数组不为空，则释放内存 */
    if (galloc->hash_values != NULL) {
        free(galloc->hash_values);
    }
    /* 如果分配器对象中的分配数组不为空，则释放内存 */
    if (galloc->hash_allocs != NULL) {
        free(galloc->hash_allocs);
    }
}
// 如果解析序列不为空，则释放其内存
if (galloc->parse_seq != NULL) {
    free(galloc->parse_seq);
}
// 释放 galloc 指针指向的内存
free(galloc);
}

// 设置解析序列
void ggml_gallocr_set_parse_seq(ggml_gallocr_t galloc, const int * list, int n) {
    // 释放解析序列的内存
    free(galloc->parse_seq);
    // 为解析序列分配内存
    galloc->parse_seq = malloc(sizeof(int) * n);

    // 将 list 中的数据复制到解析序列中
    for (int i = 0; i < n; i++) {
        galloc->parse_seq[i] = list[i];
    }
    // 设置解析序列的长度
    galloc->parse_seq_len = n;
}

// 获取哈希表中的节点
static struct hash_node * hash_get(ggml_gallocr_t galloc, struct ggml_tensor * t) {
    // 在哈希表中查找或插入元素，并返回索引
    size_t i = ggml_hash_find_or_insert(galloc->hash_set, t);
    // 返回哈希值对应的节点
    return &galloc->hash_values[i];
}
// 检查两个张量是否具有相同的布局
static bool ggml_are_same_layout(const struct ggml_tensor * a, const struct ggml_tensor * b) {
    // 如果张量类型不同，则返回false
    if (a->type != b->type) {
        return false;
    }
    // 遍历张量的维度
    for (int i = 0; i < GGML_MAX_DIMS; i++) {
        // 如果张量的元素数量不同，则返回false
        if (a->ne[i] != b->ne[i]) {
            return false;
        }
        // 如果张量的边界数量不同，则返回false
        if (a->nb[i] != b->nb[i]) {
            return false;
        }
    }
    // 如果以上条件都满足，则返回true
    return true;
}

// 检查操作是否可以原地执行
static bool ggml_op_can_inplace(enum ggml_op op) {
    // 根据操作类型进行判断
    switch (op) {
        // 如果是缩放操作或者对角线掩码零操作，则返回true
        case GGML_OP_SCALE:
        case GGML_OP_DIAG_MASK_ZERO:
# 对于给定的操作类型，如果是以下列出的操作之一，则返回 true
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

# 如果操作类型不在以上列出的操作中，则返回 false
default:
    return false;
# 定义一个函数，用于从给定的全局分配器中获取节点的分配器
static ggml_tallocr_t node_tallocr(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    # 如果全局分配器中已经存在节点的分配器，则直接返回
    if (galloc->talloc != NULL) {
        return galloc->talloc;
    }
    # 否则，根据节点在哈希表中的索引找到或插入节点的分配器
    return galloc->hash_allocs[ggml_hash_find_or_insert(galloc->hash_set, node)];
}

# 定义一个函数，用于初始化视图
static void init_view(ggml_gallocr_t galloc, struct ggml_tensor * view, bool update_backend) {
    # 获取视图的分配器
    ggml_tallocr_t alloc = node_tallocr(galloc, view);

    # 打印初始化视图的信息
    //printf("init_view: %s from src %s\n", view->name, view->view_src->name);
    # 断言视图的源不为空且数据不为空
    GGML_ASSERT(view->view_src != NULL && view->view_src->data != NULL);
    # 如果需要更新后端，则将视图的后端设置为源视图的后端
    if (update_backend) {
        view->backend = view->view_src->backend;
    }
    # 设置视图的缓冲区和数据偏移
    view->buffer  = view->view_src->buffer;
    view->data    = (char *)view->view_src->data + view->view_offs;

    # FIXME: 视图应该由拥有的缓冲区初始化，但目前这会破坏 CUDA 后端
    # 有待修复的问题：视图应该由拥有的缓冲区初始化，但目前这会破坏 CUDA 后端
// 检查 ggml_tensor_extra_gpu 环形缓冲是否会覆盖 KV 缓存额外数据
assert(ggml_tallocr_is_measure(alloc) || !view->buffer || view->buffer->backend == alloc->buffer->backend);

// 如果分配器不是 measure 类型，则初始化 tensor 的后端缓冲区
if (!alloc->measure) {
    ggml_backend_buffer_init_tensor(alloc->buffer, view);
}
}

// 分配节点内存
static void allocate_node(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    // 获取节点的分配器
    ggml_tallocr_t alloc = node_tallocr(galloc, node);

    // 如果节点的数据为空
    if (node->data == NULL) {
        // 如果节点是视图类型
        if (ggml_is_view(node)) {
            // 初始化视图
            init_view(galloc, node, true);
        } else {
            // 查看是否可以重用父节点的缓冲区（原地操作）
            if (ggml_op_can_inplace(node->op)) {
                // 遍历节点的源节点
                for (int i = 0; i < GGML_MAX_SRC; i++) {
                    struct ggml_tensor * parent = node->src[i];
                    // 如果父节点为空
                    if (parent == NULL) {
                    // 如果节点的数据是外部的，那么我们无法重用它
                    if (ggml_tallocr_is_own(alloc, parent) == false) {
                        AT_PRINTF("not reusing parent %s for %s as %p is external\n", parent->name, node->name, parent->data);
                        continue;
                    }

                    // 获取父节点的哈希表节点
                    struct hash_node * p_hn = hash_get(galloc, parent);
                    // 如果父节点的数据不为空，并且子节点数为1，视图数为0，并且子节点和父节点具有相同的布局
                    if (parent->data != NULL && p_hn->n_children == 1 && p_hn->n_views == 0 && ggml_are_same_layout(node, parent)) {
                        // 如果父节点是视图
                        if (ggml_is_view(parent)) {
                            // 获取视图的源张量和哈希表节点
                            struct ggml_tensor * view_src = parent->view_src;
                            struct hash_node * view_src_hn = hash_get(galloc, view_src);
                            // 如果视图的视图数为1，子节点数为0，并且视图的数据等于父节点的数据
                            if (view_src_hn->n_views == 1 && view_src_hn->n_children == 0 && view_src->data == parent->data) {
                                // TODO: 视图父节点的偏移量必须保留，以确保操作不会覆盖它稍后将需要的父节点数据（相同的布局要求）。
                                // 问题在于我们无法释放张量，因为分配的原始地址丢失了。
                                // 将视图源指针添加到张量中将解决此问题，并简化处理视图的代码
                                // 目前，我们只在偏移量为零时（view_src->data == parent->data）重用父节点的数据
# 打印日志，表示正在重用视图的父节点
AT_PRINTF("reusing view parent %s (%s) for %s\n", parent->name, view_src->name, node->name);
# 将视图源设置为视图源节点
node->view_src = view_src;
# 视图源节点的视图数量加一
view_src_hn->n_views += 1;
# 初始化视图
init_view(galloc, node, false);
# 返回
return;

# 如果视图源节点不存在
} else {
    # 打印日志，表示正在重用父节点
    AT_PRINTF("reusing parent %s for %s\n", parent->name, node->name);
    # 将视图源设置为父节点
    node->view_src = parent;
    # 父节点的视图数量加一
    p_hn->n_views += 1;
    # 初始化视图
    init_view(galloc, node, false);
    # 返回
    return;
# 结束 if 语句
}

# 结束函数
// 释放节点内存
static void free_node(ggml_gallocr_t galloc, struct ggml_tensor * node) {
    // 为节点分配内存
    ggml_tallocr_t alloc = node_tallocr(galloc, node);
    // 释放节点内存
    ggml_tallocr_free_tensor(alloc, node);
}

// 分配图形内存
static void ggml_tallocr_alloc_graph_impl(ggml_gallocr_t galloc, struct ggml_cgraph * gf) {
    // 获取解析序列和长度
    const int * parse_seq     = galloc->parse_seq;
    int         parse_seq_len = galloc->parse_seq_len;

    // 计算子节点和视图的数量
    for (int i = 0; i < gf->n_nodes; i++) {
        // 获取图形节点
        struct ggml_tensor * node = gf->nodes[i];

        // 如果节点是视图
        if (ggml_is_view(node)) {
            // 获取视图源节点
            struct ggml_tensor * view_src = node->view_src;
            // 增加视图源节点的视图数量
            hash_get(galloc, view_src)->n_views += 1;
            // 如果节点的缓冲区为空且数据不为空
            if (node->buffer == NULL && node->data != NULL) {
                // 预分配张量的视图，尚未调用init_view()
    // 初始化视图，为节点分配内存
    init_view(galloc, node, true);
    // 遍历节点的输入
    for (int j = 0; j < GGML_MAX_SRC; j++) {
        // 获取父节点
        struct ggml_tensor * parent = node->src[j];
        // 如果父节点为空，跳出循环
        if (parent == NULL) {
            break;
        }
        // 增加父节点的子节点数量
        hash_get(galloc, parent)->n_children += 1;
        // 如果父节点是视图且缓冲区为空且数据不为空，则初始化视图
        if (ggml_is_view(parent) && parent->buffer == NULL && parent->data != NULL) {
            init_view(galloc, parent, true);
        }
    }

    // 分配张量
    // 如果有解析序列，则按照列表分配节点，只在屏障处释放节点
    int last_barrier_pos = 0;
    // 计算节点数量
    int n_nodes = parse_seq_len ? parse_seq_len : gf->n_nodes;
    // 遍历节点列表，从0到n_nodes-1
    for (int ind = 0; ind < n_nodes; ind++) {
        // 如果parse_seq_len为0或者parse_seq[ind]不等于-1，则分配一个节点
        if (parse_seq_len == 0 || parse_seq[ind] != -1) {
            // 如果parse_seq_len为0，则i为ind，否则i为parse_seq[ind]
            int i = parse_seq_len ? parse_seq[ind] : ind;
            // 获取节点指针
            struct ggml_tensor * node = gf->nodes[i];

            // 分配父节点（叶子节点）
            for (int j = 0; j < GGML_MAX_SRC; j++) {
                // 获取父节点指针
                struct ggml_tensor * parent = node->src[j];
                // 如果父节点为空，则跳出循环
                if (parent == NULL) {
                    break;
                }
                // 分配节点
                allocate_node(galloc, parent);
            }

            // 分配节点
            allocate_node(galloc, node);

            // 打印节点操作和名称
            AT_PRINTF("exec: %s (%s) <= ", ggml_op_name(node->op), node->name);
        }
    }
// 遍历节点的输入源
for (int j = 0; j < GGML_MAX_SRC; j++) {
    // 获取当前输入源的指针
    struct ggml_tensor * parent = node->src[j];
    // 如果当前输入源为空，跳出循环
    if (parent == NULL) {
        break;
    }
    // 打印当前输入源的名称
    AT_PRINTF("%s", parent->name);
    // 如果不是最后一个输入源且下一个输入源不为空，打印逗号
    if (j < GGML_MAX_SRC - 1 && node->src[j + 1] != NULL) {
        AT_PRINTF(", ");
    }
}
// 打印换行符
AT_PRINTF("\n");
}

// 更新父节点
// 如果没有解析序列，立即更新
// 如果有解析序列，只在屏障处更新
if ((parse_seq_len == 0) || parse_seq[ind] == -1) {
    // 计算更新起始位置和结束位置
    int update_start = parse_seq_len ? last_barrier_pos : ind;
    int update_end   = parse_seq_len ? ind              : ind + 1;
    // 遍历需要更新的位置
    for (int i = update_start; i < update_end; i++) {
                # 如果 parse_seq_len 为真，则使用 parse_seq[i] 作为节点索引，否则使用 i 作为节点索引
                int node_i = parse_seq_len ? parse_seq[i] : i;
                # 获取节点索引对应的节点指针
                struct ggml_tensor * node = gf->nodes[node_i];

                # 遍历节点的源节点数组
                for (int j = 0; j < GGML_MAX_SRC; j++) {
                    # 获取当前源节点指针
                    struct ggml_tensor * parent = node->src[j];
                    # 如果当前源节点为空，则跳出循环
                    if (parent == NULL) {
                        break;
                    }
                    # 获取当前源节点在哈希表中的节点指针
                    struct hash_node * p_hn = hash_get(galloc, parent);
                    # 当前源节点的子节点数减一
                    p_hn->n_children -= 1;

                    # 打印当前源节点的信息
                    # AT_PRINTF("parent %s: %d children, %d views\n", parent->name, parent->n_children, parent->n_views);

                    # 如果当前源节点的子节点数和视图数都为零
                    if (p_hn->n_children == 0 && p_hn->n_views == 0) {
                        # 如果当前源节点是视图节点
                        if (ggml_is_view(parent)) {
                            # 获取当前视图节点的源节点指针
                            struct ggml_tensor * view_src = parent->view_src;
                            # 获取当前视图节点的源节点在哈希表中的节点指针
                            struct hash_node * view_src_hn = hash_get(galloc, view_src);
                            # 当前视图节点的源节点的子节点数减一
                            view_src_hn->n_views -= 1;
                            # 打印当前视图节点的源节点信息
                            AT_PRINTF("view_src %s: %d children, %d views\n", view_src->name, view_src_hn->n_children, view_src_hn->n_views);
                            # 如果当前视图节点的源节点的子节点数和视图数都为零
                            if (view_src_hn->n_views == 0 && view_src_hn->n_children == 0) {
# 释放节点占用的内存，参数为内存分配器和视图源
free_node(galloc, view_src);
# 如果视图源不为空
if (view_src) {
    # 释放节点占用的内存，参数为内存分配器和父节点
    free_node(galloc, parent);
}
# 打印换行符
AT_PRINTF("\n");
# 如果解析序列长度不为0
if (parse_seq_len) {
    # 更新最后一个障碍位置
    last_barrier_pos = ind + 1;
}

# 分配图形占用的内存，参数为全局内存分配器、临时内存分配器和图形结构
size_t ggml_gallocr_alloc_graph(ggml_gallocr_t galloc, ggml_tallocr_t talloc, struct ggml_cgraph * graph) {
    # 获取图形结构中访问哈希表的大小
    size_t hash_size = graph->visited_hash_table.size;
// 检查哈希表是否已初始化并且大小足够
if (galloc->hash_set.size < hash_size) {
    // 如果哈希表的大小不够，释放之前分配的内存
    if (galloc->hash_set.keys != NULL) {
        free(galloc->hash_set.keys);
    }
    if (galloc->hash_values != NULL) {
        free(galloc->hash_values);
    }
    // 重新分配哈希表的大小
    galloc->hash_set.keys = malloc(sizeof(struct ggml_tensor *) * hash_size);
    galloc->hash_set.size = hash_size;
    galloc->hash_values = malloc(sizeof(struct hash_node) * hash_size);
}

// 重置哈希表
memset(galloc->hash_set.keys, 0, sizeof(struct ggml_tensor *) * hash_size);
memset(galloc->hash_values,   0, sizeof(struct hash_node) * hash_size);

// 设置 galloc 的 talloc 属性
galloc->talloc = talloc;
// 调用 ggml_tallocr_alloc_graph_impl 函数
ggml_tallocr_alloc_graph_impl(galloc, graph);
// 重置 galloc 的 talloc 属性
galloc->talloc = NULL;
// 获取当前内存分配器的最大可分配内存大小
size_t max_size = ggml_tallocr_max_size(talloc);

// 返回最大可分配内存大小
return max_size;
}

// 为图形分配内存
void ggml_gallocr_alloc_graph_n(ggml_gallocr_t galloc, struct ggml_cgraph * graph, struct ggml_hash_set hash_set, ggml_tallocr_t * hash_node_talloc) {
    // 获取哈希集合的大小
    const size_t hash_size = hash_set.size;

    // 断言哈希集合的大小大于等于图形中节点和叶子节点的数量
    GGML_ASSERT(hash_size >= (size_t)(graph->n_nodes + graph->n_leafs));

    // 初始化内存分配器为NULL
    galloc->talloc = NULL;

    // 如果哈希值为空或者哈希值大小小于哈希集合的大小，则重新分配内存
    if (galloc->hash_values == NULL || galloc->hash_values_size < hash_size) {
        // 释放之前的哈希值内存
        free(galloc->hash_values);
        // 重新分配哈希值内存
        galloc->hash_values      = malloc(sizeof(struct hash_node) * hash_size);
        // 更新哈希值大小
        galloc->hash_values_size = hash_size;
    }
// 如果需要，释放哈希集合的键
if (galloc->hash_set.keys != NULL) {
    free(galloc->hash_set.keys);
}
// 将哈希集合赋值给galloc->hash_set
galloc->hash_set = hash_set;

// 重置哈希值
memset(galloc->hash_values, 0, sizeof(struct hash_node) * hash_size);

// 将哈希节点分配给galloc->hash_allocs
galloc->hash_allocs = hash_node_talloc;

// 通过ggml_tallocr_alloc_graph_impl函数分配图形
ggml_tallocr_alloc_graph_impl(galloc, graph);

// 移除未拥有的资源
galloc->hash_set.keys = NULL;
galloc->hash_allocs = NULL;
}

// 旧版API包装器
# 定义一个结构体 ggml_allocr，包含了两个成员变量 ggml_tallocr_t 和 ggml_gallocr_t
struct ggml_allocr {
    ggml_tallocr_t talloc;
    ggml_gallocr_t galloc;
};

# 定义一个静态函数 ggml_allocr_new_impl，接受一个 ggml_tallocr_t 类型的参数 talloc
static ggml_allocr_t ggml_allocr_new_impl(ggml_tallocr_t talloc) {
    # 分配内存给 alloc，并初始化为 struct ggml_allocr 结构体
    ggml_allocr_t alloc = (ggml_allocr_t)malloc(sizeof(struct ggml_allocr));
    *alloc = (struct ggml_allocr) {
        # 初始化 talloc 成员变量
        /*.talloc = */ talloc,
        # 初始化 galloc 成员变量为 ggml_gallocr_new() 的返回值
        /*.galloc = */ ggml_gallocr_new(),
    };
    # 返回初始化后的 alloc
    return alloc;
}

# 定义一个函数 ggml_allocr_new，接受三个参数 data, size, alignment
ggml_allocr_t ggml_allocr_new(void * data, size_t size, size_t alignment) {
    # 调用 ggml_tallocr_new 函数创建 tallocr，并传入 ggml_allocr_new_impl 函数
    return ggml_allocr_new_impl(ggml_tallocr_new(data, size, alignment));
}

# 定义一个函数 ggml_allocr_new_measure，接受一个参数 alignment
ggml_allocr_t ggml_allocr_new_measure(size_t alignment) {
    # 调用 ggml_tallocr_new_measure 函数创建 tallocr，并传入 ggml_allocr_new_impl 函数
    return ggml_allocr_new_impl(ggml_tallocr_new_measure(alignment));
}
// 从给定的缓冲区创建一个新的分配器对象
ggml_allocr_t ggml_allocr_new_from_buffer(struct ggml_backend_buffer * buffer) {
    return ggml_allocr_new_impl(ggml_tallocr_new_from_buffer(buffer));
}

// 从给定的后端和大小创建一个新的分配器对象
ggml_allocr_t ggml_allocr_new_from_backend(struct ggml_backend * backend, size_t size) {
    return ggml_allocr_new_impl(ggml_tallocr_new_from_backend(backend, size));
}

// 从给定的后端创建一个新的测量分配器对象
ggml_allocr_t ggml_allocr_new_measure_from_backend(struct ggml_backend * backend) {
    return ggml_allocr_new_impl(ggml_tallocr_new_measure_from_backend(backend));
}

// 获取分配器对象的缓冲区
struct ggml_backend_buffer * ggml_allocr_get_buffer(ggml_allocr_t alloc) {
    return ggml_tallocr_get_buffer(alloc->talloc);
}

// 设置分配器对象的解析顺序
void ggml_allocr_set_parse_seq(ggml_allocr_t alloc, const int * list, int n) {
    ggml_gallocr_set_parse_seq(alloc->galloc, list, n);
}
// 释放分配器对象及其内部分配器
void ggml_allocr_free(ggml_allocr_t alloc) {
    ggml_gallocr_free(alloc->galloc); // 释放全局分配器
    ggml_tallocr_free(alloc->talloc); // 释放张量分配器
    free(alloc); // 释放分配器对象
}

// 判断分配器是否为度量分配器
bool ggml_allocr_is_measure(ggml_allocr_t alloc) {
    return ggml_tallocr_is_measure(alloc->talloc); // 判断张量分配器是否为度量分配器
}

// 重置分配器
void ggml_allocr_reset(ggml_allocr_t alloc) {
    ggml_tallocr_reset(alloc->talloc); // 重置张量分配器
}

// 分配张量
void ggml_allocr_alloc(ggml_allocr_t alloc, struct ggml_tensor * tensor) {
    ggml_tallocr_alloc(alloc->talloc, tensor); // 使用张量分配器分配张量
}
# 返回分配器的最大大小
size_t ggml_allocr_max_size(ggml_allocr_t alloc) {
    return ggml_tallocr_max_size(alloc->talloc);
}

# 分配器分配图形
size_t ggml_allocr_alloc_graph(ggml_allocr_t alloc, struct ggml_cgraph * graph) {
    return ggml_gallocr_alloc_graph(alloc->galloc, alloc->talloc, graph);
}
```