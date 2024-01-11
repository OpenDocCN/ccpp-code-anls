# `xmrig\src\crypto\cn\CryptoNight_x86.h`

```
// 包含版权声明和许可证信息
/* XMRig
 * Copyright 2010      Jeff Garzik <jgarzik@pobox.com>
 * Copyright 2012-2014 pooler      <pooler@litecoinpool.org>
 * Copyright 2014      Lucas Jones <https://github.com/lucasjones>
 * Copyright 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * Copyright 2016      Jay D Dee   <jayddee246@gmail.com>
 * Copyright 2017-2019 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * Copyright 2018      Lee Clagett <https://github.com/vtnerd>
 * Copyright 2018-2020 SChernykh   <https://github.com/SChernykh>
 * Copyright 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   This program is free software: you can redistribute it and/or modify
 *   it under the terms of the GNU General Public License as published by
 *   the Free Software Foundation, either version 3 of the License, or
 *   (at your option) any later version.
 *
 *   This program is distributed in the hope that it will be useful,
 *   but WITHOUT ANY WARRANTY; without even the implied warranty of
 *   MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the
 *   GNU General Public License for more details.
 *
 *   You should have received a copy of the GNU General Public License
 *   along with this program. If not, see <http://www.gnu.org/licenses/>.
 */

// 定义条件编译，如果未定义则包含头文件
#ifndef XMRIG_CRYPTONIGHT_X86_H
#define XMRIG_CRYPTONIGHT_X86_H

// 根据不同的编译器包含不同的头文件
#ifdef __GNUC__
#   include <x86intrin.h>
#else
#   include <intrin.h>
#   define __restrict__ __restrict
#endif

// 包含其他头文件
#include "backend/cpu/Cpu.h"
#include "base/crypto/keccak.h"
#include "crypto/cn/CnAlgo.h"
#include "crypto/cn/CryptoNight_monero.h"
#include "crypto/cn/CryptoNight.h"
#include "crypto/cn/soft_aes.h"

// 根据条件编译包含不同的头文件
#ifdef XMRIG_VAES
#   include "crypto/cn/CryptoNight_x86_vaes.h"
#endif

// 包含 C 语言的头文件
extern "C"
{
#include "crypto/cn/c_groestl.h"
#include "crypto/cn/c_blake256.h"
#include "crypto/cn/c_jh.h"
#include "crypto/cn/c_skein.h"
}

// 定义静态内联函数，实现对输入数据进行 blake 哈希
static inline void do_blake_hash(const uint8_t *input, size_t len, uint8_t *output) {
    # 使用blake256算法对输入进行哈希处理，并将结果存储到输出中
    blake256_hash(output, input, len);
// 定义一个内联函数，用于执行 Groestl 哈希算法
static inline void do_groestl_hash(const uint8_t *input, size_t len, uint8_t *output) {
    groestl(input, len * 8, output);
}

// 定义一个内联函数，用于执行 JH 哈希算法
static inline void do_jh_hash(const uint8_t *input, size_t len, uint8_t *output) {
    jh_hash(32 * 8, input, 8 * len, output);
}

// 定义一个内联函数，用于执行 Skein 哈希算法
static inline void do_skein_hash(const uint8_t *input, size_t len, uint8_t *output) {
    xmr_skein(input, output);
}

// 定义一个函数指针数组，包含四种哈希算法的函数指针
void (* const extra_hashes[4])(const uint8_t *, size_t, uint8_t *) = {do_blake_hash, do_groestl_hash, do_jh_hash, do_skein_hash};

// 如果满足条件，则定义两个内联函数
#if (defined(__i386__) || defined(_M_IX86)) && !(defined(__clang__) && defined(__clang_major__) && (__clang_major__ >= 15))
static inline int64_t _mm_cvtsi128_si64(__m128i a)
{
    return ((uint64_t)(uint32_t)_mm_cvtsi128_si32(a) | ((uint64_t)(uint32_t)_mm_cvtsi128_si32(_mm_srli_si128(a, 4)) << 32));
}

static inline __m128i _mm_cvtsi64_si128(int64_t a) {
    return _mm_set_epi64x(0, a);
}
#endif

// 定义一个内联函数，用于执行 sl_xor 操作
static inline __m128i sl_xor(__m128i tmp1)
{
    __m128i tmp4;
    tmp4 = _mm_slli_si128(tmp1, 0x04);
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    tmp4 = _mm_slli_si128(tmp4, 0x04);
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    tmp4 = _mm_slli_si128(tmp4, 0x04);
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    return tmp1;
}

// 定义一个模板函数，用于生成 AES 密钥的子密钥
template<uint8_t rcon>
static inline void aes_genkey_sub(__m128i* xout0, __m128i* xout2)
{
    __m128i xout1 = _mm_aeskeygenassist_si128(*xout2, rcon);
    xout1  = _mm_shuffle_epi32(xout1, 0xFF); // see PSHUFD, set all elems to 4th elem
    *xout0 = sl_xor(*xout0);
    *xout0 = _mm_xor_si128(*xout0, xout1);
    xout1  = _mm_aeskeygenassist_si128(*xout0, 0x00);
    xout1  = _mm_shuffle_epi32(xout1, 0xAA); // see PSHUFD, set all elems to 3rd elem
    *xout2 = sl_xor(*xout2);
    *xout2 = _mm_xor_si128(*xout2, xout1);
}

// 定义一个模板函数，用于生成 AES 密钥的子密钥（软件实现）
template<uint8_t rcon>
static inline void soft_aes_genkey_sub(__m128i* xout0, __m128i* xout2)
{
    # 使用软件实现的 AES 密钥生成辅助函数生成下一个密钥
    __m128i xout1 = soft_aeskeygenassist<rcon>(*xout2);
    # 使用 PSHUFD 指令，将 xout1 中的所有元素设置为第四个元素的值
    xout1  = _mm_shuffle_epi32(xout1, 0xFF);
    # 对 xout0 进行按位异或操作
    *xout0 = sl_xor(*xout0);
    # 对 xout0 和 xout1 进行按位异或操作
    *xout0 = _mm_xor_si128(*xout0, xout1);
    # 使用软件实现的 AES 密钥生成辅助函数生成下一个密钥
    xout1  = soft_aeskeygenassist<0x00>(*xout0);
    # 使用 PSHUFD 指令，将 xout1 中的所有元素设置为第三个元素的值
    xout1  = _mm_shuffle_epi32(xout1, 0xAA);
    # 对 xout2 进行按位异或操作
    *xout2 = sl_xor(*xout2);
    # 对 xout2 和 xout1 进行按位异或操作
    *xout2 = _mm_xor_si128(*xout2, xout1);
// 为给定的模板参数 SOFT_AES，生成 AES 密钥
template<bool SOFT_AES>
static inline void aes_genkey(const __m128i* memory, __m128i* k0, __m128i* k1, __m128i* k2, __m128i* k3, __m128i* k4, __m128i* k5, __m128i* k6, __m128i* k7, __m128i* k8, __m128i* k9)
{
    // 从内存中加载数据到 xout0 和 xout2
    __m128i xout0 = _mm_load_si128(memory);
    __m128i xout2 = _mm_load_si128(memory + 1);
    // 将 xout0 和 xout2 分别赋值给 k0 和 k1
    *k0 = xout0;
    *k1 = xout2;

    // 根据 SOFT_AES 的值选择软件实现或硬件实现的 AES 密钥生成子函数
    SOFT_AES ? soft_aes_genkey_sub<0x01>(&xout0, &xout2) : aes_genkey_sub<0x01>(&xout0, &xout2);
    // 将 xout0 和 xout2 分别赋值给 k2 和 k3
    *k2 = xout0;
    *k3 = xout2;

    // 根据 SOFT_AES 的值选择软件实现或硬件实现的 AES 密钥生成子函数
    SOFT_AES ? soft_aes_genkey_sub<0x02>(&xout0, &xout2) : aes_genkey_sub<0x02>(&xout0, &xout2);
    // 将 xout0 和 xout2 分别赋值给 k4 和 k5
    *k4 = xout0;
    *k5 = xout2;

    // 根据 SOFT_AES 的值选择软件实现或硬件实现的 AES 密钥生成子函数
    SOFT_AES ? soft_aes_genkey_sub<0x04>(&xout0, &xout2) : aes_genkey_sub<0x04>(&xout0, &xout2);
    // 将 xout0 和 xout2 分别赋值给 k6 和 k7
    *k6 = xout0;
    *k7 = xout2;

    // 根据 SOFT_AES 的值选择软件实现或硬件实现的 AES 密钥生成子函数
    SOFT_AES ? soft_aes_genkey_sub<0x08>(&xout0, &xout2) : aes_genkey_sub<0x08>(&xout0, &xout2);
    // 将 xout0 和 xout2 分别赋值给 k8 和 k9
    *k8 = xout0;
    *k9 = xout2;
}

// 执行软件实现的 AES 加密
static FORCEINLINE void soft_aesenc(void* __restrict ptr, const void* __restrict key, const uint32_t* __restrict t)
{
    // 从指针 ptr 处加载 4 个 32 位整数到 x0、x1、x2、x3
    uint32_t x0 = ((const uint32_t*)(ptr))[0];
    uint32_t x1 = ((const uint32_t*)(ptr))[1];
    uint32_t x2 = ((const uint32_t*)(ptr))[2];
    uint32_t x3 = ((const uint32_t*)(ptr))[3];

    // 使用 t 表进行字节替换和轮密钥加
    uint32_t y0 = t[x0 & 0xff]; x0 >>= 8;
    uint32_t y1 = t[x1 & 0xff]; x1 >>= 8;
    uint32_t y2 = t[x2 & 0xff]; x2 >>= 8;
    uint32_t y3 = t[x3 & 0xff]; x3 >>= 8;
    t += 256;

    y0 ^= t[x1 & 0xff]; x1 >>= 8;
    y1 ^= t[x2 & 0xff]; x2 >>= 8;
    y2 ^= t[x3 & 0xff]; x3 >>= 8;
    y3 ^= t[x0 & 0xff]; x0 >>= 8;
    t += 256;

    y0 ^= t[x2 & 0xff]; x2 >>= 8;
    y1 ^= t[x3 & 0xff]; x3 >>= 8;
    y2 ^= t[x0 & 0xff]; x0 >>= 8;
    y3 ^= t[x1 & 0xff]; x1 >>= 8;
    t += 256;

    y0 ^= t[x3];
    y1 ^= t[x0];
    y2 ^= t[x1];
    y3 ^= t[x2];

    // 将加密结果与密钥进行异或操作，并写回 ptr 指向的内存
    ((uint32_t*)ptr)[0] = y0 ^ ((uint32_t*)key)[0];
    ((uint32_t*)ptr)[1] = y1 ^ ((uint32_t*)key)[1];
    ((uint32_t*)ptr)[2] = y2 ^ ((uint32_t*)key)[2];
    ((uint32_t*)ptr)[3] = y3 ^ ((uint32_t*)key)[3];
}
static FORCEINLINE __m128i soft_aesenc(const void* __restrict ptr, const __m128i key, const uint32_t* __restrict t)
{
    // 从指针中读取4个32位整数
    uint32_t x0 = ((const uint32_t*)(ptr))[0];
    uint32_t x1 = ((const uint32_t*)(ptr))[1];
    uint32_t x2 = ((const uint32_t*)(ptr))[2];
    uint32_t x3 = ((const uint32_t*)(ptr))[3];

    // 从 t 数组中查找对应的值，并进行位运算
    uint32_t y0 = t[x0 & 0xff]; x0 >>= 8;
    uint32_t y1 = t[x1 & 0xff]; x1 >>= 8;
    uint32_t y2 = t[x2 & 0xff]; x2 >>= 8;
    uint32_t y3 = t[x3 & 0xff]; x3 >>= 8;
    t += 256;

    // 从 t 数组中查找对应的值，并进行位运算
    y0 ^= t[x1 & 0xff]; x1 >>= 8;
    y1 ^= t[x2 & 0xff]; x2 >>= 8;
    y2 ^= t[x3 & 0xff]; x3 >>= 8;
    y3 ^= t[x0 & 0xff]; x0 >>= 8;
    t += 256;

    // 从 t 数组中查找对应的值，并进行位运算
    y0 ^= t[x2 & 0xff]; x2 >>= 8;
    y1 ^= t[x3 & 0xff]; x3 >>= 8;
    y2 ^= t[x0 & 0xff]; x0 >>= 8;
    y3 ^= t[x1 & 0xff]; x1 >>= 8;

    // 从 t 数组中查找对应的值，并进行位运算
    y0 ^= t[x3 + 256];
    y1 ^= t[x0 + 256];
    y2 ^= t[x1 + 256];
    y3 ^= t[x2 + 256];

    // 对结果进行异或操作
    return _mm_xor_si128(_mm_set_epi32(y3, y2, y1, y0), key);
}

template<bool SOFT_AES>
void aes_round(__m128i key, __m128i* x0, __m128i* x1, __m128i* x2, __m128i* x3, __m128i* x4, __m128i* x5, __m128i* x6, __m128i* x7);

template<>
NOINLINE void aes_round<true>(__m128i key, __m128i* x0, __m128i* x1, __m128i* x2, __m128i* x3, __m128i* x4, __m128i* x5, __m128i* x6, __m128i* x7)
{
    // 调用软件实现的 AES 加密函数
    *x0 = soft_aesenc((uint32_t*)x0, key, (const uint32_t*)saes_table);
    *x1 = soft_aesenc((uint32_t*)x1, key, (const uint32_t*)saes_table);
    *x2 = soft_aesenc((uint32_t*)x2, key, (const uint32_t*)saes_table);
    *x3 = soft_aesenc((uint32_t*)x3, key, (const uint32_t*)saes_table);
    *x4 = soft_aesenc((uint32_t*)x4, key, (const uint32_t*)saes_table);
    *x5 = soft_aesenc((uint32_t*)x5, key, (const uint32_t*)saes_table);
    *x6 = soft_aesenc((uint32_t*)x6, key, (const uint32_t*)saes_table);
    *x7 = soft_aesenc((uint32_t*)x7, key, (const uint32_t*)saes_table);
}

template<>
FORCEINLINE void aes_round<false>(__m128i key, __m128i* x0, __m128i* x1, __m128i* x2, __m128i* x3, __m128i* x4, __m128i* x5, __m128i* x6, __m128i* x7)
{
    # 使用AES加密算法对*x0进行加密，并使用key作为密钥
    *x0 = _mm_aesenc_si128(*x0, key);
    # 使用AES加密算法对*x1进行加密，并使用key作为密钥
    *x1 = _mm_aesenc_si128(*x1, key);
    # 使用AES加密算法对*x2进行加密，并使用key作为密钥
    *x2 = _mm_aesenc_si128(*x2, key);
    # 使用AES加密算法对*x3进行加密，并使用key作为密钥
    *x3 = _mm_aesenc_si128(*x3, key);
    # 使用AES加密算法对*x4进行加密，并使用key作为密钥
    *x4 = _mm_aesenc_si128(*x4, key);
    # 使用AES加密算法对*x5进行加密，并使用key作为密钥
    *x5 = _mm_aesenc_si128(*x5, key);
    # 使用AES加密算法对*x6进行加密，并使用key作为密钥
    *x6 = _mm_aesenc_si128(*x6, key);
    # 使用AES加密算法对*x7进行加密，并使用key作为密钥
    *x7 = _mm_aesenc_si128(*x7, key);
}

// 混合和传播函数，对输入的8个__m128i类型的参数进行混合和异或操作
inline void mix_and_propagate(__m128i& x0, __m128i& x1, __m128i& x2, __m128i& x3, __m128i& x4, __m128i& x5, __m128i& x6, __m128i& x7)
{
    // 临时变量保存x0
    __m128i tmp0 = x0;
    // 依次对x0到x7进行异或操作
    x0 = _mm_xor_si128(x0, x1);
    x1 = _mm_xor_si128(x1, x2);
    x2 = _mm_xor_si128(x2, x3);
    x3 = _mm_xor_si128(x3, x4);
    x4 = _mm_xor_si128(x4, x5);
    x5 = _mm_xor_si128(x5, x6);
    x6 = _mm_xor_si128(x6, x7);
    x7 = _mm_xor_si128(x7, tmp0);
}

// 命名空间xmrig
namespace xmrig {

// 根据模板参数interleave返回交错索引
template<int interleave>
static inline constexpr uint64_t interleaved_index(uint64_t k)
{
    return ((k & ~63ULL) << interleave) | (k & 63);
}

// 当interleave为0时，返回k
template<>
inline constexpr uint64_t interleaved_index<0>(uint64_t k)
{
    return k;
}

// 根据模板参数ALGO、SOFT_AES和interleave进行cn_explode_scratchpad函数定义
template<Algorithm::Id ALGO, bool SOFT_AES, int interleave>
static NOINLINE void cn_explode_scratchpad(cryptonight_ctx *ctx)
{
    // 根据ALGO获取算法属性
    constexpr CnAlgo<ALGO> props;

#   ifdef XMRIG_VAES
    // 如果不使用软件AES且算法不是Heavy且VAES可用，则调用cn_explode_scratchpad_vaes函数
    if (!SOFT_AES && !props.isHeavy() && cn_vaes_enabled) {
        cn_explode_scratchpad_vaes(ctx, props.memory(), props.half_mem());
        return;
    }
#   endif

    // 计算N的值
    constexpr size_t N = (props.memory() / sizeof(__m128i)) / (props.half_mem() ? 2 : 1);

    // 声明8个__m128i类型的变量
    __m128i xin0, xin1, xin2, xin3, xin4, xin5, xin6, xin7;
    __m128i k0, k1, k2, k3, k4, k5, k6, k7, k8, k9;

    // 将ctx->state强制转换为const __m128i*类型，并赋值给input
    const __m128i* input = reinterpret_cast<const __m128i*>(ctx->state);
    // 将ctx->memory强制转换为__m128i*类型，并赋值给output
    __m128i* output = reinterpret_cast<__m128i*>(ctx->memory);

    // 调用aes_genkey函数生成AES密钥
    aes_genkey<SOFT_AES>(input, &k0, &k1, &k2, &k3, &k4, &k5, &k6, &k7, &k8, &k9);

    // 如果props.half_mem()为真且ctx->first_half为假，则执行以下操作
    if (props.half_mem() && !ctx->first_half) {
        // 将ctx->save_state强制转换为const __m128i*类型，并赋值给p
        const __m128i* p = reinterpret_cast<const __m128i*>(ctx->save_state);
        // 依次加载p+0到p+7的值到xin0到xin7
        xin0 = _mm_load_si128(p + 0);
        xin1 = _mm_load_si128(p + 1);
        xin2 = _mm_load_si128(p + 2);
        xin3 = _mm_load_si128(p + 3);
        xin4 = _mm_load_si128(p + 4);
        xin5 = _mm_load_si128(p + 5);
        xin6 = _mm_load_si128(p + 6);
        xin7 = _mm_load_si128(p + 7);
    }
    else {
        // 从输入数组中加载第4个元素到xin0
        xin0 = _mm_load_si128(input + 4);
        // 从输入数组中加载第5个元素到xin1
        xin1 = _mm_load_si128(input + 5);
        // 从输入数组中加载第6个元素到xin2
        xin2 = _mm_load_si128(input + 6);
        // 从输入数组中加载第7个元素到xin3
        xin3 = _mm_load_si128(input + 7);
        // 从输入数组中加载第8个元素到xin4
        xin4 = _mm_load_si128(input + 8);
        // 从输入数组中加载第9个元素到xin5
        xin5 = _mm_load_si128(input + 9);
        // 从输入数组中加载第10个元素到xin6
        xin6 = _mm_load_si128(input + 10);
        // 从输入数组中加载第11个元素到xin7
        xin7 = _mm_load_si128(input + 11);
    }

    if (props.isHeavy()) {
        for (size_t i = 0; i < 16; i++) {
            // 对输入数据进行AES加密轮操作
            aes_round<SOFT_AES>(k0, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k1, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k2, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k3, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k4, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k5, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k6, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k7, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k8, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            // 重复进行AES加密轮操作
            aes_round<SOFT_AES>(k9, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);

            // 对输入数据进行混合和传播操作
            mix_and_propagate(xin0, xin1, xin2, xin3, xin4, xin5, xin6, xin7);
        }
    }

    // 计算输出增量
    constexpr int output_increment = (64 << interleave) / sizeof(__m128i);
    // 计算预取距离
    constexpr int prefetch_dist = 2048 / sizeof(__m128i);

    // 计算e的位置
    __m128i* e = output + (N << interleave) - prefetch_dist;
    // 计算预取指针的位置
    __m128i* prefetch_ptr = output + prefetch_dist;
    # 循环两次
    for (int i = 0; i < 2; ++i) {
        # 执行以下操作直到条件不满足
        do {
            # 预取内存中的数据到高速缓存中
            _mm_prefetch((const char*)(prefetch_ptr), _MM_HINT_T0);
            _mm_prefetch((const char*)(prefetch_ptr + output_increment), _MM_HINT_T0);

            # 执行 AES 加密的轮函数
            aes_round<SOFT_AES>(k0, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k1, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k2, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k3, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k4, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k5, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k6, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k7, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k8, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);
            aes_round<SOFT_AES>(k9, &xin0, &xin1, &xin2, &xin3, &xin4, &xin5, &xin6, &xin7);

            # 将结果存储到指定的内存位置
            _mm_store_si128(output + 0, xin0);
            _mm_store_si128(output + 1, xin1);
            _mm_store_si128(output + 2, xin2);
            _mm_store_si128(output + 3, xin3);

            _mm_store_si128(output + output_increment + 0, xin4);
            _mm_store_si128(output + output_increment + 1, xin5);
            _mm_store_si128(output + output_increment + 2, xin6);
            _mm_store_si128(output + output_increment + 3, xin7);

            # 更新指针位置
            output += output_increment * 2;
            prefetch_ptr += output_increment * 2;
        } while (output < e);  # 当条件满足时继续循环

        # 更新 e 的值
        e += prefetch_dist;
        prefetch_ptr = output;
    }
    # 如果内存一半已满并且上下文的第一半为真
    if (props.half_mem() && ctx->first_half) {
         # 将上下文保存状态转换为__m128i类型指针
         __m128i* p = reinterpret_cast<__m128i*>(ctx->save_state);
        # 将xin0的值存储到p+0指向的位置
        _mm_store_si128(p + 0, xin0);
        # 将xin1的值存储到p+1指向的位置
        _mm_store_si128(p + 1, xin1);
        # 将xin2的值存储到p+2指向的位置
        _mm_store_si128(p + 2, xin2);
        # 将xin3的值存储到p+3指向的位置
        _mm_store_si128(p + 3, xin3);
        # 将xin4的值存储到p+4指向的位置
        _mm_store_si128(p + 4, xin4);
        # 将xin5的值存储到p+5指向的位置
        _mm_store_si128(p + 5, xin5);
        # 将xin6的值存储到p+6指向的位置
        _mm_store_si128(p + 6, xin6);
        # 将xin7的值存储到p+7指向的位置
        _mm_store_si128(p + 7, xin7);
    }
// 定义一个静态模板函数，用于在内存中执行加密算法
template<Algorithm::Id ALGO, bool SOFT_AES, int interleave>
static NOINLINE void cn_implode_scratchpad(cryptonight_ctx *ctx)
{
    // 获取指定算法的属性
    constexpr CnAlgo<ALGO> props;

#   ifdef XMRIG_VAES
    // 如果不使用软件AES加密，并且算法不是重型算法，并且支持硬件AES加密
    if (!SOFT_AES && !props.isHeavy() && cn_vaes_enabled) {
        // 在内存中执行硬件AES加密
        cn_implode_scratchpad_vaes(ctx, props.memory(), props.half_mem());
        return;
    }
#   endif

    // 获取算法是否为重型算法的布尔值
    constexpr bool IS_HEAVY = props.isHeavy();
    // 计算内存大小
    constexpr size_t N = (props.memory() / sizeof(__m128i)) / (props.half_mem() ? 2 : 1);

    // 定义一系列变量
    __m128i xout0, xout1, xout2, xout3, xout4, xout5, xout6, xout7;
    __m128i k0, k1, k2, k3, k4, k5, k6, k7, k8, k9;

    // 将输入内存转换为__m128i类型指针
    const __m128i *input = reinterpret_cast<const __m128i*>(ctx->memory);
    // 将输出内存转换为__m128i类型指针
    __m128i *output = reinterpret_cast<__m128i*>(ctx->state);

    // 生成AES加密所需的密钥
    aes_genkey<SOFT_AES>(output + 2, &k0, &k1, &k2, &k3, &k4, &k5, &k6, &k7, &k8, &k9);

    // 加载输出内存中的数据到一系列变量中
    xout0 = _mm_load_si128(output + 4);
    xout1 = _mm_load_si128(output + 5);
    xout2 = _mm_load_si128(output + 6);
    xout3 = _mm_load_si128(output + 7);
    xout4 = _mm_load_si128(output + 8);
    xout5 = _mm_load_si128(output + 9);
    xout6 = _mm_load_si128(output + 10);
    xout7 = _mm_load_si128(output + 11);

    // 定义输入内存的起始位置
    const __m128i* input_begin = input;
    }

    }

    // 将一系列变量中的数据存储到输出内存中
    _mm_store_si128(output + 4, xout0);
    _mm_store_si128(output + 5, xout1);
    _mm_store_si128(output + 6, xout2);
    _mm_store_si128(output + 7, xout3);
    _mm_store_si128(output + 8, xout4);
    _mm_store_si128(output + 9, xout5);
    _mm_store_si128(output + 10, xout6);
    _mm_store_si128(output + 11, xout7);
}


} /* namespace xmrig */


// 定义一个内联函数，用于执行AES加密的轮函数
static inline __m128i aes_round_tweak_div(const __m128i &in, const __m128i &key)
{
    // 定义对齐的32位整数数组
    alignas(16) uint32_t k[4];
    alignas(16) uint32_t x[4];

    // 将密钥存储到k数组中
    _mm_store_si128((__m128i*) k, key);
    // 将输入数据异或后存储到x数组中
    _mm_store_si128((__m128i*) x, _mm_xor_si128(in, _mm_set_epi64x(0xffffffffffffffff, 0xffffffffffffffff)));

    // 定义一个宏，用于获取x数组中指定位置的字节
    #define BYTE(p, i) ((unsigned char*)&x[p])[i]
    # 对输入的 k 数组进行异或操作，使用 saes_table 中的值和 BYTE 函数计算
    k[0] ^= saes_table[0][BYTE(0, 0)] ^ saes_table[1][BYTE(1, 1)] ^ saes_table[2][BYTE(2, 2)] ^ saes_table[3][BYTE(3, 3)];
    # 对输入的 x 数组进行异或操作，使用上一步得到的 k[0] 值
    x[0] ^= k[0];
    # 对输入的 k 数组进行异或操作，使用 saes_table 中的值和 BYTE 函数计算
    k[1] ^= saes_table[0][BYTE(1, 0)] ^ saes_table[1][BYTE(2, 1)] ^ saes_table[2][BYTE(3, 2)] ^ saes_table[3][BYTE(0, 3)];
    # 对输入的 x 数组进行异或操作，使用上一步得到的 k[1] 值
    x[1] ^= k[1];
    # 对输入的 k 数组进行异或操作，使用 saes_table 中的值和 BYTE 函数计算
    k[2] ^= saes_table[0][BYTE(2, 0)] ^ saes_table[1][BYTE(3, 1)] ^ saes_table[2][BYTE(0, 2)] ^ saes_table[3][BYTE(1, 3)];
    # 对输入的 x 数组进行异或操作，使用上一步得到的 k[2] 值
    x[2] ^= k[2];
    # 对输入的 k 数组进行异或操作，使用 saes_table 中的值和 BYTE 函数计算
    k[3] ^= saes_table[0][BYTE(3, 0)] ^ saes_table[1][BYTE(0, 1)] ^ saes_table[2][BYTE(1, 2)] ^ saes_table[3][BYTE(2, 3)];
    # 取消定义 BYTE 宏
    #undef BYTE

    # 返回 k 数组的值，转换成 __m128i 类型
    return _mm_load_si128((__m128i*)k);
# 计算给定整数的平方根，返回结果为 128 位整数
static inline __m128i int_sqrt_v2(const uint64_t n0)
{
    # 将输入整数右移 12 位，转换为双精度浮点数，并加上一个固定值，得到初始值 x
    __m128d x = _mm_castsi128_pd(_mm_add_epi64(_mm_cvtsi64_si128(n0 >> 12), _mm_set_epi64x(0, 1023ULL << 52)));
    # 对 x 进行平方根运算
    x = _mm_sqrt_sd(_mm_setzero_pd(), x);
    # 将结果转换为 64 位整数
    uint64_t r = static_cast<uint64_t>(_mm_cvtsi128_si64(_mm_castpd_si128(x)));

    # 对 r 进行位移操作，得到新的 r
    const uint64_t s = r >> 20;
    r >>= 19;

    # 计算 x2 的值
    uint64_t x2 = (s - (1022ULL << 32)) * (r - s - (1022ULL << 32) + 1);
    # 根据条件判断，对 x2 和 r 进行更新
    # 如果满足条件，则调用 _addcarry_u64 和 _subborrow_u64 函数
    # 否则，对 x2 和 r 进行简单的加一操作
    # 注意：这里使用了条件编译，根据不同的编译器和架构选择不同的处理方式
    return _mm_cvtsi64_si128(r);
}

# 对给定的内存和参数进行加密处理
void v4_soft_aes_compile_code(const V4_Instruction *code, int code_size, void *machine_code, xmrig::Assembly ASM);

# 定义 xmrig 命名空间
namespace xmrig {

# 根据指定算法对内存进行加密处理
template<Algorithm::Id ALGO>
static inline void cryptonight_monero_tweak(uint64_t *mem_out, const uint8_t *l, uint64_t idx, __m128i ax0, __m128i bx0, __m128i bx1, __m128i& cx)
{
    # 获取指定算法的属性
    constexpr CnAlgo<ALGO> props;

    # 根据算法属性选择不同的处理方式
    if (props.base() == Algorithm::CN_2) {
        # 根据条件选择不同的处理方式
        VARIANT2_SHUFFLE(l, idx, ax0, bx0, bx1, cx, (((ALGO == Algorithm::CN_RWZ) || (ALGO == Algorithm::CN_UPX2)) ? 1 : 0));
        # 对 bx0 和 cx 进行异或操作，并存储结果到内存
        _mm_store_si128(reinterpret_cast<__m128i *>(mem_out), _mm_xor_si128(bx0, cx));
    } else {
        # 对 bx0 和 cx 进行异或操作，并将结果存储到内存
        __m128i tmp = _mm_xor_si128(bx0, cx);
        mem_out[0] = _mm_cvtsi128_si64(tmp);

        # 对 tmp 进行处理，得到另一个 64 位整数，并进行一定的处理
        tmp = _mm_castps_si128(_mm_movehl_ps(_mm_castsi128_ps(tmp), _mm_castsi128_ps(tmp)));
        uint64_t vh = _mm_cvtsi128_si64(tmp);

        # 将结果存储到内存
        mem_out[1] = vh ^ tweak1_table[static_cast<uint32_t>(vh) >> 24];
    }
}

# 对给定的参数进行加密处理
static inline void cryptonight_conceal_tweak(__m128i& cx, __m128& conc_var)
{
    # 对参数进行一系列复杂的数学运算
    __m128 r = _mm_add_ps(_mm_cvtepi32_ps(cx), conc_var);
    r = _mm_mul_ps(r, _mm_mul_ps(r, r));
    r = _mm_and_ps(_mm_castsi128_ps(_mm_set1_epi32(0x807FFFFF)), r);
    r = _mm_or_ps(_mm_castsi128_ps(_mm_set1_epi32(0x40000000)), r);
}
    # 保存当前的 conc_var 值
    __m128 c_old = conc_var;
    # 将 r 加到 conc_var 上
    conc_var = _mm_add_ps(conc_var, r);

    # 将 c_old 与 0x807FFFFF 进行按位与操作
    c_old = _mm_and_ps(_mm_castsi128_ps(_mm_set1_epi32(0x807FFFFF)), c_old);
    # 将 c_old 与 0x40000000 进行按位或操作
    c_old = _mm_or_ps(_mm_castsi128_ps(_mm_set1_epi32(0x40000000)), c_old);

    # 将 c_old 与 536870880.0f 进行按位乘法操作
    __m128 nc = _mm_mul_ps(c_old, _mm_set1_ps(536870880.0f));
    # 将 cx 与 nc 进行按位异或操作
    cx = _mm_xor_si128(cx, _mm_cvttps_epi32(nc));
}
#ifdef XMRIG_FEATURE_ASM
// 定义了一个静态函数，用于在支持 SSE4.1 指令集的环境下进行 Cryptonight 单次哈希计算
template<Algorithm::Id ALGO>
static void cryptonight_single_hash_gr_sse41(const uint8_t* __restrict__ input, size_t size, uint8_t* __restrict__ output, cryptonight_ctx** __restrict__ ctx, uint64_t height);
#endif

// 定义了一个模板函数，用于进行 Cryptonight 单次哈希计算
template<Algorithm::Id ALGO, bool SOFT_AES, int interleave>
inline void cryptonight_single_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
#   ifdef XMRIG_FEATURE_ASM
    // 如果不使用软件 AES 加密
    if (!SOFT_AES) {
        // 根据不同的算法类型选择不同的处理方式
        switch (ALGO) {
        case Algorithm::CN_GR_0:
        case Algorithm::CN_GR_1:
        case Algorithm::CN_GR_2:
        case Algorithm::CN_GR_3:
        case Algorithm::CN_GR_4:
        case Algorithm::CN_GR_5:
            // 如果支持 SSE4.1 指令集，则调用对应的函数进行哈希计算
            if (cn_sse41_enabled) {
                cryptonight_single_hash_gr_sse41<ALGO>(input, size, output, ctx, height);
                return;
            }
            break;

        default:
            break;
        }
    }
#   endif

    // 根据算法类型获取相关属性
    constexpr CnAlgo<ALGO> props;
    constexpr size_t MASK        = props.mask();
    constexpr Algorithm::Id BASE = props.base();

#   ifdef XMRIG_ALGO_CN_HEAVY
    // 如果是 CN_HEAVY_TUBE 算法
    constexpr bool IS_CN_HEAVY_TUBE = ALGO == Algorithm::CN_HEAVY_TUBE;
#   else
    constexpr bool IS_CN_HEAVY_TUBE = false;
#   endif

    // 如果是 CN_1 算法且输入数据长度小于 43，则直接返回全 0 的输出
    if (BASE == Algorithm::CN_1 && size < 43) {
        memset(output, 0, 32);
        return;
    }

    // 对输入数据进行 Keccak 哈希计算
    keccak(input, size, ctx[0]->state);

    // 如果算法需要使用一半内存，则设置相应标志位
    if (props.half_mem()) {
        ctx[0]->first_half = true;
    }
    // 根据算法类型和是否使用软件 AES 加密进行内存分配和初始化
    cn_explode_scratchpad<ALGO, SOFT_AES, interleave>(ctx[0]);

    // 获取哈希计算结果的高位和低位指针
    uint64_t *h0 = reinterpret_cast<uint64_t*>(ctx[0]->state);
    uint8_t *l0   = ctx[0]->memory;

#   ifdef XMRIG_FEATURE_ASM
    # 如果 SOFT_AES 为真并且 props.isR() 为真，则执行以下代码块
    if (SOFT_AES && props.isR()) {
        # 如果生成的代码数据不匹配 ALGO 和 height，则执行以下代码块
        if (!ctx[0]->generated_code_data.match(ALGO, height)) {
            # 创建一个包含256个V4_Instruction的数组code
            V4_Instruction code[256];
            # 调用v4_random_math_init函数初始化code数组，并返回code的大小
            const int code_size = v4_random_math_init<ALGO>(code, height);

            # 如果ALGO为Algorithm::CN_R，则执行以下代码块
            if (ALGO == Algorithm::CN_R) {
                # 调用v4_soft_aes_compile_code函数编译code数组，并将结果存储在ctx[0]->generated_code中
                v4_soft_aes_compile_code(code, code_size, reinterpret_cast<void*>(ctx[0]->generated_code), Assembly::NONE);
            }

            # 将生成的代码数据存储在ctx[0]->generated_code_data中，包括ALGO和height
            ctx[0]->generated_code_data = { ALGO, height };
        }

        # 将saes_table的地址转换为const uint32_t*类型，并存储在ctx[0]->saes_table中
        ctx[0]->saes_table = reinterpret_cast<const uint32_t*>(saes_table);
        # 调用ctx[0]->generated_code函数，传入ctx作为参数
        ctx[0]->generated_code(ctx);
    } else {
#   endif

    // 初始化 VARIANT1
    VARIANT1_INIT(0);
    // 初始化 VARIANT2
    VARIANT2_INIT(0);
    // 设置 VARIANT2 的舍入模式
    VARIANT2_SET_ROUNDING_MODE();
    // 初始化 VARIANT4 的随机数数学运算
    VARIANT4_RANDOM_MATH_INIT(0);

    // 计算 al0
    uint64_t al0  = h0[0] ^ h0[4];
    // 计算 ah0
    uint64_t ah0  = h0[1] ^ h0[5];
    // 计算 idx0
    uint64_t idx0 = al0;
    // 初始化 bx0
    __m128i bx0   = _mm_set_epi64x(static_cast<int64_t>(h0[3] ^ h0[7]), static_cast<int64_t>(h0[2] ^ h0[6]));
    // 初始化 bx1
    __m128i bx1   = _mm_set_epi64x(static_cast<int64_t>(h0[9] ^ h0[11]), static_cast<int64_t>(h0[8] ^ h0[10]));

    // 初始化 conc_var
    __m128 conc_var;
    // 如果 ALGO 是 CN_CCX，则将 conc_var 初始化为 0，并恢复舍入模式
    if (ALGO == Algorithm::CN_CCX) {
        conc_var = _mm_setzero_ps();
        RESTORE_ROUNDING_MODE();
    }

#       ifdef XMRIG_ALGO_CN_HEAVY
        // 如果 props 是 Heavy，则执行以下操作
        if (props.isHeavy()) {
            // 从 l0 中获取 n 和 d
            int64_t n = ((int64_t*)&l0[interleaved_index<interleave>(idx0 & MASK)])[0];
            int64_t d = ((int32_t*)&l0[interleaved_index<interleave>(idx0 & MASK)])[2];

            int64_t d5;

#           if defined(_MSC_VER) || (defined(__GNUC__) && (__GNUC__ == 8)) || !defined(XMRIG_64_BIT)
            // 如果满足条件，则执行 d5 = d | 5
            d5 = d | 5;
#           else
            // 否则执行以下汇编指令
            // 为了解决 GCC 的问题，将 d 转换为 32 位，执行 "| 5" 操作，然后再转换回 64 位
            asm("mov %1, %0\n\tor $5, %0" : "=r"(d5) : "r"(d));
#           endif

            // 计算 q
            int64_t q = n / d5;

            // 修改 l0 中的值
            ((int64_t*)&l0[interleaved_index<interleave>(idx0 & MASK)])[0] = n ^ q;

            // 如果 ALGO 是 CN_HEAVY_XHV，则执行以下操作
            if (ALGO == Algorithm::CN_HEAVY_XHV) {
                d = ~d;
            }

            // 计算 idx0
            idx0 = d ^ q;
        }
#       endif

        // 如果 BASE 是 CN_2，则将 bx1 设置为 bx0
        if (BASE == Algorithm::CN_2) {
            bx1 = bx0;
        }

        // 将 bx0 设置为 cx
        bx0 = cx;
    }

#   ifdef XMRIG_FEATURE_ASM
    }
#   endif

    // 执行 cn_implode_scratchpad 函数
    cn_implode_scratchpad<ALGO, SOFT_AES, interleave>(ctx[0]);
    // 执行 keccakf 函数
    keccakf(h0, 24);
    // 执行 extra_hashes 函数
    extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
}


} /* namespace xmrig */


#ifdef XMRIG_FEATURE_ASM
// 声明使用汇编实现的函数
extern "C" void cnv1_single_mainloop_asm(cryptonight_ctx * *ctx);
extern "C" void cnv1_double_mainloop_asm(cryptonight_ctx **ctx);
extern "C" void cnv1_quad_mainloop_asm(cryptonight_ctx **ctx);
// 声明外部 C 函数，用于调用 IvyBridge 架构的主循环
extern "C" void cnv2_mainloop_ivybridge_asm(cryptonight_ctx **ctx);
// 声明外部 C 函数，用于调用 Ryzen 架构的主循环
extern "C" void cnv2_mainloop_ryzen_asm(cryptonight_ctx **ctx);
// 声明外部 C 函数，用于调用 Bulldozer 架构的主循环
extern "C" void cnv2_mainloop_bulldozer_asm(cryptonight_ctx **ctx);
// 声明外部 C 函数，用于调用 SandyBridge 架构的双主循环
extern "C" void cnv2_double_mainloop_sandybridge_asm(cryptonight_ctx **ctx);
// 声明外部 C 函数，用于调用 RWZ 主循环
extern "C" void cnv2_rwz_mainloop_asm(cryptonight_ctx **ctx);
// 声明外部 C 函数，用于调用 RWZ 双主循环
extern "C" void cnv2_rwz_double_mainloop_asm(cryptonight_ctx **ctx);
// 声明外部 C 函数，用于调用 Zen3 架构的 UPX 双主循环
extern "C" void cnv2_upx_double_mainloop_zen3_asm(cryptonight_ctx **ctx);

// 命名空间 xmrig
namespace xmrig {

// 定义函数指针类型，用于指向主循环函数
typedef void (*cn_mainloop_fun)(cryptonight_ctx **ctx);

// 声明外部 C 函数指针，指向 IvyBridge 架构的半主循环
extern cn_mainloop_fun cn_half_mainloop_ivybridge_asm;
// 声明外部 C 函数指针，指向 Ryzen 架构的半主循环
extern cn_mainloop_fun cn_half_mainloop_ryzen_asm;
// 声明外部 C 函数指针，指向 Bulldozer 架构的半主循环
extern cn_mainloop_fun cn_half_mainloop_bulldozer_asm;
// 声明外部 C 函数指针，指向 SandyBridge 架构的双半主循环
extern cn_mainloop_fun cn_half_double_mainloop_sandybridge_asm;

// 声明外部 C 函数指针，指向 IvyBridge 架构的 TRTL 主循环
extern cn_mainloop_fun cn_trtl_mainloop_ivybridge_asm;
// 声明外部 C 函数指针，指向 Ryzen 架构的 TRTL 主循环
extern cn_mainloop_fun cn_trtl_mainloop_ryzen_asm;
// 声明外部 C 函数指针，指向 Bulldozer 架构的 TRTL 主循环
extern cn_mainloop_fun cn_trtl_mainloop_bulldozer_asm;
// 声明外部 C 函数指针，指向 SandyBridge 架构的双 TRTL 主循环
extern cn_mainloop_fun cn_trtl_double_mainloop_sandybridge_asm;

// 声明外部 C 函数指针，指向 IvyBridge 架构的 TLO 主循环
extern cn_mainloop_fun cn_tlo_mainloop_ivybridge_asm;
// 声明外部 C 函数指针，指向 Ryzen 架构的 TLO 主循环
extern cn_mainloop_fun cn_tlo_mainloop_ryzen_asm;
// 声明外部 C 函数指针，指向 Bulldozer 架构的 TLO 主循环
extern cn_mainloop_fun cn_tlo_mainloop_bulldozer_asm;
// 声明外部 C 函数指针，指向 SandyBridge 架构的双 TLO 主循环
extern cn_mainloop_fun cn_tlo_double_mainloop_sandybridge_asm;

// 声明外部 C 函数指针，指向 IvyBridge 架构的 ZLS 主循环
extern cn_mainloop_fun cn_zls_mainloop_ivybridge_asm;
// 声明外部 C 函数指针，指向 Ryzen 架构的 ZLS 主循环
extern cn_mainloop_fun cn_zls_mainloop_ryzen_asm;
// 声明外部 C 函数指针，指向 Bulldozer 架构的 ZLS 主循环
extern cn_mainloop_fun cn_zls_mainloop_bulldozer_asm;
// 声明外部 C 函数指针，指向 SandyBridge 架构的双 ZLS 主循环
extern cn_mainloop_fun cn_zls_double_mainloop_sandybridge_asm;

// 声明外部 C 函数指针，指向 IvyBridge 架构的双主循环
extern cn_mainloop_fun cn_double_mainloop_ivybridge_asm;
// 声明外部 C 函数指针，指向 Ryzen 架构的双主循环
extern cn_mainloop_fun cn_double_mainloop_ryzen_asm;
// 声明外部 C 函数指针，指向 Bulldozer 架构的双主循环
extern cn_mainloop_fun cn_double_mainloop_bulldozer_asm;
// 声明外部 C 函数指针，指向 SandyBridge 架构的双双主循环
extern cn_mainloop_fun cn_double_double_mainloop_sandybridge_asm;

// 声明外部 C 函数指针，指向 UPX2 主循环
extern cn_mainloop_fun cn_upx2_mainloop_asm;
// 声明外部 C 函数指针，指向 UPX2 双主循环
extern cn_mainloop_fun cn_upx2_double_mainloop_asm;

// 声明外部 C 函数指针，指向 GR0 单主循环
extern cn_mainloop_fun cn_gr0_single_mainloop_asm;
// 声明外部 C 函数指针，指向 GR1 单主循环
extern cn_mainloop_fun cn_gr1_single_mainloop_asm;
// 声明外部 C 函数指针，指向 GR2 单主循环
extern cn_mainloop_fun cn_gr2_single_mainloop_asm;
// 声明外部 C 函数指针，指向 GR3 单主循环
extern cn_mainloop_fun cn_gr3_single_mainloop_asm;
// 声明外部函数 cn_gr4_single_mainloop_asm 和 cn_gr5_single_mainloop_asm
extern cn_mainloop_fun cn_gr4_single_mainloop_asm;
extern cn_mainloop_fun cn_gr5_single_mainloop_asm;

// 声明外部函数 cn_gr0_double_mainloop_asm 到 cn_gr5_double_mainloop_asm
extern cn_mainloop_fun cn_gr0_double_mainloop_asm;
extern cn_mainloop_fun cn_gr1_double_mainloop_asm;
extern cn_mainloop_fun cn_gr2_double_mainloop_asm;
extern cn_mainloop_fun cn_gr3_double_mainloop_asm;
extern cn_mainloop_fun cn_gr4_double_mainloop_asm;
extern cn_mainloop_fun cn_gr5_double_mainloop_asm;

// 声明外部函数 cn_gr0_quad_mainloop_asm 到 cn_gr5_quad_mainloop_asm
extern cn_mainloop_fun cn_gr0_quad_mainloop_asm;
extern cn_mainloop_fun cn_gr1_quad_mainloop_asm;
extern cn_mainloop_fun cn_gr2_quad_mainloop_asm;
extern cn_mainloop_fun cn_gr3_quad_mainloop_asm;
extern cn_mainloop_fun cn_gr4_quad_mainloop_asm;
extern cn_mainloop_fun cn_gr5_quad_mainloop_asm;

// 结束 xmrig 命名空间
} // namespace xmrig

// 定义函数 v4_compile_code，用于编译 V4 指令
void v4_compile_code(const V4_Instruction* code, int code_size, void* machine_code, xmrig::Assembly ASM);
// 定义函数 v4_compile_code_double，用于编译双倍 V4 指令
void v4_compile_code_double(const V4_Instruction* code, int code_size, void* machine_code, xmrig::Assembly ASM);

// 定义模板函数 cn_r_compile_code，用于编译指定算法的 V4 指令
template<xmrig::Algorithm::Id ALGO>
void cn_r_compile_code(const V4_Instruction* code, int code_size, void* machine_code, xmrig::Assembly ASM)
{
    v4_compile_code(code, code_size, machine_code, ASM);
}

// 定义模板函数 cn_r_compile_code_double，用于编译指定算法的双倍 V4 指令
template<xmrig::Algorithm::Id ALGO>
void cn_r_compile_code_double(const V4_Instruction* code, int code_size, void* machine_code, xmrig::Assembly ASM)
{
    v4_compile_code_double(code, code_size, machine_code, ASM);
}

// 开始 xmrig 命名空间
namespace xmrig {

// 定义模板函数 cryptonight_single_hash_asm，用于执行单哈希计算
template<Algorithm::Id ALGO, Assembly::Id ASM>
inline void cryptonight_single_hash_asm(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
    // 获取指定算法的属性
    constexpr CnAlgo<ALGO> props;

    // 如果是 R 系列算法，并且上下文中未生成指定算法和高度的代码数据
    if (props.isR() && !ctx[0]->generated_code_data.match(ALGO, height)) {
        // 创建 V4 指令数组
        V4_Instruction code[256];
        // 获取 V4 指令的大小
        const int code_size = v4_random_math_init<ALGO>(code, height);
        // 编译 V4 指令
        cn_r_compile_code<ALGO>(code, code_size, reinterpret_cast<void*>(ctx[0]->generated_code), ASM);

        // 更新上下文中的生成代码数据
        ctx[0]->generated_code_data = { ALGO, height };
    }
}
    # 使用 Keccak 算法对输入进行哈希计算，将结果存储到 ctx[0]->state 中
    keccak(input, size, ctx[0]->state);

    # 如果内存使用量为一半，则将 ctx[0]->first_half 设置为 true
    if (props.half_mem()) {
        ctx[0]->first_half = true;
    }

    # 根据算法类型和硬件架构调用不同的函数来处理内存数据
    cn_explode_scratchpad<ALGO, false, 0>(ctx[0]);

    # 如果算法为 CN_2
    if (ALGO == Algorithm::CN_2) {
        # 根据硬件架构调用不同的函数来执行 CN_2 算法的主循环
        if (ASM == Assembly::INTEL) {
            cnv2_mainloop_ivybridge_asm(ctx);
        }
        else if (ASM == Assembly::RYZEN) {
            cnv2_mainloop_ryzen_asm(ctx);
        }
        else {
            cnv2_mainloop_bulldozer_asm(ctx);
        }
    }
    # 如果算法为 CN_HALF
    else if (ALGO == Algorithm::CN_HALF) {
        # 根据硬件架构调用不同的函数来执行 CN_HALF 算法的主循环
        if (ASM == Assembly::INTEL) {
            cn_half_mainloop_ivybridge_asm(ctx);
        }
        else if (ASM == Assembly::RYZEN) {
            cn_half_mainloop_ryzen_asm(ctx);
        }
        else {
            cn_half_mainloop_bulldozer_asm(ctx);
        }
    }
# 如果算法是 CN_PICO
else if (ALGO == Algorithm::CN_PICO_0) {
    # 如果使用的是英特尔架构
    if (ASM == Assembly::INTEL) {
        # 调用英特尔架构的 cn_trtl_mainloop_ivybridge_asm 函数
        cn_trtl_mainloop_ivybridge_asm(ctx);
    }
    # 如果使用的是 AMD RYZEN 架构
    else if (ASM == Assembly::RYZEN) {
        # 调用 AMD RYZEN 架构的 cn_trtl_mainloop_ryzen_asm 函数
        cn_trtl_mainloop_ryzen_asm(ctx);
    }
    # 如果使用的是 AMD BULLDOZER 架构
    else {
        # 调用 AMD BULLDOZER 架构的 cn_trtl_mainloop_bulldozer_asm 函数
        cn_trtl_mainloop_bulldozer_asm(ctx);
    }
}
# 如果算法是 CN_PICO_TLO
else if (ALGO == Algorithm::CN_PICO_TLO) {
    # 如果使用的是英特尔架构
    if (ASM == Assembly::INTEL) {
        # 调用英特尔架构的 cn_tlo_mainloop_ivybridge_asm 函数
        cn_tlo_mainloop_ivybridge_asm(ctx);
    }
    # 如果使用的是 AMD RYZEN 架构
    else if (ASM == Assembly::RYZEN) {
        # 调用 AMD RYZEN 架构的 cn_tlo_mainloop_ryzen_asm 函数
        cn_tlo_mainloop_ryzen_asm(ctx);
    }
    # 如果使用的是 AMD BULLDOZER 架构
    else {
        # 调用 AMD BULLDOZER 架构的 cn_tlo_mainloop_bulldozer_asm 函数
        cn_tlo_mainloop_bulldozer_asm(ctx);
    }
}
# 如果算法是 CN_RWZ
else if (ALGO == Algorithm::CN_RWZ) {
    # 调用 cnv2_rwz_mainloop_asm 函数
    cnv2_rwz_mainloop_asm(ctx);
}
# 如果算法是 CN_ZLS
else if (ALGO == Algorithm::CN_ZLS) {
    # 如果使用的是英特尔架构
    if (ASM == Assembly::INTEL) {
        # 调用英特尔架构的 cn_zls_mainloop_ivybridge_asm 函数
        cn_zls_mainloop_ivybridge_asm(ctx);
    }
    # 如果使用的是 AMD RYZEN 架构
    else if (ASM == Assembly::RYZEN) {
        # 调用 AMD RYZEN 架构的 cn_zls_mainloop_ryzen_asm 函数
        cn_zls_mainloop_ryzen_asm(ctx);
    }
    # 如果使用的是 AMD BULLDOZER 架构
    else {
        # 调用 AMD BULLDOZER 架构的 cn_zls_mainloop_bulldozer_asm 函数
        cn_zls_mainloop_bulldozer_asm(ctx);
    }
}
# 如果算法是 CN_DOUBLE
else if (ALGO == Algorithm::CN_DOUBLE) {
    # 如果使用的是英特尔架构
    if (ASM == Assembly::INTEL) {
        # 调用英特尔架构的 cn_double_mainloop_ivybridge_asm 函数
        cn_double_mainloop_ivybridge_asm(ctx);
    }
    # 如果使用的是 AMD RYZEN 架构
    else if (ASM == Assembly::RYZEN) {
        # 调用 AMD RYZEN 架构的 cn_double_mainloop_ryzen_asm 函数
        cn_double_mainloop_ryzen_asm(ctx);
    }
    # 如果使用的是 AMD BULLDOZER 架构
    else {
        # 调用 AMD BULLDOZER 架构的 cn_double_mainloop_bulldozer_asm 函数
        cn_double_mainloop_bulldozer_asm(ctx);
    }
}
# 如果定义了 XMRIG_ALGO_CN_FEMTO
else if (ALGO == Algorithm::CN_UPX2) {
    # 调用 cn_upx2_mainloop_asm 函数
    cn_upx2_mainloop_asm(ctx);
}
# 如果 props.isR() 为真
else if (props.isR()) {
    # 调用 ctx[0]->generated_code 函数
    ctx[0]->generated_code(ctx);
}

# 调用 cn_implode_scratchpad 函数
cn_implode_scratchpad<ALGO, false, 0>(ctx[0]);
# 调用 keccakf 函数
keccakf(reinterpret_cast<uint64_t*>(ctx[0]->state), 24);
# 调用 extra_hashes 函数
extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
}

# 定义模板函数 cryptonight_double_hash_asm
template<Algorithm::Id ALGO, Assembly::Id ASM>
inline void cryptonight_double_hash_asm(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
# 使用常量表达式获取算法属性
constexpr CnAlgo<ALGO> props;

# 如果算法属性是随机内存，且上下文中没有生成过相同算法和高度的代码数据
if (props.isR() && !ctx[0]->generated_code_data.match(ALGO, height)) {
    # 创建一个长度为256的 V4_Instruction 数组
    V4_Instruction code[256];
    # 使用随机数生成器初始化指定算法的代码，并返回代码的大小
    const int code_size = v4_random_math_init<ALGO>(code, height);
    # 将生成的代码编译成双线程的汇编代码
    cn_r_compile_code_double<ALGO>(code, code_size, reinterpret_cast<void*>(ctx[0]->generated_code), ASM);

    # 更新上下文中的生成代码数据
    ctx[0]->generated_code_data = { ALGO, height };
}

# 对输入数据进行 Keccak 哈希计算，将结果存储到上下文的状态中
keccak(input,        size, ctx[0]->state);
keccak(input + size, size, ctx[1]->state);

# 如果算法属性要求使用一半内存
if (props.half_mem()) {
    # 设置上下文中的第一个线程和第二个线程使用一半内存
    ctx[0]->first_half = true;
    ctx[1]->first_half = true;
}

# 如果支持 VAES 指令集，并且算法属性不是重型算法，并且 VAES 已启用
# 则使用 VAES 指令集执行双线程的内存爆破
# 否则执行普通的内存爆破
#ifdef XMRIG_VAES
if (!props.isHeavy() && cn_vaes_enabled) {
    cn_explode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
}
else
#endif
{
    cn_explode_scratchpad<ALGO, false, 0>(ctx[0]);
    cn_explode_scratchpad<ALGO, false, 0>(ctx[1]);
}

# 根据不同的算法类型执行相应的主循环汇编代码
if (ALGO == Algorithm::CN_2) {
    cnv2_double_mainloop_sandybridge_asm(ctx);
}
else if (ALGO == Algorithm::CN_HALF) {
    cn_half_double_mainloop_sandybridge_asm(ctx);
}
#ifdef XMRIG_ALGO_CN_PICO
else if (ALGO == Algorithm::CN_PICO_0) {
    cn_trtl_double_mainloop_sandybridge_asm(ctx);
}
else if (ALGO == Algorithm::CN_PICO_TLO) {
    cn_tlo_double_mainloop_sandybridge_asm(ctx);
}
#endif
#ifdef XMRIG_ALGO_CN_FEMTO
else if (ALGO == Algorithm::CN_UPX2) {
    if (Cpu::info()->arch() == ICpuInfo::ARCH_ZEN3) {
        cnv2_upx_double_mainloop_zen3_asm(ctx);
    }
    else {
        cn_upx2_double_mainloop_asm(ctx);
    }
}
#endif
else if (ALGO == Algorithm::CN_RWZ) {
    cnv2_rwz_double_mainloop_asm(ctx);
}
else if (ALGO == Algorithm::CN_ZLS) {
    cn_zls_double_mainloop_sandybridge_asm(ctx);
}
else if (ALGO == Algorithm::CN_DOUBLE) {
    cn_double_double_mainloop_sandybridge_asm(ctx);
}
else if (props.isR()) {
    ctx[0]->generated_code(ctx);
}
// 如果定义了 XMRIG_VAES
    if (!props.isHeavy() && cn_vaes_enabled) {
        // 如果不是重型算法且启用了 VAES，使用 VAES 加密算法处理内存
        cn_implode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
    }
    else
// 如果未定义 XMRIG_VAES
    {
        // 否则使用普通的加密算法处理内存
        cn_implode_scratchpad<ALGO, false, 0>(ctx[0]);
        cn_implode_scratchpad<ALGO, false, 0>(ctx[1]);
    }

    // 对 ctx[0] 的状态进行 Keccak 哈希计算
    keccakf(reinterpret_cast<uint64_t*>(ctx[0]->state), 24);
    // 对 ctx[1] 的状态进行 Keccak 哈希计算
    keccakf(reinterpret_cast<uint64_t*>(ctx[1]->state), 24);

    // 使用 ctx[0] 的状态进行额外哈希计算，结果存储在 output 中
    extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
    // 使用 ctx[1] 的状态进行额外哈希计算，结果存储在 output + 32 中
    extra_hashes[ctx[1]->state[0] & 3](ctx[1]->state, 200, output + 32);
}


} /* namespace xmrig */
#endif


namespace xmrig {


#ifdef XMRIG_FEATURE_ASM
// 根据指定的算法 ID 使用 SSE41 指令集进行单次 Cryptonight 哈希计算
template<Algorithm::Id ALGO>
static NOINLINE void cryptonight_single_hash_gr_sse41(const uint8_t* __restrict__ input, size_t size, uint8_t* __restrict__ output, cryptonight_ctx** __restrict__ ctx, uint64_t height)
{
    // 获取指定算法的属性
    constexpr CnAlgo<ALGO> props;
    // 获取指定算法的基础算法 ID
    constexpr Algorithm::Id BASE = props.base();

    // 如果基础算法是 CN_1 且输入大小小于 43，则直接返回全零的输出
    if (BASE == Algorithm::CN_1 && size < 43) {
        memset(output, 0, 32);
        return;
    }

    // 对输入数据进行 Keccak 哈希计算，结果存储在 ctx[0]->state 中
    keccak(input, size, ctx[0]->state);

    // 如果内存使用量为一半，则设置 ctx[0]->first_half 为 true
    if (props.half_mem()) {
        ctx[0]->first_half = true;
    }
    // 展开内存数据到 scratchpad
    cn_explode_scratchpad<ALGO, false, 0>(ctx[0]);

    // 初始化 VARIANT1
    VARIANT1_INIT(0);
    ctx[0]->tweak1_2 = tweak1_2_0;
    ctx[0]->tweak1_table = tweak1_table;
    // 根据不同的算法 ID 调用对应的汇编主循环函数
    if (ALGO == Algorithm::CN_GR_0) cn_gr0_single_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_1) cn_gr1_single_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_2) cn_gr2_single_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_3) cn_gr3_single_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_4) cn_gr4_single_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_5) cn_gr5_single_mainloop_asm(ctx);

    // 将 scratchpad 数据压缩到 ctx[0] 中
    cn_implode_scratchpad<ALGO, false, 0>(ctx[0]);
    // 对 ctx[0] 的状态进行 Keccak 哈希计算
    keccakf(reinterpret_cast<uint64_t*>(ctx[0]->state), 24);
    // 使用 ctx[0] 的状态进行额外哈希计算，结果存储在 output 中
    extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
}


template<Algorithm::Id ALGO>
static NOINLINE void cryptonight_double_hash_gr_sse41(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
    // 定义一个常量属性对象，用于获取算法基础信息
    constexpr CnAlgo<ALGO> props;
    // 获取算法的基础 ID
    constexpr Algorithm::Id BASE = props.base();

    // 如果基础算法是 CN_1 并且输入数据大小小于 43，则将输出数据置零并返回
    if (BASE == Algorithm::CN_1 && size < 43) {
        memset(output, 0, 64);
        return;
    }

    // 对输入数据进行 Keccak 哈希，分别存储到两个上下文的状态中
    keccak(input,        size, ctx[0]->state);
    keccak(input + size, size, ctx[1]->state);

    // 如果算法要求使用一半内存，则设置上下文的 first_half 标志为 true
    if (props.half_mem()) {
        ctx[0]->first_half = true;
        ctx[1]->first_half = true;
    }

#   ifdef XMRIG_VAES
    // 如果算法不是 Heavy 并且 VAES 可用，则使用 VAES 加速计算
    if (!props.isHeavy() && cn_vaes_enabled) {
        cn_explode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
    }
    else
#   endif
    {
        // 否则使用普通方式展开内存
        cn_explode_scratchpad<ALGO, false, 0>(ctx[0]);
        cn_explode_scratchpad<ALGO, false, 0>(ctx[1]);
    }

    // 初始化 VARIANT1
    VARIANT1_INIT(0);
    VARIANT1_INIT(1);

    // 设置上下文的 tweak1_2 和 tweak1_table
    ctx[0]->tweak1_2 = tweak1_2_0;
    ctx[1]->tweak1_2 = tweak1_2_1;
    ctx[0]->tweak1_table = tweak1_table;

    // 根据不同的算法类型调用对应的主循环汇编函数
    if (ALGO == Algorithm::CN_GR_0) cn_gr0_double_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_1) cn_gr1_double_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_2) cn_gr2_double_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_3) cn_gr3_double_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_4) cn_gr4_double_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_5) cn_gr5_double_mainloop_asm(ctx);

#   ifdef XMRIG_VAES
    // 如果算法不是 Heavy 并且 VAES 可用，则使用 VAES 加速计算
    if (!props.isHeavy() && cn_vaes_enabled) {
        cn_implode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
    }
    else
#   endif
    {
        // 否则使用普通方式压缩内存
        cn_implode_scratchpad<ALGO, false, 0>(ctx[0]);
        cn_implode_scratchpad<ALGO, false, 0>(ctx[1]);
    }

    // 对上下文的状态进行 Keccak 压缩
    keccakf(reinterpret_cast<uint64_t*>(ctx[0]->state), 24);
    keccakf(reinterpret_cast<uint64_t*>(ctx[1]->state), 24);

    // 根据上下文的状态的第一个字节的值选择额外的哈希函数进行计算
    extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
}
    # 使用哈希函数计算ctx[1]->state[0] & 3的结果，并将结果作为索引从extra_hashes中选择哈希函数
    extra_hashes[ctx[1]->state[0] & 3](ctx[1]->state, 200, output + 32);
}
#endif


template<Algorithm::Id ALGO, bool SOFT_AES>
inline void cryptonight_double_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
#   ifdef XMRIG_FEATURE_ASM
    // 如果不是软件AES加密
    if (!SOFT_AES) {
        // 根据算法类型选择相应的操作
        switch (ALGO) {
        case Algorithm::CN_GR_0:
        case Algorithm::CN_GR_1:
        case Algorithm::CN_GR_2:
        case Algorithm::CN_GR_3:
        case Algorithm::CN_GR_4:
        case Algorithm::CN_GR_5:
            // 如果支持SSE4.1指令集，则调用对应的SSE4.1加密函数
            if (cn_sse41_enabled) {
                cryptonight_double_hash_gr_sse41<ALGO>(input, size, output, ctx, height);
                return;
            }
            break;

        default:
            break;
        }
    }
#   endif

    // 根据算法类型获取属性
    constexpr CnAlgo<ALGO> props;
    // 获取掩码
    constexpr size_t MASK        = props.mask();
    // 获取基础算法类型
    constexpr Algorithm::Id BASE = props.base();

#   ifdef XMRIG_ALGO_CN_HEAVY
    // 判断是否为CN_HEAVY_TUBE算法
    constexpr bool IS_CN_HEAVY_TUBE = ALGO == Algorithm::CN_HEAVY_TUBE;
#   else
    constexpr bool IS_CN_HEAVY_TUBE = false;
#   endif

    // 如果基础算法类型为CN_1且输入数据大小小于43，则直接将输出数据置零并返回
    if (BASE == Algorithm::CN_1 && size < 43) {
        memset(output, 0, 64);
        return;
    }

    // 对输入数据进行Keccak哈希计算
    keccak(input,        size, ctx[0]->state);
    keccak(input + size, size, ctx[1]->state);

    // 获取内存指针和状态指针
    uint8_t *l0  = ctx[0]->memory;
    uint8_t *l1  = ctx[1]->memory;
    uint64_t *h0 = reinterpret_cast<uint64_t*>(ctx[0]->state);
    uint64_t *h1 = reinterpret_cast<uint64_t*>(ctx[1]->state);

    // 初始化变体1
    VARIANT1_INIT(0);
    VARIANT1_INIT(1);
    // 初始化变体2
    VARIANT2_INIT(0);
    VARIANT2_INIT(1);
    // 设置舍入模式
    VARIANT2_SET_ROUNDING_MODE();
    // 初始化随机数运算
    VARIANT4_RANDOM_MATH_INIT(0);
    VARIANT4_RANDOM_MATH_INIT(1);

    // 如果内存使用量为一半，则设置上下文的first_half为true
    if (props.half_mem()) {
        ctx[0]->first_half = true;
        ctx[1]->first_half = true;
    }

#   ifdef XMRIG_VAES
    // 如果不是软件AES加密且不是重型算法且支持VAES指令集，则调用VAES加密函数
    if (!SOFT_AES && !props.isHeavy() && cn_vaes_enabled) {
        cn_explode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
    }
    else
#   endif
    {
        # 调用 cn_explode_scratchpad 函数对 ctx[0] 进行处理
        cn_explode_scratchpad<ALGO, SOFT_AES, 0>(ctx[0]);
        # 调用 cn_explode_scratchpad 函数对 ctx[1] 进行处理
        cn_explode_scratchpad<ALGO, SOFT_AES, 0>(ctx[1]);
    }

    # 计算 al0，使用 h0[0] 和 h0[4] 的异或结果
    uint64_t al0 = h0[0] ^ h0[4];
    # 计算 al1，使用 h1[0] 和 h1[4] 的异或结果
    uint64_t al1 = h1[0] ^ h1[4];
    # 计算 ah0，使用 h0[1] 和 h0[5] 的异或结果
    uint64_t ah0 = h0[1] ^ h0[5];
    # 计算 ah1，使用 h1[1] 和 h1[5] 的异或结果
    uint64_t ah1 = h1[1] ^ h1[5];

    # 使用 h0 数组中的值创建 __m128i 类型的 bx00 和 bx01
    __m128i bx00 = _mm_set_epi64x(h0[3] ^ h0[7], h0[2] ^ h0[6]);
    __m128i bx01 = _mm_set_epi64x(h0[9] ^ h0[11], h0[8] ^ h0[10]);
    # 使用 h1 数组中的值创建 __m128i 类型的 bx10 和 bx11
    __m128i bx10 = _mm_set_epi64x(h1[3] ^ h1[7], h1[2] ^ h1[6]);
    __m128i bx11 = _mm_set_epi64x(h1[9] ^ h1[11], h1[8] ^ h1[10]);

    # 定义 conc_var0 和 conc_var1 变量
    __m128 conc_var0, conc_var1;
    # 如果 ALGO 为 Algorithm::CN_CCX，则将 conc_var0 和 conc_var1 初始化为 0
    if (ALGO == Algorithm::CN_CCX) {
        conc_var0 = _mm_setzero_ps();
        conc_var1 = _mm_setzero_ps();
        RESTORE_ROUNDING_MODE();
    }

    # 计算 idx0，使用 al0 的值
    uint64_t idx0 = al0;
    # 计算 idx1，使用 al1 的值
    uint64_t idx1 = al1;
# 如果定义了 XMRIG_ALGO_CN_HEAVY，则执行以下代码块
        if (props.isHeavy()) {
            # 从数组 l0 中读取指定索引处的 int64_t 类型数据
            int64_t n = ((int64_t*)&l0[idx0 & MASK])[0];
            # 从数组 l0 中读取指定索引处的 int32_t 类型数据
            int32_t d = ((int32_t*)&l0[idx0 & MASK])[2];
            # 计算 n 除以 (d | 0x5) 的结果
            int64_t q = n / (d | 0x5);
            # 将 n 与 q 的异或结果存入数组 l0 中指定索引处
            ((int64_t*)&l0[idx0 & MASK])[0] = n ^ q;
            # 如果 ALGO 为 Algorithm::CN_HEAVY_XHV，则执行以下代码块
            if (ALGO == Algorithm::CN_HEAVY_XHV) {
                # 对 d 取反
                d = ~d;
            }
            # 将 d 与 q 的异或结果存入 idx0
            idx0 = d ^ q;
        }

        # 从数组 l1 中读取指定索引处的 uint64_t 类型数据
        cl = ((uint64_t*) &l1[idx1 & MASK])[0];
        # 从数组 l1 中读取指定索引处的 uint64_t 类型数据
        ch = ((uint64_t*) &l1[idx1 & MASK])[1];

        # 如果 BASE 为 Algorithm::CN_2，则执行以下代码块
        if (BASE == Algorithm::CN_2) {
            # 如果 props 中的标志位 R 为真，则执行以下代码块
            if (props.isR()) {
                # 执行 VARIANT4_RANDOM_MATH 函数
                VARIANT4_RANDOM_MATH(1, al1, ah1, cl, bx10, bx11);
                # 如果 ALGO 为 Algorithm::CN_R，则执行以下代码块
                if (ALGO == Algorithm::CN_R) {
                    # 对 al1 和 ah1 进行异或操作
                    al1 ^= r1[2] | ((uint64_t)(r1[3]) << 32);
                    ah1 ^= r1[0] | ((uint64_t)(r1[1]) << 32);
                }
            } else {
                # 执行 VARIANT2_INTEGER_MATH 函数
                VARIANT2_INTEGER_MATH(1, cl, cx1);
            }
        }

        # 使用 __umul128 函数计算 idx1 与 cl 的乘积，并将结果存入 lo 和 hi
        lo = __umul128(idx1, cl, &hi);

        # 如果 BASE 为 Algorithm::CN_2，则执行以下代码块
        if (BASE == Algorithm::CN_2) {
            # 如果 ALGO 为 Algorithm::CN_R，则执行以下代码块
            if (ALGO == Algorithm::CN_R) {
                # 执行 VARIANT2_SHUFFLE 函数
                VARIANT2_SHUFFLE(l1, idx1 & MASK, ax1, bx10, bx11, cx1, 0);
            } else {
                # 执行 VARIANT2_SHUFFLE2 函数
                VARIANT2_SHUFFLE2(l1, idx1 & MASK, ax1, bx10, bx11, hi, lo, (((ALGO == Algorithm::CN_RWZ) || (ALGO == Algorithm::CN_UPX2)) ? 1 : 0));
            }
        }

        # 将 al1 加上 hi，ah1 加上 lo
        al1 += hi;
        ah1 += lo;

        # 将 al1 存入数组 l1 中指定索引处
        ((uint64_t*)&l1[idx1 & MASK])[0] = al1;

        # 如果 IS_CN_HEAVY_TUBE 为真，或者 ALGO 为 Algorithm::CN_RTO，则执行以下代码块
        if (IS_CN_HEAVY_TUBE || ALGO == Algorithm::CN_RTO) {
            # 将 ah1 与 tweak1_2_1 和 al1 的异或结果存入数组 l1 中指定索引处
            ((uint64_t*)&l1[idx1 & MASK])[1] = ah1 ^ tweak1_2_1 ^ al1;
        } else if (BASE == Algorithm::CN_1) {
            # 将 ah1 与 tweak1_2_1 的异或结果存入数组 l1 中指定索引处
            ((uint64_t*)&l1[idx1 & MASK])[1] = ah1 ^ tweak1_2_1;
        } else {
            # 将 ah1 存入数组 l1 中指定索引处
            ((uint64_t*)&l1[idx1 & MASK])[1] = ah1;
        }

        # 对 al1 和 ah1 进行异或操作
        al1 ^= cl;
        ah1 ^= ch;
        # 将 al1 存入 idx1
        idx1 = al1;
// 如果定义了 XMRIG_ALGO_CN_HEAVY，则执行以下代码块
        if (props.isHeavy()) {
            // 从 l1 数组中读取第 idx1 个元素，并将其解释为 int64_t 类型
            int64_t n = ((int64_t*)&l1[idx1 & MASK])[0];
            // 从 l1 数组中读取第 idx1 个元素，并将其解释为 int32_t 类型
            int32_t d = ((int32_t*)&l1[idx1 & MASK])[2];
            // 计算 n 除以 (d | 0x5) 的结果
            int64_t q = n / (d | 0x5);
            // 将 n 与 q 的异或结果存入 l1 数组的第 idx1 个元素
            ((int64_t*)&l1[idx1 & MASK])[0] = n ^ q;

            // 如果 ALGO 为 Algorithm::CN_HEAVY_XHV，则执行以下代码块
            if (ALGO == Algorithm::CN_HEAVY_XHV) {
                // 对 d 取反
                d = ~d;
            }

            // 将 idx1 更新为 d 与 q 的异或结果
            idx1 = d ^ q;
        }
// 如果 BASE 为 Algorithm::CN_2，则执行以下代码块
        if (BASE == Algorithm::CN_2) {
            // 将 bx00 的值赋给 bx01
            bx01 = bx00;
            // 将 bx10 的值赋给 bx11
            bx11 = bx10;
        }

        // 将 cx0 的值赋给 bx00
        bx00 = cx0;
        // 将 cx1 的值赋给 bx10
        bx10 = cx1;
    }

// 如果定义了 XMRIG_VAES，则执行以下代码块
#   ifdef XMRIG_VAES
    if (!SOFT_AES && !props.isHeavy() && cn_vaes_enabled) {
        // 调用 cn_implode_scratchpad_vaes_double 函数
        cn_implode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
    }
    else
#   endif
    {
        // 调用 cn_implode_scratchpad 函数，传入 ALGO, SOFT_AES, 0 作为参数
        cn_implode_scratchpad<ALGO, SOFT_AES, 0>(ctx[0]);
        // 调用 cn_implode_scratchpad 函数，传入 ALGO, SOFT_AES, 0 作为参数
        cn_implode_scratchpad<ALGO, SOFT_AES, 0>(ctx[1]);
    }

    // 调用 keccakf 函数，传入 h0, 24 作为参数
    keccakf(h0, 24);
    // 调用 keccakf 函数，传入 h1, 24 作为参数
    keccakf(h1, 24);

    // 调用 extra_hashes 数组中对应索引的函数，传入 ctx[0]->state, 200, output 作为参数
    extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
    // 调用 extra_hashes 数组中对应索引的函数，传入 ctx[1]->state, 200, output + 32 作为参数
    extra_hashes[ctx[1]->state[0] & 3](ctx[1]->state, 200, output + 32);
}


// 如果定义了 XMRIG_FEATURE_ASM，则执行以下代码块
#ifdef XMRIG_FEATURE_ASM
template<Algorithm::Id ALGO>
static NOINLINE void cryptonight_quad_hash_gr_sse41(const uint8_t* __restrict__ input, size_t size, uint8_t* __restrict__ output, cryptonight_ctx** __restrict__ ctx, uint64_t height)
{
    // 根据 ALGO 获取对应的算法属性
    constexpr CnAlgo<ALGO> props;
    // 获取算法的基础算法
    constexpr Algorithm::Id BASE = props.base();

    // 如果 BASE 为 Algorithm::CN_1 并且 size 小于 43，则执行以下代码块
    if (BASE == Algorithm::CN_1 && size < 43) {
        // 将 output 的前 32 * 4 个字节置为 0
        memset(output, 0, 32 * 4);
        // 返回
        return;
    }

    // 调用 keccak 函数，传入 input + size * 0, size, ctx[0]->state 作为参数
    keccak(input + size * 0, size, ctx[0]->state);
    // 调用 keccak 函数，传入 input + size * 1, size, ctx[1]->state 作为参数
    keccak(input + size * 1, size, ctx[1]->state);
    // 调用 keccak 函数，传入 input + size * 2, size, ctx[2]->state 作为参数
    keccak(input + size * 2, size, ctx[2]->state);
    // 调用 keccak 函数，传入 input + size * 3, size, ctx[3]->state 作为参数
    keccak(input + size * 3, size, ctx[3]->state);

    // 如果 props.half_mem() 返回 true，则执行以下代码块
    if (props.half_mem()) {
        // 将 ctx[0]->first_half 置为 true
        ctx[0]->first_half = true;
        // 将 ctx[1]->first_half 置为 true
        ctx[1]->first_half = true;
        // 将 ctx[2]->first_half 置为 true
        ctx[2]->first_half = true;
        // 将 ctx[3]->first_half 置为 true
        ctx[3]->first_half = true;
    }

// 如果定义了 XMRIG_VAES，则执行以下代码块
#   ifdef XMRIG_VAES
    # 如果属性不是重型，并且 cn_vaes_enabled 为真，则执行以下操作
    if (!props.isHeavy() && cn_vaes_enabled) {
        # 使用 VAES 指令集对上下文中的数据进行处理
        cn_explode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
        cn_explode_scratchpad_vaes_double(ctx[2], ctx[3], props.memory(), props.half_mem());
    }
    # 如果属性是重型或者 cn_vaes_enabled 为假，则执行以下操作
    else
    # 结束 if 语句
    {
        # 对四个上下文对象执行 cn_explode_scratchpad 函数
        cn_explode_scratchpad<ALGO, false, 0>(ctx[0]);
        cn_explode_scratchpad<ALGO, false, 0>(ctx[1]);
        cn_explode_scratchpad<ALGO, false, 0>(ctx[2]);
        cn_explode_scratchpad<ALGO, false, 0>(ctx[3]);
    }

    # 初始化 VARIANT1，设置 ctx[0] 的 tweak1_2
    VARIANT1_INIT(0); ctx[0]->tweak1_2 = tweak1_2_0;
    # 初始化 VARIANT1，设置 ctx[1] 的 tweak1_2
    VARIANT1_INIT(1); ctx[1]->tweak1_2 = tweak1_2_1;
    # 初始化 VARIANT1，设置 ctx[2] 的 tweak1_2
    VARIANT1_INIT(2); ctx[2]->tweak1_2 = tweak1_2_2;
    # 初始化 VARIANT1，设置 ctx[3] 的 tweak1_2
    VARIANT1_INIT(3); ctx[3]->tweak1_2 = tweak1_2_3;

    # 设置 ctx[0] 的 tweak1_table
    ctx[0]->tweak1_table = tweak1_table;

    # 根据不同的算法类型调用不同的函数
    if (ALGO == Algorithm::CN_GR_0) cn_gr0_quad_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_1) cn_gr1_quad_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_2) cn_gr2_quad_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_3) cn_gr3_quad_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_4) cn_gr4_quad_mainloop_asm(ctx);
    if (ALGO == Algorithm::CN_GR_5) cn_gr5_quad_mainloop_asm(ctx);

    # 如果 VAES 可用且不是重型算法，则执行以下代码块
    # 否则执行下面的 else 代码块
    # ifdef XMRIG_VAES
    if (!props.isHeavy() && cn_vaes_enabled) {
        # 对 ctx[0] 和 ctx[1] 执行 cn_implode_scratchpad_vaes_double 函数
        cn_implode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
        # 对 ctx[2] 和 ctx[3] 执行 cn_implode_scratchpad_vaes_double 函数
        cn_implode_scratchpad_vaes_double(ctx[2], ctx[3], props.memory(), props.half_mem());
    }
    else
    # endif
    {
        # 对四个上下文对象执行 cn_implode_scratchpad 函数
        cn_implode_scratchpad<ALGO, false, 0>(ctx[0]);
        cn_implode_scratchpad<ALGO, false, 0>(ctx[1]);
        cn_implode_scratchpad<ALGO, false, 0>(ctx[2]);
        cn_implode_scratchpad<ALGO, false, 0>(ctx[3]);
    }

    # 对 ctx[0] 的状态执行 keccakf 函数
    keccakf(reinterpret_cast<uint64_t*>(ctx[0]->state), 24);
    # 对 ctx[1] 的状态执行 keccakf 函数
    keccakf(reinterpret_cast<uint64_t*>(ctx[1]->state), 24);
    # 对 ctx[2] 的状态执行 keccakf 函数
    keccakf(reinterpret_cast<uint64_t*>(ctx[2]->state), 24);
    # 对 ctx[3] 的状态执行 keccakf 函数
    keccakf(reinterpret_cast<uint64_t*>(ctx[3]->state), 24);

    # 对 ctx[0] 的状态执行 extra_hashes 中对应的函数，并将结果写入 output
    extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
    # 对 ctx[1] 的状态执行 extra_hashes 中对应的函数，并将结果写入 output + 32
    extra_hashes[ctx[1]->state[0] & 3](ctx[1]->state, 200, output + 32);
    # 对 ctx[2] 的状态执行 extra_hashes 中对应的函数，并将结果写入 output + 64
    extra_hashes[ctx[2]->state[0] & 3](ctx[2]->state, 200, output + 64);
    # 对 ctx[3] 的状态执行 extra_hashes 中对应的函数，并将结果写入 output + 96
    extra_hashes[ctx[3]->state[0] & 3](ctx[3]->state, 200, output + 96);
}
#endif
# 定义 CN_STEP1 宏，用于执行第一步的加密算法操作
#define CN_STEP1(a, b0, b1, c, l, ptr, idx, conc_var) \
    # 将指针 ptr 指向 l 数组中的特定位置
    ptr = reinterpret_cast<__m128i*>(&l[idx & MASK]); \
    # 从指定位置加载数据到 c 变量中
    c = _mm_load_si128(ptr);                          \
    # 如果算法为 CN_CCX，则执行 cryptonight_conceal_tweak 操作
    if (ALGO == Algorithm::CN_CCX) {                  \
        cryptonight_conceal_tweak(c, conc_var);       \
    }

# 定义 CN_STEP2 宏，用于执行第二步的加密算法操作
#define CN_STEP2(a, b0, b1, c, l, ptr, idx)                                             \
    # 如果为 CN_HEAVY_TUBE 算法，则执行 aes_round_tweak_div 操作
    if (IS_CN_HEAVY_TUBE) {                                                             \
        c = aes_round_tweak_div(c, a);                                                  \
    }                                                                                   \
    # 如果为 SOFT_AES 算法，则执行 soft_aesenc 操作
    else if (SOFT_AES) {                                                                \
        c = soft_aesenc(&c, a, (const uint32_t*)saes_table);                            \
    } else {                                                                            \
        c = _mm_aesenc_si128(c, a);                                                     \
    }                                                                                   \
    # 如果算法为 CN_1 或 CN_2，则执行 cryptonight_monero_tweak 操作，否则执行 _mm_xor_si128 操作
    if (BASE == Algorithm::CN_1 || BASE == Algorithm::CN_2) {                           \
        cryptonight_monero_tweak<ALGO>((uint64_t*)ptr, l, idx & MASK, a, b0, b1, c);    \
    } else {                                                                            \
        _mm_store_si128(ptr, _mm_xor_si128(b0, c));                                     \
    }

# 定义 CN_STEP3 宏，用于执行第三步的加密算法操作
#define CN_STEP3(part, a, b0, b1, c, l, ptr, idx)     \
    # 将 c 转换为 64 位整数并赋值给 idx
    idx = _mm_cvtsi128_si64(c);                       \
    # 将指针 ptr 指向 l 数组中的特定位置
    ptr = reinterpret_cast<__m128i*>(&l[idx & MASK]); \
    # 从指定位置加载数据到 cl##part 和 ch##part 变量中
    uint64_t cl##part = ((uint64_t*)ptr)[0];          \
    uint64_t ch##part = ((uint64_t*)ptr)[1];

# 定义 CN_STEP4 宏，用于执行第四步的加密算法操作
#define CN_STEP4(part, a, b0, b1, c, l, mc, ptr, idx)                                                       \
    // 定义两个 64 位无符号整数变量 al##part 和 ah##part
    uint64_t al##part, ah##part;                                                                            \
    // 如果基础算法是 CN_2
    if (BASE == Algorithm::CN_2) {                                                                          \
        // 如果 props.isR() 返回 true
        if (props.isR()) {                                                                                  \
            // 将 a 的低 64 位存入 al##part，将 a 的高 64 位存入 ah##part
            al##part = _mm_cvtsi128_si64(a);                                                                \
            ah##part = _mm_cvtsi128_si64(_mm_srli_si128(a, 8));                                             \
            // 使用 VARIANT4_RANDOM_MATH 处理 al##part 和 ah##part
            VARIANT4_RANDOM_MATH(part, al##part, ah##part, cl##part, b0, b1);                               \
            // 如果算法是 CN_R
            if (ALGO == Algorithm::CN_R) {                                                                  \
                // 对 al##part 进行异或操作
                al##part ^= r##part[2] | ((uint64_t)(r##part[3]) << 32);                                    \
                // 对 ah##part 进行异或操作
                ah##part ^= r##part[0] | ((uint64_t)(r##part[1]) << 32);                                    \
            }                                                                                               \
        } else {                                                                                            \
            // 使用 VARIANT2_INTEGER_MATH 处理 cl##part 和 c
            VARIANT2_INTEGER_MATH(part, cl##part, c);                                                       \
        }                                                                                                   \
    }                                                                                                       \
    // 使用 __umul128 计算 idx 与 cl##part 的乘积，存入 lo，进位存入 hi
    lo = __umul128(idx, cl##part, &hi);                                                                     \
    # 如果基础算法是 CN_2
    if (BASE == Algorithm::CN_2) {                                                                          \
        # 如果算法是 CN_R
        if (ALGO == Algorithm::CN_R) {                                                                      \
            # 使用 VARIANT2_SHUFFLE 宏进行数据处理
            VARIANT2_SHUFFLE(l, idx & MASK, a, b0, b1, c, 0);                                               \
        } else {                                                                                            \
            # 使用 VARIANT2_SHUFFLE2 宏进行数据处理
            VARIANT2_SHUFFLE2(l, idx & MASK, a, b0, b1, hi, lo, (((ALGO == Algorithm::CN_RWZ) || (ALGO == Algorithm::CN_UPX2)) ? 1 : 0)); \
        }                                                                                                   \
    }                                                                                                       \
    # 如果算法是 CN_R
    if (ALGO == Algorithm::CN_R) {                                                                          \
        # 设置 a 的值为 ah##part 和 al##part 组成的 128 位整数
        a = _mm_set_epi64x(ah##part, al##part);                                                             \
    }                                                                                                       \
    # 将 a 和 _mm_set_epi64x(lo, hi) 相加
    a = _mm_add_epi64(a, _mm_set_epi64x(lo, hi));                                                           \
                                                                                                            \
    # 如果基础算法是CN_1
    if (BASE == Algorithm::CN_1) {                                                                          
        # 将a和mc进行异或操作，并将结果存储到ptr指向的位置
        _mm_store_si128(ptr, _mm_xor_si128(a, mc));                                                         
                                                                                                            
        # 如果是CN_HEAVY_TUBE或者ALGO是CN_RTO算法
        if (IS_CN_HEAVY_TUBE || ALGO == Algorithm::CN_RTO) {                                                
            # 将ptr指向的位置的前后8字节进行异或操作
            ((uint64_t*)ptr)[1] ^= ((uint64_t*)ptr)[0];                                                     
        }                                                                                                   
    } else {                                                                                                
        # 如果基础算法不是CN_1，则直接将a的值存储到ptr指向的位置
        _mm_store_si128(ptr, a);                                                                            
    }                                                                                                       
                                                                                                            
    # 将a和_mm_set_epi64x(ch##part, cl##part)进行异或操作
    a = _mm_xor_si128(a, _mm_set_epi64x(ch##part, cl##part));                                               
    # 将a的值转换为64位整数，并存储到idx中
    idx = _mm_cvtsi128_si64(a);                                                                             
    # 如果属性表明是重型操作
    if (props.isHeavy()) {                                                                                  
        # 从数组中读取第 idx & MASK 位置的 8 字节数据，转换成 int64_t 类型
        int64_t n = ((int64_t*)&l[idx & MASK])[0];                                                          
        # 从数组中读取第 idx & MASK 位置的 4 字节数据，转换成 int32_t 类型
        int32_t d = ((int32_t*)&l[idx & MASK])[2];                                                          
        # 计算 n 除以 (d | 0x5) 的结果，存入 q
        int64_t q = n / (d | 0x5);                                                                          
        # 将 n 与 q 的异或结果存入数组中第 idx & MASK 位置的 8 字节数据
        ((int64_t*)&l[idx & MASK])[0] = n ^ q;                                                              
        # 如果是 CN_HEAVY_XHV 算法
        if (IS_CN_HEAVY_XHV) {                                                                              
            # 对 d 取反
            d = ~d;                                                                                         
        }                                                                                                   
        # 将 d 与 q 的异或结果存入 idx
        idx = d ^ q;                                                                                        
    }                                                                                                       
    # 如果 BASE 是 CN_2 算法
    if (BASE == Algorithm::CN_2) {                                                                          
        # 将 b0 的值赋给 b1
        b1 = b0;                                                                                            
    }                                                                                                       
    # 将 c 的值赋给 b0
    b0 = c;
# 定义一个宏，用于初始化常量和变量
#define CONST_INIT(ctx, n)                                                                       \
    # 定义一个 128 位整数变量 mc##n
    __m128i mc##n;                                                                               \
    # 定义一个 128 位整数变量 division_result_xmm_##n
    __m128i division_result_xmm_##n;                                                             \
    # 定义一个 128 位整数变量 sqrt_result_xmm_##n
    __m128i sqrt_result_xmm_##n;                                                                 \
    # 如果基础算法是 CN_1
    if (BASE == Algorithm::CN_1) {                                                               \
        # 从输入数据中读取一部分内容，进行位运算后赋值给 mc##n
        mc##n = _mm_set_epi64x(*reinterpret_cast<const uint64_t*>(input + n * size + 35) ^       \
                               *(reinterpret_cast<const uint64_t*>((ctx)->state) + 24), 0);      \
    }                                                                                            \
    # 如果基础算法是 CN_2
    if (BASE == Algorithm::CN_2) {                                                               \
        # 将 h##n[12] 赋值给 division_result_xmm_##n
        division_result_xmm_##n = _mm_cvtsi64_si128(h##n[12]);                                   \
        # 将 h##n[13] 赋值给 sqrt_result_xmm_##n
        sqrt_result_xmm_##n = _mm_cvtsi64_si128(h##n[13]);                                       \
    }                                                                                            \
    # 将 h##n[1] ^ h##n[5] 赋值给 ax##n
    __m128i ax##n = _mm_set_epi64x(h##n[1] ^ h##n[5], h##n[0] ^ h##n[4]);                        \
    # 将 h##n[3] ^ h##n[7] 赋值给 bx##n##0
    __m128i bx##n##0 = _mm_set_epi64x(h##n[3] ^ h##n[7], h##n[2] ^ h##n[6]);                     \
    # 将 h##n[9] ^ h##n[11] 赋值给 bx##n##1
    __m128i bx##n##1 = _mm_set_epi64x(h##n[9] ^ h##n[11], h##n[8] ^ h##n[10]);                   \
    # 将 0 赋值给 cx##n
    __m128i cx##n = _mm_setzero_si128();                                                         \
    # 定义一个变量 conc_var##n
    __m128 conc_var##n;                                                                          \
    # 如果算法是 CN_CCX
    if (ALGO == Algorithm::CN_CCX) {                                                             \
        # 将 0 赋值给 conc_var##n
        conc_var##n = _mm_setzero_ps();                                                          \
    }                                                                                            \
    # 使用 VARIANT4_RANDOM_MATH_INIT 函数对变量 n 进行初始化
    VARIANT4_RANDOM_MATH_INIT(n);
// 使用模板参数ALGO和SOFT_AES，以及输入和输出数据指针，进行三次cryptonight哈希计算
template<Algorithm::Id ALGO, bool SOFT_AES>
inline void cryptonight_triple_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
    // 根据ALGO获取算法属性
    constexpr CnAlgo<ALGO> props;
    // 获取MASK值
    constexpr size_t MASK        = props.mask();
    // 获取基础算法ID
    constexpr Algorithm::Id BASE = props.base();

#   ifdef XMRIG_ALGO_CN_HEAVY
    // 根据条件编译判断是否为CN_HEAVY_TUBE或CN_HEAVY_XHV算法
    constexpr bool IS_CN_HEAVY_TUBE = ALGO == Algorithm::CN_HEAVY_TUBE;
    constexpr bool IS_CN_HEAVY_XHV  = ALGO == Algorithm::CN_HEAVY_XHV;
#   else
    // 如果没有定义XMRIG_ALGO_CN_HEAVY，则默认为false
    constexpr bool IS_CN_HEAVY_TUBE = false;
    constexpr bool IS_CN_HEAVY_XHV  = false;
#   endif

    // 如果基础算法为CN_1且输入数据大小小于43，则将输出数据置零并返回
    if (BASE == Algorithm::CN_1 && size < 43) {
        memset(output, 0, 32 * 3);
        return;
    }

    // 循环三次进行keccak哈希计算和内存操作
    for (size_t i = 0; i < 3; i++) {
        keccak(input + size * i, size, ctx[i]->state);
        // 如果算法属性为half_mem，则设置ctx[i]->first_half为true
        if (props.half_mem()) {
            ctx[i]->first_half = true;
        }
        // 调用cn_explode_scratchpad函数对内存进行操作
        cn_explode_scratchpad<ALGO, SOFT_AES, 0>(ctx[i]);
    }

    // 定义指向内存和状态的指针
    uint8_t* l0  = ctx[0]->memory;
    uint8_t* l1  = ctx[1]->memory;
    uint8_t* l2  = ctx[2]->memory;
    uint64_t* h0 = reinterpret_cast<uint64_t*>(ctx[0]->state);
    uint64_t* h1 = reinterpret_cast<uint64_t*>(ctx[1]->state);
    uint64_t* h2 = reinterpret_cast<uint64_t*>(ctx[2]->state);

    // 初始化ctx[0]、ctx[1]、ctx[2]的常量
    CONST_INIT(ctx[0], 0);
    CONST_INIT(ctx[1], 1);
    CONST_INIT(ctx[2], 2);
    // 设置舍入模式
    VARIANT2_SET_ROUNDING_MODE();
    // 如果算法为CN_CCX，则恢复舍入模式
    if (ALGO == Algorithm::CN_CCX) {
        RESTORE_ROUNDING_MODE();
    }

    // 定义并初始化idx0、idx1、idx2
    uint64_t idx0, idx1, idx2;
    idx0 = _mm_cvtsi128_si64(ax0);
    idx1 = _mm_cvtsi128_si64(ax1);
    idx2 = _mm_cvtsi128_si64(ax2);
}
    // 对于给定的迭代次数，执行以下操作
    for (size_t i = 0; i < props.iterations(); i++) {
        uint64_t hi, lo;
        __m128i *ptr0, *ptr1, *ptr2;

        // 执行 CN_STEP1 操作，并将结果存储在指定的变量中
        CN_STEP1(ax0, bx00, bx01, cx0, l0, ptr0, idx0, conc_var0);
        CN_STEP1(ax1, bx10, bx11, cx1, l1, ptr1, idx1, conc_var1);
        CN_STEP1(ax2, bx20, bx21, cx2, l2, ptr2, idx2, conc_var2);

        // 执行 CN_STEP2 操作
        CN_STEP2(ax0, bx00, bx01, cx0, l0, ptr0, idx0);
        CN_STEP2(ax1, bx10, bx11, cx1, l1, ptr1, idx1);
        CN_STEP2(ax2, bx20, bx21, cx2, l2, ptr2, idx2);

        // 执行 CN_STEP3 操作
        CN_STEP3(0, ax0, bx00, bx01, cx0, l0, ptr0, idx0);
        CN_STEP3(1, ax1, bx10, bx11, cx1, l1, ptr1, idx1);
        CN_STEP3(2, ax2, bx20, bx21, cx2, l2, ptr2, idx2);

        // 执行 CN_STEP4 操作
        CN_STEP4(0, ax0, bx00, bx01, cx0, l0, mc0, ptr0, idx0);
        CN_STEP4(1, ax1, bx10, bx11, cx1, l1, mc1, ptr1, idx1);
        CN_STEP4(2, ax2, bx20, bx21, cx2, l2, mc2, ptr2, idx2);
    }

    // 对于给定的迭代次数，执行以下操作
    for (size_t i = 0; i < 3; i++) {
        // 对指定的上下文执行 cn_implode_scratchpad 操作
        cn_implode_scratchpad<ALGO, SOFT_AES, 0>(ctx[i]);
        // 对指定的状态执行 keccakf 操作
        keccakf(reinterpret_cast<uint64_t*>(ctx[i]->state), 24);
        // 对指定的状态执行额外的哈希操作，并将结果存储在指定的输出位置
        extra_hashes[ctx[i]->state[0] & 3](ctx[i]->state, 200, output + 32 * i);
    }
# 定义一个内联函数，用于执行Cryptonight Quad Hash算法
template<Algorithm::Id ALGO, bool SOFT_AES>
inline void cryptonight_quad_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
#   ifdef XMRIG_FEATURE_ASM
    # 如果不是软件AES加密
    if (!SOFT_AES) {
        # 根据不同的算法类型执行不同的操作
        switch (ALGO) {
        case Algorithm::CN_GR_0:
        case Algorithm::CN_GR_1:
        case Algorithm::CN_GR_2:
        case Algorithm::CN_GR_3:
        case Algorithm::CN_GR_4:
        case Algorithm::CN_GR_5:
            # 如果支持SSE4.1指令集，则执行对应的算法
            if (cn_sse41_enabled) {
                cryptonight_quad_hash_gr_sse41<ALGO>(input, size, output, ctx, height);
                return;
            }
            break;

        default:
            break;
        }
    }
#   endif

    # 根据算法类型获取对应的属性
    constexpr CnAlgo<ALGO> props;
    constexpr size_t MASK        = props.mask();
    constexpr Algorithm::Id BASE = props.base();

#   ifdef XMRIG_ALGO_CN_HEAVY
    # 如果支持CN_HEAVY算法，则设置对应的标志位
    constexpr bool IS_CN_HEAVY_TUBE = ALGO == Algorithm::CN_HEAVY_TUBE;
    constexpr bool IS_CN_HEAVY_XHV  = ALGO == Algorithm::CN_HEAVY_XHV;
#   else
    # 否则将标志位设置为false
    constexpr bool IS_CN_HEAVY_TUBE = false;
    constexpr bool IS_CN_HEAVY_XHV  = false;
#   endif

    # 如果是CN_1算法且输入数据大小小于43，则将输出数据置为0并返回
    if (BASE == Algorithm::CN_1 && size < 43) {
        memset(output, 0, 32 * 4);
        return;
    }

    # 对输入数据进行四次Keccak哈希运算
    for (size_t i = 0; i < 4; i++) {
        keccak(input + size * i, size, ctx[i]->state);
        # 如果属性要求使用一半内存，则设置对应的标志位
        if (props.half_mem()) {
            ctx[i]->first_half = true;
        }
    }

#   ifdef XMRIG_VAES
    # 如果不是软件AES加密且不是重型算法且支持VAES指令集，则执行对应的操作
    if (!SOFT_AES && !props.isHeavy() && cn_vaes_enabled) {
        cn_explode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
        cn_explode_scratchpad_vaes_double(ctx[2], ctx[3], props.memory(), props.half_mem());
    }
    else
#   endif
    {
        # 否则执行普通的内存爆炸操作
        cn_explode_scratchpad<ALGO, SOFT_AES, 0>(ctx[0]);
        cn_explode_scratchpad<ALGO, SOFT_AES, 0>(ctx[1]);
        cn_explode_scratchpad<ALGO, SOFT_AES, 0>(ctx[2]);
        cn_explode_scratchpad<ALGO, SOFT_AES, 0>(ctx[3]);
    }
    // 为每个上下文的内存分配指针
    uint8_t* l0  = ctx[0]->memory;
    uint8_t* l1  = ctx[1]->memory;
    uint8_t* l2  = ctx[2]->memory;
    uint8_t* l3  = ctx[3]->memory;
    // 将上下文的状态转换为 uint64_t 指针
    uint64_t* h0 = reinterpret_cast<uint64_t*>(ctx[0]->state);
    uint64_t* h1 = reinterpret_cast<uint64_t*>(ctx[1]->state);
    uint64_t* h2 = reinterpret_cast<uint64_t*>(ctx[2]->state);
    uint64_t* h3 = reinterpret_cast<uint64_t*>(ctx[3]->state);

    // 初始化上下文的常量值
    CONST_INIT(ctx[0], 0);
    CONST_INIT(ctx[1], 1);
    CONST_INIT(ctx[2], 2);
    CONST_INIT(ctx[3], 3);
    // 设置舍入模式
    VARIANT2_SET_ROUNDING_MODE();
    // 如果算法为 CN_CCX，则恢复舍入模式
    if (ALGO == Algorithm::CN_CCX) {
        RESTORE_ROUNDING_MODE();
    }

    // 定义用于存储索引的变量
    uint64_t idx0, idx1, idx2, idx3;
    // 将 __m128i 类型的寄存器转换为 uint64_t 类型的索引
    idx0 = _mm_cvtsi128_si64(ax0);
    idx1 = _mm_cvtsi128_si64(ax1);
    idx2 = _mm_cvtsi128_si64(ax2);
    idx3 = _mm_cvtsi128_si64(ax3);

    // 循环迭代执行算法的步骤
    for (size_t i = 0; i < props.iterations(); i++) {
        uint64_t hi, lo;
        __m128i *ptr0, *ptr1, *ptr2, *ptr3;

        // 执行算法的第一步
        CN_STEP1(ax0, bx00, bx01, cx0, l0, ptr0, idx0, conc_var0);
        CN_STEP1(ax1, bx10, bx11, cx1, l1, ptr1, idx1, conc_var1);
        CN_STEP1(ax2, bx20, bx21, cx2, l2, ptr2, idx2, conc_var2);
        CN_STEP1(ax3, bx30, bx31, cx3, l3, ptr3, idx3, conc_var3);

        // 执行算法的第二步
        CN_STEP2(ax0, bx00, bx01, cx0, l0, ptr0, idx0);
        CN_STEP2(ax1, bx10, bx11, cx1, l1, ptr1, idx1);
        CN_STEP2(ax2, bx20, bx21, cx2, l2, ptr2, idx2);
        CN_STEP2(ax3, bx30, bx31, cx3, l3, ptr3, idx3);

        // 执行算法的第三步
        CN_STEP3(0, ax0, bx00, bx01, cx0, l0, ptr0, idx0);
        CN_STEP3(1, ax1, bx10, bx11, cx1, l1, ptr1, idx1);
        CN_STEP3(2, ax2, bx20, bx21, cx2, l2, ptr2, idx2);
        CN_STEP3(3, ax3, bx30, bx31, cx3, l3, ptr3, idx3);

        // 执行算法的第四步
        CN_STEP4(0, ax0, bx00, bx01, cx0, l0, mc0, ptr0, idx0);
        CN_STEP4(1, ax1, bx10, bx11, cx1, l1, mc1, ptr1, idx1);
        CN_STEP4(2, ax2, bx20, bx21, cx2, l2, mc2, ptr2, idx2);
        CN_STEP4(3, ax3, bx30, bx31, cx3, l3, mc3, ptr3, idx3);
    }
# 如果定义了XMRIG_VAES并且不是软件AES并且不是重型算法并且启用了VAES，则执行以下操作
if (!SOFT_AES && !props.isHeavy() && cn_vaes_enabled) {
    # 使用VAES对ctx[0]和ctx[1]的内存进行双倍压缩
    cn_implode_scratchpad_vaes_double(ctx[0], ctx[1], props.memory(), props.half_mem());
    # 使用VAES对ctx[2]和ctx[3]的内存进行双倍压缩
    cn_implode_scratchpad_vaes_double(ctx[2], ctx[3], props.memory(), props.half_mem());
}
# 如果不满足上述条件，则执行以下操作
else {
    # 对ctx[0]的内存进行压缩
    cn_implode_scratchpad<ALGO, SOFT_AES, 0>(ctx[0]);
    # 对ctx[1]的内存进行压缩
    cn_implode_scratchpad<ALGO, SOFT_AES, 0>(ctx[1]);
    # 对ctx[2]的内存进行压缩
    cn_implode_scratchpad<ALGO, SOFT_AES, 0>(ctx[2]);
    # 对ctx[3]的内存进行压缩
    cn_implode_scratchpad<ALGO, SOFT_AES, 0>(ctx[3]);
}

# 对ctx数组中的每个元素执行以下操作
for (size_t i = 0; i < 4; i++) {
    # 对ctx[i]->state进行Keccak压缩函数，执行24轮
    keccakf(reinterpret_cast<uint64_t*>(ctx[i]->state), 24);
    # 调用extra_hashes数组中的函数，对ctx[i]->state进行处理，将结果存储到output数组中
    extra_hashes[ctx[i]->state[0] & 3](ctx[i]->state, 200, output + 32 * i);
}

# 定义模板函数cryptonight_penta_hash，接受输入、输出、ctx数组、以及高度作为参数
template<Algorithm::Id ALGO, bool SOFT_AES>
inline void cryptonight_penta_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
    # 根据ALGO获取算法属性
    constexpr CnAlgo<ALGO> props;
    # 获取掩码值
    constexpr size_t MASK        = props.mask();
    # 获取基础算法ID
    constexpr Algorithm::Id BASE = props.base();

    # 如果定义了XMRIG_ALGO_CN_HEAVY，则根据ALGO的不同进行不同的处理
    # 如果ALGO为CN_HEAVY_TUBE，则IS_CN_HEAVY_TUBE为true，否则为false
    # 如果ALGO为CN_HEAVY_XHV，则IS_CN_HEAVY_XHV为true，否则为false
    if (BASE == Algorithm::CN_1 && size < 43) {
        # 如果基础算法为CN_1并且输入大小小于43，则将输出数组清零并返回
        memset(output, 0, 32 * 5);
        return;
    }

    # 对ctx数组中的每个元素执行以下操作
    for (size_t i = 0; i < 5; i++) {
        # 对输入数据进行Keccak压缩函数，将结果存储到ctx[i]->state中
        keccak(input + size * i, size, ctx[i]->state);
        # 如果内存大小为一半，则将ctx[i]->first_half设置为true
        if (props.half_mem()) {
            ctx[i]->first_half = true;
        }
        # 对ctx[i]的内存进行展开
        cn_explode_scratchpad<ALGO, SOFT_AES, 0>(ctx[i]);
    }

    # 获取ctx数组中每个元素的内存指针
    uint8_t* l0  = ctx[0]->memory;
    uint8_t* l1  = ctx[1]->memory;
    uint8_t* l2  = ctx[2]->memory;
    uint8_t* l3  = ctx[3]->memory;
    uint8_t* l4  = ctx[4]->memory;
    # 获取ctx[0]->state的uint64_t指针
    uint64_t* h0 = reinterpret_cast<uint64_t*>(ctx[0]->state);
}
    # 将ctx[1]的状态转换为uint64_t类型的指针
    uint64_t* h1 = reinterpret_cast<uint64_t*>(ctx[1]->state);
    # 将ctx[2]的状态转换为uint64_t类型的指针
    uint64_t* h2 = reinterpret_cast<uint64_t*>(ctx[2]->state);
    # 将ctx[3]的状态转换为uint64_t类型的指针
    uint64_t* h3 = reinterpret_cast<uint64_t*>(ctx[3]->state);
    # 将ctx[4]的状态转换为uint64_t类型的指针
    uint64_t* h4 = reinterpret_cast<uint64_t*>(ctx[4]->state);

    # 对ctx[0]进行常量初始化
    CONST_INIT(ctx[0], 0);
    # 对ctx[1]进行常量初始化
    CONST_INIT(ctx[1], 1);
    # 对ctx[2]进行常量初始化
    CONST_INIT(ctx[2], 2);
    # 对ctx[3]进行常量初始化
    CONST_INIT(ctx[3], 3);
    # 对ctx[4]进行常量初始化
    CONST_INIT(ctx[4], 4);
    # 设置舍入模式
    VARIANT2_SET_ROUNDING_MODE();
    # 如果算法为CN_CCX，则恢复舍入模式
    if (ALGO == Algorithm::CN_CCX) {
        RESTORE_ROUNDING_MODE();
    }

    # 将ax0的低64位转换为uint64_t类型
    uint64_t idx0, idx1, idx2, idx3, idx4;
    idx0 = _mm_cvtsi128_si64(ax0);
    # 将ax1的低64位转换为uint64_t类型
    idx1 = _mm_cvtsi128_si64(ax1);
    # 将ax2的低64位转换为uint64_t类型
    idx2 = _mm_cvtsi128_si64(ax2);
    # 将ax3的低64位转换为uint64_t类型
    idx3 = _mm_cvtsi128_si64(ax3);
    # 将ax4的低64位转换为uint64_t类型
    idx4 = _mm_cvtsi128_si64(ax4);
    // 对于给定的迭代次数，执行以下操作
    for (size_t i = 0; i < props.iterations(); i++) {
        uint64_t hi, lo;
        __m128i *ptr0, *ptr1, *ptr2, *ptr3, *ptr4;

        // 执行 CN_STEP1 操作，并将结果存储在指定的变量中
        CN_STEP1(ax0, bx00, bx01, cx0, l0, ptr0, idx0, conc_var0);
        CN_STEP1(ax1, bx10, bx11, cx1, l1, ptr1, idx1, conc_var1);
        CN_STEP1(ax2, bx20, bx21, cx2, l2, ptr2, idx2, conc_var2);
        CN_STEP1(ax3, bx30, bx31, cx3, l3, ptr3, idx3, conc_var3);
        CN_STEP1(ax4, bx40, bx41, cx4, l4, ptr4, idx4, conc_var4);

        // 执行 CN_STEP2 操作，并将结果存储在指定的变量中
        CN_STEP2(ax0, bx00, bx01, cx0, l0, ptr0, idx0);
        CN_STEP2(ax1, bx10, bx11, cx1, l1, ptr1, idx1);
        CN_STEP2(ax2, bx20, bx21, cx2, l2, ptr2, idx2);
        CN_STEP2(ax3, bx30, bx31, cx3, l3, ptr3, idx3);
        CN_STEP2(ax4, bx40, bx41, cx4, l4, ptr4, idx4);

        // 执行 CN_STEP3 操作，并将结果存储在指定的变量中
        CN_STEP3(0, ax0, bx00, bx01, cx0, l0, ptr0, idx0);
        CN_STEP3(1, ax1, bx10, bx11, cx1, l1, ptr1, idx1);
        CN_STEP3(2, ax2, bx20, bx21, cx2, l2, ptr2, idx2);
        CN_STEP3(3, ax3, bx30, bx31, cx3, l3, ptr3, idx3);
        CN_STEP3(4, ax4, bx40, bx41, cx4, l4, ptr4, idx4);

        // 执行 CN_STEP4 操作，并将结果存储在指定的变量中
        CN_STEP4(0, ax0, bx00, bx01, cx0, l0, mc0, ptr0, idx0);
        CN_STEP4(1, ax1, bx10, bx11, cx1, l1, mc1, ptr1, idx1);
        CN_STEP4(2, ax2, bx20, bx21, cx2, l2, mc2, ptr2, idx2);
        CN_STEP4(3, ax3, bx30, bx31, cx3, l3, mc3, ptr3, idx3);
        CN_STEP4(4, ax4, bx40, bx41, cx4, l4, mc4, ptr4, idx4);
    }

    // 对于给定的迭代次数，执行以下操作
    for (size_t i = 0; i < 5; i++) {
        // 执行 cn_implode_scratchpad 操作
        cn_implode_scratchpad<ALGO, SOFT_AES, 0>(ctx[i]);
        // 执行 keccakf 操作
        keccakf(reinterpret_cast<uint64_t*>(ctx[i]->state), 24);
        // 执行 extra_hashes 操作，并将结果存储在指定的变量中
        extra_hashes[ctx[i]->state[0] & 3](ctx[i]->state, 200, output + 32 * i);
    }
} 
/* 结束 xmrig 命名空间 */

} /* 结束 xmrig 命名空间 */

#endif /* 结束 XMRIG_CRYPTONIGHT_X86_H 头文件的条件编译 */
```