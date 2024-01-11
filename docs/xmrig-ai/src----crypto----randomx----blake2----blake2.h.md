# `xmrig\src\crypto\randomx\blake2\blake2.h`

```
/*
版权声明：
版权所有 (c) 2018-2019, tevador <tevador@gmail.com>
保留所有权利。

在满足以下条件的情况下，允许以源代码和二进制形式重新分发和使用：
    * 必须保留上述版权声明、此条件列表和以下免责声明。
    * 在二进制形式的重新分发中，必须在文档和/或其他提供的材料中复制上述版权声明、此条件列表和以下免责声明。
    * 未经特定事先书面许可，不得使用版权持有人的名称或贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有人和贡献者“按原样”提供，任何明示或暗示的保证，包括但不限于对适销性和适用性的暗示保证，都被否认。无论是在合同、严格责任还是侵权行为的任何情况下，版权持有人或贡献者都不对任何直接、间接、偶然、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润损失，或业务中断）承担任何责任，即使事先被告知此类损害的可能性。
*/

/* 原始代码来自于 Argon2 参考源代码包，使用 CC0 许可证
 * https://github.com/P-H-C/phc-winner-argon2
 * 版权所有 2015
 * Daniel Dinu, Dmitry Khovratovich, Jean-Philippe Aumasson 和 Samuel Neves
*/

#ifndef PORTABLE_BLAKE2_H
#define PORTABLE_BLAKE2_H

#include <stdint.h>
#include <limits.h>

#if defined(__cplusplus)
extern "C" {
#endif
    # 定义枚举类型 blake2b_constant，包含以下常量
    enum blake2b_constant {
        # 定义常量 BLAKE2B_BLOCKBYTES，值为 128
        BLAKE2B_BLOCKBYTES = 128,
        # 定义常量 BLAKE2B_OUTBYTES，值为 64
        BLAKE2B_OUTBYTES = 64,
        # 定义常量 BLAKE2B_KEYBYTES，值为 64
        BLAKE2B_KEYBYTES = 64,
        # 定义常量 BLAKE2B_SALTBYTES，值为 16
        BLAKE2B_SALTBYTES = 16,
        # 定义常量 BLAKE2B_PERSONALBYTES，值为 16
        BLAKE2B_PERSONALBYTES = 16
    };
#pragma pack(push, 1)
    // 定义一个结构体，用于存储 BLAKE2B 算法的参数
    typedef struct __blake2b_param {
        uint8_t digest_length;                   /* 1 */
        uint8_t key_length;                      /* 2 */
        uint8_t fanout;                          /* 3 */
        uint8_t depth;                           /* 4 */
        uint32_t leaf_length;                    /* 8 */
        uint64_t node_offset;                    /* 16 */
        uint8_t node_depth;                      /* 17 */
        uint8_t inner_length;                    /* 18 */
        uint8_t reserved[14];                    /* 32 */
        uint8_t salt[BLAKE2B_SALTBYTES];         /* 48 */
        uint8_t personal[BLAKE2B_PERSONALBYTES]; /* 64 */
    } blake2b_param;
#pragma pack(pop)

    // 定义一个结构体，用于存储 BLAKE2B 算法的状态
    typedef struct __blake2b_state {
        uint64_t h[8];
        uint64_t t[2];
        uint64_t f[2];
        uint8_t buf[BLAKE2B_BLOCKBYTES];
        unsigned buflen;
        unsigned outlen;
        uint8_t last_node;
    } blake2b_state;

    /* Ensure param structs have not been wrongly padded */
    /* Poor man's static_assert */
    // 确保参数结构体没有错误地填充
    // 类似于静态断言
    enum {
        blake2_size_check_0 = 1 / !!(CHAR_BIT == 8),
        blake2_size_check_2 =
        1 / !!(sizeof(blake2b_param) == sizeof(uint64_t) * CHAR_BIT)
    };

    /* Streaming API */
    // 初始化 BLAKE2B 状态
    int rx_blake2b_init(blake2b_state *S, size_t outlen);
    // 初始化带有密钥的 BLAKE2B 状态
    int rx_blake2b_init_key(blake2b_state *S, size_t outlen, const void *key, size_t keylen);
    // 使用参数初始化 BLAKE2B 状态
    int rx_blake2b_init_param(blake2b_state *S, const blake2b_param *P);
    // 更新 BLAKE2B 状态
    int rx_blake2b_update(blake2b_state *S, const void *in, size_t inlen);
    // 完成 BLAKE2B 计算
    int rx_blake2b_final(blake2b_state *S, void *out, size_t outlen);

    /* Simple API */
    // 压缩整数
    void rx_blake2b_compress_integer(blake2b_state * S, const uint8_t * block);
    // 使用 SSE41 指令集压缩
    void rx_blake2b_compress_sse41(blake2b_state * S, const uint8_t * block);
    // 默认的 BLAKE2B 计算
    int rx_blake2b_default(void* out, size_t outlen, const void* in, size_t inlen);

    // BLAKE2B 压缩函数指针
    extern void (*rx_blake2b_compress)(blake2b_state * S, const uint8_t * block);
    # 声明一个指向函数的指针，该函数接受 void* 类型的参数，并返回 int 类型的值
    extern int (*rx_blake2b)(void* out, size_t outlen, const void* in, size_t inlen);
    
    # Argon2 团队 - 开始代码
    # 声明一个函数，该函数接受 void* 类型的参数，并返回 int 类型的值
    int rxa2_blake2b_long(void *out, size_t outlen, const void *in, size_t inlen);
    # Argon2 团队 - 结束代码
# 如果是 C++ 代码，则结束 extern "C" 块
#if defined(__cplusplus)
}
#endif

# 结束头文件的条件编译
#endif
```