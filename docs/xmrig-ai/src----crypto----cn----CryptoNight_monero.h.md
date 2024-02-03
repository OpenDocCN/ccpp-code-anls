# `xmrig\src\crypto\cn\CryptoNight_monero.h`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2018 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018      SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2019 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据 GNU 通用公共许可证的条款发布，
 *   由自由软件基金会发布的版本 3 或
 *   （在您的选择下）任何以后的版本。
 *
 *   本程序是希望它有用：
 *   但没有任何保证；甚至没有暗示的保证
 *   适销性或特定用途的保证。更多细节请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CRYPTONIGHT_MONERO_H
#define XMRIG_CRYPTONIGHT_MONERO_H

#include <fenv.h>
#include <math.h>

// VARIANT ALTERATIONS
#ifndef XMRIG_ARM
#   define VARIANT1_INIT(part) \
    uint64_t tweak1_2_##part = 0; \
    if (BASE == Algorithm::CN_1) { \
        tweak1_2_##part = (*reinterpret_cast<const uint64_t*>(input + 35 + part * size) ^ \
                          *(reinterpret_cast<const uint64_t*>(ctx[part]->state) + 24)); \
    }
#else
#   define VARIANT1_INIT(part) \
    uint64_t tweak1_2_##part = 0; \
    # 如果基础算法是CN_1，则执行以下操作
    if (BASE == Algorithm::CN_1) { \
        # 从输入数据中复制tweak1_2_##part的内容到指定位置
        memcpy(&tweak1_2_##part, input + 35 + part * size, sizeof tweak1_2_##part); \
        # 对tweak1_2_##part和ctx[part]->state中的第24个uint64_t进行异或操作
        tweak1_2_##part ^= *(reinterpret_cast<const uint64_t*>(ctx[part]->state) + 24); \
    }
#endif

#define VARIANT1_1(p) \  // 定义宏 VARIANT1_1，接受参数 p
    if (BASE == Algorithm::CN_1) { \  // 如果 BASE 等于 Algorithm::CN_1
        const uint8_t tmp = reinterpret_cast<const uint8_t*>(p)[11]; \  // 将参数 p 转换为 uint8_t 指针，取第 11 个元素赋值给 tmp
        static const uint32_t table = 0x75310; \  // 定义静态常量 table 为 0x75310
        const uint8_t index = (((tmp >> 3) & 6) | (tmp & 1)) << 1; \  // 计算 index
        ((uint8_t*)(p))[11] = tmp ^ ((table >> index) & 0x30); \  // 修改参数 p 的第 11 个元素
    }

#define VARIANT1_2(p, part) \  // 定义宏 VARIANT1_2，接受参数 p 和 part
    if (BASE == Algorithm::CN_1) { \  // 如果 BASE 等于 Algorithm::CN_1
        (p) ^= tweak1_2_##part; \  // 对参数 p 进行异或操作
    }


#ifndef XMRIG_ARM  // 如果未定义 XMRIG_ARM
#   define VARIANT2_INIT(part) \  // 定义宏 VARIANT2_INIT，接受参数 part
    __m128i division_result_xmm_##part = _mm_cvtsi64_si128(static_cast<int64_t>(h##part[12])); \  // 定义 __m128i 类型的变量 division_result_xmm_##part，并赋值
    __m128i sqrt_result_xmm_##part     = _mm_cvtsi64_si128(static_cast<int64_t>(h##part[13])); \  // 定义 __m128i 类型的变量 sqrt_result_xmm_##part，并赋值

#ifdef _MSC_VER  // 如果定义了 _MSC_VER
#   define VARIANT2_SET_ROUNDING_MODE() if (BASE == Algorithm::CN_2) { _control87(RC_DOWN, MCW_RC); }  // 定义宏 VARIANT2_SET_ROUNDING_MODE，根据条件设置舍入模式
#   define RESTORE_ROUNDING_MODE() _control87(RC_NEAR, MCW_RC);  // 定义宏 RESTORE_ROUNDING_MODE，恢复舍入模式
#else  // 如果未定义 _MSC_VER
#   define VARIANT2_SET_ROUNDING_MODE() if (BASE == Algorithm::CN_2) { fesetround(FE_DOWNWARD); }  // 定义宏 VARIANT2_SET_ROUNDING_MODE，根据条件设置舍入模式
#   define RESTORE_ROUNDING_MODE() fesetround(FE_TONEAREST);  // 定义宏 RESTORE_ROUNDING_MODE，恢复舍入模式
#endif

#   define VARIANT2_INTEGER_MATH(part, cl, cx) \  // 定义宏 VARIANT2_INTEGER_MATH，接受参数 part, cl, cx
    do { \  // 执行以下操作
        const uint64_t sqrt_result = static_cast<uint64_t>(_mm_cvtsi128_si64(sqrt_result_xmm_##part)); \  // 将 sqrt_result_xmm_##part 转换为 uint64_t 类型
        const uint64_t cx_0 = _mm_cvtsi128_si64(cx); \  // 将 cx 转换为 uint64_t 类型
        cl ^= static_cast<uint64_t>(_mm_cvtsi128_si64(division_result_xmm_##part)) ^ (sqrt_result << 32); \  // 对 cl 进行异或操作
        const uint32_t d = static_cast<uint32_t>(cx_0 + (sqrt_result << 1)) | 0x80000001UL; \  // 计算 d
        const uint64_t cx_1 = _mm_cvtsi128_si64(_mm_srli_si128(cx, 8)); \  // 将 cx 右移 8 位，并转换为 uint64_t 类型
        const uint64_t division_result = static_cast<uint32_t>(cx_1 / d) + ((cx_1 % d) << 32); \  // 计算 division_result
        division_result_xmm_##part = _mm_cvtsi64_si128(static_cast<int64_t>(division_result)); \  // 将 division_result 转换为 __m128i 类型
        sqrt_result_xmm_##part = int_sqrt_v2(cx_0 + division_result); \  // 调用 int_sqrt_v2 函数
    } while (0)  // 结束 do-while 循环

#   define VARIANT2_SHUFFLE(base_ptr, offset, _a, _b, _b1, _c, reverse) \  // 定义宏 VARIANT2_SHUFFLE，接受参数 base_ptr, offset, _a, _b, _b1, _c, reverse
    // 从内存中加载128位数据到寄存器中，根据偏移量和reverse条件选择不同的地址
    const __m128i chunk1 = _mm_load_si128((__m128i *)((base_ptr) + ((offset) ^ (reverse ? 0x30 : 0x10))));
    // 从内存中加载128位数据到寄存器中，根据偏移量选择不同的地址
    const __m128i chunk2 = _mm_load_si128((__m128i *)((base_ptr) + ((offset) ^ 0x20)));
    // 从内存中加载128位数据到寄存器中，根据偏移量和reverse条件选择不同的地址
    const __m128i chunk3 = _mm_load_si128((__m128i *)((base_ptr) + ((offset) ^ (reverse ? 0x10 : 0x30)));
    // 将寄存器中的数据加上_b1后存回内存，根据偏移量选择不同的地址
    _mm_store_si128((__m128i *)((base_ptr) + ((offset) ^ 0x10)), _mm_add_epi64(chunk3, _b1));
    // 将寄存器中的数据加上_b后存回内存，根据偏移量选择不同的地址
    _mm_store_si128((__m128i *)((base_ptr) + ((offset) ^ 0x20)), _mm_add_epi64(chunk1, _b));
    // 将寄存器中的数据加上_a后存回内存，根据偏移量选择不同的地址
    _mm_store_si128((__m128i *)((base_ptr) + ((offset) ^ 0x30)), _mm_add_epi64(chunk2, _a));
    // 如果算法为CN_R，则进行一系列位运算操作
    if (ALGO == Algorithm::CN_R) {
        _c = _mm_xor_si128(_mm_xor_si128(_c, chunk3), _mm_xor_si128(chunk1, chunk2));
    }
# 定义一个宏，用于对指定内存地址的数据进行一系列操作
# 参数包括 base_ptr: 基地址指针, offset: 偏移量, _a, _b, _b1, hi, lo, reverse: 一些参数
# 使用 do { ... } while (0) 结构，确保宏展开后的代码块可以作为一个整体使用
# 在代码块内部，进行一系列的位操作和数据加载、存储操作
# 最后的 VARIANT2_INIT 和 VARIANT2_INTEGER_MATH 是另外两个宏的定义，不在本代码块内
    # 使用宏定义的参数 part 进行循环计算
    do { \
        # 将 128 位寄存器 cx 转换为 64 位整数
        const uint64_t cx_0 = _mm_cvtsi128_si64(cx); \
        # 对 cl 进行异或操作，使用预定义的 division_result_##part 和 sqrt_result_##part 进行计算
        cl ^= division_result_##part ^ (sqrt_result_##part << 32); \
        # 计算 d 的值，使用 static_cast 进行类型转换
        const uint32_t d = static_cast<uint32_t>(cx_0 + (sqrt_result_##part << 1)) | 0x80000001UL; \
        # 将 128 位寄存器 cx 右移 8 位，并转换为 64 位整数
        const uint64_t cx_1 = _mm_cvtsi128_si64(_mm_srli_si128(cx, 8)); \
        # 计算 division_result_##part 的值，使用 static_cast 进行类型转换
        division_result_##part = static_cast<uint32_t>(cx_1 / d) + ((cx_1 % d) << 32); \
        # 计算 sqrt_input 的值
        const uint64_t sqrt_input = cx_0 + division_result_##part; \
        # 计算 sqrt_result_##part 的值
        sqrt_result_##part = sqrt(sqrt_input + 18446744073709551616.0) * 2.0 - 8589934592.0; \
        # 计算 s 的值
        const uint64_t s = sqrt_result_##part >> 1; \
        # 计算 b 的值
        const uint64_t b = sqrt_result_##part & 1; \
        # 计算 r2 的值
        const uint64_t r2 = (uint64_t)(s) * (s + b) + (sqrt_result_##part << 32); \
        # 更新 sqrt_result_##part 的值
        sqrt_result_##part += ((r2 + b > sqrt_input) ? -1 : 0) + ((r2 + (1ULL << 32) < sqrt_input - s) ? 1 : 0); \
    } while (0)
# 定义一个宏，用于对给定内存地址的数据进行特定的操作
# 参数包括 base_ptr: 内存地址指针, offset: 偏移量, _a, _b, _b1, _c: 用于操作的数据，reverse: 是否反转操作
# 使用 SIMD 指令加载指定地址的数据到 chunk1
const uint64x2_t chunk1 = vld1q_u64((uint64_t*)((base_ptr) + ((offset) ^ (reverse ? 0x30 : 0x10))));
# 使用 SIMD 指令加载指定地址的数据到 chunk2
const uint64x2_t chunk2 = vld1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x20)));
# 使用 SIMD 指令加载指定地址的数据到 chunk3
const uint64x2_t chunk3 = vld1q_u64((uint64_t*)((base_ptr) + ((offset) ^ (reverse ? 0x10 : 0x30))));
# 使用 SIMD 指令将 chunk3 和 _b1 相加，并存储到指定地址
vst1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x10)), vaddq_u64(chunk3, vreinterpretq_u64_u8(_b1)));
# 使用 SIMD 指令将 chunk1 和 _b 相加，并存储到指定地址
vst1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x20)), vaddq_u64(chunk1, vreinterpretq_u64_u8(_b)));
# 使用 SIMD 指令将 chunk2 和 _a 相加，并存储到指定地址
vst1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x30)), vaddq_u64(chunk2, vreinterpretq_u64_u8(_a)));
# 如果算法为 CN_R，则对 _c 进行特定的异或操作
if (ALGO == Algorithm::CN_R) {
    _c = veorq_u64(veorq_u64(_c, chunk3), veorq_u64(chunk1, chunk2));
}
    // 从指定地址偏移 offset^0x10 处加载两个 64 位整数，与指定的 hi 和 lo 组合成一个 128 位整数
    const uint64x2_t chunk1 = veorq_u64(vld1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x10))), vcombine_u64(vcreate_u64(hi), vcreate_u64(lo)));
    // 从指定地址偏移 offset^0x20 处加载两个 64 位整数
    const uint64x2_t chunk2 = vld1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x20)));
    // 对 hi 和 lo 进行异或操作
    hi ^= ((uint64_t*)((base_ptr) + ((offset) ^ 0x20)))[0];
    lo ^= ((uint64_t*)((base_ptr) + ((offset) ^ 0x20)))[1];
    // 从指定地址偏移 offset^0x30 处加载两个 64 位整数
    const uint64x2_t chunk3 = vld1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x30)));
    // 如果 reverse 为真，则将 chunk1 和 chunk3 与 _b1 和 _b 进行加法操作后存储到指定地址偏移 offset^0x10 和 offset^0x20 处
    if (reverse) {
        vst1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x10)), vaddq_u64(chunk1, vreinterpretq_u64_u8(_b1)));
        vst1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x20)), vaddq_u64(chunk3, vreinterpretq_u64_u8(_b)));
    } else {
        // 如果 reverse 为假，则将 chunk3 和 chunk1 与 _b1 和 _b 进行加法操作后存储到指定地址偏移 offset^0x10 和 offset^0x20 处
        vst1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x10)), vaddq_u64(chunk3, vreinterpretq_u64_u8(_b1)));
        vst1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x20)), vaddq_u64(chunk1, vreinterpretq_u64_u8(_b)));
    }
    // 将 chunk2 与 _a 进行加法操作后存储到指定地址偏移 offset^0x30 处
    vst1q_u64((uint64_t*)((base_ptr) + ((offset) ^ 0x30)), vaddq_u64(chunk2, vreinterpretq_u64_u8(_a));
#endif

#define SWAP32LE(x) x  // 定义一个宏，用于将32位整数转换为小端字节序
#define SWAP64LE(x) x  // 定义一个宏，用于将64位整数转换为小端字节序
#define hash_extra_blake(data, length, hash) blake256_hash((uint8_t*)(hash), (uint8_t*)(data), (length))  // 定义一个函数宏，用于计算额外的布莱克哈希值

#ifndef NOINLINE  // 如果未定义 NOINLINE
#ifdef __GNUC__  // 如果是 GNU 编译器
#define NOINLINE __attribute__ ((noinline))  // 定义一个宏，用于指示函数不进行内联优化
#elif _MSC_VER  // 如果是 Microsoft 编译器
#define NOINLINE __declspec(noinline)  // 定义一个宏，用于指示函数不进行内联优化
#else  // 其它情况
#define NOINLINE  // 定义一个空的宏
#endif
#endif

#include "crypto/cn/r/variant4_random_math.h"  // 包含 variant4_random_math.h 文件

#define VARIANT4_RANDOM_MATH_INIT(part) \  // 定义一个宏，用于初始化 VARIANT4 随机数运算
  uint32_t r##part[9]; \  // 定义一个包含9个32位整数的数组
  struct V4_Instruction code##part[256]; \  // 定义一个包含256个 V4_Instruction 结构体的数组
  if (props.isR()) { \  // 如果 props.isR() 返回 true
    r##part[0] = static_cast<uint32_t>(h##part[12]); \  // 将 h##part[12] 转换为32位整数并赋值给 r##part[0]
    r##part[1] = static_cast<uint32_t>(h##part[12] >> 32); \  // 将 h##part[12] 右移32位后转换为32位整数并赋值给 r##part[1]
    r##part[2] = static_cast<uint32_t>(h##part[13]); \  // 将 h##part[13] 转换为32位整数并赋值给 r##part[2]
    r##part[3] = static_cast<uint32_t>(h##part[13] >> 32); \  // 将 h##part[13] 右移32位后转换为32位整数并赋值给 r##part[3]
    v4_random_math_init<ALGO>(code##part, height); \  // 调用 v4_random_math_init 函数进行初始化
  }

#define VARIANT4_RANDOM_MATH(part, al, ah, cl, bx0, bx1) \  // 定义一个宏，用于进行 VARIANT4 随机数运算
  if (props.isR()) { \  // 如果 props.isR() 返回 true
    cl ^= (r##part[0] + r##part[1]) | (static_cast<uint64_t>(r##part[2] + r##part[3]) << 32); \  // 对 cl 进行异或运算
    r##part[4] = static_cast<uint32_t>(al); \  // 将 al 转换为32位整数并赋值给 r##part[4]
    r##part[5] = static_cast<uint32_t>(ah); \  // 将 ah 转换为32位整数并赋值给 r##part[5]
    r##part[6] = static_cast<uint32_t>(_mm_cvtsi128_si32(bx0)); \  // 将 bx0 转换为32位整数并赋值给 r##part[6]
    r##part[7] = static_cast<uint32_t>(_mm_cvtsi128_si32(bx1)); \  // 将 bx1 转换为32位整数并赋值给 r##part[7]
    r##part[8] = static_cast<uint32_t>(_mm_cvtsi128_si32(_mm_srli_si128(bx1, 8))); \  // 将 bx1 右移8位后转换为32位整数并赋值给 r##part[8]
    v4_random_math(code##part, r##part); \  // 调用 v4_random_math 函数进行随机数运算
  }

extern bool cn_sse41_enabled;  // 声明一个外部的布尔变量 cn_sse41_enabled
extern bool cn_vaes_enabled;  // 声明一个外部的布尔变量 cn_vaes_enabled

#endif /* XMRIG_CRYPTONIGHT_MONERO_H */  // 结束条件编译指令
```