# `whisper.cpp\bindings\ruby\ext\ggml-impl.h`

```cpp
#pragma once

#include "ggml.h"

// GGML internal header

#include <assert.h>
#include <stddef.h>
#include <stdbool.h>
#include <string.h> // memcpy
#include <math.h>   // fabsf

#ifdef __cplusplus
extern "C" {
#endif

// static_assert should be a #define, but if it's not,
// fall back to the _Static_assert C11 keyword.
// if C99 - static_assert is noop
// ref: https://stackoverflow.com/a/53923785/4039976
#ifndef static_assert
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 201100L)
#define static_assert(cond, msg) _Static_assert(cond, msg)
#else
#define static_assert(cond, msg) struct global_scope_noop_trick
#endif
#endif

// __FMA__ and __F16C__ are not defined in MSVC, however they are implied with AVX2/AVX512
#if defined(_MSC_VER) && (defined(__AVX2__) || defined(__AVX512F__))
#ifndef __FMA__
#define __FMA__
#endif
#ifndef __F16C__
#define __F16C__
#endif
#ifndef __SSE3__
#define __SSE3__
#endif
#endif

#undef MIN
#undef MAX

// 定义宏 MIN，返回两个值中较小的一个
#define MIN(a, b) ((a) < (b) ? (a) : (b))
// 定义宏 MAX，返回两个值中较大的一个
#define MAX(a, b) ((a) > (b) ? (a) : (b))

// 16-bit float
// on Arm, we use __fp16
// on x86, we use uint16_t
#if defined(__ARM_NEON) && !defined(_MSC_VER)

// if YCM cannot find <arm_neon.h>, make a symbolic link to it, for example:
//
//   $ ln -sfn /Library/Developer/CommandLineTools/usr/lib/clang/13.1.6/include/arm_neon.h ./src/
//
#include <arm_neon.h>

// 定义宏 GGML_COMPUTE_FP16_TO_FP32，将16位浮点数转换为32位浮点数
#define GGML_COMPUTE_FP16_TO_FP32(x) ((float) (x))
// 定义宏 GGML_COMPUTE_FP32_TO_FP16，将32位浮点数转换为16位浮点数
#define GGML_COMPUTE_FP32_TO_FP16(x) (x)

// 定义宏 GGML_FP16_TO_FP32，将16位浮点数转换为32位浮点数
#define GGML_FP16_TO_FP32(x) ((float) (x))
// 定义宏 GGML_FP32_TO_FP16，将32位浮点数转换为16位浮点数
#define GGML_FP32_TO_FP16(x) (x)

#else

#ifdef __wasm_simd128__
#include <wasm_simd128.h>
#else
#ifdef __POWER9_VECTOR__
#include <altivec.h>
#undef bool
#define bool _Bool
#else
#if defined(_MSC_VER) || defined(__MINGW32__)
#include <intrin.h>
#else
#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__) || defined(__SSE3__)
#if !defined(__riscv)
#include <immintrin.h>
#endif
#endif
#endif
#endif
#endif

#ifdef __riscv_v_intrinsic
#include <riscv_vector.h>
#endif
#ifdef __F16C__

#ifdef _MSC_VER
#define GGML_COMPUTE_FP16_TO_FP32(x) _mm_cvtss_f32(_mm_cvtph_ps(_mm_cvtsi32_si128(x)))
#define GGML_COMPUTE_FP32_TO_FP16(x) _mm_extract_epi16(_mm_cvtps_ph(_mm_set_ss(x), 0)
#else
#define GGML_COMPUTE_FP16_TO_FP32(x) _cvtsh_ss(x)
#define GGML_COMPUTE_FP32_TO_FP16(x) _cvtss_sh(x, 0)
#endif

#elif defined(__POWER9_VECTOR__)

#define GGML_COMPUTE_FP16_TO_FP32(x) ggml_compute_fp16_to_fp32(x)
#define GGML_COMPUTE_FP32_TO_FP16(x) ggml_compute_fp32_to_fp16(x)
/* the inline asm below is about 12% faster than the lookup method */
#define GGML_FP16_TO_FP32(x) GGML_COMPUTE_FP16_TO_FP32(x)
#define GGML_FP32_TO_FP16(x) GGML_COMPUTE_FP32_TO_FP16(x)

// 定义将 FP16 转换为 FP32 的宏，根据不同的编译器选择不同的实现方式
static inline float ggml_compute_fp16_to_fp32(ggml_fp16_t h) {
    register float f;
    register double d;
    __asm__(
        "mtfprd %0,%2\n"
        "xscvhpdp %0,%0\n"
        "frsp %1,%0\n" :
        /* temp */ "=d"(d),
        /* out */  "=f"(f):
        /* in */   "r"(h));
    return f;
}

// 定义将 FP32 转换为 FP16 的宏，根据不同的编译器选择不同的实现方式
static inline ggml_fp16_t ggml_compute_fp32_to_fp16(float f) {
    register double d;
    register ggml_fp16_t r;
    __asm__( /* xscvdphp can work on double or single precision */
        "xscvdphp %0,%2\n"
        "mffprd %1,%0\n" :
        /* temp */ "=d"(d),
        /* out */  "=r"(r):
        /* in */   "f"(f));
    return r;
}

#else

// FP16 <-> FP32
// ref: https://github.com/Maratyszcza/FP16

// 将 32 位整数转换为单精度浮点数
static inline float fp32_from_bits(uint32_t w) {
    union {
        uint32_t as_bits;
        float as_value;
    } fp32;
    fp32.as_bits = w;
    return fp32.as_value;
}

// 将单精度浮点数转换为 32 位整数
static inline uint32_t fp32_to_bits(float f) {
    union {
        float as_value;
        uint32_t as_bits;
    } fp32;
    fp32.as_value = f;
    return fp32.as_bits;
}

// 将 FP16 转换为 FP32
static inline float ggml_compute_fp16_to_fp32(ggml_fp16_t h) {
    const uint32_t w = (uint32_t) h << 16;
    const uint32_t sign = w & UINT32_C(0x80000000);
    const uint32_t two_w = w + w;

    const uint32_t exp_offset = UINT32_C(0xE0) << 23;
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    // 如果满足条件，定义指数缩放值为 0x1.0p-112f
    const float exp_scale = 0x1.0p-112f;
#else
    // 如果不满足条件，定义指数缩放值为指定的浮点数
    const float exp_scale = fp32_from_bits(UINT32_C(0x7800000));
#endif
    // 计算归一化值
    const float normalized_value = fp32_from_bits((two_w >> 4) + exp_offset) * exp_scale;

    // 定义魔数掩码
    const uint32_t magic_mask = UINT32_C(126) << 23;
    // 定义魔数偏置
    const float magic_bias = 0.5f;
    // 计算非规格化值
    const float denormalized_value = fp32_from_bits((two_w >> 17) | magic_mask) - magic_bias;

    // 定义非规格化截断值
    const uint32_t denormalized_cutoff = UINT32_C(1) << 27;
    // 计算结果值
    const uint32_t result = sign |
        (two_w < denormalized_cutoff ? fp32_to_bits(denormalized_value) : fp32_to_bits(normalized_value));
    // 返回结果值
    return fp32_from_bits(result);
}

// 计算单精度浮点数到半精度浮点数的转换
static inline ggml_fp16_t ggml_compute_fp32_to_fp16(float f) {
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    // 定义缩放到无穷大的值
    const float scale_to_inf = 0x1.0p+112f;
    // 定义缩放到零的值
    const float scale_to_zero = 0x1.0p-110f;
#else
    // 如果不满足条件，定义缩放到无穷大和零的值为指定的浮点数
    const float scale_to_inf = fp32_from_bits(UINT32_C(0x77800000));
    const float scale_to_zero = fp32_from_bits(UINT32_C(0x08800000));
#endif
    // 计算基础值
    float base = (fabsf(f) * scale_to_inf) * scale_to_zero;

    // 获取单精度浮点数的位表示
    const uint32_t w = fp32_to_bits(f);
    // 左移一位的单精度浮点数位表示
    const uint32_t shl1_w = w + w;
    // 获取符号位
    const uint32_t sign = w & UINT32_C(0x80000000);
    // 计算偏置值
    uint32_t bias = shl1_w & UINT32_C(0xFF000000);
    if (bias < UINT32_C(0x71000000)) {
        bias = UINT32_C(0x71000000);
    }

    // 更新基础值
    base = fp32_from_bits((bias >> 1) + UINT32_C(0x07800000)) + base;
    // 获取基础值的位表示
    const uint32_t bits = fp32_to_bits(base);
    // 获取指数位和尾数位
    const uint32_t exp_bits = (bits >> 13) & UINT32_C(0x00007C00);
    const uint32_t mantissa_bits = bits & UINT32_C(0x00000FFF);
    // 计算非符号位
    const uint32_t nonsign = exp_bits + mantissa_bits;
    // 返回结果值
    return (sign >> 16) | (shl1_w > UINT32_C(0xFF000000) ? UINT16_C(0x7E00) : nonsign);
}

// 定义宏，���于计算半精度浮点数到单精度浮点数的转换
#define GGML_COMPUTE_FP16_TO_FP32(x) ggml_compute_fp16_to_fp32(x)
// 定义将 float 转换为 half precision float 的宏，调用 ggml_compute_fp32_to_fp16 函数
#define GGML_COMPUTE_FP32_TO_FP16(x) ggml_compute_fp32_to_fp16(x)

#endif // __F16C__

#endif // __ARM_NEON

// 预先计算的 f32 到 f16 的转换表（256 KB）
// 在 ggml.c 中定义，在 ggml_init() 中初始化
extern float ggml_table_f32_f16[1 << 16];

// 在 ARM NEON 中，直接将 x 转换为 x 比调用 ggml_lookup_fp16_to_fp32 更快
// 因此我们在 NEON 中为 GGML_FP16_TO_FP32 和 GGML_FP32_TO_FP16 定义了其他内容
// 对于 POWER9 也是如此
#if !defined(GGML_FP16_TO_FP32) || !defined(GGML_FP32_TO_FP16)

// 将 ggml_fp16_t 转换为 float 的内联函数
inline static float ggml_lookup_fp16_to_fp32(ggml_fp16_t f) {
    uint16_t s;
    memcpy(&s, &f, sizeof(uint16_t));
    return ggml_table_f32_f16[s];
}

// 定义将 ggml_fp16_t 转换为 float 的宏
#define GGML_FP16_TO_FP32(x) ggml_lookup_fp16_to_fp32(x)
// 定义将 float 转换为 ggml_fp16_t 的宏
#define GGML_FP32_TO_FP16(x) GGML_COMPUTE_FP32_TO_FP16(x)

#endif

// 定义哈希表已满的标志
#define GGML_HASHTABLE_FULL ((size_t)-1)
// 定义哈希表中已存在的键的标志
#define GGML_HASHTABLE_ALREADY_EXISTS ((size_t)-2)

// 检查哈希表中是否包含指定键
bool ggml_hash_contains(const struct ggml_hash_set hash_set, struct ggml_tensor *key);

// 在哈希表中查找指定键，返回键的当前索引或应插入的位置
size_t ggml_hash_find(const struct ggml_hash_set hash_set, struct ggml_tensor *key);

// 在哈希表中插入指定键，如果键已存在则返回 GGML_HASHTABLE_ALREADY_EXISTS，否则返回当前索引
size_t ggml_hash_insert(struct ggml_hash_set hash_set, struct ggml_tensor *key);

// 在哈希表中查找或插入指定键，如果键已存在则返回当前索引，否则插入并返回当前索引
size_t ggml_hash_find_or_insert(struct ggml_hash_set hash_set, struct ggml_tensor *key);

#ifdef __cplusplus
}
#endif
```