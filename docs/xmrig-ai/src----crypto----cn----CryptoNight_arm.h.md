# `xmrig\src\crypto\cn\CryptoNight_arm.h`

```cpp
/* XMRig
 * 版权所有 2010      Jeff Garzik  <jgarzik@pobox.com>
 * 版权所有 2012-2014 pooler       <pooler@litecoinpool.org>
 * 版权所有 2014      Lucas Jones  <https://github.com/lucasjones>
 * 版权所有 2014-2016 Wolf9466     <https://github.com/OhGodAPet>
 * 版权所有 2016      Jay D Dee    <jayddee246@gmail.com>
 * 版权所有 2016      Imran Yusuff <https://github.com/imranyusuff>
 * 版权所有 2017-2019 XMR-Stak    <https://github.com/fireice-uk>, <https://github.com/psychocrypt>
 * 版权所有 2018      Lee Clagett <https://github.com/vtnerd>
 * 版权所有 2018-2020 SChernykh   <https://github.com/SChernykh>
 * 版权所有 2016-2020 XMRig       <https://github.com/xmrig>, <support@xmrig.com>
 *
 *   本程序是自由软件：您可以重新发布或修改
 *   根据由自由软件基金会发布的 GNU 通用公共许可证的条款，
 *   版本 3 或（按您的选择）更高版本。
 *
 *   本程序是希望它有用的分发，
 *   但没有任何保证；甚至没有暗示的保证适用于特定用途。有关更多详细信息，请参见
 *   GNU 通用公共许可证。
 *
 *   您应该已经收到了 GNU 通用公共许可证的副本
 *   与本程序一起。如果没有，请参见 <http://www.gnu.org/licenses/>。
 */

#ifndef XMRIG_CRYPTONIGHT_ARM_H
#define XMRIG_CRYPTONIGHT_ARM_H


#include "base/crypto/keccak.h"
#include "crypto/cn/CnAlgo.h"
#include "crypto/cn/CryptoNight_monero.h"
#include "crypto/cn/CryptoNight.h"
#include "crypto/cn/soft_aes.h"


extern "C"
{
#include "crypto/cn/c_groestl.h"
#include "crypto/cn/c_blake256.h"
#include "crypto/cn/c_jh.h"
#include "crypto/cn/c_skein.h"
}


static inline void do_blake_hash(const uint8_t *input, size_t len, uint8_t *output) {
    // 执行 blake256 哈希算法
    blake256_hash(output, input, len);
}


static inline void do_groestl_hash(const uint8_t *input, size_t len, uint8_t *output) {
    # 使用 Groestl 算法对输入进行哈希计算，将结果存储到输出中
    groestl(input, len * 8, output);
// 定义一个内联函数，用于执行 JH 哈希算法
static inline void do_jh_hash(const uint8_t *input, size_t len, uint8_t *output) {
    jh_hash(32 * 8, input, 8 * len, output);
}

// 定义一个内联函数，用于执行 Skein 哈希算法
static inline void do_skein_hash(const uint8_t *input, size_t len, uint8_t *output) {
    xmr_skein(input, output);
}

// 定义一个指针数组，包含四个哈希函数的指针
void (* const extra_hashes[4])(const uint8_t *, size_t, uint8_t *) = {do_blake_hash, do_groestl_hash, do_jh_hash, do_skein_hash};

// 定义一个内联函数，用于执行 sl_xor 操作
// sl_xor(a1 a2 a3 a4) = a1 (a2^a1) (a3^a2^a1) (a4^a3^a2^a1)
static inline __m128i sl_xor(__m128i tmp1)
{
    // 左移 4 个字节
    __m128i tmp4;
    tmp4 = _mm_slli_si128(tmp1, 0x04);
    // 对 tmp1 和 tmp4 进行异或操作
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    tmp4 = _mm_slli_si128(tmp4, 0x04);
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    tmp4 = _mm_slli_si128(tmp4, 0x04);
    tmp1 = _mm_xor_si128(tmp1, tmp4);
    return tmp1;
}

// 定义一个模板函数，用于生成 AES 密钥的子函数
template<uint8_t rcon>
static inline void soft_aes_genkey_sub(__m128i* xout0, __m128i* xout2)
{
    // 调用 soft_aeskeygenassist 函数生成子密钥
    __m128i xout1 = soft_aeskeygenassist<rcon>(*xout2);
    // 将 xout1 的元素顺序调整为 4th elem
    xout1  = _mm_shuffle_epi32(xout1, 0xFF);
    // 对 xout0 进行 sl_xor 操作
    *xout0 = sl_xor(*xout0);
    // 对 xout0 和 xout1 进行异或操作
    *xout0 = _mm_xor_si128(*xout0, xout1);
    // 调用 soft_aeskeygenassist 函数生成子密钥
    xout1  = soft_aeskeygenassist<0x00>(*xout0);
    // 将 xout1 的元素顺序调整为 3rd elem
    xout1  = _mm_shuffle_epi32(xout1, 0xAA);
    // 对 xout2 进行 sl_xor 操作
    *xout2 = sl_xor(*xout2);
    // 对 xout2 和 xout1 进行异或操作
    *xout2 = _mm_xor_si128(*xout2, xout1);
}

// 定义一个模板函数，用于生成 AES 密钥
template<bool SOFT_AES>
static inline void aes_genkey(const __m128i* memory, __m128i* k0, __m128i* k1, __m128i* k2, __m128i* k3, __m128i* k4, __m128i* k5, __m128i* k6, __m128i* k7, __m128i* k8, __m128i* k9)
{
    // 从内存中加载数据到 xout0 和 xout2
    __m128i xout0 = _mm_load_si128(memory);
    __m128i xout2 = _mm_load_si128(memory + 1);
    *k0 = xout0;
    *k1 = xout2;

    // 调用 soft_aes_genkey_sub 函数生成子密钥
    soft_aes_genkey_sub<0x01>(&xout0, &xout2);
    *k2 = xout0;
    *k3 = xout2;

    // 调用 soft_aes_genkey_sub 函数生成子密钥
    soft_aes_genkey_sub<0x02>(&xout0, &xout2);
    *k4 = xout0;
    *k5 = xout2;

    // 调用 soft_aes_genkey_sub 函数生成子密钥
    soft_aes_genkey_sub<0x04>(&xout0, &xout2);
    *k6 = xout0;
    *k7 = xout2;

    // 调用 soft_aes_genkey_sub 函数生成子密钥
    soft_aes_genkey_sub<0x08>(&xout0, &xout2);
    *k8 = xout0;
}
    # 将变量 xout2 的值赋给变量 k9
    // 根据 SOFT_AES 的值选择使用软件实现的 AES 还是硬件指令集实现的 AES 进行轮加密
    if (SOFT_AES) {
        // 如果使用软件实现的 AES，则调用软件实现的 aesenc 函数进行轮加密
        *x0 = soft_aesenc((uint32_t*)x0, key);
        *x1 = soft_aesenc((uint32_t*)x1, key);
        *x2 = soft_aesenc((uint32_t*)x2, key);
        *x3 = soft_aesenc((uint32_t*)x3, key);
        *x4 = soft_aesenc((uint32_t*)x4, key);
        *x5 = soft_aesenc((uint32_t*)x5, key);
        *x6 = soft_aesenc((uint32_t*)x6, key);
        *x7 = soft_aesenc((uint32_t*)x7, key);
    }
    else {
        // 如果使用硬件指令集实现的 AES，则调用硬件指令集的 _mm_aesenc_si128 函数进行轮加密
        *x0 = _mm_aesenc_si128(*x0, key);
        *x1 = _mm_aesenc_si128(*x1, key);
        *x2 = _mm_aesenc_si128(*x2, key);
        *x3 = _mm_aesenc_si128(*x3, key);
        *x4 = _mm_aesenc_si128(*x4, key);
        *x5 = _mm_aesenc_si128(*x5, key);
        *x6 = _mm_aesenc_si128(*x6, key);
        *x7 = _mm_aesenc_si128(*x7, key);
    }
}

// 对输入的 8 个 __m128i 类型的数据进行混合和传播操作
inline void mix_and_propagate(__m128i& x0, __m128i& x1, __m128i& x2, __m128i& x3, __m128i& x4, __m128i& x5, __m128i& x6, __m128i& x7)
{
    // 临时保存 x0 的值
    __m128i tmp0 = x0;
    // 对 x0 到 x7 进行异或操作，实现混合和传播
    x0 = _mm_xor_si128(x0, x1);
    x1 = _mm_xor_si128(x1, x2);
    x2 = _mm_xor_si128(x2, x3);
    x3 = _mm_xor_si128(x3, x4);
    x4 = _mm_xor_si128(x4, x5);
    x5 = _mm_xor_si128(x5, x6);
    x6 = _mm_xor_si128(x6, x7);
    x7 = _mm_xor_si128(x7, tmp0);
}

// 命名空间 xmrig 中的模板函数，根据 ALGO 和 SOFT_AES 的值对输入的 __m128i 数组进行处理
template<Algorithm::Id ALGO, bool SOFT_AES>
static inline void cn_explode_scratchpad(const __m128i *input, __m128i *output)
{
    // 根据 ALGO 的值选择对应的算法属性
    constexpr CnAlgo<ALGO> props;

    // 定义 10 个 __m128i 类型的变量
    __m128i xin0, xin1, xin2, xin3, xin4, xin5, xin6, xin7;
    __m128i k0, k1, k2, k3, k4, k5, k6, k7, k8, k9;

    // 生成 AES 密钥
    aes_genkey<SOFT_AES>(input, &k0, &k1, &k2, &k3, &k4, &k5, &k6, &k7, &k8, &k9);

    // 从输入数组中加载数据到 xin0 到 xin5 中
    xin0 = _mm_load_si128(input + 4);
    xin1 = _mm_load_si128(input + 5);
    xin2 = _mm_load_si128(input + 6);
    xin3 = _mm_load_si128(input + 7);
    xin4 = _mm_load_si128(input + 8);
    xin5 = _mm_load_si128(input + 9);
    # 从输入数组中加载第10个元素到寄存器xin6
    xin6 = _mm_load_si128(input + 10);
    # 从输入数组中加载第11个元素到寄存器xin7
    xin7 = _mm_load_si128(input + 11);

    # 如果属性为重型
    if (props.isHeavy()) {
        # 对输入数据进行16轮AES加密
        for (size_t i = 0; i < 16; i++) {
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

            # 对输入数据进行混合和传播操作
            mix_and_propagate(xin0, xin1, xin2, xin3, xin4, xin5, xin6, xin7);
        }
    }
    # 使用循环对输入数据进行 AES 加密，每次处理 8 个 __m128i 类型的数据
    for (size_t i = 0; i < props.memory() / sizeof(__m128i); i += 8) {
        # 对输入数据进行 AES 加密轮操作，使用 k0 到 k9 作为轮密钥
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

        # 将加密后的数据存储到输出数组中
        _mm_store_si128(output + i + 0, xin0);
        _mm_store_si128(output + i + 1, xin1);
        _mm_store_si128(output + i + 2, xin2);
        _mm_store_si128(output + i + 3, xin3);
        _mm_store_si128(output + i + 4, xin4);
        _mm_store_si128(output + i + 5, xin5);
        _mm_store_si128(output + i + 6, xin6);
        _mm_store_si128(output + i + 7, xin7);
    }
# 定义一个静态内联函数，用于压缩内存中的数据块
template<Algorithm::Id ALGO, bool SOFT_AES>
static inline void cn_implode_scratchpad(const __m128i *input, __m128i *output)
{
    # 根据算法类型获取算法属性
    constexpr CnAlgo<ALGO> props;

    # 判断是否为重型算法
    constexpr bool IS_HEAVY = props.isHeavy();

    # 定义8个128位的寄存器变量
    __m128i xout0, xout1, xout2, xout3, xout4, xout5, xout6, xout7;
    # 定义10个128位的寄存器变量
    __m128i k0, k1, k2, k3, k4, k5, k6, k7, k8, k9;

    # 生成AES密钥
    aes_genkey<SOFT_AES>(output + 2, &k0, &k1, &k2, &k3, &k4, &k5, &k6, &k7, &k8, &k9);

    # 从output数组中加载128位数据到xout0-xout7寄存器变量中
    xout0 = _mm_load_si128(output + 4);
    xout1 = _mm_load_si128(output + 5);
    xout2 = _mm_load_si128(output + 6);
    xout3 = _mm_load_si128(output + 7);
    xout4 = _mm_load_si128(output + 8);
    xout5 = _mm_load_si128(output + 9);
    xout6 = _mm_load_si128(output + 10);
    xout7 = _mm_load_si128(output + 11);
    // 使用循环对输入数据进行处理，每次处理8个__m128i大小的数据
    for (size_t i = 0; i < props.memory() / sizeof(__m128i); i += 8) {
        // 对输入数据进行异或操作，并将结果存储到xout0 - xout7中
        xout0 = _mm_xor_si128(_mm_load_si128(input + i + 0), xout0);
        xout1 = _mm_xor_si128(_mm_load_si128(input + i + 1), xout1);
        xout2 = _mm_xor_si128(_mm_load_si128(input + i + 2), xout2);
        xout3 = _mm_xor_si128(_mm_load_si128(input + i + 3), xout3);
        xout4 = _mm_xor_si128(_mm_load_si128(input + i + 4), xout4);
        xout5 = _mm_xor_si128(_mm_load_si128(input + i + 5), xout5);
        xout6 = _mm_xor_si128(_mm_load_si128(input + i + 6), xout6);
        xout7 = _mm_xor_si128(_mm_load_si128(input + i + 7), xout7);

        // 对xout0 - xout7进行AES加密轮操作
        aes_round<SOFT_AES>(k0, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k1, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k2, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k3, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k4, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k5, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k6, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k7, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k8, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);
        aes_round<SOFT_AES>(k9, &xout0, &xout1, &xout2, &xout3, &xout4, &xout5, &xout6, &xout7);

        // 如果是重型操作，对xout0 - xout7进行混合和传播操作
        if (IS_HEAVY) {
            mix_and_propagate(xout0, xout1, xout2, xout3, xout4, xout5, xout6, xout7);
        }
    }

    // 将xout0 - xout5的值存储到output数组中
    _mm_store_si128(output + 4, xout0);
    _mm_store_si128(output + 5, xout1);
    _mm_store_si128(output + 6, xout2);
    _mm_store_si128(output + 7, xout3);
    _mm_store_si128(output + 8, xout4);
    _mm_store_si128(output + 9, xout5);
    # 将 xout6 中的数据存储到 output 数组的第 10 个位置
    _mm_store_si128(output + 10, xout6);
    # 将 xout7 中的数据存储到 output 数组的第 11 个位置
    _mm_store_si128(output + 11, xout7);
} /* namespace xmrig */

} /* namespace xmrig */

static inline __m128i aes_round_tweak_div(const __m128i &in, const __m128i &key)
{
    // 用于存储对齐的 4 个 32 位整数的数组
    alignas(16) uint32_t k[4];
    alignas(16) uint32_t x[4];

    // 将 key 存储到 k 数组中
    _mm_store_si128((__m128i*) k, key);
    // 将 in 异或 0xffffffffffffffff, 0xffffffffffffffff 存储到 x 数组中
    _mm_store_si128((__m128i*) x, _mm_xor_si128(in, _mm_set_epi64x(0xffffffffffffffff, 0xffffffffffffffff)));

    // 定义宏 BYTE 用于获取 x 数组中指定位置的字节
    #define BYTE(p, i) ((unsigned char*)&x[p])[i]
    // 对 k 数组进行一系列异或操作
    k[0] ^= saes_table[0][BYTE(0, 0)] ^ saes_table[1][BYTE(1, 1)] ^ saes_table[2][BYTE(2, 2)] ^ saes_table[3][BYTE(3, 3)];
    x[0] ^= k[0];
    k[1] ^= saes_table[0][BYTE(1, 0)] ^ saes_table[1][BYTE(2, 1)] ^ saes_table[2][BYTE(3, 2)] ^ saes_table[3][BYTE(0, 3)];
    x[1] ^= k[1];
    k[2] ^= saes_table[0][BYTE(2, 0)] ^ saes_table[1][BYTE(3, 1)] ^ saes_table[2][BYTE(0, 2)] ^ saes_table[3][BYTE(1, 3)];
    x[2] ^= k[2];
    k[3] ^= saes_table[0][BYTE(3, 0)] ^ saes_table[1][BYTE(0, 1)] ^ saes_table[2][BYTE(1, 2)] ^ saes_table[3][BYTE(2, 3)];
    // 取消宏定义 BYTE
    #undef BYTE

    // 返回 k 数组
    return _mm_load_si128((__m128i*)k);
}


namespace xmrig {

// 模板函数，根据不同的 ALGO 参数进行不同的操作
template<Algorithm::Id ALGO>
static inline void cryptonight_monero_tweak(const uint8_t* l, uint64_t idx, __m128i ax0, __m128i bx0, __m128i bx1, __m128i& cx)
{
    // 获取算法属性
    constexpr CnAlgo<ALGO> props;

    // 将 l 数组中指定位置的值转换为 uint64_t 指针
    uint64_t* mem_out = (uint64_t*)&l[idx];

    // 如果算法基于 CN_2
    if (props.base() == Algorithm::CN_2) {
        // 根据不同的条件进行数据重排和存储
        VARIANT2_SHUFFLE(l, idx, ax0, bx0, bx1, cx, (((ALGO == Algorithm::CN_RWZ) || (ALGO == Algorithm::CN_UPX2)) ? 1 : 0));
        _mm_store_si128((__m128i *)mem_out, _mm_xor_si128(bx0, cx));
    } else {
        // 对 bx0 和 cx 进行异或操作，并将结果存储到 mem_out 数组中
        __m128i tmp = _mm_xor_si128(bx0, cx);
        mem_out[0] = _mm_cvtsi128_si64(tmp);

        // 获取 tmp 中的高位值
        uint64_t vh = vgetq_lane_u64(tmp, 1);

        // 对 vh 进行一系列位运算，并将结果存储到 mem_out 数组中
        mem_out[1] = vh ^ tweak1_table[static_cast<uint8_t>(vh >> 24)];
    }
}

// 对 cx 进行一系列浮点数运算
static inline void cryptonight_conceal_tweak(__m128i& cx, __m128& conc_var)
{
    // 对 cx 进行一系列浮点数运算，并将结果存储到 r 中
    __m128 r = _mm_add_ps(_mm_cvtepi32_ps(cx), conc_var);
    r = _mm_mul_ps(r, _mm_mul_ps(r, r));
    r = _mm_and_ps(_mm_castsi128_ps(_mm_set1_epi32(0x807FFFFF)), r);
}
    # 将参数r与0x40000000进行按位或操作，并将结果存储在r中
    r = _mm_or_ps(_mm_castsi128_ps(_mm_set1_epi32(0x40000000)), r);

    # 将conc_var的值存储在c_old中
    __m128 c_old = conc_var;
    # 将conc_var与r进行按位加法操作，并将结果存储在conc_var中
    conc_var = _mm_add_ps(conc_var, r);

    # 将c_old与0x807FFFFF进行按位与操作，并将结果存储在c_old中
    c_old = _mm_and_ps(_mm_castsi128_ps(_mm_set1_epi32(0x807FFFFF)), c_old);
    # 将c_old与0x40000000进行按位或操作，并将结果存储在c_old中
    c_old = _mm_or_ps(_mm_castsi128_ps(_mm_set1_epi32(0x40000000)), c_old);

    # 将c_old与536870880.0f进行按位乘法操作，并将结果存储在nc中
    __m128 nc = _mm_mul_ps(c_old, _mm_set1_ps(536870880.0f));
    # 将cx与nc进行按位异或操作，并将结果存储在cx中
    cx = _mm_xor_si128(cx, _mm_cvttps_epi32(nc));
#   endif
template<Algorithm::Id ALGO, bool SOFT_AES, int interleave>
inline void cryptonight_single_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
    // 根据模板参数ALGO获取算法属性
    constexpr CnAlgo<ALGO> props;
    // 获取MASK值
    constexpr size_t MASK        = props.mask();
    // 获取基础算法ID
    constexpr Algorithm::Id BASE = props.base();

#   ifdef XMRIG_ALGO_CN_HEAVY
    // 如果定义了XMRIG_ALGO_CN_HEAVY，则判断ALGO是否为CN_HEAVY_TUBE
    constexpr bool IS_CN_HEAVY_TUBE = ALGO == Algorithm::CN_HEAVY_TUBE;
#   else
    // 否则将IS_CN_HEAVY_TUBE设置为false
    constexpr bool IS_CN_HEAVY_TUBE = false;
#   endif

    // 如果基础算法为CN_1且输入大小小于43，则将输出内存清零并返回
    if (BASE == Algorithm::CN_1 && size < 43) {
        memset(output, 0, 32);
        return;
    }

    // 对输入数据进行Keccak哈希计算
    keccak(input, size, ctx[0]->state);
    // 根据算法和SOFT_AES参数展开内存
    cn_explode_scratchpad<ALGO, SOFT_AES>(reinterpret_cast<const __m128i *>(ctx[0]->state), reinterpret_cast<__m128i *>(ctx[0]->memory));

    // 获取内存指针和状态指针
    uint8_t* l0 = ctx[0]->memory;
    uint64_t* h0 = reinterpret_cast<uint64_t*>(ctx[0]->state);

    // 初始化变体1
    VARIANT1_INIT(0);
    // 初始化变体2
    VARIANT2_INIT(0);
    // 初始化随机数运算
    VARIANT4_RANDOM_MATH_INIT(0);

    // 计算al0和ah0
    uint64_t al0 = h0[0] ^ h0[4];
    uint64_t ah0 = h0[1] ^ h0[5];
    // 创建__m128i类型的变量bx0和bx1
    __m128i bx0  = _mm_set_epi64x(h0[3] ^ h0[7], h0[2] ^ h0[6]);
    __m128i bx1  = _mm_set_epi64x(h0[9] ^ h0[11], h0[8] ^ h0[10]);

    // 如果ALGO为CN_CCX，则初始化conc_var为0
    __m128 conc_var;
    if (ALGO == Algorithm::CN_CCX) {
        conc_var = _mm_setzero_ps();
    }

    // 初始化idx0为al0
    uint64_t idx0 = al0;

#       ifdef XMRIG_ALGO_CN_HEAVY
        // 如果算法属性为重型算法，则执行以下代码
        if (props.isHeavy()) {
            // 从内存中加载数据并进行一系列计算
            const int64x2_t x = vld1q_s64(reinterpret_cast<const int64_t *>(&l0[idx0 & MASK]));
            const int64_t n   = vgetq_lane_s64(x, 0);
            const int32_t d   = vgetq_lane_s32(x, 2);
            const int64_t q   = n / (d | 0x5);

            // 将计算结果写回内存
            ((int64_t*)&l0[idx0 & MASK])[0] = n ^ q;

            // 如果ALGO为CN_HEAVY_XHV，则更新idx0
            if (ALGO == Algorithm::CN_HEAVY_XHV) {
                idx0 = (~d) ^ q;
            }
            else {
                idx0 = d ^ q;
            }
        }
#       endif

        // 如果基础算法为CN_2，则将bx1设置为bx0
        if (BASE == Algorithm::CN_2) {
            bx1 = bx0;
        }

        // 将bx0设置为cx
        bx0 = cx;
    }
    # 使用指定的算法和软件AES对内存中的数据进行压缩
    cn_implode_scratchpad<ALGO, SOFT_AES>(reinterpret_cast<const __m128i *>(ctx[0]->memory), reinterpret_cast<__m128i *>(ctx[0]->state));
    # 对h0进行24轮的Keccak压缩函数处理
    keccakf(h0, 24);
    # 使用ctx[0]->state[0]的低2位作为索引，调用对应的额外哈希函数，将结果存储在output中
    extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
# 定义一个内联函数，用于执行Cryptonight双哈希操作
template<Algorithm::Id ALGO, bool SOFT_AES>
inline void cryptonight_double_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, struct cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
    # 根据算法ID获取算法属性
    constexpr CnAlgo<ALGO> props;
    # 获取掩码大小
    constexpr size_t MASK        = props.mask();
    # 获取基础算法ID
    constexpr Algorithm::Id BASE = props.base();

#   ifdef XMRIG_ALGO_CN_HEAVY
    # 如果定义了XMRIG_ALGO_CN_HEAVY，则判断当前算法是否为CN_HEAVY_TUBE
    constexpr bool IS_CN_HEAVY_TUBE = ALGO == Algorithm::CN_HEAVY_TUBE;
#   else
    # 否则将IS_CN_HEAVY_TUBE设置为false
    constexpr bool IS_CN_HEAVY_TUBE = false;
#   endif

    # 如果基础算法为CN_1且输入大小小于43，则将输出内存清零并返回
    if (BASE == Algorithm::CN_1 && size < 43) {
        memset(output, 0, 64);
        return;
    }

    # 对输入数据进行Keccak哈希计算，分别存储到ctx[0]和ctx[1]的状态中
    keccak(input,        size, ctx[0]->state);
    keccak(input + size, size, ctx[1]->state);

    # 获取ctx[0]和ctx[1]的内存指针和状态指针
    uint8_t *l0  = ctx[0]->memory;
    uint8_t *l1  = ctx[1]->memory;
    uint64_t *h0 = reinterpret_cast<uint64_t*>(ctx[0]->state);
    uint64_t *h1 = reinterpret_cast<uint64_t*>(ctx[1]->state);

    # 初始化变体1和变体2
    VARIANT1_INIT(0);
    VARIANT1_INIT(1);
    VARIANT2_INIT(0);
    VARIANT2_INIT(1);
    # 初始化变体4的随机数运算
    VARIANT4_RANDOM_MATH_INIT(0);
    VARIANT4_RANDOM_MATH_INIT(1);

    # 将状态指针h0和h1转换为__m128i类型，并将其解压到内存l0和l1中
    cn_explode_scratchpad<ALGO, SOFT_AES>(reinterpret_cast<const __m128i *>(h0), reinterpret_cast<__m128i *>(l0));
    cn_explode_scratchpad<ALGO, SOFT_AES>(reinterpret_cast<const __m128i *>(h1), reinterpret_cast<__m128i *>(l1));

    # 计算al0、al1、ah0、ah1
    uint64_t al0 = h0[0] ^ h0[4];
    uint64_t al1 = h1[0] ^ h1[4];
    uint64_t ah0 = h0[1] ^ h0[5];
    uint64_t ah1 = h1[1] ^ h1[5];

    # 将h0和h1的部分数据进行异或操作，存储到__m128i类型的变量bx00、bx01、bx10、bx11中
    __m128i bx00 = _mm_set_epi64x(h0[3] ^ h0[7], h0[2] ^ h0[6]);
    __m128i bx01 = _mm_set_epi64x(h0[9] ^ h0[11], h0[8] ^ h0[10]);
    __m128i bx10 = _mm_set_epi64x(h1[3] ^ h1[7], h1[2] ^ h1[6]);
    __m128i bx11 = _mm_set_epi64x(h1[9] ^ h1[11], h1[8] ^ h1[10]);

    # 如果算法为CN_CCX，则初始化conc_var0和conc_var1
    __m128 conc_var0, conc_var1;
    if (ALGO == Algorithm::CN_CCX) {
        conc_var0 = _mm_setzero_ps();
        conc_var1 = _mm_setzero_ps();
    }

    # 计算idx0和idx1
    uint64_t idx0 = al0;
    uint64_t idx1 = al1;
# 如果定义了 XMRIG_ALGO_CN_HEAVY，则执行以下代码块
        if (props.isHeavy()) {
            # 从 l0 数组中读取指定索引处的值，转换成 int64x2_t 类型
            const int64x2_t x = vld1q_s64(reinterpret_cast<const int64_t *>(&l0[idx0 & MASK]));
            # 从 x 中获取第一个 int64_t 类型的值
            const int64_t n   = vgetq_lane_s64(x, 0);
            # 从 x 中获取第三个 int32_t 类型的值
            const int32_t d   = vgetq_lane_s32(x, 2);
            # 计算 n 除以 (d | 0x5) 的结果
            const int64_t q   = n / (d | 0x5);

            # 将 n 与 q 的异或结果存入 l0 数组指定索引处
            ((int64_t*)&l0[idx0 & MASK])[0] = n ^ q;

            # 如果 ALGO 为 Algorithm::CN_HEAVY_XHV，则执行以下代码块
            if (ALGO == Algorithm::CN_HEAVY_XHV) {
                # 对 idx0 进行位取反并与 q 进行异或操作
                idx0 = (~d) ^ q;
            }
            # 如果 ALGO 不为 Algorithm::CN_HEAVY_XHV，则执行以下代码块
            else {
                # 对 idx0 进行位与 q 进行异或操作
                idx0 = d ^ q;
            }
        }
#       endif

        # 从 l1 数组中读取指定索引处的值，转换成 uint64_t 类型
        cl = ((uint64_t*) &l1[idx1 & MASK])[0];
        # 从 l1 数组中读取指定索引处的值，转换成 uint64_t 类型
        ch = ((uint64_t*) &l1[idx1 & MASK])[1];

        # 如果 BASE 为 Algorithm::CN_2，则执行以下代码块
        if (BASE == Algorithm::CN_2) {
            # 如果 props 的 R 属性为真，则执行以下代码块
            if (props.isR()) {
                # 执行 VARIANT4_RANDOM_MATH 函数
                VARIANT4_RANDOM_MATH(1, al1, ah1, cl, bx10, bx11);
                # 如果 ALGO 为 Algorithm::CN_R，则执行以下代码块
                if (ALGO == Algorithm::CN_R) {
                    # 对 al1 和 ah1 分别进行异或操作
                    al1 ^= r1[2] | ((uint64_t)(r1[3]) << 32);
                    ah1 ^= r1[0] | ((uint64_t)(r1[1]) << 32);
                }
            } else {
                # 执行 VARIANT2_INTEGER_MATH 函数
                VARIANT2_INTEGER_MATH(1, cl, cx1);
            }
        }

        # 使用 idx1 与 cl 的乘积更新 lo，将结果存入 hi
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

        # 将 hi 加到 al1，将 lo 加到 ah1
        al1 += hi;
        ah1 += lo;

        # 将 al1 存入 l1 数组指定索引处
        ((uint64_t*)&l1[idx1 & MASK])[0] = al1;

        # 如果 IS_CN_HEAVY_TUBE 为真，或者 ALGO 为 Algorithm::CN_RTO，则执行以下代码块
        if (IS_CN_HEAVY_TUBE || ALGO == Algorithm::CN_RTO) {
            # 将 ah1 与 tweak1_2_1 异或后的结果存入 l1 数组指定索引处
            ((uint64_t*)&l1[idx1 & MASK])[1] = ah1 ^ tweak1_2_1 ^ al1;
        } else if (BASE == Algorithm::CN_1) {
            # 将 ah1 与 tweak1_2_1 异或后的结果存入 l1 数组指定索引处
            ((uint64_t*)&l1[idx1 & MASK])[1] = ah1 ^ tweak1_2_1;
        } else {
            # 将 ah1 存入 l1 数组指定索引处
            ((uint64_t*)&l1[idx1 & MASK])[1] = ah1;
        }

        # 对 al1 和 cl 进行异或操作
        al1 ^= cl;
        # 对 ah1 和 ch 进行异或操作
        ah1 ^= ch;
        # 将 al1 存入 idx1
        idx1 = al1;
// 如果定义了 XMRIG_ALGO_CN_HEAVY，则执行以下代码块
#ifdef XMRIG_ALGO_CN_HEAVY
    // 如果属性表明是重型算法
    if (props.isHeavy()) {
        // 从 l1 数组中加载一个 int64x2_t 类型的值
        const int64x2_t x = vld1q_s64(reinterpret_cast<const int64_t *>(&l1[idx1 & MASK]));
        // 从 x 中获取第一个 int64_t 类型的值
        const int64_t n   = vgetq_lane_s64(x, 0);
        // 从 x 中获取第三个 int32_t 类型的值
        const int32_t d   = vgetq_lane_s32(x, 2);
        // 计算 n 除以 (d 或 0x5) 的结果
        const int64_t q   = n / (d | 0x5);

        // 将 n 与 q 的异或结果存入 l1 数组
        ((int64_t*)&l1[idx1 & MASK])[0] = n ^ q;

        // 如果算法是 CN_HEAVY_XHV，则执行以下代码块
        if (ALGO == Algorithm::CN_HEAVY_XHV) {
            // 更新 idx1 的值
            idx1 = (~d) ^ q;
        }
        // 否则执行以下代码块
        else {
            // 更新 idx1 的值
            idx1 = d ^ q;
        }
    }
#endif

// 如果算法是 CN_2，则执行以下代码块
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

// 调用 cn_implode_scratchpad 函数处理 l0 数组，并将结果存入 h0 数组
cn_implode_scratchpad<ALGO, SOFT_AES>(reinterpret_cast<const __m128i *>(l0), reinterpret_cast<__m128i *>(h0));
// 调用 cn_implode_scratchpad 函数处理 l1 数组，并将结果存入 h1 数组
cn_implode_scratchpad<ALGO, SOFT_AES>(reinterpret_cast<const __m128i *>(l1), reinterpret_cast<__m128i *>(h1));

// 调用 keccakf 函数处理 h0 数组
keccakf(h0, 24);
// 调用 keccakf 函数处理 h1 数组
keccakf(h1, 24);

// 调用 extra_hashes 数组中的函数处理 ctx[0]->state 数组，并将结果存入 output 数组
extra_hashes[ctx[0]->state[0] & 3](ctx[0]->state, 200, output);
// 调用 extra_hashes 数组中的函数处理 ctx[1]->state 数组，并将结果存入 output 数组的第 33 个元素开始的位置
extra_hashes[ctx[1]->state[0] & 3](ctx[1]->state, 200, output + 32);
}

// 定义模板函数 cryptonight_triple_hash
template<Algorithm::Id ALGO, bool SOFT_AES>
inline void cryptonight_triple_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, struct cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
}

// 定义模板函数 cryptonight_quad_hash
template<Algorithm::Id ALGO, bool SOFT_AES>
inline void cryptonight_quad_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, struct cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
}

// 定义模板函数 cryptonight_penta_hash
template<Algorithm::Id ALGO, bool SOFT_AES>
inline void cryptonight_penta_hash(const uint8_t *__restrict__ input, size_t size, uint8_t *__restrict__ output, struct cryptonight_ctx **__restrict__ ctx, uint64_t height)
{
}

// 结束 xmrig 命名空间
} /* namespace xmrig */

// 结束条件编译指令块
#endif /* XMRIG_CRYPTONIGHT_ARM_H */
```