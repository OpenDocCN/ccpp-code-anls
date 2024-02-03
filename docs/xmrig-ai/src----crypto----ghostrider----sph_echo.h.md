# `xmrig\src\crypto\ghostrider\sph_echo.h`

```cpp
# 定义了 ECHO 接口，ECHO 是一组函数，根据输出大小的不同分为 224、256、384 和 512 位
# 版权声明，允许在遵守条件的情况下使用和分发软件
# 文件名和作者信息
#ifndef SPH_ECHO_H__
#define SPH_ECHO_H__

#ifdef __cplusplus
extern "C"{
#endif

#include <stddef.h>
#include "sph_types.h"

# 输出大小（以位为单位）为 ECHO-224
#define SPH_SIZE_echo224   224

# 输出大小（以位为单位）为 ECHO-256
#define SPH_SIZE_echo256   256

# 输出大小（以位为单位）为 ECHO-384
#define SPH_SIZE_echo384   384
/**
 * 定义 ECHO-512 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_echo512   512

/**
 * 这个结构是用于 ECHO 计算的上下文：它包含中间值和上一个输入块的一些数据。
 * 一旦进行了 ECHO 计算，就可以重用上下文进行另一个计算。这个特定的结构用于 ECHO-224 和 ECHO-256。
 *
 * 这个结构的内容是私有的。运行中的 ECHO 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[192];    /* 第一个字段，用于对齐 */
    size_t ptr;
    union {
        sph_u32 Vs[4][4];
#if SPH_64
        sph_u64 Vb[4][2];
#endif
    } u;
    sph_u32 C0, C1, C2, C3;
#endif
} sph_echo_small_context;

/**
 * 这个结构是用于 ECHO 计算的上下文：它包含中间值和上一个输入块的一些数据。
 * 一旦进行了 ECHO 计算，就可以重用上下文进行另一个计算。这个特定的结构用于 ECHO-384 和 ECHO-512。
 *
 * 这个结构的内容是私有的。运行中的 ECHO 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[128];    /* 第一个字段，用于对齐 */
    size_t ptr;
    union {
        sph_u32 Vs[8][4];
#if SPH_64
        sph_u64 Vb[8][2];
#endif
    } u;
    sph_u32 C0, C1, C2, C3;
#endif
} sph_echo_big_context;

/**
 * ECHO-224 上下文的类型（与常见的“小”上下文相同）。
 */
typedef sph_echo_small_context sph_echo224_context;

/**
 * ECHO-256 上下文的类型（与常见的“小”上下文相同）。
 */
typedef sph_echo_small_context sph_echo256_context;

/**
 * ECHO-384 上下文的类型（与常见的“大”上下文相同）。
 */
typedef sph_echo_big_context sph_echo384_context;
/**
 * ECHO-512 上下文的类型（与常见的“big”上下文相同）。
 */
typedef sph_echo_big_context sph_echo512_context;

/**
 * 初始化一个 ECHO-224 上下文。此过程不执行任何内存分配。
 *
 * @param cc   ECHO-224 上下文（指向 <code>sph_echo224_context</code> 的指针）
 */
void sph_echo224_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     ECHO-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_echo224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 ECHO-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28 字节）。上下文会自动重新初始化。
 *
 * @param cc    ECHO-224 上下文
 * @param dst   目标缓冲区
 */
void sph_echo224_close(void *cc, void *dst);

/**
 * 向当前计算添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    ECHO-224 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_echo224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 ECHO-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   ECHO-256 上下文（指向 <code>sph_echo256_context</code> 的指针）
 */
void sph_echo256_init(void *cc);
/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     ECHO-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_echo256(void *cc, const void *data, size_t len);

/**
 * 终止当前的 ECHO-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。上下文会自动重新初始化。
 *
 * @param cc    ECHO-256 上下文
 * @param dst   目标缓冲区
 */
void sph_echo256_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外的位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    ECHO-256 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_echo256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 ECHO-384 上下文。此过程不执行任何内存分配。
 *
 * @param cc   ECHO-384 上下文（指向 <code>sph_echo384_context</code> 的指针）
 */
void sph_echo384_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     ECHO-384 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_echo384(void *cc, const void *data, size_t len);
/**
 * 终止当前的 ECHO-384 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48 字节）。上下文会自动重新初始化。
 *
 * @param cc    ECHO-384 上下文
 * @param dst   目标缓冲区
 */
void sph_echo384_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    ECHO-384 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_echo384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化一个 ECHO-512 上下文。此过程不执行任何内存分配。
 *
 * @param cc   ECHO-512 上下文（指向 <code>sph_echo512_context</code> 的指针）
 */
void sph_echo512_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     ECHO-512 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_echo512(void *cc, const void *data, size_t len);

/**
 * 终止当前的 ECHO-512 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64 字节）。上下文会自动重新初始化。
 *
 * @param cc    ECHO-512 上下文
 * @param dst   目标缓冲区
 */
void sph_echo512_close(void *cc, void *dst);
/**
 * Add a few additional bits (0 to 7) to the current computation, then
 * terminate it and output the result in the provided buffer, which must
 * be wide enough to accomodate the result (64 bytes). If bit number i
 * in <code>ub</code> has value 2^i, then the extra bits are those
 * numbered 7 downto 8-n (this is the big-endian convention at the byte
 * level). The context is automatically reinitialized.
 *
 * @param cc    the ECHO-512 context
 * @param ub    the extra bits
 * @param n     the number of extra bits (0 to 7)
 * @param dst   the destination buffer
 */
void sph_echo512_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);
#ifdef __cplusplus
}
#endif
#endif
```