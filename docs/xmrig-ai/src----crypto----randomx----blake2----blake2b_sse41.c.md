# `xmrig\src\crypto\randomx\blake2\blake2b_sse41.c`

```
/*
 * 版权声明，声明了代码的版权和使用条件
 * 版权所有
 * 在源代码和二进制形式的再分发和使用，无论是否经过修改，都需要满足以下条件：
 *    * 源代码的再分发必须保留上述版权声明、条件列表和以下免责声明
 *    * 二进制形式的再分发必须在提供的文档和/或其他材料中复制上述版权声明、条件列表和以下免责声明
 *    * 未经特定事先书面许可，不得使用版权持有者或其贡献者的名称来认可或推广从本软件派生的产品
 * 本软件由版权持有者和贡献者按原样提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。无论在任何情况下，版权持有者或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失或业务中断）负责，无论是在合同、严格责任或侵权行为（包括疏忽或其他方式）的任何理论下，即使已被告知可能发生此类损害。
*/

/* 原始代码来自Argon2参考源代码包，使用CC0许可证
 * https://github.com/P-H-C/phc-winner-argon2
 * 版权所有 2015
 * Daniel Dinu, Dmitry Khovratovich, Jean-Philippe Aumasson, and Samuel Neves
*/

#if defined(_M_X64) || defined(__x86_64__)
#include <stdint.h>
#include <string.h>
#include <stdio.h>

#include "crypto/randomx/blake2/blake2.h"

#ifdef _MSC_VER
#include <intrin.h>
#endif

#include <smmintrin.h>
#include "blake2b-round.h"


extern const uint64_t blake2b_IV[8];


static const uint8_t blake2b_sigma_sse41[12][16] = {
    {0, 2, 4, 6, 1, 3, 5, 7, 8, 10, 12, 14, 9, 11, 13, 15},
    {14, 4, 9, 13, 10, 8, 15, 6, 1, 0, 11, 5, 12, 2, 7, 3},
    {11, 12, 5, 15, 8, 0, 2, 13, 10, 3, 7, 9, 14, 6, 1, 4},
    {7, 3, 13, 11, 9, 1, 12, 14, 2, 5, 4, 15, 6, 10, 0, 8},
    {9, 5, 2, 10, 0, 7, 4, 15, 14, 11, 6, 3, 1, 12, 8, 13},
    {2, 6, 0, 8, 12, 10, 11, 3, 4, 7, 15, 1, 13, 5, 14, 9},
    {12, 1, 14, 4, 5, 15, 13, 10, 0, 6, 9, 8, 7, 3, 2, 11},
    {13, 7, 12, 3, 11, 14, 1, 9, 5, 15, 8, 2, 0, 4, 6, 10},
    {6, 14, 11, 0, 15, 9, 3, 8, 12, 13, 1, 10, 2, 7, 4, 5},
    {10, 8, 7, 1, 2, 4, 6, 5, 15, 9, 3, 13, 11, 14, 12, 0},
    {0, 2, 4, 6, 1, 3, 5, 7, 8, 10, 12, 14, 9, 11, 13, 15},
    {14, 4, 9, 13, 10, 8, 15, 6, 1, 0, 11, 5, 12, 2, 7, 3},
};


void rx_blake2b_compress_sse41(blake2b_state* S, const uint8_t *block)
{
    __m128i row1l, row1h;
    __m128i row2l, row2h;
    __m128i row3l, row3h;
    __m128i row4l, row4h;
    __m128i b0, b1;
    __m128i t0, t1;

    const __m128i r16 = _mm_setr_epi8(2, 3, 4, 5, 6, 7, 0, 1, 10, 11, 12, 13, 14, 15, 8, 9);
    const __m128i r24 = _mm_setr_epi8(3, 4, 5, 6, 7, 0, 1, 2, 11, 12, 13, 14, 15, 8, 9, 10);

    // 加载当前状态的哈希值
    row1l = LOADU(&S->h[0]);
    row1h = LOADU(&S->h[2]);
    row2l = LOADU(&S->h[4]);
    row2h = LOADU(&S->h[6]);
    // 加载初始向量
    row3l = LOADU(&blake2b_IV[0]);
    row3h = LOADU(&blake2b_IV[2]);
    // 计算并加载初始向量与当前状态的 t 和 f 的异或值
    row4l = _mm_xor_si128(LOADU(&blake2b_IV[4]), LOADU(&S->t[0]));
    row4h = _mm_xor_si128(LOADU(&blake2b_IV[6]), LOADU(&S->f[0]));

    const uint64_t* m = (const uint64_t*)(block);

    for (uint32_t r = 0; r < 12; ++r) {
        ROUND(r);  // 执行 BLAKE2b 压缩函数的一个轮次
    }

    // 计算并加载当前状态的哈希值与初始向量的异或值
    row1l = _mm_xor_si128(row3l, row1l);
    row1h = _mm_xor_si128(row3h, row1h);
    // 将结果存储到当前状态的哈希值中
    STOREU(&S->h[0], _mm_xor_si128(LOADU(&S->h[0]), row1l));
}
    # 将row1h与S->h[2]进行异或操作，并将结果存储到S->h[2]中
    STOREU(&S->h[2], _mm_xor_si128(LOADU(&S->h[2]), row1h));
    
    # 将row4l与row2l进行异或操作，并将结果存储到row2l中
    row2l = _mm_xor_si128(row4l, row2l);
    
    # 将row4h与row2h进行异或操作，并将结果存储到row2h中
    row2h = _mm_xor_si128(row4h, row2h);
    
    # 将row2l与S->h[4]进行异或操作，并将结果存储到S->h[4]中
    STOREU(&S->h[4], _mm_xor_si128(LOADU(&S->h[4]), row2l));
    
    # 将row2h与S->h[6]进行异或操作，并将结果存储到S->h[6]中
    STOREU(&S->h[6], _mm_xor_si128(LOADU(&S->h[6]), row2h));
}
#endif

这部分代码是C/C++中的条件编译语句，用于结束一个条件编译的代码块。
```