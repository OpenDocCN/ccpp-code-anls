# `xmrig\src\base\crypto\keccak.cpp`

```
/* XMRig
 * 版权声明
 */

#include <memory.h>
#include "base/crypto/keccak.h"

// 定义常量
#define HASH_DATA_AREA 136
#define KECCAK_ROUNDS 24

// 定义宏
#ifndef ROTL64
#define ROTL64(x, y) (((x) << (y)) | ((x) >> (64 - (y))))
#endif

// 定义常量数组
const uint64_t keccakf_rndc[24] =
{
    0x0000000000000001, 0x0000000000008082, 0x800000000000808a,
    0x8000000080008000, 0x000000000000808b, 0x0000000080000001,
    0x8000000080008081, 0x8000000000008009, 0x000000000000008a,
    0x0000000000000088, 0x0000000080008009, 0x000000008000000a,
    0x000000008000808b, 0x800000000000008b, 0x8000000000008089,
    // 其余的数组元素
};
    # 这些是十六进制数值，表示程序中的特定常量或者标志
    0x8000000000008003, 0x8000000000008002, 0x8000000000000080,
    0x000000000000800a, 0x800000008000000a, 0x8000000080008081,
    0x8000000000008080, 0x0000000080000001, 0x8000000080008008
// 更新状态，执行给定轮数的循环
void xmrig::keccakf(uint64_t st[25], int rounds)
{
    // 对于每一轮
    for (int round = 0; round < rounds; ++round) {
        uint64_t bc[5];

        // Theta 步骤
        bc[0] = st[0] ^ st[5] ^ st[10] ^ st[15] ^ st[20];
        bc[1] = st[1] ^ st[6] ^ st[11] ^ st[16] ^ st[21];
        bc[2] = st[2] ^ st[7] ^ st[12] ^ st[17] ^ st[22];
        bc[3] = st[3] ^ st[8] ^ st[13] ^ st[18] ^ st[23];
        bc[4] = st[4] ^ st[9] ^ st[14] ^ st[19] ^ st[24];

        // 定义 X 宏，执行一系列操作
#define X(i) { \
            const uint64_t t = bc[(i + 4) % 5] ^ ROTL64(bc[(i + 1) % 5], 1); \
            st[i     ] ^= t; \
            st[i +  5] ^= t; \
            st[i + 10] ^= t; \
            st[i + 15] ^= t; \
            st[i + 20] ^= t; \
        }

        // 执行 X 宏
        X(0); X(1); X(2); X(3); X(4);

    }
}

// 计算给定字节长度的 keccak 哈希（md）从 "in"
typedef uint64_t state_t[25];

// 计算 keccak 哈希
void xmrig::keccak(const uint8_t *in, int inlen, uint8_t *md, int mdlen)
{
    state_t st;
    alignas(8) uint8_t temp[144];
    int i, rsiz, rsizw;

    // 计算 rsiz 和 rsizw
    rsiz = sizeof(state_t) == mdlen ? HASH_DATA_AREA : 200 - 2 * mdlen;
    rsizw = rsiz / 8;

    // 将 st 数组清零
    memset(st, 0, sizeof(st));

    // 对于每个 rsiz 长度的块
    for ( ; inlen >= rsiz; inlen -= rsiz, in += rsiz) {
        for (i = 0; i < rsizw; i++) {
            st[i] ^= ((uint64_t *) in)[i];
        }

        // 执行 keccakf 函数
        xmrig::keccakf(st, KECCAK_ROUNDS);
    }

    // 最后一个块和填充
    memcpy(temp, in, inlen);
    temp[inlen++] = 1;
    memset(temp + inlen, 0, rsiz - inlen);
    temp[rsiz - 1] |= 0x80;

    for (i = 0; i < rsizw; i++) {
        st[i] ^= ((uint64_t *) temp)[i];
    }

    // 执行 keccakf 函数
    keccakf(st, KECCAK_ROUNDS);

    // 将结果复制到 md 中
    memcpy(md, st, mdlen);
}
```