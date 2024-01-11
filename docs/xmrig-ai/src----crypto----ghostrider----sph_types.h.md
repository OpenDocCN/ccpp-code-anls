# `xmrig\src\crypto\ghostrider\sph_types.h`

```
/* $Id: sph_types.h 260 2011-07-21 01:02:38Z tp $ */
#ifndef SPH_TYPES_H__
#define SPH_TYPES_H__

#include <limits.h>

/*
 * All our I/O functions are defined over octet streams. We do not know
 * how to handle input data if bytes are not octets.
 */
#if CHAR_BIT != 8
#error This code requires 8-bit bytes
#endif

/* ============= BEGIN documentation block for Doxygen ============ */

#ifdef DOXYGEN_IGNORE

/** @hideinitializer
 * Unsigned integer type whose length is at least 32 bits; on most
 * architectures, it will have a width of exactly 32 bits. Unsigned C
 * types implement arithmetics modulo a power of 2; use the
 * <code>SPH_T32()</code> macro to ensure that the value is truncated
 * to exactly 32 bits. Unless otherwise specified, all macros and
 * functions which accept <code>sph_u32</code> values assume that these
 * values fit on 32 bits, i.e. do not exceed 2^32-1, even on architectures
 * where <code>sph_u32</code> is larger than that.
 */
typedef __arch_dependant__ sph_u32;

/** @hideinitializer
 * Signed integer type corresponding to <code>sph_u32</code>; it has
 * width 32 bits or more.
 */
typedef __arch_dependant__ sph_s32;

/** @hideinitializer
 * Unsigned integer type whose length is at least 64 bits; on most
 * architectures which feature such a type, it will have a width of
 * exactly 64 bits. C99-compliant platform will have this type; it
 * is also defined when the GNU compiler (gcc) is used, and on
 * platforms where <code>unsigned long</code> is large enough. If this
 * type is not available, then some hash functions which depends on
 * a 64-bit type will not be available (most notably SHA-384, SHA-512,
 * Tiger and WHIRLPOOL).
 */
typedef __arch_dependant__ sph_u64;

/** @hideinitializer
 * Signed integer type corresponding to <code>sph_u64</code>; it has
 * width 64 bits or more.
 */
typedef __arch_dependant__ sph_s64;
/**
 * 将标记<code>x</code>扩展为适当的<code>sph_u32</code>类型的常量表达式。根据这种类型的定义方式，可能会附加后缀，如<code>UL</code>。
 *
 * @param x   要扩展为适当的常量表达式的标记
 */
#define SPH_C32(x)

/**
 * 将32位值截断为精确的32位。在大多数系统上，这是一个空操作，编识器会识别这种情况。
 *
 * @param x   要截断的值（类型为<code>sph_u32</code>）
 */
#define SPH_T32(x)

/**
 * 将32位值左移指定位数。旋转计数必须介于1和31之间。此宏假定其第一个参数适合32位（在<code>sph_u32</code>更宽的机器上不允许额外的位）；两个参数可能会被多次评估。
 *
 * @param x   要旋转的值（类型为<code>sph_u32</code>）
 * @param n   旋转计数（介于1和31之间，包括1和31）
 */
#define SPH_ROTL32(x, n)

/**
 * 将32位值右移指定位数。旋转计数必须介于1和31之间。此宏假定其第一个参数适合32位（在<code>sph_u32</code>更宽的机器上不允许额外的位）；两个参数可能会被多次评估。
 *
 * @param x   要旋转的值（类型为<code>sph_u32</code>）
 * @param n   旋转计数（介于1和31之间，包括1和31）
 */
#define SPH_ROTR32(x, n)

/**
 * 在检测到64位类型的系统上定义此宏，并用于<code>sph_u64</code>。
 */
#define SPH_64

/**
 * 在“本机”整数大小为64位（64位值适合一个寄存器）的系统上定义此宏。
 */
#define SPH_64_TRUE
/**
 * 将标记<code>x</code>扩展为适当的常量表达式，类型为<code>sph_u64</code>。
 * 根据<code>sph_u64</code>的定义方式，可能会在参数后面添加后缀，如<code>ULL</code>。
 * 仅当检测到并使用了64位类型<code>sph_u64</code>时，才定义此宏。
 *
 * @param x   要扩展为适当的常量表达式的标记
 */
#define SPH_C64(x)

/**
 * 将64位值截断为确切的64位。在大多数系统上，这是一个无操作，由编译器识别。
 * 仅当检测到并使用了64位类型<code>sph_u64</code>时，才定义此宏。
 *
 * @param x   要截断的值（类型为<code>sph_u64</code>）
 */
#define SPH_T64(x)

/**
 * 将64位值向左旋转指定位数。旋转计数必须介于1和63之间。
 * 此宏假设其第一个参数适合64位（在<code>sph_u64</code>更宽的机器上不允许有额外的位）；
 * 两个参数都可以被多次计算。仅当检测到并使用了64位类型<code>sph_u64</code>时，才定义此宏。
 *
 * @param x   要旋转的值（类型为<code>sph_u64</code>）
 * @param n   旋转计数（介于1和63之间，包括1和63）
 */
#define SPH_ROTL64(x, n)

/**
 * 将64位值向右旋转指定位数。旋转计数必须介于1和63之间。
 * 此宏假设其第一个参数适合64位（在<code>sph_u64</code>更宽的机器上不允许有额外的位）；
 * 两个参数都可以被多次计算。仅当检测到并使用了64位类型<code>sph_u64</code>时，才定义此宏。
 *
 * @param x   要旋转的值（类型为<code>sph_u64</code>）
 * @param n   旋转计数（介于1和63之间，包括1和63）
 */
#define SPH_ROTR64(x, n)
/**
 * 如果编译平台支持，则该宏定义为<code>inline</code>或等效的构造，否则为空。
 * 用于声明内联函数，编译器会尝试将代码直接包含在调用者中。
 * 内联函数通常在头文件中定义，用作宏的替代。
 */
#define SPH_INLINE

/**
 * 如果检测到平台使用小端序，则定义该宏。
 * 这意味着<code>sph_u32</code>类型（如果定义了<code>sph_u64</code>类型也是如此）具有精确的宽度（即精确为32位或64位）。
 */
#define SPH_LITTLE_ENDIAN

/**
 * 如果检测到平台使用大端序，则定义该宏。
 * 这意味着<code>sph_u32</code>类型（如果定义了<code>sph_u64</code>类型也是如此）具有精确的宽度（即精确为32位或64位）。
 */
#define SPH_BIG_ENDIAN

/**
 * 如果在小端序约定中可以高效地从内存中读取和写入32位字（如果定义了64位字也是如此），则定义该宏。
 * 这适用于小端序平台，以及具有特殊小端序访问操作码的大端序平台（例如Ultrasparc）。
 */
#define SPH_LITTLE_FAST

/**
 * 如果在大端序约定中可以高效地从内存中读取和写入32位字（如果定义了64位字也是如此），则定义该宏。
 * 这适用于大端序平台，以及具有特殊大端序访问操作码的小端序平台。
 */
#define SPH_BIG_FAST

/**
 * 在某些平台上，该宏定义为无符号整数类型，可以将指针值转换为该类型。
 * 然后可以测试结果值是否是2、4或8的倍数，从而指示对齐指针，用于16位、32位或64位内存访问。
 */
#define SPH_UPTR
/**
 * 当定义了这个宏时，表示可以通过轻微的性能损失来进行非对齐内存访问，因此应该优先选择这种策略，而不是首先将数据复制到对齐的缓冲区。
 */
#define SPH_UNALIGNED

/**
 * 字节交换一个32位字（即<code>0x12345678</code>变成<code>0x78563412</code>）。这是一个内联函数，在某些平台上会使用内联汇编，以获得更好的性能。
 *
 * @param x   要进行字节交换的32位值
 * @return  字节交换后的值
 */
static inline sph_u32 sph_bswap32(sph_u32 x);

/**
 * 字节交换一个64位字。这是一个内联函数，在某些平台上会使用内联汇编，以获得更好的性能。只有在找到适合的64位类型<code>sph_u64</code>时才定义此函数。
 *
 * @param x   要进行字节交换的64位值
 * @return  字节交换后的值
 */
static inline sph_u64 sph_bswap64(sph_u64 x);

/**
 * 从内存中以小端约定（最低有效字节在前）解码一个16位无符号值。
 *
 * @param src   源地址
 * @return  解码后的值
 */
static inline unsigned sph_dec16le(const void *src);

/**
 * 将一个16位无符号值以小端约定（最低有效字节在前）编码到内存中。
 *
 * @param dst   目标缓冲区
 * @param val   要编码的值
 */
static inline void sph_enc16le(void *dst, unsigned val);

/**
 * 从内存中以大端约定（最高有效字节在前）解码一个16位无符号值。
 *
 * @param src   源地址
 * @return  解码后的值
 */
static inline unsigned sph_dec16be(const void *src);

/**
 * 将一个16位无符号值以大端约定（最高有效字节在前）编码到内存中。
 *
 * @param dst   目标缓冲区
 * @param val   要编码的值
 */
static inline void sph_enc16be(void *dst, unsigned val);
/**
 * 从内存中解码一个32位无符号值，使用小端序约定（最低有效字节在前）。
 *
 * @param src   源地址
 * @return  解码后的值
 */
static inline sph_u32 sph_dec32le(const void *src);

/**
 * 从内存中解码一个32位无符号值，使用小端序约定（最低有效字节在前）。此函数假设
 * 源地址适合直接访问，如果平台支持这样的操作；因此，它可能比通用的
 * <code>sph_dec32le()</code> 函数稍微快一些。
 *
 * @param src   源地址
 * @return  解码后的值
 */
static inline sph_u32 sph_dec32le_aligned(const void *src);

/**
 * 将一个32位无符号值编码到内存中，使用小端序约定（最低有效字节在前）。
 *
 * @param dst   目标缓冲区
 * @param val   要编码的值
 */
static inline void sph_enc32le(void *dst, sph_u32 val);

/**
 * 将一个32位无符号值编码到内存中，使用小端序约定（最低有效字节在前）。此函数假设
 * 目标地址适合直接访问，如果平台支持这样的操作；因此，它可能比通用的
 * <code>sph_enc32le()</code> 函数稍微快一些。
 *
 * @param dst   目标缓冲区
 * @param val   要编码的值
 */
static inline void sph_enc32le_aligned(void *dst, sph_u32 val);

/**
 * 从内存中解码一个32位无符号值，使用大端序约定（最高有效字节在前）。
 *
 * @param src   源地址
 * @return  解码后的值
 */
static inline sph_u32 sph_dec32be(const void *src);
# 从内存中解码一个32位无符号值，使用大端字节序（最高有效字节在前）。这个函数假设源地址适合直接访问，如果平台支持这样的操作，它可能比通用的`sph_dec32be()`函数稍微快一些。
# 参数src：源地址
# 返回值：解码后的值
static inline sph_u32 sph_dec32be_aligned(const void *src);

# 将一个32位无符号值编码到内存中，使用大端字节序（最高有效字节在前）。
# 参数dst：目标缓冲区
# 参数val：要编码的值
static inline void sph_enc32be(void *dst, sph_u32 val);

# 将一个32位无符号值编码到内存中，使用大端字节序（最高有效字节在前）。这个函数假设目标地址适合直接访问，如果平台支持这样的操作，它可能比通用的`sph_enc32be()`函数稍微快一些。
# 参数dst：目标缓冲区
# 参数val：要编码的值
static inline void sph_enc32be_aligned(void *dst, sph_u32 val);

# 从内存中解码一个64位无符号值，使用小端字节序（最低有效字节在前）。只有在检测到合适的64位类型并用于`sph_u64`时，才定义此函数。
# 参数src：源地址
# 返回值：解码后的值
static inline sph_u64 sph_dec64le(const void *src);
/**
 * 从内存中解码一个64位无符号值，按照小端序约定（最不重要的字节先出现）。这个函数假设源地址适合直接访问，如果平台支持这样的东西；因此，它可能比通用的sph_dec64le()函数稍微快一点。只有在检测到合适的64位类型并用于sph_u64时，才定义此函数。
 *
 * @param src   源地址
 * @return  解码后的值
 */
static inline sph_u64 sph_dec64le_aligned(const void *src);

/**
 * 将64位无符号值编码到内存中，按照小端序约定（最不重要的字节先出现）。只有在检测到合适的64位类型并用于sph_u64时，才定义此函数。
 *
 * @param dst   目标缓冲区
 * @param val   要编码的值
 */
static inline void sph_enc64le(void *dst, sph_u64 val);

/**
 * 将64位无符号值编码到内存中，按照小端序约定（最不重要的字节先出现）。这个函数假设目标地址适合直接访问，如果平台支持这样的东西；因此，它可能比通用的sph_enc64le()函数稍微快一点。只有在检测到合适的64位类型并用于sph_u64时，才定义此函数。
 *
 * @param dst   目标缓冲区
 * @param val   要编码的值
 */
static inline void sph_enc64le_aligned(void *dst, sph_u64 val);

/**
 * 从内存中解码一个64位无符号值，按照大端序约定（最重要的字节先出现）。只有在检测到合适的64位类型并用于sph_u64时，才定义此函数。
 *
 * @param src   源地址
 * @return  解码后的值
 */
static inline sph_u64 sph_dec64be(const void *src);
/**
 * Decode a 64-bit unsigned value from memory, in big-endian convention
 * (most significant byte comes first). This function assumes that the
 * source address is suitably aligned for a direct access, if the platform
 * supports such things; it can thus be marginally faster than the generic
 * <code>sph_dec64be()</code> function. This function is defined only
 * if a suitable 64-bit type was detected and used for <code>sph_u64</code>.
 *
 * @param src   the source address
 * @return  the decoded value
 */
static inline sph_u64 sph_dec64be_aligned(const void *src);

/**
 * Encode a 64-bit unsigned value into memory, in big-endian convention
 * (most significant byte comes first). This function is defined only
 * if a suitable 64-bit type was detected and used for <code>sph_u64</code>.
 *
 * @param dst   the destination buffer
 * @param val   the value to encode
 */
static inline void sph_enc64be(void *dst, sph_u64 val);

/**
 * Encode a 64-bit unsigned value into memory, in big-endian convention
 * (most significant byte comes first). This function assumes that the
 * destination address is suitably aligned for a direct access, if the
 * platform supports such things; it can thus be marginally faster than
 * the generic <code>sph_enc64be()</code> function. This function is defined
 * only if a suitable 64-bit type was detected and used for
 * <code>sph_u64</code>.
 *
 * @param dst   the destination buffer
 * @param val   the value to encode
 */
static inline void sph_enc64be_aligned(void *dst, sph_u64 val);

#endif

/* ============== END documentation block for Doxygen ============= */

#ifndef DOXYGEN_IGNORE

/*
 * We want to define the types "sph_u32" and "sph_u64" which hold
 * unsigned values of at least, respectively, 32 and 64 bits. These
 * tests should select appropriate types for most platforms. The
 * macro "SPH_64" is defined if the 64-bit is supported.
 */

#undef SPH_64
#undef SPH_64_TRUE

#if defined __STDC__ && __STDC_VERSION__ >= 199901L
/*
 * On C99 implementations, we can use <stdint.h> to get an exact 64-bit
 * type, if any, or otherwise use a wider type (which must exist, for
 * C99 conformance).
 */

#include <stdint.h>

#ifdef UINT32_MAX
typedef uint32_t sph_u32;  // 定义 32 位无符号整数类型
typedef int32_t sph_s32;    // 定义 32 位有符号整数类型
#else
typedef uint_fast32_t sph_u32;  // 定义至少 32 位宽度的无符号整数类型
typedef int_fast32_t sph_s32;    // 定义至少 32 位宽度的有符号整数类型
#endif
#if !SPH_NO_64
#ifdef UINT64_MAX
typedef uint64_t sph_u64;  // 定义 64 位无符号整数类型
typedef int64_t sph_s64;    // 定义 64 位有符号整数类型
#else
typedef uint_fast64_t sph_u64;  // 定义至少 64 位宽度的无符号整数类型
typedef int_fast64_t sph_s64;    // 定义至少 64 位宽度的有符号整数类型
#endif
#endif

#define SPH_C32(x)    ((sph_u32)(x))  // 将 x 转换为 sph_u32 类型
#if !SPH_NO_64
#define SPH_C64(x)    ((sph_u64)(x))  // 将 x 转换为 sph_u64 类型
#define SPH_64  1  // 定义 64 位整数类型标志
#endif

#else

/*
 * On non-C99 systems, we use "unsigned int" if it is wide enough,
 * "unsigned long" otherwise. This supports all "reasonable" architectures.
 * We have to be cautious: pre-C99 preprocessors handle constants
 * differently in '#if' expressions. Hence the shifts to test UINT_MAX.
 */

#if ((UINT_MAX >> 11) >> 11) >= 0x3FF

typedef unsigned int sph_u32;  // 定义无符号整数类型
typedef int sph_s32;  // 定义有符号整数类型

#define SPH_C32(x)    ((sph_u32)(x ## U))  // 将 x 转换为 sph_u32 类型

#else

typedef unsigned long sph_u32;  // 定义长整数类型
typedef long sph_s32;  // 定义有符号整数类型

#define SPH_C32(x)    ((sph_u32)(x ## UL))  // 将 x 转换为 sph_u32 类型

#endif

#if !SPH_NO_64

/*
 * We want a 64-bit type. We use "unsigned long" if it is wide enough (as
 * is common on 64-bit architectures such as AMD64, Alpha or Sparcv9),
 * "unsigned long long" otherwise, if available. We use ULLONG_MAX to
 * test whether "unsigned long long" is available; we also know that
 * gcc features this type, even if the libc header do not know it.
 */

#if ((ULONG_MAX >> 31) >> 31) >= 3

typedef unsigned long sph_u64;  // 定义长整数类型
typedef long sph_s64;  // 定义有符号整数类型

#define SPH_C64(x)    ((sph_u64)(x ## UL))  // 将 x 转换为 sph_u64 类型

#define SPH_64  1  // 定义 64 位整数类型标志

#elif ((ULLONG_MAX >> 31) >> 31) >= 3 || defined __GNUC__

typedef unsigned long long sph_u64;  // 定义长长整数类型
typedef long long sph_s64;  // 定义有符号长长整数类型

#define SPH_C64(x)    ((sph_u64)(x ## ULL))  // 将 x 转换为 sph_u64 类型

#define SPH_64  1  // 定义 64 位整数类型标志

#else

/*
 * No 64-bit type...
 */

#endif

#endif

#endif
/*
 * 如果 "unsigned long" 类型的长度为 64 位或更多，则这是一个"真正"的 64 位架构。
 * 即使在 amd64 上，Visual C 中的 "long" 类型被限制为 32 位，这也是真实的情况。
 */
#if SPH_64 && (((ULONG_MAX >> 31) >> 31) >= 3 || defined _M_X64)
#define SPH_64_TRUE   1
#endif

/*
 * 实现说明：一些处理器具有特定的操作码来执行旋转。
 * 最近版本的 gcc 识别上面的表达式，并在适当时使用相关的操作码。
 */
#define SPH_T32(x)    ((x) & SPH_C32(0xFFFFFFFF))
#define SPH_ROTL32(x, n)   SPH_T32(((x) << (n)) | ((x) >> (32 - (n)))
#define SPH_ROTR32(x, n)   SPH_ROTL32(x, (32 - (n)))

#if SPH_64

#define SPH_T64(x)    ((x) & SPH_C64(0xFFFFFFFFFFFFFFFF))
#define SPH_ROTL64(x, n)   SPH_T64(((x) << (n)) | ((x) >> (64 - (n)))
#define SPH_ROTR64(x, n)   SPH_ROTL64(x, (64 - (n)))

#endif

#ifndef DOXYGEN_IGNORE
/*
 * 如果可用，将 SPH_INLINE 定义为 "inline" 限定符。
 * 我们定义了一些小的类似宏的函数，这些函数非常适合被内联。
 */
#if (defined __STDC__ && __STDC_VERSION__ >= 199901L) || defined __GNUC__
#define SPH_INLINE inline
#elif defined _MSC_VER
#define SPH_INLINE __inline
#else
#define SPH_INLINE
#endif
#endif
/*
 * We define some macros which qualify the architecture. These macros
 * may be explicit set externally (e.g. as compiler parameters). The
 * code below sets those macros if they are not already defined.
 *
 * Most macros are boolean, thus evaluate to either zero or non-zero.
 * The SPH_UPTR macro is special, in that it evaluates to a C type,
 * or is not defined.
 *
 * SPH_UPTR             if defined: unsigned type to cast pointers into
 *
 * SPH_UNALIGNED        non-zero if unaligned accesses are efficient
 * SPH_LITTLE_ENDIAN    non-zero if architecture is known to be little-endian
 * SPH_BIG_ENDIAN       non-zero if architecture is known to be big-endian
 * SPH_LITTLE_FAST      non-zero if little-endian decoding is fast
 * SPH_BIG_FAST         non-zero if big-endian decoding is fast
 *
 * If SPH_UPTR is defined, then encoding and decoding of 32-bit and 64-bit
 * values will try to be "smart". Either SPH_LITTLE_ENDIAN or SPH_BIG_ENDIAN
 * _must_ be non-zero in those situations. The 32-bit and 64-bit types
 * _must_ also have an exact width.
 *
 * SPH_SPARCV9_GCC_32   UltraSPARC-compatible with gcc, 32-bit mode
 * SPH_SPARCV9_GCC_64   UltraSPARC-compatible with gcc, 64-bit mode
 * SPH_SPARCV9_GCC      UltraSPARC-compatible with gcc
 * SPH_I386_GCC         x86-compatible (32-bit) with gcc
 * SPH_I386_MSVC        x86-compatible (32-bit) with Microsoft Visual C
 * SPH_AMD64_GCC        x86-compatible (64-bit) with gcc
 * SPH_AMD64_MSVC       x86-compatible (64-bit) with Microsoft Visual C
 * SPH_PPC32_GCC        PowerPC, 32-bit, with gcc
 * SPH_PPC64_GCC        PowerPC, 64-bit, with gcc
 *
 * TODO: enhance automatic detection, for more architectures and compilers.
 * Endianness is the most important. SPH_UNALIGNED and SPH_UPTR help with
 * some very fast functions (e.g. MD4) when using unaligned input data.
 * The CPU-specific-with-GCC macros are useful only for inline assembly,
 * normally restrained to this header file.
 */
/*
 * 32-bit x86, aka "i386 compatible".
 */
#if defined __i386__ || defined _M_IX86
// 如果是 32 位 x86 架构，定义 SPH_DETECT_UNALIGNED 为 1
#define SPH_DETECT_UNALIGNED         1
// 如果是 32 位 x86 架构，定义 SPH_DETECT_LITTLE_ENDIAN 为 1
#define SPH_DETECT_LITTLE_ENDIAN     1
// 如果是 32 位 x86 架构，定义 SPH_DETECT_UPTR 为 sph_u32
#define SPH_DETECT_UPTR              sph_u32
#ifdef __GNUC__
// 如果是 32 位 x86 架构，并且是使用 GCC 编译器，定义 SPH_DETECT_I386_GCC 为 1
#define SPH_DETECT_I386_GCC          1
#endif
#ifdef _MSC_VER
// 如果是 32 位 x86 架构，并且是使用 MSVC 编译器，定义 SPH_DETECT_I386_MSVC 为 1
#define SPH_DETECT_I386_MSVC         1
#endif

/*
 * 64-bit x86, hereafter known as "amd64".
 */
#elif defined __x86_64 || defined _M_X64
// 如果是 64 位 x86 架构，定义 SPH_DETECT_UNALIGNED 为 1
#define SPH_DETECT_UNALIGNED         1
// 如果是 64 位 x86 架构，定义 SPH_DETECT_LITTLE_ENDIAN 为 1
#define SPH_DETECT_LITTLE_ENDIAN     1
// 如果是 64 位 x86 架构，定义 SPH_DETECT_UPTR 为 sph_u64
#define SPH_DETECT_UPTR              sph_u64
#ifdef __GNUC__
// 如果是 64 位 x86 架构，并且是使用 GCC 编译器，定义 SPH_DETECT_AMD64_GCC 为 1
#define SPH_DETECT_AMD64_GCC         1
#endif
#ifdef _MSC_VER
// 如果是 64 位 x86 架构，并且是使用 MSVC 编译器，定义 SPH_DETECT_AMD64_MSVC 为 1
#define SPH_DETECT_AMD64_MSVC        1
#endif

/*
 * 64-bit Sparc architecture (implies v9).
 */
#elif ((defined __sparc__ || defined __sparc) && defined __arch64__) \
    || defined __sparcv9
// 如果是 64 位 Sparc 架构，定义 SPH_DETECT_BIG_ENDIAN 为 1
#define SPH_DETECT_BIG_ENDIAN        1
// 如果是 64 位 Sparc 架构，定义 SPH_DETECT_UPTR 为 sph_u64
#define SPH_DETECT_UPTR              sph_u64
#ifdef __GNUC__
// 如果是 64 位 Sparc 架构，并且是使用 GCC 编译器，定义 SPH_DETECT_SPARCV9_GCC_64 为 1
#define SPH_DETECT_SPARCV9_GCC_64    1
// 如果是 64 位 Sparc 架构，并且是使用 GCC 编译器，定义 SPH_DETECT_LITTLE_FAST 为 1
#define SPH_DETECT_LITTLE_FAST       1
#endif

/*
 * 32-bit Sparc.
 */
#elif (defined __sparc__ || defined __sparc) \
    && !(defined __sparcv9 || defined __arch64__)
// 如果是 32 位 Sparc 架构，定义 SPH_DETECT_BIG_ENDIAN 为 1
#define SPH_DETECT_BIG_ENDIAN        1
// 如果是 32 位 Sparc 架构，定义 SPH_DETECT_UPTR 为 sph_u32
#define SPH_DETECT_UPTR              sph_u32
#if defined __GNUC__ && defined __sparc_v9__
// 如果是 32 位 Sparc 架构，并且是使用 GCC 编译器，并且是 v9 架构，定义 SPH_DETECT_SPARCV9_GCC_32 为 1
#define SPH_DETECT_SPARCV9_GCC_32    1
// 如果是 32 位 Sparc 架构，并且是使用 GCC 编译器，并且是 v9 架构，定义 SPH_DETECT_LITTLE_FAST 为 1
#define SPH_DETECT_LITTLE_FAST       1
#endif

/*
 * ARM, little-endian.
 */
#elif defined __arm__ && __ARMEL__
// 如果是 ARM 架构，并且是小端序，定义 SPH_DETECT_LITTLE_ENDIAN 为 1

/*
 * MIPS, little-endian.
 */
#elif MIPSEL || _MIPSEL || __MIPSEL || __MIPSEL__
// 如果是 MIPS 架构，并且是小端序，定义 SPH_DETECT_LITTLE_ENDIAN 为 1

/*
 * MIPS, big-endian.
 */
#elif MIPSEB || _MIPSEB || __MIPSEB || __MIPSEB__
// 如果是 MIPS 架构，并且是大端序，定义 SPH_DETECT_BIG_ENDIAN 为 1

/*
 * PowerPC.
 */
#elif defined __powerpc__ || defined __POWERPC__ || defined __ppc__ \
    || defined _ARCH_PPC
/*
 * 注意：我们不将跨端访问声明为“快速”：即使使用内联汇编，实现仍应假定将解码后的字
 *      保存在临时变量中比再次解码更快。
 */
#if defined __GNUC__
#if SPH_64_TRUE
#define SPH_DETECT_PPC64_GCC         1
#else
#define SPH_DETECT_PPC32_GCC         1
#endif
#endif

#if defined __BIG_ENDIAN__ || defined _BIG_ENDIAN
#define SPH_DETECT_BIG_ENDIAN        1
#elif defined __LITTLE_ENDIAN__ || defined _LITTLE_ENDIAN
#define SPH_DETECT_LITTLE_ENDIAN     1
#endif

/*
 * Itanium, 64-bit.
 */
#elif defined __ia64 || defined __ia64__ \
    || defined __itanium__ || defined _M_IA64

#if defined __BIG_ENDIAN__ || defined _BIG_ENDIAN
#define SPH_DETECT_BIG_ENDIAN        1
#else
#define SPH_DETECT_LITTLE_ENDIAN     1
#endif
#if defined __LP64__ || defined _LP64
#define SPH_DETECT_UPTR              sph_u64
#else
#define SPH_DETECT_UPTR              sph_u32
#endif

#endif

#if defined SPH_DETECT_SPARCV9_GCC_32 || defined SPH_DETECT_SPARCV9_GCC_64
#define SPH_DETECT_SPARCV9_GCC       1
#endif
#if defined SPH_DETECT_UNALIGNED && !defined SPH_UNALIGNED
#define SPH_UNALIGNED         SPH_DETECT_UNALIGNED
#endif
#if defined SPH_DETECT_UPTR && !defined SPH_UPTR
#define SPH_UPTR              SPH_DETECT_UPTR
#endif
#if defined SPH_DETECT_LITTLE_ENDIAN && !defined SPH_LITTLE_ENDIAN
#define SPH_LITTLE_ENDIAN     SPH_DETECT_LITTLE_ENDIAN
#endif
#if defined SPH_DETECT_BIG_ENDIAN && !defined SPH_BIG_ENDIAN
#define SPH_BIG_ENDIAN        SPH_DETECT_BIG_ENDIAN
#endif
#if defined SPH_DETECT_LITTLE_FAST && !defined SPH_LITTLE_FAST
#define SPH_LITTLE_FAST       SPH_DETECT_LITTLE_FAST
#endif
#if defined SPH_DETECT_BIG_FAST && !defined SPH_BIG_FAST
#define SPH_BIG_FAST    SPH_DETECT_BIG_FAST
#endif
#if defined SPH_DETECT_SPARCV9_GCC_32 && !defined SPH_SPARCV9_GCC_32
#define SPH_SPARCV9_GCC_32    SPH_DETECT_SPARCV9_GCC_32
#endif
#if defined SPH_DETECT_SPARCV9_GCC_64 && !defined SPH_SPARCV9_GCC_64
# 定义 SPH_SPARCV9_GCC_64 为 SPH_DETECT_SPARCV9_GCC_64
#define SPH_SPARCV9_GCC_64    SPH_DETECT_SPARCV9_GCC_64
#endif
# 如果定义了 SPH_DETECT_SPARCV9_GCC 且未定义 SPH_SPARCV9_GCC，则将 SPH_DETECT_SPARCV9_GCC 赋值给 SPH_SPARCV9_GCC
#if defined SPH_DETECT_SPARCV9_GCC && !defined SPH_SPARCV9_GCC
#define SPH_SPARCV9_GCC       SPH_DETECT_SPARCV9_GCC
#endif
# 如果定义了 SPH_DETECT_I386_GCC 且未定义 SPH_I386_GCC，则将 SPH_DETECT_I386_GCC 赋值给 SPH_I386_GCC
#if defined SPH_DETECT_I386_GCC && !defined SPH_I386_GCC
#define SPH_I386_GCC          SPH_DETECT_I386_GCC
#endif
# 如果定义了 SPH_DETECT_I386_MSVC 且未定义 SPH_I386_MSVC，则将 SPH_DETECT_I386_MSVC 赋值给 SPH_I386_MSVC
#if defined SPH_DETECT_I386_MSVC && !defined SPH_I386_MSVC
#define SPH_I386_MSVC         SPH_DETECT_I386_MSVC
#endif
# 如果定义了 SPH_DETECT_AMD64_GCC 且未定义 SPH_AMD64_GCC，则将 SPH_DETECT_AMD64_GCC 赋值给 SPH_AMD64_GCC
#if defined SPH_DETECT_AMD64_GCC && !defined SPH_AMD64_GCC
#define SPH_AMD64_GCC         SPH_DETECT_AMD64_GCC
#endif
# 如果定义了 SPH_DETECT_AMD64_MSVC 且未定义 SPH_AMD64_MSVC，则将 SPH_DETECT_AMD64_MSVC 赋值给 SPH_AMD64_MSVC
#if defined SPH_DETECT_AMD64_MSVC && !defined SPH_AMD64_MSVC
#define SPH_AMD64_MSVC        SPH_DETECT_AMD64_MSVC
#endif
# 如果定义了 SPH_DETECT_PPC32_GCC 且未定义 SPH_PPC32_GCC，则将 SPH_DETECT_PPC32_GCC 赋值给 SPH_PPC32_GCC
#if defined SPH_DETECT_PPC32_GCC && !defined SPH_PPC32_GCC
#define SPH_PPC32_GCC         SPH_DETECT_PPC32_GCC
#endif
# 如果定义了 SPH_DETECT_PPC64_GCC 且未定义 SPH_PPC64_GCC，则将 SPH_DETECT_PPC64_GCC 赋值给 SPH_PPC64_GCC
#if defined SPH_DETECT_PPC64_GCC && !defined SPH_PPC64_GCC
#define SPH_PPC64_GCC         SPH_DETECT_PPC64_GCC
#endif

# 如果 SPH_LITTLE_ENDIAN 被定义且 SPH_LITTLE_FAST 未被定义，则将 SPH_LITTLE_FAST 定义为 1
#if SPH_LITTLE_ENDIAN && !defined SPH_LITTLE_FAST
#define SPH_LITTLE_FAST              1
#endif
# 如果 SPH_BIG_ENDIAN 被定义且 SPH_BIG_FAST 未被定义，则将 SPH_BIG_FAST 定义为 1
#if SPH_BIG_ENDIAN && !defined SPH_BIG_FAST
#define SPH_BIG_FAST                 1
#endif

# 如果定义了 SPH_UPTR 且 SPH_LITTLE_ENDIAN 和 SPH_BIG_ENDIAN 都未被定义，则抛出错误
#if defined SPH_UPTR && !(SPH_LITTLE_ENDIAN || SPH_BIG_ENDIAN)
#error SPH_UPTR defined, but endianness is not known.
#endif

# 如果 SPH_I386_GCC 被定义且 SPH_NO_ASM 未被定义
#if SPH_I386_GCC && !SPH_NO_ASM

'''
 * 在 x86 32 位平台上，使用 gcc，我们使用 bswapl 指令来对 32 位值进行字节交换。
'''

# 定义一个内联函数 sph_bswap32，使用 bswapl 指令对 32 位值进行字节交换
static SPH_INLINE sph_u32
sph_bswap32(sph_u32 x)
{
    __asm__ __volatile__ ("bswapl %0" : "=r" (x) : "0" (x));
    return x;
}

# 如果 SPH_64 被定义
#if SPH_64

# 定义一个内联函数 sph_bswap64，使用 sph_bswap32 对 64 位值进行字节交换
static SPH_INLINE sph_u64
sph_bswap64(sph_u64 x)
{
    return ((sph_u64)sph_bswap32((sph_u32)x) << 32)
        | (sph_u64)sph_bswap32((sph_u32)(x >> 32));
}

#endif

# 如果 SPH_AMD64_GCC 被定义且 SPH_NO_ASM 未被定义
#elif SPH_AMD64_GCC && !SPH_NO_ASM

'''
 * 在 x86 64 位平台上，使用 gcc，我们使用 bswapl 指令来对 32 位和 64 位值进行字节交换。
'''

# 定义一个内联函数 sph_bswap32，使用 bswapl 指令对 32 位值进行字节交换
static SPH_INLINE sph_u32
sph_bswap32(sph_u32 x)
{
    __asm__ __volatile__ ("bswapl %0" : "=r" (x) : "0" (x));
    return x;
}

# 如果 SPH_64 被定义
#if SPH_64

# 定义一个内联函数 sph_bswap64，使用 sph_bswap32 对 64 位值进行字节交换
static SPH_INLINE sph_u64
sph_bswap64(sph_u64 x)
{
    # 使用内联汇编语言实现将寄存器中的数据进行字节顺序翻转
    __asm__ __volatile__ ("bswapq %0" : "=r" (x) : "0" (x));
    # 返回翻转后的数据
    return x;
#else

static SPH_INLINE sph_u32
sph_bswap32(sph_u32 x)
{
    x = SPH_T32((x << 16) | (x >> 16));
    x = ((x & SPH_C32(0xFF00FF00)) >> 8)
        | ((x & SPH_C32(0x00FF00FF)) << 8);
    return x;
}

#if SPH_64

/**
 * Byte-swap a 64-bit value.
 *
 * @param x   the input value
 * @return  the byte-swapped value
 */
static SPH_INLINE sph_u64
sph_bswap64(sph_u64 x)
{
    x = SPH_T64((x << 32) | (x >> 32));
    x = ((x & SPH_C64(0xFFFF0000FFFF0000)) >> 16)
        | ((x & SPH_C64(0x0000FFFF0000FFFF)) << 16);
    x = ((x & SPH_C64(0xFF00FF00FF00FF00)) >> 8)
        | ((x & SPH_C64(0x00FF00FF00FF00FF)) << 8);
    return x;
}

#endif

#endif



#else

static SPH_INLINE sph_u32
sph_bswap32(sph_u32 x)
{
    x = SPH_T32((x << 16) | (x >> 16));
    x = ((x & SPH_C32(0xFF00FF00)) >> 8)
        | ((x & SPH_C32(0x00FF00FF)) << 8);
    return x;
}

这段代码是一个条件语句的分支，当条件不满足时执行。它定义了一个名为`sph_bswap32`的静态内联函数，该函数接受一个`sph_u32`类型的参数`x`，并返回一个`sph_u32`类型的值。函数的作用是将`x`进行字节交换，即将高位字节和低位字节进行交换。具体实现如下：
- 首先，将`x`左移16位，然后将结果与`x`右移16位进行或运算，得到一个新的值赋给`x`。
- 然后，将`x`与`0xFF00FF00`进行按位与运算，再将结果右移8位，得到一个新的值。
- 最后，将`x`与`0x00FF00FF`进行按位与运算，再将结果左移8位，得到一个新的值。
- 返回`x`。


#if SPH_64

/**
 * Byte-swap a 64-bit value.
 *
 * @param x   the input value
 * @return  the byte-swapped value
 */
static SPH_INLINE sph_u64
sph_bswap64(sph_u64 x)
{
    x = SPH_T64((x << 32) | (x >> 32));
    x = ((x & SPH_C64(0xFFFF0000FFFF0000)) >> 16)
        | ((x & SPH_C64(0x0000FFFF0000FFFF)) << 16);
    x = ((x & SPH_C64(0xFF00FF00FF00FF00)) >> 8)
        | ((x & SPH_C64(0x00FF00FF00FF00FF)) << 8);
    return x;
}

#endif

这段代码是一个条件语句的分支，当条件满足时执行。它定义了一个名为`sph_bswap64`的静态内联函数，该函数接受一个`sph_u64`类型的参数`x`，并返回一个`sph_u64`类型的值。函数的作用是将`x`进行字节交换，即将高位字节和低位字节进行交换。具体实现如下：
- 首先，将`x`左移32位，然后将结果与`x`右移32位进行或运算，得到一个新的值赋给`x`。
- 然后，将`x`与`0xFFFF0000FFFF0000`进行按位与运算，再将结果右移16位，得到一个新的值。
- 接着，将`x`与`0x0000FFFF0000FFFF`进行按位与运算，再将结果左移16位，得到一个新的值。
- 最后，将`x`与`0xFF00FF00FF00FF00`进行按位与运算，再将结果右移8位，得到一个新的值。
- 返回`x`。
/*
 * On UltraSPARC systems, native ordering is big-endian, but it is
 * possible to perform little-endian read accesses by specifying the
 * address space 0x88 (ASI_PRIMARY_LITTLE). Basically, either we use
 * the opcode "lda [%reg]0x88,%dst", where %reg is the register which
 * contains the source address and %dst is the destination register,
 * or we use "lda [%reg+imm]%asi,%dst", which uses the %asi register
 * to get the address space name. The latter format is better since it
 * combines an addition and the actual access in a single opcode; but
 * it requires the setting (and subsequent resetting) of %asi, which is
 * slow. Some operations (i.e. MD5 compression function) combine many
 * successive little-endian read accesses, which may share the same
 * %asi setting. The macros below contain the appropriate inline
 * assembly.
 */

# 定义一个宏，用于设置 ASI 寄存器为 0x88
#define SPH_SPARCV9_SET_ASI   \
    sph_u32 sph_sparcv9_asi; \
    __asm__ __volatile__ ( \
        "rd %%asi,%0\n\twr %%g0,0x88,%%asi" : "=r" (sph_sparcv9_asi));

# 定义一个宏，用于将 ASI 寄存器重置为之前的值
#define SPH_SPARCV9_RESET_ASI  \
    __asm__ __volatile__ ("wr %%g0,%0,%%asi" : : "r" (sph_sparcv9_asi));

# 定义一个宏，用于进行 32 位小端读取
#define SPH_SPARCV9_DEC32LE(base, idx)   ({ \
        sph_u32 sph_sparcv9_tmp; \
        __asm__ __volatile__ ("lda [%1+" #idx "*4]%%asi,%0" \
            : "=r" (sph_sparcv9_tmp) : "r" (base)); \
        sph_sparcv9_tmp; \
    })

#endif

# 定义一个内联函数，用于将 16 位无符号整数以大端字节序写入目标地址
static SPH_INLINE void
sph_enc16be(void *dst, unsigned val)
{
    ((unsigned char *)dst)[0] = (val >> 8);
    ((unsigned char *)dst)[1] = val;
}

# 定义一个内联函数，用于从源地址以大端字节序读取 16 位无符号整数
static SPH_INLINE unsigned
sph_dec16be(const void *src)
{
    return ((unsigned)(((const unsigned char *)src)[0]) << 8)
        | (unsigned)(((const unsigned char *)src)[1]);
}

# 定义一个内联函数，用于将 16 位无符号整数以小端字节序写入目标地址
static SPH_INLINE void
sph_enc16le(void *dst, unsigned val)
{
    ((unsigned char *)dst)[0] = val;
    ((unsigned char *)dst)[1] = val >> 8;
}

# 定义一个内联函数，用于从源地址以小端字节序读取 16 位无符号整数
static SPH_INLINE unsigned
sph_dec16le(const void *src)
{
    # 将src指针指向的内存中的两个字节转换为一个无符号整数并返回
    return (unsigned)(((const unsigned char *)src)[0])
        | ((unsigned)(((const unsigned char *)src)[1]) << 8);
}

/**
 * 将一个32位值编码到提供的缓冲区中（大端序约定）。
 *
 * @param dst   目标缓冲区
 * @param val   要编码的32位值
 */
static SPH_INLINE void
sph_enc32be(void *dst, sph_u32 val)
{
#if defined SPH_UPTR
#if SPH_UNALIGNED
#if SPH_LITTLE_ENDIAN
    val = sph_bswap32(val);  // 如果是小端序，进行字节交换
#endif
    *(sph_u32 *)dst = val;  // 将值写入目标缓冲区
#else
    if (((SPH_UPTR)dst & 3) == 0) {  // 如果目标缓冲区是32位对齐的
#if SPH_LITTLE_ENDIAN
        val = sph_bswap32(val);  // 如果是小端序，进行字节交换
#endif
        *(sph_u32 *)dst = val;  // 将值写入目标缓冲区
    } else {
        ((unsigned char *)dst)[0] = (val >> 24);  // 否则，按大端序逐字节写入
        ((unsigned char *)dst)[1] = (val >> 16);
        ((unsigned char *)dst)[2] = (val >> 8);
        ((unsigned char *)dst)[3] = val;
    }
#endif
#else
    ((unsigned char *)dst)[0] = (val >> 24);  // 按大端序逐字节写入
    ((unsigned char *)dst)[1] = (val >> 16);
    ((unsigned char *)dst)[2] = (val >> 8);
    ((unsigned char *)dst)[3] = val;
#endif
}

/**
 * 将一个32位值编码到提供的缓冲区中（大端序约定）。
 * 目标缓冲区必须正确对齐。
 *
 * @param dst   目标缓冲区（32位对齐）
 * @param val   要编码的值
 */
static SPH_INLINE void
sph_enc32be_aligned(void *dst, sph_u32 val)
{
#if SPH_LITTLE_ENDIAN
    *(sph_u32 *)dst = sph_bswap32(val);  // 如果是小端序，进行字节交换后写入
#elif SPH_BIG_ENDIAN
    *(sph_u32 *)dst = val;  // 如果是大端序，直接写入
#else
    ((unsigned char *)dst)[0] = (val >> 24);  // 按大端序逐字节写入
    ((unsigned char *)dst)[1] = (val >> 16);
    ((unsigned char *)dst)[2] = (val >> 8);
    ((unsigned char *)dst)[3] = val;
#endif
}

/**
 * 从提供的缓冲区中解码一个32位值（大端序约定）。
 *
 * @param src   源缓冲区
 * @return  解码后的值
 */
static SPH_INLINE sph_u32
sph_dec32be(const void *src)
{
#if defined SPH_UPTR
#if SPH_UNALIGNED
#if SPH_LITTLE_ENDIAN
    return sph_bswap32(*(const sph_u32 *)src);  // 如果是小端序，进行字节交换后返回
#else
    return *(const sph_u32 *)src;  // 如果是大端序，直接返回
#endif
#else
    if (((SPH_UPTR)src & 3) == 0) {  // 如果源缓冲区是32位对齐的
#if SPH_LITTLE_ENDIAN
        return sph_bswap32(*(const sph_u32 *)src);  // 如果是小端序，进行字节交换后返回
/**
 * 如果是大端序，直接返回源缓冲区的值
 * 如果是小端序，按照大端序的方式解码缓冲区中的值
 * 
 * @param src   源缓冲区（32位对齐）
 * @return  解码后的值
 */
static SPH_INLINE sph_u32
sph_dec32be_aligned(const void *src)
{
#if SPH_LITTLE_ENDIAN
    // 如果是小端序，按照大端序的方式解码缓冲区中的值
    return sph_bswap32(*(const sph_u32 *)src);
#elif SPH_BIG_ENDIAN
    // 如果是大端序，直接返回源缓冲区的值
    return *(const sph_u32 *)src;
#else
    // 如果是未知序，按照大端序的方式解码缓冲区中的值
    return ((sph_u32)(((const unsigned char *)src)[0]) << 24)
        | ((sph_u32)(((const unsigned char *)src)[1]) << 16)
        | ((sph_u32)(((const unsigned char *)src)[2]) << 8)
        | (sph_u32)(((const unsigned char *)src)[3]);
#endif
}
/**
 * 将32位值按照小端序的方式编码到目标缓冲区中
 * 
 * @param dst   目标缓冲区
 * @param val   要编码的32位值
 */
static SPH_INLINE void
sph_enc32le(void *dst, sph_u32 val)
{
#if defined SPH_UPTR
#if SPH_UNALIGNED
#if SPH_BIG_ENDIAN
    // 如果是大端序，交换字节顺序
    val = sph_bswap32(val);
#endif
    // 将值写入目标缓冲区
    *(sph_u32 *)dst = val;
#else
    if (((SPH_UPTR)dst & 3) == 0) {
#if SPH_BIG_ENDIAN
        // 如果是大端序，交换字节顺序
        val = sph_bswap32(val);
#endif
        // 将值写入目标缓冲区
        *(sph_u32 *)dst = val;
    } else {
        // 如果目标缓冲区未对齐，按照小端序的方式逐字节写入值
        ((unsigned char *)dst)[0] = val;
        ((unsigned char *)dst)[1] = (val >> 8);
        ((unsigned char *)dst)[2] = (val >> 16);
        ((unsigned char *)dst)[3] = (val >> 24);
    }
#endif
#else
    // 如果未定义 SPH_UPTR，按照小端序的方式逐字节写入值
    ((unsigned char *)dst)[0] = val;
    # 将32位整数val的高8位存储到dst的第2个字节
    ((unsigned char *)dst)[1] = (val >> 8);
    # 将32位整数val的高16位存储到dst的第3个字节
    ((unsigned char *)dst)[2] = (val >> 16);
    # 将32位整数val的高24位存储到dst的第4个字节
    ((unsigned char *)dst)[3] = (val >> 24);
#endif
}

/**
 * 将一个32位的值编码到提供的缓冲区中（小端字节序约定）。
 * 目标缓冲区必须正确对齐。
 *
 * @param dst   目标缓冲区（32位对齐）
 * @param val   要编码的值
 */
static SPH_INLINE void
sph_enc32le_aligned(void *dst, sph_u32 val)
{
#if SPH_LITTLE_ENDIAN
    // 如果是小端字节序，直接将值写入目标缓冲区
    *(sph_u32 *)dst = val;
#elif SPH_BIG_ENDIAN
    // 如果是大端字节序，先将值进行字节交换，再写入目标缓冲区
    *(sph_u32 *)dst = sph_bswap32(val);
#else
    // 如果是未知字节序，按照小端字节序将值写入目标缓冲区
    ((unsigned char *)dst)[0] = val;
    ((unsigned char *)dst)[1] = (val >> 8);
    ((unsigned char *)dst)[2] = (val >> 16);
    ((unsigned char *)dst)[3] = (val >> 24);
#endif
}

/**
 * 从提供的缓冲区中解码一个32位的值（小端字节序约定）。
 *
 * @param src   源缓冲区
 * @return  解码后的值
 */
static SPH_INLINE sph_u32
sph_dec32le(const void *src)
{
#if defined SPH_UPTR
#if SPH_UNALIGNED
#if SPH_BIG_ENDIAN
    // 如果是大端字节序，先进行字节交换，再读取值
    return sph_bswap32(*(const sph_u32 *)src);
#else
    // 如果是小端字节序，直接读取值
    return *(const sph_u32 *)src;
#endif
#else
    // 如果缓冲区未对齐
    if (((SPH_UPTR)src & 3) == 0) {
#if SPH_BIG_ENDIAN
#if SPH_SPARCV9_GCC && !SPH_NO_ASM
        sph_u32 tmp;

        /*
         * "__volatile__" is needed here because without it,
         * gcc-3.4.3 miscompiles the code and performs the
         * access before the test on the address, thus triggering
         * a bus error...
         */
        // 使用汇编指令从源缓冲区读取值
        __asm__ __volatile__ (
            "lda [%1]0x88,%0" : "=r" (tmp) : "r" (src));
        return tmp;
/**
 * 从提供的缓冲区解码一个32位的值（小端字节序）。
 * 源缓冲区必须正确对齐。
 *
 * @param src   源缓冲区（32位对齐）
 * @return  解码后的值
 */
static SPH_INLINE sph_u32
sph_dec32le_aligned(const void *src)
{
#if SPH_LITTLE_ENDIAN
    // 如果是小端字节序，直接返回源缓冲区的值
    return *(const sph_u32 *)src;
#elif SPH_BIG_ENDIAN
#if SPH_SPARCV9_GCC && !SPH_NO_ASM
    // 如果是大端字节序且是 SPARCv9 架构，使用内联汇编指令 lda 从源缓冲区读取值
    sph_u32 tmp;

    __asm__ __volatile__ ("lda [%1]0x88,%0" : "=r" (tmp) : "r" (src));
    return tmp;
/*
 * Not worth it generally.
 * 一般情况下不值得这样做。
 *
#elif (SPH_PPC32_GCC || SPH_PPC64_GCC) && !SPH_NO_ASM
    # 如果是 PPC32 或 PPC64 架构，并且不禁用汇编，则执行以下代码
    sph_u32 tmp;
    # 声明一个无符号 32 位整数 tmp

    __asm__ __volatile__ ("lwbrx %0,0,%1" : "=r" (tmp) : "r" (src));
    # 使用汇编指令 lwbrx 从 src 地址读取一个字，并存储到 tmp 中
    return tmp;
    # 返回 tmp 的值
 */
#else
    # 如果不满足上述条件，则执行以下代码
    return sph_bswap32(*(const sph_u32 *)src);
    # 返回 src 指向的地址处的 32 位整数值经过字节交换后的结果
#endif
#else
    # 如果不满足 SPH_64 的条件，则执行以下代码
    return (sph_u32)(((const unsigned char *)src)[0])
        | ((sph_u32)(((const unsigned char *)src)[1]) << 8)
        | ((sph_u32)(((const unsigned char *)src)[2]) << 16)
        | ((sph_u32)(((const unsigned char *)src)[3]) << 24);
    # 返回 src 指向的地址处的 4 个字节按照大端顺序组成的 32 位整数值
#endif
}

#if SPH_64

/**
 * Encode a 64-bit value into the provided buffer (big endian convention).
 *
 * @param dst   the destination buffer
 * @param val   the 64-bit value to encode
 */
static SPH_INLINE void
sph_enc64be(void *dst, sph_u64 val)
{
    # 如果 SPH_64 定义了，则执行以下代码
#if defined SPH_UPTR
    # 如果定义了 SPH_UPTR，则执行以下代码
#if SPH_UNALIGNED
    # 如果不要求内存对齐，则执行以下代码
#if SPH_LITTLE_ENDIAN
    # 如果是小端序，则执行以下代码
    val = sph_bswap64(val);
    # 对 val 进行字节交换
#endif
    *(sph_u64 *)dst = val;
    # 将 val 的值存储到 dst 指向的地址处
#else
    # 如果要求内存对齐，则执行以下代码
    if (((SPH_UPTR)dst & 7) == 0) {
        # 如果 dst 地址是 8 字节对齐的，则执行以下代码
#if SPH_LITTLE_ENDIAN
        # 如果是小端序，则执行以下代码
        val = sph_bswap64(val);
        # 对 val 进行字节交换
#endif
        *(sph_u64 *)dst = val;
        # 将 val 的值存储到 dst 指向的地址处
    } else {
        # 如果 dst 地址不是 8 字节对齐的，则执行以下代码
        ((unsigned char *)dst)[0] = (val >> 56);
        ((unsigned char *)dst)[1] = (val >> 48);
        ((unsigned char *)dst)[2] = (val >> 40);
        ((unsigned char *)dst)[3] = (val >> 32);
        ((unsigned char *)dst)[4] = (val >> 24);
        ((unsigned char *)dst)[5] = (val >> 16);
        ((unsigned char *)dst)[6] = (val >> 8);
        ((unsigned char *)dst)[7] = val;
        # 将 val 的值按照大端序存储到 dst 指向的地址处
    }
#endif
#else
    # 如果不要求内存对齐，则执行以下代码
    ((unsigned char *)dst)[0] = (val >> 56);
    ((unsigned char *)dst)[1] = (val >> 48);
    ((unsigned char *)dst)[2] = (val >> 40);
    ((unsigned char *)dst)[3] = (val >> 32);
    ((unsigned char *)dst)[4] = (val >> 24);
    ((unsigned char *)dst)[5] = (val >> 16);
    ((unsigned char *)dst)[6] = (val >> 8);
    ((unsigned char *)dst)[7] = val;
    # 将 val 的值按照大端序存储到 dst 指向的地址处
#endif
}

/**
 * Encode a 64-bit value into the provided buffer (big endian convention).
 * The destination buffer must be properly aligned.
 *
 * @param dst   the destination buffer (64-bit aligned)
 * @param val   the value to encode
 */
# 将一个64位的值编码到提供的缓冲区中（大端字节序）
def sph_enc64be_aligned(void *dst, sph_u64 val):
    # 如果是小端字节序，将值进行字节交换后存入缓冲区
    if SPH_LITTLE_ENDIAN:
        *(sph_u64 *)dst = sph_bswap64(val)
    # 如果是大端字节序，直接将值存入缓冲区
    elif SPH_BIG_ENDIAN:
        *(sph_u64 *)dst = val
    # 如果是未知字节序，按照大端字节序将值的每个字节存入缓冲区
    else:
        ((unsigned char *)dst)[0] = (val >> 56)
        ((unsigned char *)dst)[1] = (val >> 48)
        ((unsigned char *)dst)[2] = (val >> 40)
        ((unsigned char *)dst)[3] = (val >> 32)
        ((unsigned char *)dst)[4] = (val >> 24)
        ((unsigned char *)dst)[5] = (val >> 16)
        ((unsigned char *)dst)[6] = (val >> 8)
        ((unsigned char *)dst)[7] = val

# 从提供的缓冲区中解码一个64位的值（大端字节序）
def sph_dec64be(const void *src):
    # 如果定义了 SPH_UPTR
    if defined SPH_UPTR:
        # 如果支持非对齐访问
        if SPH_UNALIGNED:
            # 如果是小端字节序，将缓冲区中的值进行字节交换后返回
            if SPH_LITTLE_ENDIAN:
                return sph_bswap64(*(const sph_u64 *)src)
            # 如果是大端字节序，直接返回缓冲区中的值
            else:
                return *(const sph_u64 *)src
        # 如果不支持非对齐访问
        else:
            # 如果缓冲区地址是8的倍数
            if (((SPH_UPTR)src & 7) == 0):
                # 如果是小端字节序，将缓冲区中的值进行字节交换后返回
                if SPH_LITTLE_ENDIAN:
                    return sph_bswap64(*(const sph_u64 *)src)
                # 如果是大端字节序，直接返回缓冲区中的值
                else:
                    return *(const sph_u64 *)src
            # 如果缓冲区地址不是8的倍数，按照大端字节序将缓冲区中的每个字节组合成一个64位的值返回
            else:
                return ((sph_u64)(((const unsigned char *)src)[0]) << 56)
                    | ((sph_u64)(((const unsigned char *)src)[1]) << 48)
                    | ((sph_u64)(((const unsigned char *)src)[2]) << 40)
                    | ((sph_u64)(((const unsigned char *)src)[3]) << 32)
                    | ((sph_u64)(((const unsigned char *)src)[4]) << 24)
                    | ((sph_u64)(((const unsigned char *)src)[5]) << 16)
                    | ((sph_u64)(((const unsigned char *)src)[6]) << 8)
                    | (sph_u64)(((const unsigned char *)src)[7])
    # 如果未定义 SPH_UPTR
    else:
        # 返回空值
        return
    # 将输入的8个字节的数据按照大端序转换成一个64位整数
    return ((sph_u64)(((const unsigned char *)src)[0]) << 56)  # 将第一个字节左移56位
        | ((sph_u64)(((const unsigned char *)src)[1]) << 48)  # 将第二个字节左移48位
        | ((sph_u64)(((const unsigned char *)src)[2]) << 40)  # 将第三个字节左移40位
        | ((sph_u64)(((const unsigned char *)src)[3]) << 32)  # 将第四个字节左移32位
        | ((sph_u64)(((const unsigned char *)src)[4]) << 24)  # 将第五个字节左移24位
        | ((sph_u64)(((const unsigned char *)src)[5]) << 16)  # 将第六个字节左移16位
        | ((sph_u64)(((const unsigned char *)src)[6]) << 8)   # 将第七个字节左移8位
        | (sph_u64)(((const unsigned char *)src)[7]);         # 加上第八个字节的值
#endif
}

/**
 * 从提供的缓冲区中解码一个64位值（大端字节序）。
 * 源缓冲区必须正确对齐。
 *
 * @param src   源缓冲区（64位对齐）
 * @return  解码后的值
 */
static SPH_INLINE sph_u64
sph_dec64be_aligned(const void *src)
{
#if SPH_LITTLE_ENDIAN
    // 如果是小端字节序，将源缓冲区的值进行字节交换
    return sph_bswap64(*(const sph_u64 *)src);
#elif SPH_BIG_ENDIAN
    // 如果是大端字节序，直接返回源缓冲区的值
    return *(const sph_u64 *)src;
#else
    // 如果是未知字节序，手动解码64位值
    return ((sph_u64)(((const unsigned char *)src)[0]) << 56)
        | ((sph_u64)(((const unsigned char *)src)[1]) << 48)
        | ((sph_u64)(((const unsigned char *)src)[2]) << 40)
        | ((sph_u64)(((const unsigned char *)src)[3]) << 32)
        | ((sph_u64)(((const unsigned char *)src)[4]) << 24)
        | ((sph_u64)(((const unsigned char *)src)[5]) << 16)
        | ((sph_u64)(((const unsigned char *)src)[6]) << 8)
        | (sph_u64)(((const unsigned char *)src)[7]);
#endif
}

/**
 * 将一个64位值编码到提供的缓冲区中（小端字节序）。
 *
 * @param dst   目标缓冲区
 * @param val   要编码的64位值
 */
static SPH_INLINE void
sph_enc64le(void *dst, sph_u64 val)
{
#if defined SPH_UPTR
#if SPH_UNALIGNED
#if SPH_BIG_ENDIAN
    // 如果是大端字节序，将值进行字节交换
    val = sph_bswap64(val);
#endif
    // 将值写入目标缓冲区
    *(sph_u64 *)dst = val;
#else
    // 如果目标缓冲区未对齐
    if (((SPH_UPTR)dst & 7) == 0) {
#if SPH_BIG_ENDIAN
        // 如果是大端字节序，将值进行字节交换
        val = sph_bswap64(val);
#endif
        // 将值写入目标缓冲区
        *(sph_u64 *)dst = val;
    } else {
        // 如果目标缓冲区未对齐，手动编码64位值
        ((unsigned char *)dst)[0] = val;
        ((unsigned char *)dst)[1] = (val >> 8);
        ((unsigned char *)dst)[2] = (val >> 16);
        ((unsigned char *)dst)[3] = (val >> 24);
        ((unsigned char *)dst)[4] = (val >> 32);
        ((unsigned char *)dst)[5] = (val >> 40);
        ((unsigned char *)dst)[6] = (val >> 48);
        ((unsigned char *)dst)[7] = (val >> 56);
    }
#endif
#else
    // 如果没有定义 SPH_UPTR，手动编码64位值
    ((unsigned char *)dst)[0] = val;
    ((unsigned char *)dst)[1] = (val >> 8);
    ((unsigned char *)dst)[2] = (val >> 16);
    ((unsigned char *)dst)[3] = (val >> 24);
    # 将64位整数val的高8位分别存储到dst的第4、5、6、7个字节中
    ((unsigned char *)dst)[4] = (val >> 32);
    ((unsigned char *)dst)[5] = (val >> 40);
    ((unsigned char *)dst)[6] = (val >> 48);
    ((unsigned char *)dst)[7] = (val >> 56);
#endif
}

/**
 * 将一个64位的值编码到提供的缓冲区中（小端字节序约定）。
 * 目标缓冲区必须正确对齐。
 *
 * @param dst   目标缓冲区（64位对齐）
 * @param val   要编码的值
 */
static SPH_INLINE void
sph_enc64le_aligned(void *dst, sph_u64 val)
{
#if SPH_LITTLE_ENDIAN
    // 如果是小端字节序，直接将值写入目标缓冲区
    *(sph_u64 *)dst = val;
#elif SPH_BIG_ENDIAN
    // 如果是大端字节序，先将值进行字节交换，再写入目标缓冲区
    *(sph_u64 *)dst = sph_bswap64(val);
#else
    // 如果是未知字节序，按照小端字节序将值写入目标缓冲区
    ((unsigned char *)dst)[0] = val;
    ((unsigned char *)dst)[1] = (val >> 8);
    ((unsigned char *)dst)[2] = (val >> 16);
    ((unsigned char *)dst)[3] = (val >> 24);
    ((unsigned char *)dst)[4] = (val >> 32);
    ((unsigned char *)dst)[5] = (val >> 40);
    ((unsigned char *)dst)[6] = (val >> 48);
    ((unsigned char *)dst)[7] = (val >> 56);
#endif
}

/**
 * 从提供的缓冲区中解码一个64位的值（小端字节序约定）。
 *
 * @param src   源缓冲区
 * @return  解码后的值
 */
static SPH_INLINE sph_u64
sph_dec64le(const void *src)
{
#if defined SPH_UPTR
#if SPH_UNALIGNED
#if SPH_BIG_ENDIAN
    // 如果是大端字节序，先进行字节交换，再返回解码后的值
    return sph_bswap64(*(const sph_u64 *)src);
#else
    // 如果是小端字节序，直接返回解码后的值
    return *(const sph_u64 *)src;
#endif
#else
    // 如果缓冲区未对齐
    if (((SPH_UPTR)src & 7) == 0) {
#if SPH_BIG_ENDIAN
#if SPH_SPARCV9_GCC_64 && !SPH_NO_ASM
        sph_u64 tmp;

        __asm__ __volatile__ (
            "ldxa [%1]0x88,%0" : "=r" (tmp) : "r" (src));
        return tmp;
/*
 * Not worth it generally.
 *
#elif SPH_PPC32_GCC && !SPH_NO_ASM
        return (sph_u64)sph_dec32le_aligned(src)
            | ((sph_u64)sph_dec32le_aligned(
                (const char *)src + 4) << 32);
#elif SPH_PPC64_GCC && !SPH_NO_ASM
        sph_u64 tmp;

        __asm__ __volatile__ (
            "ldbrx %0,0,%1" : "=r" (tmp) : "r" (src));
        return tmp;
 */
#else
        // 如果是大端字节序，先进行字节交换，再返回解码后的值
        return sph_bswap64(*(const sph_u64 *)src);
#endif
#else
        // 如果是小端字节序，直接返回解码后的值
        return *(const sph_u64 *)src;
#endif
    } else {
        # 如果输入的数据类型是小端序的，将其转换为大端序的64位整数
        return (sph_u64)(((const unsigned char *)src)[0])
            | ((sph_u64)(((const unsigned char *)src)[1]) << 8)
            | ((sph_u64)(((const unsigned char *)src)[2]) << 16)
            | ((sph_u64)(((const unsigned char *)src)[3]) << 24)
            | ((sph_u64)(((const unsigned char *)src)[4]) << 32)
            | ((sph_u64)(((const unsigned char *)src)[5]) << 40)
            | ((sph_u64)(((const unsigned char *)src)[6]) << 48)
            | ((sph_u64)(((const unsigned char *)src)[7]) << 56);
    }


注意：这段代码是一个条件语句的一部分，根据条件判断选择执行的代码块。在这个代码块中，将输入的数据从小端序转换为大端序的64位整数。
#else
    return (sph_u64)(((const unsigned char *)src)[0])
        | ((sph_u64)(((const unsigned char *)src)[1]) << 8)
        | ((sph_u64)(((const unsigned char *)src)[2]) << 16)
        | ((sph_u64)(((const unsigned char *)src)[3]) << 24)
        | ((sph_u64)(((const unsigned char *)src)[4]) << 32)
        | ((sph_u64)(((const unsigned char *)src)[5]) << 40)
        | ((sph_u64)(((const unsigned char *)src)[6]) << 48)
        | ((sph_u64)(((const unsigned char *)src)[7]) << 56);
#endif


#else
    # 将源缓冲区中的字节按照小端序解码为64位整数
    return (sph_u64)(((const unsigned char *)src)[0])
        | ((sph_u64)(((const unsigned char *)src)[1]) << 8)
        | ((sph_u64)(((const unsigned char *)src)[2]) << 16)
        | ((sph_u64)(((const unsigned char *)src)[3]) << 24)
        | ((sph_u64)(((const unsigned char *)src)[4]) << 32)
        | ((sph_u64)(((const unsigned char *)src)[5]) << 40)
        | ((sph_u64)(((const unsigned char *)src)[6]) << 48)
        | ((sph_u64)(((const unsigned char *)src)[7]) << 56);
#endif
```