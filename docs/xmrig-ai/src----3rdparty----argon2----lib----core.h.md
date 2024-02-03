# `xmrig\src\3rdparty\argon2\lib\core.h`

```cpp
/*
 * Argon2源代码包
 *
 * 由Daniel Dinu和Dmitry Khovratovich编写，2015年
 *
 * 本作品根据知识共享CC0 1.0许可证/豁免条款授权。
 *
 * 您应该已经收到了CC0公共领域奉献的副本
 * 这个软件。如果没有，请参见
 * <http://creativecommons.org/publicdomain/zero/1.0/>。
 */

#ifndef ARGON2_CORE_H
#define ARGON2_CORE_H

#include "3rdparty/argon2.h"

#if defined(_MSC_VER)
#define ALIGN(n) __declspec(align(16))
#elif defined(__GNUC__) || defined(__clang)
#define ALIGN(x) __attribute__((__aligned__(x)))
#else
#define ALIGN(x)
#endif

#define CONST_CAST(x) (x)(uintptr_t)

/**********************Argon2内部常量*******************************/

enum argon2_core_constants {
    /* 内存块大小（以字节为单位） */
    ARGON2_BLOCK_SIZE = 1024,
    ARGON2_QWORDS_IN_BLOCK = ARGON2_BLOCK_SIZE / 8,
    ARGON2_OWORDS_IN_BLOCK = ARGON2_BLOCK_SIZE / 16,

    /* 在Argon2i中由一次Blake调用生成的伪随机值的数量
       生成参考块位置 */
    ARGON2_ADDRESSES_IN_BLOCK = 128,

    /* 预哈希摘要长度及其扩展 */
    ARGON2_PREHASH_DIGEST_LENGTH = 64,
    ARGON2_PREHASH_SEED_LENGTH = 72
};

/*************************Argon2内部数据类型***********************/

/*
 * 结构体，表示（1KB）内存块，实现为128个64位字。
 * 内存块可以进行复制、异或操作。可以通过[]访问内部字（没有边界检查）。
 */
typedef struct block_ { uint64_t v[ARGON2_QWORDS_IN_BLOCK]; } block;

/*****************与内存块一起工作的函数******************/

/* 用@in初始化块的每个字节 */
void init_block_value(block *b, uint8_t in);

/* 将块@src复制到块@dst */
void copy_block(block *dst, const block *src);

/* 通过字节对@src进行异或运算到@dst */
void xor_block(block *dst, const block *src);
/*
 * Argon2实例：内存指针，迭代次数，内存量，类型和派生值。
 * 用于评估每个线程中构造的块的数量和位置
 */
typedef struct Argon2_instance_t {
    block *memory;          /* 内存指针 */
    uint32_t version;
    uint32_t passes;        /* 迭代次数 */
    uint32_t memory_blocks; /* 内存中的块数 */
    uint32_t segment_length;
    uint32_t lane_length;
    uint32_t lanes;
    uint32_t threads;
    argon2_type type;
    int print_internals; /* 是否打印内存块 */
    int keep_memory;
    argon2_context *context_ptr; /* 指向原始上下文 */
} argon2_instance_t;

/*
 * Argon2位置：我们当前构造块的位置。用于在线程之间分配工作。
 */
typedef struct Argon2_position_t {
    uint32_t pass;
    uint32_t lane;
    uint8_t slice;
    uint32_t index;
} argon2_position_t;

/* 保存线程处理FillSegment的输入的结构体 */
typedef struct Argon2_thread_data {
    argon2_instance_t *instance_ptr;
    argon2_position_t pos;
} argon2_thread_data;

/*************************Argon2核心函数********************************/

/* 为给定指针分配内存，使用上下文中指定的适当分配器。总分配的内存为num*size。
 * @param context 指定分配器的argon2_context
 * @param instance Argon2实例
 * @return 如果成功分配内存，则返回ARGON2_OK
 */
int xmrig_ar2_allocate_memory(const argon2_context *context, argon2_instance_t *instance);

/*
 * 释放给定指针处的内存，使用上下文中指定的适当解分配器。还使用clear_internal_memory清理内存。
 * @param context 指定解分配器的argon2_context
 * @param instance Argon2实例
 */
void xmrig_ar2_free_memory(const argon2_context *context, const argon2_instance_t *instance);
/* Function that securely cleans the memory. This ignores any flags set
 * regarding clearing memory. Usually one just calls clear_internal_memory.
 * @param mem Pointer to the memory
 * @param s Memory size in bytes
 */
void xmrig_ar2_secure_wipe_memory(void *v, size_t n);
/* Function that securely clears the memory if FLAG_clear_internal_memory is
 * set. If the flag isn't set, this function does nothing.
 * @param mem Pointer to the memory
 * @param s Memory size in bytes
 */
ARGON2_PUBLIC void xmrig_ar2_clear_internal_memory(void *v, size_t n);
/*
 * Computes absolute position of reference block in the lane following a skewed
 * distribution and using a pseudo-random value as input
 * @param instance Pointer to the current instance
 * @param position Pointer to the current position
 * @param pseudo_rand 32-bit pseudo-random value used to determine the position
 * @param same_lane Indicates if the block will be taken from the current lane.
 * If so we can reference the current segment
 * @pre All pointers must be valid
 */
uint32_t xmrig_ar2_index_alpha(const argon2_instance_t *instance, const argon2_position_t *position, uint32_t pseudo_rand, int same_lane);
/*
 * Function that validates all inputs against predefined restrictions and return
 * an error code
 * @param context Pointer to current Argon2 context
 * @return ARGON2_OK if everything is all right, otherwise one of error codes
 * (all defined in <argon2.h>
 */
int xmrig_ar2_validate_inputs(const argon2_context *context);
/*
 * Hashes all the inputs into @a blockhash[PREHASH_DIGEST_LENGTH], clears
 * password and secret if needed
 * @param  context  Pointer to the Argon2 internal structure containing memory
 * pointer, and parameters for time and space requirements.
 * @param  blockhash Buffer for pre-hashing digest
 * @param  type Argon2 type
 * @pre    @a blockhash must have at least @a PREHASH_DIGEST_LENGTH bytes
 * allocated
 */
# 定义一个函数，用于计算初始哈希值
void xmrig_ar2_initial_hash(uint8_t *blockhash, argon2_context *context, argon2_type type);

/*
 * 函数用于创建每个通道的前两个块
 * @param instance 指向当前实例的指针
 * @param blockhash 指向预哈希摘要的指针
 * @pre blockhash 必须指向@a PREHASH_SEED_LENGTH分配的值
 */
void xmrig_ar2_fill_first_blocks(uint8_t *blockhash, const argon2_instance_t *instance);

/*
 * 函数分配内存，使用Blake对输入进行哈希，并创建前两个块。返回指向已初始化每个通道2个块的主内存的指针
 * @param  context 指向包含内存指针和时间、空间需求参数的Argon2内部结构的指针
 * @param  instance 当前的Argon2实例
 * @return 如果成功返回0，如果内存分配失败返回-1。如果成功，@context->state将被修改。
 */
int xmrig_ar2_initialize(argon2_instance_t *instance, argon2_context *context);

/*
 * 对每个通道的最后一个块进行异或运算，对其进行哈希，生成标签。释放内存
 * @param context 指向当前Argon2上下文的指针（仅使用其中的输出参数）
 * @param instance 指向当前Argon2实例的指针
 * @pre instance->state必须指向必要数量的内存
 * @pre context->out必须指向outlen字节的内存
 * @pre 如果context->free_cbk不为NULL，则它应指向一个释放内存的函数
 */
void xmrig_ar2_finalize(const argon2_context *context, argon2_instance_t *instance);

/*
 * 用于使用来自其他线程的先前段填充段的函数
 * @param instance 指向当前实例的指针
 * @param position 当前位置
 * @pre 所有块指针必须有效
 */
void xmrig_ar2_fill_segment(const argon2_instance_t *instance, argon2_position_t position);
/*
 * 函数用于根据每个通道中的前两个块填充整个内存 t_cost 次
 * @param instance 指向当前实例的指针
 * @return 如果成功则返回 ARGON2_OK，否则返回 @context->state
 */
int xmrig_ar2_fill_memory_blocks(argon2_instance_t *instance);
#endif
```