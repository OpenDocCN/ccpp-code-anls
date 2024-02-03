# `nmap\libz\zconf.h`

```cpp
# 定义了 zlib 压缩库的配置
# 版权信息
# 如果需要为所有类型和库函数设置唯一前缀，则使用 -DZ_PREFIX 编译
# 如果需要永久设置此前缀，可以使用 configure 在 zconf.h 中设置
#ifdef Z_PREFIX     /* may be set to #if 1 by ./configure */
#  define Z_PREFIX_SET

/* all linked symbols and init macros */
#  define _dist_code            z__dist_code
#  define _length_code          z__length_code
#  define _tr_align             z__tr_align
#  define _tr_flush_bits        z__tr_flush_bits
#  define _tr_flush_block       z__tr_flush_block
#  define _tr_init              z__tr_init
#  define _tr_stored_block      z__tr_stored_block
#  define _tr_tally             z__tr_tally
#  define adler32               z_adler32
#  define adler32_combine       z_adler32_combine
#  define adler32_combine64     z_adler32_combine64
#  define adler32_z             z_adler32_z
#  ifndef Z_SOLO
#    define compress              z_compress
#    define compress2             z_compress2
#    define compressBound         z_compressBound
#  endif
#  define crc32                 z_crc32
#  define crc32_combine         z_crc32_combine
#  define crc32_combine64       z_crc32_combine64
#  define crc32_combine_gen     z_crc32_combine_gen
#  define crc32_combine_gen64   z_crc32_combine_gen64
#  define crc32_combine_op      z_crc32_combine_op
#  define crc32_z               z_crc32_z
#  define deflate               z_deflate
#  define deflateBound          z_deflateBound
#  define deflateCopy           z_deflateCopy
#  define deflateEnd            z_deflateEnd
#  define deflateGetDictionary  z_deflateGetDictionary
#endif
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义压缩库函数名为对应的别名
# 定义获取 CRC 表的函数名为对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果是在 Windows 平台下，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 如果不是独立模式，则定义对应的别名
# 定义了一系列的宏，用于重命名对应的函数或变量
# 这些宏的作用是将对应的函数或变量重命名为以 z_ 开头的新名称
# 如果定义了 Z_SOLO 宏，则还会定义额外的两个宏
# 重新定义 zlibCompileFlags 为 z_zlibCompileFlags
# 重新定义 zlibVersion 为 z_zlibVersion

/* 所有在 zlib.h 和 zconf.h 中的 zlib typedefs */
# 重新定义 Byte 为 z_Byte
# 重新定义 Bytef 为 z_Bytef
# 重新定义 alloc_func 为 z_alloc_func
# 重新定义 charf 为 z_charf
# 重新定义 free_func 为 z_free_func
# 如果未定义 Z_SOLO，则重新定义 gzFile 为 z_gzFile
# 重新定义 gz_header 为 z_gz_header
# 重新定义 gz_headerp 为 z_gz_headerp
# 重新定义 in_func 为 z_in_func
# 重新定义 intf 为 z_intf
# 重新定义 out_func 为 z_out_func
# 重新定义 uInt 为 z_uInt
# 重新定义 uIntf 为 z_uIntf
# 重新定义 uLong 为 z_uLong
# 重新定义 uLongf 为 z_uLongf
# 重新定义 voidp 为 z_voidp
# 重新定义 voidpc 为 z_voidpc
# 重新定义 voidpf 为 z_voidpf

/* 所有在 zlib.h 和 zconf.h 中的 zlib structs */
# 重新定义 gz_header_s 为 z_gz_header_s
# 重新定义 internal_state 为 z_internal_state

#endif

#if defined(__MSDOS__) && !defined(MSDOS)
# 如果定义了 __MSDOS__ 但未定义 MSDOS，则定义 MSDOS
# 如果 (定义了 OS_2 或者 __OS2__) 但未定义 OS2，则定义 OS2
# 如果定义了 _WINDOWS 但未定义 WINDOWS，则定义 WINDOWS
# 如果定义了 _WIN32 或者 _WIN32_WCE 或者 __WIN32__，则定义 WIN32，除非已经定义了 WIN32
# 如果 (定义了 MSDOS 或者 OS2 或者 WINDOWS) 但未定义 WIN32，则在非 GNU 编译器、非 FLAT 模式、非 386 模式下定义 SYS16BIT
# 如果 SYS16BIT 已定义，则定义 MAXSEG_64K
# 如果定义了 MSDOS，则定义 UNALIGNED_OK
# 如果定义了 __STDC_VERSION__，则定义 STDC，如果 __STDC_VERSION__ 大于等于 199901L，则定义 STDC99
#endif
#if !defined(STDC) && (defined(__STDC__) || defined(__cplusplus))
#  define STDC
#endif
#if !defined(STDC) && (defined(__GNUC__) || defined(__BORLANDC__))
#  define STDC
#endif
#if !defined(STDC) && (defined(MSDOS) || defined(WINDOWS) || defined(WIN32))
#  define STDC
#endif
#if !defined(STDC) && (defined(OS2) || defined(__HOS_AIX__))
#  define STDC
#endif

#if defined(__OS400__) && !defined(STDC)    /* iSeries (formerly AS/400). */
#  define STDC
#endif

#ifndef STDC
#  ifndef const /* cannot use !defined(STDC) && !defined(const) on Mac */
#    define const       /* note: need a more gentle solution here */
#  endif
#endif

#if defined(ZLIB_CONST) && !defined(z_const)
#  define z_const const
#else
#  define z_const
#endif

#ifdef Z_SOLO
   typedef unsigned long z_size_t;
#else
#  define z_longlong long long
#  if defined(NO_SIZE_T)
     typedef unsigned NO_SIZE_T z_size_t;
#  elif defined(STDC)
#    include <stddef.h>
     typedef size_t z_size_t;
#  else
     typedef unsigned long z_size_t;
#  endif
#  undef z_longlong
#endif

/* Maximum value for memLevel in deflateInit2 */
#ifndef MAX_MEM_LEVEL
#  ifdef MAXSEG_64K
#    define MAX_MEM_LEVEL 8
#  else
#    define MAX_MEM_LEVEL 9
#  endif
#endif

/* Maximum value for windowBits in deflateInit2 and inflateInit2.
 * WARNING: reducing MAX_WBITS makes minigzip unable to extract .gz files
 * created by gzip. (Files created by minigzip can still be extracted by
 * gzip.)
 */
#ifndef MAX_WBITS
#  define MAX_WBITS   15 /* 32K LZ77 window */
#endif
/* The memory requirements for deflate are (in bytes):
            (1 << (windowBits+2)) +  (1 << (memLevel+9))
 that is: 128K for windowBits=15  +  128K for memLevel = 8  (default values)
 plus a few kilobytes for small objects. For example, if you want to reduce
 the default memory requirements from 256K to 128K, compile with
     make CFLAGS="-O -DMAX_WBITS=14 -DMAX_MEM_LEVEL=7"
 Of course this will generally degrade compression (there's no free lunch).

   The memory requirements for inflate are (in bytes) 1 << windowBits
 that is, 32K for windowBits=15 (default value) plus about 7 kilobytes
 for small objects.
*/

                        /* Type declarations */

#ifndef OF /* function prototypes */
#  ifdef STDC
#    define OF(args)  args
#  else
#    define OF(args)  ()
#  endif
#endif

#ifndef Z_ARG /* function prototypes for stdarg */
#  if defined(STDC) || defined(Z_HAVE_STDARG_H)
#    define Z_ARG(args)  args
#  else
#    define Z_ARG(args)  ()
#  endif
#endif

/* The following definitions for FAR are needed only for MSDOS mixed
 * model programming (small or medium model with some far allocations).
 * This was tested only with MSC; for other MSDOS compilers you may have
 * to define NO_MEMCPY in zutil.h.  If you don't need the mixed model,
 * just define FAR to be empty.
 */
#ifdef SYS16BIT
#  if defined(M_I86SM) || defined(M_I86MM)
     /* MSC small or medium model */
#    define SMALL_MEDIUM
#    ifdef _MSC_VER
#      define FAR _far
#    else
#      define FAR far
#    endif
#  endif
#  if (defined(__SMALL__) || defined(__MEDIUM__))
     /* Turbo C small or medium model */
#    define SMALL_MEDIUM
#    ifdef __BORLANDC__
#      define FAR _far
#    else
#      define FAR far
#    endif
#  endif
#endif

#if defined(WINDOWS) || defined(WIN32)
   /* If building or using zlib as a DLL, define ZLIB_DLL.
    * This is not mandatory, but it offers a little performance increase.
    */
#  ifdef ZLIB_DLL
# 如果定义了 WIN32 并且未定义 __BORLANDC__ 或者 __BORLANDC__ 大于等于 0x500
# 则定义 ZEXTERN 为 __declspec(dllexport) 或者 __declspec(dllimport)
# 如果 ZLIB_INTERNAL 被定义，则 ZEXTERN 为 __declspec(dllexport)，否则为 __declspec(dllimport)
# 如果 ZLIB_DLL 被定义
# 如果使用 WINAPI/WINAPIV 调用约定来构建或使用 zlib，则定义 ZLIB_WINAPI
# 注意：标准的 ZLIB1.DLL 并未使用 ZLIB_WINAPI 编译
# 如果 ZLIB_WINAPI 被定义
# 如果 FAR 被定义，则取消 FAR 的定义
# 如果未定义 WIN32_LEAN_AND_MEAN，则定义 WIN32_LEAN_AND_MEAN
# 包含 windows.h 头文件
# 不需要使用 _export，使用 ZLIB.DEF 替代
# 为了完全兼容 Windows，使用 WINAPI 而不是 __stdcall
# 定义 ZEXPORT 为 WINAPI
# 如果 WIN32 被定义，则定义 ZEXPORTVA 为 WINAPIV，否则定义为 FAR CDECL
# 如果定义了 __BEOS__
# 如果定义了 ZLIB_DLL
# 如果定义了 ZLIB_INTERNAL，则将 ZEXPORT 和 ZEXPORTVA 定义为 __declspec(dllexport)，否则定义为 __declspec(dllimport)
# 如果未定义 ZEXTERN，则定义 ZEXTERN 为 extern
# 如果未定义 ZEXPORT，则定义 ZEXPORT 为空
# 如果未定义 ZEXPORTVA，则定义 ZEXPORTVA 为空
# 如果未定义 FAR，则定义 FAR 为空
# 如果未定义 __MACTYPES__
# 定义 Byte 为 unsigned char，占 8 位
# 定义 uInt 为 unsigned int，占 16 位或更多
# 定义 uLong 为 unsigned long，占 32 位或更多
# 如果定义了 SMALL_MEDIUM
# 如果使用 Borland C/C++ 或一些旧的 MSC 版本，在 typedef 内部忽略 FAR
# 则将 Bytef 定义为 Byte FAR
# 否则，将 Bytef 定义为 FAR Byte
# 将 charf 定义为 FAR char
# 将 intf 定义为 FAR int
# 将 uIntf 定义为 FAR uInt
# 将 uLongf 定义为 FAR uLong
# 如果定义了 STDC
# 将 voidpc 定义为 void const *
# 将 voidpf 定义为 void FAR *
# 将 voidp 定义为 void *
# 否则
# 将 voidpc 定义为 Byte const *
# 将 voidpf 定义为 Byte FAR *
# 将 voidp 定义为 Byte *
#endif
#if !defined(Z_U4) && !defined(Z_SOLO) && defined(STDC)
#  include <limits.h>
#  if (UINT_MAX == 0xffffffffUL)
#    define Z_U4 unsigned
#  elif (ULONG_MAX == 0xffffffffUL)
#    define Z_U4 unsigned long
#  elif (USHRT_MAX == 0xffffffffUL)
#    define Z_U4 unsigned short
#  endif
#endif
#ifdef Z_U4
   typedef Z_U4 z_crc_t;
#else
   typedef unsigned long z_crc_t;
#endif
#ifdef HAVE_UNISTD_H    /* may be set to #if 1 by ./configure */
#  define Z_HAVE_UNISTD_H
#endif
#ifdef HAVE_STDARG_H    /* may be set to #if 1 by ./configure */
#  define Z_HAVE_STDARG_H
#endif
#ifdef STDC
#  ifndef Z_SOLO
#    include <sys/types.h>      /* for off_t */
#  endif
#endif
#if defined(STDC) || defined(Z_HAVE_STDARG_H)
#  ifndef Z_SOLO
#    include <stdarg.h>         /* for va_list */
#  endif
#endif
#ifdef _WIN32
#  ifndef Z_SOLO
#    include <stddef.h>         /* for wchar_t */
#  endif
#endif
/* a little trick to accommodate both "#define _LARGEFILE64_SOURCE" and
 * "#define _LARGEFILE64_SOURCE 1" as requesting 64-bit operations, (even
 * though the former does not conform to the LFS document), but considering
 * both "#undef _LARGEFILE64_SOURCE" and "#define _LARGEFILE64_SOURCE 0" as
 * equivalently requesting no 64-bit operations
 */
#if defined(_LARGEFILE64_SOURCE) && -_LARGEFILE64_SOURCE - -1 == 1
#  undef _LARGEFILE64_SOURCE
#endif
#ifndef Z_HAVE_UNISTD_H
#  ifdef __WATCOMC__
#    define Z_HAVE_UNISTD_H
#  endif
#endif
#ifndef Z_HAVE_UNISTD_H
#  if defined(_LARGEFILE64_SOURCE) && !defined(_WIN32)
#    define Z_HAVE_UNISTD_H
#  endif
#endif
#ifndef Z_SOLO
#  if defined(Z_HAVE_UNISTD_H)
#    include <unistd.h>         /* for SEEK_*, off_t, and _LFS64_LARGEFILE */
#    ifdef VMS
#      include <unixio.h>       /* for off_t */
#    endif
#    ifndef z_off_t
#      define z_off_t off_t
#    endif
#  endif
#endif
#if defined(_LFS64_LARGEFILE) && _LFS64_LARGEFILE-0
#  define Z_LFS64
#endif
#if defined(_LARGEFILE64_SOURCE) && defined(Z_LFS64)
#  define Z_LARGE64
#endif

#if defined(_FILE_OFFSET_BITS) && _FILE_OFFSET_BITS-0 == 64 && defined(Z_LFS64)
#  define Z_WANT64
#endif

#if !defined(SEEK_SET) && !defined(Z_SOLO)
#  define SEEK_SET        0       /* 从文件开头开始寻找 */
#  define SEEK_CUR        1       /* 从当前位置开始寻找 */
#  define SEEK_END        2       /* 设置文件指针到文件末尾再加上偏移量 */
#endif

#ifndef z_off_t
#  define z_off_t long
#endif

#if !defined(_WIN32) && defined(Z_LARGE64)
#  define z_off64_t off64_t
#else
#  if defined(_WIN32) && !defined(__GNUC__) && !defined(Z_SOLO)
#    define z_off64_t __int64
#  else
#    define z_off64_t z_off_t
#  endif
#endif

/* MVS linker does not support external names larger than 8 bytes */
#if defined(__MVS__)
  #pragma map(deflateInit_,"DEIN")
  #pragma map(deflateInit2_,"DEIN2")
  #pragma map(deflateEnd,"DEEND")
  #pragma map(deflateBound,"DEBND")
  #pragma map(inflateInit_,"ININ")
  #pragma map(inflateInit2_,"ININ2")
  #pragma map(inflateEnd,"INEND")
  #pragma map(inflateSync,"INSY")
  #pragma map(inflateSetDictionary,"INSEDI")
  #pragma map(compressBound,"CMBND")
  #pragma map(inflate_table,"INTABL")
  #pragma map(inflate_fast,"INFA")
  #pragma map(inflate_copyright,"INCOPY")
#endif

#endif /* ZCONF_H */
```