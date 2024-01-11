# `xmrig\src\crypto\ghostrider\sph_shabal.h`

```
/* $Id: sph_shabal.h 175 2010-05-07 16:03:20Z tp $ */
/**
 * Shabal interface. Shabal is a family of functions which differ by
 * their output size; this implementation defines Shabal for output
 * sizes 192, 224, 256, 384 and 512 bits.
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
 * @file     sph_shabal.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_SHABAL_H__
#define SPH_SHABAL_H__

#include "sph_types.h"
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

/**
 * Output size (in bits) for Shabal-192.
 */
#define SPH_SIZE_shabal192 192

/**
 * Output size (in bits) for Shabal-224.
 */
#define SPH_SIZE_shabal224 224

/**
 * Output size (in bits) for Shabal-256.
 */
#define SPH_SIZE_shabal256 256
/**
 * 定义 Shabal-384 的输出大小（以位为单位）。
 */
#define SPH_SIZE_shabal384 384

/**
 * 定义 Shabal-512 的输出大小（以位为单位）。
 */
#define SPH_SIZE_shabal512 512

/**
 * 这个结构体是 Shabal 计算的上下文：它包含了中间值和最后一个输入块的一些数据。
 * 一旦 Shabal 计算完成，这个上下文可以被重用于另一个计算。
 *
 * 这个结构体的内容是私有的。可以通过复制上下文（例如使用简单的 memcpy()）来克隆正在运行的 Shabal 计算。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
  unsigned char buf[64]; /* 第一个字段，用于对齐 */
  size_t ptr;
  sph_u32 A[12], B[16], C[16];
  sph_u32 Whigh, Wlow;
#endif
} sph_shabal_context;

/**
 * Shabal-192 上下文的类型（与通用上下文相同）。
 */
typedef sph_shabal_context sph_shabal192_context;

/**
 * Shabal-224 上下文的类型（与通用上下文相同）。
 */
typedef sph_shabal_context sph_shabal224_context;

/**
 * Shabal-256 上下文的类型（与通用上下文相同）。
 */
typedef sph_shabal_context sph_shabal256_context;

/**
 * Shabal-384 上下文的类型（与通用上下文相同）。
 */
typedef sph_shabal_context sph_shabal384_context;

/**
 * Shabal-512 上下文的类型（与通用上下文相同）。
 */
typedef sph_shabal_context sph_shabal512_context;

/**
 * 初始化 Shabal-192 上下文。这个过程不进行内存分配。
 *
 * @param cc   Shabal-192 上下文（指向 sph_shabal192_context 的指针）
 */
void sph_shabal192_init(void *cc);

/**
 * 处理一些数据字节。允许 len 为零（此时此函数不执行任何操作）。
 *
 * @param cc     Shabal-192 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_shabal192(void *cc, const void *data, size_t len);
/**
 * 终止当前的 Shabal-192 计算，并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够宽以容纳结果（24 字节）。上下文会自动重新初始化。
 *
 * @param cc    Shabal-192 上下文
 * @param dst   目标缓冲区
 */
void sph_shabal192_close(void *cc, void *dst);

/**
 * 添加一些额外的位（0 到 7）到当前计算中，然后终止它并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够宽以容纳结果（24 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，
 * 那么额外的位是从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Shabal-192 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_shabal192_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 Shabal-224 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Shabal-224 上下文（指向 <code>sph_shabal224_context</code> 的指针）
 */
void sph_shabal224_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Shabal-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_shabal224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Shabal-224 计算，并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够宽以容纳结果（28 字节）。上下文会自动重新初始化。
 *
 * @param cc    Shabal-224 上下文
 * @param dst   目标缓冲区
 */
void sph_shabal224_close(void *cc, void *dst);
/**
 * 向当前计算添加一些额外的位（0到7），然后终止计算并将结果输出到提供的缓冲区中，
 * 缓冲区必须足够大以容纳结果（28字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位为从7到8-n的位（这是字节级别的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Shabal-224上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_shabal224_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                     void *dst);

/**
 * 初始化Shabal-256上下文。此过程不执行任何内存分配。
 *
 * @param cc   Shabal-256上下文（指向<code>sph_shabal256_context</code>的指针）
 */
void sph_shabal256_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Shabal-256上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_shabal256(void *cc, const void *data, size_t len);

/**
 * 终止当前的Shabal-256计算并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（32字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Shabal-256上下文
 * @param dst   目标缓冲区
 */
void sph_shabal256_close(void *cc, void *dst);
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (32 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the Shabal-256 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_shabal256_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                     void *dst);

/**
 * Initialize a Shabal-384 context. This process performs no memory allocation.
 *
 * @param cc   the Shabal-384 context (pointer to a
 *             <code>sph_shabal384_context</code>)
 */
void sph_shabal384_init(void *cc);

/**
 * Process some data bytes. It is acceptable that <code>len</code> is zero
 * (in which case this function does nothing).
 *
 * @param cc     the Shabal-384 context
 * @param data   the input data
 * @param len    the input data length (in bytes)
 */
void sph_shabal384(void *cc, const void *data, size_t len);

/**
 * Terminate the current Shabal-384 computation and output the result into
 * the provided buffer. The destination buffer must be wide enough to
 * accomodate the result (48 bytes). The context is automatically
 * reinitialized.
 *
 * @param cc    the Shabal-384 context
 * @param dst   the destination buffer
 */
void sph_shabal384_close(void *cc, void *dst);
*/
/**
 * 向当前计算添加一些额外的位（0到7），然后终止计算并将结果输出到提供的缓冲区中，
 * 缓冲区必须足够大以容纳结果（48字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位为从7到8-n的位（这是字节级别的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Shabal-384上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_shabal384_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                     void *dst);

/**
 * 初始化Shabal-512上下文。此过程不执行任何内存分配。
 *
 * @param cc   Shabal-512上下文（指向<code>sph_shabal512_context</code>的指针）
 */
void sph_shabal512_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Shabal-512上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_shabal512(void *cc, const void *data, size_t len);

/**
 * 终止当前的Shabal-512计算并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（64字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Shabal-512上下文
 * @param dst   目标缓冲区
 */
void sph_shabal512_close(void *cc, void *dst);
/**
 * 在当前计算的基础上添加一些额外的位（0到7），然后终止计算，并将结果输出到提供的缓冲区中，该缓冲区必须足够大以容纳结果（64字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位是从7到8-n编号的位（这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Shabal-512上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
```