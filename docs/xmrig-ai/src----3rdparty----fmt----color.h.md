# `xmrig\src\3rdparty\fmt\color.h`

```
// 格式化库的C++版本 - 支持颜色
//
// 版权所有 (c) 2018 - 至今，Victor Zverovich 和 fmt 贡献者
// 保留所有权利
//
// 有关许可信息，请参考 format.h。

#ifndef FMT_COLOR_H_
#define FMT_COLOR_H_

#include "format.h"

FMT_BEGIN_NAMESPACE

};                                     // enum class color

// 枚举类，表示终端颜色
enum class terminal_color : uint8_t {
  black = 30,
  red,
  green,
  yellow,
  blue,
  magenta,
  cyan,
  white,
  bright_black = 90,
  bright_red,
  bright_green,
  bright_yellow,
  bright_blue,
  bright_magenta,
  bright_cyan,
  bright_white
};

// 强调枚举类，表示加粗、斜体、下划线、删除线
enum class emphasis : uint8_t {
  bold = 1,
  italic = 1 << 1,
  underline = 1 << 2,
  strikethrough = 1 << 3
};

// rgb 结构体，表示红、绿、蓝颜色
// 使用名称 "rgb" 可以让一些编辑器在工具提示中显示颜色
struct rgb {
  FMT_CONSTEXPR rgb() : r(0), g(0), b(0) {}
  FMT_CONSTEXPR rgb(uint8_t r_, uint8_t g_, uint8_t b_) : r(r_), g(g_), b(b_) {}
  FMT_CONSTEXPR rgb(uint32_t hex)
      : r((hex >> 16) & 0xFF), g((hex >> 8) & 0xFF), b(hex & 0xFF) {}
  FMT_CONSTEXPR rgb(color hex)
      : r((uint32_t(hex) >> 16) & 0xFF),
        g((uint32_t(hex) >> 8) & 0xFF),
        b(uint32_t(hex) & 0xFF) {}
  uint8_t r;
  uint8_t g;
  uint8_t b;
};

namespace detail {

// color 结构体，表示rgb颜色或终端颜色
struct color_type {
  FMT_CONSTEXPR color_type() FMT_NOEXCEPT : is_rgb(), value{} {}
  FMT_CONSTEXPR color_type(color rgb_color) FMT_NOEXCEPT : is_rgb(true),
                                                           value{} {
    value.rgb_color = static_cast<uint32_t>(rgb_color);
  }
  FMT_CONSTEXPR color_type(rgb rgb_color) FMT_NOEXCEPT : is_rgb(true), value{} {
    # 将 RGB 颜色值转换为 32 位整数，分别将 r、g、b 分量左移 16、8、0 位，然后按位或操作
    value.rgb_color = (static_cast<uint32_t>(rgb_color.r) << 16) |
                      (static_cast<uint32_t>(rgb_color.g) << 8) | rgb_color.b;
  }
  # 构造函数，用于初始化终端颜色，设置 is_rgb 为 true，value 初始化为空
  FMT_CONSTEXPR color_type(terminal_color term_color) FMT_NOEXCEPT : is_rgb(),
                                                                     value{} {
    # 将终端颜色值转换为 8 位整数，存储在 value 中
    value.term_color = static_cast<uint8_t>(term_color);
  }
  # 用于标识颜色类型是否为 RGB
  bool is_rgb;
  # 联合体，用于存储终端颜色值或 RGB 颜色值
  union color_union {
    uint8_t term_color;  # 用于存储终端颜色值
    uint32_t rgb_color;  # 用于存储 RGB 颜色值
  } value;
};
}  // namespace detail

// 实验性的文本格式化支持

// 文本样式类
class text_style {
 public:
  // 构造函数，初始化文本样式
  FMT_CONSTEXPR text_style(emphasis em = emphasis()) FMT_NOEXCEPT
      : set_foreground_color(),
        set_background_color(),
        ems(em) {}

  // 重载 |= 运算符
  FMT_CONSTEXPR text_style& operator|=(const text_style& rhs) {
    // 如果前景色未设置，则使用右操作数的前景色
    if (!set_foreground_color) {
      set_foreground_color = rhs.set_foreground_color;
      foreground_color = rhs.foreground_color;
    } else if (rhs.set_foreground_color) {
      // 如果右操作数的前景色已设置，则将两个前景色进行按位或运算
      if (!foreground_color.is_rgb || !rhs.foreground_color.is_rgb)
        FMT_THROW(format_error("can't OR a terminal color"));
      foreground_color.value.rgb_color |= rhs.foreground_color.value.rgb_color;
    }

    // 如果背景色未设置，则使用右操作数的背景色
    if (!set_background_color) {
      set_background_color = rhs.set_background_color;
      background_color = rhs.background_color;
    } else if (rhs.set_background_color) {
      // 如果右操作数的背景色已设置，则将两个背景色进行按位或运算
      if (!background_color.is_rgb || !rhs.background_color.is_rgb)
        FMT_THROW(format_error("can't OR a terminal color"));
      background_color.value.rgb_color |= rhs.background_color.value.rgb_color;
    }

    // 将两个文本样式的强调属性进行按位或运算
    ems = static_cast<emphasis>(static_cast<uint8_t>(ems) |
                                static_cast<uint8_t>(rhs.ems));
    return *this;
  }

  // 友元函数，重载 | 运算符
  friend FMT_CONSTEXPR text_style operator|(text_style lhs,
                                            const text_style& rhs) {
    return lhs |= rhs;
  }

  // 重载 &= 运算符
  FMT_CONSTEXPR text_style& operator&=(const text_style& rhs) {
    // 如果前景色未设置，则使用右操作数的前景色
    if (!set_foreground_color) {
      set_foreground_color = rhs.set_foreground_color;
      foreground_color = rhs.foreground_color;
    } else if (rhs.set_foreground_color) {
      // 如果右操作数的前景色已设置，则将两个前景色进行按位与运算
      if (!foreground_color.is_rgb || !rhs.foreground_color.is_rgb)
        FMT_THROW(format_error("can't AND a terminal color"));
      foreground_color.value.rgb_color &= rhs.foreground_color.value.rgb_color;
    }

    // 如果背景色未设置，则使用右操作数的背景色
    if (!set_background_color) {
      set_background_color = rhs.set_background_color;
      background_color = rhs.background_color;
    } else if (rhs.set_background_color) {
      // 如果右操作数的背景色已设置，则将两个背景色进行按位与运算
      if (!background_color.is_rgb || !rhs.background_color.is_rgb)
        FMT_THROW(format_error("can't AND a terminal color"));
      background_color.value.rgb_color &= rhs.background_color.value.rgb_color;
    }
    } else if (rhs.set_background_color) {
      // 如果 rhs 设置了背景颜色
      if (!background_color.is_rgb || !rhs.background_color.is_rgb)
        // 如果背景颜色不是 RGB 格式，抛出格式错误异常
        FMT_THROW(format_error("can't AND a terminal color"));
      // 对当前对象的背景颜色和 rhs 的背景颜色进行按位与操作
      background_color.value.rgb_color &= rhs.background_color.value.rgb_color;
    }

    // 将当前对象的 emphasis 属性和 rhs 的 emphasis 属性进行按位与操作
    ems = static_cast<emphasis>(static_cast<uint8_t>(ems) &
                                static_cast<uint8_t>(rhs.ems));
    // 返回当前对象的引用
    return *this;
  }

  // 重载位与运算符，实现 text_style 对象的按位与操作
  friend FMT_CONSTEXPR text_style operator&(text_style lhs,
                                            const text_style& rhs) {
    return lhs &= rhs;
  }

  // 返回是否设置了前景色
  FMT_CONSTEXPR bool has_foreground() const FMT_NOEXCEPT {
    return set_foreground_color;
  }
  // 返回是否设置了背景色
  FMT_CONSTEXPR bool has_background() const FMT_NOEXCEPT {
    return set_background_color;
  }
  // 返回是否设置了 emphasis
  FMT_CONSTEXPR bool has_emphasis() const FMT_NOEXCEPT {
    return static_cast<uint8_t>(ems) != 0;
  }
  // 返回前景色
  FMT_CONSTEXPR detail::color_type get_foreground() const FMT_NOEXCEPT {
    FMT_ASSERT(has_foreground(), "no foreground specified for this style");
    return foreground_color;
  }
  // 返回背景色
  FMT_CONSTEXPR detail::color_type get_background() const FMT_NOEXCEPT {
    FMT_ASSERT(has_background(), "no background specified for this style");
    return background_color;
  }
  // 返回 emphasis
  FMT_CONSTEXPR emphasis get_emphasis() const FMT_NOEXCEPT {
    FMT_ASSERT(has_emphasis(), "no emphasis specified for this style");
    return ems;
  }

 private:
  // 私有构造函数，用于设置前景色或背景色
  FMT_CONSTEXPR text_style(bool is_foreground,
                           detail::color_type text_color) FMT_NOEXCEPT
      : set_foreground_color(),
        set_background_color(),
        ems() {
    if (is_foreground) {
      // 如果是前景色，设置前景色并标记已设置前景色
      foreground_color = text_color;
      set_foreground_color = true;
    } else {
      // 如果是背景色，设置背景色并标记已设置背景色
      background_color = text_color;
      set_background_color = true;
    }  // 结束类定义

  }  // 结束命名空间定义

  // 声明友元函数，用于设置文本样式的前景色
  friend FMT_CONSTEXPR_DECL text_style fg(detail::color_type foreground)
      FMT_NOEXCEPT;
  // 声明友元函数，用于设置文本样式的背景色
  friend FMT_CONSTEXPR_DECL text_style bg(detail::color_type background)
      FMT_NOEXCEPT;

  // 前景色
  detail::color_type foreground_color;
  // 背景色
  detail::color_type background_color;
  // 是否设置了前景色
  bool set_foreground_color;
  // 是否设置了背景色
  bool set_background_color;
  // 文本样式的强调
  emphasis ems;
};

// 返回具有指定前景色的文本样式
FMT_CONSTEXPR text_style fg(detail::color_type foreground) FMT_NOEXCEPT {
  return text_style(/*is_foreground=*/true, foreground);
}

// 返回具有指定背景色的文本样式
FMT_CONSTEXPR text_style bg(detail::color_type background) FMT_NOEXCEPT {
  return text_style(/*is_foreground=*/false, background);
}

// 返回两个强调样式的组合
FMT_CONSTEXPR text_style operator|(emphasis lhs, emphasis rhs) FMT_NOEXCEPT {
  return text_style(lhs) | rhs;
}

namespace detail {

// ANSI颜色转义序列
template <typename Char> struct ansi_color_escape {
  FMT_CONSTEXPR ansi_color_escape(detail::color_type text_color,
                                  const char* esc) FMT_NOEXCEPT {
    // 如果有终端颜色，需要输出另一个转义码序列
    if (!text_color.is_rgb) {
      bool is_background = esc == detail::data::background_color;
      uint32_t value = text_color.value.term_color;
      // 背景ASCII码与前景相同，只是多了10
      if (is_background) value += 10u;

      size_t index = 0;
      buffer[index++] = static_cast<Char>('\x1b');
      buffer[index++] = static_cast<Char>('[');

      if (value >= 100u) {
        buffer[index++] = static_cast<Char>('1');
        value %= 100u;
      }
      buffer[index++] = static_cast<Char>('0' + value / 10u);
      buffer[index++] = static_cast<Char>('0' + value % 10u);

      buffer[index++] = static_cast<Char>('m');
      buffer[index++] = static_cast<Char>('\0');
      return;
    }

    for (int i = 0; i < 7; i++) {
      buffer[i] = static_cast<Char>(esc[i]);
    }
    rgb color(text_color.value.rgb_color);
    to_esc(color.r, buffer + 7, ';');
    to_esc(color.g, buffer + 11, ';');
    to_esc(color.b, buffer + 15, 'm');
    buffer[19] = static_cast<Char>(0);
  }
  // ANSI颜色转义序列的构造函数
  FMT_CONSTEXPR ansi_color_escape(emphasis em) FMT_NOEXCEPT {
    uint8_t em_codes[4] = {};
    uint8_t em_bits = static_cast<uint8_t>(em);
    if (em_bits & static_cast<uint8_t>(emphasis::bold)) em_codes[0] = 1;
    # 如果文本样式中包含斜体，设置对应的代码为3
    if (em_bits & static_cast<uint8_t>(emphasis::italic)) em_codes[1] = 3;
    # 如果文本样式中包含下划线，设置对应的代码为4
    if (em_bits & static_cast<uint8_t>(emphasis::underline)) em_codes[2] = 4;
    # 如果文本样式中包含删除线，设置对应的代码为9
    if (em_bits & static_cast<uint8_t>(emphasis::strikethrough))
      em_codes[3] = 9;

    # 初始化索引为0
    size_t index = 0;
    # 遍历文本样式代码数组
    for (int i = 0; i < 4; ++i) {
      # 如果当前样式代码为0，跳过
      if (!em_codes[i]) continue;
      # 将转义字符和样式代码添加到缓冲区
      buffer[index++] = static_cast<Char>('\x1b');
      buffer[index++] = static_cast<Char>('[');
      buffer[index++] = static_cast<Char>('0' + em_codes[i]);
      buffer[index++] = static_cast<Char>('m');
    }
    # 在缓冲区末尾添加空字符
    buffer[index++] = static_cast<Char>(0);
  }
  # 将对象转换为常量字符指针
  FMT_CONSTEXPR operator const Char*() const FMT_NOEXCEPT { return buffer; }

  # 返回缓冲区的起始地址
  FMT_CONSTEXPR const Char* begin() const FMT_NOEXCEPT { return buffer; }
  # 返回缓冲区的结束地址
  FMT_CONSTEXPR const Char* end() const FMT_NOEXCEPT {
    return buffer + std::char_traits<Char>::length(buffer);
  }

 private:
  # 定义缓冲区，长度为7 + 3 * 4 + 1
  Char buffer[7u + 3u * 4u + 1u];

  # 将整数转换为转义字符序列
  static FMT_CONSTEXPR void to_esc(uint8_t c, Char* out,
                                   char delimiter) FMT_NOEXCEPT {
    out[0] = static_cast<Char>('0' + c / 100);
    out[1] = static_cast<Char>('0' + c / 10 % 10);
    out[2] = static_cast<Char>('0' + c % 10);
    out[3] = static_cast<Char>(delimiter);
  }
}; // 结束了一个代码块

template <typename Char>
FMT_CONSTEXPR ansi_color_escape<Char> make_foreground_color(
    detail::color_type foreground) FMT_NOEXCEPT {
  return ansi_color_escape<Char>(foreground, detail::data::foreground_color);
} // 创建前景色 ANSI 转义序列

template <typename Char>
FMT_CONSTEXPR ansi_color_escape<Char> make_background_color(
    detail::color_type background) FMT_NOEXCEPT {
  return ansi_color_escape<Char>(background, detail::data::background_color);
} // 创建背景色 ANSI 转义序列

template <typename Char>
FMT_CONSTEXPR ansi_color_escape<Char> make_emphasis(emphasis em) FMT_NOEXCEPT {
  return ansi_color_escape<Char>(em);
} // 创建强调样式 ANSI 转义序列

template <typename Char>
inline void fputs(const Char* chars, FILE* stream) FMT_NOEXCEPT {
  std::fputs(chars, stream);
} // 向文件流中输出字符序列

template <>
inline void fputs<wchar_t>(const wchar_t* chars, FILE* stream) FMT_NOEXCEPT {
  std::fputws(chars, stream);
} // 向文件流中输出宽字符序列

template <typename Char> inline void reset_color(FILE* stream) FMT_NOEXCEPT {
  fputs(detail::data::reset_color, stream);
} // 重置文件流的颜色设置

template <> inline void reset_color<wchar_t>(FILE* stream) FMT_NOEXCEPT {
  fputs(detail::data::wreset_color, stream);
} // 重置宽字符文件流的颜色设置

template <typename Char>
inline void reset_color(buffer<Char>& buffer) FMT_NOEXCEPT {
  const char* begin = data::reset_color;
  const char* end = begin + sizeof(data::reset_color) - 1;
  buffer.append(begin, end);
} // 重置缓冲区的颜色设置

template <typename Char>
void vformat_to(buffer<Char>& buf, const text_style& ts,
                basic_string_view<Char> format_str,
                basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  bool has_style = false; // 初始化样式标志为假
  if (ts.has_emphasis()) { // 如果文本样式中有强调
    has_style = true; // 设置样式标志为真
    auto emphasis = detail::make_emphasis<Char>(ts.get_emphasis()); // 创建强调样式 ANSI 转义序列
    buf.append(emphasis.begin(), emphasis.end()); // 将强调样式 ANSI 转义序列添加到缓冲区
  }
  if (ts.has_foreground()) { // 如果文本样式中有前景色
    has_style = true; // 设置样式标志为真
    auto foreground = detail::make_foreground_color<Char>(ts.get_foreground()); // 创建前景色 ANSI 转义序列
    buf.append(foreground.begin(), foreground.end()); // 将前景色 ANSI 转义序列添加到缓冲区
  }
  if (ts.has_background()) { // 如果文本样式中有背景色
    has_style = true; // 设置样式标志为真
    // 使用文本样式设置背景颜色
    auto background = detail::make_background_color<Char>(ts.get_background());
    // 将背景颜色添加到缓冲区
    buf.append(background.begin(), background.end());
  }
  // 格式化字符串和参数，将结果添加到缓冲区
  detail::vformat_to(buf, format_str, args);
  // 如果存在样式，重置颜色
  if (has_style) detail::reset_color<Char>(buf);
}  // namespace detail

// 使用 ANSI 转义序列指定文本格式，格式化字符串并将其打印到指定的文件流
template <typename S, typename Char = char_t<S>>
void vprint(std::FILE* f, const text_style& ts, const S& format,
            basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  // 创建字符缓冲区
  basic_memory_buffer<Char> buf;
  // 使用 vformat_to 函数将格式化后的字符串写入缓冲区
  detail::vformat_to(buf, ts, to_string_view(format), args);
  // 在缓冲区末尾添加空字符
  buf.push_back(Char(0));
  // 将缓冲区的内容写入文件流
  detail::fputs(buf.data(), f);
}

/**
  \rst
  使用 ANSI 转义序列指定文本格式，格式化字符串并将其打印到指定的文件流

  **示例**::

    fmt::print(fmt::emphasis::bold | fg(fmt::color::red),
               "Elapsed time: {0:.2f} seconds", 1.23);
  \endrst
 */
template <typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_string<S>::value)>
void print(std::FILE* f, const text_style& ts, const S& format_str,
           const Args&... args) {
  // 调用 vprint 函数打印格式化后的字符串到指定的文件流
  vprint(f, ts, format_str,
         fmt::make_args_checked<Args...>(format_str, args...));
}

/**
  使用 ANSI 转义序列指定文本格式，格式化字符串并将其打印到标准输出流

  示例:
    fmt::print(fmt::emphasis::bold | fg(fmt::color::red),
               "Elapsed time: {0:.2f} seconds", 1.23);
 */
template <typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_string<S>::value)>
void print(const text_style& ts, const S& format_str, const Args&... args) {
  // 调用 print 函数将格式化后的字符串打印到标准输出流
  return print(stdout, ts, format_str, args...);
}

template <typename S, typename Char = char_t<S>>
inline std::basic_string<Char> vformat(
    const text_style& ts, const S& format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  // 创建字符缓冲区
  basic_memory_buffer<Char> buf;
  // 使用 vformat_to 函数将格式化后的字符串写入缓冲区
  detail::vformat_to(buf, ts, to_string_view(format_str), args);
  // 将缓冲区的内容转换为字符串并返回
  return fmt::to_string(buf);
}

/**
  \rst
  使用 ANSI 转义序列指定文本格式，格式化参数并将结果作为字符串返回

  **示例**::

    #include <fmt/color.h>
    # 使用 fmt 库的 format 函数格式化字符串，设置为加粗和红色
    std::string message = fmt::format(fmt::emphasis::bold | fg(fmt::color::red),
                                      "The answer is {}", 42);
/*
  使用给定的文本样式格式化字符串，并将输出写入“out”中
 */
template <typename S, typename... Args, typename Char = char_t<S>>
inline std::basic_string<Char> format(const text_style& ts, const S& format_str,
                                      const Args&... args) {
  return vformat(ts, to_string_view(format_str),
                 fmt::make_args_checked<Args...>(format_str, args...));
}

/**
  使用给定的文本样式格式化字符串，并将结果写入输出迭代器“out”中
 */
template <typename OutputIt, typename Char,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value)>
OutputIt vformat_to(
    OutputIt out, const text_style& ts, basic_string_view<Char> format_str,
    basic_format_args<buffer_context<type_identity_t<Char>>> args) {
  decltype(detail::get_buffer<Char>(out)) buf(detail::get_buffer_init(out));
  detail::vformat_to(buf, ts, format_str, args);
  return detail::get_iterator(buf);
}

/**
  格式化参数与给定的文本样式，将结果写入输出迭代器“out”，并返回输出范围结束后的迭代器

  **示例**::

    std::vector<char> out;
    fmt::format_to(std::back_inserter(out),
                   fmt::emphasis::bold | fg(fmt::color::red), "{}", 42);
*/
template <typename OutputIt, typename S, typename... Args,
          FMT_ENABLE_IF(detail::is_output_iterator<OutputIt>::value&&
                            detail::is_string<S>::value)>
inline OutputIt format_to(OutputIt out, const text_style& ts,
                          const S& format_str, Args&&... args) {
  return vformat_to(out, ts, to_string_view(format_str),
                    fmt::make_args_checked<Args...>(format_str, args...));
}

FMT_END_NAMESPACE

#endif  // FMT_COLOR_H_
```