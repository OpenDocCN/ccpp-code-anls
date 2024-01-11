# `xmrig\src\3rdparty\rapidjson\rapidjson.h`

```
// 定义了 RapidJSON 的版本信息和许可证信息
// Tencent 公司和 Milo Yip 版权所有，遵循 MIT 许可证
// 可以在遵守许可证的情况下使用该文件
// 获取许可证的副本，请访问 http://opensource.org/licenses/MIT
// 根据许可证规定，软件以"原样"的基础分发，没有任何明示或暗示的担保或条件
// 请查看许可证以了解特定语言下的权限和限制

#ifndef RAPIDJSON_RAPIDJSON_H_
#define RAPIDJSON_RAPIDJSON_H_

/*!\file rapidjson.h
    \brief common definitions and configuration
    \see RAPIDJSON_CONFIG
 */

/*! \defgroup RAPIDJSON_CONFIG RapidJSON configuration
    \brief Configuration macros for library features
    一些 RapidJSON 功能是可配置的，以适应各种平台、环境和使用场景。
    大多数功能可以在编译时通过预定义的预处理宏进行配置。
    一些额外的定制可以在 \ref RAPIDJSON_ERRORS APIs 中进行。

    \note 这些宏应该在编译器命令行中给出（在适用的情况下），以避免在编译单个应用程序的不同翻译单元时出现不一致的值。
 */

#include <cstdlib>  // malloc(), realloc(), free(), size_t
#include <cstring>  // memset(), memcpy(), memmove(), memcmp()

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_VERSION_STRING
//
// ALWAYS synchronize the following 3 macros with corresponding variables in /CMakeLists.txt.
//

//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN
// token stringification
#define RAPIDJSON_STRINGIFY(x) RAPIDJSON_DO_STRINGIFY(x)
// 定义宏，将参数 x 转换为字符串
#define RAPIDJSON_DO_STRINGIFY(x) #x

// 宏：连接两个参数
#define RAPIDJSON_JOIN(X, Y) RAPIDJSON_DO_JOIN(X, Y)
#define RAPIDJSON_DO_JOIN(X, Y) RAPIDJSON_DO_JOIN2(X, Y)
#define RAPIDJSON_DO_JOIN2(X, Y) X##Y
//!@endcond

/*! \def RAPIDJSON_MAJOR_VERSION
    \ingroup RAPIDJSON_CONFIG
    \brief RapidJSON 的主要版本号，以整数表示。
*/
/*! \def RAPIDJSON_MINOR_VERSION
    \ingroup RAPIDJSON_CONFIG
    \brief RapidJSON 的次要版本号，以整数表示。
*/
/*! \def RAPIDJSON_PATCH_VERSION
    \ingroup RAPIDJSON_CONFIG
    \brief RapidJSON 的修订版本号，以整数表示。
*/
/*! \def RAPIDJSON_VERSION_STRING
    \ingroup RAPIDJSON_CONFIG
    \brief RapidJSON 的版本号，以 "<major>.<minor>.<patch>" 字符串格式表示。
*/
#define RAPIDJSON_MAJOR_VERSION 1
#define RAPIDJSON_MINOR_VERSION 1
#define RAPIDJSON_PATCH_VERSION 0
#define RAPIDJSON_VERSION_STRING \
    RAPIDJSON_STRINGIFY(RAPIDJSON_MAJOR_VERSION.RAPIDJSON_MINOR_VERSION.RAPIDJSON_PATCH_VERSION)

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_NAMESPACE_(BEGIN|END)
/*! \def RAPIDJSON_NAMESPACE
    \ingroup RAPIDJSON_CONFIG
    \brief 提供自定义的 rapidjson 命名空间

    为了避免在单个二进制文件中多次包含（不同版本的）RapidJSON 导致的符号冲突和/或“一次定义规则”错误，用户可以自定义主要的 RapidJSON 命名空间的名称。

    如果只需要单层嵌套，将 \c RAPIDJSON_NAMESPACE 定义为自定义名称（例如 \c MyRapidJSON）就足够了。如果需要多层嵌套，则还需要定义 \ref RAPIDJSON_NAMESPACE_BEGIN 和 \ref RAPIDJSON_NAMESPACE_END：

    \code
    // 在某个 .cpp 文件中
    #define RAPIDJSON_NAMESPACE my::rapidjson
    #define RAPIDJSON_NAMESPACE_BEGIN namespace my { namespace rapidjson {
    #define RAPIDJSON_NAMESPACE_END   } }
    #include "rapidjson/..."
    \endcode

    \see rapidjson
 */
/*! \def RAPIDJSON_NAMESPACE_BEGIN
    \ingroup RAPIDJSON_CONFIG
    # 提供自定义的 rapidjson 命名空间（开放表达式）
    # 参见 RAPIDJSON_NAMESPACE
/*! \def RAPIDJSON_NAMESPACE_END
    \ingroup RAPIDJSON_CONFIG
    \brief   provide custom rapidjson namespace (closing expression)
    \see RAPIDJSON_NAMESPACE
*/
#ifndef RAPIDJSON_NAMESPACE
#define RAPIDJSON_NAMESPACE rapidjson
#endif
#ifndef RAPIDJSON_NAMESPACE_BEGIN
#define RAPIDJSON_NAMESPACE_BEGIN namespace RAPIDJSON_NAMESPACE {
#endif
#ifndef RAPIDJSON_NAMESPACE_END
#define RAPIDJSON_NAMESPACE_END }
#endif



///////////////////////////////////////////////////////////////////////////////
// __cplusplus macro

//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN

#if defined(_MSC_VER)
#define RAPIDJSON_CPLUSPLUS _MSVC_LANG
#else
#define RAPIDJSON_CPLUSPLUS __cplusplus
#endif

//!@endcond



///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_HAS_STDSTRING

#ifndef RAPIDJSON_HAS_STDSTRING
#ifdef RAPIDJSON_DOXYGEN_RUNNING
#define RAPIDJSON_HAS_STDSTRING 1 // force generation of documentation
#else
#define RAPIDJSON_HAS_STDSTRING 0 // no std::string support by default
#endif
/*! \def RAPIDJSON_HAS_STDSTRING
    \ingroup RAPIDJSON_CONFIG
    \brief Enable RapidJSON support for \c std::string

    By defining this preprocessor symbol to \c 1, several convenience functions for using
    \ref rapidjson::GenericValue with \c std::string are enabled, especially
    for construction and comparison.

    \hideinitializer
*/
#endif // !defined(RAPIDJSON_HAS_STDSTRING)



#if RAPIDJSON_HAS_STDSTRING
#include <string>
#endif // RAPIDJSON_HAS_STDSTRING



///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_USE_MEMBERSMAP

/*! \def RAPIDJSON_USE_MEMBERSMAP
    \ingroup RAPIDJSON_CONFIG
    \brief Enable RapidJSON support for object members handling in a \c std::multimap

    By defining this preprocessor symbol to \c 1, \ref rapidjson::GenericValue object
    members are stored in a \c std::multimap for faster lookup and deletion times, a
    # 通过稍微减慢插入时间和分配少量内存来进行权衡
    # 隐藏初始化
#ifndef RAPIDJSON_USE_MEMBERSMAP
#define RAPIDJSON_USE_MEMBERSMAP 0 // not by default
#endif

// 如果未定义 RAPIDJSON_USE_MEMBERSMAP，则将其定义为 0，表示默认情况下不使用成员映射


/*! \def RAPIDJSON_NO_INT64DEFINE
    \ingroup RAPIDJSON_CONFIG
    \brief Use external 64-bit integer types.

    RapidJSON requires the 64-bit integer types \c int64_t and  \c uint64_t types
    to be available at global scope.

    If users have their own definition, define RAPIDJSON_NO_INT64DEFINE to
    prevent RapidJSON from defining its own types.
*/
#ifndef RAPIDJSON_NO_INT64DEFINE
//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN
#if defined(_MSC_VER) && (_MSC_VER < 1800)    // Visual Studio 2013
#include "msinttypes/stdint.h"
#include "msinttypes/inttypes.h"
#else
// Other compilers should have this.
#include <stdint.h>
#include <inttypes.h>
#endif
//!@endcond
#ifdef RAPIDJSON_DOXYGEN_RUNNING
#define RAPIDJSON_NO_INT64DEFINE
#endif
#endif // RAPIDJSON_NO_INT64TYPEDEF

// 如果未定义 RAPIDJSON_NO_INT64DEFINE，则根据不同的编译器定义 int64_t 和 uint64_t 类型，以便 RapidJSON 使用


#ifndef RAPIDJSON_FORCEINLINE
//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN
#if defined(_MSC_VER) && defined(NDEBUG)
#define RAPIDJSON_FORCEINLINE __forceinline
#elif defined(__GNUC__) && __GNUC__ >= 4 && defined(NDEBUG)
#define RAPIDJSON_FORCEINLINE __attribute__((always_inline))
#else
#define RAPIDJSON_FORCEINLINE
#endif
//!@endcond
#endif // RAPIDJSON_FORCEINLINE

// 如果未定义 RAPIDJSON_FORCEINLINE，则根据不同的编译器定义内联函数的方式


#define RAPIDJSON_LITTLEENDIAN  0   //!< Little endian machine
#define RAPIDJSON_BIGENDIAN     1   //!< Big endian machine

//! Endianness of the machine.
/*!
    \def RAPIDJSON_ENDIAN
    \ingroup RAPIDJSON_CONFIG

    GCC 4.6 provided macro for detecting endianness of the target machine. But other
    compilers may not have this. User can define RAPIDJSON_ENDIAN to either
    \ref RAPIDJSON_LITTLEENDIAN or \ref RAPIDJSON_BIGENDIAN.

// 定义了机器的大小端模式，用户可以根据实际情况定义 RAPIDJSON_ENDIAN 为 RAPIDJSON_LITTLEENDIAN 或 RAPIDJSON_BIGENDIAN
    # 使用默认检测实现，参考了以下链接：
    # https://gcc.gnu.org/onlinedocs/gcc-4.6.0/cpp/Common-Predefined-Macros.html
    # http://www.boost.org/doc/libs/1_42_0/boost/detail/endian.hpp
#ifndef RAPIDJSON_ENDIAN
// 如果 RAPIDJSON_ENDIAN 未定义，则进行以下操作

// 使用 GCC 4.6 的宏来检测机器的字节序
#  ifdef __BYTE_ORDER__
#    if __BYTE_ORDER__ == __ORDER_LITTLE_ENDIAN__
#      define RAPIDJSON_ENDIAN RAPIDJSON_LITTLEENDIAN
#    elif __BYTE_ORDER__ == __ORDER_BIG_ENDIAN__
#      define RAPIDJSON_ENDIAN RAPIDJSON_BIGENDIAN
#    else
#      error Unknown machine endianness detected. User needs to define RAPIDJSON_ENDIAN.
#    endif // __BYTE_ORDER__

// 使用 GLIBC 的 endian.h 来检测机器的字节序
#  elif defined(__GLIBC__)
#    include <endian.h>
#    if (__BYTE_ORDER == __LITTLE_ENDIAN)
#      define RAPIDJSON_ENDIAN RAPIDJSON_LITTLEENDIAN
#    elif (__BYTE_ORDER == __BIG_ENDIAN)
#      define RAPIDJSON_ENDIAN RAPIDJSON_BIGENDIAN
#    else
#      error Unknown machine endianness detected. User needs to define RAPIDJSON_ENDIAN.
#   endif // __GLIBC__

// 使用 _LITTLE_ENDIAN 和 _BIG_ENDIAN 宏来检测机器的字节序
#  elif defined(_LITTLE_ENDIAN) && !defined(_BIG_ENDIAN)
#    define RAPIDJSON_ENDIAN RAPIDJSON_LITTLEENDIAN
#  elif defined(_BIG_ENDIAN) && !defined(_LITTLE_ENDIAN)
#    define RAPIDJSON_ENDIAN RAPIDJSON_BIGENDIAN

// 使用架构宏来检测机器的字节序
#  elif defined(__sparc) || defined(__sparc__) || defined(_POWER) || defined(__powerpc__) || defined(__ppc__) || defined(__hpux) || defined(__hppa) || defined(_MIPSEB) || defined(_POWER) || defined(__s390__)
#    define RAPIDJSON_ENDIAN RAPIDJSON_BIGENDIAN
#  elif defined(__i386__) || defined(__alpha__) || defined(__ia64) || defined(__ia64__) || defined(_M_IX86) || defined(_M_IA64) || defined(_M_ALPHA) || defined(__amd64__) || defined(_M_AMD64) || defined(__x86_64__) || defined(_M_X64) || defined(__bfin__)
#    define RAPIDJSON_ENDIAN RAPIDJSON_LITTLEENDIAN
#  elif defined(_MSC_VER) && (defined(_M_ARM) || defined(_M_ARM64))
#    define RAPIDJSON_ENDIAN RAPIDJSON_LITTLEENDIAN
#  elif defined(RAPIDJSON_DOXYGEN_RUNNING)
#    define RAPIDJSON_ENDIAN
#  else
#ifndef RAPIDJSON_ENDIAN
// 如果未定义 RAPIDJSON_ENDIAN，则报错：检测到未知的机器字节序。用户需要定义 RAPIDJSON_ENDIAN。
#    error Unknown machine endianness detected. User needs to define RAPIDJSON_ENDIAN.
#  endif
#endif // RAPIDJSON_ENDIAN

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_64BIT

//! 是否使用 64 位架构
#ifndef RAPIDJSON_64BIT
// 如果未定义 RAPIDJSON_64BIT
#if defined(__LP64__) || (defined(__x86_64__) && defined(__ILP32__)) || defined(_WIN64) || defined(__EMSCRIPTEN__)
// 如果满足条件：LP64 或 (x86_64 且 ILP32) 或 WIN64 或 EMSCRIPTEN，则定义为 1
#define RAPIDJSON_64BIT 1
// 否则定义为 0
#else
#define RAPIDJSON_64BIT 0
#endif
#endif // RAPIDJSON_64BIT

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_ALIGN

//! 机器的数据对齐方式
/*! \ingroup RAPIDJSON_CONFIG
    \param x 指针对齐

    一些机器需要严格的数据对齐。默认为 8 字节。
    用户可以通过定义 RAPIDJSON_ALIGN 函数宏来自定义。
*/
#ifndef RAPIDJSON_ALIGN
// 如果未定义 RAPIDJSON_ALIGN
#define RAPIDJSON_ALIGN(x) (((x) + static_cast<size_t>(7u)) & ~static_cast<size_t>(7u))
#endif

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_UINT64_C2

//! 通过一对 32 位整数构造 64 位字面量
/*!
    64 位字面量是否带有 ULL 后缀容易引起编译器警告。
    UINT64_C() 是会引起编译问题的 C 宏。
    使用这个宏通过一对 32 位整数定义 64 位常量。
*/
#ifndef RAPIDJSON_UINT64_C2
// 如果未定义 RAPIDJSON_UINT64_C2
#define RAPIDJSON_UINT64_C2(high32, low32) ((static_cast<uint64_t>(high32) << 32) | static_cast<uint64_t>(low32))
#endif

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_48BITPOINTER_OPTIMIZATION

//! 仅使用低 48 位地址的一些指针
/*!
    \ingroup RAPIDJSON_CONFIG

    这种优化利用了当前 X86-64 架构只实现了低 48 位虚拟地址的事实。
    高 16 位可以用于存储其他数据。
    GenericValue 使用这种优化，在 64 位架构下将其大小从 24 字节减小到 16 字节。
*/
#ifndef RAPIDJSON_48BITPOINTER_OPTIMIZATION
#if defined(__amd64__) || defined(__amd64) || defined(__x86_64__) || defined(__x86_64) || defined(_M_X64) || defined(_M_AMD64)
#define RAPIDJSON_48BITPOINTER_OPTIMIZATION 1
#else
#define RAPIDJSON_48BITPOINTER_OPTIMIZATION 0
#endif
#endif // RAPIDJSON_48BITPOINTER_OPTIMIZATION

#if RAPIDJSON_48BITPOINTER_OPTIMIZATION == 1
#if RAPIDJSON_64BIT != 1
#error RAPIDJSON_48BITPOINTER_OPTIMIZATION can only be set to 1 when RAPIDJSON_64BIT=1
#endif
#define RAPIDJSON_SETPOINTER(type, p, x) (p = reinterpret_cast<type *>((reinterpret_cast<uintptr_t>(p) & static_cast<uintptr_t>(RAPIDJSON_UINT64_C2(0xFFFF0000, 0x00000000))) | reinterpret_cast<uintptr_t>(reinterpret_cast<const void*>(x))))
#define RAPIDJSON_GETPOINTER(type, p) (reinterpret_cast<type *>(reinterpret_cast<uintptr_t>(p) & static_cast<uintptr_t>(RAPIDJSON_UINT64_C2(0x0000FFFF, 0xFFFFFFFF))))
#else
#define RAPIDJSON_SETPOINTER(type, p, x) (p = (x))
#define RAPIDJSON_GETPOINTER(type, p) (p)
#endif

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_SSE2/RAPIDJSON_SSE42/RAPIDJSON_NEON/RAPIDJSON_SIMD

/*! \def RAPIDJSON_SIMD
    \ingroup RAPIDJSON_CONFIG
    \brief Enable SSE2/SSE4.2/Neon optimization.

    RapidJSON supports optimized implementations for some parsing operations
    based on the SSE2, SSE4.2 or NEon SIMD extensions on modern Intel
    or ARM compatible processors.

    To enable these optimizations, three different symbols can be defined;
    \code
    // Enable SSE2 optimization.
    #define RAPIDJSON_SSE2

    // Enable SSE4.2 optimization.
    #define RAPIDJSON_SSE42
    \endcode

    // Enable ARM Neon optimization.
    #define RAPIDJSON_NEON
    \endcode

    \c RAPIDJSON_SSE42 takes precedence over SSE2, if both are defined.

    If any of these symbols is defined, RapidJSON defines the macro
    \c RAPIDJSON_SIMD to indicate the availability of the optimized code.
*/
#if defined(RAPIDJSON_SSE2) || defined(RAPIDJSON_SSE42) \
    || defined(RAPIDJSON_NEON) || defined(RAPIDJSON_DOXYGEN_RUNNING)
#define RAPIDJSON_SIMD
#endif

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_NO_SIZETYPEDEFINE

#ifndef RAPIDJSON_NO_SIZETYPEDEFINE
/*! \def RAPIDJSON_NO_SIZETYPEDEFINE
    \ingroup RAPIDJSON_CONFIG
    \brief User-provided \c SizeType definition.

    In order to avoid using 32-bit size types for indexing strings and arrays,
    define this preprocessor symbol and provide the type rapidjson::SizeType
    before including RapidJSON:
    \code
    #define RAPIDJSON_NO_SIZETYPEDEFINE
    namespace rapidjson { typedef ::std::size_t SizeType; }
    #include "rapidjson/..."
    \endcode

    \see rapidjson::SizeType
*/
#ifdef RAPIDJSON_DOXYGEN_RUNNING
#define RAPIDJSON_NO_SIZETYPEDEFINE
#endif
RAPIDJSON_NAMESPACE_BEGIN
//! Size type (for string lengths, array sizes, etc.)
/*! RapidJSON uses 32-bit array/string indices even on 64-bit platforms,
    instead of using \c size_t. Users may override the SizeType by defining
    \ref RAPIDJSON_NO_SIZETYPEDEFINE.
*/
typedef unsigned SizeType;
RAPIDJSON_NAMESPACE_END
#endif

// always import std::size_t to rapidjson namespace
RAPIDJSON_NAMESPACE_BEGIN
using std::size_t;
RAPIDJSON_NAMESPACE_END

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_ASSERT

//! Assertion.
/*! \ingroup RAPIDJSON_CONFIG
    By default, rapidjson uses C \c assert() for internal assertions.
    User can override it by defining RAPIDJSON_ASSERT(x) macro.

    \note Parsing errors are handled and can be customized by the
          \ref RAPIDJSON_ERRORS APIs.
*/
#ifndef RAPIDJSON_ASSERT
#include <cassert>
#define RAPIDJSON_ASSERT(x) assert(x)
#endif // RAPIDJSON_ASSERT

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_STATIC_ASSERT

// Prefer C++11 static_assert, if available
#ifndef RAPIDJSON_STATIC_ASSERT
#if RAPIDJSON_CPLUSPLUS >= 201103L || ( defined(_MSC_VER) && _MSC_VER >= 1800 )
#define RAPIDJSON_STATIC_ASSERT(x) \
   static_assert(x, RAPIDJSON_STRINGIFY(x))
#endif // C++11
#endif // RAPIDJSON_STATIC_ASSERT

// 采用 boost 的 C++03 实现
#ifndef RAPIDJSON_STATIC_ASSERT
#ifndef __clang__
//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN
#endif
RAPIDJSON_NAMESPACE_BEGIN
template <bool x> struct STATIC_ASSERTION_FAILURE;
template <> struct STATIC_ASSERTION_FAILURE<true> { enum { value = 1 }; };
template <size_t x> struct StaticAssertTest {};
RAPIDJSON_NAMESPACE_END

#if defined(__GNUC__) || defined(__clang__)
#define RAPIDJSON_STATIC_ASSERT_UNUSED_ATTRIBUTE __attribute__((unused))
#else
#define RAPIDJSON_STATIC_ASSERT_UNUSED_ATTRIBUTE
#endif
#ifndef __clang__
//!@endcond
#endif

/*! \def RAPIDJSON_STATIC_ASSERT
    \brief (Internal) macro to check for conditions at compile-time
    \param x compile-time condition
    \hideinitializer
 */
#define RAPIDJSON_STATIC_ASSERT(x) \
    typedef ::RAPIDJSON_NAMESPACE::StaticAssertTest< \
      sizeof(::RAPIDJSON_NAMESPACE::STATIC_ASSERTION_FAILURE<bool(x) >)> \
    RAPIDJSON_JOIN(StaticAssertTypedef, __LINE__) RAPIDJSON_STATIC_ASSERT_UNUSED_ATTRIBUTE
#endif // RAPIDJSON_STATIC_ASSERT

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_LIKELY, RAPIDJSON_UNLIKELY

//! 编译器分支提示，表达式可能为真
/*!
    \ingroup RAPIDJSON_CONFIG
    \param x 可能为真的布尔表达式
*/
#ifndef RAPIDJSON_LIKELY
#if defined(__GNUC__) || defined(__clang__)
#define RAPIDJSON_LIKELY(x) __builtin_expect(!!(x), 1)
#else
#define RAPIDJSON_LIKELY(x) (x)
#endif
#endif

//! 编译器分支提示，表达式可能为假
/*!
    \ingroup RAPIDJSON_CONFIG
    \param x 可能为假的布尔表达式
*/
#ifndef RAPIDJSON_UNLIKELY
#if defined(__GNUC__) || defined(__clang__)
// 定义宏，用于指示条件不太可能发生
#define RAPIDJSON_UNLIKELY(x) __builtin_expect(!!(x), 0)
#else
#define RAPIDJSON_UNLIKELY(x) (x)
#endif
#endif

///////////////////////////////////////////////////////////////////////////////
// Helpers

//!@cond RAPIDJSON_HIDDEN_FROM_DOXYGEN

// 定义多行宏的开始和结束
#define RAPIDJSON_MULTILINEMACRO_BEGIN do {
#define RAPIDJSON_MULTILINEMACRO_END \
} while((void)0, 0)

// 定义版本号宏
#define RAPIDJSON_VERSION_CODE(x,y,z) \
  (((x)*100000) + ((y)*100) + (z))

// 检查是否有内建函数的宏
#if defined(__has_builtin)
#define RAPIDJSON_HAS_BUILTIN(x) __has_builtin(x)
#else
#define RAPIDJSON_HAS_BUILTIN(x) 0
#endif

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_DIAG_PUSH/POP, RAPIDJSON_DIAG_OFF

// 如果是 GNU 编译器
#if defined(__GNUC__)
#define RAPIDJSON_GNUC \
    RAPIDJSON_VERSION_CODE(__GNUC__,__GNUC_MINOR__,__GNUC_PATCHLEVEL__)
#endif

// 如果是 clang 或者是 GCC>=4.2.0
#if defined(__clang__) || (defined(RAPIDJSON_GNUC) && RAPIDJSON_GNUC >= RAPIDJSON_VERSION_CODE(4,2,0))

// 定义 pragma 宏
#define RAPIDJSON_PRAGMA(x) _Pragma(RAPIDJSON_STRINGIFY(x))
#define RAPIDJSON_DIAG_PRAGMA(x) RAPIDJSON_PRAGMA(GCC diagnostic x)
#define RAPIDJSON_DIAG_OFF(x) \
    RAPIDJSON_DIAG_PRAGMA(ignored RAPIDJSON_STRINGIFY(RAPIDJSON_JOIN(-W,x)))

// 在 Clang 和 GCC>=4.6 中支持 push/pop
#if defined(__clang__) || (defined(RAPIDJSON_GNUC) && RAPIDJSON_GNUC >= RAPIDJSON_VERSION_CODE(4,6,0))
#define RAPIDJSON_DIAG_PUSH RAPIDJSON_DIAG_PRAGMA(push)
#define RAPIDJSON_DIAG_POP  RAPIDJSON_DIAG_PRAGMA(pop)
#else // GCC >= 4.2, < 4.6
#define RAPIDJSON_DIAG_PUSH /* ignored */
#define RAPIDJSON_DIAG_POP /* ignored */
#endif

// 如果是 MSC 编译器
#elif defined(_MSC_VER)

// pragma (MSVC specific)
#define RAPIDJSON_PRAGMA(x) __pragma(x)
#define RAPIDJSON_DIAG_PRAGMA(x) RAPIDJSON_PRAGMA(warning(x))

#define RAPIDJSON_DIAG_OFF(x) RAPIDJSON_DIAG_PRAGMA(disable: x)
#define RAPIDJSON_DIAG_PUSH RAPIDJSON_DIAG_PRAGMA(push)
#define RAPIDJSON_DIAG_POP  RAPIDJSON_DIAG_PRAGMA(pop)

// 其他情况
#else

#define RAPIDJSON_DIAG_OFF(x) /* ignored */
#define RAPIDJSON_DIAG_PUSH   /* ignored */
// 定义忽略宏RAPIDJSON_DIAG_POP
#define RAPIDJSON_DIAG_POP    /* ignored */

#endif // RAPIDJSON_DIAG_*

///////////////////////////////////////////////////////////////////////////////
// C++11 features

// 如果未定义RAPIDJSON_HAS_CXX11，则根据RAPIDJSON_CPLUSPLUS的值判断是否支持C++11
#ifndef RAPIDJSON_HAS_CXX11
#define RAPIDJSON_HAS_CXX11 (RAPIDJSON_CPLUSPLUS >= 201103L)
#endif

// 如果未定义RAPIDJSON_HAS_CXX11_RVALUE_REFS，则根据条件判断是否支持C++11的右值引用
#ifndef RAPIDJSON_HAS_CXX11_RVALUE_REFS
#if RAPIDJSON_HAS_CXX11
#define RAPIDJSON_HAS_CXX11_RVALUE_REFS 1
#elif defined(__clang__)
#if __has_feature(cxx_rvalue_references) && \
    (defined(_MSC_VER) || defined(_LIBCPP_VERSION) || defined(__GLIBCXX__) && __GLIBCXX__ >= 20080306)
#define RAPIDJSON_HAS_CXX11_RVALUE_REFS 1
#else
#define RAPIDJSON_HAS_CXX11_RVALUE_REFS 0
#endif
#elif (defined(RAPIDJSON_GNUC) && (RAPIDJSON_GNUC >= RAPIDJSON_VERSION_CODE(4,3,0)) && defined(__GXX_EXPERIMENTAL_CXX0X__)) || \
      (defined(_MSC_VER) && _MSC_VER >= 1600) || \
      (defined(__SUNPRO_CC) && __SUNPRO_CC >= 0x5140 && defined(__GXX_EXPERIMENTAL_CXX0X__))

#define RAPIDJSON_HAS_CXX11_RVALUE_REFS 1
#else
#define RAPIDJSON_HAS_CXX11_RVALUE_REFS 0
#endif
#endif // RAPIDJSON_HAS_CXX11_RVALUE_REFS

// 如果支持C++11的右值引用，则包含<utility>头文件
#if RAPIDJSON_HAS_CXX11_RVALUE_REFS
#include <utility> // std::move
#endif

// 如果未定义RAPIDJSON_HAS_CXX11_NOEXCEPT，则根据条件判断是否支持C++11的noexcept
#ifndef RAPIDJSON_HAS_CXX11_NOEXCEPT
#if RAPIDJSON_HAS_CXX11
#define RAPIDJSON_HAS_CXX11_NOEXCEPT 1
#elif defined(__clang__)
#define RAPIDJSON_HAS_CXX11_NOEXCEPT __has_feature(cxx_noexcept)
#elif (defined(RAPIDJSON_GNUC) && (RAPIDJSON_GNUC >= RAPIDJSON_VERSION_CODE(4,6,0)) && defined(__GXX_EXPERIMENTAL_CXX0X__)) || \
    (defined(_MSC_VER) && _MSC_VER >= 1900) || \
    (defined(__SUNPRO_CC) && __SUNPRO_CC >= 0x5140 && defined(__GXX_EXPERIMENTAL_CXX0X__))
#define RAPIDJSON_HAS_CXX11_NOEXCEPT 1
#else
#define RAPIDJSON_HAS_CXX11_NOEXCEPT 0
#endif
#endif

// 如果未定义RAPIDJSON_NOEXCEPT，则根据是否支持C++11的noexcept来定义RAPIDJSON_NOEXCEPT
#ifndef RAPIDJSON_NOEXCEPT
#if RAPIDJSON_HAS_CXX11_NOEXCEPT
#define RAPIDJSON_NOEXCEPT noexcept
#else
#define RAPIDJSON_NOEXCEPT throw()
#endif // RAPIDJSON_HAS_CXX11_NOEXCEPT
#endif

// 尚未自动检测
#ifndef RAPIDJSON_HAS_CXX11_TYPETRAITS
#if (defined(_MSC_VER) && _MSC_VER >= 1700)
#ifndef RAPIDJSON_HAS_CXX11_TYPETRAITS
// 如果未定义 RAPIDJSON_HAS_CXX11_TYPETRAITS，则根据条件定义为 1 或 0
#if defined(__clang__)
#define RAPIDJSON_HAS_CXX11_TYPETRAITS __has_feature(cxx_type_traits)
#else
#define RAPIDJSON_HAS_CXX11_TYPETRAITS 0
#endif
#endif

#ifndef RAPIDJSON_HAS_CXX11_RANGE_FOR
// 如果未定义 RAPIDJSON_HAS_CXX11_RANGE_FOR，则根据条件定义为 1 或 0
#if defined(__clang__)
#define RAPIDJSON_HAS_CXX11_RANGE_FOR __has_feature(cxx_range_for)
#elif (defined(RAPIDJSON_GNUC) && (RAPIDJSON_GNUC >= RAPIDJSON_VERSION_CODE(4,6,0)) && defined(__GXX_EXPERIMENTAL_CXX0X__)) || \
      (defined(_MSC_VER) && _MSC_VER >= 1700) || \
      (defined(__SUNPRO_CC) && __SUNPRO_CC >= 0x5140 && defined(__GXX_EXPERIMENTAL_CXX0X__))
#define RAPIDJSON_HAS_CXX11_RANGE_FOR 1
#else
#define RAPIDJSON_HAS_CXX11_RANGE_FOR 0
#endif
#endif // RAPIDJSON_HAS_CXX11_RANGE_FOR

#ifndef RAPIDJSON_HAS_CXX17
// 如果未定义 RAPIDJSON_HAS_CXX17，则根据条件定义为 1 或 0
#define RAPIDJSON_HAS_CXX17 (RAPIDJSON_CPLUSPLUS >= 201703L)
#endif

#if RAPIDJSON_HAS_CXX17
// 如果定义了 RAPIDJSON_HAS_CXX17，则根据条件定义为不同的值
# define RAPIDJSON_DELIBERATE_FALLTHROUGH [[fallthrough]]
#elif defined(__has_cpp_attribute)
# if __has_cpp_attribute(clang::fallthrough)
#  define RAPIDJSON_DELIBERATE_FALLTHROUGH [[clang::fallthrough]]
# elif __has_cpp_attribute(fallthrough)
#  define RAPIDJSON_DELIBERATE_FALLTHROUGH __attribute__((fallthrough))
# else
#  define RAPIDJSON_DELIBERATE_FALLTHROUGH
# endif
#else
# define RAPIDJSON_DELIBERATE_FALLTHROUGH
#endif

//!@endcond

//! Assertion (in non-throwing contexts).
 /*! \ingroup RAPIDJSON_CONFIG
    Some functions provide a \c noexcept guarantee, if the compiler supports it.
    In these cases, the \ref RAPIDJSON_ASSERT macro cannot be overridden to
    throw an exception.  This macro adds a separate customization point for
    such cases.

    Defaults to C \c assert() (as \ref RAPIDJSON_ASSERT), if \c noexcept is
    supported, and to \ref RAPIDJSON_ASSERT otherwise.
 */

///////////////////////////////////////////////////////////////////////////////
// RAPIDJSON_NOEXCEPT_ASSERT

#ifndef RAPIDJSON_NOEXCEPT_ASSERT
// 如果未定义 RAPIDJSON_NOEXCEPT_ASSERT 且定义了 RAPIDJSON_ASSERT_THROWS，则包含 <cassert>
#ifdef RAPIDJSON_ASSERT_THROWS
#include <cassert>
// 定义宏，如果定义了 RAPIDJSON_NOEXCEPT，则使用 assert(x)，否则使用 RAPIDJSON_ASSERT(x)
#define RAPIDJSON_NOEXCEPT_ASSERT(x) assert(x)
#else
#define RAPIDJSON_NOEXCEPT_ASSERT(x) RAPIDJSON_ASSERT(x)
#endif // RAPIDJSON_ASSERT_THROWS
#endif // RAPIDJSON_NOEXCEPT_ASSERT

///////////////////////////////////////////////////////////////////////////////
// malloc/realloc/free

#ifndef RAPIDJSON_MALLOC
///! 为全局的 \c malloc 定制化点
#define RAPIDJSON_MALLOC(size) std::malloc(size)
#endif
#ifndef RAPIDJSON_REALLOC
///! 为全局的 \c realloc 定制化点
#define RAPIDJSON_REALLOC(ptr, new_size) std::realloc(ptr, new_size)
#endif
#ifndef RAPIDJSON_FREE
///! 为全局的 \c free 定制化点
#define RAPIDJSON_FREE(ptr) std::free(ptr)
#endif

///////////////////////////////////////////////////////////////////////////////
// new/delete

#ifndef RAPIDJSON_NEW
///! 为全局的 \c new 定制化点
#define RAPIDJSON_NEW(TypeName) new TypeName
#endif
#ifndef RAPIDJSON_DELETE
///! 为全局的 \c delete 定制化点
#define RAPIDJSON_DELETE(x) delete x
#endif

///////////////////////////////////////////////////////////////////////////////
// Type

/*! \namespace rapidjson
    \brief RapidJSON 的主要命名空间
    \see RAPIDJSON_NAMESPACE
*/
RAPIDJSON_NAMESPACE_BEGIN

//! JSON 值的类型
enum Type {
    kNullType = 0,      //!< 空值
    kFalseType = 1,     //!< 假
    kTrueType = 2,      //!< 真
    kObjectType = 3,    //!< 对象
    kArrayType = 4,     //!< 数组
    kStringType = 5,    //!< 字符串
    kNumberType = 6     //!< 数字
};

RAPIDJSON_NAMESPACE_END

#endif // RAPIDJSON_RAPIDJSON_H_
```