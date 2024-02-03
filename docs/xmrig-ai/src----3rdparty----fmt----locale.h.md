# `xmrig\src\3rdparty\fmt\locale.h`

```cpp
// 包含 C++ 的 locale 库
#include <locale>
// 包含自定义的 format.h 头文件
#include "format.h"

// 进入 fmt 命名空间
FMT_BEGIN_NAMESPACE

// 进入 detail 命名空间
namespace detail {
  // 格式化输出到指定缓冲区，使用指定的 locale
  template <typename Char>
  typename buffer_context<Char>::iterator vformat_to(
      const std::locale& loc, buffer<Char>& buf,
      basic_string_view<Char> format_str,
      basic_format_args<buffer_context<type_identity_t<Char>>> args) {
    using af = arg_formatter<typename buffer_context<Char>::iterator, Char>;
    return vformat_to<af>(buffer_appender<Char>(buf), to_string_view(format_str),
                          args, detail::locale_ref(loc));
  }

  // 格式化输出到字符串，使用指定的 locale
  template <typename Char>
  std::basic_string<Char> vformat(
      const std::locale& loc, basic_string_view<Char> format_str,
      basic_format_args<buffer_context<type_identity_t<Char>>> args) {
    basic_memory_buffer<Char> buffer;
    detail::vformat_to(loc, buffer, format_str, args);
    return fmt::to_string(buffer);
  }
}  // namespace detail

// 格式化输出到字符串，使用指定的 locale
template <typename S, typename Char = char_t<S>>
inline std::basic_string<Char> vformat(
    const std::locale& loc, const S& format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  return detail::vformat(loc, to_string_view(format_str), args);
}

// 格式化输出到字符串，使用指定的 locale
template <typename S, typename... Args, typename Char = char_t<S>>
inline std::basic_string<Char> format(const std::locale& loc,
                                      const S& format_str, Args&&... args) {
  return detail::vformat(
      loc, to_string_view(format_str),
      fmt::make_args_checked<Args...>(format_str, args...));
}

// 格式化输出到输出迭代器，使用指定的 locale
template <typename S, typename OutputIt, typename... Args,
          typename Char = enable_if_t<
              detail::is_output_iterator<OutputIt>::value, char_t<S>>>
inline OutputIt vformat_to(
    OutputIt out, const std::locale& loc, const S& format_str,
    # 定义函数，接受参数为格式化字符串、输出流和其他参数
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  # 获取输出流的缓冲区
  decltype(detail::get_buffer<Char>(out)) buf(detail::get_buffer_init(out));
  # 定义参数格式化器
  using af =
    detail::arg_formatter<typename buffer_context<Char>::iterator, Char>;
  # 格式化字符串并将结果输出到缓冲区
  vformat_to<af>(detail::buffer_appender<Char>(buf), to_string_view(format_str),
                 args, detail::locale_ref(loc));
  # 返回格式化后的迭代器
  return detail::get_iterator(buf);
// 结束 fmt 命名空间
FMT_END_NAMESPACE

// 结束 if 语句块
#endif  // FMT_LOCALE_H_
```