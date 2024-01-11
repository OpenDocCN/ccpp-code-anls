# `xmrig\src\crypto\ghostrider\sph_blake.h`

```
/* $Id: sph_blake.h 252 2011-06-07 17:55:14Z tp $ */
/**
 * BLAKE interface. BLAKE is a family of functions which differ by their
 * output size; this implementation defines BLAKE for output sizes 224,
 * 256, 384 and 512 bits. This implementation conforms to the "third
 * round" specification.
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
 * @file     sph_blake.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_BLAKE_H__
#define SPH_BLAKE_H__

#ifdef __cplusplus
extern "C"{
#endif

#include <stddef.h>
#include "sph_types.h"

/**
 * Output size (in bits) for BLAKE-224.
 */
#define SPH_SIZE_blake224   224

/**
 * Output size (in bits) for BLAKE-256.
 */
#define SPH_SIZE_blake256   256

#if SPH_64
/**
 * 定义 BLAKE-384 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_blake384   384

/**
 * 定义 BLAKE-512 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_blake512   512

#endif

/**
 * 这个结构是用于 BLAKE-224 和 BLAKE-256 计算的上下文：
 * 它包含了中间值和最后输入块的一些数据。
 * 一旦进行了 BLAKE 计算，上下文可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。一个正在运行的 BLAKE 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[64];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 H[8];
    sph_u32 S[4];
    sph_u32 T0, T1;
#endif
} sph_blake_small_context;

/**
 * 这个结构是用于 BLAKE-224 计算的上下文。它与通用的 <code>sph_blake_small_context</code> 相同。
 */
typedef sph_blake_small_context sph_blake224_context;

/**
 * 这个结构是用于 BLAKE-256 计算的上下文。它与通用的 <code>sph_blake_small_context</code> 相同。
 */
typedef sph_blake_small_context sph_blake256_context;

#if SPH_64

/**
 * 这个结构是用于 BLAKE-384 和 BLAKE-512 计算的上下文：
 * 它包含了中间值和最后输入块的一些数据。
 * 一旦进行了 BLAKE 计算，上下文可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。一个正在运行的 BLAKE 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[128];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u64 H[8];
    sph_u64 S[4];
    sph_u64 T0, T1;
#endif
} sph_blake_big_context;

/**
 * 这个结构是用于 BLAKE-384 计算的上下文。它与通用的 <code>sph_blake_small_context</code> 相同。
 */
/**
 * 定义一个 BLAKE-384 上下文，它与通用的 <code>sph_blake_big_context</code> 相同。
 */
typedef sph_blake_big_context sph_blake384_context;

/**
 * 这个结构是 BLAKE-512 计算的上下文。它与通用的 <code>sph_blake_small_context</code> 相同。
 */
typedef sph_blake_big_context sph_blake512_context;

#endif

/**
 * 初始化一个 BLAKE-224 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   BLAKE-224 上下文（指向 <code>sph_blake224_context</code> 的指针）
 */
void sph_blake224_init(void *cc);

/**
 * 处理一些数据字节。接受长度为零的情况（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     BLAKE-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_blake224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 BLAKE-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。上下文会自动重新初始化。
 *
 * @param cc    BLAKE-224 上下文
 * @param dst   目标缓冲区
 */
void sph_blake224_close(void *cc, void *dst);

/**
 * 添加一些额外的位（0到7）到当前的计算中，然后终止它并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（28字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外的位是从第 7 位到第 8-n 位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    BLAKE-224 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_blake224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);
/**
 * 初始化 BLAKE-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   BLAKE-256 上下文（指向 <code>sph_blake256_context</code> 的指针）
 */
void sph_blake256_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     BLAKE-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_blake256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 BLAKE-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32字节）。上下文会自动重新初始化。
 *
 * @param cc    BLAKE-256 上下文
 * @param dst   目标缓冲区
 */
void sph_blake256_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（32字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    BLAKE-256 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_blake256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

#if SPH_64

/**
 * 初始化 BLAKE-384 上下文。此过程不执行任何内存分配。
 *
 * @param cc   BLAKE-384 上下文（指向 <code>sph_blake384_context</code> 的指针）
 */
void sph_blake384_init(void *cc);
/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     BLAKE-384 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_blake384(void *cc, const void *data, size_t len);

/**
 * 终止当前的 BLAKE-384 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48 字节）。上下文会自动重新初始化。
 *
 * @param cc    BLAKE-384 上下文
 * @param dst   目标缓冲区
 */
void sph_blake384_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外的位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（48 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    BLAKE-384 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_blake384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 BLAKE-512 上下文。此过程不执行任何内存分配。
 *
 * @param cc   BLAKE-512 上下文（指向 <code>sph_blake512_context</code> 的指针）
 */
void sph_blake512_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     BLAKE-512 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_blake512(void *cc, const void *data, size_t len);
/**
 * 终止当前的 BLAKE-512 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64 字节）。上下文会自动重新初始化。
 *
 * @param cc    BLAKE-512 上下文
 * @param dst   目标缓冲区
 */
void sph_blake512_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    BLAKE-512 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_blake512_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

#endif

#ifdef __cplusplus
}
#endif

#endif
```