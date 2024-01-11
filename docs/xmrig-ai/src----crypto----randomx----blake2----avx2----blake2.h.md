# `xmrig\src\crypto\randomx\blake2\avx2\blake2.h`

```
#ifndef BLAKE2_AVX2_BLAKE2_H
#define BLAKE2_AVX2_BLAKE2_H

#if !defined(__cplusplus) && (!defined(__STDC_VERSION__) || __STDC_VERSION__ < 199901L)
  #if   defined(_MSC_VER)
    #define INLINE __inline
  #elif defined(__GNUC__)
    #define INLINE __inline__
  #else
    #define INLINE
  #endif
#else
  #define INLINE inline
#endif

#if defined(_MSC_VER)
#define ALIGN(x) __declspec(align(x))
#else
#define ALIGN(x) __attribute__((aligned(x)))
#endif

enum blake2s_constant {
  BLAKE2S_BLOCKBYTES = 64,  // 定义 blake2s 块大小为 64 字节
  BLAKE2S_OUTBYTES   = 32,  // 定义 blake2s 输出大小为 32 字节
  BLAKE2S_KEYBYTES   = 32,  // 定义 blake2s 密钥大小为 32 字节
  BLAKE2S_SALTBYTES  = 8,   // 定义 blake2s 盐值大小为 8 字节
  BLAKE2S_PERSONALBYTES = 8 // 定义 blake2s 个人化参数大小为 8 字节
};

enum blake2b_constant {
  BLAKE2B_BLOCKBYTES = 128,  // 定义 blake2b 块大小为 128 字节
  BLAKE2B_OUTBYTES   = 64,   // 定义 blake2b 输出大小为 64 字节
  BLAKE2B_KEYBYTES   = 64,   // 定义 blake2b 密钥大小为 64 字节
  BLAKE2B_SALTBYTES  = 16,   // 定义 blake2b 盐值大小为 16 字节
  BLAKE2B_PERSONALBYTES = 16 // 定义 blake2b 个人化参数大小为 16 字节
};

#endif
```