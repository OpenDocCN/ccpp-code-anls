# `xmrig\src\crypto\ghostrider\sph_sha2.h`

```
# 定义了一个头文件 sph_sha2.h，用于提供 SHA-224、SHA-256、SHA-384 和 SHA-512 的接口
# SHA-256 已经在 FIPS 180-2 中发布，现在通过更改通知也包括了 SHA-224（它是 SHA-256 的简单变体）。
# SHA-384 和 SHA-512 也在 FIPS 180-2 中定义。FIPS 标准可以在以下网址找到：
#    http://csrc.nist.gov/publications/fips/
#
# ==========================(LICENSE BEGIN)============================
#
# 版权所有（c）2007-2010 Projet RNRT SAPHIR
#
# 特此免费授予任何获得本软件及相关文档文件（以下简称“软件”）的人员，无偿使用本软件，包括但不限于使用、复制、修改、合并、发布、分发、再许可和/或销售本软件的副本，以及允许获得本软件的人员这样做，但须遵守以下条件：
#
# 上述版权声明和本许可声明应包含在所有副本或实质部分的软件中。
#
# 本软件按“原样”提供，不提供任何明示或暗示的保证，包括但不限于适销性、特定用途适用性和非侵权性的保证。无论在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，无论是合同、侵权行为或其他行为引起的，与本软件或使用或其他方式有关。
#
# ===========================(LICENSE END)=============================
#
# 定义了一个头文件 sph_sha2.h，用于提供 SHA-224、SHA-256、SHA-384 和 SHA-512 的接口
# 包含了一些必要的头文件和类型定义
#ifndef SPH_SHA2_H__
#define SPH_SHA2_H__

#include <stddef.h>
#include <stdint.h>
#include "sph_types.h"

# 定义了 SHA-224 的输出大小（以位为单位）
#define SPH_SIZE_sha224   224

# 定义了 SHA-256 的输出大小（以位为单位）
/**
 * 定义 SHA-256 的大小为 256
 */
#define SPH_SIZE_sha256   256

/**
 * 这个结构是 SHA-224 计算的上下文：它包含中间值和最后一个输入块的一些数据。
 * 一旦进行了 SHA-224 计算，上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 SHA-224 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[64];    /* 第一个字段，用于对齐 */
    sph_u32 val[8];
#if SPH_64
    sph_u64 count;
#else
    sph_u32 count_high, count_low;
#endif
#endif
} sph_sha224_context;

/**
 * 这个结构是 SHA-256 计算的上下文。它与 SHA-224 上下文相同。但是，上下文被初始化为 SHA-224
 * <strong>或</strong> SHA-256，但不是两者都有（内部 IV 不同）。
 */
typedef sph_sha224_context sph_sha256_context;

/**
 * 初始化 SHA-224 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   SHA-224 上下文（指向 <code>sph_sha224_context</code> 的指针）
 */
void sph_sha224_init(void *cc);

/**
 * 处理一些数据字节。接受 <code>len</code> 为零是可以接受的（在这种情况下，此函数什么也不做）。
 *
 * @param cc     SHA-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_sha224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 SHA-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28 字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    SHA-224 上下文
 * @param dst   目标缓冲区
 */
void sph_sha224_close(void *cc, void *dst);
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (28 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the SHA-224 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_sha224_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst);

/**
 * Apply the SHA-224 compression function on the provided data. The
 * <code>msg</code> parameter contains the 16 32-bit input blocks,
 * as numerical values (hence after the big-endian decoding). The
 * <code>val</code> parameter contains the 8 32-bit input blocks for
 * the compression function; the output is written in place in this
 * array.
 *
 * @param msg   the message block (16 values)
 * @param val   the function 256-bit input and output
 */
void sph_sha224_comp(const sph_u32 msg[16], sph_u32 val[8]);

/**
 * Initialize a SHA-256 context. This process performs no memory allocation.
 *
 * @param cc   the SHA-256 context (pointer to
 *             a <code>sph_sha256_context</code>)
 */
void sph_sha256_init(void *cc);

#ifdef DOXYGEN_IGNORE
/**
 * Process some data bytes, for SHA-256. This function is identical to
 * <code>sha_224()</code>
 *
 * @param cc     the SHA-224 context
 * @param data   the input data
 * @param len    the input data length (in bytes)
 */
void sph_sha256(void *cc, const void *data, size_t len);
#endif

#ifndef DOXYGEN_IGNORE
#define sph_sha256   sph_sha224
#endif
/**
 * 终止当前的 SHA-256 计算，并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（32字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    SHA-256 上下文
 * @param dst   目标缓冲区
 */
void sph_sha256_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外的位（0到7），
 * 然后终止计算并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（32字节）。
 * 如果 <code>ub</code> 中的第 i 位的值为 2^i，
 * 那么额外的位是从 7 到 8-n 编号的位（这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    SHA-256 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_sha256_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst);

#ifdef DOXYGEN_IGNORE
/**
 * 对提供的数据应用 SHA-256 压缩函数。
 * 此函数与 <code>sha224_comp()</code> 相同。
 *
 * @param msg   消息块（16个值）
 * @param val   函数的256位输入和输出
 */
void sph_sha256_comp(const sph_u32 msg[16], sph_u32 val[8]);
#endif

#ifndef DOXYGEN_IGNORE
#define sph_sha256_comp   sph_sha224_comp
#endif

void sph_sha256_full( void *dst, const void *data, size_t len );
void sha256d(void* hash, const void* data, int len);

// 这些函数不应直接调用，应使用 sha256-hash.h 中的通用函数 sha256_transform_le 和 sha256_transform_be。
void sph_sha256_transform_le( uint32_t *state_out, const uint32_t *data,
                              const uint32_t *state_in );

void sph_sha256_transform_be( uint32_t *state_out, const uint32_t *data,
                              const uint32_t *state_in );


#if SPH_64

/**
 * SHA-384 的输出大小（以位为单位）。
 */
/**
 * 定义 SHA-384 的输出大小（以比特为单位）
 */
#define SPH_SIZE_sha384   384

/**
 * 定义 SHA-512 的输出大小（以比特为单位）
 */
#define SPH_SIZE_sha512   512

/**
 * 这个结构是 SHA-384 计算的上下文：它包含中间值和最后一个输入块的一些数据。
 * 一旦进行了 SHA-384 计算，上下文可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 SHA-384 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[128];    /* 第一个字段，用于对齐 */
    sph_u64 val[8];
    sph_u64 count;
#endif
} sph_sha384_context;

/**
 * 初始化一个 SHA-384 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   SHA-384 上下文（指向 <code>sph_sha384_context</code> 的指针）
 */
void sph_sha384_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     SHA-384 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_sha384(void *cc, const void *data, size_t len);

/**
 * 终止当前的 SHA-384 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48 字节）。上下文会自动重新初始化。
 *
 * @param cc    SHA-384 上下文
 * @param dst   目标缓冲区
 */
void sph_sha384_close(void *cc, void *dst);
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (48 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the SHA-384 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_sha384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst);

/**
 * Apply the SHA-384 compression function on the provided data. The
 * <code>msg</code> parameter contains the 16 64-bit input blocks,
 * as numerical values (hence after the big-endian decoding). The
 * <code>val</code> parameter contains the 8 64-bit input blocks for
 * the compression function; the output is written in place in this
 * array.
 *
 * @param msg   the message block (16 values)
 * @param val   the function 512-bit input and output
 */
void sph_sha384_comp(const sph_u64 msg[16], sph_u64 val[8]);

/**
 * This structure is a context for SHA-512 computations. It is identical
 * to the SHA-384 context. However, a context is initialized for SHA-384
 * <strong>or</strong> SHA-512, but not both (the internal IV is not the
 * same).
 */
typedef sph_sha384_context sph_sha512_context;

/**
 * Initialize a SHA-512 context. This process performs no memory allocation.
 *
 * @param cc   the SHA-512 context (pointer to
 *             a <code>sph_sha512_context</code>)
 */
void sph_sha512_init(void *cc);

#ifdef DOXYGEN_IGNORE
/**
 * Process some data bytes, for SHA-512. This function is identical to
 * <code>sph_sha384()</code>.
 *
 * @param cc     the SHA-384 context
 * @param data   the input data
 * @param len    the input data length (in bytes)
 */
void sph_sha512(void *cc, const void *data, size_t len);
#endif
#ifndef DOXYGEN_IGNORE
#define sph_sha512   sph_sha384
#endif

/**
 * 终止当前的 SHA-512 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64 字节）。上下文会自动重新初始化。
 *
 * @param cc    SHA-512 上下文
 * @param dst   目标缓冲区
 */
void sph_sha512_close(void *cc, void *dst);

/**
 * 向当前计算添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（64 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    SHA-512 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_sha512_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst);

#ifdef DOXYGEN_IGNORE
/**
 * 应用 SHA-512 压缩函数。此函数与 <code>sph_sha384_comp()</code> 相同。
 *
 * @param msg   消息块（16 个值）
 * @param val   函数的 512 位输入和输出
 */
void sph_sha512_comp(const sph_u64 msg[16], sph_u64 val[8]);
#endif

#ifndef DOXYGEN_IGNORE
#define sph_sha512_comp   sph_sha384_comp
#endif

#endif

#endif
```