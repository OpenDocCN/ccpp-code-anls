# `xmrig\src\crypto\randomx\blake2\avx2\blake2b_avx2.c`

```
#include <stddef.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

#include "blake2.h"
#include "blake2b.h"
#include "blake2b-common.h"

// 定义一个64字节对齐的常量数组，用于初始化 blake2b_IV
ALIGN(64) static const uint64_t blake2b_IV[8] = {
  UINT64_C(0x6A09E667F3BCC908), UINT64_C(0xBB67AE8584CAA73B),
  UINT64_C(0x3C6EF372FE94F82B), UINT64_C(0xA54FF53A5F1D36F1),
  UINT64_C(0x510E527FADE682D1), UINT64_C(0x9B05688C2B3E6C1F),
  UINT64_C(0x1F83D9ABFB41BD6B), UINT64_C(0x5BE0CD19137E2179),
};

// 定义宏，用于执行 BLAKE2B 的 G1 轮函数
#define BLAKE2B_G1_V1(a, b, c, d, m) do {     \
  a = ADD(a, m);                              \
  a = ADD(a, b); d = XOR(d, a); d = ROT32(d); \
  c = ADD(c, d); b = XOR(b, c); b = ROT24(b); \
} while(0)

// 定义宏，用于执行 BLAKE2B 的 G2 轮函数
#define BLAKE2B_G2_V1(a, b, c, d, m) do {     \
  a = ADD(a, m);                              \
  a = ADD(a, b); d = XOR(d, a); d = ROT16(d); \
  c = ADD(c, d); b = XOR(b, c); b = ROT63(b); \
} while(0)

// 定义宏，用于执行 BLAKE2B 的对角置换函数
#define BLAKE2B_DIAG_V1(a, b, c, d) do {                 \
  a = _mm256_permute4x64_epi64(a, _MM_SHUFFLE(2,1,0,3)); \
  d = _mm256_permute4x64_epi64(d, _MM_SHUFFLE(1,0,3,2)); \
  c = _mm256_permute4x64_epi64(c, _MM_SHUFFLE(0,3,2,1)); \
} while(0)

// 定义宏，用于执行 BLAKE2B 的反对角置换函数
#define BLAKE2B_UNDIAG_V1(a, b, c, d) do {               \
  a = _mm256_permute4x64_epi64(a, _MM_SHUFFLE(0,3,2,1)); \
  d = _mm256_permute4x64_epi64(d, _MM_SHUFFLE(1,0,3,2)); \
  c = _mm256_permute4x64_epi64(c, _MM_SHUFFLE(2,1,0,3)); \
} while(0)

#include "blake2b-load-avx2.h"

// 定义宏，用于执行 BLAKE2B 的一轮函数
#define BLAKE2B_ROUND_V1(a, b, c, d, r, m) do { \
  __m256i b0;                                   \
  BLAKE2B_LOAD_MSG_ ##r ##_1(b0);               \
  BLAKE2B_G1_V1(a, b, c, d, b0);                \
  BLAKE2B_LOAD_MSG_ ##r ##_2(b0);               \
  BLAKE2B_G2_V1(a, b, c, d, b0);                \
  BLAKE2B_DIAG_V1(a, b, c, d);                  \
  BLAKE2B_LOAD_MSG_ ##r ##_3(b0);               \
  BLAKE2B_G1_V1(a, b, c, d, b0);                \
  BLAKE2B_LOAD_MSG_ ##r ##_4(b0);               \
  BLAKE2B_G2_V1(a, b, c, d, b0);                \
  BLAKE2B_UNDIAG_V1(a, b, c, d);                \


注释：
} while(0)

这是一个do-while循环的结束标记。


#define BLAKE2B_ROUNDS_V1(a, b, c, d, m) do {   \
  BLAKE2B_ROUND_V1(a, b, c, d,  0, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  1, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  2, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  3, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  4, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  5, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  6, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  7, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  8, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d,  9, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d, 10, (m));        \
  BLAKE2B_ROUND_V1(a, b, c, d, 11, (m));        \
} while(0)

这是一个宏定义，用于执行BLAKE2B算法的一轮操作。


#define DECLARE_MESSAGE_WORDS(m)                                       \
  const __m256i m0 = _mm256_broadcastsi128_si256(LOADU128((m) +   0)); \
  const __m256i m1 = _mm256_broadcastsi128_si256(LOADU128((m) +  16)); \
  const __m256i m2 = _mm256_broadcastsi128_si256(LOADU128((m) +  32)); \
  const __m256i m3 = _mm256_broadcastsi128_si256(LOADU128((m) +  48)); \
  const __m256i m4 = _mm256_broadcastsi128_si256(LOADU128((m) +  64)); \
  const __m256i m5 = _mm256_broadcastsi128_si256(LOADU128((m) +  80)); \
  const __m256i m6 = _mm256_broadcastsi128_si256(LOADU128((m) +  96)); \
  const __m256i m7 = _mm256_broadcastsi128_si256(LOADU128((m) + 112)); \
  __m256i t0, t1;

这是一个宏定义，用于声明并初始化BLAKE2B算法中的消息块。


#define BLAKE2B_COMPRESS_V1(a, b, m, t0, t1, f0, f1) do { \
  DECLARE_MESSAGE_WORDS(m)                                \
  const __m256i iv0 = a;                                  \
  const __m256i iv1 = b;                                  \
  __m256i c = LOAD(&blake2b_IV[0]);                       \
  __m256i d = XOR(                                        \
    LOAD(&blake2b_IV[4]),                                 \

这是一个宏定义，用于执行BLAKE2B算法的压缩函数。
    # 使用给定的参数设置一个 256 位的 AVX 寄存器
    _mm256_set_epi64x(f1, f0, t1, t0)
    # 进行 BLAKE2B 哈希算法的轮迭代
    BLAKE2B_ROUNDS_V1(a, b, c, d, m)
    # 将 a 和 c 进行异或操作
    a = XOR(a, c)
    # 将 b 和 d 进行异或操作
    b = XOR(b, d)
    # 将 a 和 iv0 进行异或操作
    a = XOR(a, iv0)
    # 将 b 和 iv1 进行异或操作
    b = XOR(b, iv1)
} while(0)

int blake2b_avx2(void* out_ptr, size_t outlen, const void* in_ptr, size_t inlen) {
  const __m256i parameter_block = _mm256_set_epi64x(0, 0, 0, 0x01010000UL | (uint32_t)outlen);
  ALIGN(64) uint8_t buffer[BLAKE2B_BLOCKBYTES];
  __m256i a = XOR(LOAD(&blake2b_IV[0]), parameter_block);  // 初始化变量a
  __m256i b = LOAD(&blake2b_IV[4]);  // 初始化变量b
  uint64_t counter = 0;  // 初始化计数器
  const uint8_t* in = (const uint8_t*)in_ptr;  // 将输入指针转换为uint8_t类型
  do {
    const uint64_t flag = (inlen <= BLAKE2B_BLOCKBYTES) ? -1 : 0;  // 根据输入长度设置标志位
    size_t block_size = BLAKE2B_BLOCKBYTES;  // 初始化块大小
    if(inlen < BLAKE2B_BLOCKBYTES) {  // 如果输入长度小于块大小
      memcpy(buffer, in, inlen);  // 将输入复制到缓冲区
      memset(buffer + inlen, 0, BLAKE2B_BLOCKBYTES - inlen);  // 在缓冲区中填充0
      block_size = inlen;  // 更新块大小
      in = buffer;  // 更新输入指针
    }
    counter += block_size;  // 更新计数器
    BLAKE2B_COMPRESS_V1(a, b, in, counter, 0, flag, 0);  // 调用压缩函数
    inlen -= block_size;  // 更新输入长度
    in    += block_size;  // 更新输入指针
  } while(inlen > 0);  // 循环直到输入长度为0

  uint8_t* out = (uint8_t*)out_ptr;  // 将输出指针转换为uint8_t类型

  switch (outlen) {
  case 64:
      STOREU(out + 32, b);  // 存储变量b的值到输出指针的偏移32处
      // Fall through

  case 32:
      STOREU(out, a);  // 存储变量a的值到输出指针
      break;

  default:
      STOREU(buffer, a);  // 存储变量a的值到缓冲区
      STOREU(buffer + 32, b);  // 存储变量b的值到缓冲区的偏移32处
      memcpy(out, buffer, outlen);  // 将缓冲区的内容复制到输出指针
      break;
  }

  _mm256_zeroupper();  // 清除AVX2寄存器状态
  return 0;  // 返回0
}
```