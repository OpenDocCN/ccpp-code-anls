# `xmrig\src\3rdparty\argon2\lib\core.c`

```cpp
/*
 * Argon2源代码包
 *
 * 由Daniel Dinu和Dmitry Khovratovich编写，2015年
 *
 * 本作品根据知识共享CC0 1.0许可证/豁免授权发布。
 *
 * 您应该已经收到了CC0公共领域奉献的副本
 * 这个软件。如果没有，请参见
 * <http://creativecommons.org/publicdomain/zero/1.0/>。
 */

/*用于内存擦除*/
#ifdef _MSC_VER
#include <windows.h>
#include <winbase.h> /* 用于SecureZeroMemory */
#endif
#if defined __STDC_LIB_EXT1__
#define __STDC_WANT_LIB_EXT1__ 1
#endif
#define VC_GE_2005(version) (version >= 1400)

#include <inttypes.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include "core.h"
#include "blake2/blake2.h"
#include "blake2/blake2-impl.h"

#include "genkat.h"

#if defined(__clang__)
#if __has_attribute(optnone)
#define NOT_OPTIMIZED __attribute__((optnone))
#endif
#elif defined(__GNUC__)
#define GCC_VERSION                                                            \
    (__GNUC__ * 10000 + __GNUC_MINOR__ * 100 + __GNUC_PATCHLEVEL__)
#if GCC_VERSION >= 40400
#define NOT_OPTIMIZED __attribute__((optimize("O0")))
#endif
#endif
#ifndef NOT_OPTIMIZED
#define NOT_OPTIMIZED
#endif

/***************实例和位置构造函数**********/
void init_block_value(block *b, uint8_t in) { memset(b->v, in, sizeof(b->v)); }

void copy_block(block *dst, const block *src) {
    memcpy(dst->v, src->v, sizeof(uint64_t) * ARGON2_QWORDS_IN_BLOCK);
}

void xor_block(block *dst, const block *src) {
    int i;
    for (i = 0; i < ARGON2_QWORDS_IN_BLOCK; ++i) {
        dst->v[i] ^= src->v[i];
    }
}

static void load_block(block *dst, const void *input) {
    unsigned i;
    for (i = 0; i < ARGON2_QWORDS_IN_BLOCK; ++i) {
        dst->v[i] = load64((const uint8_t *)input + i * sizeof(dst->v[i]));
    }
}

static void store_block(void *output, const block *src) {
    unsigned i;
    # 遍历循环，从0到ARGON2_QWORDS_IN_BLOCK-1
    for (i = 0; i < ARGON2_QWORDS_IN_BLOCK; ++i) {
        # 将src->v[i]的值以64位存储到output的内存地址中
        store64((uint8_t *)output + i * sizeof(src->v[i]), src->v[i]);
    }
}

/***************Memory functions*****************/

// 分配内存函数，根据上下文和实例参数分配内存
int xmrig_ar2_allocate_memory(const argon2_context *context, argon2_instance_t *instance) {
    // 计算需要分配的内存块数和总内存大小
    size_t blocks = instance->memory_blocks;
    size_t memory_size = blocks * ARGON2_BLOCK_SIZE;

    /* 0. Check for memory supplied by user: */
    /* NOTE: Sufficient memory size is already checked in argon2_ctx_mem() */
    // 检查用户是否提供了内存，如果提供了则直接返回
    if (instance->memory != NULL) {
        return ARGON2_OK;
    }

    /* 1. Check for multiplication overflow */
    // 检查乘法溢出
    if (blocks != 0 && memory_size / ARGON2_BLOCK_SIZE != blocks) {
        return ARGON2_MEMORY_ALLOCATION_ERROR;
    }

    /* 2. Try to allocate with appropriate allocator */
    // 尝试使用适当的分配器分配内存
    if (context->allocate_cbk) {
        (context->allocate_cbk)((uint8_t **)&instance->memory, memory_size);
    } else {
        instance->memory = malloc(memory_size);
    }

    // 检查内存是否成功分配
    if (instance->memory == NULL) {
        return ARGON2_MEMORY_ALLOCATION_ERROR;
    }

    return ARGON2_OK;
}

// 释放内存函数
void xmrig_ar2_free_memory(const argon2_context *context, const argon2_instance_t *instance) {
    // 计算内存大小
    size_t memory_size = instance->memory_blocks * ARGON2_BLOCK_SIZE;

    // 清除内存内容
    xmrig_ar2_clear_internal_memory(instance->memory, memory_size);

    // 如果用户要求保留内存，则不释放
    if (instance->keep_memory) {
        /* user-supplied memory -- do not free */
        return;
    }

    // 使用适当的释放器释放内存
    if (context->free_cbk) {
        (context->free_cbk)((uint8_t *)instance->memory, memory_size);
    } else {
        free(instance->memory);
    }
}

// 安全擦除内存函数
void NOT_OPTIMIZED xmrig_ar2_secure_wipe_memory(void *v, size_t n) {
    // 根据不同的平台使用不同的方法进行内存擦除
    #if defined(_MSC_VER) && VC_GE_2005(_MSC_VER)
    SecureZeroMemory(v, n);
    #elif defined memset_s
    memset_s(v, n, 0, n);
    #elif defined(__OpenBSD__)
    explicit_bzero(v, n);
    #else
    static void *(*const volatile memset_sec)(void *, int, size_t) = &memset;
    memset_sec(v, 0, n);
    #endif
}

// 清除内存内容函数
/* Memory clear flag defaults to true. */
int FLAG_clear_internal_memory = 0;
void xmrig_ar2_clear_internal_memory(void *v, size_t n) {
    # 如果 FLAG_clear_internal_memory 为真且 v 存在
    if (FLAG_clear_internal_memory && v) {
        # 调用函数 xmrig_ar2_secure_wipe_memory 清除 v 所指向的内存，长度为 n
        xmrig_ar2_secure_wipe_memory(v, n);
    }
void xmrig_ar2_finalize(const argon2_context *context, argon2_instance_t *instance) {
    // 检查参数是否有效，以及输出缓冲区是否存在
    if (context != NULL && instance != NULL && context->out != NULL) {
        // 定义变量
        block blockhash;
        uint32_t l;

        // 复制最后一个块的内容
        copy_block(&blockhash, instance->memory + instance->lane_length - 1);

        /* XOR 最后的块 */
        for (l = 1; l < instance->lanes; ++l) {
            uint32_t last_block_in_lane =
                l * instance->lane_length + (instance->lane_length - 1);
            xor_block(&blockhash, instance->memory + last_block_in_lane);
        }

        /* 哈希结果 */
        {
            uint8_t blockhash_bytes[ARGON2_BLOCK_SIZE];
            store_block(blockhash_bytes, &blockhash);
            xmrig_ar2_blake2b_long(context->out, context->outlen, blockhash_bytes, ARGON2_BLOCK_SIZE);
            /* 清空 blockhash 和 blockhash_bytes */
            xmrig_ar2_clear_internal_memory(blockhash.v, ARGON2_BLOCK_SIZE);
            xmrig_ar2_clear_internal_memory(blockhash_bytes, ARGON2_BLOCK_SIZE);
        }

        // 如果需要打印内部信息，则打印标签
        if (instance->print_internals) {
            print_tag(context->out, context->outlen);
        }

        // 释放内存
        xmrig_ar2_free_memory(context, instance);
    }
}

uint32_t xmrig_ar2_index_alpha(const argon2_instance_t *instance, const argon2_position_t *position, uint32_t pseudo_rand, int same_lane) {
    /*
     * Pass 0:
     *      This lane : all already finished segments plus already constructed
     * blocks in this segment
     *      Other lanes : all already finished segments
     * Pass 1+:
     *      This lane : (SYNC_POINTS - 1) last segments plus already constructed
     * blocks in this segment
     *      Other lanes : (SYNC_POINTS - 1) last segments
     */
    // 定义变量
    uint32_t reference_area_size;
    uint64_t relative_position;
    uint32_t start_position, absolute_position;
    if (0 == position->pass) {
        /* 如果是第一遍循环 */
        if (0 == position->slice) {
            /* 如果是第一个切片 */
            reference_area_size =
                position->index - 1; /* 所有除了前一个 */
        } else {
            if (same_lane) {
                /* 同一条车道 => 添加当前段 */
                reference_area_size =
                    position->slice * instance->segment_length +
                    position->index - 1;
            } else {
                reference_area_size =
                    position->slice * instance->segment_length +
                    ((position->index == 0) ? (-1) : 0);
            }
        }
    } else {
        /* 如果是第二遍循环 */
        if (same_lane) {
            reference_area_size = instance->lane_length -
                                  instance->segment_length + position->index -
                                  1;
        } else {
            reference_area_size = instance->lane_length -
                                  instance->segment_length +
                                  ((position->index == 0) ? (-1) : 0);
        }
    }

    /* 1.2.4. 将伪随机数映射到 0..<reference_area_size-1> 并产生相对位置 */
    relative_position = pseudo_rand;
    relative_position = relative_position * relative_position >> 32;
    relative_position = reference_area_size - 1 -
                        (reference_area_size * relative_position >> 32);

    /* 1.2.5 计算起始位置 */
    start_position = 0;

    if (0 != position->pass) {
        start_position = (position->slice == ARGON2_SYNC_POINTS - 1)
                             ? 0
                             : (position->slice + 1) * instance->segment_length;
    }

    /* 1.2.6. 计算绝对位置 */
    absolute_position = (start_position + relative_position) %
                        instance->lane_length; /* 绝对位置 */
    return absolute_position;
/* Single-threaded version for p=1 case */
// 为 p=1 的情况编写单线程版本
static int fill_memory_blocks_st(argon2_instance_t *instance) {
    uint32_t r, s, l;

    for (r = 0; r < instance->passes; ++r) {
        for (s = 0; s < ARGON2_SYNC_POINTS; ++s) {
            for (l = 0; l < instance->lanes; ++l) {
                argon2_position_t position = { r, l, (uint8_t)s, 0 };
                xmrig_ar2_fill_segment(instance, position);
            }
        }

        if (instance->print_internals) {
            internal_kat(instance, r); /* Print all memory blocks */
        }
    }
    return ARGON2_OK;
}

// 填充内存块的函数，单线程版本
int xmrig_ar2_fill_memory_blocks(argon2_instance_t *instance) {
    if (instance == NULL || instance->lanes == 0) {
        return ARGON2_INCORRECT_PARAMETER;
    }

    return fill_memory_blocks_st(instance);
}

// 验证输入参数的有效性
int xmrig_ar2_validate_inputs(const argon2_context *context) {
    if (NULL == context) {
        return ARGON2_INCORRECT_PARAMETER;
    }

    //if (NULL == context->out) {
    //    return ARGON2_OUTPUT_PTR_NULL;
    //}

    /* Validate output length */
    //if (ARGON2_MIN_OUTLEN > context->outlen) {
    //    return ARGON2_OUTPUT_TOO_SHORT;
    //}

    if (ARGON2_MAX_OUTLEN < context->outlen) {
        return ARGON2_OUTPUT_TOO_LONG;
    }

    /* Validate password (required param) */
    if (NULL == context->pwd) {
        if (0 != context->pwdlen) {
            return ARGON2_PWD_PTR_MISMATCH;
        }
    }

    if (ARGON2_MIN_PWD_LENGTH > context->pwdlen) {
        return ARGON2_PWD_TOO_SHORT;
    }

    if (ARGON2_MAX_PWD_LENGTH < context->pwdlen) {
        return ARGON2_PWD_TOO_LONG;
    }

    /* Validate salt (required param) */
    if (NULL == context->salt) {
        if (0 != context->saltlen) {
            return ARGON2_SALT_PTR_MISMATCH;
        }
    }

    if (ARGON2_MIN_SALT_LENGTH > context->saltlen) {
        return ARGON2_SALT_TOO_SHORT;
    }

    if (ARGON2_MAX_SALT_LENGTH < context->saltlen) {
        return ARGON2_SALT_TOO_LONG;
    }
    /* 验证密钥（可选参数） */
    if (NULL == context->secret) {
        if (0 != context->secretlen) {
            return ARGON2_SECRET_PTR_MISMATCH;
        }
    } else {
        if (ARGON2_MIN_SECRET > context->secretlen) {
            return ARGON2_SECRET_TOO_SHORT;
        }
        if (ARGON2_MAX_SECRET < context->secretlen) {
            return ARGON2_SECRET_TOO_LONG;
        }
    }

    /* 验证关联数据（可选参数） */
    if (NULL == context->ad) {
        if (0 != context->adlen) {
            return ARGON2_AD_PTR_MISMATCH;
        }
    } else {
        if (ARGON2_MIN_AD_LENGTH > context->adlen) {
            return ARGON2_AD_TOO_SHORT;
        }
        if (ARGON2_MAX_AD_LENGTH < context->adlen) {
            return ARGON2_AD_TOO_LONG;
        }
    }

    /* 验证内存成本 */
    if (ARGON2_MIN_MEMORY > context->m_cost) {
        return ARGON2_MEMORY_TOO_LITTLE;
    }

    if (ARGON2_MAX_MEMORY < context->m_cost) {
        return ARGON2_MEMORY_TOO_MUCH;
    }

    if (context->m_cost < 8 * context->lanes) {
        return ARGON2_MEMORY_TOO_LITTLE;
    }

    /* 验证时间成本 */
    if (ARGON2_MIN_TIME > context->t_cost) {
        return ARGON2_TIME_TOO_SMALL;
    }

    if (ARGON2_MAX_TIME < context->t_cost) {
        return ARGON2_TIME_TOO_LARGE;
    }

    /* 验证通道数 */
    if (ARGON2_MIN_LANES > context->lanes) {
        return ARGON2_LANES_TOO_FEW;
    }

    if (ARGON2_MAX_LANES < context->lanes) {
        return ARGON2_LANES_TOO_MANY;
    }

    /* 验证线程数 */
    if (ARGON2_MIN_THREADS > context->threads) {
        return ARGON2_THREADS_TOO_FEW;
    }

    if (ARGON2_MAX_THREADS < context->threads) {
        return ARGON2_THREADS_TOO_MANY;
    }

    if (NULL != context->allocate_cbk && NULL == context->free_cbk) {
        return ARGON2_FREE_MEMORY_CBK_NULL;
    }

    if (NULL == context->allocate_cbk && NULL != context->free_cbk) {
        return ARGON2_ALLOCATE_MEMORY_CBK_NULL;
    }
    # 返回表示ARGON2操作成功的常量
    return ARGON2_OK;
}
// 声明一个函数，用于填充每个 lane 中的第一个和第二个块
void xmrig_ar2_fill_first_blocks(uint8_t *blockhash, const argon2_instance_t *instance) {
    uint32_t l;
    // 声明一个用于存储块哈希值的字节数组
    uint8_t blockhash_bytes[ARGON2_BLOCK_SIZE];
    // 遍历每个 lane
    for (l = 0; l < instance->lanes; ++l) {
        // 将 H0||0||i 存储到 blockhash 中
        store32(blockhash + ARGON2_PREHASH_DIGEST_LENGTH, 0);
        store32(blockhash + ARGON2_PREHASH_DIGEST_LENGTH + 4, l);
        // 计算块哈希值
        xmrig_ar2_blake2b_long(blockhash_bytes, ARGON2_BLOCK_SIZE, blockhash, ARGON2_PREHASH_SEED_LENGTH);
        // 将计算得到的块哈希值加载到内存中
        load_block(&instance->memory[l * instance->lane_length + 0], blockhash_bytes);
        // 将 H0||1||i 存储到 blockhash 中
        store32(blockhash + ARGON2_PREHASH_DIGEST_LENGTH, 1);
        // 重新计算块哈希值
        xmrig_ar2_blake2b_long(blockhash_bytes, ARGON2_BLOCK_SIZE, blockhash, ARGON2_PREHASH_SEED_LENGTH);
        // 将计算得到的块哈希值加载到内存中
        load_block(&instance->memory[l * instance->lane_length + 1], blockhash_bytes);
    }
    // 清空内部内存
    xmrig_ar2_clear_internal_memory(blockhash_bytes, ARGON2_BLOCK_SIZE);
}

// 声明一个函数，用于初始化哈希值
void xmrig_ar2_initial_hash(uint8_t *blockhash, argon2_context *context,
                  argon2_type type) {
    // 声明一个 blake2b_state 结构体
    blake2b_state BlakeHash;
    // 声明一个用于存储 uint32_t 类型值的字节数组
    uint8_t value[sizeof(uint32_t)];

    // 如果 context 或 blockhash 为空，则直接返回
    if (NULL == context || NULL == blockhash) {
        return;
    }

    // 初始化 blake2b 状态
    xmrig_ar2_blake2b_init(&BlakeHash, ARGON2_PREHASH_DIGEST_LENGTH);

    // 将 context 中的 lanes 存储到哈希状态中
    store32(&value, context->lanes);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    // 将 context 中的 outlen 存储到哈希状态中
    store32(&value, context->outlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    // 将 context 中的 m_cost 存储到哈希状态中
    store32(&value, context->m_cost);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    // 将 context 中的 t_cost 存储到哈希状态中
    store32(&value, context->t_cost);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    // 将 context 中的 version 存储到哈希状态中
    store32(&value, context->version);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    // 将 type 存储到哈希状态中
    store32(&value, (uint32_t)type);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));
    # 将 context->pwdlen 的值存储为 32 位无符号整数，并更新 BlakeHash
    store32(&value, context->pwdlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    # 如果 context->pwd 不为空
    if (context->pwd != NULL) {
        # 更新 BlakeHash，将 context->pwdlen 长度的 context->pwd 数据添加到哈希中
        xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)context->pwd, context->pwdlen);

        # 如果 context->flags 包含 ARGON2_FLAG_CLEAR_PASSWORD
        if (context->flags & ARGON2_FLAG_CLEAR_PASSWORD) {
            # 清除 context->pwd 数据，并将 context->pwdlen 设为 0
            xmrig_ar2_secure_wipe_memory(context->pwd, context->pwdlen);
            context->pwdlen = 0;
        }
    }

    # 将 context->saltlen 的值存储为 32 位无符号整数，并更新 BlakeHash
    store32(&value, context->saltlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    # 如果 context->salt 不为空
    if (context->salt != NULL) {
        # 更新 BlakeHash，将 context->saltlen 长度的 context->salt 数据添加到哈希中
        xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)context->salt, context->saltlen);
    }

    # 将 context->secretlen 的值存储为 32 位无符号整数，并更新 BlakeHash
    store32(&value, context->secretlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    # 如果 context->secret 不为空
    if (context->secret != NULL) {
        # 更新 BlakeHash，将 context->secretlen 长度的 context->secret 数据添加到哈希中
        xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)context->secret, context->secretlen);

        # 如果 context->flags 包含 ARGON2_FLAG_CLEAR_SECRET
        if (context->flags & ARGON2_FLAG_CLEAR_SECRET) {
            # 清除 context->secret 数据，并将 context->secretlen 设为 0
            xmrig_ar2_secure_wipe_memory(context->secret, context->secretlen);
            context->secretlen = 0;
        }
    }

    # 将 context->adlen 的值存储为 32 位无符号整数，并更新 BlakeHash
    store32(&value, context->adlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    # 如果 context->ad 不为空
    if (context->ad != NULL) {
        # 更新 BlakeHash，将 context->adlen 长度的 context->ad 数据添加到哈希中
        xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)context->ad, context->adlen);
    }

    # 完成 BlakeHash 的计算，将结果存储到 blockhash 中
    xmrig_ar2_blake2b_final(&BlakeHash, blockhash, ARGON2_PREHASH_DIGEST_LENGTH);
}
# 初始化 xmrig_ar2 函数，接受 argon2_instance_t 实例和 argon2_context 上下文作为参数
int xmrig_ar2_initialize(argon2_instance_t *instance, argon2_context *context) {
    # 用于存储块哈希的数组
    uint8_t blockhash[ARGON2_PREHASH_SEED_LENGTH];
    # 初始化结果为 ARGON2_OK
    int result = ARGON2_OK;

    # 如果实例或上下文为空，则返回参数错误
    if (instance == NULL || context == NULL)
        return ARGON2_INCORRECT_PARAMETER;
    # 将实例的上下文指针指向传入的上下文
    instance->context_ptr = context;

    /* 1. 内存分配 */

    # 调用 xmrig_ar2_allocate_memory 函数分配内存
    result = xmrig_ar2_allocate_memory(context, instance);
    # 如果分配内存失败，则返回错误结果
    if (result != ARGON2_OK) {
        return result;
    }

    /* 2. 初始哈希 */
    /* H_0 + 8 extra bytes to produce the first blocks */
    /* uint8_t blockhash[ARGON2_PREHASH_SEED_LENGTH]; */
    /* Hashing all inputs */
    # 调用 xmrig_ar2_initial_hash 函数对输入进行哈希
    xmrig_ar2_initial_hash(blockhash, context, instance->type);
    /* Zeroing 8 extra bytes */
    # 调用 xmrig_ar2_clear_internal_memory 函数清零额外的 8 个字节
    xmrig_ar2_clear_internal_memory(blockhash + ARGON2_PREHASH_DIGEST_LENGTH, ARGON2_PREHASH_SEED_LENGTH - ARGON2_PREHASH_DIGEST_LENGTH);

    # 如果需要打印内部信息，则调用 initial_kat 函数
    if (instance->print_internals) {
        initial_kat(blockhash, context, instance->type);
    }

    /* 3. 创建第一个块，每个切片中至少有两个块 */
    # 调用 xmrig_ar2_fill_first_blocks 函数创建第一个块
    xmrig_ar2_fill_first_blocks(blockhash, instance);
    # 清空哈希
    xmrig_ar2_clear_internal_memory(blockhash, ARGON2_PREHASH_SEED_LENGTH);

    # 返回成功结果
    return ARGON2_OK;
}
```