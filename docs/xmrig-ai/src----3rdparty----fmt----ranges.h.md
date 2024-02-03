# `xmrig\src\3rdparty\fmt\ranges.h`

```cpp
// C++格式化库的范围支持 - 实验性范围支持
//
// 版权所有，Victor Zverovich，2012年至今
//
// 有关许可信息，请参阅format.h。
//
// 版权所有，Remotion (Igor Schulz)，2018年至今
// 保留所有权利
// {fmt} 对范围、容器和类型元组接口的支持。

#ifndef FMT_RANGES_H_
#define FMT_RANGES_H_

#include <initializer_list>
#include <type_traits>

#include "format.h"

// 仅输出范围中的前N个项目。
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

template <typename Char, typename Enable = void>
struct formatting_range : formatting_base<Char> {
  static FMT_CONSTEXPR_DECL const size_t range_length_limit =
      FMT_RANGE_OUTPUT_LENGTH_LIMIT;  // 仅输出范围中的前N个项目。
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

namespace detail {

template <typename RangeT, typename OutputIterator>
OutputIterator copy(const RangeT& range, OutputIterator out) {
  for (auto it = range.begin(), end = range.end(); it != end; ++it)
    *out++ = *it;
  return out;
}

template <typename OutputIterator>
// 从输入的字符串复制字符到输出迭代器，直到遇到字符串结束符
OutputIterator copy(const char* str, OutputIterator out) {
  while (*str) *out++ = *str++;  // 遍历字符串，将每个字符复制到输出迭代器中
  return out;  // 返回输出迭代器
}

// 将单个字符复制到输出迭代器中
template <typename OutputIterator>
OutputIterator copy(char ch, OutputIterator out) {
  *out++ = ch;  // 将字符复制到输出迭代器中
  return out;  // 返回输出迭代器
}

/// 如果 T 具有 std::string 接口（如 std::string_view），返回 true
template <typename T> class is_like_std_string {
  template <typename U>
  static auto check(U* p)
      -> decltype((void)p->find('a'), p->length(), (void)p->data(), int());  // 检查是否具有 find、length 和 data 方法
  template <typename> static void check(...);  // 如果没有上述方法，则使用省略号版本

 public:
  static FMT_CONSTEXPR_DECL const bool value =
      is_string<T>::value || !std::is_void<decltype(check<T>(nullptr))>::value;  // 如果是字符串类型或者 check 方法返回值不是 void，则返回 true
};

// 如果是 fmt::basic_string_view 类型，则返回 true
template <typename Char>
struct is_like_std_string<fmt::basic_string_view<Char>> : std::true_type {};

// 如果 T 不是范围类型，则返回 false
template <typename... Ts> struct conditional_helper {};

template <typename T, typename _ = void> struct is_range_ : std::false_type {};

#if !FMT_MSC_VER || FMT_MSC_VER > 1800
template <typename T>
struct is_range_<
    T, conditional_t<false,
                     conditional_helper<decltype(std::declval<T>().begin()),
                                        decltype(std::declval<T>().end())>,
                     void>> : std::true_type {};  // 如果 T 有 begin 和 end 方法，则返回 true
#endif

/// 检查是否具有 tuple_size 和 tuple_element 方法
template <typename T> class is_tuple_like_ {
  template <typename U>
  static auto check(U* p) -> decltype(std::tuple_size<U>::value, int());  // 检查是否具有 tuple_size 方法
  template <typename> static void check(...);  // 如果没有上述方法，则使用省略号版本

 public:
  static FMT_CONSTEXPR_DECL const bool value =
      !std::is_void<decltype(check<T>(nullptr))>::value;  // 如果 check 方法返回值不是 void，则返回 true
};

// 检查是否具有 integer_sequence
#if defined(__cpp_lib_integer_sequence) || FMT_MSC_VER >= 1900
template <typename T, T... N>
using integer_sequence = std::integer_sequence<T, N...>;
template <size_t... N> using index_sequence = std::index_sequence<N...>;
template <size_t N> using make_index_sequence = std::make_index_sequence<N>;
#else
// 定义一个模板结构体 integer_sequence，接受一个类型参数 T 和一个参数包 N
template <typename T, T... N> struct integer_sequence {
  // 定义 value_type 为类型 T
  using value_type = T;

  // 定义静态成员函数 size，返回参数包 N 的大小
  static FMT_CONSTEXPR size_t size() { return sizeof...(N); }
};

// 定义模板别名 index_sequence，接受一个参数包 N
template <size_t... N> using index_sequence = integer_sequence<size_t, N...>;

// 定义模板结构体 make_integer_sequence，接受一个类型参数 T，一个大小参数 N，和一个参数包 Ns
template <typename T, size_t N, T... Ns>
struct make_integer_sequence : make_integer_sequence<T, N - 1, N - 1, Ns...> {};
// 特化模板结构体 make_integer_sequence，当 N 为 0 时，继承 integer_sequence 结构体
template <typename T, T... Ns>
struct make_integer_sequence<T, 0, Ns...> : integer_sequence<T, Ns...> {};

// 定义模板别名 make_index_sequence，接受一个大小参数 N
template <size_t N>
using make_index_sequence = make_integer_sequence<size_t, N>;
#endif

// 定义函数 for_each，接受一个 index_sequence，一个元组引用 tup，一个可调用对象 f
template <class Tuple, class F, size_t... Is>
void for_each(index_sequence<Is...>, Tuple&& tup, F&& f) FMT_NOEXCEPT {
  using std::get;
  // 使用可调用对象 get<I>(tup) 现在
  const int _[] = {0, ((void)f(get<Is>(tup)), 0)...};
  (void)_;  // 阻止警告
}

// 定义函数 get_indexes，接受一个元组引用
template <class T>
FMT_CONSTEXPR make_index_sequence<std::tuple_size<T>::value> get_indexes(
    T const&) {
  return {};
}

// 定义函数 for_each，接受一个元组引用 tup，一个可调用对象 f
template <class Tuple, class F> void for_each(Tuple&& tup, F&& f) {
  const auto indexes = get_indexes(tup);
  for_each(indexes, std::forward<Tuple>(tup), std::forward<F>(f));
}

// 定义模板别名 value_type，接受一个类型 Range
template <typename Range>
using value_type = remove_cvref_t<decltype(*std::declval<Range>().begin())>;

// 定义函数 format_str_quoted，接受一个参数 add_space 和一个类型为 Arg 的参数
template <typename Arg, FMT_ENABLE_IF(!is_like_std_string<
                                      typename std::decay<Arg>::type>::value)>
FMT_CONSTEXPR const char* format_str_quoted(bool add_space, const Arg&) {
  return add_space ? " {}" : "{}";
}

// 定义函数 format_str_quoted，接受一个参数 add_space 和一个类型为 Arg 的参数
template <typename Arg, FMT_ENABLE_IF(is_like_std_string<
                                      typename std::decay<Arg>::type>::value)>
FMT_CONSTEXPR const char* format_str_quoted(bool add_space, const Arg&) {
  return add_space ? " \"{}\"" : "\"{}\"";
}

// 定义函数 format_str_quoted，接受一个参数 add_space 和一个类型为 const char* 的参数
FMT_CONSTEXPR const char* format_str_quoted(bool add_space, const char*) {
  return add_space ? " \"{}\"" : "\"{}\"";
}
// 定义函数 format_str_quoted，接受一个参数 add_space 和一个类型为 const wchar_t* 的参数
FMT_CONSTEXPR const wchar_t* format_str_quoted(bool add_space, const wchar_t*) {
  return add_space ? L" \"{}\"" : L"\"{}\"";
}
FMT_CONSTEXPR const char* format_str_quoted(bool add_space, const char) {
  // 如果需要添加空格，则返回带空格的格式化字符串，否则返回不带空格的格式化字符串
  return add_space ? " '{}'" : "'{}'";
}
FMT_CONSTEXPR const wchar_t* format_str_quoted(bool add_space, const wchar_t) {
  // 如果需要添加空格，则返回带空格的格式化字符串，否则返回不带空格的格式化字符串
  return add_space ? L" '{}'" : L"'{}'";
}
}  // namespace detail

template <typename T> struct is_tuple_like {
  // 检查类型是否类似于元组，且不是范围类型
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
      // 格式化每个值，并根据需要添加空格和分隔符
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
// 定义一个模板结构体，用于判断类型T是否为范围类型
template <typename T, typename Char> struct is_range {
  // 判断T是否为范围类型的布尔值
  static FMT_CONSTEXPR_DECL const bool value =
      detail::is_range_<T>::value && !detail::is_like_std_string<T>::value &&
      !std::is_convertible<T, std::basic_string<Char>>::value &&
      !std::is_constructible<detail::std_string_view<Char>, T>::value;
};

// 定义一个模板结构体，用于格式化范围类型T
template <typename T, typename Char>
struct formatter<
    T, Char,
    enable_if_t<fmt::is_range<T, Char>::value
// 解决 MSVC 2017 及更早版本的 bug
#if !FMT_MSC_VER || FMT_MSC_VER >= 1927
                && has_formatter<detail::value_type<T>, format_context>::value
#endif
                >> {
  // 格式化范围类型的对象
  formatting_range<Char> formatting;

  // 解析格式化上下文
  template <typename ParseContext>
  FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
    return formatting.parse(ctx);
  }

  // 格式化范围类型的值
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

// 定义一个模板结构体，用于连接元组类型的参数
template <typename Char, typename... T> struct tuple_arg_join : detail::view {
  const std::tuple<T...>& tuple;
  basic_string_view<Char> sep;

  // 连接元组类型的参数
  tuple_arg_join(const std::tuple<T...>& t, basic_string_view<Char> s)
      : tuple{t}, sep{s} {}
};
// 定义一个结构体模板，用于格式化元组的元素，并以指定的分隔符分隔
template <typename ParseContext>
FMT_CONSTEXPR auto parse(ParseContext& ctx) -> decltype(ctx.begin()) {
  return ctx.begin();
}

// 格式化元组的元素，并将结果输出到指定的上下文中
template <typename FormatContext>
typename FormatContext::iterator format(
    const tuple_arg_join<Char, T...>& value, FormatContext& ctx) {
  return format(value, ctx, detail::make_index_sequence<sizeof...(T)>{});
}

// 格式化元组的元素，并将结果输出到指定的上下文中
template <typename FormatContext, size_t... N>
typename FormatContext::iterator format(
    const tuple_arg_join<Char, T...>& value, FormatContext& ctx,
    detail::index_sequence<N...>) {
  return format_args(value, ctx, std::get<N>(value.tuple)...);
}

// 格式化元组的元素，并将结果输出到指定的上下文中
template <typename FormatContext>
typename FormatContext::iterator format_args(
    const tuple_arg_join<Char, T...>&, FormatContext& ctx) {
  // 注意：对于支持 C++17 的编译器，这个空函数实例化可以用可变参数重载中的 constexpr 分支替换
  return ctx.out();
}

// 格式化元组的元素，并将结果输出到指定的上下文中
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

/**
  \rst
  返回一个对象，用于使用指定的分隔符格式化元组。

  **示例**::

    std::tuple<int, char> t = {1, 'a'};
    fmt::print("{}", fmt::join(t, ", "));
    // 输出: "1, a"
  \endrst
 */
template <typename... T>
FMT_CONSTEXPR tuple_arg_join<char, T...> join(const std::tuple<T...>& tuple,
                                              string_view sep) {
  return {tuple, sep};
}
// 返回一个对象，该对象使用指定的分隔符对初始化列表进行格式化

template <typename... T>
FMT_CONSTEXPR tuple_arg_join<wchar_t, T...> join(const std::tuple<T...>& tuple,
                                                 wstring_view sep) {
  return {tuple, sep};
}

/**
  \rst
  返回一个对象，该对象使用指定的分隔符对初始化列表进行格式化。

  **示例**::

    fmt::print("{}", fmt::join({1, 2, 3}, ", "));
    // 输出: "1, 2, 3"
  \endrst
 */
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