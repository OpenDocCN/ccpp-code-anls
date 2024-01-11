# `xmrig\src\3rdparty\argon2\lib\blake2\blake2-impl.h`

```
#ifndef ARGON2_BLAKE2_IMPL_H
#define ARGON2_BLAKE2_IMPL_H

#include <stdint.h>

/* Argon2 Team - Begin Code */
/*
   Not an exhaustive list, but should cover the majority of modern platforms
   Additionally, the code will always be correct---this is only a performance
   tweak.
*/
#if (defined(__BYTE_ORDER__) &&                                                \
     (__BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__)) ||                           \
    defined(__LITTLE_ENDIAN__) || defined(__ARMEL__) || defined(__MIPSEL__) || \
    defined(__AARCH64EL__) || defined(__amd64__) || defined(__i386__) ||       \
    defined(_M_IX86) || defined(_M_X64) || defined(_M_AMD64) ||                \
    defined(_M_ARM)
#define NATIVE_LITTLE_ENDIAN
#endif
/* Argon2 Team - End Code */

static inline uint32_t load32(const void *src) {
#if defined(NATIVE_LITTLE_ENDIAN)
    return *(const uint32_t *)src;  // 如果是小端字节序，直接返回指针指向的 32 位整数
#else
    const uint8_t *p = (const uint8_t *)src;  // 否则将指针转换为指向 8 位整数的指针
    uint32_t w = *p++;  // 读取第一个字节
    w |= (uint32_t)(*p++) << 8;  // 读取第二个字节并左移 8 位，然后与第一个字节进行或操作
    w |= (uint32_t)(*p++) << 16;  // 读取第三个字节并左移 16 位，然后与前面的结果进行或操作
    w |= (uint32_t)(*p++) << 24;  // 读取第四个字节并左移 24 位，然后与前面的结果进行或操作
    return w;  // 返回结果
#endif
}

static inline uint64_t load64(const void *src) {
#if defined(NATIVE_LITTLE_ENDIAN)
    return *(const uint64_t *)src;  // 如果是小端字节序，直接返回指针指向的 64 位整数
#else
    const uint8_t *p = (const uint8_t *)src;  // 否则将指针转换为指向 8 位整数的指针
    uint64_t w = *p++;  // 读取第一个字节
    w |= (uint64_t)(*p++) << 8;  // 读取第二个字节并左移 8 位，然后与第一个字节进行或操作
    w |= (uint64_t)(*p++) << 16;  // 读取第三个字节并左移 16 位，然后与前面的结果进行或操作
    w |= (uint64_t)(*p++) << 24;  // 读取第四个字节并左移 24 位，然后与前面的结果进行或操作
    w |= (uint64_t)(*p++) << 32;  // 读取第五个字节并左移 32 位，然后与前面的结果进行或操作
    w |= (uint64_t)(*p++) << 40;  // 读取第六个字节并左移 40 位，然后与前面的结果进行或操作
    w |= (uint64_t)(*p++) << 48;  // 读取第七个字节并左移 48 位，然后与前面的结果进行或操作
    w |= (uint64_t)(*p++) << 56;  // 读取第八个字节并左移 56 位，然后与前面的结果进行或操作
    return w;  // 返回结果
#endif
}

static inline void store32(void *dst, uint32_t w) {
#if defined(NATIVE_LITTLE_ENDIAN)
    *(uint32_t *)dst = w;  // 如果是小端字节序，直接将 32 位整数写入目标地址
#else
    uint8_t *p = (uint8_t *)dst;  // 否则将目标地址转换为指向 8 位整数的指针
    *p++ = (uint8_t)w;  // 将 32 位整数的第一个字节写入目标地址
    w >>= 8;  // 右移 8 位
    *p++ = (uint8_t)w;  // 将 32 位整数的第二个字节写入目标地址
    w >>= 8;  // 右移 8 位
    *p++ = (uint8_t)w;  // 将 32 位整数的第三个字节写入目标地址
    w >>= 8;  // 右移 8 位
    *p++ = (uint8_t)w;  // 将 32 位整数的第四个字节写入目标地址
#endif
}

static inline void store64(void *dst, uint64_t w) {
#if defined(NATIVE_LITTLE_ENDIAN)
    *(uint64_t *)dst = w;  // 如果是小端字节序，直接将 64 位整数写入目标地址
#else
    uint8_t *p = (uint8_t *)dst;  // 否则将目标地址转换为指向 8 位整数的指针
    *p++ = (uint8_t)w;  // 将 64 位整数的第一个字节写入目标地址
    # 将变量 w 右移 8 位
    w >>= 8;
    # 将 w 强制转换为 8 位无符号整数，并存入指针 p 指向的位置，然后指针 p 向后移动
    *p++ = (uint8_t)w;
    # 重复以上操作，直到将 w 的所有字节都存入指针 p 指向的位置
#endif
}
#endif // ARGON2_BLAKE2_IMPL_H
```