# `xmrig\src\crypto\randomx\aes_hash.cpp`

```
/*
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>

保留所有权利。

在源代码和二进制形式下，无论是否修改，只要满足以下条件，就可以重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式下，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，版权持有人或贡献者均不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，无论是合同责任、严格责任还是侵权行为（包括疏忽或其他），即使事先已被告知可能发生此类损害的可能性。
*/



#include <thread>  // 导入线程库
#include <vector>  // 导入向量库
#include <array>   // 导入数组库

#include "crypto/randomx/aes_hash.hpp"  // 导入 AES 哈希库
#include "base/tools/Chrono.h"  // 导入计时库
#include "crypto/randomx/randomx.h"  // 导入 RandomX 库
#include "crypto/randomx/soft_aes.h"  // 导入软件 AES 库
#include "crypto/randomx/instruction.hpp"  // 导入指令库
#include "crypto/randomx/common.hpp"  // 导入通用库
#include "crypto/rx/Profiler.h"  // 导入性能分析库

#define AES_HASH_1R_STATE0 0xd7983aad, 0xcc82db47, 0x9fa856de, 0x92b52c0d  // 定义 AES 哈希的初始状态
#define AES_HASH_1R_STATE1 0xace78057, 0xf59e125a, 0x15c7b798, 0x338d996e
#define AES_HASH_1R_STATE2 0xe8a07ce4, 0x5079506b, 0xae62c7d0, 0x6a770017
#define AES_HASH_1R_STATE3 0x7e994948, 0x79a10005, 0x07ad828d, 0x630a240c

#define AES_HASH_1R_XKEY0 0x06890201, 0x90dc56bf, 0x8b24949f, 0xf6fa8389
#define AES_HASH_1R_XKEY1 0xed18f99b, 0xee1043c6, 0x51f4e03c, 0x61b263d1

/*
    Calculate a 512-bit hash of 'input' using 4 lanes of AES.
    The input is treated as a set of round keys for the encryption
    of the initial state.

    'inputSize' must be a multiple of 64.

    For a 2 MiB input, this has the same security as 32768-round
    AES encryption.

    Hashing throughput: >20 GiB/s per CPU core with hardware AES
*/
template<int softAes>
void hashAes1Rx4(const void *input, size_t inputSize, void *hash) {
    const uint8_t* inptr = (uint8_t*)input;  // 将输入指针转换为 uint8_t 类型
    const uint8_t* inputEnd = inptr + inputSize;  // 计算输入结束位置

    rx_vec_i128 state0, state1, state2, state3;  // 定义四个 rx_vec_i128 类型的变量
    rx_vec_i128 in0, in1, in2, in3;  // 定义四个 rx_vec_i128 类型的变量

    //intial state
    state0 = rx_set_int_vec_i128(AES_HASH_1R_STATE0);  // 初始化 state0
    state1 = rx_set_int_vec_i128(AES_HASH_1R_STATE1);  // 初始化 state1
    state2 = rx_set_int_vec_i128(AES_HASH_1R_STATE2);  // 初始化 state2
    state3 = rx_set_int_vec_i128(AES_HASH_1R_STATE3);  // 初始化 state3

    //process 64 bytes at a time in 4 lanes
    while (inptr < inputEnd) {  // 循环处理每64字节的数据
        in0 = rx_load_vec_i128((rx_vec_i128*)inptr + 0);  // 加载第一个数据块
        in1 = rx_load_vec_i128((rx_vec_i128*)inptr + 1);  // 加载第二个数据块
        in2 = rx_load_vec_i128((rx_vec_i128*)inptr + 2);  // 加载第三个数据块
        in3 = rx_load_vec_i128((rx_vec_i128*)inptr + 3);  // 加载第四个数据块

        state0 = aesenc<softAes>(state0, in0);  // 使用 AES 加密算法处理 state0
        state1 = aesdec<softAes>(state1, in1);  // 使用 AES 解密算法处理 state1
        state2 = aesenc<softAes>(state2, in2);  // 使用 AES 加密算法处理 state2
        state3 = aesdec<softAes>(state3, in3);  // 使用 AES 解密算法处理 state3

        inptr += 64;  // 移动指针到下一个数据块
    }

    //two extra rounds to achieve full diffusion
    rx_vec_i128 xkey0 = rx_set_int_vec_i128(AES_HASH_1R_XKEY0);  // 初始化 xkey0
    rx_vec_i128 xkey1 = rx_set_int_vec_i128(AES_HASH_1R_XKEY1);  // 初始化 xkey1

    state0 = aesenc<softAes>(state0, xkey0);  // 使用 AES 加密算法处理 state0
    # 使用软件实现的 AES 解密算法对 state1 进行解密，使用 xkey0 作为密钥
    state1 = aesdec<softAes>(state1, xkey0);
    # 使用软件实现的 AES 加密算法对 state2 进行加密，使用 xkey0 作为密钥
    state2 = aesenc<softAes>(state2, xkey0);
    # 使用软件实现的 AES 解密算法对 state3 进行解密，使用 xkey0 作为密钥
    state3 = aesdec<softAes>(state3, xkey0);
    
    # 使用软件实现的 AES 加密算法对 state0 进行加密，使用 xkey1 作为密钥
    state0 = aesenc<softAes>(state0, xkey1);
    # 使用软件实现的 AES 解密算法对 state1 进行解密，使用 xkey1 作为密钥
    state1 = aesdec<softAes>(state1, xkey1);
    # 使用软件实现的 AES 加密算法对 state2 进行加密，使用 xkey1 作为密钥
    state2 = aesenc<softAes>(state2, xkey1);
    # 使用软件实现的 AES 解密算法对 state3 进行解密，使用 xkey1 作为密钥
    state3 = aesdec<softAes>(state3, xkey1);
    
    # 将 state0 的值存储到 hash 的第一个位置
    rx_store_vec_i128((rx_vec_i128*)hash + 0, state0);
    # 将 state1 的值存储到 hash 的第二个位置
    rx_store_vec_i128((rx_vec_i128*)hash + 1, state1);
    # 将 state2 的值存储到 hash 的第三个位置
    rx_store_vec_i128((rx_vec_i128*)hash + 2, state2);
    # 将 state3 的值存储到 hash 的第四个位置
    rx_store_vec_i128((rx_vec_i128*)hash + 3, state3);
// 实例化模板函数，使用 false 参数
template void hashAes1Rx4<false>(const void *input, size_t inputSize, void *hash);
// 实例化模板函数，使用 true 参数
template void hashAes1Rx4<true>(const void *input, size_t inputSize, void *hash);

// 定义 AES_GEN_1R_KEY0 到 AES_GEN_1R_KEY3 四个宏
#define AES_GEN_1R_KEY0 0xb4f44917, 0xdbb5552b, 0x62716609, 0x6daca553
#define AES_GEN_1R_KEY1 0x0da1dc4e, 0x1725d378, 0x846a710d, 0x6d7caf07
#define AES_GEN_1R_KEY2 0x3e20e345, 0xf4c0794f, 0x9f947ec6, 0x3f1262f1
#define AES_GEN_1R_KEY3 0x49169154, 0x16314c88, 0xb1ba317c, 0x6aef8135

/*
    使用单一 AES 轮加密状态，生成伪随机数据填充 'buffer'。
    每 16 字节输出使用一个 AES 轮加密状态，共有 4 个通道。

    'outputSize' 必须是 64 的倍数。

    修改后的状态写回 'state'，以允许多次调用此函数。
*/
template<int softAes>
void fillAes1Rx4(void *state, size_t outputSize, void *buffer) {
    const uint8_t* outptr = (uint8_t*)buffer;
    const uint8_t* outputEnd = outptr + outputSize;

    rx_vec_i128 state0, state1, state2, state3;
    rx_vec_i128 key0, key1, key2, key3;

    key0 = rx_set_int_vec_i128(AES_GEN_1R_KEY0);
    key1 = rx_set_int_vec_i128(AES_GEN_1R_KEY1);
    key2 = rx_set_int_vec_i128(AES_GEN_1R_KEY2);
    key3 = rx_set_int_vec_i128(AES_GEN_1R_KEY3);

    state0 = rx_load_vec_i128((rx_vec_i128*)state + 0);
    state1 = rx_load_vec_i128((rx_vec_i128*)state + 1);
    state2 = rx_load_vec_i128((rx_vec_i128*)state + 2);
    state3 = rx_load_vec_i128((rx_vec_i128*)state + 3);

    while (outptr < outputEnd) {
        state0 = aesdec<softAes>(state0, key0);
        state1 = aesenc<softAes>(state1, key1);
        state2 = aesdec<softAes>(state2, key2);
        state3 = aesenc<softAes>(state3, key3);

        rx_store_vec_i128((rx_vec_i128*)outptr + 0, state0);
        rx_store_vec_i128((rx_vec_i128*)outptr + 1, state1);
        rx_store_vec_i128((rx_vec_i128*)outptr + 2, state2);
        rx_store_vec_i128((rx_vec_i128*)outptr + 3, state3);

        outptr += 64;
    }
    # 将state0的值存储到state的第0个元素中
    rx_store_vec_i128((rx_vec_i128*)state + 0, state0);
    # 将state1的值存储到state的第1个元素中
    rx_store_vec_i128((rx_vec_i128*)state + 1, state1);
    # 将state2的值存储到state的第2个元素中
    rx_store_vec_i128((rx_vec_i128*)state + 2, state2);
    # 将state3的值存储到state的第3个元素中
    rx_store_vec_i128((rx_vec_i128*)state + 3, state3);
    
    
    这段代码将四个128位整数(state0, state1, state2, state3)的值存储到一个数组(state)的不同位置中。每个位置都使用rx_store_vec_i128函数来存储对应的值。
// 实例化 fillAes1Rx4 模板函数，传入 true 作为模板参数
template void fillAes1Rx4<true>(void *state, size_t outputSize, void *buffer);
// 实例化 fillAes1Rx4 模板函数，传入 false 作为模板参数
template void fillAes1Rx4<false>(void *state, size_t outputSize, void *buffer);

// 定义静态常量指令 inst
static constexpr randomx::Instruction inst{ 0xFF, 7, 7, 0xFF, 0xFFFFFFFFU };
// 使用 16 字节对齐方式定义静态常量指令数组 inst_mask
alignas(16) static const randomx::Instruction inst_mask[2] = { inst, inst };

// 定义 fillAes4Rx4 模板函数，传入 softAes 作为模板参数
template<int softAes>
void fillAes4Rx4(void *state, size_t outputSize, void *buffer) {
    // 定义指向 buffer 的 uint8_t 指针 outptr
    const uint8_t* outptr = (uint8_t*)buffer;
    // 定义指向 buffer 结尾的 uint8_t 指针 outputEnd
    const uint8_t* outputEnd = outptr + outputSize;

    // 定义四个 rx_vec_i128 类型的状态向量和八个密钥向量
    rx_vec_i128 state0, state1, state2, state3;
    rx_vec_i128 key0, key1, key2, key3, key4, key5, key6, key7;

    // 初始化八个密钥向量
    key0 = RandomX_CurrentConfig.fillAes4Rx4_Key[0];
    key1 = RandomX_CurrentConfig.fillAes4Rx4_Key[1];
    key2 = RandomX_CurrentConfig.fillAes4Rx4_Key[2];
    key3 = RandomX_CurrentConfig.fillAes4Rx4_Key[3];
    key4 = RandomX_CurrentConfig.fillAes4Rx4_Key[4];
    key5 = RandomX_CurrentConfig.fillAes4Rx4_Key[5];
    key6 = RandomX_CurrentConfig.fillAes4Rx4_Key[6];
    key7 = RandomX_CurrentConfig.fillAes4Rx4_Key[7];

    // 加载状态向量数据到 state0、state1、state2、state3
    state0 = rx_load_vec_i128((rx_vec_i128*)state + 0);
    state1 = rx_load_vec_i128((rx_vec_i128*)state + 1);
    state2 = rx_load_vec_i128((rx_vec_i128*)state + 2);
    state3 = rx_load_vec_i128((rx_vec_i128*)state + 3);

    // 定义 TRANSFORM 宏，进行 AES 加密解密操作
#define TRANSFORM do { \
    state0 = aesdec<softAes>(state0, key0); \
    state1 = aesenc<softAes>(state1, key0); \
    state2 = aesdec<softAes>(state2, key4); \
    state3 = aesenc<softAes>(state3, key4); \
    state0 = aesdec<softAes>(state0, key1); \
    state1 = aesenc<softAes>(state1, key1); \
    state2 = aesdec<softAes>(state2, key5); \
    state3 = aesenc<softAes>(state3, key5); \
    state0 = aesdec<softAes>(state0, key2); \
    state1 = aesenc<softAes>(state1, key2); \
    state2 = aesdec<softAes>(state2, key6); \
    state3 = aesenc<softAes>(state3, key6); \
    state0 = aesdec<softAes>(state0, key3); \
    state1 = aesenc<softAes>(state1, key3); \
    state2 = aesdec<softAes>(state2, key7); \
    # 使用软件AES算法对state3进行加密，并使用key7作为密钥
// do-while 循环，执行一次循环体
} while (0)

// 循环，i 从 0 到 1，每次增加 1，outptr 每次增加 64
for (int i = 0; i < 2; ++i, outptr += 64) {
    // 执行 TRANSFORM 操作
    TRANSFORM;
    // 将 state0 的值存储到 outptr 指向的内存中
    rx_store_vec_i128((rx_vec_i128*)outptr + 0, state0);
    // 将 state1 的值存储到 outptr 指向的内存中
    rx_store_vec_i128((rx_vec_i128*)outptr + 1, state1);
    // 将 state2 的值存储到 outptr 指向的内存中
    rx_store_vec_i128((rx_vec_i128*)outptr + 2, state2);
    // 将 state3 的值存储到 outptr 指向的内存中
    rx_store_vec_i128((rx_vec_i128*)outptr + 3, state3);
}

// 静态断言，检查 inst_mask 和 rx_vec_i128 的大小是否相等
static_assert(sizeof(inst_mask) == sizeof(rx_vec_i128), "Incorrect inst_mask size");
// 创建 mask，使用 inst_mask 的值
const rx_vec_i128 mask = *reinterpret_cast<const rx_vec_i128*>(inst_mask);

// while 循环，当 outptr 小于 outputEnd 时执行循环体
while (outptr < outputEnd) {
    // 执行 TRANSFORM 操作
    TRANSFORM;
    // 将 state0 与 mask 进行按位与操作后的值存储到 outptr 指向的内存中
    rx_store_vec_i128((rx_vec_i128*)outptr + 0, rx_and_vec_i128(state0, mask));
    // 将 state1 与 mask 进行按位与操作后的值存储到 outptr 指向的内存中
    rx_store_vec_i128((rx_vec_i128*)outptr + 1, rx_and_vec_i128(state1, mask));
    // 将 state2 与 mask 进行按位与操作后的值存储到 outptr 指向的内存中
    rx_store_vec_i128((rx_vec_i128*)outptr + 2, rx_and_vec_i128(state2, mask));
    // 将 state3 与 mask 进行按位与操作后的值存储到 outptr 指向的内存中
    rx_store_vec_i128((rx_vec_i128*)outptr + 3, rx_and_vec_i128(state3, mask));
    // outptr 每次增加 64
    outptr += 64;
}

// 实例化 fillAes4Rx4 模板函数，传入 true 参数
template void fillAes4Rx4<true>(void *state, size_t outputSize, void *buffer);
// 实例化 fillAes4Rx4 模板函数，传入 false 参数
template void fillAes4Rx4<false>(void *state, size_t outputSize, void *buffer);

// 模板函数，参数为 softAes 和 unroll
template<int softAes, int unroll>
void hashAndFillAes1Rx4(void *scratchpad, size_t scratchpadSize, void *hash, void* fill_state) {
    // 在 PROFILE_SCOPE(RandomX_AES) 中执行以下代码
    PROFILE_SCOPE(RandomX_AES);

    // 将 scratchpad 转换为 uint8_t* 类型
    uint8_t* scratchpadPtr = (uint8_t*)scratchpad;
    // 计算 scratchpadEnd 的值
    const uint8_t* scratchpadEnd = scratchpadPtr + scratchpadSize;

    // 初始化 hash_state0
    rx_vec_i128 hash_state0 = rx_set_int_vec_i128(AES_HASH_1R_STATE0);
    // 初始化 hash_state1
    rx_vec_i128 hash_state1 = rx_set_int_vec_i128(AES_HASH_1R_STATE1);
    // 初始化 hash_state2
    rx_vec_i128 hash_state2 = rx_set_int_vec_i128(AES_HASH_1R_STATE2);
    // 初始化 hash_state3
    rx_vec_i128 hash_state3 = rx_set_int_vec_i128(AES_HASH_1R_STATE3);

    // 初始化 key0
    const rx_vec_i128 key0 = rx_set_int_vec_i128(AES_GEN_1R_KEY0);
    // 初始化 key1
    const rx_vec_i128 key1 = rx_set_int_vec_i128(AES_GEN_1R_KEY1);
    // 初始化 key2
    const rx_vec_i128 key2 = rx_set_int_vec_i128(AES_GEN_1R_KEY2);
    // 初始化 key3
    const rx_vec_i128 key3 = rx_set_int_vec_i128(AES_GEN_1R_KEY3);
    # 将fill_state中的数据加载到四个128位的向量中
    rx_vec_i128 fill_state0 = rx_load_vec_i128((rx_vec_i128*)fill_state + 0);
    rx_vec_i128 fill_state1 = rx_load_vec_i128((rx_vec_i128*)fill_state + 1);
    rx_vec_i128 fill_state2 = rx_load_vec_i128((rx_vec_i128*)fill_state + 2);
    rx_vec_i128 fill_state3 = rx_load_vec_i128((rx_vec_i128*)fill_state + 3);

    # 定义一个常量PREFETCH_DISTANCE为7168
    constexpr int PREFETCH_DISTANCE = 7168;
    # 计算预取指针的位置
    const char* prefetchPtr = ((const char*)scratchpad) + PREFETCH_DISTANCE;
    # 减去预取距离，得到scratchpadEnd的新值
    scratchpadEnd -= PREFETCH_DISTANCE;

    # 循环两次，每次处理64字节，分为4个lane
    for (int i = 0; i < 2; ++i) {
        # 在scratchpadPtr小于scratchpadEnd的情况下，执行以下代码块
        while (scratchpadPtr < scratchpadEnd) {
// 定义一个宏，用于对哈希状态进行处理
#define HASH_STATE(k) \
            // 使用软件AES对哈希状态进行处理
            hash_state0 = aesenc<softAes>(hash_state0, rx_load_vec_i128((rx_vec_i128*)scratchpadPtr + k * 4 + 0)); \
            hash_state1 = aesdec<softAes>(hash_state1, rx_load_vec_i128((rx_vec_i128*)scratchpadPtr + k * 4 + 1)); \
            hash_state2 = aesenc<softAes>(hash_state2, rx_load_vec_i128((rx_vec_i128*)scratchpadPtr + k * 4 + 2)); \
            hash_state3 = aesdec<softAes>(hash_state3, rx_load_vec_i128((rx_vec_i128*)scratchpadPtr + k * 4 + 3));

    }

    // 将填充状态存储到指定位置
    rx_store_vec_i128((rx_vec_i128*)fill_state + 0, fill_state0);
    rx_store_vec_i128((rx_vec_i128*)fill_state + 1, fill_state1);
    rx_store_vec_i128((rx_vec_i128*)fill_state + 2, fill_state2);
    rx_store_vec_i128((rx_vec_i128*)fill_state + 3, fill_state3);

    // 为了实现完全扩散，进行两轮额外的处理
    rx_vec_i128 xkey0 = rx_set_int_vec_i128(AES_HASH_1R_XKEY0);
    rx_vec_i128 xkey1 = rx_set_int_vec_i128(AES_HASH_1R_XKEY1);

    // 使用软件AES对哈希状态进行处理
    hash_state0 = aesenc<softAes>(hash_state0, xkey0);
    hash_state1 = aesdec<softAes>(hash_state1, xkey0);
    hash_state2 = aesenc<softAes>(hash_state2, xkey0);
    hash_state3 = aesdec<softAes>(hash_state3, xkey0);

    // 使用软件AES对哈希状态进行处理
    hash_state0 = aesenc<softAes>(hash_state0, xkey1);
    hash_state1 = aesdec<softAes>(hash_state1, xkey1);
    hash_state2 = aesenc<softAes>(hash_state2, xkey1);
    hash_state3 = aesdec<softAes>(hash_state3, xkey1);

    // 输出哈希值
    rx_store_vec_i128((rx_vec_i128*)hash + 0, hash_state0);
    rx_store_vec_i128((rx_vec_i128*)hash + 1, hash_state1);
    rx_store_vec_i128((rx_vec_i128*)hash + 2, hash_state2);
    rx_store_vec_i128((rx_vec_i128*)hash + 3, hash_state3);
}

// 实例化模板函数，对不同的参数进行处理
template void hashAndFillAes1Rx4<0,2>(void* scratchpad, size_t scratchpadSize, void* hash, void* fill_state);
template void hashAndFillAes1Rx4<1,1>(void* scratchpad, size_t scratchpadSize, void* hash, void* fill_state);
template void hashAndFillAes1Rx4<2,1>(void* scratchpad, size_t scratchpadSize, void* hash, void* fill_state);
# 声明一个模板函数，函数名为hashAndFillAes1Rx4，模板参数为2和2
template void hashAndFillAes1Rx4<2,2>(void* scratchpad, size_t scratchpadSize, void* hash, void* fill_state);
# 声明一个模板函数，函数名为hashAndFillAes1Rx4，模板参数为2和4
template void hashAndFillAes1Rx4<2,4>(void* scratchpad, size_t scratchpadSize, void* hash, void* fill_state);

# 定义一个指向hashAndFillAes1Rx4<1,1>函数的指针
hashAndFillAes1Rx4_impl* softAESImpl = &hashAndFillAes1Rx4<1,1>;

# 定义一个函数，函数名为SelectSoftAESImpl，参数为threadsCount
void SelectSoftAESImpl(size_t threadsCount)
{
  # 定义一个常量，表示测试时长为100毫秒
  constexpr uint64_t test_length_ms = 100;
  # 定义一个包含4个hashAndFillAes1Rx4_impl指针的数组impl
  const std::array<hashAndFillAes1Rx4_impl *, 4> impl = {
    &hashAndFillAes1Rx4<1,1>,
    &hashAndFillAes1Rx4<2,1>,
    &hashAndFillAes1Rx4<2,2>,
    &hashAndFillAes1Rx4<2,4>,
  };
  # 初始化fast_idx为0，fast_speed为0.0
  size_t fast_idx = 0;
  double fast_speed = 0.0;
  # 进行3次循环
  for (size_t run = 0; run < 3; ++run) {
    # 遍历impl数组
    for (size_t i = 0; i < impl.size(); ++i) {
      # 记录当前时间
      const double t1 = xmrig::Chrono::highResolutionMSecs();
      # 创建一个包含threadsCount个元素的count数组，初始化为0
      std::vector<uint32_t> count(threadsCount, 0);
      # 创建一个线程数组
      std::vector<std::thread> threads;
      # 遍历threadsCount次
      for (size_t t = 0; t < threadsCount; ++t) {
        # 向线程数组中添加一个线程
        threads.emplace_back([&, t]() {
          # 创建一个大小为10 * 1024的scratchpad数组
          std::vector<uint8_t> scratchpad(10 * 1024);
          # 创建一个大小为64的hash数组，初始化为0
          alignas(16) uint8_t hash[64] = {};
          # 创建一个大小为64的state数组，初始化为0
          alignas(16) uint8_t state[64] = {};
          # 执行循环，直到当前时间减去t1大于等于test_length_ms
          do {
          # 调用impl[i]指向的函数，传入参数，执行hashAndFillAes1Rx4函数
          (*impl[i])(scratchpad.data(), scratchpad.size(), hash, state);
          # 将count[t]加1
          ++count[t];
          } while (xmrig::Chrono::highResolutionMSecs() - t1 < test_length_ms);
        });
      }
      # 初始化total为0
      uint32_t total = 0;
      # 遍历threadsCount次
      for (size_t t = 0; t < threadsCount; ++t) {
        # 等待线程结束
        threads[t].join();
        # 将count[t]累加到total
        total += count[t];
      }
      # 记录当前时间
      const double t2 = xmrig::Chrono::highResolutionMSecs();
      # 计算速度
      const double speed = total * 1e3 / (t2 - t1);
      # 如果速度大于fast_speed，则更新fast_idx和fast_speed
      if (speed > fast_speed) {
        fast_idx = i;
        fast_speed = speed;
      }
    }
  }
  # 将softAESImpl指向impl[fast_idx]
  softAESImpl = impl[fast_idx];
}
```