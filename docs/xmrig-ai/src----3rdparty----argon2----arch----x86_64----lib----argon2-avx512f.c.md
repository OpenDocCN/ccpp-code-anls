# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-avx512f.c`

```
#include "argon2-avx512f.h"

#ifdef HAVE_AVX512F
#include <stdint.h>
#include <string.h>

#ifdef __GNUC__
#   include <x86intrin.h>
#else
#   include <intrin.h>
#endif

#define ror64(x, n) _mm512_ror_epi64((x), (n))  // 定义一个宏，用于对 64 位的 __m512i 类型数据进行循环右移操作

static __m512i f(__m512i x, __m512i y)  // 定义一个函数，接受两个 __m512i 类型的参数，返回一个 __m512i 类型的结果
{
    __m512i z = _mm512_mul_epu32(x, y);  // 计算两个 __m512i 类型数据的无符号整数乘法
    return _mm512_add_epi64(_mm512_add_epi64(x, y), _mm512_add_epi64(z, z));  // 返回四个 __m512i 类型数据的和
}

#define G1(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义一个宏，接受八个 __m512i 类型的参数，执行一系列操作
    do { \
        A0 = f(A0, B0);  // 调用函数 f 对 A0 和 B0 进行操作
        A1 = f(A1, B1);  // 调用函数 f 对 A1 和 B1 进行操作
\
        D0 = _mm512_xor_si512(D0, A0);  // 对 D0 和 A0 进行按位异或操作
        D1 = _mm512_xor_si512(D1, A1);  // 对 D1 和 A1 进行按位异或操作
\
        D0 = ror64(D0, 32);  // 对 D0 进行 32 位的循环右移操作
        D1 = ror64(D1, 32);  // 对 D1 进行 32 位的循环右移操作
\
        C0 = f(C0, D0);  // 调用函数 f 对 C0 和 D0 进行操作
        C1 = f(C1, D1);  // 调用函数 f 对 C1 和 D1 进行操作
\
        B0 = _mm512_xor_si512(B0, C0);  // 对 B0 和 C0 进行按位异或操作
        B1 = _mm512_xor_si512(B1, C1);  // 对 B1 和 C1 进行按位异或操作
\
        B0 = ror64(B0, 24);  // 对 B0 进行 24 位的循环右移操作
        B1 = ror64(B1, 24);  // 对 B1 进行 24 位的循环右移操作
    } while ((void)0, 0)

#define G2(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义一个宏，接受八个 __m512i 类型的参数，执行一系列操作
    do { \
        A0 = f(A0, B0);  // 调用函数 f 对 A0 和 B0 进行操作
        A1 = f(A1, B1);  // 调用函数 f 对 A1 和 B1 进行操作
\
        D0 = _mm512_xor_si512(D0, A0);  // 对 D0 和 A0 进行按位异或操作
        D1 = _mm512_xor_si512(D1, A1);  // 对 D1 和 A1 进行按位异或操作
\
        D0 = ror64(D0, 16);  // 对 D0 进行 16 位的循环右移操作
        D1 = ror64(D1, 16);  // 对 D1 进行 16 位的循环右移操作
\
        C0 = f(C0, D0);  // 调用函数 f 对 C0 和 D0 进行操作
        C1 = f(C1, D1);  // 调用函数 f 对 C1 和 D1 进行操作
\
        B0 = _mm512_xor_si512(B0, C0);  // 对 B0 和 C0 进行按位异或操作
        B1 = _mm512_xor_si512(B1, C1);  // 对 B1 和 C1 进行按位异或操作
\
        B0 = ror64(B0, 63);  // 对 B0 进行 63 位的循环右移操作
        B1 = ror64(B1, 63);  // 对 B1 进行 63 位的循环右移操作
    } while ((void)0, 0)

#define DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义一个宏，接受八个 __m512i 类型的参数，执行一系列操作
    do { \
        B0 = _mm512_permutex_epi64(B0, _MM_SHUFFLE(0, 3, 2, 1));  // 对 B0 进行按索引排列
        B1 = _mm512_permutex_epi64(B1, _MM_SHUFFLE(0, 3, 2, 1));  // 对 B1 进行按索引排列
\
        C0 = _mm512_permutex_epi64(C0, _MM_SHUFFLE(1, 0, 3, 2));  // 对 C0 进行按索引排列
        C1 = _mm512_permutex_epi64(C1, _MM_SHUFFLE(1, 0, 3, 2));  // 对 C1 进行按索引排列
\
        D0 = _mm512_permutex_epi64(D0, _MM_SHUFFLE(2, 1, 0, 3));  // 对 D0 进行按索引排列
        D1 = _mm512_permutex_epi64(D1, _MM_SHUFFLE(2, 1, 0, 3));  // 对 D1 进行按索引排列
    } while ((void)0, 0)

#define UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \  // 定义一个宏，接受八个 __m512i 类型的参数，执行一系列操作
    # 使用_mm512_permutex_epi64函数对寄存器B0进行排列操作，将第2、1、0、3个元素重新排列
    B0 = _mm512_permutex_epi64(B0, _MM_SHUFFLE(2, 1, 0, 3));
    # 使用_mm512_permutex_epi64函数对寄存器B1进行排列操作，将第2、1、0、3个元素重新排列
    B1 = _mm512_permutex_epi64(B1, _MM_SHUFFLE(2, 1, 0, 3));
#define BLAKE2_ROUND(A0, B0, C0, D0, A1, B1, C1, D1) \ 
    do { \ 
        // 调用 G1 和 G2 函数对 A0, B0, C0, D0, A1, B1, C1, D1 进行轮换
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        // 对角化 A0, B0, C0, D0, A1, B1, C1, D1
        DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        // 再次调用 G1 和 G2 函数对 A0, B0, C0, D0, A1, B1, C1, D1 进行轮换
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        // 反对角化 A0, B0, C0, D0, A1, B1, C1, D1
        UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \ 
    } while ((void)0, 0)

#define SWAP_HALVES(A0, A1) \ 
    do { \ 
        // 创建临时变量 t0, t1，将 A0, A1 的前半部分和后半部分进行交换
        __m512i t0, t1; \ 
        t0 = _mm512_shuffle_i64x2(A0, A1, _MM_SHUFFLE(1, 0, 1, 0)); \ 
        t1 = _mm512_shuffle_i64x2(A0, A1, _MM_SHUFFLE(3, 2, 3, 2)); \ 
        A0 = t0; \ 
        A1 = t1; \ 
    } while((void)0, 0)

#define SWAP_QUARTERS(A0, A1) \ 
    do { \ 
        // 调用 SWAP_HALVES 函数对 A0, A1 进行交换
        SWAP_HALVES(A0, A1); \ 
        // 使用 _mm512_permutexvar_epi64 函数对 A0, A1 进行四分之一位置的交换
        A0 = _mm512_permutexvar_epi64(_mm512_setr_epi64(0, 1, 4, 5, 2, 3, 6, 7), A0); \ 
        A1 = _mm512_permutexvar_epi64(_mm512_setr_epi64(0, 1, 4, 5, 2, 3, 6, 7), A1); \ 
    } while((void)0, 0)

#define UNSWAP_QUARTERS(A0, A1) \ 
    do { \ 
        // 使用 _mm512_permutexvar_epi64 函数对 A0, A1 进行四分之一位置的交换
        A0 = _mm512_permutexvar_epi64(_mm512_setr_epi64(0, 1, 4, 5, 2, 3, 6, 7), A0); \ 
        A1 = _mm512_permutexvar_epi64(_mm512_setr_epi64(0, 1, 4, 5, 2, 3, 6, 7), A1); \ 
        // 调用 SWAP_HALVES 函数对 A0, A1 进行交换
        SWAP_HALVES(A0, A1); \ 
    } while((void)0, 0)

#define BLAKE2_ROUND1(A0, C0, B0, D0, A1, C1, B1, D1) \ 
    do { \ 
        // 调用 SWAP_HALVES 函数对 A0, B0, C0, D0 进行交换
        SWAP_HALVES(A0, B0); \ 
        SWAP_HALVES(C0, D0); \ 
        // 调用 SWAP_HALVES 函数对 A1, B1, C1, D1 进行交换
        SWAP_HALVES(A1, B1); \ 
        SWAP_HALVES(C1, D1); \ 
        // 调用 BLAKE2_ROUND 函数对 A0, B0, C0, D0, A1, B1, C1, D1 进行轮换
        BLAKE2_ROUND(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        // 再次调用 SWAP_HALVES 函数对 A0, B0, C0, D0 进行交换
        SWAP_HALVES(A0, B0); \ 
        SWAP_HALVES(C0, D0); \ 
        // 再次调用 SWAP_HALVES 函数对 A1, B1, C1, D1 进行交换
        SWAP_HALVES(A1, B1); \ 
        SWAP_HALVES(C1, D1); \ 
    } while ((void)0, 0)

#define BLAKE2_ROUND2(A0, A1, B0, B1, C0, C1, D0, D1) \
    # 交换 A0 和 A1 的值
    SWAP_QUARTERS(A0, A1);
    # 交换 B0 和 B1 的值
    SWAP_QUARTERS(B0, B1);
    # 交换 C0 和 C1 的值
    SWAP_QUARTERS(C0, C1);
    # 交换 D0 和 D1 的值
    SWAP_QUARTERS(D0, D1);
    # 进行 BLAKE2 算法的一轮计算
    BLAKE2_ROUND(A0, B0, C0, D0, A1, B1, C1, D1);
    # 恢复 A0 和 A1 的值
    UNSWAP_QUARTERS(A0, A1);
    # 恢复 B0 和 B1 的值
    UNSWAP_QUARTERS(B0, B1);
    # 恢复 C0 和 C1 的值
    UNSWAP_QUARTERS(C0, C1);
    # 恢复 D0 和 D1 的值
    UNSWAP_QUARTERS(D0, D1);
    # 循环执行上述操作，直到条件不满足
    } while ((void)0, 0)
enum {
    ARGON2_VECS_IN_BLOCK = ARGON2_OWORDS_IN_BLOCK / 4,  // 定义枚举类型 ARGON2_VECS_IN_BLOCK，表示每个块中包含的向量数
};

static void fill_block(__m512i *s, const block *ref_block, block *next_block,
                       int with_xor)
{
    __m512i block_XY[ARGON2_VECS_IN_BLOCK];  // 定义一个大小为 ARGON2_VECS_IN_BLOCK 的 __m512i 数组 block_XY
    unsigned int i;  // 定义一个无符号整数 i

    if (with_xor) {  // 如果 with_xor 为真
        for (i = 0; i < ARGON2_VECS_IN_BLOCK; i++) {  // 循环 ARGON2_VECS_IN_BLOCK 次
            s[i] =_mm512_xor_si512(  // 对 s[i] 和 ref_block->v[i] 执行按位异或操作
                s[i], _mm512_loadu_si512((const __m512i *)ref_block->v + i));
            block_XY[i] = _mm512_xor_si512(  // 对 s[i] 和 next_block->v[i] 执行按位异或操作
                s[i], _mm512_loadu_si512((const __m512i *)next_block->v + i));
        }

    } else {  // 如果 with_xor 为假
        for (i = 0; i < ARGON2_VECS_IN_BLOCK; i++) {  // 循环 ARGON2_VECS_IN_BLOCK 次
            block_XY[i] = s[i] =_mm512_xor_si512(  // 对 s[i] 和 ref_block->v[i] 执行按位异或操作
                s[i], _mm512_loadu_si512((const __m512i *)ref_block->v + i));
        }
    }

    for (i = 0; i < 2; ++i) {  // 循环两次
        BLAKE2_ROUND1(  // 调用 BLAKE2_ROUND1 函数
            s[8 * i + 0], s[8 * i + 1], s[8 * i + 2], s[8 * i + 3],
            s[8 * i + 4], s[8 * i + 5], s[8 * i + 6], s[8 * i + 7]);
    }

    for (i = 0; i < 2; ++i) {  // 循环两次
        BLAKE2_ROUND2(  // 调用 BLAKE2_ROUND2 函数
            s[2 * 0 + i], s[2 * 1 + i], s[2 * 2 + i], s[2 * 3 + i],
            s[2 * 4 + i], s[2 * 5 + i], s[2 * 6 + i], s[2 * 7 + i]);
    }

    for (i = 0; i < ARGON2_VECS_IN_BLOCK; i++) {  // 循环 ARGON2_VECS_IN_BLOCK 次
        s[i] = _mm512_xor_si512(s[i], block_XY[i]);  // 对 s[i] 和 block_XY[i] 执行按位异或操作
        _mm512_storeu_si512((__m512i *)next_block->v + i, s[i]);  // 将 s[i] 存储到 next_block->v[i]
    }
}

static void next_addresses(block *address_block, block *input_block)
{
    /*Temporary zero-initialized blocks*/
    __m512i zero_block[ARGON2_VECS_IN_BLOCK];  // 定义一个大小为 ARGON2_VECS_IN_BLOCK 的 __m512i 数组 zero_block
    __m512i zero2_block[ARGON2_VECS_IN_BLOCK];  // 定义一个大小为 ARGON2_VECS_IN_BLOCK 的 __m512i 数组 zero2_block

    memset(zero_block, 0, sizeof(zero_block));  // 将 zero_block 数组清零
    memset(zero2_block, 0, sizeof(zero2_block));  // 将 zero2_block 数组清零

    /*Increasing index counter*/
    input_block->v[6]++;  // 将 input_block->v[6] 的值加一

    /*First iteration of G*/
    fill_block(zero_block, input_block, address_block, 0);  // 调用 fill_block 函数，with_xor 参数为 0

    /*Second iteration of G*/
    fill_block(zero2_block, address_block, address_block, 0);  // 调用 fill_block 函数，with_xor 参数为 0
}

void xmrig_ar2_fill_segment_avx512f(const argon2_instance_t *instance, argon2_position_t position)
    // 定义指向块的指针，用于引用块和当前块
    block *ref_block = NULL, *curr_block = NULL;
    // 定义地址块和输入块
    block address_block, input_block;
    // 伪随机数、引用索引、引用通道
    uint64_t pseudo_rand, ref_index, ref_lane;
    // 前一个偏移量、当前偏移量、起始索引、循环变量
    uint32_t prev_offset, curr_offset;
    // 状态向量数组
    __m512i state[ARGON2_VECS_IN_BLOCK];
    // 数据独立寻址标志
    int data_independent_addressing;

    // 如果实例为空，则返回
    if (instance == NULL) {
        return;
    }

    // 判断是否使用数据独立寻址
    data_independent_addressing = (instance->type == Argon2_i) ||
            (instance->type == Argon2_id && (position.pass == 0) &&
             (position.slice < ARGON2_SYNC_POINTS / 2));

    // 如果使用数据独立寻址，则初始化输入块的值
    if (data_independent_addressing) {
        init_block_value(&input_block, 0);

        input_block.v[0] = position.pass;
        input_block.v[1] = position.lane;
        input_block.v[2] = position.slice;
        input_block.v[3] = instance->memory_blocks;
        input_block.v[4] = instance->passes;
        input_block.v[5] = instance->type;
    }

    // 设置起始索引
    starting_index = 0;

    // 如果是第一个pass和第一个slice，则起始索引为2
    if ((0 == position.pass) && (0 == position.slice)) {
        starting_index = 2; /* we have already generated the first two blocks */

        // 如果使用数据独立寻址，则生成第一个地址块
        if (data_independent_addressing) {
            next_addresses(&address_block, &input_block);
        }
    }

    // 当前块的偏移量
    curr_offset = position.lane * instance->lane_length +
                  position.slice * instance->segment_length + starting_index;

    // 如果当前偏移量是本通道的最后一个块
    if (0 == curr_offset % instance->lane_length) {
        // 上一个块的偏移量为当前偏移量加上通道长度减1
        prev_offset = curr_offset + instance->lane_length - 1;
    } else {
        // 上一个块的偏移量为当前偏移量减1
        prev_offset = curr_offset - 1;
    }

    // 复制上一个块的状态向量到当前状态向量数组
    memcpy(state, ((instance->memory + prev_offset)->v), ARGON2_BLOCK_SIZE);
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
            // 目前还不能引用其他的 lanes
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
# 结束 if-else 语句块
}

# 如果定义了宏，检查 CPU 是否支持 AVX512F 指令集，返回结果
extern int cpu_flags_has_avx512f(void);
int xmrig_ar2_check_avx512f(void) { return cpu_flags_has_avx512f(); }

# 如果没有定义宏，定义一个空函数，用于填充 AVX512F 指令集的段
void xmrig_ar2_fill_segment_avx512f(const argon2_instance_t *instance, argon2_position_t position) {}
# 如果没有定义宏，返回 0，表示 CPU 不支持 AVX512F 指令集
int xmrig_ar2_check_avx512f(void) { return 0; }
```