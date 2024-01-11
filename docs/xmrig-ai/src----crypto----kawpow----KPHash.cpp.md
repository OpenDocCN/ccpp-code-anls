# `xmrig\src\crypto\kawpow\KPHash.cpp`

```
/* XMRig
 * 版权声明，列出了各个作者的版权信息和联系方式
 */

#include "backend/cpu/Cpu.h"
#include "crypto/kawpow/KPHash.h"
#include "crypto/kawpow/KPCache.h"
#include "3rdparty/libethash/ethash.h"
#include "3rdparty/libethash/ethash_internal.h"
#include "3rdparty/libethash/data_sizes.h"
// 引入所需的头文件

#ifdef _MSC_VER
#include <intrin.h>
#endif
// 如果是 MSC 编译器，引入 intrin.h 头文件

namespace xmrig {
// 命名空间定义开始
# 定义一个包含15个元素的常量数组，用于存储Ravencoin Kawpow算法的哈希函数
static const uint32_t ravencoin_kawpow[15] = {
        0x00000072, //R
        0x00000041, //A
        0x00000056, //V
        0x00000045, //E
        0x0000004E, //N
        0x00000043, //C
        0x0000004F, //O
        0x00000049, //I
        0x0000004E, //N
        0x0000004B, //K
        0x00000041, //A
        0x00000057, //W
        0x00000050, //P
        0x0000004F, //O
        0x00000057, //W
};

# 定义两个常量，用于FNV1A哈希函数
static const uint32_t fnv_prime = 0x01000193;
static const uint32_t fnv_offset_basis = 0x811c9dc5;

# 定义一个内联函数，实现FNV1A哈希算法
static inline uint32_t fnv1a(uint32_t u, uint32_t v)
{
    return (u ^ v) * fnv_prime;
}

# 定义一个内联函数，实现KISS99伪随机数生成算法
static inline uint32_t kiss99(uint32_t& z, uint32_t& w, uint32_t& jsr, uint32_t& jcong)
{
    z = 36969 * (z & 0xffff) + (z >> 16);
    w = 18000 * (w & 0xffff) + (w >> 16);

    jcong = 69069 * jcong + 1234567;

    jsr ^= (jsr << 17);
    jsr ^= (jsr >> 13);
    jsr ^= (jsr << 5);

    return (((z << 16) + w) ^ jcong) + jsr;
}

# 定义一个内联函数，实现循环左移操作
static inline uint32_t rotl(uint32_t n, uint32_t c)
{
#ifdef _MSC_VER
    return _rotl(n, c);
#else
    c &= 31;
    uint32_t neg_c = (uint32_t)(-(int32_t)c);
    return (n << c) | (n >> (neg_c & 31));
#endif
}

# 定义一个内联函数，实现循环右移操作
static inline uint32_t rotr(uint32_t n, uint32_t c)
{
#ifdef _MSC_VER
    return _rotr(n, c);
#else
    c &= 31;
    uint32_t neg_c = (uint32_t)(-(int32_t)c);
    return (n >> c) | (n << (neg_c & 31));
#endif
}

# 定义一个内联函数，实现随机合并操作
static inline void random_merge(uint32_t& a, uint32_t b, uint32_t selector)
{
    const uint32_t x = (selector >> 16) % 31 + 1;
    switch (selector % 4)
    {
    case 0:
        a = (a * 33) + b;
        break;
    case 1:
        a = (a ^ b) * 33;
        break;
    case 2:
        a = rotl(a, x) ^ b;
        break;
    case 3:
        a = rotr(a, x) ^ b;
        break;
    default:
#ifdef _MSC_VER
        __assume(false);
#else
        __builtin_unreachable();
#endif
        break;
    }
}

# 定义一个内联函数，实现计算32位整数的前导零位数
static inline uint32_t clz(uint32_t a)
{
#ifdef _MSC_VER
    unsigned long index;
    _BitScanReverse(&index, a);
    # 如果 a 为真，则返回 31 - index，否则返回 32
    return a ? (31 - index) : 32;
#else
    return a ? (uint32_t)__builtin_clz(a) : 32;
#endif
}


static inline uint32_t popcount(uint32_t a)
{
#ifdef _MSC_VER
    return __popcnt(a);
#else
    return __builtin_popcount(a);
#endif
}


// Taken from https://en.wikipedia.org/wiki/Hamming_weight
static inline uint32_t popcount_soft(uint64_t x)
{
    // 定义常量 m1，用于计算每两位的数量
    constexpr uint64_t m1 = 0x5555555555555555ull;
    // 定义常量 m2，用于计算每四位的数量
    constexpr uint64_t m2 = 0x3333333333333333ull;
    // 定义常量 m4，用于计算每八位的数量
    constexpr uint64_t m4 = 0x0f0f0f0f0f0f0f0full;
    // 定义常量 h01，用于计算每八位的数量
    constexpr uint64_t h01 = 0x0101010101010101ull;

    // 计算每两位的数量并存储到对应的两位中
    x -= (x >> 1) & m1;
    // 计算每四位的数量并存储到对应的四位中
    x = (x & m2) + ((x >> 2) & m2);
    // 计算每八位的数量并存储到对应的八位中
    x = (x + (x >> 4)) & m4;
    // 返回 x 左移 56 位后的结果
    return (x * h01) >> 56;
}


static inline uint32_t random_math(uint32_t a, uint32_t b, uint32_t selector, bool has_popcnt)
{
    // 根据选择器的值执行不同的数学运算
    switch (selector % 11)
    {
    case 0:
        return a + b;
    case 1:
        return a * b;
    case 2:
        return (uint64_t(a) * b) >> 32;
    case 3:
        return (a < b) ? a : b;
    case 4:
        return rotl(a, b);
    case 5:
        return rotr(a, b);
    case 6:
        return a & b;
    case 7:
        return a | b;
    case 8:
        return a ^ b;
    case 9:
        return clz(a) + clz(b);
    case 10:
        // 根据是否支持 popcount 执行不同的计算
        if (has_popcnt)
            return popcount(a) + popcount(b);
        else
            return popcount_soft(a) + popcount_soft(b);
    default:
#ifdef _MSC_VER
        // 告诉编译器这里不可能到达
        __assume(false);
#else
        // 告诉编译器这里不可能到达
        __builtin_unreachable();
#endif
        break;
    }
}


void KPHash::calculate(const KPCache& light_cache, uint32_t block_height, const uint8_t (&header_hash)[32], uint64_t nonce, uint32_t (&output)[8], uint32_t (&mix_hash)[8])
{
    uint32_t keccak_state[25];
    uint32_t mix[LANES][REGS];

    // 将 header_hash 的内容复制到 keccak_state 中
    memcpy(keccak_state, header_hash, sizeof(header_hash));
    # 将nonce的值复制到keccak_state的第8个字节开始的位置
    memcpy(keccak_state + 8, &nonce, sizeof(nonce));
    # 将ravencoin_kawpow的值复制到keccak_state的第10个字节开始的位置
    memcpy(keccak_state + 10, ravencoin_kawpow, sizeof(ravencoin_kawpow));

    # 对keccak_state进行800轮的混淆
    ethash_keccakf800(keccak_state);

    # 使用fnv1a哈希算法计算keccak_state[0]的哈希值，并赋值给z
    uint32_t z = fnv1a(fnv_offset_basis, keccak_state[0]);
    # 使用fnv1a哈希算法计算z和keccak_state[1]的哈希值，并赋值给w
    uint32_t w = fnv1a(z, keccak_state[1]);
    # 定义jsr和jcong变量
    uint32_t jsr, jcong;

    # 对每个lane进行操作
    for (uint32_t l = 0; l < LANES; ++l) {
        # 将z和w的值赋给z1和w1
        uint32_t z1 = z;
        uint32_t w1 = w;
        # 使用fnv1a哈希算法计算w和l的哈希值，并赋值给jsr
        jsr = fnv1a(w, l);
        # 使用fnv1a哈希算法计算jsr和l的哈希值，并赋值给jcong
        jcong = fnv1a(jsr, l);

        # 对每个register进行操作
        for (uint32_t r = 0; r < REGS; ++r) {
            # 使用kiss99算法生成随机数，并赋值给mix[l][r]
            mix[l][r] = kiss99(z1, w1, jsr, jcong);
        }
    }

    # 计算prog_number的值
    const uint32_t prog_number = block_height / PERIOD_LENGTH;

    # 定义dst_seq和src_seq数组
    uint32_t dst_seq[REGS];
    uint32_t src_seq[REGS];

    # 使用fnv1a哈希算法计算prog_number的哈希值，并赋值给z
    z = fnv1a(fnv_offset_basis, prog_number);
    # 使用fnv1a哈希算法计算z和0的哈希值，并赋值给w
    w = fnv1a(z, 0);
    # 使用fnv1a哈希算法计算w和prog_number的哈希值，并赋值给jsr
    jsr = fnv1a(w, prog_number);
    # 使用fnv1a哈希算法计算jsr和0的哈希值，并赋值给jcong
    jcong = fnv1a(jsr, 0);

    # 初始化dst_seq和src_seq数组
    for (uint32_t i = 0; i < REGS; ++i)
    {
        dst_seq[i] = i;
        src_seq[i] = i;
    }

    # 对dst_seq和src_seq数组进行洗牌操作
    for (uint32_t i = REGS; i > 1; --i)
    {
        std::swap(dst_seq[i - 1], dst_seq[kiss99(z, w, jsr, jcong) % i]);
        std::swap(src_seq[i - 1], src_seq[kiss99(z, w, jsr, jcong) % i]);
    }

    # 计算epoch和num_items的值
    const uint32_t epoch = light_cache.epoch();
    const uint32_t num_items = static_cast<uint32_t>(dag_sizes[epoch] / ETHASH_MIX_BYTES / 2);

    # 定义num_words_per_lane和max_operations变量
    constexpr size_t num_words_per_lane = 256 / (sizeof(uint32_t) * LANES);
    constexpr int max_operations = (CNT_CACHE > CNT_MATH) ? CNT_CACHE : CNT_MATH;

    # 定义cache对象，并设置相关属性
    ethash_light cache;
    cache.cache = light_cache.data();
    cache.cache_size = light_cache.size();
    cache.block_number = block_height;

    # 计算cache.num_parent_nodes的值，并设置cache.reciprocal、cache.increment和cache.shift属性
    cache.num_parent_nodes = cache.cache_size / sizeof(node);
    KPCache::calculate_fast_mod_data(cache.num_parent_nodes, cache.reciprocal, cache.increment, cache.shift);

    # 将z、w、jsr和jcong的值赋给z0、w0、jsr0和jcong0
    uint32_t z0 = z;
    uint32_t w0 = w;
    uint32_t jsr0 = jsr;
    uint32_t jcong0 = jcong;

    # 判断CPU是否支持popcnt指令，并赋值给has_popcnt
    const bool has_popcnt = Cpu::info()->has(ICpuInfo::FLAG_POPCNT);

    # 定义lane_hash数组
    uint32_t lane_hash[LANES];
    # 对每个lane进行操作
    for (uint32_t l = 0; l < LANES; ++l)
    {
        # 对每个lane的哈希值进行初始化
        lane_hash[l] = fnv_offset_basis;
        # 对每个lane的mix数组进行哈希计算
        for (uint32_t i = 0; i < REGS; ++i) {
            lane_hash[l] = fnv1a(lane_hash[l], mix[l][i]);
        }
    }
    
    # 定义常量num_words为8
    
    # 对mix_hash数组进行初始化
    for (uint32_t i = 0; i < num_words; ++i) {
        mix_hash[i] = fnv_offset_basis;
    }
    
    # 对每个lane的哈希值进行混合计算
    for (uint32_t l = 0; l < LANES; ++l)
        mix_hash[l % num_words] = fnv1a(mix_hash[l % num_words], lane_hash[l]);
    
    # 将mix_hash数组的内容复制到keccak_state数组的指定位置
    memcpy(keccak_state + 8, mix_hash, sizeof(mix_hash));
    # 将ravencoin_kawpow数组的内容复制到keccak_state数组的指定位置
    memcpy(keccak_state + 16, ravencoin_kawpow, sizeof(uint32_t) * 9);
    
    # 调用ethash_keccakf800函数对keccak_state数组进行处理
    ethash_keccakf800(keccak_state);
    
    # 将keccak_state数组的内容复制到output数组
    memcpy(output, keccak_state, sizeof(output));
# 结束 xmrig 命名空间
}

# 结束 xmrig 命名空间
} // namespace xmrig
```