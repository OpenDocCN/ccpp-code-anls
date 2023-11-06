# xmrig源码解析 15

# `src/3rdparty/fmt/format-inl.h`

这段代码是一个C++格式化库，实现了多种格式化操作，如输出、输入、格式化字符串等。主要作用是定义了一系列函数，提供了对C++字符串和变量的各种格式化处理手段，使得开发者可以方便地按照特定的格式来输出或输入数据，提高代码的可读性和可读性。


```cpp
// Formatting library for C++ - implementation
//
// Copyright (c) 2012 - 2016, Victor Zverovich
// All rights reserved.
//
// For the license information refer to format.h.

#ifndef FMT_FORMAT_INL_H_
#define FMT_FORMAT_INL_H_

#include <cassert>
#include <cctype>
#include <climits>
#include <cmath>
#include <cstdarg>
```

这段代码是一个C++程序，它包含了一些头文件，以及一个函数：strerror_s，它被重载为std::memmove。我们需要分析它的作用。

首先，它引入了两个头文件：<cstring> 和 <cwchar>，这两个头文件可能与字符串操作有关。

接下来，它包含了一个双下划线前缀的函数：strerror_s，但这个函数被重载为std::memmove，这意味着它将使用一个比原始函数更安全的实现。

此外，它还包含了一个双下划线前缀的函数：strerror_statically_thousands_seperator，但这个函数并未在函数列表中出现。

最后，它表明了它包含的文件是FMT_STATIC_THOUSANDS_SEPARATOR。

综上所述，这段代码的作用是提供一个比原始函数更安全的strerror_s实现，可能用于在应用程序中处理更多字符。


```cpp
#include <cstring>  // std::memmove
#include <cwchar>
#include <exception>

#ifndef FMT_STATIC_THOUSANDS_SEPARATOR
#  include <locale>
#endif

#ifdef _WIN32
#  include <io.h>  // _isatty
#endif

#include "format.h"

// Dummy implementations of strerror_r and strerror_s called if corresponding
```

这段代码定义了两个名为`strerror_r`和`strerror_s`的函数，它们都是`fmt::detail::null<>`类型的 inline 函数。

它们的函数签名如下：

```cppc++
inline fmt::detail::null<> strerror_r(int, char*, ...)
   return {};

inline fmt::detail::null<> strerror_s(char*, size_t, ...)
   return {};
```

根据函数签名，这两个函数都接受3个参数：

- `int` 类型的整数 `strerror_r` 和 `char *` 类型的指针 `strerror_s`,
- `size_t` 类型的整数 `strerror_s` 和 `size_t` 类型的指针 `strerror_s`,
- 如果还有其他多个 `...` 参与传递，则不必在函数签名中列出。

这两个函数的作用是在`fmt::detail::null<>`类型的数据中，根据传入的参数进行输出，输出形式为：

```cppjava
message：文件名：行号：错误信息
```

比如，如果调用这两个函数，分别传递给它们 `file` 为 `"/path/to/directory"`、`line` 为 10、`message` 为 `"Assertion Error"` 参数，它们的作用就是：

```cppkotlin
file:10:Assertion Error: assertion failed
```

这里，`fmt::detail::null<>`类型代表的是一个空的对象，如果不小心传入了一个非空对象，代码会抛出异常。


```cpp
// system functions are not available.
inline fmt::detail::null<> strerror_r(int, char*, ...) { return {}; }
inline fmt::detail::null<> strerror_s(char*, size_t, ...) { return {}; }

FMT_BEGIN_NAMESPACE
namespace detail {

FMT_FUNC void assert_fail(const char* file, int line, const char* message) {
  // Use unchecked std::fprintf to avoid triggering another assertion when
  // writing to stderr fails
  std::fprintf(stderr, "%s:%d: assertion failed: %s", file, line, message);
  // Chosen instead of std::abort to satisfy Clang in CUDA mode during device
  // code pass.
  std::terminate();
}

```

这段代码定义了一个名为FMT_SNPRINTF的函数，用于在输入格式字符串中打印输出，其含义如下：

1. 当#define FMT_SNPRINTF snprintf预先定义时，该函数将直接使用预定义的函数，等同于：
```cppc
#include <stdio.h>
#include <strerror.h>

voidFMT_SNPRINTF(const char* format, ...)
{
   char buffer[100];
   int result = strerror_snprintf(buffer, sizeof(buffer), format, ##__VA_ARGS(format, ...));
   printf("%s\n", buffer);
}
```
2. 当#define FMT_SNPRINTF fmt_snprintf预先定义时，该函数将直接调用函数fmtd EFmt_SNPRINTF，等同于：
```cppc
#include <stdio.h>
#include <strerror.h>

voidFMT_SNPRINTF(const char* format, ...)
{
   int result = fmt_snprintf(buffer, sizeof(buffer), format, ##__VA_ARGS(format, ...));
   printf("%d\n", result);
}
```
3. 这段代码还定义了一个名为FMT_SNPRINTF的函数，用于根据输入格式字符串打印错误代码的描述，该函数的实现与上述第二种情况相同，只是输出参数从整数类型转换为字符串类型。


```cpp
#ifndef _MSC_VER
#  define FMT_SNPRINTF snprintf
#else  // _MSC_VER
inline int fmt_snprintf(char* buffer, size_t size, const char* format, ...) {
  va_list args;
  va_start(args, format);
  int result = vsnprintf_s(buffer, size, _TRUNCATE, format, args);
  va_end(args);
  return result;
}
#  define FMT_SNPRINTF fmt_snprintf
#endif  // _MSC_VER

// A portable thread-safe version of strerror.
// Sets buffer to point to a string describing the error code.
```

The `safe_strerror` function takes an integer `error_code` and a pointer to a character array `buffer`, and a buffer size `buffer_size`. It returns the result of calling the `strerror_r` function, with the buffer being filled in case the `buffer_size` is too small to hold the error message.

The `dispatcher` class has three private member variables: `error_code_`, `buffer_`, and `buffer_size_`, which are all of type `int`. The class also has a private member function `handle()`, which is a noop assignment operator to avoid warnings.

The `handle()` function takes a result of the `strerror_r` function and handles it according to the following rules:

* If the `buffer_size_` is at least 1, the function returns the result of the `strerror_r` function.
* If the `buffer_size_` is too small to hold the error message, the function returns ERANGE.
* If the `strerror_r` function is not available, the function falls back to strerror_s, which is a function that takes a `detail::null<>` and a `int` error code, and returns the result of calling strerror_s with the buffer being filled in case strerror_r is not available.
* If the `strerror_r` function is not available and the buffer is too small to hold the error message, the function falls back to handle() as a fallback.

The `fallback()` function is used to handle the case when strerror_r is not available and the buffer is too small to hold the error message. It behaves in the same way as the `handle()` function, but instead of returning a function result, it returns 0.


```cpp
// This can be either a pointer to a string stored in buffer,
// or a pointer to some static immutable string.
// Returns one of the following values:
//   0      - success
//   ERANGE - buffer is not large enough to store the error message
//   other  - failure
// Buffer should be at least of size 1.
inline int safe_strerror(int error_code, char*& buffer,
                         size_t buffer_size) FMT_NOEXCEPT {
  FMT_ASSERT(buffer != nullptr && buffer_size != 0, "invalid buffer");

  class dispatcher {
   private:
    int error_code_;
    char*& buffer_;
    size_t buffer_size_;

    // A noop assignment operator to avoid bogus warnings.
    void operator=(const dispatcher&) {}

    // Handle the result of XSI-compliant version of strerror_r.
    int handle(int result) {
      // glibc versions before 2.13 return result in errno.
      return result == -1 ? errno : result;
    }

    // Handle the result of GNU-specific version of strerror_r.
    FMT_MAYBE_UNUSED
    int handle(char* message) {
      // If the buffer is full then the message is probably truncated.
      if (message == buffer_ && strlen(buffer_) == buffer_size_ - 1)
        return ERANGE;
      buffer_ = message;
      return 0;
    }

    // Handle the case when strerror_r is not available.
    FMT_MAYBE_UNUSED
    int handle(detail::null<>) {
      return fallback(strerror_s(buffer_, buffer_size_, error_code_));
    }

    // Fallback to strerror_s when strerror_r is not available.
    FMT_MAYBE_UNUSED
    int fallback(int result) {
      // If the buffer is full then the message is probably truncated.
      return result == 0 && strlen(buffer_) == buffer_size_ - 1 ? ERANGE
                                                                : result;
    }

```

这段代码是一个 C++ 的函数，名为 `dispatcher`。它是一个 `std::dispatcher` 类的成员函数，用于处理错误信息。

首先，它检查是否支持 `strerror_r` 和 `strerror_s` 函数。如果不支持，它将使用 `strerror` 函数来获取错误信息。这个函数将返回一个整数 `errno`，以及一个字符串 `buffer_` 作为错误信息。

接着，它创建了一个名为 `fallback` 的函数，用于在 `strerror` 函数不可用时作为备用。这个函数将 `errno` 初始化为 0，将 `buffer_` 初始化为 `strerror_s` 函数返回的错误信息，并将 `buffer_size_` 初始化为 `strerror_s` 函数返回的字符串长度。

最后，它创建了一个名为 `dispatcher` 的类，其中包含一个名为 `run` 的成员函数。这个函数将调用 `handle` 函数，该函数将接收一个整数 `err_code` 和一个字符串 `buffer_`。它将使用 `strerror_r` 函数获取错误信息，并将 `buffer_` 和 `buffer_size_` 作为参数传递给该函数。

最终，该代码的作用是定义了一个名为 `dispatcher` 的类，该类包含一个名为 `run` 的成员函数，用于处理错误信息。该类成员函数将使用 `std::strerror` 函数获取错误信息，并将结果存储在 `error_code_` 和 `buffer_` 成员变量中。


```cpp
#if !FMT_MSC_VER
    // Fallback to strerror if strerror_r and strerror_s are not available.
    int fallback(detail::null<>) {
      errno = 0;
      buffer_ = strerror(error_code_);
      return errno;
    }
#endif

   public:
    dispatcher(int err_code, char*& buf, size_t buf_size)
        : error_code_(err_code), buffer_(buf), buffer_size_(buf_size) {}

    int run() { return handle(strerror_r(error_code_, buffer_, buffer_size_)); }
  };
  return dispatcher(error_code, buffer, buffer_size).run();
}

```

该函数 `format_error_code` 接受三个参数：

1. `out`：一个 `detail::buffer<char>` 类型的变量，用于存储输出字符串。
2. `error_code`：一个整数类型的变量，表示错误码。
3. `message`：一个 `string_view` 类型的参数，用于存储错误信息。

函数的作用是格式化输出错误信息，其中包括错误码和错误信息。具体实现如下：

1. 首先，函数创建一个空字符串 `out`，并尝试将其 `resize` 为 0，以确保在后续使用时不会动态内存分配。
2. 定义两个字符串常量 `SEP` 和 `ERROR_STR`，分别代表“:”和“error”。
3. 计算出 `ERROR_STR` 中的字符数与 `SEP` 中的字符数之差为 2，用于存储错误信息。
4. 如果错误码是负数，则将 `abs_value` 的值赋为 0，并将 `ERROR_STR` 中的字符数加 1。
5. 在函数的 `body` 部分，创建一个字符串 Appender 对象，用于将格式化的字符串存储到输出字符串中。
6. 如果 `message` 的长度小于等于 `inline_buffer_size` 减去 `ERROR_CODE_SIZE`，则将格式化的字符串填充到输出字符串中。
7. 最后，函数确认输出字符串是否已填满，如果没有，则可以安全地输出错误信息。


```cpp
FMT_FUNC void format_error_code(detail::buffer<char>& out, int error_code,
                                string_view message) FMT_NOEXCEPT {
  // Report error code making sure that the output fits into
  // inline_buffer_size to avoid dynamic memory allocation and potential
  // bad_alloc.
  out.try_resize(0);
  static const char SEP[] = ": ";
  static const char ERROR_STR[] = "error ";
  // Subtract 2 to account for terminating null characters in SEP and ERROR_STR.
  size_t error_code_size = sizeof(SEP) + sizeof(ERROR_STR) - 2;
  auto abs_value = static_cast<uint32_or_64_or_128_t<int>>(error_code);
  if (detail::is_negative(error_code)) {
    abs_value = 0 - abs_value;
    ++error_code_size;
  }
  error_code_size += detail::to_unsigned(detail::count_digits(abs_value));
  auto it = buffer_appender<char>(out);
  if (message.size() <= inline_buffer_size - error_code_size)
    format_to(it, "{}{}", message, SEP);
  format_to(it, "{}{}", ERROR_STR, error_code);
  assert(out.size() <= inline_buffer_size);
}

```

这段代码定义了一个名为report_error的函数，它接受三个参数：一个格式化函数func，一个整数error_code和一个字符串message。这个函数的作用是在调用func函数之后，将error_code和message转换为字符串并输出到标准错误流(通常是stderr)。

函数内部使用了一个内存缓冲区full_message，用来存储func函数的输出结果。首先，函数调用func函数并将full_message缓冲区中的内容传递给函数，将error_code和message作为参数传递给func函数。然后，函数使用(void)std::fwrite函数将full_message缓冲区中的内容写入到标准错误流中，并使用标准fputc函数将一个换行符'\n'写入到标准错误流中。

标准的fwrite函数在传递给report_error函数时，如果出错，将抛出系统错误并打印错误码。通过在report_error函数内部捕获这个错误，可以确保在函数内部正确处理错误。


```cpp
FMT_FUNC void report_error(format_func func, int error_code,
                           string_view message) FMT_NOEXCEPT {
  memory_buffer full_message;
  func(full_message, error_code, message);
  // Don't use fwrite_fully because the latter may throw.
  (void)std::fwrite(full_message.data(), full_message.size(), 1, stderr);
  std::fputc('\n', stderr);
}

// A wrapper around fwrite that throws on error.
inline void fwrite_fully(const void* ptr, size_t size, size_t count,
                         FILE* stream) {
  size_t written = std::fwrite(ptr, size, count, stream);
  if (written < count) FMT_THROW(system_error(errno, "cannot write to file"));
}
}  // namespace detail

```

这段代码是一个C++代码，定义了一个名为“detail”的命名空间。这个命名空间中定义了一个模板名为“locale_ref”的结构体，这个结构体有一个指向一个Locale类型的成员变量。

进一步地，这个模板结构体中还有一个名为“get”的函数，这个函数返回一个Locale类型的对象，但是需要通过一个静态的成员变量“FMT_STATIC_THOUSANDS_SEPARATOR”进行强制检查，如果不是该定义好的类型，那么会抛出一个异常。

对于模板“std::string”，这个模板名称的函数实现了一个名为“grouping_impl”的函数，它的输入参数是一个“locale_ref”对象，这个输入参数通过一个指向Locale类型的成员变量获取。

最后，通过使用“std::use_facet”函数，根据输入的Locale对象，返回了该Locale中所有字符所组成的标准的“std::string”类型的数据类型。


```cpp
#if !defined(FMT_STATIC_THOUSANDS_SEPARATOR)
namespace detail {

template <typename Locale>
locale_ref::locale_ref(const Locale& loc) : locale_(&loc) {
  static_assert(std::is_same<Locale, std::locale>::value, "");
}

template <typename Locale> Locale locale_ref::get() const {
  static_assert(std::is_same<Locale, std::locale>::value, "");
  return locale_ ? *static_cast<const std::locale*>(locale_) : std::locale();
}

template <typename Char> FMT_FUNC std::string grouping_impl(locale_ref loc) {
  return std::use_facet<std::numpunct<Char>>(loc.get<std::locale>()).grouping();
}
```

这段代码定义了两个模板函数，分别是`thousands_sep_impl`和`decimal_point_impl`，它们都接受一个`locale_ref`作为参数，并返回一个字符类型的常量。

这两个函数的作用是：

1. `thousands_sep_impl`：如果输入的`locale`是支持千分位分隔的，则返回该`locale`所支持千分位分隔的`std::numpunct<Char>`类型，否则返回`std::numpunct<Char>()`。

2. `decimal_point_impl`：如果输入的`locale`是支持小数点分隔的，则返回该`locale`所支持小数点分隔的`std::numpunct<Char>`类型，否则返回`std::numpunct<Char>()`。

这两个函数的实现主要依赖于`std::locale`和`std::numpunct<Char>`。`std::locale`是用来获取当前输入语言的，而`std::numpunct<Char>`则是`std::numeric`库中提供的一些辅助操作符的类型之一，可以用来对字符进行一些数学运算，如求千分位、保留小数点等。


```cpp
template <typename Char> FMT_FUNC Char thousands_sep_impl(locale_ref loc) {
  return std::use_facet<std::numpunct<Char>>(loc.get<std::locale>())
      .thousands_sep();
}
template <typename Char> FMT_FUNC Char decimal_point_impl(locale_ref loc) {
  return std::use_facet<std::numpunct<Char>>(loc.get<std::locale>())
      .decimal_point();
}
}  // namespace detail
#else
template <typename Char>
FMT_FUNC std::string detail::grouping_impl(locale_ref) {
  return "\03";
}
template <typename Char> FMT_FUNC Char detail::thousands_sep_impl(locale_ref) {
  return FMT_STATIC_THOUSANDS_SEPARATOR;
}
```

这段代码定义了一个名为 "decimal_point_impl" 的模板函数，它的参数类型为 Char 类型。函数实现了一个字符串的默认输出形式，即 "."。

接下来，代码定义了两个名为 "format_error" 和 "system_error" 的函数，它们分别是输出错误信息的 FTL 函数。这两个函数的实现都不带参数，并使用了 default 行为。

接着，代码定义了一个名为 "system_error" 的函数，它的实现是初始化一个系统错误对象，并将当前错误码、格式字符串和格式参数存储在内存缓冲区中。然后，使用 format_system_error 函数将错误信息格式化并存储到缓冲区中。接着，将格式化后的字符串作为第一个参数，调用 to_string 函数将字符串转换为 string 类型，并将转换后的结果存储到第一个成员变量中。最后，将第二个成员变量设置为错误码，将第三个成员变量设置为错误信息，并将第四个成员变量设置为当前错误对象，实现了输出错误信息的功能。


```cpp
template <typename Char> FMT_FUNC Char detail::decimal_point_impl(locale_ref) {
  return '.';
}
#endif

FMT_API FMT_FUNC format_error::~format_error() FMT_NOEXCEPT = default;
FMT_API FMT_FUNC system_error::~system_error() FMT_NOEXCEPT = default;

FMT_FUNC void system_error::init(int err_code, string_view format_str,
                                 format_args args) {
  error_code_ = err_code;
  memory_buffer buffer;
  format_system_error(buffer, err_code, vformat(format_str, args));
  std::runtime_error& base = *this;
  base = std::runtime_error(to_string(buffer));
}

```

It looks like the result you are trying to achieve is a matrix of coordinate values, where each coordinate value corresponds to a point in a 2D space. In this case, each row and column of the matrix represents a single point in the 2D space, with the values in the matrix indicating the distance between the point and the origin (0,0) along each axis.

For example, if the matrix represents the points in a simple grid, with the origin at the top left corner of the grid, the values in the matrix would be a sequence of two numbers, representing the distance between the point and the left edge of the grid. Similarly, the values in the matrix would be a sequence of two numbers, representing the distance between the point and the top bottom corner of the grid.



```cpp
namespace detail {

template <> FMT_FUNC int count_digits<4>(detail::fallback_uintptr n) {
  // fallback_uintptr is always stored in little endian.
  int i = static_cast<int>(sizeof(void*)) - 1;
  while (i > 0 && n.value[i] == 0) --i;
  auto char_digits = std::numeric_limits<unsigned char>::digits / 4;
  return i >= 0 ? i * char_digits + count_digits<4, unsigned>(n.value[i]) : 1;
}

template <typename T>
const typename basic_data<T>::digit_pair basic_data<T>::digits[] = {
    {'0', '0'}, {'0', '1'}, {'0', '2'}, {'0', '3'}, {'0', '4'}, {'0', '5'},
    {'0', '6'}, {'0', '7'}, {'0', '8'}, {'0', '9'}, {'1', '0'}, {'1', '1'},
    {'1', '2'}, {'1', '3'}, {'1', '4'}, {'1', '5'}, {'1', '6'}, {'1', '7'},
    {'1', '8'}, {'1', '9'}, {'2', '0'}, {'2', '1'}, {'2', '2'}, {'2', '3'},
    {'2', '4'}, {'2', '5'}, {'2', '6'}, {'2', '7'}, {'2', '8'}, {'2', '9'},
    {'3', '0'}, {'3', '1'}, {'3', '2'}, {'3', '3'}, {'3', '4'}, {'3', '5'},
    {'3', '6'}, {'3', '7'}, {'3', '8'}, {'3', '9'}, {'4', '0'}, {'4', '1'},
    {'4', '2'}, {'4', '3'}, {'4', '4'}, {'4', '5'}, {'4', '6'}, {'4', '7'},
    {'4', '8'}, {'4', '9'}, {'5', '0'}, {'5', '1'}, {'5', '2'}, {'5', '3'},
    {'5', '4'}, {'5', '5'}, {'5', '6'}, {'5', '7'}, {'5', '8'}, {'5', '9'},
    {'6', '0'}, {'6', '1'}, {'6', '2'}, {'6', '3'}, {'6', '4'}, {'6', '5'},
    {'6', '6'}, {'6', '7'}, {'6', '8'}, {'6', '9'}, {'7', '0'}, {'7', '1'},
    {'7', '2'}, {'7', '3'}, {'7', '4'}, {'7', '5'}, {'7', '6'}, {'7', '7'},
    {'7', '8'}, {'7', '9'}, {'8', '0'}, {'8', '1'}, {'8', '2'}, {'8', '3'},
    {'8', '4'}, {'8', '5'}, {'8', '6'}, {'8', '7'}, {'8', '8'}, {'8', '9'},
    {'9', '0'}, {'9', '1'}, {'9', '2'}, {'9', '3'}, {'9', '4'}, {'9', '5'},
    {'9', '6'}, {'9', '7'}, {'9', '8'}, {'9', '9'}};

```

这段代码定义了一个名为`basic_data`的模板结构体，其中`T`是模板参数类型，可以指定具体的数据类型。

该模板结构体定义了一个名为`hex_digits`的数组，其元素为`0`到`9`的ASCII字符，用于表示十进制转十六进制字符串中的数字。

接着，定义了一个名为`fmt_powers_of_10`的宏，用于计算指定数据类型的功率大小。该宏可以接受一个表达式`factor`，通过乘以10的指定次幂，计算出各种位数的功率大小。

然后，根据上面计算得到的功率大小，定义了两个模板类`basic_data_64`和`basic_data_32`，其中`basic_data_64`对应于`uint64_t`类型，`basic_data_32`对应于`uint32_t`类型。两个模板类中都有`powers_of_10_64`和`zero_or_powers_of_10_32`成员变量，分别对应于`FMT_POWERS_OF_10`计算得到的64位和32位功率大小。

最后，在`basic_data`模板类中，通过模板参数`T`来指定具体的数据类型，然后使用上面计算得到的功率大小，来定义该模板类的成员变量。


```cpp
template <typename T>
const char basic_data<T>::hex_digits[] = "0123456789abcdef";

#define FMT_POWERS_OF_10(factor)                                             \
  factor * 10, (factor)*100, (factor)*1000, (factor)*10000, (factor)*100000, \
      (factor)*1000000, (factor)*10000000, (factor)*100000000,               \
      (factor)*1000000000

template <typename T>
const uint64_t basic_data<T>::powers_of_10_64[] = {
    1, FMT_POWERS_OF_10(1), FMT_POWERS_OF_10(1000000000ULL),
    10000000000000000000ULL};

template <typename T>
const uint32_t basic_data<T>::zero_or_powers_of_10_32[] = {0, 0,
                                                           FMT_POWERS_OF_10(1)};

```



This appears to be a Base64 encoded string of�起点。


```cpp
template <typename T>
const uint64_t basic_data<T>::zero_or_powers_of_10_64[] = {
    0, 0, FMT_POWERS_OF_10(1), FMT_POWERS_OF_10(1000000000ULL),
    10000000000000000000ULL};

// Normalized 64-bit significands of pow(10, k), for k = -348, -340, ..., 340.
// These are generated by support/compute-powers.py.
template <typename T>
const uint64_t basic_data<T>::grisu_pow10_significands[] = {
    0xfa8fd5a0081c0288, 0xbaaee17fa23ebf76, 0x8b16fb203055ac76,
    0xcf42894a5dce35ea, 0x9a6bb0aa55653b2d, 0xe61acf033d1a45df,
    0xab70fe17c79ac6ca, 0xff77b1fcbebcdc4f, 0xbe5691ef416bd60c,
    0x8dd01fad907ffc3c, 0xd3515c2831559a83, 0x9d71ac8fada6c9b5,
    0xea9c227723ee8bcb, 0xaecc49914078536d, 0x823c12795db6ce57,
    0xc21094364dfb5637, 0x9096ea6f3848984f, 0xd77485cb25823ac7,
    0xa086cfcd97bf97f4, 0xef340a98172aace5, 0xb23867fb2a35b28e,
    0x84c8d4dfd2c63f3b, 0xc5dd44271ad3cdba, 0x936b9fcebb25c996,
    0xdbac6c247d62a584, 0xa3ab66580d5fdaf6, 0xf3e2f893dec3f126,
    0xb5b5ada8aaff80b8, 0x87625f056c7c4a8b, 0xc9bcff6034c13053,
    0x964e858c91ba2655, 0xdff9772470297ebd, 0xa6dfbd9fb8e5b88f,
    0xf8a95fcf88747d94, 0xb94470938fa89bcf, 0x8a08f0f8bf0f156b,
    0xcdb02555653131b6, 0x993fe2c6d07b7fac, 0xe45c10c42a2b3b06,
    0xaa242499697392d3, 0xfd87b5f28300ca0e, 0xbce5086492111aeb,
    0x8cbccc096f5088cc, 0xd1b71758e219652c, 0x9c40000000000000,
    0xe8d4a51000000000, 0xad78ebc5ac620000, 0x813f3978f8940984,
    0xc097ce7bc90715b3, 0x8f7e32ce7bea5c70, 0xd5d238a4abe98068,
    0x9f4f2726179a2245, 0xed63a231d4c4fb27, 0xb0de65388cc8ada8,
    0x83c7088e1aab65db, 0xc45d1df942711d9a, 0x924d692ca61be758,
    0xda01ee641a708dea, 0xa26da3999aef774a, 0xf209787bb47d6b85,
    0xb454e4a179dd1877, 0x865b86925b9bc5c2, 0xc83553c5c8965d3d,
    0x952ab45cfa97a0b3, 0xde469fbd99a05fe3, 0xa59bc234db398c25,
    0xf6c69a72a3989f5c, 0xb7dcbf5354e9bece, 0x88fcf317f22241e2,
    0xcc20ce9bd35c78a5, 0x98165af37b2153df, 0xe2a0b5dc971f303a,
    0xa8d9d1535ce3b396, 0xfb9b7cd9a4a7443c, 0xbb764c4ca7a44410,
    0x8bab8eefb6409c1a, 0xd01fef10a657842c, 0x9b10a4e5e9913129,
    0xe7109bfba19c0c9d, 0xac2820d9623bf429, 0x80444b5e7aa7cf85,
    0xbf21e44003acdd2d, 0x8e679c2f5e44ff8f, 0xd433179d9c8cb841,
    0x9e19db92b4e31ba9, 0xeb96bf6ebadf77d9, 0xaf87023b9bf0ee6b,
};

```

This is a header file called "divtest_table_for_pow5_32.h" that defines the template data type "divtest\_table\_entry<uint32\_t>". The header file is then included in the source file "pow5\_32\_template\_integration.cpp".

The "divtest\_table\_for\_pow5\_32" template data type is used for testing integerdivisions in an algorithm that separates integer test results into table-driven memory based on the desired test coverage.

The "pow5\_32\_template\_integration.cpp" source file implements the algorithm.


```cpp
// Binary exponents of pow(10, k), for k = -348, -340, ..., 340, corresponding
// to significands above.
template <typename T>
const int16_t basic_data<T>::grisu_pow10_exponents[] = {
    -1220, -1193, -1166, -1140, -1113, -1087, -1060, -1034, -1007, -980, -954,
    -927,  -901,  -874,  -847,  -821,  -794,  -768,  -741,  -715,  -688, -661,
    -635,  -608,  -582,  -555,  -529,  -502,  -475,  -449,  -422,  -396, -369,
    -343,  -316,  -289,  -263,  -236,  -210,  -183,  -157,  -130,  -103, -77,
    -50,   -24,   3,     30,    56,    83,    109,   136,   162,   189,  216,
    242,   269,   295,   322,   348,   375,   402,   428,   455,   481,  508,
    534,   561,   588,   614,   641,   667,   694,   720,   747,   774,  800,
    827,   853,   880,   907,   933,   960,   986,   1013,  1039,  1066};

template <typename T>
const divtest_table_entry<uint32_t> basic_data<T>::divtest_table_for_pow5_32[] =
    {{0x00000001, 0xffffffff}, {0xcccccccd, 0x33333333},
     {0xc28f5c29, 0x0a3d70a3}, {0x26e978d5, 0x020c49ba},
     {0x3afb7e91, 0x0068db8b}, {0x0bcbe61d, 0x0014f8b5},
     {0x68c26139, 0x000431bd}, {0xae8d46a5, 0x0000d6bf},
     {0x22e90e21, 0x00002af3}, {0x3a2e9c6d, 0x00000897},
     {0x3ed61f49, 0x000001b7}};

```

It looks like the output data is a series of integers, but it's difficult to tell for sure without more information.

The first number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is原文本的前缀。

The second number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the first number in the output data, but with a different value.

The third number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The fourth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The fifth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The sixth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The seventh number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The eighth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The ninth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The tenth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The eleventh number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The twelfth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The thirteenth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The fourteenth number in the output data is `0x000000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The fifteenth number in the output data is `0x000000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with a different value.

The sixteenth number in the output data is `0x00000000000000001`, which is a 64-bit unsigned integer. It's likely that this number is the same as the previous number in the output data, but with



```cpp
template <typename T>
const divtest_table_entry<uint64_t> basic_data<T>::divtest_table_for_pow5_64[] =
    {{0x0000000000000001, 0xffffffffffffffff},
     {0xcccccccccccccccd, 0x3333333333333333},
     {0x8f5c28f5c28f5c29, 0x0a3d70a3d70a3d70},
     {0x1cac083126e978d5, 0x020c49ba5e353f7c},
     {0xd288ce703afb7e91, 0x0068db8bac710cb2},
     {0x5d4e8fb00bcbe61d, 0x0014f8b588e368f0},
     {0x790fb65668c26139, 0x000431bde82d7b63},
     {0xe5032477ae8d46a5, 0x0000d6bf94d5e57a},
     {0xc767074b22e90e21, 0x00002af31dc46118},
     {0x8e47ce423a2e9c6d, 0x0000089705f4136b},
     {0x4fa7f60d3ed61f49, 0x000001b7cdfd9d7b},
     {0x0fee64690c913975, 0x00000057f5ff85e5},
     {0x3662e0e1cf503eb1, 0x000000119799812d},
     {0xa47a2cf9f6433fbd, 0x0000000384b84d09},
     {0x54186f653140a659, 0x00000000b424dc35},
     {0x7738164770402145, 0x0000000024075f3d},
     {0xe4a4d1417cd9a041, 0x000000000734aca5},
     {0xc75429d9e5c5200d, 0x000000000170ef54},
     {0xc1773b91fac10669, 0x000000000049c977},
     {0x26b172506559ce15, 0x00000000000ec1e4},
     {0xd489e3a9addec2d1, 0x000000000002f394},
     {0x90e860bb892c8d5d, 0x000000000000971d},
     {0x502e79bf1b6f4f79, 0x0000000000001e39},
     {0xdcd618596be30fe5, 0x000000000000060b}};

```

0xad78ebc5ac620000, 0xd8d726b7177a8000, 0x878678326eac9000,
   0xa968163f0a57b400, 0xd3c21bcecceda100, 0x84595161401484a0,
   0xa56fa5b99019a5c8, 0xcecb8f27f4200f3a, 0x813f3978f8940984,
   0xa18f07d736b90be5, 0xc9f2c9cd04674ede, 0xfc6f7c4045812296,
   0x9dc5ada82b70b59d, 0xc5371912364ce305, 0xf684df56c3e01bc6,
   0x9a130b963a6c115c, 0xc097ce7bc90715b3, 0xf0bdc21abb48db20,
   0x96769950b50d88f4, 0xbc143fa4e250eb31, 0xeb194f8e1ae525fd,
   0x92efd1b8d0cf37be, 0xb7abc627050305ad, 0xe596b7b0c643c719,
   0x8f7e32ce7bea5c6f, 0xb35dbf821ae4f38b, 0xe0352f62a19e306e};



```cpp
template <typename T>
const uint64_t basic_data<T>::dragonbox_pow10_significands_64[] = {
    0x81ceb32c4b43fcf5, 0xa2425ff75e14fc32, 0xcad2f7f5359a3b3f,
    0xfd87b5f28300ca0e, 0x9e74d1b791e07e49, 0xc612062576589ddb,
    0xf79687aed3eec552, 0x9abe14cd44753b53, 0xc16d9a0095928a28,
    0xf1c90080baf72cb2, 0x971da05074da7bef, 0xbce5086492111aeb,
    0xec1e4a7db69561a6, 0x9392ee8e921d5d08, 0xb877aa3236a4b44a,
    0xe69594bec44de15c, 0x901d7cf73ab0acda, 0xb424dc35095cd810,
    0xe12e13424bb40e14, 0x8cbccc096f5088cc, 0xafebff0bcb24aaff,
    0xdbe6fecebdedd5bf, 0x89705f4136b4a598, 0xabcc77118461cefd,
    0xd6bf94d5e57a42bd, 0x8637bd05af6c69b6, 0xa7c5ac471b478424,
    0xd1b71758e219652c, 0x83126e978d4fdf3c, 0xa3d70a3d70a3d70b,
    0xcccccccccccccccd, 0x8000000000000000, 0xa000000000000000,
    0xc800000000000000, 0xfa00000000000000, 0x9c40000000000000,
    0xc350000000000000, 0xf424000000000000, 0x9896800000000000,
    0xbebc200000000000, 0xee6b280000000000, 0x9502f90000000000,
    0xba43b74000000000, 0xe8d4a51000000000, 0x9184e72a00000000,
    0xb5e620f480000000, 0xe35fa931a0000000, 0x8e1bc9bf04000000,
    0xb1a2bc2ec5000000, 0xde0b6b3a76400000, 0x8ac7230489e80000,
    0xad78ebc5ac620000, 0xd8d726b7177a8000, 0x878678326eac9000,
    0xa968163f0a57b400, 0xd3c21bcecceda100, 0x84595161401484a0,
    0xa56fa5b99019a5c8, 0xcecb8f27f4200f3a, 0x813f3978f8940984,
    0xa18f07d736b90be5, 0xc9f2c9cd04674ede, 0xfc6f7c4045812296,
    0x9dc5ada82b70b59d, 0xc5371912364ce305, 0xf684df56c3e01bc6,
    0x9a130b963a6c115c, 0xc097ce7bc90715b3, 0xf0bdc21abb48db20,
    0x96769950b50d88f4, 0xbc143fa4e250eb31, 0xeb194f8e1ae525fd,
    0x92efd1b8d0cf37be, 0xb7abc627050305ad, 0xe596b7b0c643c719,
    0x8f7e32ce7bea5c6f, 0xb35dbf821ae4f38b, 0xe0352f62a19e306e};

```

JPEG 2000 is a popular image format that uses a combination of techniques, including JPEG compression, that allows for a wide range of image quality and color representation. It is a widely used format for storing and transmitting images, and is supported by most imaging devices and software. The format is named after the manufacturer, JPEG Technology, and was first released in 1992.


```cpp
template <typename T>
const uint128_wrapper basic_data<T>::dragonbox_pow10_significands_128[] = {
#if FMT_USE_FULL_CACHE_DRAGONBOX
    {0xff77b1fcbebcdc4f, 0x25e8e89c13bb0f7b},
    {0x9faacf3df73609b1, 0x77b191618c54e9ad},
    {0xc795830d75038c1d, 0xd59df5b9ef6a2418},
    {0xf97ae3d0d2446f25, 0x4b0573286b44ad1e},
    {0x9becce62836ac577, 0x4ee367f9430aec33},
    {0xc2e801fb244576d5, 0x229c41f793cda740},
    {0xf3a20279ed56d48a, 0x6b43527578c11110},
    {0x9845418c345644d6, 0x830a13896b78aaaa},
    {0xbe5691ef416bd60c, 0x23cc986bc656d554},
    {0xedec366b11c6cb8f, 0x2cbfbe86b7ec8aa9},
    {0x94b3a202eb1c3f39, 0x7bf7d71432f3d6aa},
    {0xb9e08a83a5e34f07, 0xdaf5ccd93fb0cc54},
    {0xe858ad248f5c22c9, 0xd1b3400f8f9cff69},
    {0x91376c36d99995be, 0x23100809b9c21fa2},
    {0xb58547448ffffb2d, 0xabd40a0c2832a78b},
    {0xe2e69915b3fff9f9, 0x16c90c8f323f516d},
    {0x8dd01fad907ffc3b, 0xae3da7d97f6792e4},
    {0xb1442798f49ffb4a, 0x99cd11cfdf41779d},
    {0xdd95317f31c7fa1d, 0x40405643d711d584},
    {0x8a7d3eef7f1cfc52, 0x482835ea666b2573},
    {0xad1c8eab5ee43b66, 0xda3243650005eed0},
    {0xd863b256369d4a40, 0x90bed43e40076a83},
    {0x873e4f75e2224e68, 0x5a7744a6e804a292},
    {0xa90de3535aaae202, 0x711515d0a205cb37},
    {0xd3515c2831559a83, 0x0d5a5b44ca873e04},
    {0x8412d9991ed58091, 0xe858790afe9486c3},
    {0xa5178fff668ae0b6, 0x626e974dbe39a873},
    {0xce5d73ff402d98e3, 0xfb0a3d212dc81290},
    {0x80fa687f881c7f8e, 0x7ce66634bc9d0b9a},
    {0xa139029f6a239f72, 0x1c1fffc1ebc44e81},
    {0xc987434744ac874e, 0xa327ffb266b56221},
    {0xfbe9141915d7a922, 0x4bf1ff9f0062baa9},
    {0x9d71ac8fada6c9b5, 0x6f773fc3603db4aa},
    {0xc4ce17b399107c22, 0xcb550fb4384d21d4},
    {0xf6019da07f549b2b, 0x7e2a53a146606a49},
    {0x99c102844f94e0fb, 0x2eda7444cbfc426e},
    {0xc0314325637a1939, 0xfa911155fefb5309},
    {0xf03d93eebc589f88, 0x793555ab7eba27cb},
    {0x96267c7535b763b5, 0x4bc1558b2f3458df},
    {0xbbb01b9283253ca2, 0x9eb1aaedfb016f17},
    {0xea9c227723ee8bcb, 0x465e15a979c1cadd},
    {0x92a1958a7675175f, 0x0bfacd89ec191eca},
    {0xb749faed14125d36, 0xcef980ec671f667c},
    {0xe51c79a85916f484, 0x82b7e12780e7401b},
    {0x8f31cc0937ae58d2, 0xd1b2ecb8b0908811},
    {0xb2fe3f0b8599ef07, 0x861fa7e6dcb4aa16},
    {0xdfbdcece67006ac9, 0x67a791e093e1d49b},
    {0x8bd6a141006042bd, 0xe0c8bb2c5c6d24e1},
    {0xaecc49914078536d, 0x58fae9f773886e19},
    {0xda7f5bf590966848, 0xaf39a475506a899f},
    {0x888f99797a5e012d, 0x6d8406c952429604},
    {0xaab37fd7d8f58178, 0xc8e5087ba6d33b84},
    {0xd5605fcdcf32e1d6, 0xfb1e4a9a90880a65},
    {0x855c3be0a17fcd26, 0x5cf2eea09a550680},
    {0xa6b34ad8c9dfc06f, 0xf42faa48c0ea481f},
    {0xd0601d8efc57b08b, 0xf13b94daf124da27},
    {0x823c12795db6ce57, 0x76c53d08d6b70859},
    {0xa2cb1717b52481ed, 0x54768c4b0c64ca6f},
    {0xcb7ddcdda26da268, 0xa9942f5dcf7dfd0a},
    {0xfe5d54150b090b02, 0xd3f93b35435d7c4d},
    {0x9efa548d26e5a6e1, 0xc47bc5014a1a6db0},
    {0xc6b8e9b0709f109a, 0x359ab6419ca1091c},
    {0xf867241c8cc6d4c0, 0xc30163d203c94b63},
    {0x9b407691d7fc44f8, 0x79e0de63425dcf1e},
    {0xc21094364dfb5636, 0x985915fc12f542e5},
    {0xf294b943e17a2bc4, 0x3e6f5b7b17b2939e},
    {0x979cf3ca6cec5b5a, 0xa705992ceecf9c43},
    {0xbd8430bd08277231, 0x50c6ff782a838354},
    {0xece53cec4a314ebd, 0xa4f8bf5635246429},
    {0x940f4613ae5ed136, 0x871b7795e136be9a},
    {0xb913179899f68584, 0x28e2557b59846e40},
    {0xe757dd7ec07426e5, 0x331aeada2fe589d0},
    {0x9096ea6f3848984f, 0x3ff0d2c85def7622},
    {0xb4bca50b065abe63, 0x0fed077a756b53aa},
    {0xe1ebce4dc7f16dfb, 0xd3e8495912c62895},
    {0x8d3360f09cf6e4bd, 0x64712dd7abbbd95d},
    {0xb080392cc4349dec, 0xbd8d794d96aacfb4},
    {0xdca04777f541c567, 0xecf0d7a0fc5583a1},
    {0x89e42caaf9491b60, 0xf41686c49db57245},
    {0xac5d37d5b79b6239, 0x311c2875c522ced6},
    {0xd77485cb25823ac7, 0x7d633293366b828c},
    {0x86a8d39ef77164bc, 0xae5dff9c02033198},
    {0xa8530886b54dbdeb, 0xd9f57f830283fdfd},
    {0xd267caa862a12d66, 0xd072df63c324fd7c},
    {0x8380dea93da4bc60, 0x4247cb9e59f71e6e},
    {0xa46116538d0deb78, 0x52d9be85f074e609},
    {0xcd795be870516656, 0x67902e276c921f8c},
    {0x806bd9714632dff6, 0x00ba1cd8a3db53b7},
    {0xa086cfcd97bf97f3, 0x80e8a40eccd228a5},
    {0xc8a883c0fdaf7df0, 0x6122cd128006b2ce},
    {0xfad2a4b13d1b5d6c, 0x796b805720085f82},
    {0x9cc3a6eec6311a63, 0xcbe3303674053bb1},
    {0xc3f490aa77bd60fc, 0xbedbfc4411068a9d},
    {0xf4f1b4d515acb93b, 0xee92fb5515482d45},
    {0x991711052d8bf3c5, 0x751bdd152d4d1c4b},
    {0xbf5cd54678eef0b6, 0xd262d45a78a0635e},
    {0xef340a98172aace4, 0x86fb897116c87c35},
    {0x9580869f0e7aac0e, 0xd45d35e6ae3d4da1},
    {0xbae0a846d2195712, 0x8974836059cca10a},
    {0xe998d258869facd7, 0x2bd1a438703fc94c},
    {0x91ff83775423cc06, 0x7b6306a34627ddd0},
    {0xb67f6455292cbf08, 0x1a3bc84c17b1d543},
    {0xe41f3d6a7377eeca, 0x20caba5f1d9e4a94},
    {0x8e938662882af53e, 0x547eb47b7282ee9d},
    {0xb23867fb2a35b28d, 0xe99e619a4f23aa44},
    {0xdec681f9f4c31f31, 0x6405fa00e2ec94d5},
    {0x8b3c113c38f9f37e, 0xde83bc408dd3dd05},
    {0xae0b158b4738705e, 0x9624ab50b148d446},
    {0xd98ddaee19068c76, 0x3badd624dd9b0958},
    {0x87f8a8d4cfa417c9, 0xe54ca5d70a80e5d7},
    {0xa9f6d30a038d1dbc, 0x5e9fcf4ccd211f4d},
    {0xd47487cc8470652b, 0x7647c32000696720},
    {0x84c8d4dfd2c63f3b, 0x29ecd9f40041e074},
    {0xa5fb0a17c777cf09, 0xf468107100525891},
    {0xcf79cc9db955c2cc, 0x7182148d4066eeb5},
    {0x81ac1fe293d599bf, 0xc6f14cd848405531},
    {0xa21727db38cb002f, 0xb8ada00e5a506a7d},
    {0xca9cf1d206fdc03b, 0xa6d90811f0e4851d},
    {0xfd442e4688bd304a, 0x908f4a166d1da664},
    {0x9e4a9cec15763e2e, 0x9a598e4e043287ff},
    {0xc5dd44271ad3cdba, 0x40eff1e1853f29fe},
    {0xf7549530e188c128, 0xd12bee59e68ef47d},
    {0x9a94dd3e8cf578b9, 0x82bb74f8301958cf},
    {0xc13a148e3032d6e7, 0xe36a52363c1faf02},
    {0xf18899b1bc3f8ca1, 0xdc44e6c3cb279ac2},
    {0x96f5600f15a7b7e5, 0x29ab103a5ef8c0ba},
    {0xbcb2b812db11a5de, 0x7415d448f6b6f0e8},
    {0xebdf661791d60f56, 0x111b495b3464ad22},
    {0x936b9fcebb25c995, 0xcab10dd900beec35},
    {0xb84687c269ef3bfb, 0x3d5d514f40eea743},
    {0xe65829b3046b0afa, 0x0cb4a5a3112a5113},
    {0x8ff71a0fe2c2e6dc, 0x47f0e785eaba72ac},
    {0xb3f4e093db73a093, 0x59ed216765690f57},
    {0xe0f218b8d25088b8, 0x306869c13ec3532d},
    {0x8c974f7383725573, 0x1e414218c73a13fc},
    {0xafbd2350644eeacf, 0xe5d1929ef90898fb},
    {0xdbac6c247d62a583, 0xdf45f746b74abf3a},
    {0x894bc396ce5da772, 0x6b8bba8c328eb784},
    {0xab9eb47c81f5114f, 0x066ea92f3f326565},
    {0xd686619ba27255a2, 0xc80a537b0efefebe},
    {0x8613fd0145877585, 0xbd06742ce95f5f37},
    {0xa798fc4196e952e7, 0x2c48113823b73705},
    {0xd17f3b51fca3a7a0, 0xf75a15862ca504c6},
    {0x82ef85133de648c4, 0x9a984d73dbe722fc},
    {0xa3ab66580d5fdaf5, 0xc13e60d0d2e0ebbb},
    {0xcc963fee10b7d1b3, 0x318df905079926a9},
    {0xffbbcfe994e5c61f, 0xfdf17746497f7053},
    {0x9fd561f1fd0f9bd3, 0xfeb6ea8bedefa634},
    {0xc7caba6e7c5382c8, 0xfe64a52ee96b8fc1},
    {0xf9bd690a1b68637b, 0x3dfdce7aa3c673b1},
    {0x9c1661a651213e2d, 0x06bea10ca65c084f},
    {0xc31bfa0fe5698db8, 0x486e494fcff30a63},
    {0xf3e2f893dec3f126, 0x5a89dba3c3efccfb},
    {0x986ddb5c6b3a76b7, 0xf89629465a75e01d},
    {0xbe89523386091465, 0xf6bbb397f1135824},
    {0xee2ba6c0678b597f, 0x746aa07ded582e2d},
    {0x94db483840b717ef, 0xa8c2a44eb4571cdd},
    {0xba121a4650e4ddeb, 0x92f34d62616ce414},
    {0xe896a0d7e51e1566, 0x77b020baf9c81d18},
    {0x915e2486ef32cd60, 0x0ace1474dc1d122f},
    {0xb5b5ada8aaff80b8, 0x0d819992132456bb},
    {0xe3231912d5bf60e6, 0x10e1fff697ed6c6a},
    {0x8df5efabc5979c8f, 0xca8d3ffa1ef463c2},
    {0xb1736b96b6fd83b3, 0xbd308ff8a6b17cb3},
    {0xddd0467c64bce4a0, 0xac7cb3f6d05ddbdf},
    {0x8aa22c0dbef60ee4, 0x6bcdf07a423aa96c},
    {0xad4ab7112eb3929d, 0x86c16c98d2c953c7},
    {0xd89d64d57a607744, 0xe871c7bf077ba8b8},
    {0x87625f056c7c4a8b, 0x11471cd764ad4973},
    {0xa93af6c6c79b5d2d, 0xd598e40d3dd89bd0},
    {0xd389b47879823479, 0x4aff1d108d4ec2c4},
    {0x843610cb4bf160cb, 0xcedf722a585139bb},
    {0xa54394fe1eedb8fe, 0xc2974eb4ee658829},
    {0xce947a3da6a9273e, 0x733d226229feea33},
    {0x811ccc668829b887, 0x0806357d5a3f5260},
    {0xa163ff802a3426a8, 0xca07c2dcb0cf26f8},
    {0xc9bcff6034c13052, 0xfc89b393dd02f0b6},
    {0xfc2c3f3841f17c67, 0xbbac2078d443ace3},
    {0x9d9ba7832936edc0, 0xd54b944b84aa4c0e},
    {0xc5029163f384a931, 0x0a9e795e65d4df12},
    {0xf64335bcf065d37d, 0x4d4617b5ff4a16d6},
    {0x99ea0196163fa42e, 0x504bced1bf8e4e46},
    {0xc06481fb9bcf8d39, 0xe45ec2862f71e1d7},
    {0xf07da27a82c37088, 0x5d767327bb4e5a4d},
    {0x964e858c91ba2655, 0x3a6a07f8d510f870},
    {0xbbe226efb628afea, 0x890489f70a55368c},
    {0xeadab0aba3b2dbe5, 0x2b45ac74ccea842f},
    {0x92c8ae6b464fc96f, 0x3b0b8bc90012929e},
    {0xb77ada0617e3bbcb, 0x09ce6ebb40173745},
    {0xe55990879ddcaabd, 0xcc420a6a101d0516},
    {0x8f57fa54c2a9eab6, 0x9fa946824a12232e},
    {0xb32df8e9f3546564, 0x47939822dc96abfa},
    {0xdff9772470297ebd, 0x59787e2b93bc56f8},
    {0x8bfbea76c619ef36, 0x57eb4edb3c55b65b},
    {0xaefae51477a06b03, 0xede622920b6b23f2},
    {0xdab99e59958885c4, 0xe95fab368e45ecee},
    {0x88b402f7fd75539b, 0x11dbcb0218ebb415},
    {0xaae103b5fcd2a881, 0xd652bdc29f26a11a},
    {0xd59944a37c0752a2, 0x4be76d3346f04960},
    {0x857fcae62d8493a5, 0x6f70a4400c562ddc},
    {0xa6dfbd9fb8e5b88e, 0xcb4ccd500f6bb953},
    {0xd097ad07a71f26b2, 0x7e2000a41346a7a8},
    {0x825ecc24c873782f, 0x8ed400668c0c28c9},
    {0xa2f67f2dfa90563b, 0x728900802f0f32fb},
    {0xcbb41ef979346bca, 0x4f2b40a03ad2ffba},
    {0xfea126b7d78186bc, 0xe2f610c84987bfa9},
    {0x9f24b832e6b0f436, 0x0dd9ca7d2df4d7ca},
    {0xc6ede63fa05d3143, 0x91503d1c79720dbc},
    {0xf8a95fcf88747d94, 0x75a44c6397ce912b},
    {0x9b69dbe1b548ce7c, 0xc986afbe3ee11abb},
    {0xc24452da229b021b, 0xfbe85badce996169},
    {0xf2d56790ab41c2a2, 0xfae27299423fb9c4},
    {0x97c560ba6b0919a5, 0xdccd879fc967d41b},
    {0xbdb6b8e905cb600f, 0x5400e987bbc1c921},
    {0xed246723473e3813, 0x290123e9aab23b69},
    {0x9436c0760c86e30b, 0xf9a0b6720aaf6522},
    {0xb94470938fa89bce, 0xf808e40e8d5b3e6a},
    {0xe7958cb87392c2c2, 0xb60b1d1230b20e05},
    {0x90bd77f3483bb9b9, 0xb1c6f22b5e6f48c3},
    {0xb4ecd5f01a4aa828, 0x1e38aeb6360b1af4},
    {0xe2280b6c20dd5232, 0x25c6da63c38de1b1},
    {0x8d590723948a535f, 0x579c487e5a38ad0f},
    {0xb0af48ec79ace837, 0x2d835a9df0c6d852},
    {0xdcdb1b2798182244, 0xf8e431456cf88e66},
    {0x8a08f0f8bf0f156b, 0x1b8e9ecb641b5900},
    {0xac8b2d36eed2dac5, 0xe272467e3d222f40},
    {0xd7adf884aa879177, 0x5b0ed81dcc6abb10},
    {0x86ccbb52ea94baea, 0x98e947129fc2b4ea},
    {0xa87fea27a539e9a5, 0x3f2398d747b36225},
    {0xd29fe4b18e88640e, 0x8eec7f0d19a03aae},
    {0x83a3eeeef9153e89, 0x1953cf68300424ad},
    {0xa48ceaaab75a8e2b, 0x5fa8c3423c052dd8},
    {0xcdb02555653131b6, 0x3792f412cb06794e},
    {0x808e17555f3ebf11, 0xe2bbd88bbee40bd1},
    {0xa0b19d2ab70e6ed6, 0x5b6aceaeae9d0ec5},
    {0xc8de047564d20a8b, 0xf245825a5a445276},
    {0xfb158592be068d2e, 0xeed6e2f0f0d56713},
    {0x9ced737bb6c4183d, 0x55464dd69685606c},
    {0xc428d05aa4751e4c, 0xaa97e14c3c26b887},
    {0xf53304714d9265df, 0xd53dd99f4b3066a9},
    {0x993fe2c6d07b7fab, 0xe546a8038efe402a},
    {0xbf8fdb78849a5f96, 0xde98520472bdd034},
    {0xef73d256a5c0f77c, 0x963e66858f6d4441},
    {0x95a8637627989aad, 0xdde7001379a44aa9},
    {0xbb127c53b17ec159, 0x5560c018580d5d53},
    {0xe9d71b689dde71af, 0xaab8f01e6e10b4a7},
    {0x9226712162ab070d, 0xcab3961304ca70e9},
    {0xb6b00d69bb55c8d1, 0x3d607b97c5fd0d23},
    {0xe45c10c42a2b3b05, 0x8cb89a7db77c506b},
    {0x8eb98a7a9a5b04e3, 0x77f3608e92adb243},
    {0xb267ed1940f1c61c, 0x55f038b237591ed4},
    {0xdf01e85f912e37a3, 0x6b6c46dec52f6689},
    {0x8b61313bbabce2c6, 0x2323ac4b3b3da016},
    {0xae397d8aa96c1b77, 0xabec975e0a0d081b},
    {0xd9c7dced53c72255, 0x96e7bd358c904a22},
    {0x881cea14545c7575, 0x7e50d64177da2e55},
    {0xaa242499697392d2, 0xdde50bd1d5d0b9ea},
    {0xd4ad2dbfc3d07787, 0x955e4ec64b44e865},
    {0x84ec3c97da624ab4, 0xbd5af13bef0b113f},
    {0xa6274bbdd0fadd61, 0xecb1ad8aeacdd58f},
    {0xcfb11ead453994ba, 0x67de18eda5814af3},
    {0x81ceb32c4b43fcf4, 0x80eacf948770ced8},
    {0xa2425ff75e14fc31, 0xa1258379a94d028e},
    {0xcad2f7f5359a3b3e, 0x096ee45813a04331},
    {0xfd87b5f28300ca0d, 0x8bca9d6e188853fd},
    {0x9e74d1b791e07e48, 0x775ea264cf55347e},
    {0xc612062576589dda, 0x95364afe032a819e},
    {0xf79687aed3eec551, 0x3a83ddbd83f52205},
    {0x9abe14cd44753b52, 0xc4926a9672793543},
    {0xc16d9a0095928a27, 0x75b7053c0f178294},
    {0xf1c90080baf72cb1, 0x5324c68b12dd6339},
    {0x971da05074da7bee, 0xd3f6fc16ebca5e04},
    {0xbce5086492111aea, 0x88f4bb1ca6bcf585},
    {0xec1e4a7db69561a5, 0x2b31e9e3d06c32e6},
    {0x9392ee8e921d5d07, 0x3aff322e62439fd0},
    {0xb877aa3236a4b449, 0x09befeb9fad487c3},
    {0xe69594bec44de15b, 0x4c2ebe687989a9b4},
    {0x901d7cf73ab0acd9, 0x0f9d37014bf60a11},
    {0xb424dc35095cd80f, 0x538484c19ef38c95},
    {0xe12e13424bb40e13, 0x2865a5f206b06fba},
    {0x8cbccc096f5088cb, 0xf93f87b7442e45d4},
    {0xafebff0bcb24aafe, 0xf78f69a51539d749},
    {0xdbe6fecebdedd5be, 0xb573440e5a884d1c},
    {0x89705f4136b4a597, 0x31680a88f8953031},
    {0xabcc77118461cefc, 0xfdc20d2b36ba7c3e},
    {0xd6bf94d5e57a42bc, 0x3d32907604691b4d},
    {0x8637bd05af6c69b5, 0xa63f9a49c2c1b110},
    {0xa7c5ac471b478423, 0x0fcf80dc33721d54},
    {0xd1b71758e219652b, 0xd3c36113404ea4a9},
    {0x83126e978d4fdf3b, 0x645a1cac083126ea},
    {0xa3d70a3d70a3d70a, 0x3d70a3d70a3d70a4},
    {0xcccccccccccccccc, 0xcccccccccccccccd},
    {0x8000000000000000, 0x0000000000000000},
    {0xa000000000000000, 0x0000000000000000},
    {0xc800000000000000, 0x0000000000000000},
    {0xfa00000000000000, 0x0000000000000000},
    {0x9c40000000000000, 0x0000000000000000},
    {0xc350000000000000, 0x0000000000000000},
    {0xf424000000000000, 0x0000000000000000},
    {0x9896800000000000, 0x0000000000000000},
    {0xbebc200000000000, 0x0000000000000000},
    {0xee6b280000000000, 0x0000000000000000},
    {0x9502f90000000000, 0x0000000000000000},
    {0xba43b74000000000, 0x0000000000000000},
    {0xe8d4a51000000000, 0x0000000000000000},
    {0x9184e72a00000000, 0x0000000000000000},
    {0xb5e620f480000000, 0x0000000000000000},
    {0xe35fa931a0000000, 0x0000000000000000},
    {0x8e1bc9bf04000000, 0x0000000000000000},
    {0xb1a2bc2ec5000000, 0x0000000000000000},
    {0xde0b6b3a76400000, 0x0000000000000000},
    {0x8ac7230489e80000, 0x0000000000000000},
    {0xad78ebc5ac620000, 0x0000000000000000},
    {0xd8d726b7177a8000, 0x0000000000000000},
    {0x878678326eac9000, 0x0000000000000000},
    {0xa968163f0a57b400, 0x0000000000000000},
    {0xd3c21bcecceda100, 0x0000000000000000},
    {0x84595161401484a0, 0x0000000000000000},
    {0xa56fa5b99019a5c8, 0x0000000000000000},
    {0xcecb8f27f4200f3a, 0x0000000000000000},
    {0x813f3978f8940984, 0x4000000000000000},
    {0xa18f07d736b90be5, 0x5000000000000000},
    {0xc9f2c9cd04674ede, 0xa400000000000000},
    {0xfc6f7c4045812296, 0x4d00000000000000},
    {0x9dc5ada82b70b59d, 0xf020000000000000},
    {0xc5371912364ce305, 0x6c28000000000000},
    {0xf684df56c3e01bc6, 0xc732000000000000},
    {0x9a130b963a6c115c, 0x3c7f400000000000},
    {0xc097ce7bc90715b3, 0x4b9f100000000000},
    {0xf0bdc21abb48db20, 0x1e86d40000000000},
    {0x96769950b50d88f4, 0x1314448000000000},
    {0xbc143fa4e250eb31, 0x17d955a000000000},
    {0xeb194f8e1ae525fd, 0x5dcfab0800000000},
    {0x92efd1b8d0cf37be, 0x5aa1cae500000000},
    {0xb7abc627050305ad, 0xf14a3d9e40000000},
    {0xe596b7b0c643c719, 0x6d9ccd05d0000000},
    {0x8f7e32ce7bea5c6f, 0xe4820023a2000000},
    {0xb35dbf821ae4f38b, 0xdda2802c8a800000},
    {0xe0352f62a19e306e, 0xd50b2037ad200000},
    {0x8c213d9da502de45, 0x4526f422cc340000},
    {0xaf298d050e4395d6, 0x9670b12b7f410000},
    {0xdaf3f04651d47b4c, 0x3c0cdd765f114000},
    {0x88d8762bf324cd0f, 0xa5880a69fb6ac800},
    {0xab0e93b6efee0053, 0x8eea0d047a457a00},
    {0xd5d238a4abe98068, 0x72a4904598d6d880},
    {0x85a36366eb71f041, 0x47a6da2b7f864750},
    {0xa70c3c40a64e6c51, 0x999090b65f67d924},
    {0xd0cf4b50cfe20765, 0xfff4b4e3f741cf6d},
    {0x82818f1281ed449f, 0xbff8f10e7a8921a4},
    {0xa321f2d7226895c7, 0xaff72d52192b6a0d},
    {0xcbea6f8ceb02bb39, 0x9bf4f8a69f764490},
    {0xfee50b7025c36a08, 0x02f236d04753d5b4},
    {0x9f4f2726179a2245, 0x01d762422c946590},
    {0xc722f0ef9d80aad6, 0x424d3ad2b7b97ef5},
    {0xf8ebad2b84e0d58b, 0xd2e0898765a7deb2},
    {0x9b934c3b330c8577, 0x63cc55f49f88eb2f},
    {0xc2781f49ffcfa6d5, 0x3cbf6b71c76b25fb},
    {0xf316271c7fc3908a, 0x8bef464e3945ef7a},
    {0x97edd871cfda3a56, 0x97758bf0e3cbb5ac},
    {0xbde94e8e43d0c8ec, 0x3d52eeed1cbea317},
    {0xed63a231d4c4fb27, 0x4ca7aaa863ee4bdd},
    {0x945e455f24fb1cf8, 0x8fe8caa93e74ef6a},
    {0xb975d6b6ee39e436, 0xb3e2fd538e122b44},
    {0xe7d34c64a9c85d44, 0x60dbbca87196b616},
    {0x90e40fbeea1d3a4a, 0xbc8955e946fe31cd},
    {0xb51d13aea4a488dd, 0x6babab6398bdbe41},
    {0xe264589a4dcdab14, 0xc696963c7eed2dd1},
    {0x8d7eb76070a08aec, 0xfc1e1de5cf543ca2},
    {0xb0de65388cc8ada8, 0x3b25a55f43294bcb},
    {0xdd15fe86affad912, 0x49ef0eb713f39ebe},
    {0x8a2dbf142dfcc7ab, 0x6e3569326c784337},
    {0xacb92ed9397bf996, 0x49c2c37f07965404},
    {0xd7e77a8f87daf7fb, 0xdc33745ec97be906},
    {0x86f0ac99b4e8dafd, 0x69a028bb3ded71a3},
    {0xa8acd7c0222311bc, 0xc40832ea0d68ce0c},
    {0xd2d80db02aabd62b, 0xf50a3fa490c30190},
    {0x83c7088e1aab65db, 0x792667c6da79e0fa},
    {0xa4b8cab1a1563f52, 0x577001b891185938},
    {0xcde6fd5e09abcf26, 0xed4c0226b55e6f86},
    {0x80b05e5ac60b6178, 0x544f8158315b05b4},
    {0xa0dc75f1778e39d6, 0x696361ae3db1c721},
    {0xc913936dd571c84c, 0x03bc3a19cd1e38e9},
    {0xfb5878494ace3a5f, 0x04ab48a04065c723},
    {0x9d174b2dcec0e47b, 0x62eb0d64283f9c76},
    {0xc45d1df942711d9a, 0x3ba5d0bd324f8394},
    {0xf5746577930d6500, 0xca8f44ec7ee36479},
    {0x9968bf6abbe85f20, 0x7e998b13cf4e1ecb},
    {0xbfc2ef456ae276e8, 0x9e3fedd8c321a67e},
    {0xefb3ab16c59b14a2, 0xc5cfe94ef3ea101e},
    {0x95d04aee3b80ece5, 0xbba1f1d158724a12},
    {0xbb445da9ca61281f, 0x2a8a6e45ae8edc97},
    {0xea1575143cf97226, 0xf52d09d71a3293bd},
    {0x924d692ca61be758, 0x593c2626705f9c56},
    {0xb6e0c377cfa2e12e, 0x6f8b2fb00c77836c},
    {0xe498f455c38b997a, 0x0b6dfb9c0f956447},
    {0x8edf98b59a373fec, 0x4724bd4189bd5eac},
    {0xb2977ee300c50fe7, 0x58edec91ec2cb657},
    {0xdf3d5e9bc0f653e1, 0x2f2967b66737e3ed},
    {0x8b865b215899f46c, 0xbd79e0d20082ee74},
    {0xae67f1e9aec07187, 0xecd8590680a3aa11},
    {0xda01ee641a708de9, 0xe80e6f4820cc9495},
    {0x884134fe908658b2, 0x3109058d147fdcdd},
    {0xaa51823e34a7eede, 0xbd4b46f0599fd415},
    {0xd4e5e2cdc1d1ea96, 0x6c9e18ac7007c91a},
    {0x850fadc09923329e, 0x03e2cf6bc604ddb0},
    {0xa6539930bf6bff45, 0x84db8346b786151c},
    {0xcfe87f7cef46ff16, 0xe612641865679a63},
    {0x81f14fae158c5f6e, 0x4fcb7e8f3f60c07e},
    {0xa26da3999aef7749, 0xe3be5e330f38f09d},
    {0xcb090c8001ab551c, 0x5cadf5bfd3072cc5},
    {0xfdcb4fa002162a63, 0x73d9732fc7c8f7f6},
    {0x9e9f11c4014dda7e, 0x2867e7fddcdd9afa},
    {0xc646d63501a1511d, 0xb281e1fd541501b8},
    {0xf7d88bc24209a565, 0x1f225a7ca91a4226},
    {0x9ae757596946075f, 0x3375788de9b06958},
    {0xc1a12d2fc3978937, 0x0052d6b1641c83ae},
    {0xf209787bb47d6b84, 0xc0678c5dbd23a49a},
    {0x9745eb4d50ce6332, 0xf840b7ba963646e0},
    {0xbd176620a501fbff, 0xb650e5a93bc3d898},
    {0xec5d3fa8ce427aff, 0xa3e51f138ab4cebe},
    {0x93ba47c980e98cdf, 0xc66f336c36b10137},
    {0xb8a8d9bbe123f017, 0xb80b0047445d4184},
    {0xe6d3102ad96cec1d, 0xa60dc059157491e5},
    {0x9043ea1ac7e41392, 0x87c89837ad68db2f},
    {0xb454e4a179dd1877, 0x29babe4598c311fb},
    {0xe16a1dc9d8545e94, 0xf4296dd6fef3d67a},
    {0x8ce2529e2734bb1d, 0x1899e4a65f58660c},
    {0xb01ae745b101e9e4, 0x5ec05dcff72e7f8f},
    {0xdc21a1171d42645d, 0x76707543f4fa1f73},
    {0x899504ae72497eba, 0x6a06494a791c53a8},
    {0xabfa45da0edbde69, 0x0487db9d17636892},
    {0xd6f8d7509292d603, 0x45a9d2845d3c42b6},
    {0x865b86925b9bc5c2, 0x0b8a2392ba45a9b2},
    {0xa7f26836f282b732, 0x8e6cac7768d7141e},
    {0xd1ef0244af2364ff, 0x3207d795430cd926},
    {0x8335616aed761f1f, 0x7f44e6bd49e807b8},
    {0xa402b9c5a8d3a6e7, 0x5f16206c9c6209a6},
    {0xcd036837130890a1, 0x36dba887c37a8c0f},
    {0x802221226be55a64, 0xc2494954da2c9789},
    {0xa02aa96b06deb0fd, 0xf2db9baa10b7bd6c},
    {0xc83553c5c8965d3d, 0x6f92829494e5acc7},
    {0xfa42a8b73abbf48c, 0xcb772339ba1f17f9},
    {0x9c69a97284b578d7, 0xff2a760414536efb},
    {0xc38413cf25e2d70d, 0xfef5138519684aba},
    {0xf46518c2ef5b8cd1, 0x7eb258665fc25d69},
    {0x98bf2f79d5993802, 0xef2f773ffbd97a61},
    {0xbeeefb584aff8603, 0xaafb550ffacfd8fa},
    {0xeeaaba2e5dbf6784, 0x95ba2a53f983cf38},
    {0x952ab45cfa97a0b2, 0xdd945a747bf26183},
    {0xba756174393d88df, 0x94f971119aeef9e4},
    {0xe912b9d1478ceb17, 0x7a37cd5601aab85d},
    {0x91abb422ccb812ee, 0xac62e055c10ab33a},
    {0xb616a12b7fe617aa, 0x577b986b314d6009},
    {0xe39c49765fdf9d94, 0xed5a7e85fda0b80b},
    {0x8e41ade9fbebc27d, 0x14588f13be847307},
    {0xb1d219647ae6b31c, 0x596eb2d8ae258fc8},
    {0xde469fbd99a05fe3, 0x6fca5f8ed9aef3bb},
    {0x8aec23d680043bee, 0x25de7bb9480d5854},
    {0xada72ccc20054ae9, 0xaf561aa79a10ae6a},
    {0xd910f7ff28069da4, 0x1b2ba1518094da04},
    {0x87aa9aff79042286, 0x90fb44d2f05d0842},
    {0xa99541bf57452b28, 0x353a1607ac744a53},
    {0xd3fa922f2d1675f2, 0x42889b8997915ce8},
    {0x847c9b5d7c2e09b7, 0x69956135febada11},
    {0xa59bc234db398c25, 0x43fab9837e699095},
    {0xcf02b2c21207ef2e, 0x94f967e45e03f4bb},
    {0x8161afb94b44f57d, 0x1d1be0eebac278f5},
    {0xa1ba1ba79e1632dc, 0x6462d92a69731732},
    {0xca28a291859bbf93, 0x7d7b8f7503cfdcfe},
    {0xfcb2cb35e702af78, 0x5cda735244c3d43e},
    {0x9defbf01b061adab, 0x3a0888136afa64a7},
    {0xc56baec21c7a1916, 0x088aaa1845b8fdd0},
    {0xf6c69a72a3989f5b, 0x8aad549e57273d45},
    {0x9a3c2087a63f6399, 0x36ac54e2f678864b},
    {0xc0cb28a98fcf3c7f, 0x84576a1bb416a7dd},
    {0xf0fdf2d3f3c30b9f, 0x656d44a2a11c51d5},
    {0x969eb7c47859e743, 0x9f644ae5a4b1b325},
    {0xbc4665b596706114, 0x873d5d9f0dde1fee},
    {0xeb57ff22fc0c7959, 0xa90cb506d155a7ea},
    {0x9316ff75dd87cbd8, 0x09a7f12442d588f2},
    {0xb7dcbf5354e9bece, 0x0c11ed6d538aeb2f},
    {0xe5d3ef282a242e81, 0x8f1668c8a86da5fa},
    {0x8fa475791a569d10, 0xf96e017d694487bc},
    {0xb38d92d760ec4455, 0x37c981dcc395a9ac},
    {0xe070f78d3927556a, 0x85bbe253f47b1417},
    {0x8c469ab843b89562, 0x93956d7478ccec8e},
    {0xaf58416654a6babb, 0x387ac8d1970027b2},
    {0xdb2e51bfe9d0696a, 0x06997b05fcc0319e},
    {0x88fcf317f22241e2, 0x441fece3bdf81f03},
    {0xab3c2fddeeaad25a, 0xd527e81cad7626c3},
    {0xd60b3bd56a5586f1, 0x8a71e223d8d3b074},
    {0x85c7056562757456, 0xf6872d5667844e49},
    {0xa738c6bebb12d16c, 0xb428f8ac016561db},
    {0xd106f86e69d785c7, 0xe13336d701beba52},
    {0x82a45b450226b39c, 0xecc0024661173473},
    {0xa34d721642b06084, 0x27f002d7f95d0190},
    {0xcc20ce9bd35c78a5, 0x31ec038df7b441f4},
    {0xff290242c83396ce, 0x7e67047175a15271},
    {0x9f79a169bd203e41, 0x0f0062c6e984d386},
    {0xc75809c42c684dd1, 0x52c07b78a3e60868},
    {0xf92e0c3537826145, 0xa7709a56ccdf8a82},
    {0x9bbcc7a142b17ccb, 0x88a66076400bb691},
    {0xc2abf989935ddbfe, 0x6acff893d00ea435},
    {0xf356f7ebf83552fe, 0x0583f6b8c4124d43},
    {0x98165af37b2153de, 0xc3727a337a8b704a},
    {0xbe1bf1b059e9a8d6, 0x744f18c0592e4c5c},
    {0xeda2ee1c7064130c, 0x1162def06f79df73},
    {0x9485d4d1c63e8be7, 0x8addcb5645ac2ba8},
    {0xb9a74a0637ce2ee1, 0x6d953e2bd7173692},
    {0xe8111c87c5c1ba99, 0xc8fa8db6ccdd0437},
    {0x910ab1d4db9914a0, 0x1d9c9892400a22a2},
    {0xb54d5e4a127f59c8, 0x2503beb6d00cab4b},
    {0xe2a0b5dc971f303a, 0x2e44ae64840fd61d},
    {0x8da471a9de737e24, 0x5ceaecfed289e5d2},
    {0xb10d8e1456105dad, 0x7425a83e872c5f47},
    {0xdd50f1996b947518, 0xd12f124e28f77719},
    {0x8a5296ffe33cc92f, 0x82bd6b70d99aaa6f},
    {0xace73cbfdc0bfb7b, 0x636cc64d1001550b},
    {0xd8210befd30efa5a, 0x3c47f7e05401aa4e},
    {0x8714a775e3e95c78, 0x65acfaec34810a71},
    {0xa8d9d1535ce3b396, 0x7f1839a741a14d0d},
    {0xd31045a8341ca07c, 0x1ede48111209a050},
    {0x83ea2b892091e44d, 0x934aed0aab460432},
    {0xa4e4b66b68b65d60, 0xf81da84d5617853f},
    {0xce1de40642e3f4b9, 0x36251260ab9d668e},
    {0x80d2ae83e9ce78f3, 0xc1d72b7c6b426019},
    {0xa1075a24e4421730, 0xb24cf65b8612f81f},
    {0xc94930ae1d529cfc, 0xdee033f26797b627},
    {0xfb9b7cd9a4a7443c, 0x169840ef017da3b1},
    {0x9d412e0806e88aa5, 0x8e1f289560ee864e},
    {0xc491798a08a2ad4e, 0xf1a6f2bab92a27e2},
    {0xf5b5d7ec8acb58a2, 0xae10af696774b1db},
    {0x9991a6f3d6bf1765, 0xacca6da1e0a8ef29},
    {0xbff610b0cc6edd3f, 0x17fd090a58d32af3},
    {0xeff394dcff8a948e, 0xddfc4b4cef07f5b0},
    {0x95f83d0a1fb69cd9, 0x4abdaf101564f98e},
    {0xbb764c4ca7a4440f, 0x9d6d1ad41abe37f1},
    {0xea53df5fd18d5513, 0x84c86189216dc5ed},
    {0x92746b9be2f8552c, 0x32fd3cf5b4e49bb4},
    {0xb7118682dbb66a77, 0x3fbc8c33221dc2a1},
    {0xe4d5e82392a40515, 0x0fabaf3feaa5334a},
    {0x8f05b1163ba6832d, 0x29cb4d87f2a7400e},
    {0xb2c71d5bca9023f8, 0x743e20e9ef511012},
    {0xdf78e4b2bd342cf6, 0x914da9246b255416},
    {0x8bab8eefb6409c1a, 0x1ad089b6c2f7548e},
    {0xae9672aba3d0c320, 0xa184ac2473b529b1},
    {0xda3c0f568cc4f3e8, 0xc9e5d72d90a2741e},
    {0x8865899617fb1871, 0x7e2fa67c7a658892},
    {0xaa7eebfb9df9de8d, 0xddbb901b98feeab7},
    {0xd51ea6fa85785631, 0x552a74227f3ea565},
    {0x8533285c936b35de, 0xd53a88958f87275f},
    {0xa67ff273b8460356, 0x8a892abaf368f137},
    {0xd01fef10a657842c, 0x2d2b7569b0432d85},
    {0x8213f56a67f6b29b, 0x9c3b29620e29fc73},
    {0xa298f2c501f45f42, 0x8349f3ba91b47b8f},
    {0xcb3f2f7642717713, 0x241c70a936219a73},
    {0xfe0efb53d30dd4d7, 0xed238cd383aa0110},
    {0x9ec95d1463e8a506, 0xf4363804324a40aa},
    {0xc67bb4597ce2ce48, 0xb143c6053edcd0d5},
    {0xf81aa16fdc1b81da, 0xdd94b7868e94050a},
    {0x9b10a4e5e9913128, 0xca7cf2b4191c8326},
    {0xc1d4ce1f63f57d72, 0xfd1c2f611f63a3f0},
    {0xf24a01a73cf2dccf, 0xbc633b39673c8cec},
    {0x976e41088617ca01, 0xd5be0503e085d813},
    {0xbd49d14aa79dbc82, 0x4b2d8644d8a74e18},
    {0xec9c459d51852ba2, 0xddf8e7d60ed1219e},
    {0x93e1ab8252f33b45, 0xcabb90e5c942b503},
    {0xb8da1662e7b00a17, 0x3d6a751f3b936243},
    {0xe7109bfba19c0c9d, 0x0cc512670a783ad4},
    {0x906a617d450187e2, 0x27fb2b80668b24c5},
    {0xb484f9dc9641e9da, 0xb1f9f660802dedf6},
    {0xe1a63853bbd26451, 0x5e7873f8a0396973},
    {0x8d07e33455637eb2, 0xdb0b487b6423e1e8},
    {0xb049dc016abc5e5f, 0x91ce1a9a3d2cda62},
    {0xdc5c5301c56b75f7, 0x7641a140cc7810fb},
    {0x89b9b3e11b6329ba, 0xa9e904c87fcb0a9d},
    {0xac2820d9623bf429, 0x546345fa9fbdcd44},
    {0xd732290fbacaf133, 0xa97c177947ad4095},
    {0x867f59a9d4bed6c0, 0x49ed8eabcccc485d},
    {0xa81f301449ee8c70, 0x5c68f256bfff5a74},
    {0xd226fc195c6a2f8c, 0x73832eec6fff3111},
    {0x83585d8fd9c25db7, 0xc831fd53c5ff7eab},
    {0xa42e74f3d032f525, 0xba3e7ca8b77f5e55},
    {0xcd3a1230c43fb26f, 0x28ce1bd2e55f35eb},
    {0x80444b5e7aa7cf85, 0x7980d163cf5b81b3},
    {0xa0555e361951c366, 0xd7e105bcc332621f},
    {0xc86ab5c39fa63440, 0x8dd9472bf3fefaa7},
    {0xfa856334878fc150, 0xb14f98f6f0feb951},
    {0x9c935e00d4b9d8d2, 0x6ed1bf9a569f33d3},
    {0xc3b8358109e84f07, 0x0a862f80ec4700c8},
    {0xf4a642e14c6262c8, 0xcd27bb612758c0fa},
    {0x98e7e9cccfbd7dbd, 0x8038d51cb897789c},
    {0xbf21e44003acdd2c, 0xe0470a63e6bd56c3},
    {0xeeea5d5004981478, 0x1858ccfce06cac74},
    {0x95527a5202df0ccb, 0x0f37801e0c43ebc8},
    {0xbaa718e68396cffd, 0xd30560258f54e6ba},
    {0xe950df20247c83fd, 0x47c6b82ef32a2069},
    {0x91d28b7416cdd27e, 0x4cdc331d57fa5441},
    {0xb6472e511c81471d, 0xe0133fe4adf8e952},
    {0xe3d8f9e563a198e5, 0x58180fddd97723a6},
    {0x8e679c2f5e44ff8f, 0x570f09eaa7ea7648},
    {0xb201833b35d63f73, 0x2cd2cc6551e513da},
    {0xde81e40a034bcf4f, 0xf8077f7ea65e58d1},
    {0x8b112e86420f6191, 0xfb04afaf27faf782},
    {0xadd57a27d29339f6, 0x79c5db9af1f9b563},
    {0xd94ad8b1c7380874, 0x18375281ae7822bc},
    {0x87cec76f1c830548, 0x8f2293910d0b15b5},
    {0xa9c2794ae3a3c69a, 0xb2eb3875504ddb22},
    {0xd433179d9c8cb841, 0x5fa60692a46151eb},
    {0x849feec281d7f328, 0xdbc7c41ba6bcd333},
    {0xa5c7ea73224deff3, 0x12b9b522906c0800},
    {0xcf39e50feae16bef, 0xd768226b34870a00},
    {0x81842f29f2cce375, 0xe6a1158300d46640},
    {0xa1e53af46f801c53, 0x60495ae3c1097fd0},
    {0xca5e89b18b602368, 0x385bb19cb14bdfc4},
    {0xfcf62c1dee382c42, 0x46729e03dd9ed7b5},
    {0x9e19db92b4e31ba9, 0x6c07a2c26a8346d1},
    {0xc5a05277621be293, 0xc7098b7305241885},
    {0xf70867153aa2db38, 0xb8cbee4fc66d1ea7}
```



This appears to be a list of hexadecimal values, each of which appears to be a machine learning的唯一 security identifier (MSI)摇码。MSI is a unique 32-byte identifier that is assigned to an organization by Microsoft Encryption做为自己的数字证书， and is used to identify the identity of a computer or device in the IoT GRC (Industrial IOT Governance and Access Control).

Each value in the list is a combination of a Hash of a SafeMsSWord (a deterministic memory pool used for the generation of security keys) and a checksum, which is used to verify the integrity of the MSI.

It is worth noting that this list does not appear to be a complete and active list of valid MSIs, as it does not include the防卫prime，独立密钥，和资格键等三个标志位的值。


```cpp
#else
    {0xff77b1fcbebcdc4f, 0x25e8e89c13bb0f7b},
    {0xce5d73ff402d98e3, 0xfb0a3d212dc81290},
    {0xa6b34ad8c9dfc06f, 0xf42faa48c0ea481f},
    {0x86a8d39ef77164bc, 0xae5dff9c02033198},
    {0xd98ddaee19068c76, 0x3badd624dd9b0958},
    {0xafbd2350644eeacf, 0xe5d1929ef90898fb},
    {0x8df5efabc5979c8f, 0xca8d3ffa1ef463c2},
    {0xe55990879ddcaabd, 0xcc420a6a101d0516},
    {0xb94470938fa89bce, 0xf808e40e8d5b3e6a},
    {0x95a8637627989aad, 0xdde7001379a44aa9},
    {0xf1c90080baf72cb1, 0x5324c68b12dd6339},
    {0xc350000000000000, 0x0000000000000000},
    {0x9dc5ada82b70b59d, 0xf020000000000000},
    {0xfee50b7025c36a08, 0x02f236d04753d5b4},
    {0xcde6fd5e09abcf26, 0xed4c0226b55e6f86},
    {0xa6539930bf6bff45, 0x84db8346b786151c},
    {0x865b86925b9bc5c2, 0x0b8a2392ba45a9b2},
    {0xd910f7ff28069da4, 0x1b2ba1518094da04},
    {0xaf58416654a6babb, 0x387ac8d1970027b2},
    {0x8da471a9de737e24, 0x5ceaecfed289e5d2},
    {0xe4d5e82392a40515, 0x0fabaf3feaa5334a},
    {0xb8da1662e7b00a17, 0x3d6a751f3b936243},
    {0x95527a5202df0ccb, 0x0f37801e0c43ebc8}
```

This is a raw bytes array that appears to be a hexadecimal representation of a block of memory. Each element in the array is a 2-byte (or 16-bit) unsigned integer. The exact meaning of each number in the array is not clear without more information.



```cpp
#endif
};

#if !FMT_USE_FULL_CACHE_DRAGONBOX
template <typename T>
const uint64_t basic_data<T>::powers_of_5_64[] = {
    0x0000000000000001, 0x0000000000000005, 0x0000000000000019,
    0x000000000000007d, 0x0000000000000271, 0x0000000000000c35,
    0x0000000000003d09, 0x000000000001312d, 0x000000000005f5e1,
    0x00000000001dcd65, 0x00000000009502f9, 0x0000000002e90edd,
    0x000000000e8d4a51, 0x0000000048c27395, 0x000000016bcc41e9,
    0x000000071afd498d, 0x0000002386f26fc1, 0x000000b1a2bc2ec5,
    0x000003782dace9d9, 0x00001158e460913d, 0x000056bc75e2d631,
    0x0001b1ae4d6e2ef5, 0x000878678326eac9, 0x002a5a058fc295ed,
    0x00d3c21bcecceda1, 0x0422ca8b0a00a425, 0x14adf4b7320334b9};

```

This is a C++ header file that defines a template `basic_data<int>` that includes a range of 256 possible integer values.

The header file specifies a two-dimensional array called `foreground_color` and `background_color`, each with 0x1b开头 and each with 4 bytes of data. These arrays are used to color the integer values in the `basic_data` template, based on the value of the `T` parameter.

The `basic_data` template parameter is a type-safe type指针， but it does not explicitly declare the type of the `T` parameter. This allows the function to accept any type of integer value.

The `basic_data` template includes a range of 256 possible integer values, arranged in a two-dimensional array called `baseline_grid`. This array is defined as a const char array with a two-dimensional size of `[[1]]` by `[[2]]`. This means that the array has one row and two columns.

The `baseline_grid` array includes the values 0, 1, 2, ..., 255, as well as some default values.

The `basic_data` header file is intended to be used as a template for functions that take an integer value as input, but it does not provide any type safety guarantees for the `T` parameter. The user must ensure that the `T` parameter is of the correct type for the function to be called.


```cpp
template <typename T>
const uint32_t basic_data<T>::dragonbox_pow10_recovery_errors[] = {
    0x50001400, 0x54044100, 0x54014555, 0x55954415, 0x54115555, 0x00000001,
    0x50000000, 0x00104000, 0x54010004, 0x05004001, 0x55555544, 0x41545555,
    0x54040551, 0x15445545, 0x51555514, 0x10000015, 0x00101100, 0x01100015,
    0x00000000, 0x00000000, 0x00000000, 0x00000000, 0x04450514, 0x45414110,
    0x55555145, 0x50544050, 0x15040155, 0x11054140, 0x50111514, 0x11451454,
    0x00400541, 0x00000000, 0x55555450, 0x10056551, 0x10054011, 0x55551014,
    0x69514555, 0x05151109, 0x00155555};
#endif

template <typename T>
const char basic_data<T>::foreground_color[] = "\x1b[38;2;";
template <typename T>
const char basic_data<T>::background_color[] = "\x1b[48;2;";
```

这段代码定义了一个名为 basic_data 的模板结构体，其中包含一个名为 signs 的字符串，它是一个包含四个字符的数组，每个字符表示一个不同的符号。这个结构体还有一个名为 left_padding_shifts 和 right_padding_shifts 的数组，它们分别包含控制左和右括号 padding 的偏移量。

此外，在 template <typename T> 中，定义了一个名为 bits 的结构体，其中包含一个名为 value 的整数，它的值为 static_cast<int>(sizeof(T) * std::numeric_limits<unsigned char>::digits)，这个值表示 T 类型所占用的字节数。

最后，定义了一个名为 fp 的模板类，其中包含一个名为 normalize 的函数，该函数将一个 fp 类型的值转换为整数类型的值，并返回这个整数。


```cpp
template <typename T> const char basic_data<T>::reset_color[] = "\x1b[0m";
template <typename T> const wchar_t basic_data<T>::wreset_color[] = L"\x1b[0m";
template <typename T> const char basic_data<T>::signs[] = {0, '-', '+', ' '};
template <typename T>
const char basic_data<T>::left_padding_shifts[] = {31, 31, 0, 1, 0};
template <typename T>
const char basic_data<T>::right_padding_shifts[] = {0, 31, 0, 1, 0};

template <typename T> struct bits {
  static FMT_CONSTEXPR_DECL const int value =
      static_cast<int>(sizeof(T) * std::numeric_limits<unsigned char>::digits);
};

class fp;
template <int SHIFT = 0> fp normalize(fp value);

```

This code appears to be a C++ implementation of a mathematical calculation functions such as √2, ∛3, √10, etc. It uses the有限制符涵盖符号和指数的表示形式，即float类型。该实现中，对于无符号浮点数，将浮点数的符号作为参数传递给函数，并将结果进行截断，保留整数部分作为浮点数的表示。

它还实现了一个双精度浮点数函数，用于计算两个浮点数的比例，可以确保结果的准确性。该函数允许在函数中直接使用double类型的变量，以便在计算中使用更多内存空间。

此外，该库的实现还提供了一些辅助类型，如有限制符涵盖的布尔类型，用于判断是否可以对浮点数进行±∞的运算。


```cpp
// Lower (upper) boundary is a value half way between a floating-point value
// and its predecessor (successor). Boundaries have the same exponent as the
// value so only significands are stored.
struct boundaries {
  uint64_t lower;
  uint64_t upper;
};

// A handmade floating-point number f * pow(2, e).
class fp {
 private:
  using significand_type = uint64_t;

  template <typename Float>
  using is_supported_float = bool_constant<sizeof(Float) == sizeof(uint64_t) ||
                                           sizeof(Float) == sizeof(uint32_t)>;

 public:
  significand_type f;
  int e;

  // All sizes are in bits.
  // Subtract 1 to account for an implicit most significant bit in the
  // normalized form.
  static FMT_CONSTEXPR_DECL const int double_significand_size =
      std::numeric_limits<double>::digits - 1;
  static FMT_CONSTEXPR_DECL const uint64_t implicit_bit =
      1ULL << double_significand_size;
  static FMT_CONSTEXPR_DECL const int significand_size =
      bits<significand_type>::value;

  fp() : f(0), e(0) {}
  fp(uint64_t f_val, int e_val) : f(f_val), e(e_val) {}

  // Constructs fp from an IEEE754 double. It is a template to prevent compile
  // errors on platforms where double is not IEEE754.
  template <typename Double> explicit fp(Double d) { assign(d); }

  // Assigns d to this and return true iff predecessor is closer than successor.
  template <typename Float, FMT_ENABLE_IF(is_supported_float<Float>::value)>
  bool assign(Float d) {
    // Assume float is in the format [sign][exponent][significand].
    using limits = std::numeric_limits<Float>;
    const int float_significand_size = limits::digits - 1;
    const int exponent_size =
        bits<Float>::value - float_significand_size - 1;  // -1 for sign
    const uint64_t float_implicit_bit = 1ULL << float_significand_size;
    const uint64_t significand_mask = float_implicit_bit - 1;
    const uint64_t exponent_mask = (~0ULL >> 1) & ~significand_mask;
    const int exponent_bias = (1 << exponent_size) - limits::max_exponent - 1;
    constexpr bool is_double = sizeof(Float) == sizeof(uint64_t);
    auto u = bit_cast<conditional_t<is_double, uint64_t, uint32_t>>(d);
    f = u & significand_mask;
    int biased_e =
        static_cast<int>((u & exponent_mask) >> float_significand_size);
    // Predecessor is closer if d is a normalized power of 2 (f == 0) other than
    // the smallest normalized number (biased_e > 1).
    bool is_predecessor_closer = f == 0 && biased_e > 1;
    if (biased_e != 0)
      f += float_implicit_bit;
    else
      biased_e = 1;  // Subnormals use biased exponent 1 (min exponent).
    e = biased_e - exponent_bias - float_significand_size;
    return is_predecessor_closer;
  }

  template <typename Float, FMT_ENABLE_IF(!is_supported_float<Float>::value)>
  bool assign(Float) {
    *this = fp();
    return false;
  }
};

```

这段代码定义了一个名为 `normalize` 的模板函数，用于将一个 double 类型的值转换为 int 类型，并对其进行归一化处理。

双精度(double)类型通常占用 8 字节，其中包含 8 位整数和 24 位浮点数。将这些数字转换为整数时，会丢失 24 位浮点数。因此，为了弥补这个损失，我们可以将双精度数字转换为字符串，并对字符串进行正常化，这通常意味着将字符串转换为 lowercase，并删除末尾的换行符。

对于给定的 double 值，首先将其转换为整数类型，这涉及到使用 fp::implicit_bit 掩码将 double 值转换为整数类型。然后，代码通过 while 循环来处理 subnormals(即双精度值的副本)。在循环中，首先检查给定的 double 值是否包含 SHIFT 位，如果是，则将其转换为 int 类型，并移动到 fp::significand_size 位置，然后将 SHIFT 位向左移动 SHIFT 位和隐藏位数量减 1 的结果，以便在转换回 double 类型时能够正确处理 subnormals。然后，将剩余的 23 位二进制数转换为 int 类型，并将其添加到 fp::significand_size 位置。最后，将转换后的整数类型值返回。


```cpp
// Normalizes the value converted from double and multiplied by (1 << SHIFT).
template <int SHIFT> fp normalize(fp value) {
  // Handle subnormals.
  const auto shifted_implicit_bit = fp::implicit_bit << SHIFT;
  while ((value.f & shifted_implicit_bit) == 0) {
    value.f <<= 1;
    --value.e;
  }
  // Subtract 1 to account for hidden bit.
  const auto offset =
      fp::significand_size - fp::double_significand_size - SHIFT - 1;
  value.f <<= offset;
  value.e -= offset;
  return value;
}

```

这段代码定义了一个内联的比较运算符 `==` 和一个计算两个整数 `lhs` 和 `rhs` 乘积的函数 `multiply`。

`==` 是一个内联的布尔比较运算符，用于比较两个 `fp` 类型的变量 `x` 和 `y` 是否相等。它的实现比较简单，直接比较两个变量的 `f` 成员是否相等，如果相等则返回 `true`，否则返回 `false`。

`multiply` 是一个内联的函数，用于计算两个 `uint64_t` 类型的变量 `lhs` 和 `rhs` 的乘积，并对其进行一些计算和取模操作，最后输出结果。

具体来说，`multiply` 的实现分为两部分：

1. 计算乘积 `product`，并将其取模 `2^64` 以获取对 `2^64` 取模的余数，然后将其赋值给 `f` 变量。
2. 对于 `static_cast<__uint128_t>(lhs) * rhs` 这一部分，由于 `FMT_USE_INT128` 宏定义中已经使用了 `__uint128_t` 类型，所以这一部分的实现直接使用这个宏定义，即对两个 `uint64_t` 进行乘法运算，并将其结果赋值给 `f` 变量。
3. 对于 `static_cast<uint64_t>(product) & (1ULL << 63)` 这一部分，由于 `1` 是一个 `uint64_t` 类型，而 `1ULL` 是一个 `ullittle` 类型，它们之间的差值是 `ullittle<uint64_t<1>>`，也就是 `1`。因此，这一部分的实现是对乘积 `product` 进行右移操作，并将其与 `ullittle<uint64_t<1>>` 进行按位与操作，结果赋值给 `f` 变量。
4. 对于 `static_cast<uint64_t>(product) & (1U << 31)` 这一部分，由于 `1` 是一个 `uint64_t` 类型，而 `1U` 是一个 `uint64_t` 类型，它们之间的差值是 `ullittle<uint64_t<1>>`，也就是 `1`。因此，这一部分的实现是对乘积 `product` 进行右移操作，并将其与 `ullittle<uint64_t<1>>` 进行按位或操作，结果赋值给 `f` 变量。
5. 对于 `(static_cast<uint64_t>(product) & (1U << 32)) != 0` 这一部分，由于 `static_cast<uint64_t>(product) & (1U << 32)` 这一部分的实现是对乘积 `product` 进行右移操作，并将其与 `ullittle<uint64_t<32>>` 进行按位与操作，结果赋值给 `mid` 变量。而 `!= 0` 的判断则是检查该结果是否为 `0`，如果是，则说明两个 `uint64_t` 不相等，返回 `false`；否则说明两个 `uint64_t` 相等，返回 `true`。


```cpp
inline bool operator==(fp x, fp y) { return x.f == y.f && x.e == y.e; }

// Computes lhs * rhs / pow(2, 64) rounded to nearest with half-up tie breaking.
inline uint64_t multiply(uint64_t lhs, uint64_t rhs) {
#if FMT_USE_INT128
  auto product = static_cast<__uint128_t>(lhs) * rhs;
  auto f = static_cast<uint64_t>(product >> 64);
  return (static_cast<uint64_t>(product) & (1ULL << 63)) != 0 ? f + 1 : f;
#else
  // Multiply 32-bit parts of significands.
  uint64_t mask = (1ULL << 32) - 1;
  uint64_t a = lhs >> 32, b = lhs & mask;
  uint64_t c = rhs >> 32, d = rhs & mask;
  uint64_t ac = a * c, bc = b * c, ad = a * d, bd = b * d;
  // Compute mid 64-bit of result and round.
  uint64_t mid = (bd >> 32) + (ad & mask) + (bc & mask) + (1U << 31);
  return ac + (ad >> 32) + (bc >> 32) + (mid >> 32);
```

这段代码包括两个函数，其中第一个函数名为`operator*()`，是类型fp的联合类型操作符，用于将两个fp数组合并。第二个函数名为`get_cached_power()`，返回一个整数类型的缓存幂，该缓存幂可以用于以一种更高效的方式计算幂。

在函数内部，首先定义了一个静置的常量`c_k`，它是一个整数类型的变量，其值为1。然后定义了一个名为`pow10_exponent`的整变量，用于表示具有最小指数的缓存幂。

接下来，定义了一个名为`data::log10_2_significand`的函数，它返回以2为底，对10进行逆对数的整数部分。然后，定义了一个名为`data::grisu_pow10_significands`的函数，它返回一个整数类型的数据结构，该结构体存储了具有指定指数的幂信号。

在`get_cached_power()`函数内部，定义了一个名为`min_exponent`的整变量，用于表示要查找的最低指数。然后，定义了一个名为`pow10_exponent`的整变量，用于表示计算出来的缓存幂。接下来，定义了一个名为`significand_size`的整变量，用于存储数据类型fp的符号长度。

然后，计算出`ceil()`函数的结果，该函数用于向上取整。接着，将计算出的指数`index`与`first_dec_exp`的差值作为计算条件，使用移动算术进行位运算，并将结果存储到`pow10_exponent`中。

最后，返回计算得到的缓存幂，该缓存幂包括符号部分和指数部分。


```cpp
#endif
}

inline fp operator*(fp x, fp y) { return {multiply(x.f, y.f), x.e + y.e + 64}; }

// Returns a cached power of 10 `c_k = c_k.f * pow(2, c_k.e)` such that its
// (binary) exponent satisfies `min_exponent <= c_k.e <= min_exponent + 28`.
inline fp get_cached_power(int min_exponent, int& pow10_exponent) {
  const int shift = 32;
  const auto significand = static_cast<int64_t>(data::log10_2_significand);
  int index = static_cast<int>(
      ((min_exponent + fp::significand_size - 1) * (significand >> shift) +
       ((int64_t(1) << shift) - 1))  // ceil
      >> 32                          // arithmetic shift
  );
  // Decimal exponent of the first (smallest) cached power of 10.
  const int first_dec_exp = -348;
  // Difference between 2 consecutive decimal exponents in cached powers of 10.
  const int dec_exp_step = 8;
  index = (index - first_dec_exp - 1) / dec_exp_step + 1;
  pow10_exponent = first_dec_exp + index * dec_exp_step;
  return {data::grisu_pow10_significands[index],
          data::grisu_pow10_exponents[index]};
}

```

这段代码定义了一个名为`accumulator`的结构体，用于保存一系列二进制数的平方。如果`uint128_t`不可用，则使用`uint64_t`。该结构体包含两个成员变量：`lower`和`upper`，都为`uint64_t`类型。

此外，该结构体还重载了运算符`+=`和`>>=`，使其可以像加法器和除法器一样使用。

结构体的构造函数初始化`lower`和`upper`为0，并使用默认构造函数计算出`uint32_t`类型的值。

`operator uint32_t()`函数用于将`accumulator`对象转换为`uint32_t`类型的值，并返回`lower`成员的值。

`operator (uint64_t n)`函数接受一个`uint64_t`类型的参数`n`，并将其与`accumulator`对象相加。如果`n`小于`lower`的值，则将`upper`的值递增，以便能够将`n`的二进制位添加到`lower`中。

`operator void()`函数是一个空运算符，用于设置或取消`accumulator`对象的输入依赖。


```cpp
// A simple accumulator to hold the sums of terms in bigint::square if uint128_t
// is not available.
struct accumulator {
  uint64_t lower;
  uint64_t upper;

  accumulator() : lower(0), upper(0) {}
  explicit operator uint32_t() const { return static_cast<uint32_t>(lower); }

  void operator+=(uint64_t n) {
    lower += n;
    if (lower < n) ++upper;
  }
  void operator>>=(int shift) {
    assert(shift == 32);
    (void)shift;
    lower = (upper << 32) | (lower >> 32);
    upper >>= 32;
  }
};

```

This is a C++ implementation of a bigint data type that can be used for high-level arithmetic operations. The bigint data type is implemented as a collection of bit counts, with a single bit for each bit position in the bigint.

The `bigint` class has several member functions for performing basic arithmetic operations:

* `clear`: Clears the `bigint` instance.
* `add`: Adds two `bigint` instances, resulting in a new `bigint` instance with the result.
* `subtract`: Subtracts one `bigint` instance from another, resulting in a new `bigint` instance with the result.
* `mul`: Multiplies two `bigint` instances, resulting in a new `bigint` instance with the result.
* `invmod`: Invokes the modulo operation, returning the result as an `int`.
* `modpowmod`: Invokes the modulo operation repeatedly, returning the result as an `int`.
* `remainder`: Remains the `bigint` instance after performing the modulo operation.
* `divmod`: Divides the `bigint` instance by a `divisor` and returns the quotient.
* `align`: Aligns the `bigint` instance to the specified `divisor`.

The `divmod_assign` function can be used to perform division and remainder operations. It takes a `divisor` as an argument and returns the quotient.


```cpp
class bigint {
 private:
  // A bigint is stored as an array of bigits (big digits), with bigit at index
  // 0 being the least significant one.
  using bigit = uint32_t;
  using double_bigit = uint64_t;
  enum { bigits_capacity = 32 };
  basic_memory_buffer<bigit, bigits_capacity> bigits_;
  int exp_;

  bigit operator[](int index) const { return bigits_[to_unsigned(index)]; }
  bigit& operator[](int index) { return bigits_[to_unsigned(index)]; }

  static FMT_CONSTEXPR_DECL const int bigit_bits = bits<bigit>::value;

  friend struct formatter<bigint>;

  void subtract_bigits(int index, bigit other, bigit& borrow) {
    auto result = static_cast<double_bigit>((*this)[index]) - other - borrow;
    (*this)[index] = static_cast<bigit>(result);
    borrow = static_cast<bigit>(result >> (bigit_bits * 2 - 1));
  }

  void remove_leading_zeros() {
    int num_bigits = static_cast<int>(bigits_.size()) - 1;
    while (num_bigits > 0 && (*this)[num_bigits] == 0) --num_bigits;
    bigits_.resize(to_unsigned(num_bigits + 1));
  }

  // Computes *this -= other assuming aligned bigints and *this >= other.
  void subtract_aligned(const bigint& other) {
    FMT_ASSERT(other.exp_ >= exp_, "unaligned bigints");
    FMT_ASSERT(compare(*this, other) >= 0, "");
    bigit borrow = 0;
    int i = other.exp_ - exp_;
    for (size_t j = 0, n = other.bigits_.size(); j != n; ++i, ++j)
      subtract_bigits(i, other.bigits_[j], borrow);
    while (borrow > 0) subtract_bigits(i, 0, borrow);
    remove_leading_zeros();
  }

  void multiply(uint32_t value) {
    const double_bigit wide_value = value;
    bigit carry = 0;
    for (size_t i = 0, n = bigits_.size(); i < n; ++i) {
      double_bigit result = bigits_[i] * wide_value + carry;
      bigits_[i] = static_cast<bigit>(result);
      carry = static_cast<bigit>(result >> bigit_bits);
    }
    if (carry != 0) bigits_.push_back(carry);
  }

  void multiply(uint64_t value) {
    const bigit mask = ~bigit(0);
    const double_bigit lower = value & mask;
    const double_bigit upper = value >> bigit_bits;
    double_bigit carry = 0;
    for (size_t i = 0, n = bigits_.size(); i < n; ++i) {
      double_bigit result = bigits_[i] * lower + (carry & mask);
      carry =
          bigits_[i] * upper + (result >> bigit_bits) + (carry >> bigit_bits);
      bigits_[i] = static_cast<bigit>(result);
    }
    while (carry != 0) {
      bigits_.push_back(carry & mask);
      carry >>= bigit_bits;
    }
  }

 public:
  bigint() : exp_(0) {}
  explicit bigint(uint64_t n) { assign(n); }
  ~bigint() { assert(bigits_.capacity() <= bigits_capacity); }

  bigint(const bigint&) = delete;
  void operator=(const bigint&) = delete;

  void assign(const bigint& other) {
    auto size = other.bigits_.size();
    bigits_.resize(size);
    auto data = other.bigits_.data();
    std::copy(data, data + size, make_checked(bigits_.data(), size));
    exp_ = other.exp_;
  }

  void assign(uint64_t n) {
    size_t num_bigits = 0;
    do {
      bigits_[num_bigits++] = n & ~bigit(0);
      n >>= bigit_bits;
    } while (n != 0);
    bigits_.resize(num_bigits);
    exp_ = 0;
  }

  int num_bigits() const { return static_cast<int>(bigits_.size()) + exp_; }

  FMT_NOINLINE bigint& operator<<=(int shift) {
    assert(shift >= 0);
    exp_ += shift / bigit_bits;
    shift %= bigit_bits;
    if (shift == 0) return *this;
    bigit carry = 0;
    for (size_t i = 0, n = bigits_.size(); i < n; ++i) {
      bigit c = bigits_[i] >> (bigit_bits - shift);
      bigits_[i] = (bigits_[i] << shift) + carry;
      carry = c;
    }
    if (carry != 0) bigits_.push_back(carry);
    return *this;
  }

  template <typename Int> bigint& operator*=(Int value) {
    FMT_ASSERT(value > 0, "");
    multiply(uint32_or_64_or_128_t<Int>(value));
    return *this;
  }

  friend int compare(const bigint& lhs, const bigint& rhs) {
    int num_lhs_bigits = lhs.num_bigits(), num_rhs_bigits = rhs.num_bigits();
    if (num_lhs_bigits != num_rhs_bigits)
      return num_lhs_bigits > num_rhs_bigits ? 1 : -1;
    int i = static_cast<int>(lhs.bigits_.size()) - 1;
    int j = static_cast<int>(rhs.bigits_.size()) - 1;
    int end = i - j;
    if (end < 0) end = 0;
    for (; i >= end; --i, --j) {
      bigit lhs_bigit = lhs[i], rhs_bigit = rhs[j];
      if (lhs_bigit != rhs_bigit) return lhs_bigit > rhs_bigit ? 1 : -1;
    }
    if (i != j) return i > j ? 1 : -1;
    return 0;
  }

  // Returns compare(lhs1 + lhs2, rhs).
  friend int add_compare(const bigint& lhs1, const bigint& lhs2,
                         const bigint& rhs) {
    int max_lhs_bigits = (std::max)(lhs1.num_bigits(), lhs2.num_bigits());
    int num_rhs_bigits = rhs.num_bigits();
    if (max_lhs_bigits + 1 < num_rhs_bigits) return -1;
    if (max_lhs_bigits > num_rhs_bigits) return 1;
    auto get_bigit = [](const bigint& n, int i) -> bigit {
      return i >= n.exp_ && i < n.num_bigits() ? n[i - n.exp_] : 0;
    };
    double_bigit borrow = 0;
    int min_exp = (std::min)((std::min)(lhs1.exp_, lhs2.exp_), rhs.exp_);
    for (int i = num_rhs_bigits - 1; i >= min_exp; --i) {
      double_bigit sum =
          static_cast<double_bigit>(get_bigit(lhs1, i)) + get_bigit(lhs2, i);
      bigit rhs_bigit = get_bigit(rhs, i);
      if (sum > rhs_bigit + borrow) return 1;
      borrow = rhs_bigit + borrow - sum;
      if (borrow > 1) return -1;
      borrow <<= bigit_bits;
    }
    return borrow != 0 ? -1 : 0;
  }

  // Assigns pow(10, exp) to this bigint.
  void assign_pow10(int exp) {
    assert(exp >= 0);
    if (exp == 0) return assign(1);
    // Find the top bit.
    int bitmask = 1;
    while (exp >= bitmask) bitmask <<= 1;
    bitmask >>= 1;
    // pow(10, exp) = pow(5, exp) * pow(2, exp). First compute pow(5, exp) by
    // repeated squaring and multiplication.
    assign(5);
    bitmask >>= 1;
    while (bitmask != 0) {
      square();
      if ((exp & bitmask) != 0) *this *= 5;
      bitmask >>= 1;
    }
    *this <<= exp;  // Multiply by pow(2, exp) by shifting.
  }

  void square() {
    basic_memory_buffer<bigit, bigits_capacity> n(std::move(bigits_));
    int num_bigits = static_cast<int>(bigits_.size());
    int num_result_bigits = 2 * num_bigits;
    bigits_.resize(to_unsigned(num_result_bigits));
    using accumulator_t = conditional_t<FMT_USE_INT128, uint128_t, accumulator>;
    auto sum = accumulator_t();
    for (int bigit_index = 0; bigit_index < num_bigits; ++bigit_index) {
      // Compute bigit at position bigit_index of the result by adding
      // cross-product terms n[i] * n[j] such that i + j == bigit_index.
      for (int i = 0, j = bigit_index; j >= 0; ++i, --j) {
        // Most terms are multiplied twice which can be optimized in the future.
        sum += static_cast<double_bigit>(n[i]) * n[j];
      }
      (*this)[bigit_index] = static_cast<bigit>(sum);
      sum >>= bits<bigit>::value;  // Compute the carry.
    }
    // Do the same for the top half.
    for (int bigit_index = num_bigits; bigit_index < num_result_bigits;
         ++bigit_index) {
      for (int j = num_bigits - 1, i = bigit_index - j; i < num_bigits;)
        sum += static_cast<double_bigit>(n[i++]) * n[j--];
      (*this)[bigit_index] = static_cast<bigit>(sum);
      sum >>= bits<bigit>::value;
    }
    --num_result_bigits;
    remove_leading_zeros();
    exp_ *= 2;
  }

  // If this bigint has a bigger exponent than other, adds trailing zero to make
  // exponents equal. This simplifies some operations such as subtraction.
  void align(const bigint& other) {
    int exp_difference = exp_ - other.exp_;
    if (exp_difference <= 0) return;
    int num_bigits = static_cast<int>(bigits_.size());
    bigits_.resize(to_unsigned(num_bigits + exp_difference));
    for (int i = num_bigits - 1, j = i + exp_difference; i >= 0; --i, --j)
      bigits_[j] = bigits_[i];
    std::uninitialized_fill_n(bigits_.data(), exp_difference, 0);
    exp_ -= exp_difference;
  }

  // Divides this bignum by divisor, assigning the remainder to this and
  // returning the quotient.
  int divmod_assign(const bigint& divisor) {
    FMT_ASSERT(this != &divisor, "");
    if (compare(*this, divisor) < 0) return 0;
    FMT_ASSERT(divisor.bigits_[divisor.bigits_.size() - 1u] != 0, "");
    align(divisor);
    int quotient = 0;
    do {
      subtract_aligned(divisor);
      ++quotient;
    } while (compare(*this, divisor) >= 0);
    return quotient;
  }
};

```

这段代码定义了一个名为 `round_direction` 的枚举类型，包含三个枚举值：`unknown`、`up` 和 `down`。

接着，定义了一个名为 `get_round_direction` 的函数，接受三个参数：`divisor`、`remainder` 和 `error`。

函数首先检查 `remainder` 是否小于 `divisor`，如果是，则没有 rounding 方向需要计算，返回 `round_direction::unknown`。

接着，函数检查 `error` 是否小于 `divisor` 除以 2，如果是，则 rounding 方向为 down，返回 `round_direction::down`。

最后，函数检查 `remainder` 是否大于 `divisor` 减去 `error`，如果是，则 rounding 方向为 up，返回 `round_direction::up`。

如果 `error` 大于 `divisor` 减去 `remainder`，但是 `error` 小于 `divisor` 除以 2，或者 `remainder` 小于 `divisor` 减去 `error`，或者 `error` 大于 `divisor` 减去 `remainder`，函数将返回 `round_direction::unknown`。


```cpp
enum class round_direction { unknown, up, down };

// Given the divisor (normally a power of 10), the remainder = v % divisor for
// some number v and the error, returns whether v should be rounded up, down, or
// whether the rounding direction can't be determined due to error.
// error should be less than divisor / 2.
inline round_direction get_round_direction(uint64_t divisor, uint64_t remainder,
                                           uint64_t error) {
  FMT_ASSERT(remainder < divisor, "");  // divisor - remainder won't overflow.
  FMT_ASSERT(error < divisor, "");      // divisor - error won't overflow.
  FMT_ASSERT(error < divisor - error, "");  // error * 2 won't overflow.
  // Round down if (remainder + error) * 2 <= divisor.
  if (remainder <= divisor - remainder && error * 2 <= divisor - remainder * 2)
    return round_direction::down;
  // Round up if (remainder - error) * 2 >= divisor.
  if (remainder >= error &&
      remainder - error >= divisor - (remainder - error)) {
    return round_direction::up;
  }
  return round_direction::unknown;
}

```

This is a C++ implementation of the `divmod_integral` function, which is used to return the remainder of a division operation by 10, up to a maximum number of digits. The function takes an integer `n` and produces a string representation of the integral, starting from the result of the first division operation.

The `divmod_integral` function follows these steps:

1. If `n` is equal to zero, the function returns zero as the result.
2. If `n` is not equal to zero, the function starts by dividing `n` by 10 and keeping the result.
3. If `n` is divisible by 10, the function computes the remainder as the result of the division operation.
4. If `n` is not divisible by 10, the function generates digits for the fractional part. This is done repeatedly until the integral is either a divisor or 0.
5. The function returns the maximum number of digits that can be produced.

The `divmod_integral` function is implemented using a combination of mathematical operations, such as bitwise subtraction and bitwise or operation, and loops. The `handler.on_digit` function is used to handle the digits of the integral, which is called when the `divmod_integral` function produces a digit.


```cpp
namespace digits {
enum result {
  more,  // Generate more digits.
  done,  // Done generating digits.
  error  // Digit generation cancelled due to an error.
};
}

// Generates output using the Grisu digit-gen algorithm.
// error: the size of the region (lower, upper) outside of which numbers
// definitely do not round to value (Delta in Grisu3).
template <typename Handler>
FMT_ALWAYS_INLINE digits::result grisu_gen_digits(fp value, uint64_t error,
                                                  int& exp, Handler& handler) {
  const fp one(1ULL << -value.e, value.e);
  // The integral part of scaled value (p1 in Grisu) = value / one. It cannot be
  // zero because it contains a product of two 64-bit numbers with MSB set (due
  // to normalization) - 1, shifted right by at most 60 bits.
  auto integral = static_cast<uint32_t>(value.f >> -one.e);
  FMT_ASSERT(integral != 0, "");
  FMT_ASSERT(integral == value.f >> -one.e, "");
  // The fractional part of scaled value (p2 in Grisu) c = value % one.
  uint64_t fractional = value.f & (one.f - 1);
  exp = count_digits(integral);  // kappa in Grisu.
  // Divide by 10 to prevent overflow.
  auto result = handler.on_start(data::powers_of_10_64[exp - 1] << -one.e,
                                 value.f / 10, error * 10, exp);
  if (result != digits::more) return result;
  // Generate digits for the integral part. This can produce up to 10 digits.
  do {
    uint32_t digit = 0;
    auto divmod_integral = [&](uint32_t divisor) {
      digit = integral / divisor;
      integral %= divisor;
    };
    // This optimization by Milo Yip reduces the number of integer divisions by
    // one per iteration.
    switch (exp) {
    case 10:
      divmod_integral(1000000000);
      break;
    case 9:
      divmod_integral(100000000);
      break;
    case 8:
      divmod_integral(10000000);
      break;
    case 7:
      divmod_integral(1000000);
      break;
    case 6:
      divmod_integral(100000);
      break;
    case 5:
      divmod_integral(10000);
      break;
    case 4:
      divmod_integral(1000);
      break;
    case 3:
      divmod_integral(100);
      break;
    case 2:
      divmod_integral(10);
      break;
    case 1:
      digit = integral;
      integral = 0;
      break;
    default:
      FMT_ASSERT(false, "invalid number of digits");
    }
    --exp;
    auto remainder = (static_cast<uint64_t>(integral) << -one.e) + fractional;
    result = handler.on_digit(static_cast<char>('0' + digit),
                              data::powers_of_10_64[exp] << -one.e, remainder,
                              error, exp, true);
    if (result != digits::more) return result;
  } while (exp > 0);
  // Generate digits for the fractional part.
  for (;;) {
    fractional *= 10;
    error *= 10;
    char digit = static_cast<char>('0' + (fractional >> -one.e));
    fractional &= one.f - 1;
    --exp;
    result = handler.on_digit(digit, one.f, fractional, error, exp, false);
    if (result != digits::more) return result;
  }
}

```

This code appears to be a C++ implementation of a simple IEEE 754 arithmetic formatting library.

It provides a `fmt` function to format a number as a string, and a `digits` function to return the digits of a number.

The `fmt` function takes a number, a precision, and an optional format specifier `fmt...`, and returns a string representation of the number with the specified precision and format specifier.

If the given precision is 0, the function returns the first significant digit of the number.

If the given precision is not 0, the function first formats the number with the precision and then returns the string representation.

The `digits` function takes a number, a divisor, and an optional error flag, and returns the digits of the given number.

If the divisor is unknown, the function returns the string "error".

If the divisor is positive and the number is an integer, the function returns the number of digits in the number.

If the divisor is negative or the number is not an integer, the function returns the string "error".

The function also provides some additional information in the return statement to indicate whether the format specifier has an integral part, if the number is an exponent, or if the divisor has an unknown value.


```cpp
// The fixed precision digit handler.
struct fixed_handler {
  char* buf;
  int size;
  int precision;
  int exp10;
  bool fixed;

  digits::result on_start(uint64_t divisor, uint64_t remainder, uint64_t error,
                          int& exp) {
    // Non-fixed formats require at least one digit and no precision adjustment.
    if (!fixed) return digits::more;
    // Adjust fixed precision by exponent because it is relative to decimal
    // point.
    precision += exp + exp10;
    // Check if precision is satisfied just by leading zeros, e.g.
    // format("{:.2f}", 0.001) gives "0.00" without generating any digits.
    if (precision > 0) return digits::more;
    if (precision < 0) return digits::done;
    auto dir = get_round_direction(divisor, remainder, error);
    if (dir == round_direction::unknown) return digits::error;
    buf[size++] = dir == round_direction::up ? '1' : '0';
    return digits::done;
  }

  digits::result on_digit(char digit, uint64_t divisor, uint64_t remainder,
                          uint64_t error, int, bool integral) {
    FMT_ASSERT(remainder < divisor, "");
    buf[size++] = digit;
    if (!integral && error >= remainder) return digits::error;
    if (size < precision) return digits::more;
    if (!integral) {
      // Check if error * 2 < divisor with overflow prevention.
      // The check is not needed for the integral part because error = 1
      // and divisor > (1 << 32) there.
      if (error >= divisor || error >= divisor - error) return digits::error;
    } else {
      FMT_ASSERT(error == 1 && divisor > 2, "");
    }
    auto dir = get_round_direction(divisor, remainder, error);
    if (dir != round_direction::up)
      return dir == round_direction::down ? digits::done : digits::error;
    ++buf[size - 1];
    for (int i = size - 1; i > 0 && buf[i] > '9'; --i) {
      buf[i] = '0';
      ++buf[i - 1];
    }
    if (buf[0] > '9') {
      buf[0] = '1';
      if (fixed) buf[size++] = '0';
      else ++exp10;
    }
    return digits::done;
  }
};

```

该代码实现了一个名为Dragonbox的算法，用于计算两个64位无符号整数的乘积的128位结果。该算法采用分治策略，并使用了几个安全缓冲器（FMT_SAFEBUFFERS）来确保数据的正确性。

具体来说，该算法包括以下几个步骤：

1. 将两个64位无符号整数x和y的值分别存储在变量中。
2. 如果计算机硬件支持8位整数，那么将x和y的值直接进行乘法运算，并输出结果。
3. 如果定义了_MSC_VER和_M_X64，那么使用int128类型来存储结果，并使用umul128函数来实现64位乘法运算。
4. 如果硬件不支持上述两种情况，那么需要实现一个名为FMT_NOEXCEPT的函数来计算结果。该函数使用了一个64位无符号整型变量result，该变量包含两个8位整型变量ac和bd，以及一个32位无符号整型变量intermediate。计算结果的步骤如下：

 a. 将x的高32位和y的高32位分别存储在ac和bd中。

   b. 将x的低32位和y的低32位与一个32位无符号整型mask进行异或运算，并分别存储在a和b中。

   c. 将ac和bd的值分别乘以a和mask，并将结果存储在intermediate中。

   d. 将bc和mask的值分别乘以b和mask，并将结果存储在intermediate中。

   e. 将intermediate的高32位和低32位分别加上32位无符号整型variable_intermediate的值，即intermediate_low和intermediate_high，并将结果存储在intermediate中。

   f. 最终的结果就是intermediate的值。

3. 返回计算得到的128位无符号整型结果。


```cpp
// Implementation of Dragonbox algorithm: https://github.com/jk-jeon/dragonbox.
namespace dragonbox {
// Computes 128-bit result of multiplication of two 64-bit unsigned integers.
FMT_SAFEBUFFERS inline uint128_wrapper umul128(uint64_t x,
                                               uint64_t y) FMT_NOEXCEPT {
#if FMT_USE_INT128
  return static_cast<uint128_t>(x) * static_cast<uint128_t>(y);
#elif defined(_MSC_VER) && defined(_M_X64)
  uint128_wrapper result;
  result.low_ = _umul128(x, y, &result.high_);
  return result;
#else
  const uint64_t mask = (uint64_t(1) << 32) - uint64_t(1);

  uint64_t a = x >> 32;
  uint64_t b = x & mask;
  uint64_t c = y >> 32;
  uint64_t d = y & mask;

  uint64_t ac = a * c;
  uint64_t bc = b * c;
  uint64_t ad = a * d;
  uint64_t bd = b * d;

  uint64_t intermediate = (bd >> 32) + (ad & mask) + (bc & mask);

  return {ac + (intermediate >> 32) + (ad >> 32) + (bc >> 32),
          (intermediate << 32) + (bd & mask)};
```

这段代码定义了一个名为umul128_upper64的函数，用于计算两个64位无符号整数x和y的upper 64位异或。

函数实现为：

```cpp
#include <cstdint>

template<>
uint64_t umul128_upper64<uint64_t>(uint64_t x,
                                                     uint64_t y) FMT_NOEXCEPT {
 #if FMT_USE_INT128
 auto p = static_cast<uint128_t>(x) * static_cast<uint128_t>(y);
 return static_cast<uint64_t>(p >> 64);
 #elif defined(_MSC_VER) && defined(_M_X64)
 return __umulh(x, y);
 #else
 return umul128(x, y).high();
#endif
}
```

该函数首先通过传入x和y，计算出x和y的upper 64位异或，然后通过一些特殊情况的检查，选择正确的实现方式，最终返回结果。

具体实现方式如下：

当使用缓存安全类型(FMT_USE_INT128)时，函数实现为：

```cpp
#include <cstdint>

template<>
uint64_t umul128_upper64<uint64_t>(uint64_t x,
                                                     uint64_t y) FMT_NOEXCEPT {
 auto p = static_cast<uint128_t>(x) * static_cast<uint128_t>(y);
 return static_cast<uint64_t>(p >> 64);
}
```

在这种情况下，函数直接将x和y的乘积作为输入，并输出它们的upper 64位异或结果。

当使用支持宏定义的编译器(FMT_USE_HEADER)时，函数实现为：

```cpp
#include <cstdint>

#ifdef _MSC_VER) && defined(_M_X64)
 // ...以 _MSC_VER 和 _M_X64 为例，这里有一些自定义的宏定义...
#endif

template<>
uint64_t umul128_upper64<uint64_t>(uint64_t x,
                                                     uint64_t y) FMT_NOEXCEPT {
 #if FMT_USE_INT128
 auto p = static_cast<uint128_t>(x) * static_cast<uint128_t>(y);
 return static_cast<uint64_t>(p >> 64);
 #elif defined(_MSC_VER) && defined(_M_X64)
 return __umulh(x, y);
 #else
 return umul128(x, y).high();
#endif
}
```

在这种情况下，函数首先通过 _MSC_VER 和 _M_X64 宏定义检查，确定是否支持底层类型，如果支持，则函数实现与前面的函数一样，否则需要根据具体情况来实现。

另外，该函数还支持一种带参数的函数重载，允许用户自定义函数实现，但需要保证函数的签名与函数声明的一致性。


```cpp
#endif
}

// Computes upper 64 bits of multiplication of two 64-bit unsigned integers.
FMT_SAFEBUFFERS inline uint64_t umul128_upper64(uint64_t x,
                                                uint64_t y) FMT_NOEXCEPT {
#if FMT_USE_INT128
  auto p = static_cast<uint128_t>(x) * static_cast<uint128_t>(y);
  return static_cast<uint64_t>(p >> 64);
#elif defined(_MSC_VER) && defined(_M_X64)
  return __umulh(x, y);
#else
  return umul128(x, y).high();
#endif
}

```

这两段代码定义了两个名为"umul192_upper64"和"umul96_upper32"的函数，用于计算一个64位无符号整数和一个128位无符号整数之间的乘积的 upper 64 位和upper 32 位。

函数umul192_upper64接收两个无符号整数x和y，首先使用umul128函数将x和y的高位相乘，然后将两个结果相加并获取高位，最后返回高位的结果。

函数umul96_upper32接收一个32位无符号整数x和一个64位无符号整数y，首先使用umul128函数将x和y的高位相乘，然后将两个结果相加并获取高位，最后将高位结果转换为32位整数并返回。


```cpp
// Computes upper 64 bits of multiplication of a 64-bit unsigned integer and a
// 128-bit unsigned integer.
FMT_SAFEBUFFERS inline uint64_t umul192_upper64(uint64_t x, uint128_wrapper y)
    FMT_NOEXCEPT {
  uint128_wrapper g0 = umul128(x, y.high());
  g0 += umul128_upper64(x, y.low());
  return g0.high();
}

// Computes upper 32 bits of multiplication of a 32-bit unsigned integer and a
// 64-bit unsigned integer.
inline uint32_t umul96_upper32(uint32_t x, uint64_t y) FMT_NOEXCEPT {
  return static_cast<uint32_t>(umul128_upper64(x, y));
}

```

这两段代码定义了两个名为"umul192_middle64"和"umul96_lower64"的函数，用于计算一个64位无符号整数和一个128位无符号整数中的中间64位和低64位。

这两个函数的实现都使用了FMT_NOEXCEPT类型，即输出结果需要进行类型检查，且在函数内部对输入参数进行了取整和乘法运算，以保证结果的正确性。

具体来说，这两个函数的实现如下：

umul192_middle64函数接收两个uint64_t类型的参数x和y，并计算出x和y的并作为g01，然后计算出umul128_upper64(x, y.low())作为g10，最后将g01和g10相加得到结果。

umul96_lower64函数接收一个uint32_t类型的参数x和一个uint64_t类型的参数y，并直接将x乘以y得到结果。

这两个函数都遵循了FMT_NOEXCEPT类型检查，意味着程序在编译时就能够检测出输入参数的类型是否与函数要求的一致，如果不一致，则会产生错误。


```cpp
// Computes middle 64 bits of multiplication of a 64-bit unsigned integer and a
// 128-bit unsigned integer.
FMT_SAFEBUFFERS inline uint64_t umul192_middle64(uint64_t x, uint128_wrapper y)
    FMT_NOEXCEPT {
  uint64_t g01 = x * y.high();
  uint64_t g10 = umul128_upper64(x, y.low());
  return g01 + g10;
}

// Computes lower 64 bits of multiplication of a 32-bit unsigned integer and a
// 64-bit unsigned integer.
inline uint64_t umul96_lower64(uint32_t x, uint64_t y) FMT_NOEXCEPT {
  return x * y;
}

```

这段代码是一个计算 $log_{10}(2)$ 在给定区间内的结果的函数。函数接收一个整数变量 $e$，并返回一个整数。它的实现基于两个预定义的函数：floor_log10_pow2 和 floor_log2_pow10。

floor_log10_pow2 的实现采用了这种方法，通过将 $2$ 的幂逐步逼近 $10$，并计算相应的指数值，然后逐步逼近 $log_{10}(2)$。具体而言，它将 $2$ 的幂逐步逼近 $10$，然后计算 $log_{10}(2)$ 在 $64$ 位整数下的偏移量，最后将其乘以相应的系数，再将结果逐步逼近 $log_{10}(2)$。

floor_log2_pow10 的实现是基于对 $2$ 的幂逐步逼近 $10$ 的方法，通过计算相应的指数值。它将 $2$ 的幂逐步逼近 $10$，然后计算 $log_{10}(2)$ 在 $64$ 位整数下的偏移量，最后将其乘以相应的系数，再将结果逐步逼近 $log_{10}(2)$。

这段代码的作用就是计算 $log_{10}(2)$ 在给定区间内的结果。


```cpp
// Computes floor(log10(pow(2, e))) for e in [-1700, 1700] using the method from
// https://fmt.dev/papers/Grisu-Exact.pdf#page=5, section 3.4.
inline int floor_log10_pow2(int e) FMT_NOEXCEPT {
  FMT_ASSERT(e <= 1700 && e >= -1700, "too large exponent");
  const int shift = 22;
  return (e * static_cast<int>(data::log10_2_significand >> (64 - shift))) >>
         shift;
}

// Various fast log computations.
inline int floor_log2_pow10(int e) FMT_NOEXCEPT {
  FMT_ASSERT(e <= 1233 && e >= -1233, "too large exponent");
  const uint64_t log2_10_integer_part = 3;
  const uint64_t log2_10_fractional_digits = 0x5269e12f346e2bf9;
  const int shift_amount = 19;
  return (e * static_cast<int>(
                  (log2_10_integer_part << shift_amount) |
                  (log2_10_fractional_digits >> (64 - shift_amount)))) >>
         shift_amount;
}
```

这段代码是一个C++函数，名为"floor_log10_pow2_minus_log10_4_over_3"。它计算一个整数e的10的多少次方，并返回一个布尔值，表示e是否是2的整数次幂的因数。

函数内部首先检查e是否小于等于1700且大于等于-1700，因为这样会抛出"too large exponent"的错误。然后计算出log10_4_over_3的值，这是一个浮点数。

接着计算一个偏移量shift_amount，通过比较log10_2_significand与log10_4_over_3_fractional_digits的大小，可以得到这个偏移量。

最后，将e乘以log10_2_significand除以(64-shift_amount)，再取对shift_amount，并取反，得到一个布尔值，表示e是否是2的整数次幂的因数。


```cpp
inline int floor_log10_pow2_minus_log10_4_over_3(int e) FMT_NOEXCEPT {
  FMT_ASSERT(e <= 1700 && e >= -1700, "too large exponent");
  const uint64_t log10_4_over_3_fractional_digits = 0x1ffbfc2bbc780375;
  const int shift_amount = 22;
  return (e * static_cast<int>(data::log10_2_significand >>
                               (64 - shift_amount)) -
          static_cast<int>(log10_4_over_3_fractional_digits >>
                           (64 - shift_amount))) >>
         shift_amount;
}

// Returns true iff x is divisible by pow(2, exp).
inline bool divisible_by_power_of_2(uint32_t x, int exp) FMT_NOEXCEPT {
  FMT_ASSERT(exp >= 1, "");
  FMT_ASSERT(x != 0, "");
```



这两段代码分别定义了两个判断条件的函数，一个判断第一个参数 $x$ 是否满足 $exp$ 的二进制表示中是否包含至少 $exp$ 个 $1$，另一个判断 $x$ 是否可以被 $2$ 的幂次方 $2^exp$ 整除。

第一个判断条件可以理解为：如果 $x$ 的二进制表示中至少 $exp$ 个 $1$，那么 $x$ 至少包含 $exp$ 个 $1$ 的因子，$x$ 满足 $exp$ 位的 $1$ 的个数 $=$ $exp$。

第二个判断条件可以理解为：如果 $x$ 包含 $2$ 的幂次方 $2^exp$ 整除，那么 $x$ 满足 $x$ 能够被 $2$ 的幂次方 $2^exp$ 整除，即 $x$ 满足 $x$ 是 $2$ 的倍数。

两个判断条件的实现都使用了无后缀编码(exp)、有后缀编码(uint32_t)、整数类型(uint64_t)。


```cpp
#ifdef FMT_BUILTIN_CTZ
  return FMT_BUILTIN_CTZ(x) >= exp;
#else
  return exp < num_bits<uint32_t>() && x == ((x >> exp) << exp);
#endif
}
inline bool divisible_by_power_of_2(uint64_t x, int exp) FMT_NOEXCEPT {
  FMT_ASSERT(exp >= 1, "");
  FMT_ASSERT(x != 0, "");
#ifdef FMT_BUILTIN_CTZLL
  return FMT_BUILTIN_CTZLL(x) >= exp;
#else
  return (exp < num_bits<uint64_t>()) && x == ((x >> exp) << exp);
#endif
}

```

这段代码是一个C++函数，名为`divisible_by_power_of_5`，它判断一个整数`x`是否可以被5的给定指数`exp`整除。如果`x`可以被5的给定指数整除，那么函数返回`true`，否则返回`false`。

函数体中包含两个实现，分别针对32位和64位整数。这两个实现都使用了同一个判断条件：`exp`必须小于等于10，因为如果`exp`大于10，那么测试将无法通过编译。

具体来说，函数首先查找一个名为`data::divtest_table_for_pow5_32`的表格，它包含一个32位整数（exp为32）的测试值。然后，函数计算`x`乘以`data::divtest_table_for_pow5_32[exp].mod_inv`，并将其与`data::divtest_table_for_pow5_32[exp].max_quotient`进行比较。如果`x`可以被5的给定指数整除，那么函数返回`true`，否则返回`false`。

对于64位整数，函数的实现与32位实现类似，只是使用了不同的整数类型。


```cpp
// Returns true iff x is divisible by pow(5, exp).
inline bool divisible_by_power_of_5(uint32_t x, int exp) FMT_NOEXCEPT {
  FMT_ASSERT(exp <= 10, "too large exponent");
  return x * data::divtest_table_for_pow5_32[exp].mod_inv <=
         data::divtest_table_for_pow5_32[exp].max_quotient;
}
inline bool divisible_by_power_of_5(uint64_t x, int exp) FMT_NOEXCEPT {
  FMT_ASSERT(exp <= 23, "too large exponent");
  return x * data::divtest_table_for_pow5_64[exp].mod_inv <=
         data::divtest_table_for_pow5_64[exp].max_quotient;
}

// Replaces n by floor(n / pow(5, N)) returning true if and only if n is
// divisible by pow(5, N).
// Precondition: n <= 2 * pow(5, N + 1).
```

这段代码是一个C++模板函数，名为`check_divisibility_and_divide_by_pow5`，其参数是一个整数N。

该函数的作用是检查给定的整数N是否可以被16的幂整除，并且可以被8位整数整除。为了实现这个功能，函数使用了一个constexpr结构体数组infos，其中包含了多个测试用例。

具体来说，函数首先定义了一个constexpr变量info，该变量使用infos数组中第N-1个元素的信息来计算。然后，函数将N乘以info.magic_number，这个magic_number是一个16位的整数，用于表示在二进制中N对16取模的结果。

接下来，函数计算了一个uint32_t比较掩码，它使用info.bits_for_comparison位数的二进制补码，来表示对N的最高位进行比较的位数。这个比较掩码的值将比info.threshold小，因为info.threshold是info.bits_for_comparison的最小值，但这个最小值是8，而不是16。

接着，函数将N左移info.shift_amount位，这个位数的值表示info.bits_for_comparison中从低位到高位，每4位一组，有多少个完整的组。然后，函数判断给定的整数N是否小于等于info.threshold，如果是，函数返回true，否则返回false。

最后，函数使用n & comparison_mask来检查给定的整数N是否可以被16的幂整除。如果n包含这个掩码，说明n可以被16的幂整除，函数返回true。如果n不包含这个掩码，说明n不能被16的幂整除，函数返回false。


```cpp
template <int N>
bool check_divisibility_and_divide_by_pow5(uint32_t& n) FMT_NOEXCEPT {
  static constexpr struct {
    uint32_t magic_number;
    int bits_for_comparison;
    uint32_t threshold;
    int shift_amount;
  } infos[] = {{0xcccd, 16, 0x3333, 18}, {0xa429, 8, 0x0a, 20}};
  constexpr auto info = infos[N - 1];
  n *= info.magic_number;
  const uint32_t comparison_mask = (1u << info.bits_for_comparison) - 1;
  bool result = (n & comparison_mask) <= info.threshold;
  n >>= info.shift_amount;
  return result;
}

```

这两段代码是在计算 `floor(n / pow(10, N))`，其中 `n` 是 `int` 类型，`N` 是 `int` 类型，并且满足 `n <= pow(10, N + 1)` 这个条件。

具体来说，这两段代码首先定义了一个模板类 `small_division_by_pow10`，其中包含一个结构体 `infos`，它有三个成员：`magic_number`、`shift_amount` 和 `divisor_times_10`，分别对应于 `pow(10, N)` 中底数 `10`、指数 `N` 和除数 `10`。

然后，代码中定义了一个常量 `info`，它是一个来自于 `infos` 数组的结构体，其成员包括 `divisor_times_10` 和 `shift_amount`，这两个成员是通过 `infos[N - 1].divisor_times_10` 和 `infos[N - 1].shift_amount` 来计算得到的。

接着，代码中定义了一个函数 `divide_by_10_to_kappa_plus_1`，它接收一个 `int` 类型的 `n` 作为参数，并返回 `n` 除以 `float_info<float>::big_divisor` 所得到的结果，其中 `float_info` 是另一个模板类，这个类定义了一个 `big_divisor` 成员，用于计算 `float` 类型的浮点数的分母。

最后，代码中计算 `floor(n / pow(10, N))`，这个结果就是 `n` 除以 `pow(10, N)` 得到的结果，然后再将结果取整，得到的结果就是 `floor(n / pow(10, N))` 的值。


```cpp
// Computes floor(n / pow(10, N)) for small n and N.
// Precondition: n <= pow(10, N + 1).
template <int N> uint32_t small_division_by_pow10(uint32_t n) FMT_NOEXCEPT {
  static constexpr struct {
    uint32_t magic_number;
    int shift_amount;
    uint32_t divisor_times_10;
  } infos[] = {{0xcccd, 19, 100}, {0xa3d8, 22, 1000}};
  constexpr auto info = infos[N - 1];
  FMT_ASSERT(n <= info.divisor_times_10, "n is too large");
  return n * info.magic_number >> info.shift_amount;
}

// Computes floor(n / 10^(kappa + 1)) (float)
inline uint32_t divide_by_10_to_kappa_plus_1(uint32_t n) FMT_NOEXCEPT {
  return n / float_info<float>::big_divisor;
}
```

This code appears to define a class called "Cache" which appears to be a cache of unsigned integers. It also defines a number of helper functions for working with the cache, including the ability to compute the beta minus value, the endpoints of a "short interval" carry, and the rounding of a number up.

The "short interval" carry refers to the practice of carrying a small number of high-order bits of a value (such as a timestamp or a counter) around in memory for fast access, rather than carrying the full value. This can be useful in cases where the value is too large to fit in regular memory locations, but still needs to be accessed quickly.

The "endpoints" of a "short interval" carry refer to the values that the cache should return when a given value is requested, based on the "short interval" carry used to generate it. In this case, the endpoint of a "short interval" carry is the value plus the "beta minus" value, multiplied by 2, to ensure that it is rounded up to the nearest integer.


```cpp
// Computes floor(n / 10^(kappa + 1)) (double)
inline uint64_t divide_by_10_to_kappa_plus_1(uint64_t n) FMT_NOEXCEPT {
  return umul128_upper64(n, 0x83126e978d4fdf3c) >> 9;
}

// Various subroutines using pow10 cache
template <class T> struct cache_accessor;

template <> struct cache_accessor<float> {
  using carrier_uint = float_info<float>::carrier_uint;
  using cache_entry_type = uint64_t;

  static uint64_t get_cached_power(int k) FMT_NOEXCEPT {
    FMT_ASSERT(k >= float_info<float>::min_k && k <= float_info<float>::max_k,
               "k is out of range");
    return data::dragonbox_pow10_significands_64[k - float_info<float>::min_k];
  }

  static carrier_uint compute_mul(carrier_uint u,
                                  const cache_entry_type& cache) FMT_NOEXCEPT {
    return umul96_upper32(u, cache);
  }

  static uint32_t compute_delta(const cache_entry_type& cache,
                                int beta_minus_1) FMT_NOEXCEPT {
    return static_cast<uint32_t>(cache >> (64 - 1 - beta_minus_1));
  }

  static bool compute_mul_parity(carrier_uint two_f,
                                 const cache_entry_type& cache,
                                 int beta_minus_1) FMT_NOEXCEPT {
    FMT_ASSERT(beta_minus_1 >= 1, "");
    FMT_ASSERT(beta_minus_1 < 64, "");

    return ((umul96_lower64(two_f, cache) >> (64 - beta_minus_1)) & 1) != 0;
  }

  static carrier_uint compute_left_endpoint_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return static_cast<carrier_uint>(
        (cache - (cache >> (float_info<float>::significand_bits + 2))) >>
        (64 - float_info<float>::significand_bits - 1 - beta_minus_1));
  }

  static carrier_uint compute_right_endpoint_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return static_cast<carrier_uint>(
        (cache + (cache >> (float_info<float>::significand_bits + 1))) >>
        (64 - float_info<float>::significand_bits - 1 - beta_minus_1));
  }

  static carrier_uint compute_round_up_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return (static_cast<carrier_uint>(
                cache >>
                (64 - float_info<float>::significand_bits - 2 - beta_minus_1)) +
            1) /
           2;
  }
};

```

This is a function that assumes the implementation of a 128-bit cache that follows the Dragon Box power 10 signficands (base-2, base-3, base-4, ...) data structure. The base cache is initialized with data::dragonbox_pow10_significands_128, which is a 128-bit integer that follows the Dragon Box power 10 signficands.

The function has several helper functions:

* data::dragonbox_pow10_recovery_errors: This function returns an error code for dragon box pow 10 recomputation, based on the given error index (index) and the number of digits in the original data.
* data::powers_of_5_64: This function returns a table of the powers of 5 up to 64, which can be used for calculating the number of bit-shifts needed for a given number.
* data::dragonbox_pow10_significands_512: This function returns a 512-bit integer that follows the Dragon Box power 10 signficands (base-2, base-3, base-4, ...) data structure.

The main function takes as input a single integer offset, which is used to shift the base cache to the desired position. The function then checks if the base cache has been initialized and if the offset is zero. If the base cache has not been initialized, the function computes the required amount of bit-shift and recovers the base cache. If the base cache has already been initialized, the function checks if the specified offset is valid and performs the required bit-shift. Finally, the function returns the recovered base cache.

Note that this implementation assumes that the base cache is 128 bits in size, and that the data is in the range of [-512, 511].


```cpp
template <> struct cache_accessor<double> {
  using carrier_uint = float_info<double>::carrier_uint;
  using cache_entry_type = uint128_wrapper;

  static uint128_wrapper get_cached_power(int k) FMT_NOEXCEPT {
    FMT_ASSERT(k >= float_info<double>::min_k && k <= float_info<double>::max_k,
               "k is out of range");

#if FMT_USE_FULL_CACHE_DRAGONBOX
    return data::dragonbox_pow10_significands_128[k -
                                                  float_info<double>::min_k];
#else
    static const int compression_ratio = 27;

    // Compute base index.
    int cache_index = (k - float_info<double>::min_k) / compression_ratio;
    int kb = cache_index * compression_ratio + float_info<double>::min_k;
    int offset = k - kb;

    // Get base cache.
    uint128_wrapper base_cache =
        data::dragonbox_pow10_significands_128[cache_index];
    if (offset == 0) return base_cache;

    // Compute the required amount of bit-shift.
    int alpha = floor_log2_pow10(kb + offset) - floor_log2_pow10(kb) - offset;
    FMT_ASSERT(alpha > 0 && alpha < 64, "shifting error detected");

    // Try to recover the real cache.
    uint64_t pow5 = data::powers_of_5_64[offset];
    uint128_wrapper recovered_cache = umul128(base_cache.high(), pow5);
    uint128_wrapper middle_low =
        umul128(base_cache.low() - (kb < 0 ? 1 : 0), pow5);

    recovered_cache += middle_low.high();

    uint64_t high_to_middle = recovered_cache.high() << (64 - alpha);
    uint64_t middle_to_low = recovered_cache.low() << (64 - alpha);

    recovered_cache =
        uint128_wrapper{(recovered_cache.low() >> alpha) | high_to_middle,
                        ((middle_low.low() >> alpha) | middle_to_low)};

    if (kb < 0) recovered_cache += 1;

    // Get error.
    int error_idx = (k - float_info<double>::min_k) / 16;
    uint32_t error = (data::dragonbox_pow10_recovery_errors[error_idx] >>
                      ((k - float_info<double>::min_k) % 16) * 2) &
                     0x3;

    // Add the error back.
    FMT_ASSERT(recovered_cache.low() + error >= recovered_cache.low(), "");
    return {recovered_cache.high(), recovered_cache.low() + error};
```

This is a Rust implementation of the FMT (Fused-Vector Multiply-Accumulate) standard, which is designed to provide multiplication and accumulation capabilities for fixed-point numbers with varying bit widths.

FMT implementation for 32-bit floating-point numbers with different bit widths:
```cpprust
// Implementation of the FMT_NOEXCEPT macro
// in the ff32_fp structure

namespace ff {
 using fp_bits = std::int32_t;

 //..

 static inline fp_bits upper_bit_pos(fp_bits high) {
   return (high & 0x80000000) ? fp_bits(19) : fp_bits(18);
 }

 static inline fp_bits lower_bit_pos(fp_bits low) {
   return (low & 0x7fffffff) ? fp_bits(43) : fp_bits(42);
 }

 //..

 // Compute the FMT_NOEXCEPT implementation of theNO/NOI macro
 // for 32-bit floating-point numbers with different bit widths.
 //
 // Compile this code snippet to an unsigned long.
 //
 // (editable)
 constexpr unsigned long NO_EXPECTED_CAUSES =
     static_cast<unsigned long>(
         UINT32(0)
         ^ (float_info<double>::bits.depth_s > 32)
         ^ (float_info<double>::bits.no_expand)
         ^ (float_info<double>::bits.壶建立)
         ^ (float_info<double>::bits.normalized_mul)
         ^ (float_info<double>::bits.s sign)
         ^ (float_info<double>::bits.res_cal)
         ^ (float_info<double>::bits.i 壶建立)
         ^ (float_info<double>::bits.sat_s)
         ^ (float_info<double>::bits.res_sat)
         ^ (float_info<double>::bits.res_i)
         ^ (float_info<double>::bits.壶牛津)
         ^ (float_info<double>::bits.float_in_f)
         ^ (float_info<double>::bits.壶回拨)
         ^ (float_info<double>::bits.res_lo)
         ^ (float_info<double>::bits.res_lcond)
         ^ (float_info<double>::bits.res_mul)
         ^ (float_info<double>::bits.res_壶建)
         ^ (float_info<double>::bits.res_壶刻)
         ^ (float_info<double>::bits.res_sat_s)
         ^ (float_info<double>::bits.res_sat)
         ^ (float_info<double>::bits.res_res_i)
         ^ (float_info<double>::bits.res_sat_i)
         ^ (float_info<double>::bits.res_sat)
         ^ (float_info<double>::bits.res_i)
         ^ (float_info<double>::bits.res_f)
         ^ (float_info<double>::bits.res_i_sat)
         ^ (float_info<double>::bits.res_i_sat_l)
         ^ (float_info<double>::bits.res_i_sat_l1)
         ^ (float_info<double>::bits.res_i_sat_l2)
         ^ 0);

 // Add 32 bits to the fp_bits field to represent the
 // signed multiplication/accumulation capability.
 constexpr unsigned long NO_EXPECTED_CAUSES_32 =
     static_cast<unsigned long>(
         UINT32(0)
         ^ (float_info<double>::bits.depth_s > 32)
         ^ (float_info<double>::bits.no_expand)
         ^ (float_info<double>::bits.壶建立)
         ^ (float_info<double>::bits.normalized_mul)
         ^ (float_info<double>::bits.s sign)
         ^ (float_info<double>::bits.res_cal)
         ^ (float_info<double>::bits.i 壶建立)
         ^ (float_info<double>::bits.sat_s)
         ^ (float_info<double>::bits.res_sat)
         ^ (float_info<double>::bits.res_res_i)
         ^ (float_info<double>::bits.res_sat_i)
         ^ (float_info<double>::bits.res_f)
         ^ (float_info<double>::bits.res_i_sat)
         ^ (float_info<double>::bits.res_i_sat_l)
         ^ (float_info<double>::bits.res_i_sat_l1)
         ^ (float_info<double>::bits.res_i_sat_l2)
         ^ 0);

 // Compile this code snippet to an unsigned long.
 //
 // (editable)
 constexpr unsigned long NO_EXPECTED_CAUSES =
     static_cast<unsigned long>(
         UINT32(0)
         ^ (float_info<double>::bits.depth_s > 32)
         ^ (float_info<double>::bits.no_expand)
         ^ (float_info<double>::bits.壶建立)
         ^ (float_info<double>::bits.normalized_mul)
         ^ (float_info<double>::bits.s sign)
         ^ (float_info<double>::bits.res_cal)
         ^ (float_info<double>::bits.i 壶建立)
         ^ (float_info<double>::bits.sat_s)
         ^ (float_info<double>::bits.res_sat)
         ^ (float_info<double>::bits.res_res_i)
         ^ (float_info<double>::bits.res_sat_i)
         ^ (float_info<double>::bits.res_f)
         ^ (float_info<double>::bits.res_i_sat)
         ^ (float_info<double>::bits.res_i_sat_l)
         ^ (float_info


```
#endif
  }

  static carrier_uint compute_mul(carrier_uint u,
                                  const cache_entry_type& cache) FMT_NOEXCEPT {
    return umul192_upper64(u, cache);
  }

  static uint32_t compute_delta(cache_entry_type const& cache,
                                int beta_minus_1) FMT_NOEXCEPT {
    return static_cast<uint32_t>(cache.high() >> (64 - 1 - beta_minus_1));
  }

  static bool compute_mul_parity(carrier_uint two_f,
                                 const cache_entry_type& cache,
                                 int beta_minus_1) FMT_NOEXCEPT {
    FMT_ASSERT(beta_minus_1 >= 1, "");
    FMT_ASSERT(beta_minus_1 < 64, "");

    return ((umul192_middle64(two_f, cache) >> (64 - beta_minus_1)) & 1) != 0;
  }

  static carrier_uint compute_left_endpoint_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return (cache.high() -
            (cache.high() >> (float_info<double>::significand_bits + 2))) >>
           (64 - float_info<double>::significand_bits - 1 - beta_minus_1);
  }

  static carrier_uint compute_right_endpoint_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return (cache.high() +
            (cache.high() >> (float_info<double>::significand_bits + 1))) >>
           (64 - float_info<double>::significand_bits - 1 - beta_minus_1);
  }

  static carrier_uint compute_round_up_for_shorter_interval_case(
      const cache_entry_type& cache, int beta_minus_1) FMT_NOEXCEPT {
    return ((cache.high() >>
             (64 - float_info<double>::significand_bits - 2 - beta_minus_1)) +
            1) /
           2;
  }
};

```cpp

这段代码是关于数学中关于整数和浮点数范围检查的模板函数。

这段代码检查两个整数变量exponent和two_f，并判断它们是否属于某个特定的区间。第一个检查条件是指数是否大于某个浮点数类型的下界left_endpoint_lower_threshold，第二个检查条件是指数是否小于某个浮点数类型的上界lower_endpoint_upper_threshold。如果两个条件都满足，那么这两个整数变量就属于该区间。

具体来说，代码首先定义了一个模板类T，其中T是一个整数或浮点数。接着，在两个模板类中分别定义了is_left_endpoint_integer_shorter_interval和is_endpoint_integer函数，它们的具体实现就是检查指数是否在特定的区间内。

这两个函数使用了float_info<T>中的carrier_uint类型，该类型被用来存储整数或浮点数的值。然后，通过比较整数和浮点数之间的下界和上界，来判断整数变量exponent和浮点数变量two_f是否在同一个区间内。如果两个条件都满足，那么这两个整数变量就属于该区间，否则返回false或true。

整数检查函数is_endpoint_integer函数的具体实现是，如果整数变量exponent小于float_info<T>::case_fc_pm_half_lower_threshold，就返回false。如果整数变量exponent大于float_info<T>::case_fc_pm_half_upper_threshold，就返回true。如果整数变量exponent等于float_info<T>::divisibility_check_by_5_threshold，就返回false。

divisible_by_power_of_5函数的具体实现是，通过判断整数变量two_f是否可以被5的整数次幂整除，来判断整数变量exponent是否在float_info<T>::case_fc_pm_half_upper_threshold和float_info<T>::case_fc_pm_half_lower_threshold区间内。


```
// Various integer checks
template <class T>
bool is_left_endpoint_integer_shorter_interval(int exponent) FMT_NOEXCEPT {
  return exponent >=
             float_info<
                 T>::case_shorter_interval_left_endpoint_lower_threshold &&
         exponent <=
             float_info<T>::case_shorter_interval_left_endpoint_upper_threshold;
}
template <class T>
bool is_endpoint_integer(typename float_info<T>::carrier_uint two_f,
                         int exponent, int minus_k) FMT_NOEXCEPT {
  if (exponent < float_info<T>::case_fc_pm_half_lower_threshold) return false;
  // For k >= 0.
  if (exponent <= float_info<T>::case_fc_pm_half_upper_threshold) return true;
  // For k < 0.
  if (exponent > float_info<T>::divisibility_check_by_5_threshold) return false;
  return divisible_by_power_of_5(two_f, minus_k);
}

```cpp

这段代码定义了一个名为 is_center_integer 的模板函数，用于检查两个整数的中心是否为整数。模板函数有两个参数：两个整数 two_f 和整数 exponent；还有两个参数：一个整数 minus_k，用于指定检查除以 5 和 2 的余数。函数判断两个整数的小数点后是否有整数位，如果有，就检查整数位是否大于 float_info<T>::case_fc_upper_threshold 和 float_info<T>::case_fc_lower_threshold，即判断整数位是否大于等于 float_info<T>::Case_FC_LOWER_THRESHOLD。如果整数位大于 float_info<T>::Case_FC_LOWER_THRESHOLD，就返回 false；否则，继续判断整数位是否大于 float_info<T>::divisibility_check_by_5_threshold，即判断整数位是否大于 5。如果是，就调用 remove_trailing_zeros 函数，该函数会删除整数 n 中的 trailing zeros。最后，返回整数 n 中 trailing zeros 的个数。


```
template <class T>
bool is_center_integer(typename float_info<T>::carrier_uint two_f, int exponent,
                       int minus_k) FMT_NOEXCEPT {
  // Exponent for 5 is negative.
  if (exponent > float_info<T>::divisibility_check_by_5_threshold) return false;
  if (exponent > float_info<T>::case_fc_upper_threshold)
    return divisible_by_power_of_5(two_f, minus_k);
  // Both exponents are nonnegative.
  if (exponent >= float_info<T>::case_fc_lower_threshold) return true;
  // Exponent for 2 is negative.
  return divisible_by_power_of_2(two_f, minus_k - exponent + 1);
}

// Remove trailing zeros from n and return the number of zeros removed (float)
FMT_ALWAYS_INLINE int remove_trailing_zeros(uint32_t& n) FMT_NOEXCEPT {
```cpp

这段代码是一个C语言函数，名为“fmt_builtin_ctz”。它计算一个整数n的LU分解树中，从根节点到叶节点的最长的路径的LU大小。LU分解树是将一个整数n分解为若干个较小的整数的树，其中每个较小的整数称为“环节”。

具体来说，这段代码的作用如下：

1. 如果n已经定义为FMT_BUILTIN_CTZ（即预编译函数），则直接返回预编译函数的返回值；
2. 如果n没有定义为FMT_BUILTIN_CTZ，则计算从根节点到叶节点的最长的路径的LU大小，并返回该LU大小；
3. 如果路径长度超过float_info<float>中max_trailing_zeros的值，则将路径长度设为float_info<float>中max_trailing_zeros的值；
4. 实现了一个从根节点到叶节点的最长的路径的LU大小计算功能。


```
#ifdef FMT_BUILTIN_CTZ
  int t = FMT_BUILTIN_CTZ(n);
#else
  int t = ctz(n);
#endif
  if (t > float_info<float>::max_trailing_zeros)
    t = float_info<float>::max_trailing_zeros;

  const uint32_t mod_inv1 = 0xcccccccd;
  const uint32_t max_quotient1 = 0x33333333;
  const uint32_t mod_inv2 = 0xc28f5c29;
  const uint32_t max_quotient2 = 0x0a3d70a3;

  int s = 0;
  for (; s < t - 1; s += 2) {
    if (n * mod_inv2 > max_quotient2) break;
    n *= mod_inv2;
  }
  if (s < t && n * mod_inv1 <= max_quotient1) {
    n *= mod_inv1;
    ++s;
  }
  n >>= s;
  return s;
}

```cpp

This is a function that takes an integer `mod_inv1` and a positive integer `remainder` as input, and returns the remainder of the division `remainder / mod_inv1`.

The function uses a division algorithm to calculate the remainder, which is given by the last result on the divide operation. This algorithm is implemented using a series of if statements, each of which checks whether the result of the current division operation is zero or greater than the maximum remainder that could be动员.

The function returns 0 if the remainder is zero or the maximum remainder, and 1 if the result of the current division operation is greater than the maximum remainder. Otherwise, the function returns the remainder as the result.

It is worth noting that this function can be improved in terms of performance, as the remainder is not reduced in the recursive calls, but it is not clear how it can be optimized further.


```
// Removes trailing zeros and returns the number of zeros removed (double)
FMT_ALWAYS_INLINE int remove_trailing_zeros(uint64_t& n) FMT_NOEXCEPT {
#ifdef FMT_BUILTIN_CTZLL
  int t = FMT_BUILTIN_CTZLL(n);
#else
  int t = ctzll(n);
#endif
  if (t > float_info<double>::max_trailing_zeros)
    t = float_info<double>::max_trailing_zeros;
  // Divide by 10^8 and reduce to 32-bits
  // Since ret_value.significand <= (2^64 - 1) / 1000 < 10^17,
  // both of the quotient and the r should fit in 32-bits

  const uint32_t mod_inv1 = 0xcccccccd;
  const uint32_t max_quotient1 = 0x33333333;
  const uint64_t mod_inv8 = 0xc767074b22e90e21;
  const uint64_t max_quotient8 = 0x00002af31dc46118;

  // If the number is divisible by 1'0000'0000, work with the quotient
  if (t >= 8) {
    auto quotient_candidate = n * mod_inv8;

    if (quotient_candidate <= max_quotient8) {
      auto quotient = static_cast<uint32_t>(quotient_candidate >> 8);

      int s = 8;
      for (; s < t; ++s) {
        if (quotient * mod_inv1 > max_quotient1) break;
        quotient *= mod_inv1;
      }
      quotient >>= (s - 8);
      n = quotient;
      return s;
    }
  }

  // Otherwise, work with the remainder
  auto quotient = static_cast<uint32_t>(n / 100000000);
  auto remainder = static_cast<uint32_t>(n - 100000000 * quotient);

  if (t == 0 || remainder * mod_inv1 > max_quotient1) {
    return 0;
  }
  remainder *= mod_inv1;

  if (t == 1 || remainder * mod_inv1 > max_quotient1) {
    n = (remainder >> 1) + quotient * 10000000ull;
    return 1;
  }
  remainder *= mod_inv1;

  if (t == 2 || remainder * mod_inv1 > max_quotient1) {
    n = (remainder >> 2) + quotient * 1000000ull;
    return 2;
  }
  remainder *= mod_inv1;

  if (t == 3 || remainder * mod_inv1 > max_quotient1) {
    n = (remainder >> 3) + quotient * 100000ull;
    return 3;
  }
  remainder *= mod_inv1;

  if (t == 4 || remainder * mod_inv1 > max_quotient1) {
    n = (remainder >> 4) + quotient * 10000ull;
    return 4;
  }
  remainder *= mod_inv1;

  if (t == 5 || remainder * mod_inv1 > max_quotient1) {
    n = (remainder >> 5) + quotient * 1000ull;
    return 5;
  }
  remainder *= mod_inv1;

  if (t == 6 || remainder * mod_inv1 > max_quotient1) {
    n = (remainder >> 6) + quotient * 100ull;
    return 6;
  }
  remainder *= mod_inv1;

  n = (remainder >> 7) + quotient * 10ull;
  return 7;
}

```cpp

This is a function that appears to compute the retarded value of a root for a given root_fun, exponent, and interval. The function uses a combination of upper and lower bounds on the interval to avoid dividing by 0 and provides some additional flexibility in choosing the upper bound.

The function takes a cache object and an exponent as input, and returns the retarded value. If the interval is shorter than a certain threshold, the function uses the shorter interval and adjusts the exponent accordingly. If the interval is already less than the threshold, the function computes the round-up of the divisor and adjusts the exponent according to the rule for the tie-removing.

If the interval is already greater than the threshold, the function uses the interval and computes the signiflard of the retoaded value. If the interval is shorter than the given interval, the function uses the given interval and computes the signiflard according to the rule for the tie-removing.

The function also includes a check for the case where the given exponent is equal to the lower or upper threshold for the interval. In this case, the function returns the value without any interval or retoading.


```
// The main algorithm for shorter interval case
template <class T>
FMT_ALWAYS_INLINE FMT_SAFEBUFFERS decimal_fp<T> shorter_interval_case(
    int exponent) FMT_NOEXCEPT {
  decimal_fp<T> ret_value;
  // Compute k and beta
  const int minus_k = floor_log10_pow2_minus_log10_4_over_3(exponent);
  const int beta_minus_1 = exponent + floor_log2_pow10(-minus_k);

  // Compute xi and zi
  using cache_entry_type = typename cache_accessor<T>::cache_entry_type;
  const cache_entry_type cache = cache_accessor<T>::get_cached_power(-minus_k);

  auto xi = cache_accessor<T>::compute_left_endpoint_for_shorter_interval_case(
      cache, beta_minus_1);
  auto zi = cache_accessor<T>::compute_right_endpoint_for_shorter_interval_case(
      cache, beta_minus_1);

  // If the left endpoint is not an integer, increase it
  if (!is_left_endpoint_integer_shorter_interval<T>(exponent)) ++xi;

  // Try bigger divisor
  ret_value.significand = zi / 10;

  // If succeed, remove trailing zeros if necessary and return
  if (ret_value.significand * 10 >= xi) {
    ret_value.exponent = minus_k + 1;
    ret_value.exponent += remove_trailing_zeros(ret_value.significand);
    return ret_value;
  }

  // Otherwise, compute the round-up of y
  ret_value.significand =
      cache_accessor<T>::compute_round_up_for_shorter_interval_case(
          cache, beta_minus_1);
  ret_value.exponent = minus_k;

  // When tie occurs, choose one of them according to the rule
  if (exponent >= float_info<T>::shorter_interval_tie_lower_threshold &&
      exponent <= float_info<T>::shorter_interval_tie_upper_threshold) {
    ret_value.significand = ret_value.significand % 2 == 0
                                ? ret_value.significand
                                : ret_value.significand - 1;
  } else if (ret_value.significand < xi) {
    ++ret_value.significand;
  }
  return ret_value;
}

```cpp

The body of the function starts with a try-catch block that initializes the ret凿iled value and then continues with a series of checks and optimizations.

The first check is whether the result of the initial division operation by a larger divisor is negative. If it is, then a check for a small divisor is performed. This is done using a helper function `divide_by_10_to_kappa_plus_1`, which computes the商和余数，并使用向上取整将结果存储为带符号的整数。

If the small divisor is not found, then the loop continues with the next check. If the small divisor is found, then the loop continues with the previous branch, up to the point where the small divisor is the only remaining zero in the magnitude of the original operand.

After that, the loop continues with a check for the last remaining zero in the magnitude of the operand. If it is not found, then the loop continues with the previous branch. If it is found, then the loop ends and the ret凿iled value is assigned to the ret_value variable.

Finally, the function checks for the case where the exponent is negative and the two_fl is equal to zero. If such a case is found, then the loop continues up to the point where the small divisor is the only remaining zero in the magnitude of the original operand. This is done to allow for short-circuiting.

Overall, the function aims to optimize the computation of the ret凿iled value by finding the smallest possible divisor for the given operand, and using that divisor in the calculation of the ret凿iled value.


```
template <typename T>
FMT_SAFEBUFFERS decimal_fp<T> to_decimal(T x) FMT_NOEXCEPT {
  // Step 1: integer promotion & Schubfach multiplier calculation.

  using carrier_uint = typename float_info<T>::carrier_uint;
  using cache_entry_type = typename cache_accessor<T>::cache_entry_type;
  auto br = bit_cast<carrier_uint>(x);

  // Extract significand bits and exponent bits.
  const carrier_uint significand_mask =
      (static_cast<carrier_uint>(1) << float_info<T>::significand_bits) - 1;
  carrier_uint significand = (br & significand_mask);
  const carrier_uint exponent_mask =
      ((static_cast<carrier_uint>(1) << float_info<T>::exponent_bits) - 1)
      << float_info<T>::significand_bits;
  int exponent =
      static_cast<int>((br & exponent_mask) >> float_info<T>::significand_bits);

  if (exponent != 0) {  // Check if normal.
    exponent += float_info<T>::exponent_bias - float_info<T>::significand_bits;

    // Shorter interval case; proceed like Schubfach.
    if (significand == 0) return shorter_interval_case<T>(exponent);

    significand |=
        (static_cast<carrier_uint>(1) << float_info<T>::significand_bits);
  } else {
    // Subnormal case; the interval is always regular.
    if (significand == 0) return {0, 0};
    exponent = float_info<T>::min_exponent - float_info<T>::significand_bits;
  }

  const bool include_left_endpoint = (significand % 2 == 0);
  const bool include_right_endpoint = include_left_endpoint;

  // Compute k and beta.
  const int minus_k = floor_log10_pow2(exponent) - float_info<T>::kappa;
  const cache_entry_type cache = cache_accessor<T>::get_cached_power(-minus_k);
  const int beta_minus_1 = exponent + floor_log2_pow10(-minus_k);

  // Compute zi and deltai
  // 10^kappa <= deltai < 10^(kappa + 1)
  const uint32_t deltai = cache_accessor<T>::compute_delta(cache, beta_minus_1);
  const carrier_uint two_fc = significand << 1;
  const carrier_uint two_fr = two_fc | 1;
  const carrier_uint zi =
      cache_accessor<T>::compute_mul(two_fr << beta_minus_1, cache);

  // Step 2: Try larger divisor; remove trailing zeros if necessary

  // Using an upper bound on zi, we might be able to optimize the division
  // better than the compiler; we are computing zi / big_divisor here
  decimal_fp<T> ret_value;
  ret_value.significand = divide_by_10_to_kappa_plus_1(zi);
  uint32_t r = static_cast<uint32_t>(zi - float_info<T>::big_divisor *
                                              ret_value.significand);

  if (r > deltai) {
    goto small_divisor_case_label;
  } else if (r < deltai) {
    // Exclude the right endpoint if necessary
    if (r == 0 && !include_right_endpoint &&
        is_endpoint_integer<T>(two_fr, exponent, minus_k)) {
      --ret_value.significand;
      r = float_info<T>::big_divisor;
      goto small_divisor_case_label;
    }
  } else {
    // r == deltai; compare fractional parts
    // Check conditions in the order different from the paper
    // to take advantage of short-circuiting
    const carrier_uint two_fl = two_fc - 1;
    if ((!include_left_endpoint ||
         !is_endpoint_integer<T>(two_fl, exponent, minus_k)) &&
        !cache_accessor<T>::compute_mul_parity(two_fl, cache, beta_minus_1)) {
      goto small_divisor_case_label;
    }
  }
  ret_value.exponent = minus_k + float_info<T>::kappa + 1;

  // We may need to remove trailing zeros
  ret_value.exponent += remove_trailing_zeros(ret_value.significand);
  return ret_value;

  // Step 3: Find the significand with the smaller divisor

```cpp

The given function appears to check if a number is divisible by a given number (5^kappa or 2^kappa) and, if it is divisible, returns the商. If it is not divisible, the function returns the original number plus the number multiplied by the specified factor. The function uses a helper function check_divisibility_and_divide_by_pow5, which takes a number, a pow5 function, and a factor (5^kappa or 2^kappa).

The main body of the function starts by checking if the input number is divisible by 5^kappa. If it is divisible, the function adds the input number to its result and then checks if the result is divisible by 2^kappa. If it is divisible, the function optimizes the division by computing the division and taking the remainder. If the result is not divisible, the function adds the input number to its result.

The function also checks if the input number is divisible by 2^kappa. If it is divisible, the function returns the input number plus the input number multiplied by 2^kappa. If it is not divisible, the function returns the input number plus the number multiplied by the specified factor.


```
small_divisor_case_label:
  ret_value.significand *= 10;
  ret_value.exponent = minus_k + float_info<T>::kappa;

  const uint32_t mask = (1u << float_info<T>::kappa) - 1;
  auto dist = r - (deltai / 2) + (float_info<T>::small_divisor / 2);

  // Is dist divisible by 2^kappa?
  if ((dist & mask) == 0) {
    const bool approx_y_parity =
        ((dist ^ (float_info<T>::small_divisor / 2)) & 1) != 0;
    dist >>= float_info<T>::kappa;

    // Is dist divisible by 5^kappa?
    if (check_divisibility_and_divide_by_pow5<float_info<T>::kappa>(dist)) {
      ret_value.significand += dist;

      // Check z^(f) >= epsilon^(f)
      // We have either yi == zi - epsiloni or yi == (zi - epsiloni) - 1,
      // where yi == zi - epsiloni if and only if z^(f) >= epsilon^(f)
      // Since there are only 2 possibilities, we only need to care about the
      // parity. Also, zi and r should have the same parity since the divisor
      // is an even number
      if (cache_accessor<T>::compute_mul_parity(two_fc, cache, beta_minus_1) !=
          approx_y_parity) {
        --ret_value.significand;
      } else {
        // If z^(f) >= epsilon^(f), we might have a tie
        // when z^(f) == epsilon^(f), or equivalently, when y is an integer
        if (is_center_integer<T>(two_fc, exponent, minus_k)) {
          ret_value.significand = ret_value.significand % 2 == 0
                                      ? ret_value.significand
                                      : ret_value.significand - 1;
        }
      }
    }
    // Is dist not divisible by 5^kappa?
    else {
      ret_value.significand += dist;
    }
  }
  // Is dist not divisible by 2^kappa?
  else {
    // Since we know dist is small, we might be able to optimize the division
    // better than the compiler; we are computing dist / small_divisor here
    ret_value.significand +=
        small_division_by_pow10<float_info<T>::kappa>(dist);
  }
  return ret_value;
}
}  // namespace dragonbox

```cpp

This function appears to be a Python implementation of the MsgLite function `parse_number`. The `parse_number` function takes a string containing a number and returns the number. If the string is not a valid number, the function returns the maximum large integer that can be represented by the given string.

The function takes a number of digits as an argument and generates a string representation of that number usingascii digits. The function uses an `if` statement to check if the number is negative. If it is, the function negates the number by taking the absolute value of the number and adding 1.

The function uses a loop to generate the rest of the digits in the number and appends them to the generated string. After the loop has finished, the function checks if the generated string is a valid number. If it is not, the function returns the maximum large integer that can be represented by the given string. If the generated string is a valid number, the function returns the number.


```
// Formats value using a variation of the Fixed-Precision Positive
// Floating-Point Printout ((FPP)^2) algorithm by Steele & White:
// https://fmt.dev/p372-steele.pdf.
template <typename Double>
void fallback_format(Double d, int num_digits, bool binary32, buffer<char>& buf,
                     int& exp10) {
  bigint numerator;    // 2 * R in (FPP)^2.
  bigint denominator;  // 2 * S in (FPP)^2.
  // lower and upper are differences between value and corresponding boundaries.
  bigint lower;             // (M^- in (FPP)^2).
  bigint upper_store;       // upper's value if different from lower.
  bigint* upper = nullptr;  // (M^+ in (FPP)^2).
  fp value;
  // Shift numerator and denominator by an extra bit or two (if lower boundary
  // is closer) to make lower and upper integers. This eliminates multiplication
  // by 2 during later computations.
  const bool is_predecessor_closer =
      binary32 ? value.assign(static_cast<float>(d)) : value.assign(d);
  int shift = is_predecessor_closer ? 2 : 1;
  uint64_t significand = value.f << shift;
  if (value.e >= 0) {
    numerator.assign(significand);
    numerator <<= value.e;
    lower.assign(1);
    lower <<= value.e;
    if (shift != 1) {
      upper_store.assign(1);
      upper_store <<= value.e + 1;
      upper = &upper_store;
    }
    denominator.assign_pow10(exp10);
    denominator <<= 1;
  } else if (exp10 < 0) {
    numerator.assign_pow10(-exp10);
    lower.assign(numerator);
    if (shift != 1) {
      upper_store.assign(numerator);
      upper_store <<= 1;
      upper = &upper_store;
    }
    numerator *= significand;
    denominator.assign(1);
    denominator <<= shift - value.e;
  } else {
    numerator.assign(significand);
    denominator.assign_pow10(exp10);
    denominator <<= shift - value.e;
    lower.assign(1);
    if (shift != 1) {
      upper_store.assign(1ULL << 1);
      upper = &upper_store;
    }
  }
  // Invariant: value == (numerator / denominator) * pow(10, exp10).
  if (num_digits < 0) {
    // Generate the shortest representation.
    if (!upper) upper = &lower;
    bool even = (value.f & 1) == 0;
    num_digits = 0;
    char* data = buf.data();
    for (;;) {
      int digit = numerator.divmod_assign(denominator);
      bool low = compare(numerator, lower) - even < 0;  // numerator <[=] lower.
      // numerator + upper >[=] pow10:
      bool high = add_compare(numerator, *upper, denominator) + even > 0;
      data[num_digits++] = static_cast<char>('0' + digit);
      if (low || high) {
        if (!low) {
          ++data[num_digits - 1];
        } else if (high) {
          int result = add_compare(numerator, numerator, denominator);
          // Round half to even.
          if (result > 0 || (result == 0 && (digit % 2) != 0))
            ++data[num_digits - 1];
        }
        buf.try_resize(to_unsigned(num_digits));
        exp10 -= num_digits - 1;
        return;
      }
      numerator *= 10;
      lower *= 10;
      if (upper != &lower) *upper *= 10;
    }
  }
  // Generate the given number of digits.
  exp10 -= num_digits - 1;
  if (num_digits == 0) {
    buf.try_resize(1);
    denominator *= 10;
    buf[0] = add_compare(numerator, numerator, denominator) > 0 ? '1' : '0';
    return;
  }
  buf.try_resize(to_unsigned(num_digits));
  for (int i = 0; i < num_digits - 1; ++i) {
    int digit = numerator.divmod_assign(denominator);
    buf[i] = static_cast<char>('0' + digit);
    numerator *= 10;
  }
  int digit = numerator.divmod_assign(denominator);
  auto result = add_compare(numerator, numerator, denominator);
  if (result > 0 || (result == 0 && (digit % 2) != 0)) {
    if (digit == 9) {
      const auto overflow = '0' + 10;
      buf[num_digits - 1] = overflow;
      // Propagate the carry.
      for (int i = num_digits - 1; i > 0 && buf[i] == overflow; --i) {
        buf[i] = '0';
        ++buf[i - 1];
      }
      if (buf[0] == overflow) {
        buf[0] = '1';
        ++exp10;
      }
      return;
    }
    ++digit;
  }
  buf[num_digits - 1] = static_cast<char>('0' + digit);
}

```cpp

This is a function that adds two浮点 numbers and returns the result. It takes two arguments, a buffer `buf` containing the data and a precision `precision` (int or char). The precision is used to limit the number of significant digits.

The function first normalizes the first number and then generates the power of the decimal number. It then converts the decimal power to the corresponding exponent in the IEEE754 format.

If the precision is higher than the maximum number of significant digits, the function will use the upper bound for the number of significant digits.

The function uses a helper function `grisu_gen_digits` to generate the exponents, which takes a decimal number, a precision, and a handler. If the call to `grisu_gen_digits` fails, the function falls back to using the default format for the IEEE754 representation of the decimal number.

The function returns the number of significant digits in the result. If the precision is not specified, the default format will be used.


```
template <typename T>
int format_float(T value, int precision, float_specs specs, buffer<char>& buf) {
  static_assert(!std::is_same<T, float>::value, "");
  FMT_ASSERT(value >= 0, "value is negative");

  const bool fixed = specs.format == float_format::fixed;
  if (value <= 0) {  // <= instead of == to silence a warning.
    if (precision <= 0 || !fixed) {
      buf.push_back('0');
      return 0;
    }
    buf.try_resize(to_unsigned(precision));
    std::uninitialized_fill_n(buf.data(), precision, '0');
    return -precision;
  }

  if (!specs.use_grisu) return snprintf_float(value, precision, specs, buf);

  if (precision < 0) {
    // Use Dragonbox for the shortest format.
    if (specs.binary32) {
      auto dec = dragonbox::to_decimal(static_cast<float>(value));
      write<char>(buffer_appender<char>(buf), dec.significand);
      return dec.exponent;
    }
    auto dec = dragonbox::to_decimal(static_cast<double>(value));
    write<char>(buffer_appender<char>(buf), dec.significand);
    return dec.exponent;
  }

  // Use Grisu + Dragon4 for the given precision:
  // https://www.cs.tufts.edu/~nr/cs257/archive/florian-loitsch/printf.pdf.
  int exp = 0;
  const int min_exp = -60;  // alpha in Grisu.
  int cached_exp10 = 0;     // K in Grisu.
  fp normalized = normalize(fp(value));
  const auto cached_pow = get_cached_power(
      min_exp - (normalized.e + fp::significand_size), cached_exp10);
  normalized = normalized * cached_pow;
  // Limit precision to the maximum possible number of significant digits in an
  // IEEE754 double because we don't need to generate zeros.
  const int max_double_digits = 767;
  if (precision > max_double_digits) precision = max_double_digits;
  fixed_handler handler{buf.data(), 0, precision, -cached_exp10, fixed};
  if (grisu_gen_digits(normalized, 1, exp, handler) == digits::error) {
    exp += handler.size - cached_exp10 - 1;
    fallback_format(value, handler.precision, specs.binary32, buf, exp);
  } else {
    exp += handler.exp10;
    buf.try_resize(to_unsigned(handler.size));
  }
  if (!fixed && !specs.showpoint) {
    // Remove trailing zeros.
    auto num_digits = buf.size();
    while (num_digits > 0 && buf[num_digits - 1] == '0') {
      --num_digits;
      ++exp;
    }
    buf.try_resize(num_digits);
  }
  return exp;
}  // namespace detail

```cpp

这段代码是一个C++模板函数，名为`snprintf_float`，用于将一个`float`类型的值按照要求的长度精确地打印到缓冲区中。

具体来说，该函数接收三个参数：

1. `value`：要打印的`float`值。
2. `precision`：用于表示精度目标的整数，取值范围为0到7，其中0表示不输出的精度，6表示输出6位有效数字。
3. `specs`：格式化选项，包括精度、小数点后数字、指数和科学计数法。
4. 一个`char`类型的缓冲区`buf`，用于存储打印后留下的字符和格式信息。

函数内部首先检查缓冲区大小是否足够，然后根据传入的格式选项和最低有效精度计算输出的精度。接着，该函数根据输入的`value`和`specs`构造格式字符串，然后使用`snprintf`函数将格式字符串打印到缓冲区中。

由于`snprintf`函数会尝试插入`%e`形式的输出，它会要求输入值的精度至少达到6位。如果精度不足，函数会使用`%f`和`%g`形式来输出，分别代表6位和36位精度。


```
template <typename T>
int snprintf_float(T value, int precision, float_specs specs,
                   buffer<char>& buf) {
  // Buffer capacity must be non-zero, otherwise MSVC's vsnprintf_s will fail.
  FMT_ASSERT(buf.capacity() > buf.size(), "empty buffer");
  static_assert(!std::is_same<T, float>::value, "");

  // Subtract 1 to account for the difference in precision since we use %e for
  // both general and exponent format.
  if (specs.format == float_format::general ||
      specs.format == float_format::exp)
    precision = (precision >= 0 ? precision : 6) - 1;

  // Build the format string.
  enum { max_format_size = 7 };  // The longest format is "%#.*Le".
  char format[max_format_size];
  char* format_ptr = format;
  *format_ptr++ = '%';
  if (specs.showpoint && specs.format == float_format::hex) *format_ptr++ = '#';
  if (precision >= 0) {
    *format_ptr++ = '.';
    *format_ptr++ = '*';
  }
  if (std::is_same<T, long double>()) *format_ptr++ = 'L';
  *format_ptr++ = specs.format != float_format::hex
                      ? (specs.format == float_format::fixed ? 'f' : 'e')
                      : (specs.upper ? 'A' : 'a');
  *format_ptr = '\0';

  // Format using snprintf.
  auto offset = buf.size();
  for (;;) {
    auto begin = buf.data() + offset;
    auto capacity = buf.capacity() - offset;
```cpp

This is a C++ function that takes a buffer (a variable of memory with a size), a specified format (a string indicating the format of the data to be stored in the buffer), and an optional offset (a number of bytes beyond the start of the buffer) to indicate the position of the first data element.

The function takes two arguments: a buffer and a format string. It checks whether the format string is "hex" and, if it is, it attempts to parse the data in the buffer based on that format.

The function first checks if the buffer is of the correct size and then, if the format is "hex", it attempts to parse the data by reading it in binary format and then applying the specified formatting. If the format is not "hex", it attempts to parse the data by reading it in binary format and applying the specified formatting. If the parse is successful, the function returns the result.

If the parse is not successful, the function returns the specified offset from the buffer. If the buffer is too small, the function returns the result of the maximum value supported by the specified format.

The function is decorated with the `const` attribute, which indicates that the function does not modify the buffer.


```
#ifdef FMT_FUZZ
    if (precision > 100000)
      throw std::runtime_error(
          "fuzz mode - avoid large allocation inside snprintf");
#endif
    // Suppress the warning about a nonliteral format string.
    // Cannot use auto because of a bug in MinGW (#1532).
    int (*snprintf_ptr)(char*, size_t, const char*, ...) = FMT_SNPRINTF;
    int result = precision >= 0
                     ? snprintf_ptr(begin, capacity, format, precision, value)
                     : snprintf_ptr(begin, capacity, format, value);
    if (result < 0) {
      // The buffer will grow exponentially.
      buf.try_reserve(buf.capacity() + 1);
      continue;
    }
    auto size = to_unsigned(result);
    // Size equal to capacity means that the last character was truncated.
    if (size >= capacity) {
      buf.try_reserve(size + offset + 1);  // Add 1 for the terminating '\0'.
      continue;
    }
    auto is_digit = [](char c) { return c >= '0' && c <= '9'; };
    if (specs.format == float_format::fixed) {
      if (precision == 0) {
        buf.try_resize(size);
        return 0;
      }
      // Find and remove the decimal point.
      auto end = begin + size, p = end;
      do {
        --p;
      } while (is_digit(*p));
      int fraction_size = static_cast<int>(end - p - 1);
      std::memmove(p, p + 1, to_unsigned(fraction_size));
      buf.try_resize(size - 1);
      return -fraction_size;
    }
    if (specs.format == float_format::hex) {
      buf.try_resize(size + offset);
      return 0;
    }
    // Find and parse the exponent.
    auto end = begin + size, exp_pos = end;
    do {
      --exp_pos;
    } while (*exp_pos != 'e');
    char sign = exp_pos[1];
    assert(sign == '+' || sign == '-');
    int exp = 0;
    auto p = exp_pos + 2;  // Skip 'e' and sign.
    do {
      assert(is_digit(*p));
      exp = exp * 10 + (*p++ - '0');
    } while (p != end);
    if (sign == '-') exp = -exp;
    int fraction_size = 0;
    if (exp_pos != begin + 1) {
      // Remove trailing zeros.
      auto fraction_end = exp_pos - 1;
      while (*fraction_end == '0') --fraction_end;
      // Move the fractional part left to get rid of the decimal point.
      fraction_size = static_cast<int>(fraction_end - begin - 1);
      std::memmove(begin + 1, begin + 2, to_unsigned(fraction_size));
    }
    buf.try_resize(to_unsigned(fraction_size) + offset + 1);
    return exp - fraction_size;
  }
}

```cpp

这段代码是一个名为 "branchless-utf8" 的公共领域分支less UTF-8 解码器，由 Christopher Wellons 开发。它的作用是解码下一个字符 'c' 从缓冲区中，并在遇到错误时输出错误。

代码中首先定义了一个名为 "branchless-utf8" 的常量，然后定义了一个名为 "decodeTheNextCharacter" 的函数。函数从缓冲区中读取四个字节，而不是四个字符。接着函数会检查读取的字节数是否足够，如果是，就从缓冲区中读取多一个字节来填充。

函数的实现主要分为两部分：错误处理和正确的解码操作。

错误处理部分，代码使用 "e" 变量来存储错误信息，如果解码过程中出现了错误，那么 "e" 变量将不为零。错误信息使用 "invalid byte sequence"、"non-canonical encoding" 或 "surrogate half" 来描述。当遇到错误时，函数会将 "a" 赋值给 "err"，然后输出错误信息，并返回 "null"。

正确的解码操作部分，代码使用 "i" 变量来存储读取的字节数，然后使用解码操作将 'c' 解码为对应的编码单元。解码操作的实现依赖于具体的编码方案，这里简单实现了 ASCII 编码的utf8解码。

最后，函数还返回了 "a" 变量，用于表示下一个解码的起始位置。当遇到正确解码的 'c' 时，函数会将其存储在 "c" 变量中，并输出正确的结果。


```
// A public domain branchless UTF-8 decoder by Christopher Wellons:
// https://github.com/skeeto/branchless-utf8
/* Decode the next character, c, from buf, reporting errors in e.
 *
 * Since this is a branchless decoder, four bytes will be read from the
 * buffer regardless of the actual length of the next character. This
 * means the buffer _must_ have at least three bytes of zero padding
 * following the end of the data stream.
 *
 * Errors are reported in e, which will be non-zero if the parsed
 * character was somehow invalid: invalid byte sequence, non-canonical
 * encoding, or a surrogate half.
 *
 * The function returns a pointer to the next character. When an error
 * occurs, this pointer will be a guess that depends on the particular
 * error, but it will always advance at least one byte.
 */
```cpp

This code appears to implement a function that takes a pointer to a four-byte character array (`buf`) and a shift value (`shift`) as input. It returns a pointer to the next character in the array or the end of the array if the input is an invalid shift value.

The function starts by computing the pointer to the next character early so that the next iteration can start working on the next character. Then, it loads the first four bytes of the array into the `c` variable and accumulates a series of error conditions based on the input.

The error conditions are checked and the function returns the pointer to the next character in the array or the end of the array if any of the conditions are met. The `e` variable is assigned the result of the check and the `shifte` array is used to determine the appropriate shift for the given input.

It is worth noting that the code seems to be using the wrong header file for the `sizeof` operator, which should be `sizeof(int)` instead of `sizeof(unsigned char*)`. This could cause issues if the code is compiled on a different platform.


```
inline const char* utf8_decode(const char* buf, uint32_t* c, int* e) {
  static const char lengths[] = {1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,
                                 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0,
                                 0, 0, 2, 2, 2, 2, 3, 3, 4, 0};
  static const int masks[] = {0x00, 0x7f, 0x1f, 0x0f, 0x07};
  static const uint32_t mins[] = {4194304, 0, 128, 2048, 65536};
  static const int shiftc[] = {0, 18, 12, 6, 0};
  static const int shifte[] = {0, 6, 4, 2, 0};

  auto s = reinterpret_cast<const unsigned char*>(buf);
  int len = lengths[s[0] >> 3];

  // Compute the pointer to the next character early so that the next
  // iteration can start working on the next character. Neither Clang
  // nor GCC figure out this reordering on their own.
  const char* next = buf + len + !len;

  // Assume a four-byte character and load four bytes. Unused bits are
  // shifted out.
  *c = uint32_t(s[0] & masks[len]) << 18;
  *c |= uint32_t(s[1] & 0x3f) << 12;
  *c |= uint32_t(s[2] & 0x3f) << 6;
  *c |= uint32_t(s[3] & 0x3f) << 0;
  *c >>= shiftc[len];

  // Accumulate the various error conditions.
  *e = (*c < mins[len]) << 6;       // non-canonical encoding
  *e |= ((*c >> 11) == 0x1b) << 7;  // surrogate half?
  *e |= (*c > 0x10FFFF) << 8;       // out of range?
  *e |= (s[1] & 0xc0) >> 2;
  *e |= (s[2] & 0xc0) >> 4;
  *e |= (s[3]) >> 6;
  *e ^= 0x2a;  // top two bits of each tail byte correct?
  *e >>= shifte[len];

  return next;
}

```cpp

这段代码定义了一个名为`stringifier`的结构体类型，它是一个`FMT_INLINE`模板，用于实现格式化操作。

`stringifier`结构体包括一个名为`operator()`的成员函数，它接受一个`T`类型的参数，并返回一个`std::string`类型的结果。这个函数使用了`to_string()`函数将`T`类型转换为字符串，然后通过`std::string`类型的函数返回结果。

另外，`stringifier`还包括一个名为`operator()`的成员函数，它接受一个`basic_format_arg<format_context>::handle`类型的参数，并返回一个`std::string`类型的结果。这个函数使用了`format_parse_context`和`format_context`类型的函数，这些函数用于解析和格式化输入的文本字符串。它通过`basic_format_arg`类型的函数获取输入的格式化上下文，然后使用`format()`函数将输入的文本字符串解析为指定的格式，最后通过`std::string`类型的函数将解析后的格式化结果返回。

该代码中定义的`formatter`结构体是一个模板类，用于实现格式化输出`std::string`类型的值。该结构体包括一个名为`parse()`的成员函数和一个名为`format()`的成员函数。`parse()`函数是一个`format_parse_context`类型的函数，用于获取解析上下文，而`format()`函数则用于解析和格式化输入的文本字符串，并返回一个`std::string`类型的结果。

这两个函数的具体实现如下：


```
struct stringifier {
  template <typename T> FMT_INLINE std::string operator()(T value) const {
    return to_string(value);
  }
  std::string operator()(basic_format_arg<format_context>::handle h) const {
    memory_buffer buf;
    format_parse_context parse_ctx({});
    format_context format_ctx(buffer_appender<char>(buf), {}, {});
    h.format(parse_ctx, format_ctx);
    return to_string(buf);
  }
};
}  // namespace detail

template <> struct formatter<detail::bigint> {
  format_parse_context::iterator parse(format_parse_context& ctx) {
    return ctx.begin();
  }

  format_context::iterator format(const detail::bigint& n,
                                  format_context& ctx) {
    auto out = ctx.out();
    bool first = true;
    for (auto i = n.bigits_.size(); i > 0; --i) {
      auto value = n.bigits_[i - 1u];
      if (first) {
        out = format_to(out, "{:x}", value);
        first = false;
        continue;
      }
      out = format_to(out, "{:08x}", value);
    }
    if (n.exp_ > 0)
      out = format_to(out, "p{}", n.exp_ * detail::bigint::bigit_bits);
    return out;
  }
};

```cpp

这段代码定义了一个名为FMT_FUNC的函数，它接受一个名为s的CStringView类型的参数，并返回一个字符数组，其中包含了从UTF-8编码到UTF-16编码的字符。

函数内部首先定义了一个内部类型transcode，该类型代表将输入字符串从UTF-8编码转换为UTF-16编码的回调函数。transcode函数接受一个const char*类型的参数p，并返回一个内部类型cp表示当前正在读取的字节数，以及一个int类型的error表示当前发生的错误。

如果error为0，说明输入字符串的读取没有问题。否则，将尝试从输入字符串中读取一个有效范围从0x20000到0x2FFFF的字符，并将其存储到buffer数组中。如果无法找到所有需要读取的字符，将在buffer数组的末尾添加一个'\0'字符，以便于后续操作。

接下来，从输入字符串的起始位置开始，按照每次4个字节读取一个有效范围的字符，并将其转换为UTF-16编码。如果缓冲区数组已满，需要将当前剩余的字符读入缓冲区并继续读取。最后，在函数的末尾添加一个'\0'字符，以结束字符串。


```
FMT_FUNC detail::utf8_to_utf16::utf8_to_utf16(string_view s) {
  auto transcode = [this](const char* p) {
    auto cp = uint32_t();
    auto error = 0;
    p = utf8_decode(p, &cp, &error);
    if (error != 0) FMT_THROW(std::runtime_error("invalid utf8"));
    if (cp <= 0xFFFF) {
      buffer_.push_back(static_cast<wchar_t>(cp));
    } else {
      cp -= 0x10000;
      buffer_.push_back(static_cast<wchar_t>(0xD800 + (cp >> 10)));
      buffer_.push_back(static_cast<wchar_t>(0xDC00 + (cp & 0x3FF)));
    }
    return p;
  };
  auto p = s.data();
  const size_t block_size = 4;  // utf8_decode always reads blocks of 4 chars.
  if (s.size() >= block_size) {
    for (auto end = p + s.size() - block_size + 1; p < end;) p = transcode(p);
  }
  if (auto num_chars_left = s.data() + s.size() - p) {
    char buf[2 * block_size - 1] = {};
    memcpy(buf, p, to_unsigned(num_chars_left));
    p = buf;
    do {
      p = transcode(p);
    } while (p - buf < num_chars_left);
  }
  buffer_.push_back(0);
}

```cpp

这段代码是一个名为`format_system_error`的函数，它接受三个参数：`out`是一个字符缓冲区，`error_code`是表示错误码的整数类型，`message`是一个字符串，用于存储错误信息。

函数内部的作用是输出系统错误信息，即当`error_code`发生时，从系统日志或其他可靠的错误信息源中获取相应的错误信息，并将其输出到`out`参数所指向的字符缓冲区中。

具体实现过程如下：

1. 首先，创建一个大小为`inline_buffer_size`的字符缓冲区`buf`；
2. 循环直至`buf`中已有足够的位置来存储系统错误信息，然后尝试从`error_code`和错误信息来源中获取相应的信息；
3. 如果获取到信息成功，则将其存储到`buf`中；
4. 如果获取到信息失败，即发生ERANGE错误，则输出错误码并返回；
5. 如果前4步都成功，但`buf`中已有足够的长度来存储错误信息，则需要将`buf`的字节数加倍，并重新开始循环；
6. 最后，调用`format_error_code`函数，输出错误码和错误信息；
7. 在函数内部，还发生了一个异常处理，即如果`buf`不足或者发生ERANGE错误，则会报告错误。


```
FMT_FUNC void format_system_error(detail::buffer<char>& out, int error_code,
                                  string_view message) FMT_NOEXCEPT {
  FMT_TRY {
    memory_buffer buf;
    buf.resize(inline_buffer_size);
    for (;;) {
      char* system_message = &buf[0];
      int result =
          detail::safe_strerror(error_code, system_message, buf.size());
      if (result == 0) {
        format_to(detail::buffer_appender<char>(out), "{}: {}", message,
                  system_message);
        return;
      }
      if (result != ERANGE)
        break;  // Can't get error message, report error code instead.
      buf.resize(buf.size() * 2);
    }
  }
  FMT_CATCH(...) {}
  format_error_code(out, error_code, message);
}

```cpp

这段代码定义了三个函数：

1. `error_handler::on_error()`函数，它接收一个字符串参数`message`，并使用`FMT_THROW()`函数将其封装为一个格式化错误信息，然后将该信息抛出。
2. `report_system_error()`函数，它接收一个整数参数`error_code`和一个字符串参数`message`，并使用`report_error()`函数将其封装为一个格式化系统错误信息，然后将该信息抛出。
3. `detail::vformat()`函数，它接收一个字符串参数`format_str`和一个格式参数`args`，并将其解析为格式化字符串，然后根据解析出的格式字符串和解析出的格式参数进行格式化，最后将结果返回。

函数`error_handler::on_error()`的作用是在程序出现错误时进行处理，它接收一个错误信息字符串`message`，并使用`FMT_THROW()`函数将其封装为一个格式化错误信息，然后将该信息抛出，以便进行错误处理。

函数`report_system_error()`的作用是在需要报告系统错误时进行处理，它接收一个整数参数`error_code`和一个字符串参数`message`，并使用`report_error()`函数将其封装为一个格式化系统错误信息，然后将该信息抛出，以便进行错误处理。

函数`detail::vformat()`的作用是格式化字符串，它接收一个字符串参数`format_str`和一个格式参数`args`，并将其解析为格式化字符串，然后根据解析出的格式字符串和解析出的格式参数进行格式化，最后将结果返回。它假定`format_str`是一个字符串，而`args`是一个格式参数，它的格式字符串定义了要格式化的字符串的格式，例如`{}`可以表示将`format_arg()`函数的第一个参数作为格式化字符串的一部分。


```
FMT_FUNC void detail::error_handler::on_error(const char* message) {
  FMT_THROW(format_error(message));
}

FMT_FUNC void report_system_error(int error_code,
                                  fmt::string_view message) FMT_NOEXCEPT {
  report_error(format_system_error, error_code, message);
}

FMT_FUNC std::string detail::vformat(string_view format_str, format_args args) {
  if (format_str.size() == 2 && equal2(format_str.data(), "{}")) {
    auto arg = args.get(0);
    if (!arg) error_handler().on_error("argument not found");
    return visit_format_arg(stringifier(), arg);
  }
  memory_buffer buffer;
  detail::vformat_to(buffer, format_str, args);
  return to_string(buffer);
}

```cpp

这段代码是一个C++程序，它定义了一个名为“WriteConsoleW”的函数，该函数的实现如下：

```
int WriteConsoleW(void* output_ptr, const void* format_str, const format_args& args) {
   // 在这里实现函数逻辑
}
```cpp

该函数使用了C++的dllimport特性，它从名为“detail”的命名空间中引入了函数实现。

函数的实现主要步骤如下：

1. 根据传入的输出文件描述符f和格式字符串format_str，加载对应的C++格式化函数，并将其存储在内存缓冲区buffer中。
2. 判断输入文件是否为Windows系统，如果是，则执行操作系统函数WriteConsoleW，将其与格式化字符串和格式化参数相关联，并将结果存储到输出缓冲区written中。
3. 如果操作系统不是Windows系统，或者输入文件不是可读写文件，函数将抛出格式错误并返回。

通过执行该函数，可以实现将字符串格式化并输出到操作系统命令行窗口的功能。


```
#ifdef _WIN32
namespace detail {
using dword = conditional_t<sizeof(long) == 4, unsigned long, unsigned>;
extern "C" __declspec(dllimport) int __stdcall WriteConsoleW(  //
    void*, const void*, dword, dword*, void*);
}  // namespace detail
#endif

FMT_FUNC void vprint(std::FILE* f, string_view format_str, format_args args) {
  memory_buffer buffer;
  detail::vformat_to(buffer, format_str,
                     basic_format_args<buffer_context<char>>(args));
#ifdef _WIN32
  auto fd = _fileno(f);
  if (_isatty(fd)) {
    detail::utf8_to_utf16 u16(string_view(buffer.data(), buffer.size()));
    auto written = detail::dword();
    if (!detail::WriteConsoleW(reinterpret_cast<void*>(_get_osfhandle(fd)),
                               u16.c_str(), static_cast<uint32_t>(u16.size()),
                               &written, nullptr)) {
      FMT_THROW(format_error("failed to write to console"));
    }
    return;
  }
```cpp

这段代码定义了两个函数，一个是 `detail::fwrite_fully`，用于将一个 `memory_buffer` 对象中的数据写入到文件中，另一个是 `detail::vprint_mojibake`，用于打印字符串 `"mojibake"`，并使用特定的编码格式。

第一个函数 `detail::fwrite_fully` 的参数包括一个 `FILE` 类型的指针 `f`，一个 `memory_buffer` 对象 `buffer`，以及一个表示要写入文件的字符数组 `args`。函数的作用是将 `buffer` 对象中的数据按照 `args` 中的格式进行解析，然后将解析后的数据写入到 `f` 中。

第二个函数 `detail::vprint_mojibake` 的参数与 `detail::fwrite_fully` 函数相反，它包括一个 `FILE` 类型的指针 `f`，一个 `string_view` 类型的格式字符串 `format_str`，以及一个 `format_args` 类型的参数 `args`，用于解析格式字符串中的信息。函数的作用是在文件中打印字符串 `"mojibake"`，并使用指定的编码格式 `"mojibake"`。由于 `args` 是一个 `format_args` 对象，所以它包含的信息不会在函数内部保存，而是需要在调用函数时传递给函数。


```
#endif
  detail::fwrite_fully(buffer.data(), 1, buffer.size(), f);
}

#ifdef _WIN32
// Print assuming legacy (non-Unicode) encoding.
FMT_FUNC void detail::vprint_mojibake(std::FILE* f, string_view format_str,
                                      format_args args) {
  memory_buffer buffer;
  detail::vformat_to(buffer, format_str,
                     basic_format_args<buffer_context<char>>(args));
  fwrite_fully(buffer.data(), 1, buffer.size(), f);
}
#endif

```cpp

这段代码定义了一个名为`vprint`的函数，也被称为`vprint函数`。函数接受两个参数：`format_str`是一个表示格式字符串的`string_view`类型，`args`是一个包含两个额外的参数的`format_args`类型。

函数的作用是将`format_str`和`args`所表示的格式字符串和格式参数打印到标准输出(通常是终端)。

该函数使用了C++标准库中的`<cstdio>`头文件，因此它可以在使用该头文件的标准库应用中使用。

该函数是C++标准库中的一个保留函数，它遵循了C++准则，因此在C++应用程序中，它不需要声明，因为编译器会根据函数名称和参数类型自动推导其含义。


```
FMT_FUNC void vprint(string_view format_str, format_args args) {
  vprint(stdout, format_str, args);
}

FMT_END_NAMESPACE

#endif  // FMT_FORMAT_INL_H_

```