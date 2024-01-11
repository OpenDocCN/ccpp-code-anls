# `xmrig\src\crypto\ghostrider\sph_whirlpool.h`

```
/* $Id: sph_whirlpool.h 216 2010-06-08 09:46:57Z tp $ */
// 定义了一个头文件保护符，防止重复包含
#ifndef SPH_WHIRLPOOL_H__
#define SPH_WHIRLPOOL_H__

#include "sph_types.h"
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

#if SPH_64

/**
 * WHIRLPOOL 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_whirlpool 512

/**
 * WHIRLPOOL-0 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_whirlpool0 512

/**
 * WHIRLPOOL-1 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_whirlpool1 512

/**
 * 这个结构是 WHIRLPOOL 计算的上下文：它包含中间值和最后输入块的一些数据。
 * 一旦进行了 WHIRLPOOL 计算，上下文可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 WHIRLPOOL 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
  unsigned char buf[64]; /* 第一个字段，用于对齐 */
  sph_u64 state[8];
#if SPH_64
  sph_u64 count;
#else
  sph_u32 count_high, count_low;
#endif
#endif
} sph_whirlpool_context;

/**
 * 初始化一个 WHIRLPOOL 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   WHIRLPOOL 上下文（指向一个 <code>sph_whirlpool_context</code> 的指针）
 */
void sph_whirlpool_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 * 此函数应用普通的 WHIRLPOOL 算法。
 *
 * @param cc     WHIRLPOOL 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_whirlpool(void *cc, const void *data, size_t len);
/**
 * 终止当前的 WHIRLPOOL 计算，并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够宽以容纳结果（64 字节）。上下文会自动重新初始化。
 *
 * @param cc    WHIRLPOOL 上下文
 * @param dst   目标缓冲区
 */
void sph_whirlpool_close(void *cc, void *dst);

#define sph_whirlpool512_full(cc, dst, data, len)                              \
  do {                                                                         \
    sph_whirlpool_init(cc);                                                    \
    sph_whirlpool(cc, data, len);                                              \
    sph_whirlpool_close(cc, dst);                                              \
  } while (0)

/**
 * WHIRLPOOL-0 使用与普通 WHIRLPOOL 相同的结构。
 */
typedef sph_whirlpool_context sph_whirlpool0_context;

#ifdef DOXYGEN_IGNORE
/**
 * 初始化 WHIRLPOOL-0 上下文。此函数与 <code>sph_whirlpool_init()</code> 相同。
 *
 * @param cc   WHIRLPOOL 上下文（指向 <code>sph_whirlpool0_context</code> 的指针）
 */
void sph_whirlpool0_init(void *cc);
#endif

#ifndef DOXYGEN_IGNORE
#define sph_whirlpool0_init sph_whirlpool_init
#endif

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。此函数应用 WHIRLPOOL-0 算法。
 *
 * @param cc     WHIRLPOOL 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_whirlpool0(void *cc, const void *data, size_t len);

/**
 * 终止当前的 WHIRLPOOL-0 计算，并将结果输出到提供的缓冲区中。
 * 目标缓冲区必须足够宽以容纳结果（64 字节）。上下文会自动重新初始化。
 *
 * @param cc    WHIRLPOOL-0 上下文
 * @param dst   目标缓冲区
 */
/**
 * 关闭 WHIRLPOOL-1 上下文，将结果输出到提供的缓冲区中
 * 目标缓冲区必须足够宽以容纳结果（64字节）
 * 上下文会自动重新初始化
 *
 * @param cc    WHIRLPOOL-1 上下文
 * @param dst   目标缓冲区
 */
void sph_whirlpool1_close(void *cc, void *dst);

/**
 * WHIRLPOOL-1 使用与普通 WHIRLPOOL 相同的结构
 */
typedef sph_whirlpool_context sph_whirlpool1_context;

#ifdef DOXYGEN_IGNORE
/**
 * 初始化 WHIRLPOOL-1 上下文。此函数与 sph_whirlpool_init() 相同
 *
 * @param cc   WHIRLPOOL 上下文（指向 sph_whirlpool1_context 的指针）
 */
void sph_whirlpool1_init(void *cc);
#endif

#ifndef DOXYGEN_IGNORE
#define sph_whirlpool1_init sph_whirlpool_init
#endif

/**
 * 处理一些数据字节。允许 len 为零（此时此函数不执行任何操作）。此函数应用 WHIRLPOOL-1 算法
 *
 * @param cc     WHIRLPOOL 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_whirlpool1(void *cc, const void *data, size_t len);

#endif

#ifdef __cplusplus
}
#endif

#endif
```