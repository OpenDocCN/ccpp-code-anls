# `xmrig\src\crypto\ghostrider\sph_hamsi.h`

```cpp
/* $Id: sph_hamsi.h 216 2010-06-08 09:46:57Z tp $ */
/**
 * Hamsi interface. This code implements Hamsi with the recommended
 * parameters for SHA-3, with outputs of 224, 256, 384 and 512 bits.
 *
 * ==========================(LICENSE BEGIN)============================
 *
 * Copyright (c) 2007-2010  Projet RNRT SAPHIR
 *
 * Permission is hereby granted, free of charge, to any person obtaining
 * a copy of this software and associated documentation files (the
 * "Software"), to deal in the Software without restriction, including
 * without limitation the rights to use, copy, modify, merge, publish,
 * distribute, sublicense, and/or sell copies of the Software, and to
 * permit persons to whom the Software is furnished to do so, subject to
 * the following conditions:
 *
 * The above copyright notice and this permission notice shall be
 * included in all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 * EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 * MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
 * IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY
 * CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT,
 * TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
 * SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
 *
 * ===========================(LICENSE END)=============================
 *
 * @file     sph_hamsi.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_HAMSI_H__
#define SPH_HAMSI_H__

#include <stddef.h>
#include "sph_types.h"

#ifdef __cplusplus
extern "C"{
#endif

/**
 * Output size (in bits) for Hamsi-224.
 */
#define SPH_SIZE_hamsi224   224

/**
 * Output size (in bits) for Hamsi-256.
 */
#define SPH_SIZE_hamsi256   256

/**
 * Output size (in bits) for Hamsi-384.
 */
#define SPH_SIZE_hamsi384   384

/**
 * Output size (in bits) for Hamsi-512.
 */
/**
 * 定义 Hamsi-512 的大小为 512
 */
#define SPH_SIZE_hamsi512   512

/**
 * 这个结构是 Hamsi-224 和 Hamsi-256 计算的上下文：
 * 它包含了中间值和最后输入块的一些数据。一旦进行了 Hamsi 计算，
 * 上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。一个正在运行的 Hamsi 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char partial[4];  // 保存最后一个输入块的部分数据
    size_t partial_len;  // 最后一个输入块的部分数据的长度
    sph_u32 h[8];  // 中间值
#if SPH_64
    sph_u64 count;  // 计数
#else
    sph_u32 count_high, count_low;  // 计数的高位和低位
#endif
#endif
} sph_hamsi_small_context;

/**
 * 这个结构是 Hamsi-224 计算的上下文。它与常见的 <code>sph_hamsi_small_context</code> 相同。
 */
typedef sph_hamsi_small_context sph_hamsi224_context;

/**
 * 这个结构是 Hamsi-256 计算的上下文。它与常见的 <code>sph_hamsi_small_context</code> 相同。
 */
typedef sph_hamsi_small_context sph_hamsi256_context;

/**
 * 这个结构是 Hamsi-384 和 Hamsi-512 计算的上下文：
 * 它包含了中间值和最后输入块的一些数据。一旦进行了 Hamsi 计算，
 * 上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。一个正在运行的 Hamsi 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char partial[8];  // 保存最后一个输入块的部分数据
    size_t partial_len;  // 最后一个输入块的部分数据的长度
    sph_u32 h[16];  // 中间值
#if SPH_64
    sph_u64 count;  // 计数
#else
    sph_u32 count_high, count_low;  // 计数的高位和低位
#endif
#endif
} sph_hamsi_big_context;

/**
 * 这个结构是 Hamsi-384 计算的上下文。它与常见的 <code>sph_hamsi_small_context</code> 相同。
 */
typedef sph_hamsi_big_context sph_hamsi384_context;
/**
 * 这个结构是 Hamsi-512 计算的上下文。它与常见的 <code>sph_hamsi_small_context</code> 相同。
 */
typedef sph_hamsi_big_context sph_hamsi512_context;

/**
 * 初始化 Hamsi-224 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   Hamsi-224 上下文（指向 <code>sph_hamsi224_context</code> 的指针）
 */
void sph_hamsi224_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Hamsi-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_hamsi224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Hamsi-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。上下文会自动重新初始化。
 *
 * @param cc    Hamsi-224 上下文
 * @param dst   目标缓冲区
 */
void sph_hamsi224_close(void *cc, void *dst);

/**
 * 向当前计算添加一些额外的位（0到7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位是从第 7 位到第 8-n 位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Hamsi-224 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_hamsi224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化 Hamsi-256 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   Hamsi-256 上下文（指向 <code>sph_hamsi256_context</code> 的指针）
 */
void sph_hamsi256_init(void *cc);
/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Hamsi-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_hamsi256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Hamsi-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。上下文会自动重新初始化。
 *
 * @param cc    Hamsi-256 上下文
 * @param dst   目标缓冲区
 */
void sph_hamsi256_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Hamsi-256 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_hamsi256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化 Hamsi-384 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Hamsi-384 上下文（指向 <code>sph_hamsi384_context</code> 的指针）
 */
void sph_hamsi384_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Hamsi-384 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_hamsi384(void *cc, const void *data, size_t len);
/**
 * 终止当前的 Hamsi-384 计算，并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够宽以容纳结果（48 字节）。上下文会自动重新初始化。
 *
 * @param cc    Hamsi-384 上下文
 * @param dst   目标缓冲区
 */
void sph_hamsi384_close(void *cc, void *dst);

/**
 * 添加一些额外位（0 到 7）到当前计算中，然后终止它并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够宽以容纳结果（48 字节）。
 * 如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位
 * （这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Hamsi-384 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_hamsi384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 Hamsi-512 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Hamsi-512 上下文（指向 <code>sph_hamsi512_context</code> 的指针）
 */
void sph_hamsi512_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Hamsi-512 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_hamsi512(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Hamsi-512 计算，并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够宽以容纳结果（64 字节）。上下文会自动重新初始化。
 *
 * @param cc    Hamsi-512 上下文
 * @param dst   目标缓冲区
 */
void sph_hamsi512_close(void *cc, void *dst);
/**
 * 添加一些额外的位（0到7）到当前计算中，然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（64字节）。如果<code>ub</code>中的第i位的值为2^i，则额外的位是从7到8-n编号的（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Hamsi-512上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_hamsi512_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);
```