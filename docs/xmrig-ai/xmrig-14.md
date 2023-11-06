# xmrig源码解析 14

# `src/3rdparty/fmt/core.h`

这段代码定义了一个名为 FMT_CORE_H_ 的头文件，它是 Formatting library for C++ 的核心 API。这个头文件包含了 C++11 中定义的一些用于格式化输入输出的头文件，例如 std::FILE、std::string、std::functional 和 std::iterator 等。

这个头文件的作用是定义了 FMT_CORE_H_，从而使得 FMT_CORE_H_ 可以被其他 C++ 源代码文件包含。同时，这个头文件也包含了 FMT_CORE_H_ 的定义，使得程序在编译时就能够正确地包含它。


```cpp
// Formatting library for C++ - the core API
//
// Copyright (c) 2012 - present, Victor Zverovich
// All rights reserved.
//
// For the license information refer to format.h.

#ifndef FMT_CORE_H_
#define FMT_CORE_H_

#include <cstdio>  // std::FILE
#include <cstring>
#include <functional>
#include <iterator>
#include <memory>
```

这段代码定义了一个名为FMT的函数，它接受一个字符串参数，并输出该字符串格式化后的结果。

首先，它引入了<string>和<type_traits>头文件，这两个头文件分别用于处理字符串和类型traits。

接着，它引入了<vector>头文件，用于在vector容器中进行操作。

在函数体中，它定义了FMT函数，它接受一个字符串参数，并使用格式化字符串来输出该字符串。具体来说，它通过判断运行时是否包含clang编译器来选择使用哪种输出格式。如果包含clang编译器，它将输出一个以FMT_CLANG_VERSION为前缀的格式字符串，否则输出一个以FMT_GCC_VERSION为前缀的格式字符串。如果既包含clang又包含GNU编译器，它将输出一个以FMT_GCC_VERSION为前缀的格式字符串，否则输出一个以FMT_CLANG_VERSION为前缀的格式字符串。


```cpp
#include <string>
#include <type_traits>
#include <vector>

// The fmt library version in the form major * 10000 + minor * 100 + patch.
#define FMT_VERSION 70003

#ifdef __clang__
#  define FMT_CLANG_VERSION (__clang_major__ * 100 + __clang_minor__)
#else
#  define FMT_CLANG_VERSION 0
#endif

#if defined(__GNUC__) && !defined(__clang__)
#  define FMT_GCC_VERSION (__GNUC__ * 100 + __GNUC_MINOR__)
```

这段代码定义了一系列预处理指令，用于定义某些标志或检查版本信息。

首先，定义了一个名为 FMT_GCC_VERSION 的宏，其值为 0。这个宏可能用于某些编译器，其作用是告诉编译器应该使用 GCC 还是 Clang。

接着，定义了一个名为 FMT_ICC_VERSION 的宏，其值为 0。这个宏可能用于某些编译器，其作用是告诉编译器应该使用 Intel 编译器还是 GCC。

然后，定义了一系列条件语句，用于检查 GCC 和 Intel 编译器的支持情况。如果 __INTEL_COMPILER 为此值，则定义了 FMT_ICC_VERSION，否则定义了 FMT_ICC_VERSION，无论哪个值，都使用了 0。这表明 FMT_ICC_VERSION 的值与 GCC 版本号无关。

最后，定义了一个名为 FMT_HAS_GXX_CXX11 的宏，其值为 FMT_GCC_VERSION。这个宏可能用于某些编译器，其作用是检查该编译器是否支持 GXX 和 C++11。如果 __CPLUSPLUS__ >= 201103L 或者 __GXX_EXPERIMENTAL_CXX0X__，则定义了 FMT_HAS_GXX_CXX11，否则定义了 FMT_HAS_GXX_CXX11，无论哪个值，都使用了 0。


```cpp
#else
#  define FMT_GCC_VERSION 0
#endif

#if defined(__INTEL_COMPILER)
#  define FMT_ICC_VERSION __INTEL_COMPILER
#else
#  define FMT_ICC_VERSION 0
#endif

#if __cplusplus >= 201103L || defined(__GXX_EXPERIMENTAL_CXX0X__)
#  define FMT_HAS_GXX_CXX11 FMT_GCC_VERSION
#else
#  define FMT_HAS_GXX_CXX11 0
#endif

```

这段代码定义了一些宏，用于控制编译器的输出。

首先，通过#ifdef __NVCC__,# else,#endif 实现了一个条件编译语句，判断(__NVCC__)是否已经被定义过。如果已经被定义过，则执行# define FMT_NVCC __NVCC__，否则执行# else,#endif。

接着，通过#ifdef _MSC_VER 和 #ifdef __has_feature，实现了一个判断，如果当前系统支持多重编译模式并且定义了_MSC_VER，则执行# define FMT_MSC_VER _MSC_VER，否则执行# define FMT_MSC_VER 0。

最后，通过# define FMT_SUPPRESS_MSC_WARNING(n) __pragma(warning(suppress : n)) 和 #define FMT_SUPPRESS_MSC_WARNING(n) 0 实现了一个判断，如果当前系统不支持多重编译模式，则执行# define FMT_SUPPRESS_MSC_WARNING(n) 0。


```cpp
#ifdef __NVCC__
#  define FMT_NVCC __NVCC__
#else
#  define FMT_NVCC 0
#endif

#ifdef _MSC_VER
#  define FMT_MSC_VER _MSC_VER
#  define FMT_SUPPRESS_MSC_WARNING(n) __pragma(warning(suppress : n))
#else
#  define FMT_MSC_VER 0
#  define FMT_SUPPRESS_MSC_WARNING(n)
#endif

#ifdef __has_feature
```

这段代码定义了一系列条件语句，用于检查特定的编译器是否支持特定的C或C++特性。

首先，定义了一个名为FMT_HAS_FEATURE的宏，它使用了一个名为__has_feature的计算机类型。这个宏根据给定的参数x，检查其是否具有该特定功能。如果该功能在计算机上可用，则返回1，否则返回0。

然后，定义了一个名为FMT_HAS_INCLUDE的宏，它使用了一个名为__has_include的计算机类型。这个宏根据给定的选项x，检查其是否具有包含该特定头文件的路径。如果该头文件在计算机上可用，则返回1，否则返回0。

接着，定义了一系列条件语句，用于检查特定功能是否支持C或C++特性。这些条件语句检查给定的编译器是否支持FMT_ICC_VERSION和FMT_CMAKE_VERSION，并返回1或0。

最后，定义了一个名为FMT_HAS_CPP_ATTRIBUTE的宏，它使用了一个名为__has_cpp_attribute的计算机类型。这个宏根据给定的选项x，检查其是否具有C++特性。如果该特性在计算机上可用，则返回1，否则返回0。


```cpp
#  define FMT_HAS_FEATURE(x) __has_feature(x)
#else
#  define FMT_HAS_FEATURE(x) 0
#endif

#if defined(__has_include) && !defined(__INTELLISENSE__) && \
    (!FMT_ICC_VERSION || FMT_ICC_VERSION >= 1600)
#  define FMT_HAS_INCLUDE(x) __has_include(x)
#else
#  define FMT_HAS_INCLUDE(x) 0
#endif

#ifdef __has_cpp_attribute
#  define FMT_HAS_CPP_ATTRIBUTE(x) __has_cpp_attribute(x)
#else
```

这段代码定义了一系列头文件，用于定义C++14和C++17的新特性。

首先定义了两个预处理指令：

- `FMT_HAS_CPP_ATTRIBUTE(x)`：如果C++14中存在该特性，则编译器支持该特性，否则编译器不支持。
- `FMT_HAS_CPP14_ATTRIBUTE(attribute)`：表示该特性的C++14或C++17版本支持。使用(`__cplusplus >= 201402L && FMT_HAS_CPP_ATTRIBUTE(attribute))`来检查。

接着定义了两个宏，用于在编译时检查C++14和C++17的新特性。

- `FMT_HAS_CPP17_ATTRIBUTE(attribute)`：表示该特性的C++17版本支持。使用(`__cplusplus >= 201703L && FMT_HAS_CPP_ATTRIBUTE(attribute))`来检查。
- `FMT_USE_CONSTEXPR`：如果当前编译器不支持C++14的`constexpr`特性，则使用C++17的`constexpr`特性。通过检查`FMT_HAS_FEATURE(cxx_relaxed_constexpr)`是否为真来确定是否支持。

该代码的作用是定义了C++14和C++17的新特性，用于检查当前编译器是否支持新特性。


```cpp
#  define FMT_HAS_CPP_ATTRIBUTE(x) 0
#endif

#define FMT_HAS_CPP14_ATTRIBUTE(attribute) \
  (__cplusplus >= 201402L && FMT_HAS_CPP_ATTRIBUTE(attribute))

#define FMT_HAS_CPP17_ATTRIBUTE(attribute) \
  (__cplusplus >= 201703L && FMT_HAS_CPP_ATTRIBUTE(attribute))

// Check if relaxed C++14 constexpr is supported.
// GCC doesn't allow throw in constexpr until version 6 (bug 67371).
#ifndef FMT_USE_CONSTEXPR
#  define FMT_USE_CONSTEXPR                                           \
    (FMT_HAS_FEATURE(cxx_relaxed_constexpr) || FMT_MSC_VER >= 1910 || \
     (FMT_GCC_VERSION >= 600 && __cplusplus >= 201402L)) &&           \
        !FMT_NVCC && !FMT_ICC_VERSION
```

这段代码定义了一系列头文件 FMT_CONSTEXPR 和 FMT_OVERRIDE，以及一个fmt_use_constexpr_修饰的函数 FMT_CONSTEXPR_DECL。fmt_use_constexpr_修饰函数声明时使用constexpr关键字，而FMT_CONSTEXPR_DECL函数声明时使用decltype关键字。

constexpr是C++11中constexpr的扩展，可以使得constexpr语句中的表达式被const化，而不是仅仅求值。通过constexpr可以更方便地编写可读性更强的表达式，而不需要过多关注变量推导等细节。

fmt_ovride是一个宏定义，用于检查程序是否支持c++_override_control或gxx_cxx11。如果前两者都存在，则定义为override，否则不定义。

这段代码的作用是定义了fmt_constexpr和fmt_ovride这两个头部函数，以及fmt_constrxexpr_decl这个函数，主要作用是帮助程序员更方便地编写出更容易理解的表达式，同时也可以在程序中更灵活地使用constexpr语句。


```cpp
#endif
#if FMT_USE_CONSTEXPR
#  define FMT_CONSTEXPR constexpr
#  define FMT_CONSTEXPR_DECL constexpr
#else
#  define FMT_CONSTEXPR inline
#  define FMT_CONSTEXPR_DECL
#endif

#ifndef FMT_OVERRIDE
#  if FMT_HAS_FEATURE(cxx_override_control) || \
      (FMT_GCC_VERSION >= 408 && FMT_HAS_GXX_CXX11) || FMT_MSC_VER >= 1900
#    define FMT_OVERRIDE override
#  else
#    define FMT_OVERRIDE
```

这段代码检查是否禁止异常处理。如果是，则定义为0；如果不是，则定义为1。这个代码是一个 preprocessor 宏，通常用于在编译时检查特定条件是否满足。


```cpp
#  endif
#endif

// Check if exceptions are disabled.
#ifndef FMT_EXCEPTIONS
#  if (defined(__GNUC__) && !defined(__EXCEPTIONS)) || \
      FMT_MSC_VER && !_HAS_EXCEPTIONS
#    define FMT_EXCEPTIONS 0
#  else
#    define FMT_EXCEPTIONS 1
#  endif
#endif

// Define FMT_USE_NOEXCEPT to make fmt use noexcept (C++11 feature).
#ifndef FMT_USE_NOEXCEPT
```

这段代码定义了一系列条件样式的注释，用于定义 C++11 中异常处理（throwing）的选项。

首先，定义了一组条件，当满足其中任意一个时，定义了一个名为“FMT_DETECTED_NOEXCEPT”的宏，其值为“noexcept”，表示在出现异常情况时不会产生异常信息输出。

接着，定义了另一个名为“FMT_HAS_FEATURE(cxx_noexcept)”的条件，它用于判断是否支持 C++14 或其他支持“-noexcept”选项的 GCC 编译器。如果支持，则定义了一个名为“FMT_NOEXCEPT”的宏，其值为“noexcept”，与前面的宏相同。

然后，定义了第三个条件，即当 FMT_GCC_VERSION >= 408 时，以及 FMT_HAS_GXX_CXX11 时，定义了一个名为“FMT_NOEXCEPT”的宏，其值为“noexcept”。

最后，定义了最后一个条件，即当 FMT_MSC_VER >= 1900 时，定义了一个名为“FMT_NOEXCEPT”的宏，其值为“noexcept”。

总之，这段代码定义了一系列用于定义 C++11 中异常处理选项的注释。


```cpp
#  define FMT_USE_NOEXCEPT 0
#endif

#if FMT_USE_NOEXCEPT || FMT_HAS_FEATURE(cxx_noexcept) || \
    (FMT_GCC_VERSION >= 408 && FMT_HAS_GXX_CXX11) || FMT_MSC_VER >= 1900
#  define FMT_DETECTED_NOEXCEPT noexcept
#  define FMT_HAS_CXX11_NOEXCEPT 1
#else
#  define FMT_DETECTED_NOEXCEPT throw()
#  define FMT_HAS_CXX11_NOEXCEPT 0
#endif

#ifndef FMT_NOEXCEPT
#  if FMT_EXCEPTIONS || FMT_HAS_CXX11_NOEXCEPT
#    define FMT_NOEXCEPT FMT_DETECTED_NOEXCEPT
```

这段代码定义了一个名为 FMT_NORETURN 的宏，用于在特定条件下列出默认的返回类型。

首先，它检查了几个条件，如果这些条件都满足，那么就会定义 FMT_NORETURN，否则就不定义。这些条件如下：

- FMT_EXCEPTIONS：表示这是一个 C++ 应用程序，并且 FMT_HAS_CPP_ATTRIBUTE(noreturn) 等于 1。这个选项可以在 Visual Studio 中关闭，也可以在 NVCC 中禁用。
- FMT_MSC_VER：表示这个 C++ 编译器版本支持 ATTRIBUTE(noreturn) 属性。
- FMT_NVCC：表示这个 C++ 编译器不支持 ATTRIBUTE(noreturn) 属性。

如果这些条件中有一个不满足，那么就不定义 FMT_NORETURN。

如果所有条件都满足，那么就会定义 FMT_NORETURN，否则就不定义。FMT_NORETURN 的定义如下：

```cpp
#  define FMT_NORETURN [[noreturn]]
```

这个宏定义了一个默认的返回类型，如果调用它的函数或变量没有显式地指定返回类型，那么就会以默认的类型返回。这个宏仅仅在 FMT_EXCEPTIONS 和 FMT_HAS_CPP_ATTRIBUTE(noreturn) 这两个条件都满足时才有作用，否则就不被允许调用。


```cpp
#  else
#    define FMT_NOEXCEPT
#  endif
#endif

// [[noreturn]] is disabled on MSVC and NVCC because of bogus unreachable code
// warnings.
#if FMT_EXCEPTIONS && FMT_HAS_CPP_ATTRIBUTE(noreturn) && !FMT_MSC_VER && \
    !FMT_NVCC
#  define FMT_NORETURN [[noreturn]]
#else
#  define FMT_NORETURN
#endif

#ifndef FMT_DEPRECATED
```

这段代码是一个C/C++语言的编译器扩展（compiler flag），用于指示编译器是否支持C++14中的deprecated属性。

具体来说，这段代码会根据以下情况来定义`FMT_DEPRECATED`：

1. 如果FMT_HAS_CPP14_ATTRIBUTE(deprecated)为真，或者FMT_MSC_VER的版本大于等于1900，那么将`[[deprecated]]`添加到`FMT_DEPRECATED`中，这表示编译器应该在使用C++14或更高版本时忽略`deprecated`属性。
2. 如果定义了`__GNUC__`并且没有定义`__LCC__`，或者定义了`__clang__`，则使用`__attribute__((deprecated))`为`FMT_DEPRECATED`。
3. 如果FMT_MSC_VER的版本大于等于1900，则使用`__declspec(deprecated)`为`FMT_DEPRECATED`。
4. 如果定义了`FMT_ICC_VERSION`或`__PGI`或`__NVCC`，则编译器应该能够正确识别`deprecated`属性，无需进行特殊处理。

如果上述任何一种情况都不满足，那么编译器将输出" deprecated"警告。


```cpp
#  if FMT_HAS_CPP14_ATTRIBUTE(deprecated) || FMT_MSC_VER >= 1900
#    define FMT_DEPRECATED [[deprecated]]
#  else
#    if (defined(__GNUC__) && !defined(__LCC__)) || defined(__clang__)
#      define FMT_DEPRECATED __attribute__((deprecated))
#    elif FMT_MSC_VER
#      define FMT_DEPRECATED __declspec(deprecated)
#    else
#      define FMT_DEPRECATED /* deprecated */
#    endif
#  endif
#endif

// Workaround broken [[deprecated]] in the Intel, PGI and NVCC compilers.
#if FMT_ICC_VERSION || defined(__PGI) || FMT_NVCC
```

这段代码定义了一个名为 FMT_DEPRECATED_ALIAS 的模板别名，用于在需要时动态地给形参 FMT_DEPRECATED 赋予一些额外的含义。

具体来说，这段代码分为两部分：

1. 在一个名为 defines.h 的头文件中，定义了一个名为 FMT_DEPRECATED_ALIAS 的模板别名，它的作用是在定义 FMT_DEPRECATED 时，如果 FMT_GCC_VERSION 或者 FMT_CLANG_VERSION 存在，就使用一个带参数的 `__attribute__((always_inline))` 的别名，否则直接使用 `FMT_DEPRECATED`。

2. 在一个名为 ffmt.h 的头文件中，定义了一个名为 FMT_INLINE 的模板别名，它的作用是在需要时动态地给形参 `__FILE__` 和 `__LINE__` 赋予额外的含义。具体来说，如果 FMT_HAS_FEATURE(cxx_inline_namespaces)，或者 FMT_GCC_VERSION 或者 FMT_CLANG_VERSION 大于等于 404，就在头文件的作用域中使用 `inline` 修饰符，否则就不带修饰符。


```cpp
#  define FMT_DEPRECATED_ALIAS
#else
#  define FMT_DEPRECATED_ALIAS FMT_DEPRECATED
#endif

#ifndef FMT_INLINE
#  if FMT_GCC_VERSION || FMT_CLANG_VERSION
#    define FMT_INLINE inline __attribute__((always_inline))
#  else
#    define FMT_INLINE inline
#  endif
#endif

#ifndef FMT_BEGIN_NAMESPACE
#  if FMT_HAS_FEATURE(cxx_inline_namespaces) || FMT_GCC_VERSION >= 404 || \
      (FMT_MSC_VER >= 1900 && !_MANAGED)
```

这段代码定义了一系列命名空间。首先定义了一个名为 FMT_INLINE_NAMESPACE 的内联命名空间，接着定义了一个名为 FMT_END_NAMESPACE 的结束命名空间。然后，如果定义了 FMT_INLINE_NAMESPACE，那么就会用这个命名空间中的内容来定义后面的代码。接着定义了一个名为 FMT_BEGIN_NAMESPACE 的开始命名空间，这个命名空间中包含 FMT_INLINE_NAMESPACE。然后，在 FMT_BEGIN_NAMESPACE 中使用 FMT_INLINE_NAMESPACE 来实现 FMT_INLINE_NAMESPACE 中的定义，然后在 FMT_END_NAMESPACE 中用 v7 来结束命名空间。


```cpp
#    define FMT_INLINE_NAMESPACE inline namespace
#    define FMT_END_NAMESPACE \
      }                       \
      }
#  else
#    define FMT_INLINE_NAMESPACE namespace
#    define FMT_END_NAMESPACE \
      }                       \
      using namespace v7;     \
      }
#  endif
#  define FMT_BEGIN_NAMESPACE \
    namespace fmt {           \
    FMT_INLINE_NAMESPACE v7 {
#endif

```

这段代码是一个C/C++语言的代码，主要作用是定义了FMT_CLASS_API。它通过判断！defined(FMT_HEADER_ONLY)和 defined(_WIN32)两个条件，如果两个条件中有一个为真，则执行以下代码块，否则跳过这个代码块。

代码块内部定义了一个名为FMT_CLASS_API的宏，它的值为FMT_SUPPRESS_MSC_WARNING(4275)，其中4275是随机生成的整数。接着定义了一系列与FMT_CLASS_API相关的宏，包括FMT_API、FMT_EXPORT、FMT_EXPORTED等。最后通过#elif defined(FMT_SHARED)来判断是否为共享库，如果是则通过dllimport来导入FMT_API，如果不是，则使用dllimport来导入FMT_EXPORTED。

整个代码的作用是定义了FMT_CLASS_API这个宏，根据不同的平台和库的定义，来实现对FMT_API的定义，使得程序在定义FMT_CLASS_API时能够正确地使用它。


```cpp
#if !defined(FMT_HEADER_ONLY) && defined(_WIN32)
#  define FMT_CLASS_API FMT_SUPPRESS_MSC_WARNING(4275)
#  ifdef FMT_EXPORT
#    define FMT_API __declspec(dllexport)
#    define FMT_EXTERN_TEMPLATE_API FMT_API
#    define FMT_EXPORTED
#  elif defined(FMT_SHARED)
#    define FMT_API __declspec(dllimport)
#    define FMT_EXTERN_TEMPLATE_API FMT_API
#  endif
#else
#  define FMT_CLASS_API
#endif
#ifndef FMT_API
#  define FMT_API
```

这段代码定义了一系列预处理指令，用于格式化字符串。其作用如下：

1. `#ifdef FMT_EXTERN_TEMPLATE_API` 和 `#ifndef FMT_EXTERN_TEMPLATE_API` 定义了 FMT_EXTERN_TEMPLATE_API 和 FMT_EXTERN, respectively.这两条指令可以理解为：如果当前编译器支持 FMT_EXTERN_TEMPLATE_API，则执行 `#ifdef` 后面的语句，否则执行 `#ifndef` 后面的语句。

2. `#ifndef FMT_INSTANTIATION_DEF_API` 和 `#define FMT_INSTANTIATION_DEF_API FMT_API` 定义了 FMT_INSTANTIATION_DEF_API 和 FMT_INSTANTIATION_DEF, respectively。这两条指令可以理解为：如果当前编译器支持 FMT_INSTANTIATION_DEF_API，则执行 `#define` 后面的语句，否则执行 `#include` 后面的源代码。

3. `#ifndef FMT_HEADER_ONLY` 和 `#define FMT_EXTERN extern` 定义了 FMT_EXTERN。这条指令可以理解为：如果当前编译器支持 FMT_HEADER_ONLY，则执行 `#define` 后面的语句，否则执行 `#include` 后面的源代码。

4. `#include <string_view>` 是一个预处理指令，用于引入 libc++ 中的 string_view 头文件。这条指令可以理解为：如果当前编译器不支持 FMT_HEADER_ONLY，则执行 `#include` 后面的源代码。

综上所述，这段代码定义了一系列预处理指令，用于在编译时格式化字符串。


```cpp
#endif
#ifndef FMT_EXTERN_TEMPLATE_API
#  define FMT_EXTERN_TEMPLATE_API
#endif
#ifndef FMT_INSTANTIATION_DEF_API
#  define FMT_INSTANTIATION_DEF_API FMT_API
#endif

#ifndef FMT_HEADER_ONLY
#  define FMT_EXTERN extern
#else
#  define FMT_EXTERN
#endif

// libc++ supports string_view in pre-c++17.
```

这段代码是一个C语言的预处理指令，用于检查特定条件是否满足，并根据检查结果包含或不包含某个头文件或库。

具体来说，代码会检查以下条件是否满足：

1. 如果FMT_HAS_INCLUDE(<string_view>)并且__cplusplus > 201402L或者defined(_LIBCPP_VERSION))或者defined(_MSVC_LANG)且 _MSC_VER >= 1910，那么将包含<string_view>的头文件或库添加到编译时的链接列表中，即所谓的“链式求值”。

2. 如果FMT_HAS_INCLUDE("experimental/string_view")并且__cplusplus >= 201402L，那么将包含<experimental/string_view>的头文件或库添加到编译时的链接列表中，即所谓的“链式求值”。

3. 如果FMT_HAS_INCLUDE("experimental/string_view")并且__cplusplus >= 201402L，并且FMT_MSC_VER >= 1910，那么将包含<experimental/string_view>的头文件或库添加到编译时的链接列表中，即所谓的“链式求值”。

4. 如果FMT_HAS_INCLUDE("string_view")或者FMT_HAS_INCLUDE("experimental/string_view")，那么根据当前编译器的版本，将包含<string_view>的头文件或库添加到编译时的链接列表中，即所谓的“链式求值”。

5. 如果FMT_HAS_INCLUDE("unicode-ascii")或者FMT_HAS_INCLUDE("utf-8")，那么根据当前编译器的版本，将包含<unicode-ascii>的头文件或库添加到编译时的链接列表中，即所谓的“链式求值”。

6. 如果FMT_UNICODE并且FMT_MSC_VER >= 201402L，那么根据当前编译器的版本，将不包含任何<string_view>、<experimental/string_view>或<unicode-ascii>的头文件或库，即所谓的“链式求值”。


```cpp
#if (FMT_HAS_INCLUDE(<string_view>) &&                       \
     (__cplusplus > 201402L || defined(_LIBCPP_VERSION))) || \
    (defined(_MSVC_LANG) && _MSVC_LANG > 201402L && _MSC_VER >= 1910)
#  include <string_view>
#  define FMT_USE_STRING_VIEW
#elif FMT_HAS_INCLUDE("experimental/string_view") && __cplusplus >= 201402L
#  include <experimental/string_view>
#  define FMT_USE_EXPERIMENTAL_STRING_VIEW
#endif

#ifndef FMT_UNICODE
#  define FMT_UNICODE !FMT_MSC_VER
#endif
#if FMT_UNICODE && FMT_MSC_VER
#  pragma execution_character_set("utf-8")
```

这段代码定义了一系列模板别忘了（metafunctions）和条件编译器函数，主要作用是帮助开发者更方便地实现某些功能。

具体来说，这段代码定义了以下几个模板：

1. `enable_if_t`：如果给定的`B`为真，则返回`T`类型的模板。否则返回`nullptr`类型。
2. `conditional_t`：如果给定的`B`为真，则返回`F`类型的模板。否则返回`nullptr`类型。
3. `bool_constant`：是一个快速获取真假值的函数，它接受一个布尔类型的参数`B`，并返回相应的真值或假值。
4. `remove_reference_t`：从`std::remove_reference`函数中获取输入类型的指针类型。
5. `remove_const_t`：从`std::remove_const`函数中获取输入类型的常量类型。
6. `bool`：是一个模板，用于在编译时判断给定的`B`是否为真。


```cpp
#endif

FMT_BEGIN_NAMESPACE

// Implementations of enable_if_t and other metafunctions for older systems.
template <bool B, class T = void>
using enable_if_t = typename std::enable_if<B, T>::type;
template <bool B, class T, class F>
using conditional_t = typename std::conditional<B, T, F>::type;
template <bool B> using bool_constant = std::integral_constant<bool, B>;
template <typename T>
using remove_reference_t = typename std::remove_reference<T>::type;
template <typename T>
using remove_const_t = typename std::remove_const<T>::type;
template <typename T>
```

这段代码定义了一个模板结构体 `type_identity` 和一个模板别裁体 `monostate`。模板结构体 `type_identity` 的实现方式是通过使用 `remove_cvref_t` 来实现的，它返回了 `remove_reference_t<T>` 的类型。模板别裁体 `monostate` 的实现方式与 `type_identity` 类似，但多了一个 `constexpr` 修饰。

`remove_cvref_t` 和 `remove_reference_t` 是 C++11 中提供的模板别裁，它们的作用是帮助我们在模板参数中去除 `const` 和 `volatile` 属性。`remove_cvref_t` 主要用于移除 `const` 属性的影响，而 `remove_reference_t` 主要用于移除 `volatile` 属性的影响。

这段代码的作用是定义了一个模板结构体 `type_identity` 和一个模板别裁体 `monostate`，用于实现深度优先搜索（DFS）的返回类型。`type_identity` 是用于DFS返回类型的模板结构体，它包含一个模板别裁体 `monostate`，用于实现DFS的返回类型。`monostate` 在模板别裁体 `type_identity` 的实现中，通过使用 `constexpr` 修饰来抑制警告 `conditional expression is constant`。


```cpp
using remove_cvref_t = typename std::remove_cv<remove_reference_t<T>>::type;
template <typename T> struct type_identity { using type = T; };
template <typename T> using type_identity_t = typename type_identity<T>::type;

struct monostate {};

// An enable_if helper to be used in template parameters which results in much
// shorter symbols: https://godbolt.org/z/sWw4vP. Extra parentheses are needed
// to workaround a bug in MSVC 2019 (see #1140 and #1186).
#define FMT_ENABLE_IF(...) enable_if_t<(__VA_ARGS__), int> = 0

namespace detail {

// A helper function to suppress "conditional expression is constant" warnings.
template <typename T> constexpr T const_check(T value) { return value; }

```

这段代码是一个C语言中的函数，名为FMT_ASSERT，其作用是在程序运行时检查输入参数的合法性。 FMT_ASSERT函数接收三个参数：一个文件名（FMT_NORETURN），一个行号（int）和一个错误消息（const char*）。

这段代码的作用是：

1. 如果输入参数的文件名为空，并且行号小于0，则输出"FMT_ASSERT has been removed due to -Werror=empty-body"，此时函数将返回0。
2. 如果输入参数的条件为真，则输出一条相应的错误消息，并将返回值设为真。

FMT_ASSERT函数的实现基于两个条件判断：

1. 如果输入参数的文件名为空，则通过构造的assert_fail函数输出提示信息并返回0。
2. 如果输入参数的行号小于0，则同样通过构造的assert_fail函数输出提示信息并返回0。

通过这种自定义的错误处理逻辑，FMT_ASSERT函数可以在需要的时候提供额外的错误报告信息，使得程序在遇到困难时能够更加轻松地定位问题。


```cpp
FMT_NORETURN FMT_API void assert_fail(const char* file, int line,
                                      const char* message);

#ifndef FMT_ASSERT
#  ifdef NDEBUG
// FMT_ASSERT is not empty to avoid -Werror=empty-body.
#    define FMT_ASSERT(condition, message) ((void)0)
#  else
#    define FMT_ASSERT(condition, message)                                    \
      ((condition) /* void() fails with -Winvalid-constexpr on clang 4.0.1 */ \
           ? (void)0                                                          \
           : ::fmt::detail::assert_fail(__FILE__, __LINE__, (message)))
#  endif
#endif

```

这段代码定义了一系列模板别论，用于在不同的编译器输入中提供字符串view的实现。下面是每个条件的解释：

1. #if defined(FMT_USE_STRING_VIEW)

这个条件为真时，会定义一个名为std_string_view的模板类，它是一个基于Char类型的标准库字符串view的实现。

2. #elif defined(FMT_USE_EXPERIMENTAL_STRING_VIEW)

这个条件为真时，会定义一个名为std_string_view的模板类，它是一个基于Char类型的实验性字符串view的实现。这个实现使用了C++20中的[[experimental]]关键字。

3. #else

这个条件为真时，会定义一个名为std_string_view的结构体，它是一个无参类型，用于表示字符串view的实现。这个实现使用了C++17中的[[std::string_view]]的定义。

4. #ifdef FMT_USE_INT128

这个条件为真时，会定义一个名为int128_t的宏，用于表示当FMT_USE_INT128为真时，使用__int128_t作为模板参数的类型。这个实现使用了C++12中的[[int128]]关键字。

5. #elif defined(__SIZEOF_INT128__) && !FMT_NVCC && \
   !(FMT_CLANG_VERSION && FMT_MSC_VER)

这个条件为真时，会定义一个名为int128_t的宏，用于表示当FMT_USE_INT128为真时，使用__int128_t作为模板参数的类型。这个实现使用了C++12中的[[int128]]关键字，并且还使用了__SIZEOF_INT128__这个CGNative哉川特性。此外，这个条件还会检查FMT_NVCC和FMT_CLANG_VERSION两个编译器是否支持这个宏定义，如果不是，则会执行下面的语句。

6. #define FMT_USE_INT128 1

这个宏定义了当FMT_USE_INT128为真时，使用__int128_t作为模板参数的类型。这个宏使用了#identifier和#卫民广，分别表示宏名称和宏定义的参数。


```cpp
#if defined(FMT_USE_STRING_VIEW)
template <typename Char> using std_string_view = std::basic_string_view<Char>;
#elif defined(FMT_USE_EXPERIMENTAL_STRING_VIEW)
template <typename Char>
using std_string_view = std::experimental::basic_string_view<Char>;
#else
template <typename T> struct std_string_view {};
#endif

#ifdef FMT_USE_INT128
// Do nothing.
#elif defined(__SIZEOF_INT128__) && !FMT_NVCC && \
    !(FMT_CLANG_VERSION && FMT_MSC_VER)
#  define FMT_USE_INT128 1
using int128_t = __int128_t;
```

这段代码定义了一个名为 uint128_t 的变量，并检查了是否定义了 FMT_USE_INT128 宏。如果没有定义该宏，则将 uint128_t 定义为 struct { int128_t {}; struct uint128_t {}; } 类型。如果定义了该宏，则将 uint128_t 定义为 struct { uint128_t {}; struct uint128_t {}; } 类型。然后，定义了一个模板类 <typename Int>，其中包含一个名为 to_unsigned 的函数，该函数将一个非负整数映射为无符号整数。最后，定义了一个名为 FMT_CONSTEXPR 的宏，用于在编译时检查是否定义了 to_unsigned 函数。


```cpp
using uint128_t = __uint128_t;
#else
#  define FMT_USE_INT128 0
#endif
#if !FMT_USE_INT128
struct int128_t {};
struct uint128_t {};
#endif

// Casts a nonnegative integer to unsigned.
template <typename Int>
FMT_CONSTEXPR typename std::make_unsigned<Int>::type to_unsigned(Int value) {
  FMT_ASSERT(value >= 0, "negative value");
  return static_cast<typename std::make_unsigned<Int>::type>(value);
}

```

这段代码定义了一个名为 `is_unicode` 的模板类，该模板类用于检查一个字符是否为 Unicode 字符。

首先，代码定义了一个名为 `micro` 的数组，其元素为 `"\u00B5"`。

接着，代码使用模板元编程技术，通过 `constexpr` 关键字定义了一个名为 `is_unicode` 的模板类，该模板类有两个真体条件：

1. 如果 `FMT_UNICODE` 为真，或者 `sizeof(Char)` 等于 1，那么 `is_unicode` 返回 true。
2. 如果 `sizeof(micro)` 等于 3，且 `micro` 元素的第一个元素为 0xC2，第二个元素为 0xB5，那么 `is_unicode` 返回 true。

最后，代码在 `#ifdef __cpp_char8_t` 这里指定了 `using char8_type = char8_t`，表示如果定义了这个头文件，就可以使用 `char8_t` 类型。


```cpp
FMT_SUPPRESS_MSC_WARNING(4566) constexpr unsigned char micro[] = "\u00B5";

template <typename Char> constexpr bool is_unicode() {
  return FMT_UNICODE || sizeof(Char) != 1 ||
         (sizeof(micro) == 3 && micro[0] == 0xC2 && micro[1] == 0xB5);
}

#ifdef __cpp_char8_t
using char8_type = char8_t;
#else
enum char8_type : unsigned char {};
#endif
}  // namespace detail

#ifdef FMT_USE_INTERNAL
```

这段代码是一个模板类，名为“basic_string_view”，它是一个实现“std::basic_string_view”的模板类，可以用于在支持C++17及更高版本的编译器中使用。该模板类提供了一个对字符串的子集API，通过这个API可以方便地格式化字符串，其中包括“std::string_view”的许多功能。

具体来说，这段代码在以下几个方面进行了实现：

1. 模板类的基本成员函数和成员变量：

  ```cpp
  constexpr basic_string_view() FMT_NOEXCEPT : data_(nullptr), size_(0) {}
  ```

  ```cpp
  constexpr basic_string_view(const Char* s, size_t count) FMT_NOEXCEPT
     : data_(s),
       size_(count) {}
  ```

  这里，`basic_string_view` 的构造函数接收两个参数：一个 C 字符串 `s` 和一个整数 `count`，然后根据 `count` 来计算出字符串的长度，将结果存储在 `size_` 成员变量中，并将 `data_` 存储为字符串的起始地址，通过构造函数可以调用该构造函数两次。

2. 模板类中的 constexpr 注解：

  ```cpp
  using value_type = Char;
  using iterator = const Char*;
  ```

  这里，`basic_string_view` 的值类型为 `Char`，`value_type` 的类型也为 `Char`，而 `iterator` 的类型为 `const Char*`。

3. 模板类中的函数：

  ```cpp
  void append(const Char* str, size_t count);
  Char* begin();
  const Char* end();
  ```

  ```cpp
  void clear();
  size_t size() const;
  ```

  这里，`basic_string_view` 中的三个函数分别实现了字符串的 append、begin 和 end 操作，以及字符串的大小获取函数。

4. 模板类中的几个成员函数和 member 变量：

  ```cpp
  friend inline basic_string_view<const Char*> operator+(const Char* lhs, const Char* rhs);
  friend basic_string_view<const Char*> operator|(const Char* lhs, const Char* rhs);
  friend basic_string_view<const Char*> operator==(const Char* lhs, const Char* rhs);
  friend basic_string_view<const Char*>& operator+=(const Char* lhs, const Char* rhs);
  friend basic_string_view<const Char*> operator|=(const Char* lhs, const Char* rhs);
  friend basic_string_view<const Char*> operator==(const Char* lhs, const Char* rhs);
  friend void destroy_impl(basic_string_view<const Char*>* v);
  ```

  这里，`basic_string_view` 的几个朋友函数实现了字符串的 `+`、`==`、`<` 运算符重载，以及 `destroy_impl` 函数，该函数实现了字符串视图的析构。


```cpp
namespace internal = detail;  // DEPRECATED
#endif

/**
  An implementation of ``std::basic_string_view`` for pre-C++17. It provides a
  subset of the API. ``fmt::basic_string_view`` is used for format strings even
  if ``std::string_view`` is available to prevent issues when a library is
  compiled with a different ``-std`` option than the client code (which is not
  recommended).
 */
template <typename Char> class basic_string_view {
 private:
  const Char* data_;
  size_t size_;

 public:
  using value_type = Char;
  using iterator = const Char*;

  constexpr basic_string_view() FMT_NOEXCEPT : data_(nullptr), size_(0) {}

  /** Constructs a string reference object from a C string and a size. */
  constexpr basic_string_view(const Char* s, size_t count) FMT_NOEXCEPT
      : data_(s),
        size_(count) {}

  /**
    \rst
    Constructs a string reference object from a C string computing
    the size with ``std::char_traits<Char>::length``.
    \endrst
   */
```

This is a C++ implementation of the `std::string` class that includes a `size()` member function to return the length of the string, as well as several overloads for comparing and iterating on the elements of the string.

The `std::string` class is defined in the `std::string` header file, which is included in the standard library.

The `size_t` type is used to represent the size of the string. This is because the size of the string is often used as a comparison function, and `std::size_t` is a more convenient large-integer type for that purpose.

The `basic_string_view` template class is used to represent a view of the string, which allows basic manipulation of the string's elements. This class is a member of the `std::vector` template class, which is also a part of the standard library.

The `std::vector` template class is used to represent a container for storing and manipulating elements of type `T`. The `basic_string_view` template class is a member of this template class, and is used to represent a view of the container.

The `basic_string_view` template class is defined in the `std::string_view` header file, which is included in the standard library.

The `basic_string_view` class includes several overloads for the `<` operator, which compare two strings. These overloads are used to define the comparison operator`<` for the `basic_string_view` class.

The `std::string` class and the `basic_string_view` template class are intended to be used together to provide a convenient and powerful way to manipulate strings in C++.


```cpp
#if __cplusplus >= 201703L  // C++17's char_traits::length() is constexpr.
  FMT_CONSTEXPR
#endif
  basic_string_view(const Char* s)
      : data_(s), size_(std::char_traits<Char>::length(s)) {}

  /** Constructs a string reference from a ``std::basic_string`` object. */
  template <typename Traits, typename Alloc>
  FMT_CONSTEXPR basic_string_view(
      const std::basic_string<Char, Traits, Alloc>& s) FMT_NOEXCEPT
      : data_(s.data()),
        size_(s.size()) {}

  template <typename S, FMT_ENABLE_IF(std::is_same<
                                      S, detail::std_string_view<Char>>::value)>
  FMT_CONSTEXPR basic_string_view(S s) FMT_NOEXCEPT : data_(s.data()),
                                                      size_(s.size()) {}

  /** Returns a pointer to the string data. */
  constexpr const Char* data() const { return data_; }

  /** Returns the string size. */
  constexpr size_t size() const { return size_; }

  constexpr iterator begin() const { return data_; }
  constexpr iterator end() const { return data_ + size_; }

  constexpr const Char& operator[](size_t pos) const { return data_[pos]; }

  FMT_CONSTEXPR void remove_prefix(size_t n) {
    data_ += n;
    size_ -= n;
  }

  // Lexicographically compare this string reference to other.
  int compare(basic_string_view other) const {
    size_t str_size = size_ < other.size_ ? size_ : other.size_;
    int result = std::char_traits<Char>::compare(data_, other.data_, str_size);
    if (result == 0)
      result = size_ == other.size_ ? 0 : (size_ < other.size_ ? -1 : 1);
    return result;
  }

  friend bool operator==(basic_string_view lhs, basic_string_view rhs) {
    return lhs.compare(rhs) == 0;
  }
  friend bool operator!=(basic_string_view lhs, basic_string_view rhs) {
    return lhs.compare(rhs) != 0;
  }
  friend bool operator<(basic_string_view lhs, basic_string_view rhs) {
    return lhs.compare(rhs) < 0;
  }
  friend bool operator<=(basic_string_view lhs, basic_string_view rhs) {
    return lhs.compare(rhs) <= 0;
  }
  friend bool operator>(basic_string_view lhs, basic_string_view rhs) {
    return lhs.compare(rhs) > 0;
  }
  friend bool operator>=(basic_string_view lhs, basic_string_view rhs) {
    return lhs.compare(rhs) >= 0;
  }
};

```

这段代码定义了一个模板<T>结构体，名为`is_char`，用于检查给定的`T`是否为字符类型。如果`T`是字符类型，则该模板使用`std::true_type`作为类型；如果`T`不是字符类型，则该模板使用`std::false_type`作为类型。

该代码还定义了一个模板函数`to_string_view`，用于将一个`T`类型的字符串view对象转换为字符串。该函数的实现依赖于`fmt`库，因此需要包含`fmt`库才能成功编译和使用。

此外，该代码还定义了一个名为`my_ns`的命名空间，该命名空间中定义了一个名为`to_string_view`的函数，用于将`my_string`类型的字符串对象转换为字符串view对象。


```cpp
using string_view = basic_string_view<char>;
using wstring_view = basic_string_view<wchar_t>;

/** Specifies if ``T`` is a character type. Can be specialized by users. */
template <typename T> struct is_char : std::false_type {};
template <> struct is_char<char> : std::true_type {};
template <> struct is_char<wchar_t> : std::true_type {};
template <> struct is_char<detail::char8_type> : std::true_type {};
template <> struct is_char<char16_t> : std::true_type {};
template <> struct is_char<char32_t> : std::true_type {};

/**
  \rst
  Returns a string view of `s`. In order to add custom string type support to
  {fmt} provide an overload of `to_string_view` for it in the same namespace as
  the type for the argument-dependent lookup to work.

  **Example**::

    namespace my_ns {
    inline string_view to_string_view(const my_string& s) {
      return {s.data(), s.length()};
    }
    }
    std::string message = fmt::format(my_string("The answer is {}"), 42);
  \endrst
 */
```

这段代码定义了三个版本的`to_string_view`函数，分别用于不同的`Char`类型。

第一个版本是`template <typename Char, FMT_ENABLE_IF(is_char<Char>::value)>`，它是一个函数模板。这个版本的`to_string_view`函数接收一个`const Char*`类型的参数`s`，并返回一个`basic_string_view<Char>`类型的结果。它使用了`is_char<Char>::value`这个条件判断，只有在`Char`是`char`或其他`char`的枚举值时，这个条件才为真，函数才会返回一个`basic_string_view<Char>`类型的结果。

第二个版本是`template <typename Char, typename Traits, typename Alloc>`，它也是一个函数模板。这个版本的`to_string_view`函数接收一个`const std::basic_string<Char, Traits, Alloc>&`类型的参数`s`，并返回一个`basic_string_view<Char>`类型的结果。它使用了`std::basic_string<Char, Traits, Alloc>&`这个模板参数，这个模板参数指定了`Char`类型需要使用的`Traits`和`Alloc`类型的成员函数。

第三个版本是`template <typename Char>`，它是一个通用版本的`to_string_view`函数。这个版本的`to_string_view`函数接收一个`basic_string_view<Char>`类型的参数`s`，并返回一个`basic_string_view<Char>`类型的结果。这个版本没有使用任何模板元编程技巧，它直接将`s`作为参数传递给`to_string_view`函数，并将返回结果赋给`s`。


```cpp
template <typename Char, FMT_ENABLE_IF(is_char<Char>::value)>
inline basic_string_view<Char> to_string_view(const Char* s) {
  return s;
}

template <typename Char, typename Traits, typename Alloc>
inline basic_string_view<Char> to_string_view(
    const std::basic_string<Char, Traits, Alloc>& s) {
  return s;
}

template <typename Char>
inline basic_string_view<Char> to_string_view(basic_string_view<Char> s) {
  return s;
}

```

这段代码定义了一个名为“to_string_view”的函数模板，它接受一个名为“Char”的模板参数。

该函数模板实现了一个将给定的“detail::std_string_view<Char>”对象转换为“char_type”类型的函数。

具体来说，函数模板的实现分为两个部分：

1. 如果给定的模板参数“std::is_empty<detail::std_string_view<Char>>::value”为真，那么不执行任何操作并返回给定的“detail::std_string_view<Char>”对象。

2. 如果给定的模板参数“is_compile_string<S>::value”为真，那么实现将“S”对象转换为“char_type”类型的函数。

这里，“std::is_empty<detail::std_string_view<Char>>”是一个判断给定“Char”对象是否为空字符串的函数，如果为空字符串，则返回给定“detail::std_string_view<Char>”对象。

另外，“is_compile_string<S>”是一个判断给定“S”对象是否为编译字符串的函数，如果为编译字符串，则调用函数“to_string_view”将“S”对象转换为“char_type”类型的字符串。


```cpp
template <typename Char,
          FMT_ENABLE_IF(!std::is_empty<detail::std_string_view<Char>>::value)>
inline basic_string_view<Char> to_string_view(detail::std_string_view<Char> s) {
  return s;
}

// A base class for compile-time strings. It is defined in the fmt namespace to
// make formatting functions visible via ADL, e.g. format(FMT_STRING("{}"), 42).
struct compile_string {};

template <typename S>
struct is_compile_string : std::is_base_of<compile_string, S> {};

template <typename S, FMT_ENABLE_IF(is_compile_string<S>::value)>
constexpr basic_string_view<typename S::char_type> to_string_view(const S& s) {
  return s;
}

```

这段代码定义了一个模板结构体 `char_t_impl`，模板的第二个参数 `enable_if_t<is_string<S>::value>` 表示如果 `S` 是一个字符串类型，则可以使用 `is_string` 模板结构体中定义的 `to_string_view` 函数将其转换为 `fmt::basic_string_view` 类型。如果 `S` 不是字符串类型，则 `is_string` 模板结构体中不会定义，编译时会报错。

`to_string_view` 是一个从 `std::decltype(to_string_view(std::declval<S>()))` 中获取出 `S` 的类型并调用 `to_string_view` 函数得到的类型，它是一个字符串类型转换为 `fmt::basic_string_view` 的函数。

整个模板结构体的定义了一个名为 `char_t_impl` 的模板，它有一个参数 `S`，可以通过 `is_string<S>` 的定义来判断 `S` 是否为字符串类型，如果 `S` 是字符串类型，则 `char_t_impl` 可以安全地使用 `is_string<S>::value` 来获取 `S` 的类型，否则会抛出 `std::invalid_argument`。

对于 `is_string` 模板结构体，它的定义中包含了一个名为 `char_t_impl<S, enable_if_t<is_string<S>::value>>` 的子模板，它的实现与 `is_string<S>` 完全相同，但是通过 `enable_if_t<is_string<S>::value>` 来声明了一个可以被实例化的别名模板。


```cpp
namespace detail {
void to_string_view(...);
using fmt::v7::to_string_view;

// Specifies whether S is a string type convertible to fmt::basic_string_view.
// It should be a constexpr function but MSVC 2017 fails to compile it in
// enable_if and MSVC 2015 fails to compile it as an alias template.
template <typename S>
struct is_string : std::is_class<decltype(to_string_view(std::declval<S>()))> {
};

template <typename S, typename = void> struct char_t_impl {};
template <typename S> struct char_t_impl<S, enable_if_t<is_string<S>::value>> {
  using result = decltype(to_string_view(std::declval<S>()));
  using type = typename result::value_type;
};

```

这段代码定义了一个模板类名为“error_handler”，旨在报告在编译时错误。错误处理器的两个实例化函数分别采用了保留的模板实参类型参数“FMT_ENABLE_IF!”。

这两个实例化函数都采用“is_compile_string<S>”为判断依据，如果S不是有效的格式字符串，则会执行这两个函数中的一个。但是如果S是有效的格式字符串，则不会执行任何函数，从而不会产生任何错误报告。

这两个函数都使用了“FMT_NORETURN”宏来声明故意不带返回类型的函数，以确保在模板复杂度限制下，它们的行为是已定义的。

该代码片段定义了一个模板类“error_handler”，旨在报告编译时错误。通过检查输入的格式字符串是否有效，可以在编译时检测到错误并报告。


```cpp
// Reports a compile-time error if S is not a valid format string.
template <typename..., typename S, FMT_ENABLE_IF(!is_compile_string<S>::value)>
FMT_INLINE void check_format_string(const S&) {
#ifdef FMT_ENFORCE_COMPILE_STRING
  static_assert(is_compile_string<S>::value,
                "FMT_ENFORCE_COMPILE_STRING requires all format strings to use "
                "FMT_STRING.");
#endif
}
template <typename..., typename S, FMT_ENABLE_IF(is_compile_string<S>::value)>
void check_format_string(S);

struct error_handler {
  constexpr error_handler() = default;
  constexpr error_handler(const error_handler&) = default;

  // This function is intentionally not constexpr to give a compile-time error.
  FMT_NORETURN FMT_API void on_error(const char* message);
};
}  // namespace detail

```

这段代码定义了一个模板类`char_t`，用于表示字符类型。这个模板类使用`detail::char_t_impl`作为其内部类型参数。这个内部类型定义了一个字符类型，但没有具体的实现。

接下来的两行定义了一个`ParsingContext`类，包含了格式字符串范围和自动索引参数。这两行允许我们使用不同的字符类型（如`char`和`wchar_t`）来表示格式字符串中的字符。

总体来说，这段代码定义了一个用于解析格式字符串的`ParsingContext`类，以及一个字符类型模板类`char_t`，它用于表示格式字符串中的字符。


```cpp
/** String's character type. */
template <typename S> using char_t = typename detail::char_t_impl<S>::type;

/**
  \rst
  Parsing context consisting of a format string range being parsed and an
  argument counter for automatic indexing.

  You can use one of the following type aliases for common character types:

  +-----------------------+-------------------------------------+
  | Type                  | Definition                          |
  +=======================+=====================================+
  | format_parse_context  | basic_format_parse_context<char>    |
  +-----------------------+-------------------------------------+
  | wformat_parse_context | basic_format_parse_context<wchar_t> |
  +-----------------------+-------------------------------------+
  \endrst
 */
```

This is a implementation of an `ErrorHandler` class for a class that解析es a formatted string using manual or automatic argument indexing. The implementation provides support for a range of helper methods, such as advancing the `begin` and `end` iterators, parsing the string, and reporting errors.

The `ErrorHandler` class has a `next_arg_id_` member variable, which is an iterator that points to the next argument index, and a `format_str_` member variable, which is a template for the formatted string. The class also has a `next_arg_id_set_` member function, which sets the `next_arg_id_` member variable to `-1`, indicating that the current implementation does not support automatic argument indexing.

The `ErrorHandler` class has a `on_error` member function, which is a slot for登记给定的错误处理函数。当给定的错误报告发生时，该函数将被调用。

此外， `ErrorHandler` 的 `format_str_` 模板在使用时需要提供 `FMT_NOEXCEPT` 参数，即模板参数为真时传递默认参数。


```cpp
template <typename Char, typename ErrorHandler = detail::error_handler>
class basic_format_parse_context : private ErrorHandler {
 private:
  basic_string_view<Char> format_str_;
  int next_arg_id_;

 public:
  using char_type = Char;
  using iterator = typename basic_string_view<Char>::iterator;

  explicit constexpr basic_format_parse_context(
      basic_string_view<Char> format_str, ErrorHandler eh = {},
      int next_arg_id = 0)
      : ErrorHandler(eh), format_str_(format_str), next_arg_id_(next_arg_id) {}

  /**
    Returns an iterator to the beginning of the format string range being
    parsed.
   */
  constexpr iterator begin() const FMT_NOEXCEPT { return format_str_.begin(); }

  /**
    Returns an iterator past the end of the format string range being parsed.
   */
  constexpr iterator end() const FMT_NOEXCEPT { return format_str_.end(); }

  /** Advances the begin iterator to ``it``. */
  FMT_CONSTEXPR void advance_to(iterator it) {
    format_str_.remove_prefix(detail::to_unsigned(it - begin()));
  }

  /**
    Reports an error if using the manual argument indexing; otherwise returns
    the next argument index and switches to the automatic indexing.
   */
  FMT_CONSTEXPR int next_arg_id() {
    // Don't check if the argument id is valid to avoid overhead and because it
    // will be checked during formatting anyway.
    if (next_arg_id_ >= 0) return next_arg_id_++;
    on_error("cannot switch from manual to automatic argument indexing");
    return 0;
  }

  /**
    Reports an error if using the automatic argument indexing; otherwise
    switches to the manual indexing.
   */
  FMT_CONSTEXPR void check_arg_id(int) {
    if (next_arg_id_ > 0)
      on_error("cannot switch from automatic to manual argument indexing");
    else
      next_arg_id_ = -1;
  }

  FMT_CONSTEXPR void check_arg_id(basic_string_view<Char>) {}

  FMT_CONSTEXPR void on_error(const char* message) {
    ErrorHandler::on_error(message);
  }

  constexpr ErrorHandler error_handler() const { return *this; }
};

```

这段代码定义了一个名为 basic_format_arg 的模板类，它是一个模板元编程中的参数模板类。这个模板类有两个实现，一个是基本格式解析器（basic_format_parse_context）的类型为 char 的实现，另一个是基本格式解析器（basic_format_parse_context）的类型为 wchar_t 的实现。这两个实现都继承自同一个名为 basic_format_parse_context 的类，该类中包含了一些通用的方法，如格式化输入、输出等。

同时，该代码还定义了一个名为 basic_format_arg 的模板类，该模板类也被分为两个实现，一个是基本格式解析器（basic_format_parse_context）的类型为 char 的实现，另一个是基本格式解析器（basic_format_parse_context）的类型为 wchar_t 的实现。这两个实现都继承自同一个名为 basic_format_parse_context 的类，该类中包含了一些通用的方法，如格式化输入、输出等。

另外，该代码还定义了一个名为 dynamic_format_arg_store 的模板类。这个模板类也被分为两个实现，一个是基本格式解析器（basic_format_parse_context）的类型为 char 的实现，另一个是基本格式解析器（basic_format_parse_context）的类型为 wchar_t 的实现。这两个实现都继承自同一个名为 basic_format_parse_context 的类，该类中包含了一些通用的方法，如格式化输入、输出等。同时，这两个实现还继承自同一个名为 dynamic_format 的类，该类中包含了一些通用的方法，如格式化输出、解析等。

最后，该代码还定义了一个名为 formatter 的模板类。这个模板类也被分为两个实现，一个是基本格式解析器（basic_format_parse_context）的类型为 char 的实现，另一个是基本格式解析器（basic_format_parse_context）的类型为 wchar_t 的实现。这个模板类的实现继承自同一个名为 formatter 的类，该类中包含了一些通用的方法，如格式化输入、输出等。


```cpp
using format_parse_context = basic_format_parse_context<char>;
using wformat_parse_context = basic_format_parse_context<wchar_t>;

template <typename Context> class basic_format_arg;
template <typename Context> class basic_format_args;
template <typename Context> class dynamic_format_arg_store;

// A formatter for objects of type T.
template <typename T, typename Char = char, typename Enable = void>
struct formatter {
  // A deleted default constructor indicates a disabled formatter.
  formatter() = delete;
};

// Specifies if T has an enabled formatter specialization. A type can be
```

这段代码定义了一个通用的模板，名为`has_formatter`，用于检查给定的模板类型`T`是否具有形如`Context::template formatter_type`的格式化器。即使`T`本身不是容器，也可以通过模板元编程将其转换为容器类型，并检查其是否具有所需的格式化器。

具体来说，这段代码实现了一个checksum工具函数，用于检查给定的`T`是否为容器类型。如果`T`不是容器类型，则函数将返回`std::false_type`；否则，将返回`std::true_type`。这是通过`is_contiguous`模板结构体来实现的，该结构体判断给定的`T`是否具有连续的存储空间。

另外，`detail`命名空间中的`get_container`函数是一个模板类，用于从`std::back_insert_iterator`中获取容器引用。它接受一个`Container`模板类型的参数，并在`bi_iterator`类型的指针`it`上获取容器。通过使用该函数，可以方便地获取容器中的元素。


```cpp
// formattable even if it doesn't have a formatter e.g. via a conversion.
template <typename T, typename Context>
using has_formatter =
    std::is_constructible<typename Context::template formatter_type<T>>;

// Checks whether T is a container with contiguous storage.
template <typename T> struct is_contiguous : std::false_type {};
template <typename Char>
struct is_contiguous<std::basic_string<Char>> : std::true_type {};

namespace detail {

// Extracts a reference to the container from back_insert_iterator.
template <typename Container>
inline Container& get_container(std::back_insert_iterator<Container> it) {
  using bi_iterator = std::back_insert_iterator<Container>;
  struct accessor : bi_iterator {
    accessor(bi_iterator iter) : bi_iterator(iter) {}
    using bi_iterator::container;
  };
  return *accessor(it).container;
}

```

This is a C++ class template that defines a buffer class with lifetime coverage for its resources. The buffer class has a size, capacity, and pointer to the buffer data, and provides methods for resizing, adding elements to the end, and accessing the data.

The class is defined as a const-example, which means that the function pointer to the buffer's beginning is passed to the function for storing a pointer to the buffer's data. The buffer class has a template parameter of T, which represents the type of the data stored in the buffer, and a template parameter of I, which represents the index of an element in the buffer.

The class template parameter I is a combination of the index type and the type of the element being accessed. This allows the class to handle indexing of different types of elements.

The class template also includes a try_resize() function, which tries to resize the buffer to a buffer of a larger capacity, while it can only increase the capacity by a smaller amount than requested. If the buffer is already at its maximum capacity, the buffer will be cleared, and the buffer's data will be lost.


```cpp
/**
  \rst
  A contiguous memory buffer with an optional growing ability. It is an internal
  class and shouldn't be used directly, only via `~fmt::basic_memory_buffer`.
  \endrst
 */
template <typename T> class buffer {
 private:
  T* ptr_;
  size_t size_;
  size_t capacity_;

 protected:
  // Don't initialize ptr_ since it is not accessed to save a few cycles.
  FMT_SUPPRESS_MSC_WARNING(26495)
  buffer(size_t sz) FMT_NOEXCEPT : size_(sz), capacity_(sz) {}

  buffer(T* p = nullptr, size_t sz = 0, size_t cap = 0) FMT_NOEXCEPT
      : ptr_(p),
        size_(sz),
        capacity_(cap) {}

  ~buffer() = default;

  /** Sets the buffer data and capacity. */
  void set(T* buf_data, size_t buf_capacity) FMT_NOEXCEPT {
    ptr_ = buf_data;
    capacity_ = buf_capacity;
  }

  /** Increases the buffer capacity to hold at least *capacity* elements. */
  virtual void grow(size_t capacity) = 0;

 public:
  using value_type = T;
  using const_reference = const T&;

  buffer(const buffer&) = delete;
  void operator=(const buffer&) = delete;

  T* begin() FMT_NOEXCEPT { return ptr_; }
  T* end() FMT_NOEXCEPT { return ptr_ + size_; }

  const T* begin() const FMT_NOEXCEPT { return ptr_; }
  const T* end() const FMT_NOEXCEPT { return ptr_ + size_; }

  /** Returns the size of this buffer. */
  size_t size() const FMT_NOEXCEPT { return size_; }

  /** Returns the capacity of this buffer. */
  size_t capacity() const FMT_NOEXCEPT { return capacity_; }

  /** Returns a pointer to the buffer data. */
  T* data() FMT_NOEXCEPT { return ptr_; }

  /** Returns a pointer to the buffer data. */
  const T* data() const FMT_NOEXCEPT { return ptr_; }

  /** Clears this buffer. */
  void clear() { size_ = 0; }

  // Tries resizing the buffer to contain *count* elements. If T is a POD type
  // the new elements may not be initialized.
  void try_resize(size_t count) {
    try_reserve(count);
    size_ = count <= capacity_ ? count : capacity_;
  }

  // Tries increasing the buffer capacity to *new_capacity*. It can increase the
  // capacity by a smaller amount than requested but guarantees there is space
  // for at least one additional element either by increasing the capacity or by
  // flushing the buffer if it is full.
  void try_reserve(size_t new_capacity) {
    if (new_capacity > capacity_) grow(new_capacity);
  }

  void push_back(const T& value) {
    try_reserve(size_ + 1);
    ptr_[size_++] = value;
  }

  /** Appends data to the end of the buffer. */
  template <typename U> void append(const U* begin, const U* end);

  template <typename I> T& operator[](I index) { return ptr_[index]; }
  template <typename I> const T& operator[](I index) const {
    return ptr_[index];
  }
};

```

这段代码定义了两个结构体：buffer_traits和fixed_buffer_traits。buffer_traits是一个模板类，其中包含了一个构造函数、一个成员变量count和一个成员函数limit。fixed_buffer_traits是一个具体的模板类，其中包含了一个构造函数、一个成员变量count_和一个成员函数limit_。

buffer_traits和fixed_buffer_traits的成员函数都使用了模板参数，这样就可以在使用时将不同的数据类型（如int和double）作为模板实参。例如，在定义int类型的变量count时，可以使用模板函数来计算并初始化count的值。

buffer_traits和fixed_buffer_traits的成员变量都使用了const类型，这意味着它们被声明为常量，不能被修改。

这段代码的用途是定义了一个模板类，用于存储一个缓冲区。该模板类可以用来定义一个字节序列，该序列可以被任何具有相同count_成员变量大小的整数类型的变量所占用。同时，该模板类还提供了一个limit成员函数，用于设置缓冲区的最大大小。


```cpp
struct buffer_traits {
  explicit buffer_traits(size_t) {}
  size_t count() const { return 0; }
  size_t limit(size_t size) { return size; }
};

class fixed_buffer_traits {
 private:
  size_t count_ = 0;
  size_t limit_;

 public:
  explicit fixed_buffer_traits(size_t limit) : limit_(limit) {}
  size_t count() const { return count_; }
  size_t limit(size_t size) {
    size_t n = limit_ - count_;
    count_ += size;
    return size < n ? size : n;
  }
};

```

这段代码定义了一个名为“iterator_buffer”的模板类，该类实现了对一个输出迭代器的写入。这个类模板包括一个输出迭代器“out”和一个数据类型为“T”的整型变量“data”。

该类的实现包括两个私有成员函数：
1. “grow”函数，该函数用于在“buffer_size”达到给定的缓冲区大小时，将“buffer”中未写入的部分移出并写入到“out”上，从而实现“flush”整个缓冲区。
2. “flush”函数，用于将“buffer”中所有内容一次性写入到“out”上。

该类还有一个公有的构造函数，用于初始化一个输出迭代器“out”和一个整型变量“n”，当“n”小于“buffer_size”时，将“n”赋值为“buffer_size”。

该类有一个成员函数“out()”，用于返回一个指向“out”的指针，并在调用“flush()”函数后返回新的“out”。

该类还有一个名为“count()”的成员函数，返回“Traits”继承类中“count()”函数的返回值加上“buffer”类中“size()”函数的返回值。

最后，该类的“grow()”函数被声明为“final”的函数，意味着该函数不能被覆盖。


```cpp
// A buffer that writes to an output iterator when flushed.
template <typename OutputIt, typename T, typename Traits = buffer_traits>
class iterator_buffer : public Traits, public buffer<T> {
 private:
  OutputIt out_;
  enum { buffer_size = 256 };
  T data_[buffer_size];

 protected:
  void grow(size_t) final FMT_OVERRIDE {
    if (this->size() == buffer_size) flush();
  }
  void flush();

 public:
  explicit iterator_buffer(OutputIt out, size_t n = buffer_size)
      : Traits(n),
        buffer<T>(data_, 0, n < size_t(buffer_size) ? n : size_t(buffer_size)),
        out_(out) {}
  ~iterator_buffer() { flush(); }

  OutputIt out() {
    flush();
    return out_;
  }
  size_t count() const { return Traits::count() + this->size(); }
};

```

这两段代码定义了一个名为“iterator_buffer”的模板类，该类旨在在容器中读取或写入数据。

第一段代码定义了“iterator_buffer”模板类，该类基于“buffer”模板类，并对其进行了保护。该类包含一个名为“grow”的protected函数，用于在需要时动态地扩展缓冲区的大小。该函数在这里实现了，但并没有进行实际的实现。

第二段代码定义了一个名为“iterator_buffer”的模板类，该类使用“std::back_insert_iterator”容器类型。该类基于“buffer”模板类，并实现了“std::back_inserter”类型的函数。该函数接受一个“Container”对象和一个或多个参数，用于将数据写入该容器。该函数在这里实现了，但并没有进行实际的实现。

这两个模板类都继承自“buffer”模板类，并实现了“set”函数，用于设置缓冲区的起始和结束位置。它们还实现了“grow”函数，用于动态地扩展缓冲区的大小。但是，这两个模板类没有进行实际的实现，也没有进行任何检查，以确保它们能够在编译时正确地工作。


```cpp
template <typename T> class iterator_buffer<T*, T> : public buffer<T> {
 protected:
  void grow(size_t) final FMT_OVERRIDE {}

 public:
  explicit iterator_buffer(T* out, size_t = 0) : buffer<T>(out, 0, ~size_t()) {}

  T* out() { return &*this->end(); }
};

// A buffer that writes to a container with the contiguous storage.
template <typename Container>
class iterator_buffer<std::back_insert_iterator<Container>,
                      enable_if_t<is_contiguous<Container>::value,
                                  typename Container::value_type>>
    : public buffer<typename Container::value_type> {
 private:
  Container& container_;

 protected:
  void grow(size_t capacity) final FMT_OVERRIDE {
    container_.resize(capacity);
    this->set(&container_[0], capacity);
  }

 public:
  explicit iterator_buffer(Container& c)
      : buffer<typename Container::value_type>(c.size()), container_(c) {}
  explicit iterator_buffer(std::back_insert_iterator<Container> out, size_t = 0)
      : iterator_buffer(get_container(out)) {}
  std::back_insert_iterator<Container> out() {
    return std::back_inserter(container_);
  }
};

```

该代码定义了一个名为`counting_buffer`的模板类，其中`T`被指定为`char`。这个模板类继承自`buffer`类，因此继承了`buffer`类的所有方法。

在这个模板类的实现中，`counting_buffer`类维护了一个整数计数器`count_`，以及一个大小为`buffer_size`的整数数组`data_`。`buffer_size`是一个类成员变量，用于指定计数器的长度。

`grow`方法是一个保护成员函数，用于在需要扩容时增长计数器。当`count_`计数器的值达到`buffer_size`时，`grow`函数会被调用，此时它会执行以下操作来扩容计数器：

1. 如果当前计数器大小与`buffer_size`不相等，那么直接返回。
2. 如果计数器大小已达到`buffer_size`，则增加`count_`计数器的值并将`data_`数组清空。
3. 在此过程中，将`count_`计数器的值增加`this->size()`的大小。

`counting_buffer`类有两个构造函数，一个是默认构造函数，另一个是使用`T`指定参数的构造函数。在前者的构造函数中，将`buffer`类的默认参数传递给`counting_buffer`类的构造函数，以初始化`data_`数组和`count_`计数器。在后者的构造函数中，使用`T`参数指定要存储的数据类型，并将其赋值给`counting_buffer`类的构造函数参数。

`counting_buffer`类的`count`成员函数返回当前计数器的值，它会更新`count_`计数器和`data_`数组。


```cpp
// A buffer that counts the number of code units written discarding the output.
template <typename T = char> class counting_buffer : public buffer<T> {
 private:
  enum { buffer_size = 256 };
  T data_[buffer_size];
  size_t count_ = 0;

 protected:
  void grow(size_t) final FMT_OVERRIDE {
    if (this->size() != buffer_size) return;
    count_ += this->size();
    this->clear();
  }

 public:
  counting_buffer() : buffer<T>(data_, 0, buffer_size) {}

  size_t count() { return count_ + this->size(); }
};

```

这段代码定义了一个名为`buffer_appender`的类，该类实现了一个输出迭代器，可以将元素添加到`buffer<T>`类型的缓冲区中。

具体来说，该类模板参数为`T`，表示要存储的数据类型。然后，该类定义了一个`base`成员变量，它是一个`std::back_insert_iterator<buffer<T>>`类型的成员变量，表示一个`buffer<T>`类型的缓冲区的入口指针。

接着，该类定义了`buffer_appender`类的两个构造函数，第一个构造函数接受一个`buffer<T>`类型的参数，通过这个参数来初始化`base`成员变量，将`base`成员变量初始化为`buf`。第二个构造函数接受一个`base`类型的指针，通过这个指针来初始化`base`成员变量，将`base`成员变量初始化为`it`。

然后，该类定义了一个`operator++`方法，用于对`base`成员变量进行递增操作，并返回`this`指针。最后，通过在`operator++`方法中传入`std::back_insert_iterator<buffer<T>>`类型的参数，可以实现将元素添加到缓冲区中。


```cpp
// An output iterator that appends to the buffer.
// It is used to reduce symbol sizes for the common case.
template <typename T>
class buffer_appender : public std::back_insert_iterator<buffer<T>> {
  using base = std::back_insert_iterator<buffer<T>>;

 public:
  explicit buffer_appender(buffer<T>& buf) : base(buf) {}
  buffer_appender(base it) : base(it) {}

  buffer_appender& operator++() {
    base::operator++();
    return *this;
  }

  buffer_appender operator++(int) {
    buffer_appender tmp = *this;
    ++*this;
    return tmp;
  }
};

```

这段代码定义了一个名为`Maps`的模板元编程技术，该技术将输出迭代器（Out Iterator）作为一个模板参数，然后通过一系列的`<>`合成另一个模板参数，这个模板参数可以是`void`（代表不使用任何模板参数）、类型参数或者是模板元编程技术自身。

`< typename T, typename OutputIt >`是一个模板元编程技术，用于在编译时检查代码的语法。`<>`符号表示该技术可以合成两个模板参数。在这里，第一个模板参数是一个开放的市场（< T >），表示需要一个类型参数；第二个模板参数是一个输出迭代器（< OutputIt >）。

`< typename T >`是一个模板元编程技术，用于在编译时检查代码的语法。`< buffer_appender<T> >`是一个模板元编程技术，用于创建一个接受类型为`T`的输出迭代器。

`< typename OutputIt>`是一个模板元编程技术，用于在编译时检查代码的语法。`< OutputIt >`是一个开放的市场，表示需要一个类型参数。

`template <typename OutputIt> OutputIt get_buffer_init(OutputIt out) {
   return out;
}`是一个函数，它接受一个输出迭代器（Out Iterator）`out`作为参数，然后返回该输出迭代器。

`template <typename T> buffer<T>& get_buffer_init(buffer_appender<T> out) {
   return get_container(out);
}`是一个函数，它接受一个类型为`T`的输出迭代器`out`作为参数，然后返回该类型参数的容器对象。`get_container()`是一个已经定义的函数，它接受一个输出迭代器`out`作为参数，然后返回一个指向`T`类型对象的指针。

`template <typename Buffer>
auto get_iterator(Buffer& buf) -> decltype(buf.out()) {
   return buf.out();
}`是一个函数，它接受一个输出迭代器`buf`作为参数，然后返回该输出迭代器的类型。`get_iterator()`是一个已经定义的函数，它接受一个输出迭代器`buf`作为参数，然后返回该输出迭代器的类型。

总之，这段代码定义了一个可以将输出迭代器（Out Iterator）映射到一个内存区域的模板，这个模板使用了一个`<>`符号来合成两个模板参数，第一个模板参数是`void`，表示不使用任何模板参数；第二个模板参数是类型参数，表示需要一个类型参数。


```cpp
// Maps an output iterator into a buffer.
template <typename T, typename OutputIt>
iterator_buffer<OutputIt, T> get_buffer(OutputIt);
template <typename T> buffer<T>& get_buffer(buffer_appender<T>);

template <typename OutputIt> OutputIt get_buffer_init(OutputIt out) {
  return out;
}
template <typename T> buffer<T>& get_buffer_init(buffer_appender<T> out) {
  return get_container(out);
}

template <typename Buffer>
auto get_iterator(Buffer& buf) -> decltype(buf.out()) {
  return buf.out();
}
```

这段代码定义了一个名为`buffer_appender`的模板类，该类模板参数为`T`类型。这个模板类提供了一个名为`get_iterator`的函数，它接受一个`buffer<T>`类型的参数。

`get_iterator`函数返回一个指向`buffer_appender<T>`类型的指针。

接下来是另一个模板类定义，该类模板参数为`T`，且类型为`char`（`char`是`<T>`中的别名），名为`fallback_formatter`。该模板类声明了一个名为`__fallback_formatter`的成员函数，但该函数声明为`__delete`，这意味着该成员函数实际上是一个delete类型的函数，不会在编译期产生任何作用。

最后，该代码段定义了一个名为`view`的模板类，该类模板参数为`char`类型。注意，这个模板类没有任何成员函数或变量。


```cpp
template <typename T> buffer_appender<T> get_iterator(buffer<T>& buf) {
  return buffer_appender<T>(buf);
}

template <typename T, typename Char = char, typename Enable = void>
struct fallback_formatter {
  fallback_formatter() = delete;
};

// Specifies if T has an enabled fallback_formatter specialization.
template <typename T, typename Context>
using has_fallback_formatter =
    std::is_constructible<fallback_formatter<T, typename Context::char_type>>;

struct view {};

```

这段代码定义了一个模板结构体 named_arg，其中包含了两个模板参数：一个 typename Char 的模板参数和一个 typename T 的模板参数。named_arg 结构体模板中包含一个 const Char* 的成员 name 和一个 const T 的成员 value，并且在成员 name 和 value 上使用了 named_arg 在模板定义中的名称。

另外，还定义了一个名为 named_arg_info 的模板结构体，其中包含一个 const Char* 的成员 name 和一个 int 类型的成员 id。

接着，定义了一个名为 arg_data 的模板结构体，用于在主模板中声明参数。arg_data 的模板定义中包含一个模板参数 T，和一个模板参数 Char，以及一个整数类型的 NUM_ARGS 和 NUM_NAMED_ARGS 类型的模板参数。NUM_ARGS 是一个非空模板参数，通过它可以确定参数数量的范围，而 NUM_NAMED_ARGS 是模板中 named_arg 结构体中成员数量的一个非空模板参数。在模板定义中，通过 NUM_ARGS 和 NUM_NAMED_ARGS 可以确定参数数量的范围，从而可以知道模板参数的类型。

最终，在 main 函数中，通过传入不同的模板参数，可以获得不同的 T 类型的参数。例如，当 template 参数为 T(int, char) 时，arg_data 中 named_args_[0] 的成员 variable 将被初始化，而 named_args_[1] 将保留不变。


```cpp
template <typename Char, typename T> struct named_arg : view {
  const Char* name;
  const T& value;
  named_arg(const Char* n, const T& v) : name(n), value(v) {}
};

template <typename Char> struct named_arg_info {
  const Char* name;
  int id;
};

template <typename T, typename Char, size_t NUM_ARGS, size_t NUM_NAMED_ARGS>
struct arg_data {
  // args_[0].named_args points to named_args_ to avoid bloating format_args.
  // +1 to workaround a bug in gcc 7.5 that causes duplicated-branches warning.
  T args_[1 + (NUM_ARGS != 0 ? NUM_ARGS : +1)];
  named_arg_info<Char> named_args_[NUM_NAMED_ARGS];

  template <typename... U>
  arg_data(const U&... init) : args_{T(named_args_, NUM_NAMED_ARGS), init...} {}
  arg_data(const arg_data& other) = delete;
  const T* args() const { return args_ + 1; }
  named_arg_info<Char>* named_args() { return named_args_; }
};

```

这段代码定义了一个模板结构体 `arg_data`，其中 `T` 和 `Char` 是两个模板参数。`NUM_ARGS` 是该模板结构体的数量，初始值为 `2`，表示 `arg_data` 需要 `T` 和 `Char` 两个参数。

`arg_data` 模板结构体包括一个名为 `args_` 的成员变量，其类型为 `T` 模板类型。这个成员变量是一个大小为 `NUM_ARGS`(不等于 0)的一维数组，用于存储模板参数。如果没有传给 `arg_data` 的 `T` 模板参数，那么 `args_` 的大小将等于 `NUM_ARGS`。

`arg_data` 模板结构体还包括三个成员函数：

- `arg_data` 模板结构体中的 `args_` 成员函数，是一个支路函数，用于在编译时检查模板参数的重复引用警告。这个函数接受一个参数 `U`，作为模板参数，如果没有这个参数，则 `args_` 将只包含一个元素，即 `args_[0]`。
- `const T*` 类型的 `args` 成员函数，用于获取模板参数的值，这个函数接受一个模板参数 `U`，作为模板参数，返回值为 `args_[size_t<U>:：模板支路<sizeof(U)> - 1]`。
- `std::nullptr_t` 类型的 `named_args` 成员函数，用于设置 `arg_data` 模板结构体的 `named_args` 成员的值，这个函数接受两个参数，第一个参数是一个 `named_arg_info<Char>` 类型的对象，第二个参数是 `int` 类型的参数，用于指定模板参数的名称。

整段代码的作用是定义了一个模板结构体 `arg_data`，其中 `T` 和 `Char` 是两个模板参数，`NUM_ARGS` 是模板结构体的参数数量，初始值为 `2`。这个模板结构体可以在编译时提供一些语义信息，例如模板参数的名称和类型，以及模板值的类型和签名。


```cpp
template <typename T, typename Char, size_t NUM_ARGS>
struct arg_data<T, Char, NUM_ARGS, 0> {
  // +1 to workaround a bug in gcc 7.5 that causes duplicated-branches warning.
  T args_[NUM_ARGS != 0 ? NUM_ARGS : +1];

  template <typename... U>
  FMT_INLINE arg_data(const U&... init) : args_{init...} {}
  FMT_INLINE const T* args() const { return args_; }
  FMT_INLINE std::nullptr_t named_args() { return nullptr; }
};

template <typename Char>
inline void init_named_args(named_arg_info<Char>*, int, int) {}

template <typename Char, typename T, typename... Tail>
```

该代码定义了一个名为 `init_named_args` 的函数，它接受一个名为 `named_arg_info` 的模板类参数，该模板类包含一个字符类型的变量 `named_args` 和一个整数类型的变量 `arg_count`，以及一个字符类型的变量 `named_arg_count` 和一个整数类型的变量 `args...`。

函数中包含两个版本，第一个版本的参数列表为 `named_args`、`arg_count` 和 `named_arg_count`，而第二个版本的参数列表为 `args...`、`args...` 和 `args...`。可以推测 `named_args` 和 `args...` 可能是用来在函数内部传递给其他函数或定义的变量。

函数的具体实现包括两个版本，第一个版本接收一个名为 `args...` 的参数列表，将其与 `named_args` 中的 `arg_count` 相加，然后将 `args...` 中的参数复制到 `named_args` 中。第二个版本接收一个名为 `args...` 的参数列表，将其与 `named_args` 中的 `arg_count` 相加，然后将 `args...` 中的参数复制到 `named_args` 中。这样，第一个版本的函数可以接受一个字符串类型的参数列表，而第二个版本的函数可以接受任何类型的参数列表（包括字符串、整数和其他基本数据类型）。

在初始化函数内部，第一个版本使用了模板元编程技巧，在 `named_args` 上创建了一个新的命名参数，然后将其与 `arg_count` 相加，最后将参数列表传递给 `init_named_args` 函数。第二个版本则直接在 `named_args` 上进行参数列表的初始化，然后将所有参数复制到 `named_args` 中。

另外，该代码中还有一句注释，指出 `void` 函数的参数列表中传递了一个 `nullptr_t` 类型的参数，但这似乎没有被使用。


```cpp
void init_named_args(named_arg_info<Char>* named_args, int arg_count,
                     int named_arg_count, const T&, const Tail&... args) {
  init_named_args(named_args, arg_count + 1, named_arg_count, args...);
}

template <typename Char, typename T, typename... Tail>
void init_named_args(named_arg_info<Char>* named_args, int arg_count,
                     int named_arg_count, const named_arg<Char, T>& arg,
                     const Tail&... args) {
  named_args[named_arg_count++] = {arg.name, arg_count};
  init_named_args(named_args, arg_count + 1, named_arg_count, args...);
}

template <typename... Args>
FMT_INLINE void init_named_args(std::nullptr_t, int, int, const Args&...) {}

```

这段代码定义了一系列模板结构体，用于描述输入参数的命名方式。

首先，定义了一个模板结构体`is_named_arg`，它的别列为`std::false_type`。

接着，定义了一个模板结构体`is_named_arg<named_arg<T>>`，它的别列为`std::true_type`，其中`named_arg<T>`是一个模板实参，通过这个实参，这个模板结构体可以用来编译时检查函数参数的命名是否符合要求。

然后，定义了一个模板结构体`size_t`，用于表示输入参数的大小。

接下来，定义了一个模板函数`count()`，用于计算一个二元条件的计数。这个函数有两个实现，分别针对`true`和`false`条件，其中第一个实现是空的，用于在编译时提示函数参数不满足模板要求。第二个实现是一个大小为`0`的`size_t`，用于计算一个二元条件的计数。

接着，定义了一个模板函数`count<T...>()`，用于计算一个多元条件的计数。这个函数有一个实现，用于计算一个二元条件的计数，其中`T...`是一个模板实参，通过这个实参，这个模板函数可以用来编译时检查函数参数的命名是否符合要求。

然后，定义了一个模板结构体`count_named_args()`，用于计算命名参数的数量。这个结构体有一个实现，用于计算`is_named_arg<Args>`中所有实参的值，其中`Args`是一个模板实参。

接下来，定义了一个模板类`type`，用于描述输入参数的类型。这个模板类包括了一系列基本的输入参数类型，如`int_type`，`uint_type`，`long_long_type`，`ulong_long_type`，`int128_type`，`uint128_type`，`long_long_type`，`ulong_long_type`，`int128_type`，`uint128_type`，`bool_type`，`char_type`，`last_integer_type`，`float_type`，`double_type`，`long_double_type`，`last_numeric_type`，`cstring_type`，`string_type`，`pointer_type`，`custom_type`。

最后，定义了一个模板函数`count_named_args()`，用于计算一个多元条件的计数，这个函数的实现与上面的`count()`函数相同，但使用了`is_named_arg<Args>`这个模板结构体，而不是其中的一个实参。


```cpp
template <typename T> struct is_named_arg : std::false_type {};

template <typename T, typename Char>
struct is_named_arg<named_arg<Char, T>> : std::true_type {};

template <bool B = false> constexpr size_t count() { return B ? 1 : 0; }
template <bool B1, bool B2, bool... Tail> constexpr size_t count() {
  return (B1 ? 1 : 0) + count<B2, Tail...>();
}

template <typename... Args> constexpr size_t count_named_args() {
  return count<is_named_arg<Args>::value...>();
}

enum class type {
  none_type,
  // Integer types should go first,
  int_type,
  uint_type,
  long_long_type,
  ulong_long_type,
  int128_type,
  uint128_type,
  bool_type,
  char_type,
  last_integer_type = char_type,
  // followed by floating-point types.
  float_type,
  double_type,
  long_double_type,
  last_numeric_type = long_double_type,
  cstring_type,
  string_type,
  pointer_type,
  custom_type
};

```

这段代码定义了一个模板结构体`type_constant`，它可以将一种类型(`T`)转换为另一种类型(`enum constant`)。其中，`T`是一个模板型参数，而`enum constant`是一个已知类型，会在编译时根据给定的模板型参数进行枚举类型检查，并将已知类型映射到相应的模板类型常量上。

具体来说，这段代码可以看作是在定义一个`enum`类型的枚举，其中每个枚举元素都是一个模板类型参数(`T`)，但每个枚举元素都有一个额外的类型参数，表示要枚举的类型。例如，`int_type`表示要枚举的类型是`int`属于`enum`中的哪一个，`uint_type`表示要枚举的类型是`unsigned`属于`enum`中的哪一个，以此类推。

由于每个枚举元素都有一个额外的类型参数，因此每个枚举元素实际上是一个模板类型参数，而`type_constant`结构体就是用来声明这些模板类型的常量的。

这段代码对于那些需要根据输入类型进行枚举类型检查的人来说，可以提供一种简单而一致的方式来定义已知类型的枚举类型。


```cpp
// Maps core type T to the corresponding type enum constant.
template <typename T, typename Char>
struct type_constant : std::integral_constant<type, type::custom_type> {};

#define FMT_TYPE_CONSTANT(Type, constant) \
  template <typename Char>                \
  struct type_constant<Type, Char>        \
      : std::integral_constant<type, type::constant> {}

FMT_TYPE_CONSTANT(int, int_type);
FMT_TYPE_CONSTANT(unsigned, uint_type);
FMT_TYPE_CONSTANT(long long, long_long_type);
FMT_TYPE_CONSTANT(unsigned long long, ulong_long_type);
FMT_TYPE_CONSTANT(int128_t, int128_type);
FMT_TYPE_CONSTANT(uint128_t, uint128_type);
```

这段代码定义了几个FMT_TYPE_CONSTANT类型，包括boolean、char、float、double、long double、const char*、basic_string_view和const void*等。

FMT_TYPE_CONSTANT类型是指送信类型，用于格式化输出。其中，boolean类型包括bool_type、bool_constant、false_type、false、bool_var等，共5种类型。char类型包括char_type、char_constant、char_var等，共3种类型。float类型包括float_type、float_constant、float_var等，共3种类型。double类型包括double_type、double_constant、double_var等，共3种类型。long double类型包括long_double_type、long_double_constant、long_double_var等，共3种类型。const char*类型包括cstring_type、const_char*等，共2种类型。basic_string_view类型包括string_type、basic_string_view_const等，共2种类型。const void*类型包括pointer_type、const_void*等，共2种类型。

接下来的is_integral_type函数用于判断一个类型是否为整数类型。is_arithmetic_type函数用于判断一个类型是否为算术类型。


```cpp
FMT_TYPE_CONSTANT(bool, bool_type);
FMT_TYPE_CONSTANT(Char, char_type);
FMT_TYPE_CONSTANT(float, float_type);
FMT_TYPE_CONSTANT(double, double_type);
FMT_TYPE_CONSTANT(long double, long_double_type);
FMT_TYPE_CONSTANT(const Char*, cstring_type);
FMT_TYPE_CONSTANT(basic_string_view<Char>, string_type);
FMT_TYPE_CONSTANT(const void*, pointer_type);

constexpr bool is_integral_type(type t) {
  return t > type::none_type && t <= type::last_integer_type;
}

constexpr bool is_arithmetic_type(type t) {
  return t > type::none_type && t <= type::last_numeric_type;
}

```

这段代码定义了三个模板结构体：string_value，named_arg_value和custom_value。

string_value模板结构体定义了一个字符串类型的变量data，以及一个表示字符串的大小的成员size。

named_arg_value模板结构体定义了一个名为named_arg_info的指向字符串类型的指针data，以及一个表示字符串的大小的成员size。

custom_value模板结构体定义了一个字符串类型的变量value，以及一个用于格式化输出值的函数format和一个用于解析字符串的函数parse_context，以及一个指向void类型的指针<void>* value。这个value变量存储了一个格式化的字符串，可以根据parse_context的值输出不同的字符串。


```cpp
template <typename Char> struct string_value {
  const Char* data;
  size_t size;
};

template <typename Char> struct named_arg_value {
  const named_arg_info<Char>* data;
  size_t size;
};

template <typename Context> struct custom_value {
  using parse_context = typename Context::parse_context_type;
  const void* value;
  void (*format)(const void* arg, parse_context& parse_ctx, Context& ctx);
};

```

This is a C++ implementation of a simple custom formatter that allows for formatting of arguments of a given type using a specified format string. The custom formatter can handle various types, including floating-point numbers, double numbers, long double numbers, booleans, and character types.

The `value` class is used to store the actual argument value that is to be formatted. The formatter object, which is passed to the `format_custom_arg` function, is used to determine the format string to use for the argument. If a provided formatter cannot be found for the given argument type, the default fallback formatter is used.

The `FMT_INLINE` macro is used to indicate that the function or variable is intended to be a template parameter.


```cpp
// A formatting argument value.
template <typename Context> class value {
 public:
  using char_type = typename Context::char_type;

  union {
    int int_value;
    unsigned uint_value;
    long long long_long_value;
    unsigned long long ulong_long_value;
    int128_t int128_value;
    uint128_t uint128_value;
    bool bool_value;
    char_type char_value;
    float float_value;
    double double_value;
    long double long_double_value;
    const void* pointer;
    string_value<char_type> string;
    custom_value<Context> custom;
    named_arg_value<char_type> named_args;
  };

  constexpr FMT_INLINE value(int val = 0) : int_value(val) {}
  constexpr FMT_INLINE value(unsigned val) : uint_value(val) {}
  FMT_INLINE value(long long val) : long_long_value(val) {}
  FMT_INLINE value(unsigned long long val) : ulong_long_value(val) {}
  FMT_INLINE value(int128_t val) : int128_value(val) {}
  FMT_INLINE value(uint128_t val) : uint128_value(val) {}
  FMT_INLINE value(float val) : float_value(val) {}
  FMT_INLINE value(double val) : double_value(val) {}
  FMT_INLINE value(long double val) : long_double_value(val) {}
  FMT_INLINE value(bool val) : bool_value(val) {}
  FMT_INLINE value(char_type val) : char_value(val) {}
  FMT_INLINE value(const char_type* val) { string.data = val; }
  FMT_INLINE value(basic_string_view<char_type> val) {
    string.data = val.data();
    string.size = val.size();
  }
  FMT_INLINE value(const void* val) : pointer(val) {}
  FMT_INLINE value(const named_arg_info<char_type>* args, size_t size)
      : named_args{args, size} {}

  template <typename T> FMT_INLINE value(const T& val) {
    custom.value = &val;
    // Get the formatter type through the context to allow different contexts
    // have different extension points, e.g. `formatter<T>` for `format` and
    // `printf_formatter<T>` for `printf`.
    custom.format = format_custom_arg<
        T, conditional_t<has_formatter<T, Context>::value,
                         typename Context::template formatter_type<T>,
                         fallback_formatter<T, char_type>>>;
  }

 private:
  // Formats an argument of a custom type, such as a user-defined class.
  template <typename T, typename Formatter>
  static void format_custom_arg(const void* arg,
                                typename Context::parse_context_type& parse_ctx,
                                Context& ctx) {
    Formatter f;
    parse_ctx.advance_to(f.parse(parse_ctx));
    ctx.advance_to(f.format(*static_cast<const T*>(arg), ctx));
  }
};

```

This is a C++ template class that defines a formatting macro for arbitrary pointers. The class provides several overloads for the `map()` function, which takes a pointer to a generic type and returns a pointer to the mapped type.

The first overload `map()` takes a non-null pointer `val` and returns a pointer to it. If the `val` is a null pointer, the function returns a null pointer.

The second overload `map()` takes a generic type parameter `T` and returns a pointer to the mapped type. If the `T` is an enumeration type, the function template allows for a subset of the enumeration members to be used in the mapped type.

The third overload `map()` takes a generic type parameter `T` and returns a pointer to the mapped type. If the `T` is a string type, the function template allows for a subset of the string literals to be used in the mapped type.

The fourth overload `map()` takes a named argument `named_arg` of a generic type and returns a pointer to the mapped type.


```cpp
template <typename Context, typename T>
FMT_CONSTEXPR basic_format_arg<Context> make_arg(const T& value);

// To minimize the number of types we need to deal with, long is translated
// either to int or to long long depending on its size.
enum { long_short = sizeof(long) == sizeof(int) };
using long_type = conditional_t<long_short, int, long long>;
using ulong_type = conditional_t<long_short, unsigned, unsigned long long>;

struct unformattable {};

// Maps formatting arguments to core types.
template <typename Context> struct arg_mapper {
  using char_type = typename Context::char_type;

  FMT_CONSTEXPR int map(signed char val) { return val; }
  FMT_CONSTEXPR unsigned map(unsigned char val) { return val; }
  FMT_CONSTEXPR int map(short val) { return val; }
  FMT_CONSTEXPR unsigned map(unsigned short val) { return val; }
  FMT_CONSTEXPR int map(int val) { return val; }
  FMT_CONSTEXPR unsigned map(unsigned val) { return val; }
  FMT_CONSTEXPR long_type map(long val) { return val; }
  FMT_CONSTEXPR ulong_type map(unsigned long val) { return val; }
  FMT_CONSTEXPR long long map(long long val) { return val; }
  FMT_CONSTEXPR unsigned long long map(unsigned long long val) { return val; }
  FMT_CONSTEXPR int128_t map(int128_t val) { return val; }
  FMT_CONSTEXPR uint128_t map(uint128_t val) { return val; }
  FMT_CONSTEXPR bool map(bool val) { return val; }

  template <typename T, FMT_ENABLE_IF(is_char<T>::value)>
  FMT_CONSTEXPR char_type map(T val) {
    static_assert(
        std::is_same<T, char>::value || std::is_same<T, char_type>::value,
        "mixing character types is disallowed");
    return val;
  }

  FMT_CONSTEXPR float map(float val) { return val; }
  FMT_CONSTEXPR double map(double val) { return val; }
  FMT_CONSTEXPR long double map(long double val) { return val; }

  FMT_CONSTEXPR const char_type* map(char_type* val) { return val; }
  FMT_CONSTEXPR const char_type* map(const char_type* val) { return val; }
  template <typename T, FMT_ENABLE_IF(is_string<T>::value)>
  FMT_CONSTEXPR basic_string_view<char_type> map(const T& val) {
    static_assert(std::is_same<char_type, char_t<T>>::value,
                  "mixing character types is disallowed");
    return to_string_view(val);
  }
  template <typename T,
            FMT_ENABLE_IF(
                std::is_constructible<basic_string_view<char_type>, T>::value &&
                !is_string<T>::value && !has_formatter<T, Context>::value &&
                !has_fallback_formatter<T, Context>::value)>
  FMT_CONSTEXPR basic_string_view<char_type> map(const T& val) {
    return basic_string_view<char_type>(val);
  }
  template <
      typename T,
      FMT_ENABLE_IF(
          std::is_constructible<std_string_view<char_type>, T>::value &&
          !std::is_constructible<basic_string_view<char_type>, T>::value &&
          !is_string<T>::value && !has_formatter<T, Context>::value &&
          !has_fallback_formatter<T, Context>::value)>
  FMT_CONSTEXPR basic_string_view<char_type> map(const T& val) {
    return std_string_view<char_type>(val);
  }
  FMT_CONSTEXPR const char* map(const signed char* val) {
    static_assert(std::is_same<char_type, char>::value, "invalid string type");
    return reinterpret_cast<const char*>(val);
  }
  FMT_CONSTEXPR const char* map(const unsigned char* val) {
    static_assert(std::is_same<char_type, char>::value, "invalid string type");
    return reinterpret_cast<const char*>(val);
  }
  FMT_CONSTEXPR const char* map(signed char* val) {
    const auto* const_val = val;
    return map(const_val);
  }
  FMT_CONSTEXPR const char* map(unsigned char* val) {
    const auto* const_val = val;
    return map(const_val);
  }

  FMT_CONSTEXPR const void* map(void* val) { return val; }
  FMT_CONSTEXPR const void* map(const void* val) { return val; }
  FMT_CONSTEXPR const void* map(std::nullptr_t val) { return val; }
  template <typename T> FMT_CONSTEXPR int map(const T*) {
    // Formatting of arbitrary pointers is disallowed. If you want to output
    // a pointer cast it to "void *" or "const void *". In particular, this
    // forbids formatting of "[const] volatile char *" which is printed as bool
    // by iostreams.
    static_assert(!sizeof(T), "formatting of non-void pointers is disallowed");
    return 0;
  }

  template <typename T,
            FMT_ENABLE_IF(std::is_enum<T>::value &&
                          !has_formatter<T, Context>::value &&
                          !has_fallback_formatter<T, Context>::value)>
  FMT_CONSTEXPR auto map(const T& val)
      -> decltype(std::declval<arg_mapper>().map(
          static_cast<typename std::underlying_type<T>::type>(val))) {
    return map(static_cast<typename std::underlying_type<T>::type>(val));
  }
  template <typename T,
            FMT_ENABLE_IF(!is_string<T>::value && !is_char<T>::value &&
                          (has_formatter<T, Context>::value ||
                           has_fallback_formatter<T, Context>::value))>
  FMT_CONSTEXPR const T& map(const T& val) {
    return val;
  }

  template <typename T>
  FMT_CONSTEXPR auto map(const named_arg<char_type, T>& val)
      -> decltype(std::declval<arg_mapper>().map(val.value)) {
    return map(val.value);
  }

  unformattable map(...) { return {}; }
};

```

这段代码定义了一个枚举类型，名为 `enum`，它有两个主要的成员：`max_packed_args` 和 `is_unpacked_bit`。

`max_packed_args` 表示使用 packed 类型可以将最多 62 个参数塞进该枚举中，而 `is_unpacked_bit` 表示是否有名为参数的枚举值。

`enum : unsigned long long { pack_arg_bits = 63LL, unpack_arg_bits = 62LL, max_arg_bits = 63LL, max_named_args_bits = 62LL, pack_arg_bits_extended = 0, unpack_arg_bits_extended = 0 }`

`using mapped_type_constant = enum { pack_arg_bits = 63LL, unpack_arg_bits = 62LL, max_arg_bits = 63LL, max_named_args_bits = 62LL, pack_arg_bits_extended = 0, unpack_arg_bits_extended = 0 }`

`template <typename T, typename Context>`

`using mapped_type_constant` 是 `enum` 中定义的枚举类型，`T` 是枚举类型参数，`Context` 是 `enum` 的别名，这两个类型都被绑定为模板类型参数。

`mapped_type_constant` 被定义为：`decltype(arg_mapper<Context>().map(std::declval<const T&>()))` 的类型，这个类型是一个模板类类型，它由 `arg_mapper<Context>().map(std::declval<const T&>())` 和 `std::declval<const T&>()` 编译生成。

`template <typename T, typename Context>`

`using mapped_type_constant` 被用于定义一个类型常量，它使用了 `arg_mapper<Context>().map(std::declval<const T&>())` 函数来获取输入参数的映射类型，然后使用模板元编程技术将其转换为 `T` 类型，这个类型常量可以被用来在程序中使用。


```cpp
// A type constant after applying arg_mapper<Context>.
template <typename T, typename Context>
using mapped_type_constant =
    type_constant<decltype(arg_mapper<Context>().map(std::declval<const T&>())),
                  typename Context::char_type>;

enum { packed_arg_bits = 4 };
// Maximum number of arguments with packed types.
enum { max_packed_args = 62 / packed_arg_bits };
enum : unsigned long long { is_unpacked_bit = 1ULL << 63 };
enum : unsigned long long { has_named_args_bit = 1ULL << 62 };
}  // namespace detail

// A formatting argument. It is a trivially copyable/constructible type to
// allow storage in basic_memory_buffer.
```

这段代码定义了一个名为“basic_format_arg”的模板类，用于在给定的上下文中格式化输入数据。

该模板类有两个私有成员变量：一个类型为“Context”的私有成员变量“value_”和一个类型为“basic_format_arg”的私有成员变量“type_”。

该模板类还有一个名为“make_arg”的类型为“template <typename ContextType, typename T> friend FMT_CONSTEXPR basic_format_arg<ContextType>”的辅助成员函数，用于将给定的值“value_”格式化并返回格式化的结果。

此外，该模板类有一个名为“detail::arg_data”的类型为“struct”的私有成员函数，用于在给定的上下文中格式化输入数据。

最后，该模板类有一个名为“basic_format_args”的类型为“class”的私有成员类，用于在给定的上下文中存储格式化的输入数据。该类有一个“handle”类型的成员变量，用于在存储格式化的输入数据时指定具体的上下文。


```cpp
template <typename Context> class basic_format_arg {
 private:
  detail::value<Context> value_;
  detail::type type_;

  template <typename ContextType, typename T>
  friend FMT_CONSTEXPR basic_format_arg<ContextType> detail::make_arg(
      const T& value);

  template <typename Visitor, typename Ctx>
  friend FMT_CONSTEXPR auto visit_format_arg(Visitor&& vis,
                                             const basic_format_arg<Ctx>& arg)
      -> decltype(vis(0));

  friend class basic_format_args<Context>;
  friend class dynamic_format_arg_store<Context>;

  using char_type = typename Context::char_type;

  template <typename T, typename Char, size_t NUM_ARGS, size_t NUM_NAMED_ARGS>
  friend struct detail::arg_data;

  basic_format_arg(const detail::named_arg_info<char_type>* args, size_t size)
      : value_(args, size) {}

 public:
  class handle {
   public:
    explicit handle(detail::custom_value<Context> custom) : custom_(custom) {}

    void format(typename Context::parse_context_type& parse_ctx,
                Context& ctx) const {
      custom_.format(custom_.value, parse_ctx, ctx);
    }

   private:
    detail::custom_value<Context> custom_;
  };

  constexpr basic_format_arg() : type_(detail::type::none_type) {}

  constexpr explicit operator bool() const FMT_NOEXCEPT {
    return type_ != detail::type::none_type;
  }

  detail::type type() const { return type_; }

  bool is_integral() const { return detail::is_integral_type(type_); }
  bool is_arithmetic() const { return detail::is_arithmetic_type(type_); }
};

```

这段代码是一个C++模板函数，名为`visit_format_arg`，它接受两个参数：`Visitor`和`Context`类型。

该函数的作用是访问一个模板参数类型的`Visitor`，根据参数类型调用相应的`vis`函数，然后将返回值类型声明为` decltype(vis(0))`。

换句话说，该函数将根据传入的`Visitor`类型和`Context`类型，访问对应的`Visitor`函数，然后根据参数类型将返回值类型声明为` decltype(vis(0))`。这个`decltype`操作可以帮助编译器更好地处理模板参数。

例如，如果`Visitor`是一个`void`类型的函数，那么`decltype(vis(0))`将是一个`void`类型的函数，因为`vis`函数没有返回类型。


```cpp
/**
  \rst
  Visits an argument dispatching to the appropriate visit method based on
  the argument type. For example, if the argument type is ``double`` then
  ``vis(value)`` will be called with the value of type ``double``.
  \endrst
 */
template <typename Visitor, typename Context>
FMT_CONSTEXPR_DECL FMT_INLINE auto visit_format_arg(
    Visitor&& vis, const basic_format_arg<Context>& arg) -> decltype(vis(0)) {
  using char_type = typename Context::char_type;
  switch (arg.type_) {
  case detail::type::none_type:
    break;
  case detail::type::int_type:
    return vis(arg.value_.int_value);
  case detail::type::uint_type:
    return vis(arg.value_.uint_value);
  case detail::type::long_long_type:
    return vis(arg.value_.long_long_value);
  case detail::type::ulong_long_type:
    return vis(arg.value_.ulong_long_value);
```

这段代码是一个C++函数，名为`arg.value_.<某一种数据类型>()`，用于根据传入的`arg.value_.<某一种数据类型>()`值，返回相应的数据类型。

首先，通过`FMT_USE_INT128`这个条件判断，如果当前输入的数据类型是`int128_type`，则执行第一种情况，否则执行第二种情况。如果当前输入的数据类型是`uint128_type`，则执行第二种情况，否则执行第一种情况。

对于输入的数据类型是`int128_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.int128_value`值，返回`vis(arg.value_.int128_value)`。

对于输入的数据类型是`uint128_type`，则执行第二种情况，此时函数会根据输入的`arg.value_.uint128_value`值，返回`vis(arg.value_.uint128_value)`。

对于输入的数据类型是`int128_type`，并且输入的`arg.value_.int128_value`值不能直接访问，则执行第二种情况，此时函数会根据输入的`arg.value_.uint128_value`值，返回`vis(arg.value_.uint128_value)`。

对于输入的数据类型是`int128_type`，并且输入的`arg.value_.int128_value`值可以直接访问，则执行第一种情况，此时函数会根据输入的`arg.value_.int128_value`值，返回`vis(arg.value_.int128_value)`。

对于输入的数据类型是`int128_type`，并且输入的`arg.value_.int128_value`值是一个`bool_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.bool_value`值，返回`vis(arg.value_.bool_value)`。

对于输入的数据类型是`int128_type`，并且输入的`arg.value_.int128_value`值是一个`char_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.char_value`值，返回`vis(arg.value_.char_value)`。

对于输入的数据类型是`int128_type`，并且输入的`arg.value_.int128_value`值是一个`float_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.float_value`值，返回`vis(arg.value_.float_value)`。

对于输入的数据类型是`int128_type`，并且输入的`arg.value_.int128_value`值是一个`double_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.double_value`值，返回`vis(arg.value_.double_value)`。

对于输入的数据类型是`int128_type`，并且输入的`arg.value_.int128_value`值是一个`long_double_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.long_double_value`值，返回`vis(arg.value_.long_double_value)`。

对于输入的数据类型是`int128_type`，并且输入的`arg.value_.int128_value`值是一个`cstring_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.string.data`值，返回`vis(arg.value_.string.data)`。

对于输入的数据类型是`string_type`，则执行第二种情况，此时函数会根据输入的`arg.value_.string.data`值，返回`basic_string_view<char_type>(arg.value_.string.data, arg.value_.string.size)`。

对于输入的数据类型是`pointer_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.pointer`值，返回`vis(arg.value_.pointer)`。

对于输入的数据类型是`custom_type`，则执行第一种情况，此时函数会根据输入的`arg.value_.custom`值，返回`typename basic_format_arg<Context>::handle(arg.value_.custom)`。


```cpp
#if FMT_USE_INT128
  case detail::type::int128_type:
    return vis(arg.value_.int128_value);
  case detail::type::uint128_type:
    return vis(arg.value_.uint128_value);
#else
  case detail::type::int128_type:
  case detail::type::uint128_type:
    break;
#endif
  case detail::type::bool_type:
    return vis(arg.value_.bool_value);
  case detail::type::char_type:
    return vis(arg.value_.char_value);
  case detail::type::float_type:
    return vis(arg.value_.float_value);
  case detail::type::double_type:
    return vis(arg.value_.double_value);
  case detail::type::long_double_type:
    return vis(arg.value_.long_double_value);
  case detail::type::cstring_type:
    return vis(arg.value_.string.data);
  case detail::type::string_type:
    return vis(basic_string_view<char_type>(arg.value_.string.data,
                                            arg.value_.string.size));
  case detail::type::pointer_type:
    return vis(arg.value_.pointer);
  case detail::type::custom_type:
    return vis(typename basic_format_arg<Context>::handle(arg.value_.custom));
  }
  return vis(monostate());
}

```

这段代码定义了一个模板结构体 `formattable`，其中 `T` 是一个类型参数。这个结构体有一个成员 `std::false_type`，这意味着它是一个否定类型，即没有类型的模板结构体。

接下来是 `namespace detail` 命名空间，其中定义了一个名为 `void_t_impl` 的模板结构体。这个模板结构体有一个成员 `using type = void`，这意味着它是一个 `void` 的别名，模板实体的类型被指定为 `void`。

然后是多个使用 `using` 关键字的模板结构体，它们通过 `T...` 来指定 `T` 的类型。这些模板结构体的成员都是 `using type`，这意味着它们都有一个 `void` 的别名。

接下来是一个名为 `iterator_category` 的模板结构体，它有一个名为 `std::false_type` 的成员，这意味着它是一个否定类型，即没有类型的模板结构体。它的成员 `using Enable = void`，这意味着它有一个 `void` 的别名 `Enable`。

最后是一行用于 `detail` 命名空间中定义的 `void_t_impl` 的定义，这个定义在函数签名中作为参数出现，并用于 `std::is_same_with` 函数的实现中。


```cpp
template <typename T> struct formattable : std::false_type {};

namespace detail {

// A workaround for gcc 4.8 to make void_t work in a SFINAE context.
template <typename... Ts> struct void_t_impl { using type = void; };

template <typename... Ts>
using void_t = typename detail::void_t_impl<Ts...>::type;

// Detect the iterator category of *any* given type in a SFINAE-friendly way.
// Unfortunately, older implementations of std::iterator_traits are not safe
// for use in a SFINAE-context.
template <typename It, typename Enable = void>
struct iterator_category : std::false_type {};

```

这段代码定义了一个模板结构体，名为 iterator_category，其中包含一个指向任意类型的指针类型。这个结构体在定义时需要指定一个模板参数 T，当缺少模板参数时，会自动推断出 T 的类型为 std::random_access_iterator_tag。

在另一个模板结构体中，名为 is_output_iterator，其中包含一个指向任意类型的指针类型。这个结构体在定义时会检测给定的类型是否符合 output_iterator 的定义，如果符合，就定义了一个名为 value 的枚举类型，它的值为 true，否则定义了一个名为 value 的枚举类型，它的值为 false。

这里使用了 C++14 中关于迭代器和输出迭代器的特性。其中，iterator_category 是 C++20 中 std::output_iterator_tag 的别名，而 std::input_iterator_tag 是 std::input_iterator_tag 的别名。这两个别名在某些 C++编译器中可能已经被废弃了，需要注意。

此外，该代码还定义了一个名为 test 的函数，用于检测指针类型是否符合 output_iterator 的定义。其中，test 函数使用了 std::declval 模板，它会尝试从迭代器中获取值，并将其类型赋给 U。如果 U 是 std::input_iterator_tag，test 函数返回 true；如果 U 是 std::output_iterator_tag，test 函数返回 false。


```cpp
template <typename T> struct iterator_category<T*> {
  using type = std::random_access_iterator_tag;
};

template <typename It>
struct iterator_category<It, void_t<typename It::iterator_category>> {
  using type = typename It::iterator_category;
};

// Detect if *any* given type models the OutputIterator concept.
template <typename It> class is_output_iterator {
  // Check for mutability because all iterator categories derived from
  // std::input_iterator_tag *may* also meet the requirements of an
  // OutputIterator, thereby falling into the category of 'mutable iterators'
  // [iterator.requirements.general] clause 4. The compiler reveals this
  // property only at the point of *actually dereferencing* the iterator!
  template <typename U>
  static decltype(*(std::declval<U>())) test(std::input_iterator_tag);
  template <typename U> static char& test(std::output_iterator_tag);
  template <typename U> static const char& test(...);

  using type = decltype(test<It>(typename iterator_category<It>::type{}));

 public:
  enum { value = !std::is_const<remove_reference_t<type>>::value };
};

```

这段代码定义了一系列模板结构体，用于判断是否是后插入迭代器。

首先，is_back_insert_iterator是一个模板结构体，它有一个标记为template <typename OutputIt> 的类型参数。这个类型参数没有被定义使用任何结构体模板。它还包含一个使用std::false_type和std::true_type定义的两种情况。这两种情况分别表示模板结构体不可能是后插入迭代器和可能是后插入迭代器。

接下来，is_contiguous_back_insert_iterator也是一个模板结构体，它有一个标记为template <typename OutputIt> 的类型参数。这个类型参数使用了std::false_type和is_contiguous<Container>的定义。它还包含一个使用std::back_insert_iterator<Container>的is_contiguous_back_insert_iterator函数。这个函数使用了is_contiguous<Container>的定义，所以它的类型参数使用了Container。

最后，is_contiguous_back_insert_iterator还定义了一个模板结构体，它是一个标记为template <typename Char> 的类型参数。这个类型参数使用了buffer_appender<Char>的定义，所以它的类型参数使用了Char。

整个这段代码的作用是定义了一系列模板结构体，用于判断是否是后插入迭代器。其中，is_back_insert_iterator和is_contiguous_back_insert_iterator用于判断模板结构体是否可能是后插入迭代器，is_contiguous_back_insert_iterator用于判断模板结构体是否是后插入迭代器。


```cpp
template <typename OutputIt>
struct is_back_insert_iterator : std::false_type {};
template <typename Container>
struct is_back_insert_iterator<std::back_insert_iterator<Container>>
    : std::true_type {};

template <typename OutputIt>
struct is_contiguous_back_insert_iterator : std::false_type {};
template <typename Container>
struct is_contiguous_back_insert_iterator<std::back_insert_iterator<Container>>
    : is_contiguous<Container> {};
template <typename Char>
struct is_contiguous_back_insert_iterator<buffer_appender<Char>>
    : std::true_type {};

```

这段代码定义了一个名为locale_ref的类，其作用是提供一个对std::locale类型的指针，以避免在程序中引用std::locale类型，同时提供了一个模板类型的操作符<<，用于将一个Locale对象输出到std::cout上。

具体来说，该代码包含了一个private成员locale_和一些公私共有的成员函数。其中，locale_是一个指向std::locale类型的const void*类型的指针，而其他成员函数则用于操作locale_指向的内存，包括模板<typename Locale>类型的构造函数和get函数等。

locale_ref还提供了一个模板类型的操作符<<，用于将一个Locale对象输出到std::cout上。这个操作符将输入的Locale对象作为参数，并将它输出到std::cout上。

此外，该代码还包含一个名为encode_types的模板类，其返回值为0，表明没有可以编码的类型。


```cpp
// A type-erased reference to an std::locale to avoid heavy <locale> include.
class locale_ref {
 private:
  const void* locale_;  // A type-erased pointer to std::locale.

 public:
  locale_ref() : locale_(nullptr) {}
  template <typename Locale> explicit locale_ref(const Locale& loc);

  explicit operator bool() const FMT_NOEXCEPT { return locale_ != nullptr; }

  template <typename Locale> Locale get() const;
};

template <typename> constexpr unsigned long long encode_types() { return 0; }

```

这段代码定义了一个模板，名为`encode_types()`，模板参数包括一个类型，一个可迭代参数和一个或多个可选参数。函数实现中，通过将可选参数与`mapped_type_constant`组合，然后将组合结果与特定类型（Context）的常量进行按位与操作，得到一个输出类型，该类型将二进制编码，以使得函数签名中，类型参数可以隐式地从编译器中获取到。

接着，定义了一个`make_arg()`函数，该函数接收一个形式化的参数（T），通过对参数进行编码，以使得函数签名中，类型参数可以从编译器中获取到。函数实现中，首先通过`mapped_type_constant`获取到T和Context的常量，然后使用`encode_types()`中计算出的编码类型，将参数进行二进制编码，最后使用`mapped_type_constant`将编码后的类型信息与参数类型进行按位与操作，得到一个返回类型，该类型将二进制编码，以使得函数签名中，类型参数可以从编译器中获取到。

最后，定义了一个`check()`函数，用于检查给定的参数是否为形式化的类型。函数实现中，使用`unformattable`类型的变量并没有初始化，因此它的值将是未定义的。


```cpp
template <typename Context, typename Arg, typename... Args>
constexpr unsigned long long encode_types() {
  return static_cast<unsigned>(mapped_type_constant<Arg, Context>::value) |
         (encode_types<Context, Args...>() << packed_arg_bits);
}

template <typename Context, typename T>
FMT_CONSTEXPR basic_format_arg<Context> make_arg(const T& value) {
  basic_format_arg<Context> arg;
  arg.type_ = mapped_type_constant<T, Context>::value;
  arg.value_ = arg_mapper<Context>().map(value);
  return arg;
}

template <typename T> int check(unformattable) {
  static_assert(
      formattable<T>(),
      "Cannot format an argument. To make type T formattable provide a "
      "formatter<T> specialization: https://fmt.dev/dev/api.html#udt");
  return 0;
}
```

这段代码定义了一个名为 check 的函数模板，其中模板参数包括一个类型 T 和一个类型 U。函数的作用是返回一个只包含参数 val 的常量 U 类型。

接下来的代码定义了一个名为 make_arg 的函数，其中模板参数包括一个类型 Context、一个类型 T 和一个类型 U。函数的作用是返回一个包含参数 val 的 context 类型的值。

这两个函数的作用是帮助用户在代码中更方便地使用 fallback 和 implicit 转换。通过这两个函数，用户可以在使用一个特定的模板形式化时，通过提供一个函数指针来访问另一个特定的模板形式化。在这个例子中，如果用户在一个翻译单元中使用了某个模板形式化，但是没有找到对应的模板元编程件，那么 make_arg 函数将返回一个默认值（也就是 val 的别名）。如果使用了另一种模板形式化，那么 make_arg 函数将尝试使用这个模板来编译 val。


```cpp
template <typename T, typename U> inline const U& check(const U& val) {
  return val;
}

// The type template parameter is there to avoid an ODR violation when using
// a fallback formatter in one translation unit and an implicit conversion in
// another (not recommended).
template <bool IS_PACKED, typename Context, type, typename T,
          FMT_ENABLE_IF(IS_PACKED)>
inline value<Context> make_arg(const T& val) {
  return check<T>(arg_mapper<Context>().map(val));
}

template <bool IS_PACKED, typename Context, type, typename T,
          FMT_ENABLE_IF(!IS_PACKED)>
```

这段代码定义了一个名为dynamic_arg_list的类，它是一个模板类，用于在函数中动态地传递参数。通过const T&和const T& 类型的变量，可以安全地使用模板元编程技巧。

具体来说，这段代码的作用是提供一个可以动态地接收参数的类，可以在需要时动态地创建新节点并将其添加到链表的头部。该类使用了一个特殊类型的模板元编程作为其唯一方法。

类中的模板元编程成员函数push()用于创建一个新的节点，通过将传入的参数赋值给节点的新类型，然后将新节点添加到链表的头部。通过将创建节点的指针存储在head_上，当需要时可以将其转换为引用，以允许使用标准库中的链表操作。

通过使用const T& 和const T& 类型的变量，可以安全地使用模板元编程技巧。例如，可以使用const T& 类型的变量来接收动态传递的整数参数，使用const T& 类型的变量来接收动态传递的字符串参数。

此外，通过is_reference_wrapper模板元编程，可以保证链表中的节点可以被安全地使用，即使它们是引用类型。


```cpp
inline basic_format_arg<Context> make_arg(const T& value) {
  return make_arg<Context>(value);
}

template <typename T> struct is_reference_wrapper : std::false_type {};
template <typename T>
struct is_reference_wrapper<std::reference_wrapper<T>> : std::true_type {};

template <typename T> const T& unwrap(const T& v) { return v; }
template <typename T> const T& unwrap(const std::reference_wrapper<T>& v) {
  return static_cast<const T&>(v);
}

class dynamic_arg_list {
  // Workaround for clang's -Wweak-vtables. Unlike for regular classes, for
  // templates it doesn't complain about inability to deduce single translation
  // unit for placing vtable. So storage_node_base is made a fake template.
  template <typename = void> struct node {
    virtual ~node() = default;
    std::unique_ptr<node<>> next;
  };

  template <typename T> struct typed_node : node<> {
    T value;

    template <typename Arg>
    FMT_CONSTEXPR typed_node(const Arg& arg) : value(arg) {}

    template <typename Char>
    FMT_CONSTEXPR typed_node(const basic_string_view<Char>& arg)
        : value(arg.data(), arg.size()) {}
  };

  std::unique_ptr<node<>> head_;

 public:
  template <typename T, typename Arg> const T& push(const Arg& arg) {
    auto new_node = std::unique_ptr<typed_node<T>>(new typed_node<T>(arg));
    auto& value = new_node->value;
    new_node->next = std::move(head_);
    head_ = std::move(new_node);
    return value;
  }
};
}  // namespace detail

```

This is a C++ class template that defines a `basic_format_context` object. The `basic_format_context` class is used to implement the `OutputIt` iterator, which is an iterator that wraps around and reads characters from the output stream until it reaches the end of the stream.

The `basic_format_context` class has a few member variables, including an `OutputIt` object called `out_`, a `basic_format_args<basic_format_context>` object called `args_`, and a `detail::locale_ref` object called `loc_`. The `args_` object is used to access the arguments of the formatting, and the `loc_` object is used to store the current locale for formatting.

The class has a constructor that takes three arguments:

1. The first argument is an `OutputIt` object that represents the output stream.
2. The second argument is a `basic_format_args<basic_format_context>` object that contains the formatting arguments.
3. The third argument is a `detail::locale_ref` object that stores the current locale for formatting.

The class has a member function operator= that compares two `basic_format_context` objects, and a member function operator=(const `basic_format_context&`) that compares two `basic_format_context` objects and assigns the smaller one to the `out_` member.

The class also has two member functions that are used to access the arguments of the formatting: `arg()` and `arg()`. These functions are used to retrieve the current formatting argument from the `args_` object, and they return the corresponding format code and arguments.

Finally, the class has a member function `error_handler()` that is used to handle errors that occur during the formatting process.


```cpp
// Formatting context.
template <typename OutputIt, typename Char> class basic_format_context {
 public:
  /** The character type for the output. */
  using char_type = Char;

 private:
  OutputIt out_;
  basic_format_args<basic_format_context> args_;
  detail::locale_ref loc_;

 public:
  using iterator = OutputIt;
  using format_arg = basic_format_arg<basic_format_context>;
  using parse_context_type = basic_format_parse_context<Char>;
  template <typename T> using formatter_type = formatter<T, char_type>;

  basic_format_context(const basic_format_context&) = delete;
  void operator=(const basic_format_context&) = delete;
  /**
   Constructs a ``basic_format_context`` object. References to the arguments are
   stored in the object so make sure they have appropriate lifetimes.
   */
  basic_format_context(OutputIt out,
                       basic_format_args<basic_format_context> ctx_args,
                       detail::locale_ref loc = detail::locale_ref())
      : out_(out), args_(ctx_args), loc_(loc) {}

  format_arg arg(int id) const { return args_.get(id); }
  format_arg arg(basic_string_view<char_type> name) { return args_.get(name); }
  int arg_id(basic_string_view<char_type> name) { return args_.get_id(name); }
  const basic_format_args<basic_format_context>& args() const { return args_; }

  detail::error_handler error_handler() { return {}; }
  void on_error(const char* message) { error_handler().on_error(message); }

  // Returns an iterator to the beginning of the output range.
  iterator out() { return out_; }

  // Advances the begin iterator to ``it``.
  void advance_to(iterator it) {
    if (!detail::is_back_insert_iterator<iterator>()) out_ = it;
  }

  detail::locale_ref locale() { return loc_; }
};

```

这段代码定义了一个名为“buffer\_context”的模板类，该类使用了一种特殊的格式化方式来 append字符到缓冲区中。

具体来说，该模板类接受两个参数：一个“basic\_format\_context”和一个“detail\_buffer\_appender”，其中“basic\_format\_context”是一个更一般的模板类，用于在格式化时添加字符到缓冲区中，“detail\_buffer\_appender”是一个具体的实现，用于将模板类中的“Char”类型转换成实际的缓冲区字符。

接着，该代码定义了一个名为“format\_context”的模板类，该类使用了“buffer\_context”模板类，并将其参数设置为“char”类型。

最后，该代码定义了一个名为“wformat\_context”的模板类，该类使用了“buffer\_context”模板类，并将其参数设置为“wchar\_t”类型。

这里有一个额外的说明：在C++中，有一些库函数使用了“ fmt::”前缀，例如“fmt::vformat”。这些函数需要一个“format\_args”参数，该参数是一个包含格式化arguments的元组，可以使用“FMT_BUFFER\_CONTEXT(Char)”定义一个别名“fmt::basic\_format\_args”来避免在编译时解析器的困惑。


```cpp
template <typename Char>
using buffer_context =
    basic_format_context<detail::buffer_appender<Char>, Char>;
using format_context = buffer_context<char>;
using wformat_context = buffer_context<wchar_t>;

// Workaround an alias issue: https://stackoverflow.com/q/62767544/471164.
#define FMT_BUFFER_CONTEXT(Char) \
  basic_format_context<detail::buffer_appender<Char>, Char>

/**
  \rst
  An array of references to arguments. It can be implicitly converted into
  `~fmt::basic_format_args` for passing into type-erased formatting functions
  such as `~fmt::vformat`.
  \endrst
 */
```

这段代码定义了一个模板类 called `format_arg_store`，用于格式化字符串中的参数。

这个模板类接受一个或多个参数，参数可以是类型参数或用户提供的整数参数。在函数声明中，我们声明了 `Context` 和 `Args` 两个类型参数，其中 `Context` 是模板参数，而 `Args` 是用户提供的参数。

如果 FMT_GCC_VERSION 与 FMT_GCC_VERSION 等于 409，那么在函数声明之前我们添加了一行代码来修复 GCC 模板参数替换的错误。

在 `private` 部分，我们定义了一个静态变量 `num_args`，用于跟踪参数的数量，以及一个静态变量 `num_named_args`，用于跟踪给定的参数中命名参数的数量。我们还定义了一个变量 `is_packed`，根据参数数量判断是否是打包格式。

在 `using value_type` 语句中，我们根据 `is_packed` 和 `detail::value<Context>` 判断来确定输入的参数是基本格式化还是存储格式化，如果是基本格式化，我们创建了一个名为 `detail::value<Context>` 的类型别名。

在 `basic_format_args<Context>` 类中，我们实现了 `basic_format_args` 的默合近地络近地步把 `Context` 。

最后，在 `format_arg_store` 的构造函数中，我们实现了 `basic_format_args` 的第一部分，即对输入参数进行格式化。


```cpp
template <typename Context, typename... Args>
class format_arg_store
#if FMT_GCC_VERSION && FMT_GCC_VERSION < 409
    // Workaround a GCC template argument substitution bug.
    : public basic_format_args<Context>
#endif
{
 private:
  static const size_t num_args = sizeof...(Args);
  static const size_t num_named_args = detail::count_named_args<Args...>();
  static const bool is_packed = num_args <= detail::max_packed_args;

  using value_type = conditional_t<is_packed, detail::value<Context>,
                                   basic_format_arg<Context>>;

  detail::arg_data<value_type, typename Context::char_type, num_args,
                   num_named_args>
      data_;

  friend class basic_format_args<Context>;

  static constexpr unsigned long long desc =
      (is_packed ? detail::encode_types<Context, Args...>()
                 : detail::is_unpacked_bit | num_args) |
      (num_named_args != 0
           ? static_cast<unsigned long long>(detail::has_named_args_bit)
           : 0);

 public:
  format_arg_store(const Args&... args)
      :
```

这段代码定义了一个名为 `fmt::format_arg_store` 的类，用于在 `fmt::format_args` 的模板中存储和传递 arguments。

首先，它检查 GCC 编译器的版本是否为 409 之前，如果是，那么执行下面的语句：
```cppbash
basic_format_args<Context>(*this),
```
这将调用 `basic_format_args` 函数，获取当前 `this` 对象（可能是类的实例）的引用，并将其传递给 `basic_format_args` 函数。这个函数将获取 `fmt::format_args` 的头信息，然后解析出 arguments，最后输出它们。

如果是 GCC 编译器的版本为 409 之后，那么跳过下面的语句：
```cpppython
}
```
这个语句将 `fmt::format_arg_store` 的对象成员保持为 `null`，不输出任何内容。

接下来，定义了 `fmt::format_arg_store` 类的第二个参数，它是 `data_` 类型的参数，它的形式是通过 `is_packed` 和 `Context` 来限制的。它包含了一系列 argument，通过 `make_arg` 函数从 `args` 数组中提取出来，其中 `args...` 表示参数的迭代，`detail::mapped_type_constant<Args, Context>::value` 表示提取出参数的值。然后，它调用 `detail::init_named_args` 函数，传递给它的 named_args 数组和提取出的 arguments 数组，初始化 named_args 数组。

综上所述，这段代码创建了一个 `fmt::format_arg_store` 对象，用于存储 `fmt::format_args` 的 argument，可以用于将类的实例作为 arguments 传递给 `fmt::format_args`。`Context` 参数可用于指定默认的 context，`args...` 可以作为参数传递给 `make_arg` 函数来提取 arguments。


```cpp
#if FMT_GCC_VERSION && FMT_GCC_VERSION < 409
        basic_format_args<Context>(*this),
#endif
        data_{detail::make_arg<
            is_packed, Context,
            detail::mapped_type_constant<Args, Context>::value>(args)...} {
    detail::init_named_args(data_.named_args(), 0, 0, args...);
  }
};

/**
  \rst
  Constructs a `~fmt::format_arg_store` object that contains references to
  arguments and can be implicitly converted to `~fmt::format_args`. `Context`
  can be omitted in which case it defaults to `~fmt::context`.
  See `~fmt::arg` for lifetime considerations.
  \endrst
 */
```

这段代码定义了一个模板类，名为`format_arg_store`，模板参数为`Context`和`Args`，其中`Args`是一个可变参数类型，通过参数列表来指定模板实体的具体值。

具体来说，这段代码的作用是定义了一个可以将`Args`中的各个元素存储到一个`Context`类型的对象中的函数，并返回该对象。这个函数在模板定义中可以被隐式地使用，因此在编译时检查`format_str`的语法正确性。如果`format_str`是一个编译时可执行字符串，则编译器会检查它是否是一个有效的字符串，否则在编译时会报错。

举个例子，如果你在编译时使用`fmt::format_arg_store`，你可能会这样写：
```cppcss
int main() {
   int a = 10;
   double b = 3.14;
   fmt::format_arg_store<int, double> obj{a, b};
   fmt::format_arg_store<int, double> obj2{0, 0};
   fmt::format_arg_store<int, double> obj3{{0, 0}, 1000.0};
   fmt::format_arg_store<int, double> obj4{1, 1.1};
   fmt::format_arg_store<int, double> obj5{{10, 0}, 3600.0};
   fmt::format_arg_store<int, double> obj6{0, 0.1};
   fmt::format_arg_store<int, double> obj7{{0, 0.2}, 2600.0};
   fmt::format_arg_store<int, double> obj8{1, 2.5};
   fmt::format_arg_store<int, double> obj9{{1, 3.0}, 4200.0};
   fmt::format_arg_store<int, double> obj10{{1, 3.5}, 5200.0};
   fmt::format_arg_store<int, double> obj11{{1, 4.0}, 6200.0};
   fmt::format_arg_store<int, double> obj12{{1, 4.5}, 7200.0};
   fmt::format_arg_store<int, double> obj13{{1, 5.0}, 8200.0};
   fmt::format_arg_store<int, double> obj14{{1, 5.5}, 9200.0};
   fmt::format_arg_store<int, double> obj15{{1, 6.0}, 10200.0};
   fmt::format_arg_store<int, double> obj16{{1, 6.5}, 11200.0};
   fmt::format_arg_store<int, double> obj17{{1, 7.0}, 12200.0};
   fmt::format_arg_store<int, double> obj18{{1, 7.5}, 13200.0};
   fmt::format_arg_store<int, double> obj19{{1, 8.0}, 14200.0};
   fmt::format_arg_store<int, double> obj20{{1, 8.5}, 15200.0};
   fmt::format_arg_store<int, double> obj21{{1, 9.0}, 16200.0};
   fmt::format_arg_store<int, double> obj22{{1, 9.5}, 17200.0};
   fmt::format_arg_store<int, double> obj23{{1, 10.0}, 18200.0};
   fmt::format_arg_store<int, double> obj24{{1, 10.5}, 19200.0};
   fmt::format_arg_store<int, double> obj25{{1, 11.0}, 20200.0};
   fmt::format_arg_store<int, double> obj26{{1, 11.5}, 21200.0};
   fmt::format_arg_store<int, double> obj27{{1, 12.0}, 22200.0};
   fmt::format_arg_store<int, double> obj28{{1, 12.5}, 23200.0};
   fmt::format_arg_store<int, double> obj29{{1, 13.0}, 24200.0};
   fmt::format_arg_store<int, double> obj30{{1, 13.5}, 25200.0};
   fmt::format_arg_store<int, double> obj31{{1, 14.0}, 26200.0};
   fmt::format_arg_store<int, double> obj32{{1, 14.5}, 27200.0};
   fmt::format_arg_store<int, double> obj33{{1, 15.0}, 28200.0};
   fmt::format_arg_store<int, double> obj34{{1, 15.5}, 29200.0};
   fmt::format_arg_store<int, double> obj35{{1, 16.0}, 30200.0};
   fmt::format_arg_store<int, double> obj36{{1, 16.5}, 31200.0};
   fmt::format_arg_store<int, double> obj37{{1, 17.0}, 32200.0};
   fmt::format_arg_store<int, double> obj38{{1, 17.5}, 33200.0};
   fmt::format_arg_store<int, double> obj39{{1, 18.0}, 34200.0};
   fmt::format_arg_store<int, double> obj40{{1, 18.5}, 35200.0};
   fmt::format_arg_store<int, double> obj41{{1, 19.0}, 36200.0};
   fmt::format_arg_store<int, double> obj42{{1, 19.5}, 37200.0};
   fmt::format_arg_store<int, double> obj43{{1, 20.0}, 38200.0};
   fmt::format_arg_store<int, double> obj44{{1, 20.5}, 39200.0};
   fmt::format_arg_store<int, double> obj45{{1, 21.0}, 40200.0};
   fmt::format_arg_store<int, double> obj46{{1, 21.5}, 41200.0};
   fmt::format_arg_store<int, double> obj47{{1, 22.0}, 42200.0};
   fmt::format_arg_store<int, double> obj48{{1, 22.5}, 43200.0};
   fmt::format_arg_store<int, double> obj49{{1, 23.0}, 44200.0};
   fmt::format_arg_store<int, double> obj50{{1, 23.5}, 45200.0};
   fmt::format_


```
template <typename Context = format_context, typename... Args>
inline format_arg_store<Context, Args...> make_format_args(
    const Args&... args) {
  return {args...};
}

/**
  \rst
  Constructs a `~fmt::format_arg_store` object that contains references
  to arguments and can be implicitly converted to `~fmt::format_args`.
  If ``format_str`` is a compile-time string then `make_args_checked` checks
  its validity at compile time.
  \endrst
 */
template <typename... Args, typename S, typename Char = char_t<S>>
```cpp

这段代码定义了一个名为 `make_args_checked` 的函数，它接受一个格式字符串 `format_str` 和多个参数 `args`。

该函数首先检查参数 `args` 是否为引用类型（即 `remove_reference_t` 类）。如果是，则禁止将引用类型作为 lvalue（即格式字符串中的参数）传递给 `std::is_base_of<detail::view, remove_reference_t<Args>>`，否则允许。

接着，函数调用 `detail::check_format_string<Args...>` 来处理 `format_str`，将 `args`...（多个输入参数）存储到 `format_arg_store<buffer_context<Char>, remove_reference_t<Args>...>`（返回）中。

最后，该函数返回一个命名参数，用于在格式化函数中使用。由于该函数的参数和返回值都是引用类型，因此它被限制只能作为函数调用的实参，而不能被用作其他类型的参数。


```
inline auto make_args_checked(const S& format_str,
                              const remove_reference_t<Args>&... args)
    -> format_arg_store<buffer_context<Char>, remove_reference_t<Args>...> {
  static_assert(
      detail::count<(
              std::is_base_of<detail::view, remove_reference_t<Args>>::value &&
              std::is_reference<Args>::value)...>() == 0,
      "passing views as lvalues is disallowed");
  detail::check_format_string<Args...>(format_str);
  return {args...};
}

/**
  \rst
  Returns a named argument to be used in a formatting function. It should only
  be used in a call to a formatting function.

  **Example**::

    fmt::print("Elapsed time: {s:.2f} seconds", fmt::arg("s", 1.23));
  \endrst
 */
```cpp

这段代码定义了一个模板类 named_arg，其中 template 参数有两个类型参数 Char 和 T。该模板类在函数参数中使用 named_arg 修饰符，用于实现 named 参数的定义和使用。

arg() 函数接收两个参数，一个是字符串类型的参数 name，另一个是整型参数 arg。函数内部使用 named_arg 修饰符实现的 named 参数定义，会根据传入的参数名称返回一个模板对象，该对象包含 name 和 arg 两个成员变量。

该代码的作用是定义一个模板类 named_arg，用于实现格式化字符串中的 named 参数。通过传入不同的模板参数，可以实现对不同类型参数的 named 参数定义。在函数内部，使用 named_arg 修饰符实现的 named 参数定义，可以方便地用于格式化字符串中的 named 参数，而不需要显式地传入多个参数。


```
template <typename Char, typename T>
inline detail::named_arg<Char, T> arg(const Char* name, const T& arg) {
  static_assert(!detail::is_named_arg<T>(), "nested named arguments");
  return {name, arg};
}

/**
  \rst
  A dynamic version of `fmt::format_arg_store`.
  It's equipped with a storage to potentially temporary objects which lifetimes
  could be shorter than the format arguments object.

  It can be implicitly converted into `~fmt::basic_format_args` for passing
  into type-erased formatting functions such as `~fmt::vformat`.
  \endrst
 */
```cpp

This is a C++ template class for dynamic memory allocation that includes functionality to handle dynamic arguments and named arguments.

The `push_back` function is a member function of the `push_back` class that takes a single template argument of a type T and returns a reference to the function that should be called on the fly.

The function template includes a helper function `push_back<T>` that takes a single template argument of a type T and emplaces it into the dynamic memory allocation store. The named argument helper function is defined using the `named_arg` template parameter.

The `push_back` function can handle both named and unamed arguments.

The class also includes functionality to clear the dynamic memory allocation store and to reserve space for new dynamic memory allocation.

Note that this class is a metaprogramming h锦绣大药中方dfsdfgxyz78901234567890------


```
template <typename Context>
class dynamic_format_arg_store
#if FMT_GCC_VERSION && FMT_GCC_VERSION < 409
    // Workaround a GCC template argument substitution bug.
    : public basic_format_args<Context>
#endif
{
 private:
  using char_type = typename Context::char_type;

  template <typename T> struct need_copy {
    static constexpr detail::type mapped_type =
        detail::mapped_type_constant<T, Context>::value;

    enum {
      value = !(detail::is_reference_wrapper<T>::value ||
                std::is_same<T, basic_string_view<char_type>>::value ||
                std::is_same<T, detail::std_string_view<char_type>>::value ||
                (mapped_type != detail::type::cstring_type &&
                 mapped_type != detail::type::string_type &&
                 mapped_type != detail::type::custom_type))
    };
  };

  template <typename T>
  using stored_type = conditional_t<detail::is_string<T>::value,
                                    std::basic_string<char_type>, T>;

  // Storage of basic_format_arg must be contiguous.
  std::vector<basic_format_arg<Context>> data_;
  std::vector<detail::named_arg_info<char_type>> named_info_;

  // Storage of arguments not fitting into basic_format_arg must grow
  // without relocation because items in data_ refer to it.
  detail::dynamic_arg_list dynamic_args_;

  friend class basic_format_args<Context>;

  unsigned long long get_types() const {
    return detail::is_unpacked_bit | data_.size() |
           (named_info_.empty()
                ? 0ULL
                : static_cast<unsigned long long>(detail::has_named_args_bit));
  }

  const basic_format_arg<Context>* data() const {
    return named_info_.empty() ? data_.data() : data_.data() + 1;
  }

  template <typename T> void emplace_arg(const T& arg) {
    data_.emplace_back(detail::make_arg<Context>(arg));
  }

  template <typename T>
  void emplace_arg(const detail::named_arg<char_type, T>& arg) {
    if (named_info_.empty()) {
      constexpr const detail::named_arg_info<char_type>* zero_ptr{nullptr};
      data_.insert(data_.begin(), {zero_ptr, 0});
    }
    data_.emplace_back(detail::make_arg<Context>(detail::unwrap(arg.value)));
    auto pop_one = [](std::vector<basic_format_arg<Context>>* data) {
      data->pop_back();
    };
    std::unique_ptr<std::vector<basic_format_arg<Context>>, decltype(pop_one)>
        guard{&data_, pop_one};
    named_info_.push_back({arg.name, static_cast<int>(data_.size() - 2u)});
    data_[0].value_.named_args = {named_info_.data(), named_info_.size()};
    guard.release();
  }

 public:
  /**
    \rst
    Adds an argument into the dynamic store for later passing to a formatting
    function.

    Note that custom types and string types (but not string views) are copied
    into the store dynamically allocating memory if necessary.

    **Example**::

      fmt::dynamic_format_arg_store<fmt::format_context> store;
      store.push_back(42);
      store.push_back("abc");
      store.push_back(1.5f);
      std::string result = fmt::vformat("{} and {} and {}", store);
    \endrst
  */
  template <typename T> void push_back(const T& arg) {
    if (detail::const_check(need_copy<T>::value))
      emplace_arg(dynamic_args_.push<stored_type<T>>(arg));
    else
      emplace_arg(detail::unwrap(arg));
  }

  /**
    \rst
    Adds a reference to the argument into the dynamic store for later passing to
    a formatting function. Supports named arguments wrapped in
    ``std::reference_wrapper`` via ``std::ref()``/``std::cref()``.

    **Example**::

      fmt::dynamic_format_arg_store<fmt::format_context> store;
      char str[] = "1234567890";
      store.push_back(std::cref(str));
      int a1_val{42};
      auto a1 = fmt::arg("a1_", a1_val);
      store.push_back(std::cref(a1));

      // Changing str affects the output but only for string and custom types.
      str[0] = 'X';

      std::string result = fmt::vformat("{} and {a1_}");
      assert(result == "X234567890 and 42");
    \endrst
  */
  template <typename T> void push_back(std::reference_wrapper<T> arg) {
    static_assert(
        detail::is_named_arg<typename std::remove_cv<T>::type>::value ||
            need_copy<T>::value,
        "objects of built-in types and string views are always copied");
    emplace_arg(arg.get());
  }

  /**
    Adds named argument into the dynamic store for later passing to a formatting
    function. ``std::reference_wrapper`` is supported to avoid copying of the
    argument.
  */
  template <typename T>
  void push_back(const detail::named_arg<char_type, T>& arg) {
    const char_type* arg_name =
        dynamic_args_.push<std::basic_string<char_type>>(arg.name).c_str();
    if (detail::const_check(need_copy<T>::value)) {
      emplace_arg(
          fmt::arg(arg_name, dynamic_args_.push<stored_type<T>>(arg.value)));
    } else {
      emplace_arg(fmt::arg(arg_name, arg.value));
    }
  }

  /** Erase all elements from the store */
  void clear() {
    data_.clear();
    named_info_.clear();
    dynamic_args_ = detail::dynamic_arg_list();
  }

  /**
    \rst
    Reserves space to store at least *new_cap* arguments including
    *new_cap_named* named arguments.
    \endrst
  */
  void reserve(size_t new_cap, size_t new_cap_named) {
    FMT_ASSERT(new_cap >= new_cap_named,
               "Set of arguments includes set of named arguments");
    data_.reserve(new_cap);
    named_info_.reserve(new_cap_named);
  }
};

```cpp

This is a C++ object that represents a `basic_format_args` object. It has two member functions, `basic_format_args` and `get`.

The `basic_format_args` function takes a dynamic set of `format_arg` arguments and a count of the number of arguments. It creates an instance of the `basic_format_args` object and returns it.

The `get` function takes a `format_arg` argument and returns the argument with the specified index from the `args_` array. If the index is outside the bounds of the `args_` array, the function returns the default `format_arg` object.

The `is_packed` function returns true if `args_` is a `std::vector` of `basic_format_args` objects, and false otherwise.

The `desc_` function returns the `desc` member function of the `basic_format_args` object.


```
/**
  \rst
  A view of a collection of formatting arguments. To avoid lifetime issues it
  should only be used as a parameter type in type-erased functions such as
  ``vformat``::

    void vlog(string_view format_str, format_args args);  // OK
    format_args args = make_format_args(42);  // Error: dangling reference
  \endrst
 */
template <typename Context> class basic_format_args {
 public:
  using size_type = int;
  using format_arg = basic_format_arg<Context>;

 private:
  // A descriptor that contains information about formatting arguments.
  // If the number of arguments is less or equal to max_packed_args then
  // argument types are passed in the descriptor. This reduces binary code size
  // per formatting function call.
  unsigned long long desc_;
  union {
    // If is_packed() returns true then argument values are stored in values_;
    // otherwise they are stored in args_. This is done to improve cache
    // locality and reduce compiled code size since storing larger objects
    // may require more code (at least on x86-64) even if the same amount of
    // data is actually copied to stack. It saves ~10% on the bloat test.
    const detail::value<Context>* values_;
    const format_arg* args_;
  };

  bool is_packed() const { return (desc_ & detail::is_unpacked_bit) == 0; }
  bool has_named_args() const {
    return (desc_ & detail::has_named_args_bit) != 0;
  }

  detail::type type(int index) const {
    int shift = index * detail::packed_arg_bits;
    unsigned int mask = (1 << detail::packed_arg_bits) - 1;
    return static_cast<detail::type>((desc_ >> shift) & mask);
  }

  basic_format_args(unsigned long long desc,
                    const detail::value<Context>* values)
      : desc_(desc), values_(values) {}
  basic_format_args(unsigned long long desc, const format_arg* args)
      : desc_(desc), args_(args) {}

 public:
  basic_format_args() : desc_(0) {}

  /**
   \rst
   Constructs a `basic_format_args` object from `~fmt::format_arg_store`.
   \endrst
   */
  template <typename... Args>
  FMT_INLINE basic_format_args(const format_arg_store<Context, Args...>& store)
      : basic_format_args(store.desc, store.data_.args()) {}

  /**
   \rst
   Constructs a `basic_format_args` object from
   `~fmt::dynamic_format_arg_store`.
   \endrst
   */
  FMT_INLINE basic_format_args(const dynamic_format_arg_store<Context>& store)
      : basic_format_args(store.get_types(), store.data()) {}

  /**
   \rst
   Constructs a `basic_format_args` object from a dynamic set of arguments.
   \endrst
   */
  basic_format_args(const format_arg* args, int count)
      : basic_format_args(detail::is_unpacked_bit | detail::to_unsigned(count),
                          args) {}

  /** Returns the argument with the specified id. */
  format_arg get(int id) const {
    format_arg arg;
    if (!is_packed()) {
      if (id < max_size()) arg = args_[id];
      return arg;
    }
    if (id >= detail::max_packed_args) return arg;
    arg.type_ = type(id);
    if (arg.type_ == detail::type::none_type) return arg;
    arg.value_ = values_[id];
    return arg;
  }

  template <typename Char> format_arg get(basic_string_view<Char> name) const {
    int id = get_id(name);
    return id >= 0 ? get(id) : format_arg();
  }

  template <typename Char> int get_id(basic_string_view<Char> name) const {
    if (!has_named_args()) return -1;
    const auto& named_args =
        (is_packed() ? values_[-1] : args_[-1].value_).named_args;
    for (size_t i = 0; i < named_args.size; ++i) {
      if (named_args.data[i].name == name) return named_args.data[i].id;
    }
    return -1;
  }

  int max_size() const {
    unsigned long long max_packed = detail::max_packed_args;
    return static_cast<int>(is_packed() ? max_packed
                                        : desc_ & ~detail::is_unpacked_bit);
  }
};

```cpp

这段代码定义了两种结构体：format_args和wformat_args。它们都继承自basic_format_args<format_context>类型。

这两颗结构体的作用是定义了格式化字符串时要使用的模板参数及其类型。其中，template <typename... Args> 的作用是定义了一个模板参数，这个参数是一个或多个Args类型的参数。通过这个模板参数，可以很方便地将format_args和wformat_args的成员函数与basic_format_args<format_context>的成员函数进行别名绑定，使得代码更加易读。

另外，这两颗结构体还定义了一个通用的函数：vformat。这个函数接受一个basic_string<Char>类型的格式化字符串view，以及一个basic_format_args<buffer_context<type_identity_t<Char>>类型的参数args。通过这个函数，可以方便地格式化字符串，并返回格式化后的结果。


```
/** An alias to ``basic_format_args<context>``. */
// It is a separate type rather than an alias to make symbols readable.
struct format_args : basic_format_args<format_context> {
  template <typename... Args>
  FMT_INLINE format_args(const Args&... args) : basic_format_args(args...) {}
};
struct wformat_args : basic_format_args<wformat_context> {
  using basic_format_args::basic_format_args;
};

namespace detail {

template <typename Char, FMT_ENABLE_IF(!std::is_same<Char, char>::value)>
std::basic_string<Char> vformat(
    basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args);

```cpp

这段代码定义了一个名为FMT_API的函数，它接受一个字符串格式字符串和一个格式参数列表。它执行的操作是将格式字符串和格式参数列表应用于给定的缓冲区，并返回结果。它是在C++标准库中，尤其是<format> header文件中定义的。

具体来说，这段代码的实现方式如下：

1. 首先定义了一个名为vformat_to的函数模板类，它接受一个缓冲区和一个格式字符串作为参数，并使用basic_string_view模板来提供对格式字符串的访问。它的实现基本上就是对这个函数的定义，只是将函数实现过程中的细节留给了用户自己。

2. 接下来定义了一个名为vprint_mojibake的函数，它接受一个文件对象、一个格式字符串和一个格式参数列表作为参数。它基本上就是对这个函数的定义，只是在没有支持函数的情况下，提供了一个临时实现，以允许用户在需要时使用它。

3. 最后在FMT_API定义中，明确说明了vprint_mojibake是这个命名空间中唯一直接使用函数，这意味着任何需要使用它的地方，都必须使用这个函数，而不能通过其他途径实现。


```
FMT_API std::string vformat(string_view format_str, format_args args);

template <typename Char>
buffer_appender<Char> vformat_to(
    buffer<Char>& buf, basic_string_view<Char> format_str,
    basic_format_args<FMT_BUFFER_CONTEXT(type_identity_t<Char>)> args);

template <typename Char, typename Args,
          FMT_ENABLE_IF(!std::is_same<Char, char>::value)>
inline void vprint_mojibake(std::FILE*, basic_string_view<Char>, const Args&) {}

FMT_API void vprint_mojibake(std::FILE*, string_view, format_args);
#ifndef _WIN32
inline void vprint_mojibake(std::FILE*, string_view, format_args) {}
#endif
}  // namespace detail

```cpp

这段代码定义了一个名为`vformat_to`的函数，它接受一个输出迭代器（`OutputIt`）和一个格式字符串，并输出到一个名为`out`的输出迭代器上。

函数内部，首先定义了一个名为`buf`的变量，它是通过`detail::get_buffer<Char>`函数得到的，这个函数将在`vformat_to`函数中用来初始化输出缓冲区。然后，通过调用`detail::vformat_to`函数，将格式字符串和 arguments 格式化并写入缓冲区。最后，通过`detail::get_iterator`函数获取缓冲区的结尾迭代器，并将其返回。

该函数的实现基于两个条件：

1. 如果 `std::back_inserter<Container>` 是一个输出迭代器，则无法使用 `vformat_to<ArgFormatter>(...)` 这个 overload。
2. 如果 `detail::is_output_iterator<OutputIt>` 的值为 `true`，则 `vformat_to` 函数将被视为对 `std::back_inserter<Container>` 和 `fmt::format_to` 的组合。


```
/** Formats a string and writes the output to ``out``. */
// GCC 8 and earlier cannot handle std::back_insert_iterator<Container> with
// vformat_to<ArgFormatter>(...) overload, so SFINAE on iterator type instead.
template <typename OutputIt, typename S, typename Char = char_t<S>,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value)>
OutputIt vformat_to(
    OutputIt out, const S& format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  decltype(detail::get_buffer<Char>(out)) buf(detail::get_buffer_init(out));
  detail::vformat_to(buf, to_string_view(format_str), args);
  return detail::get_iterator(buf);
}

/**
 \rst
 Formats arguments, writes the result to the output iterator ``out`` and returns
 the iterator past the end of the output range.

 **Example**::

   std::vector<char> out;
   fmt::format_to(std::back_inserter(out), "{}", 42);
 \endrst
 */
```cpp

这段代码定义了一个模板结构体 `format_to_n_result`，其中 `template` 被分成了四个参数：`OutputIt`、`S`、`Args` 和一个 `...` 参数列表。这个模板结构体旨在通过 `format_to` 函数将一个字符串 `format_str` 和一系列 `Args` 参数连接起来并输出到 `OutputIt` 类型的结果。

进一步地，这个模板结构体在 `format_to_n_result` 函数中实现了这个目的。函数的第一个参数是一个 `OutputIt` 类型的变量 `out`，第二个参数是一个字符串 `format_str`，第三个参数是一个 `Args` 参数列表，最后一个参数是一个形式化参数 `...`。这个函数将字符串 `format_str` 和 `Args` 参数连接起来，并将它们传递给 `format_to` 函数，最后将结果赋值给 `out` 变量。

这里 `FMT_ENABLE_IF` 是一个预处理指令，它会检查 `is_output_iterator<OutputIt>::value` 和 `detail::is_string<S>::value` 是否为真。如果是，它就会启用 `is_output_iterator` 的功能，否则就会跳过这个检查，输出 `OutputIt` 类型的默认值，也就是 `void` 类型。


```
template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value&&
                            detail::is_string<S>::value)>
inline OutputIt format_to(OutputIt out, const S& format_str, Args&&... args) {
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  return vformat_to(out, to_string_view(format_str), vargs);
}

template <typename OutputIt> struct format_to_n_result {
  /** Iterator past the end of the output range. */
  OutputIt out;
  /** Total (not truncated) output size. */
  size_t size;
};

```cpp

这段代码定义了一个模板函数，名为`vformat_to_n`。，它接受三个参数：

1. `Args`：一个可变参数，表示要格式化的字符串中的元素个数。
2. `OutputIt`：一个输出迭代器，表示要输出给定的输出字符串类型。
3. `Char`：一个字符类型，表示输出字符集中的字符类型。

函数内部使用了`FMT_ENABLE_IF`进行条件编译，即只有在`OutputIt`输出迭代器有值时才会执行后续的代码。

具体实现过程如下：

1. 首先定义了一个`detail::iterator_buffer`类型的变量`buf`，它用于存储格式化后的结果字符串，该类型实现了`basic_iterator<OutputIt>`和`basic_string_view<Char>`两个接口。
2. 在`vformat_to_n`函数中，调用了`detail::vformat_to`函数，该函数将`format_str`和`args`作为参数，并将结果存储到`buf`中。
3. 调用`buf.out()`获取`buf`中存储的字符串，并获取字符串的长度，也就是`buf.count()`。
4. 返回`buf.out()`和`buf.count()`，即格式化后的字符串输出结果和剩余的输出字符数。

该函数的作用是将输入的`Args`和`OutputIt`参数，按照指定的`fmt_enable_if`语句中的条件进行格式化，并输出到指定的`OutputIt`输出迭代器中。


```
template <typename OutputIt, typename Char, typename... Args,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value)>
inline format_to_n_result<OutputIt> vformat_to_n(
    OutputIt out, size_t n, basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  detail::iterator_buffer<OutputIt, Char, detail::fixed_buffer_traits> buf(out,
                                                                           n);
  detail::vformat_to(buf, format_str, args);
  return {buf.out(), buf.count()};
}

/**
 \rst
 Formats arguments, writes up to ``n`` characters of the result to the output
 iterator ``out`` and returns the total output size and the iterator past the
 end of the output range.
 \endrst
 */
```cpp

这段代码定义了一个模板函数名为`format_to_n`，它接受三个模板参数：`OutputIt`表示输出的类型，`S`表示输入的类型，`Args`表示输入的参数列表。

第一个模板参数`FMT_ENABLE_IF(detail::is_string<S>::value&&
                                   detail::is_output_iterator<OutputIt>::value)`用于指定`OutputIt`是否是输出迭代器，如果是，则第二个模板参数`size_t n`表示输出中字符的数量。

第二个模板参数`const S& format_str`表示输入的格式字符串，`const Args&... args`表示输入参数的列表，这些参数在格式字符串中使用符号`<`和参数名称之间用空格分隔。

该函数使用`vformat_to_n`函数将格式字符串中的格式字符转换为输出迭代器，然后使用`fmt::make_args_checked`函数将输入参数的列表连接到输出迭代器上，最后将结果返回。


```
template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_string<S>::value&&
                            detail::is_output_iterator<OutputIt>::value)>
inline format_to_n_result<OutputIt> format_to_n(OutputIt out, size_t n,
                                                const S& format_str,
                                                const Args&... args) {
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  return vformat_to_n(out, n, to_string_view(format_str), vargs);
}

/**
  Returns the number of characters in the output of
  ``format(format_str, args...)``.
 */
template <typename... Args>
```cpp

此代码定义了两个函数，`inline size_t formatted_size()` 和 `inline std::basic_string<char_t<char>> vformat()`。这两个函数用于格式化字符串，并返回格式化后的字符串。

函数`inline size_t formatted_size()`接受一个`string_view`类型的格式字符串和一个可变参数数组`Args...`，它将格式化字符串中的格式字符串，并返回格式化后的字符串所占用的字节数。

函数`inline std::basic_string<char_t<char>> vformat()`接受一个字符串格式字符串和一个可变参数数组`Args...`，它将解析参数，根据格式字符串中的占位符，格式化字符串，并返回被格式化的字符串。

这两个函数的实现都涉及到底部：:，它是一个C++标准库中的一个独立的用于格式化和编辑辅助设施的库。通过这些函数，用户可以轻松地格式化和编辑字符串，而不需要显式地编写大量复杂的代码来处理它们。


```
inline size_t formatted_size(string_view format_str, Args&&... args) {
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  detail::counting_buffer<> buf;
  detail::vformat_to(buf, format_str, vargs);
  return buf.count();
}

template <typename S, typename Char = char_t<S>>
FMT_INLINE std::basic_string<Char> vformat(
    const S& format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  return detail::vformat(to_string_view(format_str), args);
}

/**
  \rst
  Formats arguments and returns the result as a string.

  **Example**::

    #include <fmt/core.h>
    std::string message = fmt::format("The answer is {}", 42);
  \endrst
```cpp

这段代码定义了一个名为`format`的函数模板，其中`template`使用`const`下一对星号`<...>`来表示该模板可以带任何参数。`template`中，通过使用`std::basic_string<char_t<S>>`作为模板参数，将其转换为`char_t<S>`。这个函数的作用是格式化字符串`format_str`，`format_args`是一个参数列表，通过这些参数，可以控制`format`函数的输出。

该代码中还定义了两个函数`vprint`，它们分别接受一个`string_view`和一个`format_args`作为参数，并输出相应的字符串。这两个函数实际就是`fmt::print`函数的别名，因为它们也接受相同的参数，只是输出的方式不同。


```
*/
// Pass char_t as a default template parameter instead of using
// std::basic_string<char_t<S>> to reduce the symbol size.
template <typename S, typename... Args, typename Char = char_t<S>>
FMT_INLINE std::basic_string<Char> format(const S& format_str, Args&&... args) {
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  return detail::vformat(to_string_view(format_str), vargs);
}

FMT_API void vprint(string_view, format_args);
FMT_API void vprint(std::FILE*, string_view, format_args);

/**
  \rst
  Formats ``args`` according to specifications in ``format_str`` and writes the
  output to the file ``f``. Strings are assumed to be Unicode-encoded unless the
  ``FMT_UNICODE`` macro is set to 0.

  **Example**::

    fmt::print(stderr, "Don't {}!", "panic");
  \endrst
 */
```cpp

这段代码是一个C++语言的模板函数，名为print。它接受三个参数：一个输出文件流对象（通常是标准输出流，也就是屏幕输出），一个格式字符串（字符串常量）和一个参数列表。

具体来说，这段代码的作用是格式化字符串中的参数，并输出到标准输出流。首先，它会检查给定的格式字符串是否为Unicode编码，如果是，就使用vprint函数将其输出。如果不是Unicode编码，则将使用detail::vprint_mojibake函数将其输出，该函数会将字符串作为"\uXXXX"格式，并使用to_string_view函数将其转换为Unicode字符序列。

在实际应用中，这段代码可以在编译器中作为类型别名使用，例如：
```c
#include <fmt/core>

int main() {
   fmt::print("Hello, World!");
   return 0;
}
```cpp
这将输出"Hello, World!"到屏幕上。


```
template <typename S, typename... Args, typename Char = char_t<S>>
inline void print(std::FILE* f, const S& format_str, Args&&... args) {
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  return detail::is_unicode<Char>()
             ? vprint(f, to_string_view(format_str), vargs)
             : detail::vprint_mojibake(f, to_string_view(format_str), vargs);
}

/**
  \rst
  Formats ``args`` according to specifications in ``format_str`` and writes
  the output to ``stdout``. Strings are assumed to be Unicode-encoded unless
  the ``FMT_UNICODE`` macro is set to 0.

  **Example**::

    fmt::print("Elapsed time: {0:.2f} seconds", 1.23);
  \endrst
 */
```cpp

这段代码是一个C++模板函数，名为print。它接受两个参数，一个是格式字符串S，另一个是任意数量的不同类型的参数Args。这个函数的作用是打印出字符串S的格式字符串，以及传递给定参数Args的值。

具体来说，这段代码首先通过fmt::make_args_checked函数，将Args类型的参数强制转换成fmt::Arg的格式，然后将它们连接起来，得到一个复杂的Args对象。接着，判断给定的Char类型参数是否是Unicode，如果是，就调用vprint函数，将格式字符串S和Args对象打印出来；如果不是Unicode，就调用vprint_mojibake函数，打印输出字符串S。最后，将输出结果返回。


```
template <typename S, typename... Args, typename Char = char_t<S>>
inline void print(const S& format_str, Args&&... args) {
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  return detail::is_unicode<Char>()
             ? vprint(to_string_view(format_str), vargs)
             : detail::vprint_mojibake(stdout, to_string_view(format_str),
                                       vargs);
}
FMT_END_NAMESPACE

#endif  // FMT_CORE_H_

```