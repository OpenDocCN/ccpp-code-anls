# `xmrig\src\crypto\randomx\blake2\blake2-impl.h`

```cpp
/*
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>

保留所有权利。

在满足以下条件的情况下，允许以源代码和二进制形式重新分发和使用：
    * 必须保留源代码中的上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式中，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者 "按原样" 提供，明示或暗示的任何保证，包括但不限于对适销性和特定用途的适用性的暗示保证，都被拒绝。无论是在合同、严格责任还是侵权行为的任何理论下，版权持有人或贡献者都不对任何直接、间接、偶然、特殊、示范性或后果性损害 (包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断) 负责，即使事先被告知此类损害的可能性。

*/

/* 原始代码来自于 Argon2 参考源代码包，使用 CC0 许可证
 * https://github.com/P-H-C/phc-winner-argon2
 * 版权所有 2015
 * Daniel Dinu, Dmitry Khovratovich, Jean-Philippe Aumasson 和 Samuel Neves
*/

#ifndef PORTABLE_BLAKE2_IMPL_H
#define PORTABLE_BLAKE2_IMPL_H

#include <stdint.h>

#include "crypto/randomx/blake2/endian.h"

static FORCE_INLINE uint64_t load48(const void *src) {
*/

# 标识符 PORTABLE_BLAKE2_IMPL_H 的 ifndef 指令，用于防止头文件的重复包含

# 包含 stdint.h 头文件，以便使用 uint64_t 类型

# 包含 "crypto/randomx/blake2/endian.h" 头文件，以便使用其中的函数和宏

# 定义了一个静态内联函数 load48，该函数接受一个指向 void 类型的指针 src 作为参数，返回一个 uint64_t 类型的值
    # 将指针 src 强制转换为指向无符号 8 位整数的指针，并赋值给指针 p
    const uint8_t *p = (const uint8_t *)src;
    # 读取 p 指向的内存中的值，将其作为 8 位整数赋值给 w，并将指针 p 后移一位
    uint64_t w = *p++;
    # 读取 p 指向的内存中的值，将其作为 8 位整数左移 8 位后与 w 按位或，并将指针 p 后移一位
    w |= (uint64_t)(*p++) << 8;
    # 重复上述过程，每次左移 8 位，并将指针 p 后移一位
    w |= (uint64_t)(*p++) << 16;
    w |= (uint64_t)(*p++) << 24;
    w |= (uint64_t)(*p++) << 32;
    w |= (uint64_t)(*p++) << 40;
    # 返回最终结果 w
    return w;
# 将一个64位整数存储到48位的内存空间中
static FORCE_INLINE void store48(void *dst, uint64_t w) {
    # 将目标内存空间转换为指向字节的指针
    uint8_t *p = (uint8_t *)dst;
    # 将低8位存储到目标内存空间中，并将w右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将低8位存储到目标内存空间中，并将w右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将低8位存储到目标内存空间中，并将w右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将低8位存储到目标内存空间中，并将w右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将低8位存储到目标内存空间中，并将w右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将低8位存储到目标内存空间中，并将w右移8位
    *p++ = (uint8_t)w;
}

# 将一个32位整数循环右移指定位数
static FORCE_INLINE uint32_t rotr32(const uint32_t w, const unsigned c) {
    # 将w右移c位，并将w左移(32 - c)位，然后进行或运算
    return (w >> c) | (w << (32 - c));
}

# 将一个64位整数循环右移指定位数
static FORCE_INLINE uint64_t rotr64(const uint64_t w, const unsigned c) {
    # 将w右移c位，并将w左移(64 - c)位，然后进行或运算
    return (w >> c) | (w << (64 - c));
}

# 结束条件判断，如果没有定义过，则执行下面的代码
#endif
```