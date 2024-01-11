# `xmrig\src\crypto\ghostrider\sph_luffa.h`

```
/* $Id: sph_luffa.h 154 2010-04-26 17:00:24Z tp $ */
# 定义了 Luffa 接口，Luffa 是一组根据输出大小不同而不同的函数；这个实现定义了输出大小为 224、256、384 和 512 位的 Luffa 函数。
/**
 * Luffa interface. Luffa is a family of functions which differ by
 * their output size; this implementation defines Luffa for output
 * sizes 224, 256, 384 and 512 bits.
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
 * @file     sph_luffa.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_LUFFA_H__
#define SPH_LUFFA_H__

#ifdef __cplusplus
extern "C"{
#endif

#include <stddef.h>
#include "sph_types.h"

/**
 * Output size (in bits) for Luffa-224.
 */
#define SPH_SIZE_luffa224   224

/**
 * Output size (in bits) for Luffa-256.
 */
#define SPH_SIZE_luffa256   256

/**
 * Output size (in bits) for Luffa-384.
 */
#define SPH_SIZE_luffa384   384
/**
 * Luffa-512 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_luffa512   512

/**
 * 这个结构是 Luffa-224 计算的上下文：它包含中间值和最后输入块的一些数据。
 * 一旦进行了 Luffa 计算，上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 Luffa 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[32];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 V[3][8];
#endif
} sph_luffa224_context;

/**
 * 这个结构是 Luffa-256 计算的上下文。它与 <code>sph_luffa224_context</code> 完全相同。
 */
typedef sph_luffa224_context sph_luffa256_context;

/**
 * 这个结构是 Luffa-384 计算的上下文。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[32];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 V[4][8];
#endif
} sph_luffa384_context;

/**
 * 这个结构是 Luffa-512 计算的上下文。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[32];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 V[5][8];
#endif
} sph_luffa512_context;

/**
 * 初始化一个 Luffa-224 上下文。这个过程不进行内存分配。
 *
 * @param cc   Luffa-224 上下文（指向一个 <code>sph_luffa224_context</code> 的指针）
 */
void sph_luffa224_init(void *cc);

/**
 * 处理一些数据字节。可以接受 <code>len</code> 为零（在这种情况下，此函数什么也不做）。
 *
 * @param cc     Luffa-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_luffa224(void *cc, const void *data, size_t len);
/**
 * 终止当前的 Luffa-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。上下文会自动重新初始化。
 *
 * @param cc    Luffa-224 上下文
 * @param dst   目标缓冲区
 */
void sph_luffa224_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外位（0到7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Luffa-224 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_luffa224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 Luffa-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Luffa-256 上下文（指向 <code>sph_luffa256_context</code> 的指针）
 */
void sph_luffa256_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Luffa-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_luffa256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Luffa-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32字节）。上下文会自动重新初始化。
 *
 * @param cc    Luffa-256 上下文
 * @param dst   目标缓冲区
 */
void sph_luffa256_close(void *cc, void *dst);
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (32 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the Luffa-256 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_luffa256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * Initialize a Luffa-384 context. This process performs no memory allocation.
 *
 * @param cc   the Luffa-384 context (pointer to a
 *             <code>sph_luffa384_context</code>)
 */
void sph_luffa384_init(void *cc);

/**
 * Process some data bytes. It is acceptable that <code>len</code> is zero
 * (in which case this function does nothing).
 *
 * @param cc     the Luffa-384 context
 * @param data   the input data
 * @param len    the input data length (in bytes)
 */
void sph_luffa384(void *cc, const void *data, size_t len);

/**
 * Terminate the current Luffa-384 computation and output the result into
 * the provided buffer. The destination buffer must be wide enough to
 * accomodate the result (48 bytes). The context is automatically
 * reinitialized.
 *
 * @param cc    the Luffa-384 context
 * @param dst   the destination buffer
 */
void sph_luffa384_close(void *cc, void *dst);
*/
/**
 * 向当前计算添加一些额外的位（0到7），然后终止计算并将结果输出到提供的缓冲区中，
 * 缓冲区必须足够大以容纳结果（48字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位为从7到8-n的位（这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Luffa-384上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_luffa384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化Luffa-512上下文。此过程不执行任何内存分配。
 *
 * @param cc   Luffa-512上下文（指向<code>sph_luffa512_context</code>的指针）
 */
void sph_luffa512_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Luffa-512上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_luffa512(void *cc, const void *data, size_t len);

/**
 * 终止当前的Luffa-512计算并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（64字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Luffa-512上下文
 * @param dst   目标缓冲区
 */
void sph_luffa512_close(void *cc, void *dst);
/**
 * sph_luffa512_addbits_and_close函数用于在当前计算的基础上添加一些额外的位（0到7位），然后终止计算并将结果输出到提供的缓冲区中。
 * 缓冲区必须足够大以容纳结果（64字节）。如果ub中的第i位的值为2^i，则额外的位是从7到8-n编号的位（这是字节级别的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Luffa-512上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
```