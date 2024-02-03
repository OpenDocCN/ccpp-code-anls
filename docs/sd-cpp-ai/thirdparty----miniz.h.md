# `stable-diffusion.cpp\thirdparty\miniz.h`

```
/*
   定义 MINIZ_EXPORT，用于导出符号
*/
#pragma once

/* 定义用于完全禁用 miniz.c 的特定部分的宏：
   如果这里的所有宏都被定义，那么剩下的功能将只有 CRC-32、adler-32、tinfl 和 tdefl。
*/

/* 定义 MINIZ_NO_STDIO 以禁用所有使用和依赖 stdio 进行文件 I/O 的功能和函数。 */
/*#define MINIZ_NO_STDIO */

/* 如果指定了 MINIZ_NO_TIME，则 ZIP 存档函数将无法获取当前时间，或 */
/* 获取/设置文件时间，并且不会调用获取/设置时间的 C 运行时函数。 */
/* 当前的缺点是写入到您的存档中的时间将是从 1979 年开始的。 */
/*#define MINIZ_NO_TIME */

/* 定义 MINIZ_NO_ARCHIVE_APIS 以禁用所有 ZIP 存档 API。 */
/*#define MINIZ_NO_ARCHIVE_APIS */

/* 定义 MINIZ_NO_ARCHIVE_WRITING_APIS 以禁用所有与写入相关的 ZIP 存档 API。 */
/*#define MINIZ_NO_ARCHIVE_WRITING_APIS */

/* 定义 MINIZ_NO_ZLIB_APIS 以移除所有 ZLIB 风格的压缩/解压缩 API。 */
/*#define MINIZ_NO_ZLIB_APIS */

/* 定义 MINIZ_NO_ZLIB_COMPATIBLE_NAME 以禁用 zlib 的名称，以防止与标准 zlib 的冲突。 */
/*#define MINIZ_NO_ZLIB_COMPATIBLE_NAMES */

/* 定义 MINIZ_NO_MALLOC 以禁用所有对 malloc、free 和 realloc 的调用。
   注意，如果定义了 MINIZ_NO_MALLOC，则用户必须始终提供自定义的用户分配/释放/重新分配回调函数给 zlib 和存档 API，
   以及一些不提供自定义用户函数的独立辅助 API（例如 tdefl_compress_mem_to_heap() 和 tinfl_decompress_mem_to_heap()）将无法工作。
 */
/*#define MINIZ_NO_MALLOC */

#if defined(__TINYC__) && (defined(__linux) || defined(__linux__))
/* TODO: 在 Linux 上使用 tcc 编译时解决 "error: include file 'sys\utime.h' 的问题 */
#define MINIZ_NO_TIME
#endif

#include <stddef.h>

#if !defined(MINIZ_NO_TIME) && !defined(MINIZ_NO_ARCHIVE_APIS)
#include <time.h>
#endif
#if defined(_M_IX86) || defined(_M_X64) || defined(__i386__) ||                \
    defined(__i386) || defined(__i486__) || defined(__i486) ||                 \
    defined(i386) || defined(__ia64__) || defined(__x86_64__)
/* 如果定义了 x86 或 x64 架构相关的宏，则设置 MINIZ_X86_OR_X64_CPU 为 1 */
#define MINIZ_X86_OR_X64_CPU 1
#else
/* 否则设置 MINIZ_X86_OR_X64_CPU 为 0 */
#define MINIZ_X86_OR_X64_CPU 0
#endif

#if (__BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__) || MINIZ_X86_OR_X64_CPU
/* 如果字节序为小端或者是 x86 或 x64 架构，则设置 MINIZ_LITTLE_ENDIAN 为 1 */
#define MINIZ_LITTLE_ENDIAN 1
#else
/* 否则设置 MINIZ_LITTLE_ENDIAN 为 0 */
#define MINIZ_LITTLE_ENDIAN 0
#endif

/* 如果未定义 MINIZ_USE_UNALIGNED_LOADS_AND_STORES 宏 */
#if !defined(MINIZ_USE_UNALIGNED_LOADS_AND_STORES)
#if MINIZ_X86_OR_X64_CPU
/* 如果是 x86 或 x64 架构，则设置 MINIZ_USE_UNALIGNED_LOADS_AND_STORES 为 1 */
#define MINIZ_USE_UNALIGNED_LOADS_AND_STORES 1
#define MINIZ_UNALIGNED_USE_MEMCPY
#else
/* 否则设置 MINIZ_USE_UNALIGNED_LOADS_AND_STORES 为 0 */
#define MINIZ_USE_UNALIGNED_LOADS_AND_STORES 0
#endif
#endif

#if defined(_M_X64) || defined(_WIN64) || defined(__MINGW64__) ||              \
    defined(_LP64) || defined(__LP64__) || defined(__ia64__) ||                \
    defined(__x86_64__)
/* 如果定义了 64 位相关的宏，则设置 MINIZ_HAS_64BIT_REGISTERS 为 1 */
#define MINIZ_HAS_64BIT_REGISTERS 1
#else
/* 否则设置 MINIZ_HAS_64BIT_REGISTERS 为 0 */
#define MINIZ_HAS_64BIT_REGISTERS 0
#endif

#ifdef __cplusplus
extern "C" {
#endif

/* ------------------- zlib-style API Definitions. */

/* 为了与 zlib 更兼容，miniz.c 在某些参数/结构成员中使用 unsigned long 类型。注意：mz_ulong 可能是 32 位或 64 位！ */
typedef unsigned long mz_ulong;

/* mz_free() 在内部使用 MZ_FREE() 宏（默认调用 free()，除非修改了 MZ_MALLOC 宏）释放从堆中分配的块。 */
MINIZ_EXPORT void mz_free(void *p);

#define MZ_ADLER32_INIT (1)
/* mz_adler32()函数在ptr==NULL时返回要使用的初始adler-32值。 */
MINIZ_EXPORT mz_ulong mz_adler32(mz_ulong adler, const unsigned char *ptr,
                                 size_t buf_len);

#define MZ_CRC32_INIT (0)
/* mz_crc32()函数在ptr==NULL时返回要使用的初始CRC-32值。 */
MINIZ_EXPORT mz_ulong mz_crc32(mz_ulong crc, const unsigned char *ptr,
                               size_t buf_len);

/* 压缩策略。 */
enum {
  MZ_DEFAULT_STRATEGY = 0,
  MZ_FILTERED = 1,
  MZ_HUFFMAN_ONLY = 2,
  MZ_RLE = 3,
  MZ_FIXED = 4
};

/* 方法 */
#define MZ_DEFLATED 8

/* 堆分配回调函数。
注意，mz_alloc_func参数类型故意与zlib的不同：items/size是size_t，而不是unsigned long。 */
typedef void *(*mz_alloc_func)(void *opaque, size_t items, size_t size);
typedef void (*mz_free_func)(void *opaque, void *address);
typedef void *(*mz_realloc_func)(void *opaque, void *address, size_t items,
                                 size_t size);

/* 压缩级别：0-9是标准的zlib风格级别，10是最佳压缩（不兼容zlib，可能非常慢），
MZ_DEFAULT_COMPRESSION=MZ_DEFAULT_LEVEL。 */
enum {
  MZ_NO_COMPRESSION = 0,
  MZ_BEST_SPEED = 1,
  MZ_BEST_COMPRESSION = 9,
  MZ_UBER_COMPRESSION = 10,
  MZ_DEFAULT_LEVEL = 6,
  MZ_DEFAULT_COMPRESSION = -1
};

#define MZ_VERSION "10.2.0"
#define MZ_VERNUM 0xA100
#define MZ_VER_MAJOR 10
#define MZ_VER_MINOR 2
#define MZ_VER_REVISION 0
#define MZ_VER_SUBREVISION 0

#ifndef MINIZ_NO_ZLIB_APIS

/* 刷新值。对于典型用法，您只需要MZ_NO_FLUSH和MZ_FINISH。其他值用于高级用法（参考zlib文档）。 */
enum {
  MZ_NO_FLUSH = 0,
  MZ_PARTIAL_FLUSH = 1,
  MZ_SYNC_FLUSH = 2,
  MZ_FULL_FLUSH = 3,
  MZ_FINISH = 4,
  MZ_BLOCK = 5
};

/* 返回状态代码。MZ_PARAM_ERROR是非标准的。 */
// 定义枚举类型，表示不同的返回状态
enum {
  MZ_OK = 0, // 操作成功
  MZ_STREAM_END = 1, // 流结束
  MZ_NEED_DICT = 2, // 需要字典
  MZ_ERRNO = -1, // 错误号
  MZ_STREAM_ERROR = -2, // 流错误
  MZ_DATA_ERROR = -3, // 数据错误
  MZ_MEM_ERROR = -4, // 内存错误
  MZ_BUF_ERROR = -5, // 缓冲区错误
  MZ_VERSION_ERROR = -6, // 版本错误
  MZ_PARAM_ERROR = -10000 // 参数错误
};

/* Window bits */
#define MZ_DEFAULT_WINDOW_BITS 15 // 定义默认窗口大小为15

struct mz_internal_state;

/* 压缩/解压缩流结构体 */
typedef struct mz_stream_s {
  const unsigned char *next_in; // 指向下一个要读取的字节
  unsigned int avail_in; // 下一个要读取的字节数
  mz_ulong total_in; // 到目前为止已消耗的字节数

  unsigned char *next_out; // 指向下一个要写入的字节
  unsigned int avail_out; // 可以写入的字节数
  mz_ulong total_out; // 到目前为止已产生的字节数

  char *msg; // 错误消息（未使用）
  struct mz_internal_state
      *state; // 内部状态，由zalloc/zfree分配

  mz_alloc_func
      zalloc; // 可选的堆分配函数（默认为malloc）
  mz_free_func zfree; // 可选的堆释放函数（默认为free）
  void *opaque; // 堆分配函数用户指针

  int data_type; // 数据类型（未使用）
  mz_ulong adler; // 源数据或未压缩数据的adler32
  mz_ulong reserved; // 未使用
} mz_stream;

typedef mz_stream *mz_streamp;

/* 返回miniz.c的版本字符串 */
MINIZ_EXPORT const char *mz_version(void);

/* mz_deflateInit()使用默认选项初始化压缩器： */
/* 参数： */
/*  pStream必须指向已初始化的mz_stream结构体。 */
/*  level必须在[MZ_NO_COMPRESSION, MZ_BEST_COMPRESSION]之间。 */
/*  level为1时启用一个经过特别优化的压缩函数，该函数纯粹为了性能而优化，而不是比率。 */
/*  （此特殊函数目前仅在定义了MINIZ_USE_UNALIGNED_LOADS_AND_STORES和MINIZ_LITTLE_ENDIAN时启用。） */
/* 返回值： */
/*  成功时返回MZ_OK。 */
/*  如果流是虚假的，则返回 MZ_STREAM_ERROR。 */
/*  如果输入参数是虚假的，则返回 MZ_PARAM_ERROR。 */
/*  如果内存不足，则返回 MZ_MEM_ERROR。 */
MINIZ_EXPORT int mz_deflateInit(mz_streamp pStream, int level);

/* mz_deflateInit2() 类似于 mz_deflate()，但具有更多控制： */
/* 附加参数： */
/*   method 必须是 MZ_DEFLATED */
/*   window_bits 必须是 MZ_DEFAULT_WINDOW_BITS（用 zlib 头/adler-32 尾部包装 deflate 流）或 -MZ_DEFAULT_WINDOW_BITS（原始 deflate/无头部或尾部） */
/*   mem_level 必须在 [1, 9] 之间（miniz.c 中会检查但会被忽略） */
MINIZ_EXPORT int mz_deflateInit2(mz_streamp pStream, int level, int method,
                                 int window_bits, int mem_level, int strategy);

/* 快速重置压缩器，无需重新分配任何内容。等同于调用 mz_deflateEnd()，然后调用 mz_deflateInit()/mz_deflateInit2()。 */
MINIZ_EXPORT int mz_deflateReset(mz_streamp pStream);

/* mz_deflate() 将输入压缩到输出，尽可能消耗输入并产生尽可能多的输出。 */
/* 参数： */
/*   pStream 是要从中读取和写入的流。必须初始化/更新 next_in、avail_in、next_out 和 avail_out 成员。 */
/*   flush 可以是 MZ_NO_FLUSH、MZ_PARTIAL_FLUSH/MZ_SYNC_FLUSH、MZ_FULL_FLUSH 或 MZ_FINISH。 */
/* 返回值： */
/*   成功时返回 MZ_OK（在刷新时，或者需要更多输入但不可用时，和/或有更多输出需要写入但输出缓冲区已满时）。 */
/*   如果所有输入都已消耗完并且所有输出字节都已写入，则返回 MZ_STREAM_END。不要再对流调用 mz_deflate()。 */
/*   如果流是虚假的，则返回 MZ_STREAM_ERROR。 */
/*   如果参数无效，则返回 MZ_PARAM_ERROR。 */
/*   如果由于输入和/或输出缓冲区为空而无法取得任何进展，则返回 MZ_BUF_ERROR。（填充输入缓冲区或释放一些输出空间并重试。） */
/* 压缩数据的函数，使用 deflate 算法 */
MINIZ_EXPORT int mz_deflate(mz_streamp pStream, int flush);

/* 结束压缩器的函数，释放资源 */
/* 返回值： */
/*  MZ_OK 表示成功 */
/*  MZ_STREAM_ERROR 表示流无效 */
MINIZ_EXPORT int mz_deflateEnd(mz_streamp pStream);

/* 返回 deflate() 函数生成数据的保守上限，假设 flush 只设置为 MZ_NO_FLUSH 或 MZ_FINISH */
MINIZ_EXPORT mz_ulong mz_deflateBound(mz_streamp pStream, mz_ulong source_len);

/* 单次调用压缩函数 mz_compress() 和 mz_compress2()： */
/* 成功返回 MZ_OK，失败返回 mz_deflate() 的错误码之一 */
MINIZ_EXPORT int mz_compress(unsigned char *pDest, mz_ulong *pDest_len,
                             const unsigned char *pSource, mz_ulong source_len);
MINIZ_EXPORT int mz_compress2(unsigned char *pDest, mz_ulong *pDest_len,
                              const unsigned char *pSource, mz_ulong source_len,
                              int level);

/* 返回调用 mz_compress() 生成数据的保守上限 */
MINIZ_EXPORT mz_ulong mz_compressBound(mz_ulong source_len);

/* 初始化解压缩器 */
MINIZ_EXPORT int mz_inflateInit(mz_streamp pStream);

/* mz_inflateInit2() 类似于 mz_inflateInit()，但有一个额外选项控制窗口大小以及流是否包含 zlib 头/尾 */
/* window_bits 必须是 MZ_DEFAULT_WINDOW_BITS（解析 zlib 头/尾）或 -MZ_DEFAULT_WINDOW_BITS（原始 deflate） */
MINIZ_EXPORT int mz_inflateInit2(mz_streamp pStream, int window_bits);

/* 快速重置压缩器，无需重新分配任何资源。等同于调用 mz_inflateEnd() 后再调用 mz_inflateInit()/mz_inflateInit2() */
MINIZ_EXPORT int mz_inflateReset(mz_streamp pStream);
/* Decompresses the input stream to the output, consuming only as much of the
 * input as needed, and writing as much to the output as possible. */
/* Parameters: */
/*   pStream is the stream to read from and write to. You must initialize/update
 * the next_in, avail_in, next_out, and avail_out members. */
/*   flush may be MZ_NO_FLUSH, MZ_SYNC_FLUSH, or MZ_FINISH. */
/*   On the first call, if flush is MZ_FINISH it's assumed the input and output
 * buffers are both sized large enough to decompress the entire stream in a
 * single call (this is slightly faster). */
/*   MZ_FINISH implies that there are no more source bytes available beside
 * what's already in the input buffer, and that the output buffer is large
 * enough to hold the rest of the decompressed data. */
/* Return values: */
/*   MZ_OK on success. Either more input is needed but not available, and/or
 * there's more output to be written but the output buffer is full. */
/*   MZ_STREAM_END if all needed input has been consumed and all output bytes
 * have been written. For zlib streams, the adler-32 of the decompressed data
 * has also been verified. */
/*   MZ_STREAM_ERROR if the stream is bogus. */
/*   MZ_DATA_ERROR if the deflate stream is invalid. */
/*   MZ_PARAM_ERROR if one of the parameters is invalid. */
/*   MZ_BUF_ERROR if no forward progress is possible because the input buffer is
 * empty but the inflater needs more input to continue, or if the output buffer
 * is not large enough. Call mz_inflate() again */
/*   with more input data, or with more room in the output buffer (except when
 * using single call decompression, described above). */
MINIZ_EXPORT int mz_inflate(mz_streamp pStream, int flush);

/* Deinitializes a decompressor. */
MINIZ_EXPORT int mz_inflateEnd(mz_streamp pStream);

/* Single-call decompression. */
/* Returns MZ_OK on success, or one of the error codes from mz_inflate() on
 * failure. */
# 定义了两个函数用于解压缩数据，分别为 mz_uncompress 和 mz_uncompress2
MINIZ_EXPORT int mz_uncompress(unsigned char *pDest, mz_ulong *pDest_len,
                               const unsigned char *pSource,
                               mz_ulong source_len);
MINIZ_EXPORT int mz_uncompress2(unsigned char *pDest, mz_ulong *pDest_len,
                                const unsigned char *pSource,
                                mz_ulong *pSource_len);

# 返回指定错误代码的字符串描述，如果错误代码无效则返回 NULL
MINIZ_EXPORT const char *mz_error(int err);

# 重新定义 zlib 兼容的名称为 miniz 的等效名称，以便 miniz.c 可以作为 miniz.c 支持的 zlib 子集的替代品
# 如果在同一项目中使用 zlib，则定义 MINIZ_NO_ZLIB_COMPATIBLE_NAMES 禁用 zlib 兼容性
#ifndef MINIZ_NO_ZLIB_COMPATIBLE_NAMES
# 定义一系列 zlib 兼容的名称为 miniz 的等效名称
typedef unsigned char Byte;
typedef unsigned int uInt;
typedef mz_ulong uLong;
typedef Byte Bytef;
typedef uInt uIntf;
typedef char charf;
typedef int intf;
typedef void *voidpf;
typedef uLong uLongf;
typedef void *voidp;
typedef void *const voidpc;
# 定义一系列 zlib 兼容的常量为 miniz 的等效常量
#define Z_NULL 0
#define Z_NO_FLUSH MZ_NO_FLUSH
#define Z_PARTIAL_FLUSH MZ_PARTIAL_FLUSH
#define Z_SYNC_FLUSH MZ_SYNC_FLUSH
#define Z_FULL_FLUSH MZ_FULL_FLUSH
#define Z_FINISH MZ_FINISH
#define Z_BLOCK MZ_BLOCK
#define Z_OK MZ_OK
#define Z_STREAM_END MZ_STREAM_END
#define Z_NEED_DICT MZ_NEED_DICT
#define Z_ERRNO MZ_ERRNO
#define Z_STREAM_ERROR MZ_STREAM_ERROR
#define Z_DATA_ERROR MZ_DATA_ERROR
#define Z_MEM_ERROR MZ_MEM_ERROR
#define Z_BUF_ERROR MZ_BUF_ERROR
#define Z_VERSION_ERROR MZ_VERSION_ERROR
#define Z_PARAM_ERROR MZ_PARAM_ERROR
#define Z_NO_COMPRESSION MZ_NO_COMPRESSION
#define Z_BEST_SPEED MZ_BEST_SPEED
#define Z_BEST_COMPRESSION MZ_BEST_COMPRESSION
#define Z_DEFAULT_COMPRESSION MZ_DEFAULT_COMPRESSION
#define Z_DEFAULT_STRATEGY MZ_DEFAULT_STRATEGY
#define Z_FILTERED MZ_FILTERED
#define Z_HUFFMAN_ONLY MZ_HUFFMAN_ONLY
#define Z_RLE MZ_RLE
#define Z_FIXED MZ_FIXED
// 定义 Z_DEFLATED 为 MZ_DEFLATED
// 定义 Z_DEFAULT_WINDOW_BITS 为 MZ_DEFAULT_WINDOW_BITS
// 定义 alloc_func 为 mz_alloc_func
// 定义 free_func 为 mz_free_func
// 定义 internal_state 为 mz_internal_state
// 定义 z_stream 为 mz_stream
// 定义 deflateInit 为 mz_deflateInit
// 定义 deflateInit2 为 mz_deflateInit2
// 定义 deflateReset 为 mz_deflateReset
// 定义 deflate 为 mz_deflate
// 定义 deflateEnd 为 mz_deflateEnd
// 定义 deflateBound 为 mz_deflateBound
// 定义 compress 为 mz_compress
// 定义 compress2 为 mz_compress2
// 定义 compressBound 为 mz_compressBound
// 定义 inflateInit 为 mz_inflateInit
// 定义 inflateInit2 为 mz_inflateInit2
// 定义 inflateReset 为 mz_inflateReset
// 定义 inflate 为 mz_inflate
// 定义 inflateEnd 为 mz_inflateEnd
// 定义 uncompress 为 mz_uncompress
// 定义 uncompress2 为 mz_uncompress2
// 定义 crc32 为 mz_crc32
// 定义 adler32 为 mz_adler32
// 定义 MAX_WBITS 为 15
// 定义 MAX_MEM_LEVEL 为 9
// 定义 zError 为 mz_error
// 定义 ZLIB_VERSION 为 MZ_VERSION
// 定义 ZLIB_VERNUM 为 MZ_VERNUM
// 定义 ZLIB_VER_MAJOR 为 MZ_VER_MAJOR
// 定义 ZLIB_VER_MINOR 为 MZ_VER_MINOR
// 定义 ZLIB_VER_REVISION 为 MZ_VER_REVISION
// 定义 ZLIB_VER_SUBREVISION 为 MZ_VER_SUBREVISION
// 定义 zlibVersion 为 mz_version
// 调用 mz_version() 函数并定义为 zlib_version
#endif /* #ifndef MINIZ_NO_ZLIB_COMPATIBLE_NAMES */

#endif /* MINIZ_NO_ZLIB_APIS */

#ifdef __cplusplus
}
#endif

#pragma once
#include <assert.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>

// 定义 mz_uint8 为 unsigned char
// 定义 mz_int16 为 signed short
// 定义 mz_uint16 为 unsigned short
// 定义 mz_uint32 为 unsigned int
// 定义 mz_uint 为 unsigned int
// 定义 mz_int64 为 int64_t
// 定义 mz_uint64 为 uint64_t
// 定义 mz_bool 为 int
// 定义 MZ_FALSE 为 0
// 定义 MZ_TRUE 为 1

// 解决 MSVC 的警告 "warning C4127: conditional expression is constant" 的宏定义
#ifdef _MSC_VER
#define MZ_MACRO_END while (0, 0)
#else
#define MZ_MACRO_END while (0)
#endif

#ifdef MINIZ_NO_STDIO
// 如果定义了 MINIZ_NO_STDIO，则将 MZ_FILE 定义为 void *
#define MZ_FILE void *
#else
#include <stdio.h>
// 否则包含 stdio.h 并将 MZ_FILE 定义为 FILE
#define MZ_FILE FILE
#ifdef MINIZ_NO_STDIO
// 如果定义了 MINIZ_NO_STDIO，则定义一个结构体 mz_dummy_time_t 代表时间
typedef struct mz_dummy_time_t_tag {
  int m_dummy;
} mz_dummy_time_t;
// 定义 MZ_TIME_T 为 mz_dummy_time_t 类型
#define MZ_TIME_T mz_dummy_time_t
#else
// 否则定义 MZ_TIME_T 为 time_t 类型
#define MZ_TIME_T time_t
#endif

// 定义宏 MZ_ASSERT 用于断言
#define MZ_ASSERT(x) assert(x)

#ifdef MINIZ_NO_MALLOC
// 如果定义了 MINIZ_NO_MALLOC，则定义 MZ_MALLOC、MZ_FREE、MZ_REALLOC 宏为 NULL
#define MZ_MALLOC(x) NULL
#define MZ_FREE(x) (void)x, ((void)0)
#define MZ_REALLOC(p, x) NULL
#else
// 否则定义 MZ_MALLOC、MZ_FREE、MZ_REALLOC 宏为 malloc、free、realloc
#define MZ_MALLOC(x) malloc(x)
#define MZ_FREE(x) free(x)
#define MZ_REALLOC(p, x) realloc(p, x)
#endif

// 定义宏 MZ_MAX 用于返回两个数中的最大值
#define MZ_MAX(a, b) (((a) > (b)) ? (a) : (b))
// 定义宏 MZ_MIN 用于返回两个数中的最小值
#define MZ_MIN(a, b) (((a) < (b)) ? (a) : (b))
// 定义宏 MZ_CLEAR_OBJ 用于将对象清零
#define MZ_CLEAR_OBJ(obj) memset(&(obj), 0, sizeof(obj))

#if MINIZ_USE_UNALIGNED_LOADS_AND_STORES && MINIZ_LITTLE_ENDIAN
// 如果定义了 MINIZ_USE_UNALIGNED_LOADS_AND_STORES 并且是小端序，则定义 MZ_READ_LE16 和 MZ_READ_LE32 宏
#define MZ_READ_LE16(p) *((const mz_uint16 *)(p))
#define MZ_READ_LE32(p) *((const mz_uint32 *)(p))
#else
// 否则定义 MZ_READ_LE16 和 MZ_READ_LE32 宏
#define MZ_READ_LE16(p)                                                        \
  ((mz_uint32)(((const mz_uint8 *)(p))[0]) |                                   \
   ((mz_uint32)(((const mz_uint8 *)(p))[1]) << 8U))
#define MZ_READ_LE32(p)                                                        \
  ((mz_uint32)(((const mz_uint8 *)(p))[0]) |                                   \
   ((mz_uint32)(((const mz_uint8 *)(p))[1]) << 8U) |                           \
   ((mz_uint32)(((const mz_uint8 *)(p))[2]) << 16U) |                          \
   ((mz_uint32)(((const mz_uint8 *)(p))[3]) << 24U))
#endif

// 定义宏 MZ_READ_LE64 用于读取 64 位小端序数据
#define MZ_READ_LE64(p)                                                        \
  (((mz_uint64)MZ_READ_LE32(p)) |                                              \
   (((mz_uint64)MZ_READ_LE32((const mz_uint8 *)(p) + sizeof(mz_uint32)))       \
    << 32U))

#ifdef _MSC_VER
// 如果是 MSC 编译器，则定义 MZ_FORCEINLINE 为 __forceinline
#define MZ_FORCEINLINE __forceinline
#elif defined(__GNUC__)
// 如果是 GCC 编译器，则定义 MZ_FORCEINLINE 为 __inline__ __attribute__((__always_inline__))
#define MZ_FORCEINLINE __inline__ __attribute__((__always_inline__))
#else
// 否则定义 MZ_FORCEINLINE 为 inline
#define MZ_FORCEINLINE inline
#endif

#ifdef __cplusplus
// 如果是 C++ 环境，则使用 extern "C" 语法
extern "C" {
#endif
/* 定义 miniz_def_alloc_func 函数，用于分配内存 */
extern MINIZ_EXPORT void *miniz_def_alloc_func(void *opaque, size_t items,
                                               size_t size);
/* 定义 miniz_def_free_func 函数，用于释放内存 */
extern MINIZ_EXPORT void miniz_def_free_func(void *opaque, void *address);
/* 定义 miniz_def_realloc_func 函数，用于重新分配内存 */
extern MINIZ_EXPORT void *miniz_def_realloc_func(void *opaque, void *address,
                                                 size_t items, size_t size);

/* 定义 MZ_UINT16_MAX 常量为 0xFFFFU */
#define MZ_UINT16_MAX (0xFFFFU)
/* 定义 MZ_UINT32_MAX 常量为 0xFFFFFFFFU */
#define MZ_UINT32_MAX (0xFFFFFFFFU)

#ifdef __cplusplus
}
#endif
/* 声明代码段只会被包含一次 */
#pragma once

#ifdef __cplusplus
extern "C" {
#endif
/* ------------------- Low-level Compression API Definitions */

/* 设置 TDEFL_LESS_MEMORY 为 1 以减少内存使用（压缩速度会稍慢，原始/动态块会更频繁输出） */
#define TDEFL_LESS_MEMORY 0

/* tdefl_init() 的压缩标志逻辑上 OR 在一起（低 12 位包含每个字典搜索的最大探测次数）： */
/* TDEFL_DEFAULT_MAX_PROBES: 压缩器默认每个字典搜索使用 128 次探测。0=仅 Huffman，1=Huffman+LZ（最快/最差压缩），4095=Huffman+LZ（最慢/最佳压缩）。 */
enum {
  TDEFL_HUFFMAN_ONLY = 0,
  TDEFL_DEFAULT_MAX_PROBES = 128,
  TDEFL_MAX_PROBES_MASK = 0xFFF
};

/* TDEFL_WRITE_ZLIB_HEADER: 如果设置，压缩器在 deflate 数据之前输出 zlib 头部，并在最后输出源数据的 Adler-32。否则，将得到原始 deflate 数据。 */
/* TDEFL_COMPUTE_ADLER32: 总是计算输入数据的 adler-32（即使不写入 zlib 头部）。 */
/* TDEFL_GREEDY_PARSING_FLAG: 设置以使用更快的贪婪解析，而不是更有效率的懒惰解析。 */
/* TDEFL_NONDETERMINISTIC_PARSING_FLAG: 启用以将压缩器的初始化时间减少到最小，但输出可能因运行时内存内容不同而有所变化（取决于内存内容）。 */
/* TDEFL_RLE_MATCHES: 仅查找 RLE 匹配（距离为 1 的匹配） */
/* 定义一些压缩选项的枚举值，用于控制压缩行为 */
/* TDEFL_FILTER_MATCHES: 如果启用，则丢弃长度小于等于5个字符的匹配项 */
/* TDEFL_FORCE_ALL_STATIC_BLOCKS: 禁用使用优化的哈夫曼表 */
/* TDEFL_FORCE_ALL_RAW_BLOCKS: 仅使用原始（未压缩）的deflate块 */
/* 低12位用于控制每个字典查找的最大哈希探测次数（参见TDEFL_MAX_PROBES_MASK） */
enum {
  TDEFL_WRITE_ZLIB_HEADER = 0x01000,
  TDEFL_COMPUTE_ADLER32 = 0x02000,
  TDEFL_GREEDY_PARSING_FLAG = 0x04000,
  TDEFL_NONDETERMINISTIC_PARSING_FLAG = 0x08000,
  TDEFL_RLE_MATCHES = 0x10000,
  TDEFL_FILTER_MATCHES = 0x20000,
  TDEFL_FORCE_ALL_STATIC_BLOCKS = 0x40000,
  TDEFL_FORCE_ALL_RAW_BLOCKS = 0x80000
};

/* 高级压缩函数： */
/* tdefl_compress_mem_to_heap() 将内存中的一个块压缩到通过malloc()分配的堆块中 */
/* 进入时： */
/*  pSrc_buf, src_buf_len: 指向要压缩的源块的指针和大小 */
/*  flags: 最大匹配查找探测次数（默认为128），逻辑上与上述标志进行OR运算。探测次数越多，压缩速度越慢但压缩效果更好 */
/* 返回时： */
/*  函数返回指向压缩数据的指针，失败时返回NULL */
/*  *pOut_len将设置为压缩数据的大小，对于无法压缩的数据可能大于src_buf_len */
/*  当不再需要返回的块时，调用者必须释放返回的块 */
MINIZ_EXPORT void *tdefl_compress_mem_to_heap(const void *pSrc_buf,
                                              size_t src_buf_len,
                                              size_t *pOut_len, int flags);

/* tdefl_compress_mem_to_mem() 将内存中的一个块压缩到另一个内存块中 */
/* 失败时返回0 */
MINIZ_EXPORT size_t tdefl_compress_mem_to_mem(void *pOut_buf,
                                              size_t out_buf_len,
                                              const void *pSrc_buf,
                                              size_t src_buf_len, int flags);
/* 压缩图像到内存中的压缩 PNG 文件 */
/* 进入时： */
/*  pImage、w、h 和 num_chans 描述要压缩的图像。num_chans 可以是 1、2、3 或 4。 */
/*  每行扫描线的图像 pitch 以字节为单位将是 w*num_chans。左上角的像素在内存中首先存储。 */
/*  level 可以取值 [0,10]，使用 MZ_NO_COMPRESSION、MZ_BEST_SPEED、MZ_BEST_COMPRESSION 等，或者使用一个合适的默认值 MZ_DEFAULT_LEVEL */
/*  如果 flip 为 true，则图像将在 Y 轴上翻转（对于 OpenGL 应用程序很有用）。 */
/* 返回时： */
/*  函数返回指向压缩数据的指针，失败时返回 NULL。 */
/*  *pLen_out 将设置为 PNG 图像文件的大小。 */
/*  调用者在不再需要时必须使用 mz_free() 释放返回的堆块（通常比 *pLen_out 大）。 */
MINIZ_EXPORT void *
tdefl_write_image_to_png_file_in_memory_ex(const void *pImage, int w, int h,
                                           int num_chans, size_t *pLen_out,
                                           mz_uint level, mz_bool flip);
MINIZ_EXPORT void *tdefl_write_image_to_png_file_in_memory(const void *pImage,
                                                           int w, int h,
                                                           int num_chans,
                                                           size_t *pLen_out);

/* 输出流接口。压缩器使用此接口来写入压缩数据。通常每次调用 TDEFL_OUT_BUF_SIZE。 */
typedef mz_bool (*tdefl_put_buf_func_ptr)(const void *pBuf, int len,
                                          void *pUser);

/* tdefl_compress_mem_to_output() 将一个块压缩到输出流中。上述辅助函数在内部使用此函数。 */
MINIZ_EXPORT mz_bool tdefl_compress_mem_to_output(
    const void *pBuf, size_t buf_len, tdefl_put_buf_func_ptr pPut_buf_func,
    void *pPut_buf_user, int flags);
// 定义枚举常量，表示压缩算法中的一些参数
enum {
  TDEFL_MAX_HUFF_TABLES = 3, // 最大哈夫曼表数
  TDEFL_MAX_HUFF_SYMBOLS_0 = 288, // 第一个哈夫曼表的最大符号数
  TDEFL_MAX_HUFF_SYMBOLS_1 = 32, // 第二个哈夫曼表的最大符号数
  TDEFL_MAX_HUFF_SYMBOLS_2 = 19, // 第三个哈夫曼表的最大符号数
  TDEFL_LZ_DICT_SIZE = 32768, // LZ 字典大小
  TDEFL_LZ_DICT_SIZE_MASK = TDEFL_LZ_DICT_SIZE - 1, // LZ 字典大小掩码
  TDEFL_MIN_MATCH_LEN = 3, // 最小匹配长度
  TDEFL_MAX_MATCH_LEN = 258 // 最大匹配长度
};

/* TDEFL_OUT_BUF_SIZE MUST be large enough to hold a single entire compressed
 * output block (using static/fixed Huffman codes). */
#if TDEFL_LESS_MEMORY
// 如果使用更少内存的选项
enum {
  TDEFL_LZ_CODE_BUF_SIZE = 24 * 1024, // LZ 编码缓冲区大小
  TDEFL_OUT_BUF_SIZE = (TDEFL_LZ_CODE_BUF_SIZE * 13) / 10, // 输出缓冲区大小
  TDEFL_MAX_HUFF_SYMBOLS = 288, // 最大哈夫曼符号数
  TDEFL_LZ_HASH_BITS = 12, // LZ 哈希位数
  TDEFL_LEVEL1_HASH_SIZE_MASK = 4095, // 一级哈希大小掩码
  TDEFL_LZ_HASH_SHIFT = (TDEFL_LZ_HASH_BITS + 2) / 3, // LZ 哈希位移
  TDEFL_LZ_HASH_SIZE = 1 << TDEFL_LZ_HASH_BITS // LZ 哈希大小
};
#else
// 如果不使用更少内存的选项
enum {
  TDEFL_LZ_CODE_BUF_SIZE = 64 * 1024, // LZ 编码缓冲区大小
  TDEFL_OUT_BUF_SIZE = (TDEFL_LZ_CODE_BUF_SIZE * 13) / 10, // 输出缓冲区大小
  TDEFL_MAX_HUFF_SYMBOLS = 288, // 最大哈夫曼符号数
  TDEFL_LZ_HASH_BITS = 15, // LZ 哈希位数
  TDEFL_LEVEL1_HASH_SIZE_MASK = 4095, // 一级哈希大小掩码
  TDEFL_LZ_HASH_SHIFT = (TDEFL_LZ_HASH_BITS + 2) / 3, // LZ 哈希位移
  TDEFL_LZ_HASH_SIZE = 1 << TDEFL_LZ_HASH_BITS // LZ 哈希大小
};
#endif

/* The low-level tdefl functions below may be used directly if the above helper
 * functions aren't flexible enough. The low-level functions don't make any heap
 * allocations, unlike the above helper functions. */
// 下面的低级 tdefl 函数可以直接使用，如果上面的辅助函数不够灵活。低级函数不进行任何堆分配，不同于上面的辅助函数。
typedef enum {
  TDEFL_STATUS_BAD_PARAM = -2, // 参数错误状态
  TDEFL_STATUS_PUT_BUF_FAILED = -1, // 写入缓冲区失败状态
  TDEFL_STATUS_OKAY = 0, // 正常状态
  TDEFL_STATUS_DONE = 1 // 完成状态
} tdefl_status;

/* Must map to MZ_NO_FLUSH, MZ_SYNC_FLUSH, etc. enums */
// 必须映射到 MZ_NO_FLUSH、MZ_SYNC_FLUSH 等枚举值
typedef enum {
  TDEFL_NO_FLUSH = 0, // 无刷新
  TDEFL_SYNC_FLUSH = 2, // 同步刷新
  TDEFL_FULL_FLUSH = 3, // 完全刷新
  TDEFL_FINISH = 4 // 完成
} tdefl_flush;

/* tdefl's compression state structure. */
// 定义 tdefl_compressor 结构体，包含了压缩算法所需的各种参数和数据结构
typedef struct {
  tdefl_put_buf_func_ptr m_pPut_buf_func; // 指向输出缓冲区的函数指针
  void *m_pPut_buf_user; // 输出缓冲区的用户数据
  mz_uint m_flags, m_max_probes[2]; // 压缩标志和最大探测次数
  int m_greedy_parsing; // 是否使用贪婪解析
  mz_uint m_adler32, m_lookahead_pos, m_lookahead_size, m_dict_size; // Adler-32 校验值、前瞻位置、前瞻大小、字典大小
  mz_uint8 *m_pLZ_code_buf, *m_pLZ_flags, *m_pOutput_buf, *m_pOutput_buf_end; // LZ 编码缓冲区、LZ 标志、输出缓冲区、输出缓冲区结束位置
  mz_uint m_num_flags_left, m_total_lz_bytes, m_lz_code_buf_dict_pos, m_bits_in, m_bit_buffer; // 剩余标志数、LZ 字节总数、LZ 编码缓冲区字典位置、位数、位缓冲区
  mz_uint m_saved_match_dist, m_saved_match_len, m_saved_lit, m_output_flush_ofs, m_output_flush_remaining, m_finished, m_block_index, m_wants_to_finish; // 保存的匹配距离、匹配长度、字面量、输出刷新偏移、输出刷新剩余、是否完成、块索引、是否希望完成
  tdefl_status m_prev_return_status; // 前一个返回状态
  const void *m_pIn_buf; // 输入缓冲区
  void *m_pOut_buf; // 输出缓冲区
  size_t *m_pIn_buf_size, *m_pOut_buf_size; // 输入缓冲区大小、输出缓冲区大小
  tdefl_flush m_flush; // 刷新标志
  const mz_uint8 *m_pSrc; // 源数据
  size_t m_src_buf_left, m_out_buf_ofs; // 源缓冲区剩余、输出缓冲区偏移
  mz_uint8 m_dict[TDEFL_LZ_DICT_SIZE + TDEFL_MAX_MATCH_LEN - 1]; // LZ 字典
  mz_uint16 m_huff_count[TDEFL_MAX_HUFF_TABLES][TDEFL_MAX_HUFF_SYMBOLS]; // 霍夫曼编码计数
  mz_uint16 m_huff_codes[TDEFL_MAX_HUFF_TABLES][TDEFL_MAX_HUFF_SYMBOLS]; // 霍夫曼编码
  mz_uint8 m_huff_code_sizes[TDEFL_MAX_HUFF_TABLES][TDEFL_MAX_HUFF_SYMBOLS]; // 霍夫曼编码大小
  mz_uint8 m_lz_code_buf[TDEFL_LZ_CODE_BUF_SIZE]; // LZ 编码缓冲区
  mz_uint16 m_next[TDEFL_LZ_DICT_SIZE]; // LZ 字典中下一个位置
  mz_uint16 m_hash[TDEFL_LZ_HASH_SIZE]; // LZ 哈希表
  mz_uint8 m_output_buf[TDEFL_OUT_BUF_SIZE]; // 输出缓冲区
} tdefl_compressor;

/* 初始化压缩器。*/
/* 没有对应的 deinit() 函数，因为 tdefl API 不会动态分配内存。*/
/* pBut_buf_func: 如果为 NULL，则输出数据将提供给指定的回调函数。在这种情况下，用户应该调用 tdefl_compress_buffer() API 进行压缩。*/
/* 如果 pBut_buf_func 为 NULL，则用户应该始终调用 tdefl_compress() API。*/
/* flags: 参见上面的枚举值（TDEFL_HUFFMAN_ONLY、TDEFL_WRITE_ZLIB_HEADER 等）。*/
MINIZ_EXPORT tdefl_status tdefl_init(tdefl_compressor *d,
                                     tdefl_put_buf_func_ptr pPut_buf_func,
                                     void *pPut_buf_user, int flags);
/* 压缩数据块，尽可能消耗指定输入缓冲区的数据，并将尽可能多的压缩数据写入指定输出缓冲区 */
MINIZ_EXPORT tdefl_status tdefl_compress(tdefl_compressor *d,
                                         const void *pIn_buf,
                                         size_t *pIn_buf_size, void *pOut_buf,
                                         size_t *pOut_buf_size,
                                         tdefl_flush flush);

/* 当 tdefl_init() 被调用时，tdefl_compress_buffer() 只能在 tdefl_put_buf_func_ptr 非空时使用 */
/* tdefl_compress_buffer() 总是消耗整个输入缓冲区 */
MINIZ_EXPORT tdefl_status tdefl_compress_buffer(tdefl_compressor *d,
                                                const void *pIn_buf,
                                                size_t in_buf_size,
                                                tdefl_flush flush);

MINIZ_EXPORT tdefl_status tdefl_get_prev_return_status(tdefl_compressor *d);
MINIZ_EXPORT mz_uint32 tdefl_get_adler32(tdefl_compressor *d);

/* 根据 zlib 风格的压缩参数创建 tdefl_compress() 标志 */
/* level 可以在 [0,10] 范围内（其中 10 是绝对最大压缩，但在某些文件上可能会更慢） */
/* window_bits 可以是 -15（原始 deflate）或 15（zlib） */
/* strategy 可以是 MZ_DEFAULT_STRATEGY、MZ_FILTERED、MZ_HUFFMAN_ONLY、MZ_RLE 或 MZ_FIXED */
MINIZ_EXPORT mz_uint tdefl_create_comp_flags_from_zip_params(int level,
                                                             int window_bits,
                                                             int strategy);

#ifndef MINIZ_NO_MALLOC
/* 在 C 中分配 tdefl_compressor 结构，以便 */
/* 非 C 语言绑定到 tdefl_ API 不需要担心结构大小和分配机制 */
MINIZ_EXPORT tdefl_compressor *tdefl_compressor_alloc(void);
#pragma once



/* ------------------- Low-level Decompression API Definitions */



#ifdef __cplusplus
extern "C" {
#endif



/* Decompression flags used by tinfl_decompress(). */
/* TINFL_FLAG_PARSE_ZLIB_HEADER: If set, the input has a valid zlib header and
 * ends with an adler32 checksum (it's a valid zlib stream). Otherwise, the
 * input is a raw deflate stream. */
/* TINFL_FLAG_HAS_MORE_INPUT: If set, there are more input bytes available
 * beyond the end of the supplied input buffer. If clear, the input buffer
 * contains all remaining input. */
/* TINFL_FLAG_USING_NON_WRAPPING_OUTPUT_BUF: If set, the output buffer is large
 * enough to hold the entire decompressed stream. If clear, the output buffer is
 * at least the size of the dictionary (typically 32KB). */
/* TINFL_FLAG_COMPUTE_ADLER32: Force adler-32 checksum computation of the
 * decompressed bytes. */



enum {
  TINFL_FLAG_PARSE_ZLIB_HEADER = 1,
  TINFL_FLAG_HAS_MORE_INPUT = 2,
  TINFL_FLAG_USING_NON_WRAPPING_OUTPUT_BUF = 4,
  TINFL_FLAG_COMPUTE_ADLER32 = 8
};



/* High level decompression functions: */
/* tinfl_decompress_mem_to_heap() decompresses a block in memory to a heap block
 * allocated via malloc(). */
/* On entry: */
/*  pSrc_buf, src_buf_len: Pointer and size of the Deflate or zlib source data
 * to decompress. */
/* On return: */
/*  Function returns a pointer to the decompressed data, or NULL on failure. */
/*  *pOut_len will be set to the decompressed data's size, which could be larger
 * than src_buf_len on uncompressible data. */
/*  The caller must call mz_free() on the returned block when it's no longer
 * needed. */



MINIZ_EXPORT void *tinfl_decompress_mem_to_heap(const void *pSrc_buf,
                                                size_t src_buf_len,
                                                size_t *pOut_len, int flags);



#ifdef __cplusplus
}
#endif
/* tinfl_decompress_mem_to_mem() decompresses a block in memory to another block
 * in memory. */
/* Returns TINFL_DECOMPRESS_MEM_TO_MEM_FAILED on failure, or the number of bytes
 * written on success. */
#define TINFL_DECOMPRESS_MEM_TO_MEM_FAILED ((size_t)(-1))
MINIZ_EXPORT size_t tinfl_decompress_mem_to_mem(void *pOut_buf,
                                                size_t out_buf_len,
                                                const void *pSrc_buf,
                                                size_t src_buf_len, int flags);
/* 定义了一个函数，用于将内存中的一个块解压缩到另一个内存块中。
 * 在失败时返回 TINFL_DECOMPRESS_MEM_TO_MEM_FAILED，成功时返回写入的字节数。 */

/* tinfl_decompress_mem_to_callback() decompresses a block in memory to an
 * internal 32KB buffer, and a user provided callback function will be called to
 * flush the buffer. */
/* Returns 1 on success or 0 on failure. */
typedef int (*tinfl_put_buf_func_ptr)(const void *pBuf, int len, void *pUser);
MINIZ_EXPORT int
tinfl_decompress_mem_to_callback(const void *pIn_buf, size_t *pIn_buf_size,
                                 tinfl_put_buf_func_ptr pPut_buf_func,
                                 void *pPut_buf_user, int flags);
/* 定义了一个函数，用于将内存中的一个块解压缩到内部的32KB缓冲区中，
 * 并调用用户提供的回调函数来刷新缓冲区。
 * 在成功时返回1，失败时返回0。 */

struct tinfl_decompressor_tag;
typedef struct tinfl_decompressor_tag tinfl_decompressor;

#ifndef MINIZ_NO_MALLOC
/* Allocate the tinfl_decompressor structure in C so that */
/* non-C language bindings to tinfl_ API don't need to worry about */
/* structure size and allocation mechanism. */
MINIZ_EXPORT tinfl_decompressor *tinfl_decompressor_alloc(void);
MINIZ_EXPORT void tinfl_decompressor_free(tinfl_decompressor *pDecomp);
#endif
/* 在 C 中分配 tinfl_decompressor 结构，以便非 C 语言绑定到 tinfl_ API 不需要担心结构大小和分配机制。 */

/* Max size of LZ dictionary. */
#define TINFL_LZ_DICT_SIZE 32768
/* LZ 字典的最大大小。 */

/* Return status. */
} tinfl_status;
/* 返回状态。 */

/* Initializes the decompressor to its initial state. */
#define tinfl_init(r)                                                          \
  do {                                                                         \
/* 初始化解压缩器到其初始状态。 */
    (r)->m_state = 0;                                                          \  # 将指针 r 指向的结构体中的 m_state 成员设置为 0
  }                                                                            \  # 结束宏定义
  MZ_MACRO_END                                                                  # 宏定义结束
/* 定义宏，用于获取 Adler32 校验值 */
#define tinfl_get_adler32(r) (r)->m_check_adler32

/* 主要的低级解压缩协程函数。这是实际解压缩所需的唯一函数。
 * 所有其他函数只是用于提高可用性的高级辅助函数。 */
/* 这是一个通用的 API，即可用作构建任何所需的更高级解压缩 API 的构建块。
 * 在极限情况下，可以每输入或输出一个字节调用一次。 */
MINIZ_EXPORT tinfl_status tinfl_decompress(
    tinfl_decompressor *r, const mz_uint8 *pIn_buf_next, size_t *pIn_buf_size,
    mz_uint8 *pOut_buf_start, mz_uint8 *pOut_buf_next, size_t *pOut_buf_size,
    const mz_uint32 decomp_flags);

/* 内部/私有部分如下。 */
enum {
  TINFL_MAX_HUFF_TABLES = 3,
  TINFL_MAX_HUFF_SYMBOLS_0 = 288,
  TINFL_MAX_HUFF_SYMBOLS_1 = 32,
  TINFL_MAX_HUFF_SYMBOLS_2 = 19,
  TINFL_FAST_LOOKUP_BITS = 10,
  TINFL_FAST_LOOKUP_SIZE = 1 << TINFL_FAST_LOOKUP_BITS
};

typedef struct {
  mz_uint8 m_code_size[TINFL_MAX_HUFF_SYMBOLS_0];
  mz_int16 m_look_up[TINFL_FAST_LOOKUP_SIZE],
      m_tree[TINFL_MAX_HUFF_SYMBOLS_0 * 2];
} tinfl_huff_table;

#if MINIZ_HAS_64BIT_REGISTERS
#define TINFL_USE_64BIT_BITBUF 1
#else
#define TINFL_USE_64BIT_BITBUF 0
#endif

#if TINFL_USE_64BIT_BITBUF
typedef mz_uint64 tinfl_bit_buf_t;
#define TINFL_BITBUF_SIZE (64)
#else
typedef mz_uint32 tinfl_bit_buf_t;
#define TINFL_BITBUF_SIZE (32)
#endif

struct tinfl_decompressor_tag {
  mz_uint32 m_state, m_num_bits, m_zhdr0, m_zhdr1, m_z_adler32, m_final, m_type,
      m_check_adler32, m_dist, m_counter, m_num_extra,
      m_table_sizes[TINFL_MAX_HUFF_TABLES];
  tinfl_bit_buf_t m_bit_buf;
  size_t m_dist_from_out_buf_start;
  tinfl_huff_table m_tables[TINFL_MAX_HUFF_TABLES];
  mz_uint8 m_raw_header[4],
      m_len_codes[TINFL_MAX_HUFF_SYMBOLS_0 + TINFL_MAX_HUFF_SYMBOLS_1 + 137];
};

#ifdef __cplusplus
}
#endif

#pragma once

/* ------------------- ZIP archive reading/writing */

#ifndef MINIZ_NO_ARCHIVE_APIS

#ifdef __cplusplus
extern "C" {
#endif

enum {
  /* 定义枚举常量，用于设置最大输入输出缓冲区大小 */
  MZ_ZIP_MAX_IO_BUF_SIZE = 8 * 1024,
  /* 定义枚举常量，用于设置最大存档文件名大小 */
  MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE = 512,
  /* 定义枚举常量，用于设置最大存档文件注释大小 */
  MZ_ZIP_MAX_ARCHIVE_FILE_COMMENT_SIZE = 512
};

typedef struct {
  /* 存档文件的中央目录文件索引 */
  mz_uint32 m_file_index;

  /* 存档文件中此条目的字节偏移量。注意，我们目前仅支持中央目录中最多 UINT_MAX 或更少字节。 */
  mz_uint64 m_central_dir_ofs;

  /* 这些字段直接从 zip 的中央目录中复制。 */
  mz_uint16 m_version_made_by;
  mz_uint16 m_version_needed;
  mz_uint16 m_bit_flag;
  mz_uint16 m_method;

#ifndef MINIZ_NO_TIME
  /* 存档文件的时间戳 */
  MZ_TIME_T m_time;
#endif

  /* CRC-32 of uncompressed data. */
  mz_uint32 m_crc32;

  /* File's compressed size. */
  mz_uint64 m_comp_size;

  /* File's uncompressed size. Note, I've seen some old archives where directory
   * entries had 512 bytes for their uncompressed sizes, but when you try to
   * unpack them you actually get 0 bytes. */
  mz_uint64 m_uncomp_size;

  /* Zip internal and external file attributes. */
  mz_uint16 m_internal_attr;
  mz_uint32 m_external_attr;

  /* Entry's local header file offset in bytes. */
  mz_uint64 m_local_header_ofs;

  /* Size of comment in bytes. */
  mz_uint32 m_comment_size;

  /* MZ_TRUE if the entry appears to be a directory. */
  mz_bool m_is_directory;

  /* MZ_TRUE if the entry uses encryption/strong encryption (which miniz_zip
   * doesn't support) */
  mz_bool m_is_encrypted;

  /* MZ_TRUE if the file is not encrypted, a patch file, and if it uses a
   * compression method we support. */
  mz_bool m_is_supported;

  /* Filename. If string ends in '/' it's a subdirectory entry. */
  /* Guaranteed to be zero terminated, may be truncated to fit. */
  char m_filename[MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE];

  /* Comment field. */
  /* Guaranteed to be zero terminated, may be truncated to fit. */
  char m_comment[MZ_ZIP_MAX_ARCHIVE_FILE_COMMENT_SIZE];

} mz_zip_archive_file_stat;

typedef size_t (*mz_file_read_func)(void *pOpaque, mz_uint64 file_ofs,
                                    void *pBuf, size_t n);
typedef size_t (*mz_file_write_func)(void *pOpaque, mz_uint64 file_ofs,
                                     const void *pBuf, size_t n);
typedef mz_bool (*mz_file_needs_keepalive)(void *pOpaque);

struct mz_zip_internal_state_tag;
typedef struct mz_zip_internal_state_tag mz_zip_internal_state;

typedef enum {
  MZ_ZIP_MODE_INVALID = 0,
  MZ_ZIP_MODE_READING = 1,
  MZ_ZIP_MODE_WRITING = 2,
  MZ_ZIP_MODE_WRITING_HAS_BEEN_FINALIZED = 3
} mz_zip_mode;
# 定义枚举类型 mz_zip_flags，表示 ZIP 文件的标志位
typedef enum {
  MZ_ZIP_FLAG_CASE_SENSITIVE = 0x0100,  # 区分大小写
  MZ_ZIP_FLAG_IGNORE_PATH = 0x0200,  # 忽略路径
  MZ_ZIP_FLAG_COMPRESSED_DATA = 0x0400,  # 压缩数据
  MZ_ZIP_FLAG_DO_NOT_SORT_CENTRAL_DIRECTORY = 0x0800,  # 不排序中央目录
  MZ_ZIP_FLAG_VALIDATE_LOCATE_FILE_FLAG =
      0x1000, /* 如果启用，将在验证每个文件时调用 mz_zip_reader_locate_file()，以确保该函数在中央目录中找到文件（用于测试） */
  MZ_ZIP_FLAG_VALIDATE_HEADERS_ONLY =
      0x2000, /* 验证本地头部，但不解压整个文件并检查 crc32 */
  MZ_ZIP_FLAG_WRITE_ZIP64 =
      0x4000, /* 始终使用 zip64 文件格式，而不是原始 zip 文件格式，并自动切换到 zip64。作为 mz_zip_writer_init*_v2 的标志参数使用 */
  MZ_ZIP_FLAG_WRITE_ALLOW_READING = 0x8000,  # 允许读取
  MZ_ZIP_FLAG_ASCII_FILENAME = 0x10000,  # ASCII 文件名
  /* 添加压缩文件后，将定位回本地文件头并设置正确的大小 */
  MZ_ZIP_FLAG_WRITE_HEADER_SET_SIZE = 0x20000
} mz_zip_flags;

# 定义枚举类型 mz_zip_type，表示 ZIP 文件的类型
typedef enum {
  MZ_ZIP_TYPE_INVALID = 0,  # 无效类型
  MZ_ZIP_TYPE_USER,  # 用户类型
  MZ_ZIP_TYPE_MEMORY,  # 内存类型
  MZ_ZIP_TYPE_HEAP,  # 堆类型
  MZ_ZIP_TYPE_FILE,  # 文件类型
  MZ_ZIP_TYPE_CFILE,  # C 文件类型
  MZ_ZIP_TOTAL_TYPES  # 总类型数
} mz_zip_type;

# miniz 错误代码。如果添加或修改此枚举，请确保更新 mz_zip_get_error_string()
# 定义枚举类型，表示 ZIP 操作可能出现的错误
typedef enum {
  MZ_ZIP_NO_ERROR = 0,  # 没有错误
  MZ_ZIP_UNDEFINED_ERROR,  # 未定义的错误
  MZ_ZIP_TOO_MANY_FILES,  # 文件数量过多
  MZ_ZIP_FILE_TOO_LARGE,  # 文件过大
  MZ_ZIP_UNSUPPORTED_METHOD,  # 不支持的压缩方法
  MZ_ZIP_UNSUPPORTED_ENCRYPTION,  # 不支持的加密方式
  MZ_ZIP_UNSUPPORTED_FEATURE,  # 不支持的功能
  MZ_ZIP_FAILED_FINDING_CENTRAL_DIR,  # 无法找到中央目录
  MZ_ZIP_NOT_AN_ARCHIVE,  # 不是一个 ZIP 文件
  MZ_ZIP_INVALID_HEADER_OR_CORRUPTED,  # 无效的头部或损坏
  MZ_ZIP_UNSUPPORTED_MULTIDISK,  # 不支持多磁盘
  MZ_ZIP_DECOMPRESSION_FAILED,  # 解压失败
  MZ_ZIP_COMPRESSION_FAILED,  # 压缩失败
  MZ_ZIP_UNEXPECTED_DECOMPRESSED_SIZE,  # 解压后大小不符合预期
  MZ_ZIP_CRC_CHECK_FAILED,  # CRC 校验失败
  MZ_ZIP_UNSUPPORTED_CDIR_SIZE,  # 不支持的中央目录大小
  MZ_ZIP_ALLOC_FAILED,  # 分配内存失败
  MZ_ZIP_FILE_OPEN_FAILED,  # 打开文件失败
  MZ_ZIP_FILE_CREATE_FAILED,  # 创建文件失败
  MZ_ZIP_FILE_WRITE_FAILED,  # 写入文件失败
  MZ_ZIP_FILE_READ_FAILED,  # 读取文件失败
  MZ_ZIP_FILE_CLOSE_FAILED,  # 关闭文件失败
  MZ_ZIP_FILE_SEEK_FAILED,  # 定位文件失败
  MZ_ZIP_FILE_STAT_FAILED,  # 获取文件状态失败
  MZ_ZIP_INVALID_PARAMETER,  # 无效参数
  MZ_ZIP_INVALID_FILENAME,  # 无效文件名
  MZ_ZIP_BUF_TOO_SMALL,  # 缓冲区太小
  MZ_ZIP_INTERNAL_ERROR,  # 内部错误
  MZ_ZIP_FILE_NOT_FOUND,  # 文件未找到
  MZ_ZIP_ARCHIVE_TOO_LARGE,  # ZIP 文件过大
  MZ_ZIP_VALIDATION_FAILED,  # 验证失败
  MZ_ZIP_WRITE_CALLBACK_FAILED,  # 写入回调失败
  MZ_ZIP_TOTAL_ERRORS  # 错误总数
} mz_zip_error;

# 定义 ZIP 存档结构体
typedef struct {
  mz_uint64 m_archive_size;  # 存档大小
  mz_uint64 m_central_directory_file_ofs;  # 中央目录文件偏移量

  /* We only support up to UINT32_MAX files in zip64 mode. */
  mz_uint32 m_total_files;  # 文件总数
  mz_zip_mode m_zip_mode;  # ZIP 模式
  mz_zip_type m_zip_type;  # ZIP 类型
  mz_zip_error m_last_error;  # 最后的错误

  mz_uint64 m_file_offset_alignment;  # 文件偏移对齐

  mz_alloc_func m_pAlloc;  # 分配函数
  mz_free_func m_pFree;  # 释放函数
  mz_realloc_func m_pRealloc;  # 重新分配函数
  void *m_pAlloc_opaque;  # 分配函数的私有数据

  mz_file_read_func m_pRead;  # 读取函数
  mz_file_write_func m_pWrite;  # 写入函数
  mz_file_needs_keepalive m_pNeeds_keepalive;  # 保持活动状态函数
  void *m_pIO_opaque;  # IO 操作的私有数据

  mz_zip_internal_state *m_pState;  # 内部状态

} mz_zip_archive;

# 定义 ZIP 读取器提取迭代状态结构体
typedef struct {
  mz_zip_archive *pZip;  # ZIP 存档指针
  mz_uint flags;  # 标志

  int status;  # 状态
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
  mz_uint file_crc32;  # 文件 CRC32 校验值
#endif
  mz_uint64 read_buf_size, read_buf_ofs, read_buf_avail, comp_remaining,
      out_buf_ofs, cur_file_ofs;  # 读取缓冲区大小、偏移、可用大小，压缩剩余大小，输出缓冲区偏移，当前文件偏移

  mz_zip_archive_file_stat file_stat;  # 文件状态
  void *pRead_buf;  # 读取缓冲区
  void *pWrite_buf;  # 写入缓冲区

  size_t out_blk_remain;  # 输出块剩余大小

  tinfl_decompressor inflator;  # 解压器

} mz_zip_reader_extract_iter_state;
/* -------- ZIP reading */

/* 初始化一个 ZIP 存档读取器。 */
/* 这些函数读取并验证存档的中央目录。 */
MINIZ_EXPORT mz_bool mz_zip_reader_init(mz_zip_archive *pZip, mz_uint64 size,
                                        mz_uint flags);

/* 从内存中初始化 ZIP 存档读取器。 */
MINIZ_EXPORT mz_bool mz_zip_reader_init_mem(mz_zip_archive *pZip,
                                            const void *pMem, size_t size,
                                            mz_uint flags);

#ifndef MINIZ_NO_STDIO
/* 从磁盘文件中读取存档。 */
/* file_start_ofs 是存档实际开始的文件偏移量，或者为 0。 */
/* actual_archive_size 是存档的真实总大小，可能小于磁盘上文件的实际大小。如果为零，则整个文件被视为存档。 */
MINIZ_EXPORT mz_bool mz_zip_reader_init_file(mz_zip_archive *pZip,
                                             const char *pFilename,
                                             mz_uint32 flags);
MINIZ_EXPORT mz_bool mz_zip_reader_init_file_v2(mz_zip_archive *pZip,
                                                const char *pFilename,
                                                mz_uint flags,
                                                mz_uint64 file_start_ofs,
                                                mz_uint64 archive_size);
MINIZ_EXPORT mz_bool mz_zip_reader_init_file_v2_rpb(mz_zip_archive *pZip,
                                                    const char *pFilename,
                                                    mz_uint flags,
                                                    mz_uint64 file_start_ofs,
                                                    mz_uint64 archive_size);

/* 从已经打开的 FILE 中读取存档，从当前文件位置开始。 */
/* 假定存档的长度为 archive_size 字节。如果 archive_size 为 0，则假定整个文件的剩余部分包含存档。 */
/* 初始化 ZIP 读取器，使用 C 文件指针作为输入，不会在调用 mz_zip_reader_end() 时关闭文件 */
MINIZ_EXPORT mz_bool mz_zip_reader_init_cfile(mz_zip_archive *pZip,
                                              MZ_FILE *pFile,
                                              mz_uint64 archive_size,
                                              mz_uint flags);
#endif

/* 结束对归档的读取，释放所有分配的内存，并在使用 mz_zip_reader_init_file() 时关闭输入归档文件 */
MINIZ_EXPORT mz_bool mz_zip_reader_end(mz_zip_archive *pZip);

/* -------- ZIP 读取或写入 */

/* 将 mz_zip_archive 结构清零 */
/* 重要：在将结构传递给任何 mz_zip 函数之前必须执行此操作 */
MINIZ_EXPORT void mz_zip_zero_struct(mz_zip_archive *pZip);

MINIZ_EXPORT mz_zip_mode mz_zip_get_mode(mz_zip_archive *pZip);
MINIZ_EXPORT mz_zip_type mz_zip_get_type(mz_zip_archive *pZip);

/* 返回归档中的文件总数 */
MINIZ_EXPORT mz_uint mz_zip_reader_get_num_files(mz_zip_archive *pZip);

MINIZ_EXPORT mz_uint64 mz_zip_get_archive_size(mz_zip_archive *pZip);
MINIZ_EXPORT mz_uint64
mz_zip_get_archive_file_start_offset(mz_zip_archive *pZip);
MINIZ_EXPORT MZ_FILE *mz_zip_get_cfile(mz_zip_archive *pZip);

/* 从文件偏移 file_ofs 开始读取 n 字节的原始归档数据到 pBuf */
MINIZ_EXPORT size_t mz_zip_read_archive_data(mz_zip_archive *pZip,
                                             mz_uint64 file_ofs, void *pBuf,
                                             size_t n);

/* 所有 mz_zip 函数都会设置 mz_zip_archive 结构中的 m_last_error 字段 */
/* 这些函数用于检索/操作此字段 */
/* 请注意，m_last_error 功能不是线程安全的 */
MINIZ_EXPORT mz_zip_error mz_zip_set_last_error(mz_zip_archive *pZip,
                                                mz_zip_error err_num);
MINIZ_EXPORT mz_zip_error mz_zip_peek_last_error(mz_zip_archive *pZip);
# 清除 ZIP 操作的最后一个错误
MINIZ_EXPORT mz_zip_error mz_zip_clear_last_error(mz_zip_archive *pZip);

# 获取 ZIP 操作的最后一个错误
MINIZ_EXPORT mz_zip_error mz_zip_get_last_error(mz_zip_archive *pZip);

# 获取 ZIP 错误码对应的错误字符串
MINIZ_EXPORT const char *mz_zip_get_error_string(mz_zip_error mz_err);

# 判断归档文件条目是否为目录条目
MINIZ_EXPORT mz_bool mz_zip_reader_is_file_a_directory(mz_zip_archive *pZip,
                                                       mz_uint file_index);

# 判断文件是否加密/强加密
MINIZ_EXPORT mz_bool mz_zip_reader_is_file_encrypted(mz_zip_archive *pZip,
                                                     mz_uint file_index);

# 判断压缩方法是否受支持，文件未加密，且不是压缩补丁文件
MINIZ_EXPORT mz_bool mz_zip_reader_is_file_supported(mz_zip_archive *pZip,
                                                     mz_uint file_index);

# 获取归档文件条目的文件名
# 返回写入 pFilename 的字节数，如果 filename_buf_size 为 0，则返回完全存储文件名所需的字节数
MINIZ_EXPORT mz_uint mz_zip_reader_get_filename(mz_zip_archive *pZip,
                                                mz_uint file_index,
                                                char *pFilename,
                                                mz_uint filename_buf_size);

# 尝试在归档的中央目录中定位文件
# 有效标志：MZ_ZIP_FLAG_CASE_SENSITIVE，MZ_ZIP_FLAG_IGNORE_PATH
# 如果找不到文件，则返回 -1
MINIZ_EXPORT int mz_zip_reader_locate_file(mz_zip_archive *pZip,
                                           const char *pName,
                                           const char *pComment, mz_uint flags);
/* 定义函数，用于在 ZIP 存档中定位文件 */
MINIZ_EXPORT mz_bool mz_zip_reader_locate_file_v2(mz_zip_archive *pZip,
                                                  const char *pName,
                                                  const char *pComment,
                                                  mz_uint flags,
                                                  mz_uint32 *file_index);

/* 返回有关存档文件条目的详细信息 */
MINIZ_EXPORT mz_bool mz_zip_reader_file_stat(mz_zip_archive *pZip,
                                             mz_uint file_index,
                                             mz_zip_archive_file_stat *pStat);

/* 如果文件是 zip64 格式，则返回 MZ_TRUE */
/* 如果文件包含 zip64 结束中央目录标记，或者如果文件包含中央目录中的任何 zip64 扩展文件信息字段，则被视为 zip64 */
MINIZ_EXPORT mz_bool mz_zip_is_zip64(mz_zip_archive *pZip);

/* 返回中央目录的总大小（以字节为单位） */
/* 当前支持的最大大小 <= MZ_UINT32_MAX */
MINIZ_EXPORT size_t mz_zip_get_central_dir_size(mz_zip_archive *pZip);

/* 使用无需内存分配将存档文件提取到内存缓冲区 */
/* 栈上必须至少有足够的空间来存储解压器的状态（大约 34KB 左右） */
MINIZ_EXPORT mz_bool mz_zip_reader_extract_to_mem_no_alloc(
    mz_zip_archive *pZip, mz_uint file_index, void *pBuf, size_t buf_size,
    mz_uint flags, void *pUser_read_buf, size_t user_read_buf_size);
MINIZ_EXPORT mz_bool mz_zip_reader_extract_file_to_mem_no_alloc(
    mz_zip_archive *pZip, const char *pFilename, void *pBuf, size_t buf_size,
    mz_uint flags, void *pUser_read_buf, size_t user_read_buf_size);

/* 将存档文件提取到内存缓冲区 */
# 从 ZIP 存档中提取文件到内存
MINIZ_EXPORT mz_bool mz_zip_reader_extract_to_mem(mz_zip_archive *pZip,
                                                  mz_uint file_index,
                                                  void *pBuf, size_t buf_size,
                                                  mz_uint flags);
# 从 ZIP 存档中提取文件到内存
MINIZ_EXPORT mz_bool mz_zip_reader_extract_file_to_mem(mz_zip_archive *pZip,
                                                       const char *pFilename,
                                                       void *pBuf,
                                                       size_t buf_size,
                                                       mz_uint flags);

# 将存档文件提取到动态分配的堆缓冲区中
# 内存将通过 mz_zip_archive 的分配/重新分配函数分配
# 失败时返回 NULL 并设置最后一个错误
MINIZ_EXPORT void *mz_zip_reader_extract_to_heap(mz_zip_archive *pZip,
                                                 mz_uint file_index,
                                                 size_t *pSize, mz_uint flags);
# 将存档文件提取到动态分配的堆缓冲区中
# 内存将通过 mz_zip_archive 的分配/重新分配函数分配
# 失败时返回 NULL 并设置最后一个错误
MINIZ_EXPORT void *mz_zip_reader_extract_file_to_heap(mz_zip_archive *pZip,
                                                      const char *pFilename,
                                                      size_t *pSize,
                                                      mz_uint flags);

# 使用回调函数提取存档文件的数据
MINIZ_EXPORT mz_bool mz_zip_reader_extract_to_callback(
    mz_zip_archive *pZip, mz_uint file_index, mz_file_write_func pCallback,
    void *pOpaque, mz_uint flags);
# 使用回调函数提取存档文件的数据
MINIZ_EXPORT mz_bool mz_zip_reader_extract_file_to_callback(
    mz_zip_archive *pZip, const char *pFilename, mz_file_write_func pCallback,
    void *pOpaque, mz_uint flags);

# 迭代地提取文件
MINIZ_EXPORT mz_zip_reader_extract_iter_state *
/* 创建一个新的 ZIP 文件读取器，用于迭代提取文件 */
mz_zip_reader_extract_iter_new(mz_zip_archive *pZip, mz_uint file_index,
                               mz_uint flags);

/* 创建一个新的 ZIP 文件读取器，用于迭代提取指定文件 */
MINIZ_EXPORT mz_zip_reader_extract_iter_state *
mz_zip_reader_extract_file_iter_new(mz_zip_archive *pZip, const char *pFilename,
                                    mz_uint flags);

/* 从 ZIP 文件读取器中读取数据到缓冲区 */
MINIZ_EXPORT size_t mz_zip_reader_extract_iter_read(
    mz_zip_reader_extract_iter_state *pState, void *pvBuf, size_t buf_size);

/* 释放 ZIP 文件读取器 */
MINIZ_EXPORT mz_bool
mz_zip_reader_extract_iter_free(mz_zip_reader_extract_iter_state *pState);

#ifndef MINIZ_NO_STDIO
/* 将 ZIP 文件中的文件提取到磁盘文件，并设置其最后访问和修改时间 */
/* 该函数仅提取文件，不包括存档目录记录 */
MINIZ_EXPORT mz_bool mz_zip_reader_extract_to_file(mz_zip_archive *pZip,
                                                   mz_uint file_index,
                                                   const char *pDst_filename,
                                                   mz_uint flags);

/* 将 ZIP 文件中的指定文件提取到磁盘文件 */
MINIZ_EXPORT mz_bool mz_zip_reader_extract_file_to_file(
    mz_zip_archive *pZip, const char *pArchive_filename,
    const char *pDst_filename, mz_uint flags);

/* 从当前位置开始将 ZIP 文件中的文件提取到目标 FILE 流 */
MINIZ_EXPORT mz_bool mz_zip_reader_extract_to_cfile(mz_zip_archive *pZip,
                                                    mz_uint file_index,
                                                    MZ_FILE *File,
                                                    mz_uint flags);

/* 将 ZIP 文件中的指定文件提取到目标 FILE 流 */
MINIZ_EXPORT mz_bool mz_zip_reader_extract_file_to_cfile(
    mz_zip_archive *pZip, const char *pArchive_filename, MZ_FILE *pFile,
    mz_uint flags);
#endif

#if 0
/* TODO */
    /* 定义一个用于流式提取的 ZIP 文件状态指针 */
    typedef void *mz_zip_streaming_extract_state_ptr;
    /* 开始流式提取指定文件 */
    mz_zip_streaming_extract_state_ptr mz_zip_streaming_extract_begin(mz_zip_archive *pZip, mz_uint file_index, mz_uint flags);
    # 获取正在解压的文件的大小
    uint64_t mz_zip_streaming_extract_get_size(mz_zip_archive *pZip, mz_zip_streaming_extract_state_ptr pState);
    
    # 获取当前正在解压的文件的偏移量
    uint64_t mz_zip_streaming_extract_get_cur_ofs(mz_zip_archive *pZip, mz_zip_streaming_extract_state_ptr pState);
    
    # 设置正在解压的文件的偏移量
    mz_bool mz_zip_streaming_extract_seek(mz_zip_archive *pZip, mz_zip_streaming_extract_state_ptr pState, uint64_t new_ofs);
    
    # 读取正在解压的文件的数据到缓冲区
    size_t mz_zip_streaming_extract_read(mz_zip_archive *pZip, mz_zip_streaming_extract_state_ptr pState, void *pBuf, size_t buf_size);
    
    # 结束当前的解压操作
    mz_bool mz_zip_streaming_extract_end(mz_zip_archive *pZip, mz_zip_streaming_extract_state_ptr pState);
#endif

/* This function compares the archive's local headers, the optional local zip64
 * extended information block, and the optional descriptor following the
 * compressed data vs. the data in the central directory. */
/* It also validates that each file can be successfully uncompressed unless the
 * MZ_ZIP_FLAG_VALIDATE_HEADERS_ONLY is specified. */
MINIZ_EXPORT mz_bool mz_zip_validate_file(mz_zip_archive *pZip,
                                          mz_uint file_index, mz_uint flags);

/* Validates an entire archive by calling mz_zip_validate_file() on each file.
 */
MINIZ_EXPORT mz_bool mz_zip_validate_archive(mz_zip_archive *pZip,
                                             mz_uint flags);

/* Misc utils/helpers, valid for ZIP reading or writing */
MINIZ_EXPORT mz_bool mz_zip_validate_mem_archive(const void *pMem, size_t size,
                                                 mz_uint flags,
                                                 mz_zip_error *pErr);
MINIZ_EXPORT mz_bool mz_zip_validate_file_archive(const char *pFilename,
                                                  mz_uint flags,
                                                  mz_zip_error *pErr);

/* Universal end function - calls either mz_zip_reader_end() or
 * mz_zip_writer_end(). */
MINIZ_EXPORT mz_bool mz_zip_end(mz_zip_archive *pZip);

/* -------- ZIP writing */

#ifndef MINIZ_NO_ARCHIVE_WRITING_APIS

/* Inits a ZIP archive writer. */
/*Set pZip->m_pWrite (and pZip->m_pIO_opaque) before calling mz_zip_writer_init
 * or mz_zip_writer_init_v2*/
/*The output is streamable, i.e. file_ofs in mz_file_write_func always increases
 * only by n*/
MINIZ_EXPORT mz_bool mz_zip_writer_init(mz_zip_archive *pZip,
                                        mz_uint64 existing_size);
MINIZ_EXPORT mz_bool mz_zip_writer_init_v2(mz_zip_archive *pZip,
                                           mz_uint64 existing_size,
                                           mz_uint flags);
# 初始化一个使用堆内存的 ZIP 写入器，用于创建 ZIP 存档
MINIZ_EXPORT mz_bool mz_zip_writer_init_heap(
    mz_zip_archive *pZip, size_t size_to_reserve_at_beginning,
    size_t initial_allocation_size);
# 初始化一个使用堆内存的 ZIP 写入器，用于创建 ZIP 存档，带有额外的标志参数
MINIZ_EXPORT mz_bool mz_zip_writer_init_heap_v2(
    mz_zip_archive *pZip, size_t size_to_reserve_at_beginning,
    size_t initial_allocation_size, mz_uint flags);

# 如果未定义 MINIZ_NO_STDIO，则以下函数可用

# 初始化一个使用文件的 ZIP 写入器，用于创建 ZIP 存档
MINIZ_EXPORT mz_bool
mz_zip_writer_init_file(mz_zip_archive *pZip, const char *pFilename,
                        mz_uint64 size_to_reserve_at_beginning);
# 初始化一个使用文件的 ZIP 写入器，用于创建 ZIP 存档，带有额外的标志参数
MINIZ_EXPORT mz_bool mz_zip_writer_init_file_v2(
    mz_zip_archive *pZip, const char *pFilename,
    mz_uint64 size_to_reserve_at_beginning, mz_uint flags);
# 初始化一个使用 C 文件的 ZIP 写入器，用于创建 ZIP 存档
MINIZ_EXPORT mz_bool mz_zip_writer_init_cfile(mz_zip_archive *pZip,
                                              MZ_FILE *pFile, mz_uint flags);

# 将 ZIP 存档读取器对象转换为写入器对象，以允许对现有存档进行高效的原地文件追加
# 对于使用 mz_zip_reader_init_file 打开的存档，pFilename 必须是存档的文件名，以便重新打开进行写入
# 如果无法重新打开文件，则将调用 mz_zip_reader_end()
# 对于使用 mz_zip_reader_init_mem 打开的存档，内存块必须可通过 realloc 回调进行扩展（默认为 realloc，除非您已覆盖它）
# 最后，对于使用 mz_zip_reader_init 打开的存档，mz_zip_archive 的用户提供的 m_pWrite 函数不能为 NULL
# 注意：不建议进行原地存档修改，除非您知道自己在做什么，因为如果在存档完成之前执行停止或出现问题
# 存档的中央目录将被破坏
MINIZ_EXPORT mz_bool mz_zip_writer_init_from_reader(mz_zip_archive *pZip,
                                                    const char *pFilename);
# 初始化一个 ZIP 归档写入器，从一个 ZIP 读取器中初始化
MINIZ_EXPORT mz_bool mz_zip_writer_init_from_reader_v2(mz_zip_archive *pZip,
                                                       const char *pFilename,
                                                       mz_uint flags);
# 初始化一个 ZIP 归档写入器，从一个 ZIP 读取器中初始化，不重新打开文件
MINIZ_EXPORT mz_bool mz_zip_writer_init_from_reader_v2_noreopen(
    mz_zip_archive *pZip, const char *pFilename, mz_uint flags);

# 将内存缓冲区的内容添加到归档中。这些函数将当前本地时间记录到归档中。
# 要添加一个目录条目，调用此方法并以斜杠结尾的归档名称和空缓冲区。
# level_and_flags - 压缩级别（0-10，参见 MZ_BEST_SPEED、MZ_BEST_COMPRESSION 等），逻辑上与一个或多个 mz_zip_flags 进行 OR 运算，或者设置为 MZ_DEFAULT_COMPRESSION。
MINIZ_EXPORT mz_bool mz_zip_writer_add_mem(mz_zip_archive *pZip,
                                           const char *pArchive_name,
                                           const void *pBuf, size_t buf_size,
                                           mz_uint level_and_flags);

# 类似于 mz_zip_writer_add_mem()，但可以指定文件注释字段，并可选择向函数提供已经压缩的数据。
# 如果指定了 MZ_ZIP_FLAG_COMPRESSED_DATA 标志，则仅使用 uncomp_size/uncomp_crc32。
MINIZ_EXPORT mz_bool mz_zip_writer_add_mem_ex(
    mz_zip_archive *pZip, const char *pArchive_name, const void *pBuf,
    size_t buf_size, const void *pComment, mz_uint16 comment_size,
    mz_uint level_and_flags, mz_uint64 uncomp_size, mz_uint32 uncomp_crc32);

# 类似于 mz_zip_writer_add_mem_ex()，但可以指定最后修改时间、本地用户额外数据、中央用户额外数据等。
MINIZ_EXPORT mz_bool mz_zip_writer_add_mem_ex_v2(
    mz_zip_archive *pZip, const char *pArchive_name, const void *pBuf,
    size_t buf_size, const void *pComment, mz_uint16 comment_size,
    mz_uint level_and_flags, mz_uint64 uncomp_size, mz_uint32 uncomp_crc32,
    MZ_TIME_T *last_modified, const char *user_extra_data_local,
    mz_uint user_extra_data_local_len, const char *user_extra_data_central,
    mz_uint user_extra_data_central_len);

这行代码定义了一个函数参数 `user_extra_data_central_len`，类型为 `mz_uint`，但是缺少了函数名和函数体，无法确定具体作用。
/* Adds the contents of a file to an archive. This function also records the
 * disk file's modified time into the archive. */
/* File data is supplied via a read callback function. User
 * mz_zip_writer_add_(c)file to add a file directly.*/
MINIZ_EXPORT mz_bool mz_zip_writer_add_read_buf_callback(
    mz_zip_archive *pZip, const char *pArchive_name,
    mz_file_read_func read_callback, void *callback_opaque, mz_uint64 max_size,
    const MZ_TIME_T *pFile_time, const void *pComment, mz_uint16 comment_size,
    mz_uint level_and_flags, mz_uint32 ext_attributes,
    const char *user_extra_data_local, mz_uint user_extra_data_local_len,
    const char *user_extra_data_central, mz_uint user_extra_data_central_len);

#ifndef MINIZ_NO_STDIO
/* Adds the contents of a disk file to an archive. This function also records
 * the disk file's modified time into the archive. */
/* level_and_flags - compression level (0-10, see MZ_BEST_SPEED,
 * MZ_BEST_COMPRESSION, etc.) logically OR'd with zero or more mz_zip_flags, or
 * just set to MZ_DEFAULT_COMPRESSION. */
MINIZ_EXPORT mz_bool mz_zip_writer_add_file(
    mz_zip_archive *pZip, const char *pArchive_name, const char *pSrc_filename,
    const void *pComment, mz_uint16 comment_size, mz_uint level_and_flags,
    mz_uint32 ext_attributes);

/* Like mz_zip_writer_add_file(), except the file data is read from the
 * specified FILE stream. */
MINIZ_EXPORT mz_bool mz_zip_writer_add_cfile(
    mz_zip_archive *pZip, const char *pArchive_name, MZ_FILE *pSrc_file,
    mz_uint64 max_size, const MZ_TIME_T *pFile_time, const void *pComment,
    mz_uint16 comment_size, mz_uint level_and_flags, mz_uint32 ext_attributes,
    const char *user_extra_data_local, mz_uint user_extra_data_local_len,
    const char *user_extra_data_central, mz_uint user_extra_data_central_len);
#endif

/* Adds a file to an archive by fully cloning the data from another archive. */
/* This function fully clones the source file's compressed data (no
 * recompression), along with its full filename, extra data (it may add or
 * modify the zip64 local header extra data field), and the optional descriptor
 * following the compressed data. */
/* 从源 ZIP 读取器中完全克隆源文件的压缩数据（不重新压缩），以及完整的文件名、额外数据（可能添加或修改 zip64 本地头额外数据字段）和压缩数据后的可选描述符。 */
MINIZ_EXPORT mz_bool mz_zip_writer_add_from_zip_reader(
    mz_zip_archive *pZip, mz_zip_archive *pSource_zip, mz_uint src_file_index);

/* Finalizes the archive by writing the central directory records followed by
 * the end of central directory record. */
/* After an archive is finalized, the only valid call on the mz_zip_archive
 * struct is mz_zip_writer_end(). */
/* An archive must be manually finalized by calling this function for it to be
 * valid. */
/* 通过写入中央目录记录，然后写入中央目录记录结束来完成归档。 */
/* 在归档完成后，对 mz_zip_archive 结构的唯一有效调用是 mz_zip_writer_end()。 */
/* 必须通过调用此函数手动完成归档才能使其有效。 */
MINIZ_EXPORT mz_bool mz_zip_writer_finalize_archive(mz_zip_archive *pZip);

/* Finalizes a heap archive, returning a pointer to the heap block and its size.
 */
/* The heap block will be allocated using the mz_zip_archive's alloc/realloc
 * callbacks. */
/* 完成堆归档，返回指向堆块及其大小的指针。 */
/* 堆块将使用 mz_zip_archive 的 alloc/realloc 回调函数分配。 */
MINIZ_EXPORT mz_bool mz_zip_writer_finalize_heap_archive(mz_zip_archive *pZip,
                                                         void **ppBuf,
                                                         size_t *pSize);

/* Ends archive writing, freeing all allocations, and closing the output file if
 * mz_zip_writer_init_file() was used. */
/* Note for the archive to be valid, it *must* have been finalized before ending
 * (this function will not do it for you). */
/* 结束归档写入，释放所有分配，并在使用 mz_zip_writer_init_file() 时关闭输出文件。 */
/* 注意，为了使归档有效，必须在结束之前完成归档（此函数不会为您完成）。 */
MINIZ_EXPORT mz_bool mz_zip_writer_end(mz_zip_archive *pZip);

/* -------- Misc. high-level helper functions: */

/* mz_zip_add_mem_to_archive_file_in_place() efficiently (but not atomically)
 * appends a memory blob to a ZIP archive. */
/* Note this is NOT a fully safe operation. If it crashes or dies in some way
 * your archive can be left in a screwed up state (without a central directory).
 */
/* mz_zip_add_mem_to_archive_file_in_place() 高效（但不是原子性地）将内存块附加到 ZIP 归档中。 */
/* 请注意，这不是完全安全的操作。如果以某种方式崩溃或死机，您的归档可能会处于混乱状态（没有中央目录）。 */
/* level_and_flags - 压缩级别（0-10，参见MZ_BEST_SPEED、MZ_BEST_COMPRESSION等）与一个或多个mz_zip_flags逻辑OR，或者设置为MZ_DEFAULT_COMPRESSION。 */
/* TODO: 或许添加一个选项，在添加失败时保留现有的中央目录？如果出现问题，我们可以截断文件（这样旧的中央目录将在末尾），如果出现问题。 */
MINIZ_EXPORT mz_bool mz_zip_add_mem_to_archive_file_in_place(
    const char *pZip_filename, const char *pArchive_name, const void *pBuf,
    size_t buf_size, const void *pComment, mz_uint16 comment_size,
    mz_uint level_and_flags);
MINIZ_EXPORT mz_bool mz_zip_add_mem_to_archive_file_in_place_v2(
    const char *pZip_filename, const char *pArchive_name, const void *pBuf,
    size_t buf_size, const void *pComment, mz_uint16 comment_size,
    mz_uint level_and_flags, mz_zip_error *pErr);

/* 从存档中读取单个文件到堆块。 */
/* 如果pComment不为NULL，则只会提取具有指定注释的文件。 */
/* 失败时返回NULL。 */
MINIZ_EXPORT void *
mz_zip_extract_archive_file_to_heap(const char *pZip_filename,
                                    const char *pArchive_name, size_t *pSize,
                                    mz_uint flags);
MINIZ_EXPORT void *mz_zip_extract_archive_file_to_heap_v2(
    const char *pZip_filename, const char *pArchive_name, const char *pComment,
    size_t *pSize, mz_uint flags, mz_zip_error *pErr);

#endif /* #ifndef MINIZ_NO_ARCHIVE_WRITING_APIS */

#ifdef __cplusplus
}
#endif

#endif /* MINIZ_NO_ARCHIVE_APIS */
// 定义用于验证 mz_uint16 数据类型大小的数组
typedef unsigned char mz_validate_uint16[sizeof(mz_uint16) == 2 ? 1 : -1];
// 定义用于验证 mz_uint32 数据类型大小的数组
typedef unsigned char mz_validate_uint32[sizeof(mz_uint32) == 4 ? 1 : -1];
// 定义用于验证 mz_uint64 数据类型大小的数组
typedef unsigned char mz_validate_uint64[sizeof(mz_uint64) == 8 ? 1 : -1];

#ifdef __cplusplus
extern "C" {
#endif

/* ------------------- zlib-style API's */

// 计算 Adler-32 校验和
mz_ulong mz_adler32(mz_ulong adler, const unsigned char *ptr, size_t buf_len) {
  // 初始化 Adler-32 校验和的两个部分
  mz_uint32 i, s1 = (mz_uint32)(adler & 0xffff), s2 = (mz_uint32)(adler >> 16);
  // 计算最后一个块的长度
  size_t block_len = buf_len % 5552;
  // 如果指针为空，则返回初始的 Adler-32 校验和值
  if (!ptr)
    return MZ_ADLER32_INIT;
  // 循环计算 Adler-32 校验和
  while (buf_len) {
    # 遍历数据块中的每8个字节，计算校验和
    for (i = 0; i + 7 < block_len; i += 8, ptr += 8) {
      s1 += ptr[0], s2 += s1;  # 更新校验和 s1 和 s2
      s1 += ptr[1], s2 += s1;  # 更新校验和 s1 和 s2
      s1 += ptr[2], s2 += s1;  # 更新校验和 s1 和 s2
      s1 += ptr[3], s2 += s1;  # 更新校验和 s1 和 s2
      s1 += ptr[4], s2 += s1;  # 更新校验和 s1 和 s2
      s1 += ptr[5], s2 += s1;  # 更新校验和 s1 和 s2
      s1 += ptr[6], s2 += s1;  # 更新校验和 s1 和 s2
      s1 += ptr[7], s2 += s1;  # 更新校验和 s1 和 s2
    }
    # 处理剩余不足8个字节的数据
    for (; i < block_len; ++i)
      s1 += *ptr++, s2 += s1;  # 更新校验和 s1 和 s2
    # 对校验和取模
    s1 %= 65521U, s2 %= 65521U;
    # 减去当前数据块长度，准备处理下一个数据块
    buf_len -= block_len;
    # 重置数据块长度
    block_len = 5552;
  }
  # 返回最终的校验和结果
  return (s2 << 16) + s1;
/* Karl Malbrain's compact CRC-32. See "A compact CCITT crc16 and crc32 C
 * implementation that balances processor cache usage against speed":
 * http://www.geocities.com/malbrain/ */
#if 0
    mz_ulong mz_crc32(mz_ulong crc, const mz_uint8 *ptr, size_t buf_len)
    {
        static const mz_uint32 s_crc32[16] = { 0, 0x1db71064, 0x3b6e20c8, 0x26d930ac, 0x76dc4190, 0x6b6b51f4, 0x4db26158, 0x5005713c,
                                               0xedb88320, 0xf00f9344, 0xd6d6a3e8, 0xcb61b38c, 0x9b64c2b0, 0x86d3d2d4, 0xa00ae278, 0xbdbdf21c };
        mz_uint32 crcu32 = (mz_uint32)crc;
        if (!ptr)
            return MZ_CRC32_INIT;
        crcu32 = ~crcu32;
        while (buf_len--)
        {
            mz_uint8 b = *ptr++;
            crcu32 = (crcu32 >> 4) ^ s_crc32[(crcu32 & 0xF) ^ (b & 0xF)];
            crcu32 = (crcu32 >> 4) ^ s_crc32[(crcu32 & 0xF) ^ (b >> 4)];
        }
        return ~crcu32;
    }
#elif defined(USE_EXTERNAL_MZCRC)
/* If USE_EXTERNAL_CRC is defined, an external module will export the
 * mz_crc32() symbol for us to use, e.g. an SSE-accelerated version.
 * Depending on the impl, it may be necessary to ~ the input/output crc values.
 */
mz_ulong mz_crc32(mz_ulong crc, const mz_uint8 *ptr, size_t buf_len);
#else
/* Faster, but larger CPU cache footprint.
 */
    crc32 = (crc32 >> 8) ^ s_crc_table[(crc32 ^ pByte_buf[0]) & 0xFF];
    crc32 = (crc32 >> 8) ^ s_crc_table[(crc32 ^ pByte_buf[1]) & 0xFF];
    crc32 = (crc32 >> 8) ^ s_crc_table[(crc32 ^ pByte_buf[2]) & 0xFF];
    crc32 = (crc32 >> 8) ^ s_crc_table[(crc32 ^ pByte_buf[3]) & 0xFF];
    pByte_buf += 4;
    buf_len -= 4;
  }

  while (buf_len) {
    crc32 = (crc32 >> 8) ^ s_crc_table[(crc32 ^ pByte_buf[0]) & 0xFF];
    ++pByte_buf;
    --buf_len;
  }

  return ~crc32;
}
#endif

void mz_free(void *p) { MZ_FREE(p); }
// 定义 miniz_def_alloc_func 函数，用于分配内存
MINIZ_EXPORT void *miniz_def_alloc_func(void *opaque, size_t items,
                                        size_t size) {
  // 忽略参数，直接返回分配的内存
  (void)opaque, (void)items, (void)size;
  return MZ_MALLOC(items * size);
}

// 定义 miniz_def_free_func 函数，用于释放内存
MINIZ_EXPORT void miniz_def_free_func(void *opaque, void *address) {
  // 忽略参数，直接释放内存
  (void)opaque, (void)address;
  MZ_FREE(address);
}

// 定义 miniz_def_realloc_func 函数，用于重新分配内存
MINIZ_EXPORT void *miniz_def_realloc_func(void *opaque, void *address,
                                          size_t items, size_t size) {
  // 忽略参数，直接返回重新分配的内存
  (void)opaque, (void)address, (void)items, (void)size;
  return MZ_REALLOC(address, items * size);
}

// 返回 miniz 版本信息
const char *mz_version(void) { return MZ_VERSION; }

// 如果没有定义 MINIZ_NO_ZLIB_APIS，则定义以下函数

// 初始化压缩流
int mz_deflateInit(mz_streamp pStream, int level) {
  return mz_deflateInit2(pStream, level, MZ_DEFLATED, MZ_DEFAULT_WINDOW_BITS, 9,
                         MZ_DEFAULT_STRATEGY);
}

// 初始化压缩流，带有更多参数
int mz_deflateInit2(mz_streamp pStream, int level, int method, int window_bits,
                    int mem_level, int strategy) {
  tdefl_compressor *pComp;
  // 计算压缩标志
  mz_uint comp_flags =
      TDEFL_COMPUTE_ADLER32 |
      tdefl_create_comp_flags_from_zip_params(level, window_bits, strategy);

  // 检查参数是否合法
  if (!pStream)
    return MZ_STREAM_ERROR;
  if ((method != MZ_DEFLATED) || ((mem_level < 1) || (mem_level > 9)) ||
      ((window_bits != MZ_DEFAULT_WINDOW_BITS) &&
       (-window_bits != MZ_DEFAULT_WINDOW_BITS)))
    return MZ_PARAM_ERROR;

  // 初始化压缩流的各个属性
  pStream->data_type = 0;
  pStream->adler = MZ_ADLER32_INIT;
  pStream->msg = NULL;
  pStream->reserved = 0;
  pStream->total_in = 0;
  pStream->total_out = 0;
  if (!pStream->zalloc)
    pStream->zalloc = miniz_def_alloc_func;
  if (!pStream->zfree)
    pStream->zfree = miniz_def_free_func;

  // 分配压缩器内存
  pComp = (tdefl_compressor *)pStream->zalloc(pStream->opaque, 1,
                                              sizeof(tdefl_compressor));
  if (!pComp)
    return MZ_MEM_ERROR;

  // 设置压缩流的状态
  pStream->state = (struct mz_internal_state *)pComp;

  // 初始化压缩器
  if (tdefl_init(pComp, NULL, NULL, comp_flags) != TDEFL_STATUS_OKAY) {
    mz_deflateEnd(pStream);
    # 如果参数错误，返回错误码 MZ_PARAM_ERROR
    return MZ_PARAM_ERROR;
  }

  # 如果参数正确，返回成功码 MZ_OK
  return MZ_OK;
// 重置压缩流状态，将压缩流的输入和输出总字节数归零，并重新初始化压缩器
int mz_deflateReset(mz_streamp pStream) {
  // 检查参数是否有效
  if ((!pStream) || (!pStream->state) || (!pStream->zalloc) ||
      (!pStream->zfree))
    return MZ_STREAM_ERROR;
  // 将压缩流的输入和输出总字节数归零
  pStream->total_in = pStream->total_out = 0;
  // 初始化压缩器
  tdefl_init((tdefl_compressor *)pStream->state, NULL, NULL,
             ((tdefl_compressor *)pStream->state)->m_flags);
  return MZ_OK;
}

// 执行压缩操作
int mz_deflate(mz_streamp pStream, int flush) {
  size_t in_bytes, out_bytes;
  mz_ulong orig_total_in, orig_total_out;
  int mz_status = MZ_OK;

  // 检查参数是否有效
  if ((!pStream) || (!pStream->state) || (flush < 0) || (flush > MZ_FINISH) ||
      (!pStream->next_out))
    return MZ_STREAM_ERROR;
  // 检查输出缓冲区是否有空间
  if (!pStream->avail_out)
    return MZ_BUF_ERROR;

  // 将部分刷新的标志转换为同步刷新的标志
  if (flush == MZ_PARTIAL_FLUSH)
    flush = MZ_SYNC_FLUSH;

  // 如果上一次压缩操作已经完成，则根据 flush 参数返回相应状态
  if (((tdefl_compressor *)pStream->state)->m_prev_return_status ==
      TDEFL_STATUS_DONE)
    return (flush == MZ_FINISH) ? MZ_STREAM_END : MZ_BUF_ERROR;

  // 保存原始的输入和输出总字节数
  orig_total_in = pStream->total_in;
  orig_total_out = pStream->total_out;
  // 循环执行压缩操作
  for (;;) {
    tdefl_status defl_status;
    in_bytes = pStream->avail_in;
    out_bytes = pStream->avail_out;

    // 执行压缩操作
    defl_status = tdefl_compress((tdefl_compressor *)pStream->state,
                                 pStream->next_in, &in_bytes, pStream->next_out,
                                 &out_bytes, (tdefl_flush)flush);
    pStream->next_in += (mz_uint)in_bytes;
    pStream->avail_in -= (mz_uint)in_bytes;
    pStream->total_in += (mz_uint)in_bytes;
    pStream->adler = tdefl_get_adler32((tdefl_compressor *)pStream->state);

    pStream->next_out += (mz_uint)out_bytes;
    pStream->avail_out -= (mz_uint)out_bytes;
    pStream->total_out += (mz_uint)out_bytes;

    // 检查压缩状态
    if (defl_status < 0) {
      mz_status = MZ_STREAM_ERROR;
      break;
    } else if (defl_status == TDEFL_STATUS_DONE) {
      mz_status = MZ_STREAM_END;
      break;
    } else if (!pStream->avail_out)
      break;
    else if ((!pStream->avail_in) && (flush != MZ_FINISH)) {
      // 如果输入流中没有可用数据，并且不是最后一次压缩操作
      if ((flush) || (pStream->total_in != orig_total_in) ||
          (pStream->total_out != orig_total_out))
        // 如果需要刷新，或者输入输出数据量有变化，则跳出循环
        break;
      // 返回缓冲区错误，需要输入数据才能继续
      return MZ_BUF_ERROR; /* Can't make forward progress without some input.
                            */
    }
  }
  // 返回压缩状态
  return mz_status;
}

// 结束压缩流的函数
int mz_deflateEnd(mz_streamp pStream) {
  // 如果压缩流为空，则返回流错误
  if (!pStream)
    return MZ_STREAM_ERROR;
  // 如果压缩流的状态存在
  if (pStream->state) {
    // 释放状态所占内存
    pStream->zfree(pStream->opaque, pStream->state);
    pStream->state = NULL;
  }
  return MZ_OK;
}

// 计算压缩后数据的上限
mz_ulong mz_deflateBound(mz_streamp pStream, mz_ulong source_len) {
  (void)pStream;
  /* 这实际上是非常保守的。(而且很糟糕，但实际上很难计算一个真正的上限，考虑到 tdefl 的阻塞方式。) */
  return MZ_MAX(128 + (source_len * 110) / 100,
                128 + source_len + ((source_len / (31 * 1024)) + 1) * 5);
}

// 压缩数据的函数
int mz_compress2(unsigned char *pDest, mz_ulong *pDest_len,
                 const unsigned char *pSource, mz_ulong source_len, int level) {
  int status;
  mz_stream stream;
  memset(&stream, 0, sizeof(stream));

  /* 如果 mz_ulong 是 64 位（啊，我讨厌 long）。 */
  if ((source_len | *pDest_len) > 0xFFFFFFFFU)
    return MZ_PARAM_ERROR;

  stream.next_in = pSource;
  stream.avail_in = (mz_uint32)source_len;
  stream.next_out = pDest;
  stream.avail_out = (mz_uint32)*pDest_len;

  status = mz_deflateInit(&stream, level);
  if (status != MZ_OK)
    return status;

  status = mz_deflate(&stream, MZ_FINISH);
  if (status != MZ_STREAM_END) {
    mz_deflateEnd(&stream);
    return (status == MZ_OK) ? MZ_BUF_ERROR : status;
  }

  *pDest_len = stream.total_out;
  return mz_deflateEnd(&stream);
}

// 压缩数据的函数，默认压缩级别
int mz_compress(unsigned char *pDest, mz_ulong *pDest_len,
                const unsigned char *pSource, mz_ulong source_len) {
  return mz_compress2(pDest, pDest_len, pSource, source_len,
                      MZ_DEFAULT_COMPRESSION);
}

// 计算压缩后数据的上限
mz_ulong mz_compressBound(mz_ulong source_len) {
  return mz_deflateBound(NULL, source_len);
}

// 解压缩状态结构体
typedef struct {
  tinfl_decompressor m_decomp;
  mz_uint m_dict_ofs, m_dict_avail, m_first_call, m_has_flushed;
  int m_window_bits;
  mz_uint8 m_dict[TINFL_LZ_DICT_SIZE];
  tinfl_status m_last_status;
} inflate_state;
# 初始化解压缩流，设置解压缩状态和参数
int mz_inflateInit2(mz_streamp pStream, int window_bits) {
  # 创建解压缩状态对象
  inflate_state *pDecomp;
  # 如果解压缩流为空，返回流错误
  if (!pStream)
    return MZ_STREAM_ERROR;
  # 如果窗口位数不是默认值，并且负窗口位数也不是默认值，返回参数错误
  if ((window_bits != MZ_DEFAULT_WINDOW_BITS) &&
      (-window_bits != MZ_DEFAULT_WINDOW_BITS))
    return MZ_PARAM_ERROR;

  # 初始化解压缩流的各个属性
  pStream->data_type = 0;
  pStream->adler = 0;
  pStream->msg = NULL;
  pStream->total_in = 0;
  pStream->total_out = 0;
  pStream->reserved = 0;
  # 如果分配器为空，设置为默认分配函数
  if (!pStream->zalloc)
    pStream->zalloc = miniz_def_alloc_func;
  # 如果释放器为空，设置为默认释放函数
  if (!pStream->zfree)
    pStream->zfree = miniz_def_free_func;

  # 使用分配器分配内存给解压缩状态对象
  pDecomp = (inflate_state *)pStream->zalloc(pStream->opaque, 1,
                                             sizeof(inflate_state));
  # 如果分配失败，返回内存错误
  if (!pDecomp)
    return MZ_MEM_ERROR;

  # 将解压缩状态对象设置为解压缩流的状态
  pStream->state = (struct mz_internal_state *)pDecomp;

  # 初始化 tinfl 解压缩器
  tinfl_init(&pDecomp->m_decomp);
  pDecomp->m_dict_ofs = 0;
  pDecomp->m_dict_avail = 0;
  pDecomp->m_last_status = TINFL_STATUS_NEEDS_MORE_INPUT;
  pDecomp->m_first_call = 1;
  pDecomp->m_has_flushed = 0;
  pDecomp->m_window_bits = window_bits;

  # 返回成功状态
  return MZ_OK;
}

# 初始化解压缩流，使用默认窗口位数
int mz_inflateInit(mz_streamp pStream) {
  return mz_inflateInit2(pStream, MZ_DEFAULT_WINDOW_BITS);
}

# 重置解压缩流
int mz_inflateReset(mz_streamp pStream) {
  # 创建解压缩状态对象
  inflate_state *pDecomp;
  # 如果解压缩流为空，返回流错误
  if (!pStream)
    return MZ_STREAM_ERROR;

  # 初始化解压缩流的各个属性
  pStream->data_type = 0;
  pStream->adler = 0;
  pStream->msg = NULL;
  pStream->total_in = 0;
  pStream->total_out = 0;
  pStream->reserved = 0;

  # 获取解压缩状态对象
  pDecomp = (inflate_state *)pStream->state;

  # 初始化 tinfl 解压缩器
  tinfl_init(&pDecomp->m_decomp);
  pDecomp->m_dict_ofs = 0;
  pDecomp->m_dict_avail = 0;
  pDecomp->m_last_status = TINFL_STATUS_NEEDS_MORE_INPUT;
  pDecomp->m_first_call = 1;
  pDecomp->m_has_flushed = 0;
  /* pDecomp->m_window_bits = window_bits */;

  # 返回成功状态
  return MZ_OK;
}

# 解压缩数据
int mz_inflate(mz_streamp pStream, int flush) {
  # 创建解压缩状态对象
  inflate_state *pState;
  mz_uint n, first_call, decomp_flags = TINFL_FLAG_COMPUTE_ADLER32;
  size_t in_bytes, out_bytes, orig_avail_in;
  tinfl_status status;

  # 如果解压缩流为空或者解压缩状态为空
    // 如果 flush 为 MZ_STREAM_ERROR，则返回错误代码
    return MZ_STREAM_ERROR;
  // 如果 flush 为 MZ_PARTIAL_FLUSH，则将其设置为 MZ_SYNC_FLUSH
  if (flush == MZ_PARTIAL_FLUSH)
    flush = MZ_SYNC_FLUSH;
  // 如果 flush 不为 MZ_SYNC_FLUSH 且不为 MZ_FINISH，则返回错误代码
  if ((flush) && (flush != MZ_SYNC_FLUSH) && (flush != MZ_FINISH))
    return MZ_STREAM_ERROR;

  // 获取解压缩状态
  pState = (inflate_state *)pStream->state;
  // 如果窗口位数大于 0，则设置解压缩标志
  if (pState->m_window_bits > 0)
    decomp_flags |= TINFL_FLAG_PARSE_ZLIB_HEADER;
  // 保存原始输入可用字节数
  orig_avail_in = pStream->avail_in;

  // 获取是否第一次调用标志
  first_call = pState->m_first_call;
  pState->m_first_call = 0;
  // 如果上次状态小于 0，则返回数据错误
  if (pState->m_last_status < 0)
    return MZ_DATA_ERROR;

  // 如果已经刷新过且不是 MZ_FINISH，则返回错误代码
  if (pState->m_has_flushed && (flush != MZ_FINISH))
    return MZ_STREAM_ERROR;
  // 设置已刷新标志
  pState->m_has_flushed |= (flush == MZ_FINISH);

  // 如果是 MZ_FINISH 且是第一次调用
  if ((flush == MZ_FINISH) && (first_call)) {
    /* MZ_FINISH on the first call implies that the input and output buffers are
     * large enough to hold the entire compressed/decompressed file. */
    // 设置解压缩标志
    decomp_flags |= TINFL_FLAG_USING_NON_WRAPPING_OUTPUT_BUF;
    // 获取输入和输出字节数
    in_bytes = pStream->avail_in;
    out_bytes = pStream->avail_out;
    // 解压缩数据
    status = tinfl_decompress(&pState->m_decomp, pStream->next_in, &in_bytes,
                              pStream->next_out, pStream->next_out, &out_bytes,
                              decomp_flags);
    // 更新状态和指针
    pState->m_last_status = status;
    pStream->next_in += (mz_uint)in_bytes;
    pStream->avail_in -= (mz_uint)in_bytes;
    pStream->total_in += (mz_uint)in_bytes;
    pStream->adler = tinfl_get_adler32(&pState->m_decomp);
    pStream->next_out += (mz_uint)out_bytes;
    pStream->avail_out -= (mz_uint)out_bytes;
    pStream->total_out += (mz_uint)out_bytes;

    // 如果状态小于 0，则返回数据错误
    if (status < 0)
      return MZ_DATA_ERROR;
    // 如果状态不是 TINFL_STATUS_DONE，则更新状态并返回缓冲区错误
    else if (status != TINFL_STATUS_DONE) {
      pState->m_last_status = TINFL_STATUS_FAILED;
      return MZ_BUF_ERROR;
    }
    // 返回流结束
    return MZ_STREAM_END;
  }
  // 如果 flush 不是 MZ_FINISH，则假设还有更多输入
  if (flush != MZ_FINISH)
    decomp_flags |= TINFL_FLAG_HAS_MORE_INPUT;

  // 如果字典可用，则获取最小值
  if (pState->m_dict_avail) {
    n = MZ_MIN(pState->m_dict_avail, pStream->avail_out);
    // 将 pState->m_dict 中偏移量为 pState->m_dict_ofs 的 n 个字节复制到 pStream->next_out 中
    memcpy(pStream->next_out, pState->m_dict + pState->m_dict_ofs, n);
    // 更新 pStream->next_out 指针位置
    pStream->next_out += n;
    // 减少 pStream->avail_out 可用空间
    pStream->avail_out -= n;
    // 增加 pStream->total_out 输出总量
    pStream->total_out += n;
    // 减少 pState->m_dict_avail 字典可用空间
    pState->m_dict_avail -= n;
    // 更新 pState->m_dict_ofs 字典偏移量
    pState->m_dict_ofs = (pState->m_dict_ofs + n) & (TINFL_LZ_DICT_SIZE - 1);
    // 返回解压状态，如果解压完成且字典可用空间为 0，则返回 MZ_STREAM_END，否则返回 MZ_OK
    return ((pState->m_last_status == TINFL_STATUS_DONE) &&
            (!pState->m_dict_avail))
               ? MZ_STREAM_END
               : MZ_OK;
  }

  // 无限循环
  for (;;) {
    // 记录输入字节数
    in_bytes = pStream->avail_in;
    // 计算输出字节数
    out_bytes = TINFL_LZ_DICT_SIZE - pState->m_dict_ofs;

    // 解压数据
    status = tinfl_decompress(
        &pState->m_decomp, pStream->next_in, &in_bytes, pState->m_dict,
        pState->m_dict + pState->m_dict_ofs, &out_bytes, decomp_flags);
    // 记录最后的解压状态
    pState->m_last_status = status;

    // 更新输入指针和可用输入空间
    pStream->next_in += (mz_uint)in_bytes;
    pStream->avail_in -= (mz_uint)in_bytes;
    pStream->total_in += (mz_uint)in_bytes;
    // 计算并更新 Adler32 校验值
    pStream->adler = tinfl_get_adler32(&pState->m_decomp);

    // 记录字典可用空间
    pState->m_dict_avail = (mz_uint)out_bytes;

    // 计算需要复制的字节数
    n = MZ_MIN(pState->m_dict_avail, pStream->avail_out);
    // 将字典中的数据复制到输出流中
    memcpy(pStream->next_out, pState->m_dict + pState->m_dict_ofs, n);
    // 更新输出指针和可用输出空间
    pStream->next_out += n;
    pStream->avail_out -= n;
    pStream->total_out += n;
    // 更新字典可用空间和字典偏移量
    pState->m_dict_avail -= n;
    pState->m_dict_ofs = (pState->m_dict_ofs + n) & (TINFL_LZ_DICT_SIZE - 1);

    // 如果解压状态小于 0，则返回 MZ_DATA_ERROR，表示流已损坏
    if (status < 0)
      return MZ_DATA_ERROR; /* Stream is corrupted (there could be some
                               uncompressed data left in the output dictionary -
                               oh well). */
    // 如果解压状态为 TINFL_STATUS_NEEDS_MORE_INPUT 且原始输入为空，则返回 MZ_BUF_ERROR，表示需要更多输入数据或设置 flush 为 MZ_FINISH 才能继续
    else if ((status == TINFL_STATUS_NEEDS_MORE_INPUT) && (!orig_avail_in))
      return MZ_BUF_ERROR; /* Signal caller that we can't make forward progress
                              without supplying more input or by setting flush
                              to MZ_FINISH. */
    else if (flush == MZ_FINISH) {
      /* 当 flush 为 MZ_FINISH 时，输出缓冲区必须足够大以容纳剩余的未压缩数据 */
      if (status == TINFL_STATUS_DONE)
        return pState->m_dict_avail ? MZ_BUF_ERROR : MZ_STREAM_END;
      /* 此处 status 必须为 TINFL_STATUS_HAS_MORE_OUTPUT，表示至少还有一个字节待输出。
       * 如果输出缓冲区没有剩余空间，则表示出现错误。 */
      else if (!pStream->avail_out)
        return MZ_BUF_ERROR;
    } else if ((status == TINFL_STATUS_DONE) || (!pStream->avail_in) ||
               (!pStream->avail_out) || (pState->m_dict_avail))
      break;
  }

  return ((status == TINFL_STATUS_DONE) && (!pState->m_dict_avail))
             ? MZ_STREAM_END
             : MZ_OK;
}

// 结束解压缩操作，释放资源
int mz_inflateEnd(mz_streamp pStream) {
  // 如果传入的流为空，返回流错误
  if (!pStream)
    return MZ_STREAM_ERROR;
  // 如果流状态存在
  if (pStream->state) {
    // 释放流状态
    pStream->zfree(pStream->opaque, pStream->state);
    pStream->state = NULL;
  }
  return MZ_OK;
}

// 解压缩数据
int mz_uncompress2(unsigned char *pDest, mz_ulong *pDest_len,
                   const unsigned char *pSource, mz_ulong *pSource_len) {
  // 创建解压缩流
  mz_stream stream;
  int status;
  // 初始化流结构体
  memset(&stream, 0, sizeof(stream));

  /* In case mz_ulong is 64-bits (argh I hate longs). */
  // 如果 mz_ulong 是 64 位（啊，我讨厌 long 类型）
  if ((*pSource_len | *pDest_len) > 0xFFFFFFFFU)
    return MZ_PARAM_ERROR;

  // 设置输入数据和输出数据
  stream.next_in = pSource;
  stream.avail_in = (mz_uint32)*pSource_len;
  stream.next_out = pDest;
  stream.avail_out = (mz_uint32)*pDest_len;

  // 初始化解压缩流
  status = mz_inflateInit(&stream);
  if (status != MZ_OK)
    return status;

  // 执行解压缩操作
  status = mz_inflate(&stream, MZ_FINISH);
  *pSource_len = *pSource_len - stream.avail_in;
  // 如果解压缩操作未完成
  if (status != MZ_STREAM_END) {
    // 结束解压缩操作，返回数据错误或流错误
    mz_inflateEnd(&stream);
    return ((status == MZ_BUF_ERROR) && (!stream.avail_in)) ? MZ_DATA_ERROR
                                                            : status;
  }
  // 设置输出数据长度
  *pDest_len = stream.total_out;

  // 结束解压缩操作，释放资源
  return mz_inflateEnd(&stream);
}

// 解压缩数据
int mz_uncompress(unsigned char *pDest, mz_ulong *pDest_len,
                  const unsigned char *pSource, mz_ulong source_len) {
  return mz_uncompress2(pDest, pDest_len, pSource, &source_len);
}

// 返回错误信息
const char *mz_error(int err) {
  static struct {
    int m_err;
    // 定义一个包含错误码和对应描述的结构体数组
    const char *m_pDesc;
  } s_error_descs[] = {{MZ_OK, ""},
                       {MZ_STREAM_END, "stream end"},
                       {MZ_NEED_DICT, "need dictionary"},
                       {MZ_ERRNO, "file error"},
                       {MZ_STREAM_ERROR, "stream error"},
                       {MZ_DATA_ERROR, "data error"},
                       {MZ_MEM_ERROR, "out of memory"},
                       {MZ_BUF_ERROR, "buf error"},
                       {MZ_VERSION_ERROR, "version error"},
                       {MZ_PARAM_ERROR, "parameter error"}};
  // 定义一个无符号整数变量 i
  mz_uint i;
  // 遍历错误描述数组
  for (i = 0; i < sizeof(s_error_descs) / sizeof(s_error_descs[0]); ++i)
    // 如果当前错误码等于目标错误码 err，则返回对应描述
    if (s_error_descs[i].m_err == err)
      return s_error_descs[i].m_pDesc;
  // 如果未找到对应错误码的描述，则返回空指针
  return NULL;
#ifdef __cplusplus
}
#endif



#ifdef __cplusplus



#endif



/*
  This is free and unencumbered software released into the public domain.

  Anyone is free to copy, modify, publish, use, compile, sell, or
  distribute this software, either in source code form or as a compiled
  binary, for any purpose, commercial or non-commercial, and by any
  means.

  In jurisdictions that recognize copyright laws, the author or authors
  of this software dedicate any and all copyright interest in the
  software to the public domain. We make this dedication for the benefit
  of the public at large and to the detriment of our heirs and
  successors. We intend this dedication to be an overt act of
  relinquishment in perpetuity of all present and future rights to this
  software under copyright law.

  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
  EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
  MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.
  IN NO EVENT SHALL THE AUTHORS BE LIABLE FOR ANY CLAIM, DAMAGES OR
  OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
  ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
  OTHER DEALINGS IN THE SOFTWARE.

  For more information, please refer to <http://unlicense.org/>
*/
#ifdef __cplusplus
extern "C" {
#endif
// 如果是 C++ 环境，使用 extern "C" 语法
// 定义低级压缩（与所有解压缩 API 无关）的相关内容

// 为了更快的初始化和线程安全，故意将这些表设为静态
static const mz_uint16 s_tdefl_len_sym[256] = {
    // 长度符号表，用于表示长度编码
    257, 258, 259, 260, 261, 262, 263, 264, 265, 265, 266, 266, 267, 267, 268,
    268, 269, 269, 269, 269, 270, 270, 270, 270, 271, 271, 271, 271, 272, 272,
    272, 272, 273, 273, 273, 273, 273, 273, 273, 273, 274, 274, 274, 274, 274,
    274, 274, 274, 275, 275, 275, 275, 275, 275, 275, 275, 276, 276, 276, 276,
    # 定义一个长数组，包含大量重复的数字
    data = {276, 276, 276, 276, 277, 277, 277, 277, 277, 277, 277, 277, 277, 277, 277,
            277, 277, 277, 277, 277, 278, 278, 278, 278, 278, 278, 278, 278, 278, 278,
            278, 278, 278, 278, 278, 278, 279, 279, 279, 279, 279, 279, 279, 279, 279,
            279, 279, 279, 279, 279, 279, 279, 280, 280, 280, 280, 280, 280, 280, 280,
            280, 280, 280, 280, 280, 280, 280, 280, 281, 281, 281, 281, 281, 281, 281,
            281, 281, 281, 281, 281, 281, 281, 281, 281, 281, 281, 281, 281, 281, 281,
            281, 281, 281, 281, 281, 281, 281, 281, 282, 282, 282, 282, 282, 282, 282,
            282, 282, 282, 282, 282, 282, 282, 282, 282, 282, 282, 282, 282, 282, 282,
            282, 282, 282, 
// 定义长度编码的额外位数，256个元素
static const mz_uint8 s_tdefl_len_extra[256] = {
    0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 2, 2, 2, 2, 2, 2,
    2, 2, 2, 2, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3,
    3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4, 4,
    4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
    4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 
    # 定义一个包含重复数字的列表
    nums = [15, 15, 15, 15, 15, 15, 15, 15, 15, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16,
            16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16,
            16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16,
            16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16,
            16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16,
            16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16, 16
// 定义一个包含512个元素的静态常量数组，用于存储小距离的额外信息
static const mz_uint8 s_tdefl_small_dist_extra[512] = {
    0, 0, 0, 0, 1, 1, 1, 1, 2, 2, 2, 2, 2, 2, 2, 2, 3, 3, 3, 3, 3, 3, 3, 3, 3,
    3, 3, 3, 3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4,
    4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
    5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5,
    5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5, 5
    # 定义一个包含数字 27 和 28 的列表
    nums = [27, 27, 27, 27, 27, 27, 27, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28,
            28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28, 28,
            28, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29,
            29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29, 29]
// 定义一个包含128个元素的静态常量数组，用于存储大距离额外信息
static const mz_uint8 s_tdefl_large_dist_extra[128] = {
    0,  0,  8,  8,  9,  9,  9,  9,  10, 10, 10, 10, 10, 10, 10, 10, 11, 11, 11,
    11, 11, 11, 11, 11, 11, 11, 11, 11, 11, 11, 11, 11, 12, 12, 12, 12, 12, 12,
    12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12, 12,
    12, 12, 12, 12, 12, 12, 12, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13,
    13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13,
    13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13, 13,
    13, 13, 13, 13, 13, 13, 13, 13, 13, 13
/* 计算最小冗余度，用于构建哈夫曼编码树 */
static void tdefl_calculate_minimum_redundancy(tdefl_sym_freq *A, int n) {
  int root, leaf, next, avbl, used, dpth;
  // 如果没有符号，则直接返回
  if (n == 0)
    return;
  // 如果只有一个符号，则设置其码长为1并返回
  else if (n == 1) {
    A[0].m_key = 1;
    return;
  }
  // 计算符号频率之和
  A[0].m_key += A[1].m_key;
  root = 0;
  leaf = 2;
  // 构建哈夫曼编码树
  for (next = 1; next < n - 1; next++) {
    if (leaf >= n || A[root].m_key < A[leaf].m_key) {
      A[next].m_key = A[root].m_key;
      A[root++].m_key = (mz_uint16)next;
    } else
      A[next].m_key = A[leaf++].m_key;
    if (leaf >= n || (root < next && A[root].m_key < A[leaf].m_key)) {
      A[next].m_key = (mz_uint16)(A[next].m_key + A[root].m_key);
      A[root++].m_key = (mz_uint16)next;
    } else
      A[next].m_key = (mz_uint16)(A[next].m_key + A[leaf++].m_key);
  }
  // 计算码长
  A[n - 2].m_key = 0;
  for (next = n - 3; next >= 0; next--)
    A[next].m_key = A[A[next].m_key].m_key + 1;
  avbl = 1;
  used = dpth = 0;
  root = n - 2;
  next = n - 1;
  // 强制限制哈夫曼编码表的最大码长
  while (avbl > 0) {
    while (root >= 0 && (int)A[root].m_key == dpth) {
      used++;
      root--;
    }
    while (avbl > used) {
      A[next--].m_key = (mz_uint16)(dpth);
      avbl--;
    }
    avbl = 2 * used;
    dpth++;
    used = 0;
  }
}

/* 限制规范哈夫曼编码表的最大码长 */
enum { TDEFL_MAX_SUPPORTED_HUFF_CODESIZE = 32 };
static void tdefl_huffman_enforce_max_code_size(int *pNum_codes,
                                                int code_list_len,
                                                int max_code_size) {
  int i;
  mz_uint32 total = 0;
  // 如果码表长度小于等于1，则直接返回
  if (code_list_len <= 1)
    return;
  // 调整码长大于最大码长的码字数量
  for (i = max_code_size + 1; i <= TDEFL_MAX_SUPPORTED_HUFF_CODESIZE; i++)
    pNum_codes[max_code_size] += pNum_codes[i];
  // 计算总和
  for (i = max_code_size; i > 0; i--)
    total += (((mz_uint32)pNum_codes[i]) << (max_code_size - i));
  // 调整码长为最大码长的码字数量，使其满足哈夫曼编码的要求
  while (total != (1UL << max_code_size)) {
    pNum_codes[max_code_size]--;
    # 从最大编码大小开始递减循环
    for (i = max_code_size - 1; i > 0; i--)
      # 如果当前编码数量不为零
      if (pNum_codes[i]) {
        # 减少当前编码数量
        pNum_codes[i]--;
        # 增加下一个编码数量并加上2
        pNum_codes[i + 1] += 2;
        # 跳出循环
        break;
      }
    # 总数减一
    total--;
  }
static void tdefl_optimize_huffman_table(tdefl_compressor *d, int table_num,
                                         int table_len, int code_size_limit,
                                         int static_table) {
  // 定义变量 i, j, l, num_codes，用于优化哈夫曼表
  int i, j, l, num_codes[1 + TDEFL_MAX_SUPPORTED_HUFF_CODESIZE];
  // 定义数组 next_code，用于存储下一个哈夫曼编码
  mz_uint next_code[TDEFL_MAX_SUPPORTED_HUFF_CODESIZE + 1];
  // 清空 num_codes 数组
  MZ_CLEAR_OBJ(num_codes);
  // 如果是静态表
  if (static_table) {
    // 遍历表长度，统计每个编码长度出现的次数
    for (i = 0; i < table_len; i++)
      num_codes[d->m_huff_code_sizes[table_num][i]]++;
  } else {
    // 定义符号频率数组 syms0, syms1，以及指向符号频率数组的指针 pSyms
    tdefl_sym_freq syms0[TDEFL_MAX_HUFF_SYMBOLS], syms1[TDEFL_MAX_HUFF_SYMBOLS],
        *pSyms;
    // 记录使用的符号数量
    int num_used_syms = 0;
    // 获取哈夫曼计数表
    const mz_uint16 *pSym_count = &d->m_huff_count[table_num][0];
    // 遍历表长度，将非零计数的符号添加到符号数组中
    for (i = 0; i < table_len; i++)
      if (pSym_count[i]) {
        syms0[num_used_syms].m_key = (mz_uint16)pSym_count[i];
        syms0[num_used_syms++].m_sym_index = (mz_uint16)i;
      }

    // 对符号数组进行基数排序
    pSyms = tdefl_radix_sort_syms(num_used_syms, syms0, syms1);
    // 计算最小冗余
    tdefl_calculate_minimum_redundancy(pSyms, num_used_syms);

    // 统计每个编码长度出现的次数
    for (i = 0; i < num_used_syms; i++)
      num_codes[pSyms[i].m_key]++;

    // 强制限制编码长度
    tdefl_huffman_enforce_max_code_size(num_codes, num_used_syms,
                                        code_size_limit);

    // 清空哈夫曼编码长度和编码数组
    MZ_CLEAR_OBJ(d->m_huff_code_sizes[table_num]);
    MZ_CLEAR_OBJ(d->m_huff_codes[table_num]);
    // 根据编码长度分配编码
    for (i = 1, j = num_used_syms; i <= code_size_limit; i++)
      for (l = num_codes[i]; l > 0; l--)
        d->m_huff_code_sizes[table_num][pSyms[--j].m_sym_index] = (mz_uint8)(i);
  }

  // 初始化下一个编码
  next_code[1] = 0;
  // 计算下一个编码
  for (j = 0, i = 2; i <= code_size_limit; i++)
    next_code[i] = j = ((j + num_codes[i - 1]) << 1);

  // 为每个符号生成哈夫曼编码
  for (i = 0; i < table_len; i++) {
    mz_uint rev_code = 0, code, code_size;
    // 如果编码长度为 0，则跳过
    if ((code_size = d->m_huff_code_sizes[table_num][i]) == 0)
      continue;
    // 生成反转编码
    code = next_code[code_size]++;
    for (l = code_size; l > 0; l--, code >>= 1)
      rev_code = (rev_code << 1) | (code & 1);
    // 存储哈夫曼编码
    d->m_huff_codes[table_num][i] = (mz_uint16)rev_code;
  }
}
# 定义一个宏，用于将指定的位数的数据放入输出缓冲区
#define TDEFL_PUT_BITS(b, l)                                                   \
  do {                                                                         \
    # 将传入的位数和长度分别赋值给变量 bits 和 len
    mz_uint bits = b;                                                          \
    mz_uint len = l;                                                           \
    # 断言传入的数据 bits 不超过长度 len 所能表示的最大值
    MZ_ASSERT(bits <= ((1U << len) - 1U));                                     \
    # 将 bits 左移当前已经使用的位数长度，并与当前缓冲区的数据进行按位或操作
    d->m_bit_buffer |= (bits << d->m_bits_in);                                 \
    # 更新已经使用的位数长度
    d->m_bits_in += len;                                                       \
    # 当已经使用的位数长度超过等于8时，将缓冲区数据写入输出缓冲区
    while (d->m_bits_in >= 8) {                                                \
      if (d->m_pOutput_buf < d->m_pOutput_buf_end)                             \
        *d->m_pOutput_buf++ = (mz_uint8)(d->m_bit_buffer);                     \
      # 将缓冲区数据右移8位
      d->m_bit_buffer >>= 8;                                                   \
      # 更新已经使用的位数长度
      d->m_bits_in -= 8;                                                       \
    }                                                                          \
  }                                                                            \
  MZ_MACRO_END

# 定义一个宏，用于获取 RLE 编码的前一个代码的大小
#define TDEFL_RLE_PREV_CODE_SIZE()                                             \
  {                                                                            \
    # 如果重复计数不为零
    if (rle_repeat_count) {                                                    \
      # 如果重复计数小于3
      if (rle_repeat_count < 3) {                                              \
        # 更新前一个代码大小对应的霍夫曼计数
        d->m_huff_count[2][prev_code_size] =                                   \
            (mz_uint16)(d->m_huff_count[2][prev_code_size] +                   \
                        rle_repeat_count);                                     \
        # 将前一个代码大小重复添加到压缩代码大小数组中
        while (rle_repeat_count--)                                             \
          packed_code_sizes[num_packed_code_sizes++] = prev_code_size;         \
      } else {                                                                 \
        # 更新16号代码大小对应的霍夫曼计数
        d->m_huff_count[2][16] = (mz_uint16)(d->m_huff_count[2][16] + 1);      \
        # 将16号代码大小和重复次数添加到压缩代码大小数组中
        packed_code_sizes[num_packed_code_sizes++] = 16;                       \
        packed_code_sizes[num_packed_code_sizes++] =                           \
            (mz_uint8)(rle_repeat_count - 3);                                  \
      }                                                                        \
      # 重置重复计数为零
      rle_repeat_count = 0;                                                    \
    }                                                                          \
  }
# 定义一个宏，用于处理零值的 RLE 编码
#define TDEFL_RLE_ZERO_CODE_SIZE()                                             \
  {                                                                            \
    # 如果存在连续的零值
    if (rle_z_count) {                                                         \
      # 如果连续的零值小于3
      if (rle_z_count < 3) {                                                   \
        # 更新零值对应的霍夫曼编码计数
        d->m_huff_count[2][0] =                                                \
            (mz_uint16)(d->m_huff_count[2][0] + rle_z_count);                  \
        # 将连续的零值写入到压缩码大小数组中
        while (rle_z_count--)                                                  \
          packed_code_sizes[num_packed_code_sizes++] = 0;                      \
      } else if (rle_z_count <= 10) {                                          \
        # 更新对应霍夫曼编码计数
        d->m_huff_count[2][17] = (mz_uint16)(d->m_huff_count[2][17] + 1);      \
        # 将霍夫曼编码和长度写入到压缩码大小数组中
        packed_code_sizes[num_packed_code_sizes++] = 17;                       \
        packed_code_sizes[num_packed_code_sizes++] =                           \
            (mz_uint8)(rle_z_count - 3);                                       \
      } else {                                                                 \
        # 更新对应霍夫曼编码计数
        d->m_huff_count[2][18] = (mz_uint16)(d->m_huff_count[2][18] + 1);      \
        # 将霍夫曼编码和长度写入到压缩码大小数组中
        packed_code_sizes[num_packed_code_sizes++] = 18;                       \
        packed_code_sizes[num_packed_code_sizes++] =                           \
            (mz_uint8)(rle_z_count - 11);                                      \
      }                                                                        \
      # 重置连续零值计数
      rle_z_count = 0;                                                         \
    }                                                                          \
  }

# 静态数组，用于存储压缩码大小符号的重新排列
static mz_uint8 s_tdefl_packed_code_size_syms_swizzle[] = {
    16, 17, 18, 0, 8, 7, 9, 6, 10, 5, 11, 4, 12, 3, 13, 2, 14, 1, 15};
// 开始动态块的压缩，初始化变量
static void tdefl_start_dynamic_block(tdefl_compressor *d) {
  int num_lit_codes, num_dist_codes, num_bit_lengths;
  mz_uint i, total_code_sizes_to_pack, num_packed_code_sizes, rle_z_count,
      rle_repeat_count, packed_code_sizes_index;
  mz_uint8
      code_sizes_to_pack[TDEFL_MAX_HUFF_SYMBOLS_0 + TDEFL_MAX_HUFF_SYMBOLS_1],
      packed_code_sizes[TDEFL_MAX_HUFF_SYMBOLS_0 + TDEFL_MAX_HUFF_SYMBOLS_1],
      prev_code_size = 0xFF;

  // 初始化 huff_count 数组
  d->m_huff_count[0][256] = 1;

  // 优化哈夫曼表
  tdefl_optimize_huffman_table(d, 0, TDEFL_MAX_HUFF_SYMBOLS_0, 15, MZ_FALSE);
  tdefl_optimize_huffman_table(d, 1, TDEFL_MAX_HUFF_SYMBOLS_1, 15, MZ_FALSE);

  // 计算字面量和距离码的数量
  for (num_lit_codes = 286; num_lit_codes > 257; num_lit_codes--)
    if (d->m_huff_code_sizes[0][num_lit_codes - 1])
      break;
  for (num_dist_codes = 30; num_dist_codes > 1; num_dist_codes--)
    if (d->m_huff_code_sizes[1][num_dist_codes - 1])
      break;

  // 复制码长度到待打包数组
  memcpy(code_sizes_to_pack, &d->m_huff_code_sizes[0][0], num_lit_codes);
  memcpy(code_sizes_to_pack + num_lit_codes, &d->m_huff_code_sizes[1][0],
         num_dist_codes);
  total_code_sizes_to_pack = num_lit_codes + num_dist_codes;
  num_packed_code_sizes = 0;
  rle_z_count = 0;
  rle_repeat_count = 0;

  // 初始化 huff_count[2] 数组
  memset(&d->m_huff_count[2][0], 0,
         sizeof(d->m_huff_count[2][0]) * TDEFL_MAX_HUFF_SYMBOLS_2);
  // 遍历待打包的码长度数组
  for (i = 0; i < total_code_sizes_to_pack; i++) {
    mz_uint8 code_size = code_sizes_to_pack[i];
    // 处理零码长度
    if (!code_size) {
      TDEFL_RLE_PREV_CODE_SIZE();
      if (++rle_z_count == 138) {
        TDEFL_RLE_ZERO_CODE_SIZE();
      }
    } else {
      TDEFL_RLE_ZERO_CODE_SIZE();
      // 处理非零码长度
      if (code_size != prev_code_size) {
        TDEFL_RLE_PREV_CODE_SIZE();
        d->m_huff_count[2][code_size] =
            (mz_uint16)(d->m_huff_count[2][code_size] + 1);
        packed_code_sizes[num_packed_code_sizes++] = code_size;
      } else if (++rle_repeat_count == 6) {
        TDEFL_RLE_PREV_CODE_SIZE();
      }
    }
    prev_code_size = code_size;
  }
  if (rle_repeat_count) {
    // 如果前一个码大小不为0，则调用TDEFL_RLE_PREV_CODE_SIZE()函数
    TDEFL_RLE_PREV_CODE_SIZE();
  } else {
    // 否则调用TDEFL_RLE_ZERO_CODE_SIZE()函数
    TDEFL_RLE_ZERO_CODE_SIZE();
  }

  // 优化哈夫曼表
  tdefl_optimize_huffman_table(d, 2, TDEFL_MAX_HUFF_SYMBOLS_2, 7, MZ_FALSE);

  // 写入2位比特
  TDEFL_PUT_BITS(2, 2);

  // 写入长度码数减去257的值，占5位
  TDEFL_PUT_BITS(num_lit_codes - 257, 5);
  // 写入距离码数减去1的值，占5位
  TDEFL_PUT_BITS(num_dist_codes - 1, 5);

  // 循环找到第一个非零码大小
  for (num_bit_lengths = 18; num_bit_lengths >= 0; num_bit_lengths--)
    if (d->m_huff_code_sizes
            [2][s_tdefl_packed_code_size_syms_swizzle[num_bit_lengths]])
      break;
  // 确保码长度至少为4
  num_bit_lengths = MZ_MAX(4, (num_bit_lengths + 1));
  // 写入码长度减去4的值，占4位
  TDEFL_PUT_BITS(num_bit_lengths - 4, 4);
  // 循环写入每个码的大小
  for (i = 0; (int)i < num_bit_lengths; i++)
    TDEFL_PUT_BITS(
        d->m_huff_code_sizes[2][s_tdefl_packed_code_size_syms_swizzle[i]], 3);

  // 循环写入压缩码大小
  for (packed_code_sizes_index = 0;
       packed_code_sizes_index < num_packed_code_sizes;) {
    // 获取下一个压缩码
    mz_uint code = packed_code_sizes[packed_code_sizes_index++];
    // 断言压缩码小于最大哈夫曼符号数
    MZ_ASSERT(code < TDEFL_MAX_HUFF_SYMBOLS_2);
    // 写入哈夫曼码和码大小
    TDEFL_PUT_BITS(d->m_huff_codes[2][code], d->m_huff_code_sizes[2][code]);
    // 如果码大于等于16，则写入额外的比特
    if (code >= 16)
      TDEFL_PUT_BITS(packed_code_sizes[packed_code_sizes_index++],
                     "\02\03\07"[code - 16]);
  }
static void tdefl_start_static_block(tdefl_compressor *d) {
  // 初始化静态哈夫曼编码表
  mz_uint i;
  mz_uint8 *p = &d->m_huff_code_sizes[0][0];

  // 初始化长度为8的哈夫曼编码
  for (i = 0; i <= 143; ++i)
    *p++ = 8;
  // 初始化长度为9的哈夫曼编码
  for (; i <= 255; ++i)
    *p++ = 9;
  // 初始化长度为7的哈夫曼编码
  for (; i <= 279; ++i)
    *p++ = 7;
  // 初始化长度为8的哈夫曼编码
  for (; i <= 287; ++i)
    *p++ = 8;

  // 将第二个哈夫曼编码表的前32个元素设置为5
  memset(d->m_huff_code_sizes[1], 5, 32);

  // 优化哈夫曼编码表
  tdefl_optimize_huffman_table(d, 0, 288, 15, MZ_TRUE);
  tdefl_optimize_huffman_table(d, 1, 32, 15, MZ_TRUE);

  // 将1的二进制表示写入输出缓冲区
  TDEFL_PUT_BITS(1, 2);
}

// 定义一个包含16个元素的位掩码数组
static const mz_uint mz_bitmasks[17] = {
    0x0000, 0x0001, 0x0003, 0x0007, 0x000F, 0x001F, 0x003F, 0x007F, 0x00FF,
    0x01FF, 0x03FF, 0x07FF, 0x0FFF, 0x1FFF, 0x3FFF, 0x7FFF, 0xFFFF};

// 如果支持非对齐加载和存储，并且是小端序，并且有64位寄存器
static mz_bool tdefl_compress_lz_codes(tdefl_compressor *d) {
  mz_uint flags;
  mz_uint8 *pLZ_codes;
  mz_uint8 *pOutput_buf = d->m_pOutput_buf;
  mz_uint8 *pLZ_code_buf_end = d->m_pLZ_code_buf;
  mz_uint64 bit_buffer = d->m_bit_buffer;
  mz_uint bits_in = d->m_bits_in;

  // 定义一个快速写入位的宏
#define TDEFL_PUT_BITS_FAST(b, l)                                              \
  {                                                                            \
    bit_buffer |= (((mz_uint64)(b)) << bits_in);                               \
    bits_in += (l);                                                            \
  }

  // 初始化标志位为1
  flags = 1;
  // 遍历LZ编码缓冲区
  for (pLZ_codes = d->m_lz_code_buf; pLZ_codes < pLZ_code_buf_end;
       flags >>= 1) {
    // 如果标志位为1，则将下一个字节作为标志位
    if (flags == 1)
      flags = *pLZ_codes++ | 0x100;
    # 如果标志位中包含1
    if (flags & 1) {
      # 定义变量s0, s1, n0, n1, sym, num_extra_bits，并初始化match_len和match_dist
      mz_uint s0, s1, n0, n1, sym, num_extra_bits;
      mz_uint match_len = pLZ_codes[0],
              match_dist = *(const mz_uint16 *)(pLZ_codes + 1);
      pLZ_codes += 3;

      # 断言检查长度符号是否有效
      MZ_ASSERT(d->m_huff_code_sizes[0][s_tdefl_len_sym[match_len]]);
      # 将长度符号编码写入输出流
      TDEFL_PUT_BITS_FAST(d->m_huff_codes[0][s_tdefl_len_sym[match_len]],
                          d->m_huff_code_sizes[0][s_tdefl_len_sym[match_len]]);
      # 将匹配长度的额外位写入输出流
      TDEFL_PUT_BITS_FAST(match_len & mz_bitmasks[s_tdefl_len_extra[match_len]],
                          s_tdefl_len_extra[match_len]);

      # 这个序列用于强制MSVC使用cmov而不是jmp
      s0 = s_tdefl_small_dist_sym[match_dist & 511];
      n0 = s_tdefl_small_dist_extra[match_dist & 511];
      s1 = s_tdefl_large_dist_sym[match_dist >> 8];
      n1 = s_tdefl_large_dist_extra[match_dist >> 8];
      sym = (match_dist < 512) ? s0 : s1;
      num_extra_bits = (match_dist < 512) ? n0 : n1;

      # 断言检查距离符号是否有效
      MZ_ASSERT(d->m_huff_code_sizes[1][sym]);
      # 将距离符号编码写入输出流
      TDEFL_PUT_BITS_FAST(d->m_huff_codes[1][sym],
                          d->m_huff_code_sizes[1][sym]);
      # 将匹配距离的额外位写入输出流
      TDEFL_PUT_BITS_FAST(match_dist & mz_bitmasks[num_extra_bits],
                          num_extra_bits);
    } else {
      # 如果标志位中不包含1
      mz_uint lit = *pLZ_codes++;
      # 断言检查字面值符号是否有效
      MZ_ASSERT(d->m_huff_code_sizes[0][lit]);
      # 将字面值符号编码写入输出流
      TDEFL_PUT_BITS_FAST(d->m_huff_codes[0][lit],
                          d->m_huff_code_sizes[0][lit]);

      # 如果标志位中不包含2且pLZ_codes未到达缓冲区末尾
      if (((flags & 2) == 0) && (pLZ_codes < pLZ_code_buf_end)) {
        flags >>= 1;
        lit = *pLZ_codes++;
        # 断言检查字面值符号是否有效
        MZ_ASSERT(d->m_huff_code_sizes[0][lit]);
        # 将字面值符号编码写入输出流
        TDEFL_PUT_BITS_FAST(d->m_huff_codes[0][lit],
                            d->m_huff_code_sizes[0][lit]);

        # 如果标志位中不包含2且pLZ_codes未到达缓冲区末尾
        if (((flags & 2) == 0) && (pLZ_codes < pLZ_code_buf_end)) {
          flags >>= 1;
          lit = *pLZ_codes++;
          # 断言检查字面值符号是否有效
          MZ_ASSERT(d->m_huff_code_sizes[0][lit]);
          # 将字面值符号编码写入输出流
          TDEFL_PUT_BITS_FAST(d->m_huff_codes[0][lit],
                              d->m_huff_code_sizes[0][lit]);
        }
      }
    }
    # 如果输出缓冲区已经超出范围，则返回假
    if (pOutput_buf >= d->m_pOutput_buf_end)
      return MZ_FALSE;

    # 将 bit_buffer 的值写入 pOutput_buf，使用 64 位整数进行类型转换
    *(mz_uint64 *)pOutput_buf = bit_buffer;
    # 更新 pOutput_buf 指针，移动到下一个位置，根据 bits_in 的位数确定移动的字节数
    pOutput_buf += (bits_in >> 3);
    # 将 bit_buffer 右移 bits_in & ~7 位，保留低 3 位
    bit_buffer >>= (bits_in & ~7);
    # 更新 bits_in，保留低 3 位
    bits_in &= 7;
#ifdef TDEFL_PUT_BITS_FAST
// 如果定义了 TDEFL_PUT_BITS_FAST 宏，则执行以下代码块
#undef TDEFL_PUT_BITS_FAST
#endif

// 设置输出缓冲区、位数和位缓冲区
d->m_pOutput_buf = pOutput_buf;
d->m_bits_in = 0;
d->m_bit_buffer = 0;

// 循环处理位数
while (bits_in) {
  // 计算要处理的位数
  mz_uint32 n = MZ_MIN(bits_in, 16);
  // 将位缓冲区中的数据写入输出缓冲区
  TDEFL_PUT_BITS((mz_uint)bit_buffer & mz_bitmasks[n], n);
  // 更新位缓冲区
  bit_buffer >>= n;
  // 更新剩余位数
  bits_in -= n;
}

// 将特殊的 Huffman 编码写入输出缓冲区
TDEFL_PUT_BITS(d->m_huff_codes[0][256], d->m_huff_code_sizes[0][256]);

// 返回是否还有输出缓冲区剩余空间
return (d->m_pOutput_buf < d->m_pOutput_buf_end);
#else
// 如果未定义 TDEFL_PUT_BITS_FAST 宏，则执行以下代码块
static mz_bool tdefl_compress_lz_codes(tdefl_compressor *d) {
  mz_uint flags;
  mz_uint8 *pLZ_codes;

  // 初始化标志位
  flags = 1;
  // 遍历 LZ 编码缓冲区
  for (pLZ_codes = d->m_lz_code_buf; pLZ_codes < d->m_pLZ_code_buf;
       flags >>= 1) {
    // 检查标志位
    if (flags == 1)
      flags = *pLZ_codes++ | 0x100;
    // 处理标志位
    if (flags & 1) {
      mz_uint sym, num_extra_bits;
      mz_uint match_len = pLZ_codes[0],
              match_dist = (pLZ_codes[1] | (pLZ_codes[2] << 8));
      pLZ_codes += 3;

      // 断言长度符号的 Huffman 编码大小
      MZ_ASSERT(d->m_huff_code_sizes[0][s_tdefl_len_sym[match_len]]);
      // 写入长度符号的 Huffman 编码
      TDEFL_PUT_BITS(d->m_huff_codes[0][s_tdefl_len_sym[match_len]],
                     d->m_huff_code_sizes[0][s_tdefl_len_sym[match_len]]);
      // 写入长度符号的额外位数
      TDEFL_PUT_BITS(match_len & mz_bitmasks[s_tdefl_len_extra[match_len]],
                     s_tdefl_len_extra[match_len]);

      // 处理匹配距离
      if (match_dist < 512) {
        sym = s_tdefl_small_dist_sym[match_dist];
        num_extra_bits = s_tdefl_small_dist_extra[match_dist];
      } else {
        sym = s_tdefl_large_dist_sym[match_dist >> 8];
        num_extra_bits = s_tdefl_large_dist_extra[match_dist >> 8];
      }
      // 断言距离符号的 Huffman 编码大小
      MZ_ASSERT(d->m_huff_code_sizes[1][sym]);
      // 写入距离符号的 Huffman 编码
      TDEFL_PUT_BITS(d->m_huff_codes[1][sym], d->m_huff_code_sizes[1][sym]);
      // 写入匹配距禇的额外位数
      TDEFL_PUT_BITS(match_dist & mz_bitmasks[num_extra_bits], num_extra_bits);
    } else {
      // 处理字面值
      mz_uint lit = *pLZ_codes++;
      // 断言字面值的 Huffman 编码大小
      MZ_ASSERT(d->m_huff_code_sizes[0][lit]);
      // 写入字面值的 Huffman 编码
      TDEFL_PUT_BITS(d->m_huff_codes[0][lit], d->m_huff_code_sizes[0][lit]);
    }
  }

  // 将特殊的 Huffman 编码写入输出缓冲区
  TDEFL_PUT_BITS(d->m_huff_codes[0][256], d->m_huff_code_sizes[0][256]);

  // 返回是否还有输出缓冲区剩余空间
  return (d->m_pOutput_buf < d->m_pOutput_buf_end);
}
#ifdef MINIZ_USE_UNALIGNED_LOADS_AND_STORES && MINIZ_LITTLE_ENDIAN && MINIZ_HAS_64BIT_REGISTERS


// 如果定义了MINIZ_USE_UNALIGNED_LOADS_AND_STORES、MINIZ_LITTLE_ENDIAN和MINIZ_HAS_64BIT_REGISTERS，则执行以下代码块
static mz_bool tdefl_compress_block(tdefl_compressor *d, mz_bool static_block) {
  // 如果是静态块，则开始静态块的压缩
  if (static_block)
    tdefl_start_static_block(d);
  // 否则开始动态块的压缩
  else
    tdefl_start_dynamic_block(d);
  // 返回压缩LZ码的结果
  return tdefl_compress_lz_codes(d);
}

// 刷新块
static int tdefl_flush_block(tdefl_compressor *d, int flush) {
  mz_uint saved_bit_buf, saved_bits_in;
  mz_uint8 *pSaved_output_buf;
  mz_bool comp_block_succeeded = MZ_FALSE;
  int n, use_raw_block =
             ((d->m_flags & TDEFL_FORCE_ALL_RAW_BLOCKS) != 0) &&
             (d->m_lookahead_pos - d->m_lz_code_buf_dict_pos) <= d->m_dict_size;
  mz_uint8 *pOutput_buf_start =
      ((d->m_pPut_buf_func == NULL) &&
       ((*d->m_pOut_buf_size - d->m_out_buf_ofs) >= TDEFL_OUT_BUF_SIZE))
          ? ((mz_uint8 *)d->m_pOut_buf + d->m_out_buf_ofs)
          : d->m_output_buf;

  // 设置输出缓冲区的起始位置
  d->m_pOutput_buf = pOutput_buf_start;
  // 设置输出缓冲区的结束位置
  d->m_pOutput_buf_end = d->m_pOutput_buf + TDEFL_OUT_BUF_SIZE - 16;

  // 断言输出刷新剩余量为0
  MZ_ASSERT(!d->m_output_flush_remaining);
  // 输出刷新偏移量为0
  d->m_output_flush_ofs = 0;
  // 输出刷新剩余量为0
  d->m_output_flush_remaining = 0;

  // 将LZ标志右移d->m_num_flags_left位
  *d->m_pLZ_flags = (mz_uint8)(*d->m_pLZ_flags >> d->m_num_flags_left);
  // 如果m_num_flags_left为8，则减去m_pLZ_code_buf的值
  d->m_pLZ_code_buf -= (d->m_num_flags_left == 8);

  // 如果设置了TDEFL_WRITE_ZLIB_HEADER标志且块索引为0，则写入ZLIB头
  if ((d->m_flags & TDEFL_WRITE_ZLIB_HEADER) && (!d->m_block_index)) {
    TDEFL_PUT_BITS(0x78, 8);
    TDEFL_PUT_BITS(0x01, 8);
  }

  // 写入flush是否为TDEFL_FINISH的标志
  TDEFL_PUT_BITS(flush == TDEFL_FINISH, 1);

  // 保存输出缓冲区的位置、位缓冲区和位数
  pSaved_output_buf = d->m_pOutput_buf;
  saved_bit_buf = d->m_bit_buffer;
  saved_bits_in = d->m_bits_in;

  // 如果不使用原始块
    # 压缩块是否成功
    comp_block_succeeded =
        tdefl_compress_block(d, (d->m_flags & TDEFL_FORCE_ALL_STATIC_BLOCKS) ||
                                    (d->m_total_lz_bytes < 48));

  /* 如果块被扩展，忘记输出缓冲区的当前内容，并发送原始块 */
  if (((use_raw_block) ||
       ((d->m_total_lz_bytes) && ((d->m_pOutput_buf - pSaved_output_buf + 1U) >=
                                  d->m_total_lz_bytes))) &&
      ((d->m_lookahead_pos - d->m_lz_code_buf_dict_pos) <= d->m_dict_size)) {
    mz_uint i;
    # 重置输出缓冲区和位缓冲区
    d->m_pOutput_buf = pSaved_output_buf;
    d->m_bit_buffer = saved_bit_buf, d->m_bits_in = saved_bits_in;
    TDEFL_PUT_BITS(0, 2);
    if (d->m_bits_in) {
      TDEFL_PUT_BITS(0, 8 - d->m_bits_in);
    }
    for (i = 2; i; --i, d->m_total_lz_bytes ^= 0xFFFF) {
      TDEFL_PUT_BITS(d->m_total_lz_bytes & 0xFFFF, 16);
    }
    for (i = 0; i < d->m_total_lz_bytes; ++i) {
      TDEFL_PUT_BITS(
          d->m_dict[(d->m_lz_code_buf_dict_pos + i) & TDEFL_LZ_DICT_SIZE_MASK],
          8);
    }
  }
  /* 检查在使用动态代码时，压缩块是否不适合输出缓冲区的极端不可能（如果不是不可能）的情况 */
  else if (!comp_block_succeeded) {
    d->m_pOutput_buf = pSaved_output_buf;
    d->m_bit_buffer = saved_bit_buf, d->m_bits_in = saved_bits_in;
    # 重新压缩块
    tdefl_compress_block(d, MZ_TRUE);
  }

  if (flush) {
    if (flush == TDEFL_FINISH) {
      if (d->m_bits_in) {
        TDEFL_PUT_BITS(0, 8 - d->m_bits_in);
      }
      if (d->m_flags & TDEFL_WRITE_ZLIB_HEADER) {
        mz_uint i, a = d->m_adler32;
        for (i = 0; i < 4; i++) {
          TDEFL_PUT_BITS((a >> 24) & 0xFF, 8);
          a <<= 8;
        }
      }
    } else {
      mz_uint i, z = 0;
      TDEFL_PUT_BITS(0, 3);
      if (d->m_bits_in) {
        TDEFL_PUT_BITS(0, 8 - d->m_bits_in);
      }
      for (i = 2; i; --i, z ^= 0xFFFF) {
        TDEFL_PUT_BITS(z & 0xFFFF, 16);
      }
    }
  }

  // 确保输出缓冲区指针小于输出缓冲区末尾指针
  MZ_ASSERT(d->m_pOutput_buf < d->m_pOutput_buf_end);

  // 将 huff_count 数组清零
  memset(&d->m_huff_count[0][0], 0,
         sizeof(d->m_huff_count[0][0]) * TDEFL_MAX_HUFF_SYMBOLS_0);
  // 将 huff_count 数组清零
  memset(&d->m_huff_count[1][0], 0,
         sizeof(d->m_huff_count[1][0]) * TDEFL_MAX_HUFF_SYMBOLS_1);

  // 设置 LZ 编码缓冲区指针
  d->m_pLZ_code_buf = d->m_lz_code_buf + 1;
  // 设置 LZ 标志缓冲区指针
  d->m_pLZ_flags = d->m_lz_code_buf;
  // 设置剩余标志位数
  d->m_num_flags_left = 8;
  // 更新 LZ 编码缓冲区字典位置
  d->m_lz_code_buf_dict_pos += d->m_total_lz_bytes;
  // 重置总 LZ 字节数
  d->m_total_lz_bytes = 0;
  // 更新块索引
  d->m_block_index++;

  // 如果输出缓冲区有数据
  if ((n = (int)(d->m_pOutput_buf - pOutput_buf_start)) != 0) {
    // 如果有输出缓冲区写入函数
    if (d->m_pPut_buf_func) {
      // 更新输入缓冲区大小
      *d->m_pIn_buf_size = d->m_pSrc - (const mz_uint8 *)d->m_pIn_buf;
      // 调用输出缓冲区写入函数
      if (!(*d->m_pPut_buf_func)(d->m_output_buf, n, d->m_pPut_buf_user))
        return (d->m_prev_return_status = TDEFL_STATUS_PUT_BUF_FAILED);
    } else if (pOutput_buf_start == d->m_output_buf) {
      // 计算需要拷贝的字节数
      int bytes_to_copy = (int)MZ_MIN(
          (size_t)n, (size_t)(*d->m_pOut_buf_size - d->m_out_buf_ofs));
      // 拷贝数据到输出缓冲区
      memcpy((mz_uint8 *)d->m_pOut_buf + d->m_out_buf_ofs, d->m_output_buf,
             bytes_to_copy);
      // 更新输出缓冲区偏移量
      d->m_out_buf_ofs += bytes_to_copy;
      // 如果还有剩余数据未拷贝完
      if ((n -= bytes_to_copy) != 0) {
        // 更新输出刷新偏移量和剩余数据量
        d->m_output_flush_ofs = bytes_to_copy;
        d->m_output_flush_remaining = n;
      }
    } else {
      // 更新输出缓冲区偏移量
      d->m_out_buf_ofs += n;
    }
  }

  // 返回输出刷新剩余数据量
  return d->m_output_flush_remaining;
// 如果 MINIZ 使用未对齐的加载和存储
#ifdef MINIZ_USE_UNALIGNED_LOADS_AND_STORES
// 如果 MINIZ 使用未对齐的加载和存储，并且使用 memcpy 函数
#ifdef MINIZ_UNALIGNED_USE_MEMCPY
// 读取未对齐的字（16位）并返回
static mz_uint16 TDEFL_READ_UNALIGNED_WORD(const mz_uint8 *p) {
  mz_uint16 ret;
  memcpy(&ret, p, sizeof(mz_uint16));
  return ret;
}
// 读取未对齐的字（16位）并返回
static mz_uint16 TDEFL_READ_UNALIGNED_WORD2(const mz_uint16 *p) {
  mz_uint16 ret;
  memcpy(&ret, p, sizeof(mz_uint16));
  return ret;
}
// 如果不使用 memcpy 函数，则定义宏来读取未对齐的字（16位）
#else
#define TDEFL_READ_UNALIGNED_WORD(p) *(const mz_uint16 *)(p)
#define TDEFL_READ_UNALIGNED_WORD2(p) *(const mz_uint16 *)(p)
#endif
// 强制内联函数，用于查找匹配项
static MZ_FORCEINLINE void
tdefl_find_match(tdefl_compressor *d, mz_uint lookahead_pos, mz_uint max_dist,
                 mz_uint max_match_len, mz_uint *pMatch_dist,
                 mz_uint *pMatch_len) {
  mz_uint dist, pos = lookahead_pos & TDEFL_LZ_DICT_SIZE_MASK,
                match_len = *pMatch_len, probe_pos = pos, next_probe_pos,
                probe_len;
  const mz_uint num_probes_left = d->m_max_probes[match_len >= 32];
  const mz_uint16 *s = (const mz_uint16 *)(d->m_dict + pos), *p, *q;
  mz_uint16 c01 = TDEFL_READ_UNALIGNED_WORD(&d->m_dict[pos + match_len - 1]),
            s01 = TDEFL_READ_UNALIGNED_WORD2(s);
  // 断言最大匹配长度不超过 TDEFL_MAX_MATCH_LEN
  MZ_ASSERT(max_match_len <= TDEFL_MAX_MATCH_LEN);
  // 如果最大匹配长度小于等于当前匹配长度，则返回
  if (max_match_len <= match_len)
    return;
  for (;;) {
    for (;;) {
      // 如果剩余探测次数为 0，则返回
      if (--num_probes_left == 0)
        return;
      // 定义宏来探测匹配项
#define TDEFL_PROBE                                                            \
  next_probe_pos = d->m_next[probe_pos];                                       \
  if ((!next_probe_pos) ||                                                     \
      ((dist = (mz_uint16)(lookahead_pos - next_probe_pos)) > max_dist))       \
    return;                                                                    \
  probe_pos = next_probe_pos & TDEFL_LZ_DICT_SIZE_MASK;                        \
  if (TDEFL_READ_UNALIGNED_WORD(&d->m_dict[probe_pos + match_len - 1]) == c01) \
    break;
      // 进行探测匹配
      TDEFL_PROBE;
      TDEFL_PROBE;
      TDEFL_PROBE;
    }
    // 如果距离为 0，则跳出循环
    if (!dist)
      break;
    # 将 probe_pos 偏移量加到字典指针上，得到 q 指针，指向待比较的数据
    q = (const mz_uint16 *)(d->m_dict + probe_pos);
    # 如果 q 指针指向的数据不等于 s01，则继续下一轮循环
    if (TDEFL_READ_UNALIGNED_WORD2(q) != s01)
      continue;
    # 初始化 p 指针指向 s，设置 probe_len 为 32
    p = s;
    probe_len = 32;
    # 进入 do-while 循环
    do {
    } while (
        # 依次比较 p 和 q 指针指向的数据是否相等，同时 probe_len 递减，直到 probe_len 为 0 或者不相等
        (TDEFL_READ_UNALIGNED_WORD2(++p) == TDEFL_READ_UNALIGNED_WORD2(++q)) &&
        (TDEFL_READ_UNALIGNED_WORD2(++p) == TDEFL_READ_UNALIGNED_WORD2(++q)) &&
        (TDEFL_READ_UNALIGNED_WORD2(++p) == TDEFL_READ_UNALIGNED_WORD2(++q)) &&
        (TDEFL_READ_UNALIGNED_WORD2(++p) == TDEFL_READ_UNALIGNED_WORD2(++q)) &&
        (--probe_len > 0));
    # 如果 probe_len 为 0，则表示找到匹配，更新匹配距离和匹配长度，跳出循环
    if (!probe_len) {
      *pMatch_dist = dist;
      *pMatch_len = MZ_MIN(max_match_len, (mz_uint)TDEFL_MAX_MATCH_LEN);
      break;
    } else if ((probe_len = ((mz_uint)(p - s) * 2) +
                            (mz_uint)(*(const mz_uint8 *)p ==
                                      *(const mz_uint8 *)q)) > match_len) {
      # 如果 probe_len 大于 match_len，则更新匹配距离和匹配长度
      *pMatch_dist = dist;
      if ((*pMatch_len = match_len = MZ_MIN(max_match_len, probe_len)) ==
          max_match_len)
        break;
      # 更新 c01 为字典中 pos + match_len - 1 处的数据
      c01 = TDEFL_READ_UNALIGNED_WORD(&d->m_dict[pos + match_len - 1]);
    }
  }
#else
// 如果不满足条件，则定义一个内联函数，用于查找匹配项
static MZ_FORCEINLINE void
tdefl_find_match(tdefl_compressor *d, mz_uint lookahead_pos, mz_uint max_dist,
                 mz_uint max_match_len, mz_uint *pMatch_dist,
                 mz_uint *pMatch_len) {
  // 定义变量
  mz_uint dist, pos = lookahead_pos & TDEFL_LZ_DICT_SIZE_MASK,
                match_len = *pMatch_len, probe_pos = pos, next_probe_pos,
                probe_len;
  // 获取最大探测次数
  mz_uint num_probes_left = d->m_max_probes[match_len >= 32];
  // 获取字典指针
  const mz_uint8 *s = d->m_dict + pos, *p, *q;
  // 获取当前位置和前一个字符
  mz_uint8 c0 = d->m_dict[pos + match_len], c1 = d->m_dict[pos + match_len - 1];
  // 断言最大匹配长度小于等于最大匹配长度
  MZ_ASSERT(max_match_len <= TDEFL_MAX_MATCH_LEN);
  // 如果最大匹配长度小于等于当前匹配长度，则返回
  if (max_match_len <= match_len)
    return;
  // 无限循环
  for (;;) {
    // 内部循环
    for (;;) {
      // 如果探测次数用完，则返回
      if (--num_probes_left == 0)
        return;
      // 定义探测宏
#define TDEFL_PROBE                                                            \
  next_probe_pos = d->m_next[probe_pos];                                       \
  // 如果下一个探测位置为空或者距离超过最大距离，则返回
  if ((!next_probe_pos) ||                                                     \
      ((dist = (mz_uint16)(lookahead_pos - next_probe_pos)) > max_dist))       \
    return;                                                                    \
  // 更新探测位置
  probe_pos = next_probe_pos & TDEFL_LZ_DICT_SIZE_MASK;                        \
  // 如果匹配成功，则跳出循环
  if ((d->m_dict[probe_pos + match_len] == c0) &&                              \
      (d->m_dict[probe_pos + match_len - 1] == c1))                            \
    break;
      // 调用探测宏
      TDEFL_PROBE;
      TDEFL_PROBE;
      TDEFL_PROBE;
    }
    // 如果距离为0，则跳出循环
    if (!dist)
      break;
    // 比较匹配项
    p = s;
    q = d->m_dict + probe_pos;
    for (probe_len = 0; probe_len < max_match_len; probe_len++)
      if (*p++ != *q++)
        break;
    // 如果探测长度大于匹配长度，则更新匹配距离和匹配长度
    if (probe_len > match_len) {
      *pMatch_dist = dist;
      if ((*pMatch_len = match_len = probe_len) == max_match_len)
        return;
      c0 = d->m_dict[pos + match_len];
      c1 = d->m_dict[pos + match_len - 1];
    }
  }
}
#endif /* #if MINIZ_USE_UNALIGNED_LOADS_AND_STORES */

#if MINIZ_USE_UNALIGNED_LOADS_AND_STORES && MINIZ_LITTLE_ENDIAN
#ifdef MINIZ_UNALIGNED_USE_MEMCPY
// 如果定义了 MINIZ_UNALIGNED_USE_MEMCPY，则使用 memcpy 函数读取未对齐的字节流并转换为 mz_uint32 类型
static mz_uint32 TDEFL_READ_UNALIGNED_WORD32(const mz_uint8 *p) {
  mz_uint32 ret;
  memcpy(&ret, p, sizeof(mz_uint32));
  return ret;
}
#else
// 否则，使用宏定义 TDEFL_READ_UNALIGNED_WORD32 直接读取未对齐的字节流并转换为 mz_uint32 类型
#define TDEFL_READ_UNALIGNED_WORD32(p) *(const mz_uint32 *)(p)
#endif

// 快速压缩函数，采用 LZRW1 风格的匹配和解析循环，用于追求原始吞吐量而非压缩比
static mz_bool tdefl_compress_fast(tdefl_compressor *d) {
  /* Faster, minimally featured LZRW1-style match+parse loop with better
   * register utilization. Intended for applications where raw throughput is
   * valued more highly than ratio. */
  mz_uint lookahead_pos = d->m_lookahead_pos,
          lookahead_size = d->m_lookahead_size, dict_size = d->m_dict_size,
          total_lz_bytes = d->m_total_lz_bytes,
          num_flags_left = d->m_num_flags_left;
  mz_uint8 *pLZ_code_buf = d->m_pLZ_code_buf, *pLZ_flags = d->m_pLZ_flags;
  mz_uint cur_pos = lookahead_pos & TDEFL_LZ_DICT_SIZE_MASK;

  // 循环处理源数据，直到源数据全部处理完毕或者需要刷新并且还有待处理的数据
  while ((d->m_src_buf_left) || ((d->m_flush) && (lookahead_size))) {
    const mz_uint TDEFL_COMP_FAST_LOOKAHEAD_SIZE = 4096;
    mz_uint dst_pos =
        (lookahead_pos + lookahead_size) & TDEFL_LZ_DICT_SIZE_MASK;
    mz_uint num_bytes_to_process = (mz_uint)MZ_MIN(
        d->m_src_buf_left, TDEFL_COMP_FAST_LOOKAHEAD_SIZE - lookahead_size);
    d->m_src_buf_left -= num_bytes_to_process;
    lookahead_size += num_bytes_to_process;

    // 处理待压缩的数据
    while (num_bytes_to_process) {
      mz_uint32 n = MZ_MIN(TDEFL_LZ_DICT_SIZE - dst_pos, num_bytes_to_process);
      // 将数据复制到字典中
      memcpy(d->m_dict + dst_pos, d->m_pSrc, n);
      if (dst_pos < (TDEFL_MAX_MATCH_LEN - 1))
        memcpy(d->m_dict + TDEFL_LZ_DICT_SIZE + dst_pos, d->m_pSrc,
               MZ_MIN(n, (TDEFL_MAX_MATCH_LEN - 1) - dst_pos));
      d->m_pSrc += n;
      dst_pos = (dst_pos + n) & TDEFL_LZ_DICT_SIZE_MASK;
      num_bytes_to_process -= n;
    }

    // 更新字典大小
    dict_size = MZ_MIN(TDEFL_LZ_DICT_SIZE - lookahead_size, dict_size);
    // 如果不需要刷新并且还未达到快速压缩的预期大小，则跳出循环
    if ((!d->m_flush) && (lookahead_size < TDEFL_COMP_FAST_LOOKAHEAD_SIZE))
      break;
#ifdef MINIZ_UNALIGNED_USE_MEMCPY
          // 如果支持非对齐内存访问，使用 memcpy 将 cur_match_dist 复制到 pLZ_code_buf 中
          memcpy(&pLZ_code_buf[1], &cur_match_dist, sizeof(cur_match_dist));
#else
          // 否则，将 cur_match_dist 转换为 mz_uint16 类型后存入 pLZ_code_buf 中
          *(mz_uint16 *)(&pLZ_code_buf[1]) = (mz_uint16)cur_match_dist;
#endif
          // pLZ_code_buf 向后移动 3 个字节
          pLZ_code_buf += 3;
          // 更新 pLZ_flags 的值，将其右移 1 位后与 0x80 进行按位或操作
          *pLZ_flags = (mz_uint8)((*pLZ_flags >> 1) | 0x80);

          // 根据 cur_match_dist 获取 s0 和 s1 的值
          s0 = s_tdefl_small_dist_sym[cur_match_dist & 511];
          s1 = s_tdefl_large_dist_sym[cur_match_dist >> 8];
          // 更新 huff_count 数组中的值
          d->m_huff_count[1][(cur_match_dist < 512) ? s0 : s1]++;

          // 更新 huff_count 数组中的值
          d->m_huff_count[0][s_tdefl_len_sym[cur_match_len - TDEFL_MIN_MATCH_LEN]]++;
        }
      } else {
        // 将 first_trigram 存入 pLZ_code_buf 中
        *pLZ_code_buf++ = (mz_uint8)first_trigram;
        // 更新 pLZ_flags 的值，将其右移 1 位
        *pLZ_flags = (mz_uint8)(*pLZ_flags >> 1);
        // 更新 huff_count 数组中的值
        d->m_huff_count[0][(mz_uint8)first_trigram]++;
      }

      // 如果标志位剩余数量为 0，则重新设置标志位
      if (--num_flags_left == 0) {
        num_flags_left = 8;
        pLZ_flags = pLZ_code_buf++;
      }

      // 更新总字节数
      total_lz_bytes += cur_match_len;
      // 更新 lookahead_pos 和 lookahead_size
      lookahead_pos += cur_match_len;
      // 更新 dict_size
      dict_size = MZ_MIN(dict_size + cur_match_len, (mz_uint)TDEFL_LZ_DICT_SIZE);
      // 更新 cur_pos
      cur_pos = (cur_pos + cur_match_len) & TDEFL_LZ_DICT_SIZE_MASK;
      // 断言 lookahead_size 大于等于 cur_match_len
      MZ_ASSERT(lookahead_size >= cur_match_len);
      // 更新 lookahead_size
      lookahead_size -= cur_match_len;

      // 如果 pLZ_code_buf 超出范围，则执行以下操作
      if (pLZ_code_buf > &d->m_lz_code_buf[TDEFL_LZ_CODE_BUF_SIZE - 8]) {
        int n;
        // 更新 d 中的相关属性
        d->m_lookahead_pos = lookahead_pos;
        d->m_lookahead_size = lookahead_size;
        d->m_dict_size = dict_size;
        d->m_total_lz_bytes = total_lz_bytes;
        d->m_pLZ_code_buf = pLZ_code_buf;
        d->m_pLZ_flags = pLZ_flags;
        d->m_num_flags_left = num_flags_left;
        // 刷新块并返回结果
        if ((n = tdefl_flush_block(d, 0)) != 0)
          return (n < 0) ? MZ_FALSE : MZ_TRUE;
        // 更新 total_lz_bytes、pLZ_code_buf、pLZ_flags 和 num_flags_left
        total_lz_bytes = d->m_total_lz_bytes;
        pLZ_code_buf = d->m_pLZ_code_buf;
        pLZ_flags = d->m_pLZ_flags;
        num_flags_left = d->m_num_flags_left;
      }
    }
    while (lookahead_size) {
      // 从字典中获取当前位置的字面量
      mz_uint8 lit = d->m_dict[cur_pos];

      // 增加总的 LZ 字节计数
      total_lz_bytes++;
      // 将字面量添加到 LZ 编码缓冲区中
      *pLZ_code_buf++ = lit;
      // 将 LZ 标志右移一位
      *pLZ_flags = (mz_uint8)(*pLZ_flags >> 1);
      // 如果标志位剩余数量减少到0，则重新设置标志位
      if (--num_flags_left == 0) {
        num_flags_left = 8;
        pLZ_flags = pLZ_code_buf++;
      }

      // 更新哈夫曼编码统计
      d->m_huff_count[0][lit]++;

      // 更新前瞻位置
      lookahead_pos++;
      // 更新字典大小，不超过最大字典大小
      dict_size = MZ_MIN(dict_size + 1, (mz_uint)TDEFL_LZ_DICT_SIZE);
      // 更新当前位置，循环使用字典
      cur_pos = (cur_pos + 1) & TDEFL_LZ_DICT_SIZE_MASK;
      // 减少前瞻大小
      lookahead_size--;

      // 如果 LZ 编码缓冲区超过最大限制
      if (pLZ_code_buf > &d->m_lz_code_buf[TDEFL_LZ_CODE_BUF_SIZE - 8]) {
        int n;
        // 保存当前状态
        d->m_lookahead_pos = lookahead_pos;
        d->m_lookahead_size = lookahead_size;
        d->m_dict_size = dict_size;
        d->m_total_lz_bytes = total_lz_bytes;
        d->m_pLZ_code_buf = pLZ_code_buf;
        d->m_pLZ_flags = pLZ_flags;
        d->m_num_flags_left = num_flags_left;
        // 刷新块并返回结果
        if ((n = tdefl_flush_block(d, 0)) != 0)
          return (n < 0) ? MZ_FALSE : MZ_TRUE;
        // 恢复状态
        total_lz_bytes = d->m_total_lz_bytes;
        pLZ_code_buf = d->m_pLZ_code_buf;
        pLZ_flags = d->m_pLZ_flags;
        num_flags_left = d->m_num_flags_left;
      }
    }
  }

  // 更新状态并返回成功
  d->m_lookahead_pos = lookahead_pos;
  d->m_lookahead_size = lookahead_size;
  d->m_dict_size = dict_size;
  d->m_total_lz_bytes = total_lz_bytes;
  d->m_pLZ_code_buf = pLZ_code_buf;
  d->m_pLZ_flags = pLZ_flags;
  d->m_num_flags_left = num_flags_left;
  return MZ_TRUE;
  }
#endif /* MINIZ_USE_UNALIGNED_LOADS_AND_STORES && MINIZ_LITTLE_ENDIAN */

// 记录字面量，更新字节流和标志位
static MZ_FORCEINLINE void tdefl_record_literal(tdefl_compressor *d,
                                                mz_uint8 lit) {
  // 增加总字节计数
  d->m_total_lz_bytes++;
  // 将字面量写入 LZ 编码缓冲区
  *d->m_pLZ_code_buf++ = lit;
  // 更新 LZ 标志位
  *d->m_pLZ_flags = (mz_uint8)(*d->m_pLZ_flags >> 1);
  // 更新标志位剩余数量
  if (--d->m_num_flags_left == 0) {
    d->m_num_flags_left = 8;
    d->m_pLZ_flags = d->m_pLZ_code_buf++;
  }
  // 更新哈夫曼编码计数
  d->m_huff_count[0][lit]++;
}

// 记录匹配，更新字节流和标志位
static MZ_FORCEINLINE void
tdefl_record_match(tdefl_compressor *d, mz_uint match_len, mz_uint match_dist) {
  mz_uint32 s0, s1;

  // 断言匹配长度和距离的有效性
  MZ_ASSERT((match_len >= TDEFL_MIN_MATCH_LEN) && (match_dist >= 1) &&
            (match_dist <= TDEFL_LZ_DICT_SIZE));

  // 增加总字节计数
  d->m_total_lz_bytes += match_len;

  // 记录匹配长度
  d->m_pLZ_code_buf[0] = (mz_uint8)(match_len - TDEFL_MIN_MATCH_LEN);

  // 计算匹配距离
  match_dist -= 1;
  d->m_pLZ_code_buf[1] = (mz_uint8)(match_dist & 0xFF);
  d->m_pLZ_code_buf[2] = (mz_uint8)(match_dist >> 8);
  d->m_pLZ_code_buf += 3;

  // 更新 LZ 标志位
  *d->m_pLZ_flags = (mz_uint8)((*d->m_pLZ_flags >> 1) | 0x80);
  // 更新标志位剩余数量
  if (--d->m_num_flags_left == 0) {
    d->m_num_flags_left = 8;
    d->m_pLZ_flags = d->m_pLZ_code_buf++;
  }

  // 更新哈夫曼编码计数
  s0 = s_tdefl_small_dist_sym[match_dist & 511];
  s1 = s_tdefl_large_dist_sym[(match_dist >> 8) & 127];
  d->m_huff_count[1][(match_dist < 512) ? s0 : s1]++;
  d->m_huff_count[0][s_tdefl_len_sym[match_len - TDEFL_MIN_MATCH_LEN]]++;
}

// 压缩正常数据
static mz_bool tdefl_compress_normal(tdefl_compressor *d) {
  const mz_uint8 *pSrc = d->m_pSrc;
  size_t src_buf_left = d->m_src_buf_left;
  tdefl_flush flush = d->m_flush;

  // 处理源数据和刷新标志
  while ((src_buf_left) || ((flush) && (d->m_lookahead_size))) {
    mz_uint len_to_move, cur_match_dist, cur_match_len, cur_pos;
    /* Update dictionary and hash chains. Keeps the lookahead size equal to
     * TDEFL_MAX_MATCH_LEN. */
    // 检查是否需要处理当前字节
    if ((d->m_lookahead_size + d->m_dict_size) >= (TDEFL_MIN_MATCH_LEN - 1)) {
      // 计算目标位置和插入位置
      mz_uint dst_pos = (d->m_lookahead_pos + d->m_lookahead_size) &
                        TDEFL_LZ_DICT_SIZE_MASK,
              ins_pos = d->m_lookahead_pos + d->m_lookahead_size - 2;
      // 计算哈希值
      mz_uint hash = (d->m_dict[ins_pos & TDEFL_LZ_DICT_SIZE_MASK]
                      << TDEFL_LZ_HASH_SHIFT) ^
                     d->m_dict[(ins_pos + 1) & TDEFL_LZ_DICT_SIZE_MASK];
      // 计算需要处理的字节数
      mz_uint num_bytes_to_process = (mz_uint)MZ_MIN(
          src_buf_left, TDEFL_MAX_MATCH_LEN - d->m_lookahead_size);
      // 设置源数据结束位置
      const mz_uint8 *pSrc_end = pSrc + num_bytes_to_process;
      // 更新剩余源数据长度
      src_buf_left -= num_bytes_to_process;
      // 更新当前查找窗口大小
      d->m_lookahead_size += num_bytes_to_process;
      // 处理每个字节
      while (pSrc != pSrc_end) {
        // 获取当前字节
        mz_uint8 c = *pSrc++;
        // 更新字典
        d->m_dict[dst_pos] = c;
        // 如果目标位置小于最大匹配长度减1，则更新字典
        if (dst_pos < (TDEFL_MAX_MATCH_LEN - 1))
          d->m_dict[TDEFL_LZ_DICT_SIZE + dst_pos] = c;
        // 更新哈希值
        hash = ((hash << TDEFL_LZ_HASH_SHIFT) ^ c) & (TDEFL_LZ_HASH_SIZE - 1);
        // 更新下一个位置
        d->m_next[ins_pos & TDEFL_LZ_DICT_SIZE_MASK] = d->m_hash[hash];
        d->m_hash[hash] = (mz_uint16)(ins_pos);
        dst_pos = (dst_pos + 1) & TDEFL_LZ_DICT_SIZE_MASK;
        ins_pos++;
      }
    } else {
      // 如果不是压缩块的情况下，执行以下操作
      while ((src_buf_left) && (d->m_lookahead_size < TDEFL_MAX_MATCH_LEN)) {
        // 当源缓冲区还有数据，并且查找缓冲区大小小于最大匹配长度时，执行循环
        mz_uint8 c = *pSrc++;
        // 从源缓冲区中读取一个字节
        mz_uint dst_pos = (d->m_lookahead_pos + d->m_lookahead_size) &
                          TDEFL_LZ_DICT_SIZE_MASK;
        // 计算目标位置
        src_buf_left--;
        // 源缓冲区剩余字节数减一
        d->m_dict[dst_pos] = c;
        // 将读取的字节存入字典
        if (dst_pos < (TDEFL_MAX_MATCH_LEN - 1))
          d->m_dict[TDEFL_LZ_DICT_SIZE + dst_pos] = c;
        // 如果目标位置小于最大匹配长度减一，则将字节存入字典
        if ((++d->m_lookahead_size + d->m_dict_size) >= TDEFL_MIN_MATCH_LEN) {
          // 如果查找缓冲区大小加上字典大小大于等于最小匹配长度
          mz_uint ins_pos = d->m_lookahead_pos + (d->m_lookahead_size - 1) - 2;
          // 计算插入位置
          mz_uint hash = ((d->m_dict[ins_pos & TDEFL_LZ_DICT_SIZE_MASK]
                           << (TDEFL_LZ_HASH_SHIFT * 2)) ^
                          (d->m_dict[(ins_pos + 1) & TDEFL_LZ_DICT_SIZE_MASK]
                           << TDEFL_LZ_HASH_SHIFT) ^
                          c) &
                         (TDEFL_LZ_HASH_SIZE - 1);
          // 计算哈希值
          d->m_next[ins_pos & TDEFL_LZ_DICT_SIZE_MASK] = d->m_hash[hash];
          // 更新下一个位置
          d->m_hash[hash] = (mz_uint16)(ins_pos);
          // 更新哈希表
        }
      }
    }
    // 更新字典大小
    d->m_dict_size =
        MZ_MIN(TDEFL_LZ_DICT_SIZE - d->m_lookahead_size, d->m_dict_size);
    // 计算最小值
    if ((!flush) && (d->m_lookahead_size < TDEFL_MAX_MATCH_LEN))
      break;
    // 如果不是刷新操作且查找缓冲区大小小于最大匹配长度，则跳出循环

    /* Simple lazy/greedy parsing state machine. */
    // 简单的懒惰/贪婪解析状态机
    len_to_move = 1;
    // 移动长度为1
    cur_match_dist = 0;
    // 当前匹配距离为0
    cur_match_len =
        d->m_saved_match_len ? d->m_saved_match_len : (TDEFL_MIN_MATCH_LEN - 1);
    // 当前匹配长度为已保存的匹配长度或最小匹配长度减一
    cur_pos = d->m_lookahead_pos & TDEFL_LZ_DICT_SIZE_MASK;
    // 当前位置为查找缓冲区位置与字典大小掩码的与运算结果
    // 检查是否需要进行 RLE 匹配或强制使用所有原始块
    if (d->m_flags & (TDEFL_RLE_MATCHES | TDEFL_FORCE_ALL_RAW_BLOCKS)) {
        // 如果字典大小不为零且不强制使用所有原始块
        if ((d->m_dict_size) && (!(d->m_flags & TDEFL_FORCE_ALL_RAW_BLOCKS))) {
            // 获取当前位置前一个位置的字节
            mz_uint8 c = d->m_dict[(cur_pos - 1) & TDEFL_LZ_DICT_SIZE_MASK];
            cur_match_len = 0;
            // 在当前匹配长度小于向前查找长度的情况下循环
            while (cur_match_len < d->m_lookahead_size) {
                // 如果当前位置的字节与 c 不相等，则跳出循环
                if (d->m_dict[cur_pos + cur_match_len] != c)
                    break;
                cur_match_len++;
            }
            // 如果当前匹配长度小于最小匹配长度，则置为零
            if (cur_match_len < TDEFL_MIN_MATCH_LEN)
                cur_match_len = 0;
            else
                cur_match_dist = 1;
        }
    } else {
        // 在字典中查找匹配
        tdefl_find_match(d, d->m_lookahead_pos, d->m_dict_size,
                         d->m_lookahead_size, &cur_match_dist, &cur_match_len);
    }
    // 如果当前匹配长度为最小匹配长度且当前匹配距离大于等于 8KB，或者当前位置等于当前匹配距离，或者启用了过滤匹配且当前匹配长度小于等于 5
    if (((cur_match_len == TDEFL_MIN_MATCH_LEN) &&
         (cur_match_dist >= 8U * 1024U)) ||
        (cur_pos == cur_match_dist) ||
        ((d->m_flags & TDEFL_FILTER_MATCHES) && (cur_match_len <= 5))) {
        // 将当前匹配距离和匹配长度置为零
        cur_match_dist = cur_match_len = 0;
    }
    // 如果存在已保存的匹配长度
    if (d->m_saved_match_len) {
        // 如果当前匹配长度大于已保存的匹配长度
        if (cur_match_len > d->m_saved_match_len) {
            // 记录当前位置的原始字节
            tdefl_record_literal(d, (mz_uint8)d->m_saved_lit);
            // 如果当前匹配长度大于等于 128
            if (cur_match_len >= 128) {
                // 记录当前匹配
                tdefl_record_match(d, cur_match_len, cur_match_dist);
                d->m_saved_match_len = 0;
                len_to_move = cur_match_len;
            } else {
                // 更新已保存的原始字节、匹配距离和匹配长度
                d->m_saved_lit = d->m_dict[cur_pos];
                d->m_saved_match_dist = cur_match_dist;
                d->m_saved_match_len = cur_match_len;
            }
        } else {
            // 记录已保存的匹配
            tdefl_record_match(d, d->m_saved_match_len, d->m_saved_match_dist);
            len_to_move = d->m_saved_match_len - 1;
            d->m_saved_match_len = 0;
        }
    } else if (!cur_match_dist)
        // 如果当前匹配距离为零，则记录当前位置的原始字节
        tdefl_record_literal(d,
                             d->m_dict[MZ_MIN(cur_pos, sizeof(d->m_dict) - 1)]);
    else if ((d->m_greedy_parsing) || (d->m_flags & TDEFL_RLE_MATCHES) ||
             (cur_match_len >= 128)) {
        // 记录当前匹配
        tdefl_record_match(d, cur_match_len, cur_match_dist);
        len_to_move = cur_match_len;
    } else {
      // 如果不是最长匹配，则保存当前位置的字面量、匹配距离和匹配长度
      d->m_saved_lit = d->m_dict[MZ_MIN(cur_pos, sizeof(d->m_dict) - 1)];
      d->m_saved_match_dist = cur_match_dist;
      d->m_saved_match_len = cur_match_len;
    }
    /* 将前瞻指针向前移动 len_to_move 字节。 */
    d->m_lookahead_pos += len_to_move;
    // 断言前瞻大小大于等于 len_to_move
    MZ_ASSERT(d->m_lookahead_size >= len_to_move);
    // 更新前瞻大小
    d->m_lookahead_size -= len_to_move;
    // 更新字典大小
    d->m_dict_size =
        MZ_MIN(d->m_dict_size + len_to_move, (mz_uint)TDEFL_LZ_DICT_SIZE);
    /* 检查是否需要将当前的 LZ 编码刷新到内部输出缓冲区。 */
    if ((d->m_pLZ_code_buf > &d->m_lz_code_buf[TDEFL_LZ_CODE_BUF_SIZE - 8]) ||
        ((d->m_total_lz_bytes > 31 * 1024) &&
         (((((mz_uint)(d->m_pLZ_code_buf - d->m_lz_code_buf) * 115) >> 7) >=
           d->m_total_lz_bytes) ||
          (d->m_flags & TDEFL_FORCE_ALL_RAW_BLOCKS)))) {
      int n;
      // 保存当前源指针和源缓冲区剩余大小
      d->m_pSrc = pSrc;
      d->m_src_buf_left = src_buf_left;
      // 刷新块，如果返回值不为0，则根据情况返回 MZ_FALSE 或 MZ_TRUE
      if ((n = tdefl_flush_block(d, 0)) != 0)
        return (n < 0) ? MZ_FALSE : MZ_TRUE;
    }
  }

  // 恢复源指针和源缓冲区剩余大小，并返回 MZ_TRUE
  d->m_pSrc = pSrc;
  d->m_src_buf_left = src_buf_left;
  return MZ_TRUE;
static tdefl_status tdefl_flush_output_buffer(tdefl_compressor *d) {
  // 如果输入缓冲区非空，则计算输入缓冲区的大小
  if (d->m_pIn_buf_size) {
    *d->m_pIn_buf_size = d->m_pSrc - (const mz_uint8 *)d->m_pIn_buf;
  }

  // 如果输出缓冲区非空
  if (d->m_pOut_buf_size) {
    // 计算需要拷贝的数据大小
    size_t n = MZ_MIN(*d->m_pOut_buf_size - d->m_out_buf_ofs,
                      d->m_output_flush_remaining);
    // 将数据从输出缓冲区拷贝到输出缓冲区的偏移位置
    memcpy((mz_uint8 *)d->m_pOut_buf + d->m_out_buf_ofs,
           d->m_output_buf + d->m_output_flush_ofs, n);
    // 更新输出缓冲区的偏移位置和剩余数据大小
    d->m_output_flush_ofs += (mz_uint)n;
    d->m_output_flush_remaining -= (mz_uint)n;
    d->m_out_buf_ofs += n;

    // 更新输出缓冲区的大小
    *d->m_pOut_buf_size = d->m_out_buf_ofs;
  }

  // 返回压缩状态
  return (d->m_finished && !d->m_output_flush_remaining) ? TDEFL_STATUS_DONE
                                                         : TDEFL_STATUS_OKAY;
}

tdefl_status tdefl_compress(tdefl_compressor *d, const void *pIn_buf,
                            size_t *pIn_buf_size, void *pOut_buf,
                            size_t *pOut_buf_size, tdefl_flush flush) {
  // 如果压缩器为空
  if (!d) {
    // 如果输入缓冲区大小非空，则置为0
    if (pIn_buf_size)
      *pIn_buf_size = 0;
    // 如果输出缓冲区大小非空，则置为0
    if (pOut_buf_size)
      *pOut_buf_size = 0;
    // 返回错误状态
    return TDEFL_STATUS_BAD_PARAM;
  }

  // 设置压缩器的输入和输出缓冲区
  d->m_pIn_buf = pIn_buf;
  d->m_pIn_buf_size = pIn_buf_size;
  d->m_pOut_buf = pOut_buf;
  d->m_pOut_buf_size = pOut_buf_size;
  d->m_pSrc = (const mz_uint8 *)(pIn_buf);
  d->m_src_buf_left = pIn_buf_size ? *pIn_buf_size : 0;
  d->m_out_buf_ofs = 0;
  d->m_flush = flush;

  // 检查参数合法性
  if (((d->m_pPut_buf_func != NULL) ==
       ((pOut_buf != NULL) || (pOut_buf_size != NULL))) ||
      (d->m_prev_return_status != TDEFL_STATUS_OKAY) ||
      (d->m_wants_to_finish && (flush != TDEFL_FINISH)) ||
      (pIn_buf_size && *pIn_buf_size && !pIn_buf) ||
      (pOut_buf_size && *pOut_buf_size && !pOut_buf)) {
    // 如果输入缓冲区大小非空，则置为0
    if (pIn_buf_size)
      *pIn_buf_size = 0;
    // 如果输出缓冲区大小非空，则置为0
    if (pOut_buf_size)
      *pOut_buf_size = 0;
    // 返回错误状态
    return (d->m_prev_return_status = TDEFL_STATUS_BAD_PARAM);
  }
  // 标记是否需要结束压缩
  d->m_wants_to_finish |= (flush == TDEFL_FINISH);

  // 如果输出缓冲区还有数据需要刷新，或者已经完成压缩
  if ((d->m_output_flush_remaining) || (d->m_finished))
    # 调用 tdefl_flush_output_buffer 函数刷新输出缓冲区，并将返回值赋给 m_prev_return_status 字段，然后返回该值
    return (d->m_prev_return_status = tdefl_flush_output_buffer(d));
#if MINIZ_USE_UNALIGNED_LOADS_AND_STORES && MINIZ_LITTLE_ENDIAN
  // 检查是否满足条件进行快速压缩
  if (((d->m_flags & TDEFL_MAX_PROBES_MASK) == 1) &&
      ((d->m_flags & TDEFL_GREEDY_PARSING_FLAG) != 0) &&
      ((d->m_flags & (TDEFL_FILTER_MATCHES | TDEFL_FORCE_ALL_RAW_BLOCKS |
                      TDEFL_RLE_MATCHES)) == 0)) {
    // 如果快速压缩失败，则返回之前的状态
    if (!tdefl_compress_fast(d))
      return d->m_prev_return_status;
  } else
#endif /* #if MINIZ_USE_UNALIGNED_LOADS_AND_STORES && MINIZ_LITTLE_ENDIAN */
  {
    // 如果不满足快速压缩条件，则进行普通压缩
    if (!tdefl_compress_normal(d))
      return d->m_prev_return_status;
  }

  // 计算 Adler32 校验值
  if ((d->m_flags & (TDEFL_WRITE_ZLIB_HEADER | TDEFL_COMPUTE_ADLER32)) &&
      (pIn_buf))
    d->m_adler32 =
        (mz_uint32)mz_adler32(d->m_adler32, (const mz_uint8 *)pIn_buf,
                              d->m_pSrc - (const mz_uint8 *)pIn_buf);

  // 刷新输出缓冲区
  if ((flush) && (!d->m_lookahead_size) && (!d->m_src_buf_left) &&
      (!d->m_output_flush_remaining)) {
    // 如果刷新块失败，则返回之前的状态
    if (tdefl_flush_block(d, flush) < 0)
      return d->m_prev_return_status;
    d->m_finished = (flush == TDEFL_FINISH);
    // 如果是完全刷新，则清空哈希表和字典
    if (flush == TDEFL_FULL_FLUSH) {
      MZ_CLEAR_OBJ(d->m_hash);
      MZ_CLEAR_OBJ(d->m_next);
      d->m_dict_size = 0;
    }
  }

  // 返回刷新输出缓冲区的状态
  return (d->m_prev_return_status = tdefl_flush_output_buffer(d));
}

// 压缩缓冲区
tdefl_status tdefl_compress_buffer(tdefl_compressor *d, const void *pIn_buf,
                                   size_t in_buf_size, tdefl_flush flush) {
  // 断言压缩器的输出缓冲区函数存在
  MZ_ASSERT(d->m_pPut_buf_func);
  return tdefl_compress(d, pIn_buf, &in_buf_size, NULL, NULL, flush);
}

// 初始化压缩器
tdefl_status tdefl_init(tdefl_compressor *d,
                        tdefl_put_buf_func_ptr pPut_buf_func,
                        void *pPut_buf_user, int flags) {
  // 设置压缩器的输出缓冲区函数和用户数据
  d->m_pPut_buf_func = pPut_buf_func;
  d->m_pPut_buf_user = pPut_buf_user;
  d->m_flags = (mz_uint)(flags);
  // 计算最大探测次数
  d->m_max_probes[0] = 1 + ((flags & 0xFFF) + 2) / 3;
  d->m_greedy_parsing = (flags & TDEFL_GREEDY_PARSING_FLAG) != 0;
  d->m_max_probes[1] = 1 + (((flags & 0xFFF) >> 2) + 2) / 3;
  // 如果不是非确定性解析，则...
    // 清空哈希表
    MZ_CLEAR_OBJ(d->m_hash);
    // 重置各个变量的值为0
    d->m_lookahead_pos = d->m_lookahead_size = d->m_dict_size =
        d->m_total_lz_bytes = d->m_lz_code_buf_dict_pos = d->m_bits_in = 0;
    d->m_output_flush_ofs = d->m_output_flush_remaining = d->m_finished =
        d->m_block_index = d->m_bit_buffer = d->m_wants_to_finish = 0;
    // 设置指针指向 LZ 编码缓冲区的下一个位置
    d->m_pLZ_code_buf = d->m_lz_code_buf + 1;
    // 设置指针指向 LZ 标志缓冲区
    d->m_pLZ_flags = d->m_lz_code_buf;
    // 将 LZ 标志缓冲区的值设为0
    *d->m_pLZ_flags = 0;
    // 设置剩余标志位数为8
    d->m_num_flags_left = 8;
    // 设置输出缓冲区指针
    d->m_pOutput_buf = d->m_output_buf;
    d->m_pOutput_buf_end = d->m_output_buf;
    // 设置先前返回状态为 OK
    d->m_prev_return_status = TDEFL_STATUS_OKAY;
    // 初始化保存的匹配距离、匹配长度和文本长度
    d->m_saved_match_dist = d->m_saved_match_len = d->m_saved_lit = 0;
    // 初始化 Adler-32 校验值
    d->m_adler32 = 1;
    // 初始化输入输出缓冲区指针和大小
    d->m_pIn_buf = NULL;
    d->m_pOut_buf = NULL;
    d->m_pIn_buf_size = NULL;
    d->m_pOut_buf_size = NULL;
    // 设置刷新标志为非刷新状态
    d->m_flush = TDEFL_NO_FLUSH;
    // 初始化源数据指针和剩余缓冲区大小
    d->m_pSrc = NULL;
    d->m_src_buf_left = 0;
    // 输出缓冲区偏移量设为0
    d->m_out_buf_ofs = 0;
    // 如果不是非确定性解析标志，则清空字典
    if (!(flags & TDEFL_NONDETERMINISTIC_PARSING_FLAG))
        MZ_CLEAR_OBJ(d->m_dict);
    // 初始化哈夫曼编码计数数组
    memset(&d->m_huff_count[0][0], 0,
           sizeof(d->m_huff_count[0][0]) * TDEFL_MAX_HUFF_SYMBOLS_0);
    memset(&d->m_huff_count[1][0], 0,
           sizeof(d->m_huff_count[1][0]) * TDEFL_MAX_HUFF_SYMBOLS_1);
    // 返回 OK 状态
    return TDEFL_STATUS_OKAY;
// 获取先前的返回状态
tdefl_status tdefl_get_prev_return_status(tdefl_compressor *d) {
  return d->m_prev_return_status;
}

// 获取 Adler-32 校验和
mz_uint32 tdefl_get_adler32(tdefl_compressor *d) { return d->m_adler32; }

// 将内存数据压缩到输出
mz_bool tdefl_compress_mem_to_output(const void *pBuf, size_t buf_len,
                                     tdefl_put_buf_func_ptr pPut_buf_func,
                                     void *pPut_buf_user, int flags) {
  tdefl_compressor *pComp;
  mz_bool succeeded;
  // 检查输入参数是否有效
  if (((buf_len) && (!pBuf)) || (!pPut_buf_func))
    return MZ_FALSE;
  // 分配内存给压缩器
  pComp = (tdefl_compressor *)MZ_MALLOC(sizeof(tdefl_compressor));
  if (!pComp)
    return MZ_FALSE;
  // 初始化压缩器
  succeeded = (tdefl_init(pComp, pPut_buf_func, pPut_buf_user, flags) ==
               TDEFL_STATUS_OKAY);
  // 压缩数据
  succeeded =
      succeeded && (tdefl_compress_buffer(pComp, pBuf, buf_len, TDEFL_FINISH) ==
                    TDEFL_STATUS_DONE);
  // 释放内存
  MZ_FREE(pComp);
  return succeeded;
}

// 定义输出缓冲区结构
typedef struct {
  size_t m_size, m_capacity;
  mz_uint8 *m_pBuf;
  mz_bool m_expandable;
} tdefl_output_buffer;

// 输出缓冲区写入函数
static mz_bool tdefl_output_buffer_putter(const void *pBuf, int len,
                                          void *pUser) {
  tdefl_output_buffer *p = (tdefl_output_buffer *)pUser;
  size_t new_size = p->m_size + len;
  // 扩展缓冲区容量
  if (new_size > p->m_capacity) {
    size_t new_capacity = p->m_capacity;
    mz_uint8 *pNew_buf;
    if (!p->m_expandable)
      return MZ_FALSE;
    do {
      new_capacity = MZ_MAX(128U, new_capacity << 1U);
    } while (new_size > new_capacity);
    pNew_buf = (mz_uint8 *)MZ_REALLOC(p->m_pBuf, new_capacity);
    if (!pNew_buf)
      return MZ_FALSE;
    p->m_pBuf = pNew_buf;
    p->m_capacity = new_capacity;
  }
  // 将数据复制到缓冲区
  memcpy((mz_uint8 *)p->m_pBuf + p->m_size, pBuf, len);
  p->m_size = new_size;
  return MZ_TRUE;
}

// 将内存数据压缩到堆中
void *tdefl_compress_mem_to_heap(const void *pSrc_buf, size_t src_buf_len,
                                 size_t *pOut_len, int flags) {
  tdefl_output_buffer out_buf;
  MZ_CLEAR_OBJ(out_buf);
  // 检查输出长度指针是否有效
  if (!pOut_len)
    return MZ_FALSE;
  else
    # 将输出长度设置为0
    *pOut_len = 0;
    # 设置输出缓冲区为可扩展
    out_buf.m_expandable = MZ_TRUE;
    # 使用tdefl_compress_mem_to_output函数将源数据压缩到输出缓冲区中
    if (!tdefl_compress_mem_to_output(
            pSrc_buf, src_buf_len, tdefl_output_buffer_putter, &out_buf, flags))
        # 如果压缩失败，则返回空指针
        return NULL;
    # 将输出缓冲区的大小赋值给pOut_len
    *pOut_len = out_buf.m_size;
    # 返回输出缓冲区的指针
    return out_buf.m_pBuf;
}

// 将内存中的数据压缩到另一块内存中
size_t tdefl_compress_mem_to_mem(void *pOut_buf, size_t out_buf_len,
                                 const void *pSrc_buf, size_t src_buf_len,
                                 int flags) {
  // 定义输出缓冲区
  tdefl_output_buffer out_buf;
  // 清空输出缓冲区对象
  MZ_CLEAR_OBJ(out_buf);
  // 如果输出缓冲区为空，则返回0
  if (!pOut_buf)
    return 0;
  // 设置输出缓冲区的指针和容量
  out_buf.m_pBuf = (mz_uint8 *)pOut_buf;
  out_buf.m_capacity = out_buf_len;
  // 将源数据压缩到输出缓冲区中
  if (!tdefl_compress_mem_to_output(
          pSrc_buf, src_buf_len, tdefl_output_buffer_putter, &out_buf, flags))
    return 0;
  // 返回压缩后的数据大小
  return out_buf.m_size;
}

// 定义用于压缩的探测次数数组
static const mz_uint s_tdefl_num_probes[11] = {0,   1,   6,   32,  16,  32,
                                               128, 256, 512, 768, 1500};

/* 根据 ZIP 参数创建压缩标志 */
mz_uint tdefl_create_comp_flags_from_zip_params(int level, int window_bits,
                                                int strategy) {
  // 初始化压缩标志
  mz_uint comp_flags =
      s_tdefl_num_probes[(level >= 0) ? MZ_MIN(10, level) : MZ_DEFAULT_LEVEL] |
      ((level <= 3) ? TDEFL_GREEDY_PARSING_FLAG : 0);
  // 如果窗口位数大于0，则设置写入 ZLIB 头标志
  if (window_bits > 0)
    comp_flags |= TDEFL_WRITE_ZLIB_HEADER;

  // 根据不同的级别和策略设置不同的压缩标志
  if (!level)
    comp_flags |= TDEFL_FORCE_ALL_RAW_BLOCKS;
  else if (strategy == MZ_FILTERED)
    comp_flags |= TDEFL_FILTER_MATCHES;
  else if (strategy == MZ_HUFFMAN_ONLY)
    comp_flags &= ~TDEFL_MAX_PROBES_MASK;
  else if (strategy == MZ_FIXED)
    comp_flags |= TDEFL_FORCE_ALL_STATIC_BLOCKS;
  else if (strategy == MZ_RLE)
    comp_flags |= TDEFL_RLE_MATCHES;

  // 返回压缩标志
  return comp_flags;
}

#ifdef _MSC_VER
#pragma warning(push)
#pragma warning(disable : 4204) /* nonstandard extension used : non-constant   \
                                   aggregate initializer (also supported by    \
                                   GNU C and C99, so no big deal) */
#endif
/* 使用 Alex Evans 编写的简单 PNG 写入函数，2011 年发布到公共领域
   https://gist.github.com/908299，更多上下文请查看
   http://altdevblogaday.org/2011/04/06/a-smaller-jpg-encoder/。
   这实际上是对 Alex 原始代码的修改，以便通过 pngcheck 验证生成的 PNG 文件。 */
void *tdefl_write_image_to_png_file_in_memory_ex(const void *pImage, int w,
                                                 int h, int num_chans,
                                                 size_t *pLen_out,
                                                 mz_uint level, mz_bool flip) {
  /* 在这里使用此数组的本地副本，以防定义了 MINIZ_NO_ZLIB_APIS。 */
  static const mz_uint s_tdefl_png_num_probes[11] = {
      0, 1, 6, 32, 16, 32, 128, 256, 512, 768, 1500};
  tdefl_compressor *pComp =
      (tdefl_compressor *)MZ_MALLOC(sizeof(tdefl_compressor));
  tdefl_output_buffer out_buf;
  int i, bpl = w * num_chans, y, z;
  mz_uint32 c;
  *pLen_out = 0;
  if (!pComp)
    return NULL;
  MZ_CLEAR_OBJ(out_buf);
  out_buf.m_expandable = MZ_TRUE;
  out_buf.m_capacity = 57 + MZ_MAX(64, (1 + bpl) * h);
  if (NULL == (out_buf.m_pBuf = (mz_uint8 *)MZ_MALLOC(out_buf.m_capacity))) {
    MZ_FREE(pComp);
    return NULL;
  }
  /* 写入虚拟头部 */
  for (z = 41; z; --z)
    tdefl_output_buffer_putter(&z, 1, &out_buf);
  /* 压缩图像数据 */
  tdefl_init(pComp, tdefl_output_buffer_putter, &out_buf,
             s_tdefl_png_num_probes[MZ_MIN(10, level)] |
                 TDEFL_WRITE_ZLIB_HEADER);
  for (y = 0; y < h; ++y) {
    tdefl_compress_buffer(pComp, &z, 1, TDEFL_NO_FLUSH);
    tdefl_compress_buffer(pComp,
                          (mz_uint8 *)pImage + (flip ? (h - 1 - y) : y) * bpl,
                          bpl, TDEFL_NO_FLUSH);
  }
  if (tdefl_compress_buffer(pComp, NULL, 0, TDEFL_FINISH) !=
      TDEFL_STATUS_DONE) {
    MZ_FREE(pComp);
    MZ_FREE(out_buf.m_pBuf);
  }
}
    // 返回空指针
    return NULL;
  }
  /* 写入真实头部 */
  // 计算输出缓冲区的长度
  *pLen_out = out_buf.m_size - 41;
  {
    // 静态常量数组，表示通道信息
    static const mz_uint8 chans[] = {0x00, 0x00, 0x04, 0x02, 0x06};
    // PNG 文件头部信息
    mz_uint8 pnghdr[41] = {0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a, 0x00,
                           0x00, 0x00, 0x0d, 0x49, 0x48, 0x44, 0x52, 0x00, 0x00,
                           0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x08, 0x00, 0x00,
                           0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
                           0x00, 0x49, 0x44, 0x41, 0x54};
    // 设置 PNG 文件头部信息中的宽度和高度
    pnghdr[18] = (mz_uint8)(w >> 8);
    pnghdr[19] = (mz_uint8)w;
    pnghdr[22] = (mz_uint8)(h >> 8);
    pnghdr[23] = (mz_uint8)h;
    // 设置 PNG 文件头部信息中的通道数
    pnghdr[25] = chans[num_chans];
    // 设置 PNG 文件头部信息中的数据长度
    pnghdr[33] = (mz_uint8)(*pLen_out >> 24);
    pnghdr[34] = (mz_uint8)(*pLen_out >> 16);
    pnghdr[35] = (mz_uint8)(*pLen_out >> 8);
    pnghdr[36] = (mz_uint8)*pLen_out;
    // 计算 PNG 文件头部信息的 CRC 校验值
    c = (mz_uint32)mz_crc32(MZ_CRC32_INIT, pnghdr + 12, 17);
    for (i = 0; i < 4; ++i, c <<= 8)
      ((mz_uint8 *)(pnghdr + 29))[i] = (mz_uint8)(c >> 24);
    // 将 PNG 文件头部信息拷贝到输出缓冲区
    memcpy(out_buf.m_pBuf, pnghdr, 41);
  }
  /* 写入尾部（IDAT CRC-32，后跟 IEND 块） */
  // 如果无法将数据写入输出缓冲区，则返回空指针
  if (!tdefl_output_buffer_putter(
          "\0\0\0\0\0\0\0\0\x49\x45\x4e\x44\xae\x42\x60\x82", 16, &out_buf)) {
    *pLen_out = 0;
    MZ_FREE(pComp);
    MZ_FREE(out_buf.m_pBuf);
    return NULL;
  }
  // 计算输出缓冲区中的数据的 CRC 校验值
  c = (mz_uint32)mz_crc32(MZ_CRC32_INIT, out_buf.m_pBuf + 41 - 4,
                          *pLen_out + 4);
  for (i = 0; i < 4; ++i, c <<= 8)
    (out_buf.m_pBuf + out_buf.m_size - 16)[i] = (mz_uint8)(c >> 24);
  /* 计算文件的最终大小，获取压缩数据缓冲区并返回 */
  // 更新文件的最终大小
  *pLen_out += 57;
  MZ_FREE(pComp);
  // 返回输出缓冲区的指针
  return out_buf.m_pBuf;
}
// 结束 C++ 语言的 extern "C" 块

// 将图像数据写入 PNG 文件并保存在内存中，返回结果数据指针
void *tdefl_write_image_to_png_file_in_memory(const void *pImage, int w, int h,
                                              int num_chans, size_t *pLen_out) {
  /* Level 6 corresponds to TDEFL_DEFAULT_MAX_PROBES or MZ_DEFAULT_LEVEL (but we
   * can't depend on MZ_DEFAULT_LEVEL being available in case the zlib API's
   * where #defined out) */
  // 调用 tdefl_write_image_to_png_file_in_memory_ex 函数，指定压缩级别为 6
  return tdefl_write_image_to_png_file_in_memory_ex(pImage, w, h, num_chans,
                                                    pLen_out, 6, MZ_FALSE);
}

#ifndef MINIZ_NO_MALLOC
/* Allocate the tdefl_compressor and tinfl_decompressor structures in C so that
 */
/* non-C language bindings to tdefL_ and tinfl_ API don't need to worry about */
/* structure size and allocation mechanism. */
// 分配 tdefl_compressor 结构体的内存空间
tdefl_compressor *tdefl_compressor_alloc(void) {
  return (tdefl_compressor *)MZ_MALLOC(sizeof(tdefl_compressor));
}

// 释放 tdefl_compressor 结构体的内存空间
void tdefl_compressor_free(tdefl_compressor *pComp) { MZ_FREE(pComp); }
#endif

#ifdef _MSC_VER
#pragma warning(pop)
#endif

#ifdef __cplusplus
}
#endif
#ifdef __cplusplus
extern "C" {
#endif

/* ------------------- Low-level Decompression (completely independent from all
 * compression API's) */

// 定义 TINFL_MEMCPY 宏，用于内存拷贝操作
#define TINFL_MEMCPY(d, s, l) memcpy(d, s, l)
// 定义 TINFL_MEMSET 宏，用于内存设置操作
#define TINFL_MEMSET(p, c, l) memset(p, c, l)

// 定义 TINFL_CR_BEGIN 宏，开始解压缩状态机
#define TINFL_CR_BEGIN                                                         \
  switch (r->m_state) {                                                        \
  case 0:
// 定义 TINFL_CR_RETURN 宏，返回解压缩状态和结果
#define TINFL_CR_RETURN(state_index, result)                                   \
  do {                                                                         \
    // 将 result 赋值给 status
    status = result;                                                           \
    // 将 state_index 赋值给 r->m_state
    r->m_state = state_index;                                                  \
    // 跳转到 common_exit 标签处
    goto common_exit;                                                          \
  // state_index 标签
  case state_index:;                                                           \
  }                                                                            \
  // 结束宏定义
  MZ_MACRO_END
# 定义一个宏，用于在状态机中返回指定结果，无限循环直到结果返回
#define TINFL_CR_RETURN_FOREVER(state_index, result)                           \
  do {                                                                         \
    for (;;) {                                                                 \
      TINFL_CR_RETURN(state_index, result);                                    \
    }                                                                          \
  }                                                                            \
  MZ_MACRO_END

# 定义一个宏，用于结束状态机
#define TINFL_CR_FINISH }

# 定义一个宏，用于从输入缓冲区中获取一个字节
#define TINFL_GET_BYTE(state_index, c)                                         \
  do {                                                                         \
    while (pIn_buf_cur >= pIn_buf_end) {                                       \
      # 如果输入缓冲区已经读取完毕，根据解压标志返回相应状态
      TINFL_CR_RETURN(state_index,                                             \
                      (decomp_flags & TINFL_FLAG_HAS_MORE_INPUT)               \
                          ? TINFL_STATUS_NEEDS_MORE_INPUT                      \
                          : TINFL_STATUS_FAILED_CANNOT_MAKE_PROGRESS);         \
    }                                                                          \
    # 从输入缓冲区中读取一个字节
    c = *pIn_buf_cur++;                                                        \
  }                                                                            \
  MZ_MACRO_END

# 定义一个宏，用于从输入缓冲区中获取指定位数的比特
#define TINFL_NEED_BITS(state_index, n)                                        \
  do {                                                                         \
    mz_uint c;                                                                 \
    # 从输入缓冲区中获取一个字节
    TINFL_GET_BYTE(state_index, c);                                            \
    # 将获取的字节左移已有比特数位，然后加到比特缓冲区中
    bit_buf |= (((tinfl_bit_buf_t)c) << num_bits);                             \
    num_bits += 8;                                                             \
  } while (num_bits < (mz_uint)(n))
# 定义宏 TINFL_SKIP_BITS，用于跳过指定数量的位
#define TINFL_SKIP_BITS(state_index, n)                                        \
  do {                                                                         \
    # 如果剩余位数小于 n，则需要更多位
    if (num_bits < (mz_uint)(n)) {                                             \
      TINFL_NEED_BITS(state_index, n);                                         \
    }                                                                          \
    # 将位缓冲区右移 n 位
    bit_buf >>= (n);                                                           \
    # 减去已处理的位数
    num_bits -= (n);                                                           \
  }                                                                            \
  MZ_MACRO_END

# 定义宏 TINFL_GET_BITS，用于获取指定数量的位
#define TINFL_GET_BITS(state_index, b, n)                                      \
  do {                                                                         \
    # 如果剩余位数小于 n，则需要更多位
    if (num_bits < (mz_uint)(n)) {                                             \
      TINFL_NEED_BITS(state_index, n);                                         \
    }                                                                          \
    # 将位缓冲区与掩码（n 位全为 1）进行与操作，获取 n 位的值
    b = bit_buf & ((1 << (n)) - 1);                                            \
    # 将位缓冲区右移 n 位
    bit_buf >>= (n);                                                           \
    # 减去已处理的位数
    num_bits -= (n);                                                           \
  }                                                                            \
  MZ_MACRO_END

# TINFL_HUFF_BITBUF_FILL() 仅在输入缓冲区中剩余的字节数少于 2 时才会使用
# 它从输入流中读取足够的字节，以便解码下一个哈夫曼码（绝对不会多读取）
# 它通过尝试使用位缓冲区中当前存在的位来完全解码一个哈夫曼码
# 如果失败，则读取另一个字节，再次尝试，直到成功或者位缓冲区包含 >=15 位（deflate 的最大哈夫曼码大小）
#define TINFL_HUFF_BITBUF_FILL(state_index, pHuff)                             \
  do {                                                                         \
    // 从快速查找表中获取当前 bit_buf 对应的编码长度
    temp = (pHuff)->m_look_up[bit_buf & (TINFL_FAST_LOOKUP_SIZE - 1)];         \
    // 如果编码长度大于等于 0，表示找到了对应的编码
    if (temp >= 0) {                                                           \
      // 获取编码长度
      code_len = temp >> 9;                                                    \
      // 如果当前 bit_buf 中的比特数大于等于编码长度，跳出循环
      if ((code_len) && (num_bits >= code_len))                                \
        break;                                                                 \
    } else if (num_bits > TINFL_FAST_LOOKUP_BITS) {                            \
      // 如果当前比特数大于快速查找表的比特数，设置编码长度为快速查找表的比特数
      code_len = TINFL_FAST_LOOKUP_BITS;                                       \
      // 从树中获取编码
      do {                                                                     \
        temp = (pHuff)->m_tree[~temp + ((bit_buf >> code_len++) & 1)];         \
      } while ((temp < 0) && (num_bits >= (code_len + 1)));                    \
      // 如果找到了编码，跳出循环
      if (temp >= 0)                                                           \
        break;                                                                 \
    }                                                                          \
    // 从输入流中获取一个字节
    TINFL_GET_BYTE(state_index, c);                                            \
    // 将获取的字节添加到 bit_buf 中
    bit_buf |= (((tinfl_bit_buf_t)c) << num_bits);                             \
    // 更新比特数
    num_bits += 8;                                                             \
  } while (num_bits < 15);

/* TINFL_HUFF_DECODE() decodes the next Huffman coded symbol. It's more complex
 * than you would initially expect because the zlib API expects the decompressor
 * to never read */
/* beyond the final byte of the deflate stream. (In other words, when this macro
 * wants to read another byte from the input, it REALLY needs another byte in
 * order to fully */
/* 解码下一个哈夫曼编码。正确处理这一点在原始的deflate（非zlib）流中尤为重要，因为它们后面没有字节对齐的adler-32。 */
/* 慢路径仅在输入缓冲区的最后执行。 */
/* v1.16：原始宏处理了传入输入缓冲区的最后情况，但我们还需要处理用户传入1+zillion字节的情况 */
/* 在deflate数据之后，我们的非保守的预读路径不会在这里启动。这更加棘手。 */
#define TINFL_HUFF_DECODE(state_index, sym, pHuff)                             \
  do {                                                                         \
    int temp;                                                                  \
    mz_uint code_len, c;                                                       \
    if (num_bits < 15) {                                                       \
      if ((pIn_buf_end - pIn_buf_cur) < 2) {                                   \
        TINFL_HUFF_BITBUF_FILL(state_index, pHuff);                            \
      } else {                                                                 \
        bit_buf |= (((tinfl_bit_buf_t)pIn_buf_cur[0]) << num_bits) |           \
                   (((tinfl_bit_buf_t)pIn_buf_cur[1]) << (num_bits + 8));      \
        pIn_buf_cur += 2;                                                      \
        num_bits += 16;                                                        \
      }                                                                        \
    }                                                                          \
    if ((temp = (pHuff)->m_look_up[bit_buf & (TINFL_FAST_LOOKUP_SIZE - 1)]) >= \
        0)                                                                     \
      code_len = temp >> 9, temp &= 511;                                       \
    else {                                                                     \  # 如果不是特殊情况，执行以下代码块
      code_len = TINFL_FAST_LOOKUP_BITS;                                       \  # 初始化编码长度为快速查找位数
      do {                                                                     \  # 开始循环
        temp = (pHuff)->m_tree[~temp + ((bit_buf >> code_len++) & 1)];         \  # 根据当前位的值和编码长度获取对应的值
      } while (temp < 0);                                                      \  # 当获取的值小于0时继续循环
    }                                                                          \  # 结束 else 语句块
    sym = temp;                                                                \  # 将获取的值赋给 sym
    bit_buf >>= code_len;                                                      \  # 将 bit_buf 右移编码长度位
    num_bits -= code_len;                                                      \  # 减去编码长度
  }                                                                            \  # 结束 do-while 循环
  MZ_MACRO_END                                                                 \  # 结束宏定义

    *pIn_buf_size = *pOut_buf_size = 0;                                       \  # 将输入缓冲区大小和输出缓冲区大小都设置为0
    return TINFL_STATUS_BAD_PARAM;                                             \  # 返回参数错误状态
  }                                                                            \  # 结束 if 语句块

  num_bits = r->m_num_bits;                                                    \  # 将 r 结构体中的 m_num_bits 赋给 num_bits
  bit_buf = r->m_bit_buf;                                                      \  # 将 r 结构体中的 m_bit_buf 赋给 bit_buf
  dist = r->m_dist;                                                            \  # 将 r 结构体中的 m_dist 赋给 dist
  counter = r->m_counter;                                                      \  # 将 r 结构体中的 m_counter 赋给 counter
  num_extra = r->m_num_extra;                                                  \  # 将 r 结构体中的 m_num_extra 赋给 num_extra
  dist_from_out_buf_start = r->m_dist_from_out_buf_start;                      \  # 将 r 结构体中的 m_dist_from_out_buf_start 赋给 dist_from_out_buf_start
  TINFL_CR_BEGIN                                                              \  # 开始解压缩

  bit_buf = num_bits = dist = counter = num_extra = r->m_zhdr0 = r->m_zhdr1 = 0;  \  # 将多个变量初始化为0
  r->m_z_adler32 = r->m_check_adler32 = 1;                                    \  # 将 r 结构体中的 m_z_adler32 和 m_check_adler32 初始化为1
  if (decomp_flags & TINFL_FLAG_PARSE_ZLIB_HEADER) {                          \  # 如果解压缩标志包含解析 ZLIB 头部
    TINFL_GET_BYTE(1, r->m_zhdr0);                                            \  # 获取字节数据到 r 结构体中的 m_zhdr0
    TINFL_GET_BYTE(2, r->m_zhdr1);                                            \  # 获取字节数据到 r 结构体中的 m_zhdr1
    counter = (((r->m_zhdr0 * 256 + r->m_zhdr1) % 31 != 0) ||                \  # 计算 counter 值
               (r->m_zhdr1 & 32) || ((r->m_zhdr0 & 15) != 8));                \  # 判断条件
    if (!(decomp_flags & TINFL_FLAG_USING_NON_WRAPPING_OUTPUT_BUF))           \  # 如果解压缩标志不包含使用非环绕输出缓冲区
      counter |= (((1U << (8U + (r->m_zhdr0 >> 4))) > 32768U) ||              \  # 计算 counter 值
                  ((out_buf_size_mask + 1) <                                   \  # 判断条件
                   (size_t)(1U << (8U + (r->m_zhdr0 >> 4))));                 \  # 计算条件
    if (counter) {                                                            \  # 如果 counter 为真
      TINFL_CR_RETURN_FOREVER(36, TINFL_STATUS_FAILED);                       \  # 返回失败状态
    }
  }

  do {                                                                         \  # 开始 do-while 循环
    TINFL_GET_BITS(3, r->m_final, 3);                                         \  # 获取位数据到 r 结构体中的 m_final
    r->m_type = r->m_final >> 1;                                              \  # 将 m_final 右移1位赋给 m_type
    # 如果解压缩器的类型为0
    if (r->m_type == 0) {
      # 跳过5位，num_bits & 7表示取num_bits的低3位
      TINFL_SKIP_BITS(5, num_bits & 7);
      # 循环4次
      for (counter = 0; counter < 4; ++counter) {
        # 如果num_bits不为0
        if (num_bits)
          # 读取6位，存入r->m_raw_header[counter]，每次8位
          TINFL_GET_BITS(6, r->m_raw_header[counter], 8);
        else
          # 读取一个字节，存入r->m_raw_header[counter]
          TINFL_GET_BYTE(7, r->m_raw_header[counter]);
      }
      # 检查头部是否有效
      if ((counter = (r->m_raw_header[0] | (r->m_raw_header[1] << 8))) !=
          (mz_uint)(0xFFFF ^
                    (r->m_raw_header[2] | (r->m_raw_header[3] << 8)))) {
        # 返回解压缩失败
        TINFL_CR_RETURN_FOREVER(39, TINFL_STATUS_FAILED);
      }
      # 解压缩数据
      while ((counter) && (num_bits)) {
        # 读取8位作为距离
        TINFL_GET_BITS(51, dist, 8);
        # 当输出缓冲区已满时
        while (pOut_buf_cur >= pOut_buf_end) {
          # 返回需要更多输出
          TINFL_CR_RETURN(52, TINFL_STATUS_HAS_MORE_OUTPUT);
        }
        # 将距离写入输出缓冲区
        *pOut_buf_cur++ = (mz_uint8)dist;
        counter--;
      }
      # 继续解压缩数据
      while (counter) {
        size_t n;
        # 当输出缓冲区已满时
        while (pOut_buf_cur >= pOut_buf_end) {
          # 返回需要更多输出
          TINFL_CR_RETURN(9, TINFL_STATUS_HAS_MORE_OUTPUT);
        }
        # 当输入缓冲区已空时
        while (pIn_buf_cur >= pIn_buf_end) {
          # 返回需要更多输入或者解压缩失败
          TINFL_CR_RETURN(38, (decomp_flags & TINFL_FLAG_HAS_MORE_INPUT)
                                  ? TINFL_STATUS_NEEDS_MORE_INPUT
                                  : TINFL_STATUS_FAILED_CANNOT_MAKE_PROGRESS);
        }
        # 计算可复制的最小字节数
        n = MZ_MIN(MZ_MIN((size_t)(pOut_buf_end - pOut_buf_cur),
                          (size_t)(pIn_buf_end - pIn_buf_cur)),
                   counter);
        # 复制数据到输出缓冲区
        TINFL_MEMCPY(pOut_buf_cur, pIn_buf_cur, n);
        pIn_buf_cur += n;
        pOut_buf_cur += n;
        counter -= (mz_uint)n;
      }
    } else if (r->m_type == 3) {
      # 返回解压缩失败
      TINFL_CR_RETURN_FOREVER(10, TINFL_STATUS_FAILED);
#if TINFL_USE_64BIT_BITBUF
            // 如果使用 64 位位缓冲区，检查当前位数是否小于 30
            if (num_bits < 30) {
              // 将输入缓冲区中的 32 位数据读取到位缓冲区中
              bit_buf |=
                  (((tinfl_bit_buf_t)MZ_READ_LE32(pIn_buf_cur)) << num_bits);
              // 更新输入缓冲区指针和位数
              pIn_buf_cur += 4;
              num_bits += 32;
            }
#else
            // 如果不使用 64 位位缓冲区，检查当前位数是否小于 15
            if (num_bits < 15) {
              // 将输入缓冲区中的 16 位数据读取到位缓冲区中
              bit_buf |=
                  (((tinfl_bit_buf_t)MZ_READ_LE16(pIn_buf_cur)) << num_bits);
              // 更新输入缓冲区指针和位数
              pIn_buf_cur += 2;
              num_bits += 16;
            }
#endif
            // 根据位缓冲区中的数据查找对应的符号
            if ((sym2 =
                     r->m_tables[0]
                         .m_look_up[bit_buf & (TINFL_FAST_LOOKUP_SIZE - 1)]) >=
                0)
              code_len = sym2 >> 9;
            else {
              code_len = TINFL_FAST_LOOKUP_BITS;
              // 根据查找表和位缓冲区中的数据解码出符号
              do {
                sym2 = r->m_tables[0]
                           .m_tree[~sym2 + ((bit_buf >> code_len++) & 1)];
              } while (sym2 < 0);
            }
            // 更新计数器、位缓冲区和位数
            counter = sym2;
            bit_buf >>= code_len;
            num_bits -= code_len;
            // 如果计数器的最高位为 1，跳出循环
            if (counter & 256)
              break;

#if !TINFL_USE_64BIT_BITBUF
            // 如果不使用 64 位位缓冲区，检查当前位数是否小于 15
            if (num_bits < 15) {
              // 将输入缓冲区中的 16 位数据读取到位缓冲区中
              bit_buf |=
                  (((tinfl_bit_buf_t)MZ_READ_LE16(pIn_buf_cur)) << num_bits);
              // 更新输入缓冲区指针和位数
              pIn_buf_cur += 2;
              num_bits += 16;
            }
#if MINIZ_USE_UNALIGNED_LOADS_AND_STORES
        else if ((counter >= 9) && (counter <= dist)) {
          // 计算源指针的结束位置
          const mz_uint8 *pSrc_end = pSrc + (counter & ~7);
          // 处理未对齐的加载和存储
          do {
#ifdef MINIZ_UNALIGNED_USE_MEMCPY
            // 使用 memcpy 将数据从源指针复制到输出缓冲区
            memcpy(pOut_buf_cur, pSrc, sizeof(mz_uint32) * 2);
#else
            // 将数据从源指针复制到输出缓冲区
            ((mz_uint32 *)pOut_buf_cur)[0] = ((const mz_uint32 *)pSrc)[0];
            ((mz_uint32 *)pOut_buf_cur)[1] = ((const mz_uint32 *)pSrc)[1];
#ifdef
            // 如果 counter 大于等于 8，将 pSrc 指针向后移动 8 个字节
            pOut_buf_cur += 8;
          } while ((pSrc += 8) < pSrc_end);
          // 如果 counter 与 7 按位与的结果小于 3
          if ((counter &= 7) < 3) {
            // 如果 counter 大于 0
            if (counter) {
              // 将 pSrc 指向的字节复制到 pOut_buf_cur 指向的位置
              pOut_buf_cur[0] = pSrc[0];
              // 如果 counter 大于 1，将 pSrc 指向的第二个字节复制到 pOut_buf_cur 指向的位置
              if (counter > 1)
                pOut_buf_cur[1] = pSrc[1];
              // pOut_buf_cur 指针向后移动 counter 个字节
              pOut_buf_cur += counter;
            }
            // 继续循环
            continue;
          }
        }
#endif
        // 当 counter 大于 2 时
        while (counter > 2) {
          // 将 pSrc 指向的三个字节复制到 pOut_buf_cur 指向的位置
          pOut_buf_cur[0] = pSrc[0];
          pOut_buf_cur[1] = pSrc[1];
          pOut_buf_cur[2] = pSrc[2];
          // pOut_buf_cur 指针向后移动 3 个字节
          pOut_buf_cur += 3;
          pSrc += 3;
          counter -= 3;
        }
        // 如果 counter 大于 0
        if (counter > 0) {
          // 将 pSrc 指向的字节复制到 pOut_buf_cur 指向的位置
          pOut_buf_cur[0] = pSrc[0];
          // 如果 counter 大于 1，将 pSrc 指向的第二个字节复制到 pOut_buf_cur 指向的位置
          if (counter > 1)
            pOut_buf_cur[1] = pSrc[1];
          // pOut_buf_cur 指针向后移动 counter 个字节
          pOut_buf_cur += counter;
        }
      }
    }
  } while (!(r->m_final & 1));

  /* Ensure byte alignment and put back any bytes from the bitbuf if we've
   * looked ahead too far on gzip, or other Deflate streams followed by
   * arbitrary data. */
  /* I'm being super conservative here. A number of simplifications can be made
   * to the byte alignment part, and the Adler32 check shouldn't ever need to
   * worry about reading from the bitbuf now. */
  // 确保字节对齐，并在我们在 gzip 或其他 Deflate 流后面跟随任意数据时，将 bitbuf 中的任何字节放回
  // 这里我非常保守。字节对齐部分可以进行一些简化，Adler32 检查现在不应该再从 bitbuf 中读取
  TINFL_SKIP_BITS(32, num_bits & 7);
  while ((pIn_buf_cur > pIn_buf_next) && (num_bits >= 8)) {
    --pIn_buf_cur;
    num_bits -= 8;
  }
  bit_buf &= (tinfl_bit_buf_t)((((mz_uint64)1) << num_bits) - (mz_uint64)1);
  MZ_ASSERT(!num_bits); /* if this assert fires then we've read beyond the end
                           of non-deflate/zlib streams with following data (such
                           as gzip streams). */

  if (decomp_flags & TINFL_FLAG_PARSE_ZLIB_HEADER) {
    for (counter = 0; counter < 4; ++counter) {
      mz_uint s;
      if (num_bits)
        TINFL_GET_BITS(41, s, 8);
      else
        TINFL_GET_BYTE(42, s);
      r->m_z_adler32 = (r->m_z_adler32 << 8) | s;
    }
  }
  TINFL_CR_RETURN_FOREVER(34, TINFL_STATUS_DONE);

  TINFL_CR_FINISH
common_exit:
  /* 只要我们不告诉调用者我们需要更多输入来取得进展： */
  /* 将位缓冲区中的字节放回，以防我们在查看 gzip 或其他 Deflate 流后面的任意数据时向前查看太远。 */
  /* 但是，我们需要非常小心，不要推回任何我们明确知道需要向前取得进展的字节，否则会将调用者锁定在一个无限循环中。 */
  if ((status != TINFL_STATUS_NEEDS_MORE_INPUT) &&
      (status != TINFL_STATUS_FAILED_CANNOT_MAKE_PROGRESS)) {
    while ((pIn_buf_cur > pIn_buf_next) && (num_bits >= 8)) {
      --pIn_buf_cur;
      num_bits -= 8;
    }
  }
  r->m_num_bits = num_bits;
  r->m_bit_buf =
      bit_buf & (tinfl_bit_buf_t)((((mz_uint64)1) << num_bits) - (mz_uint64)1);
  r->m_dist = dist;
  r->m_counter = counter;
  r->m_num_extra = num_extra;
  r->m_dist_from_out_buf_start = dist_from_out_buf_start;
  *pIn_buf_size = pIn_buf_cur - pIn_buf_next;
  *pOut_buf_size = pOut_buf_cur - pOut_buf_next;
  if ((decomp_flags &
       (TINFL_FLAG_PARSE_ZLIB_HEADER | TINFL_FLAG_COMPUTE_ADLER32)) &&
      (status >= 0)) {
    const mz_uint8 *ptr = pOut_buf_next;
    size_t buf_len = *pOut_buf_size;
    mz_uint32 i, s1 = r->m_check_adler32 & 0xffff,
                 s2 = r->m_check_adler32 >> 16;
    size_t block_len = buf_len % 5552;
    while (buf_len) {
      for (i = 0; i + 7 < block_len; i += 8, ptr += 8) {
        s1 += ptr[0], s2 += s1;
        s1 += ptr[1], s2 += s1;
        s1 += ptr[2], s2 += s1;
        s1 += ptr[3], s2 += s1;
        s1 += ptr[4], s2 += s1;
        s1 += ptr[5], s2 += s1;
        s1 += ptr[6], s2 += s1;
        s1 += ptr[7], s2 += s1;
      }
      for (; i < block_len; ++i)
        s1 += *ptr++, s2 += s1;
      s1 %= 65521U, s2 %= 65521U;
      buf_len -= block_len;
      block_len = 5552;
    }
    r->m_check_adler32 = (s2 << 16) + s1;
    # 如果解压缩状态为完成，并且解压缩标志包含解析 ZLIB 头部，并且校验和不匹配
    if ((status == TINFL_STATUS_DONE) &&
        (decomp_flags & TINFL_FLAG_PARSE_ZLIB_HEADER) &&
        (r->m_check_adler32 != r->m_z_adler32))
      # 将解压缩状态设置为校验和不匹配
      status = TINFL_STATUS_ADLER32_MISMATCH;
  }
  # 返回解压缩状态
  return status;
/* 高级别辅助函数。*/
void *tinfl_decompress_mem_to_heap(const void *pSrc_buf, size_t src_buf_len,
                                   size_t *pOut_len, int flags) {
  // 创建解压缩器
  tinfl_decompressor decomp;
  // 初始化指针变量
  void *pBuf = NULL, *pNew_buf;
  // 初始化变量
  size_t src_buf_ofs = 0, out_buf_capacity = 0;
  *pOut_len = 0;
  // 初始化解压缩器
  tinfl_init(&decomp);
  // 无限循环
  for (;;) {
    // 计算源缓冲区和目标缓冲区的大小
    size_t src_buf_size = src_buf_len - src_buf_ofs,
           dst_buf_size = out_buf_capacity - *pOut_len, new_out_buf_capacity;
    // 解压缩数据
    tinfl_status status = tinfl_decompress(
        &decomp, (const mz_uint8 *)pSrc_buf + src_buf_ofs, &src_buf_size,
        (mz_uint8 *)pBuf, pBuf ? (mz_uint8 *)pBuf + *pOut_len : NULL,
        &dst_buf_size,
        (flags & ~TINFL_FLAG_HAS_MORE_INPUT) |
            TINFL_FLAG_USING_NON_WRAPPING_OUTPUT_BUF);
    // 处理解压缩状态
    if ((status < 0) || (status == TINFL_STATUS_NEEDS_MORE_INPUT)) {
      // 释放内存
      MZ_FREE(pBuf);
      *pOut_len = 0;
      return NULL;
    }
    // 更新源缓冲区偏移量和输出长度
    src_buf_ofs += src_buf_size;
    *pOut_len += dst_buf_size;
    // 如果解压缩完成，退出循环
    if (status == TINFL_STATUS_DONE)
      break;
    // 更新输出缓冲区容量
    new_out_buf_capacity = out_buf_capacity * 2;
    if (new_out_buf_capacity < 128)
      new_out_buf_capacity = 128;
    // 重新分配内存
    pNew_buf = MZ_REALLOC(pBuf, new_out_buf_capacity);
    if (!pNew_buf) {
      // 释放内存
      MZ_FREE(pBuf);
      *pOut_len = 0;
      return NULL;
    }
    pBuf = pNew_buf;
    out_buf_capacity = new_out_buf_capacity;
  }
  return pBuf;
}
# 将内存中的数据解压缩到内存中
size_t tinfl_decompress_mem_to_mem(void *pOut_buf, size_t out_buf_len,
                                   const void *pSrc_buf, size_t src_buf_len,
                                   int flags) {
  # 创建解压缩器对象
  tinfl_decompressor decomp;
  # 定义解压缩状态
  tinfl_status status;
  # 初始化解压缩器对象
  tinfl_init(&decomp);
  # 执行解压缩操作
  status =
      tinfl_decompress(&decomp, (const mz_uint8 *)pSrc_buf, &src_buf_len,
                       (mz_uint8 *)pOut_buf, (mz_uint8 *)pOut_buf, &out_buf_len,
                       (flags & ~TINFL_FLAG_HAS_MORE_INPUT) |
                           TINFL_FLAG_USING_NON_WRAPPING_OUTPUT_BUF);
  # 返回解压缩结果
  return (status != TINFL_STATUS_DONE) ? TINFL_DECOMPRESS_MEM_TO_MEM_FAILED
                                       : out_buf_len;
}

# 将内存中的数据解压缩到回调函数中
int tinfl_decompress_mem_to_callback(const void *pIn_buf, size_t *pIn_buf_size,
                                     tinfl_put_buf_func_ptr pPut_buf_func,
                                     void *pPut_buf_user, int flags) {
  # 定义结果变量
  int result = 0;
  # 创建解压缩器对象
  tinfl_decompressor decomp;
  # 分配解压缩字典内存
  mz_uint8 *pDict = (mz_uint8 *)MZ_MALLOC(TINFL_LZ_DICT_SIZE);
  # 定义输入缓冲区偏移量和字典偏移量
  size_t in_buf_ofs = 0, dict_ofs = 0;
  # 如果内存分配失败，则返回失败状态
  if (!pDict)
    return TINFL_STATUS_FAILED;
  # 初始化解压缩器对象
  tinfl_init(&decomp);
  # 循环执行解压缩操作
  for (;;) {
    # 计算输入缓冲区大小和目标缓冲区大小
    size_t in_buf_size = *pIn_buf_size - in_buf_ofs,
           dst_buf_size = TINFL_LZ_DICT_SIZE - dict_ofs;
    # 执行解压缩操作
    tinfl_status status =
        tinfl_decompress(&decomp, (const mz_uint8 *)pIn_buf + in_buf_ofs,
                         &in_buf_size, pDict, pDict + dict_ofs, &dst_buf_size,
                         (flags & ~(TINFL_FLAG_HAS_MORE_INPUT |
                                    TINFL_FLAG_USING_NON_WRAPPING_OUTPUT_BUF)));
    # 更新输入缓冲区偏移量
    in_buf_ofs += in_buf_size;
    # 如果回调函数返回 false，则中断循环
    if ((dst_buf_size) &&
        (!(*pPut_buf_func)(pDict + dict_ofs, (int)dst_buf_size, pPut_buf_user)))
      break;
    # 如果解压缩状态不是还有更多输出，则更新结果并中断循环
    if (status != TINFL_STATUS_HAS_MORE_OUTPUT) {
      result = (status == TINFL_STATUS_DONE);
      break;
    }
    # 更新字典偏移量
    dict_ofs = (dict_ofs + dst_buf_size) & (TINFL_LZ_DICT_SIZE - 1);
  }
  # 释放解压缩字典内存
  MZ_FREE(pDict);
  # 更新输入缓冲区大小并返回结果
  *pIn_buf_size = in_buf_ofs;
  return result;
}
#ifndef MINIZ_NO_MALLOC
// 如果未定义 MINIZ_NO_MALLOC，则分配一个 tinfl_decompressor 结构体并返回指针
tinfl_decompressor *tinfl_decompressor_alloc(void) {
  tinfl_decompressor *pDecomp =
      (tinfl_decompressor *)MZ_MALLOC(sizeof(tinfl_decompressor));
  // 如果分配成功，则初始化 tinfl_decompressor 结构体
  if (pDecomp)
    tinfl_init(pDecomp);
  return pDecomp;
}

// 释放 tinfl_decompressor 结构体的内存
void tinfl_decompressor_free(tinfl_decompressor *pDecomp) { MZ_FREE(pDecomp); }
#endif

#ifdef __cplusplus
}
#endif
/**************************************************************************
 *
 * 版权 2013-2014 RAD Game Tools 和 Valve Software
 * 版权 2010-2014 Rich Geldreich 和 Tenacious Software LLC
 * 版权 2016 Martin Raiber
 * 保留所有权利
 *
 * 特此免费授予任何获得本软件及相关文档文件（以下简称“软件”）副本的人
 * 无限制地处理软件的权利，包括但不限于使用、复制、修改、合并、发布、分发、许可和/或出售
 * 软件的副本，并允许软件的提供者
 * 这样做，但须遵守以下条件：
 *
 * 以上版权声明和此许可声明应包含在
 * 所有副本或实质部分的软件中
 *
 * 本软件按“原样”提供，不附带任何明示或暗示的保证，
 * 包括但不限于适销性、
 * 特定用途适用性和非侵权性的保证。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他
 * 责任承担责任，无论是在合同行为、侵权行为还是其他行为中产生的，
 * 由于软件或使用或其他方式使用软件而产生的，
 *
 **************************************************************************/

#ifndef MINIZ_NO_ARCHIVE_APIS

#ifdef __cplusplus
extern "C" {
#endif

/* ------------------- .ZIP archive reading */

#ifdef MINIZ_NO_STDIO
// 如果未定义 MINIZ_NO_STDIO，则定义 MZ_FILE 为 void *
#define MZ_FILE void *
#else
#include <sys/stat.h>

#if defined(_MSC_VER)
// 定义将字符串转换为宽字符字符串的函数
static wchar_t *str2wstr(const char *str) {
  // 计算字符串长度
  size_t len = strlen(str) + 1;
  // 分配内存存储宽字符字符串
  wchar_t *wstr = (wchar_t *)malloc(len * sizeof(wchar_t));
  // 将多字节字符串转换为宽字符字符串
  MultiByteToWideChar(CP_UTF8, 0, str, (int)(len * sizeof(char)), wstr, (int)len);
  // 返回宽字符字符串
  return wstr;
}

// 定义打开文件的函数
static FILE *mz_fopen(const char *pFilename, const char *pMode) {
  FILE *pFile = NULL;
  // 将文件名和打开模式转换为宽字符字符串
  wchar_t *wFilename = str2wstr(pFilename);
  wchar_t *wMode = str2wstr(pMode);

  // 根据宏定义选择文件打开方式
#ifdef ZIP_ENABLE_SHARABLE_FILE_OPEN
  pFile = _wfopen(wFilename, wMode);
#else
  _wfopen_s(&pFile, wFilename, wMode);
#endif

  // 释放内存
  free(wFilename);
  free(wMode);

  // 返回文件指针
  return pFile;
}

// 定义重新打开文件的函数
static FILE *mz_freopen(const char *pPath, const char *pMode, FILE *pStream) {
  FILE *pFile = NULL;
  int res = 0;

  // 将路径和打开模式转换为宽字符字符串
  wchar_t *wPath = str2wstr(pPath);
  wchar_t *wMode = str2wstr(pMode);

  // 根据宏定义选择文件重新打开方式
#ifdef ZIP_ENABLE_SHARABLE_FILE_OPEN
  pFile = _wfreopen(wPath, wMode, pStream);
#else
  res = _wfreopen_s(&pFile, wPath, wMode, pStream);
#endif

  // 释放内存
  free(wPath);
  free(wMode);

  // 根据宏定义判断是否出错
#ifndef ZIP_ENABLE_SHARABLE_FILE_OPEN
  if (res) {
    return NULL;
  }
#endif

  // 返回文件指针
  return pFile;
}

// 定义获取文件状态的函数
static int mz_stat(const char *pPath, struct _stat64 *buffer) {
  // 将路径转换为宽字符字符串
  wchar_t *wPath = str2wstr(pPath);
  // 获取文件状态
  int res = _wstat64(wPath, buffer);

  // 释放内存
  free(wPath);

  // 返回结果
  return res;
}

// 定义创建目录的函数
static int mz_mkdir(const char *pDirname) {
  // 将目录名转换为宽字符字符串
  wchar_t *wDirname = str2wstr(pDirname);
  // 创建目录
  int res = _wmkdir(wDirname);

  // 释放内存
  free(wDirname);

  // 返回结果
  return res;
}

// 定义宏，用于简化文件操作函数的调用
#define MZ_FOPEN mz_fopen
#define MZ_FCLOSE fclose
#define MZ_FREAD fread
#define MZ_FWRITE fwrite
#define MZ_FTELL64 _ftelli64
#define MZ_FSEEK64 _fseeki64
#define MZ_FILE_STAT_STRUCT _stat64
#define MZ_FILE_STAT mz_stat
#define MZ_FFLUSH fflush
#define MZ_FREOPEN mz_freopen
#define MZ_DELETE_FILE remove
#define MZ_MKDIR(d) mz_mkdir(d)

#elif defined(__MINGW32__) || defined(__MINGW64__)
#include <windows.h>
#ifndef MINIZ_NO_TIME
#include <sys/utime.h>
#endif

// 定义宏，用于简化文件操作函数的调用
#define MZ_FOPEN(f, m) fopen(f, m)
#define MZ_FCLOSE fclose
#ifdef __TINYC__
// 如果编译器是 Tiny C，则定义文件操作相关宏
#ifndef MINIZ_NO_TIME
#include <sys/utime.h>
#endif
// 定义文件操作相关宏
#define MZ_FOPEN(f, m) fopen(f, m)
#define MZ_FCLOSE fclose
#define MZ_FREAD fread
#define MZ_FWRITE fwrite
#define MZ_FTELL64 ftell
#define MZ_FSEEK64 fseek
#define MZ_FILE_STAT_STRUCT stat
#define MZ_FILE_STAT stat
#define MZ_FFLUSH fflush
#define MZ_FREOPEN(f, m, s) freopen(f, m, s)
#define MZ_DELETE_FILE remove
// 如果是 Windows 系统，则定义创建目录的宏
#if defined(_WIN32) || defined(_WIN64)
#define MZ_MKDIR(d) _mkdir(d)
// 否则定义创建目录的宏
#else
#define MZ_MKDIR(d) mkdir(d, 0755)
#endif

#elif defined(__USE_LARGEFILE64) /* gcc, clang */
// 如果使用 gcc 或 clang，并支持大文件操作，则定义文件操作相关宏
#ifndef MINIZ_NO_TIME
#include <utime.h>
#endif
// 定义文件操作相关宏
#define MZ_FOPEN(f, m) fopen64(f, m)
#define MZ_FCLOSE fclose
#define MZ_FREAD fread
#define MZ_FWRITE fwrite
#define MZ_FTELL64 ftello64
#define MZ_FSEEK64 fseeko64
#define MZ_FILE_STAT_STRUCT stat64
#define MZ_FILE_STAT stat64
#define MZ_FFLUSH fflush
#define MZ_FREOPEN(p, m, s) freopen64(p, m, s)
#define MZ_DELETE_FILE remove
#define MZ_MKDIR(d) mkdir(d, 0755)

#elif defined(__APPLE__)
// 如果是苹果系统，则定义文件操作相关宏
#ifndef MINIZ_NO_TIME
#include <utime.h>
#endif
// 定义文件操作相关宏
#define MZ_FOPEN(f, m) fopen(f, m)
#define MZ_FCLOSE fclose
#define MZ_FREAD fread
#define MZ_FWRITE fwrite
#define MZ_FTELL64 ftello
#define MZ_FSEEK64 fseeko
#define MZ_FILE_STAT_STRUCT stat
#define MZ_FILE_STAT stat
#define MZ_FFLUSH fflush
#define MZ_FREOPEN(p, m, s) freopen(p, m, s)
#define MZ_DELETE_FILE remove
#define MZ_MKDIR(d) mkdir(d, 0755)

#else
// 如果不是上述系统，则输出警告信息
#pragma message(                                                               \
        "Using fopen, ftello, fseeko, stat() etc. path for file I/O - this path may not support large files.")
#ifndef MINIZ_NO_TIME
#include <utime.h>
#endif
// 定义文件操作相关宏
#define MZ_FOPEN(f, m) fopen(f, m)
#ifndef MINIZ_H
// 如果未定义 MINIZ_H 宏，则包含整个头文件
#define MINIZ_H

// 定义 MZ_FCLOSE 宏，用于关闭文件
#define MZ_FCLOSE fclose
// 定义 MZ_FREAD 宏，用于读取文件
#define MZ_FREAD fread
// 定义 MZ_FWRITE 宏，用于写入文件
#define MZ_FWRITE fwrite

#ifdef __STRICT_ANSI__
// 如果定义了 __STRICT_ANSI__ 宏，则使用 ftell 函数
#define MZ_FTELL64 ftell
// 如果定义了 __STRICT_ANSI__ 宏，则使用 fseek 函数
#define MZ_FSEEK64 fseek
#else
// 如果未定义 __STRICT_ANSI__ 宏，则使用 ftello 函数
#define MZ_FTELL64 ftello
// 如果未定义 __STRICT_ANSI__ 宏，则使用 fseeko 函数
#define MZ_FSEEK64 fseeko
#endif

// 定义 MZ_FILE_STAT_STRUCT 宏，用于获取文件状态信息
#define MZ_FILE_STAT_STRUCT stat
// 定义 MZ_FILE_STAT 宏，用于获取文件状态信息
#define MZ_FILE_STAT stat
// 定义 MZ_FFLUSH 宏，用于刷新文件流
#define MZ_FFLUSH fflush
// 定义 MZ_FREOPEN 宏，用于重新打开文件
#define MZ_FREOPEN(f, m, s) freopen(f, m, s)
// 定义 MZ_DELETE_FILE 宏，用于删除文件
#define MZ_DELETE_FILE remove
// 定义 MZ_MKDIR 宏，用于创建目录
#define MZ_MKDIR(d) mkdir(d, 0755)

#endif /* #ifdef _MSC_VER */
#endif /* #ifdef MINIZ_NO_STDIO */

#ifndef CHMOD
// 如果未定义 CHMOD 宏，则定义 chmod 函数
// chmod 函数用于修改文件权限
// 成功时返回 0，失败时返回 -1 并设置 errno 错误码
// int chmod(const char *path, mode_t mode);
#define CHMOD(f, m) chmod(f, m)
#endif

// 定义 MZ_TOLOWER 宏，用于将字符转换为小写
#define MZ_TOLOWER(c) ((((c) >= 'A') && ((c) <= 'Z')) ? ((c) - 'A' + 'a') : (c))

/* Various ZIP archive enums. To completely avoid cross platform compiler
 * alignment and platform endian issues, miniz.c doesn't use structs for any of
 * this stuff. */

// 定义 mz_zip_array 结构体，用于表示 ZIP 数组
typedef struct {
  void *m_p;
  size_t m_size, m_capacity;
  mz_uint m_element_size;
} mz_zip_array;

// 定义 mz_zip_internal_state_tag 结构体，用于表示 ZIP 内部状态
struct mz_zip_internal_state_tag {
  // ZIP 中央目录数组
  mz_zip_array m_central_dir;
  // ZIP 中央目录偏移数组
  mz_zip_array m_central_dir_offsets;
  // 排序后的 ZIP 中央目录偏移数组

  // 初始化标志
  uint32_t m_init_flags;

  // ZIP64 标志
  mz_bool m_zip64;

  // ZIP64 扩展信息字段标志
  mz_bool m_zip64_has_extended_info_fields;

  // 文件指针
  MZ_FILE *m_pFile;
  // 文件存档起始偏移量
  mz_uint64 m_file_archive_start_ofs;

  // 内存指针
  void *m_pMem;
  // 内存大小
  size_t m_mem_size;
  // 内存容量
  size_t m_mem_capacity;
};

// 定义 MZ_ZIP_ARRAY_SET_ELEMENT_SIZE 宏，用于设置 ZIP 数组元素大小
#define MZ_ZIP_ARRAY_SET_ELEMENT_SIZE(array_ptr, element_size)                 \
  (array_ptr)->m_element_size = element_size
#if defined(DEBUG) || defined(_DEBUG)
// 如果定义了 DEBUG 或者 _DEBUG 宏，则定义一个内联函数用于检查数组范围
static MZ_FORCEINLINE mz_uint
mz_zip_array_range_check(const mz_zip_array *pArray, mz_uint index) {
  // 断言索引小于数组大小
  MZ_ASSERT(index < pArray->m_size);
  return index;
}
// 定义宏用于获取数组元素，包含范围检查
#define MZ_ZIP_ARRAY_ELEMENT(array_ptr, element_type, index)                   \
  ((element_type *)((array_ptr)                                                \
                        ->m_p))[mz_zip_array_range_check(array_ptr, index)]
#else
// 如果未定义 DEBUG 或者 _DEBUG 宏，则定义宏用于获取数组元素，不包含范围检查
#define MZ_ZIP_ARRAY_ELEMENT(array_ptr, element_type, index)                   \
  ((element_type *)((array_ptr)->m_p))[index]
#endif

// 初始化数组结构
static MZ_FORCEINLINE void mz_zip_array_init(mz_zip_array *pArray,
                                             mz_uint32 element_size) {
  // 使用零填充数组结构
  memset(pArray, 0, sizeof(mz_zip_array));
  // 设置数组元素大小
  pArray->m_element_size = element_size;
}

// 清空数组结构
static MZ_FORCEINLINE void mz_zip_array_clear(mz_zip_archive *pZip,
                                              mz_zip_array *pArray) {
  // 释放数组内存
  pZip->m_pFree(pZip->m_pAlloc_opaque, pArray->m_p);
  // 使用零填充数组结构
  memset(pArray, 0, sizeof(mz_zip_array));
}

// 确保数组容量足够
static mz_bool mz_zip_array_ensure_capacity(mz_zip_archive *pZip,
                                            mz_zip_array *pArray,
                                            size_t min_new_capacity,
                                            mz_uint growing) {
  void *pNew_p;
  size_t new_capacity = min_new_capacity;
  // 断言数组元素大小不为零
  MZ_ASSERT(pArray->m_element_size);
  // 如果当前容量已经满足需求，则返回真
  if (pArray->m_capacity >= min_new_capacity)
    return MZ_TRUE;
  // 如果需要扩容
  if (growing) {
    new_capacity = MZ_MAX(1, pArray->m_capacity);
    // 不断扩大容量直到满足需求
    while (new_capacity < min_new_capacity)
      new_capacity *= 2;
  }
  // 重新分配内存
  if (NULL == (pNew_p = pZip->m_pRealloc(pZip->m_pAlloc_opaque, pArray->m_p,
                                         pArray->m_element_size, new_capacity)))
    return MZ_FALSE;
  // 更新数组指针和容量
  pArray->m_p = pNew_p;
  pArray->m_capacity = new_capacity;
  return MZ_TRUE;
}
# 在数组中为新元素预留空间，如果需要则扩展数组容量
static MZ_FORCEINLINE mz_bool mz_zip_array_reserve(mz_zip_archive *pZip,
                                                   mz_zip_array *pArray,
                                                   size_t new_capacity,
                                                   mz_uint growing) {
  # 如果新容量大于数组当前容量
  if (new_capacity > pArray->m_capacity) {
    # 确保数组有足够的容量来存储新元素
    if (!mz_zip_array_ensure_capacity(pZip, pArray, new_capacity, growing))
      return MZ_FALSE;
  }
  return MZ_TRUE;
}

# 调整数组大小，如果需要则扩展数组容量
static MZ_FORCEINLINE mz_bool mz_zip_array_resize(mz_zip_archive *pZip,
                                                  mz_zip_array *pArray,
                                                  size_t new_size,
                                                  mz_uint growing) {
  # 如果新大小大于数组当前容量
  if (new_size > pArray->m_capacity) {
    # 确保数组有足够的容量来存储新大小的元素
    if (!mz_zip_array_ensure_capacity(pZip, pArray, new_size, growing))
      return MZ_FALSE;
  }
  # 更新数组的大小
  pArray->m_size = new_size;
  return MZ_TRUE;
}

# 确保数组有足够的空间来容纳 n 个元素
static MZ_FORCEINLINE mz_bool mz_zip_array_ensure_room(mz_zip_archive *pZip,
                                                       mz_zip_array *pArray,
                                                       size_t n) {
  return mz_zip_array_reserve(pZip, pArray, pArray->m_size + n, MZ_TRUE);
}

# 将元素添加到数组的末尾
static MZ_FORCEINLINE mz_bool mz_zip_array_push_back(mz_zip_archive *pZip,
                                                     mz_zip_array *pArray,
                                                     const void *pElements,
                                                     size_t n) {
  # 保存原始大小
  size_t orig_size = pArray->m_size;
  # 调整数组大小以容纳新元素
  if (!mz_zip_array_resize(pZip, pArray, orig_size + n, MZ_TRUE))
    return MZ_FALSE;
  # 如果有新元素，则将其复制到数组中
  if (n > 0)
    memcpy((mz_uint8 *)pArray->m_p + orig_size * pArray->m_element_size,
           pElements, n * pArray->m_element_size);
  return MZ_TRUE;
}

#ifndef MINIZ_NO_TIME
// 将 DOS 时间和日期转换为标准时间格式
static MZ_TIME_T mz_zip_dos_to_time_t(int dos_time, int dos_date) {
  // 初始化时间结构体
  struct tm tm;
  memset(&tm, 0, sizeof(tm));
  tm.tm_isdst = -1;
  // 计算年份
  tm.tm_year = ((dos_date >> 9) & 127) + 1980 - 1900;
  // 计算月份
  tm.tm_mon = ((dos_date >> 5) & 15) - 1;
  // 计算日期
  tm.tm_mday = dos_date & 31;
  // 计算小时
  tm.tm_hour = (dos_time >> 11) & 31;
  // 计算分钟
  tm.tm_min = (dos_time >> 5) & 63;
  // 计算秒数
  tm.tm_sec = (dos_time << 1) & 62;
  // 返回标准时间格式
  return mktime(&tm);
}

#ifndef MINIZ_NO_ARCHIVE_WRITING_APIS
// 将标准时间格式转换为 DOS 时间和日期
static void mz_zip_time_t_to_dos_time(MZ_TIME_T time, mz_uint16 *pDOS_time,
                                      mz_uint16 *pDOS_date) {
#ifdef _MSC_VER
  struct tm tm_struct;
  struct tm *tm = &tm_struct;
  // 使用 localtime_s 函数获取时间结构体
  errno_t err = localtime_s(tm, &time);
  // 如果出错，返回空值
  if (err) {
    *pDOS_date = 0;
    *pDOS_time = 0;
    return;
  }
#else
  // 使用 localtime 函数获取时间结构体
  struct tm *tm = localtime(&time);
#endif /* #ifdef _MSC_VER */

  // 计算 DOS 时间
  *pDOS_time = (mz_uint16)(((tm->tm_hour) << 11) + ((tm->tm_min) << 5) +
                           ((tm->tm_sec) >> 1));
  // 计算 DOS 日期
  *pDOS_date = (mz_uint16)(((tm->tm_year + 1900 - 1980) << 9) +
                           ((tm->tm_mon + 1) << 5) + tm->tm_mday);
}
#endif /* MINIZ_NO_ARCHIVE_WRITING_APIS */

#ifndef MINIZ_NO_STDIO
#ifndef MINIZ_NO_ARCHIVE_WRITING_APIS
// 获取文件的修改时间
static mz_bool mz_zip_get_file_modified_time(const char *pFilename,
                                             MZ_TIME_T *pTime) {
  // 文件状态结构体
  struct MZ_FILE_STAT_STRUCT file_stat;

  /* On Linux with x86 glibc, this call will fail on large files (I think >=
   * 0x80000000 bytes) unless you compiled with _LARGEFILE64_SOURCE. Argh. */
  // 获取文件状态
  if (MZ_FILE_STAT(pFilename, &file_stat) != 0)
    return MZ_FALSE;

  // 将修改时间赋值给 pTime
  *pTime = file_stat.st_mtime;

  return MZ_TRUE;
}
#endif /* #ifndef MINIZ_NO_ARCHIVE_WRITING_APIS*/
// 设置文件的访问时间和修改时间
static mz_bool mz_zip_set_file_times(const char *pFilename,
                                     MZ_TIME_T access_time,
                                     MZ_TIME_T modified_time) {
  // 创建 utimbuf 结构体
  struct utimbuf t;

  // 将 t 结构体清零
  memset(&t, 0, sizeof(t));
  // 设置访问时间和修改时间
  t.actime = access_time;
  t.modtime = modified_time;

  // 调用 utime 函数设置文件时间
  return !utime(pFilename, &t);
}
#endif /* #ifndef MINIZ_NO_STDIO */
#endif /* #ifndef MINIZ_NO_TIME */

// 设置 ZIP 归档的错误信息
static MZ_FORCEINLINE mz_bool mz_zip_set_error(mz_zip_archive *pZip,
                                               mz_zip_error err_num) {
  // 如果 ZIP 归档对象存在，则设置错误信息
  if (pZip)
    pZip->m_last_error = err_num;
  return MZ_FALSE;
}

// 初始化 ZIP 读取器内部
static mz_bool mz_zip_reader_init_internal(mz_zip_archive *pZip,
                                           mz_uint flags) {
  // 忽略 flags 参数
  (void)flags;
  // 检查 ZIP 归档对象是否有效
  if ((!pZip) || (pZip->m_pState) || (pZip->m_zip_mode != MZ_ZIP_MODE_INVALID))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 设置默认内存分配、释放和重新分配函数
  if (!pZip->m_pAlloc)
    pZip->m_pAlloc = miniz_def_alloc_func;
  if (!pZip->m_pFree)
    pZip->m_pFree = miniz_def_free_func;
  if (!pZip->m_pRealloc)
    pZip->m_pRealloc = miniz_def_realloc_func;

  // 初始化 ZIP 归档对象的各个属性
  pZip->m_archive_size = 0;
  pZip->m_central_directory_file_ofs = 0;
  pZip->m_total_files = 0;
  pZip->m_last_error = MZ_ZIP_NO_ERROR;

  // 分配内存给 ZIP 内部状态对象
  if (NULL == (pZip->m_pState = (mz_zip_internal_state *)pZip->m_pAlloc(
                   pZip->m_pAlloc_opaque, 1, sizeof(mz_zip_internal_state))))
    # 设置 ZIP 对象的错误状态为内存分配失败
    return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

  # 将 ZIP 对象的内部状态清零
  memset(pZip->m_pState, 0, sizeof(mz_zip_internal_state));
  # 设置中央目录的元素大小
  MZ_ZIP_ARRAY_SET_ELEMENT_SIZE(&pZip->m_pState->m_central_dir,
                                sizeof(mz_uint8));
  # 设置中央目录偏移的元素大小
  MZ_ZIP_ARRAY_SET_ELEMENT_SIZE(&pZip->m_pState->m_central_dir_offsets,
                                sizeof(mz_uint32));
  # 设置排序后的中央目录偏移的元素大小
  MZ_ZIP_ARRAY_SET_ELEMENT_SIZE(&pZip->m_pState->m_sorted_central_dir_offsets,
                                sizeof(mz_uint32));
  # 设置 ZIP 对象的初始化标志
  pZip->m_pState->m_init_flags = flags;
  # 设置 ZIP 对象不使用 ZIP64 格式
  pZip->m_pState->m_zip64 = MZ_FALSE;
  # 设置 ZIP 对象的 ZIP64 扩展信息字段为假
  pZip->m_pState->m_zip64_has_extended_info_fields = MZ_FALSE;

  # 设置 ZIP 对象的模式为读取模式
  pZip->m_zip_mode = MZ_ZIP_MODE_READING;

  # 返回操作成功
  return MZ_TRUE;
# 定义一个静态内联函数，用于比较两个文件名的大小写不敏感排序
static MZ_FORCEINLINE mz_bool
mz_zip_reader_filename_less(const mz_zip_array *pCentral_dir_array,
                            const mz_zip_array *pCentral_dir_offsets,
                            mz_uint l_index, mz_uint r_index) {
  # 获取左边文件名的指针和长度
  const mz_uint8 *pL = &MZ_ZIP_ARRAY_ELEMENT(
                     pCentral_dir_array, mz_uint8,
                     MZ_ZIP_ARRAY_ELEMENT(pCentral_dir_offsets, mz_uint32,
                                          l_index)),
                 *pE;
  # 获取右边文件名的指针和长度
  const mz_uint8 *pR = &MZ_ZIP_ARRAY_ELEMENT(
      pCentral_dir_array, mz_uint8,
      MZ_ZIP_ARRAY_ELEMENT(pCentral_dir_offsets, mz_uint32, r_index));
  # 获取左右文件名的长度
  mz_uint l_len = MZ_READ_LE16(pL + MZ_ZIP_CDH_FILENAME_LEN_OFS),
          r_len = MZ_READ_LE16(pR + MZ_ZIP_CDH_FILENAME_LEN_OFS);
  mz_uint8 l = 0, r = 0;
  # 移动指针到文件名的起始位置
  pL += MZ_ZIP_CENTRAL_DIR_HEADER_SIZE;
  pR += MZ_ZIP_CENTRAL_DIR_HEADER_SIZE;
  # 比较两个文件名的大小写不敏感排序
  pE = pL + MZ_MIN(l_len, r_len);
  while (pL < pE) {
    if ((l = MZ_TOLOWER(*pL)) != (r = MZ_TOLOWER(*pR)))
      break;
    pL++;
    pR++;
  }
  # 返回比较结果
  return (pL == pE) ? (l_len < r_len) : (l < r);
}

# 定义一个宏，用于交换两个32位整数的值
#define MZ_SWAP_UINT32(a, b)                                                   \
  do {                                                                         \
    mz_uint32 t = a;                                                           \
    a = b;                                                                     \
    b = t;                                                                     \
  }                                                                            \
  MZ_MACRO_END

# 堆排序小写文件名，用于加速 mz_zip_reader_locate_file() 中的普通中央目录搜索
static void
# 根据文件名对 ZIP 文件的中央目录偏移量进行排序
mz_zip_reader_sort_central_dir_offsets_by_filename(mz_zip_archive *pZip) {
  # 获取 ZIP 文件的内部状态
  mz_zip_internal_state *pState = pZip->m_pState;
  # 获取中央目录偏移量数组
  const mz_zip_array *pCentral_dir_offsets = &pState->m_central_dir_offsets;
  # 获取中央目录数组
  const mz_zip_array *pCentral_dir = &pState->m_central_dir;
  # 指向索引数组的指针
  mz_uint32 *pIndices;
  # 起始和结束索引
  mz_uint32 start, end;
  # 获取文件总数
  const mz_uint32 size = pZip->m_total_files;

  # 如果文件总数小于等于1，则直接返回
  if (size <= 1U)
    return;

  # 初始化索引数组
  pIndices = &MZ_ZIP_ARRAY_ELEMENT(&pState->m_sorted_central_dir_offsets,
                                   mz_uint32, 0);

  # 初始化起始索引
  start = (size - 2U) >> 1U;
  # 循环直到排序完成
  for (;;) {
    mz_uint64 child, root = start;
    for (;;) {
      # 计算子节点索引
      if ((child = (root << 1U) + 1U) >= size)
        break;
      # 比较文件名并调整子节点索引
      child += (((child + 1U) < size) &&
                (mz_zip_reader_filename_less(pCentral_dir, pCentral_dir_offsets,
                                             pIndices[child],
                                             pIndices[child + 1U])));
      # 如果根节点小于子节点，则交换位置
      if (!mz_zip_reader_filename_less(pCentral_dir, pCentral_dir_offsets,
                                       pIndices[root], pIndices[child]))
        break;
      MZ_SWAP_UINT32(pIndices[root], pIndices[child]);
      root = child;
    }
    # 更新起始索引
    if (!start)
      break;
    start--;
  }

  # 初始化结束索引
  end = size - 1;
  # 循环直到排序完成
  while (end > 0) {
    mz_uint64 child, root = 0;
    # 将根节点与最后一个节点交换位置
    MZ_SWAP_UINT32(pIndices[end], pIndices[0]);
    for (;;) {
      # 计算子节点索引
      if ((child = (root << 1U) + 1U) >= end)
        break;
      # 比较文件名并调整子节点索引
      child +=
          (((child + 1U) < end) &&
           mz_zip_reader_filename_less(pCentral_dir, pCentral_dir_offsets,
                                       pIndices[child], pIndices[child + 1U]));
      # 如果根节点小于子节点，则交换位置
      if (!mz_zip_reader_filename_less(pCentral_dir, pCentral_dir_offsets,
                                       pIndices[root], pIndices[child]))
        break;
      MZ_SWAP_UINT32(pIndices[root], pIndices[child]);
      root = child;
    }
    end--;
  }
}
static mz_bool mz_zip_reader_locate_header_sig(mz_zip_archive *pZip,
                                               mz_uint32 record_sig,
                                               mz_uint32 record_size,
                                               mz_int64 *pOfs) {
  mz_int64 cur_file_ofs; // 当前文件偏移量
  mz_uint32 buf_u32[4096 / sizeof(mz_uint32)]; // 缓冲区，用于读取文件内容
  mz_uint8 *pBuf = (mz_uint8 *)buf_u32; // 将缓冲区转换为字节类型指针

  /* Basic sanity checks - reject files which are too small */
  // 基本的健全性检查 - 拒绝太小的文件
  if (pZip->m_archive_size < record_size)
    return MZ_FALSE;

  /* Find the record by scanning the file from the end towards the beginning. */
  // 从文件末尾向开头扫描文件，查找记录
  cur_file_ofs =
      MZ_MAX((mz_int64)pZip->m_archive_size - (mz_int64)sizeof(buf_u32), 0);
  for (;;) {
    int i,
        n = (int)MZ_MIN(sizeof(buf_u32), pZip->m_archive_size - cur_file_ofs);

    if (pZip->m_pRead(pZip->m_pIO_opaque, cur_file_ofs, pBuf, n) != (mz_uint)n)
      return MZ_FALSE;

    for (i = n - 4; i >= 0; --i) {
      mz_uint s = MZ_READ_LE32(pBuf + i); // 从缓冲区中读取一个32位的小端序整数
      if (s == record_sig) {
        if ((pZip->m_archive_size - (cur_file_ofs + i)) >= record_size)
          break;
      }
    }

    if (i >= 0) {
      cur_file_ofs += i;
      break;
    }

    /* Give up if we've searched the entire file, or we've gone back "too far"
     * (~64kb) */
    // 如果已经搜索整个文件，或者回退得太远（~64kb），则放弃
    if ((!cur_file_ofs) || ((pZip->m_archive_size - cur_file_ofs) >=
                            (MZ_UINT16_MAX + record_size)))
      return MZ_FALSE;

    cur_file_ofs = MZ_MAX(cur_file_ofs - (sizeof(buf_u32) - 3), 0);
  }

  *pOfs = cur_file_ofs; // 将找到的记录的偏移量赋值给pOfs
  return MZ_TRUE; // 返回找到记录的标志
}
  // 初始化变量，用于存储中央目录的信息
  mz_uint cdir_size = 0, cdir_entries_on_this_disk = 0, num_this_disk = 0,
          cdir_disk_index = 0;
  mz_uint64 cdir_ofs = 0;
  mz_int64 cur_file_ofs = 0;
  const mz_uint8 *p;

  // 创建缓冲区用于读取数据
  mz_uint32 buf_u32[4096 / sizeof(mz_uint32)];
  mz_uint8 *pBuf = (mz_uint8 *)buf_u32;
  // 检查是否需要对中央目录进行排序
  mz_bool sort_central_dir =
      ((flags & MZ_ZIP_FLAG_DO_NOT_SORT_CENTRAL_DIRECTORY) == 0);
  // 创建用于存储 ZIP64 结尾中央目录定位器的缓冲区
  mz_uint32 zip64_end_of_central_dir_locator_u32
      [(MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIZE + sizeof(mz_uint32) - 1) /
       sizeof(mz_uint32)];
  mz_uint8 *pZip64_locator = (mz_uint8 *)zip64_end_of_central_dir_locator_u32;

  // 创建用于存储 ZIP64 结尾中央目录头部的缓冲区
  mz_uint32 zip64_end_of_central_dir_header_u32
      [(MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE + sizeof(mz_uint32) - 1) /
       sizeof(mz_uint32)];
  mz_uint8 *pZip64_end_of_central_dir =
      (mz_uint8 *)zip64_end_of_central_dir_header_u32;

  // 初始化变量，用于存储 ZIP64 结尾中央目录的偏移量
  mz_uint64 zip64_end_of_central_dir_ofs = 0;

  /* 基本的合法性检查 - 拒绝太小的文件，并检查文件的前4个字节以确保存在本地头部 */
  if (pZip->m_archive_size < MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE)
    return mz_zip_set_error(pZip, MZ_ZIP_NOT_AN_ARCHIVE);

  // 定位并检查中央目录头部的签名
  if (!mz_zip_reader_locate_header_sig(
          pZip, MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIG,
          MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE, &cur_file_ofs))
    return mz_zip_set_error(pZip, MZ_ZIP_FAILED_FINDING_CENTRAL_DIR);

  /* 读取并验证中央目录记录的结束部分 */
  if (pZip->m_pRead(pZip->m_pIO_opaque, cur_file_ofs, pBuf,
                    MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE) !=
      MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE)
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);

  // 检查中央目录头部的签名是否正确
  if (MZ_READ_LE32(pBuf + MZ_ZIP_ECDH_SIG_OFS) !=
      MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIG)
    // 如果当前文件偏移量大于 ZIP64 结尾中央目录定位器和 ZIP64 结尾中央目录头的大小之和
    return mz_zip_set_error(pZip, MZ_ZIP_NOT_AN_ARCHIVE);

  // 如果满足条件则执行以下代码块
  if (cur_file_ofs >= (MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIZE +
                       MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE)) {
    // 从当前文件偏移量开始读取 ZIP64 结尾中央目录定位器的数据
    if (pZip->m_pRead(pZip->m_pIO_opaque,
                      cur_file_ofs - MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIZE,
                      pZip64_locator,
                      MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIZE) ==
        MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIZE) {
      // 如果 ZIP64 结尾中央目录定位器的标识符符合预期值
      if (MZ_READ_LE32(pZip64_locator + MZ_ZIP64_ECDL_SIG_OFS) ==
          MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIG) {
        // 读取 ZIP64 结尾中央目录的偏移量
        zip64_end_of_central_dir_ofs = MZ_READ_LE64(
            pZip64_locator + MZ_ZIP64_ECDL_REL_OFS_TO_ZIP64_ECDR_OFS);
        // 如果 ZIP64 结尾中央目录的偏移量超出了文件大小减去 ZIP64 结尾中央目录头的大小
        if (zip64_end_of_central_dir_ofs >
            (pZip->m_archive_size - MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE))
          return mz_zip_set_error(pZip, MZ_ZIP_NOT_AN_ARCHIVE);

        // 从 ZIP64 结尾中央目录的偏移量开始读取 ZIP64 结尾中央目录头的数据
        if (pZip->m_pRead(pZip->m_pIO_opaque, zip64_end_of_central_dir_ofs,
                          pZip64_end_of_central_dir,
                          MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE) ==
            MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE) {
          // 如果 ZIP64 结尾中央目录头的标识符符合预期值
          if (MZ_READ_LE32(pZip64_end_of_central_dir + MZ_ZIP64_ECDH_SIG_OFS) ==
              MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIG) {
            // 设置 ZIP64 标志为真
            pZip->m_pState->m_zip64 = MZ_TRUE;
          }
        }
      }
    }
  }

  // 读取总文件数
  pZip->m_total_files = MZ_READ_LE16(pBuf + MZ_ZIP_ECDH_CDIR_TOTAL_ENTRIES_OFS);
  // 读取当前磁盘上的中央目录条目数
  cdir_entries_on_this_disk =
      MZ_READ_LE16(pBuf + MZ_ZIP_ECDH_CDIR_NUM_ENTRIES_ON_DISK_OFS);
  // 读取当前磁盘编号
  num_this_disk = MZ_READ_LE16(pBuf + MZ_ZIP_ECDH_NUM_THIS_DISK_OFS);
  // 读取中央目录所在磁盘编号
  cdir_disk_index = MZ_READ_LE16(pBuf + MZ_ZIP_ECDH_NUM_DISK_CDIR_OFS);
  // 读取中央目录大小
  cdir_size = MZ_READ_LE32(pBuf + MZ_ZIP_ECDH_CDIR_SIZE_OFS);
  // 读取中央目录偏移量
  cdir_ofs = MZ_READ_LE32(pBuf + MZ_ZIP_ECDH_CDIR_OFS_OFS);

  // 如果 ZIP64 标志为真
  if (pZip->m_pState->m_zip64) {
    // 读取 ZIP64 总磁盘数
    mz_uint32 zip64_total_num_of_disks =
        MZ_READ_LE32(pZip64_locator + MZ_ZIP64_ECDL_TOTAL_NUMBER_OF_DISKS_OFS);
    // 读取 ZIP64 中央目录结束记录中的总条目数
    mz_uint64 zip64_cdir_total_entries = MZ_READ_LE64(
        pZip64_end_of_central_dir + MZ_ZIP64_ECDH_CDIR_TOTAL_ENTRIES_OFS);
    // 读取 ZIP64 中央目录结束记录中的本磁盘上的总条目数
    mz_uint64 zip64_cdir_total_entries_on_this_disk = MZ_READ_LE64(
        pZip64_end_of_central_dir + MZ_ZIP64_ECDH_CDIR_NUM_ENTRIES_ON_DISK_OFS);
    // 读取 ZIP64 中央目录结束记录中的结束记录大小
    mz_uint64 zip64_size_of_end_of_central_dir_record = MZ_READ_LE64(
        pZip64_end_of_central_dir + MZ_ZIP64_ECDH_SIZE_OF_RECORD_OFS);
    // 读取 ZIP64 中央目录结束记录中的中央目录大小
    mz_uint64 zip64_size_of_central_directory =
        MZ_READ_LE64(pZip64_end_of_central_dir + MZ_ZIP64_ECDH_CDIR_SIZE_OFS);

    // 如果结束记录大小小于规定的最小值，则返回错误
    if (zip64_size_of_end_of_central_dir_record <
        (MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE - 12))
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

    // 如果总磁盘数不为1，则返回错误
    if (zip64_total_num_of_disks != 1U)
      return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_MULTIDISK);

    /* 检查 miniz 的实际限制 */
    // 如果总条目数超过 mz_uint32 的最大值，则返回错误
    if (zip64_cdir_total_entries > MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);

    // 将总文件数设置为 ZIP64 中央目录结束记录中的总条目数
    pZip->m_total_files = (mz_uint32)zip64_cdir_total_entries;

    // 如果本磁盘上的总条目数超过 mz_uint32 的最大值，则返回错误
    if (zip64_cdir_total_entries_on_this_disk > MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);

    // 将本磁盘上的条目数设置为 ZIP64 中央目录结束记录中的本磁盘上的总条目数
    cdir_entries_on_this_disk =
        (mz_uint32)zip64_cdir_total_entries_on_this_disk;

    /* 检查 miniz 的当前实际限制（抱歉，这应该足够处理数百万个文件） */
    // 如果中央目录大小超过 mz_uint32 的最大值，则返回错误
    if (zip64_size_of_central_directory > MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_CDIR_SIZE);

    // 将中央目录大小设置为 ZIP64 中央目录结束记录中的中央目录大小
    cdir_size = (mz_uint32)zip64_size_of_central_directory;

    // 读取 ZIP64 中央目录结束记录中的本磁盘上的编号
    num_this_disk = MZ_READ_LE32(pZip64_end_of_central_dir +
                                 MZ_ZIP64_ECDH_NUM_THIS_DISK_OFS);

    // 读取 ZIP64 中央目录结束记录中的中央目录所在磁盘编号
    cdir_disk_index = MZ_READ_LE32(pZip64_end_of_central_dir +
                                   MZ_ZIP64_ECDH_NUM_DISK_CDIR_OFS);

    // 读取 ZIP64 中央目录结束记录中的中央目录偏移量
    cdir_ofs =
        MZ_READ_LE64(pZip64_end_of_central_dir + MZ_ZIP64_ECDH_CDIR_OFS_OFS);
  }

  // 如果总文件数不等于本磁盘上的条目数
  if (pZip->m_total_files != cdir_entries_on_this_disk)
    // 如果 ZIP 文件不支持多磁盘，则设置错误并返回
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_MULTIDISK);

  // 如果当前磁盘号和中央目录磁盘索引不为零，并且不是第一个磁盘或中央目录磁盘索引不为1，则设置错误并返回
  if (((num_this_disk | cdir_disk_index) != 0) &&
      ((num_this_disk != 1) || (cdir_disk_index != 1)))
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_MULTIDISK);

  // 如果中央目录大小小于总文件数乘以中央目录头大小，则设置错误并返回
  if (cdir_size < pZip->m_total_files * MZ_ZIP_CENTRAL_DIR_HEADER_SIZE)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 如果中央目录偏移加上中央目录大小大于存档大小，则设置错误并返回
  if ((cdir_ofs + (mz_uint64)cdir_size) > pZip->m_archive_size)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 设置中央目录文件偏移
  pZip->m_central_directory_file_ofs = cdir_ofs;

  // 如果总文件数不为零
  if (pZip->m_total_files) {
    mz_uint i, n;
    /* 读取整个中央目录到堆块，并分配另一个堆块来保存未排序的中央目录文件记录偏移，可能还有另一个堆块来保存排序后的索引 */
    if ((!mz_zip_array_resize(pZip, &pZip->m_pState->m_central_dir, cdir_size,
                              MZ_FALSE)) ||
        (!mz_zip_array_resize(pZip, &pZip->m_pState->m_central_dir_offsets,
                              pZip->m_total_files, MZ_FALSE)))
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

    // 如果需要对中央目录进行排序
    if (sort_central_dir) {
      if (!mz_zip_array_resize(pZip,
                               &pZip->m_pState->m_sorted_central_dir_offsets,
                               pZip->m_total_files, MZ_FALSE))
        return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }

    // 如果从存储器中读取中央目录失败，则设置错误并返回
    if (pZip->m_pRead(pZip->m_pIO_opaque, cdir_ofs,
                      pZip->m_pState->m_central_dir.m_p,
                      cdir_size) != cdir_size)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);

    /* 现在创建一个中央目录文件记录的索引，对每个记录进行基本的完整性检查 */
    p = (const mz_uint8 *)pZip->m_pState->m_central_dir.m_p;
    }
  }

  // 如果需要对中央目录进行排序，则调用函数进行排序
  if (sort_central_dir)
    mz_zip_reader_sort_central_dir_offsets_by_filename(pZip);

  // 返回成功
  return MZ_TRUE;
// 将 mz_zip_archive 结构体清零
void mz_zip_zero_struct(mz_zip_archive *pZip) {
  // 如果指针不为空，则清零结构体
  if (pZip)
    MZ_CLEAR_OBJ(*pZip);
}

// 结束 ZIP 读取操作的内部函数
static mz_bool mz_zip_reader_end_internal(mz_zip_archive *pZip,
                                          mz_bool set_last_error) {
  mz_bool status = MZ_TRUE;

  // 如果 pZip 为空，则返回假
  if (!pZip)
    return MZ_FALSE;

  // 检查 pZip 的状态和模式是否正确
  if ((!pZip->m_pState) || (!pZip->m_pAlloc) || (!pZip->m_pFree) ||
      (pZip->m_zip_mode != MZ_ZIP_MODE_READING)) {
    // 如果设置了最后错误标志，则将错误码设置为无效参数
    if (set_last_error)
      pZip->m_last_error = MZ_ZIP_INVALID_PARAMETER;

    return MZ_FALSE;
  }

  // 清理 pZip 的状态
  if (pZip->m_pState) {
    mz_zip_internal_state *pState = pZip->m_pState;
    pZip->m_pState = NULL;

    // 清理中央目录相关数据
    mz_zip_array_clear(pZip, &pState->m_central_dir);
    mz_zip_array_clear(pZip, &pState->m_central_dir_offsets);
    mz_zip_array_clear(pZip, &pState->m_sorted_central_dir_offsets);

#ifndef MINIZ_NO_STDIO
    // 如果有文件指针，则关闭文件
    if (pState->m_pFile) {
      if (pZip->m_zip_type == MZ_ZIP_TYPE_FILE) {
        if (MZ_FCLOSE(pState->m_pFile) == EOF) {
          // 如果关闭文件失败，则设置错误码为文件关闭失败
          if (set_last_error)
            pZip->m_last_error = MZ_ZIP_FILE_CLOSE_FAILED;
          status = MZ_FALSE;
        }
      }
      pState->m_pFile = NULL;
    }
#endif /* #ifndef MINIZ_NO_STDIO */

    // 释放状态内存
    pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
  }
  pZip->m_zip_mode = MZ_ZIP_MODE_INVALID;

  return status;
}

// 结束 ZIP 读取操作
mz_bool mz_zip_reader_end(mz_zip_archive *pZip) {
  return mz_zip_reader_end_internal(pZip, MZ_TRUE);
}

// 初始化 ZIP 读取操作
mz_bool mz_zip_reader_init(mz_zip_archive *pZip, mz_uint64 size,
                           mz_uint flags) {
  // 如果 pZip 为空或者没有读取函数，则返回无效参数错误
  if ((!pZip) || (!pZip->m_pRead))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 初始化 ZIP 读取操作的内部函数
  if (!mz_zip_reader_init_internal(pZip, flags))
    return MZ_FALSE;

  // 设置 ZIP 类型为用户定义类型，设置归档大小
  pZip->m_zip_type = MZ_ZIP_TYPE_USER;
  pZip->m_archive_size = size;

  // 读取中央目录信息
  if (!mz_zip_reader_read_central_dir(pZip, flags)) {
    mz_zip_reader_end_internal(pZip, MZ_FALSE);
    return MZ_FALSE;
  }

  return MZ_TRUE;
}
// 从内存中读取 ZIP 文件内容的回调函数
static size_t mz_zip_mem_read_func(void *pOpaque, mz_uint64 file_ofs,
                                   void *pBuf, size_t n) {
  // 将传入的指针转换为 ZIP 归档对象
  mz_zip_archive *pZip = (mz_zip_archive *)pOpaque;
  // 计算需要读取的数据大小
  size_t s = (file_ofs >= pZip->m_archive_size)
                 ? 0
                 : (size_t)MZ_MIN(pZip->m_archive_size - file_ofs, n);
  // 从内存中复制数据到缓冲区
  memcpy(pBuf, (const mz_uint8 *)pZip->m_pState->m_pMem + file_ofs, s);
  // 返回实际读取的数据大小
  return s;
}

// 初始化从内存中读取 ZIP 文件的读取器
mz_bool mz_zip_reader_init_mem(mz_zip_archive *pZip, const void *pMem,
                               size_t size, mz_uint flags) {
  // 检查传入的内存指针是否为空
  if (!pMem)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 检查传入的内存大小是否小于 ZIP 文件的结束中央目录头部大小
  if (size < MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE)
    return mz_zip_set_error(pZip, MZ_ZIP_NOT_AN_ARCHIVE);

  // 初始化 ZIP 读取器内部
  if (!mz_zip_reader_init_internal(pZip, flags))
    return MZ_FALSE;

  // 设置 ZIP 类型为内存类型
  pZip->m_zip_type = MZ_ZIP_TYPE_MEMORY;
  // 设置 ZIP 归档的大小
  pZip->m_archive_size = size;
  // 设置 ZIP 读取器的读取函数为从内存中读取的函数
  pZip->m_pRead = mz_zip_mem_read_func;
  // 设置 ZIP 读取器的 IO 透明指针
  pZip->m_pIO_opaque = pZip;
  pZip->m_pNeeds_keepalive = NULL;

  // 根据不同的编译环境设置内存指针
#ifdef __cplusplus
  pZip->m_pState->m_pMem = const_cast<void *>(pMem);
#else
  pZip->m_pState->m_pMem = (void *)pMem;
#endif

  // 设置 ZIP 状态中的内存大小
  pZip->m_pState->m_mem_size = size;

  // 读取 ZIP 文件的中央目录
  if (!mz_zip_reader_read_central_dir(pZip, flags)) {
    mz_zip_reader_end_internal(pZip, MZ_FALSE);
    return MZ_FALSE;
  }

  return MZ_TRUE;
}

// 从文件中读取 ZIP 文件内容的回调函数
#ifndef MINIZ_NO_STDIO
static size_t mz_zip_file_read_func(void *pOpaque, mz_uint64 file_ofs,
                                    void *pBuf, size_t n) {
  // 将传入的指针转换为 ZIP 归档对象
  mz_zip_archive *pZip = (mz_zip_archive *)pOpaque;
  // 获取当前文件指针的位置
  mz_int64 cur_ofs = MZ_FTELL64(pZip->m_pState->m_pFile);

  // 计算文件偏移量
  file_ofs += pZip->m_pState->m_file_archive_start_ofs;

  // 检查文件偏移量是否有效，如果不是则返回 0
  if (((mz_int64)file_ofs < 0) ||
      (((cur_ofs != (mz_int64)file_ofs)) &&
       (MZ_FSEEK64(pZip->m_pState->m_pFile, (mz_int64)file_ofs, SEEK_SET))))
    return 0;

  // 从文件中读取数据到缓冲区
  return MZ_FREAD(pBuf, 1, n, pZip->m_pState->m_pFile);
}
# 初始化 ZIP 归档读取器，从文件中读取 ZIP 归档内容
mz_bool mz_zip_reader_init_file(mz_zip_archive *pZip, const char *pFilename,
                                mz_uint32 flags) {
  # 调用带有额外参数的初始化函数
  return mz_zip_reader_init_file_v2(pZip, pFilename, flags, 0, 0);
}

# 初始化 ZIP 归档读取器，从文件中读取 ZIP 归档内容，带有额外参数
mz_bool mz_zip_reader_init_file_v2(mz_zip_archive *pZip, const char *pFilename,
                                   mz_uint flags, mz_uint64 file_start_ofs,
                                   mz_uint64 archive_size) {
  mz_uint64 file_size;
  MZ_FILE *pFile;

  # 检查参数是否有效
  if ((!pZip) || (!pFilename) ||
      ((archive_size) &&
       (archive_size < MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE)))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  # 打开文件进行读取
  pFile = MZ_FOPEN(pFilename, "rb");
  if (!pFile)
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_OPEN_FAILED);

  # 获取文件大小
  file_size = archive_size;
  if (!file_size) {
    if (MZ_FSEEK64(pFile, 0, SEEK_END)) {
      MZ_FCLOSE(pFile);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_SEEK_FAILED);
    }

    file_size = MZ_FTELL64(pFile);
  }

  # 检查文件大小是否符合 ZIP 归档的最小要求
  if (file_size < MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE) {
    MZ_FCLOSE(pFile);
    return mz_zip_set_error(pZip, MZ_ZIP_NOT_AN_ARCHIVE);
  }

  # 初始化 ZIP 归档读取器内部状态
  if (!mz_zip_reader_init_internal(pZip, flags)) {
    MZ_FCLOSE(pFile);
    return MZ_FALSE;
  }

  # 设置 ZIP 归档读取器的相关属性
  pZip->m_zip_type = MZ_ZIP_TYPE_FILE;
  pZip->m_pRead = mz_zip_file_read_func;
  pZip->m_pIO_opaque = pZip;
  pZip->m_pState->m_pFile = pFile;
  pZip->m_archive_size = file_size;
  pZip->m_pState->m_file_archive_start_ofs = file_start_ofs;

  # 读取 ZIP 归档的中央目录信息
  if (!mz_zip_reader_read_central_dir(pZip, flags)) {
    mz_zip_reader_end_internal(pZip, MZ_FALSE);
    return MZ_FALSE;
  }

  return MZ_TRUE;
}
# 初始化 ZIP 归档读取器，从文件中读取 ZIP 归档数据
mz_bool mz_zip_reader_init_file_v2_rpb(mz_zip_archive *pZip,
                                       const char *pFilename, mz_uint flags,
                                       mz_uint64 file_start_ofs,
                                       mz_uint64 archive_size) {
  mz_uint64 file_size;
  MZ_FILE *pFile;

  # 检查参数是否有效，如果无效则返回错误
  if ((!pZip) || (!pFilename) ||
      ((archive_size) &&
       (archive_size < MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE)))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  # 打开文件以供读取
  pFile = MZ_FOPEN(pFilename, "r+b");
  if (!pFile)
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_OPEN_FAILED);

  # 获取文件大小
  file_size = archive_size;
  if (!file_size) {
    if (MZ_FSEEK64(pFile, 0, SEEK_END)) {
      MZ_FCLOSE(pFile);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_SEEK_FAILED);
    }

    file_size = MZ_FTELL64(pFile);
  }

  /* TODO: Better sanity check archive_size and the # of actual remaining bytes
   */

  # 检查文件大小是否足够包含 ZIP 归档结束中央目录头部
  if (file_size < MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE) {
    MZ_FCLOSE(pFile);
    return mz_zip_set_error(pZip, MZ_ZIP_NOT_AN_ARCHIVE);
  }

  # 初始化 ZIP 读取器内部
  if (!mz_zip_reader_init_internal(pZip, flags)) {
    MZ_FCLOSE(pFile);
    return MZ_FALSE;
  }

  # 设置 ZIP 归档类型为文件，设置读取函数和文件指针
  pZip->m_zip_type = MZ_ZIP_TYPE_FILE;
  pZip->m_pRead = mz_zip_file_read_func;
  pZip->m_pIO_opaque = pZip;
  pZip->m_pState->m_pFile = pFile;
  pZip->m_archive_size = file_size;
  pZip->m_pState->m_file_archive_start_ofs = file_start_ofs;

  # 读取中央目录信息
  if (!mz_zip_reader_read_central_dir(pZip, flags)) {
    mz_zip_reader_end_internal(pZip, MZ_FALSE);
    return MZ_FALSE;
  }

  return MZ_TRUE;
}

# 初始化 ZIP 归档读取器，从 C 文件中读取 ZIP 归档数据
mz_bool mz_zip_reader_init_cfile(mz_zip_archive *pZip, MZ_FILE *pFile,
                                 mz_uint64 archive_size, mz_uint flags) {
  mz_uint64 cur_file_ofs;

  # 检查参数是否有效，如果无效则返回错误
  if ((!pZip) || (!pFile))
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_OPEN_FAILED);

  # 获取当前文件偏移量
  cur_file_ofs = MZ_FTELL64(pFile);

  # 如果归档大小为 0，则将文件指针移动到文件末尾
  if (!archive_size) {
    if (MZ_FSEEK64(pFile, 0, SEEK_END))
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_SEEK_FAILED);
    # 计算归档文件的大小，通过当前文件偏移量和文件指针位置计算
    archive_size = MZ_FTELL64(pFile) - cur_file_ofs;

    # 如果归档文件大小小于 ZIP 文件结束中央目录头的大小，则返回错误
    if (archive_size < MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE)
      return mz_zip_set_error(pZip, MZ_ZIP_NOT_AN_ARCHIVE);
  }

  # 初始化 ZIP 读取器内部状态
  if (!mz_zip_reader_init_internal(pZip, flags))
    return MZ_FALSE;

  # 设置 ZIP 对象的类型为 C 文件
  pZip->m_zip_type = MZ_ZIP_TYPE_CFILE;
  # 设置 ZIP 对象的读取函数为 mz_zip_file_read_func

  pZip->m_pRead = mz_zip_file_read_func;

  # 设置 ZIP 对象的 IO 透明指针为 ZIP 对象本身
  pZip->m_pIO_opaque = pZip;
  # 设置 ZIP 对象状态的文件指针为当前文件指针
  pZip->m_pState->m_pFile = pFile;
  # 设置 ZIP 对象的归档文件大小为计算得到的归档文件大小
  pZip->m_archive_size = archive_size;
  # 设置 ZIP 对象状态的文件归档起始偏移量为当前文件偏移量

  pZip->m_pState->m_file_archive_start_ofs = cur_file_ofs;

  # 如果无法读取 ZIP 中央目录信息，则结束 ZIP 读取器内部状态并返回错误
  if (!mz_zip_reader_read_central_dir(pZip, flags)) {
    mz_zip_reader_end_internal(pZip, MZ_FALSE);
    return MZ_FALSE;
  }

  # 返回读取 ZIP 文件成功
  return MZ_TRUE;
// 如果未定义 MINIZ_NO_STDIO，则结束 ifdef 块
#endif /* #ifndef MINIZ_NO_STDIO */

// 获取指定文件索引的中央目录头信息
static MZ_FORCEINLINE const mz_uint8 *mz_zip_get_cdh(mz_zip_archive *pZip,
                                                     mz_uint file_index) {
  // 如果 ZIP 对象为空，或者 ZIP 对象状态为空，或者文件索引超出总文件数，则返回空指针
  if ((!pZip) || (!pZip->m_pState) || (file_index >= pZip->m_total_files))
    return NULL;
  // 返回指向中央目录头信息的指针
  return &MZ_ZIP_ARRAY_ELEMENT(
      &pZip->m_pState->m_central_dir, mz_uint8,
      MZ_ZIP_ARRAY_ELEMENT(&pZip->m_pState->m_central_dir_offsets, mz_uint32,
                           file_index));
}

// 检查指定文件索引的文件是否加密
mz_bool mz_zip_reader_is_file_encrypted(mz_zip_archive *pZip,
                                        mz_uint file_index) {
  mz_uint m_bit_flag;
  // 获取中央目录头信息的指针
  const mz_uint8 *p = mz_zip_get_cdh(pZip, file_index);
  // 如果指针为空，设置 ZIP 错误并返回假
  if (!p) {
    mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
    return MZ_FALSE;
  }

  // 读取位标志
  m_bit_flag = MZ_READ_LE16(p + MZ_ZIP_CDH_BIT_FLAG_OFS);
  // 检查是否加密
  return (m_bit_flag &
          (MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_IS_ENCRYPTED |
           MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_USES_STRONG_ENCRYPTION)) != 0;
}

// 检查指定文件索引的文件是否受支持
mz_bool mz_zip_reader_is_file_supported(mz_zip_archive *pZip,
                                        mz_uint file_index) {
  mz_uint bit_flag;
  mz_uint method;

  // 获取中央目录头信息的指针
  const mz_uint8 *p = mz_zip_get_cdh(pZip, file_index);
  // 如果指针为空，设置 ZIP 错误并返回假
  if (!p) {
    mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
    return MZ_FALSE;
  }

  // 读取压缩方法和位标志
  method = MZ_READ_LE16(p + MZ_ZIP_CDH_METHOD_OFS);
  bit_flag = MZ_READ_LE16(p + MZ_ZIP_CDH_BIT_FLAG_OFS);

  // 如果压缩方法不为 0 或不为 MZ_DEFLATED，则设置 ZIP 错误并返回假
  if ((method != 0) && (method != MZ_DEFLATED)) {
    mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_METHOD);
    return MZ_FALSE;
  }

  // 如果文件使用加密或强加密，则设置 ZIP 错误并返回假
  if (bit_flag & (MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_IS_ENCRYPTED |
                  MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_USES_STRONG_ENCRYPTION)) {
    mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_ENCRYPTION);
    return MZ_FALSE;
  }

  // 如果文件使用压缩补丁标志，则设置 ZIP 错误并返回假
  if (bit_flag & MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_COMPRESSED_PATCH_FLAG) {
    mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_FEATURE);
    return MZ_FALSE;
  }

  // 返回真表示文件受支持
  return MZ_TRUE;
}
# 检查给定文件索引对应的文件是否为目录
mz_bool mz_zip_reader_is_file_a_directory(mz_zip_archive *pZip,
                                          mz_uint file_index) {
  # 定义变量：文件名长度、属性映射 ID、外部属性
  mz_uint filename_len, attribute_mapping_id, external_attr;
  # 获取中央目录头指针
  const mz_uint8 *p = mz_zip_get_cdh(pZip, file_index);
  # 如果指针为空，设置 ZIP 错误并返回假
  if (!p) {
    mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
    return MZ_FALSE;
  }

  # 读取文件名长度
  filename_len = MZ_READ_LE16(p + MZ_ZIP_CDH_FILENAME_LEN_OFS);
  # 如果文件名长度不为零，检查最后一个字符是否为 '/'
  if (filename_len) {
    if (*(p + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + filename_len - 1) == '/')
      return MZ_TRUE;
  }

  # 检查 DOS 目录属性位标志是否被设置
  attribute_mapping_id = MZ_READ_LE16(p + MZ_ZIP_CDH_VERSION_MADE_BY_OFS) >> 8;
  (void)attribute_mapping_id;

  external_attr = MZ_READ_LE32(p + MZ_ZIP_CDH_EXTERNAL_ATTR_OFS);
  # 如果外部属性中包含 DOS 目录属性位标志，返回真
  if ((external_attr & MZ_ZIP_DOS_DIR_ATTRIBUTE_BITFLAG) != 0) {
    return MZ_TRUE;
  }

  # 否则返回假
  return MZ_FALSE;
}

# 内部函数：获取文件统计信息
static mz_bool mz_zip_file_stat_internal(mz_zip_archive *pZip,
                                         mz_uint file_index,
                                         const mz_uint8 *pCentral_dir_header,
                                         mz_zip_archive_file_stat *pStat,
                                         mz_bool *pFound_zip64_extra_data) {
  mz_uint n;
  const mz_uint8 *p = pCentral_dir_header;

  # 如果找到 ZIP64 额外数据，设置为假
  if (pFound_zip64_extra_data)
    *pFound_zip64_extra_data = MZ_FALSE;

  # 如果中央目录头指针为空或文件统计信息指针为空，则返回假
  if ((!p) || (!pStat))
    // 返回一个表示 ZIP 文件无效参数的错误代码
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  /* 从中央目录记录中提取字段。 */
  // 设置文件索引
  pStat->m_file_index = file_index;
  // 设置中央目录偏移量
  pStat->m_central_dir_ofs = MZ_ZIP_ARRAY_ELEMENT(
      &pZip->m_pState->m_central_dir_offsets, mz_uint32, file_index);
  // 读取版本信息
  pStat->m_version_made_by = MZ_READ_LE16(p + MZ_ZIP_CDH_VERSION_MADE_BY_OFS);
  // 读取所需版本信息
  pStat->m_version_needed = MZ_READ_LE16(p + MZ_ZIP_CDH_VERSION_NEEDED_OFS);
  // 读取位标志
  pStat->m_bit_flag = MZ_READ_LE16(p + MZ_ZIP_CDH_BIT_FLAG_OFS);
  // 读取压缩方法
  pStat->m_method = MZ_READ_LE16(p + MZ_ZIP_CDH_METHOD_OFS);
#ifndef MINIZ_NO_TIME
  // 如果定义了 MINIZ_NO_TIME 宏，则不处理时间信息
  pStat->m_time =
      mz_zip_dos_to_time_t(MZ_READ_LE16(p + MZ_ZIP_CDH_FILE_TIME_OFS),
                           MZ_READ_LE16(p + MZ_ZIP_CDH_FILE_DATE_OFS));
#endif
  // 读取 CRC32 校验值
  pStat->m_crc32 = MZ_READ_LE32(p + MZ_ZIP_CDH_CRC32_OFS);
  // 读取压缩后的文件大小
  pStat->m_comp_size = MZ_READ_LE32(p + MZ_ZIP_CDH_COMPRESSED_SIZE_OFS);
  // 读取解压后的文件大小
  pStat->m_uncomp_size = MZ_READ_LE32(p + MZ_ZIP_CDH_DECOMPRESSED_SIZE_OFS);
  // 读取内部属性
  pStat->m_internal_attr = MZ_READ_LE16(p + MZ_ZIP_CDH_INTERNAL_ATTR_OFS);
  // 读取外部属性
  pStat->m_external_attr = MZ_READ_LE32(p + MZ_ZIP_CDH_EXTERNAL_ATTR_OFS);
  // 读取本地头偏移量
  pStat->m_local_header_ofs = MZ_READ_LE32(p + MZ_ZIP_CDH_LOCAL_HEADER_OFS);

  /* 复制尽可能多的文件名和注释。 */
  // 读取文件名长度
  n = MZ_READ_LE16(p + MZ_ZIP_CDH_FILENAME_LEN_OFS);
  // 限制文件名长度
  n = MZ_MIN(n, MZ_ZIP_MAX_ARCHIVE_FILENAME_SIZE - 1);
  // 复制文件名
  memcpy(pStat->m_filename, p + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE, n);
  // 添加字符串结束符
  pStat->m_filename[n] = '\0';

  // 读取注释长度
  n = MZ_READ_LE16(p + MZ_ZIP_CDH_COMMENT_LEN_OFS);
  // 限制注释长度
  n = MZ_MIN(n, MZ_ZIP_MAX_ARCHIVE_FILE_COMMENT_SIZE - 1);
  // 设置注释大小
  pStat->m_comment_size = n;
  // 复制注释
  memcpy(pStat->m_comment,
         p + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE +
             MZ_READ_LE16(p + MZ_ZIP_CDH_FILENAME_LEN_OFS) +
             MZ_READ_LE16(p + MZ_ZIP_CDH_EXTRA_LEN_OFS),
         n);
  // 添加字符串结束符
  pStat->m_comment[n] = '\0';

  /* 设置一些方便的标志 */
  // 判断文件是否为目录
  pStat->m_is_directory = mz_zip_reader_is_file_a_directory(pZip, file_index);
  // 判断文件是否加密
  pStat->m_is_encrypted = mz_zip_reader_is_file_encrypted(pZip, file_index);
  // 判断文件是否受支持
  pStat->m_is_supported = mz_zip_reader_is_file_supported(pZip, file_index);

  /* 查看是否需要读取任何 zip64 扩展信息字段。 */
  /* 令人困惑的是，这些 zip64 字段甚至可以存在于非 zip64 存档中
   * （Debian zip 在从标准输入传输到标准输出的大文件上创建它们）。 */
  if (MZ_MAX(MZ_MAX(pStat->m_comp_size, pStat->m_uncomp_size),
             pStat->m_local_header_ofs) == MZ_UINT32_MAX) {
    /* 尝试在条目的额外数据中查找 zip64 扩展信息字段 */
    # 读取 p 指针指向位置的 16 位无符号整数，表示额外字段的大小
    mz_uint32 extra_size_remaining = MZ_READ_LE16(p + MZ_ZIP_CDH_EXTRA_LEN_OFS);

    # 结束当前函数的定义，返回到调用该函数的地方
    }
  }

  # 返回真值，表示函数执行成功
  return MZ_TRUE;
# 比较两个字符串是否相等，根据传入的标志位判断是否区分大小写
static MZ_FORCEINLINE mz_bool mz_zip_string_equal(const char *pA,
                                                  const char *pB, mz_uint len,
                                                  mz_uint flags) {
  mz_uint i;
  # 如果标志位包含 MZ_ZIP_FLAG_CASE_SENSITIVE，则使用 memcmp 函数比较两个字符串
  if (flags & MZ_ZIP_FLAG_CASE_SENSITIVE)
    return 0 == memcmp(pA, pB, len);
  # 否则，遍历比较两个字符串的每个字符，忽略大小写
  for (i = 0; i < len; ++i)
    if (MZ_TOLOWER(pA[i]) != MZ_TOLOWER(pB[i]))
      return MZ_FALSE;
  return MZ_TRUE;
}

# 比较 ZIP 文件中的文件名与给定文件名，返回比较结果
static MZ_FORCEINLINE int
mz_zip_filename_compare(const mz_zip_array *pCentral_dir_array,
                        const mz_zip_array *pCentral_dir_offsets,
                        mz_uint l_index, const char *pR, mz_uint r_len) {
  # 获取 ZIP 文件中的文件名和长度
  const mz_uint8 *pL = &MZ_ZIP_ARRAY_ELEMENT(
                     pCentral_dir_array, mz_uint8,
                     MZ_ZIP_ARRAY_ELEMENT(pCentral_dir_offsets, mz_uint32,
                                          l_index)),
                 *pE;
  mz_uint l_len = MZ_READ_LE16(pL + MZ_ZIP_CDH_FILENAME_LEN_OFS);
  mz_uint8 l = 0, r = 0;
  pL += MZ_ZIP_CENTRAL_DIR_HEADER_SIZE;
  pE = pL + MZ_MIN(l_len, r_len);
  # 逐个字符比较两个文件名
  while (pL < pE) {
    if ((l = MZ_TOLOWER(*pL)) != (r = MZ_TOLOWER(*pR)))
      break;
    pL++;
    pR++;
  }
  # 返回比较结果
  return (pL == pE) ? (int)(l_len - r_len) : (l - r);
}

# 通过二分查找在 ZIP 文件中定位指定文件名的索引
static mz_bool mz_zip_locate_file_binary_search(mz_zip_archive *pZip,
                                                const char *pFilename,
                                                mz_uint32 *pIndex) {
  mz_zip_internal_state *pState = pZip->m_pState;
  const mz_zip_array *pCentral_dir_offsets = &pState->m_central_dir_offsets;
  const mz_zip_array *pCentral_dir = &pState->m_central_dir;
  mz_uint32 *pIndices = &MZ_ZIP_ARRAY_ELEMENT(
      &pState->m_sorted_central_dir_offsets, mz_uint32, 0);
  const uint32_t size = pZip->m_total_files;
  const mz_uint filename_len = (mz_uint)strlen(pFilename);

  # 如果传入的索引指针不为空，则将其初始化为 0
  if (pIndex)
    *pIndex = 0;

  # 如果 ZIP 文件中存在文件
  if (size) {
    /* 使用 mz_int64 类型来表示文件索引范围，避免特殊情况的检查 */
    /* 在 32 位 CPU 上，文件名比较仍然是主要的开销 */
    mz_int64 l = 0, h = (mz_int64)size - 1;

    while (l <= h) {
      /* 计算中间索引值 */
      mz_int64 m = l + ((h - l) >> 1);
      /* 获取文件索引 */
      uint32_t file_index = pIndices[(uint32_t)m];

      /* 比较文件名，返回比较结果 */
      int comp = mz_zip_filename_compare(pCentral_dir, pCentral_dir_offsets,
                                         file_index, pFilename, filename_len);
      /* 如果文件名相同，则返回成功 */
      if (!comp) {
        if (pIndex)
          *pIndex = file_index;
        return MZ_TRUE;
      } else if (comp < 0)
        l = m + 1;
      else
        h = m - 1;
    }
  }

  /* 如果未找到文件，则设置 ZIP 错误并返回 */
  return mz_zip_set_error(pZip, MZ_ZIP_FILE_NOT_FOUND);
// 定义函数，用于在 ZIP 存档中定位文件，并返回文件索引
int mz_zip_reader_locate_file(mz_zip_archive *pZip, const char *pName,
                              const char *pComment, mz_uint flags) {
  mz_uint32 index;
  // 调用 mz_zip_reader_locate_file_v2 函数来定位文件，并获取文件索引
  if (!mz_zip_reader_locate_file_v2(pZip, pName, pComment, flags, &index))
    return -1;
  else
    return (int)index;
}

// 定义函数，用于在 ZIP 存档中定位文件，并返回文件索引
mz_bool mz_zip_reader_locate_file_v2(mz_zip_archive *pZip, const char *pName,
                                     const char *pComment, mz_uint flags,
                                     mz_uint32 *pIndex) {
  mz_uint file_index;
  size_t name_len, comment_len;

  // 如果 pIndex 不为空，则将其值设为 0
  if (pIndex)
    *pIndex = 0;

  // 检查参数是否有效
  if ((!pZip) || (!pZip->m_pState) || (!pName))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  /* See if we can use a binary search */
  // 检查是否可以使用二分查找
  if (((pZip->m_pState->m_init_flags &
        MZ_ZIP_FLAG_DO_NOT_SORT_CENTRAL_DIRECTORY) == 0) &&
      (pZip->m_zip_mode == MZ_ZIP_MODE_READING) &&
      ((flags & (MZ_ZIP_FLAG_IGNORE_PATH | MZ_ZIP_FLAG_CASE_SENSITIVE)) == 0) &&
      (!pComment) && (pZip->m_pState->m_sorted_central_dir_offsets.m_size)) {
    return mz_zip_locate_file_binary_search(pZip, pName, pIndex);
  }

  /* Locate the entry by scanning the entire central directory */
  // 通过扫描整个中央目录来定位条目
  name_len = strlen(pName);
  if (name_len > MZ_UINT16_MAX)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  comment_len = pComment ? strlen(pComment) : 0;
  if (comment_len > MZ_UINT16_MAX)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 遍历整个中央目录
  for (file_index = 0; file_index < pZip->m_total_files; file_index++) {
    const mz_uint8 *pHeader = &MZ_ZIP_ARRAY_ELEMENT(
        &pZip->m_pState->m_central_dir, mz_uint8,
        MZ_ZIP_ARRAY_ELEMENT(&pZip->m_pState->m_central_dir_offsets, mz_uint32,
                             file_index));
    mz_uint filename_len = MZ_READ_LE16(pHeader + MZ_ZIP_CDH_FILENAME_LEN_OFS);
    const char *pFilename =
        (const char *)pHeader + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE;
    // 如果文件名长度小于给定文件名长度，则继续下一个循环
    if (filename_len < name_len)
      continue;
    // 如果注释长度不为零
    if (comment_len) {
      // 读取文件额外长度和文件注释长度
      mz_uint file_extra_len = MZ_READ_LE16(pHeader + MZ_ZIP_CDH_EXTRA_LEN_OFS),
              file_comment_len =
                  MZ_READ_LE16(pHeader + MZ_ZIP_CDH_COMMENT_LEN_OFS);
      // 获取文件注释的指针
      const char *pFile_comment = pFilename + filename_len + file_extra_len;
      // 如果文件注释长度不等于给定注释长度，或者文件注释与给定注释不相等，则继续循环
      if ((file_comment_len != comment_len) ||
          (!mz_zip_string_equal(pComment, pFile_comment, file_comment_len,
                                flags)))
        continue;
    }
    // 如果忽略路径标志被设置，并且文件名长度不为零
    if ((flags & MZ_ZIP_FLAG_IGNORE_PATH) && (filename_len)) {
      // 从文件名末尾向前查找路径分隔符或冒号
      int ofs = filename_len - 1;
      do {
        if ((pFilename[ofs] == '/') || (pFilename[ofs] == '\\') ||
            (pFilename[ofs] == ':'))
          break;
      } while (--ofs >= 0);
      ofs++;
      // 更新文件名指针和长度
      pFilename += ofs;
      filename_len -= ofs;
    }
    // 如果文件名长度等于给定名称长度，并且文件名与给定名称相等
    if ((filename_len == name_len) &&
        (mz_zip_string_equal(pName, pFilename, filename_len, flags))) {
      // 如果索引指针不为空，则将文件索引赋值给索引指针
      if (pIndex)
        *pIndex = file_index;
      // 返回真值表示找到文件
      return MZ_TRUE;
    }
  }

  // 文件未找到，设置 ZIP 错误并返回
  return mz_zip_set_error(pZip, MZ_ZIP_FILE_NOT_FOUND);
# 检查参数和状态，准备解压缩文件到内存中，不分配内存
static mz_bool mz_zip_reader_extract_to_mem_no_alloc1(
    mz_zip_archive *pZip, mz_uint file_index, void *pBuf, size_t buf_size,
    mz_uint flags, void *pUser_read_buf, size_t user_read_buf_size,
    const mz_zip_archive_file_stat *st) {
  int status = TINFL_STATUS_DONE;
  mz_uint64 needed_size, cur_file_ofs, comp_remaining,
      out_buf_ofs = 0, read_buf_size, read_buf_ofs = 0, read_buf_avail;
  mz_zip_archive_file_stat file_stat;
  void *pRead_buf;
  mz_uint32
      local_header_u32[(MZ_ZIP_LOCAL_DIR_HEADER_SIZE + sizeof(mz_uint32) - 1) /
                       sizeof(mz_uint32)];
  mz_uint8 *pLocal_header = (mz_uint8 *)local_header_u32;
  tinfl_decompressor inflator;

  # 检查 ZIP 归档对象和参数是否有效
  if ((!pZip) || (!pZip->m_pState) || ((buf_size) && (!pBuf)) ||
      ((user_read_buf_size) && (!pUser_read_buf)) || (!pZip->m_pRead))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  # 如果提供了文件状态，则使用提供的文件状态，否则获取文件状态
  if (st) {
    file_stat = *st;
  } else if (!mz_zip_reader_file_stat(pZip, file_index, &file_stat))
    return MZ_FALSE;

  /* A directory or zero length file */
  # 如果是目录或者文件长度为零，则直接返回成功
  if ((file_stat.m_is_directory) || (!file_stat.m_comp_size))
    return MZ_TRUE;

  /* Encryption and patch files are not supported. */
  # 不支持加密和补丁文件
  if (file_stat.m_bit_flag &
      (MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_IS_ENCRYPTED |
       MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_USES_STRONG_ENCRYPTION |
       MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_COMPRESSED_PATCH_FLAG))
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_ENCRYPTION);

  /* This function only supports decompressing stored and deflate. */
  # 该函数仅支持解压存储和压缩的文件
  if ((!(flags & MZ_ZIP_FLAG_COMPRESSED_DATA)) && (file_stat.m_method != 0) &&
      (file_stat.m_method != MZ_DEFLATED))
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_METHOD);

  /* Ensure supplied output buffer is large enough. */
  # 确保提供的输出缓冲区足够大
  needed_size = (flags & MZ_ZIP_FLAG_COMPRESSED_DATA) ? file_stat.m_comp_size
                                                      : file_stat.m_uncomp_size;
  if (buf_size < needed_size)
    // 返回缓冲区太小的错误
    return mz_zip_set_error(pZip, MZ_ZIP_BUF_TOO_SMALL);

  /* 读取并解析本地目录条目 */
  // 获取当前文件的本地头偏移量
  cur_file_ofs = file_stat.m_local_header_ofs;
  // 从文件中读取本地头信息
  if (pZip->m_pRead(pZip->m_pIO_opaque, cur_file_ofs, pLocal_header,
                    MZ_ZIP_LOCAL_DIR_HEADER_SIZE) !=
      MZ_ZIP_LOCAL_DIR_HEADER_SIZE)
    // 文件读取失败，返回错误
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);

  // 检查本地头的标识是否正确
  if (MZ_READ_LE32(pLocal_header) != MZ_ZIP_LOCAL_DIR_HEADER_SIG)
    // 本地头标识无效或损坏，返回错误
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 计算文件数据的偏移量
  cur_file_ofs += MZ_ZIP_LOCAL_DIR_HEADER_SIZE +
                  MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_FILENAME_LEN_OFS) +
                  MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_EXTRA_LEN_OFS);
  // 检查文件数据是否超出存档大小
  if ((cur_file_ofs + file_stat.m_comp_size) > pZip->m_archive_size)
    // 文件头无效或损坏，返回错误
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 如果文件是存储的或调用者请求压缩数据
  if ((flags & MZ_ZIP_FLAG_COMPRESSED_DATA) || (!file_stat.m_method)) {
    /* 文件是存储的或调用者请求压缩数据 */
    // 读取文件的压缩数据
    if (pZip->m_pRead(pZip->m_pIO_opaque, cur_file_ofs, pBuf,
                      (size_t)needed_size) != needed_size)
      // 文件读取失败，返回错误
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
    // 如果未禁用 ZIP 读取器的 CRC32 检查
    if ((flags & MZ_ZIP_FLAG_COMPRESSED_DATA) == 0) {
      // 如果文件未压缩
      if (mz_crc32(MZ_CRC32_INIT, (const mz_uint8 *)pBuf,
                   (size_t)file_stat.m_uncomp_size) != file_stat.m_crc32)
        // 如果 CRC32 校验失败，则返回错误
        return mz_zip_set_error(pZip, MZ_ZIP_CRC_CHECK_FAILED);
    }
#endif

    // 返回成功
    return MZ_TRUE;
  }

  /* Decompress the file either directly from memory or from a file input
   * buffer. */
  // 初始化解压缩器
  tinfl_init(&inflator);

  if (pZip->m_pState->m_pMem) {
    /* Read directly from the archive in memory. */
    // 从内存中直接读取存档
    pRead_buf = (mz_uint8 *)pZip->m_pState->m_pMem + cur_file_ofs;
    read_buf_size = read_buf_avail = file_stat.m_comp_size;
    comp_remaining = 0;
  } else if (pUser_read_buf) {
    /* Use a user provided read buffer. */
    // 使用用户提供的读取缓冲区
    if (!user_read_buf_size)
      return MZ_FALSE;
    pRead_buf = (mz_uint8 *)pUser_read_buf;
    read_buf_size = user_read_buf_size;
    read_buf_avail = 0;
    comp_remaining = file_stat.m_comp_size;
  } else {
    /* Temporarily allocate a read buffer. */
    // 临时分配一个读取缓冲区
    read_buf_size =
        MZ_MIN(file_stat.m_comp_size, (mz_uint64)MZ_ZIP_MAX_IO_BUF_SIZE);
    if (((sizeof(size_t) == sizeof(mz_uint32))) && (read_buf_size > 0x7FFFFFFF))
      return mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);

    if (NULL == (pRead_buf = pZip->m_pAlloc(pZip->m_pAlloc_opaque, 1,
                                            (size_t)read_buf_size)))
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

    read_buf_avail = 0;
    comp_remaining = file_stat.m_comp_size;
  }

  do {
    /* The size_t cast here should be OK because we've verified that the output
     * buffer is >= file_stat.m_uncomp_size above */
    // 这里的 size_t 转换应该没问题，因为我们已经验证了输出缓冲区大于等于文件的未压缩大小
    size_t in_buf_size,
        out_buf_size = (size_t)(file_stat.m_uncomp_size - out_buf_ofs);
    // 如果读取缓冲区不可用且没有内存缓冲区
    if ((!read_buf_avail) && (!pZip->m_pState->m_pMem)) {
      // 设置读取缓冲区可用大小为读取缓冲区大小和剩余压缩数据大小的最小值
      read_buf_avail = MZ_MIN(read_buf_size, comp_remaining);
      // 通过调用读取函数读取数据到读取缓冲区
      if (pZip->m_pRead(pZip->m_pIO_opaque, cur_file_ofs, pRead_buf,
                        (size_t)read_buf_avail) != read_buf_avail) {
        // 如果读取失败，设置状态为失败并设置 ZIP 错误
        status = TINFL_STATUS_FAILED;
        mz_zip_set_error(pZip, MZ_ZIP_DECOMPRESSION_FAILED);
        break;
      }
      // 更新当前文件偏移和剩余压缩数据大小
      cur_file_ofs += read_buf_avail;
      comp_remaining -= read_buf_avail;
      // 重置读取缓冲区偏移
      read_buf_ofs = 0;
    }
    // 设置输入缓冲区大小为读取缓冲区可用大小
    in_buf_size = (size_t)read_buf_avail;
    // 调用 tinfl_decompress 函数进行解压缩
    status = tinfl_decompress(
        &inflator, (mz_uint8 *)pRead_buf + read_buf_ofs, &in_buf_size,
        (mz_uint8 *)pBuf, (mz_uint8 *)pBuf + out_buf_ofs, &out_buf_size,
        TINFL_FLAG_USING_NON_WRAPPING_OUTPUT_BUF |
            (comp_remaining ? TINFL_FLAG_HAS_MORE_INPUT : 0));
    // 更新读取缓冲区可用大小和偏移，输出缓冲区偏移
    read_buf_avail -= in_buf_size;
    read_buf_ofs += in_buf_size;
    out_buf_ofs += out_buf_size;
  } while (status == TINFL_STATUS_NEEDS_MORE_INPUT);

  // 如果解压缩完成
  if (status == TINFL_STATUS_DONE) {
    /* 确保整个文件已解压缩，并检查其 CRC */
    // 如果输出缓冲区偏移不等于文件的未压缩大小，设置 ZIP 错误并将状态设置为失败
    if (out_buf_ofs != file_stat.m_uncomp_size) {
      mz_zip_set_error(pZip, MZ_ZIP_UNEXPECTED_DECOMPRESSED_SIZE);
      status = TINFL_STATUS_FAILED;
    }
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
    // 如果未禁用 ZIP 读取器的 CRC32 检查
    else if (mz_crc32(MZ_CRC32_INIT, (const mz_uint8 *)pBuf,
                      (size_t)file_stat.m_uncomp_size) != file_stat.m_crc32) {
      // 计算缓冲区的 CRC32 值并与文件的 CRC32 值进行比较，如果不匹配
      mz_zip_set_error(pZip, MZ_ZIP_CRC_CHECK_FAILED);
      // 设置 ZIP 错误为 CRC 校验失败
      status = TINFL_STATUS_FAILED;
      // 设置状态为失败
    }
#endif
  }

  if ((!pZip->m_pState->m_pMem) && (!pUser_read_buf))
    // 如果没有分配内存并且没有用户读取缓冲区
    pZip->m_pFree(pZip->m_pAlloc_opaque, pRead_buf);
    // 释放读取缓冲区

  return status == TINFL_STATUS_DONE;
  // 返回状态是否为完成状态
}

mz_bool mz_zip_reader_extract_to_mem_no_alloc(mz_zip_archive *pZip,
                                              mz_uint file_index, void *pBuf,
                                              size_t buf_size, mz_uint flags,
                                              void *pUser_read_buf,
                                              size_t user_read_buf_size) {
  // 从 ZIP 存档中提取文件到内存，不分配内存
  return mz_zip_reader_extract_to_mem_no_alloc1(pZip, file_index, pBuf,
                                                buf_size, flags, pUser_read_buf,
                                                user_read_buf_size, NULL);
}

mz_bool mz_zip_reader_extract_file_to_mem_no_alloc(
    mz_zip_archive *pZip, const char *pFilename, void *pBuf, size_t buf_size,
    mz_uint flags, void *pUser_read_buf, size_t user_read_buf_size) {
  mz_uint32 file_index;
  // 通过文件名在 ZIP 存档中定位文件
  if (!mz_zip_reader_locate_file_v2(pZip, pFilename, NULL, flags, &file_index))
    return MZ_FALSE;
  // 如果未找到文件，则返回假
  return mz_zip_reader_extract_to_mem_no_alloc(pZip, file_index, pBuf, buf_size,
                                               flags, pUser_read_buf,
                                               user_read_buf_size);
}

mz_bool mz_zip_reader_extract_to_mem(mz_zip_archive *pZip, mz_uint file_index,
                                     void *pBuf, size_t buf_size,
                                     mz_uint flags) {
  // 从 ZIP 存档中提取文件到内存
  return mz_zip_reader_extract_to_mem_no_alloc(pZip, file_index, pBuf, buf_size,
                                               flags, NULL, 0);
}
# 将 ZIP 存档中的文件提取到内存中，不分配内存
mz_bool mz_zip_reader_extract_file_to_mem(mz_zip_archive *pZip,
                                          const char *pFilename, void *pBuf,
                                          size_t buf_size, mz_uint flags) {
  # 调用不分配内存的提取函数
  return mz_zip_reader_extract_file_to_mem_no_alloc(pZip, pFilename, pBuf,
                                                    buf_size, flags, NULL, 0);
}

# 将 ZIP 存档中的文件提取到堆中
void *mz_zip_reader_extract_to_heap(mz_zip_archive *pZip, mz_uint file_index,
                                    size_t *pSize, mz_uint flags) {
  # 定义文件统计信息和分配大小
  mz_zip_archive_file_stat file_stat;
  mz_uint64 alloc_size;
  void *pBuf;

  # 如果 pSize 不为空，则将其设置为 0
  if (pSize)
    *pSize = 0;

  # 获取文件统计信息，如果失败则返回空指针
  if (!mz_zip_reader_file_stat(pZip, file_index, &file_stat))
    return NULL;

  # 根据标志位选择分配大小
  alloc_size = (flags & MZ_ZIP_FLAG_COMPRESSED_DATA) ? file_stat.m_comp_size
                                                     : file_stat.m_uncomp_size;
  
  # 如果分配大小超过 0x7FFFFFFF，且 size_t 和 mz_uint32 大小相同，则设置错误并返回空指针
  if (((sizeof(size_t) == sizeof(mz_uint32))) && (alloc_size > 0x7FFFFFFF)) {
    mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);
    return NULL;
  }

  # 分配内存，如果失败则设置错误并返回空指针
  if (NULL ==
      (pBuf = pZip->m_pAlloc(pZip->m_pAlloc_opaque, 1, (size_t)alloc_size))) {
    mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    return NULL;
  }

  # 提取文件到内存中，如果失败则释放内存并返回空指针
  if (!mz_zip_reader_extract_to_mem_no_alloc1(pZip, file_index, pBuf,
                                              (size_t)alloc_size, flags, NULL,
                                              0, &file_stat)) {
    pZip->m_pFree(pZip->m_pAlloc_opaque, pBuf);
    return NULL;
  }

  # 如果 pSize 不为空，则将其设置为分配大小
  if (pSize)
    *pSize = (size_t)alloc_size;
  return pBuf;
}

# 将 ZIP 存档中的指定文件提取到堆中
void *mz_zip_reader_extract_file_to_heap(mz_zip_archive *pZip,
                                         const char *pFilename, size_t *pSize,
                                         mz_uint flags) {
  mz_uint32 file_index;
  # 定位文件索引，如果失败则将 pSize 设置为 0 并返回 MZ_FALSE
  if (!mz_zip_reader_locate_file_v2(pZip, pFilename, NULL, flags,
                                    &file_index)) {
    if (pSize)
      *pSize = 0;
    return MZ_FALSE;
  }
  # 调用将文件提取到堆中的函数
  return mz_zip_reader_extract_to_heap(pZip, file_index, pSize, flags);
}
  // 从 ZIP 归档中提取文件到回调函数
  mz_bool mz_zip_reader_extract_to_callback(mz_zip_archive *pZip,
                                            mz_uint file_index,
                                            mz_file_write_func pCallback,
                                            void *pOpaque, mz_uint flags) {
    // 定义变量 status 用于存储解压状态
    int status = TINFL_STATUS_DONE;
    // 如果未禁用 CRC32 检查，则定义变量 file_crc32 用于存储 CRC32 值
    #ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
    mz_uint file_crc32 = MZ_CRC32_INIT;
    #endif
    // 定义多个变量用于存储读取、解压缩等信息
    mz_uint64 read_buf_size, read_buf_ofs = 0, read_buf_avail, comp_remaining,
                             out_buf_ofs = 0, cur_file_ofs;
    // 定义变量 file_stat 用于存储文件的统计信息
    mz_zip_archive_file_stat file_stat;
    // 定义指针变量 pRead_buf 和 pWrite_buf 用于读取和写入缓冲区
    void *pRead_buf = NULL;
    void *pWrite_buf = NULL;
    // 定义数组 local_header_u32 和指针 pLocal_header 用于存储本地文件头信息
    mz_uint32
        local_header_u32[(MZ_ZIP_LOCAL_DIR_HEADER_SIZE + sizeof(mz_uint32) - 1) /
                         sizeof(mz_uint32)];
    mz_uint8 *pLocal_header = (mz_uint8 *)local_header_u32;

    // 检查参数是否有效，如果无效则返回错误
    if ((!pZip) || (!pZip->m_pState) || (!pCallback) || (!pZip->m_pRead))
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

    // 获取文件的统计信息，如果失败则返回假值
    if (!mz_zip_reader_file_stat(pZip, file_index, &file_stat))
      return MZ_FALSE;

    // 如果是目录或零长度文件，则返回真值
    /* A directory or zero length file */
    if (file_stat.m_is_directory || (!file_stat.m_comp_size))
      return MZ_TRUE;

    // 如果文件使用了加密或补丁，则返回不支持的错误
    /* Encryption and patch files are not supported. */
    if (file_stat.m_bit_flag &
        (MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_IS_ENCRYPTED |
         MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_USES_STRONG_ENCRYPTION |
         MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_COMPRESSED_PATCH_FLAG))
      return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_ENCRYPTION);

    // 该函数仅支持解压存储和压缩的文件
    /* This function only supports decompressing stored and deflate. */
    if ((!(flags & MZ_ZIP_FLAG_COMPRESSED_DATA)) && (file_stat.m_method != 0) &&
        (file_stat.m_method != MZ_DEFLATED))
    // 返回不支持的方法错误
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_METHOD);

  /* 读取并对本地目录条目进行最小验证（不包括解析 zip64 信息，因为已经从中央目录获取了）
   */
  // 获取当前文件的偏移量
  cur_file_ofs = file_stat.m_local_header_ofs;
  // 从文件中读取本地目录头部信息，并进行最小验证
  if (pZip->m_pRead(pZip->m_pIO_opaque, cur_file_ofs, pLocal_header,
                    MZ_ZIP_LOCAL_DIR_HEADER_SIZE) !=
      MZ_ZIP_LOCAL_DIR_HEADER_SIZE)
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);

  // 检查本地目录头部的标识是否正确
  if (MZ_READ_LE32(pLocal_header) != MZ_ZIP_LOCAL_DIR_HEADER_SIG)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 计算当前文件的偏移量
  cur_file_ofs += MZ_ZIP_LOCAL_DIR_HEADER_SIZE +
                  MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_FILENAME_LEN_OFS) +
                  MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_EXTRA_LEN_OFS);
  // 检查文件数据是否超出存档大小
  if ((cur_file_ofs + file_stat.m_comp_size) > pZip->m_archive_size)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  /* 从内存中直接解压文件，或从文件输入缓冲区中解压文件 */
  if (pZip->m_pState->m_pMem) {
    // 如果文件数据在内存中，则直接从内存中读取
    pRead_buf = (mz_uint8 *)pZip->m_pState->m_pMem + cur_file_ofs;
    read_buf_size = read_buf_avail = file_stat.m_comp_size;
    comp_remaining = 0;
  } else {
    // 如果文件数据不在内存中，则从文件中读取
    read_buf_size =
        MZ_MIN(file_stat.m_comp_size, (mz_uint64)MZ_ZIP_MAX_IO_BUF_SIZE);
    // 分配读取缓冲区
    if (NULL == (pRead_buf = pZip->m_pAlloc(pZip->m_pAlloc_opaque, 1,
                                            (size_t)read_buf_size)))
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

    read_buf_avail = 0;
    comp_remaining = file_stat.m_comp_size;
  }

  // 如果文件是存储的或调用者请求压缩数据，则执行以下操作
  if ((flags & MZ_ZIP_FLAG_COMPRESSED_DATA) || (!file_stat.m_method)) {
    /* The file is stored or the caller has requested the compressed data. */
    // 检查 ZIP 对象的内存状态是否存在
    if (pZip->m_pState->m_pMem) {
      // 如果 size_t 和 mz_uint32 的大小相等，并且文件的压缩大小大于 MZ_UINT32_MAX，则返回内部错误
      if (((sizeof(size_t) == sizeof(mz_uint32))) &&
          (file_stat.m_comp_size > MZ_UINT32_MAX))
        return mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);

      // 调用回调函数处理读取的数据
      if (pCallback(pOpaque, out_buf_ofs, pRead_buf,
                    (size_t)file_stat.m_comp_size) != file_stat.m_comp_size) {
        // 如果回调函数处理失败，则设置 ZIP 错误并返回失败状态
        mz_zip_set_error(pZip, MZ_ZIP_WRITE_CALLBACK_FAILED);
        status = TINFL_STATUS_FAILED;
      } else if (!(flags & MZ_ZIP_FLAG_COMPRESSED_DATA)) {
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
        // 如果未禁用 ZIP 读取器的 CRC32 检查
        file_crc32 =
            (mz_uint32)mz_crc32(file_crc32, (const mz_uint8 *)pRead_buf,
                                (size_t)file_stat.m_comp_size);
#endif
      }

      // 更新当前文件偏移量和输出缓冲区偏移量
      cur_file_ofs += file_stat.m_comp_size;
      out_buf_ofs += file_stat.m_comp_size;
      comp_remaining = 0;
    } else {
      // 当还有剩余压缩数据时
      while (comp_remaining) {
        // 计算可读取的缓冲区大小
        read_buf_avail = MZ_MIN(read_buf_size, comp_remaining);
        // 通过 ZIP 对象的读取函数读取数据到 pRead_buf
        if (pZip->m_pRead(pZip->m_pIO_opaque, cur_file_ofs, pRead_buf,
                          (size_t)read_buf_avail) != read_buf_avail) {
          // 设置 ZIP 错误为文件读取失败
          mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
          status = TINFL_STATUS_FAILED;
          break;
        }

#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
        // 如果未禁用 ZIP 读取器的 CRC32 检查，并且数据未压缩
        if (!(flags & MZ_ZIP_FLAG_COMPRESSED_DATA)) {
          // 更新文件的 CRC32 值
          file_crc32 = (mz_uint32)mz_crc32(
              file_crc32, (const mz_uint8 *)pRead_buf, (size_t)read_buf_avail);
        }
#endif

        // 调用回调函数处理读取的数据
        if (pCallback(pOpaque, out_buf_ofs, pRead_buf,
                      (size_t)read_buf_avail) != read_buf_avail) {
          // 设置 ZIP 错误为写回调函数失败
          mz_zip_set_error(pZip, MZ_ZIP_WRITE_CALLBACK_FAILED);
          status = TINFL_STATUS_FAILED;
          break;
        }

        // 更新当前文件偏移量和输出缓冲区偏移量
        cur_file_ofs += read_buf_avail;
        out_buf_ofs += read_buf_avail;
        comp_remaining -= read_buf_avail;
      }
    }
  } else {
    // 初始化解压缩器
    tinfl_decompressor inflator;
    tinfl_init(&inflator);

    // 分配写缓冲区
    if (NULL == (pWrite_buf = pZip->m_pAlloc(pZip->m_pAlloc_opaque, 1,
                                             TINFL_LZ_DICT_SIZE))) {
      // 设置 ZIP 错误为分配内存失败
      mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
      status = TINFL_STATUS_FAILED;
    } else {
        // 如果不是第一次进入循环，则执行以下代码块
        do {
            // 计算当前写入缓冲区的位置
            mz_uint8 *pWrite_buf_cur = (mz_uint8 *)pWrite_buf + (out_buf_ofs & (TINFL_LZ_DICT_SIZE - 1));
            // 计算输入缓冲区和输出缓冲区的大小
            size_t in_buf_size,
                out_buf_size = TINFL_LZ_DICT_SIZE - (out_buf_ofs & (TINFL_LZ_DICT_SIZE - 1));
            // 如果读取缓冲区没有数据可用，并且没有自定义内存分配器
            if ((!read_buf_avail) && (!pZip->m_pState->m_pMem)) {
                // 读取缓冲区可用数据大小为读取缓冲区大小和压缩数据剩余大小的最小值
                read_buf_avail = MZ_MIN(read_buf_size, comp_remaining);
                // 读取数据到读取缓冲区
                if (pZip->m_pRead(pZip->m_pIO_opaque, cur_file_ofs, pRead_buf, (size_t)read_buf_avail) != read_buf_avail) {
                    // 如果读取失败，设置 ZIP 错误并跳出循环
                    mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
                    status = TINFL_STATUS_FAILED;
                    break;
                }
                // 更新当前文件偏移和剩余压缩数据大小
                cur_file_ofs += read_buf_avail;
                comp_remaining -= read_buf_avail;
                read_buf_ofs = 0;
            }

            // 输入缓冲区大小为读取缓冲区可用数据大小
            in_buf_size = (size_t)read_buf_avail;
            // 解压缩数据
            status = tinfl_decompress(
                &inflator, (const mz_uint8 *)pRead_buf + read_buf_ofs, &in_buf_size,
                (mz_uint8 *)pWrite_buf, pWrite_buf_cur, &out_buf_size,
                comp_remaining ? TINFL_FLAG_HAS_MORE_INPUT : 0);
            // 更新读取缓冲区可用数据大小和偏移
            read_buf_avail -= in_buf_size;
            read_buf_ofs += in_buf_size;

            // 如果输出缓冲区有数据
            if (out_buf_size) {
                // 调用回调函数写入数据到输出缓冲区
                if (pCallback(pOpaque, out_buf_ofs, pWrite_buf_cur, out_buf_size) != out_buf_size) {
                    // 如果写入失败，设置 ZIP 错误并跳出循环
                    mz_zip_set_error(pZip, MZ_ZIP_WRITE_CALLBACK_FAILED);
                    status = TINFL_STATUS_FAILED;
                    break;
                }
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
          // 如果未禁用 CRC32 检查，则计算文件的 CRC32 值
          file_crc32 =
              (mz_uint32)mz_crc32(file_crc32, pWrite_buf_cur, out_buf_size);
#endif
          // 如果输出缓冲区偏移量超过文件的未压缩大小，则标记解压缩失败
          if ((out_buf_ofs += out_buf_size) > file_stat.m_uncomp_size) {
            mz_zip_set_error(pZip, MZ_ZIP_DECOMPRESSION_FAILED);
            status = TINFL_STATUS_FAILED;
            break;
          }
        }
      } while ((status == TINFL_STATUS_NEEDS_MORE_INPUT) ||
               (status == TINFL_STATUS_HAS_MORE_OUTPUT));
    }
  }

  if ((status == TINFL_STATUS_DONE) &&
      (!(flags & MZ_ZIP_FLAG_COMPRESSED_DATA))) {
    /* 确保整个文件已解压缩，并检查其 CRC。 */
    if (out_buf_ofs != file_stat.m_uncomp_size) {
      mz_zip_set_error(pZip, MZ_ZIP_UNEXPECTED_DECOMPRESSED_SIZE);
      status = TINFL_STATUS_FAILED;
    }
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
    // 如果未禁用 CRC32 检查，并且文件的 CRC32 值与预期值不匹配，则标记解压缩失败
    else if (file_crc32 != file_stat.m_crc32) {
      mz_zip_set_error(pZip, MZ_ZIP_DECOMPRESSION_FAILED);
      status = TINFL_STATUS_FAILED;
    }
#endif
  }

  // 如果 ZIP 对象没有分配内存，则释放读取缓冲区
  if (!pZip->m_pState->m_pMem)
    pZip->m_pFree(pZip->m_pAlloc_opaque, pRead_buf);

  // 如果写入缓冲区存在，则释放写入缓冲区
  if (pWrite_buf)
    pZip->m_pFree(pZip->m_pAlloc_opaque, pWrite_buf);

  // 返回解压缩状态是否为完成状态
  return status == TINFL_STATUS_DONE;
}

// 从 ZIP 存档中提取文件到回调函数
mz_bool mz_zip_reader_extract_file_to_callback(mz_zip_archive *pZip,
                                               const char *pFilename,
                                               mz_file_write_func pCallback,
                                               void *pOpaque, mz_uint flags) {
  mz_uint32 file_index;
  // 如果无法定位文件索引，则返回假
  if (!mz_zip_reader_locate_file_v2(pZip, pFilename, NULL, flags, &file_index))
    return MZ_FALSE;

  // 调用函数从 ZIP 存档中提取文件到回调函数
  return mz_zip_reader_extract_to_callback(pZip, file_index, pCallback, pOpaque,
                                           flags);
}

// 解压缩迭代状态
mz_zip_reader_extract_iter_state *
# 创建一个新的 ZIP 文件解压迭代器
mz_zip_reader_extract_iter_new(mz_zip_archive *pZip, mz_uint file_index,
                               mz_uint flags) {
  # 定义本地头部数据的缓冲区
  mz_uint32
      local_header_u32[(MZ_ZIP_LOCAL_DIR_HEADER_SIZE + sizeof(mz_uint32) - 1) /
                       sizeof(mz_uint32)];
  # 将缓冲区转换为字节型指针
  mz_uint8 *pLocal_header = (mz_uint8 *)local_header_u32;

  /* 检查参数的合法性 */
  if ((!pZip) || (!pZip->m_pState))
    return NULL;

  /* 分配一个迭代器状态结构 */
  pState = (mz_zip_reader_extract_iter_state *)pZip->m_pAlloc(
      pZip->m_pAlloc_opaque, 1, sizeof(mz_zip_reader_extract_iter_state));
  if (!pState) {
    mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    return NULL;
  }

  /* 获取文件详细信息 */
  if (!mz_zip_reader_file_stat(pZip, file_index, &pState->file_stat)) {
    pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
    return NULL;
  }

  /* 不支持加密和补丁文件 */
  if (pState->file_stat.m_bit_flag &
      (MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_IS_ENCRYPTED |
       MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_USES_STRONG_ENCRYPTION |
       MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_COMPRESSED_PATCH_FLAG)) {
    mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_ENCRYPTION);
    pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
    return NULL;
  }

  /* 该函数仅支持解压存储和deflate压缩的文件 */
  if ((!(flags & MZ_ZIP_FLAG_COMPRESSED_DATA)) &&
      (pState->file_stat.m_method != 0) &&
      (pState->file_stat.m_method != MZ_DEFLATED)) {
    mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_METHOD);
    pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
    return NULL;
  }

  /* 初始化状态 - 保存参数 */
  pState->pZip = pZip;
  pState->flags = flags;

  /* 初始化状态 - 将变量重置为默认值 */
  pState->status = TINFL_STATUS_DONE;
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
  pState->file_crc32 = MZ_CRC32_INIT;
#endif
  // 重置读取缓冲区偏移量
  pState->read_buf_ofs = 0;
  // 重置输出缓冲区偏移量
  pState->out_buf_ofs = 0;
  // 重置读取缓冲区指针
  pState->pRead_buf = NULL;
  // 重置写入缓冲区指针
  pState->pWrite_buf = NULL;
  // 重置输出块剩余大小
  pState->out_blk_remain = 0;

  /* 读取并解析本地目录条目 */
  // 设置当前文件偏移量为本地头部的偏移量
  pState->cur_file_ofs = pState->file_stat.m_local_header_ofs;
  // 从 ZIP 文件中读取本地目录头部信息
  if (pZip->m_pRead(pZip->m_pIO_opaque, pState->cur_file_ofs, pLocal_header,
                    MZ_ZIP_LOCAL_DIR_HEADER_SIZE) !=
      MZ_ZIP_LOCAL_DIR_HEADER_SIZE) {
    // 设置 ZIP 错误为文件读取失败
    mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
    // 释放状态内存
    pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
    return NULL;
  }

  // 检查本地目录头部的签名是否正确
  if (MZ_READ_LE32(pLocal_header) != MZ_ZIP_LOCAL_DIR_HEADER_SIG) {
    // 设置 ZIP 错误为头部无效或损坏
    mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);
    // 释放状态内存
    pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
    return NULL;
  }

  // 更新当前文件偏移量
  pState->cur_file_ofs +=
      MZ_ZIP_LOCAL_DIR_HEADER_SIZE +
      MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_FILENAME_LEN_OFS) +
      MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_EXTRA_LEN_OFS);
  // 检查文件大小是否超出 ZIP 存档大小
  if ((pState->cur_file_ofs + pState->file_stat.m_comp_size) >
      pZip->m_archive_size) {
    // 设置 ZIP 错误为头部无效或损坏
    mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);
    // 释放状态内存
    pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
    return NULL;
  }

  /* 从内存或文件输入缓冲区直接解压文件 */
  if (pZip->m_pState->m_pMem) {
    // 设置读取缓冲区为当前文件偏移量处
    pState->pRead_buf =
        (mz_uint8 *)pZip->m_pState->m_pMem + pState->cur_file_ofs;
    // 设置读取缓冲区大小和可用大小为压缩文件大小
    pState->read_buf_size = pState->read_buf_avail =
        pState->file_stat.m_comp_size;
    // 设置剩余压缩数据大小
    pState->comp_remaining = pState->file_stat.m_comp_size;
  } else {
    // 检查是否需要解压缩数据，如果不需要或者文件的压缩方法为0，则不需要解压缩
    if (!((flags & MZ_ZIP_FLAG_COMPRESSED_DATA) ||
          (!pState->file_stat.m_method))) {
      /* Decompression required, therefore intermediate read buffer required */
      // 计算读取缓冲区的大小，取压缩大小和最大IO缓冲区大小的最小值
      pState->read_buf_size = MZ_MIN(pState->file_stat.m_comp_size,
                                     (mz_uint64)MZ_ZIP_MAX_IO_BUF_SIZE);
      // 分配读取缓冲区的内存空间
      if (NULL ==
          (pState->pRead_buf = pZip->m_pAlloc(pZip->m_pAlloc_opaque, 1,
                                              (size_t)pState->read_buf_size))) {
        // 分配内存失败，设置ZIP错误并释放内存
        mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
        pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
        return NULL;
      }
    } else {
      /* Decompression not required - we will be reading directly into user
       * buffer, no temp buf required */
      // 不需要解压缩，直接读取到用户缓冲区，不需要临时缓冲区
      pState->read_buf_size = 0;
    }
    // 重置读取缓冲区可用大小
    pState->read_buf_avail = 0;
    // 设置剩余需要解压缩的数据大小
    pState->comp_remaining = pState->file_stat.m_comp_size;
  }

  // 如果需要解压缩数据
  if (!((flags & MZ_ZIP_FLAG_COMPRESSED_DATA) ||
        (!pState->file_stat.m_method))) {
    /* Decompression required, init decompressor */
    // 初始化解压缩器
    tinfl_init(&pState->inflator);

    /* Allocate write buffer */
    // 分配写入缓冲区的内存空间
    if (NULL == (pState->pWrite_buf = pZip->m_pAlloc(pZip->m_pAlloc_opaque, 1,
                                                     TINFL_LZ_DICT_SIZE))) {
      // 分配内存失败，设置ZIP错误并释放内存
      mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
      // 如果读取缓冲区存在，则释放读取缓冲区内存
      if (pState->pRead_buf)
        pZip->m_pFree(pZip->m_pAlloc_opaque, pState->pRead_buf);
      // 释放状态内存
      pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
      return NULL;
    }
  }

  // 返回状态指针
  return pState;
}

/* 创建一个新的 ZIP 文件读取器迭代器状态 */
mz_zip_reader_extract_iter_state *
mz_zip_reader_extract_file_iter_new(mz_zip_archive *pZip, const char *pFilename,
                                    mz_uint flags) {
  mz_uint32 file_index;

  /* 通过文件名定位文件索引 */
  if (!mz_zip_reader_locate_file_v2(pZip, pFilename, NULL, flags, &file_index))
    return NULL;

  /* 构造迭代器 */
  return mz_zip_reader_extract_iter_new(pZip, file_index, flags);
}

/* 读取 ZIP 文件读取器迭代器状态中的数据 */
size_t mz_zip_reader_extract_iter_read(mz_zip_reader_extract_iter_state *pState,
                                       void *pvBuf, size_t buf_size) {
  size_t copied_to_caller = 0;

  /* 参数合法性检查 */
  if ((!pState) || (!pState->pZip) || (!pState->pZip->m_pState) || (!pvBuf))
    return 0;

  if ((pState->flags & MZ_ZIP_FLAG_COMPRESSED_DATA) ||
      (!pState->file_stat.m_method)) {
    /* 文件是存储的或者调用者请求压缩数据，计算要返回的数量 */
    copied_to_caller = (size_t)MZ_MIN(buf_size, pState->comp_remaining);

    /* ZIP 文件在内存中....还是需要从文件中读取？ */
    if (pState->pZip->m_pState->m_pMem) {
      /* 将数据复制到调用者的缓冲区中 */
      memcpy(pvBuf, pState->pRead_buf, copied_to_caller);
      pState->pRead_buf = ((mz_uint8 *)pState->pRead_buf) + copied_to_caller;
    } else {
      /* 直接读取到调用者的缓冲区中 */
      if (pState->pZip->m_pRead(pState->pZip->m_pIO_opaque,
                                pState->cur_file_ofs, pvBuf,
                                copied_to_caller) != copied_to_caller) {
        /* 无法读取所有请求的数据，标记失败并通知用户 */
        mz_zip_set_error(pState->pZip, MZ_ZIP_FILE_READ_FAILED);
        pState->status = TINFL_STATUS_FAILED;
        copied_to_caller = 0;
      }
    }

#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
    /* 如果不仅返回压缩数据，则计算 CRC */
    # 检查 pState 结构体中的 flags 是否包含 MZ_ZIP_FLAG_COMPRESSED_DATA 标志位
    if (!(pState->flags & MZ_ZIP_FLAG_COMPRESSED_DATA))
        # 如果不包含压缩数据标志位，则计算数据的 CRC32 校验值
        pState->file_crc32 = (mz_uint32)mz_crc32(
            pState->file_crc32, (const mz_uint8 *)pvBuf, copied_to_caller);
#endif

    /* Advance offsets, dec counters */
    // 增加偏移量，减少计数器
    pState->cur_file_ofs += copied_to_caller;
    pState->out_buf_ofs += copied_to_caller;
    pState->comp_remaining -= copied_to_caller;
  } else {
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
        /* Perform CRC */
        // 执行 CRC 校验
        pState->file_crc32 =
            (mz_uint32)mz_crc32(pState->file_crc32, pWrite_buf_cur, to_copy);
#endif

        /* Decrement data consumed from block */
        // 减少从块中消耗的数据
        pState->out_blk_remain -= to_copy;

        /* Inc output offset, while performing sanity check */
        // 增加输出偏移量，同时进行健全性检查
        if ((pState->out_buf_ofs += to_copy) >
            pState->file_stat.m_uncomp_size) {
          mz_zip_set_error(pState->pZip, MZ_ZIP_DECOMPRESSION_FAILED);
          pState->status = TINFL_STATUS_FAILED;
          break;
        }

        /* Increment counter of data copied to caller */
        // 增加传递给调用者的数据计数器
        copied_to_caller += to_copy;
      }
    } while ((copied_to_caller < buf_size) &&
             ((pState->status == TINFL_STATUS_NEEDS_MORE_INPUT) ||
              (pState->status == TINFL_STATUS_HAS_MORE_OUTPUT)));
  }

  /* Return how many bytes were copied into user buffer */
  // 返回拷贝到用户缓冲区的字节数
  return copied_to_caller;
}

mz_bool
mz_zip_reader_extract_iter_free(mz_zip_reader_extract_iter_state *pState) {
  int status;

  /* Argument sanity check */
  // 参数健全性检查
  if ((!pState) || (!pState->pZip) || (!pState->pZip->m_pState))
    return MZ_FALSE;

  /* Was decompression completed and requested? */
  // 解压是否已完成并请求？
  if ((pState->status == TINFL_STATUS_DONE) &&
      (!(pState->flags & MZ_ZIP_FLAG_COMPRESSED_DATA))) {
    /* Make sure the entire file was decompressed, and check its CRC. */
    // 确保整个文件已解压缩，并检查其 CRC
    if (pState->out_buf_ofs != pState->file_stat.m_uncomp_size) {
      mz_zip_set_error(pState->pZip, MZ_ZIP_UNEXPECTED_DECOMPRESSED_SIZE);
      pState->status = TINFL_STATUS_FAILED;
    }
#ifndef MINIZ_DISABLE_ZIP_READER_CRC32_CHECKS
    # 如果文件的 CRC32 校验值与文件状态中记录的 CRC32 校验值不相等，则执行以下操作
    else if (pState->file_crc32 != pState->file_stat.m_crc32) {
        # 设置 ZIP 文件的错误状态为解压失败
        mz_zip_set_error(pState->pZip, MZ_ZIP_DECOMPRESSION_FAILED);
        # 设置状态为解压失败
        pState->status = TINFL_STATUS_FAILED;
    }
#endif
  }

  /* 释放缓冲区 */
  if (!pState->pZip->m_pState->m_pMem)
    // 如果没有自定义内存分配器，则释放读取缓冲区
    pState->pZip->m_pFree(pState->pZip->m_pAlloc_opaque, pState->pRead_buf);
  if (pState->pWrite_buf)
    // 如果写入缓冲区存在，则释放写入缓冲区
    pState->pZip->m_pFree(pState->pZip->m_pAlloc_opaque, pState->pWrite_buf);

  /* 保存状态 */
  status = pState->status;

  /* 释放上下文 */
  // 释放状态结构体内存
  pState->pZip->m_pFree(pState->pZip->m_pAlloc_opaque, pState);

  // 返回状态是否为 TINFL_STATUS_DONE
  return status == TINFL_STATUS_DONE;
}

#ifndef MINIZ_NO_STDIO
static size_t mz_zip_file_write_callback(void *pOpaque, mz_uint64 ofs,
                                         const void *pBuf, size_t n) {
  (void)ofs;

  // 将数据写入文件
  return MZ_FWRITE(pBuf, 1, n, (MZ_FILE *)pOpaque);
}

// 从 ZIP 文件中提取文件到指定文件
mz_bool mz_zip_reader_extract_to_file(mz_zip_archive *pZip, mz_uint file_index,
                                      const char *pDst_filename,
                                      mz_uint flags) {
  mz_bool status;
  mz_zip_archive_file_stat file_stat;
  MZ_FILE *pFile;

  // 获取文件信息
  if (!mz_zip_reader_file_stat(pZip, file_index, &file_stat))
    return MZ_FALSE;

  // 如果是目录或不支持的文件类型，则返回错误
  if (file_stat.m_is_directory || (!file_stat.m_is_supported))
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_FEATURE);

  // 打开目标文件
  pFile = MZ_FOPEN(pDst_filename, "wb");
  if (!pFile)
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_OPEN_FAILED);

  // 提取文件到回调函数
  status = mz_zip_reader_extract_to_callback(
      pZip, file_index, mz_zip_file_write_callback, pFile, flags);

  // 关闭文件
  if (MZ_FCLOSE(pFile) == EOF) {
    if (status)
      mz_zip_set_error(pZip, MZ_ZIP_FILE_CLOSE_FAILED);

    status = MZ_FALSE;
  }

  // 设置文件时间
  if (status)
    mz_zip_set_file_times(pDst_filename, file_stat.m_time, file_stat.m_time);

  return status;
}
# 从 ZIP 归档中提取文件到文件
mz_bool mz_zip_reader_extract_file_to_file(mz_zip_archive *pZip,
                                           const char *pArchive_filename,
                                           const char *pDst_filename,
                                           mz_uint flags) {
  # 定义文件索引
  mz_uint32 file_index;
  # 定位要提取的文件在 ZIP 归档中的索引
  if (!mz_zip_reader_locate_file_v2(pZip, pArchive_filename, NULL, flags,
                                    &file_index))
    return MZ_FALSE;

  # 调用函数将文件提取到指定文件中
  return mz_zip_reader_extract_to_file(pZip, file_index, pDst_filename, flags);
}

# 从 ZIP 归档中提取到 C 文件
mz_bool mz_zip_reader_extract_to_cfile(mz_zip_archive *pZip, mz_uint file_index,
                                       MZ_FILE *pFile, mz_uint flags) {
  # 定义文件状态
  mz_zip_archive_file_stat file_stat;

  # 获取指定文件的文件状态
  if (!mz_zip_reader_file_stat(pZip, file_index, &file_stat))
    return MZ_FALSE;

  # 如果是目录或不支持的文件类型，则返回错误
  if (file_stat.m_is_directory || (!file_stat.m_is_supported))
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_FEATURE);

  # 调用函数将文件提取到回调函数中
  return mz_zip_reader_extract_to_callback(
      pZip, file_index, mz_zip_file_write_callback, pFile, flags);
}

# 从 ZIP 归档中提取文件到 C 文件
mz_bool mz_zip_reader_extract_file_to_cfile(mz_zip_archive *pZip,
                                            const char *pArchive_filename,
                                            MZ_FILE *pFile, mz_uint flags) {
  # 定义文件索引
  mz_uint32 file_index;
  # 定位要提取的文件在 ZIP 归档中的索引
  if (!mz_zip_reader_locate_file_v2(pZip, pArchive_filename, NULL, flags,
                                    &file_index))
    return MZ_FALSE;

  # 调用函数将文件提取到 C 文件中
  return mz_zip_reader_extract_to_cfile(pZip, file_index, pFile, flags);
}
#endif /* #ifndef MINIZ_NO_STDIO */

# 计算 CRC32 校验和的回调函数
static size_t mz_zip_compute_crc32_callback(void *pOpaque, mz_uint64 file_ofs,
                                            const void *pBuf, size_t n) {
  # 将 pOpaque 转换为 mz_uint32 指针
  mz_uint32 *p = (mz_uint32 *)pOpaque;
  # 忽略文件偏移量
  (void)file_ofs;
  # 计算 CRC32 校验和
  *p = (mz_uint32)mz_crc32(*p, (const mz_uint8 *)pBuf, n);
  return n;
}
  // 定义变量，用于存储文件的统计信息
  mz_zip_archive_file_stat file_stat;
  // 定义变量，用于存储 ZIP 内部状态
  mz_zip_internal_state *pState;
  // 定义变量，用于存储中央目录头部信息
  const mz_uint8 *pCentral_dir_header;
  // 初始化变量，用于标记是否在中央目录中找到 ZIP64 扩展数据
  mz_bool found_zip64_ext_data_in_cdir = MZ_FALSE;
  // 初始化变量，用于标记是否在本地目录中找到 ZIP64 扩展数据
  mz_bool found_zip64_ext_data_in_ldir = MZ_FALSE;
  // 初始化变量，用于存储本地头部信息
  mz_uint32 local_header_u32[(MZ_ZIP_LOCAL_DIR_HEADER_SIZE + sizeof(mz_uint32) - 1) / sizeof(mz_uint32)];
  // 将本地头部信息转换为字节流
  mz_uint8 *pLocal_header = (mz_uint8 *)local_header_u32;
  // 初始化变量，用于存储本地头部的偏移量
  mz_uint64 local_header_ofs = 0;
  // 初始化变量，用于存储本地头部的文件名长度、额外字段长度和 CRC32 校验值
  mz_uint32 local_header_filename_len, local_header_extra_len, local_header_crc32;
  // 初始化变量，用于存储本地头部的压缩大小和解压大小
  mz_uint64 local_header_comp_size, local_header_uncomp_size;
  // 初始化变量，用于存储未压缩数据的 CRC32 校验值
  mz_uint32 uncomp_crc32 = MZ_CRC32_INIT;
  // 初始化变量，用于标记是否存在数据描述符
  mz_bool has_data_descriptor;
  // 初始化变量，用于存储本地头部的位标志
  mz_uint32 local_header_bit_flags;

  // 初始化文件数据数组
  mz_zip_array file_data_array;
  mz_zip_array_init(&file_data_array, 1);

  // 检查参数是否有效
  if ((!pZip) || (!pZip->m_pState) || (!pZip->m_pAlloc) || (!pZip->m_pFree) ||
      (!pZip->m_pRead))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 检查文件索引是否超出总文件数范围
  if (file_index > pZip->m_total_files)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 获取 ZIP 内部状态
  pState = pZip->m_pState;

  // 获取中央目录头部信息
  pCentral_dir_header = mz_zip_get_cdh(pZip, file_index);

  // 获取文件统计信息，检查是否找到 ZIP64 扩展数据
  if (!mz_zip_file_stat_internal(pZip, file_index, pCentral_dir_header,
                                 &file_stat, &found_zip64_ext_data_in_cdir))
    return MZ_FALSE;

  /* A directory or zero length file */
  // 如果是目录或者文件长度为零，则返回真
  if (file_stat.m_is_directory || (!file_stat.m_uncomp_size))
    return MZ_TRUE;

  /* Encryption and patch files are not supported. */
  // 不支持加密和补丁文件
  if (file_stat.m_is_encrypted)
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_ENCRYPTION);

  /* This function only supports stored and deflate. */
  // 该函数仅支持存储和压缩方法
  if ((file_stat.m_method != 0) && (file_stat.m_method != MZ_DEFLATED))
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_METHOD);

  // 如果文件不受支持，则返回
    // 设置 ZIP 对象的错误信息为不支持的特性
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_FEATURE);

  /* Read and parse the local directory entry. */
  // 获取本地目录条目的偏移量
  local_header_ofs = file_stat.m_local_header_ofs;
  // 通过 ZIP 对象的读取函数读取本地头部信息
  if (pZip->m_pRead(pZip->m_pIO_opaque, local_header_ofs, pLocal_header,
                    MZ_ZIP_LOCAL_DIR_HEADER_SIZE) !=
      MZ_ZIP_LOCAL_DIR_HEADER_SIZE)
    // 设置 ZIP 对象的错误信息为文件读取失败
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);

  // 检查本地头部信息的签名是否正确
  if (MZ_READ_LE32(pLocal_header) != MZ_ZIP_LOCAL_DIR_HEADER_SIG)
    // 设置 ZIP 对象的错误信息为头部无效或损坏
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 读取本地头部信息中的各项数据
  local_header_filename_len =
      MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_FILENAME_LEN_OFS);
  local_header_extra_len =
      MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_EXTRA_LEN_OFS);
  local_header_comp_size =
      MZ_READ_LE32(pLocal_header + MZ_ZIP_LDH_COMPRESSED_SIZE_OFS);
  local_header_uncomp_size =
      MZ_READ_LE32(pLocal_header + MZ_ZIP_LDH_DECOMPRESSED_SIZE_OFS);
  local_header_crc32 = MZ_READ_LE32(pLocal_header + MZ_ZIP_LDH_CRC32_OFS);
  local_header_bit_flags =
      MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_BIT_FLAG_OFS);
  has_data_descriptor = (local_header_bit_flags & 8) != 0;

  // 检查文件名长度是否与文件状态中的文件名长度相同
  if (local_header_filename_len != strlen(file_stat.m_filename))
    // 设置 ZIP 对象的错误信息为头部无效或损坏
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 检查文件数据的大小是否超出 ZIP 存档的大小
  if ((local_header_ofs + MZ_ZIP_LOCAL_DIR_HEADER_SIZE +
       local_header_filename_len + local_header_extra_len +
       file_stat.m_comp_size) > pZip->m_archive_size)
    // 设置 ZIP 对象的错误信息为头部无效或损坏
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 调整文件数据数组的大小
  if (!mz_zip_array_resize(
          pZip, &file_data_array,
          MZ_MAX(local_header_filename_len, local_header_extra_len),
          MZ_FALSE)) {
    // 设置 ZIP 对象的错误信息为内存分配失败
    mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    // 跳转到处理失败的标签
    goto handle_failure;
  }

  // 如果文件名长度不为零
  if (local_header_filename_len) {
    // 如果读取的文件名长度与本地头部文件名长度不一致，则设置 ZIP 错误并跳转到错误处理
    if (pZip->m_pRead(pZip->m_pIO_opaque,
                      local_header_ofs + MZ_ZIP_LOCAL_DIR_HEADER_SIZE,
                      file_data_array.m_p,
                      local_header_filename_len) != local_header_filename_len) {
      mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
      goto handle_failure;
    }

    /* 我见过一个存档，它具有相同的路径名，但在本地目录中使用反斜杠，在中央目录中使用正斜杠。我们关心这个吗？
     * 目前，这种情况将导致验证失败。 */
    // 如果本地头部文件名与文件数据数组中的文件名不匹配，则设置 ZIP 错误并跳转到错误处理
    if (memcmp(file_stat.m_filename, file_data_array.m_p,
               local_header_filename_len) != 0) {
      mz_zip_set_error(pZip, MZ_ZIP_VALIDATION_FAILED);
      goto handle_failure;
    }
  }

  // 如果本地头部额外长度存在，并且压缩大小或未压缩大小为 MZ_UINT32_MAX
  if ((local_header_extra_len) &&
      ((local_header_comp_size == MZ_UINT32_MAX) ||
       (local_header_uncomp_size == MZ_UINT32_MAX))) {
    // 计算额外数据剩余大小，并获取额外数据指针
    mz_uint32 extra_size_remaining = local_header_extra_len;
    const mz_uint8 *pExtra_data = (const mz_uint8 *)file_data_array.m_p;

    // 如果读取的额外数据长度与本地头部额外数据长度不一致，则设置 ZIP 错误并跳转到错误处理
    if (pZip->m_pRead(pZip->m_pIO_opaque,
                      local_header_ofs + MZ_ZIP_LOCAL_DIR_HEADER_SIZE +
                          local_header_filename_len,
                      file_data_array.m_p,
                      local_header_extra_len) != local_header_extra_len) {
      mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
      goto handle_failure;
    }
    // 使用 do-while 循环处理额外数据字段
    do {
      // 定义字段 ID、字段数据大小、字段总大小
      mz_uint32 field_id, field_data_size, field_total_size;

      // 如果剩余额外数据大小小于两个 mz_uint16 大小，则标记为无效头或损坏
      if (extra_size_remaining < (sizeof(mz_uint16) * 2)) {
        mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);
        goto handle_failure;
      }

      // 读取字段 ID、字段数据大小、字段总大小
      field_id = MZ_READ_LE16(pExtra_data);
      field_data_size = MZ_READ_LE16(pExtra_data + sizeof(mz_uint16));
      field_total_size = field_data_size + sizeof(mz_uint16) * 2;

      // 如果字段总大小大于剩余额外数据大小，则标记为无效头或损坏
      if (field_total_size > extra_size_remaining) {
        mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);
        goto handle_failure;
      }

      // 如果字段 ID 为 ZIP64 扩展信息字段头 ID
      if (field_id == MZ_ZIP64_EXTENDED_INFORMATION_FIELD_HEADER_ID) {
        // 获取字段数据起始位置
        const mz_uint8 *pSrc_field_data = pExtra_data + sizeof(mz_uint32);

        // 如果字段数据大小小于两个 mz_uint64 大小，则标记为无效头或损坏
        if (field_data_size < sizeof(mz_uint64) * 2) {
          mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);
          goto handle_failure;
        }

        // 读取本地头无压缩大小和压缩大小
        local_header_uncomp_size = MZ_READ_LE64(pSrc_field_data);
        local_header_comp_size =
            MZ_READ_LE64(pSrc_field_data + sizeof(mz_uint64));

        // 标记找到 ZIP64 扩展数据在本地目录记录中
        found_zip64_ext_data_in_ldir = MZ_TRUE;
        break;
      }

      // 移动指针到下一个字段数据位置，更新剩余额外数据大小
      pExtra_data += field_total_size;
      extra_size_remaining -= field_total_size;
    } while (extra_size_remaining);
  }

  // 当本地头压缩大小为 0xFFFFFFFF 时，解析本地头额外数据
  // 在野外发现数据描述符位被设置，但本地头值正确且数据描述符错误的 ZIP 文件
  if ((has_data_descriptor) && (!local_header_comp_size) &&
      (!local_header_crc32)) {
    // 定义描述符缓冲区、是否有 ID、源指针、文件 CRC32、压缩大小、无压缩大小
    mz_uint8 descriptor_buf[32];
    mz_bool has_id;
    const mz_uint8 *pSrc;
    mz_uint32 file_crc32;
    mz_uint64 comp_size = 0, uncomp_size = 0;

    // 计算描述符中的 uint32 数量
    mz_uint32 num_descriptor_uint32s =
        ((pState->m_zip64) || (found_zip64_ext_data_in_ldir)) ? 6 : 4;
    // 检查文件数据是否正确读取，如果读取失败则设置 ZIP 错误并跳转到错误处理
    if (pZip->m_pRead(pZip->m_pIO_opaque,
                      local_header_ofs + MZ_ZIP_LOCAL_DIR_HEADER_SIZE +
                          local_header_filename_len + local_header_extra_len +
                          file_stat.m_comp_size,
                      descriptor_buf,
                      sizeof(mz_uint32) * num_descriptor_uint32s) !=
        (sizeof(mz_uint32) * num_descriptor_uint32s)) {
      mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
      goto handle_failure;
    }

    // 检查是否存在数据描述符
    has_id = (MZ_READ_LE32(descriptor_buf) == MZ_ZIP_DATA_DESCRIPTOR_ID);
    pSrc = has_id ? (descriptor_buf + sizeof(mz_uint32)) : descriptor_buf;

    // 读取文件的 CRC32 值
    file_crc32 = MZ_READ_LE32(pSrc);

    // 根据 ZIP64 标志或者在本地文件头中找到 ZIP64 扩展数据，读取压缩大小和未压缩大小
    if ((pState->m_zip64) || (found_zip64_ext_data_in_ldir)) {
      comp_size = MZ_READ_LE64(pSrc + sizeof(mz_uint32));
      uncomp_size = MZ_READ_LE64(pSrc + sizeof(mz_uint32) + sizeof(mz_uint64));
    } else {
      comp_size = MZ_READ_LE32(pSrc + sizeof(mz_uint32));
      uncomp_size = MZ_READ_LE32(pSrc + sizeof(mz_uint32) + sizeof(mz_uint32));
    }

    // 检查文件的 CRC32 值、压缩大小和未压缩大小是否与文件状态匹配，如果不匹配则设置 ZIP 错误并跳转到错误处理
    if ((file_crc32 != file_stat.m_crc32) ||
        (comp_size != file_stat.m_comp_size) ||
        (uncomp_size != file_stat.m_uncomp_size)) {
      mz_zip_set_error(pZip, MZ_ZIP_VALIDATION_FAILED);
      goto handle_failure;
    }
  } else {
    // 如果没有数据描述符，则检查本地文件头中的 CRC32 值、压缩大小和未压缩大小是否与文件状态匹配，如果不匹配则设置 ZIP 错误并跳转到错误处理
    if ((local_header_crc32 != file_stat.m_crc32) ||
        (local_header_comp_size != file_stat.m_comp_size) ||
        (local_header_uncomp_size != file_stat.m_uncomp_size)) {
      mz_zip_set_error(pZip, MZ_ZIP_VALIDATION_FAILED);
      goto handle_failure;
    }
  }

  // 清空文件数据数组
  mz_zip_array_clear(pZip, &file_data_array);

  // 如果不是仅验证头部标志，则提取文件数据到回调函数中
  if ((flags & MZ_ZIP_FLAG_VALIDATE_HEADERS_ONLY) == 0) {
    if (!mz_zip_reader_extract_to_callback(
            pZip, file_index, mz_zip_compute_crc32_callback, &uncomp_crc32, 0))
      return MZ_FALSE;

    /* 1 more check to be sure, although the extract checks too. */
  }
    # 如果解压后的 CRC32 校验值与文件状态中的 CRC32 校验值不相等
    if (uncomp_crc32 != file_stat.m_crc32) {
      # 设置 ZIP 错误为验证失败
      mz_zip_set_error(pZip, MZ_ZIP_VALIDATION_FAILED);
      # 返回假值
      return MZ_FALSE;
    }
  }

  # 返回真值
  return MZ_TRUE;
handle_failure:
  # 清空文件数据数组并返回失败标志
  mz_zip_array_clear(pZip, &file_data_array);
  return MZ_FALSE;
}

# 验证 ZIP 存档的有效性
mz_bool mz_zip_validate_archive(mz_zip_archive *pZip, mz_uint flags) {
  mz_zip_internal_state *pState;
  uint32_t i;

  # 检查参数是否有效
  if ((!pZip) || (!pZip->m_pState) || (!pZip->m_pAlloc) || (!pZip->m_pFree) ||
      (!pZip->m_pRead))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  pState = pZip->m_pState;

  /* 基本的合法性检查 */
  if (!pState->m_zip64) {
    if (pZip->m_total_files > MZ_UINT16_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);

    if (pZip->m_archive_size > MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);
  } else {
    if (pZip->m_total_files >= MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);

    if (pState->m_central_dir.m_size >= MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);
  }

  for (i = 0; i < pZip->m_total_files; i++) {
    if (MZ_ZIP_FLAG_VALIDATE_LOCATE_FILE_FLAG & flags) {
      mz_uint32 found_index;
      mz_zip_archive_file_stat stat;

      if (!mz_zip_reader_file_stat(pZip, i, &stat))
        return MZ_FALSE;

      if (!mz_zip_reader_locate_file_v2(pZip, stat.m_filename, NULL, 0,
                                        &found_index))
        return MZ_FALSE;

      /* 如果存档中存在重复的文件名，则此检查可能失败（在写入时我们不检查这一点 - 这取决于用户） */
      if (found_index != i)
        return mz_zip_set_error(pZip, MZ_ZIP_VALIDATION_FAILED);
    }

    if (!mz_zip_validate_file(pZip, i, flags))
      return MZ_FALSE;
  }

  return MZ_TRUE;
}

# 验证内存中的 ZIP 存档
mz_bool mz_zip_validate_mem_archive(const void *pMem, size_t size,
                                    mz_uint flags, mz_zip_error *pErr) {
  mz_bool success = MZ_TRUE;
  mz_zip_archive zip;
  mz_zip_error actual_err = MZ_ZIP_NO_ERROR;

  if ((!pMem) || (!size)) {
    if (pErr)
      *pErr = MZ_ZIP_INVALID_PARAMETER;
    # 返回一个表示失败的标志
    return MZ_FALSE;
  }

  # 初始化一个 ZIP 结构体
  mz_zip_zero_struct(&zip);

  # 使用内存中的数据初始化 ZIP 读取器
  if (!mz_zip_reader_init_mem(&zip, pMem, size, flags)) {
    # 如果初始化失败，将错误信息存入 pErr，并返回失败标志
    if (pErr)
      *pErr = zip.m_last_error;
    return MZ_FALSE;
  }

  # 验证 ZIP 存档的有效性
  if (!mz_zip_validate_archive(&zip, flags)) {
    # 如果验证失败，记录实际错误信息并设置成功标志为失败
    actual_err = zip.m_last_error;
    success = MZ_FALSE;
  }

  # 结束 ZIP 读取器的内部操作
  if (!mz_zip_reader_end_internal(&zip, success)) {
    # 如果结束操作失败，且实际错误信息为空，将 ZIP 对象的最后错误信息存入实际错误信息
    if (!actual_err)
      actual_err = zip.m_last_error;
    success = MZ_FALSE;
  }

  # 将实际错误信息存入 pErr
  if (pErr)
    *pErr = actual_err;

  # 返回操作是否成功的标志
  return success;
// 如果定义了 MINIZ_NO_STDIO，则定义一个函数用于验证 ZIP 文件的有效性
#ifndef MINIZ_NO_STDIO
mz_bool mz_zip_validate_file_archive(const char *pFilename, mz_uint flags,
                                     mz_zip_error *pErr) {
  mz_bool success = MZ_TRUE; // 初始化成功标志为真
  mz_zip_archive zip; // 定义一个 ZIP 归档对象
  mz_zip_error actual_err = MZ_ZIP_NO_ERROR; // 初始化实际错误为无错误

  if (!pFilename) { // 如果文件名为空
    if (pErr)
      *pErr = MZ_ZIP_INVALID_PARAMETER; // 设置错误为无效参数
    return MZ_FALSE; // 返回失败
  }

  mz_zip_zero_struct(&zip); // 将 ZIP 对象清零

  if (!mz_zip_reader_init_file_v2(&zip, pFilename, flags, 0, 0)) { // 初始化 ZIP 读取器
    if (pErr)
      *pErr = zip.m_last_error; // 设置错误为 ZIP 对象的最后一个错误
    return MZ_FALSE; // 返回失败
  }

  if (!mz_zip_validate_archive(&zip, flags)) { // 验证 ZIP 归档
    actual_err = zip.m_last_error; // 设置实际错误为 ZIP 对象的最后一个错误
    success = MZ_FALSE; // 设置成功标志为假
  }

  if (!mz_zip_reader_end_internal(&zip, success)) { // 结束 ZIP 读取器
    if (!actual_err)
      actual_err = zip.m_last_error; // 如果实际错误为空，则设置为 ZIP 对象的最后一个错误
    success = MZ_FALSE; // 设置成功标志为假
  }

  if (pErr)
    *pErr = actual_err; // 设置错误为实际错误

  return success; // 返回成功标志
}
#endif /* #ifndef MINIZ_NO_STDIO */

/* ------------------- .ZIP archive writing */

#ifndef MINIZ_NO_ARCHIVE_WRITING_APIS

// 写入小端格式的 16 位整数
static MZ_FORCEINLINE void mz_write_le16(mz_uint8 *p, mz_uint16 v) {
  p[0] = (mz_uint8)v; // 写入低位字节
  p[1] = (mz_uint8)(v >> 8); // 写入高位字节
}
// 写入小端格式的 32 位整数
static MZ_FORCEINLINE void mz_write_le32(mz_uint8 *p, mz_uint32 v) {
  p[0] = (mz_uint8)v; // 写入第一个字节
  p[1] = (mz_uint8)(v >> 8); // 写入第二个字节
  p[2] = (mz_uint8)(v >> 16); // 写入第三个字节
  p[3] = (mz_uint8)(v >> 24); // 写入第四个字节
}
// 写入小端格式的 64 位整数
static MZ_FORCEINLINE void mz_write_le64(mz_uint8 *p, mz_uint64 v) {
  mz_write_le32(p, (mz_uint32)v); // 写入低 32 位
  mz_write_le32(p + sizeof(mz_uint32), (mz_uint32)(v >> 32)); // 写入高 32 位
}

// 定义宏用于写入小端格式的 16 位整数
#define MZ_WRITE_LE16(p, v) mz_write_le16((mz_uint8 *)(p), (mz_uint16)(v))
// 定义宏用于写入小端格式的 32 位整数
#define MZ_WRITE_LE32(p, v) mz_write_le32((mz_uint8 *)(p), (mz_uint32)(v))
// 定义宏用于写入小端格式的 64 位整数
#define MZ_WRITE_LE64(p, v) mz_write_le64((mz_uint8 *)(p), (mz_uint64)(v))

// 写入函数，用于堆写入
static size_t mz_zip_heap_write_func(void *pOpaque, mz_uint64 file_ofs,
                                     const void *pBuf, size_t n) {
  mz_zip_archive *pZip = (mz_zip_archive *)pOpaque; // 强制转换为 ZIP 归档对象指针
  mz_zip_internal_state *pState = pZip->m_pState; // 获取 ZIP 内部状态
  mz_uint64 new_size = MZ_MAX(file_ofs + n, pState->m_mem_size); // 计算新的大小

  if (!n) // 如果写入长度为 0
    return 0;

  /* An allocation this big is likely to just fail on 32-bit systems, so don't
   * even go there. */
  // 如果分配这么大的内存在32位系统上很可能会失败，所以不要尝试
  if ((sizeof(size_t) == sizeof(mz_uint32)) && (new_size > 0x7FFFFFFF)) {
    // 如果新的大小超过了0x7FFFFFFF，设置 ZIP 错误为文件太大
    mz_zip_set_error(pZip, MZ_ZIP_FILE_TOO_LARGE);
    return 0;
  }

  if (new_size > pState->m_mem_capacity) {
    void *pNew_block;
    size_t new_capacity = MZ_MAX(64, pState->m_mem_capacity);

    while (new_capacity < new_size)
      new_capacity *= 2;

    if (NULL == (pNew_block = pZip->m_pRealloc(
                     pZip->m_pAlloc_opaque, pState->m_pMem, 1, new_capacity))) {
      // 如果重新分配内存失败，设置 ZIP 错误为分配失败
      mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
      return 0;
    }

    pState->m_pMem = pNew_block;
    pState->m_mem_capacity = new_capacity;
  }
  // 将缓冲区中的数据复制到内存块中的指定位置
  memcpy((mz_uint8 *)pState->m_pMem + file_ofs, pBuf, n);
  // 设置内存块的大小为新的大小
  pState->m_mem_size = (size_t)new_size;
  return n;
}

// 结束 ZIP 写入操作的内部函数
static mz_bool mz_zip_writer_end_internal(mz_zip_archive *pZip,
                                          mz_bool set_last_error) {
  mz_zip_internal_state *pState;
  mz_bool status = MZ_TRUE;

  // 检查参数有效性
  if ((!pZip) || (!pZip->m_pState) || (!pZip->m_pAlloc) || (!pZip->m_pFree) ||
      ((pZip->m_zip_mode != MZ_ZIP_MODE_WRITING) &&
       (pZip->m_zip_mode != MZ_ZIP_MODE_WRITING_HAS_BEEN_FINALIZED))) {
    // 设置错误信息并返回 False
    if (set_last_error)
      mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
    return MZ_FALSE;
  }

  // 保存当前状态，清空中央目录相关数组
  pState = pZip->m_pState;
  pZip->m_pState = NULL;
  mz_zip_array_clear(pZip, &pState->m_central_dir);
  mz_zip_array_clear(pZip, &pState->m_central_dir_offsets);
  mz_zip_array_clear(pZip, &pState->m_sorted_central_dir_offsets);

  // 关闭文件
#ifndef MINIZ_NO_STDIO
  if (pState->m_pFile) {
    if (pZip->m_zip_type == MZ_ZIP_TYPE_FILE) {
      if (MZ_FCLOSE(pState->m_pFile) == EOF) {
        if (set_last_error)
          mz_zip_set_error(pZip, MZ_ZIP_FILE_CLOSE_FAILED);
        status = MZ_FALSE;
      }
    }

    pState->m_pFile = NULL;
  }
#endif /* #ifndef MINIZ_NO_STDIO */

  // 释放内存
  if ((pZip->m_pWrite == mz_zip_heap_write_func) && (pState->m_pMem)) {
    pZip->m_pFree(pZip->m_pAlloc_opaque, pState->m_pMem);
    pState->m_pMem = NULL;
  }

  // 释放状态内存，设置 ZIP 模式为无效
  pZip->m_pFree(pZip->m_pAlloc_opaque, pState);
  pZip->m_zip_mode = MZ_ZIP_MODE_INVALID;
  return status;
}

// 初始化 ZIP 写入操作，支持 ZIP64
mz_bool mz_zip_writer_init_v2(mz_zip_archive *pZip, mz_uint64 existing_size,
                              mz_uint flags) {
  // 检查参数有效性
  mz_bool zip64 = (flags & MZ_ZIP_FLAG_WRITE_ZIP64) != 0;

  if ((!pZip) || (pZip->m_pState) || (!pZip->m_pWrite) ||
      (pZip->m_zip_mode != MZ_ZIP_MODE_INVALID))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 如果允许读取，检查读取函数是否存在
  if (flags & MZ_ZIP_FLAG_WRITE_ALLOW_READING) {
    if (!pZip->m_pRead)
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
  }

  // 如果用户指定了文件偏移对齐，确保是 2 的幂次方
    // 检查文件偏移对齐是否为2的幂，如果不是则返回无效参数错误
    if (pZip->m_file_offset_alignment & (pZip->m_file_offset_alignment - 1))
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
  }

  // 如果未指定内存分配函数，则使用默认的分配函数
  if (!pZip->m_pAlloc)
    pZip->m_pAlloc = miniz_def_alloc_func;
  // 如果未指定内存释放函数，则使用默认的释放函数
  if (!pZip->m_pFree)
    pZip->m_pFree = miniz_def_free_func;
  // 如果未指定内存重新分配函数，则使用默认的重新分配函数
  if (!pZip->m_pRealloc)
    pZip->m_pRealloc = miniz_def_realloc_func;

  // 设置存档大小为现有大小
  pZip->m_archive_size = existing_size;
  // 设置中央目录文件偏移为0
  pZip->m_central_directory_file_ofs = 0;
  // 设置总文件数为0
  pZip->m_total_files = 0;

  // 分配内部状态结构体的内存，如果分配失败则返回分配失败错误
  if (NULL == (pZip->m_pState = (mz_zip_internal_state *)pZip->m_pAlloc(
                   pZip->m_pAlloc_opaque, 1, sizeof(mz_zip_internal_state))))
    return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

  // 将内部状态结构体的内存清零
  memset(pZip->m_pState, 0, sizeof(mz_zip_internal_state));

  // 设置中央目录数组元素大小
  MZ_ZIP_ARRAY_SET_ELEMENT_SIZE(&pZip->m_pState->m_central_dir,
                                sizeof(mz_uint8));
  // 设置中央目录偏移数组元素大小
  MZ_ZIP_ARRAY_SET_ELEMENT_SIZE(&pZip->m_pState->m_central_dir_offsets,
                                sizeof(mz_uint32));
  // 设置排序后的中央目录偏移数组元素大小
  MZ_ZIP_ARRAY_SET_ELEMENT_SIZE(&pZip->m_pState->m_sorted_central_dir_offsets,
                                sizeof(mz_uint32));

  // 设置zip64标志
  pZip->m_pState->m_zip64 = zip64;
  // 设置zip64是否具有扩展信息字段的标志
  pZip->m_pState->m_zip64_has_extended_info_fields = zip64;

  // 设置ZIP类型为用户定义
  pZip->m_zip_type = MZ_ZIP_TYPE_USER;
  // 设置ZIP模式为写入模式
  pZip->m_zip_mode = MZ_ZIP_MODE_WRITING;

  // 返回成功
  return MZ_TRUE;
// 初始化 ZIP 写入器，使用默认的初始大小
mz_bool mz_zip_writer_init(mz_zip_archive *pZip, mz_uint64 existing_size) {
  // 调用带有默认标志的 mz_zip_writer_init_v2 函数
  return mz_zip_writer_init_v2(pZip, existing_size, 0);
}

// 初始化堆内存中的 ZIP 写入器，指定初始大小、初始分配大小和标志
mz_bool mz_zip_writer_init_heap_v2(mz_zip_archive *pZip,
                                   size_t size_to_reserve_at_beginning,
                                   size_t initial_allocation_size,
                                   mz_uint flags) {
  // 设置写入函数为 mz_zip_heap_write_func
  pZip->m_pWrite = mz_zip_heap_write_func;
  // 设置保持活动状态函数为 NULL
  pZip->m_pNeeds_keepalive = NULL;

  // 如果标志包含 MZ_ZIP_FLAG_WRITE_ALLOW_READING
  if (flags & MZ_ZIP_FLAG_WRITE_ALLOW_READING)
    // 设置读取函数为 mz_zip_mem_read_func
    pZip->m_pRead = mz_zip_mem_read_func;

  // 设置 IO 不透明指针为 pZip
  pZip->m_pIO_opaque = pZip;

  // 调用 mz_zip_writer_init_v2 函数初始化 ZIP 写入器
  if (!mz_zip_writer_init_v2(pZip, size_to_reserve_at_beginning, flags))
    return MZ_FALSE;

  // 设置 ZIP 类型为 MZ_ZIP_TYPE_HEAP
  pZip->m_zip_type = MZ_ZIP_TYPE_HEAP;

  // 如果初始分配大小不为 0
  if (0 != (initial_allocation_size = MZ_MAX(initial_allocation_size,
                                             size_to_reserve_at_beginning))) {
    // 分配内存给 pZip->m_pState->m_pMem
    if (NULL == (pZip->m_pState->m_pMem = pZip->m_pAlloc(
                     pZip->m_pAlloc_opaque, 1, initial_allocation_size))) {
      // 结束 ZIP 写入器
      mz_zip_writer_end_internal(pZip, MZ_FALSE);
      // 设置 ZIP 错误为 MZ_ZIP_ALLOC_FAILED
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }
    // 设置内存容量为初始分配大小
    pZip->m_pState->m_mem_capacity = initial_allocation_size;
  }

  return MZ_TRUE;
}

// 初始化堆内存中的 ZIP 写入器，使用默认标志
mz_bool mz_zip_writer_init_heap(mz_zip_archive *pZip,
                                size_t size_to_reserve_at_beginning,
                                size_t initial_allocation_size) {
  // 调用带有默认标志的 mz_zip_writer_init_heap_v2 函数
  return mz_zip_writer_init_heap_v2(pZip, size_to_reserve_at_beginning,
                                    initial_allocation_size, 0);
}

// 如果未定义 MINIZ_NO_STDIO
// 定义一个函数，用于将数据写入 ZIP 文件
static size_t mz_zip_file_write_func(void *pOpaque, mz_uint64 file_ofs,
                                     const void *pBuf, size_t n) {
  // 将传入的指针转换为 ZIP 归档对象
  mz_zip_archive *pZip = (mz_zip_archive *)pOpaque;
  // 获取当前文件偏移量
  mz_int64 cur_ofs = MZ_FTELL64(pZip->m_pState->m_pFile);

  // 计算文件偏移量
  file_ofs += pZip->m_pState->m_file_archive_start_ofs;

  // 检查文件偏移量是否合法，如果不合法则返回错误
  if (((mz_int64)file_ofs < 0) ||
      (((cur_ofs != (mz_int64)file_ofs)) &&
       (MZ_FSEEK64(pZip->m_pState->m_pFile, (mz_int64)file_ofs, SEEK_SET)))) {
    mz_zip_set_error(pZip, MZ_ZIP_FILE_SEEK_FAILED);
    return 0;
  }

  // 将数据写入文件
  return MZ_FWRITE(pBuf, 1, n, pZip->m_pState->m_pFile);
}

// 初始化 ZIP 归档对象，指定文件名和初始保留大小
mz_bool mz_zip_writer_init_file(mz_zip_archive *pZip, const char *pFilename,
                                mz_uint64 size_to_reserve_at_beginning) {
  // 调用带有额外参数的初始化函数
  return mz_zip_writer_init_file_v2(pZip, pFilename,
                                    size_to_reserve_at_beginning, 0);
}

// 初始化 ZIP 归档对象，指定文件名、初始保留大小和标志
mz_bool mz_zip_writer_init_file_v2(mz_zip_archive *pZip, const char *pFilename,
                                   mz_uint64 size_to_reserve_at_beginning,
                                   mz_uint flags) {
  MZ_FILE *pFile;

  // 设置写入函数
  pZip->m_pWrite = mz_zip_file_write_func;
  pZip->m_pNeeds_keepalive = NULL;

  // 如果标志允许读取，则设置读取函数
  if (flags & MZ_ZIP_FLAG_WRITE_ALLOW_READING)
    pZip->m_pRead = mz_zip_file_read_func;

  // 设置 IO 透明指针
  pZip->m_pIO_opaque = pZip;

  // 初始化 ZIP 归档对象
  if (!mz_zip_writer_init_v2(pZip, size_to_reserve_at_beginning, flags))
    return MZ_FALSE;

  // 打开文件进行写入
  if (NULL == (pFile = MZ_FOPEN(
                   pFilename,
                   (flags & MZ_ZIP_FLAG_WRITE_ALLOW_READING) ? "w+b" : "wb"))) {
    // 如果打开文件失败，则结束写入并返回错误
    mz_zip_writer_end(pZip);
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_OPEN_FAILED);
  }

  // 将文件指针设置到 ZIP 归档对象中
  pZip->m_pState->m_pFile = pFile;
  pZip->m_zip_type = MZ_ZIP_TYPE_FILE;

  // 如果有初始保留大小，则进行处理
  if (size_to_reserve_at_beginning) {
    mz_uint64 cur_ofs = 0;
    char buf[4096];

    // 清空缓冲区
    MZ_CLEAR_OBJ(buf);
    // 使用 do-while 循环来写入数据，直到 size_to_reserve_at_beginning 为 0
    do {
      // 计算每次写入的数据大小，取 buf 和 size_to_reserve_at_beginning 中较小的值
      size_t n = (size_t)MZ_MIN(sizeof(buf), size_to_reserve_at_beginning);
      // 调用 pZip 对象的写入函数，将数据写入到指定位置
      if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_ofs, buf, n) != n) {
        // 写入失败时，结束 ZIP 写入操作并返回写入失败错误
        mz_zip_writer_end(pZip);
        return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
      }
      // 更新当前偏移量
      cur_ofs += n;
      // 更新剩余需要写入的数据大小
      size_to_reserve_at_beginning -= n;
    } while (size_to_reserve_at_beginning);
  }

  // 写入完成后返回成功
  return MZ_TRUE;
}

// 初始化 ZIP 写入器，使用 C 文件指针
mz_bool mz_zip_writer_init_cfile(mz_zip_archive *pZip, MZ_FILE *pFile,
                                 mz_uint flags) {
  // 设置 ZIP 写入函数
  pZip->m_pWrite = mz_zip_file_write_func;
  // 设置 ZIP 保持连接函数为空
  pZip->m_pNeeds_keepalive = NULL;

  // 如果标志包含允许读取的写入标志
  if (flags & MZ_ZIP_FLAG_WRITE_ALLOW_READING)
    // 设置 ZIP 读取函数
    pZip->m_pRead = mz_zip_file_read_func;

  // 设置 ZIP 内部操作对象为当前 ZIP 对象
  pZip->m_pIO_opaque = pZip;

  // 初始化 ZIP 写入器，如果失败则返回假
  if (!mz_zip_writer_init_v2(pZip, 0, flags))
    return MZ_FALSE;

  // 设置 ZIP 内部状态的文件指针为传入的文件指针
  pZip->m_pState->m_pFile = pFile;
  // 记录 ZIP 内部状态的文件指针位置
  pZip->m_pState->m_file_archive_start_ofs =
      MZ_FTELL64(pZip->m_pState->m_pFile);
  // 设置 ZIP 类型为 C 文件类型
  pZip->m_zip_type = MZ_ZIP_TYPE_CFILE;

  // 返回真
  return MZ_TRUE;
}
#endif /* #ifndef MINIZ_NO_STDIO */

// 从读取器初始化 ZIP 写入器
mz_bool mz_zip_writer_init_from_reader_v2(mz_zip_archive *pZip,
                                          const char *pFilename,
                                          mz_uint flags) {
  mz_zip_internal_state *pState;

  // 检查参数有效性
  if ((!pZip) || (!pZip->m_pState) || (pZip->m_zip_mode != MZ_ZIP_MODE_READING))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 如果标志包含写入 ZIP64 标志
  if (flags & MZ_ZIP_FLAG_WRITE_ZIP64) {
    /* We don't support converting a non-zip64 file to zip64 - this seems like
     * more trouble than it's worth. (What about the existing 32-bit data
     * descriptors that could follow the compressed data?) */
    // 如果不支持将非 ZIP64 文件转换为 ZIP64，则返回错误
    if (!pZip->m_pState->m_zip64)
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
  }

  // 检查是否达到支持的最大文件数
  if (pZip->m_pState->m_zip64) {
    if (pZip->m_total_files == MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);
  } else {
    if (pZip->m_total_files == MZ_UINT16_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);

    if ((pZip->m_archive_size + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE +
         MZ_ZIP_LOCAL_DIR_HEADER_SIZE) > MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_TOO_LARGE);
  }

  // 获取 ZIP 内部状态
  pState = pZip->m_pState;

  // 如果 ZIP 内部状态的文件指针存在
  if (pState->m_pFile) {
#ifdef MINIZ_NO_STDIO
    (void)pFilename;
    # 返回一个表示 ZIP 参数无效的错误代码
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
#else
    // 如果 pZip->m_pIO_opaque 不等于 pZip，则返回无效参数错误
    if (pZip->m_pIO_opaque != pZip)
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

    // 如果 ZIP 类型为文件类型
    if (pZip->m_zip_type == MZ_ZIP_TYPE_FILE) {
      // 如果文件名为空，则返回无效参数错误
      if (!pFilename)
        return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

      /* 从 stdio 中读取归档，并且最初仅用于读取。尝试以可写方式重新打开。 */
      // 尝试以 "r+b" 方式重新打开文件
      if (NULL ==
          (pState->m_pFile = MZ_FREOPEN(pFilename, "r+b", pState->m_pFile))) {
        /* 由于 pState->m_pFile 为 NULL，mz_zip_archive 现在处于虚假状态，因此只需关闭它。 */
        // 关闭 ZIP 读取器
        mz_zip_reader_end_internal(pZip, MZ_FALSE);
        // 返回文件打开失败错误
        return mz_zip_set_error(pZip, MZ_ZIP_FILE_OPEN_FAILED);
      }
    }

    // 设置 ZIP 写入函数为 mz_zip_file_write_func
    pZip->m_pWrite = mz_zip_file_write_func;
    // 设置需要保持连接的函数为 NULL
    pZip->m_pNeeds_keepalive = NULL;
#endif /* #ifdef MINIZ_NO_STDIO */
  } else if (pState->m_pMem) {
    /* 存档位于内存块中。假设它来自可以使用 realloc 回调调整大小的堆。 */
    // 如果 pZip->m_pIO_opaque 不等于 pZip，则返回无效参数错误
    if (pZip->m_pIO_opaque != pZip)
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

    // 设置内存容量为内存大小
    pState->m_mem_capacity = pState->m_mem_size;
    // 设置 ZIP 写入函数为 mz_zip_heap_write_func
    pZip->m_pWrite = mz_zip_heap_write_func;
    // 设置需要保持连接的函数为 NULL
    pZip->m_pNeeds_keepalive = NULL;
  }
  /* 通过用户提供的读取函数读取存档 - 确保用户也指定了写入函数。 */
  else if (!pZip->m_pWrite)
    // 返回一个指示参数无效的错误代码
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  /* Start writing new files at the archive's current central directory
   * location. */
  /* TODO: We could add a flag that lets the user start writing immediately
   * AFTER the existing central dir - this would be safer. */
  // 将新文件的写入位置设置为存档当前的中央目录位置
  // TODO: 我们可以添加一个标志，让用户在现有中央目录之后立即开始写入，这样更安全。
  pZip->m_archive_size = pZip->m_central_directory_file_ofs;
  pZip->m_central_directory_file_ofs = 0;

  /* Clear the sorted central dir offsets, they aren't useful or maintained now.
   */
  /* Even though we're now in write mode, files can still be extracted and
   * verified, but file locates will be slow. */
  /* TODO: We could easily maintain the sorted central directory offsets. */
  // 清除排序后的中央目录偏移量，它们现在没有用且不再维护
  // 即使我们现在处于写入模式，文件仍然可以被提取和验证，但文件定位会很慢
  // TODO: 我们可以轻松地维护排序后的中央目录偏移量
  mz_zip_array_clear(pZip, &pZip->m_pState->m_sorted_central_dir_offsets);

  // 设置 ZIP 模式为写入模式
  pZip->m_zip_mode = MZ_ZIP_MODE_WRITING;

  // 返回真值表示成功
  return MZ_TRUE;
}
// 初始化 ZIP 写入器，从 ZIP 读取器中初始化，不重新打开文件
mz_bool mz_zip_writer_init_from_reader_v2_noreopen(mz_zip_archive *pZip,
                                                   const char *pFilename,
                                                   mz_uint flags) {
  mz_zip_internal_state *pState;

  // 检查参数是否有效
  if ((!pZip) || (!pZip->m_pState) || (pZip->m_zip_mode != MZ_ZIP_MODE_READING))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 如果设置了 ZIP64 标志，但 ZIP 对象不支持 ZIP64，则返回错误
  if (flags & MZ_ZIP_FLAG_WRITE_ZIP64) {
    /* We don't support converting a non-zip64 file to zip64 - this seems like
     * more trouble than it's worth. (What about the existing 32-bit data
     * descriptors that could follow the compressed data?) */
    if (!pZip->m_pState->m_zip64)
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
  }

  // 如果 ZIP 对象已经达到支持的最大大小，则返回错误
  if (pZip->m_pState->m_zip64) {
    if (pZip->m_total_files == MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);
  } else {
    if (pZip->m_total_files == MZ_UINT16_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);

    if ((pZip->m_archive_size + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE +
         MZ_ZIP_LOCAL_DIR_HEADER_SIZE) > MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_TOO_LARGE);
  }

  // 获取 ZIP 内部状态
  pState = pZip->m_pState;

  // 如果文件已经打开
  if (pState->m_pFile) {
#ifdef MINIZ_NO_STDIO
    (void)pFilename;
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
#else
    // 如果不支持标准输入输出，则返回错误
    if (pZip->m_pIO_opaque != pZip)
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

    // 如果 ZIP 类型是文件，但文件名为空，则返回错误
    if (pZip->m_zip_type == MZ_ZIP_TYPE_FILE) {
      if (!pFilename)
        return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
    }

    // 设置 ZIP 写入函数
    pZip->m_pWrite = mz_zip_file_write_func;
    pZip->m_pNeeds_keepalive = NULL;
#endif /* #ifdef MINIZ_NO_STDIO */
  } else if (pState->m_pMem) {
    /* Archive lives in a memory block. Assume it's from the heap that we can
     * resize using the realloc callback. */
    // 如果 ZIP 对象的 IO 指针不等于 ZIP 对象本身，则返回无效参数错误
    if (pZip->m_pIO_opaque != pZip)
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

    // 设置内存容量为当前内存大小
    pState->m_mem_capacity = pState->m_mem_size;
    // 设置 ZIP 对象的写入函数为堆写入函数
    pZip->m_pWrite = mz_zip_heap_write_func;
    // 设置 ZIP 对象的保持连接函数为空
    pZip->m_pNeeds_keepalive = NULL;
  }
  /* 如果通过用户提供的读取函数来读取存档，则确保用户也指定了写入函数。 */
  else if (!pZip->m_pWrite)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  /* 从存档的当前中央目录位置开始写入新文件。 */
  /* TODO: 我们可以添加一个标志，让用户可以在现有中央目录之后立即开始写入，这样更安全。 */
  pZip->m_archive_size = pZip->m_central_directory_file_ofs;
  pZip->m_central_directory_file_ofs = 0;

  /* 清除排序后的中央目录偏移量，它们现在没有用也不再维护。 */
  /* 即使我们现在处于写入模式，文件仍然可以被提取和验证，但文件定位将会很慢。 */
  /* TODO: 我们可以很容易地维护排序后的中央目录偏移量。 */
  mz_zip_array_clear(pZip, &pZip->m_pState->m_sorted_central_dir_offsets);

  // 设置 ZIP 对象的模式为写入模式
  pZip->m_zip_mode = MZ_ZIP_MODE_WRITING;

  // 返回真值
  return MZ_TRUE;
}

// 从读取器初始化 ZIP 写入器
mz_bool mz_zip_writer_init_from_reader(mz_zip_archive *pZip,
                                       const char *pFilename) {
  // 调用 mz_zip_writer_init_from_reader_v2 函数，传入 ZIP 对象和文件名，返回结果
  return mz_zip_writer_init_from_reader_v2(pZip, pFilename, 0);
}

/* TODO: pArchive_name is a terrible name here! */
// 向 ZIP 写入器添加内存数据
mz_bool mz_zip_writer_add_mem(mz_zip_archive *pZip, const char *pArchive_name,
                              const void *pBuf, size_t buf_size,
                              mz_uint level_and_flags) {
  // 调用 mz_zip_writer_add_mem_ex 函数，传入 ZIP 对象、文件名、数据指针、数据大小、其他参数，返回结果
  return mz_zip_writer_add_mem_ex(pZip, pArchive_name, pBuf, buf_size, NULL, 0,
                                  level_and_flags, 0, 0);
}

// 定义 ZIP 写入状态结构体
typedef struct {
  mz_zip_archive *m_pZip;
  mz_uint64 m_cur_archive_file_ofs;
  mz_uint64 m_comp_size;
} mz_zip_writer_add_state;

// ZIP 写入器添加数据的回调函数
static mz_bool mz_zip_writer_add_put_buf_callback(const void *pBuf, int len,
                                                  void *pUser) {
  // 将用户数据转换为 ZIP 写入状态结构体
  mz_zip_writer_add_state *pState = (mz_zip_writer_add_state *)pUser;
  // 调用 ZIP 对象的写入函数，写入数据到指定位置
  if ((int)pState->m_pZip->m_pWrite(pState->m_pZip->m_pIO_opaque,
                                    pState->m_cur_archive_file_ofs, pBuf,
                                    len) != len)
    return MZ_FALSE;

  // 更新当前文件偏移和压缩大小
  pState->m_cur_archive_file_ofs += len;
  pState->m_comp_size += len;
  return MZ_TRUE;
}

// 定义 ZIP64 额外数据字段的最大大小
#define MZ_ZIP64_MAX_LOCAL_EXTRA_FIELD_SIZE                                    \
  (sizeof(mz_uint16) * 2 + sizeof(mz_uint64) * 2)
#define MZ_ZIP64_MAX_CENTRAL_EXTRA_FIELD_SIZE                                  \
  (sizeof(mz_uint16) * 2 + sizeof(mz_uint64) * 3)
// 创建 ZIP64 额外数据
static mz_uint32
mz_zip_writer_create_zip64_extra_data(mz_uint8 *pBuf, mz_uint64 *pUncomp_size,
                                      mz_uint64 *pComp_size,
                                      mz_uint64 *pLocal_header_ofs) {
  mz_uint8 *pDst = pBuf;
  mz_uint32 field_size = 0;

  // 写入 ZIP64 扩展信息字段头部 ID 和长度
  MZ_WRITE_LE16(pDst + 0, MZ_ZIP64_EXTENDED_INFORMATION_FIELD_HEADER_ID);
  MZ_WRITE_LE16(pDst + 2, 0);
  pDst += sizeof(mz_uint16) * 2;

  if (pUncomp_size) {
    // 写入未压缩大小
    MZ_WRITE_LE64(pDst, *pUncomp_size);
  // 将指针 pDst 向后移动 sizeof(mz_uint64) 个字节
  pDst += sizeof(mz_uint64);
  // 更新 field_size 变量，增加 sizeof(mz_uint64) 个字节
  field_size += sizeof(mz_uint64);



  // 如果 pComp_size 不为空
  if (pComp_size) {
    // 将 pComp_size 指向的值以小端序写入 pDst
    MZ_WRITE_LE64(pDst, *pComp_size);
    // 将指针 pDst 向后移动 sizeof(mz_uint64) 个字节
    pDst += sizeof(mz_uint64);
    // 更新 field_size 变量，增加 sizeof(mz_uint64) 个字节
    field_size += sizeof(mz_uint64);
  }



  // 如果 pLocal_header_ofs 不为空
  if (pLocal_header_ofs) {
    // 将 pLocal_header_ofs 指向的值以小端序写入 pDst
    MZ_WRITE_LE64(pDst, *pLocal_header_ofs);
    // 将指针 pDst 向后移动 sizeof(mz_uint64) 个字节
    pDst += sizeof(mz_uint64);
    // 更新 field_size 变量，增加 sizeof(mz_uint64) 个字节
    field_size += sizeof(mz_uint64);
  }



  // 将 field_size 以小端序写入 pBuf + 2 的位置
  MZ_WRITE_LE16(pBuf + 2, field_size);



  // 返回 pDst 和 pBuf 之间的字节数差值，转换为 mz_uint32 类型
  return (mz_uint32)(pDst - pBuf);
// 创建本地目录头部信息，用于写入 ZIP 文件
static mz_bool mz_zip_writer_create_local_dir_header(
    mz_zip_archive *pZip, mz_uint8 *pDst, mz_uint16 filename_size,
    mz_uint16 extra_size, mz_uint64 uncomp_size, mz_uint64 comp_size,
    mz_uint32 uncomp_crc32, mz_uint16 method, mz_uint16 bit_flags,
    mz_uint16 dos_time, mz_uint16 dos_date) {
  // 忽略未使用的参数
  (void)pZip;
  // 将目标内存块清零
  memset(pDst, 0, MZ_ZIP_LOCAL_DIR_HEADER_SIZE);
  // 写入本地目录头部的标识符
  MZ_WRITE_LE32(pDst + MZ_ZIP_LDH_SIG_OFS, MZ_ZIP_LOCAL_DIR_HEADER_SIG);
  // 写入版本信息
  MZ_WRITE_LE16(pDst + MZ_ZIP_LDH_VERSION_NEEDED_OFS, method ? 20 : 0);
  // 写入位标志
  MZ_WRITE_LE16(pDst + MZ_ZIP_LDH_BIT_FLAG_OFS, bit_flags);
  // 写入压缩方法
  MZ_WRITE_LE16(pDst + MZ_ZIP_LDH_METHOD_OFS, method);
  // 写入文件时间
  MZ_WRITE_LE16(pDst + MZ_ZIP_LDH_FILE_TIME_OFS, dos_time);
  // 写入文件日期
  MZ_WRITE_LE16(pDst + MZ_ZIP_LDH_FILE_DATE_OFS, dos_date);
  // 写入未压缩数据的 CRC32 校验值
  MZ_WRITE_LE32(pDst + MZ_ZIP_LDH_CRC32_OFS, uncomp_crc32);
  // 写入压缩后的数据大小
  MZ_WRITE_LE32(pDst + MZ_ZIP_LDH_COMPRESSED_SIZE_OFS,
                MZ_MIN(comp_size, MZ_UINT32_MAX));
  // 写入未压缩数据大小
  MZ_WRITE_LE32(pDst + MZ_ZIP_LDH_DECOMPRESSED_SIZE_OFS,
                MZ_MIN(uncomp_size, MZ_UINT32_MAX));
  // 写入文件名长度
  MZ_WRITE_LE16(pDst + MZ_ZIP_LDH_FILENAME_LEN_OFS, filename_size);
  // 写入额外数据长度
  MZ_WRITE_LE16(pDst + MZ_ZIP_LDH_EXTRA_LEN_OFS, extra_size);
  // 返回操作成功
  return MZ_TRUE;
}

// 创建中央目录头部信息，用于写入 ZIP 文件
static mz_bool mz_zip_writer_create_central_dir_header(
    mz_zip_archive *pZip, mz_uint8 *pDst, mz_uint16 filename_size,
    mz_uint16 extra_size, mz_uint16 comment_size, mz_uint64 uncomp_size,
    mz_uint64 comp_size, mz_uint32 uncomp_crc32, mz_uint16 method,
    mz_uint16 bit_flags, mz_uint16 dos_time, mz_uint16 dos_date,
    // 设置 ZIP 中央目录头部的偏移和扩展属性
    mz_uint64 local_header_ofs, mz_uint32 ext_attributes) {
    // 忽略未使用的参数 pZip
    (void)pZip;
    // 将 pDst 内存块清零，大小为 MZ_ZIP_CENTRAL_DIR_HEADER_SIZE
    memset(pDst, 0, MZ_ZIP_CENTRAL_DIR_HEADER_SIZE);
    // 写入 ZIP 中央目录头部的标识符
    MZ_WRITE_LE32(pDst + MZ_ZIP_CDH_SIG_OFS, MZ_ZIP_CENTRAL_DIR_HEADER_SIG);
    // 写入 ZIP 中央目录头部的版本信息
    MZ_WRITE_LE16(pDst + MZ_ZIP_CDH_VERSION_NEEDED_OFS, method ? 20 : 0);
    // 写入 ZIP 中央目录头部的位标志
    MZ_WRITE_LE16(pDst + MZ_ZIP_CDH_BIT_FLAG_OFS, bit_flags);
    // 写入 ZIP 中央目录头部的压缩方法
    MZ_WRITE_LE16(pDst + MZ_ZIP_CDH_METHOD_OFS, method);
    // 写入 ZIP 中央目录头部的文件时间
    MZ_WRITE_LE16(pDst + MZ_ZIP_CDH_FILE_TIME_OFS, dos_time);
    // 写入 ZIP 中央目录头部的文件日期
    MZ_WRITE_LE16(pDst + MZ_ZIP_CDH_FILE_DATE_OFS, dos_date);
    // 写入 ZIP 中央目录头部的未压缩数据的 CRC32 校验值
    MZ_WRITE_LE32(pDst + MZ_ZIP_CDH_CRC32_OFS, uncomp_crc32);
    // 写入 ZIP 中央目录头部的压缩后数据的大小
    MZ_WRITE_LE32(pDst + MZ_ZIP_CDH_COMPRESSED_SIZE_OFS,
                  MZ_MIN(comp_size, MZ_UINT32_MAX));
    // 写入 ZIP 中央目录头部的未压缩数据的大小
    MZ_WRITE_LE32(pDst + MZ_ZIP_CDH_DECOMPRESSED_SIZE_OFS,
                  MZ_MIN(uncomp_size, MZ_UINT32_MAX));
    // 写入 ZIP 中央目录头部的文件名长度
    MZ_WRITE_LE16(pDst + MZ_ZIP_CDH_FILENAME_LEN_OFS, filename_size);
    // 写入 ZIP 中央目录头部的额外字段长度
    MZ_WRITE_LE16(pDst + MZ_ZIP_CDH_EXTRA_LEN_OFS, extra_size);
    // 写入 ZIP 中央目录头部的注释长度
    MZ_WRITE_LE16(pDst + MZ_ZIP_CDH_COMMENT_LEN_OFS, comment_size);
    // 写入 ZIP 中央目录头部的外部属性
    MZ_WRITE_LE32(pDst + MZ_ZIP_CDH_EXTERNAL_ATTR_OFS, ext_attributes);
    // 写入 ZIP 中央目录头部的本地文件头部的偏移
    MZ_WRITE_LE32(pDst + MZ_ZIP_CDH_LOCAL_HEADER_OFS,
                  MZ_MIN(local_header_ofs, MZ_UINT32_MAX));
    // 返回真值表示成功
    return MZ_TRUE;
# 静态函数，用于向 ZIP 文件的中央目录添加条目
static mz_bool mz_zip_writer_add_to_central_dir(
    mz_zip_archive *pZip, const char *pFilename, mz_uint16 filename_size,
    const void *pExtra, mz_uint16 extra_size, const void *pComment,
    mz_uint16 comment_size, mz_uint64 uncomp_size, mz_uint64 comp_size,
    mz_uint32 uncomp_crc32, mz_uint16 method, mz_uint16 bit_flags,
    mz_uint16 dos_time, mz_uint16 dos_date, mz_uint64 local_header_ofs,
    mz_uint32 ext_attributes, const char *user_extra_data,
    mz_uint user_extra_data_len) {
  # 获取 ZIP 文件的内部状态
  mz_zip_internal_state *pState = pZip->m_pState;
  # 计算中央目录的偏移量
  mz_uint32 central_dir_ofs = (mz_uint32)pState->m_central_dir.m_size;
  # 记录原始中央目录的大小
  size_t orig_central_dir_size = pState->m_central_dir.m_size;
  # 创建中央目录头部数据的缓冲区
  mz_uint8 central_dir_header[MZ_ZIP_CENTRAL_DIR_HEADER_SIZE];

  # 如果 ZIP 文件不支持 ZIP64 格式，并且本地头部的偏移量超过 0xFFFFFFFF，则返回错误
  if (!pZip->m_pState->m_zip64) {
    if (local_header_ofs > 0xFFFFFFFF)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_TOO_LARGE);
  }

  # 如果中央目录的大小超过 MZ_UINT32_MAX，则返回错误
  if (((mz_uint64)pState->m_central_dir.m_size +
       MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + filename_size + extra_size +
       user_extra_data_len + comment_size) >= MZ_UINT32_MAX)
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_CDIR_SIZE);

  # 创建中央目录头部数据
  if (!mz_zip_writer_create_central_dir_header(
          pZip, central_dir_header, filename_size,
          (mz_uint16)(extra_size + user_extra_data_len), comment_size,
          uncomp_size, comp_size, uncomp_crc32, method, bit_flags, dos_time,
          dos_date, local_header_ofs, ext_attributes))
    # 返回一个指定错误码的 ZIP 对象
    return mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);

  # 将中央目录头部信息、文件名、额外数据、用户额外数据、注释以及中央目录偏移量添加到中央目录数组中
  if ((!mz_zip_array_push_back(pZip, &pState->m_central_dir, central_dir_header,
                               MZ_ZIP_CENTRAL_DIR_HEADER_SIZE)) ||
      (!mz_zip_array_push_back(pZip, &pState->m_central_dir, pFilename,
                               filename_size)) ||
      (!mz_zip_array_push_back(pZip, &pState->m_central_dir, pExtra,
                               extra_size)) ||
      (!mz_zip_array_push_back(pZip, &pState->m_central_dir, user_extra_data,
                               user_extra_data_len)) ||
      (!mz_zip_array_push_back(pZip, &pState->m_central_dir, pComment,
                               comment_size)) ||
      (!mz_zip_array_push_back(pZip, &pState->m_central_dir_offsets,
                               &central_dir_ofs, 1))) {
    # 尝试将中央目录数组调整为原始状态
    mz_zip_array_resize(pZip, &pState->m_central_dir, orig_central_dir_size,
                        MZ_FALSE);
    # 返回一个指定错误码的 ZIP 对象
    return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
  }

  # 返回操作成功
  return MZ_TRUE;
}

/* 静态函数，用于验证 ZIP 存档名称的有效性 */
static mz_bool mz_zip_writer_validate_archive_name(const char *pArchive_name) {
  /* 基本的 ZIP 存档文件名有效性检查：有效的文件名不能以斜杠开头，不能包含驱动器号，也不能使用 DOS 风格的反斜杠 */
  if (*pArchive_name == '/')
    return MZ_FALSE;

  /* 确保名称不包含驱动器号或 DOS 风格的反斜杠是使用 miniz 的程序的责任 */

  return MZ_TRUE;
}

/* 计算文件对齐所需的填充字节数 */
static mz_uint
mz_zip_writer_compute_padding_needed_for_file_alignment(mz_zip_archive *pZip) {
  mz_uint32 n;
  if (!pZip->m_file_offset_alignment)
    return 0;
  n = (mz_uint32)(pZip->m_archive_size & (pZip->m_file_offset_alignment - 1));
  return (mz_uint)((pZip->m_file_offset_alignment - n) &
                   (pZip->m_file_offset_alignment - 1));
}

/* 写入零字节到 ZIP 存档 */
static mz_bool mz_zip_writer_write_zeros(mz_zip_archive *pZip,
                                         mz_uint64 cur_file_ofs, mz_uint32 n) {
  char buf[4096];
  memset(buf, 0, MZ_MIN(sizeof(buf), n));
  while (n) {
    mz_uint32 s = MZ_MIN(sizeof(buf), n);
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_file_ofs, buf, s) != s)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    cur_file_ofs += s;
    n -= s;
  }
  return MZ_TRUE;
}

/* 向 ZIP 存档中添加内存数据 */
mz_bool mz_zip_writer_add_mem_ex(mz_zip_archive *pZip,
                                 const char *pArchive_name, const void *pBuf,
                                 size_t buf_size, const void *pComment,
                                 mz_uint16 comment_size,
                                 mz_uint level_and_flags, mz_uint64 uncomp_size,
                                 mz_uint32 uncomp_crc32) {
  return mz_zip_writer_add_mem_ex_v2(
      pZip, pArchive_name, pBuf, buf_size, pComment, comment_size,
      level_and_flags, uncomp_size, uncomp_crc32, NULL, NULL, 0, NULL, 0);
}

/* 向 ZIP 存档中添加内存数据的扩展版本 */
mz_bool mz_zip_writer_add_mem_ex_v2(
    mz_zip_archive *pZip, const char *pArchive_name, const void *pBuf,
    # 定义变量 buf_size，pComment，comment_size，level_and_flags，uncomp_size，uncomp_crc32，last_modified，user_extra_data，user_extra_data_len，user_extra_data_central，user_extra_data_central_len
    size_t buf_size, const void *pComment, mz_uint16 comment_size,
    mz_uint level_and_flags, mz_uint64 uncomp_size, mz_uint32 uncomp_crc32,
    MZ_TIME_T *last_modified, const char *user_extra_data,
    mz_uint user_extra_data_len, const char *user_extra_data_central,
    mz_uint user_extra_data_central_len) {
  
  # 定义一系列变量
  mz_uint16 method = 0, dos_time = 0, dos_date = 0;
  mz_uint level, ext_attributes = 0, num_alignment_padding_bytes;
  mz_uint64 local_dir_header_ofs = 0, cur_archive_file_ofs = 0, comp_size = 0;
  size_t archive_name_size;
  mz_uint8 local_dir_header[MZ_ZIP_LOCAL_DIR_HEADER_SIZE];
  tdefl_compressor *pComp = NULL;
  mz_bool store_data_uncompressed;
  mz_zip_internal_state *pState;
  mz_uint8 *pExtra_data = NULL;
  mz_uint32 extra_size = 0;
  mz_uint8 extra_data[MZ_ZIP64_MAX_CENTRAL_EXTRA_FIELD_SIZE];
  mz_uint16 bit_flags = 0;

  # 如果 level_and_flags 小于 0，则将其设置为默认级别
  if ((int)level_and_flags < 0)
    level_and_flags = MZ_DEFAULT_LEVEL;

  # 如果 uncomp_size 不为 0 或者 buf_size 不为 0 且 level_and_flags 不包含 MZ_ZIP_FLAG_COMPRESSED_DATA，则设置 bit_flags
  if (uncomp_size ||
      (buf_size && !(level_and_flags & MZ_ZIP_FLAG_COMPRESSED_DATA)))
    bit_flags |= MZ_ZIP_LDH_BIT_FLAG_HAS_LOCATOR;

  # 如果 level_and_flags 不包含 MZ_ZIP_FLAG_ASCII_FILENAME，则设置 bit_flags
  if (!(level_and_flags & MZ_ZIP_FLAG_ASCII_FILENAME))
    bit_flags |= MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_UTF8;

  # 获取压缩级别
  level = level_and_flags & 0xF;
  store_data_uncompressed =
      ((!level) || (level_and_flags & MZ_ZIP_FLAG_COMPRESSED_DATA));

  # 检查参数是否有效，若无效则返回错误
  if ((!pZip) || (!pZip->m_pState) ||
      (pZip->m_zip_mode != MZ_ZIP_MODE_WRITING) || ((buf_size) && (!pBuf)) ||
      (!pArchive_name) || ((comment_size) && (!pComment)) ||
      (level > MZ_UBER_COMPRESSION))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  # 获取 ZIP 内部状态
  pState = pZip->m_pState;
  local_dir_header_ofs = pZip->m_archive_size;
  cur_archive_file_ofs = pZip->m_archive_size;

  # 如果 ZIP 文件数达到最大值，则返回错误
  if (pState->m_zip64) {
    if (pZip->m_total_files == MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);
  } else {
    if (pZip->m_total_files == MZ_UINT16_MAX) {
      pState->m_zip64 = MZ_TRUE;
      /*return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES); */
    }
    // 如果缓冲区大小或未压缩大小超过 0xFFFFFFFF，则设置 ZIP64 标志为真
    if ((buf_size > 0xFFFFFFFF) || (uncomp_size > 0xFFFFFFFF)) {
      pState->m_zip64 = MZ_TRUE;
      /*return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE); */
    }
  }

  // 如果未设置压缩数据标志并且未压缩大小不为零，则返回无效参数错误
  if ((!(level_and_flags & MZ_ZIP_FLAG_COMPRESSED_DATA)) && (uncomp_size))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 如果归档名称无效，则返回无效文件名错误
  if (!mz_zip_writer_validate_archive_name(pArchive_name))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_FILENAME);
#ifndef MINIZ_NO_TIME
  // 如果最后修改时间不为空，则将时间转换为 DOS 格式的时间和日期
  if (last_modified != NULL) {
    mz_zip_time_t_to_dos_time(*last_modified, &dos_time, &dos_date);
  } else {
    // 否则获取当前时间，并将时间转换为 DOS 格式的时间和日期
    MZ_TIME_T cur_time;
    time(&cur_time);
    mz_zip_time_t_to_dos_time(cur_time, &dos_time, &dos_date);
  }
#endif /* #ifndef MINIZ_NO_TIME */

  // 如果标志位中不包含压缩数据标志
  if (!(level_and_flags & MZ_ZIP_FLAG_COMPRESSED_DATA)) {
    // 计算未压缩数据的 CRC32 校验值
    uncomp_crc32 = (mz_uint32)mz_crc32(MZ_CRC32_INIT, (const mz_uint8 *)pBuf, buf_size);
    // 记录未压缩数据的大小
    uncomp_size = buf_size;
    // 如果未压缩数据大小小于等于3，则设置压缩级别为0，并标记数据未压缩
    if (uncomp_size <= 3) {
      level = 0;
      store_data_uncompressed = MZ_TRUE;
    }
  }

  // 计算归档文件名的长度
  archive_name_size = strlen(pArchive_name);
  // 如果归档文件名长度超过 MZ_UINT16_MAX，则返回无效文件名错误
  if (archive_name_size > MZ_UINT16_MAX)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_FILENAME);

  // 计算需要的字节对齐填充字节数
  num_alignment_padding_bytes = mz_zip_writer_compute_padding_needed_for_file_alignment(pZip);

  /* miniz 目前不支持中央目录大于 MZ_UINT32_MAX 字节 */
  if (((mz_uint64)pState->m_central_dir.m_size +
       MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + archive_name_size +
       MZ_ZIP64_MAX_CENTRAL_EXTRA_FIELD_SIZE + comment_size) >= MZ_UINT32_MAX)
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_CDIR_SIZE);

  // 如果不是 ZIP64 格式
  if (!pState->m_zip64) {
    // 如果归档文件变得太大，则标记为 ZIP64 格式
    if ((pZip->m_archive_size + num_alignment_padding_bytes +
         MZ_ZIP_LOCAL_DIR_HEADER_SIZE + archive_name_size +
         MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + archive_name_size + comment_size +
         user_extra_data_len + pState->m_central_dir.m_size +
         MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE + user_extra_data_central_len +
         MZ_ZIP_DATA_DESCRIPTER_SIZE32) > 0xFFFFFFFF) {
      pState->m_zip64 = MZ_TRUE;
      /*return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE); */
    }
  }

  // 如果归档文件名长度不为0且最后一个字符为 '/'
  if ((archive_name_size) && (pArchive_name[archive_name_size - 1] == '/')) {
    // 设置 DOS 子目录属性位
    ext_attributes |= MZ_ZIP_DOS_DIR_ATTRIBUTE_BITFLAG;

    // 子目录不能包含数据
    // 如果缓冲区大小或未压缩大小不为零，则返回无效参数错误
    if ((buf_size) || (uncomp_size))
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
  }

  /* 尝试在写入存档之前进行任何分配，以便如果分配失败，则文件保持不变。 （如果我们正在进行原地修改，这是一个好主意。） */
  if ((!mz_zip_array_ensure_room(
          pZip, &pState->m_central_dir,
          MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + archive_name_size + comment_size +
              (pState->m_zip64 ? MZ_ZIP64_MAX_CENTRAL_EXTRA_FIELD_SIZE : 0))) ||
      (!mz_zip_array_ensure_room(pZip, &pState->m_central_dir_offsets, 1)))
    return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

  // 如果不存储未压缩数据且缓冲区大小不为零
  if ((!store_data_uncompressed) && (buf_size)) {
    // 如果分配失败，则返回分配失败错误
    if (NULL == (pComp = (tdefl_compressor *)pZip->m_pAlloc(
                     pZip->m_pAlloc_opaque, 1, sizeof(tdefl_compressor))))
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
  }

  // 如果写入零失败
  if (!mz_zip_writer_write_zeros(pZip, cur_archive_file_ofs,
                                 num_alignment_padding_bytes)) {
    // 释放分配的内存并返回假
    pZip->m_pFree(pZip->m_pAlloc_opaque, pComp);
    return MZ_FALSE;
  }

  // 更新本地目录头偏移量
  local_dir_header_ofs += num_alignment_padding_bytes;
  // 如果文件偏移对齐值存在
  if (pZip->m_file_offset_alignment) {
    // 断言本地目录头偏移量符合文件偏移对齐值
    MZ_ASSERT((local_dir_header_ofs & (pZip->m_file_offset_alignment - 1)) ==
              0);
  }
  // 更新当前存档文件偏移量
  cur_archive_file_ofs += num_alignment_padding_bytes;

  // 清空本地目录头对象
  MZ_CLEAR_OBJ(local_dir_header);

  // 如果不存储未压缩数据或标志中包含压缩数据标志
  if (!store_data_uncompressed ||
      (level_and_flags & MZ_ZIP_FLAG_COMPRESSED_DATA)) {
    // 设置压缩方法为 DEFLATED
    method = MZ_DEFLATED;
  }

  // 如果 ZIP64 标志为真
  if (pState->m_zip64) {
    // 如果未压缩大小大于等于 MZ_UINT32_MAX 或本地目录头偏移量大于等于 MZ_UINT32_MAX
    if (uncomp_size >= MZ_UINT32_MAX || local_dir_header_ofs >= MZ_UINT32_MAX) {
      // 设置额外数据和大小
      pExtra_data = extra_data;
      extra_size = mz_zip_writer_create_zip64_extra_data(
          extra_data, (uncomp_size >= MZ_UINT32_MAX) ? &uncomp_size : NULL,
          (uncomp_size >= MZ_UINT32_MAX) ? &comp_size : NULL,
          (local_dir_header_ofs >= MZ_UINT32_MAX) ? &local_dir_header_ofs
                                                  : NULL);
    }
    // 如果创建本地目录头部失败，则返回内部错误
    if (!mz_zip_writer_create_local_dir_header(
            pZip, local_dir_header, (mz_uint16)archive_name_size,
            (mz_uint16)(extra_size + user_extra_data_len), 0, 0, 0, method,
            bit_flags, dos_time, dos_date))
      return mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);

    // 将本地目录头部写入文件
    if (pZip->m_pWrite(pZip->m_pIO_opaque, local_dir_header_ofs,
                       local_dir_header,
                       sizeof(local_dir_header)) != sizeof(local_dir_header))
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    // 更新当前归档文件偏移量
    cur_archive_file_ofs += sizeof(local_dir_header);

    // 将归档文件名写入文件
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs, pArchive_name,
                       archive_name_size) != archive_name_size) {
      pZip->m_pFree(pZip->m_pAlloc_opaque, pComp);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    }
    // 更新当前归档文件偏移量
    cur_archive_file_ofs += archive_name_size;

    // 如果存在额外数据
    if (pExtra_data != NULL) {
      // 将额外数据写入文件
      if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs, extra_data,
                         extra_size) != extra_size)
        return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

      // 更新当前归档文件偏移量
      cur_archive_file_ofs += extra_size;
    }
  } else {
    // 如果压缩大小或当前归档文件偏移量超过最大值，则返回归档文件过大错误
    if ((comp_size > MZ_UINT32_MAX) || (cur_archive_file_ofs > MZ_UINT32_MAX))
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);
    // 如果创建本地目录头部失败，则返回内部错误
    if (!mz_zip_writer_create_local_dir_header(
            pZip, local_dir_header, (mz_uint16)archive_name_size,
            (mz_uint16)user_extra_data_len, 0, 0, 0, method, bit_flags,
            dos_time, dos_date))
      return mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);

    // 将本地目录头部写入文件
    if (pZip->m_pWrite(pZip->m_pIO_opaque, local_dir_header_ofs,
                       local_dir_header,
                       sizeof(local_dir_header)) != sizeof(local_dir_header))
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    // 更新当前归档文件偏移量
    cur_archive_file_ofs += sizeof(local_dir_header);
    // 如果写入文件名失败，则释放内存并返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs, pArchive_name,
                       archive_name_size) != archive_name_size) {
      pZip->m_pFree(pZip->m_pAlloc_opaque, pComp);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    }
    // 更新当前文件偏移量
    cur_archive_file_ofs += archive_name_size;
  }

  // 如果用户额外数据长度大于0
  if (user_extra_data_len > 0) {
    // 如果写入用户额外数据失败，则返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs,
                       user_extra_data,
                       user_extra_data_len) != user_extra_data_len)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    // 更新当前文件偏移量
    cur_archive_file_ofs += user_extra_data_len;
  }

  // 如果存储数据未压缩
  if (store_data_uncompressed) {
    // 如果写入数据失败，则释放内存并返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs, pBuf,
                       buf_size) != buf_size) {
      pZip->m_pFree(pZip->m_pAlloc_opaque, pComp);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    }

    // 更新当前文件偏移量和压缩大小
    cur_archive_file_ofs += buf_size;
    comp_size = buf_size;
  } else if (buf_size) {
    // 初始化压缩状态
    mz_zip_writer_add_state state;

    state.m_pZip = pZip;
    state.m_cur_archive_file_ofs = cur_archive_file_ofs;
    state.m_comp_size = 0;

    // 如果初始化压缩失败或者压缩数据失败，则释放内存并返回压缩失败错误
    if ((tdefl_init(pComp, mz_zip_writer_add_put_buf_callback, &state,
                    tdefl_create_comp_flags_from_zip_params(
                        level, -15, MZ_DEFAULT_STRATEGY)) !=
         TDEFL_STATUS_OKAY) ||
        (tdefl_compress_buffer(pComp, pBuf, buf_size, TDEFL_FINISH) !=
         TDEFL_STATUS_DONE)) {
      pZip->m_pFree(pZip->m_pAlloc_opaque, pComp);
      return mz_zip_set_error(pZip, MZ_ZIP_COMPRESSION_FAILED);
    }

    // 更新压缩大小和当前文件偏移量
    comp_size = state.m_comp_size;
    cur_archive_file_ofs = state.m_cur_archive_file_ofs;
  }

  // 释放内存并将指针置为空
  pZip->m_pFree(pZip->m_pAlloc_opaque, pComp);
  pComp = NULL;

  // 如果未压缩大小大于0
  if (uncomp_size) {
    // 创建本地目录尾部
    mz_uint8 local_dir_footer[MZ_ZIP_DATA_DESCRIPTER_SIZE64];
    mz_uint32 local_dir_footer_size = MZ_ZIP_DATA_DESCRIPTER_SIZE32;

    // 断言位标志包含本地文件头部定位器位标志
    MZ_ASSERT(bit_flags & MZ_ZIP_LDH_BIT_FLAG_HAS_LOCATOR);
    # 将 ZIP 数据描述符 ID 写入本地目录尾部
    MZ_WRITE_LE32(local_dir_footer + 0, MZ_ZIP_DATA_DESCRIPTOR_ID);
    # 将未压缩数据的 CRC32 写入本地目录尾部
    MZ_WRITE_LE32(local_dir_footer + 4, uncomp_crc32);
    # 如果额外数据为空
    if (pExtra_data == NULL) {
      # 如果压缩大小超过 32 位整数的最大值，返回错误
      if (comp_size > MZ_UINT32_MAX)
        return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);
      # 将压缩大小和未压缩大小写入本地目录尾部
      MZ_WRITE_LE32(local_dir_footer + 8, comp_size);
      MZ_WRITE_LE32(local_dir_footer + 12, uncomp_size);
    } else {
      # 将压缩大小和未压缩大小写入本地目录尾部（64 位）
      MZ_WRITE_LE64(local_dir_footer + 8, comp_size);
      MZ_WRITE_LE64(local_dir_footer + 16, uncomp_size);
      # 更新本地目录尾部大小为 64 位
      local_dir_footer_size = MZ_ZIP_DATA_DESCRIPTER_SIZE64;
    }

    # 将本地目录尾部写入 ZIP 文件
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs,
                       local_dir_footer,
                       local_dir_footer_size) != local_dir_footer_size)
      return MZ_FALSE;

    # 更新当前归档文件偏移量
    cur_archive_file_ofs += local_dir_footer_size;
  }

  # 如果额外数据不为空
  if (pExtra_data != NULL) {
    # 创建 ZIP64 额外数据
    extra_size = mz_zip_writer_create_zip64_extra_data(
        extra_data, (uncomp_size >= MZ_UINT32_MAX) ? &uncomp_size : NULL,
        (uncomp_size >= MZ_UINT32_MAX) ? &comp_size : NULL,
        (local_dir_header_ofs >= MZ_UINT32_MAX) ? &local_dir_header_ofs : NULL);
  }

  # 将文件信息添加到中央目录
  if (!mz_zip_writer_add_to_central_dir(
          pZip, pArchive_name, (mz_uint16)archive_name_size, pExtra_data,
          (mz_uint16)extra_size, pComment, comment_size, uncomp_size, comp_size,
          uncomp_crc32, method, bit_flags, dos_time, dos_date,
          local_dir_header_ofs, ext_attributes, user_extra_data_central,
          user_extra_data_central_len))
    return MZ_FALSE;

  # 更新总文件数和归档大小
  pZip->m_total_files++;
  pZip->m_archive_size = cur_archive_file_ofs;

  # 返回成功
  return MZ_TRUE;
  # 添加一个读取缓冲区回调函数到 ZIP 归档中
  mz_bool mz_zip_writer_add_read_buf_callback(
      mz_zip_archive *pZip, const char *pArchive_name,
      mz_file_read_func read_callback, void *callback_opaque, mz_uint64 max_size,
      const MZ_TIME_T *pFile_time, const void *pComment, mz_uint16 comment_size,
      mz_uint level_and_flags, mz_uint32 ext_attributes,
      const char *user_extra_data, mz_uint user_extra_data_len,
      const char *user_extra_data_central, mz_uint user_extra_data_central_len) {
    # 根据 level_and_flags 中的标志位设置 gen_flags
    mz_uint16 gen_flags = (level_and_flags & MZ_ZIP_FLAG_WRITE_HEADER_SET_SIZE)
                              ? 0
                              : MZ_ZIP_LDH_BIT_FLAG_HAS_LOCATOR;
    # 初始化一些变量
    mz_uint uncomp_crc32 = MZ_CRC32_INIT, level, num_alignment_padding_bytes;
    mz_uint16 method = 0, dos_time = 0, dos_date = 0;
    mz_uint64 local_dir_header_ofs, cur_archive_file_ofs = 0, uncomp_size = 0,
                                    comp_size = 0;
    size_t archive_name_size;
    mz_uint8 local_dir_header[MZ_ZIP_LOCAL_DIR_HEADER_SIZE];
    mz_uint8 *pExtra_data = NULL;
    mz_uint32 extra_size = 0;
    mz_uint8 extra_data[MZ_ZIP64_MAX_CENTRAL_EXTRA_FIELD_SIZE];
    mz_zip_internal_state *pState;
    mz_uint64 file_ofs = 0, cur_archive_header_file_ofs;

    # 如果不是 ASCII 文件名，则设置 gen_flags 中的 UTF8 标志位
    if (!(level_and_flags & MZ_ZIP_FLAG_ASCII_FILENAME))
      gen_flags |= MZ_ZIP_GENERAL_PURPOSE_BIT_FLAG_UTF8;

    # 如果 level_and_flags 小于 0，则将其设置为默认级别
    if ((int)level_and_flags < 0)
      level_and_flags = MZ_DEFAULT_LEVEL;
    level = level_and_flags & 0xF;

    /* Sanity checks */
    # 进行一些合法性检查
    if ((!pZip) || (!pZip->m_pState) ||
        (pZip->m_zip_mode != MZ_ZIP_MODE_WRITING) || (!pArchive_name) ||
        ((comment_size) && (!pComment)) || (level > MZ_UBER_COMPRESSION))
      return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

    # 获取 ZIP 内部状态
    pState = pZip->m_pState;
    cur_archive_file_ofs = pZip->m_archive_size;

    # 如果不是 ZIP64 并且 max_size 大于 MZ_UINT32_MAX，则返回错误
    if ((!pState->m_zip64) && (max_size > MZ_UINT32_MAX)) {
      /* Source file is too large for non-zip64 */
      /*return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE); */
```  
    // 设置 ZIP64 标志为真
    pState->m_zip64 = MZ_TRUE;
  }

  /* 我们可以支持这个，但是为什么呢？ */
  // 如果 level_and_flags 包含 MZ_ZIP_FLAG_COMPRESSED_DATA 标志，则返回无效参数错误
  if (level_and_flags & MZ_ZIP_FLAG_COMPRESSED_DATA)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 验证存档名称是否有效
  if (!mz_zip_writer_validate_archive_name(pArchive_name))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_FILENAME);

  // 如果 ZIP64 标志为真
  if (pState->m_zip64) {
    // 如果存档中的文件数达到 MZ_UINT32_MAX，则返回文件过多错误
    if (pZip->m_total_files == MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);
  } else {
    // 如果 ZIP64 标志为假
    if (pZip->m_total_files == MZ_UINT16_MAX) {
      // 设置 ZIP64 标志为真
      pState->m_zip64 = MZ_TRUE;
      /*return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES); */
    }
  }

  // 计算存档名称的长度
  archive_name_size = strlen(pArchive_name);
  // 如果存档名称长度超过 MZ_UINT16_MAX，则返回无效文件名错误
  if (archive_name_size > MZ_UINT16_MAX)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_FILENAME);

  // 计算需要用于文件对齐的填充字节数
  num_alignment_padding_bytes =
      mz_zip_writer_compute_padding_needed_for_file_alignment(pZip);

  /* miniz 目前不支持中央目录大于 MZ_UINT32_MAX 字节 */
  // 如果中央目录大小超过 MZ_UINT32_MAX 字节，则返回不支持的中央目录大小错误
  if (((mz_uint64)pState->m_central_dir.m_size +
       MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + archive_name_size +
       MZ_ZIP64_MAX_CENTRAL_EXTRA_FIELD_SIZE + comment_size) >= MZ_UINT32_MAX)
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_CDIR_SIZE);

  // 如果 ZIP64 标志为假
  if (!pState->m_zip64) {
    // 如果存档显然会变得太大，则提前退出
    if ((pZip->m_archive_size + num_alignment_padding_bytes +
         MZ_ZIP_LOCAL_DIR_HEADER_SIZE + archive_name_size +
         MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + archive_name_size + comment_size +
         user_extra_data_len + pState->m_central_dir.m_size +
         MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE + 1024 +
         MZ_ZIP_DATA_DESCRIPTER_SIZE32 + user_extra_data_central_len) >
        0xFFFFFFFF) {
      // 设置 ZIP64 标志为真
      pState->m_zip64 = MZ_TRUE;
      /*return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE); */
    }
  }
#ifndef MINIZ_NO_TIME
  // 如果定义了 MINIZ_NO_TIME 宏，则跳过时间处理
  if (pFile_time) {
    // 将文件时间转换为 DOS 时间格式
    mz_zip_time_t_to_dos_time(*pFile_time, &dos_time, &dos_date);
  }
#endif

  // 如果最大大小小于等于3，则压缩级别设为0
  if (max_size <= 3)
    level = 0;

  // 写入指定数量的零字节到 ZIP 文件中
  if (!mz_zip_writer_write_zeros(pZip, cur_archive_file_ofs,
                                 num_alignment_padding_bytes)) {
    // 写入失败时返回文件写入失败错误
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
  }

  // 更新当前文件偏移量
  cur_archive_file_ofs += num_alignment_padding_bytes;
  // 记录本地目录头的偏移量
  local_dir_header_ofs = cur_archive_file_ofs;

  // 如果设置了文件偏移对齐值，则进行断言检查
  if (pZip->m_file_offset_alignment) {
    MZ_ASSERT((cur_archive_file_ofs & (pZip->m_file_offset_alignment - 1)) ==
              0);
  }

  // 如果最大大小不为0且压缩级别不为0，则使用 DEFLATED 压缩方法
  if (max_size && level) {
    method = MZ_DEFLATED;
  }

  // 清空本地目录头对象
  MZ_CLEAR_OBJ(local_dir_header);
  // 如果启用 ZIP64 功能
  if (pState->m_zip64) {
    // 如果最大大小大于等于 MZ_UINT32_MAX 或本地目录头偏移量大于等于 MZ_UINT32_MAX
    if (max_size >= MZ_UINT32_MAX || local_dir_header_ofs >= MZ_UINT32_MAX) {
      pExtra_data = extra_data;
      // 根据条件创建 ZIP64 额外数据
      if (level_and_flags & MZ_ZIP_FLAG_WRITE_HEADER_SET_SIZE)
        extra_size = mz_zip_writer_create_zip64_extra_data(
            extra_data, (max_size >= MZ_UINT32_MAX) ? &uncomp_size : NULL,
            (max_size >= MZ_UINT32_MAX) ? &comp_size : NULL,
            (local_dir_header_ofs >= MZ_UINT32_MAX) ? &local_dir_header_ofs
                                                    : NULL);
      else
        extra_size = mz_zip_writer_create_zip64_extra_data(
            extra_data, NULL, NULL,
            (local_dir_header_ofs >= MZ_UINT32_MAX) ? &local_dir_header_ofs
                                                    : NULL);
    }

    // 创建本地目录头
    if (!mz_zip_writer_create_local_dir_header(
            pZip, local_dir_header, (mz_uint16)archive_name_size,
            (mz_uint16)(extra_size + user_extra_data_len), 0, 0, 0, method,
            gen_flags, dos_time, dos_date))
      return mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);
    // 如果写入本地目录头部失败，则返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs,
                       local_dir_header,
                       sizeof(local_dir_header)) != sizeof(local_dir_header))
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    // 更新当前归档文件偏移量
    cur_archive_file_ofs += sizeof(local_dir_header);

    // 如果写入归档名称失败，则返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs, pArchive_name,
                       archive_name_size) != archive_name_size) {
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    }

    // 更新当前归档文件偏移量
    cur_archive_file_ofs += archive_name_size;

    // 如果写入额外数据失败，则返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs, extra_data,
                       extra_size) != extra_size)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    // 更新当前归档文件偏移量
    cur_archive_file_ofs += extra_size;
  } else {
    // 如果压缩大小或当前归档文件偏移量超过最大值，则返回归档太大错误
    if ((comp_size > MZ_UINT32_MAX) || (cur_archive_file_ofs > MZ_UINT32_MAX))
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);
    // 如果创建本地目录头部失败，则返回内部错误
    if (!mz_zip_writer_create_local_dir_header(
            pZip, local_dir_header, (mz_uint16)archive_name_size,
            (mz_uint16)user_extra_data_len, 0, 0, 0, method, gen_flags,
            dos_time, dos_date))
      return mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);

    // 如果写入本地目录头部失败，则返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs,
                       local_dir_header,
                       sizeof(local_dir_header)) != sizeof(local_dir_header))
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    // 更新当前归档文件偏移量
    cur_archive_file_ofs += sizeof(local_dir_header);

    // 如果写入归档名称失败，则返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs, pArchive_name,
                       archive_name_size) != archive_name_size) {
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    }

    // 更新当前归档文件偏移量
    cur_archive_file_ofs += archive_name_size;
  }

  // 如果用户额外数据长度大于0
    // 如果写入用户额外数据的长度不等于用户额外数据的长度，返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs,
                       user_extra_data,
                       user_extra_data_len) != user_extra_data_len)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    // 更新当前归档文件偏移量
    cur_archive_file_ofs += user_extra_data_len;
  }

  // 如果有最大大小限制
  if (max_size) {
    // 分配读取缓冲区
    void *pRead_buf =
        pZip->m_pAlloc(pZip->m_pAlloc_opaque, 1, MZ_ZIP_MAX_IO_BUF_SIZE);
    // 如果分配失败，返回内存分配失败错误
    if (!pRead_buf) {
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }

    // 如果压缩级别为0
    if (!level) {
      // 循环读取数据
      while (1) {
        // 调用读取回调函数读取数据
        size_t n = read_callback(callback_opaque, file_ofs, pRead_buf,
                                 MZ_ZIP_MAX_IO_BUF_SIZE);
        // 如果读取到的数据长度为0，跳出循环
        if (n == 0)
          break;

        // 如果读取到的数据长度大于缓冲区大小或者文件偏移量加上数据长度超过最大大小，释放缓冲区并返回文件读取失败错误
        if ((n > MZ_ZIP_MAX_IO_BUF_SIZE) || (file_ofs + n > max_size)) {
          pZip->m_pFree(pZip->m_pAlloc_opaque, pRead_buf);
          return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
        }
        // 写入数据到归档文件，如果写入长度不等于数据长度，释放缓冲区并返回文件写入失败错误
        if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs, pRead_buf,
                           n) != n) {
          pZip->m_pFree(pZip->m_pAlloc_opaque, pRead_buf);
          return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
        }
        // 更新文件偏移量和未压缩数据的 CRC32 校验值
        file_ofs += n;
        uncomp_crc32 =
            (mz_uint32)mz_crc32(uncomp_crc32, (const mz_uint8 *)pRead_buf, n);
        cur_archive_file_ofs += n;
      }
      // 更新未压缩大小和压缩大小
      uncomp_size = file_ofs;
      comp_size = uncomp_size;
    }

    // 释放读取缓冲区
    pZip->m_pFree(pZip->m_pAlloc_opaque, pRead_buf);
  }

  // 如果标志中未设置写入头部大小
  if (!(level_and_flags & MZ_ZIP_FLAG_WRITE_HEADER_SET_SIZE)) {
    // 创建本地目录尾部
    mz_uint8 local_dir_footer[MZ_ZIP_DATA_DESCRIPTER_SIZE64];
    mz_uint32 local_dir_footer_size = MZ_ZIP_DATA_DESCRIPTER_SIZE32;

    // 写入本地目录尾部数据描述符标识和未压缩数据的 CRC32 校验值
    MZ_WRITE_LE32(local_dir_footer + 0, MZ_ZIP_DATA_DESCRIPTOR_ID);
    MZ_WRITE_LE32(local_dir_footer + 4, uncomp_crc32);
    // 如果额外数据为空
    if (pExtra_data == NULL) {
      // 如果压缩大小超过最大值，返回错误
      if (comp_size > MZ_UINT32_MAX)
        return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);

      // 将压缩大小和未压缩大小写入本地目录尾部
      MZ_WRITE_LE32(local_dir_footer + 8, comp_size);
      MZ_WRITE_LE32(local_dir_footer + 12, uncomp_size);
    } else {
      // 如果额外数据不为空，将压缩大小和未压缩大小写入本地目录尾部
      MZ_WRITE_LE64(local_dir_footer + 8, comp_size);
      MZ_WRITE_LE64(local_dir_footer + 16, uncomp_size);
      // 设置本地目录尾部大小为64位
      local_dir_footer_size = MZ_ZIP_DATA_DESCRIPTER_SIZE64;
    }

    // 将本地目录尾部写入文件
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_file_ofs,
                       local_dir_footer,
                       local_dir_footer_size) != local_dir_footer_size)
      return MZ_FALSE;

    // 更新当前文件偏移量
    cur_archive_file_ofs += local_dir_footer_size;
  }

  // 如果标志包含设置文件大小的标志
  if (level_and_flags & MZ_ZIP_FLAG_WRITE_HEADER_SET_SIZE) {
    // 如果额外数据不为空
    if (pExtra_data != NULL) {
      // 创建ZIP64额外数据
      extra_size = mz_zip_writer_create_zip64_extra_data(
          extra_data, (max_size >= MZ_UINT32_MAX) ? &uncomp_size : NULL,
          (max_size >= MZ_UINT32_MAX) ? &comp_size : NULL,
          (local_dir_header_ofs >= MZ_UINT32_MAX) ? &local_dir_header_ofs
                                                  : NULL);
    }

    // 创建本地目录头部
    if (!mz_zip_writer_create_local_dir_header(
            pZip, local_dir_header, (mz_uint16)archive_name_size,
            (mz_uint16)(extra_size + user_extra_data_len),
            (max_size >= MZ_UINT32_MAX) ? MZ_UINT32_MAX : uncomp_size,
            (max_size >= MZ_UINT32_MAX) ? MZ_UINT32_MAX : comp_size,
            uncomp_crc32, method, gen_flags, dos_time, dos_date))
      return mz_zip_set_error(pZip, MZ_ZIP_INTERNAL_ERROR);

    // 更新当前文件头部偏移量
    cur_archive_header_file_ofs = local_dir_header_ofs;

    // 将本地目录头部写入文件
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_header_file_ofs,
                       local_dir_header,
                       sizeof(local_dir_header)) != sizeof(local_dir_header))
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    // 如果额外数据不为空
    if (pExtra_data != NULL) {
      // 更新当前存档头文件偏移量
      cur_archive_header_file_ofs += sizeof(local_dir_header);

      // 写入存档名称到存档文件
      if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_header_file_ofs,
                         pArchive_name,
                         archive_name_size) != archive_name_size) {
        // 如果写入失败，返回文件写入失败错误
        return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
      }

      // 更新当前存档头文件偏移量
      cur_archive_header_file_ofs += archive_name_size;

      // 写入额外数据到存档文件
      if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_archive_header_file_ofs,
                         extra_data, extra_size) != extra_size)
        // 如果写入失败，返回文件写入失败错误
        return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

      // 更新当前存档头文件偏移量
      cur_archive_header_file_ofs += extra_size;
    }
  }

  // 如果额外数据不为空
  if (pExtra_data != NULL) {
    // 创建 ZIP64 额外数据
    extra_size = mz_zip_writer_create_zip64_extra_data(
        extra_data, (uncomp_size >= MZ_UINT32_MAX) ? &uncomp_size : NULL,
        (uncomp_size >= MZ_UINT32_MAX) ? &comp_size : NULL,
        (local_dir_header_ofs >= MZ_UINT32_MAX) ? &local_dir_header_ofs : NULL);
  }

  // 将文件添加到中央目录
  if (!mz_zip_writer_add_to_central_dir(
          pZip, pArchive_name, (mz_uint16)archive_name_size, pExtra_data,
          (mz_uint16)extra_size, pComment, comment_size, uncomp_size, comp_size,
          uncomp_crc32, method, gen_flags, dos_time, dos_date,
          local_dir_header_ofs, ext_attributes, user_extra_data_central,
          user_extra_data_central_len))
    // 如果添加失败，返回假值
    return MZ_FALSE;

  // 更新总文件数
  pZip->m_total_files++;
  // 更新存档大小
  pZip->m_archive_size = cur_archive_file_ofs;

  // 返回真值
  return MZ_TRUE;
// 如果未定义 MINIZ_NO_STDIO，则定义一个函数，用于从标准输入输出读取数据
#ifndef MINIZ_NO_STDIO

// 从标准输入输出读取数据的回调函数
static size_t mz_file_read_func_stdio(void *pOpaque, mz_uint64 file_ofs,
                                      void *pBuf, size_t n) {
  // 强制转换为文件指针类型
  MZ_FILE *pSrc_file = (MZ_FILE *)pOpaque;
  // 获取当前文件指针位置
  mz_int64 cur_ofs = MZ_FTELL64(pSrc_file);

  // 如果文件偏移小于0，或者当前文件指针位置不等于文件偏移并且无法移动文件指针到指定位置，则返回0
  if (((mz_int64)file_ofs < 0) ||
      (((cur_ofs != (mz_int64)file_ofs)) &&
       (MZ_FSEEK64(pSrc_file, (mz_int64)file_ofs, SEEK_SET))))
    return 0;

  // 从文件中读取数据到缓冲区
  return MZ_FREAD(pBuf, 1, n, pSrc_file);
}

// 添加一个来自 C 文件的数据到 ZIP 归档中
mz_bool mz_zip_writer_add_cfile(
    mz_zip_archive *pZip, const char *pArchive_name, MZ_FILE *pSrc_file,
    mz_uint64 max_size, const MZ_TIME_T *pFile_time, const void *pComment,
    mz_uint16 comment_size, mz_uint level_and_flags, mz_uint32 ext_attributes,
    const char *user_extra_data, mz_uint user_extra_data_len,
    const char *user_extra_data_central, mz_uint user_extra_data_central_len) {
  // 调用添加读取缓冲区回调函数，将 C 文件的数据添加到 ZIP 归档中
  return mz_zip_writer_add_read_buf_callback(
      pZip, pArchive_name, mz_file_read_func_stdio, pSrc_file, max_size,
      pFile_time, pComment, comment_size, level_and_flags, ext_attributes,
      user_extra_data, user_extra_data_len, user_extra_data_central,
      user_extra_data_central_len);
}

// 添加一个文件到 ZIP 归档中
mz_bool mz_zip_writer_add_file(mz_zip_archive *pZip, const char *pArchive_name,
                               const char *pSrc_filename, const void *pComment,
                               mz_uint16 comment_size, mz_uint level_and_flags,
                               mz_uint32 ext_attributes) {
  MZ_FILE *pSrc_file = NULL;
  mz_uint64 uncomp_size = 0;
  MZ_TIME_T file_modified_time;
  MZ_TIME_T *pFile_time = NULL;
  mz_bool status;

  // 初始化文件修改时间
  memset(&file_modified_time, 0, sizeof(file_modified_time));

  // 如果未定义 MINIZ_NO_TIME 和 MINIZ_NO_STDIO，则获取文件修改时间
#if !defined(MINIZ_NO_TIME) && !defined(MINIZ_NO_STDIO)
  pFile_time = &file_modified_time;
  // 如果无法获取文件修改时间，则返回文件状态获取失败错误
  if (!mz_zip_get_file_modified_time(pSrc_filename, &file_modified_time))
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_STAT_FAILED);
#endif

  // 打开源文件以供读取
  pSrc_file = MZ_FOPEN(pSrc_filename, "rb");
  // 如果无法打开源文件，则返回错误
  if (!pSrc_file)
    # 返回 ZIP 文件打开失败的错误信息
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_OPEN_FAILED);

  # 将源文件指针移动到文件末尾，获取文件大小
  MZ_FSEEK64(pSrc_file, 0, SEEK_END);
  uncomp_size = MZ_FTELL64(pSrc_file);
  # 将源文件指针移动回文件开头
  MZ_FSEEK64(pSrc_file, 0, SEEK_SET);

  # 将源文件添加到 ZIP 文件中
  status = mz_zip_writer_add_cfile(
      pZip, pArchive_name, pSrc_file, uncomp_size, pFile_time, pComment,
      comment_size, level_and_flags, ext_attributes, NULL, 0, NULL, 0);

  # 关闭源文件
  MZ_FCLOSE(pSrc_file);

  # 返回添加文件到 ZIP 文件的状态
  return status;
// 结束条件判断，检查是否定义了 MINIZ_NO_STDIO，如果没有则执行下面的代码块
#endif /* #ifndef MINIZ_NO_STDIO */

// 更新 ZIP64 扩展块
static mz_bool mz_zip_writer_update_zip64_extension_block(
    mz_zip_array *pNew_ext, mz_zip_archive *pZip, const mz_uint8 *pExt,
    uint32_t ext_len, mz_uint64 *pComp_size, mz_uint64 *pUncomp_size,
    mz_uint64 *pLocal_header_ofs, mz_uint32 *pDisk_start) {
  // 分配足够的空间来存储新的 zip64 数据
  /* + 64 should be enough for any new zip64 data */
  if (!mz_zip_array_reserve(pZip, pNew_ext, ext_len + 64, MZ_FALSE))
    return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

  // 重置新扩展块的大小为0
  mz_zip_array_resize(pZip, pNew_ext, 0, MZ_FALSE);

  // 如果有需要更新的数据，则构建新的扩展块
  if ((pUncomp_size) || (pComp_size) || (pLocal_header_ofs) || (pDisk_start)) {
    mz_uint8 new_ext_block[64];
    mz_uint8 *pDst = new_ext_block;
    // 写入 ZIP64 扩展信息字段头部标识
    mz_write_le16(pDst, MZ_ZIP64_EXTENDED_INFORMATION_FIELD_HEADER_ID);
    mz_write_le16(pDst + sizeof(mz_uint16), 0);
    pDst += sizeof(mz_uint16) * 2;

    // 如果有未压缩大小，则写入
    if (pUncomp_size) {
      mz_write_le64(pDst, *pUncomp_size);
      pDst += sizeof(mz_uint64);
    }

    // 如果有压缩大小，则写入
    if (pComp_size) {
      mz_write_le64(pDst, *pComp_size);
      pDst += sizeof(mz_uint64);
    }

    // 如果有本地头偏移，则写入
    if (pLocal_header_ofs) {
      mz_write_le64(pDst, *pLocal_header_ofs);
      pDst += sizeof(mz_uint64);
    }

    // 如果有磁盘起始位置，则写入
    if (pDisk_start) {
      mz_write_le32(pDst, *pDisk_start);
      pDst += sizeof(mz_uint32);
    }

    // 计算并写入扩展块的大小
    mz_write_le16(new_ext_block + sizeof(mz_uint16),
                  (mz_uint16)((pDst - new_ext_block) - sizeof(mz_uint16) * 2));

    // 将新的扩展块添加到 ZIP 对象中
    if (!mz_zip_array_push_back(pZip, pNew_ext, new_ext_block,
                                pDst - new_ext_block))
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
  }

  // 如果有额外数据需要处理，则继续处理
  if ((pExt) && (ext_len)) {
    mz_uint32 extra_size_remaining = ext_len;
    const mz_uint8 *pExtra_data = pExt;
    do {
      // 定义字段 ID、字段数据大小、字段总大小
      mz_uint32 field_id, field_data_size, field_total_size;

      // 如果剩余额外数据大小小于两个 mz_uint16 类型的大小，则返回错误
      if (extra_size_remaining < (sizeof(mz_uint16) * 2))
        return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

      // 读取字段 ID 和字段数据大小
      field_id = MZ_READ_LE16(pExtra_data);
      field_data_size = MZ_READ_LE16(pExtra_data + sizeof(mz_uint16));
      // 计算字段总大小
      field_total_size = field_data_size + sizeof(mz_uint16) * 2;

      // 如果字段总大小大于剩余额外数据大小，则返回错误
      if (field_total_size > extra_size_remaining)
        return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

      // 如果字段 ID 不是 ZIP64 扩展信息字段头 ID
      if (field_id != MZ_ZIP64_EXTENDED_INFORMATION_FIELD_HEADER_ID) {
        // 将字段数据添加到新的扩展信息数组中
        if (!mz_zip_array_push_back(pZip, pNew_ext, pExtra_data,
                                    field_total_size))
          return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
      }

      // 移动额外数据指针和更新剩余额外数据大小
      pExtra_data += field_total_size;
      extra_size_remaining -= field_total_size;
    } while (extra_size_remaining);
  }

  // 返回操作成功
  return MZ_TRUE;
/* 
   添加一个文件从源 ZIP 读取器到目标 ZIP 写入器
   这个函数现在因为 zip64 变得相当复杂，是否需要拆分？
*/
mz_bool mz_zip_writer_add_from_zip_reader(mz_zip_archive *pZip,
                                          mz_zip_archive *pSource_zip,
                                          mz_uint src_file_index) {
  mz_uint n, bit_flags, num_alignment_padding_bytes,
      src_central_dir_following_data_size;
  mz_uint64 src_archive_bytes_remaining, local_dir_header_ofs;
  mz_uint64 cur_src_file_ofs, cur_dst_file_ofs;
  mz_uint32
      local_header_u32[(MZ_ZIP_LOCAL_DIR_HEADER_SIZE + sizeof(mz_uint32) - 1) /
                       sizeof(mz_uint32)];
  mz_uint8 *pLocal_header = (mz_uint8 *)local_header_u32;
  mz_uint8 new_central_header[MZ_ZIP_CENTRAL_DIR_HEADER_SIZE];
  size_t orig_central_dir_size;
  mz_zip_internal_state *pState;
  void *pBuf;
  const mz_uint8 *pSrc_central_header;
  mz_zip_archive_file_stat src_file_stat;
  mz_uint32 src_filename_len, src_comment_len, src_ext_len;
  mz_uint32 local_header_filename_size, local_header_extra_len;
  mz_uint64 local_header_comp_size, local_header_uncomp_size;
  mz_bool found_zip64_ext_data_in_ldir = MZ_FALSE;

  /* 检查参数有效性 */
  if ((!pZip) || (!pZip->m_pState) ||
      (pZip->m_zip_mode != MZ_ZIP_MODE_WRITING) || (!pSource_zip->m_pRead))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  pState = pZip->m_pState;

  /* 不支持从 zip64 归档复制文件到非 zip64 归档，尽管在某些情况下这是可能的 */
  if ((pSource_zip->m_pState->m_zip64) && (!pZip->m_pState->m_zip64))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  /* 获取源中央目录头的指针并解析它 */
  if (NULL ==
      (pSrc_central_header = mz_zip_get_cdh(pSource_zip, src_file_index)))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  /* 检查中央目录头的签名是否正确 */
  if (MZ_READ_LE32(pSrc_central_header + MZ_ZIP_CDH_SIG_OFS) !=
      MZ_ZIP_CENTRAL_DIR_HEADER_SIG)
    // 设置 ZIP 错误为无效头部或损坏
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 读取源中央目录头部中的文件名长度
  src_filename_len =
      MZ_READ_LE16(pSrc_central_header + MZ_ZIP_CDH_FILENAME_LEN_OFS);
  // 读取源中央目录头部中的注释长度
  src_comment_len =
      MZ_READ_LE16(pSrc_central_header + MZ_ZIP_CDH_COMMENT_LEN_OFS);
  // 读取源中央目录头部中的额外数据长度
  src_ext_len = MZ_READ_LE16(pSrc_central_header + MZ_ZIP_CDH_EXTRA_LEN_OFS);
  // 计算源中央目录头部后的数据大小
  src_central_dir_following_data_size =
      src_filename_len + src_ext_len + src_comment_len;

  /* TODO: We don't support central dir's >= MZ_UINT32_MAX bytes right now (+32
   * fudge factor in case we need to add more extra data) */
  // 检查中央目录的大小是否超过限制
  if ((pState->m_central_dir.m_size + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE +
       src_central_dir_following_data_size + 32) >= MZ_UINT32_MAX)
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_CDIR_SIZE);

  // 计算文件对齐所需的填充字节数
  num_alignment_padding_bytes =
      mz_zip_writer_compute_padding_needed_for_file_alignment(pZip);

  // 检查是否超过 ZIP 文件的文件数限制
  if (!pState->m_zip64) {
    if (pZip->m_total_files == MZ_UINT16_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);
  } else {
    /* TODO: Our zip64 support still has some 32-bit limits that may not be
     * worth fixing. */
    if (pZip->m_total_files == MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);
  }

  // 获取源文件的统计信息
  if (!mz_zip_file_stat_internal(pSource_zip, src_file_index,
                                 pSrc_central_header, &src_file_stat, NULL))
    return MZ_FALSE;

  // 设置当前源文件偏移和目标文件偏移
  cur_src_file_ofs = src_file_stat.m_local_header_ofs;
  cur_dst_file_ofs = pZip->m_archive_size;

  // 读取源存档的本地目录头部
  if (pSource_zip->m_pRead(pSource_zip->m_pIO_opaque, cur_src_file_ofs,
                           pLocal_header, MZ_ZIP_LOCAL_DIR_HEADER_SIZE) !=
      MZ_ZIP_LOCAL_DIR_HEADER_SIZE)
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);

  // 检查本地目录头部的签名是否正确
  if (MZ_READ_LE32(pLocal_header) != MZ_ZIP_LOCAL_DIR_HEADER_SIG)
    // 返回 ZIP 错误信息，表示头部无效或损坏
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);

  // 更新当前源文件偏移量
  cur_src_file_ofs += MZ_ZIP_LOCAL_DIR_HEADER_SIZE;

  /* 计算需要复制的总大小（文件名+额外数据+压缩数据） */
  local_header_filename_size =
      MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_FILENAME_LEN_OFS);
  local_header_extra_len =
      MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_EXTRA_LEN_OFS);
  local_header_comp_size =
      MZ_READ_LE32(pLocal_header + MZ_ZIP_LDH_COMPRESSED_SIZE_OFS);
  local_header_uncomp_size =
      MZ_READ_LE32(pLocal_header + MZ_ZIP_LDH_DECOMPRESSED_SIZE_OFS);
  src_archive_bytes_remaining = local_header_filename_size +
                                local_header_extra_len +
                                src_file_stat.m_comp_size;

  /* 尝试查找 zip64 扩展信息字段 */
  if ((local_header_extra_len) &&
      ((local_header_comp_size == MZ_UINT32_MAX) ||
       (local_header_uncomp_size == MZ_UINT32_MAX))) {
    // 初始化文件数据数组
    mz_zip_array file_data_array;
    const mz_uint8 *pExtra_data;
    mz_uint32 extra_size_remaining = local_header_extra_len;

    mz_zip_array_init(&file_data_array, 1);
    // 调整文件数据数组大小
    if (!mz_zip_array_resize(pZip, &file_data_array, local_header_extra_len,
                             MZ_FALSE)) {
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }

    // 读取额外数据
    if (pSource_zip->m_pRead(pSource_zip->m_pIO_opaque,
                             src_file_stat.m_local_header_ofs +
                                 MZ_ZIP_LOCAL_DIR_HEADER_SIZE +
                                 local_header_filename_size,
                             file_data_array.m_p, local_header_extra_len) !=
        local_header_extra_len) {
      mz_zip_array_clear(pZip, &file_data_array);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
    }

    pExtra_data = (const mz_uint8 *)file_data_array.m_p;
    // 使用 do-while 循环处理额外数据字段
    do {
      // 定义字段 ID、字段数据大小、字段总大小
      mz_uint32 field_id, field_data_size, field_total_size;

      // 如果剩余额外数据大小小于两个 mz_uint16 大小
      if (extra_size_remaining < (sizeof(mz_uint16) * 2)) {
        // 清空文件数据数组
        mz_zip_array_clear(pZip, &file_data_array);
        // 设置 ZIP 错误为无效头部或损坏
        return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);
      }

      // 读取字段 ID 和字段数据大小
      field_id = MZ_READ_LE16(pExtra_data);
      field_data_size = MZ_READ_LE16(pExtra_data + sizeof(mz_uint16));
      field_total_size = field_data_size + sizeof(mz_uint16) * 2;

      // 如果字段总大小大于剩余额外数据大小
      if (field_total_size > extra_size_remaining) {
        // 清空文件数据数组
        mz_zip_array_clear(pZip, &file_data_array);
        // 设置 ZIP 错误为无效头部或损坏
        return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);
      }

      // 如果字段 ID 为 ZIP64 扩展信息字段头部 ID
      if (field_id == MZ_ZIP64_EXTENDED_INFORMATION_FIELD_HEADER_ID) {
        // 获取字段数据指针
        const mz_uint8 *pSrc_field_data = pExtra_data + sizeof(mz_uint32);

        // 如果字段数据大小小于两个 mz_uint64 大小
        if (field_data_size < sizeof(mz_uint64) * 2) {
          // 清空文件数据数组
          mz_zip_array_clear(pZip, &file_data_array);
          // 设置 ZIP 错误为无效头部或损坏
          return mz_zip_set_error(pZip, MZ_ZIP_INVALID_HEADER_OR_CORRUPTED);
        }

        // 读取本地头部未压缩大小和压缩大小
        local_header_uncomp_size = MZ_READ_LE64(pSrc_field_data);
        local_header_comp_size = MZ_READ_LE64(
            pSrc_field_data +
            sizeof(mz_uint64)); /* may be 0 if there's a descriptor */

        // 标记在本地文件头部中找到 ZIP64 扩展数据
        found_zip64_ext_data_in_ldir = MZ_TRUE;
        // 跳出循环
        break;
      }

      // 移动额外数据指针和更新剩余额外数据大小
      pExtra_data += field_total_size;
      extra_size_remaining -= field_total_size;
    } while (extra_size_remaining);

    // 清空文件数据数组
    mz_zip_array_clear(pZip, &file_data_array);
  }

  // 如果不是 ZIP64 格式
  if (!pState->m_zip64) {
    /* 尝试检测新存档是否最终可能太大并提前退出（+(sizeof(mz_uint32) * 4) 用于可能存在的可选描述符，+64 是一个调整因子）。 */
    /* 我们还在存档最终化时检查，因此这不需要完美。 */
    // 计算近似新存档大小，考虑当前目标文件偏移量、对齐填充字节数、本地目录头大小、源存档剩余字节数等
    mz_uint64 approx_new_archive_size =
        cur_dst_file_ofs + num_alignment_padding_bytes +
        MZ_ZIP_LOCAL_DIR_HEADER_SIZE + src_archive_bytes_remaining +
        (sizeof(mz_uint32) * 4) + pState->m_central_dir.m_size +
        MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + src_central_dir_following_data_size +
        MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE + 64;

    // 如果近似新存档大小超过最大值，则返回存档过大错误
    if (approx_new_archive_size >= MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);
  }

  /* Write dest archive padding */
  // 写入目标存档填充字节
  if (!mz_zip_writer_write_zeros(pZip, cur_dst_file_ofs,
                                 num_alignment_padding_bytes))
    return MZ_FALSE;

  cur_dst_file_ofs += num_alignment_padding_bytes;

  local_dir_header_ofs = cur_dst_file_ofs;
  // 如果文件偏移量对齐要求存在，则进行断言检查
  if (pZip->m_file_offset_alignment) {
    MZ_ASSERT((local_dir_header_ofs & (pZip->m_file_offset_alignment - 1)) ==
              0);
  }

  /* The original zip's local header+ext block doesn't change, even with zip64,
   * so we can just copy it over to the dest zip */
  // 原始 ZIP 的本地头+扩展块不会改变，即使使用 zip64，因此可以直接复制到目标 ZIP
  if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_dst_file_ofs, pLocal_header,
                     MZ_ZIP_LOCAL_DIR_HEADER_SIZE) !=
      MZ_ZIP_LOCAL_DIR_HEADER_SIZE)
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

  cur_dst_file_ofs += MZ_ZIP_LOCAL_DIR_HEADER_SIZE;

  /* Copy over the source archive bytes to the dest archive, also ensure we have
   * enough buf space to handle optional data descriptor */
  // 将源存档字节复制到目标存档，同时确保有足够的缓冲空间来处理可选的数据描述符
  if (NULL == (pBuf = pZip->m_pAlloc(
                   pZip->m_pAlloc_opaque, 1,
                   (size_t)MZ_MAX(32U, MZ_MIN((mz_uint64)MZ_ZIP_MAX_IO_BUF_SIZE,
                                              src_archive_bytes_remaining))))
    return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

  while (src_archive_bytes_remaining) {
    n = (mz_uint)MZ_MIN((mz_uint64)MZ_ZIP_MAX_IO_BUF_SIZE,
                        src_archive_bytes_remaining);
    // 如果从源 ZIP 文件读取的数据长度不等于预期长度 n，则释放缓冲区并返回文件读取失败错误
    if (pSource_zip->m_pRead(pSource_zip->m_pIO_opaque, cur_src_file_ofs, pBuf,
                             n) != n) {
      pZip->m_pFree(pZip->m_pAlloc_opaque, pBuf);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
    }
    // 更新当前源文件偏移量
    cur_src_file_ofs += n;

    // 如果向目标 ZIP 文件写入的数据长度不等于预期长度 n，则释放缓冲区并返回文件写入失败错误
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_dst_file_ofs, pBuf, n) != n) {
      pZip->m_pFree(pZip->m_pAlloc_opaque, pBuf);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    }
    // 更新当前目标文件偏移量
    cur_dst_file_ofs += n;

    // 更新源 ZIP 文件剩余字节数
    src_archive_bytes_remaining -= n;
  }

  /* 处理可选的数据描述符 */
  // 读取本地文件头中的位标志
  bit_flags = MZ_READ_LE16(pLocal_header + MZ_ZIP_LDH_BIT_FLAG_OFS);
  // 如果位标志的第 4 位为 1，表示存在数据描述符
  if (bit_flags & 8) {
    /* 复制数据描述符 */
    // 如果源 ZIP 文件为 zip64 或者在局部目录中找到 zip64 扩展数据
    if ((pSource_zip->m_pState->m_zip64) || (found_zip64_ext_data_in_ldir)) {
      /* 源为 zip64，目标必须为 zip64 */

      /* 名称            uint32_t's */
      /* ID                1 (zip64 中可选?) */
      /* CRC            1 */
      /* 压缩大小    2 */
      /* 未压缩大小 2 */
      // 从源 ZIP 文件读取数据描述符的内容
      if (pSource_zip->m_pRead(pSource_zip->m_pIO_opaque, cur_src_file_ofs,
                               pBuf, (sizeof(mz_uint32) * 6)) !=
          (sizeof(mz_uint32) * 6)) {
        pZip->m_pFree(pZip->m_pAlloc_opaque, pBuf);
        return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
      }

      // 根据数据描述符的 ID 判断需要读取的字节数
      n = sizeof(mz_uint32) *
          ((MZ_READ_LE32(pBuf) == MZ_ZIP_DATA_DESCRIPTOR_ID) ? 6 : 5);
    } else {
      /* 如果源不是zip64 */
      mz_bool has_id;

      /* 读取源文件的数据描述符 */
      if (pSource_zip->m_pRead(pSource_zip->m_pIO_opaque, cur_src_file_ofs,
                               pBuf, sizeof(mz_uint32) * 4) !=
          sizeof(mz_uint32) * 4) {
        pZip->m_pFree(pZip->m_pAlloc_opaque, pBuf);
        return mz_zip_set_error(pZip, MZ_ZIP_FILE_READ_FAILED);
      }

      /* 检查是否存在数据描述符 */
      has_id = (MZ_READ_LE32(pBuf) == MZ_ZIP_DATA_DESCRIPTOR_ID);

      if (pZip->m_pState->m_zip64) {
        /* 如果目标是zip64，则升级数据描述符 */
        const mz_uint32 *pSrc_descriptor =
            (const mz_uint32 *)((const mz_uint8 *)pBuf +
                                (has_id ? sizeof(mz_uint32) : 0));
        const mz_uint32 src_crc32 = pSrc_descriptor[0];
        const mz_uint64 src_comp_size = pSrc_descriptor[1];
        const mz_uint64 src_uncomp_size = pSrc_descriptor[2];

        /* 更新数据描述符 */
        mz_write_le32((mz_uint8 *)pBuf, MZ_ZIP_DATA_DESCRIPTOR_ID);
        mz_write_le32((mz_uint8 *)pBuf + sizeof(mz_uint32) * 1, src_crc32);
        mz_write_le64((mz_uint8 *)pBuf + sizeof(mz_uint32) * 2, src_comp_size);
        mz_write_le64((mz_uint8 *)pBuf + sizeof(mz_uint32) * 4,
                      src_uncomp_size);

        n = sizeof(mz_uint32) * 6;
      } else {
        /* 如果目标不是zip64，直接复制 */
        n = sizeof(mz_uint32) * (has_id ? 4 : 3);
      }
    }

    /* 写入数据到目标文件 */
    if (pZip->m_pWrite(pZip->m_pIO_opaque, cur_dst_file_ofs, pBuf, n) != n) {
      pZip->m_pFree(pZip->m_pAlloc_opaque, pBuf);
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    }

    cur_src_file_ofs += n;
    cur_dst_file_ofs += n;
  }
  pZip->m_pFree(pZip->m_pAlloc_opaque, pBuf);

  /* 最后，添加新的中央目录头部 */
  orig_central_dir_size = pState->m_central_dir.m_size;

  /* 复制中央目录头部 */
  memcpy(new_central_header, pSrc_central_header,
         MZ_ZIP_CENTRAL_DIR_HEADER_SIZE);

  if (pState->m_zip64) {
    /* 这是痛苦的部分：我们需要编写一个新的中央目录头部 + 扩展块，其中包含更新的 zip64 字段，并确保不包含旧字段（如果有的话）。 */
    const mz_uint8 *pSrc_ext =
        pSrc_central_header + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE + src_filename_len;
    mz_zip_array new_ext_block;

    // 初始化新的扩展块
    mz_zip_array_init(&new_ext_block, sizeof(mz_uint8));

    // 将最大值写入新的中央目录头部的压缩大小、解压缩大小和本地头部偏移量字段
    MZ_WRITE_LE32(new_central_header + MZ_ZIP_CDH_COMPRESSED_SIZE_OFS,
                  MZ_UINT32_MAX);
    MZ_WRITE_LE32(new_central_header + MZ_ZIP_CDH_DECOMPRESSED_SIZE_OFS,
                  MZ_UINT32_MAX);
    MZ_WRITE_LE32(new_central_header + MZ_ZIP_CDH_LOCAL_HEADER_OFS,
                  MZ_UINT32_MAX);

    // 更新 zip64 扩展块，并检查是否成功
    if (!mz_zip_writer_update_zip64_extension_block(
            &new_ext_block, pZip, pSrc_ext, src_ext_len,
            &src_file_stat.m_comp_size, &src_file_stat.m_uncomp_size,
            &local_dir_header_ofs, NULL)) {
      mz_zip_array_clear(pZip, &new_ext_block);
      return MZ_FALSE;
    }

    // 将新扩展块的大小写入新的中央目录头部的额外长度字段
    MZ_WRITE_LE16(new_central_header + MZ_ZIP_CDH_EXTRA_LEN_OFS,
                  new_ext_block.m_size);

    // 将新的中央目录头部添加到中央目录数组中
    if (!mz_zip_array_push_back(pZip, &pState->m_central_dir,
                                new_central_header,
                                MZ_ZIP_CENTRAL_DIR_HEADER_SIZE)) {
      mz_zip_array_clear(pZip, &new_ext_block);
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }

    // 将源中央目录头部中的文件名部分添加到中央目录数组中
    if (!mz_zip_array_push_back(pZip, &pState->m_central_dir,
                                pSrc_central_header +
                                    MZ_ZIP_CENTRAL_DIR_HEADER_SIZE,
                                src_filename_len)) {
      mz_zip_array_clear(pZip, &new_ext_block);
      mz_zip_array_resize(pZip, &pState->m_central_dir, orig_central_dir_size,
                          MZ_FALSE);
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }
    // 如果无法将新的扩展块添加到中央目录中，则清除新扩展块，恢复原始中央目录大小，并返回分配失败错误
    if (!mz_zip_array_push_back(pZip, &pState->m_central_dir, new_ext_block.m_p,
                                new_ext_block.m_size)) {
      mz_zip_array_clear(pZip, &new_ext_block);
      mz_zip_array_resize(pZip, &pState->m_central_dir, orig_central_dir_size,
                          MZ_FALSE);
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }

    // 如果无法将源文件的注释数据添加到中央目录中，则清除新扩展块，恢复原始中央目录大小，并返回分配失败错误
    if (!mz_zip_array_push_back(pZip, &pState->m_central_dir,
                                pSrc_central_header +
                                    MZ_ZIP_CENTRAL_DIR_HEADER_SIZE +
                                    src_filename_len + src_ext_len,
                                src_comment_len)) {
      mz_zip_array_clear(pZip, &new_ext_block);
      mz_zip_array_resize(pZip, &pState->m_central_dir, orig_central_dir_size,
                          MZ_FALSE);
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }

    // 清除新扩展块
    mz_zip_array_clear(pZip, &new_ext_block);
  } else {
    /* sanity checks */
    // 检查当前目标文件偏移是否超过最大值
    if (cur_dst_file_ofs > MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);

    // 检查本地目录头偏移是否超过最大值
    if (local_dir_header_ofs >= MZ_UINT32_MAX)
      return mz_zip_set_error(pZip, MZ_ZIP_ARCHIVE_TOO_LARGE);

    // 将本地目录头偏移写入新中央目录头
    MZ_WRITE_LE32(new_central_header + MZ_ZIP_CDH_LOCAL_HEADER_OFS,
                  local_dir_header_ofs);

    // 如果无法将新中央目录头添加到中央目录中，则返回分配失败错误
    if (!mz_zip_array_push_back(pZip, &pState->m_central_dir,
                                new_central_header,
                                MZ_ZIP_CENTRAL_DIR_HEADER_SIZE))
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);

    // 如果无法将源中央目录头后续数据添加到中央目录中，则恢复原始中央目录大小，并返回分配失败错误
    if (!mz_zip_array_push_back(pZip, &pState->m_central_dir,
                                pSrc_central_header +
                                    MZ_ZIP_CENTRAL_DIR_HEADER_SIZE,
                                src_central_dir_following_data_size)) {
      mz_zip_array_resize(pZip, &pState->m_central_dir, orig_central_dir_size,
                          MZ_FALSE);
      return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
    }
  }

  /* 如果在初始检查期间出现问题，则不应触发此处 */
  if (pState->m_central_dir.m_size >= MZ_UINT32_MAX) {
    /* TODO: 支持大于等于32位的中央目录大小 */
    mz_zip_array_resize(pZip, &pState->m_central_dir, orig_central_dir_size,
                        MZ_FALSE);
    return mz_zip_set_error(pZip, MZ_ZIP_UNSUPPORTED_CDIR_SIZE);
  }

  n = (mz_uint32)orig_central_dir_size;
  if (!mz_zip_array_push_back(pZip, &pState->m_central_dir_offsets, &n, 1)) {
    mz_zip_array_resize(pZip, &pState->m_central_dir, orig_central_dir_size,
                        MZ_FALSE);
    return mz_zip_set_error(pZip, MZ_ZIP_ALLOC_FAILED);
  }

  pZip->m_total_files++;
  pZip->m_archive_size = cur_dst_file_ofs;

  return MZ_TRUE;
}

// 完成 ZIP 归档的最终化，包括写入中央目录和 ZIP64 结尾
mz_bool mz_zip_writer_finalize_archive(mz_zip_archive *pZip) {
  mz_zip_internal_state *pState;
  mz_uint64 central_dir_ofs, central_dir_size;
  mz_uint8 hdr[256];

  // 检查参数和 ZIP 模式是否正确
  if ((!pZip) || (!pZip->m_pState) || (pZip->m_zip_mode != MZ_ZIP_MODE_WRITING))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  pState = pZip->m_pState;

  // 检查是否需要使用 ZIP64 格式
  if (pState->m_zip64) {
    if ((pZip->m_total_files > MZ_UINT32_MAX) ||
        (pState->m_central_dir.m_size >= MZ_UINT32_MAX))
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);
  } else {
    if ((pZip->m_total_files > MZ_UINT16_MAX) ||
        ((pZip->m_archive_size + pState->m_central_dir.m_size +
          MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE) > MZ_UINT32_MAX))
      return mz_zip_set_error(pZip, MZ_ZIP_TOO_MANY_FILES);
  }

  central_dir_ofs = 0;
  central_dir_size = 0;
  if (pZip->m_total_files) {
    /* 写入中央目录 */
    central_dir_ofs = pZip->m_archive_size;
    central_dir_size = pState->m_central_dir.m_size;
    pZip->m_central_directory_file_ofs = central_dir_ofs;
    if (pZip->m_pWrite(pZip->m_pIO_opaque, central_dir_ofs,
                       pState->m_central_dir.m_p,
                       (size_t)central_dir_size) != central_dir_size)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    pZip->m_archive_size += central_dir_size;
  }

  if (pState->m_zip64) {
    /* 写入 ZIP64 结尾中央目录头部 */
    mz_uint64 rel_ofs_to_zip64_ecdr = pZip->m_archive_size;

    MZ_CLEAR_OBJ(hdr);
    MZ_WRITE_LE32(hdr + MZ_ZIP64_ECDH_SIG_OFS,
                  MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIG);
    MZ_WRITE_LE64(hdr + MZ_ZIP64_ECDH_SIZE_OF_RECORD_OFS,
                  MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE - sizeof(mz_uint32) -
                      sizeof(mz_uint64));
    MZ_WRITE_LE16(hdr + MZ_ZIP64_ECDH_VERSION_MADE_BY_OFS,
                  0x031E); /* TODO: always Unix */
    MZ_WRITE_LE16(hdr + MZ_ZIP64_ECDH_VERSION_NEEDED_OFS, 0x002D);
    // 将总文件数写入 ZIP64 结尾中央目录头部
    MZ_WRITE_LE64(hdr + MZ_ZIP64_ECDH_CDIR_NUM_ENTRIES_ON_DISK_OFS,
                  pZip->m_total_files);
    // 将总文件数写入 ZIP64 结尾中央目录头部
    MZ_WRITE_LE64(hdr + MZ_ZIP64_ECDH_CDIR_TOTAL_ENTRIES_OFS,
                  pZip->m_total_files);
    // 将中央目录大小写入 ZIP64 结尾中央目录头部
    MZ_WRITE_LE64(hdr + MZ_ZIP64_ECDH_CDIR_SIZE_OFS, central_dir_size);
    // 将中央目录偏移量写入 ZIP64 结尾中央目录头部
    MZ_WRITE_LE64(hdr + MZ_ZIP64_ECDH_CDIR_OFS_OFS, central_dir_ofs);
    // 将 ZIP64 结尾中央目录头部写入文件
    if (pZip->m_pWrite(pZip->m_pIO_opaque, pZip->m_archive_size, hdr,
                       MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE) !=
        MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);

    // 更新存档大小
    pZip->m_archive_size += MZ_ZIP64_END_OF_CENTRAL_DIR_HEADER_SIZE;

    /* Write zip64 end of central directory locator */
    // 清空 hdr 对象
    MZ_CLEAR_OBJ(hdr);
    // 写入 ZIP64 结尾中央目录定位器标识
    MZ_WRITE_LE32(hdr + MZ_ZIP64_ECDL_SIG_OFS,
                  MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIG);
    // 写入相对偏移量到 ZIP64 结尾中央目录记录的偏移量
    MZ_WRITE_LE64(hdr + MZ_ZIP64_ECDL_REL_OFS_TO_ZIP64_ECDR_OFS,
                  rel_ofs_to_zip64_ecdr);
    // 写入总磁盘数
    MZ_WRITE_LE32(hdr + MZ_ZIP64_ECDL_TOTAL_NUMBER_OF_DISKS_OFS, 1);
    // 将 ZIP64 结尾中央目录定位器写入文件
    if (pZip->m_pWrite(pZip->m_pIO_opaque, pZip->m_archive_size, hdr,
                       MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIZE) !=
        MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIZE)
      return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
    // 增加 ZIP 文件的总大小，包括 ZIP64 中央目录定位器的大小
    pZip->m_archive_size += MZ_ZIP64_END_OF_CENTRAL_DIR_LOCATOR_SIZE;
  }

  /* 写入中央目录结束记录 */
  // 清空 hdr 对象
  MZ_CLEAR_OBJ(hdr);
  // 写入中央目录结束记录的标识符
  MZ_WRITE_LE32(hdr + MZ_ZIP_ECDH_SIG_OFS,
                MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIG);
  // 写入中央目录中的文件数目
  MZ_WRITE_LE16(hdr + MZ_ZIP_ECDH_CDIR_NUM_ENTRIES_ON_DISK_OFS,
                MZ_MIN(MZ_UINT16_MAX, pZip->m_total_files));
  // 写入中央目录中的总文件数目
  MZ_WRITE_LE16(hdr + MZ_ZIP_ECDH_CDIR_TOTAL_ENTRIES_OFS,
                MZ_MIN(MZ_UINT16_MAX, pZip->m_total_files));
  // 写入中央目录的大小
  MZ_WRITE_LE32(hdr + MZ_ZIP_ECDH_CDIR_SIZE_OFS,
                MZ_MIN(MZ_UINT32_MAX, central_dir_size));
  // 写入中央目录的偏移量
  MZ_WRITE_LE32(hdr + MZ_ZIP_ECDH_CDIR_OFS_OFS,
                MZ_MIN(MZ_UINT32_MAX, central_dir_ofs));

  // 如果写入中央目录结束记录失败，则返回写入失败的错误
  if (pZip->m_pWrite(pZip->m_pIO_opaque, pZip->m_archive_size, hdr,
                     MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE) !=
      MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE)
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_WRITE_FAILED);
#ifndef MINIZ_NO_STDIO
  // 如果定义了 MINIZ_NO_STDIO，并且存在文件指针并且刷新文件失败，则返回文件关闭失败错误
  if ((pState->m_pFile) && (MZ_FFLUSH(pState->m_pFile) == EOF))
    return mz_zip_set_error(pZip, MZ_ZIP_FILE_CLOSE_FAILED);
#endif /* #ifndef MINIZ_NO_STDIO */

  // 增加 ZIP 存档大小
  pZip->m_archive_size += MZ_ZIP_END_OF_CENTRAL_DIR_HEADER_SIZE;

  // 设置 ZIP 存档模式为已经完成写入
  pZip->m_zip_mode = MZ_ZIP_MODE_WRITING_HAS_BEEN_FINALIZED;
  // 返回真值
  return MZ_TRUE;
}

// 完成堆存档的最终化
mz_bool mz_zip_writer_finalize_heap_archive(mz_zip_archive *pZip, void **ppBuf,
                                            size_t *pSize) {
  // 如果 ppBuf 或 pSize 为空，则返回无效参数错误
  if ((!ppBuf) || (!pSize))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 初始化 ppBuf 和 pSize
  *ppBuf = NULL;
  *pSize = 0;

  // 如果 pZip 为空或 pZip->m_pState 为空，则返回无效参数错误
  if ((!pZip) || (!pZip->m_pState))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 如果 pZip->m_pWrite 不等于 mz_zip_heap_write_func，则返回无效参数错误
  if (pZip->m_pWrite != mz_zip_heap_write_func)
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 如果无法完成存档最终化，则返回假值
  if (!mz_zip_writer_finalize_archive(pZip))
    return MZ_FALSE;

  // 设置 ppBuf 和 pSize 为存档状态的内存和内存大小
  *ppBuf = pZip->m_pState->m_pMem;
  *pSize = pZip->m_pState->m_mem_size;
  // 重置存档状态的内存和内存大小
  pZip->m_pState->m_pMem = NULL;
  pZip->m_pState->m_mem_size = pZip->m_pState->m_mem_capacity = 0;

  // 返回真值
  return MZ_TRUE;
}

// 结束 ZIP 写入
mz_bool mz_zip_writer_end(mz_zip_archive *pZip) {
  // 调用内部的 ZIP 写入结束函数
  return mz_zip_writer_end_internal(pZip, MZ_TRUE);
}

#ifndef MINIZ_NO_STDIO
// 将内存添加到原地存档文件中
mz_bool mz_zip_add_mem_to_archive_file_in_place(
    const char *pZip_filename, const char *pArchive_name, const void *pBuf,
    size_t buf_size, const void *pComment, mz_uint16 comment_size,
    mz_uint level_and_flags) {
  // 调用带有版本参数的内存添加到原地存档文件中函数
  return mz_zip_add_mem_to_archive_file_in_place_v2(
      pZip_filename, pArchive_name, pBuf, buf_size, pComment, comment_size,
      level_and_flags, NULL);
}

// 带有版本参数的将内存添加到原地存档文件中
mz_bool mz_zip_add_mem_to_archive_file_in_place_v2(
    const char *pZip_filename, const char *pArchive_name, const void *pBuf,
    size_t buf_size, const void *pComment, mz_uint16 comment_size,
  // 定义变量 level_and_flags 和 pErr，level_and_flags 为无符号整数，pErr 为指向错误信息的指针
  mz_uint level_and_flags, mz_zip_error *pErr;
  // 定义变量 status 和 created_new_archive，status 为布尔值，created_new_archive 为布尔值
  mz_bool status, created_new_archive = MZ_FALSE;
  // 定义变量 zip_archive 和 file_stat，zip_archive 为 ZIP 归档对象，file_stat 为文件状态结构体
  mz_zip_archive zip_archive;
  struct MZ_FILE_STAT_STRUCT file_stat;
  // 定义变量 actual_err，表示 ZIP 错误信息，默认为 MZ_ZIP_NO_ERROR
  mz_zip_error actual_err = MZ_ZIP_NO_ERROR;

  // 将 zip_archive 结构体清零
  mz_zip_zero_struct(&zip_archive);
  // 如果 level_and_flags 小于 0，则将其设置为默认级别
  if ((int)level_and_flags < 0)
    level_and_flags = MZ_DEFAULT_LEVEL;

  // 检查参数是否有效，包括 ZIP 文件名、归档名称、缓冲区、注释、压缩级别等
  if ((!pZip_filename) || (!pArchive_name) || ((buf_size) && (!pBuf)) ||
      ((comment_size) && (!pComment)) ||
      ((level_and_flags & 0xF) > MZ_UBER_COMPRESSION)) {
    // 如果参数无效，设置错误信息并返回 False
    if (pErr)
      *pErr = MZ_ZIP_INVALID_PARAMETER;
    return MZ_FALSE;
  }

  // 验证归档名称是否有效
  if (!mz_zip_writer_validate_archive_name(pArchive_name)) {
    // 如果归档名称无效，设置错误信息并返回 False
    if (pErr)
      *pErr = MZ_ZIP_INVALID_FILENAME;
    return MZ_FALSE;
  }

  // 如果文件很大，stat() 可能失败，需要编译时定义 _LARGEFILE64_SOURCE 为 1
  if (MZ_FILE_STAT(pZip_filename, &file_stat) != 0) {
    // 如果文件不存在，创建新的归档
    if (!mz_zip_writer_init_file_v2(&zip_archive, pZip_filename, 0,
                                    level_and_flags)) {
      // 如果初始化失败，设置错误信息并返回 False
      if (pErr)
        *pErr = zip_archive.m_last_error;
      return MZ_FALSE;
    }

    created_new_archive = MZ_TRUE;
  } else {
    // 如果文件存在，追加到现有归档
    if (!mz_zip_reader_init_file_v2(
            &zip_archive, pZip_filename,
            level_and_flags | MZ_ZIP_FLAG_DO_NOT_SORT_CENTRAL_DIRECTORY, 0,
            0)) {
      // 如果初始化失败，设置错误信息并返回 False
      if (pErr)
        *pErr = zip_archive.m_last_error;
      return MZ_FALSE;
    }

    // 从现有归档初始化 ZIP 写入器
    if (!mz_zip_writer_init_from_reader_v2(&zip_archive, pZip_filename,
                                           level_and_flags)) {
      // 如果初始化失败，设置错误信息并返回 False
      if (pErr)
        *pErr = zip_archive.m_last_error;

      // 结束 ZIP 读取器
      mz_zip_reader_end_internal(&zip_archive, MZ_FALSE);

      return MZ_FALSE;
  }
}

// 将内存中的数据添加到 ZIP 归档中
status =
    mz_zip_writer_add_mem_ex(&zip_archive, pArchive_name, pBuf, buf_size,
                             pComment, comment_size, level_and_flags, 0, 0);
// 获取实际错误码
actual_err = zip_archive.m_last_error;

/* 总是完成归档，即使由于某些原因添加失败，这样我们就有一个有效的中央目录。(这可能并不总是成功，但我们可以尝试。) */
if (!mz_zip_writer_finalize_archive(&zip_archive)) {
  if (!actual_err)
    actual_err = zip_archive.m_last_error;

  status = MZ_FALSE;
}

if (!mz_zip_writer_end_internal(&zip_archive, status)) {
  if (!actual_err)
    actual_err = zip_archive.m_last_error;

  status = MZ_FALSE;
}

if ((!status) && (created_new_archive)) {
  /* 这是一个新的归档，出现了问题，所以只需删除它。 */
  int ignoredStatus = MZ_DELETE_FILE(pZip_filename);
  (void)ignoredStatus;
}

// 如果存在错误指针，则将实际错误码赋给它
if (pErr)
  *pErr = actual_err;

// 返回操作状态
return status;
}

// 从 ZIP 文件中提取指定文件到堆中，并返回指向数据的指针
void *mz_zip_extract_archive_file_to_heap_v2(const char *pZip_filename,
                                             const char *pArchive_name,
                                             const char *pComment,
                                             size_t *pSize, mz_uint flags,
                                             mz_zip_error *pErr) {
  mz_uint32 file_index;
  mz_zip_archive zip_archive;
  void *p = NULL;

  // 如果 pSize 不为空，则将其值设为 0
  if (pSize)
    *pSize = 0;

  // 如果 pZip_filename 或 pArchive_name 为空，则返回 NULL
  if ((!pZip_filename) || (!pArchive_name)) {
    if (pErr)
      *pErr = MZ_ZIP_INVALID_PARAMETER;

    return NULL;
  }

  // 初始化 zip_archive 结构体
  mz_zip_zero_struct(&zip_archive);
  // 初始化 ZIP 读取器，打开 ZIP 文件
  if (!mz_zip_reader_init_file_v2(
          &zip_archive, pZip_filename,
          flags | MZ_ZIP_FLAG_DO_NOT_SORT_CENTRAL_DIRECTORY, 0, 0)) {
    if (pErr)
      *pErr = zip_archive.m_last_error;

    return NULL;
  }

  // 定位指定文件在 ZIP 文件中的索引
  if (mz_zip_reader_locate_file_v2(&zip_archive, pArchive_name, pComment, flags,
                                   &file_index)) {
    // 从 ZIP 文件中提取指定文件到堆中
    p = mz_zip_reader_extract_to_heap(&zip_archive, file_index, pSize, flags);
  }

  // 结束 ZIP 读取器的内部操作
  mz_zip_reader_end_internal(&zip_archive, p != NULL);

  if (pErr)
    *pErr = zip_archive.m_last_error;

  return p;
}

// 从 ZIP 文件中提取指定文件到堆中，并返回指向数据的指针
void *mz_zip_extract_archive_file_to_heap(const char *pZip_filename,
                                          const char *pArchive_name,
                                          size_t *pSize, mz_uint flags) {
  return mz_zip_extract_archive_file_to_heap_v2(pZip_filename, pArchive_name,
                                                NULL, pSize, flags, NULL);
}

#endif /* #ifndef MINIZ_NO_STDIO */

#endif /* #ifndef MINIZ_NO_ARCHIVE_WRITING_APIS */

/* ------------------- Misc utils */

// 获取 ZIP 归档的模式
mz_zip_mode mz_zip_get_mode(mz_zip_archive *pZip) {
  return pZip ? pZip->m_zip_mode : MZ_ZIP_MODE_INVALID;
}

// 获取 ZIP 归档的类型
mz_zip_type mz_zip_get_type(mz_zip_archive *pZip) {
  return pZip ? pZip->m_zip_type : MZ_ZIP_TYPE_INVALID;
}
# 设置 ZIP 归档对象的最后错误码，并返回之前的错误码
mz_zip_error mz_zip_set_last_error(mz_zip_archive *pZip, mz_zip_error err_num) {
  # 保存之前的错误码
  mz_zip_error prev_err;

  # 如果 ZIP 归档对象为空，则返回无效参数错误码
  if (!pZip)
    return MZ_ZIP_INVALID_PARAMETER;

  # 保存之前的错误码
  prev_err = pZip->m_last_error;

  # 设置新的错误码
  pZip->m_last_error = err_num;
  # 返回之前的错误码
  return prev_err;
}

# 获取 ZIP 归档对象的最后错误码
mz_zip_error mz_zip_peek_last_error(mz_zip_archive *pZip) {
  # 如果 ZIP 归档对象为空，则返回无效参数错误码
  if (!pZip)
    return MZ_ZIP_INVALID_PARAMETER;

  # 返回最后的错误码
  return pZip->m_last_error;
}

# 清除 ZIP 归档对象的最后错误码
mz_zip_error mz_zip_clear_last_error(mz_zip_archive *pZip) {
  # 调用设置错误码函数，将错误码设置为无错误
  return mz_zip_set_last_error(pZip, MZ_ZIP_NO_ERROR);
}

# 获取 ZIP 归档对象的最后错误码，并将错误码设置为无错误
mz_zip_error mz_zip_get_last_error(mz_zip_archive *pZip) {
  # 保存之前的错误码
  mz_zip_error prev_err;

  # 如果 ZIP 归档对象为空，则返回无效参数错误码
  if (!pZip)
    return MZ_ZIP_INVALID_PARAMETER;

  # 保存之前的错误码
  prev_err = pZip->m_last_error;

  # 将错误码设置为无错误
  pZip->m_last_error = MZ_ZIP_NO_ERROR;
  # 返回之前的错误码
  return prev_err;
}

# 根据错误码返回对应的错误字符串
const char *mz_zip_get_error_string(mz_zip_error mz_err) {
  switch (mz_err) {
    case MZ_ZIP_NO_ERROR:
      return "no error";
    case MZ_ZIP_UNDEFINED_ERROR:
      return "undefined error";
    case MZ_ZIP_TOO_MANY_FILES:
      return "too many files";
    case MZ_ZIP_FILE_TOO_LARGE:
      return "file too large";
    case MZ_ZIP_UNSUPPORTED_METHOD:
      return "unsupported method";
    case MZ_ZIP_UNSUPPORTED_ENCRYPTION:
      return "unsupported encryption";
    case MZ_ZIP_UNSUPPORTED_FEATURE:
      return "unsupported feature";
    case MZ_ZIP_FAILED_FINDING_CENTRAL_DIR:
      return "failed finding central directory";
    case MZ_ZIP_NOT_AN_ARCHIVE:
      return "not a ZIP archive";
    case MZ_ZIP_INVALID_HEADER_OR_CORRUPTED:
      return "invalid header or archive is corrupted";
    case MZ_ZIP_UNSUPPORTED_MULTIDISK:
      return "unsupported multidisk archive";
    case MZ_ZIP_DECOMPRESSION_FAILED:
      return "decompression failed or archive is corrupted";
    case MZ_ZIP_COMPRESSION_FAILED:
      return "compression failed";
    case MZ_ZIP_UNEXPECTED_DECOMPRESSED_SIZE:
      return "unexpected decompressed size";
    case MZ_ZIP_CRC_CHECK_FAILED:
      return "CRC-32 check failed";
    case MZ_ZIP_UNSUPPORTED_CDIR_SIZE:
      return "unsupported central directory size";
  }
}
    // 返回不支持的中央目录大小错误信息
    return "unsupported central directory size";
  case MZ_ZIP_ALLOC_FAILED:
    // 返回分配内存失败错误信息
    return "allocation failed";
  case MZ_ZIP_FILE_OPEN_FAILED:
    // 返回文件打开失败错误信息
    return "file open failed";
  case MZ_ZIP_FILE_CREATE_FAILED:
    // 返回文件创建失败错误信息
    return "file create failed";
  case MZ_ZIP_FILE_WRITE_FAILED:
    // 返回文件写入失败错误信息
    return "file write failed";
  case MZ_ZIP_FILE_READ_FAILED:
    // 返回文件读取失败错误信息
    return "file read failed";
  case MZ_ZIP_FILE_CLOSE_FAILED:
    // 返回文件关闭失败错误信息
    return "file close failed";
  case MZ_ZIP_FILE_SEEK_FAILED:
    // 返回文件查找失败错误信息
    return "file seek failed";
  case MZ_ZIP_FILE_STAT_FAILED:
    // 返回文件状态获取失败错误信息
    return "file stat failed";
  case MZ_ZIP_INVALID_PARAMETER:
    // 返回无效参数错误信息
    return "invalid parameter";
  case MZ_ZIP_INVALID_FILENAME:
    // 返回无效文件名错误信息
    return "invalid filename";
  case MZ_ZIP_BUF_TOO_SMALL:
    // 返回缓冲区太小错误信息
    return "buffer too small";
  case MZ_ZIP_INTERNAL_ERROR:
    // 返回内部错误信息
    return "internal error";
  case MZ_ZIP_FILE_NOT_FOUND:
    // 返回文件未找到错误信息
    return "file not found";
  case MZ_ZIP_ARCHIVE_TOO_LARGE:
    // 返回存档太大错误信息
    return "archive is too large";
  case MZ_ZIP_VALIDATION_FAILED:
    // 返回验证失败错误信息
    return "validation failed";
  case MZ_ZIP_WRITE_CALLBACK_FAILED:
    // 返回写入回调失败错误信息
    return "write callback failed";
  case MZ_ZIP_TOTAL_ERRORS:
    // 返回总错误数信息
    return "total errors";
  default:
    // 默认情况下不返回任何信息
    break;
  }

  // 返回未知错误信息
  return "unknown error";
/* Note: Just because the archive is not zip64 doesn't necessarily mean it
 * doesn't have Zip64 extended information extra field, argh. */
/* 注意：仅因为存档不是 zip64 并不意味着它没有 Zip64 扩展信息额外字段 */
mz_bool mz_zip_is_zip64(mz_zip_archive *pZip) {
  // 如果传入的指针为空或者指向的状态为空，则返回 MZ_FALSE
  if ((!pZip) || (!pZip->m_pState))
    return MZ_FALSE;

  // 返回存档是否为 zip64
  return pZip->m_pState->m_zip64;
}

/* 获取中央目录的大小 */
size_t mz_zip_get_central_dir_size(mz_zip_archive *pZip) {
  // 如果传入的指针为空或者指向的状态为空，则返回 0
  if ((!pZip) || (!pZip->m_pState))
    return 0;

  // 返回中央目录的大小
  return pZip->m_pState->m_central_dir.m_size;
}

/* 获取存档中文件的数量 */
mz_uint mz_zip_reader_get_num_files(mz_zip_archive *pZip) {
  // 如果传入的指针为空，则返回 0，否则返回存档中文件的数量
  return pZip ? pZip->m_total_files : 0;
}

/* 获取存档的大小 */
mz_uint64 mz_zip_get_archive_size(mz_zip_archive *pZip) {
  // 如果传入的指针为空，则返回 0，否则返回存档的大小
  if (!pZip)
    return 0;
  return pZip->m_archive_size;
}

/* 获取存档文件的起始偏移量 */
mz_uint64 mz_zip_get_archive_file_start_offset(mz_zip_archive *pZip) {
  // 如果传入的指针为空或者指向的状态为空，则返回 0
  if ((!pZip) || (!pZip->m_pState))
    return 0;
  // 返回存档文件的起始偏移量
  return pZip->m_pState->m_file_archive_start_ofs;
}

/* 获取存档文件 */
MZ_FILE *mz_zip_get_cfile(mz_zip_archive *pZip) {
  // 如果传入的指针为空或者指向的状态为空，则返回 0
  if ((!pZip) || (!pZip->m_pState))
    return 0;
  // 返回存档文件
  return pZip->m_pState->m_pFile;
}

/* 读取存档数据 */
size_t mz_zip_read_archive_data(mz_zip_archive *pZip, mz_uint64 file_ofs,
                                void *pBuf, size_t n) {
  // 如果传入的指针为空或者指向的状态为空，或者缓冲区为空，或者读取函数为空，则返回无效参数错误
  if ((!pZip) || (!pZip->m_pState) || (!pBuf) || (!pZip->m_pRead))
    return mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);

  // 调用读取函数读取存档数据
  return pZip->m_pRead(pZip->m_pIO_opaque, file_ofs, pBuf, n);
}

/* 获取存档文件名 */
mz_uint mz_zip_reader_get_filename(mz_zip_archive *pZip, mz_uint file_index,
                                   char *pFilename, mz_uint filename_buf_size) {
  mz_uint n;
  // 获取中央目录头部信息
  const mz_uint8 *p = mz_zip_get_cdh(pZip, file_index);
  // 如果中央目录头部信息为空
  if (!p) {
    // 如果文件名缓冲区大小不为 0，则将文件名缓冲区第一个字符设为空字符
    if (filename_buf_size)
      pFilename[0] = '\0';
    // 设置存档错误为无效参数错误
    mz_zip_set_error(pZip, MZ_ZIP_INVALID_PARAMETER);
    return 0;
  }
  // 读取文件名长度
  n = MZ_READ_LE16(p + MZ_ZIP_CDH_FILENAME_LEN_OFS);
  // 如果文件名缓冲区大小不为 0
  if (filename_buf_size) {
    // 将文件名长度限制在缓冲区大小减 1 以内
    n = MZ_MIN(n, filename_buf_size - 1);
    // 将文件名复制到文件名缓冲区中
    memcpy(pFilename, p + MZ_ZIP_CENTRAL_DIR_HEADER_SIZE, n);
    pFilename[n] = '\0';
  }
  // 返回文件名长度加 1
  return n + 1;
}
# 检查指定索引的文件在 ZIP 存档中的统计信息，并将结果存储在给定的文件统计结构中
mz_bool mz_zip_reader_file_stat(mz_zip_archive *pZip, mz_uint file_index,
                                mz_zip_archive_file_stat *pStat) {
  return mz_zip_file_stat_internal(
      pZip, file_index, mz_zip_get_cdh(pZip, file_index), pStat, NULL);
}

# 结束 ZIP 存档的读取或写入操作
mz_bool mz_zip_end(mz_zip_archive *pZip) {
  # 如果 ZIP 存档指针为空，则返回假
  if (!pZip)
    return MZ_FALSE;

  # 如果 ZIP 存档处于读取模式，则结束读取操作
  if (pZip->m_zip_mode == MZ_ZIP_MODE_READING)
    return mz_zip_reader_end(pZip);
  # 如果 ZIP 存档处于写入或已经完成写入的模式，则结束写入操作
#ifndef MINIZ_NO_ARCHIVE_WRITING_APIS
  else if ((pZip->m_zip_mode == MZ_ZIP_MODE_WRITING) ||
           (pZip->m_zip_mode == MZ_ZIP_MODE_WRITING_HAS_BEEN_FINALIZED))
    return mz_zip_writer_end(pZip);
#endif

  # 返回假
  return MZ_FALSE;
}

# 结束 C++ 的 extern "C" 块
#ifdef __cplusplus
}
#endif

# 结束条件编译指令，确保未定义 MINIZ_NO_ARCHIVE_APIS 时代码块结束
#endif /*#ifndef MINIZ_NO_ARCHIVE_APIS*/
```