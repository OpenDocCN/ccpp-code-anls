# `xmrig\src\3rdparty\argon2\lib\blake2\blake2.c`

```
// 包含头文件 string.h
#include <string.h>

// 包含自定义的 blake2 相关头文件
#include "blake2/blake2.h"
#include "blake2/blake2-impl.h"

// 包含自定义的 core.h 头文件
#include "core.h"

// 定义 blake2b_IV 数组，包含 8 个 64 位的常量
static const uint64_t blake2b_IV[8] = {
    UINT64_C(0x6a09e667f3bcc908), UINT64_C(0xbb67ae8584caa73b),
    UINT64_C(0x3c6ef372fe94f82b), UINT64_C(0xa54ff53a5f1d36f1),
    UINT64_C(0x510e527fade682d1), UINT64_C(0x9b05688c2b3e6c1f),
    UINT64_C(0x1f83d9abfb41bd6b), UINT64_C(0x5be0cd19137e2179)
};

// 定义 rotr64 宏，实现 64 位数的循环右移
#define rotr64(x, n) (((x) >> (n)) | ((x) << (64 - (n))))

// 定义 blake2b_sigma 数组，包含 12 个 16 个元素的数组
static const unsigned int blake2b_sigma[12][16] = {
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

// 定义 G 宏，实现 blake2b 压缩函数中的 G 函数
#define G(m, r, i, a, b, c, d) \
    do { \
        a = a + b + m[blake2b_sigma[r][2 * i + 0]]; \
        d = rotr64(d ^ a, 32); \
        c = c + d; \
        b = rotr64(b ^ c, 24); \
        a = a + b + m[blake2b_sigma[r][2 * i + 1]]; \
        d = rotr64(d ^ a, 16); \
        c = c + d; \
        b = rotr64(b ^ c, 63); \
    } while ((void)0, 0)

// 定义 ROUND 宏
#define ROUND(m, v, r) \
    # 使用宏 G 对 v 数组中的元素进行一系列操作，每次操作都会更新 m 和 r 的值
    do { \
        # 对 v 数组中的元素进行操作，更新 m 和 r 的值
        G(m, r, 0, v[0], v[4], v[ 8], v[12]); \
        G(m, r, 1, v[1], v[5], v[ 9], v[13]); \
        G(m, r, 2, v[2], v[6], v[10], v[14]); \
        G(m, r, 3, v[3], v[7], v[11], v[15]); \
        G(m, r, 4, v[0], v[5], v[10], v[15]); \
        G(m, r, 5, v[1], v[6], v[11], v[12]); \
        G(m, r, 6, v[2], v[7], v[ 8], v[13]); \
        G(m, r, 7, v[3], v[4], v[ 9], v[14]); \
    } while ((void)0, 0)
// 压缩函数，用于处理一个消息分组
void blake2b_compress(blake2b_state *S, const void *block, uint64_t f0)
{
    // 定义两个64位数组，用于存储消息分组和状态向量
    uint64_t m[16];
    uint64_t v[16];

    // 从消息分组中加载64位数据到m数组中
    m[ 0] = load64((const uint64_t *)block +  0);
    m[ 1] = load64((const uint64_t *)block +  1);
    // ... 依此类推加载消息分组中的数据到m数组中

    // 从状态向量S中加载数据到v数组中
    v[ 0] = S->h[0];
    v[ 1] = S->h[1];
    // ... 依此类推加载状态向量中的数据到v数组中

    // 进行12轮的混合运算
    ROUND(m, v, 0);
    ROUND(m, v, 1);
    // ... 依此类推进行12轮混合运算

    // 更新状态向量S的值
    S->h[0] ^= v[0] ^ v[ 8];
    S->h[1] ^= v[1] ^ v[ 9];
    // ... 依此类推更新状态向量S的值
}

// 用于增加计数器的值
static void blake2b_increment_counter(blake2b_state *S, uint64_t inc)
{
    // 增加t[0]的值
    S->t[0] += inc;
    // 如果t[0]的值小于inc，则增加t[1]的值
    S->t[1] += (S->t[0] < inc);
}

// 用于初始化状态向量S的值
static void blake2b_init_state(blake2b_state *S)
{
    // 初始化状态向量S的值
    // ...
}
    # 将 blake2b_IV 数组的内容复制到 S->h 数组中
    memcpy(S->h, blake2b_IV, sizeof(S->h));
    # 将 S->t[0] 和 S->t[1] 的值都设置为 0
    S->t[1] = S->t[0] = 0;
    # 将 S->buflen 的值设置为 0
    S->buflen = 0;
# 初始化 blake2b 状态
void xmrig_ar2_blake2b_init(blake2b_state *S, size_t outlen)
{
    # 初始化 blake2b 状态
    blake2b_init_state(S);
    # 将初始状态与参数块进行异或操作
    S->h[0] ^= (uint64_t)outlen | (UINT64_C(1) << 16) | (UINT64_C(1) << 24);
}

# 更新 blake2b 状态
void xmrig_ar2_blake2b_update(blake2b_state *S, const void *in, size_t inlen)
{
    const uint8_t *pin = (const uint8_t *)in;

    # 如果当前缓冲区长度加上输入长度大于块大小
    if (S->buflen + inlen > BLAKE2B_BLOCKBYTES) {
        size_t left = S->buflen;
        size_t fill = BLAKE2B_BLOCKBYTES - left;
        # 将输入数据填充到缓冲区
        memcpy(&S->buf[left], pin, fill);
        # 增加计数器
        blake2b_increment_counter(S, BLAKE2B_BLOCKBYTES);
        # 压缩数据
        blake2b_compress(S, S->buf, 0);
        S->buflen = 0;
        inlen -= fill;
        pin += fill;
        # 当输入长度大于块大小时，避免缓冲区拷贝
        while (inlen > BLAKE2B_BLOCKBYTES) {
            blake2b_increment_counter(S, BLAKE2B_BLOCKBYTES);
            blake2b_compress(S, pin, 0);
            inlen -= BLAKE2B_BLOCKBYTES;
            pin += BLAKE2B_BLOCKBYTES;
        }
    }
    # 将剩余的输入数据拷贝到缓冲区
    memcpy(&S->buf[S->buflen], pin, inlen);
    S->buflen += inlen;
}

# 完成 blake2b 计算
void xmrig_ar2_blake2b_final(blake2b_state *S, void *out, size_t outlen)
{
    uint8_t buffer[BLAKE2B_OUTBYTES] = {0};
    unsigned int i;

    # 增加计数器
    blake2b_increment_counter(S, S->buflen);
    # 填充缓冲区
    memset(&S->buf[S->buflen], 0, BLAKE2B_BLOCKBYTES - S->buflen); /* Padding */
    # 压缩数据
    blake2b_compress(S, S->buf, UINT64_C(0xFFFFFFFFFFFFFFFF));

    # 将完整的哈希输出到临时缓冲区
    for (i = 0; i < 8; ++i) {
        store64(buffer + i * sizeof(uint64_t), S->h[i]);
    }

    # 将结果拷贝到输出缓冲区
    memcpy(out, buffer, outlen);
    # 清空内部内存
    xmrig_ar2_clear_internal_memory(buffer, sizeof(buffer));
    xmrig_ar2_clear_internal_memory(S->buf, sizeof(S->buf));
    xmrig_ar2_clear_internal_memory(S->h, sizeof(S->h));
}

# 计算长输入的 blake2b 哈希
void xmrig_ar2_blake2b_long(void *out, size_t outlen, const void *in, size_t inlen)
{
    uint8_t *pout = (uint8_t *)out;
    blake2b_state blake_state;
    uint8_t outlen_bytes[sizeof(uint32_t)] = {0};
    # 将输出长度转换为字节数组
    store32(outlen_bytes, (uint32_t)outlen);
}
    # 如果输出长度小于等于 BLAKE2B_OUTBYTES
    if (outlen <= BLAKE2B_OUTBYTES) {
        # 初始化 BLAKE2B 状态
        xmrig_ar2_blake2b_init(&blake_state, outlen);
        # 更新 BLAKE2B 状态，包括输出长度的字节表示和输入数据
        xmrig_ar2_blake2b_update(&blake_state, outlen_bytes, sizeof(outlen_bytes));
        xmrig_ar2_blake2b_update(&blake_state, in, inlen);
        # 完成 BLAKE2B 计算，将结果存储到 pout 中
        xmrig_ar2_blake2b_final(&blake_state, pout, outlen);
    } else {
        # 如果输出长度大于 BLAKE2B_OUTBYTES
        uint32_t toproduce;
        uint8_t out_buffer[BLAKE2B_OUTBYTES];

        # 初始化 BLAKE2B 状态，输出长度为 BLAKE2B_OUTBYTES
        xmrig_ar2_blake2b_init(&blake_state, BLAKE2B_OUTBYTES);
        # 更新 BLAKE2B 状态，包括输出长度的字节表示和输入数据
        xmrig_ar2_blake2b_update(&blake_state, outlen_bytes, sizeof(outlen_bytes));
        xmrig_ar2_blake2b_update(&blake_state, in, inlen);
        # 完成 BLAKE2B 计算，将结果存储到 out_buffer 中
        xmrig_ar2_blake2b_final(&blake_state, out_buffer, BLAKE2B_OUTBYTES);

        # 将 out_buffer 的前一半字节复制到 pout 中
        memcpy(pout, out_buffer, BLAKE2B_OUTBYTES / 2);
        # 更新 pout 指针位置
        pout += BLAKE2B_OUTBYTES / 2;
        # 计算剩余需要产生的字节数
        toproduce = (uint32_t)outlen - BLAKE2B_OUTBYTES / 2;

        # 循环生成剩余的字节数据
        while (toproduce > BLAKE2B_OUTBYTES) {
            # 初始化 BLAKE2B 状态，输出长度为 BLAKE2B_OUTBYTES
            xmrig_ar2_blake2b_init(&blake_state, BLAKE2B_OUTBYTES);
            # 更新 BLAKE2B 状态，输入为 out_buffer 中的数据
            xmrig_ar2_blake2b_update(&blake_state, out_buffer, BLAKE2B_OUTBYTES);
            # 完成 BLAKE2B 计算，将结果存储到 out_buffer 中
            xmrig_ar2_blake2b_final(&blake_state, out_buffer, BLAKE2B_OUTBYTES);

            # 将 out_buffer 的前一半字节复制到 pout 中
            memcpy(pout, out_buffer, BLAKE2B_OUTBYTES / 2);
            # 更新 pout 指针位置
            pout += BLAKE2B_OUTBYTES / 2;
            # 更新剩余需要产生的字节数
            toproduce -= BLAKE2B_OUTBYTES / 2;
        }

        # 初始化 BLAKE2B 状态，输出长度为剩余需要产生的字节数
        xmrig_ar2_blake2b_init(&blake_state, toproduce);
        # 更新 BLAKE2B 状态，输入为 out_buffer 中的数据
        xmrig_ar2_blake2b_update(&blake_state, out_buffer, BLAKE2B_OUTBYTES);
        # 完成 BLAKE2B 计算，将结果存储到 out_buffer 中
        xmrig_ar2_blake2b_final(&blake_state, out_buffer, toproduce);

        # 将 out_buffer 中的数据复制到 pout 中
        memcpy(pout, out_buffer, toproduce);

        # 清除 out_buffer 的内存内容
        xmrig_ar2_clear_internal_memory(out_buffer, sizeof(out_buffer));
    }
# 闭合前面的函数定义
```