# `xmrig\src\crypto\cn\CryptoNight_x86_vaes.cpp`

```cpp
/*
 * XMRig
 * 版权所有 2010      Jeff Garzik <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler      <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466    <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee   <jayddee246@gmail.com>
 * 版权所有 2017-2019 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 * 本程序是自由软件：您可以重新发布或修改
 * 根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 * 要么是许可证的第3版，要么（在您的选择下）是任何更高版本。
 *
 * 本程序是希望它有用，但没有任何担保；甚至没有适销性或特定用途的暗示担保。
 * 有关更多详细信息，请参见 GNU 通用公共许可证。
 *
 * 您应该已经收到了 GNU 通用公共许可证的副本
 * 与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#include "CryptoNight_x86_vaes.h"
#include "CryptoNight_monero.h"
#include "CryptoNight.h"


#ifdef __GNUC__
#   include <x86intrin.h>
#if !defined(__clang__) && !defined(__ICC) && __GNUC__ < 10
// 定义内联函数 _mm256_loadu2_m128i，加载两个 __m128i 类型的数据到 __m256i 类型
static inline __m256i
__attribute__((__always_inline__))
  _mm256_loadu2_m128i(const __m128i* const hiaddr, const __m128i* const loaddr)
{
    return _mm256_inserti128_si256(
            _mm256_castsi128_si256(_mm_loadu_si128(loaddr)), _mm_loadu_si128(hiaddr), 1);
}

// 定义内联函数 _mm256_storeu2_m128i，将 __m256i 类型的数据存储到两个 __m128i 类型的地址中
static inline void
__attribute__((__always_inline__))
  _mm256_storeu2_m128i(__m128i* const hiaddr, __m128i* const loaddr, const __m256i a)
{
    # 将__m128i类型的数据a的低128位存储到loaddr指向的内存地址中
    _mm_storeu_si128(loaddr, _mm256_castsi256_si128(a));
    # 将__m128i类型的数据a的高128位存储到hiaddr指向的内存地址中
    _mm_storeu_si128(hiaddr, _mm256_extracti128_si256(a, 1));
}
#endif
#else
#   include <intrin.h>
#endif


// This will shift and xor tmp1 into itself as 4 32-bit vals such as
// sl_xor(a1 a2 a3 a4) = a1 (a2^a1) (a3^a2^a1) (a4^a3^a2^a1)
static FORCEINLINE __m128i sl_xor(__m128i tmp1)
{
    // 将 tmp1 左移 4 个字节，并与自身进行异或操作
    __m128i tmp4;
    tmp4 = _mm_slli_si128(tmp1, 0x04);
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    tmp4 = _mm_slli_si128(tmp4, 0x04);
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    tmp4 = _mm_slli_si128(tmp4, 0x04);
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    return tmp1;
}


template<uint8_t rcon>
static FORCEINLINE void aes_genkey_sub(__m128i* xout0, __m128i* xout2)
{
    // 使用 rcon 生成 AES 密钥
    __m128i xout1 = _mm_aeskeygenassist_si128(*xout2, rcon);
    xout1 = _mm_shuffle_epi32(xout1, 0xFF); // see PSHUFD, set all elems to 4th elem
    *xout0 = sl_xor(*xout0);
    *xout0 = _mm_xor_si128(*xout0, xout1);
    xout1 = _mm_aeskeygenassist_si128(*xout0, 0x00);
    xout1 = _mm_shuffle_epi32(xout1, 0xAA); // see PSHUFD, set all elems to 3rd elem
    *xout2 = sl_xor(*xout2);
    *xout2 = _mm_xor_si128(*xout2, xout1);
}


static NOINLINE void vaes_genkey(const __m128i* memory, __m256i* k0, __m256i* k1, __m256i* k2, __m256i* k3, __m256i* k4, __m256i* k5, __m256i* k6, __m256i* k7, __m256i* k8, __m256i* k9)
{
    // 从内存加载数据到 xout0 和 xout2
    __m128i xout0 = _mm_load_si128(memory);
    __m128i xout2 = _mm_load_si128(memory + 1);
    *k0 = _mm256_set_m128i(xout0, xout0);
    *k1 = _mm256_set_m128i(xout2, xout2);

    // 生成 AES 密钥
    aes_genkey_sub<0x01>(&xout0, &xout2);
    *k2 = _mm256_set_m128i(xout0, xout0);
    *k3 = _mm256_set_m128i(xout2, xout2);

    aes_genkey_sub<0x02>(&xout0, &xout2);
    *k4 = _mm256_set_m128i(xout0, xout0);
    *k5 = _mm256_set_m128i(xout2, xout2);

    aes_genkey_sub<0x04>(&xout0, &xout2);
    *k6 = _mm256_set_m128i(xout0, xout0);
    *k7 = _mm256_set_m128i(xout2, xout2);

    aes_genkey_sub<0x08>(&xout0, &xout2);
    *k8 = _mm256_set_m128i(xout0, xout0);
    *k9 = _mm256_set_m128i(xout2, xout2);
}
// 生成双倍密钥
static NOINLINE void vaes_genkey_double(const __m128i* memory1, const __m128i* memory2, __m256i* k0, __m256i* k1, __m256i* k2, __m256i* k3, __m256i* k4, __m256i* k5, __m256i* k6, __m256i* k7, __m256i* k8, __m256i* k9)
{
    // 从内存中加载128位数据到寄存器
    __m128i xout0 = _mm_load_si128(memory1);
    __m128i xout1 = _mm_load_si128(memory1 + 1);
    __m128i xout2 = _mm_load_si128(memory2);
    __m128i xout3 = _mm_load_si128(memory2 + 1);
    // 将128位数据组合成256位数据
    *k0 = _mm256_set_m128i(xout2, xout0);
    *k1 = _mm256_set_m128i(xout3, xout1);

    // 使用AES算法生成子密钥
    aes_genkey_sub<0x01>(&xout0, &xout1);
    aes_genkey_sub<0x01>(&xout2, &xout3);
    *k2 = _mm256_set_m128i(xout2, xout0);
    *k3 = _mm256_set_m128i(xout3, xout1);

    aes_genkey_sub<0x02>(&xout0, &xout1);
    aes_genkey_sub<0x02>(&xout2, &xout3);
    *k4 = _mm256_set_m128i(xout2, xout0);
    *k5 = _mm256_set_m128i(xout3, xout1);

    aes_genkey_sub<0x04>(&xout0, &xout1);
    aes_genkey_sub<0x04>(&xout2, &xout3);
    *k6 = _mm256_set_m128i(xout2, xout0);
    *k7 = _mm256_set_m128i(xout3, xout1);

    aes_genkey_sub<0x08>(&xout0, &xout1);
    aes_genkey_sub<0x08>(&xout2, &xout3);
    *k8 = _mm256_set_m128i(xout2, xout0);
    *k9 = _mm256_set_m128i(xout3, xout1);
}

// 执行AES加密的一轮操作
static FORCEINLINE void vaes_round(__m256i key, __m256i& x01, __m256i& x23, __m256i& x45, __m256i& x67)
{
    // 使用256位密钥对四组256位数据进行AES加密
    x01 = _mm256_aesenc_epi128(x01, key);
    x23 = _mm256_aesenc_epi128(x23, key);
    x45 = _mm256_aesenc_epi128(x45, key);
    x67 = _mm256_aesenc_epi128(x67, key);
}

// 执行AES加密的一轮操作
static FORCEINLINE void vaes_round(__m256i key, __m256i& x0, __m256i& x1, __m256i& x2, __m256i& x3, __m256i& x4, __m256i& x5, __m256i& x6, __m256i& x7)
{
    // 使用256位密钥对八组256位数据进行AES加密
    x0 = _mm256_aesenc_epi128(x0, key);
    x1 = _mm256_aesenc_epi128(x1, key);
    x2 = _mm256_aesenc_epi128(x2, key);
    x3 = _mm256_aesenc_epi128(x3, key);
    x4 = _mm256_aesenc_epi128(x4, key);
    x5 = _mm256_aesenc_epi128(x5, key);
    x6 = _mm256_aesenc_epi128(x6, key);
    x7 = _mm256_aesenc_epi128(x7, key);
}

namespace xmrig {
// 使用 VAES 指令集对内存中的数据进行加密
NOINLINE void cn_explode_scratchpad_vaes(cryptonight_ctx* ctx, size_t memory, bool half_mem)
{
    // 计算需要处理的数据块数量
    const size_t N = (memory / sizeof(__m256i)) / (half_mem ? 2 : 1);

    // 定义输入数据和密钥
    __m256i xin01, xin23, xin45, xin67;
    __m256i k0, k1, k2, k3, k4, k5, k6, k7, k8, k9;

    // 将输入数据和输出数据转换为对应的数据类型
    const __m128i* input = reinterpret_cast<const __m128i*>(ctx->state);
    __m256i* output = reinterpret_cast<__m256i*>(ctx->memory);

    // 生成加密所需的密钥
    vaes_genkey(input, &k0, &k1, &k2, &k3, &k4, &k5, &k6, &k7, &k8, &k9);

    // 根据是否处理一半内存和是否为第一次处理，选择不同的输入数据
    if (half_mem && !ctx->first_half) {
        const __m256i* p = reinterpret_cast<const __m256i*>(ctx->save_state);
        xin01 = _mm256_loadu_si256(p + 0);
        xin23 = _mm256_loadu_si256(p + 1);
        xin45 = _mm256_loadu_si256(p + 2);
        xin67 = _mm256_loadu_si256(p + 3);
    }
    else {
        xin01 = _mm256_loadu_si256(reinterpret_cast<const __m256i*>(input + 4));
        xin23 = _mm256_loadu_si256(reinterpret_cast<const __m256i*>(input + 6));
        xin45 = _mm256_loadu_si256(reinterpret_cast<const __m256i*>(input + 8));
        xin67 = _mm256_loadu_si256(reinterpret_cast<const __m256i*>(input + 10));
    }

    // 定义输出数据的增量和预取距离
    constexpr int output_increment = 64 / sizeof(__m256i);
    constexpr int prefetch_dist = 2048 / sizeof(__m256i);

    // 定义预取指针和最后一个需要预取的数据块的指针
    __m256i* e = output + N - prefetch_dist;
    __m256i* prefetch_ptr = output + prefetch_dist;
}
    // 循环两次，每次执行以下操作
    for (int i = 0; i < 2; ++i) {
        // 预取内存中的数据，以加速后续计算
        _mm_prefetch((const char*)(prefetch_ptr), _MM_HINT_T0);
        _mm_prefetch((const char*)(prefetch_ptr + output_increment), _MM_HINT_T0);

        // 执行 AES 加密的轮函数，对输入数据进行加密
        vaes_round(k0, xin01, xin23, xin45, xin67);
        vaes_round(k1, xin01, xin23, xin45, xin67);
        vaes_round(k2, xin01, xin23, xin45, xin67);
        vaes_round(k3, xin01, xin23, xin45, xin67);
        vaes_round(k4, xin01, xin23, xin45, xin67);
        vaes_round(k5, xin01, xin23, xin45, xin67);
        vaes_round(k6, xin01, xin23, xin45, xin67);
        vaes_round(k7, xin01, xin23, xin45, xin67);
        vaes_round(k8, xin01, xin23, xin45, xin67);
        vaes_round(k9, xin01, xin23, xin45, xin67);

        // 将加密后的数据存储到输出数组中
        _mm256_store_si256(output + 0, xin01);
        _mm256_store_si256(output + 1, xin23);
        _mm256_store_si256(output + output_increment + 0, xin45);
        _mm256_store_si256(output + output_increment + 1, xin67);

        // 更新输出数组和预取指针的位置
        output += output_increment * 2;
        prefetch_ptr += output_increment * 2;
    } while (output < e);
    // 更新结束位置和预取指针的位置
    e += prefetch_dist;
    prefetch_ptr = output;
    }

    // 如果内存使用一半且为第一半，则执行以下操作
    if (half_mem && ctx->first_half) {
        // 将加密后的数据存储到上下文的保存状态中
        __m256i* p = reinterpret_cast<__m256i*>(ctx->save_state);
        _mm256_storeu_si256(p + 0, xin01);
        _mm256_storeu_si256(p + 1, xin23);
        _mm256_storeu_si256(p + 2, xin45);
        _mm256_storeu_si256(p + 3, xin67);
    }

    // 清除 AVX 寄存器状态
    _mm256_zeroupper();
NOINLINE void cn_explode_scratchpad_vaes_double(cryptonight_ctx* ctx1, cryptonight_ctx* ctx2, size_t memory, bool half_mem)
{
    // 计算需要处理的块数
    const size_t N = (memory / sizeof(__m128i)) / (half_mem ? 2 : 1);

    // 声明变量
    __m256i xin0, xin1, xin2, xin3, xin4, xin5, xin6, xin7;
    __m256i k0, k1, k2, k3, k4, k5, k6, k7, k8, k9;

    // 将输入状态转换为 __m128i 类型指针
    const __m128i* input1 = reinterpret_cast<const __m128i*>(ctx1->state);
    const __m128i* input2 = reinterpret_cast<const __m128i*>(ctx2->state);

    // 将输出内存转换为 __m128i 类型指针
    __m128i* output1 = reinterpret_cast<__m128i*>(ctx1->memory);
    __m128i* output2 = reinterpret_cast<__m128i*>(ctx2->memory);

    // 生成双倍密钥
    vaes_genkey_double(input1, input2, &k0, &k1, &k2, &k3, &k4, &k5, &k6, &k7, &k8, &k9);

    {
        // 计算是否需要使用保存的状态
        const bool b = half_mem && !ctx1->first_half && !ctx2->first_half;
        const __m128i* p1 = b ? reinterpret_cast<const __m128i*>(ctx1->save_state) : (input1 + 4);
        const __m128i* p2 = b ? reinterpret_cast<const __m128i*>(ctx2->save_state) : (input2 + 4);
        // 加载输入数据
        xin0 = _mm256_loadu2_m128i(p2 + 0, p1 + 0);
        xin1 = _mm256_loadu2_m128i(p2 + 1, p1 + 1);
        xin2 = _mm256_loadu2_m128i(p2 + 2, p1 + 2);
        xin3 = _mm256_loadu2_m128i(p2 + 3, p1 + 3);
        xin4 = _mm256_loadu2_m128i(p2 + 4, p1 + 4);
        xin5 = _mm256_loadu2_m128i(p2 + 5, p1 + 5);
        xin6 = _mm256_loadu2_m128i(p2 + 6, p1 + 6);
        xin7 = _mm256_loadu2_m128i(p2 + 7, p1 + 7);
    }

    // 定义常量
    constexpr int output_increment = 64 / sizeof(__m128i);
    constexpr int prefetch_dist = 2048 / sizeof(__m128i);

    // 计算预取数据的指针位置
    __m128i* e = output1 + N - prefetch_dist;
    __m128i* prefetch_ptr1 = output1 + prefetch_dist;
    __m128i* prefetch_ptr2 = output2 + prefetch_dist;

}
    # 如果满足条件：半内存，且上下文1和上下文2都是第一半
    if (half_mem && ctx1->first_half && ctx2->first_half) {
        # 将上下文1和上下文2的保存状态转换为__m128i类型指针
        __m128i* p1 = reinterpret_cast<__m128i*>(ctx1->save_state);
        __m128i* p2 = reinterpret_cast<__m128i*>(ctx2->save_state);
        # 将xin0存储到p2 + 0和p1 + 0指向的位置
        _mm256_storeu2_m128i(p2 + 0, p1 + 0, xin0);
        # 将xin1存储到p2 + 1和p1 + 1指向的位置
        _mm256_storeu2_m128i(p2 + 1, p1 + 1, xin1);
        # 将xin2存储到p2 + 2和p1 + 2指向的位置
        _mm256_storeu2_m128i(p2 + 2, p1 + 2, xin2);
        # 将xin3存储到p2 + 3和p1 + 3指向的位置
        _mm256_storeu2_m128i(p2 + 3, p1 + 3, xin3);
        # 将xin4存储到p2 + 4和p1 + 4指向的位置
        _mm256_storeu2_m128i(p2 + 4, p1 + 4, xin4);
        # 将xin5存储到p2 + 5和p1 + 5指向的位置
        _mm256_storeu2_m128i(p2 + 5, p1 + 5, xin5);
        # 将xin6存储到p2 + 6和p1 + 6指向的位置
        _mm256_storeu2_m128i(p2 + 6, p1 + 6, xin6);
        # 将xin7存储到p2 + 7和p1 + 7指向的位置
        _mm256_storeu2_m128i(p2 + 7, p1 + 7, xin7);
    }

    # 清除 AVX 寄存器的上半部分，以确保与 SSE 寄存器的兼容性
    _mm256_zeroupper();
# 定义一个名为 cn_implode_scratchpad_vaes 的函数，参数为指向 cryptonight_ctx 结构的指针 ctx、内存大小 memory 和是否使用一半内存 half_mem
NOINLINE void cn_implode_scratchpad_vaes(cryptonight_ctx* ctx, size_t memory, bool half_mem)
{
    # 计算每个 __m256i 类型的元素个数 N，用于确定内存大小
    const size_t N = (memory / sizeof(__m256i)) / (half_mem ? 2 : 1);

    # 定义 8 个 __m256i 类型的变量
    __m256i xout01, xout23, xout45, xout67;
    __m256i k0, k1, k2, k3, k4, k5, k6, k7, k8, k9;

    # 将 ctx->memory 强制转换为 __m256i* 类型的指针，并赋值给 input
    const __m256i* input = reinterpret_cast<const __m256i*>(ctx->memory);
    # 将 ctx->state 强制转换为 __m256i* 类型的指针，并赋值给 output
    __m256i* output = reinterpret_cast<__m256i*>(ctx->state);

    # 调用 vaes_genkey 函数生成密钥
    vaes_genkey(reinterpret_cast<__m128i*>(output) + 2, &k0, &k1, &k2, &k3, &k4, &k5, &k6, &k7, &k8, &k9);

    # 从 output + 2 处加载 256 位数据到 xout01
    xout01 = _mm256_loadu_si256(output + 2);
    # 从 output + 3 处加载 256 位数据到 xout23
    xout23 = _mm256_loadu_si256(output + 3);
    # 从 output + 4 处加载 256 位数据到 xout45
    xout45 = _mm256_loadu_si256(output + 4);
    # 从 output + 5 处加载 256 位数据到 xout67
    xout67 = _mm256_loadu_si256(output + 5);

    # 将 input 赋值给 input_begin
    const __m256i* input_begin = input;
    // 遍历两个部分的内存，如果内存只有一半，则遍历两次
    for (size_t part = 0; part < (half_mem ? 2 : 1); ++part) {
        // 如果内存只有一半，并且是第二部分，则重置输入指针，设置标志位，调用特定函数处理内存
        if (half_mem && (part == 1)) {
            input = input_begin;
            ctx->first_half = false;
            cn_explode_scratchpad_vaes(ctx, memory, half_mem);
        }

        // 遍历输入数据
        for (size_t i = 0; i < N;) {
            // 对输入数据进行异或操作
            xout01 = _mm256_xor_si256(xout01, input[0]);
            xout23 = _mm256_xor_si256(xout23, input[1]);

            // 定义输入增量
            constexpr int input_increment = 64 / sizeof(__m256i);

            // 对输入数据进行异或操作
            xout45 = _mm256_xor_si256(xout45, input[input_increment]);
            xout67 = _mm256_xor_si256(xout67, input[input_increment + 1]);

            // 更新输入指针和计数器
            input += input_increment * 2;
            i += 4;

            // 如果计数器小于 N，则进行预取数据
            if (i < N) {
                _mm_prefetch((const char*)(input), _MM_HINT_T0);
                _mm_prefetch((const char*)(input + input_increment), _MM_HINT_T0);
            }

            // 调用 AES 加密轮函数
            vaes_round(k0, xout01, xout23, xout45, xout67);
            vaes_round(k1, xout01, xout23, xout45, xout67);
            vaes_round(k2, xout01, xout23, xout45, xout67);
            vaes_round(k3, xout01, xout23, xout45, xout67);
            vaes_round(k4, xout01, xout23, xout45, xout67);
            vaes_round(k5, xout01, xout23, xout45, xout67);
            vaes_round(k6, xout01, xout23, xout45, xout67);
            vaes_round(k7, xout01, xout23, xout45, xout67);
            vaes_round(k8, xout01, xout23, xout45, xout67);
            vaes_round(k9, xout01, xout23, xout45, xout67);
        }
    }

    // 存储结果到输出数组
    _mm256_storeu_si256(output + 2, xout01);
    _mm256_storeu_si256(output + 3, xout23);
    _mm256_storeu_si256(output + 4, xout45);
    _mm256_storeu_si256(output + 5, xout67);

    // 清空 AVX 寄存器状态
    _mm256_zeroupper();
NOINLINE void cn_implode_scratchpad_vaes_double(cryptonight_ctx* ctx1, cryptonight_ctx* ctx2, size_t memory, bool half_mem)
{
    // 计算需要处理的数据块数量
    const size_t N = (memory / sizeof(__m128i)) / (half_mem ? 2 : 1);

    // 声明需要使用的变量
    __m256i xout0, xout1, xout2, xout3, xout4, xout5, xout6, xout7;
    __m256i k0, k1, k2, k3, k4, k5, k6, k7, k8, k9;

    // 将输入内存转换为 __m128i 类型指针
    const __m128i* input1 = reinterpret_cast<const __m128i*>(ctx1->memory);
    const __m128i* input2 = reinterpret_cast<const __m128i*>(ctx2->memory);

    // 将输出状态内存转换为 __m128i 类型指针
    __m128i* output1 = reinterpret_cast<__m128i*>(ctx1->state);
    __m128i* output2 = reinterpret_cast<__m128i*>(ctx2->state);

    // 生成双倍密钥
    vaes_genkey_double(output1 + 2, output2 + 2, &k0, &k1, &k2, &k3, &k4, &k5, &k6, &k7, &k8, &k9);

    // 加载数据到临时变量
    xout0 = _mm256_loadu2_m128i(output2 + 4, output1 + 4);
    xout1 = _mm256_loadu2_m128i(output2 + 5, output1 + 5);
    xout2 = _mm256_loadu2_m128i(output2 + 6, output1 + 6);
    xout3 = _mm256_loadu2_m128i(output2 + 7, output1 + 7);
    xout4 = _mm256_loadu2_m128i(output2 + 8, output1 + 8);
    xout5 = _mm256_loadu2_m128i(output2 + 9, output1 + 9);
    xout6 = _mm256_loadu2_m128i(output2 + 10, output1 + 10);
    xout7 = _mm256_loadu2_m128i(output2 + 11, output1 + 11);

    // 保存临时变量到输出状态内存
    _mm256_storeu2_m128i(output2 + 4, output1 + 4, xout0);
    _mm256_storeu2_m128i(output2 + 5, output1 + 5, xout1);
    _mm256_storeu2_m128i(output2 + 6, output1 + 6, xout2);
    _mm256_storeu2_m128i(output2 + 7, output1 + 7, xout3);
    _mm256_storeu2_m128i(output2 + 8, output1 + 8, xout4);
    _mm256_storeu2_m128i(output2 + 9, output1 + 9, xout5);
    _mm256_storeu2_m128i(output2 + 10, output1 + 10, xout6);
    _mm256_storeu2_m128i(output2 + 11, output1 + 11, xout7);

    // 清空 AVX 寄存器状态
    _mm256_zeroupper();
}
```