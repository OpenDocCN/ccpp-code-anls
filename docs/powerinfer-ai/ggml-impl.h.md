# `PowerInfer\ggml-impl.h`

```
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

// 16-bit float
// on Arm, we use __fp16
// on x86, we use uint16_t
#if defined(__ARM_NEON) && !defined(_MSC_VER)

// if YCM cannot find <arm_neon.h>, make a symbolic link to it, for example:
//
//   $ ln -sfn /Library/Developer/CommandLineTools/usr/lib/clang/13.1.6/include/arm_neon.h ./src/
//
#include <arm_neon.h>

#define GGML_COMPUTE_FP16_TO_FP32(x) ((float) (x))
#define GGML_COMPUTE_FP32_TO_FP16(x) (x)

#define GGML_FP16_TO_FP32(x) ((float) (x))
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
// 定义将 FP16 转换为 FP32 的宏，使用 SIMD 指令进行转换
#define GGML_COMPUTE_FP16_TO_FP32(x) _mm_cvtss_f32(_mm_cvtph_ps(_mm_cvtsi32_si128(x)))
// 定义将 FP32 转换为 FP16 的宏，使用 SIMD 指令进行转换
#define GGML_COMPUTE_FP32_TO_FP16(x) _mm_extract_epi16(_mm_cvtps_ph(_mm_set_ss(x), 0)
#else
// 如果不支持 SIMD 指令，则使用默认的转换方法
#define GGML_COMPUTE_FP16_TO_FP32(x) _cvtsh_ss(x)
#define GGML_COMPUTE_FP32_TO_FP16(x) _cvtss_sh(x, 0)
#endif

#elif defined(__POWER9_VECTOR__)

// 如果支持 Power9 向量指令，则使用自定义的转换函数
#define GGML_COMPUTE_FP16_TO_FP32(x) ggml_compute_fp16_to_fp32(x)
#define GGML_COMPUTE_FP32_TO_FP16(x) ggml_compute_fp32_to_fp16(x)
// 使用内联汇编实现的 FP16 到 FP32 的转换函数
#define GGML_FP16_TO_FP32(x) GGML_COMPUTE_FP16_TO_FP32(x)
// 使用内联汇编实现的 FP32 到 FP16 的转换函数
#define GGML_FP32_TO_FP16(x) GGML_COMPUTE_FP32_TO_FP16(x)

// 使用内联汇编实现的将 FP16 转换为 FP32 的函数
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

// 使用内联汇编实现的将 FP32 转换为 FP16 的函数
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

// 如果不支持 SIMD 指令和 Power9 向量指令，则使用标准的 FP16 <-> FP32 转换方法
// 参考：https://github.com/Maratyszcza/FP16

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

// 使用内联汇编实现的将 FP16 转换为 FP32 的函数
static inline float ggml_compute_fp16_to_fp32(ggml_fp16_t h) {
    const uint32_t w = (uint32_t) h << 16;
    const uint32_t sign = w & UINT32_C(0x80000000);
    const uint32_t two_w = w + w;

    const uint32_t exp_offset = UINT32_C(0xE0) << 23;
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    // 如果满足条件，定义指数缩放值为 0x1.0p-112f
    const float exp_scale = 0x1.0p-112f;
#else
    // 如果不满足条件，定义指数缩放值为通过位操作得到的值
    const float exp_scale = fp32_from_bits(UINT32_C(0x7800000));
#endif
    // 计算规格化值
    const float normalized_value = fp32_from_bits((two_w >> 4) + exp_offset) * exp_scale;

    // 定义魔数掩码
    const uint32_t magic_mask = UINT32_C(126) << 23;
    // 定义魔数偏置
    const float magic_bias = 0.5f;
    // 计算非规格化值
    const float denormalized_value = fp32_from_bits((two_w >> 17) | magic_mask) - magic_bias;

    // 定义非规格化截断值
    const uint32_t denormalized_cutoff = UINT32_C(1) << 27;
    // 计算结果
    const uint32_t result = sign |
        (two_w < denormalized_cutoff ? fp32_to_bits(denormalized_value) : fp32_to_bits(normalized_value));
    // 返回结果
    return fp32_from_bits(result);
}

static inline ggml_fp16_t ggml_compute_fp32_to_fp16(float f) {
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    // 如果满足条件，定义缩放到无穷大的值
    const float scale_to_inf = 0x1.0p+112f;
    // 如果满足条件，定义缩放到零的值
    const float scale_to_zero = 0x1.0p-110f;
#else
    // 如果不满足条件，定义缩放到无穷大的值为通过位操作得到的值
    const float scale_to_inf = fp32_from_bits(UINT32_C(0x77800000));
    // 如果不满足条件，定义缩放到零的值为通过位操作得到的值
    const float scale_to_zero = fp32_from_bits(UINT32_C(0x08800000));
#endif
    // 计算基础值
    float base = (fabsf(f) * scale_to_inf) * scale_to_zero;

    // 将浮点数转换为 32 位无符号整数
    const uint32_t w = fp32_to_bits(f);
    // 左移一位
    const uint32_t shl1_w = w + w;
    // 提取符号位
    const uint32_t sign = w & UINT32_C(0x80000000);
    // 计算偏置
    uint32_t bias = shl1_w & UINT32_C(0xFF000000);
    if (bias < UINT32_C(0x71000000)) {
        bias = UINT32_C(0x71000000);
    }

    // 计算基础值
    base = fp32_from_bits((bias >> 1) + UINT32_C(0x07800000)) + base;
    // 将基础值转换为 32 位无符号整数
    const uint32_t bits = fp32_to_bits(base);
    // 提取指数部分
    const uint32_t exp_bits = (bits >> 13) & UINT32_C(0x00007C00);
    // 提取尾数部分
    const uint32_t mantissa_bits = bits & UINT32_C(0x00000FFF);
    // 计算非符号部分
    const uint32_t nonsign = exp_bits + mantissa_bits;
    // 返回结果
    return (sign >> 16) | (shl1_w > UINT32_C(0xFF000000) ? UINT16_C(0x7E00) : nonsign);
}

// 定义宏，用于计算从 16 位浮点数到 32 位浮点数的转换
#define GGML_COMPUTE_FP16_TO_FP32(x) ggml_compute_fp16_to_fp32(x)
// 定义将 float 转换为 half precision float 的宏，参数为 x
#define GGML_COMPUTE_FP32_TO_FP16(x) ggml_compute_fp32_to_fp16(x)

#endif // __F16C__

#endif // __ARM_NEON

// 预先计算的 f32 到 f16 的转换表 (256 KB)
// 在 ggml.c 中定义，在 ggml_init() 中初始化
extern float ggml_table_f32_f16[1 << 16];

// 在 ARM NEON 上，直接将 x 转换为 x 比调用 ggml_lookup_fp16_to_fp32 更快，
// 因此我们在 NEON 上为 GGML_FP16_TO_FP32 和 GGML_FP32_TO_FP16 定义了其他地方
// 对于 POWER9 也是如此
#if !defined(GGML_FP16_TO_FP32) || !defined(GGML_FP32_TO_FP16)

// 内联函数，将 ggml_fp16_t 转换为 float
inline static float ggml_lookup_fp16_to_fp32(ggml_fp16_t f) {
    uint16_t s;
    memcpy(&s, &f, sizeof(uint16_t));
    return ggml_table_f32_f16[s];
}

// 定义将 ggml_fp16_t 转换为 float 的宏，参数为 x
#define GGML_FP16_TO_FP32(x) ggml_lookup_fp16_to_fp32(x)
// 定义将 float 转换为 ggml_fp16_t 的宏，参数为 x
#define GGML_FP32_TO_FP16(x) GGML_COMPUTE_FP32_TO_FP16(x)

#endif

// 定义哈希表已满的标志
#define GGML_HASHTABLE_FULL ((size_t)-1)
// 定义哈希表中已存在的标志
#define GGML_HASHTABLE_ALREADY_EXISTS ((size_t)-2)

// 检查哈希表中是否包含指定 key
bool   ggml_hash_contains      (const struct ggml_hash_set hash_set, struct ggml_tensor * key);

// 如果表已满，则返回 GGML_HASHTABLE_FULL，否则返回 key 的当前索引或应该插入的位置
size_t ggml_hash_find          (const struct ggml_hash_set hash_set, struct ggml_tensor * key);

// 如果 key 已经存在，则返回 GGML_HASHTABLE_ALREADY_EXISTS，否则返回索引，如果表已满则断言
size_t ggml_hash_insert        (      struct ggml_hash_set hash_set, struct ggml_tensor * key);

// 返回索引，如果表已满则断言
size_t ggml_hash_find_or_insert(      struct ggml_hash_set hash_set, struct ggml_tensor * key);

#ifdef __cplusplus
}
#endif
```