# xmrig源码解析 16

# `src/3rdparty/fmt/format.h`

这段代码是一个C++代码片段，定义了一个名为"Formatting library for C++"的格式库。这个库的作用是帮助程序员格式化C++代码，使得代码更容易阅读和理解。

具体来说，这个库提供了对C++源代码的格式化，包括换行、缩进、注释等。可以通过在代码中包含头文件"fmt.h"来使用库中提供的格式化功能。

值得注意的是，该代码库允许任何人自由地使用、修改和分发库中的代码，前提是在使用时包含该代码库的版权和许可通知。


```cpp
/*
 Formatting library for C++

 Copyright (c) 2012 - present, Victor Zverovich

 Permission is hereby granted, free of charge, to any person obtaining
 a copy of this software and associated documentation files (the
 "Software"), to deal in the Software without restriction, including
 without limitation the rights to use, copy, modify, merge, publish,
 distribute, sublicense, and/or sell copies of the Software, and to
 permit persons to whom the Software is furnished to do so, subject to
 the following conditions:

 The above copyright notice and this permission notice shall be
 included in all copies or substantial portions of the Software.

 THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND,
 EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
 MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
 NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE
 LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
 OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION
 WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

 --- Optional exception to the license ---

 As an exception, if, as a result of your compiling your source code, portions
 of this Software are embedded into a machine-executable object form of such
 source code, you may redistribute such embedded portions in such object form
 without including the above copyright and permission notices.
 */

```

这段代码定义了一个名为FMT_FORMAT_H_的预处理头文件，其中包含了一些与格式化相关的头文件和常量。

以下是具体的作用：

1. 引入了与算法、错误、数学、标准库相关的头文件和常量。

2. 声明了FMT_ICC_VERSION，这是一个预处理指令，指定为Intel编译器编写的代码。

3. 通过包含fmtd申诉函数库头文件，使程序可以访问该函数库。

4. 通过将std::endl强制转换为int8086代码风格的ASCII字符串，使程序可以正确输出到控制台。

5. 通过将-I和-Ipu选项设置为"-I/q64"来指定输入和输出文件的宽度。

6. 通过包含"core.h"，使程序可以访问名为"core"的输出格式化函数的实现。

总之，这段代码定义了一个用于格式化和输入输出的头文件，以及一些与格式化相关的常量和指令。


```cpp
#ifndef FMT_FORMAT_H_
#define FMT_FORMAT_H_

#include <algorithm>
#include <cerrno>
#include <cmath>
#include <cstdint>
#include <limits>
#include <memory>
#include <stdexcept>

#include "core.h"

#ifdef __INTEL_COMPILER
#  define FMT_ICC_VERSION __INTEL_COMPILER
```

这段代码定义了两个宏，分别命名为 FMT_ICC_VERSION 和 FMT_CUDA_VERSION，同时还定义了一个分工会址的函数 __NVCC__。

如果没有定义 FMT_ICC_VERSION 和 FMT_CUDA_VERSION，则直接跳过，否则将定义好的宏替换到 __ICL 和 __NVCC__ 上，最终输出 FMT_ICC_VERSION 和 FMT_CUDA_VERSION。

FMT_HAS_BUILTIN macro 用于检查特定头文件是否已经被定义，如果已经定义，则返回真；否则返回 false。


```cpp
#elif defined(__ICL)
#  define FMT_ICC_VERSION __ICL
#else
#  define FMT_ICC_VERSION 0
#endif

#ifdef __NVCC__
#  define FMT_CUDA_VERSION (__CUDACC_VER_MAJOR__ * 100 + __CUDACC_VER_MINOR__)
#else
#  define FMT_CUDA_VERSION 0
#endif

#ifdef __has_builtin
#  define FMT_HAS_BUILTIN(x) __has_builtin(x)
#else
```

这段代码定义了一系列头文件，其中一些头文件是用来定义特定功能的手动版本，而其他头文件则定义了fmtw和其他实用函数的声明。

以下是每个头文件的注释：

1. `#define FMT_HAS_BUILTIN(x) 0`
这个头文件定义了一个名为`FMT_HAS_BUILTIN`的宏，它的值为0。这个宏告诉编译器它支持FMT语言，但fmtw功能需要手动实现。

2. `#ifdef FMT_GCC_VERSION || FMT_CLANG_VERSION`
这个头文件定义了一个名为`FMT_NOINLINE`的宏，它的值分别为`__attribute__((noinline))`和`__attribute__((noinline))`。这个宏告诉编译器不要将`noinline`修饰的函数和变量看作是内联函数，从而提高程序的性能。

3. `#if __cplusplus == 201103L || __cplusplus == 201402L`
这个头文件定义了一个名为`FMT_FALLTHROW`的宏，它的值为`__attribute__((fallthrough))`。这个宏告诉编译器在一些C++版本中，当`__intel_compiler`或`__pgi`同时定义时，函数可以以非汇编代码形式定义，从而实现更好的可移植性。

4. `#elif defined(__INTEL_COMPILER) || defined(__PGI)`
这个头文件定义了一个名为`FMT_FALLTHROW`的宏，它的值为`__attribute__((fallthrough))`。这个宏告诉编译器，在某些情况下一步可以定义非汇编代码形式的函数。

5. `#elif defined(__clang__)`
这个头文件定义了一个名为`FMT_FALLTHROW`的宏，它的值为`__attribute__((fallthrough))`。这个宏告诉编译器，在某些情况下一步可以定义非汇编代码形式的函数。这个头文件之前定义的`FMT_FALLTHROW`宏与此相同，但是它的兼容性较差，因为它使用了较早的C++版本。

6. `#elif FMT_GCC_VERSION >= 700 && !defined(__EDG_VERSION__) || __EDG_VERSION__ >= 520`
这个头文件定义了一个名为`FMT_NOINLINE`的宏，它的值为`__attribute__((noinline))`。这个宏告诉编译器，在GCC 7.0及更高版本中，函数可以以非汇编代码形式定义，从而实现更好的可移植性。


```cpp
#  define FMT_HAS_BUILTIN(x) 0
#endif

#if FMT_GCC_VERSION || FMT_CLANG_VERSION
#  define FMT_NOINLINE __attribute__((noinline))
#else
#  define FMT_NOINLINE
#endif

#if __cplusplus == 201103L || __cplusplus == 201402L
#  if defined(__INTEL_COMPILER) || defined(__PGI)
#    define FMT_FALLTHROUGH
#  elif defined(__clang__)
#    define FMT_FALLTHROUGH [[clang::fallthrough]]
#  elif FMT_GCC_VERSION >= 700 && \
      (!defined(__EDG_VERSION__) || __EDG_VERSION__ >= 520)
```

这段代码定义了一系列条件状语从句，用于检测编程语言中是否支持C++17中的fallthrough特性。

首先，定义了两种情况：

1. 如果定义了C++17中的fallthrough特性，则使用[[fallthrough]]来定义FMT_FALLTHROUGH。

2. 如果定义了 FMT_HAS_CPP17_ATTRIBUTE(fallthrough) 或者定义了(_MSVC_LANG)且(_MSVC_LANG) >= 201703L，则使用[[fallthrough]]来定义FMT_FALLTHROUGH。否则，使用 default 的 [[fallthrough]] 来定义FMT_FALLTHROW。

如果定义了 FMT_HAS_CPP17_ATTRIBUTE(fallthrough)，则可以使用该特性来自动处理一些可能出现但应该忽略的情况。例如，在使用 compile-time 检查中，可以使用该特性来忽略编译器可能发现的未定义行为。

如果没有定义FMT_HAS_CPP17_ATTRIBUTE(fallthrough)，则需要显式地定义FMT_FALLTHROW或FMT_FALLTHROUGH。


```cpp
#    define FMT_FALLTHROUGH [[gnu::fallthrough]]
#  else
#    define FMT_FALLTHROUGH
#  endif
#elif FMT_HAS_CPP17_ATTRIBUTE(fallthrough) || \
    (defined(_MSVC_LANG) && _MSVC_LANG >= 201703L)
#  define FMT_FALLTHROUGH [[fallthrough]]
#else
#  define FMT_FALLTHROUGH
#endif

#ifndef FMT_MAYBE_UNUSED
#  if FMT_HAS_CPP17_ATTRIBUTE(maybe_unused)
#    define FMT_MAYBE_UNUSED [[maybe_unused]]
#  else
```

这段代码定义了一个名为 FMT_MAYBE_UNUSED 的宏，它表示这是一个 unused 的头文件。然后它检查了一个名为 FMT_THROW 的头文件是否定义了。如果 FMT_THROW 存在，它会检查是否定义了一个名为 FMT_EXCEPTIONS 的头文件。如果是，它会进一步检查是否定义了一个名为 FMT_BELLIPIECOOL 的新头文件。

如果 FMT_BELLIPIECOOL 存在，那么这段代码就是对fmt库的声明。fmt库是一个 C++ 库，它提供了异常处理和格式化支持。

在这段注释中，定义了一个名为 FMT_BEGIN_NAMESPACE 的函数。这个函数是在非 POC 环境下声明的。在函数内部，定义了一个模板类 Exception。这个模板类继承自一个内部类，这个内部类实现了 do_throw 函数。

do_throw 函数是一个通用的异常处理函数。它接收一个 Exception 类型的参数，然后执行一系列操作（包括：关闭可能泄漏的内存资源，读取调试信息，等等）。然后，如果 b 变量被设置为真，它将抛出 x 异常。

这里通过这个函数，可以提供一种异常处理的方式，即使异常信息未被记录，也可以抛出异常使程序终止。


```cpp
#    define FMT_MAYBE_UNUSED
#  endif
#endif

#ifndef FMT_THROW
#  if FMT_EXCEPTIONS
#    if FMT_MSC_VER || FMT_NVCC
FMT_BEGIN_NAMESPACE
namespace detail {
template <typename Exception> inline void do_throw(const Exception& x) {
  // Silence unreachable code warnings in MSVC and NVCC because these
  // are nearly impossible to fix in a generic code.
  volatile bool b = true;
  if (b) throw x;
}
}  // namespace detail
```

这段代码定义了一系列命名的模板函数，用于在程序运行时捕获异常，并对其进行处理。

具体来说，这段代码包括了以下几种功能：

1. FMT_END_NAMESPACE：这是一个声明，表示该代码块已经结束，但不会对后续代码产生影响。

2. 模板类FMT_THROW：定义了一系列模板函数，用于在程序运行时捕获异常，并对其进行处理。其中，x参数表示要捕获的异常类型。

3. 宏定义FMT_THROW：对FMT_THROW函数进行宏定义，以便在需要使用时方便地引用。如果用户没有指定x参数，则宏定义会生成一个包含x类型特性的头文件。

4. 伪代码FMT_EXCEPTIONS：用于在程序出现异常情况时进行处理。该伪代码块定义了一系列条件，如果其中任何一种条件为真，则会执行FMT_THROW函数，否则跳过该函数。

5. 自定义异常类Exception：定义了自定义的异常类Exception，用于表示程序运行时可能发生的异常情况。

6. 自定义函数throw：重写了操作系统标准库中的throw函数，用于在程序出现异常情况时抛出异常。该函数接收x参数，表示要抛出的异常类型。

总之，这段代码定义了一系列模板函数和自定义异常类，用于在程序运行时捕获异常并对其进行处理。


```cpp
FMT_END_NAMESPACE
#      define FMT_THROW(x) detail::do_throw(x)
#    else
#      define FMT_THROW(x) throw x
#    endif
#  else
#    define FMT_THROW(x)              \
      do {                            \
        static_cast<void>(sizeof(x)); \
        FMT_ASSERT(false, "");        \
      } while (false)
#  endif
#endif

#if FMT_EXCEPTIONS
```

这段代码定义了一系列宏，用于在编程语言中处理try-catch语句。

宏定义开始的部分#define FMT_TRY define FMT_TRY子句，后面是try块，这里可以包含try语句，try语句后面可以是任意表达式，但最少要有两个表达式，一个return关键字。这里的try语句后面跟了一个if条件，判断这个if是否成立，如果成立，则执行try块内的代码，否则跳过try块。

#define FMT_CATCH(x) catch (x)这里，定义了一个宏FMT_CATCH，后面是捕获变量的定义，x可以是该函数或者该变量的任何别名。这个宏后面可以跟一个表达式，如果是捕获变量的定义，也可以省略，省略了捕获变量的定义，那么宏就自动认为x就是该函数或变量的任何别名。

#else如果条件不成立，就需要执行宏定义中#define FMT_TRY if (true)这里的代码，这里的代码可以理解为，如果条件不成立，就需要执行try块内的代码。这里，通过判断条件是否成立，如果成立，则执行try块内的代码，否则跳过try块。

#define FMT_TRY if (true)这里，定义了一个宏FMT_TRY，后面是if条件，如果这个if成立，则执行FMT_CATCH(x)这里的代码，否则跳过。这里的x可以是该函数或者该变量的任何别名。

#define FMT_CATCH(x) if (false)这里，定义了一个宏FMT_CATCH，后面是捕获变量的定义，x可以是该函数或者该变量的任何别名。这个宏后面可以跟一个表达式，如果是捕获变量的定义，也可以省略，省略了捕获变量的定义，那么宏就自动认为x就是该函数或变量的任何别名。


```cpp
#  define FMT_TRY try
#  define FMT_CATCH(x) catch (x)
#else
#  define FMT_TRY if (true)
#  define FMT_CATCH(x) if (false)
#endif

#ifndef FMT_USE_USER_DEFINED_LITERALS
// EDG based compilers (Intel, NVIDIA, Elbrus, etc), GCC and MSVC support UDLs.
#  if (FMT_HAS_FEATURE(cxx_user_literals) || FMT_GCC_VERSION >= 407 || \
       FMT_MSC_VER >= 1900) &&                                         \
      (!defined(__EDG_VERSION__) || __EDG_VERSION__ >= /* UDL feature */ 480)
#    define FMT_USE_USER_DEFINED_LITERALS 1
#  else
#    define FMT_USE_USER_DEFINED_LITERALS 0
```

这段代码是一个 conditional 预处理指令，它会根据一系列条件判断是否支持 UDL（User-Defined Local Liquid）模板。如果满足条件，就会定义 FMT_USE_UDL_TEMPLATE 为 1，否则定义为 0。

UDL 模板是一种程序员可以自定义的模板，它可以用来定义程序中复用的代码。然而，这个功能在一些编译器中可能没有得到支持，包括 GCC 和拱可以使用 6.4 的版本。

在此代码中，通过检查 __EDG_VERSION__、__NVCC__、FMT_GCC_VERSION 和 __PGI__ 是否定义，来判断是否支持 UDL 模板。如果 FMT_USE_USER_DEFINED_LITERALS 为真，并且不满足上述任何一个条件，那么就会定义 FMT_USE_UDL_TEMPLATE 为 0。否则，就会定义 FMT_USE_UDL_TEMPLATE 为 1。


```cpp
#  endif
#endif

#ifndef FMT_USE_UDL_TEMPLATE
// EDG frontend based compilers (icc, nvcc, PGI, etc) and GCC < 6.4 do not
// properly support UDL templates and GCC >= 9 warns about them.
#  if FMT_USE_USER_DEFINED_LITERALS &&                         \
      (!defined(__EDG_VERSION__) || __EDG_VERSION__ >= 501) && \
      ((FMT_GCC_VERSION >= 604 && __cplusplus >= 201402L) ||   \
       FMT_CLANG_VERSION >= 304) &&                            \
      !defined(__PGI) && !defined(__NVCC__)
#    define FMT_USE_UDL_TEMPLATE 1
#  else
#    define FMT_USE_UDL_TEMPLATE 0
#  endif
```

这段代码是一个C语言中的预处理指令，定义了几个fmtdir Floating-Point Format (FPF) 的使用选项。

#ifdef FMT_USE_FLOAT
#   if (defined(" FMT_USE_FLOAT "))
#       define FMT_USE_FLOAT 1
#   endif
#endif

#ifdef FMT_USE_DOUBLE
#   if (defined(" FMT_USE_DOUBLE "))
#       define FMT_USE_DOUBLE 1
#   endif
#endif

#ifdef FMT_USE_LONG_DOUBLE
#   if (defined(" FMT_USE_LONG_DOUBLE "))
#       define FMT_USE_LONG_DOUBLE 1
#   endif
#endif

// Defining FMT_REDUCE_INT_INSTANTIATIONS to 1, will reduce the number of
// instantiations of the function FMT_REDUCE_INT_INSTANTIATIONS that
// are defined in the translation unit with this preprocessor directive.
#define FMT_REDUCE_INT_INSTANTIATIONS 1

void FMT_REDUCE_INT_INSTANTIATIONS(int64_t x) {
   int64_t y = 0;
   switch (FMT_USE_FLOAT) {
       case 0:
           y = x;
           break;
       case 1:
           y = fmtdir_s<float>(x);
           break;
       case 2:
           y = fmtdir_s<double>(x);
           break;
       case 3:
           y = fmtdir_s<long_double>(x);
           break;
       case 4:
           y = fmtdir_s<int64_t>(x);
           break;
       case 5:
           y = fmtdir_s<float>(x);
           break;
       case 6:
           y = fmtdir_s<double>(x);
           break;
       case 7:
           y = fmtdir_s<long_double>(x);
           break;
       case 8:
           y = fmtdir_s<int64_t>(x);
           break;
       case 9:
           y = fmtdir_s<float>(x);
           break;
       case 10:
           y = fmtdir_s<double>(x);
           break;
       case 11:
           y = fmtdir_s<long_double>(x);
           break;
       case 12:
           y = fmtdir_s<int64_t>(x);
           break;
       case 13:
           y = fmtdir_s<float>(x);
           break;
       case 14:
           y = fmtdir_s<double>(x);
           break;
       case 15:
           y = fmtdir_s<long_double>(x);
           break;
       case 16:
           y = fmtdir_s<int64_t>(x);
           break;
       case 17:
           y = fmtdir_s<float>(x);
           break;
       case 18:
           y = fmtdir_s<double>(x);
           break;
       case 19:
           y = fmtdir_s<long_double>(x);
           break;
       case 20:
           y = fmtdir_s<int64_t>(x);
           break;
       case 21:
           y = fmtdir_s<float>(x);
           break;
       case 22:
           y = fmtdir_s<double>(x);
           break;
       case 23:
           y = fmtdir_s<long_double>(x);
           break;
       case 24:
           y = fmtdir_s<int64_t>(x);
           break;
       case 25:
           y = fmtdir_s<float>(x);
           break;
       case 26:
           y = fmtdir_s<double>(x);
           break;
       case 27:
           y = fmtdir_s<long_double>(x);
           break;
       case 28:
           y = fmtdir_s<int64_t>(x);
           break;
       case 29:
           y = fmtdir_s<float>(x);
           break;
       case 30:
           y = fmtdir_s<double>(x);
           break;
       case 31:
           y = fmtdir_s<long_double>(x);
           break;
       case 32:
           y = fmtdir_s<int64_t>(x);
           break;
       case 33:
           y = fmtdir_s<float>(x);
           break;
       case 34:
           y = fmtdir_s<double>(x);
           break;
       case 35:
           y = fmtdir_s<long_double>(x);
           break;
       case 36:
           y = fmtdir_s<int64_t>(x);
           break;
       case 37:
           y = fmtdir_s<float>(x);
           break;
       case 38:
           y = fmtdir_s<double>(x);
           break;
       case 39:
           y = fmtdir_s<long_double>(x);
           break;
       case 40:
           y = fmtdir_s<int64_t>(x);
           break;
       case 41:
           y = fmtdir_s<float>(x);
           break;
       case 42:
           y = fmtdir_s<double>(x);
           break;
       case 43:
           y = fmtdir_s<long_double>(x);
           break;
       case 44:
           y = fmtdir_s<int64_t>(x);
           break;
       case 45:
           y = fmtdir_s<float>(x);
           break;
       case 46:
           y = fmtdir_s<double>(x);
           break;
       case 47:
           y = fmtdir_s<long_double>(x);
           break;
       case 48:
           y = fmtdir_s<int64_t>(x);
           break;
       case 49:
           y = fmtdir_s<float>(x);
           break;
       case 50:
           y = fmtdir_s<double>(x);
           break;
       case 51:
           y = fmtdir_s<long_double>(x);
           break;
       case 52:
           y = fmtdir_s<int64_t>(x);
           break;
       case 53:
           y = fmtdir_s<float>(x);
           break;
       case 54:
           y = fmtdir_s<double>(x);
           break;
       case 55:
           y = fmtdir_s<long_double>(x);
           break;
       case 56:
           y = fmtdir_s<int64_t>(x);
           break;
       case 57:


```cpp
#endif

#ifndef FMT_USE_FLOAT
#  define FMT_USE_FLOAT 1
#endif

#ifndef FMT_USE_DOUBLE
#  define FMT_USE_DOUBLE 1
#endif

#ifndef FMT_USE_LONG_DOUBLE
#  define FMT_USE_LONG_DOUBLE 1
#endif

// Defining FMT_REDUCE_INT_INSTANTIATIONS to 1, will reduce the number of
```

这段代码定义了一个名为`int_writer`的模板类，它有一个模板参数`int`。这个模板类的作用是减少二进制文件的大小，但会导致整数格式化性能的下降。

具体来说，这段代码通过__builtin_clz()函数来计算整数在二进制文件中的长度，如果这个函数在未来可用的Clang版本中定义，则可以直接使用这个函数来计算。否则，就需要根据__builtin_clz()函数的实现来定义一个名为FMT_BUILTIN_CLZ()的函数来实现这个计算。如果FMT库中定义了FMT_REDUCE_INT_INSTANTIATIONS，则可以使用0来代替这个常量，否则就会导致编译错误。


```cpp
// int_writer template instances to just one by only using the largest integer
// type. This results in a reduction in binary size but will cause a decrease in
// integer formatting performance.
#if !defined(FMT_REDUCE_INT_INSTANTIATIONS)
#  define FMT_REDUCE_INT_INSTANTIATIONS 0
#endif

// __builtin_clz is broken in clang with Microsoft CodeGen:
// https://github.com/fmtlib/fmt/issues/519
#if (FMT_GCC_VERSION || FMT_HAS_BUILTIN(__builtin_clz)) && !FMT_MSC_VER
#  define FMT_BUILTIN_CLZ(n) __builtin_clz(n)
#endif
#if (FMT_GCC_VERSION || FMT_HAS_BUILTIN(__builtin_clzll)) && !FMT_MSC_VER
#  define FMT_BUILTIN_CLZLL(n) __builtin_clzll(n)
#endif
```

这段代码是一个条件编译语句，会根据输入的编译器版本或者已经定义好的编译器扩展（比如 GCC 和 MSVC）来定义不同的宏。

具体来说，当 FMT_GCC_VERSION 或 FMT_HAS_BUILTIN(__builtin_ctz) 时，定义 FMT_BUILTIN_CTZ(n) 为 __builtin_ctz(n)。当 FMT_GCC_VERSION 或 FMT_HAS_BUILTIN(__builtin_ctzll) 时，定义 FMT_BUILTIN_CTZLL(n) 为 __builtin_ctzll(n)。

此外，#if FMT_MSC_VER 语句会包含一些与 CPU 架构有关的头文件，如 _BitScanReverse[64] 和 _BitScanForward[64]。这是因为 FMT_MSC_VER 表示当前编译器支持 MSVC，也就是基于 IntelliS预处理器的一个版本。

最终，这段代码的作用是定义了 FMT_BUILTIN_CTZ 和 FMT_BUILTIN_CTZLL，根据输入的编译器版本来选择定义哪个宏。同时，也包含了一些与 CPU 架构有关的头文件。


```cpp
#if (FMT_GCC_VERSION || FMT_HAS_BUILTIN(__builtin_ctz))
#  define FMT_BUILTIN_CTZ(n) __builtin_ctz(n)
#endif
#if (FMT_GCC_VERSION || FMT_HAS_BUILTIN(__builtin_ctzll))
#  define FMT_BUILTIN_CTZLL(n) __builtin_ctzll(n)
#endif

#if FMT_MSC_VER
#  include <intrin.h>  // _BitScanReverse[64], _BitScanForward[64], _umul128
#endif

// Some compilers masquerade as both MSVC and GCC-likes or otherwise support
// __builtin_clz and __builtin_clzll, so only define FMT_BUILTIN_CLZ using the
// MSVC intrinsics if the clz and clzll builtins are not available.
#if FMT_MSC_VER && !defined(FMT_BUILTIN_CLZLL) && \
    !defined(FMT_BUILTIN_CTZLL) && !defined(_MANAGED)
```

这段代码是一个C++语言的类成员函数，名为“clz”。它用于检测给定的32位无符号整数x是否为0，如果是0，则返回31。如果不是0，则返回clz函数的返回值。

具体来说，代码首先定义了一个名为“_BitScanForward64”的类成员函数，它的参数是一个无符号32位整数。然后定义了一个名为“clz”的函数，它接收这个类成员函数的一个参数，并使用BitScanReverse64函数来获取这个无符号整数之前的二进制位。

接着，代码检查给定的整数x是否为0，如果是0，则输出一个警告。否则，代码使用数学操作符“^”来计算一个右移运算的结果，并将其与31进行按位或运算，最后将结果存储回到整数变量“x”中。

总结起来，这段代码定义了一个名为“clz”的函数，用于检测给定的32位无符号整数是否为0，并返回相应的结果。


```cpp
FMT_BEGIN_NAMESPACE
namespace detail {
// Avoid Clang with Microsoft CodeGen's -Wunknown-pragmas warning.
#  ifndef __clang__
#    pragma intrinsic(_BitScanForward)
#    pragma intrinsic(_BitScanReverse)
#  endif
#  if defined(_WIN64) && !defined(__clang__)
#    pragma intrinsic(_BitScanForward64)
#    pragma intrinsic(_BitScanReverse64)
#  endif

inline int clz(uint32_t x) {
  unsigned long r = 0;
  _BitScanReverse(&r, x);
  FMT_ASSERT(x != 0, "");
  // Static analysis complains about using uninitialized data
  // "r", but the only way that can happen is if "x" is 0,
  // which the callers guarantee to not happen.
  FMT_SUPPRESS_MSC_WARNING(6102)
  return 31 ^ static_cast<int>(r);
}
```

这段代码定义了一个名为 FMT_BUILTIN_CLZ 的函数，它的参数为 uint64_t 类型的变量 x。

函数内部实现了一个名为 clzll 的内部函数，它接收并返回一个整数类型的值。这个函数是双关的，既可以代表 num元素的输出，也可以代表 int 类型的输入输出。

函数实现的主要步骤如下：

1. 定义了一个名为 r 的变量，初始化为 0。
2. 判断输入的 x 是否为 0，如果是，则直接返回 0。
3. 如果 x 不是 0，就开始扫描其高 32 位和低 32 位。
4. 如果高 32 位上有一种 1，代表 x 是负数，返回 63 - (r + 32)。
5. 如果低 32 位上扫描到一种 1，代表 x 是负数，返回 63 - r。
6. 最后，返回低 32 位上的值。

这段代码的主要目的是为了实现一个名为 num2str 的函数，它可以将一个 int 类型的变量 x 输出为 num2str 函数类型的值，也可以将 num2str 函数类型的值输出为 x。通过将输入的 x 输出为 num2str 函数类型的值，可以方便地将 int 类型变量与字符串进行交互，以及实现类似于 printf 的功能。


```cpp
#  define FMT_BUILTIN_CLZ(n) detail::clz(n)

inline int clzll(uint64_t x) {
  unsigned long r = 0;
#  ifdef _WIN64
  _BitScanReverse64(&r, x);
#  else
  // Scan the high 32 bits.
  if (_BitScanReverse(&r, static_cast<uint32_t>(x >> 32))) return 63 ^ (r + 32);
  // Scan the low 32 bits.
  _BitScanReverse(&r, static_cast<uint32_t>(x));
#  endif
  FMT_ASSERT(x != 0, "");
  FMT_SUPPRESS_MSC_WARNING(6102)  // Suppress a bogus static analysis warning.
  return 63 ^ static_cast<int>(r);
}
```



这两行代码定义了两个宏，分别命名为FMT_BUILTIN_CLZLL和FMT_BUILTIN_CTZ，它们的作用是定义一个名为ctz的函数类型，以及一个名为ctzll的函数类型。

函数ctz的定义包括一个名为x的uint32类型的参数，函数返回值为int类型的整数。函数实现了一个从输入参数x中计算并返回x的clzll值(即clzll函数的内部实现)，其中clzll函数的实现依赖于FMT_BUILTIN_CLZLL macro定义。

函数ctzll的定义包括一个名为x的uint64类型的参数，函数实现了一个从输入参数x中计算并返回x的clzll值，与函数ctz的实现类似，只是返回值的类型为unsigned long long类型的整数。

这两行代码中包含了一个名为FMT_SUPPRESS_MSC_WARNING的 macro，它会抑制MSC(Msvc)警告，警告类型为“bogus static analysis warning”。这意味着尽管函数的实现没有问题，但MSC警告仍然会发布，警告类型为“bogus”。


```cpp
#  define FMT_BUILTIN_CLZLL(n) detail::clzll(n)

inline int ctz(uint32_t x) {
  unsigned long r = 0;
  _BitScanForward(&r, x);
  FMT_ASSERT(x != 0, "");
  FMT_SUPPRESS_MSC_WARNING(6102)  // Suppress a bogus static analysis warning.
  return static_cast<int>(r);
}
#  define FMT_BUILTIN_CTZ(n) detail::ctz(n)

inline int ctzll(uint64_t x) {
  unsigned long r = 0;
  FMT_ASSERT(x != 0, "");
  FMT_SUPPRESS_MSC_WARNING(6102)  // Suppress a bogus static analysis warning.
```

这段代码是一个C++语言中的函数，它使用了_BitScanForward和detail::ctzll函数。

_BitScanForward64函数是一个位移函数，用于从给定的起始位向右扫描指定长度的二进制位，并返回扫描到的二进制位的数量。该函数使用r和x作为输入参数，其中r是寄存器，x是外部变量。

detail::ctzll函数是一个常量函数，用于计算指定n的值，其中n是输入参数。该函数使用的是"自定义码点"技术，也称为CTZLL算法。

该代码的作用是实现了一个名为"fmt"的命名函数，它会根据传入的整数n计算出n的"自定义码点"值。具体地，它会根据n的奇偶性选择不同的实现方式，具体实现如下：

```cpp
#ifdef _WIN64
 _BitScanForward64(&r, x);
#else
 if (_BitScanForward(&r, static_cast<uint32_t>(x))) return static_cast<int>(r);
 _BitScanForward(&r, static_cast<uint32_t>(x >> 32));
 r += 32;
 return static_cast<int>(r);
#endif
```

首先，如果输入的n是偶数，那么函数会直接返回r的值。否则，函数会先扫描low32位，如果low32位上的值与x相等，就返回n的值。如果low32位上的值与x不相等，那么函数会扫描high32位，然后将x的高32位与r的低32位和声高32位与r的高32位。最后，函数会计算r的值，并返回r的值。

综上所述，该函数实现了一个命名函数fmt，它会根据输入的整数n计算出n的值，并返回n的值。


```cpp
#  ifdef _WIN64
  _BitScanForward64(&r, x);
#  else
  // Scan the low 32 bits.
  if (_BitScanForward(&r, static_cast<uint32_t>(x))) return static_cast<int>(r);
  // Scan the high 32 bits.
  _BitScanForward(&r, static_cast<uint32_t>(x >> 32));
  r += 32;
#  endif
  return static_cast<int>(r);
}
#  define FMT_BUILTIN_CTZLL(n) detail::ctzll(n)
}  // namespace detail
FMT_END_NAMESPACE
#endif

```

这段代码定义了一个头文件 FMT_DEPRECATED_NUMERIC_ALIGN，其中包含一个模板类 Dest 和一个模板参数 Source。这个模板类实现了一个等效于 `*reinterpret_cast<Dest*>(&source)` 的函数，但其含义更加明确，避免了类型歧义。

这个函数的作用是允许在输入参数为 int、double 或 enum 时，对其进行强制类型转换，并返回一个新的 Dest 类型的目标变量。在函数实现中，首先包含一个静态判断，确保输入参数和 Dest 变量的大小样匹配。然后通过 std::memcpy 函数将输入参数的值复制到 Dest 变量的起始位置，最后返回生成的 Dest 类型变量。


```cpp
// Enable the deprecated numeric alignment.
#ifndef FMT_DEPRECATED_NUMERIC_ALIGN
#  define FMT_DEPRECATED_NUMERIC_ALIGN 0
#endif

FMT_BEGIN_NAMESPACE
namespace detail {

// An equivalent of `*reinterpret_cast<Dest*>(&source)` that doesn't have
// undefined behavior (e.g. due to type aliasing).
// Example: uint64_t d = bit_cast<uint64_t>(2.718);
template <typename Dest, typename Source>
inline Dest bit_cast(const Source& source) {
  static_assert(sizeof(Dest) == sizeof(Source), "size mismatch");
  Dest dest;
  std::memcpy(&dest, &source, sizeof(dest));
  return dest;
}

```

这段代码定义了一个名为 is_big_endian 的函数，用于检查给定的整数是否使用了大端序。如果该整数使用了大端序，函数将返回真，否则将返回 false。

该代码还定义了一个名为 fallback_uintptr 的结构体，用于在缺少 uintptr 类型(即一些系统不支持该类型)的情况下提供一种 fallback 机制。该结构体包含一个字节数组，用于存储 void 类型的指针，以及一个默认构造函数和一个 explicit 构造函数。

default 构造函数将 fallback_uintptr 初始化为一个字节数组，如果没有提供具体的值，则其值将默认为 0。explicit 构造函数允许用户提供一个指向 void 类型的指针，然后将其转换为 fallback_uintptr 类型。如果 is_big_endian 函数返回真，则该构造函数将在创建 fallback_uintptr 对象时将字节数组的值颠倒过来。

该代码还定义了一个名为 uintptr_t 的别名，用于存储 uintptr 类型。该别名在某些系统上可能没有，例如在 Linux 3.13 和更高版本上，但提供了 fallback 机制，允许用户在缺少 uintptr 类型时使用该别名。


```cpp
inline bool is_big_endian() {
  const auto u = 1u;
  struct bytes {
    char data[sizeof(u)];
  };
  return bit_cast<bytes>(u).data[0] == 0;
}

// A fallback implementation of uintptr_t for systems that lack it.
struct fallback_uintptr {
  unsigned char value[sizeof(void*)];

  fallback_uintptr() = default;
  explicit fallback_uintptr(const void* p) {
    *this = bit_cast<fallback_uintptr>(p);
    if (is_big_endian()) {
      for (size_t i = 0, j = sizeof(void*) - 1; i < j; ++i, --j)
        std::swap(value[i], value[j]);
    }
  }
};
```

这段代码定义了一系列宏，用于在不同的编译器预处理器中定义和使用 `uintptr_t` 和 `fallback_uintptr` 类型。

首先，定义了两个条件语句 `#ifdef UINTPTR_MAX` 和 `#else`。如果预处理器定义了 `UINTPTR_MAX`，则定义了 `using uintptr_t = ::uintptr_t`，这将允许使用 `uintptr_t` 类型。否则，定义了 `using uintptr_t = fallback_uintptr`，这将允许使用 `fallback_uintptr` 类型。

接下来，定义了一个名为 `to_uintptr` 的函数，接受一个 `const void*` 类型的参数，并返回一个 `uintptr_t` 类型的值。如果预处理器定义了 `UINTPTR_MAX`，则该函数将直接返回 `uintptr_t` 类型的值。否则，将调用 `fallback_uintptr` 的函数，这将返回一个指向 `uintptr_t` 类型的指针，或者调用 `UINTPTR_MAX` 函数，这将返回 `std::numeric_limits<T>::max()` 函数，这将返回一个 `T` 类型的最大值。

最后，定义了一个名为 `max_value` 的模板函数，它是一个计算 `T` 类型最大值的模板函数。该函数返回 `std::numeric_limits<T>::max()` 函数的默认实现，但是它可以通过 `constexpr` 修饰符进行编译时计算。


```cpp
#ifdef UINTPTR_MAX
using uintptr_t = ::uintptr_t;
inline uintptr_t to_uintptr(const void* p) { return bit_cast<uintptr_t>(p); }
#else
using uintptr_t = fallback_uintptr;
inline fallback_uintptr to_uintptr(const void* p) {
  return fallback_uintptr(p);
}
#endif

// Returns the largest possible value for type T. Same as
// std::numeric_limits<T>::max() but shorter and not affected by the max macro.
template <typename T> constexpr T max_value() {
  return (std::numeric_limits<T>::max)();
}
```

这段代码定义了一个模板类数组`num_bits`，用于计算给定类型的整数在二进制表示中所需的比特数。

该模板使用了C++标准库中的`std::numeric_limits`模板类，用于保证对于各种整数类型，都能够正确地计算所需的比特数。

具体实现中，对于`int128_t`和`uint128_t`类型，直接使用`num_bits`函数计算所需的比特数，因为这两种类型的整数已经占据了128位。

对于`fallback_uintptr`类型，首先使用`sizeof(void*)`获取该类型所需的字节数，然后使用`std::numeric_limits<unsigned char>::digits`计算所需的比特数。最后将这个比特数转换为整数类型，并将其作为参数传递给`__builtin_assume`函数，用于确保在编译期就能够保证该条件为真。


```cpp
template <typename T> constexpr int num_bits() {
  return std::numeric_limits<T>::digits;
}
// std::numeric_limits<T>::digits may return 0 for 128-bit ints.
template <> constexpr int num_bits<int128_t>() { return 128; }
template <> constexpr int num_bits<uint128_t>() { return 128; }
template <> constexpr int num_bits<fallback_uintptr>() {
  return static_cast<int>(sizeof(void*) *
                          std::numeric_limits<unsigned char>::digits);
}

FMT_INLINE void assume(bool condition) {
  (void)condition;
#if FMT_HAS_BUILTIN(__builtin_assume)
  __builtin_assume(condition);
```

这段代码是一个模板元编程技巧，它的主要目的是为了在C++标准中，增加对<iterator>和<sentinel>等新类型的支持。这两个新的类型分别是在C++11中引入的，可以在C++17之前通过编译器进行支持。

具体来说，这段代码中包含两个模板类：iterator_t和sentinel_t。它们的主要作用是在定义模板实参时，根据参数的类型生成对应的模板元编程代码。在实际应用中，这两个类型的变量会在编译期进行推导，所以它们会为各种各样的类型生成对应的iterator和sentinel。

此外，还有一段代码定义了一个名为get_data的函数，该函数的实现是针对std::string类型的。它的作用是提供一个简化版的get_data函数，这个函数在需要时动态地获取字符串中的数据，而不需要在编译时就进行预先定义。

还有一段代码，定义了一个名为PreprocessorHeaders的类。该类包含了一些预处理指令，例如#ifdef和#endif，用于根据定义的模板元编程环境来选择是否输出特定的头文件。


```cpp
#endif
}

// An approximation of iterator_t for pre-C++20 systems.
template <typename T>
using iterator_t = decltype(std::begin(std::declval<T&>()));
template <typename T> using sentinel_t = decltype(std::end(std::declval<T&>()));

// A workaround for std::string not having mutable data() until C++17.
template <typename Char> inline Char* get_data(std::basic_string<Char>& s) {
  return &s[0];
}
template <typename Container>
inline typename Container::value_type* get_data(Container& c) {
  return c.data();
}

```

这段代码定义了一个名为“make_checked”的模板函数，用于检查传入的参数“p”和“size”，并返回一个指向“T”类型的指针。如果定义了“_SECURE_SCL”宏，则函数将返回一个“checked_ptr”对象，否则将返回一个“T”指针。

具体来说，如果定义了“_SECURE_SCL”宏，则说明使用的是C++ Security C++库，该库提供了对指针和容器的安全性检查。模板元编程规则允许我们使用模板元编程来定义一个函数，该函数将根据“_SECURE_SCL”宏和我们的编译器版本来返回一个“checked_ptr”对象或一个“T”指针。

如果未定义“_SECURE_SCL”宏，则函数将直接返回一个“T”指针。如果定义了该宏，则函数将返回一个“checked_ptr”对象，该对象将允许我们在编译时对传入的指针和容器进行安全检查。

此外，该代码中还有一些其他的安全性检查。例如，该代码使用了C++17中的智能指针技术，可以在编译时检测到指针的删除和越界行为。另外，该代码使用了一个称为“FMT_ENABLE_IF”的11维特性，该特性允许我们在函数内使用属性表达式，以提高代码的可读性和可维护性。


```cpp
#if defined(_SECURE_SCL) && _SECURE_SCL
// Make a checked iterator to avoid MSVC warnings.
template <typename T> using checked_ptr = stdext::checked_array_iterator<T*>;
template <typename T> checked_ptr<T> make_checked(T* p, size_t size) {
  return {p, size};
}
#else
template <typename T> using checked_ptr = T*;
template <typename T> inline T* make_checked(T* p, size_t) { return p; }
#endif

template <typename Container, FMT_ENABLE_IF(is_contiguous<Container>::value)>
#if FMT_CLANG_VERSION
__attribute__((no_sanitize("undefined")))
#endif
```

这两段代码定义了一个名为reserve的函数，它接受两个参数：一个指向容器中元素的迭代器it，以及一个整数n。函数的作用是在容器中插入n个元素，并将返回值类型指定为容器中元素的类型。

具体来说，首先通过get_container函数获取容器中的元素迭代器it所指向的容器，然后使用resize函数将容器大小增加到n个元素。由于容器大小是动态增长的，所以resize函数会在需要扩展时自动分配内存，而不是像std::resize函数那样直接在内存中重新分配。

然后，函数使用make_checked函数将返回值类型设置为get_data(it)类型，并将其与resized_size相加得到一个新的返回值。make_checked函数会检查返回值是否为有效的内存，并在需要时抛出std::bad_alloc异常。

以下是代码的伪代码：
```cppphp
inline inline_checked_ptr<typename Container::value_type>
reserve(std::back_insert_iterator<Container> it, size_t n) {
   Container& c = get_container(it);
   size_t size = c.size();
   c.resize(size + n);
   return make_checked(get_data(c) + size, n);
}

template <typename T>
inline buffer_appender<T> reserve(buffer_appender<T> it, size_t n) {
   buffer<T>& buf = get_container(it);
   buf.try_reserve(buf.size() + n);
   return it;
}
```
其中，`reserve`函数接受两个参数，一个是指向容器中元素的迭代器it，另一个是插入的元素数量n。函数返回一个新的迭代器，指向插入元素后的容器。

`reserve`函数的具体实现中，首先使用`get_container`函数获取容器中的元素迭代器it所指向的容器，然后使用`resize`函数将容器大小增加到n个元素。这里需要注意的是，`resize`函数会自动分配内存，所以不需要像`std::resize`函数那样，在循环中逐个增加元素的大小。

接下来，函数使用`make_checked`函数将返回值类型设置为容器中元素的类型，并将其与resized_size相加得到一个新的返回值。`make_checked`函数会检查返回值是否为有效的内存，并在需要时抛出`std::bad_alloc`异常。

最后，`reserve`函数的实现主要分为两部分：将容器大小增加到所需大小，以及检查返回值是否为有效的内存。


```cpp
inline checked_ptr<typename Container::value_type>
reserve(std::back_insert_iterator<Container> it, size_t n) {
  Container& c = get_container(it);
  size_t size = c.size();
  c.resize(size + n);
  return make_checked(get_data(c) + size, n);
}

template <typename T>
inline buffer_appender<T> reserve(buffer_appender<T> it, size_t n) {
  buffer<T>& buf = get_container(it);
  buf.try_reserve(buf.size() + n);
  return it;
}

```

这段代码定义了三个模板类 iterator 和 pointer to_pointer。其中，iterator 是一个模板类，可以输出一个迭代器对象；pointer to_pointer 是一个模板类，可以输出一个指向输出迭代器的指针；而 template <typename Iterator> inline Iterator& reserve(Iterator& it, size_t n) 是 Reserve 函数的实现，该函数接收一个迭代器对象 it 和一个大小 n，返回 it 继续指向原来的迭代器对象，同时将迭代器 object 由原来的迭代器对象 it 转换为指向迭代器 object it 的迭代器引用，即 it 指向的对象的迭代器引用仍然指向原来的迭代器 object it，但迭代器 object it 不再可以修改。 

constexpr T* to_pointer(OutputIt, size_t n) 的实现类似于 Reserve 函数，但是，如果迭代器对象 it 不是输出迭代器对象，而是普通迭代器，则需要手动循环输出迭代器中的元素，并计算出输出迭代器的大小，然后将 it 指向的对象的迭代器引用赋值给 T* to_pointer。 

template <typename T> T* to_pointer(buffer_appender<T> it, size_t n) 的实现则是实现 to_pointer 函数，将 it 指向的对象的迭代器引用赋值给 T* to_pointer，同时，通过 get_container 函数获取到 it 指向的对象的容器，并计算出容器的大小 n，最后返回 it 指向的对象的迭代器引用。


```cpp
template <typename Iterator> inline Iterator& reserve(Iterator& it, size_t) {
  return it;
}

template <typename T, typename OutputIt>
constexpr T* to_pointer(OutputIt, size_t) {
  return nullptr;
}
template <typename T> T* to_pointer(buffer_appender<T> it, size_t n) {
  buffer<T>& buf = get_container(it);
  auto size = buf.size();
  if (buf.capacity() < size + n) return nullptr;
  buf.try_resize(size + n);
  return buf.data() + size;
}

```

这段代码定义了一个名为counting_iterator的输出迭代器类，它能够输出一个容器中的元素，并统计对象的写入次数。该容器需要使用标准模板库中的<container>模板来定义容器类型。

具体来说，这段代码实现了一个计数器迭代器，该迭代器使用基于后插入的迭代器来遍历容器中的元素，并记录每个元素的写入次数。该计数器迭代器使用了输出迭代器标记符，因此可以像普通的迭代器一样使用，同时也使用了检查指针<typename Container::value_type>来确保容器中的元素是连续的。

另外，该代码还实现了一个简单的迭代器，该迭代器与计数器迭代器类似，但是不计算元素的写入次数，因此它只能用于输出容器中的元素。

最后，该代码还定义了一个名为counting_iterator的检查指针，该指针可以用来检查迭代器是否为计数器迭代器。


```cpp
template <typename Container, FMT_ENABLE_IF(is_contiguous<Container>::value)>
inline std::back_insert_iterator<Container> base_iterator(
    std::back_insert_iterator<Container>& it,
    checked_ptr<typename Container::value_type>) {
  return it;
}

template <typename Iterator>
inline Iterator base_iterator(Iterator, Iterator it) {
  return it;
}

// An output iterator that counts the number of objects written to it and
// discards them.
class counting_iterator {
 private:
  size_t count_;

 public:
  using iterator_category = std::output_iterator_tag;
  using difference_type = std::ptrdiff_t;
  using pointer = void;
  using reference = void;
  using _Unchecked_type = counting_iterator;  // Mark iterator as checked.

  struct value_type {
    template <typename T> void operator=(const T&) {}
  };

  counting_iterator() : count_(0) {}

  size_t count() const { return count_; }

  counting_iterator& operator++() {
    ++count_;
    return *this;
  }
  counting_iterator operator++(int) {
    auto it = *this;
    ++*this;
    return it;
  }

  friend counting_iterator operator+(counting_iterator it, difference_type n) {
    it.count_ += static_cast<size_t>(n);
    return it;
  }

  value_type operator*() const { return {}; }
};

```

这段代码定义了一个名为“truncating_iterator_base”的模板类，属于输出迭代器（iterator）的子类。这个模板类有两个保护成员变量：“out_”和“limit_”，分别用于存储输出迭代器和最大迭代次数。另一个成员变量“count_”用于记录已经遍历过的迭代次数。

该模板类的构造函数接受两个参数：一个输出迭代器（out_）和一个最大迭代次数（limit_）。函数内部，将这两个参数设置为输出迭代器和最大迭代次数，同时将“count_”初始化为0。

该模板类定义了一系列输出迭代器的辅助类型，包括：使用“std::output_iterator_tag”定义的“iterator_category”类型，使用“typename std::iterator_traits<OutputIt>::value_type”定义的“value_type”类型，以及“void”定义的“difference_type”类型。

该模板类还定义了一个名为“_Unchecked_type”的成员变量，将其声明为使用“truncating_iterator_base”类型的“_Unchecked_type”。这个成员变量被标记为使用“_Unchecked_type”类型的检查，这意味着在使用时需要手动解除检查。


```cpp
template <typename OutputIt> class truncating_iterator_base {
 protected:
  OutputIt out_;
  size_t limit_;
  size_t count_;

  truncating_iterator_base(OutputIt out, size_t limit)
      : out_(out), limit_(limit), count_(0) {}

 public:
  using iterator_category = std::output_iterator_tag;
  using value_type = typename std::iterator_traits<OutputIt>::value_type;
  using difference_type = void;
  using pointer = void;
  using reference = void;
  using _Unchecked_type =
      truncating_iterator_base;  // Mark iterator as checked.

  OutputIt base() const { return out_; }
  size_t count() const { return count_; }
};

```

这段代码定义了一个模板类 called "truncating\_iterator"，用于输出一个迭代器（可以理解为一个输出流）并统计到该迭代器结束时输出的对象数。

这个模板类有两个模板参数：一个 "OutputIt" 类型表示输出流，另一个是一个可以忽略的 "Enable" 类型，当它为真时，将使用一个特殊的 "OutputIt" 类型来表示可以迭代输出。

模板类 implementation 中，对于 "OutputIt" 类型，定义了一个黑色的 "blackhole" 成员变量，用来表示一个未迭代过的对象。这个黑色的 "blackhole" 成员变量在 "truncating\_iterator" 类的实现中没有被使用，但是需要在模板定义中进行定义。

在 "truncating\_iterator" 类的实现中，首先定义了 "truncating\_iterator" 类的 base 模板类，然后在此基础上实现了 operator++ 和 operator++(int) 方法。

operator++ 方法，用于将迭代器向前移动一步，并更新迭代器的计数器。如果计数器小于 limit 参数，就执行这个操作，并将对象输出。

operator++(int) 方法，用于将迭代器向前移动一步，并返回一个新的迭代器（指向一个输出对象）。这个新迭代器将输出一个新的对象，代替原来的迭代器。


```cpp
// An output iterator that truncates the output and counts the number of objects
// written to it.
template <typename OutputIt,
          typename Enable = typename std::is_void<
              typename std::iterator_traits<OutputIt>::value_type>::type>
class truncating_iterator;

template <typename OutputIt>
class truncating_iterator<OutputIt, std::false_type>
    : public truncating_iterator_base<OutputIt> {
  mutable typename truncating_iterator_base<OutputIt>::value_type blackhole_;

 public:
  using value_type = typename truncating_iterator_base<OutputIt>::value_type;

  truncating_iterator(OutputIt out, size_t limit)
      : truncating_iterator_base<OutputIt>(out, limit) {}

  truncating_iterator& operator++() {
    if (this->count_++ < this->limit_) ++this->out_;
    return *this;
  }

  truncating_iterator operator++(int) {
    auto it = *this;
    ++*this;
    return it;
  }

  value_type& operator*() const {
    return this->count_ < this->limit_ ? *this->out_ : blackhole_;
  }
};

```

这段代码定义了一个模板类 called "truncating\_iterator"，用于在需要时截取输入 "OutputIt" 的值，并将其存储在 "OutputIt" 类型的数据成员 "out" 中。

这个模板类实现了 "truncating\_iterator\_base" 模板类，它包含一个 "OutputIt" 类型的 "out" 成员和一个 "limit" 类型的成员变量。其中 "limit" 是一个整数类型的变量，用于指定截取的最大值。

在 "truncating\_iterator" 的构造函数中，将 "out" 和 "limit" 作为参数传递给 "truncating\_iterator\_base" 的构造函数，并初始化 "count\_" 和 "limit\_" 成员变量。

在 "truncating\_iterator" 的操作符重载函数中，实现了 "++" 和 "++" 运算符的重载，用于 incre 成员变量 "count\_" 和 "limit\_"，并将 "val" 赋值给 "out"。

由于 "truncating\_iterator" 模板类中的 "size\_t" 成员变量 "limit" 是一个整数类型，因此 "truncating\_iterator" 可以在编译器生成的上下文中使用，并且可以接受任何 "OutputIt" 类型的参数。


```cpp
template <typename OutputIt>
class truncating_iterator<OutputIt, std::true_type>
    : public truncating_iterator_base<OutputIt> {
 public:
  truncating_iterator(OutputIt out, size_t limit)
      : truncating_iterator_base<OutputIt>(out, limit) {}

  template <typename T> truncating_iterator& operator=(T val) {
    if (this->count_++ < this->limit_) *this->out_++ = val;
    return *this;
  }

  truncating_iterator& operator++() { return *this; }
  truncating_iterator& operator++(int) { return *this; }
  truncating_iterator& operator*() { return *this; }
};

```

这段代码定义了两个版本的函数，一个针对`basic_string_view<Char>`类型，另一个针对`basic_string_view<char>`类型。这两个版本的函数都返回字符串中代码点的数量。

函数的实现比较简单：对于给定的字符串，它们通过遍历字符数组并检查特定位是否为`0xc0`（即ASCII码中的`\u0b0c`）来确定是否为代码点。如果是，则将计数器`num_code_points`的值`++`。最后函数返回`num_code_points`。

对于`basic_string_view<char>`类型，需要在数据串中查找从`0x20`（ASCII码中的`\u0b20`）到`0x7f`（ASCII码中的`\u2800`）的子串，因为这些子串中确实包含有多个连续的`\u0b0c`编码。对于`basic_string_view<Char>`类型，只需要在数据串中查找从`0x20`到`\u2800`的子串。

请注意，这两个版本的函数实现是相互独立的，并且都可以在不需要修改的情况下单独使用。


```cpp
template <typename Char>
inline size_t count_code_points(basic_string_view<Char> s) {
  return s.size();
}

// Counts the number of code points in a UTF-8 string.
inline size_t count_code_points(basic_string_view<char> s) {
  const char* data = s.data();
  size_t num_code_points = 0;
  for (size_t i = 0, size = s.size(); i != size; ++i) {
    if ((data[i] & 0xc0) != 0x80) ++num_code_points;
  }
  return num_code_points;
}

```

这段代码定义了两个函数，分别是 `count_code_points` 和 `code_point_index`。这两个函数都接受一个 `basic_string_view<char8_type>` 类型的参数 `s`，并返回一个整数类型的参数。

`count_code_points` 函数的实现比较复杂，它接受一个字符串view，并计算出字符串中所有代码点的编号（即计数）。具体实现包括以下几个步骤：

1. 首先，通过 `reinterpret_cast<const char*>(s.data())` 将 `s` 中的字符串转换为指向字符数组原型的指针；
2. 使用这个指针，调用 `count_code_points` 函数，并传入这个指针和 `s` 的大小，得到当前代码点数量；
3. 初始化一个 `num_code_points` 变量为0；
4. 遍历 `s` 中的字符，对于每个字符，首先判断它是否为不可见字符（即 ASCII 码 0xc0 ），如果是，则说明当前代码点数量已经超出了最大允许数量 `n`，此时返回当前计数器；
5. 如果当前字符不是不可见字符，就递增 `num_code_points` 计数器。

`code_point_index` 函数的实现比较简单，它接受一个字符串view，和一个整数 `n`，并返回字符串中第 `n` 个代码点的编号（即计数）。具体实现包括以下几个步骤：

1. 首先，通过 `reinterpret_cast<const char*>(s.data())` 将 `s` 中的字符串转换为指向字符数组原型的指针；
2. 使用这个指针，调用 `code_point_index` 函数，并传入这个指针和 `s` 的大小，得到当前代码点数量；
3. 如果 `n` 小于当前代码点数量，就返回 `n`；
4. 如果 `n` 大于等于当前代码点数量，就返回当前代码点数量；
5. 最后返回当前代码点数量。


```cpp
inline size_t count_code_points(basic_string_view<char8_type> s) {
  return count_code_points(basic_string_view<char>(
      reinterpret_cast<const char*>(s.data()), s.size()));
}

template <typename Char>
inline size_t code_point_index(basic_string_view<Char> s, size_t n) {
  size_t size = s.size();
  return n < size ? n : size;
}

// Calculates the index of the nth code point in a UTF-8 string.
inline size_t code_point_index(basic_string_view<char8_type> s, size_t n) {
  const char8_type* data = s.data();
  size_t num_code_points = 0;
  for (size_t i = 0, size = s.size(); i != size; ++i) {
    if ((data[i] & 0xc0) != 0x80 && ++num_code_points > n) {
      return i;
    }
  }
  return s.size();
}

```

这段代码定义了一个名为 `needs_conversion` 的模板判断，用于判断两个条件是否满足：

1. 第一个条件是 `std::is_same<typename std::iterator_traits<InputIt>::value_type,
                                         char>::value` 是否成立，如果成立，说明 `InputIt` 和 `OutChar` 的值类型相同，可以进行输入输出操作。
2. 第二个条件是 `std::is_same<OutChar, char8_type>::value` 是否成立，如果成立，说明 `OutChar` 的值类型为 `char8_type` 类型，即 `OutChar` 需要进行编码和解码等操作。

通过这两个条件的共同作用，来判断是否需要对 `InputIt` 和 `OutChar` 进行特定的转换，然后就可以根据这个判断结果实现相应的复制操作。


```cpp
template <typename InputIt, typename OutChar>
using needs_conversion = bool_constant<
    std::is_same<typename std::iterator_traits<InputIt>::value_type,
                 char>::value &&
    std::is_same<OutChar, char8_type>::value>;

template <typename OutChar, typename InputIt, typename OutputIt,
          FMT_ENABLE_IF(!needs_conversion<InputIt, OutChar>::value)>
OutputIt copy_str(InputIt begin, InputIt end, OutputIt it) {
  return std::copy(begin, end, it);
}

template <typename OutChar, typename InputIt, typename OutputIt,
          FMT_ENABLE_IF(needs_conversion<InputIt, OutChar>::value)>
OutputIt copy_str(InputIt begin, InputIt end, OutputIt it) {
  return std::transform(begin, end, it,
                        [](char c) { return static_cast<char8_type>(c); });
}

```

这段代码定义了一个名为`copy_str`的函数模板，其参数为`Char`和`InputIt`类型。函数模板包含两个内部函数，一个叫做`counting_iterator`和一个叫做`copy_str`的函数。

`counting_iterator`是一个模板别名，这个别名被用来输出一个整数计数器迭代器的起始和结束位置之间的距离，这个距离是由`end`减去`begin`得到的。

`copy_str`函数的作用是将输入字符串的所有字符逆序后，返回一个新的字符计数器迭代器。这个新的计数器迭代器包含了原始字符串中所有字符的位置偏移量，包括正向和负向的偏移量。这个函数可以通过将原始字符串的起始和结束位置存储在一个`counting_iterator`变量中得到更好的性能。

接下来，定义了一个名为`is_fast_float`的函数模板，这个模板使用了`std::numeric_limits`和`sizeof`的特性，判断一个类型是否为`std::numeric_limits<T>::is_iec559`类型，如果是，则`is_fast_float`返回`true`，否则返回`false`。

最后，定义了一个名为`fmt_use_full_cache_dragonsboox`的宏，它输出一个布尔值，用于判断是否使用完整的缓存 dragonsboox。如果使用完整的缓存 dragonsboox，那么输出为`0`，否则输出为`1`。


```cpp
template <typename Char, typename InputIt>
inline counting_iterator copy_str(InputIt begin, InputIt end,
                                  counting_iterator it) {
  return it + (end - begin);
}

template <typename T>
using is_fast_float = bool_constant<std::numeric_limits<T>::is_iec559 &&
                                    sizeof(T) <= sizeof(double)>;

#ifndef FMT_USE_FULL_CACHE_DRAGONBOX
#  define FMT_USE_FULL_CACHE_DRAGONBOX 0
#endif

template <typename T>
```

这段代码定义了两个模板类：buffer 和 iterator_buffer。buffer 类是一个模板类，可以用来管理一个可变大小的动态数据缓冲区，其成员函数 append() 用来自动管理缓冲区的写入操作。而 iterator_buffer 是 output 迭代器的派生类，它是一个模板类，可以用来管理一个可变大小的动态数据缓冲区，并支持标准库的输出迭代器（std::out_）。

buffer 模板类中的 append() 函数，顾名思义，是一个将指定位置至给定位置的缓冲区元素进行写入的函数。函数接受两个参数，一个是指针 begin，表示缓冲区起始位置，一个是指针 end，表示缓冲区结束位置（但不包括该位置）。函数内部通过循环，将 begin 和 end 之间的元素，尽最大努力地从 begin 开始复制到 end 结束，同时，当缓冲区空间不足时，将使用 make_checked() 函数来扩充缓冲区空间，即创建一个比当前缓冲区大小更大的空闲缓冲区，将剩余的元素复制到该位置，并更新缓冲区大小和 begin 指向的位置。

而 iterator_buffer 模板类中的 flush() 函数，用于清空缓冲区并输出所有元素。该函数同样接受两个参数，一个是指向 std::out_ 的输出迭代器，另一个是当前缓冲区的大小。函数内部通过循环，将 std::out_ 中的元素复制到当前缓冲区中，并清空缓冲区和输出迭代器。


```cpp
template <typename U>
void buffer<T>::append(const U* begin, const U* end) {
  do {
    auto count = to_unsigned(end - begin);
    try_reserve(size_ + count);
    auto free_cap = capacity_ - size_;
    if (free_cap < count) count = free_cap;
    std::uninitialized_copy_n(begin, count, make_checked(ptr_ + size_, count));
    size_ += count;
    begin += count;
  } while (begin != end);
}

template <typename OutputIt, typename T, typename Traits>
void iterator_buffer<OutputIt, T, Traits>::flush() {
  out_ = std::copy_n(data_, this->limit(this->size()), out_);
  this->clear();
}
}  // namespace detail

```

这段代码定义了一个名为 `basic_memory_buffer` 的枚举类型，其值为 `inline_buffer_size`，用于在程序中占用基本内存空间以避免动态内存分配。这个内存缓冲区可以用来存储任意可复制和可构造的类型，如字符和字符串等，其元素存储在对象本身以实现快速的复制和构造。

为了简洁起见，该代码省略了变量和函数体的定义，留下了类型定义。通过使用这个内存缓冲区，可以方便地存储和操作字符和字符串类型的数据，而不需要显式地指定内存分配和管理。例如，可以创建一个 `basic_memory_buffer` 对象，通过调用其方法来添加、复制和查询字符串内容，而不需要担心内存分配和释放的问题。


```cpp
// The number of characters to store in the basic_memory_buffer object itself
// to avoid dynamic memory allocation.
enum { inline_buffer_size = 500 };

/**
  \rst
  A dynamically growing memory buffer for trivially copyable/constructible types
  with the first ``SIZE`` elements stored in the object itself.

  You can use one of the following type aliases for common character types:

  +----------------+------------------------------+
  | Type           | Definition                   |
  +================+==============================+
  | memory_buffer  | basic_memory_buffer<char>    |
  +----------------+------------------------------+
  | wmemory_buffer | basic_memory_buffer<wchar_t> |
  +----------------+------------------------------+

  **Example**::

     fmt::memory_buffer out;
     format_to(out, "The answer is {}.", 42);

  This will append the following output to the ``out`` object:

  .. code-block:: none

     The answer is 42.

  The output can be converted to an ``std::string`` with ``to_string(out)``.
  \endrst
 */
```

This is a C++ class template that inherits from the `fmt::basic_memory_buffer` class. It is designed to be moved, meaning that if you have an instance of the class and you pass it to a function, the function should return a new copy.

The class has two member functions: `operator` and `resize`. The `operator` function takes a `basic_memory_buffer` object and returns a copy of the allocator associated with the object. The `resize` function resizes the buffer to contain `count` elements of type `T` and tries to reserve space for the new capacity.

The class also has two member functions for the `fmt::basic_memory_buffer` class: `try_resize` and `try_reserve`. These functions are used for expanding and contracting the buffer respectively.

The class also defines a slot for appending data to the buffer. This slot is implemented using the `append` function from the `fmt::basic_memory_buffer` class, which takes a range of elements and appends them to the buffer.


```cpp
template <typename T, size_t SIZE = inline_buffer_size,
          typename Allocator = std::allocator<T>>
class basic_memory_buffer : public detail::buffer<T> {
 private:
  T store_[SIZE];

  // Don't inherit from Allocator avoid generating type_info for it.
  Allocator alloc_;

  // Deallocate memory allocated by the buffer.
  void deallocate() {
    T* data = this->data();
    if (data != store_) alloc_.deallocate(data, this->capacity());
  }

 protected:
  void grow(size_t size) final FMT_OVERRIDE;

 public:
  using value_type = T;
  using const_reference = const T&;

  explicit basic_memory_buffer(const Allocator& alloc = Allocator())
      : alloc_(alloc) {
    this->set(store_, SIZE);
  }
  ~basic_memory_buffer() { deallocate(); }

 private:
  // Move data from other to this buffer.
  void move(basic_memory_buffer& other) {
    alloc_ = std::move(other.alloc_);
    T* data = other.data();
    size_t size = other.size(), capacity = other.capacity();
    if (data == other.store_) {
      this->set(store_, capacity);
      std::uninitialized_copy(other.store_, other.store_ + size,
                              detail::make_checked(store_, capacity));
    } else {
      this->set(data, capacity);
      // Set pointer to the inline array so that delete is not called
      // when deallocating.
      other.set(other.store_, 0);
    }
    this->resize(size);
  }

 public:
  /**
    \rst
    Constructs a :class:`fmt::basic_memory_buffer` object moving the content
    of the other object to it.
    \endrst
   */
  basic_memory_buffer(basic_memory_buffer&& other) FMT_NOEXCEPT { move(other); }

  /**
    \rst
    Moves the content of the other ``basic_memory_buffer`` object to this one.
    \endrst
   */
  basic_memory_buffer& operator=(basic_memory_buffer&& other) FMT_NOEXCEPT {
    FMT_ASSERT(this != &other, "");
    deallocate();
    move(other);
    return *this;
  }

  // Returns a copy of the allocator associated with this buffer.
  Allocator get_allocator() const { return alloc_; }

  /**
    Resizes the buffer to contain *count* elements. If T is a POD type new
    elements may not be initialized.
   */
  void resize(size_t count) { this->try_resize(count); }

  /** Increases the buffer capacity to *new_capacity*. */
  void reserve(size_t new_capacity) { this->try_reserve(new_capacity); }

  // Directly append data into the buffer
  using detail::buffer<T>::append;
  template <typename ContiguousRange>
  void append(const ContiguousRange& range) {
    append(range.data(), range.data() + range.size());
  }
};

```

这段代码定义了一个名为`basic_memory_buffer`的模板类，其使用参数包括三个类型参数`T`、`SIZE`和`Allocator`。

这个模板类定义了一个`grow`方法，它的功能是 Grow(变大)这个内存缓冲区的容量，如果缓冲区的当前容量(size)大于5000，则会抛出`std::runtime_error`。

在`grow`方法中，首先定义了两个变量`old_capacity`和`new_capacity`，它们都使用了`this->capacity()`方法计算出来。然后，判断新的容量`new_capacity`是否大于`old_capacity`除以2，如果是，则用新的容量更新`old_capacity`。

接着，在`new_capacity`中分配新的数据，使用`std::allocator_traits<Allocator>::allocate`方法，其中`alloc_`是一个成员变量，用于指定内存分配器。然后，通过`std::uninitialized_copy`复制老的数据到新分配的内存中。

最后，设置新分配的内存为当前的`size`大小，并设置`this->set`方法为`new_capacity`。`deallocate`方法也在`std::allocator_traits`的定义中，不过这个方法不会抛出错误，因此不会被用来在主函数中释放内存。


```cpp
template <typename T, size_t SIZE, typename Allocator>
void basic_memory_buffer<T, SIZE, Allocator>::grow(size_t size) {
#ifdef FMT_FUZZ
  if (size > 5000) throw std::runtime_error("fuzz mode - won't grow that much");
#endif
  size_t old_capacity = this->capacity();
  size_t new_capacity = old_capacity + old_capacity / 2;
  if (size > new_capacity) new_capacity = size;
  T* old_data = this->data();
  T* new_data =
      std::allocator_traits<Allocator>::allocate(alloc_, new_capacity);
  // The following code doesn't throw, so the raw pointer above doesn't leak.
  std::uninitialized_copy(old_data, old_data + this->size(),
                          detail::make_checked(new_data, new_capacity));
  this->set(new_data, new_capacity);
  // deallocate must not throw according to the standard, but even if it does,
  // the buffer already uses the new storage and will deallocate it in
  // destructor.
  if (old_data != store_) alloc_.deallocate(old_data, old_capacity);
}

```

这段代码定义了一个名为 is_contiguous 的模板类，该类使用 basic_memory_buffer 模板元编程器。

具体来说，这段代码定义了一个名为 T 的模板类型，该类型具有一个大小为 SIZE 的参数字符串，以及一个名为 Allocator 的模板参数。然后，is_contiguous 模板类包含一个名为 basic_memory_buffer 的模板元编程器，该元编程器使用 T 和 SIZE 参数来指定要存储的数据类型和大小。

接下来，该代码定义了一个名为 format_error 的类。该类包含一个构造函数，该构造函数接收一个字符串参数，用于存储错误信息。然后，该类包含一个析构函数，用于清理空对象。另外，该类还包含一个复制构造函数和拷贝运算符，用于在复制对象时初始化新对象。

最后，该代码还包含一个名为 print_error 的函数，用于打印给定的格式化错误信息。


```cpp
using memory_buffer = basic_memory_buffer<char>;
using wmemory_buffer = basic_memory_buffer<wchar_t>;

template <typename T, size_t SIZE, typename Allocator>
struct is_contiguous<basic_memory_buffer<T, SIZE, Allocator>> : std::true_type {
};

/** A formatting error such as invalid format string. */
FMT_CLASS_API
class FMT_API format_error : public std::runtime_error {
 public:
  explicit format_error(const char* message) : std::runtime_error(message) {}
  explicit format_error(const std::string& message)
      : std::runtime_error(message) {}
  format_error(const format_error&) = default;
  format_error& operator=(const format_error&) = default;
  format_error(format_error&&) = default;
  format_error& operator=(format_error&&) = default;
  ~format_error() FMT_NOEXCEPT FMT_OVERRIDE;
};

```

这段代码定义了一个模板类 named "detail" which contains two functions template <typename T> using 和 template <typename T, FMT_ENABLE_IF(is_signed<T>::value)> named "is\_negative".

模板的使用标准是在参数的说明中使用 FMT\_CONSTEXPR 保留，其中 FMT\_CONSTEXPR 是 C++17 中引入的可选项之一，用于将计算类型中的某些方面进行模板化。

函数 is\_negative 的实现基于两个模板参数 T 和 FMT\_ENABLE\_IF(is\_signed<T>::value)。第一个模板参数 T 是 int 类型或整型数据类型，第二个模板参数 FMT\_ENABLE\_IF(is\_signed<T>::value) 是模板元编程中的真 - 条件语句，用于检查 T 是否为 int 类型或有符号整型数据类型。

通过 is\_negative 函数，可以很方便地检查一个变量 T 是否为 int 类型或有符号整型数据类型，并且返回一个 false 以外的真值。如果变量 T 是 int 类型，则 is\_negative 将返回 false，否则将返回 true。


```cpp
namespace detail {

template <typename T>
using is_signed =
    std::integral_constant<bool, std::numeric_limits<T>::is_signed ||
                                     std::is_same<T, int128_t>::value>;

// Returns true if value is negative, false otherwise.
// Same as `value < 0` but doesn't produce warnings if T is an unsigned type.
template <typename T, FMT_ENABLE_IF(is_signed<T>::value)>
FMT_CONSTEXPR bool is_negative(T value) {
  return value < 0;
}
template <typename T, FMT_ENABLE_IF(!is_signed<T>::value)>
FMT_CONSTEXPR bool is_negative(T) {
  return false;
}

```

这段代码定义了一个模板类，名为`template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>`，其中`T`是模板参数，`FMT_CONSTEXPR`表示这是一个constant表达式，而`bool is_supported_floating_point(T)`则是一个返回类型为`bool`的函数。

函数体中，首先判断`T`是否为浮点数，如果是，则执行以下语句：

```cpp
   return (std::is_same<T, float>::value && FMT_USE_FLOAT) ||
        (std::is_same<T, double>::value && FMT_USE_DOUBLE) ||
        (std::is_same<T, long double>::value && FMT_USE_LONG_DOUBLE);
```

如果`T`不是浮点数，则执行以下语句：

```cpp
   return false;
```

然后，定义了一个名为`uint32_or_64_or_128_t`的模板类，其中`T`满足以下条件时，`uint32_or_64_or_128_t`会被推导出来：

```cpp
   conditional_t<num_bits<T>() <= 32 && !FMT_REDUCE_INT_INSTANTIATIONS,
                 uint32_t,
                 conditional_t<num_bits<T>() <= 64, uint64_t, uint128_t>>;
```

这里使用了条件判断来确定要使用的数据类型，如果`T`的位数小于等于32，则使用`uint32_t`类型，否则使用`uint64_t`类型，再根据`T`是浮点数还是整型来决定使用`float`类型还是`double`类型。


```cpp
template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
FMT_CONSTEXPR bool is_supported_floating_point(T) {
  return (std::is_same<T, float>::value && FMT_USE_FLOAT) ||
         (std::is_same<T, double>::value && FMT_USE_DOUBLE) ||
         (std::is_same<T, long double>::value && FMT_USE_LONG_DOUBLE);
}

// Smallest of uint32_t, uint64_t, uint128_t that is large enough to
// represent all values of an integral type T.
template <typename T>
using uint32_or_64_or_128_t =
    conditional_t<num_bits<T>() <= 32 && !FMT_REDUCE_INT_INSTANTIATIONS,
                  uint32_t,
                  conditional_t<num_bits<T>() <= 64, uint64_t, uint128_t>>;

```

这段代码定义了一个名为FMT_EXTERN_TEMPLATE_API的结构体，它是一个128位整数类型的内部表示。这个结构体包含一个默认构造函数，一个根据高字节和低字节创建的内部变量，以及一些成员函数，如加法运算符重载和值获取等。

FMT_USE_INT128这个宏表示这个结构体使用128位整数类型，并且在内部使用了int128类型。通过这个宏，我们可以使用原来定义在FMT_USE_INT128下的内部函数，而不是定义在FMT_EXTERN_TEMPLATE_API下的函数。

FMT_NOEXCEPT这个 macro 表示如果给定的高字节和低字节被认为是有效的，内部函数的实现就不会有错误。这个 macro 是声明内部函数时需要遵守的约定。


```cpp
// 128-bit integer type used internally
struct FMT_EXTERN_TEMPLATE_API uint128_wrapper {
  uint128_wrapper() = default;

#if FMT_USE_INT128
  uint128_t internal_;

  uint128_wrapper(uint64_t high, uint64_t low) FMT_NOEXCEPT
      : internal_{static_cast<uint128_t>(low) |
                  (static_cast<uint128_t>(high) << 64)} {}

  uint128_wrapper(uint128_t u) : internal_{u} {}

  uint64_t high() const FMT_NOEXCEPT { return uint64_t(internal_ >> 64); }
  uint64_t low() const FMT_NOEXCEPT { return uint64_t(internal_); }

  uint128_wrapper& operator+=(uint64_t n) & FMT_NOEXCEPT {
    internal_ += n;
    return *this;
  }
```

这段代码定义了一个名为`uint64_t high_`和`uint64_t low_`的变量，并创建了一个名为`uint128_wrapper`的函数。

`uint128_wrapper`函数接受两个`uint64_t`类型的参数`high`和`low`，并返回一个指向`uint128`类型变量结果的指针。

`uint64_t high() const FMT_NOEXCEPT`函数返回`high_`变量的值，类型为`uint64_t`。

`uint64_t low() const FMT_NOEXCEPT`函数返回`low_`变量的值，类型为`uint64_t`。

`uint128_wrapper& operator+=(uint64_t n) & FMT_NOEXCEPT`函数接受一个`uint64_t`类型的参数`n`，并返回一个`uint128_wrapper`类型的对象，类型为`uint64_t`。

该函数通过在`uint64_t`类型的变量`low_`上递增`n`，并使用`_addcarry_u64`函数来处理溢出，然后将`low_`的值更新为`carry`的值，最后将`high_`的值更新为`carry`的值。这样，`uint128_wrapper`对象将`low_`上的值递增`n`，但不会超过`uint64_t`的最大值。


```cpp
#else
  uint64_t high_;
  uint64_t low_;

  uint128_wrapper(uint64_t high, uint64_t low) FMT_NOEXCEPT : high_{high},
                                                              low_{low} {}

  uint64_t high() const FMT_NOEXCEPT { return high_; }
  uint64_t low() const FMT_NOEXCEPT { return low_; }

  uint128_wrapper& operator+=(uint64_t n) & FMT_NOEXCEPT {
#  if defined(_MSC_VER) && defined(_M_X64)
    unsigned char carry = _addcarry_u64(0, low_, n, &low_);
    _addcarry_u64(carry, high_, 0, &high_);
    return *this;
```

这段代码是一个C++的函数，用于计算一个64位无符号整数在模幂运算下的结果，并返回该结果的指针。

该函数名为`else`，如果输入值是整数，则执行以下计算步骤：

1. 将低位和输入的整数相加，并将结果存储在`sum`中。
2. 如果`sum`小于输入的整数，则在最高位添加1，并将结果存储在`high_`中。
3. 将最低位设置为`sum`。
4. 返回输入指针指向的结果。

如果输入值不是整数，则执行以下操作：

1. 将输入的整数强制转换为整数类型。
2. 将输入的整数除以2，并记录商`max_quotient`和余数`remainder`。
3. 如果`remainder`大于零，则在最高位添加1，并将结果存储在`high_`中。
4. 在最低位中执行以下操作：
  1. 将`remainder`乘以2，并将结果存储在`low_`中。
  2. 将`remainder`设置为`0`。
  3. 返回输入指针指向的结果。

该函数使用了一个名为`divtest_table_entry`的模板结构体，其中包含两个成员变量，`mod_inv`表示模块反向操作数的整数表示，`max_quotient`表示最大商和余数。

该函数的作用是计算一个整数在模幂运算下的结果，并返回该结果的指针。它可以在需要时通过输入整数来计算，也可以通过输入反向操作数来计算。


```cpp
#  else
    uint64_t sum = low_ + n;
    high_ += (sum < low_ ? 1 : 0);
    low_ = sum;
    return *this;
#  endif
  }
#endif
};

// Table entry type for divisibility test used internally
template <typename T> struct FMT_EXTERN_TEMPLATE_API divtest_table_entry {
  T mod_inv;
  T max_quotient;
};

```

这段代码定义了一个名为`basic_data`的模板结构体，其中`T`是一个可以是`void`的类型变量。这个模板结构体定义了一些静态成员变量，包括：

- `powers_of_10_64`是一个8字节的整数数组，其中包含了以10为底数的64个对数，用于计算1到10的整数部分。
- `zero_or_powers_of_10_32`是一个8字节的整数数组，其中包含了以10为底数的32个零和以1为底数的32个对数，用于计算1到10的浮点部分。
- `zero_or_powers_of_10_64`是一个8字节的整数数组，其中包含了以10为底数的64个零和以1为底数的64个对数，用于计算1到10的浮点部分。
- `grisu_pow10_significands`是一个16字节的整数数组，其中包含了以10为底数的4个对数，用于计算1到10的整数部分。
- `grisu_pow10_exponents`是一个16字节的整数数组，其中包含了以10为底数的4个对数，用于计算1到10的浮点部分。
- `divtest_table_for_pow5_32`是一个包含4个整数元素的数组，用于计算5的幂等于32的整数部分。
- `divtest_table_for_pow5_64`是一个包含4个整数元素的数组，用于计算5的幂等于64的整数部分。
- `dragonbox_pow10_significands_64`是一个8字节的整数数组，其中包含了以10为底数的64个对数，用于计算1到10的整数部分。
- `dragonbox_pow10_recovery_errors`是一个8字节的整数数组，其中包含了由于使用DragonBoxpow10而导致的 pow10 整数恢复错误。


```cpp
// Static data is placed in this class template for the header-only config.
template <typename T = void> struct FMT_EXTERN_TEMPLATE_API basic_data {
  static const uint64_t powers_of_10_64[];
  static const uint32_t zero_or_powers_of_10_32[];
  static const uint64_t zero_or_powers_of_10_64[];
  static const uint64_t grisu_pow10_significands[];
  static const int16_t grisu_pow10_exponents[];
  static const divtest_table_entry<uint32_t> divtest_table_for_pow5_32[];
  static const divtest_table_entry<uint64_t> divtest_table_for_pow5_64[];
  static const uint64_t dragonbox_pow10_significands_64[];
  static const uint128_wrapper dragonbox_pow10_significands_128[];
  // log10(2) = 0x0.4d104d427de7fbcc...
  static const uint64_t log10_2_significand = 0x4d104d427de7fbcc;
#if !FMT_USE_FULL_CACHE_DRAGONBOX
  static const uint64_t powers_of_5_64[];
  static const uint32_t dragonbox_pow10_recovery_errors[];
```

这段代码是一个C语言的定义，定义了一些常量，包括数字 pairs、字符串常量、颜色常量等，以及一些靜態变量和函數，下面是具体的解释：

1. `#ifdef` 和 `#define` 之间的内容是注释，不会被编译器解释执行，所以可以简单地略过不看。

2. `using digit_pair = char[2];` 定义了一个名为 `digit_pair` 的字符型变量，并将其初始化为两个字符 `'0'` 和 `'1'`。

3. `static const digit_pair digits[];` 定义了一个名为 `digits` 的字符型数组，用于表示数字 pairs。

4. `static const char hex_digits[];` 定义了一个名为 `hex_digits` 的字符型数组，用于表示 hexadecimal digits。

5. `static const char foreground_color[];` 定义了一个名为 `foreground_color` 的字符型数组，用于表示前景颜色。

6. `static const char background_color[];` 定义了一个名为 `background_color` 的字符型数组，用于表示背景颜色。

7. `static const char reset_color[5];` 定义了一个名为 `reset_color` 的字符型数组，包含 5 个字符，用于表示复位颜色。

8. `static const wchar_t wreset_color[5];` 定义了一个名为 `wreset_color` 的字符型数组，包含 5 个字符，用于表示预设颜色。

9. `static const char signs[];` 定义了一个名为 `signs` 的字符型数组，用于表示符号，常见的符号有 `'+'`、`'='`、`'!'`、`'%'` 等。

10. `static const char left_padding_shifts[5];` 定义了一个名为 `left_padding_shifts` 的字符型数组，用于表示左对齐的偏移量，常见的情况下为 5。

11. `static const char right_padding_shifts[5];` 定义了一个名为 `right_padding_shifts` 的字符型数组，用于表示右对齐的偏移量，常见的情况下为 5。

12. `return` 后面的内容是一个函数，返回符号类型(`'(' || ')'`)，表示在某些 C 语言标准库函数中返回符号值时使用。


```cpp
#endif
  // GCC generates slightly better code for pairs than chars.
  using digit_pair = char[2];
  static const digit_pair digits[];
  static const char hex_digits[];
  static const char foreground_color[];
  static const char background_color[];
  static const char reset_color[5];
  static const wchar_t wreset_color[5];
  static const char signs[];
  static const char left_padding_shifts[5];
  static const char right_padding_shifts[5];
};

// Maps bsr(n) to ceil(log10(pow(2, bsr(n) + 1) - 1)).
```

这段代码是一个函数，名为`bsr2log10`，其作用是解决GCC10中某个问题的一个特定场景。在没有`FMT_EXPORTED`这个标志时，它可以被当做库函数来使用。

在这个函数中，`int`类型变量`bsr`被作为参数传递给函数，然后通过一个静态的常量数组`data`来查找相应的值。如果`bsr`匹配数组中的任何一个元素，那么函数返回该元素的值。

这里的静态常量数组`data`被定义在一个外部函数模板中，如果没有定义`FMT_EXPORTED`，那么这个函数也应该是静态的。这个模板定义了一个`basic_data`结构体，但这个结构体在这里没有被使用。


```cpp
// This is a function instead of an array to workaround a bug in GCC10 (#1810).
FMT_INLINE uint16_t bsr2log10(int bsr) {
  static constexpr uint16_t data[] = {
      1,  1,  1,  2,  2,  2,  3,  3,  3,  4,  4,  4,  4,  5,  5,  5,
      6,  6,  6,  7,  7,  7,  7,  8,  8,  8,  9,  9,  9,  10, 10, 10,
      10, 11, 11, 11, 12, 12, 12, 13, 13, 13, 13, 14, 14, 14, 15, 15,
      15, 16, 16, 16, 16, 17, 17, 17, 18, 18, 18, 19, 19, 19, 19, 20};
  return data[bsr];
}

#ifndef FMT_EXPORTED
FMT_EXTERN template struct basic_data<void>;
#endif

// This is a struct rather than an alias to avoid shadowing warnings in gcc.
```

这段代码定义了一个名为 `data` 的结构体，该结构体使用了 `basic_data` 模板元编程语言特性。

该代码中定义了一个名为 `count_digits` 的函数，它接受一个 `uint64_t` 类型的参数 `n`。该函数返回 `n` 中数字的位数，包括前导零和不包括前导零的数字位数。

函数的实现基于两个条件：`__builtin_clzll` 和 `FMT_BUILTIN_CLZLL`。如果 `__builtin_clzll` 可用，那么函数实现就是通过 `count_digits_impl` 函数计算得到的，这个函数是一个内部实现，不会在主函数中输出。如果 `__builtin_clzll` 不可用，那么函数实现就是通过 `count_digits_fallback` 函数计算得到的，这个函数包含在一个名为 `fmt_builtin_clzll.h` 的头文件中。


```cpp
struct data : basic_data<> {};

#ifdef FMT_BUILTIN_CLZLL
// Returns the number of decimal digits in n. Leading zeros are not counted
// except for n == 0 in which case count_digits returns 1.
inline int count_digits(uint64_t n) {
  // https://github.com/fmtlib/format-benchmark/blob/master/digits10
  auto t = bsr2log10(FMT_BUILTIN_CLZLL(n | 1) ^ 63);
  return t - (n < data::zero_or_powers_of_10_64[t]);
}
#else
// Fallback version of count_digits used when __builtin_clz is not available.
inline int count_digits(uint64_t n) {
  int count = 1;
  for (;;) {
    // Integer division is slow so do it for a group of four digits instead
    // of for every digit. The idea comes from the talk by Alexandrescu
    // "Three Optimization Tips for C++". See speed-test for a comparison.
    if (n < 10) return count;
    if (n < 100) return count + 1;
    if (n < 1000) return count + 2;
    if (n < 10000) return count + 3;
    n /= 10000u;
    count += 4;
  }
}
```

这段代码的作用是定义了一个名为`count_digits`的函数，其输入参数为一个128位无符号整数`n`，输出为一个整数类型的计数器`count`。

该函数采用分段方式实现，共设有4个循环，每个循环中首先判断输入的`n`是否小于10，如果是，则返回1；如果不是，则接着判断`n`是否小于100，如果是，则返回1+上一层循环的计数器，如果不是，则接着判断`n`是否小于1000，如果是，则返回1+上一层循环的计数器+2，如果不是，则接着判断`n`是否小于10000，如果是，则返回1+上一层循环的计数器+3。

循环条件的判断语句为：`n /= 10000U;` 它会将`n`除以10000，并将结果存储在整数类型的变量`n`中。

接下来，在循环体中，首先对输入的`n`进行整除运算，并将结果存储在整数类型的变量`n / 10000U`，然后执行循环体中的语句，对`n`进行累加操作，并将结果存储在整数类型的变量`count`中。

该函数实现了一个可以对128位无符号整数进行计数的功能，可以输出多个数字。


```cpp
#endif

#if FMT_USE_INT128
inline int count_digits(uint128_t n) {
  int count = 1;
  for (;;) {
    // Integer division is slow so do it for a group of four digits instead
    // of for every digit. The idea comes from the talk by Alexandrescu
    // "Three Optimization Tips for C++". See speed-test for a comparison.
    if (n < 10) return count;
    if (n < 100) return count + 1;
    if (n < 1000) return count + 2;
    if (n < 10000) return count + 3;
    n /= 10000U;
    count += 4;
  }
}
```

这段代码定义了一个模板函数 `count_digits`，它的参数是一个 `unsigned` 类型的整数 `n`，返回值为 `count_digits` 函数计算出的 `unsigned` 类型整数形式的数字中 `unsigned` 类型整数的位数。

该函数使用了 C++ 中的 `do-while` 循环和 `unsigned` 类型特殊的成员 `>>` 运算符（取对数）。函数的实现如下：
```cppcpp
#include <bits/stdc++.h>

// Counts the number of digits in n. BITS = log2(radix).
template <unsigned BITS, typename UInt> inline int count_digits(UInt n) {
 int num_digits = 0;
 do {
   ++num_digits;
 } while ((n >>= BITS) != 0);
 return num_digits;
}

// count_digits<4>(detail::fallback_uintptr n) 是模板实参函数，但它的实参类型是 `unsigned` 类型整数 `n`
```cpp
template <typename T>
void count_digits(T n) {
 int num_digits = 0;
 do {
   ++num_digits;
 } while ((n >>= 1) != 0);
 return num_digits;
}
```cpp
该代码中定义了一个名为 `count_digits` 的函数，该函数接受一个 `unsigned` 类型的整数作为参数，并返回该整数的 `unsigned` 类型整数的位数。该函数使用了 `do-while` 循环和 `unsigned` 类型特殊的成员 `>>` 运算符（取对数）来实现。

该函数的实现与问题中的描述一致，因此该函数的作用是计算一个 `unsigned` 类型的整数 `n` 的 `unsigned` 类型整数的位数。


```
#endif

// Counts the number of digits in n. BITS = log2(radix).
template <unsigned BITS, typename UInt> inline int count_digits(UInt n) {
  int num_digits = 0;
  do {
    ++num_digits;
  } while ((n >>= BITS) != 0);
  return num_digits;
}

template <> int count_digits<4>(detail::fallback_uintptr n);

#if FMT_GCC_VERSION || FMT_CLANG_VERSION
#  define FMT_ALWAYS_INLINE inline __attribute__((always_inline))
```cpp

这段代码定义了一系列头文件和定义，主要作用是定义了 FMT_ALWAYS_INLINE 和 FMT_SAFEBUFFERS 这两个宏，以及 FMT_BUILTIN_CLZ 变量。

FMT_ALWAYS_INLINE 是 FMT_SAFEBUFFERS 的别名，意味着 FMT_ALWAYS_INLINE 可以直接使用，而 FMT_SAFEBUFFERS 则不能直接使用。这两个宏的作用是定义了输出缓冲区的安全使用。

FMT_BUILTIN_CLZ 是一个可选的版本，用于在 32 位平台上计算 count_digits 函数。它定义了输出缓冲区最大长度，用于在某些情况下提高性能。


```
#elif FMT_MSC_VER
#  define FMT_ALWAYS_INLINE __forceinline
#else
#  define FMT_ALWAYS_INLINE inline
#endif

// To suppress unnecessary security cookie checks
#if FMT_MSC_VER && !FMT_CLANG_VERSION
#  define FMT_SAFEBUFFERS __declspec(safebuffers)
#else
#  define FMT_SAFEBUFFERS
#endif

#ifdef FMT_BUILTIN_CLZ
// Optional version of count_digits for better performance on 32-bit platforms.
```cpp

这段代码定义了一个名为 `count_digits` 的函数，它的参数是一个 32 位无符号整数 `n`。函数实现了一个对 `n` 的向下取整的算法，并返回这个取整结果减去 `n` 与 `data::zero_or_powers_of_10_32` 数组中第一个非零元素之间的差距。

接下来是三个头文件，它们定义了在不同输入类型的条件下，如何输出一个给定整数的个位数字分组。第一个头文件 `digits10.h` 在所有输入类型的条件下输出一个整数的个位数字分组。第二个头文件 `digits10_int.h` 在 `int` 类型的输入条件下输出整数的个位数字分组。第三个头文件 `digits10_uint.h` 在 `uint128_t` 类型的输入条件下输出整数的个位数字分组。

最后，是两个函数 `grouping_impl.h` 和 `grouping.h`。这两个函数都接受一个 `locale_ref` 类型的参数，并输出一个给定输入类型的字符串中的所有数字分组。两个函数的实现非常相似，只是使用了不同的输入类型和函数签名。


```
inline int count_digits(uint32_t n) {
  auto t = bsr2log10(FMT_BUILTIN_CLZ(n | 1) ^ 31);
  return t - (n < data::zero_or_powers_of_10_32[t]);
}
#endif

template <typename Int> constexpr int digits10() FMT_NOEXCEPT {
  return std::numeric_limits<Int>::digits10;
}
template <> constexpr int digits10<int128_t>() FMT_NOEXCEPT { return 38; }
template <> constexpr int digits10<uint128_t>() FMT_NOEXCEPT { return 38; }

template <typename Char> FMT_API std::string grouping_impl(locale_ref loc);
template <typename Char> inline std::string grouping(locale_ref loc) {
  return grouping_impl<char>(loc);
}
```cpp

这段代码定义了一系列模板类，用于将输入的文本数据按照特定的格式进行分组和转换。

`std::string grouping<wchar_t>(locale_ref loc)`是一个模板类，返回值类型为`std::string`，它将输入的`wchar_t`类型的字符串按照指定的`locale_ref`进行分组，并将分组后的字符串返回。

`template <typename Char> FMT_API Char thousands_sep_impl(locale_ref loc);`是一个模板类，其中`template <typename Char>`表示模板参数为`char`，`thousands_sep_impl<char>(loc)`是一个函数，它将输入的`locale_ref`指定的字符串中的`thousands_sep`子串转换成字符，并将结果返回。

类似地，`template <typename Char> inline Char thousands_sep(locale_ref loc)`也是一个模板类，其中`template <typename Char>`表示模板参数为`char`，`thousands_sep`是一个函数，它将输入的`locale_ref`指定的字符串中的`thousands_sep`子串转换成字符，并将结果返回。

`template <> inline wchar_t thousands_sep(locale_ref loc)`是一个模板类，其中`template <typename Char>`表示模板参数为`wchar_t`，`thousands_sep_impl<wchar_t>(loc)`是一个函数，它将输入的`locale_ref`指定的字符串中的`thousands_sep`子串转换成`wchar_t`，并将结果返回。

`template <typename Char> FMT_API Char decimal_point_impl(locale_ref loc);`是一个模板类，其中`template <typename Char>`表示模板参数为`char`，`decimal_point_impl<char>(loc)`是一个函数，它将输入的`locale_ref`指定的字符串中的`decimal_point`子串转换成字符，并将结果返回。

`template <typename Char> inline Char decimal_point(locale_ref loc)`是一个模板类，其中`template <typename Char>`表示模板参数为`char`，`decimal_point_impl<char>(loc)`是一个函数，它将输入的`locale_ref`指定的字符串中的`decimal_point`子串转换成字符，并将结果返回。

最后，`grouping_impl<wchar_t>(loc)`是一个函数，它使用`locale_ref`指定的`wchar_t`类型的字符串，按照`thousands_sep_impl<wchar_t>(loc)`中指定的规则进行分组，并将结果返回。


```
template <> inline std::string grouping<wchar_t>(locale_ref loc) {
  return grouping_impl<wchar_t>(loc);
}

template <typename Char> FMT_API Char thousands_sep_impl(locale_ref loc);
template <typename Char> inline Char thousands_sep(locale_ref loc) {
  return Char(thousands_sep_impl<char>(loc));
}
template <> inline wchar_t thousands_sep(locale_ref loc) {
  return thousands_sep_impl<wchar_t>(loc);
}

template <typename Char> FMT_API Char decimal_point_impl(locale_ref loc);
template <typename Char> inline Char decimal_point(locale_ref loc) {
  return Char(decimal_point_impl<char>(loc));
}
```cpp

这段代码定义了三个模板别名：

1. `wchar_t` 模板别名 `decimal_point` 是一个函数，采用 `locale_ref` 参数，返回一个 `wchar_t` 类型的浮点数(分数部分)。这个函数的作用是在 `std::cout` 中输出一个浮点数的数值，通常用于输出分数部分。

2. `bool` 模板别名 `equal2` 是一个函数，比较两个字符串是否相等，采用 `const char*` 参数。这个函数接收两个字符串 `lhs` 和 `rhs`，返回一个布尔值，表示两个字符串是否相等。

3. `void` 模板别名 `copy2` 是一个函数，接收两个字符 `src` 和 `dst`，从 `src` 开始复制一个字符到 `dst` 上，并把字符的值和地址一起复制。这个函数的作用是在 `std::string` 中复制一个字符串，通常用于在两个字符串之间复制字符。


```
template <> inline wchar_t decimal_point(locale_ref loc) {
  return decimal_point_impl<wchar_t>(loc);
}

// Compares two characters for equality.
template <typename Char> bool equal2(const Char* lhs, const char* rhs) {
  return lhs[0] == rhs[0] && lhs[1] == rhs[1];
}
inline bool equal2(const char* lhs, const char* rhs) {
  return memcmp(lhs, rhs, 2) == 0;
}

// Copies two characters from src to dst.
template <typename Char> void copy2(Char* dst, const char* src) {
  *dst++ = static_cast<Char>(*src++);
  *dst = static_cast<Char>(*src);
}
```cpp

这段代码定义了一个名为 `copy2` 的函数，其作用是将一个字符串 `src` 中的字符复制到另一个字符串 `dst` 中，但只复制前两个字符。

接下来定义了一个模板结构体 `format_decimal_result`，该结构体有两个成员变量 `begin` 和 `end`，分别指向写入数据的位置和数据长度。

最后，定义了两个模板函数，一个名为 `format_decimal`，用于将一个 `char` 类型的unsigned integer `value` 格式化为一个以指定大小的 `char` 指针 `out` 中，另一个名为 `fill_up_remainder`，用于将一个 `char` 指针 `out` 中的 `char` 循环填充到指定的 `char` 数组中，直到达到指定的 `char` 数组长度，并返回 `out` 和 `end` 所指向的 `char` 指针。


```
inline void copy2(char* dst, const char* src) { memcpy(dst, src, 2); }

template <typename Iterator> struct format_decimal_result {
  Iterator begin;
  Iterator end;
};

// Formats a decimal unsigned integer value writing into out pointing to a
// buffer of specified size. The caller must ensure that the buffer is large
// enough.
template <typename Char, typename UInt>
inline format_decimal_result<Char*> format_decimal(Char* out, UInt value,
                                                   int size) {
  FMT_ASSERT(size >= count_digits(value), "invalid digit count");
  out += size;
  Char* end = out;
  while (value >= 100) {
    // Integer division is slow so do it for a group of two digits instead
    // of for every digit. The idea comes from the talk by Alexandrescu
    // "Three Optimization Tips for C++". See speed-test for a comparison.
    out -= 2;
    copy2(out, data::digits[value % 100]);
    value /= 100;
  }
  if (value < 10) {
    *--out = static_cast<Char>('0' + value);
    return {out, end};
  }
  out -= 2;
  copy2(out, data::digits[value]);
  return {out, end};
}

```cpp

这段代码定义了两个模板别具功能：format_decimal_result 和 format_uint。

template <typename Char, typename UInt, typename Iterator,
         FMT_ENABLE_IF(!std::is_pointer<remove_cvref_t<Iterator>>::value)>
inline format_decimal_result<Iterator> format_decimal(Iterator out, UInt value,
                                                     int num_digits) {
 // Buffer should be large enough to hold all digits (<= digits10 + 1).
 enum { max_size = digits10<UInt>() + 1 };
 Char buffer[2 * max_size];
 auto end = format_decimal(buffer, value, num_digits).end;
 return {out, detail::copy_str<Char>(buffer, end, out)};
}

模板 <unsigned BASE_BITS, typename Char, typename UInt>
inline Char* format_uint(Char* buffer, UInt value, int num_digits,
                        bool upper = false) {
 buffer += num_digits;
 Char* end = buffer;
 do {
   const char* digits = upper ? "0123456789ABCDEF" : data::hex_digits;
   unsigned digit = (value & ((1 << BASE_BITS) - 1));
   *--buffer = static_cast<Char>(BASE_BITS < 4 ? static_cast<char>('0' + digit)
                                               : digits[digit]);
 } while ((value >>= BASE_BITS) != 0);
 return end;
}

这段代码首先定义了两个模板别具功能：format_decimal_result 和 format_uint。其中，template <typename Char, typename UInt, typename Iterator> 和 template <unsigned BASE_BITS, typename Char, typename UInt> 是模板别具的功能类型声明。

接着，定义了两个函数，分别为 format_decimal 和 format_uint。其中，format_decimal 的功能是输出一个字符类型的流，包含一个 decimal 形式的数值，函数参数包括一个输出迭代器 out、一个 UInt 类型的值 value 和一个整数 num_digits，函数内部通过循环将 num_digits 位的二进制数转换为字符，并输出到 out 中。而 format_uint 的功能是输出一个字符类型的流，包含一个 UInt 类型的值 value，函数参数包括一个字符指针 buffer、一个 UInt 类型的值 num_digits 和一个布尔值 upper，函数内部通过循环将 num_digits 位的二进制数转换为字符，并输出到 buffer 中。

最后，在函数内部使用 Char* 和 UInt* 分别来存储 buffer，由于需要输出字符类型的流，因此 Char* 是输出参数，而 UInt* 是存储参数。


```
template <typename Char, typename UInt, typename Iterator,
          FMT_ENABLE_IF(!std::is_pointer<remove_cvref_t<Iterator>>::value)>
inline format_decimal_result<Iterator> format_decimal(Iterator out, UInt value,
                                                      int num_digits) {
  // Buffer should be large enough to hold all digits (<= digits10 + 1).
  enum { max_size = digits10<UInt>() + 1 };
  Char buffer[2 * max_size];
  auto end = format_decimal(buffer, value, num_digits).end;
  return {out, detail::copy_str<Char>(buffer, end, out)};
}

template <unsigned BASE_BITS, typename Char, typename UInt>
inline Char* format_uint(Char* buffer, UInt value, int num_digits,
                         bool upper = false) {
  buffer += num_digits;
  Char* end = buffer;
  do {
    const char* digits = upper ? "0123456789ABCDEF" : data::hex_digits;
    unsigned digit = (value & ((1 << BASE_BITS) - 1));
    *--buffer = static_cast<Char>(BASE_BITS < 4 ? static_cast<char>('0' + digit)
                                                : digits[digit]);
  } while ((value >>= BASE_BITS) != 0);
  return end;
}

```cpp

这段代码定义了一个模板函数 `format_uint` ，它接受一个 `Char` 类型的输入参数 `buffer`，以及两个 `unsigned` 类型的参数 `n` 和 `num_digits`。函数用途是打印一个十进制整数，对传入的 `num_digits` 进行处理，如果 `num_digits` 小于 `BASE_BITS` 的位数，则将 `n` 的值逐步转换为对应 `BASE_BITS` 的 `unsigned char` 值，并将结果打印到 `buffer` 中。如果 `num_digits` 大于等于 `BASE_BITS` 的位数，则枚举 `BASE_BITS`，将 `unsigned_shift` 函数应用于 `n`，将结果打印到 `buffer` 中。


```
template <unsigned BASE_BITS, typename Char>
Char* format_uint(Char* buffer, detail::fallback_uintptr n, int num_digits,
                  bool = false) {
  auto char_digits = std::numeric_limits<unsigned char>::digits / 4;
  int start = (num_digits + char_digits - 1) / char_digits - 1;
  if (int start_digits = num_digits % char_digits) {
    unsigned value = n.value[start--];
    buffer = format_uint<BASE_BITS>(buffer, value, start_digits);
  }
  for (; start >= 0; --start) {
    unsigned value = n.value[start];
    buffer += char_digits;
    auto p = buffer;
    for (int i = 0; i < char_digits; ++i) {
      unsigned digit = (value & ((1 << BASE_BITS) - 1));
      *--p = static_cast<Char>(data::hex_digits[digit]);
      value >>= BASE_BITS;
    }
  }
  return buffer;
}

```cpp

这段代码定义了一个模板类It<unsigned BASE_BITS, typename Char, typename It, typename UInt>和两个函数utilsf8_to_utf16和utf8_to_utf16_converter。

It函数模板定义了一个可以输出字符串的函数format_uint。该函数接受一个输出迭代器out和一个表示uint的整数value，以及一个整数num_digits，布尔参数upper表示输出是否为大于8的数字类型。函数的作用是先将value和num_digits转换成字节数组，然后在内存中缓冲该字节数组，最后将缓冲区中的字节串通过out输出。

utf8_to_utf16函数是一个将UTF-8编码的字符串转换为UTF-16编码的字符串的工厂函数。该函数有一个私有成员变量buffer和一个public成员函数需要一个UTF-8编码的字符串对象s，并返回一个指向该字符串的引用。函数将字符串s中的所有字节转换为UTF-16编码的字符，并将结果存储在buffer中。函数的size函数返回buffer中字符的数量加1，因为需要存储最高位为0的字符。

utf8_to_utf16_converter函数是一个将UTF-8编码的字符串转换为UTF-16编码的字符串的函数实现。该函数接受一个需要转换的UTF-8编码的字符串对象s，并返回一个指向该字符串的引用。函数的实现将创建一个utf8_to_utf16类型的对象并调用其c_str函数来获取原始字符串，然后返回该字符串的引用。


```
template <unsigned BASE_BITS, typename Char, typename It, typename UInt>
inline It format_uint(It out, UInt value, int num_digits, bool upper = false) {
  // Buffer should be large enough to hold all digits (digits / BASE_BITS + 1).
  char buffer[num_bits<UInt>() / BASE_BITS + 1];
  format_uint<BASE_BITS>(buffer, value, num_digits, upper);
  return detail::copy_str<Char>(buffer, buffer + num_digits, out);
}

// A converter from UTF-8 to UTF-16.
class utf8_to_utf16 {
 private:
  wmemory_buffer buffer_;

 public:
  FMT_API explicit utf8_to_utf16(string_view s);
  operator wstring_view() const { return {&buffer_[0], size()}; }
  size_t size() const { return buffer_.size() - 1; }
  const wchar_t* c_str() const { return &buffer_[0]; }
  std::wstring str() const { return {&buffer_[0], size()}; }
};

```cpp

这段代码定义了一个模板结构体`null`，其中`T`可以是任何类型，但是没有定义任何成员函数或成员变量。接下来是一个`template`结构体`fill_t`，其中定义了一个私有枚举类型`max_size`和私有成员变量`data_`、`size_`。`fill_t`的构造函数接受一个`basic_string_view<Char>`的参数，并检查其大小是否超过`max_size`，如果是，则抛出错误。然后设置`data_`数组的大小为`size_`，并初始化其中元素的内容，使得数组大小等于`size_`。

`fill_t`还定义了一个成员函数`operator=(basic_string_view<Char> s)`，该函数接受一个`basic_string_view<Char>`的参数，并将其赋值给`T`类型的变量`this`，然后将`s`的大小转换为`T`类型的整数，并检查是否超过`max_size`。如果是，则抛出错误，否则将`data_`数组的内容复制到`this`中。

此外，`fill_t`还定义了一个成员函数`size()`和成员变量`data()`，分别返回数组大小和`data_`数组的内容。最后，`null`是一个模板结构体，定义了一个类型为`void`的模板类，其中包含了一个`fill_t`类型的成员变量`fill`，但是没有定义任何成员函数。


```
template <typename T = void> struct null {};

// Workaround an array initialization issue in gcc 4.8.
template <typename Char> struct fill_t {
 private:
  enum { max_size = 4 };
  Char data_[max_size];
  unsigned char size_;

 public:
  FMT_CONSTEXPR void operator=(basic_string_view<Char> s) {
    auto size = s.size();
    if (size > max_size) {
      FMT_THROW(format_error("invalid fill"));
      return;
    }
    for (size_t i = 0; i < size; ++i) data_[i] = s[i];
    size_ = static_cast<unsigned char>(size);
  }

  size_t size() const { return size_; }
  const Char* data() const { return data_; }

  FMT_CONSTEXPR Char& operator[](size_t index) { return data_[index]; }
  FMT_CONSTEXPR const Char& operator[](size_t index) const {
    return data_[index];
  }

  static FMT_CONSTEXPR fill_t<Char> make() {
    auto fill = fill_t<Char>();
    fill[0] = Char(' ');
    fill.size_ = 1;
    return fill;
  }
};
}  // namespace detail

```cpp

这段代码定义了一个名为 "align" 和 "sign" 的枚举类型，分别表示左侧、右侧、中和中心对齐以及正负号。同时，还定义了一个名为 "basic_format_specs" 的模板结构体，该结构体可以格式化字符类型的数据。

其中，basic_format_specs模板结构体包含了以下成员：

- 宽度（width）：对齐方式中x对齐的宽度，如果没有指定，则默认为0。
- 精度（precision）：对齐方式中x对齐的精度，如果没有指定，则默认为-1。
- 类型（type）：对齐方式中y的类型，如果没有指定，则默认为0。
- 对齐（align）：对齐方式，可以是align::none（默认，不进行对齐）、align::center（将字符中心对齐，仅适用于字符串类型的数据）、align::right（将字符向右对齐，仅适用于字符串类型的数据）、align::left（将字符向左对齐，仅适用于字符串类型的数据）或align::center、align::right、align::none。
- 正负号（sign）：正负号，可以是sign::none（默认，不进行正负号处理）、sign::plus（将字符的值视为正数）、sign::minus（将字符的值视为负数）或sign::none。
- 替代形式（alt）：如果当前对齐方式无法格式化字符串数据，则使用替代形式，其中#表示替代形式，'#'表示替代字符。
- 填充（fill）：用于对齐方式中的填充字符，当对齐方式中的某个成员为align::center、align::right、align::left时，该成员对应的填充字符定义在该结构体中。

该代码还定义了一个名为 "sign_t" 的枚举类型，与 "align_t" 类型对应，用于表示与左侧、右侧、中水平和numeric类型对齐的符号。


```
// We cannot use enum classes as bit fields because of a gcc bug
// https://gcc.gnu.org/bugzilla/show_bug.cgi?id=61414.
namespace align {
enum type { none, left, right, center, numeric };
}
using align_t = align::type;

namespace sign {
enum type { none, minus, plus, space };
}
using sign_t = sign::type;

// Format specifiers for built-in and string types.
template <typename Char> struct basic_format_specs {
  int width;
  int precision;
  char type;
  align_t align : 4;
  sign_t sign : 3;
  bool alt : 1;  // Alternate form ('#').
  detail::fill_t<Char> fill;

  constexpr basic_format_specs()
      : width(0),
        precision(-1),
        type(0),
        align(align::none),
        sign(sign::none),
        alt(false),
        fill(detail::fill_t<Char>::make()) {}
};

```cpp

这段代码定义了一个名为"float_info"的模板结构体，用于表示float类型的浮点数信息。这个模板结构体定义了一些类型特定的信息，如分母、指数、阶码等，用于在计算浮点数的值时进行转换和截断。

具体来说，这段代码定义了以下类型：

- "T" 类型，表示模板的参数类型，这个类型没有被具体定义，通常情况下会指向float类型。
- "float" 类型，表示要计算的浮点数类型。

接着，作者定义了一系列与浮点数计算相关的常量，如分母、指数、阶码、min_exponent、max_exponent等，这些常量在计算浮点数的值时会被用来进行转换和截断。

最后，作者还定义了一些与浮点数计算相关的模板型别，如 float_info<float>。这个模板型别定义了 float_info 模板结构体，其中包含了计算浮点数所需的所有类型特定的信息。


```
using format_specs = basic_format_specs<char>;

namespace detail {
namespace dragonbox {

// Type-specific information that Dragonbox uses.
template <class T> struct float_info;

template <> struct float_info<float> {
  using carrier_uint = uint32_t;
  static const int significand_bits = 23;
  static const int exponent_bits = 8;
  static const int min_exponent = -126;
  static const int max_exponent = 127;
  static const int exponent_bias = -127;
  static const int decimal_digits = 9;
  static const int kappa = 1;
  static const int big_divisor = 100;
  static const int small_divisor = 10;
  static const int min_k = -31;
  static const int max_k = 46;
  static const int cache_bits = 64;
  static const int divisibility_check_by_5_threshold = 39;
  static const int case_fc_pm_half_lower_threshold = -1;
  static const int case_fc_pm_half_upper_threshold = 6;
  static const int case_fc_lower_threshold = -2;
  static const int case_fc_upper_threshold = 6;
  static const int case_shorter_interval_left_endpoint_lower_threshold = 2;
  static const int case_shorter_interval_left_endpoint_upper_threshold = 3;
  static const int shorter_interval_tie_lower_threshold = -35;
  static const int shorter_interval_tie_upper_threshold = -35;
  static const int max_trailing_zeros = 7;
};

```cpp

这段代码定义了一个模板结构体 `float_info<double>`，用于表示浮点数的表示方式等信息。

这个模板结构体包括了一系列与浮点数相关的常量和类型，如 `carrier_uint`（表示一个64位无符号整数，用于表示浮点数的阶码） `exponent_bits`（表示指数的位数） `min_exponent`（表示浮点数的最小指数）和 `max_exponent`（表示浮点数的最大指数）等。

此外，还定义了一系列与浮点数相关的常量，如 `big_divisor`（表示大于1000的最大除数），`small_divisor`（表示小于1000的最大除数），`min_k`（表示浮点数的最小阶码），`max_k`（表示浮点数的最大阶码），以及与阶码相关的检查约束 `case_fc_pm_half_lower_threshold`、`case_fc_pm_half_upper_threshold`、`case_fc_lower_threshold` 和 `case_fc_upper_threshold` 等。

最后，还定义了一些与浮点数相关的类型，如 `divisibility_check_by_5_threshold`（表示浮点数是否可以被5整除，以决定是否需要检查阶码），`case_shorter_interval_left_endpoint_lower_threshold` 和 `case_shorter_interval_left_endpoint_upper_threshold`（表示是否需要截断浮点数，以决定截断位置），以及与浮点数相关的类型，如 `max_trailing_zeros`（表示浮点数有值的尾随零的个数）等。


```
template <> struct float_info<double> {
  using carrier_uint = uint64_t;
  static const int significand_bits = 52;
  static const int exponent_bits = 11;
  static const int min_exponent = -1022;
  static const int max_exponent = 1023;
  static const int exponent_bias = -1023;
  static const int decimal_digits = 17;
  static const int kappa = 2;
  static const int big_divisor = 1000;
  static const int small_divisor = 100;
  static const int min_k = -292;
  static const int max_k = 326;
  static const int cache_bits = 128;
  static const int divisibility_check_by_5_threshold = 86;
  static const int case_fc_pm_half_lower_threshold = -2;
  static const int case_fc_pm_half_upper_threshold = 9;
  static const int case_fc_lower_threshold = -4;
  static const int case_fc_upper_threshold = 9;
  static const int case_shorter_interval_left_endpoint_lower_threshold = 2;
  static const int case_shorter_interval_left_endpoint_upper_threshold = 3;
  static const int shorter_interval_tie_lower_threshold = -77;
  static const int shorter_interval_tie_upper_threshold = -77;
  static const int max_trailing_zeros = 16;
};

```cpp

这段代码定义了一个名为 "decimal_fp" 的结构体，用于表示一个带小数部分的浮点数。该结构体包括三个成员变量：整数部分（使用 "float_info<T>" 类型来获取）、小数部分（使用 "float_info<T>" 类型来获取）和一个指数。指数表示小数点左边有几个零，它用于计算小数部分的位数。

此外，还定义了一个名为 "to_decimal" 的函数模板，该函数将一个浮点数 "x" 转换为 "decimal_fp" 结构体类型的实例。

最后，定义了一个名为 "float_format" 的枚举类型，用于表示不同的浮点数表示形式。包括 "general"、"exp" 和 "fixed" 三个选项，分别表示使用浮点数注释、默认指数为 6 的指数表示法或固定点表示法。


```
template <typename T> struct decimal_fp {
  using significand_type = typename float_info<T>::carrier_uint;
  significand_type significand;
  int exponent;
};

template <typename T> decimal_fp<T> to_decimal(T x) FMT_NOEXCEPT;
}  // namespace dragonbox

// A floating-point presentation format.
enum class float_format : unsigned char {
  general,  // General: exponent notation or fixed point based on magnitude.
  exp,      // Exponent notation with the default precision of 6, e.g. 1.2e-3.
  fixed,    // Fixed point with the default precision of 6, e.g. 0.0012.
  hex
};

```cpp

这段代码定义了一个结构体 `float_specs`，其中包含了一些与浮点数表示相关的选项。

该结构体定义了一些整数类型选项，如 `int precision`, `float_format format`, `sign_t sign` 和 `bool upper`，表示输入浮点数的精度、数据类型和符号。还包括了一些选项，如 `bool locale` 和 `bool binary32`，表示是否使用特定的浮点数格式，以及是否使用符号表示负数。

该结构体还定义了一个模板函数 `write_exponent`，该函数将输入的指数转换为字符串。该函数接收两个参数，一个整数类型 `exp`，和一个 `It` 指针用于输出字符串。函数首先检查输入的指数是否在指定范围内，然后根据输入的符号来分别写正或负号，接着将指数转换为字符串，并按照指定的格式输出。

最后，该结构体还定义了一些辅助函数，如 `data::digits` 数组，其中包含了数字系统的所有数字，以及 `fmt:：庭数为8的FMT_ASSERT` 函数，用于在编译时检查输入的格式是否符合要求。


```
struct float_specs {
  int precision;
  float_format format : 8;
  sign_t sign : 8;
  bool upper : 1;
  bool locale : 1;
  bool binary32 : 1;
  bool use_grisu : 1;
  bool showpoint : 1;
};

// Writes the exponent exp in the form "[+-]d{2,3}" to buffer.
template <typename Char, typename It> It write_exponent(int exp, It it) {
  FMT_ASSERT(-10000 < exp && exp < 10000, "exponent out of range");
  if (exp < 0) {
    *it++ = static_cast<Char>('-');
    exp = -exp;
  } else {
    *it++ = static_cast<Char>('+');
  }
  if (exp >= 100) {
    const char* top = data::digits[exp / 100];
    if (exp >= 1000) *it++ = static_cast<Char>(top[0]);
    *it++ = static_cast<Char>(top[1]);
    exp %= 100;
  }
  const char* d = data::digits[exp];
  *it++ = static_cast<Char>(d[0]);
  *it++ = static_cast<Char>(d[1]);
  return it;
}

```cpp

这段代码定义了一个名为`format_float`的模板函数，接受一个浮点数`value`、一个整数`precision`，以及一个浮点数规格`specs`。

如果`specs`为`float_specs`，则`format_float`函数将使用`snprintf`函数格式化`value`为`precision`位数的浮点数，并将结果存储在以`buf`为底的缓冲区中。

如果`specs`包含一个或多个`float_specs`，则`format_float`函数将使用`handle_int_type_spec`函数处理输入的整数类型，根据输入的整数类型，调用相应的`on_*`函数，然后将结果直接传递给`handler`。

另外，还定义了一个名为`promote_float`的模板函数，接受一个浮点数`value`，将其直接返回。

以及一个名为`handle_int_type_spec`的函数，接受一个整数类型和一个`Handler`对象，根据输入的整数类型，将调用相应的`on_*`函数，并将结果直接传递给`handler`。


```
template <typename T>
int format_float(T value, int precision, float_specs specs, buffer<char>& buf);

// Formats a floating-point number with snprintf.
template <typename T>
int snprintf_float(T value, int precision, float_specs specs,
                   buffer<char>& buf);

template <typename T> T promote_float(T value) { return value; }
inline double promote_float(float value) { return static_cast<double>(value); }

template <typename Handler>
FMT_CONSTEXPR void handle_int_type_spec(char spec, Handler&& handler) {
  switch (spec) {
  case 0:
  case 'd':
    handler.on_dec();
    break;
  case 'x':
  case 'X':
    handler.on_hex();
    break;
  case 'b':
  case 'B':
    handler.on_bin();
    break;
  case 'o':
    handler.on_oct();
    break;
```cpp

这段代码是一个C语言中的一个函数，其中包含了一个条件分支。这个分支是为了检查函数被调用时所处的环境(即模板参数)，然后执行不同的操作。

具体来说，这段代码的作用是当函数被调用时，根据传递给函数的模板参数的类型，选择执行相应的操作，并跳转到相应的局部函数体。如果没有传递模板参数，则执行默认操作。

这段代码的逻辑比较简单，主要作用是用于函数音量的控制和处理。通过判断输入的音量是升还是降，来触发相应的音量处理，从而实现对音量的控制。


```
#ifdef FMT_DEPRECATED_N_SPECIFIER
  case 'n':
#endif
  case 'L':
    handler.on_num();
    break;
  case 'c':
    handler.on_chr();
    break;
  default:
    handler.on_error();
  }
}

template <typename ErrorHandler = error_handler, typename Char>
```cpp

这段代码定义了一个名为`float_specs`的函数，它接受一个名为`basic_format_specs`的函数参数，并返回一个名为`float_specs`的函数返回值。

函数体中首先定义了一个`float_specs`常量，它包含了一些与浮点数格式相关的常量。

然后，函数开始处理输入的`basic_format_specs`参数。对于每个输入的`type`属性，函数都会执行以下操作：

1. 如果`type`为0，则设置`result.format`为`float_format::general`，并设置`result.showpoint`为`specs.alt`，也就是输入的最低位有效位。

2. 如果`type`为`'G'`，则将`result.upper`设置为`true`，表示要输出千分位。然后跳过此行。

3. 如果`type`为`'g'`，则将`result.format`设置为`float_format::general`，这样就可以输出小数点。

4. 如果`type`为`'E'`，则将`result.upper`设置为`true`，表示要输出百分位。然后跳过此行。

5. 如果`type`为`'f'`，则将`result.format`设置为`float_format::fixed`，这样就可以输出小数点。

6. 如果`type`为`'A'`，则将`result.upper`设置为`true`，表示要输出百位。然后跳过此行。

7. 如果`type`为`'a'`，则将`result.format`设置为`float_format::hex`，这样就可以输出十六进制数字。

最后，函数返回`float_specs`常量。


```
FMT_CONSTEXPR float_specs parse_float_type_spec(
    const basic_format_specs<Char>& specs, ErrorHandler&& eh = {}) {
  auto result = float_specs();
  result.showpoint = specs.alt;
  switch (specs.type) {
  case 0:
    result.format = float_format::general;
    result.showpoint |= specs.precision > 0;
    break;
  case 'G':
    result.upper = true;
    FMT_FALLTHROUGH;
  case 'g':
    result.format = float_format::general;
    break;
  case 'E':
    result.upper = true;
    FMT_FALLTHROUGH;
  case 'e':
    result.format = float_format::exp;
    result.showpoint |= specs.precision != 0;
    break;
  case 'F':
    result.upper = true;
    FMT_FALLTHROUGH;
  case 'f':
    result.format = float_format::fixed;
    result.showpoint |= specs.precision != 0;
    break;
  case 'A':
    result.upper = true;
    FMT_FALLTHROUGH;
  case 'a':
    result.format = float_format::hex;
    break;
```cpp

这段代码是一个 C++ 中被动态链接库 FMT_DEPRECATED_N_SPECIFIER 包含的函数，它的作用是处理 Char 类型数据输出时的默认行为。

首先，定义了一个 FMT_CONSTEXPR 的函数 handle_char_specs，该函数接受两个参数：一个基本格式 specifications，一个 Handler 类型的参数。函数内部包含一个 if...else 语句，判断是否支持 Char 类型数据输出，如果是，就执行一系列操作，否则输出错误。

然后，定义了一个 FMT_CONSTEXPR 的函数 handle_int_specs，该函数与 handle_char_specs 类似，只是输出 int 类型数据。

最后，在 handle_char_specs 和 handle_int_specs 的体内部，都调用了构造函数 FMT_CONSTEXPR fmz_msg 并传入 its first argument，fmz_msg 在内部定义了一个名为 "msg" 的变量，并输出了一些辅助信息，如 FMT_DEBUG、FMT_CRITICAL、FMT_WARNING 等。输出信息的作用是帮助调试程序定位错误。

总之，这段代码的主要作用是帮助程序在输出 Char 类型数据时，根据不同的格式数据选择不同的默认行为，或者输出错误信息。


```
#ifdef FMT_DEPRECATED_N_SPECIFIER
  case 'n':
#endif
  case 'L':
    result.locale = true;
    break;
  default:
    eh.on_error("invalid type specifier");
    break;
  }
  return result;
}

template <typename Char, typename Handler>
FMT_CONSTEXPR void handle_char_specs(const basic_format_specs<Char>* specs,
                                     Handler&& handler) {
  if (!specs) return handler.on_char();
  if (specs->type && specs->type != 'c') return handler.on_int();
  if (specs->align == align::numeric || specs->sign != sign::none || specs->alt)
    handler.on_error("invalid format specifier for char");
  handler.on_char();
}

```cpp

这段代码定义了两个模板变量：

1. handle_cstring_type_spec。该模板变量有一个类型参数 Char，一个类型参数 Handler，以及一个返回类型 void。
2. check_string_type_spec。该模板变量有一个类型参数 Char，一个类型参数 ErrorHandler，以及一个返回类型 void。

handle_cstring_type_spec 和 check_string_type_spec 是用于检查输入参数中指定的 CString 类型是否符合预期的模板函数 handle_cstring_type_spec 的参数类型。

具体来说，handle_cstring_type_spec 会根据输入参数中的 CString 类型，将参数 on_string() 或 on_pointer() 函数调用，并将结果返回。而 check_string_type_spec 会检查输入参数中指定的 CString 类型是否符合 handle_cstring_type_spec 的参数类型，如果是错误的，函数体会输出一个错误并返回。


```
template <typename Char, typename Handler>
FMT_CONSTEXPR void handle_cstring_type_spec(Char spec, Handler&& handler) {
  if (spec == 0 || spec == 's')
    handler.on_string();
  else if (spec == 'p')
    handler.on_pointer();
  else
    handler.on_error("invalid type specifier");
}

template <typename Char, typename ErrorHandler>
FMT_CONSTEXPR void check_string_type_spec(Char spec, ErrorHandler&& eh) {
  if (spec != 0 && spec != 's') eh.on_error("invalid type specifier");
}

```cpp

这段代码定义了一个模板类 `int_type_checker`，它的作用是检查传入的 `ErrorHandler` 对象是否符合 `int` 类型的要求。

模板类 `int_type_checker` 中包含了一系列的 `FMT_CONSTEXPR` 类型的函数，它们用于在 `ErrorHandler` 对象中捕获异常信息并对其进行相应的处理。

具体来说，这段代码首先定义了一个模板参数为 `Char` 和 `ErrorHandler` 的函数 `check_pointer_type_spec`，它的作用是在传入 `spec` 参数时检查其是否为 `0` 或者 `'p'`，如果是，则抛出异常信息。

接着，定义了一个模板参数为 `ErrorHandler` 的类 `int_type_checker`，其中包含了一系列的 `FMT_CONSTEXPR` 类型的函数，这些函数用于在 `ErrorHandler` 对象中捕获异常信息并对其进行相应的处理。这些函数分别是 `on_dec`、`on_hex`、`on_bin`、`on_oct`、`on_num` 和 `on_chr`，它们用于在不同的输入类型下处理异常信息。

最后，在 `int_type_checker` 的 `on_error` 函数中，使用 `ErrorHandler` 的 `on_error` 函数来捕获异常信息，并输出一条异常信息。


```
template <typename Char, typename ErrorHandler>
FMT_CONSTEXPR void check_pointer_type_spec(Char spec, ErrorHandler&& eh) {
  if (spec != 0 && spec != 'p') eh.on_error("invalid type specifier");
}

template <typename ErrorHandler> class int_type_checker : private ErrorHandler {
 public:
  FMT_CONSTEXPR explicit int_type_checker(ErrorHandler eh) : ErrorHandler(eh) {}

  FMT_CONSTEXPR void on_dec() {}
  FMT_CONSTEXPR void on_hex() {}
  FMT_CONSTEXPR void on_bin() {}
  FMT_CONSTEXPR void on_oct() {}
  FMT_CONSTEXPR void on_num() {}
  FMT_CONSTEXPR void on_chr() {}

  FMT_CONSTEXPR void on_error() {
    ErrorHandler::on_error("invalid type specifier");
  }
};

```cpp

这段代码定义了一个名为 `char_specs_checker` 的模板类，旨在检查输入数据中的字符类型。该类包含两个私有成员变量 `type_` 和 `eh`。

该类的构造函数使用 `FMT_CONSTEXPR` 声明，参数包括两个类型参数 `type` 和 `ErrorHandler`。然后通过类型检查来确定 `type_` 的实际类型，并将 `eh` 作为另一个参数传递给构造函数。

该类有两个公共成员函数 `on_int()` 和 `on_char()`，都使用 `FMT_CONSTEXPR` 声明。这些函数用于处理输入数据中的字符类型，分别对应 `int_type_checker` 和 `ErrorHandler` 类的实例。

该类的实例化可以从函数 `char_specs_checker::char_specs_checker()` 开始。这个函数将 `type_` 初始化为所需的字符类型，并将 `eh` 传递给构造函数作为 `ErrorHandler` 的实例。

通过调用 `char_specs_checker::on_int()` 和 `char_specs_checker::on_char()` 函数，可以分别检查输入数据中的字符类型。


```
template <typename ErrorHandler>
class char_specs_checker : public ErrorHandler {
 private:
  char type_;

 public:
  FMT_CONSTEXPR char_specs_checker(char type, ErrorHandler eh)
      : ErrorHandler(eh), type_(type) {}

  FMT_CONSTEXPR void on_int() {
    handle_int_type_spec(type_, int_type_checker<ErrorHandler>(*this));
  }
  FMT_CONSTEXPR void on_char() {}
};

```cpp

这段代码定义了一个名为 cstring_type_checker 的模板类，该类继承自 ErrorHandler 类。在模板类中，有两个纯虚函数 on_string() 和 on_pointer()，用于在检查错误时执行相应的操作。

在纯虚函数 fill() 中，使用 const fill_t<Char> 类型的函数类型约束，表示要填充的字符串中的每个字符都只被填充一次。如果字符串中只包含一个字符，直接返回该字符对应的 fill 值即可。否则，遍历字符串中的每个字符，并将其复制到 fill 数组中。

该代码的作用是定义了一个用于检查字符串类型的错误的模板类，可以用来在编译时检查模板实参的类型是否与预期一致。通过在 fill() 函数中使用 const fill_t<Char> 类型的约束，可以确保 fill() 函数返回的字符串是唯一的，无论填充多少个字符。


```
template <typename ErrorHandler>
class cstring_type_checker : public ErrorHandler {
 public:
  FMT_CONSTEXPR explicit cstring_type_checker(ErrorHandler eh)
      : ErrorHandler(eh) {}

  FMT_CONSTEXPR void on_string() {}
  FMT_CONSTEXPR void on_pointer() {}
};

template <typename OutputIt, typename Char>
FMT_NOINLINE OutputIt fill(OutputIt it, size_t n, const fill_t<Char>& fill) {
  auto fill_size = fill.size();
  if (fill_size == 1) return std::fill_n(it, n, fill[0]);
  for (size_t i = 0; i < n; ++i) it = std::copy_n(fill.data(), fill_size, it);
  return it;
}

```cpp

这段代码定义了一个名为 `write_padded` 的函数，它的输入参数是一个输出迭代器 `out`，一个格式规范 `specs`，一个表示字符宽度的变量 `size`，以及一个表示输出显示宽度的大于零的变量 `width`。

函数实现了一个将输入的字符串 `f` 根据指定的格式规范 `specs` 进行填充并输出到输出迭代器 `out` 的过程中所需的操作。其中，填充的策略根据 `align` 参数的值决定：

* 如果 `align` 是 `align::left`，则使用数据成员 `data::left_padding_shifts` 中的偏移量填充；
* 如果 `align` 是 `align::right`，则使用数据成员 `data::right_padding_shifts` 中的偏移量填充。

该函数的实现依赖于另外两个辅助函数 `reserve` 和 `fill`，具体作用与该函数相同，此处不再赘述。


```
// Writes the output of f, padded according to format specifications in specs.
// size: output size in code units.
// width: output display width in (terminal) column positions.
template <align::type align = align::left, typename OutputIt, typename Char,
          typename F>
inline OutputIt write_padded(OutputIt out,
                             const basic_format_specs<Char>& specs, size_t size,
                             size_t width, const F& f) {
  static_assert(align == align::left || align == align::right, "");
  unsigned spec_width = to_unsigned(specs.width);
  size_t padding = spec_width > width ? spec_width - width : 0;
  auto* shifts = align == align::left ? data::left_padding_shifts
                                      : data::right_padding_shifts;
  size_t left_padding = padding >> shifts[specs.align];
  auto it = reserve(out, size + padding * specs.fill.size());
  it = fill(it, left_padding, specs.fill);
  it = f(it);
  it = fill(it, padding - left_padding, specs.fill);
  return base_iterator(out, it);
}

```cpp

这段代码定义了两个模板函数：write_padded 和 write_bytes。这两个函数都是接受一个输出迭代器（OutputIt）和一个字符串view对象（string_view）作为参数。

write_padded函数的实现主要是在重载了基本格式化控制器的功能上进行。通过重载输出迭代器的写入操作，使得当接收到一个字符串中不足size大小的字符时，可以进行字符串的截断，并将其存储到输出迭代器中。函数的实现在于为输出迭代器传入一个字符串view对象，然后将其中的所有字符输出到该迭代器中，如果不足以写出size个字符，则截断并将其存储到迭代器中。

write_bytes函数的实现主要是在重载了输出迭代器的写入操作上进行。通过重载write_padded函数，使得当接收到一个字符串view对象，且输出迭代器足够大时，可以将其中的所有字符输出到迭代器中。函数的实现在于为输出迭代器传入一个字符串view对象，然后将其中的所有字符输出到该迭代器中。

在这段代码中，还定义了一个类型align，它被用于在重载的类型参数中指定输出迭代器的宽度。通过类型的定义，可以确定align类型为align::left（左对齐）或align::right（右对齐）。


```
template <align::type align = align::left, typename OutputIt, typename Char,
          typename F>
inline OutputIt write_padded(OutputIt out,
                             const basic_format_specs<Char>& specs, size_t size,
                             const F& f) {
  return write_padded<align>(out, specs, size, size, f);
}

template <typename Char, typename OutputIt>
OutputIt write_bytes(OutputIt out, string_view bytes,
                     const basic_format_specs<Char>& specs) {
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  return write_padded(out, specs, bytes.size(), [bytes](iterator it) {
    const char* data = bytes.data();
    return copy_str<Char>(data, data + bytes.size(), it);
  });
}

```cpp

这段代码定义了一个模板结构体`write_int_data`，用于在输出中包含整数数据，但不会依赖于输出迭代器类型。

这个模板结构体有两个成员变量：`size_t`类型的`size`和`size_t`类型的`padding`。`size`表示整数的长度，`padding`表示在输入数据中始终保持的空白字符数。

`write_int_data`构造函数的参数包括一个整数`num_digits`，一个字符串`prefix`，和一个`basic_format_specs<Char>`的`align`属性。`align`用于指定输入数据对齐的方式。如果`align`是`align::numeric`，则此处的空白字符数将根据输入数据中`align`的值进行计算。如果`align`的值大于`num_digits`，则将`size`和`padding`都设置为`width - size`，其中`width`是`align`中指定的宽度。如果`align`的值小于`num_digits`，则将`size`设置为`padding`，并将`padding`设置为`align`中指定的精度与`num_digits`之差。

这个模板结构体可以被用于在输出中包含整数数据，而不必担心输出迭代器类型。例如，下面这段代码将输出一个整数：

```
int num = 42;
write_int_data<2> output("2", num, basic_format_specs<char>{align:：萤本色});
```cpp

这段代码将输出字符串`"2"`，其中整数`num`的值为42。


```
// Data for write_int that doesn't depend on output iterator type. It is used to
// avoid template code bloat.
template <typename Char> struct write_int_data {
  size_t size;
  size_t padding;

  write_int_data(int num_digits, string_view prefix,
                 const basic_format_specs<Char>& specs)
      : size(prefix.size() + to_unsigned(num_digits)), padding(0) {
    if (specs.align == align::numeric) {
      auto width = to_unsigned(specs.width);
      if (width > size) {
        padding = width - size;
        size = width;
      }
    } else if (specs.precision > num_digits) {
      size = prefix.size() + to_unsigned(specs.precision);
      padding = to_unsigned(specs.precision - num_digits);
    }
  }
};

```cpp

这段代码定义了一个名为`write_int`的函数，它接受一个输出迭代器`out`、一个整数`num_digits`、一个字符串`prefix`和一个格式化字符串`specs`，以及一个函数类型参数`F`。

函数的作用是将`F`函数的返回值赋值给`write_int`函数的返回值，其中`F`函数需要满足以下要求：

1. 返回类型为`void`。
2. 函数签名应该包含三个参数：一个输出迭代器`out`、一个整数`num_digits`和一个字符串`prefix`。
3. 函数需要将`F`函数的返回值赋值给`write_int`函数的返回值。

具体实现如下：

1. 首先定义一个名为`write_int_data`的函数，它的参数为`Char`类型，有两个参数：`num_digits`和`prefix`，还有一个格式化字符串`specs`。这个函数的作用是将`specs`中的格式化字符串解析成一个字符串，然后根据`specs`中的`<digits>`计算出`num_digits`中由`f(it)`得到的数值，并将解析出的字符串和`prefix`拼接成一个新的字符串，最后将解析出的数值输出。

2. 接下来定义一个名为`write_padded`的函数，它的参数为`OutputIt`类型、一个格式化字符串`specs`和一个字符串`prefix`，还有一个整数`num_digits`。这个函数的作用是在`write_int`函数的基础上对输出进行一定程度的填充，具体来说，它会尝试从`prefix`中复制一些字符，然后将`prefix`和`specs`中的字符串拼接成一个新的字符串，填充的字节数等于`specs`中`<digits>`的值，最后将填充后的字符串传递给`write_int`函数即可。

3. 最后定义`write_int`函数，它的参数为`OutputIt`类型、一个整数`num_digits`和一个字符串`prefix`，还有一个格式化字符串`specs`和一个函数类型参数`F`。这个函数的作用是将`F`函数的返回值赋值给`write_int`函数的返回值，具体来说，它会尝试从`specs`中解析出一个字符串，然后根据`specs`中的格式化字符串计算出`num_digits`中的数值，接着将解析出的数值输出，最后根据`specs`中的`<right-padding>`对输出进行一定程度的填充。


```
// Writes an integer in the format
//   <left-padding><prefix><numeric-padding><digits><right-padding>
// where <digits> are written by f(it).
template <typename OutputIt, typename Char, typename F>
OutputIt write_int(OutputIt out, int num_digits, string_view prefix,
                   const basic_format_specs<Char>& specs, F f) {
  auto data = write_int_data<Char>(num_digits, prefix, specs);
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  return write_padded<align::right>(out, specs, data.size, [=](iterator it) {
    if (prefix.size() != 0)
      it = copy_str<Char>(prefix.begin(), prefix.end(), it);
    it = std::fill_n(it, data.padding, static_cast<Char>('0'));
    return f(it);
  });
}

```cpp

这段代码定义了一个名为write的模板函数，其参数为输出迭代器out、字符串view s和格式化参数basic_format_specs<char> specs。

函数的实现主要步骤如下：

1. 从s的data()函数中获取字符串的内存指针data，以及s.size()函数获取字符串的长度size；
2. 如果specs.precision >= 0，且to_unsigned(specs.precision)<size，则根据specs.width计算输出字符数；
3. 如果specs.width != 0，则计算输出的宽度；
4. 定义一个名为iterator的变量，用于输出迭代器；
5. 使用write_padded函数对out参数进行预处理，将数据和宽度填充到out中；
6. 通过迭代器的copy_str()函数将字符串s复制到out中，注意输出字符串是以'\0'为结束标志的；
7. 返回write函数的返回值。

总之，write函数接受一个输出迭代器out、一个字符串view s和一个格式化参数basic_format_specs<char>，然后根据传入的参数进行一系列计算，最后输出字符串到该输出迭代器中。


```
template <typename StrChar, typename Char, typename OutputIt>
OutputIt write(OutputIt out, basic_string_view<StrChar> s,
               const basic_format_specs<Char>& specs) {
  auto data = s.data();
  auto size = s.size();
  if (specs.precision >= 0 && to_unsigned(specs.precision) < size)
    size = code_point_index(s, to_unsigned(specs.precision));
  auto width = specs.width != 0
                   ? count_code_points(basic_string_view<StrChar>(data, size))
                   : 0;
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  return write_padded(out, specs, size, width, [=](iterator it) {
    return copy_str<Char>(data, data + size, it);
  });
}

```cpp

This is a C++ implementation of the `fmt` function from the `fmt` library. The `fmt` library is a C++ library for格式ting and parsing JSON values.

The `abs_value` function takes an integer `num` and returns its absolute value. It does this by returning the number if it is negative, or the same number if it is non-negative.

The `fmt::abs_value` function is a member function of the `fmt` library that provides the same behavior as `abs_value`.

The `fmt::to_unsigned` function is a member function of the `fmt` library that converts an `int` object to an `unsigned` object. It does this by shifting the binary representation of the integer to the left, and wrapping any negative values to 0.

The `fmt::format_decimal` function is a member function of the `fmt` library that is used to format numbers as Decimal numbers. It takes two arguments: the first is the number to format, and the second is the width of the string. It does this by calculating the number of digits needed to represent the number, and then formats it as a Decimal number using the `std::fixed_point` class.

The `fmt::format_options` is a member function of the `fmt` library that is used to specify options for formatting and parsing JSON values. It takes two arguments: the first is a decimal number representing the precision of the formatting, and the second is an array of formatting options.

The `fmt::parse_decimal` function is a member function of the `fmt` library that is used to parse a Decimal number from a string. It takes two arguments: the first is the string to be parsed, and the second is the formatting options. It does this by using the `std::fixed_point` class to calculate the Decimal number from the binary representation of the string.

The `fmt::stringify` function is a member function of the `fmt` library that is used to format a JSON value as a string. It takes two arguments: the first is the JSON value to be formatted, and the second is the width of the string. It does this by calculating the number of characters needed to represent the value, and then formatting it as a string using the `std::fixed_point` class.

The `fmt::options` is a member function of the `fmt` library that is used to specify options for formatting and parsing JSON values. It takes two arguments: the first is a decimal number representing the precision of the formatting, and the second is an array of formatting options.

The `fmt::bind` function is a member function of the `fmt` library that is used to bind formatting options to a JSON value. It takes three arguments: the first is the JSON value to be formatted, the second is the formatting options, and the third is the width of the string. It does this by using the `std::fixed_point` class to calculate the Decimal number from the binary representation of the string, and then formatting it as a JSON value using the specified formatting options.


```
// The handle_int_type_spec handler that writes an integer.
template <typename OutputIt, typename Char, typename UInt> struct int_writer {
  OutputIt out;
  locale_ref locale;
  const basic_format_specs<Char>& specs;
  UInt abs_value;
  char prefix[4];
  unsigned prefix_size;

  using iterator =
      remove_reference_t<decltype(reserve(std::declval<OutputIt&>(), 0))>;

  string_view get_prefix() const { return string_view(prefix, prefix_size); }

  template <typename Int>
  int_writer(OutputIt output, locale_ref loc, Int value,
             const basic_format_specs<Char>& s)
      : out(output),
        locale(loc),
        specs(s),
        abs_value(static_cast<UInt>(value)),
        prefix_size(0) {
    static_assert(std::is_same<uint32_or_64_or_128_t<Int>, UInt>::value, "");
    if (is_negative(value)) {
      prefix[0] = '-';
      ++prefix_size;
      abs_value = 0 - abs_value;
    } else if (specs.sign != sign::none && specs.sign != sign::minus) {
      prefix[0] = specs.sign == sign::plus ? '+' : ' ';
      ++prefix_size;
    }
  }

  void on_dec() {
    auto num_digits = count_digits(abs_value);
    out = write_int(
        out, num_digits, get_prefix(), specs, [this, num_digits](iterator it) {
          return format_decimal<Char>(it, abs_value, num_digits).end;
        });
  }

  void on_hex() {
    if (specs.alt) {
      prefix[prefix_size++] = '0';
      prefix[prefix_size++] = specs.type;
    }
    int num_digits = count_digits<4>(abs_value);
    out = write_int(out, num_digits, get_prefix(), specs,
                    [this, num_digits](iterator it) {
                      return format_uint<4, Char>(it, abs_value, num_digits,
                                                  specs.type != 'x');
                    });
  }

  void on_bin() {
    if (specs.alt) {
      prefix[prefix_size++] = '0';
      prefix[prefix_size++] = static_cast<char>(specs.type);
    }
    int num_digits = count_digits<1>(abs_value);
    out = write_int(out, num_digits, get_prefix(), specs,
                    [this, num_digits](iterator it) {
                      return format_uint<1, Char>(it, abs_value, num_digits);
                    });
  }

  void on_oct() {
    int num_digits = count_digits<3>(abs_value);
    if (specs.alt && specs.precision <= num_digits && abs_value != 0) {
      // Octal prefix '0' is counted as a digit, so only add it if precision
      // is not greater than the number of digits.
      prefix[prefix_size++] = '0';
    }
    out = write_int(out, num_digits, get_prefix(), specs,
                    [this, num_digits](iterator it) {
                      return format_uint<3, Char>(it, abs_value, num_digits);
                    });
  }

  enum { sep_size = 1 };

  void on_num() {
    std::string groups = grouping<Char>(locale);
    if (groups.empty()) return on_dec();
    auto sep = thousands_sep<Char>(locale);
    if (!sep) return on_dec();
    int num_digits = count_digits(abs_value);
    int size = num_digits, n = num_digits;
    std::string::const_iterator group = groups.cbegin();
    while (group != groups.cend() && n > *group && *group > 0 &&
           *group != max_value<char>()) {
      size += sep_size;
      n -= *group;
      ++group;
    }
    if (group == groups.cend()) size += sep_size * ((n - 1) / groups.back());
    char digits[40];
    format_decimal(digits, abs_value, num_digits);
    basic_memory_buffer<Char> buffer;
    size += static_cast<int>(prefix_size);
    const auto usize = to_unsigned(size);
    buffer.resize(usize);
    basic_string_view<Char> s(&sep, sep_size);
    // Index of a decimal digit with the least significant digit having index 0.
    int digit_index = 0;
    group = groups.cbegin();
    auto p = buffer.data() + size;
    for (int i = num_digits - 1; i >= 0; --i) {
      *--p = static_cast<Char>(digits[i]);
      if (*group <= 0 || ++digit_index % *group != 0 ||
          *group == max_value<char>())
        continue;
      if (group + 1 != groups.cend()) {
        digit_index = 0;
        ++group;
      }
      p -= s.size();
      std::uninitialized_copy(s.data(), s.data() + s.size(),
                              make_checked(p, s.size()));
    }
    if (prefix_size != 0) p[-1] = static_cast<Char>('-');
    auto data = buffer.data();
    out = write_padded<align::right>(
        out, specs, usize, usize,
        [=](iterator it) { return copy_str<Char>(data, data + size, it); });
  }

  void on_chr() { *out++ = static_cast<Char>(abs_value); }

  FMT_NORETURN void on_error() {
    FMT_THROW(format_error("invalid type specifier"));
  }
};

```cpp

这段代码定义了一个名为 `write_nonfinite` 的模板函数，其参数包括 `OutputIt` 类型的输出迭代器和 `bool` 类型的指示符 `isinf`，以及 `basic_format_specs<Char>` 和 `float_specs` 类型的模板参数。

函数的作用是输出一个字符串，根据传入的 `isinf` 参数和 `float_specs` 类型的模板参数，会输出不同的字符串。其中，当 `isinf` 为 `true` 时，函数会输出 `"INF"` 或 `"inf"`；当 `isinf` 为 `false` 时，函数会输出 `"NAN"` 或 `"nan"`。

函数的实现主要分为两部分：

1. 计算字符串的长度，并记录下来。

2. 如果 `isinf` 为 `true`，则需要对字符串进行填充。具体来说，根据 `fspecs.sign` 的值，将字符串中的 `'N'` 或 `'N'` 转换成 `'N'` 并在字符串末尾加上 `str_size - 1` 个 `'N'`。然后，对于每个 `'N'`，将其复制到输出迭代器中。

如果 `isinf` 为 `false`，则不需要进行字符串填充，直接输出传入的字符串即可。


```
template <typename Char, typename OutputIt>
OutputIt write_nonfinite(OutputIt out, bool isinf,
                         const basic_format_specs<Char>& specs,
                         const float_specs& fspecs) {
  auto str =
      isinf ? (fspecs.upper ? "INF" : "inf") : (fspecs.upper ? "NAN" : "nan");
  constexpr size_t str_size = 3;
  auto sign = fspecs.sign;
  auto size = str_size + (sign ? 1 : 0);
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  return write_padded(out, specs, size, [=](iterator it) {
    if (sign) *it++ = static_cast<Char>(data::signs[sign]);
    return copy_str<Char>(str, str + str_size, it);
  });
}

```cpp

在这个函数中，首先检查输入是否为float_format::general格式。如果是，则执行以下操作：

1. 将output_exp转换为int，以便与exp_lower和exp_upper进行比较。
2. 如果use_exp_format()为true，则执行以下操作：

a. 将第一个数字及其后的所有数字存储在out中。
b. 如果specs.precision大于0，则执行以下操作：

i. 将exp_lower和exp_upper之间的所有零截断。
ii. 如果output_exp >= exp_lower，则执行以下操作：
  i. 在output_exp上插入一个逗号。
 ii. 将exp转换为字符串，使用format修饰符。
 iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
 如果output_exp < exp_lower，则执行以下操作：
   i. 在output_exp上插入一个逗号。
   ii. 将exp转换为字符串，使用format修饰符。
   iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
 如果output_exp >= exp_upper，则执行以下操作：
   i. 在output_exp上插入一个逗号。
   ii. 将exp转换为字符串，使用format修饰符。
   iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
   iv. 如果output_exp < exp_upper，则执行以下操作：
     i. 在output_exp上插入一个逗号。
     ii. 将exp转换为字符串，使用format修饰符。
     iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
   如果output_exp >= exp_upper，则执行以下操作：
     i. 在output_exp上插入一个逗号。
     ii. 将exp转换为字符串，使用format修饰符。
     iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
     iv. 如果output_exp < exp_upper，则执行以下操作：
       i. 在output_exp上插入一个逗号。
       ii. 将exp转换为字符串，使用format修饰符。
       iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
       iv. 如果output_exp < exp_upper，则执行以下操作：
         i. 在output_exp上插入一个逗号。
         ii. 将exp转换为字符串，使用format修饰符。
         iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
 c. 如果output_exp < exp_upper，则直接返回。

3. 如果use_exp_format()为false，则直接执行以下操作：

a. 将第一个数字及其后的所有数字存储在out中。
b. 如果output_exp >= exp_lower，则执行以下操作：

i. 在output_exp上插入一个逗号。
ii. 将exp转换为字符串，使用format修饰符。
iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
iv. 如果output_exp < exp_lower，则执行以下操作：
  i. 在output_exp上插入一个逗号。
 ii. 将exp转换为字符串，使用format修饰符。
 iii. 如果specs.format == float_format::fixed，则将output_exp与float_format::fixed格式进行比较。
 iv. 如果output_exp >= exp_upper，则执行以下操作：
   i. 在output_exp上插入一个逗号。
   ii. 将


```
// A decimal floating-point number significand * pow(10, exp).
struct big_decimal_fp {
  const char* significand;
  int significand_size;
  int exp;
};

template <typename OutputIt, typename Char>
OutputIt write_float(OutputIt out, const big_decimal_fp& fp, float_specs specs,
                     Char decimal_point) {
  const char* digits = fp.significand;
  const Char zero = static_cast<Char>('0');

  int output_exp = fp.exp + fp.significand_size - 1;
  auto use_exp_format = [=]() {
    if (specs.format == float_format::exp) return true;
    if (specs.format != float_format::general) return false;
    // Format numbers with the exponent in [exp_lower, exp_upper) using
    // the fixed notation, e.g. prefer 0.0001 to 1e-04.
    const int exp_lower = -4, exp_upper = 16;
    return output_exp < exp_lower ||
           output_exp >= (specs.precision > 0 ? specs.precision : exp_upper);
  };
  if (use_exp_format()) {
    // Insert a decimal point after the first digit and add an exponent.
    *out++ = static_cast<Char>(*digits);
    int num_zeros = specs.precision - fp.significand_size;
    if (fp.significand_size > 1 || specs.showpoint) *out++ = decimal_point;
    out = copy_str<Char>(digits + 1, digits + fp.significand_size, out);
    if (num_zeros > 0 && specs.showpoint)
      out = std::fill_n(out, num_zeros, zero);
    *out++ = static_cast<Char>(specs.upper ? 'E' : 'e');
    return write_exponent<Char>(output_exp, out);
  }

  int exp = fp.exp + fp.significand_size;
  if (fp.significand_size <= exp) {
    // 1234e7 -> 12340000000[.0+]
    out = copy_str<Char>(digits, digits + fp.significand_size, out);
    out = std::fill_n(out, exp - fp.significand_size, zero);
    if (specs.showpoint) {
      *out++ = decimal_point;
      int num_zeros = specs.precision - exp;
      if (num_zeros <= 0) {
        if (specs.format != float_format::fixed) *out++ = zero;
        return out;
      }
```cpp

这段代码是一个 C 语言函数，名为 `output_floating_point_number`。它输出了一个浮点数，对于输出浮点数时存在的小数点后位数不足的问题，该函数通过以下步骤解决了这些问题：

1. 如果 `num_zeros` 大于 5000，那么函数会抛出 `std::runtime_error`。
2. 如果 `exp` 大于 0，那么函数会先输出指定的数字字符数组 `digits`，然后输出指定的数字字符数组 `zero`，最后输出指定的数字字符数组 `decimal_point`。
3. 如果 `exp` 小于 0，那么函数会先输出指定的数字字符数组 `digits`，然后输出指定的小数点后位数 `fp.significand_size`（如果存在的话），最后输出指定的数字字符数组 `zero`。
4. 如果 `specs.showpoint` 为真，那么函数会尝试输出指定的数字字符数组 `digits`，然后输出指定的数字字符数组 `fp.significand_size`，最后输出指定的数字字符数组 `zero`。如果 `specs.showpoint` 为假，那么函数不会输出指定的数字字符数组 `zero`。
5. 如果 `fp.significand_size` 为 0，`specs.precision` 小于 `fp.significand_size`，那么函数会尝试输出指定的数字字符数组 `digits`，然后输出指定的数字字符数组 `fp.significand_size`，最后输出指定的数字字符数组 `zero`。
6. 如果 `fp.significand_size` 不为 0，`specs.precision` 大于 `fp.significand_size`，那么函数会尝试输出指定的数字字符数组 `zero`，然后输出指定的数字字符数组 `fp.significand_size`，最后输出指定的数字字符数组 `digits`。
7. 如果 `num_zeros` 不为 0，或者 `fp.significand_size` 不为 0，或者 `specs.showpoint` 为真，那么函数会在输出指定的数字字符数组 `zero` 或 `decimal_point` 或 `zero` 和 `decimal_point` 之后，输出指定的数字字符数组 `fp.significand_size`。
8. 如果步骤 4 到 7 中任意一步出现错误，那么函数会抛出错误。


```
#ifdef FMT_FUZZ
      if (num_zeros > 5000)
        throw std::runtime_error("fuzz mode - avoiding excessive cpu use");
#endif
      out = std::fill_n(out, num_zeros, zero);
    }
  } else if (exp > 0) {
    // 1234e-2 -> 12.34[0+]
    out = copy_str<Char>(digits, digits + exp, out);
    if (!specs.showpoint) {
      if (fp.significand_size != exp) *out++ = decimal_point;
      return copy_str<Char>(digits + exp, digits + fp.significand_size, out);
    }
    *out++ = decimal_point;
    out = copy_str<Char>(digits + exp, digits + fp.significand_size, out);
    // Add trailing zeros.
    if (specs.precision > fp.significand_size)
      out = std::fill_n(out, specs.precision - fp.significand_size, zero);
  } else {
    // 1234e-6 -> 0.001234
    *out++ = zero;
    int num_zeros = -exp;
    if (fp.significand_size == 0 && specs.precision >= 0 &&
        specs.precision < num_zeros) {
      num_zeros = specs.precision;
    }
    if (num_zeros != 0 || fp.significand_size != 0 || specs.showpoint) {
      *out++ = decimal_point;
      out = std::fill_n(out, num_zeros, zero);
      out = copy_str<Char>(digits, digits + fp.significand_size, out);
    }
  }
  return out;
}

```cpp

这段代码定义了一个名为`write_float`的函数，它接受一个输出迭代器`out`，以及输入缓冲区`significand`、指数`exp`、格式化参数`specs`和浮点格式化参数`fspecs`。

函数的作用是将给定的浮点数`significand`按照指定的格式打印到输出迭代器`out`中。具体来说，函数首先定义了一个`big_decimal_fp`类型的变量`fp`，该变量使用给定的`significand`数据和指定的指数`exp`来创建一个`big_decimal_fp`类型的变量。然后，函数调用了`write_float`函数，该函数接受一个输出迭代器`out`、一个表示浮点数`significand`的缓冲区`significand`、指数`exp`、格式化参数`specs`和浮点格式化参数`fspecs`。最后，函数根据输入的参数对`fp`进行修改，并且在`write_float`函数中使用了对齐`align::right`特性来确保输出能够向右对齐。

因此，整个函数的作用是将给定的浮点数按照指定的格式打印到输出迭代器`out`中，其中`significand`、`exp`、`specs`和`fspecs`是格式化参数。


```
// The number is given as v = significand * pow(10, exp).
template <typename OutputIt, typename Char>
OutputIt write_float(OutputIt out, const buffer<char>& significand, int exp,
                     const basic_format_specs<Char>& specs, float_specs fspecs,
                     Char decimal_point) {
  auto fp = big_decimal_fp{significand.data(),
                           static_cast<int>(significand.size()), exp};
  auto size =
      write_float(counting_iterator(), fp, fspecs, decimal_point).count();
  size += fspecs.sign ? 1 : 0;
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  return write_padded<align::right>(out, specs, size, [&](iterator it) {
    if (fspecs.sign) *it++ = static_cast<Char>(data::signs[fspecs.sign]);
    return write_float(it, fp, fspecs, decimal_point);
  });
}

```cpp

This is a C++ function that writes a binary value to a file, specified by `fspecs`. The function takes a `value` to write and a `fspecs` object, which contains the format and alignment of the value. The function returns either a `write_bytes` or `write_nonfinite` error depending on whether the value is negative, inf or inf.

The function first checks if the `value` is negative. If it is, the function sets the `sign` field of the `fspecs` object to `sign::minus` and returns a `write_nonfinite` error.

If the `value` is inf or inf, the function promotes it to a `float` and writes it to the file, using the `float_format::hex` format. The `exp` field specifies the exponent, which is set to 0 if the `fspecs` object does not specify a format.

If the `value` is a simple `float`, the function writes it to the file, using the `float_format::exp` format. The `precision` field specifies the number of digits to which the value should be formatted.

If the `fspecs` object specifies a `float_format::hex`, the function writes the `value` to the file, using the `write_bytes` function, which writes the `value` to the output stream.

If the `fspecs` object specifies a `float_format::exp`, the function first


```
template <typename Char, typename OutputIt, typename T,
          FMT_ENABLE_IF(std::is_floating_point<T>::value)>
OutputIt write(OutputIt out, T value, basic_format_specs<Char> specs,
               locale_ref loc = {}) {
  if (const_check(!is_supported_floating_point(value))) return out;
  float_specs fspecs = parse_float_type_spec(specs);
  fspecs.sign = specs.sign;
  if (std::signbit(value)) {  // value < 0 is false for NaN so use signbit.
    fspecs.sign = sign::minus;
    value = -value;
  } else if (fspecs.sign == sign::minus) {
    fspecs.sign = sign::none;
  }

  if (!std::isfinite(value))
    return write_nonfinite(out, std::isinf(value), specs, fspecs);

  if (specs.align == align::numeric && fspecs.sign) {
    auto it = reserve(out, 1);
    *it++ = static_cast<Char>(data::signs[fspecs.sign]);
    out = base_iterator(out, it);
    fspecs.sign = sign::none;
    if (specs.width != 0) --specs.width;
  }

  memory_buffer buffer;
  if (fspecs.format == float_format::hex) {
    if (fspecs.sign) buffer.push_back(data::signs[fspecs.sign]);
    snprintf_float(promote_float(value), specs.precision, fspecs, buffer);
    return write_bytes(out, {buffer.data(), buffer.size()}, specs);
  }
  int precision = specs.precision >= 0 || !specs.type ? specs.precision : 6;
  if (fspecs.format == float_format::exp) {
    if (precision == max_value<int>())
      FMT_THROW(format_error("number is too big"));
    else
      ++precision;
  }
  if (const_check(std::is_same<T, float>())) fspecs.binary32 = true;
  fspecs.use_grisu = is_fast_float<T>();
  int exp = format_float(promote_float(value), precision, fspecs, buffer);
  fspecs.precision = precision;
  Char point =
      fspecs.locale ? decimal_point<Char>(loc) : static_cast<Char>('.');
  return write_float(out, buffer, exp, specs, fspecs, point);
}

```cpp

这段代码定义了一个名为`write`的模板函数，其参数为`OutputIt`类型的输出流`out`和`T`类型的数据`value`。函数的实现中，首先对传入的`value`进行一些条件判断，如果`is_supported_floating_point`函数返回为真，则不做任何处理，否则会尝试使用`float_specs`函数来输出浮点数。

接着，会根据`value`的正负情况，分别使用`sign::minus`和`sign::plus`来对`value`进行符号处理。如果`value`为负数，则使用`sign::minus`，否则使用`sign::plus`。

然后，会使用`basic_format_specs`函数来获取输出流`out`所需要的基本格式信息，包括小数点前缀的长度和精度等。接着，会使用`dragonbox::to_decimal`函数将输入的`value`转换成带小数点的数字。

最后，会使用`write_float`函数来输出浮点数。该函数的第一个参数为输出流`out`，第二个参数为`buf`字符缓冲区，第三个参数为`dec.exponent`输出浮点数的指数，第四个参数为`fspecs`输出格式信息，最后一个参数为浮点数的类型加字符。


```
template <
    typename Char, typename OutputIt, typename T,
    FMT_ENABLE_IF(std::is_floating_point<T>::value&& is_fast_float<T>::value)>
OutputIt write(OutputIt out, T value) {
  if (const_check(!is_supported_floating_point(value))) return out;
  auto fspecs = float_specs();
  if (std::signbit(value)) {  // value < 0 is false for NaN so use signbit.
    fspecs.sign = sign::minus;
    value = -value;
  }

  auto specs = basic_format_specs<Char>();
  if (!std::isfinite(value))
    return write_nonfinite(out, std::isinf(value), specs, fspecs);

  using type = conditional_t<std::is_same<T, long double>::value, double, T>;
  auto dec = dragonbox::to_decimal(static_cast<type>(value));
  memory_buffer buf;
  write<char>(buffer_appender<char>(buf), dec.significand);
  return write_float(out, buf, dec.exponent, specs, fspecs,
                     static_cast<Char>('.'));
}

```cpp

这段代码定义了两个模板函数，一个用于输出浮点数，另一个用于输出字符串。

第一个模板函数 `write` 接受一个 `OutputIt` 类型的输出流和 `T` 类型的数据。它的作用是接受一个浮点数 `value`，并将其输出到 `OutputIt` 类型的输出流中。为了实现这个目标，它使用了 `write_padded` 函数，这个函数会在输出流中插入额外的字符以适应数据类型的大小。

第二个模板函数 `write_char` 同样接受一个 `OutputIt` 类型的输出流和一个 `Char` 类型的数据，以及一个 `basic_format_specs<Char>` 类型的模板参数。它的作用是接受一个字符 `value`，并将其输出到 `OutputIt` 类型的输出流中。它使用了 `write_padded` 函数，但这个函数的参数是 `out` 和 `specs`，而不是 `write` 的。


```
template <typename Char, typename OutputIt, typename T,
          FMT_ENABLE_IF(std::is_floating_point<T>::value &&
                        !is_fast_float<T>::value)>
inline OutputIt write(OutputIt out, T value) {
  return write(out, value, basic_format_specs<Char>());
}

template <typename Char, typename OutputIt>
OutputIt write_char(OutputIt out, Char value,
                    const basic_format_specs<Char>& specs) {
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  return write_padded(out, specs, 1, [=](iterator it) {
    *it++ = value;
    return it;
  });
}

```cpp

这段代码定义了一个名为 `write_ptr` 的模板函数，它接受三个参数：

1. `OutputIt` 类型的输出迭代器 `out`，该参数用于将格式化后的字符串输出给用户。
2. 一个指向 `Char` 类型数据的 `UIntPtr` 类型的整数 `value`，该参数用于表示要格式化的字符串中的字符。
3. 一个名为 `basic_format_specs<Char>` 的模板类，该类提供了一种用于格式化字符串的接口。

函数的实现主要步骤如下：

1. 计算字符串中字符的数量 `num_digits`，使用 `count_digits` 函数实现。
2. 根据 `num_digits` 计算出要输出的字符串长度 `size`，使用 `to_unsigned` 函数实现。
3. 定义一个名为 `write` 的函数，该函数接受一个 `basic_format_specs<Char>` 的模板参数，该函数将根据 `specs` 的值输出字符串中字符的一个或多个，同时将剩余的字符用 `base_iterator` 类型的参数 `out` 中的字符进行输出。
4. 根据 `size` 和 `write` 函数，使用 `write_padded` 函数对输出的字符串进行填充，该函数将根据 `align::right` 参数设置输出字符串对齐方式为右对齐。
5. 如果 `specs` 存在，则调用 `write_padded` 函数，否则调用 `base_iterator` 函数，该函数将使用 `reserve` 函数对 `out` 参数进行预留，以避免在调用 `write` 函数时产生 `out_size` 超出 `out` 参数的错误。


```
template <typename Char, typename OutputIt, typename UIntPtr>
OutputIt write_ptr(OutputIt out, UIntPtr value,
                   const basic_format_specs<Char>* specs) {
  int num_digits = count_digits<4>(value);
  auto size = to_unsigned(num_digits) + size_t(2);
  using iterator = remove_reference_t<decltype(reserve(out, 0))>;
  auto write = [=](iterator it) {
    *it++ = static_cast<Char>('0');
    *it++ = static_cast<Char>('x');
    return format_uint<4, Char>(it, value, num_digits);
  };
  return specs ? write_padded<align::right>(out, *specs, size, write)
               : base_iterator(out, write(reserve(out, size)));
}

```cpp

这段代码定义了一系列模板结构体，称为“is_integral”，用于检查给定的模板类型的整数是否为真。

具体来说，第一个模板结构体检查的是 `T` 是否为整数，如果是，则它也检查 `T` 是否为 `std::is_integral<T>` 的包装类型。第二个模板结构体检查的是 `int128_t` 是否为整数，如果是，则它也检查 `std::is_integral<int128_t>` 的包装类型。第三个模板结构体检查的是 `uint128_t` 是否为整数，如果是，则它也检查 `std::is_integral<uint128_t>` 的包装类型。

另外，还定义了一个模板函数 `write`，用于将字符串中的值输出到给定的输出流（`OutputIt`）中。函数有两个参数：输出流（`OutputIt`）和字符串 view 表示值。函数实现了一个 "劣质特殊化" 行为，即当给定模板类型不是字符（`char`）时，会尝试从字符串 view 中提取出字符，并将其赋值给输出流。

总结一下，这段代码定义了一系列模板结构体，用于检查给定类型的整数是否为真，并定义了一个函数来将字符串中的值输出到给定的输出流中。


```
template <typename T> struct is_integral : std::is_integral<T> {};
template <> struct is_integral<int128_t> : std::true_type {};
template <> struct is_integral<uint128_t> : std::true_type {};

template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, monostate) {
  FMT_ASSERT(false, "");
  return out;
}

template <typename Char, typename OutputIt,
          FMT_ENABLE_IF(!std::is_same<Char, char>::value)>
OutputIt write(OutputIt out, string_view value) {
  auto it = reserve(out, value.size());
  it = copy_str<Char>(value.begin(), value.end(), it);
  return base_iterator(out, it);
}

```cpp

这段代码定义了两个模板函数：`write` 和 `buffer_appender`。它们的作用是将一个 `basic_string_view<Char>` 中的字符串复制到另一个 `OutputIt` 类型的变量中。

具体来说，这两个函数的实现都包含以下步骤：

1. 创建一个 `OutputIt` 类型的变量 `out` 和一个 `basic_string_view<Char>` 类型的变量 `value`。
2. 使用 `reserve` 函数为 `out` 分配足够的空间，然后使用 `std::copy` 函数将 `value` 中的字符串复制到 `it` 中。
3. 返回 `base_iterator` 类型的指针，该指针指向 `out` 和 `it` 的起始位置。
4. 对于 `write` 函数，使用 `is_integral` 模板特性检查 `T` 是否为整数类型。如果是，则执行以下操作：
  1. 使用 `std::is_same` 函数检查 `T` 是否与 `bool` 类型相同。如果不是，则执行以下操作：
     1. 使用 `std::is_same` 函数检查 `T` 是否与 `Char` 类型相同。如果不是，则执行以下操作：
       1. 使用 `std::copy` 函数将 `value` 中的字符串复制到 `it` 中。
       2. 对于 `is_integral<T>::value`，使用 `std::copy` 函数将 `value` 中的字符串复制到 `out` 中。
       3. 对于 `!std::is_same<T, bool>::value` 和 `!std::is_same<T, Char>::value`，执行复制操作。
       4. 返回 `out`。
5. 对于 `buffer_appender` 函数，使用 `is_integral` 模板特性检查 `T` 是否为整数类型。如果不是，则执行以下操作：
  1. 使用 `std::is_same` 函数检查 `T` 是否与 `bool` 类型相同。如果不是，则执行以下操作：
     1. 使用 `std::is_same` 函数检查 `T` 是否与 `Char` 类型相同。如果不是，则执行以下操作：
       1. 使用 `std::copy` 函数将 `value` 中的字符串复制到 `it` 中。
       2. 对于 `is_integral<T>::value`，使用 `std::copy` 函数将 `value` 中的字符串复制到 `out` 中。
       3. 对于 `!std::is_same<T, bool>::value` 和 `!std::is_same<T, Char>::value`，执行复制操作。
       4. 返回 `out`。


```
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, basic_string_view<Char> value) {
  auto it = reserve(out, value.size());
  it = std::copy(value.begin(), value.end(), it);
  return base_iterator(out, it);
}

template <typename Char>
buffer_appender<Char> write(buffer_appender<Char> out,
                            basic_string_view<Char> value) {
  get_container(out).append(value.begin(), value.end());
  return out;
}

template <typename Char, typename OutputIt, typename T,
          FMT_ENABLE_IF(is_integral<T>::value &&
                        !std::is_same<T, bool>::value &&
                        !std::is_same<T, Char>::value)>
```cpp

这段代码是一个名为 `write` 的函数，其作用是输出一个 `T` 类型的值，并将其打印到 `OutputIt` 对象 `out` 中。

具体来说，代码首先将给定的 `value` 转换为 `T` 类型，并判断其是否为负数。如果是负数，则将 `abs_value` 加一，如果不是负数，则不做任何处理。

接着，代码计算 `abs_value` 泥封状态下的位数，并输出 `num_digits` 和 `size` 中的较大值，同时为负数情况下的输出字符串预置一个空字符串。

最后，代码通过 `format_decimal` 函数将 `abs_value` 输出为字符串，并在输出前添加 `'-'`，如果 `negative` 为 true，则循环输出负号。同时，代码还通过 `base_iterator` 函数将输出结果返回给调用者。


```
OutputIt write(OutputIt out, T value) {
  auto abs_value = static_cast<uint32_or_64_or_128_t<T>>(value);
  bool negative = is_negative(value);
  // Don't do -abs_value since it trips unsigned-integer-overflow sanitizer.
  if (negative) abs_value = ~abs_value + 1;
  int num_digits = count_digits(abs_value);
  auto size = (negative ? 1 : 0) + static_cast<size_t>(num_digits);
  auto it = reserve(out, size);
  if (auto ptr = to_pointer<Char>(it, size)) {
    if (negative) *ptr++ = static_cast<Char>('-');
    format_decimal<Char>(ptr, abs_value, num_digits);
    return out;
  }
  if (negative) *it++ = static_cast<Char>('-');
  it = format_decimal<Char>(it, abs_value, num_digits).end;
  return base_iterator(out, it);
}

```cpp

这段代码定义了一个名为 `write` 的模板函数，用于将 `true` 或 `false` 字符串输出到 `OutputIt` 类型的输出容器中。

第一个模板参数 `Char` 表示要输出的字符类型，第二个模板参数 `OutputIt` 表示用于输出字符串的输出容器类型。

第一个实现模板函数的代码将 `write` 函数分为两个部分，第一个部分是一个模板函数，第二个部分是一个非模板函数。

模板函数的第一部分定义了一个名为 `write` 的模板函数，它接受两个参数 `OutputIt` 和 `bool`。模板函数的第一部分定义了 `write` 函数的参数类型，它需要一个 `OutputIt` 类型的输出容器和一个 `bool` 类型的参数。

模板函数的第二部分定义了一个名为 `write` 的非模板函数，它接受一个 `char` 类型的参数和一个 `const Char*` 类型的参数。这个非模板函数的实现与模板函数的第一部分相同，只不过它的参数类型与模板函数不同。

最后，三个模板函数都被定义为 `write` 函数的一部分，它们分别接受不同的参数类型，并将其赋值给 `write` 函数的参数。


```
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, bool value) {
  return write<Char>(out, string_view(value ? "true" : "false"));
}

template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, Char value) {
  auto it = reserve(out, 1);
  *it++ = value;
  return base_iterator(out, it);
}

template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, const Char* value) {
  if (!value) {
    FMT_THROW(format_error("string pointer is null"));
  } else {
    auto length = std::char_traits<Char>::length(value);
    out = write(out, basic_string_view<Char>(value, length));
  }
  return out;
}

```cpp

这段代码定义了两个模板变量：

1. `write`模板变量，它是一个函数，接收一个输出迭代器`out`和一个字符类型的参数`value`，并返回一个输出迭代器`out`。

2. `write_ptr`模板变量，它是一个指针，指向一个将参数`value`转换为输出迭代器类型的函数。这个函数的实现类似于`std::placeholders`中的`_`运算符，它会将参数`value`存储到函数内部，然后在函数外部将其输出。

`write`函数的作用是将`value`参数通过`write_ptr`函数转换为输出迭代器类型，然后将其输出。`write_ptr`函数将参数`value`存储在一个内部指针中，然后使用`std::output_iterator`类型将其输出。`std::output_iterator`类型是一个模板元编程技术（TMP），它使得我们可以在运行时安全地输出参数。

`write`函数的实现基于两个条件：

1. 如果`T`是一个类型参数，则`write`函数的实现将依赖于`T`的类型定义。如果没有指定类型，则`write`函数的行为将为所有类型提供相同的输出。

2. 如果`T`的类型定义中包含一个名为`basic_format_context`的成员，则`write`函数的行为将依赖于`basic_format_context`的实现。如果没有`basic_format_context`成员，则`write`函数的行为将为所有类型提供相同的输出。


```
template <typename Char, typename OutputIt>
OutputIt write(OutputIt out, const void* value) {
  return write_ptr<Char>(out, to_uintptr(value), nullptr);
}

template <typename Char, typename OutputIt, typename T>
auto write(OutputIt out, const T& value) -> typename std::enable_if<
    mapped_type_constant<T, basic_format_context<OutputIt, Char>>::value ==
        type::custom_type,
    OutputIt>::type {
  using context_type = basic_format_context<OutputIt, Char>;
  using formatter_type =
      conditional_t<has_formatter<T, context_type>::value,
                    typename context_type::template formatter_type<T>,
                    fallback_formatter<T, Char>>;
  context_type ctx(out, {}, {});
  return formatter_type().format(value, ctx);
}

```cpp

这段代码定义了一个名为`default_arg_formatter`的模板类，该类接受两个模板参数：`OutputIt`表示输出迭代器，`Char`表示输出字符类型。

模板类中的`out`成员变量是一个输出迭代器，用于将格式化后的 argument 写入到 `out` 中。`args`是一个模板参数，用于在`default_arg_formatter`中声明 `args` 成员函数接受的所有参数的格式。`loc`是一个指向 `basic_format_context<Char>` 的指针，该指针用于在 `default_arg_formatter` 中记录当前正在处理的元素（如`handle`）的格式位置。

`operator()` 函数是一个模板函数，用于将给定的 argument 写入到 `out` 中。`operator()` 函数有两个模板参数：第一个参数是一个 `T` 类型的变量，代表 argument 的类型；第二个参数是一个 basic `format_arg<context>` 类型的 handle 变量。这个函数会先调用 `write<Char>` 函数将 argument 写入到 `out` 中，然后根据 `handle` 的类型调用 `basic_format_parse_context<Char>` 和 `basic_format_context<OutputIt, Char>` 来解析和格式化 argument。最后将解析后的 format string 写入到 `out` 中。

`default_arg_formatter` 的作用是定义了一个可以格式化 argument 的函数，该函数可以接受 basic `format_arg<context>` 类型的 handle 变量作为第一个模板参数。这个函数通过将 argument 写入到 `out` 中，并返回 `out` 中的内容，使得我们可以将 formatted argument 作为参数传递给 `template <typename OutputIt, typename Char> struct default_arg_formatter<OutputIt, Char>` 函数。


```
// An argument visitor that formats the argument and writes it via the output
// iterator. It's a class and not a generic lambda for compatibility with C++11.
template <typename OutputIt, typename Char> struct default_arg_formatter {
  using context = basic_format_context<OutputIt, Char>;

  OutputIt out;
  basic_format_args<context> args;
  locale_ref loc;

  template <typename T> OutputIt operator()(T value) {
    return write<Char>(out, value);
  }

  OutputIt operator()(typename basic_format_arg<context>::handle handle) {
    basic_format_parse_context<Char> parse_ctx({});
    basic_format_context<OutputIt, Char> format_ctx(out, args, loc);
    handle.format(parse_ctx, format_ctx);
    return format_ctx.out();
  }
};

```cpp

This is a C++ iterator class that inherits from the `std::iterator` class. It is used to iterate over the elements of a `std::map` or `std::begin` container.

The iterator class has several overloads for the various types of iterators that can be used with standard container iterators, including `std::map`, `std::begin`, and `std::end`. These overloads include support for iterating over integers, floating-point numbers, and boolean values.

The `operator()` method is defined to handle the special case of a `std::map` or `std::begin` container, which has a different type type than the iterator. The method checks the type of the element being requested and calls the appropriate implementation for that type using the `std::detail` namespace.

The iterator class also includes a template for an iterator of `T` that specifies the type of the element being requested. This template is generated if the `std::is_floating_point` template is not available, and it allows the iterator to accept floating-point values.

Note that the `std::map` and `std::begin` containers do not have an `operator()` method, so the iterator class must be manually iterated over the elements of these containers.


```
template <typename OutputIt, typename Char,
          typename ErrorHandler = error_handler>
class arg_formatter_base {
 public:
  using iterator = OutputIt;
  using char_type = Char;
  using format_specs = basic_format_specs<Char>;

 private:
  iterator out_;
  locale_ref locale_;
  format_specs* specs_;

  // Attempts to reserve space for n extra characters in the output range.
  // Returns a pointer to the reserved range or a reference to out_.
  auto reserve(size_t n) -> decltype(detail::reserve(out_, n)) {
    return detail::reserve(out_, n);
  }

  using reserve_iterator = remove_reference_t<decltype(
      detail::reserve(std::declval<iterator&>(), 0))>;

  template <typename T> void write_int(T value, const format_specs& spec) {
    using uint_type = uint32_or_64_or_128_t<T>;
    int_writer<iterator, Char, uint_type> w(out_, locale_, value, spec);
    handle_int_type_spec(spec.type, w);
    out_ = w.out;
  }

  void write(char value) {
    auto&& it = reserve(1);
    *it++ = value;
  }

  template <typename Ch, FMT_ENABLE_IF(std::is_same<Ch, Char>::value)>
  void write(Ch value) {
    out_ = detail::write<Char>(out_, value);
  }

  void write(string_view value) {
    auto&& it = reserve(value.size());
    it = copy_str<Char>(value.begin(), value.end(), it);
  }
  void write(wstring_view value) {
    static_assert(std::is_same<Char, wchar_t>::value, "");
    auto&& it = reserve(value.size());
    it = std::copy(value.begin(), value.end(), it);
  }

  template <typename Ch>
  void write(const Ch* s, size_t size, const format_specs& specs) {
    auto width = specs.width != 0
                     ? count_code_points(basic_string_view<Ch>(s, size))
                     : 0;
    out_ = write_padded(out_, specs, size, width, [=](reserve_iterator it) {
      return copy_str<Char>(s, s + size, it);
    });
  }

  template <typename Ch>
  void write(basic_string_view<Ch> s, const format_specs& specs = {}) {
    out_ = detail::write(out_, s, specs);
  }

  void write_pointer(const void* p) {
    out_ = write_ptr<char_type>(out_, to_uintptr(p), specs_);
  }

  struct char_spec_handler : ErrorHandler {
    arg_formatter_base& formatter;
    Char value;

    char_spec_handler(arg_formatter_base& f, Char val)
        : formatter(f), value(val) {}

    void on_int() {
      // char is only formatted as int if there are specs.
      formatter.write_int(static_cast<int>(value), *formatter.specs_);
    }
    void on_char() {
      if (formatter.specs_)
        formatter.out_ = write_char(formatter.out_, value, *formatter.specs_);
      else
        formatter.write(value);
    }
  };

  struct cstring_spec_handler : error_handler {
    arg_formatter_base& formatter;
    const Char* value;

    cstring_spec_handler(arg_formatter_base& f, const Char* val)
        : formatter(f), value(val) {}

    void on_string() { formatter.write(value); }
    void on_pointer() { formatter.write_pointer(value); }
  };

 protected:
  iterator out() { return out_; }
  format_specs* specs() { return specs_; }

  void write(bool value) {
    if (specs_)
      write(string_view(value ? "true" : "false"), *specs_);
    else
      out_ = detail::write<Char>(out_, value);
  }

  void write(const Char* value) {
    if (!value) {
      FMT_THROW(format_error("string pointer is null"));
    } else {
      auto length = std::char_traits<char_type>::length(value);
      basic_string_view<char_type> sv(value, length);
      specs_ ? write(sv, *specs_) : write(sv);
    }
  }

 public:
  arg_formatter_base(OutputIt out, format_specs* s, locale_ref loc)
      : out_(out), locale_(loc), specs_(s) {}

  iterator operator()(monostate) {
    FMT_ASSERT(false, "invalid argument type");
    return out_;
  }

  template <typename T, FMT_ENABLE_IF(is_integral<T>::value)>
  FMT_INLINE iterator operator()(T value) {
    if (specs_)
      write_int(value, *specs_);
    else
      out_ = detail::write<Char>(out_, value);
    return out_;
  }

  iterator operator()(Char value) {
    handle_char_specs(specs_,
                      char_spec_handler(*this, static_cast<Char>(value)));
    return out_;
  }

  iterator operator()(bool value) {
    if (specs_ && specs_->type) return (*this)(value ? 1 : 0);
    write(value != 0);
    return out_;
  }

  template <typename T, FMT_ENABLE_IF(std::is_floating_point<T>::value)>
  iterator operator()(T value) {
    auto specs = specs_ ? *specs_ : format_specs();
    if (const_check(is_supported_floating_point(value)))
      out_ = detail::write(out_, value, specs, locale_);
    else
      FMT_ASSERT(false, "unsupported float argument type");
    return out_;
  }

  iterator operator()(const Char* value) {
    if (!specs_) return write(value), out_;
    handle_cstring_type_spec(specs_->type, cstring_spec_handler(*this, value));
    return out_;
  }

  iterator operator()(basic_string_view<Char> value) {
    if (specs_) {
      check_string_type_spec(specs_->type, error_handler());
      write(value, *specs_);
    } else {
      write(value);
    }
    return out_;
  }

  iterator operator()(const void* value) {
    if (specs_) check_pointer_type_spec(specs_->type, error_handler());
    write_pointer(value);
    return out_;
  }
};

```cpp

这两段代码定义了两个模板函数，is_name_start() 和 parse_nonnegative_int()。

is_name_start() 是一个判断字符是否以字母 'a' 到 'z' 开头的方法，返回 true 当字符以 'a' 到 'z' 开头，或者 '_' 是字符的开头。

parse_nonnegative_int() 是一个函数，用于解析一个非负整数（即大于等于 0 的整数）的指定范围。该函数接受一个字符指针和一个错误处理函数。它将字符流转换为整数并返回一个非负整数，直到遇到 '0' 字符。当字符串中存在非数字符 '0' 时，函数会抛出错误。

该代码中使用的错误处理函数为俄罗斯的尤里·严寒，他的输出将格式化为 "number is too big" 。


```
template <typename Char> FMT_CONSTEXPR bool is_name_start(Char c) {
  return ('a' <= c && c <= 'z') || ('A' <= c && c <= 'Z') || '_' == c;
}

// Parses the range [begin, end) as an unsigned integer. This function assumes
// that the range is non-empty and the first character is a digit.
template <typename Char, typename ErrorHandler>
FMT_CONSTEXPR int parse_nonnegative_int(const Char*& begin, const Char* end,
                                        ErrorHandler&& eh) {
  FMT_ASSERT(begin != end && '0' <= *begin && *begin <= '9', "");
  unsigned value = 0;
  // Convert to unsigned to prevent a warning.
  constexpr unsigned max_int = max_value<int>();
  unsigned big = max_int / 10;
  do {
    // Check for overflow.
    if (value > big) {
      value = max_int + 1;
      break;
    }
    value = value * 10 + unsigned(*begin - '0');
    ++begin;
  } while (begin != end && '0' <= *begin && *begin <= '9');
  if (value > max_int) eh.on_error("number is too big");
  return static_cast<int>(value);
}

```cpp

该代码定义了一个模板类称为“custom_formatter”的类，用于格式化输出<typename Context>中的内容。

该类包括两个私有成员变量：一个名为“parse_ctx_”的basic_format_parse_context<char_type>&类型的变量，另一个名为“ctx_”的Context类型的变量。

该类有一个构造函数，其参数为“basic_format_parse_context<char_type>&”类型的变量和“Context”类型的变量。在构造函数中，将“parse_ctx_”和“ctx_”成员变量初始化给构造函数实参。

该类有一个名为“operator[]”的模板元编程函数，用于操作符“[]”的重载。该函数接收单个参数，该参数是一个基本格式辅助器<Context>类型的变量。

该函数的作用是接收一个新的基本格式辅助器<Context>类型的变量，并将其格式化输出，然后将结果存储回原始变量。

该类的实例可以用来操作输出<typename Context>中的字符串或更复杂的字符串格式化，如格式化输出日期、时间等。


```
template <typename Context> class custom_formatter {
 private:
  using char_type = typename Context::char_type;

  basic_format_parse_context<char_type>& parse_ctx_;
  Context& ctx_;

 public:
  explicit custom_formatter(basic_format_parse_context<char_type>& parse_ctx,
                            Context& ctx)
      : parse_ctx_(parse_ctx), ctx_(ctx) {}

  bool operator()(typename basic_format_arg<Context>::handle h) const {
    h.format(parse_ctx_, ctx_);
    return true;
  }

  template <typename T> bool operator()(T) const { return false; }
};

```cpp

这段代码定义了一个名为 `is_integer` 的模板类型，用于检查给定的模板参数 `T` 是否为整数。模板参数 `T` 不能是 `bool`、`char` 或 `wchar_t` 类型。

此外，还定义了一个名为 `width_checker` 的类，该类实现了一个 `operator()` 函数，用于检查给定的模板参数的值是否为整数。该函数使用一个名为 `handler_` 的引用参数 `ErrorHandler` 进行错误处理。

`is_integer` 模板类型的定义以及 `width_checker` 类的实现使得输入参数的值在 `T` 类型范围内时返回 `unsigned long long`，否则返回 `unsigned long long`。


```
template <typename T>
using is_integer =
    bool_constant<is_integral<T>::value && !std::is_same<T, bool>::value &&
                  !std::is_same<T, char>::value &&
                  !std::is_same<T, wchar_t>::value>;

template <typename ErrorHandler> class width_checker {
 public:
  explicit FMT_CONSTEXPR width_checker(ErrorHandler& eh) : handler_(eh) {}

  template <typename T, FMT_ENABLE_IF(is_integer<T>::value)>
  FMT_CONSTEXPR unsigned long long operator()(T value) {
    if (is_negative(value)) handler_.on_error("negative width");
    return static_cast<unsigned long long>(value);
  }

  template <typename T, FMT_ENABLE_IF(!is_integer<T>::value)>
  FMT_CONSTEXPR unsigned long long operator()(T) {
    handler_.on_error("width is not integer");
    return 0;
  }

 private:
  ErrorHandler& handler_;
};

```cpp

该代码定义了一个名为“precision_checker”的模板类，用于检查输入数据的精度。该类模板参数为一个“ErrorHandler”类型的变量，表示在检查输入时出现错误时的处理程序。

该类有一个公共的“explicit”构造函数，用于初始化错误处理程序“eh”，该构造函数没有参数。

该类有一个模板重载的函数“operator()”，用于对输入数据“value”进行操作并返回结果。如果输入数据“value”不是整数，该函数将返回“0”，并将错误信息记录为“precision is not integer”。如果输入数据“value”是整数，该函数将返回输入数据“value”，并不会产生错误信息。

该类有一个私有成员变量“handler_”，用于存储错误处理程序“eh”。

该类可以被实例化，并使用“ErrorHandler”类型的对象进行初始化。例如：

```
ErrorHandler myErrorHandler;
precision_checker myChecker(myErrorHandler);
```cpp

在使用该类时，可以通过调用“myChecker”的“operator()”函数来检查输入数据的精度，并将错误信息记录为“ErrorMessage”。例如：

```
int myValue = -5;
myChecker.operator()(myValue); // 使用 myChecker 的 operator() 函数进行操作
```cpp


```
template <typename ErrorHandler> class precision_checker {
 public:
  explicit FMT_CONSTEXPR precision_checker(ErrorHandler& eh) : handler_(eh) {}

  template <typename T, FMT_ENABLE_IF(is_integer<T>::value)>
  FMT_CONSTEXPR unsigned long long operator()(T value) {
    if (is_negative(value)) handler_.on_error("negative precision");
    return static_cast<unsigned long long>(value);
  }

  template <typename T, FMT_ENABLE_IF(!is_integer<T>::value)>
  FMT_CONSTEXPR unsigned long long operator()(T) {
    handler_.on_error("precision is not integer");
    return 0;
  }

 private:
  ErrorHandler& handler_;
};

```cpp

这段代码定义了一个名为 `specs_setter` 的模板类，用于设置 `basic_format_specs` 模板类型的字段。

该模板类包含了一系列的 `on_` 重载函数，用于设置 `basic_format_specs` 模板类型的字段。这些函数分别用于设置字段的对齐方式、填充内容、设置符号、设置字段宽度、设置精度、设置字段类型等。

除了函数外，该模板类还有一个名为 `end_precision` 的函数，用于结束 `on_` 重载函数的序列。

该模板类被声明为 `FMT_CONSTEXPR`，意味着它是一个计算时类型安全的模板类。

该模板类提供了一个方便的接口，通过使用 `specs_setter` 对象，可以设置 `basic_format_specs` 模板类型的字段。例如：
```
specs_setter<std::string> my_formatter(const std::string& str);
```cpp
该代码将创建一个名为 `my_formatter` 的 `specs_setter` 对象，并将 `str` 字段的值设置为 `"hello"`。


```
// A format specifier handler that sets fields in basic_format_specs.
template <typename Char> class specs_setter {
 public:
  explicit FMT_CONSTEXPR specs_setter(basic_format_specs<Char>& specs)
      : specs_(specs) {}

  FMT_CONSTEXPR specs_setter(const specs_setter& other)
      : specs_(other.specs_) {}

  FMT_CONSTEXPR void on_align(align_t align) { specs_.align = align; }
  FMT_CONSTEXPR void on_fill(basic_string_view<Char> fill) {
    specs_.fill = fill;
  }
  FMT_CONSTEXPR void on_plus() { specs_.sign = sign::plus; }
  FMT_CONSTEXPR void on_minus() { specs_.sign = sign::minus; }
  FMT_CONSTEXPR void on_space() { specs_.sign = sign::space; }
  FMT_CONSTEXPR void on_hash() { specs_.alt = true; }

  FMT_CONSTEXPR void on_zero() {
    specs_.align = align::numeric;
    specs_.fill[0] = Char('0');
  }

  FMT_CONSTEXPR void on_width(int width) { specs_.width = width; }
  FMT_CONSTEXPR void on_precision(int precision) {
    specs_.precision = precision;
  }
  FMT_CONSTEXPR void end_precision() {}

  FMT_CONSTEXPR void on_type(Char type) {
    specs_.type = static_cast<char>(type);
  }

 protected:
  basic_format_specs<Char>& specs_;
};

```cpp

这段代码定义了一个名为 `numeric_specs_checker` 的模板类，用于检查输入的数值表达式是否符合特定的格式要求。

该模板类接受两个参数：一个 `ErrorHandler` 类型的成员变量 `error_handler_`，以及一个 `detail::type` 类型的参数 `arg_type_`。

`require_numeric_argument()` 方法用于检查传入的 `arg_type_` 是否为数值类型，如果不是，则错误处理程序会报告一个格式错误。

`check_sign()` 方法用于检查传入的 `arg_type_` 是否为整数类型、长整数类型或字符类型，如果不是，则错误处理程序会报告一个格式错误。

`check_precision()` 方法用于检查传入的 `arg_type_` 是否为整数类型、指针类型或者 `ErrorHandler` 类型，如果不是，则错误处理程序会报告一个格式错误。

注意，上述方法中的 `is_arithmetic_type()` 和 `is_integral_type()` 函数用于检查传入的 `arg_type_` 是否为数值类型。如果 `arg_type_` 不是数值类型，则 `require_numeric_argument()` 和 `check_sign()` 方法将不会被执行。


```
template <typename ErrorHandler> class numeric_specs_checker {
 public:
  FMT_CONSTEXPR numeric_specs_checker(ErrorHandler& eh, detail::type arg_type)
      : error_handler_(eh), arg_type_(arg_type) {}

  FMT_CONSTEXPR void require_numeric_argument() {
    if (!is_arithmetic_type(arg_type_))
      error_handler_.on_error("format specifier requires numeric argument");
  }

  FMT_CONSTEXPR void check_sign() {
    require_numeric_argument();
    if (is_integral_type(arg_type_) && arg_type_ != type::int_type &&
        arg_type_ != type::long_long_type && arg_type_ != type::char_type) {
      error_handler_.on_error("format specifier requires signed argument");
    }
  }

  FMT_CONSTEXPR void check_precision() {
    if (is_integral_type(arg_type_) || arg_type_ == type::pointer_type)
      error_handler_.on_error("precision not allowed for this argument type");
  }

 private:
  ErrorHandler& error_handler_;
  detail::type arg_type_;
};

```cpp

这段代码定义了一个名为`specs_checker`的模板类，用于检查输入参数的类型是否与模板实参的类型一致。这个类继承自`Handler`类，可能用来创建一个格式化的上下文。

该类的实现主要实现了以下几个方法：

1. `on_align`：用于设置输入参数的对齐类型。
2. `on_plus`：用于设置输入参数的正号。
3. `on_minus`：用于设置输入参数的负号。
4. `on_space`：用于设置输入参数的空格。
5. `on_hash`：用于设置输入参数的哈希类型。
6. `on_zero`：用于设置输入参数的零值。
7. `end_precision`：用于结束输入参数的精度检查。

通过使用这些方法，可以确保格式化上下文中的输入参数符合要求，从而使代码更加健壮和可靠。


```
// A format specifier handler that checks if specifiers are consistent with the
// argument type.
template <typename Handler> class specs_checker : public Handler {
 private:
  numeric_specs_checker<Handler> checker_;

  // Suppress an MSVC warning about using this in initializer list.
  FMT_CONSTEXPR Handler& error_handler() { return *this; }

 public:
  FMT_CONSTEXPR specs_checker(const Handler& handler, detail::type arg_type)
      : Handler(handler), checker_(error_handler(), arg_type) {}

  FMT_CONSTEXPR specs_checker(const specs_checker& other)
      : Handler(other), checker_(error_handler(), other.arg_type_) {}

  FMT_CONSTEXPR void on_align(align_t align) {
    if (align == align::numeric) checker_.require_numeric_argument();
    Handler::on_align(align);
  }

  FMT_CONSTEXPR void on_plus() {
    checker_.check_sign();
    Handler::on_plus();
  }

  FMT_CONSTEXPR void on_minus() {
    checker_.check_sign();
    Handler::on_minus();
  }

  FMT_CONSTEXPR void on_space() {
    checker_.check_sign();
    Handler::on_space();
  }

  FMT_CONSTEXPR void on_hash() {
    checker_.require_numeric_argument();
    Handler::on_hash();
  }

  FMT_CONSTEXPR void on_zero() {
    checker_.require_numeric_argument();
    Handler::on_zero();
  }

  FMT_CONSTEXPR void end_precision() { checker_.check_precision(); }
};

```cpp

这段代码定义了一个模板类 `Handler`，模板参数包括一个模板类型参数 `typename` 和一个 `ErrorHandler` 类型参数。

这个模板类定义了一个名为 `get_dynamic_spec` 的函数，这个函数接收两个模板类型参数 `FormatArg` 和 `ErrorHandler`。

函数的实现使用了模板元编程技术，首先通过 `visit_format_arg` 函数对 `ErrorHandler` 类型的模板参数进行初始化，然后比较生成的值和 `to_unsigned(max_value<int>())` 的大小，如果生成的值大于最大值，则调用 `eh.on_error` 函数输出一个错误。最后，返回生成的 `int` 值。

接着定义了一个模板类 `auto_id`，模板参数为 `Context` 和 `ID`，这个模板类定义了一个名为 `get_arg` 的函数，接收一个 `Context` 类型的模板参数和一个 `ID` 类型的模板参数。

这个 `get_arg` 函数在接收 `ID` 类型的模板参数时，首先尝试从 `Context` 类的 `arg` 函数中获取该模板类型的参数，如果 `arg` 函数返回 `null` 表示没有找到该参数，则会调用 `ctx.on_error` 函数输出一个错误。

最终，该代码定义了一个模板类 `Handler` 和一个模板函数 `get_dynamic_spec`，以及一个模板类 `auto_id`。


```
template <template <typename> class Handler, typename FormatArg,
          typename ErrorHandler>
FMT_CONSTEXPR int get_dynamic_spec(FormatArg arg, ErrorHandler eh) {
  unsigned long long value = visit_format_arg(Handler<ErrorHandler>(eh), arg);
  if (value > to_unsigned(max_value<int>())) eh.on_error("number is too big");
  return static_cast<int>(value);
}

struct auto_id {};

template <typename Context, typename ID>
FMT_CONSTEXPR typename Context::format_arg get_arg(Context& ctx, ID id) {
  auto arg = ctx.arg(id);
  if (!arg) ctx.on_error("argument not found");
  return arg;
}

```cpp

This is a C++ class that implements the `specs_handler` template, which is used to handle the specification of widths and precisions of format elements in an `fmt` object.

The class has a dependency on the `Context` class, which provides the error handling for the `parse_ctx` object passed to the constructor.

The class has three member functions: `on_dynamic_width`, `on_dynamic_precision`, and `on_error`.

The `on_dynamic_width` function is used to set the width of a specified format element based on a dynamic width spec, which is obtained by calling the `get_dynamic_spec` function from the `basic_format_specs` class.

The `on_dynamic_precision` function is used to set the precision of a specified format element based on a dynamic precision spec, which is obtained by calling the `get_dynamic_spec` function from the `basic_format_specs` class.

The `on_error` function is used to handle errors that occur during the parsing of the format string.

The `specs_handler` class can be instantiated and passed to a `fmt` object to create a handler for a specified format element, such as `fmt::FmtObject()`.


```
// The standard format specifier handler with checking.
template <typename ParseContext, typename Context>
class specs_handler : public specs_setter<typename Context::char_type> {
 public:
  using char_type = typename Context::char_type;

  FMT_CONSTEXPR specs_handler(basic_format_specs<char_type>& specs,
                              ParseContext& parse_ctx, Context& ctx)
      : specs_setter<char_type>(specs),
        parse_context_(parse_ctx),
        context_(ctx) {}

  template <typename Id> FMT_CONSTEXPR void on_dynamic_width(Id arg_id) {
    this->specs_.width = get_dynamic_spec<width_checker>(
        get_arg(arg_id), context_.error_handler());
  }

  template <typename Id> FMT_CONSTEXPR void on_dynamic_precision(Id arg_id) {
    this->specs_.precision = get_dynamic_spec<precision_checker>(
        get_arg(arg_id), context_.error_handler());
  }

  void on_error(const char* message) { context_.on_error(message); }

 private:
  // This is only needed for compatibility with gcc 4.4.
  using format_arg = typename Context::format_arg;

  FMT_CONSTEXPR format_arg get_arg(auto_id) {
    return detail::get_arg(context_, parse_context_.next_arg_id());
  }

  FMT_CONSTEXPR format_arg get_arg(int arg_id) {
    parse_context_.check_arg_id(arg_id);
    return detail::get_arg(context_, arg_id);
  }

  FMT_CONSTEXPR format_arg get_arg(basic_string_view<char_type> arg_id) {
    parse_context_.check_arg_id(arg_id);
    return detail::get_arg(context_, arg_id);
  }

  ParseContext& parse_context_;
  Context& context_;
};

```cpp

这段代码定义了一个名为 `arg_id_kind` 的枚举类型，包括 `none`、`index` 和 `name` 三种可能的值。

接着，定义了一个模板类 `arg_ref` 来表示任意类型的参数引用。这个模板类包含一个 `arg_ref()` 构造函数和一个 `arg_ref<T>(int index)` 扩展构造函数，分别用于初始化参数引用和指定参数的索引值。

另外，还定义了一个 `arg_ref<T>(basic_string_view<const char *> name)` 扩展构造函数，用于初始化参数引用并指定参数的名称为 `basic_string_view<const char *>` 类型。

`arg_ref` 模板类中的 `value` 成员 variable 是一个结构体，包含一个整数 `index` 和一个字符串view类型的成员变量 `name`。这个结构体是用来存储参数引用的类型信息。

最后，`arg_id_kind` 枚举类型的成员变量 `kind` 被声明为公有的，并且是一个整数类型的指针变量。


```
enum class arg_id_kind { none, index, name };

// An argument reference.
template <typename Char> struct arg_ref {
  FMT_CONSTEXPR arg_ref() : kind(arg_id_kind::none), val() {}

  FMT_CONSTEXPR explicit arg_ref(int index)
      : kind(arg_id_kind::index), val(index) {}
  FMT_CONSTEXPR explicit arg_ref(basic_string_view<Char> name)
      : kind(arg_id_kind::name), val(name) {}

  FMT_CONSTEXPR arg_ref& operator=(int idx) {
    kind = arg_id_kind::index;
    val.index = idx;
    return *this;
  }

  arg_id_kind kind;
  union value {
    FMT_CONSTEXPR value(int id = 0) : index{id} {}
    FMT_CONSTEXPR value(basic_string_view<Char> n) : name(n) {}

    int index;
    basic_string_view<Char> name;
  } val;
};

```cpp

This is a C++ class that defines a handler for dynamic width and precision arguments in a JSON parsing context. The class has a method for setting the dynamic spec prototype and defines a function for handling errors.

The `dynamic_specs_handler` class has a pure implementation, which means it does not have a default implementation. The `make_arg_ref` function is used to convert an index into a reference to an argument of the specified type.

The `on_dynamic_width` and `on_dynamic_precision` functions are used to set the width and precision of the specified argument. The `on_error` function is used to handle any errors that may occur during the parsing process.


```
// Format specifiers with width and precision resolved at formatting rather
// than parsing time to allow re-using the same parsed specifiers with
// different sets of arguments (precompilation of format strings).
template <typename Char>
struct dynamic_format_specs : basic_format_specs<Char> {
  arg_ref<Char> width_ref;
  arg_ref<Char> precision_ref;
};

// Format spec handler that saves references to arguments representing dynamic
// width and precision to be resolved at formatting time.
template <typename ParseContext>
class dynamic_specs_handler
    : public specs_setter<typename ParseContext::char_type> {
 public:
  using char_type = typename ParseContext::char_type;

  FMT_CONSTEXPR dynamic_specs_handler(dynamic_format_specs<char_type>& specs,
                                      ParseContext& ctx)
      : specs_setter<char_type>(specs), specs_(specs), context_(ctx) {}

  FMT_CONSTEXPR dynamic_specs_handler(const dynamic_specs_handler& other)
      : specs_setter<char_type>(other),
        specs_(other.specs_),
        context_(other.context_) {}

  template <typename Id> FMT_CONSTEXPR void on_dynamic_width(Id arg_id) {
    specs_.width_ref = make_arg_ref(arg_id);
  }

  template <typename Id> FMT_CONSTEXPR void on_dynamic_precision(Id arg_id) {
    specs_.precision_ref = make_arg_ref(arg_id);
  }

  FMT_CONSTEXPR void on_error(const char* message) {
    context_.on_error(message);
  }

 private:
  using arg_ref_type = arg_ref<char_type>;

  FMT_CONSTEXPR arg_ref_type make_arg_ref(int arg_id) {
    context_.check_arg_id(arg_id);
    return arg_ref_type(arg_id);
  }

  FMT_CONSTEXPR arg_ref_type make_arg_ref(auto_id) {
    return arg_ref_type(context_.next_arg_id());
  }

  FMT_CONSTEXPR arg_ref_type make_arg_ref(basic_string_view<char_type> arg_id) {
    context_.check_arg_id(arg_id);
    basic_string_view<char_type> format_str(
        context_.begin(), to_unsigned(context_.end() - context_.begin()));
    return arg_ref_type(arg_id);
  }

  dynamic_format_specs<char_type>& specs_;
  ParseContext& context_;
};

```cpp

这段代码是一个C++语言的模板函数，名为“parse_arg_id”。它接受两个模板参数，一个是要解析的字符串'<br />
tchar类型的IDHandler类型的函数指针，另一个是输出此字符串时使用的模板参数'<br />
tchar类型的IDHandler类型的函数指针。函数的作用是 parsing out the arguments from a given format string by calling the function handle.<br />
truethere is no need to explain the code, the function will do the job.<br />


```
template <typename Char, typename IDHandler>
FMT_CONSTEXPR const Char* parse_arg_id(const Char* begin, const Char* end,
                                       IDHandler&& handler) {
  FMT_ASSERT(begin != end, "");
  Char c = *begin;
  if (c == '}' || c == ':') {
    handler();
    return begin;
  }
  if (c >= '0' && c <= '9') {
    int index = 0;
    if (c != '0')
      index = parse_nonnegative_int(begin, end, handler);
    else
      ++begin;
    if (begin == end || (*begin != '}' && *begin != ':'))
      handler.on_error("invalid format string");
    else
      handler(index);
    return begin;
  }
  if (!is_name_start(c)) {
    handler.on_error("invalid format string");
    return begin;
  }
  auto it = begin;
  do {
    ++it;
  } while (it != end && (is_name_start(c = *it) || ('0' <= c && c <= '9')));
  handler(basic_string_view<Char>(begin, to_unsigned(it - begin)));
  return it;
}

```cpp

这段代码定义了一个名为`width_adapter`的模板结构体，用于将`SpecHandler`类型和`IDHandler`API与动态宽度进行适配。

该结构体包括一个私有成员变量`handler`，一个公有成员函数`operator()`，和一个可选的成员函数`on_error()`。

`operator()`函数用于在模板实参被绑定到`IDHandler`实例时，动态地绑定到操作对象（例如`handler`）上，以实现宽度动态调整的功能。通过`operator()`函数，可以调用`IDHandler`中预定义的`on_dynamic_width()`函数，传入不同类型的参数，包括自动ID、ID和字符串等。

另外，`on_error()`函数用于记录错误信息，并将其传递给`handler`，以便于在模板定义中捕获和处理错误。


```
// Adapts SpecHandler to IDHandler API for dynamic width.
template <typename SpecHandler, typename Char> struct width_adapter {
  explicit FMT_CONSTEXPR width_adapter(SpecHandler& h) : handler(h) {}

  FMT_CONSTEXPR void operator()() { handler.on_dynamic_width(auto_id()); }
  FMT_CONSTEXPR void operator()(int id) { handler.on_dynamic_width(id); }
  FMT_CONSTEXPR void operator()(basic_string_view<Char> id) {
    handler.on_dynamic_width(id);
  }

  FMT_CONSTEXPR void on_error(const char* message) {
    handler.on_error(message);
  }

  SpecHandler& handler;
};

```cpp

这段代码定义了一个名为`precision_adapter`的结构体，其模板参数为`SpecHandler`和`Char`。这个结构体实现了一个对`IDHandler` API的动态精度适配，可以在需要时动态地调整精度。

该结构体包含了一个`FMT_CONSTEXPR`类型的成员变量`handler`，这个成员变量是一个引用，指向一个实现了`IDHandler`接口的`SpecHandler`类型的对象。

该结构体还包含三个`FMT_CONSTEXPR`类型的成员函数：`operator()`函数，分别对`auto_id()`、`id`和`id`类型的参数进行调用。这些函数实现了对动态 precision的支持，可以在需要时动态地调整精度。

此外，该结构体还包括一个`on_error()`函数，用于在出现错误时进行处理。这个函数将`handler`对象所实现的`on_error()`函数作为实参传递，如果`on_error()`函数在`handler`对象中没有实现，则会自动调用它。


```
// Adapts SpecHandler to IDHandler API for dynamic precision.
template <typename SpecHandler, typename Char> struct precision_adapter {
  explicit FMT_CONSTEXPR precision_adapter(SpecHandler& h) : handler(h) {}

  FMT_CONSTEXPR void operator()() { handler.on_dynamic_precision(auto_id()); }
  FMT_CONSTEXPR void operator()(int id) { handler.on_dynamic_precision(id); }
  FMT_CONSTEXPR void operator()(basic_string_view<Char> id) {
    handler.on_dynamic_precision(id);
  }

  FMT_CONSTEXPR void on_error(const char* message) {
    handler.on_error(message);
  }

  SpecHandler& handler;
};

```cpp

这段代码定义了一个名为`next_code_point`的函数模板，它接受两个参数：一个字符型变量`begin`和一个字符型变量`end`，它们分别表示要搜索的子字符串的起始和结束位置。函数返回开始位置。

该函数实现了一个简单的算法，用于查找给定字符串中的第一个连续的'<'或'>'符号的位置。通过循环遍历字符串，并检查当前字符与'<'或'>'的关系，从而确定第一个连续的'<'或'>'符号的位置。如果找到第一个连续的符号位置，则返回该位置。如果没有找到第一个连续的符号位置，则返回字符串的最后一个字符，因为最后一个字符也可能是连续的符号。

该函数模板是通过宏定义实现的，它的实现与模板函数本身无关。这个模板函数可以被用来定义一个字符串类型的变量，例如：
```
const char* str = "hello";
const char* position = next_code_point(str.begin, str.end); // 返回子的起始位置
```cpp



```
template <typename Char>
FMT_CONSTEXPR const Char* next_code_point(const Char* begin, const Char* end) {
  if (const_check(sizeof(Char) != 1) || (*begin & 0x80) == 0) return begin + 1;
  do {
    ++begin;
  } while (begin != end && (*begin & 0xc0) == 0x80);
  return begin;
}

// Parses fill and alignment.
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_align(const Char* begin, const Char* end,
                                      Handler&& handler) {
  FMT_ASSERT(begin != end, "");
  auto align = align::none;
  auto p = next_code_point(begin, end);
  if (p == end) p = begin;
  for (;;) {
    switch (static_cast<int>(*p)) {
    case '<':
      align = align::left;
      break;
    case '>':
      align = align::right;
      break;
```cpp

这段代码是一个C++语言中的函数，名为“handler”。它是一个不输出函数源代码的函数。这段代码的主要作用是判断字符'FMT_DEPRECATED_NUMERIC_ALIGN'是否匹配某种情况，并相应地进行调整。

具体来说，这段代码首先检查'FMT_DEPRECATED_NUMERIC_ALIGN'是否与'='或'^'匹配。如果是，就执行相应的case语句。在case'='或case'^'中，如果'='或'^'匹配，就执行相应的break语句，跳过当前case语句。否则，如果'='或'^'不匹配，就执行相应的else语句。

在else语句中，如果当前已经处理的字符'FMT_DEPRECATED_NUMBERIC_ALIGN'在当前字符串中，就返回处理函数。否则，就执行相应的align语句，并将调整后的字符'FMT_DEPRECATED_NUMBERIC_ALIGN'存储回当前字符串中。然后，就继续处理下一个字符。

最后，如果当前处理的字符'FMT_DEPRECATED_NUMBERIC_ALIGN'既不在当前字符串中，也不是'='或'^'，那么该函数就返回begin。


```
#if FMT_DEPRECATED_NUMERIC_ALIGN
    case '=':
      align = align::numeric;
      break;
#endif
    case '^':
      align = align::center;
      break;
    }
    if (align != align::none) {
      if (p != begin) {
        auto c = *begin;
        if (c == '{')
          return handler.on_error("invalid fill character '{'"), begin;
        handler.on_fill(basic_string_view<Char>(begin, to_unsigned(p - begin)));
        begin = p + 1;
      } else
        ++begin;
      handler.on_align(align);
      break;
    } else if (p == begin) {
      break;
    }
    p = begin;
  }
  return begin;
}

```cpp

这段代码是一个C++语言的模板函数，名为“parse_width”。

它的参数包括三个类型参数：

- Char，表示输入数据使用的字符类型；
- Handler，表示一个处理函数，用于处理解析结果；
- const Char* begin，表示开始解析的位置，end，表示结束解析的位置。

函数的作用是解析字符串中的一维数值，并将其存储在Handler中指定的位置。

具体实现过程如下：

1. 如果解析开始的位置为0，且包含数字0-9，则直接解析为非负整数，并传递给Handler。

2. 如果解析开始的位置为“{”，则将解析开始位置加1，并尝试解析一个表达式，如果解析成功则继续解析，直到遇到“}”或遇到非法字符。

3. 如果解析成功，则返回Handler指定位置的结果，否则返回错误信息并返回解析开始位置。


```
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_width(const Char* begin, const Char* end,
                                      Handler&& handler) {
  FMT_ASSERT(begin != end, "");
  if ('0' <= *begin && *begin <= '9') {
    handler.on_width(parse_nonnegative_int(begin, end, handler));
  } else if (*begin == '{') {
    ++begin;
    if (begin != end)
      begin = parse_arg_id(begin, end, width_adapter<Handler, Char>(handler));
    if (begin == end || *begin != '}')
      return handler.on_error("invalid format string"), begin;
    ++begin;
  }
  return begin;
}

```cpp

这段代码定义了一个名为 parse_precision 的模板函数，它接受两个参数：一个 Char 类型的变量 begin 和一个 Char 类型的变量 end，以及一个 Handler 类型的参数 handler。函数的功能是解析参数中指定格式的字符串，并将其转换为具有指定精度的字符串，如果解析过程中出现错误，则返回相应的错误信息。

具体来说，这段代码首先定义了一个 const 类型的变量 parse_precision，它接受两个整数类型的变量 begin 和 end，以及一个 Handler 类型的变量 handler。变量 begin 和 end 分别代表字符串的起始和终止位置，handler 则是一个将参数转换为指定格式的函数。

在函数体中，首先将 begin 变量加 1，并检查是否已经到达 end 位置。如果是，则将当前字符 '{' 取出，并将 begin 变量移动到参数 begin 和 end 之间的位置。然后，如果当前字符是 '0' 到 '9' 之间的数字，则执行 handler.on_precision 函数，并将 parse_nonnegative_int 函数返回的结果作为参数传递给 handler。

否则，如果当前字符是 '{'，则将 begin 变量移动到参数 begin 和 end 之间的位置，并将新字符 '{' 的位置记录在变量中。然后，继续执行 parse_arg_id 函数，并将 precision_adapter 函数作为参数传递给 handler，将解析结果存储在变量中。

如果在执行过程中，begin 变量已经到达 end 位置或者 '{' 字符没有取出，则执行 handler.on_error "missing precision specifier"，返回错误信息。

最后，函数使用了 handler.end_precision 函数来清理尚未清理的解析结果，然后将结果返回。


```
template <typename Char, typename Handler>
FMT_CONSTEXPR const Char* parse_precision(const Char* begin, const Char* end,
                                          Handler&& handler) {
  ++begin;
  auto c = begin != end ? *begin : Char();
  if ('0' <= c && c <= '9') {
    handler.on_precision(parse_nonnegative_int(begin, end, handler));
  } else if (c == '{') {
    ++begin;
    if (begin != end) {
      begin =
          parse_arg_id(begin, end, precision_adapter<Handler, Char>(handler));
    }
    if (begin == end || *begin++ != '}')
      return handler.on_error("invalid format string"), begin;
  } else {
    return handler.on_error("missing precision specifier"), begin;
  }
  handler.end_precision();
  return begin;
}

```cpp

这段代码是一个C++模板函数，名为parse_format_specs，用于解析文本格式并通知相应的解析器。它接受两个参数，一个是格式规范的起始位置和结束位置，另一个是一个实现了特定格式规范的解析器。

函数首先检查输入是否结束，如果是，则返回起始位置。如果不是，它将调用parse_align函数来调整字符串的结束位置，并确保解析器在正确的位置开始解析。

接下来，函数开始解析正则表达式。根据给定的起始和结束位置，它将解析+、-或空格等字符。如果解析成功，它将通知解析器适当的回调函数。

然后，函数将开始解析零 flag。如果解析成功，它将通知解析器适当的回调函数。

接下来，函数尝试解析精度。如果解析成功，它将通知解析器适当的回调函数。

最后，函数将解析给定的数据类型。如果解析成功，它将通知解析器适当的回调函数，并返回起始位置。


```
// Parses standard format specifiers and sends notifications about parsed
// components to handler.
template <typename Char, typename SpecHandler>
FMT_CONSTEXPR const Char* parse_format_specs(const Char* begin, const Char* end,
                                             SpecHandler&& handler) {
  if (begin == end || *begin == '}') return begin;

  begin = parse_align(begin, end, handler);
  if (begin == end) return begin;

  // Parse sign.
  switch (static_cast<char>(*begin)) {
  case '+':
    handler.on_plus();
    ++begin;
    break;
  case '-':
    handler.on_minus();
    ++begin;
    break;
  case ' ':
    handler.on_space();
    ++begin;
    break;
  }
  if (begin == end) return begin;

  if (*begin == '#') {
    handler.on_hash();
    if (++begin == end) return begin;
  }

  // Parse zero flag.
  if (*begin == '0') {
    handler.on_zero();
    if (++begin == end) return begin;
  }

  begin = parse_width(begin, end, handler);
  if (begin == end) return begin;

  // Parse precision.
  if (*begin == '.') {
    begin = parse_precision(begin, end, handler);
  }

  // Parse type.
  if (begin != end && *begin != '}') handler.on_type(*begin++);
  return begin;
}

```cpp

这段代码定义了两个版本的 `find` 函数，用于在二进制搜索中查找特定值。

第一个函数 `find` 接收一个指针 `first`，一个指针 `last`，以及要查找的特定值 `value`。它通过从 `first` 开始搜索，直到找到 `value` 或者到达 `last`，然后返回 `true`。这个版本的函数实现是针对 `IS_CONSTEXPR` 模板类型的。

第二个函数 `find` 的实现与第一个函数非常相似，但是它接收一个字符串 `first`，一个字符串 `last`，以及要查找的特定字符 `value`。它通过在字符串的起始和结束位置查找该字符，并返回它的第一个出现的位置。这个版本的函数实现是针对 `false` 类型的。

这两个版本的函数实现都有一个共同点，即它们都使用 `std::memchr` 函数来查找二进制搜索中的目标值。这个函数函数接受两个指针 `first` 和 `last`，以及一个目标值 `value`。它返回第一个在给定范围内出现的字符串位置，如果任何位置被找到则返回该位置。这个函数需要在使用之前定义。


```
// Return the result via the out param to workaround gcc bug 77539.
template <bool IS_CONSTEXPR, typename T, typename Ptr = const T*>
FMT_CONSTEXPR bool find(Ptr first, Ptr last, T value, Ptr& out) {
  for (out = first; out != last; ++out) {
    if (*out == value) return true;
  }
  return false;
}

template <>
inline bool find<false, char>(const char* first, const char* last, char value,
                              const char*& out) {
  out = static_cast<const char*>(
      std::memchr(first, value, detail::to_unsigned(last - first)));
  return out != nullptr;
}

```cpp

这段代码定义了一个模板结构体 `id_adapter`，可以接受两个类型参数 `Handler` 和 `Char`。

这个结构体包含一个 `Handler` 类型的成员变量 `handler`，表示处理函数的引用，以及一个整型变量 `arg_id`，表示传递给 `Handler` 的参数在函数中的参数 ID。

该结构体定义了三个成员函数：

1. `operator()` 函数，是一个模板函数，用于在 `id_adapter` 结构体中实现运算符重载。这个函数有两个参数：第一个参数是一个 `id_adapter` 结构体，第二个参数是要在 `Handler` 中引用的参数 ID。函数的作用是在 `Handler` 对象的 `on_arg_id()` 函数中传入参数 ID，并返回原始的 `Handler` 对象。

2. `operator()` 函数，与上面同名的另一个函数，但第二个参数是整型而非 `id_adapter` 结构体。这个函数的作用与第一个相同，但是需要手动引用 `Handler` 对象的 `on_arg_id()` 函数。

3. `operator()` 函数，第三个成员函数，是一个带有参数 `id` 的 `id_adapter` 结构体函数。这个函数的作用是在 `Handler` 对象的 `on_arg_id()` 函数中传入参数 `id`。

4. `on_error()` 函数，是一个 `void` 类型的函数，用于在 `Handler` 对象中处理错误信息。该函数接收一个 `const char*` 类型的参数 `message`，会在 `Handler` 对象的 `on_error()` 函数中将其记录下来。


```
template <typename Handler, typename Char> struct id_adapter {
  Handler& handler;
  int arg_id;

  FMT_CONSTEXPR void operator()() { arg_id = handler.on_arg_id(); }
  FMT_CONSTEXPR void operator()(int id) { arg_id = handler.on_arg_id(id); }
  FMT_CONSTEXPR void operator()(basic_string_view<Char> id) {
    arg_id = handler.on_arg_id(id);
  }
  FMT_CONSTEXPR void on_error(const char* message) {
    handler.on_error(message);
  }
};

template <typename Char, typename Handler>
```cpp

该函数的作用是解析输入的格式字符串，并返回其后的第一个非空字符以及处理函数的返回值。它主要通过以下步骤实现：

1. 将输入的begin和end指针分别指向当前字符位置；
2. 如果当前字符为'}'，则表示已经到达结束位置，返回处理函数的错误提示并结束；
3. 如果当前字符为'{'，则表示开始一个新的一组参数，将对应于这个参数的整数复制到begin所指向的字符位置，并将begin向后移动；
4. 如果当前字符为','，则表示要插入一个占位符，查找begin所指向的下一个字符是否为'}'，如果是，则将这个占位符的格式ID插入到begin所指向的字符位置，并将begin向后移动；
5. 如果当前字符一直循环下去，且begin没有到达结束字符'}'，则表示输入的格式字符串中可能存在错误的语法，返回处理函数的错误提示并结束；
6. 否则，继续解析格式字符串，并将解析结果返回给begion。


```
FMT_CONSTEXPR const Char* parse_replacement_field(const Char* begin,
                                                  const Char* end,
                                                  Handler&& handler) {
  ++begin;
  if (begin == end) return handler.on_error("invalid format string"), end;
  if (static_cast<char>(*begin) == '}') {
    handler.on_replacement_field(handler.on_arg_id(), begin);
  } else if (*begin == '{') {
    handler.on_text(begin, begin + 1);
  } else {
    auto adapter = id_adapter<Handler, Char>{handler, 0};
    begin = parse_arg_id(begin, end, adapter);
    Char c = begin != end ? *begin : Char();
    if (c == '}') {
      handler.on_replacement_field(adapter.arg_id, begin);
    } else if (c == ':') {
      begin = handler.on_format_specs(adapter.arg_id, begin + 1, end);
      if (begin == end || *begin != '}')
        return handler.on_error("unknown format specifier"), end;
    } else {
      return handler.on_error("missing '}' in format string"), end;
    }
  }
  return begin + 1;
}

```cpp

This code appears to be a implementation of the `fmt` function, which is part of the `fmt` package. This function takes a format string and a number of arguments and returns the formatted string or an error if the format string is invalid.

The `fmt` function has several overloads for different use cases. For example, there is a single overload for the `fmt` function, which takes a format string and a single argument, the `fmt` function and the number of arguments passed to it.

This overload first checks if the argument is negative and if so it returns an error. If the argument is not negative, it then attempts to retrieve the number of arguments passed to the `fmt` function using the `count_零_based` function. If the number of arguments passed is zero or less, it returns an error. If the number of arguments passed is greater than zero, it then returns the formatted string.

Another overload is for the `fmt` function, which takes a format string and a variable number of arguments. This overload returns the string that would be formatted if the `fmt` function were called with the specified arguments.

The last overload is for the `fmt` function, which takes a format string and a variable number of arguments. This overload returns the string that would be formatted if the `fmt` function were called with the specified arguments.


```
template <bool IS_CONSTEXPR, typename Char, typename Handler>
FMT_CONSTEXPR_DECL FMT_INLINE void parse_format_string(
    basic_string_view<Char> format_str, Handler&& handler) {
  auto begin = format_str.data();
  auto end = begin + format_str.size();
  if (end - begin < 32) {
    // Use a simple loop instead of memchr for small strings.
    const Char* p = begin;
    while (p != end) {
      auto c = *p++;
      if (c == '{') {
        handler.on_text(begin, p - 1);
        begin = p = parse_replacement_field(p - 1, end, handler);
      } else if (c == '}') {
        if (p == end || *p != '}')
          return handler.on_error("unmatched '}' in format string");
        handler.on_text(begin, p);
        begin = ++p;
      }
    }
    handler.on_text(begin, end);
    return;
  }
  struct writer {
    FMT_CONSTEXPR void operator()(const Char* pbegin, const Char* pend) {
      if (pbegin == pend) return;
      for (;;) {
        const Char* p = nullptr;
        if (!find<IS_CONSTEXPR>(pbegin, pend, '}', p))
          return handler_.on_text(pbegin, pend);
        ++p;
        if (p == pend || *p != '}')
          return handler_.on_error("unmatched '}' in format string");
        handler_.on_text(pbegin, p);
        pbegin = p + 1;
      }
    }
    Handler& handler_;
  } write{handler};
  while (begin != end) {
    // Doing two passes with memchr (one for '{' and another for '}') is up to
    // 2.5x faster than the naive one-pass implementation on big format strings.
    const Char* p = begin;
    if (*begin != '{' && !find<IS_CONSTEXPR>(begin + 1, end, '{', p))
      return write(begin, end);
    write(begin, p);
    begin = parse_replacement_field(p, end, handler);
  }
}

```cpp

这段代码定义了一个名为 `parse_format_specs` 的模板函数，它接受一个 `ParseContext` 对象作为参数，并返回一个指向 `char_type` 类型变量的指针。

函数内部，首先定义了 `char_type` 类型变量，用于表示输入数据和输出数据使用的字符类型。然后定义了一个名为 `context` 的 `buffer_context` 对象，用于管理输入数据。

接着定义了一个名为 `mapped_type` 的条件类型变量。该类型变量根据输入参数 `T` 的类型来决定是否为 `custom_type`，如果是，则执行下面的操作，否则执行下面的操作。此处使用了 `conditional_t` 特性，可以方便地分支判断。

接着定义了一个名为 `f` 的函数，它接收一个 `mapped_type` 类型的变量、一个 `char_type` 类型的变量和一个 `ParseContext` 对象作为参数。函数内部使用了 `conditional_t` 特性，分支判断 `mapped_type` 类型变量是否为常量类型，如果是，则执行下面的操作，否则执行下面的操作。此处使用了 `formatter` 函数和 `detail::fallback_formatter` 函数，对 `mapped_type` 类型的变量进行格式化。

最后，函数返回了一个指向 `char_type` 类型变量的指针，它作为 `ParseContext` 对象的格式化参数传递给 `formatter` 函数进行格式化输出。


```
template <typename T, typename ParseContext>
FMT_CONSTEXPR const typename ParseContext::char_type* parse_format_specs(
    ParseContext& ctx) {
  using char_type = typename ParseContext::char_type;
  using context = buffer_context<char_type>;
  using mapped_type =
      conditional_t<detail::mapped_type_constant<T, context>::value !=
                        type::custom_type,
                    decltype(arg_mapper<context>().map(std::declval<T>())), T>;
  auto f = conditional_t<has_formatter<mapped_type, context>::value,
                         formatter<mapped_type, char_type>,
                         detail::fallback_formatter<T, char_type>>();
  return f.parse(ctx);
}

```cpp

This is a Rust implementation of the TaskRunner class from the Rust crate追求。TaskRunner is designed to execute a function with a given arguments in a given context. It does this by providing a convenient interface for passing arguments, formatting, and handling the output of the function.

The `TaskRunner` struct has the following fields:

* `context`: The context that the function will be executed in. This is initialized in the constructor and can be used to advance to the next argument, formatting, or handle.
* `arg_types`: A map of the expected argument types for the function. This is initialized in the constructor and defaults to `Vec<Type>::empty()`.
* `output_type`: The type of the output of the function. This is initialized in the constructor and defaults to `Vec<Type>::empty()`.
* `arguments`: A vector of arguments for the function. This is initialized in the constructor and defaults to `Vec<(Type, bool)>::empty()`.
* `format_specs`: A vector of formatting specifications for the function. This is initialized in the constructor and defaults to `Vec<basic_format_specs<Char>>::empty()`.
* `handler`: An optional function that will handle the formatting of the arguments. This is required if `format_specs` is not empty and has at least one formatting specification.
* `is_function_ending`: An optional boolean that indicates whether the function is ending or not. This can be used to skip processing arguments if the function is ending.

The `on_arg_id()`, `on_arg_id()`, and `on_arg_id()` functions are used to access the arguments by their index, ID, or by a given `basic_string_view` of an argument's ID.

The `on_replacement_field()` function is used to access the `format_specs()` field of the `basic_format_specs<Char>` object, which is the vector of formatting specifications for the function.

The `on_format_specs()` function is used to provide the `on_arg_id()` function with the formatting specifications for the arguments.

The `on_error()` function is used to handle errors, such as an argument not found in the function's arguments.


```
template <typename ArgFormatter, typename Char, typename Context>
struct format_handler : detail::error_handler {
  basic_format_parse_context<Char> parse_context;
  Context context;

  format_handler(typename ArgFormatter::iterator out,
                 basic_string_view<Char> str,
                 basic_format_args<Context> format_args, detail::locale_ref loc)
      : parse_context(str), context(out, format_args, loc) {}

  void on_text(const Char* begin, const Char* end) {
    auto size = to_unsigned(end - begin);
    auto out = context.out();
    auto&& it = reserve(out, size);
    it = std::copy_n(begin, size, it);
    context.advance_to(out);
  }

  int on_arg_id() { return parse_context.next_arg_id(); }
  int on_arg_id(int id) { return parse_context.check_arg_id(id), id; }
  int on_arg_id(basic_string_view<Char> id) {
    int arg_id = context.arg_id(id);
    if (arg_id < 0) on_error("argument not found");
    return arg_id;
  }

  FMT_INLINE void on_replacement_field(int id, const Char*) {
    auto arg = get_arg(context, id);
    context.advance_to(visit_format_arg(
        default_arg_formatter<typename ArgFormatter::iterator, Char>{
            context.out(), context.args(), context.locale()},
        arg));
  }

  const Char* on_format_specs(int id, const Char* begin, const Char* end) {
    advance_to(parse_context, begin);
    auto arg = get_arg(context, id);
    custom_formatter<Context> f(parse_context, context);
    if (visit_format_arg(f, arg)) return parse_context.begin();
    basic_format_specs<Char> specs;
    using parse_context_t = basic_format_parse_context<Char>;
    specs_checker<specs_handler<parse_context_t, Context>> handler(
        specs_handler<parse_context_t, Context>(specs, parse_context, context),
        arg.type());
    begin = parse_format_specs(begin, end, handler);
    if (begin == end || *begin != '}') on_error("missing '}' in format string");
    advance_to(parse_context, begin);
    context.advance_to(
        visit_format_arg(ArgFormatter(context, &parse_context, &specs), arg));
    return begin;
  }
};

```cpp

这段代码定义了一个名为 `compile_parse_context` 的类，用于在编译时检查给定的字符串格式是否符合指定的参数列表。

该类基于 `basic_format_parse_context` 类，其中 `ErrorHandler` 类型被声明为 `error_handler`。

该类的构造函数接受一个字符串格式和一个参数列表，例如：
```php
explicit compile_parse_context(const std::string& format_str, int num_args = max_value<int>(),
                                   ErrorHandler eh = {}) : base(format_str, eh), num_args_(num_args) {}
```cpp
构造函数首先根据给定的参数列表初始化 `base` 对象，然后调用 `next_arg_id()` 函数来获取下一个参数的唯一标识符（ID），并将其存储在 `num_args_` 变量中。

`check_arg_id()` 函数作为 `base::check_arg_id()` 函数的子类，用于在获取参数 ID 时执行必要的错误处理。
```php
int next_arg_id() {
   int id = base::next_arg_id();
   if (id >= num_args_) this->on_error("argument not found");
   return id;
}
```cpp
如果获取到的参数 ID 不在 `num_args_` 参数列表中，该函数会抛出一个错误并返回下一个 ID。
```php
void check_arg_id(int id) {
   base::check_arg_id(id);
   if (id >= num_args_) this->on_error("argument not found");
}
```cpp
该 `compile_parse_context` 类提供了一些额外的功能，例如在获取参数时进行参数 ID 检查，这些功能仅在编译时进行。如果这些功能在运行时进行，会引入相当大的开销，并且可能产生冗余的错误。


```
// A parse context with extra argument id checks. It is only used at compile
// time because adding checks at runtime would introduce substantial overhead
// and would be redundant since argument ids are checked when arguments are
// retrieved anyway.
template <typename Char, typename ErrorHandler = error_handler>
class compile_parse_context
    : public basic_format_parse_context<Char, ErrorHandler> {
 private:
  int num_args_;
  using base = basic_format_parse_context<Char, ErrorHandler>;

 public:
  explicit FMT_CONSTEXPR compile_parse_context(
      basic_string_view<Char> format_str, int num_args = max_value<int>(),
      ErrorHandler eh = {})
      : base(format_str, eh), num_args_(num_args) {}

  FMT_CONSTEXPR int next_arg_id() {
    int id = base::next_arg_id();
    if (id >= num_args_) this->on_error("argument not found");
    return id;
  }

  FMT_CONSTEXPR void check_arg_id(int id) {
    base::check_arg_id(id);
    if (id >= num_args_) this->on_error("argument not found");
  }
  using base::check_arg_id;
};

```cpp

该代码定义了一个名为 `format_string_checker` 的模板类，用于检查文本中是否存在格式字符串。该模板类可以接受不同类型的参数，包括字符类型、错误处理程序类型和参数列表类型。

具体来说，该模板类通过 `basic_string_view` 类型的变量 `format_str` 获取输入文本，并使用 `ErrorHandler` 类型的变量 `eh` 来处理错误。 `parse_funcs_` 是一个指针数组，它包含了 `parse_format_specs` 和 `parse_func` 函数的地址，这些函数用于解析格式字符串和执行替换操作。

该模板类包含以下方法：

- `on_text`：在文本中插入一个空行并返回。
- `on_arg_id`：返回给定的参数在文本中的 ID，包括命名参数。
- `on_arg_id`：返回给定的参数在文本中的 ID，不包括命名参数。
- `on_arg_id`：在文本中插入一个警告并返回 ID，用于指出无法解析的参数 ID。
- `on_replacement_field`：设置或清除指定位置的替换字段。
- `on_format_specs`：返回解析 `format_specs` 中指定 ID 的格式字符串，不包括命名参数。
- `on_error`：在文本中插入错误消息并返回。

该模板类使用了 C++ 的模板元编程特性，因此它可以为不同的输入类型生成特定的行为。


```
template <typename Char, typename ErrorHandler, typename... Args>
class format_string_checker {
 public:
  explicit FMT_CONSTEXPR format_string_checker(
      basic_string_view<Char> format_str, ErrorHandler eh)
      : context_(format_str, num_args, eh),
        parse_funcs_{&parse_format_specs<Args, parse_context_type>...} {}

  FMT_CONSTEXPR void on_text(const Char*, const Char*) {}

  FMT_CONSTEXPR int on_arg_id() { return context_.next_arg_id(); }
  FMT_CONSTEXPR int on_arg_id(int id) { return context_.check_arg_id(id), id; }
  FMT_CONSTEXPR int on_arg_id(basic_string_view<Char>) {
    on_error("compile-time checks don't support named arguments");
    return 0;
  }

  FMT_CONSTEXPR void on_replacement_field(int, const Char*) {}

  FMT_CONSTEXPR const Char* on_format_specs(int id, const Char* begin,
                                            const Char*) {
    advance_to(context_, begin);
    return id < num_args ? parse_funcs_[id](context_) : begin;
  }

  FMT_CONSTEXPR void on_error(const char* message) {
    context_.on_error(message);
  }

 private:
  using parse_context_type = compile_parse_context<Char, ErrorHandler>;
  enum { num_args = sizeof...(Args) };

  // Format specifier parsing function.
  using parse_func = const Char* (*)(parse_context_type&);

  parse_context_type context_;
  parse_func parse_funcs_[num_args > 0 ? num_args : 1];
};

```cpp

这两段代码定义了一个名为`compile_string_to_view`的模板函数，它接受一个字符串参数`s`，并将其转换为`basic_string_view`类型的对象。

第一个模板函数（`compile_string_to_view`）接受一个字符串参数`s`，其中`N`是一个整数。这个函数的作用是将字符串`s`转换为一个`basic_string_view`类型的对象，并在转换过程中处理尾随的空字符串。

第二个模板函数（未命名）接受一个标准字符串视图（`std_string_view`）参数`s`，将其转换为`basic_string_view`类型的对象。这个函数的作用是将字符串视图`s`转换为一个`basic_string_view`类型的对象。


```
// Converts string literals to basic_string_view.
template <typename Char, size_t N>
FMT_CONSTEXPR basic_string_view<Char> compile_string_to_view(
    const Char (&s)[N]) {
  // Remove trailing null character if needed. Won't be present if this is used
  // with raw character array (i.e. not defined as a string).
  return {s,
          N - ((std::char_traits<Char>::to_int_type(s[N - 1]) == 0) ? 1 : 0)};
}

// Converts string_view to basic_string_view.
template <typename Char>
FMT_CONSTEXPR basic_string_view<Char> compile_string_to_view(
    const std_string_view<Char>& s) {
  return {s.data(), s.size()};
}

```cpp

这段代码定义了一个名为 `FMT_STRING_IMPL` 的头文件，该头文件包含了一个名为 `FMT_COMPILE_STRING` 的函数。

该函数的作用是从一个字符串常量 `s` 中计算一个格式化字符串，并将计算得到的字符串返回。在函数体中，首先定义了一个名为 `FMT_COMPILE_STRING` 的内部类型，该类型包含一个名为 `base` 的成员变量。

接着定义了一个名为 `using` 的宏，用于声明一个名为 `char_type` 的类型，该类型是 `s` 的第一个元素的类型。

接着定义了一个名为 `FMT_MAYBE_UNUSED` 的宏，用于声明一个名为 `FMT_CONSTEXPR` 的类型，该类型是一个可以被编译器优化掉的类型，通常用于避免不必要的警告。

然后定义了一个名为 `operator fmt::basic_string_view<char_type>() const` 的函数，该函数用于计算格式化字符串的视图，并将其返回。

最后定义了该函数的返回值，为 `FMT_COMPILE_STRING()`。

该头文件的作用是帮助用户在编译时检查字符串格式是否正确，同时也可以避免在需要时过多地检查 `s` 中的类型。


```
#define FMT_STRING_IMPL(s, base)                                  \
  [] {                                                            \
    /* Use a macro-like name to avoid shadowing warnings. */      \
    struct FMT_COMPILE_STRING : base {                            \
      using char_type = fmt::remove_cvref_t<decltype(s[0])>;      \
      FMT_MAYBE_UNUSED FMT_CONSTEXPR                              \
      operator fmt::basic_string_view<char_type>() const {        \
        return fmt::detail::compile_string_to_view<char_type>(s); \
      }                                                           \
    };                                                            \
    return FMT_COMPILE_STRING();                                  \
  }()

/**
  \rst
  Constructs a compile-time format string from a string literal *s*.

  **Example**::

    // A compile-time error because 'd' is an invalid specifier for strings.
    std::string s = fmt::format(FMT_STRING("{:d}"), "foo");
  \endrst
 */
```cpp

这段代码定义了一个模板类 Handler，其中包含一个名为 handle_dynamic_spec 的函数。

动态规格运算符 (?) 是 C++17 中引入的语法糖，它允许在编译时检查函数参数的类型，并自动推导出返回值的类型。

这段代码的作用是帮助开发人员更方便地编写模板类，特别是在需要根据输入参数的类型来执行不同的操作的情况下。通过使用 dynamic_spec 模板元编程技术，可以编写更简洁、更易于阅读和维护的代码。


```
#define FMT_STRING(s) FMT_STRING_IMPL(s, fmt::compile_string)

template <typename... Args, typename S,
          enable_if_t<(is_compile_string<S>::value), int>>
void check_format_string(S format_str) {
  FMT_CONSTEXPR_DECL auto s = to_string_view(format_str);
  using checker = format_string_checker<typename S::char_type, error_handler,
                                        remove_cvref_t<Args>...>;
  FMT_CONSTEXPR_DECL bool invalid_format =
      (parse_format_string<true>(s, checker(s, {})), true);
  (void)invalid_format;
}

template <template <typename> class Handler, typename Context>
void handle_dynamic_spec(int& value, arg_ref<typename Context::char_type> ref,
                         Context& ctx) {
  switch (ref.kind) {
  case arg_id_kind::none:
    break;
  case arg_id_kind::index:
    value = detail::get_dynamic_spec<Handler>(ctx.arg(ref.val.index),
                                              ctx.error_handler());
    break;
  case arg_id_kind::name:
    value = detail::get_dynamic_spec<Handler>(ctx.arg(ref.val.name),
                                              ctx.error_handler());
    break;
  }
}

```cpp

This is a C++ header file that defines an `arg_formatter` class template.

The `arg_formatter` class is used to format arguments for user-defined functions, such as `fmt::format`.

The `arg_formatter` class takes a `context_type` object, which represents the formatting context, a `basic_format_parse_context<char_type>` pointer, and a pointer to a `format_specs` object.

The `arg_formatter` class has a constructor that initializes the required components of the `arg_formatter` object and a `operator()` member function that formats the given arguments according to the rules defined by the `basic_format_specs` object.

This class can be used in a `fmt::format` function to format arguments for user-defined functions.


```
using format_func = void (*)(detail::buffer<char>&, int, string_view);

FMT_API void format_error_code(buffer<char>& out, int error_code,
                               string_view message) FMT_NOEXCEPT;

FMT_API void report_error(format_func func, int error_code,
                          string_view message) FMT_NOEXCEPT;

/** The default argument formatter. */
template <typename OutputIt, typename Char>
class arg_formatter : public arg_formatter_base<OutputIt, Char> {
 private:
  using char_type = Char;
  using base = arg_formatter_base<OutputIt, Char>;
  using context_type = basic_format_context<OutputIt, Char>;

  context_type& ctx_;
  basic_format_parse_context<char_type>* parse_ctx_;
  const Char* ptr_;

 public:
  using iterator = typename base::iterator;
  using format_specs = typename base::format_specs;

  /**
    \rst
    Constructs an argument formatter object.
    *ctx* is a reference to the formatting context,
    *specs* contains format specifier information for standard argument types.
    \endrst
   */
  explicit arg_formatter(
      context_type& ctx,
      basic_format_parse_context<char_type>* parse_ctx = nullptr,
      format_specs* specs = nullptr, const Char* ptr = nullptr)
      : base(ctx.out(), specs, ctx.locale()),
        ctx_(ctx),
        parse_ctx_(parse_ctx),
        ptr_(ptr) {}

  using base::operator();

  /** Formats an argument of a user-defined type. */
  iterator operator()(typename basic_format_arg<context_type>::handle handle) {
    if (ptr_) advance_to(*parse_ctx_, ptr_);
    handle.format(*parse_ctx_, ctx_);
    return ctx_.out();
  }
};
}  // namespace detail

```cpp

This is a C++ class template that inherits from `std::runtime_error`. It is designed to represent system errors, which are caught and returned by an operating system or a language runtime. The class includes a private constructor that initializes the `error_code_` member variable, as well as a protected constructor that takes a `fmt::system_error` object and formats it using `fmt::format_system_error`.

The class provides a template parameter pack of `Args` which can be used to specify the additional arguments that should be passed to the constructor. The `system_error` class can then be instantiated with an `int` error code and a `string_view` message, or it can be represented as a member of a` std::tuple` of `fmt::system_error` objects.

The class includes a destructor that is called when the object is destroyed, which should release any resources that the object has acquired.


```
template <typename OutputIt, typename Char>
using arg_formatter FMT_DEPRECATED_ALIAS =
    detail::arg_formatter<OutputIt, Char>;

/**
 An error returned by an operating system or a language runtime,
 for example a file opening error.
*/
FMT_CLASS_API
class FMT_API system_error : public std::runtime_error {
 private:
  void init(int err_code, string_view format_str, format_args args);

 protected:
  int error_code_;

  system_error() : std::runtime_error(""), error_code_(0) {}

 public:
  /**
   \rst
   Constructs a :class:`fmt::system_error` object with a description
   formatted with `fmt::format_system_error`. *message* and additional
   arguments passed into the constructor are formatted similarly to
   `fmt::format`.

   **Example**::

     // This throws a system_error with the description
     //   cannot open file 'madeup': No such file or directory
     // or similar (system message may vary).
     const char *filename = "madeup";
     std::FILE *file = std::fopen(filename, "r");
     if (!file)
       throw fmt::system_error(errno, "cannot open file '{}'", filename);
   \endrst
  */
  template <typename... Args>
  system_error(int error_code, string_view message, const Args&... args)
      : std::runtime_error("") {
    init(error_code, message, make_format_args(args...));
  }
  system_error(const system_error&) = default;
  system_error& operator=(const system_error&) = default;
  system_error(system_error&&) = default;
  system_error& operator=(system_error&&) = default;
  ~system_error() FMT_NOEXCEPT FMT_OVERRIDE;

  int error_code() const { return error_code_; }
};

```cpp

这段代码定义了一个函数，用于将操作系统或编程语言运行时返回的错误信息格式化为特定的字符串，并将其输出到标准输出（通常是终端）。

具体来说，这个函数接受两个参数：一个字符串（通常是我们自己定义的错误消息）和一个表示错误代码的整数。函数内部首先检查 *error_code* 是否为有效的错误代码。如果是有效的错误代码，则函数内部将错误信息封装为一个包含两个字符串（即错误消息和国家/地区特定的错误消息）的字符串对象，并将其输出。

如果 *error_code* 不是有效的错误代码，函数内部将输出一个字符串，其将包含类似于 "Unknown error -1" 的系统错误消息。这个系统错误消息将根据操作系统和/或编程语言而有所不同。


```
/**
  \rst
  Formats an error returned by an operating system or a language runtime,
  for example a file opening error, and writes it to *out* in the following
  form:

  .. parsed-literal::
     *<message>*: *<system-message>*

  where *<message>* is the passed message and *<system-message>* is
  the system message corresponding to the error code.
  *error_code* is a system error code as given by ``errno``.
  If *error_code* is not a valid error code such as -1, the system message
  may look like "Unknown error -1" and is platform-dependent.
  \endrst
 */
```cpp

The `format_int` and `format_unsigned` functions are utility functions that try to manipulate integer values. These functions take an integer value as input and return a pointer to the corresponding output buffer content.

The `format_int` function is used to convert an integer value to a human-readable string. It takes an integer value and returns a pointer to the beginning of the output buffer. The output buffer is filled with the integer's digits, one character per digit, and the buffer size is adjusted accordingly to account for the null character at the end (`'\0'`).

The `format_unsigned` function is used to convert an integer value to a human-readable string. It takes an integer value and returns a pointer to the beginning of the output buffer. The output buffer is filled with the integer's digits, one character per digit, and the buffer size is adjusted accordingly to account for the null character at the end (`'\0'`).

The `format_int` and `format_unsigned` functions are intended to be used in combination with the `format_sequence` function, which constructs a string by repeatedly formatting the input sequence into the output buffer. For example, the following code formatts an integer value as a human-readable string:
```
int value = 123;
formatted_value = format_int(value);
```cpp
This code will result in the output buffer being filled with the string "123".


```
FMT_API void format_system_error(detail::buffer<char>& out, int error_code,
                                 string_view message) FMT_NOEXCEPT;

// Reports a system error without throwing an exception.
// Can be used to report errors from destructors.
FMT_API void report_system_error(int error_code,
                                 string_view message) FMT_NOEXCEPT;

/** Fast integer formatter. */
class format_int {
 private:
  // Buffer should be large enough to hold all digits (digits10 + 1),
  // a sign and a null character.
  enum { buffer_size = std::numeric_limits<unsigned long long>::digits10 + 3 };
  mutable char buffer_[buffer_size];
  char* str_;

  template <typename UInt> char* format_unsigned(UInt value) {
    auto n = static_cast<detail::uint32_or_64_or_128_t<UInt>>(value);
    return detail::format_decimal(buffer_, n, buffer_size - 1).begin;
  }

  template <typename Int> char* format_signed(Int value) {
    auto abs_value = static_cast<detail::uint32_or_64_or_128_t<Int>>(value);
    bool negative = value < 0;
    if (negative) abs_value = 0 - abs_value;
    auto begin = format_unsigned(abs_value);
    if (negative) *--begin = '-';
    return begin;
  }

 public:
  explicit format_int(int value) : str_(format_signed(value)) {}
  explicit format_int(long value) : str_(format_signed(value)) {}
  explicit format_int(long long value) : str_(format_signed(value)) {}
  explicit format_int(unsigned value) : str_(format_unsigned(value)) {}
  explicit format_int(unsigned long value) : str_(format_unsigned(value)) {}
  explicit format_int(unsigned long long value)
      : str_(format_unsigned(value)) {}

  /** Returns the number of characters written to the output buffer. */
  size_t size() const {
    return detail::to_unsigned(buffer_ - str_ + buffer_size - 1);
  }

  /**
    Returns a pointer to the output buffer content. No terminating null
    character is appended.
   */
  const char* data() const { return str_; }

  /**
    Returns a pointer to the output buffer content with terminating null
    character appended.
   */
  const char* c_str() const {
    buffer_[buffer_size - 1] = '\0';
    return str_;
  }

  /**
    \rst
    Returns the content of the output buffer as an ``std::string``.
    \endrst
   */
  std::string str() const { return std::string(str_, size()); }
};

```cpp

This is a C++ class template that defines a formatter object that can handle various data types, including but not limited to:

* char
* wchar_t
* char16
* char16_t
* char32
* char32_t
* char64
* char64_t
* wchar_t16
* wchar_t32
* wchar_t64
* wchar_t32_utf8
* wchar_t64_utf8
* std::string
* std::vector
* std::map
* std::unordered_map
* std::set
* std::unordered_set
* std::decltype
* std::regex

This class is derived from the `fmt` template, which provides a flexible and extensible interface for formatting input values of various types. The class adds support for custom data types and allows for the use of conditional checks for the specific data types.

The class template has a number of member functions, including `format` which takes a value of the specified data type and an `FormatContext` object, and returns the formatted output value. The `format` function uses the `visit_format_arg` function from the `fmt` template to format the argument passed to it. This function returns the formatted output value.

The class also has a number of helper functions, including `detail::parse_float_type_spec`, `detail::parse_custom_type_spec`, `detail::handle_cstring_type_spec`, and `detail::handle_dynamic_spec`, which are used in the `format` function.


```
// A formatter specialization for the core types corresponding to detail::type
// constants.
template <typename T, typename Char>
struct formatter<T, Char,
                 enable_if_t<detail::type_constant<T, Char>::value !=
                             detail::type::custom_type>> {
  FMT_CONSTEXPR formatter() = default;

  // Parses format specifiers stopping either at the end of the range or at the
  // terminating '}'.
  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    using handler_type = detail::dynamic_specs_handler<ParseContext>;
    auto type = detail::type_constant<T, Char>::value;
    detail::specs_checker<handler_type> handler(handler_type(specs_, ctx),
                                                type);
    auto it = parse_format_specs(ctx.begin(), ctx.end(), handler);
    auto eh = ctx.error_handler();
    switch (type) {
    case detail::type::none_type:
      FMT_ASSERT(false, "invalid argument type");
      break;
    case detail::type::int_type:
    case detail::type::uint_type:
    case detail::type::long_long_type:
    case detail::type::ulong_long_type:
    case detail::type::int128_type:
    case detail::type::uint128_type:
    case detail::type::bool_type:
      handle_int_type_spec(specs_.type,
                           detail::int_type_checker<decltype(eh)>(eh));
      break;
    case detail::type::char_type:
      handle_char_specs(
          &specs_, detail::char_specs_checker<decltype(eh)>(specs_.type, eh));
      break;
    case detail::type::float_type:
      if (detail::const_check(FMT_USE_FLOAT))
        detail::parse_float_type_spec(specs_, eh);
      else
        FMT_ASSERT(false, "float support disabled");
      break;
    case detail::type::double_type:
      if (detail::const_check(FMT_USE_DOUBLE))
        detail::parse_float_type_spec(specs_, eh);
      else
        FMT_ASSERT(false, "double support disabled");
      break;
    case detail::type::long_double_type:
      if (detail::const_check(FMT_USE_LONG_DOUBLE))
        detail::parse_float_type_spec(specs_, eh);
      else
        FMT_ASSERT(false, "long double support disabled");
      break;
    case detail::type::cstring_type:
      detail::handle_cstring_type_spec(
          specs_.type, detail::cstring_type_checker<decltype(eh)>(eh));
      break;
    case detail::type::string_type:
      detail::check_string_type_spec(specs_.type, eh);
      break;
    case detail::type::pointer_type:
      detail::check_pointer_type_spec(specs_.type, eh);
      break;
    case detail::type::custom_type:
      // Custom format specifiers should be checked in parse functions of
      // formatter specializations.
      break;
    }
    return it;
  }

  template <typename FormatContext>
  auto format(const T& val, FormatContext& ctx) -> decltype(ctx.out()) {
    detail::handle_dynamic_spec<detail::width_checker>(specs_.width,
                                                       specs_.width_ref, ctx);
    detail::handle_dynamic_spec<detail::precision_checker>(
        specs_.precision, specs_.precision_ref, ctx);
    using af = detail::arg_formatter<typename FormatContext::iterator,
                                     typename FormatContext::char_type>;
    return visit_format_arg(af(ctx, nullptr, &specs_),
                            detail::make_arg<FormatContext>(val));
  }

 private:
  detail::dynamic_format_specs<Char> specs_;
};

```cpp

这段代码定义了一个模板结构体 `formatter`，其中 `formatter` 被实例化了一次 `int` 和 `unsigned char` 两次。

模板元编程中的模板实参 `int` 和 `unsigned char` 被用作 `Template Metaprogramming` (SMP) 的编程语言特性，用途是提供了一种将 `int` 和 `unsigned char` 转换为 `void` 类型函数的能力。

`FMT_FORMAT_AS` 定义了一个模板函数，其中 `Type` 被定义为 `int` 或 `unsigned char`,`Base` 被定义为 `int` 或 `unsigned char`。这个模板函数的参数是 `FormatContext` 和 `Val`，其中 `FormatContext` 是格式化数据，`Val` 是 `Type` 的值。函数返回类型是 `decltype(ctx.out())`，其中 `ctx.out()` 是格式化上下文输出的 `void` 类型别名。

由于 `int` 和 `unsigned char` 都可以通过 `FMT_FORMAT_AS` 被实例化为 `void`，因此 `formatter` 结构体可以被用来在 `void` 上下文中输出 `int` 和 `unsigned char` 类型的值。


```
#define FMT_FORMAT_AS(Type, Base)                                             \
  template <typename Char>                                                    \
  struct formatter<Type, Char> : formatter<Base, Char> {                      \
    template <typename FormatContext>                                         \
    auto format(Type const& val, FormatContext& ctx) -> decltype(ctx.out()) { \
      return formatter<Base, Char>::format(val, ctx);                         \
    }                                                                         \
  }

FMT_FORMAT_AS(signed char, int);
FMT_FORMAT_AS(unsigned char, unsigned);
FMT_FORMAT_AS(short, int);
FMT_FORMAT_AS(unsigned short, unsigned);
FMT_FORMAT_AS(long, long long);
FMT_FORMAT_AS(unsigned long, unsigned long long);
```cpp

这段代码定义了几个模板结构体，用于格式化输入参数中的字符串。其中，formatter<void*, Char> 和 formatter<const void*, Char> 是用于输出整型参数的模板结构体，formatter<Char[N], Char> 和 formatter<basic_string_view<Char>, Char> 是用于输出字符串的模板结构体。

具体来说，formatter<void*, Char> 和 formatter<const void*, Char> 的作用是接收一个空字符串作为输入参数，并输出一个空字符串。formatter<Char[N], Char> 和 formatter<basic_string_view<Char>, Char> 的作用是接收一个字符串作为输入参数，并输出一个字符串。

在模板<void*, Char>中，函数format()用于格式化整型参数。如果模板的参数是void型，则函数不会产生任何输出。如果模板的参数是Char型，则函数会尝试从函数外部的空字符串、字符串常量或字符串指针等处获取字符，如果失败则会抛出一个异常。函数的返回类型是decltype(ctx.out())，表示将函数外部的类型转换为ctx.out()类型的返回类型。

在模板<typename Char, size_t N>中，函数format()用于格式化字符串参数。函数的参数是const Char*类型，表示从函数外部的字符串指针或字符串常量中获取字符串。函数的返回类型是decltype(ctx.out())，与模板<void*, Char>中的函数返回类型相同。


```
FMT_FORMAT_AS(Char*, const Char*);
FMT_FORMAT_AS(std::basic_string<Char>, basic_string_view<Char>);
FMT_FORMAT_AS(std::nullptr_t, const void*);
FMT_FORMAT_AS(detail::std_string_view<Char>, basic_string_view<Char>);

template <typename Char>
struct formatter<void*, Char> : formatter<const void*, Char> {
  template <typename FormatContext>
  auto format(void* val, FormatContext& ctx) -> decltype(ctx.out()) {
    return formatter<const void*, Char>::format(val, ctx);
  }
};

template <typename Char, size_t N>
struct formatter<Char[N], Char> : formatter<basic_string_view<Char>, Char> {
  template <typename FormatContext>
  auto format(const Char* val, FormatContext& ctx) -> decltype(ctx.out()) {
    return formatter<basic_string_view<Char>, Char>::format(val, ctx);
  }
};

```cpp

This is a Rust implementation of a simple `format` function that takes a `Value` object and a `FormatContext` object, and returns the result of calling the `format` method on the `Context` object, passing in the value and the format context.

The `format` method takes a `Value` object and a `FormatContext` object, and returns the result of calling the `format` method on the `Context` object, passing in the value and the format context.

The `format` method uses helper functions such as `handle_specs`, `handle_dynamic_spec`, and `format` to handle the dynamic specifiers that are present in the `Value` object.

The `handle_specs` function is used to handle the dynamic specifiers, such as `width` and `precision`.

The `handle_dynamic_spec` function is used to handle the dynamic specifiers such as `align` and `space`

The `format` function also includes a helper function `format_str_` which just gets the current format string from the `Context` object, and is used in the `handle_specs` function.

The `null_handler` is a default function that is called when the function is not able to match any dynamic specifiers.


```
// A formatter for types known only at run time such as variant alternatives.
//
// Usage:
//   using variant = std::variant<int, std::string>;
//   template <>
//   struct formatter<variant>: dynamic_formatter<> {
//     auto format(const variant& v, format_context& ctx) {
//       return visit([&](const auto& val) {
//           return dynamic_formatter<>::format(val, ctx);
//       }, v);
//     }
//   };
template <typename Char = char> class dynamic_formatter {
 private:
  struct null_handler : detail::error_handler {
    void on_align(align_t) {}
    void on_plus() {}
    void on_minus() {}
    void on_space() {}
    void on_hash() {}
  };

 public:
  template <typename ParseContext>
  auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    format_str_ = ctx.begin();
    // Checks are deferred to formatting time when the argument type is known.
    detail::dynamic_specs_handler<ParseContext> handler(specs_, ctx);
    return parse_format_specs(ctx.begin(), ctx.end(), handler);
  }

  template <typename T, typename FormatContext>
  auto format(const T& val, FormatContext& ctx) -> decltype(ctx.out()) {
    handle_specs(ctx);
    detail::specs_checker<null_handler> checker(
        null_handler(), detail::mapped_type_constant<T, FormatContext>::value);
    checker.on_align(specs_.align);
    switch (specs_.sign) {
    case sign::none:
      break;
    case sign::plus:
      checker.on_plus();
      break;
    case sign::minus:
      checker.on_minus();
      break;
    case sign::space:
      checker.on_space();
      break;
    }
    if (specs_.alt) checker.on_hash();
    if (specs_.precision >= 0) checker.end_precision();
    using af = detail::arg_formatter<typename FormatContext::iterator,
                                     typename FormatContext::char_type>;
    visit_format_arg(af(ctx, nullptr, &specs_),
                     detail::make_arg<FormatContext>(val));
    return ctx.out();
  }

 private:
  template <typename Context> void handle_specs(Context& ctx) {
    detail::handle_dynamic_spec<detail::width_checker>(specs_.width,
                                                       specs_.width_ref, ctx);
    detail::handle_dynamic_spec<detail::precision_checker>(
        specs_.precision, specs_.precision_ref, ctx);
  }

  detail::dynamic_format_specs<Char> specs_;
  const Char* format_str_;
};

```cpp

这段代码定义了两个模板别具的功能，第一个是`FMT_CONSTEXPR void advance_to`，第二个是`template <typename Char, typename ErrorHandler> FMT_CONSTEXPR void advance_to(...)`。第一个函数是一个模板函数，其中`template <typename Char, typename ErrorHandler>`指定了模板参数，而后面的`FMT_CONSTEXPR void advance_to`则是在此模板定义的基础上进行了保留。第二个函数 `vformat_to` 是另一个模板函数，其中`template <typename ArgFormatter, typename Char, typename Context>`指定了模板参数，而后面的 `typename ArgFormatter::iterator out, basic_string_view<Char> format_str, basic_format_args<Context> args, detail::locale_ref loc = detail::locale_ref()`则是在此模板定义的基础上进行了保留。


```
template <typename Char, typename ErrorHandler>
FMT_CONSTEXPR void advance_to(
    basic_format_parse_context<Char, ErrorHandler>& ctx, const Char* p) {
  ctx.advance_to(ctx.begin() + (p - &*ctx.begin()));
}

/** Formats arguments and writes the output to the range. */
template <typename ArgFormatter, typename Char, typename Context>
typename Context::iterator vformat_to(
    typename ArgFormatter::iterator out, basic_string_view<Char> format_str,
    basic_format_args<Context> args,
    detail::locale_ref loc = detail::locale_ref()) {
  if (format_str.size() == 2 && detail::equal2(format_str.data(), "{}")) {
    auto arg = args.get(0);
    if (!arg) detail::error_handler().on_error("argument not found");
    using iterator = typename ArgFormatter::iterator;
    return visit_format_arg(
        detail::default_arg_formatter<iterator, Char>{out, args, loc}, arg);
  }
  detail::format_handler<ArgFormatter, Char, Context> h(out, format_str, args,
                                                        loc);
  detail::parse_format_string<false>(format_str, h);
  return h.context.out();
}

```cpp

这段代码定义了三个模板类：`ptr<T>`，`ptr<std::unique_ptr<T>>` 和 `ptr<std::shared_ptr<T>>`。这些模板类提供了一个将 `T` 转换为 `const void*` 的函数指针。

`ptr<T>` 的模板实现是 `const void* ptr(const T& p)`，它将 `T` 的指针 `p` 传递给 `const void*` 类型的模板参数，然后将其返回。

`ptr<std::unique_ptr<T>>` 和 `ptr<std::shared_ptr<T>>` 的模板实现是相同的，只是对应的模板参数不同。它们都返回 `const void*` 类型的指针，但是 `ptr<std::shared_ptr<T>>` 的模板实现将 `T` 的指针 `p` 传递给 `const std::shared_ptr<T>&` 类型的模板参数，然后将其返回。

这些模板类的作用是将 `T` 的指针转换为 `const void*` 类型的指针，以便在格式化字符串中使用。例如，可以使用 `fmt::format()` 函数将 `T` 的指针 `p` 格式化为字符串，然后输出 `{}`。输出结果为 `const T*`。


```
/**
  \rst
  Converts ``p`` to ``const void*`` for pointer formatting.

  **Example**::

    auto s = fmt::format("{}", fmt::ptr(p));
  \endrst
 */
template <typename T> inline const void* ptr(const T* p) { return p; }
template <typename T> inline const void* ptr(const std::unique_ptr<T>& p) {
  return p.get();
}
template <typename T> inline const void* ptr(const std::shared_ptr<T>& p) {
  return p.get();
}

```cpp

这段代码定义了一个名为 "bytes" 的类，用于表示二进制数据。该类有一个私有成员变量 "data_"，一个友谊模板 "formatter"，和一个构造函数。

构造函数的作用是初始化 "data_"，该值是一个字符串视图，表示二进制数据的来源。

友谊模板 "formatter" 是一个用于格式化 "bytes" 类的工具函数。它包含两个成员函数：

- "parse" 函数，用于解析 "bytes" 类的对象。它接收一个 "ParseContext" 和一个字符串容器。返回解析结果，类型为 "decltype(ctx.begin())"。
- "format" 函数，用于格式化 "bytes" 类的对象。它接收一个 "bytes" 对象和一个 "FormatContext" 上下文。返回格式化后的结果，类型为 "decltype(ctx.out())"。

这两个函数的具体实现留给读者自行查看。


```
class bytes {
 private:
  string_view data_;
  friend struct formatter<bytes>;

 public:
  explicit bytes(string_view data) : data_(data) {}
};

template <> struct formatter<bytes> {
 private:
  detail::dynamic_format_specs<char> specs_;

 public:
  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    using handler_type = detail::dynamic_specs_handler<ParseContext>;
    detail::specs_checker<handler_type> handler(handler_type(specs_, ctx),
                                                detail::type::string_type);
    auto it = parse_format_specs(ctx.begin(), ctx.end(), handler);
    detail::check_string_type_spec(specs_.type, ctx.error_handler());
    return it;
  }

  template <typename FormatContext>
  auto format(bytes b, FormatContext& ctx) -> decltype(ctx.out()) {
    detail::handle_dynamic_spec<detail::width_checker>(specs_.width,
                                                       specs_.width_ref, ctx);
    detail::handle_dynamic_spec<detail::precision_checker>(
        specs_.precision, specs_.precision_ref, ctx);
    return detail::write_bytes(ctx.out(), b.data_, specs_);
  }
};

```cpp

这段代码定义了一个模板结构体 `arg_join`，该结构体有三个成员变量：`It` 表示输入迭代器，`Sentinel` 表示输入迭代器的终止条件，`basic_string_view<Char>` 表示一个字符串片段。

该 `arg_join` 模板结构体实现了一个 `std::pair<It, Sentinel, Char>` 的函数 pair() 类型的成员函数。pair() 函数接收三个整数参数，分别为输入迭代器 `It`，输入迭代器终止条件 `Sentinel` 和一个字符串片段 `Char`。函数内部通过移动指针和字符串片段的起始和终止指针来构建字符串，然后将字符串片段中的所有字符复制到输出字符串中，最后使用 `std::format` 函数将输出字符串格式化。

该 `formatter` 模板结构体实现了 `std::pair<It, Sentinel, Char>` 和 `Char` 类型的模板函数。它包含了一个 `std::format` 函数模板，该模板可以格式化 `arg_join<It, Sentinel, Char>` 类型的实参。函数模板的实现与 `std::format` 函数的实现相似，只是使用了 `arg_join` 模板结构体作为格式化对象。

该 `formatter` 模板结构体通过 `std::format` 函数的 `std::format` 函数实用来格式化 `arg_join<It, Sentinel, Char>` 类型的实参。函数的第一个模板参数是格式化目标 `std::format` 函数的输出类型，第二个模板参数是格式化上下文 `FormatContext&`。第三个模板参数是一个 `void` 类型的参数，表示输出字符串不会被返回。




```
template <typename It, typename Sentinel, typename Char>
struct arg_join : detail::view {
  It begin;
  Sentinel end;
  basic_string_view<Char> sep;

  arg_join(It b, Sentinel e, basic_string_view<Char> s)
      : begin(b), end(e), sep(s) {}
};

template <typename It, typename Sentinel, typename Char>
struct formatter<arg_join<It, Sentinel, Char>, Char>
    : formatter<typename std::iterator_traits<It>::value_type, Char> {
  template <typename FormatContext>
  auto format(const arg_join<It, Sentinel, Char>& value, FormatContext& ctx)
      -> decltype(ctx.out()) {
    using base = formatter<typename std::iterator_traits<It>::value_type, Char>;
    auto it = value.begin;
    auto out = ctx.out();
    if (it != value.end) {
      out = base::format(*it++, ctx);
      while (it != value.end) {
        out = std::copy(value.sep.begin(), value.sep.end(), out);
        ctx.advance_to(out);
        out = base::format(*it++, ctx);
      }
    }
    return out;
  }
};

```cpp

这两行代码是模板元编程中的模板实参列表。它们定义了两个版本的arg_join模板函数，一个是It类型的模板函数，另一个是wchar_t类型的模板函数。这两个版本的函数都接受三个模板参数：It、Sentinel和字符或宽字符串类型的格式化字符串。

函数的作用是组合给定的迭代器范围`[begin, end)`和格式化字符串`sep`，并返回一个object。其中，begin和end是指给定的迭代器范围的起始和终止点，sep是字符串中元素之间的分隔符。函数将返回一个包含三个元素的object，分别是begin、end和sep。

这两行代码的具体实现如下：
```csharp
/**
Returns an object that formats the iterator range [begin, end) with elements separated by 'sep'.
*/
template <typename It, typename Sentinel>
arg_join<It, Sentinel, char> join(It begin, Sentinel end, string_view sep) {
 return {begin, end, sep};
}

/**
Returns an object that formats the range with elements separated by 'sep'.
*/
template <typename It, typename Sentinel>
arg_join<It, Sentinel, wchar_t> join(It begin, Sentinel end, wstring_view sep) {
 return {begin, end, sep};
}
```cpp
这两行代码的模板定义了两个版本的arg_join函数，分别用于It和wchar_t类型的迭代器。它们的区别在于输入和输出参数的类型，以及分隔符的类型。

函数的实现主要集中在函数参数的检查和参数的默认值设置上。这两行代码的实现比较简单，主要负责将输入的迭代器和格式化字符串进行组合，并将结果返回。


```
/**
  Returns an object that formats the iterator range `[begin, end)` with elements
  separated by `sep`.
 */
template <typename It, typename Sentinel>
arg_join<It, Sentinel, char> join(It begin, Sentinel end, string_view sep) {
  return {begin, end, sep};
}

template <typename It, typename Sentinel>
arg_join<It, Sentinel, wchar_t> join(It begin, Sentinel end, wstring_view sep) {
  return {begin, end, sep};
}

/**
  \rst
  Returns an object that formats `range` with elements separated by `sep`.

  **Example**::

    std::vector<int> v = {1, 2, 3};
    fmt::print("{}", fmt::join(v, ", "));
    // Output: "1, 2, 3"

  ``fmt::join`` applies passed format specifiers to the range elements::

    fmt::print("{:02}", fmt::join(v, ", "));
    // Output: "01, 02, 03"
  \endrst
 */
```cpp

这段代码定义了两个版本的`arg_join`模板函数，用于将给定的`Range`类型的数据串按照指定的分隔符连接成一个字符串。

具体来说，这两个版本的函数都接受两个参数：一个`Range`类型的变量和一个小字符串`sep`。函数内部首先尝试使用第一个参数的`std::begin`和`std::end`函数来获取`Range`的起始和结束位置，如果起始和结束位置相同，那么函数将返回`std::string::max`。否则，函数将返回`std::string::min`。最后，函数返回连接好的字符串。

第一个版本的函数使用了`arg_join`命名，而第二个版本的函数则使用了`detail::iterator_t`和`detail::sentinel_t`容器，这些容器也支持`std::string`和`wchar_t`类型。不过，第二个版本的函数没有在`arg_join`中使用这些容器，而是直接将输入的`Range`作为参数传递给了`std::string::view`。


```
template <typename Range>
arg_join<detail::iterator_t<Range>, detail::sentinel_t<Range>, char> join(
    Range&& range, string_view sep) {
  return join(std::begin(range), std::end(range), sep);
}

template <typename Range>
arg_join<detail::iterator_t<Range>, detail::sentinel_t<Range>, wchar_t> join(
    Range&& range, wstring_view sep) {
  return join(std::begin(range), std::end(range), sep);
}

/**
  \rst
  Converts *value* to ``std::string`` using the default format for type *T*.

  **Example**::

    #include <fmt/format.h>

    std::string answer = fmt::to_string(42);
  \endrst
 */
```cpp

这段代码定义了两个版本的 `to_string` 函数，用于将一个 `T` 类型的值转换为字符串。

第一个版本的函数模板参数为 `T` 和 `FMT_ENABLE_IF(!std::is_integral<T>::value)`。函数实现中，首先使用 `std::string` 类型的变量 `result` 来存储要转换的值。然后，通过调用 `detail::write<char>(std::back_inserter(result), value)` 来将 `value` 转换为字符串，其中 `std::back_inserter` 是 `std::string` 类的 `back_inserter` 成员函数，用于将 `result` 中的位置向后插入 `value` 并返回插入后的位置。最后，将得到的字符串存储到 `result` 中，并返回。

第二个版本的函数模板参数为 `T`。函数实现中，使用与第一个版本相似的方法来计算要存储的最大字符数，并使用 `std::string` 类型的变量 `buffer` 来存储结果字符串。不同之处在于，第二个版本使用了 `std::is_integral<T>::value` 来判断是否可以将 `T` 转换为整数类型。如果是整数类型，则可以使用 `std::string` 类型的变量 `buffer` 来存储结果字符串，否则就需要使用第一个版本中的 `std::string` 类型。

总的来说，这两个版本的 `to_string` 函数都可以将 `T` 类型的值转换为字符串，但是第一个版本对整数类型有更好的支持。


```
template <typename T, FMT_ENABLE_IF(!std::is_integral<T>::value)>
inline std::string to_string(const T& value) {
  std::string result;
  detail::write<char>(std::back_inserter(result), value);
  return result;
}

template <typename T, FMT_ENABLE_IF(std::is_integral<T>::value)>
inline std::string to_string(T value) {
  // The buffer should be large enough to store the number including the sign or
  // "false" for bool.
  constexpr int max_size = detail::digits10<T>() + 2;
  char buffer[max_size > 5 ? static_cast<unsigned>(max_size) : 5];
  char* begin = buffer;
  return std::string(begin, detail::write<char>(begin, value));
}

```cpp

这两段代码定义了两个C++标准库头文件：`to_wstring` 和 `to_string`。它们都使用了模板来为不同的参数类型提供格式化输出。

`to_wstring` 的模板参数类型为 `const T&`，表示要输出字符串的值。它使用了格式化字符串 `{}` 和 `{}` 以及 `L` 预处理指令，其中 `{}` 是 `{}` 占位符，`{}` 是一个字符串模板参数，`L` 是一个辅助 long 类型的预处理指令，用于将 `{}` 占位符解析为 `std::wstring` 类型。因此，`to_wstring` 的作用是将给定的 `const T&` 类型的值转换为 `std::wstring` 并输出。

`to_string` 的模板参数类型为 `const basic_memory_buffer<Char, SIZE>&`，表示要输出字符串的值的数据缓冲区。它使用了 `basic_memory_buffer` 头文件，这个头文件定义了一个模板类 `basic_memory_buffer`，这个类提供了一个方便的方式来管理字符串缓冲区。`to_string` 的实现是将字符串编码为 `basic_memory_buffer<Char, SIZE>` 类型的字符串，然后将编码后的字符串返回。

这两个头文件都使用了模板来为不同的参数类型提供格式化输出。`to_wstring` 将转换后的字符串打印为 `std::wstring`，而 `to_string` 将给定的字符串缓冲区中的字符串打印为 `basic_memory_buffer<Char, SIZE>` 类型的字符串。


```
/**
  Converts *value* to ``std::wstring`` using the default format for type *T*.
 */
template <typename T> inline std::wstring to_wstring(const T& value) {
  return format(L"{}", value);
}

template <typename Char, size_t SIZE>
std::basic_string<Char> to_string(const basic_memory_buffer<Char, SIZE>& buf) {
  auto size = buf.size();
  detail::assume(size < std::basic_string<Char>().max_size());
  return std::basic_string<Char>(buf.data(), size);
}

template <typename Char>
```cpp

这段代码定义了一个名为“detail::vformat_to”的函数，它接受三个参数：

1. 一个名为“buf”的细节缓冲区（character 数组）引用，
2. 一个字符串格式字符串“format_str”，
3. 一个名为“args”的格式实参。

函数用法如下：

```
detail::vformat_to<af>(buffer_appender<Char> (buf), format_str, args);
```cpp

这里，函数内部首先定义了一个名为“af”的变量，它是一个格式化处理器（arg formatter）的迭代器类型，其参数类型为“typename buffer_context<char>::iterator”和“Char”。

接着，函数内部调用了“vformat_to”函数，并传入三个参数：

1. “af”作为第一个参数，
2. “buf”作为第二个参数，
3. “format_str”作为第三个参数，
4. “args”作为第四个参数。

“vformat_to”函数的作用是接收一个格式化字符串，和一个字符串格式实参，并返回一个格式化处理器迭代器，该迭代器将格式化字符串中的每一个字符和一个格式化实参依次输出，直到字符串结束。


```
detail::buffer_appender<Char> detail::vformat_to(
    detail::buffer<Char>& buf, basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  using af = arg_formatter<typename buffer_context<Char>::iterator, Char>;
  return vformat_to<af>(buffer_appender<Char>(buf), format_str, args);
}

#ifndef FMT_HEADER_ONLY
extern template format_context::iterator detail::vformat_to(
    detail::buffer<char>&, string_view, basic_format_args<format_context>);
namespace detail {
extern template FMT_API std::string grouping_impl<char>(locale_ref loc);
extern template FMT_API std::string grouping_impl<wchar_t>(locale_ref loc);
extern template FMT_API char thousands_sep_impl<char>(locale_ref loc);
extern template FMT_API wchar_t thousands_sep_impl<wchar_t>(locale_ref loc);
```cpp

这段代码定义了三个名为“decimal_point_impl”、“format_float”和“snprintf_float”的函数模板，属于FMT_API，其作用是在C或C++语言中实现 Decimal 格式化。

1. “decimal_point_impl”函数模板，输入参数为“locale_ref”和“int precision”，返回值为“char”，代表字符类型的输出。函数实现了一个字符类型的函数，将输入的“double”或“long double”值转换成“char”类型的字符，并在字符中添加了定位的“float”或“double”的数值。这个函数与“printf”函数的作用类似，但更加灵活，可以实现字符串中的精确的“float”或“double”值。

2. “format_float”函数模板，输入参数为“double value”、“int precision”和“float_specs”三个参数，内部参数为“float_specs”，返回值为“int”，代表整数类型的输出。函数实现了一个字符类型的函数，将输入的“double”值转换成整数类型，并在字符中添加了定位的“float”或“double”的数值。这个函数可以用来格式化“float”或“double”值，并输出到字符串中。它与“printf”函数的作用类似，但更加灵活，可以实现字符串中的精确的“float”或“double”值。

3. “snprintf_float”函数模板，输入参数为“float value”、“int precision”和“float_specs”三个参数，内部参数为“float_specs”，返回值为“int”，代表整数类型的输出。函数实现了一个字符类型的函数，将输入的“float”值转换成整数类型，并在字符中添加了定位的“float”或“double”的数值。这个函数可以用来格式化“float”值，并输出到字符串中。它与“printf”函数的作用类似，但更加灵活，可以实现字符串中的精确的“float”值。


```
extern template FMT_API char decimal_point_impl(locale_ref loc);
extern template FMT_API wchar_t decimal_point_impl(locale_ref loc);
extern template int format_float<double>(double value, int precision,
                                         float_specs specs, buffer<char>& buf);
extern template int format_float<long double>(long double value, int precision,
                                              float_specs specs,
                                              buffer<char>& buf);
int snprintf_float(float value, int precision, float_specs specs,
                   buffer<char>& buf) = delete;
extern template int snprintf_float<double>(double value, int precision,
                                           float_specs specs,
                                           buffer<char>& buf);
extern template int snprintf_float<long double>(long double value,
                                                int precision,
                                                float_specs specs,
                                                buffer<char>& buf);
}  // namespace detail
```cpp

这段代码定义了两种模板类型的迭代器函数，用于将字符串格式化为具体的字符类型。

第一个模板类型是 `template <typename S, typename Char = char_t<S>, FMT_ENABLE_IF(detail::is_string<S>::value)>`，用于将字符串模板类中的 `char_t` 类型参数转换为具体的 `char` 类型参数。这个模板类型要求输入参数 `S` 必须是一个字符类型，而 `Char` 类型被定义为字符类型。通过这个模板类型，我们可以使用 `FMT_ENABLE_IF` 技术，在编译时检查 `S` 是否符合字符类型要求。

第二个模板类型是 `template <typename S, typename... Args, size_t SIZE = inline_buffer_size, typename Char = enable_if_t<detail::is_string<S>::value, char_t<S>>`，用于将格式字符串 `const S& format_str` 和任意参数 `Args...` 打包成一个 `Args` 类型的参数，并将其转换为具体的字符类型。这个模板类型要求输入参数 `S` 必须是一个字符类型，而 `Char` 类型被定义为字符类型。通过这个模板类型，我们可以使用 `enable_if_t` 技术，在编译时检查 `S` 是否符合字符类型要求。

对于这两个模板类型，我们都使用了 `detail::vformat_to` 函数作为迭代器函数，它接收一个 `basic_memory_buffer<Char, SIZE>&` 的输入缓冲区 `buf`，和一个格式字符串 `const S& format_str`，以及任意参数 `Args...`。这个函数将 `format_str` 格式化为字符串，并将其转换为具体的字符类型。


```
#endif

template <typename S, typename Char = char_t<S>,
          FMT_ENABLE_IF(detail::is_string<S>::value)>
inline typename FMT_BUFFER_CONTEXT(Char)::iterator vformat_to(
    detail::buffer<Char>& buf, const S& format_str,
    basic_format_args<FMT_BUFFER_CONTEXT(type_identity_t<Char>)> args) {
  return detail::vformat_to(buf, to_string_view(format_str), args);
}

template <typename S, typename... Args, size_t SIZE = inline_buffer_size,
          typename Char = enable_if_t<detail::is_string<S>::value, char_t<S>>>
inline typename buffer_context<Char>::iterator format_to(
    basic_memory_buffer<Char, SIZE>& buf, const S& format_str, Args&&... args) {
  const auto& vargs = fmt::make_args_checked<Args...>(format_str, args...);
  return detail::vformat_to(buf, to_string_view(format_str), vargs);
}

```cpp

这段代码定义了一个模板结构体，名为“format_context_t”，其中包含了“OutputIt”和“Char”两个类型参数。这个模板结构体可以用来在代码中使用“basic_format_context”和“basic_format_args”类型的函数。

接下来的代码定义了一个模板结构体，名为“format_args_t”，其中包含了“format_context_t”类型的指针。这个模板结构体可以用来在代码中使用“basic_format_args”类型的函数。

接着，接下来的代码定义了一个模板别名，名为“format_to_n_context”，其中包含了“buffer_context”类型的指针。这个模板别名可以用来在代码中使用“basic_format_args”类型的函数，但是需要通过引入“ FMT_DEPRECATED_ALIAS”类型来修饰。

最后，接下来的代码定义了一个模板别名，名为“format_to_n_args”，其中包含了“basic_format_args”类型的指针。这个模板别名可以用来在代码中使用“basic_format_args”类型的函数，同样需要通过引入“ FMT_DEPRECATED_ALIAS”类型来修饰。


```
template <typename OutputIt, typename Char = char>
using format_context_t = basic_format_context<OutputIt, Char>;

template <typename OutputIt, typename Char = char>
using format_args_t = basic_format_args<format_context_t<OutputIt, Char>>;

template <typename OutputIt, typename Char = typename OutputIt::value_type>
using format_to_n_context FMT_DEPRECATED_ALIAS = buffer_context<Char>;

template <typename OutputIt, typename Char = typename OutputIt::value_type>
using format_to_n_args FMT_DEPRECATED_ALIAS =
    basic_format_args<buffer_context<Char>>;

template <typename OutputIt, typename Char, typename... Args>
FMT_DEPRECATED format_arg_store<buffer_context<Char>, Args...>
```cpp

这段代码定义了两个函数，分别是 `make_format_to_n_args` 和 `vprint`。它们的作用如下：

1. `make_format_to_n_args`：

这是一个接受 `Args...` 的函数，它返回了一个格式化字符串，其中 `args...` 是传递给 `make_format_to_n_args` 的参数。这个函数的作用是将 `Args...` 中的参数格式化并存储到一个 `basic_memory_buffer<Char>` 中，然后返回这个 `basic_memory_buffer<Char>`。

2. `vprint`：

这是一个接受 `std::FILE*` 和 `basic_string_view<Char>` 的函数，它接受一个 `basic_string_view<Char>` 和一些 `wformat_args`。这个函数的作用是将 `basic_string_view<Char>` 和 `wformat_args` 格式化并存储到一个 `basic_memory_buffer<Char>` 中，然后将这个 `basic_memory_buffer<Char>` 写入到 `std::FILE*` 中。如果写入失败，函数会抛出 `system_error` 异常。

`vprint` 函数中，`basic_string_view<Char>` 被用来存储格式化字符串，`wformat_args` 被用来格式化参数。函数首先通过调用 `detail::vformat_to` 函数将 `wformat_args` 格式化并存储到一个 `basic_memory_buffer<Char>` 中。然后，它调用 `to_string` 函数将 `basic_memory_buffer<Char>` 转换为字符串。最后，如果写入失败，函数会抛出 `system_error` 异常。


```
make_format_to_n_args(const Args&... args) {
  return format_arg_store<buffer_context<Char>, Args...>(args...);
}

template <typename Char, enable_if_t<(!std::is_same<Char, char>::value), int>>
std::basic_string<Char> detail::vformat(
    basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  basic_memory_buffer<Char> buffer;
  detail::vformat_to(buffer, format_str, args);
  return to_string(buffer);
}

template <typename Char, FMT_ENABLE_IF(std::is_same<Char, wchar_t>::value)>
void vprint(std::FILE* f, basic_string_view<Char> format_str,
            wformat_args args) {
  wmemory_buffer buffer;
  detail::vformat_to(buffer, format_str, args);
  buffer.push_back(L'\0');
  if (std::fputws(buffer.data(), f) == -1)
    FMT_THROW(system_error(errno, "cannot write to file"));
}

```cpp

这段代码定义了一个名为`vprint`的函数，用于将`basic_string_view`类型中的字符串格式化并输出到`stdout`。

首先，该函数使用`FMT_USE_IF`来判断是否支持使用用户定义的格式化选项。如果不支持，则使用默认的格式化选项。

如果支持用户定义的格式化选项，那么该函数将定义一个名为`udl_formatter`的类，该类实现了`std::basic_string<Char>`的模板。该模板重载了`operator()`函数，用于在`std::basic_string<Char>`中插入用户定义的格式化选项。

具体来说，该函数在`operator()`函数中，将`format`函数作为实参，将`args...`作为模板参数，并将`s`作为模板形参。然后，该函数使用`FMT_CONSTEXPR_DECLARE`来声明一个名为`s`的常量字符数组，该数组长度为`CHARS...`。最后，该函数使用`format`函数将`s`和`args...`作为模板实参，返回生成的字符串。

该函数的作用是用于将`basic_string_view`类型中的字符串按照用户定义的格式进行格式化并输出到`stdout`。


```
template <typename Char, FMT_ENABLE_IF(std::is_same<Char, wchar_t>::value)>
void vprint(basic_string_view<Char> format_str, wformat_args args) {
  vprint(stdout, format_str, args);
}

#if FMT_USE_USER_DEFINED_LITERALS
namespace detail {

#  if FMT_USE_UDL_TEMPLATE
template <typename Char, Char... CHARS> class udl_formatter {
 public:
  template <typename... Args>
  std::basic_string<Char> operator()(Args&&... args) const {
    static FMT_CONSTEXPR_DECL Char s[] = {CHARS..., '\0'};
    return format(FMT_STRING(s), std::forward<Args>(args)...);
  }
};
```cpp

这段代码定义了一个模板结构体udl_formatter，用于格式化输入字符串中的格式化字符串。udl_formatter结构体包含一个basic_string_view类型的成员str，一个模板元编程结构体类型的成员函数operator()和一个explicit constructor。

当模板元编程结构体函数operator()被调用时，udl_formatter结构体可以接受任意数量的参数。这些参数可以是符合udl_formatter中template类型的参数，也可以是任意数量的普通参数。在函数operator()中，udl_formatter结构体使用forward<Args>解引用这些参数，然后将它们与str连接起来，并使用format函数来格式化str，最后将格式化后的str作为返回值返回。

udl_arg是一个模板结构体，用于定义输入参数。它包含一个const类型的成员str，一个模板元编程结构体类型的成员函数named_arg，以及一个explicit constructor。named_arg使用operator=的重载来接受任意数量的模板参数，并返回一个新的named_arg对象。在模板元编程结构体函数named_arg operator=(T&& value)中，T是一个已知的模板类型参数，value是一个已知的模板类型参数，它们的值被复制到named_arg对象中。


```
#  else
template <typename Char> struct udl_formatter {
  basic_string_view<Char> str;

  template <typename... Args>
  std::basic_string<Char> operator()(Args&&... args) const {
    return format(str, std::forward<Args>(args)...);
  }
};
#  endif  // FMT_USE_UDL_TEMPLATE

template <typename Char> struct udl_arg {
  const Char* str;

  template <typename T> named_arg<Char, T> operator=(T&& value) const {
    return {str, std::forward<T>(value)};
  }
};
}  // namespace detail

```cpp

这段代码定义了一个名为 `operator ""_format` 的模板函数，用于在 `fmt::literals` 命名空间中实现类似于 `fmt::format` 的函数，但使用了用户定义的模板实参类型。

具体来说，该模板函数的实现如下：

```
template <typename Char, Char... CHARS>
FMT_CONSTEXPR detail::udl_formatter<Char, CHARS...> operator""_format() {
 return detail::udl_formatter<Char, CHARS...>{};
}
```cpp

其中，`detail::udl_formatter` 是 `fmt::literals` 命名空间中定义的一个抽象模板类，它接受一个 `Char` 类型的模板参数，以及多个 `Char` 类型的模板参数。通过这些模板参数，该抽象模板类可以实现 `fmt::format` 函数的行为，即输出类似于 `"The answer is <format>"` 这样的格式化字符串。

在 `operator ""_format` 函数的实现中，首先考虑了 `FMT_USE_UDL_TEMPLATE` 预处理指令。如果预处理指令的值为 `true`，则将 `GCC diagnostic push` 和 `GCC diagnostic ignored "-Wpedantic"` 预处理指令应用于当前源文件，这意味着在编译时会检查 `fmt::format` 函数的语法是否正确。如果预处理指令的值为 `false`，则不做任何预处理，继续编译。

如果 `FMT_CLANG_VERSION` 预处理指令的值为 `true`，则将 `GCC diagnostic ignored "-Wgnu-string-literal-operator-template"` 预处理指令应用于当前源文件，同样意味着在编译时会检查 `fmt::format` 函数的语法是否正确。

如果 `FMT_USE_UDL_TEMPLATE` 和 `FMT_CLANG_VERSION` 预处理指令都为 `false`，则不做任何预处理，编译器也不会检查 `fmt::format` 函数的语法是否正确。

如果 `FMT_USE_UDL_TEMPLATE` 预处理指令的值为 `true`，但是 `FMT_CLANG_VERSION` 预处理指令的值为 `false`，则不会编译 `operator ""_format` 函数，因为 `GCC diagnostic ignored "-W pedantic"` 预处理指令的值为 `true`，编译器会检查 `fmt::format` 函数的语法是否正确。

如果 `FMT_USE_UDL_TEMPLATE` 预处理指令的值为 `true`，并且 `FMT_CLANG_VERSION` 预处理指令的值为 `true`，则编译器不会检查 `operator ""_format` 函数的语法是否正确，这种情况下也不会输出任何错误。

如果 `FMT_USE_UDL_TEMPLATE` 和 `FMT_CLANG_VERSION` 预处理指令都为 `false`，则不做任何预处理，编译器也不会检查 `operator ""_format` 函数的语法是否正确。


```
inline namespace literals {
#  if FMT_USE_UDL_TEMPLATE
#    pragma GCC diagnostic push
#    pragma GCC diagnostic ignored "-Wpedantic"
#    if FMT_CLANG_VERSION
#      pragma GCC diagnostic ignored "-Wgnu-string-literal-operator-template"
#    endif
template <typename Char, Char... CHARS>
FMT_CONSTEXPR detail::udl_formatter<Char, CHARS...> operator""_format() {
  return {};
}
#    pragma GCC diagnostic pop
#  else
/**
  \rst
  User-defined literal equivalent of :func:`fmt::format`.

  **Example**::

    using namespace fmt::literals;
    std::string message = "The answer is {}"_format(42);
  \endrst
 */
```cpp

这段代码定义了两种用户自定义格式化器，一种用于将字符串"<br />"转换为格式化字符串，另一种用于将宽度为wchar_t的格式化字符串"<br />"转换为具体宽度为size_t的整数。

FMT_CONSTEXPR是一个头文件，它定义了这两种格式化器，用于在编译时检查可变参数的类型，以避免在运行时出现类型转换错误。

operator""是形式化字符串模式运算符，它会检查给定的字符串是否与格式化字符串的模板参数匹配，如果是，则返回模板实参的值并去除模板实参的空括号。

以下是这段代码的另一个输出，它定义了一个名为"fmt::arg"的函数，它会使用格式化字符串中的第二个可变参数来格式化字符串，并输出一个带有两个可变参数的字符串。

在示例中，该函数被用于打印一个包含两个整数的字符串，第一个整数是1.23。


```
FMT_CONSTEXPR detail::udl_formatter<char> operator"" _format(const char* s,
                                                             size_t n) {
  return {{s, n}};
}
FMT_CONSTEXPR detail::udl_formatter<wchar_t> operator"" _format(
    const wchar_t* s, size_t n) {
  return {{s, n}};
}
#  endif  // FMT_USE_UDL_TEMPLATE

/**
  \rst
  User-defined literal equivalent of :func:`fmt::arg`.

  **Example**::

    using namespace fmt::literals;
    fmt::print("Elapsed time: {s:.2f} seconds", "s"_a=1.23);
  \endrst
 */
```cpp

这段代码定义了两个命名元组类型的函数，分别为FMT_CONSTEXPR detail::udl_arg<char>和FMT_CONSTEXPR detail::udl_arg<wchar_t>。它们的作用是复制一个给定的字符串s，并将其存储在udl_arg<char>或udl_arg<wchar_t>类型的变量中。

udl_arg是一个命名元组类型，其中第一个元素是一个格式化字符串（FMT_CONSTEXPR），用于指定输入和输出的字符串。在这里，<char>和<wchar_t>被用作FMT_CONSTEXPR的格式字符，表示输入和输出数据类型。

两个函数的实现是相似的，都是一个返回新元组，其中第一个元素是一个字符串s，第二个元素是一个整数size_t，用于指定复制的长度。这个新元组类型返回的值是一个udl_arg<char>或udl_arg<wchar_t>类型的变量，用于在函数调用时存储原始输入字符串。


```
FMT_CONSTEXPR detail::udl_arg<char> operator"" _a(const char* s, size_t) {
  return {s};
}
FMT_CONSTEXPR detail::udl_arg<wchar_t> operator"" _a(const wchar_t* s, size_t) {
  return {s};
}
}  // namespace literals
#endif  // FMT_USE_USER_DEFINED_LITERALS
FMT_END_NAMESPACE

#ifdef FMT_HEADER_ONLY
#  define FMT_FUNC inline
#  include "format-inl.h"
#else
#  define FMT_FUNC
```cpp

这段代码是一个嵌套的 `#ifdef` 和 `#endif` 注释，用于定义一个名为 `FMT_FORMAT_H_` 的头文件。当包含此头文件的主函数第一次被访问时，头文件内容将被编译一次，而之后的 `#ifdef` 和 `#endif` 将不再被关注。

通常情况下，这段代码会在包含 `FMT_FORMAT_H_` 的源文件中出现，并在编译时将 `#define FMT_FORMAT_H_` 替换为 `FMT_FORMAT_H_`。这个头文件可能定义了一些常量和函数，用于在程序中输出FMT库的格式信息。


```
#endif

#endif  // FMT_FORMAT_H_

```