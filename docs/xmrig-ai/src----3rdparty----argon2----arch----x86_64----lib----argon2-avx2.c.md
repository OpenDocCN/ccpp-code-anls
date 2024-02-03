# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-avx2.c`

```cpp
#include "argon2-avx2.h"
// 包含 argon2-avx2.h 头文件

#ifdef HAVE_AVX2
#include <string.h>
// 如果支持 AVX2，则包含 string.h 头文件

#ifdef __GNUC__
#   include <x86intrin.h>
#else
#   include <intrin.h>
#endif
// 如果是 GNU 编译器，则包含 x86intrin.h 头文件，否则包含 intrin.h 头文件

#define r16 (_mm256_setr_epi8( \
     2,  3,  4,  5,  6,  7,  0,  1, \
    10, 11, 12, 13, 14, 15,  8,  9, \
    18, 19, 20, 21, 22, 23, 16, 17, \
    26, 27, 28, 29, 30, 31, 24, 25))
// 定义 r16 宏，使用 _mm256_setr_epi8 创建一个 __m256i 类型的变量

#define r24 (_mm256_setr_epi8( \
     3,  4,  5,  6,  7,  0,  1,  2, \
    11, 12, 13, 14, 15,  8,  9, 10, \
    19, 20, 21, 22, 23, 16, 17, 18, \
    27, 28, 29, 30, 31, 24, 25, 26))
// 定义 r24 宏，使用 _mm256_setr_epi8 创建一个 __m256i 类型的变量

#define ror64_16(x) _mm256_shuffle_epi8((x), r16)
// 定义 ror64_16 宏，使用 _mm256_shuffle_epi8 进行位移操作

#define ror64_24(x) _mm256_shuffle_epi8((x), r24)
// 定义 ror64_24 宏，使用 _mm256_shuffle_epi8 进行位移操作

#define ror64_32(x) _mm256_shuffle_epi32((x), _MM_SHUFFLE(2, 3, 0, 1))
// 定义 ror64_32 宏，使用 _mm256_shuffle_epi32 进行位移操作

#define ror64_63(x) \
    _mm256_xor_si256(_mm256_srli_epi64((x), 63), _mm256_add_epi64((x), (x)))
// 定义 ror64_63 宏，使用 _mm256_xor_si256、_mm256_srli_epi64 和 _mm256_add_epi64 进行位移和异或操作

static __m256i f(__m256i x, __m256i y)
{
    __m256i z = _mm256_mul_epu32(x, y);
    return _mm256_add_epi64(_mm256_add_epi64(x, y), _mm256_add_epi64(z, z));
}
// 定义静态函数 f，对两个 __m256i 类型的参数进行乘法和加法操作

#define G1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        A0 = f(A0, B0); \
        A1 = f(A1, B1); \
\
        D0 = _mm256_xor_si256(D0, A0); \
        D1 = _mm256_xor_si256(D1, A1); \
\
        D0 = ror64_32(D0); \
        D1 = ror64_32(D1); \
\
        C0 = f(C0, D0); \
        C1 = f(C1, D1); \
\
        B0 = _mm256_xor_si256(B0, C0); \
        B1 = _mm256_xor_si256(B1, C1); \
\
        B0 = ror64_24(B0); \
        B1 = ror64_24(B1); \
    } while ((void)0, 0)
// 定义 G1 宏，进行一系列的位移、异或和加法操作

#define G2(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        A0 = f(A0, B0); \
        A1 = f(A1, B1); \
\
        D0 = _mm256_xor_si256(D0, A0); \
        D1 = _mm256_xor_si256(D1, A1); \
\
        D0 = ror64_16(D0); \
        D1 = ror64_16(D1); \
\
        C0 = f(C0, D0); \
        C1 = f(C1, D1); \
\
        B0 = _mm256_xor_si256(B0, C0); \
        B1 = _mm256_xor_si256(B1, C1); \
\
        B0 = ror64_63(B0); \
        B1 = ror64_63(B1); \
    } while ((void)0, 0)
// 定义 G2 宏，进行一系列的位移、异或和加法操作

#define DIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1) \
    # 使用_mm256_permute4x64_epi64函数对B0进行按索引排列的操作，得到新的B0
    B0 = _mm256_permute4x64_epi64(B0, _MM_SHUFFLE(0, 3, 2, 1));
    # 使用_mm256_permute4x64_epi64函数对B1进行按索引排列的操作，得到新的B1
    B1 = _mm256_permute4x64_epi64(B1, _MM_SHUFFLE(0, 3, 2, 1));
#define DIAGONALIZE2(A0, B0, C0, D0, A1, B1, C1, D1) \ 
    do { \ 
        // 创建临时变量 tmp1 和 tmp2
        __m256i tmp1, tmp2; \ 
        // 使用 _mm256_blend_epi32 函数对 B0 和 B1 进行混合操作，存储到 tmp1 和 tmp2
        tmp1 = _mm256_blend_epi32(B0, B1, 0xCC); \ 
        tmp2 = _mm256_blend_epi32(B0, B1, 0x33); \ 
        // 使用 _mm256_permute4x64_epi64 函数对 tmp1 和 tmp2 进行排列操作，存储到 B1 和 B0
        B1 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \ 
        B0 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \ 
        // 交换 C0 和 C1 的值
        tmp1 = C0; \ 
        C0 = C1; \ 
        C1 = tmp1; \ 
        // 使用 _mm256_blend_epi32 函数对 D0 和 D1 进行混合操作，存储到 tmp1 和 tmp2
        tmp1 = _mm256_blend_epi32(D0, D1, 0xCC); \ 
        tmp2 = _mm256_blend_epi32(D0, D1, 0x33); \ 
        // 使用 _mm256_permute4x64_epi64 函数对 tmp1 和 tmp2 进行排列操作，存储到 D0 和 D1
        D0 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \ 
        D1 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \ 
    } while ((void)0, 0)

#define UNDIAGONALIZE2(A0, B0, C0, D0, A1, B1, C1, D1) \ 
    do { \ 
        // 创建临时变量 tmp1 和 tmp2
        __m256i tmp1, tmp2; \ 
        // 使用 _mm256_blend_epi32 函数对 B0 和 B1 进行混合操作，存储到 tmp1 和 tmp2
        tmp1 = _mm256_blend_epi32(B0, B1, 0xCC); \ 
        tmp2 = _mm256_blend_epi32(B0, B1, 0x33); \ 
        // 使用 _mm256_permute4x64_epi64 函数对 tmp1 和 tmp2 进行排列操作，存储到 B0 和 B1
        B0 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \ 
        B1 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \ 
        // 交换 C0 和 C1 的值
        tmp1 = C0; \ 
        C0 = C1; \ 
        C1 = tmp1; \ 
// 定义一个宏，用于执行一系列的操作，包括将两个__m256i类型的寄存器按照指定的掩码进行混合
#define DIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        // 使用_mm256_blend_epi32函数将D0和D1按照0xCC的掩码进行混合
        tmp1 = _mm256_blend_epi32(D0, D1, 0xCC); \
        // 使用_mm256_blend_epi32函数将D0和D1按照0x33的掩码进行混合
        tmp2 = _mm256_blend_epi32(D0, D1, 0x33); \
        // 使用_mm256_permute4x64_epi64函数对tmp1进行指定顺序的64位整数元素排列
        D1 = _mm256_permute4x64_epi64(tmp1, _MM_SHUFFLE(2,3,0,1)); \
        // 使用_mm256_permute4x64_epi64函数对tmp2进行指定顺序的64位整数元素排列
        D0 = _mm256_permute4x64_epi64(tmp2, _MM_SHUFFLE(2,3,0,1)); \
    } while ((void)0, 0)

// 定义一个宏，用于执行一系列的操作，包括调用G1和G2函数，然后对寄存器进行对角化和反对角化操作
#define BLAKE2_ROUND1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        // 调用G1和G2函数
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
        // 调用DIAGONALIZE1宏，对寄存器进行对角化操作
        DIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1); \
        // 再次调用G1和G2函数
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
        // 调用UNDIAGONALIZE1宏，对寄存器进行反对角化操作
        UNDIAGONALIZE1(A0, B0, C0, D0, A1, B1, C1, D1); \
    } while ((void)0, 0)

// 定义一个宏，用于执行一系列的操作，包括调用G1和G2函数，然后对寄存器进行对角化和反对角化操作
#define BLAKE2_ROUND2(A0, A1, B0, B1, C0, C1, D0, D1) \
    do { \
        // 调用G1和G2函数
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
        // 调用DIAGONALIZE2宏，对寄存器进行对角化操作
        DIAGONALIZE2(A0, B0, C0, D0, A1, B1, C1, D1); \
        // 再次调用G1和G2函数
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \
        // 调用UNDIAGONALIZE2宏，对寄存器进行反对角化操作
        UNDIAGONALIZE2(A0, B0, C0, D0, A1, B1, C1, D1); \
    } while ((void)0, 0)

// 定义一个枚举类型，表示ARGON2_HWORDS_IN_BLOCK的值为ARGON2_OWORDS_IN_BLOCK除以2
enum {
    ARGON2_HWORDS_IN_BLOCK = ARGON2_OWORDS_IN_BLOCK / 2,
};

// 定义一个静态函数，用于填充块
static void fill_block(__m256i *s, const block *ref_block, block *next_block,
                       int with_xor)
{
    // 定义一个__m256i类型的数组，用于存储块的一半数据
    __m256i block_XY[ARGON2_HWORDS_IN_BLOCK];
    // 定义一个无符号整型变量i
    unsigned int i;

    // 如果with_xor为真
    if (with_xor) {
        // 对每个块的一半数据进行异或操作
        for (i = 0; i < ARGON2_HWORDS_IN_BLOCK; i++) {
            s[i] =_mm256_xor_si256(
                s[i], _mm256_loadu_si256((const __m256i *)ref_block->v + i));
            block_XY[i] = _mm256_xor_si256(
                s[i], _mm256_loadu_si256((const __m256i *)next_block->v + i));
        }

    } else {
        // 否则，对每个块的一半数据进行异或操作
        for (i = 0; i < ARGON2_HWORDS_IN_BLOCK; i++) {
            block_XY[i] = s[i] =_mm256_xor_si256(
                s[i], _mm256_loadu_si256((const __m256i *)ref_block->v + i));
        }
    }
}
    # 对于每个 i 在 0 到 3 之间的循环
    for (i = 0; i < 4; ++i) {
        # 调用 BLAKE2_ROUND1 函数，对应参数为 s 数组中的一组元素
        BLAKE2_ROUND1(
            s[8 * i + 0], s[8 * i + 1], s[8 * i + 2], s[8 * i + 3],
            s[8 * i + 4], s[8 * i + 5], s[8 * i + 6], s[8 * i + 7]);
    }

    # 对于每个 i 在 0 到 3 之间的循环
    for (i = 0; i < 4; ++i) {
        # 调用 BLAKE2_ROUND2 函数，对应参数为 s 数组中的一组元素
        BLAKE2_ROUND2(
            s[4 * 0 + i], s[4 * 1 + i], s[4 * 2 + i], s[4 * 3 + i],
            s[4 * 4 + i], s[4 * 5 + i], s[4 * 6 + i], s[4 * 7 + i]);
    }

    # 对于每个 i 在 0 到 ARGON2_HWORDS_IN_BLOCK 之间的循环
    for (i = 0; i < ARGON2_HWORDS_IN_BLOCK; i++) {
        # 将 s 数组中的元素与 block_XY 数组中的元素进行按位异或操作
        s[i] = _mm256_xor_si256(s[i], block_XY[i]);
        # 将结果存储到 next_block->v 数组中
        _mm256_storeu_si256((__m256i *)next_block->v + i, s[i]);
    }
}
static void next_addresses(block *address_block, block *input_block)
{
    /* 创建临时的零初始化块 */
    __m256i zero_block[ARGON2_HWORDS_IN_BLOCK];
    __m256i zero2_block[ARGON2_HWORDS_IN_BLOCK];

    // 使用零值填充 zero_block 和 zero2_block
    memset(zero_block, 0, sizeof(zero_block));
    memset(zero2_block, 0, sizeof(zero2_block));

    // 增加索引计数器
    input_block->v[6]++;

    // G 的第一次迭代
    fill_block(zero_block, input_block, address_block, 0);

    // G 的第二次迭代
    fill_block(zero2_block, address_block, address_block, 0);
}

void xmrig_ar2_fill_segment_avx2(const argon2_instance_t *instance, argon2_position_t position)
{
    block *ref_block = NULL, *curr_block = NULL;
    block address_block, input_block;
    uint64_t pseudo_rand, ref_index, ref_lane;
    uint32_t prev_offset, curr_offset;
    uint32_t starting_index, i;
    __m256i state[ARGON2_HWORDS_IN_BLOCK];
    int data_independent_addressing;

    if (instance == NULL) {
        return;
    }

    // 检查是否为数据独立寻址
    data_independent_addressing = (instance->type == Argon2_i) ||
            (instance->type == Argon2_id && (position.pass == 0) &&
             (position.slice < ARGON2_SYNC_POINTS / 2));

    if (data_independent_addressing) {
        // 初始化 input_block 的值为零
        init_block_value(&input_block, 0);

        // 设置 input_block 的值
        input_block.v[0] = position.pass;
        input_block.v[1] = position.lane;
        input_block.v[2] = position.slice;
        input_block.v[3] = instance->memory_blocks;
        input_block.v[4] = instance->passes;
        input_block.v[5] = instance->type;
    }

    starting_index = 0;

    if ((0 == position.pass) && (0 == position.slice)) {
        starting_index = 2; /* 已经生成了前两个块 */

        // 如果是数据独立寻址，生成第一个地址块
        if (data_independent_addressing) {
            next_addresses(&address_block, &input_block);
        }
    }

    // 当前块的偏移量
    # 计算当前偏移量，根据位置的车道、切片和起始索引计算得出
    curr_offset = position.lane * instance->lane_length + position.slice * instance->segment_length + starting_index;

    # 如果当前偏移量是车道长度的整数倍，表示这是该车道的最后一个块
    if (0 == curr_offset % instance->lane_length) {
        # 上一个块的偏移量是当前偏移量加上车道长度减去1
        prev_offset = curr_offset + instance->lane_length - 1;
    } else {
        # 否则，上一个块的偏移量是当前偏移量减去1
        prev_offset = curr_offset - 1;
    }

    # 从上一个块的内存中复制数据到状态中
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
# 结束当前的代码块
}

# 如果定义了宏，检查 CPU 是否支持 AVX2 指令集
extern int cpu_flags_has_avx2(void);

# 如果定义了宏，检查 CPU 是否支持 AVX2 指令集，返回结果
int xmrig_ar2_check_avx2(void) { return cpu_flags_has_avx2(); }

# 如果没有定义宏，定义一个空函数，用于填充 AVX2 段
void xmrig_ar2_fill_segment_avx2(const argon2_instance_t *instance, argon2_position_t position) {}

# 如果没有定义宏，检查 CPU 是否支持 AVX2 指令集，返回 0
int xmrig_ar2_check_avx2(void) { return 0; }
```