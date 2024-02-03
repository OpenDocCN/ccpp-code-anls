# `xmrig\src\crypto\ghostrider\sph_jh.h`

```cpp
/* $Id: sph_jh.h 216 2010-06-08 09:46:57Z tp $ */
/**
 * JH interface. JH is a family of functions which differ by
 * their output size; this implementation defines JH for output
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
 * @file     sph_jh.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_JH_H__
#define SPH_JH_H__

#ifdef __cplusplus
extern "C"{
#endif

#include <stddef.h>
#include "sph_types.h"

/**
 * Output size (in bits) for JH-224.
 */
#define SPH_SIZE_jh224   224

/**
 * Output size (in bits) for JH-256.
 */
#define SPH_SIZE_jh256   256

/**
 * Output size (in bits) for JH-384.
 */
#define SPH_SIZE_jh384   384

/**
 * Output size (in bits) for JH-512.
 */
/**
 * 定义 JH512 的大小为 512
 */
#define SPH_SIZE_jh512   512

/**
 * 这个结构是 JH 计算的上下文：它包含中间值和最后输入块的一些数据。一旦进行了 JH 计算，上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 JH 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[64];    /* 第一个字段，用于对齐 */
    size_t ptr;
    union {
#if SPH_64
        sph_u64 wide[16];
#endif
        sph_u32 narrow[32];
    } H;
#if SPH_64
    sph_u64 block_count;
#else
    sph_u32 block_count_high, block_count_low;
#endif
#endif
} sph_jh_context;

/**
 * JH-224 上下文的类型（与通用上下文相同）。
 */
typedef sph_jh_context sph_jh224_context;

/**
 * JH-256 上下文的类型（与通用上下文相同）。
 */
typedef sph_jh_context sph_jh256_context;

/**
 * JH-384 上下文的类型（与通用上下文相同）。
 */
typedef sph_jh_context sph_jh384_context;

/**
 * JH-512 上下文的类型（与通用上下文相同）。
 */
typedef sph_jh_context sph_jh512_context;

/**
 * 初始化一个 JH-224 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   JH-224 上下文（指向 <code>sph_jh224_context</code> 的指针）
 */
void sph_jh224_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     JH-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_jh224(void *cc, const void *data, size_t len);
/**
 * 终止当前的 JH-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28 字节）。上下文会自动重新初始化。
 *
 * @param cc    JH-224 上下文
 * @param dst   目标缓冲区
 */
void sph_jh224_close(void *cc, void *dst);

/**
 * 添加一些额外位（0 到 7）到当前计算中，然后终止它并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（28 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位是从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    JH-224 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_jh224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 JH-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   JH-256 上下文（指向 <code>sph_jh256_context</code> 的指针）
 */
void sph_jh256_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     JH-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_jh256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 JH-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。上下文会自动重新初始化。
 *
 * @param cc    JH-256 上下文
 * @param dst   目标缓冲区
 */
void sph_jh256_close(void *cc, void *dst);
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (32 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the JH-256 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_jh256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * Initialize a JH-384 context. This process performs no memory allocation.
 *
 * @param cc   the JH-384 context (pointer to a
 *             <code>sph_jh384_context</code>)
 */
void sph_jh384_init(void *cc);

/**
 * Process some data bytes. It is acceptable that <code>len</code> is zero
 * (in which case this function does nothing).
 *
 * @param cc     the JH-384 context
 * @param data   the input data
 * @param len    the input data length (in bytes)
 */
void sph_jh384(void *cc, const void *data, size_t len);

/**
 * Terminate the current JH-384 computation and output the result into
 * the provided buffer. The destination buffer must be wide enough to
 * accomodate the result (48 bytes). The context is automatically
 * reinitialized.
 *
 * @param cc    the JH-384 context
 * @param dst   the destination buffer
 */
void sph_jh384_close(void *cc, void *dst);
 */
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (48 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the JH-384 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_jh384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * Initialize a JH-512 context. This process performs no memory allocation.
 *
 * @param cc   the JH-512 context (pointer to a
 *             <code>sph_jh512_context</code>)
 */
void sph_jh512_init(void *cc);

/**
 * Process some data bytes. It is acceptable that <code>len</code> is zero
 * (in which case this function does nothing).
 *
 * @param cc     the JH-512 context
 * @param data   the input data
 * @param len    the input data length (in bytes)
 */
void sph_jh512(void *cc, const void *data, size_t len);

/**
 * Terminate the current JH-512 computation and output the result into
 * the provided buffer. The destination buffer must be wide enough to
 * accomodate the result (64 bytes). The context is automatically
 * reinitialized.
 *
 * @param cc    the JH-512 context
 * @param dst   the destination buffer
 */
void sph_jh512_close(void *cc, void *dst);
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (64 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the JH-512 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_jh512_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);
// 结束 C++ 的 extern "C" 块
#ifdef __cplusplus
}
#endif
// 结束条件编译
#endif
```