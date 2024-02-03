# `xmrig\src\3rdparty\argon2\arch\x86_64\lib\argon2-sse2.c`

```cpp
#include "argon2-sse2.h"

#ifdef HAVE_SSE2
#ifdef __GNUC__
#   include <x86intrin.h>
#else
#   include <intrin.h>
#endif

#define ror64_16(x) \
    _mm_shufflehi_epi16( \
        _mm_shufflelo_epi16((x), _MM_SHUFFLE(0, 3, 2, 1)), \
        _MM_SHUFFLE(0, 3, 2, 1))
#define ror64_24(x) \
    _mm_xor_si128(_mm_srli_epi64((x), 24), _mm_slli_epi64((x), 40))
#define ror64_32(x) _mm_shuffle_epi32((x), _MM_SHUFFLE(2, 3, 0, 1))
#define ror64_63(x) \
    _mm_xor_si128(_mm_srli_epi64((x), 63), _mm_add_epi64((x), (x)))

static __m128i f(__m128i x, __m128i y)
{
    // 计算 x 和 y 的无符号 32 位整数乘积
    __m128i z = _mm_mul_epu32(x, y);
    // 返回 x + y + x * y + x * y
    return _mm_add_epi64(_mm_add_epi64(x, y), _mm_add_epi64(z, z));
}

#define G1(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        // 计算 A0 和 B0 的 f 函数值
        A0 = f(A0, B0); \
        // 计算 A1 和 B1 的 f 函数值
        A1 = f(A1, B1); \
\
        // D0 异或 A0
        D0 = _mm_xor_si128(D0, A0); \
        // D1 异或 A1
        D1 = _mm_xor_si128(D1, A1); \
\
        // D0 循环右移 32 位
        D0 = ror64_32(D0); \
        // D1 循环右移 32 位
        D1 = ror64_32(D1); \
\
        // 计算 C0 和 D0 的 f 函数值
        C0 = f(C0, D0); \
        // 计算 C1 和 D1 的 f 函数值
        C1 = f(C1, D1); \
\
        // B0 异或 C0
        B0 = _mm_xor_si128(B0, C0); \
        // B1 异或 C1
        B1 = _mm_xor_si128(B1, C1); \
\
        // B0 循环右移 24 位
        B0 = ror64_24(B0); \
        // B1 循环右移 24 位
        B1 = ror64_24(B1); \
    } while ((void)0, 0)

#define G2(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        // 计算 A0 和 B0 的 f 函数值
        A0 = f(A0, B0); \
        // 计算 A1 和 B1 的 f 函数值
        A1 = f(A1, B1); \
\
        // D0 异或 A0
        D0 = _mm_xor_si128(D0, A0); \
        // D1 异或 A1
        D1 = _mm_xor_si128(D1, A1); \
\
        // D0 循环右移 16 位
        D0 = ror64_16(D0); \
        // D1 循环右移 16 位
        D1 = ror64_16(D1); \
\
        // 计算 C0 和 D0 的 f 函数值
        C0 = f(C0, D0); \
        // 计算 C1 和 D1 的 f 函数值
        C1 = f(C1, D1); \
\
        // B0 异或 C0
        B0 = _mm_xor_si128(B0, C0); \
        // B1 异或 C1
        B1 = _mm_xor_si128(B1, C1); \
\
        // B0 循环右移 63 位
        B0 = ror64_63(B0); \
        // B1 循环右移 63 位
        B1 = ror64_63(B1); \
    } while ((void)0, 0)

#define DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \
    do { \
        // 交错排列 A0, B0, C0, D0
        __m128i t0 = D0; \
        __m128i t1 = B0; \
        D0 = _mm_unpackhi_epi64(D1, _mm_unpacklo_epi64(t0, t0)); \
        D1 = _mm_unpackhi_epi64(t0, _mm_unpacklo_epi64(D1, D1)); \
        B0 = _mm_unpackhi_epi64(B0, _mm_unpacklo_epi64(B1, B1)); \
        B1 = _mm_unpackhi_epi64(B1, _mm_unpacklo_epi64(t1, t1)); \
    # 使用 do-while 循环，条件为无条件执行，直到条件为假
#define UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1) \ 
    do { \ 
        // 交换参数中的数据，将对角线上的元素放回原位
        __m128i t0 = B0; \ 
        __m128i t1 = D0; \ 
        B0 = _mm_unpackhi_epi64(B1, _mm_unpacklo_epi64(B0, B0)); \ 
        B1 = _mm_unpackhi_epi64(t0, _mm_unpacklo_epi64(B1, B1)); \ 
        D0 = _mm_unpackhi_epi64(D0, _mm_unpacklo_epi64(D1, D1)); \ 
        D1 = _mm_unpackhi_epi64(D1, _mm_unpacklo_epi64(t1, t1)); \ 
    } while ((void)0, 0)

#define BLAKE2_ROUND(A0, A1, B0, B1, C0, C1, D0, D1) \ 
    do { \ 
        // 执行 BLAKE2 算法的一轮运算
        G1(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        G2(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        // 对角线化
        DIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \ 
        // 执行 BLAKE2 算法的一轮运算
        G1(A0, B0, C1, D0, A1, B1, C0, D1); \ 
        G2(A0, B0, C1, D0, A1, B1, C0, D1); \ 
        // 取消对角线化
        UNDIAGONALIZE(A0, B0, C0, D0, A1, B1, C1, D1); \ 
    } while ((void)0, 0)

#include "argon2-template-128.h"

void xmrig_ar2_fill_segment_sse2(const argon2_instance_t *instance, argon2_position_t position)
{
    // 调用 fill_segment_128 函数填充段
    fill_segment_128(instance, position);
}

extern int cpu_flags_has_sse2(void);
int xmrig_ar2_check_sse2(void) { return cpu_flags_has_sse2(); }

#else

void xmrig_ar2_fill_segment_sse2(const argon2_instance_t *instance, argon2_position_t position) {}
int xmrig_ar2_check_sse2(void) { return 0; }

#endif
```