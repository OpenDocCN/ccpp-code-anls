# `xmrig\src\crypto\randomx\blake2\avx2\blake2b-common.h`

```cpp
#ifndef BLAKE2_AVX2_BLAKE2B_COMMON_H
#define BLAKE2_AVX2_BLAKE2B_COMMON_H

#include <stddef.h>  // 包含标准库头文件，定义了一些常用的类型和宏
#include <stdint.h>  // 包含标准整数类型定义
#include <string.h>  // 包含字符串处理相关函数的定义

#include <immintrin.h>  // 包含 AVX2 指令集的头文件

#include "blake2.h"  // 包含 blake2.h 头文件

#define LOAD128(p)  _mm_load_si128( (__m128i *)(p) )  // 定义宏，加载 128 位数据
#define STORE128(p,r) _mm_store_si128((__m128i *)(p), r)  // 定义宏，存储 128 位数据

#define LOADU128(p)  _mm_loadu_si128( (__m128i *)(p) )  // 定义宏，非对齐加载 128 位数据
#define STOREU128(p,r) _mm_storeu_si128((__m128i *)(p), r)  // 定义宏，非对齐存储 128 位数据

#define LOAD(p)  _mm256_load_si256( (__m256i *)(p) )  // 定义宏，加载 256 位数据
#define STORE(p,r) _mm256_store_si256((__m256i *)(p), r)  // 定义宏，存储 256 位数据

#define LOADU(p)  _mm256_loadu_si256( (__m256i *)(p) )  // 定义宏，非对齐加载 256 位数据
#define STOREU(p,r) _mm256_storeu_si256((__m256i *)(p), r)  // 定义宏，非对齐存储 256 位数据

static INLINE uint64_t LOADU64(void const * p) {  // 定义函数，非对齐加载 64 位数据
  uint64_t v;
  memcpy(&v, p, sizeof v);
  return v;
}

#define ROTATE16 _mm256_setr_epi8( 2, 3, 4, 5, 6, 7, 0, 1, 10, 11, 12, 13, 14, 15, 8, 9, \
                                   2, 3, 4, 5, 6, 7, 0, 1, 10, 11, 12, 13, 14, 15, 8, 9 )

#define ROTATE24 _mm256_setr_epi8( 3, 4, 5, 6, 7, 0, 1, 2, 11, 12, 13, 14, 15, 8, 9, 10, \
                                   3, 4, 5, 6, 7, 0, 1, 2, 11, 12, 13, 14, 15, 8, 9, 10 )

#define ADD(a, b) _mm256_add_epi64(a, b)  // 定义宏，执行 256 位整数加法
#define SUB(a, b) _mm256_sub_epi64(a, b)  // 定义宏，执行 256 位整数减法

#define XOR(a, b) _mm256_xor_si256(a, b)  // 定义宏，执行 256 位整数异或操作
#define AND(a, b) _mm256_and_si256(a, b)  // 定义宏，执行 256 位整数与操作
#define  OR(a, b) _mm256_or_si256(a, b)   // 定义宏，执行 256 位整数或操作

#define ROT32(x) _mm256_shuffle_epi32((x), _MM_SHUFFLE(2, 3, 0, 1))  // 定义宏，执行 256 位整数循环左移 32 位
#define ROT24(x) _mm256_shuffle_epi8((x), ROTATE24)  // 定义宏，执行 256 位整数循环左移 24 位
#define ROT16(x) _mm256_shuffle_epi8((x), ROTATE16)  // 定义宏，执行 256 位整数循环左移 16 位
#define ROT63(x) _mm256_or_si256(_mm256_srli_epi64((x), 63), ADD((x), (x)))  // 定义宏，执行 256 位整数循环左移 63 位

#endif  // 结束条件编译指令
```