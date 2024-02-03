# `xmrig\src\crypto\randomx\blake2\endian.h`

```cpp
#pragma once
#include <stdint.h>
#include <string.h>

#if defined(_MSC_VER)
#define FORCE_INLINE __forceinline
#elif defined(__GNUC__)
#define FORCE_INLINE __attribute__((always_inline)) inline
#elif defined(__clang__)
#define FORCE_INLINE __inline__
#else
#define FORCE_INLINE
#endif

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

// 定义一个宏，用于强制内联函数
static FORCE_INLINE uint32_t load32(const void *src) {
#if defined(NATIVE_LITTLE_ENDIAN)
    // 如果是小端字节序，直接将内存中的数据拷贝到一个32位整数中并返回
    uint32_t w;
    memcpy(&w, src, sizeof w);
    return w;
#else
    // 如果不是小端字节序，按照大端字节序将内存中的数据拼接成32位整数并返回
    const uint8_t *p = (const uint8_t *)src;
    uint32_t w = *p++;
    w |= (uint32_t)(*p++) << 8;
    w |= (uint32_t)(*p++) << 16;
    w |= (uint32_t)(*p++) << 24;
    return w;
#endif
}

// 定义一个宏，用于强制内联函数
static FORCE_INLINE uint64_t load64_native(const void *src) {
    // 将内存中的数据拷贝到一个64位整数中并返回
    uint64_t w;
    memcpy(&w, src, sizeof w);
    return w;
}

// 定义一个宏，用于强制内联函数
static FORCE_INLINE uint64_t load64(const void *src) {
#if defined(NATIVE_LITTLE_ENDIAN)
    // 如果是小端字节序，直接将内存中的数据拷贝到一个64位整数中并返回
    return load64_native(src);
#else
    // 如果不是小端字节序，按照大端字节序将内存中的数据拼接成64位整数并返回
    const uint8_t *p = (const uint8_t *)src;
    uint64_t w = *p++;
    w |= (uint64_t)(*p++) << 8;
    w |= (uint64_t)(*p++) << 16;
    w |= (uint64_t)(*p++) << 24;
    w |= (uint64_t)(*p++) << 32;
    w |= (uint64_t)(*p++) << 40;
    w |= (uint64_t)(*p++) << 48;
    w |= (uint64_t)(*p++) << 56;
    return w;
#endif
}

// 定义一个宏，用于强制内联函数
static FORCE_INLINE void store32(void *dst, uint32_t w) {
#if defined(NATIVE_LITTLE_ENDIAN)
    // 如果是小端字节序，直接将32位整数的值拷贝到目标内存中
    memcpy(dst, &w, sizeof w);
#else
    # 将变量w的值复制到dst指向的内存地址中，复制的字节数为w的大小。
# 如果是小端序，直接将64位整数拷贝到目标地址
static FORCE_INLINE void store64_native(void *dst, uint64_t w) {
    memcpy(dst, &w, sizeof w);
}

# 如果不是小端序，将64位整数按字节逐个存储到目标地址
static FORCE_INLINE void store64(void *dst, uint64_t w) {
#if defined(NATIVE_LITTLE_ENDIAN)
    # 调用store64_native函数将64位整数拷贝到目标地址
    store64_native(dst, w);
#else
    # 将目标地址转换为指向8位无符号整数的指针
    uint8_t *p = (uint8_t *)dst;
    # 将64位整数的低8位存储到目标地址，并将整数右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将64位整数的低16位存储到目标地址，并将整数右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将64位整数的低24位存储到目标地址，并将整数右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将64位整数的低32位存储到目标地址，并将整数右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将64位整数的低40位存储到目标地址，并将整数右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将64位整数的低48位存储到目标地址，并将整数右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将64位整数的低56位存储到目标地址，并将整数右移8位
    *p++ = (uint8_t)w;
    w >>= 8;
    # 将64位整数的低64位存储到目标地址
    *p++ = (uint8_t)w;
#endif
}
```