# `xmrig\src\3rdparty\CL\cl_platform.h`

```cpp
// 版权声明，允许在特定条件下使用和修改材料
// 版权声明和许可声明应包含在所有材料的重要部分
// 对该文件的修改可能意味着它不再准确反映 Khronos 标准
// Khronos 规范和头部信息的未修改、规范版本位于 https://www.khronos.org/registry/
// 材料按原样提供，不提供任何形式的保证
// 作者或版权持有人不对任何索赔、损害或其他责任负责
// 定义 __CL_PLATFORM_H 宏，防止重复包含
#ifndef __CL_PLATFORM_H
#define __CL_PLATFORM_H

// 包含 cl_version.h 头文件
#include <CL/cl_version.h>

// 如果是 C++ 环境，使用 C 语言调用约定
#ifdef __cplusplus
extern "C" {
#endif

// 如果是 Windows 环境
#if defined(_WIN32)
    #define CL_API_ENTRY
    #define CL_API_CALL     __stdcall
    #define CL_CALLBACK     __stdcall
// 如果不是 Windows 环境
#else
    #define CL_API_ENTRY
    #define CL_API_CALL
    #define CL_CALLBACK
#endif
/*
 * Deprecation flags refer to the last version of the header in which the
 * feature was not deprecated.
 *
 * E.g. VERSION_1_1_DEPRECATED means the feature is present in 1.1 without
 * deprecation but is deprecated in versions later than 1.1.
 */

// 定义弱链接扩展
#define CL_EXTENSION_WEAK_LINK
// 定义 OpenCL 1.0 版本的 API 后缀
#define CL_API_SUFFIX__VERSION_1_0
// 定义 OpenCL 1.0 版本的扩展后缀
#define CL_EXT_SUFFIX__VERSION_1_0
// 定义 OpenCL 1.1 版本的 API 后缀
#define CL_API_SUFFIX__VERSION_1_1
// 定义 OpenCL 1.1 版本的扩展后缀
#define CL_EXT_SUFFIX__VERSION_1_1
// 定义 OpenCL 1.2 版本的 API 后缀
#define CL_API_SUFFIX__VERSION_1_2
// 定义 OpenCL 1.2 版本的扩展后缀
#define CL_EXT_SUFFIX__VERSION_1_2
// 定义 OpenCL 2.0 版本的 API 后缀
#define CL_API_SUFFIX__VERSION_2_0
// 定义 OpenCL 2.0 版本的扩展后缀
#define CL_EXT_SUFFIX__VERSION_2_0
// 定义 OpenCL 2.1 版本的 API 后缀
#define CL_API_SUFFIX__VERSION_2_1
// 定义 OpenCL 2.1 版本的扩展后缀
#define CL_EXT_SUFFIX__VERSION_2_1
// 定义 OpenCL 2.2 版本的 API 后缀
#define CL_API_SUFFIX__VERSION_2_2
// 定义 OpenCL 2.2 版本的扩展后缀
#define CL_EXT_SUFFIX__VERSION_2_2

#ifdef __GNUC__
  // 定义已弃用的扩展后缀
  #define CL_EXT_SUFFIX_DEPRECATED __attribute__((deprecated))
  // 定义已弃用的扩展前缀
  #define CL_EXT_PREFIX_DEPRECATED
#elif defined(_WIN32)
  // 定义已弃用的扩展后缀
  #define CL_EXT_SUFFIX_DEPRECATED
  // 定义已弃用的扩展前缀
  #define CL_EXT_PREFIX_DEPRECATED __declspec(deprecated)
#else
  // 定义已弃用的扩展后缀
  #define CL_EXT_SUFFIX_DEPRECATED
  // 定义已弃用的扩展前缀
  #define CL_EXT_PREFIX_DEPRECATED
#endif

#ifdef CL_USE_DEPRECATED_OPENCL_1_0_APIS
    // 定义 OpenCL 1.0 版本的已弃用扩展后缀
    #define CL_EXT_SUFFIX__VERSION_1_0_DEPRECATED
    // 定义 OpenCL 1.0 版本的已弃用扩展前缀
    #define CL_EXT_PREFIX__VERSION_1_0_DEPRECATED
#else
    // 定义 OpenCL 1.0 版本的已弃用扩展后缀
    #define CL_EXT_SUFFIX__VERSION_1_0_DEPRECATED CL_EXT_SUFFIX_DEPRECATED
    // 定义 OpenCL 1.0 版本的已弃用扩展前缀
    #define CL_EXT_PREFIX__VERSION_1_0_DEPRECATED CL_EXT_PREFIX_DEPRECATED
#endif

#ifdef CL_USE_DEPRECATED_OPENCL_1_1_APIS
    // 定义 OpenCL 1.1 版本的已弃用扩展后缀
    #define CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED
    // 定义 OpenCL 1.1 版本的已弃用扩展前缀
    #define CL_EXT_PREFIX__VERSION_1_1_DEPRECATED
#else
    // 定义 OpenCL 1.1 版本的已弃用扩展后缀
    #define CL_EXT_SUFFIX__VERSION_1_1_DEPRECATED CL_EXT_SUFFIX_DEPRECATED
    // 定义 OpenCL 1.1 版本的已弃用扩展前缀
    #define CL_EXT_PREFIX__VERSION_1_1_DEPRECATED CL_EXT_PREFIX_DEPRECATED
#endif

#ifdef CL_USE_DEPRECATED_OPENCL_1_2_APIS
    // 定义 OpenCL 1.2 版本的已弃用扩展后缀
    #define CL_EXT_SUFFIX__VERSION_1_2_DEPRECATED
    // 定义 OpenCL 1.2 版本的已弃用扩展前缀
    #define CL_EXT_PREFIX__VERSION_1_2_DEPRECATED
#else
    // 定义 OpenCL 1.2 版本的已弃用扩展后缀
    #define CL_EXT_SUFFIX__VERSION_1_2_DEPRECATED CL_EXT_SUFFIX_DEPRECATED
    // 定义 OpenCL 1.2 版本的已弃用扩展前缀
    #define CL_EXT_PREFIX__VERSION_1_2_DEPRECATED CL_EXT_PREFIX_DEPRECATED
 #endif

#ifdef CL_USE_DEPRECATED_OPENCL_2_0_APIS
    # 定义 OpenCL 扩展后缀的版本 2.0 已经被废弃
    #define CL_EXT_SUFFIX__VERSION_2_0_DEPRECATED
    # 定义 OpenCL 扩展前缀的版本 2.0 已经被废弃
    #define CL_EXT_PREFIX__VERSION_2_0_DEPRECATED
#else
    #define CL_EXT_SUFFIX__VERSION_2_0_DEPRECATED CL_EXT_SUFFIX_DEPRECATED
    #define CL_EXT_PREFIX__VERSION_2_0_DEPRECATED CL_EXT_PREFIX_DEPRECATED
#endif

#ifdef CL_USE_DEPRECATED_OPENCL_2_1_APIS
    #define CL_EXT_SUFFIX__VERSION_2_1_DEPRECATED
    #define CL_EXT_PREFIX__VERSION_2_1_DEPRECATED
#else
    #define CL_EXT_SUFFIX__VERSION_2_1_DEPRECATED CL_EXT_SUFFIX_DEPRECATED
    #define CL_EXT_PREFIX__VERSION_2_1_DEPRECATED CL_EXT_PREFIX_DEPRECATED
#endif

#if (defined (_WIN32) && defined(_MSC_VER))

/* scalar types  */
typedef signed   __int8         cl_char;  // 定义 8 位有符号整数类型 cl_char
typedef unsigned __int8         cl_uchar;  // 定义 8 位无符号整数类型 cl_uchar
typedef signed   __int16        cl_short;  // 定义 16 位有符号整数类型 cl_short
typedef unsigned __int16        cl_ushort;  // 定义 16 位无符号整数类型 cl_ushort
typedef signed   __int32        cl_int;  // 定义 32 位有符号整数类型 cl_int
typedef unsigned __int32        cl_uint;  // 定义 32 位无符号整数类型 cl_uint
typedef signed   __int64        cl_long;  // 定义 64 位有符号整数类型 cl_long
typedef unsigned __int64        cl_ulong;  // 定义 64 位无符号整数类型 cl_ulong

typedef unsigned __int16        cl_half;  // 定义 16 位半精度浮点数类型 cl_half
typedef float                   cl_float;  // 定义 32 位单精度浮点数类型 cl_float
typedef double                  cl_double;  // 定义 64 位双精度浮点数类型 cl_double

/* Macro names and corresponding values defined by OpenCL */
#define CL_CHAR_BIT         8  // 字符类型的位数
#define CL_SCHAR_MAX        127  // 有符号字符类型的最大值
#define CL_SCHAR_MIN        (-127-1)  // 有符号字符类型的最小值
#define CL_CHAR_MAX         CL_SCHAR_MAX  // 字符类型的最大值
#define CL_CHAR_MIN         CL_SCHAR_MIN  // 字符类型的最小值
#define CL_UCHAR_MAX        255  // 无符号字符类型的最大值
#define CL_SHRT_MAX         32767  // 短整型的最大值
#define CL_SHRT_MIN         (-32767-1)  // 短整型的最小值
#define CL_USHRT_MAX        65535  // 无符号短整型的最大值
#define CL_INT_MAX          2147483647  // 整型的最大值
#define CL_INT_MIN          (-2147483647-1)  // 整型的最小值
#define CL_UINT_MAX         0xffffffffU  // 无符号整型的最大值
#define CL_LONG_MAX         ((cl_long) 0x7FFFFFFFFFFFFFFFLL)  // 长整型的最大值
#define CL_LONG_MIN         ((cl_long) -0x7FFFFFFFFFFFFFFFLL - 1LL)  // 长整型的最小值
#define CL_ULONG_MAX        ((cl_ulong) 0xFFFFFFFFFFFFFFFFULL)  // 无符号长整型的最大值

#define CL_FLT_DIG          6  // 浮点数的十进制数字位数
#define CL_FLT_MANT_DIG     24  // 浮点数的尾数位数
#define CL_FLT_MAX_10_EXP   +38  // 浮点数的最大十进制指数
#define CL_FLT_MAX_EXP      +128  // 浮点数的最大二进制指数
#define CL_FLT_MIN_10_EXP   -37  // 浮点数的最小十进制指数
#define CL_FLT_MIN_EXP      -125  // 浮点数的最小二进制指数
#define CL_FLT_RADIX        2  // 浮点数的基数
#define CL_FLT_MAX          340282346638528859811704183484516925440.0f  // 浮点数的最大值
# 定义单精度浮点数的最小正规数
#define CL_FLT_MIN          1.175494350822287507969e-38f
# 定义单精度浮点数的最小精度
#define CL_FLT_EPSILON      1.1920928955078125e-7f

# 定义半精度浮点数的有效位数
#define CL_HALF_DIG          3
# 定义半精度浮点数的尾数位数
#define CL_HALF_MANT_DIG     11
# 定义半精度浮点数的最大十进制指数
#define CL_HALF_MAX_10_EXP   +4
# 定义半精度浮点数的最大二进制指数
#define CL_HALF_MAX_EXP      +16
# 定义半精度浮点数的最小十进制指数
#define CL_HALF_MIN_10_EXP   -4
# 定义半精度浮点数的最小二进制指数
#define CL_HALF_MIN_EXP      -13
# 定义半精度浮点数的基数
#define CL_HALF_RADIX        2
# 定义半精度浮点数的最大值
#define CL_HALF_MAX          65504.0f
# 定义半精度浮点数的最小正规数
#define CL_HALF_MIN          6.103515625e-05f
# 定义半精度浮点数的最小精度
#define CL_HALF_EPSILON      9.765625e-04f

# 定义双精度浮点数的有效位数
#define CL_DBL_DIG          15
# 定义双精度浮点数的尾数位数
#define CL_DBL_MANT_DIG     53
# 定义双精度浮点数的最大十进制指数
#define CL_DBL_MAX_10_EXP   +308
# 定义双精度浮点数的最大二进制指数
#define CL_DBL_MAX_EXP      +1024
# 定义双精度浮点数的最小十进制指数
#define CL_DBL_MIN_10_EXP   -307
# 定义双精度浮点数的最小二进制指数
#define CL_DBL_MIN_EXP      -1021
# 定义双精度浮点数的基数
#define CL_DBL_RADIX        2
# 定义双精度浮点数的最大值
#define CL_DBL_MAX          1.7976931348623158e+308
# 定义双精度浮点数的最小正规数
#define CL_DBL_MIN          2.225073858507201383090e-308
# 定义双精度浮点数的最小精度
#define CL_DBL_EPSILON      2.220446049250313080847e-16

# 定义自然对数的底 e
#define CL_M_E              2.7182818284590452354
# 定义以 2 为底的对数 e
#define CL_M_LOG2E          1.4426950408889634074
# 定义以 10 为底的对数 e
#define CL_M_LOG10E         0.43429448190325182765
# 定义以 e 为底的对数 2
#define CL_M_LN2            0.69314718055994530942
# 定义以 e 为底的对数 10
#define CL_M_LN10           2.30258509299404568402
# 定义圆周率 π
#define CL_M_PI             3.14159265358979323846
# 定义 π 的一半
#define CL_M_PI_2           1.57079632679489661923
# 定义 π 的四分之一
#define CL_M_PI_4           0.78539816339744830962
# 定义 1/π
#define CL_M_1_PI           0.31830988618379067154
# 定义 2/π
#define CL_M_2_PI           0.63661977236758134308
# 定义 2/根号π
#define CL_M_2_SQRTPI       1.12837916709551257390
# 定义根号2
#define CL_M_SQRT2          1.41421356237309504880
# 定义根号1/2
#define CL_M_SQRT1_2        0.70710678118654752440

# 定义单精度浮点数的自然对数的底 e
#define CL_M_E_F            2.718281828f
# 定义单精度浮点数以 2 为底的对数 e
#define CL_M_LOG2E_F        1.442695041f
# 定义单精度浮点数以 10 为底的对数 e
#define CL_M_LOG10E_F       0.434294482f
# 定义单精度浮点数以 e 为底的对数 2
#define CL_M_LN2_F          0.693147181f
# 定义单精度浮点数以 e 为底的对数 10
#define CL_M_LN10_F         2.302585093f
# 定义单精度浮点数的圆周率 π
#define CL_M_PI_F           3.141592654f
# 定义单精度浮点数 π 的一半
#define CL_M_PI_2_F         1.570796327f
# 定义单精度浮点数 π 的四分之一
#define CL_M_PI_4_F         0.785398163f
# 定义单精度浮点数 1/π
#define CL_M_1_PI_F         0.318309886f
# 定义单精度浮点数 2/π
#define CL_M_2_PI_F         0.636619772f
# 定义单精度浮点数 2/根号π
#define CL_M_2_SQRTPI_F     1.128379167f
#define CL_M_SQRT2_F        1.414213562f
#define CL_M_SQRT1_2_F      0.707106781f

#define CL_NAN              (CL_INFINITY - CL_INFINITY)
#define CL_HUGE_VALF        ((cl_float) 1e50)
#define CL_HUGE_VAL         ((cl_double) 1e500)
#define CL_MAXFLOAT         CL_FLT_MAX
#define CL_INFINITY         CL_HUGE_VALF

#else

#include <stdint.h>

/* scalar types  */
typedef int8_t          cl_char;  // 8位有符号整数
typedef uint8_t         cl_uchar;  // 8位无符号整数
typedef int16_t         cl_short;  // 16位有符号整数
typedef uint16_t        cl_ushort;  // 16位无符号整数
typedef int32_t         cl_int;  // 32位有符号整数
typedef uint32_t        cl_uint;  // 32位无符号整数
typedef int64_t         cl_long;  // 64位有符号整数
typedef uint64_t        cl_ulong;  // 64位无符号整数

typedef uint16_t        cl_half;  // 16位浮点数
typedef float           cl_float;  // 单精度浮点数
typedef double          cl_double;  // 双精度浮点数

/* Macro names and corresponding values defined by OpenCL */
#define CL_CHAR_BIT         8  // char 类型的位数
#define CL_SCHAR_MAX        127  // 有符号 char 类型的最大值
#define CL_SCHAR_MIN        (-127-1)  // 有符号 char 类型的最小值
#define CL_CHAR_MAX         CL_SCHAR_MAX  // char 类型的最大值
#define CL_CHAR_MIN         CL_SCHAR_MIN  // char 类型的最小值
#define CL_UCHAR_MAX        255  // 无符号 char 类型的最大值
#define CL_SHRT_MAX         32767  // short 类型的最大值
#define CL_SHRT_MIN         (-32767-1)  // short 类型的最小值
#define CL_USHRT_MAX        65535  // 无符号 short 类型的最大值
#define CL_INT_MAX          2147483647  // int 类型的最大值
#define CL_INT_MIN          (-2147483647-1)  // int 类型的最小值
#define CL_UINT_MAX         0xffffffffU  // 无符号 int 类型的最大值
#define CL_LONG_MAX         ((cl_long) 0x7FFFFFFFFFFFFFFFLL)  // long 类型的最大值
#define CL_LONG_MIN         ((cl_long) -0x7FFFFFFFFFFFFFFFLL - 1LL)  // long 类型的最小值
#define CL_ULONG_MAX        ((cl_ulong) 0xFFFFFFFFFFFFFFFFULL)  // 无符号 long 类型的最大值

#define CL_FLT_DIG          6  // float 类型的有效位数
#define CL_FLT_MANT_DIG     24  // float 类型的尾数位数
#define CL_FLT_MAX_10_EXP   +38  // float 类型的最大十进制指数
#define CL_FLT_MAX_EXP      +128  // float 类型的最大二进制指数
#define CL_FLT_MIN_10_EXP   -37  // float 类型的最小十进制指数
#define CL_FLT_MIN_EXP      -125  // float 类型的最小二进制指数
#define CL_FLT_RADIX        2  // float 类型的基数
#define CL_FLT_MAX          340282346638528859811704183484516925440.0f  // float 类型的最大值
#define CL_FLT_MIN          1.175494350822287507969e-38f  // float 类型的最小值
#define CL_FLT_EPSILON      1.1920928955078125e-7f  // float 类型的最小精度

#define CL_HALF_DIG          3  // half 类型的有效位数
#define CL_HALF_MANT_DIG     11  // half 类型的尾数位数
#define CL_HALF_MAX_10_EXP   +4  // half 类型的最大十进制指数
#define CL_HALF_MAX_EXP      +16  // half 类型的最大二进制指数
#define CL_HALF_MIN_10_EXP   -4  // half 类型的最小十进制指数
// 定义半精度浮点数的最小指数
#define CL_HALF_MIN_EXP      -13
// 定义半精度浮点数的基数
#define CL_HALF_RADIX        2
// 定义半精度浮点数的最大值
#define CL_HALF_MAX          65504.0f
// 定义半精度浮点数的最小值
#define CL_HALF_MIN          6.103515625e-05f
// 定义半精度浮点数的精度
#define CL_HALF_EPSILON      9.765625e-04f

// 定义双精度浮点数的有效位数
#define CL_DBL_DIG          15
// 定义双精度浮点数的尾数位数
#define CL_DBL_MANT_DIG     53
// 定义双精度浮点数的最大的10次幂
#define CL_DBL_MAX_10_EXP   +308
// 定义双精度浮点数的最大指数
#define CL_DBL_MAX_EXP      +1024
// 定义双精度浮点数的最小的10次幂
#define CL_DBL_MIN_10_EXP   -307
// 定义双精度浮点数的最小指数
#define CL_DBL_MIN_EXP      -1021
// 定义双精度浮点数的基数
#define CL_DBL_RADIX        2
// 定义双精度浮点数的最大值
#define CL_DBL_MAX          179769313486231570814527423731704356798070567525844996598917476803157260780028538760589558632766878171540458953514382464234321326889464182768467546703537516986049910576551282076245490090389328944075868508455133942304583236903222948165808559332123348274797826204144723168738177180919299881250404026184124858368.0
// 定义双精度浮点数的最小值
#define CL_DBL_MIN          2.225073858507201383090e-308
// 定义双精度浮点数的精度
#define CL_DBL_EPSILON      2.220446049250313080847e-16

// 定义自然对数的底 e
#define CL_M_E              2.7182818284590452354
// 定义以 2 为底的对数 e
#define CL_M_LOG2E          1.4426950408889634074
// 定义以 10 为底的对数 e
#define CL_M_LOG10E         0.43429448190325182765
// 定义以 2 为底的自然对数 e
#define CL_M_LN2            0.69314718055994530942
// 定义以 10 为底的自然对数 e
#define CL_M_LN10           2.30258509299404568402
// 定义圆周率 π
#define CL_M_PI             3.14159265358979323846
// 定义 π 的一半
#define CL_M_PI_2           1.57079632679489661923
// 定义 π 的四分之一
#define CL_M_PI_4           0.78539816339744830962
// 定义 1/π
#define CL_M_1_PI           0.31830988618379067154
// 定义 2/π
#define CL_M_2_PI           0.63661977236758134308
// 定义 2/根号下π
#define CL_M_2_SQRTPI       1.12837916709551257390
// 定义根号下2
#define CL_M_SQRT2          1.41421356237309504880
// 定义根号下1/2
#define CL_M_SQRT1_2        0.70710678118654752440

// 定义单精度浮点数的自然对数的底 e
#define CL_M_E_F            2.718281828f
// 定义单精度浮点数以 2 为底的对数 e
#define CL_M_LOG2E_F        1.442695041f
// 定义单精度浮点数以 10 为底的对数 e
#define CL_M_LOG10E_F       0.434294482f
// 定义单精度浮点数以 2 为底的自然对数 e
#define CL_M_LN2_F          0.693147181f
// 定义单精度浮点数以 10 为底的自然对数 e
#define CL_M_LN10_F         2.302585093f
// 定义单精度浮点数的圆周率 π
#define CL_M_PI_F           3.141592654f
// 定义单精度浮点数 π 的一半
#define CL_M_PI_2_F         1.570796327f
// 定义单精度浮点数 π 的四分之一
#define CL_M_PI_4_F         0.785398163f
// 定义单精度浮点数 1/π
#define CL_M_1_PI_F         0.318309886f
// 定义单精度浮点数 2/π
#define CL_M_2_PI_F         0.636619772f
// 定义单精度浮点数 2/根号下π
#define CL_M_2_SQRTPI_F     1.128379167f
#ifndef CL_PLATFORM_H
#define CL_PLATFORM_H

#define CL_M_SQRT2_F        1.414213562f  // 定义浮点数常量，表示根号2
#define CL_M_SQRT1_2_F      0.707106781f  // 定义浮点数常量，表示1除以根号2

#if defined( __GNUC__ )
   #define CL_HUGE_VALF     __builtin_huge_valf()  // 如果是 GNU 编译器，使用内建函数获取浮点数常量
   #define CL_HUGE_VAL      __builtin_huge_val()   // 如果是 GNU 编译器，使用内建函数获取双精度浮点数常量
   #define CL_NAN           __builtin_nanf( "" )    // 如果是 GNU 编译器，使用内建函数获取 NaN 值
#else
   #define CL_HUGE_VALF     ((cl_float) 1e50)      // 如果不是 GNU 编译器，使用常量表示浮点数常量
   #define CL_HUGE_VAL      ((cl_double) 1e500)    // 如果不是 GNU 编译器，使用常量表示双精度浮点数常量
   float nanf( const char * );                     // 如果不是 GNU 编译器，声明获取 NaN 值的函数
   #define CL_NAN           nanf( "" )              // 如果不是 GNU 编译器，使用函数获取 NaN 值
#endif
#define CL_MAXFLOAT         CL_FLT_MAX              // 定义最大浮点数常量
#define CL_INFINITY         CL_HUGE_VALF            // 定义无穷大浮点数常量

#endif

#include <stddef.h>  // 包含标准库的头文件

/* Mirror types to GL types. Mirror types allow us to avoid deciding which 87s to load based on whether we are using GL or GLES here. */
typedef unsigned int cl_GLuint;  // 定义无符号整型变量
typedef int          cl_GLint;   // 定义整型变量
typedef unsigned int cl_GLenum;  // 定义无符号整型变量

/*
 * Vector types
 *
 *  Note:   OpenCL requires that all types be naturally aligned.
 *          This means that vector types must be naturally aligned.
 *          For example, a vector of four floats must be aligned to
 *          a 16 byte boundary (calculated as 4 * the natural 4-byte
 *          alignment of the float).  The alignment qualifiers here
 *          will only function properly if your compiler supports them
 *          and if you don't actively work to defeat them.  For example,
 *          in order for a cl_float4 to be 16 byte aligned in a struct,
 *          the start of the struct must itself be 16-byte aligned.
 *
 *          Maintaining proper alignment is the user's responsibility.
 */

/* Define basic vector types */
#if defined( __VEC__ )
   #include <altivec.h>   /* may be omitted depending on compiler. AltiVec spec provides no way to detect whether the header is required. */
   typedef __vector unsigned char     __cl_uchar16;
   typedef __vector signed char       __cl_char16;
   typedef __vector unsigned short    __cl_ushort8;
   typedef __vector signed short      __cl_short8;
   typedef __vector unsigned int      __cl_uint4;
   typedef __vector signed int        __cl_int4;
   typedef __vector float             __cl_float4;
   #define  __CL_UCHAR16__  1
   #define  __CL_CHAR16__   1
   #define  __CL_USHORT8__  1
   #define  __CL_SHORT8__   1
   #define  __CL_UINT4__    1
   #define  __CL_INT4__     1
   #define  __CL_FLOAT4__   1
#endif

#if defined( __SSE__ )
    #if defined( __MINGW64__ )
        #include <intrin.h>
    #else
        #include <xmmintrin.h>
    #endif
    #if defined( __GNUC__ )
        typedef float __cl_float4   __attribute__((vector_size(16)));
    #else
        typedef __m128 __cl_float4;
    #endif
    #define __CL_FLOAT4__   1
#endif

#if defined( __SSE2__ )
    #if defined( __MINGW64__ )
        #include <intrin.h>
    #else
        #include <emmintrin.h>
    #endif
    #if defined( __GNUC__ )
        typedef cl_uchar    __cl_uchar16    __attribute__((vector_size(16)));
        typedef cl_char     __cl_char16     __attribute__((vector_size(16)));
        typedef cl_ushort   __cl_ushort8    __attribute__((vector_size(16)));
        typedef cl_short    __cl_short8     __attribute__((vector_size(16)));
        typedef cl_uint     __cl_uint4      __attribute__((vector_size(16)));
        typedef cl_int      __cl_int4       __attribute__((vector_size(16)));
        typedef cl_ulong    __cl_ulong2     __attribute__((vector_size(16)));
        typedef cl_long     __cl_long2      __attribute__((vector_size(16)));
        typedef cl_double   __cl_double2    __attribute__((vector_size(16)));
    #else
        # 如果不满足上面的条件，则定义以下数据类型
        typedef __m128i __cl_uchar16;
        typedef __m128i __cl_char16;
        typedef __m128i __cl_ushort8;
        typedef __m128i __cl_short8;
        typedef __m128i __cl_uint4;
        typedef __m128i __cl_int4;
        typedef __m128i __cl_ulong2;
        typedef __m128i __cl_long2;
        typedef __m128d __cl_double2;
    #endif
    # 定义以下宏，表示对应的数据类型存在
    # 表示存在 16 个无符号字符的数据类型
    # 表示存在 16 个有符号字符的数据类型
    # 表示存在 8 个无符号短整型的数据类型
    # 表示存在 8 个有符号短整型的数据类型
    # 表示存在 4 个无符号整型的数据类型
    # 表示存在 4 个有符号整型的数据类型
    # 表示存在 2 个无符号长整型的数据类型
    # 表示存在 2 个有符号长整型的数据类型
    # 表示存在 2 个双精度浮点数的数据类型
    # 定义结束
    # 定义结束
    # 定义结束
    # 定义结束
    # 定义结束
    # 定义结束
    # 定义结束
    # 定义结束
    # 定义结束
#if defined( __MMX__ )
    #include <mmintrin.h>
    #if defined( __GNUC__ )
        // 定义 8 个无符号字符类型的向量
        typedef cl_uchar    __cl_uchar8     __attribute__((vector_size(8)));
        // 定义 8 个字符类型的向量
        typedef cl_char     __cl_char8      __attribute__((vector_size(8)));
        // 定义 4 个无符号短整型类型的向量
        typedef cl_ushort   __cl_ushort4    __attribute__((vector_size(8)));
        // 定义 4 个短整型类型的向量
        typedef cl_short    __cl_short4     __attribute__((vector_size(8)));
        // 定义 2 个无符号整型类型的向量
        typedef cl_uint     __cl_uint2      __attribute__((vector_size(8)));
        // 定义 2 个整型类型的向量
        typedef cl_int      __cl_int2       __attribute__((vector_size(8)));
        // 定义 1 个无符号长整型类型的向量
        typedef cl_ulong    __cl_ulong1     __attribute__((vector_size(8)));
        // 定义 1 个长整型类型的向量
        typedef cl_long     __cl_long1      __attribute__((vector_size(8)));
        // 定义 2 个单精度浮点类型的向量
        typedef cl_float    __cl_float2     __attribute__((vector_size(8)));
    #else
        // 定义 8 个无符号字符类型的向量
        typedef __m64       __cl_uchar8;
        // 定义 8 个字符类型的向量
        typedef __m64       __cl_char8;
        // 定义 4 个无符号短整型类型的向量
        typedef __m64       __cl_ushort4;
        // 定义 4 个短整型类型的向量
        typedef __m64       __cl_short4;
        // 定义 2 个无符号整型类型的向量
        typedef __m64       __cl_uint2;
        // 定义 2 个整型类型的向量
        typedef __m64       __cl_int2;
        // 定义 1 个无符号长整型类型的向量
        typedef __m64       __cl_ulong1;
        // 定义 1 个长整型类型的向量
        typedef __m64       __cl_long1;
        // 定义 2 个单精度浮点类型的向量
        typedef __m64       __cl_float2;
    #endif
    // 定义宏，表示支持 8 个无符号字符类型的向量
    #define __CL_UCHAR8__   1
    // 定义宏，表示支持 8 个字符类型的向量
    #define __CL_CHAR8__    1
    // 定义宏，表示支持 4 个无符号短整型类型的向量
    #define __CL_USHORT4__  1
    // 定义宏，表示支持 4 个短整型类型的向量
    #define __CL_SHORT4__   1
    // 定义宏，表示支持 2 个无符号整型类型的向量
    #define __CL_INT2__     1
    // 定义宏，表示支持 2 个整型类型的向量
    #define __CL_UINT2__    1
    // 定义宏，表示支持 1 个无符号长整型类型的向量
    #define __CL_ULONG1__   1
    // 定义宏，表示支持 1 个长整型类型的向量
    #define __CL_LONG1__    1
    // 定义宏，表示支持 2 个单精度浮点类型的向量
    #define __CL_FLOAT2__   1
#endif

#if defined( __AVX__ )
    #if defined( __MINGW64__ )
        #include <intrin.h>
    #else
        #include <immintrin.h>
    #endif
    #if defined( __GNUC__ )
        // 定义 8 个单精度浮点类型的向量
        typedef cl_float    __cl_float8     __attribute__((vector_size(32)));
        // 定义 4 个双精度浮点类型的向量
        typedef cl_double   __cl_double4    __attribute__((vector_size(32)));
    #else
        // 定义 8 个单精度浮点类型的向量
        typedef __m256      __cl_float8;
        // 定义 4 个双精度浮点类型的向量
        typedef __m256d     __cl_double4;
    #endif
    // 定义宏，表示支持 8 个单精度浮点类型的向量
    #define __CL_FLOAT8__   1
    // 定义宏，表示支持 4 个双精度浮点类型的向量
    #define __CL_DOUBLE4__  1
#endif

/* Define capabilities for anonymous struct members. */
#if !defined(__cplusplus) && defined(__STDC_VERSION__) && __STDC_VERSION__ >= 201112L
#define  __CL_HAS_ANON_STRUCT__ 1
#define  __CL_ANON_STRUCT__
#elif defined( __GNUC__) && ! defined( __STRICT_ANSI__ )
#define  __CL_HAS_ANON_STRUCT__ 1
#define  __CL_ANON_STRUCT__ __extension__
#elif defined( _WIN32) && defined(_MSC_VER)
    #if _MSC_VER >= 1500
   /* Microsoft Developer Studio 2008 supports anonymous structs, but
    * complains by default. */
    #define  __CL_HAS_ANON_STRUCT__ 1
    #define  __CL_ANON_STRUCT__
   /* Disable warning C4201: nonstandard extension used : nameless
    * struct/union */
    #pragma warning( push )
    #pragma warning( disable : 4201 )
    #endif
#else
#define  __CL_HAS_ANON_STRUCT__ 0
#define  __CL_ANON_STRUCT__
#endif

/* Define alignment keys */
#if defined( __GNUC__ )
    #define CL_ALIGNED(_x)          __attribute__ ((aligned(_x)))
#elif defined( _WIN32) && (_MSC_VER)
    /* Alignment keys neutered on windows because MSVC can't swallow function arguments with alignment requirements     */
    /* http://msdn.microsoft.com/en-us/library/373ak2y1%28VS.71%29.aspx                                                 */
    /* #include <crtdefs.h>                                                                                             */
    /* #define CL_ALIGNED(_x)          _CRT_ALIGN(_x)                                                                   */
    #define CL_ALIGNED(_x)
#else
   #warning  Need to implement some method to align data here
   #define  CL_ALIGNED(_x)
#endif

/* Indicate whether .xyzw, .s0123 and .hi.lo are supported */
#if __CL_HAS_ANON_STRUCT__
    /* .xyzw and .s0123...{f|F} are supported */
    #define CL_HAS_NAMED_VECTOR_FIELDS 1
    /* .hi and .lo are supported */
    #define CL_HAS_HI_LO_VECTOR_FIELDS 1
#endif

/* Define cl_vector types */

/* ---- cl_charn ---- */
typedef union
{
    cl_char  CL_ALIGNED(2) s[2];

- 如果不是 C++ 并且定义了 __STDC_VERSION__ 并且 __STDC_VERSION__ 大于等于 201112L，则定义 __CL_HAS_ANON_STRUCT__ 为 1，定义 __CL_ANON_STRUCT__ 为空
- 如果是 GNU 编译器并且不是严格 ANSI 模式，则定义 __CL_HAS_ANON_STRUCT__ 为 1，定义 __CL_ANON_STRUCT__ 为 __extension__
- 如果是 Windows 并且是 MSC 编译器，则根据 MSC 版本进行定义
- 如果是 GNU 编译器，则定义 CL_ALIGNED(_x) 为 __attribute__ ((aligned(_x)))
- 如果是 Windows 并且是 MSC 编译器，则定义 CL_ALIGNED(_x) 为空
- 否则发出警告，需要在这里实现一些方法来对齐数据
- 如果支持匿名结构体，则定义 CL_HAS_NAMED_VECTOR_FIELDS 为 1，定义 CL_HAS_HI_LO_VECTOR_FIELDS 为 1
- 定义 cl_charn 类型的联合体，包含一个 char 数组
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含两个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  x, y; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  s0, s1; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  lo, hi; };
#endif
#if defined( __CL_CHAR2__)
    // 如果定义了 __CL_CHAR2__，则定义一个 __cl_char2 类型的变量 v2
    __cl_char2     v2;
#endif
}cl_char2;

typedef union
{
    cl_char  CL_ALIGNED(4) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含四个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_char2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char2 lo, hi; };
#endif
#if defined( __CL_CHAR2__)
    // 如果定义了 __CL_CHAR2__，则定义一个包含两个 __cl_char2 类型成员的数组 v2
    __cl_char2     v2[2];
#endif
#if defined( __CL_CHAR4__)
    // 如果定义了 __CL_CHAR4__，则定义一个 __cl_char4 类型的变量 v4
    __cl_char4     v4;
#endif
}cl_char4;

/* cl_char3 is identical in size, alignment and behavior to cl_char4. See section 6.1.5. */
// cl_char3 与 cl_char4 在大小、对齐和行为上是相同的。参见第 6.1.5 节。
typedef  cl_char4  cl_char3;

typedef union
{
    cl_char   CL_ALIGNED(8) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含八个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_char4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char4 lo, hi; };
#endif
#if defined( __CL_CHAR2__)
    // 如果定义了 __CL_CHAR2__，则定义一个包含四个 __cl_char2 类型成员的数组 v2
    __cl_char2     v2[4];
#endif
#if defined( __CL_CHAR4__)
    // 如果定义了 __CL_CHAR4__，则定义一个包含两个 __cl_char4 类型成员的数组 v4
    __cl_char4     v4[2];
#endif
#if defined( __CL_CHAR8__ )
    // 如果定义了 __CL_CHAR8__，则定义一个 __cl_char8 类型的变量 v8
    __cl_char8     v8;
#endif
}cl_char8;

typedef union
{
    cl_char  CL_ALIGNED(16) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含十六个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义一个包含十六个 cl_char 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_char8 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_char8 lo, hi; };
#endif
#if defined( __CL_CHAR2__)
    // 如果定义了 __CL_CHAR2__，则定义一个包含八个 __cl_char2 类型成员的数组 v2
    __cl_char2     v2[8];
#endif
#if defined( __CL_CHAR4__)
    // 如果定义了 __CL_CHAR4__，则定义一个包含四个 __cl_char4 类型成员的数组 v4
    __cl_char4     v4[4];
#endif
#if defined( __CL_CHAR8__ )
    // 如果定义了 __CL_CHAR8__，则定义一个包含两个 __cl_char8 类型成员的数组 v8
    __cl_char8     v8[2];
#endif
#if defined( __CL_CHAR16__ )
    // 如果定义了 __CL_CHAR16__，则定义一个 __cl_char16 类型的变量 v16
    __cl_char16    v16;
#endif
}cl_char16;


/* ---- cl_ucharn ---- */
typedef union
{
    cl_uchar  CL_ALIGNED(2) s[2];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含两个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  x, y; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  s0, s1; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  lo, hi; };
#endif
#if defined( __cl_uchar2__)
    // 如果定义了 __cl_uchar2__，则定义一个 __cl_uchar2__ 类型的变量 v2
    __cl_uchar2     v2;
#endif
}cl_uchar2;

typedef union
{
    // 定义一个包含四个 cl_uchar 类型成员的数组
    cl_uchar  CL_ALIGNED(4) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含四个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_uchar2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar2 lo, hi; };
#endif
#if defined( __CL_UCHAR2__)
    // 如果定义了 __CL_UCHAR2__，则定义一个 __cl_uchar2__ 类型的数组 v2
    __cl_uchar2     v2[2];
#endif
#if defined( __CL_UCHAR4__)
    // 如果定义了 __CL_UCHAR4__，则定义一个 __cl_uchar4__ 类型的变量 v4
    __cl_uchar4     v4;
#endif
}cl_uchar4;

/* cl_uchar3 is identical in size, alignment and behavior to cl_uchar4. See section 6.1.5. */
// 定义一个与 cl_uchar4 大小、对齐和行为相同的 cl_uchar3 类型
typedef  cl_uchar4  cl_uchar3;

typedef union
{
    // 定义一个包含八个 cl_uchar 类型成员的数组
    cl_uchar   CL_ALIGNED(8) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含八个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_uchar4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar4 lo, hi; };
#endif
#if defined( __CL_UCHAR2__)
    // 如果定义了 __CL_UCHAR2__，则定义一个 __cl_uchar2__ 类型的数组 v2
    __cl_uchar2     v2[4];
#endif
#if defined( __CL_UCHAR4__)
    // 如果定义了 __CL_UCHAR4__，则定义一个 __cl_uchar4__ 类型的数组 v4
    __cl_uchar4     v4[2];
#endif
#if defined( __CL_UCHAR8__ )
    // 如果定义了 __CL_UCHAR8__，则定义一个 __cl_uchar8__ 类型的变量 v8
    __cl_uchar8     v8;
#endif
}cl_uchar8;

typedef union
{
    // 定义一个包含十六个 cl_uchar 类型成员的数组
    cl_uchar  CL_ALIGNED(16) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含十六个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义一个包含十六个 cl_uchar 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_uchar2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uchar2 lo, hi; };
#endif
#if defined( __CL_UCHAR2__)
    // 如果定义了 __CL_UCHAR2__，则定义一个 __cl_uchar2__ 类型的数组 v2
    __cl_uchar2     v2[8];
#endif
#if defined( __CL_UCHAR4__)
    // 如果定义了 __CL_UCHAR4__，则定义一个 __cl_uchar4__ 类型的数组 v4
    __cl_uchar4     v4[4];
#endif
#if defined( __CL_UCHAR8__ )
    // 如果定义了 __CL_UCHAR8__，则定义一个 __cl_uchar8__ 类型的数组 v8
    __cl_uchar8     v8[2];
#endif
#if defined( __CL_UCHAR16__ )
    // 如果定义了 __CL_UCHAR16__，则定义一个 __cl_uchar16__ 类型的变量 v16
    __cl_uchar16    v16;
#endif
}cl_uchar16;


/* ---- cl_shortn ---- */
// 继续定义其他类型的结构体，未提供完整代码
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含两个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  x, y; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  s0, s1; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  lo, hi; };
#endif
#if defined( __CL_SHORT2__)
    // 如果定义了 __CL_SHORT2__，则定义一个 __cl_short2 类型的变量 v2
    __cl_short2     v2;
#endif
}cl_short2;

typedef union
{
    cl_short  CL_ALIGNED(8) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含四个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_short2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short2 lo, hi; };
#endif
#if defined( __CL_SHORT2__)
    // 如果定义了 __CL_SHORT2__，则定义一个包含两个 __cl_short2 类型成员的数组 v2
    __cl_short2     v2[2];
#endif
#if defined( __CL_SHORT4__)
    // 如果定义了 __CL_SHORT4__，则定义一个 __cl_short4 类型的变量 v4
    __cl_short4     v4;
#endif
}cl_short4;

/* cl_short3 is identical in size, alignment and behavior to cl_short4. See section 6.1.5. */
// cl_short3 与 cl_short4 在大小、对齐和行为上是相同的。参见第 6.1.5 节。
typedef  cl_short4  cl_short3;

typedef union
{
    cl_short   CL_ALIGNED(16) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含八个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_short4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short4 lo, hi; };
#endif
#if defined( __CL_SHORT2__)
    // 如果定义了 __CL_SHORT2__，则定义一个包含四个 __cl_short2 类型成员的数组 v2
    __cl_short2     v2[4];
#endif
#if defined( __CL_SHORT4__)
    // 如果定义了 __CL_SHORT4__，则定义一个包含两个 __cl_short4 类型成员的数组 v4
    __cl_short4     v4[2];
#endif
#if defined( __CL_SHORT8__ )
    // 如果定义了 __CL_SHORT8__，则定义一个 __cl_short8 类型的变量 v8
    __cl_short8     v8;
#endif
}cl_short8;

typedef union
{
    cl_short  CL_ALIGNED(32) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含十六个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义一个包含十六个 cl_short 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_short8 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_short8 lo, hi; };
#endif
#if defined( __CL_SHORT2__)
    // 如果定义了 __CL_SHORT2__，则定义一个包含八个 __cl_short2 类型成员的数组 v2
    __cl_short2     v2[8];
#endif
#if defined( __CL_SHORT4__)
    // 如果定义了 __CL_SHORT4__，则定义一个包含四个 __cl_short4 类型成员的数组 v4
    __cl_short4     v4[4];
#endif
#if defined( __CL_SHORT8__ )
    // 如果定义了 __CL_SHORT8__，则定义一个包含两个 __cl_short8 类型成员的数组 v8
    __cl_short8     v8[2];
#endif
#if defined( __CL_SHORT16__ )
    // 如果定义了 __CL_SHORT16__，则定义一个 __cl_short16 类型的变量 v16
    __cl_short16    v16;
#endif
}cl_short16;

/* ---- cl_ushortn ---- */
typedef union
{
    cl_ushort  CL_ALIGNED(4) s[2];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含两个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  x, y; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  s0, s1; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  lo, hi; };
#endif
#if defined( __CL_USHORT2__)
    // 如果定义了 __CL_USHORT2__，则定义一个包含两个 cl_ushort 类型成员的结构体
    __cl_ushort2     v2;
#endif
}cl_ushort2;

typedef union
{
    // 定义一个包含四个 cl_ushort 类型成员的数组
    cl_ushort  CL_ALIGNED(8) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含四个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_ushort2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort2 lo, hi; };
#endif
#if defined( __CL_USHORT2__)
    // 如果定义了 __CL_USHORT2__，则定义一个包含两个 cl_ushort2 类型成员的结构体数组
    __cl_ushort2     v2[2];
#endif
#if defined( __CL_USHORT4__)
    // 如果定义了 __CL_USHORT4__，则定义一个包含四个 cl_ushort 类型成员的结构体
    __cl_ushort4     v4;
#endif
}cl_ushort4;

/* cl_ushort3 is identical in size, alignment and behavior to cl_ushort4. See section 6.1.5. */
// 定义一个与 cl_ushort4 在大小、对齐和行为上相同的 cl_ushort3 类型
typedef  cl_ushort4  cl_ushort3;

typedef union
{
    // 定义一个包含八个 cl_ushort 类型成员的数组
    cl_ushort   CL_ALIGNED(16) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含八个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_ushort4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort4 lo, hi; };
#endif
#if defined( __CL_USHORT2__)
    // 如果定义了 __CL_USHORT2__，则定义一个包含八个 cl_ushort2 类型成员的结构体数组
    __cl_ushort2     v2[4];
#endif
#if defined( __CL_USHORT4__)
    // 如果定义了 __CL_USHORT4__，则定义一个包含四个 cl_ushort4 类型成员的结构体数组
    __cl_ushort4     v4[2];
#endif
#if defined( __CL_USHORT8__ )
    // 如果定义了 __CL_USHORT8__，则定义一个包含八个 cl_ushort 类型成员的结构体
    __cl_ushort8     v8;
#endif
}cl_ushort8;

typedef union
{
    // 定义一个包含十六个 cl_ushort 类型成员的数组
    cl_ushort  CL_ALIGNED(32) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含十六个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义一个包含十六个 cl_ushort 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_ushort8 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ushort8 lo, hi; };
#endif
#if defined( __CL_USHORT2__)
    // 如果定义了 __CL_USHORT2__，则定义一个包含十六个 cl_ushort2 类型成员的结构体数组
    __cl_ushort2     v2[8];
#endif
#if defined( __CL_USHORT4__)
    // 如果定义了 __CL_USHORT4__，则定义一个包含十六个 cl_ushort4 类型成员的结构体数组
    __cl_ushort4     v4[4];
#endif
#if defined( __CL_USHORT8__ )
    // 如果定义了 __CL_USHORT8__，则定义一个包含十六个 cl_ushort 类型成员的结构体数组
    __cl_ushort8     v8[2];
#endif
#if defined( __CL_USHORT16__ )
    // 如果定义了 __CL_USHORT16__，则定义一个包含十六个 cl_ushort 类型成员的结构体
    __cl_ushort16    v16;
#endif
}cl_ushort16;


/* ---- cl_halfn ---- */
typedef union
{
    // 定义一个包含两个 cl_half 类型成员的数组
    cl_half  CL_ALIGNED(4) s[2];
#if __CL_HAS_ANON_STRUCT__
    // 如果支持匿名结构体，则定义包含两个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  x, y; };
    // 如果支持匿名结构体，则定义包含两个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  s0, s1; };
    // 如果支持匿名结构体，则定义包含两个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  lo, hi; };
#endif
#if defined( __CL_HALF2__)
    // 如果定义了 __CL_HALF2__，则定义一个 __cl_half2 类型的变量 v2
    __cl_half2     v2;
#endif
}cl_half2;

typedef union
{
    // 包含四个 cl_half 类型成员的数组
    cl_half  CL_ALIGNED(8) s[4];
#if __CL_HAS_ANON_STRUCT__
    // 如果支持匿名结构体，则定义包含四个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  x, y, z, w; };
    // 如果支持匿名结构体，则定义包含四个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  s0, s1, s2, s3; };
    // 如果支持匿名结构体，则定义包含两个 cl_half2 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half2 lo, hi; };
#endif
#if defined( __CL_HALF2__)
    // 如果定义了 __CL_HALF2__，则定义一个包含两个 __cl_half2 类型成员的数组 v2
    __cl_half2     v2[2];
#endif
#if defined( __CL_HALF4__)
    // 如果定义了 __CL_HALF4__，则定义一个 __cl_half4 类型的变量 v4
    __cl_half4     v4;
#endif
}cl_half4;

/* cl_half3 is identical in size, alignment and behavior to cl_half4. See section 6.1.5. */
// 定义 cl_half3 类型，与 cl_half4 类型在大小、对齐和行为上完全相同
typedef  cl_half4  cl_half3;

typedef union
{
    // 包含八个 cl_half 类型成员的数组
    cl_half   CL_ALIGNED(16) s[8];
#if __CL_HAS_ANON_STRUCT__
    // 如果支持匿名结构体，则定义包含八个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  x, y, z, w; };
    // 如果支持匿名结构体，则定义包含八个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  s0, s1, s2, s3, s4, s5, s6, s7; };
    // 如果支持匿名结构体，则定义包含两个 cl_half4 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half4 lo, hi; };
#endif
#if defined( __CL_HALF2__)
    // 如果定义了 __CL_HALF2__，则定义一个包含四个 __cl_half2 类型成员的数组 v2
    __cl_half2     v2[4];
#endif
#if defined( __CL_HALF4__)
    // 如果定义了 __CL_HALF4__，则定义一个包含两个 __cl_half4 类型成员的数组 v4
    __cl_half4     v4[2];
#endif
#if defined( __CL_HALF8__ )
    // 如果定义了 __CL_HALF8__，则定义一个 __cl_half8 类型的变量 v8
    __cl_half8     v8;
#endif
}cl_half8;

typedef union
{
    // 包含十六个 cl_half 类型成员的数组
    cl_half  CL_ALIGNED(32) s[16];
#if __CL_HAS_ANON_STRUCT__
    // 如果支持匿名结构体，则定义包含十六个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
    // 如果支持匿名结构体，则定义包含十六个 cl_half 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
    // 如果支持匿名结构体，则定义包含两个 cl_half8 类型成员的匿名结构体
    __CL_ANON_STRUCT__ struct{ cl_half8 lo, hi; };
#endif
#if defined( __CL_HALF2__)
    // 如果定义了 __CL_HALF2__，则定义一个包含八个 __cl_half2 类型成员的数组 v2
    __cl_half2     v2[8];
#endif
#if defined( __CL_HALF4__)
    // 如果定义了 __CL_HALF4__，则定义一个包含四个 __cl_half4 类型成员的数组 v4
    __cl_half4     v4[4];
#endif
#if defined( __CL_HALF8__ )
    // 如果定义了 __CL_HALF8__，则定义一个包含两个 __cl_half8 类型成员的数组 v8
    __cl_half8     v8[2];
#endif
#if defined( __CL_HALF16__ )
    // 如果定义了 __CL_HALF16__，则定义一个 __cl_half16 类型的变量 v16
    __cl_half16    v16;
#endif
}cl_half16;

/* ---- cl_intn ---- */
typedef union
{
    // 包含两个 cl_int 类型成员的数组
    cl_int  CL_ALIGNED(8) s[2];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含两个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  x, y; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  s0, s1; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  lo, hi; };
#endif
#if defined( __CL_INT2__)
    // 如果定义了 __CL_INT2__，则定义一个 __cl_int2 类型的变量 v2
    __cl_int2     v2;
#endif
}cl_int2;

typedef union
{
    // 定义一个包含四个 cl_int 类型成员的数组 s，且要求其对齐到 16 字节边界
    cl_int  CL_ALIGNED(16) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含四个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_int2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int2 lo, hi; };
#endif
#if defined( __CL_INT2__)
    // 如果定义了 __CL_INT2__，则定义一个包含两个 __cl_int2 类型成员的数组 v2
    __cl_int2     v2[2];
#endif
#if defined( __CL_INT4__)
    // 如果定义了 __CL_INT4__，则定义一个 __cl_int4 类型的变量 v4
    __cl_int4     v4;
#endif
}cl_int4;

/* cl_int3 is identical in size, alignment and behavior to cl_int4. See section 6.1.5. */
// 定义一个别名 cl_int3，其大小、对齐和行为与 cl_int4 相同
typedef  cl_int4  cl_int3;

typedef union
{
    // 定义一个包含八个 cl_int 类型成员的数组 s，且要求其对齐到 32 字节边界
    cl_int   CL_ALIGNED(32) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含八个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_int4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int4 lo, hi; };
#endif
#if defined( __CL_INT2__)
    // 如果定义了 __CL_INT2__，则定义一个包含四个 __cl_int2 类型成员的数组 v2
    __cl_int2     v2[4];
#endif
#if defined( __CL_INT4__)
    // 如果定义了 __CL_INT4__，则定义一个包含两个 __cl_int4 类型成员的数组 v4
    __cl_int4     v4[2];
#endif
#if defined( __CL_INT8__ )
    // 如果定义了 __CL_INT8__，则定义一个 __cl_int8 类型的变量 v8
    __cl_int8     v8;
#endif
}cl_int8;

typedef union
{
    // 定义一个包含十六个 cl_int 类型成员的数组 s，且要求其对齐到 64 字节边界
    cl_int  CL_ALIGNED(64) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含十六个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义一个包含十六个 cl_int 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_int8 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_int8 lo, hi; };
#endif
#if defined( __CL_INT2__)
    // 如果定义了 __CL_INT2__，则定义一个包含八个 __cl_int2 类型成员的数组 v2
    __cl_int2     v2[8];
#endif
#if defined( __CL_INT4__)
    // 如果定义了 __CL_INT4__，则定义一个包含四个 __cl_int4 类型成员的数组 v4
    __cl_int4     v4[4];
#endif
#if defined( __CL_INT8__ )
    // 如果定义了 __CL_INT8__，则定义一个包含两个 __cl_int8 类型成员的数组 v8
    __cl_int8     v8[2];
#endif
#if defined( __CL_INT16__ )
    // 如果定义了 __CL_INT16__，则定义一个 __cl_int16 类型的变量 v16
    __cl_int16    v16;
#endif
}cl_int16;

/* ---- cl_uintn ---- */
typedef union
{
    // 定义一个包含两个 cl_uint 类型成员的数组 s，且要求其对齐到 8 字节边界
    cl_uint  CL_ALIGNED(8) s[2];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含两个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  x, y; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  s0, s1; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  lo, hi; };
#endif
#if defined( __CL_UINT2__)
    // 如果定义了 __CL_UINT2__，则定义一个包含两个 cl_uint 类型成员的结构体
    __cl_uint2     v2;
#endif
}cl_uint2;

typedef union
{
    cl_uint  CL_ALIGNED(16) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含四个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_uint2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint2 lo, hi; };
#endif
#if defined( __CL_UINT2__)
    // 如果定义了 __CL_UINT2__，则定义一个包含两个 __cl_uint2 类型成员的结构体
    __cl_uint2     v2[2];
#endif
#if defined( __CL_UINT4__)
    // 如果定义了 __CL_UINT4__，则定义一个包含四个 cl_uint 类型成员的结构体
    __cl_uint4     v4;
#endif
}cl_uint4;

/* cl_uint3 is identical in size, alignment and behavior to cl_uint4. See section 6.1.5. */
// cl_uint3 与 cl_uint4 在大小、对齐和行为上是相同的。参见第 6.1.5 节。
typedef  cl_uint4  cl_uint3;

typedef union
{
    cl_uint   CL_ALIGNED(32) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含八个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_uint4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint4 lo, hi; };
#endif
#if defined( __CL_UINT2__)
    // 如果定义了 __CL_UINT2__，则定义一个包含四个 __cl_uint2 类型成员的结构体
    __cl_uint2     v2[4];
#endif
#if defined( __CL_UINT4__)
    // 如果定义了 __CL_UINT4__，则定义一个包含四个 cl_uint4 类型成员的结构体
    __cl_uint4     v4[2];
#endif
#if defined( __CL_UINT8__ )
    // 如果定义了 __CL_UINT8__，则定义一个包含八个 cl_uint 类型成员的结构体
    __cl_uint8     v8;
#endif
}cl_uint8;

typedef union
{
    cl_uint  CL_ALIGNED(64) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含十六个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义一个包含十六个 cl_uint 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_uint2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_uint8 lo, hi; };
#endif
#if defined( __CL_UINT2__)
    // 如果定义了 __CL_UINT2__，则定义一个包含八个 __cl_uint2 类型成员的结构体
    __cl_uint2     v2[8];
#endif
#if defined( __CL_UINT4__)
    // 如果定义了 __CL_UINT4__，则定义一个包含四个 cl_uint4 类型成员的结构体
    __cl_uint4     v4[4];
#endif
#if defined( __CL_UINT8__ )
    // 如果定义了 __CL_UINT8__，则定义一个包含八个 cl_uint 类型成员的结构体
    __cl_uint8     v8[2];
#endif
#if defined( __CL_UINT16__ )
    // 如果定义了 __CL_UINT16__，则定义一个包含十六个 cl_uint 类型成员的结构体
    __cl_uint16    v16;
#endif
}cl_uint16;

/* ---- cl_longn ---- */
typedef union
{
    cl_long  CL_ALIGNED(16) s[2];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含两个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  x, y; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  s0, s1; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  lo, hi; };
#endif
#if defined( __CL_LONG2__)
    // 如果定义了 __CL_LONG2__，则定义一个 __cl_long2 类型变量
    __cl_long2     v2;
#endif
}cl_long2;

typedef union
{
    cl_long  CL_ALIGNED(32) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含四个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_long2 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long2 lo, hi; };
#endif
#if defined( __CL_LONG2__)
    // 如果定义了 __CL_LONG2__，则定义一个包含两个 __cl_long2 类型变量的数组
    __cl_long2     v2[2];
#endif
#if defined( __CL_LONG4__)
    // 如果定义了 __CL_LONG4__，则定义一个 __cl_long4 类型变量
    __cl_long4     v4;
#endif
}cl_long4;

/* cl_long3 is identical in size, alignment and behavior to cl_long4. See section 6.1.5. */
// cl_long3 与 cl_long4 在大小、对齐和行为上是相同的。参见第 6.1.5 节。
typedef  cl_long4  cl_long3;

typedef union
{
    cl_long   CL_ALIGNED(64) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含八个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_long4 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long4 lo, hi; };
#endif
#if defined( __CL_LONG2__)
    // 如果定义了 __CL_LONG2__，则定义一个包含四个 __cl_long2 类型变量的数组
    __cl_long2     v2[4];
#endif
#if defined( __CL_LONG4__)
    // 如果定义了 __CL_LONG4__，则定义一个包含两个 __cl_long4 类型变量的数组
    __cl_long4     v4[2];
#endif
#if defined( __CL_LONG8__ )
    // 如果定义了 __CL_LONG8__，则定义一个 __cl_long8 类型变量
    __cl_long8     v8;
#endif
}cl_long8;

typedef union
{
    cl_long  CL_ALIGNED(128) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含十六个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义一个包含十六个 cl_long 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_long8 类型变量的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_long8 lo, hi; };
#endif
#if defined( __CL_LONG2__)
    // 如果定义了 __CL_LONG2__，则定义一个包含八个 __cl_long2 类型变量的数组
    __cl_long2     v2[8];
#endif
#if defined( __CL_LONG4__)
    // 如果定义了 __CL_LONG4__，则定义一个包含四个 __cl_long4 类型变量的数组
    __cl_long4     v4[4];
#endif
#if defined( __CL_LONG8__ )
    // 如果定义了 __CL_LONG8__，则定义一个包含两个 __cl_long8 类型变量的数组
    __cl_long8     v8[2];
#endif
#if defined( __CL_LONG16__ )
    // 如果定义了 __CL_LONG16__，则定义一个 __cl_long16 类型变量
    __cl_long16    v16;
#endif
}cl_long16;


/* ---- cl_ulongn ---- */
typedef union
{
    cl_ulong  CL_ALIGNED(16) s[2];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义包含两个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  x, y; };
   // 如果支持匿名结构体，则定义包含两个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  s0, s1; };
   // 如果支持匿名结构体，则定义包含两个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  lo, hi; };
#endif
#if defined( __CL_ULONG2__)
    // 如果定义了 __CL_ULONG2__，则定义一个包含两个 cl_ulong 类型成员的结构体
    __cl_ulong2     v2;
#endif
}cl_ulong2;

typedef union
{
    cl_ulong  CL_ALIGNED(32) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义包含四个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  x, y, z, w; };
   // 如果支持匿名结构体，则定义包含四个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义包含两个 cl_ulong2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong2 lo, hi; };
#endif
#if defined( __CL_ULONG2__)
    // 如果定义了 __CL_ULONG2__，则定义一个包含两个 cl_ulong2 类型成员的结构体数组
    __cl_ulong2     v2[2];
#endif
#if defined( __CL_ULONG4__)
    // 如果定义了 __CL_ULONG4__，则定义一个包含四个 cl_ulong 类型成员的结构体
    __cl_ulong4     v4;
#endif
}cl_ulong4;

/* cl_ulong3 is identical in size, alignment and behavior to cl_ulong4. See section 6.1.5. */
// cl_ulong3 与 cl_ulong4 在大小、对齐和行为上是相同的。参见第 6.1.5 节。
typedef  cl_ulong4  cl_ulong3;

typedef union
{
    cl_ulong   CL_ALIGNED(64) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义包含八个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  x, y, z, w; };
   // 如果支持匿名结构体，则定义包含八个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义包含两个 cl_ulong4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong4 lo, hi; };
#endif
#if defined( __CL_ULONG2__)
    // 如果定义了 __CL_ULONG2__，则定义一个包含四个 cl_ulong2 类型成员的结构体数组
    __cl_ulong2     v2[4];
#endif
#if defined( __CL_ULONG4__)
    // 如果定义了 __CL_ULONG4__，则定义一个包含四个 cl_ulong4 类型成员的结构体数组
    __cl_ulong4     v4[2];
#endif
#if defined( __CL_ULONG8__ )
    // 如果定义了 __CL_ULONG8__，则定义一个包含八个 cl_ulong 类型成员的结构体
    __cl_ulong8     v8;
#endif
}cl_ulong8;

typedef union
{
    cl_ulong  CL_ALIGNED(128) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义包含十六个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义包含十六个 cl_ulong 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义包含两个 cl_ulong8 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_ulong8 lo, hi; };
#endif
#if defined( __CL_ULONG2__)
    // 如果定义了 __CL_ULONG2__，则定义一个包含十六个 cl_ulong2 类型成员的结构体数组
    __cl_ulong2     v2[8];
#endif
#if defined( __CL_ULONG4__)
    // 如果定义了 __CL_ULONG4__，则定义一个包含十六个 cl_ulong4 类型成员的结构体数组
    __cl_ulong4     v4[4];
#endif
#if defined( __CL_ULONG8__ )
    // 如果定义了 __CL_ULONG8__，则定义一个包含十六个 cl_ulong8 类型成员的结构体数组
    __cl_ulong8     v8[2];
#endif
#if defined( __CL_ULONG16__ )
    // 如果定义了 __CL_ULONG16__，则定义一个包含十六个 cl_ulong 类型成员的结构体
    __cl_ulong16    v16;
#endif
}cl_ulong16;


/* --- cl_floatn ---- */

typedef union
{
    cl_float  CL_ALIGNED(8) s[2];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含两个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float  x, y; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float  s0, s1; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float  lo, hi; };
#endif
#if defined( __CL_FLOAT2__)
    // 如果定义了 __CL_FLOAT2__，则定义一个 __cl_float2 类型的变量 v2
    __cl_float2     v2;
#endif
}cl_float2;

typedef union
{
    // 定义一个包含四个 cl_float 类型成员的数组，并且要求 16 字节对齐
    cl_float  CL_ALIGNED(16) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含四个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float   x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含四个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float   s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_float2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float2  lo, hi; };
#endif
#if defined( __CL_FLOAT2__)
    // 如果定义了 __CL_FLOAT2__，则定义一个包含两个 __cl_float2 类型成员的数组
    __cl_float2     v2[2];
#endif
#if defined( __CL_FLOAT4__)
    // 如果定义了 __CL_FLOAT4__，则定义一个 __cl_float4 类型的变量 v4
    __cl_float4     v4;
#endif
}cl_float4;

/* cl_float3 is identical in size, alignment and behavior to cl_float4. See section 6.1.5. */
// 定义 cl_float3 类型，其大小、对齐和行为与 cl_float4 相同
typedef  cl_float4  cl_float3;

typedef union
{
    // 定义一个包含八个 cl_float 类型成员的数组，并且要求 32 字节对齐
    cl_float   CL_ALIGNED(32) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含八个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float   x, y, z, w; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float   s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义一个包含两个 cl_float4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float4  lo, hi; };
#endif
#if defined( __CL_FLOAT2__)
    // 如果定义了 __CL_FLOAT2__，则定义一个包含四个 __cl_float2 类型成员的数组
    __cl_float2     v2[4];
#endif
#if defined( __CL_FLOAT4__)
    // 如果定义了 __CL_FLOAT4__，则定义一个包含两个 __cl_float4 类型成员的数组
    __cl_float4     v4[2];
#endif
#if defined( __CL_FLOAT8__ )
    // 如果定义了 __CL_FLOAT8__，则定义一个 __cl_float8 类型的变量 v8
    __cl_float8     v8;
#endif
}cl_float8;

typedef union
{
    // 定义一个包含十六个 cl_float 类型成员的数组，并且要求 64 字节对齐
    cl_float  CL_ALIGNED(64) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义一个包含十六个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义一个包含十六个 cl_float 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义一个包含八个 cl_float8 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_float8 lo, hi; };
#endif
#if defined( __CL_FLOAT2__)
    // 如果定义了 __CL_FLOAT2__，则定义一个包含八个 __cl_float2 类型成员的数组
    __cl_float2     v2[8];
#endif
#if defined( __CL_FLOAT4__)
    // 如果定义了 __CL_FLOAT4__，则定义一个包含四个 __cl_float4 类型成员的数组
    __cl_float4     v4[4];
#endif
#if defined( __CL_FLOAT8__ )
    // 如果定义了 __CL_FLOAT8__，则定义一个包含两个 __cl_float8 类型成员的数组
    __cl_float8     v8[2];
#endif
#if defined( __CL_FLOAT16__ )
    // 如果定义了 __CL_FLOAT16__，则定义一个 __cl_float16 类型的变量 v16
    __cl_float16    v16;
#endif
}cl_float16;

/* --- cl_doublen ---- */

typedef union
{
    // 定义一个包含两个 cl_double 类型成员的数组，并且要求 16 字节对齐
    cl_double  CL_ALIGNED(16) s[2];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义包含两个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double  x, y; };
   // 如果支持匿名结构体，则定义包含两个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double s0, s1; };
   // 如果支持匿名结构体，则定义包含两个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double lo, hi; };
#endif
#if defined( __CL_DOUBLE2__)
    // 如果定义了 __CL_DOUBLE2__，则定义一个 __cl_double2 类型的变量 v2
    __cl_double2     v2;
#endif
}cl_double2;

typedef union
{
    // 定义一个包含四个 cl_double 类型成员的数组 s，对齐到 32 字节
    cl_double  CL_ALIGNED(32) s[4];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义包含四个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double  x, y, z, w; };
   // 如果支持匿名结构体，则定义包含四个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double  s0, s1, s2, s3; };
   // 如果支持匿名结构体，则定义包含两个 cl_double2 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double2 lo, hi; };
#endif
#if defined( __CL_DOUBLE2__)
    // 如果定义了 __CL_DOUBLE2__，则定义一个包含两个 __cl_double2 类型成员的数组 v2
    __cl_double2     v2[2];
#endif
#if defined( __CL_DOUBLE4__)
    // 如果定义了 __CL_DOUBLE4__，则定义一个 __cl_double4 类型的变量 v4
    __cl_double4     v4;
#endif
}cl_double4;

/* cl_double3 is identical in size, alignment and behavior to cl_double4. See section 6.1.5. */
// 定义一个别名 cl_double3，与 cl_double4 在大小、对齐和行为上完全相同
typedef  cl_double4  cl_double3;

typedef union
{
    // 定义一个包含八个 cl_double 类型成员的数组 s，对齐到 64 字节
    cl_double   CL_ALIGNED(64) s[8];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义包含八个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double  x, y, z, w; };
   // 如果支持匿名结构体，则定义包含八个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double  s0, s1, s2, s3, s4, s5, s6, s7; };
   // 如果支持匿名结构体，则定义包含两个 cl_double4 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double4 lo, hi; };
#endif
#if defined( __CL_DOUBLE2__)
    // 如果定义了 __CL_DOUBLE2__，则定义一个包含四个 __cl_double2 类型成员的数组 v2
    __cl_double2     v2[4];
#endif
#if defined( __CL_DOUBLE4__)
    // 如果定义了 __CL_DOUBLE4__，则定义一个包含两个 __cl_double4 类型成员的数组 v4
    __cl_double4     v4[2];
#endif
#if defined( __CL_DOUBLE8__ )
    // 如果定义了 __CL_DOUBLE8__，则定义一个 __cl_double8 类型的变量 v8
    __cl_double8     v8;
#endif
}cl_double8;

typedef union
{
    // 定义一个包含十六个 cl_double 类型成员的数组 s，对齐到 128 字节
    cl_double  CL_ALIGNED(128) s[16];
#if __CL_HAS_ANON_STRUCT__
   // 如果支持匿名结构体，则定义包含十六个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double  x, y, z, w, __spacer4, __spacer5, __spacer6, __spacer7, __spacer8, __spacer9, sa, sb, sc, sd, se, sf; };
   // 如果支持匿名结构体，则定义包含十六个 cl_double 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double  s0, s1, s2, s3, s4, s5, s6, s7, s8, s9, sA, sB, sC, sD, sE, sF; };
   // 如果支持匿名结构体，则定义包含两个 cl_double8 类型成员的匿名结构体
   __CL_ANON_STRUCT__ struct{ cl_double8 lo, hi; };
#endif
#if defined( __CL_DOUBLE2__)
    // 如果定义了 __CL_DOUBLE2__，则定义一个包含八个 __cl_double2 类型成员的数组 v2
    __cl_double2     v2[8];
#endif
#if defined( __CL_DOUBLE4__)
    // 如果定义了 __CL_DOUBLE4__，则定义一个包含四个 __cl_double4 类型成员的数组 v4
    __cl_double4     v4[4];
#endif
#if defined( __CL_DOUBLE8__ )
    // 如果定义了 __CL_DOUBLE8__，则定义一个包含两个 __cl_double8 类型成员的数组 v8
    __cl_double8     v8[2];
#endif
#if defined( __CL_DOUBLE16__ )
    // 如果定义了 __CL_DOUBLE16__，则定义一个 __cl_double16 类型的变量 v16
    __cl_double16    v16;
#endif
}cl_double16;
/* 宏定义，用于便于调试
 * 用法：
 *   将 CL_PROGRAM_STRING_DEBUG_INFO 放在源代码的第一行之前
 *   第一行以 CL_PROGRAM_STRING_DEBUG_INFO \ 结尾
 *   其后的每一行 OpenCL C 源代码必须以 \n\ 结尾
 *   最后一行以 "; 结尾
 *
 *   示例：
 *
 *   const char *my_program = CL_PROGRAM_STRING_DEBUG_INFO "\
 *   kernel void foo( int a, float * b )             \n\
 *   {                                               \n\
 *      // my comment                                \n\
 *      *b[ get_global_id(0)] = a;                   \n\
 *   }                                               \n\
 *   ";
 *
 * 这样可以正确设置源字符串的行、列和文件信息，以便进行源级调试。
 */
#define  __CL_STRINGIFY( _x )               # _x
#define  _CL_STRINGIFY( _x )                __CL_STRINGIFY( _x )
#define  CL_PROGRAM_STRING_DEBUG_INFO       "#line "  _CL_STRINGIFY(__LINE__) " \"" __FILE__ "\" \n\n"

#ifdef __cplusplus
}
#endif

#undef __CL_HAS_ANON_STRUCT__
#undef __CL_ANON_STRUCT__
#if defined( _WIN32) && defined(_MSC_VER)
    #if _MSC_VER >=1500
    #pragma warning( pop )
    #endif
#endif

#endif  /* __CL_PLATFORM_H  */
```