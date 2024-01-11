# `xmrig\src\3rdparty\fmt\ostream.h`

```
// C++格式化库的std::ostream支持
//
// 版权所有 (c) 2012 - 现在，Victor Zverovich
// 保留所有权利
//
// 有关许可信息，请参阅format.h。

#ifndef FMT_OSTREAM_H_
#define FMT_OSTREAM_H_

#include <ostream>

#include "format.h"

FMT_BEGIN_NAMESPACE

template <typename Char> class basic_printf_parse_context;
template <typename OutputIt, typename Char> class basic_printf_context;

namespace detail {

template <class Char> class formatbuf : public std::basic_streambuf<Char> {
 private:
  using int_type = typename std::basic_streambuf<Char>::int_type;
  using traits_type = typename std::basic_streambuf<Char>::traits_type;

  buffer<Char>& buffer_;

 public:
  formatbuf(buffer<Char>& buf) : buffer_(buf) {}

 protected:
  // put-area实际上总是空的。这使得实现更简单，并且有一个优势，即streambuf和buffer始终同步，并且sputc永远不会写入未初始化的内存。显而易见的缺点是，每次调用sputc总是导致(虚拟)调用overflow。这里对于sputn没有缺点，因为这总是导致调用xsputn。

  int_type overflow(int_type ch = traits_type::eof()) FMT_OVERRIDE {
    if (!traits_type::eq_int_type(ch, traits_type::eof()))
      buffer_.push_back(static_cast<Char>(ch));
    return ch;
  }

  std::streamsize xsputn(const Char* s, std::streamsize count) FMT_OVERRIDE {
    buffer_.append(s, s + count);
    return count;
  }
};

struct converter {
  template <typename T, FMT_ENABLE_IF(is_integral<T>::value)> converter(T);
};

template <typename Char> struct test_stream : std::basic_ostream<Char> {
 private:
  void_t<> operator<<(converter);
};

// 隐藏内置类型的插入操作符。
template <typename Char, typename Traits>
void_t<> operator<<(std::basic_ostream<Char, Traits>&, Char);
template <typename Char, typename Traits>
void_t<> operator<<(std::basic_ostream<Char, Traits>&, char);
// 重载运算符<<，使得可以将字符写入到输出流中
template <typename Traits>
void_t<> operator<<(std::basic_ostream<char, Traits>&, char);
// 重载运算符<<，使得可以将有符号字符写入到输出流中
template <typename Traits>
void_t<> operator<<(std::basic_ostream<char, Traits>&, signed char);
// 重载运算符<<，使得可以将无符号字符写入到输出流中
template <typename Traits>
void_t<> operator<<(std::basic_ostream<char, Traits>&, unsigned char);

// 检查类型T是否具有用户定义的operator<<（例如不是std::ostream的成员）
template <typename T, typename Char> class is_streamable {
 private:
  // 检查是否存在重载的operator<<
  template <typename U>
  static bool_constant<!std::is_same<decltype(std::declval<test_stream<Char>&>()
                                              << std::declval<U>()),
                                     void_t<>>::value>
  test(int);
  // 默认情况下返回false_type
  template <typename> static std::false_type test(...);
  // 使用test函数的返回类型
  using result = decltype(test<T>(0));

 public:
  // 检查结果的值
  static const bool value = result::value;
};

// 将buf的内容写入到os中
template <typename Char>
void write_buffer(std::basic_ostream<Char>& os, buffer<Char>& buf) {
  const Char* buf_data = buf.data();
  using unsigned_streamsize = std::make_unsigned<std::streamsize>::type;
  unsigned_streamsize size = buf.size();
  unsigned_streamsize max_size = to_unsigned(max_value<std::streamsize>());
  do {
    unsigned_streamsize n = size <= max_size ? size : max_size;
    os.write(buf_data, static_cast<std::streamsize>(n));
    buf_data += n;
    size -= n;
  } while (size != 0);
}

// 格式化值并将其写入到buf中
template <typename Char, typename T>
void format_value(buffer<Char>& buf, const T& value,
                  locale_ref loc = locale_ref()) {
  formatbuf<Char> format_buf(buf);
  std::basic_ostream<Char> output(&format_buf);
  // 如果未定义FMT_STATIC_THOUSANDS_SEPARATOR，则使用指定的locale
  #if !defined(FMT_STATIC_THOUSANDS_SEPARATOR)
  if (loc) output.imbue(loc.get<std::locale>());
  #endif
  output << value;
  output.exceptions(std::ios_base::failbit | std::ios_base::badbit);
  buf.try_resize(buf.size());
}

// 格式化具有重载的ostream operator<<的类型T的对象
template <typename T, typename Char>
namespace detail {
// 定义一个通用的格式化器，用于处理可流输出的类型
template <typename T, typename Char, enable_if_t<is_streamable<T, Char>::value>>
struct fallback_formatter : private formatter<basic_string_view<Char>, Char> {
  // 解析格式化字符串
  FMT_CONSTEXPR auto parse(basic_format_parse_context<Char>& ctx)
      -> decltype(ctx.begin()) {
    return formatter<basic_string_view<Char>, Char>::parse(ctx);
  }
  // 解析 printf 格式化字符串
  template <typename ParseCtx,
            FMT_ENABLE_IF(std::is_same<
                          ParseCtx, basic_printf_parse_context<Char>>::value)>
  auto parse(ParseCtx& ctx) -> decltype(ctx.begin()) {
    return ctx.begin();
  }

  // 格式化输出到基本格式上下文
  template <typename OutputIt>
  auto format(const T& value, basic_format_context<OutputIt, Char>& ctx)
      -> OutputIt {
    // 创建字符缓冲区
    basic_memory_buffer<Char> buffer;
    // 格式化值到缓冲区
    format_value(buffer, value, ctx.locale());
    // 创建基本字符串视图
    basic_string_view<Char> str(buffer.data(), buffer.size());
    // 调用基本格式化器的格式化方法
    return formatter<basic_string_view<Char>, Char>::format(str, ctx);
  }
  // 格式化输出到 printf 格式上下文
  template <typename OutputIt>
  auto format(const T& value, basic_printf_context<OutputIt, Char>& ctx)
      -> OutputIt {
    // 创建字符缓冲区
    basic_memory_buffer<Char> buffer;
    // 格式化值到缓冲区
    format_value(buffer, value, ctx.locale());
    // 将缓冲区内容复制到输出流
    return std::copy(buffer.begin(), buffer.end(), ctx.out());
  }
};
}  // namespace detail

// 打印格式化数据到流 os
template <typename Char>
void vprint(std::basic_ostream<Char>& os, basic_string_view<Char> format_str,
            basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  // 创建字符缓冲区
  basic_memory_buffer<Char> buffer;
  // 格式化数据到缓冲区
  detail::vformat_to(buffer, format_str, args);
  // 将缓冲区内容写入输出流
  detail::write_buffer(os, buffer);
}

/**
  \rst
  打印格式化数据到流 *os*。

  **Example**::

    fmt::print(cerr, "Don't {}!", "panic");
  \endrst
 */
// 打印格式化数据到流 os
template <typename S, typename... Args,
          typename Char = enable_if_t<detail::is_string<S>::value, char_t<S>>>
void print(std::basic_ostream<Char>& os, const S& format_str, Args&&... args) {
  // 调用 vprint 函数，将格式化字符串和参数转换为视图，并进行格式化输出
  vprint(os, to_string_view(format_str),
         fmt::make_args_checked<Args...>(format_str, args...));
}
FMT_END_NAMESPACE

#endif  // FMT_OSTREAM_H_
```