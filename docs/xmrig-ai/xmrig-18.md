# xmrig源码解析 18

# `src/3rdparty/fmt/ranges.h`

这段代码定义了一个名为FMT_RANGES_H_的文件，其中包含了一个experimental range support(实验范围支持)。该支持在C++11中引入，并一直延续到C++20。它的作用是提供一个格式化库，用于在C++程序中格式化可变长度的输入和输出。它支持多种可变长度的输入和输出，包括整型、浮点型和字符串等。

在该文件中，定义了一个<initializer_list>头文件，它是一个C++11中的新特性，允许在函数参数列表中使用初始izer列表。这个头文件也被FMT_RANGES_H_中的实验范围支持所使用，用于支持可变长度的输入和输出。

此外，该文件中还包括一些说明文件，如FMT_RANGES_H_说明文件和FMT_RANGES_DEFINE_H_说明文件，它们提供了关于该实验范围支持和FMT_RANGES_H_头文件更详细的说明。


```cpp
// Formatting library for C++ - experimental range support
//
// Copyright (c) 2012 - present, Victor Zverovich
// All rights reserved.
//
// For the license information refer to format.h.
//
// Copyright (c) 2018 - present, Remotion (Igor Schulz)
// All Rights Reserved
// {fmt} support for ranges, containers and types tuple interface.

#ifndef FMT_RANGES_H_
#define FMT_RANGES_H_

#include <initializer_list>
```

这段代码是一个C++程序，它定义了一个名为“formatting_base”的模板结构体，该结构体有一个名为“parse”的纯虚函数。

模板<typename Char>结构体定义了两个模板参数：一个是“ParseContext”类型的参数，另一个是“fmt”参数，其中“fmt”是一个未定义的类型。

“parse”函数是纯虚函数，它的实现与参数字符串上下文对象的“begin”成员的实现无关。它的目的是提供一种访问格式化字符串中当前正在解析的字符的机制。返回类型是“ decltype(ctx.begin())”类型，它表示 ParseContext 对象中当前正在解析的字符的类型，并且该类型被声明为纯虚类型，因此可以安全的跨越到参数字符串上下文对象的类型。

最后，该代码中定义了一个名为“fmt_range_output_limit”的宏，它的值为“FMT_RANGE_OUTPUT_LENGTH_LIMIT”，如果该编译器支持该宏，则只输出格式化字符串中从范围的第一个字符到格式化字符串中最后一个字符之间的字符，否则输出范围将包含第一个字符到格式化字符串中最后一个字符之间的所有字符。


```cpp
#include <type_traits>

#include "format.h"

// output only up to N items from the range.
#ifndef FMT_RANGE_OUTPUT_LENGTH_LIMIT
#  define FMT_RANGE_OUTPUT_LENGTH_LIMIT 256
#endif

FMT_BEGIN_NAMESPACE

template <typename Char> struct formatting_base {
  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    return ctx.begin();
  }
};

```

这段代码定义了两个模板结构体：formatting_range和formatting_tuple。它们都基于char类型，并采用了格式化输出库（可能是用于C或C++）的标准模板元编程技术。

formatting_range模板结构体有一个关于size_t类型变量range_length_limit的静态常量，以及三个成员变量：prefix、delimiter和postfix。这些成员变量都具有默认值（此处为空字符串），以及add_delimiter_spaces和add_prepostfix_space的默认值，它们分别表示是否应该在输出中添加嵌套的括号和前置/后缀的空间。

formatting_tuple模板结构体也有三个成员变量：prefix、delimiter和postfix，以及两个静态常量：add_delimiter_spaces和add_prepostfix_space。这些成员变量都具有默认值，以及add_delimiter_spaces和add_prepostfix_space的默认值。

这两个模板结构体似乎用于定义输出字符串中预先定义的占位符（如{}，括号，分号等）以及是否包含这些占位符。具体而言，formatting_range和formatting_tuple的结构体都包含了一个output_statement，其中output_statement定义了要输出的字符序列，并在其中包含了占位符。通过在结构体中声明的静态常量和成员变量，可以控制output_statement中占位符的插入和删除。


```cpp
template <typename Char, typename Enable = void>
struct formatting_range : formatting_base<Char> {
  static FMT_CONSTEXPR_DECL const size_t range_length_limit =
      FMT_RANGE_OUTPUT_LENGTH_LIMIT;  // output only up to N items from the
                                      // range.
  Char prefix;
  Char delimiter;
  Char postfix;
  formatting_range() : prefix('{'), delimiter(','), postfix('}') {}
  static FMT_CONSTEXPR_DECL const bool add_delimiter_spaces = true;
  static FMT_CONSTEXPR_DECL const bool add_prepostfix_space = false;
};

template <typename Char, typename Enable = void>
struct formatting_tuple : formatting_base<Char> {
  Char prefix;
  Char delimiter;
  Char postfix;
  formatting_tuple() : prefix('('), delimiter(','), postfix(')') {}
  static FMT_CONSTEXPR_DECL const bool add_delimiter_spaces = true;
  static FMT_CONSTEXPR_DECL const bool add_prepostfix_space = false;
};

```

这两段代码都是使用了 C++ 的模板库，定义了两个模板函数，一个是拷贝一个范围（RangeT）类型的数据，另一个是拷贝一个字符串（char）类型的数据。

在这两段代码中，首先定义了两个输出迭代器（OutputIterator）类型的变量 out 和 str，用于存储数据范围和数据串中的元素。然后，在模板实现中，使用了循环来遍历数据范围或数据串中的元素，将元素的值输出到 out 迭代器中，并返回 out。其中，循环的条件是 out 迭代器已经指向了数据范围或数据串的最后一个元素。

这两段代码的作用是，将一个数据范围或数据串中的元素拷贝到另一个输出迭代器中。可以用于许多应用场景，比如将一个数据范围中的元素导出到一个文件中，或者将一个字符串中的元素导出到一个字符数组中。


```cpp
namespace detail {

template <typename RangeT, typename OutputIterator>
OutputIterator copy(const RangeT& range, OutputIterator out) {
  for (auto it = range.begin(), end = range.end(); it != end; ++it)
    *out++ = *it;
  return out;
}

template <typename OutputIterator>
OutputIterator copy(const char* str, OutputIterator out) {
  while (*str) *out++ = *str++;
  return out;
}

```

这段代码定义了一个模板类`is_like_std_string`，该模板判断一个给定的值是否类似于`std::string`。`std::string`接口是`std::string_view`的实现，因此该接口也被称为`std::string::view`。

该代码中定义了一个名为`copy`的模板函数，它接受一个输出迭代器和一个字符变量`ch`。函数的作用是将`ch`复制到输出迭代器`out`中，并返回该输出迭代器。

该代码中还定义了一个模板函数`is_like_std_string`，该函数有一个模板参数`U`，一个模板参数为`void`的函数`check`和一个模板参数列表`...`。函数的作用是判断给定的值是否类似于`std::string`，并返回一个`const bool`类型的真值。函数的具体实现包括两个模板函数：`is_string<T>::value`和`!std::is_void<decltype(check<T>(nullptr))>::value`。这两个模板函数的具体实现如下：

```cpp说是_IS_STRING_IMPL_DEF_
decltype(check<T>(nullptr))`);
```

这两个函数的作用是：

1. `is_string<T>::value`：如果给定的值`T`具有`std::string`接口，则返回`true`，否则返回`false`。
2. `!std::is_void<decltype(check<T>(nullptr))>::value`：如果给定的值`T`不具有`std::string`接口，则需要判断`check<T>(nullptr)`是否为`void`类型。如果为`void`类型，则返回`true`，否则返回`false`。

该代码中还定义了一个模板类`OutputIterator`，该模板类提供了一个`copy`函数，接受一个输出迭代器和一个字符变量`ch`。函数的作用是将`ch`复制到输出迭代器`out`中，并返回该输出迭代器。


```cpp
template <typename OutputIterator>
OutputIterator copy(char ch, OutputIterator out) {
  *out++ = ch;
  return out;
}

/// Return true value if T has std::string interface, like std::string_view.
template <typename T> class is_like_std_string {
  template <typename U>
  static auto check(U* p)
      -> decltype((void)p->find('a'), p->length(), (void)p->data(), int());
  template <typename> static void check(...);

 public:
  static FMT_CONSTEXPR_DECL const bool value =
      is_string<T>::value || !std::is_void<decltype(check<T>(nullptr))>::value;
};

```

这段代码定义了一系列结构体和模板以处理字符串的类似于标准库字符串的标识符。以下是代码的概述：

1. is_like_std_string模板结构体模板，有一个参数Char，它是一个类型参数。这个模板结构体用于表示一个字符串是否类似于标准库中的字符串，即以<fmt::basic_string_view<Char>>为格式的基本字符串 view。

2. conditional_helper模板模板，有一个参数Ts，它是一个类型参数。这个模板用于在is_range_结构体模板中提供条件判断。

3. is_range_结构体模板，有一个参数T，它是一个类型参数。这个模板用于表示一个范围，如字符串中的位置范围等。它的实现基于一个decltype函数，它返回T的别裁类型（条件为真时为T，否则为void类型）。

4. conditional_t模板模板，有一个参数Ts，它是一个类型参数。这个模板用于表示一个条件，即判断Ts是否为null。

5. template <typename T> struct is_range { // 对于T类型，is_range表示... }模板，用于表示一个范围，它是一个is_range_结构体模板的别裁类型。

6. template <typename... Ts> struct conditional_helper { // 用于处理其他类型参数...

7. template <typename... Ts> struct conditional_helper<Ts...> { // 对于Ts...类型参数，conditional_helper是一个虚函数，因为它只在其中定义了一个虚函数体。

8. template <typename T> struct is_range_<T, conditional_t<
   decltype(std::declval<T>().begin()) == 'a' &&
   decltype(std::declval<T>().end()) == 'z' ? std::false_type : std::true_type>> : std::false_type; // 对于T类型，is_range_表示...

9. template <typename... Ts> struct is_range<Ts...> { // 用于表示Ts...类型参数，is_range是一个is_range_结构体模板的别裁类型。

10. 最后，不要忘记在适当的位置初始化和使用这些模板结构体和模板函数。


```cpp
template <typename Char>
struct is_like_std_string<fmt::basic_string_view<Char>> : std::true_type {};

template <typename... Ts> struct conditional_helper {};

template <typename T, typename _ = void> struct is_range_ : std::false_type {};

#if !FMT_MSC_VER || FMT_MSC_VER > 1800
template <typename T>
struct is_range_<
    T, conditional_t<false,
                     conditional_helper<decltype(std::declval<T>().begin()),
                                        decltype(std::declval<T>().end())>,
                     void>> : std::true_type {};
#endif

```

这段代码是一个模板类，名为`is_tuple_like_`。它的作用是检查一个给定的模板型参数`U`是否属于`tuple_like`类型。如果是，就返回`std::tuple_size<U>::value`，否则返回`int()`。

具体来说，这个模板类有两个模板，一个模板参数是`U`，另一个模板参数可以省略，但是不能是一个模板参数。第一个模板用于检查给定的`U`是否属于`tuple_like`类型，如果是，就返回`std::tuple_size<U>::value`，否则返回`int()`。第二个模板用于检查给定的模板参数是否可以被赋值给`U`，如果可以，就返回`true`，否则返回`false`。

该代码的实现没有产生任何输出，因此它只是一个声明，不会输出任何值。


```cpp
/// tuple_size and tuple_element check.
template <typename T> class is_tuple_like_ {
  template <typename U>
  static auto check(U* p) -> decltype(std::tuple_size<U>::value, int());
  template <typename> static void check(...);

 public:
  static FMT_CONSTEXPR_DECL const bool value =
      !std::is_void<decltype(check<T>(nullptr))>::value;
};

// Check for integer_sequence
#if defined(__cpp_lib_integer_sequence) || FMT_MSC_VER >= 1900
template <typename T, T... N>
using integer_sequence = std::integer_sequence<T, N...>;
```

这段代码定义了一系列模板结构体，用于表示序列中元素的类型和数量。其中，第一个模板结构体是`index_sequence`，用于在序列中根据元素的下标进行索引。第二个模板结构体是`make_index_sequence`，用于根据给定的模板参数序列中的元素的下标生成索引序列。

接下来的代码定义了一系列模板结构体`integer_sequence`，用于表示整数序列中的元素类型和数量。这些模板结构体使用了`T`作为其元素类型，`size_t`作为其成员变量类型，并使用`size()`函数获取元素的数量。

最后，定义了一系列模板结构体`make_integer_sequence`和`make_integer_sequence`，用于根据不同的模板参数生成整数序列。这些模板结构体使用了`T`，`N`和`Ns`作为其元素类型，并使用`size()`函数获取元素的数量。其中，`make_integer_sequence`结构体是用于根据模板参数中的元素生成整数序列，而`make_integer_sequence`结构体是用于根据模板参数中的元素生成整数序列，但是该模板结构体需要手动指定生成的整数序列中的元素数量。


```cpp
template <size_t... N> using index_sequence = std::index_sequence<N...>;
template <size_t N> using make_index_sequence = std::make_index_sequence<N>;
#else
template <typename T, T... N> struct integer_sequence {
  using value_type = T;

  static FMT_CONSTEXPR size_t size() { return sizeof...(N); }
};

template <size_t... N> using index_sequence = integer_sequence<size_t, N...>;

template <typename T, size_t N, T... Ns>
struct make_integer_sequence : make_integer_sequence<T, N - 1, N - 1, Ns...> {};
template <typename T, T... Ns>
struct make_integer_sequence<T, 0, Ns...> : integer_sequence<T, Ns...> {};

```

这段代码定义了一个名为`make_index_sequence`的模板类，可以计算一个序列中所有元素在数据结构中的索引。其中，`T`是一个模板参数，指定了要使用的数据结构类型。

该代码中还定义了一个辅助函数`for_each`，该函数接收一个`index_sequence`类型的参数，它是一个由`size_t...Is`组成的序列。该函数的实现是在一个`for`循环中，依次将参数`tup`和`f`传递给其中的每一个元素，并使用`get`函数获取该元素在数据结构中的索引，然后将对应的值赋给`tup`参数。

该代码还定义了一个名为`get_indexes`的函数，该函数接收一个`T`类型的参数，返回一个`make_index_sequence<std::tuple_size<T>::value>`类型的值。它的实现是使用模板元编程技术，通过重复定义模板函数来实现的。

该代码中使用的`std::tuple_size`函数可以用来计算一个`T`类型数据结构中的元素数量。


```cpp
template <size_t N>
using make_index_sequence = make_integer_sequence<size_t, N>;
#endif

template <class Tuple, class F, size_t... Is>
void for_each(index_sequence<Is...>, Tuple&& tup, F&& f) FMT_NOEXCEPT {
  using std::get;
  // using free function get<I>(T) now.
  const int _[] = {0, ((void)f(get<Is>(tup)), 0)...};
  (void)_;  // blocks warnings
}

template <class T>
FMT_CONSTEXPR make_index_sequence<std::tuple_size<T>::value> get_indexes(
    T const&) {
  return {};
}

```

这段代码定义了一个模板函数 `for_each`，接受两个参数：一个 `Tuple` 类型的变量 `tup` 和一个 `F` 类型的函数引用 `f`。

函数的作用是遍历 `tup` 中的元素，将 `f` 中的函数应用于每个元素，并将 `Tuple` 类型转换为 `std::decltype<Tuple>::type` 以便于模板元编程。

接下来是两个模板定义：

1. 定义了一个名为 `value_type` 的模板别称，它的形式为 `decltype(*std::declval<Range>().begin())`，其中 `Range` 是某个模板元编程语言中使用的模板类或模板函数。

2. 定义了一个名为 `format_str_quoted` 的函数，它接受一个 `bool` 类型的参数 `add_space` 和一个 `const Arg&` 类型的参数。函数的作用是在给定的 `add_space` 是否为真时，将字符串 "{}" 或 "{}" 添加到字符串中。

最后，该代码定义的模板函数 `for_each` 在 `std::vector` 和 `std::transform` 的实现中被广泛使用。


```cpp
template <class Tuple, class F> void for_each(Tuple&& tup, F&& f) {
  const auto indexes = get_indexes(tup);
  for_each(indexes, std::forward<Tuple>(tup), std::forward<F>(f));
}

template <typename Range>
using value_type = remove_cvref_t<decltype(*std::declval<Range>().begin())>;

template <typename Arg, FMT_ENABLE_IF(!is_like_std_string<
                                      typename std::decay<Arg>::type>::value)>
FMT_CONSTEXPR const char* format_str_quoted(bool add_space, const Arg&) {
  return add_space ? " {}" : "{}";
}

template <typename Arg, FMT_ENABLE_IF(is_like_std_string<
                                      typename std::decay<Arg>::type>::value)>
```

这段代码是一个函数，名为 FMT_CONSTEXPR，它接受两个参数，一个是 bool 类型的 add\_space 参数，另一个是 const Arg 类型的 argument。

函数的作用是定义了一个输出字符串的格式字符串，根据 add\_space 的值，选择不同的字符串格式，并返回。

具体来说，当 add\_space 为 true 时，函数输出 "{}"，其中 {} 是格式字符串中的占位符，表示要插入的参数；当 add\_space 为 false 时，函数输出 "}"，其中的 } 是格式字符串中的占位符，表示不需要插入参数。

对于 const wchar\_t* format\_str\_quoted 和 const char* format\_str\_quoted，它们的实现与上面两个函数相同，只是输出参数的类型不同。


```cpp
FMT_CONSTEXPR const char* format_str_quoted(bool add_space, const Arg&) {
  return add_space ? " \"{}\"" : "\"{}\"";
}

FMT_CONSTEXPR const char* format_str_quoted(bool add_space, const char*) {
  return add_space ? " \"{}\"" : "\"{}\"";
}
FMT_CONSTEXPR const wchar_t* format_str_quoted(bool add_space, const wchar_t*) {
  return add_space ? L" \"{}\"" : L"\"{}\"";
}

FMT_CONSTEXPR const char* format_str_quoted(bool add_space, const char) {
  return add_space ? " '{}'" : "'{}'";
}
FMT_CONSTEXPR const wchar_t* format_str_quoted(bool add_space, const wchar_t) {
  return add_space ? L" '{}'" : L"'{}'";
}
}  // namespace detail

```

This is a C++ template struct that represents a tuple with a single element of type T. It includes a formatter that templates on a function that takes a tuple of elements of type T and a context of type FormatContext.

The formatter has two variants:

1. If the tuple has a single element of type T, the formatter prints a space between the element and the next element in the tuple.
2. If the tuple has more than one element of type T, the formatter prints the element as a double braces, followed by a space between the next two elements in the tuple.

The template parameter for the formatter is a tuple of three types: the type of the tuple element (T), a Char, and a pointer to a FormatContext. The FormatContext is a pointer to a template parameter of type FormatContext.

The `format()` method of the formatter is defined with a single template parameter of type T, which is the type of the tuple element. The method takes a const TupleT& parameter, which is the tuple being formatted.

The `parse()` method of the formatter is defined with a single template parameter of type T, which is the type of the input. The method takes a FormatContext parameter, which is the context for the parse operation. The method returns the type of the input.

The `format()` method templates on the `parse()` method using the `for_each` and `format_each` helper functions. The `for_each` function is used to iterate over the elements of the tuple, while the `format_each` function is used to apply the formatting to each element.

Overall, this struct provides a simple way to format a tuple element as a string, while also handling cases where the tuple has more than one element.


```cpp
template <typename T> struct is_tuple_like {
  static FMT_CONSTEXPR_DECL const bool value =
      detail::is_tuple_like_<T>::value && !detail::is_range_<T>::value;
};

template <typename TupleT, typename Char>
struct formatter<TupleT, Char, enable_if_t<fmt::is_tuple_like<TupleT>::value>> {
 private:
  // C++11 generic lambda for format()
  template <typename FormatContext> struct format_each {
    template <typename T> void operator()(const T& v) {
      if (i > 0) {
        if (formatting.add_prepostfix_space) {
          *out++ = ' ';
        }
        out = detail::copy(formatting.delimiter, out);
      }
      out = format_to(out,
                      detail::format_str_quoted(
                          (formatting.add_delimiter_spaces && i > 0), v),
                      v);
      ++i;
    }

    formatting_tuple<Char>& formatting;
    size_t& i;
    typename std::add_lvalue_reference<decltype(
        std::declval<FormatContext>().out())>::type out;
  };

 public:
  formatting_tuple<Char> formatting;

  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    return formatting.parse(ctx);
  }

  template <typename FormatContext = format_context>
  auto format(const TupleT& values, FormatContext& ctx) -> decltype(ctx.out()) {
    auto out = ctx.out();
    size_t i = 0;
    detail::copy(formatting.prefix, out);

    detail::for_each(values, format_each<FormatContext>{formatting, i, out});
    if (formatting.add_prepostfix_space) {
      *out++ = ' ';
    }
    detail::copy(formatting.postfix, out);

    return ctx.out();
  }
};

```

This is a C++ template that defines a formatter class that represents a CSV (Comma Separated Values) formatter. The formatter class has two template arguments: a type template (T) and a character type template (Char).

The third argument of the formatter class template is a third template argument that represents the enable\_if\_t, which is used to enable or disable the generation of code for helper functions that may or may not implement the same interface as the formatter.

The formatter class has two methods: parse() and format(). The parse() method takes a parse context (ParseContext&) and returns the type of the value that the formatter should start parsing. The format() method takes a FormatContext& and returns the type of the element that the formatter should format. The format() method takes a comma separated value stream (e.g., a CSV line) and returns the type of the element that the formatter should format.

The format() method has a series of helper methods that are intended to be used by users of the formatter. These helper methods include copy() templates that are used to copy elements of the formatter to the input stream or the output stream. Additionally, it includes a delimiter template that is used to specify the separator between the columns in a CSV line.


```cpp
template <typename T, typename Char> struct is_range {
  static FMT_CONSTEXPR_DECL const bool value =
      detail::is_range_<T>::value && !detail::is_like_std_string<T>::value &&
      !std::is_convertible<T, std::basic_string<Char>>::value &&
      !std::is_constructible<detail::std_string_view<Char>, T>::value;
};

template <typename T, typename Char>
struct formatter<
    T, Char,
    enable_if_t<fmt::is_range<T, Char>::value
// Workaround a bug in MSVC 2017 and earlier.
#if !FMT_MSC_VER || FMT_MSC_VER >= 1927
                && has_formatter<detail::value_type<T>, format_context>::value
#endif
                >> {
  formatting_range<Char> formatting;

  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    return formatting.parse(ctx);
  }

  template <typename FormatContext>
  typename FormatContext::iterator format(const T& values, FormatContext& ctx) {
    auto out = detail::copy(formatting.prefix, ctx.out());
    size_t i = 0;
    auto it = values.begin();
    auto end = values.end();
    for (; it != end; ++it) {
      if (i > 0) {
        if (formatting.add_prepostfix_space) *out++ = ' ';
        out = detail::copy(formatting.delimiter, out);
      }
      out = format_to(out,
                      detail::format_str_quoted(
                          (formatting.add_delimiter_spaces && i > 0), *it),
                      *it);
      if (++i > formatting.range_length_limit) {
        out = format_to(out, " ... <other elements>");
        break;
      }
    }
    if (formatting.add_prepostfix_space) *out++ = ' ';
    return detail::copy(formatting.postfix, out);
  }
};

```

This is a C++ template for a formatter that takes a tuple of arguments of a fixed type, char and a variable number of template arguments of a similar type. The formatter is used to format the input string according to the rules defined by the user.

The template meta-information of the formatter includes a template parameter of type Char, which specifies that the type of the input string is a character. The template parameter is then expanded to include the other template arguments of the tuple, of which the fixed type of the last template argument is specified.

The template method include the parse method, which is used to return the beginning of the input string, since the beginning of the string is considered to be the first argument.

The format method is used to insert the values of the tuple into the input string. The first template parameter of the format method is the format context, which is used to specify the index sequence of the elements to insert. The second template parameter is the tuple argument joined with the specified format context, which is used to join the elements of the tuple into a single sequence. The third template parameter is the input string, which is passed to the format method.

The private member functions, format and format\_args, are used to implement the formatting logic. The format function is used to insert the elements of the tuple into the input string, and the format\_args function is used to expand the elements of the tuple and insert them into the input string.


```cpp
template <typename Char, typename... T> struct tuple_arg_join : detail::view {
  const std::tuple<T...>& tuple;
  basic_string_view<Char> sep;

  tuple_arg_join(const std::tuple<T...>& t, basic_string_view<Char> s)
      : tuple{t}, sep{s} {}
};

template <typename Char, typename... T>
struct formatter<tuple_arg_join<Char, T...>, Char> {
  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    return ctx.begin();
  }

  template <typename FormatContext>
  typename FormatContext::iterator format(
      const tuple_arg_join<Char, T...>& value, FormatContext& ctx) {
    return format(value, ctx, detail::make_index_sequence<sizeof...(T)>{});
  }

 private:
  template <typename FormatContext, size_t... N>
  typename FormatContext::iterator format(
      const tuple_arg_join<Char, T...>& value, FormatContext& ctx,
      detail::index_sequence<N...>) {
    return format_args(value, ctx, std::get<N>(value.tuple)...);
  }

  template <typename FormatContext>
  typename FormatContext::iterator format_args(
      const tuple_arg_join<Char, T...>&, FormatContext& ctx) {
    // NOTE: for compilers that support C++17, this empty function instantiation
    // can be replaced with a constexpr branch in the variadic overload.
    return ctx.out();
  }

  template <typename FormatContext, typename Arg, typename... Args>
  typename FormatContext::iterator format_args(
      const tuple_arg_join<Char, T...>& value, FormatContext& ctx,
      const Arg& arg, const Args&... args) {
    using base = formatter<typename std::decay<Arg>::type, Char>;
    auto out = ctx.out();
    out = base{}.format(arg, ctx);
    if (sizeof...(Args) > 0) {
      out = std::copy(value.sep.begin(), value.sep.end(), out);
      ctx.advance_to(out);
      return format_args(value, ctx, args...);
    }
    return out;
  }
};

```

这段代码定义了一个名为 `tuple_arg_join` 的模板元编程变量，其参数为一个 `std::tuple` 容器中的 `T...` 类型。该模板元编程变量有一个额外的参数 `string_view` `sep`，它是一个表示元素分隔符的字符串。

模板元编程变量定义了一个名为 `join` 的函数，该函数接受一个 `std::tuple` 容器和 `string_view` `sep` 作为参数。函数返回一个新的 `tuple` 对象，其中元素的第一个部分是 `T` 类型的值，第二个部分是 `sep` 分隔的子元素。

通过 `template` 关键字，该函数可以接受不同的 `T...` 类型的参数。例如，如果在调用 `join` 时传递的是 `std::tuple<int, char>` 和 `std::tuple<int, char, int>`，则 `join` 函数返回一个新的 `tuple` 对象，其中元素的第一个部分是 `int`，第二个部分是 `'a'`，第三个部分是 `int`。

最终的结果是，`fmt::print` 函数将 `tuple` 对象作为第一个参数，`string_view` `sep` 作为第二个参数，并使用 `fmt::join` 函数将元素连接起来输出结果。例如，如果在调用 `fmt::print` 时传递的是 `"1, a"`，则输出结果为 `"1, a"`。


```cpp
/**
  \rst
  Returns an object that formats `tuple` with elements separated by `sep`.

  **Example**::

    std::tuple<int, char> t = {1, 'a'};
    fmt::print("{}", fmt::join(t, ", "));
    // Output: "1, a"
  \endrst
 */
template <typename... T>
FMT_CONSTEXPR tuple_arg_join<char, T...> join(const std::tuple<T...>& tuple,
                                              string_view sep) {
  return {tuple, sep};
}

```

这段代码定义了一个名为 `FMT_CONSTEXPR tuple_arg_join<wchar_t, T...>` 的模板函数，它采用模板元编程技术，可以用于不同类型的元编程(如 C++, Rust, etc.)中。

该函数接受一个可变参数 `T...` 和一个固定参数 `sep`，它使用这些参数来格式化 `initializer_list`。

函数的作用是将给定的 `initializer_list` 和 `sep` 连接起来，并将它们返回给用户。这里的 `initializer_list` 是一个格式化字符串，可以使用字符串连接符 `,` 来连接多个初始化器。通过 `sep`, 函数将会在连接每个初始化器时使用 `sep` 中的字符作为分隔符，并将它们连接成一个 `tuple`。

由于 `T...` 是一个可变参数，因此 `FMT_CONSTEXPR tuple_arg_join` 可以接受任何数量的可变参数。当函数接受两个参数时(即 `T...` 和 `sep`)，它会将 `T...` 和 `sep` 连接起来，并返回一个新的 `tuple`。当函数接受三个参数时(即 `T...`、`sep` 和 `T`)，它将 `T...` 和 `sep` 连接起来，并将它们附加到 `T` 上，返回一个新的 `tuple`。

该函数可以用于在模板库中构建格式化字符串，如 `fmt::print`。例如，以下代码将输出 `"1, 2, 3"`:

```cpp
fmt::print("{}", fmt::join({1, 2, 3}, ", "));
``` 




```cpp
template <typename... T>
FMT_CONSTEXPR tuple_arg_join<wchar_t, T...> join(const std::tuple<T...>& tuple,
                                                 wstring_view sep) {
  return {tuple, sep};
}

/**
  \rst
  Returns an object that formats `initializer_list` with elements separated by
  `sep`.

  **Example**::

    fmt::print("{}", fmt::join({1, 2, 3}, ", "));
    // Output: "1, 2, 3"
  \endrst
 */
```

这段代码定义了两个arg_join模板结构体，一个是char类型的，另一个是wchar_t类型的（即Unicode字符）。两个模板结构体都接受两个参数：一个是要连接的输入序列（std::initializer_list<T>），另一个是用于分隔输入序列的符串（string_view sep）。

两个模板结构体的实现部分相同，都是一个带参数的函数join()，它接收输入序列list，以及分隔符串sep。函数的实现主要是在两个const模板螺旋式调用了arg_join库的函数，通过这两个函数，实现输入序列的连接。

最终，函数join()返回一个arg_join类型的对象，使用了两个模板结构体中的烧录类型，具体取决于输入序列类型T的类型，可以是char或者wchar_t。


```cpp
template <typename T>
arg_join<const T*, const T*, char> join(std::initializer_list<T> list,
                                        string_view sep) {
  return join(std::begin(list), std::end(list), sep);
}

template <typename T>
arg_join<const T*, const T*, wchar_t> join(std::initializer_list<T> list,
                                           wstring_view sep) {
  return join(std::begin(list), std::end(list), sep);
}

FMT_END_NAMESPACE

#endif  // FMT_RANGES_H_

```

# `src/3rdparty/getopt/getopt.h`

这段代码是一个头文件，名为 `__GETOPT_H__`，它定义了一个函数 `__getopt_h`。这个头文件的作用是告诉编译器如何定义这个函数，即告诉编译器在编译之前需要做什么准备工作。

具体来说，这个头文件中包含了一些声明和定义，它们描述了函数 `__getopt_h` 的参数和返回值。函数 `__getopt_h` 的作用是定义了 `__getopt_h` 函数，它接受一个整数参数 `opt`，用于获取命令行参数中的 `--` 或 `-` 选项。

此外，头文件中还包含了一些关于版权的声明，它们指出该头文件及其代码的版权归属，以及允许如何使用、复制、修改和分发这些代码。


```cpp
#ifndef __GETOPT_H__
/**
 * DISCLAIMER
 * This file is part of the mingw-w64 runtime package.
 *
 * The mingw-w64 runtime package and its code is distributed in the hope that it
 * will be useful but WITHOUT ANY WARRANTY.  ALL WARRANTIES, EXPRESSED OR
 * IMPLIED ARE HEREBY DISCLAIMED.  This includes but is not limited to
 * warranties of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
 */
 /*
 * Copyright (c) 2002 Todd C. Miller <Todd.Miller@courtesan.com>
 *
 * Permission to use, copy, modify, and distribute this software for any
 * purpose with or without fee is hereby granted, provided that the above
 * copyright notice and this permission notice appear in all copies.
 *
 * THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
 * WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
 * MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
 * ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
 * WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
 * ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
 * OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
 *
 * Sponsored in part by the Defense Advanced Research Projects
 * Agency (DARPA) and Air Force Research Laboratory, Air Force
 * Materiel Command, USAF, under agreement number F39502-99-1-0512.
 */
```

这段代码是一个头文件，它定义了一个名为`extern_values`的函数。这个函数的作用是告诉用户这个软件是一个开源的、经过NetBSD基金会赞助的、仅在有限的责任下允许分发和使用的软件。它还包含一些声明，告诉用户这个软件的一些限制条件。

具体来说，这个函数下面的声明包括：

1. 该软件在发行任何二进制形式（如二进制文件、可执行文件等）时，必须包括一份由版权拥有者（NetBSD基金会）和贡献者（ Dieter Baron和Thomas Klausner）签署的版权通知，以及一个特殊的声明，其中包含以下内容：

   ```cpp
   Copyright (c) 2000 The NetBSD Foundation, Inc. All rights reserved.
   This code is derived from software contributed to The NetBSD Foundation by Dieter Baron and Thomas Klausner.
   Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:
   ... (省略了其余的条件，以下是关于限制条件的简短说明)
   ```

   2. 该软件在发行任何源代码形式时，必须包括一份由版权拥有者（NetBSD基金会）和贡献者（ Dieter Baron和Thomas Klausner）签署的版权通知，以及一个特殊的声明，其中包含以下内容：

   ```cpp
   Copyright (c) 2000 The NetBSD Foundation, Inc. All rights reserved.
   This code is derived from software contributed to The NetBSD Foundation by Dieter Baron and Thomas Klausner.
   Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:
   ... (省略了其余的条件，以下是关于限制条件的简短说明)
   ```

   3. 该软件的任何二进制形式都必须在文件中包含上述版权通知和声明。

最后，在函数体中，它提到了`extern_values`函数的作用是返回`true`，这意味着`extern_values`函数确实存在，并且它的返回值对它的使用方有用处。


```cpp
/*-
 * Copyright (c) 2000 The NetBSD Foundation, Inc.
 * All rights reserved.
 *
 * This code is derived from software contributed to The NetBSD Foundation
 * by Dieter Baron and Thomas Klausner.
 *
 * Redistribution and use in source and binary forms, with or without
 * modification, are permitted provided that the following conditions
 * are met:
 * 1. Redistributions of source code must retain the above copyright
 *    notice, this list of conditions and the following disclaimer.
 * 2. Redistributions in binary form must reproduce the above copyright
 *    notice, this list of conditions and the following disclaimer in the
 *    documentation and/or other materials provided with the distribution.
 *
 * THIS SOFTWARE IS PROVIDED BY THE NETBSD FOUNDATION, INC. AND CONTRIBUTORS
 * ``AS IS'' AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED
 * TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
 * PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE FOUNDATION OR CONTRIBUTORS
 * BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR
 * CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF
 * SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS
 * INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN
 * CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE)
 * ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE
 * POSSIBILITY OF SUCH DAMAGE.
 */

```

这段代码是一个C/C++语言的预处理指令，名为“#pragma warning(disable:4996)”。它的作用是在编译时将4996号警告信息从编译器中省略掉。这里的__GETOPT_H__是定义了一个宏，会在编译时生成一个包含所有选项的定义表。

具体来说，这段代码以下面几行为前提：

1. 在#pragma warning(disable:4996)之前，如果 warning 设置为真（即 `__GNUC__ __ pow__ (4996) < 0`），则会输出警告信息，其中`__GNUC__`和`__pow__`是C/C++的预处理指令，`__pow__`用于计算两个整数的幂。
2. 在#define __GETOPT_H__之前，会生成一个定义表，其中包含所有选项。这里使用的是宏定义，而不是函数定义，所以生成的定义表是存储在 `__GETOPT_H__` 和 `__GETOPT_L__` 之间的。
3. 在#include <crtdefs.h>、#include <errno.h>、#include <stdlib.h>、#include <string.h>、#include <stdarg.h>、#include <stdio.h>、#include <windows.h> 等头文件包含的内容，是在编译时定义的。

总之，这段代码的作用是定义了一个宏，用于在编译时生成一个包含所有选项的定义表，从而避免警告信息的输出。


```cpp
#pragma warning(disable:4996)

#define __GETOPT_H__

/* All the headers include this file. */
#include <crtdefs.h>
#include <errno.h>
#include <stdlib.h>
#include <string.h>
#include <stdarg.h>
#include <stdio.h>
#include <windows.h>

#ifdef __cplusplus
extern "C" {
```

这段代码定义了一个名为“REPLACE_GETOPT”的宏，它使用系统命令（由 getopt(3) 调用），用于设置获取命令中 opt 和 optarg 参数的相关选项。

具体来说，该宏包含以下参数：

* opt：一个代表选项名称的标识符，可以是选项名称（通过使用“-”来标识选项）或选项描述（通过使用“--”来标识选项）。
* optind：一个标识符，用于标识 argv 数组中第一个非选项的索引。
* optopt：一个字符，用于检查选项是否有效。
* optreset：一个整数，用于重置 getopt 函数的返回值。
* optarg：一个字符指针，指向传递给 getopt(3) 的非选项部分。

如果定义了宏 REPLACE_GETOPT，则可以用如下方式使用：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <getopt.h>

#define REPLACE_GETOPT

int usage(int argc, char *argv[]) {
   if (argc < 2) {
       printf("Usage: %s %s\n", argv[0], argv[1]);
       return 1;
   }
   return 0;
}

int main(int argc, char *argv[]) {
   if (REPLACE_GETOPT) {
       int optind = atoi(argv[1]);
       if (optind < 1 || optind >= argc) {
           printf("Usage: %s %s\n", argv[0], argv[1]);
           return usage(argc, argv);
       }
       int optopt = atoi(argv[optind]);
       if (optopt != '?' || optopt < 0 || optopt >= argc) {
           printf("Usage: %s %s\n", argv[0], argv[1]);
           return usage(argc, argv);
       }
   }
   return usage(argc, argv);
}
```

这段代码的作用是：

1. 如果定义了宏 REPLACE_GETOPT，则可以像这样使用：

```cpp
$ ./myprogram -o option1 -o option2 --option3
```

2. 如果不定义宏 REPLACE_GETOPT，则无法使用上述方式。


```cpp
#endif

#define	REPLACE_GETOPT		/* use this getopt as the system getopt(3) */

#ifdef REPLACE_GETOPT
int	opterr = 1;		/* if error message should be printed */
int	optind = 1;		/* index into parent argv vector */
int	optopt = '?';		/* character checked for validity */
#undef	optreset		/* see getopt.h */
#define	optreset		__mingw_optreset
int	optreset;		/* reset getopt */
char    *optarg;		/* argument associated with option */
#endif

//extern int optind;		/* index of first non-option in argv      */
```

这段代码定义了几个外部变量 optopt、opterr 和 optarg，以及几个宏定义 PRINT_ERROR、FLAG_PERMUTE、FLAG_ALLARGS 和 FLAG_LONGONLY。

optopt 是一个整型变量，是一个选项字符，由用户设置为零时表示不使用该选项。

opterr 是一个整型变量，用于存储当前选项的错误信息，当用户设置了选项时，opterr 会根据设置选项的情况返回不同的值。

optarg 是一个指向当前选项 arguments 的指针。

宏定义 FLAG_PERMUTE 表示如果用户设置了选项，则可以将所有选项 non-options 重新排列到 argv 的末尾。

宏定义 FLAG_ALLARGS 表示将所有选项 non-options 视为 args，将其放在 option-1 的后面。

宏定义 FLAG_LONGONLY 表示仅处理 option-1，就像 getopt_long_only 函数一样。

整型变量 BADCH 和 BADARG 分别返回当前选项的错误信息和非选项的参数名称，如果用户设置了选项，则返回不同的值。


```cpp
//extern int optopt;		/* single option character, as parsed     */
//extern int opterr;		/* flag to enable built-in diagnostics... */
//				/* (user may set to zero, to suppress)    */
//
//extern char *optarg;		/* pointer to argument of current option  */

#define PRINT_ERROR	((opterr) && (*options != ':'))

#define FLAG_PERMUTE	0x01	/* permute non-options to the end of argv */
#define FLAG_ALLARGS	0x02	/* treat non-options as args to option "-1" */
#define FLAG_LONGONLY	0x04	/* operate as getopt_long_only */

/* return values */
#define	BADCH		(int)'?'
#define	BADARG		((*options == ':') ? (int)':' : (int)'?')
```

这段代码定义了一个宏，名为INORDER，其含义为int类型，值为1。

该代码中包含两个条件编译指令，分别为：

1. 在__CYGWIN__这个标识符存在的情况下，定义了名为__progname的函数，其含义为给定的程序名称。如果没有这个标识符存在，则会定义一个名为__progname的extern函数，函数名与当前程序名称相同，但需要在函数定义前加上extern关键字。

2. 在__CYGWIN__这个标识符不存在的情况下，定义了一个名为__declspec(dllimport)的extern函数指针变量，该函数指针指向__progname函数。

该代码中还包含一个名为EMSG的静态字符数组，用于存储函数运行时可能产生的错误信息。

另外，该代码还定义了一个名为getopt_internal的函数，该函数接受4个参数：int类型的选项参数i、字符串参数p、const char *类型的选项参数const char *options，以及int类型的选项参数opt; int类型的返回值为int，表示从options参数中解析出的最长周长。

最后，该代码还定义了一个名为parse_long_options的函数，该函数同样接受4个参数：char *const *类型的选项参数options，const char *类型的选项参数const char *options，以及int类型的选项参数opt; int类型的返回值为int，表示从options参数中解析出的最长周长。

该代码还定义了一个名为gcd的函数，用于计算两个整数的最大公约数。


```cpp
#define	INORDER 	(int)1

#ifndef __CYGWIN__
#define __progname __argv[0]
#else
extern char __declspec(dllimport) *__progname;
#endif

static char EMSG[] = "";

static int getopt_internal(int, char * const *, const char *,
			   const struct option *, int *, int);
static int parse_long_options(char * const *, const char *,
			      const struct option *, int *, int);
static int gcd(int, int);
```

这段代码是一个C语言中的函数，名为“permute_args”。函数声明在函数头中，但源代码没有给出。函数的作用是处理命令行参数的排列组合问题，支持多种选项（option），并且允许在命令行参数中包含选项 -h、--help 或者 --option 等。

具体来说，这段代码可以处理如下命令行参数：

1. 传递给函数的参数：int类型，参数个数不限，但必须包含一个int类型的参数，该参数作为第一个选项。
2. 传递给函数的参数：int类型，参数个数不限，但必须包含一个int类型的参数，该参数作为第二个选项。
3. 传递给函数的参数：int类型，参数个数不限，但必须包含一个int类型的参数，该参数作为第三个选项。
4. 传递给函数的参数：char类型，参数个数不限。

函数内部通过以下步骤实现这些功能：

1. 根据传递给函数的参数类型和参数个数，首先检查是否包含第一个选项。如果是，则将第一个选项的参数作为起始值，并将该选项标记为已选状态。
2. 如果包含第一个选项，则检查是否包含第二个选项。如果是，则将第二个选项的参数作为起始值，并将该选项标记为已选状态。
3. 重复步骤 2，直到所有的选项都被处理完毕。
4. 将已选状态的参数拼接起来，输出结果。

函数输出的错误信息是：

1. 缺少选项参数时，输出“option requires an argument -- %c”。
2. 选项参数名拼接错误时，输出“option requires an argument -- %.*s”。
3. 不存在选项时，输出“option doesn't take an argument -- %.*s”。
4. 选项参数无效时，输出“unknown option -- %c”。


```cpp
static void permute_args(int, int, int, char * const *);

static char *place = EMSG; /* option letter processing */

/* XXX: set optreset to 1 rather than these two */
static int nonopt_start = -1; /* first non option argument (for permute) */
static int nonopt_end = -1;   /* first option after non options (for permute) */

/* Error messages */
static const char recargchar[] = "option requires an argument -- %c";
static const char recargstring[] = "option requires an argument -- %s";
static const char ambig[] = "ambiguous option -- %.*s";
static const char noarg[] = "option doesn't take an argument -- %.*s";
static const char illoptchar[] = "unknown option -- %c";
static const char illoptstring[] = "unknown option -- %s";

```



这段代码定义了两个函数：`_vwarnx` 和 `warnx`。它们都接受两个参数：`const char *fmt` 和一个可变参数 `va_list ap`。

`_vwarnx` 函数的作用是在警告信息中使用格式化字符串和可变参数。它的参数是 `const char *fmt` 和一个可变参数 `va_list ap`。首先，它将 `fmt` 中的字符串打印到控制台中的最后一行。然后，它使用 `vfprintf` 函数将 `fmt` 和 `ap` 组成的字符串打印到控制台中的最后一行。最后，它打印一个换行符。

`warnx` 函数的作用与 `_vwarnx` 函数类似，但是它的输出形式更简单。它也接受一个可变参数 `va_list ap`。首先，它将 `fmt` 中的字符串打印到控制台中的最后一行。然后，它使用 `fprintf` 函数将 `fmt` 和 `ap` 组成的字符串打印到控制台中的最后一行。最后，它不打印换行符。

这两个函数的主要区别在于输出方式。`_vwarnx` 函数使用 `fprintf` 函数将格式化字符串打印到控制台中的最后一行，然后使用 `vfprintf` 函数将 `fmt` 和 `ap` 组成的字符串打印到控制台中的最后一行。这种形式可能会更易阅读，但 `vfprintf` 函数可能因为无法正确解析传入的格式化字符串而导致一些错误。而 `warnx` 函数直接使用 `fprintf` 函数将 `fmt` 和 `ap` 组成的字符串打印到控制台中的最后一行，没有额外的错误风险，但这种形式可能不够灵活，可能会因为缺少其他可选的输出方式而导致一些问题。


```cpp
static void
_vwarnx(const char *fmt,va_list ap)
{
  (void)fprintf(stderr,"%s: ",__progname);
  if (fmt != NULL)
    (void)vfprintf(stderr,fmt,ap);
  (void)fprintf(stderr,"\n");
}

static void
warnx(const char *fmt,...)
{
  va_list ap;
  va_start(ap,fmt);
  _vwarnx(fmt,ap);
  va_end(ap);
}

```

这段代码定义了一个名为 `gcd` 的函数，它的参数为两个整数 `a` 和 `b`。该函数计算 `a` 和 `b` 之间的最大公因数。

函数的实现过程如下：首先取两个数中的较小值（取模），然后将这个较小的值不断赋值给较大的值，直到两个值相等或者其中一个值为0。在每次赋值过程中，将得到的新的较大值（即原来的较小值）除以原来的较小值（即新的较大值），得到的商就是当前最小公因数。最后返回最小公因数。

这段代码定义的 `gcd` 函数可以高效地计算两个整数之间的最大公因数，并且可以在编译时进行优化，避免了不必要的计算开销。


```cpp
/*
 * Compute the greatest common divisor of a and b.
 */
static int
gcd(int a, int b)
{
	int c;

	c = a % b;
	while (c != 0) {
		a = b;
		b = c;
		c = a % b;
	}

	return (b);
}

```

这段代码定义了一个名为 permute_args 的函数，它的作用是交换非优选开始到非优选结束以及从非优选结束到优选结束的块。函数的参数包括一个整数数组 `nargv`，该数组包含了程序运行时传递给程序的命令行参数。

函数的实现采用以下步骤：

1. 根据输入参数中的 `nargv` 计算出非优选块和优选块的数量以及 cycles 数组。非优选块的数量为 `nnonopts`，优选块的数量为 `nopts`，cycles 数组用于记录每个非优选块的起始和结束位置，起始位置从非优选结束标志开始，结束位置从 `opt_end` 开始（不包括该标志），以保证循环的顺序正确）。
2. 初始化 swap 变量，用于保存当前正在交换的参数。
3. 遍历 cycles 数组中的每个块，从非优选开始，计算出该块的起始和结束位置，交换参数位置，并更新 swap 变量。
4. 循环结束后，交换参数位置，使得参数能够正确循环回到输入的参数数组中。

由于函数中使用了 `const` 修饰的变量，`permute_args` 的实现对于传入的参数不进行修改，仅在函数内部输出交换后的参数。


```cpp
/*
 * Exchange the block from nonopt_start to nonopt_end with the block
 * from nonopt_end to opt_end (keeping the same order of arguments
 * in each block).
 */
static void
permute_args(int panonopt_start, int panonopt_end, int opt_end,
	char * const *nargv)
{
	int cstart, cyclelen, i, j, ncycle, nnonopts, nopts, pos;
	char *swap;

	/*
	 * compute lengths of blocks and number and size of cycles
	 */
	nnonopts = panonopt_end - panonopt_start;
	nopts = opt_end - panonopt_end;
	ncycle = gcd(nnonopts, nopts);
	cyclelen = (opt_end - panonopt_start) / ncycle;

	for (i = 0; i < ncycle; i++) {
		cstart = panonopt_end+i;
		pos = cstart;
		for (j = 0; j < cyclelen; j++) {
			if (pos >= panonopt_end)
				pos -= nnonopts;
			else
				pos += nopts;
			swap = nargv[pos];
			/* LINTED const cast */
			((char **) nargv)[pos] = nargv[cstart];
			/* LINTED const cast */
			((char **)nargv)[cstart] = swap;
		}
	}
}

```

这段代码定义了一个名为`getopt`的函数，它是`<stdio.h>`中的一部分，用于解析命令行参数。函数的作用是接受一个`nargc`表示参数数量，一个`nargv`表示参数列表，以及一个`options`表示选项字符串。函数内部使用`getopt_internal`函数来解析选项，如果没有传递`FLAG_PERMUTE`参数，将会覆盖`<getopt.h>`中提供的实现。选项字符串中的`'-'`表示忽略参数的前缀。如果解析失败，函数将返回`-1`并设置`errno`为`ENOTSUPPORT`。


```cpp
#ifdef REPLACE_GETOPT
/*
 * getopt --
 *	Parse argc/argv argument vector.
 *
 * [eventually this will replace the BSD getopt]
 */
int
getopt(int nargc, char * const *nargv, const char *options)
{

	/*
	 * We don't pass FLAG_PERMUTE to getopt_internal() since
	 * the BSD getopt(3) (unlike GNU) has never done this.
	 *
	 * Furthermore, since many privileged programs call getopt()
	 * before dropping privileges it makes sense to keep things
	 * as simple (and bug-free) as possible.
	 */
	return (getopt_internal(nargc, nargv, options, NULL, NULL, 0));
}
```

这段代码是一个C语言编译器扩展选项。它通过引入了一个名为“_BSD_SOURCE”的头文件，告诉编译器在编译执行之前需要包含一些非标准的函数和定义。

通过分析代码，我们可以看到以下内容：

1. 定义了一个名为“optreset”的函数，该函数在选项参数解析之前被定义。
2. 通过包含“_BSD_SOURCE”头文件，告诉编译器在解析选项参数时，使用“__mingw_optreset”函数，而不是标准的“getopt”函数。
3. 在“#ifdef _BSD_SOURCE”和“#endif”注释中，有一个名为“optreset”的定义，它是通过“__mingw_optreset”实现的。
4. 通过“#ifdef __cplusplus”注释，该头文件可以被包含。


```cpp
#endif /* REPLACE_GETOPT */

//extern int getopt(int nargc, char * const *nargv, const char *options);

#ifdef _BSD_SOURCE
/*
 * BSD adds the non-standard `optreset' feature, for reinitialisation
 * of `getopt' parsing.  We support this feature, for applications which
 * proclaim their BSD heritage, before including this header; however,
 * to maintain portability, developers are advised to avoid it.
 */
# define optreset  __mingw_optreset
extern int optreset;
#endif
#ifdef __cplusplus
}
```

这段代码是一个POSIX头文件，用于指定GNU C库中的`getopt' API。`#endif`是一个预编译指令，它会关闭`unistd.h`头文件中预先定义的`__GETOPT_H__`声明块，并在需要时打开一个名为`__GETOPT_LONG_H__`的声明块。

`#if !defined(__UNISTD_H_SOURCED__) && !defined(__GETOPT_LONG_H__)`是一个条件编译语句，它会检查两个标志是否被定义：`__UNISTD_H_SOURCED__`和`__GETOPT_LONG_H__`。如果两个标志都不被定义，那么它会执行`__GETOPT_LONG_H__`的定义，否则不会执行。

如果`__UNISTD_H_SOURCED__`被定义，但是`__GETOPT_LONG_H__`没有被定义，那么该头文件不会包含`__GETOPT_LONG_H__`的定义，从而不会覆盖`unistd.h`头文件中预先定义的`__GETOPT_LONG_H__`声明块。这样，用户就需要手动下载并包含`__GETOPT_LONG_H__`头文件，才能在他们的应用程序中使用`getopt' API`。


```cpp
#endif
/*
 * POSIX requires the `getopt' API to be specified in `unistd.h';
 * thus, `unistd.h' includes this header.  However, we do not want
 * to expose the `getopt_long' or `getopt_long_only' APIs, when
 * included in this manner.  Thus, close the standard __GETOPT_H__
 * declarations block, and open an additional __GETOPT_LONG_H__
 * specific block, only when *not* __UNISTD_H_SOURCED__, in which
 * to declare the extended API.
 */
#endif /* !defined(__GETOPT_H__) */

#if !defined(__UNISTD_H_SOURCED__) && !defined(__GETOPT_LONG_H__)
#define __GETOPT_LONG_H__

```

这段代码定义了一个名为"option"的结构体，用于表示命令行选项。该结构体包含一个名为"name"的常量整数类型的变量，表示选项的名称，以及一个名为"has_arg"的整数类型的变量，表示该选项是否需要一个参数。还有一个名为"flag"的整数类型的变量，表示该选项的状态，可以存储为零或指向某个对象。最后，"val"是一个整数类型的变量，表示该选项对应的实际值。

该代码接下来定义了一个名为"option_enum"的枚举类型，用于定义命令行选项的值。该枚举类型包含三个枚举成员，分别对应"no_argument"、"required_argument"和"optional_argument"的值。

该代码最后没有做任何其他事情，但是定义了两个变量，用于声明一个选项结构体和一个枚举类型，分别用于存储命令行选项的名称和值。


```cpp
#ifdef __cplusplus
extern "C" {
#endif

struct option		/* specification for a long form option...	*/
{
  const char *name;		/* option name, without leading hyphens */
  int         has_arg;		/* does it take an argument?		*/
  int        *flag;		/* where to save its status, or NULL	*/
  int         val;		/* its associated status value		*/
};

enum    		/* permitted values for its `has_arg' field...	*/
{
  no_argument = 0,      	/* option never takes an argument	*/
  required_argument,		/* option always requires an argument	*/
  optional_argument		/* option may take an argument		*/
};

```

It looks like this is a code snippet that processes command line options. The `long_options` array appears to keep track of all the available options and their corresponding arguments. The `match` variable seems to keep track of which argument the user is requesting.

The code first checks if the user has requested the `-h` or `--help` option. If it has, it prints out the help menu and returns a value indicating that the option is not recognized.

If the user has requested an option that has a valid argument, the code checks if the required argument has already been specified. If it has, the code sets the `optarg` variable to the argument value and continues processing. If the required argument has not been specified, the code checks if the `-o` or `--opt` option has been specified. If it has, the code sets the `optind` variable to the next available position in the `long_options` array and continues processing.

If the user has requested an option that does not have a required argument or a `-o` option, the code sets the `optind` variable to the next available position in the `long_options` array and returns a value indicating that the option is not recognized.

If the user has requested an option that is not a valid option, the code prints out an error message and returns a value indicating that the option is not recognized.

If the user has requested the `-h` or `--help` option, the code prints out the help menu and returns a value indicating that the option is not recognized.

If the user has not specified any options, the code returns a value indicating that the option is not recognized.


```cpp
/*
 * parse_long_options --
 *	Parse long options in argc/argv argument vector.
 * Returns -1 if short_too is set and the option does not match long_options.
 */
static int
parse_long_options(char * const *nargv, const char *options,
	const struct option *long_options, int *idx, int short_too)
{
	char *current_argv, *has_equal;
	size_t current_argv_len;
	int i, ambiguous, match;

#define IDENTICAL_INTERPRETATION(_x, _y)                                \
	(long_options[(_x)].has_arg == long_options[(_y)].has_arg &&    \
	 long_options[(_x)].flag == long_options[(_y)].flag &&          \
	 long_options[(_x)].val == long_options[(_y)].val)

	current_argv = place;
	match = -1;
	ambiguous = 0;

	optind++;

	if ((has_equal = strchr(current_argv, '=')) != NULL) {
		/* argument found (--option=arg) */
		current_argv_len = has_equal - current_argv;
		has_equal++;
	} else
		current_argv_len = strlen(current_argv);

	for (i = 0; long_options[i].name; i++) {
		/* find matching long option */
		if (strncmp(current_argv, long_options[i].name,
		    current_argv_len))
			continue;

		if (strlen(long_options[i].name) == current_argv_len) {
			/* exact match */
			match = i;
			ambiguous = 0;
			break;
		}
		/*
		 * If this is a known short option, don't allow
		 * a partial match of a single character.
		 */
		if (short_too && current_argv_len == 1)
			continue;

		if (match == -1)	/* partial match */
			match = i;
		else if (!IDENTICAL_INTERPRETATION(i, match))
			ambiguous = 1;
	}
	if (ambiguous) {
		/* ambiguous abbreviation */
		if (PRINT_ERROR)
			warnx(ambig, (int)current_argv_len,
			     current_argv);
		optopt = 0;
		return (BADCH);
	}
	if (match != -1) {		/* option found */
		if (long_options[match].has_arg == no_argument
		    && has_equal) {
			if (PRINT_ERROR)
				warnx(noarg, (int)current_argv_len,
				     current_argv);
			/*
			 * XXX: GNU sets optopt to val regardless of flag
			 */
			if (long_options[match].flag == NULL)
				optopt = long_options[match].val;
			else
				optopt = 0;
			return (BADARG);
		}
		if (long_options[match].has_arg == required_argument ||
		    long_options[match].has_arg == optional_argument) {
			if (has_equal)
				optarg = has_equal;
			else if (long_options[match].has_arg ==
			    required_argument) {
				/*
				 * optional argument doesn't use next nargv
				 */
				optarg = nargv[optind++];
			}
		}
		if ((long_options[match].has_arg == required_argument)
		    && (optarg == NULL)) {
			/*
			 * Missing argument; leading ':' indicates no error
			 * should be generated.
			 */
			if (PRINT_ERROR)
				warnx(recargstring,
				    current_argv);
			/*
			 * XXX: GNU sets optopt to val regardless of flag
			 */
			if (long_options[match].flag == NULL)
				optopt = long_options[match].val;
			else
				optopt = 0;
			--optind;
			return (BADARG);
		}
	} else {			/* unknown option */
		if (short_too) {
			--optind;
			return (-1);
		}
		if (PRINT_ERROR)
			warnx(illoptstring, current_argv);
		optopt = 0;
		return (BADCH);
	}
	if (idx)
		*idx = match;
	if (long_options[match].flag) {
		*long_options[match].flag = long_options[match].val;
		return (0);
	} else
		return (long_options[match].val);
```

这段代码定义了一个名为“getopt_internal”的函数，用于解析命令行参数。

getopt_internal函数接受四个参数：

1. nargc：参数数量，包括命令行参数和选项。
2. nargv：参数列表，包括命令行参数。
3. options：选项字符串，包含用户定义的选项。
4. long_options：long选项数组，包含长期选项。
5. idx：下标，指向第一个参数。
6. flags：标志，用于指示解析选项前缀的匹配程度。

函数内部首先检查选项数组是否为空，如果为空，则返回-1。然后，根据选项数组中的元素，执行相应的操作。

具体来说，函数首先设置opts翻转，然后检查设置的选项是否为`-`或`+`。如果是，则设置相应的标志。接下来，设置非选项参数的起始和结束位置。

最后，函数内部根据opts翻转设置标志，以指示解析选项前缀的匹配程度。如果opts数组中不包含`-`或`+`选项，则函数将始终返回`-1`。


```cpp
#undef IDENTICAL_INTERPRETATION
}

/*
 * getopt_internal --
 *	Parse argc/argv argument vector.  Called by user level routines.
 */
static int
getopt_internal(int nargc, char * const *nargv, const char *options,
	const struct option *long_options, int *idx, int flags)
{
	char *oli;				/* option letter list index */
	int optchar, short_too;
	static int posixly_correct = -1;

	if (options == NULL)
		return (-1);

	/*
	 * XXX Some GNU programs (like cvs) set optind to 0 instead of
	 * XXX using optreset.  Work around this braindamage.
	 */
	if (optind == 0)
		optind = optreset = 1;

	/*
	 * Disable GNU extensions if POSIXLY_CORRECT is set or options
	 * string begins with a '+'.
	 *
	 * CV, 2009-12-14: Check POSIXLY_CORRECT anew if optind == 0 or
	 *                 optreset != 0 for GNU compatibility.
	 */
	if (posixly_correct == -1 || optreset != 0)
		posixly_correct = (getenv("POSIXLY_CORRECT") != NULL);
	if (*options == '-')
		flags |= FLAG_ALLARGS;
	else if (posixly_correct || *options == '+')
		flags &= ~FLAG_PERMUTE;
	if (*options == '+' || *options == '-')
		options++;

	optarg = NULL;
	if (optreset)
		nonopt_start = nonopt_end = -1;
```

This is a program that appears to print the specified option characters as an error message. If the user specifies an option with the format `-O optstring`, the program will attempt to interpret the option string using the specified `optstring` and return the corresponding error code.

The program takes an optional number of command-line options using the format `-O optstring:arg描述`, where `optstring` is the option name and `arg` is the option description. The description `arg` is passed as the second argument.

If the user specifies the option with the format `-W long-option`, the program will attempt to interpret the option string as a `long-option` and return the corresponding error code.

If the user-specified option is `-W long-option` and the specified option is not `-W` or `long-option`, the program will return the error code returned by `syscall_errno()` with the message "illegal option type".

If the user specifies an option with the format `-W long-option:option_string:desc`, the program will attempt to interpret the option string as a `long-option` and return the corresponding error code. If the specified option is not a valid `long-option`, the program will return the error code returned by `syscall_errno()` with the message "illegal option type".


```cpp
start:
	if (optreset || !*place) {		/* update scanning pointer */
		optreset = 0;
		if (optind >= nargc) {          /* end of argument vector */
			place = EMSG;
			if (nonopt_end != -1) {
				/* do permutation, if we have to */
				permute_args(nonopt_start, nonopt_end,
				    optind, nargv);
				optind -= nonopt_end - nonopt_start;
			}
			else if (nonopt_start != -1) {
				/*
				 * If we skipped non-options, set optind
				 * to the first of them.
				 */
				optind = nonopt_start;
			}
			nonopt_start = nonopt_end = -1;
			return (-1);
		}
		if (*(place = nargv[optind]) != '-' ||
		    (place[1] == '\0' && strchr(options, '-') == NULL)) {
			place = EMSG;		/* found non-option */
			if (flags & FLAG_ALLARGS) {
				/*
				 * GNU extension:
				 * return non-option as argument to option 1
				 */
				optarg = nargv[optind++];
				return (INORDER);
			}
			if (!(flags & FLAG_PERMUTE)) {
				/*
				 * If no permutation wanted, stop parsing
				 * at first non-option.
				 */
				return (-1);
			}
			/* do permutation */
			if (nonopt_start == -1)
				nonopt_start = optind;
			else if (nonopt_end != -1) {
				permute_args(nonopt_start, nonopt_end,
				    optind, nargv);
				nonopt_start = optind -
				    (nonopt_end - nonopt_start);
				nonopt_end = -1;
			}
			optind++;
			/* process next argument */
			goto start;
		}
		if (nonopt_start != -1 && nonopt_end == -1)
			nonopt_end = optind;

		/*
		 * If we have "-" do nothing, if "--" we are done.
		 */
		if (place[1] != '\0' && *++place == '-' && place[1] == '\0') {
			optind++;
			place = EMSG;
			/*
			 * We found an option (--), so if we skipped
			 * non-options, we have to permute.
			 */
			if (nonopt_end != -1) {
				permute_args(nonopt_start, nonopt_end,
				    optind, nargv);
				optind -= nonopt_end - nonopt_start;
			}
			nonopt_start = nonopt_end = -1;
			return (-1);
		}
	}

	/*
	 * Check long options if:
	 *  1) we were passed some
	 *  2) the arg is not just "-"
	 *  3) either the arg starts with -- we are getopt_long_only()
	 */
	if (long_options != NULL && place != nargv[optind] &&
	    (*place == '-' || (flags & FLAG_LONGONLY))) {
		short_too = 0;
		if (*place == '-')
			place++;		/* --foo long option */
		else if (*place != ':' && strchr(options, *place) != NULL)
			short_too = 1;		/* could be short option too */

		optchar = parse_long_options(nargv, options, long_options,
		    idx, short_too);
		if (optchar != -1) {
			place = EMSG;
			return (optchar);
		}
	}

	if ((optchar = (int)*place++) == (int)':' ||
	    (optchar == (int)'-' && *place != '\0') ||
	    (oli = (char*)strchr(options, optchar)) == NULL) {
		/*
		 * If the user specified "-" and  '-' isn't listed in
		 * options, return -1 (non-option) as per POSIX.
		 * Otherwise, it is an unknown option character (or ':').
		 */
		if (optchar == (int)'-' && *place == '\0')
			return (-1);
		if (!*place)
			++optind;
		if (PRINT_ERROR)
			warnx(illoptchar, optchar);
		optopt = optchar;
		return (BADCH);
	}
	if (long_options != NULL && optchar == 'W' && oli[1] == ';') {
		/* -W long-option */
		if (*place)			/* no space */
			/* NOTHING */;
		else if (++optind >= nargc) {	/* no arg */
			place = EMSG;
			if (PRINT_ERROR)
				warnx(recargchar, optchar);
			optopt = optchar;
			return (BADARG);
		} else				/* white space */
			place = nargv[optind];
		optchar = parse_long_options(nargv, options, long_options,
		    idx, 0);
		place = EMSG;
		return (optchar);
	}
	if (*++oli != ':') {			/* doesn't take argument */
		if (!*place)
			++optind;
	} else {				/* takes (optional) argument */
		optarg = NULL;
		if (*place)			/* no white space */
			optarg = place;
		else if (oli[1] != ':') {	/* arg not optional */
			if (++optind >= nargc) {	/* no arg */
				place = EMSG;
				if (PRINT_ERROR)
					warnx(recargchar, optchar);
				optopt = optchar;
				return (BADARG);
			} else
				optarg = nargv[optind];
		}
		place = EMSG;
		++optind;
	}
	/* dump back option letter */
	return (optchar);
}

```

这两段代码定义了一个名为`getopt_long`的函数，用于解析命令行参数选项。函数接受四个参数：

1. `nargc`：参数数量，即命令行参数的数量。
2. `nargv`：参数列表，包含`nargc`个参数。
3. `options`：选项参数，可能是命令行参数或选项字符串。
4. `long_options`：long选项参数，可能是选项字符串或选项列表。
5. `idx`：下标，用于标识已经解析过的选项。

函数内部使用`getopt_internal`函数，根据输入参数的格式进行解析。如果遇到无效的选项，函数可能会直接返回`EOF`并返回一个负值。而如果成功解析完所有的选项，函数返回的是`0`。


```cpp
/*
 * getopt_long --
 *	Parse argc/argv argument vector.
 */
int
getopt_long(int nargc, char * const *nargv, const char *options,
    const struct option *long_options, int *idx)
{

	return (getopt_internal(nargc, nargv, options, long_options, idx,
	    FLAG_PERMUTE));
}

/*
 * getopt_long_only --
 *	Parse argc/argv argument vector.
 */
```

这段代码定义了一个名为`getopt_long_only`的函数，它是`getopt_internal`函数的别名。这两函数的作用是接收`nargc`个参数，包括命令行参数和选项等，并返回一个整数类型的选项编号。

`getopt_long_only`函数接受四个参数：

1. `nargc`：表示命令行参数的数量。
2. `nargv`：指向命令行参数的指针数组，每个元素都是一个指向字符串的指针。
3. `options`：指向一个字符串的指针，这个字符串被认为是命令行选项。
4. `long_options`：指向一个包含多个选项名的指针，每个选项名是一个指向结构体的指针。
5. `idx`：一个整数类型的变量，用于保存选项编号。

`getopt_long_only`函数首先通过`getopt_internal`函数处理`nargc`个命令行参数，如果处理成功，就返回它的选项编号。如果处理失败，就返回一个错误的选项编号。如果`nargc`小于命令行参数的数量，`getopt_long_only`就会自动尝试使用`long_options`中的选项来填充剩余的参数。

需要注意的是，`getopt_long_only`函数的实现与`getopt_long`函数略有不同，但它主要是为了简化`getopt_long`函数的调用而设计的。


```cpp
int
getopt_long_only(int nargc, char * const *nargv, const char *options,
    const struct option *long_options, int *idx)
{

	return (getopt_internal(nargc, nargv, options, long_options, idx,
	    FLAG_PERMUTE|FLAG_LONGONLY));
}

//extern int getopt_long(int nargc, char * const *nargv, const char *options,
//    const struct option *long_options, int *idx);
//extern int getopt_long_only(int nargc, char * const *nargv, const char *options,
//    const struct option *long_options, int *idx);
/*
 * Previous MinGW implementation had...
 */
```

这段代码是一个C语言编译器的预处理指令，用于定义头文件和预处理指令。它包含两个条件判断，用于检查是否支持 long 函数式编程风格。

第一个条件判断是 `#ifdef __cplusplus`，这是一个预处理指令，用于在编译时检查 C++ 编译器是否支持 long 函数式编程风格。如果这个条件判断为真，那么编译器会编译这个文件，否则不会编译。

第二个条件判断是 `#ifdef __unistd_h_sourced__` 和 `#ifdef __getopt_llong_h__`，这也是两个预处理指令，用于检查是否支持管道（Pipe）输出模式。如果这两个条件判断为真，那么编译器会编译这个文件，否则不会编译。

如果 `#ifdef __cplusplus` 和 `#ifdef __unistd_h_sourced__` 和 `#ifdef __getopt_llong_h__` 中的任意一个或多个条件判断为真，那么编译器会编译 `.h` 文件，并使用预处理指令 `#define __declspec(d伊斯兰兰) 1` 对 `.h` 文件进行定义，定义的名称与头文件名相同，但 `.h` 文件扩展名是 `.d`。

否则，编译器不会编译 `.h` 文件，因此不会生成 `.d` 文件。

总之，这段代码定义了 C 语言编译器的预处理指令，用于定义和输出文件头文件，以及检查是否支持管道输出模式。


```cpp
#ifndef HAVE_DECL_GETOPT
/*
 * ...for the long form API only; keep this for compatibility.
 */
# define HAVE_DECL_GETOPT	1
#endif

#ifdef __cplusplus
}
#endif

#endif /* !defined(__UNISTD_H_SOURCED__) && !defined(__GETOPT_LONG_H__) */

```