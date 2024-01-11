# `xmrig\src\crypto\ghostrider\sph_keccak.h`

```
/* $Id: sph_keccak.h 216 2010-06-08 09:46:57Z tp $ */
/**
 * Keccak interface. This is the interface for Keccak with the
 * recommended parameters for SHA-3, with output lengths 224, 256,
 * 384 and 512 bits.
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
 * @file     sph_keccak.h
 * @author   Thomas Pornin <thomas.pornin@cryptolog.com>
 */

#ifndef SPH_KECCAK_H__
#define SPH_KECCAK_H__

#ifdef __cplusplus
extern "C" {
#endif

// Taken from keccak-gate.h
extern int hard_coded_eb;

#include "sph_types.h"
#include <stddef.h>

/**
 * Output size (in bits) for Keccak-224.
 */
#define SPH_SIZE_keccak224 224

/**
 * Output size (in bits) for Keccak-256.
 */
#define SPH_SIZE_keccak256 256

/**
 * Output size (in bits) for Keccak-384.
 */
# 定义 Keccak-384 的输出大小为 384 位
#define SPH_SIZE_keccak384 384

# 定义 Keccak-512 的输出大小为 512 位
#define SPH_SIZE_keccak512 512

# 定义 Keccak 计算的上下文结构，包含中间值和最后一个输入块的一些数据
# 一个 Keccak 计算完成后，可以复制上下文来重用
# 这个结构的内容是私有的，可以通过复制上下文来克隆一个运行中的 Keccak 计算
typedef struct {
#ifndef DOXYGEN_IGNORE
  unsigned char buf[144]; /* first field, for alignment */
  size_t ptr, lim;
  union {
#if SPH_64
    sph_u64 wide[25];
#endif
    sph_u32 narrow[50];
  } u;
#endif
} sph_keccak_context;

# 定义 Keccak-224 上下文的类型，与通用上下文相同
typedef sph_keccak_context sph_keccak224_context;

# 定义 Keccak-256 上下文的类型，与通用上下文相同
typedef sph_keccak_context sph_keccak256_context;

# 定义 Keccak-384 上下文的类型，与通用上下文相同
typedef sph_keccak_context sph_keccak384_context;

# 定义 Keccak-512 上下文的类型，与通用上下文相同
typedef sph_keccak_context sph_keccak512_context;

# 初始化 Keccak-224 上下文，这个过程不进行内存分配
# 参数 cc 是 Keccak-224 上下文的指针
void sph_keccak224_init(void *cc);

# 处理一些数据字节，如果 len 为零则不进行任何操作
# 参数 cc 是 Keccak-224 上下文
# 参数 data 是输入数据
# 参数 len 是输入数据的长度（以字节为单位）
void sph_keccak224(void *cc, const void *data, size_t len);
/**
 * 终止当前的 Keccak-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28 字节）。上下文会自动重新初始化。
 *
 * @param cc    Keccak-224 上下文
 * @param dst   目标缓冲区
 */
void sph_keccak224_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（28 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Keccak-224 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_keccak224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化 Keccak-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Keccak-256 上下文（指向 <code>sph_keccak256_context</code> 的指针）
 */
void sph_keccak256_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Keccak-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_keccak256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Keccak-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。上下文会自动重新初始化。
 *
 * @param cc    Keccak-256 上下文
 * @param dst   目标缓冲区
 */
void sph_keccak256_close(void *cc, void *dst);
/**
 * 将一些附加的位（0到7）添加到当前计算中，然后终止计算并将结果输出到提供的缓冲区中，
 * 缓冲区必须足够大以容纳结果（32字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则附加的位为从7到8-n的位（这是字节级别的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Keccak-256上下文
 * @param ub    附加的位
 * @param n     附加位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_keccak256_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                     void *dst);

/**
 * 初始化Keccak-384上下文。此过程不执行任何内存分配。
 *
 * @param cc   Keccak-384上下文（指向<code>sph_keccak384_context</code>的指针）
 */
void sph_keccak384_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Keccak-384上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_keccak384(void *cc, const void *data, size_t len);

/**
 * 终止当前的Keccak-384计算并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（48字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Keccak-384上下文
 * @param dst   目标缓冲区
 */
void sph_keccak384_close(void *cc, void *dst);
*/
/**
 * 将一些附加的位（0到7）添加到当前计算中，然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够大以容纳结果（48字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则附加的位为从7到8-n编号的位（这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Keccak-384上下文
 * @param ub    附加的位
 * @param n     附加位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_keccak384_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                     void *dst);

/**
 * 初始化Keccak-512上下文。此过程不执行任何内存分配。
 *
 * @param cc   Keccak-512上下文（指向<code>sph_keccak512_context</code>的指针）
 */
void sph_keccak512_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Keccak-512上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_keccak512(void *cc, const void *data, size_t len);

/**
 * 终止当前的Keccak-512计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够大以容纳结果（64字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Keccak-512上下文
 * @param dst   目标缓冲区
 */
void sph_keccak512_close(void *cc, void *dst);
*/
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (64 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the Keccak-512 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_keccak512_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                     void *dst);
// 结束 C++ 的 extern "C" 块
#ifdef __cplusplus
}
#endif
// 结束条件编译
#endif
```