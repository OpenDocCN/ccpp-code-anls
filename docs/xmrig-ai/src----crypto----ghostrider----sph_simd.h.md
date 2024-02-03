# `xmrig\src\crypto\ghostrider\sph_simd.h`

```cpp
/* $Id: sph_simd.h 154 2010-04-26 17:00:24Z tp $ */
/**
 * SIMD interface. SIMD is a family of functions which differ by
 * their output size; this implementation defines SIMD for output
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
 * @file     sph_simd.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_SIMD_H__
#define SPH_SIMD_H__

#ifdef __cplusplus
extern "C"{
#endif

#include <stddef.h>
#include "sph_types.h"

/**
 * Output size (in bits) for SIMD-224.
 */
#define SPH_SIZE_simd224   224

/**
 * Output size (in bits) for SIMD-256.
 */
#define SPH_SIZE_simd256   256

/**
 * Output size (in bits) for SIMD-384.
 */
#define SPH_SIZE_simd384   384
/**
 * 定义 SIMD-512 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_simd512   512

/**
 * 这个结构是用于 SIMD 计算的上下文：它包含了中间值和上一个输入块的一些数据。
 * 一旦进行了 SIMD 计算，上下文就可以被重用于另一个计算。这个特定的结构用于 SIMD-224 和 SIMD-256。
 *
 * 这个结构的内容是私有的。一个正在运行的 SIMD 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[64];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 state[16];
    sph_u32 count_low, count_high;
#endif
} sph_simd_small_context;

/**
 * 这个结构是用于 SIMD 计算的上下文：它包含了中间值和上一个输入块的一些数据。
 * 一旦进行了 SIMD 计算，上下文就可以被重用于另一个计算。这个特定的结构用于 SIMD-384 和 SIMD-512。
 *
 * 这个结构的内容是私有的。一个正在运行的 SIMD 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[128];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 state[32];
    sph_u32 count_low, count_high;
#endif
} sph_simd_big_context;

/**
 * SIMD-224 上下文的类型（与常见的“small”上下文相同）。
 */
typedef sph_simd_small_context sph_simd224_context;

/**
 * SIMD-256 上下文的类型（与常见的“small”上下文相同）。
 */
typedef sph_simd_small_context sph_simd256_context;

/**
 * SIMD-384 上下文的类型（与常见的“big”上下文相同）。
 */
typedef sph_simd_big_context sph_simd384_context;

/**
 * SIMD-512 上下文的类型（与常见的“big”上下文相同）。
 */
typedef sph_simd_big_context sph_simd512_context;
/**
 * 初始化一个 SIMD-224 上下文。此过程不执行任何内存分配。
 *
 * @param cc   SIMD-224 上下文（指向 <code>sph_simd224_context</code> 的指针）
 */
void sph_simd224_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（此时此函数不执行任何操作）。
 *
 * @param cc     SIMD-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_simd224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 SIMD-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。上下文会自动重新初始化。
 *
 * @param cc    SIMD-224 上下文
 * @param dst   目标缓冲区
 */
void sph_simd224_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些附加位（0到7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。如果 <code>ub</code> 中的第i位的值为2^i，则额外位为从7到8-n编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    SIMD-224 上下文
 * @param ub    附加位
 * @param n     附加位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_simd224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 SIMD-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   SIMD-256 上下文（指向 <code>sph_simd256_context</code> 的指针）
 */
void sph_simd256_init(void *cc);
/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     SIMD-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_simd256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 SIMD-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。上下文会自动重新初始化。
 *
 * @param cc    SIMD-256 上下文
 * @param dst   目标缓冲区
 */
void sph_simd256_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外的位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    SIMD-256 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_simd256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 SIMD-384 上下文。此过程不执行任何内存分配。
 *
 * @param cc   SIMD-384 上下文（指向 <code>sph_simd384_context</code> 的指针）
 */
void sph_simd384_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     SIMD-384 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_simd384(void *cc, const void *data, size_t len);
/**
 * 终止当前的SIMD-384计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48字节）。上下文会自动重新初始化。
 *
 * @param cc    SIMD-384上下文
 * @param dst   目标缓冲区
 */
void sph_simd384_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些附加位（0到7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外位为从7到8-n的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    SIMD-384上下文
 * @param ub    附加位
 * @param n     附加位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_simd384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个SIMD-512上下文。此过程不执行任何内存分配。
 *
 * @param cc   SIMD-512上下文（指向<code>sph_simd512_context</code>的指针）
 */
void sph_simd512_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     SIMD-512上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_simd512(void *cc, const void *data, size_t len);

/**
 * 终止当前的SIMD-512计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64字节）。上下文会自动重新初始化。
 *
 * @param cc    SIMD-512上下文
 * @param dst   目标缓冲区
 */
void sph_simd512_close(void *cc, void *dst);
/**
 * 在当前计算的基础上添加一些额外的位（0到7），然后终止计算并将结果输出到提供的缓冲区中，
 * 该缓冲区必须足够大以容纳结果（64字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位是从7到8-n编号的位（这是字节级别的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    SIMD-512上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_simd512_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);
#ifdef __cplusplus
}
#endif

#endif


注释：这段代码是一个函数声明，声明了一个名为`sph_simd512_addbits_and_close`的函数，该函数接受四个参数：`cc`表示SIMD-512上下文，`ub`表示额外的位，`n`表示额外位的数量，`dst`表示目标缓冲区。函数的作用是在当前计算的基础上添加一些额外的位，然后终止计算并将结果输出到提供的缓冲区中。函数的具体实现在其他地方。
```