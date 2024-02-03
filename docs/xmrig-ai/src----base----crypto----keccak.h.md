# `xmrig\src\base\crypto\keccak.h`

```cpp
// 声明头文件防止重复包含
#ifndef XMRIG_KECCAK_H
#define XMRIG_KECCAK_H

// 包含必要的头文件
#include <cstdint>
#include <cstring>

// 声明命名空间
namespace xmrig {

// 计算给定长度的字节流的 keccak 哈希值
void keccak(const uint8_t *in, int inlen, uint8_t *md, int mdlen);

// 重载函数，计算给定长度的字节流的 keccak 哈希值
inline void keccak(const uint8_t *in, size_t inlen, uint8_t *md)
{
    keccak(in, static_cast<int>(inlen), md, 200);
}

// 重载函数，计算给定长度的字符流的 keccak 哈希值
inline void keccak(const char *in, size_t inlen, uint8_t *md)
{
    keccak(reinterpret_cast<const uint8_t *>(in), static_cast<int>(inlen), md, 200);
}

// 更新状态的函数声明
void keccakf(uint64_t st[25], int norounds);

// 结束头文件声明
#endif
} /* 结束 xmrig 命名空间 */

#endif /* 结束 XMRIG_KECCAK_H 头文件 */
```