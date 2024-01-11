# `xmrig\src\crypto\ghostrider\sph_cubehash.h`

```
/* $Id: sph_cubehash.h 180 2010-05-08 02:29:25Z tp $ */
/**
 * CubeHash interface. CubeHash is a family of functions which differ by
 * their output size; this implementation defines CubeHash for output
 * sizes 224, 256, 384 and 512 bits, with the "standard parameters"
 * (CubeHash16/32 with the CubeHash specification notations).
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
 * @file     sph_cubehash.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_CUBEHASH_H__
#define SPH_CUBEHASH_H__

#ifdef __cplusplus
extern "C"{
#endif

#include <stddef.h>
#include "sph_types.h"

/**
 * Output size (in bits) for CubeHash-224.
 */
#define SPH_SIZE_cubehash224   224

/**
 * Output size (in bits) for CubeHash-256.
 */
/**
 * 定义 CubeHash256 的输出大小（以比特为单位）
 */
#define SPH_SIZE_cubehash256   256

/**
 * 定义 CubeHash384 的输出大小（以比特为单位）
 */
#define SPH_SIZE_cubehash384   384

/**
 * 定义 CubeHash512 的输出大小（以比特为单位）
 */
#define SPH_SIZE_cubehash512   512

/**
 * 这个结构是 CubeHash 计算的上下文：它包含中间值和最后输入块的一些数据。
 * 一旦 CubeHash 计算完成，上下文可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 CubeHash 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[32];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 state[32];
#endif
} sph_cubehash_context;

/**
 * CubeHash-224 上下文的类型（与通用上下文相同）
 */
typedef sph_cubehash_context sph_cubehash224_context;

/**
 * CubeHash-256 上下文的类型（与通用上下文相同）
 */
typedef sph_cubehash_context sph_cubehash256_context;

/**
 * CubeHash-384 上下文的类型（与通用上下文相同）
 */
typedef sph_cubehash_context sph_cubehash384_context;

/**
 * CubeHash-512 上下文的类型（与通用上下文相同）
 */
typedef sph_cubehash_context sph_cubehash512_context;

/**
 * 初始化一个 CubeHash-224 上下文。这个过程不进行内存分配。
 *
 * @param cc   CubeHash-224 上下文（指向 <code>sph_cubehash224_context</code> 的指针）
 */
void sph_cubehash224_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     CubeHash-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_cubehash224(void *cc, const void *data, size_t len);
/**
 * 终止当前的 CubeHash-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。上下文会自动重新初始化。
 *
 * @param cc    CubeHash-224 上下文
 * @param dst   目标缓冲区
 */
void sph_cubehash224_close(void *cc, void *dst);

/**
 * 添加一些额外的位（0到7）到当前计算中，然后终止它并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（28字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位是从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    CubeHash-224 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_cubehash224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化 CubeHash-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   CubeHash-256 上下文（指向 <code>sph_cubehash256_context</code> 的指针）
 */
void sph_cubehash256_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     CubeHash-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_cubehash256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 CubeHash-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32字节）。上下文会自动重新初始化。
 *
 * @param cc    CubeHash-256 上下文
 * @param dst   目标缓冲区
 */
void sph_cubehash256_close(void *cc, void *dst);
/**
 * 向当前计算添加一些额外的位（0到7），然后终止计算，并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（32字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位是从7到8-n编号的位（这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    CubeHash-256上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_cubehash256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个CubeHash-384上下文。此过程不执行任何内存分配。
 *
 * @param cc   CubeHash-384上下文（指向<code>sph_cubehash384_context</code>的指针）
 */
void sph_cubehash384_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     CubeHash-384上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_cubehash384(void *cc, const void *data, size_t len);

/**
 * 终止当前的CubeHash-384计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    CubeHash-384上下文
 * @param dst   目标缓冲区
 */
void sph_cubehash384_close(void *cc, void *dst);
/**
 * 将一些额外的位（0到7）添加到当前计算中，然后终止计算，并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（48字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位是从7到8-n编号的位（这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    CubeHash-384上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_cubehash384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个CubeHash-512上下文。此过程不执行任何内存分配。
 *
 * @param cc   CubeHash-512上下文（指向<code>sph_cubehash512_context</code>的指针）
 */
void sph_cubehash512_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     CubeHash-512上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_cubehash512(void *cc, const void *data, size_t len);

/**
 * 终止当前的CubeHash-512计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    CubeHash-512上下文
 * @param dst   目标缓冲区
 */
void sph_cubehash512_close(void *cc, void *dst);
/**
 * 将一些额外的位（0到7）添加到当前计算中，然后终止计算，并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（64字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位是从7到8-n编号的位（这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    CubeHash-512上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_cubehash512_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);
#ifdef __cplusplus
}
#endif

#endif
```