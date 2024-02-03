# `xmrig\src\crypto\ghostrider\sph_groestl.h`

```cpp
# 定义了 Groestl 接口的相关信息和许可证声明
# 包含了必要的头文件和库
# 定义了 Groestl-224 的输出大小
# 定义了 Groestl-256 的输出大小
# 定义了 Groestl-384 的输出大小
/**
 * Groestl-512 的输出大小（以比特为单位）。
 */
#define SPH_SIZE_groestl512 512

/**
 * 这个结构是 Groestl-224 和 Groestl-256 计算的上下文：
 * 它包含了中间值和最后输入块的一些数据。
 * 一旦进行了 Groestl 计算，上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 Groestl 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
  unsigned char buf[64]; /* 第一个字段，用于对齐 */
  size_t ptr;
  union {
#if SPH_64
    sph_u64 wide[8];
#endif
    sph_u32 narrow[16];
  } state;
#if SPH_64
  sph_u64 count;
#else
  sph_u32 count_high, count_low;
#endif
#endif
} sph_groestl_small_context;

/**
 * 这个结构是 Groestl-224 计算的上下文。它与常见的 <code>sph_groestl_small_context</code> 相同。
 */
typedef sph_groestl_small_context sph_groestl224_context;

/**
 * 这个结构是 Groestl-256 计算的上下文。它与常见的 <code>sph_groestl_small_context</code> 相同。
 */
typedef sph_groestl_small_context sph_groestl256_context;

/**
 * 这个结构是 Groestl-384 和 Groestl-512 计算的上下文：
 * 它包含了中间值和最后输入块的一些数据。
 * 一旦进行了 Groestl 计算，上下文就可以被重用于另一个计算。
 *
 * 这个结构的内容是私有的。运行中的 Groestl 计算可以通过复制上下文（例如使用简单的 <code>memcpy()</code>）来克隆。
 */
typedef struct {
#ifndef DOXYGEN_IGNORE
  unsigned char buf[128]; /* 第一个字段，用于对齐 */
  size_t ptr;
  union {
#if SPH_64
    sph_u64 wide[16];
#endif
    sph_u32 narrow[32];
  } state;
#if SPH_64
  sph_u64 count;
#else
  sph_u32 count_high, count_low;
#endif
#endif
} sph_groestl_big_context;
/**
 * 这个结构是 Groestl-384 计算的上下文。它与常见的 <code>sph_groestl_small_context</code> 相同。
 */
typedef sph_groestl_big_context sph_groestl384_context;

/**
 * 这个结构是 Groestl-512 计算的上下文。它与常见的 <code>sph_groestl_small_context</code> 相同。
 */
typedef sph_groestl_big_context sph_groestl512_context;

/**
 * 初始化 Groestl-224 上下文。这个过程不执行任何内存分配。
 *
 * @param cc   Groestl-224 上下文 (指向 <code>sph_groestl224_context</code> 的指针)
 */
void sph_groestl224_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Groestl-224 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_groestl224(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Groestl-224 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。上下文会自动重新初始化。
 *
 * @param cc    Groestl-224 上下文
 * @param dst   目标缓冲区
 */
void sph_groestl224_close(void *cc, void *dst);

/**
 * 向当前计算添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（28字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Groestl-224 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
/**
 * 将一些额外的位（0到7）添加到当前计算中，然后终止计算并将结果输出到提供的缓冲区中，该缓冲区必须足够宽以容纳结果（32字节）。
 * 如果<code>ub</code>中的第i位的值为2^i，则额外位为从7到8-n编号的位（这是字节级的大端约定）。
 * 上下文会自动重新初始化。
 *
 * @param cc    Groestl-256上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_groestl256_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                      void *dst);

/**
 * 初始化Groestl-384上下文。此过程不执行任何内存分配。
 *
 * @param cc   Groestl-384上下文（指向<code>sph_groestl384_context</code>的指针）
 */
void sph_groestl384_init(void *cc);
/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Groestl-384 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_groestl384(void *cc, const void *data, size_t len);

/**
 * 终止当前的 Groestl-384 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48字节）。上下文会自动重新初始化。
 *
 * @param cc    Groestl-384 上下文
 * @param dst   目标缓冲区
 */
void sph_groestl384_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外的位（0到7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（48字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Groestl-384 上下文
 * @param ub    额外的位
 * @param n     额外位的数量（0到7）
 * @param dst   目标缓冲区
 */
void sph_groestl384_addbits_and_close(void *cc, unsigned ub, unsigned n, void *dst);

/**
 * 初始化 Groestl-512 上下文。此过程不执行任何内存分配。
 *
 * @param cc   Groestl-512 上下文（指向 <code>sph_groestl512_context</code> 的指针）
 */
void sph_groestl512_init(void *cc);

/**
 * 处理一些数据字节。允许 <code>len</code> 为零（在这种情况下，此函数不执行任何操作）。
 *
 * @param cc     Groestl-512 上下文
 * @param data   输入数据
 * @param len    输入数据长度（以字节为单位）
 */
void sph_groestl512(void *cc, const void *data, size_t len);
/**
 * 终止当前的 Groestl-512 计算，并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64 字节）。上下文会自动重新初始化。
 *
 * @param cc    Groestl-512 上下文
 * @param dst   目标缓冲区
 */
void sph_groestl512_close(void *cc, void *dst);

/**
 * 向当前计算中添加一些额外位（0 到 7），然后终止计算并将结果输出到提供的缓冲区中。目标缓冲区必须足够宽以容纳结果（64 字节）。如果 <code>ub</code> 中的第 i 位的值为 2^i，则额外位为从 7 到 8-n 编号的位（这是字节级的大端约定）。上下文会自动重新初始化。
 *
 * @param cc    Groestl-512 上下文
 * @param ub    额外位
 * @param n     额外位的数量（0 到 7）
 * @param dst   目标缓冲区
 */
void sph_groestl512_addbits_and_close(void *cc, unsigned ub, unsigned n,
                                      void *dst);

#ifdef __cplusplus
}
#endif

#endif
```