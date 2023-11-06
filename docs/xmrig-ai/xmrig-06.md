# xmrig源码解析 6

# `src/3rdparty/argon2/lib/argon2-template-64.h`

这段代码是一个C语言程序，它定义了一些宏和函数。

首先，`#include <string.h>` 是头文件，它包含了一个与字符串处理无关的函数。

接着，`#include "core.h"` 是另一张头文件，它包含了一个名为`core.h`的函数指针。

然后，定义了一个名为`MASK_32` 的常量，它是一个无符号64位整数，二进制形式为`0xFFFFFFFFF`。

接下来，定义了一个名为`F` 的函数，它接受两个整数参数`x`和`y`，计算它们的和并加上一个乘积，结果使用无符号64位整数表示。

再定义了一个名为`G` 的函数，它接受四个整数参数`a`，`b`，`c`和`d`，计算它们的和并执行一个循环，直到乘积为0。

最后，定义了一个名为`h`的函数，它的作用是输出一个字符串，其中包含了一些`printf`函数的格式化字符，以及一些定义好的宏。


```cpp
#include <string.h>

#include "core.h"

#define MASK_32 UINT64_C(0xFFFFFFFF)

#define F(x, y) ((x) + (y) + 2 * ((x) & MASK_32) * ((y) & MASK_32))

#define G(a, b, c, d) \
    do { \
        a = F(a, b); \
        d = rotr64(d ^ a, 32); \
        c = F(c, d); \
        b = rotr64(b ^ c, 24); \
        a = F(a, b); \
        d = rotr64(d ^ a, 16); \
        c = F(c, d); \
        b = rotr64(b ^ c, 63); \
    } while ((void)0, 0)

```



该代码定义了一个名为BLAKE2_ROUND_NOMSG的宏，其含义为对输入参数进行多次哈希函数计算，并将结果存储在对应的输入参数中。

具体来说，该宏包含一个do-while循环，循环变量为void，每次循环内部包含一个4x4的矩阵参数阵，表示输入参数的哈希函数计算方式。在每次循环中，通过调用一个名为G的函数，对矩阵进行计算，并计算出该次计算的结果。

该代码的作用是实现一个简单的哈希函数计算，该函数可以对任意数量的输入参数进行哈希函数计算，并返回计算结果。该函数可以被用于密码学算法等场景中，例如将任意长度的消息进行哈希，获取唯一的哈希值。


```cpp
#define BLAKE2_ROUND_NOMSG(v0, v1, v2, v3, v4, v5, v6, v7, \
                           v8, v9, v10, v11, v12, v13, v14, v15) \
    do { \
        G(v0, v4, v8,  v12); \
        G(v1, v5, v9,  v13); \
        G(v2, v6, v10, v14); \
        G(v3, v7, v11, v15); \
        G(v0, v5, v10, v15); \
        G(v1, v6, v11, v12); \
        G(v2, v7, v8,  v13); \
        G(v3, v4, v9,  v14); \
    } while ((void)0, 0)

#define BLAKE2_ROUND_NOMSG1(v) \
    BLAKE2_ROUND_NOMSG( \
        (v)[ 0], (v)[ 1], (v)[ 2], (v)[ 3], \
        (v)[ 4], (v)[ 5], (v)[ 6], (v)[ 7], \
        (v)[ 8], (v)[ 9], (v)[10], (v)[11], \
        (v)[12], (v)[13], (v)[14], (v)[15])

```

This code appears to be a section of a larger program written in C language. It defines a function called `apply_blake2`, which takes in a single parameter `blockR` representing a 256-bit block of data, and applies a sequence of Blake2 functions to this block.

The `apply_blake2` function takes in several 64-bit arguments:

* The first argument is a pointer to the input block (`blockR`)
* The second argument is a pointer to a temporary block that will be used to store the results of the Blake2 functions
* The third argument is a 64-bit integer that represents the index range for the Blake2 functions to apply. This is used to calculate the appropriate round number of the block.

The function applies the Blake2 functions to the input block and stores the results in the temporary block. The temporary block is then assigned to the input block, effectively updating it with the results of the Blake2 functions.

The `BLAKE2_ROUND_NOMSG1` function appears to be a wrapper function for the Blake2 round-number minting function. It takes in a single argument (the `blockR`) and returns the round number that should be used for the current block.

The `BLAKE2_ROUND_NOMSG2` function is also a wrapper function for the Blake2 round-number minting function, but it takes in multiple arguments (the `blockR` and the `next_block` pointers) and returns the round number that should be used for the input block.

Overall, this code appears to be a small but important part of a larger program that implementing various cryptographic algorithms using the Blake2 hashing algorithm.


```cpp
#define BLAKE2_ROUND_NOMSG2(v) \
    BLAKE2_ROUND_NOMSG( \
        (v)[  0], (v)[  1], (v)[ 16], (v)[ 17], \
        (v)[ 32], (v)[ 33], (v)[ 48], (v)[ 49], \
        (v)[ 64], (v)[ 65], (v)[ 80], (v)[ 81], \
        (v)[ 96], (v)[ 97], (v)[112], (v)[113])

static void fill_block(const block *prev_block, const block *ref_block,
                       block *next_block, int with_xor)
{
    block blockR, block_tmp;

    copy_block(&blockR, ref_block);
    xor_block(&blockR, prev_block);
    copy_block(&block_tmp, &blockR);
    if (with_xor) {
        xor_block(&block_tmp, next_block);
    }

    /* Apply Blake2 on columns of 64-bit words: (0,1,...,15) , then
    (16,17,..31)... finally (112,113,...127) */
    BLAKE2_ROUND_NOMSG1(blockR.v + 0 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 1 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 2 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 3 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 4 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 5 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 6 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 7 * 16);

    /* Apply Blake2 on rows of 64-bit words: (0,1,16,17,...112,113), then
    (2,3,18,19,...,114,115).. finally (14,15,30,31,...,126,127) */
    BLAKE2_ROUND_NOMSG2(blockR.v + 0 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 1 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 2 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 3 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 4 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 5 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 6 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 7 * 2);

    copy_block(next_block, &block_tmp);
    xor_block(next_block, &blockR);
}

```

This code appears to be a part of a larger software application that handles data


```cpp
static void next_addresses(block *address_block, block *input_block,
                           const block *zero_block)
{
    input_block->v[6]++;
    fill_block(zero_block, input_block, address_block, 0);
    fill_block(zero_block, address_block, address_block, 0);
}

static void fill_segment_64(const argon2_instance_t *instance,
                            argon2_position_t position)
{
    block *ref_block, *curr_block, *prev_block;
    block address_block, input_block, zero_block;
    uint64_t pseudo_rand, ref_index, ref_lane;
    uint32_t prev_offset, curr_offset;
    uint32_t starting_index, i;
    int data_independent_addressing;

    if (instance == NULL) {
        return;
    }

    data_independent_addressing = (instance->type == Argon2_i) ||
            (instance->type == Argon2_id && (position.pass == 0) &&
             (position.slice < ARGON2_SYNC_POINTS / 2));

    if (data_independent_addressing) {
        init_block_value(&zero_block, 0);
        init_block_value(&input_block, 0);

        input_block.v[0] = position.pass;
        input_block.v[1] = position.lane;
        input_block.v[2] = position.slice;
        input_block.v[3] = instance->memory_blocks;
        input_block.v[4] = instance->passes;
        input_block.v[5] = instance->type;
    }

    starting_index = 0;

    if ((0 == position.pass) && (0 == position.slice)) {
        starting_index = 2; /* we have already generated the first two blocks */

        /* Don't forget to generate the first block of addresses: */
        if (data_independent_addressing) {
            next_addresses(&address_block, &input_block, &zero_block);
        }
    }

    /* Offset of the current block */
    curr_offset = position.lane * instance->lane_length +
                  position.slice * instance->segment_length + starting_index;

    if (0 == curr_offset % instance->lane_length) {
        /* Last block in this lane */
        prev_offset = curr_offset + instance->lane_length - 1;
    } else {
        /* Previous block */
        prev_offset = curr_offset - 1;
    }

    for (i = starting_index; i < instance->segment_length;
         ++i, ++curr_offset, ++prev_offset) {
        /*1.1 Rotating prev_offset if needed */
        if (curr_offset % instance->lane_length == 1) {
            prev_offset = curr_offset - 1;
        }

        /* 1.2 Computing the index of the reference block */
        /* 1.2.1 Taking pseudo-random value from the previous block */
        if (data_independent_addressing) {
            if (i % ARGON2_ADDRESSES_IN_BLOCK == 0) {
                next_addresses(&address_block, &input_block, &zero_block);
            }
            pseudo_rand = address_block.v[i % ARGON2_ADDRESSES_IN_BLOCK];
        } else {
            pseudo_rand = instance->memory[prev_offset].v[0];
        }

        /* 1.2.2 Computing the lane of the reference block */
        ref_lane = ((pseudo_rand >> 32)) % instance->lanes;

        if ((position.pass == 0) && (position.slice == 0)) {
            /* Can not reference other lanes yet */
            ref_lane = position.lane;
        }

        /* 1.2.3 Computing the number of possible reference block within the
         * lane.
         */
        position.index = i;
        ref_index = xmrig_ar2_index_alpha(instance, &position, pseudo_rand & 0xFFFFFFFF, ref_lane == position.lane);

        /* 2 Creating a new block */
        ref_block =
            instance->memory + instance->lane_length * ref_lane + ref_index;
        curr_block = instance->memory + curr_offset;
        prev_block = instance->memory + prev_offset;

        /* version 1.2.1 and earlier: overwrite, not XOR */
        if (0 == position.pass || ARGON2_VERSION_10 == instance->version) {
            fill_block(prev_block, ref_block, curr_block, 0);
        } else {
            fill_block(prev_block, ref_block, curr_block, 1);
        }
    }
}

```

# `src/3rdparty/argon2/lib/argon2.c`

这段代码是一个Argon2源代码包，它定义了一些函数来处理与Argon2相关的任务。下面是这段代码的一些说明：

* **主要函数**：这段代码定义了三个函数：`argon2_status_e`，`argon2_messages_e` 和 `argon2_reset_e`。这些函数用于与Argon2服务器通信，并从客户端发送消息。

* **辅助函数**：`argon2_h16_printf` 函数用于将十六进制字符串转换为ASCII字符串。

* **错误处理**：`argon2_error_set`函数用于设置Argon2错误码。

* **日志记录**：`argon2_register_message_handler`函数用于将函数作为Argon2服务器消息处理程序注册。

* **初始化**：`argon2_init`函数用于初始化Argon2服务器。

* **运行时检查**：`argon2_run_ checks`函数用于在运行时检查Argon2服务器是否正常运行。

* **错误处理**：`argon2_handle_error`函数用于处理Argon2错误。

* **客户端消息**：`argon2_send_message_with_cost`函数用于发送客户端消息给Argon2服务器，并返回消息ID。

* **服务器消息**：`argon2_send_message`函数用于发送服务器消息给Argon2客户端，并返回消息ID。

* **时间等待**：`argon2_wait_until_ timeout`函数用于在Argon2服务器响应客户端消息之前等待一段时间。


```cpp
/*
 * Argon2 source code package
 *
 * Written by Daniel Dinu and Dmitry Khovratovich, 2015
 *
 * This work is licensed under a Creative Commons CC0 1.0 License/Waiver.
 *
 * You should have received a copy of the CC0 Public Domain Dedication along
 * with
 * this software. If not, see
 * <http://creativecommons.org/publicdomain/zero/1.0/>.
 */

#include <string.h>
#include <stdlib.h>
```

这段代码包括以下几个部分：

1. 引入了第三方的 Argon2 库，定义了一个名为 argon2_type2string 的函数，该函数接受一个 argon2_type 类型的参数以及一个 uppercase 参数。

2. 在函数内部，使用一个 switch 语句来根据输入的 argon2_type 类型，返回相应的字符串表示。

3. 使用 const 指针变量 argon2_type2string，定义了该函数的实现，但不想在函数内部使用该指针变量，而是要使用一个宏定义来调用该函数。

4. 在函数外部，通过调用 argon2_type2string 函数，传入 argon2_type 和 uppercase 参数，输出相应的字符串类型名称。


```cpp
#include <stdio.h>

#include "3rdparty/argon2.h"
#include "encoding.h"
#include "core.h"

const char *argon2_type2string(argon2_type type, int uppercase) {
    switch (type) {
        case Argon2_d:
            return uppercase ? "Argon2d" : "argon2d";
        case Argon2_i:
            return uppercase ? "Argon2i" : "argon2i";
        case Argon2_id:
            return uppercase ? "Argon2id" : "argon2id";
    }

    return NULL;
}

```

这段代码定义了一个名为 "argon2_compute_memory_blocks" 的函数，其作用是计算在基于 ARGON2 算法的时分多址系统中，需要多少个内存区域才能够支持指定数量的学生(或者执行单元)。

函数的参数包括：

- memory_blocks: 一个 32 字节的整数数组，用于保存计算得到的内存区域大小。
- segment_length: 一个 32 字节的整数数组，用于保存每个内存区域的长度(即每个时间段的大小)，也用于计算每个时间段需要的内存区域数量。
- m_cost: 一个 32 字节的整数，用于指定计算所需的最高成本(即每个时间段的最大允许成本)，超出这个成本的内存区域将自动扩展。
- lanes: 一个 32 字节的整数，用于指定每个时间段可以并行执行的执行单元数量。

函数首先根据最高成本(m_cost)计算出需要的最小内存区域大小，然后根据每个时间段可以并行执行的执行单元数量计算出每个时间段需要的内存区域大小。最后，函数检查计算得到的内存区域大小是否小于 2 * ARGON2_SYNC_POINTS * lanes，如果是，则将内存区域大小调整为 2 * ARGON2_SYNC_POINTS * lanes。最终，函数保存计算得到的内存区域大小，以供下一次调用时使用。


```cpp
static void argon2_compute_memory_blocks(uint32_t *memory_blocks,
                                         uint32_t *segment_length,
                                         uint32_t m_cost, uint32_t lanes)
{
    /* Minimum memory_blocks = 8L blocks, where L is the number of lanes */
    *memory_blocks = m_cost;
    if (*memory_blocks < 2 * ARGON2_SYNC_POINTS * lanes) {
        *memory_blocks = 2 * ARGON2_SYNC_POINTS * lanes;
    }

    *segment_length = *memory_blocks / (lanes * ARGON2_SYNC_POINTS);
    /* Ensure that all segments have equal length */
    *memory_blocks = *segment_length * (lanes * ARGON2_SYNC_POINTS);
}

```

This function appears to be an implementation of the Argon2 algorithm for secure commitment in the Bitcoin-style secure chain.

It takes a single argument of type `argon2_commitment_t` which is the commitment to be hashed.

The function first checks if the input `type` is equal to `ARGON2_COMMITMENT_CONFIRMED` and if `Argon2_id` is not equal to `type`. If this condition is not met, the function returns `ARGON2_INCORRECT_TYPE`.

It then checks if the input `memory` is not a null pointer and if it is not, it returns `ARGON2_MEMORY_ALLOCATION_ERROR`.

It then initializes the instance variables of the Argon2 algorithm, including the version, the memory block input, the hashing algorithm, the number of blocks, the number of lanes, and the number of threads.

It then checks if the memory block is larger than the maximum allowed memory size and if it is not, it returns `ARGON2_MEMORY_ALLOCATION_ERROR`.

It then initializes the hash table used by the algorithm, if it has not been done before.

It then calls the `xmrig_ar2_initialize` function with the instance variables to initialize the algorithm.

It then calls the `xmrig_ar2_fill_memory_blocks` function to fill the memory blocks with the specified input `type`.

It then calls the `xmrig_ar2_finalize` function to finalize the algorithm.

It finally returns `ARGON2_OK` on success or any other error.


```cpp
size_t argon2_memory_size(uint32_t m_cost, uint32_t parallelism) {
    uint32_t memory_blocks, segment_length;
    argon2_compute_memory_blocks(&memory_blocks, &segment_length, m_cost,
                                 parallelism);
    return memory_blocks * ARGON2_BLOCK_SIZE;
}

int argon2_ctx_mem(argon2_context *context, argon2_type type, void *memory,
                   size_t memory_size) {
    /* 1. Validate all inputs */
    int result = xmrig_ar2_validate_inputs(context);
    uint32_t memory_blocks, segment_length;
    argon2_instance_t instance;

    if (ARGON2_OK != result) {
        return result;
    }

    if (Argon2_d != type && Argon2_i != type && Argon2_id != type) {
        return ARGON2_INCORRECT_TYPE;
    }

    /* 2. Align memory size */
    argon2_compute_memory_blocks(&memory_blocks, &segment_length,
                                 context->m_cost, context->lanes);

    /* check for sufficient memory size: */
    if (memory != NULL && (memory_size % ARGON2_BLOCK_SIZE != 0 ||
                           memory_size / ARGON2_BLOCK_SIZE < memory_blocks)) {
        return ARGON2_MEMORY_ALLOCATION_ERROR;
    }

    instance.version = context->version;
    instance.memory = (block *)memory;
    instance.passes = context->t_cost;
    instance.memory_blocks = memory_blocks;
    instance.segment_length = segment_length;
    instance.lane_length = segment_length * ARGON2_SYNC_POINTS;
    instance.lanes = context->lanes;
    instance.threads = context->threads;
    instance.type = type;
    instance.print_internals = !!(context->flags & ARGON2_FLAG_GENKAT);
    instance.keep_memory = memory != NULL;

    if (instance.threads > instance.lanes) {
        instance.threads = instance.lanes;
    }

    /* 3. Initialization: Hashing inputs, allocating memory, filling first
     * blocks
     */
    result = xmrig_ar2_initialize(&instance, context);

    if (ARGON2_OK != result) {
        return result;
    }

    /* 4. Filling memory */
    result = xmrig_ar2_fill_memory_blocks(&instance);

    if (ARGON2_OK != result) {
        return result;
    }
    /* 5. Finalization */
    xmrig_ar2_finalize(context, &instance);

    return ARGON2_OK;
}

```

This function appears to be responsible for generating an Argon2 hash from a raw message and an optional encoding. It takes a input message, a raw hash flag, and an optional encoding in either case as arguments. The input message is converted to an internal representation, and if raw hash flag is set, the raw hash is generated and written to the output. If encoding flag is set, the encoding is generated and written to the output, but only if the raw hash is not generated already.


```cpp
int argon2_ctx(argon2_context *context, argon2_type type) {
    return argon2_ctx_mem(context, type, NULL, 0);
}

int argon2_hash(const uint32_t t_cost, const uint32_t m_cost,
                const uint32_t parallelism, const void *pwd,
                const size_t pwdlen, const void *salt, const size_t saltlen,
                void *hash, const size_t hashlen, char *encoded,
                const size_t encodedlen, argon2_type type,
                const uint32_t version){

    argon2_context context;
    int result;
    uint8_t *out;

    if (pwdlen > ARGON2_MAX_PWD_LENGTH) {
        return ARGON2_PWD_TOO_LONG;
    }

    if (saltlen > ARGON2_MAX_SALT_LENGTH) {
        return ARGON2_SALT_TOO_LONG;
    }

    if (hashlen > ARGON2_MAX_OUTLEN) {
        return ARGON2_OUTPUT_TOO_LONG;
    }

    if (hashlen < ARGON2_MIN_OUTLEN) {
        return ARGON2_OUTPUT_TOO_SHORT;
    }

    out = malloc(hashlen);
    if (!out) {
        return ARGON2_MEMORY_ALLOCATION_ERROR;
    }

    context.out = (uint8_t *)out;
    context.outlen = (uint32_t)hashlen;
    context.pwd = CONST_CAST(uint8_t *)pwd;
    context.pwdlen = (uint32_t)pwdlen;
    context.salt = CONST_CAST(uint8_t *)salt;
    context.saltlen = (uint32_t)saltlen;
    context.secret = NULL;
    context.secretlen = 0;
    context.ad = NULL;
    context.adlen = 0;
    context.t_cost = t_cost;
    context.m_cost = m_cost;
    context.lanes = parallelism;
    context.threads = parallelism;
    context.allocate_cbk = NULL;
    context.free_cbk = NULL;
    context.flags = ARGON2_DEFAULT_FLAGS;
    context.version = version;

    result = argon2_ctx(&context, type);

    if (result != ARGON2_OK) {
        xmrig_ar2_clear_internal_memory(out, hashlen);
        free(out);
        return result;
    }

    /* if raw hash requested, write it */
    if (hash) {
        memcpy(hash, out, hashlen);
    }

    /* if encoding requested, write it */
    if (encoded && encodedlen) {
        if (encode_string(encoded, encodedlen, &context, type) != ARGON2_OK) {
            xmrig_ar2_clear_internal_memory(out, hashlen); /* wipe buffers if error */
            xmrig_ar2_clear_internal_memory(encoded, encodedlen);
            free(out);
            return ARGON2_ENCODING_FAIL;
        }
    }
    xmrig_ar2_clear_internal_memory(out, hashlen);
    free(out);

    return ARGON2_OK;
}

```



这两个函数都涉及到了 Argon2 哈希算法，其中 `argon2i_hash_encoded` 是将输入参数进行哈希编码后的形式，而 `argon2i_hash_raw` 则是对输入参数进行哈希编码前的形式。

具体来说，这两个函数接受 4 组输入参数：

- `t_cost` 和 `m_cost` 是要进行哈希编码的参数，它们用于计算哈希值。
- `parallelism` 是并行度，用于确定哈希算法的并行处理能力。
- `pwd` 是输入数据的主指针，用于存储输入数据。
- `salt` 和 `saltlen` 是输入数据的盐值和盐盐长度，用于防止哈希碰撞和提高哈希效果。
- `encoded` 和 `encodedlen` 是输出数据编码后的指针和编码长度，用于存储编码后的数据。
- `hash` 和 `hashlen` 是输入数据哈希后的结果和编码长度，用于存储哈希结果。

函数 `argon2i_hash_encoded` 将输入参数进行哈希编码，并返回编码后的结果。哈希编码的过程中， `argon2_hash` 函数会被用来计算哈希值，这个函数的实现比较复杂，需要通过 Argon2 协议栈进行调用。

函数 `argon2i_hash_raw` 与 `argon2i_hash_encoded` 类似，只不过它返回的是输入数据的哈希编码前的结果，而 `argon2_hash` 函数的实现需要通过调用 `argon2_hash` 函数获取。因此，在 `argon2i_hash_raw` 函数中，输入数据需要自己提供哈希算法所需要的参数，包括 `salt` 和 `saltlen`。


```cpp
int argon2i_hash_encoded(const uint32_t t_cost, const uint32_t m_cost,
                         const uint32_t parallelism, const void *pwd,
                         const size_t pwdlen, const void *salt,
                         const size_t saltlen, const size_t hashlen,
                         char *encoded, const size_t encodedlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       NULL, hashlen, encoded, encodedlen, Argon2_i,
                       ARGON2_VERSION_NUMBER);
}

int argon2i_hash_raw(const uint32_t t_cost, const uint32_t m_cost,
                     const uint32_t parallelism, const void *pwd,
                     const size_t pwdlen, const void *salt,
                     const size_t saltlen, void *hash, const size_t hashlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       hash, hashlen, NULL, 0, Argon2_i, ARGON2_VERSION_NUMBER);
}

```

这两段代码是 Argon2d 哈希函数，分别实现基于密码和输入参数的哈希编码和解码。

第一个函数 `argon2d_hash_encoded` 实现将输入的密码哈希编码并返回编码后的哈希值。函数的参数包括哈希参数、密码、哈希生成的盐、盐的长度以及编码后的字符串和编码的长度。经过调用 `argon2_hash` 函数后，将得到一个哈希值，该值将被存储在 `encoded` 字眼中，并在调用结束后返回。

第二个函数 `argon2d_hash_raw` 实现与第一个函数类似，但输出的是未经过哈希编码的哈希值。该函数的参数与第一个函数相同，但哈希值将存储在 `hash` 字眼中，而不是编码后的字符串中。


```cpp
int argon2d_hash_encoded(const uint32_t t_cost, const uint32_t m_cost,
                         const uint32_t parallelism, const void *pwd,
                         const size_t pwdlen, const void *salt,
                         const size_t saltlen, const size_t hashlen,
                         char *encoded, const size_t encodedlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       NULL, hashlen, encoded, encodedlen, Argon2_d,
                       ARGON2_VERSION_NUMBER);
}

int argon2d_hash_raw(const uint32_t t_cost, const uint32_t m_cost,
                     const uint32_t parallelism, const void *pwd,
                     const size_t pwdlen, const void *salt,
                     const size_t saltlen, void *hash, const size_t hashlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       hash, hashlen, NULL, 0, Argon2_d, ARGON2_VERSION_NUMBER);
}

```

这两段代码是Argon2 ID哈希函数的实现。具体来说，

argon2_hash()函数将输入的密码(关键字)和盐(salt)与Argon2哈希算法结合，生成哈希值，并将其编码为ID格式。

argon2_hash_raw()函数与argon2_hash()函数的实现基本相同，但是对输入的哈希值进行了编码。

这两段代码的目的是实现一个Argon2 ID哈希函数，可以接受密码和盐作为输入参数，并输出一个编码后的哈希值。该哈希函数可以在不使用对称加密的情况下对密码进行安全哈希，以防止密码泄露。


```cpp
int argon2id_hash_encoded(const uint32_t t_cost, const uint32_t m_cost,
                          const uint32_t parallelism, const void *pwd,
                          const size_t pwdlen, const void *salt,
                          const size_t saltlen, const size_t hashlen,
                          char *encoded, const size_t encodedlen) {

    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       NULL, hashlen, encoded, encodedlen, Argon2_id,
                       ARGON2_VERSION_NUMBER);
}

int argon2id_hash_raw(const uint32_t t_cost, const uint32_t m_cost,
                      const uint32_t parallelism, const void *pwd,
                      const size_t pwdlen, const void *salt,
                      const size_t saltlen, void *hash, const size_t hashlen) {
    return argon2_hash(t_cost, m_cost, parallelism, pwd, pwdlen, salt, saltlen,
                       hash, hashlen, NULL, 0, Argon2_id,
                       ARGON2_VERSION_NUMBER);
}

```

这段代码是一个名为 `argon2id_hash_raw_ex` 的函数，它是 Argon2 哈希函数的一部分。它的作用是实现哈希算法，对输入的密码和盐进行哈希运算，并将哈希结果存储到输出缓冲区中。

具体来说，这段代码的功能如下：

1. 初始化 Argon2 哈希上下文，包括输出缓冲区、哈希长度、输入密码、输入盐、盐长度等信息。
2. 调用 Argon2 哈希函数 `argon2_ctx_mem`，将哈希上下文和输入数据传递给函数，并获取输出结果。
3. 将输出结果存储到输出缓冲区中，并返回。

这段代码的实现依赖于 Argon2 哈希函数的实现，它通常采用并行计算方式，对输入数据进行多次哈希运算，从而提高哈希效率。


```cpp
int argon2id_hash_raw_ex(const uint32_t t_cost, const uint32_t m_cost,
                         const uint32_t parallelism, const void *pwd,
                         const size_t pwdlen, const void *salt,
                         const size_t saltlen, void *hash, const size_t hashlen, void *memory) {
    argon2_context context;

    context.out = (uint8_t *)hash;
    context.outlen = (uint32_t)hashlen;
    context.pwd = CONST_CAST(uint8_t *)pwd;
    context.pwdlen = (uint32_t)pwdlen;
    context.salt = CONST_CAST(uint8_t *)salt;
    context.saltlen = (uint32_t)saltlen;
    context.secret = NULL;
    context.secretlen = 0;
    context.ad = NULL;
    context.adlen = 0;
    context.t_cost = t_cost;
    context.m_cost = m_cost;
    context.lanes = parallelism;
    context.threads = parallelism;
    context.allocate_cbk = NULL;
    context.free_cbk = NULL;
    context.flags = ARGON2_DEFAULT_FLAGS;
    context.version = ARGON2_VERSION_NUMBER;

    return argon2_ctx_mem(&context, Argon2_id, memory, m_cost * 1024);
}

```

It looks like the code is trying to implement Argon2 authentication in an Unix-like system. The main question is: how does this code work and what does it do?

The code has several functions and variables that are defined, but not many of them have any clear explanation or purpose. For example, the `ARGON2_OK` constant is defined, but it's not clear what it represents or what the code uses it for.

The `decode_string` function is used to decode the incoming `encoded` string, but it's unclear what the `ctx.out` variable is used for or what it's passed to.

The `argon2_verify_ctx` function is used to verify the input `ctx` and the output `desired_result`, but it's unclear what the `ctx.pwd` variable is used for or what it's passed to.

Overall, the code seems to be implementing Argon2 authentication, but it's difficult to understand the details and purpose of each function and variable.


```cpp
static int argon2_compare(const uint8_t *b1, const uint8_t *b2, size_t len) {
    size_t i;
    uint8_t d = 0U;

    for (i = 0U; i < len; i++) {
        d |= b1[i] ^ b2[i];
    }
    return (int)((1 & ((d - 1) >> 8)) - 1);
}

int argon2_verify(const char *encoded, const void *pwd, const size_t pwdlen,
                  argon2_type type) {

    argon2_context ctx;
    uint8_t *desired_result = NULL;

    int ret = ARGON2_OK;

    size_t encoded_len;
    uint32_t max_field_len;

    if (pwdlen > ARGON2_MAX_PWD_LENGTH) {
        return ARGON2_PWD_TOO_LONG;
    }

    if (encoded == NULL) {
        return ARGON2_DECODING_FAIL;
    }

    encoded_len = strlen(encoded);
    if (encoded_len > UINT32_MAX) {
        return ARGON2_DECODING_FAIL;
    }

    /* No field can be longer than the encoded length */
    max_field_len = (uint32_t)encoded_len;

    ctx.saltlen = max_field_len;
    ctx.outlen = max_field_len;

    ctx.salt = malloc(ctx.saltlen);
    ctx.out = malloc(ctx.outlen);
    if (!ctx.salt || !ctx.out) {
        ret = ARGON2_MEMORY_ALLOCATION_ERROR;
        goto fail;
    }

    ctx.pwd = (uint8_t *)pwd;
    ctx.pwdlen = (uint32_t)pwdlen;

    ret = decode_string(&ctx, encoded, type);
    if (ret != ARGON2_OK) {
        goto fail;
    }

    /* Set aside the desired result, and get a new buffer. */
    desired_result = ctx.out;
    ctx.out = malloc(ctx.outlen);
    if (!ctx.out) {
        ret = ARGON2_MEMORY_ALLOCATION_ERROR;
        goto fail;
    }

    ret = argon2_verify_ctx(&ctx, (char *)desired_result, type);
    if (ret != ARGON2_OK) {
        goto fail;
    }

```

这段代码定义了两个名为 "argon2i\_verify" 和 "argon2d\_verify" 的函数，用于执行 Argon2 客户端身份验证的不同方面。

"argon2i\_verify" 函数的参数包括三个字符串类型的输入 "encoded" 和 "pwd" 以及一个指向字符数组的指针 "pwdlen"，返回一个整数类型的变量 "ret"。这个函数的作用是执行 Argon2 客户端身份验证，首先释放已分配的内存资源，然后执行 Argon2 客户端身份验证，将输入的 "encoded" 和 "pwd" 作为参数传递给 "argon2\_verify" 函数，并将 Argon2 客户端身份验证的返回值作为输出返回。

"argon2d\_verify" 函数的参数与 "argon2i\_verify" 函数相同，只是使用了 "d" 而不是 "i"，因此它的作用是执行 Argon2 客户端身份验证，而不是 "i"。

这两个函数的具体实现可能还需要与其它代码相关联，才能实现完整的 Argon2 客户端身份验证过程。


```cpp
fail:
    free(ctx.salt);
    free(ctx.out);
    free(desired_result);

    return ret;
}

int argon2i_verify(const char *encoded, const void *pwd, const size_t pwdlen) {

    return argon2_verify(encoded, pwd, pwdlen, Argon2_i);
}

int argon2d_verify(const char *encoded, const void *pwd, const size_t pwdlen) {

    return argon2_verify(encoded, pwd, pwdlen, Argon2_d);
}

```

这是一个与Argon2库交互的代码，实现了不同上下文对Argon2ID进行验证的功能。

argon2id_verify函数接收三个参数：
1.编码后的ID数据，通过调用argon2_verify函数进行验证。
2.一个指向密码（user password）的指针，用于验证用户密码是否正确。
3.一个大小为pwdlen的整数，用于携带密码进行验证。

argon2d_ctx函数接收一个argon2_context类型的上下文对象和一个Argon2_d类型的参数，返回一个argon2_ctx类型的上下文对象。

argon2i_ctx函数与argon2d_ctx函数类似，只是与Argon2_i类型的参数和返回值匹配。

argon2id_ctx函数与argon2d_ctx函数类似，只是与Argon2_id类型的参数和返回值匹配。


```cpp
int argon2id_verify(const char *encoded, const void *pwd, const size_t pwdlen) {

    return argon2_verify(encoded, pwd, pwdlen, Argon2_id);
}

int argon2d_ctx(argon2_context *context) {
    return argon2_ctx(context, Argon2_d);
}

int argon2i_ctx(argon2_context *context) {
    return argon2_ctx(context, Argon2_i);
}

int argon2id_ctx(argon2_context *context) {
    return argon2_ctx(context, Argon2_id);
}

```

这段代码是一个用于 VerifyArgon2 消息的函数。它接受一个 Argon2 上下文对象和一个哈希值作为参数。

首先，它调用 `argon2_ctx()` 函数验证输入的 Argon2 类型。如果验证失败，函数返回失败的结果。

然后，它比较输入的哈希值和输出中的哈希值是否匹配。如果哈希不匹配，函数返回 `ARGON2_VERIFY_MISMATCH`。

如果哈希匹配，函数返回成功。

对于 `argon2d_verify_ctx()` 函数，它的参数和 `argon2_verify_ctx()` 函数相同，但使用了 `Argon2_d` 而不是 `Argon2_c` 类型。这可能是因为 `argon2d_verify_ctx()` 函数需要使用 `Argon2_d` 类型来与 `argon2d_context` 对象进行匹配。


```cpp
int argon2_verify_ctx(argon2_context *context, const char *hash,
                      argon2_type type) {
    int ret = argon2_ctx(context, type);
    if (ret != ARGON2_OK) {
        return ret;
    }

    if (argon2_compare((uint8_t *)hash, context->out, context->outlen)) {
        return ARGON2_VERIFY_MISMATCH;
    }

    return ARGON2_OK;
}

int argon2d_verify_ctx(argon2_context *context, const char *hash) {
    return argon2_verify_ctx(context, hash, Argon2_d);
}

```

This is a JavaScript function that takes an error code and a message as input and returns a more specific error message in the Argon2 framework.

The message can be used to diagnose and debug different issues that can occur when using the Argon2 framework, such as a missing required parameter, a type mismatch, a pointer out of bounds, etc.

The function takes the error code and message from the `argon2_error_碼` function and returns a more specific error message based on the different error codes.

For example, if the error code is `ARGON2_MEMORY_ALLOCATION_ERROR`, the function will return a more specific error message than if it is `ARGON2_THREAD_FAIL`.


```cpp
int argon2i_verify_ctx(argon2_context *context, const char *hash) {
    return argon2_verify_ctx(context, hash, Argon2_i);
}

int argon2id_verify_ctx(argon2_context *context, const char *hash) {
    return argon2_verify_ctx(context, hash, Argon2_id);
}

const char *argon2_error_message(int error_code) {
    switch (error_code) {
    case ARGON2_OK:
        return "OK";
    case ARGON2_OUTPUT_PTR_NULL:
        return "Output pointer is NULL";
    case ARGON2_OUTPUT_TOO_SHORT:
        return "Output is too short";
    case ARGON2_OUTPUT_TOO_LONG:
        return "Output is too long";
    case ARGON2_PWD_TOO_SHORT:
        return "Password is too short";
    case ARGON2_PWD_TOO_LONG:
        return "Password is too long";
    case ARGON2_SALT_TOO_SHORT:
        return "Salt is too short";
    case ARGON2_SALT_TOO_LONG:
        return "Salt is too long";
    case ARGON2_AD_TOO_SHORT:
        return "Associated data is too short";
    case ARGON2_AD_TOO_LONG:
        return "Associated data is too long";
    case ARGON2_SECRET_TOO_SHORT:
        return "Secret is too short";
    case ARGON2_SECRET_TOO_LONG:
        return "Secret is too long";
    case ARGON2_TIME_TOO_SMALL:
        return "Time cost is too small";
    case ARGON2_TIME_TOO_LARGE:
        return "Time cost is too large";
    case ARGON2_MEMORY_TOO_LITTLE:
        return "Memory cost is too small";
    case ARGON2_MEMORY_TOO_MUCH:
        return "Memory cost is too large";
    case ARGON2_LANES_TOO_FEW:
        return "Too few lanes";
    case ARGON2_LANES_TOO_MANY:
        return "Too many lanes";
    case ARGON2_PWD_PTR_MISMATCH:
        return "Password pointer is NULL, but password length is not 0";
    case ARGON2_SALT_PTR_MISMATCH:
        return "Salt pointer is NULL, but salt length is not 0";
    case ARGON2_SECRET_PTR_MISMATCH:
        return "Secret pointer is NULL, but secret length is not 0";
    case ARGON2_AD_PTR_MISMATCH:
        return "Associated data pointer is NULL, but ad length is not 0";
    case ARGON2_MEMORY_ALLOCATION_ERROR:
        return "Memory allocation error";
    case ARGON2_FREE_MEMORY_CBK_NULL:
        return "The free memory callback is NULL";
    case ARGON2_ALLOCATE_MEMORY_CBK_NULL:
        return "The allocate memory callback is NULL";
    case ARGON2_INCORRECT_PARAMETER:
        return "Argon2_Context context is NULL";
    case ARGON2_INCORRECT_TYPE:
        return "There is no such version of Argon2";
    case ARGON2_OUT_PTR_MISMATCH:
        return "Output pointer mismatch";
    case ARGON2_THREADS_TOO_FEW:
        return "Not enough threads";
    case ARGON2_THREADS_TOO_MANY:
        return "Too many threads";
    case ARGON2_MISSING_ARGS:
        return "Missing arguments";
    case ARGON2_ENCODING_FAIL:
        return "Encoding failed";
    case ARGON2_DECODING_FAIL:
        return "Decoding failed";
    case ARGON2_THREAD_FAIL:
        return "Threading failure";
    case ARGON2_DECODING_LENGTH_FAIL:
        return "Some of encoded parameters are too long or too short";
    case ARGON2_VERIFY_MISMATCH:
        return "The password does not match the supplied hash";
    default:
        return "Unknown error code";
    }
}

```

这段代码定义了一个名为 `argon2_encodedlen` 的函数，用于将输入参数进行 Argon2 编码并返回编码后的长度。

函数参数包括：

- `t_cost`: 传递给函数的成本参数，单位为微分(μ)
- `m_cost`: 传递给函数的成本参数，单位为微分(μ)，与 `t_cost` 中的相同
- `parallelism`: 并行度参数，用于控制并行算法的核心数量
- `saltlen`: 盐长度参数，单位为字节(byte)
- `hashlen`: 哈希长度参数，单位为字节(byte)
- `type`: Argon2 数据类型，可以是 `argon2_type_t` 中的任何一种。

函数实现中，首先将输入参数中的字符串进行 Argon2 编码，然后按照不同参数类型对编码结果进行长度计算，最后将所有计算结果拼接起来并返回编码后的长度。

具体来说，函数实现中的字符串编码部分如下：

```cpp
const char *argon2_type2string(argon2_type type, uint32_t argon2_encode) {
   const char *str = "$$v=$m=,t=,p=$$";
   const int len = strlen(str);
   uint64_t argon2_encode_len = argon2_encode * len;
   char *str_ptr = (char*) malloc((argon2_encode_len + 1) * sizeof(char));
   memcpy(str_ptr, str, argon2_encode_len);
   str_ptr[argon2_encode_len] = '\0';
   return str_ptr;
}
```

该函数中的 `argon2_type2string` 函数将输入参数 `argon2_type` 和 `argon2_encode` 作为参数，返回一个编码后的字符串。函数实现中，将字符串常量 `"$$v=$m=,t=,p=$$"` 作为输入参数，使用 `argon2_encode` 函数对其进行编码，并生成一个 `uint64_t` 类型的参数 `argon2_encode_len`，该参数表示编码后的字符串长度。然后，函数再次调用 `memcpy` 函数将编码后的字符串拷贝到一个名为 `str_ptr` 的 `char` 类型指针上，并使用 `argon2_encode_len` 参数的长度对 `str_ptr` 进行长度计算，最后将结果返回。

整个函数中，由于字符串是经过 Argon2 编码后的，因此生成的字符串长度会比输入参数中的字符串长度略长一些，这一点在函数实现中已经考虑到，所以在计算编码后的字符串长度时，还进行了 `argon2_encode` 的参数类型检查，确保不会出现溢出等情况。


```cpp
size_t argon2_encodedlen(uint32_t t_cost, uint32_t m_cost, uint32_t parallelism,
                         uint32_t saltlen, uint32_t hashlen, argon2_type type) {
    return strlen("$$v=$m=,t=,p=$$") + strlen(argon2_type2string(type, 0)) +
            numlen(t_cost) + numlen(m_cost) + numlen(parallelism) +
            b64len(saltlen) + b64len(hashlen) + numlen(ARGON2_VERSION_NUMBER) +
            1;
}

```

# `src/3rdparty/argon2/lib/core.c`

这段代码是一个Argon2源代码包，用于在Argon2的包发布过程中对内存进行清理。这段代码由Daniel Dinu和Dmitry Khovratovich编写，并于2015年发布。

在这段注释中，作者提到这段代码遵循创意版权许可（CC0）1.0许可证或弃权。这意味着，如果您想在未经授权的情况下复制、修改或分发此代码，您将不犯法。但请注意，如果您打算在商业项目中使用这段代码，您需要替换这段代码为经过授权的相应部分。

在这段注释的下面是一行预防性说明，用于告诉用户在编译或运行此代码时可能遇到的问题。

这里定义了一个内存清除函数，它使用了_MSC_VER编译器选项。如果用户在一个支持_MSC_VER选项的操作系统上运行这段代码，它将清除Argon2的内存并使Argon2处于不稳定状态。


```cpp
/*
 * Argon2 source code package
 *
 * Written by Daniel Dinu and Dmitry Khovratovich, 2015
 *
 * This work is licensed under a Creative Commons CC0 1.0 License/Waiver.
 *
 * You should have received a copy of the CC0 Public Domain Dedication along
 * with
 * this software. If not, see
 * <http://creativecommons.org/publicdomain/zero/1.0/>.
 */

/*For memory wiping*/
#ifdef _MSC_VER
```

这段代码的作用是定义了一些定义和变量，然后导入了一些头文件，接着通过条件判断是否支持某种库函数，如果支持就进行了包含。接着定义了一些变量，包括一个名为“__STDC_WANT_LIB_EXT1__”的定义，通过判断当前系统版本是否在定义的范围内来决定是否包含这个特定库。然后引入了几个头文件，以及包含了一些函数声明，这些函数都是通过和操作系统交互来实现的。接下来通过条件判断当前系统是否是GeOS版本2.0或更高版本，如果是就执行接下来的代码。最后定义了一些字符串和输入输出函数。


```cpp
#include <windows.h>
#include <winbase.h> /* For SecureZeroMemory */
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
```

这段代码包含了一个头文件 "blake2/blake2-impl.h"，一个引入了 "genkat.h" 的头文件，以及一些定义和宏。

宏定义：
```cpp
#define NOT_OPTIMIZED __attribute__((optnone))
#define GCC_VERSION __GNUC__ * 10000 + __GNUC__MINOR__ * 100 + __GNUC__PATCHLEVEL__)
#if GCC_VERSION >= 40400
#define NOT_OPTIMIZED __attribute__((optimize("O0")))
#endif
```

函数声明：
```cpp
void init_blake2();
```

初始化 "blake2" 的实现。
```cpp
void init_blake2() {
   // initialize to the default behavior
   init_blake2_impl();
}
```

未定义的函数声明：
```cpp
void init_blake2_impl();
```

这是一个定义头文件层次结构的代码片段。通过引入 "genkat.h"，可能启动此代码片段的是 "genkat" 库，而不是 "blake2"。


```cpp
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
```

这段代码是一个C语言的预处理指令，作用是定义了一个名为“NOT_OPTIMIZED”的定义，只有在该定义存在时才会编译通过。NOT_OPTIMIZED定义了三个函数：init_block_value、copy_block和xor_block。

init_block_value函数的作用是在定义了block结构体类型的变量b之后，初始化block结构体变量的值，并使用memset函数将传入的in参数存储到block结构体变量b中。

copy_block函数的作用是复制block结构体变量dst和src之间的所有成员变量值，并使用memcpy函数将src结构体变量src的值复制到dst结构体变量d中。

xor_block函数的作用是对block结构体变量dst中所有成员变量进行按位异或操作，并使用memcpy函数将结果存储到dst中。这里的按位异或操作是通过将src结构体变量src的值与const block结构体变量src指向的内存区域进行异或操作来实现的。


```cpp
#ifndef NOT_OPTIMIZED
#define NOT_OPTIMIZED
#endif

/***************Instance and Position constructors**********/
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

```



以上代码定义了两个名为 `load_block` 和 `store_block` 的函数，用于在神经网络中对输入数据进行预处理和后处理。

`load_block` 函数接收一个目标块(block)和一个输入数据，其主要作用是读取输入数据中的前 8 个字节，并将其存储到目标块的对应位置中。

具体来说，函数中使用了一个名为 `for` 的循环，循环从输入数据的第一个字节开始，每次读取 4 个字节，将其存储到目标块的对应位置中。由于输入数据中的每个字节都是无符号整数，因此函数中的 `load64` 函数会将其解码为 32 位无符号整数，并将其存储到目标块的对应位置中。

`store_block` 函数则与 `load_block` 函数相反，其主要作用是将处理过的目标块存储回输入数据中。函数接收一个输出块(block)和一个输入数据，其主要作用是将处理过的目标块中的前 8 个字节存储回输入数据中。

具体来说，函数中同样使用了一个名为 `for` 的循环，循环从目标块的第一个字节开始，每次读取 4 个字节，将其存储到输入数据中。由于目标块中的每个字节都是无符号整数，因此函数中的 `store64` 函数会将其解码为 32 位无符号整数，并将其存储到输入数据中。


```cpp
static void load_block(block *dst, const void *input) {
    unsigned i;
    for (i = 0; i < ARGON2_QWORDS_IN_BLOCK; ++i) {
        dst->v[i] = load64((const uint8_t *)input + i * sizeof(dst->v[i]));
    }
}

static void store_block(void *output, const block *src) {
    unsigned i;
    for (i = 0; i < ARGON2_QWORDS_IN_BLOCK; ++i) {
        store64((uint8_t *)output + i * sizeof(src->v[i]), src->v[i]);
    }
}

/***************Memory functions*****************/

```

这段代码是 Argon2 库中的一个函数，名为 `xmrig_ar2_allocate_memory()`。它接受两个参数：`argon2_context` 指针和 `argon2_instance_t` 结构体。

函数的作用是分配给 `argon2_instance_t` 类型的实例足够的内存，以使其能够正常运行。它主要实现以下几点：

1. 检查内存是否由用户供应。如果实例中已经存在内存，那么函数返回 `ARGON2_OK`。否则，函数会继续执行。
2. 检查内存是否可以分配。它分为以下两个部分：

a. 检查 `blocks` 是否为 0。如果是，那么函数认为用户已经指定了所有内存，可以返回 `ARGON2_OK`。
b. 如果 `blocks` 不是 0，那么函数会检查分配的内存是否可以整除 `ARGON2_BLOCK_SIZE`。如果不能整除，那么函数返回 `ARGON2_MEMORY_ALLOCATION_ERROR`。
3. 如果内存分配成功，函数会执行以下操作：

a. 如果函数指针 `context->allocate_cbk` 存在，那么函数使用 `context->allocate_cbk` 作为函数指针来分配内存。
b. 如果 `context->allocate_cbk` 不存在，那么函数将使用系统调用 `malloc` 动态分配内存。
4. 函数的返回值：如果内存分配成功，函数返回 `ARGON2_OK`。如果分配失败，函数返回 `ARGON2_MEMORY_ALLOCATION_ERROR`。


```cpp
int xmrig_ar2_allocate_memory(const argon2_context *context, argon2_instance_t *instance) {
    size_t blocks = instance->memory_blocks;
    size_t memory_size = blocks * ARGON2_BLOCK_SIZE;

    /* 0. Check for memory supplied by user: */
    /* NOTE: Sufficient memory size is already checked in argon2_ctx_mem() */
    if (instance->memory != NULL) {
        return ARGON2_OK;
    }

    /* 1. Check for multiplication overflow */
    if (blocks != 0 && memory_size / ARGON2_BLOCK_SIZE != blocks) {
        return ARGON2_MEMORY_ALLOCATION_ERROR;
    }

    /* 2. Try to allocate with appropriate allocator */
    if (context->allocate_cbk) {
        (context->allocate_cbk)((uint8_t **)&instance->memory, memory_size);
    } else {
        instance->memory = malloc(memory_size);
    }

    if (instance->memory == NULL) {
        return ARGON2_MEMORY_ALLOCATION_ERROR;
    }

    return ARGON2_OK;
}

```

这段代码是一个名为 "xmrig\_ar2\_free\_memory" 的函数，其作用是释放给定的 AR2 上下文的 instance 实例所占用的内存。

具体来说，函数首先计算 instance 实例的内存块数量和每个内存块的大小，然后调用内部函数 xmrig\_ar2\_clear\_internal\_memory 清除 instance 实例自身的内存，接着判断 instance 是否启用了保留内存功能，如果是，则不做额外的内存释放，否则释放 instance 实例的内存。

最后，如果 AR2 上下文中定义了 free\_cbk 函数，则调用它来释放 instance 实例所占用的内存；否则，直接释放内存。


```cpp
void xmrig_ar2_free_memory(const argon2_context *context, const argon2_instance_t *instance) {
    size_t memory_size = instance->memory_blocks * ARGON2_BLOCK_SIZE;

    xmrig_ar2_clear_internal_memory(instance->memory, memory_size);

    if (instance->keep_memory) {
        /* user-supplied memory -- do not free */
        return;
    }

    if (context->free_cbk) {
        (context->free_cbk)((uint8_t *)instance->memory, memory_size);
    } else {
        free(instance->memory);
    }
}

```



这段代码定义了一个名为 NOT_OPTIMIZED 的函数，它采用以下的方式清除内存：

1. 如果定义了 _MSC_VER，并且 VC_GE_2005(_MSC_VER)，则调用 SecureZeroMemory 函数，该函数使用系统调用，在内存的起始位置和长度范围内清空内存。
2. 如果定义了 memset_s，则调用 memset_s 函数，该函数使用函数指针存储的函数，传入参数 n 和起始位置，用于清空内存。
3. 如果定义了 __OpenBSD__，则调用 explicit_bzero 函数，该函数使用函数指针存储的函数，传入参数 v 和 n，用于清空内存。
4. 否则，根据系统是否支持内存标记，调用相应的函数或函数指针。

FLAG_clear_internal_memory 是一个整型变量，用于记录是否已经清除过内部内存。


```cpp
void NOT_OPTIMIZED xmrig_ar2_secure_wipe_memory(void *v, size_t n) {
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

/* Memory clear flag defaults to true. */
int FLAG_clear_internal_memory = 0;
```

这段代码定义了两个函数：xmrig_ar2_clear_internal_memory 和 xmrig_ar2_finalize。它们在 Aron2 安全框架中用于不同的事情。

xmrig_ar2_clear_internal_memory 的作用是清除传入的内部内存，将其清零并输出。这个函数是在 Aron2 客户端在清理输出缓冲区时调用。如果FLAG_clear_internal_memory为真，则会执行以下操作：

1. 从实例的内存中定位输入缓冲区。
2. 从内存中 copiesided 地输出当前缓冲区。
3. 清除输出缓冲区中所有内存。

xmrig_ar2_finalize 的作用是在 Aron2 客户端实例完成后执行。它清理并输出实例的内存，并清除打印缓冲区。这个函数是在 Aron2 客户端实例被销毁时调用，或者客户端打印缓冲区已满，需要清理输出缓冲区时调用。

这两个函数都使用 xmrig_ar2_secure_wipe_memory 函数来输出内存，这个函数会在 Aron2 客户端实例的内存中安全地写入，不会破坏客户端的内存，因此被用来清除输出缓冲区。


```cpp
void xmrig_ar2_clear_internal_memory(void *v, size_t n) {
    if (FLAG_clear_internal_memory && v) {
        xmrig_ar2_secure_wipe_memory(v, n);
    }
}

void xmrig_ar2_finalize(const argon2_context *context, argon2_instance_t *instance) {
    if (context != NULL && instance != NULL && context->out != NULL) {
        block blockhash;
        uint32_t l;

        copy_block(&blockhash, instance->memory + instance->lane_length - 1);

        /* XOR the last blocks */
        for (l = 1; l < instance->lanes; ++l) {
            uint32_t last_block_in_lane =
                l * instance->lane_length + (instance->lane_length - 1);
            xor_block(&blockhash, instance->memory + last_block_in_lane);
        }

        /* Hash the result */
        {
            uint8_t blockhash_bytes[ARGON2_BLOCK_SIZE];
            store_block(blockhash_bytes, &blockhash);
            xmrig_ar2_blake2b_long(context->out, context->outlen, blockhash_bytes, ARGON2_BLOCK_SIZE);
            /* clear blockhash and blockhash_bytes */
            xmrig_ar2_clear_internal_memory(blockhash.v, ARGON2_BLOCK_SIZE);
            xmrig_ar2_clear_internal_memory(blockhash_bytes, ARGON2_BLOCK_SIZE);
        }

        if (instance->print_internals) {
            print_tag(context->out, context->outlen);
        }

        xmrig_ar2_free_memory(context, instance);
    }
}

```

This code appears to be a segmentation function that takes in position and slice information and produces a relative position for a given layer. It uses a two-pass approach, first computing the reference area size and position for


```cpp
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
    uint32_t reference_area_size;
    uint64_t relative_position;
    uint32_t start_position, absolute_position;

    if (0 == position->pass) {
        /* First pass */
        if (0 == position->slice) {
            /* First slice */
            reference_area_size =
                position->index - 1; /* all but the previous */
        } else {
            if (same_lane) {
                /* The same lane => add current segment */
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
        /* Second pass */
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

    /* 1.2.4. Mapping pseudo_rand to 0..<reference_area_size-1> and produce
     * relative position */
    relative_position = pseudo_rand;
    relative_position = relative_position * relative_position >> 32;
    relative_position = reference_area_size - 1 -
                        (reference_area_size * relative_position >> 32);

    /* 1.2.5 Computing starting position */
    start_position = 0;

    if (0 != position->pass) {
        start_position = (position->slice == ARGON2_SYNC_POINTS - 1)
                             ? 0
                             : (position->slice + 1) * instance->segment_length;
    }

    /* 1.2.6. Computing absolute position */
    absolute_position = (start_position + relative_position) %
                        instance->lane_length; /* absolute position */
    return absolute_position;
}

```

这段代码是一个用于将AR2缓存中的数据写入到内存中的单线程函数。它接收一个AR2实例对象作为参数，并使用该实例对象来设置AR2缓冲区的起始位置和大小。

函数的主要部分是一个循环，该循环从0到实例的passes计数器中不包括的循环。在循环内部，它使用三个变量r、s和l来跟踪当前计数器，分别代表AR2缓冲区中的列号，行号和列。

在循环体内，它首先定义了一个argon2_position_t类型的变量position，并使用xmrig_ar2_fill_segment函数将其标记为完成的AR2缓冲区块的位置和长度。

接下来，它通过判断实例的print_internals标志来决定是否打印出当前内存区域中的所有数据。

该函数返回ARGON2_OK作为其结果，表示AR2缓冲区的所有数据已正确写入到内存中。


```cpp
/* Single-threaded version for p=1 case */
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

```

This function appears to validate the arguments passed to it, and returns an error code based on whether any of the conditions specified in the if-else statement are met.

The function checks whether the memory allocated by the program meets certain requirements, such as not using too much memory, not taking too long to complete, having the appropriate number of lanes, and not having too many threads.

If any of the conditions specified in the if-else statement are not met, the function returns an error code.


```cpp
int xmrig_ar2_fill_memory_blocks(argon2_instance_t *instance) {
    if (instance == NULL || instance->lanes == 0) {
        return ARGON2_INCORRECT_PARAMETER;
    }

    return fill_memory_blocks_st(instance);
}

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

    /* Validate secret (optional param) */
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

    /* Validate associated data (optional param) */
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

    /* Validate memory cost */
    if (ARGON2_MIN_MEMORY > context->m_cost) {
        return ARGON2_MEMORY_TOO_LITTLE;
    }

    if (ARGON2_MAX_MEMORY < context->m_cost) {
        return ARGON2_MEMORY_TOO_MUCH;
    }

    if (context->m_cost < 8 * context->lanes) {
        return ARGON2_MEMORY_TOO_LITTLE;
    }

    /* Validate time cost */
    if (ARGON2_MIN_TIME > context->t_cost) {
        return ARGON2_TIME_TOO_SMALL;
    }

    if (ARGON2_MAX_TIME < context->t_cost) {
        return ARGON2_TIME_TOO_LARGE;
    }

    /* Validate lanes */
    if (ARGON2_MIN_LANES > context->lanes) {
        return ARGON2_LANES_TOO_FEW;
    }

    if (ARGON2_MAX_LANES < context->lanes) {
        return ARGON2_LANES_TOO_MANY;
    }

    /* Validate threads */
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

    return ARGON2_OK;
}

```

这段代码的作用是实现 `xmrig_ar2_fill_first_blocks` 函数。该函数接受两个参数：一个指向 `argon2_instance_t` 类型的实例，以及一个指向 `uint8_t` 类型一维区块链哈希的指针。函数的主要目的是在初始化后，对每个通道的第一个和第二个区块进行处理，以某种方式计算并存储预哈希值。

具体来说，这段代码执行以下操作：

1. 为每个通道的第一个区块计算预哈希值。这一步的具体实现是通过调用 `xmrig_ar2_blake2b_long` 函数实现的。这个函数接受一个 32 字节的哈希块、一个 32 字节的字符串 ARGON2 实例和预设的种子（ARGON2_PREHASH_SEED_LENGTH）作为输入参数。函数的返回值是一个 32 字节的整数，表示预哈希值。
2. 将预哈希值存储到每个通道的第一个区块的哈希块中。这一步的具体实现是通过调用 `store_block` 函数实现的。这个函数接收一个 32 字节的整数、一个指向哈希块的指针和一个指向字符串的指针作为输入参数。函数的返回值是一个 32 字节的整数，表示更新后的哈希块。
3. 重复执行步骤 1-2，对每个通道的第二个区块进行处理。这一步的具体实现与步骤 1 类似，只是使用了不同的函数 `xmrig_ar2_blake2b_long`。
4. 最后，清除内部内存。


```cpp
void xmrig_ar2_fill_first_blocks(uint8_t *blockhash, const argon2_instance_t *instance) {
    uint32_t l;
    /* Make the first and second block in each lane as G(H0||0||i) or
       G(H0||1||i) */
    uint8_t blockhash_bytes[ARGON2_BLOCK_SIZE];
    for (l = 0; l < instance->lanes; ++l) {

        store32(blockhash + ARGON2_PREHASH_DIGEST_LENGTH, 0);
        store32(blockhash + ARGON2_PREHASH_DIGEST_LENGTH + 4, l);
        xmrig_ar2_blake2b_long(blockhash_bytes, ARGON2_BLOCK_SIZE, blockhash, ARGON2_PREHASH_SEED_LENGTH);
        load_block(&instance->memory[l * instance->lane_length + 0], blockhash_bytes);

        store32(blockhash + ARGON2_PREHASH_DIGEST_LENGTH, 1);
        xmrig_ar2_blake2b_long(blockhash_bytes, ARGON2_BLOCK_SIZE, blockhash, ARGON2_PREHASH_SEED_LENGTH);
        load_block(&instance->memory[l * instance->lane_length + 1], blockhash_bytes);
    }
    xmrig_ar2_clear_internal_memory(blockhash_bytes, ARGON2_BLOCK_SIZE);
}

```

This is a function signature for an algorithm that uses the AR24架构. It appears to be a part of a library that implements the ARC4182 standard for secure authentication in the ARC24 architecture.

The function takes a password block (pwd) and its length, and performs a blur hash of it using the AR24 algorithm. The AR24 algorithm is a variation of the AR256 algorithm that is designed to be more secure and efficient than the AR256 algorithm.

The function first converts the password block to a read-only pointer to a buffer of user-provided data (salt). It then loops through the block and performs a blur hash of each element in the block using the AR24 algorithm.

The AR24 algorithm performs a one-way cryptographic hash function that produces a 256-bit hash value for each 16-bit input block. The hash value is generated by taking the 16-bit input block and与世界聊message authentication code (MAC) blocks using the AR256 algorithm.

The function returns the hashed password block as a memory-truncated read-only pointer to a buffer containing the hashed password block.


```cpp
void xmrig_ar2_initial_hash(uint8_t *blockhash, argon2_context *context,
                  argon2_type type) {
    blake2b_state BlakeHash;
    uint8_t value[sizeof(uint32_t)];

    if (NULL == context || NULL == blockhash) {
        return;
    }

    xmrig_ar2_blake2b_init(&BlakeHash, ARGON2_PREHASH_DIGEST_LENGTH);

    store32(&value, context->lanes);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    store32(&value, context->outlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    store32(&value, context->m_cost);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    store32(&value, context->t_cost);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    store32(&value, context->version);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    store32(&value, (uint32_t)type);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    store32(&value, context->pwdlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    if (context->pwd != NULL) {
        xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)context->pwd,
                       context->pwdlen);

        if (context->flags & ARGON2_FLAG_CLEAR_PASSWORD) {
            xmrig_ar2_secure_wipe_memory(context->pwd, context->pwdlen);
            context->pwdlen = 0;
        }
    }

    store32(&value, context->saltlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    if (context->salt != NULL) {
        xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)context->salt,
                       context->saltlen);
    }

    store32(&value, context->secretlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    if (context->secret != NULL) {
        xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)context->secret, context->secretlen);

        if (context->flags & ARGON2_FLAG_CLEAR_SECRET) {
            xmrig_ar2_secure_wipe_memory(context->secret, context->secretlen);
            context->secretlen = 0;
        }
    }

    store32(&value, context->adlen);
    xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)&value, sizeof(value));

    if (context->ad != NULL) {
        xmrig_ar2_blake2b_update(&BlakeHash, (const uint8_t *)context->ad, context->adlen);
    }

    xmrig_ar2_blake2b_final(&BlakeHash, blockhash, ARGON2_PREHASH_DIGEST_LENGTH);
}

```

这段代码是Argon2库中的一个函数，作用是初始化一个Argon2实例，并将其配置为使用给定的Argon2上下文。

具体来说，代码的主要步骤如下：

1. 分配内存并初始化上下文：从Argon2库中分配一块内存，并将其设置为要使用的上下文对象。

2. 生成种子：使用xmrig_ar2_initial_hash函数生成一个种子，这个种子将用于生成所有的块。

3. 初始化哈希表：使用xmrig_ar2_allocate_memory函数初始化一个内部哈希表，这个哈希表将用于存储块的信息。

4. 填充块：使用xmrig_ar2_fill_first_blocks函数填充第一个块。

5. 清理哈希表：使用xmrig_ar2_clear_internal_memory函数清除哈希表中的所有内存，并使用xmrig_ar2_clear_internal_memory函数清除之前分配的内存。

6. 检查是否打印内部信息：如果设置了打印内部信息，则使用初始化库中的函数进行打印。

7. 返回结果：如果初始化成功，则返回ARGON2_OK。


```cpp
int xmrig_ar2_initialize(argon2_instance_t *instance, argon2_context *context) {
    uint8_t blockhash[ARGON2_PREHASH_SEED_LENGTH];
    int result = ARGON2_OK;

    if (instance == NULL || context == NULL)
        return ARGON2_INCORRECT_PARAMETER;
    instance->context_ptr = context;

    /* 1. Memory allocation */

    result = xmrig_ar2_allocate_memory(context, instance);
    if (result != ARGON2_OK) {
        return result;
    }

    /* 2. Initial hashing */
    /* H_0 + 8 extra bytes to produce the first blocks */
    /* uint8_t blockhash[ARGON2_PREHASH_SEED_LENGTH]; */
    /* Hashing all inputs */
    xmrig_ar2_initial_hash(blockhash, context, instance->type);
    /* Zeroing 8 extra bytes */
    xmrig_ar2_clear_internal_memory(blockhash + ARGON2_PREHASH_DIGEST_LENGTH, ARGON2_PREHASH_SEED_LENGTH - ARGON2_PREHASH_DIGEST_LENGTH);

    if (instance->print_internals) {
        initial_kat(blockhash, context, instance->type);
    }

    /* 3. Creating first blocks, we always have at least two blocks in a slice
     */
    xmrig_ar2_fill_first_blocks(blockhash, instance);
    /* Clearing the hash */
    xmrig_ar2_clear_internal_memory(blockhash, ARGON2_PREHASH_SEED_LENGTH);

    return ARGON2_OK;
}

```