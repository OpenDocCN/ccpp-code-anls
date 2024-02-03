# `xmrig\src\3rdparty\argon2\lib\blake2\blake2.h`

```cpp
#ifndef ARGON2_BLAKE2_H
#define ARGON2_BLAKE2_H

#include <stddef.h>
#include <stdint.h>

// 定义 BLAKE2B 相关常量
enum blake2b_constant {
    BLAKE2B_BLOCKBYTES = 128,  // BLAKE2B 块大小
    BLAKE2B_OUTBYTES = 64,      // BLAKE2B 输出大小
    BLAKE2B_KEYBYTES = 64,      // BLAKE2B 密钥大小
    BLAKE2B_SALTBYTES = 16,     // BLAKE2B 盐值大小
    BLAKE2B_PERSONALBYTES = 16  // BLAKE2B 个性化参数大小
};

// 定义 BLAKE2B 状态结构体
typedef struct __blake2b_state {
    uint64_t h[8];          // 存储哈希值
    uint64_t t[2];          // 存储消息长度
    uint8_t buf[BLAKE2B_BLOCKBYTES];  // 存储消息块
    size_t buflen;          // 消息块长度
} blake2b_state;

/* Streaming API */
// 初始化 BLAKE2B 状态
void xmrig_ar2_blake2b_init(blake2b_state *S, size_t outlen);
// 更新 BLAKE2B 状态
void xmrig_ar2_blake2b_update(blake2b_state *S, const void *in, size_t inlen);
// 完成 BLAKE2B 计算
void xmrig_ar2_blake2b_final(blake2b_state *S, void *out, size_t outlen);

// 计算长消息的 BLAKE2B 哈希值
void xmrig_ar2_blake2b_long(void *out, size_t outlen, const void *in, size_t inlen);

#endif // ARGON2_BLAKE2_H
```