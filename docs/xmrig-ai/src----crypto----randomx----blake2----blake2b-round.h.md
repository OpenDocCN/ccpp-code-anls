# `xmrig\src\crypto\randomx\blake2\blake2b-round.h`

```
/*
   BLAKE2 reference source code package - optimized C implementations

   Copyright 2012, Samuel Neves <sneves@dei.uc.pt>.  You may use this under the
   terms of the CC0, the OpenSSL Licence, or the Apache Public License 2.0, at
   your option.  The terms of these licenses can be found at:

   - CC0 1.0 Universal : http://creativecommons.org/publicdomain/zero/1.0
   - OpenSSL license   : https://www.openssl.org/source/license.html
   - Apache 2.0        : http://www.apache.org/licenses/LICENSE-2.0

   More information about the BLAKE2 hash function can be found at
   https://blake2.net.
*/
#ifndef BLAKE2B_ROUND_H
#define BLAKE2B_ROUND_H

#define LOADU(p)  _mm_loadu_si128( (const __m128i *)(p) )  // 定义宏，加载未对齐的内存地址中的 128 位数据
#define STOREU(p,r) _mm_storeu_si128((__m128i *)(p), r)  // 定义宏，将 128 位数据存储到未对齐的内存地址中

#define TOF(reg) _mm_castsi128_ps((reg))  // 定义宏，将 128 位整数寄存器转换为 128 位浮点数寄存器
#define TOI(reg) _mm_castps_si128((reg))  // 定义宏，将 128 位浮点数寄存器转换为 128 位整数寄存器

#define LIKELY(x) __builtin_expect((x),1)  // 定义宏，指示编译器 x 表达式的可能性

/* Microarchitecture-specific macros */
#define _mm_roti_epi64(x, c) \  // 定义宏，对 128 位整数寄存器进行循环左移或循环右移
    (-(c) == 32) ? _mm_shuffle_epi32((x), _MM_SHUFFLE(2,3,0,1))  \
    : (-(c) == 24) ? _mm_shuffle_epi8((x), r24) \
    : (-(c) == 16) ? _mm_shuffle_epi8((x), r16) \
    : (-(c) == 63) ? _mm_xor_si128(_mm_srli_epi64((x), -(c)), _mm_add_epi64((x), (x)))  \
    : _mm_xor_si128(_mm_srli_epi64((x), -(c)), _mm_slli_epi64((x), 64-(-(c)))

#define G1(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1) \  // 定义宏，执行 BLAKE2B 压缩函数中的 G 函数
  row1l = _mm_add_epi64(_mm_add_epi64(row1l, b0), row2l); \  // 计算 row1l
  row1h = _mm_add_epi64(_mm_add_epi64(row1h, b1), row2h); \  // 计算 row1h
  \
  row4l = _mm_xor_si128(row4l, row1l); \  // 计算 row4l
  row4h = _mm_xor_si128(row4h, row1h); \  // 计算 row4h
  \
  row4l = _mm_roti_epi64(row4l, -32); \  // 对 row4l 进行循环左移
  row4h = _mm_roti_epi64(row4h, -32); \  // 对 row4h 进行循环左移
  \
  row3l = _mm_add_epi64(row3l, row4l); \  // 计算 row3l
  row3h = _mm_add_epi64(row3h, row4h); \  // 计算 row3h
  \
  row2l = _mm_xor_si128(row2l, row3l); \  // 计算 row2l
  row2h = _mm_xor_si128(row2h, row3h); \  // 计算 row2h
  \
  row2l = _mm_roti_epi64(row2l, -24); \  // 对 row2l 进行循环左移
  row2h = _mm_roti_epi64(row2h, -24); \  // 对 row2h 进行循环左移
# 定义宏，对输入的参数进行一系列操作
#define G2(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1) \
  # 对row1l和row2l进行加法操作，并将结果存储在row1l中
  row1l = _mm_add_epi64(_mm_add_epi64(row1l, b0), row2l); \
  # 对row1h和row2h进行加法操作，并将结果存储在row1h中
  row1h = _mm_add_epi64(_mm_add_epi64(row1h, b1), row2h); \
  \
  # 对row4l和row1l进行异或操作，并将结果存储在row4l中
  row4l = _mm_xor_si128(row4l, row1l); \
  # 对row4h和row1h进行异或操作，并将结果存储在row4h中
  row4h = _mm_xor_si128(row4h, row1h); \
  \
  # 对row4l进行循环左移16位
  row4l = _mm_roti_epi64(row4l, -16); \
  # 对row4h进行循环左移16位
  row4h = _mm_roti_epi64(row4h, -16); \
  \
  # 对row3l和row4l进行加法操作，并将结果存储在row3l中
  row3l = _mm_add_epi64(row3l, row4l); \
  # 对row3h和row4h进行加法操作，并将结果存储在row3h中
  row3h = _mm_add_epi64(row3h, row4h); \
  \
  # 对row2l和row3l进行异或操作，并将结果存储在row2l中
  row2l = _mm_xor_si128(row2l, row3l); \
  # 对row2h和row3h进行异或操作，并将结果存储在row2h中
  row2h = _mm_xor_si128(row2h, row3h); \
  \
  # 对row2l进行循环左移63位
  row2l = _mm_roti_epi64(row2l, -63); \
  # 对row2h进行循环左移63位
  row2h = _mm_roti_epi64(row2h, -63); \

# 定义宏，对输入的参数进行一系列操作
#define DIAGONALIZE(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h) \
  # 将row2h和row2l按字节对齐，存储在t0中
  t0 = _mm_alignr_epi8(row2h, row2l, 8); \
  # 将row2l和row2h按字节对齐，存储在t1中
  t1 = _mm_alignr_epi8(row2l, row2h, 8); \
  # 将t0赋值给row2l
  row2l = t0; \
  # 将t1赋值给row2h
  row2h = t1; \
  \
  # 将row3l赋值给t0
  t0 = row3l; \
  # 将row3h赋值给row3l
  row3l = row3h; \
  # 将t0赋值给row3h
  row3h = t0;    \
  \
  # 将row4h和row4l按字节对齐，存储在t0中
  t0 = _mm_alignr_epi8(row4h, row4l, 8); \
  # 将row4l和row4h按字节对齐，存储在t1中
  t1 = _mm_alignr_epi8(row4l, row4h, 8); \
  # 将t1赋值给row4l
  row4l = t1; \
  # 将t0赋值给row4h;

# 定义宏，对输入的参数进行一系列操作
#define UNDIAGONALIZE(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h) \
  # 将row2l和row2h按字节对齐，存储在t0中
  t0 = _mm_alignr_epi8(row2l, row2h, 8); \
  # 将row2h和row2l按字节对齐，存储在t1中
  t1 = _mm_alignr_epi8(row2h, row2l, 8); \
  # 将t0赋值给row2l
  row2l = t0; \
  # 将t1赋值给row2h
  row2h = t1; \
  \
  # 将row3l赋值给t0
  t0 = row3l; \
  # 将row3h赋值给row3l
  row3l = row3h; \
  # 将t0赋值给row3h
  row3h = t0; \
  \
  # 将row4l和row4h按字节对齐，存储在t0中
  t0 = _mm_alignr_epi8(row4l, row4h, 8); \
  # 将row4h和row4l按字节对齐，存储在t1中
  t1 = _mm_alignr_epi8(row4h, row4l, 8); \
  # 将t1赋值给row4l
  row4l = t1; \
  # 将t0赋值给row4h;

# 定义宏，对输入的参数进行一系列操作
#define LOAD_MSG(r, i, b0, b1) \
do { \
  # 将m数组中的元素按照指定规则赋值给b0
  b0 = _mm_set_epi64x(m[blake2b_sigma_sse41[r][i * 4 + 1]], m[blake2b_sigma_sse41[r][i * 4 + 0]]); \
  # 将m数组中的元素按照指定规则赋值给b1
  b1 = _mm_set_epi64x(m[blake2b_sigma_sse41[r][i * 4 + 3]], m[blake2b_sigma_sse41[r][i * 4 + 2]]); \
} while(0)
#define ROUND(r) \  # 定义一个名为ROUND的宏，参数为r
  LOAD_MSG(r, 0, b0, b1); \  # 调用LOAD_MSG函数，参数为r, 0, b0, b1
  G1(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1); \  # 调用G1函数，参数为row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1
  LOAD_MSG(r, 1, b0, b1); \  # 调用LOAD_MSG函数，参数为r, 1, b0, b1
  G2(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1); \  # 调用G2函数，参数为row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1
  DIAGONALIZE(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h); \  # 调用DIAGONALIZE函数，参数为row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h
  LOAD_MSG(r, 2, b0, b1); \  # 调用LOAD_MSG函数，参数为r, 2, b0, b1
  G1(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1); \  # 调用G1函数，参数为row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1
  LOAD_MSG(r, 3, b0, b1); \  # 调用LOAD_MSG函数，参数为r, 3, b0, b1
  G2(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1); \  # 调用G2函数，参数为row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h,b0,b1
  UNDIAGONALIZE(row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h);  # 调用UNDIAGONALIZE函数，参数为row1l,row2l,row3l,row4l,row1h,row2h,row3h,row4h
#endif  # 结束宏定义
```