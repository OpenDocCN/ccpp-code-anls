# `xmrig\src\base\crypto\sha3.cpp`

```cpp
/*
 * Works when compiled for either 32-bit or 64-bit targets, optimized for
 * 64 bit.
 *
 * Canonical implementation of Init/Update/Finalize for SHA-3 byte input.
 *
 * SHA3-256, SHA3-384, SHA-512 are implemented. SHA-224 can easily be added.
 *
 * Based on code from http://keccak.noekeon.org/ .
 *
 * I place the code that I wrote into public domain, free to use.
 *
 * I would appreciate if you give credits to this work if you used it to
 * write or test * your code.
 *
 * Aug 2015. Andrey Jivsov. crypto@brainhub.org
 * ---------------------------------------------------------------------- */
 
#include <cstdio>
#include <cstdint>
#include <cstring>

#include "sha3.h"
#include "base/crypto/keccak.h"

#define SHA3_ASSERT( x )
#if defined(_MSC_VER)
#define SHA3_TRACE( format, ...)
#define SHA3_TRACE_BUF( format, buf, l, ...)
#else
#define SHA3_TRACE(format, args...)
#define SHA3_TRACE_BUF(format, buf, l, args...)
#endif

/*
 * This flag is used to configure "pure" Keccak, as opposed to NIST SHA3.
 */
#define SHA3_USE_KECCAK_FLAG 0x80000000
#define SHA3_CW(x) ((x) & (~SHA3_USE_KECCAK_FLAG))

#if defined(_MSC_VER)
#define SHA3_CONST(x) x
#else
#define SHA3_CONST(x) x##L
#endif

#define KECCAK_ROUNDS 24


/* *************************** Public Inteface ************************ */

/* For Init or Reset call these: */
sha3_return_t
sha3_Init(void *priv, unsigned bitSize) {
    sha3_context *ctx = (sha3_context *) priv;
    if( bitSize != 256 && bitSize != 384 && bitSize != 512 )
        return SHA3_RETURN_BAD_PARAMS;
    memset(ctx, 0, sizeof(*ctx));
    ctx->capacityWords = 2 * bitSize / (8 * sizeof(uint64_t));
    return SHA3_RETURN_OK;
}

void
sha3_Init256(void *priv)
{
    sha3_Init(priv, 256);
}

void
sha3_Init384(void *priv)
{
    sha3_Init(priv, 384);
}

void
sha3_Init512(void *priv)
{
    sha3_Init(priv, 512);
}

SHA3_FLAGS
sha3_SetFlags(void *priv, SHA3_FLAGS flags)
{
    # 将私有数据转换为 sha3_context 类型
    sha3_context *ctx = (sha3_context *) priv;
    # 将 flags 转换为 SHA3_FLAGS 类型，并且与 SHA3_FLAGS_KECCAK 进行按位与操作
    flags = static_cast<SHA3_FLAGS>(static_cast<int>(flags) & SHA3_FLAGS_KECCAK);
    # 如果 flags 等于 SHA3_FLAGS_KECCAK，则将 SHA3_USE_KECCAK_FLAG 添加到 ctx->capacityWords 中
    ctx->capacityWords |= (flags == SHA3_FLAGS_KECCAK ? SHA3_USE_KECCAK_FLAG : 0);
    # 返回 flags
    return flags;
}
// 更新 SHA3 上下文的数据
void sha3_Update(void *priv, void const *bufIn, size_t len)
{
    // 将私有指针转换为 SHA3 上下文指针
    sha3_context *ctx = (sha3_context *) priv;

    // 计算需要多少字节才能凑成一个完整的字
    unsigned old_tail = (8 - ctx->byteIndex) & 7;

    size_t words;
    unsigned tail;
    size_t i;

    // 将输入缓冲区转换为 uint8_t 类型的指针
    const uint8_t *buf = reinterpret_cast<const uint8_t*>(bufIn);

    // 跟踪输出调用信息
    SHA3_TRACE_BUF("called to update with:", buf, len);

    // 断言当前字节索引小于 8
    SHA3_ASSERT(ctx->byteIndex < 8);
    // 断言当前单词索引小于上下文中 s 数组的单词数
    SHA3_ASSERT(ctx->wordIndex < sizeof(ctx->s) / sizeof(ctx->s[0]));

    if(len < old_tail) {        // 如果输入长度小于需要凑成一个完整字的字节数
        SHA3_TRACE("because %d<%d, store it and return", (unsigned)len,
                (unsigned)old_tail);
        // 将剩余的字节存储起来并返回
        while (len--)
            ctx->saved |= (uint64_t) (*(buf++)) << ((ctx->byteIndex++) * 8);
        SHA3_ASSERT(ctx->byteIndex < 8);
        return;
    }

    if(old_tail) {              // 如果有剩余的字节可以组成一个完整的字
        SHA3_TRACE("completing one word with %d bytes", (unsigned)old_tail);
        // 完成一个完整的字
        len -= old_tail;
        while (old_tail--)
            ctx->saved |= (uint64_t) (*(buf++)) << ((ctx->byteIndex++) * 8);

        // 将保存的数据添加到 sponge 中
        ctx->s[ctx->wordIndex] ^= ctx->saved;
        SHA3_ASSERT(ctx->byteIndex == 8);
        ctx->byteIndex = 0;
        ctx->saved = 0;
        if(++ctx->wordIndex ==
                (SHA3_KECCAK_SPONGE_WORDS - SHA3_CW(ctx->capacityWords))) {
            xmrig::keccakf(ctx->s, KECCAK_ROUNDS);
            ctx->wordIndex = 0;
        }
    }

    // 直接处理完整的字
    SHA3_ASSERT(ctx->byteIndex == 0);

    words = len / sizeof(uint64_t);
    tail = len - words * sizeof(uint64_t);

    SHA3_TRACE("have %d full words to process", (unsigned)words);
}
    # 遍历从 0 到 words 变量的值
    for(i = 0; i < words; i++, buf += sizeof(uint64_t)) {
        # 将 buf 数组中的 8 个字节按照小端序转换为一个 64 位的整数
        const uint64_t t = (uint64_t) (buf[0]) |
                ((uint64_t) (buf[1]) << 8 * 1) |
                ((uint64_t) (buf[2]) << 8 * 2) |
                ((uint64_t) (buf[3]) << 8 * 3) |
                ((uint64_t) (buf[4]) << 8 * 4) |
                ((uint64_t) (buf[5]) << 8 * 5) |
                ((uint64_t) (buf[6]) << 8 * 6) |
                ((uint64_t) (buf[7]) << 8 * 7);
#if defined(__x86_64__ ) || defined(__i386__)
        // 如果是 x86_64 或者 i386 架构，则进行内存比较
        SHA3_ASSERT(memcmp(&t, buf, 8) == 0);
#endif
        // 对上下文中的状态进行异或操作
        ctx->s[ctx->wordIndex] ^= t;
        // 如果 wordIndex 达到了指定条件，则进行 Keccak 运算
        if(++ctx->wordIndex ==
                (SHA3_KECCAK_SPONGE_WORDS - SHA3_CW(ctx->capacityWords))) {
            xmrig::keccakf(ctx->s, KECCAK_ROUNDS);
            ctx->wordIndex = 0;
        }
    }

    // 输出剩余待处理的字节数
    SHA3_TRACE("have %d bytes left to process, save them", (unsigned)tail);

    /* finally, save the partial word */
    // 断言条件，确保 byteIndex 为 0，tail 小于 8
    SHA3_ASSERT(ctx->byteIndex == 0 && tail < 8);
    // 将剩余的字节保存到上下文中
    while (tail--) {
        SHA3_TRACE("Store byte %02x '%c'", *buf, *buf);
        ctx->saved |= (uint64_t) (*(buf++)) << ((ctx->byteIndex++) * 8);
    }
    SHA3_ASSERT(ctx->byteIndex < 8);
    SHA3_TRACE("Have saved=0x%016" PRIx64 " at the end", ctx->saved);
}

/* This is simply the 'update' with the padding block.
 * The padding block is 0x01 || 0x00* || 0x80. First 0x01 and last 0x80
 * bytes are always present, but they can be the same byte.
 */
void const *
sha3_Finalize(void *priv)
{
    sha3_context *ctx = (sha3_context *) priv;

    SHA3_TRACE("called with %d bytes in the buffer", ctx->byteIndex);

    /* Append 2-bit suffix 01, per SHA-3 spec. Instead of 1 for padding we
     * use 1<<2 below. The 0x02 below corresponds to the suffix 01.
     * Overall, we feed 0, then 1, and finally 1 to start padding. Without
     * M || 01, we would simply use 1 to start padding. */

    uint64_t t;

    if( ctx->capacityWords & SHA3_USE_KECCAK_FLAG ) {
        /* Keccak version */
        // 如果是 Keccak 版本，则进行特定的位运算
        t = (uint64_t)(((uint64_t) 1) << (ctx->byteIndex * 8));
    }
    else {
        /* SHA3 version */
        // 如果是 SHA3 版本，则进行特定的位运算
        t = (uint64_t)(((uint64_t)(0x02 | (1 << 2))) << ((ctx->byteIndex) * 8));
    }

    // 对上下文中的状态进行异或操作
    ctx->s[ctx->wordIndex] ^= ctx->saved ^ t;

    // 对上下文中的状态进行特定的异或操作
    ctx->s[SHA3_KECCAK_SPONGE_WORDS - SHA3_CW(ctx->capacityWords) - 1] ^=
            SHA3_CONST(0x8000000000000000UL);
    // 进行 Keccak 运算
    xmrig::keccakf(ctx->s, KECCAK_ROUNDS);
    # 返回 ctx->s 的前32个字节。对于小端平台，不需要进行转换，可以使用条件编译进行包装
    {
        # 定义循环变量 i
        unsigned i;
        # 遍历 ctx->s 数组
        for(i = 0; i < SHA3_KECCAK_SPONGE_WORDS; i++) {
            # 将 ctx->s[i] 转换为 32 位无符号整数
            const unsigned t1 = (uint32_t) ctx->s[i];
            # 将 ctx->s[i] 右移16位两次，然后转换为 32 位无符号整数
            const unsigned t2 = (uint32_t) ((ctx->s[i] >> 16) >> 16);
            # 将 t1 的每个字节存入 ctx->sb 数组
            ctx->sb[i * 8 + 0] = (uint8_t) (t1);
            ctx->sb[i * 8 + 1] = (uint8_t) (t1 >> 8);
            ctx->sb[i * 8 + 2] = (uint8_t) (t1 >> 16);
            ctx->sb[i * 8 + 3] = (uint8_t) (t1 >> 24);
            # 将 t2 的每个字节存入 ctx->sb 数组
            ctx->sb[i * 8 + 4] = (uint8_t) (t2);
            ctx->sb[i * 8 + 5] = (uint8_t) (t2 >> 8);
            ctx->sb[i * 8 + 6] = (uint8_t) (t2 >> 16);
            ctx->sb[i * 8 + 7] = (uint8_t) (t2 >> 24);
        }
    }
    
    # 输出 ctx->sb 数组的前32个字节的内容
    SHA3_TRACE_BUF("Hash: (first 32 bytes)", ctx->sb, 256 / 8);
    
    # 返回 ctx->sb 数组
    return (ctx->sb);
# 定义一个外部的 C 函数 sha3_HashBuffer，用于计算哈希值
extern "C" sha3_return_t sha3_HashBuffer( unsigned bitSize, enum SHA3_FLAGS flags, const void *in, unsigned inBytes, void *out, unsigned outBytes ) {
    # 定义一个错误码和一个 SHA3 上下文
    sha3_return_t err;
    sha3_context c;

    # 初始化 SHA3 上下文
    err = sha3_Init(&c, bitSize);
    # 如果初始化失败，则返回错误码
    if( err != SHA3_RETURN_OK )
        return err;
    # 设置 SHA3 上下文的标志位
    if( sha3_SetFlags(&c, flags) != flags ) {
        return SHA3_RETURN_BAD_PARAMS;
    }
    # 更新 SHA3 上下文的输入数据
    sha3_Update(&c, in, inBytes);
    # 完成 SHA3 计算，返回哈希值
    const void *h = sha3_Finalize(&c);

    # 如果输出字节数大于哈希位数的字节数，则将输出字节数设置为哈希位数的字节数
    if(outBytes > bitSize/8)
        outBytes = bitSize/8;
    # 将哈希值拷贝到输出缓冲区
    memcpy(out, h, outBytes);
    # 返回计算结果
    return SHA3_RETURN_OK;
}
```