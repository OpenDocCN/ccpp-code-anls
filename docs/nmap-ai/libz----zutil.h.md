# `nmap\libz\zutil.h`

```
/* zutil.h -- 内部接口和压缩库的配置
 * 版权所有 (C) 1995-2022 Jean-loup Gailly, Mark Adler
 * 发行和使用条件，请参见 zlib.h 中的版权声明
 */

/* 警告：此文件不应该被应用程序使用。它是压缩库的实现的一部分，并且可能会发生变化。应用程序应该只使用 zlib.h。
 */

/* @(#) $Id$ */

#ifndef ZUTIL_H
#define ZUTIL_H

#ifdef HAVE_HIDDEN
#  define ZLIB_INTERNAL __attribute__((visibility ("hidden")))
#else
#  define ZLIB_INTERNAL
#endif

#include "zlib.h"

#if defined(STDC) && !defined(Z_SOLO)
#  if !(defined(_WIN32_WCE) && defined(_MSC_VER))
#    include <stddef.h>
#  endif
#  include <string.h>
#  include <stdlib.h>
#endif

#ifndef local
#  define local static
#endif
/* "static" 在 C 语言中有两种完全不同的含义，我们为了可读性定义 "local" 用于非静态含义的 "static"
   (如果你的调试器无法找到静态符号，请使用 -Dlocal 编译)
 */

typedef unsigned char  uch;
typedef uch FAR uchf;
typedef unsigned short ush;
typedef ush FAR ushf;
typedef unsigned long  ulg;

#if !defined(Z_U8) && !defined(Z_SOLO) && defined(STDC)
#  include <limits.h>
#  if (ULONG_MAX == 0xffffffffffffffff)
#    define Z_U8 unsigned long
#  elif (ULLONG_MAX == 0xffffffffffffffff)
#    define Z_U8 unsigned long long
#  elif (UINT_MAX == 0xffffffffffffffff)
#    define Z_U8 unsigned
#  endif
#endif

extern z_const char * const z_errmsg[10]; /* 由 2-zlib_error 索引 */
/* (为了避免与 Visual C++ 的愚蠢警告，给出了大小) */

#define ERR_MSG(err) z_errmsg[Z_NEED_DICT-(err)]

#define ERR_RETURN(strm,err) \
  return (strm->msg = ERR_MSG(err), (err))
/* 仅在状态已知为有效时使用 */

        /* 常用常量 */

#ifndef DEF_WBITS
#  define DEF_WBITS MAX_WBITS
#endif
/* 解压缩的默认 windowBits。MAX_WBITS 仅用于压缩 */

#if MAX_MEM_LEVEL >= 8
// 定义默认的内存级别为8
#define DEF_MEM_LEVEL 8
#else
// 如果不是上述情况，则默认内存级别为最大内存级别
#define DEF_MEM_LEVEL  MAX_MEM_LEVEL
#endif
/* 默认的内存级别 */

// 定义存储块类型为0
#define STORED_BLOCK 0
// 定义静态树类型为1
#define STATIC_TREES 1
// 定义动态树类型为2
#define DYN_TREES    2
/* 三种块类型 */

// 定义最小匹配长度为3
#define MIN_MATCH  3
// 定义最大匹配长度为258
#define MAX_MATCH  258
/* 最小和最大匹配长度 */

// 在zlib头部中定义预设字典标志为0x20
#define PRESET_DICT 0x20 /* 在zlib头部中定义预设字典标志 */

        /* 目标依赖 */

// 如果是MSDOS或者是Windows但不是WIN32
#if defined(MSDOS) || (defined(WINDOWS) && !defined(WIN32))
// 定义操作系统代码为0x00
#  define OS_CODE  0x00
// 如果不是Z_SOLO
#  ifndef Z_SOLO
// 如果是__TURBOC__或者__BORLANDC__
#    if defined(__TURBOC__) || defined(__BORLANDC__)
// 如果是ANSI关键字并且是大型或紧凑模式
#      if (__STDC__ == 1) && (defined(__LARGE__) || defined(__COMPACT__))
         /* 允许仅启用ANSI关键字的编译 */
         void _Cdecl farfree( void *block );
         void *_Cdecl farmalloc( unsigned long nbytes );
#      else
#        include <alloc.h>
#      endif
#    else /* MSC或DJGPP */
#      include <malloc.h>
#    endif
#  endif
#endif

// 如果是AMIGA
#ifdef AMIGA
// 定义操作系统代码为1
#  define OS_CODE  1
#endif

// 如果是VAXC或者VMS
#if defined(VAXC) || defined(VMS)
// 定义操作系统代码为2
#  define OS_CODE  2
// 定义F_OPEN宏
#  define F_OPEN(name, mode) \
     fopen((name), (mode), "mbc=60", "ctx=stm", "rfm=fix", "mrs=512")
#endif

// 如果是__370__
#ifdef __370__
// 如果__TARGET_LIB__小于0x20000000
#  if __TARGET_LIB__ < 0x20000000
// 定义操作系统代码为4
#    define OS_CODE 4
// 如果__TARGET_LIB__小于0x40000000
#  elif __TARGET_LIB__ < 0x40000000
// 定义操作系统代码为11
#    define OS_CODE 11
#  else
// 否则定义操作系统代码为8
#    define OS_CODE 8
#  endif
#endif

// 如果是ATARI或者atarist
#if defined(ATARI) || defined(atarist)
// 定义操作系统代码为5
#  define OS_CODE  5
#endif

// 如果是OS2
#ifdef OS2
// 定义操作系统代码为6
#  define OS_CODE  6
// 如果是M_I86并且不是Z_SOLO
#  if defined(M_I86) && !defined(Z_SOLO)
#    include <malloc.h>
#  endif
#endif

// 如果是MACOS或者TARGET_OS_MAC
#if defined(MACOS) || defined(TARGET_OS_MAC)
// 定义操作系统代码为7
#  define OS_CODE  7
// 如果不是Z_SOLO
#  ifndef Z_SOLO
// 如果是__MWERKS__并且__dest_os不是__be_os并且__dest_os不是__win32_os
#    if defined(__MWERKS__) && __dest_os != __be_os && __dest_os != __win32_os
#      include <unix.h> /* 用于fdopen */
#    else
#      ifndef fdopen
#        define fdopen(fd,mode) NULL /* 没有fdopen() */
#      endif
#    endif
#  endif
#endif

// 如果是__acorn
#ifdef __acorn
// 定义操作系统代码为13
#  define OS_CODE 13
#endif

// 如果是WIN32并且不是__CYGWIN__
#if defined(WIN32) && !defined(__CYGWIN__)
// 定义操作系统代码为10
#  define OS_CODE  10
#endif

// 如果是_BEOS_
#ifdef _BEOS_
// 定义操作系统代码为16
#  define OS_CODE  16
#endif
#ifdef __TOS_OS400__
# 如果定义了 __TOS_OS400__，则定义 OS_CODE 为 18
#  define OS_CODE 18
#endif

#ifdef __APPLE__
# 如果定义了 __APPLE__，则定义 OS_CODE 为 19
#  define OS_CODE 19
#endif

#if defined(_BEOS_) || defined(RISCOS)
# 如果定义了 _BEOS_ 或 RISCOS，则定义 fdopen(fd,mode) 为 NULL，表示没有 fdopen() 函数
#  define fdopen(fd,mode) NULL /* No fdopen() */
#endif

#if (defined(_MSC_VER) && (_MSC_VER > 600)) && !defined __INTERIX
# 如果定义了 _MSC_VER 大于 600 且没有定义 __INTERIX
#  如果定义了 _WIN32_WCE，则定义 fdopen(fd,mode) 为 NULL，表示没有 fdopen() 函数
#  否则，定义 fdopen(fd,type) 为 _fdopen(fd,type)
#  if defined(_WIN32_WCE)
#    define fdopen(fd,mode) NULL /* No fdopen() */
#  else
#    define fdopen(fd,type)  _fdopen(fd,type)
#  endif
#endif

#if defined(__BORLANDC__) && !defined(MSDOS)
# 如果定义了 __BORLANDC__ 且没有定义 MSDOS
#  禁止警告 -8004
#  禁止警告 -8008
#  禁止警告 -8066
#endif

/* provide prototypes for these when building zlib without LFS */
# 如果没有定义 _WIN32 且（没有定义 _LARGEFILE64_SOURCE 或者 _LFS64_LARGEFILE-0 等于 0）
#  定义 adler32_combine64 和 crc32_combine64 的原型
#  定义 crc32_combine_gen64 的原型
#if !defined(_WIN32) && \
    (!defined(_LARGEFILE64_SOURCE) || _LFS64_LARGEFILE-0 == 0)
    ZEXTERN uLong ZEXPORT adler32_combine64 OF((uLong, uLong, z_off_t));
    ZEXTERN uLong ZEXPORT crc32_combine64 OF((uLong, uLong, z_off_t));
    ZEXTERN uLong ZEXPORT crc32_combine_gen64 OF((z_off_t));
#endif

        /* common defaults */

#ifndef OS_CODE
# 如果没有定义 OS_CODE，则定义 OS_CODE 为 3，假设为 Unix
#  define OS_CODE  3     /* assume Unix */
#endif

#ifndef F_OPEN
# 如果没有定义 F_OPEN，则定义 F_OPEN 为 fopen((name), (mode))
#  define F_OPEN(name, mode) fopen((name), (mode))
#endif

         /* functions */

#if defined(pyr) || defined(Z_SOLO)
# 如果定义了 pyr 或 Z_SOLO，则定义 NO_MEMCPY
#  define NO_MEMCPY
#endif
#if defined(SMALL_MEDIUM) && !defined(_MSC_VER) && !defined(__SC__)
# 如果定义了 SMALL_MEDIUM 且没有定义 _MSC_VER 且没有定义 __SC__
#  使用自定义函数处理小型和中型模型的情况
#  定义 NO_MEMCPY
#endif
#if defined(STDC) && !defined(HAVE_MEMCPY) && !defined(NO_MEMCPY)
# 如果定义了 STDC 且没有定义 HAVE_MEMCPY 且没有定义 NO_MEMCPY
#  定义 HAVE_MEMCPY
#endif
#ifdef HAVE_MEMCPY
# 如果定义了 HAVE_MEMCPY
#  如果定义了 SMALL_MEDIUM，则定义 zmemcpy 为 _fmemcpy，zmemcmp 为 _fmemcmp，zmemzero 为 _fmemset
#  否则，定义 zmemcpy 为 memcpy，zmemcmp 为 memcmp，zmemzero 为 memset
#  ifdef SMALL_MEDIUM /* MSDOS small or medium model */
#    define zmemcpy _fmemcpy
#    define zmemcmp _fmemcmp
#    define zmemzero(dest, len) _fmemset(dest, 0, len)
#  else
#    define zmemcpy memcpy
#    define zmemcmp memcmp
#    define zmemzero(dest, len) memset(dest, 0, len)
#  endif
#else
   void ZLIB_INTERNAL zmemcpy OF((Bytef* dest, const Bytef* source, uInt len));
   int ZLIB_INTERNAL zmemcmp OF((const Bytef* s1, const Bytef* s2, uInt len));
   void ZLIB_INTERNAL zmemzero OF((Bytef* dest, uInt len));
#endif

/* Diagnostic functions */
#ifdef ZLIB_DEBUG
#  include <stdio.h>
   extern int ZLIB_INTERNAL z_verbose;
   extern void ZLIB_INTERNAL z_error OF((char *m));
#  define Assert(cond,msg) {if(!(cond)) z_error(msg);}
#  define Trace(x) {if (z_verbose>=0) fprintf x ;}
#  define Tracev(x) {if (z_verbose>0) fprintf x ;}
#  define Tracevv(x) {if (z_verbose>1) fprintf x ;}
#  define Tracec(c,x) {if (z_verbose>0 && (c)) fprintf x ;}
#  define Tracecv(c,x) {if (z_verbose>1 && (c)) fprintf x ;}
#else
#  define Assert(cond,msg)
#  define Trace(x)
#  define Tracev(x)
#  define Tracevv(x)
#  define Tracec(c,x)
#  define Tracecv(c,x)
#endif

#ifndef Z_SOLO
   voidpf ZLIB_INTERNAL zcalloc OF((voidpf opaque, unsigned items,
                                    unsigned size));
   void ZLIB_INTERNAL zcfree  OF((voidpf opaque, voidpf ptr));
#endif

#define ZALLOC(strm, items, size) \
           (*((strm)->zalloc))((strm)->opaque, (items), (size))
#define ZFREE(strm, addr)  (*((strm)->zfree))((strm)->opaque, (voidpf)(addr))
#define TRY_FREE(s, p) {if (p) ZFREE(s, p);}

/* Reverse the bytes in a 32-bit value */
#define ZSWAP32(q) ((((q) >> 24) & 0xff) + (((q) >> 8) & 0xff00) + \
                    (((q) & 0xff00) << 8) + (((q) & 0xff) << 24))

#endif /* ZUTIL_H */
```