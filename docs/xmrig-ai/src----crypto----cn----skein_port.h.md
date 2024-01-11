# `xmrig\src\crypto\cn\skein_port.h`

```
#ifndef _SKEIN_PORT_H_
#define _SKEIN_PORT_H_

#include <limits.h>
#include <stdint.h>

#ifndef RETURN_VALUES
#  define RETURN_VALUES
#  if defined( DLL_EXPORT )
#    if defined( _MSC_VER ) || defined ( __INTEL_COMPILER )
#      define VOID_RETURN    __declspec( dllexport ) void __stdcall  // 定义 VOID_RETURN 宏，用于导出 void 返回类型的函数
#      define INT_RETURN     __declspec( dllexport ) int  __stdcall   // 定义 INT_RETURN 宏，用于导出 int 返回类型的函数
#    elif defined( __GNUC__ )
#      define VOID_RETURN    __declspec( __dllexport__ ) void          // 定义 VOID_RETURN 宏，用于导出 void 返回类型的函数
#      define INT_RETURN     __declspec( __dllexport__ ) int           // 定义 INT_RETURN 宏，用于导出 int 返回类型的函数
#    else
#      error Use of the DLL is only available on the Microsoft, Intel and GCC compilers  // 如果不是上述编译器，则报错
#    endif
#  elif defined( DLL_IMPORT )
#    if defined( _MSC_VER ) || defined ( __INTEL_COMPILER )
#      define VOID_RETURN    __declspec( dllimport ) void __stdcall    // 定义 VOID_RETURN 宏，用于导入 void 返回类型的函数
#      define INT_RETURN     __declspec( dllimport ) int  __stdcall    // 定义 INT_RETURN 宏，用于导入 int 返回类型的函数
#    elif defined( __GNUC__ )
#      define VOID_RETURN    __declspec( __dllimport__ ) void          // 定义 VOID_RETURN 宏，用于导入 void 返回类型的函数
#      define INT_RETURN     __declspec( __dllimport__ ) int           // 定义 INT_RETURN 宏，用于导入 int 返回类型的函数
#    else
#      error Use of the DLL is only available on the Microsoft, Intel and GCC compilers  // 如果不是上述编译器，则报错
#    endif
#  elif defined( __WATCOMC__ )
#    define VOID_RETURN  void __cdecl   // 定义 VOID_RETURN 宏，用于 Watcom 编译器下的 void 返回类型的函数
#    define INT_RETURN   int  __cdecl   // 定义 INT_RETURN 宏，用于 Watcom 编译器下的 int 返回类型的函数
#  else
#    define VOID_RETURN  void           // 定义 VOID_RETURN 宏，用于其他编译器下的 void 返回类型的函数
#    define INT_RETURN   int            // 定义 INT_RETURN 宏，用于其他编译器下的 int 返回类型的函数
#  endif
#endif

/*  These defines are used to declare buffers in a way that allows
    faster operations on longer variables to be used.  In all these
    defines 'size' must be a power of 2 and >= 8

    dec_unit_type(size,x)       declares a variable 'x' of length
                                'size' bits

    dec_bufr_type(size,bsize,x) declares a buffer 'x' of length 'bsize'
                                bytes defined as an array of variables
                                each of 'size' bits (bsize must be a
                                multiple of size / 8)

    ptr_cast(x,size)            casts a pointer to a pointer to a
                                varaiable of length 'size' bits
*/
// 定义一个宏，用于生成指定位数的无符号整型
#define ui_type(size)               uint##size##_t
// 定义一个宏，用于声明指定位数的无符号整型
#define dec_unit_type(size,x)       typedef ui_type(size) x
// 定义一个宏，用于声明指定位数和数组大小的无符号整型数组
#define dec_bufr_type(size,bsize,x) typedef ui_type(size) x[bsize / (size >> 3)]
// 定义一个宏，用于将指针转换为指定位数的无符号整型指针
#define ptr_cast(x,size)            ((ui_type(size)*)(x))

// 声明一个无符号整型
typedef unsigned int    uint_t;             /* native unsigned integer */
// 声明一个 8 位无符号整型
typedef uint8_t         u08b_t;             /*  8-bit unsigned integer */
// 声明一个 64 位无符号整型
typedef uint64_t        u64b_t;             /* 64-bit unsigned integer */

// 如果未定义 RotL_64 宏，则定义 RotL_64 宏
#ifndef RotL_64
#define RotL_64(x,N)    (((x) << (N)) | ((x) >> (64-(N))))
#endif

/*
 * Skein is "natively" little-endian (unlike SHA-xxx), for optimal
 * performance on x86 CPUs.  The Skein code requires the following
 * definitions for dealing with endianness:
 *
 *    SKEIN_NEED_SWAP:  0 for little-endian, 1 for big-endian
 *    Skein_Put64_LSB_First
 *    Skein_Get64_LSB_First
 *    Skein_Swap64
 *
 * If SKEIN_NEED_SWAP is defined at compile time, it is used here
 * along with the portable versions of Put64/Get64/Swap64, which
 * are slow in general.
 *
 * Otherwise, an "auto-detect" of endianness is attempted below.
 * If the default handling doesn't work well, the user may insert
 * platform-specific code instead (e.g., for big-endian CPUs).
 *
 */
// 如果未定义 SKEIN_NEED_SWAP 宏，则进行以下处理
#ifndef SKEIN_NEED_SWAP /* compile-time "override" for endianness? */

// 定义大端和小端的字节序
#define IS_BIG_ENDIAN      4321 /* byte 0 is most significant (mc68k) */
#define IS_LITTLE_ENDIAN   1234 /* byte 0 is least significant (i386) */

// 如果字节序为小端并且未定义 PLATFORM_BYTE_ORDER 宏，则定义 PLATFORM_BYTE_ORDER 为小端
#if BYTE_ORDER == LITTLE_ENDIAN && !defined(PLATFORM_BYTE_ORDER)
#  define PLATFORM_BYTE_ORDER IS_LITTLE_ENDIAN
#endif

// 如果字节序为大端并且未定义 PLATFORM_BYTE_ORDER 宏，则定义 PLATFORM_BYTE_ORDER 为大端
#if BYTE_ORDER == BIG_ENDIAN && !defined(PLATFORM_BYTE_ORDER)
#  define PLATFORM_BYTE_ORDER IS_BIG_ENDIAN
#endif

// 对于 IA64，假设为小端字节序，如果需要修改则用户可以插入特定平台的代码
#if defined(__ia64) || defined(__ia64__) || defined(_M_IA64)
#  define PLATFORM_MUST_ALIGN (1)
#ifndef PLATFORM_BYTE_ORDER
#  define PLATFORM_BYTE_ORDER IS_LITTLE_ENDIAN
#endif
#endif

#ifndef   PLATFORM_MUST_ALIGN
#  define PLATFORM_MUST_ALIGN (0)
#endif


#if   PLATFORM_BYTE_ORDER == IS_BIG_ENDIAN
    /* 如果平台的字节顺序是大端序，则执行以下代码 */
#define SKEIN_NEED_SWAP   (1)
#elif PLATFORM_BYTE_ORDER == IS_LITTLE_ENDIAN
    /* 如果平台的字节顺序是小端序，则执行以下代码 */
#define SKEIN_NEED_SWAP   (0)
#if   PLATFORM_MUST_ALIGN == 0              /* 是否可以使用“快速”版本？ */
#define Skein_Put64_LSB_First(dst08,src64,bCnt) memcpy(dst08,src64,bCnt)
#define Skein_Get64_LSB_First(dst64,src08,wCnt) memcpy(dst64,src08,8*(wCnt))
#endif
#else
#error "Skein needs endianness setting!"
#endif

#endif /* ifndef SKEIN_NEED_SWAP */

/*
 ******************************************************************
 *      提供仍然需要的任何定义。
 ******************************************************************
 */
#ifndef Skein_Swap64  /* 对于大端序进行交换，对于小端序不进行操作 */
#if     SKEIN_NEED_SWAP
#define Skein_Swap64(w64)                       \
  ( (( ((u64b_t)(w64))       & 0xFF) << 56) |   \
    (((((u64b_t)(w64)) >> 8) & 0xFF) << 48) |   \
    (((((u64b_t)(w64)) >>16) & 0xFF) << 40) |   \
    (((((u64b_t)(w64)) >>24) & 0xFF) << 32) |   \
    (((((u64b_t)(w64)) >>32) & 0xFF) << 24) |   \
    (((((u64b_t)(w64)) >>40) & 0xFF) << 16) |   \
    (((((u64b_t)(w64)) >>48) & 0xFF) <<  8) |   \
    (((((u64b_t)(w64)) >>56) & 0xFF)      ) )
#else
#define Skein_Swap64(w64)  (w64)
#endif
#endif  /* ifndef Skein_Swap64 */


#ifndef Skein_Put64_LSB_First
void    Skein_Put64_LSB_First(u08b_t *dst,const u64b_t *src,size_t bCnt)
#ifdef  SKEIN_PORT_CODE /* 在这里实例化函数代码？ */
    { /* 这个版本是完全可移植的（大端序或小端序），但速度较慢 */
    size_t n;

    for (n=0;n<bCnt;n++)
        dst[n] = (u08b_t) (src[n>>3] >> (8*(n&7)));
    }
#else
    ;    /* 仅输出函数原型 */
#endif
#endif   /* ifndef Skein_Put64_LSB_First */


#ifndef Skein_Get64_LSB_First
void    Skein_Get64_LSB_First(u64b_t *dst,const u08b_t *src,size_t wCnt)
#ifdef  SKEIN_PORT_CODE /* instantiate the function code here? */
    { /* this version is fully portable (big-endian or little-endian), but slow */
    size_t n;

    // 遍历源数据，每次处理8个字节，将其转换为64位整数存入目标数组
    for (n=0;n<8*wCnt;n+=8)
        dst[n/8] = (((u64b_t) src[n  ])      ) +
                   (((u64b_t) src[n+1]) <<  8) +
                   (((u64b_t) src[n+2]) << 16) +
                   (((u64b_t) src[n+3]) << 24) +
                   (((u64b_t) src[n+4]) << 32) +
                   (((u64b_t) src[n+5]) << 40) +
                   (((u64b_t) src[n+6]) << 48) +
                   (((u64b_t) src[n+7]) << 56) ;
    }
#else
    ;    /* output only the function prototype */
#endif
#endif   /* ifndef Skein_Get64_LSB_First */

#endif   /* ifndef _SKEIN_PORT_H_ */
```