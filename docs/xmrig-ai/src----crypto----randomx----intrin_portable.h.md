# `xmrig\src\crypto\randomx\intrin_portable.h`

```
/*
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>

保留所有权利。

在源代码和二进制形式下，无论是否修改，只要满足以下条件，都可以重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式下，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者“按原样”提供，不提供任何明示或暗示的担保，
包括但不限于对适销性和特定用途的适用性的暗示担保。
在任何情况下，无论是合同、严格责任还是侵权行为（包括疏忽或其他原因），
都不承担任何直接、间接、偶然、特殊、惩罚性或后果性的损害赔偿责任，
包括但不限于替代商品或服务的采购、使用、数据或利润损失或业务中断，
即使事先被告知此类损害的可能性。
*/

#pragma once

#include <cstdint>
#include "crypto/randomx/blake2/endian.h"

// 将无符号32位整数转换为有符号的2的补码表示
constexpr int32_t unsigned32ToSigned2sCompl(uint32_t x) {
    // 如果-1等于~0，则返回x的有符号表示，否则根据x的大小返回对应的有符号表示
    return (-1 == ~0) ? (int32_t)x : (x > INT32_MAX ? (-(int32_t)(UINT32_MAX - x) - 1) : (int32_t)x);
}

// 将无符号64位整数转换为有符号的2的补码表示
constexpr int64_t unsigned64ToSigned2sCompl(uint64_t x) {
    // 如果-1等于~0，则返回x的有符号表示，否则根据x的大小返回对应的有符号表示
    return (-1 == ~0) ? (int64_t)x : (x > INT64_MAX ? (-(int64_t)(UINT64_MAX - x) - 1) : (int64_t)x);
}

// 将32位无符号整数进行符号扩展为64位有符号整数
constexpr uint64_t signExtend2sCompl(uint32_t x) {
    # 如果 (-1 == ~0) 为真，则返回 (int64_t)(int32_t)(x)，否则执行下一步
    # (-1 == ~0) 判断是否为真，即判断 -1 是否等于按位取反的 0
    # 如果为真，将 x 转换为 int32_t 类型，再转换为 int64_t 类型，并返回结果
    # 如果为假，执行下一步
    # 判断 x 是否大于 INT32_MAX
    # 如果大于 INT32_MAX，则将 x 转换为 uint64_t 类型，并将高 32 位设置为 0xffffffff，返回结果
    # 如果小于等于 INT32_MAX，则将 x 转换为 uint64_t 类型，并返回结果
// 定义常量，用于四舍五入
constexpr int RoundToNearest = 0;
// 定义常量，用于向下取整
constexpr int RoundDown = 1;
// 定义常量，用于向上取整
constexpr int RoundUp = 2;
// 定义常量，用于向零取整
constexpr int RoundToZero = 3;

// 如果编译器不定义 __SSE2__，但是 SSE2 可用，手动定义 __SSE2__
#if !defined(__SSE2__) && (defined(_M_X64) || (defined(_M_IX86_FP) && _M_IX86_FP == 2))
#define __SSE2__ 1
#endif

// 如果编译器是 MSVC，并且定义了 __SSE2__，则定义 __AES__
#if defined(_MSC_VER) && defined(__SSE2__)
#define __AES__
#endif

// MSVC 提供的 x86 目标的库函数 "sqrt" 不能给出正确的结果，因此我们必须使用内联汇编直接调用 x87 fsqrt
#if !defined(__SSE2__)
#if defined(_M_IX86)
// 定义一个内联函数，用于调用 x87 fsqrt
inline double __cdecl rx_sqrt(double x) {
    __asm {
        fld x
        fsqrt
    }
}
#define rx_sqrt rx_sqrt

// 设置双精度浮点数精度
void rx_set_double_precision();
#define RANDOMX_USE_X87

#elif defined(__i386)

// 设置双精度浮点数精度
void rx_set_double_precision();
#define RANDOMX_USE_X87

#endif
#endif //__SSE2__

// 如果没有定义 rx_sqrt，则使用 sqrt
#if !defined(rx_sqrt)
#define rx_sqrt sqrt
#endif

// 如果没有定义 RANDOMX_USE_X87，则定义 rx_set_double_precision(x)
#if !defined(RANDOMX_USE_X87)
#define rx_set_double_precision(x)
#endif

#ifdef __SSE2__
#ifdef __GNUC__
#include <x86intrin.h>
#else
#include <intrin.h>
#endif

// 定义类型别名
typedef __m128i rx_vec_i128;
typedef __m128d rx_vec_f128;

// 定义宏，用于对齐内存分配
#define rx_aligned_alloc(a, b) _mm_malloc(a,b)
// 定义宏，用于对齐内存释放
#define rx_aligned_free(a) _mm_free(a)
// 定义宏，用于预取非临近数据
#define rx_prefetch_nta(x) _mm_prefetch((const char *)(x), _MM_HINT_NTA)
// 定义宏，用于预取 T0 级别数据
#define rx_prefetch_t0(x) _mm_prefetch((const char *)(x), _MM_HINT_T0)

// 定义宏，用于加载 __m128d 类型的数据
#define rx_load_vec_f128 _mm_load_pd
// 定义宏，用于存储 __m128d 类型的数据
#define rx_store_vec_f128 _mm_store_pd
// 定义宏，用于两个 __m128d 类型的数据相加
#define rx_add_vec_f128 _mm_add_pd
// 定义宏，用于两个 __m128d 类型的数据相减
#define rx_sub_vec_f128 _mm_sub_pd
// 定义宏，用于两个 __m128d 类型的数据相乘
#define rx_mul_vec_f128 _mm_mul_pd
// 定义宏，用于两个 __m128d 类型的数据相除
#define rx_div_vec_f128 _mm_div_pd
// 定义宏，用于对 __m128d 类型的数据进行平方根运算
#define rx_sqrt_vec_f128 _mm_sqrt_pd

// 定义内联函数，用于交换 __m128d 类型的数据
FORCE_INLINE rx_vec_f128 rx_swap_vec_f128(rx_vec_f128 a) {
    return _mm_shuffle_pd(a, a, 1);
}

// 定义内联函数，用于设置 __m128d 类型的数据
FORCE_INLINE rx_vec_f128 rx_set_vec_f128(uint64_t x1, uint64_t x0) {
    return _mm_castsi128_pd(_mm_set_epi64x(x1, x0));
}

// 定义内联函数，用于设置 __m128d 类型的数据，所有元素都相同
FORCE_INLINE rx_vec_f128 rx_set1_vec_f128(uint64_t x) {
    return _mm_castsi128_pd(_mm_set1_epi64x(x));
}
// 定义将两个双精度浮点数向量进行按位异或操作的宏
#define rx_xor_vec_f128 _mm_xor_pd
// 定义将两个双精度浮点数向量进行按位与操作的宏
#define rx_and_vec_f128 _mm_and_pd
// 定义将两个整型向量进行按位与操作的宏
#define rx_and_vec_i128 _mm_and_si128
// 定义将两个双精度浮点数向量进行按位或操作的宏
#define rx_or_vec_f128 _mm_or_pd

#ifdef __AES__
// 如果支持 AES 指令集，则定义将两个整型向量进行 AES 加密操作的宏
#define rx_aesenc_vec_i128 _mm_aesenc_si128
// 如果支持 AES 指令集，则定义将两个整型向量进行 AES 解密操作的宏
#define rx_aesdec_vec_i128 _mm_aesdec_si128
// 定义支持 AES 指令集的标识符
#define HAVE_AES
#endif //__AES__

// 定义将整型向量转换为整型数的函数
FORCE_INLINE int rx_vec_i128_x(rx_vec_i128 a) {
    return _mm_cvtsi128_si32(a);
}

// 定义将整型向量转换为整型数的函数
FORCE_INLINE int rx_vec_i128_y(rx_vec_i128 a) {
    return _mm_cvtsi128_si32(_mm_shuffle_epi32(a, 0x55));
}

// 定义将整型向量转换为整型数的函数
FORCE_INLINE int rx_vec_i128_z(rx_vec_i128 a) {
    return _mm_cvtsi128_si32(_mm_shuffle_epi32(a, 0xaa));
}

// 定义将整型向量转换为整型数的函数
FORCE_INLINE int rx_vec_i128_w(rx_vec_i128 a) {
    return _mm_cvtsi128_si32(_mm_shuffle_epi32(a, 0xff));
}

// 定义设置整型向量的函数
#define rx_set_int_vec_i128 _mm_set_epi32
// 定义将两个整型向量进行按位异或操作的宏
#define rx_xor_vec_i128 _mm_xor_si128
// 定义加载整型向量的函数
#define rx_load_vec_i128 _mm_load_si128
// 定义存储整型向量的函数
#define rx_store_vec_i128 _mm_store_si128

// 定义将内存中的整型数据转换为双精度浮点数向量的函数
FORCE_INLINE rx_vec_f128 rx_cvt_packed_int_vec_f128(const void* addr) {
    __m128i ix = _mm_loadl_epi64((const __m128i*)addr);
    return _mm_cvtepi32_pd(ix);
}

// 定义默认的浮点数状态寄存器值
constexpr uint32_t rx_mxcsr_default = 0x9FC0; //Flush to zero, denormals are zero, default rounding mode, all exceptions disabled

// 定义重置浮点数状态的函数
FORCE_INLINE void rx_reset_float_state() {
    _mm_setcsr(rx_mxcsr_default);
}

// 定义设置舍入模式的函数
FORCE_INLINE void rx_set_rounding_mode(uint32_t mode) {
    _mm_setcsr(rx_mxcsr_default | (mode << 13));
}

#elif defined(__PPC64__) && defined(__ALTIVEC__) && defined(__VSX__) //sadly only POWER7 and newer will be able to use SIMD acceleration. Earlier processors cant use doubles or 64 bit integers with SIMD
#include <cstdint>
#include <stdexcept>
#include <cstdlib>
#include <altivec.h>
#undef vector
#undef pixel
#undef bool

// 定义 128 位整型向量类型
typedef __vector uint8_t __m128i;
typedef __vector uint32_t __m128l;
typedef __vector int      __m128li;
typedef __vector uint64_t __m128ll;
// 定义 128 位双精度浮点数向量类型
typedef __vector double __m128d;

// 定义整型向量类型别名
typedef __m128i rx_vec_i128;
// 定义双精度浮点数向量类型别名
typedef __m128d rx_vec_f128;
// 定义联合体类型
typedef union{
    # 声明一个名为i的128位整数向量
    rx_vec_i128 i;
    # 声明一个名为d的128位浮点数向量
    rx_vec_f128 d;
    # 声明一个包含两个64位无符号整数的数组
    uint64_t u64[2];
    # 声明一个包含两个双精度浮点数的数组
    double   d64[2];
    # 声明一个包含四个32位无符号整数的数组
    uint32_t u32[4];
    # 声明一个包含四个32位整数的数组
    int i32[4];
} vec_u;

#define rx_aligned_alloc(a, b) malloc(a)  // 定义宏，用于分配对齐内存
#define rx_aligned_free(a) free(a)  // 定义宏，用于释放对齐内存
#define rx_prefetch_nta(x)  // 定义宏，用于非临时数据预取
#define rx_prefetch_t0(x)  // 定义宏，用于T0级别数据预取

/* Splat 64-bit long long to 2 64-bit long longs */
FORCE_INLINE __m128i vec_splat2sd (int64_t scalar)
{ return (__m128i) vec_splats (scalar); }  // 定义函数，将64位长整型扩展为2个64位长整型

FORCE_INLINE rx_vec_f128 rx_load_vec_f128(const double* pd) {
#if defined(NATIVE_LITTLE_ENDIAN)
    return (rx_vec_f128)vec_vsx_ld(0,pd);  // 如果是小端序，使用vec_vsx_ld加载数据
#else
    vec_u t;  // 定义联合体t
    t.u64[0] = load64(pd + 0);  // 将pd+0处的64位数据加载到t的第一个成员
    t.u64[1] = load64(pd + 1);  // 将pd+1处的64位数据加载到t的第二个成员
    return (rx_vec_f128)t.d;  // 返回t的双精度浮点数成员
#endif
}

FORCE_INLINE void rx_store_vec_f128(double* mem_addr, rx_vec_f128 a) {
#if defined(NATIVE_LITTLE_ENDIAN)
    vec_vsx_st(a,0,(rx_vec_f128*)mem_addr);  // 如果是小端序，使用vec_vsx_st存储数据
#else
    vec_u _a;  // 定义联合体_a
    _a.d = a;  // 将a赋值给_a的双精度浮点数成员
    store64(mem_addr + 0, _a.u64[0]);  // 将_a的第一个成员存储到mem_addr+0处
    store64(mem_addr + 1, _a.u64[1]);  // 将_a的第二个成员存储到mem_addr+1处
#endif
}

FORCE_INLINE rx_vec_f128 rx_swap_vec_f128(rx_vec_f128 a) {
    return (rx_vec_f128)vec_perm((__m128i)a,(__m128i)a,(__m128i){8,9,10,11,12,13,14,15,0,1,2,3,4,5,6,7});  // 交换a的高低位
}

FORCE_INLINE rx_vec_f128 rx_add_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    return (rx_vec_f128)vec_add(a,b);  // 返回a和b的加法结果
}

FORCE_INLINE rx_vec_f128 rx_sub_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    return (rx_vec_f128)vec_sub(a,b);  // 返回a和b的减法结果
}

FORCE_INLINE rx_vec_f128 rx_mul_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    return (rx_vec_f128)vec_mul(a,b);  // 返回a和b的乘法结果
}

FORCE_INLINE rx_vec_f128 rx_div_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    return (rx_vec_f128)vec_div(a,b);  // 返回a和b的除法结果
}

FORCE_INLINE rx_vec_f128 rx_sqrt_vec_f128(rx_vec_f128 a) {
    return (rx_vec_f128)vec_sqrt(a);  // 返回a的平方根
}

FORCE_INLINE rx_vec_i128 rx_set1_long_vec_i128(uint64_t a) {
    return (rx_vec_i128)vec_splat2sd(a);  // 返回a的扩展成2个64位长整型的结果
}

FORCE_INLINE rx_vec_f128 rx_vec_i128_vec_f128(rx_vec_i128 a) {
    return (rx_vec_f128)a;  // 返回a的双精度浮点数形式
}

FORCE_INLINE rx_vec_f128 rx_set_vec_f128(uint64_t x1, uint64_t x0) {
    return (rx_vec_f128)(__m128ll){x0,x1};  // 返回由x1和x0组成的双精度浮点数
}

FORCE_INLINE rx_vec_f128 rx_set1_vec_f128(uint64_t x) {
    return (rx_vec_f128)vec_splat2sd(x);  // 返回x扩展成2个64位长整型的结果
}
# 定义一个函数，用于将两个 rx_vec_f128 类型的向量进行异或操作，并返回结果向量
FORCE_INLINE rx_vec_f128 rx_xor_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    return (rx_vec_f128)vec_xor(a,b);
}

# 定义一个函数，用于将两个 rx_vec_f128 类型的向量进行与操作，并返回结果向量
FORCE_INLINE rx_vec_f128 rx_and_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    return (rx_vec_f128)vec_and(a,b);
}

# 定义一个函数，用于将两个 rx_vec_i128 类型的向量进行与操作，并返回结果向量
FORCE_INLINE rx_vec_i128 rx_and_vec_i128(rx_vec_i128 a, rx_vec_i128 b) {
    return (rx_vec_i128)vec_and(a, b);
}

# 定义一个函数，用于将两个 rx_vec_f128 类型的向量进行或操作，并返回结果向量
FORCE_INLINE rx_vec_f128 rx_or_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    return (rx_vec_f128)vec_or(a,b);
}

# 如果定义了 __CRYPTO__ 宏
#if defined(__CRYPTO__)

    # 定义一个函数，用于将 __m128i 类型的向量进行反转，并返回结果向量
    FORCE_INLINE __m128ll vrev(__m128i v){
        # 如果是小端序，使用 vec_perm 函数进行反转
        # 否则，使用 vec_perm 函数进行反转
    }

    # 定义一个函数，用于将 rx_vec_i128 类型的向量和 rx_vec_i128 类型的密钥进行 AES 加密操作，并返回结果向量
    FORCE_INLINE rx_vec_i128 rx_aesenc_vec_i128(rx_vec_i128 v, rx_vec_i128 rkey) {
        # 将输入向量和密钥进行反转
        # 使用 __builtin_crypto_vcipher 函数进行 AES 加密操作
        # 将结果向量进行反转，并返回结果
    }

    # 定义一个函数，用于将 rx_vec_i128 类型的向量和 rx_vec_i128 类型的密钥进行 AES 解密操作，并返回结果向量
    FORCE_INLINE rx_vec_i128 rx_aesdec_vec_i128(rx_vec_i128 v, rx_vec_i128 rkey) {
        # 将输入向量和零向量进行反转
        # 使用 __builtin_crypto_vncipher 函数进行 AES 解密操作
        # 将结果向量和密钥进行异或操作，并返回结果
    }

    # 定义宏 HAVE_AES

#endif //__CRYPTO__

# 定义一个函数，用于获取 rx_vec_i128 类型向量的 x 分量
FORCE_INLINE int rx_vec_i128_x(rx_vec_i128 a) {
    # 将向量转换为 vec_u 类型
    # 返回向量的第一个元素
}

# 定义一个函数，用于获取 rx_vec_i128 类型向量的 y 分量
FORCE_INLINE int rx_vec_i128_y(rx_vec_i128 a) {
    # 将向量转换为 vec_u 类型
    # 返回向量的第二个元素
}

# 定义一个函数，用于获取 rx_vec_i128 类型向量的 z 分量
FORCE_INLINE int rx_vec_i128_z(rx_vec_i128 a) {
    # 将向量转换为 vec_u 类型
    # 返回向量的第三个元素
}

# 定义一个函数，用于获取 rx_vec_i128 类型向量的 w 分量
FORCE_INLINE int rx_vec_i128_w(rx_vec_i128 a) {
    # 将向量转换为 vec_u 类型
    # 返回向量的第四个元素
}

# 定义一个函数，用于创建一个 rx_vec_i128 类型的向量，根据给定的四个整数值
FORCE_INLINE rx_vec_i128 rx_set_int_vec_i128(int _I3, int _I2, int _I1, int _I0) {
    return (rx_vec_i128)((__m128li){_I0,_I1,_I2,_I3});
};

# 定义一个函数，用于将两个 rx_vec_i128 类型的向量进行异或操作，并返回结果向量
FORCE_INLINE rx_vec_i128 rx_xor_vec_i128(rx_vec_i128 _A, rx_vec_i128 _B) {
    return (rx_vec_i128)vec_xor(_A,_B);
}
# 定义一个函数，用于加载 rx_vec_i128 类型的数据
FORCE_INLINE rx_vec_i128 rx_load_vec_i128(rx_vec_i128 const *_P) {
    # 如果是小端字节序，直接返回 _P
    # 否则，将 _P 转换为 uint32_t* 类型的指针
    # 创建一个 vec_u 类型的变量 c
    # 将 _P 指针指向的数据按照 32 位进行加载，并分别存储到 c.u32 数组中
    # 将 c.i 转换为 rx_vec_i128 类型并返回
}

# 定义一个函数，用于存储 rx_vec_i128 类型的数据
FORCE_INLINE void rx_store_vec_i128(rx_vec_i128 *_P, rx_vec_i128 _B) {
    # 如果是小端字节序，直接将 _B 存储到 _P
    # 否则，将 _P 转换为 uint32_t* 类型的指针
    # 创建一个 vec_u 类型的变量 B
    # 将 _B 转换为 vec_u 类型的数据，并存储到 B 中
    # 将 B.u32 数组中的数据按照 32 位进行存储到 _P 指针指向的位置
}

# 定义一个函数，用于将地址 addr 处的数据转换为 rx_vec_f128 类型的数据
FORCE_INLINE rx_vec_f128 rx_cvt_packed_int_vec_f128(const void* addr) {
    # 创建一个 vec_u 类型的变量 x
    # 将地址 addr 处的数据按照 32 位进行加载，并转换为有符号的 2 的补码表示
    # 将转换后的数据分别存储到 x.d64 数组中
    # 将 x.d 转换为 rx_vec_f128 类型并返回
}

# 定义一个宏，用于设置默认的浮点环境
#define RANDOMX_DEFAULT_FENV

#elif defined(__aarch64__)

#include <stdlib.h>
#include <arm_neon.h>
#include <arm_acle.h>

# 定义 rx_vec_i128 类型为 uint8x16_t
# 定义 rx_vec_f128 类型为 float64x2_t

# 定义一个函数，用于按照指定的对齐方式分配内存
inline void* rx_aligned_alloc(size_t size, size_t align) {
    # 创建一个指针 p
    # 如果使用 posix_memalign 函数成功分配内存，则返回 p
    # 否则，返回 0
};

# 定义一个宏，用于释放通过 rx_aligned_alloc 函数分配的内存
#define rx_aligned_free(a) free(a)

# 定义一个函数，用于预取非临近数据
inline void rx_prefetch_nta(void* ptr) {
    # 使用汇编指令 prfm 预取 ptr 指针指向的数据
}

# 定义一个函数，用于预取 T0 级别的数据
inline void rx_prefetch_t0(const void* ptr) {
    # 使用汇编指令 prfm 预取 ptr 指针指向的数据
}

# 定义一个函数，用于加载 rx_vec_f128 类型的数据
FORCE_INLINE rx_vec_f128 rx_load_vec_f128(const double* pd) {
    # 使用 arm_neon.h 中的 vld1q_f64 函数加载 pd 指针指向的数据，并返回结果
}

# 定义一个函数，用于存储 rx_vec_f128 类型的数据
FORCE_INLINE void rx_store_vec_f128(double* mem_addr, rx_vec_f128 val) {
    # 使用 arm_neon.h 中的 vst1q_f64 函数将 val 存储到 mem_addr 指针指向的位置
}

# 定义一个函数，用于交换 rx_vec_f128 类型的数据
FORCE_INLINE rx_vec_f128 rx_swap_vec_f128(rx_vec_f128 a) {
    # 创建一个 float64x2_t 类型的变量 temp，并初始化为 0
    # 将 a 的第一个元素复制到 temp 的第二个元素
    # 将 a 的第一个元素复制到 a 的第二个元素
    # 将 temp 的第二个元素复制到 temp 的第一个元素
    # 返回交换后的 a
}
# 定义一个函数，用于将两个64位整数转换为128位浮点数向量
FORCE_INLINE rx_vec_f128 rx_set_vec_f128(uint64_t x1, uint64_t x0) {
    # 使用x0创建一个64位整数向量temp0
    uint64x2_t temp0 = vdupq_n_u64(x0);
    # 使用x1创建一个64位整数向量temp1
    uint64x2_t temp1 = vdupq_n_u64(x1);
    # 将temp0的第1个元素和temp1的第0个元素复制到一个新的128位浮点数向量中，并返回该向量
    return vreinterpretq_f64_u64(vcopyq_laneq_u64(temp0, 1, temp1, 0));
}

# 定义一个函数，用于将一个64位整数复制到128位浮点数向量中的每个元素
FORCE_INLINE rx_vec_f128 rx_set1_vec_f128(uint64_t x) {
    # 将x复制到一个64位整数向量中，并将该向量转换为128位浮点数向量后返回
    return vreinterpretq_f64_u64(vdupq_n_u64(x));
}

# 定义宏，用于将两个128位浮点数向量相加
#define rx_add_vec_f128 vaddq_f64

# 定义宏，用于将两个128位浮点数向量相减
#define rx_sub_vec_f128 vsubq_f64

# 定义宏，用于将两个128位浮点数向量相乘
#define rx_mul_vec_f128 vmulq_f64

# 定义宏，用于将两个128位浮点数向量相除
#define rx_div_vec_f128 vdivq_f64

# 定义宏，用于将128位浮点数向量开平方
#define rx_sqrt_vec_f128 vsqrtq_f64

# 定义一个函数，用于将两个128位浮点数向量进行按位异或操作
FORCE_INLINE rx_vec_f128 rx_xor_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    # 将a和b分别转换为64位整数向量，并进行按位异或操作后返回
    return vreinterpretq_f64_u8(veorq_u8(vreinterpretq_u8_f64(a), vreinterpretq_u8_f64(b)));
}

# 定义一个函数，用于将两个128位浮点数向量进行按位与操作
FORCE_INLINE rx_vec_f128 rx_and_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    # 将a和b分别转换为64位整数向量，并进行按位与操作后返回
    return vreinterpretq_f64_u8(vandq_u8(vreinterpretq_u8_f64(a), vreinterpretq_u8_f64(b)));
}

# 定义宏，用于将两个128位整数向量进行按位与操作
#define rx_and_vec_i128 vandq_u8

# 定义一个函数，用于将两个128位浮点数向量进行按位或操作
FORCE_INLINE rx_vec_f128 rx_or_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    # 将a和b分别转换为64位整数向量，并进行按位或操作后返回
    return vreinterpretq_f64_u8(vorrq_u8(vreinterpretq_u8_f64(a), vreinterpretq_u8_f64(b)));
}

# 如果支持ARM加密扩展，则定义以下函数和宏
#ifdef __ARM_FEATURE_CRYPTO

# 定义一个函数，用于将两个128位整数向量进行AES加密操作
FORCE_INLINE rx_vec_i128 rx_aesenc_vec_i128(rx_vec_i128 a, rx_vec_i128 key) {
    # 定义一个全零的128位整数向量
    const uint8x16_t zero = { 0 };
    # 将a和zero分别进行AES加密操作，并进行中间结果的混淆后与key进行按位异或操作后返回
    return vaesmcq_u8(vaeseq_u8(a, zero)) ^ key;
}

# 定义一个函数，用于将两个128位整数向量进行AES解密操作
FORCE_INLINE rx_vec_i128 rx_aesdec_vec_i128(rx_vec_i128 a, rx_vec_i128 key) {
    # 定义一个全零的128位整数向量
    const uint8x16_t zero = { 0 };
    # 将a和zero分别进行AES解密操作，并进行中间结果的混淆后与key进行按位异或操作后返回
    return vaesimcq_u8(vaesdq_u8(a, zero)) ^ key;
}

# 定义宏，表示支持AES加密
#define HAVE_AES

#endif

# 定义宏，用于将两个128位整数向量进行按位异或操作
#define rx_xor_vec_i128 veorq_u8

# 定义一个函数，用于获取128位整数向量a的第0个元素
FORCE_INLINE int rx_vec_i128_x(rx_vec_i128 a) {
    # 将a转换为32位有符号整数向量，并获取第0个元素后返回
    return vgetq_lane_s32(vreinterpretq_s32_u8(a), 0);
}

# 定义一个函数，用于获取128位整数向量a的第1个元素
FORCE_INLINE int rx_vec_i128_y(rx_vec_i128 a) {
    # 将a转换为32位有符号整数向量，并获取第1个元素后返回
    return vgetq_lane_s32(vreinterpretq_s32_u8(a), 1);
}

# 定义一个函数，用于获取128位整数向量a的第2个元素
FORCE_INLINE int rx_vec_i128_z(rx_vec_i128 a) {
    # 将a转换为32位有符号整数向量，并获取第2个元素后返回
    return vgetq_lane_s32(vreinterpretq_s32_u8(a), 2);
}

# 定义一个函数，用于获取128位整数向量a的第3个元素
FORCE_INLINE int rx_vec_i128_w(rx_vec_i128 a) {
    # 将a转换为32位有符号整数向量，并获取第3个元素后返回
    return vgetq_lane_s32(vreinterpretq_s32_u8(a), 3);
}

# 定义一个函数，用于将四个整数转换为128位整数向量
FORCE_INLINE rx_vec_i128 rx_set_int_vec_i128(int _I3, int _I2, int _I1, int _I0) {
    # 定义一个包含四个整数的数组
    int32_t data[4];
    # 将_I0的值赋给data列表的第一个元素
    data[0] = _I0;
    # 将_I1的值赋给data列表的第二个元素
    data[1] = _I1;
    # 将_I2的值赋给data列表的第三个元素
    data[2] = _I2;
    # 将_I3的值赋给data列表的第四个元素
    data[3] = _I3;
    # 将data列表中的四个元素作为参数传入vld1q_s32函数，将结果转换为无符号8位整数向量
    return vreinterpretq_u8_s32(vld1q_s32(data));
};

#define rx_xor_vec_i128 veorq_u8

FORCE_INLINE rx_vec_i128 rx_load_vec_i128(const rx_vec_i128* mem_addr) {
    // 从内存地址加载 rx_vec_i128 类型的数据
    return vld1q_u8((const uint8_t*)mem_addr);
}

FORCE_INLINE void rx_store_vec_i128(rx_vec_i128* mem_addr, rx_vec_i128 val) {
    // 将 rx_vec_i128 类型的数据存储到内存地址
    vst1q_u8((uint8_t*)mem_addr, val);
}

FORCE_INLINE rx_vec_f128 rx_cvt_packed_int_vec_f128(const void* addr) {
    // 从地址中加载数据，转换成 rx_vec_f128 类型
    double lo = unsigned32ToSigned2sCompl(load32((uint8_t*)addr + 0));
    double hi = unsigned32ToSigned2sCompl(load32((uint8_t*)addr + 4));
    rx_vec_f128 x{};
    x = vsetq_lane_f64(lo, x, 0);
    x = vsetq_lane_f64(hi, x, 1);
    return x;
}

#define RANDOMX_DEFAULT_FENV

#else //portable fallback

#include <cstdint>
#include <stdexcept>
#include <cstdlib>
#include <cmath>

typedef union {
    uint64_t u64[2];
    uint32_t u32[4];
    uint16_t u16[8];
    uint8_t u8[16];
} rx_vec_i128;

typedef union {
    struct {
        double lo;
        double hi;
    };
    rx_vec_i128 i;
} rx_vec_f128;

#define rx_aligned_alloc(a, b) malloc(a)
#define rx_aligned_free(a) free(a)
#define rx_prefetch_nta(x)
#define rx_prefetch_t0(x)

FORCE_INLINE rx_vec_f128 rx_load_vec_f128(const double* pd) {
    // 从内存地址加载 rx_vec_f128 类型的数据
    rx_vec_f128 x;
    x.i.u64[0] = load64(pd + 0);
    x.i.u64[1] = load64(pd + 1);
    return x;
}

FORCE_INLINE void rx_store_vec_f128(double* mem_addr, rx_vec_f128 a) {
    // 将 rx_vec_f128 类型的数据存储到内存地址
    store64(mem_addr + 0, a.i.u64[0]);
    store64(mem_addr + 1, a.i.u64[1]);
}

FORCE_INLINE rx_vec_f128 rx_swap_vec_f128(rx_vec_f128 a) {
    // 交换 rx_vec_f128 类型数据的高低位
    double temp = a.hi;
    a.hi = a.lo;
    a.lo = temp;
    return a;
}

FORCE_INLINE rx_vec_f128 rx_add_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    // 对两个 rx_vec_f128 类型的数据进行加法运算
    rx_vec_f128 x;
    x.lo = a.lo + b.lo;
    x.hi = a.hi + b.hi;
    return x;
}

FORCE_INLINE rx_vec_f128 rx_sub_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    // 对两个 rx_vec_f128 类型的数据进行减法运算
    rx_vec_f128 x;
    x.lo = a.lo - b.lo;
    x.hi = a.hi - b.hi;
    return x;
}

FORCE_INLINE rx_vec_f128 rx_mul_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    // 对两个 rx_vec_f128 类型的数据进行乘法运算
    rx_vec_f128 x;
    x.lo = a.lo * b.lo;
    x.hi = a.hi * b.hi;
    # 返回变量 x 的值
    return x;
# 定义一个函数，实现两个128位向量的除法运算
FORCE_INLINE rx_vec_f128 rx_div_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    # 定义一个128位向量x
    rx_vec_f128 x;
    # 将a.lo和b.lo对应位置的元素进行除法运算，结果存入x.lo
    x.lo = a.lo / b.lo;
    # 将a.hi和b.hi对应位置的元素进行除法运算，结果存入x.hi
    x.hi = a.hi / b.hi;
    # 返回结果向量x
    return x;
}

# 定义一个函数，实现128位向量的平方根运算
FORCE_INLINE rx_vec_f128 rx_sqrt_vec_f128(rx_vec_f128 a) {
    # 定义一个128位向量x
    rx_vec_f128 x;
    # 对a.lo中的元素进行平方根运算，结果存入x.lo
    x.lo = rx_sqrt(a.lo);
    # 对a.hi中的元素进行平方根运算，结果存入x.hi
    x.hi = rx_sqrt(a.hi);
    # 返回结果向量x
    return x;
}

# 定义一个函数，将64位整数a扩展成128位整数向量
FORCE_INLINE rx_vec_i128 rx_set1_long_vec_i128(uint64_t a) {
    # 定义一个128位整数向量x
    rx_vec_i128 x;
    # 将a的值存入x的低64位和高64位
    x.u64[0] = a;
    x.u64[1] = a;
    # 返回结果向量x
    return x;
}

# 定义一个函数，将128位整数向量a转换成128位浮点数向量
FORCE_INLINE rx_vec_f128 rx_vec_i128_vec_f128(rx_vec_i128 a) {
    # 定义一个128位浮点数向量x
    rx_vec_f128 x;
    # 将a赋值给x
    x.i = a;
    # 返回结果向量x
    return x;
}

# 定义一个函数，设置128位浮点数向量的值
FORCE_INLINE rx_vec_f128 rx_set_vec_f128(uint64_t x1, uint64_t x0) {
    # 定义一个128位浮点数向量v
    rx_vec_f128 v;
    # 将x0和x1分别存入v的低64位和高64位
    v.i.u64[0] = x0;
    v.i.u64[1] = x1;
    # 返回结果向量v
    return v;
}

# 定义一个函数，设置128位浮点数向量的所有元素为x
FORCE_INLINE rx_vec_f128 rx_set1_vec_f128(uint64_t x) {
    # 定义一个128位浮点数向量v
    rx_vec_f128 v;
    # 将x的值存入v的低64位和高64位
    v.i.u64[0] = x;
    v.i.u64[1] = x;
    # 返回结果向量v
    return v;
}

# 定义一个函数，实现128位浮点数向量的按位异或运算
FORCE_INLINE rx_vec_f128 rx_xor_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    # 定义一个128位浮点数向量x
    rx_vec_f128 x;
    # 将a和b对应位置的元素进行按位异或运算，结果存入x
    x.i.u64[0] = a.i.u64[0] ^ b.i.u64[0];
    x.i.u64[1] = a.i.u64[1] ^ b.i.u64[1];
    # 返回结果向量x
    return x;
}

# 定义一个函数，实现128位浮点数向量的按位与运算
FORCE_INLINE rx_vec_f128 rx_and_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    # 定义一个128位浮点数向量x
    rx_vec_f128 x;
    # 将a和b对应位置的元素进行按位与运算，结果存入x
    x.i.u64[0] = a.i.u64[0] & b.i.u64[0];
    x.i.u64[1] = a.i.u64[1] & b.i.u64[1];
    # 返回结果向量x
    return x;
}

# 定义一个函数，实现128位整数向量的按位与运算
FORCE_INLINE rx_vec_i128 rx_and_vec_i128(rx_vec_i128 a, rx_vec_i128 b) {
    # 定义一个128位整数向量x
    rx_vec_i128 x;
    # 将a和b对应位置的元素进行按位与运算，结果存入x
    x.u64[0] = a.u64[0] & b.u64[0];
    x.u64[1] = a.u64[1] & b.u64[1];
    # 返回结果向量x
    return x;
}

# 定义一个函数，实现128位浮点数向量的按位或运算
FORCE_INLINE rx_vec_f128 rx_or_vec_f128(rx_vec_f128 a, rx_vec_f128 b) {
    # 定义一个128位浮点数向量x
    rx_vec_f128 x;
    # 将a和b对应位置的元素进行按位或运算，结果存入x
    x.i.u64[0] = a.i.u64[0] | b.i.u64[0];
    x.i.u64[1] = a.i.u64[1] | b.i.u64[1];
    # 返回结果向量x
    return x;
}

# 定义一个函数，返回128位整数向量a的第一个元素
FORCE_INLINE int rx_vec_i128_x(rx_vec_i128 a) {
    # 返回a的第一个元素
    return a.u32[0];
}

# 定义一个函数，返回128位整数向量a的第二个元素
FORCE_INLINE int rx_vec_i128_y(rx_vec_i128 a) {
    # 返回a的第二个元素
    return a.u32[1];
}

# 定义一个函数，返回128位整数向量a的第三个元素
FORCE_INLINE int rx_vec_i128_z(rx_vec_i128 a) {
    # 返回a的第三个元素
    return a.u32[2];
}

# 定义一个函数，返回128位整数向量a的第四个元素
FORCE_INLINE int rx_vec_i128_w(rx_vec_i128 a) {
    # 返回a的第四个元素
    return a.u32[3];
}

# 定义一个函数，设置128位整数向量的值
FORCE_INLINE rx_vec_i128 rx_set_int_vec_i128(int _I3, int _I2, int _I1, int _I0) {
    # 定义一个128位整数向量v
    rx_vec_i128 v;
    # 将_I0、_I1、_I2、_I3分别存入v的四个元素
    v.u32[0] = _I0;
    # 将_I1的值赋给v的第一个32位整数
    v.u32[1] = _I1;
    # 将_I2的值赋给v的第二个32位整数
    v.u32[2] = _I2;
    # 将_I3的值赋给v的第三个32位整数
    v.u32[3] = _I3;
    # 返回v
    return v;
# 定义一个函数，用于将两个128位整数进行异或操作，并返回结果
def rx_xor_vec_i128(_A, _B):
    # 创建一个新的128位整数对象
    c = rx_vec_i128()
    # 将_A和_B的每个32位整数进行异或操作，并将结果赋值给c的对应位置
    c.u32[0] = _A.u32[0] ^ _B.u32[0]
    c.u32[1] = _A.u32[1] ^ _B.u32[1]
    c.u32[2] = _A.u32[2] ^ _B.u32[2]
    c.u32[3] = _A.u32[3] ^ _B.u32[3]
    # 返回结果c
    return c

# 定义一个函数，用于从指针_P指向的内存地址加载一个128位整数
def rx_load_vec_i128(_P):
    # 如果是小端字节序，直接返回指针_P指向的128位整数对象
    if defined(NATIVE_LITTLE_ENDIAN):
        return *_P
    # 如果不是小端字节序，将指针_P转换为32位整数指针
    ptr = (uint32_t*)_P
    # 创建一个新的128位整数对象
    c = rx_vec_i128()
    # 从ptr指向的内存地址加载每个32位整数，并将结果赋值给c的对应位置
    c.u32[0] = load32(ptr + 0)
    c.u32[1] = load32(ptr + 1)
    c.u32[2] = load32(ptr + 2)
    c.u32[3] = load32(ptr + 3)
    # 返回结果c
    return c

# 定义一个函数，用于将一个128位整数存储到指针_P指向的内存地址
def rx_store_vec_i128(_P, _B):
    # 如果是小端字节序，直接将_B赋值给指针_P指向的128位整数对象
    if defined(NATIVE_LITTLE_ENDIAN):
        *_P = _B
    # 如果不是小端字节序，将指针_P转换为32位整数指针
    ptr = (uint32_t*)_P
    # 将_B的每个32位整数存储到ptr指向的内存地址的对应位置
    store32(ptr + 0, _B.u32[0])
    store32(ptr + 1, _B.u32[1])
    store32(ptr + 2, _B.u32[2])
    store32(ptr + 3, _B.u32[3])

# 定义一个函数，用于将一个指针指向的内存地址中的数据转换为一个128位浮点数
def rx_cvt_packed_int_vec_f128(addr):
    # 创建一个新的128位浮点数对象
    x = rx_vec_f128()
    # 将addr指向的内存地址中的前4个字节转换为有符号的32位整数，并将结果转换为双精度浮点数赋值给x的低64位
    x.lo = (double)unsigned32ToSigned2sCompl(load32((uint8_t*)addr + 0))
    # 将addr指向的内存地址中的后4个字节转换为有符号的32位整数，并将结果转换为双精度浮点数赋值给x的高64位
    x.hi = (double)unsigned32ToSigned2sCompl(load32((uint8_t*)addr + 4))
    # 返回结果x
    return x

# 定义一个常量字符串，表示平台不支持硬件AES
static const char* platformError = "Platform doesn't support hardware AES";

# 引入异常处理模块
#include <stdexcept>

# 定义一个函数，用于进行AES加密操作，如果平台不支持硬件AES，则抛出运行时错误
FORCE_INLINE rx_vec_i128 rx_aesenc_vec_i128(v, rkey):
    throw std::runtime_error(platformError)

# 定义一个函数，用于进行AES解密操作，如果平台不支持硬件AES，则抛出运行时错误
FORCE_INLINE rx_vec_i128 rx_aesdec_vec_i128(v, rkey):
    throw std::runtime_error(platformError)

# 如果使用默认的浮点环境
#ifdef RANDOMX_DEFAULT_FENV

# 重置浮点状态
void rx_reset_float_state();

# 设置舍入模式
void rx_set_rounding_mode(mode);

#endif

# 加载一个双精度浮点数
double loadDoublePortable(addr);

# 两个64位整数相乘，返回高64位结果
uint64_t mulh(uint64_t, uint64_t);

# 两个有符号的64位整数相乘，返回高64位结果
int64_t smulh(int64_t, int64_t);

# 将一个64位整数左循环移位指定的位数
uint64_t rotl64(uint64_t, unsigned int);

# 将一个64位整数右循环移位指定的位数
uint64_t rotr64(uint64_t, unsigned int);
```