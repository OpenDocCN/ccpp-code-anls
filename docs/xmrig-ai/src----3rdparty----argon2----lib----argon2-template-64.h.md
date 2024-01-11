# `xmrig\src\3rdparty\argon2\lib\argon2-template-64.h`

```
#include <string.h>

#include "core.h"

#define MASK_32 UINT64_C(0xFFFFFFFF)  // 定义32位掩码

#define F(x, y) ((x) + (y) + 2 * ((x) & MASK_32) * ((y) & MASK_32))  // 定义 F 函数

#define G(a, b, c, d) \  // 定义 G 函数
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

#define BLAKE2_ROUND_NOMSG(v0, v1, v2, v3, v4, v5, v6, v7, \  // 定义 BLAKE2_ROUND_NOMSG 函数
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

#define BLAKE2_ROUND_NOMSG1(v) \  // 定义 BLAKE2_ROUND_NOMSG1 函数
    BLAKE2_ROUND_NOMSG( \
        (v)[ 0], (v)[ 1], (v)[ 2], (v)[ 3], \
        (v)[ 4], (v)[ 5], (v)[ 6], (v)[ 7], \
        (v)[ 8], (v)[ 9], (v)[10], (v)[11], \
        (v)[12], (v)[13], (v)[14], (v)[15])

#define BLAKE2_ROUND_NOMSG2(v) \  // 定义 BLAKE2_ROUND_NOMSG2 函数
    BLAKE2_ROUND_NOMSG( \
        (v)[  0], (v)[  1], (v)[ 16], (v)[ 17], \
        (v)[ 32], (v)[ 33], (v)[ 48], (v)[ 49], \
        (v)[ 64], (v)[ 65], (v)[ 80], (v)[ 81], \
        (v)[ 96], (v)[ 97], (v)[112], (v)[113])

static void fill_block(const block *prev_block, const block *ref_block,  // 定义 fill_block 函数
                       block *next_block, int with_xor)
{
    block blockR, block_tmp;

    copy_block(&blockR, ref_block);  // 复制 ref_block 到 blockR
    xor_block(&blockR, prev_block);  // 对 blockR 和 prev_block 进行异或操作
    copy_block(&block_tmp, &blockR);  // 复制 blockR 到 block_tmp
    if (with_xor) {
        xor_block(&block_tmp, next_block);  // 如果 with_xor 为真，则对 block_tmp 和 next_block 进行异或操作
    }

    /* Apply Blake2 on columns of 64-bit words: (0,1,...,15) , then
    (16,17,..31)... finally (112,113,...127) */
    BLAKE2_ROUND_NOMSG1(blockR.v + 0 * 16);  // 对 blockR.v 中的一组数据应用 BLAKE2_ROUND_NOMSG1 函数
    BLAKE2_ROUND_NOMSG1(blockR.v + 1 * 16);  // 对 blockR.v 中的一组数据应用 BLAKE2_ROUND_NOMSG1 函数
    BLAKE2_ROUND_NOMSG1(blockR.v + 2 * 16);  // 对 blockR.v 中的一组数据应用 BLAKE2_ROUND_NOMSG1 函数
    BLAKE2_ROUND_NOMSG1(blockR.v + 3 * 16);  // 对 blockR.v 中的一组数据应用 BLAKE2_ROUND_NOMSG1 函数
    # 对于给定的块，应用 BLAKE2_ROUND_NOMSG1 函数
    BLAKE2_ROUND_NOMSG1(blockR.v + 4 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 5 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 6 * 16);
    BLAKE2_ROUND_NOMSG1(blockR.v + 7 * 16);

    # 在64位字的行上应用 Blake2：(0,1,16,17,...112,113)，然后(2,3,18,19,...,114,115).. 最后(14,15,30,31,...,126,127)
    BLAKE2_ROUND_NOMSG2(blockR.v + 0 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 1 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 2 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 3 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 4 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 5 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 6 * 2);
    BLAKE2_ROUND_NOMSG2(blockR.v + 7 * 2);

    # 复制块到下一个块，然后对下一个块和当前块进行异或操作
    copy_block(next_block, &block_tmp);
    xor_block(next_block, &blockR);
    # 更新输入块的第6个元素
    input_block->v[6]++;
    # 使用零块填充地址块和输入块
    fill_block(zero_block, input_block, address_block, 0);
    fill_block(zero_block, address_block, address_block, 0);
}

# 填充64位段
static void fill_segment_64(const argon2_instance_t *instance,
                            argon2_position_t position)
{
    block *ref_block, *curr_block, *prev_block;
    block address_block, input_block, zero_block;
    uint64_t pseudo_rand, ref_index, ref_lane;
    uint32_t prev_offset, curr_offset;
    uint32_t starting_index, i;
    int data_independent_addressing;

    # 如果实例为空，则返回
    if (instance == NULL) {
        return;
    }

    # 判断是否为数据独立寻址
    data_independent_addressing = (instance->type == Argon2_i) ||
            (instance->type == Argon2_id && (position.pass == 0) &&
             (position.slice < ARGON2_SYNC_POINTS / 2));

    # 如果是数据独立寻址
    if (data_independent_addressing) {
        # 初始化零块和输入块的值
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

    # 如果是第一次pass和slice
    if ((0 == position.pass) && (0 == position.slice)) {
        starting_index = 2; # 我们已经生成了前两个块

        # 不要忘记生成地址的第一个块
        if (data_independent_addressing) {
            next_addresses(&address_block, &input_block, &zero_block);
        }
    }

    # 当前块的偏移量
    curr_offset = position.lane * instance->lane_length +
                  position.slice * instance->segment_length + starting_index;

    # 如果当前块的偏移量是lane_length的倍数
    if (0 == curr_offset % instance->lane_length) {
        # 这一lane中的最后一个块
        prev_offset = curr_offset + instance->lane_length - 1;
    } else {
        /* 如果不是第一个块，则设置前一个块的偏移量为当前偏移量减一 */
        prev_offset = curr_offset - 1;
    }

    for (i = starting_index; i < instance->segment_length;
         ++i, ++curr_offset, ++prev_offset) {
        /*1.1 如果需要，旋转 prev_offset */
        if (curr_offset % instance->lane_length == 1) {
            prev_offset = curr_offset - 1;
        }

        /* 1.2 计算参考块的索引 */
        /* 1.2.1 从前一个块中取伪随机值 */
        if (data_independent_addressing) {
            if (i % ARGON2_ADDRESSES_IN_BLOCK == 0) {
                next_addresses(&address_block, &input_block, &zero_block);
            }
            pseudo_rand = address_block.v[i % ARGON2_ADDRESSES_IN_BLOCK];
        } else {
            pseudo_rand = instance->memory[prev_offset].v[0];
        }

        /* 1.2.2 计算参考块所在的 lane */
        ref_lane = ((pseudo_rand >> 32)) % instance->lanes;

        if ((position.pass == 0) && (position.slice == 0)) {
            /* 目前无法引用其他 lane */
            ref_lane = position.lane;
        }

        /* 1.2.3 计算 lane 中可能的参考块数量 */
        position.index = i;
        ref_index = xmrig_ar2_index_alpha(instance, &position, pseudo_rand & 0xFFFFFFFF, ref_lane == position.lane);

        /* 2 创建一个新的块 */
        ref_block =
            instance->memory + instance->lane_length * ref_lane + ref_index;
        curr_block = instance->memory + curr_offset;
        prev_block = instance->memory + prev_offset;

        /* 版本 1.2.1 及更早版本：覆盖，而不是异或 */
        if (0 == position.pass || ARGON2_VERSION_10 == instance->version) {
            fill_block(prev_block, ref_block, curr_block, 0);
        } else {
            fill_block(prev_block, ref_block, curr_block, 1);
        }
    }
# 闭合前面的函数定义
```