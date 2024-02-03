# `xmrig\src\crypto\cn\sse2neon.h`

```cpp
// 如果未定义 SSE2NEON_H，则定义 SSE2NEON_H
#ifndef SSE2NEON_H
#define SSE2NEON_H

// 该头文件提供了一个简单的 API 翻译层，将 SSE 内联函数翻译成对应的 Arm/Aarch64 NEON 版本

// 贡献者
// 以下是对该工作做出贡献的人员列表
// ...

// 结束条件编译指令
#endif
/*
 * sse2neon 是在 MIT 许可下免费可重新分发的。
 *
 * 在此免费授予任何获得本软件及相关文档文件（以下简称“软件”）副本的人，无偿处理本软件的权利，包括但不限于使用、复制、修改、合并、发布、分发、再许可和/或出售本软件的副本，并允许获得本软件的人员这样做，但须遵守以下条件：
 *
 * 上述版权声明和本许可声明应包含在所有副本或重要部分的软件中。
 *
 * 本软件按“原样”提供，不提供任何形式的明示或暗示的担保，包括但不限于适销性、特定用途的适用性和非侵权性的担保。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是在合同行为、侵权行为或其他方面产生的，与软件或使用或其他交易有关。
 */

/* 可调整的配置 */

/* 启用精确的数学运算实现
 * 这会稍微减慢计算速度，但可以与 x86 SSE 一致地得到结果。（例如，可以解决渲染结果中的空洞或 NaN 像素）
 */
/* _mm_min|max_ps|ss|pd|sd */
#ifndef SSE2NEON_PRECISE_MINMAX
#define SSE2NEON_PRECISE_MINMAX (0)
#endif
/* _mm_rcp_ps and _mm_div_ps */
#ifndef SSE2NEON_PRECISE_DIV
#define SSE2NEON_PRECISE_DIV (0)
#endif
/* _mm_sqrt_ps and _mm_rsqrt_ps */
#ifndef SSE2NEON_PRECISE_SQRT
#define SSE2NEON_PRECISE_SQRT (0)
#endif
/* _mm_dp_pd */
#ifndef SSE2NEON_PRECISE_DP
#define SSE2NEON_PRECISE_DP (0)
#endif

/* 在 MSVC 平台上启用 windows.h 的包含
 * 这使得在 Windows 上 _mm_clflush 在没有内置功能的情况下也能正常工作。
 */
#ifndef SSE2NEON_INCLUDE_WINDOWS_H
#define SSE2NEON_INCLUDE_WINDOWS_H (0)
#endif
/* compiler specific definitions */
#if defined(__GNUC__) || defined(__clang__)
#pragma push_macro("FORCE_INLINE")  // 保存当前宏定义，并定义新的宏
#pragma push_macro("ALIGN_STRUCT")  // 保存当前宏定义，并定义新的宏
#define FORCE_INLINE static inline __attribute__((always_inline))  // 定义 FORCE_INLINE 宏
#define ALIGN_STRUCT(x) __attribute__((aligned(x)))  // 定义 ALIGN_STRUCT 宏
#define _sse2neon_likely(x) __builtin_expect(!!(x), 1)  // 定义 _sse2neon_likely 宏
#define _sse2neon_unlikely(x) __builtin_expect(!!(x), 0)  // 定义 _sse2neon_unlikely 宏
#elif defined(_MSC_VER)
#if _MSVC_TRADITIONAL
#error Using the traditional MSVC preprocessor is not supported! Use /Zc:preprocessor instead.
#endif
#ifndef FORCE_INLINE
#define FORCE_INLINE static inline  // 如果未定义 FORCE_INLINE 宏，则定义为 static inline
#endif
#ifndef ALIGN_STRUCT
#define ALIGN_STRUCT(x) __declspec(align(x))  // 如果未定义 ALIGN_STRUCT 宏，则定义为 __declspec(align(x))
#endif
#define _sse2neon_likely(x) (x)  // 定义 _sse2neon_likely 宏
#define _sse2neon_unlikely(x) (x)  // 定义 _sse2neon_unlikely 宏
#else
#pragma message("Macro name collisions may happen with unsupported compilers.")  // 如果使用不支持的编译器，则输出警告信息
#endif

/* C language does not allow initializing a variable with a function call. */
#ifdef __cplusplus
#define _sse2neon_const static const  // 如果是 C++ 编译，则定义 _sse2neon_const 为 static const
#else
#define _sse2neon_const const  // 如果是 C 编译，则定义 _sse2neon_const 为 const
#endif

#include <stdint.h>  // 包含标准整数类型定义
#include <stdlib.h>  // 包含标准库函数定义

#if defined(_WIN32)
/* Definitions for _mm_{malloc,free} are provided by <malloc.h>
 * from both MinGW-w64 and MSVC.
 */
#define SSE2NEON_ALLOC_DEFINED  // 定义 SSE2NEON_ALLOC_DEFINED 宏
#endif

/* If using MSVC */
#ifdef _MSC_VER
#include <intrin.h>  // 包含 MSVC 内置函数定义
#if SSE2NEON_INCLUDE_WINDOWS_H
#include <processthreadsapi.h>  // 包含进程和线程 API 定义
#include <windows.h>  // 包含 Windows API 定义
#endif

#if !defined(__cplusplus)
#error sse2neon only supports C++ compilation with this compiler  // 如果不是 C++ 编译，则输出错误信息
#endif

#ifdef SSE2NEON_ALLOC_DEFINED
#include <malloc.h>  // 如果定义了 SSE2NEON_ALLOC_DEFINED 宏，则包含内存分配函数定义
#endif

#if (defined(_M_AMD64) || defined(__x86_64__)) || \
    (defined(_M_ARM64) || defined(__arm64__))
#define SSE2NEON_HAS_BITSCAN64  // 如果是 x86-64 或 ARM64 架构，则定义 SSE2NEON_HAS_BITSCAN64 宏
#endif
#endif

#if defined(__GNUC__) || defined(__clang__)
#define _sse2neon_define0(type, s, body) \  // 定义 _sse2neon_define0 宏
    __extension__({                      \
        type _a = (s);                   \
        body                             \
    })
#define _sse2neon_define1(type, s, body) \  // 定义 _sse2neon_define1 宏
    # 定义一个扩展，包含类型和主体部分
    __extension__({                      \
        # 定义一个类型为_s的变量_a
        type _a = (s);                   \
        # 执行主体部分
        body                             \
    })
#define _sse2neon_define2(type, a, b, body) \  // 定义一个宏，接受类型、a、b和body作为参数
    __extension__({                         \  // 使用__extension__包裹，表示这是一个语言扩展
        type _a = (a), _b = (b);            \  // 定义类型为_a和_b的变量，并初始化为a和b
        body                                \  // 执行传入的body代码块
    })

#define _sse2neon_return(ret) (ret)  // 定义一个宏，用于返回传入的ret值

#else  // 如果不满足上述条件
#define _sse2neon_define0(type, a, body) [=](type _a) { body }(a)  // 定义一个lambda表达式，接受类型和a作为参数
#define _sse2neon_define1(type, a, body) [](type _a) { body }(a)   // 定义一个lambda表达式，接受类型和a作为参数
#define _sse2neon_define2(type, a, b, body) \  // 定义一个lambda表达式，接受类型、a和b作为参数
    [](type _a, type _b) { body }((a), (b))
#define _sse2neon_return(ret) return ret  // 返回传入的ret值
#endif

#define _sse2neon_init(...) \  // 定义一个宏，接受可变数量的参数
    {                       \  // 代码块开始
        __VA_ARGS__         \  // 使用可变数量的参数
    }                       \  // 代码块结束

/* Compiler barrier */
#if defined(_MSC_VER)  // 如果定义了_MSC_VER
#define SSE2NEON_BARRIER() _ReadWriteBarrier()  // 使用_ReadWriteBarrier()作为编译器屏障
#else  // 如果不满足上述条件
#define SSE2NEON_BARRIER()                     \  // 定义一个编译器屏障
    do {                                       \  // 执行以下代码块
        __asm__ __volatile__("" ::: "memory"); \  // 内联汇编，使用内存约束
        (void) 0;                              \  // 空操作
    } while (0)  // 循环条件为0，表示只执行一次
#endif

/* Memory barriers
 * __atomic_thread_fence does not include a compiler barrier; instead,
 * the barrier is part of __atomic_load/__atomic_store's "volatile-like"
 * semantics.
 */
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 201112L)  // 如果满足条件
#include <stdatomic.h>  // 包含stdatomic.h头文件
#endif

FORCE_INLINE void _sse2neon_smp_mb(void)  // 定义一个FORCE_INLINE函数_sse2neon_smp_mb
{
    SSE2NEON_BARRIER();  // 调用SSE2NEON_BARRIER()函数
#if defined(__STDC_VERSION__) && (__STDC_VERSION__ >= 201112L) && \  // 如果满足条件
    !defined(__STDC_NO_ATOMICS__)  // 如果不定义__STDC_NO_ATOMICS__
    atomic_thread_fence(memory_order_seq_cst);  // 使用原子操作的内存屏障
#elif defined(__GNUC__) || defined(__clang__)  // 如果满足条件
    __atomic_thread_fence(__ATOMIC_SEQ_CST);  // 使用__ATOMIC_SEQ_CST内存屏障
#else /* MSVC */  // 如果不满足上述条件
    __dmb(_ARM64_BARRIER_ISH);  // 使用ARM64的内存屏障
#endif
}

/* Architecture-specific build options */
/* FIXME: #pragma GCC push_options is only available on GCC */
#if defined(__GNUC__)  // 如果定义了__GNUC__
#if defined(__arm__) && __ARM_ARCH == 7  // 如果定义了__arm__并且__ARM_ARCH等于7
/* According to ARM C Language Extensions Architecture specification,
 * __ARM_NEON is defined to a value indicating the Advanced SIMD (NEON)
 * architecture supported.
 */
#if !defined(__ARM_NEON) || !defined(__ARM_NEON__)  // 如果未定义__ARM_NEON或者未定义__ARM_NEON__
#error "You must enable NEON instructions (e.g. -mfpu=neon) to use SSE2NEON."
#endif
#if !defined(__clang__)
#pragma GCC push_options
#pragma GCC target("fpu=neon")
#endif
#elif defined(__aarch64__) || defined(_M_ARM64)
#if !defined(__clang__) && !defined(_MSC_VER)
#pragma GCC push_options
#pragma GCC target("+simd")
#endif
#elif __ARM_ARCH == 8
#if !defined(__ARM_NEON) || !defined(__ARM_NEON__)
#error \
    "You must enable NEON instructions (e.g. -mfpu=neon-fp-armv8) to use SSE2NEON."
#endif
#if !defined(__clang__) && !defined(_MSC_VER)
#pragma GCC push_options
#endif
#else
#error "Unsupported target. Must be either ARMv7-A+NEON or ARMv8-A."
#endif
#endif

#include <arm_neon.h>
#if (!defined(__aarch64__) && !defined(_M_ARM64)) && (__ARM_ARCH == 8)
#if defined __has_include && __has_include(<arm_acle.h>)
#include <arm_acle.h>
#endif
#endif

/* Apple Silicon cache lines are double of what is commonly used by Intel, AMD
 * and other Arm microarchitectures use.
 * From sysctl -a on Apple M1:
 * hw.cachelinesize: 128
 */
#if defined(__APPLE__) && (defined(__aarch64__) || defined(__arm64__))
#define SSE2NEON_CACHELINE_SIZE 128
#else
#define SSE2NEON_CACHELINE_SIZE 64
#endif

/* Rounding functions require either Aarch64 instructions or libm fallback */
#if !defined(__aarch64__) && !defined(_M_ARM64)
#include <math.h>
#endif

/* On ARMv7, some registers, such as PMUSERENR and PMCCNTR, are read-only
 * or even not accessible in user mode.
 * To write or access to these registers in user mode,
 * we have to perform syscall instead.
 */
#if (!defined(__aarch64__) && !defined(_M_ARM64))
#include <sys/time.h>
#endif

/* "__has_builtin" can be used to query support for built-in functions
 * provided by gcc/clang and other compilers that support it.
 */
#ifndef __has_builtin /* GCC prior to 10 or non-clang compilers */
/* Compatibility with gcc <= 9 */
#if defined(__GNUC__) && (__GNUC__ <= 9)
#define __has_builtin(x) HAS##x
#define HAS__builtin_popcount 1
#define HAS__builtin_popcountll 1
// 定义宏，用于指示编译器是否支持__builtin_popcountll函数

#if (__GNUC__ >= 5) || ((__GNUC__ == 4) && (__GNUC_MINOR__ >= 7))
#define HAS__builtin_shuffle 1
#else
#define HAS__builtin_shuffle 0
#endif
// 根据编译器版本判断是否支持__builtin_shuffle函数，并定义宏

#define HAS__builtin_shufflevector 0
#define HAS__builtin_nontemporal_store 0
#else
#define __has_builtin(x) 0
#endif
#endif
// 定义宏，用于指示编译器是否支持__builtin_shufflevector和__builtin_nontemporal_store函数

/**
 * MACRO for shuffle parameter for _mm_shuffle_ps().
 * Argument fp3 is a digit[0123] that represents the fp from argument "b"
 * of mm_shuffle_ps that will be placed in fp3 of result. fp2 is the same
 * for fp2 in result. fp1 is a digit[0123] that represents the fp from
 * argument "a" of mm_shuffle_ps that will be places in fp1 of result.
 * fp0 is the same for fp0 of result.
 */
#define _MM_SHUFFLE(fp3, fp2, fp1, fp0) \
    (((fp3) << 6) | ((fp2) << 4) | ((fp1) << 2) | ((fp0)))
// 定义宏，用于生成_mm_shuffle_ps()函数的参数

#if __has_builtin(__builtin_shufflevector)
#define _sse2neon_shuffle(type, a, b, ...) \
    __builtin_shufflevector(a, b, __VA_ARGS__)
#elif __has_builtin(__builtin_shuffle)
#define _sse2neon_shuffle(type, a, b, ...) \
    __extension__({                        \
        type tmp = {__VA_ARGS__};          \
        __builtin_shuffle(a, b, tmp);      \
    })
#endif
// 根据编译器是否支持__builtin_shufflevector和__builtin_shuffle函数，定义宏_sse2neon_shuffle

#ifdef _sse2neon_shuffle
#define vshuffle_s16(a, b, ...) _sse2neon_shuffle(int16x4_t, a, b, __VA_ARGS__)
#define vshuffleq_s16(a, b, ...) _sse2neon_shuffle(int16x8_t, a, b, __VA_ARGS__)
#define vshuffle_s32(a, b, ...) _sse2neon_shuffle(int32x2_t, a, b, __VA_ARGS__)
#define vshuffleq_s32(a, b, ...) _sse2neon_shuffle(int32x4_t, a, b, __VA_ARGS__)
#define vshuffle_s64(a, b, ...) _sse2neon_shuffle(int64x1_t, a, b, __VA_ARGS__)
#define vshuffleq_s64(a, b, ...) _sse2neon_shuffle(int64x2_t, a, b, __VA_ARGS__)
#endif
// 根据宏_sse2neon_shuffle的定义，定义一系列vshuffle函数

/* Rounding mode macros. */
#define _MM_FROUND_TO_NEAREST_INT 0x00
#define _MM_FROUND_TO_NEG_INF 0x01
#define _MM_FROUND_TO_POS_INF 0x02
#define _MM_FROUND_TO_ZERO 0x03
#define _MM_FROUND_CUR_DIRECTION 0x04
#define _MM_FROUND_NO_EXC 0x08
#define _MM_FROUND_RAISE_EXC 0x00
// 定义一系列舍入模式的宏
#define _MM_FROUND_NINT (_MM_FROUND_TO_NEAREST_INT | _MM_FROUND_RAISE_EXC)
#define _MM_FROUND_FLOOR (_MM_FROUND_TO_NEG_INF | _MM_FROUND_RAISE_EXC)
#define _MM_FROUND_CEIL (_MM_FROUND_TO_POS_INF | _MM_FROUND_RAISE_EXC)
#define _MM_FROUND_TRUNC (_MM_FROUND_TO_ZERO | _MM_FROUND_RAISE_EXC)
#define _MM_FROUND_RINT (_MM_FROUND_CUR_DIRECTION | _MM_FROUND_RAISE_EXC)
#define _MM_FROUND_NEARBYINT (_MM_FROUND_CUR_DIRECTION | _MM_FROUND_NO_EXC)
#define _MM_ROUND_NEAREST 0x0000
#define _MM_ROUND_DOWN 0x2000
#define _MM_ROUND_UP 0x4000
#define _MM_ROUND_TOWARD_ZERO 0x6000
/* Flush zero mode macros. */
#define _MM_FLUSH_ZERO_MASK 0x8000
#define _MM_FLUSH_ZERO_ON 0x8000
#define _MM_FLUSH_ZERO_OFF 0x0000
/* Denormals are zeros mode macros. */
#define _MM_DENORMALS_ZERO_MASK 0x0040
#define _MM_DENORMALS_ZERO_ON 0x0040
#define _MM_DENORMALS_ZERO_OFF 0x0000
# 定义舍入模式的宏，用于浮点数运算
# 定义舍入模式为最接近整数
#define _MM_FROUND_NINT (_MM_FROUND_TO_NEAREST_INT | _MM_FROUND_RAISE_EXC)
# 定义舍入模式为向下取整
#define _MM_FROUND_FLOOR (_MM_FROUND_TO_NEG_INF | _MM_FROUND_RAISE_EXC)
# 定义舍入模式为向上取整
#define _MM_FROUND_CEIL (_MM_FROUND_TO_POS_INF | _MM_FROUND_RAISE_EXC)
# 定义舍入模式为截断
#define _MM_FROUND_TRUNC (_MM_FROUND_TO_ZERO | _MM_FROUND_RAISE_EXC)
# 定义舍入模式为最接近整数
#define _MM_FROUND_RINT (_MM_FROUND_CUR_DIRECTION | _MM_FROUND_RAISE_EXC)
# 定义舍入模式为最接近整数，不触发异常
#define _MM_FROUND_NEARBYINT (_MM_FROUND_CUR_DIRECTION | _MM_FROUND_NO_EXC)
# 定义舍入模式为最接近整数
#define _MM_ROUND_NEAREST 0x0000
# 定义舍入模式为向下取整
#define _MM_ROUND_DOWN 0x2000
# 定义舍入模式为向上取整
#define _MM_ROUND_UP 0x4000
# 定义舍入模式为朝零方向取整
#define _MM_ROUND_TOWARD_ZERO 0x6000
/* Flush zero mode macros. */
# 定义清零模式的宏
#define _MM_FLUSH_ZERO_MASK 0x8000
# 定义开启清零模式
#define _MM_FLUSH_ZERO_ON 0x8000
# 定义关闭清零模式
#define _MM_FLUSH_ZERO_OFF 0x0000
/* Denormals are zeros mode macros. */
# 定义非规格化数为零模式的宏
#define _MM_DENORMALS_ZERO_MASK 0x0040
# 定义开启非规格化数为零模式
#define _MM_DENORMALS_ZERO_ON 0x0040
# 定义关闭非规格化数为零模式
#define _MM_DENORMALS_ZERO_OFF 0x0000

/* indicate immediate constant argument in a given range */
# 指示给定范围内的立即常量参数
#define __constrange(a, b) const

/* A few intrinsics accept traditional data types like ints or floats, but
 * most operate on data types that are specific to SSE.
 * If a vector type ends in d, it contains doubles, and if it does not have
 * a suffix, it contains floats. An integer vector type can contain any type
 * of integer, from chars to shorts to unsigned long longs.
 */
# 一些内在函数接受传统的数据类型，如整数或浮点数，但大多数操作的是特定于 SSE 的数据类型。
# 如果向量类型以 d 结尾，则包含双精度浮点数，如果没有后缀，则包含单精度浮点数。整数向量类型可以包含任何类型的整数，从字符到短整型到无符号长整型。
typedef int64x1_t __m64;
typedef float32x4_t __m128; /* 128-bit vector containing 4 floats */
// On ARM 32-bit architecture, the float64x2_t is not supported.
// The data type __m128d should be represented in a different way for related
// intrinsic conversion.
#if defined(__aarch64__) || defined(_M_ARM64)
typedef float64x2_t __m128d; /* 128-bit vector containing 2 doubles */
#else
typedef float32x4_t __m128d;
#endif
typedef int64x2_t __m128i; /* 128-bit vector containing integers */

# __int64 is defined in the Intrinsics Guide which maps to different datatype
# in different data model
# 在内在指南中定义了 __int64，它在不同的数据模型中映射到不同的数据类型
#if !(defined(_WIN32) || defined(_WIN64) || defined(__int64))
#if (defined(__x86_64__) || defined(__i386__))
#define __int64 long long
#else
#define __int64 int64_t
#endif
#endif

/* type-safe casting between types */

#define vreinterpretq_m128_f16(x) vreinterpretq_f32_f16(x)  // 将128位寄存器中的数据重新解释为16位浮点数
#define vreinterpretq_m128_f32(x) (x)  // 将128位寄存器中的数据重新解释为32位浮点数
#define vreinterpretq_m128_f64(x) vreinterpretq_f32_f64(x)  // 将128位寄存器中的数据重新解释为64位浮点数

#define vreinterpretq_m128_u8(x) vreinterpretq_f32_u8(x)  // 将128位寄存器中的数据重新解释为8位无符号整数
#define vreinterpretq_m128_u16(x) vreinterpretq_f32_u16(x)  // 将128位寄存器中的数据重新解释为16位无符号整数
#define vreinterpretq_m128_u32(x) vreinterpretq_f32_u32(x)  // 将128位寄存器中的数据重新解释为32位无符号整数
#define vreinterpretq_m128_u64(x) vreinterpretq_f32_u64(x)  // 将128位寄存器中的数据重新解释为64位无符号整数

#define vreinterpretq_m128_s8(x) vreinterpretq_f32_s8(x)  // 将128位寄存器中的数据重新解释为8位有符号整数
#define vreinterpretq_m128_s16(x) vreinterpretq_f32_s16(x)  // 将128位寄存器中的数据重新解释为16位有符号整数
#define vreinterpretq_m128_s32(x) vreinterpretq_f32_s32(x)  // 将128位寄存器中的数据重新解释为32位有符号整数
#define vreinterpretq_m128_s64(x) vreinterpretq_f32_s64(x)  // 将128位寄存器中的数据重新解释为64位有符号整数

#define vreinterpretq_f16_m128(x) vreinterpretq_f16_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为16位浮点数
#define vreinterpretq_f32_m128(x) (x)  // 将128位寄存器中的数据重新解释为32位浮点数
#define vreinterpretq_f64_m128(x) vreinterpretq_f64_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为64位浮点数

#define vreinterpretq_u8_m128(x) vreinterpretq_u8_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为8位无符号整数
#define vreinterpretq_u16_m128(x) vreinterpretq_u16_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为16位无符号整数
#define vreinterpretq_u32_m128(x) vreinterpretq_u32_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为32位无符号整数
#define vreinterpretq_u64_m128(x) vreinterpretq_u64_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为64位无符号整数

#define vreinterpretq_s8_m128(x) vreinterpretq_s8_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为8位有符号整数
#define vreinterpretq_s16_m128(x) vreinterpretq_s16_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为16位有符号整数
#define vreinterpretq_s32_m128(x) vreinterpretq_s32_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为32位有符号整数
#define vreinterpretq_s64_m128(x) vreinterpretq_s64_f32(x)  // 将128位寄存器中的数据重新解释为32位浮点数，再转换为64位有符号整数

#define vreinterpretq_m128i_s8(x) vreinterpretq_s64_s8(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为8位有符号整数
#define vreinterpretq_m128i_s16(x) vreinterpretq_s64_s16(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为16位有符号整数
#define vreinterpretq_m128i_s32(x) vreinterpretq_s64_s32(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为32位有符号整数
#define vreinterpretq_m128i_s64(x) (x)  // 将128位寄存器中的数据重新解释为64位有符号整数

#define vreinterpretq_m128i_u8(x) vreinterpretq_s64_u8(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为8位无符号整数
#define vreinterpretq_m128i_u16(x) vreinterpretq_s64_u16(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为16位无符号整数
#define vreinterpretq_m128i_u32(x) vreinterpretq_s64_u32(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为32位无符号整数
#define vreinterpretq_m128i_u64(x) vreinterpretq_s64_u64(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为64位无符号整数

#define vreinterpretq_f32_m128i(x) vreinterpretq_f32_s64(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为32位浮点数
#define vreinterpretq_f64_m128i(x) vreinterpretq_f64_s64(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为64位浮点数

#define vreinterpretq_s8_m128i(x) vreinterpretq_s8_s64(x)  // 将128位寄存器中的数据重新解释为64位有符号整数，再转换为8位有符号整数
#define vreinterpretq_s16_m128i(x) vreinterpretq_s16_s64(x)
# 将128位整数向量转换为128位整数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpretq_s32_m128i(x) vreinterpretq_s32_s64(x)
# 将128位整数向量转换为128位整数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpretq_s64_m128i(x) (x)
# 将128位整数向量转换为128位整数向量，不进行实际的转换

#define vreinterpretq_u8_m128i(x) vreinterpretq_u8_s64(x)
# 将128位整数向量转换为128位整数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpretq_u16_m128i(x) vreinterpretq_u16_s64(x)
# 将128位整数向量转换为128位整数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpretq_u32_m128i(x) vreinterpretq_u32_s64(x)
# 将128位整数向量转换为128位整数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpretq_u64_m128i(x) vreinterpretq_u64_s64(x)
# 将128位整数向量转换为128位整数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpret_m64_s8(x) vreinterpret_s64_s8(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为8位整数向量

#define vreinterpret_m64_s16(x) vreinterpret_s64_s16(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为16位整数向量

#define vreinterpret_m64_s32(x) vreinterpret_s64_s32(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为32位整数向量

#define vreinterpret_m64_s64(x) (x)
# 将64位整数向量转换为64位整数向量，不进行实际的转换

#define vreinterpret_m64_u8(x) vreinterpret_s64_u8(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为8位整数向量

#define vreinterpret_m64_u16(x) vreinterpret_s64_u16(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为16位整数向量

#define vreinterpret_m64_u32(x) vreinterpret_s64_u32(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为32位整数向量

#define vreinterpret_m64_u64(x) vreinterpret_s64_u64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpret_m64_f16(x) vreinterpret_s64_f16(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为16位浮点数向量

#define vreinterpret_m64_f32(x) vreinterpret_s64_f32(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为32位浮点数向量

#define vreinterpret_m64_f64(x) vreinterpret_s64_f64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为64位浮点数向量

#define vreinterpret_u8_m64(x) vreinterpret_u8_s64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为8位整数向量

#define vreinterpret_u16_m64(x) vreinterpret_u16_s64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为16位整数向量

#define vreinterpret_u32_m64(x) vreinterpret_u32_s64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为32位整数向量

#define vreinterpret_u64_m64(x) vreinterpret_u64_s64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpret_s8_m64(x) vreinterpret_s8_s64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为8位整数向量

#define vreinterpret_s16_m64(x) vreinterpret_s16_s64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为16位整数向量

#define vreinterpret_s32_m64(x) vreinterpret_s32_s64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为32位整数向量

#define vreinterpret_s64_m64(x) (x)
# 将64位整数向量转换为64位整数向量，不进行实际的转换

#define vreinterpret_f32_m64(x) vreinterpret_f32_s64(x)
# 将64位整数向量转换为64位整数向量，但是底层实现是将其转换为32位浮点数向量

#if defined(__aarch64__) || defined(_M_ARM64)
# 如果是 ARM64 架构

#define vreinterpretq_m128d_s32(x) vreinterpretq_f64_s32(x)
# 将128位双精度浮点数向量转换为128位双精度浮点数向量，但是底层实现是将其转换为32位整数向量

#define vreinterpretq_m128d_s64(x) vreinterpretq_f64_s64(x)
# 将128位双精度浮点数向量转换为128位双精度浮点数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpretq_m128d_u64(x) vreinterpretq_f64_u64(x)
# 将128位双精度浮点数向量转换为128位双精度浮点数向量，但是底层实现是将其转换为64位无符号整数向量

#define vreinterpretq_m128d_f32(x) vreinterpretq_f64_f32(x)
# 将128位双精度浮点数向量转换为128位双精度浮点数向量，但是底层实现是将其转换为32位浮点数向量

#define vreinterpretq_m128d_f64(x) (x)
# 将128位双精度浮点数向量转换为128位双精度浮点数向量，不进行实际的转换

#define vreinterpretq_s64_m128d(x) vreinterpretq_s64_f64(x)
# 将128位双精度浮点数向量转换为128位双精度浮点数向量，但是底层实现是将其转换为64位整数向量

#define vreinterpretq_u32_m128d(x) vreinterpretq_u32_f64(x)
# 将128位双精度浮点数向量转换为128位双精度浮点数向量，但是底层实现是将其转换为32位无符号整数向量

#define vreinterpretq_u64_m128d(x) vreinterpretq_u64_f64(x)
# 将128位双精度浮点数向量转换为128位双精度浮点数向量，但是底层实现是将其转换为64位无符号整数向量
// 定义了一个宏，用于将__m128d类型的变量重新解释为__m128d类型
#define vreinterpretq_f64_m128d(x) (x)
// 定义了一个宏，用于将__m128d类型的变量重新解释为__m128类型
#define vreinterpretq_f32_m128d(x) vreinterpretq_f32_f64(x)
#else
// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_m128d_s32(x) vreinterpretq_f32_s32(x)
// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_m128d_s64(x) vreinterpretq_f32_s64(x)

// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_m128d_u32(x) vreinterpretq_f32_u32(x)
// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_m128d_u64(x) vreinterpretq_f32_u64(x)

// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_m128d_f32(x) (x)

// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_s64_m128d(x) vreinterpretq_s64_f32(x)

// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_u32_m128d(x) vreinterpretq_u32_f32(x)
// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_u64_m128d(x) vreinterpretq_u64_f32(x)

// 定义了一个宏，用于将__m128类型的变量重新解释为__m128d类型
#define vreinterpretq_f32_m128d(x) (x)
#endif

// 在该头文件中定义了一个名为'SIMDVec'的结构体，可以被应用程序用来直接访问__m128结构体的内容。
// 需要注意的是，直接访问__m128结构体是微软的不良编码实践：@see: https://learn.microsoft.com/en-us/cpp/cpp/m128
//
// 然而，一些旧的源代码可能会尝试直接访问__m128结构体的内容，因此开发人员可以使用SIMDVec作为其别名。
// 任何类型转换必须由开发人员手动完成，因为你不能对基本NEON数据类型进行类型转换或别名操作。
//
// 该联合体旨在允许使用MSVC编译器提供的名称直接访问__m128变量。这个联合体应该只在尝试访问向量的成员作为整数值时使用。
// GCC/clang允许通过简单的数组访问操作符（在C语言自4.6版本以来，在C++语言自4.8版本以来）直接访问浮点数成员。
//
// 理想情况下，不应该使用直接访问SIMD向量，因为这可能会导致性能损失。然而，如果真的需要，原始的__m128变量可以与指向该联合体的指针进行别名，并用于访问单个组件。
// 使用该联合体应该隐藏在一个宏的后面
// 定义一个联合体类型 SIMDVec，用于访问不同类型的数据
typedef union ALIGN_STRUCT(16) SIMDVec {
    float m128_f32[4];     // 作为浮点数 - 不要使用，只是为了方便添加
    int8_t m128_i8[16];    // 作为有符号8位整数
    int16_t m128_i16[8];   // 作为有符号16位整数
    int32_t m128_i32[4];   // 作为有符号32位整数
    int64_t m128_i64[2];   // 作为有符号64位整数
    uint8_t m128_u8[16];   // 作为无符号8位整数
    uint16_t m128_u16[8];  // 作为无符号16位整数
    uint32_t m128_u32[4];  // 作为无符号32位整数
    uint64_t m128_u64[2];  // 作为无符号64位整数
} SIMDVec;

// 使用 SIMDVec 进行类型转换
#define vreinterpretq_nth_u64_m128i(x, n) (((SIMDVec *) &x)->m128_u64[n])
#define vreinterpretq_nth_u32_m128i(x, n) (((SIMDVec *) &x)->m128_u32[n])
#define vreinterpretq_nth_u8_m128i(x, n) (((SIMDVec *) &x)->m128_u8[n])

/* SSE 宏 */
#define _MM_GET_FLUSH_ZERO_MODE _sse2neon_mm_get_flush_zero_mode
#define _MM_SET_FLUSH_ZERO_MODE _sse2neon_mm_set_flush_zero_mode
#define _MM_GET_DENORMALS_ZERO_MODE _sse2neon_mm_get_denormals_zero_mode
#define _MM_SET_DENORMALS_ZERO_MODE _sse2neon_mm_set_denormals_zero_mode

// 函数声明
// SSE
FORCE_INLINE unsigned int _MM_GET_ROUNDING_MODE(void);
FORCE_INLINE __m128 _mm_move_ss(__m128, __m128);
FORCE_INLINE __m128 _mm_or_ps(__m128, __m128);
FORCE_INLINE __m128 _mm_set_ps1(float);
FORCE_INLINE __m128 _mm_setzero_ps(void);
// SSE2
FORCE_INLINE __m128i _mm_and_si128(__m128i, __m128i);
FORCE_INLINE __m128i _mm_castps_si128(__m128);
FORCE_INLINE __m128i _mm_cmpeq_epi32(__m128i, __m128i);
FORCE_INLINE __m128i _mm_cvtps_epi32(__m128);
FORCE_INLINE __m128d _mm_move_sd(__m128d, __m128d);
FORCE_INLINE __m128i _mm_or_si128(__m128i, __m128i);
FORCE_INLINE __m128i _mm_set_epi32(int, int, int, int);
FORCE_INLINE __m128i _mm_set_epi64x(int64_t, int64_t);
FORCE_INLINE __m128d _mm_set_pd(double, double);
// 定义一个强制内联函数，设置一个包含相同值的 32 位整数的 128 位整数
FORCE_INLINE __m128i _mm_set1_epi32(int);

// 定义一个强制内联函数，设置一个包含全零的 128 位整数
FORCE_INLINE __m128i _mm_setzero_si128(void);

// 定义一个强制内联函数，对 128 位双精度浮点数向上取整
FORCE_INLINE __m128d _mm_ceil_pd(__m128d);

// 定义一个强制内联函数，对 128 位单精度浮点数向上取整
FORCE_INLINE __m128 _mm_ceil_ps(__m128);

// 定义一个强制内联函数，对 128 位双精度浮点数向下取整
FORCE_INLINE __m128d _mm_floor_pd(__m128d);

// 定义一个强制内联函数，对 128 位单精度浮点数向下取整
FORCE_INLINE __m128 _mm_floor_ps(__m128);

// 定义一个强制内联函数，对 128 位双精度浮点数按指定模式四舍五入
FORCE_INLINE __m128d _mm_round_pd(__m128d, int);

// 定义一个强制内联函数，对 128 位单精度浮点数按指定模式四舍五入
FORCE_INLINE __m128 _mm_round_ps(__m128, int);

// 定义一个强制内联函数，计算 8 位无符号整数的 CRC32 校验值
FORCE_INLINE uint32_t _mm_crc32_u8(uint32_t, uint8_t);

// 对于缺乏特定类型支持的编译器的向后兼容性

// 如果是旧版本的 gcc，且不是 clang，且满足以下条件，则定义一个强制内联函数
#if defined(__GNUC__) && !defined(__clang__) &&                        \
    ((__GNUC__ <= 13 && defined(__arm__)) ||                           \
     (__GNUC__ == 10 && __GNUC_MINOR__ < 3 && defined(__aarch64__)) || \
     (__GNUC__ <= 9 && defined(__aarch64__)))
FORCE_INLINE uint8x16x4_t _sse2neon_vld1q_u8_x4(const uint8_t *p)
{
    uint8x16x4_t ret;
    ret.val[0] = vld1q_u8(p + 0);
    ret.val[1] = vld1q_u8(p + 16);
    ret.val[2] = vld1q_u8(p + 32);
    ret.val[3] = vld1q_u8(p + 48);
    return ret;
}
#else
// 包装 vld1q_u8_x4
FORCE_INLINE uint8x16x4_t _sse2neon_vld1q_u8_x4(const uint8_t *p)
{
    return vld1q_u8_x4(p);
}
#endif

// 如果不是 aarch64 架构，也不是 ARM64 架构
// 模拟 vaddv u8 变体
FORCE_INLINE uint8_t _sse2neon_vaddv_u8(uint8x8_t v8)
{
    const uint64x1_t v1 = vpaddl_u32(vpaddl_u16(vpaddl_u8(v8)));
    return vget_lane_u8(vreinterpret_u8_u64(v1), 0);
}
#else
// 包装 vaddv_u8
FORCE_INLINE uint8_t _sse2neon_vaddv_u8(uint8x8_t v8)
{
    return vaddv_u8(v8);
}
#endif

// 如果不是 aarch64 架构，也不是 ARM64 架构
// 模拟 vaddvq u8 变体
FORCE_INLINE uint8_t _sse2neon_vaddvq_u8(uint8x16_t a)
{
    uint8x8_t tmp = vpadd_u8(vget_low_u8(a), vget_high_u8(a));
    uint8_t res = 0;
    for (int i = 0; i < 8; ++i)
        res += tmp[i];
    return res;
}
#else
// 包装 vaddvq_u8
FORCE_INLINE uint8_t _sse2neon_vaddvq_u8(uint8x16_t a)
{
    return vaddvq_u8(a);
}
#endif
#if !defined(__aarch64__) && !defined(_M_ARM64)
// 如果不是 aarch64 架构和 _M_ARM64 宏未定义，则使用下面的代码块
/* emulate vaddvq u16 variant */
// 模拟实现 vaddvq u16 变体
FORCE_INLINE uint16_t _sse2neon_vaddvq_u16(uint16x8_t a)
{
    // 将 8 个 16 位整数相加并拓展为 4 个 32 位整数
    uint32x4_t m = vpaddlq_u16(a);
    // 将 4 个 32 位整数相加并拓展为 2 个 64 位整数
    uint64x2_t n = vpaddlq_u32(m);
    // 将 2 个 64 位整数相加
    uint64x1_t o = vget_low_u64(n) + vget_high_u64(n);

    // 返回相加结果的第一个 32 位整数
    return vget_lane_u32((uint32x2_t) o, 0);
}
#else
// 如果是 aarch64 架构或者 _M_ARM64 宏已定义，则使用下面的代码块
// Wraps vaddvq_u16
// 包装 vaddvq_u16 函数
FORCE_INLINE uint16_t _sse2neon_vaddvq_u16(uint16x8_t a)
{
    // 直接调用 vaddvq_u16 函数
    return vaddvq_u16(a);
}
#endif
/* 函数命名约定
 * SSE指令的命名约定很简单。一个通用的SSE指令函数如下所示：
 *   _mm_<name>_<data_type>
 *
 * 这个格式的部分如下所示：
 * 1. <name> 描述指令执行的操作
 * 2. <data_type> 标识函数主要参数的数据类型
 *
 * 最后一个部分，<data_type>，有点复杂。它标识输入值的内容，并且可以设置为以下任意值：
 * + ps - 向量包含浮点数（ps代表打包的单精度）
 * + pd - 向量包含双精度数（pd代表打包的双精度）
 * + epi8/epi16/epi32/epi64 - 向量包含8位/16位/32位/64位有符号整数
 * + epu8/epu16/epu32/epu64 - 向量包含8位/16位/32位/64位无符号整数
 * + si128 - 未指定的128位向量或256位向量
 * + m128/m128i/m128d - 当输入向量类型与返回向量类型不同时标识输入向量类型
 *
 * 例如，_mm_setzero_ps。_mm表示函数返回一个128位向量。末尾的_ps表示参数向量包含浮点数。
 *
 * 完整的例子：字节重排 - pshufb (_mm_shuffle_epi8)
 *   // 设置打包的16位整数。128位，每16位8个短整数
 *   __m128i v_in = _mm_setr_epi16(1, 2, 3, 4, 5, 6, 7, 8);
 *   // 设置打包的8位整数
 *   // 128位，每8位16个字符
 *   __m128i v_perm = _mm_setr_epi8(1, 0,  2,  3, 8, 9, 10, 11,
 *                                  4, 5, 12, 13, 6, 7, 14, 15);
 *   // 重排打包的8位整数
 *   __m128i v_out = _mm_shuffle_epi8(v_in, v_perm); // pshufb
 */

/* 用于_mm_prefetch的常量。 */
enum _mm_hint {
    _MM_HINT_NTA = 0, /* 将数据加载到L1和L2缓存中，并标记为NTA */
    _MM_HINT_T0 = 1,  /* 将数据加载到L1和L2缓存中 */
    # 定义常量 _MM_HINT_T1，表示将数据仅加载到 L2 缓存中
    _MM_HINT_T1 = 2,  /* load data to L2 cache only */
    # 定义常量 _MM_HINT_T2，表示将数据加载到 L2 缓存中，并标记为NTA（Non-Temporal Access）
    _MM_HINT_T2 = 3,  /* load data to L2 cache only, mark it as NTA */
};

// 将位字段映射到 FPCR（浮点控制寄存器）
typedef struct {
    uint16_t res0;  // 保留字段
    uint8_t res1 : 6;  // 保留字段
    uint8_t bit22 : 1;  // 第22位
    uint8_t bit23 : 1;  // 第23位
    uint8_t bit24 : 1;  // 第24位
    uint8_t res2 : 7;  // 保留字段
#if defined(__aarch64__) || defined(_M_ARM64)
    uint32_t res3;  // 保留字段
#endif
} fpcr_bitfield;

// 将 a 的高64位放入结果的低端，将 b 的低64位放入结果的高端
FORCE_INLINE __m128 _mm_shuffle_ps_1032(__m128 a, __m128 b)
{
    float32x2_t a32 = vget_high_f32(vreinterpretq_f32_m128(a));  // 获取 a 的高32位
    float32x2_t b10 = vget_low_f32(vreinterpretq_f32_m128(b));  // 获取 b 的低32位
    return vreinterpretq_m128_f32(vcombine_f32(a32, b10));  // 合并结果为一个新的 __m128 类型
}

// 取 a 的低两个32位值并交换它们，放入结果的高端；取 b 的高两个32位值并交换它们，放入结果的低端
FORCE_INLINE __m128 _mm_shuffle_ps_2301(__m128 a, __m128 b)
{
    float32x2_t a01 = vrev64_f32(vget_low_f32(vreinterpretq_f32_m128(a)));  // 取 a 的低32位并交换
    float32x2_t b23 = vrev64_f32(vget_high_f32(vreinterpretq_f32_m128(b)));  // 取 b 的高32位并交换
    return vreinterpretq_m128_f32(vcombine_f32(a01, b23));  // 合并结果为一个新的 __m128 类型
}

// 取 a 的低32位值并放入结果的高端；取 b 的高32位值并放入结果的低端
FORCE_INLINE __m128 _mm_shuffle_ps_0321(__m128 a, __m128 b)
{
    float32x2_t a21 = vget_high_f32(
        vextq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(a), 3));  // 取 a 的低32位并放入结果的高端
    float32x2_t b03 = vget_low_f32(
        vextq_f32(vreinterpretq_f32_m128(b), vreinterpretq_f32_m128(b), 3));  // 取 b 的高32位并放入结果的低端
    return vreinterpretq_m128_f32(vcombine_f32(a21, b03));  // 合并结果为一个新的 __m128 类型
}

// 取 a 的低32位值并放入结果的低端；取 b 的高32位值并放入结果的高端
FORCE_INLINE __m128 _mm_shuffle_ps_2103(__m128 a, __m128 b)
{
    float32x2_t a03 = vget_low_f32(
        vextq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(a), 3));  // 取 a 的低32位并放入结果的低端
    float32x2_t b21 = vget_high_f32(
        vextq_f32(vreinterpretq_f32_m128(b), vreinterpretq_f32_m128(b), 3));  // 取 b 的高32位并放入结果的高端
    return vreinterpretq_m128_f32(vcombine_f32(a03, b21));  // 合并结果为一个新的 __m128 类型
}

// 未完待续，缺少了函数体
    # 从128位寄存器a中获取低位的两个32位浮点数，存储到float32x2_t类型的变量a10中
    float32x2_t a10 = vget_low_f32(vreinterpretq_f32_m128(a));
    # 从128位寄存器b中获取低位的两个32位浮点数，存储到float32x2_t类型的变量b10中
    float32x2_t b10 = vget_low_f32(vreinterpretq_f32_m128(b));
    # 将两个32位浮点数的向量a10和b10合并成一个128位寄存器，并返回结果
    return vreinterpretq_m128_f32(vcombine_f32(a10, b10));
// 将参数 a 的低64位和参数 b 的高64位组合成一个新的__m128类型的数据
FORCE_INLINE __m128 _mm_shuffle_ps_1001(__m128 a, __m128 b)
{
    // 将参数 a 转换为float32x2_t类型，然后进行反转，取低32位
    float32x2_t a01 = vrev64_f32(vget_low_f32(vreinterpretq_f32_m128(a)));
    // 取参数 b 的低32位
    float32x2_t b10 = vget_low_f32(vreinterpretq_f32_m128(b));
    // 将 a01 和 b10 组合成一个新的__m128类型的数据
    return vreinterpretq_m128_f32(vcombine_f32(a01, b10));
}

// 将参数 a 的低64位和参数 b 的低64位组合成一个新的__m128类型的数据
FORCE_INLINE __m128 _mm_shuffle_ps_0101(__m128 a, __m128 b)
{
    // 将参数 a 转换为float32x2_t类型，然后进行反转，取低32位
    float32x2_t a01 = vrev64_f32(vget_low_f32(vreinterpretq_f32_m128(a)));
    // 将参数 b 转换为float32x2_t类型，然后进行反转，取低32位
    float32x2_t b01 = vrev64_f32(vget_low_f32(vreinterpretq_f32_m128(b)));
    // 将 a01 和 b01 组合成一个新的__m128类型的数据
    return vreinterpretq_m128_f32(vcombine_f32(a01, b01));
}

// 将参数 a 的高64位和参数 b 的低64位组合成一个新的__m128类型的数据
FORCE_INLINE __m128 _mm_shuffle_ps_3210(__m128 a, __m128 b)
{
    // 取参数 a 的低32位
    float32x2_t a10 = vget_low_f32(vreinterpretq_f32_m128(a));
    // 取参数 b 的高32位
    float32x2_t b32 = vget_high_f32(vreinterpretq_f32_m128(b));
    // 将 a10 和 b32 组合成一个新的__m128类型的数据
    return vreinterpretq_m128_f32(vcombine_f32(a10, b32));
}

// 将参数 a 的低32位和参数 b 的低32位组合成一个新的__m128类型的数据
FORCE_INLINE __m128 _mm_shuffle_ps_0011(__m128 a, __m128 b)
{
    // 取参数 a 的低32位，然后复制成一个新的float32x2_t类型
    float32x2_t a11 = vdup_lane_f32(vget_low_f32(vreinterpretq_f32_m128(a)), 1);
    // 取参数 b 的低32位，然后复制成一个新的float32x2_t类型
    float32x2_t b00 = vdup_lane_f32(vget_low_f32(vreinterpretq_f32_m128(b)), 0);
    // 将 a11 和 b00 组合成一个新的__m128类型的数据
    return vreinterpretq_m128_f32(vcombine_f32(a11, b00));
}

// 将参数 a 的高32位和参数 b 的低32位组合成一个新的__m128类型的数据
FORCE_INLINE __m128 _mm_shuffle_ps_0022(__m128 a, __m128 b)
{
    // 取参数 a 的高32位，然后复制成一个新的float32x2_t类型
    float32x2_t a22 = vdup_lane_f32(vget_high_f32(vreinterpretq_f32_m128(a)), 0);
    // 取参数 b 的低32位，然后复制成一个新的float32x2_t类型
    float32x2_t b00 = vdup_lane_f32(vget_low_f32(vreinterpretq_f32_m128(b)), 0);
    // 将 a22 和 b00 组合成一个新的__m128类型的数据
    return vreinterpretq_m128_f32(vcombine_f32(a22, b00));
}

// 将参数 a 的低32位和参数 b 的高32位组合成一个新的__m128类型的数据
FORCE_INLINE __m128 _mm_shuffle_ps_2200(__m128 a, __m128 b)
{
    // 取参数 a 的低32位，然后复制成一个新的float32x2_t类型
    float32x2_t a00 = vdup_lane_f32(vget_low_f32(vreinterpretq_f32_m128(a)), 0);
    // 取参数 b 的高32位，然后复制成一个新的float32x2_t类型
    float32x2_t b22 = vdup_lane_f32(vget_high_f32(vreinterpretq_f32_m128(b)), 0);
    // 将 a00 和 b22 组合成一个新的__m128类型的数据
    return vreinterpretq_m128_f32(vcombine_f32(a00, b22));
}

// 将参数 a 的低32位和参数 b 的高32位组合成一个新的__m128类型的数据
FORCE_INLINE __m128 _mm_shuffle_ps_3202(__m128 a, __m128 b)
{
    // 取参数 a 的第0个32位浮点数
    float32_t a0 = vgetq_lane_f32(vreinterpretq_f32_m128(a), 0);
    // 取参数 a 的高32位，然后复制成一个新的float32x2_t类型
    float32x2_t a22 = vdup_lane_f32(vget_high_f32(vreinterpretq_f32_m128(a)), 0);
    # 创建一个包含两个32位浮点数的向量，将a0插入到a22的第一个位置
    float32x2_t a02 = vset_lane_f32(a0, a22, 1); /* TODO: use vzip ?*/
    # 从一个128位寄存器中获取高位的两个32位浮点数
    float32x2_t b32 = vget_high_f32(vreinterpretq_f32_m128(b));
    # 将两个32位浮点数的向量合并成一个128位寄存器，并返回结果
    return vreinterpretq_m128_f32(vcombine_f32(a02, b32));
}

// 定义一个名为 _mm_shuffle_ps_1133 的函数，参数为两个 __m128 类型的变量 a 和 b
FORCE_INLINE __m128 _mm_shuffle_ps_1133(__m128 a, __m128 b)
{
    // 从 a 中获取高位的单精度浮点数，复制成一个 float32x2_t 类型的变量 a33
    float32x2_t a33 =
        vdup_lane_f32(vget_high_f32(vreinterpretq_f32_m128(a)), 1);
    // 从 b 中获取低位的单精度浮点数，复制成一个 float32x2_t 类型的变量 b11
    float32x2_t b11 = vdup_lane_f32(vget_low_f32(vreinterpretq_f32_m128(b)), 1);
    // 将 a33 和 b11 合并成一个 __m128 类型的变量并返回
    return vreinterpretq_m128_f32(vcombine_f32(a33, b11));
}

// 定义一个名为 _mm_shuffle_ps_2010 的函数，参数为两个 __m128 类型的变量 a 和 b
FORCE_INLINE __m128 _mm_shuffle_ps_2010(__m128 a, __m128 b)
{
    // 从 a 中获取低位的单精度浮点数，复制成一个 float32x2_t 类型的变量 a10
    float32x2_t a10 = vget_low_f32(vreinterpretq_f32_m128(a));
    // 从 b 中获取第二个单精度浮点数，复制成一个 float32_t 类型的变量 b2
    float32_t b2 = vgetq_lane_f32(vreinterpretq_f32_m128(b), 2);
    // 从 b 中获取低位的单精度浮点数，复制成一个 float32x2_t 类型的变量 b00
    float32x2_t b00 = vdup_lane_f32(vget_low_f32(vreinterpretq_f32_m128(b)), 0);
    // 将 b2 插入到 b00 的第二个位置，得到一个 float32x2_t 类型的变量 b20
    float32x2_t b20 = vset_lane_f32(b2, b00, 1);
    // 将 a10 和 b20 合并成一个 __m128 类型的变量并返回
    return vreinterpretq_m128_f32(vcombine_f32(a10, b20));
}

// 定义一个名为 _mm_shuffle_ps_2001 的函数，参数为两个 __m128 类型的变量 a 和 b
FORCE_INLINE __m128 _mm_shuffle_ps_2001(__m128 a, __m128 b)
{
    // 从 a 中获取低位的单精度浮点数，反转后复制成一个 float32x2_t 类型的变量 a01
    float32x2_t a01 = vrev64_f32(vget_low_f32(vreinterpretq_f32_m128(a)));
    // 从 b 中获取第二个单精度浮点数，复制成一个 float32_t 类型的变量 b2
    float32_t b2 = vgetq_lane_f32(b, 2);
    // 从 b 中获取低位的单精度浮点数，复制成一个 float32x2_t 类型的变量 b00
    float32x2_t b00 = vdup_lane_f32(vget_low_f32(vreinterpretq_f32_m128(b)), 0);
    // 将 b2 插入到 b00 的第二个位置，得到一个 float32x2_t 类型的变量 b20
    float32x2_t b20 = vset_lane_f32(b2, b00, 1);
    // 将 a01 和 b20 合并成一个 __m128 类型的变量并返回
    return vreinterpretq_m128_f32(vcombine_f32(a01, b20));
}

// 定义一个名为 _mm_shuffle_ps_2032 的函数，参数为两个 __m128 类型的变量 a 和 b
FORCE_INLINE __m128 _mm_shuffle_ps_2032(__m128 a, __m128 b)
{
    // 从 a 中获取高位的单精度浮点数，复制成一个 float32x2_t 类型的变量 a32
    float32x2_t a32 = vget_high_f32(vreinterpretq_f32_m128(a));
    // 从 b 中获取第二个单精度浮点数，复制成一个 float32_t 类型的变量 b2
    float32_t b2 = vgetq_lane_f32(b, 2);
    // 从 b 中获取低位的单精度浮点数，复制成一个 float32x2_t 类型的变量 b00
    float32x2_t b00 = vdup_lane_f32(vget_low_f32(vreinterpretq_f32_m128(b)), 0);
    // 将 b2 插入到 b00 的第二个位置，得到一个 float32x2_t 类型的变量 b20
    float32x2_t b20 = vset_lane_f32(b2, b00, 1);
    // 将 a32 和 b20 合并成一个 __m128 类型的变量并返回
    return vreinterpretq_m128_f32(vcombine_f32(a32, b20));
}

// 对于 MSVC，我们只检查是否为 ARM64，因为 WoA 支持的每个 ARM64 处理器都有加密扩展。
// 如果将来发生变化，可以通过运行时方法进行验证：
// IsProcessorFeaturePresent(PF_ARM_V8_CRYPTO_INSTRUCTIONS_AVAILABLE)
#if (defined(_M_ARM64) && !defined(__clang__)) || \
    (defined(__ARM_FEATURE_CRYPTO) &&             \
     (defined(__aarch64__) || __has_builtin(__builtin_arm_crypto_vmullp64)))
// 包装 vmull_p64
FORCE_INLINE uint64x2_t _sse2neon_vmull_p64(uint64x1_t _a, uint64x1_t _b)
{
    # 从一个64位向量中获取指定位置的64位数据，并存储到变量a中
    poly64_t a = vget_lane_p64(vreinterpret_p64_u64(_a), 0);
    # 从一个64位向量中获取指定位置的64位数据，并存储到变量b中
    poly64_t b = vget_lane_p64(vreinterpret_p64_u64(_b), 0);
#if defined(_MSC_VER)
    # 如果定义了_MSC_VER，使用a1和b1创建64位整数向量，然后返回a1和b1的64位整数乘积的128位整数向量
    __n64 a1 = {a}, b1 = {b};
    return vreinterpretq_u64_p128(vmull_p64(a1, b1));
#else
    # 如果未定义_MSC_VER，直接返回a和b的64位整数乘积的128位整数向量
    return vreinterpretq_u64_p128(vmull_p64(a, b));
#endif
}
#else  // ARMv7 polyfill
// ARMv7/some A64 lacks vmull_p64, but it has vmull_p8.
//
// vmull_p8 calculates 8 8-bit->16-bit polynomial multiplies, but we need a
// 64-bit->128-bit polynomial multiply.
//
// It needs some work and is somewhat slow, but it is still faster than all
// known scalar methods.
//
// Algorithm adapted to C from
// https://www.workofard.com/2017/07/ghash-for-low-end-cores/, which is adapted
// from "Fast Software Polynomial Multiplication on ARM Processors Using the
// NEON Engine" by Danilo Camara, Conrado Gouvea, Julio Lopez and Ricardo Dahab
static uint64x2_t _sse2neon_vmull_p64(uint64x1_t _a, uint64x1_t _b)
{
    # 将输入的64位整数向量_a和_b转换为8位多项式向量a和b
    poly8x8_t a = vreinterpret_p8_u64(_a);
    poly8x8_t b = vreinterpret_p8_u64(_b);

    // Masks
    # 创建掩码
    uint8x16_t k48_32 = vcombine_u8(vcreate_u8(0x0000ffffffffffff),
                                    vcreate_u8(0x00000000ffffffff));
    uint8x16_t k16_00 = vcombine_u8(vcreate_u8(0x000000000000ffff),
                                    vcreate_u8(0x0000000000000000));

    // Do the multiplies, rotating with vext to get all combinations
    # 进行乘法运算，使用vext进行旋转以获取所有组合
    uint8x16_t d = vreinterpretq_u8_p16(vmull_p8(a, b));  // D = A0 * B0
    uint8x16_t e =
        vreinterpretq_u8_p16(vmull_p8(a, vext_p8(b, b, 1)));  // E = A0 * B1
    uint8x16_t f =
        vreinterpretq_u8_p16(vmull_p8(vext_p8(a, a, 1), b));  // F = A1 * B0
    uint8x16_t g =
        vreinterpretq_u8_p16(vmull_p8(a, vext_p8(b, b, 2)));  // G = A0 * B2
    uint8x16_t h =
        vreinterpretq_u8_p16(vmull_p8(vext_p8(a, a, 2), b));  // H = A2 * B0
    uint8x16_t i =
        vreinterpretq_u8_p16(vmull_p8(a, vext_p8(b, b, 3)));  // I = A0 * B3
    uint8x16_t j =
        vreinterpretq_u8_p16(vmull_p8(vext_p8(a, a, 3), b));  // J = A3 * B0
    // 将向量 a 和向量 b 中的元素进行逐个相乘，然后将结果转换为 8 位无符号整数向量
    uint8x16_t k =
        vreinterpretq_u8_p16(vmull_p8(a, vext_p8(b, b, 4)));  // L = A0 * B4

    // 对向量 e 和向量 f 进行按位异或操作，得到新的向量 l
    uint8x16_t l = veorq_u8(e, f);  // L = E + F
    // 对向量 g 和向量 h 进行按位异或操作，得到新的向量 m
    uint8x16_t m = veorq_u8(g, h);  // M = G + H
    // 对向量 i 和向量 j 进行按位异或操作，得到新的向量 n
    uint8x16_t n = veorq_u8(i, j);  // N = I + J

    // 交错合并向量，使用 vzip1 和 vzip2 防止 Clang 发出 TBL 指令
#if defined(__aarch64__)
    // 如果是 aarch64 架构，则使用 vzip1q_u64 和 vzip2q_u64 函数进行操作
    uint8x16_t lm_p0 = vreinterpretq_u8_u64(
        vzip1q_u64(vreinterpretq_u64_u8(l), vreinterpretq_u64_u8(m)));
    uint8x16_t lm_p1 = vreinterpretq_u8_u64(
        vzip2q_u64(vreinterpretq_u64_u8(l), vreinterpretq_u64_u8(m)));
    uint8x16_t nk_p0 = vreinterpretq_u8_u64(
        vzip1q_u64(vreinterpretq_u64_u8(n), vreinterpretq_u64_u8(k)));
    uint8x16_t nk_p1 = vreinterpretq_u8_u64(
        vzip2q_u64(vreinterpretq_u64_u8(n), vreinterpretq_u64_u8(k)));
#else
    // 如果不是 aarch64 架构，则使用 vcombine_u8 和 vget_low_u8/vget_high_u8 函数进行操作
    uint8x16_t lm_p0 = vcombine_u8(vget_low_u8(l), vget_low_u8(m));
    uint8x16_t lm_p1 = vcombine_u8(vget_high_u8(l), vget_high_u8(m));
    uint8x16_t nk_p0 = vcombine_u8(vget_low_u8(n), vget_low_u8(k));
    uint8x16_t nk_p1 = vcombine_u8(vget_high_u8(n), vget_high_u8(k));
#endif
    // t0 = (L) (P0 + P1) << 8
    // t1 = (M) (P2 + P3) << 16
    // 计算 t0 和 t1 的值
    uint8x16_t t0t1_tmp = veorq_u8(lm_p0, lm_p1);
    uint8x16_t t0t1_h = vandq_u8(lm_p1, k48_32);
    uint8x16_t t0t1_l = veorq_u8(t0t1_tmp, t0t1_h);

    // t2 = (N) (P4 + P5) << 24
    // t3 = (K) (P6 + P7) << 32
    // 计算 t2 和 t3 的值
    uint8x16_t t2t3_tmp = veorq_u8(nk_p0, nk_p1);
    uint8x16_t t2t3_h = vandq_u8(nk_p1, k16_00);
    uint8x16_t t2t3_l = veorq_u8(t2t3_tmp, t2t3_h);

    // De-interleave
#if defined(__aarch64__)
    // 如果是 aarch64 架构，则使用 vuzp1q_u64 和 vuzp2q_u64 函数进行操作
    uint8x16_t t0 = vreinterpretq_u8_u64(
        vuzp1q_u64(vreinterpretq_u64_u8(t0t1_l), vreinterpretq_u64_u8(t0t1_h)));
    uint8x16_t t1 = vreinterpretq_u8_u64(
        vuzp2q_u64(vreinterpretq_u64_u8(t0t1_l), vreinterpretq_u64_u8(t0t1_h)));
    uint8x16_t t2 = vreinterpretq_u8_u64(
        vuzp1q_u64(vreinterpretq_u64_u8(t2t3_l), vreinterpretq_u64_u8(t2t3_h)));
    uint8x16_t t3 = vreinterpretq_u8_u64(
        vuzp2q_u64(vreinterpretq_u64_u8(t2t3_l), vreinterpretq_u64_u8(t2t3_h)));
#else
    // 如果不是 aarch64 架构，则使用 vcombine_u8 和 vget_low_u8/vget_high_u8 函数进行操作
    uint8x16_t t1 = vcombine_u8(vget_high_u8(t0t1_l), vget_high_u8(t0t1_h));
    uint8x16_t t0 = vcombine_u8(vget_low_u8(t0t1_l), vget_low_u8(t0t1_h));
    uint8x16_t t3 = vcombine_u8(vget_high_u8(t2t3_l), vget_high_u8(t2t3_h));
    # 将两个 uint8x8_t 类型的向量合并成一个 uint8x16_t 类型的向量
    uint8x16_t t2 = vcombine_u8(vget_low_u8(t2t3_l), vget_low_u8(t2t3_h));
// 结束条件判断，如果定义了宏 #endif
#endif

    // 移位交叉乘积
    uint8x16_t t0_shift = vextq_u8(t0, t0, 15);  // t0 << 8
    uint8x16_t t1_shift = vextq_u8(t1, t1, 14);  // t1 << 16
    uint8x16_t t2_shift = vextq_u8(t2, t2, 13);  // t2 << 24
    uint8x16_t t3_shift = vextq_u8(t3, t3, 12);  // t3 << 32

    // 累加乘积
    uint8x16_t cross1 = veorq_u8(t0_shift, t1_shift);
    uint8x16_t cross2 = veorq_u8(t2_shift, t3_shift);
    uint8x16_t mix = veorq_u8(d, cross1);
    uint8x16_t r = veorq_u8(mix, cross2);
    return vreinterpretq_u64_u8(r);
}
#endif  // ARMv7 polyfill

// C 等效代码:
//   __m128i _mm_shuffle_epi32_default(__m128i a,
//                                     __constrange(0, 255) int imm) {
//       __m128i ret;
//       ret[0] = a[imm        & 0x3];   ret[1] = a[(imm >> 2) & 0x3];
//       ret[2] = a[(imm >> 4) & 0x03];  ret[3] = a[(imm >> 6) & 0x03];
//       return ret;
//   }
// 定义宏 _mm_shuffle_epi32_default，实现对__m128i类型数据的操作
#define _mm_shuffle_epi32_default(a, imm)                                   \
    vreinterpretq_m128i_s32(vsetq_lane_s32(                                 \
        vgetq_lane_s32(vreinterpretq_s32_m128i(a), ((imm) >> 6) & 0x3),     \
        vsetq_lane_s32(                                                     \
            vgetq_lane_s32(vreinterpretq_s32_m128i(a), ((imm) >> 4) & 0x3), \
            vsetq_lane_s32(vgetq_lane_s32(vreinterpretq_s32_m128i(a),       \
                                          ((imm) >> 2) & 0x3),              \
                           vmovq_n_s32(vgetq_lane_s32(                      \
                               vreinterpretq_s32_m128i(a), (imm) & (0x3))), \
                           1),                                              \
            2),                                                             \
        3))

// 将a的高64位放入结果的低端
// 将a的低64位放入结果的高端
FORCE_INLINE __m128i _mm_shuffle_epi_1032(__m128i a)
    # 从128位寄存器a中获取高位的两个32位整数，存储到a32中
    int32x2_t a32 = vget_high_s32(vreinterpretq_s32_m128i(a));
    # 从128位寄存器a中获取低位的两个32位整数，存储到a10中
    int32x2_t a10 = vget_low_s32(vreinterpretq_s32_m128i(a));
    # 将a32和a10合并成一个新的128位寄存器，并返回
    return vreinterpretq_m128i_s32(vcombine_s32(a32, a10));
// 从参数 a 中取出低两个32位的值并交换它们，放在结果的低端
// 从参数 a 中取出高两个32位的值并交换它们，放在结果的高端
FORCE_INLINE __m128i _mm_shuffle_epi_2301(__m128i a)
{
    int32x2_t a01 = vrev64_s32(vget_low_s32(vreinterpretq_s32_m128i(a)));
    int32x2_t a23 = vrev64_s32(vget_high_s32(vreinterpretq_s32_m128i(a)));
    return vreinterpretq_m128i_s32(vcombine_s32(a01, a23));
}

// 将参数 a 的最低32位旋转到最高32位，并将其余部分向下移位
FORCE_INLINE __m128i _mm_shuffle_epi_0321(__m128i a)
{
    return vreinterpretq_m128i_s32(
        vextq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(a), 1));
}

// 将参数 a 的最高32位旋转到最低32位，并将其余部分向上移位
FORCE_INLINE __m128i _mm_shuffle_epi_2103(__m128i a)
{
    return vreinterpretq_m128i_s32(
        vextq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(a), 3));
}

// 获取参数 a 的低64位，并将其放在高64位
// 获取参数 a 的低64位，并将其放在低64位
FORCE_INLINE __m128i _mm_shuffle_epi_1010(__m128i a)
{
    int32x2_t a10 = vget_low_s32(vreinterpretq_s32_m128i(a));
    return vreinterpretq_m128i_s32(vcombine_s32(a10, a10));
}

// 获取参数 a 的低64位，交换第0和第1个元素，并将其放在低64位
// 获取参数 a 的低64位，并将其放在高64位
FORCE_INLINE __m128i _mm_shuffle_epi_1001(__m128i a)
{
    int32x2_t a01 = vrev64_s32(vget_low_s32(vreinterpretq_s32_m128i(a)));
    int32x2_t a10 = vget_low_s32(vreinterpretq_s32_m128i(a));
    return vreinterpretq_m128i_s32(vcombine_s32(a01, a10));
}

// 获取参数 a 的低64位，交换第0和第1个元素，并将其放在高64位
// 获取参数 a 的低64位，交换第0和第1个元素，并将其放在低64位
FORCE_INLINE __m128i _mm_shuffle_epi_0101(__m128i a)
{
    # 将输入向量a的低位32位进行反转，并存储到a01中
    int32x2_t a01 = vrev64_s32(vget_low_s32(vreinterpretq_s32_m128i(a)));
    # 将a01中的数据转换为128位整数向量，并将a01中的数据复制到高位和低位，返回结果
    return vreinterpretq_m128i_s32(vcombine_s32(a01, a01));
// 定义一个强制内联函数，用于对__m128i类型的数据进行特定的元素重排操作
FORCE_INLINE __m128i _mm_shuffle_epi_2211(__m128i a)
{
    // 从输入数据a中提取低位的两个32位整数，并复制到一个新的int32x2_t类型变量a11中
    int32x2_t a11 = vdup_lane_s32(vget_low_s32(vreinterpretq_s32_m128i(a)), 1);
    // 从输入数据a中提取高位的两个32位整数，并复制到一个新的int32x2_t类型变量a22中
    int32x2_t a22 = vdup_lane_s32(vget_high_s32(vreinterpretq_s32_m128i(a)), 0);
    // 将a11和a22合并成一个新的__m128i类型数据，并返回
    return vreinterpretq_m128i_s32(vcombine_s32(a11, a22));
}

// 定义一个强制内联函数，用于对__m128i类型的数据进行特定的元素重排操作
FORCE_INLINE __m128i _mm_shuffle_epi_0122(__m128i a)
{
    // 从输入数据a中提取高位的两个32位整数，并复制到一个新的int32x2_t类型变量a22中
    int32x2_t a22 = vdup_lane_s32(vget_high_s32(vreinterpretq_s32_m128i(a)), 0);
    // 从输入数据a中提取低位的两个32位整数，并进行翻转后复制到一个新的int32x2_t类型变量a01中
    int32x2_t a01 = vrev64_s32(vget_low_s32(vreinterpretq_s32_m128i(a)));
    // 将a22和a01合并成一个新的__m128i类型数据，并返回
    return vreinterpretq_m128i_s32(vcombine_s32(a22, a01));
}

// 定义一个强制内联函数，用于对__m128i类型的数据进行特定的元素重排操作
FORCE_INLINE __m128i _mm_shuffle_epi_3332(__m128i a)
{
    // 从输入数据a中提取高位的两个32位整数，并复制到一个新的int32x2_t类型变量a32中
    int32x2_t a32 = vget_high_s32(vreinterpretq_s32_m128i(a));
    // 从输入数据a中提取高位的两个32位整数，并复制到一个新的int32x2_t类型变量a33中
    int32x2_t a33 = vdup_lane_s32(vget_high_s32(vreinterpretq_s32_m128i(a)), 1);
    // 将a32和a33合并成一个新的__m128i类型数据，并返回
    return vreinterpretq_m128i_s32(vcombine_s32(a32, a33));
}

// 根据宏定义判断处理器架构，选择不同的实现方式
#if defined(__aarch64__) || defined(_M_ARM64)
// 如果是ARM64架构，则定义_mm_shuffle_epi32_splat为特定的内联函数
#define _mm_shuffle_epi32_splat(a, imm) \
    vreinterpretq_m128i_s32(vdupq_laneq_s32(vreinterpretq_s32_m128i(a), (imm)))
#else
// 如果不是ARM64架构，则定义_mm_shuffle_epi32_splat为特定的内联函数
#define _mm_shuffle_epi32_splat(a, imm) \
    vreinterpretq_m128i_s32(            \
        vdupq_n_s32(vgetq_lane_s32(vreinterpretq_s32_m128i(a), (imm))))
#endif

// 定义一个宏，用于实现对__m128类型的数据进行特定的元素重排操作
#define _mm_shuffle_ps_default(a, b, imm)                                      \
    # 使用 SIMD 指令对两个 128 位浮点数寄存器进行重新解释和重新排列
    vreinterpretq_m128_f32(vsetq_lane_f32(                                     \
        # 从第二个参数寄存器中取出指定位置的 32 位浮点数，并插入到第一个参数寄存器的指定位置
        vgetq_lane_f32(vreinterpretq_f32_m128(b), ((imm) >> 6) & 0x3),         \
        # 从第二个参数寄存器中取出指定位置的 32 位浮点数，并插入到第一个参数寄存器的指定位置
        vsetq_lane_f32(                                                        \
            vgetq_lane_f32(vreinterpretq_f32_m128(b), ((imm) >> 4) & 0x3),     \
            # 从第二个参数寄存器中取出指定位置的 32 位浮点数，并插入到第一个参数寄存器的指定位置
            vsetq_lane_f32(                                                    \
                vgetq_lane_f32(vreinterpretq_f32_m128(a), ((imm) >> 2) & 0x3), \
                # 从第一个参数寄存器中取出指定位置的 32 位浮点数，并插入到第一个参数寄存器的指定位置
                vmovq_n_f32(                                                   \
                    vgetq_lane_f32(vreinterpretq_f32_m128(a), (imm) & (0x3))), \
                1),                                                            \
            2),                                                                \
        3))
// 使用 imm8 中的控制信息对 a 中的低 64 位的 16 位整数进行混洗，将结果存储在 dst 的低 64 位，高 64 位从 a 复制到 dst
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_shufflelo_epi16
#define _mm_shufflelo_epi16_function(a, imm)                                  \
    _sse2neon_define1(                                                        \
        __m128i, a, int16x8_t ret = vreinterpretq_s16_m128i(_a);              \
        int16x4_t lowBits = vget_low_s16(ret);                                \
        ret = vsetq_lane_s16(vget_lane_s16(lowBits, (imm) & (0x3)), ret, 0);  \
        ret = vsetq_lane_s16(vget_lane_s16(lowBits, ((imm) >> 2) & 0x3), ret, \
                             1);                                              \
        ret = vsetq_lane_s16(vget_lane_s16(lowBits, ((imm) >> 4) & 0x3), ret, \
                             2);                                              \
        ret = vsetq_lane_s16(vget_lane_s16(lowBits, ((imm) >> 6) & 0x3), ret, \
                             3);                                              \
        _sse2neon_return(vreinterpretq_m128i_s16(ret));)

// 使用 imm8 中的控制信息对 a 中的高 64 位的 16 位整数进行混洗，将结果存储在 dst 的高 64 位，低 64 位从 a 复制到 dst
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_shufflehi_epi16
#define _mm_shufflehi_epi16_function(a, imm)                                   \
    # 定义一个宏函数，将__m128i类型的a转换为int16x8_t类型的ret
    _sse2neon_define1(                                                         \
        __m128i, a, int16x8_t ret = vreinterpretq_s16_m128i(_a);               \
        # 获取ret中的高16位数据
        int16x4_t highBits = vget_high_s16(ret);                               \
        # 将imm中指定位置的16位数据设置到ret的第4个位置
        ret = vsetq_lane_s16(vget_lane_s16(highBits, (imm) & (0x3)), ret, 4);  \
        # 将imm中指定位置的16位数据设置到ret的第5个位置
        ret = vsetq_lane_s16(vget_lane_s16(highBits, ((imm) >> 2) & 0x3), ret, \
                             5);                                               \
        # 将imm中指定位置的16位数据设置到ret的第6个位置
        ret = vsetq_lane_s16(vget_lane_s16(highBits, ((imm) >> 4) & 0x3), ret, \
                             6);                                               \
        # 将imm中指定位置的16位数据设置到ret的第7个位置
        ret = vsetq_lane_s16(vget_lane_s16(highBits, ((imm) >> 6) & 0x3), ret, \
                             7);                                               \
        # 返回转换后的ret
        _sse2neon_return(vreinterpretq_m128i_s16(ret));)
//_mm_empty是arm上的一个空操作
FORCE_INLINE void _mm_empty(void) {}

// 将a和b中的打包的单精度（32位）浮点元素相加，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_add_ps
FORCE_INLINE __m128 _mm_add_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_f32(
        vaddq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
}

// 将a和b中的较低单精度（32位）浮点元素相加，将结果存储在dst的较低元素中，并将a的上3个打包元素复制到dst的上元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_add_ss
FORCE_INLINE __m128 _mm_add_ss(__m128 a, __m128 b)
{
    float32_t b0 = vgetq_lane_f32(vreinterpretq_f32_m128(b), 0);
    float32x4_t value = vsetq_lane_f32(b0, vdupq_n_f32(0), 0);
    // 结果中的上半部分必须是<a>的剩余部分。
    return vreinterpretq_m128_f32(vaddq_f32(a, value));
}

// 计算a和b中的打包的单精度（32位）浮点元素的按位AND，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_and_ps
FORCE_INLINE __m128 _mm_and_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_s32(
        vandq_s32(vreinterpretq_s32_m128(a), vreinterpretq_s32_m128(b)));
}

// 计算a的按位NOT，然后与b进行AND，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_andnot_ps
FORCE_INLINE __m128 _mm_andnot_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_s32(
        vbicq_s32(vreinterpretq_s32_m128(b),
                  vreinterpretq_s32_m128(a)));  // *NOTE* 参数交换
}

// 对a和b中的打包的无符号16位整数求平均，并将结果存储在dst中。
// 定义一个名为 _mm_avg_pu16 的函数，参数为两个 __m64 类型的变量 a 和 b
FORCE_INLINE __m64 _mm_avg_pu16(__m64 a, __m64 b)
{
    // 返回两个 __m64 类型变量 a 和 b 中每个对应位置元素的平均值
    return vreinterpret_m64_u16(
        vrhadd_u16(vreinterpret_u16_m64(a), vreinterpret_u16_m64(b)));
}

// 定义一个名为 _mm_avg_pu8 的函数，参数为两个 __m64 类型的变量 a 和 b
FORCE_INLINE __m64 _mm_avg_pu8(__m64 a, __m64 b)
{
    // 返回两个 __m64 类型变量 a 和 b 中每个对应位置元素的平均值
    return vreinterpret_m64_u8(
        vrhadd_u8(vreinterpret_u8_m64(a), vreinterpret_u8_m64(b)));
}

// 定义一个名为 _mm_cmpeq_ps 的函数，参数为两个 __m128 类型的变量 a 和 b
FORCE_INLINE __m128 _mm_cmpeq_ps(__m128 a, __m128 b)
{
    // 返回两个 __m128 类型变量 a 和 b 中每个对应位置元素的相等性比较结果
    return vreinterpretq_m128_u32(
        vceqq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
}

// 定义一个名为 _mm_cmpeq_ss 的函数，参数为两个 __m128 类型的变量 a 和 b
FORCE_INLINE __m128 _mm_cmpeq_ss(__m128 a, __m128 b)
{
    // 返回两个 __m128 类型变量 a 和 b 中每个对应位置元素的相等性比较结果
    // 将结果存储在 dst 的低位元素中，并将 a 的其他元素复制到 dst 的高位元素中
    return _mm_move_ss(a, _mm_cmpeq_ps(a, b));
}

// 定义一个名为 _mm_cmpge_ps 的函数，参数为两个 __m128 类型的变量 a 和 b
FORCE_INLINE __m128 _mm_cmpge_ps(__m128 a, __m128 b)
{
    // 返回两个 __m128 类型变量 a 和 b 中每个对应位置元素的大于或等于比较结果
    return vreinterpretq_m128_u32(
        vcgeq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
}
// 比较两个 __m128 类型的参数 a 和 b 中的单精度浮点数元素，如果 a 中的元素大于等于 b 中的元素，则返回结果，否则返回 0
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpge_ss
FORCE_INLINE __m128 _mm_cmpge_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmpge_ps(a, b));
}

// 比较两个 __m128 类型的参数 a 和 b 中的单精度浮点数元素，如果 a 中的元素大于 b 中的元素，则返回结果，否则返回 0
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpgt_ps
FORCE_INLINE __m128 _mm_cmpgt_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_u32(
        vcgtq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
}

// 比较两个 __m128 类型的参数 a 和 b 中的单精度浮点数元素，如果 a 中的元素大于 b 中的元素，则返回结果，否则返回 0
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpgt_ss
FORCE_INLINE __m128 _mm_cmpgt_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmpgt_ps(a, b));
}

// 比较两个 __m128 类型的参数 a 和 b 中的单精度浮点数元素，如果 a 中的元素小于等于 b 中的元素，则返回结果，否则返回 0
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmple_ps
FORCE_INLINE __m128 _mm_cmple_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_u32(
        vcleq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
}

// 比较两个 __m128 类型的参数 a 和 b 中的单精度浮点数元素，如果 a 中的元素小于等于 b 中的元素，则返回结果，否则返回 0
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmple_ss
FORCE_INLINE __m128 _mm_cmple_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmple_ps(a, b));
}

// 比较两个 __m128 类型的参数 a 和 b 中的单精度浮点数元素
// 使用 SIMD 指令集中的比较指令，比较两个 __m128 类型的参数 a 和 b 中的每个元素，将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmplt_ps
FORCE_INLINE __m128 _mm_cmplt_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_u32(
        vcltq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
}

// 比较两个 __m128 类型参数 a 和 b 中的低位单精度（32位）浮点数元素，将结果存储在 dst 的低位元素中，并将 a 的高3个元素复制到 dst 的高位元素中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmplt_ss
FORCE_INLINE __m128 _mm_cmplt_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmplt_ps(a, b));
}

// 比较两个参数 a 和 b 中的每个单精度（32位）浮点数元素是否不相等，并将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpneq_ps
FORCE_INLINE __m128 _mm_cmpneq_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_u32(vmvnq_u32(
        vceqq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b))));
}

// 比较两个参数 a 和 b 中的低位单精度（32位）浮点数元素是否不相等，将结果存储在 dst 的低位元素中，并将 a 的高3个元素复制到 dst 的高位元素中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpneq_ss
FORCE_INLINE __m128 _mm_cmpneq_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmpneq_ps(a, b));
}

// 比较两个参数 a 和 b 中的每个单精度（32位）浮点数元素是否不大于或等于，并将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnge_ps
FORCE_INLINE __m128 _mm_cmpnge_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_u32(vmvnq_u32(
        vcgeq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b))));
}
// 比较两个单精度浮点数的低位元素，如果不大于或等于，则将结果存储在目标的低位元素中，并将a的上3个打包元素复制到目标的上位元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnge_ss
FORCE_INLINE __m128 _mm_cmpnge_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmpnge_ps(a, b));
}

// 比较a和b中打包的单精度浮点数元素，如果不大于，则将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpngt_ps
FORCE_INLINE __m128 _mm_cmpngt_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_u32(vmvnq_u32(
        vcgtq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b))));
}

// 比较a和b中的单精度浮点数的低位元素，如果不大于，则将结果存储在目标的低位元素中，并将a的上3个打包元素复制到目标的上位元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpngt_ss
FORCE_INLINE __m128 _mm_cmpngt_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmpngt_ps(a, b));
}

// 比较a和b中打包的单精度浮点数元素，如果不小于或等于，则将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnle_ps
FORCE_INLINE __m128 _mm_cmpnle_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_u32(vmvnq_u32(
        vcleq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b))));
}

// 比较a和b中的单精度浮点数的低位元素，如果不小于或等于，则将结果存储在目标的低位元素中，并将a的上3个打包元素复制到目标的上位元素中
// 定义一个名为 _mm_cmpnle_ss 的函数，参数为两个 __m128 类型的变量 a 和 b
FORCE_INLINE __m128 _mm_cmpnle_ss(__m128 a, __m128 b)
{
    // 返回一个 __m128 类型的值，调用 _mm_cmpnle_ps 函数对 a 和 b 进行比较
    return _mm_move_ss(a, _mm_cmpnle_ps(a, b));
}

// 对比两个 __m128 类型的变量 a 和 b 中的单精度浮点数元素，判断是否不小于，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnlt_ps
FORCE_INLINE __m128 _mm_cmpnlt_ps(__m128 a, __m128 b)
{
    // 调用 NEON 指令，对 a 和 b 进行比较，返回结果
    return vreinterpretq_m128_u32(vmvnq_u32(
        vcltq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b))));
}

// 对比两个 __m128 类型的变量 a 和 b 中的单精度浮点数元素，判断是否不小于，并将结果存储在 dst 的低位元素中，同时将 a 的上三个元素复制到 dst 的上三个位置
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnlt_ss
FORCE_INLINE __m128 _mm_cmpnlt_ss(__m128 a, __m128 b)
{
    // 返回一个 __m128 类型的值，调用 _mm_cmpnlt_ps 函数对 a 和 b 进行比较
    return _mm_move_ss(a, _mm_cmpnlt_ps(a, b));
}

// 对比两个 __m128 类型的变量 a 和 b 中的单精度浮点数元素，判断是否都不是 NaN，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpord_ps
//
// 参考：
// http://stackoverflow.com/questions/8627331/what-does-ordered-unordered-comparison-mean
// http://stackoverflow.com/questions/29349621/neon-isnanval-intrinsics
FORCE_INLINE __m128 _mm_cmpord_ps(__m128 a, __m128 b)
{
    // 注意：NEON 没有内置的有序比较
    // 需要比较 a eq a 和 b eq b 来检查 NaN
    // 对结果进行 AND 运算得到最终结果
    uint32x4_t ceqaa =
        vceqq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(a));
    uint32x4_t ceqbb =
        vceqq_f32(vreinterpretq_f32_m128(b), vreinterpretq_f32_m128(b));
    return vreinterpretq_m128_u32(vandq_u32(ceqaa, ceqbb));
}

// 对比两个 __m128 类型的变量 a 和 b 中的单精度浮点数元素的低位元素，判断是否不是 NaN，并将结果存储在 dst 的低位元素中
// 使用内联汇编指令实现单精度浮点数比较，判断是否有 NaN，将结果存储在目标寄存器的低位，并将 a 的上3个打包元素复制到目标寄存器的上位
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpord_ss
FORCE_INLINE __m128 _mm_cmpord_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmpord_ps(a, b));
}

// 比较打包的单精度（32位）浮点元素 a 和 b，判断是否有 NaN，并将结果存储在目标寄存器
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpunord_ps
FORCE_INLINE __m128 _mm_cmpunord_ps(__m128 a, __m128 b)
{
    uint32x4_t f32a =
        vceqq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(a));
    uint32x4_t f32b =
        vceqq_f32(vreinterpretq_f32_m128(b), vreinterpretq_f32_m128(b));
    return vreinterpretq_m128_u32(vmvnq_u32(vandq_u32(f32a, f32b)));
}

// 比较 a 和 b 的低位单精度（32位）浮点元素，判断是否有 NaN，将结果存储在目标寄存器的低位，并将 a 的上3个打包元素复制到目标寄存器的上位
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpunord_ss
FORCE_INLINE __m128 _mm_cmpunord_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_cmpunord_ps(a, b));
}

// 比较 a 和 b 的低位单精度（32位）浮点元素是否相等，并返回布尔结果（0或1）
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_comieq_ss
FORCE_INLINE int _mm_comieq_ss(__m128 a, __m128 b)
{
    uint32x4_t a_eq_b =
        vceqq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b));
    return vgetq_lane_u32(a_eq_b, 0) & 0x1;
}

// 比较 a 和 b 的低位单精度（32位）浮点元素是否大于或等于，并返回布尔结果（0或1）。
// 使用 SIMD 指令集实现单精度浮点数比较操作，返回结果为大于等于
FORCE_INLINE int _mm_comige_ss(__m128 a, __m128 b)
{
    // 将参数 a 和 b 转换为 4 个单精度浮点数向量，然后比较大小
    uint32x4_t a_ge_b =
        vcgeq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b));
    // 返回比较结果向量的第一个元素，并进行位与运算
    return vgetq_lane_u32(a_ge_b, 0) & 0x1;
}

// 使用 SIMD 指令集实现单精度浮点数比较操作，返回结果为大于
FORCE_INLINE int _mm_comigt_ss(__m128 a, __m128 b)
{
    // 将参数 a 和 b 转换为 4 个单精度浮点数向量，然后比较大小
    uint32x4_t a_gt_b =
        vcgtq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b));
    // 返回比较结果向量的第一个元素，并进行位与运算
    return vgetq_lane_u32(a_gt_b, 0) & 0x1;
}

// 使用 SIMD 指令集实现单精度浮点数比较操作，返回结果为小于等于
FORCE_INLINE int _mm_comile_ss(__m128 a, __m128 b)
{
    // 将参数 a 和 b 转换为 4 个单精度浮点数向量，然后比较大小
    uint32x4_t a_le_b =
        vcleq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b));
    // 返回比较结果向量的第一个元素，并进行位与运算
    return vgetq_lane_u32(a_le_b, 0) & 0x1;
}

// 使用 SIMD 指令集实现单精度浮点数比较操作，返回结果为小于
FORCE_INLINE int _mm_comilt_ss(__m128 a, __m128 b)
{
    // 将参数 a 和 b 转换为 4 个单精度浮点数向量，然后比较大小
    uint32x4_t a_lt_b =
        vcltq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b));
    // 返回比较结果向量的第一个元素，并进行位与运算
    return vgetq_lane_u32(a_lt_b, 0) & 0x1;
}

// 使用 SIMD 指令集实现单精度浮点数比较操作，返回结果为不等于
FORCE_INLINE int _mm_comineq_ss(__m128 a, __m128 b)
{
    // 调用 _mm_comieq_ss 函数取反
    return !_mm_comieq_ss(a, b);
}

// 将参数 b 中的 4 个有符号 32 位整数转换为单精度浮点数向量
// 将 32 位浮点数元素转换为整数，并将结果存储在 dst 的低 2 个元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvt_pi2ps
FORCE_INLINE __m128 _mm_cvt_pi2ps(__m128 a, __m64 b)
{
    return vreinterpretq_m128_f32(
        vcombine_f32(vcvt_f32_s32(vreinterpret_s32_m64(b)),
                     vget_high_f32(vreinterpretq_f32_m128(a))));
}

// 将打包的单精度（32 位）浮点元素转换为打包的 32 位整数，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvt_ps2pi
FORCE_INLINE __m64 _mm_cvt_ps2pi(__m128 a)
{
#if (defined(__aarch64__) || defined(_M_ARM64)) || \
    defined(__ARM_FEATURE_DIRECTED_ROUNDING)
    return vreinterpret_m64_s32(
        vget_low_s32(vcvtnq_s32_f32(vrndiq_f32(vreinterpretq_f32_m128(a)))));
#else
    return vreinterpret_m64_s32(vcvt_s32_f32(vget_low_f32(
        vreinterpretq_f32_m128(_mm_round_ps(a, _MM_FROUND_CUR_DIRECTION)))));
#endif
}

// 将有符号 32 位整数 b 转换为单精度（32 位）浮点元素，将结果存储在 dst 的低元素中，并将 a 中的上 3 个打包元素复制到 dst 的上元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvt_si2ss
FORCE_INLINE __m128 _mm_cvt_si2ss(__m128 a, int b)
{
    return vreinterpretq_m128_f32(
        vsetq_lane_f32((float) b, vreinterpretq_f32_m128(a), 0));
}

// 将 a 中的较低单精度（32 位）浮点元素转换为 32 位整数，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvt_ss2si
FORCE_INLINE int _mm_cvt_ss2si(__m128 a)
{
#if (defined(__aarch64__) || defined(_M_ARM64)) || \
    defined(__ARM_FEATURE_DIRECTED_ROUNDING)
    # 将输入的浮点数向下取整后转换为整型，再取出第0个元素
    return vgetq_lane_s32(vcvtnq_s32_f32(vrndiq_f32(vreinterpretq_f32_m128(a))),
                          0);
// 如果不满足条件，则执行以下代码
#else
    // 从参数 a 中提取第一个 32 位浮点数，并返回
    float32_t data = vgetq_lane_f32(
        vreinterpretq_f32_m128(_mm_round_ps(a, _MM_FROUND_CUR_DIRECTION)), 0);
    // 将提取的浮点数转换为整型并返回
    return (int32_t) data;
#endif
}

// 将参数 a 中的 16 位整数转换为单精度浮点数，并存储结果在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpi16_ps
FORCE_INLINE __m128 _mm_cvtpi16_ps(__m64 a)
{
    return vreinterpretq_m128_f32(
        vcvtq_f32_s32(vmovl_s16(vreinterpret_s16_m64(a))));
}

// 将参数 b 中的 32 位整数转换为单精度浮点数，并存储结果在 dst 的低 2 个元素中，然后将参数 a 中的上 2 个元素复制到 dst 的上 2 个元素中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpi32_ps
FORCE_INLINE __m128 _mm_cvtpi32_ps(__m128 a, __m64 b)
{
    return vreinterpretq_m128_f32(
        vcombine_f32(vcvt_f32_s32(vreinterpret_s32_m64(b)),
                     vget_high_f32(vreinterpretq_f32_m128(a))));
}

// 将参数 a 中的有符号 32 位整数转换为单精度浮点数，并存储结果在 dst 的低 2 个元素中，然后将参数 b 中的有符号 32 位整数转换为单精度浮点数，并存储结果在 dst 的上 2 个元素中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpi32x2_ps
FORCE_INLINE __m128 _mm_cvtpi32x2_ps(__m64 a, __m64 b)
{
    return vreinterpretq_m128_f32(vcvtq_f32_s32(
        vcombine_s32(vreinterpret_s32_m64(a), vreinterpret_s32_m64(b))));
}

// 将参数 a 中的低 8 位整数转换为单精度浮点数，并存储结果在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpi8_ps
FORCE_INLINE __m128 _mm_cvtpi8_ps(__m64 a)
{
    # 将输入的 64 位整数向量重新解释为 8 位有符号整数向量
    vreinterpret_s8_m64(a)
    # 将 8 位有符号整数向量扩展为 16 位有符号整数向量
    vmovl_s8(vreinterpret_s8_m64(a))
    # 取出 16 位有符号整数向量的低 8 位，扩展为 32 位有符号整数向量
    vget_low_s16(vmovl_s8(vreinterpret_s8_m64(a)))
    # 将 32 位有符号整数向量扩展为 64 位浮点数向量
    vcvtq_f32_s32(vmovl_s16(vget_low_s16(vmovl_s8(vreinterpret_s8_m64(a)))))
    # 将 64 位浮点数向量重新解释为 128 位浮点数向量
    vreinterpretq_m128_f32(vcvtq_f32_s32(
        vmovl_s16(vget_low_s16(vmovl_s8(vreinterpret_s8_m64(a))))));
// 将 packed single-precision (32-bit) 浮点元素转换为 packed 16 位整数，并将结果存储在 dst 中
// 注意：对于输入值在 0x7FFF 和 0x7FFFFFFF 之间的情况，此内联函数将生成 0x7FFF，而不是 0x8000
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtps_pi16
FORCE_INLINE __m64 _mm_cvtps_pi16(__m128 a)
{
    return vreinterpret_m64_s16(
        vqmovn_s32(vreinterpretq_s32_m128i(_mm_cvtps_epi32(a))));
}

// 将 packed single-precision (32-bit) 浮点元素转换为 packed 32 位整数，并将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtps_pi32
#define _mm_cvtps_pi32(a) _mm_cvt_ps2pi(a)

// 将 packed single-precision (32-bit) 浮点元素转换为 packed 8 位整数，并将结果存储在 dst 的低 4 个元素中
// 注意：对于输入值在 0x7F 和 0x7FFFFFFF 之间的情况，此内联函数将生成 0x7F，而不是 0x80
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtps_pi8
FORCE_INLINE __m64 _mm_cvtps_pi8(__m128 a)
{
    return vreinterpret_m64_s8(vqmovn_s16(
        vcombine_s16(vreinterpret_s16_m64(_mm_cvtps_pi16(a)), vdup_n_s16(0))));
}

// 将 packed 无符号 16 位整数转换为 packed single-precision (32-bit) 浮点元素，并将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpu16_ps
FORCE_INLINE __m128 _mm_cvtpu16_ps(__m64 a)
{
    return vreinterpretq_m128_f32(
        vcvtq_f32_u32(vmovl_u16(vreinterpret_u16_m64(a))));
}

// 将 packed 无符号 8 位整数的低位转换为 packed single-precision (32-bit) 浮点元素，并将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpu8_ps
FORCE_INLINE __m128 _mm_cvtpu8_ps(__m64 a)
{
    # 将输入的 8 位无符号整数向量转换为 16 位无符号整数向量，并将结果存储在低位
    vreinterpret_u8_m64(a)
    # 将上一步得到的 16 位无符号整数向量扩展为 32 位无符号整数向量，并将结果存储在低位
    vmovl_u8(vreinterpret_u8_m64(a))
    # 将上一步得到的 32 位无符号整数向量的低位转换为 16 位无符号整数向量
    vget_low_u16(vmovl_u8(vreinterpret_u8_m64(a)))
    # 将上一步得到的 16 位无符号整数向量扩展为 32 位无符号整数向量
    vmovl_u16(vget_low_u16(vmovl_u8(vreinterpret_u8_m64(a))))
    # 将上一步得到的 32 位无符号整数向量转换为 32 位浮点数向量
    vcvtq_f32_u32(vmovl_u16(vget_low_u16(vmovl_u8(vreinterpret_u8_m64(a)))))
    # 将上一步得到的 32 位浮点数向量转换为 128 位浮点数向量
    vreinterpretq_m128_f32(vcvtq_f32_u32(vmovl_u16(vget_low_u16(vmovl_u8(vreinterpret_u8_m64(a))))))
    # 返回最终结果
    return vreinterpretq_m128_f32(vcvtq_f32_u32(vmovl_u16(vget_low_u16(vmovl_u8(vreinterpret_u8_m64(a))))));
// 将有符号的32位整数b转换为单精度（32位）浮点数元素，并将结果存储在dst的低位元素中，然后将a的上3个打包元素复制到dst的上位元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi32_ss
#define _mm_cvtsi32_ss(a, b) _mm_cvt_si2ss(a, b)

// 将有符号的64位整数b转换为单精度（32位）浮点数元素，并将结果存储在dst的低位元素中，然后将a的上3个打包元素复制到dst的上位元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi64_ss
FORCE_INLINE __m128 _mm_cvtsi64_ss(__m128 a, int64_t b)
{
    return vreinterpretq_m128_f32(
        vsetq_lane_f32((float) b, vreinterpretq_f32_m128(a), 0));
}

// 将a的低位单精度（32位）浮点数元素复制到dst。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtss_f32
FORCE_INLINE float _mm_cvtss_f32(__m128 a)
{
    return vgetq_lane_f32(vreinterpretq_f32_m128(a), 0);
}

// 将a中的低位单精度（32位）浮点数元素转换为32位整数，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtss_si32
#define _mm_cvtss_si32(a) _mm_cvt_ss2si(a)

// 将a中的低位单精度（32位）浮点数元素转换为64位整数，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtss_si64
FORCE_INLINE int64_t _mm_cvtss_si64(__m128 a)
{
#if (defined(__aarch64__) || defined(_M_ARM64)) || \
    defined(__ARM_FEATURE_DIRECTED_ROUNDING)
    return (int64_t) vgetq_lane_f32(vrndiq_f32(vreinterpretq_f32_m128(a)), 0);
#else
    float32_t data = vgetq_lane_f32(
        vreinterpretq_f32_m128(_mm_round_ps(a, _MM_FROUND_CUR_DIRECTION)), 0);
    return (int64_t) data;
#endif
// 将向量 a 中的打包的单精度（32位）浮点元素转换为截断的打包32位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtt_ps2pi
FORCE_INLINE __m64 _mm_cvtt_ps2pi(__m128 a)
{
    return vreinterpret_m64_s32(
        vget_low_s32(vcvtq_s32_f32(vreinterpretq_f32_m128(a))));
}

// 将向量 a 中的较低单精度（32位）浮点元素转换为截断的32位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtt_ss2si
FORCE_INLINE int _mm_cvtt_ss2si(__m128 a)
{
    return vgetq_lane_s32(vcvtq_s32_f32(vreinterpretq_f32_m128(a)), 0);
}

// 将向量 a 中的打包的单精度（32位）浮点元素转换为截断的打包32位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvttps_pi32
#define _mm_cvttps_pi32(a) _mm_cvtt_ps2pi(a)

// 将向量 a 中的较低单精度（32位）浮点元素转换为截断的32位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvttss_si32
#define _mm_cvttss_si32(a) _mm_cvtt_ss2si(a)

// 将向量 a 中的较低单精度（32位）浮点元素转换为截断的64位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvttss_si64
FORCE_INLINE int64_t _mm_cvttss_si64(__m128 a)
{
    return (int64_t) vgetq_lane_f32(vreinterpretq_f32_m128(a), 0);
}

// 将向量 a 中的打包的单精度（32位）浮点元素除以向量 b 中的打包元素，并将结果存储在 dst 中。
// 由于 ARMv7-A NEON 缺乏精确的除法内在函数，我们在使用牛顿-拉弗森方法之前通过将 a 乘以 b 的倒数来实现除法
// 用于近似计算结果的方法
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_div_ps
FORCE_INLINE __m128 _mm_div_ps(__m128 a, __m128 b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 ARM64 指令进行单精度浮点数除法运算
    return vreinterpretq_m128_f32(
        vdivq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
#else
    // 如果不是 ARM64 架构，则使用 Newton-Raphson 迭代法进行单精度浮点数除法运算
    float32x4_t recip = vrecpeq_f32(vreinterpretq_f32_m128(b));
    recip = vmulq_f32(recip, vrecpsq_f32(recip, vreinterpretq_f32_m128(b)));
    // 为了提高精度，进行额外的 Newton-Raphson 迭代
    recip = vmulq_f32(recip, vrecpsq_f32(recip, vreinterpretq_f32_m128(b)));
    return vreinterpretq_m128_f32(vmulq_f32(vreinterpretq_f32_m128(a), recip));
#endif
}

// 将 a 中的低位单精度浮点数（32位）除以 b 中的低位单精度浮点数（32位），将结果存储在 dst 的低位，并将 a 中的上3个元素复制到 dst 的上位
// 警告：ARMv7-A 与 Intel 不产生相同的结果，不符合 IEEE 标准
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_div_ss
FORCE_INLINE __m128 _mm_div_ss(__m128 a, __m128 b)
{
    // 从 _mm_div_ps 返回的结果中提取第一个单精度浮点数，并存储在 dst 的低位
    float32_t value =
        vgetq_lane_f32(vreinterpretq_f32_m128(_mm_div_ps(a, b)), 0);
    return vreinterpretq_m128_f32(
        vsetq_lane_f32(value, vreinterpretq_f32_m128(a), 0));
}

// 从 a 中提取一个16位整数，使用 imm8 选择，将结果存储在 dst 的低位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_extract_pi16
#define _mm_extract_pi16(a, imm) \
    (int32_t) vget_lane_u16(vreinterpret_u16_m64(a), (imm))

// 释放使用 _mm_malloc 分配的对齐内存
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_free
#if !defined(SSE2NEON_ALLOC_DEFINED)
FORCE_INLINE void _mm_free(void *addr)
{
    free(addr);
}
#endif
// 强制内联函数，获取当前的浮点控制寄存器的值
FORCE_INLINE uint64_t _sse2neon_get_fpcr(void)
{
    uint64_t value;
#if defined(_MSC_VER)
    // 使用 ARM64_FPCR 寄存器读取状态寄存器的值
    value = _ReadStatusReg(ARM64_FPCR);
#else
    // 使用汇编语句读取 FPCR 寄存器的值
    __asm__ __volatile__("mrs %0, FPCR" : "=r"(value)); /* read */
#endif
    return value;
}

// 强制内联函数，设置浮点控制寄存器的值
FORCE_INLINE void _sse2neon_set_fpcr(uint64_t value)
{
#if defined(_MSC_VER)
    // 使用 ARM64_FPCR 寄存器写入状态寄存器的值
    _WriteStatusReg(ARM64_FPCR, value);
#else
    // 使用汇编语句写入 FPCR 寄存器的值
    __asm__ __volatile__("msr FPCR, %0" ::"r"(value));  /* write */
#endif
}

// 宏：获取 MXCSR 控制和状态寄存器中的 flush zero 位
// flush zero 可能包含以下标志之一：_MM_FLUSH_ZERO_ON 或 _MM_FLUSH_ZERO_OFF
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_MM_GET_FLUSH_ZERO_MODE
FORCE_INLINE unsigned int _sse2neon_mm_get_flush_zero_mode(void)
{
    union {
        fpcr_bitfield field;
#if defined(__aarch64__) || defined(_M_ARM64)
        uint64_t value;
#else
        uint32_t value;
#endif
    } r;

#if defined(__aarch64__) || defined(_M_ARM64)
    // 获取当前浮点控制寄存器的值
    r.value = _sse2neon_get_fpcr();
#else
    // 使用汇编语句读取 FPSCR 寄存器的值
    __asm__ __volatile__("vmrs %0, FPSCR" : "=r"(r.value)); /* read */
#endif

    // 根据浮点控制寄存器的值判断 flush zero 是否开启
    return r.field.bit24 ? _MM_FLUSH_ZERO_ON : _MM_FLUSH_ZERO_OFF;
}

// 宏：获取 MXCSR 控制和状态寄存器中的舍入模式位
// 舍入模式可能包含以下标志之一：_MM_ROUND_NEAREST、_MM_ROUND_DOWN、_MM_ROUND_UP、_MM_ROUND_TOWARD_ZERO
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_MM_GET_ROUNDING_MODE
FORCE_INLINE unsigned int _MM_GET_ROUNDING_MODE(void)
{
    union {
        fpcr_bitfield field;
#if defined(__aarch64__) || defined(_M_ARM64)
        uint64_t value;
#else
        uint32_t value;
#endif
    } r;

#if defined(__aarch64__) || defined(_M_ARM64)
    // 获取当前浮点控制寄存器的值
    r.value = _sse2neon_get_fpcr();
#else
    // 使用汇编语句读取 FPSCR 寄存器的值
    __asm__ __volatile__("vmrs %0, FPSCR" : "=r"(r.value)); /* read */
#endif

    // 根据浮点控制寄存器的值判断舍入模式
    if (r.field.bit22) {
        return r.field.bit23 ? _MM_ROUND_TOWARD_ZERO : _MM_ROUND_UP;
    } else {
        # 如果条件不满足，则返回下面的值
        return r.field.bit23 ? _MM_ROUND_DOWN : _MM_ROUND_NEAREST;
    }
// 定义宏，将16位整数i插入到dst中，位置由imm8指定
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_insert_pi16
#define _mm_insert_pi16(a, b, imm) \
    vreinterpret_m64_s16(vset_lane_s16((b), vreinterpret_s16_m64(a), (imm)))

// 从内存加载128位数据（由4个打包的单精度（32位）浮点元素组成）到dst中。mem_addr必须对齐到16字节边界，否则可能会生成通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_load_ps
FORCE_INLINE __m128 _mm_load_ps(const float *p)
{
    return vreinterpretq_m128_f32(vld1q_f32(p));
}

// 从内存加载单精度（32位）浮点元素到dst的所有元素中。
//
//   dst[31:0] := MEM[mem_addr+31:mem_addr]
//   dst[63:32] := MEM[mem_addr+31:mem_addr]
//   dst[95:64] := MEM[mem_addr+31:mem_addr]
//   dst[127:96] := MEM[mem_addr+31:mem_addr]
//
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_load_ps1
#define _mm_load_ps1 _mm_load1_ps

// 从内存加载单精度（32位）浮点元素到dst的低位，并将上面3个元素置零。mem_addr不需要对齐到特定边界。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_load_ss
FORCE_INLINE __m128 _mm_load_ss(const float *p)
{
    return vreinterpretq_m128_f32(vsetq_lane_f32(*p, vdupq_n_f32(0), 0));
}

// 从内存加载单精度（32位）浮点元素到dst的所有元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_load1_ps
FORCE_INLINE __m128 _mm_load1_ps(const float *p)
{
    return vreinterpretq_m128_f32(vld1q_dup_f32(p));
}

// 从内存加载2个单精度（32位）浮点元素到
// 从内存中加载 16 位整数到 dst 的第一个元素，内存地址不需要对齐
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_loadl_epi16
FORCE_INLINE __m128i _mm_loadl_epi16(__m128i const *mem_addr)
{
    return vreinterpretq_m128i_s16(vcombine_s16(vld1_s16((const int16_t *) mem_addr), vdup_n_s16(0)));
}
// 从内存中加载一个未对齐的 16 位整数到 dst 的第一个元素
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_loadu_si16
FORCE_INLINE __m128i _mm_loadu_si16(const void *p)
{
    return vreinterpretq_m128i_s16(
        vsetq_lane_s16(*(const int16_t *) p, vdupq_n_s16(0), 0));
}

// 从内存中加载一个未对齐的 64 位整数到 dst 的第一个元素
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_loadu_si64
FORCE_INLINE __m128i _mm_loadu_si64(const void *p)
{
    return vreinterpretq_m128i_s64(
        vcombine_s64(vld1_s64((const int64_t *) p), vdup_n_s64(0)));
}

// 分配 size 字节的内存，按照 align 中指定的对齐方式对齐，并返回指向分配内存的指针。应使用 _mm_malloc 释放使用 _mm_malloc 分配的内存。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_malloc
#if !defined(SSE2NEON_ALLOC_DEFINED)
FORCE_INLINE void *_mm_malloc(size_t size, size_t align)
{
    void *ptr;
    if (align == 1)
        return malloc(size);
    if (align == 2 || (sizeof(void *) == 8 && align == 4))
        align = sizeof(void *);
    if (!posix_memalign(&ptr, align, size))
        return ptr;
    return NULL;
}
#endif

// 根据掩码条件将 8 位整数元素从 a 存储到内存中（当对应元素中最高位未设置时，不存储元素），并使用非临时内存提示。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_maskmove_si64
FORCE_INLINE void _mm_maskmove_si64(__m64 a, __m64 mask, char *mem_addr)
{
    int8x8_t shr_mask = vshr_n_s8(vreinterpret_s8_m64(mask), 7);
    __m128 b = _mm_load_ps((const float *) mem_addr);
    int8x8_t masked =
        vbsl_s8(vreinterpret_u8_s8(shr_mask), vreinterpret_s8_m64(a),
                vreinterpret_s8_u64(vget_low_u64(vreinterpretq_u64_m128(b))));
    vst1_s8((int8_t *) mem_addr, masked);
}
// 使用掩码条件性地将 a 中的 8 位整数元素存储到内存中（当对应元素中最高位未设置时，不存储元素），并使用非暂时性内存提示。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_m_maskmovq
#define _m_maskmovq(a, mask, mem_addr) _mm_maskmove_si64(a, mask, mem_addr)

// 比较 a 和 b 中打包的有符号 16 位整数，并将最大值存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_pi16
FORCE_INLINE __m64 _mm_max_pi16(__m64 a, __m64 b)
{
    return vreinterpret_m64_s16(
        vmax_s16(vreinterpret_s16_m64(a), vreinterpret_s16_m64(b)));
}

// 比较 a 和 b 中打包的单精度（32 位）浮点数元素，并将最大值存储在 dst 中。当输入为 NaN 或有符号零值时，dst 不遵循 IEEE 浮点算术标准（IEEE 754）的最大值。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_ps
FORCE_INLINE __m128 _mm_max_ps(__m128 a, __m128 b)
{
#if SSE2NEON_PRECISE_MINMAX
    float32x4_t _a = vreinterpretq_f32_m128(a);
    float32x4_t _b = vreinterpretq_f32_m128(b);
    return vreinterpretq_m128_f32(vbslq_f32(vcgtq_f32(_a, _b), _a, _b));
#else
    return vreinterpretq_m128_f32(
        vmaxq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
#endif
}

// 比较 a 和 b 中打包的无符号 8 位整数，并将最大值存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_pu8
FORCE_INLINE __m64 _mm_max_pu8(__m64 a, __m64 b)
{
    return vreinterpret_m64_u8(
        vmax_u8(vreinterpret_u8_m64(a), vreinterpret_u8_m64(b)));
}

// 比较 a 和 b 中的较低单精度（32 位）浮点数元素，将最大值存储在 dst 的较低元素中，并复制上面的 3 个
// 使用_mm_max_ps函数比较a和b中的单精度浮点数，将结果中的最大值提取出来
FORCE_INLINE __m128 _mm_max_ss(__m128 a, __m128 b)
{
    float32_t value = vgetq_lane_f32(_mm_max_ps(a, b), 0);
    return vreinterpretq_m128_f32(
        vsetq_lane_f32(value, vreinterpretq_f32_m128(a), 0));
}

// 比较a和b中的有符号16位整数，将最小值存储在dst中
FORCE_INLINE __m64 _mm_min_pi16(__m64 a, __m64 b)
{
    return vreinterpret_m64_s16(
        vmin_s16(vreinterpret_s16_m64(a), vreinterpret_s16_m64(b)));
}

// 比较a和b中的单精度浮点数，将结果中的最小值提取出来
FORCE_INLINE __m128 _mm_min_ps(__m128 a, __m128 b)
{
#if SSE2NEON_PRECISE_MINMAX
    float32x4_t _a = vreinterpretq_f32_m128(a);
    float32x4_t _b = vreinterpretq_f32_m128(b);
    return vreinterpretq_m128_f32(vbslq_f32(vcltq_f32(_a, _b), _a, _b));
#else
    return vreinterpretq_m128_f32(
        vminq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
#endif
}

// 比较a和b中的无符号8位整数，将最小值存储在dst中
FORCE_INLINE __m64 _mm_min_pu8(__m64 a, __m64 b)
{
    return vreinterpret_m64_u8(
        vmin_u8(vreinterpret_u8_m64(a), vreinterpret_u8_m64(b)));
}
// 比较两个__m128类型的参数a和b中的低位单精度（32位）浮点数元素，将最小值存储在dst的低位元素中，并将a的上3个打包元素复制到dst的上位元素中。当输入为NaN或有符号零值时，dst不遵循IEEE浮点运算标准（IEEE 754）的最小值。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_min_ss
FORCE_INLINE __m128 _mm_min_ss(__m128 a, __m128 b)
{
    // 从_mm_min_ps(a, b)中获取第0个元素的值
    float32_t value = vgetq_lane_f32(_mm_min_ps(a, b), 0);
    // 将获取的值插入到a的第0个元素位置，然后重新解释为__m128类型
    return vreinterpretq_m128_f32(
        vsetq_lane_f32(value, vreinterpretq_f32_m128(a), 0));
}

// 将b的低位单精度（32位）浮点数元素移动到dst的低位元素，并将a的上3个打包元素复制到dst的上位元素。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_move_ss
FORCE_INLINE __m128 _mm_move_ss(__m128 a, __m128 b)
{
    // 从b中获取第0个元素的值，然后插入到a的第0个元素位置，最后重新解释为__m128类型
    return vreinterpretq_m128_f32(
        vsetq_lane_f32(vgetq_lane_f32(vreinterpretq_f32_m128(b), 0),
                       vreinterpretq_f32_m128(a), 0));
}

// 将b的上2个单精度（32位）浮点数元素移动到dst的低2个元素，并将a的上2个元素复制到dst的上2个元素。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movehl_ps
FORCE_INLINE __m128 _mm_movehl_ps(__m128 a, __m128 b)
{
#if defined(aarch64__)
    // 如果是aarch64架构，使用特定的指令进行操作
    return vreinterpretq_m128_u64(
        vzip2q_u64(vreinterpretq_u64_m128(b), vreinterpretq_u64_m128(a)));
#else
    // 如果不是aarch64架构，分别从a和b中获取高位的两个32位浮点数元素，然后合并成一个__m128类型
    float32x2_t a32 = vget_high_f32(vreinterpretq_f32_m128(a));
    float32x2_t b32 = vget_high_f32(vreinterpretq_f32_m128(b));
    return vreinterpretq_m128_f32(vcombine_f32(b32, a32));
#endif
}

// 将b的低2个单精度（32位）浮点数元素移动到dst的上2个元素，并将a的低2个元素复制到dst的低2个元素
// 将 dst 的低 2 个元素设置为参数 __A 和 __B 的低位部分
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movelh_ps
FORCE_INLINE __m128 _mm_movelh_ps(__m128 __A, __m128 __B)
{
    // 从参数 __A 中获取低 2 个 32 位浮点数，并转换成 2 个 32 位浮点数的数组
    float32x2_t a10 = vget_low_f32(vreinterpretq_f32_m128(__A));
    // 从参数 __B 中获取低 2 个 32 位浮点数，并转换成 2 个 32 位浮点数的数组
    float32x2_t b10 = vget_low_f32(vreinterpretq_f32_m128(__B));
    // 将两个 32 位浮点数的数组合并成一个 128 位浮点数，并返回结果
    return vreinterpretq_m128_f32(vcombine_f32(a10, b10));
}

// 从参数 a 的每个 8 位元素的最高位创建掩码，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movemask_pi8
FORCE_INLINE int _mm_movemask_pi8(__m64 a)
{
    // 将参数 a 转换成 8 位无符号整数的数组
    uint8x8_t input = vreinterpret_u8_m64(a);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，使用 ARM64 特定的实现
    static const int8_t shift[8] = {0, 1, 2, 3, 4, 5, 6, 7};
    // 将 input 中的每个元素右移 7 位，并将结果与 shift 数组中的值左移后相加
    uint8x8_t tmp = vshr_n_u8(input, 7);
    return vaddv_u8(vshl_u8(tmp, vld1_s8(shift)));
#else
    // 如果不是 ARM64 架构，使用通用的实现
    // 参考 `_mm_movemask_epi8` 的实现
    uint16x4_t high_bits = vreinterpret_u16_u8(vshr_n_u8(input, 7));
    uint32x2_t paired16 =
        vreinterpret_u32_u16(vsra_n_u16(high_bits, high_bits, 7));
    uint8x8_t paired32 =
        vreinterpret_u8_u32(vsra_n_u32(paired16, paired16, 14));
    return vget_lane_u8(paired32, 0) | ((int) vget_lane_u8(paired32, 4) << 4);
#endif
}

// 根据参数 a 中每个打包的单精度（32 位）浮点元素的最高位设置掩码 dst 的每个位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movemask_ps
FORCE_INLINE int _mm_movemask_ps(__m128 a)
{
    // 将参数 a 转换成 32 位无符号整数的数组
    uint32x4_t input = vreinterpretq_u32_m128(a);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，使用 ARM64 特定的实现
    static const int32_t shift[4] = {0, 1, 2, 3};
    // 将 input 中的每个元素右移 31 位，并将结果与 shift 数组中的值左移后相加
    uint32x4_t tmp = vshrq_n_u32(input, 31);
    return vaddvq_u32(vshlq_u32(tmp, vld1q_s32(shift)));
#else
    // 如果不是 ARM64 架构，使用通用的实现
    // 使用与 _mm_movemask_epi8 完全相同的方法，请参阅该函数以获取详细信息
    // 通过 32 位无符号右移操作，将除了符号位以外的所有位都移出
    # 使用 vshrq_n_u32 函数将输入向右移动 31 位，然后使用 vreinterpretq_u64_u32 函数将结果转换为 64 位无符号整数
    uint64x2_t high_bits = vreinterpretq_u64_u32(vshrq_n_u32(input, 31));
    # 使用 vsraq_n_u64 函数将两个 64 位整数进行无符号右移和加法操作，然后使用 vreinterpretq_u8_u64 函数将结果转换为 128 位无符号整数
    uint8x16_t paired = vreinterpretq_u8_u64(vsraq_n_u64(high_bits, high_bits, 31));
    # 使用 vgetq_lane_u8 函数提取 paired 中的第 0 个和第 8 个元素，然后进行位运算得到最终结果
    return vgetq_lane_u8(paired, 0) | (vgetq_lane_u8(paired, 8) << 2);
#endif
}

// Multiply packed single-precision (32-bit) floating-point elements in a and b,
// and store the results in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mul_ps
FORCE_INLINE __m128 _mm_mul_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_f32(
        vmulq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
}

// Multiply the lower single-precision (32-bit) floating-point element in a and
// b, store the result in the lower element of dst, and copy the upper 3 packed
// elements from a to the upper elements of dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mul_ss
FORCE_INLINE __m128 _mm_mul_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_mul_ps(a, b));
}

// Multiply the packed unsigned 16-bit integers in a and b, producing
// intermediate 32-bit integers, and store the high 16 bits of the intermediate
// integers in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mulhi_pu16
FORCE_INLINE __m64 _mm_mulhi_pu16(__m64 a, __m64 b)
{
    return vreinterpret_m64_u16(vshrn_n_u32(
        vmull_u16(vreinterpret_u16_m64(a), vreinterpret_u16_m64(b)), 16));
}

// Compute the bitwise OR of packed single-precision (32-bit) floating-point
// elements in a and b, and store the results in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_or_ps
FORCE_INLINE __m128 _mm_or_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_s32(
        vorrq_s32(vreinterpretq_s32_m128(a), vreinterpretq_s32_m128(b)));
}

// Average packed unsigned 8-bit integers in a and b, and store the results in
// dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_m_pavgb
#define _m_pavgb(a, b) _mm_avg_pu8(a, b)

// Average packed unsigned 16-bit integers in a and b, and store the results in
// dst.
// 定义一个宏，用于计算两个 __m64 类型参数的平均值
#define _m_pavgw(a, b) _mm_avg_pu16(a, b)

// 定义一个宏，用于从参数 a 中提取一个 16 位整数，根据 imm8 的值选择，并将结果存储在目标的低位元素中
#define _m_pextrw(a, imm) _mm_extract_pi16(a, imm)

// 定义一个宏，用于将参数 a 复制到目标中，并根据 imm8 的值在目标中插入 16 位整数 i
#define _m_pinsrw(a, i, imm) _mm_insert_pi16(a, i, imm)

// 定义一个宏，用于比较参数 a 和 b 中的打包的有符号 16 位整数，并将最大值存储在目标中
#define _m_pmaxsw(a, b) _mm_max_pi16(a, b)

// 定义一个宏，用于比较参数 a 和 b 中的打包的无符号 8 位整数，并将最大值存储在目标中
#define _m_pmaxub(a, b) _mm_max_pu8(a, b)

// 定义一个宏，用于比较参数 a 和 b 中的打包的有符号 16 位整数，并将最小值存储在目标中
#define _m_pminsw(a, b) _mm_min_pi16(a, b)

// 定义一个宏，用于比较参数 a 和 b 中的打包的无符号 8 位整数，并将最小值存储在目标中
#define _m_pminub(a, b) _mm_min_pu8(a, b)

// 定义一个宏，用于从参数 a 中每个 8 位元素的最高位创建掩码，并将结果存储在目标中
#define _m_pmovmskb(a) _mm_movemask_pi8(a)

// 定义一个宏，用于将参数 a 和 b 中的打包的无符号 16 位整数相乘，产生中间的 32 位整数，并将中间整数的高 16 位存储在目标中
// 定义一个宏，用于执行无符号16位整数的乘法高位运算
#define _m_pmulhuw(a, b) _mm_mulhi_pu16(a, b)

// 预取地址 p 处的数据行到指定缓存层次的位置，使用指定的局部性提示 i
FORCE_INLINE void _mm_prefetch(char const *p, int i)
{
    (void) i; // 忽略参数 i
#if defined(_MSC_VER)
    switch (i) {
    case _MM_HINT_NTA:
        __prefetch2(p, 1); // 非临时访问预取
        break;
    case _MM_HINT_T0:
        __prefetch2(p, 0); // T0级别预取
        break;
    case _MM_HINT_T1:
        __prefetch2(p, 2); // T1级别预取
        break;
    case _MM_HINT_T2:
        __prefetch2(p, 4); // T2级别预取
        break;
    }
#else
    switch (i) {
    case _MM_HINT_NTA:
        __builtin_prefetch(p, 0, 0); // 非临时访问预取
        break;
    case _MM_HINT_T0:
        __builtin_prefetch(p, 0, 3); // T0级别预取
        break;
    case _MM_HINT_T1:
        __builtin_prefetch(p, 0, 2); // T1级别预取
        break;
    case _MM_HINT_T2:
        __builtin_prefetch(p, 0, 1); // T2级别预取
        break;
    }
#endif
}

// 计算两个打包的无符号8位整数的绝对差值，然后水平求和每8个差值，得到四个无符号16位整数，将这些16位整数打包到 dst 的低16位
#define _m_psadbw(a, b) _mm_sad_pu8(a, b)

// 使用 imm8 中的控制信息对 a 中的16位整数进行洗牌，并将结果存储在 dst 中
#define _m_pshufw(a, imm) _mm_shuffle_pi16(a, imm)

// 计算 a 中单精度（32位）浮点数的近似倒数，并将结果存储在 dst 中。此近似的最大相对误差小于1.5*2^-12。
// 计算输入向量中每个元素的近似倒数，并返回结果向量
FORCE_INLINE __m128 _mm_rcp_ps(__m128 in)
{
    // 使用 NEON 指令计算输入向量元素的近似倒数
    float32x4_t recip = vrecpeq_f32(vreinterpretq_f32_m128(in));
    // 通过一系列 NEON 指令进一步优化近似倒数的精度
    recip = vmulq_f32(recip, vrecpsq_f32(recip, vreinterpretq_f32_m128(in)));
    // 将 NEON 向量转换为 SSE 向量并返回结果
    return vreinterpretq_m128_f32(recip);
}

// 计算输入向量中低位单精度浮点数的近似倒数，并将结果存储在目标向量的低位，同时将输入向量的高3个元素复制到目标向量的高位
FORCE_INLINE __m128 _mm_rcp_ss(__m128 a)
{
    // 调用 _mm_rcp_ps 函数计算输入向量的近似倒数，并将结果存储在目标向量的低位
    return _mm_move_ss(a, _mm_rcp_ps(a));
}

// 计算输入向量中每个元素的近似平方根的倒数，并将结果存储在目标向量中
FORCE_INLINE __m128 _mm_rsqrt_ps(__m128 in)
{
    // 使用 NEON 指令计算输入向量元素的近似平方根的倒数
    float32x4_t out = vrsqrteq_f32(vreinterpretq_f32_m128(in));

    // 生成用于检测输入向量中是否存在 0.0f/-0.0f 的掩码
    const uint32x4_t pos_inf = vdupq_n_u32(0x7F800000);
    const uint32x4_t neg_inf = vdupq_n_u32(0xFF800000);
    const uint32x4_t has_pos_zero =
        vceqq_u32(pos_inf, vreinterpretq_u32_f32(out));
    const uint32x4_t has_neg_zero =
        vceqq_u32(neg_inf, vreinterpretq_u32_f32(out));

    // 通过一系列 NEON 指令进一步优化近似平方根的倒数的精度
    out = vmulq_f32(
        out, vrsqrtsq_f32(vmulq_f32(vreinterpretq_f32_m128(in), out), out));

    // 如果输入向量中存在 0.0f/-0.0f，则将输出向量对应元素设置为正无穷/负无穷
    out = vbslq_f32(has_pos_zero, (float32x4_t) pos_inf, out);
    # 使用条件选择指令，如果 has_neg_zero 中有任何元素为真，则将 neg_inf 中对应位置的元素赋值给 out，否则保持 out 不变
    out = vbslq_f32(has_neg_zero, (float32x4_t) neg_inf, out);
    # 将 out 中的数据重新解释为 128 位的浮点数向量，并返回
    return vreinterpretq_m128_f32(out);
// 计算输入参数中低位单精度（32位）浮点数的近似倒数平方根，将结果存储在目标参数的低位，并将输入参数的高3个元素复制到目标参数的高位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_rsqrt_ss
FORCE_INLINE __m128 _mm_rsqrt_ss(__m128 in)
{
    return vsetq_lane_f32(vgetq_lane_f32(_mm_rsqrt_ps(in), 0), in, 0);
}

// 计算参数 a 和 b 中打包的无符号8位整数的绝对差值，然后水平求和每个连续的8个差值，以产生四个无符号16位整数，并将这些无符号16位整数打包在目标参数的低16位中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sad_pu8
FORCE_INLINE __m64 _mm_sad_pu8(__m64 a, __m64 b)
{
    uint64x1_t t = vpaddl_u32(vpaddl_u16(
        vpaddl_u8(vabd_u8(vreinterpret_u8_m64(a), vreinterpret_u8_m64(b)))));
    return vreinterpret_m64_u16(
        vset_lane_u16((int) vget_lane_u64(t, 0), vdup_n_u16(0), 0));
}

// 宏：将 MXCSR 控制和状态寄存器的清零位设置为无符号32位整数 a 中的值。清零位可以包含以下任一标志：_MM_FLUSH_ZERO_ON 或 _MM_FLUSH_ZERO_OFF
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_MM_SET_FLUSH_ZERO_MODE
FORCE_INLINE void _sse2neon_mm_set_flush_zero_mode(unsigned int flag)
{
    // AArch32 高级 SIMD 算术始终使用 Flush-to-zero 设置，不管 FZ 位的值如何。
    union {
        fpcr_bitfield field;
#if defined(__aarch64__) || defined(_M_ARM64)
        uint64_t value;
#else
        uint32_t value;
#endif
    } r;

#if defined(__aarch64__) || defined(_M_ARM64)
    r.value = _sse2neon_get_fpcr();
#else
    __asm__ __volatile__("vmrs %0, FPSCR" : "=r"(r.value)); /* 读取 */
#endif
    # 将 flag 和 _MM_FLUSH_ZERO_MASK 进行按位与操作，判断是否等于 _MM_FLUSH_ZERO_ON
    r.field.bit24 = (flag & _MM_FLUSH_ZERO_MASK) == _MM_FLUSH_ZERO_ON;
#if defined(__aarch64__) || defined(_M_ARM64)
    _sse2neon_set_fpcr(r.value);
#else
    __asm__ __volatile__("vmsr FPSCR, %0" ::"r"(r));        /* write */
#endif
}
// 根据条件编译的平台类型，设置浮点控制寄存器的值
// 如果是 ARM64 平台，则调用 _sse2neon_set_fpcr 函数
// 否则，使用内联汇编语句将值写入 FPSCR 寄存器

// Set packed single-precision (32-bit) floating-point elements in dst with the
// supplied values.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_set_ps
FORCE_INLINE __m128 _mm_set_ps(float w, float z, float y, float x)
{
    float ALIGN_STRUCT(16) data[4] = {x, y, z, w};
    return vreinterpretq_m128_f32(vld1q_f32(data));
}
// 使用给定的值设置目标中的打包单精度（32位）浮点元素
// 返回一个包含设置值的 __m128 类型的变量

// Broadcast single-precision (32-bit) floating-point value a to all elements of
// dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_set_ps1
FORCE_INLINE __m128 _mm_set_ps1(float _w)
{
    return vreinterpretq_m128_f32(vdupq_n_f32(_w));
}
// 将单精度（32位）浮点值 a 广播到目标的所有元素中
// 返回一个包含广播值的 __m128 类型的变量

// Macro: Set the rounding mode bits of the MXCSR control and status register to
// the value in unsigned 32-bit integer a. The rounding mode may contain any of
// the following flags: _MM_ROUND_NEAREST, _MM_ROUND_DOWN, _MM_ROUND_UP,
// _MM_ROUND_TOWARD_ZERO
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_MM_SET_ROUNDING_MODE
FORCE_INLINE void _MM_SET_ROUNDING_MODE(int rounding)
{
    union {
        fpcr_bitfield field;
#if defined(__aarch64__) || defined(_M_ARM64)
        uint64_t value;
#else
        uint32_t value;
#endif
    } r;

#if defined(__aarch64__) || defined(_M_ARM64)
    r.value = _sse2neon_get_fpcr();
#else
    __asm__ __volatile__("vmrs %0, FPSCR" : "=r"(r.value)); /* read */
#endif
    // 根据不同的平台类型，获取当前浮点控制寄存器的值

    switch (rounding) {
    case _MM_ROUND_TOWARD_ZERO:
        r.field.bit22 = 1;
        r.field.bit23 = 1;
        break;
    case _MM_ROUND_DOWN:
        r.field.bit22 = 0;
        r.field.bit23 = 1;
        break;
    case _MM_ROUND_UP:
        r.field.bit22 = 1;
        r.field.bit23 = 0;
        break;
    default:  //_MM_ROUND_NEAREST
        r.field.bit22 = 0;
        r.field.bit23 = 0;
    }
    // 根据指定的舍入模式设置浮点控制寄存器的相应位
#if defined(__aarch64__) || defined(_M_ARM64)
    _sse2neon_set_fpcr(r.value);
#else
    __asm__ __volatile__("vmsr FPSCR, %0" ::"r"(r));        /* write */
#endif
}
// 如果是 ARM64 架构，则调用 _sse2neon_set_fpcr 函数设置浮点控制寄存器的值
// 否则，使用内联汇编语句将浮点控制寄存器的值写入 FPSCR 寄存器

// Copy single-precision (32-bit) floating-point element a to the lower element
// of dst, and zero the upper 3 elements.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_set_ss
FORCE_INLINE __m128 _mm_set_ss(float a)
{
    return vreinterpretq_m128_f32(vsetq_lane_f32(a, vdupq_n_f32(0), 0));
}
// 将单精度（32位）浮点数 a 复制到目标寄存器的低位，并将高位 3 个元素置零

// Broadcast single-precision (32-bit) floating-point value a to all elements of
// dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_set1_ps
FORCE_INLINE __m128 _mm_set1_ps(float _w)
{
    return vreinterpretq_m128_f32(vdupq_n_f32(_w));
}
// 将单精度（32位）浮点数 _w 广播到目标寄存器的所有元素

// Set the MXCSR control and status register with the value in unsigned 32-bit
// integer a.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setcsr
// FIXME: _mm_setcsr() implementation supports changing the rounding mode only.
FORCE_INLINE void _mm_setcsr(unsigned int a)
{
    _MM_SET_ROUNDING_MODE(a);
}
// 使用无符号 32 位整数 a 设置 MXCSR 控制和状态寄存器的值

// Get the unsigned 32-bit value of the MXCSR control and status register.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_getcsr
// FIXME: _mm_getcsr() implementation supports reading the rounding mode only.
FORCE_INLINE unsigned int _mm_getcsr(void)
{
    return _MM_GET_ROUNDING_MODE();
}
// 获取 MXCSR 控制和状态寄存器的无符号 32 位整数值

// Set packed single-precision (32-bit) floating-point elements in dst with the
// supplied values in reverse order.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setr_ps
FORCE_INLINE __m128 _mm_setr_ps(float w, float z, float y, float x)
{
    float ALIGN_STRUCT(16) data[4] = {w, z, y, x};
    return vreinterpretq_m128_f32(vld1q_f32(data));
}
// 使用提供的值以相反顺序设置目标寄存器中的打包单精度（32位）浮点数元素

// Return vector of type __m128 with all elements set to zero.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setzero_ps
# 定义一个函数，返回一个全为0的__m128类型的变量
FORCE_INLINE __m128 _mm_setzero_ps(void)
{
    return vreinterpretq_m128_f32(vdupq_n_f32(0));
}

# 根据 imm8 中的控制参数对a中的16位整数进行重新排列，并将结果存储在dst中
# 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_shuffle_pi16
#ifdef _sse2neon_shuffle
# 定义宏_mm_shuffle_pi16，使用sse2neon_shuffle
#define _mm_shuffle_pi16(a, imm)                                       \
    vreinterpret_m64_s16(vshuffle_s16(                                 \
        vreinterpret_s16_m64(a), vreinterpret_s16_m64(a), (imm & 0x3), \
        ((imm >> 2) & 0x3), ((imm >> 4) & 0x3), ((imm >> 6) & 0x3)))
#else
# 定义宏_mm_shuffle_pi16，使用sse2neon_define1
#define _mm_shuffle_pi16(a, imm)                                              \
    _sse2neon_define1(                                                        \
        __m64, a, int16x4_t ret;                                              \
        ret = vmov_n_s16(                                                     \
            vget_lane_s16(vreinterpret_s16_m64(_a), (imm) & (0x3)));          \
        ret = vset_lane_s16(                                                  \
            vget_lane_s16(vreinterpret_s16_m64(_a), ((imm) >> 2) & 0x3), ret, \
            1);                                                               \
        ret = vset_lane_s16(                                                  \
            vget_lane_s16(vreinterpret_s16_m64(_a), ((imm) >> 4) & 0x3), ret, \
            2);                                                               \
        ret = vset_lane_s16(                                                  \
            vget_lane_s16(vreinterpret_s16_m64(_a), ((imm) >> 6) & 0x3), ret, \
            3);                                                               \
        _sse2neon_return(vreinterpret_m64_s16(ret));)
#endif

# 对于在此指令之前发出的所有存储到内存的指令执行序列化操作。保证每个存储指令都
# 已经被执行
// 执行一个序列化操作，确保在该指令之前发出的所有从内存加载和存储到内存的指令都在程序顺序上是全局可见的，位于该指令之后的任何内存指令之前。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sfence
FORCE_INLINE void _mm_sfence(void)
{
    _sse2neon_smp_mb();
}

// 执行一个序列化操作，确保在该指令之前发出的所有从内存加载和存储到内存的指令都在程序顺序上是全局可见的，位于该指令之后的任何内存指令之前。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mfence
FORCE_INLINE void _mm_mfence(void)
{
    _sse2neon_smp_mb();
}

// 执行一个序列化操作，确保在该指令之前发出的所有从内存加载的指令都在程序顺序上是全局可见的，位于该指令之后的任何加载指令之前。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_lfence
FORCE_INLINE void _mm_lfence(void)
{
    _sse2neon_smp_mb();
}

// 通过内联汇编实现的宏，用于执行 _mm_shuffle_ps 操作
#ifdef _sse2neon_shuffle
#define _mm_shuffle_ps(a, b, imm)                                              \
    __extension__({                                                            \
        float32x4_t _input1 = vreinterpretq_f32_m128(a);                       \
        float32x4_t _input2 = vreinterpretq_f32_m128(b);                       \
        float32x4_t _shuf =                                                    \
            vshuffleq_s32(_input1, _input2, (imm) & (0x3), ((imm) >> 2) & 0x3, \
                          (((imm) >> 4) & 0x3) + 4, (((imm) >> 6) & 0x3) + 4); \
        vreinterpretq_m128_f32(_shuf);                                         \
    })
// 如果不是 ARM 架构，使用通用的宏定义来实现 _mm_shuffle_ps
#else  // generic
#define _mm_shuffle_ps(a, b, imm)                            \
#endif

// 计算输入向量中每个单精度浮点数的平方根，并将结果存储在目标向量中
// 由于 ARMv7-A NEON 没有精确的平方根内联函数，我们通过将输入与其倒数平方根相乘来实现平方根，然后使用牛顿-拉弗森方法来近似结果。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sqrt_ps
FORCE_INLINE __m128 _mm_sqrt_ps(__m128 in)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，使用 NEON 指令来计算平方根
    return vreinterpretq_m128_f32(vsqrtq_f32(vreinterpretq_f32_m128(in)));
#else
    // 如果不是 ARM64 架构，使用 NEON 指令来计算平方根
    float32x4_t recip = vrsqrteq_f32(vreinterpretq_f32_m128(in));

    // 测试 vrsqrteq_f32(0) -> 正无穷大 的情况
    // 将其改为零，以便 s * 1/sqrt(s) 的结果也是零
    const uint32x4_t pos_inf = vdupq_n_u32(0x7F800000);
    const uint32x4_t div_by_zero =
        vceqq_u32(pos_inf, vreinterpretq_u32_f32(recip));
    recip = vreinterpretq_f32_u32(
        vandq_u32(vmvnq_u32(div_by_zero), vreinterpretq_u32_f32(recip)));

    recip = vmulq_f32(
        vrsqrtsq_f32(vmulq_f32(recip, recip), vreinterpretq_f32_m128(in)),
        recip);
    // 用于提高精度的额外牛顿-拉弗森迭代
    recip = vmulq_f32(
        vrsqrtsq_f32(vmulq_f32(recip, recip), vreinterpretq_f32_m128(in)),
        recip);

    // 平方根(s) = s * 1/sqrt(s)
    return vreinterpretq_m128_f32(vmulq_f32(vreinterpretq_f32_m128(in), recip));
#endif
}

// 计算输入向量中的低位单精度浮点数的平方根，将结果存储在目标向量的低位，并将输入向量的上3个元素复制到目标向量的上位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sqrt_ss
FORCE_INLINE __m128 _mm_sqrt_ss(__m128 in)
    # 将输入向量中的每个元素开平方，并取出第一个元素的值
    float32_t value =
        vgetq_lane_f32(vreinterpretq_f32_m128(_mm_sqrt_ps(in)), 0);
    # 将开平方后的值插入到输入向量的第一个位置，并返回结果向量
    return vreinterpretq_m128_f32(
        vsetq_lane_f32(value, vreinterpretq_f32_m128(in), 0));
// 存储从 a 中组成的 128 位数据（由 4 个打包的单精度（32 位）浮点元素组成）到内存中。mem_addr 必须对齐到 16 字节边界，否则可能会生成通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_store_ps
FORCE_INLINE void _mm_store_ps(float *p, __m128 a)
{
    vst1q_f32(p, vreinterpretq_f32_m128(a));
}

// 将 a 中的低单精度（32 位）浮点元素存储到内存中的 4 个连续元素中。mem_addr 必须对齐到 16 字节边界，否则可能会生成通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_store_ps1
FORCE_INLINE void _mm_store_ps1(float *p, __m128 a)
{
    float32_t a0 = vgetq_lane_f32(vreinterpretq_f32_m128(a), 0);
    vst1q_f32(p, vdupq_n_f32(a0));
}

// 将 a 中的低单精度（32 位）浮点元素存储到内存中。mem_addr 不需要对齐到特定边界。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_store_ss
FORCE_INLINE void _mm_store_ss(float *p, __m128 a)
{
    vst1q_lane_f32(p, vreinterpretq_f32_m128(a), 0);
}

// 将 a 中的低单精度（32 位）浮点元素存储到内存中的 4 个连续元素中。mem_addr 必须对齐到 16 字节边界，否则可能会生成通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_store1_ps
#define _mm_store1_ps _mm_store_ps1

// 将 a 中的上部 2 个单精度（32 位）浮点元素存储到内存中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_storeh_pi
FORCE_INLINE void _mm_storeh_pi(__m64 *p, __m128 a)
{
    *p = vreinterpret_m64_f32(vget_high_f32(a));
}

// 将 a 中的下部 2 个单精度（32 位）浮点元素存储到内存中。
// 将__m128类型的数据a的低64位存储到指针p指向的内存中
FORCE_INLINE void _mm_storel_pi(__m64 *p, __m128 a)
{
    *p = vreinterpret_m64_f32(vget_low_f32(a));
}

// 将__m128类型的数据a中的4个单精度浮点数以相反的顺序存储到内存中，要求内存地址p必须按16字节边界对齐，否则可能会生成通用保护异常
FORCE_INLINE void _mm_storer_ps(float *p, __m128 a)
{
    float32x4_t tmp = vrev64q_f32(vreinterpretq_f32_m128(a));
    float32x4_t rev = vextq_f32(tmp, tmp, 2);
    vst1q_f32(p, rev);
}

// 将__m128类型的数据a中的4个单精度浮点数以原始顺序存储到内存中，内存地址p不需要按特定边界对齐
FORCE_INLINE void _mm_storeu_ps(float *p, __m128 a)
{
    vst1q_f32(p, vreinterpretq_f32_m128(a));
}

// 将__m128i类型的数据a中的16位整数存储到地址p指向的内存中
FORCE_INLINE void _mm_storeu_si16(void *p, __m128i a)
{
    vst1q_lane_s16((int16_t *) p, vreinterpretq_s16_m128i(a), 0);
}

// 将__m128i类型的数据a中的64位整数存储到地址p指向的内存中
FORCE_INLINE void _mm_storeu_si64(void *p, __m128i a)
{
    vst1q_lane_s64((int64_t *) p, vreinterpretq_s64_m128i(a), 0);
}

// 将__m64类型的数据a存储到地址p指向的内存中，使用非临时内存提示
FORCE_INLINE void _mm_stream_pi(__m64 *p, __m64 a)
{
    vst1_s64((int64_t *) p, vreinterpret_s64_m64(a));
}
// 使用非临时内存提示将__m128类型的数据a存储到指针p指向的内存中
// 如果支持__builtin_nontemporal_store内建函数，则使用该函数进行非临时存储
// 否则使用vst1q_f32函数将a中的数据存储到p指向的内存中
FORCE_INLINE void _mm_stream_ps(float *p, __m128 a)
{
#if __has_builtin(__builtin_nontemporal_store)
    __builtin_nontemporal_store(a, (float32x4_t *) p);
#else
    vst1q_f32(p, vreinterpretq_f32_m128(a));
#endif
}

// 从a中减去b中对应位置的元素，并将结果存储到dst中
// 使用vsubq_f32函数实现减法操作，并使用vreinterpretq_m128_f32函数将结果转换为__m128类型
FORCE_INLINE __m128 _mm_sub_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_f32(
        vsubq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
}

// 从a中减去b中的低位元素，并将结果存储到dst的低位元素中，同时将a的高位元素复制到dst的高位元素中
// 使用_mm_sub_ps函数计算低位元素的减法操作，并使用_mm_move_ss函数将结果存储到dst的低位元素中
FORCE_INLINE __m128 _mm_sub_ss(__m128 a, __m128 b)
{
    return _mm_move_ss(a, _mm_sub_ps(a, b));
}

// 宏：对由row0、row1、row2和row3组成的4x4矩阵进行转置，并将转置后的矩阵存储到这些向量中
// 使用MM_TRANSPOSE4_PS宏可以将4x4矩阵进行转置操作
#define _MM_TRANSPOSE4_PS(row0, row1, row2, row3)         \
    # 使用 NEON 指令对输入的四个 float32x4_t 向量进行转置操作
    do {                                                  \
        # 将输入的两个 float32x4_t 向量按照 2x2 的顺序交错排列
        float32x4x2_t ROW01 = vtrnq_f32(row0, row1);      \
        float32x4x2_t ROW23 = vtrnq_f32(row2, row3);      \
        # 重新组合交错排列后的结果，得到转置后的第一行和第二行
        row0 = vcombine_f32(vget_low_f32(ROW01.val[0]),   \
                            vget_low_f32(ROW23.val[0]));  \
        row1 = vcombine_f32(vget_low_f32(ROW01.val[1]),   \
                            vget_low_f32(ROW23.val[1]));  \
        # 重新组合交错排列后的结果，得到转置后的第三行和第四行
        row2 = vcombine_f32(vget_high_f32(ROW01.val[0]),  \
                            vget_high_f32(ROW23.val[0])); \
        row3 = vcombine_f32(vget_high_f32(ROW01.val[1]),  \
                            vget_high_f32(ROW23.val[1])); \
    } while (0)
// 定义宏，将带有'u'的版本别名为对应的无'u'版本
#define _mm_ucomieq_ss _mm_comieq_ss
#define _mm_ucomige_ss _mm_comige_ss
#define _mm_ucomigt_ss _mm_comigt_ss
#define _mm_ucomile_ss _mm_comile_ss
#define _mm_ucomilt_ss _mm_comilt_ss
#define _mm_ucomineq_ss _mm_comineq_ss

// 返回一个具有未定义元素的__m128i类型的向量
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=mm_undefined_si128
FORCE_INLINE __m128i _mm_undefined_si128(void)
{
#if defined(__GNUC__) || defined(__clang__)
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wuninitialized"
#endif
    __m128i a;
#if defined(_MSC_VER)
    a = _mm_setzero_si128();
#endif
    return a;
#if defined(__GNUC__) || defined(__clang__)
#pragma GCC diagnostic pop
#endif
}

// 返回一个具有未定义元素的__m128类型的向量
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_undefined_ps
FORCE_INLINE __m128 _mm_undefined_ps(void)
{
#if defined(__GNUC__) || defined(__clang__)
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wuninitialized"
#endif
    __m128 a;
#if defined(_MSC_VER)
    a = _mm_setzero_ps();
#endif
    return a;
#if defined(__GNUC__) || defined(__clang__)
#pragma GCC diagnostic pop
#endif
}

// 从a和b的高半部分解压和交错单精度（32位）浮点元素，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_unpackhi_ps
FORCE_INLINE __m128 _mm_unpackhi_ps(__m128 a, __m128 b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128_f32(
        vzip2q_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
#else
    float32x2_t a1 = vget_high_f32(vreinterpretq_f32_m128(a));
    float32x2_t b1 = vget_high_f32(vreinterpretq_f32_m128(b));
    float32x2x2_t result = vzip_f32(a1, b1);
    # 将两个128位的浮点数向量合并成一个新的128位浮点数向量，并返回结果
    return vreinterpretq_m128_f32(vcombine_f32(result.val[0], result.val[1]));
#endif
}

// 从 a 和 b 的低半部分解压和交错单精度（32位）浮点元素，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_unpacklo_ps
FORCE_INLINE __m128 _mm_unpacklo_ps(__m128 a, __m128 b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128_f32(
        vzip1q_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
#else
    float32x2_t a1 = vget_low_f32(vreinterpretq_f32_m128(a));
    float32x2_t b1 = vget_low_f32(vreinterpretq_f32_m128(b));
    float32x2x2_t result = vzip_f32(a1, b1);
    return vreinterpretq_m128_f32(vcombine_f32(result.val[0], result.val[1]));
#endif
}

// 计算 a 和 b 中打包的单精度（32位）浮点元素的按位异或，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_xor_ps
FORCE_INLINE __m128 _mm_xor_ps(__m128 a, __m128 b)
{
    return vreinterpretq_m128_s32(
        veorq_s32(vreinterpretq_s32_m128(a), vreinterpretq_s32_m128(b)));
}

/* SSE2 */

// 在 a 和 b 中添加打包的16位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_add_epi16
FORCE_INLINE __m128i _mm_add_epi16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s16(
        vaddq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
}

// 在 a 和 b 中添加打包的32位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_add_epi32
FORCE_INLINE __m128i _mm_add_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s32(
        vaddq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 在 a 和 b 中添加打包的64位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_add_epi64
// 使用 SIMD 指令实现 128 位整数的加法操作
FORCE_INLINE __m128i _mm_add_epi64(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s64(
        vaddq_s64(vreinterpretq_s64_m128i(a), vreinterpretq_s64_m128i(b)));
}

// 使用 SIMD 指令实现 128 位整数的加法操作
FORCE_INLINE __m128i _mm_add_epi8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s8(
        vaddq_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
}

// 使用 SIMD 指令实现 128 位双精度浮点数的加法操作
FORCE_INLINE __m128d _mm_add_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(
        vaddq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    double *da = (double *) &a;
    double *db = (double *) &b;
    double c[2];
    c[0] = da[0] + db[0];
    c[1] = da[1] + db[1];
    return vld1q_f32((float32_t *) c);
#endif
}

// 使用 SIMD 指令实现 128 位双精度浮点数的加法操作
FORCE_INLINE __m128d _mm_add_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return _mm_move_sd(a, _mm_add_pd(a, b));
#else
    double *da = (double *) &a;
    double *db = (double *) &b;
    double c[2];
    c[0] = da[0] + db[0];
    c[1] = da[1];
    return vld1q_f32((float32_t *) c);
#endif
}

// 使用 SIMD 指令实现 64 位整数的加法操作
FORCE_INLINE __m64 _mm_add_si64(__m64 a, __m64 b)
{
    # 将参数a和b从m64类型转换为s64类型，然后相加
    return vreinterpret_m64_s64(
        vadd_s64(vreinterpret_s64_m64(a), vreinterpret_s64_m64(b)));
// 通过饱和加法，将a和b中的打包有符号16位整数相加，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_adds_epi16
FORCE_INLINE __m128i _mm_adds_epi16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s16(
        vqaddq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
}

// 通过饱和加法，将a和b中的打包有符号8位整数相加，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_adds_epi8
FORCE_INLINE __m128i _mm_adds_epi8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s8(
        vqaddq_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
}

// 通过饱和加法，将a和b中的打包无符号16位整数相加，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_adds_epu16
FORCE_INLINE __m128i _mm_adds_epu16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u16(
        vqaddq_u16(vreinterpretq_u16_m128i(a), vreinterpretq_u16_m128i(b)));
}

// 通过饱和加法，将a和b中的打包无符号8位整数相加，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_adds_epu8
FORCE_INLINE __m128i _mm_adds_epu8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u8(
        vqaddq_u8(vreinterpretq_u8_m128i(a), vreinterpretq_u8_m128i(b)));
}

// 计算a和b中的打包双精度（64位）浮点元素的按位AND，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_and_pd
FORCE_INLINE __m128d _mm_and_pd(__m128d a, __m128d b)
{
    return vreinterpretq_m128d_s64(
        vandq_s64(vreinterpretq_s64_m128d(a), vreinterpretq_s64_m128d(b)));
}

// 计算a和b中的128位（表示整数数据）的按位AND，并将结果存储在dst中
// 使用 SSE 指令集实现按位与操作，对两个 128 位整数寄存器进行按位与操作
FORCE_INLINE __m128i _mm_and_si128(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s32(
        vandq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 使用 SSE 指令集实现按位非和按位与操作，对两个 128 位双精度浮点数寄存器进行按位非和按位与操作
FORCE_INLINE __m128d _mm_andnot_pd(__m128d a, __m128d b)
{
    // *NOTE* 参数交换
    return vreinterpretq_m128d_s64(
        vbicq_s64(vreinterpretq_s64_m128d(b), vreinterpretq_s64_m128d(a)));
}

// 使用 SSE 指令集实现按位非和按位与操作，对两个 128 位整数寄存器进行按位非和按位与操作
FORCE_INLINE __m128i _mm_andnot_si128(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s32(
        vbicq_s32(vreinterpretq_s32_m128i(b),
                  vreinterpretq_s32_m128i(a)));  // *NOTE* 参数交换
}

// 使用 SSE 指令集实现无符号 16 位整数的平均值计算，对两个 128 位整数寄存器进行平均值计算
FORCE_INLINE __m128i _mm_avg_epu16(__m128i a, __m128i b)
{
    return (__m128i) vrhaddq_u16(vreinterpretq_u16_m128i(a),
                                 vreinterpretq_u16_m128i(b));
}

// 使用 SSE 指令集实现无符号 8 位整数的平均值计算，对两个 128 位整数寄存器进行平均值计算
FORCE_INLINE __m128i _mm_avg_epu8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u8(
        vrhaddq_u8(vreinterpretq_u8_m128i(a), vreinterpretq_u8_m128i(b)));
}

// 使用 SSE 指令集实现按字节左移操作，将寄存器 a 按 imm8 指定的字节数左移，空出的位用 0 填充
// 定义宏，将参数 a 向左移动 imm 个字节，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_bslli_si128
#define _mm_bslli_si128(a, imm) _mm_slli_si128(a, imm)

// 定义宏，将参数 a 向右移动 imm 个字节，并在移动过程中填充零，将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_bsrli_si128
#define _mm_bsrli_si128(a, imm) _mm_srli_si128(a, imm)

// 将类型为 __m128d 的向量转换为类型为 __m128 的向量。此内联函数仅用于编译，不生成任何指令，因此具有零延迟。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_castpd_ps
FORCE_INLINE __m128 _mm_castpd_ps(__m128d a)
{
    return vreinterpretq_m128_s64(vreinterpretq_s64_m128d(a));
}

// 将类型为 __m128d 的向量转换为类型为 __m128i 的向量。此内联函数仅用于编译，不生成任何指令，因此具有零延迟。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_castpd_si128
FORCE_INLINE __m128i _mm_castpd_si128(__m128d a)
{
    return vreinterpretq_m128i_s64(vreinterpretq_s64_m128d(a));
}

// 将类型为 __m128 的向量转换为类型为 __m128d 的向量。此内联函数仅用于编译，不生成任何指令，因此具有零延迟。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_castps_pd
FORCE_INLINE __m128d _mm_castps_pd(__m128 a)
{
    return vreinterpretq_m128d_s32(vreinterpretq_s32_m128(a));
}

// 将类型为 __m128 的向量转换为类型为 __m128i 的向量。此内联函数仅用于编译，不生成任何指令，因此具有零延迟。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_castps_si128
FORCE_INLINE __m128i _mm_castps_si128(__m128 a)
{
    return vreinterpretq_m128i_s32(vreinterpretq_s32_m128(a));
}

// 将类型为 __m128i 的向量转换为类型为 __m128d 的向量。此内联函数仅用于编译，不生成任何指令，因此具有零延迟。
// 将类型为__m128i的向量转换为类型为__m128d的向量。此内联函数仅用于编译，不生成任何指令，因此具有零延迟。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_castsi128_pd
FORCE_INLINE __m128d _mm_castsi128_pd(__m128i a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(vreinterpretq_f64_m128i(a));
#else
    return vreinterpretq_m128d_f32(vreinterpretq_f32_m128i(a));
#endif
}

// 将类型为__m128i的向量转换为类型为__m128的向量。此内联函数仅用于编译，不生成任何指令，因此具有零延迟。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_castsi128_ps
FORCE_INLINE __m128 _mm_castsi128_ps(__m128i a)
{
    return vreinterpretq_m128_s32(vreinterpretq_s32_m128i(a));
}

// 使包含p的缓存行在缓存层次结构的所有级别上失效并刷新。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_clflush
#if defined(__APPLE__)
#include <libkern/OSCacheControl.h>
#endif
FORCE_INLINE void _mm_clflush(void const *p)
{
    (void) p;

    /* sys_icache_invalidate自macOS 10.5起受支持。
     * 但是，在非越狱的iOS设备上无法使用，尽管编译是成功的。
     */
#if defined(__APPLE__)
    sys_icache_invalidate((void *) (uintptr_t) p, SSE2NEON_CACHELINE_SIZE);
#elif defined(__GNUC__) || defined(__clang__)
    uintptr_t ptr = (uintptr_t) p;
    __builtin___clear_cache((char *) ptr,
                            (char *) ptr + SSE2NEON_CACHELINE_SIZE);
#elif (_MSC_VER) && SSE2NEON_INCLUDE_WINDOWS_H
    FlushInstructionCache(GetCurrentProcess(), p, SSE2NEON_CACHELINE_SIZE);
#endif
}

// 比较a和b中打包的16位整数是否相等，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpeq_epi16
FORCE_INLINE __m128i _mm_cmpeq_epi16(__m128i a, __m128i b)
{
    # 将两个128位寄存器中的有符号16位整数转换为无符号16位整数，并比较它们的相等性
    return vreinterpretq_m128i_u16(
        vceqq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
// 比较两个打包的32位整数a和b是否相等，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpeq_epi32
FORCE_INLINE __m128i _mm_cmpeq_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u32(
        vceqq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 比较两个打包的8位整数a和b是否相等，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpeq_epi8
FORCE_INLINE __m128i _mm_cmpeq_epi8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u8(
        vceqq_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
}

// 比较两个打包的双精度（64位）浮点元素a和b是否相等，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpeq_pd
FORCE_INLINE __m128d _mm_cmpeq_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_u64(
        vceqq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    // (a == b) -> (a_lo == b_lo) && (a_hi == b_hi)
    uint32x4_t cmp =
        vceqq_u32(vreinterpretq_u32_m128d(a), vreinterpretq_u32_m128d(b));
    uint32x4_t swapped = vrev64q_u32(cmp);
    return vreinterpretq_m128d_u32(vandq_u32(cmp, swapped));
#endif
}

// 比较a和b的较低双精度（64位）浮点元素是否相等，将结果存储在dst的较低元素中，并将a的较高元素复制到dst的较高元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpeq_sd
FORCE_INLINE __m128d _mm_cmpeq_sd(__m128d a, __m128d b)
{
    return _mm_move_sd(a, _mm_cmpeq_pd(a, b));
}

// 比较两个打包的双精度（64位）浮点元素a和b是否大于或等于，并将结果存储在dst中
// 定义一个名为 _mm_cmpge_pd 的函数，用于比较两个 __m128d 类型的参数 a 和 b 的大于或等于关系
FORCE_INLINE __m128d _mm_cmpge_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用指令进行比较并返回结果
    return vreinterpretq_m128d_u64(
        vcgeq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    // 如果不是 ARM64 架构，则进行以下操作
    // 将 a 的低位和高位分别转换为 uint64_t 类型
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    // 将 b 的低位和高位分别转换为 uint64_t 类型
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    // 创建一个长度为 2 的 uint64_t 数组
    uint64_t d[2];
    // 如果 a0 大于等于 b0，则将 d[0] 设置为全 1，否则设置为 0
    d[0] = (*(double *) &a0) >= (*(double *) &b0) ? ~UINT64_C(0) : UINT64_C(0);
    // 如果 a1 大于等于 b1，则将 d[1] 设置为全 1，否则设置为 0
    d[1] = (*(double *) &a1) >= (*(double *) &b1) ? ~UINT64_C(0) : UINT64_C(0);
    // 将数组 d 转换为 __m128d 类型并返回
    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较两个 __m128d 类型参数 a 和 b 的低位双精度浮点数的大于或等于关系
// 将结果存储在 dst 的低位，并将 a 的高位复制到 dst 的高位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpge_sd
FORCE_INLINE __m128d _mm_cmpge_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则调用 _mm_cmpge_pd 函数进行比较，并将结果移动到 dst 的低位
    return _mm_move_sd(a, _mm_cmpge_pd(a, b));
#else
    // 如果不是 ARM64 架构，则进行以下操作
    // 将 a 的低位和高位分别转换为 uint64_t 类型
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    // 将 b 的低位和高位分别转换为 uint64_t 类型
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    // 创建一个长度为 2 的 uint64_t 数组
    uint64_t d[2];
    // 如果 a0 大于等于 b0，则将 d[0] 设置为全 1，否则设置为 0
    d[0] = (*(double *) &a0) >= (*(double *) &b0) ? ~UINT64_C(0) : UINT64_C(0);
    // 将 a1 复制到 d[1]
    d[1] = a1;
    // 将数组 d 转换为 __m128d 类型并返回
    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较两个参数 a 和 b 中的打包的有符号 16 位整数，判断是否大于，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpgt_epi16
// 使用 SIMD 指令比较两个 16 位有符号整数向量 a 和 b 的大于关系，并将结果存储在 dst 中
FORCE_INLINE __m128i _mm_cmpgt_epi16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u16(
        vcgtq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
}

// 使用 SIMD 指令比较两个 32 位有符号整数向量 a 和 b 的大于关系，并将结果存储在 dst 中
FORCE_INLINE __m128i _mm_cmpgt_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u32(
        vcgtq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 使用 SIMD 指令比较两个 8 位有符号整数向量 a 和 b 的大于关系，并将结果存储在 dst 中
FORCE_INLINE __m128i _mm_cmpgt_epi8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u8(
        vcgtq_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
}

// 使用 SIMD 指令比较两个双精度（64 位）浮点数向量 a 和 b 的大于关系，并将结果存储在 dst 中
FORCE_INLINE __m128d _mm_cmpgt_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_u64(
        vcgtq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    // 如果不是 ARM64 架构，则使用标量方式比较双精度浮点数
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    // 比较双精度浮点数并存储结果
    d[0] = (*(double *) &a0) > (*(double *) &b0) ? ~UINT64_C(0) : UINT64_C(0);
    d[1] = (*(double *) &a1) > (*(double *) &b1) ? ~UINT64_C(0) : UINT64_C(0);

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较两个双精度（64 位）浮点数向量 a 和 b 的低位元素的大于关系
// 使用 SIMD 指令比较两个双精度浮点数的大小，将结果存储在目标寄存器的低位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpgt_sd
FORCE_INLINE __m128d _mm_cmpgt_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，使用 _mm_cmpgt_pd() 和 _mm_move_sd() 函数进行比较
    return _mm_move_sd(a, _mm_cmpgt_pd(a, b));
#else
    // 如果不是 ARM64 架构，使用位操作和类型转换进行比较
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    d[0] = (*(double *) &a0) > (*(double *) &b0) ? ~UINT64_C(0) : UINT64_C(0);
    d[1] = a1;

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 使用 SIMD 指令比较两个双精度浮点数的大小，将结果存储在目标寄存器中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmple_pd
FORCE_INLINE __m128d _mm_cmple_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，使用 _mm_cmple_pd() 函数进行比较
    return vreinterpretq_m128d_u64(
        vcleq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    // 如果不是 ARM64 架构，使用位操作和类型转换进行比较
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    d[0] = (*(double *) &a0) <= (*(double *) &b0) ? ~UINT64_C(0) : UINT64_C(0);
    d[1] = (*(double *) &a1) <= (*(double *) &b1) ? ~UINT64_C(0) : UINT64_C(0);

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 使用 SIMD 指令比较两个双精度浮点数的大小，将结果存储在目标寄存器的低位
// 将 a 的上半部分复制到 dst 的上半部分
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmple_sd
FORCE_INLINE __m128d _mm_cmple_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 _mm_cmple_pd() 函数
    return _mm_move_sd(a, _mm_cmple_pd(a, b));
#else
    // 否则，展开 "_mm_cmpge_pd()" 以减少不必要的操作
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    // 比较 a0 和 b0，如果 a0 <= b0，则将 d[0] 设置为全1，否则设置为全0
    d[0] = (*(double *) &a0) <= (*(double *) &b0) ? ~UINT64_C(0) : UINT64_C(0);
    d[1] = a1;

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较打包的有符号 16 位整数 a 和 b，如果 a < b，则将结果存储在 dst 中
// 注意：此内联函数会发出带有操作数顺序交换的 pcmpgtw 指令
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmplt_epi16
FORCE_INLINE __m128i _mm_cmplt_epi16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u16(
        vcltq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
}

// 比较打包的有符号 32 位整数 a 和 b，如果 a < b，则将结果存储在 dst 中
// 注意：此内联函数会发出带有操作数顺序交换的 pcmpgtd 指令
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmplt_epi32
FORCE_INLINE __m128i _mm_cmplt_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u32(
        vcltq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 比较打包的有符号 8 位整数 a 和 b，如果 a < b，则将结果存储在 dst 中
// 注意：此内联函数会发出带有操作数顺序交换的 pcmpgtb 指令
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmplt_epi8
// 使用 FORCE_INLINE 宏定义内联函数，比较两个 __m128i 类型的参数 a 和 b 中的每个字节是否小于，返回结果
FORCE_INLINE __m128i _mm_cmplt_epi8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u8(
        vcltq_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
}

// 比较两个 __m128d 类型的参数 a 和 b 中的双精度浮点数是否小于，返回结果
// 如果是 ARM64 架构，则使用 vcltq_f64 函数进行比较，否则进行手动比较
FORCE_INLINE __m128d _mm_cmplt_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_u64(
        vcltq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    d[0] = (*(double *) &a0) < (*(double *) &b0) ? ~UINT64_C(0) : UINT64_C(0);
    d[1] = (*(double *) &a1) < (*(double *) &b1) ? ~UINT64_C(0) : UINT64_C(0);

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较两个 __m128d 类型的参数 a 和 b 中的双精度浮点数是否小于，返回结果
// 如果是 ARM64 架构，则使用 _mm_move_sd 函数进行比较，否则进行手动比较
FORCE_INLINE __m128d _mm_cmplt_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return _mm_move_sd(a, _mm_cmplt_pd(a, b));
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    d[0] = (*(double *) &a0) < (*(double *) &b0) ? ~UINT64_C(0) : UINT64_C(0);
    d[1] = a1;

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}
// 比较两个 packed double-precision (64-bit) 浮点数元素 a 和 b 是否不相等，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpneq_pd
FORCE_INLINE __m128d _mm_cmpneq_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpretq_m128d_s32(vmvnq_s32(vreinterpretq_s32_u64(
        vceqq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)))));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    // (a == b) -> (a_lo == b_lo) && (a_hi == b_hi)
    uint32x4_t cmp =
        vceqq_u32(vreinterpretq_u32_m128d(a), vreinterpretq_u32_m128d(b));
    uint32x4_t swapped = vrev64q_u32(cmp);
    return vreinterpretq_m128d_u32(vmvnq_u32(vandq_u32(cmp, swapped)));
#endif
}

// 比较两个 double-precision (64-bit) 浮点数元素 a 和 b 的低位是否不相等，将结果存储在 dst 的低位，并将 a 的高位复制到 dst 的高位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpneq_sd
FORCE_INLINE __m128d _mm_cmpneq_sd(__m128d a, __m128d b)
{
    return _mm_move_sd(a, _mm_cmpneq_pd(a, b));
}

// 比较两个 packed double-precision (64-bit) 浮点数元素 a 和 b 是否不大于或等于，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnge_pd
FORCE_INLINE __m128d _mm_cmpnge_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpretq_m128d_u64(veorq_u64(
        vcgeq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)),
        vdupq_n_u64(UINT64_MAX)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    # 将a0和b0的值转换为double类型后进行比较，如果a0小于等于b0，则d[0]的值为全1，否则为0
    d[0] = !((*(double *) &a0) >= (*(double *) &b0)) ? ~UINT64_C(0) : UINT64_C(0);
    # 将a1和b1的值转换为double类型后进行比较，如果a1小于等于b1，则d[1]的值为全1，否则为0
    d[1] = !((*(double *) &a1) >= (*(double *) &b1)) ? ~UINT64_C(0) : UINT64_C(0);
    # 将d数组中的值转换为128位的双精度浮点数，返回结果
    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较 a 和 b 中的双精度（64位）浮点数元素，判断是否不大于或等于，将结果存储在 dst 的低位元素中，并将 a 的高位元素复制到 dst 的高位元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnge_sd
FORCE_INLINE __m128d _mm_cmpnge_sd(__m128d a, __m128d b)
{
    return _mm_move_sd(a, _mm_cmpnge_pd(a, b));
}

// 比较 a 和 b 中的打包双精度（64位）浮点数元素，判断是否不大于，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_cmpngt_pd
FORCE_INLINE __m128d _mm_cmpngt_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_u64(veorq_u64(
        vcgtq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)),
        vdupq_n_u64(UINT64_MAX)));
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    d[0] =
        !((*(double *) &a0) > (*(double *) &b0)) ? ~UINT64_C(0) : UINT64_C(0);
    d[1] =
        !((*(double *) &a1) > (*(double *) &b1)) ? ~UINT64_C(0) : UINT64_C(0);

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较 a 和 b 中的双精度（64位）浮点数元素，判断是否不大于，将结果存储在 dst 的低位元素中，并将 a 的高位元素复制到 dst 的高位元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpngt_sd
FORCE_INLINE __m128d _mm_cmpngt_sd(__m128d a, __m128d b)
{
    return _mm_move_sd(a, _mm_cmpngt_pd(a, b));
}

// 比较 a 和 b 中的打包双精度（64位）浮点数元素
// 使用 SIMD 指令比较两个 __m128d 类型的变量 a 和 b 的值，如果 a 中的值不小于等于 b 中的值，则将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnle_pd
FORCE_INLINE __m128d _mm_cmpnle_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 ARM64 指令进行比较
    return vreinterpretq_m128d_u64(veorq_u64(
        vcleq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)),
        vdupq_n_u64(UINT64_MAX)));
#else
    // 如果不是 ARM64 架构，则使用普通的比较方式
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    // 对 a0 和 b0 进行比较，将结果存储在 d[0] 中
    d[0] =
        !((*(double *) &a0) <= (*(double *) &b0)) ? ~UINT64_C(0) : UINT64_C(0);
    // 对 a1 和 b1 进行比较，将结果存储在 d[1] 中
    d[1] =
        !((*(double *) &a1) <= (*(double *) &b1)) ? ~UINT64_C(0) : UINT64_C(0);

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 使用 SIMD 指令比较两个 __m128d 类型的变量 a 和 b 的值，如果 a 中的值不小于等于 b 中的值，则将结果存储在 dst 的低位中，同时将 a 的高位复制到 dst 的高位中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnle_sd
FORCE_INLINE __m128d _mm_cmpnle_sd(__m128d a, __m128d b)
{
    return _mm_move_sd(a, _mm_cmpnle_pd(a, b));
}

// 使用 SIMD 指令比较两个 __m128d 类型的变量 a 和 b 的值，如果 a 中的值不小于 b 中的值，则将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnlt_pd
FORCE_INLINE __m128d _mm_cmpnlt_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 ARM64 指令进行比较
    return vreinterpretq_m128d_u64(veorq_u64(
        vcltq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)),
        vdupq_n_u64(UINT64_MAX)));
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    # 将参数 a 的高位转换为 64 位整数
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    # 将参数 b 的低位转换为 64 位整数
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    # 将参数 b 的高位转换为 64 位整数
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    # 创建一个长度为 2 的 64 位整数数组
    uint64_t d[2];
    # 比较 a0 和 b0 的值，如果 a0 不小于 b0，则将 d[0] 设置为全 1，否则设置为 0
    d[0] =
        !((*(double *) &a0) < (*(double *) &b0)) ? ~UINT64_C(0) : UINT64_C(0);
    # 比较 a1 和 b1 的值，如果 a1 不小于 b1，则将 d[1] 设置为全 1，否则设置为 0
    d[1] =
        !((*(double *) &a1) < (*(double *) &b1)) ? ~UINT64_C(0) : UINT64_C(0);
    # 将 64 位整数数组转换为 128 位双精度浮点数向量并返回
    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较两个双精度（64位）浮点数的低位元素，判断是否不小于，将结果存储在dst的低位元素，并将a的高位元素复制到dst的高位元素
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpnlt_sd
FORCE_INLINE __m128d _mm_cmpnlt_sd(__m128d a, __m128d b)
{
    return _mm_move_sd(a, _mm_cmpnlt_pd(a, b));
}

// 比较打包的双精度（64位）浮点数a和b，判断是否都不是NaN，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpord_pd
FORCE_INLINE __m128d _mm_cmpord_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 排除NaN，任何两个浮点数都可以进行比较
    uint64x2_t not_nan_a =
        vceqq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(a));
    uint64x2_t not_nan_b =
        vceqq_f64(vreinterpretq_f64_m128d(b), vreinterpretq_f64_m128d(b));
    return vreinterpretq_m128d_u64(vandq_u64(not_nan_a, not_nan_b));
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    d[0] = ((*(double *) &a0) == (*(double *) &a0) &&
            (*(double *) &b0) == (*(double *) &b0))
               ? ~UINT64_C(0)
               : UINT64_C(0);
    d[1] = ((*(double *) &a1) == (*(double *) &a1) &&
            (*(double *) &b1) == (*(double *) &b1))
               ? ~UINT64_C(0)
               : UINT64_C(0);

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较两个双精度（64位）浮点数的低位元素，判断是否都不是NaN，将结果存储在dst的低位元素，并
// 将源寄存器 a 和 b 中的上半部分复制到目标寄存器的上半部分
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpord_sd
FORCE_INLINE __m128d _mm_cmpord_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则调用 _mm_cmpord_pd 函数
    return _mm_move_sd(a, _mm_cmpord_pd(a, b));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t d[2];
    // 检查 a0 和 b0 是否为 NaN，如果是则将 d[0] 置为全 1，否则置为全 0
    d[0] = ((*(double *) &a0) == (*(double *) &a0) &&
            (*(double *) &b0) == (*(double *) &b0))
               ? ~UINT64_C(0)
               : UINT64_C(0);
    d[1] = a1;

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较两个 __m128d 类型的寄存器 a 和 b 中的双精度浮点数元素，判断是否有 NaN，并将结果存储在目标寄存器中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpunord_pd
FORCE_INLINE __m128d _mm_cmpunord_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 两个 NaN 在比较操作中不相等
    uint64x2_t not_nan_a =
        vceqq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(a));
    uint64x2_t not_nan_b =
        vceqq_f64(vreinterpretq_f64_m128d(b), vreinterpretq_f64_m128d(b));
    return vreinterpretq_m128d_s32(
        vmvnq_s32(vreinterpretq_s32_u64(vandq_u64(not_nan_a, not_nan_b))));
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    // 检查 a0 和 b0 是否为 NaN，如果是则将 d[0] 置为全 0，否则置为全 1
    d[0] = ((*(double *) &a0) == (*(double *) &a0) &&
            (*(double *) &b0) == (*(double *) &b0))
               ? UINT64_C(0)
               : ~UINT64_C(0);
    # 将 a1 和 b1 的地址转换为 double 类型的指针，然后比较它们的值是否相等，如果相等则将 d[1] 设置为 0，否则设置为 0 的补码
    d[1] = ((*(double *) &a1) == (*(double *) &a1) &&
            (*(double *) &b1) == (*(double *) &b1))
               ? UINT64_C(0)
               : ~UINT64_C(0);

    # 将 d 转换为 128 位的无符号整数类型，并加载到寄存器中，然后返回结果
    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较 a 和 b 中的低位双精度（64位）浮点元素，查看是否有任何一个是 NaN，将结果存储在 dst 的低位元素中，并将 a 的高位元素复制到 dst 的高位元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpunord_sd
FORCE_INLINE __m128d _mm_cmpunord_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return _mm_move_sd(a, _mm_cmpunord_pd(a, b));
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));  // 获取 a 的低位 64 位数据
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));  // 获取 b 的低位 64 位数据
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));  // 获取 a 的高位 64 位数据
    uint64_t d[2];
    d[0] = ((*(double *) &a0) == (*(double *) &a0) &&  // 判断 a0 是否为 NaN
            (*(double *) &b0) == (*(double *) &b0))  // 判断 b0 是否为 NaN
               ? UINT64_C(0)  // 如果都不是 NaN，则为 0
               : ~UINT64_C(0);  // 如果有一个是 NaN，则为 1
    d[1] = a1;  // 将 a1 赋值给 d 的第二个元素

    return vreinterpretq_m128d_u64(vld1q_u64(d));  // 将 d 转换为 __m128d 类型并返回
#endif
}

// 比较 a 和 b 中的低位双精度（64位）浮点元素，判断是否大于或等于，并返回布尔结果（0 或 1）。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_comige_sd
FORCE_INLINE int _mm_comige_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vgetq_lane_u64(vcgeq_f64(a, b), 0) & 0x1;  // 返回 a 和 b 中的低位双精度浮点数是否大于或等于的布尔结果
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));  // 获取 a 的低位 64 位数据
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));  // 获取 b 的低位 64 位数据

    return (*(double *) &a0 >= *(double *) &b0);  // 返回 a0 是否大于或等于 b0 的布尔结果
#endif
}

// 比较 a 和 b 中的低位双精度（64位）浮点元素，判断是否大于，并返回布尔结果（0 或 1）。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_comigt_sd
FORCE_INLINE int _mm_comigt_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vgetq_lane_u64(vcgtq_f64(a, b), 0) & 0x1;  // 返回 a 和 b 中的低位双精度浮点数是否大于的布尔结果
#else
    # 将128位寄存器a的低64位转换为64位整数
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    # 将128位寄存器b的低64位转换为64位整数
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));

    # 将a0和b0强制转换为double类型指针，然后比较它们指向的double值的大小
    return (*(double *) &a0 > *(double *) &b0);
#endif
}

// 比较 a 和 b 中的低位双精度（64位）浮点数元素，判断是否小于或等于，并返回布尔结果（0或1）
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_comile_sd
FORCE_INLINE int _mm_comile_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vgetq_lane_u64(vcleq_f64(a, b), 0) & 0x1;
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));

    return (*(double *) &a0 <= *(double *) &b0);
#endif
}

// 比较 a 和 b 中的低位双精度（64位）浮点数元素，判断是否小于，并返回布尔结果（0或1）
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_comilt_sd
FORCE_INLINE int _mm_comilt_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vgetq_lane_u64(vcltq_f64(a, b), 0) & 0x1;
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));

    return (*(double *) &a0 < *(double *) &b0);
#endif
}

// 比较 a 和 b 中的低位双精度（64位）浮点数元素，判断是否相等，并返回布尔结果（0或1）
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_comieq_sd
FORCE_INLINE int _mm_comieq_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vgetq_lane_u64(vceqq_f64(a, b), 0) & 0x1;
#else
    uint32x4_t a_not_nan =
        vceqq_u32(vreinterpretq_u32_m128d(a), vreinterpretq_u32_m128d(a));
    uint32x4_t b_not_nan =
        vceqq_u32(vreinterpretq_u32_m128d(b), vreinterpretq_u32_m128d(b));
    uint32x4_t a_and_b_not_nan = vandq_u32(a_not_nan, b_not_nan);
    uint32x4_t a_eq_b =
        vceqq_u32(vreinterpretq_u32_m128d(a), vreinterpretq_u32_m128d(b));
    # 使用 NEON 指令进行两个 128 位无符号整数向量的按位与操作
    uint64x2_t and_results = vandq_u64(vreinterpretq_u64_u32(a_and_b_not_nan),
                                       vreinterpretq_u64_u32(a_eq_b));
    # 从结果向量中获取指定索引位置的 64 位整数，并与 0x1 进行按位与操作，返回结果
    return vgetq_lane_u64(and_results, 0) & 0x1;
#endif
}

// 比较两个双精度（64位）浮点数元素a和b的低位，判断是否不相等，并返回布尔结果（0或1）
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_comineq_sd
FORCE_INLINE int _mm_comineq_sd(__m128d a, __m128d b)
{
    return !_mm_comieq_sd(a, b);
}

// 将a中的打包的有符号32位整数转换为打包的双精度（64位）浮点数元素，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepi32_pd
FORCE_INLINE __m128d _mm_cvtepi32_pd(__m128i a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(
        vcvtq_f64_s64(vmovl_s32(vget_low_s32(vreinterpretq_s32_m128i(a)))));
#else
    double a0 = (double) vgetq_lane_s32(vreinterpretq_s32_m128i(a), 0);
    double a1 = (double) vgetq_lane_s32(vreinterpretq_s32_m128i(a), 1);
    return _mm_set_pd(a1, a0);
#endif
}

// 将a中的打包的有符号32位整数转换为打包的单精度（32位）浮点数元素，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepi32_ps
FORCE_INLINE __m128 _mm_cvtepi32_ps(__m128i a)
{
    return vreinterpretq_m128_f32(vcvtq_f32_s32(vreinterpretq_s32_m128i(a)));
}

// 将a中的打包的双精度（64位）浮点数元素转换为打包的32位整数，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpd_epi32
FORCE_INLINE __m128i _mm_cvtpd_epi32(__m128d a)
{
// 在clang上不支持vrnd32xq_f64
#if defined(__ARM_FEATURE_FRINT) && !defined(__clang__)
    float64x2_t rounded = vrnd32xq_f64(vreinterpretq_f64_m128d(a));
    int64x2_t integers = vcvtq_s64_f64(rounded);
    return vreinterpretq_m128i_s32(
        vcombine_s32(vmovn_s64(integers), vdup_n_s32(0)));
#else
    __m128d rnd = _mm_round_pd(a, _MM_FROUND_CUR_DIRECTION);
    # 将随机数rnd的内存表示转换为双精度浮点数，并取出第一个双精度浮点数
    double d0 = ((double *) &rnd)[0];
    # 将随机数rnd的内存表示转换为双精度浮点数，并取出第二个双精度浮点数
    double d1 = ((double *) &rnd)[1];
    # 使用两个32位整数和两个双精度浮点数的值创建一个128位整数
    return _mm_set_epi32(0, 0, (int32_t) d1, (int32_t) d0);
#endif
}

// 将 a 中的双精度（64位）浮点数转换为打包的32位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpd_pi32
FORCE_INLINE __m64 _mm_cvtpd_pi32(__m128d a)
{
    // 对 a 进行舍入
    __m128d rnd = _mm_round_pd(a, _MM_FROUND_CUR_DIRECTION);
    // 从 rnd 中提取双精度浮点数
    double d0 = ((double *) &rnd)[0];
    double d1 = ((double *) &rnd)[1];
    // 创建一个包含 d0 和 d1 的整型数组
    int32_t ALIGN_STRUCT(16) data[2] = {(int32_t) d0, (int32_t) d1};
    // 将整型数组转换为 m64 类型并返回
    return vreinterpret_m64_s32(vld1_s32(data));
}

// 将 a 中的双精度（64位）浮点数转换为打包的单精度（32位）浮点数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpd_ps
FORCE_INLINE __m128 _mm_cvtpd_ps(__m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    float32x2_t tmp = vcvt_f32_f64(vreinterpretq_f64_m128d(a));
    return vreinterpretq_m128_f32(vcombine_f32(tmp, vdup_n_f32(0)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    float a0 = (float) ((double *) &a)[0];
    float a1 = (float) ((double *) &a)[1];
    return _mm_set_ps(0, 0, a1, a0);
#endif
}

// 将 a 中的打包的有符号32位整数转换为双精度（64位）浮点数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtpi32_pd
FORCE_INLINE __m128d _mm_cvtpi32_pd(__m64 a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpretq_m128d_f64(
        vcvtq_f64_s64(vmovl_s32(vreinterpret_s32_m64(a))));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    double a0 = (double) vget_lane_s32(vreinterpret_s32_m64(a), 0);
    double a1 = (double) vget_lane_s32(vreinterpret_s32_m64(a), 1);
    return _mm_set_pd(a1, a0);
#endif
}

// 将 a 中的单精度（32位）浮点数转换为打包的32位整数，并将结果存储在 dst 中。
// 定义一个函数，将__m128类型的浮点数向量转换为__m128i类型的整数向量
FORCE_INLINE __m128i _mm_cvtps_epi32(__m128 a)
{
    // 如果是 ARM 平台并且支持指令集特性__ARM_FEATURE_FRINT，则使用指令进行转换
    #if defined(__ARM_FEATURE_FRINT)
        return vreinterpretq_m128i_s32(vcvtq_s32_f32(vrnd32xq_f32(a)));
    // 如果是 ARM64 平台或者支持指令集特性__ARM_FEATURE_DIRECTED_ROUNDING，则根据舍入模式进行转换
    #elif (defined(__aarch64__) || defined(_M_ARM64)) || defined(__ARM_FEATURE_DIRECTED_ROUNDING)
        switch (_MM_GET_ROUNDING_MODE()) {
            case _MM_ROUND_NEAREST:
                return vreinterpretq_m128i_s32(vcvtnq_s32_f32(a));
            case _MM_ROUND_DOWN:
                return vreinterpretq_m128i_s32(vcvtmq_s32_f32(a));
            case _MM_ROUND_UP:
                return vreinterpretq_m128i_s32(vcvtpq_s32_f32(a));
            default:  // _MM_ROUND_TOWARD_ZERO
                return vreinterpretq_m128i_s32(vcvtq_s32_f32(a));
        }
    // 如果不是 ARM 平台，则通过指针转换进行转换
    #else
        float *f = (float *) &a;
        switch (_MM_GET_ROUNDING_MODE()) {
    # 对输入的浮点数向最近的整数进行舍入
    case _MM_ROUND_NEAREST: {
        # 创建一个32位整数向量，每个元素都是0x80000000
        uint32x4_t signmask = vdupq_n_u32(0x80000000);
        # 创建一个32位浮点数向量，每个元素都是+/- 0.5
        float32x4_t half = vbslq_f32(signmask, vreinterpretq_f32_m128(a),
                                     vdupq_n_f32(0.5f)); /* +/- 0.5 */
        # 将浮点数向量a加上0.5后转换为整数向量
        int32x4_t r_normal = vcvtq_s32_f32(vaddq_f32(
            vreinterpretq_f32_m128(a), half)); /* round to integer: [a + 0.5]*/
        # 将浮点数向量a直接转换为整数向量
        int32x4_t r_trunc = vcvtq_s32_f32(
            vreinterpretq_f32_m128(a)); /* truncate to integer: [a] */
        # 创建一个全为1的整数向量
        int32x4_t plusone = vreinterpretq_s32_u32(vshrq_n_u32(
            vreinterpretq_u32_s32(vnegq_s32(r_trunc)), 31)); /* 1 or 0 */
        # 计算偶数向下舍入的整数向量
        int32x4_t r_even = vbicq_s32(vaddq_s32(r_trunc, plusone),
                                     vdupq_n_s32(1)); /* ([a] + {0,1}) & ~1 */
        # 计算浮点数向量a与其整数部分的差值
        float32x4_t delta = vsubq_f32(
            vreinterpretq_f32_m128(a),
            vcvtq_f32_s32(r_trunc)); /* compute delta: delta = (a - [a]) */
        # 判断差值是否为+/- 0.5
        uint32x4_t is_delta_half =
            vceqq_f32(delta, half); /* delta == +/- 0.5 */
        # 根据差值是否为+/- 0.5来选择舍入后的整数向量
        return vreinterpretq_m128i_s32(
            vbslq_s32(is_delta_half, r_even, r_normal));
    }
    # 向下舍入为最接近的整数
    case _MM_ROUND_DOWN:
        return _mm_set_epi32(floorf(f[3]), floorf(f[2]), floorf(f[1]),
                             floorf(f[0]));
    # 向上舍入为最接近的整数
    case _MM_ROUND_UP:
        return _mm_set_epi32(ceilf(f[3]), ceilf(f[2]), ceilf(f[1]),
                             ceilf(f[0]));
    # 向零舍入为最接近的整数
    default:  // _MM_ROUND_TOWARD_ZERO
        return _mm_set_epi32((int32_t) f[3], (int32_t) f[2], (int32_t) f[1],
                             (int32_t) f[0]);
    }
#endif
}

// 将 packed single-precision (32-bit) floating-point elements 转换为 packed double-precision (64-bit) floating-point elements，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtps_pd
FORCE_INLINE __m128d _mm_cvtps_pd(__m128 a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(
        vcvt_f64_f32(vget_low_f32(vreinterpretq_f32_m128(a))));
#else
    double a0 = (double) vgetq_lane_f32(vreinterpretq_f32_m128(a), 0);
    double a1 = (double) vgetq_lane_f32(vreinterpretq_f32_m128(a), 1);
    return _mm_set_pd(a1, a0);
#endif
}

// 将 a 的低位 double-precision (64-bit) floating-point 元素复制到 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsd_f64
FORCE_INLINE double _mm_cvtsd_f64(__m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return (double) vgetq_lane_f64(vreinterpretq_f64_m128d(a), 0);
#else
    return ((double *) &a)[0];
#endif
}

// 将 a 的低位 double-precision (64-bit) floating-point 元素转换为 32 位整数，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsd_si32
FORCE_INLINE int32_t _mm_cvtsd_si32(__m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return (int32_t) vgetq_lane_f64(vrndiq_f64(vreinterpretq_f64_m128d(a)), 0);
#else
    __m128d rnd = _mm_round_pd(a, _MM_FROUND_CUR_DIRECTION);
    double ret = ((double *) &rnd)[0];
    return (int32_t) ret;
#endif
}

// 将 a 的低位 double-precision (64-bit) floating-point 元素转换为 64 位整数，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsd_si64
FORCE_INLINE int64_t _mm_cvtsd_si64(__m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return (int64_t) vgetq_lane_f64(vrndiq_f64(vreinterpretq_f64_m128d(a)), 0);
#else
    # 使用SSE指令对双精度浮点数a进行当前舍入方向的舍入
    __m128d rnd = _mm_round_pd(a, _MM_FROUND_CUR_DIRECTION);
    # 将舍入后的结果转换为双精度浮点数类型
    double ret = ((double *) &rnd)[0];
    # 将双精度浮点数类型的结果转换为64位整数类型并返回
    return (int64_t) ret;
#endif
}

// 将a中的较低双精度（64位）浮点元素转换为64位整数，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsd_si64x
#define _mm_cvtsd_si64x _mm_cvtsd_si64

// 将b中的较低双精度（64位）浮点元素转换为单精度（32位）浮点元素，将结果存储在dst的较低元素中，并将a中的上3个打包元素复制到dst的上元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsd_ss
FORCE_INLINE __m128 _mm_cvtsd_ss(__m128 a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128_f32(vsetq_lane_f32(
        vget_lane_f32(vcvt_f32_f64(vreinterpretq_f64_m128d(b)), 0),
        vreinterpretq_f32_m128(a), 0));
#else
    return vreinterpretq_m128_f32(vsetq_lane_f32((float) ((double *) &b)[0],
                                                 vreinterpretq_f32_m128(a), 0));
#endif
}

// 将a中的较低32位整数复制到dst。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi128_si32
FORCE_INLINE int _mm_cvtsi128_si32(__m128i a)
{
    return vgetq_lane_s32(vreinterpretq_s32_m128i(a), 0);
}

// 将a中的较低64位整数复制到dst。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi128_si64
FORCE_INLINE int64_t _mm_cvtsi128_si64(__m128i a)
{
    return vgetq_lane_s64(vreinterpretq_s64_m128i(a), 0);
}

// 将a中的较低64位整数复制到dst。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi128_si64x
#define _mm_cvtsi128_si64x(a) _mm_cvtsi128_si64(a)

// 将有符号32位整数b转换为双精度（64位）浮点元素，将结果存储在dst的较低元素中，并从a中复制上元素到dst的上元素。
// 将 32 位整数 b 转换为双精度浮点数，并将结果存储在 a 的低位元素中
// 如果是 ARM64 架构，则使用特定的指令进行转换
// 否则，将 b 转换为双精度浮点数，然后存储在 a 的低位元素中
FORCE_INLINE __m128d _mm_cvtsi32_sd(__m128d a, int32_t b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(
        vsetq_lane_f64((double) b, vreinterpretq_f64_m128d(a), 0));
#else
    double bf = (double) b;
    return vreinterpretq_m128d_s64(
        vsetq_lane_s64(*(int64_t *) &bf, vreinterpretq_s64_m128d(a), 0));
#endif
}

// 将 a 的低 64 位整数复制到 dst
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi128_si64x
#define _mm_cvtsi128_si64x(a) _mm_cvtsi128_si64(a)

// 将 32 位整数 a 复制到 dst 的低位元素，并将 dst 的高位元素置零
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi32_si128
FORCE_INLINE __m128i _mm_cvtsi32_si128(int a)
{
    return vreinterpretq_m128i_s32(vsetq_lane_s32(a, vdupq_n_s32(0), 0));
}

// 将有符号 64 位整数 b 转换为双精度浮点数，并将结果存储在 a 的低位元素中，同时将 a 的高位元素复制到 dst 的高位元素
// 如果是 ARM64 架构，则使用特定的指令进行转换
// 否则，将 b 转换为双精度浮点数，然后存储在 a 的低位元素中
FORCE_INLINE __m128d _mm_cvtsi64_sd(__m128d a, int64_t b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(
        vsetq_lane_f64((double) b, vreinterpretq_f64_m128d(a), 0));
#else
    double bf = (double) b;
    return vreinterpretq_m128d_s64(
        vsetq_lane_s64(*(int64_t *) &bf, vreinterpretq_s64_m128d(a), 0));
#endif
}

// 将 64 位整数 a 复制到 dst 的低位元素，并将 dst 的高位元素置零
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi64_si128
FORCE_INLINE __m128i _mm_cvtsi64_si128(int64_t a)
{
    return vreinterpretq_m128i_s64(vsetq_lane_s64(a, vdupq_n_s64(0), 0));
}
// 将64位整数a复制到dst的低位元素，并将上位元素置零
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi64x_si128
#define _mm_cvtsi64x_si128(a) _mm_cvtsi64_si128(a)

// 将有符号64位整数b转换为双精度（64位）浮点元素，将结果存储在dst的低位元素，并将上位元素从a复制到dst的上位元素
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtsi64x_sd
#define _mm_cvtsi64x_sd(a, b) _mm_cvtsi64_sd(a, b)

// 将b中的低位单精度（32位）浮点元素转换为双精度（64位）浮点元素，将结果存储在dst的低位元素，并将a中的上位元素复制到dst的上位元素
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtss_sd
FORCE_INLINE __m128d _mm_cvtss_sd(__m128d a, __m128 b)
{
    double d = (double) vgetq_lane_f32(vreinterpretq_f32_m128(b), 0);
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(
        vsetq_lane_f64(d, vreinterpretq_f64_m128d(a), 0));
#else
    return vreinterpretq_m128d_s64(
        vsetq_lane_s64(*(int64_t *) &d, vreinterpretq_s64_m128d(a), 0));
#endif
}

// 将a中的打包双精度（64位）浮点元素转换为带截断的打包32位整数，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvttpd_epi32
FORCE_INLINE __m128i _mm_cvttpd_epi32(__m128d a)
{
    double a0 = ((double *) &a)[0];
    double a1 = ((double *) &a)[1];
    return _mm_set_epi32(0, 0, (int32_t) a1, (int32_t) a0);
}

// 将a中的打包双精度（64位）浮点元素转换为带截断的打包32位整数，并将结果存储在dst中
// 将双精度（64位）浮点型元素在a中除以b中的打包元素，并将结果存储在dst中。
// 定义一个函数，实现两个 __m128d 类型的参数相除的功能
FORCE_INLINE __m128d _mm_div_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 ARM64 指令集实现两个参数相除，并返回结果
    return vreinterpretq_m128d_f64(
        vdivq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    // 如果不是 ARM64 架构，则将参数 a 和 b 转换为 double 指针，分别取出对应的值进行相除
    double *da = (double *) &a;
    double *db = (double *) &b;
    double c[2];
    c[0] = da[0] / db[0];
    c[1] = da[1] / db[1];
    // 将结果存储在 c 数组中，并返回作为 __m128d 类型
    return vld1q_f32((float32_t *) c);
#endif
}

// 定义一个函数，实现两个 __m128d 类型的参数相除的功能，只影响低位的值
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_div_sd
FORCE_INLINE __m128d _mm_div_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 ARM64 指令集实现两个参数相除，并返回结果
    float64x2_t tmp =
        vdivq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b));
    return vreinterpretq_m128d_f64(
        vsetq_lane_f64(vgetq_lane_f64(vreinterpretq_f64_m128d(a), 1), tmp, 1));
#else
    // 如果不是 ARM64 架构，则调用 _mm_div_pd 函数，将结果作为低位值，高位值不变
    return _mm_move_sd(a, _mm_div_pd(a, b));
#endif
}

// 从参数 a 中提取一个 16 位整数，根据 imm8 的值选择位置，并将结果存储在 dst 的低位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_extract_epi16
// FORCE_INLINE int _mm_extract_epi16(__m128i a, __constrange(0,8) int imm)
#define _mm_extract_epi16(a, imm) \
    vgetq_lane_u16(vreinterpretq_u16_m128i(a), (imm))

// 将参数 a 复制到 dst，并在指定的位置 imm8 处插入 16 位整数 i
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_insert_epi16
// FORCE_INLINE __m128i _mm_insert_epi16(__m128i a, int b,
//                                       __constrange(0,8) int imm)
#define _mm_insert_epi16(a, b, imm) \
    # 使用a的值重新解释为128位整数向量，并将b的值设置为其中的一个16位整数，并返回结果
    vreinterpretq_m128i_s16(        \
        vsetq_lane_s16((b), vreinterpretq_s16_m128i(a), (imm)))
// 从内存中加载128位数据（由2个打包的双精度（64位）浮点元素组成）到目标寄存器dst中。mem_addr必须对齐到16字节边界，否则可能会生成通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_load_pd
FORCE_INLINE __m128d _mm_load_pd(const double *p)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(vld1q_f64(p));
#else
    const float *fp = (const float *) p;
    float ALIGN_STRUCT(16) data[4] = {fp[0], fp[1], fp[2], fp[3]};
    return vreinterpretq_m128d_f32(vld1q_f32(data));
#endif
}

// 从内存中加载一个双精度（64位）浮点元素到目标寄存器的两个元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_load_pd1
#define _mm_load_pd1 _mm_load1_pd

// 从内存中加载一个双精度（64位）浮点元素到目标寄存器的低位，并将上位元素置零。mem_addr不需要对齐到特定边界。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_load_sd
FORCE_INLINE __m128d _mm_load_sd(const double *p)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(vsetq_lane_f64(*p, vdupq_n_f64(0), 0));
#else
    const float *fp = (const float *) p;
    float ALIGN_STRUCT(16) data[4] = {fp[0], fp[1], 0, 0};
    return vreinterpretq_m128d_f32(vld1q_f32(data));
#endif
}

// 从内存中加载128位整数数据到目标寄存器dst中。mem_addr必须对齐到16字节边界，否则可能会生成通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_load_si128
FORCE_INLINE __m128i _mm_load_si128(const __m128i *p)
{
    return vreinterpretq_m128i_s32(vld1q_s32((const int32_t *) p));
}

// 从内存中加载一个双精度（64位）浮点元素到目标寄存器的两个元素中。
// 使用内联函数定义加载双精度浮点数的操作，将内存中的数据加载到寄存器中
FORCE_INLINE __m128d _mm_load1_pd(const double *p)
{
    // 如果是 ARM64 架构，则使用指令加载双精度浮点数
    return vreinterpretq_m128d_f64(vld1q_dup_f64(p));
    // 如果不是 ARM64 架构，则使用指令加载64位整数
    return vreinterpretq_m128d_s64(vdupq_n_s64(*(const int64_t *) p));
}

// 从内存中加载双精度浮点数元素到 dst 的高位元素，并将低位元素从 a 复制到 dst
// mem_addr 不需要对齐到任何特定边界
FORCE_INLINE __m128d _mm_loadh_pd(__m128d a, const double *p)
{
    // 如果是 ARM64 架构，则使用指令加载双精度浮点数
    return vreinterpretq_m128d_f64(
        vcombine_f64(vget_low_f64(vreinterpretq_f64_m128d(a)), vld1_f64(p)));
    // 如果不是 ARM64 架构，则使用指令加载单精度浮点数
    return vreinterpretq_m128d_f32(vcombine_f32(
        vget_low_f32(vreinterpretq_f32_m128d(a)), vld1_f32((const float *) p)));
}

// 从内存中加载64位整数到 dst 的第一个元素
FORCE_INLINE __m128i _mm_loadl_epi64(__m128i const *p)
{
    /* 将指针 p 指向的值的低64位加载到结果的低64位，将结果的高64位清零 */
    return vreinterpretq_m128i_s32(
        vcombine_s32(vld1_s32((int32_t const *) p), vcreate_s32(0)));
}

// 从内存中加载双精度浮点数元素到 dst 的低位元素，并将高位元素从 a 复制到 dst
// mem_addr 不需要对齐到任何特定边界
FORCE_INLINE __m128d _mm_loadl_pd(__m128d a, const double *p)
{
    // 如果是 ARM64 架构，则使用指令加载双精度浮点数
    # 将两个双精度浮点数向量合并成一个双精度浮点数向量
    return vreinterpretq_m128d_f64(
        # 将指针p指向的双精度浮点数加载到向量中
        vcombine_f64(vld1_f64(p), 
        # 将128位寄存器a的高64位转换成双精度浮点数向量
        vget_high_f64(vreinterpretq_f64_m128d(a))));
// 如果不是 ARM64 架构，则执行以下代码
#else
    // 从内存中加载两个双精度（64位）浮点数，以相反的顺序存储到目标寄存器中
    // mem_addr 必须对齐到 16 字节边界，否则可能会生成通用保护异常
    // https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_loadr_pd
FORCE_INLINE __m128d _mm_loadr_pd(const double *p)
{
    // 如果是 ARM64 架构，则执行以下代码
#if defined(__aarch64__) || defined(_M_ARM64)
    float64x2_t v = vld1q_f64(p);
    return vreinterpretq_m128d_f64(vextq_f64(v, v, 1));
    // 如果不是 ARM64 架构，则执行以下代码
#else
    int64x2_t v = vld1q_s64((const int64_t *) p);
    return vreinterpretq_m128d_s64(vextq_s64(v, v, 1));
#endif
}

// 从未对齐的内存中加载两个双精度浮点值
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_loadu_pd
FORCE_INLINE __m128d _mm_loadu_pd(const double *p)
{
    return _mm_load_pd(p);
}

// 从内存中加载128位整数数据到目标寄存器中，mem_addr 不需要对齐到特定边界
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_loadu_si128
FORCE_INLINE __m128i _mm_loadu_si128(const __m128i *p)
{
    return vreinterpretq_m128i_s32(vld1q_s32((const int32_t *) p));
}

// 从内存中加载未对齐的32位整数到目标寄存器的第一个元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_loadu_si32
FORCE_INLINE __m128i _mm_loadu_si32(const void *p)
{
    return vreinterpretq_m128i_s32(
        vsetq_lane_s32(*(const int32_t *) p, vdupq_n_s32(0), 0));
}

// 将寄存器 a 和 b 中的打包的有符号16位整数相乘，得到中间的有符号32位整数
// 水平相加相邻的中间32位整数，并将结果打包到目标寄存器中
// 使用 SIMD 指令计算两个 128 位整数向量 a 和 b 的每个元素的乘积，然后将结果相加
FORCE_INLINE __m128i _mm_madd_epi16(__m128i a, __m128i b)
{
    // 计算 a 和 b 的低 64 位元素的乘积
    int32x4_t low = vmull_s16(vget_low_s16(vreinterpretq_s16_m128i(a)),
                              vget_low_s16(vreinterpretq_s16_m128i(b)));
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则计算 a 和 b 的高 64 位元素的乘积
    int32x4_t high =
        vmull_high_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b));

    // 返回低位和高位元素相加的结果
    return vreinterpretq_m128i_s32(vpaddq_s32(low, high));
#else
    // 如果不是 ARM64 架构，则计算 a 和 b 的高 64 位元素的乘积
    int32x4_t high = vmull_s16(vget_high_s16(vreinterpretq_s16_m128i(a)),
                               vget_high_s16(vreinterpretq_s16_m128i(b)));

    // 分别计算低位和高位元素的和
    int32x2_t low_sum = vpadd_s32(vget_low_s32(low), vget_high_s32(low));
    int32x2_t high_sum = vpadd_s32(vget_low_s32(high), vget_high_s32(high));

    // 将低位和高位的和组合成一个 128 位整数向量返回
    return vreinterpretq_m128i_s32(vcombine_s32(low_sum, high_sum));
#endif
}

// 使用掩码条件将 128 位整数向量 a 中的元素存储到内存中，根据掩码的设置决定是否存储
FORCE_INLINE void _mm_maskmoveu_si128(__m128i a, __m128i mask, char *mem_addr)
{
    // 将掩码右移 7 位，得到掩码的最高位
    int8x16_t shr_mask = vshrq_n_s8(vreinterpretq_s8_m128i(mask), 7);
    // 从内存地址 mem_addr 加载一个 128 位浮点数向量
    __m128 b = _mm_load_ps((const float *) mem_addr);
    // 根据掩码的设置，选择 a 或 b 中的元素组成新的 128 位整数向量
    int8x16_t masked =
        vbslq_s8(vreinterpretq_u8_s8(shr_mask), vreinterpretq_s8_m128i(a),
                 vreinterpretq_s8_m128(b));
    // 将结果存储到内存地址 mem_addr
    vst1q_s8((int8_t *) mem_addr, masked);
}

// 比较两个 128 位整数向量 a 和 b 中对应元素的大小，将较大的元素存储到结果向量中
FORCE_INLINE __m128i _mm_max_epi16(__m128i a, __m128i b)
{
    # 将输入的两个128位整数向量a和b进行类型转换，转换成对应的128位浮点数向量
    return vreinterpretq_m128i_s16(
        # 对两个128位整数向量a和b进行类型转换，转换成对应的128位有符号整数向量
        vmaxq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
// 结束函数定义
}

// 比较两个 packed unsigned 8-bit integers 的最大值，并将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_epu8
FORCE_INLINE __m128i _mm_max_epu8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u8(
        vmaxq_u8(vreinterpretq_u8_m128i(a), vreinterpretq_u8_m128i(b)));
}

// 比较两个 packed double-precision (64-bit) floating-point elements 的最大值，并将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_pd
FORCE_INLINE __m128d _mm_max_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
#if SSE2NEON_PRECISE_MINMAX
    float64x2_t _a = vreinterpretq_f64_m128d(a);
    float64x2_t _b = vreinterpretq_f64_m128d(b);
    return vreinterpretq_m128d_f64(vbslq_f64(vcgtq_f64(_a, _b), _a, _b));
#else
    return vreinterpretq_m128d_f64(
        vmaxq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#endif
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    d[0] = (*(double *) &a0) > (*(double *) &b0) ? a0 : b0;
    d[1] = (*(double *) &a1) > (*(double *) &b1) ? a1 : b1;

    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较两个 packed double-precision (64-bit) floating-point elements 的最大值，并将结果存储在 dst 的低位元素中，将 a 的高位元素复制到 dst 的高位元素中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_sd
FORCE_INLINE __m128d _mm_max_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return _mm_move_sd(a, _mm_max_pd(a, b));
#else
    double *da = (double *) &a;
    // 将变量b的地址强制转换为double指针类型，然后赋值给指针变量db
    double *db = (double *) &b;
    // 创建一个包含两个元素的double数组c，第一个元素取da[0]和db[0]中较大的值，第二个元素取da[1]的值
    double c[2] = {da[0] > db[0] ? da[0] : db[0], da[1]};
    // 将c数组中的值转换为float32_t类型的值，然后加载到128位寄存器中，并将其转换为两个双精度浮点数
    return vreinterpretq_m128d_f32(vld1q_f32((float32_t *) c));
#endif
}

// 比较两个包含有符号16位整数的寄存器a和b，并将最小值存储在目标寄存器中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_min_epi16
FORCE_INLINE __m128i _mm_min_epi16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s16(
        vminq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
}

// 比较两个包含无符号8位整数的寄存器a和b，并将最小值存储在目标寄存器中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_min_epu8
FORCE_INLINE __m128i _mm_min_epu8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u8(
        vminq_u8(vreinterpretq_u8_m128i(a), vreinterpretq_u8_m128i(b)));
}

// 比较两个包含双精度（64位）浮点数的寄存器a和b，并将最小值存储在目标寄存器中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_min_pd
FORCE_INLINE __m128d _mm_min_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
#if SSE2NEON_PRECISE_MINMAX
    float64x2_t _a = vreinterpretq_f64_m128d(a);
    float64x2_t _b = vreinterpretq_f64_m128d(b);
    return vreinterpretq_m128d_f64(vbslq_f64(vcltq_f64(_a, _b), _a, _b));
#else
    return vreinterpretq_m128d_f64(
        vminq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#endif
#else
    uint64_t a0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(a));
    uint64_t a1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(a));
    uint64_t b0 = (uint64_t) vget_low_u64(vreinterpretq_u64_m128d(b));
    uint64_t b1 = (uint64_t) vget_high_u64(vreinterpretq_u64_m128d(b));
    uint64_t d[2];
    d[0] = (*(double *) &a0) < (*(double *) &b0) ? a0 : b0;
    d[1] = (*(double *) &a1) < (*(double *) &b1) ? a1 : b1;
    return vreinterpretq_m128d_u64(vld1q_u64(d));
#endif
}

// 比较寄存器a和b中的低位双精度（64位）浮点数，并将最小值存储在目标寄存器中
// 定义一个函数，实现将两个 __m128d 类型的参数中的较小值存储到目标的低位元素中，并将较大值的高位元素复制到目标的高位元素中
// 链接到 Intel 官方文档，介绍 _mm_min_sd 函数的作用和用法
FORCE_INLINE __m128d _mm_min_sd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则调用 _mm_move_sd 函数，将 a 和 b 中的较小值存储到目标的低位元素中
    return _mm_move_sd(a, _mm_min_pd(a, b));
#else
    // 如果不是 ARM64 架构，则进行以下操作
    double *da = (double *) &a;
    double *db = (double *) &b;
    double c[2] = {da[0] < db[0] ? da[0] : db[0], da[1]};
    // 将 c 数组中的值转换为 __m128d 类型，并返回
    return vreinterpretq_m128d_f32(vld1q_f32((float32_t *) c));
#endif
}

// 将参数 a 中的低位 64 位整数复制到目标的低位元素中，并将高位元素置零
// 链接到 Intel 官方文档，介绍 _mm_move_epi64 函数的作用和用法
FORCE_INLINE __m128i _mm_move_epi64(__m128i a)
{
    // 将参数 a 转换为 s64 类型，然后将第二个元素置零，并返回结果
    return vreinterpretq_m128i_s64(
        vsetq_lane_s64(0, vreinterpretq_s64_m128i(a), 1));
}

// 将参数 b 中的低位双精度（64 位）浮点数复制到目标的低位元素中，并将参数 a 中的高位元素复制到目标的高位元素中
// 链接到 Intel 官方文档，介绍 _mm_move_sd 函数的作用和用法
FORCE_INLINE __m128d _mm_move_sd(__m128d a, __m128d b)
{
    // 将参数 a 和 b 转换为 f32 类型，然后将它们的低位和高位元素组合在一起，并返回结果
    return vreinterpretq_m128d_f32(
        vcombine_f32(vget_low_f32(vreinterpretq_f32_m128d(b)),
                     vget_high_f32(vreinterpretq_f32_m128d(a))));
}

// 从参数 a 中每个 8 位元素的最高位创建掩码，并将结果存储在目标中
// 链接到 Intel 官方文档，介绍 _mm_movemask_epi8 函数的作用和用法
FORCE_INLINE int _mm_movemask_epi8(__m128i a)
{
    // 使用逐渐增加的宽度移位+加法来收集符号位
    // 由于在小端序中进行扩展移位会相当混乱，因此所有操作都将以大端序顺序进行说明。这会产生不同的结果-在大端序机器上，位实际上会被颠倒。
    // 将输入向量转换为8位无符号整数向量
    uint8x16_t input = vreinterpretq_u8_m128i(a);

    // 通过无符号右移操作，将除符号位外的所有位都移出
    // 高位字节的向量：
    // 89 ff 1d c0 00 10 99 33
    // \  \  \  \  \  \  \  \    high_bits = (uint16x4_t)(input >> 7)
    //  |  |  |  |  |  |  |  |
    // 01 01 00 01 00 00 01 00
    //
    // 第一个重要通道的位：
    // 10001001 (89)
    // \______
    //        |
    // 00000001 (01)
    uint16x8_t high_bits = vreinterpretq_u16_u8(vshrq_n_u8(input, 7));

    // 通过16位无符号右移和加法，将偶数通道合并在一起
    // 'xx'代表最终结果中将被忽略的垃圾数据
    // 在重要字节中，加法的作用类似于二进制的OR
    //
    // 01 01 00 01 00 00 01 00
    //  \_ |  \_ |  \_ |  \_ |   paired16 = (uint32x4_t)(input + (input >> 7))
    //    \|    \|    \|    \|
    // xx 03 xx 01 xx 00 xx 02
    //
    // 00000001 00000001 (01 01)
    //        \_______ |
    //                \|
    // xxxxxxxx xxxxxx11 (xx 03)
    uint32x4_t paired16 =
        vreinterpretq_u32_u16(vsraq_n_u16(high_bits, high_bits, 7));

    // 通过更宽的32位右移和加法重复上述过程
    // xx 03 xx 01 xx 00 xx 02
    //     \____ |     \____ |  paired32 = (uint64x1_t)(paired16 + (paired16 >>
    //     14))
    //          \|          \|
    // xx xx xx 0d xx xx xx 02
    //
    // 00000011 00000001 (03 01)
    //        \\_____ ||
    //         '----.\||
    // xxxxxxxx xxxx1101 (xx 0d)
    uint64x2_t paired32 =
        vreinterpretq_u64_u32(vsraq_n_u32(paired16, paired16, 14));

    // 最后，通过更宽的64位右移和加法，得到我们在低8位通道中的结果
    // xx xx xx 0d xx xx xx 02
    //            \_________ |   paired64 = (uint8x8_t)(paired32 + (paired32 >>
    //            28))
    //                      \|
    // xx xx xx xx xx xx xx d2
    //
    // 将两个 32 位整数向右移动 28 位，然后将结果重新解释为 16 个 8 位整数
    uint8x16_t paired64 =
        vreinterpretq_u8_u64(vsraq_n_u64(paired32, paired32, 28));
    
    // 从每个 64 位 lane 中提取低 8 位，使用两个 8 位提取操作
    // 返回 paired64 的第 0 个元素，以及将 paired64 的第 8 个元素左移 8 位后的结果
    // 注意：对于小端序的系统，将返回正确的数值 4b (01001011)
    return vgetq_lane_u8(paired64, 0) | ((int) vgetq_lane_u8(paired64, 8) << 8);
// 设置掩码 dst 的每个位，基于 a 中对应的打包的双精度（64位）浮点元素的最高有效位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movemask_pd
FORCE_INLINE int _mm_movemask_pd(__m128d a)
{
    // 将 a 转换为 uint64x2_t 类型
    uint64x2_t input = vreinterpretq_u64_m128d(a);
    // 将 input 向右移动 63 位，得到高位的值
    uint64x2_t high_bits = vshrq_n_u64(input, 63);
    // 将高位的值转换为 int 类型，并将两个值按位或运算
    return (int) (vgetq_lane_u64(high_bits, 0) |
                  (vgetq_lane_u64(high_bits, 1) << 1));
}

// 将 a 的低位 64 位整数复制到 dst
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movepi64_pi64
FORCE_INLINE __m64 _mm_movepi64_pi64(__m128i a)
{
    // 将 a 转换为 int64x2_t 类型，再取得低位的值
    return vreinterpret_m64_s64(vget_low_s64(vreinterpretq_s64_m128i(a)));
}

// 将 64 位整数 a 复制到 dst 的低位，并将上位元素置零
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movpi64_epi64
FORCE_INLINE __m128i _mm_movpi64_epi64(__m64 a)
{
    // 将 a 转换为 int64x1_t 类型，再将其与 0 组合成 int64x2_t 类型
    return vreinterpretq_m128i_s64(
        vcombine_s64(vreinterpret_s64_m64(a), vdup_n_s64(0)));
}

// 将 a 和 b 中每个打包的 64 位元素的低位无符号 32 位整数相乘，并将无符号 64 位结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mul_epu32
FORCE_INLINE __m128i _mm_mul_epu32(__m128i a, __m128i b)
{
    // vmull_u32 扩展而不是掩码，因此我们进行下转换
    uint32x2_t a_lo = vmovn_u64(vreinterpretq_u64_m128i(a));
    uint32x2_t b_lo = vmovn_u64(vreinterpretq_u64_m128i(b));
    // 将 a_lo 和 b_lo 中的值相乘，并将结果转换为 uint64x2_t 类型
    return vreinterpretq_m128i_u64(vmull_u32(a_lo, b_lo));
}

// 将 a 和 b 中的打包的双精度（64位）浮点元素相乘，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mul_pd
FORCE_INLINE __m128d _mm_mul_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    # 将两个128位的浮点数向量a和b进行乘法运算，并将结果转换为128位的浮点数向量
    return vreinterpretq_m128d_f64(
        vmulq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    // 将 a 和 b 的地址转换为 double 类型的指针
    double *da = (double *) &a;
    double *db = (double *) &b;
    // 创建一个长度为 2 的 double 数组 c
    double c[2];
    // 计算 da[0] 和 db[0] 的乘积，并将结果存储在 c[0] 中
    c[0] = da[0] * db[0];
    // 计算 da[1] 和 db[1] 的乘积，并将结果存储在 c[1] 中
    c[1] = da[1] * db[1];
    // 将 c 转换为 float32_t 类型的指针，并返回结果
    return vld1q_f32((float32_t *) c);
#endif
}

// 将 a 的低位双精度（64位）浮点数乘以 b，将结果存储在 dst 的低位，并将 a 的高位复制到 dst 的高位。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=mm_mul_sd
FORCE_INLINE __m128d _mm_mul_sd(__m128d a, __m128d b)
{
    // 调用 _mm_mul_pd 函数计算 a 和 b 的乘积，并将结果传递给 _mm_move_sd 函数
    return _mm_move_sd(a, _mm_mul_pd(a, b));
}

// 将 a 和 b 的低位无符号 32 位整数相乘，并将结果存储在 dst 中的无符号 64 位整数中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mul_su32
FORCE_INLINE __m64 _mm_mul_su32(__m64 a, __m64 b)
{
    // 将 a 和 b 转换为 uint32x2_t 类型，并使用 vmull_u32 函数计算乘积
    return vreinterpret_m64_u64(vget_low_u64(
        vmull_u32(vreinterpret_u32_m64(a), vreinterpret_u32_m64(b))));
}

// 将 a 和 b 中的打包有符号 16 位整数相乘，生成中间的 32 位整数，并将中间整数的高 16 位存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mulhi_epi16
FORCE_INLINE __m128i _mm_mulhi_epi16(__m128i a, __m128i b)
{
    /* FIXME: issue with large values because of result saturation */
    // 使用 vqdmulhq_s16 函数计算 a 和 b 的乘积，并将结果除以 2，然后将结果转换为 int16x8_t 类型
    // 返回结果的高 16 位
    // int16x8_t ret = vqdmulhq_s16(vreinterpretq_s16_m128i(a),
    // vreinterpretq_s16_m128i(b)); /* =2*a*b */ return
    // vreinterpretq_m128i_s16(vshrq_n_s16(ret, 1));
    // 将 a 和 b 的低 16 位转换为 int16x4_t 类型
    int16x4_t a3210 = vget_low_s16(vreinterpretq_s16_m128i(a));
    int16x4_t b3210 = vget_low_s16(vreinterpretq_s16_m128i(b));
    // 使用 vmull_s16 函数计算 a3210 和 b3210 的乘积，并将结果存储在 ab3210 中
    int32x4_t ab3210 = vmull_s16(a3210, b3210); /* 3333222211110000 */
    // 将 a 和 b 的高 16 位转换为 int16x4_t 类型
    int16x4_t a7654 = vget_high_s16(vreinterpretq_s16_m128i(a));
    int16x4_t b7654 = vget_high_s16(vreinterpretq_s16_m128i(b));
    // 使用 vmull_s16 函数计算 a7654 和 b7654 的乘积，并将结果存储在 ab7654 中
    int32x4_t ab7654 = vmull_s16(a7654, b7654); /* 7777666655554444 */
    # 将输入的两个 128 位寄存器中的数据按照 16 位无符号整数进行交错解压缩
    uint16x8x2_t r = vuzpq_u16(vreinterpretq_u16_s32(ab3210), vreinterpretq_u16_s32(ab7654));
    # 返回结果中的第二个 128 位寄存器中的数据，转换成 128 位整数类型
    return vreinterpretq_m128i_u16(r.val[1]);
// 以无符号 16 位整数形式，将 a 和 b 中的数据进行乘法运算，得到中间的 32 位整数，并将中间整数的高 16 位存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mulhi_epu16
FORCE_INLINE __m128i _mm_mulhi_epu16(__m128i a, __m128i b)
{
    // 从 a 中获取低 16 位无符号整数
    uint16x4_t a3210 = vget_low_u16(vreinterpretq_u16_m128i(a));
    // 从 b 中获取低 16 位无符号整数
    uint16x4_t b3210 = vget_low_u16(vreinterpretq_u16_m128i(b));
    // 将 a3210 和 b3210 中的每个元素进行无符号 16 位整数乘法，得到 32 位整数
    uint32x4_t ab3210 = vmull_u16(a3210, b3210);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则继续执行以下代码
    // 从 a 和 b 中获取高 16 位无符号整数
    uint32x4_t ab7654 = vmull_high_u16(vreinterpretq_u16_m128i(a), vreinterpretq_u16_m128i(b));
    // 将 ab3210 和 ab7654 中的元素进行重新排列，得到一个 16 位无符号整数的 8 元素向量
    uint16x8_t r = vuzp2q_u16(vreinterpretq_u16_u32(ab3210), vreinterpretq_u16_u32(ab7654));
    // 将 r 转换为 128 位整数向量并返回
    return vreinterpretq_m128i_u16(r);
#else
    // 如果不是 ARM64 架构，则执行以下代码
    // 从 a 中获取高 16 位无符号整数
    uint16x4_t a7654 = vget_high_u16(vreinterpretq_u16_m128i(a));
    // 从 b 中获取高 16 位无符号整数
    uint16x4_t b7654 = vget_high_u16(vreinterpretq_u16_m128i(b));
    // 将 a7654 和 b7654 中的每个元素进行无符号 16 位整数乘法，得到 32 位整数
    uint32x4_t ab7654 = vmull_u16(a7654, b7654);
    // 将 ab3210 和 ab7654 中的元素进行重新排列，得到一个 16 位无符号整数的 8 元素向量
    uint16x8x2_t r = vuzpq_u16(vreinterpretq_u16_u32(ab3210), vreinterpretq_u16_u32(ab7654));
    // 将 r 的第二个元素向量转换为 128 位整数向量并返回
    return vreinterpretq_m128i_u16(r.val[1]);
#endif
}

// 以 16 位整数形式，将 a 和 b 中的数据进行乘法运算，得到中间的 32 位整数，并将中间整数的低 16 位存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mullo_epi16
FORCE_INLINE __m128i _mm_mullo_epi16(__m128i a, __m128i b)
{
    // 将 a 和 b 中的数据转换为有符号 16 位整数，进行乘法运算，并返回结果
    return vreinterpretq_m128i_s16(vmulq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
}

// 计算 a 和 b 中的双精度（64 位）浮点数的按位或，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=mm_or_pd
FORCE_INLINE __m128d _mm_or_pd(__m128d a, __m128d b)
{
    // 将 a 和 b 中的数据转换为有符号 64 位整数，进行按位或运算，并返回结果
    return vreinterpretq_m128d_s64(vorrq_s64(vreinterpretq_s64_m128d(a), vreinterpretq_s64_m128d(b)));
}
// 计算 a 和 b 中 128 位整数数据的按位或，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_or_si128
FORCE_INLINE __m128i _mm_or_si128(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s32(
        vorrq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 将 a 和 b 中的打包有符号 16 位整数转换为打包的 8 位整数，使用有符号饱和，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_packs_epi16
FORCE_INLINE __m128i _mm_packs_epi16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s8(
        vcombine_s8(vqmovn_s16(vreinterpretq_s16_m128i(a)),
                    vqmovn_s16(vreinterpretq_s16_m128i(b))));
}

// 将 a 和 b 中的打包有符号 32 位整数转换为打包的 16 位整数，使用有符号饱和，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_packs_epi32
FORCE_INLINE __m128i _mm_packs_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s16(
        vcombine_s16(vqmovn_s32(vreinterpretq_s32_m128i(a)),
                     vqmovn_s32(vreinterpretq_s32_m128i(b))));
}

// 将 a 和 b 中的打包有符号 16 位整数转换为打包的 8 位整数，使用无符号饱和，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_packus_epi16
FORCE_INLINE __m128i _mm_packus_epi16(const __m128i a, const __m128i b)
{
    return vreinterpretq_m128i_u8(
        vcombine_u8(vqmovun_s16(vreinterpretq_s16_m128i(a)),
                    vqmovun_s16(vreinterpretq_s16_m128i(b))));
}

// 暂停处理器。这通常用于自旋等待循环，根据 x86 处理器的不同，典型值在 40-100 个周期范围内。
// 'yield' 指令不太适合，因为在大多数情况下它实际上是一个 nop
# Arm cores. Experience with several databases has shown has shown an 'isb' is
# a reasonable approximation.
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_pause
# 定义一个名为 _mm_pause 的函数，用于在 Arm 架构上执行指令
# 在不同的编译器下，使用不同的指令实现
# 在 MSC 编译器下，使用 __isb 函数执行 _ARM64_BARRIER_SY 指令
# 在其它编译器下，使用汇编指令 isb
FORCE_INLINE void _mm_pause(void)
{
#if defined(_MSC_VER)
    __isb(_ARM64_BARRIER_SY);
#else
    __asm__ __volatile__("isb\n");
#endif
}

# Compute the absolute differences of packed unsigned 8-bit integers in a and
# b, then horizontally sum each consecutive 8 differences to produce two
# unsigned 16-bit integers, and pack these unsigned 16-bit integers in the low
# 16 bits of 64-bit elements in dst.
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sad_epu8
# 定义一个名为 _mm_sad_epu8 的函数，用于计算两个 128 位无符号 8 位整数向量的绝对差值
# 然后将每个连续的 8 个差值进行水平求和，得到两个无符号 16 位整数
# 最后将这两个无符号 16 位整数打包到 64 位元素的低 16 位中
FORCE_INLINE __m128i _mm_sad_epu8(__m128i a, __m128i b)
{
    uint16x8_t t = vpaddlq_u8(vabdq_u8((uint8x16_t) a, (uint8x16_t) b));
    return vreinterpretq_m128i_u64(vpaddlq_u32(vpaddlq_u16(t)));
}

# Set packed 16-bit integers in dst with the supplied values.
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_set_epi16
# 定义一个名为 _mm_set_epi16 的函数，用于将给定的值设置为打包的 16 位整数
FORCE_INLINE __m128i _mm_set_epi16(short i7,
                                   short i6,
                                   short i5,
                                   short i4,
                                   short i3,
                                   short i2,
                                   short i1,
                                   short i0)
{
    int16_t ALIGN_STRUCT(16) data[8] = {i0, i1, i2, i3, i4, i5, i6, i7};
    return vreinterpretq_m128i_s16(vld1q_s16(data));
}

# Set packed 32-bit integers in dst with the supplied values.
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_set_epi32
# 定义一个名为 _mm_set_epi32 的函数，用于将给定的值设置为打包的 32 位整数
FORCE_INLINE __m128i _mm_set_epi32(int i3, int i2, int i1, int i0)
{
    int32_t ALIGN_STRUCT(16) data[4] = {i0, i1, i2, i3};
    return vreinterpretq_m128i_s32(vld1q_s32(data));
}

# Set packed 64-bit integers in dst with the supplied values.
# 定义一个名为 _mm_set_epi64 的函数，用于将给定的值设置为打包的 64 位整数
// 使用两个 64 位整数创建一个 128 位整数
FORCE_INLINE __m128i _mm_set_epi64(__m64 i1, __m64 i2)
{
    return _mm_set_epi64x(vget_lane_s64(i1, 0), vget_lane_s64(i2, 0));
}

// 使用两个 64 位整数创建一个 128 位整数
FORCE_INLINE __m128i _mm_set_epi64x(int64_t i1, int64_t i2)
{
    return vreinterpretq_m128i_s64(
        vcombine_s64(vcreate_s64(i2), vcreate_s64(i1)));
}

// 使用提供的值在目标中设置打包的 8 位整数
FORCE_INLINE __m128i _mm_set_epi8(signed char b15,
                                  signed char b14,
                                  signed char b13,
                                  signed char b12,
                                  signed char b11,
                                  signed char b10,
                                  signed char b9,
                                  signed char b8,
                                  signed char b7,
                                  signed char b6,
                                  signed char b5,
                                  signed char b4,
                                  signed char b3,
                                  signed char b2,
                                  signed char b1,
                                  signed char b0)
{
    int8_t ALIGN_STRUCT(16)
        data[16] = {(int8_t) b0,  (int8_t) b1,  (int8_t) b2,  (int8_t) b3,
                    (int8_t) b4,  (int8_t) b5,  (int8_t) b6,  (int8_t) b7,
                    (int8_t) b8,  (int8_t) b9,  (int8_t) b10, (int8_t) b11,
                    (int8_t) b12, (int8_t) b13, (int8_t) b14, (int8_t) b15};
    return (__m128i) vld1q_s8(data);
}

// 使用提供的值在目标中设置打包的双精度（64 位）浮点元素
# 定义一个函数，用于将两个 double 类型的值 e1 和 e0 封装成 __m128d 类型的向量
def _mm_set_pd(double e1, double e0):
    # 创建一个长度为 2 的数组 data，存储 e0 和 e1 的值
    double ALIGN_STRUCT(16) data[2] = {e0, e1};
    # 判断编译环境是否为 ARM 架构，如果是则使用 ARM 指令集进行转换，否则使用 x86 指令集进行转换
    # 将 data 数组转换为 __m128d 类型的向量
    return vreinterpretq_m128d_f64(vld1q_f64((float64_t *) data));

# 定义一个宏，用于将一个 double 类型的值 a 广播到 __m128d 类型的向量的所有元素
#define _mm_set_pd1 _mm_set1_pd

# 定义一个函数，用于将一个 double 类型的值 a 复制到 __m128d 类型的向量的低位元素，并将高位元素置零
def _mm_set_sd(double a):
    # 判断编译环境是否为 ARM 架构，如果是则使用 ARM 指令集进行转换，否则使用 x86 指令集进行转换
    # 将 a 的值复制到 __m128d 类型的向量的低位元素，并将高位元素置零
    return vreinterpretq_m128d_f64(vsetq_lane_f64(a, vdupq_n_f64(0), 0));

# 定义一个函数，用于将一个 short 类型的值 w 广播到 __m128i 类型的向量的所有元素
def _mm_set1_epi16(short w):
    # 将 w 的值广播到 __m128i 类型的向量的所有元素
    return vreinterpretq_m128i_s16(vdupq_n_s16(w));

# 定义一个函数，用于将一个 int 类型的值 _i 广播到 __m128i 类型的向量的所有元素
def _mm_set1_epi32(int _i):
    # 将 _i 的值广播到 __m128i 类型的向量的所有元素
    return vreinterpretq_m128i_s32(vdupq_n_s32(_i));

# 定义一个函数，用于将一个 __m64 类型的值 _i 广播到 __m128i 类型的向量的所有元素
def _mm_set1_epi64(__m64 _i):
    # 将 _i 的值广播到 __m128i 类型的向量的所有元素
    return vreinterpretq_m128i_s64(vdupq_lane_s64(_i, 0));

# 定义一个函数，用于将一个 int64_t 类型的值 _i 广播到 __m128i 类型的向量的所有元素
def _mm_set1_epi64x(int64_t _i):
    # 将_i的值复制到一个64位整数的向量寄存器中，并将其视为128位整数的向量寄存器的内容，然后返回这个向量寄存器的值。
// 广播 8 位整数 a 到 dst 的所有元素
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_set1_epi8
FORCE_INLINE __m128i _mm_set1_epi8(signed char w)
{
    return vreinterpretq_m128i_s8(vdupq_n_s8(w));
}

// 将双精度（64 位）浮点值 a 广播到 dst 的所有元素
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_set1_pd
FORCE_INLINE __m128d _mm_set1_pd(double d)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(vdupq_n_f64(d));
#else
    return vreinterpretq_m128d_s64(vdupq_n_s64(*(int64_t *) &d));
#endif
}

// 使用反向顺序设置 dst 中的打包 16 位整数与提供的值
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setr_epi16
FORCE_INLINE __m128i _mm_setr_epi16(short w0,
                                    short w1,
                                    short w2,
                                    short w3,
                                    short w4,
                                    short w5,
                                    short w6,
                                    short w7)
{
    int16_t ALIGN_STRUCT(16) data[8] = {w0, w1, w2, w3, w4, w5, w6, w7};
    return vreinterpretq_m128i_s16(vld1q_s16((int16_t *) data));
}

// 使用反向顺序设置 dst 中的打包 32 位整数与提供的值
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setr_epi32
FORCE_INLINE __m128i _mm_setr_epi32(int i3, int i2, int i1, int i0)
{
    int32_t ALIGN_STRUCT(16) data[4] = {i3, i2, i1, i0};
    return vreinterpretq_m128i_s32(vld1q_s32(data));
}

// 使用反向顺序设置 dst 中的打包 64 位整数与提供的值
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setr_epi64
FORCE_INLINE __m128i _mm_setr_epi64(__m64 e1, __m64 e0)
{
    # 将两个128位整数向量重新解释为两个64位整数向量，并返回结果
    return vreinterpretq_m128i_s64(vcombine_s64(e1, e0));
// 设置 dst 中的打包的 8 位整数，以相反的顺序使用提供的值。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setr_epi8
FORCE_INLINE __m128i _mm_setr_epi8(signed char b0,
                                   signed char b1,
                                   signed char b2,
                                   signed char b3,
                                   signed char b4,
                                   signed char b5,
                                   signed char b6,
                                   signed char b7,
                                   signed char b8,
                                   signed char b9,
                                   signed char b10,
                                   signed char b11,
                                   signed char b12,
                                   signed char b13,
                                   signed char b14,
                                   signed char b15)
{
    // 创建一个包含提供的值的数组，以相反的顺序
    int8_t ALIGN_STRUCT(16)
        data[16] = {(int8_t) b0,  (int8_t) b1,  (int8_t) b2,  (int8_t) b3,
                    (int8_t) b4,  (int8_t) b5,  (int8_t) b6,  (int8_t) b7,
                    (int8_t) b8,  (int8_t) b9,  (int8_t) b10, (int8_t) b11,
                    (int8_t) b12, (int8_t) b13, (int8_t) b14, (int8_t) b15};
    // 将数组转换为 __m128i 类型的向量
    return (__m128i) vld1q_s8(data);
}

// 设置 dst 中的打包的双精度（64 位）浮点元素，以相反的顺序使用提供的值。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setr_pd
FORCE_INLINE __m128d _mm_setr_pd(double e1, double e0)
{
    // 调用 _mm_set_pd 函数，将提供的值按相反的顺序设置到 __m128d 类型的向量中
    return _mm_set_pd(e0, e1);
}

// 返回所有元素都设置为零的 __m128d 类型的向量。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setzero_pd
FORCE_INLINE __m128d _mm_setzero_pd(void)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用 vdupq_n_f64 函数创建一个包含零的向量，并将其转换为 __m128d 类型的向量
    return vreinterpretq_m128d_f64(vdupq_n_f64(0));
#else
    # 将一个128位的浮点数向量中的每个元素都设置为0，并将其转换为128位的双精度浮点数向量
    return vreinterpretq_m128d_f32(vdupq_n_f32(0));
#endif
}

// 返回一个所有元素都设置为零的 __m128i 类型的向量。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_setzero_si128
FORCE_INLINE __m128i _mm_setzero_si128(void)
{
    // 使用 vdupq_n_s32 函数将 0 复制到一个 int32x4_t 类型的向量中，并使用 vreinterpretq_m128i_s32 函数将其转换为 __m128i 类型的向量。
    return vreinterpretq_m128i_s32(vdupq_n_s32(0));
}

// 使用 imm8 中的控制参数对 a 中的 32 位整数进行洗牌，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_shuffle_epi32
// FORCE_INLINE __m128i _mm_shuffle_epi32(__m128i a,
//                                        __constrange(0,255) int imm)
#if defined(_sse2neon_shuffle)
#define _mm_shuffle_epi32(a, imm)                                            \
    __extension__({                                                          \
        // 使用 vreinterpretq_s32_m128i 函数将 a 转换为 int32x4_t 类型的向量。
        int32x4_t _input = vreinterpretq_s32_m128i(a);                       \
        // 使用 vshuffleq_s32 函数对 _input 进行洗牌操作，根据 imm 中的控制参数进行洗牌，并将结果转换为 int32x4_t 类型的向量。
        int32x4_t _shuf =                                                    \
            vshuffleq_s32(_input, _input, (imm) & (0x3), ((imm) >> 2) & 0x3, \
                          ((imm) >> 4) & 0x3, ((imm) >> 6) & 0x3);           \
        // 使用 vreinterpretq_m128i_s32 函数将 _shuf 转换为 __m128i 类型的向量。
        vreinterpretq_m128i_s32(_shuf);                                      \
    })
#else  // generic
#define _mm_shuffle_epi32(a, imm)                           \
#endif

// 使用 imm8 中的控制参数对双精度（64 位）浮点数元素进行洗牌，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_shuffle_pd
#ifdef _sse2neon_shuffle
#define _mm_shuffle_pd(a, b, imm8)                                            \
    // 使用 vreinterpretq_s64_m128d 函数将 a 和 b 转换为 int64x2_t 类型的向量。
    vreinterpretq_m128d_s64(                                                  \
        // 使用 vshuffleq_s64 函数对转换后的向量进行洗牌操作，根据 imm8 中的控制参数进行洗牌。
        vshuffleq_s64(vreinterpretq_s64_m128d(a), vreinterpretq_s64_m128d(b), \
                      imm8 & 0x1, ((imm8 & 0x2) >> 1) + 2))
#else
#define _mm_shuffle_pd(a, b, imm8)                                     \
    # 将两个 64 位整数转换为双精度浮点数，并将结果转换为 128 位整数
    _mm_castsi128_pd(
        # 创建一个 128 位整数，其中高 64 位来自 b 的第 (imm8 & 0x2) >> 1 个元素，低 64 位来自 a 的第 imm8 & 0x1 个元素
        _mm_set_epi64x(
            vgetq_lane_s64(vreinterpretq_s64_m128d(b), (imm8 & 0x2) >> 1), 
            vgetq_lane_s64(vreinterpretq_s64_m128d(a), imm8 & 0x1)
        )
    )
#endif
// 结束条件编译指令

// 定义宏，用于将__m128i类型的数据a按照imm参数指定的方式进行高16位的重新排列
#if defined(_sse2neon_shuffle)
#define _mm_shufflehi_epi16(a, imm)                                           \
    __extension__({                                                           \
        int16x8_t _input = vreinterpretq_s16_m128i(a);                        \
        int16x8_t _shuf =                                                     \
            vshuffleq_s16(_input, _input, 0, 1, 2, 3, ((imm) & (0x3)) + 4,    \
                          (((imm) >> 2) & 0x3) + 4, (((imm) >> 4) & 0x3) + 4, \
                          (((imm) >> 6) & 0x3) + 4);                          \
        vreinterpretq_m128i_s16(_shuf);                                       \
    })
// 使用SSE2NEON的shuffle函数实现_mm_shufflehi_epi16宏
#else  // generic
#define _mm_shufflehi_epi16(a, imm) _mm_shufflehi_epi16_function((a), (imm))
// 使用通用的函数实现_mm_shufflehi_epi16宏
#endif

// 定义宏，用于将__m128i类型的数据a按照imm参数指定的方式进行低16位的重新排列
#if defined(_sse2neon_shuffle)
#define _mm_shufflelo_epi16(a, imm)                                  \
    __extension__({                                                  \
        int16x8_t _input = vreinterpretq_s16_m128i(a);               \
        int16x8_t _shuf = vshuffleq_s16(                             \
            _input, _input, ((imm) & (0x3)), (((imm) >> 2) & 0x3),   \
            (((imm) >> 4) & 0x3), (((imm) >> 6) & 0x3), 4, 5, 6, 7); \
        vreinterpretq_m128i_s16(_shuf);                              \
    })
// 使用SSE2NEON的shuffle函数实现_mm_shufflelo_epi16宏
#else  // generic
#define _mm_shufflelo_epi16(a, imm) _mm_shufflelo_epi16_function((a), (imm))
// 使用通用的函数实现_mm_shufflelo_epi16宏
#endif

// 将a中的16位整数左移count中指定的位数，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sll_epi16
FORCE_INLINE __m128i _mm_sll_epi16(__m128i a, __m128i count)
// 定义_mm_sll_epi16函数，实现将a中的16位整数左移count中指定的位数
    # 将count向量中的第一个64位整数转换为64位无符号整数
    uint64_t c = vreinterpretq_nth_u64_m128i(count, 0);
    # 如果c中的值超出了15，返回全零向量
    if (_sse2neon_unlikely(c & ~15))
        return _mm_setzero_si128();
    # 将c中的值扩展为8个16位整数的向量
    int16x8_t vc = vdupq_n_s16((int16_t) c);
    # 将a中的16位整数向左移动c位，然后将结果转换为128位整数向量
    return vreinterpretq_m128i_s16(vshlq_s16(vreinterpretq_s16_m128i(a), vc));
// Shift packed 32-bit integers in a left by count while shifting in zeros, and
// store the results in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sll_epi32
FORCE_INLINE __m128i _mm_sll_epi32(__m128i a, __m128i count)
{
    // 将 count 转换为 64 位整数
    uint64_t c = vreinterpretq_nth_u64_m128i(count, 0);
    // 如果 count 超出范围，则返回全零的结果
    if (_sse2neon_unlikely(c & ~31))
        return _mm_setzero_si128();

    // 将 c 转换为 32 位整数并复制到每个元素中
    int32x4_t vc = vdupq_n_s32((int32_t) c);
    // 对 a 中的每个元素左移 c 位，并将结果转换为 128 位整数
    return vreinterpretq_m128i_s32(vshlq_s32(vreinterpretq_s32_m128i(a), vc));
}

// Shift packed 64-bit integers in a left by count while shifting in zeros, and
// store the results in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sll_epi64
FORCE_INLINE __m128i _mm_sll_epi64(__m128i a, __m128i count)
{
    // 将 count 转换为 64 位整数
    uint64_t c = vreinterpretq_nth_u64_m128i(count, 0);
    // 如果 count 超出范围，则返回全零的结果
    if (_sse2neon_unlikely(c & ~63))
        return _mm_setzero_si128();

    // 将 c 转换为 64 位整数并复制到每个元素中
    int64x2_t vc = vdupq_n_s64((int64_t) c);
    // 对 a 中的每个元素左移 c 位，并将结果转换为 128 位整数
    return vreinterpretq_m128i_s64(vshlq_s64(vreinterpretq_s64_m128i(a), vc));
}

// Shift packed 16-bit integers in a left by imm8 while shifting in zeros, and
// store the results in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_slli_epi16
FORCE_INLINE __m128i _mm_slli_epi16(__m128i a, int imm)
{
    // 如果 imm 超出范围，则返回全零的结果
    if (_sse2neon_unlikely(imm & ~15))
        return _mm_setzero_si128();
    // 对 a 中的每个元素左移 imm 位，并将结果转换为 128 位整数
    return vreinterpretq_m128i_s16(
        vshlq_s16(vreinterpretq_s16_m128i(a), vdupq_n_s16(imm)));
}

// Shift packed 32-bit integers in a left by imm8 while shifting in zeros, and
// store the results in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_slli_epi32
FORCE_INLINE __m128i _mm_slli_epi32(__m128i a, int imm)
{
    // 如果 imm 超出范围，则返回全零的结果
    if (_sse2neon_unlikely(imm & ~31))
        return _mm_setzero_si128();
    // 对 a 中的每个元素左移 imm 位，并将结果转换为 128 位整数
    return vreinterpretq_m128i_s32(
        vshlq_s32(vreinterpretq_s32_m128i(a), vdupq_n_s32(imm)));
}
// 使用 imm8 将 packed 64 位整数向左移位，移入零，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_slli_epi64
FORCE_INLINE __m128i _mm_slli_epi64(__m128i a, int imm)
{
    // 如果 imm & ~63 不为 0，则返回全零的 __m128i 对象
    if (_sse2neon_unlikely(imm & ~63))
        return _mm_setzero_si128();
    // 将 a 转换为 int64x2_t 类型，然后左移 imm 位，最后将结果转换为 __m128i 类型返回
    return vreinterpretq_m128i_s64(
        vshlq_s64(vreinterpretq_s64_m128i(a), vdupq_n_s64(imm)));
}

// 使用 imm8 字节将 a 向左移位，移入零，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_slli_si128
#define _mm_slli_si128(a, imm)                                              \
    _sse2neon_define1(                                                      \
        __m128i, a, int8x16_t ret;                                          \
        // 如果 imm 为 0，则返回 a
        if (_sse2neon_unlikely(imm == 0)) ret = vreinterpretq_s8_m128i(_a); \
        // 如果 imm & ~15 不为 0，则返回全零的 __m128i 对象
        else if (_sse2neon_unlikely((imm) & ~15)) ret = vdupq_n_s8(0);      \
        // 否则，将 a 的左移 imm 位的结果存储在 ret 中
        else ret = vextq_s8(vdupq_n_s8(0), vreinterpretq_s8_m128i(_a),      \
                            ((imm <= 0 || imm > 15) ? 0 : (16 - imm)));     \
        // 返回 ret 转换为 __m128i 类型
        _sse2neon_return(vreinterpretq_m128i_s8(ret));)

// 计算 a 中 packed 双精度（64 位）浮点元素的平方根，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sqrt_pd
FORCE_INLINE __m128d _mm_sqrt_pd(__m128d a)
{
    // 如果是 ARM64 架构，则使用 ARM64 指令计算平方根并返回结果
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(vsqrtq_f64(vreinterpretq_f64_m128d(a)));
#else
    // 否则，分别计算 a 中两个元素的平方根，并返回结果
    double a0 = sqrt(((double *) &a)[0]);
    double a1 = sqrt(((double *) &a)[1]);
    return _mm_set_pd(a1, a0);
#endif
}

// 计算 b 中较低的双精度（64 位）浮点元素的平方根，将结果存储在 dst 的较低元素中，并将 a 的较高元素复制到 dst 的较高元素中
// 定义一个函数，实现对两个双精度浮点数进行平方根计算
FORCE_INLINE __m128d _mm_sqrt_sd(__m128d a, __m128d b)
{
    // 如果是 ARM64 架构，则调用 _mm_move_sd 函数，否则调用 _mm_set_pd 和 sqrt 函数
#if defined(__aarch64__) || defined(_M_ARM64)
    return _mm_move_sd(a, _mm_sqrt_pd(b));
#else
    return _mm_set_pd(((double *) &a)[1], sqrt(((double *) &b)[0]));
#endif
}

// 对 16 位整数进行右移位操作，同时保持符号位不变，并将结果存储在 dst 中
FORCE_INLINE __m128i _mm_sra_epi16(__m128i a, __m128i count)
{
    // 获取 count 中的第一个元素
    int64_t c = vgetq_lane_s64(count, 0);
    // 如果 c 中包含超出范围的位移值，则返回 _mm_cmplt_epi16(a, _mm_setzero_si128())
    if (_sse2neon_unlikely(c & ~15))
        return _mm_cmplt_epi16(a, _mm_setzero_si128());
    // 否则，进行位移操作并返回结果
    return vreinterpretq_m128i_s16(
        vshlq_s16((int16x8_t) a, vdupq_n_s16((int) -c)));
}

// 对 32 位整数进行右移位操作，同时保持符号位不变，并将结果存储在 dst 中
FORCE_INLINE __m128i _mm_sra_epi32(__m128i a, __m128i count)
{
    // 获取 count 中的第一个元素
    int64_t c = vgetq_lane_s64(count, 0);
    // 如果 c 中包含超出范围的位移值，则返回 _mm_cmplt_epi32(a, _mm_setzero_si128())
    if (_sse2neon_unlikely(c & ~31))
        return _mm_cmplt_epi32(a, _mm_setzero_si128());
    // 否则，进行位移操作并返回结果
    return vreinterpretq_m128i_s32(
        vshlq_s32((int32x4_t) a, vdupq_n_s32((int) -c)));
}

// 对 16 位整数进行右移位操作，同时保持符号位不变，并将结果存储在 dst 中
FORCE_INLINE __m128i _mm_srai_epi16(__m128i a, int imm)
{
    // 计算实际的位移值
    const int count = (imm & ~15) ? 15 : imm;
    // 进行位移操作并返回结果
    return (__m128i) vshlq_s16((int16x8_t) a, vdupq_n_s16(-count));
}

// 对 32 位整数进行右移位操作，同时保持符号位不变，并将结果存储在 dst 中
// 定义宏，将参数 a 中的 32 位整数向右移动，移动位数由 imm 指定
#define _mm_srai_epi32(a, imm)                                                \
    _sse2neon_define0(                                                        \
        __m128i, a, __m128i ret; if (_sse2neon_unlikely((imm) == 0)) {        \
            ret = _a;                                                         \
        } else if (_sse2neon_likely(0 < (imm) && (imm) < 32)) {               \
            ret = vreinterpretq_m128i_s32(                                    \
                vshlq_s32(vreinterpretq_s32_m128i(_a), vdupq_n_s32(-(imm)))); \
        } else {                                                              \
            ret = vreinterpretq_m128i_s32(                                    \
                vshrq_n_s32(vreinterpretq_s32_m128i(_a), 31));                \
        } _sse2neon_return(ret);)

// 将参数 a 中的 16 位整数向右移动，移动位数由 count 指定，移动过程中补 0，结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_srl_epi16
FORCE_INLINE __m128i _mm_srl_epi16(__m128i a, __m128i count)
{
    // 将 count 转换为 uint64_t 类型
    uint64_t c = vreinterpretq_nth_u64_m128i(count, 0);
    // 如果移动位数超出 0-15 的范围，则返回全 0 的结果
    if (_sse2neon_unlikely(c & ~15))
        return _mm_setzero_si128();

    // 将移动位数转换为 int16x8_t 类型
    int16x8_t vc = vdupq_n_s16(-(int16_t) c);
    // 将 a 中的 16 位整数向左移动，移动位数由 vc 指定，结果存储在 dst 中
    return vreinterpretq_m128i_u16(vshlq_u16(vreinterpretq_u16_m128i(a), vc));
}

// 将参数 a 中的 32 位整数向右移动，移动位数由 count 指定，移动过程中补 0，结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_srl_epi32
FORCE_INLINE __m128i _mm_srl_epi32(__m128i a, __m128i count)
{
    // 将 count 转换为 uint64_t 类型
    uint64_t c = vreinterpretq_nth_u64_m128i(count, 0);
    // 如果移动位数超出 0-31 的范围，则返回全 0 的结果
    if (_sse2neon_unlikely(c & ~31))
        return _mm_setzero_si128();

    // 将移动位数转换为 int32x4_t 类型
    int32x4_t vc = vdupq_n_s32(-(int32_t) c);
    // 将 a 中的 32 位整数向左移动，移动位数由 vc 指定，结果存储在 dst 中
    return vreinterpretq_m128i_u32(vshlq_u32(vreinterpretq_u32_m128i(a), vc));
}
// 将打包的64位整数向右移动指定的位数，同时在右侧补0，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_srl_epi64
FORCE_INLINE __m128i _mm_srl_epi64(__m128i a, __m128i count)
{
    // 将count转换为64位整数
    uint64_t c = vreinterpretq_nth_u64_m128i(count, 0);
    // 如果移动的位数超出了64位整数的范围，则返回全0的__m128i对象
    if (_sse2neon_unlikely(c & ~63))
        return _mm_setzero_si128();

    // 创建一个64位整数的向量，其所有元素都是-c
    int64x2_t vc = vdupq_n_s64(-(int64_t) c);
    // 将a向左移动-c位，并将结果转换为__m128i类型返回
    return vreinterpretq_m128i_u64(vshlq_u64(vreinterpretq_u64_m128i(a), vc));
}

// 将打包的16位整数向右移动指定的位数，同时在右侧补0，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_srli_epi16
#define _mm_srli_epi16(a, imm)                                                \
    _sse2neon_define0(                                                        \
        __m128i, a, __m128i ret; if (_sse2neon_unlikely((imm) & ~15)) {       \
            ret = _mm_setzero_si128();                                        \
        } else {                                                              \
            ret = vreinterpretq_m128i_u16(                                    \
                vshlq_u16(vreinterpretq_u16_m128i(_a), vdupq_n_s16(-(imm)))); \
        } _sse2neon_return(ret);)

// 将打包的32位整数向右移动指定的位数，同时在右侧补0，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_srli_epi32
// FORCE_INLINE __m128i _mm_srli_epi32(__m128i a, __constrange(0,255) int imm)
#define _mm_srli_epi32(a, imm)                                                \
    # 定义一个名为_sse2neon_define0的宏，接受参数__m128i, a, __m128i ret
    # 如果条件(imm) & ~31不成立，将ret设置为全零的__m128i类型
    # 否则，将ret设置为对_a进行32位有符号整数左移-(imm)位的结果，并返回ret
// 定义宏函数 _mm_srli_epi64，用于将 packed 64 位整数向右移动 imm8 位，并在移动过程中补零，将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_srli_epi64
#define _mm_srli_epi64(a, imm)                                                \
    _sse2neon_define0(                                                        \
        __m128i, a, __m128i ret; if (_sse2neon_unlikely((imm) & ~63)) {       \
            ret = _mm_setzero_si128();                                        \
        } else {                                                              \
            ret = vreinterpretq_m128i_u64(                                    \
                vshlq_u64(vreinterpretq_u64_m128i(_a), vdupq_n_s64(-(imm)))); \
        } _sse2neon_return(ret);)

// 定义宏函数 _mm_srli_si128，用于将 a 向右移动 imm8 个字节，并在移动过程中补零，将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_srli_si128
#define _mm_srli_si128(a, imm)                                         \
    _sse2neon_define1(                                                 \
        __m128i, a, int8x16_t ret;                                     \
        if (_sse2neon_unlikely((imm) & ~15)) ret = vdupq_n_s8(0);      \
        else ret = vextq_s8(vreinterpretq_s8_m128i(_a), vdupq_n_s8(0), \
                            (imm > 15 ? 0 : imm));                     \
        _sse2neon_return(vreinterpretq_m128i_s8(ret));)

// 将 a 中的 128 位（由 2 个 packed double-precision (64-bit) floating-point 元素组成）存储到内存中。
// mem_addr 必须对齐到 16 字节边界，否则可能会引发 general-protection 异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_store_pd
FORCE_INLINE void _mm_store_pd(double *mem_addr, __m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    vst1q_f64((float64_t *) mem_addr, vreinterpretq_f64_m128d(a));
#else
    # 将128位的双精度浮点数向量a转换为32位的单精度浮点数向量，并存储到内存地址mem_addr中
    vst1q_f32((float32_t *) mem_addr, vreinterpretq_f32_m128d(a));
#endif
}

// 将 a 的低双精度（64位）浮点元素存储到内存中的2个连续元素中。
// mem_addr 必须对齐到16字节边界，否则可能会引发通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_store_pd1
FORCE_INLINE void _mm_store_pd1(double *mem_addr, __m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 将 a 的低64位浮点数转换为64位浮点数向量，并存储到 mem_addr 中
    float64x1_t a_low = vget_low_f64(vreinterpretq_f64_m128d(a));
    vst1q_f64((float64_t *) mem_addr,
              vreinterpretq_f64_m128d(vcombine_f64(a_low, a_low)));
#else
    // 将 a 的低32位浮点数转换为32位浮点数向量，并存储到 mem_addr 中
    float32x2_t a_low = vget_low_f32(vreinterpretq_f32_m128d(a));
    vst1q_f32((float32_t *) mem_addr,
              vreinterpretq_f32_m128d(vcombine_f32(a_low, a_low)));
#endif
}

// 将 a 的低双精度（64位）浮点元素存储到内存中。
// mem_addr 不需要对齐到任何特定边界。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=mm_store_sd
FORCE_INLINE void _mm_store_sd(double *mem_addr, __m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 将 a 的低64位浮点数转换为64位浮点数向量，并存储到 mem_addr 中
    vst1_f64((float64_t *) mem_addr, vget_low_f64(vreinterpretq_f64_m128d(a)));
#else
    // 将 a 的低64位无符号整数转换为64位无符号整数向量，并存储到 mem_addr 中
    vst1_u64((uint64_t *) mem_addr, vget_low_u64(vreinterpretq_u64_m128d(a)));
#endif
}

// 将 a 的128位整数数据存储到内存中。
// mem_addr 必须对齐到16字节边界，否则可能会引发通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_store_si128
FORCE_INLINE void _mm_store_si128(__m128i *p, __m128i a)
{
    // 将 a 的32位有符号整数转换为32位有符号整数向量，并存储到 p 中
    vst1q_s32((int32_t *) p, vreinterpretq_s32_m128i(a));
}

// 将 a 的低双精度（64位）浮点元素存储到内存中的2个连续元素中。
// mem_addr 必须对齐到16字节边界，否则可能会引发通用保护异常。
// 定义宏，将 _mm_store1_pd 重命名为 _mm_store_pd1

// 将参数 a 中的高位双精度浮点数存储到内存中
FORCE_INLINE void _mm_storeh_pd(double *mem_addr, __m128d a)
{
    // 如果是 ARM64 架构，则使用 ARM64 指令集将高位双精度浮点数存储到内存中
    vst1_f64((float64_t *) mem_addr, vget_high_f64(vreinterpretq_f64_m128d(a)));
    // 如果不是 ARM64 架构，则使用其他指令集将高位双精度浮点数存储到内存中
    vst1_f32((float32_t *) mem_addr, vget_high_f32(vreinterpretq_f32_m128d(a)));
}

// 将参数 b 中的第一个64位整数存储到内存中
FORCE_INLINE void _mm_storel_epi64(__m128i *a, __m128i b)
{
    // 使用 ARM64 指令集将低位64位整数存储到内存中
    vst1_u64((uint64_t *) a, vget_low_u64(vreinterpretq_u64_m128i(b)));
}

// 将参数 a 中的低位双精度浮点数存储到内存中
FORCE_INLINE void _mm_storel_pd(double *mem_addr, __m128d a)
{
    // 如果是 ARM64 架构，则使用 ARM64 指令集将低位双精度浮点数存储到内存中
    vst1_f64((float64_t *) mem_addr, vget_low_f64(vreinterpretq_f64_m128d(a)));
    // 如果不是 ARM64 架构，则使用其他指令集将低位双精度浮点数存储到内存中
    vst1_f32((float32_t *) mem_addr, vget_low_f32(vreinterpretq_f32_m128d(a)));
}

// 将参数 a 中的两个双精度浮点数以相反的顺序存储到内存中
FORCE_INLINE void _mm_storer_pd(double *mem_addr, __m128d a)
{
    // 将参数 a 转换为单精度浮点数，并将两个单精度浮点数以相反的顺序存储到内存中
    float32x4_t f = vreinterpretq_f32_m128d(a);
    _mm_store_pd(mem_addr, vreinterpretq_m128d_f32(vextq_f32(f, f, 2)));
}
// 从寄存器 a 中存储 128 位的双精度浮点数据到内存中，mem_addr 不需要对齐到特定边界
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_storeu_pd
FORCE_INLINE void _mm_storeu_pd(double *mem_addr, __m128d a)
{
    // 调用 _mm_store_pd 函数，将寄存器 a 中的数据存储到内存中
    _mm_store_pd(mem_addr, a);
}

// 从寄存器 a 中存储 128 位的整数数据到内存中，mem_addr 不需要对齐到特定边界
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_storeu_si128
FORCE_INLINE void _mm_storeu_si128(__m128i *p, __m128i a)
{
    // 调用 vst1q_s32 函数，将寄存器 a 中的数据存储到内存中
    vst1q_s32((int32_t *) p, vreinterpretq_s32_m128i(a));
}

// 从寄存器 a 的第一个元素中存储 32 位整数到内存中，mem_addr 不需要对齐到特定边界
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_storeu_si32
FORCE_INLINE void _mm_storeu_si32(void *p, __m128i a)
{
    // 调用 vst1q_lane_s32 函数，将寄存器 a 中的数据存储到内存中
    vst1q_lane_s32((int32_t *) p, vreinterpretq_s32_m128i(a), 0);
}

// 从寄存器 a 中存储 128 位的双精度浮点数据到内存中，使用非临时内存提示。mem_addr 必须对齐到 16 字节边界，否则可能会生成通用保护异常
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_stream_pd
FORCE_INLINE void _mm_stream_pd(double *p, __m128d a)
{
    // 根据不同的编译器和架构调用不同的函数来实现非临时存储
    #if __has_builtin(__builtin_nontemporal_store)
        __builtin_nontemporal_store(a, (__m128d *) p);
    #elif defined(__aarch64__) || defined(_M_ARM64)
        vst1q_f64(p, vreinterpretq_f64_m128d(a));
    #else
        vst1q_s64((int64_t *) p, vreinterpretq_s64_m128d(a));
    #endif
}

// 从寄存器 a 中存储 128 位的整数数据到内存中，使用非临时内存提示。mem_addr 必须对齐到 16 字节边界，否则可能会生成通用保护异常
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_stream_si128
// 使用非临时提示将 128 位整数 a 存储到内存中，以最小化缓存污染。如果包含地址 mem_addr 的缓存行已经在缓存中，则将更新缓存。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_stream_si32
FORCE_INLINE void _mm_stream_si32(int *p, int a)
{
    vst1q_lane_s32((int32_t *) p, vdupq_n_s32(a), 0);
}

// 使用非临时提示将 64 位整数 a 存储到内存中，以最小化缓存污染。如果包含地址 mem_addr 的缓存行已经在缓存中，则将更新缓存。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_stream_si64
FORCE_INLINE void _mm_stream_si64(__int64 *p, __int64 a)
{
    vst1_s64((int64_t *) p, vdup_n_s64((int64_t) a));
}

// 从 b 中减去 a 中的打包的 16 位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sub_epi16
FORCE_INLINE __m128i _mm_sub_epi16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s16(
        vsubq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
}

// 从 b 中减去 a 中的打包的 32 位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sub_epi32
FORCE_INLINE __m128i _mm_sub_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s32(
        vsubq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 从 b 中减去 a 中的打包的 64 位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sub_epi64
# 从 a 中减去 b 中的 64 位整数，并将结果存储在 dst 中
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sub_si64
FORCE_INLINE __m64 _mm_sub_si64(__m64 a, __m64 b)
{
    # 将参数a和b从m64类型转换为s64类型，然后进行64位有符号整数的减法运算
    return vreinterpret_m64_s64(
        vsub_s64(vreinterpret_s64_m64(a), vreinterpret_s64_m64(b)));
// 从 b 中减去 a 中的打包的有符号 16 位整数，使用饱和运算，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_subs_epi16
FORCE_INLINE __m128i _mm_subs_epi16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s16(
        vqsubq_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
}

// 从 b 中减去 a 中的打包的有符号 8 位整数，使用饱和运算，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_subs_epi8
FORCE_INLINE __m128i _mm_subs_epi8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s8(
        vqsubq_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
}

// 从 b 中减去 a 中的打包的无符号 16 位整数，使用饱和运算，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_subs_epu16
FORCE_INLINE __m128i _mm_subs_epu16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u16(
        vqsubq_u16(vreinterpretq_u16_m128i(a), vreinterpretq_u16_m128i(b)));
}

// 从 b 中减去 a 中的打包的无符号 8 位整数，使用饱和运算，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_subs_epu8
FORCE_INLINE __m128i _mm_subs_epu8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u8(
        vqsubq_u8(vreinterpretq_u8_m128i(a), vreinterpretq_u8_m128i(b)));
}

#define _mm_ucomieq_sd _mm_comieq_sd
#define _mm_ucomige_sd _mm_comige_sd
#define _mm_ucomigt_sd _mm_comigt_sd
#define _mm_ucomile_sd _mm_comile_sd
#define _mm_ucomilt_sd _mm_comilt_sd
#define _mm_ucomineq_sd _mm_comineq_sd

// 返回类型为 __m128d 的向量，其中元素未定义
// 定义一个函数，返回一个未初始化的__m128d类型的变量
FORCE_INLINE __m128d _mm_undefined_pd(void)
{
#if defined(__GNUC__) || defined(__clang__)
#pragma GCC diagnostic push
#pragma GCC diagnostic ignored "-Wuninitialized"
#endif
    // 声明一个__m128d类型的变量a
    __m128d a;
#if defined(_MSC_VER)
    // 如果是_MSC_VER编译器，则将a初始化为0
    a = _mm_setzero_pd();
#endif
    // 返回变量a
    return a;
#if defined(__GNUC__) || defined(__clang__)
#pragma GCC diagnostic pop
#endif
}

// 从a和b的高半部分解压和交错16位整数，并将结果存储在dst中
FORCE_INLINE __m128i _mm_unpackhi_epi16(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128i_s16(
        vzip2q_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
#else
    // 获取a和b的高16位整数
    int16x4_t a1 = vget_high_s16(vreinterpretq_s16_m128i(a));
    int16x4_t b1 = vget_high_s16(vreinterpretq_s16_m128i(b));
    // 将a1和b1进行16位整数的交错
    int16x4x2_t result = vzip_s16(a1, b1);
    // 将交错后的结果合并为一个__m128i类型的变量并返回
    return vreinterpretq_m128i_s16(vcombine_s16(result.val[0], result.val[1]));
#endif
}

// 从a和b的高半部分解压和交错32位整数，并将结果存储在dst中
FORCE_INLINE __m128i _mm_unpackhi_epi32(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128i_s32(
        vzip2q_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
#else
    // 获取a和b的高32位整数
    int32x2_t a1 = vget_high_s32(vreinterpretq_s32_m128i(a));
    int32x2_t b1 = vget_high_s32(vreinterpretq_s32_m128i(b));
    // 将a1和b1进行32位整数的交错
    int32x2x2_t result = vzip_s32(a1, b1);
    // 将交错后的结果合并为一个__m128i类型的变量并返回
    return vreinterpretq_m128i_s32(vcombine_s32(result.val[0], result.val[1]));
#endif
}

// 从a和b的高半部分解压和交错64位整数，并将结果存储在dst中
// 使用 SSE 指令集实现的函数，用于将两个 64 位整数的高位部分进行交错展开
FORCE_INLINE __m128i _mm_unpackhi_epi64(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 ARM64 指令集实现
    return vreinterpretq_m128i_s64(
        vzip2q_s64(vreinterpretq_s64_m128i(a), vreinterpretq_s64_m128i(b)));
#else
    // 如果不是 ARM64 架构，则使用普通的整数操作实现
    int64x1_t a_h = vget_high_s64(vreinterpretq_s64_m128i(a));
    int64x1_t b_h = vget_high_s64(vreinterpretq_s64_m128i(b));
    return vreinterpretq_m128i_s64(vcombine_s64(a_h, b_h));
#endif
}

// 使用 SSE 指令集实现的函数，用于将两个 8 位整数的高位部分进行交错展开
FORCE_INLINE __m128i _mm_unpackhi_epi8(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 ARM64 指令集实现
    return vreinterpretq_m128i_s8(
        vzip2q_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
#else
    // 如果不是 ARM64 架构，则使用普通的整数操作实现
    int8x8_t a1 =
        vreinterpret_s8_s16(vget_high_s16(vreinterpretq_s16_m128i(a)));
    int8x8_t b1 =
        vreinterpret_s8_s16(vget_high_s16(vreinterpretq_s16_m128i(b)));
    int8x8x2_t result = vzip_s8(a1, b1);
    return vreinterpretq_m128i_s8(vcombine_s8(result.val[0], result.val[1]));
#endif
}

// 使用 SSE 指令集实现的函数，用于将两个双精度浮点数的高位部分进行交错展开
FORCE_INLINE __m128d _mm_unpackhi_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用 ARM64 指令集实现
    return vreinterpretq_m128d_f64(
        vzip2q_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    // 如果不是 ARM64 架构，则使用普通的整数操作实现
    return vreinterpretq_m128d_s64(
        vcombine_s64(vget_high_s64(vreinterpretq_s64_m128d(a)),
                     vget_high_s64(vreinterpretq_s64_m128d(b))));
#endif
}
// 从 a 和 b 的低半部分解包和交错 16 位整数，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_unpacklo_epi16
FORCE_INLINE __m128i _mm_unpacklo_epi16(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用 ARM64 架构的指令实现解包和交错操作
    return vreinterpretq_m128i_s16(
        vzip1q_s16(vreinterpretq_s16_m128i(a), vreinterpretq_s16_m128i(b)));
#else
    // 获取 a 和 b 的低 16 位整数
    int16x4_t a1 = vget_low_s16(vreinterpretq_s16_m128i(a));
    int16x4_t b1 = vget_low_s16(vreinterpretq_s16_m128i(b));
    // 将 a1 和 b1 进行交错操作
    int16x4x2_t result = vzip_s16(a1, b1);
    // 将交错后的结果重新组合成一个 128 位整数
    return vreinterpretq_m128i_s16(vcombine_s16(result.val[0], result.val[1]));
#endif
}

// 从 a 和 b 的低半部分解包和交错 32 位整数，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_unpacklo_epi32
FORCE_INLINE __m128i _mm_unpacklo_epi32(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用 ARM64 架构的指令实现解包和交错操作
    return vreinterpretq_m128i_s32(
        vzip1q_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
#else
    // 获取 a 和 b 的低 32 位整数
    int32x2_t a1 = vget_low_s32(vreinterpretq_s32_m128i(a));
    int32x2_t b1 = vget_low_s32(vreinterpretq_s32_m128i(b));
    // 将 a1 和 b1 进行交错操作
    int32x2x2_t result = vzip_s32(a1, b1);
    // 将交错后的结果重新组合成一个 128 位整数
    return vreinterpretq_m128i_s32(vcombine_s32(result.val[0], result.val[1]));
#endif
}

// 从 a 和 b 的低半部分解包和交错 64 位整数，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_unpacklo_epi64
FORCE_INLINE __m128i _mm_unpacklo_epi64(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用 ARM64 架构的指令实现解包和交错操作
    return vreinterpretq_m128i_s64(
        vzip1q_s64(vreinterpretq_s64_m128i(a), vreinterpretq_s64_m128i(b)));
#else
    // 获取 a 和 b 的低 64 位整数
    int64x1_t a_l = vget_low_s64(vreinterpretq_s64_m128i(a));
    int64x1_t b_l = vget_low_s64(vreinterpretq_s64_m128i(b));
    // 将 a_l 和 b_l 进行交错操作
    return vreinterpretq_m128i_s64(vcombine_s64(a_l, b_l));
#endif
}
// 从 a 和 b 的低半部分解压和交错 8 位整数，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_unpacklo_epi8
FORCE_INLINE __m128i _mm_unpacklo_epi8(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpretq_m128i_s8(
        vzip1q_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    int8x8_t a1 = vreinterpret_s8_s16(vget_low_s16(vreinterpretq_s16_m128i(a)));
    int8x8_t b1 = vreinterpret_s8_s16(vget_low_s16(vreinterpretq_s16_m128i(b)));
    int8x8x2_t result = vzip_s8(a1, b1);
    return vreinterpretq_m128i_s8(vcombine_s8(result.val[0], result.val[1]));
#endif
}

// 从 a 和 b 的低半部分解压和交错双精度（64 位）浮点元素，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_unpacklo_pd
FORCE_INLINE __m128d _mm_unpacklo_pd(__m128d a, __m128d b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpretq_m128d_f64(
        vzip1q_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    return vreinterpretq_m128d_s64(
        vcombine_s64(vget_low_s64(vreinterpretq_s64_m128d(a)),
                     vget_low_s64(vreinterpretq_s64_m128d(b))));
#endif
}

// 计算 a 和 b 中打包的双精度（64 位）浮点元素的按位异或，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_xor_pd
FORCE_INLINE __m128d _mm_xor_pd(__m128d a, __m128d b)
{
    return vreinterpretq_m128d_s64(
        veorq_s64(vreinterpretq_s64_m128d(a), vreinterpretq_s64_m128d(b)));
}

// 计算 a 和 b 中的 128 位（表示整数数据）的按位异或，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_xor_si128
# 定义一个函数，实现两个 __m128i 类型的参数进行异或操作，并返回结果
def _mm_xor_si128(a, b):
    # 将 a 和 b 转换为 s32 类型的向量，再进行异或操作
    return vreinterpretq_m128i_s32(
        veorq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)))

# SSE3

# 定义一个函数，实现两个 __m128d 类型的参数进行加减操作，并返回结果
def _mm_addsub_pd(a, b):
    # 定义一个 __m128d 类型的常量 mask，值为 [1.0, -1.0]
    _sse2neon_const __m128d mask = _mm_set_pd(1.0f, -1.0f)
    # 如果是 ARM64 架构，使用 vfmaq_f64 函数进行加减操作
    if defined(__aarch64__) or defined(_M_ARM64):
        return vreinterpretq_m128d_f64(vfmaq_f64(vreinterpretq_f64_m128d(a),
                                                 vreinterpretq_f64_m128d(b),
                                                 vreinterpretq_f64_m128d(mask)))
    # 否则，使用 _mm_add_pd 和 _mm_mul_pd 函数进行加减操作
    else:
        return _mm_add_pd(_mm_mul_pd(b, mask), a)

# 定义一个函数，实现两个 __m128 类型的参数进行加减操作，并返回结果
def _mm_addsub_ps(a, b):
    # 定义一个 __m128 类型的常量 mask，值为 [-1.0, 1.0, -1.0, 1.0]
    _sse2neon_const __m128 mask = _mm_setr_ps(-1.0f, 1.0f, -1.0f, 1.0f)
    # 如果是 ARM64 架构或支持 FMA 指令集，使用 vfmaq_f32 函数进行加减操作
    if (defined(__aarch64__) or defined(_M_ARM64)) or defined(__ARM_FEATURE_FMA):
        return vreinterpretq_m128_f32(vfmaq_f32(vreinterpretq_f32_m128(a),
                                                vreinterpretq_f32_m128(mask),
                                                vreinterpretq_f32_m128(b)))
    # 否则，使用 _mm_add_ps 和 _mm_mul_ps 函数进行加减操作
    else:
        return _mm_add_ps(_mm_mul_ps(b, mask), a)

# 定义一个函数，实现两个 __m128d 类型的参数进行水平加操作，并返回结果
def _mm_hadd_pd(a, b):
    # 如果是 ARM64 架构，使用 vfmaq_f64 函数进行水平加操作
    if defined(__aarch64__) or defined(_M_ARM64):
    # 将两个128位的浮点数向量a和b进行相加，并将结果转换为128位的浮点数向量
    return vreinterpretq_m128d_f64(
        # 将128位的浮点数向量a转换为128位的浮点数向量
        vpaddq_f64(vreinterpretq_f64_m128d(a), vreinterpretq_f64_m128d(b)));
// 如果不是 ARM 架构，则执行以下代码块
else
    // 将 a 强制转换为 double 类型的指针，并赋值给 da
    double *da = (double *) &a;
    // 将 b 强制转换为 double 类型的指针，并赋值给 db
    double *db = (double *) &b;
    // 创建一个包含 da[0] + da[1] 和 db[0] + db[1] 的 double 数组 c
    double c[] = {da[0] + da[1], db[0] + db[1]};
    // 将 c 转换为 uint64_t 类型的指针，并加载到 128 位寄存器中，然后将结果转换为 __m128 类型
    return vreinterpretq_m128d_u64(vld1q_u64((uint64_t *) c));
#endif
}

// 水平相加 a 和 b 中相邻的单精度（32 位）浮点数元素，并将结果打包到 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hadd_ps
FORCE_INLINE __m128 _mm_hadd_ps(__m128 a, __m128 b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 将 a 和 b 分别转换为 float32x4_t 类型，并将其转换为 float32x2_t 类型，然后使用 vpaddq_f32 函数进行相加
    return vreinterpretq_m128_f32(
        vpaddq_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(b)));
#else
    // 将 a 转换为 float32x4_t 类型，并使用 vget_low_f32 和 vget_high_f32 函数分别获取低位和高位的 float32x2_t 类型
    float32x2_t a10 = vget_low_f32(vreinterpretq_f32_m128(a));
    float32x2_t a32 = vget_high_f32(vreinterpretq_f32_m128(a));
    // 将 b 转换为 float32x4_t 类型，并使用 vget_low_f32 和 vget_high_f32 函数分别获取低位和高位的 float32x2_t 类型
    float32x2_t b10 = vget_low_f32(vreinterpretq_f32_m128(b));
    float32x2_t b32 = vget_high_f32(vreinterpretq_f32_m128(b));
    // 使用 vpadd_f32 函数对 a10 和 a32 进行相加，然后使用 vcombine_f32 函数将结果合并为一个 float32x4_t 类型
    // 同样的操作对 b10 和 b32 进行相加，然后将结果转换为 __m128 类型
    return vreinterpretq_m128_f32(
        vcombine_f32(vpadd_f32(a10, a32), vpadd_f32(b10, b32)));
#endif
}

// 水平相减 a 和 b 中相邻的双精度（64 位）浮点数元素，并将结果打包到 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hsub_pd
FORCE_INLINE __m128d _mm_hsub_pd(__m128d _a, __m128d _b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 将 _a 和 _b 分别转换为 float64x2_t 类型，并使用 vuzp1q_f64 和 vuzp2q_f64 函数进行相减
    float64x2_t a = vreinterpretq_f64_m128d(_a);
    float64x2_t b = vreinterpretq_f64_m128d(_b);
    // 使用 vsubq_f64 函数对 a 和 b 进行相减，然后使用 vreinterpretq_m128d_f64 函数将结果转换为 __m128d 类型
    return vreinterpretq_m128d_f64(
        vsubq_f64(vuzp1q_f64(a, b), vuzp2q_f64(a, b)));
#else
    // 将 _a 强制转换为 double 类型的指针，并赋值给 da
    double *da = (double *) &_a;
    // 将 _b 强制转换为 double 类型的指针，并赋值给 db
    double *db = (double *) &_b;
    // 创建一个包含 da[0] - da[1] 和 db[0] - db[1] 的 double 数组 c
    double c[] = {da[0] - da[1], db[0] - db[1]};
    // 将 c 转换为 uint64_t 类型的指针，并加载到 128 位寄存器中，然后将结果转换为 __m128d 类型
    return vreinterpretq_m128d_u64(vld1q_u64((uint64_t *) c));
#endif
}

// 水平相减 a 和 b 中相邻的单精度（32 位）浮点数元素，并将结果打包到 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hsub_ps
FORCE_INLINE __m128 _mm_hsub_ps(__m128 _a, __m128 _b)
{
    // 将_m128类型的数据_a重新解释为float32x4_t类型的数据a
    float32x4_t a = vreinterpretq_f32_m128(_a);
    // 将_m128类型的数据_b重新解释为float32x4_t类型的数据b
    float32x4_t b = vreinterpretq_f32_m128(_b);
    // 如果是 ARM64 架构，使用vuzp1q_f32和vuzp2q_f32函数进行操作
    #if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128_f32(
        vsubq_f32(vuzp1q_f32(a, b), vuzp2q_f32(a, b)));
    // 如果不是 ARM64 架构，使用vuzpq_f32函数进行操作
    #else
    float32x4x2_t c = vuzpq_f32(a, b);
    return vreinterpretq_m128_f32(vsubq_f32(c.val[0], c.val[1]));
    #endif
}

// 从非对齐内存中加载128位整数数据到dst中。当数据跨越缓存行边界时，此intrinsic可能比_mm_loadu_si128表现更好。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_lddqu_si128
#define _mm_lddqu_si128 _mm_loadu_si128

// 从内存中加载双精度（64位）浮点元素到dst的两个元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_loaddup_pd
#define _mm_loaddup_pd _mm_load1_pd

// 复制a的低双精度（64位）浮点元素，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movedup_pd
FORCE_INLINE __m128d _mm_movedup_pd(__m128d a)
{
    // 如果是 ARM64 架构，使用vdupq_laneq_f64和vreinterpretq_f64_m128d函数进行操作
    #if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(
        vdupq_laneq_f64(vreinterpretq_f64_m128d(a), 0));
    // 如果不是 ARM64 架构，使用vdupq_n_u64和vreinterpretq_u64_m128d函数进行操作
    #else
    return vreinterpretq_m128d_u64(
        vdupq_n_u64(vgetq_lane_u64(vreinterpretq_u64_m128d(a), 0)));
    #endif
}

// 复制a的奇索引单精度（32位）浮点元素，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_movehdup_ps
FORCE_INLINE __m128 _mm_movehdup_ps(__m128 a)
{
    // 如果是 ARM64 架构，使用vtrn2q_f32和vreinterpretq_f32_m128函数进行操作
    #if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128_f32(
        vtrn2q_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(a)));
    // 如果是_sse2neon_shuffle，使用vshuffleq_s32和vreinterpretq_f32_m128函数进行操作
    #elif defined(_sse2neon_shuffle)
    return vreinterpretq_m128_f32(vshuffleq_s32(
        vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(a), 1, 1, 3, 3));
    #else
    # 从128位寄存器a中提取第1个32位浮点数，并赋值给a1
    float32_t a1 = vgetq_lane_f32(vreinterpretq_f32_m128(a), 1);
    # 从128位寄存器a中提取第3个32位浮点数，并赋值给a3
    float32_t a3 = vgetq_lane_f32(vreinterpretq_f32_m128(a), 3);
    # 创建一个包含4个32位浮点数的数组，值分别为a1, a1, a3, a3
    float ALIGN_STRUCT(16) data[4] = {a1, a1, a3, a3};
    # 从数组data中加载4个32位浮点数，创建一个128位寄存器，并返回
    return vreinterpretq_m128_f32(vld1q_f32(data));
#endif
}

// 从a中复制偶数索引的单精度（32位）浮点元素，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_moveldup_ps
FORCE_INLINE __m128 _mm_moveldup_ps(__m128 a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128_f32(
        vtrn1q_f32(vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(a)));
#elif defined(_sse2neon_shuffle)
    return vreinterpretq_m128_f32(vshuffleq_s32(
        vreinterpretq_f32_m128(a), vreinterpretq_f32_m128(a), 0, 0, 2, 2));
#else
    float32_t a0 = vgetq_lane_f32(vreinterpretq_f32_m128(a), 0);
    float32_t a2 = vgetq_lane_f32(vreinterpretq_f32_m128(a), 2);
    float ALIGN_STRUCT(16) data[4] = {a0, a0, a2, a2};
    return vreinterpretq_m128_f32(vld1q_f32(data));
#endif
}

/* SSSE3 */

// 计算a中打包的有符号16位整数的绝对值，并将无符号结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_abs_epi16
FORCE_INLINE __m128i _mm_abs_epi16(__m128i a)
{
    return vreinterpretq_m128i_s16(vabsq_s16(vreinterpretq_s16_m128i(a)));
}

// 计算a中打包的有符号32位整数的绝对值，并将无符号结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_abs_epi32
FORCE_INLINE __m128i _mm_abs_epi32(__m128i a)
{
    return vreinterpretq_m128i_s32(vabsq_s32(vreinterpretq_s32_m128i(a)));
}

// 计算a中打包的有符号8位整数的绝对值，并将无符号结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_abs_epi8
FORCE_INLINE __m128i _mm_abs_epi8(__m128i a)
{
    return vreinterpretq_m128i_s8(vabsq_s8(vreinterpretq_s8_m128i(a)));
}

// 计算a中打包的有符号16位整数的绝对值，并将无符号结果存储在dst中。
// 使用 SIMD 指令计算 16 位有符号整数的绝对值，并将无符号结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_abs_pi16
FORCE_INLINE __m64 _mm_abs_pi16(__m64 a)
{
    return vreinterpret_m64_s16(vabs_s16(vreinterpret_s16_m64(a)));
}

// 使用 SIMD 指令计算 32 位有符号整数的绝对值，并将无符号结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_abs_pi32
FORCE_INLINE __m64 _mm_abs_pi32(__m64 a)
{
    return vreinterpret_m64_s32(vabs_s32(vreinterpret_s32_m64(a)));
}

// 使用 SIMD 指令计算 8 位有符号整数的绝对值，并将无符号结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_abs_pi8
FORCE_INLINE __m64 _mm_abs_pi8(__m64 a)
{
    return vreinterpret_m64_s8(vabs_s8(vreinterpret_s8_m64(a)));
}

// 将 a 和 b 中的 16 字节块连接成一个 32 字节临时结果，将结果右移 imm8 字节，并将低 16 字节存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_alignr_epi8
#if defined(__GNUC__) && !defined(__clang__)
#define _mm_alignr_epi8(a, b, imm)                                            \
    # 定义一个内联函数，实现将两个128位寄存器转换为8位无符号整数类型的操作
    __extension__({                                                           \
        # 将参数a转换为8位无符号整数类型的128位寄存器_a
        uint8x16_t _a = vreinterpretq_u8_m128i(a);                            \
        # 将参数b转换为8位无符号整数类型的128位寄存器_b
        uint8x16_t _b = vreinterpretq_u8_m128i(b);                            \
        # 定义一个128位寄存器ret
        __m128i ret;                                                          \
        # 如果传入的imm参数超出范围，则将ret初始化为全0的128位寄存器
        if (_sse2neon_unlikely((imm) & ~31))                                  \
            ret = vreinterpretq_m128i_u8(vdupq_n_u8(0));                      \
        # 如果传入的imm参数大于等于16，则将ret初始化为a右移imm位的结果
        else if (imm >= 16)                                                   \
            ret = _mm_srli_si128(a, imm >= 16 ? imm - 16 : 0);                \
        # 如果传入的imm参数小于16，则将ret初始化为_a和_b按imm位进行拼接的结果
        else                                                                  \
            ret =                                                             \
                vreinterpretq_m128i_u8(vextq_u8(_b, _a, imm < 16 ? imm : 0)); \
        # 返回ret
        ret;                                                                  \
    })
#else
// 如果不满足条件，则定义 _mm_alignr_epi8 宏
#define _mm_alignr_epi8(a, b, imm)                                          \
    // 定义 _sse2neon_define2 宏，将 __m128i 类型的 a 和 b 转换为 uint8x16_t 类型
    _sse2neon_define2(                                                      \
        __m128i, a, b, uint8x16_t __a = vreinterpretq_u8_m128i(_a);         \
        uint8x16_t __b = vreinterpretq_u8_m128i(_b); __m128i ret;           \
        // 如果 imm 超出范围，则返回全 0 的结果
        if (_sse2neon_unlikely((imm) & ~31)) ret =                          \
            vreinterpretq_m128i_u8(vdupq_n_u8(0));                          \
        // 如果 imm 大于等于 16，则将 a 向右移 imm-16 个字节
        else if (imm >= 16) ret =                                           \
            _mm_srli_si128(_a, imm >= 16 ? imm - 16 : 0);                   \
        // 否则，将 a 和 b 按 imm 的值进行拼接，并返回结果
        else ret =                                                          \
            vreinterpretq_m128i_u8(vextq_u8(__b, __a, imm < 16 ? imm : 0)); \
        // 返回结果
        _sse2neon_return(ret);)

#endif

// 将 a 和 b 中的 8 字节块连接成一个 16 字节的临时结果，将结果向右移 imm8 个字节，并将低 8 字节存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_alignr_pi8
#define _mm_alignr_pi8(a, b, imm)                                           \
    # 定义一个宏函数，用于将__m64类型的数据a和b按照imm参数指定的规则进行处理，并返回结果
    _sse2neon_define2(                                                      \
        __m64, a, b, __m64 ret; if (_sse2neon_unlikely((imm) >= 16)) {      \
            # 如果imm参数大于等于16，则返回一个全0的__m64类型数据
            ret = vreinterpret_m64_s8(vdup_n_s8(0));                        \
        } else {                                                            \
            uint8x8_t tmp_low;                                              \
            uint8x8_t tmp_high;                                             \
            if ((imm) >= 8) {                                               \
                # 如果imm参数大于等于8，则将_a转换为uint8x8_t类型的数据，并将_b转换为全0的uint8x8_t类型数据
                const int idx = (imm) -8;                                   \
                tmp_low = vreinterpret_u8_m64(_a);                          \
                tmp_high = vdup_n_u8(0);                                    \
                # 将tmp_low和tmp_high按照idx参数指定的规则进行拼接，并转换为__m64类型的数据
                ret = vreinterpret_m64_u8(vext_u8(tmp_low, tmp_high, idx)); \
            } else {                                                        \
                # 如果imm参数小于8，则将_b转换为uint8x8_t类型的数据，并将_a转换为uint8x8_t类型的数据
                const int idx = (imm);                                      \
                tmp_low = vreinterpret_u8_m64(_b);                          \
                tmp_high = vreinterpret_u8_m64(_a);                         \
                # 将tmp_low和tmp_high按照idx参数指定的规则进行拼接，并转换为__m64类型的数据
                ret = vreinterpret_m64_u8(vext_u8(tmp_low, tmp_high, idx)); \
            }                                                               \
        } _sse2neon_return(ret);)
// 水平相加a和b中相邻的16位整数，并将有符号16位结果打包到dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hadd_epi16
FORCE_INLINE __m128i _mm_hadd_epi16(__m128i _a, __m128i _b)
{
    int16x8_t a = vreinterpretq_s16_m128i(_a);  // 将__m128i类型的_a转换为int16x8_t类型的a
    int16x8_t b = vreinterpretq_s16_m128i(_b);  // 将__m128i类型的_b转换为int16x8_t类型的b
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128i_s16(vpaddq_s16(a, b));  // 如果是ARM64架构，使用vpaddq_s16函数进行16位整数相加
#else
    return vreinterpretq_m128i_s16(
        vcombine_s16(vpadd_s16(vget_low_s16(a), vget_high_s16(a)),  // 否则，使用vpadd_s16函数进行16位整数相加
                     vpadd_s16(vget_low_s16(b), vget_high_s16(b))));
#endif
}

// 水平相加a和b中相邻的32位整数，并将有符号32位结果打包到dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hadd_epi32
FORCE_INLINE __m128i _mm_hadd_epi32(__m128i _a, __m128i _b)
{
    int32x4_t a = vreinterpretq_s32_m128i(_a);  // 将__m128i类型的_a转换为int32x4_t类型的a
    int32x4_t b = vreinterpretq_s32_m128i(_b);  // 将__m128i类型的_b转换为int32x4_t类型的b
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128i_s32(vpaddq_s32(a, b));  // 如果是ARM64架构，使用vpaddq_s32函数进行32位整数相加
#else
    return vreinterpretq_m128i_s32(
        vcombine_s32(vpadd_s32(vget_low_s32(a), vget_high_s32(a)),  // 否则，使用vpadd_s32函数进行32位整数相加
                     vpadd_s32(vget_low_s32(b), vget_high_s32(b))));
#endif
}

// 水平相加a和b中相邻的16位整数，并将有符号16位结果打包到dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hadd_pi16
FORCE_INLINE __m64 _mm_hadd_pi16(__m64 a, __m64 b)
{
    return vreinterpret_m64_s16(
        vpadd_s16(vreinterpret_s16_m64(a), vreinterpret_s16_m64(b)));  // 使用vpadd_s16函数进行16位整数相加
}

// 水平相加a和b中相邻的32位整数，并将有符号32位结果打包到dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hadd_pi32
FORCE_INLINE __m64 _mm_hadd_pi32(__m64 a, __m64 b)
{
    # 将参数a和b分别转换为64位整数向量
    vreinterpret_s32_m64(a)
    vreinterpret_s32_m64(b)
    # 将两个64位整数向量的元素逐个相加
    vpadd_s32(vreinterpret_s32_m64(a), vreinterpret_s32_m64(b))
    # 将相加的结果转换为64位整数向量
    vreinterpret_m64_s32(vpadd_s32(vreinterpret_s32_m64(a), vreinterpret_s32_m64(b)))
    # 返回转换后的64位整数向量
    return vreinterpret_m64_s32(vpadd_s32(vreinterpret_s32_m64(a), vreinterpret_s32_m64(b)))
// 水平相加a和b中相邻的有符号16位整数，使用饱和运算，并将结果打包为有符号16位整数dst。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hadds_epi16
FORCE_INLINE __m128i _mm_hadds_epi16(__m128i _a, __m128i _b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 将__m128i类型的_a和_b转换为int16x8_t类型的变量a和b
    int16x8_t a = vreinterpretq_s16_m128i(_a);
    int16x8_t b = vreinterpretq_s16_m128i(_b);
    // 使用vuzp1q_s16和vuzp2q_s16函数对a和b进行解交错操作，并使用vqaddq_s16函数进行饱和相加
    return vreinterpretq_s64_s16(
        vqaddq_s16(vuzp1q_s16(a, b), vuzp2q_s16(a, b)));
#else
    // 将__m128i类型的_a和_b转换为int32x4_t类型的变量a和b
    int32x4_t a = vreinterpretq_s32_m128i(_a);
    int32x4_t b = vreinterpretq_s32_m128i(_b);
    // 使用vmovn_s32和vshrn_n_s32函数对a和b进行解交错操作，并使用vqaddq_s16函数进行饱和相加
    int16x8_t ab0246 = vcombine_s16(vmovn_s32(a), vmovn_s32(b));
    int16x8_t ab1357 = vcombine_s16(vshrn_n_s32(a, 16), vshrn_n_s32(b, 16));
    // 将结果重新转换为__m128i类型的变量，并返回
    return vreinterpretq_m128i_s16(vqaddq_s16(ab0246, ab1357));
#endif
}

// 水平相加a和b中相邻的有符号16位整数，使用饱和运算，并将结果打包为有符号16位整数dst。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hadds_pi16
FORCE_INLINE __m64 _mm_hadds_pi16(__m64 _a, __m64 _b)
{
    // 将__m64类型的_a和_b转换为int16x4_t类型的变量a和b
    int16x4_t a = vreinterpret_s16_m64(_a);
    int16x4_t b = vreinterpret_s16_m64(_b);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用vuzp1_s16和vuzp2_s16函数对a和b进行解交错操作，并使用vqadd_s16函数进行饱和相加
    return vreinterpret_s64_s16(vqadd_s16(vuzp1_s16(a, b), vuzp2_s16(a, b)));
#else
    // 使用vuzp_s16函数对a和b进行解交错操作，并使用vqadd_s16函数进行饱和相加
    int16x4x2_t res = vuzp_s16(a, b);
    // 将结果重新转换为__m64类型的变量，并返回
    return vreinterpret_s64_s16(vqadd_s16(res.val[0], res.val[1]));
#endif
}

// 水平相减a和b中相邻的16位整数，并将结果打包为有符号16位整数dst。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hsub_epi16
FORCE_INLINE __m128i _mm_hsub_epi16(__m128i _a, __m128i _b)
{
    // 将__m128i类型的_a和_b转换为int16x8_t类型的变量a和b
    int16x8_t a = vreinterpretq_s16_m128i(_a);
    int16x8_t b = vreinterpretq_s16_m128i(_b);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpretq_m128i_s16(
        // 将 a 和 b 按照元素位置交替组合，然后对应位置相减，最后将结果转换为 128 位整数向量
        vsubq_s16(vuzp1q_s16(a, b), vuzp2q_s16(a, b)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    int16x8x2_t c = vuzpq_s16(a, b);
    // 将 a 和 b 按照元素位置交替组合，然后对应位置相减，最后将结果转换为 128 位整数向量
    return vreinterpretq_m128i_s16(vsubq_s16(c.val[0], c.val[1]));
#endif
}

// 水平减去 a 和 b 中相邻的 32 位整数，并将有符号的 32 位结果打包到 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hsub_epi32
FORCE_INLINE __m128i _mm_hsub_epi32(__m128i _a, __m128i _b)
{
    int32x4_t a = vreinterpretq_s32_m128i(_a);
    int32x4_t b = vreinterpretq_s32_m128i(_b);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpretq_m128i_s32(
        // 将 a 和 b 按照元素位置交替组合，然后对应位置相减，最后将结果转换为 128 位整数向量
        vsubq_s32(vuzp1q_s32(a, b), vuzp2q_s32(a, b)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    int32x4x2_t c = vuzpq_s32(a, b);
    // 将 a 和 b 按照元素位置交替组合，然后对应位置相减，最后将结果转换为 128 位整数向量
    return vreinterpretq_m128i_s32(vsubq_s32(c.val[0], c.val[1]));
#endif
}

// 水平减去 a 和 b 中相邻的 16 位整数，并将有符号的 16 位结果打包到 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hsub_pi16
FORCE_INLINE __m64 _mm_hsub_pi16(__m64 _a, __m64 _b)
{
    int16x4_t a = vreinterpret_s16_m64(_a);
    int16x4_t b = vreinterpret_s16_m64(_b);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpret_m64_s16(vsub_s16(vuzp1_s16(a, b), vuzp2_s16(a, b)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    int16x4x2_t c = vuzp_s16(a, b);
    // 将 a 和 b 按照元素位置交替组合，然后对应位置相减，最后将结果转换为 64 位整数向量
    return vreinterpret_m64_s16(vsub_s16(c.val[0], c.val[1]));
#endif
}

// 水平减去 a 和 b 中相邻的 32 位整数，并将有符号的 32 位结果打包到 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=mm_hsub_pi32
FORCE_INLINE __m64 _mm_hsub_pi32(__m64 _a, __m64 _b)
{
    int32x2_t a = vreinterpret_s32_m64(_a);
    int32x2_t b = vreinterpret_s32_m64(_b);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则执行以下代码
    return vreinterpret_m64_s32(vsub_s32(vuzp1_s32(a, b), vuzp2_s32(a, b)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    int32x2x2_t c = vuzp_s32(a, b);
    # 将两个 64 位整数向量的每个元素分别减去
    return vreinterpret_m64_s32(vsub_s32(c.val[0], c.val[1]));
#endif
}

// 水平相减两个有符号16位整数的相邻对，使用饱和运算，并将结果打包成有符号16位整数存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hsubs_epi16
FORCE_INLINE __m128i _mm_hsubs_epi16(__m128i _a, __m128i _b)
{
    // 将__m128i类型的_a转换为int16x8_t类型的a
    int16x8_t a = vreinterpretq_s16_m128i(_a);
    // 将__m128i类型的_b转换为int16x8_t类型的b
    int16x8_t b = vreinterpretq_s16_m128i(_b);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用饱和运算，水平相减a和b的相邻对，并将结果打包成int16x8_t类型
    return vreinterpretq_m128i_s16(
        vqsubq_s16(vuzp1q_s16(a, b), vuzp2q_s16(a, b)));
#else
    // 使用饱和运算，水平相减a和b的相邻对，并将结果打包成int16x8x2_t类型的c
    int16x8x2_t c = vuzpq_s16(a, b);
    // 将int16x8x2_t类型的c的val[0]和val[1]进行水平相减，并将结果打包成int16x8_t类型
    return vreinterpretq_m128i_s16(vqsubq_s16(c.val[0], c.val[1]));
#endif
}

// 水平相减两个有符号16位整数的相邻对，使用饱和运算，并将结果打包成有符号16位整数存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_hsubs_pi16
FORCE_INLINE __m64 _mm_hsubs_pi16(__m64 _a, __m64 _b)
{
    // 将__m64类型的_a转换为int16x4_t类型的a
    int16x4_t a = vreinterpret_s16_m64(_a);
    // 将__m64类型的_b转换为int16x4_t类型的b
    int16x4_t b = vreinterpret_s16_m64(_b);
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用饱和运算，水平相减a和b的相邻对，并将结果打包成int16x4_t类型
    return vreinterpret_m64_s16(vqsub_s16(vuzp1_s16(a, b), vuzp2_s16(a, b)));
#else
    // 使用饱和运算，水平相减a和b的相邻对，并将结果打包成int16x4x2_t类型的c
    int16x4x2_t c = vuzp_s16(a, b);
    // 将int16x4x2_t类型的c的val[0]和val[1]进行水平相减，并将结果打包成int16x4_t类型
    return vreinterpret_m64_s16(vqsub_s16(c.val[0], c.val[1]));
#endif
}

// 将a中的每个无符号8位整数与b中对应的有符号8位整数相乘，得到中间的有符号16位整数。
// 水平相加中间的有符号16位整数的相邻对，并将饱和的结果打包存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_maddubs_epi16
FORCE_INLINE __m128i _mm_maddubs_epi16(__m128i _a, __m128i _b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 将__m128i类型的_a转换为uint8x16_t类型的a
    uint8x16_t a = vreinterpretq_u8_m128i(_a);
    // 将__m128i类型的_b转换为int8x16_t类型的b
    int8x16_t b = vreinterpretq_s8_m128i(_b);
    // 将a的低8位转换为无符号16位整数，将b的低8位转换为有符号16位整数，相乘得到中间的有符号16位整数
    int16x8_t tl = vmulq_s16(vreinterpretq_s16_u16(vmovl_u8(vget_low_u8(a))),
                             vmovl_s8(vget_low_s8(b)));
    # 创建一个 int16x8_t 类型的变量 th，使用 vget_high_u8(a) 获取 a 的高 8 位，并将其转换为 uint16x8_t 类型，再将其转换为 int16x8_t 类型
    int16x8_t th = vmulq_s16(vreinterpretq_s16_u16(vmovl_u8(vget_high_u8(a))),
                             vmovl_s8(vget_high_s8(b)));
    # 返回一个 int16x8_t 类型的变量，使用 vuzp1q_s16(tl, th) 将 tl 和 th 的元素按照交错顺序组合，再使用 vqaddq_s16 进行饱和加法操作，最后使用 vreinterpretq_m128i_s16 将结果转换为 m128i 类型
    return vreinterpretq_m128i_s16(
        vqaddq_s16(vuzp1q_s16(tl, th), vuzp2q_s16(tl, th)));
#else
    // 如果 x86 选择只进行零扩展或者符号扩展，而不是两者都进行，那么这将简单得多。
    // 这可能可以更好地进行优化。
    // 将 __m128i 类型的 _a 转换为 uint16x8_t 类型的 a
    uint16x8_t a = vreinterpretq_u16_m128i(_a);
    // 将 __m128i 类型的 _b 转换为 int16x8_t 类型的 b
    int16x8_t b = vreinterpretq_s16_m128i(_b);

    // 对 a 进行零扩展
    // 将 a 右移 8 位，然后将结果转换为 int16x8_t 类型的 a_odd
    int16x8_t a_odd = vreinterpretq_s16_u16(vshrq_n_u16(a, 8));
    // 将 a 与 0xff00 进行按位与运算，然后将结果转换为 int16x8_t 类型的 a_even
    int16x8_t a_even = vreinterpretq_s16_u16(vbicq_u16(a, vdupq_n_u16(0xff00)));

    // 通过左移再右移来进行符号扩展
    // 将 b 左移 8 位，然后右移 8 位，将结果转换为 int16x8_t 类型的 b_even
    int16x8_t b_even = vshrq_n_s16(vshlq_n_s16(b, 8), 8);
    // 将 b 右移 8 位，将结果转换为 int16x8_t 类型的 b_odd
    int16x8_t b_odd = vshrq_n_s16(b, 8);

    // 乘法运算
    // 将 a_even 和 b_even 进行乘法运算，将结果保存在 int16x8_t 类型的 prod1 中
    int16x8_t prod1 = vmulq_s16(a_even, b_even);
    // 将 a_odd 和 b_odd 进行乘法运算，将结果保存在 int16x8_t 类型的 prod2 中
    int16x8_t prod2 = vmulq_s16(a_odd, b_odd);

    // 饱和加法
    // 将 prod1 和 prod2 进行饱和加法运算，将结果转换为 __m128i 类型的返回值
    return vreinterpretq_m128i_s16(vqaddq_s16(prod1, prod2));
#endif
}

// 对每个 unsigned 8 位整数 a 和对应的 signed 8 位整数 b 进行垂直乘法运算，
// 生成中间的 signed 16 位整数。水平相加相邻的中间 signed 16 位整数，
// 并将饱和结果打包到 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_maddubs_pi16
FORCE_INLINE __m64 _mm_maddubs_pi16(__m64 _a, __m64 _b)
{
    // 将 __m64 类型的 _a 转换为 uint16x4_t 类型的 a
    uint16x4_t a = vreinterpret_u16_m64(_a);
    // 将 __m64 类型的 _b 转换为 int16x4_t 类型的 b
    int16x4_t b = vreinterpret_s16_m64(_b);

    // 对 a 进行零扩展
    // 将 a 右移 8 位，然后将结果转换为 int16x4_t 类型的 a_odd
    int16x4_t a_odd = vreinterpret_s16_u16(vshr_n_u16(a, 8));
    // 将 a 与 0xff 进行按位与运算，然后将结果转换为 int16x4_t 类型的 a_even
    int16x4_t a_even = vreinterpret_s16_u16(vand_u16(a, vdup_n_u16(0xff)));

    // 通过左移再右移来进行符号扩展
    // 将 b 左移 8 位，然后右移 8 位，将结果转换为 int16x4_t 类型的 b_even
    int16x4_t b_even = vshr_n_s16(vshl_n_s16(b, 8), 8);
    // 将 b 右移 8 位，将结果转换为 int16x4_t 类型的 b_odd
    int16x4_t b_odd = vshr_n_s16(b, 8);

    // 乘法运算
    // 将 a_even 和 b_even 进行乘法运算，将结果保存在 int16x4_t 类型的 prod1 中
    int16x4_t prod1 = vmul_s16(a_even, b_even);
    // 将 a_odd 和 b_odd 进行乘法运算，将结果保存在 int16x4_t 类型的 prod2 中
    int16x4_t prod2 = vmul_s16(a_odd, b_odd);

    // 饱和加法
    // 将 prod1 和 prod2 进行饱和加法运算，将结果转换为 __m64 类型的返回值
    return vreinterpret_m64_s16(vqadd_s16(prod1, prod2));
}

// 对 a 和 b 中的每个 packed signed 16 位整数进行乘法运算，
// 生成中间的 signed 32 位整数。向右移动 15 位并四舍五入，
// 并将打包的 16 位整数存储在 dst 中。
# 定义一个函数，用于将两个 packed signed 16-bit integers 进行乘法运算，并返回结果
def _mm_mulhrs_epi16(a, b):
    # 由于饱和问题，以下代码存在问题
    # return vreinterpretq_m128i_s16(vqrdmulhq_s16(a, b));

    # 将 a 和 b 的低位 16 位分别取出，进行有符号 16 位整数乘法运算
    mul_lo = vmull_s16(vget_low_s16(vreinterpretq_s16_m128i(a)),
                       vget_low_s16(vreinterpretq_s16_m128i(b)))
    # 将 a 和 b 的高位 16 位分别取出，进行有符号 16 位整数乘法运算
    mul_hi = vmull_s16(vget_high_s16(vreinterpretq_s16_m128i(a)),
                       vget_high_s16(vreinterpretq_s16_m128i(b)))

    # 对乘法结果进行舍入、缩小和右移操作
    # narrow = (int16_t)((mul + 16384) >> 15);
    narrow_lo = vrshrn_n_s32(mul_lo, 15)
    narrow_hi = vrshrn_n_s32(mul_hi, 15)

    # 将低位和高位的结果合并
    return vreinterpretq_m128i_s16(vcombine_s16(narrow_lo, narrow_hi))


# 对 packed signed 16-bit integers 进行乘法运算，并返回结果
def _mm_mulhrs_pi16(a, b):
    # 将 a 和 b 进行有符号 16 位整数乘法运算
    mul_extend = vmull_s16((vreinterpret_s16_m64(a)), (vreinterpret_s16_m64(b)))

    # 对乘法结果进行舍入、缩小和右移操作
    return vreinterpret_m64_s16(vrshrn_n_s32(mul_extend, 15))


# 根据掩码 b 对 packed 8-bit integers a 进行洗牌操作，并将结果存储在 dst 中
def _mm_shuffle_epi8(a, b):
    # 将 a 转换为 int8x16_t 类型
    tbl = vreinterpretq_s8_m128i(a)   # input a
    # 将 b 转换为 uint8x16_t 类型
    idx = vreinterpretq_u8_m128i(b)  # input b
    # 将 idx 和 0x8F 进行按位与操作，避免使用无意义的位
    idx_masked = vandq_u8(idx, vdupq_n_u8(0x8F))
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，使用指定的操作进行处理
    return vreinterpretq_m128i_s8(vqtbl1q_s8(tbl, idx_masked));
#elif defined(__GNUC__)
    // 如果是 GNU 编译器，使用内联汇编进行处理
    int8x16_t ret;
    // %e 和 %f 分别代表偶数和奇数的 D 寄存器
    __asm__ __volatile__(
        "vtbl.8  %e[ret], {%e[tbl], %f[tbl]}, %e[idx]\n"
        "vtbl.8  %f[ret], {%e[tbl], %f[tbl]}, %f[idx]\n"
        : [ret] "=&w"(ret)
        : [tbl] "w"(tbl), [idx] "w"(idx_masked));
    return vreinterpretq_m128i_s8(ret);
#else
    // 如果不是 ARM64 架构，使用另一种处理方式
    int8x8x2_t a_split = {vget_low_s8(tbl), vget_high_s8(tbl)};
    return vreinterpretq_m128i_s8(
        vcombine_s8(vtbl2_s8(a_split, vget_low_u8(idx_masked)),
                    vtbl2_s8(a_split, vget_high_u8(idx_masked))));
#endif
}

// 根据掩码在 a 中对 8 位整数进行洗牌，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_shuffle_pi8
FORCE_INLINE __m64 _mm_shuffle_pi8(__m64 a, __m64 b)
{
    const int8x8_t controlMask =
        vand_s8(vreinterpret_s8_m64(b), vdup_n_s8((int8_t) (0x1 << 7 | 0x07)));
    int8x8_t res = vtbl1_s8(vreinterpret_s8_m64(a), controlMask);
    return vreinterpret_m64_s8(res);
}

// 当 b 中对应的有符号 16 位整数为负数时，对 a 中的 16 位整数进行取反，并将结果存储在 dst 中
// 当 b 中对应的元素为零时，将 dst 中的元素清零
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sign_epi16
FORCE_INLINE __m128i _mm_sign_epi16(__m128i _a, __m128i _b)
{
    int16x8_t a = vreinterpretq_s16_m128i(_a);
    int16x8_t b = vreinterpretq_s16_m128i(_b);

    // 有符号右移：比 vclt 更快
    // (b < 0) ? 0xFFFF : 0
    uint16x8_t ltMask = vreinterpretq_u16_s16(vshrq_n_s16(b, 15));
    // (b == 0) ? 0xFFFF : 0
#if defined(__aarch64__) || defined(_M_ARM64)
    # 创建一个16位整数类型的向量，用于存储零值掩码
    int16x8_t zeroMask = vreinterpretq_s16_u16(vceqzq_s16(b));
#else
    // 创建一个掩码，用于将 b 中小于 0 的元素转换为全 1，其它元素转换为全 0
    int16x8_t zeroMask = vreinterpretq_s16_u16(vceqq_s16(b, vdupq_n_s16(0)));
#endif

    // 根据 ltMask，对 a 进行选择，如果 ltMask 对应位置为 1，则选择 a 的负数，否则选择 a 本身
    int16x8_t masked = vbslq_s16(ltMask, vnegq_s16(a), a);
    // 将 masked 和 ~zeroMask 进行按位与操作
    int16x8_t res = vbicq_s16(masked, zeroMask);
    // 将结果转换为 __m128i 类型并返回
    return vreinterpretq_m128i_s16(res);
}

// 当 b 中对应的有符号 32 位整数小于 0 时，对 a 中对应的 32 位整数取反，并将结果存储在 dst 中
// 当 b 中对应的元素为 0 时，将 dst 中对应的元素置为 0
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sign_epi32
FORCE_INLINE __m128i _mm_sign_epi32(__m128i _a, __m128i _b)
{
    // 将 _a 和 _b 转换为 int32x4_t 类型
    int32x4_t a = vreinterpretq_s32_m128i(_a);
    int32x4_t b = vreinterpretq_s32_m128i(_b);

    // 使用有符号右移操作符进行有符号右移：比 vclt 更快
    // (b < 0) ? 0xFFFFFFFF : 0
    uint32x4_t ltMask = vreinterpretq_u32_s32(vshrq_n_s32(b, 31));

    // (b == 0) ? 0xFFFFFFFF : 0
#if defined(__aarch64__) || defined(_M_ARM64)
    int32x4_t zeroMask = vreinterpretq_s32_u32(vceqzq_s32(b));
#else
    int32x4_t zeroMask = vreinterpretq_s32_u32(vceqq_s32(b, vdupq_n_s32(0)));
#endif

    // 根据 ltMask，对 a 进行选择，如果 ltMask 对应位置为 1，则选择 a 的负数，否则选择 a 本身
    int32x4_t masked = vbslq_s32(ltMask, vnegq_s32(a), a);
    // 将 masked 和 ~zeroMask 进行按位与操作
    int32x4_t res = vbicq_s32(masked, zeroMask);
    // 将结果转换为 __m128i 类型并返回
    return vreinterpretq_m128i_s32(res);
}

// 当 b 中对应的有符号 8 位整数小于 0 时，对 a 中对应的 8 位整数取反，并将结果存储在 dst 中
// 当 b 中对应的元素为 0 时，将 dst 中对应的元素置为 0
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sign_epi8
FORCE_INLINE __m128i _mm_sign_epi8(__m128i _a, __m128i _b)
{
    // 将 _a 和 _b 转换为 int8x16_t 类型
    int8x16_t a = vreinterpretq_s8_m128i(_a);
    int8x16_t b = vreinterpretq_s8_m128i(_b);
    // 使用有符号右移操作符进行快速判断是否小于0，如果小于0则返回0xFF，否则返回0
    uint8x16_t ltMask = vreinterpretq_u8_s8(vshrq_n_s8(b, 7));

    // 判断b是否等于0，如果等于0则返回0xFF，否则返回0
#if defined(__aarch64__) || defined(_M_ARM64)
    # 如果是 ARM64 架构，使用 vceqzq_s8 函数判断 b 是否全为 0，返回一个全为 0 的掩码
    int8x16_t zeroMask = vreinterpretq_s8_u8(vceqzq_s8(b));
#else
    # 如果不是 ARM64 架构，使用 vceqq_s8 函数判断 b 是否与全为 0 的向量相等，返回一个全为 0 的掩码
    int8x16_t zeroMask = vreinterpretq_s8_u8(vceqq_s8(b, vdupq_n_s8(0)));
#endif

    // 根据 ltMask 选择 a 或者负的 a（vnegq_s8(a) 返回负的 a）
    // 并根据掩码进行位运算，将 zeroMask 中为 1 的位取反
    int8x16_t masked = vbslq_s8(ltMask, vnegq_s8(a), a);
    // res = masked & (~zeroMask)
    // 将 masked 和 zeroMask 进行位与运算，并将结果转换为 128 位整数向量
    int8x16_t res = vbicq_s8(masked, zeroMask);

    // 将结果向量转换为 128 位整数向量
    return vreinterpretq_m128i_s8(res);
}

// 当 b 中对应的有符号 16 位整数为负数时，对 a 中的打包的 16 位整数取反，并将结果存储在 dst 中。
// 当 b 中对应的元素为 0 时，dst 中的元素被清零。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sign_pi16
FORCE_INLINE __m64 _mm_sign_pi16(__m64 _a, __m64 _b)
{
    // 将 _a 和 _b 转换为 16 位整数向量
    int16x4_t a = vreinterpret_s16_m64(_a);
    int16x4_t b = vreinterpret_s16_m64(_b);

    // 使用有符号右移操作符：比 vclt 更快
    // (b < 0) ? 0xFFFF : 0
    // 将 b 中小于 0 的元素右移 15 位，得到一个掩码
    uint16x4_t ltMask = vreinterpret_u16_s16(vshr_n_s16(b, 15));

    // (b == 0) ? 0xFFFF : 0
#if defined(__aarch64__) || defined(_M_ARM64)
    # 如果是 ARM64 架构，使用 vceqz_s16 函数判断 b 是否全为 0，返回一个全为 0 的掩码
    int16x4_t zeroMask = vreinterpret_s16_u16(vceqz_s16(b));
#else
    # 如果不是 ARM64 架构，使用 vceq_s16 函数判断 b 是否与全为 0 的向量相等，返回一个全为 0 的掩码
    int16x4_t zeroMask = vreinterpret_s16_u16(vceq_s16(b, vdup_n_s16(0)));
#endif

    // 根据 ltMask 选择 a 或者负的 a（vneg_s16(a) 返回负的 a）
    // 并根据掩码进行位运算，将 zeroMask 中为 1 的位取反
    int16x4_t masked = vbsl_s16(ltMask, vneg_s16(a), a);
    // res = masked & (~zeroMask)
    // 将 masked 和 zeroMask 进行位与运算，并将结果转换为 64 位整数向量
    int16x4_t res = vbic_s16(masked, zeroMask);

    // 将结果向量转换为 64 位整数向量
    return vreinterpret_m64_s16(res);
}

// 当 b 中对应的有符号 32 位整数为负数时，对 a 中的打包的 32 位整数取反，并将结果存储在 dst 中。
// 当 b 中对应的元素为 0 时，dst 中的元素被清零。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sign_pi32
FORCE_INLINE __m64 _mm_sign_pi32(__m64 _a, __m64 _b)
{
    // 将 _a 和 _b 转换为 32 位整数向量
    int32x2_t a = vreinterpret_s32_m64(_a);
    // 将_m64类型的变量_b重新解释为int32x2_t类型的变量b
    int32x2_t b = vreinterpret_s32_m64(_b);

    // 使用有符号右移操作进行比较，比vclt更快
    // 如果b < 0，则ltMask的值为0xFFFFFFFF，否则为0
    uint32x2_t ltMask = vreinterpret_u32_s32(vshr_n_s32(b, 31));

    // 如果b == 0，则ltMask的值为0xFFFFFFFF，否则为0
#if defined(__aarch64__) || defined(_M_ARM64)
    #ifdef __aarch64__
        // 如果是 ARM64 架构，使用 vceqz_s32 函数判断 b 是否全为 0，并将结果转换为 int32x2_t 类型的向量
        int32x2_t zeroMask = vreinterpret_s32_u32(vceqz_s32(b));
    #else
        // 如果是 ARM 架构，使用 vceq_s32 函数判断 b 是否等于 vdup_n_s32(0)，并将结果转换为 int32x2_t 类型的向量
        int32x2_t zeroMask = vreinterpret_s32_u32(vceq_s32(b, vdup_n_s32(0)));
    #endif
#else
    // 如果不是 ARM64 或 ARM 架构，使用 vceq_s32 函数判断 b 是否等于 vdup_n_s32(0)，并将结果转换为 int32x2_t 类型的向量
    int32x2_t zeroMask = vreinterpret_s32_u32(vceq_s32(b, vdup_n_s32(0)));
#endif

// 根据 ltMask 的值，选择 a 或者 -a，并将结果存储在 masked 中
int32x2_t masked = vbsl_s32(ltMask, vneg_s32(a), a);
// 将 masked 和 ~zeroMask 进行按位与操作，并将结果存储在 res 中
int32x2_t res = vbic_s32(masked, zeroMask);

// 将 res 转换为 __m64 类型的向量，并返回
return vreinterpret_m64_s32(res);
}

// 当 b 中对应的有符号 8 位整数为负数时，对 a 中的打包的 8 位整数取反，并将结果存储在 dst 中。
// 当 b 中对应的元素为 0 时，将 dst 中对应的元素置为 0。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_sign_pi8
FORCE_INLINE __m64 _mm_sign_pi8(__m64 _a, __m64 _b)
{
    // 将 __m64 类型的向量 _a 转换为 int8x8_t 类型的向量 a
    int8x8_t a = vreinterpret_s8_m64(_a);
    // 将 __m64 类型的向量 _b 转换为 int8x8_t 类型的向量 b
    int8x8_t b = vreinterpret_s8_m64(_b);

    // 使用有符号右移操作符 vshr_n_s8 进行有符号右移，判断 b 是否小于 0，并将结果转换为 uint8x8_t 类型的向量 ltMask
    // (b < 0) ? 0xFF : 0
    uint8x8_t ltMask = vreinterpret_u8_s8(vshr_n_s8(b, 7));

    // 使用 vceq_s8 函数判断 b 是否等于 vdup_n_s8(0)，并将结果转换为 int8x8_t 类型的向量 zeroMask
    // (b == 0) ? 0xFF : 0
#if defined(__aarch64__) || defined(_M_ARM64)
    int8x8_t zeroMask = vreinterpret_s8_u8(vceqz_s8(b));
#else
    int8x8_t zeroMask = vreinterpret_s8_u8(vceq_s8(b, vdup_n_s8(0)));
#endif

    // 根据 ltMask 的值，选择 a 或者 -a，并将结果存储在 masked 中
    int8x8_t masked = vbsl_s8(ltMask, vneg_s8(a), a);
    // 将 masked 和 ~zeroMask 进行按位与操作，并将结果存储在 res 中
    int8x8_t res = vbic_s8(masked, zeroMask);

    // 将 res 转换为 __m64 类型的向量，并返回
    return vreinterpret_m64_s8(res);
}

/* SSE4.1 */

// 使用控制掩码 imm8，从 a 和 b 中混合打包的 16 位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_blend_epi16
// FORCE_INLINE __m128i _mm_blend_epi16(__m128i a, __m128i b,
//                                      __constrange(0,255) int imm)
#define _mm_blend_epi16(a, b, imm)                                      \
    # 定义一个宏函数，用于将__m128i类型的a和b进行按位选择操作，并返回结果
    _sse2neon_define2(
        __m128i, a, b, 
        # 定义一个uint16_t类型的数组_mask，根据imm的值初始化数组元素
        const uint16_t _mask[8] = 
            _sse2neon_init(((imm) & (1 << 0)) ? (uint16_t) -1 : 0x0,
                           ((imm) & (1 << 1)) ? (uint16_t) -1 : 0x0,
                           ((imm) & (1 << 2)) ? (uint16_t) -1 : 0x0,
                           ((imm) & (1 << 3)) ? (uint16_t) -1 : 0x0,
                           ((imm) & (1 << 4)) ? (uint16_t) -1 : 0x0,
                           ((imm) & (1 << 5)) ? (uint16_t) -1 : 0x0,
                           ((imm) & (1 << 6)) ? (uint16_t) -1 : 0x0,
                           ((imm) & (1 << 7)) ? (uint16_t) -1 : 0x0);
        # 将_mask数组加载到uint16x8_t类型的_mask_vec变量中
        uint16x8_t _mask_vec = vld1q_u16(_mask);
        # 将__m128i类型的_a转换为uint16x8_t类型的__a
        uint16x8_t __a = vreinterpretq_u16_m128i(_a);
        # 将__m128i类型的_b转换为uint16x8_t类型的__b
        uint16x8_t __b = vreinterpretq_u16_m128i(_b);
        # 将__a和__b根据_mask_vec进行按位选择操作，并将结果转换为__m128i类型
        _sse2neon_return(vreinterpretq_m128i_u16(vbslq_u16(_mask_vec, __b, __a)));
    )
// 使用掩码 imm8，将 a 和 b 中的打包的双精度（64位）浮点数元素进行混合，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_blend_pd
#define _mm_blend_pd(a, b, imm)                                              \
    _sse2neon_define2(                                                       \
        __m128d, a, b,                                                       \
        const uint64_t _mask[2] =                                            \
            _sse2neon_init(((imm) & (1 << 0)) ? ~UINT64_C(0) : UINT64_C(0),  \
                           ((imm) & (1 << 1)) ? ~UINT64_C(0) : UINT64_C(0)); \
        uint64x2_t _mask_vec = vld1q_u64(_mask);                             \
        uint64x2_t __a = vreinterpretq_u64_m128d(_a);                        \
        uint64x2_t __b = vreinterpretq_u64_m128d(_b); _sse2neon_return(      \
            vreinterpretq_m128d_u64(vbslq_u64(_mask_vec, __b, __a)));)

// 使用掩码 imm8，将 a 和 b 中的打包的单精度（32位）浮点数元素进行混合，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_blend_ps
FORCE_INLINE __m128 _mm_blend_ps(__m128 _a, __m128 _b, const char imm8)
{
    const uint32_t ALIGN_STRUCT(16)
        data[4] = {((imm8) & (1 << 0)) ? UINT32_MAX : 0,
                   ((imm8) & (1 << 1)) ? UINT32_MAX : 0,
                   ((imm8) & (1 << 2)) ? UINT32_MAX : 0,
                   ((imm8) & (1 << 3)) ? UINT32_MAX : 0};
    uint32x4_t mask = vld1q_u32(data);
    float32x4_t a = vreinterpretq_f32_m128(_a);
    float32x4_t b = vreinterpretq_f32_m128(_b);
    return vreinterpretq_m128_f32(vbslq_f32(mask, b, a));
}

// 使用掩码混合 a 和 b 中的打包的8位整数，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_blendv_epi8
# 使用掩码将 a 和 b 中的打包的双精度（64位）浮点数元素混合，并将结果存储在 dst 中。
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_blendv_pd
FORCE_INLINE __m128d _mm_blendv_pd(__m128d _a, __m128d _b, __m128d _mask)
{
    # 使用有符号右移创建一个带有符号位的掩码
    uint64x2_t mask =
        vreinterpretq_u64_s64(vshrq_n_s64(vreinterpretq_s64_m128d(_mask), 63));
    # 将 _a 转换为 uint64x2_t 类型
    uint64x2_t a = vreinterpretq_u64_m128d(_a);
    # 将 _b 转换为 uint64x2_t 类型
    uint64x2_t b = vreinterpretq_u64_m128d(_b);
    # 使用掩码将 b 和 a 混合，并将结果转换为 __m128d 类型
    return vreinterpretq_m128d_u64(vbslq_u64(mask, b, a));
}

# 使用掩码将 a 和 b 中的打包的单精度（32位）浮点数元素混合，并将结果存储在 dst 中。
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_blendv_ps
FORCE_INLINE __m128 _mm_blendv_ps(__m128 _a, __m128 _b, __m128 _mask)
{
    # 使用有符号右移创建一个带有符号位的掩码
    uint32x4_t mask =
        vreinterpretq_u32_s32(vshrq_n_s32(vreinterpretq_s32_m128(_mask), 31));
    # 将 _a 转换为 float32x4_t 类型
    float32x4_t a = vreinterpretq_f32_m128(_a);
    # 将 _b 转换为 float32x4_t 类型
    float32x4_t b = vreinterpretq_f32_m128(_b);
    # 使用掩码将 b 和 a 混合，并将结果转换为 __m128 类型
    return vreinterpretq_m128_f32(vbslq_f32(mask, b, a));
}
// 向上取整，将 packed double-precision (64-bit) 浮点数元素 a 的每个元素都取整到最接近的整数值，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_ceil_pd
FORCE_INLINE __m128d _mm_ceil_pd(__m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，使用 ARM64 指令实现向上取整
    return vreinterpretq_m128d_f64(vrndpq_f64(vreinterpretq_f64_m128d(a)));
#else
    // 否则，将 a 转换为 double 数组，然后对数组中的每个元素进行向上取整，并将结果存储在 dst 中
    double *f = (double *) &a;
    return _mm_set_pd(ceil(f[1]), ceil(f[0]));
#endif
}

// 向上取整，将 packed single-precision (32-bit) 浮点数元素 a 的每个元素都取整到最接近的整数值，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_ceil_ps
FORCE_INLINE __m128 _mm_ceil_ps(__m128 a)
{
#if (defined(__aarch64__) || defined(_M_ARM64)) || \
    defined(__ARM_FEATURE_DIRECTED_ROUNDING)
    // 如果是 ARM64 架构或支持指定舍入模式的 ARM 架构，使用 ARM 指令实现向上取整
    return vreinterpretq_m128_f32(vrndpq_f32(vreinterpretq_f32_m128(a)));
#else
    // 否则，将 a 转换为 float 数组，然后对数组中的每个元素进行向上取整，并将结果存储在 dst 中
    float *f = (float *) &a;
    return _mm_set_ps(ceilf(f[3]), ceilf(f[2]), ceilf(f[1]), ceilf(f[0]));
#endif
}

// 向上取整，将 packed double-precision (64-bit) 浮点数元素 b 的低位元素取整到最接近的整数值，并将结果存储在 dst 的低位元素中，然后将 a 的高位元素复制到 dst 的高位元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_ceil_sd
FORCE_INLINE __m128d _mm_ceil_sd(__m128d a, __m128d b)
{
    // 调用 _mm_ceil_pd 函数对 b 进行向上取整，然后将结果与 a 进行合并
    return _mm_move_sd(a, _mm_ceil_pd(b));
}

// 向上取整，将 packed single-precision (32-bit) 浮点数元素 b 的低位元素取整到最接近的整数值，并将结果存储在 dst 的低位元素中，然后将 a 的高位元素复制到 dst 的高位元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_ceil_ss
FORCE_INLINE __m128 _mm_ceil_ss(__m128 a, __m128 b)
{
    // 调用 _mm_ceil_ps 函数对 b 进行向上取整，然后将结果与 a 进行合并
    return _mm_move_ss(a, _mm_ceil_ps(b));
}

// 比较 packed 64-bit 整数元素 a 和 b 的相等性，并将结果存储在 dst 中
// 定义一个函数，用于比较两个__m128i类型的参数是否相等
FORCE_INLINE __m128i _mm_cmpeq_epi64(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是 ARM64 架构，则使用vceqq_u64函数比较两个__m128i类型的参数是否相等
    return vreinterpretq_m128i_u64(
        vceqq_u64(vreinterpretq_u64_m128i(a), vreinterpretq_u64_m128i(b)));
#else
    // 如果不是 ARM64 架构，则执行以下代码
    // ARMv7架构缺少vceqq_u64函数
    // (a == b) -> (a_lo == b_lo) && (a_hi == b_hi)
    uint32x4_t cmp =
        vceqq_u32(vreinterpretq_u32_m128i(a), vreinterpretq_u32_m128i(b));
    uint32x4_t swapped = vrev64q_u32(cmp);
    return vreinterpretq_m128i_u32(vandq_u32(cmp, swapped));
#endif
}

// 将打包的16位整数符号扩展为32位整数，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepi16_epi32
FORCE_INLINE __m128i _mm_cvtepi16_epi32(__m128i a)
{
    return vreinterpretq_m128i_s32(
        vmovl_s16(vget_low_s16(vreinterpretq_s16_m128i(a))));
}

// 将打包的16位整数符号扩展为64位整数，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepi16_epi64
FORCE_INLINE __m128i _mm_cvtepi16_epi64(__m128i a)
{
    int16x8_t s16x8 = vreinterpretq_s16_m128i(a);     /* xxxx xxxx xxxx 0B0A */
    int32x4_t s32x4 = vmovl_s16(vget_low_s16(s16x8)); /* 000x 000x 000B 000A */
    int64x2_t s64x2 = vmovl_s32(vget_low_s32(s32x4)); /* 0000 000B 0000 000A */
    return vreinterpretq_m128i_s64(s64x2);
}

// 将打包的32位整数符号扩展为64位整数，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepi32_epi64
FORCE_INLINE __m128i _mm_cvtepi32_epi64(__m128i a)
{
    return vreinterpretq_m128i_s64(
        vmovl_s32(vget_low_s32(vreinterpretq_s32_m128i(a))));
}

// 将打包的8位整数符号扩展为16位整数，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepi8_epi16
# 将 packed 8-bit 整数 a 的符号扩展为 packed 16-bit 整数，并将结果存储在 dst 中
FORCE_INLINE __m128i _mm_cvtepi8_epi16(__m128i a):
    # 将 a 转换为 int8x16_t 类型的变量 s8x16，其中每个元素都是一个 8-bit 整数
    int8x16_t s8x16 = vreinterpretq_s8_m128i(a);    /* xxxx xxxx xxxx DCBA */
    # 将 s8x16 的低 8 个元素转换为 int16x8_t 类型的变量 s16x8，其中每个元素都是一个 16-bit 整数
    int16x8_t s16x8 = vmovl_s8(vget_low_s8(s8x16)); /* 0x0x 0x0x 0D0C 0B0A */
    # 将 s16x8 转换为 __m128i 类型的变量，并返回结果
    return vreinterpretq_m128i_s16(s16x8);

# 将 packed 8-bit 整数 a 的符号扩展为 packed 32-bit 整数，并将结果存储在 dst 中
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepi8_epi32
FORCE_INLINE __m128i _mm_cvtepi8_epi32(__m128i a):
    # 将 a 转换为 int8x16_t 类型的变量 s8x16，其中每个元素都是一个 8-bit 整数
    int8x16_t s8x16 = vreinterpretq_s8_m128i(a);      /* xxxx xxxx xxxx DCBA */
    # 将 s8x16 的低 8 个元素转换为 int16x8_t 类型的变量 s16x8，其中每个元素都是一个 16-bit 整数
    int16x8_t s16x8 = vmovl_s8(vget_low_s8(s8x16));   /* 0x0x 0x0x 0D0C 0B0A */
    # 将 s16x8 的低 4 个元素转换为 int32x4_t 类型的变量 s32x4，其中每个元素都是一个 32-bit 整数
    int32x4_t s32x4 = vmovl_s16(vget_low_s16(s16x8)); /* 000D 000C 000B 000A */
    # 将 s32x4 转换为 __m128i 类型的变量，并返回结果
    return vreinterpretq_m128i_s32(s32x4);

# 将 packed 8-bit 整数 a 的符号扩展为 packed 64-bit 整数，并将结果存储在 dst 中
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepi8_epi64
FORCE_INLINE __m128i _mm_cvtepi8_epi64(__m128i a):
    # 将 a 转换为 int8x16_t 类型的变量 s8x16，其中每个元素都是一个 8-bit 整数
    int8x16_t s8x16 = vreinterpretq_s8_m128i(a);      /* xxxx xxxx xxxx xxBA */
    # 将 s8x16 的低 8 个元素转换为 int16x8_t 类型的变量 s16x8，其中每个元素都是一个 16-bit 整数
    int16x8_t s16x8 = vmovl_s8(vget_low_s8(s8x16));   /* 0x0x 0x0x 0x0x 0B0A */
    # 将 s16x8 的低 4 个元素转换为 int32x4_t 类型的变量 s32x4，其中每个元素都是一个 32-bit 整数
    int32x4_t s32x4 = vmovl_s16(vget_low_s16(s16x8)); /* 000x 000x 000B 000A */
    # 将 s32x4 的低 2 个元素转换为 int64x2_t 类型的变量 s64x2，其中每个元素都是一个 64-bit 整数
    int64x2_t s64x2 = vmovl_s32(vget_low_s32(s32x4)); /* 0000 000B 0000 000A */
    # 将 s64x2 转换为 __m128i 类型的变量，并返回结果
    return vreinterpretq_m128i_s64(s64x2);

# 将 packed 16-bit 无符号整数 a 的零扩展为 packed 32-bit 整数，并将结果存储在 dst 中
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cvtepu16_epi32
FORCE_INLINE __m128i _mm_cvtepu16_epi32(__m128i a):
    # 将 a 转换为 uint16x8_t 类型的变量 s16x8，其中每个元素都是一个 16-bit 无符号整数
    uint16x8_t s16x8 = vreinterpretq_u16_m128i(a);
    # 将 s16x8 的低 8 个元素转换为 uint32x4_t 类型的变量 s32x4，其中每个元素都是一个 32-bit 无符号整数
    uint32x4_t s32x4 = vmovl_u16(vget_low_u16(s16x8));
    # 将 s32x4 转换为 __m128i 类型的变量，并返回结果
    return vreinterpretq_m128i_u32(s32x4);
// 将 packed unsigned 16-bit integers a 转换为 packed 64-bit integers，结果存储在 dst 中
FORCE_INLINE __m128i _mm_cvtepu16_epi64(__m128i a)
{
    // 将 a 转换为 uint16x8_t 类型的变量 u16x8，即将 a 的低 8 个字节转换为 16 位无符号整数
    uint16x8_t u16x8 = vreinterpretq_u16_m128i(a);     /* xxxx xxxx xxxx 0B0A */
    // 将 u16x8 的低 4 个字节转换为 32 位无符号整数，并存储在 u32x4 中
    uint32x4_t u32x4 = vmovl_u16(vget_low_u16(u16x8)); /* 000x 000x 000B 000A */
    // 将 u32x4 的低 2 个字节转换为 64 位无符号整数，并存储在 u64x2 中
    uint64x2_t u64x2 = vmovl_u32(vget_low_u32(u32x4)); /* 0000 000B 0000 000A */
    // 将 u64x2 转换为 __m128i 类型的变量，并返回
    return vreinterpretq_m128i_u64(u64x2);
}

// 将 packed unsigned 32-bit integers a 转换为 packed 64-bit integers，结果存储在 dst 中
FORCE_INLINE __m128i _mm_cvtepu32_epi64(__m128i a)
{
    // 将 a 转换为 uint32x4_t 类型的变量 u32x4，即将 a 的低 4 个字节转换为 32 位无符号整数
    return vreinterpretq_m128i_u64(
        vmovl_u32(vget_low_u32(vreinterpretq_u32_m128i(a))));
}

// 将 packed unsigned 8-bit integers a 转换为 packed 16-bit integers，结果存储在 dst 中
FORCE_INLINE __m128i _mm_cvtepu8_epi16(__m128i a)
{
    // 将 a 转换为 uint8x16_t 类型的变量 u8x16，即将 a 的低 16 个字节转换为 8 位无符号整数
    uint8x16_t u8x16 = vreinterpretq_u8_m128i(a);    /* xxxx xxxx HGFE DCBA */
    // 将 u8x16 的低 8 个字节转换为 16 位无符号整数，并存储在 u16x8 中
    uint16x8_t u16x8 = vmovl_u8(vget_low_u8(u8x16)); /* 0H0G 0F0E 0D0C 0B0A */
    // 将 u16x8 转换为 __m128i 类型的变量，并返回
    return vreinterpretq_m128i_u16(u16x8);
}

// 将 packed unsigned 8-bit integers a 转换为 packed 32-bit integers，结果存储在 dst 中
FORCE_INLINE __m128i _mm_cvtepu8_epi32(__m128i a)
{
    // 将 a 转换为 uint8x16_t 类型的变量 u8x16，即将 a 的低 16 个字节转换为 8 位无符号整数
    uint8x16_t u8x16 = vreinterpretq_u8_m128i(a);      /* xxxx xxxx xxxx DCBA */
    // 将 u8x16 的低 8 个字节转换为 16 位无符号整数，并存储在 u16x8 中
    uint16x8_t u16x8 = vmovl_u8(vget_low_u8(u8x16));   /* 0x0x 0x0x 0D0C 0B0A */
    // 将 u16x8 的低 4 个字节转换为 32 位无符号整数，并存储在 u32x4 中
    uint32x4_t u32x4 = vmovl_u16(vget_low_u16(u16x8)); /* 000D 000C 000B 000A */
    // 将 u32x4 转换为 __m128i 类型的变量，并返回
    return vreinterpretq_m128i_u32(u32x4);
}

// 将 packed unsigned 8-bit integers a 的低 8 个字节转换为 packed 64-bit integers，结果存储在 dst 中
// 将__m128i类型的参数a转换为uint8x16_t类型的变量u8x16，即将a的每个字节拆分为16个8位无符号整数
uint8x16_t u8x16 = vreinterpretq_u8_m128i(a);      /* xxxx xxxx xxxx xxBA */
// 将u8x16的低8位拆分为8个16位无符号整数，并将结果存储在u16x8中
uint16x8_t u16x8 = vmovl_u8(vget_low_u8(u8x16));   /* 0x0x 0x0x 0x0x 0B0A */
// 将u16x8的低16位拆分为4个32位无符号整数，并将结果存储在u32x4中
uint32x4_t u32x4 = vmovl_u16(vget_low_u16(u16x8)); /* 000x 000x 000B 000A */
// 将u32x4的低32位拆分为2个64位无符号整数，并将结果存储在u64x2中
uint64x2_t u64x2 = vmovl_u32(vget_low_u32(u32x4)); /* 0000 000B 0000 000A */
// 将u64x2转换为__m128i类型，并返回结果
return vreinterpretq_m128i_u64(u64x2);



// 根据imm参数的低4位生成掩码值
const int64_t bit0Mask = imm & 0x01 ? UINT64_MAX : 0;
const int64_t bit1Mask = imm & 0x02 ? UINT64_MAX : 0;
#if !SSE2NEON_PRECISE_DP
const int64_t bit4Mask = imm & 0x10 ? UINT64_MAX : 0;
const int64_t bit5Mask = imm & 0x20 ? UINT64_MAX : 0;
#endif
// 条件乘法
#if !SSE2NEON_PRECISE_DP
// 将a和b进行乘法运算，并将结果存储在mul中
__m128d mul = _mm_mul_pd(a, b);
// 将bit4Mask和bit5Mask转换为__m128d类型的掩码，并与mul进行按位与运算，将结果存储在tmp中
const __m128d mulMask =
    _mm_castsi128_pd(_mm_set_epi64x(bit5Mask, bit4Mask));
__m128d tmp = _mm_and_pd(mul, mulMask);
#else
#if defined(__aarch64__) || defined(_M_ARM64)
// 如果是ARM64架构，根据imm的第4位和第5位判断是否进行乘法运算，并将结果存储在d0和d1中
double d0 = (imm & 0x10) ? vgetq_lane_f64(vreinterpretq_f64_m128d(a), 0) *
                               vgetq_lane_f64(vreinterpretq_f64_m128d(b), 0)
                         : 0;
double d1 = (imm & 0x20) ? vgetq_lane_f64(vreinterpretq_f64_m128d(a), 1) *
                               vgetq_lane_f64(vreinterpretq_f64_m128d(b), 1)
                         : 0;
#else
// 如果不是ARM64架构，根据imm的第4位判断是否进行乘法运算，并将结果存储在d0中
double d0 = (imm & 0x10) ? ((double *) &a)[0] * ((double *) &b)[0] : 0;
    # 如果 imm & 0x20 为真，则计算 ((double *) &a)[1] * ((double *) &b)[1]，否则结果为 0
    double d1 = (imm & 0x20) ? ((double *) &a)[1] * ((double *) &b)[1] : 0;
#endif
    # 创建一个包含两个双精度浮点数的向量，值为d1和d0
    __m128d tmp = _mm_set_pd(d1, d0);
#endif
    # 求两个双精度浮点数向量的乘积之和
#if defined(__aarch64__) || defined(_M_ARM64)
    # 使用vpaddd_f64函数对tmp向量进行求和
    double sum = vpaddd_f64(vreinterpretq_f64_m128d(tmp));
#else
    # 将tmp向量转换为两个双精度浮点数指针，然后将两个双精度浮点数相加得到sum
    double sum = *((double *) &tmp) + *(((double *) &tmp) + 1);
#endif
    # 条件性地存储sum的值
    # 创建一个掩码，用于选择要存储的位
    const __m128d sumMask =
        _mm_castsi128_pd(_mm_set_epi64x(bit1Mask, bit0Mask));
    # 将sum的值存储到res向量中，并根据sumMask选择要存储的位
    __m128d res = _mm_and_pd(_mm_set_pd1(sum), sumMask);
    # 返回结果向量
    return res;
}

# 条件性地将两个单精度浮点数向量a和b的元素进行乘法运算，并使用imm的低4位条件性地存储四个乘积之和到dst中
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_dp_ps
FORCE_INLINE __m128 _mm_dp_ps(__m128 a, __m128 b, const int imm)
{
    # 对a和b的对应元素进行乘法运算
    float32x4_t elementwise_prod = _mm_mul_ps(a, b);

#if defined(__aarch64__) || defined(_M_ARM64)
    # 快捷方式
    if (imm == 0xFF) {
        # 对elementwise_prod向量的所有元素求和，并将结果复制到一个新的向量中
        return _mm_set1_ps(vaddvq_f32(elementwise_prod));
    }

    if ((imm & 0x0F) == 0x0F) {
        # 根据imm的位选择要置零的元素，并将elementwise_prod向量的所有元素求和，并将结果复制到一个新的向量中
        if (!(imm & (1 << 4)))
            elementwise_prod = vsetq_lane_f32(0.0f, elementwise_prod, 0);
        if (!(imm & (1 << 5)))
            elementwise_prod = vsetq_lane_f32(0.0f, elementwise_prod, 1);
        if (!(imm & (1 << 6)))
            elementwise_prod = vsetq_lane_f32(0.0f, elementwise_prod, 2);
        if (!(imm & (1 << 7)))
            elementwise_prod = vsetq_lane_f32(0.0f, elementwise_prod, 3);

        return _mm_set1_ps(vaddvq_f32(elementwise_prod));
    }
#endif

    # 初始化s为0.0
    float s = 0.0f;

    # 根据imm的位选择要累加的元素，并将其累加到s中
    if (imm & (1 << 4))
        s += vgetq_lane_f32(elementwise_prod, 0);
    if (imm & (1 << 5))
        s += vgetq_lane_f32(elementwise_prod, 1);
    if (imm & (1 << 6))
        s += vgetq_lane_f32(elementwise_prod, 2);
    if (imm & (1 << 7))
        s += vgetq_lane_f32(elementwise_prod, 3);
    // 创建一个包含4个元素的浮点数数组，根据imm的不同位进行条件判断，如果满足条件则赋值为s，否则赋值为0.0f
    const float32_t res[4] = {
        (imm & 0x1) ? s : 0.0f,
        (imm & 0x2) ? s : 0.0f,
        (imm & 0x4) ? s : 0.0f,
        (imm & 0x8) ? s : 0.0f,
    };
    // 将res数组中的数据加载到一个128位的浮点数向量中，并返回结果
    return vreinterpretq_m128_f32(vld1q_f32(res));
// 从a中提取一个32位整数，使用imm8选择，将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_extract_epi32
#define _mm_extract_epi32(a, imm) \
    vgetq_lane_s32(vreinterpretq_s32_m128i(a), (imm))

// 从a中提取一个64位整数，使用imm8选择，将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_extract_epi64
#define _mm_extract_epi64(a, imm) \
    vgetq_lane_s64(vreinterpretq_s64_m128i(a), (imm))

// 从a中提取一个8位整数，使用imm8选择，将结果存储在dst的低位元素中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_extract_epi8
#define _mm_extract_epi8(a, imm) vgetq_lane_u8(vreinterpretq_u8_m128i(a), (imm))

// 从a中提取所选的单精度（32位）浮点数
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_extract_ps
#define _mm_extract_ps(a, imm) vgetq_lane_s32(vreinterpretq_s32_m128(a), (imm))

// 将a中的打包双精度（64位）浮点数元素向下舍入为整数值，并将结果作为打包双精度浮点数元素存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_floor_pd
FORCE_INLINE __m128d _mm_floor_pd(__m128d a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128d_f64(vrndmq_f64(vreinterpretq_f64_m128d(a)));
#else
    double *f = (double *) &a;
    return _mm_set_pd(floor(f[1]), floor(f[0]));
#endif
}

// 将a中的打包单精度（32位）浮点数元素向下舍入为整数值，并将结果作为打包单精度浮点数元素存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_floor_ps
FORCE_INLINE __m128 _mm_floor_ps(__m128 a)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    return vreinterpretq_m128_f32(vrndmq_f32(vreinterpretq_f32_m128(a)));
#else
    float *f = (float *) &a;
    return _mm_set_ps(floor(f[3]), floor(f[2]), floor(f[1]), floor(f[0]));
#endif
}
// 将参数 a 中的每个单精度浮点数值向下取整，并将结果作为打包的单精度浮点数元素存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_floor_ps
FORCE_INLINE __m128 _mm_floor_ps(__m128 a)
{
#if (defined(__aarch64__) || defined(_M_ARM64)) || \
    defined(__ARM_FEATURE_DIRECTED_ROUNDING)
    // 如果是 ARM64 架构或者支持 ARM 指令集的平台，使用 ARM 指令实现向下取整
    return vreinterpretq_m128_f32(vrndmq_f32(vreinterpretq_f32_m128(a)));
#else
    // 否则，将参数 a 转换为 float 指针，然后分别向下取整并存储在 dst 中
    float *f = (float *) &a;
    return _mm_set_ps(floorf(f[3]), floorf(f[2]), floorf(f[1]), floorf(f[0]));
#endif
}

// 将参数 b 中的低位双精度（64位）浮点数向下取整为整数值，将结果作为双精度浮点数元素存储在 dst 的低位元素中，并将参数 a 的高位元素复制到 dst 的高位元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_floor_sd
FORCE_INLINE __m128d _mm_floor_sd(__m128d a, __m128d b)
{
    // 将参数 a 移动到 dst，同时将参数 b 向下取整并存储在 dst 的低位元素中
    return _mm_move_sd(a, _mm_floor_pd(b));
}

// 将参数 b 中的低位单精度（32位）浮点数向下取整为整数值，将结果作为单精度浮点数元素存储在 dst 的低位元素中，并将参数 a 的高位 3 个打包元素复制到 dst 的高位元素中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_floor_ss
FORCE_INLINE __m128 _mm_floor_ss(__m128 a, __m128 b)
{
    // 将参数 a 移动到 dst，同时将参数 b 向下取整并存储在 dst 的低位元素中
    return _mm_move_ss(a, _mm_floor_ps(b));
}

// 将参数 a 复制到 dst，并在 dst 中由 imm8 指定的位置插入 32 位整数 i。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_insert_epi32
// FORCE_INLINE __m128i _mm_insert_epi32(__m128i a, int b,
//                                       __constrange(0,4) int imm)
#define _mm_insert_epi32(a, b, imm) \
    vreinterpretq_m128i_s32(        \
        vsetq_lane_s32((b), vreinterpretq_s32_m128i(a), (imm)))
// 将 a 复制到 dst，并在 dst 的指定位置 imm8 处插入 64 位整数 i
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_insert_epi64
// FORCE_INLINE __m128i _mm_insert_epi64(__m128i a, __int64 b,
//                                       __constrange(0,2) int imm)
#define _mm_insert_epi64(a, b, imm) \
    vreinterpretq_m128i_s64(        \
        vsetq_lane_s64((b), vreinterpretq_s64_m128i(a), (imm)))

// 将 a 复制到 dst，并在 dst 的指定位置 imm8 处插入 8 位整数 i 的低 8 位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_insert_epi8
// FORCE_INLINE __m128i _mm_insert_epi8(__m128i a, int b,
//                                      __constrange(0,16) int imm)
#define _mm_insert_epi8(a, b, imm) \
    vreinterpretq_m128i_s8(vsetq_lane_s8((b), vreinterpretq_s8_m128i(a), (imm)))

// 将 a 复制到 tmp，然后使用 imm8 中的控制将 b 中的单精度（32 位）浮点数元素插入 tmp。
// 使用 imm8 中的掩码将 tmp 存储到 dst（当相应位设置时，元素被清零）。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=insert_ps
#define _mm_insert_ps(a, b, imm8)                                            \
    # 定义一个宏，将__m128类型的a和b参数转换为float32x4_t类型的tmp1
    float32x4_t tmp1 = vsetq_lane_f32(vgetq_lane_f32(_b, (imm8 >> 6) & 0x3), vreinterpretq_f32_m128(_a), 0);
    # 定义一个宏，将tmp1中指定位置的元素转换为float32x4_t类型的tmp2
    float32x4_t tmp2 = vsetq_lane_f32(vgetq_lane_f32(tmp1, 0), vreinterpretq_f32_m128(_a), ((imm8 >> 4) & 0x3));
    # 初始化一个包含4个元素的uint32_t数组
    const uint32_t data[4] = _sse2neon_init(((imm8) & (1 << 0)) ? UINT32_MAX : 0, ((imm8) & (1 << 1)) ? UINT32_MAX : 0, ((imm8) & (1 << 2)) ? UINT32_MAX : 0, ((imm8) & (1 << 3)) ? UINT32_MAX : 0);
    # 将数组中的数据加载到uint32x4_t类型的mask中
    uint32x4_t mask = vld1q_u32(data);
    # 创建一个所有元素都为0的float32x4_t类型的向量
    float32x4_t all_zeros = vdupq_n_f32(0);
    # 返回根据条件选择的结果
    _sse2neon_return(vreinterpretq_m128_f32(vbslq_f32(mask, all_zeros, vreinterpretq_f32_m128(tmp2)));
// 比较 a 和 b 中打包的有符号32位整数，并将最大值打包存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_epi32
FORCE_INLINE __m128i _mm_max_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s32(
        vmaxq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 比较 a 和 b 中打包的有符号8位整数，并将最大值打包存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_epi8
FORCE_INLINE __m128i _mm_max_epi8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s8(
        vmaxq_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
}

// 比较 a 和 b 中打包的无符号16位整数，并将最大值打包存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_epu16
FORCE_INLINE __m128i _mm_max_epu16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u16(
        vmaxq_u16(vreinterpretq_u16_m128i(a), vreinterpretq_u16_m128i(b)));
}

// 比较 a 和 b 中打包的无符号32位整数，并将最大值打包存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_max_epu32
FORCE_INLINE __m128i _mm_max_epu32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u32(
        vmaxq_u32(vreinterpretq_u32_m128i(a), vreinterpretq_u32_m128i(b)));
}

// 比较 a 和 b 中打包的有符号32位整数，并将最小值打包存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_min_epi32
FORCE_INLINE __m128i _mm_min_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s32(
        vminq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 比较 a 和 b 中打包的有符号8位整数，并将最小值打包存储在 dst 中。
// 使用 SIMD 指令集实现对两个 __m128i 类型的参数进行有符号 8 位整数比较，并返回最小值
FORCE_INLINE __m128i _mm_min_epi8(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s8(
        vminq_s8(vreinterpretq_s8_m128i(a), vreinterpretq_s8_m128i(b)));
}

// 使用 SIMD 指令集实现对两个 __m128i 类型的参数进行无符号 16 位整数比较，并返回最小值
FORCE_INLINE __m128i _mm_min_epu16(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u16(
        vminq_u16(vreinterpretq_u16_m128i(a), vreinterpretq_u16_m128i(b)));
}

// 使用 SIMD 指令集实现对两个 __m128i 类型的参数进行无符号 32 位整数比较，并返回最小值
FORCE_INLINE __m128i _mm_min_epu32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u32(
        vminq_u32(vreinterpretq_u32_m128i(a), vreinterpretq_u32_m128i(b)));
}

// 使用 SIMD 指令集实现对 __m128i 类型的参数进行无符号 16 位整数比较，并返回最小值及其索引
FORCE_INLINE __m128i _mm_minpos_epu16(__m128i a)
{
    __m128i dst;
    uint16_t min, idx = 0;
#if defined(__aarch64__) || defined(_M_ARM64)
    // 找到最小值
    min = vminvq_u16(vreinterpretq_u16_m128i(a));

    // 获取最小值的索引
    static const uint16_t idxv[] = {0, 1, 2, 3, 4, 5, 6, 7};
    uint16x8_t minv = vdupq_n_u16(min);
    uint16x8_t cmeq = vceqq_u16(minv, vreinterpretq_u16_m128i(a));
    idx = vminvq_u16(vornq_u16(vld1q_u16(idxv), cmeq));
#else
    // 找到最小值
    __m64 tmp;
    tmp = vreinterpret_m64_u16(
        vmin_u16(vget_low_u16(vreinterpretq_u16_m128i(a)),
                 vget_high_u16(vreinterpretq_u16_m128i(a))));
    # 将输入的 64 位整数向量中的每两个 16 位整数取最小值
    tmp = vreinterpret_m64_u16(
        vpmin_u16(vreinterpret_u16_m64(tmp), vreinterpret_u16_m64(tmp)));
    # 再次将输入的 64 位整数向量中的每两个 16 位整数取最小值
    tmp = vreinterpret_m64_u16(
        vpmin_u16(vreinterpret_u16_m64(tmp), vreinterpret_u16_m64(tmp)));
    # 从结果向量中获取最小值
    min = vget_lane_u16(vreinterpret_u16_m64(tmp), 0);
    # 获取最小值的索引
    int i;
    for (i = 0; i < 8; i++) {
        # 如果最小值等于向量中的某个值，则记录该值的索引并跳出循环
        if (min == vgetq_lane_u16(vreinterpretq_u16_m128i(a), 0)) {
            idx = (uint16_t) i;
            break;
        }
        # 向右移动 2 个字节，继续比较下一个值
        a = _mm_srli_si128(a, 2);
    }
#endif
    // 生成结果
    dst = _mm_setzero_si128();  // 将dst初始化为全零
    dst = vreinterpretq_m128i_u16(
        vsetq_lane_u16(min, vreinterpretq_u16_m128i(dst), 0));  // 将dst的第0个元素替换为min
    dst = vreinterpretq_m128i_u16(
        vsetq_lane_u16(idx, vreinterpretq_u16_m128i(dst), 1));  // 将dst的第1个元素替换为idx
    return dst;  // 返回结果dst
}

// 计算a中四个无符号8位整数与b中对应四个无符号8位整数的绝对差的和（SADs），并将16位结果存储在dst中。
// 使用b中的一个四元组和a中的八个四元组执行八个SADs。从imm8指定的偏移量开始，从a中选择连续的8位整数形成八个四元组。
// 从imm8指定的偏移量开始，从b中选择一个四元组。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mpsadbw_epu8
FORCE_INLINE __m128i _mm_mpsadbw_epu8(__m128i a, __m128i b, const int imm)
{
    uint8x16_t _a, _b;

    switch (imm & 0x4) {
    case 0:
        // 什么都不做
        _a = vreinterpretq_u8_m128i(a);  // 将a转换为uint8x16_t类型
        break;
    case 4:
        _a = vreinterpretq_u8_u32(vextq_u32(vreinterpretq_u32_m128i(a),
                                            vreinterpretq_u32_m128i(a), 1));  // 将a的元素循环右移1位，并转换为uint8x16_t类型
        break;
    default:
#if defined(__GNUC__) || defined(__clang__)
        __builtin_unreachable();  // 不可达代码，编译器优化
#elif defined(_MSC_VER)
        __assume(0);  // 不可达代码，编译器优化
#endif
        break;
    }

    switch (imm & 0x3) {
    case 0:
        _b = vreinterpretq_u8_u32(
            vdupq_n_u32(vgetq_lane_u32(vreinterpretq_u32_m128i(b), 0)));  // 将b的第0个元素复制为四元组
        break;
    case 1:
        _b = vreinterpretq_u8_u32(
            vdupq_n_u32(vgetq_lane_u32(vreinterpretq_u32_m128i(b), 1)));  // 将b的第1个元素复制为四元组
        break;
    case 2:
        _b = vreinterpretq_u8_u32(
            vdupq_n_u32(vgetq_lane_u32(vreinterpretq_u32_m128i(b), 2)));  // 将b的第2个元素复制为四元组
        break;
    case 3:
        _b = vreinterpretq_u8_u32(
            vdupq_n_u32(vgetq_lane_u32(vreinterpretq_u32_m128i(b), 3)));  // 将b的第3个元素复制为四元组
        break;
    default:
#if defined(__GNUC__) || defined(__clang__)
        __builtin_unreachable();
#elif defined(_MSC_VER)
        __assume(0);
#endif
        break;
    }

这段代码是一个条件语句，用于在编译器中指定某些代码永远不会被执行。如果编译器是GCC或Clang，则使用`__builtin_unreachable()`函数表示代码不可达。如果编译器是MSVC，则使用`__assume(0)`表示代码不可达。无论哪种情况，都会跳出循环。


    int16x8_t c04, c15, c26, c37;
    uint8x8_t low_b = vget_low_u8(_b);
    c04 = vreinterpretq_s16_u16(vabdl_u8(vget_low_u8(_a), low_b));
    uint8x16_t _a_1 = vextq_u8(_a, _a, 1);
    c15 = vreinterpretq_s16_u16(vabdl_u8(vget_low_u8(_a_1), low_b));
    uint8x16_t _a_2 = vextq_u8(_a, _a, 2);
    c26 = vreinterpretq_s16_u16(vabdl_u8(vget_low_u8(_a_2), low_b));
    uint8x16_t _a_3 = vextq_u8(_a, _a, 3);
    c37 = vreinterpretq_s16_u16(vabdl_u8(vget_low_u8(_a_3), low_b));

这段代码定义了四个`int16x8_t`类型的变量`c04`、`c15`、`c26`和`c37`，以及四个`uint8x16_t`类型的变量`_a_1`、`_a_2`和`_a_3`。它们用于存储一些向量运算的结果。其中，`vget_low_u8()`函数用于获取向量`_a`和`_b`的低8位元素，`vextq_u8()`函数用于将向量`_a`的元素按指定的偏移量进行重新排列，`vreinterpretq_s16_u16()`函数用于将`uint8x8_t`类型的向量转换为`int16x8_t`类型的向量，`vabdl_u8()`函数用于计算两个`uint8x8_t`类型的向量的绝对差值，并将结果扩展为`int16x8_t`类型的向量。


#if defined(__aarch64__) || defined(_M_ARM64)
    // |0|4|2|6|
    c04 = vpaddq_s16(c04, c26);
    // |1|5|3|7|
    c15 = vpaddq_s16(c15, c37);

    int32x4_t trn1_c =
        vtrn1q_s32(vreinterpretq_s32_s16(c04), vreinterpretq_s32_s16(c15));
    int32x4_t trn2_c =
        vtrn2q_s32(vreinterpretq_s32_s16(c04), vreinterpretq_s32_s16(c15));
    return vreinterpretq_m128i_s16(vpaddq_s16(vreinterpretq_s16_s32(trn1_c),
                                              vreinterpretq_s16_s32(trn2_c)));
#else
    int16x4_t c01, c23, c45, c67;
    c01 = vpadd_s16(vget_low_s16(c04), vget_low_s16(c15));
    c23 = vpadd_s16(vget_low_s16(c26), vget_low_s16(c37));
    c45 = vpadd_s16(vget_high_s16(c04), vget_high_s16(c15));
    c67 = vpadd_s16(vget_high_s16(c26), vget_high_s16(c37));

    return vreinterpretq_m128i_s16(
        vcombine_s16(vpadd_s16(c01, c23), vpadd_s16(c45, c67)));
#endif

这段代码是一个条件语句，根据编译器的类型选择不同的代码路径。如果编译器是ARM64架构的，则执行`#if`分支的代码。否则，执行`#else`分支的代码。

在`#if`分支中，首先对`c04`和`c26`进行垂直相加操作，将结果存储在`c04`中。然后对`c15`和`c37`进行垂直相加操作，将结果存储在`c15`中。接下来，使用`vtrn1q_s32()`和`vtrn2q_s32()`函数对`c04`和`c15`进行交错操作，将结果分别存储在`trn1_c`和`trn2_c`中。最后，使用`vpaddq_s16()`函数对`trn1_c`和`trn2_c`进行水平相加操作，并将结果转换为`__m128i`类型的向量返回。

在`#else`分支中，首先对`c04`和`c15`的低位元素进行水平相加操作，将结果存储在`c01`中。然后对`c26`和`c37`的低位元素进行水平相加操作，将结果存储在`c23`中。接下来，对`c04`和`c15`的高位元素进行水平相加操作，将结果存储在`c45`中。最后，对`c26`和`c37`的高位元素进行水平相加操作，将结果存储在`c67`中。最后，使用`vcombine_s16()`函数将`c01`和`c23`以及`c45`和`c67`进行组合，并将结果转换为`__m128i`类型的向量返回。


// Multiply the low signed 32-bit integers from each packed 64-bit element in
// a and b, and store the signed 64-bit results in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mul_epi32
FORCE_INLINE __m128i _mm_mul_epi32(__m128i a, __m128i b)
{
    // vmull_s32 upcasts instead of masking, so we downcast.
    int32x2_t a_lo = vmovn_s64(vreinterpretq_s64_m128i(a));
    int32x2_t b_lo = vmovn_s64(vreinterpretq_s64_m128i(b));

这段代码是一个函数的定义，用于将两个`__m128i`类型的向量`a`和`b`的低位有符号32位整数进行相乘，并将结果存储在`__m128i`类型的向量`dst`中。

首先，使用`vreinterpretq_s64_m128i()`函数将向量`a`和`b`转换为`int64x2_t`类型的向量。然后，使用`vmovn_s64()`函数将`int64x2_t`类型的向量转换为`int32x2_t`类型的向量，并将结果分别存储在`a_lo`和`b_lo`中。
    # 返回一个将两个32位整数向量a_lo和b_lo相乘的结果，结果是一个128位整数向量。
// 乘法操作，将两个 packed 32 位整数相乘，得到中间的 64 位整数，将低 32 位存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_mullo_epi32
FORCE_INLINE __m128i _mm_mullo_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_s32(
        vmulq_s32(vreinterpretq_s32_m128i(a), vreinterpretq_s32_m128i(b)));
}

// 将 packed 32 位有符号整数从 a 和 b 转换为使用无符号饱和度的 packed 16 位整数，并将结果存储在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_packus_epi32
FORCE_INLINE __m128i _mm_packus_epi32(__m128i a, __m128i b)
{
    return vreinterpretq_m128i_u16(
        vcombine_u16(vqmovun_s32(vreinterpretq_s32_m128i(a)),
                     vqmovun_s32(vreinterpretq_s32_m128i(b))));
}

// 对 packed 双精度（64 位）浮点元素进行舍入操作，使用舍入参数，并将结果存储为 packed 双精度浮点元素在 dst 中
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_round_pd
FORCE_INLINE __m128d _mm_round_pd(__m128d a, int rounding)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    switch (rounding) {
    case (_MM_FROUND_TO_NEAREST_INT | _MM_FROUND_NO_EXC):
        return vreinterpretq_m128d_f64(vrndnq_f64(vreinterpretq_f64_m128d(a)));
    case (_MM_FROUND_TO_NEG_INF | _MM_FROUND_NO_EXC):
        return _mm_floor_pd(a);
    case (_MM_FROUND_TO_POS_INF | _MM_FROUND_NO_EXC):
        return _mm_ceil_pd(a);
    case (_MM_FROUND_TO_ZERO | _MM_FROUND_NO_EXC):
        return vreinterpretq_m128d_f64(vrndq_f64(vreinterpretq_f64_m128d(a)));
    default:  //_MM_FROUND_CUR_DIRECTION
        return vreinterpretq_m128d_f64(vrndiq_f64(vreinterpretq_f64_m128d(a)));
    }
#else
    double *v_double = (double *) &a;
    # 如果舍入模式为向最近整数舍入且不产生异常，或者舍入模式为当前方向舍入且当前舍入模式为最近舍入
    if (rounding == (_MM_FROUND_TO_NEAREST_INT | _MM_FROUND_NO_EXC) ||
        (rounding == _MM_FROUND_CUR_DIRECTION &&
         _MM_GET_ROUNDING_MODE() == _MM_ROUND_NEAREST)) {
        # 创建一个长度为2的数组res，用于存储结果
        double res[2], tmp;
        # 遍历v_double数组的两个元素
        for (int i = 0; i < 2; i++) {
            # 如果v_double[i]小于0，将其取反赋值给tmp，否则直接赋值给tmp
            tmp = (v_double[i] < 0) ? -v_double[i] : v_double[i];
            # 向下取整，赋值给roundDown
            double roundDown = floor(tmp);  // Round down value
            # 向上取整，赋值给roundUp
            double roundUp = ceil(tmp);     // Round up value
            # 计算tmp与roundDown的差值，赋值给diffDown
            double diffDown = tmp - roundDown;
            # 计算roundUp与tmp的差值，赋值给diffUp
            double diffUp = roundUp - tmp;
            # 如果diffDown小于diffUp
            if (diffDown < diffUp) {
                /* If it's closer to the round down value, then use it */
                # 将roundDown赋值给res[i]
                res[i] = roundDown;
            # 如果diffDown大于diffUp
            } else if (diffDown > diffUp) {
                /* If it's closer to the round up value, then use it */
                # 将roundUp赋值给res[i]
                res[i] = roundUp;
            # 如果diffDown等于diffUp
            } else {
                /* If it's equidistant between round up and round down value,
                 * pick the one which is an even number */
                # 计算roundDown的一半，赋值给half
                double half = roundDown / 2;
                # 如果half不等于向下取整的half
                if (half != floor(half)) {
                    /* If the round down value is odd, return the round up value
                     */
                    # 将roundUp赋值给res[i]
                    res[i] = roundUp;
                # 如果roundDown的一半是奇数
                else {
                    /* If the round up value is odd, return the round down value
                     */
                    # 将roundDown赋值给res[i]
                    res[i] = roundDown;
                }
            }
            # 如果v_double[i]小于0，将res[i]取反赋值给res[i]，否则不变
            res[i] = (v_double[i] < 0) ? -res[i] : res[i];
        }
        # 将res[1]和res[0]作为参数，创建一个128位的浮点数向量返回
        return _mm_set_pd(res[1], res[0]);
    # 如果舍入模式为向负无穷舍入且不产生异常，或者舍入模式为当前方向舍入且当前舍入模式为向下舍入
    else if (rounding == (_MM_FROUND_TO_NEG_INF | _MM_FROUND_NO_EXC) ||
               (rounding == _MM_FROUND_CUR_DIRECTION &&
                _MM_GET_ROUNDING_MODE() == _MM_ROUND_DOWN)) {
        # 返回a的向下取整结果
        return _mm_floor_pd(a);
    # 如果舍入模式为向正无穷舍入且不产生异常，或者舍入模式为当前方向舍入且当前舍入模式为向上舍入
    else if (rounding == (_MM_FROUND_TO_POS_INF | _MM_FROUND_NO_EXC) ||
               (rounding == _MM_FROUND_CUR_DIRECTION &&
                _MM_GET_ROUNDING_MODE() == _MM_ROUND_UP)) {
        # 返回a的向上取整结果
        return _mm_ceil_pd(a);
    }
    # 返回一个包含两个双精度浮点数的向量，其中第一个数是根据条件判断后的结果，第二个数是根据条件判断后的结果
    return _mm_set_pd(v_double[1] > 0 ? floor(v_double[1]) : ceil(v_double[1]),
                      v_double[0] > 0 ? floor(v_double[0]) : ceil(v_double[0]));
#endif
}

// 使用指定的舍入参数对输入的单精度浮点数进行舍入，并将结果存储在目标寄存器中
// software.intel.com/sites/landingpage/IntrinsicsGuide/#text=_mm_round_ps
FORCE_INLINE __m128 _mm_round_ps(__m128 a, int rounding)
{
#if (defined(__aarch64__) || defined(_M_ARM64)) || \
    defined(__ARM_FEATURE_DIRECTED_ROUNDING)
    // 根据舍入参数进行不同的舍入操作
    switch (rounding) {
    case (_MM_FROUND_TO_NEAREST_INT | _MM_FROUND_NO_EXC):
        return vreinterpretq_m128_f32(vrndnq_f32(vreinterpretq_f32_m128(a)));
    case (_MM_FROUND_TO_NEG_INF | _MM_FROUND_NO_EXC):
        return _mm_floor_ps(a);
    case (_MM_FROUND_TO_POS_INF | _MM_FROUND_NO_EXC):
        return _mm_ceil_ps(a);
    case (_MM_FROUND_TO_ZERO | _MM_FROUND_NO_EXC):
        return vreinterpretq_m128_f32(vrndq_f32(vreinterpretq_f32_m128(a)));
    default:  //_MM_FROUND_CUR_DIRECTION
        return vreinterpretq_m128_f32(vrndiq_f32(vreinterpretq_f32_m128(a)));
    }
#else
    // 将输入的__m128类型转换为float指针
    float *v_float = (float *) &a;
    # 如果舍入模式为向最近整数舍入且不产生异常，或者舍入模式为当前方向且当前方向为最近舍入
    if (rounding == (_MM_FROUND_TO_NEAREST_INT | _MM_FROUND_NO_EXC) ||
        (rounding == _MM_FROUND_CUR_DIRECTION &&
         _MM_GET_ROUNDING_MODE() == _MM_ROUND_NEAREST)) {
        # 创建一个掩码，用于提取浮点数的符号位
        uint32x4_t signmask = vdupq_n_u32(0x80000000);
        # 创建一个浮点数向量，其中元素为0.5，根据符号位掩码选择正负0.5
        float32x4_t half = vbslq_f32(signmask, vreinterpretq_f32_m128(a),
                                     vdupq_n_f32(0.5f)); /* +/- 0.5 */
        # 将浮点数向量加上0.5后转换为整数向量，实现向最近整数舍入
        int32x4_t r_normal = vcvtq_s32_f32(vaddq_f32(
            vreinterpretq_f32_m128(a), half)); /* round to integer: [a + 0.5]*/
        # 将浮点数向量转换为整数向量，实现截断到整数
        int32x4_t r_trunc = vcvtq_s32_f32(
            vreinterpretq_f32_m128(a)); /* truncate to integer: [a] */
        # 创建一个整数向量，元素为1或0，根据截断到整数后的向量元素是否为负数确定
        int32x4_t plusone = vreinterpretq_s32_u32(vshrq_n_u32(
            vreinterpretq_u32_s32(vnegq_s32(r_trunc)), 31)); /* 1 or 0 */
        # 将截断到整数后的向量加上1，再与掩码取反后的向量进行按位与操作，实现向偶数舍入
        int32x4_t r_even = vbicq_s32(vaddq_s32(r_trunc, plusone),
                                     vdupq_n_s32(1)); /* ([a] + {0,1}) & ~1 */
        # 计算浮点数向量与截断到整数后的向量的差值，得到浮点数向量的小数部分
        float32x4_t delta = vsubq_f32(
            vreinterpretq_f32_m128(a),
            vcvtq_f32_s32(r_trunc)); /* compute delta: delta = (a - [a]) */
        # 判断浮点数向量的小数部分是否等于0.5，得到一个掩码向量
        uint32x4_t is_delta_half =
            vceqq_f32(delta, half); /* delta == +/- 0.5 */
        # 根据掩码向量选择向偶数舍入后的整数向量或向最近整数舍入后的整数向量，并转换为浮点数向量
        return vreinterpretq_m128_f32(
            vcvtq_f32_s32(vbslq_s32(is_delta_half, r_even, r_normal)));
    # 如果舍入模式为向负无穷舍入且不产生异常，或者舍入模式为当前方向且当前方向为向下舍入
    } else if (rounding == (_MM_FROUND_TO_NEG_INF | _MM_FROUND_NO_EXC) ||
               (rounding == _MM_FROUND_CUR_DIRECTION &&
                _MM_GET_ROUNDING_MODE() == _MM_ROUND_DOWN)) {
        # 调用内置函数_mm_floor_ps，向负无穷舍入
        return _mm_floor_ps(a);
    # 如果舍入模式为向正无穷舍入且不产生异常，或者舍入模式为当前方向且当前方向为向上舍入
    } else if (rounding == (_MM_FROUND_TO_POS_INF | _MM_FROUND_NO_EXC) ||
               (rounding == _MM_FROUND_CUR_DIRECTION &&
                _MM_GET_ROUNDING_MODE() == _MM_ROUND_UP)) {
        # 调用内置函数_mm_ceil_ps，向正无穷舍入
        return _mm_ceil_ps(a);
    }
    # 返回一个包含四个浮点数的数据结构，根据条件判断每个浮点数是否大于0，如果是则向下取整，否则向上取整
    return _mm_set_ps(v_float[3] > 0 ? floorf(v_float[3]) : ceilf(v_float[3]),
                      v_float[2] > 0 ? floorf(v_float[2]) : ceilf(v_float[2]),
                      v_float[1] > 0 ? floorf(v_float[1]) : ceilf(v_float[1]),
                      v_float[0] > 0 ? floorf(v_float[0]) : ceilf(v_float[0]));
#endif
}

// 使用指定的舍入参数对双精度浮点数元素 b 进行舍入，将结果存储为双精度浮点数元素 dst 的低位元素，并将 a 的上位元素复制到 dst 的上位元素
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_round_sd
FORCE_INLINE __m128d _mm_round_sd(__m128d a, __m128d b, int rounding)
{
    return _mm_move_sd(a, _mm_round_pd(b, rounding));
}

// 使用指定的舍入参数对单精度浮点数元素 b 进行舍入，将结果存储为单精度浮点数元素 dst 的低位元素，并将 a 的上位 3 个打包元素复制到 dst 的上位元素。根据舍入参数进行舍入，舍入参数可以是以下之一：
//     (_MM_FROUND_TO_NEAREST_INT |_MM_FROUND_NO_EXC) // 四舍五入，并且不产生异常
//     (_MM_FROUND_TO_NEG_INF |_MM_FROUND_NO_EXC)     // 向下舍入，并且不产生异常
//     (_MM_FROUND_TO_POS_INF |_MM_FROUND_NO_EXC)     // 向上舍入，并且不产生异常
//     (_MM_FROUND_TO_ZERO |_MM_FROUND_NO_EXC)        // 截断，并且不产生异常 _MM_FROUND_CUR_DIRECTION // 使用 MXCSR.RC；参见 _MM_SET_ROUNDING_MODE
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_round_ss
FORCE_INLINE __m128 _mm_round_ss(__m128 a, __m128 b, int rounding)
{
    return _mm_move_ss(a, _mm_round_ps(b, rounding));
}

// 从内存中加载 128 位整数数据到 dst，使用非临时内存提示。mem_addr 必须对齐到 16 字节边界，否则可能会生成通用保护异常。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_stream_load_si128
FORCE_INLINE __m128i _mm_stream_load_si128(__m128i *p)
{
#if __has_builtin(__builtin_nontemporal_store)
    return __builtin_nontemporal_load(p);
#else
    return vreinterpretq_m128i_s64(vld1q_s64((int64_t *) p));
#endif
}

// 计算 a 的按位取反，然后与一个包含全 1 的 128 位向量进行按位与运算，如果结果为零则返回 1，否则返回 0。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_test_all_ones
FORCE_INLINE int _mm_test_all_ones(__m128i a)
{
    // 将 a 的第 0 和第 1 个 64 位元素进行按位与运算，并将结果转换为 uint64_t 类型
    return (uint64_t) (vgetq_lane_s64(a, 0) & vgetq_lane_s64(a, 1)) ==
           ~(uint64_t) 0;
}

// 计算 a 和 mask 的 128 位（表示整数数据）的按位与，并返回 1 如果结果为零，否则返回 0。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_test_all_zeros
FORCE_INLINE int _mm_test_all_zeros(__m128i a, __m128i mask)
{
    // 将 a 和 mask 进行按位与运算，并将结果转换为 int64x2_t 类型
    int64x2_t a_and_mask =
        vandq_s64(vreinterpretq_s64_m128i(a), vreinterpretq_s64_m128i(mask));
    // 如果 a_and_mask 的第 0 和第 1 个 64 位元素都为零，则返回 1，否则返回 0
    return !(vgetq_lane_s64(a_and_mask, 0) | vgetq_lane_s64(a_and_mask, 1));
}

// 计算 a 和 mask 的 128 位（表示整数数据）的按位与，并将 ZF 设置为 1 如果结果为零，否则设置 ZF 为 0。
// 计算 a 的按位取反，然后与 mask 进行按位与运算，并将 CF 设置为 1 如果结果为零，否则设置 CF 为 0。
// 如果 ZF 和 CF 值都为零，则返回 1，否则返回 0。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=mm_test_mix_ones_zero
FORCE_INLINE int _mm_test_mix_ones_zeros(__m128i a, __m128i mask)
{
    // 将 mask 和 a 进行按位与运算，并将结果转换为 uint64x2_t 类型
    uint64x2_t zf =
        vandq_u64(vreinterpretq_u64_m128i(mask), vreinterpretq_u64_m128i(a));
    // 将 mask 和 a 的按位取反进行按位与运算，并将结果转换为 uint64x2_t 类型
    uint64x2_t cf =
        vbicq_u64(vreinterpretq_u64_m128i(mask), vreinterpretq_u64_m128i(a));
    // 将 zf 和 cf 进行按位与运算，并将结果转换为 uint64x2_t 类型
    uint64x2_t result = vandq_u64(zf, cf);
    // 如果 result 的第 0 和第 1 个 64 位元素都为零，则返回 1，否则返回 0
    return !(vgetq_lane_u64(result, 0) | vgetq_lane_u64(result, 1));
}

// 计算 a 和 b 的 128 位（表示整数数据）的按位与，并将 ZF 设置为 1 如果结果为零，否则设置 ZF 为 0。
// 计算 a 的按位取反，然后与 b 进行按位与运算，并将 CF 设置为 1 如果结果为零，否则设置 CF 为 0。
// 如果 ZF 和 CF 值都为零，则返回 1，否则返回 0。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=mm_test_mix_ones_zero
FORCE_INLINE int _mm_test_mix_ones_zeros(__m128i a, __m128i b)
{
    // 将 a 和 b 进行按位与运算，并将结果转换为 uint64x2_t 类型
    uint64x2_t zf =
        vandq_u64(vreinterpretq_u64_m128i(a), vreinterpretq_u64_m128i(b));
    // 将 a 的按位取反与 b 进行按位与运算，并将结果转换为 uint64x2_t 类型
    uint64x2_t cf =
        vbicq_u64(vreinterpretq_u64_m128i(a), vreinterpretq_u64_m128i(b));
    // 将 zf 和 cf 进行按位与运算，并将结果转换为 uint64x2_t 类型
    uint64x2_t result = vandq_u64(zf, cf);
    // 如果 result 的第 0 和第 1 个 64 位元素都为零，则返回 1，否则返回 0
    return !(vgetq_lane_u64(result, 0) | vgetq_lane_u64(result, 1));
}
// 对 a 进行按位取反，然后与 b 进行按位与，并将 CF 设置为 1（如果结果为零），否则将 CF 设置为 0。返回 CF 的值。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_testc_si128
FORCE_INLINE int _mm_testc_si128(__m128i a, __m128i b)
{
    // 将 b 和 a 转换为 int64x2_t 类型，并对其进行按位取反与按位与操作
    int64x2_t s64 =
        vbicq_s64(vreinterpretq_s64_m128i(b), vreinterpretq_s64_m128i(a));
    // 如果 s64 的第一个或第二个元素不为零，则返回 0；否则返回 1
    return !(vgetq_lane_s64(s64, 0) | vgetq_lane_s64(s64, 1));
}

// 计算 a 和 b 中的 128 位（表示整数数据）的按位与，并将 ZF 设置为 1（如果结果为零），否则将 ZF 设置为 0。对 a 进行按位取反与 b 进行按位与，并将 CF 设置为 1（如果结果为零），否则将 CF 设置为 0。如果 ZF 和 CF 值都为零，则返回 1；否则返回 0。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_testnzc_si128
#define _mm_testnzc_si128(a, b) _mm_test_mix_ones_zeros(a, b)

// 计算 a 和 b 中的 128 位（表示整数数据）的按位与，并将 ZF 设置为 1（如果结果为零），否则将 ZF 设置为 0。对 a 进行按位取反与 b 进行按位与，并将 CF 设置为 1（如果结果为零），否则将 CF 设置为 0。返回 ZF 的值。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_testz_si128
FORCE_INLINE int _mm_testz_si128(__m128i a, __m128i b)
{
    // 将 a 和 b 转换为 int64x2_t 类型，并对其进行按位与操作
    int64x2_t s64 =
        vandq_s64(vreinterpretq_s64_m128i(a), vreinterpretq_s64_m128i(b));
    // 如果 s64 的第一个或第二个元素不为零，则返回 0；否则返回 1
    return !(vgetq_lane_s64(s64, 0) | vgetq_lane_s64(s64, 1));
}

/* SSE4.2 */

// 用于比较字符串的掩码数组，用于比较 16 位字符
static const uint16_t ALIGN_STRUCT(16) _sse2neon_cmpestr_mask16b[8] = {
    0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80,
};
// 用于比较字符串的掩码数组，用于比较 8 位字符
static const uint8_t ALIGN_STRUCT(16) _sse2neon_cmpestr_mask8b[16] = {
    0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80,
    0x01, 0x02, 0x04, 0x08, 0x10, 0x20, 0x40, 0x80,
};

/* 指定源数据格式 */
#define _SIDD_UBYTE_OPS 0x00 /* unsigned 8-bit characters */
#define _SIDD_UWORD_OPS 0x01 /* 无符号 16 位字符 */
#define _SIDD_SBYTE_OPS 0x02 /* 有符号 8 位字符 */
#define _SIDD_SWORD_OPS 0x03 /* 有符号 16 位字符 */

/* 指定比较操作 */
#define _SIDD_CMP_EQUAL_ANY 0x00     /* 比较任意相等：strchr */
#define _SIDD_CMP_RANGES 0x04        /* 比较范围 */
#define _SIDD_CMP_EQUAL_EACH 0x08    /* 比较每个相等：strcmp */
#define _SIDD_CMP_EQUAL_ORDERED 0x0C /* 比较有序相等 */

/* 指定极性 */
#define _SIDD_POSITIVE_POLARITY 0x00
#define _SIDD_MASKED_POSITIVE_POLARITY 0x20
#define _SIDD_NEGATIVE_POLARITY 0x10 /* 反转结果 */
#define _SIDD_MASKED_NEGATIVE_POLARITY \
    0x30 /* 仅在字符串结束前反转结果 */

/* 在 _mm_cmpXstri 中指定输出选择 */
#define _SIDD_LEAST_SIGNIFICANT 0x00
#define _SIDD_MOST_SIGNIFICANT 0x40

/* 在 _mm_cmpXstrm 中指定输出选择 */
#define _SIDD_BIT_MASK 0x00
#define _SIDD_UNIT_MASK 0x40

/* C 宏的模式匹配。
 * https://github.com/pfultz2/Cloak/wiki/C-Preprocessor-tricks,-tips,-and-idioms
 */

/* 连接 */
#define SSE2NEON_PRIMITIVE_CAT(a, ...) a##__VA_ARGS__
#define SSE2NEON_CAT(a, b) SSE2NEON_PRIMITIVE_CAT(a, b)

#define SSE2NEON_IIF(c) SSE2NEON_PRIMITIVE_CAT(SSE2NEON_IIF_, c)
/* 运行第二个参数 */
#define SSE2NEON_IIF_0(t, ...) __VA_ARGS__
/* 运行第一个参数 */
#define SSE2NEON_IIF_1(t, ...) t

#define SSE2NEON_COMPL(b) SSE2NEON_PRIMITIVE_CAT(SSE2NEON_COMPL_, b)
#define SSE2NEON_COMPL_0 1
#define SSE2NEON_COMPL_1 0

#define SSE2NEON_DEC(x) SSE2NEON_PRIMITIVE_CAT(SSE2NEON_DEC_, x)
#define SSE2NEON_DEC_1 0
#define SSE2NEON_DEC_2 1
#define SSE2NEON_DEC_3 2
#define SSE2NEON_DEC_4 3
#define SSE2NEON_DEC_5 4
#define SSE2NEON_DEC_6 5
#define SSE2NEON_DEC_7 6
#define SSE2NEON_DEC_8 7
#define SSE2NEON_DEC_9 8
#define SSE2NEON_DEC_10 9
#define SSE2NEON_DEC_11 10
#define SSE2NEON_DEC_12 11
#define SSE2NEON_DEC_13 12
#define SSE2NEON_DEC_14 13
#define SSE2NEON_DEC_15 14
#define SSE2NEON_DEC_16 15

/* detection */

// 宏定义，用于检查参数个数是否为N
#define SSE2NEON_CHECK_N(x, n, ...) n
// 宏定义，用于检查参数是否为空
#define SSE2NEON_CHECK(...) SSE2NEON_CHECK_N(__VA_ARGS__, 0, )
// 宏定义，用于返回参数和1
#define SSE2NEON_PROBE(x) x, 1,
// 宏定义，用于返回参数的非
#define SSE2NEON_NOT(x) SSE2NEON_CHECK(SSE2NEON_PRIMITIVE_CAT(SSE2NEON_NOT_, x))
// 宏定义，用于返回参数的布尔值
#define SSE2NEON_BOOL(x) SSE2NEON_COMPL(SSE2NEON_NOT(x))
// 宏定义，用于根据条件返回表达式
#define SSE2NEON_IF(c) SSE2NEON_IIF(SSE2NEON_BOOL(c))

// 宏定义，用于吃掉参数
#define SSE2NEON_EAT(...)
// 宏定义，用于展开参数
#define SSE2NEON_EXPAND(...) __VA_ARGS__
// 宏定义，用于根据条件展开表达式
#define SSE2NEON_WHEN(c) SSE2NEON_IF(c)(SSE2NEON_EXPAND, SSE2NEON_EAT)

/* recursion */
/* deferred expression */

// 宏定义，用于返回空
#define SSE2NEON_EMPTY()
// 宏定义，用于延迟表达式
#define SSE2NEON_DEFER(id) id SSE2NEON_EMPTY()
// 宏定义，用于阻塞参数
#define SSE2NEON_OBSTRUCT(...) __VA_ARGS__ SSE2NEON_DEFER(SSE2NEON_EMPTY)()
// 宏定义，用于展开参数
#define SSE2NEON_EXPAND(...) __VA_ARGS__

// 宏定义，用于递归展开表达式
#define SSE2NEON_EVAL(...) \
    SSE2NEON_EVAL1(SSE2NEON_EVAL1(SSE2NEON_EVAL1(__VA_ARGS__)))
#define SSE2NEON_EVAL1(...) \
    SSE2NEON_EVAL2(SSE2NEON_EVAL2(SSE2NEON_EVAL2(__VA_ARGS__)))
#define SSE2NEON_EVAL2(...) \
    SSE2NEON_EVAL3(SSE2NEON_EVAL3(SSE2NEON_EVAL3(__VA_ARGS__)))
#define SSE2NEON_EVAL3(...) __VA_ARGS__

// 宏定义，用于重复执行宏
#define SSE2NEON_REPEAT(count, macro, ...)                         \
    SSE2NEON_WHEN(count)                                           \
    (SSE2NEON_OBSTRUCT(SSE2NEON_REPEAT_INDIRECT)()(                \
        SSE2NEON_DEC(count), macro,                                \
        __VA_ARGS__) SSE2NEON_OBSTRUCT(macro)(SSE2NEON_DEC(count), \
                                              __VA_ARGS__))
#define SSE2NEON_REPEAT_INDIRECT() SSE2NEON_REPEAT

// 宏定义，用于定义字节大小和字节的数量
#define SSE2NEON_SIZE_OF_byte 8
#define SSE2NEON_NUMBER_OF_LANES_byte 16
// 宏定义，用于定义字大小和字的数量
#define SSE2NEON_SIZE_OF_word 16
#define SSE2NEON_NUMBER_OF_LANES_word 8

// 宏定义，用于比较相等并填充通道
#define SSE2NEON_COMPARE_EQUAL_THEN_FILL_LANE(i, type)                         \
    # 将第i个元素的值从向量b中提取出来，并转换为指定类型的向量
    vdupq_n_##type(vgetq_lane_##type(vreinterpretq_##type##_m128i(b), i))
    
    # 将提取出来的向量值与向量a进行比较，返回比较结果的向量
    vceqq_##type(vdupq_n_##type(vgetq_lane_##type(vreinterpretq_##type##_m128i(b), i)), vreinterpretq_##type##_m128i(a))
    
    # 将比较结果的向量转换为指定类型的向量
    vreinterpretq_m128i_##type(vceqq_##type(vdupq_n_##type(vgetq_lane_##type(vreinterpretq_##type##_m128i(b), i)), vreinterpretq_##type##_m128i(a)))
    
    # 将转换后的向量值赋值给mtx[i]
    mtx[i] = vreinterpretq_m128i_##type(vceqq_##type(vdupq_n_##type(vgetq_lane_##type(vreinterpretq_##type##_m128i(b), i)), vreinterpretq_##type##_m128i(a)))
# 定义宏，用于将指定索引的元素从一个寄存器复制到另一个寄存器
#define SSE2NEON_FILL_LANE(i, type) \
    vec_b[i] =                      \
        vdupq_n_##type(vgetq_lane_##type(vreinterpretq_##type##_m128i(b), i));

# 定义宏，用于比较两个寄存器中的元素是否相等，并将结果填充到指定的寄存器中
#define PCMPSTR_RANGES(a, b, mtx, data_type_prefix, type_prefix, size,        \
                       number_of_lanes, byte_or_word)                         \
    } while (0)

# 定义宏，用于比较两个寄存器中的元素是否相等，并将结果填充到指定的寄存器中
#define PCMPSTR_EQ(a, b, mtx, size, number_of_lanes)                         \
    do {                                                                     \
        SSE2NEON_EVAL(SSE2NEON_REPEAT(number_of_lanes,                       \
                                      SSE2NEON_COMPARE_EQUAL_THEN_FILL_LANE, \
                                      SSE2NEON_CAT(u, size)))                \
    } while (0)

# 定义宏，用于比较两个寄存器中的元素是否相等，并返回结果
#define SSE2NEON_CMP_EQUAL_ANY_IMPL(type)                                     \
    static int _sse2neon_cmp_##type##_equal_any(__m128i a, int la, __m128i b, \
                                                int lb)                       \
    {                                                                         \
        __m128i mtx[16];                                                      \
        PCMPSTR_EQ(a, b, mtx, SSE2NEON_CAT(SSE2NEON_SIZE_OF_, type),          \
                   SSE2NEON_CAT(SSE2NEON_NUMBER_OF_LANES_, type));            \
        return SSE2NEON_CAT(                                                  \
            _sse2neon_aggregate_equal_any_,                                   \
            SSE2NEON_CAT(                                                     \
                SSE2NEON_CAT(SSE2NEON_SIZE_OF_, type),                        \
                SSE2NEON_CAT(x, SSE2NEON_CAT(SSE2NEON_NUMBER_OF_LANES_,       \
                                             type))))(la, lb, mtx);           \
    }

# 定义宏，用于比较两个寄存器中的元素是否在指定范围内，并返回结果
#define SSE2NEON_CMP_RANGES_IMPL(type, data_type, us, byte_or_word)            \
    // 定义一个静态函数，用于比较两个__m128i类型的数据范围
    static int _sse2neon_cmp_##us##type##_ranges(__m128i a, int la, __m128i b, \
                                                 int lb)                       \
    {                                                                          \
        // 定义一个__m128i类型的数组
        __m128i mtx[16];                                                       \
        // 调用PCMPSTR_RANGES宏，对a和b进行比较，并将结果存储在mtx数组中
        PCMPSTR_RANGES(                                                        \
            a, b, mtx, data_type, us, SSE2NEON_CAT(SSE2NEON_SIZE_OF_, type),   \
            SSE2NEON_CAT(SSE2NEON_NUMBER_OF_LANES_, type), byte_or_word);      \
        // 调用_sse2neon_aggregate_ranges_宏，对mtx数组中的数据进行聚合处理
        return SSE2NEON_CAT(                                                   \
            _sse2neon_aggregate_ranges_,                                       \
            SSE2NEON_CAT(                                                      \
                SSE2NEON_CAT(SSE2NEON_SIZE_OF_, type),                         \
                SSE2NEON_CAT(x, SSE2NEON_CAT(SSE2NEON_NUMBER_OF_LANES_,        \
                                             type))))(la, lb, mtx);            \
    }
#define SSE2NEON_CMP_EQUAL_ORDERED_IMPL(type)                                  \
    // 定义比较相等有序的宏函数，参数为类型
    static int _sse2neon_cmp_##type##_equal_ordered(__m128i a, int la,         \
                                                    __m128i b, int lb)         \
    {                                                                          \
        // 定义临时存储结果的数组
        __m128i mtx[16];                                                       \
        // 调用 PCMPSTR_EQ 宏函数进行比较，得到结果存储在 mtx 数组中
        PCMPSTR_EQ(a, b, mtx, SSE2NEON_CAT(SSE2NEON_SIZE_OF_, type),           \
                   SSE2NEON_CAT(SSE2NEON_NUMBER_OF_LANES_, type));             \
        // 调用 _sse2neon_aggregate_equal_ordered_ 宏函数进行聚合操作
        return SSE2NEON_CAT(                                                   \
            _sse2neon_aggregate_equal_ordered_,                                \
            SSE2NEON_CAT(                                                      \
                SSE2NEON_CAT(SSE2NEON_SIZE_OF_, type),                         \
                SSE2NEON_CAT(x,                                                \
                             SSE2NEON_CAT(SSE2NEON_NUMBER_OF_LANES_, type))))( \
            SSE2NEON_CAT(SSE2NEON_NUMBER_OF_LANES_, type), la, lb, mtx);       \
    }

static int _sse2neon_aggregate_equal_any_8x16(int la, int lb, __m128i mtx[16])
{
    // 初始化结果变量
    int res = 0;
    // 计算掩码
    int m = (1 << la) - 1;
    // 加载掩码数据
    uint8x8_t vec_mask = vld1_u8(_sse2neon_cmpestr_mask8b);
    // 进行位与运算
    uint8x8_t t_lo = vtst_u8(vdup_n_u8(m & 0xff), vec_mask);
    uint8x8_t t_hi = vtst_u8(vdup_n_u8(m >> 8), vec_mask);
    uint8x16_t vec = vcombine_u8(t_lo, t_hi);
    // 遍历 mtx 数组
    for (int j = 0; j < lb; j++) {
        // 进行位与运算
        mtx[j] = vreinterpretq_m128i_u8(
            vandq_u8(vec, vreinterpretq_u8_m128i(mtx[j])));
        // 右移 7 位
        mtx[j] = vreinterpretq_m128i_u8(
            vshrq_n_u8(vreinterpretq_u8_m128i(mtx[j]), 7));
        // 判断是否有非零元素
        int tmp = _sse2neon_vaddvq_u8(vreinterpretq_u8_m128i(mtx[j])) ? 1 : 0;
        // 更新结果
        res |= (tmp << j);
    }
    return res;
}

static int _sse2neon_aggregate_equal_any_16x8(int la, int lb, __m128i mtx[16])
{
    // 初始化结果变量
    int res = 0;
    // 计算掩码
    int m = (1 << la) - 1;
    # 创建一个 16 位无符号整数向量，其中每个元素都是 m 的副本，并与 _sse2neon_cmpestr_mask16b 向量进行逻辑与运算
    uint16x8_t vec = vtstq_u16(vdupq_n_u16(m), vld1q_u16(_sse2neon_cmpestr_mask16b));
    
    # 对于 j 从 0 到 lb-1 的每个值
    for (int j = 0; j < lb; j++) {
        # 将 mtx[j] 向量与 vec 向量进行逻辑与运算，并将结果重新解释为 m128i 类型的向量
        mtx[j] = vreinterpretq_m128i_u16(vandq_u16(vec, vreinterpretq_u16_m128i(mtx[j])));
        # 将 mtx[j] 向量右移 15 位，并将结果重新解释为 m128i 类型的向量
        mtx[j] = vreinterpretq_m128i_u16(vshrq_n_u16(vreinterpretq_u16_m128i(mtx[j]), 15));
        # 判断 mtx[j] 向量中是否存在非零元素，如果存在则将 tmp 设置为 1，否则设置为 0
        int tmp = _sse2neon_vaddvq_u16(vreinterpretq_u16_m128i(mtx[j])) ? 1 : 0;
        # 将 tmp 左移 j 位，并与 res 进行按位或运算
        res |= (tmp << j);
    }
    
    # 返回结果 res
    return res;
// 定义宏，生成比较相等的函数，包括字节和字
#define SSE2NEON_GENERATE_CMP_EQUAL_ANY(prefix) \
    prefix##IMPL(byte) \  // 生成比较相等的字节函数
    prefix##IMPL(word)    // 生成比较相等的字函数

// 关闭 clang 格式化
/* clang-format off */

// 通过宏生成比较相等的函数，包括字节和字
SSE2NEON_GENERATE_CMP_EQUAL_ANY(SSE2NEON_CMP_EQUAL_ANY_)

// 16x8 聚合范围的函数
static int _sse2neon_aggregate_ranges_16x8(int la, int lb, __m128i mtx[16])
{
    int res = 0;  // 结果初始化为 0
    int m = (1 << la) - 1;  // 计算 m 的值
    uint16x8_t vec = vtstq_u16(vdupq_n_u16(m), vld1q_u16(_sse2neon_cmpestr_mask16b));  // 使用 m 和掩码创建 16 位整数向量
    for (int j = 0; j < lb; j++) {  // 遍历 lb 次
        mtx[j] = vreinterpretq_m128i_u16(vandq_u16(vec, vreinterpretq_u16_m128i(mtx[j])));  // 对 mtx[j] 进行位与运算
        mtx[j] = vreinterpretq_m128i_u16(vshrq_n_u16(vreinterpretq_u16_m128i(mtx[j]), 15));  // 对 mtx[j] 进行右移 15 位
        __m128i tmp = vreinterpretq_m128i_u32(vshrq_n_u32(vreinterpretq_u32_m128i(mtx[j]), 16));  // 对 mtx[j] 进行右移 16 位
        uint32x4_t vec_res = vandq_u32(vreinterpretq_u32_m128i(mtx[j]), vreinterpretq_u32_m128i(tmp));  // 对 mtx[j] 和 tmp 进行位与运算
#if defined(__aarch64__) || defined(_M_ARM64)
        int t = vaddvq_u32(vec_res) ? 1 : 0;  // 如果 vec_res 中有非零元素，则 t 为 1，否则为 0
#else
        uint64x2_t sumh = vpaddlq_u32(vec_res);  // 对 vec_res 进行累加
        int t = vgetq_lane_u64(sumh, 0) + vgetq_lane_u64(sumh, 1);  // 计算累加结果
#endif
        res |= (t << j);  // 将 t 左移 j 位后与 res 进行或运算
    }
    return res;  // 返回结果
}

// 8x16 聚合范围的函数
static int _sse2neon_aggregate_ranges_8x16(int la, int lb, __m128i mtx[16])
{
    int res = 0;  // 结果初始化为 0
    int m = (1 << la) - 1;  // 计算 m 的值
    uint8x8_t vec_mask = vld1_u8(_sse2neon_cmpestr_mask8b);  // 加载掩码
    uint8x8_t t_lo = vtst_u8(vdup_n_u8(m & 0xff), vec_mask);  // 对低 8 位进行掩码测试
    uint8x8_t t_hi = vtst_u8(vdup_n_u8(m >> 8), vec_mask);  // 对高 8 位进行掩码测试
    uint8x16_t vec = vcombine_u8(t_lo, t_hi);  // 合并 t_lo 和 t_hi
    # 遍历 lb 次循环
    for (int j = 0; j < lb; j++) {
        # 将 vec 和 mtx[j] 进行按位与操作，并将结果转换为 128 位整数
        mtx[j] = vreinterpretq_m128i_u8(
            vandq_u8(vec, vreinterpretq_u8_m128i(mtx[j])));
        # 将 mtx[j] 右移 7 位，并将结果转换为 128 位整数
        mtx[j] = vreinterpretq_m128i_u8(
            vshrq_n_u8(vreinterpretq_u8_m128i(mtx[j]), 7));
        # 将 mtx[j] 转换为 128 位无符号整数，右移 8 位，并将结果转换为 128 位整数
        __m128i tmp = vreinterpretq_m128i_u16(
            vshrq_n_u16(vreinterpretq_u16_m128i(mtx[j]), 8));
        # 将 mtx[j] 和 tmp 进行按位与操作，并将结果转换为 128 位无符号整数
        uint16x8_t vec_res = vandq_u16(vreinterpretq_u16_m128i(mtx[j]),
                                       vreinterpretq_u16_m128i(tmp));
        # 调用 _sse2neon_vaddvq_u16 函数对 vec_res 进行求和，并判断是否大于 0，将结果转换为整数类型 t
        int t = _sse2neon_vaddvq_u16(vec_res) ? 1 : 0;
        # 将 t 左移 j 位，并与 res 进行按位或操作
        res |= (t << j);
    }
    # 返回最终结果 res
    return res;
# 定义 SSE2NEON_CMP_RANGES_IS_BYTE 宏为 1
#define SSE2NEON_CMP_RANGES_IS_BYTE 1
# 定义 SSE2NEON_CMP_RANGES_IS_WORD 宏为 0
#define SSE2NEON_CMP_RANGES_IS_WORD 0

/* clang-format off */
# 根据给定的前缀生成比较范围的宏定义
#define SSE2NEON_GENERATE_CMP_RANGES(prefix)             \
    prefix##IMPL(byte, uint, u, prefix##IS_BYTE)         \
    prefix##IMPL(byte, int, s, prefix##IS_BYTE)          \
    prefix##IMPL(word, uint, u, prefix##IS_WORD)         \
    prefix##IMPL(word, int, s, prefix##IS_WORD)
/* clang-format on */

# 使用 SSE2NEON_CMP_RANGES_ 作为前缀生成比较范围的宏
SSE2NEON_GENERATE_CMP_RANGES(SSE2NEON_CMP_RANGES_)

# 取消定义 SSE2NEON_CMP_RANGES_IS_BYTE 宏
#undef SSE2NEON_CMP_RANGES_IS_BYTE
# 取消定义 SSE2NEON_CMP_RANGES_IS_WORD 宏
#undef SSE2NEON_CMP_RANGES_IS_WORD

# 定义静态函数 _sse2neon_cmp_byte_equal_each，用于比较两个 __m128i 类型的变量是否相等
static int _sse2neon_cmp_byte_equal_each(__m128i a, int la, __m128i b, int lb)
{
    # 将两个 __m128i 类型的变量转换为 uint8x16_t 类型，并进行比较
    uint8x16_t mtx = vceqq_u8(vreinterpretq_u8_m128i(a), vreinterpretq_u8_m128i(b));
    # 计算 m0、m1、tb 的值
    int m0 = (la < lb) ? 0 : ((1 << la) - (1 << lb));
    int m1 = 0x10000 - (1 << la);
    int tb = 0x10000 - (1 << lb);
    uint8x8_t vec_mask, vec0_lo, vec0_hi, vec1_lo, vec1_hi;
    uint8x8_t tmp_lo, tmp_hi, res_lo, res_hi;
    vec_mask = vld1_u8(_sse2neon_cmpestr_mask8b);
    vec0_lo = vtst_u8(vdup_n_u8(m0), vec_mask);
    vec0_hi = vtst_u8(vdup_n_u8(m0 >> 8), vec_mask);
    vec1_lo = vtst_u8(vdup_n_u8(m1), vec_mask);
    vec1_hi = vtst_u8(vdup_n_u8(m1 >> 8), vec_mask);
    tmp_lo = vtst_u8(vdup_n_u8(tb), vec_mask);
    tmp_hi = vtst_u8(vdup_n_u8(tb >> 8), vec_mask);

    res_lo = vbsl_u8(vec0_lo, vdup_n_u8(0), vget_low_u8(mtx));
    res_hi = vbsl_u8(vec0_hi, vdup_n_u8(0), vget_high_u8(mtx));
    res_lo = vbsl_u8(vec1_lo, tmp_lo, res_lo);
    res_hi = vbsl_u8(vec1_hi, tmp_hi, res_hi);
    res_lo = vand_u8(res_lo, vec_mask);
    res_hi = vand_u8(res_hi, vec_mask);

    # 计算结果并返回
    int res = _sse2neon_vaddv_u8(res_lo) + (_sse2neon_vaddv_u8(res_hi) << 8);
    return res;
}

# 定义静态函数 _sse2neon_cmp_word_equal_each，用于比较两个 __m128i 类型的变量是否相等
static int _sse2neon_cmp_word_equal_each(__m128i a, int la, __m128i b, int lb)
{
    # 将两个 __m128i 类型的变量转换为 uint16x8_t 类型，并进行比较
    uint16x8_t mtx = vceqq_u16(vreinterpretq_u16_m128i(a), vreinterpretq_u16_m128i(b));
    # 计算 m0、m1、tb 的值
    int m0 = (la < lb) ? 0 : ((1 << la) - (1 << lb));
    int m1 = 0x100 - (1 << la);
    int tb = 0x100 - (1 << lb);
    # 加载16个16位整数到vec_mask中
    uint16x8_t vec_mask = vld1q_u16(_sse2neon_cmpestr_mask16b);
    # 将m0与vec_mask进行逻辑与操作
    uint16x8_t vec0 = vtstq_u16(vdupq_n_u16(m0), vec_mask);
    # 将m1与vec_mask进行逻辑与操作
    uint16x8_t vec1 = vtstq_u16(vdupq_n_u16(m1), vec_mask);
    # 将tb与vec_mask进行逻辑与操作
    uint16x8_t tmp = vtstq_u16(vdupq_n_u16(tb), vec_mask);
    # 根据vec0的值选择0或者mtx
    mtx = vbslq_u16(vec0, vdupq_n_u16(0), mtx);
    # 根据vec1的值选择tmp或者mtx
    mtx = vbslq_u16(vec1, tmp, mtx);
    # mtx与vec_mask进行逻辑与操作
    mtx = vandq_u16(mtx, vec_mask);
    # 返回mtx中所有元素的和
    return _sse2neon_vaddvq_u16(mtx);
# 定义宏，用于判断 SSE2NEON_AGGREGATE_EQUAL_ORDER_IMPL 函数中的数据类型是否为无符号字节型
#define SSE2NEON_AGGREGATE_EQUAL_ORDER_IS_UBYTE 1
# 定义宏，用于判断 SSE2NEON_AGGREGATE_EQUAL_ORDER_IMPL 函数中的数据类型是否为无符号字型
#define SSE2NEON_AGGREGATE_EQUAL_ORDER_IS_UWORD 0

# 定义宏，用于生成 SSE2NEON_AGGREGATE_EQUAL_ORDER_IMPL 函数
#define SSE2NEON_AGGREGATE_EQUAL_ORDER_IMPL(size, number_of_lanes, data_type)  \
    static int _sse2neon_aggregate_equal_ordered_##size##x##number_of_lanes(   \
        int bound, int la, int lb, __m128i mtx[16])                            \
    }

# 定义宏，用于生成 SSE2NEON_AGGREGATE_EQUAL_ORDER_ 函数
/* clang-format off */
#define SSE2NEON_GENERATE_AGGREGATE_EQUAL_ORDER(prefix) \
    prefix##IMPL(8, 16, prefix##IS_UBYTE)               \
    prefix##IMPL(16, 8, prefix##IS_UWORD)
/* clang-format on */

# 生成 SSE2NEON_AGGREGATE_EQUAL_ORDER_ 函数
SSE2NEON_GENERATE_AGGREGATE_EQUAL_ORDER(SSE2NEON_AGGREGATE_EQUAL_ORDER_)

# 取消之前定义的宏
#undef SSE2NEON_AGGREGATE_EQUAL_ORDER_IS_UBYTE
#undef SSE2NEON_AGGREGATE_EQUAL_ORDER_IS_UWORD

# 定义宏，用于生成 SSE2NEON_CMP_EQUAL_ORDERED_ 函数
/* clang-format off */
#define SSE2NEON_GENERATE_CMP_EQUAL_ORDERED(prefix) \
    prefix##IMPL(byte)                              \
    prefix##IMPL(word)
/* clang-format on */

# 生成 SSE2NEON_CMP_EQUAL_ORDERED_ 函数
SSE2NEON_GENERATE_CMP_EQUAL_ORDERED(SSE2NEON_CMP_EQUAL_ORDERED_)

# 定义 SSE2NEON_CMPESTR_LIST 列表
#define SSE2NEON_CMPESTR_LIST                          \
    _(CMP_UBYTE_EQUAL_ANY, cmp_byte_equal_any)         \
    _(CMP_UWORD_EQUAL_ANY, cmp_word_equal_any)         \
    _(CMP_SBYTE_EQUAL_ANY, cmp_byte_equal_any)         \
    _(CMP_SWORD_EQUAL_ANY, cmp_word_equal_any)         \
    _(CMP_UBYTE_RANGES, cmp_ubyte_ranges)              \
    _(CMP_UWORD_RANGES, cmp_uword_ranges)              \
    _(CMP_SBYTE_RANGES, cmp_sbyte_ranges)              \
    _(CMP_SWORD_RANGES, cmp_sword_ranges)              \
    _(CMP_UBYTE_EQUAL_EACH, cmp_byte_equal_each)       \
    _(CMP_UWORD_EQUAL_EACH, cmp_word_equal_each)       \
    _(CMP_SBYTE_EQUAL_EACH, cmp_byte_equal_each)       \
    _(CMP_SWORD_EQUAL_EACH, cmp_word_equal_each)       \
    _(CMP_UBYTE_EQUAL_ORDERED, cmp_byte_equal_ordered) \
    _(CMP_UWORD_EQUAL_ORDERED, cmp_word_equal_ordered) \
    _(CMP_SBYTE_EQUAL_ORDERED, cmp_byte_equal_ordered) \
    _(CMP_SWORD_EQUAL_ORDERED, cmp_word_equal_ordered)

# 定义枚举类型，用于表示 SSE2NEON_CMPESTR_LIST 列表中的每个元素
enum {
#define _(name, func_suffix) name,
    这是一个变量名，表示一个列表。
# 定义一个函数指针类型，用于存储比较字符串的函数
typedef int (*cmpestr_func_t)(__m128i a, int la, __m128i b, int lb);
# 定义一个函数指针数组，用于存储比较字符串的函数
static cmpestr_func_t _sse2neon_cmpfunc_table[] = {
    # 将宏展开后的函数指针添加到数组中
    _sse2neon_##func_suffix,
};

# 定义一个内联函数，用于处理_sse2neon_sido_negative函数的逻辑
FORCE_INLINE int _sse2neon_sido_negative(int res, int lb, int imm8, int bound)
{
    # 根据imm8的值进行不同的处理
    switch (imm8 & 0x30) {
    # 如果imm8的值为_SIDD_NEGATIVE_POLARITY，将res取反
    case _SIDD_NEGATIVE_POLARITY:
        res ^= 0xffffffff;
        break;
    # 如果imm8的值为_SIDD_MASKED_NEGATIVE_POLARITY，将res与(1 << lb) - 1进行异或
    case _SIDD_MASKED_NEGATIVE_POLARITY:
        res ^= (1 << lb) - 1;
        break;
    default:
        break;
    }

    # 根据bound的值进行不同的处理
    return res & ((bound == 8) ? 0xFF : 0xFFFF);
}

# 定义一个内联函数，用于计算一个无符号整数的前导零的个数
FORCE_INLINE int _sse2neon_clz(unsigned int x)
{
    # 根据编译器类型进行不同的处理
    # 如果是MSC编译器，使用_BitScanReverse函数计算前导零的个数
    # 否则，使用__builtin_clz函数计算前导零的个数
}

# 定义一个内联函数，用于计算一个无符号整数的末尾零的个数
FORCE_INLINE int _sse2neon_ctz(unsigned int x)
{
    # 根据编译器类型进行不同的处理
    # 如果是MSC编译器，使用_BitScanForward函数计算末尾零的个数
    # 否则，使用__builtin_ctz函数计算末尾零的个数
}

# 定义一个内联函数，用于计算一个无符号长整数的末尾零的个数
FORCE_INLINE int _sse2neon_ctzll(unsigned long long x)
{
    # 根据编译器类型进行不同的处理
    # 如果是MSC编译器，使用_BitScanForward64函数计算末尾零的个数
    # 否则，使用__builtin_ctzll函数计算末尾零的个数
}

# 定义一个宏，用于返回两个数中的最小值
#define SSE2NEON_MIN(x, y) (x) < (y) ? (x) : (y)

# 定义一个宏，用于设置SSE2NEON_CMPSTRX_LEN_PAIR宏中的上界
#define SSE2NEON_CMPSTR_SET_UPPER(var, imm) \
    const int var = (imm & 0x01) ? 8 : 16

# 定义一个宏，用于计算两个字符串的长度，并将其存储在la和lb中
#define SSE2NEON_CMPESTRX_LEN_PAIR(a, b, la, lb) \
    # 计算la的绝对值
    int tmp1 = la ^ (la >> 31);                  
    la = tmp1 - (la >> 31);                      
    # 计算lb的绝对值
    int tmp2 = lb ^ (lb >> 31);                  
    lb = tmp2 - (lb >> 31);                      
    # 将la和bound中的较小值存储在la中
    la = SSE2NEON_MIN(la, bound);                
    # 使用 SSE2NEON_MIN 函数对 lb 和 bound 进行比较，并返回较小的值
    lb = SSE2NEON_MIN(lb, bound)
// 比较字符串 a 和 b 中所有字符的所有配对，然后聚合结果。
// 由于 PCMPESTR* 和 PCMPISTR* 的唯一区别是计算字符串长度的方式，我们使用 SSE2NEON_CMP{I,E}STRX_GET_LEN 来获取字符串 a 和 b 的长度。
#define SSE2NEON_COMP_AGG(a, b, la, lb, imm8, IE)                  \
    // 设置边界值
    SSE2NEON_CMPSTR_SET_UPPER(bound, imm8);                        
    // 计算字符串 a 和 b 的长度
    SSE2NEON_##IE##_LEN_PAIR(a, b, la, lb);                        
    // 调用比较函数表中对应的函数，得到比较结果
    int r2 = (_sse2neon_cmpfunc_table[imm8 & 0x0f])(a, la, b, lb); 
    // 对比较结果进行处理
    r2 = _sse2neon_sido_negative(r2, lb, imm8, bound)

// 生成比较结果的索引
#define SSE2NEON_CMPSTR_GENERATE_INDEX(r2, bound, imm8)          \
    return (r2 == 0) ? bound                                     \
                     : ((imm8 & 0x40) ? (31 - _sse2neon_clz(r2)) \
                                      : _sse2neon_ctz(r2))

// 生成比较结果的掩码
#define SSE2NEON_CMPSTR_GENERATE_MASK(dst)                                     \
    // 将 0 转换为 128 位整数类型的掩码
    __m128i dst = vreinterpretq_m128i_u8(vdupq_n_u8(0));                      
    # 根据 imm8 的值进行条件判断
    if (imm8 & 0x40) {
        # 如果 bound 的值为 8
        if (bound == 8) {
            # 创建一个包含 r2 值的 16 位整数向量
            uint16x8_t tmp = vtstq_u16(vdupq_n_u16(r2), vld1q_u16(_sse2neon_cmpestr_mask16b));
            # 使用 tmp 向量的值作为掩码，将 dst 向量中对应位置为 -1 的元素替换为 -1
            dst = vreinterpretq_m128i_u16(vbslq_u16(tmp, vdupq_n_u16(-1), vreinterpretq_u16_m128i(dst)));
        } else {
            # 创建一个包含 r2 值的 8 位整数向量
            uint8x16_t vec_r2 = vcombine_u8(vdup_n_u8(r2), vdup_n_u8(r2 >> 8));
            # 使用 vec_r2 向量的值作为掩码，将 dst 向量中对应位置为 -1 的元素替换为 -1
            dst = vreinterpretq_m128i_u8(vbslq_u8(vec_r2, vdupq_n_u8(-1), vreinterpretq_u8_m128i(dst)));
        }
    } else {
        # 如果 bound 的值为 16
        if (bound == 16) {
            # 将 r2 的低 16 位值设置到 dst 向量的第一个元素
            dst = vreinterpretq_m128i_u16(vsetq_lane_u16(r2 & 0xffff, vreinterpretq_u16_m128i(dst), 0));
        } else {
            # 将 r2 的低 8 位值设置到 dst 向量的第一个元素
            dst = vreinterpretq_m128i_u8(vsetq_lane_u8(r2 & 0xff, vreinterpretq_u8_m128i(dst), 0));
        }
    }
    # 返回 dst 向量
    return dst
// 使用 imm8 控制参数比较 a 和 b 中长度为 la 和 lb 的打包字符串，并返回 1，如果 b 不包含空字符且结果掩码为零，否则返回 0
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpestra
FORCE_INLINE int _mm_cmpestra(__m128i a,
                              int la,
                              __m128i b,
                              int lb,
                              const int imm8)
{
    // 复制 lb 到 lb_cpy
    int lb_cpy = lb;
    // 调用 SSE2NEON_COMP_AGG 宏，使用 CMPESTRX 控制参数比较 a 和 b，生成结果
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPESTRX);
    // 返回结果是否为非零，并且 lb_cpy 大于 bound
    return !r2 & (lb_cpy > bound);
}

// 使用 imm8 控制参数比较 a 和 b 中长度为 la 和 lb 的打包字符串，并返回 1，如果结果掩码为非零，否则返回 0
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpestrc
FORCE_INLINE int _mm_cmpestrc(__m128i a,
                              int la,
                              __m128i b,
                              int lb,
                              const int imm8)
{
    // 调用 SSE2NEON_COMP_AGG 宏，使用 CMPESTRX 控制参数比较 a 和 b，生成结果
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPESTRX);
    // 返回结果是否为非零
    return r2 != 0;
}

// 使用 imm8 控制参数比较 a 和 b 中长度为 la 和 lb 的打包字符串，并将生成的索引存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpestri
FORCE_INLINE int _mm_cmpestri(__m128i a,
                              int la,
                              __m128i b,
                              int lb,
                              const int imm8)
{
    // 调用 SSE2NEON_COMP_AGG 宏，使用 CMPESTRX 控制参数比较 a 和 b，生成结果
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPESTRX);
    // 调用 SSE2NEON_CMPSTR_GENERATE_INDEX 宏，生成索引并存储在 r2 和 bound 中
    SSE2NEON_CMPSTR_GENERATE_INDEX(r2, bound, imm8);
}

// 使用 imm8 控制参数比较 a 和 b 中长度为 la 和 lb 的打包字符串，并将生成的掩码存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpestrm
FORCE_INLINE __m128i
_mm_cmpestrm(__m128i a, int la, __m128i b, int lb, const int imm8)
{
    # 使用 SSE2NEON_COMP_AGG 函数对 a 和 b 进行比较聚合操作，la 和 lb 分别表示 a 和 b 的长度，imm8 表示比较操作的模式，CMPESTRX 表示比较操作的类型
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPESTRX);
    # 使用 SSE2NEON_CMPSTR_GENERATE_MASK 函数生成比较结果的掩码，并将结果保存在 dst 中
    SSE2NEON_CMPSTR_GENERATE_MASK(dst);
// 比较具有长度 la 和 lb 的 a 和 b 中的打包字符串，使用 imm8 中的控制，并返回结果位掩码的第 0 位
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpestro
FORCE_INLINE int _mm_cmpestro(__m128i a,
                              int la,
                              __m128i b,
                              int lb,
                              const int imm8)
{
    // 调用宏 SSE2NEON_COMP_AGG，使用 CMPESTRX 控制
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPESTRX);
    // 返回结果位掩码的第 0 位
    return r2 & 1;
}

// 比较具有长度 la 和 lb 的 a 和 b 中的打包字符串，使用 imm8 中的控制，并返回 1（如果 a 中的任何字符为空）或 0（否则）
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpestrs
FORCE_INLINE int _mm_cmpestrs(__m128i a,
                              int la,
                              __m128i b,
                              int lb,
                              const int imm8)
{
    // 忽略参数 a, b, lb
    (void) a;
    (void) b;
    (void) lb;
    // 调用宏 SSE2NEON_CMPSTR_SET_UPPER，设置上界 bound，使用 imm8 控制
    SSE2NEON_CMPSTR_SET_UPPER(bound, imm8);
    // 返回 la 是否小于等于 (bound - 1)
    return la <= (bound - 1);
}

// 比较具有长度 la 和 lb 的 a 和 b 中的打包字符串，使用 imm8 中的控制，并返回 1（如果 b 中的任何字符为空）或 0（否则）
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpestrz
FORCE_INLINE int _mm_cmpestrz(__m128i a,
                              int la,
                              __m128i b,
                              int lb,
                              const int imm8)
{
    // 忽略参数 a, b, la
    (void) a;
    (void) b;
    (void) la;
    // 调用宏 SSE2NEON_CMPSTR_SET_UPPER，设置上界 bound，使用 imm8 控制
    SSE2NEON_CMPSTR_SET_UPPER(bound, imm8);
    // 返回 lb 是否小于等于 (bound - 1)
    return lb <= (bound - 1);
}

// 定义宏 SSE2NEON_CMPISTRX_LENGTH
#define SSE2NEON_CMPISTRX_LENGTH(str, len, imm8)                         \
    # 使用 do-while 循环，目的是为了能够在宏定义中使用多条语句
    do {
        # 检查 imm8 的最低位是否为 1
        if (imm8 & 0x01) {
            # 创建一个掩码，用于判断 str 中的每个元素是否等于 0
            uint16x8_t equal_mask_##str = vceqq_u16(vreinterpretq_u16_m128i(str), vdupq_n_u16(0));
            # 将掩码右移 4 位，并转换为 8 位整数类型
            uint8x8_t res_##str = vshrn_n_u16(equal_mask_##str, 4);
            # 将 res_##str 转换为 64 位整数类型，并取出其中的第一个元素
            uint64_t matches_##str = vget_lane_u64(vreinterpret_u64_u8(res_##str), 0);
            # 使用 SSE2NEON 库中的函数 _sse2neon_ctzll 计算 matches_##str 的低位 0 的个数，并右移 3 位
            len = _sse2neon_ctzll(matches_##str) >> 3;
        } else {
            # 创建一个掩码，用于判断 str 中的每个元素是否等于 0
            uint16x8_t equal_mask_##str = vreinterpretq_u16_u8(vceqq_u8(vreinterpretq_u8_m128i(str), vdupq_n_u8(0)));
            # 将掩码右移 4 位，并转换为 8 位整数类型
            uint8x8_t res_##str = vshrn_n_u16(equal_mask_##str, 4);
            # 将 res_##str 转换为 64 位整数类型，并取出其中的第一个元素
            uint64_t matches_##str = vget_lane_u64(vreinterpret_u64_u8(res_##str), 0);
            # 使用 SSE2NEON 库中的函数 _sse2neon_ctzll 计算 matches_##str 的低位 0 的个数，并右移 2 位
            len = _sse2neon_ctzll(matches_##str) >> 2;
        }
    } while (0)
// 定义一个宏，用于比较带有隐式长度的打包字符串a和b，使用imm8中的控制，并将长度存储在la和lb中
#define SSE2NEON_CMPISTRX_LEN_PAIR(a, b, la, lb) \
    int la, lb;                                  \
    do {                                         \
        SSE2NEON_CMPISTRX_LENGTH(a, la, imm8);   \
        SSE2NEON_CMPISTRX_LENGTH(b, lb, imm8);   \
    } while (0)

// 使用imm8中的控制，比较带有隐式长度的打包字符串a和b，并返回1，如果b不包含空字符且结果掩码为零，则返回0
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpistra
FORCE_INLINE int _mm_cmpistra(__m128i a, __m128i b, const int imm8)
{
    // 使用宏SSE2NEON_COMP_AGG，比较带有隐式长度的打包字符串a和b，并将长度存储在la和lb中，使用CMPISTRX控制
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPISTRX);
    // 返回结果，如果r2为0且lb大于等于bound，则返回1，否则返回0
    return !r2 & (lb >= bound);
}

// 使用imm8中的控制，比较带有隐式长度的打包字符串a和b，并返回1，如果结果掩码非零，则返回0
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpistrc
FORCE_INLINE int _mm_cmpistrc(__m128i a, __m128i b, const int imm8)
{
    // 使用宏SSE2NEON_COMP_AGG，比较带有隐式长度的打包字符串a和b，并将长度存储在la和lb中，使用CMPISTRX控制
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPISTRX);
    // 返回结果，如果r2不等于0，则返回1，否则返回0
    return r2 != 0;
}

// 使用imm8中的控制，比较带有隐式长度的打包字符串a和b，并将生成的索引存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpistri
FORCE_INLINE int _mm_cmpistri(__m128i a, __m128i b, const int imm8)
{
    // 使用宏SSE2NEON_COMP_AGG，比较带有隐式长度的打包字符串a和b，并将长度存储在la和lb中，使用CMPISTRX控制
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPISTRX);
    // 使用宏SSE2NEON_CMPSTR_GENERATE_INDEX，生成索引r2，bound和imm8
    SSE2NEON_CMPSTR_GENERATE_INDEX(r2, bound, imm8);
}

// 使用imm8中的控制，比较带有隐式长度的打包字符串a和b，并将生成的掩码存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpistrm
FORCE_INLINE __m128i _mm_cmpistrm(__m128i a, __m128i b, const int imm8)
{
    // 使用宏SSE2NEON_COMP_AGG，比较带有隐式长度的打包字符串a和b，并将长度存储在la和lb中，使用CMPISTRX控制
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPISTRX);
    // 使用宏SSE2NEON_CMPSTR_GENERATE_MASK，生成掩码dst
    SSE2NEON_CMPSTR_GENERATE_MASK(dst);
}

// 使用imm8中的控制，比较带有隐式长度的打包字符串a和b，并将生成的掩码存储在dst中
// 比较两个128位整数向量a和b的每个元素，根据imm8中的控制条件返回一个位掩码，其中只有结果的第0位有效。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpistro
FORCE_INLINE int _mm_cmpistro(__m128i a, __m128i b, const int imm8)
{
    // 使用宏展开，将a和b的每个元素进行比较，并根据imm8中的控制条件生成结果
    SSE2NEON_COMP_AGG(a, b, la, lb, imm8, CMPISTRX);
    // 返回结果的第0位
    return r2 & 1;
}

// 使用隐式长度比较a和b中的打包字符串，根据imm8中的控制条件返回1（如果a中有任何字符为null）或者返回0（如果a中没有字符为null）。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpistrs
FORCE_INLINE int _mm_cmpistrs(__m128i a, __m128i b, const int imm8)
{
    // 忽略b的值
    (void) b;
    // 使用宏展开，根据imm8中的控制条件设置上界
    SSE2NEON_CMPSTR_SET_UPPER(bound, imm8);
    // 定义变量la
    int la;
    // 使用宏展开，根据imm8中的控制条件计算a的长度
    SSE2NEON_CMPISTRX_LENGTH(a, la, imm8);
    // 返回la是否小于等于（上界-1）
    return la <= (bound - 1);
}

// 使用隐式长度比较a和b中的打包字符串，根据imm8中的控制条件返回1（如果b中有任何字符为null）或者返回0（如果b中没有字符为null）。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_cmpistrz
FORCE_INLINE int _mm_cmpistrz(__m128i a, __m128i b, const int imm8)
{
    // 忽略a的值
    (void) a;
    // 使用宏展开，根据imm8中的控制条件设置上界
    SSE2NEON_CMPSTR_SET_UPPER(bound, imm8);
    // 定义变量lb
    int lb;
    // 使用宏展开，根据imm8中的控制条件计算b的长度
    SSE2NEON_CMPISTRX_LENGTH(b, lb, imm8);
    // 返回lb是否小于等于（上界-1）
    return lb <= (bound - 1);
}

// 比较a和b中的两个有符号64位整数，如果a中的任何一个整数大于b中的对应整数，则返回结果。
FORCE_INLINE __m128i _mm_cmpgt_epi64(__m128i a, __m128i b)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 如果是ARM64架构，使用SIMD指令vcgtq_s64进行比较
    return vreinterpretq_m128i_u64(
        vcgtq_s64(vreinterpretq_s64_m128i(a), vreinterpretq_s64_m128i(b)));
#else
    // 如果不是ARM64架构，使用SIMD指令vqsubq_s64和vshrq_n_s64进行比较
    return vreinterpretq_m128i_s64(vshrq_n_s64(
        vqsubq_s64(vreinterpretq_s64_m128i(b), vreinterpretq_s64_m128i(a)),
        63));
#endif
}

// 从crc的初始值开始，累积一个CRC32值，用于无符号16位整数v，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_crc32_u16
FORCE_INLINE uint32_t _mm_crc32_u16(uint32_t crc, uint16_t v)
{
#if defined(__aarch64__) && defined(__ARM_FEATURE_CRC32)
    # 如果定义了 __aarch64__ 和 __ARM_FEATURE_CRC32，则执行以下代码块
    __asm__ __volatile__("crc32ch %w[c], %w[c], %w[v]\n\t"
                         : [c] "+r"(crc)
                         : [v] "r"(v));
#elif ((__ARM_ARCH == 8) && defined(__ARM_FEATURE_CRC32)) || \
    (defined(_M_ARM64) && !defined(__clang__))
    # 如果定义了 ((__ARM_ARCH == 8) && defined(__ARM_FEATURE_CRC32)) 或者 (defined(_M_ARM64) && !defined(__clang__))，则执行以下代码块
    crc = __crc32ch(crc, v);
#else
    # 如果以上条件都不满足，则执行以下代码块
    crc = _mm_crc32_u8(crc, v & 0xff);
    crc = _mm_crc32_u8(crc, (v >> 8) & 0xff);
#endif
    # 返回计算得到的 crc 值
    return crc;
}

// Starting with the initial value in crc, accumulates a CRC32 value for
// unsigned 32-bit integer v, and stores the result in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_crc32_u32
FORCE_INLINE uint32_t _mm_crc32_u32(uint32_t crc, uint32_t v)
{
#if defined(__aarch64__) && defined(__ARM_FEATURE_CRC32)
    # 如果定义了 __aarch64__ 和 __ARM_FEATURE_CRC32，则执行以下代码块
    __asm__ __volatile__("crc32cw %w[c], %w[c], %w[v]\n\t"
                         : [c] "+r"(crc)
                         : [v] "r"(v));
#elif ((__ARM_ARCH == 8) && defined(__ARM_FEATURE_CRC32)) || \
    (defined(_M_ARM64) && !defined(__clang__))
    # 如果定义了 ((__ARM_ARCH == 8) && defined(__ARM_FEATURE_CRC32)) 或者 (defined(_M_ARM64) && !defined(__clang__))，则执行以下代码块
    crc = __crc32cw(crc, v);
#else
    # 如果以上条件都不满足，则执行以下代码块
    crc = _mm_crc32_u16(crc, v & 0xffff);
    crc = _mm_crc32_u16(crc, (v >> 16) & 0xffff);
#endif
    # 返回计算得到的 crc 值
    return crc;
}

// Starting with the initial value in crc, accumulates a CRC32 value for
// unsigned 64-bit integer v, and stores the result in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_crc32_u64
FORCE_INLINE uint64_t _mm_crc32_u64(uint64_t crc, uint64_t v)
{
#if defined(__aarch64__) && defined(__ARM_FEATURE_CRC32)
    # 如果定义了 __aarch64__ 和 __ARM_FEATURE_CRC32，则执行以下代码块
    __asm__ __volatile__("crc32cx %w[c], %w[c], %x[v]\n\t"
                         : [c] "+r"(crc)
                         : [v] "r"(v));
#elif (defined(_M_ARM64) && !defined(__clang__))
    # 如果定义了 (defined(_M_ARM64) && !defined(__clang__))，则执行以下代码块
    crc = __crc32cd((uint32_t) crc, v);
#else
    # 如果以上条件都不满足，则执行以下代码块
    crc = _mm_crc32_u32((uint32_t) (crc), v & 0xffffffff);
    crc = _mm_crc32_u32((uint32_t) (crc), (v >> 32) & 0xffffffff);
#endif
    # 返回计算得到的 crc 值
    return crc;
}
// 从 crc 的初始值开始，累积一个无符号8位整数 v 的 CRC32 值，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_crc32_u8
FORCE_INLINE uint32_t _mm_crc32_u8(uint32_t crc, uint8_t v)
{
#if defined(__aarch64__) && defined(__ARM_FEATURE_CRC32)
    // 使用汇编指令 crc32cb 计算 CRC32 值
    __asm__ __volatile__("crc32cb %w[c], %w[c], %w[v]\n\t"
                         : [c] "+r"(crc)
                         : [v] "r"(v));
#elif ((__ARM_ARCH == 8) && defined(__ARM_FEATURE_CRC32)) || \
    (defined(_M_ARM64) && !defined(__clang__))
    // 使用内置函数 __crc32cb 计算 CRC32 值
    crc = __crc32cb(crc, v);
#else
    // 如果不支持 CRC32 指令，则使用普通的位运算计算 CRC32 值
    crc ^= v;
    for (int bit = 0; bit < 8; bit++) {
        if (crc & 1)
            crc = (crc >> 1) ^ UINT32_C(0x82f63b78);
        else
            crc = (crc >> 1);
    }
#endif
    return crc;
}

/* AES */

#if !defined(__ARM_FEATURE_CRYPTO) && (!defined(_M_ARM64) || defined(__clang__))
/* clang-format off */
// 定义 SSE2NEON_AES_SBOX 和 SSE2NEON_AES_RSBOX 宏
#define SSE2NEON_AES_SBOX(w)                                           \
    }
#define SSE2NEON_AES_RSBOX(w)                                          \
    }
/* clang-format on */

// 使用 X 宏技巧定义 SSE2NEON_AES_H0 宏
#define SSE2NEON_AES_H0(x) (x)
// 定义 _sse2neon_sbox 和 _sse2neon_rsbox 数组
static const uint8_t _sse2neon_sbox[256] = SSE2NEON_AES_SBOX(SSE2NEON_AES_H0);
static const uint8_t _sse2neon_rsbox[256] = SSE2NEON_AES_RSBOX(SSE2NEON_AES_H0);
#undef SSE2NEON_AES_H0

/* x_time 函数和矩阵乘法函数 */
#if !defined(__aarch64__) && !defined(_M_ARM64)
// 定义 SSE2NEON_XT 和 SSE2NEON_MULTIPLY 宏
#define SSE2NEON_XT(x) (((x) << 1) ^ ((((x) >> 7) & 1) * 0x1b))
#define SSE2NEON_MULTIPLY(x, y)                                  \
    (((y & 1) * x) ^ ((y >> 1 & 1) * SSE2NEON_XT(x)) ^           \
     ((y >> 2 & 1) * SSE2NEON_XT(SSE2NEON_XT(x))) ^              \
     ((y >> 3 & 1) * SSE2NEON_XT(SSE2NEON_XT(SSE2NEON_XT(x)))) ^ \
     ((y >> 4 & 1) * SSE2NEON_XT(SSE2NEON_XT(SSE2NEON_XT(SSE2NEON_XT(x))))))
#endif

// 在没有加密扩展的情况下，使用常规 NEON 实现 aesenc
// 定义一个名为_mm_aesenc_si128的函数，接受两个__m128i类型的参数a和RoundKey
FORCE_INLINE __m128i _mm_aesenc_si128(__m128i a, __m128i RoundKey)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    // 定义一个名为shift_rows的静态常量数组，包含16个元素
    static const uint8_t shift_rows[] = {
        0x0, 0x5, 0xa, 0xf, 0x4, 0x9, 0xe, 0x3,
        0x8, 0xd, 0x2, 0x7, 0xc, 0x1, 0x6, 0xb,
    };
    // 定义一个名为ror32by8的静态常量数组，包含16个元素
    static const uint8_t ror32by8[] = {
        0x1, 0x2, 0x3, 0x0, 0x5, 0x6, 0x7, 0x4,
        0x9, 0xa, 0xb, 0x8, 0xd, 0xe, 0xf, 0xc,
    };

    // 定义一个名为v的uint8x16_t类型变量
    uint8x16_t v;
    // 将参数a转换为uint8x16_t类型，赋值给w
    uint8x16_t w = vreinterpretq_u8_m128i(a);

    /* shift rows */
    // 使用vqtbl1q_u8函数对w进行按位选择，选择的索引值来自shift_rows数组
    w = vqtbl1q_u8(w, vld1q_u8(shift_rows));

    /* sub bytes */
    // 使用vqtbl4q_u8函数对_sse2neon_sbox数组进行按位选择，选择的索引值来自w
    v = vqtbl4q_u8(_sse2neon_vld1q_u8_x4(_sse2neon_sbox), w);
    // 对w-0x40进行按位选择，选择的索引值来自_sse2neon_sbox + 0x40
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0x40), w - 0x40);
    // 对w-0x80进行按位选择，选择的索引值来自_sse2neon_sbox + 0x80
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0x80), w - 0x80);
    // 对w-0xc0进行按位选择，选择的索引值来自_sse2neon_sbox + 0xc0
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0xc0), w - 0xc0);

    /* mix columns */
    // 对v进行左移1位，然后与((int8x16_t) v >> 7) & 0x1b进行按位异或
    w = (v << 1) ^ (uint8x16_t) (((int8x16_t) v >> 7) & 0x1b);
    // 对w进行按位异或，选择的索引值来自vrev32q_u16((uint16x8_t) v)
    w ^= (uint8x16_t) vrev32q_u16((uint16x8_t) v);
    // 对w进行按位异或，选择的索引值来自ror32by8数组
    w ^= vqtbl1q_u8(v ^ w, vld1q_u8(ror32by8));

    /* add round key */
    // 将w转换为__m128i类型，然后与RoundKey进行按位异或
    return vreinterpretq_m128i_u8(w) ^ RoundKey;

#else /* ARMv7-A implementation for a table-based AES */
// 定义一个宏SSE2NEON_AES_B2W，接受4个参数b0, b1, b2, b3
#define SSE2NEON_AES_B2W(b0, b1, b2, b3)                 \
    (((uint32_t) (b3) << 24) | ((uint32_t) (b2) << 16) | \
     ((uint32_t) (b1) << 8) | (uint32_t) (b0))
// 定义宏，将 'x' 在 GF(2^8) 中乘以 2
#define SSE2NEON_AES_F2(x) ((x << 1) ^ (((x >> 7) & 1) * 0x011b /* WPOLY */))
// 定义宏，将 'x' 在 GF(2^8) 中乘以 3
#define SSE2NEON_AES_F3(x) (SSE2NEON_AES_F2(x) ^ x)
// 定义宏，用于计算 U0
#define SSE2NEON_AES_U0(p) \
    SSE2NEON_AES_B2W(SSE2NEON_AES_F2(p), p, p, SSE2NEON_AES_F3(p))
// 定义宏，用于计算 U1
#define SSE2NEON_AES_U1(p) \
    SSE2NEON_AES_B2W(SSE2NEON_AES_F3(p), SSE2NEON_AES_F2(p), p, p)
// 定义宏，用于计算 U2
#define SSE2NEON_AES_U2(p) \
    SSE2NEON_AES_B2W(p, SSE2NEON_AES_F3(p), SSE2NEON_AES_F2(p), p)
// 定义宏，用于计算 U3
#define SSE2NEON_AES_U3(p) \
    SSE2NEON_AES_B2W(p, p, SSE2NEON_AES_F3(p), SSE2NEON_AES_F2(p))

// 生成包含每个可能的 shift_rows()、sub_bytes() 和 mix_columns() 排列的表
static const uint32_t ALIGN_STRUCT(16) aes_table[4][256] = {
    SSE2NEON_AES_SBOX(SSE2NEON_AES_U0),
    SSE2NEON_AES_SBOX(SSE2NEON_AES_U1),
    SSE2NEON_AES_SBOX(SSE2NEON_AES_U2),
    SSE2NEON_AES_SBOX(SSE2NEON_AES_U3),
};
#undef SSE2NEON_AES_B2W
#undef SSE2NEON_AES_F2
#undef SSE2NEON_AES_F3
#undef SSE2NEON_AES_U0
#undef SSE2NEON_AES_U1
#undef SSE2NEON_AES_U2
#undef SSE2NEON_AES_U3

// 从 a 中获取 a[31:0]
uint32_t x0 = _mm_cvtsi128_si32(a);
// 从 a 中获取 a[63:32]
uint32_t x1 =
    _mm_cvtsi128_si32(_mm_shuffle_epi32(a, 0x55));
// 从 a 中获取 a[95:64]
uint32_t x2 =
    _mm_cvtsi128_si32(_mm_shuffle_epi32(a, 0xAA));
// 从 a 中获取 a[127:96]
uint32_t x3 =
    _mm_cvtsi128_si32(_mm_shuffle_epi32(a, 0xFF));

// 完成 mix_columns() 中的模加步骤
    # 使用给定的参数构建一个128位的__m128i对象
    out = _mm_set_epi32(
        # 使用AES表进行异或运算，得到out的第一个32位
        (aes_table[0][x3 & 0xff] ^ aes_table[1][(x0 >> 8) & 0xff] ^
         aes_table[2][(x1 >> 16) & 0xff] ^ aes_table[3][x2 >> 24]),
        # 使用AES表进行异或运算，得到out的第二个32位
        (aes_table[0][x2 & 0xff] ^ aes_table[1][(x3 >> 8) & 0xff] ^
         aes_table[2][(x0 >> 16) & 0xff] ^ aes_table[3][x1 >> 24]),
        # 使用AES表进行异或运算，得到out的第三个32位
        (aes_table[0][x1 & 0xff] ^ aes_table[1][(x2 >> 8) & 0xff] ^
         aes_table[2][(x3 >> 16) & 0xff] ^ aes_table[3][x0 >> 24]),
        # 使用AES表进行异或运算，得到out的第四个32位
        (aes_table[0][x0 & 0xff] ^ aes_table[1][(x1 >> 8) & 0xff] ^
         aes_table[2][(x2 >> 16) & 0xff] ^ aes_table[3][x3 >> 24])
    )
    
    # 返回out和RoundKey的异或结果
    return _mm_xor_si128(out, RoundKey);
// 结束条件判断，结束当前函数
#endif
}

// 在使用RoundKey中的轮密钥对数据（state）执行AES解密流程的一个轮次，并将结果存储在dst中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesdec_si128
FORCE_INLINE __m128i _mm_aesdec_si128(__m128i a, __m128i RoundKey)
{
#if defined(__aarch64__)
    // 逆移位行
    static const uint8_t inv_shift_rows[] = {
        0x0, 0xd, 0xa, 0x7, 0x4, 0x1, 0xe, 0xb,
        0x8, 0x5, 0x2, 0xf, 0xc, 0x9, 0x6, 0x3,
    };
    // 以8位为单位的32位循环右移
    static const uint8_t ror32by8[] = {
        0x1, 0x2, 0x3, 0x0, 0x5, 0x6, 0x7, 0x4,
        0x9, 0xa, 0xb, 0x8, 0xd, 0xe, 0xf, 0xc,
    };

    uint8x16_t v;
    uint8x16_t w = vreinterpretq_u8_m128i(a);

    // 逆置换字节
    w = vqtbl1q_u8(w, vld1q_u8(inv_shift_rows));

    // 逆S盒变换
    v = vqtbl4q_u8(_sse2neon_vld1q_u8_x4(_sse2neon_rsbox), w);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_rsbox + 0x40), w - 0x40);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_rsbox + 0x80), w - 0x80);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_rsbox + 0xc0), w - 0xc0);

    // 逆混合列
    // 在GF(2^8)中将'v'乘以4
    w = (v << 1) ^ (uint8x16_t) (((int8x16_t) v >> 7) & 0x1b);
    w = (w << 1) ^ (uint8x16_t) (((int8x16_t) w >> 7) & 0x1b);
    v ^= w;
    v ^= (uint8x16_t) vrev32q_u16((uint16x8_t) w);

    w = (v << 1) ^ (uint8x16_t) (((int8x16_t) v >> 7) &
                                 0x1b);  // 在GF(2^8)中将'v'乘以2
    w ^= (uint8x16_t) vrev32q_u16((uint16x8_t) v);
    w ^= vqtbl1q_u8(v ^ w, vld1q_u8(ror32by8));

    // 添加轮密钥
    return vreinterpretq_m128i_u8(w) ^ RoundKey;

#else /* ARMv7-A NEON implementation */
    /* FIXME: optimized for NEON */
    uint8_t i, e, f, g, h, v[4][4];
    uint8_t *_a = (uint8_t *) &a;
    for (i = 0; i < 16; ++i) {
        v[((i / 4) + (i % 4)) % 4][i % 4] = _sse2neon_rsbox[_a[i]];
    }

    // 逆混合列
    # 对于每个 i 在 0 到 3 之间的循环
    for (i = 0; i < 4; ++i) {
        # 将 v[i][0] 赋值给 e
        e = v[i][0];
        # 将 v[i][1] 赋值给 f
        f = v[i][1];
        # 将 v[i][2] 赋值给 g
        g = v[i][2];
        # 将 v[i][3] 赋值给 h
        h = v[i][3];
    
        # 使用 SSE2NEON_MULTIPLY 函数对 e、f、g、h 进行乘法运算，并进行异或操作
        v[i][0] = SSE2NEON_MULTIPLY(e, 0x0e) ^ SSE2NEON_MULTIPLY(f, 0x0b) ^
                  SSE2NEON_MULTIPLY(g, 0x0d) ^ SSE2NEON_MULTIPLY(h, 0x09);
        v[i][1] = SSE2NEON_MULTIPLY(e, 0x09) ^ SSE2NEON_MULTIPLY(f, 0x0e) ^
                  SSE2NEON_MULTIPLY(g, 0x0b) ^ SSE2NEON_MULTIPLY(h, 0x0d);
        v[i][2] = SSE2NEON_MULTIPLY(e, 0x0d) ^ SSE2NEON_MULTIPLY(f, 0x09) ^
                  SSE2NEON_MULTIPLY(g, 0x0e) ^ SSE2NEON_MULTIPLY(h, 0x0b);
        v[i][3] = SSE2NEON_MULTIPLY(e, 0x0b) ^ SSE2NEON_MULTIPLY(f, 0x0d) ^
                  SSE2NEON_MULTIPLY(g, 0x09) ^ SSE2NEON_MULTIPLY(h, 0x0e);
    }
    
    # 将 v 转换为 m128i 类型的数据，并与 RoundKey 进行异或操作，返回结果
    return vreinterpretq_m128i_u8(vld1q_u8((uint8_t *) v)) ^ RoundKey;
#endif
}

// 对数据（state）执行 AES 加密流的最后一轮，使用 RoundKey 中的轮密钥，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesenclast_si128
FORCE_INLINE __m128i _mm_aesenclast_si128(__m128i a, __m128i RoundKey)
{
#if defined(__aarch64__)
    // 定义行移位的映射表
    static const uint8_t shift_rows[] = {
        0x0, 0x5, 0xa, 0xf, 0x4, 0x9, 0xe, 0x3,
        0x8, 0xd, 0x2, 0x7, 0xc, 0x1, 0x6, 0xb,
    };

    // 定义变量 v 和 w
    uint8x16_t v;
    uint8x16_t w = vreinterpretq_u8_m128i(a);

    // 行移位
    w = vqtbl1q_u8(w, vld1q_u8(shift_rows));

    // 字节替换
    v = vqtbl4q_u8(_sse2neon_vld1q_u8_x4(_sse2neon_sbox), w);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0x40), w - 0x40);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0x80), w - 0x80);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0xc0), w - 0xc0);

    // 轮密钥加
    return vreinterpretq_m128i_u8(v) ^ RoundKey;

#else /* ARMv7-A implementation */
    # 创建一个长度为16的uint8_t类型的数组v，用于存储结果
    uint8_t v[16] = {
        # 将a向量的第0个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 0)],
        # 将a向量的第5个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 5)],
        # 将a向量的第10个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 10)],
        # 将a向量的第15个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 15)],
        # 将a向量的第4个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 4)],
        # 将a向量的第9个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 9)],
        # 将a向量的第14个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 14)],
        # 将a向量的第3个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 3)],
        # 将a向量的第8个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 8)],
        # 将a向量的第13个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 13)],
        # 将a向量的第2个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 2)],
        # 将a向量的第7个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 7)],
        # 将a向量的第12个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 12)],
        # 将a向量的第1个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 1)],
        # 将a向量的第6个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 6)],
        # 将a向量的第11个元素通过vgetq_lane_u8函数取出，然后通过_sse2neon_sbox查找对应的值
        _sse2neon_sbox[vgetq_lane_u8(vreinterpretq_u8_m128i(a), 11)],
    };
    
    # 将数组v转换为m128i类型的向量，并与RoundKey进行异或操作，返回结果
    return vreinterpretq_m128i_u8(vld1q_u8(v)) ^ RoundKey;
#endif
}

// 对使用 RoundKey 中的轮密钥在数据（state）上执行 AES 解密流的最后一轮，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesdeclast_si128
FORCE_INLINE __m128i _mm_aesdeclast_si128(__m128i a, __m128i RoundKey)
{
#if defined(__aarch64__)
    // 逆移位行
    static const uint8_t inv_shift_rows[] = {
        0x0, 0xd, 0xa, 0x7, 0x4, 0x1, 0xe, 0xb,
        0x8, 0x5, 0x2, 0xf, 0xc, 0x9, 0x6, 0x3,
    };

    uint8x16_t v;
    uint8x16_t w = vreinterpretq_u8_m128i(a);

    // 逆代换字节
    w = vqtbl1q_u8(w, vld1q_u8(inv_shift_rows));

    // 逆代换字节
    v = vqtbl4q_u8(_sse2neon_vld1q_u8_x4(_sse2neon_rsbox), w);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_rsbox + 0x40), w - 0x40);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_rsbox + 0x80), w - 0x80);
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_rsbox + 0xc0), w - 0xc0);

    // 添加轮密钥
    return vreinterpretq_m128i_u8(v) ^ RoundKey;

#else /* ARMv7-A NEON implementation */
    /* FIXME: optimized for NEON */
    uint8_t v[4][4];
    uint8_t *_a = (uint8_t *) &a;
    for (int i = 0; i < 16; ++i) {
        v[((i / 4) + (i % 4)) % 4][i % 4] = _sse2neon_rsbox[_a[i]];
    }

    return vreinterpretq_m128i_u8(vld1q_u8((uint8_t *) v)) ^ RoundKey;
#endif
}

// 对 a 执行 InvMixColumns 转换，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesimc_si128
FORCE_INLINE __m128i _mm_aesimc_si128(__m128i a)
{
#if defined(__aarch64__)
    // 循环右移 32 位，每次移动 8 位
    static const uint8_t ror32by8[] = {
        0x1, 0x2, 0x3, 0x0, 0x5, 0x6, 0x7, 0x4,
        0x9, 0xa, 0xb, 0x8, 0xd, 0xe, 0xf, 0xc,
    };
    uint8x16_t v = vreinterpretq_u8_m128i(a);
    uint8x16_t w;

    // 在 GF(2^8) 中将 'v' 乘以 4
    w = (v << 1) ^ (uint8x16_t) (((int8x16_t) v >> 7) & 0x1b);
    w = (w << 1) ^ (uint8x16_t) (((int8x16_t) w >> 7) & 0x1b);
    v ^= w;
    # 使用按位异或操作符对 v 和 vrev32q_u16((uint16x8_t) w) 进行按位异或操作
    v ^= (uint8x16_t) vrev32q_u16((uint16x8_t) w);

    # 在有限域 GF(2^8) 中将 v 乘以 2
    w = (v << 1) ^ (uint8x16_t) (((int8x16_t) v >> 7) & 0x1b);
    # 对 w 进行按位异或操作
    w ^= (uint8x16_t) vrev32q_u16((uint16x8_t) v);
    # 使用 vqtbl1q_u8 函数对 v ^ w 进行按位异或操作
    w ^= vqtbl1q_u8(v ^ w, vld1q_u8(ror32by8));
    # 返回 vreinterpretq_m128i_u8(w) 的结果
    return vreinterpretq_m128i_u8(w);
#else /* ARMv7-A NEON implementation */
    # 如果不是 ARMv7-A NEON 实现，则执行以下代码
    uint8_t i, e, f, g, h, v[4][4];
    # 定义变量 i, e, f, g, h, v，其中 v 是一个 4x4 的二维数组
    vst1q_u8((uint8_t *) v, vreinterpretq_u8_m128i(a));
    # 将 m128i 类型的寄存器 a 转换为 uint8_t 类型，并存储到数组 v 中
    for (i = 0; i < 4; ++i) {
        # 循环遍历数组 v 的每一行
        e = v[i][0];
        f = v[i][1];
        g = v[i][2];
        h = v[i][3];

        v[i][0] = SSE2NEON_MULTIPLY(e, 0x0e) ^ SSE2NEON_MULTIPLY(f, 0x0b) ^
                  SSE2NEON_MULTIPLY(g, 0x0d) ^ SSE2NEON_MULTIPLY(h, 0x09);
        # 对数组 v 的每一行进行一系列的位运算操作
        v[i][1] = SSE2NEON_MULTIPLY(e, 0x09) ^ SSE2NEON_MULTIPLY(f, 0x0e) ^
                  SSE2NEON_MULTIPLY(g, 0x0b) ^ SSE2NEON_MULTIPLY(h, 0x0d);
        v[i][2] = SSE2NEON_MULTIPLY(e, 0x0d) ^ SSE2NEON_MULTIPLY(f, 0x09) ^
                  SSE2NEON_MULTIPLY(g, 0x0e) ^ SSE2NEON_MULTIPLY(h, 0x0b);
        v[i][3] = SSE2NEON_MULTIPLY(e, 0x0b) ^ SSE2NEON_MULTIPLY(f, 0x0d) ^
                  SSE2NEON_MULTIPLY(g, 0x09) ^ SSE2NEON_MULTIPLY(h, 0x0e);
    }
    # 循环结束后，返回结果
    return vreinterpretq_m128i_u8(vld1q_u8((uint8_t *) v));
#endif
}

// Assist in expanding the AES cipher key by computing steps towards generating
// a round key for encryption cipher using data from a and an 8-bit round
// constant specified in imm8, and store the result in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aeskeygenassist_si128
//
// Emits the Advanced Encryption Standard (AES) instruction aeskeygenassist.
// This instruction generates a round key for AES encryption. See
// https://kazakov.life/2017/11/01/cryptocurrency-mining-on-ios-devices/
// for details.
# 辅助扩展 AES 密钥，通过使用来自 a 的数据和 imm8 中指定的 8 位轮常数，计算生成加密密码的轮密钥，并将结果存储在 dst 中
# https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aeskeygenassist_si128
#
# 发出高级加密标准 (AES) 指令 aeskeygenassist。
# 此指令生成 AES 加密的轮密钥。有关详细信息，请参见 https://kazakov.life/2017/11/01/cryptocurrency-mining-on-ios-devices/
FORCE_INLINE __m128i _mm_aeskeygenassist_si128(__m128i a, const int rcon)
{
#if defined(__aarch64__)
    # 如果是 aarch64 架构
    uint8x16_t _a = vreinterpretq_u8_m128i(a);
    # 将 m128i 类型的寄存器 a 转换为 uint8x16_t 类型
    uint8x16_t v = vqtbl4q_u8(_sse2neon_vld1q_u8_x4(_sse2neon_sbox), _a);
    # 使用 vqtbl4q_u8 函数进行数据重排
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0x40), _a - 0x40);
    # 使用 vqtbx4q_u8 函数进行数据重排
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0x80), _a - 0x80);
    # 使用 vqtbx4q_u8 函数进行数据重排
    v = vqtbx4q_u8(v, _sse2neon_vld1q_u8_x4(_sse2neon_sbox + 0xc0), _a - 0xc0);
    # 使用 vqtbx4q_u8 函数进行数据重排

    uint32x4_t v_u32 = vreinterpretq_u32_u8(v);
    # 将 uint8x16_t 类型的数据转换为 uint32x4_t 类型
    # 创建一个包含4个32位无符号整数的向量，其中每个元素都是v_u32向右循环移位8位的结果
    uint32x4_t ror_v = vorrq_u32(vshrq_n_u32(v_u32, 8), vshlq_n_u32(v_u32, 24));
    
    # 创建一个包含4个32位无符号整数的向量，其中每个元素都是ror_v和rcon进行异或操作的结果
    uint32x4_t ror_xor_v = veorq_u32(ror_v, vdupq_n_u32(rcon));
    
    # 将v_u32和ror_xor_v的高位元素进行交换，并将结果转换为128位整数向量
    return vreinterpretq_m128i_u32(vtrn2q_u32(v_u32, ror_xor_v));
#else /* ARMv7-A NEON implementation */
#else /* ARMv7-A NEON 实现 */
    uint32_t X1 = _mm_cvtsi128_si32(_mm_shuffle_epi32(a, 0x55));
    // 将 a 的第一个 32 位整数转换为 32 位整数 X1，并通过 _mm_shuffle_epi32 函数进行位移操作
    uint32_t X3 = _mm_cvtsi128_si32(_mm_shuffle_epi32(a, 0xFF));
    // 将 a 的第一个 32 位整数转换为 32 位整数 X3，并通过 _mm_shuffle_epi32 函数进行位移操作
    for (int i = 0; i < 4; ++i) {
        // 循环 4 次
        ((uint8_t *) &X1)[i] = _sse2neon_sbox[((uint8_t *) &X1)[i]];
        // 将 X1 的第 i 个字节转换为 uint8_t 类型，并通过 _sse2neon_sbox 查找对应的值进行替换
        ((uint8_t *) &X3)[i] = _sse2neon_sbox[((uint8_t *) &X3)[i]];
        // 将 X3 的第 i 个字节转换为 uint8_t 类型，并通过 _sse2neon_sbox 查找对应的值进行替换
    }
    return _mm_set_epi32(((X3 >> 8) | (X3 << 24)) ^ rcon, X3,
                         ((X1 >> 8) | (X1 << 24)) ^ rcon, X1);
    // 将 X3 和 X1 进行位移和异或操作，并通过 _mm_set_epi32 函数返回结果
#endif
}
#undef SSE2NEON_AES_SBOX
#undef SSE2NEON_AES_RSBOX

#if defined(__aarch64__)
#undef SSE2NEON_XT
#undef SSE2NEON_MULTIPLY
#endif

#else /* __ARM_FEATURE_CRYPTO */
#else /* __ARM_FEATURE_CRYPTO */
// Implements equivalent of 'aesenc' by combining AESE (with an empty key) and
// AESMC and then manually applying the real key as an xor operation. This
// unfortunately means an additional xor op; the compiler should be able to
// optimize this away for repeated calls however. See
// https://blog.michaelbrase.com/2018/05/08/emulating-x86-aes-intrinsics-on-armv8-a
// for more details.
// 通过组合 AESE（使用空密钥）和 AESMC 来实现 'aesenc' 的等效操作，然后手动将真实密钥作为异或操作应用。这意味着额外的异或操作；然而，编译器应该能够优化掉重复调用的情况。更多详情请参见 https://blog.michaelbrase.com/2018/05/08/emulating-x86-aes-intrinsics-on-armv8-a
FORCE_INLINE __m128i _mm_aesenc_si128(__m128i a, __m128i b)
// 执行 'aesenc' 的一个轮次，使用 a 中的数据（状态）和 RoundKey 中的轮密钥，并将结果存储在 dst 中。
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesdec_si128
{
    return vreinterpretq_m128i_u8(veorq_u8(
        vaesmcq_u8(vaeseq_u8(vreinterpretq_u8_m128i(a), vdupq_n_u8(0))),
        vreinterpretq_u8_m128i(b)));
}

// Perform one round of an AES decryption flow on data (state) in a using the
// round key in RoundKey, and store the result in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesdec_si128
// 在 a 中使用 RoundKey 中的轮密钥对数据（状态）执行 AES 解密流程的一个轮次，并将结果存储在 dst 中。

FORCE_INLINE __m128i _mm_aesdec_si128(__m128i a, __m128i RoundKey)
// 执行一个 AES 解密流程的轮次，使用 a 中的数据（状态）和 RoundKey 中的轮密钥，并将结果存储在 dst 中。
{
    return vreinterpretq_m128i_u8(veorq_u8(
        vaesimcq_u8(vaesdq_u8(vreinterpretq_u8_m128i(a), vdupq_n_u8(0))),
        vreinterpretq_u8_m128i(RoundKey)));
}

// Perform the last round of an AES encryption flow on data (state) in a using
// the round key in RoundKey, and store the result in dst.
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesenclast_si128
// 在 a 中使用 RoundKey 中的轮密钥对数据（状态）执行 AES 加密流程的最后一轮，并将结果存储在 dst 中。
// 使用AES指令集对输入的128位整数a和RoundKey执行最后一轮AES加密，并返回结果
FORCE_INLINE __m128i _mm_aesenclast_si128(__m128i a, __m128i RoundKey)
{
    return _mm_xor_si128(vreinterpretq_m128i_u8(vaeseq_u8(
                             vreinterpretq_u8_m128i(a), vdupq_n_u8(0))),
                         RoundKey);
}

// 在数据（state）a上执行AES解密流的最后一轮，使用RoundKey中的轮密钥，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesdeclast_si128
FORCE_INLINE __m128i _mm_aesdeclast_si128(__m128i a, __m128i RoundKey)
{
    return vreinterpretq_m128i_u8(
        veorq_u8(vaesdq_u8(vreinterpretq_u8_m128i(a), vdupq_n_u8(0)),
                 vreinterpretq_u8_m128i(RoundKey)));
}

// 对a执行InvMixColumns转换，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aesimc_si128
FORCE_INLINE __m128i _mm_aesimc_si128(__m128i a)
{
    return vreinterpretq_m128i_u8(vaesimcq_u8(vreinterpretq_u8_m128i(a)));
}

// 通过计算来自a和imm8中指定的8位轮常数的数据，协助扩展AES密码密钥的生成步骤，生成用于加密密码的轮密钥，并将结果存储在dst中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_aeskeygenassist_si128
FORCE_INLINE __m128i _mm_aeskeygenassist_si128(__m128i a, const int rcon)
{
    // AESE对A执行ShiftRows和SubBytes
    uint8x16_t u8 = vaeseq_u8(vreinterpretq_u8_m128i(a), vdupq_n_u8(0));

#ifndef _MSC_VER
    uint8x16_t dest = {
        // 撤消AESE的ShiftRows步骤并提取X1和X3
        u8[0x4], u8[0x1], u8[0xE], u8[0xB],  // SubBytes(X1)
        u8[0x1], u8[0xE], u8[0xB], u8[0x4],  // ROT(SubBytes(X1))
        u8[0xC], u8[0x9], u8[0x6], u8[0x3],  // SubBytes(X3)
        u8[0x9], u8[0x6], u8[0x3], u8[0xC],  // ROT(SubBytes(X3))
    };
    uint32x4_t r = {0, (unsigned) rcon, 0, (unsigned) rcon};
    # 将dest强制转换为128位整数向量，并与r进行按位异或运算
    return vreinterpretq_m128i_u8(dest) ^ vreinterpretq_m128i_u32(r);
#else
    // 如果不满足条件，则执行以下代码块
    // 这里使用了一个技巧，因为 MSVC 严格遵循 CPP 标准，特别是 C++03 8.5.1 子节 15，规定联合体必须由其第一个成员类型初始化。

    // 根据 Windows ARM64 ABI，它始终是小端序，所以这里可以这样做
    __n128 dest{
        ((uint64_t) u8.n128_u8[0x4] << 0) | ((uint64_t) u8.n128_u8[0x1] << 8) |
            ((uint64_t) u8.n128_u8[0xE] << 16) |
            ((uint64_t) u8.n128_u8[0xB] << 24) |
            ((uint64_t) u8.n128_u8[0x1] << 32) |
            ((uint64_t) u8.n128_u8[0xE] << 40) |
            ((uint64_t) u8.n128_u8[0xB] << 48) |
            ((uint64_t) u8.n128_u8[0x4] << 56),
        ((uint64_t) u8.n128_u8[0xC] << 0) | ((uint64_t) u8.n128_u8[0x9] << 8) |
            ((uint64_t) u8.n128_u8[0x6] << 16) |
            ((uint64_t) u8.n128_u8[0x3] << 24) |
            ((uint64_t) u8.n128_u8[0x9] << 32) |
            ((uint64_t) u8.n128_u8[0x6] << 40) |
            ((uint64_t) u8.n128_u8[0x3] << 48) |
            ((uint64_t) u8.n128_u8[0xC] << 56)};

    // 对 dest 的第二个 64 位整数执行异或运算
    dest.n128_u32[1] = dest.n128_u32[1] ^ rcon;
    // 对 dest 的第四个 64 位整数执行异或运算
    dest.n128_u32[3] = dest.n128_u32[3] ^ rcon;

    // 返回 dest
    return dest;
#endif
}
#endif

/* Others */

// 对两个 64 位整数执行无进位乘法，根据 imm8 从 a 和 b 中选择，并将结果存储在 dst 中
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_clmulepi64_si128
FORCE_INLINE __m128i _mm_clmulepi64_si128(__m128i _a, __m128i _b, const int imm)
{
    // 将 _a 转换为 uint64x2_t 类型
    uint64x2_t a = vreinterpretq_u64_m128i(_a);
    // 将 _b 转换为 uint64x2_t 类型
    uint64x2_t b = vreinterpretq_u64_m128i(_b);
    // 根据 imm & 0x11 的值进行不同的处理
    switch (imm & 0x11) {
    case 0x00:
        // 返回 vget_low_u64(a) 和 vget_low_u64(b) 的乘积
        return vreinterpretq_m128i_u64(
            _sse2neon_vmull_p64(vget_low_u64(a), vget_low_u64(b)));
    case 0x01:
        // 返回 vget_high_u64(a) 和 vget_low_u64(b) 的乘积
        return vreinterpretq_m128i_u64(
            _sse2neon_vmull_p64(vget_high_u64(a), vget_low_u64(b)));
    # 根据给定的 case 值进行不同的操作
    case 0x10:
        # 将 a 的低位 64 位和 b 的高位 64 位进行 64 位乘法运算，并将结果转换为 m128i 类型
        return vreinterpretq_m128i_u64(
            _sse2neon_vmull_p64(vget_low_u64(a), vget_high_u64(b)));
    case 0x11:
        # 将 a 的高位 64 位和 b 的高位 64 位进行 64 位乘法运算，并将结果转换为 m128i 类型
        return vreinterpretq_m128i_u64(
            _sse2neon_vmull_p64(vget_high_u64(a), vget_high_u64(b)));
    default:
        # 如果 case 值不是 0x10 或 0x11，则终止程序
        abort();
    }
// 获取 SSE2NEON_MM_DENORMALS_ZERO_MODE 的值
FORCE_INLINE unsigned int _sse2neon_mm_get_denormals_zero_mode(void)
{
    // 定义联合体，用于存储 fpcr_bitfield 字段
    union {
        fpcr_bitfield field;
#if defined(__aarch64__) || defined(_M_ARM64)
        uint64_t value;
#else
        uint32_t value;
#endif
    } r;

    // 根据架构类型获取 fpcr 寄存器的值
#if defined(__aarch64__) || defined(_M_ARM64)
    r.value = _sse2neon_get_fpcr();
#else
    __asm__ __volatile__("vmrs %0, FPSCR" : "=r"(r.value)); /* read */
#endif

    // 返回 denormals zero 模式的值
    return r.field.bit24 ? _MM_DENORMALS_ZERO_ON : _MM_DENORMALS_ZERO_OFF;
}

// 计算无符号 32 位整数 a 中置为 1 的位数，并将结果返回
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_popcnt_u32
FORCE_INLINE int _mm_popcnt_u32(unsigned int a)
{
    // 根据架构类型选择不同的实现方式
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用内置函数或者特定编译器的内置函数进行计算
    #if __has_builtin(__builtin_popcount)
        return __builtin_popcount(a);
    #elif defined(_MSC_VER)
        return _CountOneBits(a);
    #else
        return (int) vaddlv_u8(vcnt_u8(vcreate_u8((uint64_t) a)));
    #endif
#else
    // 使用 NEON 指令进行计算
    uint32_t count = 0;
    uint8x8_t input_val, count8x8_val;
    uint16x4_t count16x4_val;
    uint32x2_t count32x2_val;

    input_val = vld1_u8((uint8_t *) &a);
    count8x8_val = vcnt_u8(input_val);
    count16x4_val = vpaddl_u8(count8x8_val);
    count32x2_val = vpaddl_u16(count16x4_val);

    vst1_u32(&count, count32x2_val);
    return count;
#endif
}

// 计算无符号 64 位整数 a 中置为 1 的位数，并将结果返回
// 参考链接：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=_mm_popcnt_u64
FORCE_INLINE int64_t _mm_popcnt_u64(uint64_t a)
{
    // 根据架构类型选择不同的实现方式
#if defined(__aarch64__) || defined(_M_ARM64)
    // 使用内置函数或者特定编译器的内置函数进行计算
    #if __has_builtin(__builtin_popcountll)
        return __builtin_popcountll(a);
    #elif defined(_MSC_VER)
        return _CountOneBits64(a);
    #else
        return (int64_t) vaddlv_u8(vcnt_u8(vcreate_u8(a)));
    #endif
#else
    // 使用 NEON 指令进行计算
    uint64_t count = 0;
    uint8x8_t input_val, count8x8_val;
    uint16x4_t count16x4_val;
    uint32x2_t count32x2_val;
    uint64x1_t count64x1_val;
    # 从内存地址 &a 处加载 8 个无符号 8 位整数到寄存器中
    input_val = vld1_u8((uint8_t *) &a);
    # 计算输入寄存器中每个字节的位数为 1 的个数
    count8x8_val = vcnt_u8(input_val);
    # 将输入寄存器中的 8 个 8 位整数相邻两两相加，得到 4 个 16 位整数
    count16x4_val = vpaddl_u8(count8x8_val);
    # 将输入寄存器中的 4 个 16 位整数相邻两两相加，得到 2 个 32 位整数
    count32x2_val = vpaddl_u16(count16x4_val);
    # 将输入寄存器中的 2 个 32 位整数相加，得到一个 64 位整数
    count64x1_val = vpaddl_u32(count32x2_val);
    # 将结果写入内存地址 &count 处
    vst1_u64(&count, count64x1_val);
    # 返回结果
    return count;
#endif
}

// 设置 SSE2NEON_MM_SET_DENORMALS_ZERO_MODE 函数，用于设置 denormals zero 模式
FORCE_INLINE void _sse2neon_mm_set_denormals_zero_mode(unsigned int flag)
{
    // 定义一个联合体，用于存储 FPCR 寄存器的值
    union {
        fpcr_bitfield field;
#if defined(__aarch64__) || defined(_M_ARM64)
        uint64_t value;
#else
        uint32_t value;
#endif
    } r;

#if defined(__aarch64__) || defined(_M_ARM64)
    // 获取 FPCR 寄存器的值
    r.value = _sse2neon_get_fpcr();
#else
    // 从 FPSCR 寄存器中读取值
    __asm__ __volatile__("vmrs %0, FPSCR" : "=r"(r.value)); /* read */
#endif

    // 根据传入的 flag 值判断是否开启 denormals zero 模式
    r.field.bit24 = (flag & _MM_DENORMALS_ZERO_MASK) == _MM_DENORMALS_ZERO_ON;

#if defined(__aarch64__) || defined(_M_ARM64)
    // 设置 FPCR 寄存器的值
    _sse2neon_set_fpcr(r.value);
#else
    // 将值写入 FPSCR 寄存器
    __asm__ __volatile__("vmsr FPSCR, %0" ::"r"(r));        /* write */
#endif
}

// 返回处理器时间戳计数器的当前 64 位值
// https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html#text=rdtsc
FORCE_INLINE uint64_t _rdtsc(void)
{
#if defined(__aarch64__) || defined(_M_ARM64)
    uint64_t val;

    /* 根据 ARM DDI 0487F.c，从 Armv8.0 到 Armv8.5（含）版本，系统计数器至少为 56 位宽；
     * 从 Armv8.6 版本开始，计数器必须为 64 位宽。因此，系统计数器可能小于 64 位宽，
     * 并且如果 'cap_user_time_short' 标志为 true，则其被归属为 'cap_user_time_short'。
     */
#if defined(_MSC_VER)
    // 使用 _ReadStatusReg 函数获取 ARM64_SYSREG(3, 3, 14, 0, 2) 寄存器的值
    val = _ReadStatusReg(ARM64_SYSREG(3, 3, 14, 0, 2));
#else
    // 从 cntvct_el0 寄存器中读取值
    __asm__ __volatile__("mrs %0, cntvct_el0" : "=r"(val));
#endif

    // 返回计数器的值
    return val;
#else
    uint32_t pmccntr, pmuseren, pmcntenset;
    // 读取用户模式性能监控单元（PMU）的用户使能寄存器（PMUSERENR）的访问权限
    __asm__ __volatile__("mrc p15, 0, %0, c9, c14, 0" : "=r"(pmuseren));
    // 如果 pmuseren 的最低位为 1，则允许读取用户模式代码的 PMUSERENR
    if (pmuseren & 1) {  // Allows reading PMUSERENR for user mode code.
        // 从性能监控寄存器中读取计数器使能集合
        __asm__ __volatile__("mrc p15, 0, %0, c9, c12, 1" : "=r"(pmcntenset));
        // 如果计数器正在计数
        if (pmcntenset & 0x80000000UL) {  // Is it counting?
            // 从性能监控计数器中读取计数值
            __asm__ __volatile__("mrc p15, 0, %0, c9, c13, 0" : "=r"(pmccntr));
            // 计数器设置为每 64 个周期计数一次
            return (uint64_t) (pmccntr) << 6;
        }
    }

    // 作为回退，因为我们无法在用户模式下启用 PMUSERENR，所以使用系统调用
    // 获取当前时间
    struct timeval tv;
    gettimeofday(&tv, NULL);
    // 返回当前时间的微秒数
    return (uint64_t) (tv.tv_sec) * 1000000 + tv.tv_usec;
#endif
}

#if defined(__GNUC__) || defined(__clang__)
#pragma pop_macro("ALIGN_STRUCT")
#pragma pop_macro("FORCE_INLINE")
#endif

#if defined(__GNUC__) && !defined(__clang__)
#pragma GCC pop_options
#endif

#endif



#endif

这是一个条件编译指令的结束标记。


}

#if defined(__GNUC__) || defined(__clang__)
#pragma pop_macro("ALIGN_STRUCT")
#pragma pop_macro("FORCE_INLINE")
#endif

如果编译器是`GNUC`或`clang`，则执行以下操作：
- 取消之前使用`#pragma push_macro`保存的`ALIGN_STRUCT`宏定义
- 取消之前使用`#pragma push_macro`保存的`FORCE_INLINE`宏定义


#if defined(__GNUC__) && !defined(__clang__)
#pragma GCC pop_options
#endif

如果编译器是`GNUC`且不是`clang`，则执行以下操作：
- 取消之前使用`#pragma GCC push_options`保存的编译选项


#endif

这是一个条件编译指令的结束标记。
```