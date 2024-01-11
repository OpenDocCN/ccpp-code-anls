# `xmrig\src\3rdparty\rapidjson\msinttypes\stdint.h`

```
// 如果不是 Microsoft Visual Studio 编译器，则执行以下代码
#ifndef _MSC_VER // [
#error "Use this header only with Microsoft Visual C++ compilers!"
#endif // _MSC_VER ]

#ifndef _MSC_STDINT_H_ // [
#define _MSC_STDINT_H_

#if _MSC_VER > 1000
#pragma once
#endif

// miloyip: Originally Visual Studio 2010 uses its own stdint.h. However it generates warning with INT64_C(), so change to use this file for vs2010.
#if _MSC_VER >= 1600 // [
#include <stdint.h>

#if !defined(__cplusplus) || defined(__STDC_CONSTANT_MACROS) // [   See footnote 224 at page 260

#undef INT8_C
#undef INT16_C
#undef INT32_C
#undef INT64_C
#undef UINT8_C
#undef UINT16_C
#undef UINT32_C
#undef UINT64_C

// 7.18.4.1 Macros for minimum-width integer constants

#define INT8_C(val)  val##i8
#define INT16_C(val) val##i16
#define INT32_C(val) val##i32
#define INT64_C(val) val##i64

#define UINT8_C(val)  val##ui8
#define UINT16_C(val) val##ui16
#define UINT32_C(val) val##ui32
#define UINT64_C(val) val##ui64

// 7.18.4.2 Macros for greatest-width integer constants
// These #ifndef's are needed to prevent collisions with <boost/cstdint.hpp>.
// Check out Issue 9 for the details.
#ifndef INTMAX_C //   [
#  define INTMAX_C   INT64_C
#endif // INTMAX_C    ]
#ifndef UINTMAX_C //  [
#  define UINTMAX_C  UINT64_C
#endif // UINTMAX_C   ]

#endif // __STDC_CONSTANT_MACROS ]

#else // ] _MSC_VER >= 1700 [

#include <limits.h>

// For Visual Studio 6 in C++ mode and for many Visual Studio versions when
// compiling for ARM we have to wrap <wchar.h> include with 'extern "C++" {}'
// or compiler would give many errors like this:
//   error C2733: second C linkage of overloaded function 'wmemchr' not allowed
#if defined(__cplusplus) && !defined(_M_ARM)
extern "C" {
#endif
#  include <wchar.h>
#if defined(__cplusplus) && !defined(_M_ARM)
}
#endif

// Define _W64 macros to mark types changing their size, like intptr_t.
#ifndef _W64
#  if !defined(__midl) && (defined(_X86_) || defined(_M_IX86)) && _MSC_VER >= 1300
#     define _W64 __w64
#  else
#     define _W64
#  endif
#endif
// 7.18.1 Integer types

// 7.18.1.1 Exact-width integer types

// 在一些旧版本的 Visual Studio（如 Visual Studio 6）和嵌入式 Visual C++ 4 中，
// 无法意识到 char 类型和 __int8 类型具有相同的大小，
// 因此我们放弃在这些版本中使用 __intX 类型。
#if (_MSC_VER < 1300)
   // 定义有符号 8 位整数类型
   typedef signed char       int8_t;
   // 定义有符号 16 位整数类型
   typedef signed short      int16_t;
   // 定义有符号 32 位整数类型
   typedef signed int        int32_t;
   // 定义无符号 8 位整数类型
   typedef unsigned char     uint8_t;
   // 定义无符号 16 位整数类型
   typedef unsigned short    uint16_t;
   // 定义无符号 32 位整数类型
   typedef unsigned int      uint32_t;
#else
   // 定义有符号 8 位整数类型
   typedef signed __int8     int8_t;
   // 定义有符号 16 位整数类型
   typedef signed __int16    int16_t;
   // 定义有符号 32 位整数类型
   typedef signed __int32    int32_t;
   // 定义无符号 8 位整数类型
   typedef unsigned __int8   uint8_t;
   // 定义无符号 16 位整数类型
   typedef unsigned __int16  uint16_t;
   // 定义无符号 32 位整数类型
   typedef unsigned __int32  uint32_t;
#endif
// 定义有符号 64 位整数类型
typedef signed __int64       int64_t;
// 定义无符号 64 位整数类型
typedef unsigned __int64     uint64_t;


// 7.18.1.2 Minimum-width integer types
// 定义最小宽度的有符号 8 位整数类型
typedef int8_t    int_least8_t;
// 定义最小宽度的有符号 16 位整数类型
typedef int16_t   int_least16_t;
// 定义最小宽度的有符号 32 位整数类型
typedef int32_t   int_least32_t;
// 定义最小宽度的有符号 64 位整数类型
typedef int64_t   int_least64_t;
// 定义最小宽度的无符号 8 位整数类型
typedef uint8_t   uint_least8_t;
// 定义最小宽度的无符号 16 位整数类型
typedef uint16_t  uint_least16_t;
// 定义最小宽度的无符号 32 位整数类型
typedef uint32_t  uint_least32_t;
// 定义最小宽度的无符号 64 位整数类型
typedef uint64_t  uint_least64_t;

// 7.18.1.3 Fastest minimum-width integer types
// 定义最快的最小宽度的有符号 8 位整数类型
typedef int8_t    int_fast8_t;
// 定义最快的最小宽度的有符号 16 位整数类型
typedef int16_t   int_fast16_t;
// 定义最快的最小宽度的有符号 32 位整数类型
typedef int32_t   int_fast32_t;
// 定义最快的最小宽度的有符号 64 位整数类型
typedef int64_t   int_fast64_t;
// 定义最快的最小宽度的无符号 8 位整数类型
typedef uint8_t   uint_fast8_t;
// 定义最快的最小宽度的无符号 16 位整数类型
typedef uint16_t  uint_fast16_t;
// 定义最快的最小宽度的无符号 32 位整数类型
typedef uint32_t  uint_fast32_t;
// 定义最快的最小宽度的无符号 64 位整数类型
typedef uint64_t  uint_fast64_t;

// 7.18.1.4 Integer types capable of holding object pointers
#ifdef _WIN64 // [
   // 定义有符号 64 位整数类型，用于存储对象指针
   typedef signed __int64    intptr_t;
   // 定义无符号 64 位整数类型，用于存储对象指针
   typedef unsigned __int64  uintptr_t;
#else // _WIN64 ][
   // 定义有符号整数类型，用于存储对象指针
   typedef _W64 signed int   intptr_t;
   // 定义无符号整数类型，用于存储对象指针
   typedef _W64 unsigned int uintptr_t;
#endif // _WIN64 ]

// 7.18.1.5 Greatest-width integer types
// 定义最大宽度的有符号 64 位整数类型
typedef int64_t   intmax_t;
// 定义最大宽度的无符号 64 位整数类型
typedef uint64_t  uintmax_t;


// 7.18.2 Limits of specified-width integer types

#if !defined(__cplusplus) || defined(__STDC_LIMIT_MACROS) // [   See footnote 220 at page 257 and footnote 221 at page 259

// 7.18.2.1 Limits of exact-width integer types
// 定义最小宽度整数类型的最小值和最大值
#define INT8_MIN     ((int8_t)_I8_MIN)
#define INT8_MAX     _I8_MAX
#define INT16_MIN    ((int16_t)_I16_MIN)
#define INT16_MAX    _I16_MAX
#define INT32_MIN    ((int32_t)_I32_MIN)
#define INT32_MAX    _I32_MAX
#define INT64_MIN    ((int64_t)_I64_MIN)
#define INT64_MAX    _I64_MAX
#define UINT8_MAX    _UI8_MAX
#define UINT16_MAX   _UI16_MAX
#define UINT32_MAX   _UI32_MAX
#define UINT64_MAX   _UI64_MAX

// 定义最小宽度整数类型的最小值和最大值
#define INT_LEAST8_MIN    INT8_MIN
#define INT_LEAST8_MAX    INT8_MAX
#define INT_LEAST16_MIN   INT16_MIN
#define INT_LEAST16_MAX   INT16_MAX
#define INT_LEAST32_MIN   INT32_MIN
#define INT_LEAST32_MAX   INT32_MAX
#define INT_LEAST64_MIN   INT64_MIN
#define INT_LEAST64_MAX   INT64_MAX
#define UINT_LEAST8_MAX   UINT8_MAX
#define UINT_LEAST16_MAX  UINT16_MAX
#define UINT_LEAST32_MAX  UINT32_MAX
#define UINT_LEAST64_MAX  UINT64_MAX

// 定义最快最小宽度整数类型的最小值和最大值
#define INT_FAST8_MIN    INT8_MIN
#define INT_FAST8_MAX    INT8_MAX
#define INT_FAST16_MIN   INT16_MIN
#define INT_FAST16_MAX   INT16_MAX
#define INT_FAST32_MIN   INT32_MIN
#define INT_FAST32_MAX   INT32_MAX
#define INT_FAST64_MIN   INT64_MIN
#define INT_FAST64_MAX   INT64_MAX
#define UINT_FAST8_MAX   UINT8_MAX
#define UINT_FAST16_MAX  UINT16_MAX
#define UINT_FAST32_MAX  UINT32_MAX
#define UINT_FAST64_MAX  UINT64_MAX

// 定义能够容纳对象指针的整数类型的最小值和最大值
#ifdef _WIN64 // [
#  define INTPTR_MIN   INT64_MIN
#  define INTPTR_MAX   INT64_MAX
#  define UINTPTR_MAX  UINT64_MAX
#else // _WIN64 ][
#  define INTPTR_MIN   INT32_MIN
#  define INTPTR_MAX   INT32_MAX
#  define UINTPTR_MAX  UINT32_MAX
#endif // _WIN64 ]

// 定义最大宽度整数类型的最小值和最大值
#define INTMAX_MIN   INT64_MIN
#define INTMAX_MAX   INT64_MAX
#define UINTMAX_MAX  UINT64_MAX

// 定义其他整数类型的限制
#ifdef _WIN64 // [
#  define PTRDIFF_MIN  _I64_MIN
#  define PTRDIFF_MAX  _I64_MAX
#else  // _WIN64 ][
// 定义 PTRDIFF_MIN 为 _I32_MIN
#define PTRDIFF_MIN  _I32_MIN
// 定义 PTRDIFF_MAX 为 _I32_MAX
#define PTRDIFF_MAX  _I32_MAX
#endif  // _WIN64 ]

// 定义 SIG_ATOMIC_MIN 为 INT_MIN
#define SIG_ATOMIC_MIN  INT_MIN
// 定义 SIG_ATOMIC_MAX 为 INT_MAX
#define SIG_ATOMIC_MAX  INT_MAX

#ifndef SIZE_MAX // [
#  ifdef _WIN64 // [
// 如果 _WIN64 定义了，定义 SIZE_MAX 为 _UI64_MAX
#     define SIZE_MAX  _UI64_MAX
#  else // _WIN64 ][
// 如果 _WIN64 未定义，定义 SIZE_MAX 为 _UI32_MAX
#     define SIZE_MAX  _UI32_MAX
#  endif // _WIN64 ]
#endif // SIZE_MAX ]

// 如果 WCHAR_MIN 未定义，定义 WCHAR_MIN 为 0
#ifndef WCHAR_MIN // [
#  define WCHAR_MIN  0
#endif  // WCHAR_MIN ]
// 如果 WCHAR_MAX 未定义，定义 WCHAR_MAX 为 _UI16_MAX
#ifndef WCHAR_MAX // [
#  define WCHAR_MAX  _UI16_MAX
#endif  // WCHAR_MAX ]

// 定义 WINT_MIN 为 0
#define WINT_MIN  0
// 定义 WINT_MAX 为 _UI16_MAX
#define WINT_MAX  _UI16_MAX

#endif // __STDC_LIMIT_MACROS ]

// 7.18.4 Limits of other integer types

#if !defined(__cplusplus) || defined(__STDC_CONSTANT_MACROS) // [   See footnote 224 at page 260

// 7.18.4.1 Macros for minimum-width integer constants

// 定义 INT8_C(val) 为 val##i8
#define INT8_C(val)  val##i8
// 定义 INT16_C(val) 为 val##i16
#define INT16_C(val) val##i16
// 定义 INT32_C(val) 为 val##i32
#define INT32_C(val) val##i32
// 定义 INT64_C(val) 为 val##i64
#define INT64_C(val) val##i64

// 定义 UINT8_C(val) 为 val##ui8
#define UINT8_C(val)  val##ui8
// 定义 UINT16_C(val) 为 val##ui16
#define UINT16_C(val) val##ui16
// 定义 UINT32_C(val) 为 val##ui32
#define UINT32_C(val) val##ui32
// 定义 UINT64_C(val) 为 val##ui64
#define UINT64_C(val) val##ui64

// 7.18.4.2 Macros for greatest-width integer constants
// 这些 #ifndef 是为了防止与 <boost/cstdint.hpp> 发生冲突。
// 详情请参阅 Issue 9。
#ifndef INTMAX_C //   [
// 如果 INTMAX_C 未定义，定义为 INT64_C
#  define INTMAX_C   INT64_C
#endif // INTMAX_C    ]
#ifndef UINTMAX_C //  [
// 如果 UINTMAX_C 未定义，定义为 UINT64_C
#  define UINTMAX_C  UINT64_C
#endif // UINTMAX_C   ]

#endif // __STDC_CONSTANT_MACROS ]

#endif // _MSC_VER >= 1600 ]

#endif // _MSC_STDINT_H_ ]
```