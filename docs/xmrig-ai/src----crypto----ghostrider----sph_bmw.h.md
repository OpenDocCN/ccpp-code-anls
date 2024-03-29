# `xmrig\src\crypto\ghostrider\sph_bmw.h`

```cpp
/* $Id: sph_bmw.h 216 2010-06-08 09:46:57Z tp $ */
/**
 * BMW interface. BMW (aka "Blue Midnight Wish") is a family of
 * functions which differ by their output size; this implementation
 * defines BMW for output sizes 224, 256, 384 and 512 bits.
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
 * @file     sph_bmw.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_BMW_H__
#define SPH_BMW_H__

#ifdef __cplusplus
extern "C"{
#endif

#include <stddef.h>
#include "sph_types.h"

/**
 * Output size (in bits) for BMW-224.
 */
#define SPH_SIZE_bmw224   224

/**
 * Output size (in bits) for BMW-256.
 */
#define SPH_SIZE_bmw256   256

#if SPH_64

/**
 * Output size (in bits) for BMW-384.
 */
#define SPH_SIZE_bmw384   384
/**
 * Output size (in bits) for BMW-512.
 */
#define SPH_SIZE_bmw512   512

#endif

/**
 * This structure is a context for BMW-224 and BMW-256 computations:
 * it contains the intermediate values and some data from the last
 * entered block. Once a BMW computation has been performed, the
 * context can be reused for another computation.
 *
 * The contents of this structure are private. A running BMW
 * computation can be cloned by copying the context (e.g. with a simple
 * <code>memcpy()</code>).
 */

#if !defined(__AVX2__)

typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[64];    /* first field, for alignment */
    size_t ptr;
    sph_u32 H[16];
#if SPH_64
    sph_u64 bit_count;
#else
    sph_u32 bit_count_high, bit_count_low;
#endif
#endif
} sph_bmw_small_context;

/**
 * This structure is a context for BMW-224 computations. It is
 * identical to the common <code>sph_bmw_small_context</code>.
 */
typedef sph_bmw_small_context sph_bmw224_context;

/**
 * This structure is a context for BMW-256 computations. It is
 * identical to the common <code>sph_bmw_small_context</code>.
 */
typedef sph_bmw_small_context sph_bmw256_context;

#endif // !AVX2

#if SPH_64

/**
 * This structure is a context for BMW-384 and BMW-512 computations:
 * it contains the intermediate values and some data from the last
 * entered block. Once a BMW computation has been performed, the
 * context can be reused for another computation.
 *
 * The contents of this structure are private. A running BMW
 * computation can be cloned by copying the context (e.g. with a simple
 * <code>memcpy()</code>).
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[128];    /* first field, for alignment */
    size_t ptr;
    sph_u64 H[16];
    sph_u64 bit_count;
#endif
} sph_bmw_big_context;

/**
 * This structure is a context for BMW-384 computations. It is
 * identical to the common <code>sph_bmw_small_context</code>.
 */
typedef sph_bmw_big_context sph_bmw384_context;
/**
 * 这个结构是用于 BMW-512 计算的上下文。它与常见的 <code>sph_bmw_small_context</code> 完全相同。
 */
typedef sph_bmw_big_context sph_bmw512_context;

#endif

#if !defined(__AVX2__)

/**
 * 初始化一个 BMW-224 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   BMW-224 上下文 (指向 <code>sph_bmw224_context</code> 的指针)
 */
void sph_bmw224_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     BMW-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_bmw224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 BMW-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。上下文会自动重新初始化。
 *
 * @param cc    BMW-224 上下文
 * @param dst   目标缓冲区
 */
void sph_bmw224_close(void *cc, void *dst);

/**
 * 添加一些额外的位（0 到 7）到当前计算中，然后终止它并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（28字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外的位是从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    BMW-224 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_bmw224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 BMW-256 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   BMW-256 上下文 (指向 <code>sph_bmw256_context</code> 的指针)
 */
void sph_bmw256_init(void *cc);
/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     BMW-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_bmw256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 BMW-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。上下文会自动重新初始化。
 *
 * @param cc    BMW-256 上下文
 * @param dst   目标缓冲区
 */
void sph_bmw256_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外的位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（32 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    BMW-256 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_bmw256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

#endif // !AVX2

#if SPH_64

/**
 * 初始化 BMW-384 上下文。此过程不执行任何内存分配。
 *
 * @param cc   BMW-384 上下文（指向 <code>sph_bmw384_context</code> 的指针）
 */
void sph_bmw384_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     BMW-384 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_bmw384(void *cc, const void *data, size_t len);
/**
 * 终止当前的 BMW-384 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48字节）。上下文会自动重新初始化。
 *
 * @param cc    BMW-384 上下文
 * @param dst   目标缓冲区
 */
void sph_bmw384_close(void *cc, void *dst);

/**
 * 添加一些额外的位（0到7）到当前计算中，然后终止它并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（48字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外的位是从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    BMW-384 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_bmw384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 BMW-512 上下文。此过程不执行任何内存分配。
 *
 * @param cc   BMW-512 上下文（指向 <code>sph_bmw512_context</code> 的指针）
 */
void sph_bmw512_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     BMW-512 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_bmw512(void *cc, const void *data, size_t len);

/**
 * 终止当前的 BMW-512 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64字节）。上下文会自动重新初始化。
 *
 * @param cc    BMW-512 上下文
 * @param dst   目标缓冲区
 */
void sph_bmw512_close(void *cc, void *dst);
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (64 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the BMW-512 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_bmw512_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);
```