# `xmrig\src\3rdparty\rapidjson\msinttypes\inttypes.h`

```cpp
// 如果不是 Microsoft Visual Studio 编译器，则执行以下代码
#ifndef _MSC_VER // [
#error "Use this header only with Microsoft Visual C++ compilers!"
#endif // _MSC_VER ]

#ifndef _MSC_INTTYPES_H_ // [
#define _MSC_INTTYPES_H_

#if _MSC_VER > 1000
#pragma once
#endif

#include "stdint.h"

// miloyip: VC supports inttypes.h since VC2013
#if _MSC_VER >= 1800
#include <inttypes.h>
#else

// 7.8 Format conversion of integer types

typedef struct {
   intmax_t quot;
   intmax_t rem;
} imaxdiv_t;

// 7.8.1 Macros for format specifiers

#if !defined(__cplusplus) || defined(__STDC_FORMAT_MACROS) // [   See footnote 185 at page 198

// The fprintf macros for signed integers are:
#define PRId8       "d"
#define PRIi8       "i"
#define PRIdLEAST8  "d"
#define PRIiLEAST8  "i"
#define PRIdFAST8   "d"
#define PRIiFAST8   "i"

#define PRId16       "hd"
#define PRIi16       "hi"
#define PRIdLEAST16  "hd"
#define PRIiLEAST16  "hi"
#define PRIdFAST16   "hd"
#define PRIiFAST16   "hi"

#define PRId32       "I32d"
#define PRIi32       "I32i"
#define PRIdLEAST32  "I32d"
#define PRIiLEAST32  "I32i"
#define PRIdFAST32   "I32d"
#define PRIiFAST32   "I32i"

#define PRId64       "I64d"
#define PRIi64       "I64i"
#define PRIdLEAST64  "I64d"
#define PRIiLEAST64  "I64i"
#define PRIdFAST64   "I64d"
#define PRIiFAST64   "I64i"

#define PRIdMAX     "I64d"
#define PRIiMAX     "I64i"

#define PRIdPTR     "Id"
#define PRIiPTR     "Ii"

// The fprintf macros for unsigned integers are:
#define PRIo8       "o"
#define PRIu8       "u"
#define PRIx8       "x"
#define PRIX8       "X"
#define PRIoLEAST8  "o"
#define PRIuLEAST8  "u"
#define PRIxLEAST8  "x"
#define PRIXLEAST8  "X"
#define PRIoFAST8   "o"
#define PRIuFAST8   "u"
#define PRIxFAST8   "x"
#define PRIXFAST8   "X"

#define PRIo16       "ho"
#define PRIu16       "hu"
#define PRIx16       "hx"
#define PRIX16       "hX"
#define PRIoLEAST16  "ho"
#define PRIuLEAST16  "hu"
#define PRIxLEAST16  "hx"
#define PRIXLEAST16  "hX"
#define PRIoFAST16   "ho"
#define PRIuFAST16   "hu"
#define PRIxFAST16   "hx"
#define PRIXFAST16   "hX"
// 定义格式化输出宏，用于32位有符号整数的八进制、十进制、十六进制输出
#define PRIo32       "I32o"
#define PRIu32       "I32u"
#define PRIx32       "I32x"
#define PRIX32       "I32X"
// 定义格式化输出宏，用于最小宽度为32位的有符号整数的八进制、十进制、十六进制输出
#define PRIoLEAST32  "I32o"
#define PRIuLEAST32  "I32u"
#define PRIxLEAST32  "I32x"
#define PRIXLEAST32  "I32X"
// 定义格式化输出宏，用于最快速度有符号整数的八进制、十进制、十六进制输出
#define PRIoFAST32   "I32o"
#define PRIuFAST32   "I32u"
#define PRIxFAST32   "I32x"
#define PRIXFAST32   "I32X"

// 定义格式化输出宏，用于64位有符号整数的八进制、十进制、十六进制输出
#define PRIo64       "I64o"
#define PRIu64       "I64u"
#define PRIx64       "I64x"
#define PRIX64       "I64X"
// 定义格式化输出宏，用于最小宽度为64位的有符号整数的八进制、十进制、十六进制输出
#define PRIoLEAST64  "I64o"
#define PRIuLEAST64  "I64u"
#define PRIxLEAST64  "I64x"
#define PRIXLEAST64  "I64X"
// 定义格式化输出宏，用于最快速度有符号整数的八进制、十进制、十六进制输出
#define PRIoFAST64   "I64o"
#define PRIuFAST64   "I64u"
#define PRIxFAST64   "I64x"
#define PRIXFAST64   "I64X"

// 定义格式化输出宏，用于最大宽度有符号整数的八进制、十进制、十六进制输出
#define PRIoMAX     "I64o"
#define PRIuMAX     "I64u"
#define PRIxMAX     "I64x"
#define PRIXMAX     "I64X"

// 定义格式化输出宏，用于指针的八进制、十进制、十六进制输出
#define PRIoPTR     "Io"
#define PRIuPTR     "Iu"
#define PRIxPTR     "Ix"
#define PRIXPTR     "IX"

// 定义格式化输入宏，用于有符号8位整数的十进制、十六进制输入
#define SCNd8       "d"
#define SCNi8       "i"
#define SCNdLEAST8  "d"
#define SCNiLEAST8  "i"
#define SCNdFAST8   "d"
#define SCNiFAST8   "i"

// 定义格式化输入宏，用于有符号16位整数的十进制、十六进制输入
#define SCNd16       "hd"
#define SCNi16       "hi"
#define SCNdLEAST16  "hd"
#define SCNiLEAST16  "hi"
#define SCNdFAST16   "hd"
#define SCNiFAST16   "hi"

// 定义格式化输入宏，用于有符号32位整数的十进制、十六进制输入
#define SCNd32       "ld"
#define SCNi32       "li"
#define SCNdLEAST32  "ld"
#define SCNiLEAST32  "li"
#define SCNdFAST32   "ld"
#define SCNiFAST32   "li"

// 定义格式化输入宏，用于有符号64位整数的十进制、十六进制输入
#define SCNd64       "I64d"
#define SCNi64       "I64i"
#define SCNdLEAST64  "I64d"
#define SCNiLEAST64  "I64i"
#define SCNdFAST64   "I64d"
#define SCNiFAST64   "I64i"

// 定义格式化输入宏，用于最大宽度有符号整数的十进制、十六进制输入
#define SCNdMAX     "I64d"
#define SCNiMAX     "I64i"

#ifdef _WIN64 // [
// 如果是64位Windows系统，定义格式化输入宏，用于指针的十进制、十六进制输入
#  define SCNdPTR     "I64d"
#  define SCNiPTR     "I64i"
#else  // _WIN64 ][
// 如果不是64位Windows系统，定义格式化输入宏，用于指针的十进制、十六进制输入
#  define SCNdPTR     "ld"
#  define SCNiPTR     "li"
#endif  // _WIN64 ]

// 定义格式化输入宏，用于无符号8位整数的八进制、十进制、十六进制输入
#define SCNo8       "o"
#define SCNu8       "u"
#define SCNx8       "x"
#define SCNX8       "X"
// 定义格式化输入宏，用于最小宽度为8位的无符号整数的八进制、十进制输入
#define SCNoLEAST8  "o"
#define SCNuLEAST8  "u"
// 定义了用于格式化输出最小宽度为8位的整数的宏
#define SCNxLEAST8  "x"
#define SCNXLEAST8  "X"
#define SCNoFAST8   "o"
#define SCNuFAST8   "u"
#define SCNxFAST8   "x"
#define SCNXFAST8   "X"

// 定义了用于格式化输出16位整数的宏
#define SCNo16       "ho"
#define SCNu16       "hu"
#define SCNx16       "hx"
#define SCNX16       "hX"
#define SCNoLEAST16  "ho"
#define SCNuLEAST16  "hu"
#define SCNxLEAST16  "hx"
#define SCNXLEAST16  "hX"
#define SCNoFAST16   "ho"
#define SCNuFAST16   "hu"
#define SCNxFAST16   "hx"
#define SCNXFAST16   "hX"

// 定义了用于格式化输出32位整数的宏
#define SCNo32       "lo"
#define SCNu32       "lu"
#define SCNx32       "lx"
#define SCNX32       "lX"
#define SCNoLEAST32  "lo"
#define SCNuLEAST32  "lu"
#define SCNxLEAST32  "lx"
#define SCNXLEAST32  "lX"
#define SCNoFAST32   "lo"
#define SCNuFAST32   "lu"
#define SCNxFAST32   "lx"
#define SCNXFAST32   "lX"

// 定义了用于格式化输出64位整数的宏
#define SCNo64       "I64o"
#define SCNu64       "I64u"
#define SCNx64       "I64x"
#define SCNX64       "I64X"
#define SCNoLEAST64  "I64o"
#define SCNuLEAST64  "I64u"
#define SCNxLEAST64  "I64x"
#define SCNXLEAST64  "I64X"
#define SCNoFAST64   "I64o"
#define SCNuFAST64   "I64u"
#define SCNxFAST64   "I64x"
#define SCNXFAST64   "I64X"

// 定义了用于格式化输出最大宽度整数的宏
#define SCNoMAX     "I64o"
#define SCNuMAX     "I64u"
#define SCNxMAX     "I64x"
#define SCNXMAX     "I64X"

// 根据操作系统类型定义指针格式化输出的宏
#ifdef _WIN64 // [
#  define SCNoPTR     "I64o"
#  define SCNuPTR     "I64u"
#  define SCNxPTR     "I64x"
#  define SCNXPTR     "I64X"
#else  // _WIN64 ][
#  define SCNoPTR     "lo"
#  define SCNuPTR     "lu"
#  define SCNxPTR     "lx"
#  define SCNXPTR     "lX"
#endif  // _WIN64 ]

#endif // __STDC_FORMAT_MACROS ]

// 定义了用于最大宽度整数的绝对值函数的宏
#define imaxabs _abs64

// 定义了用于最大宽度整数的除法函数
#ifdef STATIC_IMAXDIV // [
static
#else // STATIC_IMAXDIV ][
_inline
#endif // STATIC_IMAXDIV ]
imaxdiv_t __cdecl imaxdiv(intmax_t numer, intmax_t denom)
{
   // 定义 imaxdiv_t 结构体类型的 result 变量

   // 计算 numer 除以 denom 的商，赋值给 result 的 quot 成员
   result.quot = numer / denom;
   // 计算 numer 除以 denom 的余数，赋值给 result 的 rem 成员
   result.rem = numer % denom;

   // 如果 numer 小于 0 且余数大于 0
   if (numer < 0 && result.rem > 0) {
      // 进行了错误的除法运算；需要修正
      // 商增加 1
      ++result.quot;
      // 余数减去 denom
      result.rem -= denom;
   }

   // 返回 result 结构体
   return result;
}

// 定义 strtoimax 和 strtoumax 宏，分别替换为 _strtoi64 和 _strtoui64
#define strtoimax _strtoi64
#define strtoumax _strtoui64

// 定义 wcstoimax 和 wcstoumax 宏，分别替换为 _wcstoi64 和 _wcstoui64
#define wcstoimax _wcstoi64
#define wcstoumax _wcstoui64

#endif // 如果 _MSC_VER 大于等于 1800

#endif // 如果 _MSC_INTTYPES_H_ 未定义
```