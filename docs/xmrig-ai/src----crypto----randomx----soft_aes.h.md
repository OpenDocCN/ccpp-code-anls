# `xmrig\src\crypto\randomx\soft_aes.h`

```cpp
/*
版权声明，保留所有权利
在源代码和二进制形式的重新分发和使用中，无论是否经过修改，都需要满足以下条件：
* 源代码的重新分发必须保留上述版权声明、条件列表和以下免责声明。
* 二进制形式的重新分发必须在提供的文档和/或其他材料中复制上述版权声明、条件列表和以下免责声明。
* 未经特定事先书面许可，不得使用版权持有者或其贡献者的名称来认可或推广从本软件衍生的产品。

本软件由版权持有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是合同责任、严格责任还是侵权（包括疏忽或其他方式）责任的任何理论，版权持有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责，即使已被告知可能发生此类损害的可能性。

*/
#pragma once

#include <stdint.h>
#include "crypto/randomx/intrin_portable.h"

// 定义外部全局变量
extern uint32_t lutEnc0[256];
extern uint32_t lutEnc1[256];
extern uint32_t lutEnc2[256];
extern uint32_t lutEnc3[256];
extern uint32_t lutDec0[256];
extern uint32_t lutDec1[256];
extern uint32_t lutDec2[256];
extern uint32_t lutDec3[256];
# 定义一个模板函数，用于对输入的 rx_vec_i128 进行 AES 加密操作，参数 soft 为加密模式
template<int soft> rx_vec_i128 aesenc(rx_vec_i128 in, rx_vec_i128 key);
# 定义一个模板函数，用于对输入的 rx_vec_i128 进行 AES 解密操作，参数 soft 为解密模式
template<int soft> rx_vec_i128 aesdec(rx_vec_i128 in, rx_vec_i128 key);

# 当加密模式为 1 时，对输入的 rx_vec_i128 进行 AES 加密操作
template<>
FORCE_INLINE rx_vec_i128 aesenc<1>(rx_vec_i128 in, rx_vec_i128 key) {
    # 定义一个长度为 16 的 volatile uint8_t 数组 s，用于存储输入的 rx_vec_i128 数据
    volatile uint8_t s[16];
    # 将输入的 rx_vec_i128 数据复制到数组 s 中
    memcpy((void*) s, &in, 16);

    # 使用查找表 lutEnc0 对数组 s 中的元素进行 AES 加密操作
    uint32_t s0 = lutEnc0[s[ 0]];
    uint32_t s1 = lutEnc0[s[ 4]];
    uint32_t s2 = lutEnc0[s[ 8]];
    uint32_t s3 = lutEnc0[s[12]];

    # 使用查找表 lutEnc1 对数组 s 中的元素进行 AES 加密操作，并与 s0-s3 进行异或运算
    s0 ^= lutEnc1[s[ 5]];
    s1 ^= lutEnc1[s[ 9]];
    s2 ^= lutEnc1[s[13]];
    s3 ^= lutEnc1[s[ 1]];

    # 使用查找表 lutEnc2 对数组 s 中的元素进行 AES 加密操作，并与 s0-s3 进行异或运算
    s0 ^= lutEnc2[s[10]];
    s1 ^= lutEnc2[s[14]];
    s2 ^= lutEnc2[s[ 2]];
    s3 ^= lutEnc2[s[ 6]];

    # 使用查找表 lutEnc3 对数组 s 中的元素进行 AES 加密操作，并与 s0-s3 进行异或运算
    s0 ^= lutEnc3[s[15]];
    s1 ^= lutEnc3[s[ 3]];
    s2 ^= lutEnc3[s[ 7]];
    s3 ^= lutEnc3[s[11]];

    # 将 s3, s2, s1, s0 组成一个新的 rx_vec_i128，并与 key 进行异或运算，返回结果
    return rx_xor_vec_i128(rx_set_int_vec_i128(s3, s2, s1, s0), key);
}

# 当解密模式为 1 时，对输入的 rx_vec_i128 进行 AES 解密操作
template<>
FORCE_INLINE rx_vec_i128 aesdec<1>(rx_vec_i128 in, rx_vec_i128 key) {
    # 定义一个长度为 16 的 volatile uint8_t 数组 s，用于存储输入的 rx_vec_i128 数据
    volatile uint8_t s[16];
    # 将输入的 rx_vec_i128 数据复制到数组 s 中
    memcpy((void*) s, &in, 16);

    # 使用查找表 lutDec0 对数组 s 中的元素进行 AES 解密操作
    uint32_t s0 = lutDec0[s[ 0]];
    uint32_t s1 = lutDec0[s[ 4]];
    uint32_t s2 = lutDec0[s[ 8]];
    uint32_t s3 = lutDec0[s[12]];

    # 使用查找表 lutDec1 对数组 s 中的元素进行 AES 解密操作，并与 s0-s3 进行异或运算
    s0 ^= lutDec1[s[13]];
    s1 ^= lutDec1[s[ 1]];
    s2 ^= lutDec1[s[ 5]];
    s3 ^= lutDec1[s[ 9]];

    # 使用查找表 lutDec2 对数组 s 中的元素进行 AES 解密操作，并与 s0-s3 进行异或运算
    s0 ^= lutDec2[s[10]];
    s1 ^= lutDec2[s[14]];
    s2 ^= lutDec2[s[ 2]];
    s3 ^= lutDec2[s[ 6]];

    # 使用查找表 lutDec3 对数组 s 中的元素进行 AES 解密操作，并与 s0-s3 进行异或运算
    s0 ^= lutDec3[s[ 7]];
    s1 ^= lutDec3[s[11]];
    s2 ^= lutDec3[s[15]];
    s3 ^= lutDec3[s[ 3]];

    # 将 s3, s2, s1, s0 组成一个新的 rx_vec_i128，并与 key 进行异或运算，返回结果
    return rx_xor_vec_i128(rx_set_int_vec_i128(s3, s2, s1, s0), key);
}

# 当加密模式为 2 时，对输入的 rx_vec_i128 进行 AES 加密操作
template<>
FORCE_INLINE rx_vec_i128 aesenc<2>(rx_vec_i128 in, rx_vec_i128 key) {
    # 定义四个变量 s0, s1, s2, s3，分别存储输入的 rx_vec_i128 的四个元素
    uint32_t s0, s1, s2, s3;

    # 将输入的 rx_vec_i128 的四个元素分别赋值给 s0, s1, s2, s3
    s0 = rx_vec_i128_w(in);
    s1 = rx_vec_i128_z(in);
    s2 = rx_vec_i128_y(in);
    s3 = rx_vec_i128_x(in);
    # 使用给定的四个32位整数进行异或和查表操作，生成一个128位整数向量
    rx_vec_i128 out = rx_set_int_vec_i128(
        (lutEnc0[s0 & 0xff] ^ lutEnc1[(s3 >> 8) & 0xff] ^ lutEnc2[(s2 >> 16) & 0xff] ^ lutEnc3[s1 >> 24]),
        (lutEnc0[s1 & 0xff] ^ lutEnc1[(s0 >> 8) & 0xff] ^ lutEnc2[(s3 >> 16) & 0xff] ^ lutEnc3[s2 >> 24]),
        (lutEnc0[s2 & 0xff] ^ lutEnc1[(s1 >> 8) & 0xff] ^ lutEnc2[(s0 >> 16) & 0xff] ^ lutEnc3[s3 >> 24]),
        (lutEnc0[s3 & 0xff] ^ lutEnc1[(s2 >> 8) & 0xff] ^ lutEnc2[(s1 >> 16) & 0xff] ^ lutEnc3[s0 >> 24])
    );
    
    # 将生成的128位整数向量与密钥进行异或操作
    return rx_xor_vec_i128(out, key);
# 定义一个模板函数，用于对输入的 rx_vec_i128 进行 AES 解密操作，输入参数为 in 和 key
template<>
# 强制内联函数，用于优化代码执行效率
FORCE_INLINE rx_vec_i128 aesdec<2>(rx_vec_i128 in, rx_vec_i128 key) {
    # 定义四个变量 s0, s1, s2, s3，用于存储 in 的四个字节
    uint32_t s0, s1, s2, s3;

    # 将 in 的四个字节分别赋值给 s0, s1, s2, s3
    s0 = rx_vec_i128_w(in);
    s1 = rx_vec_i128_z(in);
    s2 = rx_vec_i128_y(in);
    s3 = rx_vec_i128_x(in);

    # 根据 s0, s1, s2, s3 的值，通过查找表进行解密操作，并将结果存储在 out 中
    rx_vec_i128 out = rx_set_int_vec_i128(
        (lutDec0[s0 & 0xff] ^ lutDec1[(s1 >> 8) & 0xff] ^ lutDec2[(s2 >> 16) & 0xff] ^ lutDec3[s3 >> 24]),
        (lutDec0[s1 & 0xff] ^ lutDec1[(s2 >> 8) & 0xff] ^ lutDec2[(s3 >> 16) & 0xff] ^ lutDec3[s0 >> 24]),
        (lutDec0[s2 & 0xff] ^ lutDec1[(s3 >> 8) & 0xff] ^ lutDec2[(s0 >> 16) & 0xff] ^ lutDec3[s1 >> 24]),
        (lutDec0[s3 & 0xff] ^ lutDec1[(s0 >> 8) & 0xff] ^ lutDec2[(s1 >> 16) & 0xff] ^ lutDec3[s2 >> 24])
    );

    # 将 out 和 key 进行异或操作，并返回结果
    return rx_xor_vec_i128(out, key);
}

# 定义一个模板函数，用于对输入的 rx_vec_i128 进行 AES 加密操作，输入参数为 in 和 key
template<>
# 强制内联函数，用于优化代码执行效率
FORCE_INLINE rx_vec_i128 aesenc<0>(rx_vec_i128 in, rx_vec_i128 key) {
    # 调用 rx_aesenc_vec_i128 函数对 in 和 key 进行 AES 加密操作，并返回结果
    return rx_aesenc_vec_i128(in, key);
}

# 定义一个模板函数，用于对输入的 rx_vec_i128 进行 AES 解密操作，输入参数为 in 和 key
template<>
# 强制内联函数，用于优化代码执行效率
FORCE_INLINE rx_vec_i128 aesdec<0>(rx_vec_i128 in, rx_vec_i128 key) {
    # 调用 rx_aesdec_vec_i128 函数对 in 和 key 进行 AES 解密操作，并返回结果
    return rx_aesdec_vec_i128(in, key);
}
```