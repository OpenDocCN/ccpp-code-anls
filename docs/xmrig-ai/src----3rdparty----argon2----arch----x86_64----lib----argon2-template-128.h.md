# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-template-128.h`

```cpp
#include <string.h>  // 包含字符串处理函数的头文件

#ifdef __GNUC__  // 如果是 GNU 编译器
#   include <x86intrin.h>  // 包含 SIMD 操作的头文件
#else  // 否则
#   include <intrin.h>  // 包含 SIMD 操作的头文件
#endif

#include "core.h"  // 包含核心功能的头文件

static void fill_block(__m128i *s, const block *ref_block, block *next_block,
                       int with_xor)
{
    __m128i block_XY[ARGON2_OWORDS_IN_BLOCK];  // 定义一个 SIMD 数据类型数组
    unsigned int i;  // 定义一个无符号整数

    if (with_xor) {  // 如果 with_xor 为真
        for (i = 0; i < ARGON2_OWORDS_IN_BLOCK; i++) {  // 循环 ARGON2_OWORDS_IN_BLOCK 次
            s[i] = _mm_xor_si128(  // 对 s[i] 和 ref_block->v[i] 进行按位异或操作
                        s[i], _mm_loadu_si128((const __m128i *)ref_block->v + i));
            block_XY[i] = _mm_xor_si128(  // 对 s[i] 和 next_block->v[i] 进行按位异或操作
                        s[i], _mm_loadu_si128((const __m128i *)next_block->v + i));
        }
    } else {  // 否则
        for (i = 0; i < ARGON2_OWORDS_IN_BLOCK; i++) {  // 循环 ARGON2_OWORDS_IN_BLOCK 次
            block_XY[i] = s[i] = _mm_xor_si128(  // 对 s[i] 和 ref_block->v[i] 进行按位异或操作
                        s[i], _mm_loadu_si128((const __m128i *)ref_block->v + i));
        }
    }

    for (i = 0; i < 8; ++i) {  // 循环 8 次
        BLAKE2_ROUND(  // 调用 BLAKE2_ROUND 函数
            s[8 * i + 0], s[8 * i + 1], s[8 * i + 2], s[8 * i + 3],
            s[8 * i + 4], s[8 * i + 5], s[8 * i + 6], s[8 * i + 7]);
    }

    for (i = 0; i < 8; ++i) {  // 循环 8 次
        BLAKE2_ROUND(  // 调用 BLAKE2_ROUND 函数
            s[8 * 0 + i], s[8 * 1 + i], s[8 * 2 + i], s[8 * 3 + i],
            s[8 * 4 + i], s[8 * 5 + i], s[8 * 6 + i], s[8 * 7 + i]);
    }

    for (i = 0; i < ARGON2_OWORDS_IN_BLOCK; i++) {  // 循环 ARGON2_OWORDS_IN_BLOCK 次
        s[i] = _mm_xor_si128(s[i], block_XY[i]);  // 对 s[i] 和 block_XY[i] 进行按位异或操作
        _mm_storeu_si128((__m128i *)next_block->v + i, s[i]);  // 将 s[i] 存储到 next_block->v[i]
    }
}

static void next_addresses(block *address_block, block *input_block)
{
    /*Temporary zero-initialized blocks*/
    __m128i zero_block[ARGON2_OWORDS_IN_BLOCK];  // 定义一个 SIMD 数据类型数组
    __m128i zero2_block[ARGON2_OWORDS_IN_BLOCK];  // 定义一个 SIMD 数据类型数组

    memset(zero_block, 0, sizeof(zero_block));  // 将 zero_block 数组清零
    memset(zero2_block, 0, sizeof(zero2_block));  // 将 zero2_block 数组清零

    /*Increasing index counter*/
    input_block->v[6]++;  // input_block->v[6] 自增1

    /*First iteration of G*/
    fill_block(zero_block, input_block, address_block, 0);  // 调用 fill_block 函数

    /*Second iteration of G*/
    fill_block(zero2_block, address_block, address_block, 0);  // 调用 fill_block 函数
}
# 填充128位段的函数
static void fill_segment_128(const argon2_instance_t *instance,
                             argon2_position_t position)
{
    block *ref_block = NULL, *curr_block = NULL;  # 定义两个指向block结构的指针
    block address_block, input_block;  # 定义两个block结构的变量
    uint64_t pseudo_rand, ref_index, ref_lane;  # 定义三个64位整数变量
    uint32_t prev_offset, curr_offset;  # 定义两个32位整数变量
    uint32_t starting_index, i;  # 定义两个32位整数变量
    __m128i state[ARGON2_OWORDS_IN_BLOCK];  # 定义一个包含ARGON2_OWORDS_IN_BLOCK个__m128i类型元素的数组
    int data_independent_addressing;  # 定义一个整数变量

    if (instance == NULL) {  # 如果实例为空，则返回
        return;
    }

    data_independent_addressing = (instance->type == Argon2_i) ||  # 判断是否为独立数据寻址
            (instance->type == Argon2_id && (position.pass == 0) &&
             (position.slice < ARGON2_SYNC_POINTS / 2));

    if (data_independent_addressing) {  # 如果是独立数据寻址
        init_block_value(&input_block, 0);  # 初始化input_block的值为0

        input_block.v[0] = position.pass;  # 设置input_block的第一个元素为pass
        input_block.v[1] = position.lane;  # 设置input_block的第二个元素为lane
        input_block.v[2] = position.slice;  # 设置input_block的第三个元素为slice
        input_block.v[3] = instance->memory_blocks;  # 设置input_block的第四个元素为memory_blocks
        input_block.v[4] = instance->passes;  # 设置input_block的第五个元素为passes
        input_block.v[5] = instance->type;  # 设置input_block的第六个元素为type
    }

    starting_index = 0;  # 初始化starting_index为0

    if ((0 == position.pass) && (0 == position.slice)) {  # 如果pass和slice都为0
        starting_index = 2;  # 则将starting_index设置为2，因为已经生成了前两个块

        /* Don't forget to generate the first block of addresses: */
        if (data_independent_addressing) {  # 如果是独立数据寻址
            next_addresses(&address_block, &input_block);  # 生成地址块
        }
    }

    /* Offset of the current block */
    curr_offset = position.lane * instance->lane_length +  # 计算当前块的偏移量
                  position.slice * instance->segment_length + starting_index;

    if (0 == curr_offset % instance->lane_length) {  # 如果当前偏移量对lane_length取模为0
        /* Last block in this lane */
        prev_offset = curr_offset + instance->lane_length - 1;  # 上一个块的偏移量为当前偏移量加上lane_length减1
    } else {
        /* Previous block */
        prev_offset = curr_offset - 1;  # 否则上一个块的偏移量为当前偏移量减1
    }

    memcpy(state, ((instance->memory + prev_offset)->v), ARGON2_BLOCK_SIZE);  # 复制上一个块的数据到state数组中
}
    for (i = starting_index; i < instance->segment_length;
         ++i, ++curr_offset, ++prev_offset) {
        /*1.1 Rotating prev_offset if needed */
        // 如果需要，旋转 prev_offset
        if (curr_offset % instance->lane_length == 1) {
            prev_offset = curr_offset - 1;
        }

        /* 1.2 Computing the index of the reference block */
        // 计算引用块的索引
        /* 1.2.1 Taking pseudo-random value from the previous block */
        // 从前一个块中获取伪随机值
        if (data_independent_addressing) {
            if (i % ARGON2_ADDRESSES_IN_BLOCK == 0) {
                next_addresses(&address_block, &input_block);
            }
            pseudo_rand = address_block.v[i % ARGON2_ADDRESSES_IN_BLOCK];
        } else {
            pseudo_rand = instance->memory[prev_offset].v[0];
        }

        /* 1.2.2 Computing the lane of the reference block */
        // 计算引用块所在的 lane
        ref_lane = ((pseudo_rand >> 32)) % instance->lanes;

        if ((position.pass == 0) && (position.slice == 0)) {
            /* Can not reference other lanes yet */
            // 目前还不能引用其他的 lane
            ref_lane = position.lane;
        }

        /* 1.2.3 Computing the number of possible reference block within the
         * lane.
         */
        // 计算 lane 中可能的引用块数量
        position.index = i;
        ref_index = xmrig_ar2_index_alpha(instance, &position, pseudo_rand & 0xFFFFFFFF, ref_lane == position.lane);

        /* 2 Creating a new block */
        // 创建一个新的块
        ref_block =
            instance->memory + instance->lane_length * ref_lane + ref_index;
        curr_block = instance->memory + curr_offset;

        /* version 1.2.1 and earlier: overwrite, not XOR */
        // 版本 1.2.1 及之前版本：覆盖，不是异或
        if (0 == position.pass || ARGON2_VERSION_10 == instance->version) {
            fill_block(state, ref_block, curr_block, 0);
        } else {
            fill_block(state, ref_block, curr_block, 1);
        }
    }
# 闭合前面的函数定义
```