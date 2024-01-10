# `nmap\libz\gzguts.h`

```
/* gzguts.h -- zlib内部头文件定义，用于gz*操作
 * 版权所有 (C) 2004-2019 Mark Adler
 * 发行和使用条件，请参见zlib.h中的版权声明
 */

#ifdef _LARGEFILE64_SOURCE
#  ifndef _LARGEFILE_SOURCE
#    define _LARGEFILE_SOURCE 1
#  endif
#  ifdef _FILE_OFFSET_BITS
#    undef _FILE_OFFSET_BITS
#  endif
#endif

#ifdef HAVE_HIDDEN
#  define ZLIB_INTERNAL __attribute__((visibility ("hidden")))
#else
#  define ZLIB_INTERNAL
#endif

#include <stdio.h>
#include "zlib.h"
#ifdef STDC
#  include <string.h>
#  include <stdlib.h>
#  include <limits.h>
#endif

#ifndef _POSIX_SOURCE
#  define _POSIX_SOURCE
#endif
#include <fcntl.h>

#ifdef _WIN32
#  include <stddef.h>
#endif

#if defined(__TURBOC__) || defined(_MSC_VER) || defined(_WIN32)
#  include <io.h>
#endif

#if defined(_WIN32)
#  define WIDECHAR
#endif

#ifdef WINAPI_FAMILY
#  define open _open
#  define read _read
#  define write _write
#  define close _close
#endif

#ifdef NO_DEFLATE       /* 为了与旧定义兼容 */
#  define NO_GZCOMPRESS
#endif

#if defined(STDC99) || (defined(__TURBOC__) && __TURBOC__ >= 0x550)
#  ifndef HAVE_VSNPRINTF
#    define HAVE_VSNPRINTF
#  endif
#endif

#if defined(__CYGWIN__)
#  ifndef HAVE_VSNPRINTF
#    define HAVE_VSNPRINTF
#  endif
#endif

#if defined(MSDOS) && defined(__BORLANDC__) && (BORLANDC > 0x410)
#  ifndef HAVE_VSNPRINTF
#    define HAVE_VSNPRINTF
#  endif
#endif

#ifndef HAVE_VSNPRINTF
#  ifdef MSDOS
/* 在一些MS-DOS编译器上可能存在vsnprintf（DJGPP？），
   但目前我们假设它不存在。 */
#    define NO_vsnprintf
#  endif
#  ifdef __TURBOC__
#    define NO_vsnprintf
#  endif
#  ifdef WIN32
/* 在Win32中，vsnprintf可用作“非ANSI”_vsnprintf。 */
#    if !defined(vsnprintf) && !defined(NO_vsnprintf)
#      if !defined(_MSC_VER) || ( defined(_MSC_VER) && _MSC_VER < 1500 )
#         define vsnprintf _vsnprintf
#      endif
#    endif
#  endif
#  ifdef __SASC
#    define NO_vsnprintf
#  endif
# 如果定义了 VMS，则定义 NO_vsnprintf
# 如果定义了 __OS400__，则定义 NO_vsnprintf
# 如果定义了 __MVS__，则定义 NO_vsnprintf
#endif

/* _snprintf 不能保证结果的空终止，但在 gzlib.c 中只用于保证结果适合提供的空间 */
#if defined(_MSC_VER) && _MSC_VER < 1900
#  define snprintf _snprintf
#endif

#ifndef local
#  define local static
#endif
/* 由于 "static" 在 C 语言中有两个完全不同的含义，我们为了可读性定义了 "local" 作为 "static" 的非静态含义
   （如果你的调试器无法找到静态符号，请使用 -Dlocal 编译） */

/* gz* 函数总是使用库分配函数 */
#ifndef STDC
  extern voidp  malloc OF((uInt size));
  extern void   free   OF((voidpf ptr));
#endif

/* 获取 errno 和 strerror 的定义 */
#if defined UNDER_CE
#  include <windows.h>
#  define zstrerror() gz_strwinerror((DWORD)GetLastError())
#else
#  ifndef NO_STRERROR
#    include <errno.h>
#    define zstrerror() strerror(errno)
#  else
#    define zstrerror() "stdio error (consult errno)"
#  endif
#endif

/* 在没有启用 LFS 的情况下为 zlib 构建提供这些函数的原型 */
#if !defined(_LARGEFILE64_SOURCE) || _LFS64_LARGEFILE-0 == 0
    ZEXTERN gzFile ZEXPORT gzopen64 OF((const char *, const char *));
    ZEXTERN z_off64_t ZEXPORT gzseek64 OF((gzFile, z_off64_t, int));
    ZEXTERN z_off64_t ZEXPORT gztell64 OF((gzFile));
    ZEXTERN z_off64_t ZEXPORT gzoffset64 OF((gzFile));
#endif

/* 默认的 memLevel */
#if MAX_MEM_LEVEL >= 8
#  define DEF_MEM_LEVEL 8
#else
#  define DEF_MEM_LEVEL  MAX_MEM_LEVEL
#endif

/* 默认的 i/o 缓冲区大小 -- 在读取时将其加倍（这个值和两倍的值必须能够适应无符号类型） */
#define GZBUFSIZE 8192

/* gzip 模式，同时对传递的结构进行一些完整性检查 */
#define GZ_NONE 0
#define GZ_READ 7247
#define GZ_WRITE 31153
#define GZ_APPEND 1     /* mode set to GZ_WRITE after the file is opened */

/* values for gz_state how */
#define LOOK 0      /* look for a gzip header */
#define COPY 1      /* copy input directly */
#define GZIP 2      /* decompress a gzip stream */

/* internal gzip file state data structure */
typedef struct {
        /* exposed contents for gzgetc() macro */
    struct gzFile_s x;      /* "x" for exposed */
                            /* x.have: number of bytes available at x.next */
                            /* x.next: next output data to deliver or write */
                            /* x.pos: current position in uncompressed data */
        /* used for both reading and writing */
    int mode;               /* see gzip modes above */
    int fd;                 /* file descriptor */
    char *path;             /* path or fd for error messages */
    unsigned size;          /* buffer size, zero if not allocated yet */
    unsigned want;          /* requested buffer size, default is GZBUFSIZE */
    unsigned char *in;      /* input buffer (double-sized when writing) */
    unsigned char *out;     /* output buffer (double-sized when reading) */
    int direct;             /* 0 if processing gzip, 1 if transparent */
        /* just for reading */
    int how;                /* 0: get header, 1: copy, 2: decompress */
    z_off64_t start;        /* where the gzip data started, for rewinding */
    int eof;                /* true if end of input file reached */
    int past;               /* true if read requested past end */
        /* just for writing */
    int level;              /* compression level */
    int strategy;           /* compression strategy */
    int reset;              /* true if a reset is pending after a Z_FINISH */
        /* seek request */
    z_off64_t skip;         /* amount to skip (already rewound if backwards) */
    int seek;               /* true if seek request pending */
        /* error information */
    # 定义一个整型变量，用于存储错误代码
    int err;                /* error code */
    # 定义一个字符指针变量，用于存储错误消息
    char *msg;              /* error message */
    # 定义一个 zlib 压缩或解压流的结构体变量，用于存储压缩或解压的相关信息
    z_stream strm;          /* stream structure in-place (not a pointer) */
} gz_state;
typedef gz_state FAR *gz_statep;

/* 定义了一个名为gz_state的结构体，以及指向gz_state结构体的指针类型gz_statep */

/* 共享函数 */
void ZLIB_INTERNAL gz_error OF((gz_statep, int, const char *));
/* 定义了一个名为gz_error的函数，接受一个gz_statep类型的指针、一个整数和一个字符串作为参数 */

#if defined UNDER_CE
char ZLIB_INTERNAL *gz_strwinerror OF((DWORD error));
#endif
/* 如果定义了UNDER_CE宏，则定义了一个名为gz_strwinerror的函数，接受一个DWORD类型的参数，并返回一个char类型的指针 */

/* GT_OFF(x), where x is an unsigned value, is true if x > maximum z_off64_t
   value -- needed when comparing unsigned to z_off64_t, which is signed
   (possible z_off64_t types off_t, off64_t, and long are all signed) */
#ifdef INT_MAX
#  define GT_OFF(x) (sizeof(int) == sizeof(z_off64_t) && (x) > INT_MAX)
#else
unsigned ZLIB_INTERNAL gz_intmax OF((void));
#  define GT_OFF(x) (sizeof(int) == sizeof(z_off64_t) && (x) > gz_intmax())
#endif
/* 定义了一个名为GT_OFF的宏，根据不同条件返回一个布尔值，用于比较无符号值x和最大的z_off64_t值 */
```