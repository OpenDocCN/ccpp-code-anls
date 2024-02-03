# `xmrig\src\crypto\randomx\blake2\blake2b.c`

```cpp
/*
版权声明，允许在源代码和二进制形式下重新分发和使用，但需要满足以下条件：
* 在源代码的重新分发中必须保留上述版权声明、条件列表和以下免责声明。
* 在二进制形式的重新分发中，必须在提供的文档或其他材料中复制上述版权声明、条件列表和以下免责声明。
* 未经特定事先书面许可，不得使用版权持有者的名称或其贡献者的名称来认可或推广从本软件派生的产品。

本软件由版权持有者和贡献者“按原样”提供，不提供任何明示或暗示的担保，包括但不限于对适销性和特定用途的适用性的暗示担保。在任何情况下，无论是在合同、严格责任还是侵权（包括疏忽或其他方式）的情况下，版权持有者或贡献者均不对任何直接、间接、偶发、特殊、惩罚性或后果性损害（包括但不限于替代商品或服务的采购、使用、数据或利润的损失，或业务中断）负责，即使已被告知可能发生此类损害的可能性。
*/

/* Argon2 参考源代码包中的原始代码，使用 CC0 许可证
 * https://github.com/P-H-C/phc-winner-argon2
 * 版权所有 2015
 * Daniel Dinu, Dmitry Khovratovich, Jean-Philippe Aumasson 和 Samuel Neves
*/

#include <stdint.h>
#include <string.h>
#include <stdio.h>

#include "crypto/randomx/blake2/blake2.h"
#include "crypto/randomx/blake2/blake2-impl.h"

const uint64_t blake2b_IV[8] = {
    # 定义一个包含8个64位无符号整数的数组，作为初始常量
    UINT64_C(0x6a09e667f3bcc908), UINT64_C(0xbb67ae8584caa73b),
    UINT64_C(0x3c6ef372fe94f82b), UINT64_C(0xa54ff53a5f1d36f1),
    UINT64_C(0x510e527fade682d1), UINT64_C(0x9b05688c2b3e6c1f),
    UINT64_C(0x1f83d9abfb41bd6b), UINT64_C(0x5be0cd19137e2179)
# 定义一个常量数组，用于后续计算
static const uint8_t blake2b_sigma[12][16] = {
    {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15},
    {14, 10, 4, 8, 9, 15, 13, 6, 1, 12, 0, 2, 11, 7, 5, 3},
    {11, 8, 12, 0, 5, 2, 15, 13, 10, 14, 3, 6, 7, 1, 9, 4},
    {7, 9, 3, 1, 13, 12, 11, 14, 2, 6, 5, 10, 4, 0, 15, 8},
    {9, 0, 5, 7, 2, 4, 10, 15, 14, 1, 11, 12, 6, 8, 3, 13},
    {2, 12, 6, 10, 0, 11, 8, 3, 4, 13, 7, 5, 15, 14, 1, 9},
    {12, 5, 1, 15, 14, 13, 4, 10, 0, 7, 6, 3, 9, 2, 8, 11},
    {13, 11, 7, 14, 12, 1, 3, 9, 5, 0, 15, 4, 8, 6, 2, 10},
    {6, 15, 14, 9, 11, 3, 0, 8, 12, 2, 13, 7, 1, 4, 10, 5},
    {10, 2, 8, 4, 7, 6, 1, 5, 15, 11, 9, 14, 3, 12, 13, 0},
    {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15},
    {14, 10, 4, 8, 9, 15, 13, 6, 1, 12, 0, 2, 11, 7, 5, 3},
};

# 设置最后一个节点的标志位
static FORCE_INLINE void blake2b_set_lastnode(blake2b_state *S) {
    S->f[1] = (uint64_t)-1;
}

# 设置最后一个块的标志位
static FORCE_INLINE void blake2b_set_lastblock(blake2b_state *S) {
    if (S->last_node) {
        blake2b_set_lastnode(S);
    }
    S->f[0] = (uint64_t)-1;
}

# 增加计数器的值
static FORCE_INLINE void blake2b_increment_counter(blake2b_state *S,
    uint64_t inc) {
    S->t[0] += inc;
    S->t[1] += (S->t[0] < inc);
}

# 使状态无效
static FORCE_INLINE void blake2b_invalidate_state(blake2b_state *S) {
    # 清除内部内存
    # clear_internal_memory(S, sizeof(*S));      /* wipe */
    # 设置最后一个块的标志位，使其无效
    blake2b_set_lastblock(S); /* invalidate for further use */
}

# 初始化状态
static FORCE_INLINE void blake2b_init0(blake2b_state *S) {
    # 将状态的内存清零
    memset(S, 0, sizeof(*S));
    # 将初始向量复制到状态的哈希值中
    memcpy(S->h, blake2b_IV, sizeof(S->h));
}

# 使用参数初始化 blake2b 状态
int rx_blake2b_init_param(blake2b_state *S, const blake2b_param *P) {
    # 将参数转换为字节数组
    const unsigned char *p = (const unsigned char *)P;
    unsigned int i;

    # 如果参数或状态为空，则返回错误
    if (NULL == P || NULL == S) {
        return -1;
    }

    # 初始化状态
    blake2b_init0(S);
    # 将初始向量与参数块进行异或运算
    for (i = 0; i < 8; ++i) {
        S->h[i] ^= load64(&p[i * sizeof(S->h[i])]);
    }
    # 设置输出长度
    S->outlen = P->digest_length;
    return 0;
}

# 顺序初始化 blake2b 状态
int rx_blake2b_init(blake2b_state *S, size_t outlen) {
    # 创建一个名为P的blake2b_param结构体对象

    if (S == NULL) {
        # 如果输入的S为空指针，则返回-1
        return -1;
    }

    if ((outlen == 0) || (outlen > BLAKE2B_OUTBYTES)) {
        # 如果输出长度为0或者超过了BLAKE2B_OUTBYTES的长度，则将S的状态置为无效，并返回-1
        blake2b_invalidate_state(S);
        return -1;
    }

    /* 设置无密钥的BLAKE2的参数块 */
    # 设置摘要长度为outlen
    P.digest_length = (uint8_t)outlen;
    # 设置密钥长度为0
    P.key_length = 0;
    # 设置分支数为1
    P.fanout = 1;
    # 设置深度为1
    P.depth = 1;
    # 设置叶子长度为0
    P.leaf_length = 0;
    # 设置节点偏移为0
    P.node_offset = 0;
    # 设置节点深度为0
    P.node_depth = 0;
    # 设置内部长度为0
    P.inner_length = 0;
    # 将保留字段全部置为0
    memset(P.reserved, 0, sizeof(P.reserved));
    # 将盐值全部置为0
    memset(P.salt, 0, sizeof(P.salt));
    # 将个人化参数全部置为0
    memset(P.personal, 0, sizeof(P.personal));

    # 调用rx_blake2b_init_param函数，使用参数P初始化S的状态
    return rx_blake2b_init_param(S, &P);
}
// 结束函数 rx_blake2b_init_key

int rx_blake2b_init_key(blake2b_state *S, size_t outlen, const void *key, size_t keylen) {
    blake2b_param P;

    if (S == NULL) {
        return -1;
    }
    // 如果输出长度为 0 或者超过 BLAKE2B_OUTBYTES，使状态无效并返回 -1
    if ((outlen == 0) || (outlen > BLAKE2B_OUTBYTES)) {
        blake2b_invalidate_state(S);
        return -1;
    }
    // 如果 key 为 0 或者 key 长度为 0 或者超过 BLAKE2B_KEYBYTES，使状态无效并返回 -1
    if ((key == 0) || (keylen == 0) || (keylen > BLAKE2B_KEYBYTES)) {
        blake2b_invalidate_state(S);
        return -1;
    }
    // 设置参数块用于带密钥的 BLAKE2
    P.digest_length = (uint8_t)outlen;
    P.key_length = (uint8_t)keylen;
    P.fanout = 1;
    P.depth = 1;
    P.leaf_length = 0;
    P.node_offset = 0;
    P.node_depth = 0;
    P.inner_length = 0;
    memset(P.reserved, 0, sizeof(P.reserved));
    memset(P.salt, 0, sizeof(P.salt));
    memset(P.personal, 0, sizeof(P.personal));
    // 如果初始化参数失败，使状态无效并返回 -1
    if (rx_blake2b_init_param(S, &P) < 0) {
        blake2b_invalidate_state(S);
        return -1;
    }
    {
        uint8_t block[BLAKE2B_BLOCKBYTES];
        memset(block, 0, BLAKE2B_BLOCKBYTES);
        memcpy(block, key, keylen);
        rx_blake2b_update(S, block, BLAKE2B_BLOCKBYTES);
        /* Burn the key from stack */
        // 从堆栈中清除密钥
        //clear_internal_memory(block, BLAKE2B_BLOCKBYTES);
    }
    return 0;
}

void rx_blake2b_compress_integer(blake2b_state *S, const uint8_t *block) {
    uint64_t m[16];
    uint64_t v[16];
    unsigned int i, r;

    for (i = 0; i < 16; ++i) {
        m[i] = load64(block + i * sizeof(m[i]));
    }

    for (i = 0; i < 8; ++i) {
        v[i] = S->h[i];
    }

    v[8] = blake2b_IV[0];
    v[9] = blake2b_IV[1];
    v[10] = blake2b_IV[2];
    v[11] = blake2b_IV[3];
    v[12] = blake2b_IV[4] ^ S->t[0];
    v[13] = blake2b_IV[5] ^ S->t[1];
    v[14] = blake2b_IV[6] ^ S->f[0];
    v[15] = blake2b_IV[7] ^ S->f[1];

#define G(r, i, a, b, c, d)                                                    \
    # 使用 do-while 循环执行以下操作
    do {                                                                       \
        # 计算 a 的新值
        a = a + b + m[blake2b_sigma[r][2 * i + 0]];                            \
        # 对 d 进行循环右移 32 位
        d = rotr64(d ^ a, 32);                                                 \
        # 计算 c 的新值
        c = c + d;                                                             \
        # 对 b 进行循环右移 24 位
        b = rotr64(b ^ c, 24);                                                 \
        # 计算 a 的新值
        a = a + b + m[blake2b_sigma[r][2 * i + 1]];                            \
        # 对 d 进行循环右移 16 位
        d = rotr64(d ^ a, 16);                                                 \
        # 计算 c 的新值
        c = c + d;                                                             \
        # 对 b 进行循环右移 63 位
        b = rotr64(b ^ c, 63);                                                 \
    } while ((void)0, 0)
#define ROUND(r)                                                               \
    do {                                                                       \
        G(r, 0, v[0], v[4], v[8], v[12]);                                      \
        G(r, 1, v[1], v[5], v[9], v[13]);                                      \
        G(r, 2, v[2], v[6], v[10], v[14]);                                     \
        G(r, 3, v[3], v[7], v[11], v[15]);                                     \
        G(r, 4, v[0], v[5], v[10], v[15]);                                     \
        G(r, 5, v[1], v[6], v[11], v[12]);                                     \
        G(r, 6, v[2], v[7], v[8], v[13]);                                      \
        G(r, 7, v[3], v[4], v[9], v[14]);                                      \
    } while ((void)0, 0)


# 定义一个宏函数ROUND，用于进行一轮的计算
#define ROUND(r)                                                               \
    do {                                                                       \
        # 调用G函数进行计算，参数分别为轮数r和8个变量v的索引
        G(r, 0, v[0], v[4], v[8], v[12]);                                      \
        G(r, 1, v[1], v[5], v[9], v[13]);                                      \
        G(r, 2, v[2], v[6], v[10], v[14]);                                     \
        G(r, 3, v[3], v[7], v[11], v[15]);                                     \
        G(r, 4, v[0], v[5], v[10], v[15]);                                     \
        G(r, 5, v[1], v[6], v[11], v[12]);                                     \
        G(r, 6, v[2], v[7], v[8], v[13]);                                      \
        G(r, 7, v[3], v[4], v[9], v[14]);                                      \
    } while ((void)0, 0)



for (r = 0; r < 12; ++r) {
    ROUND(r);
}


# 循环12次，每次调用ROUND宏函数进行一轮计算
for (r = 0; r < 12; ++r) {
    ROUND(r);
}



for (i = 0; i < 8; ++i) {
    S->h[i] = S->h[i] ^ v[i] ^ v[i + 8];
}


# 循环8次，将S->h[i]与v[i]和v[i+8]进行异或运算，并将结果赋值给S->h[i]
for (i = 0; i < 8; ++i) {
    S->h[i] = S->h[i] ^ v[i] ^ v[i + 8];
}



#undef G
#undef ROUND
}


# 取消之前定义的宏函数G和ROUND
#undef G
#undef ROUND
}



int rx_blake2b_update(blake2b_state *S, const void *in, size_t inlen) {
    const uint8_t *pin = (const uint8_t *)in;

    if (inlen == 0) {
        return 0;
    }

    /* Sanity check */
    if (S == NULL || in == NULL) {
        return -1;
    }

    /* Is this a reused state? */
    if (S->f[0] != 0) {
        return -1;
    }

    if (S->buflen + inlen > BLAKE2B_BLOCKBYTES) {
        /* Complete current block */
        size_t left = S->buflen;
        size_t fill = BLAKE2B_BLOCKBYTES - left;
        memcpy(&S->buf[left], pin, fill);
        blake2b_increment_counter(S, BLAKE2B_BLOCKBYTES);
        rx_blake2b_compress(S, S->buf);
        S->buflen = 0;
        inlen -= fill;
        pin += fill;
        /* Avoid buffer copies when possible */
        while (inlen > BLAKE2B_BLOCKBYTES) {
            blake2b_increment_counter(S, BLAKE2B_BLOCKBYTES);
            rx_blake2b_compress(S, pin);
            inlen -= BLAKE2B_BLOCKBYTES;
            pin += BLAKE2B_BLOCKBYTES;
        }


# 更新blake2b状态
int rx_blake2b_update(blake2b_state *S, const void *in, size_t inlen) {
    # 将输入转换为uint8_t类型的指针
    const uint8_t *pin = (const uint8_t *)in;

    # 如果输入长度为0，直接返回0
    if (inlen == 0) {
        return 0;
    }

    /* Sanity check */
    # 对S和in进行空指针检查，如果有空指针，则返回-1
    if (S == NULL || in == NULL) {
        return -1;
    }

    /* Is this a reused state? */
    # 检查S->f[0]是否为0，如果不为0，则表示状态被重用，返回-1
    if (S->f[0] != 0) {
        return -1;
    }

    # 如果S->buflen + inlen大于BLAKE2B_BLOCKBYTES，则需要处理完当前块
    if (S->buflen + inlen > BLAKE2B_BLOCKBYTES) {
        /* Complete current block */
        # 计算剩余空间和需要填充的空间
        size_t left = S->buflen;
        size_t fill = BLAKE2B_BLOCKBYTES - left;
        # 将输入数据填充到缓冲区中
        memcpy(&S->buf[left], pin, fill);
        # 增加计数器
        blake2b_increment_counter(S, BLAKE2B_BLOCKBYTES);
        # 压缩缓冲区数据
        rx_blake2b_compress(S, S->buf);
        # 重置缓冲区长度
        S->buflen = 0;
        # 更新输入长度和指针
        inlen -= fill;
        pin += fill;
        /* Avoid buffer copies when possible */
        # 当输入长度大于BLAKE2B_BLOCKBYTES时，避免缓冲区拷贝
        while (inlen > BLAKE2B_BLOCKBYTES) {
            # 增加计数器
            blake2b_increment_counter(S, BLAKE2B_BLOCKBYTES);
            # 压缩输入数据
            rx_blake2b_compress(S, pin);
            # 更新输入长度和指针
            inlen -= BLAKE2B_BLOCKBYTES;
            pin += BLAKE2B_BLOCKBYTES;
        }
    // 将输入数据复制到S->buf中，起始位置为S->buflen，长度为inlen
    memcpy(&S->buf[S->buflen], pin, inlen);
    // 更新S->buflen，增加inlen的长度
    S->buflen += (unsigned int)inlen;
    // 返回0，表示成功
    return 0;
}

int rx_blake2b_final(blake2b_state *S, void *out, size_t outlen) {
    uint8_t buffer[BLAKE2B_OUTBYTES] = { 0 };
    unsigned int i;

    /* Sanity checks */
    // 检查参数是否合法
    if (S == NULL || out == NULL || outlen < S->outlen) {
        return -1;
    }

    /* Is this a reused state? */
    // 检查是否是重用的状态
    if (S->f[0] != 0) {
        return -1;
    }

    // 增加计数器
    blake2b_increment_counter(S, S->buflen);
    // 设置最后一个块
    blake2b_set_lastblock(S);
    // 填充剩余的字节
    memset(&S->buf[S->buflen], 0, BLAKE2B_BLOCKBYTES - S->buflen); /* Padding */
    // 压缩数据
    rx_blake2b_compress(S, S->buf);

    // 将哈希值存储到临时缓冲区
    for (i = 0; i < 8; ++i) { /* Output full hash to temp buffer */
        store64(buffer + sizeof(S->h[i]) * i, S->h[i]);
    }

    // 将结果拷贝到输出缓冲区
    memcpy(out, buffer, S->outlen);
    // 清空内部内存
    //clear_internal_memory(buffer, sizeof(buffer));
    //clear_internal_memory(S->buf, sizeof(S->buf));
    //clear_internal_memory(S->h, sizeof(S->h));
    return 0;
}

int rx_blake2b_default(void *out, size_t outlen, const void *in, size_t inlen) {
    blake2b_state S;
    int ret = -1;

    /* Verify parameters */
    // 验证参数
    if (NULL == in && inlen > 0) {
        goto fail;
    }

    if (NULL == out || outlen == 0 || outlen > BLAKE2B_OUTBYTES) {
        goto fail;
    }

    // 初始化状态
    if (rx_blake2b_init(&S, outlen) < 0) {
        goto fail;
    }

    // 更新状态
    if (rx_blake2b_update(&S, in, inlen) < 0) {
        goto fail;
    }
    // 完成状态
    ret = rx_blake2b_final(&S, out, outlen);

fail:
    // 清空内部内存
    //clear_internal_memory(&S, sizeof(S));
    return ret;
}

/* Argon2 Team - Begin Code */
int rxa2_blake2b_long(void *pout, size_t outlen, const void *in, size_t inlen) {
    uint8_t *out = (uint8_t *)pout;
    blake2b_state blake_state;
    uint8_t outlen_bytes[sizeof(uint32_t)] = { 0 };
    int ret = -1;

    if (outlen > UINT32_MAX) {
        goto fail;
    }

    /* Ensure little-endian byte order! */
    // 确保是小端字节序
    store32(outlen_bytes, (uint32_t)outlen);

#define TRY(statement)                                                         \
    # 使用 do-while 循环执行语句，并将结果赋给 ret
    do {                                                                       \
        ret = statement;                                                       \
        # 如果 ret 小于 0，则跳转到 fail 标签处
        if (ret < 0) {                                                         \
            goto fail;                                                         \
        }                                                                      \
    } while ((void)0, 0)

    # 如果输出长度小于等于 BLAKE2B_OUTBYTES
    if (outlen <= BLAKE2B_OUTBYTES) {
        # 尝试初始化 BLAKE2B 状态
        TRY(rx_blake2b_init(&blake_state, outlen));
        # 尝试更新 BLAKE2B 状态，包括输出长度的字节和其大小
        TRY(rx_blake2b_update(&blake_state, outlen_bytes, sizeof(outlen_bytes)));
        # 尝试更新 BLAKE2B 状态，包括输入数据和其长度
        TRY(rx_blake2b_update(&blake_state, in, inlen));
        # 尝试完成 BLAKE2B 计算，并将结果存入 out 中
        TRY(rx_blake2b_final(&blake_state, out, outlen));
    }
    # 如果输出长度大于 BLAKE2B_OUTBYTES
    else {
        # 定义需要生成的字节数
        uint32_t toproduce;
        # 定义输出缓冲区
        uint8_t out_buffer[BLAKE2B_OUTBYTES];
        # 定义输入缓冲区
        uint8_t in_buffer[BLAKE2B_OUTBYTES];
        # 尝试初始化 BLAKE2B 状态，输出长度为 BLAKE2B_OUTBYTES
        TRY(rx_blake2b_init(&blake_state, BLAKE2B_OUTBYTES));
        # 尝试更新 BLAKE2B 状态，包括输出长度的字节和其大小
        TRY(rx_blake2b_update(&blake_state, outlen_bytes, sizeof(outlen_bytes)));
        # 尝试更新 BLAKE2B 状态，包括输入数据和其长度
        TRY(rx_blake2b_update(&blake_state, in, inlen));
        # 尝试完成 BLAKE2B 计算，并将结果存入 out_buffer 中
        TRY(rx_blake2b_final(&blake_state, out_buffer, BLAKE2B_OUTBYTES));
        # 将 out_buffer 的前一半字节复制到 out 中
        memcpy(out, out_buffer, BLAKE2B_OUTBYTES / 2);
        # 将 out 指针移动到后一半位置
        out += BLAKE2B_OUTBYTES / 2;
        # 计算还需要生成的字节数
        toproduce = (uint32_t)outlen - BLAKE2B_OUTBYTES / 2;

        # 当还需要生成的字节数大于 BLAKE2B_OUTBYTES 时，循环执行以下操作
        while (toproduce > BLAKE2B_OUTBYTES) {
            # 将 out_buffer 复制到 in_buffer
            memcpy(in_buffer, out_buffer, BLAKE2B_OUTBYTES);
            # 尝试使用 BLAKE2B 计算，并将结果存入 out_buffer 中
            TRY(rx_blake2b(out_buffer, BLAKE2B_OUTBYTES, in_buffer, BLAKE2B_OUTBYTES));
            # 将 out_buffer 的前一半字节复制到 out 中
            memcpy(out, out_buffer, BLAKE2B_OUTBYTES / 2);
            # 将 out 指针移动到后一半位置
            out += BLAKE2B_OUTBYTES / 2;
            # 更新还需要生成的字节数
            toproduce -= BLAKE2B_OUTBYTES / 2;
        }

        # 将 out_buffer 复制到 in_buffer
        memcpy(in_buffer, out_buffer, BLAKE2B_OUTBYTES);
        # 尝试使用 BLAKE2B 计算，并将结果存入 out_buffer 中
        TRY(rx_blake2b(out_buffer, toproduce, in_buffer, BLAKE2B_OUTBYTES));
        # 将 out_buffer 的前 toproduce 字节复制到 out 中
        memcpy(out, out_buffer, toproduce);
    }
fail:
    // 清除内部内存中的数据，使用blake_state的大小作为参数
    // clear_internal_memory(&blake_state, sizeof(blake_state));
    // 返回ret变量的值
    return ret;
// 取消TRY宏定义
#undef TRY
}
// Argon2 Team - 结束代码
```