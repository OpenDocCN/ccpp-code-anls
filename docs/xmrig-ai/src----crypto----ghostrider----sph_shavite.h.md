# `xmrig\src\crypto\ghostrider\sph_shavite.h`

```
/* $Id: sph_shavite.h 208 2010-06-02 20:33:00Z tp $ */
/**
 * SHAvite-3 interface. This code implements SHAvite-3 with the
 * recommended parameters for SHA-3, with outputs of 224, 256, 384 and
 * 512 bits. In the following, we call the function "SHAvite" (without
 * the "-3" suffix), thus "SHAvite-224" is "SHAvite-3 with a 224-bit
 * output".
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
 * @file     sph_shavite.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_SHAVITE_H__
#define SPH_SHAVITE_H__

#include <stddef.h>
#include "sph_types.h"

#ifdef __cplusplus
extern "C"{
#endif

/**
 * Output size (in bits) for SHAvite-224.
 */
#define SPH_SIZE_shavite224   224

/**
 * Output size (in bits) for SHAvite-256.
 */
/**
 * 定义 SHAvite-256 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_shavite256   256

/**
 * 定义 SHAvite-384 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_shavite384   384

/**
 * 定义 SHAvite-512 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_shavite512   512

/**
 * 这个结构是用于 SHAvite-224 和 SHAvite-256 计算的上下文：
 * 它包含了中间值和最后输入块的一些数据。
 * 一旦进行了 SHAvite 计算，上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。一个正在运行的 SHAvite 计算可以通过复制上下文来克隆（例如使用简单的 <code>memcpy()</code>）。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[64];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 h[8];
    sph_u32 count0, count1;
#endif
} sph_shavite_small_context;

/**
 * 这个结构是用于 SHAvite-224 计算的上下文。它与常见的 <code>sph_shavite_small_context</code> 相同。
 */
typedef sph_shavite_small_context sph_shavite224_context;

/**
 * 这个结构是用于 SHAvite-256 计算的上下文。它与常见的 <code>sph_shavite_small_context</code> 相同。
 */
typedef sph_shavite_small_context sph_shavite256_context;

/**
 * 这个结构是用于 SHAvite-384 和 SHAvite-512 计算的上下文：
 * 它包含了中间值和最后输入块的一些数据。
 * 一旦进行了 SHAvite 计算，上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。一个正在运行的 SHAvite 计算可以通过复制上下文来克隆（例如使用简单的 <code>memcpy()</code>）。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[128];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u32 h[16];
    sph_u32 count0, count1, count2, count3;
#endif
} sph_shavite_big_context;
/**
 * 定义了一个 SHAvite-384 计算的上下文结构，与通用的 sph_shavite_small_context 结构相同
 */
typedef sph_shavite_big_context sph_shavite384_context;

/**
 * 定义了一个 SHAvite-512 计算的上下文结构，与通用的 sph_shavite_small_context 结构相同
 */
typedef sph_shavite_big_context sph_shavite512_context;

/**
 * 初始化一个 SHAvite-224 上下文。此过程不执行任何内存分配。
 *
 * @param cc   SHAvite-224 上下文（指向 sph_shavite224_context 的指针）
 */
void sph_shavite224_init(void *cc);

/**
 * 处理一些数据字节。允许 len 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     SHAvite-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_shavite224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 SHAvite-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。上下文会自动重新初始化。
 *
 * @param cc    SHAvite-224 上下文
 * @param dst   目标缓冲区
 */
void sph_shavite224_close(void *cc, void *dst);

/**
 * 向当前计算添加一些额外的位（0到7），然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（28字节）。如果 ub 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    SHAvite-224 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_shavite224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);
/**
 * 初始化一个 SHAvite-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   SHAvite-256 上下文（指向 <code>sph_shavite256_context</code> 的指针）
 */
void sph_shavite256_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     SHAvite-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_shavite256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 SHAvite-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。上下文会自动重新初始化。
 *
 * @param cc    SHAvite-256 上下文
 * @param dst   目标缓冲区
 */
void sph_shavite256_close(void *cc, void *dst);

/**
 * 添加一些额外的位（0 到 7）到当前计算中，然后终止它并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（32 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外的位是从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    SHAvite-256 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_shavite256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 SHAvite-384 上下文。此过程不执行任何内存分配。
 *
 * @param cc   SHAvite-384 上下文（指向 <code>sph_shavite384_context</code> 的指针）
 */
void sph_shavite384_init(void *cc);
/**
 * 处理一些数据字节。如果 len 为零，那么这个函数不做任何操作。
 *
 * @param cc     SHAvite-384 上下文
 * @param data   输入数据
 * @param len    输入数据的长度（以字节为单位）
 */
void sph_shavite384(void *cc, const void *data, size_t len);

/**
 * 终止当前的 SHAvite-384 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够大以容纳结果（48 字节）。上下文会自动重新初始化。
 *
 * @param cc    SHAvite-384 上下文
 * @param dst   目标缓冲区
 */
void sph_shavite384_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些附加位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够大以容纳结果（48 字节）。如果 ub 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    SHAvite-384 上下文
 * @param ub    附加位
 * @param n     附加位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_shavite384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化 SHAvite-512 上下文。此过程不执行任何内存分配。
 *
 * @param cc   SHAvite-512 上下文（指向 sph_shavite512_context 的指针）
 */
void sph_shavite512_init(void *cc);

/**
 * 处理一些数据字节。如果 len 为零，那么这个函数不做任何操作。
 *
 * @param cc     SHAvite-512 上下文
 * @param data   输入数据
 * @param len    输入数据的长度（以字节为单位）
 */
void sph_shavite512(void *cc, const void *data, size_t len);


注释：这些代码是一些函数的声明，用于实现 SHAvite-384 和 SHAvite-512 哈希算法。这些函数用于处理数据字节，并根据算法规则进行计算和输出结果。每个函数都有不同的参数和功能，详细的注释描述了每个函数的作用和使用方法。
/**
 * 终止当前的 SHAvite-512 计算，并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（64字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    SHAvite-512 上下文
 * @param dst   目标缓冲区
 */
void sph_shavite512_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外的位（0到7），然后终止计算并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（64字节）。
 * 如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外的位为从 7 到 8-n 编号的位
 * （这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    SHAvite-512 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_shavite512_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

#ifdef __cplusplus
}
#endif

#endif


注释：这段代码是一个C语言的函数声明，声明了两个函数`sph_shavite512_close`和`sph_shavite512_addbits_and_close`。这些函数用于终止SHAvite-512计算并输出结果。`sph_shavite512_close`函数将结果输出到提供的缓冲区中，而`sph_shavite512_addbits_and_close`函数在终止计算之前还会添加一些额外的位。这些函数的参数和返回值的含义在注释中有详细说明。
```