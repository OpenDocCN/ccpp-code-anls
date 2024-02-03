# `xmrig\src\crypto\ghostrider\sph_skein.h`

```cpp
# 定义了 Skein 接口，包括 Skein-256、Skein-512 和 Skein-1024 三个主要函数
# 本代码实现了 Skein-512，可以根据输出长度进一步参数化
# 在 SHA-3 竞赛中，Skein-512 用于输出长度为 224、256、384 和 512 位的情况
# 因此，我们将 Skein-224、Skein-256、Skein-384 和 Skein-512 分别称为 Skein-512-224、Skein-512-256、Skein-512-384 和 Skein-512-512

# 版权声明
# 本软件及相关文档（以下简称“软件”）的版权归 Projet RNRT SAPHIR 所有
# 任何获得本软件及相关文档副本的人，可以无限制地使用、复制、修改、合并、发布、分发、再许可和/或销售本软件的副本
# 但需要满足以下条件：
# 在所有副本或实质部分中包含上述版权声明和本许可声明
# 本软件按“原样”提供，不提供任何明示或暗示的保证，包括但不限于适销性、特定用途适用性和非侵权性的保证
# 无论是在合同行为、侵权行为还是其他情况下，作者或版权持有人均不对任何索赔、损害赔偿或其他责任承担责任
# 与软件或使用或其他交易有关

# 定义了 sph_skein.h 文件的宏，防止重复包含
#ifndef SPH_SKEIN_H__
#define SPH_SKEIN_H__
#ifdef __cplusplus
extern "C"{
#endif

#include <stddef.h>
#include "sph_types.h"

#if SPH_64

/**
 * Skein-224 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_skein224   224

/**
 * Skein-256 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_skein256   256

/**
 * Skein-384 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_skein384   384

/**
 * Skein-512 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_skein512   512

/**
 * 这个结构是 Skein 计算的上下文（带有 384 或 512 位输出）：它包含中间值和最后输入块的一些数据。
 * 一旦进行了 Skein 计算，上下文可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 Skein 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
    unsigned char buf[64];    /* 第一个字段，用于对齐 */
    size_t ptr;
    sph_u64 h0, h1, h2, h3, h4, h5, h6, h7;
    sph_u64 bcount;
#endif
} sph_skein_big_context;

/**
 * Skein-224 上下文的类型（与常见的“big”上下文相同）。
 */
typedef sph_skein_big_context sph_skein224_context;

/**
 * Skein-256 上下文的类型（与常见的“big”上下文相同）。
 */
typedef sph_skein_big_context sph_skein256_context;

/**
 * Skein-384 上下文的类型（与常见的“big”上下文相同）。
 */
typedef sph_skein_big_context sph_skein384_context;

/**
 * Skein-512 上下文的类型（与常见的“big”上下文相同）。
 */
typedef sph_skein_big_context sph_skein512_context;

/**
 * 初始化 Skein-224 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Skein-224 上下文（指向 <code>sph_skein224_context</code> 的指针）
 */
void sph_skein224_init(void *cc);
/**
 * 处理一些数据字节。如果 len 为零，则不执行任何操作。
 *
 * @param cc     Skein-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_skein224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Skein-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够大以容纳结果（28 字节）。上下文会自动重新初始化。
 *
 * @param cc    Skein-224 上下文
 * @param dst   目标缓冲区
 */
void sph_skein224_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些附加位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够大以容纳结果（28 字节）。如果 ub 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Skein-224 上下文
 * @param ub    附加位
 * @param n     附加位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_skein224_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化 Skein-256 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Skein-256 上下文（指向 sph_skein256_context 的指针）
 */
void sph_skein256_init(void *cc);

/**
 * 处理一些数据字节。如果 len 为零，则不执行任何操作。
 *
 * @param cc     Skein-256 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_skein256(void *cc, const void *data, size_t len);
/**
 * 终止当前的 Skein-256 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。上下文会自动重新初始化。
 *
 * @param cc    Skein-256 上下文
 * @param dst   目标缓冲区
 */
void sph_skein256_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（32 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Skein-256 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_skein256_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化 Skein-384 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Skein-384 上下文（指向 <code>sph_skein384_context</code> 的指针）
 */
void sph_skein384_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Skein-384 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_skein384(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Skein-384 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48 字节）。上下文会自动重新初始化。
 *
 * @param cc    Skein-384 上下文
 * @param dst   目标缓冲区
 */
void sph_skein384_close(void *cc, void *dst);
/**
 * 将一些额外的位（0到7位）添加到当前计算中，然后终止计算并将结果输出到提供的缓冲区中，
 * 缓冲区必须足够大以容纳结果（48字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外的位是从7到8-n编号的位（这是字节级别的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Skein-384上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_skein384_addbits_and_close(
    void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化Skein-512上下文。此过程不执行任何内存分配。
 *
 * @param cc   Skein-512上下文（指向<code>sph_skein512_context</code>的指针）
 */
void sph_skein512_init(void *cc);

/**
 * 处理一些数据字节。允许<code>len</code>为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Skein-512上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_skein512(void *cc, const void *data, size_t len);

/**
 * 终止当前的Skein-512计算并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够大以容纳结果（64字节）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Skein-512上下文
 * @param dst   目标缓冲区
 */
void sph_skein512_close(void *cc, void *dst);
/**
 * This code block is the end of a C header file. It contains the declaration of a function named "sph_skein512_addbits_and_close".
 * The function takes several parameters: a context object, a value representing additional bits, the number of additional bits, and a destination buffer.
 * The function adds the additional bits to the current computation, terminates it, and outputs the result in the destination buffer.
 * The destination buffer must be large enough to accommodate the result, which is 64 bytes in size.
 * The function also automatically reinitializes the context.
 * This code block is enclosed in an "#ifdef __cplusplus" block, which indicates that the code is written in C++.
 * The "#endif" statements mark the end of the "#ifdef" blocks.
 * The "#endif" at the end of the code block marks the end of the "#ifndef" block that checks if the header file has already been included.
 */
```