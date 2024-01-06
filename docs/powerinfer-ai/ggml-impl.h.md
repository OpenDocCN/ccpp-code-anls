# `PowerInfer\ggml-impl.h`

```
#pragma once
// 防止头文件被多次包含

#include "ggml.h"
// 包含自定义的头文件 ggml.h

// GGML internal header
// GGML 内部头文件

#include <assert.h>
// 包含断言库
#include <stddef.h>
// 包含标准库定义
#include <stdbool.h>
// 包含布尔类型定义
#include <string.h> // memcpy
// 包含字符串操作函数 memcpy
#include <math.h>   // fabsf
// 包含数学函数 fabsf

#ifdef __cplusplus
extern "C" {
#endif
// 如果是 C++ 环境，使用 C 语言的方式进行编译

// static_assert should be a #define, but if it's not,
// fall back to the _Static_assert C11 keyword.
// if C99 - static_assert is noop
// ref: https://stackoverflow.com/a/53923785/4039976
// 如果 static_assert 不是一个 #define，就使用 C11 的 _Static_assert 关键字
// 如果 static_assert 宏未定义，则根据 C 标准版本定义 static_assert 宏
#ifndef static_assert
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 201100L)
#define static_assert(cond, msg) _Static_assert(cond, msg)
#else
#define static_assert(cond, msg) struct global_scope_noop_trick
#endif
#endif

// 如果在 MSVC 编译器下，并且定义了 AVX2 或 AVX512F，则定义 __FMA__ 和 __F16C__ 宏
#if defined(_MSC_VER) && (defined(__AVX2__) || defined(__AVX512F__))
#ifndef __FMA__
#define __FMA__
#endif
#ifndef __F16C__
#define __F16C__
#endif
// 如果未定义 __SSE3__ 宏，则定义 __SSE3__ 宏
#ifndef __SSE3__
#define __SSE3__
#endif
#endif
// 如果是 ARM 架构并且不是在 Visual Studio 编译环境下
// 在 Arm 架构上，我们使用 __fp16
// 在 x86 架构上，我们使用 uint16_t
#if defined(__ARM_NEON) && !defined(_MSC_VER)

// 如果 YCM 找不到 <arm_neon.h>，可以创建一个符号链接指向它，例如：
//
//   $ ln -sfn /Library/Developer/CommandLineTools/usr/lib/clang/13.1.6/include/arm_neon.h ./src/
//
#include <arm_neon.h>

// 定义将 16 位浮点数转换为 32 位浮点数的宏
#define GGML_COMPUTE_FP16_TO_FP32(x) ((float) (x))
#define GGML_COMPUTE_FP32_TO_FP16(x) (x)

// 定义将 16 位浮点数转换为 32 位浮点数的宏
#define GGML_FP16_TO_FP32(x) ((float) (x))
#define GGML_FP32_TO_FP16(x) (x)

#else
#ifdef __wasm_simd128__
// 如果支持 WebAssembly SIMD 128，则包含对应的头文件
#include <wasm_simd128.h>
#else
#ifdef __POWER9_VECTOR__
// 如果支持 Power9 向量指令集，则包含对应的头文件，并重新定义 bool 类型
#include <altivec.h>
#undef bool
#define bool _Bool
#else
#if defined(_MSC_VER) || defined(__MINGW32__)
// 如果是在 Windows 平台下编译，则包含对应的头文件
#include <intrin.h>
#else
#if defined(__AVX__) || defined(__AVX2__) || defined(__AVX512F__) || defined(__SSSE3__) || defined(__SSE3__)
#if !defined(__riscv)
// 如果支持 AVX、AVX2、AVX512F、SSSE3 或 SSE3 指令集，并且不是在 RISC-V 平台下编译，则包含对应的头文件
#include <immintrin.h>
#endif
#endif
#endif
#endif
#endif
#endif
#ifdef __riscv_v_intrinsic
#include <riscv_vector.h>
#endif
// 如果定义了 __riscv_v_intrinsic，则包含 riscv_vector.h 头文件

#ifdef __F16C__
// 如果定义了 __F16C__，则执行以下代码

#ifdef _MSC_VER
// 如果是在 Microsoft Visual C++ 编译器下
#define GGML_COMPUTE_FP16_TO_FP32(x) _mm_cvtss_f32(_mm_cvtph_ps(_mm_cvtsi32_si128(x)))
#define GGML_COMPUTE_FP32_TO_FP16(x) _mm_extract_epi16(_mm_cvtps_ph(_mm_set_ss(x), 0), 0)
#else
// 如果不是在 Microsoft Visual C++ 编译器下
#define GGML_COMPUTE_FP16_TO_FP32(x) _cvtsh_ss(x)
#define GGML_COMPUTE_FP32_TO_FP16(x) _cvtss_sh(x, 0)
#endif
// 定义了将 FP16 转换为 FP32 和将 FP32 转换为 FP16 的宏

#elif defined(__POWER9_VECTOR__)
// 如果定义了 __POWER9_VECTOR__，则执行以下代码

#define GGML_COMPUTE_FP16_TO_FP32(x) ggml_compute_fp16_to_fp32(x)
#define GGML_COMPUTE_FP32_TO_FP16(x) ggml_compute_fp32_to_fp16(x)
// 定义了将 FP16 转换为 FP32 和将 FP32 转换为 FP16 的宏
/* the inline asm below is about 12% faster than the lookup method */
#define GGML_FP16_TO_FP32(x) GGML_COMPUTE_FP16_TO_FP32(x)
// 定义了将 FP16 转换为 FP32 的宏
// 定义一个宏，将输入的单精度浮点数转换为半精度浮点数
#define GGML_FP32_TO_FP16(x) GGML_COMPUTE_FP32_TO_FP16(x)

// 将半精度浮点数转换为单精度浮点数
static inline float ggml_compute_fp16_to_fp32(ggml_fp16_t h) {
    register float f; // 定义一个浮点数变量
    register double d; // 定义一个双精度浮点数变量
    __asm__( // 内联汇编，执行汇编指令
        "mtfprd %0,%2\n" // 将 h 转换为双精度浮点数
        "xscvhpdp %0,%0\n" // 将双精度浮点数转换为单精度浮点数
        "frsp %1,%0\n" : // 将结果存入 f 中
        /* temp */ "=d"(d), // 临时变量
        /* out */  "=f"(f): // 输出变量
        /* in */   "r"(h)); // 输入变量
    return f; // 返回单精度浮点数
}

// 将单精度浮点数转换为半精度浮点数
static inline ggml_fp16_t ggml_compute_fp32_to_fp16(float f) {
    register double d; // 定义一个双精度浮点数变量
    register ggml_fp16_t r; // 定义一个半精度浮点数变量
    __asm__( /* xscvdphp can work on double or single precision */
        "xscvdphp %0,%2\n" // 将单精度浮点数转换为双精度浮点数
        "mffprd %1,%0\n" :
        /* temp */ "=d"(d),
        /* out */  "=r"(r):
        /* in */   "f"(f));
    return r;
}

#else

// FP16 <-> FP32
// ref: https://github.com/Maratyszcza/FP16

// 将 32 位无符号整数转换为单精度浮点数
static inline float fp32_from_bits(uint32_t w) {
    // 定义一个联合体，用于存储 32 位整数和单精度浮点数
    union {
        uint32_t as_bits;
        float as_value;
    } fp32;
    // 将输入的 32 位整数赋值给联合体的整数部分
    fp32.as_bits = w;
    // 返回联合体的浮点数部分
    return fp32.as_value;
}
// 将单精度浮点数转换为其对应的32位二进制表示
static inline uint32_t fp32_to_bits(float f) {
    // 创建一个联合体，用于存储浮点数和对应的32位二进制表示
    union {
        float as_value;
        uint32_t as_bits;
    } fp32;
    // 将输入的单精度浮点数存储到联合体中
    fp32.as_value = f;
    // 返回对应的32位二进制表示
    return fp32.as_bits;
}

// 计算16位半精度浮点数转换为32位单精度浮点数
static inline float ggml_compute_fp16_to_fp32(ggml_fp16_t h) {
    // 将16位半精度浮点数左移16位，转换为32位整数
    const uint32_t w = (uint32_t) h << 16;
    // 提取符号位
    const uint32_t sign = w & UINT32_C(0x80000000);
    // 计算两倍的32位整数表示的值
    const uint32_t two_w = w + w;

    // 计算指数的偏移量
    const uint32_t exp_offset = UINT32_C(0xE0) << 23;
    // 计算指数的缩放因子
    const float exp_scale = 0x1.0p-112f; // 如果编译器支持C99标准或者GNU扩展，则使用0x1.0p-112f
    // 如果编译器不支持C99标准或者GNU扩展，则使用自定义的32位二进制表示转换为单精度浮点数的函数
    const float exp_scale = fp32_from_bits(UINT32_C(0x7800000));
// 计算规格化值
const float normalized_value = fp32_from_bits((two_w >> 4) + exp_offset) * exp_scale;

// 定义用于计算非规格化值的掩码和偏置
const uint32_t magic_mask = UINT32_C(126) << 23;
const float magic_bias = 0.5f;
// 计算非规格化值
const float denormalized_value = fp32_from_bits((two_w >> 17) | magic_mask) - magic_bias;

// 定义用于判断是否为非规格化值的截断值
const uint32_t denormalized_cutoff = UINT32_C(1) << 27;
// 计算最终结果
const uint32_t result = sign |
    (two_w < denormalized_cutoff ? fp32_to_bits(denormalized_value) : fp32_to_bits(normalized_value));
// 返回结果
return fp32_from_bits(result);
}

// 计算单精度浮点数到半精度浮点数的转换
static inline ggml_fp16_t ggml_compute_fp32_to_fp16(float f) {
// 根据不同的编译环境定义不同的缩放值
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 199901L) || defined(__GNUC__) && !defined(__STRICT_ANSI__)
    const float scale_to_inf = 0x1.0p+112f;
    const float scale_to_zero = 0x1.0p-110f;
#else
    const float scale_to_inf = fp32_from_bits(UINT32_C(0x77800000));
    const float scale_to_zero = fp32_from_bits(UINT32_C(0x08800000));
// 结束条件判断
#endif

// 计算基数
float base = (fabsf(f) * scale_to_inf) * scale_to_zero;

// 将浮点数转换为32位整数
const uint32_t w = fp32_to_bits(f);
const uint32_t shl1_w = w + w;
const uint32_t sign = w & UINT32_C(0x80000000);
uint32_t bias = shl1_w & UINT32_C(0xFF000000);

// 如果偏置小于0x71000000，则将偏置设置为0x71000000
if (bias < UINT32_C(0x71000000)) {
    bias = UINT32_C(0x71000000);
}

// 计算基数
base = fp32_from_bits((bias >> 1) + UINT32_C(0x07800000)) + base;
const uint32_t bits = fp32_to_bits(base);
const uint32_t exp_bits = (bits >> 13) & UINT32_C(0x00007C00);
const uint32_t mantissa_bits = bits & UINT32_C(0x00000FFF);
const uint32_t nonsign = exp_bits + mantissa_bits;

// 返回结果
return (sign >> 16) | (shl1_w > UINT32_C(0xFF000000) ? UINT16_C(0x7E00) : nonsign);
}

// 定义宏，用于将FP16转换为FP32
#define GGML_COMPUTE_FP16_TO_FP32(x) ggml_compute_fp16_to_fp32(x)
// 定义宏 GGML_COMPUTE_FP32_TO_FP16，用于将 float32 转换为 float16
#define GGML_COMPUTE_FP32_TO_FP16(x) ggml_compute_fp32_to_fp16(x)

#endif // 如果没有定义 __F16C__，则结束条件编译

#endif // 如果没有定义 __ARM_NEON，也结束条件编译

// 预先计算的 f32 到 f16 的转换表（256 KB）
// 在 ggml.c 中定义，在 ggml_init() 中初始化
extern float ggml_table_f32_f16[1 << 16];

// 在 ARM NEON 上，直接将 x 转换为 x 比调用 ggml_lookup_fp16_to_fp32 更快，
// 因此我们在其他地方为 NEON 定义了 GGML_FP16_TO_FP32 和 GGML_FP32_TO_FP16。
// 对于 POWER9 也是如此。
#if !defined(GGML_FP16_TO_FP32) || !defined(GGML_FP32_TO_FP16)

// 内联函数，用于查找将 ggml_fp16_t 转换为 float32 的值
inline static float ggml_lookup_fp16_to_fp32(ggml_fp16_t f) {
    uint16_t s;
    memcpy(&s, &f, sizeof(uint16_t));
    return ggml_table_f32_f16[s];
}
// 将输入的 FP16 数据转换为 FP32 数据
#define GGML_FP16_TO_FP32(x) ggml_lookup_fp16_to_fp32(x)
// 将输入的 FP32 数据转换为 FP16 数据
#define GGML_FP32_TO_FP16(x) GGML_COMPUTE_FP32_TO_FP16(x)

#endif

// 表示哈希表已满
#define GGML_HASHTABLE_FULL ((size_t)-1)
// 表示哈希表中已存在相同的键
#define GGML_HASHTABLE_ALREADY_EXISTS ((size_t)-2)

// 检查哈希表中是否包含指定的键
bool   ggml_hash_contains      (const struct ggml_hash_set hash_set, struct ggml_tensor * key);

// 如果哈希表已满，则返回 GGML_HASHTABLE_FULL；否则返回键的当前索引或应插入的位置
size_t ggml_hash_find          (const struct ggml_hash_set hash_set, struct ggml_tensor * key);

// 如果键已存在，则返回 GGML_HASHTABLE_ALREADY_EXISTS；否则返回索引，如果表已满则断言
size_t ggml_hash_insert        (      struct ggml_hash_set hash_set, struct ggml_tensor * key);

// 返回索引，如果表已满则断言
size_t ggml_hash_find_or_insert(      struct ggml_hash_set hash_set, struct ggml_tensor * key);
#ifdef __cplusplus
// 如果是 C++ 环境
}
#endif
// 结束 C++ 环境的定义
```